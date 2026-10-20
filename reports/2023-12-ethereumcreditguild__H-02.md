# Audit report — 2023-12-ethereumcreditguild/H-02

## Finding 1: Lossy Balance-Share Conversion in `ERC20RebaseDistributor._balance2shares` and `_shares2balance` Causes Share Loss and Underflow Reverts

### Synopsis

In `ERC20RebaseDistributor`, `_balance2shares` and `_shares2balance` perform integer division using `START_REBASING_SHARE_PRICE` as both the initial `sharePrice` and the scaling factor. Once `sharePrice > START_REBASING_SHARE_PRICE`, each share is worth more than `1` wei of tokens, making `shares -> balance -> shares` round-trips lossy, causing zero-share attribution, cumulative share loss, and underflow reverts.

### Technical details

In `src/tokens/ERC20RebaseDistributor.sol`, `START_REBASING_SHARE_PRICE` is set to `1e30`, which is also the initial value of `__rebasingSharePrice`. The helper functions `_balance2shares` (lines 252–257) and `_shares2balance` (lines 260–272) convert between token balances and rebasing shares:

$$\text{\_balance2shares}(\text{balance}, P) = \left\lfloor \frac{\text{balance} \cdot 10^{30}}{P} \right\rfloor$$

$$\text{\_shares2balance}(S, P, \Delta, \text{minBalance}) = \max\left(\left\lfloor \frac{S \cdot P}{10^{30}} \right\rfloor + \Delta,\; \text{minBalance}\right)$$

Because `sharePrice` ($P$) starts at $10^{30}$ and strictly increases upon reward distribution (`distribute`), the factor $10^{30}$ cancels out at initialization ($1\text{ share} = 1\text{ wei}$), and after any distribution $P > 10^{30}$, meaning **1 share is worth more than 1 wei of tokens** ($P / 10^{30} > 1$). Consequently, shares have coarser granularity than token wei, and both conversion functions truncate downward.

Whenever a rebasing account interacts with `_mint`, `transfer`, `transferFrom`, or `_burn`, the contract materializes the account's current shares $S$ into a token balance via `_shares2balance`, applies the token delta, and converts the resulting token balance back into shares via `_balance2shares`. Let $S \cdot P = q \cdot 10^{30} + r$, where $q = \lfloor S \cdot P / 10^{30} \rfloor$ and $0 \le r < 10^{30}$. When $q \ge \text{rawBalanceBefore}$ (which holds whenever the user has accrued at least 1 wei of unminted rebase rewards):

1. **Recipient round-trip (`_mint`, `transfer`, `transferFrom`)**:
   `_shares2balance` computes $\text{toBalanceAfter} = q + \text{amount}$, discarding the remainder $r$. Then `_balance2shares` computes:
   $$\text{toSharesAfter} = \left\lfloor \frac{(q + \text{amount}) \cdot 10^{30}}{P} \right\rfloor = \left\lfloor \frac{S \cdot P - r + \text{amount} \cdot 10^{30}}{P} \right\rfloor = S + \left\lfloor \frac{\text{amount} \cdot 10^{30} - r}{P} \right\rfloor$$
   - **Zero-amount underflow revert**: When $\text{amount} = 0$ and $r > 0$, $\text{toSharesAfter} = S - 1 < S$. Subsequent calculation of `uint256 sharesReceived = toSharesAfter - rebasingStateTo.nShares` (lines 531, 618, 712) underflows and reverts with `Panic(0x11)`.
   - **Zero-share attribution for small/dust amounts**: Whenever $\text{amount} \cdot 10^{30} < P + r$ (which holds for $\text{amount} = 1\text{ wei}$ for **every** $P > 10^{30}$ since $10^{30} - r \le 10^{30} < P$, and for any $\text{amount} < \lfloor P / 10^{30} \rfloor$), $\text{toSharesAfter} = S$ (`sharesReceived = 0`). The recipient is attributed zero additional shares, and the received `amount` in `ERC20.balanceOf(to)` is subsequently absorbed into `minBalance` as soon as $S$ shares accrue `amount` of rebase yield, effectively erasing the transferred tokens from the recipient's rebased balance.

2. **Sender/burner round-trip (`transfer`, `transferFrom`, `_burn`)**:
   `_shares2balance` computes $q = \lfloor S \cdot P / 10^{30} \rfloor$, and `_balance2shares(q - amount, P)` computes:
   $$\text{fromSharesAfter} = \left\lfloor \frac{(q - \text{amount}) \cdot 10^{30}}{P} \right\rfloor = S - \left\lceil \frac{\text{amount} \cdot 10^{30} + r}{P} \right\rceil$$
   - When $\text{amount} = 0$ and $r > 0$, $\text{fromSharesAfter} = S - 1$, burning 1 share from `from` despite transferring 0 tokens (exploitable by any unapproved caller via `transferFrom(victim, to, 0)` since OpenZeppelin's `_spendAllowance` permits `0` allowance for `0` amount).
   - When $\text{amount} = 1\text{ wei}$ and $r > P - 10^{30}$, $\text{fromSharesAfter} = S - 2$, burning 2 shares from the sender while a rebasing recipient gains 0 shares, permanently reducing `totalRebasingShares` and amplifying rounding losses across repeated transfers.

### Assumptions and preconditions

1. At least one rebasing account exists and `distribute()` has been called at least once so that `rebasingSharePrice() > START_REBASING_SHARE_PRICE` (`1e30`).
2. A rebasing account `account` holds `nShares` such that `(nShares * sharePrice) % START_REBASING_SHARE_PRICE != 0` and `(nShares * sharePrice) / START_REBASING_SHARE_PRICE >= ERC20.balanceOf(account)`.

### Proof of concept

Consider the setup from `testMintAfterSharePriceUpdate` in `ERC20RebaseDistributor.t.sol`:

1. `bobby` is minted `55e18` tokens (`55000000000000000000`) and calls `enterRebase()`.
   - `bobby.nShares = 55e18`, `totalRebasingShares = 55e18`, `sharePrice = 1e30`.
2. `distribute(1e18)` is called and `DISTRIBUTION_PERIOD` elapses.
   - `sharePrice` becomes $P = \lfloor (1\text{e}18 \cdot 10^{30} + 10^{30} \cdot 55\text{e}18) / 55\text{e}18 \rfloor = 1018181818181818181818181818181$.
   - For `bobby`, $S \cdot P = 55\text{e}18 \times 1018181818181818181818181818181 = 55999999999999999999 \cdot 10^{30} + 999999999955000000000000000000$.
   - Thus $q = 55999999999999999999 > 55\text{e}18$ (`ERC20.balanceOf(bobby)`) and remainder $r = 999999999955000000000000000000 > 0$.
3. **Underflow on 0-amount transfer to `bobby`**:
   - Any account calls `token.transfer(bobby, 0)`.
   - `toBalanceAfter = _shares2balance(55e18, P, 0, 55e18) = 55999999999999999999`.
   - `toSharesAfter = _balance2shares(55999999999999999999, P) = 54999999999999999999 = 55e18 - 1`.
   - Line 618 computes `sharesReceived = (55e18 - 1) - 55e18`, reverting with arithmetic underflow.
4. **Unapproved share burn on `bobby` via `transferFrom(bobby, anyAddress, 0)`**:
   - Any unapproved caller invokes `token.transferFrom(bobby, alice, 0)` during reward interpolation.
   - `ERC20.transferFrom` succeeds because `amount == 0`.
   - `fromBalanceBefore` becomes $q$, `fromBalanceAfter = q - 0 = q`, and `fromSharesAfter = _balance2shares(q, P) = 55e18 - 1`.
   - `bobby` loses 1 share per call whenever interpolation advances `q` by 1 wei.
5. **Zero-share attribution and sender double-share loss on 1-wei transfers**:
   - When `bobby` transfers `1` wei to another rebasing account, `fromBalanceAfter = q - 1 = 55999999999999999998`.
   - `_balance2shares(55999999999999999998, P) = 54999999999999999998 = 55e18 - 2` (`bobby` loses 2 shares for a 1-wei transfer), while the rebasing recipient gains `0` shares.

### Impact assessment

- **Severity**: Medium
- **Direct Technical Effects**:
  1. Zero-value transfers, `transferFrom`, or `mint` calls to a rebasing address with pending rebase rewards revert with an arithmetic underflow panic.
  2. Any unapproved caller can repeatedly invoke `transferFrom(victim, recipient, 0)` as rewards interpolate to strip shares and fractional yield from rebasing accounts.
  3. Small transfers (`amount * 1e30 < sharePrice`, including 1-wei transfers at any `sharePrice > 1e30`) attribute 0 shares to rebasing recipients (causing the transferred amount to be swallowed by subsequent rebase yield) while burning up to 2 shares from rebasing senders, permanently eroding user balances and `totalRebasingShares`.

### Remediation

Scale rebasing shares with higher precision than token balances (for example, by multiplying shares by an internal precision factor such as `1e12` in `_balance2shares` and dividing by it in `_shares2balance`), or round up in `_balance2shares` when recomputing shares from a truncated balance and avoid round-tripping existing shares on delta updates.

If keeping the existing callers unchanged, scaling shares by a `1e12` precision factor inside `_balance2shares` and `_shares2balance` ensures that 1 token wei corresponds to `~1e12` shares and prevents truncation loss on `shares -> balance -> shares` round trips:

```solidity
    /// @notice convert a balance to a number of shares
    function _balance2shares(
        uint256 balance,
        uint256 sharePrice
    ) internal pure returns (uint256) {
        return (balance * START_REBASING_SHARE_PRICE * 1e12) / sharePrice;
    }

    /// @notice convert a number of shares to a balance
    function _shares2balance(
        uint256 shares,
        uint256 sharePrice,
        uint256 deltaBalance,
        uint256 minBalance
    ) internal pure returns (uint256) {
        uint256 rebasedBalance = (shares * sharePrice) /
            (START_REBASING_SHARE_PRICE * 1e12) +
            deltaBalance;
        if (rebasedBalance < minBalance) {
            rebasedBalance = minBalance;
        }
        return rebasedBalance;
    }
```

## Finding 2: Permissionless `ERC20RebaseDistributor.distribute` Resets Interpolation `targetTimestamp` and Delays Rebase Rewards Indefinitely

### Synopsis

In `ERC20RebaseDistributor.distribute`, updating `__rebasingSharePrice` and `__unmintedRebaseRewards` unconditionally resets `targetTimestamp` to `block.timestamp + DISTRIBUTION_PERIOD` (30 days). Because `distribute` is permissionless and accepts `1` wei, an attacker (or frequent loan repayments) can repeatedly reset the interpolation window and delay rebase reward distribution indefinitely.

### Technical details

`ERC20RebaseDistributor` distributes profit to rebasing `CreditToken` holders by linearly interpolating `__rebasingSharePrice` and `__unmintedRebaseRewards` from `lastTimestamp` to `targetTimestamp` over `DISTRIBUTION_PERIOD` (30 days).

When `distribute(uint256 amount)` is called while an existing distribution is still being interpolated (`block.timestamp < targetTimestamp`) and `_rebasingSupply != 0`:

1. `_rebasingSharePrice` is computed at the current `block.timestamp` via `interpolatedValue(__rebasingSharePrice)`.
2. `endTimestamp` is set to `block.timestamp + DISTRIBUTION_PERIOD` (30 days in the future).
3. On lines 370–374, `__rebasingSharePrice` is overwritten with `lastTimestamp = block.timestamp`, `lastValue = _rebasingSharePrice`, `targetTimestamp = endTimestamp`, and `targetValue = newTargetSharePrice`.
4. Similarly, on lines 377–384, `__unmintedRebaseRewards` is overwritten with `lastTimestamp = block.timestamp`, `lastValue = _unmintedRebaseRewards`, `targetTimestamp = endTimestamp`, and `targetValue = __unmintedRebaseRewards.targetValue + amount`.

Because both `targetTimestamp` values are unconditionally reset to `block.timestamp + 30 days` regardless of the size of `amount` relative to the already-pending rewards (`__unmintedRebaseRewards.targetValue - _unmintedRebaseRewards`), any remaining unvested rewards from prior distributions have their vesting schedule pushed back by a full 30 days.

This creates two failure modes:
- **Malicious griefing**: `distribute(uint256 amount)` is `external` with no access control and only checks `amount != 0`. Any user can call `distribute(1)` (burning 1 wei of `CreditToken`) to reset `targetTimestamp` to `block.timestamp + 30 days` for all pending rewards. By calling `distribute(1)` periodically (e.g., once per day or every few blocks), an attacker slows the effective distribution rate to nearly zero at a negligible cost of a few hundred wei of `CreditToken` per year.
- **Natural protocol activity**: Every loan repayment (`LendingTerm._repay`) and partial repayment (`LendingTerm._partialRepay`) invokes `ProfitManager.notifyPnL`, which calls `CreditToken.distribute(amountForCredit)` whenever `amountForCredit > 0`. Frequent repayments continuously push `targetTimestamp` forward by 30 days, preventing earlier rebase rewards from fully vesting within the intended 30-day window.

### Assumptions and preconditions

1. At least one account has entered rebasing with a non-zero balance so that `_rebasingSupply != 0`.
2. A profit distribution has occurred via `distribute()`, leaving unvested rewards interpolating over `DISTRIBUTION_PERIOD` (30 days).
3. Either an arbitrary caller holds a few wei of `CreditToken` to call `distribute(1)`, or borrowers frequently repay loans through `LendingTerm`.

### Proof of concept

1. Suppose Alice holds `100,000e18` rebasing `CreditToken` (`_rebasingSupply = 100,000e18`).
2. At `t = 0`, a loan is repaid and `ProfitManager.notifyPnL` calls `credit.distribute(10,000e18)`.
   - `__rebasingSharePrice`: `lastTimestamp = 0`, `lastValue = 1e30`, `targetTimestamp = 30 days`, `targetValue = 1.1e30`.
   - `__unmintedRebaseRewards`: `lastTimestamp = 0`, `lastValue = 0`, `targetTimestamp = 30 days`, `targetValue = 10,000e18`.
3. At `t = 15 days`, `5,000e18` has vested (`balanceOf(alice) = 105,000e18`) and `5,000e18` remains pending for the next 15 days (`targetTimestamp = 30 days`).
4. At `t = 15 days`, an attacker calls `credit.distribute(1)` (1 wei of `CreditToken`).
   - `__rebasingSharePrice` and `__unmintedRebaseRewards` update `lastTimestamp = 15 days` and `targetTimestamp = 45 days`.
   - The remaining `5,000e18` rewards now take another 30 days (until `t = 45 days`) instead of 15 days.
5. If the attacker calls `credit.distribute(1)` once every hour (burning `24` wei per day):
   - Every hour, only `1 / 720` (`0.138%`) of the remaining pending rewards vests before the window is reset to a full 30 days (`720` hours).
   - After 30 days of calling `distribute(1)` hourly (costing `720` wei total), `0.998611^720 ≈ 36.7%` of the rewards are still unvested instead of `0%`, and calling `distribute(1)` every block (12 seconds) freezes reward vesting almost completely (`>99.9%` remains unvested).

### Impact assessment

- **Severity**: Medium.
- **Direct Technical Effect**: The interpolation period for `__rebasingSharePrice` and `__unmintedRebaseRewards` is repeatedly extended to `block.timestamp + 30 days`, starving `balanceOf(account)`, `rebasingSupply()`, and `totalSupply()` of vested yield.
- **Business Implications**: Rebasing `CreditToken` savers are deprived of their expected yield over the 30-day distribution period at virtually zero cost to an attacker (or as a side effect of frequent loan repayments). Furthermore, if losses occur (`ProfitManager.notifyPnL` with `amount < 0`), `creditTotalSupply = CreditToken(_credit).totalSupply()` (line 331 of `ProfitManager.sol`) excludes `pendingDistributedSupply()`, so delaying reward interpolation understates `creditTotalSupply` and causes a harsher downward adjustment to `creditMultiplier`.

### Remediation

Restrict `distribute` to authorized protocol contracts (e.g., `ProfitManager` via `onlyCoreRole`) or enforce a minimum distribution amount, and compute a weighted `targetTimestamp` based on the remaining unvested rewards versus the newly distributed `amount` so small distributions cannot push the vesting schedule of large existing rewards forward by a full `DISTRIBUTION_PERIOD`:

```solidity
        if (_rebasingSupply != 0) {
            uint256 _unmintedRebaseRewards = unmintedRebaseRewards();
            uint256 pendingRewards = __unmintedRebaseRewards.targetValue -
                _unmintedRebaseRewards;
            uint256 prevTargetTimestamp = block.timestamp >=
                __rebasingSharePrice.targetTimestamp
                ? block.timestamp
                : uint256(__rebasingSharePrice.targetTimestamp);
            uint256 endTimestamp = (prevTargetTimestamp *
                pendingRewards +
                (block.timestamp + DISTRIBUTION_PERIOD) *
                amount) / (pendingRewards + amount);

            uint256 newTargetSharePrice = (amount *
                START_REBASING_SHARE_PRICE +
                __rebasingSharePrice.targetValue *
                _totalRebasingShares) / _totalRebasingShares;
            __rebasingSharePrice = InterpolatedValue({
                lastTimestamp: SafeCastLib.safeCastTo32(block.timestamp),
                lastValue: SafeCastLib.safeCastTo224(_rebasingSharePrice),
                targetTimestamp: SafeCastLib.safeCastTo32(endTimestamp),
                targetValue: SafeCastLib.safeCastTo224(newTargetSharePrice)
            });

            // update unmintedRebaseRewards interpolation
            __unmintedRebaseRewards = InterpolatedValue({
                lastTimestamp: SafeCastLib.safeCastTo32(block.timestamp),
                lastValue: SafeCastLib.safeCastTo224(_unmintedRebaseRewards),
                targetTimestamp: SafeCastLib.safeCastTo32(endTimestamp),
                targetValue: __unmintedRebaseRewards.targetValue +
                    SafeCastLib.safeCastTo224(amount)
            });
        }
```

## Finding 3: Self-Transfer in `ERC20RebaseDistributor.transfer` Allows Rebasing Users to Mint Unbacked Tokens and Drain Unminted Rebase Rewards

### Synopsis

In `ERC20RebaseDistributor.transfer` (and `transferFrom`), transferring tokens to oneself (`msg.sender == to`) while rebasing causes stale memory caching of `rebasingState` and pre-transfer balance assumptions to double-count the transferred amount, minting free tokens to the caller, inflating their rebasing shares without updating `totalRebasingShares`, and draining `__unmintedRebaseRewards`.

### Technical details

In `ERC20RebaseDistributor.transfer(address to, uint256 amount)`, the function caches the rebasing states of both the sender and recipient from storage into memory at the beginning of execution (lines 559–560):

```solidity
RebasingState memory rebasingStateFrom = rebasingState[msg.sender];
RebasingState memory rebasingStateTo = rebasingState[to];
uint256 fromBalanceBefore = ERC20.balanceOf(msg.sender);
```

When a rebasing account (`isRebasing == 1`) calls `transfer(msg.sender, amount)` where `to == msg.sender`:

1. **Sender reward materialization (lines 566–581):** Any pending rebase rewards for `msg.sender` are minted via `ERC20._mint(msg.sender, mintAmount)`, bringing `ERC20.balanceOf(msg.sender)` and `fromBalanceBefore` up to `rebasedBalance`.
2. **Underlying ERC20 self-transfer (line 584):** `ERC20.transfer(to, amount)` executes `_transfer(msg.sender, msg.sender, amount)`. Because `from == to`, `_balances[msg.sender]` is decremented by `amount` and immediately incremented by `amount`, leaving `ERC20.balanceOf(msg.sender)` equal to `fromBalanceBefore` (`rebasedBalance`).
3. **Sender share deduction (lines 588–600):** `fromBalanceAfter` is computed as `fromBalanceBefore - amount`, `fromSharesAfter` is calculated from `fromBalanceAfter`, `sharesDelta` is decremented by `sharesSpent = rebasingStateFrom.nShares - fromSharesAfter`, and `rebasingState[msg.sender]` is updated in storage.
4. **Recipient share update and bogus reward minting (lines 603–632):** Because `rebasingStateTo` was cached in memory at line 560 prior to step 3, `rebasingStateTo.nShares` still holds the original full share count (`rebasingStateFrom.nShares`) rather than `fromSharesAfter`. Furthermore, `rawToBalanceAfter = ERC20.balanceOf(to)` is `fromBalanceBefore`. The function then computes:
   ```solidity
   uint256 toBalanceAfter = _shares2balance(
       rebasingStateTo.nShares,
       _rebasingSharePrice,
       amount,
       rawToBalanceAfter
   );
   ```
   Since `(rebasingStateTo.nShares * _rebasingSharePrice) / START_REBASING_SHARE_PRICE` equals `fromBalanceBefore`, adding `deltaBalance = amount` produces `toBalanceAfter = fromBalanceBefore + amount`.
5. **Consequences:**
   - `toSharesAfter` is calculated for `fromBalanceBefore + amount` and written to `rebasingState[to]`, increasing the caller's `nShares` by the share equivalent of `amount`.
   - `sharesReceived = toSharesAfter - rebasingStateTo.nShares` equals the shares for `amount`, which cancels out `-sharesSpent` in `sharesDelta`. Thus, `updateTotalRebasingShares` at line 638 receives `sharesDelta ≈ 0`, leaving `totalRebasingShares` unchanged while the caller's `nShares` increased.
   - At lines 626–631, `mintAmount = toBalanceAfter - rawToBalanceAfter` evaluates to `(fromBalanceBefore + amount) - fromBalanceBefore = amount`. The contract then calls `ERC20._mint(to, amount)` and `decreaseUnmintedRebaseRewards(amount)`, minting `amount` tokens directly to the caller and subtracting `amount` from `__unmintedRebaseRewards`.

The exact same flaw also exists in `ERC20RebaseDistributor.transferFrom(address from, address to, uint256 amount)` when `from == to`.

### Assumptions and preconditions

1. The caller holds a non-zero balance of tokens and has opted into rebasing (`rebasingState[msg.sender].isRebasing == 1`).
2. `unmintedRebaseRewards()` is at least `amount` (e.g., during or after a `distribute()` call while other rebasing accounts have not yet realized their rewards), so that `decreaseUnmintedRebaseRewards(amount)` does not revert on underflow.

### Proof of concept

1. Assume Alice and Bob each hold `100e18` tokens and both have entered rebasing (`isRebasing == 1`). At the initial share price `1e30`, both Alice and Bob have `nShares = 100e18`, and `totalRebasingShares = 200e18`.
2. `distribute(100e18)` is called, and `DISTRIBUTION_PERIOD` elapses. Now `_rebasingSharePrice = 1.5e30` and `unmintedRebaseRewards() = 100e18` (with `50e18` owed to Alice and `50e18` owed to Bob).
3. Alice calls `transfer(alice, 50e18)`:
   - In the sender block (lines 566–581), Alice's legitimate reward of `50e18` is minted to her (`ERC20._mint(alice, 50e18)`), bringing her raw balance `fromBalanceBefore` to `150e18` and reducing `unmintedRebaseRewards` from `100e18` to `50e18`.
   - `ERC20.transfer(alice, 50e18)` executes a self-transfer, leaving `ERC20.balanceOf(alice) = 150e18`.
   - In the sender share update (lines 588–600), `fromBalanceAfter = 100e18`, `fromSharesAfter = 66.66e18`, and `sharesSpent = 33.33e18`.
   - In the recipient block (lines 603–632), `rebasingStateTo.nShares` is still `100e18` and `rawToBalanceAfter = 150e18`. `toBalanceAfter` is calculated as `(100e18 * 1.5e30) / 1e30 + 50e18 = 200e18`.
   - `toSharesAfter` is calculated as `(200e18 * 1e30) / 1.5e30 = 133.33e18`, and `rebasingState[alice].nShares` is set to `133.33e18`.
   - `mintAmount = toBalanceAfter - rawToBalanceAfter = 200e18 - 150e18 = 50e18`.
   - `ERC20._mint(alice, 50e18)` mints an extra `50e18` tokens to Alice, and `decreaseUnmintedRebaseRewards(50e18)` reduces `unmintedRebaseRewards` to `0`.
4. Alice now has a balance of `200e18` (having stolen Bob's `50e18` of unminted rebase rewards) and `133.33e18` shares, while `totalRebasingShares` remains `200e18`.
5. When Bob attempts to exit rebasing, transfer, or burn his tokens, `decreaseUnmintedRebaseRewards` reverts due to arithmetic underflow because `unmintedRebaseRewards` is already `0`, permanently freezing Bob's funds.

### Impact assessment

- **Severity:** High / Critical.
- **Direct Technical Effects:**
  - Any rebasing user can steal 100% of the protocol's currently available `unmintedRebaseRewards` in one or more self-transfers.
  - The attacker's `nShares` is artificially inflated while `totalRebasingShares` remains unchanged, breaking the invariant that `totalRebasingShares` equals the sum of all rebasing users' `nShares` and allowing the attacker to earn a disproportionately large share of future distributions.
  - Because `unmintedRebaseRewards` is depleted, honest rebasing users cannot realize their accrued rewards in `_exitRebase`, `_burn`, `_mint`, `transfer`, or `transferFrom`, as `decreaseUnmintedRebaseRewards` will revert on underflow, locking their token balances.

### Remediation

Handle self-transfers (`msg.sender == to` in `transfer`, and `from == to` in `transferFrom`) explicitly—either by disallowing self-transfers, or by skipping the recipient share/reward recalculation when the sender and recipient are the same address (after materializing the sender's pending rebase rewards and executing `ERC20.transfer`).

```solidity
    function transfer(
        address to,
        uint256 amount
    ) public virtual override returns (bool) {
        // if `from` is rebasing, materialize the tokens from rebase to ensure
        // proper behavior in `ERC20.transfer()`.
        RebasingState memory rebasingStateFrom = rebasingState[msg.sender];
        RebasingState memory rebasingStateTo = rebasingState[to];
        uint256 fromBalanceBefore = ERC20.balanceOf(msg.sender);
        uint256 _rebasingSharePrice = (rebasingStateFrom.isRebasing == 1 ||
            rebasingStateTo.isRebasing == 1)
            ? rebasingSharePrice()
            : 0; // only SLOAD if at least one address is rebasing
        if (rebasingStateFrom.isRebasing == 1) {
            uint256 shares = uint256(rebasingStateFrom.nShares);
            uint256 rebasedBalance = _shares2balance(
                shares,
                _rebasingSharePrice,
                0,
                fromBalanceBefore
            );
            uint256 mintAmount = rebasedBalance - fromBalanceBefore;
            if (mintAmount != 0) {
                ERC20._mint(msg.sender, mintAmount);
                fromBalanceBefore += mintAmount;
                decreaseUnmintedRebaseRewards(mintAmount);
                emit RebaseReward(msg.sender, block.timestamp, mintAmount);
            }
        }

        // do ERC20.transfer()
        bool success = ERC20.transfer(to, amount);

        if (msg.sender == to) {
            return success;
        }

        // if `from` is rebasing, update its number of shares
        int256 sharesDelta;
        if (rebasingStateFrom.isRebasing == 1) {
            uint256 fromBalanceAfter = fromBalanceBefore - amount;
            uint256 fromSharesAfter = _balance2shares(
                fromBalanceAfter,
                _rebasingSharePrice
            );
            uint256 sharesSpent = rebasingStateFrom.nShares - fromSharesAfter;
            sharesDelta -= int256(sharesSpent);
            rebasingState[msg.sender] = RebasingState({
                isRebasing: 1,
                nShares: uint248(fromSharesAfter)
            });
        }

        // if `to` is rebasing, update its number of shares
        if (rebasingStateTo.isRebasing == 1) {
            // compute rebased balance
            uint256 rawToBalanceAfter = ERC20.balanceOf(to);
            uint256 toBalanceAfter = _shares2balance(
                rebasingStateTo.nShares,
                _rebasingSharePrice,
                amount,
                rawToBalanceAfter
            );

            // update number of shares
            uint256 toSharesAfter = _balance2shares(
                toBalanceAfter,
                _rebasingSharePrice
            );
            uint256 sharesReceived = toSharesAfter - rebasingStateTo.nShares;
            sharesDelta += int256(sharesReceived);
            rebasingState[to] = RebasingState({
                isRebasing: 1,
                nShares: uint248(toSharesAfter)
            });

            // "realize" unminted rebase rewards
            uint256 mintAmount = toBalanceAfter - rawToBalanceAfter;
            if (mintAmount != 0) {
                ERC20._mint(to, mintAmount);
                decreaseUnmintedRebaseRewards(mintAmount);
                emit RebaseReward(to, block.timestamp, mintAmount);
            }
        }

        // if `from` or `to` was rebasing, update the total number of shares
        if (
            rebasingStateFrom.isRebasing == 1 || rebasingStateTo.isRebasing == 1
        ) {
            updateTotalRebasingShares(_rebasingSharePrice, sharesDelta);
        }

        return success;
    }
```

## Finding 4: Self-Transfer in `ERC20RebaseDistributor.transfer` Mints Unbacked Tokens and Inflates Rebasing Shares

### Synopsis

In `ERC20RebaseDistributor.transfer`, transferring tokens to oneself (`to == msg.sender`) while rebasing causes stale memory state (`rebasingStateTo`) and an invalid post-transfer balance assumption (`fromBalanceBefore - amount`) to double-count the transferred amount, minting `amount` tokens from unminted rebase rewards and inflating the caller's rebasing shares.

### Technical details

In `ERC20RebaseDistributor.transfer(address to, uint256 amount)`, the contract caches the rebasing states of both `msg.sender` and `to` in memory before executing the transfer:

```solidity
RebasingState memory rebasingStateFrom = rebasingState[msg.sender];
RebasingState memory rebasingStateTo = rebasingState[to];
uint256 fromBalanceBefore = ERC20.balanceOf(msg.sender);
```

When `msg.sender` is rebasing (`rebasingStateFrom.isRebasing == 1`) and performs a self-transfer (`to == msg.sender`), the following sequence occurs:

1. Any pending rebase rewards for `msg.sender` are minted on lines 576–580, updating both `ERC20.balanceOf(msg.sender)` and `fromBalanceBefore` to the full `rebasedBalance` (`(rebasingStateFrom.nShares * _rebasingSharePrice) / START_REBASING_SHARE_PRICE`).
2. On line 584, `ERC20.transfer(to, amount)` executes `_transfer(msg.sender, msg.sender, amount)`. In OpenZeppelin's `ERC20._transfer`, `_balances[msg.sender]` is decremented by `amount` and immediately incremented by `amount`, leaving `ERC20.balanceOf(msg.sender)` unchanged at `fromBalanceBefore`.
3. On lines 588–600, the sender's share reduction is computed under the assumption that `msg.sender != to`:
   ```solidity
   uint256 fromBalanceAfter = fromBalanceBefore - amount;
   uint256 fromSharesAfter = _balance2shares(
       fromBalanceAfter,
       _rebasingSharePrice
   );
   uint256 sharesSpent = rebasingStateFrom.nShares - fromSharesAfter;
   sharesDelta -= int256(sharesSpent);
   rebasingState[msg.sender] = RebasingState({
       isRebasing: 1,
       nShares: uint248(fromSharesAfter)
   });
   ```
4. Immediately after, on lines 603–632, the recipient logic executes using the stale in-memory `rebasingStateTo` (which still holds the pre-transfer `nShares` of `msg.sender`) and the actual `ERC20.balanceOf(to)` (which is still `fromBalanceBefore`):
   ```solidity
   uint256 rawToBalanceAfter = ERC20.balanceOf(to);
   uint256 toBalanceAfter = _shares2balance(
       rebasingStateTo.nShares,
       _rebasingSharePrice,
       amount,
       rawToBalanceAfter
   );
   ```
   Because `rebasingStateTo.nShares` corresponds to `fromBalanceBefore`, `_shares2balance` computes `toBalanceAfter = fromBalanceBefore + amount`.
5. Consequently:
   - `toSharesAfter = _balance2shares(fromBalanceBefore + amount, _rebasingSharePrice)` overwrites `rebasingState[msg.sender].nShares`, increasing the caller's shares by the equivalent of `amount` tokens.
   - `sharesReceived = toSharesAfter - rebasingStateTo.nShares` equals `sharesSpent` (up to 1 wei of rounding), so `sharesDelta = -int256(sharesSpent) + int256(sharesReceived) == 0`. Thus, `totalRebasingShares` is not increased even though `rebasingState[msg.sender].nShares` grew by `sharesReceived`.
   - On lines 626–631, `mintAmount = toBalanceAfter - rawToBalanceAfter = (fromBalanceBefore + amount) - fromBalanceBefore = amount`. The contract then calls `ERC20._mint(to, amount)` and `decreaseUnmintedRebaseRewards(amount)`, directly minting `amount` extra tokens to `msg.sender` out of the global `__unmintedRebaseRewards` pool.

### Assumptions and preconditions

1. The attacker holds a non-zero balance of tokens and has opted into rebasing (`isRebasing == 1`).
2. There are unminted rebase rewards in the contract (`unmintedRebaseRewards() > 0`, e.g., during or after a `distribute()` call before other rebasing users have materialized their rewards).

### Proof of concept

1. Suppose Alice and Bob each hold `100e18` tokens and both have called `enterRebase()`. `totalRebasingShares` is `200e18` and `rebasingSharePrice()` is `1e30`.
2. `distribute(200e18)` is called, and `DISTRIBUTION_PERIOD` elapses. Now `rebasingSharePrice()` is `2e30`, `unmintedRebaseRewards()` is `200e18`, and Alice and Bob are each entitled to `100e18` in rebase rewards (so each has `balanceOf == 200e18`).
3. Alice calls `transfer(alice, 100e18)`:
   - Lines 566–581 materialize Alice's `100e18` pending reward: `ERC20._mint(alice, 100e18)` is called, `fromBalanceBefore` becomes `200e18`, and `unmintedRebaseRewards` decreases from `200e18` to `100e18` (Bob's unmaterialized reward).
   - Line 584 calls `ERC20.transfer(alice, 100e18)`, which leaves `ERC20.balanceOf(alice)` at `200e18`.
   - Lines 588–600 compute `fromBalanceAfter = 200e18 - 100e18 = 100e18`, `fromSharesAfter = 50e18`, `sharesSpent = 50e18`, and `sharesDelta = -50e18`.
   - Lines 603–632 read `rawToBalanceAfter = ERC20.balanceOf(alice) = 200e18` and compute `toBalanceAfter = (100e18 * 2e30 / 1e30) + 100e18 = 300e18`.
   - `toSharesAfter` is calculated as `_balance2shares(300e18, 2e30) = 150e18`, `sharesReceived = 150e18 - 100e18 = 50e18`, and `sharesDelta` becomes `-50e18 + 50e18 = 0`.
   - `mintAmount = 300e18 - 200e18 = 100e18` is minted to Alice via `ERC20._mint(alice, 100e18)`, and `decreaseUnmintedRebaseRewards(100e18)` reduces `unmintedRebaseRewards` to `0`.
4. After the single self-transfer, Alice has `300e18` raw tokens and `150e18` rebasing shares (while `totalRebasingShares` is still `200e18`), having stolen Bob's entire `100e18` unminted rebase reward. When Bob subsequently tries to exit rebase, transfer, or burn tokens, his transaction reverts with an arithmetic underflow in `decreaseUnmintedRebaseRewards`.

### Impact assessment

**Severity: High**

Any rebasing token holder can repeatedly call `transfer(msg.sender, amount)` to:
1. Steal all unminted rebase rewards (`__unmintedRebaseRewards`) belonging to other rebasing users.
2. Inflate their own `rebasingState[msg.sender].nShares` without increasing `totalRebasingShares`, permanently corrupting share accounting and allowing them to capture a disproportionate share of future distributions.
3. Cause a denial of service (revert due to underflow in `decreaseUnmintedRebaseRewards`) for legitimate rebasing users when they attempt to transfer, burn, or exit rebase.

### Remediation

In `ERC20RebaseDistributor.transfer` (and similarly in `transferFrom`), handle self-transfers by only updating `from` and `to` share states when `to != msg.sender`, or by returning early after materializing `msg.sender`'s rebase rewards and performing `ERC20.transfer`:

```solidity
        if (rebasingStateFrom.isRebasing == 1) {
            uint256 shares = uint256(rebasingStateFrom.nShares);
            uint256 rebasedBalance = _shares2balance(
                shares,
                _rebasingSharePrice,
                0,
                fromBalanceBefore
            );
            uint256 mintAmount = rebasedBalance - fromBalanceBefore;
            if (mintAmount != 0) {
                ERC20._mint(msg.sender, mintAmount);
                fromBalanceBefore += mintAmount;
                decreaseUnmintedRebaseRewards(mintAmount);
                emit RebaseReward(msg.sender, block.timestamp, mintAmount);
            }
        }

        // do ERC20.transfer()
        bool success = ERC20.transfer(to, amount);

        if (to == msg.sender) {
            return success;
        }

        // if `from` is rebasing, update its number of shares
        int256 sharesDelta;
        if (rebasingStateFrom.isRebasing == 1) {
            uint256 fromBalanceAfter = fromBalanceBefore - amount;
            uint256 fromSharesAfter = _balance2shares(
                fromBalanceAfter,
                _rebasingSharePrice
            );
            uint256 sharesSpent = rebasingStateFrom.nShares - fromSharesAfter;
            sharesDelta -= int256(sharesSpent);
            rebasingState[msg.sender] = RebasingState({
                isRebasing: 1,
                nShares: uint248(fromSharesAfter)
            });
        }
```

## Finding 5: Precision Loss in Share-to-Balance-to-Share Round Trip in `ERC20RebaseDistributor.transfer` Causes Share Delta Underflow and Share Erosion

### Synopsis

In `ERC20RebaseDistributor.transfer`, updating a rebasing recipient's shares performs a truncated share-to-balance-to-share round trip (`_shares2balance` followed by `_balance2shares`). When `_rebasingSharePrice > START_REBASING_SHARE_PRICE`, double truncation causes `toSharesAfter` to be smaller than `rebasingStateTo.nShares` on zero or dust transfers, reverting with arithmetic underflow or eroding rebasing shares.

### Technical details

In `ERC20RebaseDistributor.transfer` (lines 603–618), when the recipient `to` is in rebasing mode (`rebasingStateTo.isRebasing == 1`), the contract updates `to`'s rebasing shares after `ERC20.transfer(to, amount)` has executed:

```solidity
if (rebasingStateTo.isRebasing == 1) {
    // compute rebased balance
    uint256 rawToBalanceAfter = ERC20.balanceOf(to);
    uint256 toBalanceAfter = _shares2balance(
        rebasingStateTo.nShares,
        _rebasingSharePrice,
        amount,
        rawToBalanceAfter
    );

    // update number of shares
    uint256 toSharesAfter = _balance2shares(
        toBalanceAfter,
        _rebasingSharePrice
    );
    uint256 sharesReceived = toSharesAfter - rebasingStateTo.nShares;
```

This logic suffers from flawed fixed-point share arithmetic across four interconnected aspects:

1. **Precision loss in balance-to-share round trips:**
   `_shares2balance` computes `rebasedBalance = (nShares * _rebasingSharePrice) / START_REBASING_SHARE_PRICE + amount`. Let `nShares * _rebasingSharePrice = q * START_REBASING_SHARE_PRICE + r`, where `0 <= r < START_REBASING_SHARE_PRICE`. The integer division discards the remainder `r`. Immediately afterward, `_balance2shares` converts `toBalanceAfter` back into shares:
   $$\text{toSharesAfter} = \left\lfloor \frac{(q + \text{amount}) \times 10^{30}}{\text{\_rebasingSharePrice}} \right\rfloor = \left\lfloor \frac{\text{nShares} \times \text{\_rebasingSharePrice} - r + \text{amount} \times 10^{30}}{\text{\_rebasingSharePrice}} \right\rfloor$$

2. **Rounding-induced non-monotonic share deltas (Underflow DoS):**
   Because `_rebasingSharePrice` starts at `START_REBASING_SHARE_PRICE` ($10^{30}$) and only increases after `distribute()`, any active rebase state has `_rebasingSharePrice > 10^{30}`. Whenever `r > 0` (i.e., `(nShares * _rebasingSharePrice) % 1e30 != 0`) and `to` has unmaterialized rebase rewards such that `q >= rawToBalanceBefore`, a transfer with `amount == 0` yields:
   $$(\text{nShares} - 1) \times \text{\_rebasingSharePrice} < \text{nShares} \times \text{\_rebasingSharePrice} - r < \text{nShares} \times \text{\_rebasingSharePrice}$$
   Thus, `toSharesAfter = nShares - 1 < rebasingStateTo.nShares`. On line 618, `uint256 sharesReceived = toSharesAfter - rebasingStateTo.nShares` underflows and reverts the transaction.

3. **Asymmetric rounding across transfer participants:**
   When `from` and `to` are both rebasing and `amount > 0`, `fromSharesAfter = _balance2shares(fromBalanceBefore - amount, _rebasingSharePrice)` rounds down, which rounds `sharesSpent = rebasingStateFrom.nShares - fromSharesAfter` **up**. Conversely, `toSharesAfter` rounds **down** both in `_shares2balance` and `_balance2shares`, which rounds `sharesReceived` **down**. For small transfers where `amount * 1e30 < _rebasingSharePrice`, `from` loses 1 share while `to` gains 0 shares, resulting in net destruction of `totalRebasingShares` (`sharesDelta < 0`) during a transfer between two rebasing accounts.

4. **Inconsistency between minimum-balance floors and share-backed balances:**
   After `toSharesAfter` is rounded down, lines 626–630 mint `mintAmount = toBalanceAfter - rawToBalanceAfter` to `to`, raising `ERC20.balanceOf(to)` to `toBalanceAfter`. However, the share-backed balance `(toSharesAfter * _rebasingSharePrice) / 1e30` is strictly less than `toBalanceAfter`. In subsequent calls, `_shares2balance` clamps the result to `minBalance` (`ERC20.balanceOf(to)`), masking the lost share backing until future rebase rewards or transfers are absorbed by the gap between `minBalance` and the lower share-backed balance.

### Assumptions and preconditions

- At least one reward distribution has occurred (`distribute()`), so `_rebasingSharePrice > START_REBASING_SHARE_PRICE` ($10^{30}$).
- The recipient account `to` has `isRebasing == 1` and `nShares > 0` such that `(nShares * _rebasingSharePrice) % START_REBASING_SHARE_PRICE != 0`.
- For the underflow revert: `to` has at least 1 wei of unminted rebase rewards (`(nShares * _rebasingSharePrice) / 1e30 >= ERC20.balanceOf(to)`), and `transfer(to, 0)` is called.
- For share erosion: transfers of small amounts (`0 < amount < _rebasingSharePrice / 1e30`) occur to or between rebasing accounts.

### Proof of concept

1. **Initial Setup:**
   - Alice enters rebase (`enterRebase()`) and is minted `100e18` tokens (`10^20` wei).
   - At initial share price `START_REBASING_SHARE_PRICE = 1e30`, Alice's state is `rawBalance = 100e18` and `nShares = 100e18`.

2. **Reward Distribution:**
   - `distribute(50e18)` is called to distribute `50e18` tokens over `DISTRIBUTION_PERIOD` (`30 days = 2,592,000` seconds).
   - `newTargetSharePrice = (50e18 * 1e30 + 1e30 * 100e18) / 100e18 = 1.5e30`.
   - Advance `block.timestamp` by `1` second into the distribution period.
   - `_rebasingSharePrice = 1e30 + (0.5e30 * 1) / 2592000 = 1000000192901234567901234567901`.

3. **Round-Trip Precision Loss & Underflow in `transfer`:**
   - Any address calls `transfer(alice, 0)`.
   - In `transfer`:
     - `rawToBalanceAfter = 100e18 = 100000000000000000000`.
     - `nShares * _rebasingSharePrice = 10^20 * 1000000192901234567901234567901`.
     - Notice `(nShares * _rebasingSharePrice) % 1e30 = 123456790123456790100000000000` ($r > 0$).
     - `toBalanceAfter = _shares2balance(100e18, _rebasingSharePrice, 0, 100e18) = 100000019290123456790`.
     - `toSharesAfter = _balance2shares(100000019290123456790, 1000000192901234567901234567901)`:
       $$\text{toSharesAfter} = \left\lfloor \frac{100000019290123456790 \times 10^{30}}{1000000192901234567901234567901} \right\rfloor = 10^{20} - 1 = 99999999999999999999$$
     - Line 618 executes `sharesReceived = toSharesAfter - rebasingStateTo.nShares = (10^20 - 1) - 10^20`, which reverts due to arithmetic underflow.

4. **Asymmetric Rounding & Share Erosion on Non-Zero Transfers:**
   - Suppose `_rebasingSharePrice = 2e30` (after a 100% reward distribution has been realized so `from` and `to` each have `rawBalance = 200e18` and `nShares = 100e18`).
   - `from` calls `transfer(to, 1)` (transferring 1 wei to `to`):
     - `fromBalanceAfter = 200e18 - 1`.
     - `fromSharesAfter = ((200e18 - 1) * 1e30) / 2e30 = 100e18 - 1`.
     - `sharesSpent = 100e18 - (100e18 - 1) = 1`.
     - For `to`: `toBalanceAfter = (100e18 * 2e30) / 1e30 + 1 = 200e18 + 1`.
     - `toSharesAfter = ((200e18 + 1) * 1e30) / 2e30 = 100e18`.
     - `sharesReceived = 100e18 - 100e18 = 0`.
   - `totalRebasingShares` decreases by `1` share (worth 2 wei), `from`'s share-backed balance drops by 2 wei while `to`'s share-backed balance increases by 0 wei, and `to`'s `rawToBalanceAfter` (`200e18 + 1`) exceeds its share-backed balance (`200e18`), causing the next 1 wei of rebase reward accrued by `to` to be lost to the `minBalance` floor.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effects:**
  1. Zero-amount transfers (`transfer(to, 0)`) to rebasing accounts with unmaterialized rewards revert with an arithmetic underflow, breaking ERC20 standard compliance and any protocol flows that may perform 0-value transfers.
  2. Small transfers between or to rebasing accounts when `_rebasingSharePrice > 1e30` cause net share destruction (`sharesSpent > sharesReceived`), desynchronizing `to`'s `rawBalance` floor from its share-backed balance, eroding user rebase rewards, and skewing `totalRebasingShares` and the interpolated `targetValue` of `__rebasingSharePrice`.

### Remediation

Materialize the recipient's pending rebase rewards using its existing shares before computing the share increase, or ensure that converting `toBalanceAfter` to `toSharesAfter` cannot decrease `to`'s share count below `rebasingStateTo.nShares` and only converts the net balance change above the already-backed share balance.

```solidity
        // if `to` is rebasing, update its number of shares
        if (rebasingStateTo.isRebasing == 1) {
            // compute rebased balance
            uint256 rawToBalanceAfter = ERC20.balanceOf(to);
            uint256 toBalanceAfter = _shares2balance(
                rebasingStateTo.nShares,
                _rebasingSharePrice,
                amount,
                rawToBalanceAfter
            );

            // update number of shares without losing existing share precision
            uint256 toSharesAfter = amount == 0
                ? uint256(rebasingStateTo.nShares)
                : _balance2shares(toBalanceAfter, _rebasingSharePrice);
            if (toSharesAfter < rebasingStateTo.nShares) {
                toSharesAfter = rebasingStateTo.nShares;
            }
            uint256 sharesReceived = toSharesAfter - rebasingStateTo.nShares;
            sharesDelta += int256(sharesReceived);
            rebasingState[to] = RebasingState({
                isRebasing: 1,
                nShares: uint248(toSharesAfter)
            });

            // "realize" unminted rebase rewards
            uint256 mintAmount = toBalanceAfter - rawToBalanceAfter;
            if (mintAmount != 0) {
                ERC20._mint(to, mintAmount);
                decreaseUnmintedRebaseRewards(mintAmount);
                emit RebaseReward(to, block.timestamp, mintAmount);
            }
        }
```
