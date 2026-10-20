# Audit report — 2024-06-vultisig/H-03

## Finding 1: Spot Price Manipulation in `ILOManager.launch` Enables Denial of Service of Project Launch

### Synopsis

`ILOManager.launch` enforces a strict equality check between the Uniswap V3 pool's current spot `sqrtPriceX96` and the project's cached `initialPoolPriceX96`. Because the Uniswap V3 pool is initialized upon project creation and is publicly accessible, any actor can alter `slot0.sqrtPriceX96` via a swap and cause `launch` to revert.

### Technical details

When a project is initialized via `ILOManager.initProject`, the contract creates and initializes the corresponding Uniswap V3 pool at `params.initialPoolPriceX96` (line 61) and records this value in `_cachedProject[uniV3PoolAddress].initialPoolPriceX96` (line 142).

Later, when `ILOManager.launch(address uniV3PoolAddress)` is called after `launchTime`, it checks the pool's current spot price in `slot0()`:

```solidity
(uint160 sqrtPriceX96, , , , , , ) = IUniswapV3Pool(uniV3PoolAddress).slot0();
require(_cachedProject[uniV3PoolAddress].initialPoolPriceX96 == sqrtPriceX96, "UV3P");
```

In Uniswap V3, `UniswapV3Pool.swap` is permissionless once the pool is initialized. Before `ILOManager.launch` is executed:
1. If the pool has zero in-range liquidity, any user can call `swap` with a tiny `amountSpecified` and a chosen `sqrtPriceLimitX96`, which shifts `slot0.sqrtPriceX96` directly to `sqrtPriceLimitX96` at zero token cost (`amountIn = 0`, `amountOut = 0`) because `SwapMath.computeSwapStep` advances the price across uninitialized ticks when `liquidity == 0`.
2. Even if someone attempts to restore the exact `initialPoolPriceX96`, an attacker can either front-run the `launch` transaction with a minimal swap or mint concentrated liquidity in the pool around a different tick so that restoring the exact `uint160` `sqrtPriceX96` value requires significant capital or becomes practically infeasible due to rounding in `SqrtPriceMath`.

Consequently, the `require(_cachedProject[uniV3PoolAddress].initialPoolPriceX96 == sqrtPriceX96, "UV3P")` check on line 190 fails, preventing `IILOPool(initializedPools[i]).launch()` from executing.

### Assumptions and preconditions

- A project and at least one associated `ILOPool` have been initialized via `ILOManager.initProject` and `ILOManager.initILOPool`.
- The Uniswap V3 pool associated with the project is initialized and publicly reachable by any external account.

### Proof of concept

1. The project admin calls `initProject` with `initialPoolPriceX96 = P_init` and `launchTime = T_launch`, which initializes the Uniswap V3 pool at `sqrtPriceX96 = P_init`.
2. Investors participate in the ILO sale via `ILOPool.buy` between `saleInfo.start` and `saleInfo.end`, meeting the `softCap`.
3. Prior to or by front-running `ILOManager.launch(uniV3PoolAddress)`, an attacker calls `IUniswapV3Pool(uniV3PoolAddress).swap(...)` with `zeroForOne = true` (or `false`), `amountSpecified = 1`, and `sqrtPriceLimitX96 = P_init - 1` (or `P_init + 1`).
4. Because liquidity in the pool is `0` (or minimal liquidity added by the attacker), `slot0.sqrtPriceX96` updates to `P_init - 1`.
5. When `ILOManager.launch(uniV3PoolAddress)` is called after `T_launch`, line 190 evaluates `_cachedProject[uniV3PoolAddress].initialPoolPriceX96 == sqrtPriceX96` (`P_init == P_init - 1`), which is `false`, reverting the transaction with `"UV3P"`.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Any third party can trivially and repeatedly block `ILOManager.launch` from succeeding at near-zero cost.
- **Business Impact:** Projects cannot deploy raised liquidity into Uniswap V3 or complete their ILO launch. If the denial of service persists until `refundDeadline`, investors can trigger refunds (`ILOPool.claimRefund`), permanently bricking the entire token launch.

### Remediation

Avoid initializing the Uniswap V3 pool in `initProject` if the pool does not yet exist, or programmatically restore the pool price to `initialPoolPriceX96` during `launch`, or restrict interactions with the sale token before launch so external actors cannot provide liquidity against it and restore the zero-liquidity price via a swap inside `launch` before minting ILO liquidity.

Specifically, if `ILOManager.launch` swaps the pool price back to `_cachedProject[uniV3PoolAddress].initialPoolPriceX96` (or initializes the pool during `launch` instead of `initProject`), the strict spot price precondition cannot be used to DoS the launch:

```solidity
    /// @inheritdoc IILOManager
    function launch(address uniV3PoolAddress) external override {
        require(block.timestamp > _cachedProject[uniV3PoolAddress].launchTime, "LT");
        (uint160 sqrtPriceX96, , , , , , ) = IUniswapV3Pool(uniV3PoolAddress).slot0();
        if (sqrtPriceX96 == 0) {
            IUniswapV3Pool(uniV3PoolAddress).initialize(_cachedProject[uniV3PoolAddress].initialPoolPriceX96);
        } else if (sqrtPriceX96 != _cachedProject[uniV3PoolAddress].initialPoolPriceX96) {
            bool zeroForOne = sqrtPriceX96 > _cachedProject[uniV3PoolAddress].initialPoolPriceX96;
            IUniswapV3Pool(uniV3PoolAddress).swap(
                address(this),
                zeroForOne,
                type(int256).max,
                _cachedProject[uniV3PoolAddress].initialPoolPriceX96,
                ""
            );
        }
        address[] memory initializedPools = _initializedILOPools[uniV3PoolAddress];
        require(initializedPools.length > 0, "NP");
        for (uint256 i = 0; i < initializedPools.length; i++) {
            IILOPool(initializedPools[i]).launch();
        }

        emit ProjectLaunch(uniV3PoolAddress);
    }
```

## Finding 2: Missing `refundDeadline` Validation in `ILOManager.launch` Allows Project Launch After Refund Period Begins

### Synopsis

`ILOManager.launch` verifies that `block.timestamp` is greater than `launchTime`, but fails to verify that `block.timestamp` is strictly before `refundDeadline`. Consequently, a project can still be launched after its refund deadline has passed, conflicting with the refund phase and locking investor funds.

### Technical details

In `src/ILOManager.sol`, each project is configured with a `launchTime` and a `refundDeadline` (initialized in `initProject` as `params.launchTime + DEFAULT_DEADLINE_OFFSET` and modifiable via `setRefundDeadlineForProject`). By design, the window during which a project may launch is between `launchTime` and `refundDeadline`; once `block.timestamp >= refundDeadline`, the project enters the refund phase where investors can claim back their `RAISE_TOKEN` via `ILOPool.claimRefund` (which checks `require(block.timestamp >= _project.refundDeadline, "RFT")`) and the project admin can recover `SALE_TOKEN` via `ILOManager.claimRefund` (which checks `require(_cachedProject[uniV3PoolAddress].refundDeadline < block.timestamp, "RFT")`).

However, `ILOManager.launch` (lines 187–195) only checks the lower time bound:

```solidity
require(block.timestamp > _cachedProject[uniV3PoolAddress].launchTime, "LT");
```

Neither `ILOManager.launch` nor `ILOPool.launch` checks that `block.timestamp < _cachedProject[uniV3PoolAddress].refundDeadline`. Instead, `ILOPool.launch` only checks `require(!_refundTriggered, "IRF")`, where `_refundTriggered` is a boolean flag on each individual `ILOPool` that remains `false` until a user or the manager actually executes a refund transaction (`claimRefund` or `claimProjectRefund`) on that pool.

Because `ILOManager.launch` omits the upper bound check against `refundDeadline`:
1. After `block.timestamp >= _cachedProject[uniV3PoolAddress].refundDeadline`, both `ILOManager.launch` and `ILOPool.claimRefund` / `ILOManager.claimRefund` become simultaneously callable as long as `_refundTriggered` has not yet been set on any of the project's pools.
2. Anyone can call `ILOManager.launch(uniV3PoolAddress)` after `refundDeadline` (including by front-running the first `claimRefund` transaction from an investor or the project admin), which sets `_launchSucceeded = true` across all child `ILOPool` contracts, deploys the raised tokens into the Uniswap V3 pool, and permanently blocks investors and the project admin from claiming refunds (`require(!_launchSucceeded, "PL")`).

### Assumptions and preconditions

1. A project and one or more child `ILOPool` contracts have been initialized, the sale period has ended (`block.timestamp > launchTime`), and the pools have met their `softCap` and hold sufficient `SALE_TOKEN`.
2. The project was not launched prior to `_cachedProject[uniV3PoolAddress].refundDeadline` (for example, due to the Uniswap V3 pool price deviating from `initialPoolPriceX96` until after `refundDeadline`, or the project team deciding not to proceed with the launch).
3. `block.timestamp >= _cachedProject[uniV3PoolAddress].refundDeadline` has been reached, and no refund transaction has been mined yet on any of the project's `ILOPool` instances (`_refundTriggered == false`).

### Proof of concept

1. A project is initialized with `launchTime = T` and `refundDeadline = T + 7 days`, and an `ILOPool` reaches its `softCap` during the sale period.
2. Time advances past the refund deadline such that `block.timestamp == T + 7 days + 1`. The project has failed to launch within the allowed window and should now only allow refunds.
3. An investor (or the project admin) submits a transaction calling `ILOPool.claimRefund(tokenId)` (or `ILOManager.claimRefund(uniV3PoolAddress)`) to recover their tokens.
4. Before or ahead of the refund transaction being mined, a caller invokes `ILOManager.launch(uniV3PoolAddress)` while the Uniswap V3 pool's `sqrtPriceX96` equals `initialPoolPriceX96`.
5. `ILOManager.launch` passes `require(block.timestamp > _cachedProject[uniV3PoolAddress].launchTime, "LT")` because `T + 7 days + 1 > T`, and `ILOPool.launch()` passes `require(!_refundTriggered, "IRF")` because no refund has been processed yet.
6. The pools launch after `refundDeadline`, deploying the raised funds into Uniswap V3 and setting `_launchSucceeded = true`. The pending `claimRefund` transaction reverts with `"PL"`.

### Impact assessment

**Severity: Medium**

- **Technical Impact:** The lifecycle boundary between the launch period (`launchTime < block.timestamp < refundDeadline`) and the refund period (`block.timestamp >= refundDeadline`) is not enforced during `ILOManager.launch`. Both transitions remain active concurrently after `refundDeadline` until the first refund is claimed.
- **Business Impact:** Investors who participated in an ILO expecting a guaranteed refund if the project did not launch by `refundDeadline` can have their `RAISE_TOKEN` forcibly committed to a launched pool after the deadline has already expired, or have their `claimRefund` transactions front-run by `launch`. Similarly, a project admin attempting to reclaim `SALE_TOKEN` via `ILOManager.claimRefund` after the deadline can be front-run by a third party calling `ILOManager.launch`.

### Remediation

Enforce in `ILOManager.launch` that the current timestamp is strictly before the project's `refundDeadline`.

```solidity
    /// @inheritdoc IILOManager
    function launch(address uniV3PoolAddress) external override {
        require(
            block.timestamp > _cachedProject[uniV3PoolAddress].launchTime &&
            block.timestamp < _cachedProject[uniV3PoolAddress].refundDeadline,
            "LT"
        );
        (uint160 sqrtPriceX96, , , , , , ) = IUniswapV3Pool(uniV3PoolAddress).slot0();
        require(_cachedProject[uniV3PoolAddress].initialPoolPriceX96 == sqrtPriceX96, "UV3P");
        address[] memory initializedPools = _initializedILOPools[uniV3PoolAddress];
        require(initializedPools.length > 0, "NP");
        for (uint256 i = 0; i < initializedPools.length; i++) {
            IILOPool(initializedPools[i]).launch();
        }

        emit ProjectLaunch(uniV3PoolAddress);
    }
```

## Finding 3: Callback-Driven Reentrancy in `ILOManager.launch` Allows Duplicate Pool Launches and Conflicting Launch/Refund States

### Synopsis

`ILOManager.launch` iterates over child `ILOPool` contracts and invokes `IILOPool(initializedPools[i]).launch()` without a reentrancy guard or prior project-level state transition, while `ILOPool.launch` sets `_launchSucceeded = true` only after external token transfers. Token callbacks can therefore reenter `launch` or trigger `claimRefund` mid-launch, corrupting lifecycle state.

### Technical details

In `src/ILOManager.sol`, `launch(address uniV3PoolAddress)` (lines 187–195) validates the launch timestamp and current Uniswap V3 pool price, then loops over `_initializedILOPools[uniV3PoolAddress]` to call `IILOPool(initializedPools[i]).launch()` on line 194:

```solidity
function launch(address uniV3PoolAddress) external override {
    require(block.timestamp > _cachedProject[uniV3PoolAddress].launchTime, "LT");
    (uint160 sqrtPriceX96, , , , , , ) = IUniswapV3Pool(uniV3PoolAddress).slot0();
    require(_cachedProject[uniV3PoolAddress].initialPoolPriceX96 == sqrtPriceX96, "UV3P");
    address[] memory initializedPools = _initializedILOPools[uniV3PoolAddress];
    require(initializedPools.length > 0, "NP");
    for (uint256 i = 0; i < initializedPools.length; i++) {
        IILOPool(initializedPools[i]).launch();
    }

    emit ProjectLaunch(uniV3PoolAddress);
}
```

Neither `ILOManager.launch` nor `ILOPool.launch` enforces a `nonReentrant` lock or updates a lifecycle state flag prior to external interactions (violating the Checks-Effects-Interactions pattern):

1. `ILOManager` does not record a project-level `launched` flag in `_cachedProject[uniV3PoolAddress]` before iterating over `initializedPools`.
2. In `src/ILOPool.sol`, `ILOPool.launch()` checks `require(!_launchSucceeded, "PL")` and `require(!_refundTriggered, "IRF")` at lines 270–272, but defers setting `_launchSucceeded = true` until line 334—after executing external token transfers inside `addLiquidity` (lines 296–303, via `uniswapV3MintCallback` -> `pay`) and `_refundProject(_project.admin)` (line 332, via `TransferHelper.safeTransfer(SALE_TOKEN, projectAdmin, refundAmount)`).

If `SALE_TOKEN` or `RAISE_TOKEN` implements transfer callbacks (such as ERC777 `tokensToSend` or `tokensReceived` hooks), an external callback is triggered while `_launchSucceeded` is still `false` on the currently executing `ILOPool` (and on any remaining unlaunched pools in `initializedPools`):

- **Reentrant `ILOManager.launch` execution:** During `_refundProject(_project.admin)` at line 332, the Uniswap V3 pool's `mint` lock (`slot0.unlocked`) has already been released and `slot0.sqrtPriceX96` remains equal to `initialPoolPriceX96` (as `mint` does not alter `sqrtPriceX96`). A reentrant call to `ILOManager.launch(uniV3PoolAddress)` passes all prerequisites in `ILOManager.launch` and `ILOPool.launch` because `_launchSucceeded` is still `false`, minting duplicate project vesting NFTs (lines 311–329) and deploying liquidity a second time if token balances permit.
- **Mid-launch `claimRefund` state corruption:** Because `_launchSucceeded` is still `false` during the token transfer callback, a call to `ILOPool.claimRefund(tokenId)` passes `require(!_launchSucceeded, "PL")` in the `refundable` modifier (line 340) and sets `_refundTriggered = true` (line 344). When the outer `ILOPool.launch()` call resumes, it sets `_launchSucceeded = true` (line 334) without re-checking `_refundTriggered`. This leaves the pool with both `_refundTriggered == true` and `_launchSucceeded == true`, which permanently bypasses the `if (!_refundTriggered)` guard in `refundable()` (line 338) and allows further refunds even after the pool has launched.

### Assumptions and preconditions

1. A project and at least one `ILOPool` have been initialized, and the pool has raised at least `saleInfo.softCap`.
2. Either `SALE_TOKEN` or `RAISE_TOKEN` supports transfer callbacks (e.g., an ERC777-compliant token), allowing execution control to pass to an external contract during `addLiquidity` or `_refundProject`.
3. `block.timestamp > _cachedProject[uniV3PoolAddress].launchTime` and the Uniswap V3 pool's `sqrtPriceX96` equals `initialPoolPriceX96`.

### Proof of concept

1. A project is created with an ERC777 `SALE_TOKEN` (or `RAISE_TOKEN`), and an `ILOPool` is initialized and reaches its `softCap`.
2. Once `block.timestamp > launchTime`, `ILOManager.launch(uniV3PoolAddress)` is called.
3. `ILOManager.launch` calls `IILOPool(initializedPools[0]).launch()`.
4. `ILOPool.launch` verifies `!_launchSucceeded` and `!_refundTriggered`, adds liquidity via `addLiquidity`, mints project vesting NFTs, and calls `_refundProject(_project.admin)` at line 332 BEFORE setting `_launchSucceeded = true` at line 334.
5. During the `SALE_TOKEN` transfer in `_refundProject` (or `pay` in `uniswapV3MintCallback`), the ERC777 transfer hook invokes the callback on the sender/recipient:
   - **Scenario A (Duplicate Launch):** The callback reenters `ILOManager.launch(uniV3PoolAddress)`. Because `_launchSucceeded` is still `false` and `slot0.sqrtPriceX96 == initialPoolPriceX96`, `ILOPool.launch()` executes a second time, minting duplicate vesting positions for all project recipients in `_vestingConfigs`.
   - **Scenario B (Conflicting Launch and Refund State):** When `block.timestamp >= refundDeadline`, an investor's callback invokes `ILOPool.claimRefund(tokenId)`. Because `_launchSucceeded` is still `false`, `refundable()` succeeds, sets `_refundTriggered = true`, and refunds `RAISE_TOKEN`. The outer `launch()` then completes and sets `_launchSucceeded = true`. Both `_refundTriggered` and `_launchSucceeded` are now `true`, allowing remaining investors to call `claimRefund` on a launched pool because `if (!_refundTriggered)` in `refundable()` evaluates to `false`.

### Impact assessment

**Severity: High**

- **Direct Technical Effect:** Callback-driven reentrancy during `ILOManager.launch` violates the mutual exclusion invariant between `_launchSucceeded` and `_refundTriggered`, and allows `ILOPool.launch()` to be entered multiple times for the same pool.
- **Business Implications:** Reentering `ILOManager.launch` mints duplicate vesting NFTs (`_positionVests`) to project recipients, inflating total vested liquidity shares beyond the liquidity actually minted in Uniswap V3 and causing subsequent `claim` operations to fail or drain other users' shares. Triggering `claimRefund` during the launch callback allows investors to withdraw their `RAISE_TOKEN` while the pool still transitions to `_launchSucceeded = true`.

### Remediation

Apply a `nonReentrant` guard or record the project launch state in `ILOManager.launch` before invoking external child pool calls, and ensure `_launchSucceeded = true` is set before external calls in `ILOPool.launch` (following the Checks-Effects-Interactions pattern).

In `src/ILOManager.sol`:

```solidity
    /// @inheritdoc IILOManager
    function launch(address uniV3PoolAddress) external override nonReentrant {
        require(block.timestamp > _cachedProject[uniV3PoolAddress].launchTime, "LT");
        (uint160 sqrtPriceX96, , , , , , ) = IUniswapV3Pool(uniV3PoolAddress).slot0();
        require(_cachedProject[uniV3PoolAddress].initialPoolPriceX96 == sqrtPriceX96, "UV3P");
        address[] memory initializedPools = _initializedILOPools[uniV3PoolAddress];
        require(initializedPools.length > 0, "NP");
        for (uint256 i = 0; i < initializedPools.length; i++) {
            IILOPool(initializedPools[i]).launch();
        }

        emit ProjectLaunch(uniV3PoolAddress);
    }
```

## Finding 4: Price manipulation or single-pool failure in `ILOManager.launch` permanently blocks project launch across all ILO pools

### Synopsis

`ILOManager.launch` enforces a strict equality check on the Uniswap V3 pool's `sqrtPriceX96` and atomically launches all initialized `ILOPool` instances in a loop without error handling or time bounds. An attacker can manipulate the pool price—or a single under-subscribed `ILOPool` can revert—blocking the entire project from launching.

### Technical details

In `src/ILOManager.sol`, the `launch(address uniV3PoolAddress)` function (lines 187–198) is responsible for launching all `ILOPool` contracts associated with a project once `launchTime` has passed:

```solidity
function launch(address uniV3PoolAddress) external override {
    require(block.timestamp > _cachedProject[uniV3PoolAddress].launchTime, "LT");
    (uint160 sqrtPriceX96, , , , , , ) = IUniswapV3Pool(uniV3PoolAddress).slot0();
    require(_cachedProject[uniV3PoolAddress].initialPoolPriceX96 == sqrtPriceX96, "UV3P");
    address[] memory initializedPools = _initializedILOPools[uniV3PoolAddress];
    require(initializedPools.length > 0, "NP");
    for (uint256 i = 0; i < initializedPools.length; i++) {
        IILOPool(initializedPools[i]).launch();
    }

    emit ProjectLaunch(uniV3PoolAddress);
}
```

This implementation introduces critical denial-of-service vectors:

1. **Strict Spot Price Equality Check (`UV3P`)**: When `initProject` is called, the Uniswap V3 pool is created and initialized at `initialPoolPriceX96` (line 61). Because the Uniswap V3 pool is public and unlocked immediately upon initialization, any external actor can mint a tiny amount of liquidity and execute a swap in `uniV3PoolAddress` before `ILOManager.launch` is called (or front-run the `launch` transaction). This changes `slot0().sqrtPriceX96` away from `_cachedProject[uniV3PoolAddress].initialPoolPriceX96`. Furthermore, if an attacker mints single-sided liquidity or swaps across ticks, restoring the exact `uint160` `sqrtPriceX96` value can be made economically prohibitive or mathematically impossible due to integer rounding in `SwapMath.computeSwapStep`, causing line 190 (`require(_cachedProject[uniV3PoolAddress].initialPoolPriceX96 == sqrtPriceX96, "UV3P")`) to permanently revert.
2. **Coupled Component Readiness and Atomic Batch Revert Propagation**: In the loop at lines 193–195, `ILOManager.launch` iterates over every pool in `_initializedILOPools[uniV3PoolAddress]` and calls `IILOPool(initializedPools[i]).launch()`. Inside `ILOPool.launch()` (line 274 of `src/ILOPool.sol`), each pool requires `totalRaised >= saleInfo.softCap` (`"SC"`), and also requires that `block.timestamp < _project.refundDeadline` has not allowed a user to trigger a refund (`!_refundTriggered`). If a project has multiple `ILOPool` instances (e.g., for different whitelisted tiers or tick ranges) and even a single pool fails to reach its `softCap` (or is front-run with `claimRefund` at `refundDeadline`), `IILOPool(initializedPools[i]).launch()` reverts. Because all pools are coupled in an atomic batch without a mechanism to skip failed pools or remove unready pools, a failure in one pool prevents all other successful pools in the project from launching.
3. **Lack of Refund Deadline Check in `launch`**: `ILOManager.launch` does not check `block.timestamp < _cachedProject[uniV3PoolAddress].refundDeadline`. After `refundDeadline` passes, if `claimRefund` has not yet been called on an `ILOPool`, anyone can still call `ILOManager.launch()` even though the project is already in the refund period. Conversely, if a single user calls `claimRefund` on one `ILOPool` after `refundDeadline`, `_refundTriggered` becomes `true` on that pool, causing `ILOManager.launch()` to revert for the entire batch.

### Assumptions and preconditions

1. A project is initialized via `ILOManager.initProject` and one or more ILO pools are created via `ILOManager.initILOPool`.
2. For the price manipulation vector: An attacker has a minimal amount of `raiseToken` or `saleToken` (or whichever token is required to mint/swap in the initialized Uniswap V3 pool) to interact with the Uniswap V3 pool prior to `ILOManager.launch`.
3. For the coupled pool failure vector: The project admin initializes multiple `ILOPool` contracts for the project, and at least one pool fails to meet its `softCap` while others succeed.

### Proof of concept

#### Scenario A: Spot Price Manipulation DoS (`UV3P`)
1. The project admin calls `initProject` with `initialPoolPriceX96` and `launchTime = T`. The Uniswap V3 pool is deployed and initialized at `sqrtPriceX96 = initialPoolPriceX96`.
2. Investors participate in the ILO by calling `ILOPool.buy(...)` between `start` and `end`.
3. Before `ILOManager.launch(uniV3PoolAddress)` is executed at `block.timestamp > T`, an attacker calls `IUniswapV3Pool(uniV3PoolAddress).mint(...)` with a minimal liquidity amount and then calls `IUniswapV3Pool(uniV3PoolAddress).swap(...)` with a few wei of token0 or token1.
4. The pool's `slot0.sqrtPriceX96` changes from `initialPoolPriceX96` to `initialPoolPriceX96 +/- delta`.
5. When `ILOManager.launch(uniV3PoolAddress)` is called, line 190 checks `_cachedProject[uniV3PoolAddress].initialPoolPriceX96 == sqrtPriceX96`, which evaluates to `false` and reverts with `"UV3P"`.
6. Even if the project admin attempts to swap back to `initialPoolPriceX96`, the attacker can front-run the `launch` transaction in the mempool with another 1-wei swap (or mint high liquidity around the target price so shifting the price back requires massive capital), permanently blocking the launch until `refundDeadline` expires.

#### Scenario B: Single-Component Revert Propagation Across Coupled ILO Pools (`SC`)
1. The project admin initializes `ILOPool_1` (e.g., `softCap = 80,000 USDC`) and `ILOPool_2` (e.g., `softCap = 20,000 USDC`).
2. `ILOPool_1` raises `100,000 USDC` (meeting its soft cap), while `ILOPool_2` only raises `15,000 USDC` (failing to meet its soft cap).
3. After `launchTime`, `ILOManager.launch(uniV3PoolAddress)` is called.
4. In the loop at lines 193–195, `ILOPool_1.launch()` succeeds, but `ILOPool_2.launch()` reverts at `require(totalRaised >= saleInfo.softCap, "SC")`.
5. The entire `ILOManager.launch` transaction reverts, preventing `ILOPool_1` from launching and forcing the entire project into a refund state after `refundDeadline`.

### Impact assessment

- **Severity**: High / Medium
- **Technical Impact**: `ILOManager.launch` permanently or repeatedly reverts, causing a complete denial of service of the project launch mechanism.
- **Business Impact**: Projects cannot deploy their raised liquidity to Uniswap V3 or distribute vested position NFTs. Investors and project admins are forced to wait until `refundDeadline` (7 days after `launchTime`) to reclaim their funds, wasting gas and halting the token launch.

### Remediation

To fix the coupled batch revert and spot price DoS:
1. Decouple individual `ILOPool` launches by using a `try/catch` block or allowing `ILOManager.launch` to skip pools that did not meet their `softCap` (allowing those specific pools to be refunded while successful pools launch), and enforce that `block.timestamp < _cachedProject[uniV3PoolAddress].refundDeadline`.
2. Ensure the Uniswap V3 pool price is either restored/reconciled during `launch` or only initialized inside `ILOManager.launch` rather than `initProject`, so external actors cannot alter `sqrtPriceX96` before the ILO launches.

```solidity
    /// @inheritdoc IILOManager
    function launch(address uniV3PoolAddress) external override {
        Project storage _project = _cachedProject[uniV3PoolAddress];
        require(block.timestamp > _project.launchTime, "LT");
        require(block.timestamp < _project.refundDeadline, "RFT");
        (uint160 sqrtPriceX96, , , , , , ) = IUniswapV3Pool(uniV3PoolAddress).slot0();
        require(_project.initialPoolPriceX96 == sqrtPriceX96, "UV3P");
        address[] memory initializedPools = _initializedILOPools[uniV3PoolAddress];
        require(initializedPools.length > 0, "NP");
        uint256 launchedCount;
        for (uint256 i = 0; i < initializedPools.length; i++) {
            try IILOPool(initializedPools[i]).launch() {
                launchedCount++;
            } catch {}
        }
        require(launchedCount > 0, "NP");

        emit ProjectLaunch(uniV3PoolAddress);
    }
```

## Finding 5: Incremental Liquidity Calculation in `ILOPool.buy` Causes Rounding Loss and Permanently Locked Pool Liquidity

### Synopsis

`ILOPool.buy` computes investor liquidity incrementally from each call's `raiseAmount` while `ILOPool.launch` mints Uniswap V3 liquidity from aggregate `totalRaised`. Repeated floor rounding across multiple `buy` calls under-allocates investor `_position.liquidity` relative to total minted liquidity, permanently locking the unallocated liquidity inside `ILOPool`.

### Technical details

In `ILOPool.buy` (`src/ILOPool.sol`), each contribution updates two cumulative ledgers:
- The global pool ledger at line 137: `totalRaised += raiseAmount;`
- The per-position ledger at line 152: `_position.raiseAmount += raiseAmount;`

When an investor already holds a position NFT (`balanceOf(recipient) != 0`, lines 143–148), `buy` retrieves the existing `tokenId` and accumulates `_position.raiseAmount`. However, at lines 155–167, `buy` calculates `liquidityDelta` solely from the incremental `raiseAmount` of the current transaction rather than from the updated cumulative `_position.raiseAmount`:

1. `LiquidityAmounts.getLiquidityForAmount0` (line 156) or `LiquidityAmounts.getLiquidityForAmount1` (line 158) computes the raw liquidity for `raiseAmount` using `FullMath.mulDiv`, which truncates (rounds down).
2. Line 164 scales that result by the investor share configuration:
   `liquidityDelta = uint128(FullMath.mulDiv(liquidityDelta, _vestingConfigs[0].shares, BPS));`
   This performs a second floor division before adding `liquidityDelta` to `_position.liquidity` (line 167) and `_positionVests[tokenId].totalLiquidity` (line 170).

Because floor division is superadditive ($\lfloor a/d \rfloor + \lfloor b/d \rfloor \le \lfloor (a+b)/d \rfloor$), splitting a contribution of size $R = \sum_i r_i$ across multiple `buy` calls for the same position (or across multiple positions) yields strictly less liquidity than computing liquidity from the aggregate contribution:
$$\sum_i \left\lfloor \frac{\text{getLiquidity}(r_i) \cdot \text{shares}_0}{\text{BPS}} \right\rfloor \le \left\lfloor \frac{\text{getLiquidity}\left(\sum_i r_i\right) \cdot \text{shares}_0}{\text{BPS}} \right\rfloor$$

When `ILOPool.launch` executes (lines 285–329):
- `_saleAmountNeeded(totalRaised)` computes the aggregate `liquidity` for the entire `totalRaised` (lines 288, 290) and mints that exact `liquidity` into the Uniswap V3 pool (line 296).
- Project vesting positions (`_vestingConfigs[1..]`) receive `FullMath.mulDiv(liquidity, projectConfig.shares, BPS)` based on the aggregate `liquidity` (line 316).
- Investor positions only hold the sum of their incrementally rounded `liquidityDelta` values.

Consequently, an investor who purchases in multiple instalments receives less position liquidity than an investor who contributes the exact same total `_position.raiseAmount` in a single transaction, and the difference between the aggregate investor share of `liquidity` minted at `launch()` and the sum of investor `_position.liquidity` values is never assigned to any NFT position. Because `claim()` only burns up to each token's `_claimableLiquidity(tokenId)` (bounded by `_positionVests[tokenId].totalLiquidity`), the residual liquidity remains permanently locked in the Uniswap V3 position held by `ILOPool`.

### Assumptions and preconditions

1. An ILO pool is initialized with valid tick bounds (`TICK_LOWER < TICK_UPPER`), price `SQRT_RATIO_X96`, and `_vestingConfigs[0].shares < BPS`.
2. One or more whitelisted investors (or any users when `_openToAll` is enabled) execute multiple `buy` calls during the active sale window (`saleInfo.start < block.timestamp < saleInfo.end`).
3. `(raiseAmount * intermediate) % (sqrtRatioBX96 - sqrtRatioAX96) != 0` (when `RAISE_TOKEN == token0`), `(raiseAmount * Q96) % (sqrtRatioBX96 - sqrtRatioAX96) != 0` (when `RAISE_TOKEN == token1`), or `(rawLiquidity * _vestingConfigs[0].shares) % BPS != 0` across the individual `buy` calls.

### Proof of concept

Consider an initialized `ILOPool` where:
- `RAISE_TOKEN == _cachedPoolKey.token1`
- `SQRT_RATIO_X96 - SQRT_RATIO_LOWER_X96 = FixedPoint96.Q96` (so `getLiquidityForAmount1(SQRT_RATIO_LOWER_X96, SQRT_RATIO_X96, amount1) == amount1`)
- `_vestingConfigs[0].shares = 2000` (20%, i.e., `2000 / 10000`) and `_vestingConfigs[1].shares = 8000` (80%)

1. Investor A calls `buy(9, investorA)` twice (for a cumulative `_position.raiseAmount` of `18`):
   - First call (`raiseAmount = 9`):
     - `rawLiquidity = 9`
     - `liquidityDelta = mulDiv(9, 2000, 10000) = 1` (since `1 > 0`, `require(liquidityDelta > 0, "ZA")` passes)
     - `_position.raiseAmount = 9`, `_position.liquidity = 1`
   - Second call (`raiseAmount = 9`):
     - `rawLiquidity = 9`
     - `liquidityDelta = mulDiv(9, 2000, 10000) = 1`
     - `_position.raiseAmount = 18`, `_position.liquidity = 2`, `_positionVests[tokenId].totalLiquidity = 2`
2. Had Investor A contributed `18` in a single call (or had `_position.liquidity` been derived from cumulative `_position.raiseAmount = 18`):
   - `rawLiquidity = 18`
   - `_position.liquidity = mulDiv(18, 2000, 10000) = 3`
3. When `launch()` is called with `totalRaised = 18` (assuming `softCap <= 18` or alongside other contributions):
   - `_saleAmountNeeded(18)` computes total Uniswap V3 liquidity `liquidity = 18` and mints `18` liquidity to the Uniswap V3 pool.
   - The project vesting position receives `mulDiv(18, 8000, 10000) = 14` liquidity.
   - Investor A's position only has `2` claimable liquidity instead of `3`.
   - After all vesting schedules complete and all positions call `claim()`, `18 - (14 + 2) = 2` units of liquidity (and their underlying `SALE_TOKEN` and `RAISE_TOKEN` principal plus accrued fees) remain permanently stuck in the Uniswap V3 pool under `ILOPool`'s position.

Across many `buy` calls or when `SQRT_RATIO_UPPER_X96 - SQRT_RATIO_X96` is large relative to `SQRT_RATIO_X96`, the per-call truncation in `getLiquidityForAmount0/1` and `mulDiv(..., shares, BPS)` compounds across every incremental purchase.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** Investors who add to their position across multiple `buy` transactions receive less LP liquidity than warranted by their cumulative `_position.raiseAmount`. Furthermore, the aggregate liquidity minted into the Uniswap V3 pool during `launch()` exceeds the sum of all NFT positions' `totalLiquidity`, leaving the unallocated liquidity (and the corresponding `RAISE_TOKEN` and `SALE_TOKEN` principal) permanently locked in `ILOPool` with no withdrawal path.

### Remediation

Compute the position's target liquidity from its cumulative `_position.raiseAmount` and set `liquidityDelta` to the difference between the new target position liquidity and the existing `_position.liquidity`. Additionally, in `launch()`, either mint only the sum of allocated position liquidities or allocate any residual rounding dust to one of the positions.

In `ILOPool.buy` (`src/ILOPool.sol`):

```solidity
        Position storage _position = _positions[tokenId];
        require(raiseAmount <= saleInfo.maxCapPerUser - _position.raiseAmount, "UC");
        _position.raiseAmount += raiseAmount;

        // get total amount of liquidity associated with cumulative position raise amount
        uint128 totalPositionLiquidity;
        if (RAISE_TOKEN == _cachedPoolKey.token0) {
            totalPositionLiquidity = LiquidityAmounts.getLiquidityForAmount0(SQRT_RATIO_X96, SQRT_RATIO_UPPER_X96, _position.raiseAmount);
        } else {
            totalPositionLiquidity = LiquidityAmounts.getLiquidityForAmount1(SQRT_RATIO_LOWER_X96, SQRT_RATIO_X96, _position.raiseAmount);
        }

        // calculate total share liquidity investor receives by INVESTOR_SHARES config
        totalPositionLiquidity = uint128(FullMath.mulDiv(totalPositionLiquidity, _vestingConfigs[0].shares, BPS));
        liquidityDelta = totalPositionLiquidity - _position.liquidity;
        require(liquidityDelta > 0, "ZA");

        // update investor's liquidity
        _position.liquidity = totalPositionLiquidity;

        // update total liquidity locked for vest
        _positionVests[tokenId].totalLiquidity = totalPositionLiquidity;
```

## Finding 6: Bypass of `maxCapPerUser` in `ILOPool.buy` via ERC721 position transfer allows a whitelisted user to purchase up to the entire `hardCap`

### Synopsis

`ILOPool.buy` enforces `saleInfo.maxCapPerUser` against the `raiseAmount` recorded on a single NFT position (`_positions[tokenId].raiseAmount`) rather than tracking cumulative contributions per user address. Because `ILOPool` position NFTs are freely transferable ERC721 tokens, a whitelisted user can transfer their NFT to another address and repeatedly call `buy` to bypass `maxCapPerUser`.

### Technical details

In `ILOPool.buy`, access is restricted to whitelisted `recipient` addresses via `require(_isWhitelisted(recipient), "UA")` at line 133, and each user's total contribution is intended to be capped at `saleInfo.maxCapPerUser`.

However, `ILOPool.buy` determines whether `recipient` already has a position and checks the user cap as follows (lines 143–152):

```solidity
if (balanceOf(recipient) == 0) {
    _mint(recipient, (tokenId = _nextId++));
    _positionVests[tokenId].schedule = _vestingConfigs[0].schedule;
} else {
    tokenId = tokenOfOwnerByIndex(recipient, 0);
}

Position storage _position = _positions[tokenId];
require(raiseAmount <= saleInfo.maxCapPerUser - _position.raiseAmount, "UC");
_position.raiseAmount += raiseAmount;
```

This logic relies on the assumption that a `recipient` can never obtain more than one NFT position and that `_positions[tokenId].raiseAmount` equals the cumulative amount raised by `recipient`. Because `ILOPool` inherits from OpenZeppelin's `ERC721` without restricting token transfers, this assumption does not hold:

1. A whitelisted user `A` calls `buy(saleInfo.maxCapPerUser, A)`, which mints `tokenId = 1` to `A` and sets `_positions[1].raiseAmount = saleInfo.maxCapPerUser`.
2. User `A` transfers `tokenId = 1` to another address `B` (which does not need to be whitelisted). Now `balanceOf(A)` is `0`.
3. User `A` calls `buy(saleInfo.maxCapPerUser, A)` again. Because `balanceOf(A) == 0`, a new NFT (`tokenId = 2`) is minted to `A` with `_positions[2].raiseAmount == 0`, allowing `A` to contribute another `saleInfo.maxCapPerUser`.
4. Alternatively, if an investor receives another position NFT prior to buying so that `tokenOfOwnerByIndex(recipient, 0)` resolves to a different token ID with a smaller `raiseAmount`, subsequent purchases are credited against that token's `raiseAmount` rather than the investor's total cumulative contribution.

### Assumptions and preconditions

1. An `ILOPool` is initialized with `saleInfo.maxCapPerUser < saleInfo.hardCap`.
2. The sale is active (`block.timestamp > saleInfo.start && block.timestamp < saleInfo.end`).
3. The caller (or `recipient`) is whitelisted in the `ILOPool` (or `_openToAll` is enabled) and holds sufficient `RAISE_TOKEN` to exceed `saleInfo.maxCapPerUser`.

### Proof of concept

1. Suppose `saleInfo.hardCap = 100,000e18` and `saleInfo.maxCapPerUser = 60,000e18`, and `INVESTOR` is whitelisted.
2. During the sale window, `INVESTOR` calls `ILOPool.buy(60,000e18, INVESTOR)`.
   - `balanceOf(INVESTOR)` is `0`, so `tokenId = 1` is minted to `INVESTOR`.
   - `_positions[1].raiseAmount` becomes `60,000e18`.
3. `INVESTOR` calls `ILOPool.transferFrom(INVESTOR, secondaryAddress, 1)` to move `tokenId = 1` to another address controlled by `INVESTOR`.
   - `balanceOf(INVESTOR)` is now `0`.
4. `INVESTOR` calls `ILOPool.buy(40,000e18, INVESTOR)`.
   - Because `balanceOf(INVESTOR) == 0`, `ILOPool.buy` mints `tokenId = 2` to `INVESTOR`.
   - `_positions[2].raiseAmount` starts at `0`, so `require(40,000e18 <= 60,000e18 - 0, "UC")` succeeds.
5. `INVESTOR` has contributed `100,000e18` (the entire `hardCap`), bypassing the `60,000e18` `maxCapPerUser` limit and preventing any other whitelisted users from participating in the sale.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** The `saleInfo.maxCapPerUser` limit and whitelist allocation controls are completely ineffective. A single whitelisted account can mint arbitrary numbers of position NFTs and purchase up to the pool's entire `saleInfo.hardCap`.
- **Business Impact:** A single whale or bot with one whitelisted address can monopolize the entire ILO allocation, starving all other whitelisted participants and undermining fair token distribution.

### Remediation

Track cumulative raised amounts per user address in a dedicated storage mapping and enforce `saleInfo.maxCapPerUser` against that per-user total, or prohibit transfers of investor position NFTs before the pool launches.

```solidity
mapping(address => uint256) public userRaised;
```

In `ILOPool.buy`:

```solidity
        totalRaised += raiseAmount;

        require(totalSold() <= saleInfo.maxSaleAmount, "SA");

        require(raiseAmount <= saleInfo.maxCapPerUser - userRaised[recipient], "UC");
        userRaised[recipient] += raiseAmount;

        // if investor already have a position, just increase raise amount and liquidity
        // otherwise, mint new nft for investor and assign vesting schedules
        if (balanceOf(recipient) == 0) {
            _mint(recipient, (tokenId = _nextId++));
            _positionVests[tokenId].schedule = _vestingConfigs[0].schedule;
        } else {
            tokenId = tokenOfOwnerByIndex(recipient, 0);
        }

        Position storage _position = _positions[tokenId];
        _position.raiseAmount += raiseAmount;
```

## Finding 7: Lack of Balance-Delta Verification in `ILOPool.buy` Causes Accounting Insolvency and Blocks Pool Launch for Fee-on-Transfer `RAISE_TOKEN`

### Synopsis

`ILOPool.buy` transfers `RAISE_TOKEN` via `TransferHelper.safeTransferFrom` without measuring the actual token balance received by the contract. When `RAISE_TOKEN` charges a transfer fee, credited liquidity and raised amounts exceed actual holdings, inflating investor positions and causing `launch()` and `claimRefund()` to revert due to insufficient balance.

### Technical details

In `ILOPool.buy` (`src/ILOPool.sol`), the contract accepts a nominal `raiseAmount` parameter and uses it directly to compute the investor's liquidity and update pool accounting before executing the token transfer:

1. At lines 137 and 152, `totalRaised` and `_position.raiseAmount` are incremented by the nominal `raiseAmount`.
2. At lines 155–164, `liquidityDelta` is calculated directly from the nominal `raiseAmount` via `LiquidityAmounts.getLiquidityForAmount0` (line 156) or `LiquidityAmounts.getLiquidityForAmount1` (line 158) and scaled by `_vestingConfigs[0].shares / BPS` (line 164).
3. At lines 167–170, `_position.liquidity` and `_positionVests[tokenId].totalLiquidity` are increased by `liquidityDelta`.
4. At line 173, `TransferHelper.safeTransferFrom(RAISE_TOKEN, msg.sender, address(this), raiseAmount)` pulls `raiseAmount` from `msg.sender` into `address(this)`.

Because `ILOPool.buy` does not check the contract's `RAISE_TOKEN` balance before and after `TransferHelper.safeTransferFrom` at line 173, any fee-on-transfer or deflationary behavior on `RAISE_TOKEN` results in `ILOPool` receiving strictly less than `raiseAmount`.

This discrepancy between nominal contribution and actual tokens received breaks two core protocol invariants:
- **Overstated Liquidity & Raise Accounting:** `_position.liquidity`, `_positionVests[tokenId].totalLiquidity`, `_position.raiseAmount`, and `totalRaised` reflect the pre-fee `raiseAmount` rather than the actual tokens deposited into `ILOPool`.
- **Insolvency During Settlement (`launch` and `claimRefund`):** In `ILOPool.launch()` (lines 285–303), the contract computes required liquidity from `totalRaised`, sets `amount0 = totalRaised; amount0Min = totalRaised` (or `amount1 = totalRaised; amount1Min = totalRaised`), and calls `addLiquidity`. When the Uniswap V3 pool invokes `uniswapV3MintCallback`, `ILOPool` attempts to pay `amount0Owed` or `amount1Owed` (equal to `totalRaised`), which reverts because the contract's actual `RAISE_TOKEN` balance is only `totalRaised - totalTransferFees`. Likewise, if the sale fails and investors call `claimRefund()`, the last investors to claim will face a revert because the contract holds less `RAISE_TOKEN` than the sum of `_position.raiseAmount`.

### Assumptions and preconditions

1. An ILO pool is initialized with a `RAISE_TOKEN` that deducts a transfer fee on `transferFrom` (or exhibits deflationary transfer mechanics).
2. One or more whitelisted investors (or any user when `_openToAll` is `true`) participate in the sale by calling `ILOPool.buy(raiseAmount, recipient)` during the active sale window (`saleInfo.start < block.timestamp < saleInfo.end`).

### Proof of concept

1. Suppose an `ILOPool` is initialized with a `RAISE_TOKEN` that charges a 1% fee on transfers, `softCap = 80,000e18`, and `hardCap = 100,000e18`.
2. During the sale period, investors call `buy(raiseAmount, recipient)` for a cumulative nominal `raiseAmount` of `90,000e18` (satisfying `totalRaised >= saleInfo.softCap`).
3. For each `buy` call, `ILOPool` records the full `raiseAmount` in `totalRaised`, `_position.raiseAmount`, and computes `liquidityDelta` (lines 156–170) based on the full `raiseAmount`.
4. At line 173, `TransferHelper.safeTransferFrom(RAISE_TOKEN, msg.sender, address(this), raiseAmount)` transfers `raiseAmount` from `msg.sender`, but due to the 1% fee, `ILOPool` only receives `89,100e18` of `RAISE_TOKEN` in total while `totalRaised` is recorded as `90,000e18`.
5. After `saleInfo.end`, `ILOManager` calls `ILOPool.launch()`.
6. `launch()` sets the desired and minimum `RAISE_TOKEN` amount to `totalRaised` (`90,000e18`) and calls `pool.mint(...)` with liquidity corresponding to `90,000e18`.
7. Inside `uniswapV3MintCallback`, `ILOPool` attempts to transfer `90,000e18` of `RAISE_TOKEN` to the Uniswap V3 pool, which reverts because `IERC20(RAISE_TOKEN).balanceOf(address(this))` is only `89,100e18`.
8. Because `launch()` permanently reverts, the ILO fails and transitions to the refund period. When investors call `claimRefund()`, each early claimant withdraws their full pre-fee `_position.raiseAmount`, exhausting the contract's `89,100e18` balance and leaving the remaining `900e18` of claims permanently unrefundable.

### Impact assessment

- **Severity:** Medium / High.
- **Direct Technical Effects:** `totalRaised`, `_position.raiseAmount`, and `_position.liquidity` are overstated relative to the actual `RAISE_TOKEN` balance held by `ILOPool`.
- **Business Implications:** The ILO pool cannot be launched (`launch()` always reverts due to insufficient `RAISE_TOKEN` balance during the Uniswap V3 mint callback), and when users subsequently attempt to recover their funds via `claimRefund()`, the last users to claim cannot withdraw their funds because early claimants withdraw the unadjusted nominal `raiseAmount` until the contract balance is depleted.

### Remediation

Measure the actual amount of `RAISE_TOKEN` received by comparing the contract's token balance before and after `TransferHelper.safeTransferFrom` (or reject fee-on-transfer tokens by requiring the balance delta to equal `raiseAmount`), and use the actual received amount for all liquidity and raise accounting calculations.

```solidity
        // transfer fund into contract and measure actual received amount
        uint256 balanceBefore = IERC20(RAISE_TOKEN).balanceOf(address(this));
        TransferHelper.safeTransferFrom(RAISE_TOKEN, msg.sender, address(this), raiseAmount);
        uint256 actualRaiseAmount = IERC20(RAISE_TOKEN).balanceOf(address(this)) - balanceBefore;
        require(actualRaiseAmount == raiseAmount, "FOT");

        // get amount of liquidity associated with raise amount
        if (RAISE_TOKEN == _cachedPoolKey.token0) {
            liquidityDelta = LiquidityAmounts.getLiquidityForAmount0(SQRT_RATIO_X96, SQRT_RATIO_UPPER_X96, actualRaiseAmount);
        } else {
            liquidityDelta = LiquidityAmounts.getLiquidityForAmount1(SQRT_RATIO_LOWER_X96, SQRT_RATIO_X96, actualRaiseAmount);
        }

        require(liquidityDelta > 0, "ZA");

        // calculate amount of share liquidity investor recieve by INVESTOR_SHARES config
        liquidityDelta = uint128(FullMath.mulDiv(liquidityDelta, _vestingConfigs[0].shares, BPS));
        
        // increase investor's liquidity
        _position.liquidity += liquidityDelta;

        // update total liquidity locked for vest and assiging vesing schedules
        _positionVests[tokenId].totalLiquidity = _position.liquidity;
```

## Finding 8: Misplaced Zero-Liquidity Validation in `ILOPool.buy` Allows Purchases That Round Down to Zero Investor Liquidity

### Synopsis

In `ILOPool.buy`, the `require(liquidityDelta > 0, "ZA")` check is executed before scaling `liquidityDelta` by the investor share ratio (`_vestingConfigs[0].shares / BPS`). Consequently, integer division truncation in `FullMath.mulDiv` can reduce `liquidityDelta` to zero while still collecting the caller's `raiseAmount`.

### Technical details

When an investor participates in an ILO sale via `ILOPool.buy(uint256 raiseAmount, address recipient)`, the contract computes the position liquidity to credit to `recipient` in two steps:

1. On lines 155–159 of `src/ILOPool.sol`, the total liquidity corresponding to `raiseAmount` across the configured tick range is calculated via `LiquidityAmounts.getLiquidityForAmount0` or `LiquidityAmounts.getLiquidityForAmount1`:
   ```solidity
   if (RAISE_TOKEN == _cachedPoolKey.token0) {
       liquidityDelta = LiquidityAmounts.getLiquidityForAmount0(SQRT_RATIO_X96, SQRT_RATIO_UPPER_X96, raiseAmount);
   } else {
       liquidityDelta = LiquidityAmounts.getLiquidityForAmount1(SQRT_RATIO_LOWER_X96, SQRT_RATIO_X96, raiseAmount);
   }
   ```
2. On line 161, the contract validates that the computed liquidity is non-zero:
   ```solidity
   require(liquidityDelta > 0, "ZA");
   ```
3. Immediately afterward, on line 164, `liquidityDelta` is scaled down to the fraction of total pool liquidity allocated to investors (`_vestingConfigs[0].shares` out of `BPS = 10000`):
   ```solidity
   liquidityDelta = uint128(FullMath.mulDiv(liquidityDelta, _vestingConfigs[0].shares, BPS));
   ```

Because `FullMath.mulDiv` performs floor division, any initial `liquidityDelta > 0` that satisfies:
$$\text{liquidityDelta} \times \text{\_vestingConfigs[0].shares} < \text{BPS}$$
will pass the `require(liquidityDelta > 0, "ZA")` check on line 161 and then be truncated to `0` on line 164 (furthermore, `_validateSharesAndVests` in `src/base/ILOVest.sol` does not prevent `_vestingConfigs[0].shares` from being small or even `0`).

Execution then proceeds to lines 167–173:
- `_position.liquidity` and `_positionVests[tokenId].totalLiquidity` are increased by `0`.
- If `recipient` had no prior position NFT, a new NFT is minted with `0` liquidity.
- `_position.raiseAmount` and `totalRaised` remain incremented by `raiseAmount`.
- `TransferHelper.safeTransferFrom` transfers `raiseAmount` of `RAISE_TOKEN` from `msg.sender` to the pool.

As a result, if the pool launches successfully, the investor receives zero claimable liquidity for the contributed `raiseAmount`.

### Assumptions and preconditions

1. An `ILOPool` is initialized and currently within the active sale window (`saleInfo.start < block.timestamp < saleInfo.end`).
2. `_vestingConfigs[0].shares` is less than `BPS` (`10000`), which is standard whenever project/team/treasury vesting shares exist (for example, `_vestingConfigs[0].shares = 2000` in `test/Mock.t.sol`).
3. The `recipient` address is whitelisted (or `_openToAll` is `true`).

### Proof of concept

1. Assume an `ILOPool` is initialized with `_vestingConfigs[0].shares = 2000` (20% investor share, `BPS = 10000`), as in the project's test configuration (`test/Mock.t.sol`).
2. Suppose the pool's `SQRT_RATIO_LOWER_X96`, `SQRT_RATIO_X96`, and `SQRT_RATIO_UPPER_X96` are configured such that a small `raiseAmount` yields an unscaled `liquidityDelta` of `4` from `LiquidityAmounts.getLiquidityForAmount0` (or `getLiquidityForAmount1`).
3. A whitelisted user calls `buy(raiseAmount, user)`:
   - At line 156/158, `liquidityDelta` is computed as `4`.
   - At line 161, `require(liquidityDelta > 0, "ZA")` evaluates `4 > 0` and succeeds.
   - At line 164, `liquidityDelta = uint128(FullMath.mulDiv(4, 2000, 10000))` evaluates `floor(8000 / 10000) = 0`.
   - At lines 167–170, `_position.liquidity` and `_positionVests[tokenId].totalLiquidity` are incremented by `0`.
   - At line 173, `raiseAmount` of `RAISE_TOKEN` is transferred from the caller to the pool contract.
4. When the pool launches via `launch()`, `totalRaised` (including the caller's `raiseAmount`) is deployed into the Uniswap V3 pool, but the user's position has `0` liquidity and cannot claim any tokens.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** The zero-amount guard (`"ZA"`) fails to catch purchases where the raw liquidity is non-zero but the investor's proportional share truncates to zero. `totalRaised` and `_position.raiseAmount` increase while `_position.liquidity` increases by zero.
- **Business Effect:** Investors making small purchases (or purchasing in pools with high tick ranges / low investor share percentages) can pay `RAISE_TOKEN` into the pool while receiving zero LP entitlement upon pool launch. Additionally, a third party can repeatedly call `buy` with dust `raiseAmount` for a whitelisted `recipient` to consume their `maxCapPerUser` allowance and inflate `totalRaised` without granting `recipient` any liquidity.

### Remediation

Perform the `require(liquidityDelta > 0, "ZA")` validation after scaling `liquidityDelta` by `_vestingConfigs[0].shares / BPS`.

```solidity
        // get amount of liquidity associated with raise amount
        if (RAISE_TOKEN == _cachedPoolKey.token0) {
            liquidityDelta = LiquidityAmounts.getLiquidityForAmount0(SQRT_RATIO_X96, SQRT_RATIO_UPPER_X96, raiseAmount);
        } else {
            liquidityDelta = LiquidityAmounts.getLiquidityForAmount1(SQRT_RATIO_LOWER_X96, SQRT_RATIO_X96, raiseAmount);
        }

        // calculate amount of share liquidity investor recieve by INVESTOR_SHARES config
        liquidityDelta = uint128(FullMath.mulDiv(liquidityDelta, _vestingConfigs[0].shares, BPS));

        require(liquidityDelta > 0, "ZA");

        // increase investor's liquidity
        _position.liquidity += liquidityDelta;

        // update total liquidity locked for vest and assiging vesing schedules
        _positionVests[tokenId].totalLiquidity = _position.liquidity;

        // transfer fund into contract
        TransferHelper.safeTransferFrom(RAISE_TOKEN, msg.sender, address(this), raiseAmount);
```

## Finding 9: Rounding Down of Liquidity in `ILOPool.launch` Causes `addLiquidity` Slippage Check to Revert and Permanently Blocks Project Launch

### Synopsis

In `ILOPool.launch()`, `amount0Min` or `amount1Min` is set to the exact `totalRaised` amount while `liquidity` is computed via `_saleAmountNeeded(totalRaised)`, which rounds down. When `UniswapV3Pool.mint()` recomputes token amounts from the truncated `liquidity`, the actual raise token amount owed can be less than `totalRaised`, causing `addLiquidity()` to revert with `'Price slippage check'`.

### Technical details

When a project is launched via `ILOManager.launch()`, it verifies that the Uniswap V3 pool's `slot0().sqrtPriceX96` equals `initialPoolPriceX96` and invokes `ILOPool.launch()` on each initialized ILO pool.

In `ILOPool.launch()` (`src/ILOPool.sol`), the contract calculates the liquidity to mint and sets the minimum token bounds passed to `addLiquidity()`:

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

// actually deploy liquidity to uniswap pool
(amount0, amount1) = addLiquidity(AddLiquidityParams({
    pool: IUniswapV3Pool(uniV3PoolAddress),
    liquidity: liquidity,
    amount0Desired: amount0,
    amount1Desired: amount1,
    amount0Min: amount0Min,
    amount1Min: amount1Min
}));
```

Inside `_saleAmountNeeded(totalRaised)`:
1. **When `RAISE_TOKEN` is `token1` (`_cachedPoolKey.token0 == SALE_TOKEN`):**
   `liquidity` is computed via `LiquidityAmounts.getLiquidityForAmount1(SQRT_RATIO_LOWER_X96, SQRT_RATIO_X96, totalRaised)`:
   $$\text{liquidity} = \left\lfloor \frac{\text{totalRaised} \times 2^{96}}{\text{SQRT\_RATIO\_X96} - \text{SQRT\_RATIO\_LOWER\_X96}} \right\rfloor$$
   Let $\Delta = \text{SQRT\_RATIO\_X96} - \text{SQRT\_RATIO\_LOWER\_X96}$ and $Q_{96} = 2^{96}$. Writing $\text{totalRaised} \times Q_{96} = \text{liquidity} \times \Delta + r$ with $0 \le r < \Delta$, the subsequent call to `UniswapV3Pool.mint()` computes the required `amount1` via `SqrtPriceMath.getAmount1Delta`:
   $$\text{amount1} = \left\lceil \frac{\text{liquidity} \times \Delta}{Q_{96}} \right\rceil = \text{totalRaised} - \left\lfloor \frac{r}{Q_{96}} \right\rfloor$$
   Whenever $\Delta > Q_{96}$ and $r \ge Q_{96}$, $\lfloor r / Q_{96} \rfloor \ge 1$, meaning $\text{amount1} < \text{totalRaised} = \text{amount1Min}$.

2. **When `RAISE_TOKEN` is `token0` (`_cachedPoolKey.token0 == RAISE_TOKEN`):**
   `liquidity` is computed via `LiquidityAmounts.getLiquidityForAmount0(SQRT_RATIO_X96, SQRT_RATIO_UPPER_X96, totalRaised)`, which performs two consecutive floor divisions (`FullMath.mulDiv` for `intermediate` and `FullMath.mulDiv` for `liquidity`). When `UniswapV3Pool.mint()` recomputes `amount0` from this truncated `liquidity` via `SqrtPriceMath.getAmount0Delta`, the resulting `amount0` can likewise be strictly less than `totalRaised` (`amount0 < amount0Min`).

In `LiquidityManagement.addLiquidity()` (`src/base/LiquidityManagement.sol` line 56), the contract enforces:
```solidity
require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
```
Because `amount1 < amount1Min` (or `amount0 < amount0Min`), `addLiquidity()` reverts with `'Price slippage check'` even though the pool price matches `SQRT_RATIO_X96` exactly. Furthermore, because `launch()` can only be executed after `launchTime` (when the sale period `saleInfo.end` has already concluded), `totalRaised` can no longer be modified via `buy()`, permanently preventing the ILO pool and project from launching.

### Assumptions and preconditions

1. An ILO pool is initialized and reaches at least `saleInfo.softCap` in `totalRaised`.
2. Either `RAISE_TOKEN` is `token1` with `SQRT_RATIO_X96 - SQRT_RATIO_LOWER_X96 > 2^96` (as in the protocol's own test configuration where `initialPoolPriceX96 = 2 * 2^96` and `tickLower = -887270`), or `RAISE_TOKEN` is `token0` where `getLiquidityForAmount0` loses precision in `intermediate` and `liquidity`.
3. `totalRaised` settles at a value where the remainder lost when flooring `liquidity` causes `UniswapV3Pool.mint()` to require at least 1 wei less of `RAISE_TOKEN` than `totalRaised` (which occurs naturally with ~50% probability in the test configuration, or intentionally via a dust `buy()` by a whitelisted participant).

### Proof of concept

Consider the project configuration from `test/Mock.t.sol` and `test/IntegrationTestBase.sol`:
- `token0 = SALE_TOKEN`, `token1 = RAISE_TOKEN (USDC)`
- `SQRT_RATIO_X96 = 158456325028528675187087900672` ($2 \times 2^{96}$)
- `tickLower = -887270` $\implies$ `SQRT_RATIO_LOWER_X96 = 4295343490`
- $\Delta = \text{SQRT\_RATIO\_X96} - \text{SQRT\_RATIO\_LOWER\_X96} = 158456325028528675182792557182$

1. Investors participate in the sale such that `totalRaised = 90000 ether + 1` ($90000000000000000000001$).
2. After `launchTime`, `ILOManager.launch()` is called, which calls `ILOPool.launch()`.
3. `_saleAmountNeeded(totalRaised)` computes:
   $$\text{liquidity} = \left\lfloor \frac{90000000000000000000001 \times 2^{96}}{158456325028528675182792557182} \right\rfloor = 45000000000000000001220$$
   The division remainder is $r = 81122098007185595620233099116 > 2^{96}$ ($79228162514264337593543950336$).
4. `ILOPool.launch()` sets `amount1Min = totalRaised = 90000000000000000000001` and calls `addLiquidity()`.
5. `UniswapV3Pool.mint()` computes `amount1` from `liquidity = 45000000000000000001220`:
   $$\text{amount1} = \left\lceil \frac{45000000000000000001220 \times 158456325028528675182792557182}{2^{96}} \right\rceil = 90000000000000000000000$$
6. Inside `addLiquidity()`, `amount1 < params.amount1Min` ($90000000000000000000000 < 90000000000000000000001$), causing the transaction to revert with `'Price slippage check'`.
7. Because the sale period has already ended (`block.timestamp > launchTime > saleInfo.end`), `totalRaised` is immutable, and the project can never be launched.

### Impact assessment

**Severity: High**

The vulnerability causes a permanent Denial of Service of `ILOPool.launch()` (and consequently `ILOManager.launch()` for the entire project, as `ILOManager.launch()` loops over all initialized ILO pools). Because `totalRaised` cannot be altered after `saleInfo.end` and `ILOManager.launch()` requires the Uniswap V3 pool price to match `initialPoolPriceX96`, a pool hitting this rounding condition can never launch, forcing the ILO to fail and wait until `refundDeadline` for refunds. This occurs with high probability during normal operation and can also be triggered deterministically at negligible cost (1 wei of `RAISE_TOKEN`) by any whitelisted buyer before the sale ends.

### Remediation

Allow a small slippage tolerance or rounding margin on `amount0Min` and `amount1Min` (or omit the strict `totalRaised` minimum check in `ILOPool.launch()`, as `ILOManager.launch()` already validates `sqrtPriceX96 == initialPoolPriceX96` immediately prior to launching the pools). Additionally, refund any unused `RAISE_TOKEN` dust or apply an appropriate slippage factor:

```solidity
            // calculate sale amount of tokens needed for launching pool
            if (token0Addr == RAISE_TOKEN) {
                amount0 = totalRaised;
                (amount1, liquidity) = _saleAmountNeeded(totalRaised);
            } else {
                (amount0, liquidity) = _saleAmountNeeded(totalRaised);
                amount1 = totalRaised;
            }

            // actually deploy liquidity to uniswap pool
            (amount0, amount1) = addLiquidity(AddLiquidityParams({
                pool: IUniswapV3Pool(uniV3PoolAddress),
                liquidity: liquidity,
                amount0Desired: amount0,
                amount1Desired: amount1,
                amount0Min: 0,
                amount1Min: 0
            }));
```

## Finding 10: Flawed Slippage and Liquidity Execution Check in `LiquidityManagement.addLiquidity` Causes `ILOPool.launch` Denial of Service and Price Manipulation Exposure

### Synopsis

`LiquidityManagement.addLiquidity` mints a pre-calculated `params.liquidity` directly into the Uniswap V3 pool without enforcing upper desired token bounds (`amount0Desired`/`amount1Desired`) while enforcing strict lower bounds (`amount0 >= amount0Min && amount1 >= amount1Min`) against `totalRaised`. Consequently, rounding discrepancies or permissionless pool price movements cause `ILOPool.launch()` to revert, permanently locking launch and forcing investor refunds.

### Technical details

In `ILOPool.launch()`, the contract prepares to deploy the raised liquidity into the project's `UniswapV3Pool` (`_cachedUniV3PoolAddress`). It sets the raised token's desired and minimum amounts to exact `totalRaised` (and the sale token's minimum amount to `0`), calculates `(saleAmount, liquidity) = _saleAmountNeeded(totalRaised)` based on the cached initial price `SQRT_RATIO_X96`, and calls `LiquidityManagement.addLiquidity`:

```solidity
(amount0, amount1) = addLiquidity(AddLiquidityParams({
    pool: IUniswapV3Pool(uniV3PoolAddress),
    liquidity: liquidity,
    amount0Desired: amount0,
    amount1Desired: amount1,
    amount0Min: amount0Min,
    amount1Min: amount1Min
}));
```

In `src/base/LiquidityManagement.sol` (lines 36–57), `addLiquidity` was modified from the standard Uniswap V3 periphery implementation:

1. **Quote-to-execution price inconsistency & unused desired bounds:** Instead of computing the liquidity delta dynamically from the pool's current `slot0.sqrtPriceX96` and bounding token expenditures by `params.amount0Desired` and `params.amount1Desired`, `addLiquidity` ignores `amount0Desired` and `amount1Desired` altogether and passes the pre-computed `params.liquidity` directly to `params.pool.mint(...)` (lines 48–54).
2. **Strict minimum check sensitive to price manipulation and rounding:** At line 56, `addLiquidity` enforces:
   ```solidity
   require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
   ```
   Because `params.amount0Min` (when `token0 == RAISE_TOKEN`) or `params.amount1Min` (when `token1 == RAISE_TOKEN`) equals `totalRaised` with zero slippage tolerance:
   - **Pool-price manipulation DoS:** The underlying `UniswapV3Pool` is created and initialized before the ILO sale concludes. Since Uniswap V3 pool actions (`mint` and `swap`) are permissionless, any external actor can mint a minimal position and swap a negligible amount of tokens in the pool to shift `slot0.sqrtPriceX96`.
     - If the pool price is shifted toward the sale token, `params.pool.mint` requires less `RAISE_TOKEN` (`amount < totalRaised`) and more `SALE_TOKEN` (`amount > amountDesired`). This immediately causes `require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check')` at line 56 to revert (or reverts if `SALE_TOKEN` exceeds the contract's balance).
     - If the pool price is shifted toward the raise token, `params.pool.mint` requires `RAISE_TOKEN > totalRaised` and less `SALE_TOKEN` (since the sale token's `amountMin` is `0`, line 56 provides no lower-bound protection on `SALE_TOKEN`). Because `ILOPool` only holds `totalRaised` of `RAISE_TOKEN` (or if an attacker tops up the difference to extract discounted `SALE_TOKEN` via arbitrage), the callback payment either reverts due to insufficient `RAISE_TOKEN` balance or deploys liquidity at a manipulated price.
   - **Rounding-induced revert:** Even if the pool price remains untouched at `SQRT_RATIO_X96`, `_saleAmountNeeded(totalRaised)` computes `liquidity` by rounding down (`LiquidityAmounts.getLiquidityForAmount0` / `getLiquidityForAmount1`). When `UniswapV3Pool.mint` converts that truncated `liquidity` back into `amount0` and `amount1` via `SqrtPriceMath.getAmount0Delta` / `getAmount1Delta`, integer truncation in `liquidity` can yield an `amount0` (or `amount1`) strictly less than `totalRaised`, causing line 56 (`amount >= totalRaised`) to revert without any external intervention.

### Assumptions and preconditions

1. An `ILOPool` has been initialized for a project whose `UniswapV3Pool` is deployed and initialized at `SQRT_RATIO_X96`.
2. The ILO sale has reached at least `saleInfo.softCap` (`totalRaised >= saleInfo.softCap`), making the pool eligible to launch via `ILOPool.launch()`.
3. Either:
   - The conversion from `totalRaised` to `liquidity` and back to token delta in `UniswapV3Pool.mint` loses at least 1 wei due to integer division rounding, or
   - Any unprivileged user interacts with the `UniswapV3Pool` (via `mint` and `swap`) prior to `launch()` to move `slot0.sqrtPriceX96` away from `SQRT_RATIO_X96`.

### Proof of concept

1. A project initializes an `ILOPool` with `RAISE_TOKEN` as `token0` and `SALE_TOKEN` as `token1`, and the corresponding `UniswapV3Pool` is initialized at `SQRT_RATIO_X96`.
2. Investors participate in the ILO sale until `totalRaised >= saleInfo.softCap`.
3. Before the manager executes `launch()`, an unprivileged user calls `UniswapV3Pool.mint` with a tiny liquidity position around `SQRT_RATIO_X96` and calls `UniswapV3Pool.swap` with `zeroForOne = false` (selling a small amount of `token1` for `token0`), increasing `slot0.sqrtPriceX96`.
4. Alternatively, even without step 3, for values of `totalRaised` and `SQRT_RATIO_X96` where `getLiquidityForAmount0` rounds down such that `SqrtPriceMath.getAmount0Delta(SQRT_RATIO_X96, SQRT_RATIO_UPPER_X96, liquidity, true) < totalRaised`, the round-trip `amount0` is strictly less than `totalRaised`.
5. When `ILOManager` calls `ILOPool.launch()`, `addLiquidity` invokes `params.pool.mint(...)` with `params.liquidity`.
6. Because `slot0.sqrtPriceX96` is higher (or due to `liquidity` truncation), `UniswapV3Pool.mint` computes `amount0 < totalRaised`.
7. `addLiquidity` executes `require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check')` at line 56, where `params.amount0Min == totalRaised`. The check fails and the entire `launch()` transaction reverts.
8. Once the launch window expires because `launch()` cannot succeed, any investor can call `claimRefund()`, permanently setting `_refundTriggered = true` and bricking the ILO launch.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** `LiquidityManagement.addLiquidity` reverts with `'Price slippage check'` (or token transfer failure / unguarded sale token slippage) whenever the pool's current `sqrtPriceX96` deviates from `SQRT_RATIO_X96` or when `liquidity` rounding produces `amount < totalRaised`.
- **Business Implications:** Any malicious actor can grief and block the launch of any ILO pool at negligible cost by slightly nudging the Uniswap V3 pool price before `launch()`, or the launch can fail organically due to rounding. Once blocked until the refund deadline, investors can trigger refunds (`_refundTriggered = true`), permanently preventing the project from launching (`require(!_refundTriggered, "IRF")`). Conversely, if an attacker shifts the price toward `RAISE_TOKEN` and supplies the small excess `RAISE_TOKEN` to `ILOPool`, the lack of `amount1Min` (or `amount0Min` when `SALE_TOKEN` is `token0`) and upper bound checks allows liquidity to be minted at a manipulated price and arbitraged.

### Remediation

Ensure that the Uniswap V3 pool price at `launch()` matches the expected `SQRT_RATIO_X96` (or dynamically compute liquidity from `amount0Desired` and `amount1Desired` while restoring desired/minimum slippage tolerances for both tokens), and account for rounding when checking minimum token amounts in `LiquidityManagement.addLiquidity`:

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
        (uint160 sqrtPriceX96, , , , , , ) = params.pool.slot0();
        require(sqrtPriceX96 == SQRT_RATIO_X96, 'Price manipulated');

        (amount0, amount1) = params.pool.mint(
            address(this),
            TICK_LOWER,
            TICK_UPPER,
            params.liquidity,
            ""
        );

        require(amount0 <= params.amount0Desired && amount1 <= params.amount1Desired, 'Excessive token amount');
        require(amount0 >= params.amount0Min && amount1 >= params.amount1Min, 'Price slippage check');
    }
```

Additionally, in `ILOPool.launch()`, set a small rounding tolerance on `amount0Min` and `amount1Min` for both `RAISE_TOKEN` and `SALE_TOKEN` so that 1-wei integer truncation in `liquidity` calculation does not cause `addLiquidity` to revert.
