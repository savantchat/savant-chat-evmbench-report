# Audit report — 2024-01-init-capital-invitational/H-03

## Finding 1: Swapped Return Values in `MarginTradingHook._calculateRepaySize` and `MarginTradingHook._calculateFillOrderInfo` Cause Share-Versus-Underlying Accounting Mismatch in `fillOrder`

### Synopsis

In `MarginTradingHook`, `_calculateRepaySize` returns `(uint repayAmt, uint repayShares)`, whereas `_calculateFillOrderInfo` (and `MarginTradingLens._fillOrder`) destructures the return tuple as `(repayShares, repayAmt)`. This swaps debt shares and underlying debt amounts, corrupting `amtOut` calculations and causing `fillOrder` repayments to revert or miscalculate amounts once interest accrues.

### Technical details

In `contracts/hook/MarginTradingHook.sol`, `_calculateRepaySize` is declared with the return parameters `(uint repayAmt, uint repayShares)`:

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

However, in `_calculateFillOrderInfo` (line 536) and `MarginTradingLens._fillOrder` (line 78), the caller unpacks the return tuple in the opposite order:

```solidity
(repayShares, repayAmt) = _calculateRepaySize(_order, _marginPos);
```

As a result:
1. `repayShares` is assigned the underlying token debt amount (`repayAmt`).
2. `repayAmt` is assigned the debt share count (`repayShares`).

In `_calculateFillOrderInfo` (lines 544, 549, 556, and 561), `amtOut` is computed by subtracting the value of `repayAmt` from the collateral value at `_order.limitPrice_e36`. Because `repayAmt` holds debt shares rather than the underlying token amount (which is larger than debt shares once interest accrues in the lending pool), the debt deduction is understated and `amtOut` is overstated across all four branches:
- `_collToken == _order.tokenOut && _marginPos.isLongBaseAsset` (line 544)
- `_collToken == _order.tokenOut && !_marginPos.isLongBaseAsset` (line 549)
- `_collToken != _order.tokenOut && _marginPos.isLongBaseAsset` (line 556)
- `_collToken != _order.tokenOut && !_marginPos.isLongBaseAsset` (line 561)

Furthermore, in `fillOrder` (lines 387–394):
- Line 389 transfers `repayAmt` (actually the smaller `repayShares` value) of `borrToken` from `msg.sender` to `MarginTradingHook`.
- Line 394 calls `IInitCore(CORE).repay(marginPos.borrPool, repayShares, order.initPosId)`, passing `repayShares` (actually the larger underlying `repayAmt` value) as the `_shares` argument.
- Inside `InitCore._repay`, the core contract computes `amtToRepay = ILendingPool(_pool).debtShareToAmtCurrent(sharesToRepay)` and attempts to pull `amtToRepay` from `MarginTradingHook`. Because `amtToRepay > repayShares` after any interest accrual, `MarginTradingHook` does not hold enough `borrToken` (unless leftover tokens exist in the contract), causing `fillOrder` to revert.

### Assumptions and preconditions

1. A margin trading position has been opened with borrowed assets in `marginPos.borrPool`, and a stop-loss or take-profit order has been placed.
2. Time passes and interest accrues on `marginPos.borrPool` such that `totalDebt > totalDebtShares` (so `repayAmt > repayShares`).

### Proof of concept

1. Alice opens a short ETH / long USDC margin position (or long ETH / short USDC position) borrowing `1e18` WETH when `totalDebt == totalDebtShares`, receiving `1e18` debt shares, and places a stop-loss or take-profit order for 100% of the collateral.
2. Time elapses and borrow interest accrues in the WETH `LendingPool` so that `1e18` debt shares now correspond to `1.1e18` WETH of underlying debt (`debtShareToAmtCurrent(1e18) == 1.1e18`).
3. When the trigger price is reached, an executor Bob calls `fillOrder(orderId)`.
4. `_calculateRepaySize` computes `repayShares = 1e18` and `repayAmt = 1.1e18`, returning `(1.1e18, 1e18)`.
5. `_calculateFillOrderInfo` assigns `repayShares = 1.1e18` and `repayAmt = 1e18`.
6. In `_calculateFillOrderInfo` (lines 544–561), `amtOut` is calculated using `repayAmt = 1e18` instead of `1.1e18`, overcharging Bob by `0.1e18` WETH worth of `order.tokenOut`.
7. In `fillOrder`, line 389 transfers only `repayAmt = 1e18` WETH from Bob to `MarginTradingHook`, while line 394 calls `IInitCore(CORE).repay(borrPool, 1.1e18, initPosId)`.
8. `InitCore._repay` caps `sharesToRepay` at the position's `1e18` debt shares and attempts to pull `debtShareToAmtCurrent(1e18) = 1.1e18` WETH from `MarginTradingHook`, which only holds `1e18` WETH, causing the transaction to revert due to insufficient token balance.

### Impact assessment

- **Severity:** High
- **Technical Effect:** `amtOut`, `repayAmt`, and `repayShares` are miscalculated whenever `totalDebt != totalDebtShares` in `borrPool`. `fillOrder` reverts due to insufficient `borrToken` transferred into `MarginTradingHook`, or overcharges the order filler for `amtOut` and repays an unintended number of debt shares on partial fills.
- **Business Impact:** Core stop-loss and take-profit order execution in `MarginTradingHook` breaks for any position whose borrow pool has accrued interest, preventing users' stop-loss orders from being filled when market prices move against them and exposing positions to liquidation.

### Remediation

Update the return tuple order in `_calculateRepaySize` so that it matches `(uint repayShares, uint repayAmt)` as expected by `_calculateFillOrderInfo` and `MarginTradingLens._fillOrder` (or reorder the destructuring assignment at the call sites).

```solidity
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

## Finding 2: Unsafe MarginPos storage upgrade corrupts asset roles used by MarginTradingHook.fillOrder

### Synopsis

MarginTradingHook’s upgraded `MarginPos` layout inserts `baseAsset` and `quoteAsset` ahead of the existing direction flag. Existing mapping entries are decoded with corrupted asset roles, causing `fillOrder` to revert, validate inverted triggers, or settle legacy stop-loss and take-profit orders using the wrong direction.

### Technical details

`MarginTradingHook` stores `MarginPos` values in the persistent `__marginPositions` mapping. In the prior layout, `isLongBaseAsset` occupied the third struct storage slot after `collPool` and `borrPool`. The highlighted layout places two new addresses before that flag:

- Slot 0: `collPool`
- Slot 1: `borrPool`
- Slot 2: `baseAsset`
- Slot 3: packed `quoteAsset` and `isLongBaseAsset`

Solidity does not transform mapping values during an implementation upgrade. Consequently, a legacy long position whose third slot contains the Boolean value `1` is decoded as:

- `baseAsset = address(1)`
- `quoteAsset = address(0)`
- `isLongBaseAsset = false`

A legacy short position generally decodes all three new fields as zero or false. Thus, neither position preserves the required relationships:

- Base and quote assets must be the distinct underlyings of the collateral and borrowing pools.
- `isLongBaseAsset` must agree with whether the collateral underlying is the base asset and the borrowed underlying is the quote asset.

The corrupted addresses are subsequently used to obtain oracle prices, while the corrupted direction flag selects trigger inequalities and settlement formulas. An unsupported zero or `address(1)` oracle asset normally causes every legacy order fill to revert. If the oracle returns usable values for either address, a legacy long position can instead be processed as short, permitting trigger checks and payment calculations under the wrong economic direction.

Even if the new fields were appended after the old Boolean, legacy entries would initially contain zero base and quote addresses. Therefore, preserving field order and migrating or lazily initializing all existing positions are both required.

### Assumptions and preconditions

- A previous deployed implementation stored `isLongBaseAsset` directly after `collPool` and `borrPool`.
- The implementation is upgraded while the `__marginPositions` mapping contains existing positions.
- The upgrade does not migrate or version every existing mapping entry before asset-role-dependent operations are enabled.
- At least one legacy position has an active order or later creates an order using its corrupted metadata.
- The upgrade is performed honestly as part of normal protocol maintenance; no administrator compromise or malicious upgrade is required.

### Proof of concept

1. Under the previous implementation, open a long base-asset position and create an active stop-loss order.
2. Observe that its mapping value stores `collPool` in slot 0, `borrPool` in slot 1, and `true` as value `1` in slot 2.
3. Upgrade the proxy to the implementation using the highlighted `MarginPos` layout without migrating the position.
4. Call `getMarginPos` for the legacy position.
5. Observe that `collPool` and `borrPool` remain correct, but `baseAsset` is `address(1)`, `quoteAsset` is zero, and `isLongBaseAsset` is false.
6. Move the real market price beyond the stop-loss trigger and call `fillOrder`.
7. The call either reverts during oracle pricing of the malformed assets or processes the long position using short-position trigger and settlement branches.
8. The order cannot provide the intended stop-loss protection, leaving the leveraged position exposed to further losses and liquidation.

### Impact assessment

**Severity: High.**

A routine upgrade can corrupt the economic identity of every pre-existing margin position. Active stop-loss and take-profit orders can become permanently unfillable, potentially leaving leveraged users exposed until liquidation. Where malformed oracle lookups return values rather than reverting, orders may be triggered and settled using inverted long/short logic and incorrect payment amounts.

The issue is systemic because all legacy mapping entries share the incompatible layout. After the honest upgrade, no privileged access is needed to call the public order-filling path. The worst case includes widespread failure of protective orders, incorrect collateral transfers, user losses, emergency upgrade requirements, and significant loss of confidence in the margin-trading product.

### Remediation

Preserve every existing member at its original slot and offset. Append new fields only after the legacy direction flag, and replace positional construction with named-field initialization:

```solidity
struct MarginPos {
    address collPool;
    address borrPool;
    bool isLongBaseAsset;
    address baseAsset;
    address quoteAsset;
}
```

```solidity
__marginPositions[initPosId] = MarginPos({
    collPool: _collPool,
    borrPool: _borrPool,
    isLongBaseAsset: isLongBaseAsset,
    baseAsset: baseToken,
    quoteAsset: quoteToken
});
```

Add a versioned migration or lazy initialization path for every legacy position. It must derive the two underlyings from the stored pools, obtain a nonzero and distinct base/quote assignment, verify the direction against the collateral and borrowing roles, and mark the position migrated before permitting order operations.

If the incompatible layout has already been deployed and used to create new positions, do not simply reorder the struct again. Introduce a new versioned mapping or dedicated migration representation so legacy and post-upgrade entries can be decoded according to their actual schema.

## Finding 3: Failure to Update Storage When Cancelling Empty Position Orders in `MarginTradingHook.fillOrder` Leaves Orders Active

### Synopsis

In `MarginTradingHook.fillOrder`, the target `Order` is loaded into `memory` rather than `storage`. When a position has zero collateral, setting `order.status = OrderStatus.Cancelled` only mutates the transient memory copy, leaving the order `Active` in storage for repeated execution or unexpected future fills.

### Technical details

In `contracts/hook/MarginTradingHook.sol`, `fillOrder(uint _orderId)` loads the order struct from the `__orders` mapping into memory on line 373:

```solidity
Order memory order = __orders[_orderId];
_require(order.status == OrderStatus.Active, Errors.INVALID_INPUT);
```

Next, on lines 379–383, `fillOrder` checks whether the position's collateral in `POS_MANAGER` has been depleted (`getCollAmt == 0`). If the position is empty, the function attempts to cancel the order and return early:

```solidity
if (IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool) == 0) {
    order.status = OrderStatus.Cancelled;
    emit CancelOrder(order.initPosId, _orderId);
    return;
}
```

Because `order` is declared with the `memory` data location on line 373, the assignment `order.status = OrderStatus.Cancelled` on line 380 only modifies the in-memory copy of the struct. Unlike line 398 (`__orders[_orderId].status = OrderStatus.Filled;`), which explicitly writes to storage, line 380 never updates `__orders[_orderId].status` in contract storage before returning on line 382.

Consequently:
1. `__orders[_orderId].status` remains `OrderStatus.Active` in storage even after `CancelOrder(order.initPosId, _orderId)` is emitted.
2. Any caller can invoke `fillOrder(_orderId)` repeatedly while the position collateral is zero, re-executing the cancellation branch and emitting duplicate `CancelOrder` events each time.
3. If the position owner subsequently replenishes collateral in the same position via `addCollateral` or `increasePos` (believing the old order was cancelled as indicated by the `CancelOrder` event), the stale order remains `OrderStatus.Active` and can be filled by an executor whenever its trigger price condition is met.

### Assumptions and preconditions

1. A user has opened a margin position via `MarginTradingHook.openPos` and created an active stop-loss or take-profit order via `addStopLossOrder` or `addTakeProfitOrder`.
2. The position's collateral balance in `POS_MANAGER` reaches zero (e.g., after the user removes collateral or closes/reduces the position).
3. `fillOrder(_orderId)` is called while the position collateral is zero.

### Proof of concept

1. Alice opens a margin position (`posId = 1`, `initPosId = 1`) and places a stop-loss or take-profit order (`orderId = 1`), which sets `__orders[1].status = OrderStatus.Active`.
2. Alice closes or removes all collateral from her position so that `IPosManager(POS_MANAGER).getCollAmt(1, marginPos.collPool) == 0`.
3. An executor (or keeper bot) calls `MarginTradingHook.fillOrder(1)`.
4. Inside `fillOrder(1)`:
   - Line 373 loads `__orders[1]` into `Order memory order`.
   - Line 379 evaluates `getCollAmt(1, marginPos.collPool) == 0` as `true`.
   - Line 380 sets `order.status = OrderStatus.Cancelled` in memory only.
   - Line 381 emits `CancelOrder(1, 1)` and line 382 returns.
5. Querying `hook.getOrder(1).status` still returns `OrderStatus.Active` (`1`).
6. Calling `MarginTradingHook.fillOrder(1)` again succeeds instead of reverting with `INVALID_INPUT` (`#203`), emitting another `CancelOrder(1, 1)` event.
7. Alice later reuses her position by calling `addCollateral` or `increasePos` on `posId = 1`. Because `__orders[1].status` is still `OrderStatus.Active`, an executor can call `fillOrder(1)` once the oracle price satisfies `order.triggerPrice_e36`, unexpectedly liquidating/closing Alice's newly funded position using the stale order parameters.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: State transition from `OrderStatus.Active` to `OrderStatus.Cancelled` is lost, corrupting order lifecycle accounting and emitting forged/duplicate `CancelOrder` events while keeping the order active in storage.
- **Business Impact**: Position owners who empty a position and observe a `CancelOrder` event (or expect automatic cancellation on empty positions) remain exposed to stale orders if they later add collateral or increase their position size, resulting in unintended order execution and financial loss.

### Remediation

Update the order's status directly in storage (`__orders[_orderId].status`) when cancelling an order for an empty position.

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
```

## Finding 4: Uncancelled and Unsynchronized Active Orders in `MarginTradingHook.fillOrder` Allow Execution of Stale or Conflicting Orders Against Shared Position Collateral

### Synopsis

In `MarginTradingHook.fillOrder`, active orders are neither invalidated when a position is modified or closed-and-reopened nor synchronized when other orders or manual position reductions consume shared collateral. Instead, `fillOrder` allows any active order to execute whenever `getCollAmt > 0`, silently clamping `order.collAmt` to remaining collateral.

### Technical details

In `MarginTradingHook`, a user can create multiple stop-loss or take-profit orders tied to the same `initPosId` via `addStopLossOrder` and `addTakeProfitOrder`. When an order is created in `_createOrder` (or modified in `updateOrder`), its `collAmt` is only validated against the position's current collateral balance at that moment (`_require(_collAmt <= collAmt, Errors.INPUT_TOO_HIGH)`), without tracking cumulative collateral committed across active orders in `__posOrderIds[initPosId]` and without invalidating orders when the position's collateral changes.

When `fillOrder` is called (`contracts/hook/MarginTradingHook.sol`, lines 373–388):
1. At line 379 (`if (IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool) == 0)`), `fillOrder` only checks whether the position's current collateral is zero at the exact moment `fillOrder` is invoked.
2. At line 387 (`_calculateFillOrderInfo(order, marginPos, collToken)`), execution delegates to `_calculateRepaySize` (lines 590–599), which checks `if (_order.collAmt > totalCollAmt) _order.collAmt = totalCollAmt;` (modifying `_order.collAmt` in memory by reference) and scales `repayShares` proportionally to `_order.collAmt / totalCollAmt`.

This creates two closely related order lifecycle integrity flaws during `fillOrder`:
- **Conflicting execution rights over shared collateral:** Multiple active orders on the same `initPosId` (or an active order combined with a manual partial `reducePos` / `removeCollateral` call by the user) compete for the same underlying collateral pool shares. When one order (or a manual reduction) consumes part of the position's collateral, other active orders are not reduced or cancelled. Because `getCollAmt` at line 379 remains non-zero and `_calculateFillOrderInfo` at line 387 clamps `order.collAmt` down to `totalCollAmt`, a second order (or an order that the user already manually satisfied via `reducePos`) can still be filled, draining the remaining collateral from the position.
- **Stale orders after position changes:** When a user closes their position completely via `reducePos` or `removeCollateral` (reducing collateral to `0`), active orders in `__posOrderIds[initPosId]` are not cancelled. If the user subsequently reuses the same `posId` (`initPosId`) by calling `increasePos` or `addCollateral`, `getCollAmt(order.initPosId, marginPos.collPool)` at line 379 becomes non-zero again, allowing the stale order from the prior position state to be filled via `fillOrder` against the newly deposited collateral.

### Assumptions and preconditions

1. A position owner opens a margin trading position via `MarginTradingHook.openPos` and creates one or more stop-loss or take-profit orders.
2. Either:
   - The position owner creates multiple orders whose combined `collAmt` exceeds the position's collateral (or manually reduces part of the position via `reducePos` / `removeCollateral` after creating a partial order), or
   - The position owner reduces the position's collateral to zero via `reducePos` / `removeCollateral` and later adds collateral or increases the position again under the same `posId` (`initPosId`) before `fillOrder` is called on the old order.
3. The oracle mark price satisfies `_validateTriggerPrice(order, marginPos)` at the time an executor calls `fillOrder`.

### Proof of concept

1. Alice opens a long WETH/USDT margin position (`posId = 1`, `initPosId = 1`) with `100` collateral shares (`collAmt = 100`).
2. Alice places a TakeProfit order (`orderId = 1`) for `60` collateral shares (`collAmt = 60`) at `triggerPrice = 2500`.
3. When the price approaches `2500`, Alice manually takes profit on `60` collateral shares by calling `reducePos(1, 60, ...)`, leaving `40` collateral shares in `initPosId = 1` intending to keep the remaining `40` shares open, or alternatively places a second order (`orderId = 2`) for `60` collateral shares that gets filled first.
4. When the mark price reaches `2500`, an arbitrageur calls `fillOrder(1)`:
   - At line 379, `IPosManager(POS_MANAGER).getCollAmt(1, marginPos.collPool)` returns `40` (`!= 0`).
   - At line 385, `_validateTriggerPrice` passes.
   - At line 387, `_calculateFillOrderInfo` calls `_calculateRepaySize`, which sees `_order.collAmt (60) > totalCollAmt (40)` and clamps `_order.collAmt` in memory to `40`.
   - `fillOrder` repays the remaining debt and decollateralizes all remaining `40` collateral shares to the executor, completely closing Alice's remaining position.
5. Similarly, if Alice had reduced all `100` collateral shares via `reducePos` (leaving `0` collateral) and later called `increasePos(1, ...)` to open a new trade on `posId = 1` with `100` collateral shares, `orderId = 1` remains `OrderStatus.Active` and can be filled via `fillOrder(1)` as soon as the trigger price condition is met, unexpectedly closing her new position.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:** Stale or conflicting orders execute against collateral that was already reduced, allocated to another order, or newly deposited into a reused position ID.
- **Business Implications:** Users can suffer unexpected position closures, unintended full liquidation of remaining collateral after partial closes or partial order fills, and loss of funds due to execution of stale limit/trigger prices on newly added collateral.

### Remediation

Track or synchronize active orders across position lifecycle changes and order executions:
1. When a position's collateral is reduced to zero (in `reducePos` / `removeCollateral`) or when orders conflict with remaining collateral, cancel or update active orders in `__posOrderIds[initPosId]`.
2. In `fillOrder`, either revert/cancel when `totalCollAmt < order.collAmt` (if partial fills of reduced collateral are not desired) or update/cancel remaining active orders in `__posOrderIds[order.initPosId]` so that total filled collateral across orders cannot exceed the intended allocation, and ensure cancelled status updates are persisted to storage (`__orders[_orderId].status = OrderStatus.Cancelled`).

```solidity
    function fillOrder(uint _orderId) external {
        Order memory order = __orders[_orderId];
        _require(order.status == OrderStatus.Active, Errors.INVALID_INPUT);
        MarginPos memory marginPos = __marginPositions[order.initPosId];
        address collToken = ILendingPool(marginPos.collPool).underlyingToken();
        address borrToken = ILendingPool(marginPos.borrPool).underlyingToken();
        uint collAmt = IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool);
        // if position is empty or has insufficient collateral for the order, cancel order in storage
        if (collAmt < order.collAmt) {
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

## Finding 5: Stale oracle prices in MarginTradingHook.fillOrder can prematurely execute stop-loss orders

### Synopsis

`MarginTradingHook.fillOrder` accepts a price-only oracle response without proving its observation is fresh. With an unbounded source-age configuration, a keeper can execute a stop-loss using an obsolete ratio, forcing the owner to sell collateral even though the current trigger condition is false.

### Technical details

The trigger check at line 385 is the sole market-price gate before an order is settled. The called `_validateTriggerPrice` obtains the base- and quote-asset prices through `getPrice_e36`, but these responses contain no observation timestamp. Consequently, `fillOrder` cannot determine whether the computed ratio represents the current market.

Freshness is delegated entirely to the configured oracle readers. The supplied configuration sets relevant API3 and Pyth maximum stale times to `type(uint).max`, meaning an old observation may continue returning successfully indefinitely. `InitOracle` considers a source valid when its call succeeds and therefore propagates such an obsolete price.

This is exploitable for a long stop-loss when the stale base/quote ratio is below the trigger while the current market has recovered above it. The comparison still passes, after which settlement occurs at the user’s fixed limit price. An executor can pay the debt and owner proceeds at that outdated limit price, receive the collateral shares, and sell the collateral at the higher current market price.

The executor does not need to manipulate the oracle. They can monitor for feed interruptions or stale observations and selectively fill only orders whose obsolete trigger and limit price produce a profit.

### Assumptions and preconditions

- An active stop-loss or equivalent downward-trigger order exists.
- At least one oracle leg returns an obsolete observation without reverting.
- The effective maximum stale time permits that observation, as with the supplied unbounded configuration.
- The stale ratio satisfies the order trigger while a fresh market ratio would not.
- The order’s limit price makes settlement profitable relative to the current market.
- The executor has sufficient tokens and allowances to fund the repayment and owner payment.
- No malicious owner or administrator behavior is required.

### Proof of concept

1. Alice opens a long WETH/USDT position containing collateral shares redeemable for 10 WETH and owing 10,000 USDT.
2. Alice creates a full-position stop-loss with:
   - trigger price: `2,000e36` USDT/WETH;
   - limit price: `1,900e36` USDT/WETH;
   - output token: USDT.
3. Let the accepted oracle observation remain at `1,800e36`, with an old timestamp that is not rejected because the configured maximum stale time is unbounded.
4. Allow the actual current WETH market price to recover to `2,500e36` without the accepted oracle observation updating.
5. Bob calls `fillOrder`.
6. The stale ratio of `1,800e36` satisfies the long stop-loss condition `markPrice <= triggerPrice`, even though the current price of `2,500e36` does not.
7. At the `1,900e36` limit, Bob provides approximately 19,000 USDT in total: 10,000 USDT to repay the debt and 9,000 USDT as Alice’s residual proceeds.
8. Bob receives collateral shares representing 10 WETH, currently worth approximately 25,000 USDT.
9. Ignoring fees and interest, Bob realizes approximately 6,000 USDT of value while Alice’s position is prematurely sold at the stale limit price.

### Impact assessment

**Severity: High.**

A temporary feed interruption can expose every compatible active order to opportunistic execution. Attackers require no privileged role and do not need to cause the stale observation; they only need to monitor the oracle and submit fills when an obsolete trigger produces favorable settlement terms.

A full-position order can force the sale of all committed collateral at a materially outdated price. In the worst case, many users’ stop-loss orders are harvested during the same oracle incident, causing substantial user losses, keeper-extracted value, and reputational damage to the protocol.

### Remediation

Replace the trigger check at line 385 with freshness-aware validation:

- Configure finite, heartbeat-appropriate maximum ages for every oracle source; never use `type(uint).max` for production price feeds.
- Extend the oracle boundary to return each price’s observation timestamp together with its value.
- Before evaluating the trigger, require both base and quote observations to be no older than their configured maximum ages.
- Revert and leave the order active when either timestamp is unavailable or stale.
- Add tests where a stale ratio satisfies the trigger while a fresh ratio does not, and assert that filling reverts.

## Finding 6: Decimal-blind settlement in MarginTradingHook.fillOrder lets fillers underpay for collateral

### Synopsis

`MarginTradingHook.fillOrder` calculates limit-order settlement from raw collateral and debt token amounts without reconciling their decimals. On mixed-decimal pairs, a permissionless filler can receive full collateral while severely underpaying the owner, or render protective orders unfillable.

### Technical details

The collateral and borrowing tokens are obtained at lines 376–377, and line 387 calculates `amtOut` from the position’s collateral and debt. The downstream calculation converts collateral shares into an underlying-token amount and debt shares into an underlying-token amount, so both values are denominated in their respective ERC-20 base units.

The limit price is an economic price ratio scaled by `1e36`; it does not encode either token’s decimals. Nevertheless, the settlement calculation directly adds, subtracts, multiplies, or divides the raw underlying amounts using that price. No conversion to a common precision occurs.

For example, for a short position whose collateral is the quote asset, the owner’s expected quote-token payout is economically:

`collateral - debt × limitPrice`

However, if collateral has `dc` decimals and debt has `db` decimals, the raw-unit conversion must include `10^dc / 10^db`. Omitting that factor misvalues the debt by `10^(db-dc)`.

For a USDT/WBTC position, USDT commonly has 6 decimals while WBTC has 8. A raw WBTC debt amount is consequently treated as if it were already denominated at USDT’s precision, exaggerating its quote value by 100 times. When the resulting subtraction remains positive, the filler pays substantially less than the owner’s actual equity while still receiving all specified collateral shares.

The inverse decimal relationship can instead produce arithmetic underflow or an unrealistically large `amtOut`, preventing stop-loss and take-profit orders from executing. This is not ordinary trigger-to-limit MEV: the settlement itself uses inconsistent denominations.

### Assumptions and preconditions

- The position’s collateral and borrowing tokens use different decimal precisions.
- An owner has created an active order whose trigger condition is satisfied.
- The position direction and leverage cause the incorrectly calculated payout to remain positive.
- The filler has the required token balances and allowances.
- Lending-pool share conversion and redemption operate normally.
- The order uses a documented `1e36` whole-token limit price.

### Proof of concept

1. Use a WBTC/USDT market where:
   - WBTC has 8 decimals.
   - USDT has 6 decimals.
   - WBTC is the base asset.
   - USDT is the quote asset.
   - The limit and oracle price are `100,000e36` USDT per WBTC.
2. Create a short-WBTC position with:
   - Final collateral underlying value of `100,000e6` raw USDT.
   - Debt of `0.005e8 = 500,000` raw WBTC units.
3. Create an immediately triggerable full-collateral order with USDT as `tokenOut`.
4. The correct debt value is:
   - `0.005 WBTC × 100,000 USDT/WBTC = 500 USDT`.
5. Therefore, the correct owner payout is:
   - `100,000 - 500 = 99,500 USDT`.
   - In raw units: `99,500e6`.
6. The decimal-blind calculation instead evaluates the raw amounts as:
   - `100,000e6 - 500,000 × 100,000`
   - `= 100,000,000,000 - 50,000,000,000`
   - `= 50,000e6`, or only `50,000 USDT`.
7. A filler approves `0.005 WBTC` and `50,000 USDT`, then calls `fillOrder`.
8. The position’s WBTC debt is repaid and the filler receives collateral shares representing approximately `100,000 USDT`.
9. The filler spends approximately `50,500 USDT` of value and receives approximately `100,000 USDT` of collateral, capturing about `49,500 USDT` from the order owner.

### Impact assessment

**Severity: High.**

A permissionless filler can exploit a valid mixed-decimal order to acquire collateral substantially below its intended limit-price value. The victim’s debt can be fully repaid and the position cleanly closed, so normal health checks do not detect the economic loss.

The attack requires an active order with a favorable decimal relationship and leverage level, but configured assets already include common 6-, 8-, and 18-decimal tokens. Other mixed-decimal positions may have their protective orders permanently unfillable because the erroneous calculation underflows or demands an excessive payout. This can cause direct principal loss and undermine the protocol’s advertised stop-loss and take-profit functionality.

### Remediation

Normalize collateral and debt amounts to a common precision before applying the limit-price formulas, then convert the resulting payout into `order.tokenOut`’s native decimals.

The corrected calculation should:

- Read and validate both underlying tokens’ decimals.
- Convert collateral and repayment amounts into a common fixed-point denomination.
- Apply the long/short price conversion in that common denomination.
- Convert only the final `amtOut` into the output token’s native units.
- Preserve explicit owner-favoring rounding by using full-precision `mulDiv` operations rather than unsafe chained multiplication.
- Reject unsupported decimal ranges if exponentiation could overflow.
- Add tests for 6/8, 6/18, and 8/18 decimal combinations across:
  - Long and short positions.
  - Collateral-token and borrowing-token outputs.
  - Full and partial order fills.
  - Low- and ordinary-leverage positions.

## Finding 7: Reentrancy and CEI Violation in `MarginTradingHook.fillOrder` Allows Multiple Order Executions and Position State Manipulation

### Synopsis

`MarginTradingHook.fillOrder` lacks reentrancy protection and violates the Checks-Effects-Interactions (CEI) pattern by deferring `__orders[_orderId].status = OrderStatus.Filled` until after multiple external token transfers and core calls, enabling callback-driven reentrancy to re-execute orders or mutate position state mid-settlement.

### Technical details

In `MarginTradingHook.fillOrder(uint _orderId)` (`contracts/hook/MarginTradingHook.sol`), the contract loads `__orders[_orderId]` into memory at line 373, validates that `order.status == OrderStatus.Active` at line 374, and computes the settlement parameters `(amtOut, repayShares, repayAmt)` via `_calculateFillOrderInfo` at line 387 (which also caps the in-memory `order.collAmt` to the position's current collateral in `_calculateRepaySize`).

However, `fillOrder` does not have a `nonReentrant` modifier (and `MarginTradingHook` does not inherit `ReentrancyGuardUpgradeable`), and it does not update `__orders[_orderId].status` in storage before performing external interactions:

1. `IERC20(borrToken).safeTransferFrom(msg.sender, address(this), repayAmt)` (line 389)
2. `IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut)` (line 391)
3. `_ensureApprove(borrToken, repayAmt)` (line 393)
4. `IInitCore(CORE).repay(marginPos.borrPool, repayShares, order.initPosId)` (line 394)
5. `IInitCore(CORE).decollateralize(order.initPosId, marginPos.collPool, order.collAmt, msg.sender)` (line 396)

Only at line 398 is the storage state updated:
```solidity
__orders[_orderId].status = OrderStatus.Filled;
```

Because `__orders[_orderId].status` remains `OrderStatus.Active` throughout lines 389–396 and the settlement parameters (`amtOut`, `repayShares`, `repayAmt`, `order.collAmt`) are already cached in memory prior to line 389:
- If `borrToken` or `order.tokenOut` supports transfer callbacks (e.g., ERC777 tokens or tokens with transfer hooks), either `msg.sender` (the order executor) or `order.recipient` (the position owner) receives control at line 389 or line 391 before any debt repayment, collateral removal, or order status update has taken effect.
- A reentrant call to `fillOrder(_orderId)` sees `__orders[_orderId].status == OrderStatus.Active` and unreduced position collateral/debt shares, allowing a partial order (e.g., `order.collAmt <= totalCollAmt / 2`) to be filled multiple times against the same position.
- Alternatively, during the callback at line 389 or 391, the position owner or executor can invoke other `MarginTradingHook` functions (such as `repayDebt`, `reducePos`, or another order's `fillOrder` for the same `initPosId`) to mutate the position's collateral or debt while the outer `fillOrder` continues executing with stale in-memory snapshots (`repayShares`, `repayAmt`, `order.collAmt`, `amtOut`).

### Assumptions and preconditions

1. An active stop-loss or take-profit order exists in `MarginTradingHook` whose trigger price condition (`_validateTriggerPrice`) is currently met.
2. Either `borrToken` or `order.tokenOut` invokes a hook/callback on the sender or recipient during `safeTransferFrom` (such as an ERC777 token or callback-enabled token).
3. The position has sufficient collateral or open orders for a reentrant `fillOrder` or cross-function position update to succeed without reverting on underflow or health checks.

### Proof of concept

1. Alice opens a leveraged margin position (`initPosId`) on `MarginTradingHook` with `totalCollAmt = 100` collateral shares and creates a take-profit or stop-loss order (`_orderId`) for `collAmt = 50` shares (50% of the position).
2. When the oracle price reaches `triggerPrice_e36`, an executor calls `fillOrder(_orderId)`.
3. `fillOrder` checks `order.status == OrderStatus.Active`, calculates `(amtOut, repayShares, repayAmt)` for `50` collateral shares based on `totalCollAmt = 100`, and calls `IERC20(borrToken).safeTransferFrom(msg.sender, address(this), repayAmt)` at line 389 (or transfers `order.tokenOut` to `order.recipient` at line 391).
4. During the token transfer callback, the caller re-enters `fillOrder(_orderId)` before line 394 (`repay`), line 396 (`decollateralize`), and line 398 (`status = OrderStatus.Filled`) have executed.
5. In the reentrant `fillOrder(_orderId)` call:
   - `__orders[_orderId].status` is still `OrderStatus.Active`.
   - `IPosManager(POS_MANAGER).getCollAmt(order.initPosId, marginPos.collPool)` is still `100`, and `getPosDebtShares` is still unchanged.
   - The inner `fillOrder` transfers `repayAmt` and `amtOut`, repays `50%` of the debt, decollateralizes `50` collateral shares to `msg.sender`, and sets `__orders[_orderId].status = OrderStatus.Filled`.
6. Execution returns to the outer `fillOrder(_orderId)` at line 391/393:
   - Using the stale in-memory `repayShares` and `order.collAmt = 50`, the outer call executes `IInitCore(CORE).repay` and `IInitCore(CORE).decollateralize` a second time, removing the remaining `50` collateral shares from Alice's position even though the order only authorized `50` shares in total.
   - Line 398 overwrites `__orders[_orderId].status = OrderStatus.Filled`.

### Impact assessment

- **Severity:** High / Medium
- **Technical Effect:** A single order can be executed multiple times, or position state (debt shares and collateral shares) can be mutated mid-execution while `fillOrder` proceeds with stale pre-callback settlement amounts (`repayShares`, `repayAmt`, `amtOut`, `order.collAmt`).
- **Business Impact:** Unintended full liquidation/decollateralization of a user's margin position beyond the `collAmt` specified in the order, or favorable arbitrage extraction by a malicious executor/recipient at the expense of the counterparty.

### Remediation

Apply the Checks-Effects-Interactions (CEI) pattern by updating `__orders[_orderId].status = OrderStatus.Filled` in storage before performing any external token transfers or `CORE` calls, and add reentrancy protection (`ReentrancyGuardUpgradeable` / `nonReentrant`) to `MarginTradingHook` state-changing functions.

```solidity
        // update order status before external interactions
        __orders[_orderId].status = OrderStatus.Filled;
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
```

## Finding 8: Stale pair orientation in MarginTradingHook.fillOrder breaks protection orders after quote-asset reconfiguration

### Synopsis

`MarginTradingHook` permanently caches each position’s base asset, quote asset, and long/short direction despite allowing the pair’s quote asset to change. Existing positions can therefore validate and settle orders using a stale reciprocal price convention, disabling protection orders or producing incorrectly priced fills.

### Technical details

When a position is opened, `MarginPos` stores `baseAsset`, `quoteAsset`, and the derived `isLongBaseAsset` flag. These values establish both:

- The numerator and denominator used to calculate the mark price.
- The inequality used to trigger stop-loss and take-profit orders.
- The interpretation of `limitPrice_e36` during settlement.

However, `setQuoteAsset` can overwrite `__quoteAssets[token0][token1]` without checking whether the pair was previously configured. Because a pair contains only two assets, changing the quote asset necessarily swaps the base and quote assets and reverses the pair’s price convention.

Existing `MarginPos` records are not migrated or versioned. Consequently, after reconfiguration:

- `getBaseAssetAndQuoteAsset` exposes the new pair orientation.
- `getMarginPos` still exposes the old orientation and direction.
- `fillOrder` calculates the mark price and trigger direction from the stale position fields.
- Settlement interprets the order’s limit price using the stale, reciprocal denomination.

Integrations following the canonical pair getter can therefore construct a valid stop-loss or take-profit in the new convention while the existing position continues using the old convention. The order may be rejected during creation, fail to trigger after the intended threshold is crossed, or eventually settle using a limit price denominated in the reciprocal asset pair.

### Assumptions and preconditions

- A pair has at least one existing margin position.
- The honest governor legitimately changes which token is the pair’s quote asset.
- Existing positions and their order functionality are expected to remain usable after that configuration change.
- A user or automation service derives trigger and limit prices from the current pair configuration.
- For direct financial loss, an incorrectly denominated order must become executable; otherwise the primary consequence is failure of stop-loss or take-profit protection.

### Proof of concept

1. Configure a pair of equal-decimal tokens `A/B` with `B` as the quote asset.
2. Open a position with `A` as collateral and `B` as the borrowed token.
3. Observe that the position stores:
   - `baseAsset = A`
   - `quoteAsset = B`
   - `isLongBaseAsset = true`
4. Through the normal governor operation, call `setQuoteAsset(A, B, A)`.
5. Query `getBaseAssetAndQuoteAsset(A, B)` and observe that it now reports:
   - `baseAsset = B`
   - `quoteAsset = A`
6. Query the existing position and observe that it still reports the old `A/B` orientation and direction.
7. Calculate a stop-loss trigger and limit using the current `B/A` convention exposed by the pair getter.
8. Attempt to create that order on the existing position. Validation based on the cached convention can reject an otherwise valid protection order.
9. If creation permits the order, move the oracle prices so that `B/A` crosses the intended stop-loss threshold.
10. Call `fillOrder`. The function instead evaluates `A/B` with the old long/short inequality, so the protection order does not execute at its intended threshold.
11. If the stale trigger is eventually reached, the settlement branches also interpret the supplied `B/A` limit as `A/B`, producing a payout based on the wrong denomination.

### Impact assessment

**Severity: Medium.**

Exploitation requires a legitimate quote-asset reconfiguration while older positions remain active. No malicious governor behavior is required.

The primary impact is loss of availability and reliability for stop-loss and take-profit orders. A stop-loss can fail during an adverse market movement, allowing an otherwise protected position to become liquidatable. If a reciprocally denominated order reaches the stale trigger, a permissionless filler may settle it at a value materially different from the owner’s intended limit.

Broader consequences include unexpected liquidations, user losses, inconsistent behavior between official getters, and loss of confidence in automated protection orders.

### Remediation

Prefer making quote-asset orientation immutable once a pair has been configured. Idempotent calls setting the same quote asset may remain allowed.

```solidity
address currentQuoteAsset = __quoteAssets[token0][token1];
_require(
    currentQuoteAsset == address(0) || currentQuoteAsset == _quoteAsset,
    Errors.INVALID_INPUT
);
__quoteAssets[token0][token1] = _quoteAsset;
```

If quote orientation must remain mutable, version each pair configuration and bind every position and order to a version. A reconfiguration must then either preserve old-version pricing endpoints or atomically cancel and migrate all affected orders and positions.

## Finding 9: Missing Order Ownership Validation in `MarginTradingHook.updateOrder` Allows Unauthorized Modification of Arbitrary Active Orders

### Synopsis

`MarginTradingHook.updateOrder` verifies that the caller owns the position `_posId`, but fails to verify that the target order `_orderId` belongs to that position (`order.initPosId == initPosId`). Consequently, any user with an open margin position can overwrite the parameters of any other user's active order.

### Technical details

In `contracts/hook/MarginTradingHook.sol`, margin orders are stored globally in the `__orders` mapping indexed by `_orderId`, and each `Order` struct records the `initPosId` of the position it belongs to.

When cancelling an order in `cancelOrder` (lines 361–369), the contract resolves the caller's position ID via `uint initPosId = initPosIds[msg.sender][_posId]` and explicitly validates that the order belongs to the caller's position:

```solidity
uint initPosId = initPosIds[msg.sender][_posId];
_require(initPosId != 0, Errors.POSITION_NOT_FOUND);
Order storage order = __orders[_orderId];
_require(order.initPosId == initPosId, Errors.INVALID_INPUT);
```

However, in `updateOrder` (lines 513–525), this ownership check is omitted:

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

Because `order.initPosId == initPosId` is never checked:
1. A caller only needs to own any valid position `_posId` (`initPosIds[msg.sender][_posId] != 0`) with at least `_collAmt` collateral shares to pass all checks in `updateOrder`.
2. The caller can supply any active `_orderId` belonging to another user's position and arbitrarily overwrite `order.triggerPrice_e36`, `order.limitPrice_e36`, `order.collAmt`, and `order.tokenOut`.
3. Once modified, `fillOrder` uses `order.initPosId` (the victim's position) along with the attacker-supplied `triggerPrice_e36`, `limitPrice_e36`, and `collAmt` to execute the order against the victim's position.

### Assumptions and preconditions

1. A victim has an open margin position and at least one `Active` stop-loss or take-profit order (`_orderId`).
2. Another user (attacker) can open a margin position via `openPos` so that `initPosIds[attacker][_posId] != 0` with collateral amount `>= _collAmt`.

### Proof of concept

1. Alice opens a long ETH/USDT margin position (`initPosId = 1`, `posId = 1` for Alice) with 10 WETH collateral and 15,000 USDT debt (mark price = 2,000 USDT/WETH).
2. Alice creates a stop-loss order (`orderId = 1`) via `addStopLossOrder` with `_triggerPrice_e36 = 1800e36`, `_limitPrice_e36 = 1800e36`, `_tokenOut = USDT`, and `_collAmt = 10 WETH` (in collateral shares).
3. Bob opens a margin position (`initPosId = 2`, `posId = 1` for Bob) with 10 WETH collateral (or a flash-funded position) so that `initPosIds[Bob][1] = 2`.
4. Bob calls `updateOrder` passing his own `_posId = 1` and Alice's `_orderId = 1`, setting:
   - `_triggerPrice_e36 = 2500e36` (so `markPrice_e36 <= triggerPrice_e36` is immediately satisfied at the current price of 2,000 USDT/WETH),
   - `_limitPrice_e36 = 1500e36` (so the required payout `amtOut = collTokenAmt * limitPrice - repayAmt` evaluates to `10 * 1500 - 15000 = 0`),
   - `_tokenOut = USDT`,
   - `_collAmt = 10 WETH` (in collateral shares).
5. Because `initPosIds[Bob][1] == 2 != 0` and Bob's position has `>= 10 WETH` collateral shares, `updateOrder` succeeds and overwrites Alice's order (`__orders[1]`).
6. Bob immediately calls `fillOrder(1)`. Bob repays only Alice's 15,000 USDT debt (`repayAmt`), pays `amtOut = 0` to Alice, and receives all 10 WETH collateral shares (worth 20,000 USDT), stealing Alice's entire net equity of 5,000 USDT.

### Impact assessment

**Severity: Critical (High)**

- **Direct Loss of Funds:** An attacker can modify any active stop-loss or take-profit order to make it immediately fillable at a manipulated `limitPrice_e36` where `amtOut` is reduced to zero (or near zero), allowing the attacker to seize the victim's position collateral by only repaying the position's debt and extracting all positive equity.
- **Order Sabotage:** Even with a minimal dust position (`_collAmt = 1`), an attacker can reduce any active order's `collAmt` to `1` wei or set unreachable trigger prices, neutralizing users' stop-loss and take-profit protections.

### Remediation

Validate that `order.initPosId` matches the caller's `initPosId` in `updateOrder`, matching the ownership check in `cancelOrder` and the token validation in `_createOrder`.

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

## Finding 10: Missing `_tokenOut` validation in `MarginTradingHook.updateOrder` allows setting arbitrary output tokens for margin orders

### Synopsis

`MarginTradingHook.updateOrder` fails to validate that the updated `_tokenOut` parameter belongs to the margin position's asset pair (`baseAsset` or `quoteAsset`), unlike `_createOrder`. This omission breaks downstream settlement assumptions in `_calculateFillOrderInfo` and `fillOrder`, resulting in corrupted payout calculations or unauthorized token transfers from order executors.

### Technical details

When a stop-loss or take-profit order is created via `MarginTradingHook._createOrder`, line 483 enforces that the desired output token `_tokenOut` must be either the position's `baseAsset` or `quoteAsset`:

```solidity
_require(_tokenOut == marginPos.baseAsset || _tokenOut == marginPos.quoteAsset, Errors.INVALID_INPUT);
```

This invariant is critical for order settlement. During `fillOrder`, `_calculateFillOrderInfo` (lines 532–564) determines the `amtOut` to be paid by the executor (`msg.sender`) to `order.recipient` using a binary check:

```solidity
if (_collToken == _order.tokenOut) {
    ...
} else {
    ...
}
```

The `else` branch implicitly assumes that if `_order.tokenOut != _collToken`, then `_order.tokenOut` must be `_borrToken` (the other asset in the `baseAsset`/`quoteAsset` pair), and therefore computes `amtOut` denominated in `_borrToken` units and decimals.

However, in `MarginTradingHook.updateOrder` (lines 504–526), there is no check verifying that `_tokenOut == marginPos.baseAsset || _tokenOut == marginPos.quoteAsset`. Line 524 directly overwrites `order.tokenOut = _tokenOut` with any arbitrary address supplied by the caller.

Consequently, when `updateOrder` is called with an arbitrary `_tokenOut` address:
1. `_calculateFillOrderInfo` evaluates `_collToken == _order.tokenOut` as `false` and enters the `else` branch, computing `amtOut` in terms of `_borrToken`.
2. `fillOrder` (line 391) executes `IERC20(order.tokenOut).safeTransferFrom(msg.sender, order.recipient, amtOut)`, transferring `amtOut` (scaled for `_borrToken`) of the arbitrary `order.tokenOut` from the executor to `order.recipient`.

### Assumptions and preconditions

1. A user has an open margin position in `MarginTradingHook` and an active stop-loss or take-profit order.
2. `updateOrder` is called to modify the active order with a `_tokenOut` address that is neither `marginPos.baseAsset` nor `marginPos.quoteAsset`.

### Proof of concept

1. Alice opens a long WETH/USDT margin position via `openPos` where `collToken = WETH` (`baseAsset`) and `borrToken = USDT` (`quoteAsset`).
2. Alice creates a valid stop-loss or take-profit order via `addTakeProfitOrder` with `_tokenOut = USDT`.
3. Alice calls `updateOrder` on her active order and passes `_tokenOut = WBTC` (or any other ERC20 token that an executor has approved to `MarginTradingHook`, or an invalid/non-pair token). Because `updateOrder` lacks the asset-domain check, `order.tokenOut` is updated to `WBTC`.
4. When `fillOrder` is called by an executor (who has approved tokens to `MarginTradingHook` for arbitrage/order filling):
   - `_calculateFillOrderInfo` checks `_collToken == _order.tokenOut` (`WETH == WBTC`, which is `false`) and calculates `amtOut` in `USDT` terms.
   - `fillOrder` transfers `amtOut` of `WBTC` from the executor to Alice (`order.recipient`), or reverts if the token is invalid/unapproved, breaking order settlement.

### Impact assessment

**Severity: Medium / High**

- **Direct Technical Effect:** `order.tokenOut` can be set to any arbitrary address via `updateOrder`, violating the order parameter invariant enforced during order creation.
- **Business and Financial Impact:** If an order executor (e.g., an automated keeper or arbitrage bot that approves multiple tokens to `MarginTradingHook` to fill orders across various pairs) fills the order without off-chain filtering of `order.tokenOut`, the executor will transfer `amtOut` (calculated in `borrToken` units) of a much more valuable token (such as `WETH` or `WBTC`) to the order recipient, leading to direct theft of the executor's approved funds. Alternatively, if the user accidentally passes an invalid `_tokenOut` address, the order becomes unfillable and fails to protect the position.

### Remediation

Add the missing `_tokenOut` validation in `MarginTradingHook.updateOrder` to ensure consistency with `_createOrder`.

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

## Finding 11: In-place mutation in MarginTradingHook.updateOrder lets makers front-run and overcharge order fillers

### Synopsis

`MarginTradingHook.updateOrder` mutates settlement-critical fields of an active order under the same ID. A maker can front-run a pending fill, replace the quoted size and price, and make the executor pay its previously approved amount for substantially less collateral, causing direct executor losses.

### Technical details

`updateOrder` obtains a storage reference to an existing active order and overwrites its trigger price, limit price, collateral amount, and output token without changing the order ID or invalidating the old version.

Order execution is therefore subject to a time-of-check/time-of-use mismatch. A filler evaluates the current order, determines that execution is profitable, approves the required tokens, and submits `fillOrder(_orderId)`. However, the fill operation identifies the order only by `_orderId`; it does not commit to the terms the filler observed through an order hash, version, expected collateral amount, or maximum payment.

The legitimate maker can observe the pending fill and front-run it with `updateOrder`. In particular, the maker can:

1. Reduce `collAmt` to a small fraction of the original amount, which passes line 519 because only an upper bound is enforced.
2. Change `limitPrice_e36` so that the calculated output-token payment remains near the amount the filler already approved.
3. Keep `tokenOut` unchanged and set a still-satisfied trigger, avoiding any dependence on unrelated authorization or token-validation issues.

For a long position where the output is the borrowed quote token, settlement calculates output approximately as:

`amtOut = collateralUnderlying × limitPrice / 1e36 − repayAmount`

Because the updated limit price has no effective bound, the maker can choose a high limit that makes a very small collateral amount demand nearly the original payment. The pending fill then executes successfully against the modified storage record. The filler transfers the expected quantity of tokens but receives only the newly reduced collateral amount, while the maker retains the rest of the position and receives the inflated payment.

This remains exploitable even if the caller is correctly authorized for the target order: the defect is mutation of an active, externally referenced order in place rather than a missing ownership check.

### Assumptions and preconditions

- A non-privileged malicious maker has created an active order for its own margin position.
- A permissionless filler reads the order and broadcasts a fill transaction based on the observed terms.
- The maker can observe the pending transaction and obtain earlier execution through priority fees or an MEV relay.
- The filler has sufficient balance and allowance for the payment expected under the original order. An exact allowance is sufficient because the maker can keep the total updated pull at or below that amount.
- The maker selects updated values that keep the trigger condition satisfied and avoid arithmetic reverts.
- Core contracts, pools, and supported tokens are honestly deployed and operate as documented.

### Proof of concept

1. The attacker opens a long base-asset position and creates an active order whose `tokenOut` is the borrowed quote token.
2. The order initially covers the full collateral amount `C` and has a fair limit price. A filler observes that filling it requires total quote-token transfers of `P` and returns `C` collateral-pool shares.
3. The filler approves at least `P` quote tokens to the hook and broadcasts `fillOrder(orderId)`.
4. The attacker observes the pending transaction and front-runs it with `updateOrder` for the same legitimately owned order.
5. The attacker sets `collAmt` to a much smaller value `c`, where `0 < c << C`.
6. The attacker keeps the same `tokenOut`, sets an immediately satisfied trigger, and chooses a larger limit price such that the updated output payment plus updated debt repayment is at most `P`.
7. The update passes because the order remains active, the attacker’s position exists, and `c` does not exceed its current collateral.
8. The filler’s transaction then executes against the same order ID but reads the updated terms. It transfers up to `P` quote tokens while receiving only `c` collateral-pool shares.
9. The attacker receives the inflated output payment and retains approximately `C − c` collateral in the position. The filler suffers the difference between its payment and the value of `c`.

### Impact assessment

**Severity: High.**

A malicious maker can cause direct loss to permissionless order executors by changing an order after they have priced and submitted a fill. The attack requires transaction-ordering control over a pending fill but no privileged role, malicious deployment, or unauthorized access to another user’s order.

With persistent allowances, the loss can approach the filler’s available balance or allowance. Even when the filler approves only the originally calculated amount, the maker can shrink the delivered collateral and adjust the limit price so the modified order consumes the same approved amount. Repeated exploitation would undermine keeper participation and the reliability of stop-loss and take-profit execution.

### Remediation

Do not mutate settlement terms of an active order under the same externally referenced order ID.

The preferred solution is an atomic cancel-and-replace model:

- Mark the existing order as cancelled.
- Create the updated terms under a new monotonically increasing order ID.
- Emit cancellation and creation/replacement events that identify both IDs.
- Ensure a pending fill referencing the old ID reverts on the status check.

Alternatively, add an order version or deterministic hash and require `fillOrder` callers to provide the expected version or hash together with maximum token payments and minimum collateral received. Every update must increment the version, causing transactions based on stale terms to revert rather than execute at changed terms.

## Finding 12: Intermediate overflow in MarginTradingHook._calculateFillOrderInfo blocks valid order fills

### Synopsis

`MarginTradingHook._calculateFillOrderInfo` multiplies a repayment value by the `1e36` price scale before division. A sufficiently large valid order therefore triggers `uint256` overflow even when the final quotient fits, reverting permissionless fills and disabling protective orders, potentially exposing leveraged positions to liquidation.

### Technical details

For long positions whose requested output token is the collateral token, the repayment amount is converted into collateral units using:

`repayAmt × 1e36 ÷ limitPrice_e36`

The implementation performs the multiplication as an ordinary checked `uint256` operation before applying the division. Consequently, any `repayAmt` greater than:

\[
\left\lfloor\frac{2^{256}-1}{10^{36}}\right\rfloor
\]

causes an arithmetic panic, regardless of whether the mathematically correct quotient fits in `uint256`.

This is an intermediate-range failure rather than an overflow of the intended result. For example, with `limitPrice_e36 = 1e36`, the quotient is simply `repayAmt` and remains representable, but the intermediate `repayAmt * 1e36` overflows.

Because `_calculateFillOrderInfo` is executed before the order’s transfers, repayment, decollateralization, and status update, the overflow reverts the entire `fillOrder` transaction. An otherwise valid stop-loss or take-profit order can therefore become deterministically unfillable.

### Assumptions and preconditions

- The order is active and its trigger-price condition has been reached.
- The position is classified as long, and the order requests the collateral token as output.
- The repayment value used by the calculation exceeds `type(uint256).max / 1e36`.
- The supported market’s token denomination, debt ceiling, liquidity, and accrued interest permit such a repayment value.
- The limit price is nonzero, and the mathematically correct converted repayment and resulting `amtOut` both fit in `uint256`.
- No dishonest owner or administrator behavior is required.

### Proof of concept

1. Define:

   \[
   R = \left\lfloor\frac{2^{256}-1}{10^{36}}\right\rfloor + 1
   \]

2. Create a valid long position and active order for which the calculated repayment value is `R`.
3. Set `tokenOut` to the collateral token.
4. Use `limitPrice_e36 = 1e36` and ensure the collateral’s underlying amount is at least `R + 1`.
5. Reach the order’s trigger condition and call `fillOrder`.
6. The intended conversion is:

   \[
   R \times 10^{36} \div 10^{36} = R
   \]

   so the quotient and an `amtOut` of at least one unit are representable.
7. Nevertheless, evaluation first attempts `R * 1e36`, which exceeds `uint256`.
8. The transaction reverts with an arithmetic panic, leaving the order active and the position unchanged and unprotected.

### Impact assessment

**Severity: Medium.**

The vulnerability causes deterministic denial of order execution for affected positions. Any keeper or arbitrageur attempting to fill the order encounters the same revert, so additional gas or alternative transaction ordering cannot resolve it.

If the affected order is a stop loss, failure to execute can leave a leveraged position exposed during adverse market movement. The position owner may suffer liquidation losses, and sufficiently rapid price deterioration could increase bad-debt risk for the lending protocol. Take-profit automation is similarly rendered unreliable.

Exploitation requires a sufficiently large smallest-unit repayment value, so practical exposure depends on supported token denominations and configured market limits. The worst case is a high-denomination market in which valid protective orders systematically become unfillable.

### Remediation

Use OpenZeppelin’s full-precision `mulDiv`, which evaluates the 512-bit intermediate product and only reverts if the final quotient itself cannot fit.

```solidity
amtOut = collTokenAmt - repayAmt.mulDiv(ONE_E36, _order.limitPrice_e36);
```

Apply equivalent full-precision arithmetic to every fixed-point conversion that multiplies token amounts by price values or scaling constants before division.

## Finding 13: Unbounded Subtraction in `MarginTradingHook._calculateFillOrderInfo` Reverts `fillOrder` When Position Equity at Limit Price Is Negative

### Synopsis

In `MarginTradingHook._calculateFillOrderInfo`, `amtOut` is computed by directly subtracting the debt repayment value from the collateral value evaluated at `_order.limitPrice_e36`. When accrued borrow interest or a low limit price causes the debt value to exceed the collateral value at `_order.limitPrice_e36`, the subtraction underflows and reverts `fillOrder`.

### Technical details

When an executor calls `MarginTradingHook.fillOrder(_orderId)`, it invokes `_calculateFillOrderInfo(order, marginPos, collToken)` (lines 532–564 of `contracts/hook/MarginTradingHook.sol`) to compute `amtOut` (the amount of `order.tokenOut` to pay the order recipient), `repayShares`, and `repayAmt`.

In `_calculateFillOrderInfo`:
- Line 549 (`_collToken == _order.tokenOut` and `!_marginPos.isLongBaseAsset`):
  `amtOut = collTokenAmt - (repayAmt * _order.limitPrice_e36 / ONE_E36);`
- Line 556 (`_collToken != _order.tokenOut` and `_marginPos.isLongBaseAsset`):
  `amtOut = (collTokenAmt * _order.limitPrice_e36).ceilDiv(ONE_E36) - repayAmt;`
- Line 561 (`_collToken != _order.tokenOut` and `!_marginPos.isLongBaseAsset`):
  `amtOut = (collTokenAmt * ONE_E36).ceilDiv(_order.limitPrice_e36) - repayAmt;`

Each branch assumes that the collateral value evaluated at `_order.limitPrice_e36` is strictly greater than or equal to the debt repayment value (`repayAmt` converted at `_order.limitPrice_e36`). However:
1. Order creation (`addStopLossOrder` / `addTakeProfitOrder`) does not enforce that `collTokenAmt` valued at `_limitPrice_e36` covers the position's debt.
2. Even if the collateral value at `_limitPrice_e36` initially exceeds the debt when the order is placed, `repayAmt` is dynamically calculated via `ILendingPool(_marginPos.borrPool).debtShareToAmtCurrent(repayShares)` (line 598) and grows over time as borrow interest accrues.

If the debt value exceeds the collateral value evaluated at `_order.limitPrice_e36` (even while the position still has positive equity at `markPrice_e36`, making the fill profitable for an executor if `amtOut == 0`), the unchecked subtraction underflows with a Solidity `Panic(0x11)` arithmetic error. Consequently, `fillOrder` unconditionally reverts instead of capping `amtOut` at `0`.

### Assumptions and preconditions

1. A user has an open margin position in `MarginTradingHook` and an active stop-loss (or take-profit) order.
2. The oracle `markPrice_e36` satisfies `_validateTriggerPrice(order, marginPos)` so the order is eligible to be filled.
3. Due to borrow interest accrual or the configured `_order.limitPrice_e36`, the debt value at `_order.limitPrice_e36` exceeds the collateral value at `_order.limitPrice_e36` (while collateral value at `markPrice_e36` can still exceed `repayAmt`).

### Proof of concept

1. Alice opens a long ETH / short USDC margin position holding `collTokenAmt = 2 ETH` as collateral and borrowing `2,500 USDC` when ETH is `$1,500`.
2. Alice places a stop-loss order with `tokenOut = USDC` (`_collToken != _order.tokenOut`, `_marginPos.isLongBaseAsset == true`), `triggerPrice_e36 = 1,300e36`, and `limitPrice_e36 = 1,260e36`. Initially, at `limitPrice_e36`, the collateral is worth `2 * 1,260 = 2,520 USDC > 2,500 USDC`.
3. Over time, borrow interest accrues in the USDC lending pool such that Alice's `repayAmt` grows from `2,500 USDC` to `2,550 USDC`.
4. The market price of ETH drops to `$1,290`, which satisfies `markPrice_e36 <= triggerPrice_e36` (`1,290 <= 1,300`). At the mark price of `$1,290`, the `2 ETH` collateral is worth `$2,580 > $2,550`, so an executor (Bob) has a `$30` incentive to fill the order if `amtOut == 0`.
5. Bob calls `fillOrder(orderId)`. Execution reaches line 556:
   `amtOut = (collTokenAmt * _order.limitPrice_e36).ceilDiv(ONE_E36) - repayAmt;`
   which evaluates `2,520e6 - 2,550e6`.
6. Because `2,520e6 < 2,550e6`, the subtraction underflows and `fillOrder` reverts with `Panic(0x11)`, preventing Alice's stop-loss order from being executed and exposing the position to liquidation.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** `fillOrder` reverts with an arithmetic underflow whenever `repayAmt` exceeds the collateral value calculated at `_order.limitPrice_e36`.
- **Business Impact:** Stop-loss orders meant to protect leveraged positions from liquidation cannot be filled when triggered if interest accrual or limit price slippage pushes the limit-price equity below zero, even when the position remains solvent at the current mark price and profitable for executors to fill. Users suffer avoidable liquidations and liquidation penalties.

### Remediation

Clamp `amtOut` to `0` when the debt value at `_order.limitPrice_e36` exceeds the collateral value, rather than allowing the subtraction to underflow.

```solidity
            } else {
                // short eth hold usdc
                // 2000 - 1 * 1500 = 500 usdc
                // (c - borrow * limit)
                uint repayInColl = repayAmt * _order.limitPrice_e36 / ONE_E36;
                amtOut = collTokenAmt > repayInColl ? collTokenAmt - repayInColl : 0;
            }
        } else {
            if (_marginPos.isLongBaseAsset) {
                // long eth hold usdc
                // (2 * 1500 - 1500) = 1500 usdc
                // ((c * limit - borrow)
                uint collInBorr = (collTokenAmt * _order.limitPrice_e36).ceilDiv(ONE_E36);
                amtOut = collInBorr > repayAmt ? collInBorr - repayAmt : 0;
            } else {
                // short eth hold eth
                // (3000 - 1 * 1500) / 1500 = 1 eth
                // (c - borrow * limit) / limit
                uint collInBorr = (collTokenAmt * ONE_E36).ceilDiv(_order.limitPrice_e36);
                amtOut = collInBorr > repayAmt ? collInBorr - repayAmt : 0;
            }
```

## Finding 14: Decimal-blind conversions in MarginTradingHook._calculateFillOrderInfo let fillers capture position equity

### Synopsis

`MarginTradingHook._calculateFillOrderInfo` applies e36 whole-token base/quote prices directly to raw token amounts without normalizing token decimals. On unequal-decimal markets, public fillers can receive collateral while underpaying the owner, or protective orders become unfillable, causing direct equity loss and liquidation exposure.

### Technical details

`collTokenAmt` and `repayAmt` are denominated in the respective tokens’ smallest units. In contrast, `limitPrice_e36` represents the exchange rate between whole base and quote tokens, scaled by `1e36`. The highlighted calculations implicitly assume that the base and quote tokens have identical decimals.

Let:

- `P` be `limitPrice_e36`;
- `dB` be the base token’s decimals;
- `dQ` be the quote token’s decimals;
- `Braw` and `Qraw` be amounts in the tokens’ smallest units.

Converting base into quote requires:

`quoteRaw = baseRaw * P * 10^dQ / (1e36 * 10^dB)`

However, line 549 deducts `repayAmt * P / 1e36`. It therefore omits the factor `10^dQ / 10^dB`. For a short ETH/USDT position, where ETH has 18 decimals and USDT has 6, the deduction is inflated by `1e12`.

The same omission occurs in both highlighted cross-token branches:

- Line 556 converts base collateral to quote output without `10^dQ / 10^dB`.
- Line 561 converts quote collateral to base output without `10^dB / 10^dQ`.

No subsequent value check corrects `amtOut`. During `fillOrder`, the executor transfers this incorrectly calculated output to the recipient and receives the selected collateral pool shares. A public filler can selectively execute orders where the error reduces `amtOut`, capturing the difference. Where the error increases `amtOut` or causes subtraction underflow, rational fillers will not execute the order or execution will revert, defeating stop-loss and take-profit protection.

### Assumptions and preconditions

- At least one supported margin pair has different base-token and quote-token decimals.
- A user creates an active stop-loss or take-profit order for such a pair.
- The order’s trigger condition is reached.
- For direct value extraction, the position’s debt-to-collateral ratio is low enough that the incorrectly inflated deduction does not underflow.
- The attacker can obtain and approve the repayment token and the incorrectly calculated output amount.
- The collateral pool shares are transferable and retain value corresponding to their underlying collateral.

### Proof of concept

1. Use a short ETH/USDT market:
   - Base token: WETH with 18 decimals.
   - Quote and collateral token: USDT with 6 decimals.
   - Limit price: `2,000e36` USDT per ETH.
2. Create a healthy position whose selected collateral is worth 1,000,000 USDT:
   - `collTokenAmt = 1,000,000 * 1e6 = 1e12`.
   - Current debt repayment amount is `250,000,000` WETH wei, or `2.5e-10` WETH.
3. Create an order covering all collateral with USDT as `tokenOut`, and arrange for its trigger condition to be satisfied.
4. The correctly normalized debt value in raw USDT is:

   `floor(250,000,000 * 2,000e36 * 1e6 / (1e36 * 1e18)) = 0`

   Economically, the debt is only `0.0000005` USDT.
5. Line 549 instead computes:

   `250,000,000 * 2,000e36 / 1e36 = 500,000,000,000`

   This is interpreted as 500,000 USDT.
6. Consequently, `amtOut` is calculated as:

   `1,000,000 USDT - 500,000 USDT = 500,000 USDT`
7. An attacker calls `fillOrder`, supplies the negligible WETH debt and 500,000 USDT to the recipient, and receives collateral pool shares worth approximately 1,000,000 USDT.
8. The attacker captures approximately 500,000 USDT of the owner’s equity. By selecting debt that makes the erroneous deduction approach the full collateral amount, the attacker can capture nearly all selected collateral value.

### Impact assessment

**Severity: High.**

Any public account can exploit an already active, triggered order without compromising an administrator or oracle. For favorable decimal directions and position ratios, the executor can acquire valuable collateral shares while paying the owner substantially less than the order’s intended limit-price value.

The worst case is loss of nearly all collateral selected by the order. In the opposite decimal direction or at ordinary leverage, the same root cause produces excessive output requirements or arithmetic underflow. This makes stop-loss and take-profit orders unfillable, potentially leaving positions exposed until liquidation. The asymmetry is especially harmful because fillers can execute only profitable mispricings while ignoring orders that overcharge them.

### Remediation

Normalize every base/quote conversion for both token decimals while preserving the documented owner-favoring rounding:

- At line 549, deduct a base-to-quote conversion rounded down:

  `baseRaw * P * 10^dQ / (1e36 * 10^dB)`

- At line 556, convert base collateral to quote output using the same normalized expression rounded up, then subtract quote-denominated repayment.
- At line 561, convert quote collateral to base output, rounded up:

  `quoteRaw * 1e36 * 10^dB / (P * 10^dQ)`

Use a full-precision multiplication/division utility so the additional decimal factors cannot overflow and intermediate rounding does not alter the final result. Prefer storing validated token-unit factors with configured pool metadata rather than repeatedly trusting arbitrary token metadata.

Add independent tests covering 18/6, 8/18, and equal-decimal pairs for both long and short positions, both possible output tokens, low-debt positions, and partial order sizes. Assertions should compare economic values after decimal normalization rather than merely comparing two implementations of the same formula.
