# Audit report — 2023-12-ethereumcreditguild/H-01

## Finding 1: Deferred Rebase Distribution in `ProfitManager.notifyPnL` Causes `CreditToken` Supply Inconsistencies and Unintended Token Burn

### Synopsis

In `ProfitManager.notifyPnL`, calling `CreditToken.distribute(amountForCredit)` immediately burns `amountForCredit` from `ProfitManager` while either deferring its re-addition to `totalSupply()` over `DISTRIBUTION_PERIOD` or omitting it entirely when `_rebasingSupply == 0`, corrupting `totalBorrowedCredit()` and `creditMultiplier` loss calculations.

### Technical details

When positive profit is reported to `ProfitManager.notifyPnL(address gauge, int256 amount)`, the lender share `amountForCredit` is distributed at lines 378–380 of `src/governance/ProfitManager.sol`:

```solidity
// distribute to lenders
if (amountForCredit != 0) {
    CreditToken(_credit).distribute(amountForCredit);
}
```

In `ERC20RebaseDistributor.distribute(uint256 amount)` (`src/tokens/ERC20RebaseDistributor.sol` lines 338–386):
1. Line 342 unconditionally executes `_burn(msg.sender, amount)`, which immediately reduces `ERC20._totalSupply` by `amountForCredit`.
2. At line 362, `__rebasingSharePrice` and `__unmintedRebaseRewards` are only updated `if (_rebasingSupply != 0)`:
   - **When `_rebasingSupply == 0` (no accounts currently rebasing):** `amountForCredit` is burned at line 342, but `__unmintedRebaseRewards` is never incremented. As a result, those `CREDIT` tokens are permanently destroyed rather than distributed, permanently reducing both `CreditToken.totalSupply()` and `CreditToken.targetTotalSupply()` by `amountForCredit`. Because those tokens were not burned via principal repayment or redeemed from the PSM, `ProfitManager.totalBorrowedCredit()` (`CreditToken(credit).targetTotalSupply() - SimplePSM(psm).redeemableCredit()` at line 172) becomes understated by `amountForCredit` and can revert with an arithmetic underflow if `targetTotalSupply()` drops below `redeemableCredit()`.
   - **When `_rebasingSupply != 0`:** `_burn(msg.sender, amount)` immediately deducts `amountForCredit` from `ERC20._totalSupply`, whereas `unmintedRebaseRewards()` only interpolates linearly from its prior value up to `__unmintedRebaseRewards.targetValue + amount` over `DISTRIBUTION_PERIOD` (lines 377–384). Consequently, `CreditToken.totalSupply()` immediately decreases by `amountForCredit` upon `notifyPnL` and remains understated throughout the `DISTRIBUTION_PERIOD`. If a subsequent loss is reported in `ProfitManager.notifyPnL` (lines 331–334), `creditTotalSupply = CreditToken(_credit).totalSupply()` excludes the unvested portion of distributed profits, causing `newCreditMultiplier` to be discounted more aggressively than warranted by actual bad debt.

### Assumptions and preconditions

1. Profit sharing is configured such that `creditSplit > 0`, making `amountForCredit > 0` when a positive PnL is reported via `notifyPnL`.
2. Either no `CREDIT` holders have opted into rebasing (`_rebasingSupply == 0`) when `notifyPnL` is called, or a loss (`amount < 0`) is reported to `notifyPnL` while a previous profit distribution is still vesting within `DISTRIBUTION_PERIOD`.

### Proof of concept

1. Assume 1,000 `CREDIT` is minted via `SimplePSM` (`redeemableCredit() == 1000e18`, `totalSupply() == 1000e18`, `targetTotalSupply() == 1000e18`), and 100 `CREDIT` is borrowed from a `LendingTerm` (`targetTotalSupply() == 1100e18`, `totalBorrowedCredit() == 100e18`).
2. Suppose `creditSplit == 1e18` (100% of profit goes to `CREDIT` lenders) and initially no `CREDIT` holder has entered rebasing (`_rebasingSupply == 0`).
3. The borrower repays their 100 `CREDIT` principal plus 10 `CREDIT` interest (acquired from the PSM Minter) via `LendingTerm._repay`.
4. `LendingTerm._repay` burns 100 `CREDIT` principal (`targetTotalSupply() == 1000e18`) and transfers 10 `CREDIT` interest to `ProfitManager`, then calls `ProfitManager.notifyPnL(address(this), 10e18)`.
5. `ProfitManager.notifyPnL` calls `CreditToken(_credit).distribute(10e18)` at line 379.
6. Inside `distribute(10e18)`, `_burn(msg.sender, 10e18)` burns the 10 `CREDIT` from `ProfitManager`, reducing `ERC20._totalSupply` (and `targetTotalSupply()`) to `990e18`. Because `_rebasingSupply == 0`, `__unmintedRebaseRewards` is not updated.
7. The 10 `CREDIT` of interest is permanently lost, and `ProfitManager.totalBorrowedCredit()` now computes `targetTotalSupply() (990e18) - redeemableCredit() (1000e18)`, reverting due to arithmetic underflow.
8. Alternatively, if `_rebasingSupply > 0` at step 5, `CreditToken.totalSupply()` immediately drops from `1000e18` to `990e18` at the timestamp of `distribute(10e18)`. If another loan suffers a bad debt loss of `99e18` in the same block (after depleting any surplus buffer), `ProfitManager.notifyPnL` computes `newCreditMultiplier = creditMultiplier * (990e18 - 99e18) / 990e18 = 0.9e18` instead of `(1000e18 - 99e18) / 1000e18 = 0.901e18`, over-penalizing `creditMultiplier`.

### Impact assessment

- **Severity:** Medium / High.
- **Technical Impact:**
  - When `_rebasingSupply == 0`, `amountForCredit` is permanently burned instead of being distributed, violating the profit distribution invariant and reducing `targetTotalSupply()` below actual circulating/redeemable obligations, which can cause `ProfitManager.totalBorrowedCredit()` to underreport borrowed credit or revert with an underflow.
  - When `_rebasingSupply != 0`, `CreditToken.totalSupply()` is understated by the unvested portion of `amountForCredit` during `DISTRIBUTION_PERIOD`, causing any bad debt loss processed during that window to excessively mark down `creditMultiplier`.

### Remediation

In `ProfitManager.notifyPnL`, only call `CreditToken(_credit).distribute(amountForCredit)` when `CreditToken(_credit).rebasingSupply() != 0` (redirecting `amountForCredit` to `surplusBuffer` when no holders are rebasing), and use `targetTotalSupply()` when computing `creditMultiplier` markdowns so unvested rebase rewards are accounted for:

```solidity
            // distribute to lenders
            if (amountForCredit != 0) {
                if (CreditToken(_credit).rebasingSupply() != 0) {
                    CreditToken(_credit).distribute(amountForCredit);
                } else {
                    surplusBuffer += amountForCredit;
                    emit SurplusBufferUpdate(
                        block.timestamp,
                        surplusBuffer
                    );
                }
            }
```

## Finding 2: Unconditional `amountForGuild` deduction in `ProfitManager.notifyPnL` permanently locks profit when gauge weight is zero

### Synopsis

In `ProfitManager.notifyPnL`, `amountForGuild` is unconditionally calculated and subtracted from `amountForCredit` without checking whether the gauge has a non-zero weight. When `_gaugeWeight == 0`, the gauge profit index is not updated and `amountForGuild` is never redistributed, permanently locking those `CREDIT` tokens in `ProfitManager`.

### Technical details

When positive profit (`amount > 0`) is reported to `ProfitManager.notifyPnL(address gauge, int256 amount)`, the caller (`LendingTerm`) first transfers `amount` of `CREDIT` tokens to `ProfitManager`. `notifyPnL` then divides `uint256(amount)` into four parts (`src/governance/ProfitManager.sol` lines 346-358):

- `amountForSurplusBuffer`
- `amountForGuild` (lines 349-350)
- `amountForOther` (lines 352-353)
- `amountForCredit = uint256(amount) - amountForSurplusBuffer - amountForGuild - amountForOther` (lines 355-358)

Each share is then distributed:
1. `amountForSurplusBuffer` increments `surplusBuffer` (lines 361-367).
2. `amountForOther` is transferred to `_profitSharingConfig.otherRecipient` (lines 370-375).
3. `amountForCredit` is distributed to rebasing `CREDIT` holders via `CreditToken(_credit).distribute(amountForCredit)` (lines 378-380).
4. `amountForGuild` is intended to remain in `ProfitManager` to back future reward claims via `claimGaugeRewards`, which uses `gaugeProfitIndex[gauge]` (lines 383-401).

However, at lines 388-400, `notifyPnL` checks `uint256 _gaugeWeight = uint256(GuildToken(guild).getGaugeWeight(gauge))` and only updates `gaugeProfitIndex[gauge]` if `_gaugeWeight != 0`. When `_gaugeWeight == 0`, `gaugeProfitIndex[gauge]` is not updated because there are no `GUILD` stakers voting for `gauge`, yet `amountForGuild` was already deducted from `amountForCredit` at lines 349-358. As a result, `amountForGuild` is neither distributed to `CREDIT` holders nor added to `surplusBuffer`, nor can it ever be claimed via `claimGaugeRewards`. Those `CREDIT` tokens remain permanently stranded in `ProfitManager`, violating the protocol's profit distribution conservation invariant.

### Assumptions and preconditions

1. `profitSharingConfig.guildSplit` is configured to a non-zero value (e.g., `0.01e18` in `GIP_0.sol` or `0.5e18` in `ProfitManager.t.sol`).
2. A gauge (`LendingTerm`) has active loans or active collateral auctions while its `GuildToken.getGaugeWeight(gauge)` is `0` (for example, after a previous loss caused all `GUILD` stakers on that gauge to be slashed via `applyGaugeLoss`, removing all weight from the gauge while remaining loans or auctions are still open).
3. One of the remaining loans is repaid (or partially repaid, or its auction settles via `onBid` with `creditFromBidder > principal`), generating positive interest and calling `ProfitManager.notifyPnL(gauge, int256(interest))`.

### Proof of concept

1. `ProfitSharingConfig` is set with `guildSplit = 0.5e18` (50%) and `creditSplit = 0.5e18` (50%).
2. `gauge1` has two loans open: Loan A and Loan B. Alice is the sole `GUILD` voter on `gauge1` with weight `50e18`.
3. Loan A defaults and incurs a loss (`notifyPnL(gauge1, -loss)`), setting `lastGaugeLoss[gauge1] = block.timestamp`.
4. `GuildToken.applyGaugeLoss(gauge1, alice)` is called. Alice's `50e18` weight on `gauge1` is decremented and burned, bringing `GuildToken.getGaugeWeight(gauge1)` to `0`.
5. Loan B is later repaid with `20e18` `CREDIT` in interest. `LendingTerm._repay` transfers `20e18` `CREDIT` to `ProfitManager` and calls `ProfitManager.notifyPnL(gauge1, 20e18)`.
6. In `notifyPnL`:
   - `amountForGuild` is computed as `10e18` (lines 349-350).
   - `amountForCredit` is computed as `20e18 - 10e18 = 10e18` (lines 355-358).
   - `10e18` `CREDIT` is distributed to `CREDIT` holders via `CreditToken.distribute(10e18)`.
   - At lines 388-391, `_gaugeWeight` is `0`, so `gaugeProfitIndex[gauge1]` is not updated.
7. The `10e18` `CREDIT` allocated to `amountForGuild` is never distributed to `CREDIT` holders, `surplusBuffer`, or `GUILD` voters, and remains permanently locked in `ProfitManager`.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: Conservation failure between profit share allocations and token distributions in `ProfitManager.notifyPnL`. Whenever a gauge with zero weight produces profit, the `guildSplit` portion of the profit is permanently frozen in the `ProfitManager` contract.
- **Business Impact**: Direct loss of yield/protocol revenue intended for `CREDIT` lenders or the protocol's `surplusBuffer`.

### Remediation

Check the gauge weight before computing `amountForGuild` (or set `amountForGuild = 0` when `_gaugeWeight == 0`) so that if no `GUILD` holders are voting for the gauge, the `guildSplit` share is not deducted from `amountForCredit` (or is redirected to `surplusBuffer`).

```solidity
            ProfitSharingConfig
                memory _profitSharingConfig = profitSharingConfig;

            uint256 _gaugeWeight = uint256(
                GuildToken(guild).getGaugeWeight(gauge)
            );

            uint256 amountForSurplusBuffer = (uint256(amount) *
                uint256(_profitSharingConfig.surplusBufferSplit)) / 1e9;

            uint256 amountForGuild = _gaugeWeight == 0
                ? 0
                : (uint256(amount) *
                    uint256(_profitSharingConfig.guildSplit)) / 1e9;

            uint256 amountForOther = (uint256(amount) *
                uint256(_profitSharingConfig.otherSplit)) / 1e9;

            uint256 amountForCredit = uint256(amount) -
                amountForSurplusBuffer -
                amountForGuild -
                amountForOther;

            // distribute to surplus buffer
            if (amountForSurplusBuffer != 0) {
                surplusBuffer = _surplusBuffer + amountForSurplusBuffer;
                emit SurplusBufferUpdate(
                    block.timestamp,
                    _surplusBuffer + amountForSurplusBuffer
                );
            }

            // distribute to other
            if (amountForOther != 0) {
                CreditToken(_credit).transfer(
                    _profitSharingConfig.otherRecipient,
                    amountForOther
                );
            }

            // distribute to lenders
            if (amountForCredit != 0) {
                CreditToken(_credit).distribute(amountForCredit);
            }

            // distribute to the guild
            if (amountForGuild != 0) {
                uint256 _gaugeProfitIndex = gaugeProfitIndex[gauge];
                if (_gaugeProfitIndex == 0) {
                    _gaugeProfitIndex = 1e18;
                }
                gaugeProfitIndex[gauge] =
                    _gaugeProfitIndex +
                    (amountForGuild * 1e18) /
                    _gaugeWeight;
            }
```

## Finding 3: Unthrottled `CreditToken.distribute` calls in `ProfitManager.notifyPnL` reset the rebase vesting period and delay lender yield indefinitely

### Synopsis

In `ProfitManager.notifyPnL`, every positive profit notification with a non-zero lender share directly invokes `CreditToken.distribute(amountForCredit)`. Because `ERC20RebaseDistributor.distribute` resets the interpolation `targetTimestamp` to `block.timestamp + DISTRIBUTION_PERIOD` for all unvested rewards, frequent or tiny loan repayments continuously extend the vesting schedule and delay lender yield indefinitely.

### Technical details

When a borrower repays or partially repays a loan in `LendingTerm._repay` or `LendingTerm._partialRepay` (or when a called loan is settled with profit in `LendingTerm.onBid`), the interest portion is transferred to `ProfitManager` and `ProfitManager.notifyPnL(address(this), int256(interest))` is called.

In `ProfitManager.notifyPnL` (lines 377–380 of `src/governance/ProfitManager.sol`), if `amountForCredit != 0`, the contract immediately calls `CreditToken(_credit).distribute(amountForCredit)`:

```solidity
// distribute to lenders
if (amountForCredit != 0) {
    CreditToken(_credit).distribute(amountForCredit);
}
```

In `ERC20RebaseDistributor.distribute` (lines 362–385 of `src/tokens/ERC20RebaseDistributor.sol`), the rebasing share price and unminted rewards interpolations are updated as follows:

```solidity
uint256 endTimestamp = block.timestamp + DISTRIBUTION_PERIOD;
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
```

Whenever `distribute` is called while a prior distribution is still vesting (`block.timestamp < __rebasingSharePrice.targetTimestamp`), the newly computed `newTargetSharePrice` combines the unvested portion of all previous distributions (`__rebasingSharePrice.targetValue - _rebasingSharePrice`) with the newly distributed `amount`, and stretches the linear interpolation of the entire combined unvested balance over a fresh `DISTRIBUTION_PERIOD` starting from `block.timestamp`.

Because `ProfitManager.notifyPnL` does not batch, buffer, or threshold `amountForCredit` before calling `CreditToken.distribute`, even a `1 wei` lender profit share resets the vesting countdown for all previously unvested rewards back to a full `DISTRIBUTION_PERIOD`. Both normal protocol repayment activity across active loans and intentional dust repayments via `LendingTerm._repay` or `LendingTerm._partialRepay` repeatedly push `targetTimestamp` forward, causing severe vesting slowdown or indefinite griefing of rebasing `CREDIT` rewards.

### Assumptions and preconditions

1. `profitSharingConfig` is configured such that `creditSplit > 0` (the portion of profit allocated to `CREDIT` lenders is non-zero).
2. At least one account is rebasing in `CreditToken` (`_rebasingSupply != 0`).
3. At least one `LendingTerm` has active loans that can be repaid or partially repaid after at least one block has elapsed (`borrowTime < block.timestamp`).

### Proof of concept

1. Suppose `DISTRIBUTION_PERIOD` is 30 days, `_totalRebasingShares` is `100e18` (with share price `1e30`), and `creditSplit` is `0.5e18` (50%).
2. At $t = 0$, a large loan repayment generates `200e18` CREDIT in interest, so `ProfitManager.notifyPnL` calls `CreditToken.distribute(100e18)`.
   - `__rebasingSharePrice.lastValue` is `1e30`, `targetValue` is `2e30`, and `targetTimestamp` is $t = 30\text{ days}$ (vesting at `3.33e18` CREDIT per day).
3. At $t = 15\text{ days}$, halfway through the distribution period:
   - `50e18` CREDIT has vested (`_rebasingSharePrice == 1.5e30`), and `50e18` CREDIT remains unvested, scheduled to finish vesting over the remaining 15 days (by $t = 30\text{ days}$).
4. At $t = 15\text{ days}$, another borrower (or an attacker using a minimal loan) triggers `_repay` or `_partialRepay` after 1 second of interest accrual, resulting in `amountForCredit = 2 wei` being passed to `CreditToken.distribute(2)`.
5. `ERC20RebaseDistributor.distribute(2)` sets `lastTimestamp = 15 days`, `lastValue = 1.5e30`, and `targetTimestamp = 45 days`.
   - The `50e18` CREDIT of unvested rewards from Step 2 now takes another 30 days (until $t = 45\text{ days}$) to vest, immediately cutting the reward emission rate in half.
6. By repeating small repayments once per day, each day only $1/30$ of the remaining unvested rewards vests while the remaining $29/30$ is pushed out by another 30 days, preventing rebasing `CREDIT` holders from ever realizing the full distributed yield on schedule.

### Impact assessment

- **Severity**: Medium
- **Direct Technical Effect**: Every positive `notifyPnL` call that distributes `amountForCredit` resets the vesting duration of all previously unvested rebase rewards in `CreditToken` to a full `DISTRIBUTION_PERIOD`.
- **Business Implications**: Rebasing `CREDIT` holders receive a significantly lower effective APY than intended whenever regular loan repayments occur during an active distribution window, and an attacker can cheaply grief all rebasing `CREDIT` holders by executing periodic small loan repayments to stretch reward vesting indefinitely.

### Remediation

Accumulate lender profits in a pending buffer within `ProfitManager` and only call `CreditToken.distribute` when the previous distribution period has completed (or when a minimum time interval / minimum distribution threshold is met relative to the active distribution).

```solidity
            // distribute to other
            if (amountForOther != 0) {
                CreditToken(_credit).transfer(
                    _profitSharingConfig.otherRecipient,
                    amountForOther
                );
            }

            // distribute to lenders
            if (amountForCredit != 0) {
                uint256 totalForCredit = pendingCreditDistribution + amountForCredit;
                if (block.timestamp >= lastCreditDistribution + CreditToken(_credit).DISTRIBUTION_PERIOD()) {
                    pendingCreditDistribution = 0;
                    lastCreditDistribution = block.timestamp;
                    CreditToken(_credit).distribute(totalForCredit);
                } else {
                    pendingCreditDistribution = totalForCredit;
                }
            }
```

## Finding 4: Missing `userGaugeProfitIndex` Checkpoint Update in `ProfitManager.claimGaugeRewards` Allows Stealing Historical Gauge Rewards and Draining `ProfitManager`

### Synopsis

In `ProfitManager.claimGaugeRewards`, when a user's current gauge weight is zero, the function returns early without checkpointing `userGaugeProfitIndex[user][gauge]` to the current `gaugeProfitIndex[gauge]`. Consequently, users entering or re-entering a gauge after profits have been notified via `notifyPnL` are attributed historical rewards they never earned, allowing them to drain `CREDIT` from `ProfitManager`.

### Technical details

When a gauge generates profit, `ProfitManager.notifyPnL` (lines 388–400) updates `gaugeProfitIndex[gauge]` by adding `(amountForGuild * 1e18) / _gaugeWeight` (initializing `_gaugeProfitIndex` to `1e18` if it was `0`).

To ensure users only earn rewards accrued while they are actively staking `GUILD` in a gauge, `GuildToken._incrementGaugeWeight` (line 258) and `GuildToken._decrementGaugeWeight` (line 220) call `ProfitManager.claimGaugeRewards(user, gauge)` before modifying the user's gauge weight.

However, in `ProfitManager.claimGaugeRewards` (lines 409–436):

```solidity
    function claimGaugeRewards(
        address user,
        address gauge
    ) public returns (uint256 creditEarned) {
        uint256 _userGaugeWeight = uint256(
            GuildToken(guild).getUserGaugeWeight(user, gauge)
        );
        if (_userGaugeWeight == 0) {
            return 0;
        }
        uint256 _gaugeProfitIndex = gaugeProfitIndex[gauge];
        uint256 _userGaugeProfitIndex = userGaugeProfitIndex[user][gauge];
        if (_gaugeProfitIndex == 0) {
            _gaugeProfitIndex = 1e18;
        }
        if (_userGaugeProfitIndex == 0) {
            _userGaugeProfitIndex = 1e18;
        }
        uint256 deltaIndex = _gaugeProfitIndex - _userGaugeProfitIndex;
        if (deltaIndex != 0) {
            creditEarned = (_userGaugeWeight * deltaIndex) / 1e18;
            userGaugeProfitIndex[user][gauge] = _gaugeProfitIndex;
        }
```

When a user with `_userGaugeWeight == 0` calls `GuildToken.incrementGauge(gauge, weight)`:
1. `GuildToken._incrementGaugeWeight` calls `ProfitManager.claimGaugeRewards(user, gauge)` **before** incrementing `getUserGaugeWeight[user][gauge]`.
2. Inside `claimGaugeRewards`, `_userGaugeWeight` is read as `0` (lines 413–415), so execution hits `if (_userGaugeWeight == 0) { return 0; }` at lines 416–418 and returns without setting `userGaugeProfitIndex[user][gauge] = _gaugeProfitIndex`.
3. `super._incrementGaugeWeight(user, gauge, weight)` then sets `getUserGaugeWeight[user][gauge] = weight`.
4. At this point, the user has `_userGaugeWeight = weight`, while `userGaugeProfitIndex[user][gauge]` is still `0` (interpreted as the initial baseline `1e18` at lines 424–426) for a first-time staker, or stuck at a stale index from when the user previously exited the gauge completely.
5. In the same transaction, the user can call `ProfitManager.claimGaugeRewards(user, gauge)` (or `decrementGauge`) and receive `creditEarned = (weight * (_gaugeProfitIndex - _userGaugeProfitIndex)) / 1e18` for all historical profits accumulated prior to the user's entry/re-entry.

Furthermore, if `GUILD` transfers are enabled (or via multiple accounts holding `GUILD`), a user can repeatedly increment weight, claim historical rewards, decrement weight, transfer `GUILD` to a fresh address, and repeat, draining not only unclaimed `GUILD` voter rewards in `ProfitManager`, but also all `CREDIT` held in `surplusBuffer` and `termSurplusBuffer`.

### Assumptions and preconditions

1. At least one gauge has accumulated profits via `ProfitManager.notifyPnL` such that `gaugeProfitIndex[gauge] > 1e18`.
2. A user (having `0` current weight in `gauge` and a `userGaugeProfitIndex[user][gauge]` lower than `gaugeProfitIndex[gauge]`, e.g., `0` / `1e18`) holds `GUILD` tokens and calls `GuildToken.incrementGauge(gauge, weight)`.
3. `ProfitManager` holds `CREDIT` tokens (from unclaimed gauge rewards, `surplusBuffer`, or `termSurplusBuffer`).

### Proof of concept

Consider the following scenario:
1. Alice holds `100e18 GUILD` and votes `100e18` weight for `gauge1` (`_gaugeWeight = 100e18`). At this time, `gaugeProfitIndex[gauge1] == 0`.
2. A loan repayment on `gauge1` notifies profit via `ProfitManager.notifyPnL(gauge1, amount)` where `amountForGuild = 100e18 CREDIT`.
   - `ProfitManager` receives `100e18 CREDIT`.
   - In `notifyPnL` (lines 392–399), `gaugeProfitIndex[gauge1]` is updated from `1e18` to `1e18 + (100e18 * 1e18) / 100e18 = 2e18`.
3. Bob holds `100e18 GUILD` and has never voted for `gauge1` (`getUserGaugeWeight[bob][gauge1] == 0`, `userGaugeProfitIndex[bob][gauge1] == 0`).
4. Bob calls `guild.incrementGauge(gauge1, 100e18)`:
   - `GuildToken._incrementGaugeWeight` calls `profitManager.claimGaugeRewards(bob, gauge1)`.
   - Because `getUserGaugeWeight[bob][gauge1]` is still `0`, `claimGaugeRewards` returns `0` at line 417 without updating `userGaugeProfitIndex[bob][gauge1]`.
   - `getUserGaugeWeight[bob][gauge1]` is then incremented to `100e18`.
5. Bob immediately calls `profitManager.claimGaugeRewards(bob, gauge1)`:
   - `_userGaugeWeight = 100e18`.
   - `_gaugeProfitIndex = 2e18`.
   - `_userGaugeProfitIndex` is `0`, which is set to `1e18` at line 425.
   - `deltaIndex = 2e18 - 1e18 = 1e18`.
   - `creditEarned = (100e18 * 1e18) / 1e18 = 100e18`.
   - `ProfitManager` transfers `100e18 CREDIT` to Bob.
6. Bob steals the `100e18 CREDIT` that belonged to Alice (or drains `ProfitManager`'s `surplusBuffer` if Alice already claimed). When Alice later tries to claim her rewards (or unstake her `GUILD`, which triggers `claimGaugeRewards`), her transaction either drains the `surplusBuffer` or reverts due to insufficient `CREDIT` balance in `ProfitManager`.

### Impact assessment

- **Severity**: High
- **Direct Technical Impact**: Any `GUILD` holder entering or re-entering a gauge with `gaugeProfitIndex[gauge] > 1e18` is credited with historical rewards accrued while they had zero weight in the gauge.
- **Financial & Protocol Impact**:
  1. Direct theft of unclaimed `CREDIT` rewards belonging to honest `GUILD` stakers, and direct drainage of `surplusBuffer` and `termSurplusBuffer` held on `ProfitManager`.
  2. Once `ProfitManager`'s `CREDIT` balance is depleted below the remaining unclaimed rewards, honest `GUILD` stakers cannot decrement their gauge weight or transfer/burn their `GUILD` tokens because `_decrementGaugeWeight` calls `claimGaugeRewards`, which reverts when `CreditToken.transfer` fails due to insufficient balance.

### Remediation

In `ProfitManager.claimGaugeRewards`, always update `userGaugeProfitIndex[user][gauge]` to `_gaugeProfitIndex` (initializing `_gaugeProfitIndex` to `1e18` if it is `0`) even when `_userGaugeWeight == 0`:

```solidity
    function claimGaugeRewards(
        address user,
        address gauge
    ) public returns (uint256 creditEarned) {
        uint256 _userGaugeWeight = uint256(
            GuildToken(guild).getUserGaugeWeight(user, gauge)
        );
        uint256 _gaugeProfitIndex = gaugeProfitIndex[gauge];
        if (_gaugeProfitIndex == 0) {
            _gaugeProfitIndex = 1e18;
        }
        if (_userGaugeWeight == 0) {
            userGaugeProfitIndex[user][gauge] = _gaugeProfitIndex;
            return 0;
        }
        uint256 _userGaugeProfitIndex = userGaugeProfitIndex[user][gauge];
        if (_userGaugeProfitIndex == 0) {
            _userGaugeProfitIndex = 1e18;
        }
        uint256 deltaIndex = _gaugeProfitIndex - _userGaugeProfitIndex;
        if (deltaIndex != 0) {
            creditEarned = (_userGaugeWeight * deltaIndex) / 1e18;
            userGaugeProfitIndex[user][gauge] = _gaugeProfitIndex;
        }
        if (creditEarned != 0) {
            emit ClaimRewards(block.timestamp, user, gauge, creditEarned);
            CreditToken(credit).transfer(user, creditEarned);
        }
    }
```

## Finding 5: Unconditional Profit Index Advancement and Truncation in `ProfitManager.claimGaugeRewards` Causes Reward Loss and `SurplusGuildMinter` Accounting Desynchronization

### Synopsis

In `ProfitManager.claimGaugeRewards`, `userGaugeProfitIndex[user][gauge]` is unconditionally updated to `_gaugeProfitIndex` whenever `deltaIndex != 0`, discarding any truncated remainder from `(_userGaugeWeight * deltaIndex) / 1e18` (including when `creditEarned == 0`). Permissionless calls can repeatedly forfeit rewards and desynchronize `SurplusGuildMinter` reward accounting.

### Technical details

In `ProfitManager.sol`, gauge profits allocated to `GUILD` stakers increase `gaugeProfitIndex[gauge]` by `(amountForGuild * 1e18) / _gaugeWeight`. When `claimGaugeRewards(address user, address gauge)` is called for a user with non-zero weight (`_userGaugeWeight > 0`), the reward is computed and the user's index is updated as follows:

```solidity
uint256 deltaIndex = _gaugeProfitIndex - _userGaugeProfitIndex;
if (deltaIndex != 0) {
    creditEarned = (_userGaugeWeight * deltaIndex) / 1e18;
    userGaugeProfitIndex[user][gauge] = _gaugeProfitIndex;
}
if (creditEarned != 0) {
    emit ClaimRewards(block.timestamp, user, gauge, creditEarned);
    CreditToken(credit).transfer(user, creditEarned);
}
```

This logic exhibits two tightly coupled precision and conservation flaws at lines 428–434:

1. **Index advancement on zero or truncated payout**: Whenever `deltaIndex != 0`, `userGaugeProfitIndex[user][gauge]` is set to the full `_gaugeProfitIndex` at line 430 regardless of whether `creditEarned` is `0` or how much precision was lost in the integer division `(_userGaugeWeight * deltaIndex) / 1e18` at line 429. If `_userGaugeWeight * deltaIndex < 1e18`, `creditEarned` evaluates to `0`, yet `userGaugeProfitIndex[user][gauge]` is still overwritten with `_gaugeProfitIndex`, permanently erasing the user's accrued `deltaIndex` without transferring any `CREDIT`. Even when `creditEarned > 0`, the remainder `(_userGaugeWeight * deltaIndex) % 1e18` is forfeited rather than retained in the user's unclaimed index position.
2. **Permissionless settlement and nested distributor divergence**: Because `claimGaugeRewards(user, gauge)` is `public` and unrestricted, any caller can invoke it for any `user` and `gauge` after each profit notification. Furthermore, `SurplusGuildMinter.getRewards` calls `ProfitManager(profitManager).claimRewards(address(this))` and then reads `ProfitManager(profitManager).userGaugeProfitIndex(address(this), term)` as `_profitIndex` to compute each staker's `creditReward` and `guildReward`. If `claimGaugeRewards(address(surplusGuildMinter), term)` is triggered across multiple small `deltaIndex` increments where `(_sgmWeight * stepDeltaIndex) / 1e18` truncates (or rounds to `0`), `ProfitManager` transfers less `CREDIT` (or `0` `CREDIT`) to `SurplusGuildMinter`, while `userGaugeProfitIndex[surplusGuildMinter][term]` advances by the full untruncated sum of `stepDeltaIndex`. When a staker later claims over the cumulative index difference in `SurplusGuildMinter.getRewards`, `SurplusGuildMinter` calculates a larger `creditReward` than the `CREDIT` it actually received from `ProfitManager`, causing `SurplusGuildMinter` to become insolvent (reverting `CreditToken(credit).transfer(user, creditReward)` during `getRewards` / `unstake`) and minting `guildReward` for `CREDIT` rewards that were never collected.

### Assumptions and preconditions

1. A user (or `SurplusGuildMinter`) has a non-zero gauge weight (`_userGaugeWeight > 0`) in an active gauge.
2. One or more profit notifications (`notifyPnL` with `amount > 0`) occur on the gauge such that `_userGaugeWeight * deltaIndex` is not an exact multiple of `1e18` (or `_userGaugeWeight * deltaIndex < 1e18`).
3. `claimGaugeRewards(user, gauge)` is called (either directly by any address, via `claimRewards`, or via `SurplusGuildMinter.getRewards`).

### Proof of concept

1. Suppose `gauge1` has total weight `1000e18`. Alice (or `SurplusGuildMinter`) has `_userGaugeWeight = 1e17` (0.1 GUILD) on `gauge1`, while Bob holds the remaining `999.9e18` weight. Initially, `gaugeProfitIndex[gauge1] = 1e18` and `userGaugeProfitIndex[alice][gauge1] = 1e18`.
2. A loan repayment notifies `amountForGuild = 5000e18` CREDIT profit on `gauge1`. `gaugeProfitIndex[gauge1]` increases by `(5000e18 * 1e18) / 1000e18 = 5e18` to `6e18`.
3. `claimGaugeRewards(alice, gauge1)` is called:
   - `deltaIndex = 6e18 - 1e18 = 5e18`.
   - `creditEarned = (1e17 * 5e18) / 1e18 = 0.5e18`.
   - However, if that same `5000e18` profit was notified across 10 smaller repayments of `500e18` each (so each step increases `gaugeProfitIndex[gauge1]` by `0.5e18`), and `claimGaugeRewards(alice, gauge1)` is called after each repayment:
   - On each call, `deltaIndex = 0.5e18`, and `creditEarned = (1e17 * 0.5e18) / 1e18 = 0.05e18`.
   - If `_userGaugeWeight` is `1e9` (or `deltaIndex` per step is `9` wei of index for `_userGaugeWeight = 1e17`):
     - `deltaIndex = 9`, `creditEarned = (1e17 * 9) / 1e18 = 0`.
     - Line 430 sets `userGaugeProfitIndex[alice][gauge1] = _gaugeProfitIndex`.
     - Line 432 checks `if (creditEarned != 0)` which is false, so `0` `CREDIT` is transferred.
4. Repeating step 3 across multiple profit notifications results in `userGaugeProfitIndex[alice][gauge1]` advancing by the full cumulative `deltaIndex` while Alice (or `SurplusGuildMinter`) receives `0` `CREDIT`.
5. If the account is `SurplusGuildMinter`, `userGaugeProfitIndex[surplusGuildMinter][gauge1]` has still advanced by the sum of all `deltaIndex` steps. When the staker later calls `SurplusGuildMinter.getRewards(staker, gauge1)` after enough steps have accumulated such that `(userStake.guild * totalDeltaIndex) / 1e18 >= 1`, `SurplusGuildMinter` attempts to transfer `creditReward >= 1` to `staker` at line 263 of `SurplusGuildMinter.sol` despite having received `0` `CREDIT` from `ProfitManager`, causing the transaction to revert due to insufficient `CREDIT` balance.

### Impact assessment

- **Severity**: Medium
- **Direct Technical Effects**:
  1. Users with smaller gauge weights or frequent claims lose accrued `CREDIT` rewards due to repeated truncation and unconditional index updates when `creditEarned == 0`.
  2. `ProfitManager` retains unclaimed `CREDIT` tokens that can never be distributed because the corresponding `userGaugeProfitIndex` has already been advanced.
  3. `SurplusGuildMinter`'s internal reward accounting becomes desynchronized from its actual `CREDIT` balance, allowing unbacked `GUILD` rewards to be minted and causing `SurplusGuildMinter.getRewards` (and dependent functions like `stake` and `unstake`) to revert when `SurplusGuildMinter` lacks the `CREDIT` tokens required to pay the computed `creditReward`.

### Remediation

Only update `userGaugeProfitIndex[user][gauge]` when `creditEarned != 0`, and advance `userGaugeProfitIndex[user][gauge]` only by the index delta corresponding to the `creditEarned` actually paid out (or retain the unaccounted remainder):

```solidity
    function claimGaugeRewards(
        address user,
        address gauge
    ) public returns (uint256 creditEarned) {
        uint256 _userGaugeWeight = uint256(
            GuildToken(guild).getUserGaugeWeight(user, gauge)
        );
        if (_userGaugeWeight == 0) {
            return 0;
        }
        uint256 _gaugeProfitIndex = gaugeProfitIndex[gauge];
        uint256 _userGaugeProfitIndex = userGaugeProfitIndex[user][gauge];
        if (_gaugeProfitIndex == 0) {
            _gaugeProfitIndex = 1e18;
        }
        if (_userGaugeProfitIndex == 0) {
            _userGaugeProfitIndex = 1e18;
        }
        uint256 deltaIndex = _gaugeProfitIndex - _userGaugeProfitIndex;
        if (deltaIndex != 0) {
            creditEarned = (_userGaugeWeight * deltaIndex) / 1e18;
        }
        if (creditEarned != 0) {
            uint256 indexConsumed = (creditEarned * 1e18) / _userGaugeWeight;
            userGaugeProfitIndex[user][gauge] =
                _userGaugeProfitIndex +
                indexConsumed;
            emit ClaimRewards(block.timestamp, user, gauge, creditEarned);
            CreditToken(credit).transfer(user, creditEarned);
        }
    }
```

## Finding 6: Same-Block Loss Checkpoint Initialization in `GuildToken._incrementGaugeWeight` Allows Stakers to Bypass Slashing

### Synopsis

In `GuildToken._incrementGaugeWeight`, initializing `lastGaugeLossApplied[gauge][user]` to `block.timestamp` when `getUserGaugeWeight[user][gauge] == 0` allows users who allocate gauge weight in the same block as a gauge loss to bypass `applyGaugeLoss` and `_decrementGaugeWeight` loss checks, enabling stakers to escape slashing.

### Technical details

In `GuildToken`, gauge stakers are meant to be slashed via `applyGaugeLoss` whenever a `LendingTerm` gauge incurs bad debt (`ProfitManager.notifyPnL` calling `GuildToken.notifyGaugeLoss`). Loss tracking relies on two timestamps:
- `lastGaugeLoss[gauge]`: set to `block.timestamp` in `notifyGaugeLoss(gauge)` when a loss is reported.
- `lastGaugeLossApplied[gauge][user]`: tracks the timestamp at which the user last applied or initialized their loss checkpoint for `gauge`.

Slashing eligibility and withdrawal restrictions are enforced using strict/non-strict timestamp comparisons:
1. `applyGaugeLoss(gauge, who)` requires `_lastGaugeLoss != 0 && _lastGaugeLossApplied < _lastGaugeLoss` (line 138).
2. `_decrementGaugeWeight(user, gauge, weight)` requires `_lastGaugeLossApplied >= _lastGaugeLoss` (line 215).
3. `_incrementGaugeWeight(user, gauge, weight)` checks (lines 247–256):
```solidity
uint256 _lastGaugeLoss = lastGaugeLoss[gauge];
uint256 _lastGaugeLossApplied = lastGaugeLossApplied[gauge][user];
if (getUserGaugeWeight[user][gauge] == 0) {
    lastGaugeLossApplied[gauge][user] = block.timestamp;
} else {
    require(
        _lastGaugeLossApplied >= _lastGaugeLoss,
        "GuildToken: pending loss"
    );
}
```

When a user with `getUserGaugeWeight[user][gauge] == 0` increments weight in `gauge`, `lastGaugeLossApplied[gauge][user]` is set to `block.timestamp` (line 250). If a loss occurs on `gauge` later in the same block (or within the same transaction), `notifyGaugeLoss(gauge)` sets `lastGaugeLoss[gauge] = block.timestamp`.

At this point, `lastGaugeLossApplied[gauge][user] == lastGaugeLoss[gauge] == block.timestamp`, even though the user had active weight allocated to `gauge` when the loss was realized:
- `applyGaugeLoss(gauge, user)` reverts with `"GuildToken: no loss to apply"` because `_lastGaugeLossApplied < _lastGaugeLoss` evaluates to `false`.
- `_decrementGaugeWeight(user, gauge, weight)` succeeds because `_lastGaugeLossApplied >= _lastGaugeLoss` evaluates to `true`.

Furthermore, an existing staker whose `GUILD` weight is locked in `gauge` due to debt ceiling utilization (`issuance > debtCeilingAfterDecrement` in `_decrementGaugeWeight`) can use a secondary address with `0` weight in `gauge` within the same transaction/block as a loss-inducing auction bid (`LendingTerm.onBid`):
1. The secondary address calls `incrementGauge(gauge, weight)`, setting `lastGaugeLossApplied[gauge][secondary] = block.timestamp` and increasing the gauge's debt ceiling.
2. The primary address calls `decrementGauge(gauge, weight)`, which now passes the debt ceiling check and exits `gauge` before the loss is reported.
3. The auction bid executes in the same transaction/block, closing the underwater loan (reducing `issuance`) and calling `notifyPnL` -> `notifyGaugeLoss(gauge)`, which sets `lastGaugeLoss[gauge] = block.timestamp`.
4. The secondary address immediately calls `decrementGauge(gauge, weight)`. Because `lastGaugeLossApplied[gauge][secondary] == lastGaugeLoss[gauge] == block.timestamp` and `issuance` was reduced by the loan closure, the secondary address also exits without being slashed.

### Assumptions and preconditions

1. A `LendingTerm` gauge has an active loan that will incur a loss (e.g., via a liquidation auction bid in `AuctionHouse` calling `LendingTerm.onBid` with `creditFromBidder < principal`).
2. An account with `getUserGaugeWeight[user][gauge] == 0` holds unallocated `GUILD` tokens (or `transferable` is enabled / `GUILD` is minted via `SurplusGuildMinter` or held in a secondary account) to allocate weight to `gauge` in the same block or transaction before the loss is notified.

### Proof of concept

1. Assume `bob` has `100e18` `GUILD` staked in `gauge1` from an earlier block (`t = 100`), and `gauge1` has active loan issuance utilizing its debt ceiling so `bob` cannot call `decrementGauge(gauge1, 100e18)` (`"GuildToken: debt ceiling used"`).
2. A loan in `gauge1` is called and reaches a liquidation auction price where `creditFromBidder < principal` (so bidding on it will trigger a loss in `gauge1`).
3. At `t = 200`, in a single transaction before bidding on the auction:
   - An accomplice account `alice` (with `getUserGaugeWeight[alice][gauge1] == 0` and `100e18` unallocated `GUILD`) calls `GuildToken.incrementGauge(gauge1, 100e18)`.
   - Inside `_incrementGaugeWeight`, `getUserGaugeWeight[alice][gauge1] == 0` is true, so `lastGaugeLossApplied[gauge1][alice]` is set to `200` (`block.timestamp`).
   - `bob` calls `GuildToken.decrementGauge(gauge1, 100e18)`. Because `alice`'s weight increased `gauge1`'s debt ceiling, `issuance <= debtCeilingAfterDecrement` holds, and `bob`'s weight in `gauge1` is reduced to `0`.
   - The attacker bids on the auction (`AuctionHouse.bid` -> `LendingTerm.onBid`), closing the defaulted loan (reducing `issuance`) and calling `ProfitManager.notifyPnL(gauge1, pnl)` with `pnl < 0`.
   - `ProfitManager.notifyPnL` calls `GuildToken.notifyGaugeLoss(gauge1)`, setting `lastGaugeLoss[gauge1] = 200` (`block.timestamp`).
   - `alice` calls `GuildToken.decrementGauge(gauge1, 100e18)`. Inside `_decrementGaugeWeight`, `_lastGaugeLossApplied >= _lastGaugeLoss` (`200 >= 200`) passes, and since the loan was closed, the debt ceiling check also passes.
4. Both `bob` and `alice` retain their full `100e18` `GUILD` balances, completely evading the gauge loss slashing.

### Impact assessment

- **Severity**: High
- **Technical Effect**: Stakers who allocate weight to a gauge in the same block as a loss notification cannot be slashed via `applyGaugeLoss` and can immediately withdraw their gauge weight via `decrementGauge`.
- **Business Impact**: Violates the core protocol invariant that `GUILD` stakers backing a `LendingTerm` cannot exit when bad debt is realized and must be slashed for losses in gauges they vote for. Existing stakers locked by debt ceiling usage can atomically swap their exposure to a fresh account in the same block as a liquidation and completely avoid slashing.

### Remediation

In `GuildToken._incrementGaugeWeight`, when `getUserGaugeWeight[user][gauge] == 0`, set `lastGaugeLossApplied[gauge][user]` to `_lastGaugeLoss` (the timestamp of the last realized loss on the gauge) and require that a loss has not already occurred in the current block (or set it to `_lastGaugeLoss` so any loss notified in the current block has `_lastGaugeLoss == block.timestamp > _lastGaugeLossApplied`), ensuring any subsequent loss in the same block will still slash the user:

```solidity
    function _incrementGaugeWeight(
        address user,
        address gauge,
        uint256 weight
    ) internal override {
        uint256 _lastGaugeLoss = lastGaugeLoss[gauge];
        uint256 _lastGaugeLossApplied = lastGaugeLossApplied[gauge][user];
        if (getUserGaugeWeight[user][gauge] == 0) {
            require(
                _lastGaugeLoss < block.timestamp,
                "GuildToken: loss in block"
            );
            lastGaugeLossApplied[gauge][user] = _lastGaugeLoss;
        } else {
            require(
                _lastGaugeLossApplied >= _lastGaugeLoss,
                "GuildToken: pending loss"
            );
        }

        ProfitManager(profitManager).claimGaugeRewards(user, gauge);

        super._incrementGaugeWeight(user, gauge, weight);
    }
```
