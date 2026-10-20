# Audit report — 2026-01-tempo-stablecoin-dex/H-02

## Finding 1: Missing Authorization Check in `StablecoinDEX.cancel` Allows Arbitrary Users to Cancel Any Open Order

### Synopsis

`StablecoinDEX.cancel` fails to verify that `msg.sender` is the maker of the target order before invoking `_cancelOrder`. Consequently, any external caller can unilaterally cancel arbitrary active limit orders in `orders`, causing unauthorized order state transitions, orderbook liquidity denial-of-service, and market manipulation.

### Technical details

In `StablecoinDEX.sol`, makers create limit orders via `placeOrder`, which stores the order record in the `orders` mapping (`mapping(uint128 => Order) public orders` at line 73) with `order.maker = msg.sender` and links the order ID into `orderHeads[baseToken][quoteToken]`.

When cancelling an open order via `cancel(uint128 orderId)` (lines 258–263):

```solidity
function cancel(uint128 orderId) external {
    Order storage order = orders[orderId];
    if (order.maker == address(0)) revert OrderDoesNotExist();

    _cancelOrder(orderId, order);
}
```

The function only validates that `order.maker != address(0)` (line 260) and immediately proceeds to `_cancelOrder(orderId, order)` (line 262). Although the contract defines `error Unauthorized();` at line 139, `cancel` never checks `order.maker == msg.sender`.

Furthermore, `_cancelOrder` (lines 269–298) unlinks the order from `orderHeads` and neighbor pointers, refunds `order.remaining` to `balances[maker][baseToken]`, and only deletes individual fields (`maker`, `baseToken`, `quoteToken`, `price`, `amount`) while leaving `remaining`, `prev`, and `next` populated in `orders[orderId]`. Because `cancel` is completely permissionless for any existing `orderId`, any third party can force an open order into a cancelled state against the maker's intent.

### Assumptions and preconditions

1. A maker has deposited `baseToken` into `StablecoinDEX` and placed a valid open limit order via `placeOrder`, creating an active entry in `orders[orderId]` with `order.maker != address(0)`.
2. The order has not yet been completely filled or cancelled.

### Proof of concept

1. `maker` deposits `10,000e6` of `baseToken` via `deposit(baseToken, 10_000e6)` and places a limit order via `placeOrder(baseToken, quoteToken, 1e18, 10_000e6)`, creating `orderId`.
2. An unprivileged third-party account (`attacker`, where `attacker != maker`) observes `orderId` via `getOrders` or the `OrderPlaced` event.
3. `attacker` calls `dex.cancel(orderId)`.
4. `cancel` checks `if (order.maker == address(0))` (which is false) and executes `_cancelOrder(orderId, order)`.
5. The order is unlinked from `orderHeads[baseToken][quoteToken]`, `orders[orderId].maker` is cleared to `address(0)`, and subsequent attempts by a legitimate `taker` to call `dex.fillOrder(orderId, amount)` revert with `OrderDoesNotExist()`.

### Impact assessment

- **Severity**: High / Medium
- **Direct Technical Effect**: Any unprivileged address can cancel any open order in `orders` at any time.
- **Business Implications**:
  - **Orderbook Denial of Service**: An attacker can continuously cancel all active orders across token pairs, completely halting swaps for Machine Payments Protocol (MPP) takers.
  - **Competitive Griefing & Price Manipulation**: Competing market makers can front-run taker fills by cancelling rival makers' better-priced orders so that takers are forced to fill the attacker's worse-priced orders.

### Remediation

Enforce that only `order.maker` can cancel their order by reverting with `Unauthorized()` when `msg.sender != order.maker` in `cancel` (and clean up the full `Order` struct on cancellation):

```solidity
    function cancel(uint128 orderId) external {
        Order storage order = orders[orderId];
        if (order.maker == address(0)) revert OrderDoesNotExist();
        if (order.maker != msg.sender) revert Unauthorized();

        _cancelOrder(orderId, order);
    }
```

## Finding 2: Post-Increment of `nextOrderId` in `StablecoinDEX.placeOrder` Assigns Sentinel Value `0` to the First Order, Corrupting the Order Book Linked List

### Synopsis

In `StablecoinDEX.placeOrder`, `nextOrderId` is post-incremented (`orderId = nextOrderId++`), assigning `orderId = 0` to the first order. Because `0` is also the linked-list sentinel for an empty pointer, the first order is treated as non-existent by list operations, orphaning it and allowing its later cancellation or fill to wipe out the entire pair's order book head.

### Technical details

The `StablecoinDEX` contract maintains a doubly linked list of open orders for each `(baseToken, quoteToken)` pair using `orderHeads[baseToken][quoteToken]` as the list head and `Order.prev` / `Order.next` for neighbor links. Across the contract, the ID `0` is reserved as the sentinel value indicating "no order" / null pointer:

- In `placeOrder` (line 236): `if (currentHead != 0)` checks whether the list already has a head order.
- In `_cancelOrder` (lines 276, 283) and `_removeFilledOrder` (lines 345, 351): `if (order.prev != 0)` and `if (order.next != 0)` determine whether an order is the head/tail of the list.
- In `getOrders` (lines 457, 462): `if (head == 0)` and `while (current != 0 ...)` terminate traversal at `0`.

However, `nextOrderId` is declared at line 83 (`uint128 public nextOrderId;`) without an initial value, defaulting to `0`. In `placeOrder` at line 222:

```solidity
orderId = nextOrderId++;
```

Because post-increment returns the value prior to incrementing, the very first call to `placeOrder` assigns `orderId = 0` and increments `nextOrderId` to `1`. This violates the sentinel invariant and causes multiple structural failures in the order book:

1. **Immediate unreachability of Order `0`**: Line 240 sets `orderHeads[baseToken][quoteToken] = 0`. Calling `getOrders` for this token pair checks `if (head == 0) return new uint128[](0);` (line 457), making Order `0` invisible to order book queries.
2. **Orphaning of Order `0` upon subsequent orders**: When a second order (`orderId = 1`) is placed on the same pair, `currentHead` is read as `0` (line 235). The check `if (currentHead != 0)` (line 236) evaluates to `false`, skipping the linkage between Order `0` and Order `1`, and overwriting `orderHeads[baseToken][quoteToken]` with `1`.
3. **Destruction of the active order book head**: Because Order `0` remains stored in `orders[0]` with `prev == 0` and `next == 0`, when Order `0` is subsequently cancelled via `cancel(0)` or completely filled via `fillOrder(0, ...)`, `_cancelOrder` / `_removeFilledOrder` sees `order.prev == 0` and assumes Order `0` is the current head of the list. It then executes `orderHeads[baseToken][quoteToken] = order.next;` (setting `orderHeads` to `0`), unlinking the entire chain of active orders (`orderId >= 1`) from `orderHeads`.

### Assumptions and preconditions

- The `StablecoinDEX` contract is freshly deployed with `nextOrderId == 0`.
- A user places the first order (`orderId == 0`) via `placeOrder`.

### Proof of concept

1. **Initial Order Placement**:
   - Maker A deposits `10,000e6` of `baseToken` and calls `placeOrder(baseToken, quoteToken, 1e18, 10_000e6)`.
   - Line 222 sets `orderId = 0` and updates `nextOrderId = 1`.
   - `orders[0]` is populated, and `orderHeads[baseToken][quoteToken]` is set to `0`.
   - Any call to `getOrders(baseToken, quoteToken, 10)` sees `head == 0` at line 457 and returns an empty array `[]`, even though Order `0` is active and Maker A's tokens are locked.

2. **Subsequent Order Placement**:
   - Maker B deposits `10,000e6` of `baseToken` and calls `placeOrder(baseToken, quoteToken, 1e18, 10_000e6)` for the same pair.
   - Line 222 sets `orderId = 1` and updates `nextOrderId = 2`.
   - Line 235 reads `currentHead = orderHeads[baseToken][quoteToken]`, which is `0`.
   - Line 236 (`if (currentHead != 0)`) is false, so `orders[0].prev` and `orders[1].next` remain `0`.
   - Line 240 sets `orderHeads[baseToken][quoteToken] = 1`.

3. **Order Book Head Wipeout**:
   - Maker A (or anyone, or a taker filling Order `0`) calls `cancel(0)` (or `fillOrder(0, 10_000e6)`).
   - Inside `_cancelOrder(0, orders[0])`, `order.prev` is `0` and `order.next` is `0`.
   - The `else` branch at lines 278-281 executes: `orderHeads[baseToken][quoteToken] = order.next;` which sets `orderHeads[baseToken][quoteToken] = 0`.
   - Maker B's active order (`orderId = 1`) and any other orders linked to it are now completely detached from `orderHeads[baseToken][quoteToken]`, rendering the pair's order book empty to `getOrderHead` and `getOrders`.

### Impact assessment

- **Severity**: High / Medium.
- **Technical Impact**: The first order placed on the DEX is unreachable via `getOrders` and `getOrderHead`, and fails to link with subsequent orders on the same token pair. Furthermore, cancelling or filling Order `0` after other orders have been placed on that token pair overwrites `orderHeads[baseToken][quoteToken]` with `0`, detaching all active orders in that market's linked list.
- **Business Impact**: Market makers and takers relying on `getOrderHead` and `getOrders` to discover liquidity cannot see Order `0` or any orders detached when Order `0` is removed, breaking order discovery and matching for the affected trading pair.

### Remediation

Pre-increment `nextOrderId` in `placeOrder` so that valid order IDs start at `1`, preserving `0` exclusively as the sentinel value for empty linked-list pointers.

```solidity
        // Lock tokens
        balances[msg.sender][baseToken] -= amount;

        // Create order
        orderId = ++nextOrderId;
        orders[orderId] = Order({
            maker: msg.sender,
            baseToken: baseToken,
            quoteToken: quoteToken,
            price: price,
            amount: amount,
            remaining: amount,
            prev: 0,
            next: 0
        });

        // Insert into linked list (at head for simplicity)
        uint128 currentHead = orderHeads[baseToken][quoteToken];
        if (currentHead != 0) {
            orders[currentHead].prev = orderId;
            orders[orderId].next = currentHead;
        }
        orderHeads[baseToken][quoteToken] = orderId;
```

## Finding 3: Sentinel Collision With `orderId = 0` in `StablecoinDEX._cancelOrder` Corrupts `orderHeads` and Orphans Active Orders

### Synopsis

`StablecoinDEX` uses `0` both as the linked-list null sentinel and as the first valid `orderId`. When order `0` is cancelled after newer orders are placed on the same pair, `_cancelOrder` misinterprets `order.prev == 0` as proof that order `0` is the current head and resets `orderHeads` to `0`.

### Technical details

In `StablecoinDEX`, `nextOrderId` is uninitialized and defaults to `0`, while `0` is simultaneously used across the orderbook doubly-linked list as the sentinel value meaning "no order" (`orderHeads[baseToken][quoteToken] == 0`, `order.prev == 0`, `order.next == 0`).

1. When the first order is created via `placeOrder`, it is assigned `orderId = 0`, and `orderHeads[baseToken][quoteToken]` is set to `0` (line 240). Because `0` collides with the empty-list sentinel, `getOrders` immediately treats the orderbook for this pair as empty (line 457).
2. When a second order (`orderId = 1`) is placed on the same `(baseToken, quoteToken)` pair, `placeOrder` reads `currentHead = orderHeads[baseToken][quoteToken]` which is `0`. The check `if (currentHead != 0)` at line 236 evaluates to `false`, so `orders[0].prev` is never set to `1` and `orders[1].next` is never set to `0`. Instead, `orderHeads[baseToken][quoteToken]` is overwritten with `1`, leaving `orders[0]` active in storage (`orders[0].maker != address(0)`) with `orders[0].prev == 0` and `orders[0].next == 0`.
3. As additional orders (`1, 2, ...`) are placed on the same `(baseToken, quoteToken)` pair, they link properly into a chain headed at `orderHeads[baseToken][quoteToken]`.
4. When `cancel(0)` is subsequently invoked, `_cancelOrder` executes lines 276–281:
   ```solidity
   if (order.prev != 0) {
       orders[order.prev].next = order.next;
   } else {
       // This was the head
       orderHeads[baseToken][quoteToken] = order.next;
   }
   ```
   Because `orders[0].prev` is `0`, `_cancelOrder` assumes order `0` is still the head of the `(baseToken, quoteToken)` order list and overwrites `orderHeads[baseToken][quoteToken]` with `orders[0].next` (`0`).

This resets `orderHeads[baseToken][quoteToken]` to `0`, severing the head pointer for the token pair and orphaning all open orders (`1, 2, ...`) that were placed on that market.

### Assumptions and preconditions

1. Order `0` (`orderId == 0`) is placed on a `(baseToken, quoteToken)` pair.
2. At least one additional order (`orderId >= 1`) is placed on the same `(baseToken, quoteToken)` pair before order `0` is cancelled or filled.
3. `cancel(0)` is called to cancel order `0`.

### Proof of concept

1. Maker A deposits `baseToken` and calls `placeOrder(baseToken, quoteToken, 1e18, 10_000e6)`.
   - `orderId = 0` is created with `orders[0].prev = 0` and `orders[0].next = 0`.
   - `orderHeads[baseToken][quoteToken]` is set to `0`.
2. Maker B deposits `baseToken` and calls `placeOrder(baseToken, quoteToken, 1e18, 10_000e6)`.
   - `orderId = 1` is created.
   - `currentHead` is read as `0`, so `if (currentHead != 0)` is false.
   - `orderHeads[baseToken][quoteToken]` is updated to `1`. `orders[0].prev` and `orders[0].next` remain `0`.
3. Maker B places another order `orderId = 2` on `(baseToken, quoteToken)`.
   - `currentHead` is `1 != 0`, so `orders[1].prev = 2`, `orders[2].next = 1`, and `orderHeads[baseToken][quoteToken] = 2`.
   - Calling `getOrders(baseToken, quoteToken, 10)` returns `[2, 1]`.
4. `cancel(0)` is called.
   - Inside `_cancelOrder(0, orders[0])`, `orders[0].prev` is `0`.
   - Execution enters the `else` branch at line 278 and executes `orderHeads[baseToken][quoteToken] = orders[0].next` (`0`) at line 280.
5. `orderHeads[baseToken][quoteToken]` is now `0`. Calling `getOrders(baseToken, quoteToken, 10)` returns `[]`, and any new order placed on `(baseToken, quoteToken)` sees `currentHead == 0`, permanently detaching orders `1` and `2` from the orderbook linked list.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Cancelling `orderId = 0` after newer orders exist on the same token pair resets `orderHeads[baseToken][quoteToken]` to `0`, corrupting the linked-list head pointer and detaching all active orders on that pair from the orderbook. Additionally, `orderId = 0` itself is invisible to `getOrders` and cannot be properly linked with subsequent orders because `0` is treated as the null sentinel.
- **Business Impact:** Market makers' active orders on the affected `(baseToken, quoteToken)` pair disappear from orderbook discovery (`getOrders` and `getOrderHead`), halting normal order discovery and matching for those orders until makers manually cancel each orphaned order by its ID.

### Remediation

Reserve `0` exclusively as the null sentinel by initializing `nextOrderId = 1`, and in `_cancelOrder` ensure `orderHeads[baseToken][quoteToken]` is only updated when the cancelled order is actually the current head:

```solidity
    function _cancelOrder(uint128 orderId, Order storage order) internal {
        uint128 refund = order.remaining;
        address maker = order.maker;
        address baseToken = order.baseToken;
        address quoteToken = order.quoteToken;

        // Update linked list - unlink from neighbors
        if (order.prev != 0) {
            orders[order.prev].next = order.next;
        } else if (orderHeads[baseToken][quoteToken] == orderId) {
            // This was the head
            orderHeads[baseToken][quoteToken] = order.next;
        }

        if (order.next != 0) {
            orders[order.next].prev = order.prev;
        }
```
