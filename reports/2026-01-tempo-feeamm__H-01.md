# Audit report — 2026-01-tempo-feeamm/H-01

## Finding 1: Valuation Inconsistency Between `FeeAMM.mint` and `FeeAMM.burn` Enables Reserve-Composition-Dependent LP Dilution

### Synopsis

In `FeeAMM.mint`, LP share issuance prices existing `reserveUserToken` at a discounted rate (`N / SCALE = 9985 / 10000`) while accepting single-sided `validatorToken` deposits at par, whereas `FeeAMM.burn` redeems both reserves at unweighted pro-rata parity, enabling transaction-order-sensitive LP dilution.

### Technical details

`FeeAMM` maintains two reserves per pool: `reserveUserToken` and `reserveValidatorToken`. Swappers trade `userToken` for `validatorToken` via `executeFeeSwap` at rate `M / SCALE` (`0.9970`), increasing `reserveUserToken` and decreasing `reserveValidatorToken`, and rebalance via `rebalanceSwap` at rate `N / SCALE` (`0.9985`).

When liquidity is added to an already-initialized pool in `FeeAMM.mint` (lines 194–196), the minter deposits only `validatorToken` (`amountValidatorToken`) and receives `liquidity` shares computed as:

```solidity
uint256 product = (N * uint256(pool.reserveUserToken)) / SCALE;
uint256 denom = uint256(pool.reserveValidatorToken) + product;
liquidity = (amountValidatorToken * _totalSupply) / denom;
```

This share issuance formula creates three coupled economic flaws:

1. **Mint-Redemption Valuation Inconsistency**: During `mint`, the pool's existing `reserveUserToken` is valued at the discounted factor `N / SCALE` (`0.9985`, truncated downward by integer division), while the incoming `amountValidatorToken` is credited at full face value (`1.0`). However, during redemption in `burn` / `_calculateBurnAmounts` (lines 280–285), every LP share claims a direct pro-rata slice of both `pool.reserveUserToken` and `pool.reserveValidatorToken` without applying the `N / SCALE` discount.
2. **Reserve-Composition-Dependent Dilution**: Because `denom` shrinks as a larger fraction of pool reserves is held in `reserveUserToken` rather than `reserveValidatorToken`, minting single-sided `validatorToken` when `reserveUserToken > 0` issues disproportionately more LP shares relative to nominal reserves and dilutes existing LPs' ownership of `reserveUserToken` and accrued swap fees. In the extreme case where `reserveValidatorToken == 0` and `reserveUserToken == 1`, `product` truncates to `0` and `denom` becomes `0`, causing line 196 to revert due to division by zero.
3. **Transaction-Order-Sensitive Share Issuance**: The number of shares minted for a fixed `amountValidatorToken` depends on whether `mint` executes before or after `executeFeeSwap` / `rebalanceSwap`. An attacker can mint immediately after `executeFeeSwap` shifts pool composition into `reserveUserToken` (when `denom` is lowest) and immediately call `burn` to withdraw a pro-rata share of both `userToken` and `validatorToken`, capturing part of the `1 - N / SCALE` (`0.15%`) valuation discount at the expense of existing LPs.

### Assumptions and preconditions

1. A pool for `(userToken, validatorToken)` has been initialized with liquidity via `FeeAMM.mint`.
2. At least one `executeFeeSwap` transaction has occurred so that `pool.reserveUserToken > 0`.
3. An actor holds `validatorToken` and calls `mint` (optionally followed by `burn`) while `pool.reserveUserToken > 0`.

### Proof of concept

1. **Initial Liquidity Setup**:
   - LP1 calls `mint(userToken, validatorToken, 200_000e6, lp1)`.
   - Pool state: `reserveValidatorToken = 200_000e6`, `reserveUserToken = 0`, `totalSupply = 100_000e6`, `liquidityBalances[lp1] = 99_999_000`.
2. **Fee Swap Shifts Reserve Composition**:
   - A swapper calls `executeFeeSwap(userToken, validatorToken, 200_000e6)`.
   - `amountOut = (200_000e6 * 9970) / 10000 = 199_400e6`.
   - Pool state: `reserveUserToken = 200_000e6`, `reserveValidatorToken = 600e6`, `totalSupply = 100_000e6`. Total nominal reserves in the pool are `200_600e6` (`200_000e6` `userToken` + `600e6` `validatorToken`).
3. **Dilutive Mint**:
   - In `mint`, `product = (9985 * 200_000e6) / 10000 = 199_700e6` and `denom = 600e6 + 199_700e6 = 200_300e6`.
   - Attacker calls `mint(userToken, validatorToken, 200_300e6, attacker)`.
   - At line 196, `liquidity = (200_300e6 * 100_000e6) / 200_300e6 = 100_000e6`.
   - Pool state: `reserveUserToken = 200_000e6`, `reserveValidatorToken = 200_900e6`, `totalSupply = 200_000e6`.
4. **Immediate Burn / Redemption**:
   - Attacker calls `burn(userToken, validatorToken, 100_000e6, attacker)` to redeem their 50% share of `totalSupply`.
   - Attacker receives `amountUserToken = 100_000e6` and `amountValidatorToken = 100_450e6` (totaling `200_450e6` in nominal USD tokens for a `200_300e6` deposit).
   - LP1's remaining pool reserves drop from `200_600e6` to `200_450e6` (`100_000e6` `userToken` + `100_450e6` `validatorToken`), losing `150e6` of accrued swap fees to the attacker in a single atomic transaction.

### Impact assessment

- **Severity**: Medium / High.
- **Technical Impact**: LP share issuance in `FeeAMM.mint` is inconsistent with share redemption in `FeeAMM.burn` and depends on transient reserve composition. Existing LPs suffer dilution of up to `(1 - N / SCALE)` (`0.15%`) of `reserveUserToken` per mint-burn cycle whenever `reserveUserToken > 0`, and `mint` reverts with division by zero if `reserveValidatorToken == 0` and `(N * reserveUserToken) < SCALE`.
- **Business Impact**: Arbitrageurs can atomically sandwich or time `mint` and `burn` around fee swaps to siphon accrued swap fees from passive liquidity providers, disincentivizing liquidity provision.

### Remediation

Ensure that `mint` and `burn` use a consistent reserve valuation model, or require liquidity providers to deposit proportional amounts of both `userToken` and `validatorToken` when `reserveUserToken > 0` (and round `denom` upward in favor of the pool when computing `liquidity` in single-sided mints so existing LPs are never diluted):

```solidity
        if (pool.reserveUserToken == 0 && pool.reserveValidatorToken == 0) {
            if (amountValidatorToken / 2 <= MIN_LIQUIDITY) {
                revert("INSUFFICIENT_LIQUIDITY");
            }
            liquidity = amountValidatorToken / 2 - MIN_LIQUIDITY;
            totalSupply[poolId] += MIN_LIQUIDITY;
        } else {
            uint256 denom = uint256(pool.reserveValidatorToken) +
                uint256(pool.reserveUserToken);
            liquidity = (amountValidatorToken * _totalSupply) / denom;
        }
```

## Finding 2: Reentrancy and Stale-State Accounting in `FeeAMM.mint` Allow Pool Reserve Manipulation and LP Share Inflation

### Synopsis

`FeeAMM.mint` performs an external `IERC20(validatorToken).transferFrom` call before updating `pool.reserveValidatorToken`, `totalSupply[poolId]`, and `liquidityBalances[poolId][to]`. If `validatorToken` supports transfer callbacks, an attacker can re-enter `FeeAMM` while pool state is stale or partially initialized, corrupting LP accounting and draining pool reserves.

### Technical details

In `FeeAMM.mint` (`contracts/FeeAMM.sol`), the function reads `pools[poolId]` and caches `_totalSupply = totalSupply[poolId]` (lines 184–185), computes `liquidity` (lines 187–197), and then executes an external call to `IERC20(validatorToken).transferFrom(msg.sender, address(this), amountValidatorToken)` (lines 205–209) before updating the pool reserves and LP token supply/balances (lines 211–214):

```solidity
        } else {
            uint256 product = (N * uint256(pool.reserveUserToken)) / SCALE;
            uint256 denom = uint256(pool.reserveValidatorToken) + product;
            liquidity = (amountValidatorToken * _totalSupply) / denom;
        }
...
        IERC20(validatorToken).transferFrom(
            msg.sender,
            address(this),
            amountValidatorToken
        );

        pool.reserveValidatorToken += uint128(amountValidatorToken);

        totalSupply[poolId] += liquidity;
        liquidityBalances[poolId][to] += liquidity;
```

This violates the Checks-Effects-Interactions (CEI) pattern and lacks reentrancy protection (`nonReentrant`), creating two interrelated state-consistency flaws when `validatorToken` invokes a callback during `transferFrom` (e.g., ERC-777 `tokensToSend` / `tokensReceived` hooks or callback-enabled token implementations):

1. **Partially Initialized State Exposure on Initial Mint**:
   When a pool is empty (`pool.reserveUserToken == 0 && pool.reserveValidatorToken == 0`), line 192 increments `totalSupply[poolId] += MIN_LIQUIDITY`, but `pool.reserveValidatorToken` remains `0` and `liquidity` is not added to `totalSupply[poolId]` until after `transferFrom` returns. During the `transferFrom` callback:
   - A re-entrant call to `mint` still observes `pool.reserveUserToken == 0 && pool.reserveValidatorToken == 0`, entering the initialization branch a second time and adding `MIN_LIQUIDITY` to `totalSupply[poolId]` again. This permanently breaks the protocol invariant `totalSupply[poolId] == sum(liquidityBalances) + MIN_LIQUIDITY`.
   - Furthermore, because `totalSupply[poolId]` has already been incremented by `MIN_LIQUIDITY` for each nested initial mint while `pool.reserveValidatorToken` only reflects completed inner mints, a subsequent re-entrant `mint` in the `else` branch (line 196) calculates `liquidity = (amountValidatorToken * _totalSupply) / denom` using an inflated `_totalSupply` relative to `denom`, minting disproportionately large `liquidity` to the caller.

2. **Cross-Function Reentrancy and Stale-State Accounting**:
   Because `mint` caches `_totalSupply` and computes `liquidity` at line 196 prior to `transferFrom`, and delays updating `pool.reserveValidatorToken`, `totalSupply[poolId]`, and `liquidityBalances[poolId][to]` until lines 211–214 (and similarly `burn` transfers tokens before updating reserves and `totalSupply`), re-entering across `mint`, `burn`, or swap functions allows operations to execute against stale `pool.reserveValidatorToken` and `totalSupply[poolId]` values.

### Assumptions and preconditions

1. `validatorToken` (or `userToken` when combined with `burn`) is an ERC-20 token implementation that triggers a callback to the sender or recipient during transfers (such as an ERC-777 token or a token with transfer hooks).
2. An attacker can call `FeeAMM.mint` (and optionally `FeeAMM.burn`) and execute re-entrant calls to `FeeAMM` within the token transfer callback.

### Proof of concept

Consider a pool where `validatorToken` invokes a `tokensToSend` callback on `msg.sender` during `transferFrom`:

1. **Initial Call (`mint` #1)**:
   The pool is initially empty (`reserveUserToken = 0`, `reserveValidatorToken = 0`, `totalSupply = 0`).
   Party A (or an attacker) calls `mint(userToken, validatorToken, 200_000, attacker)`:
   - Line 187: `reserveUserToken == 0 && reserveValidatorToken == 0` is `true`.
   - Line 191: `liquidity` is computed as `200_000 / 2 - 1000 = 99_000`.
   - Line 192: `totalSupply[poolId]` is updated to `1000`.
   - Line 205: `validatorToken.transferFrom` is called while `pool.reserveValidatorToken` is still `0` and `totalSupply[poolId]` is `1000`.

2. **First Re-entrant Call (`mint` #2 inside `transferFrom` callback)**:
   Inside the callback of `mint` #1, the attacker calls `mint(userToken, validatorToken, 2002, attacker)`:
   - Line 187: `pool.reserveUserToken == 0 && pool.reserveValidatorToken == 0` is still `true` because `mint` #1 has not yet updated `pool.reserveValidatorToken`.
   - Line 191: `liquidity` is computed as `2002 / 2 - 1000 = 1`.
   - Line 192: `totalSupply[poolId]` is incremented by `1000` again, becoming `2000`.
   - Lines 205–214: `mint` #2 completes, setting `pool.reserveValidatorToken = 2002`, `totalSupply[poolId] = 2001`, and `liquidityBalances[poolId][attacker] = 1`.

3. **Second Re-entrant Call (`mint` #3 inside `transferFrom` callback)**:
   Still inside the callback of `mint` #1, the attacker calls `mint(userToken, validatorToken, 200_000, attacker)`:
   - Line 185: `_totalSupply` is read as `2001`.
   - Line 187: `pool.reserveValidatorToken` is now `2002` (non-zero), so execution enters the `else` branch (line 193).
   - Line 195: `denom = 2002 + 0 = 2002`.
   - Line 196: `liquidity = (200_000 * 2001) / 2002 = 199_900` (nearly 1:1 LP tokens per validator token, instead of the intended 1:2 ratio of `100_000` LP tokens).
   - Lines 205–214: `mint` #3 completes, updating `pool.reserveValidatorToken = 202_002`, `totalSupply[poolId] = 201_901`, and `liquidityBalances[poolId][attacker] = 199_901`.

4. **Completion of Outer Call (`mint` #1)**:
   Control returns to line 211 of `mint` #1:
   - `pool.reserveValidatorToken` becomes `202_002 + 200_000 = 402_002`.
   - `totalSupply[poolId]` becomes `201_901 + 99_000 = 300_901`.
   - Notice that `MIN_LIQUIDITY` (`1000`) was added to `totalSupply[poolId]` twice, violating `totalSupply == sum(liquidityBalances) + MIN_LIQUIDITY`, and `mint` #3 received `199_900` LP tokens for `200_000` validator tokens while `mint` #1 received only `99_000` LP tokens for the same `200_000` validator tokens.

### Impact assessment

- **Severity**: High
- **Direct Technical Effects**:
  - Breaks the core accounting invariant `totalSupply[poolId] == sum(liquidityBalances) + MIN_LIQUIDITY`.
  - Allows an attacker to mint LP shares at a distorted `_totalSupply / denom` ratio by exploiting partially updated state across nested calls.
- **Business Implications**:
  - Honest liquidity providers suffer direct dilution of their pool share and loss of underlying `validatorToken` and `userToken` reserves when the attacker burns the inflated LP tokens.

### Remediation

Apply the Checks-Effects-Interactions (CEI) pattern in `FeeAMM.mint` by updating `pool.reserveValidatorToken`, `totalSupply[poolId]`, and `liquidityBalances[poolId][to]` before calling `IERC20(validatorToken).transferFrom`, and add a reentrancy guard (`nonReentrant`) to all state-modifying functions in `FeeAMM`.

```solidity
    function mint(
        address userToken,
        address validatorToken,
        uint256 amountValidatorToken,
        address to
    ) external returns (uint256 liquidity) {
        require(userToken != validatorToken, "IDENTICAL_ADDRESSES");
        require(amountValidatorToken != 0, "INVALID_AMOUNT");

        bytes32 poolId = getPoolId(userToken, validatorToken);

        Pool storage pool = pools[poolId];
        uint256 _totalSupply = totalSupply[poolId];

        if (pool.reserveUserToken == 0 && pool.reserveValidatorToken == 0) {
            if (amountValidatorToken / 2 <= MIN_LIQUIDITY) {
                revert("INSUFFICIENT_LIQUIDITY");
            }
            liquidity = amountValidatorToken / 2 - MIN_LIQUIDITY;
            totalSupply[poolId] += MIN_LIQUIDITY;
        } else {
            uint256 product = (N * uint256(pool.reserveUserToken)) / SCALE;
            uint256 denom = uint256(pool.reserveValidatorToken) + product;
            liquidity = (amountValidatorToken * _totalSupply) / denom;
        }

        if (liquidity == 0) {
            revert("INSUFFICIENT_LIQUIDITY");
        }

        _requireU128(amountValidatorToken);
        require(
            uint256(pool.reserveValidatorToken) + amountValidatorToken <= type(uint128).max,
            "INSUFFICIENT_RESERVES"
        );

        pool.reserveValidatorToken += uint128(amountValidatorToken);

        totalSupply[poolId] += liquidity;
        liquidityBalances[poolId][to] += liquidity;

        IERC20(validatorToken).transferFrom(
            msg.sender,
            address(this),
            amountValidatorToken
        );

        emit Mint(
            msg.sender,
            to,
            userToken,
            validatorToken,
            amountValidatorToken,
            liquidity
        );
    }
```

## Finding 3: Unchecked `uint128` Reserve Accumulator Addition in `FeeAMM.mint` Allows `reserveValidatorToken` Overflow

### Synopsis

In `FeeAMM.mint`, `_requireU128` only validates that `amountValidatorToken` fits in `uint128`, not that the updated reserve `uint256(pool.reserveValidatorToken) + amountValidatorToken` fits in `uint128`. Under Solidity `0.7.6`, `pool.reserveValidatorToken += uint128(amountValidatorToken)` silently wraps around on overflow, corrupting pool reserves and share pricing.

### Technical details

`FeeAMM` is compiled with Solidity `0.7.6` (`foundry.toml`), where unsigned integer arithmetic wraps on overflow unless explicitly checked. Pool reserves are stored in the `Pool` struct as `uint128`:

```solidity
struct Pool {
    uint128 reserveUserToken;
    uint128 reserveValidatorToken;
}
```

In `rebalanceSwap` (lines 146–150), the contract explicitly checks that adding `amountIn` to `pool.reserveValidatorToken` does not exceed `type(uint128).max`. However, in `mint` (lines 196–223), the contract only checks the input argument in isolation at line 203 before incrementing the `uint128` reserve accumulator at line 211:

```solidity
liquidity = (amountValidatorToken * _totalSupply) / denom;
...
_requireU128(amountValidatorToken);

IERC20(validatorToken).transferFrom(
    msg.sender,
    address(this),
    amountValidatorToken
);

pool.reserveValidatorToken += uint128(amountValidatorToken);
```

Because `_requireU128(amountValidatorToken)` only enforces `amountValidatorToken <= type(uint128).max`, any call to `mint` where `uint256(pool.reserveValidatorToken) + amountValidatorToken > type(uint128).max` passes validation and causes `pool.reserveValidatorToken` at line 211 to silently wrap modulo $2^{128}$.

Once `pool.reserveValidatorToken` wraps to a small value:
1. $2^{128}$ units of `validatorToken` held by `FeeAMM` become unaccounted for in `pool.reserveValidatorToken`, so LPs burning shares via `burn()` only receive a fraction of the underlying validator tokens.
2. In subsequent `mint()` calls, the denominator `denom = uint256(pool.reserveValidatorToken) + product` is drastically understated relative to `_totalSupply`, allowing a subsequent caller to mint disproportionately large `liquidity` for a tiny `amountValidatorToken` deposit and drain `pool.reserveUserToken` via `burn()`.

### Assumptions and preconditions

1. The total circulating supply available to depositors of `validatorToken` (e.g., high-decimal tokens or tokens with large supply) can exceed `type(uint128).max`, or multiple `mint` operations accumulate `reserveValidatorToken` past `type(uint128).max`.
2. A caller invokes `FeeAMM.mint` with `amountValidatorToken <= type(uint128).max` such that `uint256(pool.reserveValidatorToken) + amountValidatorToken > type(uint128).max`.

### Proof of concept

1. Suppose a pool is initialized via `mint(userToken, validatorToken, amount1, lp1)` with `amount1 = type(uint128).max - 1e6`.
   - `pool.reserveValidatorToken` becomes `type(uint128).max - 1e6`.
   - `totalSupply[poolId]` becomes `(type(uint128).max - 1e6) / 2`.
2. A swapper executes `executeFeeSwap` with `10_000e6` of `userToken`, increasing `pool.reserveUserToken` by `10_000e6` and reducing `pool.reserveValidatorToken` by `9_970e6` (`pool.reserveValidatorToken = type(uint128).max - 10_970e6`).
3. Calling `mint(userToken, validatorToken, 10_970e6 + 100, attacker)` passes `_requireU128(amountValidatorToken)` at line 203.
4. At line 211, `pool.reserveValidatorToken += uint128(10_970e6 + 100)` overflows `uint128` and wraps around to `99`, while `totalSupply[poolId]` remains around `2^{127}`.
5. Now `denom = 99 + (9985 * 10_000e6) / 10_000 = 9_985_000_099`, which is orders of magnitude smaller than `totalSupply[poolId]`.
6. In a second `mint` call with a small amount of `validatorToken`, the caller receives an enormous amount of `liquidity` shares (`amountValidatorToken * _totalSupply / denom`), diluting `lp1` to nearly 0% of the pool and allowing the caller to `burn` those shares to extract almost all of `pool.reserveUserToken`.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Silent `uint128` overflow of `pool.reserveValidatorToken`, breaking the core reserve accounting invariant (`reserveValidatorToken <= IERC20(validatorToken).balanceOf(address(this))`) and corrupting LP share pricing in `mint` and `burn`.
- **Business Impact**: Permanent freezing of $2^{128}$ validator tokens per overflow and theft of accumulated `userToken` reserves from existing liquidity providers through share dilution.

### Remediation

Validate that the new cumulative `reserveValidatorToken` fits within `uint128` before updating the reserve in `mint`.

```solidity
        if (liquidity == 0) {
            revert("INSUFFICIENT_LIQUIDITY");
        }

        _requireU128(uint256(pool.reserveValidatorToken) + amountValidatorToken);

        IERC20(validatorToken).transferFrom(
            msg.sender,
            address(this),
            amountValidatorToken
        );

        pool.reserveValidatorToken += uint128(amountValidatorToken);
```

## Finding 4: Reentrancy in `FeeAMM.burn` via External Token Transfers Before State Updates Allows Draining Pool Reserves

### Synopsis

In `FeeAMM.burn`, external token transfers (`IERC20(userToken).transfer` and `IERC20(validatorToken).transfer`) are executed before updating `liquidityBalances`, `totalSupply`, and pool reserves. A malicious or hook-enabled token (e.g., ERC-777) or recipient callback can reenter `FeeAMM` against stale state and drain pool funds.

### Technical details

In `contracts/FeeAMM.sol`, the `burn` function (lines 226–269) violates the Checks-Effects-Interactions (CEI) pattern and lacks a reentrancy guard:

1. At line 239, `burn` checks `liquidityBalances[poolId][msg.sender] < liquidity`.
2. At lines 243–247, `_calculateBurnAmounts(pool, poolId, liquidity)` computes `amountUserToken` and `amountValidatorToken` using the current `pool.reserveUserToken`, `pool.reserveValidatorToken`, and `totalSupply[poolId]`.
3. At lines 252–253, `burn` performs external calls to transfer tokens to `to`:
   ```solidity
   IERC20(userToken).transfer(to, amountUserToken);
   IERC20(validatorToken).transfer(to, amountValidatorToken);
   ```
4. Only after both external token transfers complete does `burn` update the internal accounting state at lines 255–258:
   ```solidity
   liquidityBalances[poolId][msg.sender] -= liquidity;
   totalSupply[poolId] -= liquidity;
   pool.reserveUserToken -= uint128(amountUserToken);
   pool.reserveValidatorToken -= uint128(amountValidatorToken);
   ```

Because `FeeAMM` is compiled with Solidity `0.7.6` (without automatic arithmetic underflow checks) and performs external token transfers while `liquidityBalances[poolId][msg.sender]`, `totalSupply[poolId]`, `pool.reserveUserToken`, and `pool.reserveValidatorToken` still hold their pre-burn values:

- If `userToken` or `validatorToken` implements transfer hooks (such as ERC-777 `_callTokensReceived` / `tokensToSend`) or otherwise hands control to `msg.sender` or `to`, the caller can reenter `FeeAMM.burn` (or other pool functions like `mint`, `executeFeeSwap`, `rebalanceSwap`).
- Upon reentering `burn`, `liquidityBalances[poolId][msg.sender]` has not yet been decremented, so the balance check at line 239 passes again for the same LP shares, and `_calculateBurnAmounts` calculates payouts against the unreduced reserves.
- Furthermore, because Solidity `0.7.6` does not revert on integer underflow by default, even if partial amounts or repeated burns exceed the initial reserve or balance when the call stack unwinds, the unchecked subtractions at lines 255–258 can wrap around (`uint256` / `uint128` underflow), corrupting pool accounting and allowing complete drainage of token balances held by `FeeAMM`.

### Assumptions and preconditions

1. A pool exists with non-zero liquidity and reserves in `FeeAMM`.
2. Either `userToken` or `validatorToken` supports transfer callbacks/hooks (e.g., ERC-777 `tokensReceived`) or transfers control to an external contract during `transfer`.
3. The attacker holds a non-zero `liquidityBalances[poolId][msg.sender]` position in the pool.

### Proof of concept

1. Suppose a pool for `(userToken, validatorToken)` has `reserveUserToken = 100,000e6`, `reserveValidatorToken = 100,000e6`, and `totalSupply = 100,000e6`.
2. An attacker holds `50,000e6` LP shares (`liquidityBalances[poolId][attacker] = 50,000e6`).
3. The attacker calls `feeAmm.burn(userToken, validatorToken, 50_000e6, attacker)`.
4. `burn` verifies `liquidityBalances[poolId][attacker] >= 50,000e6` and calculates `amountUserToken = 50,000e6` and `amountValidatorToken = 50,000e6`.
5. `burn` executes `IERC20(userToken).transfer(attacker, 50,000e6)` at line 252.
6. During the token transfer callback to `attacker`, `liquidityBalances[poolId][attacker]` is still `50,000e6` and reserves are still `100,000e6`.
7. Inside the callback, `attacker` calls `feeAmm.burn(userToken, validatorToken, 50,000e6, attacker)` a second time.
8. The second `burn` call passes the check at line 239 (`50,000e6 < 50,000e6` is false), computes another `50,000e6` payout of both tokens, and transfers the remaining `50,000e6` of `userToken` and `validatorToken` to `attacker`.
9. Both `burn` frames finish and decrement `liquidityBalances`, `totalSupply`, and `pool` reserves twice, draining `100,000e6` of each token using only `50,000e6` of LP shares (and underflowing `liquidityBalances[poolId][attacker]` in Solidity 0.7.6).

### Impact assessment

- **Severity**: High / Critical
- **Technical Impact**: Complete drainage of `userToken` and `validatorToken` reserves from the pool, as well as underflow corruption of `liquidityBalances`, `totalSupply`, and pool reserves.
- **Business Impact**: Direct theft of liquidity providers' funds and total insolvency of the affected `FeeAMM` pools.

### Remediation

Apply the Checks-Effects-Interactions (CEI) pattern in `FeeAMM.burn` by updating `liquidityBalances`, `totalSupply`, `pool.reserveUserToken`, and `pool.reserveValidatorToken` before executing the external `IERC20.transfer` calls (and optionally add a `nonReentrant` guard).

```solidity
    function burn(
        address userToken,
        address validatorToken,
        uint256 liquidity,
        address to
    ) external returns (uint256 amountUserToken, uint256 amountValidatorToken) {
        require(userToken != validatorToken, "IDENTICAL_ADDRESSES");
        require(liquidity != 0, "INVALID_AMOUNT");

        bytes32 poolId = getPoolId(userToken, validatorToken);

        Pool storage pool = pools[poolId];

        if (liquidityBalances[poolId][msg.sender] < liquidity) {
            revert("INSUFFICIENT_LIQUIDITY");
        }

        (amountUserToken, amountValidatorToken) = _calculateBurnAmounts(
            pool,
            poolId,
            liquidity
        );

        _requireU128(amountUserToken);
        _requireU128(amountValidatorToken);

        liquidityBalances[poolId][msg.sender] -= liquidity;
        totalSupply[poolId] -= liquidity;
        pool.reserveUserToken -= uint128(amountUserToken);
        pool.reserveValidatorToken -= uint128(amountValidatorToken);

        IERC20(userToken).transfer(to, amountUserToken);
        IERC20(validatorToken).transfer(to, amountValidatorToken);

        emit Burn(
            msg.sender,
            userToken,
            validatorToken,
            amountUserToken,
            amountValidatorToken,
            liquidity,
            to
        );
    }
```

## Finding 5: Unchecked ERC20 `transfer` Return Values in `FeeAMM.burn` Allow Silent Transfer Failures and Permanent Loss of Liquidity

### Synopsis

In `FeeAMM.burn`, calls to `IERC20(userToken).transfer` and `IERC20(validatorToken).transfer` do not check boolean return values or use `SafeERC20.safeTransfer`. For tokens that return `false` on failure instead of reverting, transfers can fail silently while LP balances and pool reserves are still decremented, causing permanent loss of user funds.

### Technical details

In `contracts/FeeAMM.sol`, the `burn` function calculates the underlying token amounts owed to the liquidity provider (`amountUserToken` and `amountValidatorToken`) and executes raw `IERC20.transfer` calls at lines 252–253:

```solidity
IERC20(userToken).transfer(to, amountUserToken);
IERC20(validatorToken).transfer(to, amountValidatorToken);
```

According to the EIP-20 specification (and as reflected in the imported `@openzeppelin/contracts/token/ERC20/IERC20.sol`), `transfer` returns a `bool` indicating whether the operation succeeded, and standard-compliant tokens may return `false` rather than reverting when a transfer fails (e.g., due to insufficient actual token balance in the contract, paused state, blocklists, or zero-amount restrictions). Furthermore, non-standard tokens that do not return a boolean value at all will cause calls through the standard `IERC20` interface to revert during return-data ABI decoding.

Because `FeeAMM.burn` ignores the boolean return values of both `transfer` calls and does not use OpenZeppelin's `SafeERC20.safeTransfer`:
1. If either `IERC20(userToken).transfer(to, amountUserToken)` or `IERC20(validatorToken).transfer(to, amountValidatorToken)` returns `false`, execution continues uninterrupted to lines 255–258.
2. The caller's `liquidityBalances[poolId][msg.sender]` and `totalSupply[poolId]` are permanently reduced by `liquidity`.
3. `pool.reserveUserToken` and `pool.reserveValidatorToken` are decremented by `amountUserToken` and `amountValidatorToken` even though the tokens never left the contract.

As a result, the liquidity provider loses their LP shares without receiving the corresponding underlying tokens, and the uncollected tokens remain stranded in the `FeeAMM` contract outside tracked pool reserves.

### Assumptions and preconditions

1. A pool in `FeeAMM` is created using a `userToken` or `validatorToken` implementation that returns `false` on transfer failure instead of reverting (as permitted by the ERC-20 standard).
2. A liquidity provider holds a non-zero LP balance in the pool and calls `burn`.
3. One or both token transfers fail at the token level (for example, the `to` address is restricted/blacklisted on one of the tokens, transfers are temporarily paused, or the contract's actual token balance is lower than the tracked reserve).

### Proof of concept

1. Suppose `userToken` is an ERC-20 token that returns `false` when a transfer fails (e.g., when the recipient `to` is restricted or the token is paused), while `validatorToken` is a standard ERC-20 token.
2. A liquidity provider (`lp`) holds `100,000e6` LP shares in the `(userToken, validatorToken)` pool, corresponding to `50,000e6` `userToken` and `150,000e6` `validatorToken`.
3. `lp` calls `feeAmm.burn(address(userToken), address(validatorToken), 100_000e6, to)`.
4. `_calculateBurnAmounts` computes `amountUserToken = 50,000e6` and `amountValidatorToken = 150,000e6`.
5. At line 252, `IERC20(userToken).transfer(to, 50_000e6)` fails and returns `false`. Because the return value is ignored, the transaction does not revert.
6. At line 253, `IERC20(validatorToken).transfer(to, 150_000e6)` succeeds.
7. At lines 255–258, `liquidityBalances[poolId][lp]` and `totalSupply[poolId]` are decreased by `100,000e6`, and `pool.reserveUserToken` is decreased by `50,000e6`.
8. `lp` has permanently burned all `100,000e6` LP shares, never received the `50,000e6` `userToken`, and those `50,000e6` `userToken` remain permanently stuck in `FeeAMM` because `pool.reserveUserToken` was already decremented.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: Silent failure of ERC-20 transfers during `burn` causes pool accounting (`liquidityBalances`, `totalSupply`, `reserveUserToken`, and `reserveValidatorToken`) to be updated as if the payout succeeded when no tokens (or only one of the two tokens) were actually transferred.
- **Business Implications**: Liquidity providers suffer direct, irreversible loss of their redeemed token entitlements, and the untransferred tokens become permanently locked in the contract outside tracked reserves.

### Remediation

Use OpenZeppelin's `SafeERC20.safeTransfer` (or explicitly wrap the `transfer` calls in `require(...)`) to ensure that any transfer returning `false` (or failing with non-standard return data) reverts the transaction.

```solidity
    function burn(
        address userToken,
        address validatorToken,
        uint256 liquidity,
        address to
    ) external returns (uint256 amountUserToken, uint256 amountValidatorToken) {
        require(userToken != validatorToken, "IDENTICAL_ADDRESSES");
        require(liquidity != 0, "INVALID_AMOUNT");

        bytes32 poolId = getPoolId(userToken, validatorToken);

        Pool storage pool = pools[poolId];

        if (liquidityBalances[poolId][msg.sender] < liquidity) {
            revert("INSUFFICIENT_LIQUIDITY");
        }

        (amountUserToken, amountValidatorToken) = _calculateBurnAmounts(
            pool,
            poolId,
            liquidity
        );

        _requireU128(amountUserToken);
        _requireU128(amountValidatorToken);

        SafeERC20.safeTransfer(IERC20(userToken), to, amountUserToken);
        SafeERC20.safeTransfer(IERC20(validatorToken), to, amountValidatorToken);

        liquidityBalances[poolId][msg.sender] -= liquidity;
        totalSupply[poolId] -= liquidity;
        pool.reserveUserToken -= uint128(amountUserToken);
        pool.reserveValidatorToken -= uint128(amountValidatorToken);

        emit Burn(
            msg.sender,
            userToken,
            validatorToken,
            amountUserToken,
            amountValidatorToken,
            liquidity,
            to
        );
    }
```
