# Audit report — 2026-01-tempo-mpp-streams/H-03

## Finding 1: Unchecked Fee-on-Transfer Token Deposit in `TempoStreamChannel.openChannel` Credits Nominal Instead of Received Amount and Breaks Channel Solvency

### Synopsis

`TempoStreamChannel.openChannel` (and `addDeposit`) records the caller-supplied nominal `deposit` amount in `channel.deposit` without measuring the actual token balance change across `safeTransferFrom`. For fee-on-transfer or deflationary tokens, `channel.deposit` exceeds the tokens actually received, causing shared-pool insolvency and draining other channels' deposits upon settlement or closure.

### Technical details

In `TempoStreamChannel.openChannel` (lines 221–262), the payer specifies an `IERC20 token` and a nominal `deposit` amount (lines 223–224). The contract transfers tokens into custody and initializes the `Channel` struct as follows:

```solidity
token.safeTransferFrom(msg.sender, address(this), deposit);

channels[channelId] = Channel({
    payer: msg.sender,
    payee: payee,
    authorizedSigner: authorizedSigner,
    token: token,
    deposit: deposit,
    ...
});
```

Similarly, `addDeposit` (lines 270–283) executes `channel.token.safeTransferFrom(msg.sender, address(this), amount)` and increments `channel.deposit += amount` directly.

Neither function checks `token.balanceOf(address(this))` before and after the transfer to determine the net tokens actually received by the contract. When `token` charges a transfer fee (or otherwise delivers fewer tokens than the nominal `deposit` / `amount`), `channel.deposit` is overstated relative to the actual token balance held by `TempoStreamChannel`.

Downstream settlement and withdrawal functions (`settle` at lines 319–326, `finalize` at lines 369–373, and `close` at lines 424–433) rely on `channel.deposit` as the authoritative backing balance:
- If `TempoStreamChannel` holds tokens from other channels using the same token, settling or closing the over-credited channel transfers the full nominal `channel.deposit` out of the contract's shared token balance, stealing collateral from other channels.
- If no other channels hold excess balances of that token, the final `safeTransfer` in `settle`, `finalize`, or `close` reverts due to insufficient token balance in the contract, permanently locking the remaining tokens in the channel.

### Assumptions and preconditions

- A channel is opened (or topped up) using an ERC-20 / TIP-20 token that deducts a transfer fee or otherwise delivers less than the requested transfer amount on `transferFrom`.
- Either multiple channels share the same token in `TempoStreamChannel` (enabling cross-channel drainage) or a single channel attempts to settle/refund its full recorded `channel.deposit` (resulting in stuck funds).

### Proof of concept

1. Assume `token` is a fee-on-transfer token charging a 1% fee on transfers.
2. Honest User A calls `openChannel` with `deposit = 10,000e6`. The contract receives `9,900e6` tokens, while `channels[channelA].deposit` is recorded as `10,000e6`.
3. Attacker B calls `openChannel` (setting both `payer` and `payee` under their control) with `deposit = 10,000e6`. The contract receives another `9,900e6` tokens (total contract balance: `19,800e6`), and `channels[channelB].deposit` is recorded as `10,000e6`.
4. Attacker B immediately settles or cooperatively closes `channelB` for the full `10,000e6` nominal deposit. `TempoStreamChannel` transfers `10,000e6` tokens to Attacker B, dipping into User A's deposit by `100e6` tokens.
5. When User A later attempts to settle or finalize `channelA` for their remaining balance, only `9,800e6` tokens remain in the contract while `channelA` expects `10,000e6`, causing `safeTransfer` to revert and locking User A's funds.

### Impact assessment

- **Severity**: High
- **Technical Effect**: Violation of the protocol's core solvency invariant (`token.balanceOf(address(channel)) >= sum(deposit - settled)`).
- **Business Impact**: Direct loss of funds for other channel participants sharing the same token, or permanent freezing of channel funds when the final settlement or refund transaction reverts due to insufficient contract balance.

### Remediation

Measure the contract's token balance before and after `safeTransferFrom` in `openChannel` (and `addDeposit`), and record only the actual received amount (or revert if the received amount is zero or less than `deposit`).

```solidity
    function openChannel(
        address payee,
        IERC20 token,
        uint256 deposit,
        address authorizedSigner,
        uint256 deadline
    ) external nonReentrant returns (bytes32 channelId) {
        if (payee == address(0)) revert ZeroAddress();
        if (deposit == 0) revert InvalidAmount();

        channelId = keccak256(
            abi.encodePacked(
                msg.sender,
                payee,
                address(token),
                block.timestamp,
                channelCounter++
            )
        );

        uint256 balanceBefore = token.balanceOf(address(this));
        token.safeTransferFrom(msg.sender, address(this), deposit);
        uint256 received = token.balanceOf(address(this)) - balanceBefore;
        if (received == 0) revert InvalidAmount();

        channels[channelId] = Channel({
            payer: msg.sender,
            payee: payee,
            authorizedSigner: authorizedSigner,
            token: token,
            deposit: received,
            settled: 0,
            gracePeriodEnd: 0,
            finalized: false
        });

        emit ChannelOpened(
            channelId,
            msg.sender,
            payee,
            address(token),
            received,
            authorizedSigner
        );
    }
```

## Finding 2: Missing zero-address validation for `authorizedSigner` in `TempoStreamChannel.openChannel` enables signature verification bypass in `settle`

### Synopsis

`TempoStreamChannel.openChannel` validates that `payee` is non-zero but fails to validate that `authorizedSigner` is not `address(0)`. Because `_recoverSigner` returns `address(0)` on malformed or invalid signatures, any channel opened with `authorizedSigner == address(0)` allows anyone to bypass voucher signature verification in `settle` and drain the deposit.

### Technical details

In `TempoStreamChannel.openChannel` (lines 221–228), the contract checks `if (payee == address(0)) revert ZeroAddress();`, and the NatSpec (lines 78 and 217) states that `authorizedSigner` can be the payer or a delegated hot wallet. However, `openChannel` neither rejects `authorizedSigner == address(0)` nor defaults a zero `authorizedSigner` to `msg.sender`.

When settling vouchers in `settle` (lines 293–334), the contract recovers the signer using `_recoverSigner(digest, signature)` (line 315) and checks:

```solidity
if (signer != channel.authorizedSigner) revert InvalidSigner();
```

In `_recoverSigner` (lines 184–203):
1. If `signature.length != 65`, the function directly returns `address(0)` (lines 188–190) without reverting.
2. Even for a 65-byte signature, `ecrecover(digest, v, r, s)` (line 202) returns `address(0)` whenever the signature parameters (`v`, `r`, `s`) are invalid.

Consequently, if a channel is initialized with `authorizedSigner == address(0)` (for instance, by a payer assuming `address(0)` defaults to `payer`, or due to a client-side configuration error), `_recoverSigner` will return `address(0)` for any empty or invalid signature. The check `signer != channel.authorizedSigner` evaluates `address(0) != address(0)`, which is `false`, completely bypassing authentication and allowing an arbitrary `Voucher` with `cumulativeAmount = channel.deposit` to be settled immediately to `channel.payee`.

### Assumptions and preconditions

1. A payer calls `openChannel` with `authorizedSigner` set to `address(0)` (e.g., expecting the contract to default the authorized signer to `msg.sender` when no delegate is specified, or due to a misconfigured client).
2. The channel has a positive token deposit (`deposit > 0`).

### Proof of concept

1. The payer calls `openChannel(payee, token, 10_000e6, address(0), block.timestamp + 1 hours)`, transferring `10_000e6` tokens into `TempoStreamChannel`.
2. The channel is stored with `channel.authorizedSigner = address(0)` and `channel.deposit = 10_000e6`.
3. The payee (or any caller) constructs a `Voucher` with:
   - `channelId`: the newly opened channel ID
   - `cumulativeAmount`: `10_000e6`
   - `nonce`: `1`
   - `expiry`: `block.timestamp + 1 hours`
4. The caller invokes `settle(voucher, "")` with an empty signature (`signature.length == 0`).
5. Inside `_recoverSigner`, `signature.length != 65` is true, returning `address(0)`.
6. Inside `settle`, `signer != channel.authorizedSigner` (`address(0) != address(0)`) is false, so validation succeeds and `10_000e6` tokens are transferred to `channel.payee` without any valid signature from the payer or an authorized delegate.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Complete bypass of EIP-712 voucher signature authentication in `settle` (and `close` for the voucher signature check) when `authorizedSigner` is `address(0)`.
- **Business Impact:** Any funds deposited into a channel initialized with `authorizedSigner == address(0)` can be immediately claimed in full by the payee using an unsigned voucher before the payer can initiate and finalize a channel closure (`GRACE_PERIOD` of 1 hour).

### Remediation

Validate that `authorizedSigner` is not `address(0)` in `openChannel` (or default it to `msg.sender` if `address(0)` is passed). Reverting with `ZeroAddress()` when `authorizedSigner == address(0)` enforces explicit signer specification:

```solidity
    function openChannel(
        address payee,
        IERC20 token,
        uint256 deposit,
        address authorizedSigner,
        uint256 deadline
    ) external nonReentrant returns (bytes32 channelId) {
        if (payee == address(0) || authorizedSigner == address(0)) revert ZeroAddress();
        if (deposit == 0) revert InvalidAmount();
```

## Finding 3: Missing `deadline` validation in `TempoStreamChannel.openChannel` allows stale transactions to execute after expiration

### Synopsis

`TempoStreamChannel.openChannel` accepts a `deadline` parameter intended to enforce the timestamp by which the transaction must be included, but completely omits validating it against `block.timestamp`. Consequently, delayed or stuck transactions can execute arbitrarily late, unexpectedly locking the payer's tokens into an unwanted payment channel.

### Technical details

According to the NatSpec documentation of `TempoStreamChannel.openChannel` (line 218), the `deadline` parameter specifies the timestamp by which the channel creation transaction must be included in a block:

```solidity
function openChannel(
    address payee,
    IERC20 token,
    uint256 deposit,
    address authorizedSigner,
    uint256 deadline
) external nonReentrant returns (bytes32 channelId) {
    if (payee == address(0)) revert ZeroAddress();
    if (deposit == 0) revert InvalidAmount();
```

However, `openChannel` never checks whether `block.timestamp > deadline` (or `block.timestamp <= deadline`). The `deadline` argument is completely unused throughout the function body.

In Tempo's HTTP 402 Payment Required workflow, an AI agent or LLM service consumer submits an `openChannel` transaction to pre-fund metered inference billing within a specific time window. If the transaction is delayed in the mempool (e.g., due to low gas price, network congestion, or temporary sequencer/validator downtime), the consumer may assume the request expired after `deadline` and either abandon the session or open an alternative channel. When the stale transaction is eventually included on-chain after `deadline`, it still succeeds, transferring `deposit` tokens from `msg.sender` into the contract and locking them for at least `GRACE_PERIOD` (1 hour) before they can be recovered via `initiateClose` and `finalize`.

### Assumptions and preconditions

1. A payer submits an `openChannel` transaction with a specific `deadline` timestamp and has approved `TempoStreamChannel` to spend their TIP-20 tokens.
2. The transaction is not included before `deadline` (e.g., due to mempool delay or low fee) and is included in a block where `block.timestamp > deadline`.

### Proof of concept

1. A payer (AI agent) approves `TempoStreamChannel` to spend `10,000e6` USDC and broadcasts `openChannel(payee, token, 10000e6, signer, t0 + 5 minutes)` at timestamp `t0`.
2. Due to network congestion or low gas fees, the transaction remains pending in the mempool past `t0 + 5 minutes`.
3. Seeing that the deadline `t0 + 5 minutes` has passed without confirmation, the payer assumes the transaction can no longer succeed and opens a new channel or routes the workload to another provider.
4. At `t0 + 2 hours`, the stale transaction is included in a block. Because `openChannel` does not check `block.timestamp > deadline`, the call succeeds and transfers `10,000e6` USDC from the payer into the newly created channel.
5. The payer's `10,000e6` USDC is unexpectedly locked in the duplicate/obsolete channel and cannot be reclaimed until `initiateClose` is called and the full `GRACE_PERIOD` (1 hour) elapses.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Stale `openChannel` transactions bypass expiration constraints and execute at arbitrary future timestamps, locking the payer's ERC-20 funds in an unintended channel for at least `GRACE_PERIOD` and exposing the deposit to voucher settlement if the delegated `authorizedSigner` hot wallet was rotated or compromised in the interim.

### Remediation

Validate that `block.timestamp` has not exceeded `deadline` at the beginning of `openChannel`, reverting if the deadline has passed.

```solidity
    error TransactionExpired();

    function openChannel(
        address payee,
        IERC20 token,
        uint256 deposit,
        address authorizedSigner,
        uint256 deadline
    ) external nonReentrant returns (bytes32 channelId) {
        if (block.timestamp > deadline) revert TransactionExpired();
        if (payee == address(0)) revert ZeroAddress();
        if (deposit == 0) revert InvalidAmount();
```

## Finding 4: Missing Zero-Address Check on Recovered Signer in `TempoStreamChannel.settle` Allows Signature Verification Bypass

### Synopsis

In `TempoStreamChannel.settle`, the address returned by `_recoverSigner` is not checked against `address(0)` before being compared to `channel.authorizedSigner`. If a channel's `authorizedSigner` is `address(0)`, any malformed or invalid signature recovers to `address(0)` and bypasses signature authorization, allowing unauthorized settlement of channel funds.

### Technical details

The `settle` function verifies voucher authorization by hashing the `Voucher` struct using EIP-712 (`_hashTypedDataV4`), recovering the signer via `_recoverSigner(digest, signature)`, and checking `if (signer != channel.authorizedSigner) revert InvalidSigner();` at line 317.

The helper `_recoverSigner` (lines 184–203) returns `address(0)` both when `signature.length != 65` and when the underlying `ecrecover` precompile fails to recover a valid public key (e.g., invalid `v`, `r`, or `s` parameters). Although the contract declares `error InvalidSignature()` at line 157, `settle` never checks `if (signer == address(0)) revert InvalidSignature();` after calling `_recoverSigner`.

Consequently, if a channel is configured with `channel.authorizedSigner == address(0)` (for example, when the payer passes `address(0)` during channel creation expecting it to default to `payer` or disable delegated signing), `_recoverSigner` will return `address(0)` for any invalid or empty signature. The check `signer != channel.authorizedSigner` evaluates to `address(0) != address(0)` (`false`), allowing anyone to forge arbitrary vouchers and drain the channel's deposit to `channel.payee`.

### Assumptions and preconditions

1. A channel exists in `channels` with `authorizedSigner` set to `address(0)` and a non-zero token `deposit`.
2. The channel has not yet been finalized (`channel.finalized == false`).

### Proof of concept

1. A `payer` opens a channel with `deposit = 10,000e6` and `authorizedSigner = address(0)`.
2. An attacker or malicious `payee` constructs a `Voucher` with:
   - `channelId`: the target channel's ID
   - `cumulativeAmount`: `10,000e6` (the full deposit)
   - `nonce`: `1` (greater than `settledNonces[channelId]`)
   - `expiry`: `block.timestamp + 1 hours`
3. The caller invokes `settle(voucher, "")` with an empty signature (`signature.length == 0`).
4. Inside `_recoverSigner(digest, signature)`, `signature.length != 65` is true, so `_recoverSigner` returns `address(0)`.
5. At line 317, `if (signer != channel.authorizedSigner)` compares `address(0) != address(0)`, which does not revert.
6. `settle` updates `channel.settled` to `10,000e6` and transfers the entire `10,000e6` deposit to `channel.payee` without a valid signature.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: Signature verification in `settle` is completely bypassed whenever `channel.authorizedSigner == address(0)`.
- **Business Implications**: Unauthorized settlement of up to 100% of the escrowed deposit in affected channels, resulting in direct loss of funds for the payer because the payer can no longer reclaim the deposit via `initiateClose` and `finalize`.

### Remediation

Validate that the recovered `signer` is not `address(0)` immediately after calling `_recoverSigner` in `settle`, reverting with `InvalidSignature()`. Additionally, consider rejecting malleable upper-range `s` values using OpenZeppelin's `ECDSA` library.

```solidity
        bytes32 digest = _hashTypedDataV4(structHash);
        address signer = _recoverSigner(digest, signature);

        if (signer == address(0)) revert InvalidSignature();
        if (signer != channel.authorizedSigner) revert InvalidSigner();
```

## Finding 5: Lack of Monotonic Cumulative Amount and Caller Enforcement in `TempoStreamChannel.settle` Allows Payer to Invalidate Unsettled Payee Vouchers

### Synopsis

In `TempoStreamChannel.settle`, anyone can settle a voucher signed by `authorizedSigner`, and `voucher.cumulativeAmount` is not required to exceed `channel.settled`. A payer/signer can settle a higher-nonce voucher with `cumulativeAmount == channel.settled` (or minimal delta), advancing `settledNonces` and permanently blocking the payee's legitimate higher-value voucher.

### Technical details

In `TempoStreamChannel`, off-chain payment vouchers (`Voucher`) contain both a `cumulativeAmount` (the total cumulative amount owed to `channel.payee`) and a replay-protection `nonce`. The `authorizedSigner` (which is the `payer` or a delegate acting on behalf of the `payer`) signs vouchers and delivers them off-chain to `channel.payee` as services are consumed.

In `settle(Voucher calldata voucher, bytes calldata signature)` (lines 293–334):
1. Line 301 checks `if (voucher.nonce <= settledNonces[voucher.channelId]) revert NonceTooLow();`.
2. Lines 314–317 recover the EIP-712 signer and verify `if (signer != channel.authorizedSigner) revert InvalidSigner();`.
3. Line 319 computes the payout increment:
   ```solidity
   uint256 delta = voucher.cumulativeAmount - channel.settled;
   ```
4. Lines 323–324 update the channel settlement state:
   ```solidity
   channel.settled = voucher.cumulativeAmount;
   settledNonces[voucher.channelId] = voucher.nonce;
   ```

There is a critical state-integrity flaw in how `settle` enforces cumulative amount progression and settlement authorization:
- `settle` does not require `voucher.cumulativeAmount > channel.settled` (when `voucher.cumulativeAmount == channel.settled`, `delta` is `0` and the transaction succeeds while advancing `settledNonces[voucher.channelId]`; when `voucher.cumulativeAmount < channel.settled`, it panics with an arithmetic underflow).
- `settle` can be invoked by any caller (`msg.sender` is not restricted to `channel.payee`), while the voucher signature is produced by `channel.authorizedSigner` (the payer's side of the channel).

Because the `authorizedSigner` controls voucher generation, the payer/signer can sign a new voucher with a higher nonce (e.g., `voucher.nonce = N + 1` or `type(uint256).max`) but a `cumulativeAmount` equal to the currently settled amount `channel.settled` (resulting in `delta == 0`, or `channel.settled + 1` wei) and call `settle` directly (or front-run the payee's `settle` transaction). This updates `settledNonces[voucher.channelId]` to the higher nonce without paying the payee the cumulative amount owed from earlier vouchers. When the payee subsequently attempts to settle their legitimate voucher `(nonce = N, cumulativeAmount = X)` where `X > channel.settled`, `settle` reverts with `NonceTooLow()`.

### Assumptions and preconditions

- A channel has been opened by `payer` with a non-zero `deposit` of `token` and a valid `authorizedSigner`.
- The `authorizedSigner` has issued one or more signed vouchers to `payee` (e.g., `nonce = 1, cumulativeAmount = 2500e6`) in exchange for off-chain services, which `payee` has not yet settled on-chain.
- The `payer` (or `authorizedSigner`) submits a `settle` transaction before `payee`'s settlement transaction is executed on-chain.

### Proof of concept

1. `payer` opens a channel with `DEPOSIT_AMOUNT = 10_000e6`, designating `payee` and `authorizedSigner`. At this point, `channel.settled = 0` and `settledNonces[channelId] = 0`.
2. `authorizedSigner` signs Voucher A (`channelId`, `cumulativeAmount = 5_000e6`, `nonce = 1`, `expiry = block.timestamp + 1 hours`) and sends it off-chain to `payee` to pay for services.
3. Before `payee` settles Voucher A on-chain, `authorizedSigner` signs Voucher B (`channelId`, `cumulativeAmount = 0`, `nonce = 2`, `expiry = block.timestamp + 1 hours`).
4. `payer` calls `settle(Voucher B, sigB)` and `initiateClose(channelId)`:
   - In `settle`, `Voucher B` has `nonce = 2 > settledNonces[channelId] (0)` and a valid signature from `authorizedSigner`.
   - Line 319 calculates `delta = 0 - 0 = 0`.
   - Line 323 sets `channel.settled = 0`.
   - Line 324 sets `settledNonces[channelId] = 2`.
   - Line 326 transfers `0` tokens to `payee`.
5. `payee` attempts to call `settle(Voucher A, sigA)` during the grace period to claim their `5_000e6` payout.
6. The call reverts at line 301 with `NonceTooLow()` because `Voucher A.nonce (1) <= settledNonces[channelId] (2)`.
7. After `GRACE_PERIOD` (1 hour) passes, `payer` calls `finalize(channelId)` and recovers the entire `10_000e6` deposit, stealing the `5_000e6` owed to `payee`.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Complete bypass of payment channel settlement guarantees. `settledNonces` can be advanced without monotonically increasing `channel.settled` (or with an arbitrarily small `delta`), rendering valid unsettled vouchers permanently unclaimable.
- **Business Impact:** Direct theft of unpaid service fees owed to `payee` (up to 100% of `channel.deposit - channel.settled`), breaking the core escrow guarantee of `TempoStreamChannel`.

### Remediation

Restrict `settle` so that only `channel.payee` can submit vouchers for settlement, and enforce that `voucher.cumulativeAmount` is strictly greater than `channel.settled`.

```solidity
        if (msg.sender != channel.payee) revert UnauthorizedCaller();
        if (signer != channel.authorizedSigner) revert InvalidSigner();
        if (voucher.cumulativeAmount <= channel.settled)
            revert NonceTooLow();

        uint256 delta = voucher.cumulativeAmount - channel.settled;
        if (delta > channel.deposit - channel.settled)
            revert InsufficientDeposit();

        channel.settled = voucher.cumulativeAmount;
        settledNonces[voucher.channelId] = voucher.nonce;

        channel.token.safeTransfer(channel.payee, delta);
```
