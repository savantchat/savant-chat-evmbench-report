# Audit report — 2025-10-sequence/H-01

## Finding 1: Checkpointer Validation Bypass via Chained Signatures in `BaseSig.recover`

### Synopsis

In `BaseSig.recover`, a chained signature can invoke `recoverChained` without setting the checkpointer flag on the outer header (or by nesting chained signatures), allowing inner signatures to populate `_checkpointer` while `_ignoreCheckpointer` is `true` and completely bypassing `ICheckpointer.snapshotFor` validation.

### Technical details

In Sequence v3 configurations that use a checkpointer, the checkpointer address is committed at the top of the configuration Merkle tree (`imageHash`):

```solidity
imageHash = LibOptim.fkeccak256(imageHash, bytes32(threshold));
imageHash = LibOptim.fkeccak256(imageHash, bytes32(checkpoint));
imageHash = LibOptim.fkeccak256(imageHash, bytes32(uint256(uint160(_checkpointer))));
```

When a normal signature is validated via `BaseSig.recover(_payload, _signature, false, address(0))`, setting bit 6 (`0x40`) in `signatureFlag` reads `_checkpointer` from the signature and—because `_ignoreCheckpointer` is `false`—queries `ICheckpointer(_checkpointer).snapshotFor(address(this), checkpointerData)` to obtain `snapshot`. Lines 138–140 then enforce that the recovered `imageHash` either matches `snapshot.imageHash` or has a strictly newer `checkpoint > snapshot.checkpoint`.

However, when `signatureFlag & 0x01 == 0x01` (lines 109–111), `BaseSig.recover` delegates recovery to `recoverChained`:

```solidity
if (signatureFlag & 0x01 == 0x01) {
  return recoverChained(_payload, _checkpointer, snapshot, _signature[rindex:]);
}
```

In `recoverChained`, each link in the chain is recovered via recursive calls to `recover` with `_ignoreCheckpointer = true`:

```solidity
address checkpointer = nrindex == _signature.length ? _checkpointer : address(0);

if (prevCheckpoint == type(uint256).max) {
  (threshold, weight, imageHash, checkpoint, opHash) =
    recover(_payload, _signature[rindex:nrindex], true, checkpointer);
} else {
  (threshold, weight, imageHash, checkpoint,) =
    recover(linkedPayload, _signature[rindex:nrindex], true, checkpointer);
}
```

This creates two related flaws in how `recover` handles chained signatures and checkpointers:

1. **Unchecked Checkpointer Override in Inner Signature:** If the outer chained signature sets `signatureFlag = 0x01` (chained signature, bit `0x40` unset), `_checkpointer` remains `address(0)` and `snapshot` remains empty (`Snapshot(bytes32(0), 0)`). In `recoverChained`, the final (or sole) link receives `checkpointer = address(0)` and `_ignoreCheckpointer = true`. Inside the recursive `recover` call, setting `signatureFlag & 0x40 == 0x40` satisfies `signatureFlag & 0x40 == 0x40 && _checkpointer == address(0)` (line 88), reading `_checkpointer` from the inner signature so the final `imageHash` matches the wallet's stored configuration, while `if (!_ignoreCheckpointer)` (line 93) skips calling `ICheckpointer(_checkpointer).snapshotFor(...)`. Both `snapshot.imageHash` in `recover` (line 138) and `_snapshot.imageHash` in `recoverChained` (line 191) remain `bytes32(0)`, completely skipping snapshot validation.
2. **Nested Chained Signatures Drop `snapshot`:** Even if the outer chained signature sets `0x40` and fetches a non-empty `snapshot`, `recover` does not forbid `signatureFlag & 0x01 == 0x01` when `_ignoreCheckpointer == true`. If the last link in `recoverChained` is itself a chained signature (`0x01`), the recursive `recover` call has `_checkpointer` set to the outer checkpointer and a fresh empty `Snapshot memory snapshot`, which it passes to an inner `recoverChained(_payload, _checkpointer, snapshot, _signature[rindex:])` at line 110. The inner `recoverChained` recovers the wallet's `imageHash` using `_checkpointer` with an empty `_snapshot`, and if an earlier link in the outer chain already matched `_snapshot.imageHash` (or if the final link has a forged higher checkpoint before chaining down), snapshot enforcement is bypassed.

### Assumptions and preconditions

1. A Sequence wallet is configured with a checkpointer contract (`_checkpointer != address(0)`) and currently has an older configuration (`imageHash_old`, `checkpoint_old`) stored on-chain (or counterfactually deployed) on a chain that has not yet been updated.
2. The wallet configuration has since been updated off-chain / via the checkpointer to a newer configuration (`imageHash_new`, `checkpoint_new > checkpoint_old`) that revokes or lowers the weight of a previously authorized signer.
3. The revoked signer still holds the private key(s) sufficient to meet the threshold of `imageHash_old`.

### Proof of concept

1. Assume a wallet on Chain B has on-chain `imageHash = imageHash1` (with `checkpoint = 1`, `threshold = 1`, signer `A` of weight `1`, and checkpointer `C`).
2. The wallet owner updates the wallet configuration to `imageHash2` (with `checkpoint = 2`, `threshold = 1`, signer `B` of weight `1`, checkpointer `C`, evicting signer `A`), and records `Snapshot(imageHash2, 2)` in checkpointer `C`.
3. If evicted signer `A` submits a normal signature under `imageHash1` with `signatureFlag = 0x40` (checkpointer enabled), `BaseSig.recover` queries `C.snapshotFor(...)`, receives `Snapshot(imageHash2, 2)`, sees `snapshot.imageHash != imageHash1` and `checkpoint (1) <= snapshot.checkpoint (2)`, and reverts with `UnusedSnapshot`.
4. Instead, signer `A` constructs an inner signature `innerSig` for an arbitrary transaction `_payload` signed by `A`:
   - `innerSig` has `signatureFlag = 0x44` (bit `0x40` set for checkpointer, `checkpointSize = 1`, `thresholdSize = 1`, non-chained), followed by the 20-byte address of `C` (and NO `checkpointerDataSize` or `checkpointerData`), `checkpoint = 1`, `threshold = 1`, and `A`'s compact ECDSA signature.
5. Signer `A` wraps `innerSig` in a 1-element chained signature `outerSig`:
   - Byte 0: `0x01` (`signatureFlag` with bit `0x01` set for chained, bit `0x40` unset).
   - Bytes 1..3: `uint24(innerSig.length)`.
   - Remaining bytes: `innerSig`.
6. When `BaseAuth.signatureValidation` calls `BaseSig.recover(_payload, outerSig, false, address(0))`:
   - In the outer `recover`, `signatureFlag` is `0x01`, so `_checkpointer = address(0)` and `snapshot = Snapshot(bytes32(0), 0)`.
   - Line 110 calls `recoverChained(_payload, address(0), Snapshot(bytes32(0), 0), outerSig[1:])`.
   - `recoverChained` reads `innerSig` and calls `recover(_payload, innerSig, true, address(0))`.
   - In the inner `recover`, `signatureFlag & 0x40 == 0x40 && _checkpointer == address(0)` is `true`, so `_checkpointer` is set to `C`, while `if (!_ignoreCheckpointer)` is `false` so `C.snapshotFor` is never called.
   - The inner `recover` computes `imageHash = imageHash1` (incorporating `_checkpointer = C`) and returns `(threshold = 1, weight = 1, imageHash = imageHash1, checkpoint = 1, opHash)`.
   - `BaseAuth.signatureValidation` verifies `weight >= threshold` and `_isValidImage(imageHash1) == true`, allowing the evicted signer `A` to execute arbitrary transactions from the wallet.

### Impact assessment

**Severity: High**

Any evicted or compromised historical signer from an outdated wallet configuration on a lagging chain (or counterfactual wallet deployment) can completely bypass the checkpointer module and execute unauthorized transactions or take over the wallet, defeating the core security guarantee of `ICheckpointer`.

### Remediation

1. Disallow nested chained signatures by reverting (or not entering `recoverChained`) when `_ignoreCheckpointer` is `true`.
2. In `recoverChained`, for the final link in the chain (`nrindex == _signature.length`), do not allow the inner signature to independently supply `_checkpointer` when `_ignoreCheckpointer` is `true`—require that any checkpointer used by the final link was already provided and validated in the outer `recover` call (where `_ignoreCheckpointer == false`).

```solidity
    // Recover the imageHash checkpointer if any
    // If _ignoreCheckpointer is true and _checkpointer is already provided (final chained link),
    // do not allow overriding _checkpointer without a snapshot check.
    if (signatureFlag & 0x40 == 0x40 && _checkpointer == address(0)) {
      (_checkpointer, rindex) = _signature.readAddress(rindex);

      if (!_ignoreCheckpointer) {
        // Next 3 bytes determine the checkpointer data size
        uint256 checkpointerDataSize;
        (checkpointerDataSize, rindex) = _signature.readUint24(rindex);

        // Read the checkpointer data
        bytes memory checkpointerData = _signature[rindex:rindex + checkpointerDataSize];

        // Call the middleware
        snapshot = ICheckpointer(_checkpointer).snapshotFor(address(this), checkpointerData);

        rindex += checkpointerDataSize;
      }
    }

    // If signature type is 01 or 11 we do a chained signature (only allowed at top-level)
    if (signatureFlag & 0x01 == 0x01) {
      if (_ignoreCheckpointer) {
        revert InvalidSignatureFlag(signatureFlag);
      }
      return recoverChained(_payload, _checkpointer, snapshot, _signature[rindex:]);
    }
```

Additionally, in `recoverChained`, ensure the final signature link cannot set its own `_checkpointer` when the outer chained signature did not validate a checkpointer (e.g., by only allowing `signatureFlag & 0x40 == 0x40` when `!_ignoreCheckpointer` or when `nrindex != _signature.length`).

## Finding 2: Cursor Desynchronization in `BaseSig.recover` When Decoding Checkpointer-Enabled Signatures

### Synopsis

In `BaseSig.recover`, advancing the signature read cursor (`rindex`) past the encoded checkpointer address and length-prefixed proof data is improperly conditioned on `_checkpointer == address(0)` and `!_ignoreCheckpointer`. When recovering chained or ignored-checkpointer signatures with flag `0x40` set, `rindex` becomes desynchronized, corrupting subsequent field decoding and breaking signature validation.

### Technical details

In `src/modules/auth/BaseSig.sol`, `BaseSig.recover` parses a signature starting with a 1-byte `signatureFlag`. When bit 6 (`0x40`) of `signatureFlag` is set, the signature includes a 20-byte checkpointer address followed by a 3-byte length prefix (`checkpointerDataSize`) and `checkpointerDataSize` bytes of proof data (`checkpointerData`).

However, lines 88–105 gate the cursor advancements (`rindex`) for these framed fields behind `_checkpointer == address(0)` and `!_ignoreCheckpointer`:

1. **Skipping checkpointer fields when `_checkpointer != address(0)` (Line 88):**
   In `recoverChained` (line 164), the final signature link in the chain (`nrindex == _signature.length`) is passed `checkpointer = _checkpointer` (the non-zero checkpointer address recovered from the outer chained signature header) and `_ignoreCheckpointer = true`. When `recover` is called for this final link and its `signatureFlag` has bit `0x40` set, the condition `if (signatureFlag & 0x40 == 0x40 && _checkpointer == address(0))` on line 88 evaluates to `false`. Consequently, `recover` skips reading the 20-byte checkpointer address (line 91) and any following checkpointer data fields, leaving `rindex` at `1`.

2. **Skipping length-prefixed checkpointer data when `_ignoreCheckpointer == true` (Lines 93–105):**
   When `recover` is called with `_ignoreCheckpointer = true` (such as for intermediate links in `recoverChained` where `checkpointer == address(0)`), line 91 advances `rindex` by 20 bytes to read `_checkpointer`, but lines 93–105 wrap `readUint24` (line 96) and `rindex += checkpointerDataSize` (line 104) inside `if (!_ignoreCheckpointer)`. If the encoded signature includes the 3-byte `checkpointerDataSize` length prefix and `checkpointerData` payload, `rindex` is not advanced past those `3 + checkpointerDataSize` bytes.

Because `rindex` is not advanced past the checkpointer framing bytes in `_signature`, subsequent reads in `recover`—specifically `checkpoint` (line 119), `threshold` (line 125), and `recoverBranch` (line 130)—consume the unskipped checkpointer address or proof bytes at the wrong offset. This corrupts the recovered `checkpoint`, `threshold`, `weight`, and `imageHash` or causes decoding to revert.

### Assumptions and preconditions

1. A wallet configuration uses a checkpointer (`_checkpointer != address(0)`), so signatures generated for this configuration have bit 6 (`0x40`) set in `signatureFlag`.
2. Signature recovery is invoked in a mode where either `_checkpointer != address(0)` is pre-supplied (the final signature link in `recoverChained`) or `_ignoreCheckpointer == true` is set on a signature containing length-prefixed checkpointer data.

### Proof of concept

1. A Sequence wallet is initially configured with `config1` (checkpoint `1`, threshold `1`, signer `S1`, and checkpointer `C` at a non-zero address).
2. The wallet configuration is updated off-chain to `config2` (checkpoint `2`, threshold `1`, signer `S2`, and checkpointer `C`).
3. To execute a payload on-chain using a chained signature:
   - The outer chained signature has `signatureFlag = 0x41` (`0x40` checkpointer flag | `0x01` chained flag), followed by the 20-byte address `C`, the 3-byte `checkpointerDataSize`, and `checkpointerData`.
   - The first chained link (for `config2`) and the final chained link (for `config1`) are encoded with `signatureFlag & 0x40 == 0x40`, containing the 20-byte checkpointer address `C` (and 3-byte `checkpointerDataSize` + `checkpointerData`).
4. During `BaseAuth.signatureValidation`, `BaseSig.recover` reads `C` and `snapshot` from the outer header and calls `recoverChained(_payload, C, snapshot, _signature[rindex:])`.
5. In `recoverChained`:
   - On the first link (`config2`), `recover` is called with `_ignoreCheckpointer = true` and `checkpointer = address(0)`. Line 91 reads the 20-byte address `C`, but because `if (!_ignoreCheckpointer)` is `false`, lines 96 and 104 do not advance `rindex` past the 3-byte `checkpointerDataSize` and `checkpointerData`.
   - On the final link (`config1`, where `nrindex == _signature.length`), `recover` is called with `_ignoreCheckpointer = true` and `checkpointer = C` (`!= address(0)`). At line 88, `signatureFlag & 0x40 == 0x40 && _checkpointer == address(0)` evaluates to `false`, so `rindex` is not even advanced past the 20-byte checkpointer address `C`.
6. In both cases, `rindex` is desynchronized and lines 119, 125, and 130 read the checkpointer bytes as `checkpoint`, `threshold`, and Merkle tree branch flags, causing signature recovery to fail or derive an invalid `imageHash`.

### Impact assessment

- **Severity:** High / Medium
- **Technical Effect:** `rindex` becomes misaligned with the actual field boundaries in `_signature`, causing `checkpoint`, `threshold`, and `recoverBranch` to decode unintended bytes.
- **Business Impact:** Wallets configured with a checkpointer cannot validate chained signatures (which are specifically what the checkpointer mechanism is designed to secure across chains, as described in `docs/CHAINED.md`), resulting in denial of service for chained configuration updates and transactions whenever a checkpointer is active.

### Remediation

Always parse and advance `rindex` past the checkpointer address and length-prefixed checkpointer data whenever `signatureFlag & 0x40 == 0x40` is set, while only conditionally overriding `_checkpointer` (when `_checkpointer == address(0)`) and invoking `ICheckpointer(_checkpointer).snapshotFor` (when `!_ignoreCheckpointer`):

```solidity
    if (signatureFlag & 0x40 == 0x40) {
      address checkpointer;
      (checkpointer, rindex) = _signature.readAddress(rindex);

      if (_checkpointer == address(0)) {
        _checkpointer = checkpointer;
      }

      // Next 3 bytes determine the checkpointer data size
      uint256 checkpointerDataSize;
      (checkpointerDataSize, rindex) = _signature.readUint24(rindex);

      if (!_ignoreCheckpointer) {
        // Read the checkpointer data
        bytes memory checkpointerData = _signature[rindex:rindex + checkpointerDataSize];

        // Call the middleware
        snapshot = ICheckpointer(_checkpointer).snapshotFor(address(this), checkpointerData);
      }

      rindex += checkpointerDataSize;
    }
```

## Finding 3: Unrestricted Recursive `recoverChained` Dispatch in `BaseSig.recover` Allows Nested Chained Signatures to Bypass Checkpoint Monotonicity (`WrongChainedCheckpointOrder`)

### Synopsis

`BaseSig.recover` unconditionally dispatches to `recoverChained` whenever `signatureFlag & 0x01 == 0x01`, even when `recover` is invoked recursively for an individual link inside `recoverChained`. This allows nested chained signatures that reset `prevCheckpoint` to `type(uint256).max`, bypassing `WrongChainedCheckpointOrder` and enabling evicted signers to replay revoked configurations.

### Technical details

In `src/modules/auth/BaseSig.sol`, `recover` checks bit 0 of `signatureFlag` at lines 109–111:

```solidity
// If signature type is 01 or 11 we do a chained signature
if (signatureFlag & 0x01 == 0x01) {
  return recoverChained(_payload, _checkpointer, snapshot, _signature[rindex:]);
}
```

When `recoverChained` validates a chained signature (lines 143–194), it iterates over each link in the chain from the newest configuration (signing `_payload`) back to the oldest configuration (matching the wallet's stored `imageHash`). To prevent configuration rollback loops and replay of evicted configurations, `recoverChained` initializes a local variable `uint256 prevCheckpoint = type(uint256).max;` (line 153) and enforces strictly decreasing checkpoints across consecutive links (lines 183–188):

```solidity
if (checkpoint >= prevCheckpoint) {
  revert WrongChainedCheckpointOrder(checkpoint, prevCheckpoint);
}

linkedPayload.imageHash = imageHash;
prevCheckpoint = checkpoint;
```

However, `recoverChained` validates each link by calling `recover` (lines 167–171), and only receives back the `(threshold, weight, imageHash, checkpoint, opHash)` of the final state returned by `recover`. Because `recover` at lines 109–110 does not prevent a link inside a chained signature from also having `signatureFlag & 0x01 == 0x01`, any link in `recoverChained` can itself be encoded as a nested chained signature.

When a link in the outer `recoverChained` (where `prevCheckpoint` is already constrained to some value $C_{\text{outer}}$) is a nested chained signature:
1. `recover` at line 110 invokes a new `recoverChained` call where the inner `prevCheckpoint` is freshly initialized to `type(uint256).max`.
2. The first sub-link of the inner `recoverChained` can use a configuration with checkpoint $C_{\text{inner,0}} \ge C_{\text{outer}}$ because it is only compared against `type(uint256).max`.
3. Subsequent sub-links in the inner `recoverChained` step down to a final sub-link with checkpoint $C_{\text{inner,last}} < C_{\text{outer}}$.
4. The inner `recoverChained` returns only $C_{\text{inner,last}}$ (and its corresponding `imageHash`) to the outer `recoverChained`.
5. The outer `recoverChained` checks `C_inner,last >= C_outer` (which is `false`), completely missing the non-monotonic jump $C_{\text{outer}} \to C_{\text{inner,0}}$.

### Assumptions and preconditions

1. A wallet configuration history contains a configuration update from a higher-checkpoint configuration $C_3$ (e.g., `checkpoint = 3`) back to an earlier configuration $C_1$ (e.g., `checkpoint = 1`) where a previously authorized configuration $C_2$ (`checkpoint = 2`) had been active and later revoked, or a non-monotonic checkpoint transition was signed across configurations.
2. The wallet's on-chain `imageHash` is at an ancestor configuration $C_0$ (`checkpoint = 0`) that precedes $C_3$.

### Proof of concept

Consider a wallet with on-chain configuration $C_0$ (`checkpoint = 0`, signers `A & B`) and the following signed off-chain configuration updates:
- $C_0$ (`checkpoint = 0`, signers `A & B`) signed `KIND_CONFIG_UPDATE` to $C_3$ (`checkpoint = 3`, signers `A & B`).
- $C_1$ (`checkpoint = 1`, signers `A & B`) signed `KIND_CONFIG_UPDATE` to $C_2$ (`checkpoint = 2`, signer `C` with sufficient weight to meet threshold).
- $C_3$ (`checkpoint = 3`, signers `A & B`) signed `KIND_CONFIG_UPDATE` to $C_1$ (`checkpoint = 1`, signers `A & B`).

In a flat chained signature `Payload <- C2 (cp 2) <- C1 (cp 1) <- C3 (cp 3) <- C0 (cp 0)`, `recoverChained` reverts at the transition from $C_1$ (`prevCheckpoint = 1`) to $C_3$ (`checkpoint = 3`) with `WrongChainedCheckpointOrder(3, 1)` because `3 >= 1`.

Evicted signer `C` bypasses this check by constructing a nested chained signature:
1. **Outer Chained Signature** (`signatureFlag = 0x01`):
   - **Outer Link 0**: Normal signature by `C` using $C_2$ (`checkpoint = 2`) over `_payload`.
     - Outer `recoverChained` checks `2 >= type(uint256).max` (`false`), sets `linkedPayload.imageHash = C2`, and sets outer `prevCheckpoint = 2`.
   - **Outer Link 1**: Normal signature using $C_1$ (`checkpoint = 1`) over `KIND_CONFIG_UPDATE(C2)`.
     - Outer `recoverChained` checks `1 >= 2` (`false`), sets `linkedPayload.imageHash = C1`, and sets outer `prevCheckpoint = 1`.
   - **Outer Link 2**: Chained signature (`signatureFlag = 0x01`) over `KIND_CONFIG_UPDATE(C1)` containing two sub-links:
     - When `recover` processes Outer Link 2, line 109 (`signatureFlag & 0x01 == 0x01`) enters a recursive `recoverChained` call where inner `prevCheckpoint` is reset to `type(uint256).max`.
     - **Inner Sub-link 0**: Normal signature using $C_3$ (`checkpoint = 3`) over `KIND_CONFIG_UPDATE(C1)`.
       - Inner `recoverChained` checks `3 >= type(uint256).max` (`false`), sets inner `linkedPayload.imageHash = C3`, and sets inner `prevCheckpoint = 3`.
     - **Inner Sub-link 1**: Normal signature using $C_0$ (`checkpoint = 0`) over `KIND_CONFIG_UPDATE(C3)`.
       - Inner `recoverChained` checks `0 >= 3` (`false`), sets inner `imageHash = C0`, and sets inner `checkpoint = 0`.
     - The inner `recoverChained` returns `(threshold, weight, imageHash = C0, checkpoint = 0, opHash)` to the outer `recoverChained`.
2. The outer `recoverChained` checks `checkpoint >= prevCheckpoint` (`0 >= 1`, which is `false`), accepts the link, and returns `imageHash = C0`.
3. `BaseAuth.signatureValidation` verifies `_isValidImage(C0) == true` and authorizes `_payload` signed by evicted signer `C`.

### Impact assessment

**Severity: High**

An evicted or compromised signer from a historical configuration can bypass the strict checkpoint ordering guarantee (`WrongChainedCheckpointOrder`) in chained signatures by nesting a chained signature inside another chained signature. If a wallet's state channel ever reuses a configuration or signs a transition to a lower checkpoint, the evicted signer can regain unauthorized control over the wallet and execute arbitrary transactions or sign arbitrary messages.

### Remediation

Disallow chained signatures inside links of an already-chained signature (or reject nested `recoverChained` invocations). If `_ignoreCheckpointer` is used exclusively to indicate recursive `recover` calls from `recoverChained` in production, revert when `signatureFlag & 0x01 == 0x01` and `_ignoreCheckpointer` is `true` (or pass an explicit `bool _isChained` flag / forbid `signatureFlag & 0x01 == 0x01` on inner links):

```solidity
    // If signature type is 01 or 11 we do a chained signature
    if (signatureFlag & 0x01 == 0x01) {
      if (_ignoreCheckpointer) {
        revert InvalidSignatureFlag(signatureFlag);
      }
      return recoverChained(_payload, _checkpointer, snapshot, _signature[rindex:]);
    }
```

## Finding 4: Checkpointer Snapshot Enforcement Bypass in `BaseSig.recover` Allows Stale Configurations to Sign Payloads

### Synopsis

In `BaseSig.recover`, `_checkpointer` is hashed into `imageHash` (line 134) even when `_ignoreCheckpointer` skipped fetching `snapshot`, while lines 138–140 only enforce snapshot validity if `snapshot.imageHash != bytes32(0)`. An evicted signer can wrap a stale configuration in a chained signature to bypass checkpointer validation completely.

### Technical details

In Sequence v3 wallets, a configuration can specify a `_checkpointer` contract that announces the latest valid configuration (`Snapshot({ imageHash, checkpoint })`) so that lagging chains reject signatures from older, superseded configurations where signers may have been evicted.

In `BaseSig.recover` (`src/modules/auth/BaseSig.sol`), checkpointer recovery and enforcement work as follows:

1. At lines 88–106, if bit 6 (`0x40`) of `signatureFlag` is set and `_checkpointer == address(0)`, `recover` reads `_checkpointer` from `_signature`. However, it only reads `checkpointerData` and queries `snapshot = ICheckpointer(_checkpointer).snapshotFor(address(this), checkpointerData)` when `!_ignoreCheckpointer` (line 93).
2. At line 134, `recover` unconditionally binds `_checkpointer` into the recovered configuration root:
   ```solidity
   imageHash = LibOptim.fkeccak256(imageHash, bytes32(uint256(uint160(_checkpointer))));
   ```
3. At lines 138–140, `recover` only enforces snapshot freshness when `snapshot.imageHash != bytes32(0)`:
   ```solidity
   if (snapshot.imageHash != bytes32(0) && snapshot.imageHash != imageHash && checkpoint <= snapshot.checkpoint) {
     revert UnusedSnapshot(snapshot);
   }
   ```

When a signature is submitted as a chained signature (`signatureFlag & 0x01 == 0x01` at line 109) without setting bit 6 (`0x40`) on the outer chained header:
- In the outer `recover` call, `_checkpointer` remains `address(0)` and `snapshot` remains empty (`snapshot.imageHash == bytes32(0)`).
- `recoverChained` is invoked with `_checkpointer = address(0)` and `_snapshot = Snapshot(bytes32(0), 0)`.
- Inside `recoverChained`, each link in the chain (which can even be a single-element chain containing only the stale on-chain configuration, or a chain ending at that stale configuration) calls `recover(..., true, checkpointer)` where `_ignoreCheckpointer == true` and `checkpointer == address(0)` (line 164).
- Inside the inner `recover` call, the attacker sets bit 6 (`0x40`) on the inner `signatureFlag`. Because `_checkpointer == address(0)`, line 91 reads the wallet's real `_checkpointer` address from the inner signature, and line 134 hashes it into `imageHash` so the resulting `imageHash` matches the wallet's on-chain storage.
- Because `_ignoreCheckpointer` is `true`, `ICheckpointer(_checkpointer).snapshotFor` is never called, leaving `snapshot.imageHash == bytes32(0)` at line 138 (and `_snapshot.imageHash == bytes32(0)` at line 191).
- Similarly, if a chained signature transitions through an intermediate configuration that defines a checkpointer (`nrindex != _signature.length`), `recoverChained` passes `checkpointer = address(0)` and `_ignoreCheckpointer = true`, allowing line 134 to compute the valid intermediate `imageHash` without consulting its checkpointer or enforcing its snapshot at line 138.

As a result, any configuration protected by a checkpointer can be used without querying the checkpointer or enforcing its snapshot, completely defeating checkpointer-based replay and rollback protection.

### Assumptions and preconditions

1. A Sequence wallet is deployed on a chain where its on-chain configuration `C1` (with `checkpoint = 1`) defines a non-zero `_checkpointer` contract.
2. Off-chain (and recorded in the checkpointer), the wallet's configuration has been updated to a newer configuration `C2` (with `checkpoint = 2`) that revokes a previously authorized signer `S`, but `C2` has not yet been settled on the lagging chain.
3. The revoked signer `S` possesses sufficient weight in the stale configuration `C1` to meet `C1`'s threshold.

### Proof of concept

1. The wallet on Chain B currently stores `imageHash = C1` (`checkpoint = 1`, `threshold = 1`, signer `S` with weight `1`, `_checkpointer = P`).
2. The wallet owner updates the configuration to `C2` (`checkpoint = 2`, `threshold = 1`, signer `Owner` with weight `1`, `_checkpointer = P`), evicting `S`. Checkpointer `P`'s `snapshotFor` now returns `Snapshot({ imageHash: C2, checkpoint: 2 })`.
3. If `S` submits a normal (non-chained) signature using `C1`, `S` must set bit 6 (`0x40`) so `_checkpointer = P` is hashed into `imageHash` at line 134. `recover` queries `P.snapshotFor(...)`, receives `Snapshot(C2, 2)`, and reverts at line 138–139 (`snapshot.imageHash != C1 && 1 <= 2`) with `UnusedSnapshot`.
4. To bypass this enforcement, `S` takes the inner signature for `C1` (with `signatureFlag` having bit 6 `0x40` set, followed by `abi.encodePacked(P)` and no checkpointer data length/bytes) and wraps it in a chained signature container:
   - Outer byte: `0x01` (chained signature flag set, bit 6 `0x40` unset).
   - Next 3 bytes: `uint24(innerSig.length)`.
   - Remaining bytes: `innerSig`.
5. When `BaseAuth.signatureValidation` calls `BaseSig.recover(_payload, wrappedSig, false, address(0))`:
   - The outer `recover` sees `signatureFlag = 0x01` (bit 6 unset) and calls `recoverChained(_payload, address(0), Snapshot(bytes32(0), 0), wrappedSig[1:])`.
   - `recoverChained` calls `recover(_payload, innerSig, true, address(0))`.
   - The inner `recover` sees `innerSig`'s bit 6 (`0x40`) set and `_checkpointer == address(0)`, so it reads `_checkpointer = P` at line 91, skips `snapshotFor` because `_ignoreCheckpointer == true`, hashes `P` into `imageHash` at line 134 to produce `C1`, and skips the `UnusedSnapshot` revert at line 138 because `snapshot.imageHash == bytes32(0)`.
   - `recoverChained` finishes with `imageHash = C1` and `_snapshot.imageHash == bytes32(0)`, skipping line 191.
6. `BaseAuth.signatureValidation` verifies `_isValidImage(C1) == true` and executes `S`'s unauthorized transaction payload.

### Impact assessment

- **Severity:** High / Critical
- **Prerequisites:** A wallet configuration using a checkpointer where an older configuration remains in storage (or in a valid chain prefix) on any chain after a signer revocation or threshold rotation has been published to the checkpointer.
- **Consequences:** Revoked or compromised signers from any stale configuration can completely bypass the checkpointer module and execute arbitrary transactions (including draining all wallet funds or taking over the wallet configuration) on any chain that has not yet settled the latest configuration on-chain.

### Remediation

Ensure that whenever a configuration specifies a non-zero `_checkpointer` at line 134, the checkpointer snapshot has actually been fetched and validated (or disallow overriding `_checkpointer` when `_ignoreCheckpointer` is set, and query the checkpointer whenever `_checkpointer != address(0)` is recovered):

```solidity
    // Recover the tree
    opHash = _payload.hash();
    (weight, imageHash) = recoverBranch(_payload, opHash, _signature[rindex:]);

    imageHash = LibOptim.fkeccak256(imageHash, bytes32(threshold));
    imageHash = LibOptim.fkeccak256(imageHash, bytes32(checkpoint));
    imageHash = LibOptim.fkeccak256(imageHash, bytes32(uint256(uint160(_checkpointer))));

    // If a checkpointer is part of the recovered configuration, ensure its snapshot was not bypassed
    if (_checkpointer != address(0) && _ignoreCheckpointer && (signatureFlag & 0x40 == 0x40)) {
      revert UnusedSnapshot(snapshot);
    }

    // If the snapshot is used, either the imageHash must match
    // or the checkpoint must be greater than the snapshot checkpoint
    if (snapshot.imageHash != bytes32(0) && snapshot.imageHash != imageHash && checkpoint <= snapshot.checkpoint) {
      revert UnusedSnapshot(snapshot);
    }
```

## Finding 5: Nested Chained Signatures in `BaseSig.recoverChained` Reset `prevCheckpoint` and Bypass Strict Checkpoint Ordering

### Synopsis

In `BaseSig.recoverChained`, each link in a chained signature is validated by calling `BaseSig.recover`, which permits inner links to themselves be encoded as chained signatures (`signatureFlag & 0x01 == 0x01`). Because each recursive `recoverChained` call re-initializes `prevCheckpoint = type(uint256).max` and returns only its final link's `checkpoint`, intermediate configurations can violate strict checkpoint ordering (`WrongChainedCheckpointOrder`), enabling replay of revoked signer configurations.

### Technical details

Sequence wallets use chained signatures (`BaseSig.recoverChained`, lines 143–194) to prove a sequence of configuration updates from an older configuration stored on-chain to the active configuration signing `_payload`. As documented in `docs/CHAINED.md`, each configuration commits to a `checkpoint`, and `recoverChained` enforces that checkpoints strictly decrease as verification walks backward from the newest configuration to the oldest configuration:

```solidity
uint256 prevCheckpoint = type(uint256).max;

while (rindex < _signature.length) {
  ...
  if (prevCheckpoint == type(uint256).max) {
    (threshold, weight, imageHash, checkpoint, opHash) =
      recover(_payload, _signature[rindex:nrindex], true, checkpointer);
  } else {
    (threshold, weight, imageHash, checkpoint,) =
      recover(linkedPayload, _signature[rindex:nrindex], true, checkpointer);
  }
  ...
  if (checkpoint >= prevCheckpoint) {
    revert WrongChainedCheckpointOrder(checkpoint, prevCheckpoint);
  }

  linkedPayload.imageHash = imageHash;
  prevCheckpoint = checkpoint;
}
```

However, `recoverChained` delegates each link `_signature[rindex:nrindex]` to `BaseSig.recover` (lines 168 and 171), and `BaseSig.recover` (lines 109–111) unconditionally allows the sub-signature to have the chained signature flag set (`signatureFlag & 0x01 == 0x01`), triggering a recursive call to `recoverChained`.

When an outer chained signature contains a nested chained signature at a subsequent link (`prevCheckpoint != type(uint256).max`):

1. The outer loop has just processed an earlier link (e.g., `Config_1` with `checkpoint = C_1`), setting the outer `linkedPayload.imageHash = Config_1` and outer `prevCheckpoint = C_1`.
2. In the next outer iteration, `recover(linkedPayload, innerChainedSig, true, checkpointer)` invokes `recoverChained(linkedPayload, ...)` recursively.
3. The recursive `recoverChained` initializes a fresh local `prevCheckpoint = type(uint256).max` (line 153).
4. On the first iteration of the inner `recoverChained` (`prevCheckpoint == type(uint256).max`, line 166), it validates `linkedPayload` (the `KIND_CONFIG_UPDATE` approving `Config_1`) against the first inner configuration `Config_2` with `checkpoint = C_2`. Because the inner `prevCheckpoint` is `type(uint256).max`, `C_2` is never compared against the outer `prevCheckpoint` (`C_1`).
5. Subsequent iterations of the inner `recoverChained` transition from `Config_2` (`C_2`) down to an older configuration `Config_0` (`checkpoint = C_0`, where `C_0 < C_2`), and the recursive call returns only the final values `(threshold_0, weight_0, imageHash_0, C_0)` to the outer `recoverChained`.
6. The outer `recoverChained` then checks `if (checkpoint >= prevCheckpoint)` using `C_0 < C_1`, completely skipping the requirement that `C_2 < C_1`.

Because `KIND_CONFIG_UPDATE` payloads (`Payload.toEIP712`, lines 252–254) only hash the target `imageHash` and `parentWallets` without binding to the signing configuration's own `imageHash` or `checkpoint`, any signature by a signer set (e.g., `Signers: A & B`) approving `Config_1` (`checkpoint = 1`, `Signers: A, B & C`) remains cryptographically valid under a later configuration `Config_2` (`checkpoint = 2`) that shares the same signer set (`Signers: A & B`). By nesting `[Config_2 (cp = 2), Config_0 (cp = 0)]` inside an outer chain starting at `Config_1 (cp = 1)`, the forbidden backward transition `Checkpoint 2 -> Checkpoint 1` (`2 >= 1`) is completely bypassed.

### Assumptions and preconditions

1. A wallet is anchored on-chain at configuration `Config_0` (`checkpoint = C_0`, e.g., `0`).
2. A configuration update to `Config_1` (`checkpoint = C_1`, e.g., `1`, which includes a signer `C`) was previously signed by a signer set `S` (for example, on an earlier configuration before `C` was evicted, or via a `noChainId` configuration update).
3. The wallet transitioned from `Config_0` (`checkpoint = 0`) to `Config_2` (`checkpoint = C_2`, e.g., `2`, where `C_2 >= C_1`) whose active signer threshold can be satisfied by the same signer set `S` (with `C` evicted), relying on `C_2 >= C_1` to block replay of the transition into `Config_1`.

### Proof of concept

Consider the following configuration state channel progression:

1. **On-chain Configuration (`Config_0`)**: `checkpoint = 0`, signer `X` (weight 1, threshold 1).
2. **Revoked / Superseded Configuration (`Config_1`)**: `checkpoint = 1`, signers `A` (weight 1), `B` (weight 1), and `C` (weight 1), threshold 1. Signers `A` and `B` previously signed a `KIND_CONFIG_UPDATE` payload approving `Config_1.imageHash` (e.g., prior to `C` being evicted).
3. **Current Off-chain Configuration (`Config_2`)**: `checkpoint = 2`, signers `A` (weight 1) and `B` (weight 1), threshold 2 (`C` is not a signer). Signer `X` (`Config_0`) signed a `KIND_CONFIG_UPDATE` payload approving `Config_2.imageHash` with `checkpoint = 2 > 1` so that `Config_1` (`checkpoint = 1`) cannot be chained after `Config_2`.

To execute an arbitrary transaction payload `_payload` using the evicted signer `C`:

1. `C` signs `_payload` under `Config_1` (`checkpoint = 1`), producing sub-signature `sig_1`.
2. `C` takes `A` and `B`'s existing signatures over `KIND_CONFIG_UPDATE(Config_1.imageHash)` and packages them with `Config_2`'s Merkle tree (`checkpoint = 2`, threshold 2), producing sub-signature `sig_2_to_1`.
3. `C` takes `X`'s signature over `KIND_CONFIG_UPDATE(Config_2.imageHash)` under `Config_0` (`checkpoint = 0`), producing sub-signature `sig_0_to_2`.
4. Instead of submitting the flat 3-element chain `[sig_1, sig_2_to_1, sig_0_to_2]` (which reverts at `sig_2_to_1` with `WrongChainedCheckpointOrder(2, 1)`), `C` encodes `[sig_2_to_1, sig_0_to_2]` as an inner chained signature `sig_nested` (with leading flag `0x01`), and submits the 2-element outer chained signature `[sig_1, sig_nested]`:
   - **Outer Iteration 0 (`sig_1`)**: Recovers `Config_1`, `checkpoint = 1`, `opHash = _payload.hash()`. Sets outer `prevCheckpoint = 1` and outer `linkedPayload.imageHash = Config_1.imageHash`.
   - **Outer Iteration 1 (`sig_nested`)**: Calls `recover(linkedPayload, sig_nested, ...)`, which enters `recoverChained` with a fresh inner `prevCheckpoint = type(uint256).max`.
     - **Inner Iteration 0 (`sig_2_to_1`)**: Validates `linkedPayload` (`KIND_CONFIG_UPDATE(Config_1.imageHash)`) under `Config_2` (`checkpoint = 2`). Passes `2 < type(uint256).max` and sets inner `prevCheckpoint = 2` and inner `linkedPayload.imageHash = Config_2.imageHash`.
     - **Inner Iteration 1 (`sig_0_to_2`)**: Validates `KIND_CONFIG_UPDATE(Config_2.imageHash)` under `Config_0` (`checkpoint = 0`). Passes `0 < 2` and returns `imageHash = Config_0.imageHash` and `checkpoint = 0` to the outer loop.
   - **Outer Loop Completion**: Checks `checkpoint < prevCheckpoint` (`0 < 1`), which succeeds, and returns `imageHash = Config_0.imageHash`, passing `_isValidImage(imageHash)` in `BaseAuth.signatureValidation`.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: The strict checkpoint ordering invariant (`checkpoint < prevCheckpoint` across every link in a chained signature) can be bypassed whenever two links are wrapped in a nested chained signature.
- **Business Implications**: An evicted or revoked signer from an earlier checkpoint configuration (`Config_1`) can regain full signing authority over the wallet after the wallet has advanced to a higher checkpoint configuration (`Config_2`) that reuses a prior signer subset, leading to unauthorized transaction execution and loss of wallet funds.

### Remediation

Prevent nested chained signatures inside `recoverChained` (or pass `prevCheckpoint` into recursive `recover` / `recoverChained` calls so checkpoint monotonicity is enforced across all levels). The simplest and most gas-efficient fix is to reject the chained signature flag (`0x01`) inside `recoverChained` before invoking `recover` on each link:

```solidity
  error ChainedSignatureCannotBeNested();

  function recoverChained(
    Payload.Decoded memory _payload,
    address _checkpointer,
    Snapshot memory _snapshot,
    bytes calldata _signature
  ) internal view returns (uint256 threshold, uint256 weight, bytes32 imageHash, uint256 checkpoint, bytes32 opHash) {
    Payload.Decoded memory linkedPayload;
    linkedPayload.kind = Payload.KIND_CONFIG_UPDATE;

    uint256 rindex;
    uint256 prevCheckpoint = type(uint256).max;

    while (rindex < _signature.length) {
      uint256 nrindex;

      {
        uint256 sigSize;
        (sigSize, rindex) = _signature.readUint24(rindex);
        nrindex = sigSize + rindex;
      }

      if (uint8(_signature[rindex]) & 0x01 == 0x01) {
        revert ChainedSignatureCannotBeNested();
      }

      address checkpointer = nrindex == _signature.length ? _checkpointer : address(0);

      if (prevCheckpoint == type(uint256).max) {
        (threshold, weight, imageHash, checkpoint, opHash) =
          recover(_payload, _signature[rindex:nrindex], true, checkpointer);
      } else {
        (threshold, weight, imageHash, checkpoint,) =
          recover(linkedPayload, _signature[rindex:nrindex], true, checkpointer);
      }
```

## Finding 6: Missing `parentWallets` Propagation in `BaseSig.recoverChained` Causes Context Loss and Signature Validation Failure

### Synopsis

In `BaseSig.recoverChained`, the `linkedPayload` struct used to validate chained configuration updates is initialized without copying `_payload.parentWallets`. This strips the parent wallet context during `recoverSapientSignature` validation, breaking compliant Sapient chained signatures and allowing configuration update links to be replayed across different parent wallet contexts.

### Technical details

When a Sequence wallet acts as a Sapient signer (`ISapient`) for a parent wallet, `BaseAuth.recoverSapientSignature` records the caller context by appending `msg.sender` to `_payload.parentWallets` (lines 125–132 of `src/modules/auth/BaseAuth.sol`) before invoking `signatureValidation(_payload, _signature)` and `BaseSig.recover(_payload, _signature, false, address(0))`.

In `src/modules/Payload.sol`, the EIP-712 typehash for configuration updates explicitly commits to the `parentWallets` array:

```solidity
/// @dev keccak256("ConfigUpdate(bytes32 imageHash,address[] wallets)")
bytes32 private constant CONFIG_UPDATE_TYPEHASH = 0x11fdeb7e8373a1aa96bfac8d0ea91526b2c5d15e5cee20e0543e780258f3e8e4;
```

In `Payload.toEIP712`, `walletsHash` is computed as `keccak256(abi.encodePacked(_decoded.parentWallets))` and included in the struct hash for `KIND_CONFIG_UPDATE` (lines 242 and 252–254 of `src/modules/Payload.sol`).

However, when `BaseSig.recover` delegates to `BaseSig.recoverChained` (lines 143–150 of `src/modules/auth/BaseSig.sol`), `linkedPayload` is initialized as follows:

```solidity
Payload.Decoded memory linkedPayload;
linkedPayload.kind = Payload.KIND_CONFIG_UPDATE;
```

`linkedPayload.parentWallets` is left as an empty array instead of being populated with `_payload.parentWallets`. Consequently, while the first link in the chained signature (lines 166–168) is verified against `_payload` (which includes `_payload.parentWallets`), every subsequent configuration update link (lines 169–172) is verified against `linkedPayload` with an empty `parentWallets` array.

This causes two issues:
1. **Validation Failure for Context-Bound Chained Signatures:** Any chained configuration update signature properly signed over the `parentWallets` EIP-712 field will hash to a different `opHash` during `recover(linkedPayload, ...)`, recovering an unexpected signer address and deriving an invalid `imageHash`, which causes `recoverSapientSignature` to revert with `InvalidSapientSignature`.
2. **Cross-Wallet Context Loss / Replay:** Because `linkedPayload.parentWallets` is always empty, the EIP-712 digest for chained configuration transitions inside `recoverSapientSignature` does not bind to the calling parent wallet(s). A configuration transition signed for direct use on the Sapient wallet (where `parentWallets` is empty) can be replayed within `recoverSapientSignature` calls from any parent wallet.

### Assumptions and preconditions

- A Sequence wallet (`WalletB`, inheriting `BaseAuth`) is configured as a Sapient signer (`FLAG_SIGNATURE_SAPIENT`) in the configuration of a parent wallet (`WalletA`).
- A chained signature (`signatureFlag & 0x01 == 0x01`) is used to authenticate `WalletB` via `recoverSapientSignature`.

### Proof of concept

1. `WalletA` includes `WalletB` as a Sapient signer in its configuration tree (`FLAG_SIGNATURE_SAPIENT`).
2. `WalletB` transitions its configuration from `Config1` (checkpoint 1) to `Config2` (checkpoint 2).
3. To authorize an operation on `WalletA` using a chained signature on `WalletB`, `WalletB`'s signers sign:
   - The final operation payload `_payload` with `parentWallets = [WalletA]` under `Config2`.
   - The configuration update link `ConfigUpdate(imageHash: Config2, wallets: [WalletA])` under `Config1`.
4. During execution on `WalletA`, `BaseSig.recoverBranch` calls `ISapient(WalletB).recoverSapientSignature(_payload, sapientSig)`.
5. `WalletB.recoverSapientSignature` appends `WalletA` to `_payload.parentWallets` and calls `signatureValidation` -> `BaseSig.recover` -> `BaseSig.recoverChained`.
6. In `recoverChained`:
   - Iteration 0 calls `recover(_payload, ...)` with `_payload.parentWallets = [WalletA]`, recovering `Config2` as expected.
   - Iteration 1 calls `recover(linkedPayload, ...)` where `linkedPayload.imageHash = Config2`, but `linkedPayload.parentWallets = []`.
7. Because `linkedPayload.hash()` hashes `wallets = []` instead of `wallets = [WalletA]`, `ecrecover` recovers a random address instead of `Config1`'s signer, producing a mismatched `imageHash` and reverting with `InvalidSapientSignature`.
8. Conversely, if `Config1`'s signer signed `ConfigUpdate(imageHash: Config2, wallets: [])` intending it only for `WalletB` directly, an attacker who controls `Config2` can reuse that link inside `WalletB.recoverSapientSignature` for `WalletA` (or any other parent wallet) because `linkedPayload.parentWallets` is empty.

### Impact assessment

**Severity: Medium / High**

- **Broken Core Functionality:** Chained signatures for Sapient signers that properly include `parentWallets` in the `ConfigUpdate` EIP-712 struct cannot be validated and always revert.
- **Authorization Context Loss:** Chained configuration update links lose their `parentWallets` binding during Sapient signature recovery, allowing links signed without parent wallets (or for a nested parent wallet chain) to be replayed across unintended parent wallet contexts.

### Remediation

Propagate `_payload.parentWallets` to `linkedPayload.parentWallets` when initializing `linkedPayload` in `BaseSig.recoverChained`.

```solidity
  function recoverChained(
    Payload.Decoded memory _payload,
    address _checkpointer,
    Snapshot memory _snapshot,
    bytes calldata _signature
  ) internal view returns (uint256 threshold, uint256 weight, bytes32 imageHash, uint256 checkpoint, bytes32 opHash) {
    Payload.Decoded memory linkedPayload;
    linkedPayload.kind = Payload.KIND_CONFIG_UPDATE;
    linkedPayload.parentWallets = _payload.parentWallets;
```

## Finding 7: Recursive Chained Signature Recovery in `BaseSig.recoverChained` Bypasses Checkpoint Ordering and Snapshot Validation

### Synopsis

In `BaseSig.recoverChained`, each signature segment is validated by recursively calling `BaseSig.recover`. Because `recover` permits the chained signature flag (`0x01`) on inner segments, an attacker can nest chained signatures inside `recoverChained`, resetting `prevCheckpoint` across recursion boundaries and bypassing `WrongChainedCheckpointOrder` and `UnusedSnapshot` enforcement.

### Technical details

In `src/modules/auth/BaseSig.sol`, `recoverChained` iterates through length-prefixed signature segments in `_signature` and validates each segment by calling `recover` (lines 167–172):

```solidity
      if (prevCheckpoint == type(uint256).max) {
        (threshold, weight, imageHash, checkpoint, opHash) =
          recover(_payload, _signature[rindex:nrindex], true, checkpointer);
      } else {
        (threshold, weight, imageHash, checkpoint,) =
          recover(linkedPayload, _signature[rindex:nrindex], true, checkpointer);
      }
```

In a valid flat chained signature, configurations must progress in strictly increasing checkpoint order from the oldest (on-chain) configuration to the newest configuration signing `_payload`. During recovery (which walks backwards from the newest configuration to the oldest), `recoverChained` enforces that each recovered `checkpoint` is strictly less than `prevCheckpoint` (lines 183–185):

```solidity
      if (checkpoint >= prevCheckpoint) {
        revert WrongChainedCheckpointOrder(checkpoint, prevCheckpoint);
      }

      linkedPayload.imageHash = imageHash;
      prevCheckpoint = checkpoint;
```

However, `recover` does not prevent an inner segment `_signature[rindex:nrindex]` from also setting the chained signature bit (`signatureFlag & 0x01 == 0x01` at line 109). When an inner segment has `signatureFlag & 0x01 == 0x01`, `recover` recursively invokes `recoverChained`:

1. **Checkpoint Order Bypass (`WrongChainedCheckpointOrder`)**:
   - Suppose outer segment 0 (`Outer_0`) has `checkpoint = C_0` and `imageHash = H_0`. After outer iteration 0, `linkedPayload.imageHash = H_0` and the outer `prevCheckpoint = C_0`.
   - Suppose outer segment 1 is a nested chained signature consisting of two inner segments `[Inner_0, Inner_1]` with checkpoints `C_inner0` and `C_inner1`.
   - When line 171 calls `recover(linkedPayload, _signature[rindex:nrindex], true, checkpointer)`, the recursive `recoverChained` initializes a fresh local `uint256 prevCheckpoint = type(uint256).max` (line 153).
   - In inner iteration 0, `Inner_0` validates `linkedPayload` (approving `H_0` with checkpoint `C_0`), so the actual state transition in the configuration chain is `Inner_0 (C_inner0) -> Outer_0 (C_0)`. However, `Inner_0`'s checkpoint `C_inner0` is compared against the inner `prevCheckpoint` (`type(uint256).max`), **not** against the outer `prevCheckpoint` (`C_0`).
   - In inner iteration 1, `Inner_1` validates `ConfigUpdate(H_inner0)` and checks `C_inner1 < C_inner0`.
   - The inner `recoverChained` then returns only the final values `(threshold, weight, H_inner1, C_inner1, opHash)` to the outer `recoverChained`.
   - At line 183 of the outer `recoverChained`, `if (checkpoint >= prevCheckpoint)` compares `C_inner1` against `C_0` (`C_inner1 < C_0`), completely hiding `C_inner0`.
   - Consequently, a transition from a higher checkpoint `C_inner0` backwards to a lower checkpoint `C_0` (`C_inner0 >= C_0`) succeeds as long as the nested chain terminates at a configuration `Inner_1` with `C_inner1 < C_0` (e.g., `C_inner1 = 1 -> C_inner0 = 10 -> C_0 = 5`).

2. **Snapshot Tracking and Checkpointer Bypass (`UnusedSnapshot`)**:
   - When `recover` is called at lines 168 and 171 with `_ignoreCheckpointer = true`, a nested `recoverChained` receives a zero-initialized `Snapshot memory snapshot` (`snapshot.imageHash == bytes32(0)`). Any intermediate configuration (`Inner_0`) recovered inside the nested call is never checked against the outer `_snapshot.imageHash` at lines 179–181 (`if (_snapshot.imageHash == imageHash)`), causing valid snapshots present inside nested segments to go unrecognized and revert at line 191 (`revert UnusedSnapshot(_snapshot)`).
   - Conversely, if the outer chained signature omits the checkpointer flag (`0x40`), `_checkpointer` is `address(0)` and `_snapshot.imageHash` is `bytes32(0)`. Because lines 168 and 171 pass `_ignoreCheckpointer = true` and `checkpointer = address(0)`, the inner segment can set `0x40` to read the wallet's `_checkpointer` address (line 91) and reconstruct the wallet's on-chain `imageHash` (line 134) while skipping `ICheckpointer(_checkpointer).snapshotFor` (line 93) and bypassing lines 179–181 and 191–193.

3. **Empty Nested Chain Acceptance**:
   - If a chained signature payload has length 0 (`sigSize == 1` with byte `0x01`), `while (rindex < _signature.length)` does not execute, returning `threshold = 0` and `weight = 0`, which satisfies `weight < threshold` (`0 < 0` is false) at line 174 without validating any signature.

### Assumptions and preconditions

1. A Sequence wallet uses chained signatures (`KIND_CONFIG_UPDATE` links) across configuration updates.
2. Either a backward configuration link exists (such as repeating an earlier signer configuration with a lower checkpoint as described in `docs/CHAINED.md`) or a checkpointer module is configured on the wallet to invalidate older configurations.

### Proof of concept

Consider a wallet with initial on-chain configuration `Config_1` (`checkpoint = 1`, `imageHash = H_1`), an intermediate configuration `Config_2` (`checkpoint = 5`, `imageHash = H_2`, where signer `C` is authorized), and a newer configuration `Config_3` (`checkpoint = 10`, `imageHash = H_3`, where signer `C` is evicted):

1. Suppose `Config_1` (`checkpoint = 1`) signed a configuration update to `Config_3` (`checkpoint = 10`), and `Config_3` (`checkpoint = 10`) signed a configuration update to `Config_2` (`checkpoint = 5`, e.g., reverting to a prior configuration without incrementing the checkpoint).
2. In a flat chained signature `[Sig_Config2_Payload (cp=5), Sig_Config3_to_Config2 (cp=10), Sig_Config1_to_Config3 (cp=1)]`:
   - Iteration 0 recovers `Config_2` (`checkpoint = 5`), setting `prevCheckpoint = 5`.
   - Iteration 1 recovers `Config_3` (`checkpoint = 10`).
   - Line 183 checks `checkpoint >= prevCheckpoint` (`10 >= 5`), which reverts with `WrongChainedCheckpointOrder(10, 5)`.
3. Now, signer `C` frames the last two links as a nested chained signature (`signatureFlag = 0x01`):
   - Outer Segment 0: `Sig_Config2_Payload` (`checkpoint = 5`, signing `_payload`).
   - Outer Segment 1: A chained signature (`0x01`) containing `[Sig_Config3_to_Config2 (cp=10), Sig_Config1_to_Config3 (cp=1)]`.
4. Execution in `recoverChained`:
   - Outer Iteration 0 recovers `Config_2` (`imageHash = H_2`, `checkpoint = 5`). `linkedPayload.imageHash` is set to `H_2` and outer `prevCheckpoint` is set to `5`.
   - Outer Iteration 1 calls `recover(linkedPayload, Outer_Segment_1, true, checkpointer)`, which enters `recoverChained` with a fresh inner `prevCheckpoint = type(uint256).max`.
   - Inner Iteration 0 recovers `Config_3` (`imageHash = H_3`, `checkpoint = 10`) signing `linkedPayload` (`ConfigUpdate(H_2)`). Because `10 < type(uint256).max`, line 183 passes and inner `prevCheckpoint` becomes `10`.
   - Inner Iteration 1 recovers `Config_1` (`imageHash = H_1`, `checkpoint = 1`) signing `ConfigUpdate(H_3)`. Because `1 < 10`, line 183 passes and the inner `recoverChained` returns `(threshold, weight, H_1, checkpoint = 1, _)`.
   - Back in Outer Iteration 1, line 183 checks `checkpoint >= prevCheckpoint` using `1 >= 5`, which is false.
   - `recoverChained` returns `imageHash = H_1` (matching the wallet's on-chain `imageHash`), allowing evicted signer `C` at `checkpoint = 5` to authorize `_payload` despite the forbidden `10 -> 5` checkpoint transition.

### Impact assessment

**Severity: High**

Breaking the strict checkpoint ordering invariant (`WrongChainedCheckpointOrder`) and checkpointer snapshot enforcement (`UnusedSnapshot`) in `recoverChained` allows evicted signers from superseded wallet configurations to authorize arbitrary payloads on the wallet, leading to unauthorized transaction execution and loss of wallet funds.

### Remediation

Disallow nested chained signatures inside `recoverChained` (so that each link in a chained signature must be a non-chained signature), require non-empty chains in `recoverChained`, and prevent inner segments in `recoverChained` from overriding the wallet's final `_checkpointer` when `_ignoreCheckpointer` is set.

```solidity
  function recoverChained(
    Payload.Decoded memory _payload,
    address _checkpointer,
    Snapshot memory _snapshot,
    bytes calldata _signature
  ) internal view returns (uint256 threshold, uint256 weight, bytes32 imageHash, uint256 checkpoint, bytes32 opHash) {
    if (_signature.length == 0) {
      revert InvalidSignatureFlag(0x01);
    }

    Payload.Decoded memory linkedPayload;
    linkedPayload.kind = Payload.KIND_CONFIG_UPDATE;

    uint256 rindex;
    uint256 prevCheckpoint = type(uint256).max;

    while (rindex < _signature.length) {
      uint256 nrindex;

      {
        uint256 sigSize;
        (sigSize, rindex) = _signature.readUint24(rindex);
        nrindex = sigSize + rindex;
      }

      bool isLast = nrindex == _signature.length;
      address checkpointer = isLast ? _checkpointer : address(0);

      // Disallow nested chained signatures inside recoverChained
      if (uint8(_signature[rindex]) & 0x01 == 0x01) {
        revert InvalidSignatureFlag(uint8(_signature[rindex]));
      }

      // If this is the last link in the chain and top-level did not provide a checkpointer,
      // disallow setting the checkpointer flag while _ignoreCheckpointer is true
      if (isLast && checkpointer == address(0) && (uint8(_signature[rindex]) & 0x40 == 0x40)) {
        revert InvalidSignatureFlag(uint8(_signature[rindex]));
      }

      if (prevCheckpoint == type(uint256).max) {
        (threshold, weight, imageHash, checkpoint, opHash) =
          recover(_payload, _signature[rindex:nrindex], true, checkpointer);
      } else {
        (threshold, weight, imageHash, checkpoint,) =
          recover(linkedPayload, _signature[rindex:nrindex], true, checkpointer);
      }

      if (weight < threshold) {
        revert LowWeightChainedSignature(_signature[rindex:nrindex], threshold, weight);
      }
      rindex = nrindex;

      if (_snapshot.imageHash == imageHash) {
        _snapshot.imageHash = bytes32(0);
      }

      if (checkpoint >= prevCheckpoint) {
        revert WrongChainedCheckpointOrder(checkpoint, prevCheckpoint);
      }

      linkedPayload.imageHash = imageHash;
      prevCheckpoint = checkpoint;
    }

    if (_snapshot.imageHash != bytes32(0) && checkpoint <= _snapshot.checkpoint) {
      revert UnusedSnapshot(_snapshot);
    }
  }
```
