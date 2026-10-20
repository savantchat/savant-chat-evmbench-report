# Audit report — 2024-01-init-capital-invitational/H-02

## Finding 1: Premature Position Registration and Missing Reentrancy Protection in `MarginTradingHook.openPos` Allows Cross-Function Reentrancy on Partially Initialized Positions

### Synopsis

`MarginTradingHook.openPos` records `__marginPositions[initPosId]`, `lastPosIds[msg.sender]`, and `initPosIds[msg.sender][posId]` in storage before executing `_increasePosInternal` and without a `nonReentrant` guard, exposing partially initialized position state and intermediate contract token balances to cross-function reentrancy during token funding and swap callbacks.

### Technical details

In `MarginTradingHook.openPos` (lines 98–120), a new margin position is created and registered in storage before the position's funding, borrowing, swap, and collateralization lifecycle is completed:

1. Line 98 calls `IInitCore(CORE).createPos(_mode, _viewer)` to mint the underlying `initPosId` NFT to `MarginTradingHook`.
2. Lines 105–108 populate `__marginPositions[initPosId]`, increment `posId = ++lastPosIds[msg.sender]`, and bind `initPosIds[msg.sender][posId] = initPosId`.
3. Line 109 then calls `_increasePosInternal(...)`, which performs external calls in two stages:
   - `_transmitTokenIn(_param.tokenIn, _param.amtIn)` (line 151), which invokes `IERC20(_tokenIn).safeTransferFrom(msg.sender, address(this), amtToTransfer)`.
   - `IMulticall(CORE).multicall(multicallData)` (line 176), which borrows `_borrAmt` to `MarginTradingHook` and triggers `IInitCore(CORE).callback` -> `MarginTradingHook.coreCallback` -> `IBaseSwapHelper(swapHelper).swap(swapInfo)`.

Unlike `MoneyMarketHook`, which inherits `ReentrancyGuardUpgradeable` and protects `execute` with `nonReentrant`, `MarginTradingHook` has no reentrancy guard on `openPos` or its sibling external functions (`increasePos`, `addCollateral`, `removeCollateral`, `repayDebt`, `reducePos`, `fillOrder`). Furthermore, inside `InitCore`, `multicall` and `callback` do not hold `InitCore`'s `nonReentrant` lock while `coreCallback` is executing, and `MoeSwapHelper.swap` decodes an arbitrary user-supplied router `path` from `_data` (`abi.decode(_swapInfo.data, (address[], uint))`).

Because `MarginTradingHook` relies on `balanceOf(address(this))` across multiple functions—including `depositNative` / `refundNative` (lines 68, 75–82), `removeCollateral` (line 222), `reducePos` (line 327), and `coreCallback` (lines 408, 412)—and exposes `initPosIds[msg.sender][posId]` prior to `_increasePosInternal`, a reentrant call during `_transmitTokenIn` or during the DEX swap in `coreCallback` can interact with the partially initialized position or drain intermediate token balances (such as pre-deposited `WNATIVE` or swapped tokens) held by `MarginTradingHook`.

### Assumptions and preconditions

1. A caller invokes `MarginTradingHook.openPos` with either a token that triggers a transfer hook during `_transmitTokenIn` or a multi-hop swap `path` encoded in `_data` that routes through a token/pair giving execution control back to the caller during `MoeSwapHelper.swap`.
2. `MarginTradingHook` holds in-flight tokens during `openPos` (such as `WNATIVE` deposited via `depositNative` or tokens present in the hook during swap execution) or the caller interacts with the newly registered `posId` before `_increasePosInternal` finishes.

### Proof of concept

1. A user calls `MarginTradingHook.openPos` providing native token via `msg.value` (which `depositNative` wraps into `WNATIVE` held by `MarginTradingHook`) or collateral/borrow tokens, along with `_data` encoding a swap path `[borrToken, hookToken, collToken]`.
2. `openPos` creates `initPosId`, writes `__marginPositions[initPosId]`, and assigns `initPosIds[msg.sender][posId] = initPosId` (lines 98–108).
3. `openPos` calls `_increasePosInternal` (line 109), which initiates `InitCore.multicall`.
4. Step 0 of `multicall` borrows `_borrAmt` of `borrToken` to `MarginTradingHook`.
5. Step 1 of `multicall` invokes `InitCore.callback` -> `MarginTradingHook.coreCallback` -> `MoeSwapHelper.swap`.
6. During the router swap through `hookToken`, execution reenters `MarginTradingHook` while `posId` is already mapped in `initPosIds[msg.sender][posId]` and while `MarginTradingHook` holds in-flight token balances (`WNATIVE` / `collToken`).
7. The reentrant call invokes a `MarginTradingHook` function (such as an operation with `refundNative` or position modification on `posId`), sweeping `IERC20(WNATIVE).balanceOf(address(this))` or altering the position state before the outer `coreCallback`, `mintTo`, `collateralize`, and `_validateHealth` steps complete.

### Impact assessment

- **Severity**: High / Medium
- **Technical Impact**: Cross-function reentrancy allows manipulation of in-flight contract token balances (`balanceOf(address(this))`) and nested operations on a partially initialized margin position before collateralization and health validation complete.
- **Business Impact**: Potential accounting corruption between `MarginTradingHook` and `InitCore`, unexpected slippage/health check bypasses or reverts, and loss of in-flight tokens held transiently by `MarginTradingHook`.

### Remediation

Add reentrancy protection (`ReentrancyGuardUpgradeable` / `nonReentrant`) to `MarginTradingHook` entrypoints and defer exposing `lastPosIds` and `initPosIds` until `_increasePosInternal` has completed, or guard all external position management functions against reentrancy:

```solidity
    function openPos(
        uint16 _mode,
        address _viewer,
        address _tokenIn,
        uint _amtIn,
        address _borrPool,
        uint _borrAmt,
        address _collPool,
        bytes calldata _data,
        uint _minHealth_e18
    ) external payable nonReentrant depositNative refundNative returns (uint posId, uint initPosId, uint health_e18) {
        initPosId = IInitCore(CORE).createPos(_mode, _viewer);
        address borrToken = ILendingPool(_borrPool).underlyingToken();
        {
            (address baseToken, address quoteToken) = getBaseAssetAndQuoteAsset(
                ILendingPool(_collPool).underlyingToken(), borrToken
            );
            bool isLongBaseAsset = baseToken != borrToken;
            __marginPositions[initPosId] = MarginPos(_collPool, _borrPool, baseToken, quoteToken, isLongBaseAsset);
        }
        health_e18 = _increasePosInternal(
            IncreasePosInternalParam({
                initPosId: initPosId,
                tokenIn: _tokenIn,
                amtIn: _amtIn,
                borrPool: _borrPool,
                borrAmt: _borrAmt,
                collPool: _collPool,
                data: _data,
                minHealth_e18: _minHealth_e18
            })
        );
        posId = ++lastPosIds[msg.sender];
        initPosIds[msg.sender][posId] = initPosId;
    }
```

## Finding 2: Memory instead of storage update in `MarginTradingHook.fillOrder` fails to persist order cancellation when position collateral is zero

### Synopsis

In `MarginTradingHook.fillOrder`, when a position has zero collateral, the function attempts to cancel the order by updating `order.status = OrderStatus.Cancelled`. Because `order` is loaded as a `memory` struct rather than a `storage` reference, the status change is discarded upon return, leaving the order `Active` in storage.

### Technical details

In `contracts/hook/MarginTradingHook.sol`, `fillOrder(uint _orderId)` loads the order from storage into memory at line 373:

```solidity
Order memory order = __orders[_orderId];
```

At lines 379–383, the function checks whether the position associated with the order currently holds zero collateral in `POS_MANAGER`. If the collateral amount is zero, the contract intends to automatically cancel the order, emit a `CancelOrder` event, and return early:

```solidity
if (IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool) == 0) {
    order.status = OrderStatus.Cancelled;
    emit CancelOrder(order.initPosId, _orderId);
    return;
}
```

Unlike line 398 (`__orders[_orderId].status = OrderStatus.Filled;`), which explicitly writes to the `__orders` storage mapping, line 380 writes to `order.status`, which only mutates the transient `memory` copy. When `fillOrder` returns at line 382, the modification is lost and `__orders[_orderId].status` remains `OrderStatus.Active` in contract storage.

Consequently:
1. Off-chain indexers and user interfaces monitoring the emitted `CancelOrder(order.initPosId, _orderId)` event will record the order as cancelled, diverging from the authoritative on-chain state (`OrderStatus.Active`).
2. `fillOrder(_orderId)` can be called repeatedly to emit duplicate `CancelOrder` events while the position remains empty.
3. If the position owner subsequently adds collateral or increases the position under the same `posId` (`initPosId`), the stale order remains `OrderStatus.Active` and can be filled against the newly added collateral despite having been reported as cancelled.

### Assumptions and preconditions

1. A user has opened a margin position (`initPosId`) via `MarginTradingHook.openPos` and created a stop-loss or take-profit order (`_orderId`).
2. The user closes or reduces the position's collateral to `0` (e.g., via `reducePos` or `removeCollateral`) without explicitly calling `cancelOrder`.
3. Any actor calls `fillOrder(_orderId)` while the position's collateral is `0`, or the user later adds collateral / increases the position on the same `_posId`.

### Proof of concept

1. Alice opens a margin position via `MarginTradingHook.openPos`, receiving `posId = 1` (mapped to `initPosId = 1`), and places a stop-loss or take-profit order via `addStopLossOrder`, creating `orderId = 1` with `__orders[1].status == OrderStatus.Active`.
2. Alice fully closes her position via `reducePos` (reducing all collateral and repaying all debt), so `IPosManager(POS_MANAGER).getCollAmt(1, marginPos.collPool)` becomes `0`.
3. An executor (or keeper) calls `MarginTradingHook.fillOrder(1)`.
4. Inside `fillOrder(1)`, `getCollAmt(1, marginPos.collPool) == 0` evaluates to `true`. The function sets `order.status = OrderStatus.Cancelled` in memory, emits `CancelOrder(1, 1)`, and returns.
5. Querying `hook.getOrder(1).status` still returns `OrderStatus.Active` because `__orders[1].status` was never updated in storage.
6. Alice later reuses her position (`posId = 1`) by calling `increasePos` or `addCollateral` at a new market price. Because she and off-chain UIs observed `CancelOrder(1, 1)`, she assumes order `1` is cancelled.
7. When the market price satisfies the old `triggerPrice_e36` of order `1`, an arbitrager calls `fillOrder(1)` and fills the stale order against Alice's newly deposited collateral at outdated parameters.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** Order cancellation in `fillOrder` does not persist in storage, causing a mismatch between emitted `CancelOrder` events and on-chain state (`__orders[_orderId].status`).
- **Business / Financial Impact:** A user who closes a position and later reuses the same position ID (`addCollateral` or `increasePos`) remains exposed to a "cancelled" stale order being unexpectedly filled at outdated trigger and limit prices, leading to unintended position closure and financial loss.

### Remediation

Update the storage mapping `__orders[_orderId].status` directly when cancelling the order in `fillOrder`.

```solidity
    /// @inheritdoc IMarginTradingHook
    function fillOrder(uint _orderId) external {
        Order memory order = __orders[_orderId];
        _require(order.status == OrderStatus.Active, Errors.INVALID_INPUT);
        MarginPos memory marginPos = __marginPositions[order.initPosId];
        address collToken = ILendingPool(marginPos.collPool).underlyingToken();
        address borrToken = ILendingPool(marginPos.borrPool).underlyingToken();
        // if position is empty, cancel order
        if (IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool) == 0) {
            __orders[_orderId].status = OrderStatus.Cancelled;
            emit CancelOrder(order.initPosId, _orderId);
            return;
        }
        // validate trigger price condition
        _validateTriggerPrice(order, marginPos);
        // calculate fill order info
        (uint amtOut, uint repayShares, uint repayAmt) = _calculateFillOrderInfo(order, marginPos, collToken);
        // transfer in repay tokens
        IERC20(borrToken).safeTransferFrom(msg.sender, address(this), repayAmt);
        // transfer order owner's desired tokens to the specified recipient
        IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut);
        // repay tokens
        _ensureApprove(borrToken, repayAmt);
        IInitCore(CORE).repay(marginPos.borrPool, repayShares, order.initPosId);
        // decollateralize coll pool tokens to executor
        IInitCore(CORE).decollateralize(order.initPosId, marginPos.collPool, order.collAmt, msg.sender);
        // update order status
        __orders[_orderId].status = OrderStatus.Filled;
        emit FillOrder(order.initPosId, _orderId, order.tokenOut, amtOut);
    }
```

## Finding 3: Mismatched Return Value Order in `MarginTradingHook._calculateFillOrderInfo` Swaps `repayShares` and `repayAmt` in `MarginTradingHook.fillOrder`

### Synopsis

In `MarginTradingHook`, `_calculateRepaySize` returns `(uint repayAmt, uint repayShares)`, but `_calculateFillOrderInfo` destructures the tuple as `(repayShares, repayAmt)`. Consequently, `fillOrder` uses debt shares as the token transfer amount and passes the token amount as debt shares to `InitCore.repay`, causing order execution to revert or miscalculate settlement amounts.

### Technical details

In `contracts/hook/MarginTradingHook.sol`, `_calculateRepaySize` (lines 590–599) is defined with the return signature:

```solidity
function _calculateRepaySize(Order memory _order, MarginPos memory _marginPos)
    internal
    returns (uint repayAmt, uint repayShares)
```

However, when `_calculateFillOrderInfo` (lines 532–564) calls `_calculateRepaySize` at line 536, it unpacks the return values in the reverse order:

```solidity
(repayShares, repayAmt) = _calculateRepaySize(_order, _marginPos);
```

This inversion swaps the debt share denomination and the underlying debt token amount throughout `_calculateFillOrderInfo` and `fillOrder` (lines 387–400):

1. **Corrupted `amtOut` Calculation**: In `_calculateFillOrderInfo` (lines 544, 549, 556, and 561), `amtOut` (the amount of `order.tokenOut` paid to `order.recipient`) is computed by deducting the value of `repayAmt`. Because `repayAmt` actually holds the debt share count rather than the underlying asset amount—and `1` debt share accrues to `> 1` underlying asset over time via `ILendingPool.debtShareToAmtCurrent`—the borrower's debt obligation is understated and `amtOut` is overstated.
2. **Insufficient Token Pull and Approval**: In `fillOrder`, line 387 receives `(amtOut, repayShares, repayAmt)` from `_calculateFillOrderInfo`. Line 389 (`IERC20(borrToken).safeTransferFrom(msg.sender, address(this), repayAmt)`) transfers the smaller debt share count instead of the actual underlying token amount required to repay the debt, and line 393 (`_ensureApprove(borrToken, repayAmt)`) checks allowance against that smaller share count.
3. **Excessive Share Repayment and Revert in `InitCore.repay`**: Line 394 (`IInitCore(CORE).repay(marginPos.borrPool, repayShares, order.initPosId)`) passes `repayShares` (which actually holds the larger underlying token amount) as the `_shares` parameter to `InitCore.repay`. Inside `InitCore._repay` (lines 545–551), `sharesToRepay` is capped at `positionDebtShares`, and `amtToRepay = ILendingPool(_pool).debtShareToAmtCurrent(sharesToRepay)` computes an underlying token amount strictly greater than the `repayAmt` (debt share count) pulled into `MarginTradingHook` at line 389. When `InitCore._repay` executes `IERC20(tokenToRepay).safeTransferFrom(msg.sender, _pool, amtToRepay)` at line 551, the transfer from `MarginTradingHook` reverts due to insufficient token balance.

### Assumptions and preconditions

1. A user has opened a margin position via `MarginTradingHook.openPos` and created an active stop-loss or take-profit order (`addStopLossOrder` or `addTakeProfitOrder`).
2. Time has elapsed such that the borrow pool (`marginPos.borrPool`) has accrued non-zero interest, causing `ILendingPool(marginPos.borrPool).debtShareToAmtCurrent(shares) > shares`.
3. The oracle mark price satisfies `_validateTriggerPrice` for the order.

### Proof of concept

1. Alice opens a margin position via `MarginTradingHook.openPos`, borrowing `1,000e18` of `borrToken` from `borrPool`.
2. Interest accrues in `borrPool` over time such that Alice's `1,000e18` debt shares now correspond to `1,100e18` of `borrToken` (`debtShareToAmtCurrent(1,000e18) == 1,100e18`).
3. Alice places a stop-loss or take-profit order for 100% of her collateral (`order.collAmt == totalCollAmt`).
4. When the trigger price is reached, an executor calls `MarginTradingHook.fillOrder(orderId)`.
5. `_calculateRepaySize` computes `repayShares = 1,000e18` and `repayAmt = 1,100e18`, returning `(1,100e18, 1,000e18)`.
6. `_calculateFillOrderInfo` assigns `repayShares = 1,100e18` and `repayAmt = 1,000e18`, and computes `amtOut` using `1,000e18` instead of `1,100e18`.
7. In `fillOrder`:
   - Line 389 transfers `repayAmt` (`1,000e18`) of `borrToken` from the executor to `MarginTradingHook`.
   - Line 394 calls `IInitCore(CORE).repay(marginPos.borrPool, 1,100e18, order.initPosId)`.
8. Inside `InitCore._repay`:
   - `sharesToRepay` is capped at `positionDebtShares` (`1,000e18`).
   - `amtToRepay = debtShareToAmtCurrent(1,000e18) = 1,100e18`.
   - `IERC20(borrToken).safeTransferFrom(address(marginTradingHook), borrPool, 1,100e18)` reverts because `MarginTradingHook` only holds `1,000e18` of `borrToken`.

### Impact assessment

- **Severity**: High.
- **Technical Effect**: `MarginTradingHook.fillOrder` consistently reverts whenever the borrow pool has accrued any interest (`repayAmt > repayShares`), rendering stop-loss and take-profit orders unfillable. If `MarginTradingHook` happens to hold residual `borrToken` balances, partial orders will repay more debt shares than proportional to the order's collateral size while overpaying `amtOut` to the order recipient.
- **Business Implications**: Users relying on stop-loss orders to protect leveraged positions from liquidation cannot have their orders executed during adverse market movements, leading to unexpected liquidations and loss of user funds.

### Remediation

Unpack the return values of `_calculateRepaySize` in the matching order `(repayAmt, repayShares)` inside `_calculateFillOrderInfo`.

```solidity
    function _calculateFillOrderInfo(Order memory _order, MarginPos memory _marginPos, address _collToken)
        internal
        returns (uint amtOut, uint repayShares, uint repayAmt)
    {
        (repayAmt, repayShares) = _calculateRepaySize(_order, _marginPos);
        uint collTokenAmt = ILendingPool(_marginPos.collPool).toAmtCurrent(_order.collAmt);
```

## Finding 4: Reentrancy and CEI Violation in `MarginTradingHook.fillOrder` Allows Recursive and Cross-Function Order Manipulation

### Synopsis

`MarginTradingHook.fillOrder` lacks reentrancy protection and violates the Checks-Effects-Interactions pattern by updating `__orders[_orderId].status` to `OrderStatus.Filled` only after executing multiple external token transfers and core calls. Token transfer callbacks can re-enter `fillOrder` or other hook functions against stale order and position snapshots.

### Technical details

In `contracts/hook/MarginTradingHook.sol`, `fillOrder(uint _orderId)` loads `__orders[_orderId]` into memory at line 373, verifies `order.status == OrderStatus.Active` at line 374, and computes `(amtOut, repayShares, repayAmt)` via `_calculateFillOrderInfo` at line 387 based on the current collateral and debt shares of `order.initPosId`.

However, `fillOrder` performs external interactions at lines 389–396 before updating storage state at line 398:

```solidity
// transfer in repay tokens
IERC20(borrToken).safeTransferFrom(msg.sender, address(this), repayAmt);
// transfer order owner's desired tokens to the specified recipient
IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut);
// repay tokens
_ensureApprove(borrToken, repayAmt);
IInitCore(CORE).repay(marginPos.borrPool, repayShares, order.initPosId);
// decollateralize coll pool tokens to executor
IInitCore(CORE).decollateralize(order.initPosId, marginPos.collPool, order.collAmt, msg.sender);
// update order status
__orders[_orderId].status = OrderStatus.Filled;
```

Because `MarginTradingHook` does not apply a `nonReentrant` guard and defers writing `__orders[_orderId].status = OrderStatus.Filled` until line 398, two security flaws arise whenever `borrToken` or `order.tokenOut` supports transfer hooks (such as ERC777 tokens or ERC20 tokens with sender/recipient callbacks):

1. **Recursive Order Settlement (`fillOrder` -> `fillOrder`)**: During `safeTransferFrom` at line 389 or line 391, `__orders[_orderId].status` in storage is still `OrderStatus.Active`, and neither `IInitCore(CORE).repay` nor `IInitCore(CORE).decollateralize` has executed yet. A callback to `msg.sender` or `order.recipient` can recursively invoke `fillOrder(_orderId)`. If the position holds enough collateral (e.g., an order configured for a partial close of `order.collAmt <= totalCollAmt / 2`), both the inner and outer `fillOrder` executions succeed using the pre-interaction snapshots, settling the same order multiple times and extracting multiple `order.collAmt` slices from the user's position.
2. **Cross-Function Reentrancy with Stale Snapshots**: When `IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut)` at line 391 invokes a recipient hook on `order.recipient` (the position owner), `msg.sender` has already paid `repayAmt` to `MarginTradingHook` and `amtOut` to `order.recipient`. Because the order is still `Active` and the position has not yet been modified in `InitCore`, `order.recipient` can re-enter `MarginTradingHook` to call `repayDebt`, `removeCollateral`, `reducePos`, `updateOrder`, or `cancelOrder` while the outer `fillOrder` resumes execution with stale `order`, `repayShares`, and `repayAmt` values.

### Assumptions and preconditions

1. An active stop-loss or take-profit order exists in `MarginTradingHook` whose trigger price condition (`_validateTriggerPrice`) is satisfied.
2. Either `borrToken` or `order.tokenOut` (or the underlying collateral/borrow token in the lending pool) implements transfer callbacks (e.g., ERC777 `tokensToSend`/`tokensReceived` or callback-enabled ERC20 tokens), allowing `msg.sender` or `order.recipient` to execute code during `safeTransferFrom`.
3. For recursive `fillOrder` exploitation, the position holds sufficient collateral and debt to cover the repeated decollateralization and repayment (`order.collAmt` is a partial portion of the position's total collateral).

### Proof of concept

1. Alice opens a leveraged position via `MarginTradingHook.openPos` with `totalCollAmt = 100` shares in `collPool` and `totalDebtShares = 100` in `borrPool`.
2. Alice creates a take-profit or stop-loss order (`orderId = 1`) for half of her position (`order.collAmt = 50`).
3. Once the oracle price reaches `triggerPrice_e36`, an executor calls `fillOrder(1)`:
   - `fillOrder(1)` reads `order.status == OrderStatus.Active` and computes `repayShares = 50`, `repayAmt`, and `amtOut` from the initial state (`totalCollAmt = 100`, `totalDebtShares = 100`).
   - At line 389 (`IERC20(borrToken).safeTransferFrom(msg.sender, address(this), repayAmt)`) or line 391 (`IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut)`), the token contract triggers a transfer hook on the caller/recipient.
4. Inside the callback, `fillOrder(1)` is invoked a second time:
   - Because `__orders[1].status` has not yet been updated to `OrderStatus.Filled` and `repay`/`decollateralize` have not yet been called, the inner `fillOrder(1)` sees the exact same `OrderStatus.Active`, `totalCollAmt = 100`, and `totalDebtShares = 100`.
   - The inner `fillOrder(1)` transfers `repayAmt` and `amtOut`, repays `50` debt shares, decollateralizes `50` collateral shares to `msg.sender`, and sets `__orders[1].status = OrderStatus.Filled`.
5. Execution returns to the outer `fillOrder(1)` right after the token transfer:
   - The outer call proceeds to line 394 (`IInitCore(CORE).repay`) and line 396 (`IInitCore(CORE).decollateralize`) using the cached `repayShares = 50` and `order.collAmt = 50`, repaying the remaining `50` debt shares and decollateralizing the remaining `50` collateral shares to `msg.sender`, then writing `__orders[1].status = OrderStatus.Filled` again at line 398.
   - As a result, a single order intended to close 50% of the position is filled twice, liquidating 100% of the position's collateral.

### Impact assessment

- **Severity**: High
- **Technical Impact**: An active order can be settled multiple times in a single transaction, or position state (`collAmt`, `debtShares`, order parameters, or status) can be mutated mid-settlement via cross-function reentrancy while `fillOrder` continues using stale pre-transfer calculations.
- **Business Impact**: Position owners can suffer unauthorized full or excess liquidation of their margin positions beyond the configured `order.collAmt`, or order executors can have their transferred repayment tokens consumed against a manipulated position state.

### Remediation

Apply the Checks-Effects-Interactions (CEI) pattern in `fillOrder` by updating `__orders[_orderId].status = OrderStatus.Filled` before performing any external calls, and add OpenZeppelin's `ReentrancyGuardUpgradeable` (`nonReentrant`) to `MarginTradingHook` functions that modify position or order state.

```solidity
    function fillOrder(uint _orderId) external nonReentrant {
        Order memory order = __orders[_orderId];
        _require(order.status == OrderStatus.Active, Errors.INVALID_INPUT);
        MarginPos memory marginPos = __marginPositions[order.initPosId];
        address collToken = ILendingPool(marginPos.collPool).underlyingToken();
        address borrToken = ILendingPool(marginPos.borrPool).underlyingToken();
        // if position is empty, cancel order
        if (IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool) == 0) {
            __orders[_orderId].status = OrderStatus.Cancelled;
            emit CancelOrder(order.initPosId, _orderId);
            return;
        }
        // validate trigger price condition
        _validateTriggerPrice(order, marginPos);
        // update order status before external interactions
        __orders[_orderId].status = OrderStatus.Filled;
        // calculate fill order info
        (uint amtOut, uint repayShares, uint repayAmt) = _calculateFillOrderInfo(order, marginPos, collToken);
        // transfer in repay tokens
        IERC20(borrToken).safeTransferFrom(msg.sender, address(this), repayAmt);
        // transfer order owner's desired tokens to the specified recipient
        IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut);
        // repay tokens
        _ensureApprove(borrToken, repayAmt);
        IInitCore(CORE).repay(marginPos.borrPool, repayShares, order.initPosId);
        // decollateralize coll pool tokens to executor
        IInitCore(CORE).decollateralize(order.initPosId, marginPos.collPool, order.collAmt, msg.sender);
        emit FillOrder(order.initPosId, _orderId, order.tokenOut, amtOut);
    }
```

## Finding 5: Missing Position Ownership Validation in `MarginTradingHook.updateOrder` Allows Unauthorized Modification of Active Orders

### Synopsis

`MarginTradingHook.updateOrder` fails to verify that the target order belongs to the caller's margin position (`order.initPosId == initPosId`), unlike `cancelOrder`. Consequently, any user with a valid margin position can maliciously overwrite the parameters of any active order in the protocol, enabling theft of user funds or forced losses.

### Technical details

In `MarginTradingHook`, users manage margin positions mapped via `initPosIds[msg.sender][_posId]`. When an order is created in `_createOrder` (lines 470–501), the order struct is initialized with `initPosId: initPosId` (line 489) and `recipient: msg.sender` (line 496) to bind the order to the position owner.

In `cancelOrder` (lines 361–369), the contract retrieves the caller's `initPosId` and explicitly verifies ownership of the order at line 365:
```solidity
uint initPosId = initPosIds[msg.sender][_posId];
_require(initPosId != 0, Errors.POSITION_NOT_FOUND);
Order storage order = __orders[_orderId];
_require(order.initPosId == initPosId, Errors.INVALID_INPUT);
```

However, in `updateOrder` (lines 504–526), this authorization check is omitted:
```solidity
Order storage order = __orders[_orderId];
_require(order.status == OrderStatus.Active, Errors.INVALID_INPUT);
uint initPosId = initPosIds[msg.sender][_posId];
_require(initPosId != 0, Errors.POSITION_NOT_FOUND);
MarginPos memory marginPos = __marginPositions[initPosId];
uint collAmt = IPosManager(POS_MANAGER).getCollAmt(initPosId, marginPos.collPool);
_require(_collAmt <= collAmt, Errors.INPUT_TOO_HIGH);

order.triggerPrice_e36 = _triggerPrice_e36;
order.limitPrice_e36 = _limitPrice_e36;
order.collAmt = _collAmt;
order.tokenOut = _tokenOut;
```

Because `updateOrder` never checks that `order.initPosId == initPosId` (and also lacks the `_tokenOut` validation present at line 483 in `_createOrder`), any attacker who owns a position `_posId` (such that `initPosIds[attacker][_posId] != 0` and has sufficient collateral `collAmt >= _collAmt`) can pass another user's active `_orderId` to `updateOrder`. 

When `fillOrder(_orderId)` is subsequently executed, it uses `order.initPosId` (the victim's position) to decollateralize `order.collAmt` to the `fillOrder` caller (`msg.sender`), while pricing the repayment and output amount using the attacker-supplied `triggerPrice_e36`, `limitPrice_e36`, `collAmt`, and `tokenOut`.

### Assumptions and preconditions

1. A victim has an open margin position (`initPosId_victim`) with an active stop-loss or take-profit order (`_orderId`).
2. An attacker can open (or already holds) a margin position (`_posId_attacker`) with collateral amount at least equal to the desired `_collAmt` to set on the victim's order (or any non-zero amount to corrupt the order).

### Proof of concept

1. **Victim Setup**: Alice opens a long ETH/USDT margin position (`posId = 1`, `initPosId = 1`) with 10 WETH collateral and 10,000 USDT debt (current ETH price = 2,000 USDT). Alice calls `addTakeProfitOrder` creating `orderId = 1` with `triggerPrice_e36 = 2500e36`, `limitPrice_e36 = 2500e36`, `tokenOut = USDT`, and `collAmt = 10 WETH` (in pool shares).
2. **Attacker Setup**: Bob opens a margin position (`posId = 1`, `initPosId = 2`) with 10 WETH collateral (or flashloans to open it).
3. **Unauthorized Order Modification**: Bob calls `updateOrder` passing his own `_posId = 1` and Alice's `_orderId = 1`:
   - `_triggerPrice_e36 = 1` (ensuring `markPrice_e36 >= _order.triggerPrice_e36` is immediately satisfied at the current price of 2,000 USDT).
   - `_limitPrice_e36 = 1000e36` (artificially low limit price so `amtOut` paid to Alice becomes `0`: `(10 * 1000) - 10000 = 0`).
   - `_tokenOut = USDT`.
   - `_collAmt = 10 WETH` (in pool shares, validated against Bob's position collateral).
4. **Order Execution & Theft**: In the same transaction, Bob calls `fillOrder(1)`:
   - `_validateTriggerPrice` passes because `2000e36 >= 1`.
   - `_calculateFillOrderInfo` computes `repayAmt = 10,000 USDT` and `amtOut = 0 USDT` (due to the manipulated `limitPrice_e36`).
   - Bob pays 10,000 USDT to repay Alice's debt and pays 0 USDT to Alice (`order.recipient`).
   - `fillOrder` decollateralizes 10 WETH worth of collateral shares (worth 20,000 USDT) from Alice's position (`order.initPosId = 1`) directly to Bob (`msg.sender`).
   - Bob steals Alice's entire net equity of 10,000 USDT.

### Impact assessment

- **Severity**: Critical / High
- **Direct Technical Effect**: Any user with a margin position can arbitrarily modify the `triggerPrice_e36`, `limitPrice_e36`, `collAmt`, and `tokenOut` of any active order belonging to any other position in `MarginTradingHook`.
- **Business and Financial Impact**: Attackers can immediately trigger and fill any user's active stop-loss or take-profit order at a manipulated `limitPrice_e36` (and/or a malicious/worthless `tokenOut`), allowing the attacker to seize the victim's position collateral simply by repaying the position's debt and paying zero (or worthless tokens) to the victim. This results in direct theft of the victim's margin equity.

### Remediation

In `MarginTradingHook.updateOrder`, validate that `order.initPosId == initPosId` to ensure the caller owns the position associated with the order, and validate that `_tokenOut` is either `marginPos.baseAsset` or `marginPos.quoteAsset` (matching the validation in `_createOrder`).

```solidity
    /// @inheritdoc IMarginTradingHook
    function updateOrder(
        uint _posId,
        uint _orderId,
        uint _triggerPrice_e36,
        address _tokenOut,
        uint _limitPrice_e36,
        uint _collAmt
    ) external {
        _require(_collAmt != 0, Errors.ZERO_VALUE);
        Order storage order = __orders[_orderId];
        _require(order.status == OrderStatus.Active, Errors.INVALID_INPUT);
        uint initPosId = initPosIds[msg.sender][_posId];
        _require(initPosId != 0, Errors.POSITION_NOT_FOUND);
        _require(order.initPosId == initPosId, Errors.INVALID_INPUT);
        MarginPos memory marginPos = __marginPositions[initPosId];
        _require(_tokenOut == marginPos.baseAsset || _tokenOut == marginPos.quoteAsset, Errors.INVALID_INPUT);
        uint collAmt = IPosManager(POS_MANAGER).getCollAmt(initPosId, marginPos.collPool);
        _require(_collAmt <= collAmt, Errors.INPUT_TOO_HIGH);

        order.triggerPrice_e36 = _triggerPrice_e36;
        order.limitPrice_e36 = _limitPrice_e36;
        order.collAmt = _collAmt;
        order.tokenOut = _tokenOut;
        emit UpdateOrder(initPosId, _orderId, _tokenOut, _triggerPrice_e36, _limitPrice_e36, _collAmt);
    }
```

## Finding 6: Flawed Collateral Commitment Validation in `MarginTradingHook._createOrder` Allows Overcommitted and Stale Order Sizes That Cannot Be Filled

### Synopsis

In `MarginTradingHook._createOrder`, the order's `_collAmt` is only checked against a point-in-time snapshot of the position's total collateral (`getCollAmt`), ignoring collateral already committed to other active orders and subsequent collateral reductions. Consequently, active orders can exceed available position collateral, causing `fillOrder` to revert with an underflow.

### Technical details

When a user creates a stop-loss or take-profit order via `addStopLossOrder` or `addTakeProfitOrder`, `MarginTradingHook._createOrder` validates the requested collateral size `_collAmt` at lines 484–485:

```solidity
uint collAmt = IPosManager(POS_MANAGER).getCollAmt(initPosId, marginPos.collPool);
_require(_collAmt <= collAmt, Errors.INPUT_TOO_HIGH); // check specified coll amt is feasible
```

It then stores the order in `__orders[orderId]` with `collAmt: _collAmt` and `status: OrderStatus.Active` (lines 487–498) and appends `orderId` to `__posOrderIds[initPosId]` (line 499).

This collateral commitment logic is flawed in two interrelated ways:

1. **Unbounded cumulative commitments across active orders:** `_createOrder` checks `_collAmt <= collAmt` against the position's entire collateral balance in `PosManager` without subtracting the `collAmt` already committed to existing `OrderStatus.Active` orders registered in `__posOrderIds[initPosId]`. A position owner can create multiple active orders (e.g., multiple partial take-profit or stop-loss orders) whose individual `_collAmt` values are each `<= collAmt`, but whose sum exceeds the position's total collateral.
2. **Stale point-in-time collateral snapshot:** The collateral committed in `Order.collAmt` at line 492 is a static share amount verified only at order creation time, while the position's collateral in `PosManager` can decrease before execution—either because one of the other active orders is filled via `fillOrder`, or because the position's collateral is reduced via `removeCollateral`, `reducePos`, or `liquidate`.

When the position's remaining collateral `totalCollAmt` falls below `order.collAmt` (while remaining `> 0`, so the empty-position cancellation check at line 379 is bypassed), the order remains `OrderStatus.Active` but becomes unfillable:
- In `_calculateFillOrderInfo` (line 536), `_calculateRepaySize(_order, _marginPos)` attempts to cap `_order.collAmt` to `totalCollAmt` (line 595: `if (_order.collAmt > totalCollAmt) _order.collAmt = totalCollAmt;`), but `_order` is a `memory` struct passed by value, so the capped value never propagates back to `_calculateFillOrderInfo` or `fillOrder`.
- `_calculateFillOrderInfo` therefore computes `collTokenAmt` and `amtOut` using the stale, unscaled `_order.collAmt`, and `fillOrder` (line 396) calls `IInitCore(CORE).decollateralize(order.initPosId, marginPos.collPool, order.collAmt, msg.sender)`.
- Inside `PosManager.removeCollateralTo` (line 254), `posCollInfo.collAmts[_pool] - _shares` reverts with an arithmetic underflow because `order.collAmt > posCollInfo.collAmts[_pool]`.

### Assumptions and preconditions

- A user has an open margin trading position with non-zero collateral in `PosManager`.
- The user creates an order via `addStopLossOrder` or `addTakeProfitOrder` (invoking `_createOrder`), and before this order is filled, the position's collateral is partially reduced—either by filling another active order created on the same position, or by reducing collateral through `removeCollateral`, `reducePos`, or partial liquidation.

### Proof of concept

1. Alice opens a margin position (`initPosId = 1`) holding `100e18` collateral shares in `marginPos.collPool`.
2. Alice calls `addTakeProfitOrder` (`_createOrder`) to create Order 1 with `_collAmt = 60e18`. In `_createOrder` (lines 484–485), `60e18 <= 100e18` passes, and Order 1 is saved as `OrderStatus.Active`.
3. Alice calls `addTakeProfitOrder` (`_createOrder`) to create Order 2 with `_collAmt = 60e18`. Because `_createOrder` only checks `60e18 <= 100e18` against `getCollAmt` and ignores Order 1's commitment, Order 2 also passes validation and is saved as `OrderStatus.Active` (alternatively, Alice creates a single order with `_collAmt = 100e18` and later withdraws `40e18` via `removeCollateral`).
4. When the trigger price is reached, an executor fills Order 1 via `fillOrder(1)`. `60e18` collateral shares are decollateralized, leaving `40e18` collateral shares in `PosManager` for `initPosId = 1`.
5. When an executor attempts to fill Order 2 via `fillOrder(2)`:
   - Line 379 checks `getCollAmt(1, marginPos.collPool) == 0`, which is false (`40e18 != 0`).
   - `_calculateFillOrderInfo` calls `_calculateRepaySize`, which caps its local copy `_order.collAmt` to `40e18`, but `order.collAmt` in `_calculateFillOrderInfo` and `fillOrder` remains `60e18`.
   - `_calculateFillOrderInfo` computes `amtOut` using `60e18` worth of collateral while only `40e18` worth of debt is repaid.
   - Line 396 calls `IInitCore(CORE).decollateralize(1, marginPos.collPool, 60e18, msg.sender)`, which reverts in `PosManager.removeCollateralTo` at `40e18 - 60e18` due to arithmetic underflow.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Active stop-loss and take-profit orders whose committed `collAmt` exceeds the position's remaining collateral cannot be executed by order fillers, as `fillOrder` unconditionally reverts on underflow during `decollateralize` (and miscalculates `amtOut` prior to the revert).
- **Business Impact:** Core stop-loss and take-profit functionality breaks whenever multiple partial orders are placed that exceed 100% of collateral in aggregate or whenever a position's collateral is partially reduced via withdrawals, position reductions, or partial liquidations, leaving users' positions unprotected during adverse market movements.

### Remediation

Track and validate cumulative collateral commitments across active orders of the same type in `_createOrder`, and ensure that if a position's actual collateral falls below an order's committed `collAmt` at execution time, the capped collateral amount is propagated to `_calculateFillOrderInfo` and `fillOrder`.

Within `_createOrder`, enforce that the sum of `_collAmt` across active orders of the same `OrderType` for `initPosId` does not exceed the position's available `collAmt`:

```solidity
        _require(_tokenOut == marginPos.baseAsset || _tokenOut == marginPos.quoteAsset, Errors.INVALID_INPUT);
        uint collAmt = IPosManager(POS_MANAGER).getCollAmt(initPosId, marginPos.collPool);
        uint[] memory orderIds = __posOrderIds[initPosId];
        uint committedCollAmt;
        for (uint i = 0; i < orderIds.length; ++i) {
            Order storage existingOrder = __orders[orderIds[i]];
            if (existingOrder.status == OrderStatus.Active && existingOrder.orderType == _orderType) {
                committedCollAmt += existingOrder.collAmt;
            }
        }
        _require(_collAmt + committedCollAmt <= collAmt, Errors.INPUT_TOO_HIGH); // check specified coll amt is feasible

        __orders[orderId] = (
            Order({
                initPosId: initPosId,
                triggerPrice_e36: _triggerPrice_e36,
                limitPrice_e36: _limitPrice_e36,
                collAmt: _collAmt,
                tokenOut: _tokenOut,
                orderType: _orderType,
                status: OrderStatus.Active,
                recipient: msg.sender
            })
        );
        __posOrderIds[initPosId].push(orderId);
        emit CreateOrder(initPosId, orderId, _tokenOut, _triggerPrice_e36, _limitPrice_e36, _collAmt, _orderType);
    }
```

## Finding 7: Missing `_tokenOut` validation in `MarginTradingHook.updateOrder` allows setting arbitrary settlement tokens

### Synopsis

`MarginTradingHook.updateOrder` fails to validate that `_tokenOut` matches the margin position's `baseAsset` or `quoteAsset`, unlike `_createOrder`. A position owner can update an active order's `tokenOut` to an arbitrary ERC20 token, causing `fillOrder` to miscalculate `amtOut` and pull arbitrary approved tokens from order executors.

### Technical details

When creating a stop-loss or take-profit order in `MarginTradingHook._createOrder`, line 483 enforces settlement-asset compatibility with the position's trading pair:

```solidity
_require(_tokenOut == marginPos.baseAsset || _tokenOut == marginPos.quoteAsset, Errors.INVALID_INPUT);
```

This invariant is required because `_calculateFillOrderInfo` (lines 532–564) assumes `_order.tokenOut` is exclusively either the position's collateral token (`_collToken`) or its borrow token (`borrToken`):

```solidity
if (_collToken == _order.tokenOut) {
    ...
} else {
    // Assumes _order.tokenOut == borrToken
    ...
}
```

However, in `MarginTradingHook.updateOrder` (lines 504–526), the check verifying that `_tokenOut` is either `marginPos.baseAsset` or `marginPos.quoteAsset` is omitted before assigning `order.tokenOut = _tokenOut;` on line 524.

Consequently, a user can create a valid order via `addStopLossOrder` or `addTakeProfitOrder` and subsequently call `updateOrder` to set `order.tokenOut` to any arbitrary address. When an executor calls `fillOrder`:
1. `_calculateFillOrderInfo` takes the `else` branch (since `_collToken != _order.tokenOut`) and computes `amtOut` denominated in `borrToken` units and decimals.
2. Line 391 executes `IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut)`, transferring `amtOut` of the arbitrary `order.tokenOut` from the executor (`msg.sender`) to `order.recipient` if the executor has approved `MarginTradingHook` for that token, or reverting / failing to settle properly otherwise.

### Assumptions and preconditions

1. A user has opened a margin position via `MarginTradingHook.openPos` and created an active stop-loss or take-profit order.
2. For token draining from an executor, the executor (`msg.sender` of `fillOrder`) holds balances and has granted token allowance to `MarginTradingHook` for another ERC20 token (which is standard for keeper/arbitrage contracts that pre-approve `MarginTradingHook` across multiple supported tokens, as seen in `_fillOrder` in `TestMarginTradingHelper.sol`).

### Proof of concept

1. Alice opens a margin position with `collToken = WETH` and `borrToken = USDT` (or another low-value/high-decimal token) and creates a valid stop-loss order via `addStopLossOrder(posId, triggerPrice, USDT, limitPrice, collAmt)`.
2. Alice calls `updateOrder(posId, orderId, triggerPrice, WBTC, limitPrice, collAmt)`, setting `_tokenOut` to `WBTC` (or another high-value token not part of the position's pair). The call succeeds because `updateOrder` does not validate `_tokenOut`.
3. A keeper/executor bot that has approved `MarginTradingHook` to spend various tokens (including `WBTC` and `USDT`) calls `fillOrder(orderId)` when the trigger condition is met.
4. `_calculateFillOrderInfo` sees `_collToken (WETH) != order.tokenOut (WBTC)` and calculates `amtOut` as if `order.tokenOut` were `borrToken` (`USDT`).
5. `fillOrder` executes `IERC20(WBTC).safeTransferFrom(msg.sender, order.recipient, amtOut)`, transferring `amtOut` of `WBTC` (scaled to `USDT` amounts) from the executor to Alice.

### Impact assessment

- **Severity:** High / Medium
- **Technical Impact:** Order parameter integrity is violated on update, breaking the core invariant that `order.tokenOut` must be either `marginPos.baseAsset` or `marginPos.quoteAsset`.
- **Business Impact:** Keepers/arbitragers who pre-approve `MarginTradingHook` for multiple tokens to fill orders can have arbitrary approved tokens drained at the numeric quantity of `borrToken`, or orders can be bricked/manipulated with incompatible settlement assets.

### Remediation

Validate that `_tokenOut` is either `marginPos.baseAsset` or `marginPos.quoteAsset` in `updateOrder` (and ensure the order belongs to `initPosId`), matching the validation in `_createOrder`.

```solidity
    function updateOrder(
        uint _posId,
        uint _orderId,
        uint _triggerPrice_e36,
        address _tokenOut,
        uint _limitPrice_e36,
        uint _collAmt
    ) external {
        _require(_collAmt != 0, Errors.ZERO_VALUE);
        Order storage order = __orders[_orderId];
        _require(order.status == OrderStatus.Active, Errors.INVALID_INPUT);
        uint initPosId = initPosIds[msg.sender][_posId];
        _require(initPosId != 0, Errors.POSITION_NOT_FOUND);
        MarginPos memory marginPos = __marginPositions[initPosId];
        _require(_tokenOut == marginPos.baseAsset || _tokenOut == marginPos.quoteAsset, Errors.INVALID_INPUT);
        uint collAmt = IPosManager(POS_MANAGER).getCollAmt(initPosId, marginPos.collPool);
        _require(_collAmt <= collAmt, Errors.INPUT_TOO_HIGH);

        order.triggerPrice_e36 = _triggerPrice_e36;
        order.limitPrice_e36 = _limitPrice_e36;
        order.collAmt = _collAmt;
        order.tokenOut = _tokenOut;
        emit UpdateOrder(initPosId, _orderId, _tokenOut, _triggerPrice_e36, _limitPrice_e36, _collAmt);
    }
```
