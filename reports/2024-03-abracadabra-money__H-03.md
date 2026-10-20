# Audit report — 2024-03-abracadabra-money/H-03

## Finding 1: Imbalanced `MIM` and `USDB` Locked Totals in `BlastOnboardingBoot.bootstrap` Allow Immediate Drainage of Excess Pool Reserves

### Synopsis

`BlastOnboardingBoot.bootstrap` initializes the `MagicLP` pool via `router.createPool` using the full `totals[MIM].locked` and `totals[USDB].locked` balances without ensuring they match the pool price ratio `I`. Because `MagicLP.buyShares` sets target reserves to the minimum side while recording the full balances as reserves, any attacker can immediately drain the excess reserve (`|totals[MIM].locked - totals[USDB].locked|`).

### Technical details

During the onboarding phase in `BlastOnboarding`, users independently deposit and lock `MIM` and `USDB`. There is no mechanism enforcing that `totals[MIM].locked` and `totals[USDB].locked` are equal (or in the ratio `I = 1e18`) when the onboarding state transitions to `State.Closed`.

In `BlastOnboardingBoot.sol`, `previewTotalPoolShares()` calls `router.previewCreatePool(I, baseAmount, quoteAmount)` (lines 86–90), which calculates the balanced amounts `baseAdjustedInAmount` and `quoteAdjustedInAmount`:

```solidity
shares = quoteInAmount < DecimalMath.mulFloor(baseInAmount, i) ? DecimalMath.divFloor(quoteInAmount, i) : baseInAmount;
baseAdjustedInAmount = shares;
quoteAdjustedInAmount = DecimalMath.mulFloor(shares, i);
```

However, `BlastOnboardingBoot.bootstrap` (lines 96–106) passes the raw, unadjusted `baseAmount = totals[MIM].locked` and `quoteAmount = totals[USDB].locked` directly to `router.createPool`:

```solidity
uint256 baseAmount = totals[MIM].locked;
uint256 quoteAmount = totals[USDB].locked;
MIM.safeApprove(address(router), type(uint256).max);
USDB.safeApprove(address(router), type(uint256).max);

(pool, totalPoolShares) = router.createPool(MIM, USDB, FEE_RATE, I, K, address(this), baseAmount, quoteAmount);
```

`Router.createPool` transfers the full `baseAmount` and `quoteAmount` to the newly created `MagicLP` clone and invokes `IMagicLP(clone).buyShares(address(this))`.

In `MagicLP.buyShares` (when `totalSupply == 0`), the initial shares and target reserves (`_BASE_TARGET_` $B_0$ and `_QUOTE_TARGET_` $Q_0$) are set based on the smaller side:
- $S = \min(B, \lfloor Q / I \rfloor)$
- $B_0 = S$ and $Q_0 = \lfloor S \cdot I \rfloor$
- Meanwhile, `_setReserve(baseBalance, quoteBalance)` sets the actual reserves to the full transferred balances: $B = \text{baseAmount}$ and $Q = \text{quoteAmount}$, with initial state `R = RState.ONE`.

In the DODO V2 PMM pricing state machine (`PMMPricing`), when $R = \text{ONE}$, the invariant assumes $B = B_0$ and $Q = Q_0$. When $B > B_0$ (i.e., `totals[MIM].locked > totals[USDB].locked` for $I = 10^{18}$):
1. A swapper selling a tiny amount of base token ($\Delta B$) triggers `_ROneSellBaseToken`, receiving $\Delta Q$ quote tokens and flipping the pool's state to `RState.BELOW_ONE`.
2. In `RState.BELOW_ONE`, `PMMPricing.sellQuoteToken` computes the tokens required to return to `RState.ONE` as:
   - `backToOnePayQuote = state.Q0 - state.Q` (which equals $\Delta Q$)
   - `backToOneReceiveBase = state.B - state.B0` (which equals $(B - B_0) + \Delta B$)
3. By selling back $\Delta Q$ quote tokens, the swapper receives `backToOneReceiveBase`, extracting the entire excess base reserve $B - B_0 = \text{totals[MIM].locked} - \text{totals[USDB].locked}$ at zero net cost.

Symmetrically, if $Q > Q_0$ (`totals[USDB].locked > totals[MIM].locked`), selling a tiny amount of quote token flips `R` to `RState.ABOVE_ONE`, and selling base token back to `RState.ONE` extracts the entire excess quote reserve $Q - Q_0 = \text{totals[USDB].locked} - \text{totals[MIM].locked}$.

### Assumptions and preconditions

1. Users have deposited and locked unequal value amounts of `MIM` and `USDB` during the onboarding phase (`totals[MIM].locked != totals[USDB].locked` given $I = 10^{18}$).
2. The owner closes the onboarding phase and calls `BlastOnboardingBoot.bootstrap`.

### Proof of concept

Assume $I = 10^{18}$, `totals[MIM].locked = 2,000,000e18`, and `totals[USDB].locked = 1,000,000e18`.

1. The owner calls `bootstrap(minAmountOut)`.
2. `bootstrap` calls `router.createPool` with `baseAmount = 2,000,000e18` and `quoteAmount = 1,000,000e18`.
3. `Router.createPool` transfers `2,000,000e18` MIM and `1,000,000e18` USDB to the new `pool` and calls `pool.buyShares(address(onboarding))`.
4. `pool.buyShares` calculates `shares = min(2,000,000e18, 1,000,000e18) = 1,000,000e18`, sets `_BASE_TARGET_ = 1,000,000e18`, `_QUOTE_TARGET_ = 1,000,000e18`, `_BASE_RESERVE_ = 2,000,000e18`, `_QUOTE_RESERVE_ = 1,000,000e18`, `_R_STATE_ = RState.ONE`, and mints `1,000,000e18 - 1001` shares to `BlastOnboarding`.
5. An attacker transfers `1e18` MIM to `pool` and calls `pool.sellBase(attacker)`.
   - Because `_R_STATE_ == RState.ONE`, the pool computes $\Delta Q \approx 1e18$ USDB, transfers $\Delta Q$ USDB to the attacker, updates `_R_STATE_ = RState.BELOW_ONE`, and updates reserves to `_BASE_RESERVE_ = 2,000,001e18` and `_QUOTE_RESERVE_ = 1,000,000e18 - \Delta Q`.
6. The attacker transfers $\Delta Q$ USDB back to `pool` and calls `pool.sellQuote(attacker)`.
   - Because `_R_STATE_ == RState.BELOW_ONE` and `payQuoteAmount == backToOnePayQuote` ($Q_0 - Q = \Delta Q$), `PMMPricing.sellQuoteToken` returns `receiveBaseAmount = backToOneReceiveBase = state.B - state.B0 = 2,000,001e18 - 1,000,000e18 = 1,000,001e18` MIM (less LP/maintainer fee).
7. The attacker profits `~1,000,000e18` MIM in a single transaction, and the onboarding users lose `1,000,000e18` MIM of their locked liquidity.

### Impact assessment

**Severity: High**

Any difference between `totals[MIM].locked` and `totals[USDB].locked` when `bootstrap()` is executed is immediately extractable by any external actor via two pool swaps, resulting in a direct, permanent loss of user-locked funds equal to `|totals[MIM].locked - totals[USDB].locked|`. Additionally, because `_claimable(user)` distributes `totalPoolShares` (which only reflects `2 * min(totals[MIM].locked, totals[USDB].locked)` of underlying liquidity after the excess is arbitraged) proportionally to `userLocked / totalLocked`, all locked users suffer a pro-rata loss of principal.

### Remediation

Before calling `router.createPool` in `BlastOnboardingBoot.bootstrap`, either:
1. Rebalance the excess locked token against external liquidity or a peg stability mechanism so that `baseAmount` and `quoteAmount` match the price ratio `I`, or
2. Only supply the adjusted amounts `(baseAdjustedInAmount, quoteAdjustedInAmount)` returned by `router.previewCreatePool(I, baseAmount, quoteAmount)` to `router.createPool`, and allow users (or the protocol) to handle or refund the unutilized excess locked tokens rather than donating them as unbalanced reserves into `MagicLP`.

```solidity
    function bootstrap(uint256 minAmountOut) external onlyOwner onlyState(State.Closed) returns (address, address, uint256) {
        if (pool != address(0)) {
            revert ErrAlreadyBootstrapped();
        }

        (uint256 baseAmount, uint256 quoteAmount, ) = router.previewCreatePool(
            I,
            totals[MIM].locked,
            totals[USDB].locked
        );
        MIM.safeApprove(address(router), baseAmount);
        USDB.safeApprove(address(router), quoteAmount);

        (pool, totalPoolShares) = router.createPool(MIM, USDB, FEE_RATE, I, K, address(this), baseAmount, quoteAmount);
```

## Finding 2: Insufficient Slippage and Liquidity Balance Validation in `BlastOnboardingBoot.bootstrap` Allows Skewed Pool Initialization

### Synopsis

In `BlastOnboardingBoot.bootstrap`, liquidity pool creation transfers unadjusted `MIM` and `USDB` locked totals into `Router.createPool` and only validates `totalPoolShares < minAmountOut`. Because `MagicLP.buyShares` mints shares and sets target reserves using only the minimum side of the deposit while recording full balances as reserves, imbalanced deposits or pre-deployment token donations corrupt initial PMM reserves and pricing without triggering the slippage check.

### Technical details

When `BlastOnboardingBoot.bootstrap(uint256 minAmountOut)` is executed to bootstrap the MIM/USDB `MagicLP` pool, it reads the total locked amounts `baseAmount = totals[MIM].locked` and `quoteAmount = totals[USDB].locked` and passes them directly to `router.createPool`:

```solidity
(pool, totalPoolShares) = router.createPool(MIM, USDB, FEE_RATE, I, K, address(this), baseAmount, quoteAmount);

if (totalPoolShares < minAmountOut) {
    revert ErrInsufficientAmountOut();
}
```

Inside `Router.createPool`, the pool is deployed deterministically via `IFactory(factory).create(baseToken, quoteToken, lpFeeRate, i, k)` (using `LibClone.cloneDeterministic`), the full `baseAmount` and `quoteAmount` are transferred to the new `MagicLP` clone, and `IMagicLP(clone).buyShares(to)` is invoked.

In `MagicLP.buyShares` (lines 375–407 of `src/mimswap/MagicLP.sol`), when `totalSupply() == 0`:

1. Minted `shares` (`totalPoolShares`) is determined solely by the smaller side of the liquidity relative to the oracle price `_I_`:
   ```solidity
   shares = quoteBalance < DecimalMath.mulFloor(baseBalance, _I_)
       ? DecimalMath.divFloor(quoteBalance, _I_)
       : baseBalance;
   _BASE_TARGET_ = shares.toUint112();
   _QUOTE_TARGET_ = DecimalMath.mulFloor(shares, _I_).toUint112();
   ```
2. After burning `1001` shares, `shares` is minted to `BlastOnboardingBoot`, and `_setReserve(baseBalance, quoteBalance)` sets `_BASE_RESERVE_ = baseBalance` and `_QUOTE_RESERVE_ = quoteBalance`.

This creates several closely related flaws at lines 108–124 of `src/blast/BlastOnboardingBoot.sol`:

- **Imbalanced Liquidity Valuation and Target/Reserve Divergence:** Unlike `Router.previewCreatePool` and `Router._adjustAddLiquidity` (which compute `baseAdjustedInAmount = shares` and `quoteAdjustedInAmount = DecimalMath.mulFloor(shares, i)`), `bootstrap` deposits the entire unadjusted `baseAmount` and `quoteAmount` into the pool. Whenever `quoteBalance != DecimalMath.mulFloor(baseBalance, _I_)` (either due to imbalanced user locking during onboarding or precision truncation in `DecimalMath.mulFloor(shares, _I_)`), all excess tokens on the larger side are absorbed into `_BASE_RESERVE_` or `_QUOTE_RESERVE_` without minting additional `totalPoolShares` or increasing `_BASE_TARGET_` and `_QUOTE_TARGET_`.
- **Initial Reserve and Price Manipulation via Deterministic Pool Address:** Because `Factory.create` deploys `pool` via `CREATE2` with a predictable salt derived from `(tx.origin, MIM, USDB, FEE_RATE, I, K)`, the pool address can be precomputed via `IFactory.predictDeterministicAddress` before `bootstrap` is called. Any tokens transferred directly to the predicted `pool` address prior to `bootstrap` are included in `baseBalance` and `quoteBalance` during `buyShares`, skewing the initial reserve-to-target ratio (`_BASE_RESERVE_ > _BASE_TARGET_` or `_QUOTE_RESERVE_ > _QUOTE_TARGET_`) while `_RState_` remains at `RState.ONE`.
- **Share-Denominated Slippage Validation Gap:** The check at line 108 (`if (totalPoolShares < minAmountOut)`) only verifies that `totalPoolShares` meets a minimum threshold. Because `totalPoolShares` only reflects `min(quoteBalance / _I_, baseBalance) - 1001`, it cannot detect excess unbacked reserves on either side of the pool, pre-deployment token donations (which only increase `baseBalance` or `quoteBalance` and thus never decrease `totalPoolShares`), or the resulting distortion of the pool's PMM pricing curve and LP share valuation when staked into `LockingMultiRewards`.

### Assumptions and preconditions

1. `BlastOnboarding` has transitioned to `State.Closed` after users locked `MIM` and `USDB` tokens.
2. Either the total locked amounts `totals[MIM].locked` and `totals[USDB].locked` are not in the exact ratio `I` (`quoteAmount != DecimalMath.mulFloor(baseAmount, I)`), or an external actor transfers `MIM` or `USDB` to the deterministic `pool` address prior to the `bootstrap` transaction.

### Proof of concept

1. Suppose `I = 1e18` (1 MIM = 1 USDB). During the onboarding period, users lock a total of `baseAmount = 2,000,000e18` MIM and `quoteAmount = 1,000,000e18` USDB.
2. The owner previews the expected shares via `previewTotalPoolShares()`, which calls `router.previewCreatePool(I, 2,000,000e18, 1,000,000e18)` and returns:
   - `baseAdjustedInAmount = 1,000,000e18`
   - `quoteAdjustedInAmount = 1,000,000e18`
   - `shares = 1,000,000e18 - 1001`
3. The owner calls `bootstrap(minAmountOut)` with `minAmountOut = 1,000,000e18 - 1001`.
4. `router.createPool` transfers the entire `2,000,000e18` MIM and `1,000,000e18` USDB into the newly created `MagicLP` pool and calls `buyShares(address(this))`.
5. `MagicLP.buyShares` computes `shares = min(1,000,000e18, 2,000,000e18) = 1,000,000e18`, sets `_BASE_TARGET_ = 1,000,000e18` and `_QUOTE_TARGET_ = 1,000,000e18`, mints `totalPoolShares = 1,000,000e18 - 1001` to `BlastOnboardingBoot`, and sets `_BASE_RESERVE_ = 2,000,000e18` and `_QUOTE_RESERVE_ = 1,000,000e18`.
6. At line 108, `totalPoolShares < minAmountOut` evaluates to `false` (`1,000,000e18 - 1001 < 1,000,000e18 - 1001`), so the check passes.
7. Alternatively, even if `totals[MIM].locked` and `totals[USDB].locked` were balanced at `1,000,000e18` each, a third party can precompute the `pool` address via `factory.predictDeterministicAddress(MIM, USDB, FEE_RATE, I, K)` and transfer tokens to `pool` before `bootstrap` executes, causing `baseBalance` or `quoteBalance` to exceed `_BASE_TARGET_` or `_QUOTE_TARGET_` while `totalPoolShares >= minAmountOut` still passes.
8. As a result, the pool is initialized with `_BASE_RESERVE_ = 2,000,000e18 > _BASE_TARGET_ = 1,000,000e18` (an immediate PMM state and pricing distortion), and only `1,000,000e18 - 1001` shares are minted and approved to `LockingMultiRewards` despite `3,000,000e18` worth of tokens being deposited into the pool.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** The `MagicLP` pool is initialized with `_BASE_RESERVE_ != _BASE_TARGET_` or `_QUOTE_RESERVE_ != _QUOTE_TARGET_` while in `RState.ONE`, violating the DODO PMM equilibrium invariant at genesis, skewing swap pricing, and minting `totalPoolShares` that only reflect `min(quoteBalance / I, baseBalance)` rather than the full deposited liquidity.
- **Business Impact:** Onboarding participants who locked tokens suffer immediate value distortion or arbitrage losses in the bootstrapped pool, and the `minAmountOut` slippage check in `bootstrap` fails to prevent bootstrapping with imbalanced reserves.

### Remediation

In `BlastOnboardingBoot.bootstrap`, compute the balanced input amounts using `router.previewCreatePool(I, baseAmount, quoteAmount)` (or validate that the reserves and targets of the created `MagicLP` pool match the expected balanced amounts) and only transfer the adjusted amounts into the pool, or revert if the unadjusted deposits deviate beyond acceptable bounds.

```solidity
        (uint256 baseAdjustedInAmount, uint256 quoteAdjustedInAmount, uint256 expectedShares) = router.previewCreatePool(
            I,
            baseAmount,
            quoteAmount
        );
        if (expectedShares < minAmountOut || baseAdjustedInAmount < baseAmount || quoteAdjustedInAmount < quoteAmount) {
            revert ErrInsufficientAmountOut();
        }

        // Create staking contract
        // 3x boosting for locker, 7 days reward duration, 13 weeks lp locking
        // make this contract temporary the owner the set it as an operator
        // for permissionned `stakeFor` during the claiming process and then
        // transfer the ownership to the onboarding owner.
        staking = new LockingMultiRewards(pool, 30_000, 7 days, 13 weeks, address(this));
        staking.setOperator(address(this), true);
        staking.transferOwnership(owner);

        // Approve staking contract
        pool.safeApprove(address(staking), totalPoolShares);

        emit LogLiquidityBootstrapped(pool, address(staking), totalPoolShares);
```

## Finding 3: Precision Loss in Reciprocal Price Calculation in `MagicLP.querySellQuote` and `MagicLP.querySellBase` Causes Severe Swap Mispricing and Target Drift

### Synopsis

In `MagicLP.querySellBase` and `MagicLP.querySellQuote`, calls to `getPMMState()` and `PMMPricing.sellQuoteToken()` rely on `DecimalMath.reciprocalFloor(state.i)` (`10**36 / state.i`) to invert the oracle price `i`. When `state.i` is large or does not evenly divide `10**36`, truncation in `reciprocalFloor` causes severe precision loss, underpaying quote-to-base swaps and underestimating the base target `B0`.

### Technical details

In `MagicLP`, `_I_` (`state.i`) represents the price of one base token in terms of quote tokens scaled by `10**18` (taking token decimal differences into account: `i = price * 10**(18 + quoteDecimals - baseDecimals)`).

When executing `querySellBase` (lines 171–178) and `querySellQuote` (lines 180–185):
1. Both functions first call `getPMMState()` (lines 171 and 184), which invokes `PMMPricing.adjustedTarget(state)` (line 201). When `state.R == RState.ABOVE_ONE`, `adjustedTarget` computes the dynamic base target `state.B0` by passing `DecimalMath.reciprocalFloor(state.i)` as the price parameter to `Math._SolveQuadraticFunctionForTarget` (`PMMPricing.sol` line 197).
2. `querySellQuote` then calls `PMMPricing.sellQuoteToken(state, payQuoteAmount)` (line 185), which computes `receiveBaseAmount` across all `RState` branches using `DecimalMath.reciprocalFloor(state.i)` as the effective price (`PMMPricing.sol` line 185).

`DecimalMath.reciprocalFloor(uint256 target)` (`DecimalMath.sol` lines 39–41) computes:
```solidity
return ONE2 / target; // 10**36 / state.i
```
Because integer division truncates the reciprocal `10**36 / state.i` *before* it is multiplied by the quote token delta (`payQuoteAmount` or `state.Q - state.Q0`), it introduces severe precision loss whenever `state.i` is large (for instance, when the base token has fewer decimals than the quote token, such as a 6- or 8-decimal base token paired with an 18-decimal quote token, or when the base token has a high unit price):

- **Systematic Underpayment and Zero-Output Swaps in `querySellQuote` (`185`):**
  For a 6-decimal base token worth 60,000 of an 18-decimal quote token, `state.i = 60,000 * 10**(18 + 18 - 6) = 6 * 10**34`. `DecimalMath.reciprocalFloor(state.i)` truncates `10**36 / (6 * 10**34) = 16.666...` down to `16`—a permanent **4% pricing error** on all quote-to-base swaps. If `state.i > 10**36`, `DecimalMath.reciprocalFloor(state.i)` evaluates to `0`, causing `PMMPricing.sellQuoteToken` to return `receiveBaseAmount = 0` for any `payQuoteAmount` when `R != BELOW_ONE`.

- **Base Target (`B0`) Underestimation and Discontinuity in `querySellBase` (`171-172`):**
  When `state.R == RState.ABOVE_ONE`, `getPMMState()` (`171`) computes `state.B0` via `adjustedTarget` using the truncated `DecimalMath.reciprocalFloor(state.i)`. This underestimates `state.B0` and therefore underestimates `backToOnePayBase = state.B0 - state.B` (`PMMPricing.sol` line 46).
  Meanwhile, inside `PMMPricing.sellBaseToken` (`172`), Case 2.1 (`payBaseAmount < backToOnePayBase`) prices the trade via `_RAboveSellBaseToken` (`PMMPricing.sol` line 172), which calls `Math._GeneralIntegrate` with the untruncated `state.i`, whereas Case 2.2/2.3 (`payBaseAmount >= backToOnePayBase`) awards the full `backToOneReceiveQuote = state.Q - state.Q0` for paying only the underestimated `backToOnePayBase` and locks `newBaseTarget = state.B0` into `_BASE_TARGET_`.

### Assumptions and preconditions

1. A `MagicLP` pool is configured with a pair where `_I_` (`state.i`) is large (e.g., when `_BASE_TOKEN_` has fewer decimals than `_QUOTE_TOKEN_`, such as 6 or 8 decimals vs. 18 decimals, and/or `_BASE_TOKEN_` has a higher unit price than `_QUOTE_TOKEN_`) or does not evenly divide `10**36`.
2. Users or routers interact with `querySellQuote` / `sellQuote` or `querySellBase` / `sellBase` (when `_RState_ == ABOVE_ONE`).

### Proof of concept

Consider a `MagicLP` pool where `_BASE_TOKEN_` has 6 decimals (e.g., a 6-decimal token worth 60,000 quote units) and `_QUOTE_TOKEN_` has 18 decimals (e.g., MIM), with curvature `K = 0` (or small `K` for a stable/pegged pair):
1. The oracle price is initialized to `state.i = 60,000 * 10**(18 + 18 - 6) = 6 * 10**34`.
2. In `DecimalMath.reciprocalFloor(state.i)`, the reciprocal price is calculated as:
   `10**36 / (6 * 10**34) = 16` (truncating the fractional `0.6666...`, which represents a 4% loss of precision).
3. When a trader calls `querySellQuote(trader, 60_000e18)` (`184-185`) at `RState.ONE`:
   - `PMMPricing.sellQuoteToken` computes the base output using `i_reciprocal = 16`:
     `receiveBaseAmount = (60_000e18 * 16) / 1e18 = 960_000` (0.96 base tokens), instead of the fair `1_000_000` (1.00 base token).
4. Suppose the pool is in `RState.ABOVE_ONE` with `state.B = 10_000_000` (10 base tokens), `state.Q = 660_000e18`, and `state.Q0 = 600_000e18` (`backToOneReceiveQuote = 60_000e18`, worth `1_000_000` base units at `state.i = 6 * 10**34`):
   - In `querySellBase` (`171-172`), `getPMMState()` calls `PMMPricing.adjustedTarget(state)`, which computes `state.B0 = 10_000_000 + (60_000e18 * 16) / 1e18 = 10_960_000`, so `backToOnePayBase = state.B0 - state.B = 960_000`.
   - If a trader sells `payBaseAmount = 959_999` (`backToOnePayBase - 1`), Case 2.1 calls `_RAboveSellBaseToken` with the exact `state.i = 6 * 10**34`, returning `(959_999 * 6 * 10**34) / 1e18 = 57_599.94e18` quote tokens.
   - If the trader sells just 1 unit more (`payBaseAmount = 960_000 == backToOnePayBase`), Case 2.2 triggers and returns `backToOneReceiveQuote = 60_000e18` quote tokens—yielding an extra `2_400.06e18` quote tokens for 1 base unit and updating `_BASE_TARGET_` to the underestimated `10_960_000`.

### Impact assessment

- **Severity:** High / Medium
- **Direct Technical Effect:** `DecimalMath.reciprocalFloor(state.i)` loses precision whenever `state.i` is large or does not divide `10**36`, causing inaccurate quote-to-base swap outputs in `querySellQuote`, underestimated base target `B0` in `getPMMState()`, and a pricing discontinuity in `querySellBase` when transitioning out of `RState.ABOVE_ONE`.
- **Business Implications:** Traders selling quote tokens receive systematically less base token than dictated by the PMM curve (up to 100% loss if `state.i > 10**36`), while arbitrageurs can exploit the underestimated `backToOnePayBase` in `RState.ABOVE_ONE` to extract excess quote tokens and permanently depress `_BASE_TARGET_` at the expense of liquidity providers.

### Remediation

Avoid pre-inverting `state.i` via `DecimalMath.reciprocalFloor(state.i)`. Instead, pass `state.i` directly into dedicated inverse-pricing helpers (or scale the reciprocal with higher precision using 512-bit `Math.mulDiv` before dividing by `state.i`) so that division by `state.i` occurs after multiplying by the quote token delta `delta`.

For example, in `PMMPricing.sol` and `Math.sol`, compute quote-to-base target and trade amounts by dividing `delta * DecimalMath.ONE` by `state.i` (using full-precision `mulDiv` where necessary) rather than multiplying `delta` by `DecimalMath.reciprocalFloor(state.i)`:

```solidity
    function adjustedTarget(PMMState memory state) internal pure {
        if (state.R == RState.BELOW_ONE) {
            state.Q0 = Math._SolveQuadraticFunctionForTarget(state.Q, state.B - state.B0, state.i, state.K);
        } else if (state.R == RState.ABOVE_ONE) {
            state.B0 = Math._SolveQuadraticFunctionForTargetQuote(
                state.B,
                state.Q - state.Q0,
                state.i,
                state.K
            );
        }
    }
```

Where `_SolveQuadraticFunctionForTargetQuote` and the quote-selling trade functions compute `i * delta` as `DecimalMath.divFloor(delta, i)` (i.e., `(delta * DecimalMath.ONE) / i`) instead of `DecimalMath.mulFloor(DecimalMath.reciprocalFloor(i), delta)`.

## Finding 4: Fee Rounding Down in `MagicLP.querySellBase` and `MagicLP.querySellQuote` Allows Zero-Fee Swaps and Undercharges Traders

### Synopsis

In `MagicLP.querySellBase` and `MagicLP.querySellQuote`, both the liquidity provider fee (`lpFee`) and maintainer fee (`mtFee`) are calculated using `DecimalMath.mulFloor` instead of rounding up. Because fees are subtracted from the swap output, rounding down biases payouts in favor of traders and enables zero-fee swaps on small trade amounts.

### Technical details

When a swap is quoted or executed via `sellBase` or `sellQuote`, `MagicLP` calls `querySellBase` (lines 167–178) or `querySellQuote` (lines 180–191) to compute the output token amount, maintainer fee (`mtFee`), new `RState`, and updated target reserve.

In `querySellBase` (lines 174–176):

```solidity
(uint256 lpFeeRate, uint256 mtFeeRate) = _MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_);
mtFee = DecimalMath.mulFloor(receiveQuoteAmount, mtFeeRate);
receiveQuoteAmount = receiveQuoteAmount - DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate) - mtFee;
```

And identically in `querySellQuote` (lines 187–189):

```solidity
(uint256 lpFeeRate, uint256 mtFeeRate) = _MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_);
mtFee = DecimalMath.mulFloor(receiveBaseAmount, mtFeeRate);
receiveBaseAmount = receiveBaseAmount - DecimalMath.mulFloor(receiveBaseAmount, lpFeeRate) - mtFee;
```

`IFeeRateModel.getFeeRate` splits the configured `_LP_FEE_RATE_` (which can be as low as `MIN_LP_FEE_RATE = 1e14`, i.e., `0.01%`) into an adjusted `lpFeeRate` and `mtFeeRate` such that `lpFeeRate + mtFeeRate == _LP_FEE_RATE_`.

There are two compounding arithmetic flaws in this fee calculation:

1. **Wrong rounding direction on fee deduction:** Both `DecimalMath.mulFloor(receiveQuoteAmount, mtFeeRate)` and `DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate)` truncate toward zero (`(target * d) / 1e18`). Because these two fee values are subtracted from `receiveQuoteAmount` (and `receiveBaseAmount`), rounding the fees down systematically rounds the net amount transferred out to the trader **up**, reducing the fee retained by LPs in the pool and the fee sent to the maintainer.
2. **Independent truncation of split fee components:** Because the total fee rate is split into `lpFeeRate` and `mtFeeRate` before applying `mulFloor` to each independently, both components suffer separate truncation errors:
   $$\lfloor \text{amount} \times \text{lpFeeRate} / 10^{18} \rfloor + \lfloor \text{amount} \times \text{mtFeeRate} / 10^{18} \rfloor \le \lfloor \text{amount} \times (\text{lpFeeRate} + \text{mtFeeRate}) / 10^{18} \rfloor$$
   Whenever `receiveQuoteAmount < 1e18 / lpFeeRate` (for `lpFeeRate = 1e14`, any `receiveQuoteAmount < 10,000` token units; or `< 20,000` units if split evenly between `lpFeeRate` and `mtFeeRate`), both `DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate)` and `mtFee` evaluate to `0`.

Consequently, any swap producing an output smaller than `1e18 / max(lpFeeRate, mtFeeRate)` pays zero LP fee and zero maintainer fee. For pools involving low-decimal tokens (such as 6-decimal USDT/USDC or 8-decimal WBTC), `10,000` to `20,000` raw units can represent non-trivial value per swap, allowing traders to split orders into multiple small swaps in a single transaction to completely evade protocol and LP fees.

### Assumptions and preconditions

- A `MagicLP` pool is deployed and initialized with a valid `_LP_FEE_RATE_` between `MIN_LP_FEE_RATE` (`1e14`) and `MAX_LP_FEE_RATE` (`1e16`).
- A trader executes one or more swaps via `sellBase` or `sellQuote` (for example, batching smaller swaps through a contract or router).

### Proof of concept

1. Consider the `MIM/USDT` pool from `MagicLPTest` in `test/MIMSwap.t.sol`, where `MIM` has 18 decimals, `USDT` has 6 decimals, `i = 1_000_000` (`1e6`), and `_LP_FEE_RATE_ = MIN_LP_FEE_RATE = 1e14` (`0.01%`), split equally into `lpFeeRate = 5e13` and `mtFeeRate = 5e13` (or `lpFeeRate = 1e14, mtFeeRate = 0`).
2. A trader transfers `0.009999e18` MIM (`~0.01 MIM`) to the `MagicLP` contract and calls `sellBase(trader)`.
3. Inside `querySellBase`:
   - `PMMPricing.sellBaseToken` computes `receiveQuoteAmount = 9,999` (`0.009999` USDT in 6 decimals).
   - `mtFee = DecimalMath.mulFloor(9999, mtFeeRate) = (9999 * 5e13) / 1e18 = 0`.
   - `lpFee = DecimalMath.mulFloor(9999, lpFeeRate) = (9999 * 5e13) / 1e18 = 0`.
   - `receiveQuoteAmount` remains `9,999` with `0` fee deducted.
4. By looping this swap in a single transaction, the trader swaps arbitrary volumes without paying any LP fee or maintainer fee.

### Impact assessment

- **Severity:** Medium.
- **Technical Effect:** Swap fees (`lpFee` and `mtFee`) are consistently rounded down in favor of the trader, reducing to zero when `outputAmount * feeRate < 1e18`.
- **Business Impact:** Liquidity providers and the protocol maintainer lose accrued swap fees on every trade due to downward truncation, and traders on low-decimal or high-unit-value tokens (e.g., USDT, USDC, WBTC) can completely bypass fees by splitting swaps into smaller chunks.

### Remediation

Round the LP and maintainer fee deductions up using `DecimalMath.mulCeil` (while ensuring the fees do not exceed the gross output amount when `receiveQuoteAmount` or `receiveBaseAmount` is `1` wei).

```solidity
    function querySellBase(
        address trader,
        uint256 payBaseAmount
    ) public view returns (uint256 receiveQuoteAmount, uint256 mtFee, PMMPricing.RState newRState, uint256 newBaseTarget) {
        PMMPricing.PMMState memory state = getPMMState();
        (receiveQuoteAmount, newRState) = PMMPricing.sellBaseToken(state, payBaseAmount);

        (uint256 lpFeeRate, uint256 mtFeeRate) = _MT_FEE_RATE_MODEL_.getFeeRate(trader, _LP_FEE_RATE_);
        mtFee = DecimalMath.mulCeil(receiveQuoteAmount, mtFeeRate);
        uint256 lpFee = DecimalMath.mulCeil(receiveQuoteAmount, lpFeeRate);
        uint256 totalFee = lpFee + mtFee;
        if (totalFee > receiveQuoteAmount) {
            receiveQuoteAmount = 0;
            mtFee = 0;
        } else {
            receiveQuoteAmount -= totalFee;
        }
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
        uint256 lpFee = DecimalMath.mulCeil(receiveBaseAmount, lpFeeRate);
        uint256 totalFee = lpFee + mtFee;
        if (totalFee > receiveBaseAmount) {
            receiveBaseAmount = 0;
            mtFee = 0;
        } else {
            receiveBaseAmount -= totalFee;
        }
        newQuoteTarget = state.Q0;
    }
```

## Finding 5: Missing `nonReadReentrant` guard on `MagicLP.getReserves`, `MagicLP.getMidPrice`, and `MagicLP.getPMMStateForCall` exposes inconsistent transient state during external callbacks

### Synopsis

`MagicLP` inherits Solady's `ReentrancyGuard` but fails to apply the `nonReadReentrant` modifier to its valuation and state view functions (`getReserves`, `getMidPrice`, and `getPMMStateForCall`). During `flashLoan` and token transfer callbacks in `sellShares`, `sellBase`, and `sellQuote`, external contracts can read stale reserves and inconsistent transient state via read-only reentrancy.

### Technical details

`MagicLP` inherits `ReentrancyGuard` from Solady (`lib/solady/src/utils/ReentrancyGuard.sol`), which provides both `nonReentrant` for state-changing functions and `nonReadReentrant` to protect view functions from read-only reentrancy. While state-modifying functions in `MagicLP` (`sellBase`, `sellQuote`, `flashLoan`, `buyShares`, `sellShares`, `sync`, `setParameters`, `ratioSync`) are protected with `nonReentrant`, the state and valuation view functions in `src/mimswap/MagicLP.sol` (`getPMMStateForCall` at line 204, `getMidPrice` at line 215, and `getReserves` at line 219) lack the `nonReadReentrant` modifier.

This exposes divergent cached reserves and transient state to external callers during execution callbacks before `_sync()` or `_setReserve()` updates storage:

1. **In `flashLoan` (lines 290–353):**
   `_transferBaseOut(assetTo, baseAmount)` and `_transferQuoteOut(assetTo, quoteAmount)` transfer the underlying tokens out of the pool at lines 291–292, and `ICallee(assetTo).FlashLoanCall(msg.sender, baseAmount, quoteAmount, data)` is invoked at line 295 before `_BASE_RESERVE_`, `_QUOTE_RESERVE_`, `_BASE_TARGET_`, `_QUOTE_TARGET_`, and `_RState_` are updated (lines 320–350). During `FlashLoanCall`, the live token balances of `MagicLP` have been depleted by `baseAmount` and `quoteAmount`, yet `getReserves()`, `getMidPrice()`, and `getPMMStateForCall()` continue to return the pre-loan cached reserves and PMM state. An attacker simultaneously holds the flash-loaned tokens while `MagicLpAggregator.latestAnswer()` (which calls `pair.getReserves()` at line 34 of `MagicLpAggregator.sol`) and any external integrators querying `getMidPrice()` or `getPMMStateForCall()` still observe the full pre-loan reserves in `MagicLP`, allowing the same underlying capital to be double-counted across protocols.

2. **In `sellShares` (lines 413–455), `sellBase` (lines 244–265), and `sellQuote` (lines 267–288):**
   In `sellShares`, `_BASE_TARGET_` and `_QUOTE_TARGET_` are reduced at lines 438–439 and `_burn(msg.sender, shareAmount)` decreases `totalSupply()` at line 445 before `_transferBaseOut` (line 446), `_transferQuoteOut` (line 447), and `_sync()` (line 448) are executed. If either token triggers a transfer hook/callback during `_transferBaseOut` or `_transferQuoteOut`, `pair.totalSupply()` has already decreased while `getReserves()` still returns the unreduced `_BASE_RESERVE_` and `_QUOTE_RESERVE_`, causing `MagicLpAggregator.latestAnswer()` (`minAnswer * (baseReserve + quoteReserve) / pair.totalSupply()`) and `getMidPrice()` (using reduced targets with unreduced reserves) to return severely corrupted values.

### Assumptions and preconditions

1. An external consumer such as `MagicLpAggregator` (used by Cauldrons to price `MagicLP` collateral) or another integrated contract reads `getReserves()`, `getMidPrice()`, or `getPMMStateForCall()` from `MagicLP`.
2. A non-privileged caller invokes `MagicLP.flashLoan()` (with `data.length > 0` to trigger `FlashLoanCall`) or `MagicLP.sellShares()` / `sellBase()` / `sellQuote()` when a token with transfer callbacks is used.

### Proof of concept

1. Assume a `MagicLP` pool holds `10,000,000e18` base tokens and `10,000,000e18` quote tokens (`_BASE_RESERVE_ = 10,000,000e18`, `_QUOTE_RESERVE_ = 10,000,000e18`, `totalSupply() = 10,000,000e18`).
2. An attacker calls `MagicLP.flashLoan(10,000,000e18, 10,000,000e18, attackerContract, "0x01")`.
3. `MagicLP` sets the reentrancy lock (`_REENTRANCY_GUARD_SLOT = address(this)`), transfers `10,000,000e18` base tokens and `10,000,000e18` quote tokens to `attackerContract`, and calls `attackerContract.FlashLoanCall(...)` before updating `_BASE_RESERVE_` or `_QUOTE_RESERVE_`.
4. Inside `FlashLoanCall`, `attackerContract` holds the entire `10,000,000e18` base and `10,000,000e18` quote tokens (while the pool's actual ERC20 balances are `0`), or in a `sellShares` transfer callback, `_burn` has already reduced `totalSupply()` by 90% to `1,000,000e18` while `_BASE_RESERVE_` and `_QUOTE_RESERVE_` are still `10,000,000e18`.
5. `attackerContract` interacts with a Cauldron or protocol that calls `MagicLpAggregator.latestAnswer()` (or `MagicLP.getMidPrice()` / `MagicLP.getPMMStateForCall()`). Because `getReserves()`, `getMidPrice()`, and `getPMMStateForCall()` lack `nonReadReentrant`, the call succeeds and reads the stale pre-callback `_BASE_RESERVE_` and `_QUOTE_RESERVE_`.
6. After exploiting the stale valuation or double-counted liquidity during the callback, `attackerContract` returns the flash-loaned tokens to `MagicLP` so `flashLoan()` completes without reverting.

### Impact assessment

**Severity: High**

External protocols and `MagicLpAggregator` relying on `getReserves()`, `getMidPrice()`, or `getPMMStateForCall()` are vulnerable to read-only reentrancy during `flashLoan` and token transfer callbacks. An attacker can double-count the pool's entire underlying liquidity during `FlashLoanCall` (holding all pool tokens while `MagicLP` view functions still report full reserves) or inflate LP share valuations in `MagicLpAggregator.latestAnswer()` during `sellShares` token callbacks, leading to undercollateralized borrowing and theft of funds from lending markets.

### Remediation

Apply the `nonReadReentrant` modifier from Solady's `ReentrancyGuard` to `getPMMStateForCall`, `getMidPrice`, and `getReserves` (as well as `getPMMState` if exposed externally), and extract an internal `_getMidPrice()` helper for `_twapUpdate()` so internal calls during `nonReentrant` execution do not revert:

```solidity
    function getPMMStateForCall() external view nonReadReentrant returns (uint256 i, uint256 K, uint256 B, uint256 Q, uint256 B0, uint256 Q0, uint256 R) {
        PMMPricing.PMMState memory state = getPMMState();
        i = state.i;
        K = state.K;
        B = state.B;
        Q = state.Q;
        B0 = state.B0;
        Q0 = state.Q0;
        R = uint256(state.R);
    }

    function getMidPrice() public view nonReadReentrant returns (uint256 midPrice) {
        return _getMidPrice();
    }

    function _getMidPrice() internal view returns (uint256 midPrice) {
        return PMMPricing.getMidPrice(getPMMState());
    }

    function getReserves() external view nonReadReentrant returns (uint256 baseReserve, uint256 quoteReserve) {
        baseReserve = _BASE_RESERVE_;
        quoteReserve = _QUOTE_RESERVE_;
    }

    function getUserFeeRate(address user) external view nonReadReentrant returns (uint256 lpFeeRate, uint256 mtFeeRate) {
        return _MT_FEE_RATE_MODEL_.getFeeRate(user, _LP_FEE_RATE_);
    }
```

## Finding 6: Manipulation of Spot Reserves Returned by `MagicLP.getReserves` Allows Arbitrarily Inflating LP Token Valuation

### Synopsis

`MagicLP.getReserves()` returns the raw cached spot reserves (`_BASE_RESERVE_` and `_QUOTE_RESERVE_`), which are consumed by `MagicLpAggregator.latestAnswer()` to value `MagicLP` collateral. An attacker can flash-loan tokens and skew the pool via swaps or `sync()` to massively inflate `baseReserve + quoteReserve` and drain lending Cauldrons.

### Technical details

`MagicLP.getReserves()` (lines 219–222 of `src/mimswap/MagicLP.sol`) exposes the pool's instantaneous cached spot reserves:

```solidity
function getReserves() external view returns (uint256 baseReserve, uint256 quoteReserve) {
    baseReserve = _BASE_RESERVE_;
    quoteReserve = _QUOTE_RESERVE_;
}
```

`MagicLpAggregator.latestAnswer()` relies on `pair.getReserves()` to price `MagicLP` tokens as collateral for Abracadabra Cauldrons:

$$\text{Price}_{\text{LP}} = \frac{\min(\text{Price}_{\text{base}}, \text{Price}_{\text{quote}}) \times (\text{baseReserve} + \text{quoteReserve})}{\text{totalSupply}}$$

Although the protocol assumes that for closely priced stable pairs (such as `MIM/USDB` with $i \approx 10^{18}$ and small $K$), the sum of reserves $\text{baseReserve} + \text{quoteReserve}$ remains constant per LP share, this invariant does not hold for the PMM bonding curve implemented in `PMMPricing`:

1. **Asymmetric Reserve Growth Under Large Swaps (`sellBase` / `sellQuote`):**
   When a user sells `payBaseAmount` of `_BASE_TOKEN_` via `sellBase()` (starting from $R = \text{ONE}$ where $B = B_0$ and $Q = Q_0$), `_ROneSellBaseToken` solves the quadratic PMM invariant where the maximum quote tokens that can ever leave the pool is bounded above by the target reserve $Q_0$. However, `_BASE_RESERVE_` increases by the entire `payBaseAmount`. When `payBaseAmount` is large relative to $Q_0$, `receiveQuoteAmount` approaches $Q_0$, while the new sum of spot reserves becomes:
   $$\text{baseReserve}' + \text{quoteReserve}' = (B_0 + \text{payBaseAmount}) + (Q_0 - \text{receiveQuoteAmount}) \approx B_0 + \text{payBaseAmount}$$
   Because `totalSupply()` does not change during a swap, the sum $\text{baseReserve} + \text{quoteReserve}$ returned by `getReserves()` increases almost linearly with `payBaseAmount` and can be inflated to an arbitrary multiple of the pool's true equilibrium value. Afterward, calling `sellQuote()` with the received quote tokens traverses the exact inverse branch in `PMMPricing.sellQuoteToken` (`backToOnePayQuote = state.Q0 - state.Q`, `backToOneReceiveBase = state.B - state.B0`), returning the attacker's base tokens minus only the small LP fee.

2. **Direct Reserve Inflation via `sync()`:**
   `MagicLP.sync()` is permissionless and updates `_BASE_RESERVE_` and `_QUOTE_RESERVE_` to match the contract's current token balances without minting LP shares. An attacker who holds a majority of a pool's LP shares can transfer tokens directly to `MagicLP`, call `sync()` to inflate `getReserves()`, borrow against the overvalued LP collateral, and then call `sellShares()` to withdraw the donated reserves back proportionally.

In contrast, the PMM's equilibrium target reserves `(B0, Q0)` computed via `getPMMState()` (`PMMPricing.adjustedTarget`) represent the pool's reserve balances at equilibrium ($R = \text{ONE}$) and only grow by accrued LP fees rather than unbalanced trade volume.

### Assumptions and preconditions

- A `MagicLP` pool is deployed and initialized for a token pair (e.g., `MIM/USDB` with $i \approx 10^{18}$ and $K > 0$), and its LP tokens are integrated as collateral in a lending market (Cauldron) priced via `MagicLpAggregator`.
- The attacker can obtain sufficient liquidity (e.g., via flash loans) relative to the pool's reserves to execute a large swap or reserve donation within a single transaction.

### Proof of concept

1. Assume a `MagicLP` pool for `MIM/USDB` is initialized with $i = 10^{18}$, $K = 5 \times 10^{14}$ ($0.05\%$), and `lpFeeRate` = $10^{14}$ ($0.01\%$), holding equilibrium reserves of $\text{B} = \text{B}_0 = 1{,}000{,}000 \times 10^{18}$ MIM and $\text{Q} = \text{Q}_0 = 1{,}000{,}000 \times 10^{18}$ USDB, with `totalSupply()` = $1{,}000{,}000 \times 10^{18}$.
2. Initially, `getReserves()` returns $(1{,}000{,}000 \times 10^{18}, 1{,}000{,}000 \times 10^{18})$. Assuming both tokens are priced at $\$1$ ($10^{18}$), `MagicLpAggregator.latestAnswer()` returns:
   $$\frac{10^{18} \times (1{,}000{,}000 \times 10^{18} + 1{,}000{,}000 \times 10^{18})}{1{,}000{,}000 \times 10^{18}} = 2 \times 10^{18} \text{ (\$2.00 per LP token)}$$
3. An attacker holds a modest amount of `MagicLP` shares as collateral in a Cauldron and flash-loans $50{,}000{,}000 \times 10^{18}$ MIM.
4. The attacker transfers $50{,}000{,}000 \times 10^{18}$ MIM to the `MagicLP` pool and calls `sellBase(attacker)`. Because the pool only holds $1{,}000{,}000 \times 10^{18}$ USDB, the attacker receives slightly less than $1{,}000{,}000 \times 10^{18}$ USDB.
5. `_setReserve()` updates `_BASE_RESERVE_` to $51{,}000{,}000 \times 10^{18}$ and `_QUOTE_RESERVE_` to near $0$.
6. Now `getReserves()` returns `baseReserve + quoteReserve` $\approx 51{,}000{,}000 \times 10^{18}$, while `totalSupply()` is still $1{,}000{,}000 \times 10^{18}$. `MagicLpAggregator.latestAnswer()` now reports $\approx 51 \times 10^{18}$ ($\$51.00$ per LP token, a $25.5\times$ inflation).
7. The attacker borrows the maximum available MIM from the Cauldron against their inflated `MagicLP` collateral position.
8. The attacker transfers the USDB received in Step 4 back to `MagicLP` and calls `sellQuote(attacker)`, recovering $\approx 49{,}990{,}000 \times 10^{18}$ MIM, repays the flash loan, and retains the borrowed Cauldron funds as profit.

### Impact assessment

- **Severity:** High / Critical
- **Technical Impact:** `getReserves()` exposes spot reserve values whose sum is not invariant under PMM swaps or direct token synchronization, allowing single-transaction oracle price inflation of `MagicLP` tokens.
- **Business Impact:** Any Cauldron accepting `MagicLP` collateral priced via `MagicLpAggregator` can be drained of all borrowable funds through a flash-loan attack at the cost of only swap fees, leaving the protocol with unbacked bad debt.

### Remediation

Update `getReserves()` (or provide a dedicated oracle reserve getter protected by `nonReadReentrant`) to return the equilibrium target reserves `(state.B0, state.Q0)` computed by `getPMMState()` rather than the raw spot reserves `(_BASE_RESERVE_, _QUOTE_RESERVE_)`, and guard the view functions with `nonReadReentrant` so transient mid-execution states cannot be read during external callbacks:

```solidity
    function getPMMStateForCall() external view nonReadReentrant returns (uint256 i, uint256 K, uint256 B, uint256 Q, uint256 B0, uint256 Q0, uint256 R) {
        PMMPricing.PMMState memory state = getPMMState();
        i = state.i;
        K = state.K;
        B = state.B;
        Q = state.Q;
        B0 = state.B0;
        Q0 = state.Q0;
        R = uint256(state.R);
    }

    function getMidPrice() public view nonReadReentrant returns (uint256 midPrice) {
        return PMMPricing.getMidPrice(getPMMState());
    }

    function getReserves() external view nonReadReentrant returns (uint256 baseReserve, uint256 quoteReserve) {
        PMMPricing.PMMState memory state = getPMMState();
        baseReserve = state.B0;
        quoteReserve = state.Q0;
    }

    function getUserFeeRate(address user) external view nonReadReentrant returns (uint256 lpFeeRate, uint256 mtFeeRate) {
        return _MT_FEE_RATE_MODEL_.getFeeRate(user, _LP_FEE_RATE_);
    }
```

*(Note: If `MagicLP.sync()` remains publicly callable, `sync()` should either be restricted or `MagicLpAggregator` should derive equilibrium reserves without allowing unbacked reserve donations to increase `B0` and `Q0`.)*

## Finding 7: Reserve-Target Desynchronization Across `RState` Transitions in `MagicLP.sellBase`

### Synopsis

In `MagicLP.sellBase`, LP fees retained in the pool and zero-input swaps are not accounted for when updating target reserves across `RState` transitions. Transitioning to or from `RState.ONE` desynchronizes `_BASE_TARGET_` and `_QUOTE_TARGET_` from actual reserves, corrupting `getPMMState()`, `getMidPrice()`, and subsequent swap pricing.

### Technical details

`MagicLP` implements the DODO V2 Proactive Market Maker (PMM) model across three pricing regimes (`PMMPricing.RState.ONE`, `ABOVE_ONE`, and `BELOW_ONE`). The pricing logic in `getPMMState()` and `PMMPricing.adjustedTarget()` relies on strict invariants between the current reserves (`_BASE_RESERVE_`, `_QUOTE_RESERVE_`) and the target reserves (`_BASE_TARGET_`, `_QUOTE_TARGET_`):

1. When `_RState_ == RState.BELOW_ONE`, `B >= B0` and `Q <= Q0`. `PMMPricing.adjustedTarget()` holds `state.B0 = _BASE_TARGET_` fixed and dynamically solves `state.Q0` from `state.Q` and `state.B - state.B0`.
2. When `_RState_ == RState.ABOVE_ONE`, `Q >= Q0` and `B <= B0`. `PMMPricing.adjustedTarget()` holds `state.Q0 = _QUOTE_TARGET_` fixed and dynamically solves `state.B0` from `state.B` and `state.Q - state.Q0`.
3. When `_RState_ == RState.ONE`, `PMMPricing.adjustedTarget()` performs no adjustment at all; it assumes the equilibrium invariant `_BASE_RESERVE_ == _BASE_TARGET_` (`B == B0`) and `_QUOTE_RESERVE_ == _QUOTE_TARGET_` (`Q == Q0`).

In `MagicLP.sellBase` (lines 244–262), the target update logic violates these regime invariants in three interconnected ways:

1. **Missing `_QUOTE_TARGET_` synchronization on `ABOVE_ONE -> ONE` transition:**
   When `_RState_ == RState.ABOVE_ONE` and a trader sells `baseInput == backToOnePayBase` (`state.B0 - state.B`), `PMMPricing.sellBaseToken()` enters Case 2.2, returning `receiveQuoteAmount = backToOneReceiveQuote = state.Q - state.Q0` and `newR = RState.ONE`.
   In `querySellBase()`, `lpFee = DecimalMath.mulFloor(receiveQuoteAmount, lpFeeRate)` and `mtFee` are subtracted from `receiveQuoteAmount`. `sellBase()` then transfers out only `receiveQuoteAmount` and `mtFee` (lines 252–253), leaving `lpFee` inside the contract's quote token balance.
   Consequently, the post-swap quote balance is `state.Q0 + lpFee`. However, at lines 256–260:
   ```solidity
   if (_RState_ != uint32(newRState)) {
       _BASE_TARGET_ = newBaseTarget.toUint112();
       _RState_ = uint32(newRState);
       emit RChange(newRState);
   }
   ```
   `sellBase()` updates `_BASE_TARGET_` to `state.B0` (`== baseBalance`) and sets `_RState_` to `RState.ONE`, but never updates `_QUOTE_TARGET_` to include the retained `lpFee`. `_setReserve()` then sets `_QUOTE_RESERVE_ = state.Q0 + lpFee > _QUOTE_TARGET_`. Because `adjustedTarget()` is a no-op in `RState.ONE`, the pool remains in `RState.ONE` with `_QUOTE_RESERVE_ > _QUOTE_TARGET_`.

2. **Propagating desynchronized targets on `ONE -> BELOW_ONE` transition:**
   By symmetry, when `sellQuote()` transitions `_RState_` from `BELOW_ONE` to `ONE` (`quoteInput == backToOnePayQuote`), the base LP fee (`lpFee_base`) remains in `_BASE_RESERVE_` (`state.B0 + lpFee_base`) while `_BASE_TARGET_` stays at `state.B0`.
   When `sellBase()` is next called while `_RState_ == RState.ONE`, `getPMMState()` does not adjust `state.B0` or `state.Q0`. `querySellBase()` computes `_ROneSellBaseToken` using the understated `state.Q0 = _QUOTE_TARGET_` and returns `newRState = RState.BELOW_ONE` with `newBaseTarget = state.B0` (`_BASE_TARGET_ < _BASE_RESERVE_`).
   `sellBase()` then transitions `_RState_` to `BELOW_ONE` with `_BASE_TARGET_` still excluding `lpFee_base` and `_BASE_RESERVE_ = state.B0 + lpFee_base + baseInput`. In subsequent calls under `BELOW_ONE`, `state.B - state.B0` in `adjustedTarget()` is overstated by `lpFee_base`.

3. **Zero-input regime flip from `ONE` to `BELOW_ONE`:**
   `sellBase()` does not validate that `baseInput > 0` (line 246). When `_RState_ == RState.ONE` and `sellBase()` is called with `baseInput == 0`, `PMMPricing.sellBaseToken()` unconditionally returns `newR = RState.BELOW_ONE` and `receiveQuoteAmount = 0`. Line 256 (`if (_RState_ != uint32(newRState))`) then transitions `_RState_` from `ONE` to `BELOW_ONE` without any swap taking place, immediately activating `BELOW_ONE` target adjustments on any pre-existing `_BASE_RESERVE_ - _BASE_TARGET_` discrepancy.

### Assumptions and preconditions

- The `MagicLP` pool is initialized with valid token addresses, parameters (`_I_`, `_K_`, `_LP_FEE_RATE_ >= MIN_LP_FEE_RATE`), and non-zero liquidity.
- At least one swap moves the pool into `RState.ABOVE_ONE` (via `sellQuote`) or `RState.BELOW_ONE` (via `sellBase`) and subsequently back to `RState.ONE`.

### Proof of concept

1. Initialize a `MagicLP` pool with `1000e18` base tokens and `1000e18` quote tokens (`_I_ = 1e18`, `_LP_FEE_RATE_ = 1e16` (1%), `mtFeeRate = 0`). Initially:
   - `_RState_ = RState.ONE`
   - `_BASE_RESERVE_ = _BASE_TARGET_ = 1000e18`
   - `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1000e18`
2. A trader sells quote tokens via `sellQuote()`, moving `_RState_` to `RState.ABOVE_ONE` with `_QUOTE_RESERVE_ = 1500e18`, `_QUOTE_TARGET_ = 1000e18`, and `adjustedTarget()` solving `state.B0` (e.g., `1000.5e18` while `_BASE_RESERVE_ = 505e18`, so `backToOnePayBase = state.B0 - state.B = 495.5e18` and `backToOneReceiveQuote = state.Q - state.Q0 = 500e18`).
3. A trader transfers `baseInput = backToOnePayBase = 495.5e18` base tokens to the pool and calls `sellBase(trader)`:
   - `PMMPricing.sellBaseToken()` enters Case 2.2 (`payBaseAmount == backToOnePayBase`), returning `receiveQuoteAmount = 500e18` and `newR = RState.ONE`.
   - `querySellBase()` deducts the 1% LP fee (`5e18`), returning `receiveQuoteAmount = 495e18`, `mtFee = 0`, `newRState = RState.ONE`, and `newBaseTarget = 1000.5e18`.
   - `sellBase()` transfers `495e18` quote tokens to `trader`, leaving `1500e18 - 495e18 = 1005e18` quote tokens in the pool.
   - At lines 256–262, `sellBase()` sets `_BASE_TARGET_ = 1000.5e18`, `_RState_ = RState.ONE`, `_BASE_RESERVE_ = 1000.5e18`, and `_QUOTE_RESERVE_ = 1005e18`, while `_QUOTE_TARGET_` remains `1000e18`.
4. The pool is now in `RState.ONE`, but `_QUOTE_RESERVE_` (`1005e18`) exceeds `_QUOTE_TARGET_` (`1000e18`) by `5e18`.
5. On the next `sellBase()` call from `RState.ONE`, `getPMMState()` does not run `adjustedTarget()` because `state.R == RState.ONE`, so `_ROneSellBaseToken` prices the swap against `state.Q0 = 1000e18` instead of `1005e18`. Conversely, if a `BELOW_ONE -> ONE` transition in `sellQuote()` left `_BASE_RESERVE_ > _BASE_TARGET_` in `RState.ONE`, `getMidPrice()` immediately computes `R = (B0^2 / B) / B < 1` at `RState.ONE`, and calling `sellBase()` with `baseInput = 0` flips `_RState_` to `RState.BELOW_ONE` with `state.B - state.B0 = lpFee_base > 0`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Breaking the `B == B0` and `Q == Q0` invariant in `RState.ONE` causes `getPMMState()` to return mismatched reserves and targets whenever `_RState_ == RState.ONE`, distorts `getMidPrice()` and TWAP (`_BASE_PRICE_CUMULATIVE_LAST_`), and causes subsequent swaps transitioning out of `RState.ONE` to use stale target reserves.
- **Business Impact:** Traders receive inaccurate swap quotes around the `R = 1` equilibrium boundary, and any external price consumer or oracle relying on `getPMMState()`, `getMidPrice()`, `_BASE_TARGET_()`, or `_QUOTE_TARGET_()` observes inconsistent pool state.

### Remediation

Prevent zero-input swaps in `sellBase()` from mutating `_RState_`, ensure `RState.ONE` uses the current reserves as targets when starting from `RState.ONE`, and synchronize `_QUOTE_TARGET_` to the post-transfer quote balance whenever `sellBase()` transitions `_RState_` to `RState.ONE`.

```solidity
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

## Finding 8: Unattributed Balance-Derived Input Calculation in `MagicLP.sellBase` Allows Theft of Excess Base Tokens and Positive Rebase Yield

### Synopsis

`MagicLP.sellBase` is a permissionless function that calculates swap input (`baseInput`) as the difference between the contract's current `_BASE_TOKEN_` balance and the cached `_BASE_RESERVE_`. Any unattributed excess `_BASE_TOKEN_` balance—from non-atomic deposits, direct transfers, or positive token rebases—can be swapped and stolen by any caller.

### Technical details

In `MagicLP.sellBase(address to)` (`src/mimswap/MagicLP.sol`), the contract does not pull `_BASE_TOKEN_` from `msg.sender` via `transferFrom` nor restrict access to a trusted router. Instead, lines 245–246 compute the swap input directly from the spot token balance and the stored reserve:

```solidity
uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
```

In line 250, `querySellBase(tx.origin, baseInput)` prices the entire `baseInput` against the PMM state, and in line 252, `_transferQuoteOut(to, receiveQuoteAmount)` transfers the resulting `_QUOTE_TOKEN_` output to the caller-specified `to` address. Finally, line 262 updates `_BASE_RESERVE_` to `baseBalance` and `_QUOTE_RESERVE_` to the remaining quote token balance.

This balance-derived accounting introduces critical flaws whenever the pool's actual `_BASE_TOKEN_` balance diverges from `_BASE_RESERVE_`:

1. **Unattributed Excess-Balance Theft:** Any `_BASE_TOKEN_` transferred to the `MagicLP` contract outside of an atomic router transaction (for example, a user or integration transferring `_BASE_TOKEN_` directly before calling `sellBase` or `buyShares`, or tokens mistakenly sent to the pool, which cannot be recovered via `rescue()` because line 462 reverts with `ErrNotAllowed` for `_BASE_TOKEN_`) creates a positive difference `baseBalance > _BASE_RESERVE_`. Any third party can call `sellBase(attacker)` to consume that excess as their own `baseInput` and receive `_QUOTE_TOKEN_`.
2. **Rebasing and Yield-Induced Reserve Drift:** If `_BASE_TOKEN_` is a rebasing or yield-accruing token (such as native-yield tokens when in rebasing mode or external rebasing ERC20s) whose `balanceOf(address(this))` increases between reserve updates, the accrued balance increase is not automatically credited to liquidity providers via `sync()` or `ratioSync()`. Instead, any arbitrageur can call `sellBase(attacker)` before `sync()`/`ratioSync()` is executed, treating the entire yield/rebase increment as their swap input and extracting its value in `_QUOTE_TOKEN_` from the pool reserves.
3. **Negative Rebase / Deflationary Token DoS:** Conversely, if `_BASE_TOKEN_` undergoes a negative rebase or balance reduction such that `baseBalance < _BASE_RESERVE_` (or if a user's incoming transfer is smaller than a prior negative drift), the subtraction `baseBalance - uint256(_BASE_RESERVE_)` on line 246 reverts due to Solidity 0.8 arithmetic underflow unless the excess added exceeds the deficit.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized and holds active `_BASE_TOKEN_` and `_QUOTE_TOKEN_` liquidity (`_BASE_RESERVE_ > 0` and `_QUOTE_RESERVE_ > 0`).
2. The contract's actual `_BASE_TOKEN_.balanceOf(address(this))` exceeds `_BASE_RESERVE_` due to a positive token rebase/yield accrual prior to `sync()`/`ratioSync()`, an unbundled/non-atomic deposit transfer, or a direct `_BASE_TOKEN_` transfer to the pool address.

### Proof of concept

1. Assume an initialized `MagicLP` pool has `_BASE_RESERVE_ = 100,000e18` and `_QUOTE_RESERVE_ = 100,000e18` with `_I_ = 1e18`.
2. `_BASE_TOKEN_.balanceOf(address(lp))` increases to `101,000e18` (e.g., via a 1% positive rebase on `_BASE_TOKEN_` or a non-atomic transfer of `1,000e18` `_BASE_TOKEN_` to `address(lp)`), while `_BASE_RESERVE_` remains `100,000e18`.
3. An unprivileged caller invokes `lp.sellBase(callerAddress)` without transferring any `_BASE_TOKEN_`.
4. Inside `sellBase`:
   - Line 245 reads `baseBalance = 101,000e18`.
   - Line 246 computes `baseInput = 101,000e18 - 100,000e18 = 1,000e18`.
   - Line 250 calls `querySellBase(tx.origin, 1,000e18)`, returning `receiveQuoteAmount` (~`999.9e18` `_QUOTE_TOKEN_` after fees).
   - Line 252 transfers `receiveQuoteAmount` of `_QUOTE_TOKEN_` to `callerAddress`.
   - Line 262 updates `_BASE_RESERVE_` to `101,000e18` and `_QUOTE_RESERVE_` to the reduced quote balance (~`99,000e18`).
5. The caller receives ~`999.9e18` `_QUOTE_TOKEN_` at zero cost, permanently draining the unattributed `_BASE_TOKEN_` value from the pool / depositor.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Complete loss of any unattributed `_BASE_TOKEN_` balance held by the pool above `_BASE_RESERVE_` (including unbundled user transfers and unsynced positive rebase/yield accruals), which is converted into `_QUOTE_TOKEN_` and transferred to the caller.
- **Business Impact:** Liquidity providers lose accrued rebasing yield on `_BASE_TOKEN_` to arbitrageurs if `sellBase` is called before `ratioSync()`/`sync()`, and users who transfer `_BASE_TOKEN_` directly to the pool outside an atomic transaction lose their funds to front-runners.

### Remediation

Have `sellBase` explicitly pull the exact `amountIn` of `_BASE_TOKEN_` from `msg.sender` (measuring the actual balance delta across the transfer to support fee-on-transfer tokens safely) and enforce a minimum output check, rather than attributing all pre-existing excess balance in the contract to the caller:

```solidity
    function sellBase(address to, uint256 baseInput, uint256 minReceiveQuoteAmount) external nonReentrant returns (uint256 receiveQuoteAmount) {
        uint256 baseBalanceBefore = _BASE_TOKEN_.balanceOf(address(this));
        _BASE_TOKEN_.safeTransferFrom(msg.sender, address(this), baseInput);
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        baseInput = baseBalance - baseBalanceBefore;

        uint256 mtFee;
        uint256 newBaseTarget;
        PMMPricing.RState newRState;
        (receiveQuoteAmount, mtFee, newRState, newBaseTarget) = querySellBase(tx.origin, baseInput);

        if (receiveQuoteAmount < minReceiveQuoteAmount) {
            revert ErrNotEnough();
        }

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

## Finding 9: Inconsistent Target and R-State Updates in `MagicLP.sellBase` and `MagicLP.sellQuote` Violate Reserve-Target Invariants Across Boundary Swaps

### Synopsis

In `MagicLP.sellBase` and `MagicLP.sellQuote`, R-state transitions only update a single pre-fee target variable (`_BASE_TARGET_` or `_QUOTE_TARGET_`) and allow zero-input degenerate transitions from `RState.ONE`. This violates the `RState.ONE` reserve-target invariant after fee retention, causing stale target reuse and mispriced swaps across sequential trades.

### Technical details

In `MagicLP`, the Proactive Market Maker (PMM) curve relies on three regimes tracked by `_RState_`:
- `RState.ONE`: Equilibrium state where reserves match targets (`_BASE_RESERVE_ == _BASE_TARGET_` and `_QUOTE_RESERVE_ == _QUOTE_TARGET_`).
- `RState.BELOW_ONE`: Base token reserve exceeds target (`_BASE_RESERVE_ > _BASE_TARGET_`), and `state.Q0` is dynamically derived in `PMMPricing.adjustedTarget(state)` via `Math._SolveQuadraticFunctionForTarget(state.Q, state.B - state.B0, state.i, state.K)`.
- `RState.ABOVE_ONE`: Quote token reserve exceeds target (`_QUOTE_RESERVE_ > _QUOTE_TARGET_`), and `state.B0` is dynamically derived in `PMMPricing.adjustedTarget(state)` via `Math._SolveQuadraticFunctionForTarget(state.B, state.Q - state.Q0, DecimalMath.reciprocalFloor(state.i), state.K)`.

When `getPMMState()` is called while `_RState_ == RState.ONE`, `PMMPricing.adjustedTarget(state)` performs no adjustment and directly uses the stored `_BASE_TARGET_` (`state.B0`) and `_QUOTE_TARGET_` (`state.Q0`). Furthermore, `_ROneSellBaseToken` and `_ROneSellQuoteToken` compute swap outputs assuming `state.Q == state.Q0` and `state.B == state.B0`, respectively (`Math._SolveQuadraticFunctionForTrade(state.Q0, state.Q0, ...)` and `Math._SolveQuadraticFunctionForTrade(state.B0, state.B0, ...)`).

However, in `sellBase` (lines 256–262) and `sellQuote` (lines 279–285), state and target updates are inconsistent with actual post-swap reserves:

1. **Reserve-Target Invariant Violation on Boundary Transitions to `RState.ONE`**:
   When the pool is in `RState.ABOVE_ONE` and a trader calls `sellBase` with `baseInput == backToOnePayBase` (`state.B0 - state.B`), `PMMPricing.sellBaseToken` returns `rawReceiveQuoteAmount = state.Q - state.Q0` and `newRState = RState.ONE`. `querySellBase` then deducts `lpFee` and `mtFee` from `receiveQuoteAmount`. Because `lpFee` remains in the pool, the post-swap quote balance is `state.Q0 + lpFee`.
   At lines 256–262:
   ```solidity
   if (_RState_ != uint32(newRState)) {
       _BASE_TARGET_ = newBaseTarget.toUint112();
       _RState_ = uint32(newRState);
       emit RChange(newRState);
   }

   _setReserve(baseBalance, _QUOTE_TOKEN_.balanceOf(address(this)));
   ```
   `sellBase` updates `_BASE_TARGET_` to `state.B0` and sets `_RState_` to `RState.ONE`, but leaves `_QUOTE_TARGET_` at the stale value `state.Q0` while `_QUOTE_RESERVE_` is set to `state.Q0 + lpFee`. Symmetrically, when `sellQuote` transitions from `RState.BELOW_ONE` to `RState.ONE` (`quoteInput == state.Q0 - state.Q`), `_BASE_RESERVE_` becomes `state.B0 + lpFee` while `_BASE_TARGET_` remains at the stale value `state.B0`. Contrast this with `correctRState()` (lines 140–142, 145–147) and `_resetTargetAndReserve()` (lines 536–540), which synchronize both `_BASE_TARGET_` and `_QUOTE_TARGET_` to the current reserves whenever setting `_RState_ = RState.ONE`.

2. **Stale Target Reuse and Degenerate Transitions Across Sequential Swaps**:
   Once the pool is in `RState.ONE` with a reserve surplus (`_QUOTE_RESERVE_ = Q0 + lpFee > _QUOTE_TARGET_` after `sellBase`, or `_BASE_RESERVE_ = B0 + lpFee > _BASE_TARGET_` after `sellQuote`):
   - `getPMMState()` does not run `adjustedTarget()` in `RState.ONE`, so the next swap reuses the stale target.
   - Calling `sellBase` with `baseInput == 0` (or a dust input) when `_BASE_RESERVE_ = B0 + lpFee > _BASE_TARGET_` in `RState.ONE` unconditionally transitions `_RState_` to `RState.BELOW_ONE` while locking in the stale `_BASE_TARGET_ = B0`.
   - On the subsequent `getPMMState()` call in `RState.BELOW_ONE`, `adjustedTarget()` sees `state.B - state.B0 = lpFee` and computes `state.Q0 = _SolveQuadraticFunctionForTarget(Q0, lpFee, i, K)`. By the concavity of the square root in `_SolveQuadraticFunctionForTarget`, `state.Q0 - Q0 < i * lpFee`. Consequently, the pool treats the accumulated `lpFee` surplus as an excess base reserve below equilibrium (`R < 1`) and sells those `lpFee` base tokens at a discounted price below the oracle price `i`.

3. **Unaccounted LP Fees on Boundary-Crossing Transitions (`ABOVE_ONE -> BELOW_ONE` and `BELOW_ONE -> ABOVE_ONE`)**:
   When `sellBase` crosses directly from `RState.ABOVE_ONE` to `RState.BELOW_ONE` (`baseInput > backToOnePayBase`), `PMMPricing.sellBaseToken` prices the second leg `_ROneSellBaseToken(state, payBaseAmount - backToOnePayBase)` using the pre-fee `state.Q0` rather than accounting for the LP fee retained during the `ABOVE_ONE -> ONE` leg, and if the second leg's output is smaller than the first leg's retained LP fee, the pool enters `RState.BELOW_ONE` with both `_BASE_RESERVE_ > _BASE_TARGET_` and `_QUOTE_RESERVE_ > _QUOTE_TARGET_`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized and has active liquidity with a non-zero LP fee rate (`_LP_FEE_RATE_ >= MIN_LP_FEE_RATE`).
2. The pool is in `RState.BELOW_ONE` or `RState.ABOVE_ONE` following one or more swaps.
3. Any unprivileged user can call `sellBase` and `sellQuote` directly on the `MagicLP` contract.

### Proof of concept

1. **Initial Imbalance (`RState.ONE -> RState.BELOW_ONE`)**:
   - Assume a pool has `_BASE_RESERVE_ = _BASE_TARGET_ = 1000e18`, `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1000e18`, `_I_ = 1e18`, `_K_ = 1e16` (1%), and `lpFeeRate = 1e16` (1%, with `mtFeeRate = 0`).
   - A trader sells `500e18` base tokens via `sellBase`. The pool transitions to `_RState_ = RState.BELOW_ONE` with `_BASE_TARGET_ = 1000e18` and `_BASE_RESERVE_ = 1500e18`.

2. **Boundary Return to `RState.ONE` (`RState.BELOW_ONE -> RState.ONE`)**:
   - In `getPMMState()`, `state.Q0` is dynamically computed from `state.Q` and `state.B - state.B0 = 500e18`.
   - A trader transfers exactly `backToOnePayQuote = state.Q0 - state.Q` quote tokens to the pool and calls `sellQuote`.
   - `PMMPricing.sellQuoteToken` hits the `payQuoteAmount == backToOnePayQuote` branch, returning `rawReceiveBaseAmount = state.B - state.B0 = 500e18` and `newR = RState.ONE`.
   - `querySellQuote` deducts the 1% LP fee (`5e18` base tokens), so `receiveBaseAmount = 495e18` base tokens are transferred out.
   - `sellQuote` updates `_QUOTE_TARGET_ = state.Q0` and `_RState_ = RState.ONE`, and sets `_BASE_RESERVE_ = 1500e18 - 495e18 = 1005e18`, while `_BASE_TARGET_` remains at the stale value `1000e18`.

3. **Degenerate Transition Locking Stale Target (`RState.ONE -> RState.BELOW_ONE`)**:
   - An arbitrageur calls `sellBase` without transferring any tokens (`baseInput = 0`).
   - Because `_RState_ == RState.ONE`, `getPMMState()` does not update `state.B0` (`1000e18`), and `PMMPricing.sellBaseToken(state, 0)` returns `receiveQuoteAmount = 0` and `newRState = RState.BELOW_ONE`.
   - `sellBase` sets `_BASE_TARGET_ = 1000e18` and `_RState_ = RState.BELOW_ONE`, with `_BASE_RESERVE_ = 1005e18` and `_QUOTE_RESERVE_ = state.Q0`.

4. **Discounted Arbitrage of LP Fee Surplus**:
   - In `RState.BELOW_ONE`, `getPMMState()` now computes `state.Q0 = _SolveQuadraticFunctionForTarget(state.Q, 1005e18 - 1000e18, 1e18, 1e16)`, where `backToOnePayQuote = state.Q0 - state.Q < 5e18` while `backToOneReceiveBase = state.B - state.B0 = 5e18`.
   - The arbitrageur calls `sellQuote` with `backToOnePayQuote` quote tokens to extract the `5e18` base tokens at a below-peg price (`< 1e18`).

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: The `RState.ONE` invariant (`_BASE_RESERVE_ == _BASE_TARGET_` and `_QUOTE_RESERVE_ == _QUOTE_TARGET_`) is broken whenever a swap returns the pool to `RState.ONE`, and zero-input swaps can flip `_RState_` away from `RState.ONE` while locking in stale targets.
- **Business Impact**: Accumulated LP fees retained on boundary-returning swaps are not incorporated into the pool's target reserves at `RState.ONE`. Arbitrageurs can trigger a zero-input state flip and purchase the retained fee surplus at a discounted `R < 1` or `R > 1` price, leaking value from liquidity providers.

### Remediation

Prevent zero-input swaps in `sellBase` and `sellQuote`, and ensure that when transitioning to `RState.ONE`, both `_BASE_TARGET_` and `_QUOTE_TARGET_` are synchronized with the actual post-swap reserves (as done in `correctRState` and `_resetTargetAndReserve`).

```solidity
    function sellBase(address to) external nonReentrant returns (uint256 receiveQuoteAmount) {
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
        if (baseInput == 0) {
            return 0;
        }
        uint256 mtFee;
        uint256 newBaseTarget;
        PMMPricing.RState newRState;
        (receiveQuoteAmount, mtFee, newRState, newBaseTarget) = querySellBase(tx.origin, baseInput);

        _transferQuoteOut(to, receiveQuoteAmount);
        _transferQuoteOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);

        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));

        // update TARGET
        if (_RState_ != uint32(newRState)) {
            if (newRState == PMMPricing.RState.ONE) {
                _BASE_TARGET_ = baseBalance.toUint112();
                _QUOTE_TARGET_ = quoteBalance.toUint112();
            } else {
                _BASE_TARGET_ = newBaseTarget.toUint112();
            }
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }

        _setReserve(baseBalance, quoteBalance);

        emit Swap(address(_BASE_TOKEN_), address(_QUOTE_TOKEN_), baseInput, receiveQuoteAmount, msg.sender, to);
    }

    function sellQuote(address to) external nonReentrant returns (uint256 receiveBaseAmount) {
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
        uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
        if (quoteInput == 0) {
            return 0;
        }
        uint256 mtFee;
        uint256 newQuoteTarget;
        PMMPricing.RState newRState;
        (receiveBaseAmount, mtFee, newRState, newQuoteTarget) = querySellQuote(tx.origin, quoteInput);

        _transferBaseOut(to, receiveBaseAmount);
        _transferBaseOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);

        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));

        // update TARGET
        if (_RState_ != uint32(newRState)) {
            if (newRState == PMMPricing.RState.ONE) {
                _BASE_TARGET_ = baseBalance.toUint112();
                _QUOTE_TARGET_ = quoteBalance.toUint112();
            } else {
                _QUOTE_TARGET_ = newQuoteTarget.toUint112();
            }
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }

        _setReserve(baseBalance, quoteBalance);

        emit Swap(address(_QUOTE_TOKEN_), address(_BASE_TOKEN_), quoteInput, receiveBaseAmount, msg.sender, to);
    }
```

## Finding 10: Unchecked Target Growth Beyond `uint112` in `MagicLP.sellBase` and `MagicLP.sellQuote` Causes Swap DoS on `RState` Transitions

### Synopsis

In `MagicLP.sellBase` (and `MagicLP.sellQuote`), the dynamically computed target `newBaseTarget` (`newQuoteTarget`) is calculated as a `uint256` in `PMMPricing.adjustedTarget`, but narrowed via `.toUint112()` only when `_RState_` changes. When the computed target exceeds `type(uint112).max`, any swap attempting to transition `_RState_` reverts with `Overflow()`.

### Technical details

In `MagicLP`, the pool reserves (`_BASE_RESERVE_`, `_QUOTE_RESERVE_`) and target reserves (`_BASE_TARGET_`, `_QUOTE_TARGET_`) are stored as `uint112` state variables, while the pricing engine (`PMMPricing.PMMState`, `querySellBase`, and `querySellQuote`) computes and propagates targets (`B0`, `Q0`, `newBaseTarget`, `newQuoteTarget`) as `uint256`.

When `sellBase` is called (`src/mimswap/MagicLP.sol:244`), it invokes `querySellBase(tx.origin, baseInput)`, which calls `getPMMState()` and `PMMPricing.adjustedTarget(state)`:

```solidity
} else if (state.R == RState.ABOVE_ONE) {
    state.B0 = Math._SolveQuadraticFunctionForTarget(
        state.B,
        state.Q - state.Q0,
        DecimalMath.reciprocalFloor(state.i),
        state.K
    );
}
```

And symmetrically, when `state.R == RState.BELOW_ONE`, `adjustedTarget` computes `state.Q0` using `state.i`:

```solidity
if (state.R == RState.BELOW_ONE) {
    state.Q0 = Math._SolveQuadraticFunctionForTarget(state.Q, state.B - state.B0, state.i, state.K);
}
```

Inside `Math._SolveQuadraticFunctionForTarget`, the virtual target `V0` is derived by scaling the excess reserve `delta` (`state.Q - state.Q0` or `state.B - state.B0`) by the price factor (`DecimalMath.reciprocalFloor(state.i)` or `state.i`) and adding it to `V1`.

A storage-width incompatibility arises because:
1. While `_RState_` remains unchanged (e.g., trades within `RState.ABOVE_ONE` where `payBaseAmount < backToOnePayBase`, or repeated `sellQuote` calls that keep `_RState_ == RState.ABOVE_ONE`), the `if (_RState_ != uint32(newRState))` branch at line 256 is not entered. Only `_setReserve` is executed, which checks that the actual token balances fit within `uint112`, without checking whether the implied target `state.B0` (or `state.Q0`) fits within `uint112`.
2. `newBaseTarget` (`state.B0`) can exceed `type(uint112).max` even when both `state.B <= type(uint112).max` and `state.Q <= type(uint112).max`:
   - When `_I_ < 1e18` (so `DecimalMath.reciprocalFloor(_I_) > 1e18`, up to `10^36`), the quote reserve delta `state.Q - state.Q0` is amplified by `10^18 / _I_` in `_SolveQuadraticFunctionForTarget`, allowing a `uint112`-bounded quote reserve to produce a `state.B0` value far above `type(uint112).max` (and symmetrically for `newQuoteTarget` when `_I_ > 1e18` in `RState.BELOW_ONE`).
   - Even when `_I_ == 1e18`, accumulated LP fees retained in the pool while in `RState.ABOVE_ONE` (or reserve increases synced via `sync()`) increase `state.B + (state.Q - state.Q0)` above `type(uint112).max` while `state.B` and `state.Q` each remain below `type(uint112).max`.
3. Once `newBaseTarget > type(uint112).max`, any call to `sellBase` that supplies `payBaseAmount >= backToOnePayBase` changes `newRState` to `RState.ONE` or `RState.BELOW_ONE`, entering the `if (_RState_ != uint32(newRState))` block and executing `_BASE_TARGET_ = newBaseTarget.toUint112()` at line 257, which reverts with `SafeCastLib.Overflow()`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with valid parameters (`_I_`, `_K_`, `_LP_FEE_RATE_`) and is in `RState.ABOVE_ONE` (for `sellBase`) or `RState.BELOW_ONE` (for `sellQuote`).
2. Either the oracle price `_I_` deviates from `1e18` (e.g., `_I_ < 1e18` due to token price or decimal differences so `reciprocalFloor(_I_) > 1e18`, or `_I_ > 1e18` in `BELOW_ONE`), or the pool's reserves and accumulated fees while in `ABOVE_ONE` / `BELOW_ONE` are large enough that the solved quadratic target exceeds `type(uint112).max` while individual token balances remain `<= type(uint112).max`.

### Proof of concept

1. Consider a `MagicLP` pool with `_I_ = 1e12` (e.g., a base token with 18 decimals and a quote token with 6 decimals, which passes `Router._validateDecimals` since `quoteDecimals` vs `baseDecimals` difference is within 12, or a pair where 1 quote unit equals `1e6` base units) and `_K_ = 0` (or small `K` such as `1e14`). Here, `DecimalMath.reciprocalFloor(_I_) = 1e24`.
2. The pool is initialized at `RState.ONE` with `_BASE_RESERVE_ = _BASE_TARGET_ = 1e30` and `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1e18` (both well within `type(uint112).max ≈ 5.19e33`).
3. A swap in `sellQuote` transitions the pool to `RState.ABOVE_ONE` (`_RState_ = 1`).
4. While in `RState.ABOVE_ONE`, quote tokens are added to the pool (via swaps with LP fees or a transfer followed by `sync()`) such that `_QUOTE_RESERVE_ - _QUOTE_TARGET_ = 6e27` (still well below `type(uint112).max`).
5. In `getPMMState()`, `PMMPricing.adjustedTarget` computes:
   `state.B0 = state.B + DecimalMath.mulFloor(1e24, 6e27) ≈ 6e33 > type(uint112).max (5.192e33)`.
6. Small `sellBase` trades where `baseInput < state.B0 - state.B` keep `newRState == RState.ABOVE_ONE`, skipping line 257 and succeeding.
7. When a trader attempts to sell enough base tokens (`baseInput >= state.B0 - state.B`) to return the pool to `RState.ONE` or `RState.BELOW_ONE`, `_RState_ != uint32(newRState)` evaluates to `true` and `_BASE_TARGET_ = newBaseTarget.toUint112()` at line 257 reverts with `Overflow()`, locking the pool in `RState.ABOVE_ONE`.

### Impact assessment

- **Severity**: Medium
- **Technical Effect**: Swaps in `sellBase` (or `sellQuote`) that transition `_RState_` out of `ABOVE_ONE` (or `BELOW_ONE`) revert due to `uint112` cast overflow on `newBaseTarget` (or `newQuoteTarget`).
- **Business Impact**: The pool becomes stuck in a single `RState` regime and cannot process rebalancing swaps across the equilibrium point `R = 1` unless liquidity providers remove liquidity or call `_resetTargetAndReserve` via `buyShares`.

### Remediation

Widen `_BASE_TARGET_` and `_QUOTE_TARGET_` to `uint256` (or remove the narrowing `uint112` truncation by storing targets as `uint256`) so that virtual targets computed by `PMMPricing.adjustedTarget` do not revert when `_RState_` transitions, or cap the target to `type(uint112).max` if `uint112` storage layout must be preserved.

```solidity
        // update TARGET
        if (_RState_ != uint32(newRState)) {
            _BASE_TARGET_ = (newBaseTarget > type(uint112).max ? type(uint112).max : newBaseTarget).toUint112();
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }
```

## Finding 11: Unsynchronized Balance-Delta Accounting in `MagicLP.sellBase` and `MagicLP.sellQuote` Causes Reserve-Target Drift and Surplus Consumption

### Synopsis

`MagicLP.sellBase` and `MagicLP.sellQuote` rely on raw `_QUOTE_TOKEN_.balanceOf(address(this))` balance-delta accounting to update reserves and compute swap inputs. Any unattributed quote balance delta (from rebasing/yield accrual, pre-transferred tokens, or self-recipient swaps) desynchronizes `_QUOTE_RESERVE_` from `_QUOTE_TARGET_` or allows unauthorized extraction of surplus tokens.

### Technical details

In `src/mimswap/MagicLP.sol`, swap accounting in `sellBase` and `sellQuote` is implemented by comparing current ERC20 balances against the cached `_BASE_RESERVE_` and `_QUOTE_RESERVE_` state variables and overwriting both reserves with spot balances at the end of every swap:

1. In `sellBase(address to)`:
   - `querySellBase(tx.origin, baseInput)` (line 250) calls `getPMMState()`, which loads `state.Q = _QUOTE_RESERVE_` and `state.Q0 = _QUOTE_TARGET_` rather than the actual `_QUOTE_TOKEN_.balanceOf(address(this))`.
   - If the pool transitions across `RState` boundaries, lines 257–260 update `_BASE_TARGET_` and `_RState_` based on `state.Q = _QUOTE_RESERVE_`.
   - At line 262, `_setReserve(baseBalance, _QUOTE_TOKEN_.balanceOf(address(this)))` overwrites `_QUOTE_RESERVE_` with the current spot balance `_QUOTE_TOKEN_.balanceOf(address(this))` rather than `_QUOTE_RESERVE_ - receiveQuoteAmount - mtFee`.

2. In `sellQuote(address to)`:
   - Lines 268–269 compute the swap input directly as the difference between the spot balance and the cached reserve:
     ```solidity
     uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
     uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
     ```
   - Line 273 then passes `quoteInput` into `querySellQuote(tx.origin, quoteInput)`.

This raw balance-delta design creates a critical accounting inconsistency whenever `_QUOTE_TOKEN_.balanceOf(address(this))` diverges from `_QUOTE_RESERVE_` outside of a single isolated swap:

- **Rebasing / Accrued-Yield and Unattributed Surplus Consumption:** When `_QUOTE_TOKEN_` accrues yield in balance (e.g., a native yield token on Blast before `updateTokenClaimables()` is executed on the clone, or any rebasing token) or when quote tokens are transferred into the pool prior to a `sellBase` call:
  - Calling `sellQuote(to)` (lines 268–273) treats the entire positive balance delta (`quoteBalance - _QUOTE_RESERVE_`) as `quoteInput` contributed by the caller, allowing any caller to extract `_BASE_TOKEN_` for free without supplying quote tokens.
  - Conversely, if `sellBase(to)` is executed first (or if `to == address(this)`, such as a multi-hop path in `Router._swap` where `path[i] == path[i + 1]`), `querySellBase` and the target updates at lines 257–260 are calculated using the stale `_QUOTE_RESERVE_`, and line 262 (`_setReserve(baseBalance, _QUOTE_TOKEN_.balanceOf(address(this)))`) silently absorbs the entire excess `_QUOTE_TOKEN_` balance into `_QUOTE_RESERVE_` without updating `_QUOTE_TARGET_`. A subsequent `sellQuote` call (lines 268–269) then observes `quoteBalance - _QUOTE_RESERVE_ == 0`, swallowing the tokens into `_QUOTE_RESERVE_` while leaving `_QUOTE_RESERVE_`, `_QUOTE_TARGET_`, and `_RState_` out of sync.
- **Deficit / Fee-on-Transfer Desynchronization:** Conversely, if `_QUOTE_TOKEN_` experiences a balance deficit or transfer fee on outbound transfer, line 262 sets `_QUOTE_RESERVE_` below `_QUOTE_TARGET_` while `_RState_ == ABOVE_ONE`, causing subsequent swaps to revert with arithmetic underflow in `PMMPricing.adjustedTarget` (`state.Q - state.Q0`) and `sellQuote` (`quoteBalance - _QUOTE_RESERVE_`).

### Assumptions and preconditions

- A `MagicLP` (or `BlastMagicLP`) pool is deployed and initialized with liquidity.
- Either `_QUOTE_TOKEN_` accrues a positive balance delta (e.g., rebasing token or Blast native yield token prior to `updateTokenClaimables()`), tokens are pre-transferred to the pool before `sellBase` is invoked, or `sellBase` is called with `to == address(lp)` (directly or via `Router._swap` with repeated pool addresses in `path`).

### Proof of concept

1. Assume a `MagicLP` pool is initialized with `1,000e18` `_BASE_TOKEN_` and `1,000e18` `_QUOTE_TOKEN_`, so `_BASE_RESERVE_ = 1,000e18`, `_QUOTE_RESERVE_ = 1,000e18`, `_BASE_TARGET_ = 1,000e18`, `_QUOTE_TARGET_ = 1,000e18`, and `_RState_ = RState.ONE`.
2. **Case A (Unattributed Surplus Extraction via `sellQuote`):**
   - `_QUOTE_TOKEN_` accrues `50e18` in yield (or `50e18` `_QUOTE_TOKEN_` is present above `_QUOTE_RESERVE_`), bringing `_QUOTE_TOKEN_.balanceOf(address(lp))` to `1,050e18` while `_QUOTE_RESERVE_` remains `1,000e18`.
   - An unprivileged caller invokes `lp.sellQuote(caller)` without transferring any tokens.
   - At lines 268–269, `quoteBalance = 1,050e18` and `quoteInput = 1,050e18 - 1,000e18 = 50e18`.
   - At line 273, `querySellQuote` calculates `receiveBaseAmount` (~`50e18` minus fees) for `50e18` quote input, and the pool transfers ~`50e18` `_BASE_TOKEN_` to `caller`.
3. **Case B (Reserve Overwrite in `sellBase` Swallowing Quote Tokens and Desynchronizing Targets):**
   - A caller executes a two-step swap or calls `Router.swapTokensForTokens` with `path = [lp, lp]` and `directions = 2` (binary `10`: `sellBase` then `sellQuote`) with `100e18` `_BASE_TOKEN_`.
   - In the first step, `lp.sellBase(address(lp))` transfers `receiveQuoteAmount` (~`100e18` `_QUOTE_TOKEN_`) to `address(lp)`.
   - Lines 257–258 update `_BASE_TARGET_ = 1,000e18` and `_RState_ = BELOW_ONE` assuming ~`100e18` `_QUOTE_TOKEN_` left the pool.
   - Line 262 executes `_setReserve(baseBalance, _QUOTE_TOKEN_.balanceOf(address(this)))`, setting `_QUOTE_RESERVE_` to ~`1,000e18` (retaining the `receiveQuoteAmount` that was sent to `address(lp)`).
   - In the second step, `lp.sellQuote(to)` executes lines 268–269: `quoteBalance - uint256(_QUOTE_RESERVE_)` evaluates to `0`, returning `0` `_BASE_TOKEN_` to the user while permanently locking the user's `100e18` input in the pool with corrupted `_RState_` and target values.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:** Unattributed `_QUOTE_TOKEN_` balance increases can be stolen by any caller via `sellQuote`, while `sellBase` overwrites `_QUOTE_RESERVE_` with `_QUOTE_TOKEN_.balanceOf(address(this))` after computing PMM state transitions on the old `_QUOTE_RESERVE_`, swallowing pre-existing or self-transferred quote balances and corrupting the PMM relation between reserves, targets, and `_RState_`.
- **Business Implications:** Liquidity providers lose accrued token yield or surplus balances to arbitrageurs, and users or integrators executing multi-hop swaps or batched operations lose funds and distort pool pricing.

### Remediation

Validate that `to != address(this)` in `sellBase` and `sellQuote`, and update the outbound token reserve deterministically from the tracked reserve minus the transferred output and maintainer fee (or explicitly synchronize reserves and targets before computing PMM pricing):

```solidity
        if (to == address(this)) {
            revert ErrNotAllowed();
        }

        // update TARGET
        if (_RState_ != uint32(newRState)) {
            _BASE_TARGET_ = newBaseTarget.toUint112();
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }

        _setReserve(baseBalance, uint256(_QUOTE_RESERVE_) - receiveQuoteAmount - mtFee);

        emit Swap(address(_BASE_TOKEN_), address(_QUOTE_TOKEN_), baseInput, receiveQuoteAmount, msg.sender, to);
    }

    function sellQuote(address to) external nonReentrant returns (uint256 receiveBaseAmount) {
        if (to == address(this)) {
            revert ErrNotAllowed();
        }
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
        uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
        uint256 mtFee;
        uint256 newQuoteTarget;
        PMMPricing.RState newRState;
        (receiveBaseAmount, mtFee, newRState, newQuoteTarget) = querySellQuote(tx.origin, quoteInput);
```

## Finding 12: State Updates After External Token Transfers and Stale Balance Snapshots in `MagicLP.sellBase` and `MagicLP.sellQuote` Expose Partially Settled Pool State

### Synopsis

In `MagicLP.sellBase` and `MagicLP.sellQuote`, external token transfers (`_transferQuoteOut` / `_transferBaseOut`) are executed before updating `_BASE_TARGET_` / `_QUOTE_TARGET_`, `_RState_`, and pool reserves via `_setReserve`, while mixing pre-transfer balance snapshots with post-transfer balance reads, exposing partially settled state and invalidating balance snapshots during transfer callbacks.

### Technical details

In `src/mimswap/MagicLP.sol`, both `sellBase` (lines 244–265) and `sellQuote` (lines 267–288) settle swaps using the following sequence:

1. A pre-transfer balance snapshot of the input token is recorded (`uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this))` at line 245 in `sellBase`, and `uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this))` at line 268 in `sellQuote`).
2. The swap output and new PMM state parameters (`receiveQuoteAmount` / `receiveBaseAmount`, `mtFee`, `newRState`, `newBaseTarget` / `newQuoteTarget`) are computed via `querySellBase` (line 250) or `querySellQuote` (line 273).
3. External token transfers are executed via `_transferQuoteOut(to, receiveQuoteAmount)` and `_transferQuoteOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee)` (lines 252–253), or `_transferBaseOut` (lines 275–276).
4. Only after these external interactions complete does the contract update `_BASE_TARGET_` / `_QUOTE_TARGET_` and `_RState_` (lines 256–260 and 279–283) and call `_setReserve` (lines 262 and 285):
   ```solidity
   if (_RState_ != uint32(newRState)) {
       _BASE_TARGET_ = newBaseTarget.toUint112();
       _RState_ = uint32(newRState);
       emit RChange(newRState);
   }

   _setReserve(baseBalance, _QUOTE_TOKEN_.balanceOf(address(this)));
   ```

This settlement order introduces two tightly coupled flaws during external token transfer callbacks (e.g., ERC-777, ERC-1363, or tokens with transfer hooks) or self-referential transfers:

- **Unsettled State Exposure During Callbacks:** When `_transferQuoteOut` or `_transferBaseOut` invokes the token contract and triggers a callback to `to`, the pool's actual ERC-20 balances already reflect the incoming input tokens and outgoing output tokens, but `_BASE_RESERVE_`, `_QUOTE_RESERVE_`, `_BASE_TARGET_`, `_QUOTE_TARGET_`, `_RState_`, and `_BASE_PRICE_CUMULATIVE_LAST_` (updated inside `_setReserve` -> `_twapUpdate`) still hold their stale pre-swap values. Because view functions (`getReserves`, `getPMMState`, `querySellBase`, `querySellQuote`) and `correctRState()` do not check the reentrancy lock (`nonReadReentrant` / `nonReentrant`), any contract querying the pool or periphery (`Router.previewAddLiquidity`, `Router.previewRemoveLiquidity`, `MagicLpAggregator`, or external integrations) during the callback observes an inconsistent, partially settled state.
- **Invalidated Pre-Transfer Balance Snapshots:** `_setReserve` at line 262 combines the pre-interaction snapshot `baseBalance` (taken at line 245) with a post-interaction live query `_QUOTE_TOKEN_.balanceOf(address(this))` (and symmetrically `quoteBalance` from line 268 with `_BASE_TOKEN_.balanceOf(address(this))` at line 285). Any balance modification during the external transfer interaction—or passing `to == address(this)` so that the transferred output tokens remain in the pool—causes the recorded reserves and PMM target state (`_BASE_TARGET_` / `_QUOTE_TARGET_` and `_RState_`) to desynchronize from actual swap deltas.

### Assumptions and preconditions

1. A `MagicLP` pool is deployed and initialized with liquidity.
2. Either `_BASE_TOKEN_` or `_QUOTE_TOKEN_` supports transfer callbacks/hooks (such as ERC-777 or ERC-1363) allowing execution control to pass to `to` during `_transferQuoteOut` / `_transferBaseOut`, or a caller invokes `sellBase` / `sellQuote` in a workflow that interacts with the pool's balances or state before settlement completes.

### Proof of concept

1. A pool is active with `_BASE_RESERVE_ = 1000e18`, `_QUOTE_RESERVE_ = 1000e18`, `_BASE_TARGET_ = 1000e18`, `_QUOTE_TARGET_ = 1000e18`, and `_RState_ = RState.ONE`.
2. An attacker transfers `500e18` of `_BASE_TOKEN_` to the `MagicLP` contract and calls `sellBase(attacker)`.
3. `sellBase` snapshots `baseBalance = 1500e18` (line 245), calculates `receiveQuoteAmount` and `newRState = RState.BELOW_ONE` (line 250), and calls `_transferQuoteOut(attacker, receiveQuoteAmount)` (line 252).
4. During the token transfer callback to `attacker`:
   - `_BASE_TOKEN_.balanceOf(address(lp))` is `1500e18` and `_QUOTE_TOKEN_.balanceOf(address(lp))` is `~500e18`.
   - However, lines 257–262 have not executed yet: `_BASE_RESERVE_` is still `1000e18`, `_QUOTE_RESERVE_` is still `1000e18`, and `_RState_` is still `RState.ONE`.
   - Any protocol or periphery contract queried by `attacker` during this callback reads stale reserves (`1000e18`, `1000e18`) and stale `_RState_` (`ONE`) while actual token balances are (`1500e18`, `500e18`), or if `attacker` transfers additional `_BASE_TOKEN_` into the pool during the callback, line 262 overwrites `_BASE_RESERVE_` with the stale pre-callback `baseBalance` (`1500e18`) instead of the actual post-callback balance.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** During swap settlement callbacks, external consumers of `MagicLP` state (`getReserves`, `getPMMState`, `getMidPrice`, `Router` liquidity calculations) read stale reserves, targets, TWAP prices, and `RState` that do not match the pool's actual ERC-20 balances, and any input-token balance changes during the callback are ignored by `_setReserve` due to using the pre-transfer `baseBalance` / `quoteBalance` snapshot.
- **Business Impact:** Dependent contracts and periphery functions relying on `MagicLP` reserve/target invariants or balance-to-reserve synchronization can be manipulated during swap callbacks, leading to mispriced LP operations or corrupted reserve accounting.

### Remediation

Follow the Checks-Effects-Interactions (CEI) pattern by computing the new reserves and updating `_BASE_TARGET_` / `_QUOTE_TARGET_`, `_RState_`, and `_setReserve` before performing external token transfers, or re-read both token balances after the external transfers complete and protect state-reading functions with `nonReadReentrant`.

```solidity
    function sellBase(address to) external nonReentrant returns (uint256 receiveQuoteAmount) {
        if (to == address(this)) {
            revert ErrNotAllowed();
        }
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
        uint256 baseInput = baseBalance - uint256(_BASE_RESERVE_);
        uint256 mtFee;
        uint256 newBaseTarget;
        PMMPricing.RState newRState;
        (receiveQuoteAmount, mtFee, newRState, newBaseTarget) = querySellBase(tx.origin, baseInput);

        // update TARGET and RESERVE before external calls
        if (_RState_ != uint32(newRState)) {
            _BASE_TARGET_ = newBaseTarget.toUint112();
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }

        _setReserve(baseBalance, quoteBalance - receiveQuoteAmount - mtFee);

        _transferQuoteOut(to, receiveQuoteAmount);
        _transferQuoteOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);

        emit Swap(address(_BASE_TOKEN_), address(_QUOTE_TOKEN_), baseInput, receiveQuoteAmount, msg.sender, to);
    }

    function sellQuote(address to) external nonReentrant returns (uint256 receiveBaseAmount) {
        if (to == address(this)) {
            revert ErrNotAllowed();
        }
        uint256 baseBalance = _BASE_TOKEN_.balanceOf(address(this));
        uint256 quoteBalance = _QUOTE_TOKEN_.balanceOf(address(this));
        uint256 quoteInput = quoteBalance - uint256(_QUOTE_RESERVE_);
        uint256 mtFee;
        uint256 newQuoteTarget;
        PMMPricing.RState newRState;
        (receiveBaseAmount, mtFee, newRState, newQuoteTarget) = querySellQuote(tx.origin, quoteInput);

        // update TARGET and RESERVE before external calls
        if (_RState_ != uint32(newRState)) {
            _QUOTE_TARGET_ = newQuoteTarget.toUint112();
            _RState_ = uint32(newRState);
            emit RChange(newRState);
        }

        _setReserve(baseBalance - receiveBaseAmount - mtFee, quoteBalance);

        _transferBaseOut(to, receiveBaseAmount);
        _transferBaseOut(_MT_FEE_RATE_MODEL_.maintainer(), mtFee);

        emit Swap(address(_QUOTE_TOKEN_), address(_BASE_TOKEN_), quoteInput, receiveBaseAmount, msg.sender, to);
    }
```

## Finding 13: Precision Loss in `PMMPricing.sellBaseToken` Target Reconstruction Allows Draining Base Reserves and Ratcheting Down `_BASE_TARGET_`

### Synopsis

In `PMMPricing.sellBaseToken` (lines 51–59), selling base tokens while in `RState.ABOVE_ONE` relies on a dynamically reconstructed target `state.B0` from `adjustedTarget`. Truncation in `DecimalMath.reciprocalFloor(state.i)` and `Math._SolveQuadraticFunctionForTarget` underestimates `state.B0` (down to `state.B`), enabling repeated swaps to underpay `backToOnePayBase`, drain reserves, and permanently lower `_BASE_TARGET_`.

### Technical details

When a `MagicLP` pool is in `RState.ABOVE_ONE` (`state.Q > state.Q0` and `state.B < state.B0`), `MagicLP.querySellBase` calls `getPMMState()`, which invokes `PMMPricing.adjustedTarget(state)` (lines 190–201 of `src/mimswap/libraries/PMMPricing.sol`) instead of using the stored `_BASE_TARGET_`:

```solidity
} else if (state.R == RState.ABOVE_ONE) {
    state.B0 = Math._SolveQuadraticFunctionForTarget(
        state.B,
        state.Q - state.Q0,
        DecimalMath.reciprocalFloor(state.i),
        state.K
    );
}
```

In `Math._SolveQuadraticFunctionForTarget` (lines 79–104 of `src/mimswap/libraries/Math.sol`), `state.B0` is reconstructed via:
1. `i_recip = DecimalMath.reciprocalFloor(state.i) = 1e36 / state.i`, which rounds down `1 / i`.
2. `_sqrt = sqrt(((ki * delta) / V1) + DecimalMath.ONE2)`, where `ki = 4 * k * i_recip`, `delta = state.Q - state.Q0`, and `V1 = state.B`.
3. `premium = DecimalMath.divFloor(_sqrt - DecimalMath.ONE, k * 2) + DecimalMath.ONE`.
4. `state.B0 = DecimalMath.mulFloor(V1, premium)`.

Because `_sqrt` is computed at `1e18` fixed-point scale before `_sqrt - 1e18` is divided by `2 * k`, any integer truncation in `sqrt` is amplified by `1e18 / (2 * k)` (for example, by `5,000x` when `k = 1e14`). In particular, whenever `((ki * delta) / V1) <= 2e18` (i.e., `delta <= (2e18 * state.B) / (4 * state.K * i_recip)`), `sqrt(1e36 + ((ki * delta) / V1))` truncates to `1e18`, yielding `premium = 1e18` and `state.B0 = state.B` even though `state.Q > state.Q0`.

When `PMMPricing.sellBaseToken` subsequently executes lines 46–62:

```solidity
uint256 backToOnePayBase = state.B0 - state.B;
uint256 backToOneReceiveQuote = state.Q - state.Q0;
if (payBaseAmount < backToOnePayBase) {
    // case 2.1: R status do not change
    receiveQuoteAmount = _RAboveSellBaseToken(state, payBaseAmount);
    newR = RState.ABOVE_ONE;
    if (receiveQuoteAmount > backToOneReceiveQuote) {
        receiveQuoteAmount = backToOneReceiveQuote;
    }
} else if (payBaseAmount == backToOnePayBase) {
    // case 2.2: R status changes to ONE
    receiveQuoteAmount = backToOneReceiveQuote;
    newR = RState.ONE;
}
```

- Because `state.B0` is underestimated, `backToOnePayBase = state.B0 - state.B` is strictly smaller than the true base token deficit required to return the pool to `RState.ONE`.
- In Case 2.1 (lines 51–58), `_RAboveSellBaseToken(state, payBaseAmount)` integrates the curve using `Math._GeneralIntegrate(state.B0, state.B + payBaseAmount, state.B, state.i, state.K)` with the underestimated `state.B0` and keeps `newR = RState.ABOVE_ONE`, causing subsequent swaps in `RState.ABOVE_ONE` to repeatedly reconstruct `state.B0` from the updated reserves and compound rounding errors.
- At the Case 2.1 / Case 2.2 boundary (line 59, `else if (payBaseAmount == backToOnePayBase)`), a trader only needs to supply the underestimated `backToOnePayBase` (which is `0` whenever `state.B0` truncates to `state.B`) to receive the full `backToOneReceiveQuote = state.Q - state.Q0` and transition `newR` to `RState.ONE`.
- In `MagicLP.sellBase` (lines 256–260), transitioning from `RState.ABOVE_ONE` to `RState.ONE` overwrites `_BASE_TARGET_` in storage with `newBaseTarget = state.B0`. Repeating small `sellQuote` swaps followed by `sellBase` swaps at `payBaseAmount == backToOnePayBase` ratchets `_BASE_TARGET_` downward and drains base tokens from the pool.

### Assumptions and preconditions

- A `MagicLP` pool is initialized and has active liquidity with `K > 0` (for example, the standard stable pool configuration `I = 1e18`, `K = 1e14`, `lpFeeRate = 1e14`).
- The pool is placed into `RState.ABOVE_ONE` via `sellQuote` with a quote excess `delta = Q - Q0` where `((4 * K * DecimalMath.reciprocalFloor(I) * delta) / B) <= 2e18` (or where `_SolveQuadraticFunctionForTarget` underestimates `state.B0`).

### Proof of concept

Consider a `MagicLP` pool with `_BASE_RESERVE_ = _BASE_TARGET_ = 10,000,000e18` (`1e25`), `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 10,000,000e18` (`1e25`), `_I_ = 1e18`, `_K_ = 1e14`, and `_LP_FEE_RATE_ = 1e14` (0.01%):

1. **Push pool into `RState.ABOVE_ONE`:**
   - A trader transfers `5e10` wei of quote token to `MagicLP` and calls `sellQuote`.
   - `_ROneSellQuoteToken` computes `receiveBaseAmount = 5e10` wei. After the 0.01% fee (`5e6` wei), the trader receives `4.9995e10` wei of base token.
   - Pool state updates to `_RState_ = RState.ABOVE_ONE`, `_BASE_RESERVE_ = 1e25 - 4.9995e10`, `_QUOTE_RESERVE_ = 1e25 + 5e10`, and `_QUOTE_TARGET_ = 1e25`.

2. **Trigger `sellBaseToken` at the `backToOnePayBase` boundary with zero input:**
   - The trader calls `sellBase` without transferring any base tokens (`baseInput = 0`).
   - `getPMMState()` calls `adjustedTarget(state)`:
     - `ki = 4 * 1e14 * 1e18 = 4e32`
     - `(ki * delta) / V1 = (4e32 * 5e10) / (1e25 - 4.9995e10) = 2000000000000009999 < 2e18 + 1`
     - `_sqrt = sqrt(1e36 + 2000000000000009999) = 1e18`
     - `state.B0 = state.B = 1e25 - 4.9995e10`
   - In `PMMPricing.sellBaseToken`:
     - `backToOnePayBase = state.B0 - state.B = 0`
     - `backToOneReceiveQuote = state.Q - state.Q0 = 5e10`
     - `payBaseAmount < backToOnePayBase` (`0 < 0`) is false; execution hits line 59 (`else if (payBaseAmount == backToOnePayBase)` with `0 == 0`).
     - `sellBaseToken` returns `receiveQuoteAmount = 5e10` and `newR = RState.ONE`.
   - `MagicLP.sellBase` transfers `4.9995e10` wei of quote token back to the trader and overwrites `_BASE_TARGET_` with `state.B0 = 1e25 - 4.9995e10` while setting `_RState_ = RState.ONE`.

3. **Repeat:**
   - In one cycle, the trader spent `5e10 - 4.9995e10 = 5e6` wei of quote token and extracted `4.9995e10` wei of base token while permanently reducing `_BASE_TARGET_`. Repeating this cycle drains the pool's base reserves.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Precision loss in target reconstruction (`state.B0`) and unchecked zero/underestimated `backToOnePayBase` in `PMMPricing.sellBaseToken` allow base reserves to be extracted without paying the required base token input and permanently corrupt `_BASE_TARGET_`.
- **Business Impact:** Liquidity providers in `MagicLP` pools can suffer direct loss of funds through repeated swap cycles that ratchet down `_BASE_TARGET_` and drain base tokens.

### Remediation

Ensure that `payBaseAmount` cannot settle `backToOneReceiveQuote` when `state.B0` suffers truncation in `RState.ABOVE_ONE` (for example, by rounding up in target reconstruction or requiring `receiveQuoteAmount` in `RState.ABOVE_ONE` to be bounded by `_RAboveSellBaseToken` when `backToOnePayBase` is truncated, and transitioning `newR` to `RState.ONE` whenever `receiveQuoteAmount` reaches `backToOneReceiveQuote`):

```solidity
            if (payBaseAmount < backToOnePayBase) {
                // case 2.1: R status do not change
                receiveQuoteAmount = _RAboveSellBaseToken(state, payBaseAmount);
                newR = RState.ABOVE_ONE;
                if (receiveQuoteAmount >= backToOneReceiveQuote) {
                    // [Important corner case!] may enter this branch when some precision problem happens. And consequently contribute to negative spare quote amount
                    // to make sure spare quote>=0, mannually set receiveQuote=backToOneReceiveQuote
                    receiveQuoteAmount = backToOneReceiveQuote;
                    newR = RState.ONE;
                }
            } else if (payBaseAmount == backToOnePayBase && backToOnePayBase > 0) {
```

## Finding 14: Incorrect `RState` and Target Retention in `PMMPricing.sellBaseToken` When Quote Output reaches `backToOneReceiveQuote`

### Synopsis

In `PMMPricing.sellBaseToken`, when `state.R == RState.ABOVE_ONE` and `payBaseAmount < backToOnePayBase`, rounding/precision in `adjustedTarget` and `_GeneralIntegrate` can cause `receiveQuoteAmount` to equal or exceed `backToOneReceiveQuote` (`state.Q - state.Q0`). Although `receiveQuoteAmount` is capped to `backToOneReceiveQuote`, `newR` remains `RState.ABOVE_ONE`, breaking PMM state invariants or underflowing reserves.

### Technical details

In `MagicLP`, when the pool is in state `RState.ABOVE_ONE`, the base reserve is below its target (`B < B0`) and the quote reserve is above its target (`Q > Q0`). Notably, when `getPMMState()` is called on a pool in `RState.ABOVE_ONE`, `PMMPricing.adjustedTarget(state)` dynamically recalculates `state.B0` from the current `state.B` and `state.Q - state.Q0` using `Math._SolveQuadraticFunctionForTarget`:

```solidity
state.B0 = Math._SolveQuadraticFunctionForTarget(
    state.B,
    state.Q - state.Q0,
    DecimalMath.reciprocalFloor(state.i),
    state.K
);
```

Because `DecimalMath.reciprocalFloor(state.i)` and `_SolveQuadraticFunctionForTarget` use integer division and rounding, `backToOnePayBase = state.B0 - state.B` can be slightly overestimated relative to the exact base amount required to extract `backToOneReceiveQuote = state.Q - state.Q0` via `_RAboveSellBaseToken`.

In `PMMPricing.sellBaseToken` (lines 50–58):

```solidity
if (payBaseAmount < backToOnePayBase) {
    // case 2.1: R status do not change
    receiveQuoteAmount = _RAboveSellBaseToken(state, payBaseAmount);
    newR = RState.ABOVE_ONE;
    if (receiveQuoteAmount > backToOneReceiveQuote) {
        receiveQuoteAmount = backToOneReceiveQuote;
    }
}
```

When `payBaseAmount < backToOnePayBase` is close to `backToOnePayBase`, `_RAboveSellBaseToken(state, payBaseAmount)` can return `receiveQuoteAmount >= backToOneReceiveQuote`, which is then capped to `backToOneReceiveQuote = state.Q - state.Q0`.

However, `newR` is still set to `RState.ABOVE_ONE` instead of transitioning to `RState.ONE` (or updating `_BASE_TARGET_`). This causes two severe issues in `MagicLP.sellBase`:

1. **Broken `_BASE_TARGET_` and `RState` invariant when LP fee is zero or dust:** If the total fee (`lpFee + mtFee`) rounds down to `0` (or if LP fee is 0), the full `backToOneReceiveQuote = state.Q - state.Q0` is transferred out of the pool. The new quote reserve becomes `state.Q - (state.Q - state.Q0) = state.Q0`. However, because `newR` remained `RState.ABOVE_ONE`, `MagicLP.sellBase` does not update `_RState_` to `RState.ONE` and does not update `_BASE_TARGET_` to `state.B0`. On the next call to `getPMMState()`, `state.R` is still `RState.ABOVE_ONE`, and `adjustedTarget` computes `state.B0` with `delta = state.Q - state.Q0 = 0`, collapsing `state.B0` down to the current `state.B` instead of preserving the true base target! Even when fees are non-zero, `_BASE_TARGET_` in storage is never updated during `ABOVE_ONE` trades (since `if (_RState_ != uint32(newRState))` in `sellBase` is skipped), relying on `Q - Q0` to reconstruct `B0`; once `Q` approaches `Q0` while `B < B0` (or `B + payBaseAmount < B0`), the pool is actually at equilibrium `RState.ONE` with target `B0`, so failing to transition to `RState.ONE` and record `_BASE_TARGET_ = newBaseTarget` permanently corrupts `B0` on subsequent operations.
2. **Symmetric failure in `sellQuoteToken`:** Even more directly, if `receiveQuoteAmount == backToOneReceiveQuote`, the pool has exhausted its excess quote tokens (`Q - Q0`) and reached equilibrium (`RState.ONE`), yet `newR` is kept as `RState.ABOVE_ONE`, leaving `_BASE_TARGET_` stale in storage (`_BASE_TARGET_` is only written when `_RState_ != uint32(newRState)` at lines 256–260 of `MagicLP.sol`).

### Assumptions and preconditions

1. The `MagicLP` pool is currently in `_RState_ == RState.ABOVE_ONE` (which occurs naturally after any user sells quote tokens when `R == ONE` or `R == ABOVE_ONE`).
2. `state.i` and `state.K` are configured such that `DecimalMath.reciprocalFloor(state.i)` or `_SolveQuadraticFunctionForTarget` introduces rounding upwards in `B0`, making `backToOnePayBase = state.B0 - state.B` larger than the `payBaseAmount` needed for `_GeneralIntegrate` to reach `backToOneReceiveQuote = state.Q - state.Q0`.
3. A trader (or attacker) sells `payBaseAmount < backToOnePayBase` of base token such that `_RAboveSellBaseToken(state, payBaseAmount) >= backToOneReceiveQuote`.

### Proof of concept

1. Suppose a `MagicLP` pool has `i` such that `DecimalMath.reciprocalFloor(i)` rounds down (for example, `i = 1e18 + 1`, so `reciprocalFloor(i) = 999999999999999999`).
2. A trader sells quote tokens, moving the pool into `RState.ABOVE_ONE` with `Q > Q0` and `B < B0`.
3. In `getPMMState()`, `adjustedTarget` computes `state.B0 = _SolveQuadraticFunctionForTarget(B, Q - Q0, reciprocalFloor(i), K)`. Because `reciprocalFloor(i)` is used to convert quote delta `Q - Q0` into base target delta, and `_GeneralIntegrate` multiplies by `i`, there exists a value `payBaseAmount = backToOnePayBase - 1` where `_RAboveSellBaseToken(state, payBaseAmount) >= state.Q - state.Q0 = backToOneReceiveQuote`.
4. The trader calls `sellBase` with `baseInput = backToOnePayBase - 1`.
5. Inside `PMMPricing.sellBaseToken`:
   - `payBaseAmount < backToOnePayBase` is `true` (enters lines 51–58).
   - `receiveQuoteAmount` is capped at `backToOneReceiveQuote = state.Q - state.Q0`.
   - `newR` is set to `RState.ABOVE_ONE`.
6. Inside `MagicLP.sellBase`:
   - Since `_RState_ == uint32(newRState)` (both are `ABOVE_ONE`), lines 256–260 are skipped: `_BASE_TARGET_` is NOT updated to `newBaseTarget` (`state.B0`), and `_RState_` remains `ABOVE_ONE`.
   - Quote tokens of amount `backToOneReceiveQuote` (minus fees) are transferred out, reducing `_QUOTE_RESERVE_` to `Q0` (plus LP fee).
7. On the next call to `getPMMState()`, `state.R` is still `ABOVE_ONE`, so `adjustedTarget` overwrites `state.B0` using `delta = state.Q - state.Q0 ≈ 0`. As a result, `state.B0` drops from the original target down to `state.B`, permanently corrupting the pool's target base reserve `B0` and shifting the PMM pricing curve and LP share valuation.

### Impact assessment

**Severity: Medium / High**

When the corner case at lines 54–58 is triggered (`receiveQuoteAmount >= backToOneReceiveQuote`), the pool outputs the entire quote surplus above `Q0`, effectively returning the quote reserve to `Q0` (equilibrium), while paying less than `backToOnePayBase`. Because `newR` is kept as `RState.ABOVE_ONE` instead of transitioning to `RState.ONE`, `MagicLP.sellBase` fails to persist `newBaseTarget` into `_BASE_TARGET_` and leaves `_RState_` as `ABOVE_ONE`. In subsequent transactions, `adjustedTarget` recomputes `B0` from the depleted `Q - Q0`, causing `B0` to collapse to the lower `B` reserve. This permanently corrupts the pool's target reserve invariant (`B0`), distorts pool pricing, and causes loss of value for liquidity providers.

### Remediation

When `receiveQuoteAmount` reaches or exceeds `backToOneReceiveQuote` in the `payBaseAmount < backToOnePayBase` branch, the pool's excess quote reserve (`Q - Q0`) is fully consumed, meaning the pool has reached the `RState.ONE` boundary. Therefore, the condition should include equality (`>=`) and transition `newR` to `RState.ONE` so that `MagicLP.sellBase` properly updates `_RState_` to `RState.ONE` and saves `newBaseTarget` (`state.B0`) into `_BASE_TARGET_`.

```solidity
            if (payBaseAmount < backToOnePayBase) {
                // case 2.1: R status do not change
                receiveQuoteAmount = _RAboveSellBaseToken(state, payBaseAmount);
                newR = RState.ABOVE_ONE;
                if (receiveQuoteAmount > backToOneReceiveQuote) {
                    // [Important corner case!] may enter this branch when some precision problem happens. And consequently contribute to negative spare quote amount
                    // to make sure spare quote>=0, mannually set receiveQuote=backToOneReceiveQuote
                    receiveQuoteAmount = backToOneReceiveQuote;
                    newR = RState.ONE;
                } else if (receiveQuoteAmount == backToOneReceiveQuote) {
                    newR = RState.ONE;
                }
            }
```

## Finding 15: Piecewise Pricing Discontinuity Across Equilibrium in `PMMPricing.sellBaseToken` Enables Value-Extracting Cyclic Trades

### Synopsis

In `PMMPricing.sellBaseToken`, when `state.R == RState.ABOVE_ONE` and `payBaseAmount >= backToOnePayBase`, the function unconditionally outputs the full `backToOneReceiveQuote` (`state.Q - state.Q0`) despite `backToOnePayBase` (`state.B0 - state.B`) being underestimated by downward rounding in `adjustedTarget`, creating an output discontinuity across equilibrium that can be exploited via cyclic swaps to drain pool reserves.

### Technical details

In `MagicLP`, trades query the current pool state via `getPMMState()`, which calls `PMMPricing.adjustedTarget(state)` (lines 190–201 of `PMMPricing.sol`). When the pool is in `RState.ABOVE_ONE` (`state.B < state.B0` and `state.Q > state.Q0`), `adjustedTarget` dynamically recalculates the base target `state.B0`:

```solidity
state.B0 = Math._SolveQuadraticFunctionForTarget(
    state.B,
    state.Q - state.Q0,
    DecimalMath.reciprocalFloor(state.i),
    state.K
);
```

In `Math._SolveQuadraticFunctionForTarget` (lines 79–104 of `Math.sol`), `state.B0` is rounded down through four successive floor operations:
1. `DecimalMath.reciprocalFloor(state.i)` floors `1e36 / state.i`.
2. `_sqrt = sqrt(((ki * delta) / V1) + DecimalMath.ONE2)` floors the square root to an integer before dividing by `2 * k`.
3. `premium = DecimalMath.divFloor(_sqrt - DecimalMath.ONE, k * 2) + DecimalMath.ONE` amplifies the integer truncation of `_sqrt` by `1e18 / (2 * k)`. In particular, whenever `(4 * k * reciprocalFloor(i) * (state.Q - state.Q0)) / state.B < 2e18 + 1`, `_sqrt` truncates to `1e18`, yielding `premium = 1e18`.
4. `DecimalMath.mulFloor(V1, premium)` floors the resulting target `state.B0`.

Because `state.B0` is underestimated (and equals `state.B` whenever `_sqrt == 1e18`), `backToOnePayBase = state.B0 - state.B` in `PMMPricing.sellBaseToken` (line 46) is strictly less than the true base token input required to return the pool to equilibrium (and is `0` when `state.B0 == state.B`).

In `PMMPricing.sellBaseToken` (lines 50–67):
- When `payBaseAmount < backToOnePayBase` (line 50), `receiveQuoteAmount` is calculated along the curve via `_RAboveSellBaseToken(state, payBaseAmount)` using the underestimated `state.B0`, yielding an amount strictly smaller than `backToOneReceiveQuote`.
- However, at `payBaseAmount == backToOnePayBase` (lines 60–62) and `payBaseAmount > backToOnePayBase` (lines 63–67), `sellBaseToken` bypasses `_RAboveSellBaseToken` and unconditionally awards the full `backToOneReceiveQuote = state.Q - state.Q0`:

```solidity
} else if (payBaseAmount == backToOnePayBase) {
    // case 2.2: R status changes to ONE
    receiveQuoteAmount = backToOneReceiveQuote;
    newR = RState.ONE;
} else {
    // case 2.3: R status changes to BELOW_ONE
    receiveQuoteAmount = backToOneReceiveQuote + _ROneSellBaseToken(state, payBaseAmount - backToOnePayBase);
    newR = RState.BELOW_ONE;
}
```

This creates a sharp upward discontinuity in `receiveQuoteAmount` at `payBaseAmount = backToOnePayBase`. Furthermore, when combined with `PMMPricing.sellQuoteToken` at lines 76–79 (which transitions `state.R` from `RState.ONE` to `RState.ABOVE_ONE`), a trader can transition the pool from `RState.ONE` to `RState.ABOVE_ONE` via `sellQuote` (receiving base tokens) and then immediately call `sellBase` with `payBaseAmount = backToOnePayBase` (which is `0` or less than the base tokens received), recovering `backToOneReceiveQuote = state.Q - state.Q0` while `MagicLP.sellBase` (lines 256–260) permanently lowers `_BASE_TARGET_` to the underestimated `state.B0` and resets `_RState_` to `RState.ONE`.

### Assumptions and preconditions

1. A `MagicLP` pool is initialized and has active liquidity with small `K` (e.g., `K = 1e14` as configured in `MIMSwap.t.sol`) or token decimal/price configurations where downward rounding in `_SolveQuadraticFunctionForTarget` exceeds the LP/maintainer fee rate (`MIN_LP_FEE_RATE = 1e14`).
2. The pool is in `RState.ONE` (or `RState.ABOVE_ONE` with `state.Q > state.Q0`).

### Proof of concept

Consider a pool at equilibrium (`_RState_ == RState.ONE`) with parameters matching the Blast deployment configuration (`_K_ = 1e14`, `_I_ = 1e18`, `_LP_FEE_RATE_ = 1e14`) and reserves `_BASE_RESERVE_ = _BASE_TARGET_ = 10,000,000e18` (`1e25`) and `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 10,000,000e18` (`1e25`):

1. **Transition from `ONE` to `ABOVE_ONE` (`sellQuoteToken`, lines 76–79):**
   - A trader supplies `payQuoteAmount = 4.999e10` wei of quote token and calls `sellQuote`.
   - In `PMMPricing.sellQuoteToken` (lines 77–79), `_ROneSellQuoteToken` computes `receiveBaseAmount ≈ 4.999e10` wei of base token (minus the `0.01%` fee), and `newR` is set to `RState.ABOVE_ONE`.
   - After the swap, `_BASE_RESERVE_ ≈ 1e25 - 4.999e10`, `_QUOTE_RESERVE_ = 1e25 + 4.999e10`, `_QUOTE_TARGET_ = 1e25`, and `_RState_ = RState.ABOVE_ONE`.

2. **Target underestimation in `adjustedTarget`:**
   - When `getPMMState()` is called for the next `sellBase` operation, `adjustedTarget` invokes `Math._SolveQuadraticFunctionForTarget(state.B, 4.999e10, 1e18, 1e14)`.
   - Inside `_SolveQuadraticFunctionForTarget`, `ki = 4 * 1e14 * 1e18 = 4e32`, so `(ki * delta) / V1 = (4e32 * 4.999e10) / (1e25 - 4.999e10) ≈ 1.9996e18 < 2e18`.
   - Consequently, `_sqrt = sqrt(1e36 + 1.9996e18) = 1e18`, `premium = 1e18`, and `state.B0 = state.B`.

3. **Zero-input equilibrium reset (`sellBaseToken`, lines 60–62):**
   - The trader calls `sellBase` with `baseInput = 0`.
   - In `PMMPricing.sellBaseToken`, `backToOnePayBase = state.B0 - state.B = 0` and `backToOneReceiveQuote = state.Q - state.Q0 = 4.999e10`.
   - Since `payBaseAmount == backToOnePayBase` (`0 == 0`), execution enters lines 60–62, setting `receiveQuoteAmount = 4.999e10` and `newR = RState.ONE`.
   - `MagicLP.sellBase` transfers `4.999e10` (minus the `0.01%` fee) quote tokens back to the trader, updates `_BASE_TARGET_` down to `state.B0 = 1e25 - 4.999e10`, and sets `_RState_ = RState.ONE`.
   - The trader retains `~4.998e10` base tokens while only paying the `0.01%` fee on the quote leg (`~9.998e6` wei), netting a risk-free profit per cycle and leaving the pool back in `RState.ONE` where the cycle can be repeated.

### Impact assessment

- **Severity:** High / Medium
- **Technical Impact:** Breaking the continuity and round-trip conservation invariants of the PMM pricing curve across the `R = 1` boundary allows attackers to extract value from the pool by repeatedly cycling between `RState.ONE` and `RState.ABOVE_ONE`,ratcheting `_BASE_TARGET_` downward on every iteration.
- **Business Impact:** Liquidity providers suffer direct loss of base token reserves, and the pool's target accounting (`_BASE_TARGET_`) becomes progressively degraded.

### Remediation

Ensure that `state.B0` in `adjustedTarget` is rounded up (or that `backToOneReceiveQuote` in `sellBaseToken` at lines 60–67 is capped by `_RAboveSellBaseToken(state, backToOnePayBase)` and zero-input state transitions are disallowed) so that crossing equilibrium cannot yield more quote tokens than the curve integral for `backToOnePayBase`:

```solidity
            } else if (payBaseAmount == backToOnePayBase) {
                // case 2.2: R status changes to ONE
                uint256 maxReceiveQuote = _RAboveSellBaseToken(state, backToOnePayBase);
                receiveQuoteAmount = maxReceiveQuote < backToOneReceiveQuote ? maxReceiveQuote : backToOneReceiveQuote;
                newR = RState.ONE;
            } else {
                // case 2.3: R status changes to BELOW_ONE
                uint256 maxReceiveQuote = _RAboveSellBaseToken(state, backToOnePayBase);
                uint256 aboveReceiveQuote = maxReceiveQuote < backToOneReceiveQuote ? maxReceiveQuote : backToOneReceiveQuote;
                receiveQuoteAmount = aboveReceiveQuote + _ROneSellBaseToken(state, payBaseAmount - backToOnePayBase);
                newR = RState.BELOW_ONE;
            }
```

## Finding 16: Premature Truncation in `PMMPricing` Quadratic Trade Evaluation (`_ROneSellBaseToken`, `_RBelowSellBaseToken`, `_ROneSellQuoteToken`) Overpays Minimum-Increment Trades Across and Below Equilibrium

### Synopsis

In `PMMPricing.sellBaseToken` and `PMMPricing.sellQuoteToken`, the calls to `_ROneSellBaseToken`, `_RBelowSellBaseToken`, and `_ROneSellQuoteToken` rely on `Math._SolveQuadraticFunctionForTrade`, which floors `bAbs` and `squareRoot` prior to addition, causing a 1-wei input past equilibrium (`65`) or below equilibrium (`71`) to output a full quote token unit and enabling profitable trade-splitting and cyclic swaps.

### Technical details

In `src/mimswap/libraries/PMMPricing.sol`, `sellBaseToken` and `sellQuoteToken` compute swap outputs across three highlighted branches by delegating to `Math._SolveQuadraticFunctionForTrade(V0, V1, delta, i, k)`:

1. `64-66` (`state.R == RState.ABOVE_ONE` and `payBaseAmount > backToOnePayBase`):
   ```solidity
   receiveQuoteAmount = backToOneReceiveQuote + _ROneSellBaseToken(state, payBaseAmount - backToOnePayBase);
   newR = RState.BELOW_ONE;
   ```
   where `_ROneSellBaseToken` evaluates `Math._SolveQuadraticFunctionForTrade(state.Q0, state.Q0, payBaseAmount - backToOnePayBase, state.i, state.K)`.
2. `71` (`state.R == RState.BELOW_ONE`):
   ```solidity
   receiveQuoteAmount = _RBelowSellBaseToken(state, payBaseAmount);
   ```
   where `_RBelowSellBaseToken` evaluates `Math._SolveQuadraticFunctionForTrade(state.Q0, state.Q, payBaseAmount, state.i, state.K)`.
3. `78` (`state.R == RState.ONE` in `sellQuoteToken`):
   ```solidity
   receiveBaseAmount = _ROneSellQuoteToken(state, payQuoteAmount);
   ```
   where `_ROneSellQuoteToken` evaluates `Math._SolveQuadraticFunctionForTrade(state.B0, state.B0, payQuoteAmount, DecimalMath.reciprocalFloor(state.i), state.K)`.

Inside `Math._SolveQuadraticFunctionForTrade` (`src/mimswap/libraries/Math.sol`, lines 170–204), when $k < 0.5 \times 10^{18}$ and `delta` is small, `bAbs >= part2` holds (`bSig == false`):
```solidity
uint256 part2 = (((k * V0) / V1) * V0) + (i * delta);
uint256 bAbs = (DecimalMath.ONE - k) * V1;
...
bAbs = bAbs - part2;
bSig = false;
...
bAbs = bAbs / DecimalMath.ONE;
uint256 squareRoot = DecimalMath.mulFloor((DecimalMath.ONE - k) * 4, DecimalMath.mulFloor(k, V0) * V0);
squareRoot = sqrt((bAbs * bAbs) + squareRoot);
uint256 denominator = (DecimalMath.ONE - k) * 2;
...
numerator = bAbs + squareRoot;
uint256 V2 = DecimalMath.divCeil(numerator, denominator);
return V1 - V2;
```

At equilibrium ($V_1 = V_0$, as in `_ROneSellBaseToken` at `65` and `_ROneSellQuoteToken` at `78`), before subtracting `i * delta`, $(10^{18} - k) V_0 - k V_0 = (10^{18} - 2k) V_0$. Whenever $2 k V_0$ is a multiple of $10^{18}$ (for example, in the `MagicLPTest` pool configuration where `V0 = 1000e6 = 10^9` USDT, `i = 1e6`, and `k = 5e14`, or in `MIMSwapTest` where `i = 997724689700000000 < 1e18` and `k = 1e14`), $(10^{18} - 2k) V_0 = M \cdot 10^{18}$ for the exact integer $M = \frac{(10^{18} - 2k) V_0}{10^{18}}$.

When a trader supplies `delta = 1` wei (e.g., `payBaseAmount = backToOnePayBase + 1` at `65`):
1. Subtracting `i * delta > 0` from $M \cdot 10^{18}$ and flooring via `bAbs = bAbs / DecimalMath.ONE` causes `bAbs` to drop by a full integer unit from $M$ to $M - 1$.
2. Because $M^2 + 4(1-k)k V_0^2 = V_0^2$, replacing $M$ with $M - 1$ inside `sqrt((bAbs * bAbs) + squareRoot)` yields:
   $$(M - 1)^2 + 4(1-k)k V_0^2 = V_0^2 - 2(1 - 2k)V_0 + 1 = (V_0 - 1)^2 + 4k V_0 < V_0^2$$
   for all $k < 0.5 \times 10^{18}$. Thus, the integer `sqrt` also drops by a full unit from $V_0$ to $V_0 - 1$.
3. Consequently, `numerator = bAbs + squareRoot` drops by two full units to $2(1-k)V_0 - 2$.
4. Dividing by `denominator = 2(1-k)` produces:
   $$V_2 = \left\lceil V_0 - \frac{1}{1 - k} \right\rceil = V_0 - 1$$
   because $\frac{1}{1 - k} > 1$ for all $k > 0$.
5. `Math._SolveQuadraticFunctionForTrade` returns `V1 - V2 = 1` full unit of the output token for a `1`-wei excess input, even when `i * delta` is orders of magnitude smaller than `1e18` (e.g., $10^{-12}$ output units in `MIM/USDT` where `i = 1e6`, or `< 1` output unit in `MIM/USDB` where `i < 1e18`).

### Assumptions and preconditions

1. A `MagicLP` pool is initialized with valid parameters where $0 < K < 0.5 \times 10^{18}$ and either `i < 1e18` (e.g., `MIM/USDT` with 18-decimal base and 6-decimal quote where `i = 1e6`, or `MIM/USDB` where `i = 997724689700000000`) or `reciprocalFloor(i) < 1e18` (when quote decimals exceed base decimals).
2. The pool is at or above equilibrium (`state.R == RState.ABOVE_ONE` or `state.R == RState.ONE`) or in `RState.BELOW_ONE` at a reserve level where `((DecimalMath.ONE - k) * V1 - ((k * V0) / V1) * V0) % DecimalMath.ONE < i * delta`.

### Proof of concept

Consider the `MagicLPTest` pool deployed in `test/MIMSwap.t.sol`:
- Base token (`MIM`, 18 decimals): `_BASE_RESERVE_ = _BASE_TARGET_ = 1000e18`
- Quote token (`USDT`, 6 decimals): `_QUOTE_RESERVE_ = _QUOTE_TARGET_ = 1000e6` (`1_000_000_000`)
- `_I_ = 1_000_000` ($10^6$), `_K_ = 500_000_000_000_000` ($5 \times 10^{14}$), `_LP_FEE_RATE_ = 1e14` (0.01%)

1. **Move pool to `RState.ABOVE_ONE`**:
   The attacker sells `1` unit of USDT ($10^{-6}$ USDT) via `sellQuote` (`78`). The pool transfers out `~999,899,999,999` wei of MIM ($\approx 10^{12}$ wei) and transitions to `_RState_ = RState.ABOVE_ONE` with `state.Q = 1_000_000_001`, `state.Q0 = 1_000_000_000`, `backToOneReceiveQuote = 1`, and `backToOnePayBase = state.B0 - state.B = 999_999_999_999` wei of MIM.
2. **Cross equilibrium by 1 wei (`65`)**:
   The attacker calls `sellBase` with `baseInput = backToOnePayBase + 1 = 1_000_000_000_000` wei of MIM ($10^{-6}$ MIM).
   - `PMMPricing.sellBaseToken` enters the `else` branch at `63-67`.
   - `backToOneReceiveQuote` is `1` unit of USDT.
   - `_ROneSellBaseToken(state, 1)` calls `Math._SolveQuadraticFunctionForTrade(1_000_000_000, 1_000_000_000, 1, 1_000_000, 500_000_000_000_000)`, which returns `1` unit of USDT for the `1` wei of excess MIM.
   - Total `receiveQuoteAmount` is `1 + 1 = 2` units of USDT.
   - In `querySellBase`, `DecimalMath.mulFloor(2, 1e14) = 0`, so 0 fee is deducted and the attacker receives `2` units of USDT.
3. **Result of cycle**:
   Across the two swaps, the attacker net pays $\approx 10^8$ wei of MIM ($10^{-10}$ USD) and receives `1` net unit of USDT ($10^{-6}$ USD, worth $10^{12}$ wei of MIM), extracting $10,000\times$ the input cost and repeating the cycle or splitting trades into 1-wei boundary increments to drain quote reserves.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: `PMMPricing.sellBaseToken` (`65`, `71`) and `PMMPricing.sellQuoteToken` (`78`) overstate the output amount by 1 full output token unit whenever a small input crosses a `DecimalMath.ONE` truncation boundary in `bAbs`, violating monotonicity of effective price and conservation of pool value.
- **Business Implications**: Attackers can split trades into many small boundary-crossing increments or execute cyclic swaps across equilibrium to extract excess quote or base tokens without paying proportional input value or LP fees (since `mulFloor(1, lpFeeRate) == 0`).

### Remediation

Preserve the `DecimalMath.ONE` ($10^{18}$) scaling precision in `Math._SolveQuadraticFunctionForTrade` when computing `bAbs` and `squareRoot` (or cap the output of `_ROneSellBaseToken`, `_RBelowSellBaseToken`, and `_ROneSellQuoteToken` so that `receiveAmount` cannot exceed the linear fair amount `DecimalMath.mulFloor(i, delta)` when `state.R == RState.ONE` or `BELOW_ONE`).

In `src/mimswap/libraries/PMMPricing.sol`, ensure that crossing into `BELOW_ONE` or trading at/below `ONE` bounds the marginal output by the unpenalized fair exchange amount and only transitions `RState` when a non-zero output is produced:

```solidity
            } else if (payBaseAmount == backToOnePayBase) {
                // case 2.2: R status changes to ONE
                receiveQuoteAmount = backToOneReceiveQuote;
                newR = RState.ONE;
            } else {
                // case 2.3: R status changes to BELOW_ONE
                uint256 extraQuote = _ROneSellBaseToken(state, payBaseAmount - backToOnePayBase);
                uint256 maxFairQuote = DecimalMath.mulFloor(state.i, payBaseAmount - backToOnePayBase);
                if (extraQuote > maxFairQuote) {
                    extraQuote = maxFairQuote;
                }
                receiveQuoteAmount = backToOneReceiveQuote + extraQuote;
                newR = extraQuote > 0 ? RState.BELOW_ONE : RState.ONE;
            }
        } else {
            // state.R == RState.BELOW_ONE
            // case 3: R<1
            uint256 maxFairQuote = DecimalMath.mulFloor(state.i, payBaseAmount);
            receiveQuoteAmount = _RBelowSellBaseToken(state, payBaseAmount);
            if (receiveQuoteAmount > maxFairQuote) {
                receiveQuoteAmount = maxFairQuote;
            }
            newR = RState.BELOW_ONE;
        }
    }

    function sellQuoteToken(PMMState memory state, uint256 payQuoteAmount) internal pure returns (uint256 receiveBaseAmount, RState newR) {
        if (state.R == RState.ONE) {
            uint256 maxFairBase = DecimalMath.divFloor(payQuoteAmount, state.i);
            receiveBaseAmount = _ROneSellQuoteToken(state, payQuoteAmount);
            if (receiveBaseAmount > maxFairBase) {
                receiveBaseAmount = maxFairBase;
            }
            newR = receiveBaseAmount > 0 ? RState.ABOVE_ONE : RState.ONE;
```

## Finding 17: Incorrect handling of `RState.ONE` in `PMMPricing.getMidPrice` and `PMMPricing.adjustedTarget` corrupts mid-price and TWAP oracle calculations

### Synopsis

`PMMPricing.getMidPrice` and `PMMPricing.adjustedTarget` fail to handle the `RState.ONE` regime explicitly. When `_BASE_TARGET_` and `_BASE_RESERVE_` diverge in `RState.ONE`, `getMidPrice` evaluates the `ABOVE_ONE` pricing formula using mismatched target and reserve values instead of returning `state.i`, corrupting spot mid-price queries and `_BASE_PRICE_CUMULATIVE_LAST_` TWAP accumulations.

### Technical details

In the DODO Proactive Market Maker (PMM) model implemented by `MagicLP` and `PMMPricing`, the pool operates in one of three regimes tracked by `_RState_`: `RState.ONE` (equilibrium), `RState.ABOVE_ONE` ($B < B_0, Q > Q_0$), and `RState.BELOW_ONE` ($B > B_0, Q < Q_0$). At equilibrium (`RState.ONE`), the effective target reserves equal the active reserves ($B_0 = B$ and $Q_0 = Q$), the price multiplier $R$ is $1$ (`DecimalMath.ONE`), and the marginal mid-price is the peg price `state.i`.

When `MagicLP.getMidPrice()` (also used in `MagicLP._twapUpdate()`) is called, it constructs the pool state via `getPMMState()`, which invokes `PMMPricing.adjustedTarget(state)` (lines 190–201), and then passes `state` to `PMMPricing.getMidPrice(state)` (lines 203–213):

1. In `PMMPricing.adjustedTarget(state)` (ending at line 201), only `RState.BELOW_ONE` and `RState.ABOVE_ONE` are handled. When `state.R == RState.ONE`, `state.B0` and `state.Q0` are left at the raw storage values `_BASE_TARGET_` and `_QUOTE_TARGET_`.
2. In `PMMPricing.getMidPrice(state)` (lines 203–208), the conditional checks `if (state.R == RState.BELOW_ONE)` on line 204 and uses a catch-all `} else {` on line 208 for both `RState.ABOVE_ONE` and `RState.ONE`:
   ```solidity
   if (state.R == RState.BELOW_ONE) {
       uint256 R = DecimalMath.divFloor((state.Q0 * state.Q0) / state.Q, state.Q);
       R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
       return DecimalMath.divFloor(state.i, R);
   } else {
       uint256 R = DecimalMath.divFloor((state.B0 * state.B0) / state.B, state.B);
       R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
       return DecimalMath.mulFloor(state.i, R);
   }
   ```

The `else` branch implicitly assumes that whenever `state.R == RState.ONE`, `state.B0 == state.B` holds so that `R == DecimalMath.ONE` and `getMidPrice` returns `state.i`. However, in `MagicLP`, `state.B0` (`_BASE_TARGET_`) and `state.B` (`_BASE_RESERVE_`) diverge while `_RState_ == uint32(RState.ONE)` across several standard execution paths:

- **Transition from `BELOW_ONE` to `ONE` in `sellQuote`**: In `PMMPricing.sellQuoteToken` (lines 92–95), when `payQuoteAmount == backToOnePayQuote`, `receiveBaseAmount` is set to `backToOneReceiveBase` (`state.B - state.B0`) and `newR` is set to `RState.ONE`. In `MagicLP.querySellQuote` (lines 187–189), LP fees (`lpFeeRate`) and maintainer fees (`mtFee`) are deducted from `receiveBaseAmount`. In `MagicLP.sellQuote` (lines 275–285), the LP fee portion of base tokens remains in the pool and is recorded in `_BASE_RESERVE_`, while `_BASE_TARGET_` is not updated (line 280 only updates `_QUOTE_TARGET_`). Furthermore, any base token LP fees accrued during prior `sellQuote` swaps in `BELOW_ONE` also remain in `_BASE_RESERVE_`. Consequently, the pool enters `RState.ONE` with `_BASE_RESERVE_ > _BASE_TARGET_`.
- **Unbalanced liquidity deposits in `buyShares`**: On initial mint (lines 381–383), if `quoteBalance < DecimalMath.mulFloor(baseBalance, _I_)`, `shares` is `DecimalMath.divFloor(quoteBalance, _I_) < baseBalance`. `_BASE_TARGET_` is set to `shares` while `_BASE_RESERVE_` is set to `baseBalance`, leaving `_BASE_TARGET_ < _BASE_RESERVE_` in `RState.ONE`. On subsequent deposits (lines 397–403), if `baseInputRatio > quoteInputRatio`, `mintRatio` equals `quoteInputRatio`, so `_BASE_TARGET_` increases by `mintRatio` while `_BASE_RESERVE_` increases by `baseInputRatio`.
- **Direct balance synchronization via `sync()`**: If base tokens are transferred directly to the pool and `sync()` is called (lines 567–579), `_BASE_RESERVE_` is updated to `baseBalance` while `_BASE_TARGET_` remains unchanged.

In all such cases where `state.R == RState.ONE` and `state.B > state.B0`, `getMidPrice` computes `R = (1 - K) + K * (B0 / B)^2 < 1e18` and returns `i * R < i`, producing a distorted mid-price that contradicts the actual `RState.ONE` trading curve (which prices trades from `state.i`) and corrupts `_BASE_PRICE_CUMULATIVE_LAST_` in `_twapUpdate()`.

### Assumptions and preconditions

- A `MagicLP` pool is initialized with non-zero slippage coefficient `_K_ > 0`.
- The pool is in `RState.ONE` with `_BASE_RESERVE_ != _BASE_TARGET_` (for example, via an unbalanced `buyShares` deposit, a `sellQuote` trade that brings `_RState_` from `BELOW_ONE` back to `ONE`, or a direct token transfer followed by `sync()`).

### Proof of concept

1. Initialize a `MagicLP` pool with `_I_ = 1e18` ($1.0$) and `_K_ = 1e18` ($1.0$). Initially, `_RState_ = uint32(RState.ONE)`.
2. A liquidity provider transfers `2000e18` base tokens and `1000e18` quote tokens to the pool and calls `buyShares(alice)`:
   - Because `quoteBalance (1000e18) < DecimalMath.mulFloor(baseBalance, _I_) (2000e18)`, `shares` is calculated as `DecimalMath.divFloor(1000e18, 1e18) = 1000e18`.
   - `_BASE_TARGET_` (`B0`) is set to `1000e18` and `_QUOTE_TARGET_` (`Q0`) is set to `1000e18`.
   - `_BASE_RESERVE_` (`B`) is set to `2000e18` and `_QUOTE_RESERVE_` (`Q`) is set to `1000e18`.
   - `_RState_` remains `RState.ONE`.
3. Alternatively, starting from a balanced pool (`B = B0 = 1000e18, Q = Q0 = 1000e18`), a trader sells base tokens to move the pool to `RState.BELOW_ONE`, and then sells exact `backToOnePayQuote` quote tokens via `sellQuote`:
   - `PMMPricing.sellQuoteToken` returns `newR = RState.ONE`.
   - Due to LP fees retained in the base reserve, `_BASE_RESERVE_` (`B`) is now greater than `_BASE_TARGET_` (`B0`), while `_RState_` is set to `RState.ONE`.
4. When `getMidPrice()` is called (directly or inside `_twapUpdate()`):
   - `getPMMState()` calls `PMMPricing.adjustedTarget(state)`, which does nothing because `state.R == RState.ONE`.
   - `PMMPricing.getMidPrice(state)` enters the `else` branch on line 208.
   - In the `buyShares` scenario (`B0 = 1000e18, B = 2000e18, K = 1e18`), `R = ((1000e18 * 1000e18) / 2000e18) * 1e18 / 2000e18 = 0.25e18`.
   - `getMidPrice()` returns `0.25e18` instead of the true equilibrium mid-price `state.i = 1e18`.

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: `getMidPrice()` returns an erroneous price whenever `state.R == RState.ONE` and `_BASE_TARGET_ != _BASE_RESERVE_`. Because `_twapUpdate()` accumulates `getMidPrice() * timeElapsed` into `_BASE_PRICE_CUMULATIVE_LAST_`, the on-chain TWAP oracle and any external consumers relying on `getMidPrice()` or `_BASE_PRICE_CUMULATIVE_LAST_` receive corrupted price data. Furthermore, an attacker can intentionally donate base tokens and call `sync()` while the pool is in `RState.ONE` to depress `getMidPrice()` toward `state.i * (1 - state.K)` without executing a swap or changing `RState`.

### Remediation

Explicitly handle `RState.ONE` in `PMMPricing.getMidPrice` (and/or `PMMPricing.adjustedTarget`) so that `getMidPrice` returns `state.i` when `state.R == RState.ONE`, and `adjustedTarget` aligns `state.B0` and `state.Q0` with `state.B` and `state.Q`:

```solidity
    function adjustedTarget(PMMState memory state) internal pure {
        if (state.R == RState.BELOW_ONE) {
            state.Q0 = Math._SolveQuadraticFunctionForTarget(state.Q, state.B - state.B0, state.i, state.K);
        } else if (state.R == RState.ABOVE_ONE) {
            state.B0 = Math._SolveQuadraticFunctionForTarget(
                state.B,
                state.Q - state.Q0,
                DecimalMath.reciprocalFloor(state.i),
                state.K
            );
        } else {
            state.B0 = state.B;
            state.Q0 = state.Q;
        }
    }

    function getMidPrice(PMMState memory state) internal pure returns (uint256) {
        if (state.R == RState.BELOW_ONE) {
            uint256 R = DecimalMath.divFloor((state.Q0 * state.Q0) / state.Q, state.Q);
            R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
            return DecimalMath.divFloor(state.i, R);
        } else if (state.R == RState.ABOVE_ONE) {
            uint256 R = DecimalMath.divFloor((state.B0 * state.B0) / state.B, state.B);
            R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
            return DecimalMath.mulFloor(state.i, R);
        } else {
            return state.i;
        }
    }
```

## Finding 18: Precision Loss from Unscaled Intermediate Division in `PMMPricing.getMidPrice` Distorts Mid-Price and TWAP Calculation

### Synopsis

In `PMMPricing.getMidPrice`, when `state.R == RState.BELOW_ONE`, the reserve ratio `R` is calculated via `DecimalMath.divFloor((state.Q0 * state.Q0) / state.Q, state.Q)`. Performing the unscaled integer division `(state.Q0 * state.Q0) / state.Q` prior to fixed-point scaling truncates precision, distorting the mid-price and `MagicLP` TWAP accumulator.

### Technical details

In `src/mimswap/libraries/PMMPricing.sol`, `getMidPrice` calculates the current pool mid-price used by `MagicLP.getMidPrice()` and the cumulative price oracle update in `MagicLP._twapUpdate()`.

When the pool state is `RState.BELOW_ONE`, lines 204–207 compute:

```solidity
if (state.R == RState.BELOW_ONE) {
    uint256 R = DecimalMath.divFloor((state.Q0 * state.Q0) / state.Q, state.Q);
    R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
    return DecimalMath.divFloor(state.i, R);
}
```

`DecimalMath.divFloor(target, d)` is defined in `src/mimswap/libraries/DecimalMath.sol` as `(target * ONE) / d` where `ONE = 10 ** 18`. Because `(state.Q0 * state.Q0) / state.Q` is evaluated as the `target` argument before `DecimalMath.divFloor` multiplies by `10 ** 18`, the formula executes two sequential divisions with the `10 ** 18` scaling applied only to the second division:

$$R = \left\lfloor \frac{\left\lfloor \frac{Q_0^2}{Q} \right\rfloor \times 10^{18}}{Q} \right\rfloor$$

The first division $\lfloor Q_0^2 / Q \rfloor$ is completely unscaled and truncates the quotient to integer token units of `state.Q`. Whenever `state.Q` has low decimals (such as 6-decimal quote tokens like USDT/USDC) or when liquidity in the pool is small, truncating $Q_0^2 / Q$ before multiplying by `10 ** 18` discards up to `10 ** 18 / state.Q` wei of fixed-point precision in `R`. This underestimated `R` is then multiplied by `state.K` and used as the divisor in `DecimalMath.divFloor(state.i, R)`, inflating the returned mid-price and skewing `_BASE_PRICE_CUMULATIVE_LAST_` in `MagicLP._twapUpdate()`.

### Assumptions and preconditions

1. A `MagicLP` pool is in the `PMMPricing.RState.BELOW_ONE` state (i.e., base tokens have been sold into the pool such that `_BASE_RESERVE_ > _BASE_TARGET_` and `state.Q0 > state.Q`).
2. `state.Q` is non-zero and not an exact divisor of `state.Q0 * state.Q0`, with greater relative error when `state.Q` is small or uses low token decimals (e.g., 6 decimals).

### Proof of concept

Consider a pool in `RState.BELOW_ONE` with parameters `state.i = 1e18`, `state.K = 1e18`, quote reserve `state.Q = 1000`, and adjusted quote target `state.Q0 = 1049`:

1. The exact 18-decimal value of $R = (Q_0 / Q)^2$ is:
   $$\left\lfloor \frac{1049^2 \times 10^{18}}{1000^2} \right\rfloor = \left\lfloor \frac{1,100,401 \times 10^{18}}{1,000,000} \right\rfloor = 1.100401 \times 10^{18}$$
2. In `PMMPricing.getMidPrice` (line 205), the inner division `(state.Q0 * state.Q0) / state.Q` computes:
   $$\lfloor 1,100,401 / 1000 \rfloor = 1100$$
   The fractional part (`0.401`) is truncated prior to scaling.
3. `DecimalMath.divFloor(1100, 1000)` then computes:
   $$(1100 \times 10^{18}) / 1000 = 1.100000 \times 10^{18}$$
4. Line 207 computes `DecimalMath.divFloor(state.i, R)`:
   - Actual return value: $\lfloor 10^{36} / (1.100000 \times 10^{18}) \rfloor = 909,090,909,090,909,090$
   - Expected return value: $\lfloor 10^{36} / (1.100401 \times 10^{18}) \rfloor = 908,759,624,900,377,226$
5. When `MagicLP._twapUpdate()` executes during a swap or liquidity operation, this inaccurate mid-price is multiplied by `timeElapsed` and permanently added to `_BASE_PRICE_CUMULATIVE_LAST_`.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** `PMMPricing.getMidPrice` systematically underestimates `R` (and consequently overestimates the returned mid-price) when `state.R == RState.BELOW_ONE` due to premature truncation in the unscaled intermediate division.
- **Business Implications:** External callers relying on `MagicLP.getMidPrice()` or the TWAP accumulator `_BASE_PRICE_CUMULATIVE_LAST_` receive inaccurate price data, particularly for pools with low-decimal quote tokens or low quote reserves.

### Remediation

Scale the numerator by `DecimalMath.ONE` before dividing by `state.Q * state.Q` (or use `Math.mulDiv` to avoid intermediate overflow while performing a single division).

```solidity
    function getMidPrice(PMMState memory state) internal pure returns (uint256) {
        if (state.R == RState.BELOW_ONE) {
            uint256 R = DecimalMath.divFloor(state.Q0 * state.Q0, state.Q * state.Q);
            R = (DecimalMath.ONE - state.K) + DecimalMath.mulFloor(state.K, R);
            return DecimalMath.divFloor(state.i, R);
        } else {
```
