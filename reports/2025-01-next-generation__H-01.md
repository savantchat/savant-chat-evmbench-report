# Audit report — 2025-01-next-generation/H-01

## Finding 1: Rounding Down to Zero in `EURFToken` Transaction Fee Calculation Allows Fee Avoidance via Split Transfers

### Synopsis

In `EURFToken.transfer`, `EURFToken.transferFrom`, and `EURFToken.transferWithAuthorization`, transaction fees are settled via `transferSanity` and `_payTxFee` using `FeesHandlerUpgradeable.calculateTxFee`. Because `(txAmount * _txfeeRate) / FEE_RATIO` rounds down toward zero and zero-fee transfers are permitted, users can split transfers into small chunks to completely avoid paying transaction fees.

### Technical details

The `EURFToken` contract charges a transaction fee on every token transfer when `_txfeeRate > 0`. The three public transfer entrypoints in `contracts/Token.sol`—`transfer` (lines 161–164), `transferFrom` (lines 166–169), and `transferWithAuthorization` (lines 171–182)—each call `transferSanity(sender, recipient, amount)` (lines 156–159) prior to executing the underlying token transfer:

```solidity
function transferSanity(address sender, address recipient, uint256 amount) internal {
    adminSanity(sender, recipient);
    if (_txfeeRate > 0) _payTxFee(sender, amount);
}
```

When `_txfeeRate > 0`, `transferSanity` invokes `_payTxFee(address from, uint256 txAmount)` (lines 126–131 of `contracts/Token.sol`), which computes the fee via `FeesHandlerUpgradeable.calculateTxFee(uint256 txAmount)` (lines 52–54 of `contracts/FeesHandlerUpgradeable.sol`):

```solidity
function calculateTxFee(uint256 txAmount) public view returns (uint256) {
    return (txAmount * _txfeeRate) / FEE_RATIO;
}
```

Here, `FEE_RATIO` is `10000` (representing 100.00% in basis points). Because Solidity performs integer division with truncation toward zero, any transfer where `txAmount * _txfeeRate < FEE_RATIO` (i.e., `txAmount <= (FEE_RATIO - 1) / _txfeeRate`) results in `txFees == 0`.

In `_payTxFee`:
1. `calculateTxFee(txAmount)` returns `0`.
2. The balance check `if (balanceOf(from) < txFees + txAmount)` passes as long as `balanceOf(from) >= txAmount`.
3. `if (_feesFaucet != address(0)) _update(from, _feesFaucet, 0)` executes a zero-value transfer to `_feesFaucet` without reverting.
4. `emit FeesPaid(from, 0)` is emitted, and the principal `txAmount` is transferred to `recipient` in `super.transfer`, `super.transferFrom`, or `super.transferWithAuthorization`.

Because neither `calculateTxFee`, `_payTxFee`, `transferSanity`, nor the transfer entrypoints (`transfer`, `transferFrom`, `transferWithAuthorization`) round the fee up or require `txFees > 0` when `_txfeeRate > 0`, any user or batching contract can divide a transfer of arbitrary size into multiple smaller calls of `amount < FEE_RATIO / _txfeeRate` and transfer the entire balance while paying zero fees.

### Assumptions and preconditions

1. The `ADMIN` role has configured a non-zero transaction fee rate (`0 < _txfeeRate <= FEE_RATIO`) via `setTxFeeRate` and set `_feesFaucet` via `setFeeFaucet`.
2. The sender holds `EURFToken` tokens and initiates transfers directly or through a helper contract using `transfer`, `transferFrom`, or `transferWithAuthorization`.

### Proof of concept

1. Assume `ADMIN` sets `_txfeeRate = 100` (1% fee, i.e., `100 / 10000`) and sets `_feesFaucet` to a valid fee recipient address.
2. Bob holds `10,000` units of `EURF` and wants to transfer `1,000` units to Alice.
3. If Bob calls `transfer(alice, 1000)` in a single call:
   - `calculateTxFee(1000) = (1000 * 100) / 10000 = 10`.
   - Bob pays `10` units to `_feesFaucet` and `1,000` units to Alice (total `1,010` deducted from Bob).
4. Instead, Bob (or a helper contract approved by Bob calling `transferFrom(bob, alice, 99)` in a loop) executes 10 transfers of `99` units and 1 transfer of `10` units:
   - For each transfer of `99` units: `calculateTxFee(99) = (99 * 100) / 10000 = 9900 / 10000 = 0`.
   - For the transfer of `10` units: `calculateTxFee(10) = (10 * 100) / 10000 = 1000 / 10000 = 0`.
5. Alice receives the full `1,000` units (`10 * 99 + 10`), `_feesFaucet` receives `0` units, and Bob completely bypasses the 1% transaction fee.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: Complete bypass of the `_txfeeRate` fee mechanism across `transfer`, `transferFrom`, and `transferWithAuthorization`, as well as emission of misleading `FeesPaid(from, 0)` and zero-value `Transfer(from, _feesFaucet, 0)` events.
- **Business Impact**: Loss of protocol fee revenue intended to sustain the EURF ecosystem. With realistic stablecoin fee rates (e.g., `_txfeeRate = 1` to `10` bps), the zero-fee threshold reaches up to `9,999` units per call, making automated batch splitting via smart contracts economically viable even after gas costs (especially on L2/sidechains like Polygon where EURF is deployed).

### Remediation

Round the fee calculation up (ceiling division) in `calculateTxFee` when `txAmount > 0 && _txfeeRate > 0`, or enforce a minimum fee / minimum transfer amount in `_payTxFee` / `transferSanity` so that non-zero transfers under an active `_txfeeRate` always incur at least 1 unit of fee.

```solidity
    function calculateTxFee(uint256 txAmount) public view returns (uint256) {
        if (txAmount == 0 || _txfeeRate == 0) return 0;
        return ((txAmount * _txfeeRate) + FEE_RATIO - 1) / FEE_RATIO;
    }
```

Additionally, avoid executing `_update` and emitting `FeesPaid` when `txFees == 0`:

```solidity
    function _payTxFee(address from, uint256 txAmount) internal override {
        uint256 txFees = calculateTxFee(txAmount);
        if (balanceOf(from) < txFees + txAmount) revert BalanceTooLow(txFees + txAmount, balanceOf(from));
        if (txFees > 0 && _feesFaucet != address(0)) {
            _update(from, _feesFaucet, txFees);
            emit FeesPaid(from, txFees);
        }
    }
```

## Finding 2: Missing Expiration Deadline and Nonce Cancellation in `Forwarder.execute` Leaves Signed Meta-Transactions Permanently Deliverable

### Synopsis

`Forwarder.execute` verifies and executes signed `ForwardRequest` payloads without enforcing any expiration timestamp (`deadline`) and without providing a nonce invalidation mechanism. Consequently, any unexecuted or previously reverted signed transfer request remains permanently valid until the signer's forwarder nonce is consumed by another forwarded transaction.

### Technical details

In `contracts/Forwarder.sol`, the `ForwardRequest` struct and `execute` function define and validate gasless meta-transactions for `EURFToken.transfer`:

```solidity
struct ForwardRequest {
    address from;
    address to;
    uint256 value;
    uint256 gas;
    uint256 nonce;
    bytes data;
}
```

When `Forwarder.execute` (lines 93–118) is called, it performs:
1. `_verifyNonce(req)` (line 100), checking `nonces[req.from] == req.nonce`.
2. `_verifySig(req, domainSeparator, requestTypeHash, suffixData, sig)` (line 101), recovering the signer of the EIP-712 payload.
3. `_updateNonce(req)` (line 102), incrementing `nonces[req.from]`.
4. Forwarding the `transfer(address,uint256)` call to `_eurfAddress` (line 112) and charging `_gaslessBasefee` via `_eurf.payGaslessBasefee(req.from, _msgSender())` (line 115).

Unlike `EURFToken.transferWithAuthorization` (in `contracts/Token.sol`, lines 171–182), which enforces a signed `deadline` parameter, `ForwardRequest` and `Forwarder.execute` include no expiration timestamp (`deadline` or `validUntilTime`). Furthermore, `Forwarder` exposes no function allowing `req.from` to increment `nonces[req.from]` directly to cancel a pending signature, and performing a direct on-chain `EURFToken.transfer` does not increment `Forwarder.nonces[req.from]`.

This creates an authorization lifetime and invalidation gap:
- If a signed `ForwardRequest` fails on-chain (for example, because `req.from` temporarily lacks sufficient balance to cover `amount + txFees + _gaslessBasefee`, or while the token is paused), the entire `execute` call reverts and `nonces[req.from]` remains unchanged. The signed payload remains valid off-chain and can be re-submitted by any relayer or recipient once `req.from` acquires more tokens.
- If a relayer withholds a signed `ForwardRequest`, the signer cannot revoke it directly; the only way to consume the nonce is to execute another gasless `transfer` through `Forwarder.execute` (which incurs `_gaslessBasefee` and `_txfeeRate` and is impossible if the user currently lacks the balance required for fees).
- If `ADMIN` updates `_txfeeRate` or `_gaslessBasefee` before a delayed `ForwardRequest` is executed, the stale signature can still be executed under the new fee parameters that the signer did not anticipate when signing.

### Assumptions and preconditions

- A token holder (`req.from`) signs a `ForwardRequest` off-chain to transfer `EURFToken` via `Forwarder.execute`.
- The signed request is either not immediately executed by the relayer or initially reverts on-chain (e.g., due to temporary insufficient balance in `req.from`).
- The signer has not executed a different `ForwardRequest` with the same `nonce` through `Forwarder.execute`.

### Proof of concept

1. Bob holds `1,000` EURF (`1000 * 10^6` units) and signs a `ForwardRequest` with `nonce = 0` to transfer `1,001` EURF to Alice via `Forwarder`.
2. The relayer (or Alice) submits `Forwarder.execute(req, ...)`. Because Bob's balance is `1,000 < 1,001`, the inner `transfer` call reverts, causing `require(success, "NGEUR Forwarder: failed tx execution")` on line 113 to revert the transaction. Bob's `nonces[bob]` remains `0`.
3. Bob considers the payment failed and instead pays Alice via a regular on-chain `EURFToken.transfer(alice, 500)`. Because direct transfers do not touch `Forwarder.nonces`, `nonces[bob]` is still `0`.
4. Later, Bob receives `1,000` EURF from another party, bringing his balance to `1,500` EURF.
5. Alice (or any observer who saw the reverted transaction in step 2) calls `Forwarder.execute` with Bob's original signed `ForwardRequest` (`nonce = 0`).
6. `_verifyNonce` and `_verifySig` pass because `nonces[bob] == 0` and no expiration timestamp exists. `Forwarder.execute` transfers an additional `1,001` EURF from Bob to Alice and deducts `_gaslessBasefee` from Bob.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: Signed meta-transactions cannot expire on their own and cannot be directly invalidated without executing a competing fee-bearing forwarded transfer. Reverted or withheld meta-transactions can be replayed at any future block when account balances or fee rates change.
- **Business Impact**: Users may suffer unexpected duplicate transfers or higher-than-expected fee deductions from stale off-chain signatures that they reasonably believed had failed or expired.

### Remediation

Include a `deadline` (or `validUntilTime`) parameter in `ForwardRequest` so that every signature is bound to an explicit expiration timestamp verified in `Forwarder.execute` (and `Forwarder.verify`), and optionally expose a nonce invalidation function for signers.

```solidity
    function execute(
        ForwardRequest calldata req,
        bytes32 domainSeparator,
        bytes32 requestTypeHash,
        bytes calldata suffixData,
        bytes calldata sig
    ) external payable returns (bool success, bytes memory ret) {
        require(req.deadline == 0 || req.deadline >= block.timestamp, "NGEUR Forwarder: request expired");
        _verifyNonce(req);
        _verifySig(req, domainSeparator, requestTypeHash, suffixData, sig);
        _updateNonce(req);

        require(req.to == _eurfAddress, "NGEUR Forwarder: can only forward NGEUR transactions");

        bytes4 transferSelector = bytes4(keccak256("transfer(address,uint256)"));
        bytes4 reqTransferSelector = bytes4(req.data[:4]);

        require(reqTransferSelector == transferSelector, "NGEUR Forwarder: can only forward transfer transactions");

        // solhint-disable-next-line avoid-low-level-calls
        (success, ret) = req.to.call{gas: req.gas, value: req.value}(abi.encodePacked(req.data, req.from));
        require(success, "NGEUR Forwarder: failed tx execution");

        _eurf.payGaslessBasefee(req.from, _msgSender());

        return (success, ret);
    }
```

Ensure `uint256 deadline` is also added to `ForwardRequest`, `GENERIC_PARAMS`, and `_getEncoded`.

## Finding 3: Missing Calldata Length Validation in `Forwarder.execute` Causes Appended Sender Address to Overlap with ABI-Decoded `transfer` Amount

### Synopsis

In `Forwarder.execute`, `req.data` is only checked for the 4-byte `transfer(address,uint256)` selector before appending the 20-byte `req.from` address via `abi.encodePacked(req.data, req.from)`. When `req.data` is between 48 and 67 bytes, `EURFToken.transfer` ABI-decodes part or all of the appended `req.from` suffix as the transfer `amount`.

### Technical details

In `contracts/Forwarder.sol`, `execute` validates that the target is `_eurfAddress` and that the first 4 bytes of `req.data` match the `transfer(address,uint256)` function selector (`0xa9059cbb`), then forwards the call using ERC-2771 context packing:

```solidity
bytes4 transferSelector = bytes4(keccak256("transfer(address,uint256)"));
bytes4 reqTransferSelector = bytes4(req.data[:4]);

require(reqTransferSelector == transferSelector, "NGEUR Forwarder: can only forward transfer transactions");

// solhint-disable-next-line avoid-low-level-calls
(success, ret) = req.to.call{gas: req.gas, value: req.value}(abi.encodePacked(req.data, req.from));
```

A standard ABI-encoded call to `transfer(address recipient, uint256 amount)` requires `4 + 32 + 32 = 68` bytes in `req.data`, resulting in `68 + 20 = 88` bytes of calldata after `req.from` (20 bytes) is appended by `abi.encodePacked(req.data, req.from)`.

However, neither `Forwarder.execute` nor `EURFToken` verifies that `req.data.length == 68` (or `msg.data.length >= 88`):
1. Solidity's generated ABI dispatcher for `EURFToken.transfer(address,uint256)` only verifies that `msg.data.length >= 68`, decoding `recipient` from `msg.data[4:36]` and `amount` from `msg.data[36:68]`.
2. `ERC20MetaTxUpgradeable._msgSender()` extracts the ERC-2771 sender from the final 20 bytes of `msg.data` (`msg.data[msg.data.length - 20 : msg.data.length]`).

If `req.data` has a length `L` where `48 <= L < 68`, the forwarded calldata `abi.encodePacked(req.data, req.from)` has length `L + 20 >= 68`. The ABI decoder in `EURFToken.transfer` succeeds and reads the `amount` word from `msg.data[36:68]`. Because `req.data` ends at byte offset `L < 68`, bytes `[L:68]` of the `amount` word are taken directly from the first `68 - L` bytes of the appended `req.from` address, while `_msgSender()` simultaneously reads `msg.data[L:L+20]` as `req.from`.

### Assumptions and preconditions

1. `Forwarder` is configured as the trusted forwarder on `EURFToken` via `setTrustedForwarder`.
2. A signer (`req.from`) holding `EURFToken` tokens signs a `ForwardRequest` whose `req.data` has length `L` in the range `48 <= L < 68` (for instance, a truncated calldata payload containing the 4-byte `transfer` selector, 32-byte `recipient`, and `L - 36` zero bytes where off-chain tooling sees only zero bytes in the `amount` slice of `req.data`).

### Proof of concept

1. Suppose `req.from` (`0x1000000000000000000000000000000000000001`) holds `1,000e6` EURF tokens (`1,000,000,000` base units).
2. A `ForwardRequest` is constructed with `req.to = address(eurfToken)` and a 65-byte `req.data` consisting of:
   - Bytes `0..4`: `0xa9059cbb` (`transfer(address,uint256)` selector)
   - Bytes `4..36`: `abi.encode(recipient)`
   - Bytes `36..65`: 29 zero bytes (`0x00...00`)
3. In `req.data`, all bytes after `recipient` are `0x00`.
4. When `Forwarder.execute` is called, it checks `bytes4(req.data[:4]) == 0xa9059cbb` (which passes) and executes `req.to.call{gas: req.gas, value: req.value}(abi.encodePacked(req.data, req.from))`.
5. The calldata received by `EURFToken` is `65 + 20 = 85` bytes (`>= 68` bytes):
   - `msg.data[4:36]` decodes to `recipient`.
   - `msg.data[36:68]` consists of the 29 zero bytes from `req.data[36:65]` followed by the first 3 bytes of `req.from` (`0x100000`), which ABI-decodes to `amount = 0x100000 = 1,048,576` base units (`1.048576` EURF).
   - `msg.data[65:85]` is read by `_msgSender()` as `req.from`.
6. `EURFToken.transfer` transfers `1,048,576` base units from `req.from` to `recipient` even though `req.data` contained only zero bytes after the `recipient` parameter.

### Impact assessment

- **Severity**: Medium
- **Technical Effect**: The ERC-2771 context suffix (`req.from`) is partially or wholly interpreted as the `uint256 amount` parameter of `EURFToken.transfer(address,uint256)` whenever `48 <= req.data.length < 68`.
- **Business Impact**: Any user signing a `ForwardRequest` with truncated `req.data` (where all explicit bytes in the `amount` position are zero) will unexpectedly transfer a non-zero amount derived from the high bytes of their own address to `recipient`, leading to unauthorized loss of user funds.

### Remediation

Validate in `Forwarder.execute` (and `Forwarder.verify`) that `req.data.length == 68` so that `req.data` contains a complete 4-byte selector, 32-byte `recipient`, and 32-byte `amount` before `req.from` is appended.

```solidity
    function execute(
        ForwardRequest calldata req,
        bytes32 domainSeparator,
        bytes32 requestTypeHash,
        bytes calldata suffixData,
        bytes calldata sig
    ) external payable returns (bool success, bytes memory ret) {
        _verifyNonce(req);
        _verifySig(req, domainSeparator, requestTypeHash, suffixData, sig);
        _updateNonce(req);

        require(req.to == _eurfAddress, "NGEUR Forwarder: can only forward NGEUR transactions");
        require(req.data.length == 68, "NGEUR Forwarder: invalid transfer calldata length");

        bytes4 transferSelector = bytes4(keccak256("transfer(address,uint256)"));
        bytes4 reqTransferSelector = bytes4(req.data[:4]);

        require(reqTransferSelector == transferSelector, "NGEUR Forwarder: can only forward transfer transactions");

        // solhint-disable-next-line avoid-low-level-calls
        (success, ret) = req.to.call{gas: req.gas, value: req.value}(abi.encodePacked(req.data, req.from));
        require(success, "NGEUR Forwarder: failed tx execution");

        _eurf.payGaslessBasefee(req.from, _msgSender());

        return (success, ret);
    }
```

## Finding 4: Revert on Failed Forwarded Call in `Forwarder.execute` Rolls Back Nonce Consumption, Enabling Stale Meta-Transaction Replay and Paymaster Griefing

### Synopsis

In `Forwarder.execute`, calling `require(success)` after the low-level forwarded call reverts the entire transaction upon inner-call failure, rolling back `_updateNonce(req)` and bypassing `payGaslessBasefee`. Because `ForwardRequest` lacks an expiration deadline, failed meta-transactions remain indefinitely replayable and allow signers to grief paymasters without consuming nonces.

### Technical details

In `contracts/Forwarder.sol`, `execute` processes signed `ForwardRequest` structs to perform gasless `EURFToken.transfer` operations on behalf of `req.from`:

```solidity
_verifyNonce(req);
_verifySig(req, domainSeparator, requestTypeHash, suffixData, sig);
_updateNonce(req);
...
(success, ret) = req.to.call{gas: req.gas, value: req.value}(abi.encodePacked(req.data, req.from));
require(success, "NGEUR Forwarder: failed tx execution");

_eurf.payGaslessBasefee(req.from, _msgSender());

return (success, ret);
```

Although `_updateNonce(req)` (line 125) increments `nonces[req.from]` prior to the external call and `execute` is declared to return `(bool success, bytes memory ret)` (line 117), line 113 enforces `require(success, "NGEUR Forwarder: failed tx execution")`.

This creates a critical inconsistency in nonce consumption and execution order:

1. **Unconsumed Nonce and Indefinite Replay of Failed Requests:**
   `ForwardRequest` (`from, to, value, gas, nonce, data`) does not include a `deadline` field, and `Forwarder` provides no function for a user to cancel or increment `nonces[req.from]` directly. If a user signs a transfer request that fails during execution (for instance, because `req.from`'s balance is temporarily below `amount + txFees + _gaslessBasefee`, the token contract is paused, or the recipient is temporarily blacklisted), line 113 reverts the entire transaction. Consequently, the nonce increment in `_updateNonce(req)` is rolled back. The signed payload remains valid and unconsumed, allowing any actor or relayer to replay the stale signature later once `req.from` acquires sufficient tokens or the pause/blacklist is lifted.

2. **Paymaster Gas Griefing via Nonce Rollback:**
   Because `_eurf.payGaslessBasefee(req.from, _msgSender())` (line 115) is executed after `req.to.call` and rolled back if `require(success)` reverts, a signer `req.from` can pass off-chain `verify()` checks and then front-run the paymaster's `execute` transaction by transferring part of their `EURFToken` balance away. The inner `transfer` reverts with `BalanceTooLow` or `ERC20InsufficientBalance`, causing `require(success)` on line 113 to revert the outer call. The paymaster pays the transaction gas without receiving `_gaslessBasefee`, and `nonces[req.from]` is not incremented, leaving the same signature reusable to repeat the griefing attack.

### Assumptions and preconditions

- `Forwarder` is configured as the `_trustedForwarder` on `EURFToken`.
- A user (`req.from`) signs a valid `ForwardRequest` for `EURFToken.transfer(recipient, amount)` with their current nonce `nonces[req.from]`.
- Either the inner `EURFToken.transfer` fails when first submitted (e.g., due to insufficient balance, pause, or blacklist state) or `req.from` front-runs the relayer's `execute` call to reduce their balance before execution.

### Proof of concept

1. Alice holds `1,000` EURF (with `_gaslessBasefee = 10` and `_txfeeRate = 0`).
2. Alice signs a `ForwardRequest` with `nonce = 0` to transfer `1,000` EURF to Bob (`req.data = abi.encodeWithSelector(IERC20.transfer.selector, bob, 1000)`).
3. Off-chain, the relayer calls `Forwarder.verify(...)`, which succeeds because `verify()` only checks `_verifyNonce`, `_verifySig`, and the `transfer` selector.
4. The relayer broadcasts `Forwarder.execute(...)`.
5. Either:
   - Alice's total required balance is `1,000 + 10 = 1,010` EURF (or Alice front-runs the relayer with a direct transfer of `100` EURF so her balance is `900` EURF).
   - Inside `execute()`, `_updateNonce(req)` sets `nonces[alice] = 1`.
   - When `req.to.call` fails (or `payGaslessBasefee` reverts), the entire transaction reverts at line 113 (or line 115), rolling `nonces[alice]` back to `0` and paying the relayer `0` EURF.
6. Later, Alice receives `2,000` EURF and considers the earlier failed payment abandoned. Because `nonces[alice]` is still `0` and `ForwardRequest` has no deadline, Bob (or any relayer) replays the original signature from Step 2 via `Forwarder.execute(...)`, transferring `1,000` EURF from Alice to Bob against her current intent.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Rollback of `_updateNonce(req)` on inner-call failure prevents nonce consumption for failed meta-transactions, leaving non-expiring signatures permanently replayable and allowing signers to waste paymaster gas without paying `_gaslessBasefee`.
- **Business Impact:** Users cannot safely abandon failed gasless transfers without executing a replacement meta-transaction at the same nonce, and relayers/paymasters operating the gasless infrastructure are exposed to repeated uncompensated gas exhaustion.

### Remediation

Collect the gasless base fee from `req.from` and allow the nonce increment in `_updateNonce(req)` to persist even when the inner `req.to.call` fails (while ensuring the relayer forwarded sufficient gas per EIP-150 so the relayer cannot intentionally force an out-of-gas failure in the subcall):

```solidity
        _eurf.payGaslessBasefee(req.from, _msgSender());

        // solhint-disable-next-line avoid-low-level-calls
        (success, ret) = req.to.call{gas: req.gas, value: req.value}(abi.encodePacked(req.data, req.from));
        require(gasleft() > req.gas / 63, "NGEUR Forwarder: insufficient gas for forwarded call");

        return (success, ret);
```

## Finding 5: Malformed EIP-712 Type String Definition in `Forwarder.registerRequestType` Breaks Custom Request Type Registration

### Synopsis

In `Forwarder.registerRequestType`, the EIP-712 `requestType` string is constructed via `abi.encodePacked(typeName, "(", GENERIC_PARAMS, ",", typeSuffix)` without a closing parenthesis `")"` or validation of `typeSuffix`. This produces a malformed EIP-712 type schema hash that is incompatible with standard EIP-712 signers and typed-data encoders.

### Technical details

The `Forwarder` contract implements EIP-712 signature verification for meta-transactions. During `initialize` (line 71), the default request type is registered by enclosing `GENERIC_PARAMS` with a closing parenthesis:

```solidity
string memory requestType = string(abi.encodePacked("ForwardRequest(", GENERIC_PARAMS, ")"));
```

However, in `registerRequestType` (lines 128–136), when the owner registers an extended or custom request type, the function constructs `requestType` at line 134 as:

```solidity
string memory requestType = string(abi.encodePacked(typeName, "(", GENERIC_PARAMS, ",", typeSuffix));
```

This implementation has the following schema integrity flaws:
1. **Missing Closing Parenthesis (`")"`):** Unlike `initialize` (and `GENERIC_PARAMS`, which contains only comma-separated field declarations `"address from,address to,uint256 value,uint256 gas,uint256 nonce,bytes data"`), line 134 omits the closing `")"` at the end of `abi.encodePacked`. If a caller passes a field declaration list in `typeSuffix` (consistent with `GENERIC_PARAMS`), the resulting type string is `<typeName>(<GENERIC_PARAMS>,<typeSuffix>`, which is syntactically invalid under EIP-712.
2. **Unvalidated `typeSuffix` and Trailing Comma:** Unlike `typeName` (which is checked for `"("` and `")"` in lines 129–132), `typeSuffix` is not validated at all, and an unconditionally prepended `","` means an empty `typeSuffix` results in a trailing comma (`<typeName>(<GENERIC_PARAMS>,`), while arbitrary parentheses in `typeSuffix` can produce ambiguous or malformed EIP-712 type definitions. Additionally, any signed extension fields defined in `typeSuffix` and passed via `suffixData` in `_verifySig` / `_getEncoded` are never decoded or enforced during `execute`.

Because EIP-712 compliant wallets (e.g., MetaMask `eth_signTypedData_v4`) strictly validate and canonicalize the `types` schema (`TypeName(type1 name1,type2 name2,...)`), a type hash computed from a malformed type string without a closing parenthesis cannot be matched by standard EIP-712 signing tools, causing `_verifySig` to revert with `"NGEUR Forwarder: signature mismatch"` or `"NGEUR Forwarder: invalid request typehash"`.

### Assumptions and preconditions

1. The `Forwarder` contract is deployed and initialized.
2. The contract `owner` calls `registerRequestType(typeName, typeSuffix)` providing the extension parameter definition in `typeSuffix` (consistent with the format of `GENERIC_PARAMS`).
3. A user signs a typed message using a standard EIP-712 wallet (such as `eth_signTypedData_v4`), which computes `requestTypeHash` over the canonical type string `<typeName>(<GENERIC_PARAMS>,<typeSuffix>)` (with a closing parenthesis).

### Proof of concept

1. The owner calls `forwarder.registerRequestType("CustomForwardRequest", "uint256 deadline")` to register an extended request type.
2. Line 134 constructs `requestType` as:
   `"CustomForwardRequest(address from,address to,uint256 value,uint256 gas,uint256 nonce,bytes data,uint256 deadline"` (missing the trailing `")"`).
3. `registerRequestTypeInternal` stores `typeHashes[keccak256(bytes(requestType))] = true` for the malformed string.
4. Off-chain, a user signs the `CustomForwardRequest` struct using an EIP-712 compliant signer, which hashes the canonical type string `"CustomForwardRequest(address from,address to,uint256 value,uint256 gas,uint256 nonce,bytes data,uint256 deadline)"` (including `")"`).
5. When the relayer calls `forwarder.execute(req, domainSeparator, requestTypeHash, suffixData, sig)`:
   - If the relayer passes the canonical `requestTypeHash` (with `")"`), `_verifySig` reverts at line 151 with `"NGEUR Forwarder: invalid request typehash"`.
   - If the relayer passes the registered malformed `requestTypeHash` (without `")"`), the digest calculated at lines 152–154 differs from the EIP-712 digest signed by the user's wallet, and `_verifySig` reverts at line 155 with `"NGEUR Forwarder: signature mismatch"`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Any request type registered via `registerRequestType` with standard parameter definitions produces a malformed EIP-712 type string and broken `typeHash`, rendering custom request types unusable with EIP-712 compliant signers.
- **Business Impact:** Protocol upgrades or extensions that rely on `registerRequestType` to introduce additional signed metadata fields for gasless EURF transactions will fail signature verification in production wallets.

### Remediation

Validate `typeSuffix` to ensure it does not contain parentheses and properly handle both empty and non-empty `typeSuffix` values while appending the closing `")"` to the encoded `requestType` string:

```solidity
    function registerRequestType(string calldata typeName, string calldata typeSuffix) external onlyOwner {
        for (uint i = 0; i < bytes(typeName).length; i++) {
            bytes1 c = bytes(typeName)[i];
            require(c != "(" && c != ")", "NGEUR Forwarder: invalid typename");
        }
        for (uint i = 0; i < bytes(typeSuffix).length; i++) {
            bytes1 c = bytes(typeSuffix)[i];
            require(c != "(" && c != ")", "NGEUR Forwarder: invalid typesuffix");
        }

        string memory requestType = bytes(typeSuffix).length > 0
            ? string(abi.encodePacked(typeName, "(", GENERIC_PARAMS, ",", typeSuffix, ")"))
            : string(abi.encodePacked(typeName, "(", GENERIC_PARAMS, ")"));
        registerRequestTypeInternal(requestType);
    }
```

## Finding 6: Unchecked `domainSeparator` in `Forwarder._verifySig` Enables Cross-Chain and Cross-Contract Signature Replay

### Synopsis

`Forwarder._verifySig` accepts an arbitrary caller-supplied `domainSeparator` parameter without validating it against a computed EIP-712 domain separator or a registry of valid domains. This allows attackers to replay signed `ForwardRequest` meta-transactions across different chains, chain forks, or redeployed `Forwarder` instances, leading to unauthorized token transfers and fee deductions.

### Technical details

In `Forwarder.sol`, both `verify` and `execute` accept a `bytes32 domainSeparator` argument directly from the caller and forward it to `_verifySig` (lines 144–156):

```solidity
function _verifySig(
    ForwardRequest memory req,
    bytes32 domainSeparator,
    bytes32 requestTypeHash,
    bytes memory suffixData,
    bytes memory sig
) internal view {
    require(typeHashes[requestTypeHash], "NGEUR Forwarder: invalid request typehash");
    bytes32 digest = keccak256(
        abi.encodePacked("\x19\x01", domainSeparator, keccak256(_getEncoded(req, requestTypeHash, suffixData)))
    );
    require(digest.recover(sig) == req.from, "NGEUR Forwarder: signature mismatch");
}
```

While `_verifySig` checks that `requestTypeHash` is registered in `typeHashes` (line 151), it performs no validation on `domainSeparator`. In a standard EIP-2770 / GSN forwarder or EIP-712 implementation, the contract either:
1. Computes the `domainSeparator` dynamically on-chain using `block.chainid` and `address(this)` (e.g., via OpenZeppelin's `EIP712Upgradeable._domainSeparatorV4()`), or
2. Validates `require(domains[domainSeparator], "Invalid domain separator")` where registered domains bind `block.chainid` and `verifyingContract` (`address(this)`).

Because `domainSeparator` is completely unverified and `ForwardRequest` only binds `from`, `to`, `value`, `gas`, `nonce`, and `data` (lines 158–168), the EIP-712 domain isolation guarantees (`chainId` and `verifyingContract`) are bypassed:

1. **Cross-Contract Replay (New Forwarder Deployment):** If the protocol administrator deploys and sets a new `Forwarder` contract as the `_trustedForwarder` on `EURFToken` (as supported by `setTrustedForwarder`), all `nonces` in the new `Forwarder` start at `0`. Any previously executed meta-transaction from the old `Forwarder` still has `req.to == _eurfAddress`, and an attacker can pass the old `domainSeparator` and signature to the new `Forwarder.execute()` to replay all historical transfers of `req.from` starting from nonce `0`.
2. **Cross-Chain / Chain-Fork Replay:** As stated in the specification, `EURF` is designed to be released on Ethereum, Polygon, and other EVM chains. If `EURFToken` is deployed at the same address across multiple chains (e.g., via CREATE2 or identical deployer nonces) or in the event of a chain fork, a signature and `domainSeparator` created for Chain A can be submitted directly to `Forwarder.execute()` on Chain B, bypassing `chainId` and `verifyingContract` checks entirely.

### Assumptions and preconditions

1. A user (`req.from`) has signed at least one valid `ForwardRequest` to transfer `EURFToken` (`req.to == _eurfAddress`).
2. Either:
   - The `Forwarder` contract is replaced by a newly deployed `Forwarder` instance via `setTrustedForwarder`, resetting user nonces to `0` on the new contract; or
   - `EURFToken` exists at the same address on another EVM chain (or a chain fork occurs) where the user also holds `EURFToken` and has the same nonce on that chain's `Forwarder`.

### Proof of concept

1. User `bob` signs a `ForwardRequest` with `nonce = 0` to transfer `50 EURF` to `alice` on Chain A (or Forwarder Instance 1), using `domainSeparator_1` (which encodes Chain A's `chainId` and Forwarder 1's address).
2. `forwardOperator` submits `execute(req, domainSeparator_1, requestTypeHash, suffixData, sig)` on Forwarder 1. The transaction succeeds and `bob` transfers `50 EURF` to `alice` (plus `_gaslessBasefee` to `forwardOperator`).
3. Later, the admin deploys Forwarder 2 and calls `eurfToken.setTrustedForwarder(forwarder2)` (or `EURFToken` and `Forwarder` are active on Chain B / a chain fork where `eurfToken` has the same address).
4. An attacker (or `alice` / a malicious relayer) calls `forwarder2.execute(req, domainSeparator_1, requestTypeHash, suffixData, sig)`.
5. Inside `forwarder2._verifySig`:
   - `typeHashes[requestTypeHash]` is `true` (initialized in `initialize`).
   - `digest` is computed using the caller-supplied `domainSeparator_1`.
   - `digest.recover(sig)` returns `bob` (`req.from`).
6. Because `forwarder2.nonces[bob]` is `0`, `_verifyNonce(req)` passes, and the transfer of `50 EURF` from `bob` to `alice` (and `_gaslessBasefee` to the attacker `msg.sender`) is executed a second time without `bob`'s authorization.

### Impact assessment

- **Severity:** High
- **Technical Effect:** Complete bypass of EIP-712 domain separation (`chainId` and `verifyingContract`) in `Forwarder`.
- **Business Impact:** Unauthorized replay of token transfers (`transfer`) and gasless base fee payments (`payGaslessBasefee`), resulting in direct theft/loss of user funds (`EURF` tokens) across chains, chain forks, or forwarder upgrades/migrations.

### Remediation

Inherit OpenZeppelin's `EIP712Upgradeable` in `Forwarder` and compute the domain separator on-chain using `_domainSeparatorV4()` (which binds `block.chainid` and `address(this)`), or maintain a mapping of registered `domains` validated in `_verifySig`.

Preferred solution using `EIP712Upgradeable` (or validating against an on-chain computed domain separator):

```solidity
    function _verifySig(
        ForwardRequest memory req,
        bytes32 domainSeparator,
        bytes32 requestTypeHash,
        bytes memory suffixData,
        bytes memory sig
    ) internal view {
        require(domainSeparator == _domainSeparatorV4(), "NGEUR Forwarder: invalid domain separator");
        require(typeHashes[requestTypeHash], "NGEUR Forwarder: invalid request typehash");
        bytes32 digest = keccak256(
            abi.encodePacked("\x19\x01", domainSeparator, keccak256(_getEncoded(req, requestTypeHash, suffixData)))
        );
        require(digest.recover(sig) == req.from, "NGEUR Forwarder: signature mismatch");
    }
```

## Finding 7: Missing `adminSanity` check in `EURFToken.payGaslessBasefee` allows blacklisted accounts to receive tokens as paymaster

### Synopsis

`EURFToken.payGaslessBasefee` directly invokes `_update(payer, paymaster, _gaslessBasefee)` without performing the `adminSanity` compliance check on `payer` and `paymaster`. Consequently, a blacklisted address can act as the `paymaster` (`msg.sender`) in `Forwarder.execute` and receive `EURF` tokens, bypassing recipient blacklist restrictions.

### Technical details

In `EURFToken` (`contracts/Token.sol`), token transfer restrictions (pause state, sender/recipient blacklisting, and transfers to the token contract itself) are enforced through `adminSanity(address from, address to)` in `ERC20AdminUpgradeable` (`contracts/ERC20AdminUpgradeable.sol` lines 75–82):

```solidity
function adminSanity(address from, address to) internal view {
    if (!hasRole(ADMIN, _msgSender())) {
        if (paused()) revert PausedError();
        if (isBlacklisted(from)) revert SenderBlacklistedError(from);
    }
    if (isBlacklisted(to)) revert RecipientBlacklistedError(to);
    if (to == address(this)) revert TransferToContractError();
}
```

Standard transfers (`transfer`, `transferFrom`, `transferWithAuthorization`) call `transferSanity`, which invokes `adminSanity(sender, recipient)` before calling `_update`.

However, when a gasless meta-transaction is executed via `Forwarder.execute` (`contracts/Forwarder.sol` lines 93–118), the forwarder first executes the user's `transfer` call and then calls:

```solidity
_eurf.payGaslessBasefee(req.from, _msgSender());
```

In `EURFToken.payGaslessBasefee` (`contracts/Token.sol` lines 147–152):

```solidity
function payGaslessBasefee(address payer, address paymaster) external override {
    if (!isTrustedForwarder(msg.sender)) revert InvalidForwarder();
    if (balanceOf(payer) < _gaslessBasefee) revert BalanceTooLow(_gaslessBasefee, balanceOf(payer));
    _update(payer, paymaster, _gaslessBasefee);
    emit GaslessBasefeePaid(payer, paymaster, _gaslessBasefee);
}
```

`payGaslessBasefee` directly calls OpenZeppelin's internal `_update(payer, paymaster, _gaslessBasefee)` on line 150 without calling `adminSanity(payer, paymaster)` (or otherwise verifying `!isBlacklisted(paymaster)`). Because `Forwarder.execute` is permissionless and sets `paymaster` to `_msgSender()`, any blacklisted account can submit a valid `ForwardRequest` (such as a zero-value self-transfer signed by a non-blacklisted accomplice account) to `Forwarder.execute` and receive `_gaslessBasefee` tokens per transaction despite being blacklisted.

### Assumptions and preconditions

1. The `Forwarder` contract is deployed and configured as the trusted forwarder in `EURFToken` via `setTrustedForwarder`.
2. `_gaslessBasefee` is set to a non-zero value by the `ADMIN`.
3. An address `paymaster` has been blacklisted by the `ADMIN` via `blacklist(paymaster)`.
4. A non-blacklisted `payer` account holds at least `_gaslessBasefee` EURF tokens and signs a valid `ForwardRequest` for `transfer` (either as an accomplice to transfer funds to the blacklisted `paymaster`, or as a regular user whose signed meta-transaction is front-run/relayed by the blacklisted `paymaster`).

### Proof of concept

1. Suppose `blacklistedUser` is blacklisted (`isBlacklisted(blacklistedUser) == true`) and `_gaslessBasefee` is `10e6` (10 EURF).
2. A non-blacklisted account `alice` (either a colluding account controlled by the same entity or an ordinary user) holds `100e6` EURF and signs a `ForwardRequest` calling `EURFToken.transfer(alice, 0)`.
3. `blacklistedUser` calls `Forwarder.execute(req, domainSeparator, requestTypeHash, suffixData, sig)` as `msg.sender`.
4. `Forwarder.execute` executes `EURFToken.transfer(alice, 0)` with `req.from = alice`. Inside `transfer`, `transferSanity(alice, alice, 0)` passes because `alice` is not blacklisted.
5. `Forwarder.execute` then calls `_eurf.payGaslessBasefee(alice, blacklistedUser)`.
6. `EURFToken.payGaslessBasefee` checks `isTrustedForwarder(msg.sender)` and `balanceOf(alice) >= _gaslessBasefee`, and then executes `_update(alice, blacklistedUser, 10e6)`.
7. `blacklistedUser` successfully receives `10e6` EURF tokens, violating the protocol's blacklist compliance invariant. By repeating this with sequential nonces, `alice` can transfer arbitrary amounts of EURF to `blacklistedUser` in increments of `_gaslessBasefee`.

### Impact assessment

- **Severity**: Medium / High (Compliance & Restriction Bypass).
- **Technical Impact**: Blacklisted addresses can receive EURF tokens via gasless basefee payments, directly violating the invariant that blacklisted recipients cannot receive tokens (`RecipientBlacklistedError`).
- **Business Impact**: As EURF is a regulated stablecoin with mandatory AML/compliance blacklisting requirements, allowing blacklisted entities to receive tokens undermines regulatory compliance guarantees.

### Remediation

Enforce `adminSanity(payer, paymaster)` inside `payGaslessBasefee` before calling `_update`.

```solidity
    function payGaslessBasefee(address payer, address paymaster) external override {
        if (!isTrustedForwarder(msg.sender)) revert InvalidForwarder();
        adminSanity(payer, paymaster);
        if (balanceOf(payer) < _gaslessBasefee) revert BalanceTooLow(_gaslessBasefee, balanceOf(payer));
        _update(payer, paymaster, _gaslessBasefee);
        emit GaslessBasefeePaid(payer, paymaster, _gaslessBasefee);
    }
```

## Finding 8: Unsigned Dynamic Fee Deduction in `EURFToken.payGaslessBasefee` and `EURFToken.transferSanity` Exposes Meta-Transaction Signers to Unapproved Fee Liabilities

### Synopsis

In `EURFToken.payGaslessBasefee` and `EURFToken.transferSanity`, dynamic protocol fees (`_gaslessBasefee` and `_txfeeRate`) are debited directly from the meta-transaction signer's balance without being committed to in the user's EIP-712 signature, allowing relayers or fee updates to charge signers more tokens than authorized in their signed intent.

### Technical details

`EURFToken` supports gasless meta-transactions through two EIP-712 signature mechanisms:
1. **EIP-2771 Forwarded Transfers (`Forwarder.execute`)**: A token holder (`payer`) signs a `ForwardRequest(address from, address to, uint256 value, uint256 gas, uint256 nonce, bytes data)` where `data` encodes `transfer(address recipient, uint256 amount)`. When `Forwarder.execute` processes this request:
   - It invokes `EURFToken.transfer(recipient, amount)`, which calls `transferSanity(_msgSender(), recipient, amount)` (`contracts/Token.sol`, lines 156–159). If `_txfeeRate > 0`, `transferSanity` invokes `_payTxFee(sender, amount)` (line 158), deducting `txFees = (amount * _txfeeRate) / 10000` from `payer`.
   - It then invokes `EURFToken.payGaslessBasefee(req.from, _msgSender())` (`contracts/Token.sol`, lines 147–152), which checks `balanceOf(payer) < _gaslessBasefee` (line 149) and executes `_update(payer, paymaster, _gaslessBasefee)` (line 150) to transfer `_gaslessBasefee` from `payer` to the executing `paymaster`.
2. **EIP-3009 Authorization Transfers (`transferWithAuthorization`)**: A token holder (`holder`) signs an authorization for `(holder, spender, value, deadline)`. When submitted on-chain, `transferWithAuthorization` calls `transferSanity(holder, spender, value)` (line 180 -> lines 156–159), deducting `txFees` from `holder` in addition to the signed `value`.

In both flows, the fee liabilities (`_gaslessBasefee` in `payGaslessBasefee` and `txFees` in `transferSanity`) are read dynamically from mutable contract storage at execution time rather than being validated against a signed fee commitment or maximum fee bound in the EIP-712 payload. Furthermore, `ForwardRequest` lacks an expiration timestamp (`deadline`). Consequently:
- A signer who signs a `ForwardRequest` or `transferWithAuthorization` for `amount` has `amount + txFees + _gaslessBasefee` (or `value + txFees`) debited from their balance without having cryptographically bound the fee amount or rate in the signed digest.
- When `ADMIN` legitimately updates `_gaslessBasefee` via `setGaslessBasefee` (to reflect changing network gas costs) or `_txfeeRate` via `setTxFeeRate` while a signed meta-transaction is pending—or if a `paymaster` withholds a signed `ForwardRequest` until `_gaslessBasefee` is raised—`payGaslessBasefee` and `transferSanity` will deduct the higher fee directly from `payer`'s balance without their consent.

### Assumptions and preconditions

1. The `payer` (or `holder`) holds a sufficient `EURFToken` balance to cover the transfer `amount` plus the updated fees (`txFees` and/or `_gaslessBasefee`).
2. The `payer` signs an off-chain `ForwardRequest` (or `transferWithAuthorization`) when `_gaslessBasefee` and/or `_txfeeRate` are at an initial value (e.g., `10` units).
3. Before the signed request is executed on-chain, `ADMIN` updates `_gaslessBasefee` via `setGaslessBasefee` (e.g., during a gas price spike) or `_txfeeRate` via `setTxFeeRate` to a higher value (e.g., `500` units), or a `paymaster` delays broadcasting the non-expiring `ForwardRequest` until such an update occurs.

### Proof of concept

1. Assume `payer` (Bob) has `1,000 EURF` (`1000e6`), `_txfeeRate` is `0`, and `_gaslessBasefee` is currently `10 EURF` (`10e6`).
2. Bob signs a `ForwardRequest` authorizing `EURFToken.transfer(alice, 100e6)` expecting a total balance debit of `110e6` (`100e6` to Alice and `10e6` base fee to the relayer).
3. Network gas prices increase, and `ADMIN` legitimately calls `EURFToken.setGaslessBasefee(200e6)` and `EURFToken.setTxFeeRate(1000)` (10%).
4. The relayer (`paymaster`) calls `Forwarder.execute` with Bob's previously signed `ForwardRequest`.
5. `Forwarder.execute` calls `EURFToken.transfer(alice, 100e6)`:
   - `transferSanity(bob, alice, 100e6)` (line 158) calls `_payTxFee(bob, 100e6)`, deducting `10e6` from Bob to `_feesFaucet`.
   - `super.transfer(alice, 100e6)` deducts `100e6` from Bob to Alice.
6. `Forwarder.execute` then calls `EURFToken.payGaslessBasefee(bob, paymaster)` (lines 149–151):
   - Line 149 checks `balanceOf(bob) < 200e6` (`890e6 < 200e6` is false).
   - Line 150 executes `_update(bob, paymaster, 200e6)`, transferring `200e6` from Bob to `paymaster`.
7. Bob's balance is reduced by `310e6` instead of the `110e6` expected when signing, with `paymaster` receiving `200e6`.

### Impact assessment

- **Severity**: Medium
- **Technical Effect**: Off-chain signatures do not enforce a ceiling on the total tokens debited from the signer's account (`amount + txFees + _gaslessBasefee`), breaking the invariant that a meta-transaction signer cannot lose more tokens than authorized in their signed message.
- **Business Impact**: Users utilizing gasless transactions (`Forwarder`) or EIP-3009 authorizations (`transferWithAuthorization`) are exposed to unexpected balance deductions whenever fee rates or base fees are adjusted prior to execution, or when relayers strategically delay non-expiring `ForwardRequest` submissions until `_gaslessBasefee` is raised.

### Remediation

Include maximum fee parameters (or a signed fee snapshot and expiration `deadline`) in the signed meta-transaction payloads, and enforce in `payGaslessBasefee` and `transferSanity` that the charged fees do not exceed the user's signed limits. Alternatively, if keeping the ERC-20 `transfer` calldata signature unchanged, pass the user-signed `maxGaslessBasefee` and `maxTxFee` to `payGaslessBasefee` / `transferSanity` (verified via `ForwardRequest` suffix data or struct fields):

```solidity
    function payGaslessBasefee(address payer, address paymaster) external override {
        if (!isTrustedForwarder(msg.sender)) revert InvalidForwarder();
        adminSanity(payer, paymaster);
        if (balanceOf(payer) < _gaslessBasefee) revert BalanceTooLow(_gaslessBasefee, balanceOf(payer));
        _update(payer, paymaster, _gaslessBasefee);
        emit GaslessBasefeePaid(payer, paymaster, _gaslessBasefee);
    }
```

To fully bind fee liability to the user's signature:
1. Include `uint256 maxGaslessBasefee`, `uint256 maxTxFeeRate`, and `uint256 deadline` in `Forwarder.ForwardRequest` (or EIP-712 `suffixData`) and validate `block.timestamp <= deadline`, `_gaslessBasefee <= maxGaslessBasefee`, and `_txfeeRate <= maxTxFeeRate` before executing the transfer and base fee payment.
2. In `transferWithAuthorization`, either deduct `txFees` from the `value` received by `spender` so that `holder` is never debited more than the signed `value`, or include `maxTxFee` in the signed authorization struct.

## Finding 9: Unbounded Fee Deduction Beyond Signed Value in `EURFToken.transferWithAuthorization` Lets Spender Debit Up to Twice the Authorized Amount from Holder

### Synopsis

In `EURFToken.transferWithAuthorization`, transaction fees are deducted from the `holder` on top of the signed `value` without being covered by the EIP-712 signature or bounded by a signed maximum fee parameter, allowing up to `value + (value * _txfeeRate / 10000)` to be debited from the signer's balance.

### Technical details

`EURFToken` inherits `ERC20MetaTxUpgradeable`, which implements EIP-712 meta-transaction transfers via `transferWithAuthorization`. In `ERC20MetaTxUpgradeable`, the `holder` signs a typed digest over the following struct:

```solidity
TransferWithAuthorization(address owner,address spender,uint256 value,uint256 nonce,uint256 deadline)
```

This signature expresses the `holder`'s authorization for `value` tokens to be transferred from their account to `spender`.

However, `EURFToken.transferWithAuthorization` (lines 171–182 of `contracts/Token.sol`) overrides this function to call `transferSanity(holder, spender, value)` prior to `super.transferWithAuthorization(...)`:

```solidity
function transferWithAuthorization(
    address holder,
    address spender,
    uint256 value,
    uint256 deadline,
    uint8 v,
    bytes32 r,
    bytes32 s
) public override returns (bool) {
    transferSanity(holder, spender, value);
    return super.transferWithAuthorization(holder, spender, value, deadline, v, r, s);
}
```

Inside `transferSanity`, if `_txfeeRate > 0`, `_payTxFee(holder, value)` is invoked:

```solidity
function _payTxFee(address from, uint256 txAmount) internal override {
    uint256 txFees = calculateTxFee(txAmount);
    if (balanceOf(from) < txFees + txAmount) revert BalanceTooLow(txFees + txAmount, balanceOf(from));
    if (_feesFaucet != address(0)) _update(from, _feesFaucet, txFees);
    emit FeesPaid(from, txFees);
}
```

This design creates two intertwined authorization flaws:
1. **Total debit exceeds signed `value` scope:** The `holder` only signs an authorization for `value`, yet `holder`'s balance is debited by `value + txFees` (where `txFees = (value * _txfeeRate) / 10000` can be up to 100% of `value` since `FEE_RATIO = 10000`). Note also that the technical specification for `updateGasFees` states *"Fees will be payed by spender"*, or alternatively that `value` represents the maximum authorized deduction from `holder`.
2. **No signature binding to execution-time fee rate:** Because `_TWA_TYPEHASH` does not include `_txfeeRate` or a `maxFee` parameter, any off-chain signature remains valid even if `_txfeeRate` is updated between the time the `holder` signs the authorization and the time `transferWithAuthorization` is executed on-chain before `deadline`.

### Assumptions and preconditions

1. A `holder` signs a valid EIP-712 `TransferWithAuthorization` message for `value` with a future `deadline`.
2. `_txfeeRate` is non-zero at the time `transferWithAuthorization` is executed (either already non-zero when signed or updated by `ADMIN` before the signature is submitted on-chain).
3. The `holder` has a token balance of at least `value + calculateTxFee(value)`.

### Proof of concept

1. Suppose `holder` has a balance of `2,000e6` EURF and signs a `TransferWithAuthorization` allowing `spender` to receive `value = 1,000e6` EURF with a `deadline` 1 day in the future, expecting at most `1,000e6` EURF (or the current fee rate, e.g., `0`) to be debited from their balance.
2. Before `spender` broadcasts the transaction, `_txfeeRate` is set to `1000` (10%) and `_feesFaucet` is configured.
3. `spender` calls `EURFToken.transferWithAuthorization(holder, spender, 1000e6, deadline, v, r, s)`.
4. `transferSanity(holder, spender, 1000e6)` executes `_payTxFee(holder, 1000e6)`, transferring `100e6` EURF from `holder` to `_feesFaucet`.
5. `super.transferWithAuthorization` verifies the signature over `value = 1000e6` and transfers `1,000e6` EURF from `holder` to `spender`.
6. `holder`'s balance decreases by `1,100e6` EURF instead of the authorized `1,000e6` EURF.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Signers of `TransferWithAuthorization` messages have more tokens deducted from their balance than the `value` they cryptographically authorized, and are exposed to fee rate changes between signing and execution up to `deadline`.
- **Business Impact:** Violates EIP-712 / ERC-3009 authorization guarantees where a signed `value` must bound the total amount debited from the signer's account, potentially causing unexpected balance depletion or breaking integrations that rely on exact signed amounts.

### Remediation

Either deduct the transaction fee from the signed `value` (so the `holder` is debited at most `value` and `spender` receives `value - txFees`), or include a `maxFee` parameter in the signed `TransferWithAuthorization` struct and verify `txFees <= maxFee`.

If deducting the fee from the authorized `value` so that total charges never exceed the signed amount:

```solidity
    function transferWithAuthorization(
        address holder,
        address spender,
        uint256 value,
        uint256 deadline,
        uint8 v,
        bytes32 r,
        bytes32 s
    ) public override returns (bool) {
        adminSanity(holder, spender);
        uint256 txFees = _txfeeRate > 0 ? calculateTxFee(value) : 0;
        if ( deadline < block.timestamp ) revert DeadLineExpired(deadline);

        bytes32 structHash = keccak256(abi.encode(_TWA_TYPEHASH, holder, spender, value, _useNonce(holder), deadline));
        bytes32 hash = _hashTypedDataV4(structHash);
        if (ECDSA.recover(hash, v, r, s) != holder) revert InvalidSignature();

        if (txFees > 0) {
            if (_feesFaucet != address(0)) _update(holder, _feesFaucet, txFees);
            emit FeesPaid(holder, txFees);
        }
        _update(holder, spender, value - txFees);
        return true;
    }
```

Alternatively, if `value` is intended to be the net amount received by `spender`, include `uint256 maxFee` in `_TWA_TYPEHASH` and the `transferWithAuthorization` parameters, and revert if `calculateTxFee(value) > maxFee`.

## Finding 10: Missing Spender Blacklist Check in `EURFToken.transferFrom` Allows Blacklisted Accounts to Execute Approved Transfers

### Synopsis

`EURFToken.transferFrom` validates blacklist status only for the token owner (`sender`) and the `recipient` via `transferSanity`, omitting any blacklist check on the caller (`_msgSender()`). Consequently, a blacklisted spender with an existing allowance can still initiate token transfers on behalf of other accounts.

### Technical details

In `contracts/Token.sol`, `EURFToken` overrides `transfer` and `transferFrom` to enforce administrative transfer restrictions (`adminSanity`) and transaction fees (`_payTxFee`) via `transferSanity`:

```solidity
function transferSanity(address sender, address recipient, uint256 amount) internal {
    adminSanity(sender, recipient);
    if (_txfeeRate > 0) _payTxFee(sender, amount);
}

function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
    transferSanity(sender, recipient, amount);
    return super.transferFrom(sender, recipient, amount);
}
```

`transferSanity` delegates policy enforcement to `ERC20AdminUpgradeable.adminSanity(address from, address to)`:

```solidity
function adminSanity(address from, address to) internal view {
    if (!hasRole(ADMIN, _msgSender())) {
        if (paused()) revert PausedError();
        if (isBlacklisted(from)) revert SenderBlacklistedError(from);
    }
    if (isBlacklisted(to)) revert RecipientBlacklistedError(to);
    if (to == address(this)) revert TransferToContractError();
}
```

In `transfer(recipient, amount)`, the caller (`_msgSender()`) and the token holder (`from`) are the same account, so passing `_msgSender()` as `from` checks whether the caller is blacklisted.

However, in `transferFrom(sender, recipient, amount)`, three distinct parties can be involved:
1. `sender`: the token holder (`from`) whose balance is debited.
2. `recipient`: the receiving account (`to`) whose balance is credited.
3. `_msgSender()`: the spender initiating the transfer using their allowance.

Because `transferFrom` only passes `sender` and `recipient` to `transferSanity(sender, recipient, amount)`, `adminSanity` checks `isBlacklisted(sender)` and `isBlacklisted(recipient)`, but never checks `isBlacklisted(_msgSender())`. Furthermore, neither `approve` nor `permit` checks blacklist status. As a result, a blacklisted account (`_msgSender()`) that holds an allowance (granted either before or after being blacklisted, or via `permit`) can call `transferFrom` and transfer tokens from `sender` to any non-blacklisted `recipient`.

### Assumptions and preconditions

1. The `EURFToken` contract is initialized and unpaused.
2. A non-blacklisted token holder (`sender`) has granted an allowance to another account (`spender`) via `approve` or `permit` (or `spender` submits a signed `permit` transaction).
3. The `ADMIN` role has blacklisted `spender` (`isBlacklisted(spender) == true`).
4. The `recipient` address is not blacklisted and is not `address(this)`.

### Proof of concept

1. Alice (not blacklisted) holds `1,000e6` EURF and approves Bob (`spender`) for `500e6` EURF via `eurfToken.approve(bob, 500e6)`.
2. The `ADMIN` account blacklists Bob by calling `eurfToken.blacklist(bob)` so that `eurfToken.isBlacklisted(bob)` returns `true`.
3. Bob calls `eurfToken.transferFrom(alice, charlie, 500e6)`, where Charlie is a non-blacklisted address controlled by or chosen by Bob.
4. Inside `transferFrom`:
   - `transferSanity(alice, charlie, 500e6)` invokes `adminSanity(alice, charlie)`.
   - `isBlacklisted(alice)` is `false` and `isBlacklisted(charlie)` is `false`.
   - `isBlacklisted(bob)` (`_msgSender()`) is never checked.
5. `super.transferFrom(alice, charlie, 500e6)` consumes Bob's allowance and transfers `500e6` EURF from Alice to Charlie, allowing the blacklisted account Bob to move funds to an unblacklisted address.

### Impact assessment

- **Severity**: Medium / High (Compliance & Access Control Bypass)
- **Technical Impact**: Blacklisted accounts retain the ability to act as spenders and move funds from any account that has granted them an allowance (or signed a permit) to arbitrary non-blacklisted recipients.
- **Business Impact**: As a regulated euro-backed stablecoin (registered with the French AMF), EURF relies on blacklist enforcement to freeze sanctioned or compromised actors. Allowing blacklisted addresses to initiate third-party token transfers undermines regulatory compliance guarantees.

### Remediation

Check that the caller (`_msgSender()`) is not blacklisted in `transferFrom` (unless the caller has the `ADMIN` role, consistent with `adminSanity`).

```solidity
    function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
        if (!hasRole(ADMIN, _msgSender()) && isBlacklisted(_msgSender())) {
            revert SenderBlacklistedError(_msgSender());
        }
        transferSanity(sender, recipient, amount);
        return super.transferFrom(sender, recipient, amount);
    }
```

## Finding 11: Unchecked Fee Deduction Against Allowance in `EURFToken.transferFrom` Allows Spenders to Exceed Approved Allowance

### Synopsis

In `EURFToken.transferFrom`, `transferSanity` deducts transaction fees (`txFees`) directly from the token owner (`sender`) via `_payTxFee` without checking or deducting `txFees` from the caller's (`_msgSender()`) allowance (or charging the spender), enabling an approved spender to drain more tokens from `sender` than authorized.

### Technical details

In `contracts/Token.sol`, `EURFToken.transferFrom` overrides `ERC20Upgradeable.transferFrom` as follows:

```solidity
function transferSanity(address sender, address recipient, uint256 amount) internal {
    adminSanity(sender, recipient);
    if (_txfeeRate > 0) _payTxFee(sender, amount);
}

function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
    transferSanity(sender, recipient, amount);
    return super.transferFrom(sender, recipient, amount);
}
```

When `_txfeeRate > 0`, calling `transferFrom(sender, recipient, amount)` executes two separate balance deductions from the token owner `sender`:

1. `transferSanity(sender, recipient, amount)` (line 167) invokes `_payTxFee(sender, amount)` (line 158), which calculates `txFees = calculateTxFee(amount)` and transfers `txFees` directly from `sender` to `_feesFaucet` using `_update(sender, _feesFaucet, txFees)` (line 129).
2. `super.transferFrom(sender, recipient, amount)` (line 168) invokes `_spendAllowance(sender, _msgSender(), amount)` and `_transfer(sender, recipient, amount)`, which checks and decrements `allowance(sender, _msgSender())` by only `amount` while deducting `amount` from `sender`'s balance.

This creates a mismatch between the allowance check and the actual balance deduction from `sender` (as well as contradicting the technical specification for `updateGasFees`, which states: *"Enable ADMIN to set a percentage taken on amount on each transfer. Fees will be payed by spender"*):

- If the transaction fee is deducted from `sender`, `_spendAllowance` only verifies `allowance(sender, _msgSender()) >= amount` instead of `amount + txFees`. As a result, a spender approved for `amount` causes `sender` to lose `amount + txFees` tokens, exceeding the maximum spending limit authorized by `sender`.
- Alternatively, if the spender (`_msgSender()`) is intended to pay the transaction fee, passing `sender` instead of `_msgSender()` to `_payTxFee` wrongly charges the token owner instead of the spender.

### Assumptions and preconditions

1. `ADMIN` has configured a non-zero transaction fee rate (`_txfeeRate > 0`) and a valid `_feesFaucet` address.
2. A token holder (`sender`) has granted a non-infinite allowance (`0 < allowance < type(uint256).max`) to a `spender` (`_msgSender()`) and holds a balance of at least `amount + calculateTxFee(amount)`.

### Proof of concept

1. `ADMIN` sets `_feesFaucet` to `feesFaucet` and sets `_txfeeRate = 1000` (10%, as in `test/Token.js`).
2. Alice (`sender`) holds `1,000` EURF and approves Bob (`spender`) for `500` EURF via `approve(bob, 500)`, expecting Bob to be able to reduce her balance by at most `500` EURF.
3. Bob calls `transferFrom(alice, bob, 500)`.
4. Inside `transferSanity(alice, bob, 500)`, `_payTxFee(alice, 500)` computes `txFees = (500 * 1000) / 10000 = 50` EURF and executes `_update(alice, feesFaucet, 50)`, reducing Alice's balance from `1,000` to `950` EURF.
5. Inside `super.transferFrom(alice, bob, 500)`, `_spendAllowance(alice, bob, 500)` checks `allowance(alice, bob) >= 500`, reduces Bob's allowance from `500` to `0`, and transfers `500` EURF from Alice to Bob.
6. Alice's final balance is `450` EURF: `550` EURF was deducted from Alice's account even though she only authorized Bob to spend `500` EURF.

### Impact assessment

**Severity: Medium**

- **Direct Technical Impact:** Any approved spender calling `transferFrom` deducts `amount + calculateTxFee(amount)` from the token owner's balance while only consuming `amount` of allowance. Because `_txfeeRate` can be set up to `FEE_RATIO` (`10000`, or 100%), a spender can drain up to twice the allowance granted by the token owner.
- **Business & Ecosystem Impact:** Breaks the standard ERC20 allowance invariant that a spender cannot debit an owner's balance by more than `allowance(owner, spender)`, putting users and DeFi integrations that grant exact allowances at risk of unexpected balance depletion.

### Remediation

Consume the spender's allowance for the transaction fee as well before deducting the fee from `sender` (or charge the fee to `_msgSender()` if the spender is intended to pay the fee). If `sender` pays the fee, spend the allowance for `txFees` in `transferFrom`:

```solidity
    function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
        if (_txfeeRate > 0) {
            _spendAllowance(sender, _msgSender(), calculateTxFee(amount));
        }
        transferSanity(sender, recipient, amount);
        return super.transferFrom(sender, recipient, amount);
    }
```
