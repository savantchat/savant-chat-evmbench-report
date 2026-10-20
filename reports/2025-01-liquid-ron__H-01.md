# Audit report — 2025-01-liquid-ron/H-01

## Finding 1: `LiquidProxy.harvestAndDelegateRewards` Always Reverts Because `RoninValidator.delegateRewards` Does Not Return a `uint256`

### Synopsis

`LiquidProxy.harvestAndDelegateRewards` expects `IRoninValidator(roninStaking).delegateRewards` to return a `uint256` representing the claimed and delegated reward amount. However, on the actual Ronin Staking (`RewardPool`) contract, the external `delegateRewards` function has no return value, causing ABI decoding in `LiquidProxy.harvestAndDelegateRewards` to always revert.

### Technical details

In `src/LiquidProxy.sol`, `harvestAndDelegateRewards` calls `delegateRewards` on `roninStaking` via the `IRoninValidator` interface and assigns its return value to `claimableAmount`:

```solidity
function harvestAndDelegateRewards(
    address[] calldata _consensusAddrs,
    address _consensusAddrDst
) external onlyVault returns (uint256) {
    uint256 claimableAmount = IRoninValidator(roninStaking).delegateRewards(_consensusAddrs, _consensusAddrDst);
    return claimableAmount;
}
```

In `src/interfaces/IRoninValidators.sol` (line 16), `delegateRewards` is declared as:

```solidity
function delegateRewards(address[] calldata, address consensusAddrDst) external returns (uint256 amount);
```

However, in the actual Ronin Staking contract (`RewardPool.sol` in `ronin-dpos-contracts`), while the `internal` helper `_delegateRewards` returns `uint256 amount`, the `external` function `delegateRewards` does **not** return a value:

```solidity
function delegateRewards(
    address[] calldata _consensusAddrs,
    address _consensusAddrDst
) external override nonReentrant {
    _delegateRewards(_consensusAddrs, _consensusAddrDst, _msgSender(), _getPeriod());
}
```

Note that `claimRewards` in the same Ronin `RewardPool` contract similarly has no return value, which is why `LiquidProxy.harvest` calls `claimRewards` without expecting a return value and instead measures `address(this).balance`.

In Solidity `0.8.20`, when an external call is made through an interface that declares a `uint256` return value, the compiler emits a post-call check verifying that `returndatasize() >= 32` before ABI-decoding the return data. Because the real Ronin Staking contract's `delegateRewards` returns 0 bytes of return data (`returndatasize() == 0`), the call at line 54 of `src/LiquidProxy.sol` will unconditionally revert during return-data decoding. This issue was masked in the test suite because `MockRonStaking.delegateRewards` was written to return `uint256 rewards` to match the erroneous `IRoninValidator` interface.

### Assumptions and preconditions

- `LiquidProxy` is deployed pointing to the official Ronin Staking contract (`roninStaking`) where `delegateRewards(address[] calldata, address)` does not return a `uint256`.
- An operator or owner calls `LiquidRon.harvestAndDelegateRewards`, which invokes `LiquidProxy.harvestAndDelegateRewards`.

### Proof of concept

1. Users deposit RON into `LiquidRon`, and an operator delegates stakes from a `LiquidProxy` instance (e.g., `stakingProxies[0]`) to validators `consensusAddrs`.
2. Rewards accrue over time on the Ronin Staking contract for `stakingProxies[0]`.
3. The operator (or owner) calls `LiquidRon.harvestAndDelegateRewards(0, consensusAddrs, consensusAddrDst)` to compound the rewards into `consensusAddrDst`.
4. `LiquidRon` calls `LiquidProxy(stakingProxies[0]).harvestAndDelegateRewards(consensusAddrs, consensusAddrDst)`.
5. `LiquidProxy` executes `IRoninValidator(roninStaking).delegateRewards(consensusAddrs, consensusAddrDst)`.
6. The Ronin Staking contract executes `delegateRewards` and succeeds with 0 bytes of return data (`returndatasize() == 0`).
7. `LiquidProxy` attempts to ABI-decode a 32-byte `uint256` from the empty return data and immediately reverts the entire transaction.

### Impact assessment

- **Severity**: Medium / High (Broken Core Functionality / Permanent Denial of Service of reward compounding).
- **Technical Impact**: `LiquidProxy.harvestAndDelegateRewards` and `LiquidRon.harvestAndDelegateRewards` will always revert on mainnet/testnet when called against the real Ronin Staking contract.
- **Business Impact**: Operators cannot use `harvestAndDelegateRewards` to compound staking rewards directly into a destination validator. Instead, rewards can only be claimed via `harvest` (which withdraws RON to the vault as WRON) and re-delegated separately.

### Remediation

Update `IRoninValidator.delegateRewards` so that it does not expect a return value, and compute `claimableAmount` in `LiquidProxy.harvestAndDelegateRewards` by querying the proxy's claimable rewards via `IRoninValidator(roninStaking).getRewards(address(this), _consensusAddrs)` (or by measuring the change in `getStakingAmount(_consensusAddrDst, address(this))`) before calling `delegateRewards`:

```solidity
    function harvestAndDelegateRewards(
        address[] calldata _consensusAddrs,
        address _consensusAddrDst
    ) external onlyVault returns (uint256) {
        uint256[] memory rewards = IRoninValidator(roninStaking).getRewards(address(this), _consensusAddrs);
        uint256 claimableAmount;
        for (uint256 i = 0; i < rewards.length; i++) {
            claimableAmount += rewards[i];
        }
        IRoninValidator(roninStaking).delegateRewards(_consensusAddrs, _consensusAddrDst);
        return claimableAmount;
    }
```

## Finding 2: Unbacked Operator Fee Accrual in `LiquidProxy.harvestAndDelegateRewards` and `LiquidRon.harvestAndDelegateRewards` Causes Vault Liquidity Mismatch and Share Price Distortion

### Synopsis

`LiquidProxy.harvestAndDelegateRewards` delegates 100% of claimed staking rewards directly back into the Ronin staking contract without transferring the operator fee portion to the `LiquidRon` vault. However, `LiquidRon.harvestAndDelegateRewards` still increments `operatorFeeAmount`, creating an unbacked fee liability that drains user-deposited vault liquidity or causes `fetchOperatorFee` to revert.

### Technical details

In `LiquidProxy.sol`, `harvestAndDelegateRewards` calls `IRoninValidator(roninStaking).delegateRewards(_consensusAddrs, _consensusAddrDst)` and returns the total `claimableAmount`:

```solidity
function harvestAndDelegateRewards(
    address[] calldata _consensusAddrs,
    address _consensusAddrDst
) external onlyVault returns (uint256) {
    uint256 claimableAmount = IRoninValidator(roninStaking).delegateRewards(_consensusAddrs, _consensusAddrDst);
    return claimableAmount;
}
```

When `IRoninValidator.delegateRewards` executes on the Ronin staking contract, the entirety of `claimableAmount` (100% of the harvested rewards) is immediately re-staked into `_consensusAddrDst` under the proxy's address (and becomes subject to the 72-hour staking lockup). No RON or WRON is transferred to `LiquidProxy` or forwarded to the `LiquidRon` vault.

However, the caller `LiquidRon.harvestAndDelegateRewards` treats the returned `harvestedAmount` identically to `LiquidRon.harvest` and accrues the operator fee in `operatorFeeAmount`:

```solidity
uint256 harvestedAmount = ILiquidProxy(stakingProxies[_proxyIndex]).harvestAndDelegateRewards(
    _consensusAddrs,
    _consensusAddrDst
);
operatorFeeAmount += (harvestedAmount * operatorFee) / BIPS;
```

When `feeRecipient` subsequently calls `LiquidRon.fetchOperatorFee()`, the contract attempts to unwrap and transfer `operatorFeeAmount` from the vault's own liquid WRON balance (`IWRON(wron).withdraw(amount)` in `_withdrawRONTo`).

This creates two coupled accounting and liquidity failures:
1. **Unbacked Fee Liability / Insufficient Vault Liquidity:** Because 100% of the rewards were re-staked in `roninStaking` rather than sent to the vault as WRON, the vault receives zero WRON from `harvestAndDelegateRewards`. If the vault has no idle WRON (or less idle WRON than `operatorFeeAmount`), calling `fetchOperatorFee()` reverts when trying to withdraw WRON. If the vault holds idle WRON from user deposits or withdrawal fulfilment buffers, `fetchOperatorFee()` consumes user-deposited WRON to pay the operator fee.
2. **Duplicate Reward Recognition in `totalAssets()` Before Fee Collection:** The full `harvestedAmount` (including the operator fee portion) is added to the proxy's staked balance in `roninStaking`, inflating the staked assets reported to the vault while `operatorFeeAmount` remains an unpaid liability until `fetchOperatorFee()` is called. When `fetchOperatorFee()` is eventually executed using liquid WRON in the vault, the vault's WRON balance drops abruptly by `operatorFeeAmount`, reducing `totalAssets()` at the moment of fee withdrawal and penalizing users who withdraw after `fetchOperatorFee()`.

### Assumptions and preconditions

1. `operatorFee` is non-zero (which is the default and intended protocol configuration).
2. At least one staking proxy has accrued staking rewards on the Ronin staking contract.
3. An operator (or owner) calls `LiquidRon.harvestAndDelegateRewards` to compound rewards into a destination validator.

### Proof of concept

1. A user deposits `1000 RON` into `LiquidRon`, which wraps it into `1000 WRON`.
2. The operator delegates the full `1000 RON` across validators via `LiquidRon.delegateAmount`. The vault's liquid WRON balance (`IWRON(wron).balanceOf(address(liquidRon))`) is now `0`, and total staked RON is `1000 RON`.
3. Over time, `100 RON` of staking rewards accrue. Assume `operatorFee` is `1000` (10%).
4. The operator calls `LiquidRon.harvestAndDelegateRewards(0, consensusAddrs, consensusAddrDst)`.
5. `LiquidProxy.harvestAndDelegateRewards` calls `delegateRewards`, which compounds all `100 RON` into `consensusAddrDst`. Total staked RON is now `1100 RON`, while `IWRON(wron).balanceOf(address(liquidRon))` remains `0`.
6. `LiquidRon.harvestAndDelegateRewards` increases `operatorFeeAmount` by `10 RON`.
7. When `feeRecipient` calls `LiquidRon.fetchOperatorFee()`, `_withdrawRONTo(feeRecipient, 10 ether)` calls `IWRON(wron).withdraw(10 ether)`, which reverts because `LiquidRon` holds `0 WRON`.
8. If another user subsequently deposits `10 RON` into the vault, `fetchOperatorFee()` will succeed by taking that user's newly deposited `10 WRON`, leaving the vault with `0` liquid WRON and decreasing `totalAssets()` upon fee withdrawal.

### Impact assessment

- **Severity:** High / Medium
- **Technical Impact:** `operatorFeeAmount` is accrued without corresponding WRON liquidity in the vault, causing `fetchOperatorFee()` to either revert (freezing fee collection when vault funds are fully staked) or consume liquid WRON belonging to user deposits/withdrawal reserves, while distorting `totalAssets()` and share prices across fee collection boundaries.

### Remediation

Ensure that `totalAssets()` accounts for `operatorFeeAmount` as a liability (deducting `operatorFeeAmount` from `totalAssets()`), or claim the rewards first in `LiquidProxy.harvestAndDelegateRewards`, send the `operatorFee` portion in WRON to the `vault`, and only delegate the net reward amount (`claimableAmount - feeAmount`) to `_consensusAddrDst`.

```solidity
    function harvestAndDelegateRewards(
        address[] calldata _consensusAddrs,
        address _consensusAddrDst,
        uint256 _operatorFee,
        uint256 _bips
    ) external onlyVault returns (uint256) {
        uint256 claimedAmount = IRoninValidator(roninStaking).claimRewards(_consensusAddrs);
        uint256 feeAmount = (claimedAmount * _operatorFee) / _bips;
        uint256 delegateAmount = claimedAmount - feeAmount;
        if (feeAmount > 0) {
            _depositRONTo(vault, feeAmount);
        }
        if (delegateAmount > 0) {
            IRoninValidator(roninStaking).delegate{value: delegateAmount}(_consensusAddrDst);
        }
        return claimedAmount;
    }
```

## Finding 3: Flawed Boolean Logic in `LiquidRon.onlyOperator` Modifier Prevents Authorized Operators from Calling Staking and Epoch Finalization Functions

### Synopsis

The `onlyOperator` modifier in `LiquidRon` uses an incorrect boolean condition (`msg.sender != owner() || operator[msg.sender]`), causing calls from any authorized non-owner operator—or an owner who is also marked as an operator—to unconditionally revert with `ErrInvalidOperator()`.

### Technical details

In `src/LiquidRon.sol`, the `onlyOperator` modifier (lines 90–93) is intended to allow either the contract `owner()` or any address registered in the `operator` mapping (`operator[msg.sender] == true`) to execute privileged staking and withdrawal management functions.

However, line 91 implements the check as:
```solidity
if (msg.sender != owner() || operator[msg.sender]) revert ErrInvalidOperator();
```

This logic is flawed in two ways:
1. It uses the logical OR (`||`) operator instead of logical AND (`&&`). For any caller where `msg.sender != owner()`, the first clause evaluates to `true`, immediately short-circuiting and reverting with `ErrInvalidOperator()` regardless of whether `operator[msg.sender]` is `true`.
2. The second clause checks `operator[msg.sender]` instead of `!operator[msg.sender]`. Consequently, if the `owner()` address is also granted operator status via `updateOperator(owner(), true)`, the second clause evaluates to `true` and causes calls from the owner to revert as well.

As a result, every function protected by `onlyOperator`—specifically `harvest`, `harvestAndDelegateRewards`, `delegateAmount`, `redelegateAmount`, `undelegateAmount`, and `finaliseRonRewardsForEpoch`—is completely inaccessible to designated operators.

### Assumptions and preconditions

- The contract owner designates a separate address `op` (`op != owner()`) as an operator by calling `updateOperator(op, true)`, or sets `updateOperator(owner(), true)`.
- The designated operator attempts to call any function restricted by the `onlyOperator` modifier.

### Proof of concept

1. The contract `owner` calls `LiquidRon.updateOperator(operatorAddr, true)`, setting `operator[operatorAddr] = true` (where `operatorAddr != owner()`).
2. `operatorAddr` calls `LiquidRon.harvest(0, consensusAddrs)` (or any other `onlyOperator` function such as `delegateAmount` or `finaliseRonRewardsForEpoch`).
3. Execution enters the `onlyOperator` modifier at line 91: `if (msg.sender != owner() || operator[msg.sender]) revert ErrInvalidOperator();`.
4. Because `operatorAddr != owner()` is `true` (and `operator[operatorAddr]` is also `true`), the condition evaluates to `true`.
5. The transaction reverts with `ErrInvalidOperator()`, preventing the operator from managing validator stakes, harvesting rewards, or finalizing withdrawal epochs.

### Impact assessment

**Severity: Medium / High**

Designated operators are completely locked out of all operator-restricted functionality (`harvest`, `harvestAndDelegateRewards`, `delegateAmount`, `redelegateAmount`, `undelegateAmount`, and `finaliseRonRewardsForEpoch`). Automated off-chain operator bots or delegated operator accounts cannot manage validator delegations, compound yield, or finalize user withdrawal epochs unless they are given full `owner` privileges over the `LiquidRon` contract (or if the `owner` itself is inadvertently added as an operator, locking out the owner until `updateOperator(owner(), false)` is called).

### Remediation

Update the `onlyOperator` modifier in `src/LiquidRon.sol` to revert only when the caller is neither the owner nor an authorized operator:

```solidity
    /// @dev Modifier to restrict access of a function to an operator or owner
    modifier onlyOperator() {
        if (msg.sender != owner() && !operator[msg.sender]) revert ErrInvalidOperator();
        _;
    }
```

## Finding 4: Unaccounted `operatorFeeAmount` in `LiquidRon.harvest` and `LiquidRon.fetchOperatorFee` Inflates `totalAssets` and Distorts Share Price

### Synopsis

In `LiquidRon`, operator fees accrued during `harvest()` and `harvestAndDelegateRewards()` are accumulated in `operatorFeeAmount` while the underlying WRON remains in the vault's asset balance. Because `totalAssets()` does not deduct `operatorFeeAmount`, harvesting inflates share price and `fetchOperatorFee()` decreases `totalAssets()`, enabling share-price manipulation and fee insolvency.

### Technical details

When rewards are still pending in the Ronin staking contract, `LiquidRon.getTotalRewards()` (lines 276–285) computes the net rewards belonging to vault shareholders by deducting the operator fee portion:

```solidity
totalFees = (totalRewards * operatorFee) / BIPS;
return totalRewards - totalFees;
```

However, once `harvest()` (lines 135–139) is executed:
1. `ILiquidProxy(stakingProxies[_proxyIndex]).harvest(_consensusAddrs)` claims the full reward amount from the staking contract, wraps it into WRON, and transfers the entire `harvestedAmount` directly into the `LiquidRon` vault contract (`_depositRONTo(vault, claimedAmount)`).
2. `LiquidRon.harvest()` increments the accrued fee tracker on line 137:
   ```solidity
   operatorFeeAmount += (harvestedAmount * operatorFee) / BIPS;
   ```
   (Similarly, `harvestAndDelegateRewards()` re-stakes the full reward amount into `getTotalStaked()` and increments `operatorFeeAmount` on line 155.)
3. The vault's `totalAssets()` function (lines 293–295) calculates total managed assets as:
   ```solidity
   function totalAssets() public view override returns (uint256) {
       return super.totalAssets() + getTotalStaked() + getTotalRewards();
   }
   ```
   Because `super.totalAssets()` returns `IERC20(asset()).balanceOf(address(this))` (which now includes the newly harvested `operatorFeeAmount`) and `totalAssets()` does not subtract `operatorFeeAmount`, the accrued operator fee is misaccounted as user-owned vault assets until `fetchOperatorFee()` (lines 121–126) is called to withdraw `operatorFeeAmount` from the vault.

This creates three interrelated accounting flaws:
- **Instant Share Price Inflation on Harvest:** Immediately after `harvest()` or `harvestAndDelegateRewards()`, `totalAssets()` increases by the operator fee `(harvestedAmount * operatorFee) / BIPS` that was previously excluded by `getTotalRewards()`.
- **Share Price Drop on Fee Withdrawal:** When `fetchOperatorFee()` is called (lines 121–126), `_withdrawRONTo(feeRecipient, amount)` removes `operatorFeeAmount` of WRON from the vault, causing `totalAssets()` to drop by `amount`. Users who call `withdraw()` or `redeem()` before `fetchOperatorFee()` extract a portion of the operator's accrued fees, whereas users who withdraw after `fetchOperatorFee()` receive fewer assets per share.
- **Fee Payout Liquidity Mismatch:** If users redeem all shares before `fetchOperatorFee()` is invoked (or if fees are accrued via `harvestAndDelegateRewards()` while little idle WRON sits in the vault), the vault's WRON balance will be less than `operatorFeeAmount`, causing `fetchOperatorFee()` to revert.

### Assumptions and preconditions

- `operatorFee` is non-zero (e.g., initialized to `250` BIPS = 2.5%).
- Staked RON has accrued rewards in the Ronin staking contract, and an operator calls `harvest()` or `harvestAndDelegateRewards()`.
- `fetchOperatorFee()` is not called atomically in the same transaction as every `harvest()`.

### Proof of concept

1. **Initial Deposit & Delegation:**
   - User A and User B each deposit `500 RON` into `LiquidRon` (total `1,000 RON`, `1,000e18` shares; `operatorFee = 1000` BIPS = 10% for illustration).
   - The operator delegates `1,000 RON` to a validator via `delegateAmount()`.
2. **Reward Accrual (Pre-Harvest):**
   - `100 RON` in staking rewards accrue.
   - `getTotalRewards()` returns `100 - 10 = 90 RON`.
   - `totalAssets()` is `0 (vault) + 1,000 (staked) + 90 (net rewards) = 1,090 RON`.
3. **Harvest:**
   - The operator calls `undelegateAmount()` and `harvest()`.
   - `LiquidProxy.harvest()` transfers the full `100 WRON` of rewards to `LiquidRon`. `operatorFeeAmount` becomes `10 RON`.
   - `super.totalAssets()` is now `1,100 WRON`, while `getTotalStaked()` and `getTotalRewards()` are `0`.
   - `totalAssets()` now returns `1,100 RON` instead of `1,090 RON` because `operatorFeeAmount` (`10 RON`) is not deducted.
4. **Front-Running `fetchOperatorFee()` / Unfair Redemption:**
   - User A calls `redeem(500e18, userA, userA)` before `fetchOperatorFee()` is called.
   - User A receives `500 * 1,100 / 1,000 = 550 RON` (capturing `5 RON` of the operator fee). Vault WRON balance drops to `550 WRON`.
   - Next, `feeRecipient` calls `fetchOperatorFee()`, withdrawing `operatorFeeAmount = 10 RON`. Vault WRON balance drops to `540 WRON`.
   - User B then calls `redeem(500e18, userB, userB)` and receives only `540 RON` instead of `545 RON`.
   - Alternatively, if both User A and User B redeem their `500e18` shares before `fetchOperatorFee()`, they withdraw all `1,100 RON`, leaving `0 WRON` in the vault and causing `fetchOperatorFee()` to revert due to insufficient WRON balance.

### Impact assessment

- **Severity:** High
- **Technical Impact:** `totalAssets()` overstates the vault's net assets by `operatorFeeAmount` whenever rewards are harvested. Share value increases artificially on `harvest()` and decreases when `fetchOperatorFee()` is called.
- **Business Impact:** Users who withdraw before `fetchOperatorFee()` steal a portion of the accrued operator fees at the expense of remaining shareholders or the `feeRecipient`. Arbitrageurs can deposit right before `harvest()` and withdraw immediately after to extract the `operatorFeeAmount` jump, or front-run `fetchOperatorFee()` to avoid the share price drop.

### Remediation

Deduct `operatorFeeAmount` when computing `totalAssets()` (or transfer the fee directly to `feeRecipient` / `escrow` upon harvest so that accrued fees are never commingled with user vault assets).

```solidity
    /// @dev Gets the total amount of assets the vault controls
    function totalAssets() public view override returns (uint256) {
        return super.totalAssets() + getTotalStaked() + getTotalRewards() - operatorFeeAmount;
    }
```

## Finding 5: Unbounded Proxy and Validator Aggregation in `LiquidRon.totalAssets` Causes Withdrawal Denial of Service in `LiquidRon.withdraw`

### Synopsis

`LiquidRon.totalAssets()` dynamically aggregates staked balances and rewards across all staking proxies and tracked validators via external calls on every invocation. Because `LiquidRon.withdraw()` invokes `totalAssets()` twice per withdrawal and proxy/validator counts grow without bound, gas consumption scales as $O(P \times V)$, causing withdrawal liveness failures.

### Technical details

In `LiquidRon.sol`, `totalAssets()` (line 294) computes the vault's total managed assets on the fly by summing liquid vault tokens with the results of `getTotalStaked()` and `getTotalRewards()`:

```solidity
function totalAssets() public view override returns (uint256) {
    return super.totalAssets() + getTotalStaked() + getTotalRewards();
}
```

Both helper functions perform live aggregation across all deployed staking proxies (`stakingProxyCount`, denoted $P$) and all tracked consensus addresses (`validatorCount`, denoted $V$):

1. `getTotalStaked()` calls `_getValidators()` (allocating and copying $V$ addresses) and iterates $P$ times over `_getTotalStakedInProxy(i, consensusAddrs)`. On each iteration, `_getTotalStakedInProxy` allocates a `users` array of length $V$, populates it in an $O(V)$ loop, makes an external call to `IRoninValidator(roninStaking).getManyStakingAmounts(_consensusAddrs, users)` (which itself iterates $V$ times and returns a length-$V$ array), and sums the returned array in another $O(V)$ loop.
2. `getTotalRewards()` again calls `_getValidators()` and iterates $P$ times over `_getTotalRewardsInProxy(i, consensusAddrs)`. Each iteration performs an external call to `IRoninValidator(roninStaking).getRewards(user, _consensusAddrs)` (which computes pending rewards across $V$ validators and returns a length-$V$ array) and sums the result in an $O(V)$ loop.

When a user calls `LiquidRon.withdraw(_assets, _receiver, _owner)` (lines 303–311), execution delegates to `ERC4626.withdraw()` (line 308), which calls:
- `maxWithdraw(_owner)` $\rightarrow$ `_convertToAssets()` $\rightarrow$ `totalAssets()`
- `previewWithdraw(_assets)` $\rightarrow$ `_convertToShares()` $\rightarrow$ `totalAssets()`

Consequently, a single `withdraw()` call executes `totalAssets()` twice, triggering $4 \times P$ external `STATICCALL` operations to `roninStaking` and $O(P \times V)$ memory allocations, storage reads, and loop iterations.

This architecture introduces two compounding liveness weaknesses:
- **Unbounded aggregation cost:** `deployStakingProxy()` only appends to `stakingProxies` (`stakingProxyCount++`) with no mechanism to remove or deactivate old proxies. Similarly, `_tryPushValidator()` appends new validators without a maximum cap, and `pruneValidatorList()` must itself execute $O(P \times V)$ external calls and cannot prune a validator if any proxy holds a non-zero stake or reward on it. As $P$ and $V$ grow over the protocol's lifetime, `totalAssets()` and `withdraw()` can exceed the block gas limit.
- **External staking-query failure propagation:** Because `totalAssets()` synchronously queries `roninStaking` across every `(proxy, validator)` pair during `withdraw()`, any revert or failure in `getManyStakingAmounts` or `getRewards` for any single validator or proxy propagates directly into `totalAssets()` and reverts all user withdrawals, even when sufficient liquid WRON is already sitting idle in the vault.

### Assumptions and preconditions

1. The owner deploys multiple `LiquidProxy` contracts over time via `deployStakingProxy()` to manage the 72-hour Ronin staking lockup across delegations and undelegations, and operators delegate across multiple Ronin validators.
2. Either the product $P \times V$ (number of proxies times number of tracked validators) grows large enough that $4P$ external calls and $O(P \times V)$ loops approach or exceed the transaction/block gas limit, or the external `roninStaking` view query reverts for at least one tracked validator/proxy.

### Proof of concept

1. Over normal protocol operation, the owner deploys $P$ staking proxies via `deployStakingProxy()` (none of which can ever be removed because no proxy-removal mechanism exists) and operators delegate stake across $V$ consensus addresses.
2. A user with liquid shares calls `LiquidRon.withdraw(_assets, _receiver, _owner)` (lines 303–311) to withdraw WRON held in the vault.
3. `super.withdraw()` invokes `maxWithdraw(_owner)`, which calls `totalAssets()` (line 294).
4. `totalAssets()` executes `getTotalStaked()` ($P$ external calls to `getManyStakingAmounts`, each iterating over $V$ validators) and `getTotalRewards()` ($P$ external calls to `getRewards`, each iterating over $V$ validators).
5. `super.withdraw()` then invokes `previewWithdraw(_assets)`, which calls `totalAssets()` a second time, repeating all $2P$ external calls and $O(P \times V)$ loops.
6. When $P \times V$ becomes sufficiently large (or if `roninStaking.getRewards` / `getManyStakingAmounts` reverts for any queried validator), `totalAssets()` runs out of gas or reverts, causing `withdraw()` to revert and preventing users from withdrawing their funds.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** `totalAssets()` and `withdraw()` incur $O(P \times V)$ gas overhead with $4P$ external calls per withdrawal, and have a hard liveness dependency on external staking view calls succeeding for every historical proxy and tracked validator.
- **Business Impact:** Users can experience excessive gas costs or a permanent Denial of Service (DoS) when attempting to withdraw funds via `withdraw()` (as well as other ERC-4626 entrypoints relying on `totalAssets()`), freezing user liquidity in the vault.

### Remediation

Track total staked principal in a state variable updated during delegation, reward delegation, and undelegation, and periodically checkpoint or cache accrued rewards (or bound the number of active proxies and validators and allow pruning empty staking proxies) so that `totalAssets()` and `withdraw()` execute in $O(1)$ gas without unbounded external calls.

```solidity
    /// @dev Gets the total amount of assets the vault controls using cached/tracked state
    function totalAssets() public view override returns (uint256) {
        return super.totalAssets() + totalStaked + cachedTotalRewards;
    }

    /// @dev We override to prevent wrong event emission and send native ron back to user
    ///      Acts as the ERC4626 withdraw function
    function withdraw(
        uint256 _assets,
        address _receiver,
        address _owner
    ) public override whenNotPaused returns (uint256) {
        uint256 shares = super.withdraw(_assets, address(this), _owner);
        _withdrawRONTo(_receiver, _assets);
        emit Withdraw(msg.sender, _receiver, _owner, _assets, shares);
        return shares;
    }
```

## Finding 6: Mismatch Between Managed `totalAssets()` and Liquid Vault Reserves Causes `LiquidRon.withdraw` to Revert Within `maxWithdraw` Limits

### Synopsis

In `LiquidRon`, `totalAssets()` aggregates liquid vault tokens with illiquid staked RON and unclaimed rewards, while `withdraw()` (via `super.withdraw()`) relies on `maxWithdraw()` without accounting for actual liquid reserves in the vault, causing withdrawals within reported limits to revert and violating EIP-4626 withdrawal guarantees.

### Technical details

`LiquidRon` inherits from OpenZeppelin's `ERC4626` and overrides `totalAssets()` at line 294:

```solidity
function totalAssets() public view override returns (uint256) {
    return super.totalAssets() + getTotalStaked() + getTotalRewards();
}
```

Here, `super.totalAssets()` returns `IERC20(asset()).balanceOf(address(this))` (the actual liquid `WRON` held by the vault), whereas `getTotalStaked()` and `getTotalRewards()` represent illiquid RON delegated to validators through `LiquidProxy` contracts and unclaimed staking rewards in `roninStaking`.

At lines 303–311, `LiquidRon.withdraw()` overrides `ERC4626.withdraw()`:

```solidity
function withdraw(
    uint256 _assets,
    address _receiver,
    address _owner
) public override whenNotPaused returns (uint256) {
    uint256 shares = super.withdraw(_assets, address(this), _owner);
    _withdrawRONTo(_receiver, _assets);
    emit Withdraw(msg.sender, _receiver, _owner, _assets, shares);
    return shares;
}
```

When `super.withdraw(_assets, address(this), _owner)` executes:
1. It validates `_assets <= maxWithdraw(_owner)`. Because `LiquidRon` does not override `maxWithdraw()`, `ERC4626.maxWithdraw(_owner)` computes `_convertToAssets(balanceOf(_owner), Math.Rounding.Floor)` using `totalAssets()`. This calculation assumes the user's full share value (including staked RON and unharvested validator rewards) is immediately available for withdrawal.
2. It burns `shares` and calls `_withdraw()`, which executes `SafeERC20.safeTransfer(IERC20(asset()), address(this), _assets)`, followed by `_withdrawRONTo(_receiver, _assets)` to unwrap `_assets` of `WRON` from `address(this)`.

Because the vault's actual liquid `WRON` balance is only `super.totalAssets()` (`getAssetsInVault()`), any call to `withdraw()` where `super.totalAssets() < _assets <= maxWithdraw(_owner)` reverts with `ERC20InsufficientBalance` during the token transfer/unwrap. This directly violates the EIP-4626 specification for `maxWithdraw`, which requires `maxWithdraw` to return the maximum amount of underlying assets that can be withdrawn from the `owner` balance in `withdraw` without reverting, and breaks external integrations or routers that size withdrawals using `maxWithdraw()`.

### Assumptions and preconditions

- Users have deposited RON/WRON into `LiquidRon` and received vault shares.
- An operator has delegated a portion or all of the vault's RON to validators via `delegateAmount()`, or staking rewards have accrued in `roninStaking`, so `getTotalStaked() + getTotalRewards() > 0` and `getAssetsInVault() < totalAssets()`.

### Proof of concept

1. Alice deposits `1,000 RON` into `LiquidRon` via `deposit()`, receiving `1,000e18` shares. At this point, `getAssetsInVault() == 1,000e18` and `totalAssets() == 1,000e18`.
2. The operator calls `delegateAmount()` to stake `800 RON` into `roninStaking` via a `LiquidProxy`.
3. Now, `super.totalAssets()` (`getAssetsInVault()`) is `200e18` WRON, `getTotalStaked()` is `800e18`, and `totalAssets()` remains `1,000e18`.
4. Alice (or an integrated EIP-4626 DeFi protocol on her behalf) queries `liquidRon.maxWithdraw(alice)`, which returns `1,000e18`.
5. Alice calls `liquidRon.withdraw(500e18, alice, alice)` (well below `maxWithdraw(alice)`).
6. Inside `super.withdraw()`, the check `_assets <= maxWithdraw(alice)` (`500e18 <= 1,000e18`) passes, and Alice's shares are burned in `_withdraw()`.
7. `SafeERC20.safeTransfer(IERC20(asset()), address(this), 500e18)` attempts to transfer `500e18` WRON from `address(this)`, which only holds `200e18` WRON, causing the transaction to revert with `ERC20InsufficientBalance(address(liquidRon), 200e18, 500e18)`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** `maxWithdraw()` (and `maxRedeem()`) overstates the immediately withdrawable liquidity by including staked and unharvested RON from `totalAssets()`, causing `withdraw()` to revert with `ERC20InsufficientBalance` for valid amounts below `maxWithdraw(owner)`.
- **Business Impact:** Breaks EIP-4626 composability and causes denial of service for integrators, aggregators, and users that rely on `maxWithdraw()` to determine how much can be withdrawn synchronously via `withdraw()` before falling back to the asynchronous `requestWithdrawal()` flow.

### Remediation

Override `maxWithdraw()` (and correspondingly `maxRedeem()`) so that synchronous withdrawals via `withdraw()` are capped by the actual liquid reserves available in the vault (`getAssetsInVault()`, excluding any reserved operator fees):

```solidity
    function maxWithdraw(address _owner) public view override returns (uint256) {
        uint256 ownerAssets = super.maxWithdraw(_owner);
        uint256 availableAssets = getAssetsInVault();
        return ownerAssets < availableAssets ? ownerAssets : availableAssets;
    }

    function maxRedeem(address _owner) public view override returns (uint256) {
        uint256 ownerShares = super.maxRedeem(_owner);
        uint256 availableShares = _convertToShares(getAssetsInVault(), Math.Rounding.Floor);
        return ownerShares < availableShares ? ownerShares : availableShares;
    }
```

## Finding 7: `LiquidRon.totalAssets` fails to deduct `operatorFeeAmount`, inflating share price after `harvest` and `harvestAndDelegateRewards`

### Synopsis

`LiquidRon.totalAssets()` calculates the vault's total managed assets without subtracting `operatorFeeAmount`. When rewards are harvested via `harvest()` or `harvestAndDelegateRewards()`, accrued operator fees remain inside the vault's balance or staked amount, artificially inflating `totalAssets()` and allowing users to withdraw operator fees or suffer unfair share valuation.

### Technical details

In `LiquidRon`, pending staking rewards across all `LiquidProxy` contracts are accounted for in `totalAssets()` via `getTotalRewards()`:

```solidity
function getTotalRewards() public view returns (uint256) {
    address[] memory consensusAddrs = _getValidators();
    uint256 proxyCount = stakingProxyCount;
    uint256 totalRewards;
    uint256 totalFees;

    for (uint256 i = 0; i < proxyCount; i++) totalRewards += _getTotalRewardsInProxy(i, consensusAddrs);
    totalFees = (totalRewards * operatorFee) / BIPS;
    return totalRewards - totalFees;
}
```

Before rewards are harvested, `getTotalRewards()` properly deducts the operator fee (`totalFees`) from `totalRewards`.

However, when an operator calls `harvest()` or `harvestAndDelegateRewards()`:
1. In `harvest()`, the proxy claims the full `harvestedAmount` of rewards from `roninStaking`, wraps it into `WRON`, and transfers the entire `harvestedAmount` (including the fee portion) to the `LiquidRon` vault. `operatorFeeAmount` is incremented by `(harvestedAmount * operatorFee) / BIPS`.
2. In `harvestAndDelegateRewards()`, the proxy claims and re-stakes the full `harvestedAmount` (including the fee portion) into `roninStaking`. `operatorFeeAmount` is similarly incremented by `(harvestedAmount * operatorFee) / BIPS`.

In both cases, once the rewards are harvested:
- The unclaimed rewards on `roninStaking` become `0`, so `getTotalRewards()` no longer deducts the fee for those harvested rewards.
- The full `harvestedAmount` (including the operator fee portion) is now part of `super.totalAssets()` (the vault's `WRON` balance) or `getTotalStaked()`.
- Because `totalAssets()` (at line 294) is computed as:
  ```solidity
  function totalAssets() public view override returns (uint256) {
      return super.totalAssets() + getTotalStaked() + getTotalRewards();
  }
  ```
  it does not subtract `operatorFeeAmount`.

Consequently, immediately after `harvest()` or `harvestAndDelegateRewards()` is executed, `totalAssets()` increases by `operatorFeeAmount` until `fetchOperatorFee()` is called by `feeRecipient`. During this window:
- The share price (`totalAssets() / totalSupply()`) is artificially inflated.
- Users calling `withdraw()` or `redeem()` receive more `RON` than they are entitled to, directly extracting funds reserved for `operatorFeeAmount`.
- Users calling `deposit()` or `mint()` receive fewer shares than they should for their deposited assets.
- If `finaliseRonRewardsForEpoch()` is called, the locked price per share for the epoch is calculated using the inflated `previewRedeem(lockedShares)` value.
- If shareholders redeem their shares before `fetchOperatorFee()` is called, the vault will not have enough `WRON` left to pay `operatorFeeAmount`, causing `fetchOperatorFee()` to revert or draw from remaining users' principal.

### Assumptions and preconditions

1. `operatorFee` is greater than `0` (e.g., configured to `250` BIPS / 2.5% in the deployment script).
2. Staking rewards have accrued and an operator calls `harvest()` or `harvestAndDelegateRewards()`, causing `operatorFeeAmount > 0`.
3. `feeRecipient` has not yet called `fetchOperatorFee()` in the same transaction.

### Proof of concept

1. Suppose a user deposits `1000 RON` into `LiquidRon` and receives `1000e18` shares. The operator delegates the `1000 RON` to a validator, and `operatorFee` is set to `10%` (`1000` BIPS) for simplicity.
2. After some time, `100 RON` of staking rewards accrue on the validator.
   - Before harvesting:
     - `super.totalAssets()` = `0`
     - `getTotalStaked()` = `1000 RON`
     - `getTotalRewards()` = `100 - 10 = 90 RON`
     - `totalAssets()` = `1090 RON`
3. The operator calls `harvest()` (or `undelegateAmount()` + `harvest()`).
   - `100 RON` is claimed and deposited as `WRON` into `LiquidRon`.
   - `operatorFeeAmount` increases from `0` to `10 RON`.
   - Now:
     - `super.totalAssets()` = `100 RON` (plus `1000 RON` if undelegated)
     - `getTotalRewards()` = `0`
     - `totalAssets()` = `1100 RON` (inflated by the `10 RON` `operatorFeeAmount`).
4. The user calls `withdraw(1100 ether, user, user)` or `redeem(1000 ether, user, user)`.
   - The user burns all `1000e18` shares and receives the full `1100 RON`, including the `10 RON` owed to `feeRecipient`.
5. When `feeRecipient` subsequently calls `fetchOperatorFee()`, the transaction reverts because the vault holds `0 WRON` while `operatorFeeAmount` is `10 RON`.

### Impact assessment

- **Severity**: High / Medium
- **Technical Impact**: `totalAssets()` is overstated by `operatorFeeAmount` whenever `operatorFeeAmount > 0`. This distorts share-to-asset conversions across all ERC-4626 operations (`deposit`, `mint`, `withdraw`, `redeem`) and epoch withdrawal finalization (`finaliseRonRewardsForEpoch`).
- **Business Impact**: Exiting users can steal accrued operator fees (`operatorFeeAmount`), depositors receive fewer shares than warranted, and `feeRecipient` may be unable to claim accrued fees via `fetchOperatorFee()` due to insufficient vault balance or may inadvertently drain remaining users' assets when claiming fees.

### Remediation

Update `totalAssets()` in `src/LiquidRon.sol` to subtract `operatorFeeAmount`:

```solidity
    /// @dev Gets the total amount of assets the vault controls
    function totalAssets() public view override returns (uint256) {
        return super.totalAssets() + getTotalStaked() + getTotalRewards() - operatorFeeAmount;
    }
```

## Finding 8: Unchecked `IWRON.transfer` Return Value in `RonHelper._depositRONTo` Risks Silent Transfer Failure and Stuck or Misaccounted Funds

### Synopsis

In `RonHelper._depositRONTo`, the call to `IWRON(wron).transfer(to, amount)` omits the ERC-20 boolean return value in both the `IWRON` interface definition and the call site. If the token transfer returns `false` instead of reverting, execution continues silently, leaving wrapped RON stuck or corrupting escrow accounting.

### Technical details

In `src/RonHelper.sol`, the custom interface `IWRON` declares `transfer` without the standard ERC-20 `returns (bool)` return value:

```solidity
interface IWRON {
    function deposit() external payable;
    function withdraw(uint256) external;
    function transfer(address, uint256) external;
}
```

In `RonHelper._depositRONTo` (lines 30–33), native RON is wrapped into WRON and then transferred to the recipient `to`:

```solidity
function _depositRONTo(address to, uint256 amount) internal {
    IWRON(wron).deposit{value: amount}();
    IWRON(wron).transfer(to, amount);
}
```

Unlike `LiquidRon._withdraw` (which uses `SafeERC20.safeTransfer(IERC20(asset()), receiver, assets)`), `_depositRONTo` invokes `IWRON(wron).transfer(to, amount)` directly without checking the boolean return value or using OpenZeppelin's `SafeERC20.safeTransfer`.

Because the ERC-20 standard permits `transfer` to signal failure by returning `false` rather than reverting, a failed transfer in `_depositRONTo` will not revert the transaction:

1. **In `LiquidProxy.harvest` and `LiquidProxy.undelegateAmount`:** `_depositRONTo(vault, amount)` wraps the claimed or undelegated native RON into WRON inside the `LiquidProxy` contract. If `transfer(vault, amount)` returns `false`, the transaction still succeeds, leaving the WRON tokens permanently locked inside `LiquidProxy` (which has no function to transfer out ERC-20 tokens) while `harvest` still returns `claimedAmount` to `LiquidRon` and accrues operator fees on rewards that never reached the vault.
2. **In `LiquidRon.deposit()` and `LiquidRon.receive()`:** `_depositRONTo(escrow, msg.value)` mints `msg.value` of WRON to `LiquidRon` and attempts to transfer it to `escrow` before calling `Escrow(escrow).deposit(msg.value, msg.sender)`. Because `escrow` also holds WRON reserved for finalized user withdrawal requests, if `transfer(escrow, msg.value)` returns `false` while `escrow` holds at least `msg.value` WRON for pending withdrawals, `Escrow.deposit` will still succeed by pulling `msg.value` WRON from `escrow`'s withdrawal reserves into `LiquidRon`. As a result, `LiquidRon` retains both the newly wrapped `msg.value` WRON and the `msg.value` WRON pulled from `escrow`, inflating `totalAssets()` and depleting `escrow` funds needed for user withdrawal claims in `LiquidRon.redeem(uint256 _epoch)`.

### Assumptions and preconditions

1. The underlying `wron` token implementation returns `false` on a failed `transfer` call under certain failure conditions rather than reverting.
2. Either a `LiquidProxy` operation (`harvest` or `undelegateAmount`) or a native RON deposit (`LiquidRon.deposit()` or `LiquidRon.receive()`) is executed when `wron.transfer` returns `false`.

### Proof of concept

1. Assume `escrow` currently holds `100e18` WRON reserved for finalized user withdrawal requests.
2. A user calls `LiquidRon.deposit{value: 100e18}()`.
3. `LiquidRon.deposit()` calls `_depositRONTo(escrow, 100e18)`.
4. `IWRON(wron).deposit{value: 100e18}()` mints `100e18` WRON to `LiquidRon`.
5. `IWRON(wron).transfer(escrow, 100e18)` fails and returns `false` without reverting. Because `_depositRONTo` ignores the return value, execution continues with `100e18` WRON remaining in `LiquidRon` and `escrow` still holding only its original `100e18` WRON.
6. `Escrow(escrow).deposit(100e18, msg.sender)` calls `LiquidRon.deposit(100e18, msg.sender)`, which transfers the existing `100e18` WRON from `escrow` to `LiquidRon` and mints shares to `msg.sender`.
7. `LiquidRon` has now gained `200e18` WRON from a `100e18` RON deposit, while `escrow` has `0` WRON left, causing subsequent calls to `LiquidRon.redeem(_epoch)` by users with finalized withdrawals to revert due to insufficient WRON balance in `escrow`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Unchecked ERC-20 `transfer` failure in `_depositRONTo` leads to silent loss of WRON transfers, permanently locking harvested or undelegated WRON inside `LiquidProxy` contracts or consuming `escrow` withdrawal reserves during native RON deposits.
- **Business Impact:** Potential permanent freezing of staking rewards/principal inside `LiquidProxy` and insolvency of the `Escrow` contract for users claiming finalized withdrawals.

### Remediation

Use OpenZeppelin's `SafeERC20.safeTransfer` (or check the boolean return value of `IERC20(wron).transfer`) in `RonHelper._depositRONTo` to ensure the transaction reverts if the token transfer does not succeed.

```solidity
    /// @dev Deposit RON tokens into WRON to the recipient
    /// @param to The recipient of the WRON tokens
    /// @param amount The amount of RON to deposit
    function _depositRONTo(address to, uint256 amount) internal {
        IWRON(wron).deposit{value: amount}();
        SafeERC20.safeTransfer(IERC20(wron), to, amount);
    }
```
