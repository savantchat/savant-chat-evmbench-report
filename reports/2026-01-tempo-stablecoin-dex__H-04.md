# Audit report — 2026-01-tempo-stablecoin-dex/H-04

## Finding 1: Nominal Transfer Amount Credited in `StablecoinDEX.deposit` Causes Accounting Insolvency with Fee-on-Transfer Tokens

### Synopsis

In `StablecoinDEX.deposit`, user balances and `totalDeposits` are credited with the nominal `amount` parameter instead of measuring the actual tokens received by the contract. When fee-on-transfer tokens are deposited, internal accounting exceeds actual token holdings, leading to protocol insolvency and failed withdrawals.

### Technical details

In `contracts/StablecoinDEX.sol`, the `deposit` function transfers tokens from `msg.sender` to `address(this)` and directly updates state variables using the input parameter `amount`:

```solidity
IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
balances[msg.sender][token] += amount;
totalDeposits[token] += amount;
```

Certain ERC-20 tokens (including stablecoins with configurable transfer fees such as USDT or PAX/USDP, or deflationary tokens) deduct a fee during `transferFrom`, resulting in the recipient contract receiving less than the requested `amount`. Because `StablecoinDEX` does not check `IERC20(token).balanceOf(address(this))` before and after the `safeTransferFrom` call, `balances[msg.sender][token]` and `totalDeposits[token]` are inflated by the fee amount on every deposit.

This directly violates the protocol invariant `token.balanceOf(address(dex)) == dex.getTotalLiquidity(address(token))`. Furthermore, depositors are credited with unbacked token balances that they can use to fill orders or withdraw up to the full nominal `amount`, draining tokens deposited by other users and leaving the last users unable to withdraw their funds.

### Assumptions and preconditions

- A token that charges a transfer fee (fee-on-transfer) is used on the permissionless `StablecoinDEX`.
- Users deposit and subsequently trade or withdraw the fee-on-transfer token.

### Proof of concept

1. Assume `TokenFOT` is an ERC-20 token charging a 1% fee on transfers.
2. User A deposits `10,000e6` of `TokenFOT` into `StablecoinDEX`. The contract receives `9,900e6` tokens, while `balances[UserA][TokenFOT]` and `totalDeposits[TokenFOT]` are recorded as `10,000e6`.
3. User B deposits `10,000e6` of `TokenFOT` into `StablecoinDEX`. The contract receives another `9,900e6` tokens (total actual balance: `19,800e6`), while `balances[UserB][TokenFOT]` is `10,000e6` and `totalDeposits[TokenFOT]` is `20,000e6`.
4. User B calls `withdraw(address(TokenFOT), 10_000e6)`. The contract transfers `10,000e6` to User B, leaving `9,800e6` tokens in `StablecoinDEX`.
5. User A attempts to withdraw their `10,000e6` balance via `withdraw(address(TokenFOT), 10_000e6)`. The transaction reverts in `safeTransfer` because the contract only holds `9,800e6` tokens.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** `totalDeposits[token]` and the sum of `balances[user][token]` exceed `IERC20(token).balanceOf(address(this))`, breaking the core custody accounting invariant and causing later withdrawals to revert due to insufficient token balance in the contract.
- **Business Impact:** Loss of user funds (insolvency for the affected token pool) and frozen withdrawals for the remaining depositors, or direct theft of liquidity if an attacker repeatedly deposits, swaps/withdraws using the unbacked phantom balance against honest liquidity providers.

### Remediation

Measure the contract's token balance before and after the `safeTransferFrom` call, and credit `balances[msg.sender][token]` and `totalDeposits[token]` with the actual received amount.

```solidity
    function deposit(address token, uint128 amount) external {
        if (token == address(0)) revert ZeroAddress();
        if (amount == 0) revert InvalidAmount();

        uint256 balanceBefore = IERC20(token).balanceOf(address(this));
        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);
        uint128 received = uint128(IERC20(token).balanceOf(address(this)) - balanceBefore);
        if (received == 0) revert InvalidAmount();

        balances[msg.sender][token] += received;
        totalDeposits[token] += received;

        emit Deposited(msg.sender, token, received);
    }
```

## Finding 2: Missing Balance Validation and Unchecked Underflow in `StablecoinDEX.emergencyWithdraw` and `StablecoinDEX._processWithdrawal` Allow Complete Protocol Drain

### Synopsis

`StablecoinDEX.emergencyWithdraw` invokes `_processWithdrawal` without validating the caller's token balance, while `_processWithdrawal` decrements `balances` and `totalDeposits` inside an `unchecked` block. Any user can call `emergencyWithdraw` with an arbitrary amount to underflow their balance and steal all ERC-20 tokens held by the DEX.

### Technical details

In `StablecoinDEX`, standard withdrawals through `withdraw(address token, uint128 amount)` enforce input and entitlement checks before transferring tokens:

1. `if (token == address(0)) revert ZeroAddress();`
2. `if (amount == 0) revert InvalidAmount();`
3. `if (balances[msg.sender][token] < amount) revert InsufficientBalance();`

However, `emergencyWithdraw(address token, uint128 amount)` (lines 390–393) bypasses all of these validations and directly calls the internal helper `_processWithdrawal(msg.sender, token, amount)` (lines 372–382).

Inside `_processWithdrawal`, the contract decrements both the user's balance and the global deposit accounting inside an `unchecked` block:

```solidity
unchecked {
    balances[user][token] -= amount;
    totalDeposits[token] -= amount;
}
IERC20(token).safeTransfer(user, amount);
```

Because neither `emergencyWithdraw` nor `_processWithdrawal` verifies that `balances[user][token] >= amount`, calling `emergencyWithdraw` with an `amount` greater than the caller's recorded balance does not revert in Solidity 0.8.28. Instead:
- `balances[msg.sender][token]` wraps around via integer underflow to a huge `uint128` value (`type(uint128).max - amount + balances[msg.sender][token] + 1`).
- `totalDeposits[token]` similarly decrements without bounds checking (or underflows if `amount > totalDeposits[token]`).
- `IERC20(token).safeTransfer(user, amount)` transfers up to the entire token balance of the `StablecoinDEX` contract to `msg.sender`, including funds deposited by other users and tokens currently locked in open orders.

### Assumptions and preconditions

- The `StablecoinDEX` contract holds a non-zero balance of at least one ERC-20 `token` (deposited by legitimate users or makers).
- An unprivileged caller invokes `emergencyWithdraw(token, amount)` where `amount` exceeds `balances[msg.sender][token]` and is less than or equal to `IERC20(token).balanceOf(address(dex))`.

### Proof of concept

1. Honest users deposit `100,000e6` of `baseToken` into `StablecoinDEX` via `deposit(address(baseToken), 100_000e6)`. The DEX now holds `100,000e6` `baseToken` and `totalDeposits[baseToken] == 100_000e6`.
2. An attacker with `0` deposited balance (`balances[attacker][baseToken] == 0`) calls `dex.emergencyWithdraw(address(baseToken), 100_000e6)`.
3. Inside `_processWithdrawal(attacker, address(baseToken), 100_000e6)`:
   - The `unchecked` block underflows `balances[attacker][baseToken]` from `0` to `type(uint128).max - 100_000e6 + 1`.
   - `totalDeposits[baseToken]` is reduced from `100,000e6` to `0`.
   - `IERC20(baseToken).safeTransfer(attacker, 100_000e6)` transfers all `100,000e6` tokens from the DEX to the attacker.
4. The transaction succeeds, leaving the DEX completely drained of `baseToken` while the attacker also retains a massive underflowed internal balance.

### Impact assessment

- **Severity:** Critical.
- **Technical Effect:** Complete loss of all ERC-20 tokens held by the `StablecoinDEX` contract and corruption of internal `balances` and `totalDeposits` accounting via arithmetic underflow.
- **Business Impact:** Any unprivileged address can single-handedly drain 100% of protocol liquidity across all tokens in a single transaction with zero capital required.

### Remediation

Validate the withdrawal parameters and user balance before modifying state, and remove the unsafe `unchecked` block (or strictly enforce the precondition prior to the `unchecked` subtraction) in `_processWithdrawal` and `emergencyWithdraw`:

```solidity
    function _processWithdrawal(
        address user,
        address token,
        uint128 amount
    ) internal {
        if (token == address(0)) revert ZeroAddress();
        if (amount == 0) revert InvalidAmount();
        if (balances[user][token] < amount) revert InsufficientBalance();

        balances[user][token] -= amount;
        totalDeposits[token] -= amount;

        IERC20(token).safeTransfer(user, amount);
    }

    function emergencyWithdraw(address token, uint128 amount) external {
        _processWithdrawal(msg.sender, token, amount);
        emit Withdrawn(msg.sender, token, amount);
    }
```
