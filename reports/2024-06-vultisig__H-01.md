# Audit report — 2024-06-vultisig/H-01

## Finding 1: Incorrect Fee Collection and Settlement in `ILOPool.claim` Drains Other Positions' Accrued Fees to `FEE_TAKER` and Blocks Subsequent Claims

### Synopsis

In `ILOPool.claim`, calling `pool.burn` accrues swap fees for the entire aggregate `ILOPool` liquidity position in `UniswapV3Pool`, and `pool.collect` withdraws all accrued fees across all positions. Because `ILOPool` transfers `amountCollected - amount` to `FEE_TAKER`, one user's claim sweeps other positions' fees to `FEE_TAKER`, causing subsequent claims to underflow and revert.

### Technical details

`ILOPool` manages multiple individual investor and project positions (represented as ERC-721 `tokenId`s), but deposits all of their liquidity into a single shared `UniswapV3Pool` position keyed by `(address(this), TICK_LOWER, TICK_UPPER)`.

When a user calls `ILOPool.claim(tokenId)`:

1. At line 205, `pool.burn(TICK_LOWER, TICK_UPPER, liquidity2Claim)` is invoked on the `UniswapV3Pool`:
   ```solidity
   (amount0, amount1) = pool.burn(TICK_LOWER, TICK_UPPER, liquidity2Claim);
   ```
   Inside `UniswapV3Pool.burn` -> `_modifyPosition` -> `_updatePosition`, the pool updates the shared position `(address(this), TICK_LOWER, TICK_UPPER)`. Specifically, `Position.update` calculates the newly accrued swap fees across the **entire aggregate liquidity** (`_self.liquidity`) held by `ILOPool` since the last touch, adds the total aggregate fees to the pool position's `tokensOwed0` and `tokensOwed1`, and updates the pool position's `feeGrowthInside0LastX128` and `feeGrowthInside1LastX128` to the current global inside fee growth. `pool.burn` then adds the burned principal (`amount0`, `amount1`) for `liquidity2Claim` to `tokensOwed0` and `tokensOwed1`.

2. At lines 208–231, `ILOPool.claim` computes the user's net principal and net fee share (`amount0` and `amount1`) exclusively for the single `tokenId`'s `positionLiquidity`:
   ```solidity
   uint256 fees0 = FullMath.mulDiv(
       feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
       positionLiquidity,
       FixedPoint128.Q128
   );
   ```

3. At lines 242–248, `ILOPool.claim` calls `pool.collect` with `type(uint128).max`:
   ```solidity
   (uint128 amountCollected0, uint128 amountCollected1) = pool.collect(
       address(this),
       TICK_LOWER,
       TICK_UPPER,
       type(uint128).max,
       type(uint128).max
   );
   ```
   This withdraws the **entire** `tokensOwed0` and `tokensOwed1` from `UniswapV3Pool`, which equals:
   - The burned principal for `tokenId`, plus
   - The swap fees accrued by `tokenId`, plus
   - The swap fees accrued by **all other positions** in `ILOPool` since the last pool interaction.

4. At lines 259–260, `ILOPool.claim` transfers the remainder of the collected tokens to `feeTaker`:
   ```solidity
   TransferHelper.safeTransfer(_cachedPoolKey.token0, feeTaker, amountCollected0-amount0);
   TransferHelper.safeTransfer(_cachedPoolKey.token1, feeTaker, amountCollected1-amount1);
   ```
   Because `amountCollected0` includes the swap fees belonging to all other positions while `amount0` only includes `tokenId`'s net principal and net fees, all swap fees accrued by every other position in `ILOPool` are erroneously sent to `feeTaker`.

5. When a second position holder (`tokenId2`) later calls `claim(tokenId2)`:
   - `position.feeGrowthInside0LastX128` for `tokenId2` still reflects the checkpoint prior to `tokenId`'s claim, so `fees0` and `fees1` for `tokenId2` include the fees accrued before `tokenId`'s claim.
   - However, those fees were already withdrawn from `UniswapV3Pool` and sent to `feeTaker` during `tokenId`'s claim.
   - Consequently, `amountCollected0` (or `amountCollected1`) returned by `pool.collect` during `tokenId2`'s claim only contains `tokenId2`'s burned principal and any fees accrued after `tokenId`'s claim.
   - When `amount0 > amountCollected0`, the subtraction `amountCollected0 - amount0` at line 259 (compiled with Solidity `0.7.6` without checked arithmetic) underflows to a huge `uint256` value, causing `TransferHelper.safeTransfer` to revert with `"ST"`. Even if `ILOPool` has no spare token balance, `TransferHelper.safeTransfer` at line 252 or line 259 will revert, permanently blocking `tokenId2` from claiming both their vested liquidity principal and their fees.

### Assumptions and preconditions

- The ILO pool has been successfully launched via `launch()` with at least two positions (e.g., an investor position and project vesting positions, or multiple investor positions).
- At least one swap has occurred in the `UniswapV3Pool` within `[TICK_LOWER, TICK_UPPER]`, generating non-zero `feeGrowthInside0LastX128` or `feeGrowthInside1LastX128`.
- Vesting has started so that positions can call `claim()`.

### Proof of concept

1. Suppose `ILOPool` launches with total liquidity $L = 1000$, split equally between Investor A (`tokenId = 1`, liquidity = 500) and Investor B (`tokenId = 2`, liquidity = 500). Initially, `feeGrowthInside0LastX128 = 0` for both positions.
2. Trading occurs on the `UniswapV3Pool`, accruing 100 `token0` in total swap fees for the `ILOPool` position (50 `token0` attributable to Investor A and 50 `token0` attributable to Investor B).
3. Once vesting starts, Investor A calls `claim(1)`:
   - `pool.burn(TICK_LOWER, TICK_UPPER, liquidity2Claim_A)` is called. Suppose `liquidity2Claim_A` corresponds to 200 `token0` of principal.
   - Inside `UniswapV3Pool`, `Position.update` calculates 100 `token0` of accrued fees across the full $L = 1000$ liquidity and adds both the 200 `token0` principal and 100 `token0` aggregate fees to `tokensOwed0` (`tokensOwed0 = 300`).
   - `ILOPool` deducts platform fee (e.g., 0.1% of 200 = 0.2 `token0`, leaving 199.8 `token0`) and calculates Investor A's fee share (50 `token0` minus 10% performance fee = 45 `token0`). Thus, `amount0 = 199.8 + 45 = 244.8` `token0`.
   - `pool.collect` collects `amountCollected0 = 300` `token0`.
   - Investor A receives `amount0 = 244.8` `token0`.
   - `feeTaker` receives `amountCollected0 - amount0 = 300 - 244.8 = 55.2` `token0` (which includes Investor B's 50 `token0` of accrued fees).
4. In the same block (or before enough new fees accrue to cover the deficit), Investor B calls `claim(2)`:
   - `pool.burn(TICK_LOWER, TICK_UPPER, liquidity2Claim_B)` burns Investor B's claimable liquidity (200 `token0` principal). Since `feeGrowthInside0LastX128` in the pool has not increased since step 3, 0 new fees are added to `tokensOwed0` (`tokensOwed0 = 200`).
   - `ILOPool` computes Investor B's `amount0`: 199.8 `token0` net principal + 45 `token0` net fees (since `positions[2].feeGrowthInside0LastX128` was still 0) = `244.8` `token0`.
   - `pool.collect` only collects `amountCollected0 = 200` `token0`.
   - At line 252, transferring `244.8` `token0` to Investor B fails due to insufficient token balance in `ILOPool` (or if `ILOPool` holds extra tokens, line 259 underflows `200 - 244.8` to `type(uint256).max - 44.8 + 1` and reverts in `safeTransfer`).
   - Investor B's `claim(2)` transaction reverts, locking Investor B's position.

### Impact assessment

- **Severity**: High / Critical.
- **Direct Technical Effects**:
  1. Theft/misattribution of yield: Every claim sends all other positions' unclaimed Uniswap V3 swap fees to `FEE_TAKER`.
  2. Permanent or prolonged Denial of Service on `claim`: Subsequent claimants cannot withdraw their vested liquidity or fees because the pool no longer holds the fees accounted to them, causing `claim` to revert on insufficient balance or arithmetic underflow.

### Remediation

Instead of collecting `type(uint128).max` from the `UniswapV3Pool` and transferring the entire excess `amountCollected - amount` to `feeTaker`, explicitly track the exact platform and performance fees deducted for the claiming position, and only collect the exact total amount (`amount0 + fee0`, `amount1 + fee1`) attributable to the claiming `tokenId` from the `UniswapV3Pool` (leaving other positions' accrued fees in the pool's `tokensOwed0` and `tokensOwed1`).

```solidity
    function claim(uint256 tokenId)
        external
        payable
        override
        isAuthorizedForToken(tokenId)
        returns (uint256 amount0, uint256 amount1)
    {
        // only can claim if the launch is successfully
        require(_launchSucceeded, "PNL");

        // calculate amount of unlocked liquidity for the position
        uint128 liquidity2Claim = _claimableLiquidity(tokenId);
        IUniswapV3Pool pool = IUniswapV3Pool(_cachedUniV3PoolAddress);
        Position storage position = _positions[tokenId];
        uint128 totalCollect0;
        uint128 totalCollect1;
        {
            IILOManager.Project memory _project = IILOManager(MANAGER).project(address(pool));

            uint128 positionLiquidity = position.liquidity;
            require(positionLiquidity >= liquidity2Claim);

            // get amount of token0 and token1 that pool will return for us
            (uint256 principal0, uint256 principal1) = pool.burn(TICK_LOWER, TICK_UPPER, liquidity2Claim);

            // get amount of token0 and token1 after deduct platform fee
            (amount0, amount1) = _deductFees(principal0, principal1, _project.platformFee);

            bytes32 positionKey = PositionKey.compute(address(this), TICK_LOWER, TICK_UPPER);

            // calculate amount of fees that position generated
            (, uint256 feeGrowthInside0LastX128, uint256 feeGrowthInside1LastX128, , ) = pool.positions(positionKey);
            uint256 rawFees0 = FullMath.mulDiv(
                feeGrowthInside0LastX128 - position.feeGrowthInside0LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            );

            uint256 rawFees1 = FullMath.mulDiv(
                feeGrowthInside1LastX128 - position.feeGrowthInside1LastX128,
                positionLiquidity,
                FixedPoint128.Q128
            );

            // amount of fees after deduct performance fee
            (uint256 fees0, uint256 fees1) = _deductFees(rawFees0, rawFees1, _project.performanceFee);

            // fees is combined with liquidity token amount to return to the user
            amount0 += fees0;
            amount1 += fees1;

            totalCollect0 = uint128(principal0 + rawFees0);
            totalCollect1 = uint128(principal1 + rawFees1);

            position.feeGrowthInside0LastX128 = feeGrowthInside0LastX128;
            position.feeGrowthInside1LastX128 = feeGrowthInside1LastX128;

            // subtraction is safe because we checked positionLiquidity is gte liquidity2Claim
            position.liquidity = positionLiquidity - liquidity2Claim;
            emit DecreaseLiquidity(tokenId, liquidity2Claim, amount0, amount1);
        }
        // collect only the principal and fees belonging to this position
        (uint128 amountCollected0, uint128 amountCollected1) = pool.collect(
            address(this),
            TICK_LOWER,
            TICK_UPPER,
            totalCollect0,
            totalCollect1
        );
        emit Collect(tokenId, address(this), amountCollected0, amountCollected1);

        // transfer token for user
        TransferHelper.safeTransfer(_cachedPoolKey.token0, ownerOf(tokenId), amount0);
        TransferHelper.safeTransfer(_cachedPoolKey.token1, ownerOf(tokenId), amount1);

        emit Claim(ownerOf(tokenId), tokenId, liquidity2Claim, amount0, amount1, position.feeGrowthInside0LastX128, position.feeGrowthInside1LastX128);

        address feeTaker = IILOManager(MANAGER).FEE_TAKER();
        // transfer fee to fee taker
        TransferHelper.safeTransfer(_cachedPoolKey.token0, feeTaker, amountCollected0 - amount0);
        TransferHelper.safeTransfer(_cachedPoolKey.token1, feeTaker, amountCollected1 - amount1);
    }
```

## Finding 2: Silent `uint128` Overflow in `ILOPool._unlockedLiquidity` Corrupts Claimable Liquidity Calculation in `ILOPool.claim`

### Synopsis

In `ILOPool._unlockedLiquidity`, the expression `vest.shares * totalLiquidity` is evaluated using `uint128` arithmetic before being implicitly upcast to `uint256` for `FullMath.mulDiv`. Because the contract uses Solidity `0.7.6`, this multiplication silently wraps on overflow during active vesting periods, causing `ILOPool.claim` to compute an incorrect `liquidity2Claim`.

### Technical details

When a position holder calls `ILOPool.claim(uint256 tokenId)`, line 195 computes the currently claimable liquidity via `_claimableLiquidity(tokenId)`:

```solidity
uint128 liquidity2Claim = _claimableLiquidity(tokenId);
```

`_claimableLiquidity(tokenId)` (lines 457–461) calls `_unlockedLiquidity(tokenId)` (lines 400–431) to determine how much liquidity has unlocked according to the position's `LinearVest[]` schedule:

```solidity
if (vest.end < block.timestamp) {
    liquidityUnlocked += uint128(FullMath.mulDiv(
        vest.shares, 
        totalLiquidity, 
        BPS
    ));
} else {
    liquidityUnlocked += uint128(FullMath.mulDiv(
        vest.shares * totalLiquidity, 
        block.timestamp - vest.start, 
        (vest.end - vest.start) * BPS
    ));
}
```

In `IILOVest.LinearVest`, `vest.shares` has type `uint16` (up to `10000` BPS), `vest.start` and `vest.end` have type `uint64`, and `totalLiquidity` in `_unlockedLiquidity` has type `uint128`.

In the `if (vest.end < block.timestamp)` branch (lines 418–422), `vest.shares` and `totalLiquidity` are passed as separate arguments to `FullMath.mulDiv(uint256 a, uint256 b, uint256 denominator)`, so both operands are implicitly upcast to `uint256` prior to multiplication.

However, in the `else` branch (lines 424–428), when `vest.start <= block.timestamp <= vest.end`, the contract computes `vest.shares * totalLiquidity` as the first argument to `FullMath.mulDiv`. According to Solidity's type promotion rules, multiplying a `uint16` (`vest.shares`) by a `uint128` (`totalLiquidity`) is performed in the common type `uint128` before the result is implicitly converted to `uint256`. Similarly, `(vest.end - vest.start) * BPS` is evaluated in `uint64` before conversion to `uint256`.

Because `ILOPool` is compiled with `pragma solidity =0.7.6;`, arithmetic operations do not revert on overflow and instead wrap modulo $2^{128}$ (and $2^{64}$). Whenever `totalLiquidity > type(uint128).max / vest.shares` (for example, `totalLiquidity > 3.4028e34` when `vest.shares == 10000`), `vest.shares * totalLiquidity` silently overflows `uint128`. Consequently:
- `_unlockedLiquidity(tokenId)` returns a wrapped, severely underestimated unlocked liquidity value throughout the linear vesting window `[vest.start, vest.end]`.
- In `_claimableLiquidity(tokenId)`, if the wrapped `liquidityUnlocked` falls below `liquidityClaimed`, the function returns `0`, preventing the user from claiming newly vested liquidity in `claim(tokenId)` (lines 195–205) until after `block.timestamp > vest.end`.

### Assumptions and preconditions

1. An ILO pool has been initialized and launched (`_launchSucceeded == true`).
2. A position (investor or project vesting position) holds `totalLiquidity` such that `vest.shares * totalLiquidity > type(uint128).max` (e.g., `totalLiquidity >= 3.41e34` for a vesting schedule with `vest.shares = 10000` BPS, which is reachable with high-supply or high-decimal tokens or narrow tick ranges).
3. The position owner calls `claim(tokenId)` while an active vesting schedule is in progress (`vest.start <= block.timestamp <= vest.end`).

### Proof of concept

1. Suppose a launched ILO pool mints a vesting position (`tokenId`) with `totalLiquidity = 4e34` (`< type(uint128).max ≈ 3.4028e38`) and a single `LinearVest` schedule with `shares = 10000` (100%), `start = 1000`, `end = 2000`.
2. At `block.timestamp = 1500` (50% of the vesting duration elapsed), the expected unlocked liquidity is:
   $$\frac{4 \times 10^{34} \times 10000 \times (1500 - 1000)}{(2000 - 1000) \times 10000} = 2 \times 10^{34}$$
3. When the user calls `claim(tokenId)` (or queries `vestingStatus(tokenId)`):
   - Line 195 calls `_claimableLiquidity(tokenId)`, which calls `_unlockedLiquidity(tokenId)`.
   - Line 425 evaluates `vest.shares * totalLiquidity` in `uint128`:
     $$(10000 \times 4 \times 10^{34}) \bmod 2^{128} = 4 \times 10^{38} - 340,282,366,920,938,463,463,374,607,431,768,211,456 \approx 5.97176 \times 10^{37}$$
   - `FullMath.mulDiv` divides this wrapped product by `1000 * 10000`, yielding `liquidityUnlocked ≈ 2.98588e33` instead of `2e34` (less than 15% of the actual unlocked entitlement).
4. If the user had already claimed `3e33` from an earlier vest or earlier point in the schedule before wrap-around, `liquidityClaimed < liquidityUnlocked` is `false`, so `_claimableLiquidity(tokenId)` returns `0` and `pool.burn(TICK_LOWER, TICK_UPPER, 0)` burns `0` liquidity, denying the user access to their vested tokens until `block.timestamp > vest.end`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** `uint128` overflow during linear vesting calculation corrupts `_unlockedLiquidity(tokenId)` and `_claimableLiquidity(tokenId)`, causing `claim(tokenId)` to burn and return only a fraction of the user's vested liquidity (or `0` liquidity) for the entire duration of the active vesting period.
- **Business Impact:** Investors and project vesting recipients with large liquidity positions cannot withdraw their vested liquidity according to the agreed linear vesting schedule and are forced to wait until the vesting period completely expires (`block.timestamp > vest.end`).

### Remediation

Explicitly cast `vest.shares` (and `vest.end - vest.start`) to `uint256` before performing multiplication in `_unlockedLiquidity`:

```solidity
    function _unlockedLiquidity(uint256 tokenId) internal view override returns (uint128 liquidityUnlocked) {
        PositionVest storage _positionVest = _positionVests[tokenId];
        LinearVest[] storage vestingSchedule = _positionVest.schedule;
        uint128 totalLiquidity = _positionVest.totalLiquidity;

        for (uint256 index = 0; index < vestingSchedule.length; index++) {

            LinearVest storage vest = vestingSchedule[index];

            // if vest is not started, skip this vest and all following vest
            if (block.timestamp < vest.start) {
                break;
            }

            // if vest already end, all the shares are unlocked
            // otherwise we calculate shares of unlocked times and get the unlocked share number
            // all vest after current unlocking vest is ignored
            if (vest.end < block.timestamp) {
                liquidityUnlocked += uint128(FullMath.mulDiv(
                    vest.shares, 
                    totalLiquidity, 
                    BPS
                ));
            } else {
                liquidityUnlocked += uint128(FullMath.mulDiv(
                    uint256(vest.shares) * totalLiquidity, 
                    block.timestamp - vest.start, 
                    uint256(vest.end - vest.start) * BPS
                ));
            }
        }
    }
```

## Finding 3: Strict Minimum Amount Check and Unused Desired Amounts in `LiquidityManagement.addLiquidity` Cause `ILOPool.launch` to Revert Due to Liquidity Rounding

### Synopsis

In `LiquidityManagement.addLiquidity`, the slippage check requires `amount0 >= params.amount0Min && amount1 >= params.amount1Min` while ignoring `amount0Desired` and `amount1Desired`. Because `ILOPool.launch` sets `amount0Min` or `amount1Min` to `totalRaised` and computes `liquidity` via downward-rounding math, `pool.mint` frequently returns an actual token amount strictly less than `totalRaised`, permanently bricking pool launch.

### Technical details

When launching an ILO pool, `ILOPool.launch()` (lines 285–303 of `src/ILOPool.sol`) determines the `liquidity` and token amounts to mint into the Uniswap V3 pool based on `totalRaised`:

```solidity
if (token0Addr == RAISE_TOKEN) {
    amount0 = totalRaised;
    amount0Min = totalRaised;
    (amount1, liquidity) = _saleAmountNeeded(totalRaised);
} else {
    (amount0, liquidity) = _saleAmountNeeded(totalRaised);
    amount1 = totalRaised;
    amount1Min = totalRaised;
}
```

In `_saleAmountNeeded(totalRaised)` (lines 382–397 of `src/ILOPool.sol`), `liquidity` is computed from `totalRaised` using `LiquidityAmounts.getLiquidityForAmount0` or `LiquidityAmounts.getLiquidityForAmount1`:

1. **When `RAISE_TOKEN == token1` (`SALE_TOKEN == token0`):**
   `LiquidityAmounts.getLiquidityForAmount1` computes:
   $$\text{liquidity} = \left\lfloor \frac{\text{totalRaised} \times 2^{96}}{\text{SQRT\_RATIO\_X96} - \text{SQRT\_RATIO\_LOWER\_X96}} \right\rfloor$$
   In `LiquidityManagement.addLiquidity` (lines 41–57 of `src/base/LiquidityManagement.sol`), `params.pool.mint` calculates the actual `amount1` owed using `SqrtPriceMath.getAmount1Delta`:
   $$\text{amount1} = \left\lceil \frac{\text{liquidity} \times (\text{SQRT\_RATIO\_X96} - \text{SQRT\_RATIO\_LOWER\_X96})}{2^{96}} \right\rceil$$
   Whenever $\text{SQRT\_RATIO\_X96} - \text{SQRT\_RATIO\_LOWER\_X96} > 2^{96}$ (i.e., a single unit of liquidity represents more than 1 wei of `token1`), the floor division in `getLiquidityForAmount1` truncates the remainder, resulting in $\text{amount1} < \text{totalRaised} = \text{params.amount1Min}$.

2. **When `RAISE_TOKEN == token0` (`SALE_TOKEN == token1`):**
   `LiquidityAmounts.getLiquidityForAmount0` first computes:
   $$\text{intermediate} = \left\lfloor \frac{\text{SQRT\_RATIO\_X96} \times \text{SQRT\_RATIO\_UPPER\_X96}}{2^{96}} \right\rfloor$$
   and then:
   $$\text{liquidity} = \left\lfloor \frac{\text{totalRaised} \times \text{intermediate}}{\text{SQRT\_RATIO\_UPPER\_X96} - \text{SQRT\_RATIO\_X96}} \right\rfloor$$
   Because both `intermediate` and `liquidity` round down, whereas `SqrtPriceMath.getAmount0Delta` in `pool.mint` divides directly by `SQRT_RATIO_UPPER_X96` and `SQRT_RATIO_X96` without the truncated `intermediate`, the resulting `amount0` returned by `pool.mint` is frequently strictly less than `totalRaised` ($\text{amount0} < \text{params.amount0Min}$).

In `LiquidityManagement.addLiquidity` (`src/base/LiquidityManagement.sol`, lines 48–56):

```solidity
(amount0, amount1) = params.pool.mint(
    address(this),
    TICK_LOWER,
    TICK_UPPER,
    params.liquidity,
    ""
);

require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
```

Because `addLiquidity` enforces `amount0 >= params.amount0Min && amount1 >= params.amount1Min` where `amount0Min` or `amount1Min` is `totalRaised`, any downward rounding of at least 1 wei causes `addLiquidity` to revert with `'Price slippage check'`. Additionally, `params.amount0Desired` and `params.amount1Desired` in `AddLiquidityParams` (lines 34–35) are completely unused, so upper expenditure bounds are not checked in `addLiquidity`.

### Assumptions and preconditions

1. An ILO project and `ILOPool` are initialized with valid price and tick boundaries where either:
   - `RAISE_TOKEN == token1` and `SQRT_RATIO_X96 - SQRT_RATIO_LOWER_X96 > 2**96`, or
   - `RAISE_TOKEN == token0` and `SQRT_RATIO_X96` / `SQRT_RATIO_UPPER_X96` cause `intermediate` or `liquidity` truncation in `getLiquidityForAmount0`.
2. Investors purchase ILO shares via `ILOPool.buy` such that `totalRaised >= saleInfo.softCap` and `totalRaised` is not an exact multiple of the liquidity step size.

### Proof of concept

1. Consider a project configured with parameters analogous to `Mock.t.sol` (`initialPoolPriceX96 = 2 * 2**96 = 158456325028528675187087900672`, `TICK_LOWER = -887270` with `SQRT_RATIO_LOWER_X96 = 4295343490`, `TICK_UPPER = 887270`), where `SALE_TOKEN` is `token0` and `RAISE_TOKEN` is `token1`.
2. Here, $\Delta\sqrt{P} = \text{SQRT\_RATIO\_X96} - \text{SQRT\_RATIO\_LOWER\_X96} = 158456325028528675182792557182 \approx 2 \times 2^{96}$.
3. Suppose investors buy in the sale such that `totalRaised = 90_000e18 + 1` (`90000000000000000000001`).
4. When `ILOManager.launch` calls `ILOPool.launch`, `_saleAmountNeeded(totalRaised)` computes:
   $$\text{liquidity} = \left\lfloor \frac{90000000000000000000001 \times 2^{96}}{158456325028528675182792557182} \right\rfloor = 45000000000000000001219$$
5. `ILOPool.launch` sets `amount1Min = totalRaised = 90000000000000000000001` and calls `addLiquidity`.
6. Inside `LiquidityManagement.addLiquidity`, `params.pool.mint` computes `amount1` via `SqrtPriceMath.getAmount1Delta`:
   $$\text{amount1} = \left\lceil \frac{45000000000000000001219 \times 158456325028528675182792557182}{2^{96}} \right\rceil = 90000000000000000000000$$
7. At line 56 of `src/base/LiquidityManagement.sol`:
   `require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');`
   evaluates `90000000000000000000000 >= 90000000000000000000001`, which is `false`, reverting the transaction with `'Price slippage check'`.
8. Even if `totalRaised` initially lands on an exact boundary where no truncation occurs, any whitelisted buyer can call `ILOPool.buy` with a small `raiseAmount` before the sale ends so that `totalRaised` falls on a value that rounds down, guaranteeing that `ILOManager.launch` reverts.

### Impact assessment

- **Severity:** High
- **Technical Impact:** `LiquidityManagement.addLiquidity` reverts with `'Price slippage check'` during `ILOPool.launch()`. Because `ILOManager.launch()` iterates over all initialized pools and requires every `IILOPool(initializedPools[i]).launch()` call to succeed, a revert in `addLiquidity` prevents the entire project from launching.
- **Business Impact:** Projects that successfully reach their soft cap (or hard cap) cannot launch their liquidity pools and must wait until `refundDeadline` passes to unwind the sale via refunds, resulting in a complete denial of service of the core ILO launch functionality.

### Remediation

Compute `liquidity` inside `addLiquidity` using `LiquidityAmounts.getLiquidityForAmounts` from `amount0Desired` and `amount1Desired` (or account for rounding tolerance in `amount0Min` and `amount1Min` and enforce `amount0 <= params.amount0Desired && amount1 <= params.amount1Desired`), and allow a minor slippage/rounding tolerance on `amount0Min` and `amount1Min`:

```solidity
    struct AddLiquidityParams {
        IUniswapV3Pool pool;
        uint128 liquidity;
        uint256 amount0Desired;
        uint256 amount1Desired;
        uint256 amount0Min;
        uint256 amount1Min;
    }

    /// @notice Add liquidity to an initialized pool
    function addLiquidity(AddLiquidityParams memory params)
        internal
        returns (
            uint256 amount0,
            uint256 amount1
        )
    {
        (amount0, amount1) = params.pool.mint(
            address(this),
            TICK_LOWER,
            TICK_UPPER,
            params.liquidity,
            ""
        );

        require(amount0 <= params.amount0Desired && amount1 <= params.amount1Desired, 'Desired amount exceeded');
        require(
            amount0 >= FullMath.mulDiv(params.amount0Min, BPS - 10, BPS) &&
            amount1 >= FullMath.mulDiv(params.amount1Min, BPS - 10, BPS),
            'Price slippage check'
        );
    }
```
