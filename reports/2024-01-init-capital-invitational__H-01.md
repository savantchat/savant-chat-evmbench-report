# Audit report — 2024-01-init-capital-invitational/H-01

## Finding 1: Non-Zero to Non-Zero `safeApprove` in `BaseMappingIdHook._ensureApprove` Causes Denial of Service on Allowance Renewal

### Synopsis

`BaseMappingIdHook._ensureApprove` calls `SafeERC20.safeApprove(CORE, type(uint).max)` whenever the current allowance falls below `_amt`, without first resetting the allowance to zero. Once the allowance decreases to a non-zero value smaller than `_amt`, all subsequent calls to `_ensureApprove` permanently revert, blocking debt repayment and position management.

### Technical details

In `contracts/hook/BaseMappingIdHook.sol`, the internal helper `_ensureApprove` is implemented as follows:

```solidity
function _ensureApprove(address _token, uint _amt) internal {
    if (IERC20(_token).allowance(address(this), CORE) < _amt) {
        IERC20(_token).safeApprove(CORE, type(uint).max);
    }
}
```

OpenZeppelin's `SafeERC20.safeApprove` enforces a strict requirement that either the new allowance is `0` or the current allowance is `0`:

```solidity
require(
    (value == 0) || (token.allowance(address(this), spender) == 0),
    "SafeERC20: approve from non-zero to non-zero allowance"
);
```

Furthermore, tokens such as `USDT` (which is configured as a supported token in this protocol) also revert in their own `approve` implementation when changing an existing non-zero allowance to another non-zero value.

When `_ensureApprove` is called for the first time for a given `_token`, the allowance from the hook contract (`MarginTradingHook`) to `CORE` is `0`, so `safeApprove(CORE, type(uint).max)` succeeds. However, for any ERC20 token that decrements allowance in `transferFrom` even when the allowance is `type(uint).max` (such as `USDT` and many standard ERC20 implementations), every repayment made via `CORE.repay` decreases the hook's remaining allowance to `CORE`.

Once the remaining allowance drops to a non-zero value `0 < allowance < _amt`, the condition `IERC20(_token).allowance(address(this), CORE) < _amt` evaluates to `true`, and `_ensureApprove` attempts to call `IERC20(_token).safeApprove(CORE, type(uint).max)`. Because both the current allowance and the target value (`type(uint).max`) are non-zero, `safeApprove` (and the underlying token's `approve` if it enforces the same check) reverts.

Because `MarginTradingHook` inherits from `BaseMappingIdHook` and relies on `_ensureApprove(borrToken, ...)` in `repayDebt` (line 249), `_reducePosInternal` (line 298), and `fillOrder` (line 393), all of these functions will revert whenever `_amt` exceeds the remaining non-zero allowance.

### Assumptions and preconditions

1. A lending pool's underlying token is an ERC20 token that decrements allowance upon `transferFrom` even when the allowance is set to `type(uint).max` (e.g., `USDT` or standard ERC20 implementations without infinite-allowance checks).
2. Cumulative repayments through `MarginTradingHook` (or large flashloan-assisted repayments) reduce the hook's remaining allowance to `CORE` to a non-zero value smaller than the next required repayment amount `_amt`.

### Proof of concept

1. A user opens a margin position on `MarginTradingHook` borrowing `USDT` (or another ERC20 token that decrements `type(uint).max` allowances on `transferFrom`).
2. The user calls `repayDebt` for the first time. Inside `_ensureApprove(USDT, amtToRepay)`, `USDT.allowance(address(hook), CORE)` is `0`, which is less than `amtToRepay`.
3. `USDT.safeApprove(CORE, type(uint).max)` succeeds, setting the allowance to `type(uint).max`, and `CORE.repay` pulls `amtToRepay`, reducing the allowance to `type(uint).max - amtToRepay`.
4. Over time (or via large flashloan-backed position increases and repayments), cumulative repayments reduce `USDT.allowance(address(hook), CORE)` to a non-zero value `R` (e.g., `1,000e6`).
5. A user attempts to call `repayDebt`, `reducePos`, or `fillOrder` where the required repayment amount `amtToRepay` is `1,500e6` (`amtToRepay > R > 0`).
6. `_ensureApprove` checks `allowance < amtToRepay` (`1,000e6 < 1,500e6`, which is `true`) and executes `USDT.safeApprove(CORE, type(uint).max)`.
7. `SafeERC20.safeApprove` reverts with `"SafeERC20: approve from non-zero to non-zero allowance"`, blocking the user from repaying debt, reducing/closing their margin position, or having their stop-loss/take-profit orders filled.

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: `_ensureApprove` reverts whenever `0 < allowance(address(this), CORE) < _amt`.
- **Business Impact**: Users in `MarginTradingHook` cannot repay their debt (`repayDebt`), reduce or close their margin positions (`reducePos`), or have stop-loss/take-profit orders executed (`fillOrder`) for the affected borrow token, exposing their positions to accrued interest and forced liquidation unless someone performs a repayment equal to the exact remaining allowance down to the last wei.

### Remediation

Reset the allowance to `0` before approving `type(uint).max` (or use `forceApprove` if upgrading to a newer OpenZeppelin version).

```solidity
    function _ensureApprove(address _token, uint _amt) internal {
        if (IERC20(_token).allowance(address(this), CORE) < _amt) {
            IERC20(_token).safeApprove(CORE, 0);
            IERC20(_token).safeApprove(CORE, type(uint).max);
        }
    }
```

## Finding 2: Missing Reentrancy Protection and Deferred Order Status Update in `MarginTradingHook.fillOrder` Allow Settlement Reentrancy

### Synopsis

`MarginTradingHook.fillOrder` lacks reentrancy protection and defers updating `__orders[_orderId].status` in storage until after multiple external calls. This violates the Checks-Effects-Interactions pattern, enabling recursive same-order execution, cross-order reentrancy, and cross-function position-state mutation during order settlement.

### Technical details

In ` contracts/hook/MarginTradingHook.sol`, `fillOrder(uint _orderId)` (line 372) loads the target order into memory (`Order memory order = __orders[_orderId];` at line 373) and checks `order.status == OrderStatus.Active` at line 374. However, the function does not use a `nonReentrant` guard, and it defers writing the updated status (`__orders[_orderId].status = OrderStatus.Filled;`) to storage until line 398—after performing external calls to token contracts and `InitCore`:

1. `IERC20(borrToken).safeTransferFrom(msg.sender, address(this), repayAmt)` (line 389)
2. `IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut)` (line 391)
3. `IInitCore(CORE).repay(marginPos.borrPool, repayShares, order.initPosId)` (line 394)
4. `IInitCore(CORE).decollateralize(order.initPosId, marginPos.collPool, order.collAmt, msg.sender)` (line 396)

Additionally, when a position's collateral is zero (lines 379–383), `order.status = OrderStatus.Cancelled` only mutates the in-memory `order` struct and never persists the cancellation to storage.

Because the order status remains `OrderStatus.Active` in storage and position debt/collateral state updates in `PosManager` only happen at lines 394–396, an external call hook (such as an ERC777/callback-enabled token transfer to `order.recipient` at line 391 or from `msg.sender` at line 389) allows reentering `MarginTradingHook` mid-settlement:

- **Recursive execution before order finalization**: Reentering `fillOrder(_orderId)` before line 398 passes the `order.status == OrderStatus.Active` check at line 374 again, allowing a partial order (where `order.collAmt < totalCollAmt`) to be filled multiple times.
- **Cross-order reentrancy**: Reentering `fillOrder` for another active order on the same position before lines 394–396 causes `_calculateRepaySize` (lines 590–599) to compute `repayShares` and `order.collAmt` against stale `totalCollAmt` and `totalDebtShares` values.
- **Cross-function position-state mutation**: During the transfer of `order.tokenOut` to `order.recipient` at line 391, `msg.sender` has already transferred `repayAmt` of `borrToken` into `MarginTradingHook` (line 389). If `order.recipient` reenters `repayDebt` (line 229) or `InitCore.repay` to pay down the position's debt shares before line 394 executes, `InitCore._repay` (line 546) caps `sharesToRepay` to the remaining `positionDebtShares`. The excess `borrToken` transferred by `msg.sender` remains stranded in `MarginTradingHook` and can be swept via `removeCollateral` (line 222) or `reducePos` (line 327), both of which transfer `IERC20(token).balanceOf(address(this))` to their caller.

### Assumptions and preconditions

1. A margin trading position has been created in `MarginTradingHook` with at least one active stop-loss or take-profit order whose trigger price condition is met.
2. Either `order.tokenOut` or `borrToken` supports transfer callbacks (e.g., ERC777 or a token with receiver hooks), or execution control is handed to an untrusted contract during token transfer interactions.

### Proof of concept

1. A position owner (`order.recipient`, a contract) opens a margin position with 10 WETH collateral and 10,000 USDT debt, and places a take-profit or stop-loss order for 50% of the collateral (`order.collAmt = 5 WETH`).
2. When the oracle mark price reaches `triggerPrice_e36`, an executor calls `fillOrder(_orderId)`.
3. `_calculateFillOrderInfo` (line 387) calculates `repayShares` and `repayAmt` (5,000 USDT) and `amtOut` based on the initial position state.
4. At line 389, `MarginTradingHook` pulls 5,000 USDT (`repayAmt`) from the executor into `address(this)`.
5. At line 391, `MarginTradingHook` transfers `amtOut` of `order.tokenOut` to `order.recipient`.
6. Using the token transfer callback, `order.recipient` (or `msg.sender` in a recursive attack) reenters `MarginTradingHook`:
   - **Case 1 (Recursive order fill)**: `msg.sender` reenters `fillOrder(_orderId)`. Because `__orders[_orderId].status` is still `OrderStatus.Active` in storage, the order is filled a second time, draining the remaining 50% of the position's collateral at the order's limit price even though the user only authorized selling 50%.
   - **Case 2 (Cross-function state mutation)**: `order.recipient` calls `repayDebt` (or `InitCore.repay`) to repay the position's 10,000 USDT debt. When the outer `fillOrder` continues to line 394 (`IInitCore(CORE).repay`), `positionDebtShares` is now 0, so `InitCore._repay` transfers 0 USDT from `MarginTradingHook`, while line 396 still decollateralizes 5 WETH to `msg.sender`. `order.recipient` then calls `removeCollateral` or `reducePos` with `_tokenOut = USDT` to sweep the 5,000 USDT left inside `MarginTradingHook`.

### Impact assessment

- **Severity**: High
- **Technical Impact**: An attacker can bypass single-fill order semantics to fill partial orders multiple times, execute multiple orders against stale position debt/collateral ratios, or mutate position state mid-settlement to strand and sweep the executor's repayment tokens.
- **Business Impact**: Direct loss of user collateral or executor funds during stop-loss and take-profit order execution.

### Remediation

Add reentrancy protection (`ReentrancyGuardUpgradeable.nonReentrant`) to state-changing functions in `MarginTradingHook` (`openPos`, `increasePos`, `addCollateral`, `removeCollateral`, `repayDebt`, `reducePos`, `cancelOrder`, `updateOrder`, and `fillOrder`), load `order` as a `storage` pointer or persist status updates prior to external calls following the Checks-Effects-Interactions pattern, and ensure empty-position order cancellations persist to storage.

```solidity
    /// @inheritdoc IMarginTradingHook
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
        // update order status before external interactions
        __orders[_orderId].status = OrderStatus.Filled;
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
        emit FillOrder(order.initPosId, _orderId, order.tokenOut, amtOut);
    }
```

## Finding 3: Transient Memory Update in `MarginTradingHook.fillOrder` Fails to Persist `Cancelled` Order Status

### Synopsis

In `MarginTradingHook.fillOrder`, the order struct is loaded into `memory` rather than `storage`. When a position has zero collateral, `fillOrder` sets `order.status = OrderStatus.Cancelled` on the transient memory copy and returns, leaving the order `Active` in storage and fillable if the position is later replenished.

### Technical details

In `MarginTradingHook.fillOrder` (`contracts/hook/MarginTradingHook.sol`), the target order is loaded from the `__orders` mapping into a local `memory` variable on line 373:

```solidity
Order memory order = __orders[_orderId];
```

Subsequently, lines 379–383 check whether the margin position holds any collateral in `POS_MANAGER`. If the collateral amount is `0` (for example, after the position owner removes all collateral or closes the position), the function intends to cancel the order and exit early:

```solidity
if (IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool) == 0) {
    order.status = OrderStatus.Cancelled;
    emit CancelOrder(order.initPosId, _orderId);
    return;
}
```

Unlike line 398 (`__orders[_orderId].status = OrderStatus.Filled;`) and `cancelOrder` on line 364 (`Order storage order = __orders[_orderId];`), line 380 mutates `order.status` on the transient in-memory struct instead of the persistent storage slot `__orders[_orderId].status`.

As a result:
1. The transaction succeeds and emits a `CancelOrder(order.initPosId, _orderId)` event, leading off-chain indexers and the position owner to believe the order has been cancelled.
2. In contract storage, `__orders[_orderId].status` remains `OrderStatus.Active`.
3. If the position owner later reuses the same position by calling `addCollateral` or `increasePos`, the stale order remains active and eligible for execution via `fillOrder` once trigger price conditions are met, executing an unintended trade against the newly deposited collateral.
4. Furthermore, any caller can repeatedly invoke `fillOrder(_orderId)` while collateral is `0` to emit forged `CancelOrder` events for the same `_orderId`.

### Assumptions and preconditions

1. A position owner has opened a margin position via `openPos` and created an active stop-loss or take-profit order via `addStopLossOrder` or `addTakeProfitOrder`.
2. The position's collateral balance in `PosManager` becomes `0` (e.g., the owner calls `reducePos` or `removeCollateral` to withdraw all collateral).
3. `fillOrder` is called for that `_orderId` while the position has `0` collateral, or the user relies on the emitted `CancelOrder` event believing the order is no longer active and later adds collateral to the same position via `addCollateral` or `increasePos`.

### Proof of concept

1. Alice opens a long ETH/USDT margin position (`posId = 1`, `initPosId = 1`) via `MarginTradingHook.openPos` and creates a stop-loss order (`orderId = 1`) via `addStopLossOrder`.
2. Alice closes/reduces her position completely or removes all collateral so that `IPosManager(POS_MANAGER).getCollAmt(1, collPool) == 0`.
3. An executor (or keeper) calls `MarginTradingHook.fillOrder(1)`.
4. Inside `fillOrder(1)`:
   - Line 373 copies `__orders[1]` into memory (`Order memory order`).
   - Line 379 evaluates `getCollAmt(1, marginPos.collPool) == 0` as `true`.
   - Line 380 sets the memory variable `order.status = OrderStatus.Cancelled`.
   - Line 381 emits `CancelOrder(1, 1)` and line 382 returns.
5. Querying `hook.getOrder(1).status` still returns `OrderStatus.Active` because `__orders[1].status` in storage was never modified.
6. Alice later deposits new collateral into `posId = 1` using `increasePos` or `addCollateral`, assuming `orderId = 1` was already cancelled (as indicated by the `CancelOrder` event).
7. When the oracle price crosses `order.triggerPrice_e36`, an arbitrager calls `fillOrder(1)` and fills the stale order against Alice's new position collateral.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: Persistent storage state (`__orders[_orderId].status`) diverges from the intended state transition and the emitted `CancelOrder` event. Orders that should be permanently cancelled when a position is emptied remain `OrderStatus.Active`.
- **Business Impact**: Users who replenish an emptied position via `increasePos` or `addCollateral` are exposed to unexpected execution of stale stop-loss or take-profit orders at outdated trigger and limit prices, resulting in unintended position closure or unfavorable trade execution.

### Remediation

Load `order` as a `storage` pointer in `fillOrder` (or explicitly update `__orders[_orderId].status = OrderStatus.Cancelled;` in storage before returning).

```solidity
    /// @inheritdoc IMarginTradingHook
    function fillOrder(uint _orderId) external {
        Order storage order = __orders[_orderId];
        _require(order.status == OrderStatus.Active, Errors.INVALID_INPUT);
        MarginPos memory marginPos = __marginPositions[order.initPosId];
        address collToken = ILendingPool(marginPos.collPool).underlyingToken();
        address borrToken = ILendingPool(marginPos.borrPool).underlyingToken();
        // if position is empty, cancel order
        if (IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool) == 0) {
            order.status = OrderStatus.Cancelled;
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
        order.status = OrderStatus.Filled;
        emit FillOrder(order.initPosId, _orderId, order.tokenOut, amtOut);
    }
```

## Finding 4: Reversed Return Values from `_calculateRepaySize` in `MarginTradingHook.fillOrder` Cause Debt-Share and Underlying-Amount Confusion

### Synopsis

In `MarginTradingHook`, `_calculateRepaySize` returns `(repayAmt, repayShares)`, but `_calculateFillOrderInfo` (and `MarginTradingLens._fillOrder`) destructures the return values in reverse order as `(repayShares, repayAmt)`. Consequently, `fillOrder` treats debt shares as the underlying borrow token amount and vice versa, corrupting order payout calculations and causing order execution to revert or over-repay debt once interest accrues.

### Technical details

In `MarginTradingHook.sol`, the internal helper `_calculateRepaySize` (lines 590–599) declares its return parameters as `(uint repayAmt, uint repayShares)`:

```solidity
function _calculateRepaySize(Order memory _order, MarginPos memory _marginPos)
    internal
    returns (uint repayAmt, uint repayShares)
{
    uint totalCollAmt = IPosManager(POS_MANAGER).getCollAmt(_order.initPosId, _marginPos.collPool);
    if (_order.collAmt > totalCollAmt) _order.collAmt = totalCollAmt;
    uint totalDebtShares = IPosManager(POS_MANAGER).getPosDebtShares(_order.initPosId, _marginPos.borrPool);
    repayShares = totalDebtShares * _order.collAmt / totalCollAmt;
    repayAmt = ILendingPool(_marginPos.borrPool).debtShareToAmtCurrent(repayShares);
}
```

However, `_calculateFillOrderInfo` (lines 532–564) unpacks the tuple in the opposite order at line 536 (the same mistake is also present in `MarginTradingLens._fillOrder` at line 78):

```solidity
(repayShares, repayAmt) = _calculateRepaySize(_order, _marginPos);
```

As a result:
1. `repayShares` is assigned the underlying debt token amount (`repayAmt`), and `repayAmt` is assigned the debt share count (`repayShares`).
2. Inside `_calculateFillOrderInfo` (lines 539–563), `amtOut` is calculated by subtracting the debt value using `repayAmt` (which actually holds the smaller `repayShares` value after interest accrual), artificially inflating `amtOut` required from the order executor.
3. Inside `fillOrder` (lines 372–400):
   - Line 387 receives the swapped `repayShares` and `repayAmt`.
   - Line 389 transfers `repayAmt` (actually `repayShares`) of `borrToken` from `msg.sender` to `MarginTradingHook`.
   - Line 393 approves `repayAmt` (actually `repayShares`) of `borrToken` to `CORE`.
   - Line 394 calls `IInitCore(CORE).repay(marginPos.borrPool, repayShares, order.initPosId)`, passing the underlying amount in place of `_shares`.
4. In `InitCore._repay` (lines 545–551), `sharesToRepay` is capped at `positionDebtShares`, and `amtToRepay = ILendingPool(_pool).debtShareToAmtCurrent(sharesToRepay)` is computed and pulled from `MarginTradingHook` via `safeTransferFrom`. Because `amtToRepay` exceeds the `repayShares` amount of `borrToken` transferred into `MarginTradingHook` whenever borrow interest has accrued (`totalDebt > totalDebtShares`), `safeTransferFrom` reverts due to insufficient token balance or allowance.

### Assumptions and preconditions

1. A user has opened a margin trading position via `MarginTradingHook.openPos` and placed an active stop-loss or take-profit order.
2. Any non-zero amount of interest has accrued on the borrow `LendingPool` (either before the position was opened or while it is open) so that the pool's debt exchange rate is strictly greater than 1:1 (`totalDebt > totalDebtShares`).
3. The oracle mark price reaches the order's `triggerPrice_e36` so that `fillOrder` can be called.

### Proof of concept

1. Suppose `borrPool` (e.g., USDT pool) has accrued interest such that `totalDebt = 1.1e18` and `totalDebtShares = 1.0e18` (1 debt share = 1.1 USDT).
2. Alice opens a long WETH / borrow USDT margin position with `totalCollAmt = 2 WETH` and `totalDebtShares = 1000e6` shares (representing `1100e6` USDT of debt), and creates a stop-loss order for the full position (`collAmt = 2 WETH`, `limitPrice_e36 = 1500e36`, `tokenOut = WETH`).
3. When the trigger price condition is met, an executor calls `fillOrder(orderId)`.
4. `fillOrder` calls `_calculateFillOrderInfo`, which calls `_calculateRepaySize`:
   - `_calculateRepaySize` computes `repayShares = 1000e6` and `repayAmt = 1100e6`.
   - `_calculateRepaySize` returns `(1100e6, 1000e6)`.
5. `_calculateFillOrderInfo` assigns `repayShares = 1100e6` and `repayAmt = 1000e6`:
   - `amtOut` is computed using `repayAmt = 1000e6` instead of `1100e6`, overcharging the executor by `100e6 / 1500` WETH.
6. In `fillOrder`:
   - Line 389 transfers only `repayAmt = 1000e6` USDT from the executor to `MarginTradingHook`.
   - Line 393 approves `CORE` for `repayAmt` (`1000e6` if previous allowance was 0, or `type(uint).max`).
   - Line 394 calls `CORE.repay(borrPool, 1100e6, initPosId)`.
7. `InitCore._repay` caps `sharesToRepay` at `positionDebtShares = 1000e6`, computes `amtToRepay = debtShareToAmtCurrent(1000e6) = 1100e6` USDT, and attempts to transfer `1100e6` USDT from `MarginTradingHook` to `borrPool`.
8. Because `MarginTradingHook` only holds `1000e6` USDT, the transfer reverts, permanently blocking stop-loss and take-profit order execution.

### Impact assessment

- **Severity**: High
- **Direct Technical Impact**: Stop-loss and take-profit orders in `MarginTradingHook.fillOrder` (as well as quotes in `MarginTradingLens._fillOrder`) swap debt shares and underlying debt amounts. Whenever the borrow pool's debt share exchange rate exceeds 1:1 due to accrued interest, `MarginTradingHook` pulls fewer borrow tokens from the executor than `InitCore.repay` requires, causing `fillOrder` to revert. Even if leftover tokens existed in the hook, partial order fills would repay more debt shares than proportional to the order's collateral size and miscalculate `amtOut`.
- **Business Impact**: Core stop-loss and take-profit functionality fails in production markets where lending pools have accrued interest, leaving margin traders unable to have their positions automatically closed to cap losses or lock in profits, potentially leading to unexpected liquidations and user fund loss.

### Remediation

Update `_calculateRepaySize` in `MarginTradingHook.sol` to return `(uint repayShares, uint repayAmt)` so that it matches the destructuring order in `_calculateFillOrderInfo` (or alternatively swap the destructuring order in `_calculateFillOrderInfo` and `MarginTradingLens._fillOrder`):

```solidity
    /// @notice if the specified order size is larger than the position's collateral, repay size is scaled proportionally.
    /// @dev calculate repay size of the given margin order
    /// @param _order order info
    /// @param _marginPos margin position info
    /// @return repayShares repay shares
    /// @return repayAmt repay amount
    function _calculateRepaySize(Order memory _order, MarginPos memory _marginPos)
        internal
        returns (uint repayShares, uint repayAmt)
    {
        uint totalCollAmt = IPosManager(POS_MANAGER).getCollAmt(_order.initPosId, _marginPos.collPool);
        if (_order.collAmt > totalCollAmt) _order.collAmt = totalCollAmt;
        uint totalDebtShares = IPosManager(POS_MANAGER).getPosDebtShares(_order.initPosId, _marginPos.borrPool);
        repayShares = totalDebtShares * _order.collAmt / totalCollAmt;
        repayAmt = ILendingPool(_marginPos.borrPool).debtShareToAmtCurrent(repayShares);
    }
```

## Finding 5: Missing `_tokenOut` validation in `MarginTradingHook.updateOrder` allows arbitrary settlement tokens in orders

### Synopsis

`MarginTradingHook.updateOrder` omits the validation ensuring `_tokenOut` is either the margin position's `baseAsset` or `quoteAsset`, diverging from `_createOrder`. A user can update an active order's `tokenOut` to an arbitrary ERC20 address, breaking settlement calculation assumptions in `_calculateFillOrderInfo` and `fillOrder`.

### Technical details

When creating a stop-loss or take-profit order in `MarginTradingHook._createOrder` (line 483), the contract enforces that the desired output token (`_tokenOut`) belongs to the position's trading pair:

```solidity
_require(_tokenOut == marginPos.baseAsset || _tokenOut == marginPos.quoteAsset, Errors.INVALID_INPUT);
```

This invariant is required because `MarginTradingHook._calculateFillOrderInfo` (lines 539–563) computes `amtOut` under the binary assumption that `_order.tokenOut` is either `_collToken` (`_collToken == _order.tokenOut`) or the borrow token (the `else` branch, where `amtOut` is calculated in borrow token units by subtracting `repayAmt`).

However, in `MarginTradingHook.updateOrder` (lines 504–526), this check on `_tokenOut` is completely omitted:

```solidity
MarginPos memory marginPos = __marginPositions[initPosId];
uint collAmt = IPosManager(POS_MANAGER).getCollAmt(initPosId, marginPos.collPool);
_require(_collAmt <= collAmt, Errors.INPUT_TOO_HIGH);

order.triggerPrice_e36 = _triggerPrice_e36;
order.limitPrice_e36 = _limitPrice_e36;
order.collAmt = _collAmt;
order.tokenOut = _tokenOut;
```

Consequently, after creating a valid order, the order creator can call `updateOrder` and set `order.tokenOut` to any arbitrary address:
1. During `fillOrder`, `_calculateFillOrderInfo` takes the `else` branch whenever `_order.tokenOut != _collToken`, computing `amtOut` denominated in the position's `borrToken`.
2. Line 391 (`IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut)`) then attempts to transfer `amtOut` of the arbitrary `order.tokenOut` from the executor (`msg.sender`) to `order.recipient`.
3. If `order.tokenOut` is set to a higher-value token that an executor/solver contract has approved to `MarginTradingHook`, the executor will transfer `amtOut` units of the more valuable token while only receiving the position's collateral. Conversely, if `order.tokenOut` is set to a non-existent or unapproved token, the order becomes unfillable and reverts in `fillOrder`.

### Assumptions and preconditions

1. A user has an open margin position in `MarginTradingHook` and an active stop-loss or take-profit order.
2. For token draining of an order executor, the executor (arbitrager/keeper) holds and has approved `MarginTradingHook` to spend another ERC20 token (e.g., WBTC or WETH used across other margin pairs) and executes `fillOrder`.

### Proof of concept

1. Alice opens a Long ETH-USDC margin position (`baseAsset = WETH`, `quoteAsset = USDC`, `collToken = WETH`, `borrToken = USDC`) and calls `addTakeProfitOrder` with `_tokenOut = USDC`.
2. Alice calls `updateOrder` for her active order, passing `_tokenOut = WBTC` (an asset with much higher unit price or different decimals that keeper bots have approved to `MarginTradingHook` for filling BTC positions).
3. Because `updateOrder` does not validate `_tokenOut`, `order.tokenOut` is updated to `WBTC`.
4. When the trigger price is reached and a keeper calls `fillOrder`, `_calculateFillOrderInfo` sees `_collToken (WETH) != order.tokenOut (WBTC)` and enters the `else` branch (lines 551–556), calculating `amtOut` as a quantity of USDC (e.g., `1500e6`).
5. `fillOrder` executes `IERC20(WBTC).safeTransferFrom(msg.sender, order.recipient, 1500e6)`, transferring `1500e6` units of WBTC (15 WBTC, worth ratusan of thousands of dollars) instead of 1,500 USDC from the keeper to Alice.

### Impact assessment

- **Severity**: Medium / High
- **Direct Technical Effect**: Violation of the order settlement-asset invariant (`order.tokenOut` must be `baseAsset` or `quoteAsset`), causing `_calculateFillOrderInfo` to compute `amtOut` in `borrToken` terms while `fillOrder` transfers an arbitrary `order.tokenOut`.
- **Business Implications**: Order executors/keepers that approve multiple tokens to `MarginTradingHook` can lose funds by transferring high-value tokens at the numerical amount of a lower-value borrow token, or orders can be bricked by setting invalid `tokenOut` addresses.

### Remediation

Validate in `updateOrder` that `_tokenOut` is either `marginPos.baseAsset` or `marginPos.quoteAsset`, matching the validation performed in `_createOrder`.

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

## Finding 6: Overlapping Active Order Collateral Commitments in `MarginTradingHook._createOrder` and `MarginTradingHook.updateOrder` Allow Unintended Position Liquidation

### Synopsis

`MarginTradingHook._createOrder` and `MarginTradingHook.updateOrder` validate an order's `_collAmt` solely against the position's total collateral without accounting for collateral already committed to other active orders on the same `initPosId`. This allows overlapping active orders to be sequentially filled, liquidating more collateral than intended or executing stale orders after collateral additions.

### Technical details

When a user creates a stop-loss or take-profit order via `addStopLossOrder` or `addTakeProfitOrder`, `MarginTradingHook._createOrder` checks whether the requested `_collAmt` does not exceed the position's current total collateral held in `PosManager`:

```solidity
uint collAmt = IPosManager(POS_MANAGER).getCollAmt(initPosId, marginPos.collPool);
_require(_collAmt <= collAmt, Errors.INPUT_TOO_HIGH); // check specified coll amt is feasible
```

It then creates a new `Order` struct with `status: OrderStatus.Active` and `collAmt: _collAmt` (lines 487–498) and appends the new `orderId` to `__posOrderIds[initPosId]` (line 499). Similarly, `MarginTradingHook.updateOrder` (lines 504–526) checks `_require(_collAmt <= collAmt, Errors.INPUT_TOO_HIGH)` only against the standalone `_collAmt` parameter.

This design has two interconnected flaws in how position-bound order commitments are managed:

1. **Unbounded Overlapping Collateral Commitments:** Neither `_createOrder` nor `updateOrder` sums the `collAmt` of existing `OrderStatus.Active` orders in `__posOrderIds[initPosId]` (such as active orders of the same `OrderType`), nor do they cancel or replace prior active orders when a new commitment is created. Multiple active orders can therefore commit more than 100% of the position's collateral.
2. **Silent Clamping and Persistent Active Status During Execution:** In `fillOrder` (lines 372–400), `_calculateRepaySize` (lines 590–599) does not revert if an order's `collAmt` exceeds the remaining position collateral after an earlier order was filled; instead, it clamps `_order.collAmt` to `totalCollAmt`:
   ```solidity
   uint totalCollAmt = IPosManager(POS_MANAGER).getCollAmt(_order.initPosId, _marginPos.collPool);
   if (_order.collAmt > totalCollAmt) _order.collAmt = totalCollAmt;
   ```
   Furthermore, if an earlier order consumes all collateral (`totalCollAmt == 0`), a subsequent call to `fillOrder` on a remaining active order executes lines 379–383:
   ```solidity
   if (IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool) == 0) {
       order.status = OrderStatus.Cancelled;
       emit CancelOrder(order.initPosId, _orderId);
       return;
   }
   ```
   Because `order` in `fillOrder` is loaded into `memory` (`Order memory order = __orders[_orderId]`), setting `order.status = OrderStatus.Cancelled` only mutates memory and never updates `__orders[_orderId].status` in storage. If the user later deposits additional collateral into the same position (`initPosId`), the leftover active order in `__posOrderIds[initPosId]` becomes executable again and can be filled against the newly added collateral.

### Assumptions and preconditions

1. A user has an open margin trading position (`initPosId`) with non-zero collateral in `PosManager`.
2. The user creates more than one order on the position (for example, multiple partial orders, or both stop-loss and take-profit orders, or creates a new order without manually cancelling a prior order), or updates an existing order via `updateOrder`.
3. Market prices reach the trigger price condition required by `_validateTriggerPrice`.

### Proof of concept

1. Alice opens a long ETH/USDT margin position (`posId = 1`, `initPosId = 1`) holding `10 WETH` of collateral shares in `marginPos.collPool`.
2. Alice calls `addTakeProfitOrder` (`_createOrder`) with `_collAmt = 6 WETH` (`orderId = 1`). The check `_collAmt <= 10 WETH` passes, and `orderId = 1` is pushed to `__posOrderIds[1]` as `OrderStatus.Active`.
3. Alice later calls `addTakeProfitOrder` (`_createOrder`) with updated price targets and `_collAmt = 6 WETH` (`orderId = 2`), expecting at most `6 WETH` to be closed or forgetting to cancel `orderId = 1`. Because `_createOrder` checks `6 WETH <= 10 WETH` without summing active orders in `__posOrderIds[1]`, `orderId = 2` is also marked `OrderStatus.Active`.
4. When the mark price reaches both trigger prices, an executor calls `fillOrder(1)`, which repays 60% of the position's debt and decollateralizes `6 WETH`, leaving `4 WETH` of collateral in the position.
5. The executor immediately calls `fillOrder(2)`. In `_calculateRepaySize`, `order.collAmt` (`6 WETH`) exceeds `totalCollAmt` (`4 WETH`), so it is clamped to `4 WETH`. The remaining `4 WETH` is decollateralized and 100% of Alice's position is closed instead of the intended partial amount.
6. Alternatively, if `orderId = 1` committed all `10 WETH` and was filled (reducing position collateral to `0`), `orderId = 2` remains `OrderStatus.Active` in storage even if `fillOrder(2)` is called while collateral is `0`. When Alice later adds new collateral to `posId = 1`, an executor can fill the stale `orderId = 2` against her new collateral.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:** Active collateral commitments across `__posOrderIds[initPosId]` can exceed the position's actual collateral balance, and uncancelled orders remain active in storage across collateral depletion and replenishment cycles.
- **Business Implications:** Users can suffer unintended full position liquidation from overlapping partial orders or unexpected execution of stale orders after adding new collateral or increasing their position at unfavourable limit prices.

### Remediation

Track the total collateral committed across active orders of the same type (or require at most one active stop-loss and one active take-profit order per position) and ensure that total committed collateral in `_createOrder` and `updateOrder` does not exceed the position's collateral. Additionally, ensure storage is updated when cancelling orders on empty positions.

In `MarginTradingHook._createOrder` and `MarginTradingHook.updateOrder`:

```solidity
    function _createOrder(
        uint _posId,
        uint _triggerPrice_e36,
        address _tokenOut,
        uint _limitPrice_e36,
        uint _collAmt,
        OrderType _orderType
    ) internal returns (uint orderId) {
        orderId = ++lastOrderId;
        _require(_collAmt != 0, Errors.ZERO_VALUE);
        uint initPosId = initPosIds[msg.sender][_posId];
        _require(initPosId != 0, Errors.POSITION_NOT_FOUND);
        MarginPos memory marginPos = __marginPositions[initPosId];
        _require(_tokenOut == marginPos.baseAsset || _tokenOut == marginPos.quoteAsset, Errors.INVALID_INPUT);
        uint collAmt = IPosManager(POS_MANAGER).getCollAmt(initPosId, marginPos.collPool);
        uint committedCollAmt = _getCommittedCollAmt(initPosId, _orderType, 0);
        _require(committedCollAmt + _collAmt <= collAmt, Errors.INPUT_TOO_HIGH);

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
        uint committedCollAmt = _getCommittedCollAmt(initPosId, order.orderType, _orderId);
        _require(committedCollAmt + _collAmt <= collAmt, Errors.INPUT_TOO_HIGH);

        order.triggerPrice_e36 = _triggerPrice_e36;
        order.limitPrice_e36 = _limitPrice_e36;
        order.collAmt = _collAmt;
        order.tokenOut = _tokenOut;
        emit UpdateOrder(initPosId, _orderId, _tokenOut, _triggerPrice_e36, _limitPrice_e36, _collAmt);
    }

    function _getCommittedCollAmt(uint _initPosId, OrderType _orderType, uint _excludeOrderId)
        internal
        view
        returns (uint committedAmt)
    {
        uint[] memory orderIds = __posOrderIds[_initPosId];
        for (uint i = 0; i < orderIds.length; ++i) {
            uint id = orderIds[i];
            if (id == _excludeOrderId) continue;
            Order storage existingOrder = __orders[id];
            if (existingOrder.status == OrderStatus.Active && existingOrder.orderType == _orderType) {
                committedAmt += existingOrder.collAmt;
            }
        }
    }
```

## Finding 7: Missing Order Ownership Verification in `MarginTradingHook.updateOrder` Allows Arbitrary Order Modification and Theft of Position Collateral

### Synopsis

`MarginTradingHook.updateOrder` verifies that the caller owns a valid position (`_posId`), but fails to verify that the target `_orderId` actually belongs to that position (`order.initPosId == initPosId`). Consequently, any user with a margin position can overwrite another user's active order parameters and drain their position's net equity via `fillOrder`.

### Technical details

In `MarginTradingHook.sol`, each margin trading position owned by a user is mapped via `initPosIds[msg.sender][_posId]` to an underlying `initPosId`. When a user creates a stop-loss or take-profit order in `_createOrder` (lines 470–501), the created `Order` struct records `initPosId: initPosId` (line 489) and `recipient: msg.sender` (line 496) in `__orders[orderId]`.

When cancelling an order in `cancelOrder` (lines 361–369), the contract explicitly checks that the order belongs to the caller's position:
```solidity
uint initPosId = initPosIds[msg.sender][_posId];
_require(initPosId != 0, Errors.POSITION_NOT_FOUND);
Order storage order = __orders[_orderId];
_require(order.initPosId == initPosId, Errors.INVALID_INPUT);
```

However, in `updateOrder` (lines 504–526), this ownership binding check is completely omitted:
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

Because `updateOrder` never verifies `order.initPosId == initPosId`, an attacker who owns any position (`_posId`) with sufficient collateral (`collAmt >= _collAmt`) can pass their own `_posId` alongside a victim's active `_orderId`. The validation checks pass against the attacker's `initPosId`, while the state updates overwrite the victim's `Order` storage record (`triggerPrice_e36`, `limitPrice_e36`, `collAmt`, and `tokenOut`).

By setting `_triggerPrice_e36` so that `_validateTriggerPrice` immediately passes at the current oracle mark price, and setting `_limitPrice_e36` to a value vastly unfavorable to the victim (such that `amtOut` calculated in `_calculateFillOrderInfo` is `0` or near zero), the attacker can immediately call `fillOrder(_orderId)` to repay only the victim's debt and seize the victim's collateral shares via `IInitCore(CORE).decollateralize`.

### Assumptions and preconditions

1. A victim has an open margin position and an `Active` stop-loss or take-profit order (`_orderId`).
2. The attacker opens (or already holds) a margin position in `MarginTradingHook` with collateral shares at least equal to the `_collAmt` they wish to set on the victim's order (which can also be funded transiently via a flashloan).

### Proof of concept

1. Alice opens a long WETH/USDT margin position (`initPosId = 1`, `posId = 1`) depositing collateral worth 2 WETH (at 2,000 USDT/WETH = 4,000 USDT) and borrowing 2,000 USDT.
2. Alice creates a take-profit order (`orderId = 1`) with `triggerPrice_e36 = 2500e36`, `limitPrice_e36 = 2500e36`, `tokenOut = WETH`, and `collAmt = fullCollShares`.
3. Bob opens a margin position (`initPosId = 2`, `posId = 1`) with at least `fullCollShares` of collateral.
4. While the mark price is still `2000e36`, Bob calls `updateOrder(1, 1, 2000e36, WETH, 1000e36, fullCollShares)`:
   - `initPosIds[BOB][1]` resolves to `initPosId = 2` (non-zero).
   - `_collAmt <= collAmt` passes against Bob's position (`initPosId = 2`).
   - Alice's order (`orderId = 1`) is overwritten with `triggerPrice_e36 = 2000e36` and `limitPrice_e36 = 1000e36`.
5. Bob immediately calls `fillOrder(1)`:
   - `_validateTriggerPrice` checks `markPrice_e36 (2000e36) >= triggerPrice_e36 (2000e36)`, which succeeds.
   - `_calculateFillOrderInfo` computes `amtOut = collTokenAmt - repayAmt * 1e36 / limitPrice_e36 = 2 WETH - 2000 / 1000 WETH = 0`.
   - Bob repays Alice's 2,000 USDT debt, pays `0` WETH to Alice, and receives all of Alice's 2 WETH collateral shares (worth 4,000 USDT), stealing Alice's 2,000 USDT net equity.

### Impact assessment

- **Severity:** High / Critical
- **Technical Impact:** Unauthorized modification of any active order's trigger price, limit price, collateral size, and output token.
- **Business Impact:** Direct theft of users' margin position collateral (net equity) by forcing premature order execution at manipulated limit prices, or denial of service against users' stop-loss/take-profit protection by shrinking `collAmt` to `1` wei or setting unreachable trigger prices.

### Remediation

In `MarginTradingHook.updateOrder`, validate that `order.initPosId` matches the caller's `initPosId` (and ensure `_tokenOut` is either the position's `baseAsset` or `quoteAsset`, consistent with `_createOrder`).

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
