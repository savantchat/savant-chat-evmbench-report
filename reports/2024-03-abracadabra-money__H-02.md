# Audit report — 2024-03-abracadabra-money/H-02

## Finding 1: Checked timestamp wraparound in MagicLP._twapUpdate freezes long-lived pools

### Synopsis

MagicLP stores its TWAP checkpoint as a wrapping `uint32` timestamp but subtracts checkpoints using Solidity 0.8 checked arithmetic. At the `2^32`-second rollover, updates revert, disabling swaps and liquidity withdrawals for long-lived pools and potentially locking all pooled assets.

### Technical details

Initialization records the low 32 bits of `block.timestamp` in `_BLOCK_TIMESTAMP_LAST_`. Subsequent TWAP updates likewise truncate the current timestamp to 32 bits and calculate the elapsed period by subtracting the previous checkpoint.

This design requires modular subtraction. However, Solidity 0.8 performs checked unsigned arithmetic by default. After Unix timestamp `2^32`, the truncated current timestamp wraps from a value near `type(uint32).max` to a small value. Subtracting the larger pre-rollover checkpoint from the smaller post-rollover timestamp therefore reverts with arithmetic panic `0x11`.

For example:

- Previous checkpoint: `4,294,967,290`
- Timestamp after rollover: `5`
- Intended modular elapsed time: `11`
- Actual result: checked-underflow revert

Because the transaction reverts before `_BLOCK_TIMESTAMP_LAST_` is updated, every subsequent operation reaching `_twapUpdate()` continues using the stale pre-rollover checkpoint and reverts. The affected call graph includes `sync()` through `_sync()`, swaps through `_setReserve()`, liquidity issuance and redemption, flash loans, parameter updates, and reserve-ratio synchronization.

The checkpoint becomes usable again only when the next 32-bit timestamp epoch reaches the stale value, potentially decades or nearly 136 years later.

### Assumptions and preconditions

- A pool remains deployed across a `2^32`-second timestamp boundary.
- Its last successful TWAP checkpoint before the boundary is greater than the wrapped timestamp afterward.
- A user invokes an operation that reaches `_twapUpdate()` after the rollover.
- The pool cannot be migrated or have its timestamp storage directly repaired before the boundary.
- Normal checkpoint intervals are shorter than `2^32` seconds, as required for 32-bit modular elapsed-time accounting.

### Proof of concept

1. Initialize a MagicLP pool while the timestamp is below but close to `2^32`.
2. Call `sync()` at timestamp `2^32 - 6`, causing `_BLOCK_TIMESTAMP_LAST_` to become `4,294,967,290`.
3. Advance the timestamp to `2^32 + 5`.
4. Call `sync()` again.
5. The current 32-bit timestamp is now `5`.
6. `_twapUpdate()` attempts to subtract `4,294,967,290` from `5`.
7. Observe arithmetic panic `0x11`.
8. Confirm `_BLOCK_TIMESTAMP_LAST_` remains `4,294,967,290`.
9. Attempt a swap or `sellShares()` call and observe that it also reverts when its reserve-update path reaches `_twapUpdate()`.

No token-balance manipulation or privileged account is required.

### Impact assessment

**Severity: Medium.**

Activation is far in the future, reducing present likelihood, but the eventual impact is severe and deterministic for pools that survive across the rollover. Core pool operations become unavailable, including liquidity redemption, potentially locking all underlying assets for decades.

TWAP accumulation also stops completely. Integrations relying on pool state or cumulative pricing may fail, and protocol operators cannot necessarily recover through ordinary parameter or reserve-management functions because those paths also invoke the affected update logic.

### Remediation

Use explicit unchecked `uint32` subtraction so elapsed time follows the intended modulo-`2^32` semantics. This preserves the existing storage layout and is safe provided checkpoints occur less than `2^32` seconds apart.

```solidity
function _twapUpdate() internal {
    uint32 blockTimestamp = uint32(block.timestamp);
    uint32 timeElapsed;

    unchecked {
        timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;
    }

    if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
        unchecked {
            _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
        }
    }

    _BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
}
```

Alternatively, use a non-truncating timestamp type, but that requires careful storage-layout migration for existing pools. Add boundary tests covering timestamps immediately before and after `2^32`.

## Finding 2: Permissionless `MagicLP.sync` Desynchronizes Reserves from Targets and Neutralizes `ratioSync`

### Synopsis

`MagicLP.sync()` is an external, permissionless function that updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to match current token balances without adjusting `_BASE_TARGET_`, `_QUOTE_TARGET_`, or `_RState_`. This desynchronizes PMM reserves from targets, causes unsafe pricing-regime transitions, and renders the owner's `ratioSync()` maintenance operation ineffective.

### Technical details

`MagicLP` implements the DODO V2 Proactive Market Maker (PMM) algorithm (`PMMPricing.sol`), which prices assets based on the relationship between current reserves (`B = _BASE_RESERVE_`, `Q = _QUOTE_RESERVE_`), target reserves (`B0 = _BASE_TARGET_`, `Q0 = _QUOTE_TARGET_`), and the pricing regime `_RState_`:
- When `_RState_ == RState.ONE`: the curve requires `B == B0` and `Q == Q0`. `PMMPricing.adjustedTarget()` performs no target recalculation in this state.
- When `_RState_ == RState.BELOW_ONE`: `B >= B0` and `Q <= Q0`, and `adjustedTarget()` dynamically computes `state.Q0` from `state.Q` and `state.B - state.B0`.
- When `_RState_ == RState.ABOVE_ONE`: `Q >= Q0` and `B <= B0`, and `adjustedTarget()` dynamically computes `state.B0` from `state.B` and `state.Q - state.Q0`.

To account for exogenous token balance changes (such as rebases, airdrops, or direct transfers) while preserving the PMM invariant ratios, `MagicLP` defines `ratioSync()` (lines 504–522), restricted to `onlyImplementationOwner`:

```solidity
if (baseBalance != _BASE_RESERVE_) {
    _BASE_TARGET_ = uint112((uint256(_BASE_TARGET_) * baseBalance) / uint256(_BASE_RESERVE_));
    _BASE_RESERVE_ = uint112(baseBalance);
}
if (quoteBalance != _QUOTE_RESERVE_) {
    _QUOTE_TARGET_ = uint112((uint256(_QUOTE_TARGET_) * quoteBalance) / uint256(_QUOTE_RESERVE_));
    _QUOTE_RESERVE_ = uint112(quoteBalance);
}
```

However, `MagicLP.sync()` (lines 134–136) exposes `_sync()` (lines 567–579) as a permissionless external function:

```solidity
function sync() external nonReentrant {
    _sync();
}
```

`_sync()` directly overwrites `_BASE_RESERVE_` and `_QUOTE_RESERVE_` with `baseBalance.toUint112()` and `quoteBalance.toUint112()` without updating `_BASE_TARGET_`, `_QUOTE_TARGET_`, or `_RState_`. This creates multiple state consistency failures:

1. **Order-Dependent Maintenance Neutralization (`sync()` before `ratioSync()`):** `ratioSync()` relies on the difference between `baseBalance` and `_BASE_RESERVE_` (and `quoteBalance` and `_QUOTE_RESERVE_`) to scale `_BASE_TARGET_` and `_QUOTE_TARGET_`. If any caller invokes `sync()` before `ratioSync()`, `_BASE_RESERVE_` and `_QUOTE_RESERVE_` are set equal to the current balances. A subsequent call to `ratioSync()` evaluates `baseBalance != _BASE_RESERVE_` and `quoteBalance != _QUOTE_RESERVE_` as `false`, permanently skipping the target adjustment.
2. **Reserve-Target Desynchronization and Unsafe Pricing-Regime Transitions:**
   - When `_RState_ == RState.ONE`, calling `sync()` after a balance increase sets `B > B0` or `Q > Q0` while keeping `_RState_ == RState.ONE`. A subsequent small swap transitions `_RState_` into `BELOW_ONE` or `ABOVE_ONE`, where `PMMPricing.adjustedTarget()` treats the unscaled `B - B0` or `Q - Q0` delta as excess inventory caused by trading, inflating the opposite target (`state.Q0` or `state.B0`) and mispricing pool swaps.
   - When `_RState_ == RState.BELOW_ONE` (or `ABOVE_ONE`), an increase in `_BASE_RESERVE_` (or `_QUOTE_RESERVE_`) via `sync()` increases `state.B - state.B0` (or `state.Q - state.Q0`) without scaling `B0` (or `Q0`). `adjustedTarget()` then computes an inflated `state.Q0` (or `state.B0`), which gets written to storage as `_QUOTE_TARGET_` (or `_BASE_TARGET_`) when a swap transitions `_RState_`.
   - Conversely, if a negative balance change occurs in `BELOW_ONE` such that `baseBalance < _BASE_TARGET_` (or `quoteBalance < _QUOTE_TARGET_` in `ABOVE_ONE`) and `sync()` is called, `state.B - state.B0` (or `state.Q - state.Q0`) in `PMMPricing.adjustedTarget()` reverts with an arithmetic underflow on all swaps, flash loans, and `getMidPrice()` queries unless `correctRState()` is called—which forcefully resets `_RState_ = RState.ONE` and overwrites both targets with the current reserves.

### Assumptions and preconditions

1. A `MagicLP` pool has been initialized and provided with initial liquidity via `buyShares()`.
2. The pool's `_BASE_TOKEN_` or `_QUOTE_TOKEN_` balance changes outside of a swap/liquidity operation (e.g., via token rebase, reward distribution, or direct token transfer to the pool).

### Proof of concept

1. A `MagicLP` pool holds `1,000e18` base tokens and `1,000e18` quote tokens at equilibrium (`_RState_ = RState.ONE`, `_BASE_RESERVE_ = _BASE_TARGET_ = 1,000e18`, `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1,000e18`, `_I_ = 1e18`).
2. A small base sale moves the pool into `_RState_ = RState.BELOW_ONE` with `_BASE_RESERVE_ = 1,010e18`, `_BASE_TARGET_ = 1,000e18`, and `_QUOTE_RESERVE_ ≈ 990e18`.
3. An exogenous balance increase (e.g., `100e18` base tokens from a rebase or reward distribution) increases `_BASE_TOKEN_.balanceOf(address(pool))` to `1,110e18`.
4. Before the owner can call `ratioSync()` to proportionally scale `_BASE_TARGET_` and `_BASE_RESERVE_`, an external caller invokes `pool.sync()`.
5. `_BASE_RESERVE_` is updated to `1,110e18` while `_BASE_TARGET_` remains `1,000e18`.
6. When the owner subsequently calls `pool.ratioSync()`, `baseBalance != _BASE_RESERVE_` is `false` (`1,110e18 == 1,110e18`), so `_BASE_TARGET_` is not scaled.
7. In `getPMMState()`, `PMMPricing.adjustedTarget()` now sees `state.B - state.B0 = 110e18` instead of `10e18` and solves for an inflated `state.Q0 ≈ 1,100e18` against `state.Q = 990e18`.
8. The caller sells `~110e18` quote tokens via `sellQuote()`, receiving the `110e18` excess base tokens at the discounted `BELOW_ONE` curve price and transitioning `_RState_` back to `ONE` (or `ABOVE_ONE` with `_QUOTE_TARGET_` overwritten to the inflated `state.Q0`).

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Desynchronizes `_BASE_RESERVE_` and `_QUOTE_RESERVE_` from `_BASE_TARGET_` and `_QUOTE_TARGET_`, bypasses the target scaling logic in `ratioSync()`, distorts the PMM pricing curve and TWAP/mid-price calculations, and can cause underflow reverts in `PMMPricing.adjustedTarget()`.
- **Business Impact:** Exogenous token balance increases intended for liquidity providers can be arbitraged away at distorted prices, and negative balance adjustments can halt swaps and price queries until `correctRState()` resets the pool's targets.

### Remediation

Restrict `sync()` to `onlyImplementationOwner` or remove the external `sync()` entry point so that exogenous balance updates must go through `ratioSync()` (or a properly synchronized admin maintenance call).

```solidity
    function sync() external nonReentrant onlyImplementationOwner {
        _sync();
    }
```

## Finding 3: Retroactive TWAP manipulation in MagicLP.sync corrupts cumulative prices

### Synopsis

`MagicLP.sync` updates cached reserves before accruing the TWAP. Anyone can alter token balances and call `sync`, causing the resulting manipulated PMM price to be applied retroactively across the entire elapsed interval and corrupting cumulative prices used by downstream oracle or collateral systems.

### Technical details

The permissionless `sync()` entry point calls `_sync()`. That internal routine:

1. Reads the pool’s current token balances.
2. Overwrites `_BASE_RESERVE_` and `_QUOTE_RESERVE_` with those balances.
3. Calls `_twapUpdate()` only after the reserve changes.

However, `_twapUpdate()` calculates `timeElapsed` from the previous `_BLOCK_TIMESTAMP_LAST_`, then calls `getMidPrice()`. Because the reserves have already been overwritten, `getMidPrice()` uses the new reserve state while the elapsed time belongs to the old reserve state.

Consequently, the accumulator records:

`new manipulated price × entire time since the previous update`

instead of:

`old price × time during which the old state actually existed`.

An attacker can exploit this without controlling an admin account:

1. Wait while an initialized pool remains idle, accumulating a large uncheckpointed interval.
2. Transfer base or quote tokens directly to the pool to alter its actual balances.
3. Immediately call the permissionless `sync()`.
4. The new balance-dependent PMM price is retroactively attributed to the entire idle interval.

For example, when `RState` is `ONE`, increasing the base balance while leaving `_BASE_TARGET_` unchanged changes the ratio used by `PMMPricing.getMidPrice()`. Although the manipulated balance existed only immediately before `sync`, the cumulative price treats it as having existed since the previous checkpoint.

The `nonReentrant` modifier does not prevent this attack because it is caused by incorrect accounting order within a legitimate, non-reentrant call.

### Assumptions and preconditions

- The pool has been initialized and has nonzero reserves.
- A nonzero amount of time has elapsed since the previous TWAP update.
- The configured PMM parameters make the mid-price responsive to the changed reserve balance, particularly `K > 0`.
- The attacker can transfer at least one pool token directly to the pool.
- A downstream component trusts the cumulative price or a TWAP derived from it for an economically relevant decision.
- The value extractable from the downstream consumer exceeds the cost of temporarily or permanently changing the pool balance.

### Proof of concept

1. Start with an initialized pool at timestamp `T0`, with reserves equal to their targets and cumulative price `C0`.
2. Record the current mid-price as `P0`.
3. Allow `Δt` seconds to pass without any call that updates `_BLOCK_TIMESTAMP_LAST_`.
4. Transfer a sufficiently large amount of the base token directly to the pool. The actual base balance changes, but `_BASE_RESERVE_` and the cumulative price remain unchanged.
5. In the same block as the transfer, call `MagicLP.sync()`.
6. Observe that `_sync()` first stores the donated balance as `_BASE_RESERVE_`.
7. `_twapUpdate()` then evaluates the new mid-price `P1` and records approximately:

   `C1 = C0 + P1 × Δt`

8. The correct cumulative value for the preceding interval should instead have been:

   `C1 = C0 + P0 × Δt`

9. Query a consumer that computes the average as `(C1 - C0) / Δt`. It reports approximately `P1`, even though `P1` existed only immediately before `sync()`.

### Impact assessment

**Severity: High** when the cumulative price is trusted for lending, collateral valuation, liquidation, or settlement.

The flaw defeats the fundamental time-weighting property of the accumulator: an attacker can make a momentary end-of-window balance manipulation appear to have persisted throughout the full window. This materially reduces the capital and duration normally required to manipulate a TWAP.

Worst-case consequences include inflated collateral valuation, excessive borrowing, avoided liquidation, wrongful liquidation, or incorrect settlement followed by unrecoverable protocol bad debt. If the accumulator has no economically relevant consumer, the immediate impact is limited to oracle integrity, but the exposed cumulative value remains unsafe for integration.

### Remediation

Checkpoint the cumulative price before modifying any reserve or other price-defining state. Apply the same ordering invariant to every helper that updates reserves or PMM targets.

For `_sync()`, use:

```solidity
function _sync() internal {
    uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
    uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));

    _twapUpdate();

    if (baseBalance != _BASE_RESERVE_) {
        _BASE_RESERVE_ = baseBalance.toUint112();
    }
    if (quoteBalance != _QUOTE_RESERVE_) {
        _QUOTE_RESERVE_ = quoteBalance.toUint112();
    }
}
```

Add regression tests that wait for a nonzero interval, alter balances immediately before `sync()`, and assert that the elapsed interval is accumulated using the pre-sync price.

## Finding 4: Missing `nonReentrant` Modifier in `MagicLP.correctRState` Enables Cross-Function Reentrancy

### Synopsis

`MagicLP.correctRState()` is the only state-mutating operational function in `MagicLP` that lacks the `nonReentrant` modifier. An attacker can re-enter `correctRState()` during token transfer or flash-loan/share-withdrawal callbacks while reserves and target variables are in an inconsistent intermediate state, corrupting PMM target and `RState` accounting.

### Technical details

In `src/mimswap/MagicLP.sol`, all state-changing pool operations—`sync()` (line 134), `sellBase()` (line 244), `sellQuote()` (line 267), `flashLoan()` (line 290), `buyShares()` (line 360), `sellShares()` (line 420), `setParameters()` (line 479), and `ratioSync()` (line 504)—are protected by the `nonReentrant` modifier.

However, the state-maintenance function `correctRState()` at line 138 omits the `nonReentrant` modifier:

```solidity
function correctRState() external {
    if (_RState_ == uint32(PMMPricing.RState.BELOW_ONE) && _BASE_RESERVE_ < _BASE_TARGET_) {
        _RState_ = uint32(PMMPricing.RState.ONE);
        _BASE_TARGET_ = _BASE_RESERVE_;
        _QUOTE_TARGET_ = _QUOTE_RESERVE_;
    }
    if (_RState_ == uint32(PMMPricing.RState.ABOVE_ONE) && _QUOTE_RESERVE_ < _QUOTE_TARGET_) {
        _RState_ = uint32(PMMPricing.RState.ONE);
        _BASE_TARGET_ = _BASE_RESERVE_;
        _QUOTE_TARGET_ = _QUOTE_RESERVE_;
    }
}
```

During execution of `nonReentrant` functions such as `sellShares()`, `flashLoan()`, `sellBase()`, and `sellQuote()`, external calls are made while `_BASE_RESERVE_`, `_QUOTE_RESERVE_`, `_BASE_TARGET_`, `_QUOTE_TARGET_`, and `_RState_` are only partially updated:

1. **In `sellShares()` (lines 438–448):** `_BASE_TARGET_` and `_QUOTE_TARGET_` are decremented at lines 438–439 *before* `_transferBaseOut()` and `_transferQuoteOut()` at lines 446–447, while `_BASE_RESERVE_` and `_QUOTE_RESERVE_` are only updated afterward via `_sync()` at line 448. If `correctRState()` is invoked during the token transfer callback, it overwrites `_BASE_TARGET_` and `_QUOTE_TARGET_` with the stale pre-withdrawal `_BASE_RESERVE_` and `_QUOTE_RESERVE_` and sets `_RState_ = ONE`. When `_sync()` subsequently executes at line 448, reserves are reduced to the post-withdrawal balances while targets remain at the pre-withdrawal values, breaking the core PMM invariant (`B == B0` and `Q == Q0` when `R == ONE`).
2. **In `flashLoan()` (lines 290–353) and swap functions (`sellBase()` / `sellQuote()`):** External calls (`ICallee(assetTo).FlashLoanCall` at line 295 and outbound token transfers at lines 252–253, 275–276, 291–292, 319, and 341) occur before reserves are synchronized with actual token balances at line 350 (`_sync()`) or lines 262 / 285 (`_setReserve()`). Re-entering `correctRState()` during these callbacks allows mutating `_RState_`, `_BASE_TARGET_`, and `_QUOTE_TARGET_` based on stale reserve values mid-execution.

### Assumptions and preconditions

- A `MagicLP` pool is deployed and initialized.
- The pool is in (or transitions through) a state where `_RState_ == BELOW_ONE` with `_BASE_RESERVE_ < _BASE_TARGET_` or `_RState_ == ABOVE_ONE` with `_QUOTE_RESERVE_ < _QUOTE_TARGET_` (e.g., following a balance/target desynchronization or rounding/rebasing event).
- An attacker triggers an operation with an external callback (`flashLoan`, `sellShares`, or a token transfer hook) and re-enters `correctRState()` before reserves are updated.

### Proof of concept

1. Assume a `MagicLP` pool is in `RState.BELOW_ONE` (or `RState.ABOVE_ONE`) where the reserve-to-target precondition for `correctRState()` is satisfied.
2. An attacker calls `sellShares()` (with a token supporting transfer callbacks) or `flashLoan()` (which invokes `ICallee(assetTo).FlashLoanCall()`).
3. In `sellShares()`, lines 438–439 reduce `_BASE_TARGET_` and `_QUOTE_TARGET_` proportionally to `shareAmount / totalShares`, while `_BASE_RESERVE_` and `_QUOTE_RESERVE_` still hold the pre-withdrawal balances.
4. During the outbound transfer callback at line 446/447 (prior to `_sync()` at line 448), the attacker calls `MagicLP.correctRState()`. Because `correctRState()` lacks `nonReentrant`, the call succeeds and resets `_RState_ = ONE`, `_BASE_TARGET_ = _BASE_RESERVE_` (pre-withdrawal), and `_QUOTE_TARGET_ = _QUOTE_RESERVE_` (pre-withdrawal).
5. Execution returns to `sellShares()`, which runs `_sync()` at line 448 and updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to the lower post-withdrawal balances.
6. The pool is left in `_RState_ == ONE` with `_BASE_TARGET_ > _BASE_RESERVE_` and `_QUOTE_TARGET_ > _QUOTE_RESERVE_`, desynchronizing target liquidity from actual reserves and skewing subsequent swap pricing and LP share calculations.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Cross-function reentrancy into `correctRState()` during mid-transaction callbacks overwrites `_RState_`, `_BASE_TARGET_`, and `_QUOTE_TARGET_` using stale `_BASE_RESERVE_` and `_QUOTE_RESERVE_` values before reserves are synced.
- **Business Impact:** Corruption of PMM pricing state (`B0`, `Q0`, and `RState`), leading to mispriced swaps, distorted TWAP/mid-price calculations, or denial of service in subsequent PMM target calculations (`adjustedTarget()`).

### Remediation

Add the `nonReentrant` modifier to `correctRState()` in `src/mimswap/MagicLP.sol` so that it cannot be invoked reentrantly during another pool operation.

```solidity
    function correctRState() external nonReentrant {
        if (_RState_ == uint32(PMMPricing.RState.BELOW_ONE) && _BASE_RESERVE_ < _BASE_TARGET_) {
            _RState_ = uint32(PMMPricing.RState.ONE);
            _BASE_TARGET_ = _BASE_RESERVE_;
            _QUOTE_TARGET_ = _QUOTE_RESERVE_;
        }
        if (_RState_ == uint32(PMMPricing.RState.ABOVE_ONE) && _QUOTE_RESERVE_ < _QUOTE_TARGET_) {
            _RState_ = uint32(PMMPricing.RState.ONE);
            _BASE_TARGET_ = _BASE_RESERVE_;
            _QUOTE_TARGET_ = _QUOTE_RESERVE_;
        }
    }
```

## Finding 5: Missing TWAP checkpoint in MagicLP.correctRState lets recovery transitions corrupt cumulative pricing

### Synopsis

`MagicLP.correctRState` changes the PMM pricing regime and targets without checkpointing cumulative pricing. An attacker can invoke recovery before a price-moving trade, causing the subsequent TWAP update to apply the manipulated post-trade price retroactively across the entire uncheckpointed interval.

### Technical details

`correctRState` repairs an invalid PMM regime by changing `_RState_` to `ONE` and replacing both pricing targets with the cached reserves. These mutations directly affect `getMidPrice()`, which derives the current price from the regime, reserves, and targets.

Unlike other reserve or target reconciliation paths, neither recovery branch invokes `_twapUpdate()`. Consequently, `_BLOCK_TIMESTAMP_LAST_` remains unchanged after the pricing regime has been reset.

The next swap or synchronization operation eventually invokes `_twapUpdate()`. Because reserve-changing paths update the state before calling `_twapUpdate()`, the accumulator uses the later post-operation price and multiplies it by all time elapsed since the previous checkpoint—including time before the recovery and subsequent price-moving operation.

An attacker can therefore:

1. Call `correctRState` when either recovery condition is satisfied.
2. Leave the cumulative-price timestamp stale.
3. Execute a swap that moves the PMM price in a favorable direction.
4. Cause that manipulated price to be credited retroactively over the entire stale interval.

The defect is an incomplete reconciliation of pricing state: the PMM regime and targets are repaired, but the cumulative pricing state associated with that transition is not checkpointed.

### Assumptions and preconditions

- The pool reaches either recovery condition: `BELOW_ONE` with `_BASE_RESERVE_ < _BASE_TARGET_`, or `ABOVE_ONE` with `_QUOTE_RESERVE_ < _QUOTE_TARGET_`.
- The cumulative price is consumed by an oracle or another economically relevant integration.
- Sufficient time has elapsed since the previous TWAP checkpoint to create a material cumulative-price distortion.
- The attacker can invoke the permissionless recovery function and subsequently perform or induce a price-moving operation.
- The attacker has sufficient capital, potentially flash-borrowed, to move the pool price by the required amount.

### Proof of concept

1. Start with `_BLOCK_TIMESTAMP_LAST_ = t0` and a pool satisfying one of the recovery predicates.
2. Wait until time `t1`, creating a meaningful uncheckpointed interval `t1 - t0`.
3. Call `correctRState`.
4. Observe that the PMM regime and targets are reset, but `_BLOCK_TIMESTAMP_LAST_` remains `t0`.
5. Execute a sufficiently large swap at time `t2` to move `getMidPrice()` in the direction favorable to the attacker.
6. During reserve reconciliation, `_twapUpdate()` executes after the swap state has been installed.
7. It adds approximately `postSwapMidPrice * (t2 - t0)` to `_BASE_PRICE_CUMULATIVE_LAST_`, even though the post-swap price only existed at `t2`.
8. Query the affected TWAP consumer and observe that the resulting average is biased toward the attacker-selected post-swap price.

### Impact assessment

**Severity: Medium.**

The vulnerability corrupts cumulative pricing and can enable manipulation of any downstream component relying on that accumulator. Potential consequences include incorrect LP valuation, mispriced collateral, improper borrowing limits, unfair liquidations, or bad debt.

Exploitation requires a reachable recovery state and a security-sensitive consumer of the cumulative price. The distortion increases with the time since the previous checkpoint and the magnitude of the attacker-induced price movement. If the cumulative price directly secures lending collateral, the practical impact may be substantially higher.

### Remediation

Checkpoint cumulative pricing whenever a recovery branch changes the PMM regime and targets. Since the pre-recovery state may be mathematically invalid, perform the checkpoint immediately after installing the recovered state.

```solidity
function correctRState() external {
    if (_RState_ == uint32(PMMPricing.RState.BELOW_ONE) && _BASE_RESERVE_ < _BASE_TARGET_) {
        _RState_ = uint32(PMMPricing.RState.ONE);
        _BASE_TARGET_ = _BASE_RESERVE_;
        _QUOTE_TARGET_ = _QUOTE_RESERVE_;
        _twapUpdate();
    } else if (_RState_ == uint32(PMMPricing.RState.ABOVE_ONE) && _QUOTE_RESERVE_ < _QUOTE_TARGET_) {
        _RState_ = uint32(PMMPricing.RState.ONE);
        _BASE_TARGET_ = _BASE_RESERVE_;
        _QUOTE_TARGET_ = _QUOTE_RESERVE_;
        _twapUpdate();
    }
}
```

Alternatively, introduce a dedicated recovery checkpoint that advances the timestamp without accumulating an undefined pre-recovery price. The chosen policy should be applied consistently across every function that changes reserves, targets, or the PMM regime.

## Finding 6: Checked Multiplication in `Math._SolveQuadraticFunctionForTarget` and `Math._SolveQuadraticFunctionForTrade` Causes `MagicLP.querySellBase` to Revert on Intermediate Overflow

### Synopsis

When porting DODO V2's math library from Solidity 0.6 to Solidity 0.8, the overflow checks `(ki * delta) / ki == delta` and `(idelta * V1) / idelta == V1` in `Math.sol` were left in checked arithmetic, causing `MagicLP.querySellBase` (and swaps/flashloans) to revert with an overflow panic instead of using the fallback division-first branch.

### Technical details

`MagicLP` is a Solidity 0.8 port of DODO V2's `DODOStablePool`. When `MagicLP.querySellBase` (`src/mimswap/MagicLP.sol:167-178`) is called (directly or via `sellBase` and `flashLoan`), it executes:
1. `PMMPricing.PMMState memory state = getPMMState();` (`src/mimswap/MagicLP.sol:171`), which calls `PMMPricing.adjustedTarget(state)` (`src/mimswap/libraries/PMMPricing.sol:190-201`) and invokes `Math._SolveQuadraticFunctionForTarget` (`src/mimswap/libraries/Math.sol:79-104`).
2. `(receiveQuoteAmount, newRState) = PMMPricing.sellBaseToken(state, payBaseAmount);` (`src/mimswap/MagicLP.sol:172`), which invokes `Math._SolveQuadraticFunctionForTrade` (`src/mimswap/libraries/Math.sol:131-205`).

In the original Solidity 0.6 DODO V2 implementation, SafeMath `.mul()` was used for checked multiplications, while raw `*` was intentionally used in two intermediate overflow checks so that wrapping multiplication would cause the equality check to evaluate to `false` and fall back to dividing before multiplying:

1. In `Math._SolveQuadraticFunctionForTarget` (`src/mimswap/libraries/Math.sol:93-100`):
```solidity
uint256 ki = (4 * k) * i;
if (ki == 0) {
    _sqrt = DecimalMath.ONE;
} else if ((ki * delta) / ki == delta) {
    _sqrt = sqrt(((ki * delta) / V1) + DecimalMath.ONE2);
} else {
    _sqrt = sqrt(((ki / V1) * delta) + DecimalMath.ONE2);
}
```
2. In `Math._SolveQuadraticFunctionForTrade` (`src/mimswap/libraries/Math.sol:152-159`):
```solidity
uint256 idelta = i * delta;
if (idelta == 0) {
    temp = 0;
} else if ((idelta * V1) / idelta == V1) {
    temp = (idelta * V1) / (V0 * V0);
} else {
    temp = (((delta * V1) / V0) * i) / V0;
}
```

Because `Math.sol` is compiled with Solidity `>=0.8.0`, the multiplications `ki * delta` (line 96) and `idelta * V1` (line 155) are performed with default checked arithmetic. Whenever the intermediate product `ki * delta` or `idelta * V1` exceeds `type(uint256).max`, the transaction immediately reverts with `Panic(0x11)` at the `else if` condition rather than entering the `else` branch (lines 98-100 and 157-159), rendering both fallback branches unreachable dead code.

Within valid pool parameters (`k <= MAX_K = 1e18`, `i <= MAX_I = 1e36` or `DecimalMath.reciprocalFloor(i) <= 1e36`, and reserves/deltas up to `type(uint112).max ≈ 5.19e33`), `ki * delta` can reach `~2.07e88` and `idelta * V1` can reach `~2.69e103`, both of which exceed `type(uint256).max ≈ 1.1579e77`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with valid parameters permitted by `MagicLP.init` (e.g., high `i` or low `i` producing a large `DecimalMath.reciprocalFloor(i)`, and non-zero `k` up to `MAX_K = 1e18`).
2. The pool reserves or swap amount (`delta`) are large enough that the intermediate product `ki * delta` (when `R != ONE`) or `idelta * V1` (when `k == 1e18`) exceeds `type(uint256).max`, while the final quotient after dividing by `V1` or `V0 * V0` fits within `uint256`.

### Proof of concept

1. Consider a `MagicLP` pool initialized with `k = 1e18` (`MAX_K`) and `i = 1e30` (within `MAX_I = 1e36`), or with `i = 1e6` (such as a 6-decimal quote token per 18-decimal base token, where `DecimalMath.reciprocalFloor(i) = 1e30` when `R == ABOVE_ONE`).
2. Suppose the pool is in `RState.ABOVE_ONE` with `B = 10_000e18` (`1e22`) and `Q - Q0 = 10_000e30` (`1e34`, which fits in `uint112`), or in `RState.BELOW_ONE` with `i = 1e30` and `B - B0 = 1e30`.
3. A user or periphery contract calls `querySellBase` (or `sellBase` / `flashLoan`).
4. `querySellBase` calls `getPMMState()`, which calls `PMMPricing.adjustedTarget(state)` -> `Math._SolveQuadraticFunctionForTarget`.
5. In `Math._SolveQuadraticFunctionForTarget`, `ki = 4 * 1e18 * 1e30 = 4e48`. With `delta = 1e30`, the expression `ki * delta = 4e78 > type(uint256).max (1.1579e77)` is evaluated at `Math.sol:96`.
6. Similarly, in `Math._SolveQuadraticFunctionForTrade` with `k = 1e18`, `i = 1e30`, `delta = 1e24`, and `V1 = 1e24`, `idelta = 1e54` and `idelta * V1 = 1e78 > type(uint256).max` at `Math.sol:155`.
7. Instead of evaluating the condition as `false` and executing the fallback division-first `else` branch (`((ki / V1) * delta)` or `(((delta * V1) / V0) * i) / V0`), Solidity 0.8 reverts with `Panic(0x11)`, causing a denial of service on `querySellBase`, `sellBase`, `querySellQuote`, `sellQuote`, and `flashLoan`.

### Impact assessment

**Severity: Medium**

When the intermediate product `ki * delta` or `idelta * V1` overflows `uint256`, core pool operations that depend on `getPMMState()`, `querySellBase()`, and `querySellQuote()`—including `sellBase()`, `sellQuote()`, `flashLoan()`, `buyShares()`, and `sellShares()`—revert with an arithmetic panic instead of using the intended fallback calculation. This can freeze trading and liquidity operations in pools with large price scalars `i` (or small `i` whose reciprocal is large) and high reserve deltas.

### Remediation

Perform the overflow-check multiplications inside an `unchecked` block (or use `FixedPointMathLib.fullMulDiv`) so that intermediate `uint256` overflow is handled gracefully without reverting.

```solidity
    function _SolveQuadraticFunctionForTarget(uint256 V1, uint256 delta, uint256 i, uint256 k) internal pure returns (uint256) {
        if (k == 0) {
            return V1 + DecimalMath.mulFloor(i, delta);
        }

        if (V1 == 0) {
            return 0;
        }
        uint256 _sqrt;
        uint256 ki = (4 * k) * i;
        if (ki == 0) {
            _sqrt = DecimalMath.ONE;
        } else {
            uint256 kiDelta;
            unchecked {
                kiDelta = ki * delta;
            }
            if (kiDelta / ki == delta) {
                _sqrt = sqrt((kiDelta / V1) + DecimalMath.ONE2);
            } else {
                _sqrt = sqrt(((ki / V1) * delta) + DecimalMath.ONE2);
            }
        }
        uint256 premium = DecimalMath.divFloor(_sqrt - DecimalMath.ONE, k * 2) + DecimalMath.ONE;
        return DecimalMath.mulFloor(V1, premium);
    }
```

And in `Math._SolveQuadraticFunctionForTrade`:

```solidity
        if (k == DecimalMath.ONE) {
            uint256 temp;
            uint256 idelta = i * delta;
            if (idelta == 0) {
                temp = 0;
            } else {
                uint256 ideltaV1;
                unchecked {
                    ideltaV1 = idelta * V1;
                }
                if (ideltaV1 / idelta == V1) {
                    temp = ideltaV1 / (V0 * V0);
                } else {
                    temp = (((delta * V1) / V0) * i) / V0;
                }
            }
            return (V1 * temp) / (temp + DecimalMath.ONE);
        }
```

## Finding 7: Precision Loss in Target Reserve Recalibration Allows Zero-Input `MagicLP.sellBase` Calls to Drain Quote Reserves and Ratchet Down Base Target

### Synopsis

In `MagicLP.querySellBase`, fixed-point truncation in `getPMMState` (`Math._SolveQuadraticFunctionForTarget`) can evaluate `state.B0` equal to `state.B` while `state.R == RState.ABOVE_ONE` and `state.Q > state.Q0`. Because `MagicLP.sellBase` lacks a non-zero input check, calling `sellBase` with `0` base input triggers the `payBaseAmount == backToOnePayBase` branch in `PMMPricing.sellBaseToken`, paying out `state.Q - state.Q0` for free and permanently lowering `_BASE_TARGET_`.

### Technical details

When a `MagicLP` pool is in `RState.ABOVE_ONE` (after quote tokens have been sold into the pool via `sellQuote`), `_QUOTE_TARGET_` (`Q0`) is fixed in storage while the base target `B0` is dynamically recomputed on each query/trade inside `MagicLP.querySellBase` (lines 167–178) via `getPMMState()` -> `PMMPricing.adjustedTarget(state)` -> `Math._SolveQuadraticFunctionForTarget`:

```solidity
// src/mimswap/libraries/PMMPricing.sol
} else if (state.R == RState.ABOVE_ONE) {
    state.B0 = Math._SolveQuadraticFunctionForTarget(
        state.B,
        state.Q - state.Q0,
        DecimalMath.reciprocalFloor(state.i),
        state.K
    );
}
```

In `Math._SolveQuadraticFunctionForTarget(V1, delta, i, k)` (lines 79–104), the target reserve $V_0 = V_1 \left(1 + \frac{\sqrt{1 + 4 k i \Delta / V_1} - 1}{2k}\right)$ is computed in three fixed-point steps that round down:

```solidity
// src/mimswap/libraries/Math.sol
_sqrt = sqrt(((ki * delta) / V1) + DecimalMath.ONE2);
uint256 premium = DecimalMath.divFloor(_sqrt - DecimalMath.ONE, k * 2) + DecimalMath.ONE;
return DecimalMath.mulFloor(V1, premium);
```

This introduces severe fixed-point precision loss when `delta = state.Q - state.Q0` is small relative to `V1 / (2 * k * i)`:
1. Because `DecimalMath.ONE2` is $10^{36}$ and $(10^{18} + 1)^2 = 10^{36} + 2 \cdot 10^{18} + 1$, whenever $\lfloor(4 k i \Delta) / V_1\rfloor \le 2 \cdot 10^{18}$ (i.e., $i \cdot \Delta \le \frac{V_1 \cdot 10^{36}}{2k}$), integer `sqrt` truncates `_sqrt` to `DecimalMath.ONE` ($10^{18}$). Every 1-unit truncation in `_sqrt` is subsequently amplified by $\frac{V_1}{2k}$ in `DecimalMath.divFloor(..., k * 2)` and `DecimalMath.mulFloor(V1, premium)`.
2. Furthermore, when `V1` (`state.B`) is denominated in a low-decimal token (e.g., 6 or 8 decimals), `DecimalMath.mulFloor(V1, premium)` truncates `(V1 * (premium - 1e18)) / 1e18` to `0` whenever `premium - 1e18 < 1e18 / V1`.

In both cases, `_SolveQuadraticFunctionForTarget` returns `state.B0 == state.B` even though `state.Q > state.Q0`.

Next, `MagicLP.querySellBase` calls `PMMPricing.sellBaseToken(state, payBaseAmount)` (lines 45–62):

```solidity
// src/mimswap/libraries/PMMPricing.sol
} else if (state.R == RState.ABOVE_ONE) {
    uint256 backToOnePayBase = state.B0 - state.B;
    uint256 backToOneReceiveQuote = state.Q - state.Q0;
    if (payBaseAmount < backToOnePayBase) {
        ...
    } else if (payBaseAmount == backToOnePayBase) {
        // case 2.2: R status changes to ONE
        receiveQuoteAmount = backToOneReceiveQuote;
        newR = RState.ONE;
    }
```

When `state.B0 == state.B`, `backToOnePayBase` is `0`, while `backToOneReceiveQuote` is `state.Q - state.Q0 > 0`. Although `MagicLP` declares `error ErrNoBaseInput()` (line 45), `MagicLP.sellBase` never validates that `baseInput > 0` (nor does `querySellBase` check `payBaseAmount > 0`). Consequently, calling `sellBase` with `baseInput == 0` satisfies `payBaseAmount == backToOnePayBase` (`0 == 0`) and enters Case 2.2:
- `querySellBase` returns `receiveQuoteAmount = state.Q - state.Q0` (minus fees), `newRState = RState.ONE`, and `newBaseTarget = state.B0 = state.B`.
- `MagicLP.sellBase` transfers `receiveQuoteAmount` of quote tokens to the caller for **0** base tokens, transitions `_RState_` from `ABOVE_ONE` to `ONE`, and overwrites `_BASE_TARGET_` with the reduced `state.B`.

### Assumptions and preconditions

- A `MagicLP` pool is initialized with valid parameters (`0 < _K_ <= MAX_K`, `0 < _I_ <= MAX_I`, `MIN_LP_FEE_RATE <= _LP_FEE_RATE_ <= MAX_LP_FEE_RATE`) and non-zero liquidity.
- For the repeatable drain cycle, $\frac{V_1 \cdot 10^{18}}{2k \cdot i}$ (for 18-decimal tokens) is large enough that `sellQuote` yields a positive `receiveBaseAmount` after fees while `(4 * k * i_recip * quoteInput) / state.B <= 2e18` (for example, the default `MIM/USDB` configuration in `test/MIMSwap.t.sol` with `i ≈ 1e18`, `k = 1e14`, `lpFeeRate = 1e14`, and reserves of `1,000,000e18`).

### Proof of concept

Consider a `MagicLP` pool configured with the parameters from `_createDefaultLp()` in `test/MIMSwap.t.sol`:
- `_BASE_TOKEN_` = MIM (18 decimals), `_QUOTE_TOKEN_` = USDB (18 decimals)
- `_I_ = 1e18`, `_K_ = 1e14` ($0.0001$), `_LP_FEE_RATE_ = 1e14` ($0.01\%$)
- Initial reserves at `RState.ONE`: `_BASE_RESERVE_ = _BASE_TARGET_ = 1_000_000e18` ($10^{24}$), `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1_000_000e18` ($10^{24}$).

1. **Transition to `RState.ABOVE_ONE` and extract Base tokens via `sellQuote`**:
   - A caller transfers `quoteInput = 4,999,999,999` wei of USDB to the pool and calls `sellQuote(caller)`.
   - `querySellQuote` calls `Math._SolveQuadraticFunctionForTrade(10^24, 10^24, 4_999_999_999, 10^18, 10^14)`, which computes `V2 = 10^24 - 4,999,999,999` and gross `receiveBaseAmount = 4,999,999,999` wei of MIM.
   - After the $0.01\%$ fee (`lpFee = 249,999`, `mtFee = 250,000`), the caller receives `4,999,499,999` wei of MIM.
   - Pool state after `sellQuote`:
     - `_RState_ = RState.ABOVE_ONE`
     - `_QUOTE_TARGET_ = 10^24`, `_QUOTE_RESERVE_ = 10^24 + 4,999,999,999`
     - `_BASE_RESERVE_ = 10^24 - 4,999,749,999`

2. **Reclaim the Quote tokens for free via zero-input `sellBase`**:
   - The caller immediately invokes `sellBase(caller)` without transferring any MIM (`baseInput = 0`).
   - `querySellBase(tx.origin, 0)` calls `getPMMState()` -> `PMMPricing.adjustedTarget(state)` -> `Math._SolveQuadraticFunctionForTarget(state.B, 4_999_999_999, 10^18, 10^14)`:
     - `ki = 4 * 10^14 * 10^18 = 4 * 10^32`
     - `(ki * delta) / V1 = (4 * 10^32 * 4_999_999_999) / (10^24 - 4_999_749_999) = 1,999,999,999,600,009,999 < 2 * 10^18 + 1`
     - `_sqrt = sqrt(10^36 + 1,999,999,999,600,009,999) = 10^18 = DecimalMath.ONE`
     - `premium = DecimalMath.ONE`
     - `state.B0 = state.B = 10^24 - 4,999,749,999`
   - `PMMPricing.sellBaseToken(state, 0)` computes `backToOnePayBase = state.B0 - state.B = 0` and `backToOneReceiveQuote = state.Q - state.Q0 = 4,999,999,999`.
   - Because `payBaseAmount == backToOnePayBase` (`0 == 0`), it returns `receiveQuoteAmount = 4,999,999,999` and `newR = RState.ONE`.
   - `sellBase` transfers `4,999,499,999` wei of USDB back to the caller, overwrites `_BASE_TARGET_` with `10^24 - 4,999,749,999`, and resets `_RState_` to `RState.ONE`.

3. **Result**:
   - The caller spent only `1,000,000` wei of USDB in fees while extracting `4,999,499,999` wei of MIM, and the pool is back in `RState.ONE` with `_BASE_TARGET_ == _BASE_RESERVE_`, allowing the cycle to be repeated continuously.

### Impact assessment

**Severity: High**

- **Direct Loss of Pool Reserves**: Anyone can call `sellBase` with `0` base input whenever `state.R == RState.ABOVE_ONE` and `state.B0 == state.B` to take `state.Q - state.Q0` quote tokens without providing base tokens, or loop `sellQuote` + `sellBase(0)` to progressively drain base reserves while ratcheting `_BASE_TARGET_` downward.
- **Permanent Corruption of Pool Target (`_BASE_TARGET_`)**: Updating `_BASE_TARGET_` to the uncompensated `state.B` on a zero-input swap permanently shifts the PMM equilibrium target against liquidity providers.

### Remediation

1. Enforce non-zero trade inputs in `sellBase` and `sellQuote` (using `ErrNoBaseInput` / `ErrZeroQuoteAmount`), and return `0` without changing `RState` in `PMMPricing.sellBaseToken` / `PMMPricing.sellQuoteToken` when the input amount is `0`.
2. In `Math._SolveQuadraticFunctionForTarget`, avoid premature truncation when `delta > 0` by rounding `_sqrt` or `premium` up (or falling back to the linear/first-order Taylor term `V1 + DecimalMath.mulCeil(i, delta)` when `_sqrt == DecimalMath.ONE` and `delta > 0`) so that `B0 > B` whenever `Q > Q0`.

```solidity
// In src/mimswap/MagicLP.sol
    function querySellBase(
        address trader,
        uint256 payBaseAmount
    ) public view returns (uint256 receiveQuoteAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newBaseTarget) {
        PMMPricing.PMMState memory state = getPMMState();
        if (payBaseAmount == 0) {
            return (0, 0, state.R, state.B0);
        }
        (receiveQuoteAmount, newRState) = PMMPricing.sellBaseToken(state, payBaseAmount);

        (uint256 lpFeeRate, uint256 mtFeeRate) = _MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_);
        mtFee = DecimalMath.mulFloor(receiveQuoteAmount, mtFeeRate);
        receiveQuoteAmount = receiveQuoteAmount - DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate) - mtFee;
        newBaseTarget = state.B0;
    }

    function sellBase(address to) external nonReentrant returns (uint256 receiveQuoteAmount) {
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
        if (baseInput == 0) {
            revert ErrNoBaseInput();
        }
        uint256 mtFee;
        uint256 newBaseTarget;
        PMMPricing.RState newRState;
        (receiveQuoteAmount, mtFee, newRState, newBaseTarget) = querySellBase(tx.origin, baseInput);

        _transferQuoteOut(to, receiveQuoteAmount);
        _transferQuoteOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);

        // update TARGET
        if (_RState_ != uint32(newRState)) {
            _BASE_TARGET_ = newBaseTarget.toUint112();
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }

        _setReserve(baseBalance, _QUOTE_TOKEN_.balanceOf(address(this)));

        emit Swap(address(_BASE_TOKEN_), address(_QUOTE_TOKEN_), baseInput, receiveQuoteAmount, msg.sender, to);
    }
```

## Finding 8: Intermediate fixed-point overflow in MagicLP.querySellBase freezes large-reserve pools

### Synopsis

`MagicLP.querySellBase` forwards valid uint112-sized PMM state into an overflow-prone fixed-point multiplication. With sufficiently large raw reserves and an interior `K`, every base-sale quote panics, disabling base-side swaps and dependent routes although the mathematical result fits within uint256.

### Technical details

The highlighted call to `PMMPricing.sellBaseToken` delegates pricing to `Math._SolveQuadraticFunctionForTrade`. For `0 < K < 1e18`, that function calculates the quadratic curvature term at `Math.sol:184`.

The calculation first derives a value proportional to `K * V0² / 1e18`, then passes it to `DecimalMath.mulFloor` with a factor proportional to `4 * (1e18 - K)`. `DecimalMath.mulFloor` evaluates `target * d` using ordinary checked uint256 multiplication before dividing by `1e18`.

Consequently, the intermediate product can overflow even though:

- `V0` fits in the pool’s uint112 reserve representation;
- all configured curve parameters satisfy their documented bounds;
- the final fixed-point result fits in uint256.

For example, consider a balanced pool with:

- `V0 = V1 = 10^30`;
- `K = 5 * 10^17`;
- `i = 10^18`;
- `payBaseAmount = 1`;
- `RState.ONE`.

These values are valid because `10^30 < type(uint112).max`.

The relevant intermediate values are:

- `DecimalMath.mulFloor(K, V0) = 5 * 10^29`;
- multiplying that by `V0` gives `5 * 10^59`, which fits uint256;
- `4 * (1e18 - K) = 2 * 10^18`;
- the raw product passed through `DecimalMath.mulFloor` is therefore `10^78`.

Since `10^78 > type(uint256).max`, Solidity raises arithmetic panic `0x11`. However, the intended result after division by `1e18` is only `10^60`, which fits comfortably in uint256. This is therefore an avoidable intermediate-range failure, not an intrinsically unrepresentable PMM result.

The failure occurs before fee calculation. Any execution path relying on `querySellBase`, including an actual base sale, consequently reverts.

### Assumptions and preconditions

- The pair uses tokens whose raw balances can reach approximately `10^30`; high-decimal tokens can reach this level without unusually large whole-token quantities.
- The pair has a valid curve parameter strictly between zero and `1e18`, with values near `5e17` producing the lowest overflow threshold.
- Balanced liquidity raises the relevant reserve and target above the arithmetic threshold while remaining below uint112 maximum.
- A user or liquidity provider can permissionlessly add liquidity, or ordinary protocol growth can reach the affected reserve size.
- The pool implementation uses Solidity 0.8 checked arithmetic, as shown.

### Proof of concept

1. Create a pair for two ERC20 tokens with sufficient supply, such as tokens using 24 decimals.
2. Initialize the pair with:
   - `i = 1e18`;
   - `k = 5e17`;
   - any permitted LP fee;
   - the protocol’s valid fee model.
3. Add `10^30` raw units of each token as balanced liquidity and mint shares.
4. Verify that the pool is in `RState.ONE` and that its quote reserve and quote target are `10^30`.
5. Call `querySellBase(attacker, 1)`.
6. The query reaches the general quadratic branch with `V0 = V1 = 10^30`.
7. The curvature calculation attempts the checked multiplication `2e18 * 5e59 = 1e78`.
8. Observe arithmetic panic `0x11`, despite the intended divided result being `1e60`.
9. Transfer a nonzero base input and invoke `sellBase`; observe that execution reverts through the same quote path.
10. Repeat with router routes requiring that base-side sale and observe that they also cannot execute while the pool remains above the threshold.

### Impact assessment

**Severity: Medium.**

A non-privileged liquidity provider can push a susceptible pool over the arithmetic threshold, after which all base-side quote and swap operations revert. The same failure can arise organically as TVL grows.

The attack does not directly transfer assets to the attacker, but it can halt one side of the market, break router paths, strand separately pre-transferred swap inputs, and require liquidity reduction, migration, or an implementation replacement. The economic threshold can be practical for high-decimal tokens even though it is large for conventional 18-decimal assets.

### Remediation

Use a 512-bit full-precision multiply-divide operation for the curvature term. This preserves the intended fixed-point result without requiring the numerator product to fit in uint256.

```solidity
import {Math as OZMath} from "openzeppelin-contracts/contracts/utils/math/Math.sol";

uint256 curvature = DecimalMath.mulFloor(k, V0) * V0;
uint256 squareRoot = OZMath.mulDiv(
    (DecimalMath.ONE - k) * 4,
    curvature,
    DecimalMath.ONE
);
squareRoot = sqrt((bAbs * bAbs) + squareRoot);
```

Apply the same full-precision approach to other PMM expressions that multiply before dividing. As a less desirable alternative, enforce a conservative reserve and target ceiling during liquidity operations, derived from the maximum intermediate value for every permitted `K`.

## Finding 9: Fee Truncation via `DecimalMath.mulFloor` in `MagicLP.querySellBase` and `MagicLP.querySellQuote` Allows Fee-Free Swaps and Biases Output Against the Pool

### Synopsis

In `MagicLP.querySellBase` (and `MagicLP.querySellQuote`), maintainer and LP fees are computed separately using `DecimalMath.mulFloor` before being subtracted from the swap output. Rounding fees down instead of up systematically overpays traders and allows attackers to completely bypass LP and maintainer fees via small swaps.

### Technical details

In `src/mimswap/MagicLP.sol`, `querySellBase` (lines 174–176) queries `_MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_)` to split `_LP_FEE_RATE_` into `lpFeeRate` (the adjusted LP fee rate) and `mtFeeRate` (the maintainer fee rate), such that `lpFeeRate + mtFeeRate == _LP_FEE_RATE_`. It then calculates the fee deductions from `receiveQuoteAmount`:

```solidity
mtFee = DecimalMath.mulFloor(receiveQuoteAmount, mtFeeRate);
receiveQuoteAmount = receiveQuoteAmount - DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate) - mtFee;
```

The same pattern is used in `querySellQuote` (lines 187–189) for `receiveBaseAmount`.

There are two compounding fixed-point rounding flaws in this calculation:
1. **Rounding Direction Bias (`mulFloor` instead of `mulCeil`):** Both the maintainer fee (`mtFee`) and the LP fee (`DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate)`) are rounded down toward zero via `DecimalMath.mulFloor(target, d) = (target * d) / 1e18`. Because these fees are subtracted from `receiveQuoteAmount`, rounding the fees down rounds the net `receiveQuoteAmount` paid to the trader **up**, violating the standard AMM invariant that rounding must favor the pool and protocol.
2. **Double Truncation Across Split Fee Rates:** Because `_LP_FEE_RATE_` is split in half by `FeeRateModelImpl.getFeeRate` (`mtFeeRate = Math.divCeil(lpFeeRate, 2)` and `adjustedLpFeeRate = lpFeeRate - mtFeeRate`), applying `mulFloor` independently to each half loses up to 2 wei of fee per swap compared to `mulCeil` and doubles the threshold below which fees truncate to zero.

When `receiveQuoteAmount * mtFeeRate < 1e18` and `receiveQuoteAmount * lpFeeRate < 1e18`, both `mtFee` and the LP fee evaluate to `0`. At `MIN_LP_FEE_RATE = 1e14` (0.01%), `lpFeeRate = 5e13` and `mtFeeRate = 5e13`, meaning any swap yielding `receiveQuoteAmount < 20,000` units incurs zero LP fee and zero maintainer fee. For 6-decimal quote tokens (such as USDT or USDC), `19,999` units is `~$0.02`, allowing traders to split swaps or execute micro-swaps on L2 (Blast) without paying any fees.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with a valid `_LP_FEE_RATE_` between `MIN_LP_FEE_RATE` (`1e14`) and `MAX_LP_FEE_RATE` (`1e16`), particularly with low-decimal tokens (e.g., 6-decimal stablecoins such as USDT/USDC) or low `_LP_FEE_RATE_`.
2. A trader executes swaps through `sellBase`, `sellQuote`, or `flashLoan` where `(receiveAmount * feeRate) % 1e18 != 0` or `receiveAmount < 1e18 / max(lpFeeRate, mtFeeRate)`.

### Proof of concept

1. Consider an initialized `MagicLP` pool for `MIM` (18 decimals, base token) and `USDT` (6 decimals, quote token) with `_LP_FEE_RATE_ = 1e14` (0.01%), `i = 1e6`, and `_MT_FEE_RATE_MODEL_` pointing to `FeeRateModel` with `FeeRateModelImpl`.
2. `_MT_FEE_RATE_MODEL_.getFeeRate(trader, 1e14)` returns `lpFeeRate = 5e13` and `mtFeeRate = 5e13`.
3. A trader transfers `19,999 * 1e12` wei of `MIM` (`0.019999 MIM`) to the pool and calls `sellBase(trader)` (or bundles multiple sub-threshold swaps in a single transaction contract).
4. Inside `querySellBase`, `PMMPricing.sellBaseToken` computes `receiveQuoteAmount = 19,999` (`0.019999 USDT`).
5. Line 175 computes `mtFee = DecimalMath.mulFloor(19_999, 5e13) = (19_999 * 5e13) / 1e18 = 0`.
6. Line 176 computes the LP fee as `DecimalMath.mulFloor(19_999, 5e13) = 0`, resulting in `receiveQuoteAmount = 19_999 - 0 - 0 = 19,999`.
7. The trader receives the full `19,999` quote tokens without paying either the LP fee or the maintainer fee.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** Both LP fees (which accrue to pool reserves for liquidity providers) and maintainer fees (`mtFee` transferred to `_MT_FEE_RATE_MODEL_.maintainer()`) are systematically undercharged on swaps and completely zeroed out for small trade outputs.
- **Business Implications:** Liquidity providers and the protocol maintainer suffer continuous fee leakage across swaps, and arbitrageurs or traders on low-gas L2s (Blast) can split trades into smaller increments to evade protocol and LP fees entirely.

### Remediation

Use `DecimalMath.mulCeil` instead of `DecimalMath.mulFloor` when calculating both the maintainer fee and the LP fee so that fee rounding always favors the pool and protocol maintainer:

```solidity
        (uint256 lpFeeRate, uint256 mtFeeRate) = _MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_);
        mtFee = DecimalMath.mulCeil(receiveQuoteAmount, mtFeeRate);
        receiveQuoteAmount = receiveQuoteAmount - DecimalMath.mulCeil(receiveQuoteAmount, lpFeeRate) - mtFee;
        newBaseTarget = state.B0;
    }

    function querySellQuote(
        address trader,
        uint256 payQuoteAmount
    ) public view returns (uint256 receiveBaseAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newQuoteTarget) {
        PMMPricing.PMMState memory state = getPMMState();
        (receiveBaseAmount, newRState) = PMMPricing.sellQuoteToken(state, payQuoteAmount);

        (uint256 lpFeeRate, uint256 mtFeeRate) = _MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_);
        mtFee = DecimalMath.mulCeil(receiveBaseAmount, mtFeeRate);
        receiveBaseAmount = receiveBaseAmount - DecimalMath.mulCeil(receiveBaseAmount, lpFeeRate) - mtFee;
        newQuoteTarget = state.Q0;
    }
```

## Finding 10: Unaccounted LP Fees During PMM State Transitions in `MagicLP.querySellBase` and `MagicLP.querySellQuote` Cause Reserve-Target Invariant Violations and Target Divergence

### Synopsis

In `MagicLP.querySellBase` and `MagicLP.querySellQuote`, `PMMPricing` computes `newRState` and target values before LP fees are deducted from the output amount. Because the LP fee remains in the pool's output token reserve, transitions reaching or crossing `RState.ONE` leave persisted targets, adjusted targets, and reserves inconsistent.

### Technical details

In `MagicLP`, `querySellBase` (lines 167–178) and `querySellQuote` (lines 180–191) determine the swap output, maintainer fee (`mtFee`), post-swap `RState` (`newRState`), and updated target (`newBaseTarget` or `newQuoteTarget`):

```solidity
    function querySellBase(
        address trader,
        uint256 payBaseAmount
    ) public view returns (uint256 receiveQuoteAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newBaseTarget) {
        PMMPricing.PMMState memory state = getPMMState();
        (receiveQuoteAmount, newRState) = PMMPricing.sellBaseToken(state, payBaseAmount);

        (uint256 lpFeeRate, uint256 mtFeeRate) = _MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_);
        mtFee = DecimalMath.mulFloor(receiveQuoteAmount, mtFeeRate);
        receiveQuoteAmount = receiveQuoteAmount - DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate) - mtFee;
        newBaseTarget = state.B0;
    }
```

`querySellQuote` performs the symmetric calculation via `PMMPricing.sellQuoteToken(state, payQuoteAmount)` and sets `newQuoteTarget = state.Q0`.

When the pool is away from equilibrium (`RState.BELOW_ONE` or `RState.ABOVE_ONE`), `getPMMState()` calls `PMMPricing.adjustedTarget(state)` (lines 190–201 of `PMMPricing.sol`):
- In `RState.BELOW_ONE` ($B > B_0, Q < Q_0$), `state.Q0` is dynamically adjusted upward from `state.Q` and `state.B - state.B0` to account for accumulated LP fees, while `state.B0` remains `_BASE_TARGET_`.
- In `RState.ABOVE_ONE` ($Q > Q_0, B < B_0$), `state.B0` is dynamically adjusted upward from `state.B` and `state.Q - state.Q0`, while `state.Q0` remains `_QUOTE_TARGET_`.
- In `RState.ONE`, `adjustedTarget(state)` performs no adjustment and assumes $B = B_0$ and $Q = Q_0$.

When a trade moves the pool toward or across equilibrium—for example, `sellQuote` when `state.R == RState.BELOW_ONE` in `PMMPricing.sellQuoteToken` (lines 83–99 of `PMMPricing.sol`):

```solidity
        } else {
            uint256 backToOnePayQuote = state.Q0 - state.Q;
            uint256 backToOneReceiveBase = state.B - state.B0;
            if (payQuoteAmount < backToOnePayQuote) {
                ...
            } else if (payQuoteAmount == backToOnePayQuote) {
                receiveBaseAmount = backToOneReceiveBase;
                newR = RState.ONE;
            } else {
                receiveBaseAmount = backToOneReceiveBase + _ROneSellQuoteToken(state, payQuoteAmount - backToOnePayQuote);
                newR = RState.ABOVE_ONE;
            }
        }
```

`PMMPricing.sellQuoteToken` assumes the full gross `backToOneReceiveBase = state.B - state.B0` exits the pool so that the base reserve reaches `state.B0` at `payQuoteAmount == backToOnePayQuote`. However, `querySellQuote` subsequently deducts `lpFee = DecimalMath.mulFloor(receiveBaseAmount, lpFeeRate)` from `receiveBaseAmount` (and `querySellBase` deducts `lpFee` from `receiveQuoteAmount`). Because `sellQuote` (lines 275–276) only transfers out `receiveBaseAmount + mtFee = receiveBaseAmount_gross - lpFee`, the `lpFee` remains in the pool's reserve.

This causes three interrelated state-transition defects (symmetrically in both `querySellQuote` / `sellQuote` and `querySellBase` / `sellBase`):

1. **Equilibrium-Boundary State & Target Divergence (`BELOW_ONE -> ONE` and `ABOVE_ONE -> ONE`):**
   When `payQuoteAmount == backToOnePayQuote` in `BELOW_ONE`, `newRState` is `RState.ONE`. `sellQuote` updates `_QUOTE_TARGET_ = state.Q0` and `_RState_ = RState.ONE`, leaving `_BASE_TARGET_ = state.B0`. However, `_BASE_RESERVE_` becomes `state.B0 + lpFee > _BASE_TARGET_`. Because `PMMPricing.adjustedTarget` does nothing when `state.R == RState.ONE`, subsequent calls to `getPMMState()` return `state.B0 = _BASE_TARGET_ < state.B`. The retained `lpFee` is excluded from `state.B0` during subsequent `RState.ONE` pricing (`_ROneSellQuoteToken`) and target persistence (`newBaseTarget = state.B0` if the next trade is `sellBase`).

2. **Reserve-Target Invariant Violation Across Equilibrium (`BELOW_ONE -> ABOVE_ONE` and `ABOVE_ONE -> BELOW_ONE`):**
   When `payQuoteAmount > backToOnePayQuote` in `BELOW_ONE`, let $\Delta Q = \text{payQuoteAmount} - \text{backToOnePayQuote}$ and $\Delta B = \text{\_ROneSellQuoteToken}(\text{state}, \Delta Q)$.
   - First, `_ROneSellQuoteToken` prices $\Delta Q$ using the unadjusted `state.B0` rather than `state.B0 + lpFee_leg1`, creating a pricing discontinuity relative to splitting the trade just after the boundary.
   - Second, whenever $\Delta B < \text{lpFee}$, the net base leaving the pool is $(B - B_0) + \Delta B - \text{lpFee} < B - B_0$, so the new `_BASE_RESERVE_` is $B_0 + \text{lpFee} - \Delta B > B_0$. `sellQuote` sets `_RState_ = RState.ABOVE_ONE` and updates `_QUOTE_TARGET_ = state.Q0`, but leaves `_BASE_TARGET_ = B_0`. Consequently, the pool enters `RState.ABOVE_ONE` with `_BASE_RESERVE_ > _BASE_TARGET_`, violating the PMM invariant that base reserve is below base target in `ABOVE_ONE` (and causing persisted `_BASE_TARGET_` to diverge from the adjusted `state.B0` computed in `getPMMState()`).

### Assumptions and preconditions

- The pool is initialized with valid parameters (`_LP_FEE_RATE_ >= MIN_LP_FEE_RATE`) and non-zero liquidity.
- At least one swap has moved the pool into `RState.BELOW_ONE` or `RState.ABOVE_ONE`.
- A subsequent swap (via `sellQuote`, `sellBase`, or `flashLoan`) pays an input amount equal to or slightly exceeding the amount needed to return the pool to `RState.ONE`.

### Proof of concept

1. **Initial State:**
   Suppose a pool is initialized with $i = 10^{18}$ (price = 1), $K = 0$ (constant-price limit for simplicity of illustration), `_LP_FEE_RATE_ = 1e16` (1%, split into `lpFeeRate = 0.5%` and `mtFeeRate = 0.5%`), and initial liquidity `_BASE_RESERVE_ = _BASE_TARGET_ = 1000e18`, `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1000e18`, `_RState_ = RState.ONE`.

2. **Move to `RState.BELOW_ONE`:**
   Trader A sells $100\text{e18}$ Base via `sellBase`:
   - Gross quote output is $100\text{e18}$.
   - `mtFee = 0.5e18`, `lpFee = 0.5e18`, `receiveQuoteAmount = 99e18`.
   - Pool state becomes:
     - `_RState_ = RState.BELOW_ONE`
     - `_BASE_TARGET_ = 1000e18`, `_BASE_RESERVE_ = 1100e18`
     - `_QUOTE_TARGET_ = 1000e18`, `_QUOTE_RESERVE_ = 900.5e18`

3. **Transition Back to Equilibrium (`payQuoteAmount == backToOnePayQuote`):**
   In `getPMMState()`, `adjustedTarget` computes `state.Q0 = 900.5e18 + (1100e18 - 1000e18) = 1000.5e18`, so `backToOnePayQuote = 1000.5e18 - 900.5e18 = 100e18` and `backToOneReceiveBase = 1100e18 - 1000e18 = 100e18`.
   - Trader B sells exactly $100\text{e18}$ Quote via `sellQuote`.
   - `PMMPricing.sellQuoteToken` returns `receiveBaseAmount = 100e18` and `newRState = RState.ONE`.
   - `querySellQuote` deducts `mtFee = 0.5e18` and `lpFee = 0.5e18`, returning `receiveBaseAmount = 99e18` and `newQuoteTarget = 1000.5e18`.
   - `sellQuote` transfers $99.5\text{e18}$ Base out, sets `_QUOTE_TARGET_ = 1000.5e18`, sets `_RState_ = RState.ONE`, and updates reserves:
     - `_BASE_RESERVE_ = 1100e18 - 99.5e18 = 1000.5e18`
     - `_BASE_TARGET_ = 1000e18` (not updated)
     - `_QUOTE_RESERVE_ = 1000.5e18`
     - `_QUOTE_TARGET_ = 1000.5e18`
   - Now `_RState_ == RState.ONE`, but `_BASE_RESERVE_ (1000.5e18) > _BASE_TARGET_ (1000e18)`, and `getPMMState()` does not adjust `state.B0` in `RState.ONE`.

4. **Alternative: Crossing Equilibrium by 1 Wei (`payQuoteAmount = backToOnePayQuote + 1`):**
   If in Step 3 Trader B instead sells $100\text{e18} + 1$ wei of Quote:
   - `PMMPricing.sellQuoteToken` sets `newRState = RState.ABOVE_ONE` and `receiveBaseAmount = 100e18 + 1`.
   - After `lpFee = 0.5e18` and `mtFee = 0.5e18` are deducted, the pool transfers out $99.5\text{e18} + 1$ wei of Base.
   - Pool state becomes:
     - `_RState_ = RState.ABOVE_ONE`
     - `_BASE_RESERVE_ = 1000.5e18 - 1 > _BASE_TARGET_ (1000e18)`
     - `_QUOTE_RESERVE_ = 1000.5e18 + 1 > _QUOTE_TARGET_ (1000.5e18)`
   - Both `_BASE_RESERVE_ > _BASE_TARGET_` and `_QUOTE_RESERVE_ > _QUOTE_TARGET_` hold simultaneously while `_RState_ == RState.ABOVE_ONE`, violating the `ABOVE_ONE` reserve-target invariant and causing persisted `_BASE_TARGET_ (1000e18)` to diverge from `getPMMState().B0 (1000.5e18)`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:**
  1. Persisted target state (`_BASE_TARGET_` and `_QUOTE_TARGET_`) diverges from actual reserves when landing on `RState.ONE` and diverges from dynamically adjusted targets (`state.B0` / `state.Q0`) when crossing `RState.ONE`.
  2. In `RState.ONE`, `getPMMState()` fails to include the LP fee from the equilibrium-restoring trade in `state.B0` or `state.Q0`, skewing subsequent trade pricing and target updates.
  3. External contracts or aggregators reading `_BASE_TARGET_` and `_QUOTE_TARGET_` directly (such as `IMagicLP._BASE_TARGET_()` / `_QUOTE_TARGET_()`) observe stale targets that violate the PMM reserve-target inequality invariants.

### Remediation

Update target calculation and persistence when transitioning to or across `RState.ONE` so that the retained LP fee is accounted for in the target of the output token. Specifically, in `querySellBase` and `querySellQuote` (and the corresponding target updates in `sellBase`, `sellQuote`, and `flashLoan`), ensure that:
1. When a trade reaches `RState.ONE`, both targets match the post-swap reserves (or the output token target is increased by the retained `lpFee`).
2. When a trade crosses `RState.ONE` (`BELOW_ONE -> ABOVE_ONE` or `ABOVE_ONE -> BELOW_ONE`), the LP fee earned on the `backToOne` portion is reflected in the target before pricing and transitioning into the opposite `RState`.

```solidity
    function querySellBase(
        address trader,
        uint256 payBaseAmount
    ) public view returns (uint256 receiveQuoteAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newBaseTarget) {
        PMMPricing.PMMState memory state = getPMMState();
        (receiveQuoteAmount, newRState) = PMMPricing.sellBaseToken(state, payBaseAmount);

        (uint256 lpFeeRate, uint256 mtFeeRate) = _MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_);
        uint256 lpFee = DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate);
        mtFee = DecimalMath.mulFloor(receiveQuoteAmount, mtFeeRate);
        receiveQuoteAmount = receiveQuoteAmount - lpFee - mtFee;
        newBaseTarget = state.B0;
    }
```

And in `sellBase` / `sellQuote` / `flashLoan`, when `_RState_` transitions from `ABOVE_ONE` or `BELOW_ONE`, update both `_BASE_TARGET_` and `_QUOTE_TARGET_` (including the retained `lpFee` when reaching or crossing `RState.ONE`, and resetting `newRState` to `RState.ONE` if the retained `lpFee` prevents the reserve from dropping below target).

## Finding 11: Zero-input quotes in MagicLP.querySellQuote let sellQuote change the PMM regime without a trade

### Synopsis

`MagicLP.querySellQuote` accepts a zero quote amount and forwards `PMMPricing`’s synthetic `ABOVE_ONE` result to `sellQuote`. An attacker can consequently change the pricing regime and trigger adjusted-target recalculation without moving reserves, obtaining manipulated pricing at LPs’ expense.

### Technical details

When `payQuoteAmount` is zero and the pool is in `RState.ONE`, `querySellQuote` still invokes `PMMPricing.sellQuoteToken`. The pricing library returns zero base output but unconditionally returns `RState.ABOVE_ONE` for the `ONE` branch.

The highlighted fee calculations also produce zero, and `newQuoteTarget` is returned normally. There is no zero-amount normalization or rejection before the result reaches `sellQuote`.

This creates the following state transition:

1. `sellQuote` observes no unaccounted quote balance, so `quoteInput` is zero.
2. `querySellQuote` returns:
   - `receiveBaseAmount = 0`
   - `mtFee = 0`
   - `newRState = ABOVE_ONE`
   - `newQuoteTarget = state.Q0`
3. No tokens are transferred, but `sellQuote` persists `ABOVE_ONE` because the returned regime differs from the stored `ONE` regime.
4. On the next `getPMMState` call, line 201 executes `adjustedTarget` under the attacker-induced `ABOVE_ONE` regime. That branch recomputes `B0` using current reserves and `Q - Q0`, despite no economic trade having caused the regime change.

This becomes economically relevant when `RState.ONE` has reserve surplus relative to a stored target. Such a state is reachable after an exact return-to-one swap: line 189 deducts the LP fee from the trader’s output, so the fee remains in the pool even though pricing reports `ONE`. A zero-input regime change can then cause that surplus to be incorporated into the adjusted target before the attacker’s next trade, changing PMM depth and output without a corresponding reserve movement.

### Assumptions and preconditions

- The pool is initialized, liquid, and permissionlessly callable.
- The pool is in `RState.ONE`.
- For economic exploitation rather than state corruption alone, an actual reserve differs from its stored target, for example because an exact return-to-one swap left an LP fee in the pool.
- The pool has a nonzero `K`, so changing the adjusted target changes trade price impact.
- The attacker can submit a transaction without first transferring quote tokens to the pool.

### Proof of concept

1. Begin with a funded pool in `RState.ONE`.
2. Sell base to move the pool to `BELOW_ONE`.
3. Read the adjusted PMM state and transfer exactly `Q0 - Q` quote tokens to the pool.
4. Call `sellQuote`. This returns the pricing state to `ONE`.
5. Because the base output was fee-adjusted at line 189, verify that:
   - the quote reserve equals the new quote target;
   - the base reserve exceeds the stored base target by the retained LP fee.
6. Record the result of `querySellQuote(attacker, attackAmount)` for a nonzero `attackAmount`.
7. Without transferring any quote tokens, call `sellQuote(attacker)`.
8. Observe that no token balance changes, but `_RState_` changes from `ONE` to `ABOVE_ONE`.
9. Call `getPMMState`. Because `Q == Q0`, the `ABOVE_ONE` adjustment sets the effective `B0` from the current base reserve, incorporating the retained fee surplus.
10. Query the same nonzero `attackAmount` again. For nonzero `K`, the larger effective target reduces price impact and increases the attacker’s base output relative to step 6.
11. Transfer `attackAmount` quote tokens and execute `sellQuote` to realize the manipulated quote.

### Impact assessment

**Severity: Medium.**

Any account can induce the regime transition at zero token cost. The attack violates the fundamental requirement that PMM regime changes correspond to reserve movement and allows an attacker to control when reserve surplus is incorporated into pricing targets.

The immediate effect is incorrect quoting and settlement against attacker-selected adjusted targets. Over repeated boundary transitions, this can divert value represented by retained LP fees through more favorable execution and disrupt integrations that assume reserve changes accompany regime changes. Exploitation requires a reachable `ONE` state with target/reserve divergence, so it is not an unconditional full-pool drain.

### Remediation

Handle zero input before calling `PMMPricing.sellQuoteToken`. The preferred behavior is to return the current regime and adjusted target without manufacturing a transition. As defense in depth, `sellQuote` may instead reject zero observed input.

```solidity
function querySellQuote(
    address trader,
    uint256 payQuoteAmount
) public view returns (uint256 receiveBaseAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newQuoteTarget) {
    PMMPricing.PMMState memory state = getPMMState();

    if (payQuoteAmount == 0) {
        return (0, 0, state.R, state.Q0);
    }

    (receiveBaseAmount, newRState) = PMMPricing.sellQuoteToken(state, payQuoteAmount);

    (uint256 lpFeeRate, uint256 mtFeeRate) = _MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_);
    mtFee = DecimalMath.mulFloor(receiveBaseAmount, mtFeeRate);
    receiveBaseAmount = receiveBaseAmount - DecimalMath.mulFloor(receiveBaseAmount, lpFeeRate) - mtFee;
    newQuoteTarget = state.Q0;
}
```

## Finding 12: Reciprocal truncation in MagicLP.querySellQuote can underpay quote sellers by nearly 50%

### Synopsis

`MagicLP.querySellQuote` delegates quote-side pricing to a fixed-point reciprocal of `_I_` that is rounded before the trade amount is applied. For valid prices with coarse reciprocals, this can underquote base output by nearly 50%, transferring quote sellers’ value to pool reserves and LP holders.

### Technical details

At line 185, `querySellQuote` obtains the gross base output from `PMMPricing.sellQuoteToken`. The quote-side PMM routines represent the inverse price using:

`floor(1e36 / state.i)`

This reciprocal is then multiplied by the quote input using 18-decimal fixed-point arithmetic. In the linear `K == 0` case, the implemented calculation is equivalent to:

`floor(payQuoteAmount * floor(1e36 / i) / 1e18)`

The mathematically equivalent calculation without premature reciprocal truncation is:

`floor(payQuoteAmount * 1e18 / i)`

These expressions are not equivalent. Truncating the reciprocal before multiplying discards a fractional error for every unit of input, so the error scales with the trade rather than remaining bounded to one output unit.

The effect is particularly severe when `i` is large and the encoded reciprocal is small. For example, with `i = 6e35`, the exact fixed-point reciprocal is `5/3`, but `reciprocalFloor` reduces it to `1`. Consequently, sufficiently large quote sales receive approximately 60% of the correct gross base output. As `i` approaches `5e35` from above, the underpayment approaches 50%.

This is a valid-price condition rather than an invalid or malicious initialization: large `i` values can correctly represent similarly priced tokens with substantially different token decimals. Subsequent fee processing only applies percentages to the already-understated output and therefore cannot recover the missing value.

### Assumptions and preconditions

- An honestly created pool uses an economically correct `_I_` whose value does not divide `1e36` and whose fixed-point reciprocal is sufficiently coarse.
- Such a value can arise from the relative decimals and whole-token price of the paired assets; no malicious core deployment is required.
- The pool has enough base liquidity to satisfy both the implemented and mathematically precise outputs.
- The quote input is large enough for reciprocal precision loss to accumulate into a material output difference.
- The swap is executed directly or through an integration whose minimum output is derived from the affected quote.
- The paired tokens otherwise behave as standard ERC-20 tokens.

### Proof of concept

1. Create an honestly parameterized pool with:
   - `_I_ = 6e35`;
   - `_K_ = 0`;
   - LP fee rate `1e14`;
   - base token with 0 decimals and quote token with 18 decimals.
2. This `_I_` represents one base token being worth `0.6` quote tokens:
   - one base unit should return `6e17` quote units;
   - `6e17` quote units should therefore buy one base unit.
3. Seed the pool in state `ONE` with:
   - `2e13` base units;
   - `1.2e31` quote units.
4. Sell `6e30` quote units.
5. The implementation first calculates:

   `floor(1e36 / 6e35) = 1`

6. The gross base output returned through line 185 is therefore:

   `floor(6e30 * 1 / 1e18) = 6e12`

7. A calculation that preserves the original price denominator gives:

   `floor(6e30 * 1e18 / 6e35) = 1e13`

8. With the deployed fee model splitting the `1e14` fee rate equally, the implemented net output is `5,999,400,000,000` base units, while the precision-preserving net output is `9,999,000,000,000` base units.
9. The seller consequently receives approximately 40% less base than the fixed-point price requires. The untransferred value remains in the pool and increases the value attributable to LP shares; an LP can capture a proportional part of it, including through liquidity added around a known large trade.

### Impact assessment

**Severity: Medium.**

A quote seller can lose up to nearly half of the expected base output before ordinary swap fees. Integrations that trust `querySellQuote` will reproduce the incorrect quote and may set minimum-output checks around that value, allowing the transaction to succeed despite the pricing defect.

The discrepancy benefits existing or just-in-time LP holders because the base that should have been paid remains in pool reserves. The issue does not directly let an attacker drain the pool, and meaningful exploitation requires a pool whose unit-adjusted price has a coarse reciprocal, but losses can be substantial for every affected quote-side trade.

Broader consequences include asymmetric bid/ask behavior, unreliable routing, economically incorrect PMM execution, and loss of confidence in pools pairing tokens with materially different decimal scales.

### Remediation

Refactor quote-side PMM pricing so it operates on the original `_I_` as a rational value and postpones division until the final result. In the linear case, calculate quote-to-base output using a full-precision `mulDiv(payQuoteAmount, 1e18, i)` operation rather than first materializing `floor(1e36 / i)`.

The nonlinear PMM and adjusted-target formulas must likewise preserve the reciprocal numerator and denominator throughout their calculations. Use 512-bit full-precision multiplication and division where intermediate products can exceed 256 bits.

Do not replace reciprocal floor rounding with ceiling rounding: that would reverse the bias and could make the pool overpay traders.

If a full rational-math refactor is not immediately possible, a restrictive alternative is to permit only `_I_` values for which `1e36 % _I_ == 0`, ensuring that the current reciprocal representation is exact. This reduces supported prices and should therefore be treated as a temporary mitigation.

Add invariant tests around reciprocal discontinuities and compare quote-side outputs against a high-precision reference implementation for both `K == 0` and nonlinear `K` values.

## Finding 13: Intermediate overflow in `MagicLP.getPMMState` can halt pool swaps

### Synopsis

`MagicLP.getPMMState` invokes target-adjustment arithmetic that multiplies valid price, curvature, and reserve-delta values before fixed-point division. Reachable imbalanced states can therefore overflow under Solidity 0.8, reverting quotes, swaps, and flash-loan settlement and leaving the affected pool unavailable.

### Technical details

Line 201 unconditionally delegates state normalization to `PMMPricing.adjustedTarget`. For `BELOW_ONE`, the adjustment derives `Q0` from `Q`, `B - B0`, `i`, and `K`; for `ABOVE_ONE`, it analogously derives `B0` using the reciprocal price and `Q - Q0`.

The target solver evaluates a discriminant containing the fixed-point term:

`i * K * delta * 4 / 1e18`

The multiplication is performed before division. This is unsafe under the contract’s permitted ranges:

- `i` can be as large as `1e36`.
- `K` can be as large as `1e18`.
- Reserve deltas can approach the `uint112` range.

For example, with `i = 1e36`, `K = 1e18`, and `delta = 1e23`, the intermediate numerator is `4e77`, exceeding `type(uint256).max`, approximately `1.158e77`. Solidity 0.8 consequently reverts.

This is not merely rejection of an unrepresentable result. After fixed-point division, the term is only `4e59`; with `V1 = 1e24`, the resulting target is approximately `3.16e29`, comfortably below `type(uint112).max`, approximately `5.19e33`. The calculation therefore fails solely because of its intermediate ordering.

A non-privileged user can create the required delta after the pool has entered an imbalanced R-state. For example, the user can transfer additional base tokens into a `BELOW_ONE` pool and call public `sync()`, increasing `B - B0` without changing `B0` or the R-state. Subsequent calls to `getPMMState` overflow during target adjustment.

Both quote functions depend on `getPMMState`, and the swap and conditional flash-loan settlement paths depend on those quote functions. The poisoned state persists because each affected operation reverts before it can update reserves or targets.

### Assumptions and preconditions

- The pool uses valid, honestly configured parameters whose `i * K` product is sufficiently large.
- The underlying token has enough supply for an attacker to create the required one-sided reserve delta.
- The pool has entered `BELOW_ONE` or `ABOVE_ONE`; target adjustment is not performed while the state is `ONE`.
- The attacker can transfer tokens directly to the pool and invoke the permissionless reserve synchronization function, or can create the same imbalance through ordinary trading.
- No privileged actor is required, and the owner or administrator does not act maliciously.

### Proof of concept

1. Use a valid pool configuration with `i = 1e36` and `K = 1e18`.
2. Seed the pool with nonzero base and quote liquidity.
3. Perform a small base sale so the pool enters `BELOW_ONE`.
4. Transfer enough base tokens directly to the pool and call `sync()` so that `B - B0` is at least `1e23`. This remains below the `uint112` limit.
5. Call `getPMMState()`.
6. Line 201 invokes target adjustment for the `BELOW_ONE` state.
7. The target solver attempts to evaluate an intermediate numerator of `1e36 * 1e18 * 1e23 * 4 = 4e77`.
8. The multiplication exceeds `type(uint256).max` and the call reverts.
9. Call either quote method or execute a swap. It reaches the same target adjustment and reverts before repairing the state.
10. Observe that the mathematically expected target remains representable: the scaled term is `4e59`, producing a target of approximately `3.16e29` when the current quote reserve is around `1e24`.

### Impact assessment

**Severity: Medium.**

An attacker can cause a pool-level denial of service under affected parameter and reserve conditions. Price queries, both swap directions, and flash-loan settlement paths requiring PMM quotes become unavailable. Integrations relying on on-chain quotation may also fail.

The attack requires a sufficiently large one-sided token amount, so its cost depends on the configured price scale and the underlying asset. The worst case is prolonged market unavailability and disruption of liquidity access until reserves, targets, or parameters are changed through an unaffected recovery path.

### Remediation

Implement the target discriminant using overflow-safe fixed-point arithmetic. Given the protocol’s parameter and `uint112` reserve bounds, the multiplication can be decomposed around the fixed-point denominator without losing precision:

```solidity
function _SolveQuadraticFunctionForTarget(
    uint256 V1,
    uint256 delta,
    uint256 i,
    uint256 k
) internal pure returns (uint256 V0) {
    uint256 fourDelta = delta * 4;
    uint256 ik = i * k;

    uint256 scaledTerm =
        (ik / DecimalMath.ONE) * fourDelta +
        ((ik % DecimalMath.ONE) * fourDelta) / DecimalMath.ONE;

    uint256 discriminant = (V1 * V1) + scaledTerm;
    V0 = (sqrt(discriminant) + V1) / 2;
}
```

Alternatively, use a proven 512-bit `mulDiv` implementation to calculate the scaled term. Add boundary tests for both R-state branches, maximum supported `i` and `K`, and cases where the intermediate numerator exceeds `uint256` while the final target remains representable.

## Finding 14: Unbalanced Initial Deposit Accounting in `MagicLP.buyShares` Corrupts Reserve-Target Invariants and Enables Subsequent LP Deposit Theft

### Synopsis

In `MagicLP.buyShares`, initial `shares`, `_BASE_TARGET_`, and `_QUOTE_TARGET_` are computed from `min(baseBalance, quoteBalance / _I_)`, while `_BASE_RESERVE_` and `_QUOTE_RESERVE_` record the full token balances. An unbalanced initial deposit breaks PMM target-reserve invariants, causes swap underflows, and lets an initial depositor steal subsequent LP deposits.

### Technical details

When `MagicLP.buyShares` is called for the first time (`totalSupply() == 0`), lines 381–383 compute the initial `shares`, `_BASE_TARGET_`, and `_QUOTE_TARGET_` using only the smaller side of the deposit relative to the oracle price `_I_`:

```solidity
shares = quoteBalance < DecimalMath.mulFloor(baseBalance, _I_) ? DecimalMath.divFloor(quoteBalance, _I_) : baseBalance;
_BASE_TARGET_ = shares.toUint112();
_QUOTE_TARGET_ = DecimalMath.mulFloor(shares, _I_).toUint112();
```

However, at the end of `buyShares` (line 407), `_setReserve(baseBalance, quoteBalance)` sets `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to the full `baseBalance` and `quoteBalance`.

When `quoteBalance != DecimalMath.mulFloor(baseBalance, _I_)` (which occurs naturally via `Router.createPool` / `Router.createPoolETH` since they do not call `_adjustAddLiquidity`, via pre-existing token balances, or via a malicious initial deposit), two critical accounting failures occur:

1. **Disproportionate Share Minting and Subsequent LP Theft**:
   `totalSupply()` is initialized to `shares = min(baseBalance, quoteBalance / _I_)`, completely ignoring the excess balance on the larger side, while `_BASE_RESERVE_` and `_QUOTE_RESERVE_` record the full balances. On subsequent `buyShares` calls (`totalSupply() > 0`, lines 395–404):
   ```solidity
   uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
   uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
   uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
   shares = DecimalMath.mulFloor(totalSupply(), mintRatio);
   ```
   Because `mintRatio` is bounded by the inflated reserve on the larger side (`quoteInput / quoteReserve` or `baseInput / baseReserve`) and multiplied by the artificially small `totalSupply()`, subsequent liquidity providers depositing at the true price `_I_` receive drastically fewer shares than their contributed value. The initial depositor retains ownership of the majority of `totalSupply()` and can immediately call `sellShares` to withdraw a proportional share of both `baseBalance` and `quoteBalance`, stealing the victim's deposit on the smaller-reserve token.

2. **PMM Target-Reserve Invariant Violation (`_BASE_TARGET_ < _BASE_RESERVE_` or `_QUOTE_TARGET_ < _QUOTE_RESERVE_`)**:
   The pool starts in `_RState_ == RState.ONE` (0) with `_QUOTE_RESERVE_ > _QUOTE_TARGET_` (or `_BASE_RESERVE_ > _BASE_TARGET_`). In DODO/PMM pricing (`Math.sol`), the curve integration requires `V0 >= V1 >= V2 > 0` (`_QUOTE_TARGET_ >= _QUOTE_RESERVE_` when `RState == BELOW_ONE`, and `_BASE_TARGET_ >= _BASE_RESERVE_` when `RState == ABOVE_ONE`). Once a swap transitions `_RState_` from `ONE` to `BELOW_ONE` (or `ABOVE_ONE`) while the reserve still exceeds the target, reverse swaps revert with an arithmetic underflow when computing `state.Q0 - state.Q` (or `state.B0 - state.B`).

### Assumptions and preconditions

1. A `MagicLP` clone has just been initialized (`totalSupply() == 0`), or is being created via `Router.createPool` / `Router.createPoolETH` or `Factory.create`.
2. The initial deposit has `quoteBalance != DecimalMath.mulFloor(baseBalance, _I_)` (either intentionally by an attacker bootstrapping the pool with a small amount of one token and a large amount of the other, or unintentionally by a pool creator supplying unbalanced initial amounts).

### Proof of concept

Consider a `MagicLP` pool where `_BASE_TOKEN_` (MIM, 18 decimals) and `_QUOTE_TOKEN_` (USDB, 18 decimals) have price `_I_ = 1e18` (1 MIM = 1 USDB):

1. **Initial Unbalanced Bootstrap**:
   - An initial depositor transfers `baseBalance = 1e12` wei (`0.000001` MIM) and `quoteBalance = 1,000e18` wei (`1,000` USDB) to the pool and calls `buyShares(attacker)`.
   - `DecimalMath.mulFloor(baseBalance, _I_) = 1e12`. Since `quoteBalance < 1e12` is false, `shares = baseBalance = 1e12`.
   - `_BASE_TARGET_ = 1e12` and `_QUOTE_TARGET_ = 1e12`.
   - `1001` shares are minted to `address(0)` and `1e12 - 1001` shares (99.9999999% of `totalSupply`) are minted to `attacker`.
   - `_setReserve` sets `_BASE_RESERVE_ = 1e12` and `_QUOTE_RESERVE_ = 1,000e18`.

2. **Subsequent LP Deposit**:
   - A second LP deposits `baseInput = 1,000e18` (`1,000` MIM) and `quoteInput = 1,000e18` (`1,000` USDB) and calls `buyShares(victim)`.
   - `baseInputRatio = DecimalMath.divFloor(1,000e18, 1e12) = 1,000e24`.
   - `quoteInputRatio = DecimalMath.divFloor(1,000e18, 1,000e18) = 1e18`.
   - `mintRatio = min(1,000e24, 1e18) = 1e18`.
   - `shares = DecimalMath.mulFloor(1e12, 1e18) = 1e12`.
   - `victim` receives `1e12` shares, bringing `totalSupply()` to `2e12`. The pool now holds `1,000e18 + 1e12` MIM and `2,000e18` USDB.

3. **Redemption and Profit Extraction**:
   - `attacker` calls `sellShares(1e12 - 1001, attacker, 0, 0, "", block.timestamp)`.
   - Because `attacker` holds `(1e12 - 1001) / 2e12 ≈ 50%` of `totalSupply()`, `sellShares` transfers `~500e18` MIM and `~1,000e18` USDB to `attacker`.
   - `attacker` recovers their entire `1,000` USDB plus `500` MIM stolen from `victim`, resulting in an immediate loss of `500` MIM for `victim`.

### Impact assessment

- **Severity**: High
- **Direct Technical Effects**:
  - Subsequent liquidity providers lose up to ~99% of their deposited single-side tokens to the initial depositor when adding balanced liquidity via `buyShares`.
  - Poolcreators supplying unbalanced initial liquidity via `Router.createPool` or `Router.createPoolETH` receive no shares for the excess token amount and initialize the pool with `_BASE_RESERVE_ > _BASE_TARGET_` or `_QUOTE_RESERVE_ > _QUOTE_TARGET_`, causing subsequent swaps to revert or misprice trades.

### Remediation

During initial liquidity provision (`totalSupply() == 0`), ensure that the reserves and targets are initialized to identical balanced values (either by setting `_BASE_TARGET_` and `_QUOTE_TARGET_` to match the actual reserves and valuing both tokens in `shares`, or by refunding any excess above `_BASE_TARGET_` and `_QUOTE_TARGET_` before syncing reserves):

```solidity
        if (totalSupply() == 0) {
            // case 1. initial supply
            if (quoteBalance == 0) {
                revert ErrZeroQuoteAmount();
            }

            shares = quoteBalance < DecimalMath.mulFloor(baseBalance, _I_) ? DecimalMath.divFloor(quoteBalance, _I_) : baseBalance;
            _BASE_TARGET_ = shares.toUint112();
            _QUOTE_TARGET_ = DecimalMath.mulFloor(shares, _I_).toUint112();

            if (_QUOTE_TARGET_ == 0) {
                revert ErrZeroQuoteTarget();
            }

            if (shares <= 2001) {
                revert ErrMintAmountNotEnough();
            }

            uint256 unusedBase = baseBalance - _BASE_TARGET_;
            if (unusedBase > 0) {
                _transferBaseOut(to, unusedBase);
                baseBalance = _BASE_TARGET_;
            }

            uint256 unusedQuote = quoteBalance - _QUOTE_TARGET_;
            if (unusedQuote > 0) {
                _transferQuoteOut(to, unusedQuote);
                quoteBalance = _QUOTE_TARGET_;
            }

            _mint(address(0), 1001);
            shares -= 1001;
```

## Finding 15: Prefunded bootstrap theft in MagicLP.buyShares lets the first caller seize another user’s liquidity

### Synopsis

`MagicLP.buyShares` derives initial LP ownership from the pool’s entire prefunded balances without binding those balances to the caller. A permissionless attacker can front-run initialization after a user transfers both tokens, mint almost all redeemable shares, and withdraw the user’s bootstrap liquidity.

### Technical details

The initial-liquidity branch is selected solely because `totalSupply()` is zero. It does not authenticate the intended initializer or establish who supplied the tokens.

Line 381 calculates `shares` from the contract’s complete base and quote balances. ERC20 balances do not encode who transferred the tokens, so the calculation implicitly awards ownership of every prefunded token to whichever account calls `buyShares` first and chooses the share recipient.

This is particularly dangerous because `buyShares` uses a push-based funding model: it does not pull tokens from the caller. A direct liquidity provider must therefore transfer tokens to the pool before invoking the method. During that interval, any account can call `buyShares(attacker)` without contributing tokens.

The minimum-liquidity mint does not prevent the theft. Lines 393–394 permanently lock 1,001 shares and assign the remaining initial shares to the caller-selected recipient. For a sufficiently large bootstrap deposit, the locked fraction is negligible, allowing the attacker to redeem nearly all of the victim’s assets. The balances and initial price can be perfectly balanced; no reserve-target inconsistency is required.

### Assumptions and preconditions

- An initialized pool exists with zero LP total supply.
- A victim transfers both pool tokens to the pool before invoking `buyShares`.
- The transfer and share-minting operation are not executed atomically through a protected integration.
- The deposited balances produce more than 2,001 initial shares and a nonzero quote target.
- An attacker can observe the pool balances and execute `buyShares` before the victim.
- The attacker can subsequently redeem the LP shares through the normal withdrawal path.

### Proof of concept

1. Consider a fresh pool with equal-decimal tokens, `_I_ = 1e18`, zero reserves, and zero LP supply.
2. The victim transfers 1,000,000 base units and 1,000,000 quote units to the pool, intending to call `buyShares(victim)` afterward.
3. Before the victim’s call executes, the attacker calls `buyShares(attacker)` without transferring either token.
4. Line 381 computes:
   - `mulFloor(1,000,000, 1e18) = 1,000,000`
   - `shares = min(1,000,000, 1,000,000) = 1,000,000`
5. The pool locks 1,001 shares and leaves 998,999 shares for the attacker.
6. The pool records the victim’s token balances as reserves. The victim’s later `buyShares` call has no new base input and reverts.
7. The attacker redeems 998,999 of the 1,000,000 total shares.
8. The attacker receives 998,999 units of each token, while only 1,001 units of each remain backing the permanently locked shares.

The attacker therefore steals approximately 99.8999% of the victim’s balanced bootstrap liquidity.

### Impact assessment

**Severity: High.**

An unprivileged attacker can steal nearly all tokens transferred to an empty pool before its first share mint. The attack requires transaction-ordering access and a non-atomic bootstrap workflow, but requires no capital contribution, oracle manipulation, privileged role, or unbalanced deposit.

The maximum loss approaches the complete prefunded balance of both pool assets, reduced only by the fixed minimum-liquidity fraction. Beyond direct user losses, successful exploitation can make pool creation unsafe through direct contract interaction and undermine integrations that separate token transfers from share minting.

### Remediation

Replace the push-based bootstrap process with an atomic, caller-attributed deposit:

- Accept explicit base and quote amounts and pull them from a specified payer with `transferFrom` during the same transaction.
- Calculate initial shares from the actual balance deltas produced by those transfers, not from arbitrary pre-existing balances.
- Define a safe policy for pre-existing balances, such as authorized recovery or permanently protocol-owned liquidity; never award them to the next caller.
- Bind the intended recipient, amounts, pool address, deadline, and nonce to the bootstrap operation.
- Preserve the minimum-liquidity lock after establishing correct depositor attribution.

Alternatively, retain push funding only if first initialization is restricted to a factory-controlled, one-time bootstrap operation that funds and mints atomically. A signed bootstrap authorization can also permit relaying while fixing the beneficiary so a relayer cannot redirect shares.

## Finding 16: Base-unit LP shares in MagicLP.buyShares make minimum liquidity decimal-dependent

### Synopsis

`MagicLP.buyShares` derives LP shares in raw base-token precision but applies fixed 2,001/1,001 bootstrap constants. For low-decimal base assets, initialization near the threshold permanently assigns half of meaningful starting liquidity to `address(0)`; an equivalent 18-decimal deposit loses negligible value.

### Technical details

During initial liquidity provision, line 381 calculates `shares` as a base-token-denominated quantity: either `baseBalance` directly or `quoteBalance` converted into base-token units. Consequently, the precision of `shares` follows the base token’s decimals rather than a canonical LP-token precision.

The subsequent logic nevertheless treats `shares` as having a universal scale:

- Initial liquidity is accepted when `shares > 2001`.
- Exactly 1,001 shares are minted to `address(0)`.
- The remaining shares are minted to the initial provider.

For the minimum passing value of 2,002 shares, 1,001 shares—exactly 50% of the total supply—are permanently assigned to the zero address. Because the numerical value of 2,002 depends on the base token’s decimals, the economic consequences vary drastically:

- With a 2-decimal base token, 2,002 units represent 20.02 tokens, so 10.01 tokens’ proportional value becomes permanently inaccessible.
- With an 18-decimal base token, the same 20.02-token deposit produces `20.02e18` shares, and only 1,001 atomic shares are locked.

The issue does not require a conversion-rounding remainder. Exact, balanced deposits still exhibit the discrepancy. The fixed bootstrap constants are therefore not expressed in the same canonical precision across pools.

### Assumptions and preconditions

- The protocol supports at least one pool whose base token has fewer than 18 decimals.
- The pool is uninitialized and has zero LP total supply.
- Initial balances produce a raw base-denominated share amount close to, but greater than, 2,001.
- The underlying tokens perform ordinary balance-preserving transfers.
- Shares minted to `address(0)` cannot be transferred or redeemed through another recovery mechanism.
- No external system compensates the initial provider for the value represented by the locked shares.

### Proof of concept

1. Create an uninitialized pool whose base and quote tokens both use 2 decimals.
2. Configure `_I_ = 1e18`, representing a 1:1 raw-unit conversion.
3. Transfer 2,002 base units and 2,002 quote units to the pool. Each balance represents 20.02 tokens.
4. Call `buyShares(initialProvider)`.
5. The comparison at line 381 is exact:
   - `DecimalMath.mulFloor(2002, 1e18) = 2002`.
   - Therefore, `shares = 2002`.
6. The check at line 389 succeeds because `2002 > 2001`.
7. Line 393 mints 1,001 shares to `address(0)`.
8. Line 394 reduces the provider’s amount to 1,001 shares, which are then minted to the provider.
9. The pool now has 2,002 total shares, but the provider controls only 1,001, exactly 50%.
10. Redeeming all provider-owned shares returns only 1,001 base units and 1,001 quote units. The other 10.01 units of each token remain represented by the inaccessible zero-address shares.
11. As a control, repeat with 18-decimal tokens and the same 20.02-token deposit. The pool derives `20.02e18` shares, so locking 1,001 atomic shares removes only approximately `5e-17` of the supply rather than 50%.

### Impact assessment

**Severity: Medium.**

A low-decimal pool initialized near the fixed threshold can permanently strand up to approximately 50% of the initial provider’s assets. Deposits producing 2,001 or fewer raw shares also cannot initialize the pool, even when those atomic units represent economically substantial whole-token amounts.

The issue is most consequential for low-decimal, high-value assets and pool-creation integrations that do not expose the decimal-dependent lock fraction. In the worst supported configuration, substantial base and quote balances become permanently unavailable to liquidity providers, damaging pool usability and causing direct capital loss.

### Remediation

Decouple the raw base target from the LP-share denomination:

1. Calculate a `baseTarget` in raw base-token units.
2. Store `_BASE_TARGET_` and derive `_QUOTE_TARGET_` from that raw target.
3. Independently normalize the initial LP `shares` to the LP token’s canonical precision, preferably 18 decimals.
4. Apply the 2,001 minimum and 1,001 lock only after normalization.
5. Use checked, full-precision scaling and explicitly constrain supported token-decimal ranges.
6. Update pool previews and tests to cover 0-, 2-, 6-, and 18-decimal base tokens.

If LP shares must remain base-token-denominated, an alternative is to derive the minimum and locked amounts from base-token decimals and ensure all mint guards use the same normalized unit. Restricting supported base assets to a single decimal precision is a simpler but less flexible alternative.

## Finding 17: Share Valuation Distortion and Reserve Ratio Manipulation in `MagicLP.buyShares`

### Synopsis

In `MagicLP.buyShares`, share minting during normal operation computes `mintRatio` as the minimum of `baseInputRatio` and `quoteInputRatio` before multiplying by `totalSupply()`, while absorbing the full balances of both tokens into reserves. This enables donation-driven share valuation distortion and reserve-ratio manipulation to steal depositor funds or DoS liquidity provision.

### Technical details

In `src/mimswap/MagicLP.sol`, the `buyShares` function calculates the LP shares to mint when `totalSupply() > 0` (`case 2. normal case`, lines 395–407) as follows:

```solidity
        } else if (baseReserve > 0 && quoteReserve > 0) {
            // case 2. normal case
            uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
            uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
            uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
            shares = DecimalMath.mulFloor(totalSupply(), mintRatio);

            _BASE_TARGET_ = (uint256(_BASE_TARGET_) + DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)).toUint112();
            _QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)).toUint112();
        }

        _mint(to, shares);
        _setReserve(baseBalance, quoteBalance);
```

This design contains two compounding flaws in how share valuation and reserve accounting are handled:

1. **Disproportionate Deposit Absorption via Reserve-Ratio Skewing (`min(baseInputRatio, quoteInputRatio)` with Full Reserve Update):**
   `mintRatio` is determined solely by the smaller of `baseInputRatio` (`(baseInput * 1e18) / baseReserve`) and `quoteInputRatio` (`(quoteInput * 1e18) / quoteReserve`). However, at line 407, `_setReserve(baseBalance, quoteBalance)` records the entire `baseBalance` and `quoteBalance` (including any excess above `mintRatio`) into `_BASE_RESERVE_` and `_QUOTE_RESERVE_` without refunding the excess input to the depositor. If the reserve ratio is skewed prior to `buyShares`—for example, by an attacker donating one token and calling `flashLoan(0, 0, ...)` (which invokes `_sync()`) or by initializing a pool with a small deposit and donating a single token—the limiting ratio becomes arbitrarily small while 100% of the other token contributed by the victim is absorbed into the pool reserves and redistributed to existing share holders.

2. **Donation-Driven Share Inflation and Intermediate Division Truncation:**
   During the initial deposit (`totalSupply() == 0`), the contract mints `1001` dead shares to `address(0)` and requires the initial depositor to mint at least `1001` shares (`totalSupply() >= 2002`). Furthermore, `_mint` (line 594) reverts with `ErrMintAmountNotEnough()` whenever `shares <= 1000`. In subsequent deposits, `buyShares` first divides `baseInput` by `baseReserve` (`DecimalMath.divFloor(baseInput, baseReserve)`) and then multiplies `totalSupply()` by `mintRatio` (`DecimalMath.mulFloor(totalSupply(), mintRatio)`).
   - If an attacker mints the minimum initial liquidity (`totalSupply() = 2002`) and donates tokens directly to the pool followed by `_sync()`, `baseReserve` and/or `quoteReserve` become large while `totalSupply()` remains `2002`.
   - Any subsequent deposit where `mintRatio < (1001 * 1e18) / 2002` (i.e., `baseInput` or `quoteInput` is less than ~50% of the donated reserve) results in `shares <= 1000`, causing `_mint` to revert with `ErrMintAmountNotEnough()`.
   - Even when a subsequent depositor provides enough liquidity to exceed `1000` shares, truncation in `mintRatio` and asymmetrical reserve inflation allow the attacker to capture a disproportionate share of the victim's deposit via `sellShares`.

### Assumptions and preconditions

1. A `MagicLP` (or `BlastMagicLP`) pool clone has been deployed and initialized.
2. Either an attacker is the first liquidity provider (depositing the minimum `2002` wei to establish `totalSupply() = 2002` and then donating tokens to skew or inflate reserves), or an attacker front-runs a liquidity provider's transaction that supplies `baseInput` and `quoteInput` to `buyShares` without tight minimum-share slippage enforcement.

### Proof of concept

Consider a pool initialized with `_I_ = 1e18` where `baseToken` and `quoteToken` both have 18 decimals:

1. **Initial Minimal Deposit:**
   An attacker transfers `2002` wei of `baseToken` and `2002` wei of `quoteToken` to the pool and calls `buyShares(attacker)`.
   - `1001` shares are minted to `address(0)` and `1001` shares are minted to `attacker`.
   - `totalSupply() = 2002`, `_BASE_RESERVE_ = 2002`, `_QUOTE_RESERVE_ = 2002`.

2. **Asymmetrical Reserve Donation & Sync:**
   Suppose a user intends to deposit `1000e18` `baseToken` and `1000e18` `quoteToken` into the pool (or the attacker front-runs the deposit).
   - The attacker transfers `1000e18` of `baseToken` directly to the pool and triggers `_sync()` (e.g., via `flashLoan(0, 0, attacker, "")`).
   - Now `_BASE_RESERVE_ = 1000e18 + 2002` and `_QUOTE_RESERVE_ = 2002`, while `totalSupply() = 2002`.

3. **Victim Liquidity Provision (`buyShares`):**
   The victim transfers `1000e18` of `baseToken` and `1000e18` of `quoteToken` to the pool and calls `buyShares(victim)`:
   - `baseInputRatio = DecimalMath.divFloor(1000e18, 1000e18 + 2002) = 999999999999999997` (~`1e18`).
   - `quoteInputRatio = DecimalMath.divFloor(1000e18, 2002) = 499500499500499500499500499500499500`.
   - `mintRatio = min(baseInputRatio, quoteInputRatio) = 999999999999999997`.
   - `shares = DecimalMath.mulFloor(2002, 999999999999999997) = 2001`.
   - `_setReserve(baseBalance, quoteBalance)` sets `_BASE_RESERVE_ = 2000e18 + 2002` and `_QUOTE_RESERVE_ = 1000e18 + 2002`.
   - Note also that if the victim had deposited `< 500.5e18` of `baseToken`, `mintRatio` would be `< 0.5005e18`, yielding `shares <= 1000` and causing `buyShares` to revert with `ErrMintAmountNotEnough()`.

4. **Value Extraction (`sellShares`):**
   - Total supply is now `2002 + 2001 = 4003` shares. The attacker holds `1001` shares (~25% of total supply).
   - The attacker calls `sellShares(1001, attacker, 0, 0, "", block.timestamp)` and receives `~500e18` `baseToken` and `~250e18` `quoteToken` (plus the attacker can also swap against the skewed PMM targets/reserves before withdrawing to extract the remaining donated `baseToken` and stolen `quoteToken`).

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  1. **Theft of Depositor Funds (Disproportionate Deposit Value Redistribution):** When `baseInputRatio != quoteInputRatio` due to reserve manipulation or front-running, the excess token input is absorbed into the pool's reserves without minting corresponding shares or refunding the excess to the depositor, transferring value directly to existing LP share holders.
  2. **Denial of Service on Liquidity Provision:** By inflating reserves via donation when `totalSupply()` is small, an attacker forces `DecimalMath.mulFloor(totalSupply(), mintRatio)` to evaluate to `<= 1000` for normal deposit sizes, causing `_mint` to revert with `ErrMintAmountNotEnough()`.

### Remediation

1. Calculate minted shares directly as `Math.min((baseInput * totalSupply()) / baseReserve, (quoteInput * totalSupply()) / quoteReserve)` to avoid intermediate fixed-point truncation in `mintRatio`.
2. Only absorb token inputs proportional to the actual `mintRatio` into `_BASE_RESERVE_` and `_QUOTE_RESERVE_` (or refund any excess `baseInput` / `quoteInput` above the limiting ratio back to the depositor), and enforce a higher minimum initial share lock (e.g., `1e6` dead shares or virtual shares/reserves) so that single-wei initial deposits cannot cheaply distort share valuation.

```solidity
        } else if (baseReserve > 0 && quoteReserve > 0) {
            // case 2. normal case
            uint256 totalShares = totalSupply();
            uint256 baseShares = (baseInput * totalShares) / baseReserve;
            uint256 quoteShares = (quoteInput * totalShares) / quoteReserve;
            shares = quoteShares < baseShares ? quoteShares : baseShares;

            uint256 mintRatio = DecimalMath.divFloor(shares, totalShares);
            uint256 usedBaseInput = DecimalMath.mulCeil(baseReserve, mintRatio);
            uint256 usedQuoteInput = DecimalMath.mulCeil(quoteReserve, mintRatio);

            _BASE_TARGET_ = (uint256(_BASE_TARGET_) + DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)).toUint112();
            _QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)).toUint112();

            if (baseInput > usedBaseInput) {
                _transferBaseOut(to, baseInput - usedBaseInput);
                baseBalance = _BASE_TOKEN_.balanceOf(address(this));
            }
            if (quoteInput > usedQuoteInput) {
                _transferQuoteOut(to, quoteInput - usedQuoteInput);
                quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
            }
        }

        _mint(to, shares);
        _setReserve(baseBalance, quoteBalance);
```

## Finding 18: Precision Loss via Intermediate Floor Division in `MagicLP.buyShares` Under-Mints LP Shares and Under-Updates Targets

### Synopsis

In `MagicLP.buyShares`, `baseInputRatio` and `quoteInputRatio` are computed via `DecimalMath.divFloor` with `1e18` precision before being multiplied by `totalSupply()`, `_BASE_TARGET_`, and `_QUOTE_TARGET_`. This premature truncation causes significant precision loss, under-minting LP shares and permanently leaking value from new depositors to existing share holders.

### Technical details

In `MagicLP.buyShares` (lines 395–406 of `src/mimswap/MagicLP.sol`), when adding liquidity to an already initialized pool (`totalSupply() > 0` and `baseReserve > 0 && quoteReserve > 0`), the contract calculates the number of shares to mint and the target adjustments as follows:

```solidity
uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
shares = DecimalMath.mulFloor(totalSupply(), mintRatio);

_BASE_TARGET_ = (uint256(_BASE_TARGET_) + DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)).toUint112();
_QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)).toUint112();
```

`DecimalMath.divFloor(a, b)` computes `(a * 1e18) / b`, and `DecimalMath.mulFloor(a, b)` computes `(a * b) / 1e18`. Performing division before multiplication introduces intermediate rounding error of up to `1` unit in `1e18` (`mintRatio` is truncated to a multiple of `1e-18`).

When `totalSupply()`, `baseReserve`, or `quoteReserve` are larger than `1e18` (which is standard for 18-decimal tokens whenever pool reserves exceed 1 whole token, or when an attacker/whale inflates `baseReserve` and `totalSupply`), this truncation error gets multiplied by `totalSupply() / 1e18`:

1. **Zero or Severely Under-Minted Shares for Small/Moderate Deposits**: If `baseReserve > baseInput * 1e18` (for example, a large pool or a pool with high-decimal/low-unit-value tokens where a deposit is less than `1e-18` of total reserves, or where `baseInput * 1e18 / baseReserve` has a significant remainder), `mintRatio` rounds down significantly. Even if the true mathematical share amount `(baseInput * totalSupply()) / baseReserve` is well above the minimum mint threshold (`1000`), `mintRatio` can round down to `0` (causing `_mint` to revert with `ErrMintAmountNotEnough`) or lose up to `totalSupply() / 1e18` wei of shares per deposit.
2. **Disproportionate Deposit Value Redistribution**: Because `_setReserve(baseBalance, quoteBalance)` absorbs the entire `baseInput` and `quoteInput` into the pool reserves while minting fewer shares than `(input * totalSupply) / reserve`, the un-minted share value is redistributed to existing LP share holders at the expense of the depositor. If an adversary temporarily provides large liquidity (e.g., `baseReserve = 1e24`, `totalSupply = 1e24`), every subsequent depositor loses up to `1e6` wei of shares due to `mintRatio` floor truncation, which the adversary immediately captures upon calling `sellShares`.
3. **Permanent Target Drift**: `_BASE_TARGET_` and `_QUOTE_TARGET_` are similarly updated using `DecimalMath.mulFloor(target, mintRatio)`, rounding down twice (once in `mintRatio`, once in `mulFloor`), while `sellShares` rounds up the target reduction via `divCeil`. Repeated deposits and withdrawals systematically erode `_BASE_TARGET_` and `_QUOTE_TARGET_` relative to reserves.

### Assumptions and preconditions

1. An initialized `MagicLP` pool exists with `totalSupply() > 0`, `_BASE_RESERVE_ > 0`, and `_QUOTE_RESERVE_ > 0`.
2. The pool's `totalSupply()` and reserves exceed `1e18` (e.g., standard 18-decimal ERC20 tokens with more than 1 token in liquidity).
3. A user calls `buyShares` (directly or via `Router`) to provide liquidity.

### Proof of concept

1. Suppose a `MagicLP` pool has `_I_ = 1e18`, `baseReserve = 2000e18`, `quoteReserve = 2000e18`, and `totalSupply = 2000e18`.
2. A liquidity provider transfers `baseInput = 1.999999999999999999e18` (`2e18 - 1`) and `quoteInput = 1.999999999999999999e18` (`2e18 - 1`) to the pool and calls `buyShares(user)`.
3. Mathematically, since `totalSupply == baseReserve == quoteReserve`, the user should receive `(baseInput * totalSupply) / baseReserve = 1.999999999999999999e18` shares.
4. However, `DecimalMath.divFloor(baseInput, baseReserve)` computes:
   `(1999999999999999999 * 1e18) / 2000e18 = 999999999999999` (`0.000999999999999999e18`).
5. Then `shares = DecimalMath.mulFloor(2000e18, 999999999999999) = 1999999999999998000` (`1.999999999999998e18`).
6. The user loses `1999` wei of shares (nearly `2 * (totalSupply / 1e18)`), while the full `1.999999999999999999e18` of both tokens are added to `_BASE_RESERVE_` and `_QUOTE_RESERVE_`.
7. If the pool instead has `10,000,000e18` (`1e25`) in reserves and `totalSupply = 1e25`, any deposit loss is magnified up to `1e7` wei per deposit, or if a user deposits `0.005e18` (`5e15` wei, which is `5,000,000,000,000,000` units, well above the `1000` minimum share requirement), `baseInputRatio = (5e15 * 1e18) / 1e25 = 0`, causing `shares = 0` and reverting the transaction with `ErrMintAmountNotEnough()`.

### Impact assessment

- **Severity**: Medium
- **Direct Technical Effect**: `buyShares` systematically under-mints LP shares and under-increments `_BASE_TARGET_` and `_QUOTE_TARGET_` whenever `totalSupply()` or reserves exceed `1e18`, and completely reverts valid deposits when `input * 1e18 < reserve` even though `(input * totalSupply) / reserve > 1000`.
- **Business Implications**: Liquidity providers receive fewer shares than their proportional contribution to the pool, immediately losing the truncated portion of their deposit to existing LPs, and smaller deposits into large pools are bricked.

### Remediation

Avoid intermediate `1e18`-scaled division (`divFloor` followed by `mulFloor`) when calculating `shares` and updating `_BASE_TARGET_` and `_QUOTE_TARGET_`. Instead, multiply before dividing (or use full-precision `mulDiv`) and determine the limiting side directly:

```solidity
        } else if (baseReserve > 0 && quoteReserve > 0) {
            // case 2. normal case
            uint256 totalShares = totalSupply();
            if (quoteInput * baseReserve < baseInput * quoteReserve) {
                shares = (quoteInput * totalShares) / quoteReserve;
                _BASE_TARGET_ = (uint256(_BASE_TARGET_) + (uint256(_BASE_TARGET_) * quoteInput) / quoteReserve).toUint112();
                _QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + (uint256(_QUOTE_TARGET_) * quoteInput) / quoteReserve).toUint112();
            } else {
                shares = (baseInput * totalShares) / baseReserve;
                _BASE_TARGET_ = (uint256(_BASE_TARGET_) + (uint256(_BASE_TARGET_) * baseInput) / baseReserve).toUint112();
                _QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + (uint256(_QUOTE_TARGET_) * baseInput) / baseReserve).toUint112();
            }
        }
```

## Finding 19: Disproportionate Reserve and Target Updates in `MagicLP.buyShares` Violate PMM Reserve-to-Target Invariants

### Synopsis

In `MagicLP.buyShares`, `_BASE_TARGET_` and `_QUOTE_TARGET_` are scaled by `mintRatio` (the minimum of the base and quote input ratios), whereas `_setReserve` updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to the full token balances without adjusting `_RState_` or refunding excess inputs, corrupting PMM state invariants and breaking swaps.

### Technical details

`MagicLP` implements a Proactive Market Maker (PMM) model that tracks current reserves (`_BASE_RESERVE_`, `_QUOTE_RESERVE_`), equilibrium targets (`_BASE_TARGET_`, `_QUOTE_TARGET_`), and the pricing regime (`_RState_`). The PMM pricing logic relies on strict consistency between the active `_RState_` and the relationship between reserves and targets:

- When `_RState_ == RState.ONE`: `_BASE_RESERVE_ == _BASE_TARGET_` and `_QUOTE_RESERVE_ == _QUOTE_TARGET_`.
- When `_RState_ == RState.ABOVE_ONE`: `_BASE_RESERVE_ < _BASE_TARGET_` and `_QUOTE_RESERVE_ > _QUOTE_TARGET_`.
- When `_RState_ == RState.BELOW_ONE`: `_BASE_RESERVE_ > _BASE_TARGET_` and `_QUOTE_RESERVE_ < _QUOTE_TARGET_`.

When adding liquidity to an already initialized pool (`totalSupply() > 0`) in `MagicLP.buyShares` (`src/mimswap/MagicLP.sol`, lines 395–407):

```solidity
        } else if (baseReserve > 0 && quoteReserve > 0) {
            // case 2. normal case
            uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
            uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
            uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
            shares = DecimalMath.mulFloor(totalSupply(), mintRatio);

            _BASE_TARGET_ = (uint256(_BASE_TARGET_) + DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)).toUint112();
            _QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)).toUint112();
        }

        _mint(to, shares);
        _setReserve(baseBalance, quoteBalance);
```

Because `MagicLP.buyShares` is permissionless and computes `baseInput = baseBalance - baseReserve` and `quoteInput = quoteBalance - quoteReserve` from the contract's raw ERC20 balances, `baseInputRatio` and `quoteInputRatio` do not have to be equal.

1. `_BASE_TARGET_` and `_QUOTE_TARGET_` are increased proportionally by `mintRatio = min(baseInputRatio, quoteInputRatio)` (lines 402–403).
2. However, `_setReserve(baseBalance, quoteBalance)` (line 407) updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to the full `baseBalance` and `quoteBalance`, absorbing the entire excess of the larger input into the pool reserves without updating `_RState_` or recalculating targets.

This breaks PMM state consistency in several ways:

- **Underflow DoS in `ABOVE_ONE` / `BELOW_ONE` regimes:**
  - When `_RState_ == RState.ABOVE_ONE` (`_BASE_RESERVE_ < _BASE_TARGET_`), if an imbalanced `buyShares` call provides excess base tokens such that the new `baseBalance` exceeds the updated `_BASE_TARGET_`, `_BASE_RESERVE_` becomes greater than `_BASE_TARGET_` while `_RState_` remains `RState.ABOVE_ONE`. In `PMMPricing.sellBaseToken`, computing `backToOnePayBase = state.B0 - state.B` (`_BASE_TARGET_ - _BASE_RESERVE_`) reverts with an arithmetic underflow.
  - Symmetrically, when `_RState_ == RState.BELOW_ONE` (`_QUOTE_RESERVE_ < _QUOTE_TARGET_`), providing excess quote tokens such that `quoteBalance > _QUOTE_TARGET_` causes `_QUOTE_RESERVE_ > _QUOTE_TARGET_` while `_RState_` remains `RState.BELOW_ONE`. In `PMMPricing.sellQuoteToken`, computing `backToOnePayQuote = state.Q0 - state.Q` (`_QUOTE_TARGET_ - _QUOTE_RESERVE_`) reverts with an arithmetic underflow.
- **Inconsistent state transitions from `RState.ONE`:**
  - When `_RState_ == RState.ONE`, excess quote (or base) tokens push `_QUOTE_RESERVE_ > _QUOTE_TARGET_` (or `_BASE_RESERVE_ > _BASE_TARGET_`) while `_RState_` stays `RState.ONE`. A subsequent `sellBase` (or `sellQuote`) transitions `_RState_` to `BELOW_ONE` (or `ABOVE_ONE`) while still having `_QUOTE_RESERVE_ > _QUOTE_TARGET_` (or `_BASE_RESERVE_ > _BASE_TARGET_`), causing subsequent swaps in the opposite direction to underflow.

### Assumptions and preconditions

1. A `MagicLP` clone pool has been initialized and has existing liquidity (`totalSupply() > 0`, `_BASE_RESERVE_ > 0`, `_QUOTE_RESERVE_ > 0`).
2. A caller (or an attacker donating tokens prior to a `buyShares` call) supplies base and quote tokens in a ratio that differs from `_BASE_RESERVE_ : _QUOTE_RESERVE_`, with the smaller input ratio still sufficient to mint at least `1001` wei of LP shares (`_mint` requires `amount > 1000`).

### Proof of concept

Consider an initialized `MagicLP` pool with:
- `_BASE_RESERVE_ = 1,000e18`, `_BASE_TARGET_ = 1,000e18`
- `_QUOTE_RESERVE_ = 1,000e18`, `_QUOTE_TARGET_ = 1,000e18`
- `totalSupply() = 1,000e18`
- `_RState_ = RState.ONE`

1. **Transition pool slightly into `RState.ABOVE_ONE`:**
   - A small `sellQuote` swap is executed (e.g., selling `1e6` wei of quote token for roughly `1e6` wei of base token).
   - Pool state is now `_RState_ = RState.ABOVE_ONE`, with `_BASE_RESERVE_ = 1,000e18 - 1e6` and `_BASE_TARGET_ = 1,000e18` (`_BASE_RESERVE_ < _BASE_TARGET_` by `1e6` wei).

2. **Imbalanced `buyShares` call:**
   - A caller transfers `quoteInput = 1,001,000` wei and `baseInput = 3,000,000` wei to the `MagicLP` pool and calls `buyShares(caller)`.
   - `quoteInputRatio = (1,001,000 * 1e18) / _QUOTE_RESERVE_ = 1001`.
   - `baseInputRatio = (3,000,000 * 1e18) / _BASE_RESERVE_ = 3000`.
   - `mintRatio = min(1001, 3000) = 1001`.
   - `shares = (1,000e18 * 1001) / 1e18 = 1001` (passes `amount > 1000` check in `_mint`).
   - `_BASE_TARGET_` increases by `(1,000e18 * 1001) / 1e18 = 1,001,000` wei, becoming `1,000e18 + 1,001,000`.
   - `_setReserve(baseBalance, quoteBalance)` sets `_BASE_RESERVE_` to `(1,000e18 - 1e6) + 3,000,000 = 1,000e18 + 2,000,000`.

3. **Resulting broken state:**
   - `_RState_` is still `RState.ABOVE_ONE`, which requires `_BASE_RESERVE_ < _BASE_TARGET_`.
   - However, `_BASE_RESERVE_` (`1,000e18 + 2,000,000`) is now strictly greater than `_BASE_TARGET_` (`1,000e18 + 1,001,000`).
   - Any subsequent call to `sellBase` attempts to compute `state.B0 - state.B` (`_BASE_TARGET_ - _BASE_RESERVE_`) and reverts with an arithmetic underflow.

### Impact assessment

- **Severity:** High / Medium
- **Direct Technical Effect:** The PMM invariant between `_BASE_RESERVE_`, `_QUOTE_RESERVE_`, `_BASE_TARGET_`, `_QUOTE_TARGET_`, and `_RState_` is violated. Swaps (`sellBase` or `sellQuote`) revert due to arithmetic underflow, and the pool's pricing curve is distorted whenever excess tokens are absorbed into reserves without corresponding target/state updates.
- **Business Implications:** Because `deficit = _BASE_TARGET_ - _BASE_RESERVE_` after a small swap can be just a few thousand wei, an attacker can break the pool's swap functionality at negligible cost (fractions of a cent), resulting in denial of service for traders and LPs until an administrator intervenes.

### Remediation

In `MagicLP.buyShares`, ensure that only proportional token inputs matching `mintRatio` are added to the reserves (and refund any excess tokens to `to`), or require that inputs are proportional so that reserves and targets always scale by the same `mintRatio`.

```solidity
        } else if (baseReserve > 0 && quoteReserve > 0) {
            // case 2. normal case
            uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
            uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
            uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
            shares = DecimalMath.mulFloor(totalSupply(), mintRatio);

            uint256 usedBaseInput = DecimalMath.mulCeil(baseReserve, mintRatio);
            uint256 usedQuoteInput = DecimalMath.mulCeil(quoteReserve, mintRatio);

            _BASE_TARGET_ = (uint256(_BASE_TARGET_) + DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)).toUint112();
            _QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)).toUint112();

            if (baseInput > usedBaseInput) {
                _transferBaseOut(to, baseInput - usedBaseInput);
            }
            if (quoteInput > usedQuoteInput) {
                _transferQuoteOut(to, quoteInput - usedQuoteInput);
            }

            baseBalance = _BASE_TOKEN_.balanceOf(address(this));
            quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
        }

        _mint(to, shares);
        _setReserve(baseBalance, quoteBalance);
```

## Finding 20: Unhandled Depleted-Reserve State in `MagicLP.buyShares` Permanently Bricks Liquidity Provision

### Synopsis

In `MagicLP.buyShares`, when `totalSupply() > 0` and either `_BASE_RESERVE_` or `_QUOTE_RESERVE_` is depleted to `0`, execution skips both the initial-supply branch and the `else if (baseReserve > 0 && quoteReserve > 0)` branch. Consequently, `shares` remains `0` and `_mint` unconditionally reverts, permanently disabling liquidity deposits.

### Technical details

During the first liquidity deposit in `MagicLP.buyShares`, `1001` shares are permanently minted to `address(0)` (line 393), ensuring that `totalSupply()` remains strictly greater than zero (`totalSupply() >= 1001`) for the entire remaining lifetime of the pool.

For all subsequent calls to `buyShares(address to)`, the function evaluates:
```solidity
if (totalSupply() == 0) {
    ...
} else if (baseReserve > 0 && quoteReserve > 0) {
    // case 2. normal case
    ...
}

_mint(to, shares);
_setReserve(baseBalance, quoteBalance);
```

Because `init()` permits `_K_ = 0` (`if (k > MAX_K) revert ErrInvalidK()`), a pool configured with a constant-price curve (`_K_ == 0`) allows traders to completely deplete either `_BASE_RESERVE_` or `_QUOTE_RESERVE_` down to `0`. Furthermore, reserve depletion on one side can also occur if small residual balances are drained across swaps and share redemptions while the `1001` dead shares remain in `totalSupply()`.

Once either `baseReserve == 0` or `quoteReserve == 0` while `totalSupply() > 0`:
1. The `if (totalSupply() == 0)` condition evaluates to `false`.
2. The `else if (baseReserve > 0 && quoteReserve > 0)` condition at line 395 also evaluates to `false`.
3. There is no fallback `else` branch to compute `shares` or update `_BASE_TARGET_` and `_QUOTE_TARGET_`, leaving `shares` at its default value of `0`.
4. `_mint(to, 0)` is called at line 406, which checks `if (amount <= 1000) revert ErrMintAmountNotEnough();` (line 594) and unconditionally reverts.

Even if a user transfers both base and quote tokens directly to the pool to restore balances, `buyShares` reads the stored `_BASE_RESERVE_` and `_QUOTE_RESERVE_` before updating them, so `buyShares` will still revert. If someone calls `_sync()` (e.g., via `flashLoan` or `sellShares`) after donating tokens to make both reserves non-zero without minting shares, the donated tokens are absorbed into existing shares (including the `1001` dead shares at `address(0)`), forcing the depositor to forfeit funds just to unbrick `buyShares`.

### Assumptions and preconditions

1. A `MagicLP` clone has been initialized and received its initial liquidity deposit, so `totalSupply() >= 1001`.
2. Either `_BASE_RESERVE_` or `_QUOTE_RESERVE_` reaches `0` (for example, via a swap that buys the entire reserve of one side when `_K_ == 0`, or when a nearly empty pool has one reserve rounded down to `0`).

### Proof of concept

1. A pool is initialized with `baseToken`, `quoteToken`, and `_K_ = 0`, `_I_ = 1e18`.
2. Alice provides initial liquidity of `1000e18` `baseToken` and `1000e18` `quoteToken` via `buyShares(alice)`. `totalSupply()` becomes `1000e18`, `_BASE_RESERVE_ = 1000e18`, and `_QUOTE_RESERVE_ = 1000e18`.
3. A trader swaps enough `baseToken` into the pool via `sellBase` to purchase the entire `1000e18` `quoteToken` reserve. After the swap, `_QUOTE_RESERVE_` is updated to `0` while `_BASE_RESERVE_ > 0` and `totalSupply() == 1000e18`.
4. Bob attempts to add liquidity by transferring `baseToken` and `quoteToken` to the pool and calling `buyShares(bob)`.
5. Inside `buyShares(bob)`:
   - `totalSupply() == 0` is `false` (`totalSupply() == 1000e18`).
   - `baseReserve > 0 && quoteReserve > 0` is `false` (`quoteReserve == 0`).
   - `shares` remains `0`.
   - `_mint(bob, 0)` reverts with `ErrMintAmountNotEnough()`.

### Impact assessment

**Severity: Medium**

Any `MagicLP` pool whose base or quote reserve reaches `0` enters a broken state where `buyShares` always reverts with `ErrMintAmountNotEnough()`. Liquidity providers cannot add liquidity through standard `Router` or `MagicLP.buyShares` flows unless they donate uncompensated tokens directly to the pool and trigger a reserve sync (forfeiting those tokens to existing shareholders and burned shares).

### Remediation

Handle the single-sided depleted reserve case in `MagicLP.buyShares` (when `totalSupply() > 0` and one of `baseReserve` or `quoteReserve` is `0`, or if both reserves are `0`) by calculating `mintRatio` from the non-zero reserve (or re-initializing targets proportionally when both reserves are `0`):

```solidity
        } else if (baseReserve > 0 && quoteReserve > 0) {
            // case 2. normal case
            uint256 baseInputRatio = DecimalMath.divFloor(baseInput, baseReserve);
            uint256 quoteInputRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
            uint256 mintRatio = quoteInputRatio < baseInputRatio ? quoteInputRatio : baseInputRatio;
            shares = DecimalMath.mulFloor(totalSupply(), mintRatio);

            _BASE_TARGET_ = (uint256(_BASE_TARGET_) + DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)).toUint112();
            _QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)).toUint112();
        } else if (baseReserve > 0) {
            uint256 mintRatio = DecimalMath.divFloor(baseInput, baseReserve);
            shares = DecimalMath.mulFloor(totalSupply(), mintRatio);

            _BASE_TARGET_ = (uint256(_BASE_TARGET_) + DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)).toUint112();
            _QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)).toUint112();
        } else if (quoteReserve > 0) {
            uint256 mintRatio = DecimalMath.divFloor(quoteInput, quoteReserve);
            shares = DecimalMath.mulFloor(totalSupply(), mintRatio);

            _BASE_TARGET_ = (uint256(_BASE_TARGET_) + DecimalMath.mulFloor(uint256(_BASE_TARGET_), mintRatio)).toUint112();
            _QUOTE_TARGET_ = (uint256(_QUOTE_TARGET_) + DecimalMath.mulFloor(uint256(_QUOTE_TARGET_), mintRatio)).toUint112();
        }
```

## Finding 21: Zero PMM target persists through MagicLP.buyShares and prevents cross-decimal pools from recovering

### Synopsis

`MagicLP.buyShares` treats positive dust reserves with a zero PMM target as a normal pool. Because target scaling is multiplicative, new liquidity cannot restore the target, preserving invalid accounting and potentially disabling or corrupting swaps in cross-decimal pools.

### Technical details

The normal-liquidity branch at line 395 checks only that both reserves are positive. It does not verify that `_BASE_TARGET_` and `_QUOTE_TARGET_` are also positive, despite initialization explicitly requiring a nonzero quote target.

A zero target is reachable after a near-complete withdrawal in a correctly configured cross-decimal pool. Consider an 18-decimal base token, a 6-decimal quote token, and `_I_ = 1_000_000`:

1. Initial balances of `1e21` base units and `1e9` quote units produce:
   - `totalSupply = 1e21`
   - `_BASE_TARGET_ = 1e21`
   - `_QUOTE_TARGET_ = 1e9`
2. The initial provider owns `1e21 - 1001` shares because 1001 shares are permanently minted to `address(0)`.
3. Redeeming all provider-owned shares leaves:
   - `totalSupply = 1001`
   - base reserve and base target equal to `1001`
   - quote reserve equal to one raw unit
   - quote target equal to zero because target reduction rounds upward while the remaining quote balance rounds to one raw unit.

When liquidity is subsequently added, line 395 enters the normal branch because both reserves remain positive. For every positive `mintRatio`, line 403 evaluates:

`0 + floor(0 × mintRatio) = 0`

Consequently, `_QUOTE_TARGET_` remains zero regardless of how much liquidity is added. Lines 406 and 407 nevertheless mint LP shares and commit the new token balances as reserves.

This is an absorbing accounting state: further proportional additions cannot restore the target. It is independent of precision loss when calculating a nonzero ratio; even an exact full-precision ratio still leaves a zero target unchanged. The resulting state violates the nonzero-target invariant enforced during initial liquidity creation and can cause downstream PMM pricing arithmetic to revert or produce degenerate results.

### Assumptions and preconditions

- The pool is honestly deployed with valid parameters and tokens having different decimal scales, such as an 18-decimal base and 6-decimal quote.
- An LP owns enough shares to reduce the remaining represented quote target below one raw quote-token unit.
- The withdrawal leaves positive raw balances because of the permanently locked LP supply.
- Liquidity is later added to the same pool instead of deploying a replacement pool.
- PMM pricing relies on the nonzero targets established as an initialization invariant.
- For deliberate griefing, the attacker must acquire nearly all outstanding LP shares; the same state can also arise through an ordinary pool wind-down.

### Proof of concept

1. Create an 18-decimal base/6-decimal quote pool with `_I_ = 1_000_000`.
2. Transfer `1e21` base units and `1e9` quote units into the pool.
3. Call `buyShares` for the initial provider.
4. Verify:
   - `totalSupply() == 1e21`
   - `_BASE_TARGET_ == 1e21`
   - `_QUOTE_TARGET_ == 1e9`
   - the provider owns `1e21 - 1001` shares.
5. Redeem all provider-owned shares through `sellShares`.
6. Verify the residual state:
   - `totalSupply() == 1001`
   - `_BASE_RESERVE_ == 1001`
   - `_QUOTE_RESERVE_ == 1`
   - `_BASE_TARGET_ == 1001`
   - `_QUOTE_TARGET_ == 0`.
7. Transfer `1e21` additional base units and `1e9` additional quote units into the pool.
8. Call `buyShares` again.
9. The reserve checks at line 395 pass, and the computed share amount exceeds the minimum mint threshold.
10. Verify that LP shares are minted and reserves increase, but `_QUOTE_TARGET_` remains zero at line 403.
11. Repeat another liquidity addition and confirm the target remains zero.
12. Inspect or invoke PMM pricing and swap paths to observe failure or degenerate behavior caused by the invalid zero target.

### Impact assessment

**Severity: Medium.**

A non-privileged LP with sufficient share ownership can put a cross-decimal pool into a zero-target dust state while recovering nearly all of their capital. The highlighted liquidity path then accepts meaningful new liquidity without restoring valid PMM accounting.

The affected pool may be unable to resume reliable trading under the same address. Swap routing, price quoting, and oracle integrations can fail, while liquidity providers may need to redeem and migrate to a replacement pool. The worst case is persistent denial of service and operational disruption for an established pool rather than a direct unrestricted asset drain.

### Remediation

The normal-liquidity branch must not be selected solely from positive reserves. Require both PMM targets to be nonzero as well.

For a preexisting dust state with a zero target, route execution to a dedicated rebootstrap path that:

- reconstructs both targets using `_I_`, current balances, and existing LP ownership;
- preserves the economic claims represented by the remaining supply;
- requires both reconstructed targets to be positive;
- performs all validation before minting shares or committing reserves.

If a provably invariant-preserving rebootstrap cannot be implemented, revert before minting and require pool migration. Add a final postcondition before `_mint` ensuring both targets are nonzero. Do not merely continue multiplicatively scaling a zero target.

## Finding 22: Unattributed prefunding in MagicLP.buyShares lets a front-runner steal deposits

### Synopsis

`MagicLP.buyShares` attributes all pending balance growth to the next caller and mints the resulting shares to caller-selected `to`. An attacker can front-run a depositor’s post-transfer call, claim their liquidity, and redeem the shares to steal both deposited assets.

### Technical details

During normal liquidity provision, lines 397–400 calculate shares from the differences between the LP’s current token balances and its recorded reserves. Those differences contain no information about who transferred the tokens.

Line 406 nevertheless mints all calculated shares to the freely specified `to` address. There is no commitment, depositor record, authorization, or atomic token pull binding the balance increases to the share recipient. Consequently, possession of pending deposits is effectively granted to whichever account calls `buyShares` first.

This is exploitable whenever an integration transfers the base and quote tokens before calling `buyShares` in a separately orderable transaction. A front-runner can call `buyShares(attacker)` after the transfers but before the depositor’s call. Line 407 then records the full balances as reserves, so the depositor’s subsequent call sees no remaining input and cannot obtain shares.

The attack is independent of deposit imbalance or precision loss: a perfectly proportional deposit with exact arithmetic is fully claimable by the front-runner. The `nonReentrant` modifier does not prevent competing transactions.

### Assumptions and preconditions

- The pool is initialized with positive base and quote reserves and nonzero total LP supply.
- A user or integration transfers both assets to the LP before invoking `buyShares`.
- The token transfers and `buyShares` are separate transactions or are otherwise exposed to adversarial ordering.
- The pending deposit is large enough to mint more than the contract’s minimum share threshold.
- The attacker can observe the pending balances and execute `buyShares` before the intended depositor.
- The attacker can subsequently redeem the stolen LP shares.

### Proof of concept

1. Start with an initialized pool containing 100,000 base tokens, 100,000 quote tokens, and 100,000 total LP shares.
2. A victim transfers 10,000 base tokens and 10,000 quote tokens to the pool, intending to call `buyShares(victim)` afterward.
3. Before the victim’s `buyShares` transaction executes, an attacker calls `buyShares(attacker)`.
4. Lines 397 and 398 calculate both input ratios as `10,000 / 100,000 = 0.1`.
5. Lines 399 and 400 calculate 10,000 new LP shares.
6. Line 406 mints those 10,000 shares to the attacker, despite the attacker not supplying the pending assets.
7. Line 407 updates both reserves to 110,000.
8. The victim’s later call finds no unaccounted input and reverts or receives no shares.
9. The attacker redeems 10,000 out of 110,000 total shares, receiving approximately 10,000 base and 10,000 quote tokens—the victim’s complete proportional deposit.

### Impact assessment

**Severity: High.**

A permissionless attacker can steal the full proportional value of liquidity supplied through a non-atomic transfer-then-mint flow. The attack requires no privileged role, capital beyond transaction fees, or manipulation of token prices.

Atomic Router operations avoid an inter-transaction race, but the raw public interface remains unsafe for direct users and integrations. Tests and interfaces that demonstrate prefunding followed by `buyShares` may encourage precisely the vulnerable usage pattern. Repeated exploitation could cause direct user losses, integration failures, and loss of confidence in liquidity provisioning.

### Remediation

Replace the push-based, unattributed balance accounting with an atomic pull-based liquidity function:

- Accept explicit base and quote amounts, recipient, minimum shares, and deadline.
- Pull both tokens from `msg.sender` within the same `buyShares` transaction.
- Measure actual received amounts using balance snapshots around those transfers.
- Calculate shares exclusively from amounts received from that caller during the transaction.
- Validate minimum shares before minting.
- Update reserves only after the caller-attributed deposit has been processed.
- Remove or disable the public function that allows arbitrary callers to convert pre-existing balance deltas into shares.

If push-based transfers must remain supported, use a trusted atomic adapter or a deposit commitment that records the depositor, exact amounts, recipient, and nonce before shares can be minted. Merely requiring `to == msg.sender` is insufficient because an attacker can still call first and mint the victim-funded shares to itself.

## Finding 23: Historical TWAP backfill in MagicLP.buyShares fabricates pre-liquidity price history

### Synopsis

`MagicLP.buyShares` updates stored reserves through `_setReserve` before advancing the price accumulator. On first funding, the accumulator therefore prices the entire interval since initialization at the newly established price, allowing delayed initial liquidity to fabricate TWAP history and bypass oracle warm-up periods.

### Technical details

An initialized pool starts with zero `_BASE_RESERVE_` and `_QUOTE_RESERVE_`, while `_BLOCK_TIMESTAMP_LAST_` is set during `init`. If the pool remains unfunded, no valid market price exists during that interval.

- `timeElapsed = fundingTimestamp - initializationTimestamp`;
- both current reserves as nonzero; and
- `getMidPrice()` calculated from the newly funded PMM state.

It therefore performs the equivalent of:

`cumulative += newlyEstablishedPrice * entirePreFundingInterval`

The accumulator falsely represents the pool as having traded at that price since initialization. The correct behavior is to advance the timestamp while the old reserves are still zero, without accumulating a price for the no-liquidity interval.

This breaks the accumulator’s temporal invariant: elapsed time must be attributed to the state that existed during that time, not to the state installed at the end of the interval. An observation taken when the clone was initialized and another immediately after first funding can consequently appear to provide a mature TWAP window even though the pool has only just acquired liquidity.

### Assumptions and preconditions

- An honestly created and initialized clone remains unfunded for a nontrivial period.
- A non-privileged user can provide the first valid liquidity and call `buyShares`.
- The supplied amounts satisfy the initial mint requirements.
- A downstream component treats the pool’s cumulative price history as valid without independently tracking the first-liquidity timestamp.
- The attacker benefits from bypassing the downstream component’s intended TWAP observation or warm-up period.

### Proof of concept

1. Create or identify a correctly initialized pool at timestamp `t0` with:
   - both stored reserves equal to zero;
   - `_BASE_PRICE_CUMULATIVE_LAST_ = 0`; and
   - `_BLOCK_TIMESTAMP_LAST_ = t0`.
2. Record the cumulative price as an initial oracle observation.
3. Leave the pool unfunded for `Δt`, such as one day.
4. At `t1 = t0 + Δt`, transfer sufficient base and quote tokens to the pool and call `buyShares`.
6. `_setReserve` installs the positive balances before `_twapUpdate` runs.
7. `_twapUpdate` observes positive reserves and adds `getMidPrice() * Δt` to the cumulative price.
8. Read the cumulative price immediately after funding. It now represents `Δt` of price history, despite the pool having had no liquidity during that period.
9. A consumer comparing the observations from steps 2 and 8 derives an apparently mature TWAP and may accept the pool immediately, bypassing the intended post-liquidity observation period.

For example, if the newly established mid-price is `1e18` and the pool remained empty for `86,400` seconds, the accumulator incorrectly increases by `86,400e18` during the first funding transaction.

### Impact assessment

**Severity: Medium.**

The issue does not directly transfer pool assets, but it corrupts the pool’s price history and defeats the security assumptions of TWAP consumers. An attacker can make a newly funded pool appear to have a long observation history immediately after liquidity activation.

Potential consequences include premature oracle acceptance, incorrect collateral valuation, unsafe swap routing, or manipulation of downstream systems that rely on observation age to resist spot-price manipulation. The worst case is financial loss in an integrated lending or valuation protocol that accepts the fabricated TWAP as established market history.

### Remediation

Checkpoint the accumulator before activating the new reserves. With the old reserves still zero, `_twapUpdate` advances `_BLOCK_TIMESTAMP_LAST_` without recording a price. The subsequent `_setReserve` call sees zero elapsed time and cannot backfill the pre-liquidity interval.

```solidity
        _twapUpdate();
        _setReserve(baseBalance, quoteBalance);
```

As defense in depth, downstream consumers should also track the first timestamp at which both reserves became nonzero and require a complete observation window after that timestamp.

## Finding 24: Precision Loss and Conversion Asymmetry in `DecimalMath.reciprocalFloor` Distorts PMM Quote-Side Pricing and Target Calculations

### Synopsis

`DecimalMath.reciprocalFloor` computes `10**36 / target` to invert the pool price `i`, truncating the reciprocal to an `1e18` fixed-point integer before it is used in `DecimalMath.mulFloor` and `Math` pricing functions. For large or non-divisible `i` values (up to `MAX_I = 10**36`), this quantization causes severe conversion asymmetry and miscalculates reserve targets in `PMMPricing`.

### Technical details

In `src/mimswap/libraries/DecimalMath.sol`, fixed-point arithmetic is implemented with a scaling factor of `ONE = 10 ** 18` and `ONE2 = 10 ** 36`:

```solidity
    function mulFloor(uint256 target, uint256 d) internal pure returns (uint256) {
        return (target * d) / ONE;
    }
...
    function reciprocalFloor(uint256 target) internal pure returns (uint256) {
        return ONE2 / target;
    }
```

In `MagicLP.init`, the oracle price `_I_` (`state.i`, representing the price of 1 Base token in Quote token units scaled by `10 ** 18`) is validated against `MAX_I = 10 ** 36`:

```solidity
        if (i == 0 || i > MAX_I) {
            revert ErrInvalidI();
        }
```

When pricing base-side operations (`sellBaseToken`, `_GeneralIntegrate`, `_SolveQuadraticFunctionForTarget` in `BELOW_ONE`), `PMMPricing` passes `state.i` directly to `Math` functions, which multiply by `i` and divide by `ONE` (or `ONE2`). However, for quote-side operations and `ABOVE_ONE` target adjustments, `PMMPricing` first inverts `state.i` via `DecimalMath.reciprocalFloor(state.i)` and then passes that quantized reciprocal as the price parameter `i` into `Math` functions (which subsequently call `DecimalMath.mulFloor`):

1. `PMMPricing._ROneSellQuoteToken` (line 129)
2. `PMMPricing._RBelowSellQuoteToken` (line 144)
3. `PMMPricing._RAboveSellQuoteToken` (line 185)
4. `PMMPricing.adjustedTarget` when `state.R == RState.ABOVE_ONE` (lines 194-199)

This creates two compounded fixed-point precision and rounding defects:

1. **Severe Reciprocal Quantization for High `i`:** Because `state.i` is allowed up to `MAX_I = 10 ** 36` (e.g., when the Quote token has significantly more decimals than the Base token or a much lower unit price), `ONE2 / state.i` loses precision as `state.i` grows above `10 ** 18`. For `state.i` in `(5 * 10 ** 35, 10 ** 36]`, `DecimalMath.reciprocalFloor(state.i)` evaluates to `1`, losing up to 50% of the actual reciprocal value (`ONE2 / (5 * 10 ** 35 + 1) == 1` instead of `~2`).
2. **Conversion Asymmetry and Underestimated `B0` Target in `ABOVE_ONE`:** When `state.R == RState.ABOVE_ONE`, `PMMPricing.adjustedTarget` recomputes the base target `state.B0` in `Math._SolveQuadraticFunctionForTarget` by composing `DecimalMath.reciprocalFloor(state.i)` with `DecimalMath.mulFloor`. Because `reciprocalFloor` rounds down `1 / i` rather than rounding up (via `reciprocalCeil`) or dividing by `state.i` directly after multiplying by `delta`, `state.B0` is systematically underestimated relative to the true curve invariant. When a subsequent `sellBaseToken` call occurs while `state.R == RState.ABOVE_ONE`:
   - If `payBaseAmount < backToOnePayBase`, `_RAboveSellBaseToken` integrates using the unquantized `state.i` and the underestimated `state.B0`.
   - If `payBaseAmount >= backToOnePayBase` (`state.B0 - state.B`), the trader only needs to supply the underestimated `state.B0 - state.B` base tokens to extract the full `backToOneReceiveQuote = state.Q - state.Q0` quote tokens.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with a valid price `i` (`1 <= i <= 10 ** 36`) that does not evenly divide `10 ** 36`, with the distortion becoming increasingly severe as `i` approaches `MAX_I` (e.g., when Base has 6 decimals and Quote has 18 decimals, where `i` is scaled by `10 ** 30`).
2. Traders interact with the pool via `sellQuote` and `sellBase`.

### Proof of concept

Consider a `MagicLP` pool initialized with `K = 0` (or small `K`) and `i = 6 * 10 ** 35` (well within `MAX_I = 10 ** 36`):

1. **Reciprocal Quantization:**
   - The true reciprocal price of Quote in terms of Base scaled by `10 ** 18` is `10 ** 36 / (6 * 10 ** 35) = 1.666...`.
   - `DecimalMath.reciprocalFloor(6 * 10 ** 35)` returns `10 ** 36 / (6 * 10 ** 35) = 1`.
2. **Asymmetric Quote-to-Base Swap (`sellQuote`):**
   - Suppose the pool starts at `R = RState.ONE` with `B = B0 = 1000` and `Q = Q0 = 600 * 10 ** 36`.
   - A user sells `payQuoteAmount = 1.9 * 10 ** 18` Quote tokens via `sellQuote`.
   - `_ROneSellQuoteToken` computes `receiveBaseAmount = DecimalMath.mulFloor(1, 1.9 * 10 ** 18) = 1` Base unit (instead of `floor(1.9 * 10 ** 18 / 0.6 * 10 ** 18) = 3` Base units, a 66% loss to the seller due to composed truncation in `reciprocalFloor` and `mulFloor`).
   - Pool state updates to `_RState_ = ABOVE_ONE`, `B = B0 - 1`, `Q = Q0 + 1.9 * 10 ** 18`, `_QUOTE_TARGET_ = Q0`.
3. **Target Underestimation and Arbitrage on `sellBase`:**
   - If a user instead sells `payQuoteAmount = 0.9 * 10 ** 18` Quote tokens when `R = ONE`, `receiveBaseAmount = DecimalMath.mulFloor(1, 0.9 * 10 ** 18) = 0`, while `_RState_` becomes `ABOVE_ONE` and `Q` increases by `0.9 * 10 ** 18`.
   - On the next `getPMMState()` call, `adjustedTarget` computes `state.B0 = state.B + DecimalMath.mulFloor(1, 0.9 * 10 ** 18) = state.B`.
   - Thus `backToOnePayBase = state.B0 - state.B = 0`, while `backToOneReceiveQuote = state.Q - state.Q0 = 0.9 * 10 ** 18`. Any caller executing `sellBase` with `1` wei of Base satisfies `payBaseAmount > backToOnePayBase` and extracts the entire `0.9 * 10 ** 18` Quote surplus.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Composing `DecimalMath.reciprocalFloor(state.i)` with `DecimalMath.mulFloor` introduces significant quantization error for quote-side swaps and underestimates `state.B0` when `R == RState.ABOVE_ONE`.
- **Business Impact:** Traders selling Quote tokens into pools with large or non-divisible `i` receive less Base token than dictated by the PMM formula, and arbitrageurs can extract excess Quote tokens when returning the pool from `ABOVE_ONE` to `ONE` or `BELOW_ONE`.

### Remediation

Avoid pre-truncating `1 / i` via `DecimalMath.reciprocalFloor` before multiplying by token amounts. Where a reciprocal must be used to compute an upper-bound target (`state.B0` in `adjustedTarget`), round the reciprocal up using `reciprocalCeil` or perform full-precision multiplication before dividing by `i` using OpenZeppelin's `Math.mulDiv`:

```solidity
    function mulFloor(uint256 target, uint256 d) internal pure returns (uint256) {
        return OpenZeppelinMath.mulDiv(target, d, ONE);
    }

    function mulCeil(uint256 target, uint256 d) internal pure returns (uint256) {
        return OpenZeppelinMath.mulDiv(target, d, ONE, OpenZeppelinMath.Rounding.Up);
    }

    function divFloor(uint256 target, uint256 d) internal pure returns (uint256) {
        return OpenZeppelinMath.mulDiv(target, ONE, d);
    }

    function divCeil(uint256 target, uint256 d) internal pure returns (uint256) {
        return OpenZeppelinMath.mulDiv(target, ONE, d, OpenZeppelinMath.Rounding.Up);
    }
```

Additionally, in `PMMPricing`, replace `DecimalMath.reciprocalFloor(state.i)` in `adjustedTarget` with `DecimalMath.reciprocalCeil(state.i)` (or pass `state.i` directly and divide by `state.i` inside the `Math` helpers after multiplying by `delta`) so that `state.B0` is never underestimated.

## Finding 25: Intermediate Multiplication Overflow Before Fixed-Point Rescaling in `DecimalMath` Causes Swap DoS in `MagicLP`

### Synopsis

`DecimalMath.mulFloor`, `DecimalMath.mulCeil`, `DecimalMath.divFloor`, and `DecimalMath.divCeil` perform 256-bit multiplication (`target * d` and `target * ONE`) before dividing by the scaling factor (`ONE` or `d`). For valid `uint112` pool reserves and targets in `MagicLP`, this unscaled intermediate product overflows `uint256`, causing swaps to revert and locking pool trading.

### Technical details

In `src/mimswap/libraries/DecimalMath.sol`, the fixed-point multiplication and division helpers compute intermediate products directly in 256-bit arithmetic prior to rescaling:

```solidity
function mulFloor(uint256 target, uint256 d) internal pure returns (uint256) {
    return (target * d) / ONE;
}

function mulCeil(uint256 target, uint256 d) internal pure returns (uint256) {
    return (target * d).divCeil(ONE);
}

function divFloor(uint256 target, uint256 d) internal pure returns (uint256) {
    return (target * ONE) / d;
}

function divCeil(uint256 target, uint256 d) internal pure returns (uint256) {
    return (target * ONE).divCeil(d);
}
```

In `MagicLP.sol`, pool reserves (`_BASE_RESERVE_`, `_QUOTE_RESERVE_`) and target reserves (`_BASE_TARGET_`, `_QUOTE_TARGET_`) are stored as `uint112` (maximum value `2^112 - 1 ≈ 5.19 × 10^33`), while the oracle price `_I_` can be up to `MAX_I = 10^36` and `_K_` can be up to `MAX_K = 10^18`.

When executing swaps (`sellBase` / `sellQuote`) or querying swap outputs (`querySellBase` / `querySellQuote`), `PMMPricing` invokes `Math._SolveQuadraticFunctionForTrade` and `Math._GeneralIntegrate`:

1. **`Math._SolveQuadraticFunctionForTrade` (line 184):**
   ```solidity
   uint256 squareRoot = DecimalMath.mulFloor((DecimalMath.ONE - k) * 4, DecimalMath.mulFloor(k, V0) * V0); // 4(1-k)kQ0^2
   ```
   Here, `target = (DecimalMath.ONE - k) * 4` (up to `4 × 10^18`) and `d = DecimalMath.mulFloor(k, V0) * V0 ≈ (k * V0^2) / 10^18`. Inside `DecimalMath.mulFloor(target, d)`, the intermediate product `target * d` is:
   $$\text{target} \times d \approx 4(10^{18} - k) \cdot \frac{k \cdot V_0^2}{10^{18}}$$
   For $k = 0.5 \times 10^{18}$, $\text{target} = 2 \times 10^{18}$ and $d = 0.5 \times V_0^2$, so $\text{target} \times d = 10^{18} \times V_0^2$. Because `type(uint256).max` is $\approx 1.1579 \times 10^{77}$, `target * d` overflows `uint256` whenever $V_0 > 3.40 \times 10^{29}$—nearly four orders of magnitude below the `uint112` bound ($5.19 \times 10^{33}$) permitted during liquidity provision in `MagicLP.buyShares`. Even though the rescaled result $(\text{target} \times d) / 10^{18} = V_0^2 \le 2.70 \times 10^{67}$ fits easily in `uint256`, the intermediate multiplication reverts with Panic `0x11`.

2. **`Math._GeneralIntegrate` (lines 62–63) & `Math._SolveQuadraticFunctionForTrade` (line 199):**
   When `state.R != RState.ONE`, `PMMPricing.adjustedTarget` computes `state.Q0` or `state.B0` via `Math._SolveQuadraticFunctionForTarget`, which can yield a target $V_0 \approx V_1 + i \cdot \Delta$ up to $\approx 5.19 \times 10^{51}$ when $i \le 10^{36}$. Passing $(V_0 \times V_0) / V_1$ into `DecimalMath.divFloor(..., V2)` (which multiplies by `ONE = 10^18` before dividing by `V2`) or subsequently into `DecimalMath.mulFloor(k, V0V0V1V2)` overflows `uint256` prior to rescaling.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with valid parameters (e.g., `_I_ = 10^18`, `0 < _K_ < 10^18` such as `_K_ = 0.5 * 10^18`).
2. The pool accumulates or is initialized with target reserves `V0` (`_BASE_TARGET_` or `_QUOTE_TARGET_`) exceeding $\sqrt{\text{type(uint256).max} / (4 \cdot (10^{18} - k) \cdot k / 10^{18})}$ (e.g., $V_0 \ge 3.41 \times 10^{29}$ for $k = 0.5 \times 10^{18}$), or an adjusted target $V_0$ reaching that threshold when `i` is large.

### Proof of concept

1. Initialize a `MagicLP` pool with `i = 1e18` (`1.0`), `k = 5e17` (`0.5`), and `lpFeeRate = 1e14`.
2. Deposit `baseInput = 1e30` and `quoteInput = 1e30` (both well below `type(uint112).max ≈ 5.19e33`) and call `MagicLP.buyShares(alice)`.
   - `buyShares` succeeds: `shares = 1e30 - 1001`, `_BASE_TARGET_ = 1e30`, `_QUOTE_TARGET_ = 1e30`, `_BASE_RESERVE_ = 1e30`, `_QUOTE_RESERVE_ = 1e30`.
3. Transfer any non-zero amount of quote token (e.g., `1e18`) to the pool and call `MagicLP.sellQuote(bob)` (or call `querySellQuote(bob, 1e18)`).
4. `querySellQuote` calls `PMMPricing.sellQuoteToken` -> `PMMPricing._ROneSellQuoteToken` -> `Math._SolveQuadraticFunctionForTrade(1e30, 1e30, 1e18, 1e18, 5e17)`.
5. At `Math.sol:184`:
   - `DecimalMath.mulFloor(5e17, 1e30) * 1e30` evaluates to `5e29 * 1e30 = 5e59`.
   - `(DecimalMath.ONE - k) * 4` evaluates to `5e17 * 4 = 2e18`.
   - `DecimalMath.mulFloor(2e18, 5e59)` attempts to compute `2e18 * 5e59 = 1e78`, which exceeds `type(uint256).max` (`1.1579e77`) and reverts with Panic `0x11`, permanently blocking all swaps on the pool until liquidity is withdrawn.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Intermediate 256-bit multiplication overflow in `DecimalMath` causes `Math._SolveQuadraticFunctionForTrade` and `Math._GeneralIntegrate` to revert for valid `uint112` reserve and target values that should be supported by the pool.
- **Business Impact:** Any `MagicLP` pool whose target reserves exceed $\approx 3.4 \times 10^{29}$ (for $k = 0.5$)—or whose price parameter `i` produces an adjusted target above this bound—suffers a complete denial of service on `sellBase`, `sellQuote`, `querySellBase`, and `querySellQuote`. Liquidity providers can deposit into the pool via `buyShares`, but traders cannot swap against the pool.

### Remediation

Use 512-bit intermediate multiplication and division (such as OpenZeppelin's `Math.mulDiv`) in `DecimalMath.mulFloor`, `DecimalMath.mulCeil`, `DecimalMath.divFloor`, and `DecimalMath.divCeil` to avoid intermediate overflow before rescaling:

```solidity
import {Math as OZMath} from "openzeppelin-contracts/contracts/utils/math/Math.sol";

library DecimalMath {
    using Math for uint256;

    uint256 internal constant ONE = 10 ** 18;
    uint256 internal constant ONE2 = 10 ** 36;

    function mulFloor(uint256 target, uint256 d) internal pure returns (uint256) {
        return OZMath.mulDiv(target, d, ONE);
    }

    function mulCeil(uint256 target, uint256 d) internal pure returns (uint256) {
        return OZMath.mulDiv(target, d, ONE, OZMath.Rounding.Up);
    }

    function divFloor(uint256 target, uint256 d) internal pure returns (uint256) {
        return OZMath.mulDiv(target, ONE, d);
    }

    function divCeil(uint256 target, uint256 d) internal pure returns (uint256) {
        return OZMath.mulDiv(target, ONE, d, OZMath.Rounding.Up);
    }

    function reciprocalFloor(uint256 target) internal pure returns (uint256) {
        return ONE2 / target;
    }

    function reciprocalCeil(uint256 target) internal pure returns (uint256) {
        return ONE2.divCeil(target);
    }
```

## Finding 26: Fee rounding in DecimalMath.mulFloor lets MagicLP.querySellBase and querySellQuote undercharge swaps

### Synopsis

`DecimalMath.mulFloor` truncates fractional fees to zero, and MagicLP’s `querySellBase` and `querySellQuote` apply it separately to LP and maintainer fees. A trader can split swaps into sub-threshold outputs, avoid both charges, and systematically deprive liquidity providers and the maintainer of revenue.

### Technical details

`DecimalMath.mulFloor` computes:

`floor(target × d / 1e18)`

Consequently, it returns zero whenever both operands are nonzero but `target × d < 1e18`. MagicLP uses this result to calculate the LP and maintainer fees independently. Neither fractional remainder is retained between swaps.

For a gross swap output `G`, adjusted LP fee rate `rLP`, and maintainer fee rate `rMT`, both fees are zero when:

- `G < 1e18 / rLP`
- `G < 1e18 / rMT`

With the minimum configured total fee rate of `1e14` split equally by `FeeRateModelImpl`, each component rate is `5e13`. Therefore, any gross output below `20,000` raw token units incurs neither fee.

Because swaps have no protocol-level minimum amount and fee remainders are not accumulated, a trader can divide a larger swap into outputs below this threshold. The PMM state changes after each swap, but the attacker can adapt each subsequent input to keep its gross output below the threshold. This is particularly material for tokens with few decimals, where thousands of raw units can represent substantial economic value.

For example, two swaps producing `19,999` output units each charge zero fees. Applying the same two component fee calculations to a single `39,998`-unit output charges one unit to the LP and one unit to the maintainer. The split execution therefore gives the trader two additional output units and eliminates all fee revenue.

### Assumptions and preconditions

- The pool’s output token has sufficiently coarse units for fee-rounding savings to exceed transaction costs.
- The factory or another supported creation path permits such a token pair.
- The pool has sufficient liquidity for repeated sub-threshold swaps.
- A nonzero LP fee is configured and divided between the LP and maintainer components.
- The attacker can call the public swap functions repeatedly and adjust inputs according to current pool state.
- The tokens follow standard transfer and balance semantics.
- No off-chain mechanism rejects or economically prevents small swaps.

### Proof of concept

1. Create or identify a pool containing two legitimate zero-decimal tokens with equal raw-unit value.
2. Initialize it with:
   - `i = 1e18`
   - `k = 0`
   - LP fee rate `1e14`
   - At least `1,000,000` units of each token.
3. Use the fee model that divides the total rate into `5e13` for the LP and `5e13` for the maintainer.
4. Transfer `19,999` base-token units into the pool and invoke the base-selling swap.
5. With `k = 0` and `i = 1e18`, the gross quote output is `19,999` units.
6. Observe that both fee calculations return zero because:

   `floor(19,999 × 5e13 / 1e18) = 0`

7. Repeat the transfer and swap with another `19,999` base-token units. The second swap also pays no fees.
8. The attacker receives `39,998` quote-token units in total, while both the maintainer and LP receive no fee.
9. Compare this with one `39,998`-unit gross swap:

   `floor(39,998 × 5e13 / 1e18) = 1`

   Each fee component charges one unit, so the trader receives only `39,996` units, the maintainer receives one unit, and one unit remains for LPs.
10. Repeat the split execution to bypass the configured fee over larger cumulative volume.

### Impact assessment

**Severity: Medium.**

A non-privileged trader can systematically bypass both fee components by splitting trades. The direct loss equals the fees that should have accrued to liquidity providers and the maintainer. This also weakens the pool’s economic protection against arbitrage because trades can consume liquidity without paying the configured compensation.

The attack’s profitability depends on output-token granularity and transaction costs. It is most significant for supported low-decimal tokens, where one raw unit has meaningful value. In the worst case, repeated fee-free trades eliminate essentially all swap-fee revenue for affected pools and cause sustained LP and protocol revenue loss.

### Remediation

Do not use downward rounding for swap fees without preserving fractional remainders.

The preferred solution is to calculate each fee component with upward rounding and reject trades whose gross output is insufficient to cover the resulting fees. Alternatively, calculate one aggregate fee with upward rounding, allocate it between the LP and maintainer according to a deterministic policy, and carry allocation remainders forward if exact long-term distribution is required.

A remainder-accumulation mechanism is another valid solution: store discarded fixed-point fee fractions and convert them into whole token units on later swaps. This preserves long-term fee accuracy without overcharging individual transactions.

Do not globally change `mulFloor`, because its downward-rounding semantics may be required by non-fee pricing calculations. Instead, change the fee-calculation call sites to use explicit fee-safe rounding and impose a minimum economically meaningful swap output.

## Finding 27: Missing Zero-Reserve (`V1 == 0`) Check in `Math._SolveQuadraticFunctionForTrade` Causes Division-by-Zero Panic

### Synopsis

`Math._SolveQuadraticFunctionForTrade` checks for `V0 == 0` and `delta == 0`, but fails to check whether the current reserve `V1` is zero. While the `k == 0` and `k == 1e18` branches return `0` when `V1 == 0`, the general `0 < k < 1e18` branch divides by `V1` at line 170, causing an unhandled division-by-zero panic.

### Technical details

In `src/mimswap/libraries/Math.sol`, `_SolveQuadraticFunctionForTrade(uint256 V0, uint256 V1, uint256 delta, uint256 i, uint256 k)` solves the PMM quadratic pricing equation for the post-trade reserve `V2` given target reserve `V0`, current reserve `V1`, input trade amount `delta`, price `i`, and slippage coefficient `k`, returning the trade output `V1 - V2`.

At the entry of `_SolveQuadraticFunctionForTrade` (lines 131–138), the function validates:
```solidity
if (V0 == 0) {
    revert ErrIsZero();
}

if (delta == 0) {
    return 0;
}
```
However, unlike `_SolveQuadraticFunctionForTarget` (which explicitly checks `if (V1 == 0) { return 0; }` at lines 89–91), `_SolveQuadraticFunctionForTrade` does not guard against a degenerate current reserve `V1 == 0`:

1. When `k == 0` (line 141), `DecimalMath.mulFloor(i, delta) > V1 ? V1 : DecimalMath.mulFloor(i, delta)` evaluates to `V1 = 0`.
2. When `k == DecimalMath.ONE` (lines 144–160), `(V1 * temp) / (temp + DecimalMath.ONE)` evaluates to `0`.
3. When `0 < k < DecimalMath.ONE`, execution proceeds to line 170:
   ```solidity
   uint256 part2 = (((k * V0) / V1) * V0) + (i * delta); // kQ0^2/Q1-i*deltaB
   ```
   Because `V1 == 0`, `((k * V0) / V1)` reverts with a Solidity division-by-zero panic (`Panic(0x12)`).

Furthermore, if `V0 == 0` because `_SolveQuadraticFunctionForTarget` returned `0` for `V1 == 0` in `PMMPricing.adjustedTarget` (line 192 or 194), even a zero-input query (`delta == 0`) reverts with `ErrIsZero()` at line 132 because `if (V0 == 0)` is evaluated before `if (delta == 0)`.

### Assumptions and preconditions

1. `_SolveQuadraticFunctionForTrade` is invoked with `V0 > 0`, `V1 == 0`, `delta > 0`, and `0 < k < 10**18` (or via `PMMPricing` when one side's reserve is depleted while the target reserve `V0` remains non-zero).
2. Alternatively, a zero-amount trade query (`delta == 0`) is invoked when `V1 == 0` and `V0 == 0` (after `PMMPricing.adjustedTarget` sets the target to `0` via `_SolveQuadraticFunctionForTarget`).

### Proof of concept

1. Consider a pool state where target reserve `V0 = 1000e18`, current reserve `V1 = 0`, price `i = 1e18`, slippage parameter `k = 5e17` (`0.5e18`, within `(0, 1e18)`), and trade input `delta = 10e18`.
2. When `k == 0` or `k == 1e18`, calling `Math._SolveQuadraticFunctionForTrade(V0, 0, delta, i, k)` returns `0` because no reserve `V1` is available to withdraw.
3. With `k = 5e17`, `Math._SolveQuadraticFunctionForTrade(1000e18, 0, 10e18, 1e18, 5e17)` bypasses the `V0 == 0`, `delta == 0`, `k == 0`, and `k == DecimalMath.ONE` checks.
4. Execution reaches line 170:
   ```solidity
   uint256 part2 = (((k * V0) / V1) * V0) + (i * delta);
   ```
5. The division `(5e17 * 1000e18) / 0` triggers an unhandled arithmetic panic (`Panic(0x12)`).

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Unhandled division-by-zero panic (`Panic(0x12)`) in `Math._SolveQuadraticFunctionForTrade` when `V1 == 0` and `0 < k < 1e18`, and inconsistent boundary behavior across `k` parameter regimes (`k == 0` and `k == 1e18` return `0` while `0 < k < 1e18` reverts) as well as reverting on `delta == 0` queries when `V0 == 0`.
- **Business Impact:** View queries (`querySellBase`, `querySellQuote`) and router price quotes can unexpectedly revert with a division-by-zero panic instead of returning `0` output when a pool reserve is exhausted.

### Remediation

Check `delta == 0` and `V1 == 0` at the beginning of `_SolveQuadraticFunctionForTrade` so that zero-input trades and depleted-reserve states consistently return `0` without dividing by zero:

```solidity
    function _SolveQuadraticFunctionForTrade(uint256 V0, uint256 V1, uint256 delta, uint256 i, uint256 k) internal pure returns (uint256) {
        if (delta == 0 || V1 == 0) {
            return 0;
        }

        if (V0 == 0) {
            revert ErrIsZero();
        }
```

## Finding 28: Zero-target validation in Math._SolveQuadraticFunctionForTrade freezes valid zero-curvature MagicLP trades

### Synopsis

`Math._SolveQuadraticFunctionForTrade` unconditionally rejects a zero target before entering its documented `k == 0` constant-price branch. Because that branch does not use the target, a zero-target `MagicLP` with positive reserves becomes unswappable, enabling persistent denial of service and potentially stranding pool assets.

### Technical details

The function reverts whenever `V0 == 0` before considering the curve parameter `k`. This validation is necessary for the quadratic branches, which use `V0` in their pricing calculations, but it is not necessary when `k == 0`.

For `k == 0`, the intended result is simply:

`min(floor(i * delta / 1e18), V1)`

That calculation only depends on the price, input amount, and current reserve. It neither divides by nor otherwise uses `V0`. The comments also explicitly state that the function supports the `k == 0` case.

Consequently, a valid zero-curvature pool whose target has become zero cannot execute an otherwise well-defined constant-price trade. Such a state may arise through reserve exhaustion, target rounding, liquidity removal, or reserve synchronization where current reserves and stored targets do not change together. Calls through `PMMPricing` propagate the revert into `querySellBase`, `querySellQuote`, `sellBase`, or `sellQuote`.

The behavior is discontinuous: with identical `V1`, `delta`, `i`, and `k == 0`, a positive `V0` returns the correct bounded output, while `V0 == 0` reverts even though `V0` has no mathematical effect on the applicable formula.

### Assumptions and preconditions

- The pool uses the documented and supported `k == 0` configuration.
- The target passed as `V0` has become zero while `V1` contains a positive reserve or the trade otherwise has a nonzero bounded output.
- The degenerate target state is reachable through permissionless trading, liquidity, rounding, or reserve-synchronization behavior.
- The affected swap direction invokes `_SolveQuadraticFunctionForTrade`.
- No privileged intervention restores the target before exploitation; owner and administrator accounts remain honest.

### Proof of concept

1. Consider a valid zero-curvature pool state with:
   - `V0 = 0`
   - `V1 = 100`
   - `delta = 10`
   - `i = 1e18`
   - `k = 0`
2. Under the constant-price formula, the correct result is `min(10, 100) = 10`.
3. Invoke `_SolveQuadraticFunctionForTrade` with these parameters.
4. The function reaches the unconditional `V0 == 0` validation and reverts with `ErrIsZero`.
5. The documented `k == 0` branch is never reached.
6. Invoke the corresponding public quote or swap path on a pool in this state. The revert propagates and prevents the trade and reserve update.
7. Repeating the operation continues to fail until another mechanism restores a nonzero target.

### Impact assessment

**Severity: Medium.**

A non-privileged actor able to induce the degenerate target state can disable the affected swap direction in a valid zero-curvature pool. Positive reserves may remain inaccessible through normal trading, and routers, aggregators, or rebalancing operations relying on the pool will revert.

The issue does not directly transfer assets to the attacker, but its worst-case effect is persistent loss of pool availability and stranded liquidity requiring an exceptional recovery operation. The impact depends on whether supported deployment and lifecycle paths permit `k == 0` pools and allow targets to reach zero independently of reserves.

### Remediation

Scope the zero-target validation to branches that actually require a nonzero target. The constant-price branch should remain available when `V0 == 0`.

```solidity
if (V0 == 0 && k != 0) {
    revert ErrIsZero();
}
```

Additionally, add tests covering `k == 0` with zero and nonzero targets, positive reserves, exhausted reserves, and state transitions that round a target down to zero.

## Finding 29: Silent Output Capping in `Math._SolveQuadraticFunctionForTrade` When `k == 0` Causes Uncompensated Input Consumption and Target Inflation

### Synopsis

In `Math._SolveQuadraticFunctionForTrade`, when `k == 0` (constant-price curve), trade output is silently clamped to `V1` whenever `DecimalMath.mulFloor(i, delta) > V1` instead of reverting. In `MagicLP.sellBase` and `MagicLP.sellQuote`, this consumes the trader's entire input while paying out at most `V1`, causing direct loss of excess input tokens and inflating the pool's virtual target.

### Technical details

`Math._SolveQuadraticFunctionForTrade` calculates the output token amount `|V1 - V2|` when selling `delta` tokens into the PMM curve. For `k > 0`, the bonding curve is asymptotic to `V2 = 0`, meaning `V1 - V2 < V1` for any finite input `delta`. However, at the parameter extreme `k == 0`, the pool operates as a constant-price curve with fixed price `i`.

At lines 140–142 of `src/mimswap/libraries/Math.sol`:

```solidity
if (k == 0) {
    return DecimalMath.mulFloor(i, delta) > V1 ? V1 : DecimalMath.mulFloor(i, delta);
}
```

When a trade input `delta` produces a linear output `DecimalMath.mulFloor(i, delta)` that exceeds the available reserve (or target segment) `V1`, the function does not revert. Instead, it silently caps the returned output to `V1`.

This creates two severe accounting and pricing violations in `MagicLP`:

1. **Uncompensated Input Consumption (`sellBase` / `sellQuote`):**
   In `MagicLP.sellBase` (lines 245–262) and `MagicLP.sellQuote` (lines 268–285), the input amount is determined as the difference between the current token balance of the contract and the stored reserve (`baseInput = baseBalance - _BASE_RESERVE_`). After calling `querySellBase` / `querySellQuote` (which delegates to `PMMPricing` and `Math._SolveQuadraticFunctionForTrade`), the pool transfers out the capped `receiveQuoteAmount` (or `receiveBaseAmount`) and then sets the new reserve to the full token balance via `_setReserve(baseBalance, ...)`. Consequently, all input tokens exceeding the amount needed to purchase `V1` are absorbed into the pool's reserves while the trader receives zero additional output for the excess input.

2. **Target Reserve Inflation (`adjustedTarget`) and Reserve Depletion:**
   When `R` transitions to or remains in `RState.BELOW_ONE` (after `sellBase`) or `RState.ABOVE_ONE` (after `sellQuote`), subsequent calls to `MagicLP.getPMMState()` invoke `PMMPricing.adjustedTarget(state)` (lines 190–201 of `PMMPricing.sol`), which calls `Math._SolveQuadraticFunctionForTarget`:
   ```solidity
   if (k == 0) {
       return V1 + DecimalMath.mulFloor(i, delta);
   }
   ```
   Because `state.B` (or `state.Q`) increased by the full un-capped input `delta` while `state.Q` (or `state.B`) only decreased by the capped `V1` (less fees), the recalculated target `state.Q0` (or `state.B0`) is inflated by `DecimalMath.mulFloor(i, delta) - V1`, permanently distorting the pool's `R` state transitions (`backToOnePayQuote = state.Q0 - state.Q`). Additionally, if `V1` is small enough that `DecimalMath.mulFloor(V1, lpFeeRate) == 0` and `mtFee == 0`, transferring out `V1` reduces the output reserve to `0`, causing subsequent calls to `PMMPricing.getMidPrice` to revert with a division-by-zero panic.

### Assumptions and preconditions

- A `MagicLP` pool is configured with `K == 0` (constant-price mode, which is explicitly supported as indicated by the `support k=1 & k=0 case` comments in `Math.sol`).
- A trader (or router) executes `sellBase` or `sellQuote` with an input `delta` such that `DecimalMath.mulFloor(i, delta) > V1` (for `sellBase`) or `DecimalMath.mulFloor(DecimalMath.reciprocalFloor(i), delta) > V1` (for `sellQuote`).

### Proof of concept

1. Consider a `MagicLP` pool initialized with `_K_ = 0`, `_I_ = 1e18` (1:1 price), `_LP_FEE_RATE_ = 1e14` (0.01%), `mtFeeRate = 0`, and initial reserves `_BASE_RESERVE_ = _BASE_TARGET_ = 1000e18` and `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1000e18` (`_RState_ = RState.ONE`).
2. A trader transfers `baseInput = 1500e18` Base tokens to the pool and calls `sellBase(trader)` without an external router check (or with a low `minimumOut` parameter).
3. `MagicLP.sellBase` calls `querySellBase`, which calls `PMMPricing.sellBaseToken(state, 1500e18)` -> `_ROneSellBaseToken(state, 1500e18)` -> `Math._SolveQuadraticFunctionForTrade(1000e18, 1000e18, 1500e18, 1e18, 0)`.
4. In `Math._SolveQuadraticFunctionForTrade`, `k == 0` is true. `DecimalMath.mulFloor(1e18, 1500e18) = 1500e18 > V1 (1000e18)`, so line 141 returns `V1 = 1000e18`.
5. After deducting the 0.01% LP fee (`0.1e18`), the trader receives `999.9e18` Quote tokens, and `_setReserve` updates `_BASE_RESERVE_ = 2500e18` and `_QUOTE_RESERVE_ = 0.1e18`, with `_BASE_TARGET_ = 1000e18` and `_RState_ = BELOW_ONE`.
6. The trader lost `500e18` Base tokens for zero Quote output. Furthermore, `getPMMState()` now computes `state.Q0 = state.Q + i * (state.B - state.B0) = 0.1e18 + 1500e18 = 1500.1e18` (inflated by `500e18` over the actual initial quote target of `1000e18`).

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:** Trades exceeding available reserves in `k == 0` pools do not revert; instead, the output is silently capped at `V1` while the pool absorbs the entire input token amount, and the virtual target (`Q0` or `B0`) computed by `adjustedTarget` becomes inflated by the uncompensated input amount.
- **Business Implications:** Traders suffer direct loss of funds on large swaps in `k == 0` pools if slippage protection is loose or omitted, and the pool's PMM state accounting (`Q0` / `B0` targets) is corrupted.

### Remediation

Revert with a custom error when the linear trade output exceeds the available reserve `V1` (or `>= V1` if zero reserve must be prevented) instead of silently clamping the output to `V1`.

```solidity
        if (k == 0) {
            uint256 output = DecimalMath.mulFloor(i, delta);
            if (output > V1) {
                revert ErrOutOfReserve();
            }
            return output;
        }
```

## Finding 30: Premature multiplication in Math._SolveQuadraticFunctionForTrade can disable zero-curvature MagicLP swaps

### Synopsis

`Math._SolveQuadraticFunctionForTrade` evaluates an overflowing fixed-point multiplication before applying its `k == 0` reserve cap. A sufficiently large swap therefore reverts despite a bounded valid result, and an excess-balance donation can deny swaps and strand liquidity in an affected `MagicLP` pool.

### Technical details

When `k == 0`, line 141 calculates `DecimalMath.mulFloor(i, delta)` before limiting the result to `V1`. `mulFloor` internally evaluates:

`(i * delta) / 1e18`

Because the contracts compile under Solidity 0.8, `i * delta` reverts whenever:

`delta > type(uint256).max / i`

This happens before division and before the ternary can cap the output. Calling `mulFloor` twice in the same expression does not provide any protection; the first evaluation already reverts.

The mathematically intended result remains representable because this branch returns at most `V1`. For MagicLP callers, `V1` is a reserve or target amount constrained to the pool’s `uint112` accounting domain. Therefore, if `i * delta` exceeds `uint256`, the fixed-point quotient is already vastly greater than `V1`, and the correct saturated result is simply `V1`.

MagicLP derives swap input from the difference between the contract’s token balance and its recorded reserve. Consequently, transferring an amount above the overflow threshold directly to a `k == 0` pool causes the subsequent swap quote to revert. Since execution never reaches the reserve update, the excess remains part of every later computed input, potentially keeping that swap direction unavailable.

### Assumptions and preconditions

- An honestly configured pool uses the supported `k == 0` setting.
- The price parameter `i` is nonzero.
- The attacker can transfer enough input tokens that `delta > type(uint256).max / i`.
- The input token permits balances and transfers of the required magnitude.
- For persistent denial of service, the pool has no permissionless recovery path that removes the excess balance or updates reserves without evaluating the affected quote.

### Proof of concept

1. Consider a pool or direct library invocation with:
   - `k = 0`
   - `i = 1e18`
   - `V0 = V1 = 1e18`
   - `delta = type(uint256).max / 1e18 + 1`
2. The exact fixed-point fair amount is `delta`, because `i` equals the fixed-point unit.
3. Since `delta > V1`, the intended capped output is `V1`, or `1e18`.
4. Invoke `_SolveQuadraticFunctionForTrade(V0, V1, delta, i, k)`.
5. Execution enters the branch at line 140.
6. Line 141 calls `DecimalMath.mulFloor(i, delta)`.
7. The intermediate product `1e18 * delta` exceeds `type(uint256).max`, causing Solidity panic `0x11`.
8. The function therefore reverts instead of returning the valid, bounded result `V1`.
9. To reproduce through MagicLP, mint or acquire the required input amount, transfer it directly to a `k == 0` pool, and call the corresponding sell method. The quote reverts before reserves are updated, so the excess continues to affect later calls.

### Impact assessment

**Severity: MEDIUM.**

The vulnerability causes deterministic denial of service for sufficiently large swaps in supported zero-curvature pools. An attacker can potentially make one swap direction persistently unusable by donating an overflowing input balance, disrupting routing, price quoting, withdrawals that depend on swaps, and other integrations.

There is no direct theft from the arithmetic failure itself. Exploitation requires control of a potentially very large token amount, with the exact threshold determined by `i`. Worst-case consequences include unavailable pool liquidity, stranded excess tokens, failed integrations, and migration or privileged recovery being required to restore operation.

### Remediation

Apply the reserve cap before evaluating the potentially overflowing multiplication. Because MagicLP’s `V1` values are within `uint112`, `V1 * 1e18` is safe and can be used to derive the minimum input that saturates the output:

```solidity
if (k == 0) {
    if (i == 0 || delta == 0 || V1 == 0) {
        return 0;
    }

    assert(V1 <= type(uint112).max);

    uint256 saturationInput = divCeil(V1 * DecimalMath.ONE, i);
    if (delta >= saturationInput) {
        return V1;
    }

    return DecimalMath.mulFloor(i, delta);
}
```

When `delta` is below `saturationInput`, `i * delta < V1 * 1e18`, so the multiplication is guaranteed not to overflow under the asserted reserve bound. If this library must support arbitrary `uint256` values for `V1`, use an audited 512-bit product comparison and capped `mulDiv` implementation instead of relying on the pool’s `uint112` invariant.

## Finding 31: Overflow in Math._SolveQuadraticFunctionForTrade can freeze MagicLP k=1 swaps

### Synopsis

`Math._SolveQuadraticFunctionForTrade` materializes `i * delta` in 256-bit arithmetic for `k == 1`. A sufficiently large price/input pair reverts before the bounded PMM output is calculated, allowing balance-based `MagicLP` swaps to be denied and previously transferred inputs to become unusable.

### Technical details

For `k == DecimalMath.ONE`, line 152 calculates `idelta = i * delta` using Solidity 0.8 checked arithmetic. If the mathematical product exceeds `type(uint256).max`, the operation reverts with arithmetic panic `0x11`.

This intermediate product can overflow even though the final trade output is representable. The branch’s output is asymptotically bounded below `V1`, because the relevant expression has the form:

`V1 * temp / (temp + 1e18)`

Consequently, increasing `i * delta` should make the output approach `V1`; it should not require representing the unbounded product in a single 256-bit word.

This is especially problematic for `MagicLP`, where the trade input is inferred from the contract’s current token balance minus its recorded reserve. If tokens were transferred to the pool in an earlier transaction and the resulting `delta` triggers line 152, every subsequent swap on that side observes the same excess balance and reverts before reserve accounting can advance. The condition therefore persists until another mechanism removes or accounts for the excess tokens.

The overflow occurs before any later arithmetic can process or reduce the product. It must be addressed at the initial multiplication rather than by relying on downstream calculations.

### Assumptions and preconditions

- The affected pool legitimately supports `k == 1`.
- The pool’s supported price normalization and token amounts permit `i * delta` to exceed 256 bits.
- A user or attacker can transfer the relevant input token directly to the pool before invoking the swap.
- The transfer and swap occur in separate transactions, so the swap revert does not roll back the earlier transfer.
- No immediately available recovery path removes or accounts for the overflowing excess balance.

### Proof of concept

1. Consider a `k == 1` pool state with `V0 = V1 = 1e18`.
2. Use individually valid `uint256` values:
   - `i = 2^200`
   - `delta = 2^56`
3. The exact intermediate product is `2^256`, which cannot be represented by `uint256`.
4. The mathematical PMM output is nevertheless valid and bounded below `V1`; with these values it approaches `1e18 - 1`.
5. Transfer `delta` input tokens to the pool in a separate transaction.
6. Invoke the corresponding `MagicLP` sell method.
7. Execution reaches line 152 and reverts with arithmetic panic `0x11`.
8. Invoke the sell method again without removing the excess balance. It calculates the same or a larger `delta` and reverts again, demonstrating persistent directional swap denial.

### Impact assessment

**Severity: Medium.**

The issue can disable one swap direction in an affected `k == 1` pool and can strand tokens transferred before the reverting call. Routers that transfer and swap atomically generally protect the initiating user through transaction rollback, but direct transfers or multi-transaction integrations remain exposed.

Exploitation requires a pool and input range capable of overflowing `i * delta`, potentially making the attack capital-intensive. In the worst case, the affected pool remains unable to process that swap direction until an authorized or permissionless recovery mechanism removes the excess, disrupting trading and integrations dependent on the pool.

### Remediation

Do not use `unchecked`, because wrapping the product would produce an invalid price.

Prefer evaluating the complete `k == 1` rational expression with an audited full-precision routine capable of handling multi-word intermediate numerators and denominators while preserving the intended round-down behavior. The implementation must avoid materializing `i * delta` in a single `uint256`.

As an alternative, establish and enforce a protocol-wide bound proving that the maximum effective price multiplied by the maximum processable input always fits in 256 bits. If out-of-range balances can still be transferred directly, also provide a safe permissionless mechanism to remove excess tokens or change the swap API to consume an explicitly bounded amount rather than the entire balance surplus.

Add invariant and boundary tests covering `delta = max / i`, `delta = max / i + 1`, both swap directions, and all supported price-normalization extremes.

## Finding 32: Catastrophic root cancellation in MagicLP.sellBase lets dust trades extract pool reserves

### Synopsis

MagicLP.sellBase reaches Math._SolveQuadraticFunctionForTrade, whose positive quadratic root subtracts nearly equal, floor-rounded values. For valid `K` values near `1e18`, a one-unit input receives an enormous output, allowing an unprivileged trader to extract pool reserves through dust swaps.

### Technical details

The quadratic solver calculates the positive root using the numerically unstable expression described at line 115:

`(-b + sqrt(b² + c)) / (2a)`

When `bSig` is true, line 191 implements this as `squareRoot - bAbs`. Both operands can be large and nearly equal. Because `sqrt` rounds down to an integer, subtracting them destroys almost all significant precision.

The subsequent `DecimalMath.divCeil` does not recover that precision. When `K` is close to `DecimalMath.ONE`, its denominator, `2 * (ONE - K)`, is very small. Consequently, losing less than one unit before the fixed-point division can underestimate the calculated post-trade reserve `V2` by up to approximately `0.5e18` raw units. Since the trader’s output is `V1 - V2`, this underestimation becomes a corresponding overpayment.

For an exact example, let:

- `E = DecimalMath.ONE = 1e18`
- `V0 = V1 = E`
- `delta = 1`
- `i = E`
- `k = E - 1`

The implementation calculates:

- `part2 = E²`
- `bAbs = E - 1`
- discriminant addition `= 4E - 4`
- radicand `= E² + 2E - 3`
- floor square root `= E`
- numerator `= E - (E - 1) = 1`
- denominator `= 2`
- `V2 = ceil(1 * E / 2) = E / 2`
- output `= V1 - V2 = E / 2`

Thus, one atomic unit of input is quoted half a token of output for an 18-decimal asset. The mathematically stable root instead leaves `V2` approximately equal to `E`, producing only a dust-sized output.

The output reaches `MagicLP.sellBase` through `PMMPricing._ROneSellBaseToken` and is transferred to the attacker after only percentage-based fees, which do not prevent the disproportionate payout.

### Assumptions and preconditions

- The affected pool uses a valid `K` sufficiently close to `DecimalMath.ONE`; the exact reproduction uses `ONE - 1`.
- The input and output assets have enough decimal precision for raw-unit cancellation to represent meaningful value; the exact example uses 18-decimal assets.
- The pool is at or near the `RState.ONE` equilibrium with nonzero reserves and targets.
- The attacker can transfer one atomic input unit to the pool and call `sellBase`.
- Normal swap fees are materially below 100%, so the erroneous output remains profitable after fees.
- No privileged account or malicious core deployment is required; the attack uses an otherwise supported curve parameter.

### Proof of concept

1. Consider an equilibrium MagicLP pool with:
   - `B = B0 = 1e18`
   - `Q = Q0 = 1e18`
   - `i = 1e18`
   - `K = 1e18 - 1`
   - `RState.ONE`
2. Transfer one raw unit of the base token to the pool.
3. Call `sellBase(attacker)`.
4. The call reaches `_SolveQuadraticFunctionForTrade(1e18, 1e18, 1, 1e18, 1e18 - 1)`.
5. The solver computes `bAbs = 1e18 - 1` and a floor square root of `1e18`.
6. Catastrophic cancellation reduces the root numerator to `1`.
7. Fixed-point ceiling division by `2` produces `V2 = 5e17`.
8. The solver returns `V1 - V2 = 5e17` output units.
9. After ordinary percentage fees, the attacker receives almost half an output token for one atomic input unit.
10. Repeat the analysis with the token directions reversed to verify that the same root calculation affects the symmetric pricing path.

### Impact assessment

**Severity: High.**

An unprivileged attacker can receive economically significant output for a negligible input, directly depleting LP-owned reserves. In the exact equilibrium example, one atomic input removes approximately half of a one-token output reserve before fees.

For larger pools, each dust trade can still receive a large fixed-point rounding windfall, making repeated extraction economically viable on low-cost networks. The worst case is substantial or complete loss of affected pool liquidity, insolvency of LP shares, broken downstream collateral valuation, and loss of confidence in MagicLP pricing.

The principal prerequisite is a valid curve parameter sufficiently close to `ONE`. Because the solver explicitly supports the full boundary range, callers should not be expected to recognize or compensate for this hidden numerical discontinuity.

### Remediation

Evaluate the positive root using its conjugate form when `bSig` is true:

`V2 = 2kV0² / (sqrt(b² + 4(1-k)kV0²) + b)`

This avoids subtracting nearly equal numbers and cancels the dangerously small `1 - k` factor. Use an audited full-precision `mulDiv` implementation and round `V2` upward so that trader output remains conservatively rounded downward.

```solidity
uint256 V2;
if (bSig) {
    uint256 kV0Squared = DecimalMath.mulFloor(k, V0) * V0;
    V2 = OZMath.mulDiv(
        kV0Squared,
        2,
        squareRoot + bAbs,
        OZMath.Rounding.Up
    );
} else {
    uint256 denominator = (DecimalMath.ONE - k) * 2;
    V2 = DecimalMath.divCeil(bAbs + squareRoot, denominator);
}

return V2 > V1 ? 0 : V1 - V2;
```

Import OpenZeppelin `Math` under the `OZMath` alias or use an equivalent audited 512-bit `mulDiv` implementation. As defense in depth, constrain `K` away from `ONE` until the stable solver is deployed, and add differential tests against arbitrary-precision arithmetic for dust inputs and boundary `K` values.
