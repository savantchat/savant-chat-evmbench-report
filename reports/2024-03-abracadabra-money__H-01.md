# Audit report — 2024-03-abracadabra-money/H-01

## Finding 1: Revert on Intermediate Overflow in `MagicLP.getPMMState` via `PMMPricing.adjustedTarget` and `Math._SolveQuadraticFunctionForTarget`

### Synopsis

`MagicLP.getPMMState` (and `MagicLP.getPMMStateForCall`) invokes `PMMPricing.adjustedTarget`, which delegates quadratic target reconstruction to `Math._SolveQuadraticFunctionForTarget`. Because the legacy Solidity 0.6 overflow check `(ki * delta) / ki == delta` was ported to Solidity 0.8 without an `unchecked` block, intermediate multiplication overflow reverts instead of taking the fallback branch, bricking pool pricing and state updates.

### Technical details

In `MagicLP.getPMMState` (`src/mimswap/MagicLP.sol:193-202`), the pool loads the current reserves (`_BASE_RESERVE_`, `_QUOTE_RESERVE_`), targets (`_BASE_TARGET_`, `_QUOTE_TARGET_`), and `R` state (`_RState_`), and then calls `PMMPricing.adjustedTarget(state)` at line 201 (`getPMMStateForCall` at lines 204-213 calls `getPMMState()`):

```solidity
state.B = _BASE_RESERVE_;
state.Q = _QUOTE_RESERVE_;
state.B0 = _BASE_TARGET_; // will be calculated in adjustedTarget
state.Q0 = _QUOTE_TARGET_;
state.R = PMMPricing.RState(_RState_);
PMMPricing.adjustedTarget(state);
```

`PMMPricing.adjustedTarget` (`src/mimswap/libraries/PMMPricing.sol:190-201`) recomputes the dynamic target reserve `Q0` (when `R == RState.BELOW_ONE`) or `B0` (when `R == RState.ABOVE_ONE`) by calling `Math._SolveQuadraticFunctionForTarget`:

```solidity
if (state.R == RState.BELOW_ONE) {
    state.Q0 = Math._SolveQuadraticFunctionForTarget(state.Q, state.B - state.B0, state.i, state.K);
} else if (state.R == RState.ABOVE_ONE) {
    state.B0 = Math._SolveQuadraticFunctionForTarget(
        state.B,
        state.Q - state.Q0,
        DecimalMath.reciprocalFloor(state.i),
        state.K
    );
}
```

In `Math._SolveQuadraticFunctionForTarget` (`src/mimswap/libraries/Math.sol:79-104`), lines 93-100 compute `_sqrt`:

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

In the original DODO V2 codebase (written for Solidity 0.6.9), `(ki * delta) / ki == delta` relied on wrapping `uint256` arithmetic to detect whether `ki * delta` overflowed `2^256 - 1`; if it overflowed, execution safely entered the `else` branch and evaluated `((ki / V1) * delta) + DecimalMath.ONE2` by dividing `ki` by `V1` first.

When ported to Solidity `>=0.8.0`, `ki * delta` at line 96 is evaluated using default checked arithmetic. Consequently:
1. Whenever `ki * delta > type(uint256).max`, the condition `(ki * delta) / ki == delta` immediately reverts with `Panic(0x11)` (arithmetic overflow) rather than evaluating to `false`.
2. The fallback `else` branch at lines 98-100 is dead code and can never be reached.

Because `k <= 10^18`, the effective price `i` (or `DecimalMath.reciprocalFloor(state.i) = 10^36 / state.i` when `R == ABOVE_ONE`) can be up to `10^36`, and `delta` (`state.B - state.B0` or `state.Q - state.Q0`) can be up to `type(uint112).max` (`~5.19 * 10^33`), the product `ki * delta = 4 * k * i * delta` can reach `~2.07 * 10^88`, far exceeding `type(uint256).max` (`~1.1579 * 10^77`).

### Assumptions and preconditions

1. A `MagicLP` clone is initialized with valid parameters (`_K_ > 0` and `_I_` within allowed bounds, e.g., a high `_I_` when `R == BELOW_ONE` or a low `_I_` such that `DecimalMath.reciprocalFloor(_I_)` is large when `R == ABOVE_ONE`).
2. The pool enters `RState.BELOW_ONE` (with `delta = _BASE_RESERVE_ - _BASE_TARGET_`) or `RState.ABOVE_ONE` (with `delta = _QUOTE_RESERVE_ - _QUOTE_TARGET_`) with a sufficiently large `delta` such that `4 * k * i * delta > type(uint256).max`.

### Proof of concept

1. Consider a `MagicLP` pool configured with `k = 10^18` (`_K_ = 1e18`) and `i = 10^6` (`_I_ = 1e6`, e.g., pricing a 6-decimal quote token against an 18-decimal base token, or vice versa).
2. When the pool is in `RState.ABOVE_ONE`, `PMMPricing.adjustedTarget` computes the effective price as:
   `DecimalMath.reciprocalFloor(state.i) = 10^36 / 10^6 = 10^30`.
3. In `Math._SolveQuadraticFunctionForTarget`:
   `ki = 4 * k * i = 4 * 10^18 * 10^30 = 4 * 10^48`.
4. A swap or flash loan pushes `delta = state.Q - state.Q0` above `2.9 * 10^28` (well below `type(uint112).max ≈ 5.19 * 10^33`).
5. On any subsequent call to `getPMMState()`, `getPMMStateForCall()`, `getMidPrice()`, `querySellBase()`, `querySellQuote()`, `sellBase()`, `sellQuote()`, or `_twapUpdate()` (in the next block when `timeElapsed > 0`), execution reaches `Math._SolveQuadraticFunctionForTarget`:
   `ki * delta = 4 * 10^48 * 3 * 10^28 = 1.2 * 10^77 > type(uint256).max`.
6. Line 96 `(ki * delta) / ki == delta` reverts with `Panic(0x11)` instead of falling through to `else { _sqrt = sqrt(((ki / V1) * delta) + DecimalMath.ONE2); }`.

### Impact assessment

- **Severity**: Medium / High (Denial of Service of pool operations and price queries).
- **Technical Impact**: `getPMMState()`, `getPMMStateForCall()`, `getMidPrice()`, `querySellBase()`, and `querySellQuote()` revert. Because `_setReserve()` calls `_twapUpdate()`, which calls `getMidPrice()` whenever `timeElapsed > 0`, all core pool operations (`sellBase`, `sellQuote`, `flashLoan`, and liquidity operations that update reserves) also revert in subsequent blocks, locking the pool state.
- **Business Impact**: Trading, flash loans, TWAP updates, and external integrations relying on `getPMMState` / `getMidPrice` become unavailable when `ki * delta` exceeds `type(uint256).max`.

### Remediation

Wrap the overflow-detection multiplication in an `unchecked` block (or use `uint256 prod; unchecked { prod = ki * delta; }`) in `Math._SolveQuadraticFunctionForTarget` so that overflow is properly detected and routed to the fallback division-first branch:

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
            uint256 prod;
            unchecked {
                prod = ki * delta;
            }
            if (prod / ki == delta) {
                _sqrt = sqrt((prod / V1) + DecimalMath.ONE2);
            } else {
                _sqrt = sqrt(((ki / V1) * delta) + DecimalMath.ONE2);
            }
        }
        uint256 premium = DecimalMath.divFloor(_sqrt - DecimalMath.ONE, k * 2) + DecimalMath.ONE;
        // V0 is greater than or equal to V1 according to the solution
        return DecimalMath.mulFloor(V1, premium);
    }
```

## Finding 2: Target and Reserve Desynchronization in `MagicLP.getPMMState` Corrupts PMM Pricing State Across `RState` Transitions

### Synopsis

`MagicLP.getPMMState` loads persisted targets (`_BASE_TARGET_`, `_QUOTE_TARGET_`) and cached reserves (`_BASE_RESERVE_`, `_QUOTE_RESERVE_`) and delegates reconstruction to `PMMPricing.adjustedTarget`. Because state transitions only persist one target while retaining LP fees in reserves, `getPMMState` produces inconsistent reserve-to-target states and miscalculates swap quotes and mid-prices.

### Technical details

In `MagicLP`, the PMM pricing invariant depends on the consistency between the current reserves (`B = _BASE_RESERVE_`, `Q = _QUOTE_RESERVE_`), the reference targets at equilibrium (`B0 = _BASE_TARGET_`, `Q0 = _QUOTE_TARGET_`), and the regime indicator (`R = _RState_`):

- When `R == RState.ONE`, the pool is at equilibrium and the PMM algorithm assumes `B == B0` and `Q == Q0`.
- When `R == RState.BELOW_ONE`, `B > B0` and `Q < Q0`, and `B0` is the fixed reference target while `Q0` is dynamically reconstructed from `Q` and `B - B0`.
- When `R == RState.ABOVE_ONE`, `Q > Q0` and `B < B0`, and `Q0` is the fixed reference target while `B0` is dynamically reconstructed from `B` and `Q - Q0`.

In `MagicLP.getPMMState()` (lines 193–202):

```solidity
function getPMMState() public view returns (PMMPricing.PMMState memory state) {
    state.i = _I_;
    state.K = _K_;
    state.B = _BASE_RESERVE_;
    state.Q = _QUOTE_RESERVE_;
    state.B0 = _BASE_TARGET_; // will be calculated in adjustedTarget
    state.Q0 = _QUOTE_TARGET_;
    state.R = PMMPricing.RState(_RState_);
    PMMPricing.adjustedTarget(state);
}
```

`PMMPricing.adjustedTarget(state)` only recomputes `state.Q0` when `state.R == RState.BELOW_ONE` and only recomputes `state.B0` when `state.R == RState.ABOVE_ONE`. When `state.R == RState.ONE`, `adjustedTarget` performs no adjustment at all, leaving `state.B0 = _BASE_TARGET_` and `state.Q0 = _QUOTE_TARGET_`.

This design breaks PMM state consistency during swap and flash loan transitions in two ways:

1. **Incomplete target synchronization when transitioning to `RState.ONE`:**
   When `state.R == RState.ABOVE_ONE` and a trader calls `sellBase` with `payBaseAmount == backToOnePayBase` (`state.B0 - state.B`), `PMMPricing.sellBaseToken` sets `receiveQuoteAmount = backToOneReceiveQuote` (`state.Q - state.Q0`) and `newR = RState.ONE`. `querySellBase` then deducts `lpFee` and `mtFee` from `receiveQuoteAmount`. Because `lpFee` is retained in the pool, the post-swap quote reserve becomes `_QUOTE_RESERVE_ = state.Q0 + lpFee > _QUOTE_TARGET_`. However, `sellBase` (lines 256–260) only updates `_BASE_TARGET_ = newBaseTarget.toUint112()` and `_RState_ = uint32(RState.ONE)`, leaving `_QUOTE_TARGET_` at the pre-fee `state.Q0` (and in `flashLoan`, the caller can leave even more excess quote tokens in the pool while still triggering the transition to `RState.ONE`). Symmetrically, when `sellQuote` transitions from `BELOW_ONE` to `ONE`, `_BASE_RESERVE_` retains `lpFee` (`_BASE_RESERVE_ = state.B0 + lpFee > _BASE_TARGET_`), while only `_QUOTE_TARGET_` is updated.
   On subsequent calls to `getPMMState()`, because `state.R == RState.ONE`, `PMMPricing.adjustedTarget(state)` leaves `state.B0 = _BASE_TARGET_ < state.B` or `state.Q0 = _QUOTE_TARGET_ < state.Q`.

2. **R-state and reserve imbalance mismatch when crossing `RState.ONE` with small overshoot:**
   When `state.R == RState.ABOVE_ONE` and `sellBase` is called with `payBaseAmount = backToOnePayBase + deltaB` where `_ROneSellBaseToken(state, deltaB) < lpFee`, `newRState` transitions to `RState.BELOW_ONE`, yet the net quote tokens leaving the pool (`grossReceiveQuote - lpFee`) is less than `state.Q - state.Q0`. Consequently, the pool enters `RState.BELOW_ONE` with `_QUOTE_RESERVE_ > _QUOTE_TARGET_`, and `getPMMState()` reconstructs `state.Q0` from the inflated `_QUOTE_RESERVE_` while `_QUOTE_TARGET_` in storage remains stale. Similarly, if the pool is at `RState.ONE` with `_BASE_RESERVE_ = _BASE_TARGET_ + lpFee` and a tiny `sellBase` (`1 wei`) transitions `_RState_` to `BELOW_ONE`, `state.B - state.B0` in `getPMMState()` abruptly jumps from `0` to `lpFee + 1`, causing an immediate discontinuity in reconstructed `state.Q0` and `getMidPrice()`.

### Assumptions and preconditions

- A `MagicLP` clone pool is initialized and has active liquidity (`_BASE_RESERVE_ > 0` and `_QUOTE_RESERVE_ > 0`) with a non-zero LP fee rate (`_LP_FEE_RATE_ > 0`) or `flashLoan` usage.
- A trade moves the pool away from `RState.ONE` (into `ABOVE_ONE` or `BELOW_ONE`) and a subsequent trade returns the pool to or across `RState.ONE`.

### Proof of concept

1. Initialize a `MagicLP` pool with `1000e18` base tokens and `1000e18` quote tokens (`_BASE_RESERVE_ = _BASE_TARGET_ = 1000e18`, `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1000e18`, `_RState_ = ONE`).
2. User A sells `100e18` base tokens via `sellBase`. The pool transitions to `_RState_ = BELOW_ONE`, `_BASE_RESERVE_ = 1100e18`, `_BASE_TARGET_ = 1000e18`, and `_QUOTE_RESERVE_ < 1000e18`.
3. In `getPMMState()`, `PMMPricing.adjustedTarget(state)` computes `state.Q0` from `_QUOTE_RESERVE_` and `_BASE_RESERVE_ - _BASE_TARGET_ = 100e18`.
4. User B sells exactly `backToOnePayQuote = state.Q0 - _QUOTE_RESERVE_` quote tokens via `sellQuote`.
5. `PMMPricing.sellQuoteToken` matches `payQuoteAmount == backToOnePayQuote`, returning `receiveBaseAmount = state.B - state.B0 = 100e18` and `newR = RState.ONE`.
6. `querySellQuote` deducts `lpFee` from `receiveBaseAmount`. `sellQuote` transfers out `100e18 - lpFee - mtFee` base tokens, sets `_QUOTE_TARGET_ = state.Q0` and `_RState_ = ONE`, and updates `_BASE_RESERVE_` to `1000e18 + lpFee` while leaving `_BASE_TARGET_ = 1000e18`.
7. Call `getPMMState()` (or `getPMMStateForCall()`): it returns `state.R = RState.ONE` with `state.B = 1000e18 + lpFee` and `state.B0 = 1000e18` (`state.B != state.B0`).
8. Next, transfer `1 wei` of base token to the pool and call `sellBase`. Because `state.R == RState.ONE`, `_BASE_TARGET_` remains `1000e18` while `_RState_` flips to `BELOW_ONE`. On the next call to `getPMMState()`, `state.B - state.B0` jumps from `0` to `lpFee + 1 wei`, causing `adjustedTarget(state)` to abruptly increase `state.Q0` and skew `getMidPrice()` and subsequent swap quotes.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** `getPMMState()` and `getPMMStateForCall()` return inconsistent `PMMState` values where `B != B0` or `Q != Q0` at `RState.ONE`, or where `RState` does not match the actual reserve-to-target relationship. This causes inaccurate swap quotes in `querySellBase` / `querySellQuote`, discontinuities in `getMidPrice()` (which also feeds `_twapUpdate()`), and mispricing for external callers relying on `getPMMStateForCall()`.

### Remediation

Ensure that `getPMMState()` (and `PMMPricing.adjustedTarget()`) maintains reserve–target consistency across all `RState` regimes—including synchronizing targets with reserves when `state.R == RState.ONE` and guarding against reserve–target inversions:

```solidity
    function getPMMState() public view returns (PMMPricing.PMMState memory state) {
        state.i = _I_;
        state.K = _K_;
        state.B = _BASE_RESERVE_;
        state.Q = _QUOTE_RESERVE_;
        state.B0 = _BASE_TARGET_;
        state.Q0 = _QUOTE_TARGET_;
        state.R = PMMPricing.RState(_RState_);
        if (
            (state.R == PMMPricing.RState.BELOW_ONE && state.B < state.B0) ||
            (state.R == PMMPricing.RState.ABOVE_ONE && state.Q < state.Q0)
        ) {
            state.R = PMMPricing.RState.ONE;
        }
        if (state.R == PMMPricing.RState.ONE) {
            state.B0 = state.B;
            state.Q0 = state.Q;
        } else {
            PMMPricing.adjustedTarget(state);
        }
    }
```

## Finding 3: Spot Reserve Manipulation in `MagicLP.getReserves` Enables Flash-Loan Inflation of LP Token Valuation

### Synopsis

`MagicLP.getReserves()` returns raw spot reserves (`_BASE_RESERVE_` and `_QUOTE_RESERVE_`), which `MagicLpAggregator` sums to price LP shares. Because PMM swaps with `K > 0` (and direct donations via `sync()`) skew spot reserves without changing `totalSupply`, an attacker can flash-loan and swap to arbitrarily inflate LP collateral valuation.

### Technical details

In `src/mimswap/MagicLP.sol`, `getReserves()` (lines 219–222) returns the current cached spot balances `_BASE_RESERVE_` and `_QUOTE_RESERVE_`:

```solidity
function getReserves() external view returns (uint256 baseReserve, uint256 quoteReserve) {
    baseReserve = _BASE_RESERVE_;
    quoteReserve = _QUOTE_RESERVE_;
}
```

Downstream, `MagicLpAggregator.latestAnswer()` queries `pair.getReserves()` via `_getReserves()` and computes the LP token price as:

$$\text{price} = \frac{\min(\text{basePrice}, \text{quotePrice}) \times (\text{baseReserve} + \text{quoteReserve})}{\text{totalSupply}}$$

Even for closely priced stablecoin pairs (such as MIM/USDB where $i \approx 1$), the sum of spot reserves `_BASE_RESERVE_ + _QUOTE_RESERVE_` is not invariant across swaps under the DODO PMM pricing algorithm (`PMMPricing`) whenever $K > 0$:

1. **Asymmetric Reserve Skew on Swaps**: When a user sells a large amount of `payBaseAmount` via `sellBase()`, `_BASE_RESERVE_` increases by the full `payBaseAmount`, whereas `_QUOTE_RESERVE_` decreases by `receiveQuoteAmount`, which is strictly bounded above by the pool's quote target $Q_0$ (`Math._SolveQuadraticFunctionForTrade`). Consequently, for any large `payBaseAmount`, `_BASE_RESERVE_ + _QUOTE_RESERVE_` grows nearly linearly with `payBaseAmount` while `totalSupply()` remains constant.
2. **Bounded Swap Fee**: In `querySellBase()` (lines 174–176), the LP fee and maintainer fee are calculated as a percentage of `receiveQuoteAmount` (which is capped at $Q_0$) rather than `payBaseAmount`. Thus, an attacker can deposit an arbitrarily large `payBaseAmount` (up to `type(uint112).max - _BASE_RESERVE_`) while paying fees only on the bounded quote output.
3. **Path-Independent Reversal**: After exploiting the inflated LP token price in a single transaction, the attacker can call `sellQuote()` to return the pool to `RState.ONE`. In `PMMPricing.sellQuoteToken()` (lines 84–95), paying `backToOnePayQuote = state.Q0 - state.Q` returns the exact `backToOneReceiveBase = state.B - state.B0`, allowing the attacker to recover their entire excess base input.

Additionally, calling `sync()` after transferring tokens directly to `MagicLP` updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` returned by `getReserves()` without minting shares or updating `_BASE_TARGET_` and `_QUOTE_TARGET_`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with $K > 0$ (e.g., the default $K = 10^{14}$ used in `MIMSwap.t.sol`).
2. `MagicLpAggregator` (or another consumer relying on `getReserves()`) is used to value `MagicLP` shares as collateral.
3. The attacker can obtain capital (e.g., via a flash loan) in either `_BASE_TOKEN_` or `_QUOTE_TOKEN_`.

### Proof of concept

1. Assume a `MagicLP` MIM/USDB pool is at equilibrium (`RState.ONE`) with:
   - `_BASE_RESERVE_ = _BASE_TARGET_ = 100,000e18` MIM
   - `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 100,000e18` USDB
   - `_I_ = 1e18`, `_K_ = 1e14` (0.01%), `_LP_FEE_RATE_ = 1e14` (0.01%)
   - `totalSupply() = 100,000e18`
   - Initial sum of reserves from `getReserves()`: `200,000e18` ($2.00 per LP share).
2. The attacker holds a small LP position as collateral in a Cauldron and flash-loans `10,000,000e18` MIM.
3. The attacker transfers `10,000,000e18` MIM to `MagicLP` and calls `sellBase(attacker)`.
   - `receiveQuoteAmount` is capped below `100,000e18` USDB (approx. `99,980e18` USDB after the 0.01% fee on output).
   - `_BASE_RESERVE_` is updated to `10,100,000e18` MIM and `_QUOTE_RESERVE_` is updated to `~20e18` USDB.
4. `getReserves()` now returns `(10,100,000e18, 20e18)`. When the Cauldron queries `MagicLpAggregator.latestAnswer()`, the numerator `baseReserve + quoteReserve` is `10,100,020e18` while `totalSupply()` is still `100,000e18`, inflating the LP share price by over 50x (from $2.00 to ~$101.00 per share).
5. The attacker borrows the maximum available funds from the Cauldron against their overvalued LP collateral.
6. The attacker transfers `~100,000e18` USDB (`state.Q0 - state.Q`) to `MagicLP` and calls `sellQuote(attacker)`, returning the pool to `RState.ONE`, recovering `~9,999,000e18` MIM (`state.B - state.B0` minus the 0.01% fee), and repaying the flash loan.

### Impact assessment

- **Severity**: High / Critical
- **Technical Effect**: The sum of reserves returned by `MagicLP.getReserves()` can be inflated by orders of magnitude within a single transaction at negligible fee cost.
- **Business Impact**: Any lending market (Cauldron) or protocol relying on `MagicLpAggregator` to price `MagicLP` collateral can be drained of all borrowable assets via flash-loan price manipulation.

### Remediation

To ensure reserve observations reflect the pool's equilibrium value rather than transient post-swap spot skews, return the equilibrium target reserves (`state.B0` and `state.Q0`) computed by `getPMMState()` in `getReserves()` (and guard against read-only reentrancy with `nonReadReentrant`):

```solidity
    function getReserves() external view nonReadReentrant returns (uint256 baseReserve, uint256 quoteReserve) {
        PMMPricing.PMMState memory state = getPMMState();
        baseReserve = state.B0;
        quoteReserve = state.Q0;
    }
```

*(Note: Internal periphery helpers that require raw spot reserve accounting to compute pending token inputs, such as `Router`, should read `_BASE_RESERVE_` and `_QUOTE_RESERVE_` directly.)*

## Finding 4: Unaccounted Balance and Rebase Yield Attribution in `MagicLP.sellBase` and `MagicLP.getQuoteInput` Allows Theft of Excess Pool Tokens

### Synopsis

`MagicLP.sellBase` and `MagicLP.getQuoteInput` derive swap input amounts strictly from the difference between the pool's current ERC20 token balance and its recorded reserve (`balanceOf(address(this)) - reserve`). Any positive balance divergence from rebasing yield or unsynchronized transfers is attributed to the next caller, enabling permissionless extraction of value.

### Technical details

In `src/mimswap/MagicLP.sol`, swap input accounting is balance-derived rather than pull-based or yield-segregated:

1. In `getQuoteInput()` (lines 232–234) and `getBaseInput()` (lines 228–230), the available input is computed as:
   ```solidity
   return _QUOTE_TOKEN_.balanceOf(address(this)) - uint256(_QUOTE_RESERVE_);
   ```
2. In `sellBase(address to)` (lines 244–265), the contract reads the current base token balance and treats the entire surplus over `_BASE_RESERVE_` as the caller's payment (`baseInput`):
   ```solidity
   uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
   uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
   ```
   It then prices `baseInput` via `querySellBase(tx.origin, baseInput)` (line 250), transfers `receiveQuoteAmount` of `_QUOTE_TOKEN_` to `to` (line 252), and updates `_BASE_RESERVE_` to `baseBalance` via `_setReserve` (line 262).

This accounting breaks whenever a token's balance in the pool changes outside of synchronized pool actions:

- **Rebasing and Yield-Related Reserve Divergence:** In `BlastMagicLP` (`src/blast/BlastMagicLP.sol`), native yield tokens only switch from Blast's default `AUTOMATIC` (rebasing) yield mode to `CLAIMABLE` mode if they are registered in `BlastTokenRegistry.nativeYieldTokens` at initialization (lines 104–112) or after an operator manually invokes `updateTokenClaimables()` (lines 64–66, as seen in `test/MIMSwap.t.sol`). While a pool token is in rebasing mode—or if any standard rebasing token is used in a `MagicLP` pool—positive rebases automatically increase `_BASE_TOKEN_.balanceOf(address(this))` or `_QUOTE_TOKEN_.balanceOf(address(this))` while `_BASE_RESERVE_` and `_QUOTE_RESERVE_` remain static until the owner calls `ratioSync()` (line 504) or `sync()` (line 134).
- **Input Attribution Failure:** Because `sellBase` attributes the entire `baseBalance - _BASE_RESERVE_` delta to `msg.sender`, any unprivileged caller can invoke `sellBase(to)` without transferring any `_BASE_TOKEN_` and receive `_QUOTE_TOKEN_` in exchange for the pool's accrued yield or unsynchronized token surplus.
- **Underflow on Negative Divergence:** Conversely, if a token experiences a negative rebase or balance contraction such that `balanceOf(address(this)) < reserve`, the subtraction in `getQuoteInput()` (line 233) and `sellBase()` (lines 245–246) reverts due to arithmetic underflow, halting swaps until `sync()` or `ratioSync()` is executed.

### Assumptions and preconditions

1. A `MagicLP` / `BlastMagicLP` pool holds a token whose `balanceOf(address(this))` increases relative to `_BASE_RESERVE_` or `_QUOTE_RESERVE_` (e.g., a rebasing token, a Blast native-yield token prior to `updateTokenClaimables()` activation, or an unsynchronized token transfer made before `ratioSync()`/`sync()`).
2. An unprivileged caller invokes `sellBase` before `sync()` or `ratioSync()` is executed.

### Proof of concept

1. A `BlastMagicLP` pool is deployed and initialized with `_BASE_TOKEN_` (a rebasing token or a Blast yield-bearing token not yet enabled as claimable on the pool) and `_QUOTE_TOKEN_`, with initial reserves `_BASE_RESERVE_ = 100,000e18` and `_QUOTE_RESERVE_ = 100,000e18`.
2. Yield accrues via rebase, increasing `_BASE_TOKEN_.balanceOf(address(this))` from `100,000e18` to `101,000e18` while `_BASE_RESERVE_` remains `100,000e18`.
3. Before the owner calls `ratioSync()` or `updateTokenClaimables()`, an observer calls `lp.sellBase(observer)` directly without sending any `_BASE_TOKEN_`.
4. Inside `sellBase`:
   - `baseBalance` is read as `101,000e18` (line 245).
   - `baseInput` is calculated as `101,000e18 - 100,000e18 = 1,000e18` (line 246).
   - `querySellBase` computes `receiveQuoteAmount` for `1,000e18` of base token.
   - `_transferQuoteOut(observer, receiveQuoteAmount)` transfers ~`1,000e18` (minus fees) of `_QUOTE_TOKEN_` to `observer`.
   - `_setReserve` updates `_BASE_RESERVE_` to `101,000e18` and `_QUOTE_RESERVE_` to the reduced quote balance.
5. The caller extracts the accrued yield in `_QUOTE_TOKEN_` at the expense of the pool's liquidity providers or fee collector.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:** Any positive divergence between actual token balances and tracked reserves is treated as swap input and can be immediately extracted by any caller via `sellBase` (or `sellQuote`). Any negative divergence causes `getQuoteInput` and `sellBase` to revert with an arithmetic underflow.
- **Business Impact:** Loss of accrued yield or unsynchronized liquidity for LPs and the protocol fee collector (`feeTo`), or temporary denial of service on pool swaps if token balances contract below recorded reserves.

### Remediation

Ensure that rebasing tokens are either strictly configured as claimable upon initialization (reverting if a rebasing token cannot be set to claimable) or explicitly synchronize/account for accrued yield prior to computing swap inputs, and pull exact user-specified input amounts or cap/synchronize reserve divergences before executing trades:

```solidity
    function getQuoteInput() public view returns (uint256 input) {
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
        return quoteBalance > uint256(_QUOTE_RESERVE_) ? quoteBalance - uint256(_QUOTE_RESERVE_) : 0;
    }

    function version() external pure virtual returns (string memory) {
        return "MagicLP 1.0.0";
    }

    //////////////////////////////////////////////////////////////////////////////////////
    /// TRADE FUNCTIONS
    //////////////////////////////////////////////////////////////////////////////////////

    function sellBase(address to, uint256 payBaseAmount) external nonReentrant returns (uint256 receiveQuoteAmount) {
        _BASE_TOKEN_.safeTransferFrom(msg.sender, address(this), payBaseAmount);
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 baseInput = payBaseAmount;
```

## Finding 5: Reserve-Target-Regime Invariant Violation in `MagicLP.sellBase` When Transitioning From `ABOVE_ONE` to `ONE` or `BELOW_ONE`

### Synopsis

In `MagicLP.sellBase`, transitioning `_RState_` from `ABOVE_ONE` to `ONE` (or `BELOW_ONE`) retains LP fees in `_QUOTE_RESERVE_` without updating `_QUOTE_TARGET_`. This violates the `RState.ONE` target-reserve equality invariant (`_QUOTE_RESERVE_ == _QUOTE_TARGET_`), causing subsequent trades and mid-price calculations to double-count retained LP fees and misprice swaps.

### Technical details

In `MagicLP`, the PMM pricing state (`PMMPricing.PMMState`) relies on strict regime invariants between reserves (`B = _BASE_RESERVE_`, `Q = _QUOTE_RESERVE_`), targets (`B0 = _BASE_TARGET_`, `Q0 = _QUOTE_TARGET_`), and the regime state `_RState_`:
- In `RState.ONE`: `B == B0` and `Q == Q0`.
- In `RState.ABOVE_ONE`: `B <= B0` and `Q >= Q0`, where `Q0` is fixed in storage (`_QUOTE_TARGET_`) and `B0` is dynamically computed in `PMMPricing.adjustedTarget(state)` via `Math._SolveQuadraticFunctionForTarget(state.B, state.Q - state.Q0, DecimalMath.reciprocalFloor(state.i), state.K)`.
- In `RState.BELOW_ONE`: `B >= B0` and `Q <= Q0`, where `B0` is fixed in storage (`_BASE_TARGET_`) and `Q0` is dynamically computed in `PMMPricing.adjustedTarget(state)`.

When `sellBase(address to)` (at `src/mimswap/MagicLP.sol:244`) is called while `_RState_ == uint32(PMMPricing.RState.ABOVE_ONE)`:
1. `querySellBase(tx.origin, baseInput)` fetches `state = getPMMState()` (where `state.B0` is dynamically solved so that paying `backToOnePayBase = state.B0 - state.B` base tokens corresponds to withdrawing `backToOneReceiveQuote = state.Q - state.Q0` quote tokens to return the curve to `R = 1`).
2. Inside `PMMPricing.sellBaseToken(state, payBaseAmount)`:
   - **Case 2.2 (`payBaseAmount == backToOnePayBase`)**: Returns `receiveQuoteAmount = backToOneReceiveQuote = state.Q - state.Q0` and `newR = RState.ONE`.
   - **Case 2.3 (`payBaseAmount > backToOnePayBase`)**: Returns `receiveQuoteAmount = backToOneReceiveQuote + _ROneSellBaseToken(state, payBaseAmount - backToOnePayBase)` and `newR = RState.BELOW_ONE`.
3. `querySellBase` then deducts both the LP fee and maintainer fee from `receiveQuoteAmount`:
   ```solidity
   mtFee = DecimalMath.mulFloor(receiveQuoteAmount, mtFeeRate);
   receiveQuoteAmount = receiveQuoteAmount - DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate) - mtFee;
   newBaseTarget = state.B0;
   ```
4. In `sellBase`, only `receiveQuoteAmount` and `mtFee` are transferred out of the pool (`_transferQuoteOut`). The LP fee (`lpFee = DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate)`) remains inside the contract's quote token balance.
5. `sellBase` then updates the target and reserve state:
   ```solidity
   if (_RState_ != uint32(newRState)) {
       _BASE_TARGET_ = newBaseTarget.toUint112();
       _RState_ = uint32(newRState);
       emit RChange(newRState);
   }

   _setReserve(baseBalance, _QUOTE_TOKEN_.balanceOf(address(this)));
   ```

This creates two curve-transition inconsistencies:
- **Transition from `ABOVE_ONE` to `ONE` (`baseInput == backToOnePayBase`)**: The pool transfers out `(state.Q - state.Q0) - lpFee` quote tokens, leaving `_QUOTE_RESERVE_ = state.Q0 + lpFee`, while `_QUOTE_TARGET_` remains at `state.Q0`. `_RState_` is set to `RState.ONE`, where `PMMPricing.adjustedTarget` performs no adjustment on `state.B0` or `state.Q0`. Thus, the pool enters `RState.ONE` with `_QUOTE_RESERVE_ > _QUOTE_TARGET_`. If a subsequent `sellQuote` is executed (even for 1 wei), `_RState_` transitions back to `ABOVE_ONE` and sets `_QUOTE_TARGET_ = state.Q0` (still excluding `lpFee`). On the next `getPMMState()` call, `state.Q - state.Q0` still contains `lpFee`, causing `adjustedTarget` to inflate `state.B0` a second time by the already-accounted LP fee.
- **Transition from `ABOVE_ONE` to `BELOW_ONE` (`baseInput > backToOnePayBase`)**: In Case 2.3, `_ROneSellBaseToken(state, payBaseAmount - backToOnePayBase)` prices the `BELOW_ONE` leg using the stale `state.Q0` rather than the actual post-fee quote reserve at the `R = 1` boundary (`state.Q0 + lpFee_from_leg1`).

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with non-zero liquidity and a non-zero `_LP_FEE_RATE_` (which is enforced by the protocol's `MIN_LP_FEE_RATE`).
2. The pool is currently in `_RState_ == uint32(PMMPricing.RState.ABOVE_ONE)` (i.e., after any `sellQuote` swap).

### Proof of concept

Consider a `MagicLP` pool with `i = 1e18`, `K -> 0` (stable curve), and `_LP_FEE_RATE_ = 0.01e18` (1%, with `mtFeeRate = 0` for simplicity):
1. **Initial State (`RState.ONE`)**:
   - `_BASE_RESERVE_ = 1000e18`, `_BASE_TARGET_ = 1000e18`
   - `_QUOTE_RESERVE_ = 1000e18`, `_QUOTE_TARGET_ = 1000e18`
2. **Move Pool to `ABOVE_ONE`**:
   - A trader transfers `100e18` quote tokens and calls `sellQuote`.
   - Pre-fee output is `~100e18` base tokens; after the 1% LP fee (`1e18`), `99e18` base tokens are sent to the trader.
   - Pool state is now `_RState_ = ABOVE_ONE`, `_BASE_RESERVE_ = 901e18`, `_QUOTE_RESERVE_ = 1100e18`, `_QUOTE_TARGET_ = 1000e18`.
   - `getPMMState()` computes `state.B0 = 1001e18` (reflecting the `1e18` base LP fee earned), so `backToOnePayBase = 1001e18 - 901e18 = 100e18` and `backToOneReceiveQuote = 1100e18 - 1000e18 = 100e18`.
3. **Trigger Boundary Transition in `sellBase` (`ABOVE_ONE -> ONE`)**:
   - A trader transfers `baseInput = 100e18` base tokens (`== backToOnePayBase`) into the pool and calls `sellBase(to)`.
   - `PMMPricing.sellBaseToken` enters Case 2.2 (`payBaseAmount == backToOnePayBase`), returning `receiveQuoteAmount = 100e18` and `newR = RState.ONE`.
   - `querySellBase` deducts the 1% LP fee (`1e18`), returning `receiveQuoteAmount = 99e18` and `newBaseTarget = 1001e18`.
   - `sellBase` transfers `99e18` quote tokens out and sets `_BASE_TARGET_ = 1001e18`, `_RState_ = RState.ONE`, `_BASE_RESERVE_ = 1001e18`, and `_QUOTE_RESERVE_ = 1100e18 - 99e18 = 1001e18`.
   - `_QUOTE_TARGET_` remains `1000e18`.
4. **Resulting Broken Invariant**:
   - The pool is in `RState.ONE`, but `_QUOTE_RESERVE_ (1001e18) != _QUOTE_TARGET_ (1000e18)`.
   - If any user now sells `1 wei` of quote token via `sellQuote`, `_RState_` becomes `ABOVE_ONE` with `_QUOTE_TARGET_ = 1000e18` and `_QUOTE_RESERVE_ = 1001e18 + 1`.
   - On the next `getPMMState()` call, `adjustedTarget` sees `state.Q - state.Q0 = 1e18 + 1` and computes `state.B0 = 1002e18`, inflating `B0` by an extra `1e18` that does not correspond to any base fee earned and shifting the pool's mid-price and reserve targets away from the true asset balances.

### Impact assessment

- **Severity**: Medium / High.
- **Technical Effect**: Breaks the PMM `RState.ONE` target-reserve invariant (`_QUOTE_RESERVE_ == _QUOTE_TARGET_`), miscalculates `state.Q0` during `ABOVE_ONE -> BELOW_ONE` transitions, and causes `state.B0` to be inflated when re-entering `ABOVE_ONE` after an `ABOVE_ONE -> ONE` transition.
- **Business Impact**: Distorts pool pricing, TWAP cumulative price updates (`_twapUpdate`), and target reserves (`_BASE_TARGET_` / `_QUOTE_TARGET_`), allowing traders to extract value from liquidity providers due to artificial target inflation.

### Remediation

When `sellBase` transitions `_RState_` to `RState.ONE` (or when syncing targets at the `R = 1` boundary), update `_QUOTE_TARGET_` to match the post-transfer quote balance so that retained LP fees are reflected in `_QUOTE_TARGET_` and the `RState.ONE` invariant (`_QUOTE_RESERVE_ == _QUOTE_TARGET_`) holds:

```solidity
    function sellBase(address to) external nonReentrant returns (uint256 receiveQuoteAmount) {
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
        uint256 mtFee;
        uint256 newBaseTarget;
        PMMPricing.RState newRState;
        (receiveQuoteAmount, mtFee, newRState, newBaseTarget) = querySellBase(tx.origin, baseInput);

        _transferQuoteOut(to, receiveQuoteAmount);
        _transferQuoteOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);

        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));

        // update TARGET
        if (_RState_ != uint32(newRState)) {
            _BASE_TARGET_ = newBaseTarget.toUint112();
            if (newRState == PMMPricing.RState.ONE) {
                _QUOTE_TARGET_ = quoteBalance.toUint112();
            }
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }

        _setReserve(baseBalance, quoteBalance);

        emit Swap(address(_BASE_TOKEN_), address(_QUOTE_TOKEN_), baseInput, receiveQuoteAmount, msg.sender, to);
    }
```

## Finding 6: Inconsistent Balance-Based Settlement Accounting in `MagicLP.sellBase` Corrupts PMM Reserve and Target State

### Synopsis

`MagicLP.sellBase` computes swap outputs and PMM target updates using cached reserves (`_BASE_RESERVE_` and `_QUOTE_RESERVE_`) and a pre-settlement `baseBalance` snapshot, but updates `_QUOTE_RESERVE_` using a post-settlement `_QUOTE_TOKEN_.balanceOf(address(this))` snapshot. This discrepancy allows reserve-target desynchronization and PMM pricing corruption.

### Technical details

In `MagicLP.sellBase` (`src/mimswap/MagicLP.sol`), swap settlement relies on balance-based accounting:

1. At lines 245–246, the contract reads `baseBalance = _BASE_TOKEN_.balanceOf(address(this))` and computes `baseInput = baseBalance - uint256(_BASE_RESERVE_)`.
2. At line 250, `querySellBase(tx.origin, baseInput)` loads the PMM state via `getPMMState()`, which uses the stored `_BASE_RESERVE_` (`state.B`), `_QUOTE_RESERVE_` (`state.Q`), `_BASE_TARGET_` (`state.B0`), and `_QUOTE_TARGET_` (`state.Q0`) to compute `receiveQuoteAmount`, `mtFee`, `newRState`, and `newBaseTarget`.
3. At lines 252–253, the contract transfers `receiveQuoteAmount` to `to` and `mtFee` to `_MT_FEE_RATE_MODEL_.maintainer()`.
4. At lines 256–260, if `_RState_` changed, `_BASE_TARGET_` is updated to `newBaseTarget` and `_RState_` is updated to `newRState`.
5. At line 262, `_setReserve(baseBalance, _QUOTE_TOKEN_.balanceOf(address(this)))` updates `_BASE_RESERVE_` to the pre-settlement `baseBalance` and `_QUOTE_RESERVE_` to the post-settlement `_QUOTE_TOKEN_.balanceOf(address(this))`.

This settlement accounting is flawed because the PMM state transition in `querySellBase` assumes that the quote reserve starts at `_QUOTE_RESERVE_` and decreases by `receiveQuoteAmount + mtFee` (leaving only the LP fee in the pool), whereas line 262 blindly sets `_QUOTE_RESERVE_` to the current `_QUOTE_TOKEN_.balanceOf(address(this))`:

- If `to == address(this)` (which is not restricted in `sellBase`), `_transferQuoteOut(to, receiveQuoteAmount)` transfers `receiveQuoteAmount` back to the `MagicLP` contract itself. `_QUOTE_TOKEN_.balanceOf(address(this))` does not decrease by `receiveQuoteAmount`, yet `_BASE_TARGET_` and `_RState_` are updated as if `receiveQuoteAmount` left the pool, and `_QUOTE_RESERVE_` retains `receiveQuoteAmount`.
- If any unaccounted `_QUOTE_TOKEN_` balance exists in the pool prior to `sellBase` (e.g., from uncollected yield/rebasing drift or direct transfers), `querySellBase` prices the trade and calculates `newBaseTarget` against the old `_QUOTE_RESERVE_`, while `_setReserve` absorbs the entire extra `_QUOTE_TOKEN_` balance into `_QUOTE_RESERVE_` without adjusting `_QUOTE_TARGET_`. In `PMMPricing.adjustedTarget`, `state.Q0` (when `R == BELOW_ONE`) or `state.B0` (when `R == ABOVE_ONE`) is dynamically derived from `state.Q` and `state.B - state.B0` (or `state.Q - state.Q0`), so inflating `_QUOTE_RESERVE_` relative to `_QUOTE_TARGET_` and `_BASE_TARGET_` distorts the PMM invariant and mid-price (`_twapUpdate`). Conversely, if `_QUOTE_TOKEN_` balance drifts downward below `_QUOTE_TARGET_` while `_RState_ == ABOVE_ONE`, updating `_QUOTE_RESERVE_` to `_QUOTE_TOKEN_.balanceOf(address(this))` makes `_QUOTE_RESERVE_ < _QUOTE_TARGET_`, causing `state.Q - state.Q0` in `PMMPricing.adjustedTarget` (line 196) to revert with arithmetic underflow on subsequent swaps and price queries.

### Assumptions and preconditions

- A `MagicLP` pool is initialized and holds active liquidity in `_BASE_TOKEN_` and `_QUOTE_TOKEN_`.
- Either a caller invokes `sellBase` with `to == address(this)`, or `_QUOTE_TOKEN_.balanceOf(address(this))` diverges from `_QUOTE_RESERVE_` prior to or during the settlement of `sellBase`.

### Proof of concept

1. Assume a `MagicLP` pool is in `RState.ABOVE_ONE` with `_BASE_RESERVE_ = 900e18`, `_BASE_TARGET_ = 1000e18`, `_QUOTE_RESERVE_ = 1100e18`, and `_QUOTE_TARGET_ = 1000e18` (with `i = 1e18` and negligible `K` and fees for clarity).
2. A caller transfers `100e18` of `_BASE_TOKEN_` to the pool and calls `sellBase(address(lp))`.
3. `baseInput` is computed as `1000e18 - 900e18 = 100e18`.
4. `querySellBase` computes `backToOnePayBase = 1000e18 - 900e18 = 100e18` and `backToOneReceiveQuote = 1100e18 - 1000e18 = 100e18`, returning `receiveQuoteAmount ≈ 100e18`, `newRState = RState.ONE`, and `newBaseTarget = 1000e18`.
5. `_transferQuoteOut(address(lp), receiveQuoteAmount)` transfers the quote tokens from the pool to itself, leaving `_QUOTE_TOKEN_.balanceOf(address(lp))` at `~1100e18`.
6. `sellBase` sets `_BASE_TARGET_ = 1000e18` and `_RState_ = uint32(RState.ONE)`, leaving `_QUOTE_TARGET_ = 1000e18`.
7. Line 262 executes `_setReserve(1000e18, _QUOTE_TOKEN_.balanceOf(address(this)))`, setting `_BASE_RESERVE_ = 1000e18` and `_QUOTE_RESERVE_ = 1100e18`.
8. The pool is now in `RState.ONE` with `_QUOTE_RESERVE_ = 1100e18 != _QUOTE_TARGET_ = 1000e18`, violating the `RState.ONE` invariant (`B == B0` and `Q == Q0`) and distorting subsequent PMM target calculations and reserve accounting.

### Impact assessment

**Severity: Medium**

Desynchronization between `_QUOTE_RESERVE_`, `_QUOTE_TARGET_`, `_BASE_TARGET_`, and `_RState_` corrupts the PMM pricing curve and TWAP oracle updates (`_twapUpdate`), and can cause subsequent swap/query operations to misprice trades or revert with arithmetic underflow in `PMMPricing.adjustedTarget`.

### Remediation

Validate that `to != address(this)` and update reserves consistently with the pre-settlement reserve state and actual settled amounts, or synchronize reserves before computing the PMM state transition.

```solidity
    function sellBase(address to) external nonReentrant returns (uint256 receiveQuoteAmount) {
        if (to == address(this)) {
            revert ErrInvalidRecipient();
        }
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
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

        _setReserve(baseBalance, uint256(_QUOTE_RESERVE_) - receiveQuoteAmount - mtFee);

        emit Swap(address(_BASE_TOKEN_), address(_QUOTE_TOKEN_), baseInput, receiveQuoteAmount, msg.sender, to);
    }
```

## Finding 7: Unvalidated Self-Transfer and Direct Balance Sync in `MagicLP.sellQuote` Corrupts PMM Reserve and Target Accounting

### Synopsis

In `MagicLP.sellQuote`, `_BASE_RESERVE_` is updated to `_BASE_TOKEN_.balanceOf(address(this))` after transferring `receiveBaseAmount` to `to`, without disallowing `to == address(this)` or accounting for unattributed base token balances, allowing a caller to desynchronize `_BASE_RESERVE_` from `_RState_` and `_QUOTE_TARGET_` and corrupt PMM pricing and targets.

### Technical details

`MagicLP.sellQuote(address to)` computes the swap output and PMM state transition from the cached reserves (`_BASE_RESERVE_` and `_QUOTE_RESERVE_`) and the incoming quote token delta:

```solidity
uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
uint256 mtFee;
uint256 newQuoteTarget;
PMMPricing.RState newRState;
(receiveBaseAmount, mtFee, newRState, newQuoteTarget) = querySellQuote(tx.origin, quoteInput);
```

Inside `querySellQuote`, `getPMMState()` loads `state.B = _BASE_RESERVE_` and `state.Q = _QUOTE_RESERVE_`, adjusts the active target (`state.Q0` or `state.B0`), and `PMMPricing.sellQuoteToken(state, quoteInput)` computes `receiveBaseAmount` and `newRState` under the invariant that `receiveBaseAmount + mtFee` of `_BASE_TOKEN_` will leave the pool (reducing the pool's base reserve to `_BASE_RESERVE_ - receiveBaseAmount - mtFee`).

However, `sellQuote` then performs:

```solidity
_transferBaseOut(to, receiveBaseAmount);
_transferBaseOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);

// update TARGET
if (_RState_ != uint32(newRState)) {
    _QUOTE_TARGET_ = newQuoteTarget.toUint112();
    _RState_ = uint32(newRState);
    emit RChange(newRState);
}

_setReserve(_BASE_TOKEN_.balanceOf(address(this)), quoteBalance);
```

Because `sellQuote` does not validate `to != address(this)` and blindly overwrites `_BASE_RESERVE_` with `_BASE_TOKEN_.balanceOf(address(this))` at line 285 (rather than accounting only for the expected reserve delta or syncing prior to pricing):

1. **Self-transfer via `to == address(this)`:** When `to == address(this)`, `_transferBaseOut(to, receiveBaseAmount)` transfers `receiveBaseAmount` of `_BASE_TOKEN_` from the pool back to itself. `_BASE_TOKEN_.balanceOf(address(this))` only decreases by `mtFee` instead of `receiveBaseAmount + mtFee`. Line 285 then sets `_BASE_RESERVE_` to this unreduced balance while `_QUOTE_RESERVE_` is set to `quoteBalance` and `_RState_` / `_QUOTE_TARGET_` are updated as though `receiveBaseAmount` had exited the pool.
2. **PMM Target and Oracle Corruption:**
   - When `newRState == PMMPricing.RState.ABOVE_ONE`, `getPMMState()` calls `PMMPricing.adjustedTarget()`, which computes `state.B0 = Math._SolveQuadraticFunctionForTarget(state.B, state.Q - state.Q0, DecimalMath.reciprocalFloor(state.i), state.K)`. Because `state.B` (`_BASE_RESERVE_`) still contains `receiveBaseAmount` while `state.Q - state.Q0` reflects the full `quoteInput`, `state.B0` is solved from an inflated `V1 = state.B`. This immediately distorts `getMidPrice()` (called during `_twapUpdate()` inside `_setReserve`) and, when a subsequent `sellBase` transitions `_RState_` back to `ONE` or `BELOW_ONE`, permanently writes the inflated `state.B0` to `_BASE_TARGET_`.
   - When `_RState_ == PMMPricing.RState.BELOW_ONE` and `quoteInput == backToOnePayQuote`, `newRState` transitions to `RState.ONE` while `_BASE_RESERVE_` remains greater than `_BASE_TARGET_`, violating the PMM invariant that `B <= B0` when `R >= 1` (`RState.ONE` / `RState.ABOVE_ONE`) because `adjustedTarget()` is a no-op in `RState.ONE`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized and holds non-zero reserves of `_BASE_TOKEN_` and `_QUOTE_TOKEN_`.
2. `_BASE_TOKEN_` and `_QUOTE_TOKEN_` are standard ERC20 tokens that allow transfers where sender and recipient are both `address(this)`.
3. Any external caller can directly invoke `MagicLP.sellQuote(address to)` by transferring `_QUOTE_TOKEN_` to the pool and passing `to = address(lp)`.

### Proof of concept

1. Assume a `MagicLP` pool is initialized with `_BASE_RESERVE_ = 1000e18`, `_QUOTE_RESERVE_ = 1000e18`, `_BASE_TARGET_ = 1000e18`, `_QUOTE_TARGET_ = 1000e18`, `_I_ = 1e18`, `_K_ = 1e18`, and `_RState_ = RState.ONE`.
2. A caller transfers `900e18` `_QUOTE_TOKEN_` to the pool contract and calls `sellQuote(address(lp))`.
3. Inside `sellQuote`:
   - `quoteBalance = 1900e18` and `quoteInput = 900e18`.
   - `querySellQuote` calculates `receiveBaseAmount` (~`473.68e18` before fees), `newRState = RState.ABOVE_ONE`, and `newQuoteTarget = 1000e18`.
   - `_transferBaseOut(address(lp), receiveBaseAmount)` transfers `receiveBaseAmount` from the pool to itself; `_BASE_TOKEN_.balanceOf(address(lp))` remains `~1000e18` (less `mtFee`).
   - `_QUOTE_TARGET_` is set to `1000e18` and `_RState_` is set to `RState.ABOVE_ONE`.
   - `_setReserve(_BASE_TOKEN_.balanceOf(address(this)), 1900e18)` sets `_BASE_RESERVE_ = ~1000e18` (instead of `~526.32e18`) and `_QUOTE_RESERVE_ = 1900e18`.
4. On any subsequent call to `getPMMState()` (including inside `_twapUpdate()` at line 564), `PMMPricing.adjustedTarget()` computes `state.B0` from `state.B = 1000e18` and `state.Q - state.Q0 = 900e18`, yielding `state.B0 = ~1572.38e18` instead of `1000e18`.
5. Alternatively, if the pool is in `RState.BELOW_ONE` (e.g., `_BASE_RESERVE_ = 1500e18, _BASE_TARGET_ = 1000e18, _QUOTE_RESERVE_ = 526.32e18, _QUOTE_TARGET_ = 1000e18`) and the caller transfers `backToOnePayQuote` (`473.68e18`) of `_QUOTE_TOKEN_` and calls `sellQuote(address(lp))`, `_RState_` transitions to `RState.ONE` with `_BASE_TARGET_ = 1000e18` while `_BASE_RESERVE_` remains `~1500e18`, breaking the `RState.ONE` invariant `B <= B0` and distorting `getMidPrice()`.

### Impact assessment

**Severity: Medium / High**

- **Technical Impact:** Calling `sellQuote(address(this))` (or having unattributed `_BASE_TOKEN_` balances absorbed at line 285) breaks the PMM invariant relating `_BASE_RESERVE_`, `_QUOTE_RESERVE_`, `_BASE_TARGET_`, `_QUOTE_TARGET_`, and `_RState_`. This corrupts `state.B0` / `state.Q0`, `getMidPrice()`, and `_BASE_PRICE_CUMULATIVE_LAST_` (TWAP).
- **Business Impact:** Corrupting the pool's target parameters (`_BASE_TARGET_`, `_QUOTE_TARGET_`, `_RState_`) and `getMidPrice()` skews swap pricing for subsequent traders and distorts downstream price consumers (such as `MagicLpAggregator` or TWAP consumers).

### Remediation

Validate that `to != address(this)` in `sellQuote` (and `sellBase`), and update `_BASE_RESERVE_` based on the accounted reserve or ensure no self-transfer can leave `receiveBaseAmount` in the pool's balance when updating reserves:

```solidity
    function sellQuote(address to) external nonReentrant returns (uint256 receiveBaseAmount) {
        if (to == address(this)) {
            revert ErrZeroAddress();
        }
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
        uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
        uint256 mtFee;
        uint256 newQuoteTarget;
        PMMPricing.RState newRState;
        (receiveBaseAmount, mtFee, newRState, newQuoteTarget) = querySellQuote(tx.origin, quoteInput);

        _transferBaseOut(to, receiveBaseAmount);
        _transferBaseOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);

        // update TARGET
        if (_RState_ != uint32(newRState)) {
            _QUOTE_TARGET_ = newQuoteTarget.toUint112();
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }

        _setReserve(_BASE_TOKEN_.balanceOf(address(this)), quoteBalance);

        emit Swap(address(_QUOTE_TOKEN_), address(_BASE_TOKEN_), quoteInput, receiveBaseAmount, msg.sender, to);
    }
```

## Finding 8: Missing `_BASE_TARGET_` Update on Equilibrium Transition in `MagicLP.sellQuote` Causes Reserve-Target Desynchronization

### Synopsis

In `MagicLP.sellQuote`, when a quote-token swap brings the pool from `RState.BELOW_ONE` back to equilibrium (`RState.ONE`) or crosses into `RState.ABOVE_ONE`, the LP fee is retained in base tokens inside `_BASE_RESERVE_`, but `_BASE_TARGET_` is not updated to reflect the retained fee, breaking the PMM reserve-target invariant.

### Technical details

In DODO V2 / `MagicLP`'s PMM model, the pool tracks reserves `(_BASE_RESERVE_, _QUOTE_RESERVE_)` and equilibrium targets `(_BASE_TARGET_, _QUOTE_TARGET_)` across three regimes (`RState.ONE`, `RState.ABOVE_ONE`, `RState.BELOW_ONE`):

1. When `_RState_ == RState.BELOW_ONE`, `_BASE_RESERVE_ > _BASE_TARGET_` (`B > B0`) and `_QUOTE_RESERVE_ < state.Q0` (`Q < Q0`). `getPMMState()` calls `PMMPricing.adjustedTarget(state)`, which dynamically computes `state.Q0` from `state.Q` and `state.B - state.B0` while using the stored `_BASE_TARGET_` as `state.B0`.
2. When `sellQuote` is called in `RState.BELOW_ONE` with `quoteInput == backToOnePayQuote` (`state.Q0 - state.Q`), `PMMPricing.sellQuoteToken` returns:
   - `receiveBaseAmount = backToOneReceiveBase = state.B - state.B0`
   - `newR = RState.ONE`
3. In `querySellQuote` (lines 187–190), the LP fee (`lpFee = DecimalMath.mulFloor(receiveBaseAmount, lpFeeRate)`) and maintainer fee (`mtFee`) are deducted from `receiveBaseAmount`:
   ```solidity
   mtFee = DecimalMath.mulFloor(receiveBaseAmount, mtFeeRate);
   receiveBaseAmount = receiveBaseAmount - DecimalMath.mulFloor(receiveBaseAmount, lpFeeRate) - mtFee;
   newQuoteTarget = state.Q0;
   ```
4. In `sellQuote` (lines 275–285), only `receiveBaseAmount + mtFee = (state.B - state.B0) - lpFee` base tokens are transferred out of the pool. The `lpFee` base tokens remain in the contract, so the new base balance is `_BASE_TARGET_ + lpFee`.
5. However, lines 279–283 only update `_QUOTE_TARGET_` and `_RState_`:
   ```solidity
   if (_RState_ != uint32(newRState)) {
       _QUOTE_TARGET_ = newQuoteTarget.toUint112();
       _RState_ = uint32(newRState);
       emit RChange(newRState);
   }
   ```
   `_BASE_TARGET_` is left unchanged at the old `B0`, while `_BASE_RESERVE_` is set to `B0 + lpFee` on line 285.

Because `PMMPricing.adjustedTarget(state)` performs no target adjustment when `state.R == RState.ONE`, the pool enters `RState.ONE` with `state.B0 < state.B` (`_BASE_TARGET_ < _BASE_RESERVE_`), violating the equilibrium invariant `B == B0` and `Q == Q0`:
- `_setReserve` immediately calls `_twapUpdate() -> getMidPrice()`, which executes the `else` branch of `PMMPricing.getMidPrice`:
  ```solidity
  uint256 R = DecimalMath.divFloor((state.B0 * state.B0) / state.B, state.B);
  R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
  return DecimalMath.mulFloor(state.i, R);
  ```
  Since `state.B0 < state.B`, `R < 1e18` inside the `R >= 1` branch, producing an underestimated mid-price `< state.i` at equilibrium.
- If the next swap is `sellBase`, `_ROneSellBaseToken` transitions `_RState_` back to `BELOW_ONE` and sets `_BASE_TARGET_ = state.B0` (still missing `lpFee`), leaving `state.B - state.B0` permanently inflated by `lpFee` without corresponding quote tokens having left the pool, which causes `adjustedTarget` in `BELOW_ONE` to over-calculate `state.Q0`.
- Similarly, when `quoteInput > backToOnePayQuote` (`BELOW_ONE -> ABOVE_ONE`), `_ROneSellQuoteToken` computes the `ONE -> ABOVE_ONE` leg using the unadjusted `state.B0` rather than accounting for the LP fee retained on the `BELOW_ONE -> ONE` leg, and `_BASE_TARGET_` in storage remains stale.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with a non-zero `_LP_FEE_RATE_` (where the LP fee portion `lpFeeRate > 0`).
2. The pool is currently in `_RState_ == uint32(PMMPricing.RState.BELOW_ONE)` with `_BASE_RESERVE_ > _BASE_TARGET_` (e.g., after any `sellBase` swap).
3. A user executes `sellQuote` with `quoteInput == state.Q0 - state.Q` (`backToOnePayQuote`) or crosses equilibrium.

### Proof of concept

1. Assume a `MagicLP` pool is initialized with `B = 1000e18`, `Q = 1000e18`, `B0 = 1000e18`, `Q0 = 1000e18`, `i = 1e18`, `K = 0.01e18`, `lpFeeRate = 0.001e18` (0.1%), and `mtFeeRate = 0`.
2. User A sells `100e18` base tokens via `sellBase`. The pool transitions to `_RState_ = BELOW_ONE` with `_BASE_TARGET_ = 1000e18` and `_BASE_RESERVE_ = 1100e18`.
3. `getPMMState()` computes `state.Q0` via `adjustedTarget`, and `backToOnePayQuote = state.Q0 - _QUOTE_RESERVE_`.
4. User B transfers `backToOnePayQuote` quote tokens to the pool and calls `sellQuote(userB)`:
   - `PMMPricing.sellQuoteToken` returns gross `receiveBaseAmount = 1100e18 - 1000e18 = 100e18` and `newR = RState.ONE`.
   - `querySellQuote` deducts `0.1e18` base tokens as the LP fee, returning `receiveBaseAmount = 99.9e18`, `mtFee = 0`, `newRState = RState.ONE`, `newQuoteTarget = state.Q0`.
   - `sellQuote` transfers `99.9e18` base tokens to User B, sets `_QUOTE_TARGET_ = state.Q0` and `_RState_ = RState.ONE`, and sets `_BASE_RESERVE_ = 1000.1e18` and `_QUOTE_RESERVE_ = state.Q0`.
5. `_BASE_TARGET_` remains `1000e18` while `_BASE_RESERVE_` is `1000.1e18` in `RState.ONE`.
6. Repeating `sellBase` and `sellQuote` back to `RState.ONE` continually accumulates retained base LP fees in `_BASE_RESERVE_` without updating `_BASE_TARGET_`, progressively diverging `_BASE_RESERVE_` from `_BASE_TARGET_`, distorting `getMidPrice()`, `_twapUpdate()`, and `state.Q0` in subsequent `BELOW_ONE` states.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: Reserve-target invariant violation (`_BASE_RESERVE_ > _BASE_TARGET_` at `RState.ONE`), inaccurate TWAP and `getMidPrice()` values, and miscalculated target `state.Q0` and trade outputs on subsequent swaps.
- **Business Impact**: Pricing inaccuracy for traders, LPs, and downstream consumers relying on `getMidPrice()`, TWAP, or pool targets.

### Remediation

When `sellQuote` transitions `_RState_` to `PMMPricing.RState.ONE`, update `_BASE_TARGET_` to match the new base reserve (which includes the retained base-token LP fee):

```solidity
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));

        // update TARGET
        if (_RState_ != uint32(newRState)) {
            if (newRState == PMMPricing.RState.ONE) {
                _BASE_TARGET_ = baseBalance.toUint112();
            }
            _QUOTE_TARGET_ = newQuoteTarget.toUint112();
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }

        _setReserve(baseBalance, quoteBalance);
```

## Finding 9: Unchecked Timestamp Rollover Missing in `MagicLP._twapUpdate` Permanently Bricks Pool Operations

### Synopsis

In `MagicLP._twapUpdate`, `blockTimestamp` is computed modulo `2 ** 32`, but the subtraction `uint32 timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_` is performed outside an `unchecked` block. When `block.timestamp` rolls over `2 ** 32`, this subtraction underflows and reverts, permanently bricking all pool operations, swaps, and withdrawals.

### Technical details

`MagicLP` is a Solidity 0.8 port of DODO V2's `DODOStablePool`. In `_twapUpdate` (`src/mimswap/MagicLP.sol`), the contract tracks the 32-bit truncated timestamp and cumulative price following the Uniswap V2 / DODO V2 TWAP pattern:

```solidity
function _twapUpdate() internal {
    uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
    uint32 timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;

    if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
        /// @dev It is desired and expected for this value to
        /// overflow once it has hit the max of `type.uint256`.
        unchecked {
            _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
        }
    }

    _BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
}
```

At line 546, `block.timestamp` is explicitly reduced modulo `2 ** 32` into a `uint32`. In the original Solidity 0.6 implementation, `blockTimestamp - _BLOCK_TIMESTAMP_LAST_` relied on modular `uint32` wraparound semantics when `block.timestamp` crosses a multiple of `2 ** 32` so that `timeElapsed` still equals the true elapsed time.

When porting the contract to Solidity 0.8, the developers wrapped the cumulative price update at line 553 in an `unchecked` block to preserve wraparound behavior, but left line 547 (`uint32 timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;`) under Solidity 0.8's default checked arithmetic.

Once `block.timestamp` reaches `2 ** 32` (4,294,967,296) and rolls over in `uint32(block.timestamp % 2 ** 32)`, `blockTimestamp` becomes smaller than `_BLOCK_TIMESTAMP_LAST_` (which holds a value prior to the rollover). Consequently, `blockTimestamp - _BLOCK_TIMESTAMP_LAST_` reverts with `Panic(0x11)` (arithmetic underflow).

Because `_twapUpdate()` is invoked unconditionally at the end of `_setReserve()`, `_sync()`, `_resetTargetAndReserve()`, and `ratioSync()`, every function that updates reserves or synchronizes balances — including `buyShares()`, `sellShares()`, `sellBase()`, `sellQuote()`, `flashLoan()`, `sync()`, `setParameters()`, and `ratioSync()` — will permanently revert.

### Assumptions and preconditions

1. A `MagicLP` pool has been initialized (`init()` sets `_BLOCK_TIMESTAMP_LAST_ = uint32(block.timestamp % 2 ** 32)`) or interacted with prior to `block.timestamp = 2 ** 32` such that `_BLOCK_TIMESTAMP_LAST_ > 0`.
2. `block.timestamp` advances past `2 ** 32` such that `uint32(block.timestamp % 2 ** 32) < _BLOCK_TIMESTAMP_LAST_`.

### Proof of concept

1. A `MagicLP` clone is initialized and liquidity is deposited via `buyShares()` at timestamp `t_1 = 2 ** 32 - 10` (`_BLOCK_TIMESTAMP_LAST_ = 4294967286`).
2. Time advances by 20 seconds to `t_2 = 2 ** 32 + 10`.
3. A liquidity provider calls `sellShares()` to withdraw their underlying base and quote tokens (or a trader calls `sellBase()` / `sellQuote()`).
4. `sellShares()` calls `_sync()`, which calls `_twapUpdate()`.
5. In `_twapUpdate()`, `blockTimestamp = uint32((2 ** 32 + 10) % 2 ** 32) = 10`.
6. Line 547 evaluates `uint32 timeElapsed = 10 - 4294967286` under Solidity 0.8 checked arithmetic, which underflows and reverts the transaction.
7. Because no function can update `_BLOCK_TIMESTAMP_LAST_` without executing `_twapUpdate()` (until `uint32(block.timestamp % 2 ** 32)` catches up to `_BLOCK_TIMESTAMP_LAST_` at the end of the next `2 ** 32`-second cycle), all pool operations and LP withdrawals are frozen.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** `_twapUpdate()` reverts with an arithmetic underflow whenever `blockTimestamp < _BLOCK_TIMESTAMP_LAST_` after the `2 ** 32` rollover, causing a denial of service across `_setReserve()`, `_sync()`, `_resetTargetAndReserve()`, and `ratioSync()`.
- **Business Implications:** All trading (`sellBase`, `sellQuote`, `flashLoan`), liquidity provisioning (`buyShares`), liquidity withdrawals (`sellShares`), and administrative pool adjustments (`setParameters`, `ratioSync`) are permanently blocked, freezing all user funds in the pool.

### Remediation

Compute `timeElapsed` inside an `unchecked` block so that `uint32` subtraction wraps around cleanly across the `2 ** 32` boundary, matching the intended modular timestamp design.

```solidity
    function _twapUpdate() internal {
        uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
        uint32 timeElapsed;
        unchecked {
            timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;
        }

        if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
            /// @dev It is desired and expected for this value to
            /// overflow once it has hit the max of `type.uint256`.
            unchecked {
                _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
            }
        }

        _BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
    }
```

## Finding 10: Premature State and Reserve Updates Before `_twapUpdate` in `MagicLP` Allow Retroactive TWAP Manipulation

### Synopsis

In `MagicLP`, `_setReserve` (as well as `_sync`, `_resetTargetAndReserve`, and `ratioSync`) updates the pool's reserves and PMM state variables before calling `_twapUpdate()`. Consequently, `_twapUpdate()` multiplies the new post-trade `getMidPrice()` by the historical `timeElapsed`, allowing an attacker to retroactively manipulate the cumulative TWAP price within a single transaction.

### Technical details

Time-Weighted Average Price (TWAP) accumulators must attribute the elapsed time since the last observation (`timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_`) to the price that was in effect *during* that interval—namely, the price prior to the current state-changing operation.

In `MagicLP`, `_twapUpdate()` computes the cumulative price update as:

```solidity
if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
    unchecked {
        _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
    }
}
_BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
```

Here, `getMidPrice()` calls `getPMMState()`, which derives the current mid price from `_I_`, `_K_`, `_BASE_RESERVE_`, `_QUOTE_RESERVE_`, `_BASE_TARGET_`, `_QUOTE_TARGET_`, and `_RState_`.

However, in `_setReserve()` (lines 560–565) and `_sync()` (lines 567–579), `_BASE_RESERVE_` and `_QUOTE_RESERVE_` are overwritten with the new post-operation balances *before* `_twapUpdate()` is called:

```solidity
function _setReserve(uint256 baseReserve, uint256 quoteReserve) internal {
    _BASE_RESERVE_ = baseReserve.toUint112();
    _QUOTE_RESERVE_ = quoteReserve.toUint112();

    _twapUpdate();
}
```

Furthermore, in all callers that invoke `_setReserve()` or `_sync()` (such as `sellBase`, `sellQuote`, `flashLoan`, `buyShares`, `sellShares`, `setParameters`, and `ratioSync`), the target and R-state variables (`_BASE_TARGET_`, `_QUOTE_TARGET_`, `_RState_`) are also mutated prior to calling `_setReserve()` or `_sync()`.

Because `_twapUpdate()` is executed after the reserves and PMM state are updated:
1. The first transaction in a block (where `timeElapsed > 0`) records `getMidPrice() * timeElapsed` using the **new** post-transaction price rather than the historical price that prevailed over `timeElapsed`.
2. Any subsequent transaction in the same block sees `timeElapsed == 0` (since `_BLOCK_TIMESTAMP_LAST_` was already updated to `blockTimestamp`), so restoring the pool's reserves in the same block or transaction does not undo the corrupted accumulator delta.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized and has non-zero reserves (`_BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0`).
2. At least 1 second has elapsed since the pool's last state update (`uint32(block.timestamp % 2 ** 32) > _BLOCK_TIMESTAMP_LAST_`).
3. An external contract, user, or integration relies on `_BASE_PRICE_CUMULATIVE_LAST_` to compute a TWAP for the pool.

### Proof of concept

1. Assume a `MagicLP` pool has been idle for `timeElapsed = 3600` seconds (1 hour) with a stable mid price $P_0 = 1\text{e}18$. During this entire hour, the true price of the pool was $P_0$.
2. At `block.timestamp`, before any other interaction with the pool in the current block, an actor executes a large swap (e.g., via `sellBase` or `sellQuote`, optionally funded with a flash loan) that moves the pool's state so that the post-swap `getMidPrice()` is $P_1 = 5\text{e}18$.
3. Inside `sellBase` / `sellQuote`, the contract updates `_BASE_TARGET_` / `_QUOTE_TARGET_` and `_RState_`, and calls `_setReserve()`, which updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to the post-swap balances and then calls `_twapUpdate()`.
4. `_twapUpdate()` adds $P_1 \times 3600$ to `_BASE_PRICE_CUMULATIVE_LAST_` instead of $P_0 \times 3600$, and sets `_BLOCK_TIMESTAMP_LAST_ = blockTimestamp`.
5. Immediately in the same transaction, the actor swaps back to restore the pool reserves and price to $P_0$. During this second swap, `timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_ = 0`, so `_BASE_PRICE_CUMULATIVE_LAST_` is not updated.
6. The actor only pays the pool's swap fees, while the TWAP accumulator permanently records a price of $P_1$ for the entire preceding 1-hour interval.

### Impact assessment

- **Severity:** High / Medium (depending on downstream TWAP consumers).
- **Technical Impact:** Complete loss of TWAP oracle resistance to single-transaction / flash-loan price manipulation. The entire elapsed time since the previous interaction is weighted by the post-trade spot price rather than the pre-trade price.
- **Business Impact:** Any protocol or integration consuming `_BASE_PRICE_CUMULATIVE_LAST_` as a TWAP oracle can be manipulated atomically within a single transaction with zero market duration risk, potentially leading to bad debt, unfair liquidations, or mispriced asset operations.

### Remediation

Update `_twapUpdate()` **before** modifying `_BASE_RESERVE_`, `_QUOTE_RESERVE_`, `_BASE_TARGET_`, `_QUOTE_TARGET_`, `_RState_`, `_I_`, or `_K_` in all state-changing functions, or pass the pre-mutation `PMMState` (or pre-mutation `midPrice`) into `_twapUpdate()` / `_setReserve()` / `_sync()`.

Specifically, if `_twapUpdate()` is called at the beginning of state-changing operations (prior to updating targets, R-state, and reserves) or if `_setReserve` / `_sync` receive the pre-trade mid price, the accumulator will properly weight the historical price over `timeElapsed`:

```solidity
    function _twapUpdate() internal {
        uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
        uint32 timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;

        if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
            /// @dev It is desired and expected for this value to
            /// overflow once it has hit the max of `type.uint256`.
            unchecked {
                _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
            }
        }

        _BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
    }

    function _setReserve(uint256 baseReserve, uint256 quoteReserve) internal {
        _BASE_RESERVE_ = baseReserve.toUint112();
        _QUOTE_RESERVE_ = quoteReserve.toUint112();
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
    }
```

*(Ensure `_twapUpdate()` is invoked at the start of `sellBase`, `sellQuote`, `flashLoan`, `buyShares`, `sellShares`, `setParameters`, `ratioSync`, and `sync` before any reserves, targets, or PMM parameters are modified.)*

## Finding 11: Unhandled `getMidPrice()` Revert in `MagicLP._twapUpdate` Propagates Pricing Failures Into Reserve Updates

### Synopsis

In `MagicLP._twapUpdate`, `getMidPrice()` is called unconditionally whenever `timeElapsed > 0` and reserves are non-zero. Because `getMidPrice()` can revert under extreme reserve, target, or price states, and `_twapUpdate()` is invoked by `_setReserve()` and `_sync()`, pricing arithmetic failures propagate into and block core pool operations and withdrawals.

### Technical details

In `src/mimswap/MagicLP.sol`, every reserve-updating operation—including `sellBase`, `sellQuote`, `buyShares` (via `_setReserve` at lines 560–565), `sellShares`, `flashLoan`, `sync` (via `_sync` at lines 567–579), `setParameters` (via `_resetTargetAndReserve` at line 542), and `ratioSync` (line 521)—calls `_twapUpdate()`:

```solidity
    function _twapUpdate() internal {
        uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
        uint32 timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;

        if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
            /// @dev It is desired and expected for this value to
            /// overflow once it has hit the max of `type.uint256`.
            unchecked {
                _BASE_PRICE_CUMULATIVE_LAST_ += getMidPrice() * timeElapsed;
            }
        }

        _BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
    }
```

Although `getMidPrice() * timeElapsed` is placed inside an `unchecked` block on line 553, Solidity's `unchecked` semantics do not propagate into internal or external function calls. Inside `getMidPrice()` (line 215), `getPMMState()` calls `PMMPricing.adjustedTarget(state)` and then `PMMPricing.getMidPrice(state)` using checked Solidity 0.8 arithmetic:

1. **Pre-trade vs. post-trade and `timeElapsed == 0` bypass:**
   - When `timeElapsed == 0` (i.e., after an initial state update in the same block), `_twapUpdate()` skips calling `getMidPrice()`.
   - Furthermore, `buyShares()` and `sync()` do not call `getPMMState()` or `getMidPrice()` during their own logic, and `sellBase()` / `sellQuote()` only evaluate `getPMMState()` on the *pre-trade* reserves before updating `_BASE_RESERVE_` and `_QUOTE_RESERVE_` via `_setReserve()`.

2. **Extreme-state pricing reverts inside `getMidPrice()`:**
   Once `_BASE_RESERVE_` and `_QUOTE_RESERVE_` are updated, `getMidPrice()` can revert on the new reserve state in several ways:
   - **Target squaring and ratio overflow in `PMMPricing.getMidPrice`:**
     `PMMPricing.getMidPrice` computes `DecimalMath.divFloor((state.Q0 * state.Q0) / state.Q, state.Q)` (when `R == BELOW_ONE`) or `DecimalMath.divFloor((state.B0 * state.B0) / state.B, state.B)` (otherwise), followed by `DecimalMath.mulFloor(state.K, R)` and `DecimalMath.divFloor(state.i, R)` or `DecimalMath.mulFloor(state.i, R)`. When `state.Q` or `state.B` becomes very small relative to `state.Q0` or `state.B0` (for example, after a large trade in a low-`K` or `K = 0` pool, or after a large imbalance), or when `state.Q0` / `state.B0` computed in `adjustedTarget` exceeds `type(uint128).max` (which occurs for large `i` up to `MAX_I = 10**36` or small `i` where `reciprocalFloor(i)` is large), `state.Q0 * state.Q0`, `state.B0 * state.B0`, `((B0 * B0) / B) * 1e18`, `state.K * R`, or `state.i * R` overflows `uint256` and reverts.
   - **Checked overflow in `Math._SolveQuadraticFunctionForTarget`:**
     Called by `getPMMState() -> PMMPricing.adjustedTarget(state)`, line 96 checks `else if ((ki * delta) / ki == delta)` outside `unchecked`. Because `ki = 4 * k * i` can reach `4 * 10**54` and `delta` (`state.B - state.B0` or `state.Q - state.Q0`) can reach `type(uint112).max`, `ki * delta` reverts with an arithmetic overflow panic in Solidity 0.8 instead of falling back to the `else` branch `((ki / V1) * delta)`.
   - **Division by zero when `R == 0`:**
     When `R` rounds down to `0` (for example, in `BELOW_ONE` if `state.K == MAX_K` and `(state.Q0 * state.Q0) / state.Q` is small relative to `state.Q`), `DecimalMath.divFloor(state.i, R)` reverts due to division by zero.
   - **Target underflow before `correctRState()`:**
     If `_BASE_RESERVE_ < _BASE_TARGET_` while `_RState_ == BELOW_ONE` or `_QUOTE_RESERVE_ < _QUOTE_TARGET_` while `_RState_ == ABOVE_ONE` (the exact states `correctRState()` on lines 138–149 is designed to fix), `state.B - state.B0` or `state.Q - state.Q0` in `adjustedTarget` underflows and reverts.

Because `_twapUpdate()` does not guard against reverts from `getMidPrice()`, any pricing failure on the updated reserves causes `_setReserve()` and `_sync()` to revert whenever `timeElapsed > 0`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized and has non-zero reserves (`_BASE_RESERVE_ != 0` and `_QUOTE_RESERVE_ != 0`).
2. The pool reaches an extreme reserve or target configuration (either through trading depleting one side of a low-`K` pool, high/low oracle price constant `_I_`, or token transfers synced in a block where `timeElapsed == 0` so `_twapUpdate()` skips `getMidPrice()`).
3. At least one second elapses (`block.timestamp > _BLOCK_TIMESTAMP_LAST_`, so `timeElapsed > 0`).

### Proof of concept

1. Consider a `MagicLP` pool initialized with valid parameters (for example, `_I_ = 1e30` and `_K_ = 1e18`, or `_I_ = 1e18` and `_K_ = 0`).
2. In block `T`, any interaction (such as `sync()`) sets `_BLOCK_TIMESTAMP_LAST_ = T`.
3. In the same block `T`, `timeElapsed` is `0`. A user executes a large trade or transfers tokens and calls `sync()` or `buyShares()`:
   - For a pool with `_I_ = 1e30` in `RState.BELOW_ONE` where `delta = _BASE_RESERVE_ - _BASE_TARGET_ = 1e24`, `_setReserve()` updates the reserves and calls `_twapUpdate()`. Because `timeElapsed == 0`, `getMidPrice()` is not executed and the transaction succeeds.
   - Alternatively, in a pool with `_K_ = 0` (or very small `_K_`), a trader swaps base tokens for almost all quote tokens in the pool (leaving `_QUOTE_RESERVE_ = 1` wei) while `_I_ = 1e30`. `querySellBase` uses the *pre-trade* `state.Q`, which succeeds, and when `timeElapsed == 0`, `_setReserve()` completes without calling `getMidPrice()` on the new `_QUOTE_RESERVE_ = 1`.
4. In the next block at timestamp `T + 1` (`timeElapsed = 1 > 0`), a liquidity provider calls `sellShares()` to withdraw their liquidity (or a user calls `sellBase()`, `sellQuote()`, `buyShares()`, or `sync()`).
5. `_sync()` / `_setReserve()` invokes `_twapUpdate()`. Since `timeElapsed > 0` and both reserves are non-zero, `_twapUpdate()` calls `getMidPrice()`.
6. Inside `getMidPrice()`, `getPMMState()` calls `adjustedTarget(state)` where `ki * delta` overflows `uint256` in `_SolveQuadraticFunctionForTarget` (or `state.Q0 * state.Q0` / `DecimalMath.mulFloor(state.i, R)` overflows `uint256` in `PMMPricing.getMidPrice`).
7. The revert inside `getMidPrice()` propagates out of `_twapUpdate()` and reverts `sellShares()` and all other pool operations when `timeElapsed > 0`.

### Impact assessment

- **Severity:** Medium / High
- **Technical Effect:** Arithmetic reverts inside `getMidPrice()` during `_twapUpdate()` cause `_setReserve()` and `_sync()` to revert whenever `timeElapsed > 0`.
- **Business Impact:** All pool state transitions that update reserves—including swaps (`sellBase`, `sellQuote`, `flashLoan`), liquidity provisioning (`buyShares`), liquidity withdrawals (`sellShares`), and reserve synchronization (`sync`, `ratioSync`, `setParameters`)—fail in any transaction that executes as the first update in a new timestamp (`timeElapsed > 0`), freezing normal user and router interactions unless a workaround transaction first updates `_BLOCK_TIMESTAMP_LAST_` or restores the pool state.

### Remediation

Prevent pricing calculation failures in `_twapUpdate()` from reverting reserve updates by wrapping the mid-price retrieval in a safe try/catch call (or computing the mid-price with overflow-safe bounds checking) so that `_BLOCK_TIMESTAMP_LAST_` and pool reserves can always be updated even when the pool is in an extreme state:

```solidity
    function _twapUpdate() internal {
        uint32 blockTimestamp = uint32(block.timestamp % 2 ** 32);
        uint32 timeElapsed;
        unchecked {
            timeElapsed = blockTimestamp - _BLOCK_TIMESTAMP_LAST_;
        }

        if (timeElapsed > 0 && _BASE_RESERVE_ != 0 && _QUOTE_RESERVE_ != 0) {
            try this.getMidPrice() returns (uint256 midPrice) {
                /// @dev It is desired and expected for this value to
                /// overflow once it has hit the max of `type.uint256`.
                unchecked {
                    _BASE_PRICE_CUMULATIVE_LAST_ += midPrice * timeElapsed;
                }
            } catch {}
        }

        _BLOCK_TIMESTAMP_LAST_ = blockTimestamp;
    }
```

## Finding 12: Precision Loss in `PMMPricing.adjustedTarget` Reciprocal Price Calculation and Nonlinear Target Solver Underestimates `state.B0`

### Synopsis

In `PMMPricing.adjustedTarget`, when `state.R == RState.ABOVE_ONE`, `state.B0` is recomputed using `DecimalMath.reciprocalFloor(state.i)` inside `Math._SolveQuadraticFunctionForTarget`. Truncation in the reciprocal and subsequent nonlinear square-root and division steps systematically underestimates `state.B0`, mispricing swaps and permanently eroding `_BASE_TARGET_`.

### Technical details

When a `MagicLP` pool is in `RState.ABOVE_ONE` (i.e., quote reserves exceed target `Q > Q0` and base reserves are below target `B < B0`), `MagicLP.getPMMState()` invokes `PMMPricing.adjustedTarget(state)` (lines 194–200 of `src/mimswap/libraries/PMMPricing.sol`) to dynamically recompute the effective base target `state.B0`:

```solidity
state.B0 = Math._SolveQuadraticFunctionForTarget(
    state.B,
    state.Q - state.Q0,
    DecimalMath.reciprocalFloor(state.i),
    state.K
);
```

This calculation suffers from severe fixed-point precision loss and rounding-error amplification across two stages:

1. **Precision Loss in `DecimalMath.reciprocalFloor(state.i)` (Line 197):**
   `DecimalMath.reciprocalFloor(state.i)` computes `(10 ** 36) / state.i`. In `MagicLP`, `state.i` is the unit price of the base token denominated in the quote token, scaled by `10 ** (18 + quoteDecimals - baseDecimals)`, and is valid up to `MAX_I = 10 ** 36`.
   - Whenever `state.i` does not evenly divide `10 ** 36`, integer division truncates the reciprocal downward.
   - When `state.i > 10 ** 18`—such as when the base token has fewer decimals than the quote token (e.g., a 6-decimal USDC/USDT base token paired with an 18-decimal MIM/USDB quote token where `state.i = 10 ** 30`, or an 8-decimal WBTC base token where `state.i ~ 6 * 10 ** 32`)—`DecimalMath.reciprocalFloor(state.i)` retains only `10 ** 36 / state.i` significant units (e.g., `10 ** 6` for `state.i = 10 ** 30`, or `1` at `state.i = 10 ** 36`), losing up to 18 orders of magnitude of precision.

2. **Rounding-Error Amplification in `Math._SolveQuadraticFunctionForTarget` (Lines 194–199):**
   `Math._SolveQuadraticFunctionForTarget(V1, delta, i_recip, k)` computes:
   $$\text{ki} = 4 \cdot k \cdot i_{\text{recip}}, \quad \text{sqrt} = \left\lfloor \sqrt{\left\lfloor \frac{\text{ki} \cdot \Delta Q}{B} \right\rfloor + 10^{36}} \right\rfloor$$
   $$\text{premium} = \left\lfloor \frac{(\text{sqrt} - 10^{18}) \cdot 10^{18}}{2 \cdot k} \right\rfloor + 10^{18}, \quad B_0 = \left\lfloor \frac{B \cdot \text{premium}}{10^{18}} \right\rfloor$$
   - Because `k` can be as small as `10 ** 14` (e.g., `0.0001e18` in deployment tests), the term `(ki * delta) / B` is much smaller than `10 ** 36`. Adding `10 ** 36` and taking the integer square root compresses the signal into the low bits of `_sqrt - 10 ** 18`.
   - Subsequently, `DecimalMath.divFloor(_sqrt - 10 ** 18, 2 * k)` multiplies any truncation error in `_sqrt` by `10 ** 18 / (2 * k)` (up to `5,000x` at `k = 10 ** 14`).
   - In particular, whenever `(4 * k * i_recip * (Q - Q0)) / B < 2 * 10 ** 18 + 1`, the integer square root rounds down to `10 ** 18`, causing `premium` to evaluate to `10 ** 18` and `state.B0` to evaluate to `state.B` (`state.B0 - state.B == 0`) even though `state.Q > state.Q0`.

In `PMMPricing.sellBaseToken`, when `state.R == RState.ABOVE_ONE`, the amount of base tokens required to return the pool to equilibrium (`RState.ONE`) is `backToOnePayBase = state.B0 - state.B`, which yields `backToOneReceiveQuote = state.Q - state.Q0`. Because `state.B0` is underestimated, `backToOnePayBase` is smaller than the true base deficit, allowing a trader calling `sellBase` to extract `state.Q - state.Q0` quote tokens while paying less base token than required, and causing `MagicLP.sellBase` (line 257) to overwrite `_BASE_TARGET_` with the underestimated `newBaseTarget = state.B0` upon transitioning to `RState.BELOW_ONE`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with valid parameters (`0 < _I_ <= 10 ** 36` and `0 <= _K_ <= 10 ** 18`), particularly where `_I_` is large (e.g., low-decimal base token paired with an 18-decimal quote token) or `_K_` is small.
2. The pool enters `RState.ABOVE_ONE` via a quote-to-base swap (`sellQuote` or `flashLoan`).

### Proof of concept

Consider a `MagicLP` pool for a 6-decimal base token (e.g., USDC) and an 18-decimal quote token (e.g., USDB/MIM) with parameters:
- `_I_ = 1.5 * 10 ** 30` (1 Base = 1.5 Quote)
- `_K_ = 10 ** 14` (`0.0001e18`, matching the default `K` in `MIMSwap.t.sol`)
- Initial reserves at `RState.ONE`: `_BASE_RESERVE_ = _BASE_TARGET_ = 1,000,000e6` (`10 ** 12`), `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1,500,000e18` (`1.5 * 10 ** 24`).

1. **Transition to `RState.ABOVE_ONE`:**
   A swap or sequence of swaps moves the pool to `RState.ABOVE_ONE` with `Q - Q0 = 0.003e18` (`3 * 10 ** 15` quote wei) and `B = 10 ** 12 - 2000` (`999,999.998e6` base wei, i.e., `2000` base wei below the initial `_BASE_TARGET_`).

2. **Target Recomputation in `adjustedTarget`:**
   When `getPMMState()` is called:
   - `DecimalMath.reciprocalFloor(1.5 * 10 ** 30) = (10 ** 36) / (1.5 * 10 ** 30) = 666,666` (truncating the repeating `.666...` fraction).
   - In `Math._SolveQuadraticFunctionForTarget`:
     - `ki = 4 * (10 ** 14) * 666,666 = 2.666664 * 10 ** 20`.
     - `(ki * delta) / V1 = (2.666664 * 10 ** 20 * 3 * 10 ** 15) / (10 ** 12 - 2000) = 7.999992015999984 * 10 ** 23`.
     - Inside `sqrt`: `10 ** 36 + 7.999992015999984 * 10 ** 23`. Because `(10 ** 18 + 399,999) ** 2 < 10 ** 36 + 7.999992015999984 * 10 ** 23 < (10 ** 18 + 400,000) ** 2`, `_sqrt` floors to `10 ** 18 + 399,999`.
     - `premium = DecimalMath.divFloor(399,999, 2 * 10 ** 14) + 10 ** 18 = 1,999,995,000 + 10 ** 18`.
     - `state.B0 = DecimalMath.mulFloor(10 ** 12 - 2000, 10 ** 18 + 1,999,995,000) = (10 ** 12 - 2000) + 1999 = 10 ** 12 - 1`.
   - Even more starkly, if `Q - Q0 = 7.5 * 10 ** 9` quote wei (worth `5` base wei), `(ki * delta) / V1 = 1.999998 * 10 ** 18 < 2 * 10 ** 18 + 1`, so `_sqrt == 10 ** 18`, `premium == 10 ** 18`, and `state.B0 == state.B` (`backToOnePayBase == 0`).

3. **Exploitation via `sellBase`:**
   When a caller subsequently invokes `sellBase` with `baseInput = 1` wei while `backToOnePayBase == 0`, `sellBaseToken` enters the `payBaseAmount > backToOnePayBase` branch, awarding the full `backToOneReceiveQuote = state.Q - state.Q0` plus `_ROneSellBaseToken(state, 1)` and updating `_BASE_TARGET_` in storage to the reduced `state.B0`, permanently shrinking the pool's base target.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:** `state.B0` is systematically underestimated whenever `state.R == RState.ABOVE_ONE`. This causes `sellBaseToken` to undercharge base tokens for returning the pool to `RState.ONE`, distorts `getMidPrice()` in the `RState.ABOVE_ONE` branch (which depends on `(state.B0 * state.B0) / state.B`), and permanently reduces `_BASE_TARGET_` when `sellBase` transitions to `RState.BELOW_ONE`.
- **Business Implications:** Liquidity providers suffer gradual value leakage across repeated swaps around `RState.ONE` / `RState.ABOVE_ONE`, especially in pairs where `state.i > 10 ** 18` or `state.K` is small.

### Remediation

Avoid pre-truncating `1 / state.i` via `DecimalMath.reciprocalFloor(state.i)` before solving the quadratic target equation. Instead, pass `state.i` directly to a dedicated target solver (or use full 512-bit precision via `Math.mulDiv`) that divides by `state.i` only after multiplying by `delta`, and compute the quadratic root without intermediate `1e18` truncation:

```solidity
        } else if (state.R == RState.ABOVE_ONE) {
            state.B0 = Math._SolveQuadraticFunctionForTargetReciprocal(
                state.B,
                state.Q - state.Q0,
                state.i,
                state.K
            );
        }
```

Where `Math._SolveQuadraticFunctionForTargetReciprocal(uint256 V1, uint256 delta, uint256 i, uint256 k)` computes `delta / i` integrated with `4 * k` using high-precision arithmetic (`(4 * k * delta * DecimalMath.ONE2) / (i * V1)`) before taking the square root.

## Finding 13: Division Before Multiplication in `PMMPricing.getMidPrice` Causes Precision Loss and Potential Division-by-Zero

### Synopsis

In `PMMPricing.getMidPrice`, the quadratic reserve ratios `(state.Q0 * state.Q0) / state.Q` and `(state.B0 * state.B0) / state.B` perform unscaled integer division before `DecimalMath.divFloor` scales the numerator by `1e18`, causing severe precision loss in `R` and the TWAP oracle, or a division-by-zero revert when `R` rounds to zero.

### Technical details

In `src/mimswap/libraries/PMMPricing.sol`, `getMidPrice` computes the PMM pricing multiplier $R = (1 - k) + k \cdot (Q_0 / Q)^2$ (when `state.R == RState.BELOW_ONE`) or $R = (1 - k) + k \cdot (B_0 / B)^2$ (when `state.R != RState.BELOW_ONE`):

```solidity
    function getMidPrice(PMMState memory state) internal pure returns (uint256) {
        if (state.R == RState.BELOW_ONE) {
            uint256 R = DecimalMath.divFloor((state.Q0 * state.Q0) / state.Q, state.Q);
            R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
            return DecimalMath.divFloor(state.i, R);
        } else {
            uint256 R = DecimalMath.divFloor((state.B0 * state.B0) / state.B, state.B);
            R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
            return DecimalMath.mulFloor(state.i, R);
        }
    }
```

`DecimalMath.divFloor(target, d)` is defined in `src/mimswap/libraries/DecimalMath.sol` as `(target * ONE) / d` where `ONE = 10 ** 18`. Because lines 205 and 209 evaluate `(state.Q0 * state.Q0) / state.Q` and `(state.B0 * state.B0) / state.B` prior to calling `DecimalMath.divFloor(..., state.Q)` and `DecimalMath.divFloor(..., state.B)`, the first division is executed in raw token units without `1e18` fixed-point scaling:

$$R_{\text{raw}} = \left\lfloor \frac{\lfloor V_0^2 / V \rfloor \cdot 10^{18}}{V} \right\rfloor$$

This division-before-multiplication flaw has two direct consequences:

1. **Precision Loss in `R` and TWAP (`_BASE_PRICE_CUMULATIVE_LAST_`)**: Whenever reserves `state.Q` or `state.B` are small (such as pools with low-decimal tokens, pools initialized near the minimum share threshold of `2001` wei in `MagicLP.buyShares`, or pools after liquidity withdrawals in `MagicLP.sellShares`), the unscaled integer quotient `(V0 * V0) / V` truncates the fractional component before scaling by `1e18`. Since `MagicLP._twapUpdate()` accumulates `getMidPrice() * timeElapsed` on every state-modifying operation, this rounding error permanently distorts the pool's mid-price and TWAP accumulator.
2. **Truncation to Zero and Division-by-Zero DoS**: When `state.R == RState.ONE`, `PMMPricing.adjustedTarget` does not recompute `state.B0`. If `state.B > state.B0` (for example, after an unbalanced initial liquidity deposit in `MagicLP.buyShares` where `quoteBalance < DecimalMath.mulFloor(baseBalance, _I_)` sets `_BASE_TARGET_ = shares < baseBalance`, or after a direct token transfer followed by `sync()`), `state.B` can exceed `state.B0 * state.B0`, causing `(state.B0 * state.B0) / state.B` on line 209 to truncate to `0`. Similarly, if `(state.Q0 * state.Q0) / state.Q` on line 205 truncates to `0` when `state.K == DecimalMath.ONE` (`1e18`), `R` evaluates to `0`, and `DecimalMath.divFloor(state.i, R)` on line 207 reverts with a division-by-zero panic, blocking `_twapUpdate()` and all core pool operations (`sellBase`, `sellQuote`, `buyShares`, `sellShares`, `flashLoan`, and `sync`).

### Assumptions and preconditions

- A `MagicLP` pool is initialized and holds non-zero base and quote reserves.
- The pool either operates with small token reserve quantities (e.g., low-decimal tokens or low liquidity near the minimum mint threshold) or has `state.B > state.B0` while in `RState.ONE` (e.g., via an unbalanced deposit in `buyShares` or token donation followed by `sync()`).

### Proof of concept

1. Initialize a `MagicLP` pool with `_I_ = 1e18` and `_K_ = 1e18` (or `5e14`).
2. Mint initial liquidity via `buyShares` with `baseBalance = 3000` wei and `quoteBalance = 3000` wei (satisfying `shares = 3000 > 2001`), which sets `_BASE_TARGET_ = 3000`, `_QUOTE_TARGET_ = 3000`, `_BASE_RESERVE_ = 3000`, `_QUOTE_RESERVE_ = 3000`, and `_RState_ = 0` (`RState.ONE`).
3. Swap quote tokens via `sellQuote` or base tokens via `sellBase` so that `state.B = 2000` and `state.B0 = 2999` (with `state.R == RState.ABOVE_ONE`).
4. Call `getMidPrice()`:
   - On line 209, `(state.B0 * state.B0) / state.B` computes `(2999 * 2999) / 2000 = 4497` (truncating `0.0005` in unscaled units).
   - `DecimalMath.divFloor(4497, 2000)` returns `2_248_500_000_000_000_000` (`2.2485e18`), whereas the exact fixed-point value of $(2999 / 2000)^2 \times 10^{18}$ is `2_248_500_250_000_000_000`, losing 14 digits of precision.
5. Alternatively, while in `RState.ONE` with `_BASE_TARGET_ = 3000`, if `baseBalance` is increased to `9_000_001` wei (either during initial `buyShares` with `baseBalance = 9_000_001` and `quoteBalance = 3000`, or via direct transfer and `sync()`), `(state.B0 * state.B0) / state.B` on line 209 evaluates to `(3000 * 3000) / 9_000_001 = 0`, collapsing the quadratic ratio term `R` to `0` (and yielding `getMidPrice() == 0` when `_K_ = 1e18`).

### Impact assessment

- **Severity**: Medium.
- **Technical Impact**: Inaccurate calculation of the quadratic reserve factor `R` in `PMMPricing.getMidPrice`, corrupting the spot mid-price returned by `MagicLP.getMidPrice()` and the cumulative price `_BASE_PRICE_CUMULATIVE_LAST_` recorded by `_twapUpdate()`, or reverting with division by zero on line 207 if `R` truncates to `0` in `RState.BELOW_ONE`.
- **Business Impact**: External integrators, keepers, and TWAP consumers relying on `getMidPrice()` or `_BASE_PRICE_CUMULATIVE_LAST_` receive distorted pricing data, and affected pools can suffer denial of service on swaps and liquidity operations if `getMidPrice()` reverts during `_twapUpdate()`.

### Remediation

Compute the scaled ratio first using `DecimalMath.divFloor` (or scale by `DecimalMath.ONE` before dividing) and then square the fixed-point ratio using `DecimalMath.mulFloor`, avoiding unscaled integer division before multiplication:

```solidity
    function getMidPrice(PMMState memory state) internal pure returns (uint256) {
        if (state.R == RState.BELOW_ONE) {
            uint256 R = DecimalMath.divFloor(state.Q0, state.Q);
            R = DecimalMath.mulFloor(R, R);
            R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
            return DecimalMath.divFloor(state.i, R);
        } else {
            uint256 R = DecimalMath.divFloor(state.B0, state.B);
            R = DecimalMath.mulFloor(R, R);
            R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
            return DecimalMath.mulFloor(state.i, R);
        }
    }
```
