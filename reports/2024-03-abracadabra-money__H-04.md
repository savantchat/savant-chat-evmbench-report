# Audit report — 2024-03-abracadabra-money/H-04

## Finding 1: Missing Zero Exchange Rate Check in `CauldronV4._isSolvent` Allows Undercollateralized Borrowing

### Synopsis

`CauldronV4._isSolvent` fails to validate that `_exchangeRate` is non-zero before multiplying it by the user's borrow balance. When `_exchangeRate` is zero (e.g., when `exchangeRate` is uninitialized and the oracle fails to update, or returns zero), any borrower with non-zero collateral is deemed solvent, enabling pool drainage.

### Technical details

In `CauldronV4`, a user's solvency is verified by `_isSolvent(address user, uint256 _exchangeRate)` (lines 183–200) and exposed publicly via `isSolvent(address user)` (lines 202–204).

Rather than dividing the collateral valuation by `_exchangeRate` (which would revert on division by zero when `_exchangeRate == 0`), line 199 multiplies the debt side of the inequality by `_exchangeRate`:

```solidity
return
    bentoBox.toAmount(
        collateral,
        collateralShare.mul(EXCHANGE_RATE_PRECISION / COLLATERIZATION_RATE_PRECISION).mul(COLLATERIZATION_RATE),
        false
    ) >=
    // Moved exchangeRate here instead of dividing the other side to preserve more precision
    borrowPart.mul(_totalBorrow.elastic).mul(_exchangeRate) / _totalBorrow.base;
```

However, `_isSolvent` does not check whether `_exchangeRate == 0`. A zero `_exchangeRate` is reachable in `_isSolvent`:
1. When a `CauldronV4` clone is initialized, the storage variable `exchangeRate` defaults to `0`.
2. During `borrow`, `removeCollateral`, or `cook`, `updateExchangeRate()` queries `oracle.get(oracleData)`. If `updated` is `false`, `updateExchangeRate()` falls back to returning the cached `exchangeRate` (`0` if never updated). Similarly, if the oracle returns `(true, 0)`, `_exchangeRate` is `0`.
3. `isSolvent(address user)` directly passes the storage variable `exchangeRate` to `_isSolvent(user, exchangeRate)`, which is `0` prior to the first successful `updateExchangeRate()` call.

When `_exchangeRate == 0`, the right-hand side `borrowPart.mul(_totalBorrow.elastic).mul(_exchangeRate) / _totalBorrow.base` evaluates to `0`. Since `bentoBox.toAmount(...)` is an unsigned integer (`>= 0`), the solvency check evaluates to `true` for any user holding at least `1` wei of `collateralShare`, regardless of the size of `borrowPart`.

### Assumptions and preconditions

1. A `CauldronV4` market has been deployed and funded with `magicInternetMoney` (MIM) in `bentoBox`.
2. Either `exchangeRate` has not yet been initialized (`exchangeRate == 0`) and `oracle.get(oracleData)` returns `updated = false`, or `oracle.get(oracleData)` returns a rate of `0` (or `isSolvent` is queried before `exchangeRate` is initialized).

### Proof of concept

1. A new `CauldronV4` instance is initialized and funded with 1,000,000 MIM in `bentoBox`. At this point, the storage variable `exchangeRate` is `0`.
2. A borrower deposits `1` wei of `collateral` into `bentoBox` and calls `addCollateral` to receive `userCollateralShare[borrower] = 1`.
3. While the oracle is temporarily unavailable (`oracle.get(oracleData)` returns `(false, 0)`), the borrower calls `borrow(borrower, 1_000_000e18)`.
4. Execution enters `_borrow`, assigning `userBorrowPart[borrower]` and transferring the MIM shares in `bentoBox` to the borrower.
5. The `solvent` modifier executes `updateExchangeRate()`, which sees `updated == false` and returns `_exchangeRate = exchangeRate = 0`.
6. `_isSolvent(borrower, 0)` computes:
   - Left-hand side: `bentoBox.toAmount(collateral, 1 * (1e18 / 1e5) * COLLATERIZATION_RATE, false) >= 0`.
   - Right-hand side: `borrowPart * _totalBorrow.elastic * 0 / _totalBorrow.base = 0`.
7. Because `lhs >= 0` holds, `_isSolvent` returns `true` and the transaction succeeds, allowing the borrower to take all 1,000,000 MIM backed by only 1 wei of collateral.

### Impact assessment

**Severity: High**

If `_exchangeRate` is `0` (due to an uninitialized `exchangeRate` paired with a failed oracle update, or an invalid `0` oracle response), the solvency check in `_isSolvent` completely breaks down. A user with dust collateral (1 wei) passes the solvency check for an arbitrarily large debt position, leading to theft of all borrowable MIM tokens in the Cauldron and bad debt for the protocol.

### Remediation

Validate that `_exchangeRate` is non-zero in `_isSolvent`, returning `false` if `_exchangeRate == 0`.

```solidity
    function _isSolvent(address user, uint256 _exchangeRate) virtual internal view returns (bool) {
        // accrue must have already been called!
        uint256 borrowPart = userBorrowPart[user];
        if (borrowPart == 0) return true;
        uint256 collateralShare = userCollateralShare[user];
        if (collateralShare == 0 || _exchangeRate == 0) return false;

        Rebase memory _totalBorrow = totalBorrow;

        return
            bentoBox.toAmount(
                collateral,
                collateralShare.mul(EXCHANGE_RATE_PRECISION / COLLATERIZATION_RATE_PRECISION).mul(COLLATERIZATION_RATE),
                false
            ) >=
            // Moved exchangeRate here instead of dividing the other side to preserve more precision
            borrowPart.mul(_totalBorrow.elastic).mul(_exchangeRate) / _totalBorrow.base;
    }
```

## Finding 2: Unchecked Oracle Update Status in `CauldronV4.solvent` Allows Borrowing and Collateral Withdrawal Using Stale or Uninitialized Exchange Rates

### Synopsis

In `CauldronV4`, the `solvent` modifier and `updateExchangeRate` function fail to enforce that the oracle returned a fresh, valid price. When `oracle.get` returns `updated == false`, `updateExchangeRate` falls back to the cached `exchangeRate`, allowing users to borrow or withdraw collateral using stale or uninitialized (`0`) prices.

### Technical details

In `src/cauldrons/CauldronV4.sol`, the `updateExchangeRate()` function (lines 217–227) queries `oracle.get(oracleData)` to obtain `(bool updated, uint256 rate)`. If `updated` is `true`, the state variable `exchangeRate` is updated to `rate`. However, if `updated` is `false` (indicating the oracle failed to fetch a fresh price or is offline/stale), `updateExchangeRate()` silently falls back to the previously cached `rate = exchangeRate` (lines 223–226).

The `solvent` modifier (lines 207–211), which guards critical state-changing operations such as `removeCollateral` (line 283) and `borrow` (line 320), invokes `updateExchangeRate()` but explicitly discards the `updated` boolean return value:

```solidity
modifier solvent() {
    _;
    (, uint256 _exchangeRate) = updateExchangeRate();
    require(_isSolvent(msg.sender, _exchangeRate), "Cauldron: user insolvent");
}
```

The same pattern occurs at the end of `cook()` (lines 529–532). Because `_isSolvent` calculates the required collateral as:

```solidity
bentoBox.toAmount(
    collateral,
    collateralShare.mul(EXCHANGE_RATE_PRECISION / COLLATERIZATION_RATE_PRECISION).mul(COLLATERIZATION_RATE),
    false
) >= borrowPart.mul(_totalBorrow.elastic).mul(_exchangeRate) / _totalBorrow.base
```

this design introduces two severe failure modes during solvency checks:
1. **Stale Cached-Price Acceptance:** If the collateral price drops significantly while the oracle is unable to provide a fresh rate (`updated == false`), `updateExchangeRate()` returns the outdated, lower `exchangeRate` (since `exchangeRate` represents how much collateral is needed to buy `1e18` MIM, a lower rate means higher collateral valuation). Users can continue borrowing MIM or withdrawing collateral at the stale favorable valuation, creating bad debt for the protocol.
2. **Uninitialized Rate Acceptance:** On a newly initialized Cauldron clone where `exchangeRate` has not yet been set (`exchangeRate == 0`), if `oracle.get(oracleData)` returns `updated == false`, `_exchangeRate` evaluates to `0`. In `_isSolvent`, the right-hand side (`borrowPart * _totalBorrow.elastic * 0 / _totalBorrow.base`) evaluates to `0`. Consequently, any user with at least `1` wei of `collateralShare` is deemed solvent regardless of how much MIM they borrow, enabling complete drainage of the Cauldron's available MIM liquidity.

### Assumptions and preconditions

1. A `CauldronV4` instance is deployed and funded with borrowable MIM in `BentoBox`, or holds user collateral.
2. Either:
   - The underlying `oracle.get(oracleData)` temporarily fails to update and returns `(false, ...)` while the true market price of the collateral has depreciated relative to the cached `exchangeRate`, OR
   - The Cauldron is newly initialized (`exchangeRate == 0`) and `oracle.get(oracleData)` returns `(false, ...)`.

### Proof of concept

1. Consider a `CauldronV4` market where `1 WETH = 2000 MIM`, so `exchangeRate = 5e14` (`0.0005e18` WETH per `1e18` MIM).
2. A user deposits `1 WETH` as collateral and borrows `1000 MIM`.
3. The market price of WETH crashes to `1000 MIM` (true `exchangeRate` should be `1e15`), and the oracle enters a failure/stale state where `oracle.get(oracleData)` returns `(false, 0)`.
4. The user calls `borrow(user, 400e18)` (or `removeCollateral`).
5. Inside the `solvent` modifier:
   - `updateExchangeRate()` calls `oracle.get(oracleData)`, which returns `updated = false`.
   - `updateExchangeRate()` falls back to the stale `exchangeRate = 5e14`.
   - `solvent()` ignores `updated == false` and calls `_isSolvent(msg.sender, 5e14)`.
   - Using the stale rate of `5e14`, the user's `1 WETH` collateral is still valued at `2000 MIM`, so borrowing an additional `400 MIM` passes the solvency check even though the position is actually undercollateralized (`1400 MIM` debt against `1000 MIM` worth of WETH).
6. Alternatively, if `exchangeRate == 0` on a fresh Cauldron and `oracle.get` returns `(false, 0)`, a user deposits `1 wei` of collateral and calls `borrow()` for the entire MIM balance of the Cauldron. `_exchangeRate` is `0`, so the required collateral amount in `_isSolvent` is `0`, and the transaction succeeds.

### Impact assessment

**Severity: High**

Accepting a stale or uninitialized exchange rate during solvency-sensitive operations (`borrow` and `removeCollateral`) allows users to open undercollateralized debt positions or withdraw collateral that backs existing debt. In the worst case (an uninitialized rate of `0` or a severe market downturn coupled with oracle downtime), attackers can drain all available MIM from the Cauldron, resulting in direct, unrecoverable bad debt and loss of protocol funds.

### Remediation

In the `solvent` modifier (and in `cook` when `status.needsSolvencyCheck` is true), require that `updateExchangeRate()` either freshly updated the rate or that `exchangeRate` is valid and updated, or strictly require `updated == true` and `_exchangeRate > 0` when checking solvency:

```solidity
    modifier solvent() {
        _;
        (bool updated, uint256 _exchangeRate) = updateExchangeRate();
        require(updated && _exchangeRate > 0, "Cauldron: rate not updated");
        require(_isSolvent(msg.sender, _exchangeRate), "Cauldron: user insolvent");
    }
```

## Finding 3: Stale Reserve Comparison in `MagicLP.flashLoan` Allows Draining Rebasing Yield Drift and Breaks Repayment Accounting

### Synopsis

`MagicLP.flashLoan` validates loan repayment and calculates swap inputs by comparing post-callback token balances against cached `_BASE_RESERVE_` and `_QUOTE_RESERVE_` values instead of pre-loan balances. With rebasing or yield-bearing tokens, positive balance drift can be stolen for free and negative drift causes valid repayments to revert.

### Technical details

In `src/mimswap/MagicLP.sol`, `flashLoan` (lines 290–309) transfers `baseAmount` and `quoteAmount` to `assetTo` (lines 291–292), invokes `ICallee(assetTo).FlashLoanCall` (line 295), and then fetches the current token balances:

```solidity
uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
```

Instead of recording the pool's actual token balances prior to transferring out the flash-loaned assets, `flashLoan` checks whether the loan was repaid—and computes the implicit swap input—by comparing `baseBalance` and `quoteBalance` against the cached storage variables `_BASE_RESERVE_` and `_QUOTE_RESERVE_` (lines 302–309):

```solidity
if (baseBalance < _BASE_RESERVE_ && quoteBalance < _QUOTE_RESERVE_) {
    revert ErrFlashLoanFailed();
}

if (baseBalance < _BASE_RESERVE_) {
    uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
```

While `sellShares` (lines 431–436) distributes the live `balanceOf(address(this))` pro-rata to liquidity providers, `_BASE_RESERVE_` and `_QUOTE_RESERVE_` only reflect the balances at the last `_sync()` or reserve update. When a pool holds tokens subject to rebasing or yield-driven balance drift (such as Blast native yield tokens like USDB or WETH before claimable mode is enabled via `updateTokenClaimables`, or any rebasing ERC20 token):

1. **Positive Balance Drift (Yield / Positive Rebase):** As token balances increase automatically via rebasing, `_BASE_TOKEN_.balanceOf(address(this)) > _BASE_RESERVE_` and/or `_QUOTE_TOKEN_.balanceOf(address(this)) > _QUOTE_RESERVE_`. Any caller can invoke `flashLoan` to withdraw the entire un-synced balance drift (`baseBalance - _BASE_RESERVE_` and `quoteBalance - _QUOTE_RESERVE_`) with `data = ""` and zero repayment, because post-withdrawal balances still satisfy `baseBalance >= _BASE_RESERVE_` and `quoteBalance >= _QUOTE_RESERVE_`. Alternatively, the caller can request `baseAmount` and rely on the un-synced quote drift in `quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_)` (line 309) to pay for the base tokens without transferring any quote tokens. At the end of `flashLoan`, `_sync()` updates the reserves, permanently depriving LPs of the accrued yield.
2. **Negative Balance Drift (Negative Rebase) or Transfer-Tax:** If a rebasing token adjusts downward such that the live balance falls below `_BASE_RESERVE_` (or if a transfer-tax token reduces the net amount received upon repayment), a borrower who repays the exact `baseAmount` and `quoteAmount` borrowed will still leave `baseBalance < _BASE_RESERVE_`. Execution then enters line 308 and reverts with `ErrFlashLoanFailed()` unless the borrower subsidizes the pool's entire historical reserve deficit.

### Assumptions and preconditions

1. A `MagicLP` (or `BlastMagicLP`) pool is initialized with at least one token whose balance can change outside of pool swaps (e.g., a rebasing token or a Blast native yield token not yet configured as claimable in `BlastTokenRegistry`).
2. The token's balance on the pool contract drifts upward (positive rebase/yield) or downward (negative rebase) relative to `_BASE_RESERVE_` or `_QUOTE_RESERVE_` before `sync()` is called.

### Proof of concept

1. Suppose a `MagicLP` pool holds `1,000e18` of `_BASE_TOKEN_` and `1,000e18` of `_QUOTE_TOKEN_`, with `_BASE_RESERVE_ = 1,000e18` and `_QUOTE_RESERVE_ = 1,000e18`.
2. Due to a positive rebase (yield accrual), the pool's actual balances increase by `50e18` each, so `_BASE_TOKEN_.balanceOf(address(pool)) == 1,050e18` and `_QUOTE_TOKEN_.balanceOf(address(pool)) == 1,050e18`, while `_BASE_RESERVE_` and `_QUOTE_RESERVE_` remain `1,000e18`.
3. An attacker calls `pool.flashLoan(50e18, 50e18, attacker, "")`.
4. Lines 291–292 transfer `50e18` of `_BASE_TOKEN_` and `50e18` of `_QUOTE_TOKEN_` to `attacker`. Since `data.length == 0`, no callback is executed and the attacker repays nothing.
5. Lines 298–299 read `baseBalance = 1,000e18` and `quoteBalance = 1,000e18`.
6. Lines 302 and 308 evaluate `baseBalance < _BASE_RESERVE_` (`1,000e18 < 1,000e18`), which is `false`, and line 330 evaluates `quoteBalance < _QUOTE_RESERVE_`, which is also `false`.
7. The flash loan succeeds without repayment, transferring `50e18` of both tokens to the attacker at the expense of the pool's liquidity providers.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Un-synced positive balance drift from rebasing or yield-bearing tokens can be extracted for free via `flashLoan` without repayment, or credited as `quoteInput` / `baseInput` to drain the counterparty reserve token. Conversely, negative balance drift breaks `flashLoan` repayment accounting unless the borrower covers the entire reserve shortfall.
- **Business Impact:** Liquidity providers lose accrued rebase yield in the pool to arbitrary callers, and flash loan integrators face unexpected transaction reverts or overcharges when interacting with rebasing or fee-on-transfer tokens.

### Remediation

Synchronize the reserves (or snapshot the pre-loan balances and update `_BASE_RESERVE_` and `_QUOTE_RESERVE_`) at the start of `flashLoan` before transferring tokens out, so that repayment verification and swap input calculations are measured against the pool's actual pre-loan balances:

```solidity
    function flashLoan(uint256 baseAmount, uint256 quoteAmount, address assetTo, bytes calldata data) external nonReentrant {
        _sync();

        _transferBaseOut(assetTo, baseAmount);
        _transferQuoteOut(assetTo, quoteAmount);

        if (data.length > 0) {
            ICallee(assetTo).FlashLoanCall(msg.sender, baseAmount, quoteAmount, data);
        }

        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));

        // no input -> pure loss
        if (baseBalance < _BASE_RESERVE_ && quoteBalance < _QUOTE_RESERVE_) {
            revert ErrFlashLoanFailed();
        }

        // sell quote case
        // quote input + base output
        if (baseBalance < _BASE_RESERVE_) {
            uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
```

## Finding 4: Overpayment and Partial-Settlement Desynchronization in `MagicLP.flashLoan` Corrupts PMM Target Accounting and Locks Excess Funds

### Synopsis

In `MagicLP.flashLoan`, cross-asset repayment calculates `receiveBaseAmount`, `mtFee`, `newRState`, and `newQuoteTarget` from the full `quoteInput`, but only verifies `uint256(_BASE_RESERVE_) - baseBalance <= receiveBaseAmount` without transferring the excess `receiveBaseAmount - (uint256(_BASE_RESERVE_) - baseBalance)` to `assetTo`, locking borrower overpayment and desynchronizing PMM state targets from actual reserves.

### Technical details

When a flash loan in `MagicLP.flashLoan` results in a net base token deficit (`baseBalance < _BASE_RESERVE_`), the pool settles the deficit as a quote-to-base swap using the quote token surplus `quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_)` (lines 308–326 of `src/mimswap/MagicLP.sol`):

```solidity
if (baseBalance < _BASE_RESERVE_) {
    uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
    (uint256 receiveBaseAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newQuoteTarget) = querySellQuote(
        tx.origin,
        quoteInput
    );

    if (uint256(_BASE_RESERVE_) - baseBalance > receiveBaseAmount) {
        revert ErrFlashLoanFailed();
    }

    _transferBaseOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);
    if (_RState_ != uint32(newRState)) {
        _QUOTE_TARGET_ = newQuoteTarget.toUint112();
        _RState_ = uint32(newRState);
        emit RChange(newRState);
    }
    emit Swap(address(_QUOTE_TOKEN_), address(_BASE_TOKEN_), quoteInput, receiveBaseAmount, msg.sender, assetTo);
}
```

In a standard `sellQuote` call, `querySellQuote(tx.origin, quoteInput)` computes `receiveBaseAmount` (net of `lpFee` and `mtFee`), `newRState`, and `newQuoteTarget` under the invariant that `receiveBaseAmount + mtFee` base tokens leave `_BASE_RESERVE_`. `sellQuote` enforces this by transferring `receiveBaseAmount` to the recipient and `mtFee` to the maintainer before updating `_BASE_RESERVE_`.

In `flashLoan`, however, the borrower has only withdrawn `baseDeficit = uint256(_BASE_RESERVE_) - baseBalance` prior to settlement. Line 315 checks `baseDeficit > receiveBaseAmount`, permitting `baseDeficit < receiveBaseAmount` (for example, when a borrower partially repays in base tokens and overpays in quote tokens, or includes a slippage buffer in `quoteInput`), but never transfers the remaining `receiveBaseAmount - baseDeficit` base tokens to `assetTo`.

This causes three accounting inconsistencies:
1. **Unrefunded Overpayment & Fee Overcharge:** The difference `receiveBaseAmount - baseDeficit` is never sent to `assetTo` and is instead absorbed into `_BASE_RESERVE_` by `_sync()` at line 350, while `mtFee` is charged on the full `quoteInput` and `Swap` emits the full `receiveBaseAmount`.
2. **`RState.ONE` Invariant Violation:** If the pool starts in `RState.BELOW_ONE` (`_BASE_RESERVE_ > _BASE_TARGET_` and `_QUOTE_RESERVE_ < state.Q0`) and `quoteInput == state.Q0 - state.Q` (`backToOnePayQuote`) while `baseDeficit < receiveBaseAmount`, `_RState_` is updated to `RState.ONE` and `_QUOTE_TARGET_` is set to `state.Q0`, even though `_BASE_RESERVE_` remains greater than `_BASE_TARGET_`. Because `PMMPricing.adjustedTarget` does not recompute targets when `R == RState.ONE`, `getMidPrice()` (lines 209–211 of `PMMPricing.sol`) computes `(state.B0 * state.B0) / state.B / state.B < 1`, distorting the pool's mid-price, TWAP accumulator (`_twapUpdate`), and `MagicLpAggregator` oracle price.
3. **`RState.ABOVE_ONE` Target Inflation:** When `newRState == RState.ABOVE_ONE`, `_RState_` is set to `ABOVE_ONE` and `_QUOTE_TARGET_` is set to `state.Q0`, while `_BASE_RESERVE_` retains the unwithdrawn `receiveBaseAmount - baseDeficit` base tokens. On subsequent calls to `getPMMState()`, `PMMPricing.adjustedTarget` (lines 193–200 of `PMMPricing.sol`) solves for `state.B0` from the inflated `state.B` and `state.Q - state.Q0` as if `receiveBaseAmount` had left `state.B`. A subsequent `sellBase` that transitions `_RState_` out of `ABOVE_ONE` then writes this inflated `newBaseTarget` (`state.B0`) permanently into `_BASE_TARGET_`.

(The same issue exists symmetrically in the `quoteBalance < _QUOTE_RESERVE_` branch at lines 330–348.)

### Assumptions and preconditions

1. An initialized `MagicLP` pool has non-zero reserves (`_BASE_RESERVE_ > 0` and `_QUOTE_RESERVE_ > 0`).
2. A caller invokes `flashLoan` where `baseBalance < _BASE_RESERVE_` after `FlashLoanCall` and provides `quoteInput = quoteBalance - _QUOTE_RESERVE_` such that `receiveBaseAmount > _BASE_RESERVE_ - baseBalance`.

### Proof of concept

1. Assume an initialized `MagicLP` pool in `RState.BELOW_ONE` with `_BASE_TARGET_ = 1000e18`, `_BASE_RESERVE_ = 1500e18`, `_QUOTE_RESERVE_ = 500.1e18`, and `state.Q0 = 1000e18` (e.g., following a `500e18` `sellBase` trade on a balanced `1000e18 / 1000e18` pool).
2. A borrower calls `flashLoan(1, 0, borrower, data)` borrowing `1 wei` of `_BASE_TOKEN_`.
3. Inside `FlashLoanCall`, the borrower transfers `quoteInput = state.Q0 - _QUOTE_RESERVE_ = 499.9e18` of `_QUOTE_TOKEN_` into the pool.
4. `flashLoan` calls `querySellQuote(tx.origin, 499.9e18)`, which hits the `payQuoteAmount == backToOnePayQuote` branch in `PMMPricing.sellQuoteToken` (line 92 of `PMMPricing.sol`) and returns:
   - `receiveBaseAmount ≈ 499.95e18` (i.e., `backToOneReceiveBase = 500e18` minus fees),
   - `newRState = PMMPricing.RState.ONE`,
   - `newQuoteTarget = 1000e18`.
5. At line 315, `uint256(_BASE_RESERVE_) - baseBalance` is `1 wei`, which is `<= receiveBaseAmount`, so the check passes without reverting and without transferring the remaining `~499.95e18` `_BASE_TOKEN_` to `borrower`.
6. Lines 320–323 set `_QUOTE_TARGET_ = 1000e18` and `_RState_ = uint32(RState.ONE)`.
7. Line 350 calls `_sync()`, setting `_BASE_RESERVE_ = 1500e18 - 1 - mtFee` and `_QUOTE_RESERVE_ = 1000e18`.
8. The pool is now in `_RState_ == RState.ONE` with `_BASE_TARGET_ = 1000e18` and `_BASE_RESERVE_ ≈ 1500e18`. Calling `getMidPrice()` evaluates `(state.B0 * state.B0) / state.B / state.B = (1000e18)^2 / (1500e18)^2 ≈ 0.444e18` instead of `1e18`, corrupting the pool mid-price and TWAP, while the borrower's `~499.95e18` base token output remains locked in the pool.

### Impact assessment

- **Severity:** Medium / High.
- **Technical Impact:** Any excess quote input in a cross-asset flash loan repayment is retained by the pool without transferring the corresponding `receiveBaseAmount - (_BASE_RESERVE_ - baseBalance)` base tokens to `assetTo`, while `_RState_` and `_QUOTE_TARGET_` are updated as if the full `receiveBaseAmount` had exited the pool. This breaks the PMM invariant between reserves and target reserves (`_BASE_TARGET_` / `_QUOTE_TARGET_`), skewing `getMidPrice()`, the TWAP price accumulator, and `MagicLpAggregator` pricing.
- **Business Impact:** Borrowers who overpay or use slippage buffers during cross-asset flash loan repayments lose the excess value, and attackers or accidental overpayments can desynchronize pool pricing and target reserve parameters.

### Remediation

Transfer any excess `receiveBaseAmount` above the net base token deficit `uint256(_BASE_RESERVE_) - baseBalance` to `assetTo` (and symmetrically in the `quoteBalance < _QUOTE_RESERVE_` branch) so that the actual reserve reduction matches the swap output used to compute `mtFee`, `newRState`, and `newQuoteTarget`.

```solidity
        // sell quote case
        // quote input + base output
        if (baseBalance < _BASE_RESERVE_) {
            uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
            (uint256 receiveBaseAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newQuoteTarget) = querySellQuote(
                tx.origin,
                quoteInput
            );

            uint256 baseDeficit = uint256(_BASE_RESERVE_) - baseBalance;
            if (baseDeficit > receiveBaseAmount) {
                revert ErrFlashLoanFailed();
            }
            if (receiveBaseAmount > baseDeficit) {
                _transferBaseOut(assetTo, receiveBaseAmount - baseDeficit);
            }

            _transferBaseOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);
            if (_RState_ != uint32(newRState)) {
                _QUOTE_TARGET_ = newQuoteTarget.toUint112();
                _RState_ = uint32(newRState);
                emit RChange(newRState);
            }
            emit Swap(address(_QUOTE_TOKEN_), address(_BASE_TOKEN_), quoteInput, receiveBaseAmount, msg.sender, assetTo);
        }
```

## Finding 5: Divergence Between Quoted and Realized Quote Outflow in `MagicLP.flashLoan` Corrupts PMM Regime and Target State

### Synopsis

In `MagicLP.flashLoan`, the sell-base settlement branch computes `newRState` and `newBaseTarget` via `querySellBase` assuming `receiveQuoteAmount` quote tokens leave the pool, but only requires `_QUOTE_RESERVE_ - quoteBalance <= receiveQuoteAmount`, allowing partial quote withdrawals to trigger invalid PMM regime transitions and reserve-target invariant violations.

### Technical details

In `src/mimswap/MagicLP.sol`, `flashLoan` supports settling a flash loan as a base-for-quote swap when `quoteBalance < _QUOTE_RESERVE_` (lines 330–350):

```solidity
if (quoteBalance < _QUOTE_RESERVE_) {
    uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
    (uint256 receiveQuoteAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newBaseTarget) = querySellBase(
        tx.origin,
        baseInput
    );

    if (uint256(_QUOTE_RESERVE_) - quoteBalance > receiveQuoteAmount) {
        revert ErrFlashLoanFailed();
    }

    _transferQuoteOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);
    if (_RState_ != uint32(newRState)) {
        _BASE_TARGET_ = newBaseTarget.toUint112();
        _RState_ = uint32(newRState);
        emit RChange(newRState);
    }
    emit Swap(address(_BASE_TOKEN_), address(_QUOTE_TOKEN_), baseInput, receiveQuoteAmount, msg.sender, assetTo);
}

_sync();
```

Unlike `sellBase`, which explicitly transfers `receiveQuoteAmount` out of the pool (`_transferQuoteOut(to, receiveQuoteAmount)`) so that the post-swap quote balance equals `_QUOTE_RESERVE_ - receiveQuoteAmount - mtFee`, `flashLoan` only verifies that the net quote outflow `uint256(_QUOTE_RESERVE_) - quoteBalance` does not exceed `receiveQuoteAmount`.

`querySellBase(tx.origin, baseInput)` calls `PMMPricing.sellBaseToken(state, baseInput)`, which determines `newRState` and `newBaseTarget` (`state.B0`) under the assumption that the full `receiveQuoteAmount + mtFee` is removed from `_QUOTE_RESERVE_`:

1. **Invalid transition from `ABOVE_ONE` to `ONE` (`_QUOTE_RESERVE_ > _QUOTE_TARGET_` in `RState.ONE`):**
   When `_RState_ == RState.ABOVE_ONE`, `_BASE_RESERVE_ < state.B0` and `_QUOTE_RESERVE_ > _QUOTE_TARGET_`. In this regime, `_QUOTE_TARGET_` (`Q0`) is fixed in storage while `state.B0` is dynamically derived in `PMMPricing.adjustedTarget(state)` from `state.B` and `state.Q - state.Q0`.
   If a borrower supplies `baseInput == state.B0 - state.B` (`backToOnePayBase`) while taking only `1 wei` of quote token (`_QUOTE_RESERVE_ - quoteBalance == 1 <= receiveQuoteAmount`), `querySellBase` returns `newRState = RState.ONE` and `newBaseTarget = state.B0`.
   `flashLoan` sets `_RState_ = RState.ONE` and `_BASE_TARGET_ = state.B0`, while `_sync()` sets `_BASE_RESERVE_ = state.B0` and `_QUOTE_RESERVE_ = quoteBalance - mtFee = state.Q - 1 - mtFee`.
   Because `PMMPricing.adjustedTarget` performs no target adjustment when `state.R == RState.ONE`, the pool is left in `RState.ONE` with `_QUOTE_RESERVE_ > _QUOTE_TARGET_`. Subsequent `sellBase` calls in `RState.ONE` execute `_ROneSellBaseToken` using `state.Q0` (`_QUOTE_TARGET_`) rather than the actual `_QUOTE_RESERVE_`, mispricing swaps until a regime change occurs.

2. **Invalid transition from `ABOVE_ONE` to `BELOW_ONE`:**
   If `_RState_ == RState.ABOVE_ONE` and the borrower supplies `baseInput > state.B0 - state.B` while only withdrawing `1 wei` of quote token, `flashLoan` transitions `_RState_` to `RState.BELOW_ONE` and sets `_BASE_TARGET_ = state.B0`, even though `_QUOTE_RESERVE_` remains greater than `_QUOTE_TARGET_`. On the next call to `getPMMState()`, `PMMPricing.adjustedTarget` in `RState.BELOW_ONE` recomputes `state.Q0` via `Math._SolveQuadraticFunctionForTarget(state.Q, state.B - state.B0, state.i, state.K)` using the unreduced quote reserve `state.Q`, permanently inflating the effective quote target `Q0`.

3. **Inaccurate `Swap` event accounting:**
   Line 347 emits `Swap(..., baseInput, receiveQuoteAmount, ...)` even though the actual quote amount taken by `assetTo` is `uint256(_QUOTE_RESERVE_) - quoteBalance`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized and has active liquidity (`_BASE_RESERVE_ > 0` and `_QUOTE_RESERVE_ > 0`).
2. The pool is in `RState.ABOVE_ONE` (reachable via any prior `sellQuote` trade) or `RState.ONE`.
3. A caller invokes `flashLoan` and leaves `quoteBalance = _QUOTE_RESERVE_ - 1` while transferring `baseInput > 0` of `_BASE_TOKEN_` into the pool.

### Proof of concept

1. Initialize a `MagicLP` pool with `1000e18` base tokens and `1000e18` quote tokens (`_BASE_RESERVE_ = _BASE_TARGET_ = 1000e18`, `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1000e18`, `_RState_ = RState.ONE`).
2. Execute a `sellQuote` trade of `100e18` quote tokens so the pool transitions to `_RState_ = RState.ABOVE_ONE`, with `_QUOTE_RESERVE_ = 1100e18 > _QUOTE_TARGET_ (1000e18)` and `_BASE_RESERVE_ < _BASE_TARGET_`.
3. Query `getPMMState()` to obtain `state.B0` and compute `backToOnePayBase = state.B0 - state.B`.
4. Call `flashLoan(0, 1, callee, data)` where `callee.FlashLoanCall` transfers `backToOnePayBase` of `_BASE_TOKEN_` to the pool and keeps the `1 wei` of `_QUOTE_TOKEN_`.
5. Inside `flashLoan`:
   - `quoteBalance` is `_QUOTE_RESERVE_ - 1 < _QUOTE_RESERVE_`.
   - `baseInput` equals `backToOnePayBase`.
   - `querySellBase` returns `newRState = RState.ONE`, `newBaseTarget = state.B0`, and `receiveQuoteAmount` (~`100e18`).
   - `_QUOTE_RESERVE_ - quoteBalance` (`1`) is `<= receiveQuoteAmount`, so the check at line 337 passes.
   - `_BASE_TARGET_` is set to `state.B0` and `_RState_` is set to `RState.ONE`.
   - `_sync()` sets `_BASE_RESERVE_ = state.B0` and `_QUOTE_RESERVE_ = 1100e18 - 1 - mtFee`.
6. Observe that `_RState_ == RState.ONE` and `_BASE_RESERVE_ == _BASE_TARGET_`, but `_QUOTE_RESERVE_` (~`1100e18`) is significantly greater than `_QUOTE_TARGET_` (`1000e18`). Calling `getPMMState()` now returns `state.Q = ~1100e18` and `state.Q0 = 1000e18` in `RState.ONE`, breaking the PMM target invariant `Q == Q0` at `R = 1`.

### Impact assessment

- **Severity:** Medium / High.
- **Technical Impact:** The PMM invariant linking `_RState_`, `_BASE_RESERVE_`, `_QUOTE_RESERVE_`, `_BASE_TARGET_`, and `_QUOTE_TARGET_` is broken because quoted regime transitions are applied to a realized reserve state that retained the unwithdrawn quote tokens. This distorts PMM pricing curves (`getMidPrice`, `querySellBase`, `querySellQuote`), corrupts TWAP updates in `_twapUpdate()`, and emits inaccurate `Swap` event amounts.

### Remediation

Require the net quote token outflow in `flashLoan` to match `receiveQuoteAmount` (or transfer the remaining `receiveQuoteAmount - (_QUOTE_RESERVE_ - quoteBalance)` quote tokens to `assetTo`) so that the realized post-settlement reserves always match the state quoted by `querySellBase`.

```solidity
        // sell base case
        // base input + quote output
        if (quoteBalance < _QUOTE_RESERVE_) {
            uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
            (uint256 receiveQuoteAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newBaseTarget) = querySellBase(
                tx.origin,
                baseInput
            );

            if (uint256(_QUOTE_RESERVE_) - quoteBalance != receiveQuoteAmount) {
                revert ErrFlashLoanFailed();
            }

            _transferQuoteOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);
            if (_RState_ != uint32(newRState)) {
                _BASE_TARGET_ = newBaseTarget.toUint112();
                _RState_ = uint32(newRState);
                emit RChange(newRState);
            }
            emit Swap(address(_BASE_TOKEN_), address(_QUOTE_TOKEN_), baseInput, receiveQuoteAmount, msg.sender, assetTo);
        }

        _sync();
```

## Finding 6: Retroactive Post-Trade Price Attribution in `MagicLP.flashLoan` Allows Single-Block TWAP Manipulation

### Synopsis

In `MagicLP.flashLoan`, state variables determining the pool's mid-price (`_BASE_TARGET_`, `_RState_`, `_BASE_RESERVE_`, and `_QUOTE_RESERVE_`) are updated before `_twapUpdate()` accumulates the time-weighted price. This attributes the post-trade price to the entire elapsed time since the last observation, enabling flash-swap manipulation of `_BASE_PRICE_CUMULATIVE_LAST_`.

### Technical details

The `MagicLP` contract maintains a cumulative price accumulator `_BASE_PRICE_CUMULATIVE_LAST_` updated in `_twapUpdate()` (lines 545–558):

```solidity
function _twapUpdate() internal {
    uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
    uint32 timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;

    if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
        unchecked {
            _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
        }
    }

    _BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
}
```

For a time-weighted average price (TWAP) accumulator to be manipulation-resistant, the price accumulated over the elapsed interval `[_BLOCK_TIMESTAMP_LAST_, blockTimestamp]` must be the price that was in effect *prior* to the state change in the current transaction.

However, in the `quoteBalance < _QUOTE_RESERVE_` branch of `MagicLP.flashLoan` (lines 330–350):
1. Lines 342–345 overwrite `_BASE_TARGET_` and `_RState_` with the post-swap `newBaseTarget` and `newRState` values before any price observation is recorded.
2. Line 350 invokes `_sync()`, which updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to the new token balances (lines 571–576) and only then calls `_twapUpdate()` (line 578).
3. When `_twapUpdate()` calls `getMidPrice()`, `getPMMState()` reads the newly updated `_BASE_RESERVE_`, `_QUOTE_RESERVE_`, `_BASE_TARGET_`, `_QUOTE_TARGET_`, and `_RState_`.

As a result, the new post-trade `getMidPrice()` is multiplied by the entire `timeElapsed` since the last pool interaction. Because subsequent transactions in the same block see `timeElapsed == 0`, an attacker can use a flash loan to skew the pool reserves and R-state on the first interaction in a block, lock the manipulated `getMidPrice() * timeElapsed` into `_BASE_PRICE_CUMULATIVE_LAST_`, and immediately swap back within the same transaction or block at minimal cost (only paying the pool fee).

### Assumptions and preconditions

1. An initialized `MagicLP` clone has non-zero reserves (`_BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0`).
2. At least one second has elapsed since the pool's last state-updating transaction (`uint32(block.timestamp % 2 ** 32) > _BLOCK_TIMESTAMP_LAST_`).
3. An external consumer or keeper relies on `_BASE_PRICE_CUMULATIVE_LAST_` to compute a TWAP over a window covering this interval.

### Proof of concept

1. Assume a `MagicLP` pool has `_BASE_RESERVE_ = 1,000,000e18`, `_QUOTE_RESERVE_ = 1,000,000e18`, `_I_ = 1e18`, `_K_ = 1e18`, `_RState_ = ONE`, and `_BLOCK_TIMESTAMP_LAST_ = t_0`. Initially, `getMidPrice() = 1e18`.
2. No trades occur for `timeElapsed = 3,600` seconds (`block.timestamp = t_0 + 3,600`). The true time-weighted price over those 3,600 seconds was `1e18`, so `_BASE_PRICE_CUMULATIVE_LAST_` should increase by `1e18 * 3,600`.
3. At `t_0 + 3,600`, a caller invokes `flashLoan(0, quoteAmount, attackerContract, data)` borrowing a large `quoteAmount` (e.g., `800,000e18` quote tokens) and transferring in the required `baseInput` during `FlashLoanCall` so that `quoteBalance < _QUOTE_RESERVE_` (lines 330–348).
4. Lines 343–344 set `_BASE_TARGET_ = newBaseTarget` and `_RState_ = uint32(RState.BELOW_ONE)`.
5. Line 350 calls `_sync()`, which sets `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to the skewed balances and calls `_twapUpdate()`.
6. `_twapUpdate()` computes `getMidPrice()` under `RState.BELOW_ONE` with the depleted `_QUOTE_RESERVE_`, yielding a significantly lower mid-price (e.g., `~0.04e18`), and adds `getMidPrice() * 3,600` to `_BASE_PRICE_CUMULATIVE_LAST_`, then sets `_BLOCK_TIMESTAMP_LAST_ = t_0 + 3,600`.
7. Immediately in the next call within the same transaction (`block.timestamp == t_0 + 3,600`), the caller swaps the quote tokens back for base tokens. Because `timeElapsed == 0` in `_twapUpdate()`, the cumulative price is not updated again, leaving `_BASE_PRICE_CUMULATIVE_LAST_` permanently corrupted over the 3,600-second window while the caller only pays the pool swap fees.

### Impact assessment

- **Severity:** Medium / High (depending on downstream TWAP consumers).
- **Direct Technical Effect:** The cumulative price accumulator `_BASE_PRICE_CUMULATIVE_LAST_` records post-trade spot prices across historical idle intervals (`timeElapsed`) instead of pre-trade prices.
- **Business Implications:** Any oracle or integration deriving a TWAP from `_BASE_PRICE_CUMULATIVE_LAST_` can be manipulated atomically within a single transaction without holding capital across blocks.

### Remediation

Update the TWAP accumulator using the pre-trade pool state *before* mutating `_BASE_TARGET_`, `_QUOTE_TARGET_`, `_RState_`, `_BASE_RESERVE_`, or `_QUOTE_RESERVE_`. Specifically, invoke `_twapUpdate()` before modifying targets/R-state in `flashLoan` (and in `_sync()` / `_setReserve()`, update the TWAP before writing the new reserves):

```solidity
        // sell base case
        // base input + quote output
        if (quoteBalance < _QUOTE_RESERVE_) {
            uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
            (uint256 receiveQuoteAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newBaseTarget) = querySellBase(
                tx.origin,
                baseInput
            );

            if (uint256(_QUOTE_RESERVE_) - quoteBalance > receiveQuoteAmount) {
                revert ErrFlashLoanFailed();
            }

            _twapUpdate();

            _transferQuoteOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);
            if (_RState_ != uint32(newRState)) {
                _BASE_TARGET_ = newBaseTarget.toUint112();
                _RState_ = uint32(newRState);
                emit RChange(newRState);
            }
            emit Swap(address(_BASE_TOKEN_), address(_QUOTE_TOKEN_), baseInput, receiveQuoteAmount, msg.sender, assetTo);
        }

        _sync();
```

Additionally, ensure `_sync()` and `_setReserve()` call `_twapUpdate()` prior to overwriting `_BASE_RESERVE_` and `_QUOTE_RESERVE_`.

## Finding 7: Incorrect `_twapUpdate()` Ordering in `MagicLP._setReserve` and `MagicLP._sync` Allows Trivial TWAP Manipulation

### Synopsis

In `MagicLP._setReserve` and `MagicLP._sync`, `_twapUpdate()` is invoked after updating `_BASE_RESERVE_` and `_QUOTE_RESERVE_` rather than before. This attributes the post-transaction mid-price to the entire `timeElapsed` since the previous pool update, allowing an attacker to trivially manipulate `_BASE_PRICE_CUMULATIVE_LAST_` within a single block.

### Technical details

Time-Weighted Average Price (TWAP) accumulators must weight the price that was active *before* a state-changing operation by the duration (`timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_`) for which that historical state persisted. Only after accumulating the historical price over `timeElapsed` should the pool's reserves be updated to reflect the current transaction.

In `MagicLP.sol`, `_twapUpdate()` calculates the price increment using `getMidPrice()`, which reads `_BASE_RESERVE_` and `_QUOTE_RESERVE_` via `getPMMState()`:

```solidity
function _twapUpdate() internal {
    uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
    uint32 timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;

    if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
        unchecked {
            _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
        }
    }

    _BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
}
```

However, in both `_setReserve` (lines 560–565) and `_sync` (lines 567–579), the contract updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` *before* calling `_twapUpdate()`:

```solidity
function _setReserve(uint256 baseReserve, uint256 quoteReserve) internal {
    _BASE_RESERVE_ = baseReserve.toUint112();
    _QUOTE_RESERVE_ = quoteReserve.toUint112();

    _twapUpdate();
}

function _sync() internal {
    uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
    uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));

    if (baseBalance != _BASE_RESERVE_) {
        _BASE_RESERVE_ = baseBalance.toUint112();
    }
    if (quoteBalance != _QUOTE_RESERVE_) {
        _QUOTE_RESERVE_ = quoteBalance.toUint112();
    }

    _twapUpdate();
}
```

Because `_twapUpdate()` updates `_BLOCK_TIMESTAMP_LAST_ = blockTimestamp` on the first call in a given block, only the first transaction in a block has `timeElapsed > 0`, while any subsequent transaction in the same block has `timeElapsed == 0`. Consequently, the new reserves resulting from the first transaction in a block are retroactively applied to the entire `timeElapsed` interval since the previous checkpoint, and the reserves can immediately be restored in the same transaction or block without any further change to `_BASE_PRICE_CUMULATIVE_LAST_`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized and has non-zero reserves (`_BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0`).
2. At least 1 second has elapsed since the last transaction that triggered `_twapUpdate()` on the pool (`uint32(block.timestamp % 2 ** 32) > _BLOCK_TIMESTAMP_LAST_`).
3. An external consumer or integrator relies on `_BASE_PRICE_CUMULATIVE_LAST_` to compute a TWAP for the `MagicLP` pool.

### Proof of concept

1. Suppose a `MagicLP` pool has balanced reserves (`_BASE_RESERVE_ = 100,000e18`, `_QUOTE_RESERVE_ = 100,000e18`) and `getMidPrice()` is `1e18` at timestamp $t_0 = 1000$.
2. No pool interactions occur for $1000$ seconds. At timestamp $t_1 = 2000$ (`timeElapsed = 1000`), an attacker executes a transaction that:
   - Transfers a large amount of `_BASE_TOKEN_` (or `_QUOTE_TOKEN_`) into the pool (potentially sourced via an external flash loan) and calls `sync()` (which invokes `_sync()`) or `sellBase()` (which invokes `_setReserve()`).
   - Inside `_sync()` / `_setReserve()`, `_BASE_RESERVE_` and `_QUOTE_RESERVE_` are updated to the skewed balances first.
   - `_twapUpdate()` then executes with `timeElapsed = 1000`, adding `manipulatedMidPrice * 1000` to `_BASE_PRICE_CUMULATIVE_LAST_` and setting `_BLOCK_TIMESTAMP_LAST_ = 2000`.
3. Within the same transaction (at $t_1 = 2000$), the attacker swaps back to restore the pool reserves to their original state and repays the flash loan.
4. During the reverse swap, `_setReserve()` calls `_twapUpdate()`, which sees `timeElapsed = 2000 - 2000 = 0` and leaves `_BASE_PRICE_CUMULATIVE_LAST_` unchanged.
5. Any TWAP consumer measuring the average price between $t_0$ and $t_1$ observes `manipulatedMidPrice` across the entire 1000-second window, while the attacker only paid swap fees (or zero pool fees if skewing via `sync()` and `sellShares()`).

### Impact assessment

- **Severity:** High / Medium
- **Technical Impact:** The cumulative price accumulator (`_BASE_PRICE_CUMULATIVE_LAST_`) records post-trade prices over historical time intervals instead of pre-trade prices. This completely defeats the time-weighting property of the TWAP, allowing atomic, single-transaction manipulation of the cumulative price proportional to the time elapsed since the last pool interaction.
- **Business Impact:** Any lending market, oracle, or external protocol using `_BASE_PRICE_CUMULATIVE_LAST_` to derive a TWAP can be atomically manipulated via flash loans, leading to mispriced collateral, unfair liquidations, or bad debt.

### Remediation

Invoke `_twapUpdate()` *before* updating `_BASE_RESERVE_` and `_QUOTE_RESERVE_` in `_setReserve` and `_sync` (and ensure any caller-side state variables affecting `getPMMState()`, such as `_BASE_TARGET_`, `_QUOTE_TARGET_`, and `_RState_`, are also updated only after `_twapUpdate()` has accumulated the prior state's mid-price).

Within the highlighted functions, update `_setReserve` and `_sync` as follows:

```solidity
    function _setReserve(uint256 baseReserve, uint256 quoteReserve) internal {
        _twapUpdate();

        _BASE_RESERVE_ = baseReserve.toUint112();
        _QUOTE_RESERVE_ = quoteReserve.toUint112();
    }

    function _sync() internal {
        _twapUpdate();

        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));

        if (baseBalance != _BASE_RESERVE_) {
            _BASE_RESERVE_ = baseBalance.toUint112();
        }
        if (quoteBalance != _QUOTE_RESERVE_) {
            _QUOTE_RESERVE_ = quoteBalance.toUint112();
        }
    }
```

## Finding 8: Unsynchronized Target Updates in `MagicLP._sync` and `MagicLP._setReserve` Corrupt PMM Reserve-Target Invariants

### Synopsis

`MagicLP._sync()` and `MagicLP._setReserve()` update `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to match current token balances without proportionally updating `_BASE_TARGET_` and `_QUOTE_TARGET_`. Any asynchronous balance increase or donation synced via `sync()`, `flashLoan()`, `sellShares()`, or swaps causes reserve-target divergence, corrupts PMM pricing, and neutralizes `ratioSync()`.

### Technical details

`MagicLP` implements DODO V2's Proactive Market Maker (PMM) algorithm via `PMMPricing`. The PMM state (`PMMState`) relies on strict invariants between the current reserves (`B = _BASE_RESERVE_`, `Q = _QUOTE_RESERVE_`), the target reserves at equilibrium (`B0 = _BASE_TARGET_`, `Q0 = _QUOTE_TARGET_`), and the pool state (`R = _RState_`):

1. When `_RState_ == RState.ONE`, the pool must satisfy `B == B0` and `Q == Q0`. `PMMPricing.adjustedTarget()` performs no target recalculation in this state, and `_ROneSellBaseToken` / `_ROneSellQuoteToken` price trades directly from `Q0` and `B0` while `getMidPrice()` evaluates `(B0 * B0) / (B * B)`.
2. When `_RState_ == RState.BELOW_ONE`, `PMMPricing.adjustedTarget()` dynamically computes `state.Q0` from `state.Q` and the excess base reserve `state.B - state.B0`, and `sellQuoteToken()` sets `backToOneReceiveBase = state.B - state.B0`.
3. When `_RState_ == RState.ABOVE_ONE`, `PMMPricing.adjustedTarget()` dynamically computes `state.B0` from `state.B` and the excess quote reserve `state.Q - state.Q0`, and `sellBaseToken()` sets `backToOneReceiveQuote = state.Q - state.Q0`.

To preserve these invariants when token balances change outside of swaps, `MagicLP.ratioSync()` (lines 504–522) scales `_BASE_TARGET_` and `_QUOTE_TARGET_` proportionally (`_BASE_TARGET_ * baseBalance / _BASE_RESERVE_`) before updating `_BASE_RESERVE_` and `_QUOTE_RESERVE_`.

However, `_sync()` (lines 567–579)—which is exposed permissionlessly via `sync()` (line 134) and invoked in `flashLoan()` (line 350) and `sellShares()` (line 448)—updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` directly to `_BASE_TOKEN_.balanceOf(address(this))` and `_QUOTE_TOKEN_.balanceOf(address(this))` without updating `_BASE_TARGET_` or `_QUOTE_TARGET_`. Likewise, `_setReserve()` (lines 560–565) in `sellBase()` (line 262) and `sellQuote()` (line 285) overwrites the output token reserve with its raw `balanceOf(address(this))` snapshot without adjusting targets for any uncredited balance delta.

Consequently:
- If `_sync()` is called when `baseBalance != _BASE_RESERVE_` or `quoteBalance != _QUOTE_RESERVE_`, `_BASE_RESERVE_` and `_QUOTE_RESERVE_` diverge from `_BASE_TARGET_` and `_QUOTE_TARGET_`.
- When `_RState_ == RState.ONE`, a synced increase in `_BASE_RESERVE_` causes `B > B0`. A subsequent dust `sellBase()` call transitions `_RState_` to `BELOW_ONE` while recording the stale `_BASE_TARGET_ = B0`. `getPMMState()` then treats the entire `B - B0` difference as base tokens sold along the curve, inflating `state.Q0` in `adjustedTarget()` and allowing `sellQuote()` to extract `state.B - state.B0` at discounted `BELOW_ONE` curve pricing or lock in the inflated `state.Q0` as the new `_QUOTE_TARGET_` upon crossing `RState.ONE`.
- Once `_sync()` or `_setReserve()` sets `_BASE_RESERVE_ = baseBalance` and `_QUOTE_RESERVE_ = quoteBalance`, `ratioSync()` is rendered ineffective because its `baseBalance != _BASE_RESERVE_` and `quoteBalance != _QUOTE_RESERVE_` checks evaluate to `false`.

### Assumptions and preconditions

1. An initialized `MagicLP` pool has active liquidity (`_BASE_RESERVE_ > 0`, `_QUOTE_RESERVE_ > 0`).
2. The pool's `_BASE_TOKEN_` or `_QUOTE_TOKEN_` balance increases outside of standard swap input accounting (e.g., via direct token transfer/donation, unconfigured rebasing yield, or excess flash loan repayment), or an attacker intentionally donates/transfers tokens to manipulate target calculations.

### Proof of concept

1. Assume a `MagicLP` pool is initialized with `i = 1e18`, `K = 5e14`, and liquidity of `100,000e18` base tokens and `100,000e18` quote tokens:
   - `_BASE_RESERVE_ = 100,000e18`, `_BASE_TARGET_ = 100,000e18`
   - `_QUOTE_RESERVE_ = 100,000e18`, `_QUOTE_TARGET_ = 100,000e18`
   - `_RState_ = RState.ONE`
2. Suppose `10,000e18` base tokens are accrued or transferred directly to the pool (so `_BASE_TOKEN_.balanceOf(address(this)) == 110,000e18`), intended to be synced via `ratioSync()`.
3. Before the owner calls `ratioSync()`, an arbitrary caller invokes `sync()` (or triggers `_sync()` via `flashLoan()` / `sellShares()`):
   - `_sync()` sets `_BASE_RESERVE_ = 110,000e18`.
   - `_BASE_TARGET_` remains `100,000e18` and `_RState_` remains `RState.ONE`.
4. If the owner now calls `ratioSync()`, `baseBalance != _BASE_RESERVE_` (`110,000e18 != 110,000e18`) is `false`, so `_BASE_TARGET_` is never scaled to `110,000e18`.
5. A caller transfers `1 wei` of base token to the pool and calls `sellBase()`:
   - `querySellBase()` runs with `state.R == RState.ONE` and `state.B0 == 100,000e18`.
   - `sellBase()` transitions `_RState_` to `RState.BELOW_ONE`, sets `_BASE_TARGET_ = 100,000e18`, and updates `_BASE_RESERVE_ = 110,000e18 + 1`.
6. On the next call to `getPMMState()`, `PMMPricing.adjustedTarget()` sees `state.R == RState.BELOW_ONE` and `state.B - state.B0 = 10,000e18 + 1`, computing `state.Q0 ≈ 110,000e18` even though only `100,000e18` quote tokens exist in the pool. If a trader then swaps quote tokens via `sellQuote()` past `backToOnePayQuote`, `_QUOTE_TARGET_` is permanently overwritten with `~110,000e18` while `_BASE_TARGET_` remains `100,000e18`, permanently skewing the pool's PMM targets and mid-price.

### Impact assessment

- **Severity**: Medium / High.
- **Technical Impact**: Reserve updates in `_sync()` and `_setReserve()` desynchronize `_BASE_RESERVE_` and `_QUOTE_RESERVE_` from `_BASE_TARGET_` and `_QUOTE_TARGET_`, corrupting `getPMMState()`, `getMidPrice()`, TWAP updates, and swap target transitions while permanently preventing `ratioSync()` from scaling targets.
- **Business Impact**: Unaccounted yield or token donations fail to accrue proportionally to liquidity providers and instead distort pool pricing and oracle mid-prices (affecting downstream consumers such as `MagicLpAggregator`) or can be arbitraged away by traders.

### Remediation

Scale `_BASE_TARGET_` and `_QUOTE_TARGET_` proportionally whenever `_sync()` synchronizes unexpected balance changes (matching `ratioSync()`), or track reserves strictly via internal accounting deltas rather than raw `balanceOf(address(this))` snapshots in `_setReserve()` and `_sync()`.

```solidity
    function _setReserve(uint256 baseReserve, uint256 quoteReserve) internal {
        _BASE_RESERVE_ = baseReserve.toUint112();
        _QUOTE_RESERVE_ = quoteReserve.toUint112();

        _twapUpdate();
    }

    function _sync() internal {
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));

        if (baseBalance != _BASE_RESERVE_) {
            if (_BASE_RESERVE_ > 0) {
                _BASE_TARGET_ = ((uint256(_BASE_TARGET_) * baseBalance) / uint256(_BASE_RESERVE_)).toUint112();
            }
            _BASE_RESERVE_ = baseBalance.toUint112();
        }
        if (quoteBalance != _QUOTE_RESERVE_) {
            if (_QUOTE_RESERVE_ > 0) {
                _QUOTE_TARGET_ = ((uint256(_QUOTE_TARGET_) * quoteBalance) / uint256(_QUOTE_RESERVE_)).toUint112();
            }
            _QUOTE_RESERVE_ = quoteBalance.toUint112();
        }

        _twapUpdate();
    }
```

## Finding 9: Missing Return Statement in `MagicLpAggregator._getReserves` Causes `latestAnswer` to Always Return Zero

### Synopsis

In `MagicLpAggregator._getReserves()`, the reserve values retrieved from `pair.getReserves()` are assigned to local variables without being returned, and the return parameters in the function signature are unnamed. Consequently, `_getReserves()` always returns `(0, 0)`, causing `latestAnswer()` and `latestRoundData()` to always return a price of `0`.

### Technical details

The `MagicLpAggregator` contract computes the price of a `MagicLP` token in `latestAnswer()` (lines 37–46) by querying the underlying pool's reserves via the internal helper `_getReserves()` on line 42:

```solidity
function _getReserves() internal view virtual returns (uint256, uint256) {
    (uint256 baseReserve, uint256 quoteReserve) = pair.getReserves();
}
```

In `_getReserves()` (lines 33–35):
1. The return types in the function header `returns (uint256, uint256)` are unnamed, so Solidity initializes two unnamed return variables on the stack to `(0, 0)`.
2. On line 34, `pair.getReserves()` is called and its return values are captured into newly declared local variables `baseReserve` and `quoteReserve`.
3. There is no `return` statement in the function body.

When `_getReserves()` finishes execution, the local variables `baseReserve` and `quoteReserve` are discarded, and the default-initialized unnamed return values `(0, 0)` are returned to the caller.

In `latestAnswer()` (lines 42–45):
```solidity
(uint256 baseReserve, uint256 quoteReserve) = _getReserves();
baseReserve = baseReserve * (10 ** (WAD - baseDecimals));
quoteReserve = quoteReserve * (10 ** (WAD - quoteDecimals));
return int256(minAnswer * (baseReserve + quoteReserve) / pair.totalSupply());
```
Because `baseReserve` and `quoteReserve` are both `0`, `baseReserve + quoteReserve` evaluates to `0`, and `latestAnswer()` unconditionally returns `0` (as long as `pair.totalSupply() > 0`). Notice that this bug was masked in the test suite (`test/MagicLpAggregator.t.sol`) because the test contract `MagicLpAggregatorExt` overrides `_getReserves()` with `return IDodo(address(pair)).getVaultReserve();`.

### Assumptions and preconditions

- `MagicLpAggregator` is deployed and used directly (without overriding `_getReserves()`) to price `MagicLP` tokens.
- `pair.totalSupply()` is non-zero.

### Proof of concept

1. Deploy a `MagicLP` pool with non-zero base and quote reserves (e.g., `1000e18` MIM and `1000e18` USDB) and non-zero `totalSupply()`.
2. Deploy `MagicLpAggregator` pointing to the `MagicLP` pool and valid `baseOracle` and `quoteOracle` feeds (each returning `1e8` with `8` decimals).
3. Call `MagicLpAggregator.latestAnswer()`.
4. Inside `latestAnswer()`, `minAnswer` evaluates to `1e18`.
5. `latestAnswer()` calls `_getReserves()`, which fetches `(1000e18, 1000e18)` into local variables `baseReserve` and `quoteReserve` on line 34, does not return them, and exits returning `(0, 0)`.
6. `latestAnswer()` computes `1e18 * (0 + 0) / pair.totalSupply()` and returns `0` instead of the expected LP token price (`~2e18`).

### Impact assessment

- **Severity:** High
- **Technical Impact:** `MagicLpAggregator.latestAnswer()` and `MagicLpAggregator.latestRoundData()` always return `0` for any `MagicLP` pool.
- **Business Impact:** `MagicLpAggregator` is completely non-functional for pricing `MagicLP` collateral in Cauldrons. Any Cauldron or downstream consumer relying on this aggregator will value the LP collateral at zero, breaking borrowing workflows or causing immediate, unwarranted liquidations of solvent user positions.

### Remediation

Return the result of `pair.getReserves()` directly in `_getReserves()` (or use named return variables).

```solidity
    function _getReserves() internal view virtual returns (uint256, uint256) {
        return pair.getReserves();
    }
```

## Finding 10: Unvalidated Chainlink Oracle Reads in `MagicLpAggregator.latestAnswer` Allow Stale or Non-Positive Prices

### Synopsis

`MagicLpAggregator.latestAnswer` queries `baseOracle.latestAnswer()` and `quoteOracle.latestAnswer()` directly, casting signed `int256` answers to `uint256` without checking for non-positive values (`<= 0`) or verifying feed freshness via `latestRoundData()`. Stale, zero, or negative oracle responses consequently corrupt the LP token valuation.

### Technical details

In `src/oracles/aggregators/MagicLpAggregator.sol`, `latestAnswer()` computes the USD value of a `MagicLP` token using external Chainlink-compatible price feeds (`baseOracle` and `quoteOracle`, typed as `IAggregator`):

```solidity
uint256 baseAnswerNomalized = uint256(baseOracle.latestAnswer()) * (10 ** (WAD - baseOracle.decimals()));
uint256 quoteAnswerNormalized = uint256(quoteOracle.latestAnswer()) * (10 ** (WAD - quoteOracle.decimals()));
uint256 minAnswer = baseAnswerNomalized < quoteAnswerNormalized ? baseAnswerNomalized : quoteAnswerNormalized;
```

Although `IAggregator` exposes `latestRoundData()`, `MagicLpAggregator.latestAnswer()` relies on the deprecated `latestAnswer()` interface method and performs no input integrity checks on the returned values:

1. **Missing Staleness Validation:** Because `latestAnswer()` does not return round metadata (`updatedAt`, `roundId`, `answeredInRound`), the contract cannot verify when the price was last updated. If either underlying Chainlink aggregator stalls, loses heartbeat updates, or is deprecated, `MagicLpAggregator` continues serving arbitrarily stale prices as current.
2. **Invalid Price-Domain Handling:** `latestAnswer()` returns a signed `int256`, which is immediately cast to `uint256` on lines 38 and 39.
   - If an oracle returns `0` (for example, during an uninitialized round or feed outage), `minAnswer` becomes `0`, causing `latestAnswer()` to return `0` and rendering all LP collateral valueless.
   - If an oracle returns a negative value, `uint256(answer)` underflows modulo $2^{256}$ into a massive positive integer (which can also cause arithmetic overflow when multiplied by `10 ** (WAD - decimals)`).

### Assumptions and preconditions

1. `MagicLpAggregator` is deployed and used by downstream lending markets (Cauldrons) to value `MagicLP` collateral tokens.
2. Either `baseOracle` or `quoteOracle` experiences delayed heartbeat updates (returning a stale price) or reports a non-positive value (`<= 0`) due to a feed malfunction or circuit breaker event.

### Proof of concept

1. `MagicLpAggregator` is configured with `baseOracle` (e.g., DAI/USD) and `quoteOracle` (e.g., USDT/USD) for a stable `MagicLP` pool holding `1,000e18` base tokens and `1,000e6` quote tokens with `2,000e18` total supply.
2. Under normal operation, both feeds return `1e8` (with 8 decimals), normalizing to `1e18`. `latestAnswer()` returns `1e18 * (1,000e18 + 1,000e18) / 2,000e18 = 1e18`.
3. **Stale Feed Scenario:** `quoteOracle` stops updating while the quote token depegs to $0.70 on the market. Because `latestAnswer()` checks neither `updatedAt` nor `answeredInRound`, `MagicLpAggregator.latestAnswer()` continues using the stale $1.00 price, overvaluing the LP token collateral and allowing borrowers to borrow excess funds or avoid liquidation.
4. **Zero/Malformed Price Scenario:** `quoteOracle` experiences a feed failure and returns `0`. Line 39 computes `quoteAnswerNormalized = 0`, line 40 sets `minAnswer = 0`, and line 45 returns `0`, allowing healthy borrower positions using this LP token as collateral to be unfairly liquidated.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** `MagicLpAggregator.latestAnswer()` can return stale prices or corrupt valuations (`0` or wrapped `uint256` values) without reverting.
- **Business Impact:** Downstream Cauldrons relying on `MagicLpAggregator` to price collateral can suffer bad debt from overvalued collateral during token depegs when a feed is stale, or wrongful liquidations of solvent users if an oracle reports `0`.

### Remediation

Replace `latestAnswer()` calls with `latestRoundData()` on both `baseOracle` and `quoteOracle`, and validate that the returned prices are strictly positive (`> 0`) and not stale (`block.timestamp - updatedAt <= maxStalePeriod`):

```solidity
    function latestAnswer() public view override returns (int256) {
        (, int256 baseAnswer, , uint256 baseUpdatedAt, ) = baseOracle.latestRoundData();
        (, int256 quoteAnswer, , uint256 quoteUpdatedAt, ) = quoteOracle.latestRoundData();
        require(baseAnswer > 0 && quoteAnswer > 0, "Invalid oracle answer");
        require(
            block.timestamp - baseUpdatedAt <= MAX_STALE_PERIOD &&
            block.timestamp - quoteUpdatedAt <= MAX_STALE_PERIOD,
            "Stale oracle price"
        );

        uint256 baseAnswerNomalized = uint256(baseAnswer) * (10 ** (WAD - baseOracle.decimals()));
        uint256 quoteAnswerNormalized = uint256(quoteAnswer) * (10 ** (WAD - quoteOracle.decimals()));
        uint256 minAnswer = baseAnswerNomalized < quoteAnswerNormalized ? baseAnswerNomalized : quoteAnswerNormalized;

        (uint256 baseReserve, uint256 quoteReserve) = _getReserves();
        baseReserve = baseReserve * (10 ** (WAD - baseDecimals));
        quoteReserve = quoteReserve * (10 ** (WAD - quoteDecimals));
        return int256(minAnswer * (baseReserve + quoteReserve) / pair.totalSupply());
    }
```

## Finding 11: Spot Reserve Manipulation in `MagicLpAggregator.latestAnswer` Allows Flash-Loan Inflation of LP Token Price

### Synopsis

`MagicLpAggregator.latestAnswer()` values MagicLP tokens by summing the pool's spot reserves (`baseReserve + quoteReserve`) multiplied by `minAnswer` and dividing by `pair.totalSupply()`. Because PMM swap curves require increasingly more input tokens for diminishing output as reserves become imbalanced, an attacker can use a flash loan to skew the pool reserves, drastically inflating `baseReserve + quoteReserve` without changing `totalSupply()`.

### Technical details

In `src/oracles/aggregators/MagicLpAggregator.sol`, `latestAnswer()` computes the price of one LP share as:

```solidity
(uint256 baseReserve, uint256 quoteReserve) = _getReserves();
baseReserve = baseReserve * (10 ** (WAD - baseDecimals));
quoteReserve = quoteReserve * (10 ** (WAD - quoteDecimals));
return int256(minAnswer * (baseReserve + quoteReserve) / pair.totalSupply());
```

This valuation formula assumes that the sum of the normalized spot reserves `baseReserve + quoteReserve` is invariant across swaps for a given `totalSupply()`. However, `MagicLP` implements the DODO V2 Proactive Market Maker (PMM) bonding curve (`PMMPricing.sol`), where the marginal exchange rate diverges from the oracle price $i$ as reserves deviate from the target reserves $(B_0, Q_0)$ according to the slippage coefficient $K$:

$$P_{\text{margin}} = i \left(1 - K + K \left(\frac{Q_0}{Q}\right)^2\right) \quad \text{or} \quad \frac{i}{1 - K + K \left(\frac{B_0}{B}\right)^2}$$

When a user executes a large swap via `MagicLP.sellBase()` or `MagicLP.sellQuote()`, the input reserve increases by $\Delta_{\text{in}}$ while the output reserve decreases by $\Delta_{\text{out}}$, which is strictly bounded by the available reserve in the pool. Because the PMM curve becomes progressively steeper as the output reserve is depleted, $\Delta_{\text{in}} \gg \Delta_{\text{out}}$ for large trades. Consequently:

1. The sum of spot reserves $(B + \Delta_{\text{in}}) + (Q - \Delta_{\text{out}}) = B + Q + (\Delta_{\text{in}} - \Delta_{\text{out}})$ increases substantially—and can be inflated arbitrarily high by pushing the pool far along the bonding curve.
2. `pair.totalSupply()` remains completely unchanged during swaps.
3. `minAnswer` (derived from external Chainlink oracles `baseOracle` and `quoteOracle`) remains unchanged.

As a result, `latestAnswer()` reports an artificially inflated LP token valuation within a single transaction. After exploiting a downstream lending market (such as an Abracadabra Cauldron) using the inflated collateral valuation, the attacker can swap back in the opposite direction to restore the pool reserves and repay the flash loan, paying only the pool's swap fee.

### Assumptions and preconditions

1. A `MagicLP` pool is configured with a non-zero slippage parameter `_K_ > 0` (standard for MIMSwap stable and pegged pools, as seen in `MIMSwap.t.sol`).
2. `MagicLpAggregator` is used by a lending market (e.g., Cauldron) to value `MagicLP` collateral tokens.
3. The attacker has access to capital (e.g., via flash loans) in either the base or quote token of the `MagicLP` pair.

### Proof of concept

1. Consider a `MagicLP` pool for two 18-decimal stablecoins (both priced at `$1`, so `minAnswer = 1e18`) with initial balanced reserves `B = B0 = 1,000,000e18`, `Q = Q0 = 1,000,000e18`, and `totalSupply = 1,000,000e18`.
2. Initially, `MagicLpAggregator.latestAnswer()` returns:
   `1e18 * (1,000,000e18 + 1,000,000e18) / 1,000,000e18 = 2e18` ($2.00 per LP token).
3. An attacker flash-loans a large amount of `baseToken` (e.g., `10,000,000e18`) and swaps it into the pool via `pair.sellBase()`.
4. Due to the PMM curve (`K > 0`), `quoteReserve` cannot drop below `0` (it decreases by at most `< 1,000,000e18`), while `baseReserve` increases by the full `10,000,000e18` to `11,000,000e18`.
5. While the pool is in this imbalanced state, `baseReserve + quoteReserve > 11,000,000e18` and `totalSupply` is still `1,000,000e18`.
6. The attacker deposits a small amount of `MagicLP` tokens as collateral into a Cauldron and borrows MIM. The Cauldron queries `MagicLpAggregator.latestAnswer()`, which now returns `> 11e18` (> $11.00 per LP token, an inflation of over 5.5x).
7. The attacker borrows MIM far exceeding the true value of their LP collateral, swaps the `quoteToken` back for `baseToken` via `pair.sellQuote()` to restore the pool to its original state (incurring only the LP swap fee), repays the flash loan, and keeps the excess borrowed MIM as profit, leaving the protocol with bad debt.

### Impact assessment

- **Severity:** High
- **Technical Impact:** The oracle price returned by `MagicLpAggregator.latestAnswer()` can be atomically inflated within a single transaction to an arbitrarily high multiple of its fair value.
- **Business Impact:** Any Cauldron or lending market relying on `MagicLpAggregator` to price collateral can be drained of its borrowable MIM liquidity via undercollateralized loans, resulting in direct, unrecoverable bad debt for the protocol.

### Remediation

Instead of valuing the LP token using raw spot reserves (`baseReserve + quoteReserve`), compute the equilibrium/target reserves `(B0, Q0)` from the PMM state (via `getPMMState()`, which solves the quadratic PMM invariant in `PMMPricing.adjustedTarget(state)`), or price the LP token based on the fair equilibrium reserves at the oracle price ratio so that moving along the bonding curve cannot inflate the reserve sum.

Specifically, in `MagicLpAggregator.sol`, query the adjusted target reserves `(B0, Q0)` when `RState == ONE` or solve for the equilibrium reserves at the oracle price, or use `getPMMState()` target values:

```solidity
    function latestAnswer() public view override returns (int256) {
        uint256 baseAnswerNormalized = uint256(baseOracle.latestAnswer()) * (10 ** (WAD - baseOracle.decimals()));
        uint256 quoteAnswerNormalized = uint256(quoteOracle.latestAnswer()) * (10 ** (WAD - quoteOracle.decimals()));
        uint256 minAnswer = baseAnswerNormalized < quoteAnswerNormalized ? baseAnswerNormalized : quoteAnswerNormalized;

        PMMPricing.PMMState memory state = IMagicLP(address(pair)).getPMMState();
        uint256 baseTarget = state.B0 * (10 ** (WAD - baseDecimals));
        uint256 quoteTarget = state.Q0 * (10 ** (WAD - quoteDecimals));
        return int256(minAnswer * (baseTarget + quoteTarget) / pair.totalSupply());
    }
```

*(Note: Ensure that `MagicLP` also enforces `nonReadReentrant` or updates targets before external calls so `getPMMState()` cannot be read in an inconsistent mid-execution state.)*

## Finding 12: Lack of `totalSupply` decimal normalization in `MagicLpAggregator.latestAnswer` inflates LP price when `baseDecimals < 18`

### Synopsis

`MagicLpAggregator.latestAnswer()` normalizes `baseReserve`, `quoteReserve`, and the oracle answers to 18 decimals (`WAD`), but divides by `pair.totalSupply()` without normalizing it from `baseDecimals` to 18 decimals. Because `MagicLP` mints shares in `baseDecimals`, the returned price is inflated by `10 ** (18 - baseDecimals)` whenever `baseDecimals < 18`.

### Technical details

In `src/oracles/aggregators/MagicLpAggregator.sol`, `MagicLpAggregator` implements `IAggregator` with `decimals()` returning `18` (`WAD`). In `latestAnswer()` (lines 38–45), the contract normalizes both oracle prices and both token reserves to 18 decimals (`WAD`):

```solidity
uint256 baseAnswerNomalized = uint256(baseOracle.latestAnswer()) * (10 ** (WAD - baseOracle.decimals()));
uint256 quoteAnswerNormalized = uint256(quoteOracle.latestAnswer()) * (10 ** (WAD - quoteOracle.decimals()));
uint256 minAnswer = baseAnswerNomalized < quoteAnswerNormalized ? baseAnswerNomalized : quoteAnswerNormalized;

(uint256 baseReserve, uint256 quoteReserve) = _getReserves();
baseReserve = baseReserve * (10 ** (WAD - baseDecimals));
quoteReserve = quoteReserve * (10 ** (WAD - quoteDecimals));
return int256(minAnswer * (baseReserve + quoteReserve) / pair.totalSupply());
```

After lines 38–44:
- `minAnswer` is scaled to `10 ** 18` (`WAD`).
- `baseReserve` and `quoteReserve` are each scaled to `10 ** 18` (`WAD`).
- The numerator `minAnswer * (baseReserve + quoteReserve)` is therefore scaled to `10 ** 36` (`10 ** (2 * WAD)`).

For the quotient on line 45 to have 18 decimals (`10 ** WAD`), the denominator `pair.totalSupply()` must also be scaled to `10 ** 18`.

However, in `MagicLP` (and DODO V2 `DODOStablePool`), LP shares are denominated in `baseDecimals` (the decimals of `_BASE_TOKEN_`). Specifically, during initial liquidity provision in `MagicLP.buyShares()`, minted shares are calculated as:
`shares = quoteInput < DecimalMath.mulFloor(baseInput, _I_) ? DecimalMath.divFloor(quoteInput, _I_) : baseInput;`
Since `_I_` is scaled to `10 ** (18 - baseDecimals + quoteDecimals)`, both `DecimalMath.divFloor(quoteInput, _I_)` and `baseInput` have `baseDecimals` decimals, and all subsequent share mints are scaled proportionally to `totalSupply()`.

Consequently, `pair.totalSupply()` has `baseDecimals` decimals rather than 18 decimals. Dividing the `10 ** 36` numerator by an unnormalized `pair.totalSupply()` produces an answer with `36 - baseDecimals` decimals instead of `18` decimals, overstating the price by a factor of `10 ** (18 - baseDecimals)`.

### Assumptions and preconditions

1. A `MagicLP` (or DODO V2) pool is deployed where `_BASE_TOKEN_` has fewer than 18 decimals (for example, USDC or USDT with 6 decimals as `_BASE_TOKEN_` and MIM with 18 decimals as `_QUOTE_TOKEN_`).
2. `MagicLpAggregator` is deployed for this pair to report the 18-decimal USD price per LP share.

### Proof of concept

Consider a `MagicLP` pool where `_BASE_TOKEN_` is USDT (`baseDecimals = 6`) and `_QUOTE_TOKEN_` is MIM (`quoteDecimals = 18`), with both tokens priced at $1 (`1e8` on 8-decimal Chainlink feeds) and `_I_ = 1e30` (`10 ** (18 - 6 + 18)`):

1. A liquidity provider deposits `1,000e6` USDT and `1,000e18` MIM into the pool.
2. `MagicLP.buyShares()` mints `shares = baseInput = 1,000e6`, so `pair.totalSupply() = 1,000e6`.
3. When `MagicLpAggregator.latestAnswer()` is called:
   - `baseAnswerNomalized = 1e8 * 10 ** (18 - 8) = 1e18`
   - `quoteAnswerNormalized = 1e8 * 10 ** (18 - 8) = 1e18`
   - `minAnswer = 1e18`
   - `baseReserve = 1,000e6 * 10 ** (18 - 6) = 1,000e18`
   - `quoteReserve = 1,000e18 * 10 ** (18 - 18) = 1,000e18`
4. Line 45 computes:
   `minAnswer * (baseReserve + quoteReserve) / pair.totalSupply() = (1e18 * 2,000e18) / 1,000e6 = 2e30`.
5. Instead of returning `2e18` ($2.00 in 18 decimals), `latestAnswer()` returns `2e30` ($2,000,000,000,000.00 in 18 decimals), which is inflated by `10 ** 12` (`10 ** (18 - 6)`).

### Impact assessment

**Severity: High**

Any protocol component or Cauldron relying on `MagicLpAggregator.latestAnswer()` to return an 18-decimal price for a pool whose base token has fewer than 18 decimals will receive a price inflated by `10 ** (18 - baseDecimals)`. In lending markets (Cauldrons) or external integrations that price LP shares using `MagicLpAggregator`, this massive overvaluation allows borrowers to borrow far more than the true value of their collateral, draining lending pool funds.

### Remediation

Normalize `pair.totalSupply()` from `baseDecimals` to `WAD` (18 decimals) before dividing on line 45.

```solidity
    function latestAnswer() public view override returns (int256) {
        uint256 baseAnswerNomalized = uint256(baseOracle.latestAnswer()) * (10 ** (WAD - baseOracle.decimals()));
        uint256 quoteAnswerNormalized = uint256(quoteOracle.latestAnswer()) * (10 ** (WAD - quoteOracle.decimals()));
        uint256 minAnswer = baseAnswerNomalized < quoteAnswerNormalized ? baseAnswerNomalized : quoteAnswerNormalized;

        (uint256 baseReserve, uint256 quoteReserve) = _getReserves();
        baseReserve = baseReserve * (10 ** (WAD - baseDecimals));
        quoteReserve = quoteReserve * (10 ** (WAD - quoteDecimals));
        uint256 totalSupply = pair.totalSupply() * (10 ** (WAD - baseDecimals));
        return int256(minAnswer * (baseReserve + quoteReserve) / totalSupply);
    }
```
