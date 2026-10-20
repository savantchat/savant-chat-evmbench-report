# Audit report — 2025-04-virtuals/H-01

## Finding 1: Unnormalized Validator Vote Weighting in `AgentRewardV2._distributeValidatorRewards` Breaks Reward Budget Conservation

### Synopsis

In `AgentRewardV2._distributeValidatorRewards`, each validator's share is computed as `(amount * votes) / totalStaked` using current `getVotes(validator)`, and only `validatorRewards - participationReward` is added to `validatorPoolRewards`. Consequently, `sum(validatorRewards)` diverges from `amount`, permanently locking unallocated rewards or over-allocating beyond the validator reward budget.

### Technical details

In `AgentRewardV2._distributeAgentRewards`, the total reward allocated to the validator pool for a given `virtualId` is computed on line 208 as `reward.validatorAmount = amount - reward.contributorAmount` and passed as `amount` into `_distributeValidatorRewards(reward.validatorAmount, virtualId, reward.id, reward.totalStaked)`.

Inside `_distributeValidatorRewards` (lines 231–249), the contract iterates over all registered validators and computes each validator's allocation:

```solidity
uint256 votes = IERC5805(stakingAddress).getVotes(validator);
uint256 validatorRewards = (amount * votes) / totalStaked;

uint256 participationReward = totalProposals == 0
    ? 0
    : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
_validatorRewards[validator][rewardId] = participationReward;

validatorPoolRewards += validatorRewards - participationReward;
```

This accounting violates budget conservation (`sum(participationReward) + delta(validatorPoolRewards) == amount`) in two ways:

1. **Under-allocation and permanently stranded funds (`sum(validatorRewards) < amount`):**
   Under `ERC20Votes` / `VotesUpgradeable`, token balances only count toward `getVotes(delegatee)` when explicitly delegated, and stakers may hold undelegated tokens (`delegates(account) == address(0)`) or delegate to an address that is not registered as a validator in `AgentNft`. Furthermore, each loop iteration truncates `(amount * votes) / totalStaked`. Because `validatorPoolRewards` only accumulates `validatorRewards - participationReward` per validator rather than `amount - sum(participationReward)`, the unallocated portion `amount - sum(validatorRewards)` is credited to neither `_validatorRewards` nor `validatorPoolRewards` and remains permanently locked in `AgentRewardV2`.

2. **Over-allocation / insolvency (`sum(validatorRewards) > amount`):**
   Line 239 reads the current live voting weight `IERC5805(stakingAddress).getVotes(validator)` at the time `_distributeValidatorRewards` is executed, whereas `totalStaked` is fixed at `reward.totalStaked`. If additional tokens are staked and delegated to validators before `_distributeValidatorRewards` is called for that `reward`, the sum of current `votes` across validators can exceed `reward.totalStaked`. In that case, `sum(validatorRewards) > amount`, crediting more total rewards across `_validatorRewards` and `validatorPoolRewards` than the `reward.validatorAmount` funded for this virtual agent.

### Assumptions and preconditions

- A virtual persona (`virtualId`) has been initialized and has a reward entry in `_rewards[virtualId]` with `reward.validatorAmount > 0`.
- Either (a) some staked tokens are undelegated, delegated to a non-validator address, or produce division rounding across validators (`sum(votes) < totalStaked`), or (b) additional stake is delegated to registered validators between the recording of `reward.totalStaked` and the execution of `_distributeValidatorRewards` (`sum(votes) > totalStaked`).

### Proof of concept

1. Suppose `virtualId = 1` has `reward.totalStaked = 100,000e18` and `reward.validatorAmount = 10,000e18` (`amount = 10,000e18`).
2. Validator `V1` has `60,000e18` delegated votes, while `40,000e18` of staked tokens belong to stakers who have not delegated to a registered validator (or left `delegates` as `address(0)`).
3. When `_distributeValidatorRewards(10,000e18, 1, rewardId, 100,000e18)` executes:
   - For `V1`, `votes = 60,000e18`, so `validatorRewards = (10,000e18 * 60,000e18) / 100,000e18 = 6,000e18`.
   - Suppose `V1` participated in 100% of proposals: `participationReward = 6,000e18`, `_validatorRewards[V1][rewardId] = 6,000e18`, and `validatorPoolRewards += 0`.
4. The remaining `4,000e18` of `reward.validatorAmount` is never assigned to `_validatorRewards` and never added to `validatorPoolRewards`. It cannot be claimed by validators, stakers, or governance via `withdrawValidatorPoolRewards`, permanently locking `4,000e18` reward tokens in the contract.
5. Conversely, if a user stakes and delegates an additional `50,000e18` to `V1` after `reward.totalStaked = 100,000e18` was recorded but before `_distributeValidatorRewards` runs, `getVotes(V1)` returns `150,000e18`. Line 240 computes `validatorRewards = (10,000e18 * 150,000e18) / 100,000e18 = 15,000e18`, allocating `5,000e18` more rewards than the `10,000e18` budget available.

### Impact assessment

- **Severity:** High / Medium (Loss of funds / Protocol insolvency)
- **Technical Impact:** Total distributed validator and penalty pool rewards fail to equal `reward.validatorAmount`. Unallocated tokens are permanently frozen in `AgentRewardV2`, while over-allocated rewards drain tokens reserved for other virtual agents, contributors, or stakers.

### Remediation

Query historical voting power at the reward's snapshot block (`getPastVotes`) so that validator voting weights correspond to the same checkpoint as `totalStaked`, cap or normalize the allocation, and assign the full unallocated remainder `amount - totalParticipationRewards` to `validatorPoolRewards`:

```solidity
    function _distributeValidatorRewards(
        uint256 amount,
        uint256 virtualId,
        uint48 rewardId,
        uint256 totalStaked
    ) private {
        if (totalStaked == 0 || amount == 0) {
            validatorPoolRewards += amount;
            return;
        }

        IAgentNft nft = IAgentNft(agentNft);
        uint256 validatorCount = nft.validatorCount(virtualId);
        uint256 totalProposals = nft.totalProposals(virtualId);
        address stakingAddress = getVirtualTokenAddress(nft, virtualId);
        uint32 rewardBlock = _mainRewards[_rewards[virtualId][_rewards[virtualId].length - 1].mainIndex].blockNumber;

        uint256 totalDistributed;
        for (uint256 i = 0; i < validatorCount; i++) {
            address validator = nft.validatorAt(virtualId, i);

            uint256 votes = IERC5805(stakingAddress).getPastVotes(validator, rewardBlock);
            uint256 validatorRewards = (amount * votes) / totalStaked;

            uint256 participationReward = totalProposals == 0
                ? 0
                : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
            _validatorRewards[validator][rewardId] = participationReward;
            totalDistributed += participationReward;
        }

        validatorPoolRewards += amount - totalDistributed;
    }
```

## Finding 2: Unbounded Validator Registry Growth in `ValidatorRegistry._addValidator` Enables Denial of Service of Reward Distribution

### Synopsis

`ValidatorRegistry._addValidator` appends validator addresses to the `_validators[virtualId]` array without any upper bound or removal mechanism, and `AgentNftV2.addValidator` exposes this registration permissionlessly. An attacker can bloat `_validators[virtualId]`, causing functions that iterate over all validators—such as `AgentRewardV2._distributeValidatorRewards` and `ValidatorRegistry.totalUptimeScore`—to revert due to block gas limit exhaustion.

### Technical details

In `contracts/virtualPersona/ValidatorRegistry.sol`, `_addValidator` (lines 30–34) records each new validator in a mapping and appends the address to the dynamic array `_validators[virtualId]`:

```solidity
function _addValidator(uint256 virtualId, address validator) internal {
    _validatorsMap[virtualId][validator] = true;
    _validators[virtualId].push(validator);
    emit NewValidator(virtualId, validator);
}
```

This design exhibits three compounding issues:

1. **Adversarial persistent-state expansion**: `AgentNftV2.addValidator(uint256 virtualId, address validator)` (lines 133–139 of `AgentNftV2.sol`) is a `public` function with no access control or minimum stake validation. Any caller can invoke `addValidator` with arbitrary `validator` addresses for any `virtualId`, repeatedly executing `_validators[virtualId].push(validator)` and `_initValidatorScore(virtualId, validator)`. Furthermore, there is no function in `ValidatorRegistry` to remove inactive or zero-stake validators.
2. **Unbounded enumeration costs**: Downstream consumers iterate over the entire `_validators[virtualId]` array via `validatorCount(virtualId)`. Specifically:
   - `AgentRewardV2._distributeValidatorRewards` (lines 223–250 of `AgentRewardV2.sol`) loops from `i = 0` to `validatorCount` on every reward distribution for a `virtualId`.
   - `ValidatorRegistry.totalUptimeScore` (lines 60–66 of `ValidatorRegistry.sol`) loops from `i = 0` to `validatorCount(virtualId)` to sum `validatorScore`.
3. **Per-entry external-call amplification**: Inside the loop in `AgentRewardV2._distributeValidatorRewards` (lines 234–249), each iteration performs multiple external calls and storage operations:
   - `nft.validatorAt(virtualId, i)`
   - `getVirtualTokenAddress(nft, virtualId)`
   - `IERC5805(stakingAddress).getVotes(validator)`
   - `nft.validatorScore(virtualId, validator)`, which in turn performs an external call `IAgentDAO(info.dao).scoreOf(account)` (line 147 of `AgentNftV2.sol`)
   - Storage writes to `_validatorRewards[validator][rewardId]` and `validatorPoolRewards`

Once `_validators[virtualId]` accumulates enough entries, the gas required to execute the loop in `_distributeValidatorRewards` exceeds the transaction/block gas limit, permanently blocking reward distribution for that `virtualId`.

### Assumptions and preconditions

- A valid `virtualId` exists in `AgentNftV2` (or is targeted prior to minting).
- An attacker has sufficient gas to submit transactions calling `AgentNftV2.addValidator(virtualId, validator)` with distinct addresses (e.g., via a batch helper contract).

### Proof of concept

1. Suppose `virtualId = 1` is an active agent persona receiving daily rewards via `AgentRewardV2.distributeRewards`.
2. An attacker deploys a small batching contract that loops and calls `AgentNftV2.addValidator(1, address(uint160(i)))` for thousands of distinct addresses `i`.
3. For each call, `isValidator(1, validator)` is `false`, so `_addValidator(1, validator)` sets `_validatorsMap[1][validator] = true` and pushes `validator` to `_validators[1]`.
4. When the protocol backend subsequently calls `AgentRewardV2.distributeRewards`, execution enters `_distributeValidatorRewards(amount, 1, rewardId, totalStaked)`.
5. `_distributeValidatorRewards` queries `validatorCount = nft.validatorCount(1)` and attempts to iterate through all entries in `_validators[1]`, executing multiple external calls (`validatorAt`, `getVotes`, `validatorScore` -> `dao.scoreOf`) and storage accesses per entry.
6. The transaction runs out of gas and reverts, permanently preventing rewards from being distributed for `virtualId = 1`.

### Impact assessment

- **Severity**: High
- **Technical Effect**: Permanent Denial of Service (DoS) of `AgentRewardV2._distributeValidatorRewards` (and `ValidatorRegistry.totalUptimeScore`) for any targeted `virtualId`, or across the entire `distributeRewards` batch if the affected `virtualId` is included.
- **Business Impact**: Core protocol reward distribution for stakers, validators, and contributors of affected virtual personas is frozen, halting daily profit distribution as described in the protocol specification.

### Remediation

Restrict who can register a validator (e.g., only the staking contract or authorized roles when a meaningful minimum stake is delegated), cap the maximum number of active validators per `virtualId`, or refactor reward distribution to use a pull-based checkpoint calculation (or paginated distribution) rather than an unbounded loop over `_validators[virtualId]`.

In `ValidatorRegistry.sol`, enforce a maximum validator cap in `_addValidator`:

```solidity
    uint256 public constant MAX_VALIDATORS = 100;

    function _addValidator(uint256 virtualId, address validator) internal {
        require(_validators[virtualId].length < MAX_VALIDATORS, "Max validators reached");
        _validatorsMap[virtualId][validator] = true;
        _validators[virtualId].push(validator);
        emit NewValidator(virtualId, validator);
    }
```

Additionally, ensure `AgentNftV2.addValidator` is access-controlled (e.g., restricted to `VALIDATOR_ADMIN_ROLE` or the corresponding `veToken` contract) so arbitrary addresses cannot fill the validator set.

## Finding 3: Token-Source Divergence Between Validator Voting Weight and Total Staked in `AgentRewardV2._distributeValidatorRewards` Causes Revert or Corrupted Reward Accounting

### Synopsis

In `AgentRewardV2._distributeValidatorRewards`, validator voting power `votes` is queried from `getVirtualTokenAddress(nft, virtualId)` (`prevTokens[virtualId]` or `nft.virtualInfo(virtualId).token`), whereas `totalStaked` and `AgentNftV2` staking power use `virtualLPs[virtualId].veToken`. This token-source mismatch reverts reward distribution or miscalculates validator rewards.

### Technical details

In `AgentRewardV2._distributeValidatorRewards` (`contracts/AgentRewardV2.sol`, lines 238–240), each validator's share of the reward `amount` is calculated as:

```solidity
address stakingAddress = getVirtualTokenAddress(nft, virtualId);
uint256 votes = IERC5805(stakingAddress).getVotes(validator);
uint256 validatorRewards = (amount * votes) / totalStaked;
```

`getVirtualTokenAddress(nft, virtualId)` (lines 561–567) resolves `stakingAddress` to `prevTokens[virtualId]` if configured, and falls back to `nft.virtualInfo(virtualId).token` otherwise.

However, in `AgentNftV2` (`contracts/virtualPersona/AgentNftV2.sol`):
1. During `mint` (lines 103–108) and `migrateVirtual` (lines 267–273), `virtualInfos[virtualId].token` is set to the standard ERC20 persona token (`AgentToken`), while the `IERC5805` voting escrow token (`AgentVeToken`) that tracks staked balances and delegated votes is stored in `virtualLPs[virtualId].veToken`.
2. `AgentNftV2.totalStaked(virtualId)` (lines 194–196) returns `IERC20(virtualLPs[virtualId].veToken).totalSupply()`, and `AgentNftV2.getVotes(virtualId, validator)` (lines 198–200) returns `IERC5805(virtualLPs[virtualId].veToken).getVotes(validator)`.

Consequently, `_distributeValidatorRewards` suffers from a token-source divergence between `votes` and `totalStaked`:
- **When `prevTokens[virtualId] == address(0)`:** `getVirtualTokenAddress` returns `nft.virtualInfo(virtualId).token` (the non-voting ERC20 `AgentToken` rather than `virtualLPs[virtualId].veToken`). Calling `IERC5805(stakingAddress).getVotes(validator)` either reverts (if the ERC20 token does not implement `IERC5805`) or reads votes from a different token than `virtualLPs[virtualId].veToken`.
- **When `prevTokens[virtualId] != address(0)`:** `votes` is read from the legacy token `prevTokens[virtualId]`, while `totalStaked` in `AgentNftV2.totalStaked(virtualId)` is read from `virtualLPs[virtualId].veToken.totalSupply()` (or does not match the supply of `stakingAddress`). If legacy `votes` exceed the new `veToken` `totalStaked`, `validatorRewards` exceeds `amount`, over-allocating `_validatorRewards` and inflating `validatorPoolRewards`. If `votes` is smaller relative to `totalStaked`, validator rewards are under-allocated and the unallocated remainder is permanently locked in the contract.

### Assumptions and preconditions

1. `AgentRewardV2` is configured with `agentNft` pointing to `AgentNftV2` (either via proxy upgrade or `updateRefContracts`).
2. A virtual persona is created via `AgentNftV2.mint` or migrated via `AgentNftV2.migrateVirtual`, so its staking/voting token is stored in `virtualLPs[virtualId].veToken`.
3. Reward distribution triggers `_distributeAgentRewards` and `_distributeValidatorRewards` for `virtualId`.

### Proof of concept

1. A virtual persona `virtualId = 1` is migrated on `AgentNftV2` via `migrateVirtual(1, dao, agentToken, pool, veToken)`, where `veToken` has `totalSupply() == 10,000e18` (`nft.totalStaked(1) == 10,000e18`).
2. **Case A (`prevTokens[1] == address(0)`):**
   - `getVirtualTokenAddress(nft, 1)` returns `nft.virtualInfo(1).token`, which is `agentToken` (a standard ERC20 without `IERC5805.getVotes`).
   - When `_distributeValidatorRewards` executes line 239 (`IERC5805(stakingAddress).getVotes(validator)`), the call reverts, blocking reward distribution.
3. **Case B (`prevTokens[1]` set to the legacy V1 staking token):**
   - On the legacy token `prevTokens[1]`, `validator` still has `votes = 50,000e18`, while on `AgentNftV2`, `totalStaked = veToken.totalSupply() = 10,000e18`.
   - In `_distributeValidatorRewards` with `amount = 1,000e18`:
     `validatorRewards = (1,000e18 * 50,000e18) / 10,000e18 = 5,000e18`, which is 5x the total `amount = 1,000e18` allocated to the virtual persona's validator pool.
   - `_validatorRewards[validator][rewardId]` and `validatorPoolRewards` are credited with `5,000e18` in total, allowing `claimAllRewards` / `withdrawValidatorPoolRewards` to drain `4,000e18` of other users' reward tokens held by `AgentRewardV2`.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Either reward distribution reverts (Denial of Service when `stakingAddress` resolves to `virtualInfo(virtualId).token` which is not an `IERC5805` contract) or validator rewards are calculated using mismatched numerator (`votes`) and denominator (`totalStaked`) token sources.
- **Business Impact:** Overstated `validatorRewards` inflate `_validatorRewards` and `validatorPoolRewards` beyond the allocated `reward.validatorAmount`, allowing validators or pool withdrawals to drain reward tokens belonging to other virtual personas/contributors, while understated `validatorRewards` permanently strand reward tokens in the contract.

### Remediation

Ensure that `votes` and `totalStaked` are sourced from the same staking token. If `AgentRewardV2` is intended to use `nft.getVotes(virtualId, validator)` consistent with `nft.totalStaked(virtualId)` (or to derive both `votes` and `totalStaked` from the same resolved staking token address outside the loop), update `_distributeValidatorRewards` accordingly:

```solidity
        uint256 validatorCount = nft.validatorCount(virtualId);
        uint256 totalProposals = nft.totalProposals(virtualId);
        address stakingAddress = getVirtualTokenAddress(nft, virtualId);
        uint256 stakingTotalSupply = IERC20(stakingAddress).totalSupply();
        if (stakingTotalSupply == 0) {
            return;
        }

        for (uint256 i = 0; i < validatorCount; i++) {
            address validator = nft.validatorAt(virtualId, i);

            // Get validator revenue by votes weightage
            uint256 votes = IERC5805(stakingAddress).getVotes(validator);
            uint256 validatorRewards = (amount * votes) / stakingTotalSupply;
```

Alternatively, if `AgentRewardV2` should use `AgentNftV2`'s active `veToken` matching `nft.totalStaked(virtualId)`, query `nft.getVotes(virtualId, validator)` (or `nft.virtualLP(virtualId).veToken` when `prevTokens[virtualId] == address(0)`).

## Finding 4: Checkpoint Misalignment Between Live Distribution State and Historical Snapshot in `AgentRewardV2._distributeValidatorRewards` Enables Reward Manipulation and Insolvency

### Synopsis

`AgentRewardV2._distributeValidatorRewards` calculates validator group rewards and penalty pool allocations using live state (`getVotes`, `validatorScore`, `totalProposals`, and current validator set), whereas `reward.totalStaked` and reward claim calculations (`_getClaimableStakerRewardsAt`) use historical snapshots at `mainReward.blockNumber`. This checkpoint misalignment enables reward theft, unfair forfeiture, and contract insolvency.

### Technical details

In `AgentRewardV2`, reward creation and per-agent reward distribution occur in separate transactions:

1. A `MainReward` checkpoint is created at `mainReward.blockNumber`, and the agent's `Reward` struct records `reward.totalStaked` at that snapshot block.
2. Later, `distributeRewardsForAgents(uint32 mainRewardIndex, uint256[] memory virtualIds)` calls `_distributeAgentRewards`, which invokes `_distributeValidatorRewards(reward.validatorAmount, virtualId, reward.id, reward.totalStaked)` (`contracts/AgentRewardV2.sol:210`).
3. When stakers and validators subsequently claim rewards via `_getClaimableStakerRewardsAt` (`contracts/AgentRewardV2.sol:308-332`) and `_getClaimableValidatorRewardsAt` (`contracts/AgentRewardV2.sol:351-363`), their claims are evaluated against historical checkpoints at `mainReward.blockNumber`:
   - `delegatee = token.getPastDelegates(account, mainReward.blockNumber)` (line 318)
   - `settings = getPastRewardSettings(mainReward.blockNumber)` (lines 324, 358)
   - `tokens = token.getPastBalanceOf(account, mainReward.blockNumber)` (line 328)
   - `votes = IERC5805(stakingAddress).getPastVotes(delegatee, mainReward.blockNumber)` (line 329)
   - Claimable staker reward: `(((validatorGroupRewards * tokens) / votes) * uint256(settings.stakerShares)) / DENOMINATOR` (line 331)

However, `_distributeValidatorRewards` (`contracts/AgentRewardV2.sol:223-250`) does not evaluate validator weights or scores at `mainReward.blockNumber`:
- Line 232 fetches the live proposal count `nft.totalProposals(virtualId)`.
- Line 239 fetches live voting power `IERC5805(stakingAddress).getVotes(validator)` instead of `getPastVotes(validator, mainReward.blockNumber)`, and line 240 divides this live value by the historical `reward.totalStaked`.
- Lines 243-246 (`243-246`) compute `participationReward` using the live score `nft.validatorScore(virtualId, validator)` instead of the historical `nft.getPastValidatorScore(virtualId, validator, mainReward.blockNumber)`, storing the result in `_validatorRewards[validator][rewardId]`.
- Line 248 (`248`) adds `validatorRewards - participationReward` to `validatorPoolRewards`.

This mismatch between live allocation inputs and historical snapshot inputs creates multiple exploitable inconsistencies:

1. **Over-Allocation and Contract Insolvency via Post-Snapshot Staking**: If additional tokens are staked and delegated to validators after `mainReward.blockNumber` but before `distributeRewardsForAgents` is called, the sum of live `getVotes(validator)` across validators exceeds the historical `reward.totalStaked`. Consequently, the sum of `validatorRewards` (allocated across `_validatorRewards[validator][rewardId]` at line 246 and `validatorPoolRewards` at line 248) exceeds `reward.validatorAmount`, causing total obligations to exceed the reward tokens held by the contract.
2. **Staker Reward Theft and Loss via Post-Snapshot Delegation Changes**: If a staker was delegated to Validator A at `mainReward.blockNumber` and re-delegates to Validator B before `distributeRewardsForAgents` executes:
   - Validator A's live `getVotes(A)` decreases, reducing or zeroing `_validatorRewards[A][rewardId]`. Because the staker's `getPastDelegates(staker, mainReward.blockNumber)` is still Validator A, the staker's claimable reward is reduced or wiped out.
   - Validator B's live `getVotes(B)` increases, inflating `_validatorRewards[B][rewardId]`. When existing stakers who were already delegated to Validator B at `mainReward.blockNumber` call `_getClaimableStakerRewardsAt`, their share is divided by `getPastVotes(B, mainReward.blockNumber)` (which excludes the newly re-delegated votes). Those existing stakers therefore claim the entire inflated `validatorGroupRewards` of Validator B, stealing the rewards generated by the re-delegated voting power.
   - If Validator B had `0` votes at `mainReward.blockNumber` (`getPastVotes(B, mainReward.blockNumber) == 0`), the staker share of `_validatorRewards[B][rewardId]` becomes permanently unclaimable because no staker had `B` as `getPastDelegates` at `mainReward.blockNumber`.
3. **Participation Score Manipulation Across Stages**: Because `245` queries live `nft.validatorScore(virtualId, validator)` and `AgentNftV2.addValidator` (`contracts/virtualPersona/AgentNftV2.sol:133-139`) is permissionless and initializes `_baseValidatorScore` to current `totalProposals`, any delegatee with votes at `mainReward.blockNumber` who was not registered (or who missed proposals before registration) can call `addValidator` just before `distributeRewardsForAgents` to obtain a full participation score (`validatorScore == totalProposals`), bypassing participation penalties in `243-248`.

### Assumptions and preconditions

1. A `MainReward` checkpoint has been created at `mainReward.blockNumber`, and `distributeRewardsForAgents` is executed in a subsequent block (`block.number > mainReward.blockNumber`).
2. Staking balances, vote delegations, validator registrations, or DAO proposal scores change between `mainReward.blockNumber` and the block in which `distributeRewardsForAgents` is executed.

### Proof of concept

1. At `mainReward.blockNumber = 100`, an agent has `reward.totalStaked = 1,000e18`:
   - Staker 1 holds `100e18` veTokens delegated to Validator V (`getPastVotes(V, 100) = 100e18`).
   - Staker 2 holds `900e18` veTokens delegated to Validator W (`getPastVotes(W, 100) = 900e18`).
2. Suppose `reward.validatorAmount = 1,000e18`, `stakerShares = 9000` (90%), and both V and W have 100% participation (`validatorScore == totalProposals`).
3. At block 101 (before `distributeRewardsForAgents` is called), Staker 2 re-delegates their `900e18` votes from Validator W to Validator V. Now live `getVotes(V) = 1,000e18` and live `getVotes(W) = 0`.
4. At block 102, `GOV_ROLE` calls `distributeRewardsForAgents`:
   - For Validator V: `validatorRewards = (1,000e18 * 1,000e18) / 1,000e18 = 1,000e18`. Line 246 sets `_validatorRewards[V][reward.id] = 1,000e18`.
   - For Validator W: `validatorRewards = (1,000e18 * 0) / 1,000e18 = 0`. Line 246 sets `_validatorRewards[W][reward.id] = 0`.
5. When Staker 1 claims rewards, `_getClaimableStakerRewardsAt` looks up `mainReward.blockNumber = 100`:
   - `delegatee = getPastDelegates(Staker1, 100) = V`
   - `tokens = getPastBalanceOf(Staker1, 100) = 100e18`
   - `votes = getPastVotes(V, 100) = 100e18`
   - Claimable amount = `(((1,000e18 * 100e18) / 100e18) * 9000) / 10000 = 900e18`.
   - Staker 1 receives `900e18` (100% of all staker rewards, 10x their rightful `90e18`), and Validator V claims `100e18` (10x their rightful `10e18`).
6. When Staker 2 claims rewards, `getPastDelegates(Staker2, 100) = W` and `_validatorRewards[W][reward.id] = 0`, so Staker 2 and Validator W receive `0`.

### Impact assessment

- **Severity**: High
- **Technical Impact**: `_validatorRewards` and `validatorPoolRewards` are miscalculated relative to the historical snapshot at `mainReward.blockNumber`. Stakers and validators can steal rewards belonging to other stakers/validators by shifting delegations between snapshot creation and reward distribution, unallocated staker rewards can become permanently stuck if `getPastVotes` was zero at the snapshot block, and new stakes after the snapshot inflate total reward liabilities beyond the contract's token balance.
- **Business Impact**: Direct loss of user funds (reward tokens), unfair forfeiture of rewards by honest stakers and validators, and potential protocol insolvency preventing legitimate users from claiming their rewards.

### Remediation

Pass `mainReward.blockNumber` (and historical reward settings at `mainReward.blockNumber`) into `_distributeAgentRewards` and `_distributeValidatorRewards`, and use historical checkpoint lookups (`getPastVotes` and `getPastValidatorScore`) aligned with `mainReward.blockNumber`. Additionally, snapshot `totalProposals` at the reward checkpoint (as implemented in `AgentRewardV3`).

```solidity
    function _distributeValidatorRewards(
        uint256 amount,
        uint256 virtualId,
        uint48 rewardId,
        uint256 totalStaked,
        uint32 timepoint
    ) private {
        IAgentNft nft = IAgentNft(agentNft);
        // Calculate weighted validator shares
        uint256 validatorCount = nft.validatorCount(virtualId);
        uint256 totalProposals = nft.totalProposals(virtualId);

        for (uint256 i = 0; i < validatorCount; i++) {
            address validator = nft.validatorAt(virtualId, i);

            // Get validator revenue by historical votes weightage at snapshot timepoint
            address stakingAddress = getVirtualTokenAddress(nft, virtualId);
            uint256 votes = IERC5805(stakingAddress).getPastVotes(validator, timepoint);
            uint256 validatorRewards = (amount * votes) / totalStaked;

            // Calc validator reward based on historical participation rate at snapshot timepoint
            uint256 score = nft.getPastValidatorScore(virtualId, validator, timepoint);
            if (score > totalProposals) {
                score = totalProposals;
            }
            uint256 participationReward = totalProposals == 0
                ? 0
                : (validatorRewards * score) / totalProposals;
            _validatorRewards[validator][rewardId] = participationReward;

            validatorPoolRewards += validatorRewards - participationReward;
        }
    }
```

## Finding 5: Zero Participation Reward When `totalProposals` Is Zero in `AgentRewardV2._distributeValidatorRewards` Diverts All Validator and Staker Rewards to Penalty Pool

### Synopsis

In `AgentRewardV2._distributeValidatorRewards`, when an agent has zero governance proposals (`totalProposals == 0`), `participationReward` is set to `0` instead of `validatorRewards`. Consequently, 100% of both validator and staker rewards are misclassified as missed-voting penalties and diverted to `validatorPoolRewards`.

### Technical details

When `distributeRewardsForAgents` distributes rewards for a `virtualId`, `_distributeAgentRewards` splits the agent's reward into `contributorAmount` and `validatorAmount` and passes `reward.validatorAmount` to `_distributeValidatorRewards`.

In `_distributeValidatorRewards` (`contracts/AgentRewardV2.sol`), each validator's group allocation (shared between the validator and the stakers delegating to that validator) is calculated proportionally to their delegated votes:

```solidity
uint256 validatorRewards = (amount * votes) / totalStaked;

// Calc validator reward based on participation rate
uint256 participationReward = totalProposals == 0
    ? 0
    : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
_validatorRewards[validator][rewardId] = participationReward;

validatorPoolRewards += validatorRewards - participationReward;
```

The purpose of scaling `validatorRewards` by `validatorScore / totalProposals` is to penalize validators who miss voting on governance proposals, accumulating the deducted penalty in `validatorPoolRewards` (`// Accumulate the penalties from missed proposal voting`, line 43).

However, when `totalProposals == 0` (no proposals have been created in the agent's DAO yet), the ternary expression evaluates `participationReward` to `0` rather than `validatorRewards`:

1. `_validatorRewards[validator][rewardId]` is recorded as `0` (line 246).
2. Because both `_getClaimableStakerRewardsAt` (lines 326–331) and `_getClaimableValidatorRewardsAt` (lines 360–362) compute claimable payouts directly as a fraction of `_validatorRewards[validator][reward.id]`, neither validators nor stakers receive any rewards for that distribution.
3. On line 248, `validatorPoolRewards += validatorRewards - participationReward` adds the entire `validatorRewards` amount to `validatorPoolRewards`, treating 100% of the reward as a missed-proposal penalty even though no proposals existed to vote on.

### Assumptions and preconditions

1. A virtual persona (agent) has been minted and has staked tokens delegated to one or more validators (`totalStaked > 0` and `votes > 0`).
2. No governance proposals have been created yet in the agent's `AgentDAO` (`nft.totalProposals(virtualId) == 0`).
3. `distributeRewardsForAgents` is called for a `mainRewardIndex` that includes this `virtualId`.

### Proof of concept

1. A new virtual agent (`virtualId = 1`) is minted via `AgentNftV2.mint`. The founder is registered as a validator and holds staked voting power (`votes = totalStaked > 0`).
2. Before any proposal is submitted to the agent's `AgentDAO`, `nft.totalProposals(1)` returns `0`.
3. Governance creates a `MainReward` and calls `AgentRewardV2.distributeRewardsForAgents(mainRewardIndex, [1])` where `reward.validatorAmount = 90,000e18`.
4. Inside `_distributeValidatorRewards`:
   - `validatorRewards` is computed as `(90,000e18 * votes) / totalStaked = 90,000e18`.
   - Because `totalProposals == 0`, line 244 sets `participationReward = 0`.
   - Line 246 sets `_validatorRewards[validator][rewardId] = 0`.
   - Line 248 increments `validatorPoolRewards` by `90,000e18 - 0 = 90,000e18`.
5. When the validator or any delegating staker calls `claimAllRewards([1], [])`, `_getClaimableValidatorRewards` and `_getClaimableStakerRewards` both return `0`, and their entire `90,000e18` reward allocation is permanently lost to `validatorPoolRewards`.

### Impact assessment

- **Severity:** High / Medium
- **Direct Technical Effect:** 100% of validator and staker rewards for any agent without governance proposals (`totalProposals == 0`) are zeroed out in `_validatorRewards` and diverted to `validatorPoolRewards`.
- **Business Implications:** Newly launched agents and agents that have not yet had contribution proposals cannot distribute any rewards to their stakers or validators, breaking the core staking incentive mechanism and unfairly penalizing participants who have not missed any votes.

### Remediation

When `totalProposals == 0`, no proposals have been missed, so `participationReward` should default to the full `validatorRewards` amount rather than `0`.

```solidity
            // Calc validator reward based on participation rate
            uint256 participationReward = totalProposals == 0
                ? validatorRewards
                : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
            _validatorRewards[validator][rewardId] = participationReward;

            validatorPoolRewards += validatorRewards - participationReward;
```

## Finding 6: Uncheckpointed NFT Ownership Check in `AgentRewardV2._claimServiceRewards` Causes Reward Entitlement Misattribution Across Transfers and NFT Contracts

### Synopsis

`AgentRewardV2._claimServiceRewards` authorizes service reward claims solely by checking the current `ownerOf(nftId)` on `contributionNft` and computing claimable rewards from cumulative per-NFT totals in `_serviceRewards[nftId]`. This causes ownership-transition entitlement inconsistencies and reward-to-NFT identity mismatches when NFTs are transferred or when `serviceNft` and `contributionNft` states diverge.

### Technical details

In `AgentRewardV2`, staker and validator rewards are accounted for per-epoch (`MainReward.blockNumber`) using historical checkpoints (`getPastBalanceOf`, `getPastDelegates`, `getPastVotes`) and per-account claim tracking (`_claimedStakerRewards[account][virtualId]` and `_claimedValidatorRewards[account][virtualId]`).

By contrast, contributor/service rewards are accumulated into a single mutable struct per `serviceId` inside `_distributeContributorRewards` (lines 252–303):
- `serviceId` is retrieved from `IAgentNft(agentNft).getAllServices(virtualId)` and its impact and core type are queried from `IServiceNft(serviceNft)`.
- Cumulative reward totals are added directly to `_serviceRewards[serviceId].amount` and `_serviceRewards[serviceId].parentAmount`.

When rewards are claimed in `_claimServiceRewards(uint256 nftId)` (lines 452–457):
```solidity
function _claimServiceRewards(uint256 nftId) internal {
    address account = _msgSender();
    require(IERC721(contributionNft).ownerOf(nftId) == account, "Not NFT owner");

    ServiceReward storage serviceReward = _serviceRewards[nftId];
    uint256 total = (serviceReward.amount - serviceReward.totalClaimed);
```

This authorization and accounting logic introduces two closely related flaws at lines 454–457:

1. **Ownership-Transition Entitlement Inconsistency**: Unlike `veVirtualToken` and `AgentToken` (which are explicitly non-transferable), `ContributionNft` is a transferable `ERC721Upgradeable` token. Because `_claimServiceRewards` only verifies `IERC721(contributionNft).ownerOf(nftId) == account` at claim time and computes `total = serviceReward.amount - serviceReward.totalClaimed` without checkpointing ownership at `mainReward.blockNumber` (or settling rewards upon NFT transfer):
   - A contributor who held `nftId` across multiple reward distribution cycles loses all unclaimed accrued rewards as soon as the NFT is transferred to a new owner, as line 454 will revert with `"Not NFT owner"`.
   - Conversely, a seller of `contributionNft` can front-run an NFT purchase transaction to drain all unclaimed `_serviceRewards[nftId]` and child `parentAmount` rewards that a buyer priced into the acquisition via `getServiceReward(nftId)`.

2. **Reward-to-NFT Identity Mismatch**: `_serviceRewards` is indexed by `serviceId` and populated using `IServiceNft(serviceNft)` parameters (`getImpact(serviceId)` and `getCore(serviceId)`), whereas `_claimServiceRewards` checks ownership on `IERC721(contributionNft)` rather than `serviceNft` (or verifying that the caller is entitled to the specific `serviceId` at the distribution block). If `contributionNft` is transferred independently of `serviceNft` or if their token IDs/owners diverge, reward claims are authorized against the wrong NFT state.

### Assumptions and preconditions

- A valid contribution has been accepted into an agent's services, minting the corresponding NFT(s), and `distributeRewards` / `distributeRewardsForAgents` has been called by `GOV_ROLE` to accrue rewards into `_serviceRewards[nftId]`.
- The `ContributionNft` (`nftId`) is transferred between two accounts (e.g., via `transferFrom` or `safeTransferFrom`) while unclaimed rewards exist in `_serviceRewards[nftId]` or its children's `parentAmount`.

### Proof of concept

1. **Setup & Accrual**:
   - Alice creates a contribution (`nftId = 1`) which is accepted as a service for `virtualId = 1`.
   - Governance calls `distributeRewards` and `distributeRewardsForAgents(0, [1])`.
   - `_serviceRewards[1].amount` is credited with `1,000` `rewardToken`. `getServiceReward(1)` shows `amount = 1000, totalClaimed = 0`.

2. **Ownership Transition**:
   - **Scenario A (Lost Contributor Entitlement)**: Alice transfers `ContributionNft(1)` to Bob without first calling `claimAllRewards([], [1])`. When Alice subsequently attempts to claim the `1,000` `rewardToken` earned during her ownership period, `_claimServiceRewards(1)` reverts at line 454 (`require(IERC721(contributionNft).ownerOf(nftId) == account, "Not NFT owner")`). Bob can now call `claimAllRewards([], [1])` and receive the `1,000` `rewardToken` accrued prior to his ownership.
   - **Scenario B (Marketplace Front-Running)**: Bob agrees to buy `ContributionNft(1)` from Alice after inspecting `getServiceReward(1)` (`1,000` unclaimed `rewardToken`). Alice front-runs Bob's purchase execution with `claimAllRewards([], [1])`, setting `serviceReward.totalClaimed = 1000` and extracting the rewards before ownership transfers to Bob.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: Unclaimed service and parent-contribution rewards are not bound to the owner at the time of reward distribution, allowing retroactive reward transfer upon NFT transfer or front-running during secondary-market sales, and decoupling `serviceNft` reward accrual from `contributionNft` ownership.
- **Business Impact**: Contributors lose earned rewards if they transfer their `ContributionNft` before claiming, and secondary transfers of `ContributionNft` tokens are vulnerable to reward extraction front-running.

### Remediation

Bind service rewards to checkpointed historical ownership at `mainReward.blockNumber` (matching the pattern used for staker rewards with `ERC721Votes` / `getPastVotes`), or automatically settle and transfer pending rewards to the current owner inside `ContributionNft._update` before ownership changes. Additionally, ensure that the NFT contract queried for ownership matches the NFT identity governing the reward.

If checkpointed per-reward accounting is used in `AgentRewardV2`, update `_claimServiceRewards` to verify historical ownership or credit the entitled account per reward epoch:

```solidity
    function getServiceReward(uint256 nftId) public view returns (ServiceReward memory) {
        return _serviceRewards[nftId];
    }

    function _claimServiceRewards(uint256 nftId) internal noReentrant {
        address account = _msgSender();
        require(
            IERC721(contributionNft).ownerOf(nftId) == account &&
                (IERC721(serviceNft).ownerOf(nftId) == account || IContributionNft(contributionNft).getChildren(nftId).length > 0),
            "Not NFT owner"
        );

        ServiceReward storage serviceReward = _serviceRewards[nftId];
        uint256 total = (serviceReward.amount - serviceReward.totalClaimed);
```

*(Note: To fully prevent ownership-transition inconsistencies across transfers, `ContributionNft._update` should also trigger reward settlement for `tokenId` and its parent token on `AgentRewardV2` prior to transferring ownership, or `_serviceRewards` should be checkpointed per `mainReward.blockNumber`.)*

## Finding 7: Pre-Registration State Contamination and Duplicate Validator Initialization in `AgentNftV2.mint`

### Synopsis

`AgentNftV2.mint` unconditionally invokes `_addValidator(virtualId, founder)` without checking if the founder was already registered via the public `addValidator` function prior to minting. This allows pre-registration state contamination of upcoming `virtualId`s and duplicate `founder` entries in `_validators[virtualId]`, corrupting reward distribution and validator score accounting.

### Technical details

In `AgentNftV2.sol`, the sequential identifier for the next agent NFT to be minted is publicly exposed via `nextVirtualId()` (lines 81–83), and pending agent applications—including the `proposer` (`founder`) address—are publicly readable in `AgentFactoryV2.getApplication`.

At the same time, `AgentNftV2.addValidator(uint256 virtualId, address validator)` (lines 133–139) is a `public` function with no check verifying that `virtualId` has already been minted (e.g., `virtualId < _nextVirtualId` or ` virtualInfos[virtualId].dao != address(0)`):

```solidity
function addValidator(uint256 virtualId, address validator) public {
    if (isValidator(virtualId, validator)) {
        return;
    }
    _addValidator(virtualId, validator);
    _initValidatorScore(virtualId, validator);
}
```

When `AgentNftV2.mint` (lines 85–114) is subsequently called during `executeApplication` to register the new persona for `virtualId == _nextVirtualId`, it directly invokes the internal `ValidatorRegistry` functions at lines 111–112 instead of checking `isValidator(virtualId, founder)`:

```solidity
_addValidator(virtualId, founder);
_initValidatorScore(virtualId, founder);
```

In `ValidatorRegistry._addValidator` (lines 30–34), the validator address is unconditionally appended to the `_validators[virtualId]` array:

```solidity
function _addValidator(uint256 virtualId, address validator) internal {
    _validatorsMap[virtualId][validator] = true;
    _validators[virtualId].push(validator);
    emit NewValidator(virtualId, validator);
}
```

If an external caller invokes `addValidator(nextVirtualId(), founder)` before `mint` is executed:
1. The validator registry state for `virtualId` is contaminated prior to registration: `_validatorsMap[virtualId][founder]` is set to `true` and `founder` is pushed onto `_validators[virtualId]`.
2. When `mint` executes, `_addValidator(virtualId, founder)` is called directly without checking `_validatorsMap[virtualId][founder]`, pushing `founder` onto `_validators[virtualId]` a second time.

Because `founder` now appears twice in `_validators[virtualId]`:
- `ValidatorRegistry.totalUptimeScore(virtualId)` (lines 60–66) sums `validatorScore(virtualId, founder)` twice, inflating the total uptime score.
- `AgentRewardV2._distributeValidatorRewards` (lines 223–250) iterates over `validatorCount(virtualId)` and processes `founder` twice. On the second iteration, `_validatorRewards[founder][rewardId]` is overwritten while `validatorPoolRewards += validatorRewards - participationReward` is incremented a second time, permanently over-accounting `validatorPoolRewards` when the founder has less than 100% participation.
- Furthermore, because `addValidator` does not verify that `virtualId` exists, arbitrary validator entries can be pre-populated for unminted `virtualId`s, bloating `_validators[virtualId]` and causing out-of-gas reverts in loops over `validatorCount(virtualId)`.

### Assumptions and preconditions

1. An application has been submitted via `proposeAgent` (or is about to be executed via `executeApplication`), making the upcoming `virtualId` (`nextVirtualId()`) and `founder` (`application.proposer`) known.
2. Any unprivileged account can call `AgentNftV2.addValidator(virtualId, founder)` before `AgentNftV2.mint` is invoked for that `virtualId`.

### Proof of concept

1. A proposer submits an agent application via `AgentFactoryV2.proposeAgent(...)`. The upcoming virtual persona ID is `virtualId = agentNft.nextVirtualId()`, and the founder address is `founder = application.proposer`.
2. Before `executeApplication` is executed, an account calls `agentNft.addValidator(virtualId, founder)`.
3. Since `isValidator(virtualId, founder)` is currently `false`, `addValidator` calls `_addValidator(virtualId, founder)`, setting `_validatorsMap[virtualId][founder] = true` and pushing `founder` to `_validators[virtualId]` at index `0`.
4. `executeApplication` is called, which invokes `agentNft.mint(virtualId, _vault, ..., founder, ...)`.
5. Inside `mint`, `_addValidator(virtualId, founder)` is called directly at line 111, pushing `founder` to `_validators[virtualId]` again at index `1`.
6. Now `agentNft.validatorCount(virtualId)` returns `2`, with both `validatorAt(virtualId, 0)` and `validatorAt(virtualId, 1)` returning `founder`.
7. When `AgentRewardV2._distributeValidatorRewards` runs for `virtualId`, the loop processes `founder` twice, adding `validatorRewards - participationReward` to `validatorPoolRewards` twice and corrupting reward accounting.

### Impact assessment

- **Severity**: Medium / High
- **Direct Technical Effects**: Duplicate `founder` entries in `_validators[virtualId]` and pre-registration pollution of validator lists for unminted `virtualId`s.
- **Business Implications**: `totalUptimeScore(virtualId)` double-counts the founder's validator score, `AgentRewardV2._distributeValidatorRewards` double-counts `validatorPoolRewards` deductions for the founder, and excessive pre-registered validators can cause denial-of-service (out-of-gas) during reward distribution.

### Remediation

In `AgentNftV2.mint`, use `addValidator(virtualId, founder)` (or check `!isValidator(virtualId, founder)`) so a validator can never be pushed twice into `_validators[virtualId]`, and require in `addValidator` that `virtualId` has already been minted (`_requireOwned(virtualId)` or `virtualId < _nextVirtualId`).

```solidity
    function mint(
        uint256 virtualId,
        address to,
        string memory newTokenURI,
        address payable theDAO,
        address founder,
        uint8[] memory coreTypes,
        address pool,
        address token
    ) external onlyRole(MINTER_ROLE) returns (uint256) {
        require(virtualId == _nextVirtualId, "Invalid virtualId");
        _nextVirtualId++;
        _mint(to, virtualId);
        _setTokenURI(virtualId, newTokenURI);
        VirtualInfo storage info = virtualInfos[virtualId];
        info.dao = theDAO;
        info.coreTypes = coreTypes;
        info.founder = founder;
        IERC5805 daoToken = GovernorVotes(theDAO).token();
        info.token = token;

        VirtualLP storage lp = virtualLPs[virtualId];
        lp.pool = pool;
        lp.veToken = address(daoToken);

        _stakingTokenToVirtualId[address(daoToken)] = virtualId;
        addValidator(virtualId, founder);
        return virtualId;
    }

    function addValidator(uint256 virtualId, address validator) public {
        _requireOwned(virtualId);
        if (isValidator(virtualId, validator)) {
            return;
        }
        _addValidator(virtualId, validator);
        _initValidatorScore(virtualId, validator);
    }
```

## Finding 8: Premature validator registration of `founder` in `AgentNftV2.mint` corrupts validator membership and baseline score accounting

### Synopsis

`AgentNftV2.mint` unconditionally registers `founder` as a validator and initializes their base validator score to zero at genesis, even though initial staked LP voting power is delegated to `defaultDelegatee`. This corrupts validator membership and permanently breaks `founder`'s historical score baseline if they receive delegation after proposals occur.

### Technical details

When a new virtual persona is bootstrapped in `AgentFactoryV3._executeApplication` (and `AgentFactoryV4._executeApplication`), the factory executes `IAgentNft(nft).mint` (step C5) and subsequently stakes the initial LP tokens on behalf of `application.proposer` (`founder`) while delegating the resulting `veToken` voting power to `defaultDelegatee` (step C7):

```solidity
IAgentVeToken(veToken).stake(IERC20(lp).balanceOf(address(this)), application.proposer, defaultDelegatee);
```

In the protocol's validator architecture (`ValidatorRegistry` and `AgentNftV2`), an account becomes a validator when it receives `veToken` delegation. At the moment of first registration, `_initValidatorScore(virtualId, validator)` records the current proposal count (`totalProposals(virtualId)`) in `_baseValidatorScore[validator][virtualId]`. In `AgentRewardV2._distributeValidatorRewards`, validator rewards are scaled by participation rate:

```solidity
uint256 participationReward = totalProposals == 0
    ? 0
    : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
```

where `validatorScore(virtualId, validator)` is `_baseValidatorScore[validator][virtualId] + _getScoreOf(virtualId, validator)`. Setting `_baseValidatorScore` to `totalProposals(virtualId)` when a validator first joins ensures that a new validator is not penalized for proposals created before they held voting power.

However, in `AgentNftV2.mint` (lines 111–112), the contract unconditionally executes:

```solidity
_addValidator(virtualId, founder);
_initValidatorScore(virtualId, founder);
```

Because `mint` is called at genesis when `totalProposals(virtualId) == 0`, this causes three accounting inconsistencies whenever `defaultDelegatee != founder`:
1. **Validator membership inconsistency:** `founder` is added to `_validators[virtualId]` and marked `isValidator(virtualId, founder) == true` despite having `0` delegated voting power (`getVotes(founder) == 0`), inflating `validatorCount(virtualId)` with an inactive address that `_distributeValidatorRewards` must iterate over on every reward distribution.
2. **Incorrect initial score attribution & historical baseline corruption:** `_baseValidatorScore[founder][virtualId]` is locked at `0` at genesis. Because `founder` has no voting power while delegated to `defaultDelegatee`, `founder` cannot vote on proposals (`_getScoreOf(virtualId, founder)` remains `0`).
3. **Permanent reward loss upon later delegation:** When `founder` later self-delegates or receives delegation after `N` proposals have already been created, `isValidator(virtualId, founder)` is already `true`. Consequently, ` founder` is not treated as a newly registered validator and `_initValidatorScore(virtualId, founder)` is not called to update `_baseValidatorScore[founder][virtualId]` to `N`. `founder`'s `validatorScore` remains `0` instead of starting at baseline `N`, causing `(validatorRewards * validatorScore) / totalProposals` to underpay `founder` and divert their share to `validatorPoolRewards`.

### Assumptions and preconditions

1. `defaultDelegatee` in `AgentFactoryV3` / `AgentFactoryV4` is configured to an address different from `application.proposer` (`founder`).
2. One or more proposals are created on the persona's `AgentDAO` while `founder` has no delegated votes.
3. `founder` (or another staker) subsequently delegates `veToken` voting power to `founder` to participate as a validator.

### Proof of concept

1. A proposer (`founder`) submits and executes an application via `AgentFactoryV3.executeApplication` where `defaultDelegatee != founder`.
2. `AgentFactoryV3._executeApplication` calls `AgentNftV2.mint(..., founder, ...)`.
3. Inside `AgentNftV2.mint`, `_addValidator(virtualId, founder)` sets `_validatorsMap[virtualId][founder] = true` and appends `founder` to `_validators[virtualId]`. `_initValidatorScore(virtualId, founder)` sets `_baseValidatorScore[founder][virtualId] = totalProposals(virtualId) = 0`.
4. `AgentFactoryV3._executeApplication` then calls `IAgentVeToken(veToken).stake(..., founder, defaultDelegatee)`, delegating 100% of `founder`'s staked LP tokens to `defaultDelegatee`. At this point, `veToken.getVotes(founder) == 0`.
5. 10 proposals are created in `AgentDAO` (`totalProposals(virtualId) == 10`). Because `founder` has 0 votes, `founder` does not cast votes on these 10 proposals, so `_getScoreOf(virtualId, founder) == 0`.
6. `founder` calls `veToken.delegate(founder)` to become an active validator. Because `isValidator(virtualId, founder)` is already `true` from step 3, `_initValidatorScore(virtualId, founder)` is not triggered during delegation, leaving `_baseValidatorScore[founder][virtualId] == 0` instead of `10`.
7. On Proposal 11, `founder` votes (`_getScoreOf(virtualId, founder) == 1`, `totalProposals(virtualId) == 11`).
8. When `AgentRewardV2._distributeValidatorRewards` runs, `nft.validatorScore(virtualId, founder)` returns `0 + 1 = 1` instead of `10 + 1 = 11`. `founder` only receives `1 / 11` of their `validatorRewards`, losing `10 / 11` (~90.9%) of their entitled validator rewards to `validatorPoolRewards`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Corrupts validator set state (`_validators` and `_validatorsMap`) from genesis and causes `_baseValidatorScore[founder][virtualId]` to remain `0` instead of the proposal count at the time `founder` actually receives delegation.
- **Business Impact:** Direct loss of validator participation rewards for any agent founder who delegates to themselves (or receives delegation) after proposals have been created, as well as unnecessary gas overhead iterating over an inactive validator entry during reward distributions.

### Remediation

Remove `_addValidator(virtualId, founder)` and `_initValidatorScore(virtualId, founder)` from `AgentNftV2.mint`, allowing validator registration and base score initialization to occur naturally when `veToken` voting power is delegated (such as during the subsequent `veToken.stake` call in the factory).

```solidity
    function mint(
        uint256 virtualId,
        address to,
        string memory newTokenURI,
        address payable theDAO,
        address founder,
        uint8[] memory coreTypes,
        address pool,
        address token
    ) external onlyRole(MINTER_ROLE) returns (uint256) {
        require(virtualId == _nextVirtualId, "Invalid virtualId");
        _nextVirtualId++;
        _mint(to, virtualId);
        _setTokenURI(virtualId, newTokenURI);
        VirtualInfo storage info = virtualInfos[virtualId];
        info.dao = theDAO;
        info.coreTypes = coreTypes;
        info.founder = founder;
        IERC5805 daoToken = GovernorVotes(theDAO).token();
        info.token = token;

        VirtualLP storage lp = virtualLPs[virtualId];
        lp.pool = pool;
        lp.veToken = address(daoToken);

        _stakingTokenToVirtualId[address(daoToken)] = virtualId;
        return virtualId;
    }
```

## Finding 9: Non-Checkpointed Base Score in `ValidatorRegistry.getPastValidatorScore` Contaminates Historical Validator Scores with Current State

### Synopsis

In `ValidatorRegistry`, `_baseValidatorScore` is stored as a single mutable `uint256` rather than a time-checkpointed trace. Consequently, `getPastValidatorScore` combines a historical DAO score at `timepoint` with the latest `_baseValidatorScore`, causing temporal misalignment and allowing post-snapshot validator enrollment to retroactively inflate historical validator scores.

### Technical details

`ValidatorRegistry` tracks a validator's score as the sum of an initial baseline score (`_baseValidatorScore`) recorded when the validator is registered and the validator's earned proposal participation score in `AgentDAO`.

In `ValidatorRegistry.sol`:
```solidity
function _initValidatorScore(uint256 virtualId, address validator) internal {
    _baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId);
}
```
And for historical queries:
```solidity
function getPastValidatorScore(
    uint256 virtualId,
    address validator,
    uint256 timepoint
) public view virtual returns (uint256) {
    return _baseValidatorScore[validator][virtualId] + _getPastScore(virtualId, validator, timepoint);
}
```

Here, `_getPastScore(virtualId, validator, timepoint)` delegates to `IAgentDAO(info.dao).getPastScore(account, timepoint)`, which returns the checkpointed score of the validator at the historical `timepoint`. However, `_baseValidatorScore[validator][virtualId]` is not checkpointed by block number/timepoint; it simply holds the value of `_getMaxScore(virtualId)` (`totalProposals(virtualId)`) at the moment `_initValidatorScore` was executed.

This creates a temporal misalignment between the two score components in `getPastValidatorScore`:
1. **Retroactive contamination of past timepoints:** If an account is not a validator at `timepoint` $T_1$ (when `totalProposals` is $P_1$), its base score at $T_1$ is `0`. If the account is later added as a validator at $T_2 > T_1$ (when `totalProposals` has grown to $P_2 > P_1$), `_initValidatorScore` sets `_baseValidatorScore[validator][virtualId] = P_2`. Any subsequent call to `getPastValidatorScore(virtualId, validator, T_1)` will return $P_2 + \text{getPastScore}(T_1)$, retroactively applying a future baseline $P_2$ to the past timepoint $T_1$—even exceeding the total number of proposals $P_1$ that existed at $T_1$.
2. **Permissionless manipulation:** Because `AgentNftV2.addValidator(uint256 virtualId, address validator)` is `public` and unrestricted, any caller can trigger `_initValidatorScore` for any non-validator address at a later block and immediately alter the value returned by `getPastValidatorScore` for all prior timepoints.

### Assumptions and preconditions

1. A virtual persona (`virtualId`) has been minted and has an associated `AgentDAO` where proposals are created over time.
2. `getPastValidatorScore(virtualId, validator, timepoint)` is queried for a historical `timepoint` $T_1$.
3. The `validator` address is registered via `addValidator` (or `stake`) at a timepoint $T_2 > T_1$ after additional proposals have been created in `AgentDAO`.

### Proof of concept

1. At block 100 ($T_1$), `virtualId = 1` has `totalProposals(1) == 2`. Account `alice` is not yet a registered validator (`_baseValidatorScore[alice][1] == 0`) and has `dao.getPastScore(alice, 100) == 0`.
2. Calling `getPastValidatorScore(1, alice, 100)` at block 101 returns `0 + 0 = 0`.
3. Between blocks 101 and 200, 8 new proposals are created in `AgentDAO`, so at block 200 ($T_2$), `totalProposals(1) == 10`.
4. At block 200, `alice` (or any caller) invokes `AgentNftV2.addValidator(1, alice)`.
5. `_initValidatorScore(1, alice)` sets `_baseValidatorScore[alice][1] = _getMaxScore(1) = 10`.
6. Calling `getPastValidatorScore(1, alice, 100)` for the historical block 100 now returns `_baseValidatorScore[alice][1] + dao.getPastScore(alice, 100) = 10 + 0 = 10`, even though `alice` was not a validator at block 100 and only 2 proposals existed at block 100.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** `getPastValidatorScore` violates historical state integrity. Historical queries for a given `timepoint` are not immutable and are contaminated by subsequent state transitions when `_initValidatorScore` is called at a later block.
- **Business Impact:** Any governance, reward calculation, or off-chain/on-chain accounting mechanism relying on `IValidatorRegistry.getPastValidatorScore` receives corrupted, inflated historical participation scores that can exceed the total proposals that existed at the queried timepoint.

### Remediation

Store `_baseValidatorScore` using OpenZeppelin's `Checkpoints.Trace208` keyed by the clock/block number so that `getPastValidatorScore` looks up the base validator score that was active at `timepoint` via `upperLookupRecent`.

```solidity
    using Checkpoints for Checkpoints.Trace208;

    mapping(uint256 virtualId => mapping(address account => bool isValidator)) private _validatorsMap;
    mapping(address account => mapping(uint256 virtualId => Checkpoints.Trace208 scoreCheckpoints)) private _baseValidatorScore;
    mapping(uint256 virtualId => address[] validators) private _validators;

    function _initValidatorScore(uint256 virtualId, address validator) internal {
        _baseValidatorScore[validator][virtualId].push(
            SafeCast.toUint48(block.number),
            SafeCast.toUint208(_getMaxScore(virtualId))
        );
    }

    function validatorScore(uint256 virtualId, address validator) public view virtual returns (uint256) {
        return _baseValidatorScore[validator][virtualId].latest() + _getScoreOf(virtualId, validator);
    }

    function getPastValidatorScore(
        uint256 virtualId,
        address validator,
        uint256 timepoint
    ) public view virtual returns (uint256) {
        if (timepoint >= block.number) {
            revert ERC5805FutureLookup(timepoint, SafeCast.toUint48(block.number));
        }
        return
            _baseValidatorScore[validator][virtualId].upperLookupRecent(SafeCast.toUint48(timepoint)) +
            _getPastScore(virtualId, validator, timepoint);
    }
```

## Finding 10: Missing Uniqueness Check in `ValidatorRegistry._addValidator` Allows Duplicate Validator Registration

### Synopsis

`ValidatorRegistry._addValidator` appends a validator address to `_validators[virtualId]` without checking if `_validatorsMap[virtualId][validator]` is already true. An unprivileged caller can pre-register a persona's `founder` via `AgentNftV2.addValidator` before `mint`, causing `mint` to push a duplicate `founder` entry and corrupt validator reward accounting.

### Technical details

In `contracts/virtualPersona/ValidatorRegistry.sol`, `_addValidator` (lines 30–34) updates the validator registry state as follows:

```solidity
function _addValidator(uint256 virtualId, address validator) internal {
    _validatorsMap[virtualId][validator] = true;
    _validators[virtualId].push(validator);
    emit NewValidator(virtualId, validator);
}
```

Unlike `AgentNftV2.addValidator` (lines 133–139), which checks `if (isValidator(virtualId, validator)) return;` before invoking `_addValidator`, `_addValidator` itself does not verify whether `_validatorsMap[virtualId][validator]` is already `true` before pushing `validator` onto the `_validators[virtualId]` array.

In `AgentNftV2`:
1. `addValidator(uint256 virtualId, address validator)` is `public` with no access control and no check that `virtualId` has already been minted (`virtualId < _nextVirtualId`).
2. `mint(uint256 virtualId, ..., address founder, ...)` (lines 85–114) directly invokes `_addValidator(virtualId, founder)` at line 111 without checking `isValidator(virtualId, founder)`.

Consequently, if any third party calls `addValidator(nextVirtualId, founder)` before `mint` is executed for `nextVirtualId`:
- `addValidator` calls `_addValidator(nextVirtualId, founder)`, which sets `_validatorsMap[nextVirtualId][founder] = true` and appends `founder` to `_validators[nextVirtualId]` at index `0`.
- When `MINTER_ROLE` subsequently calls `mint(nextVirtualId, ..., founder, ...)`, `mint` calls `_addValidator(nextVirtualId, founder)` directly.
- Because `_addValidator` does not check `_validatorsMap[nextVirtualId][founder]`, it appends `founder` to `_validators[nextVirtualId]` a second time at index `1`.

This violates the uniqueness invariant of `_validators[virtualId]`. Downstream functions that iterate over `validatorCount(virtualId)` and `validatorAt(virtualId, i)`—specifically `ValidatorRegistry.totalUptimeScore` (lines 60–66) and `AgentRewardV2._distributeValidatorRewards` (lines 234–249)—process the `founder` address twice. In `AgentRewardV2._distributeValidatorRewards`, every iteration where `participationReward < validatorRewards` adds `validatorRewards - participationReward` to `validatorPoolRewards` (line 248), double-counting the founder's unallocated reward share into `validatorPoolRewards`.

### Assumptions and preconditions

1. A new virtual persona is about to be minted with sequential ID `virtualId = nextVirtualId()` and a known `founder` address (either observed in the mempool or known beforehand).
2. A third party calls `AgentNftV2.addValidator(virtualId, founder)` before `AgentNftV2.mint(virtualId, ..., founder, ...)` is executed.

### Proof of concept

1. Query `AgentNftV2.nextVirtualId()`, which returns `1`.
2. Before `mint` is called for `virtualId = 1` with `founder = 0xFounder`, an external caller invokes `AgentNftV2.addValidator(1, 0xFounder)`.
   - `isValidator(1, 0xFounder)` is `false`.
   - `_addValidator(1, 0xFounder)` sets `_validatorsMap[1][0xFounder] = true` and pushes `0xFounder` to `_validators[1]` (`_validators[1] = [0xFounder]`).
3. The authorized minter calls `AgentNftV2.mint(1, to, uri, theDAO, 0xFounder, coreTypes, pool, token)`.
   - Line 111 executes `_addValidator(1, 0xFounder)`.
   - Because `_addValidator` does not check `_validatorsMap[1][0xFounder]`, it pushes `0xFounder` to `_validators[1]` again (`_validators[1] = [0xFounder, 0xFounder]`).
4. `validatorCount(1)` now returns `2`, and both `validatorAt(1, 0)` and `validatorAt(1, 1)` return `0xFounder`.
5. When `AgentRewardV2._distributeValidatorRewards` runs for `virtualId = 1` where `0xFounder` has staked voting power and `participationReward < validatorRewards`, the loop processes `0xFounder` twice and increments `validatorPoolRewards` twice by `validatorRewards - participationReward`, inflating `validatorPoolRewards` beyond the actual tokens available.

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: Duplicate validator entries in `_validators[virtualId]` inflate `validatorCount(virtualId)` and `totalUptimeScore(virtualId)`, and cause `AgentRewardV2._distributeValidatorRewards` to accumulate `validatorRewards - participationReward` into `validatorPoolRewards` multiple times for the same validator.
- **Business Impact**: Reward accounting in `AgentRewardV2` becomes inconsistent with the actual reward token balance held by the contract, which can cause subsequent reward claims from `validatorPoolRewards` to fail due to insufficient balance.

### Remediation

Enforce validator uniqueness directly inside `ValidatorRegistry._addValidator` so that an address already marked in `_validatorsMap[virtualId]` cannot be pushed to `_validators[virtualId]` more than once.

```solidity
    function _addValidator(uint256 virtualId, address validator) internal {
        if (_validatorsMap[virtualId][validator]) {
            return;
        }
        _validatorsMap[virtualId][validator] = true;
        _validators[virtualId].push(validator);
        emit NewValidator(virtualId, validator);
    }
```

## Finding 11: Unadjusted Base Score Initialization in `ValidatorRegistry._initValidatorScore` Allows `validatorScore` to Exceed `totalProposals` and Bricks Reward Distribution

### Synopsis

In `ValidatorRegistry._initValidatorScore`, a newly registered validator's `_baseValidatorScore` is set to `_getMaxScore(virtualId)` (`totalProposals`) without accounting for existing `dao.scoreOf(validator)` or active proposals. This allows `validatorScore` to exceed `totalProposals`, inflating participation rewards or causing arithmetic underflow in `AgentRewardV2._distributeValidatorRewards`.

### Technical details

In `ValidatorRegistry.sol`, validator participation is tracked using a baseline score initialized upon validator registration and a cumulative voting score queried from `AgentDAO`:

```solidity
function _initValidatorScore(uint256 virtualId, address validator) internal {
    _baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId);
}

function validatorScore(uint256 virtualId, address validator) public view virtual returns (uint256) {
    return _baseValidatorScore[validator][virtualId] + _getScoreOf(virtualId, validator);
}
```

In `AgentNftV2.sol`, `__ValidatorRegistry_init` binds `_getScoreOf` to `_validatorScoreOf` (`IAgentDAO(info.dao).scoreOf(account)`) and `_getMaxScore` to `totalProposals` (`IAgentDAO(info.dao).proposalCount()`).

The reward calculation in `AgentRewardV2._distributeValidatorRewards` relies on the invariant that `nft.validatorScore(virtualId, validator) <= totalProposals`:

```solidity
uint256 participationReward = totalProposals == 0
    ? 0
    : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
_validatorRewards[validator][rewardId] = participationReward;

validatorPoolRewards += validatorRewards - participationReward;
```

However, `_initValidatorScore` violates this invariant in two ways:

1. **Pre-registration voting via delegation:** Voting in `AgentDAO` only requires delegated voting power on the `veToken` (`IERC5805`), not prior registration in `ValidatorRegistry`. A user who receives delegated votes via `veToken.delegate(validator)` (rather than staking directly) is not automatically registered as a validator. They can vote on proposals and accumulate `_getScoreOf(virtualId, validator) > 0`. When `addValidator(virtualId, validator)` is subsequently called (either directly via the permissionless `AgentNftV2.addValidator` or via staking), `_initValidatorScore` sets `_baseValidatorScore[validator][virtualId] = totalProposals`. `validatorScore` then evaluates to `totalProposals + _getScoreOf(virtualId, validator) > totalProposals`. Additionally, a delegate who missed some past proposals and voted on others can call `addValidator` later to reset their baseline to `totalProposals` while keeping their prior votes, erasing their missed-proposal penalty.
2. **Registration during an active proposal:** `dao.proposalCount()` increments immediately when `propose()` is called. If an account with snapshot voting power registers via `addValidator` after a proposal is created and then casts a vote on that proposal, `_baseValidatorScore` already includes that proposal in `totalProposals`, and casting the vote increments `_getScoreOf` by 1 without increasing `totalProposals`, again resulting in `validatorScore > totalProposals`.

Once `validatorScore(virtualId, validator) > totalProposals` for any validator with non-zero `validatorRewards`, `participationReward` exceeds `validatorRewards`, and `validatorRewards - participationReward` at `AgentRewardV2.sol:248` reverts with a Solidity 0.8 arithmetic underflow panic (`0x11`).

### Assumptions and preconditions

1. A virtual persona (`virtualId`) has been minted and has an associated `AgentDAO` and `veToken`.
2. An account holds delegated voting power (e.g., via `veToken.delegate(account)`) without having `addValidator(virtualId, account)` called yet, or registers via `addValidator` after a proposal is created.
3. At least one proposal is created and voted on by the account.

### Proof of concept

1. `founder` (or any staker) delegates voting power on `veToken` to `validatorA` via `veToken.delegate(validatorA)`. Note that `delegate()` does not call `agentNft.addValidator(virtualId, validatorA)`.
2. A proposal is created in `AgentDAO`. `totalProposals(virtualId)` is now `1`.
3. `validatorA` casts a valid vote on the proposal in `AgentDAO`. `dao.scoreOf(validatorA)` (`_getScoreOf(virtualId, validatorA)`) becomes `1`.
4. Anyone calls the public function `agentNft.addValidator(virtualId, validatorA)`.
5. `_initValidatorScore(virtualId, validatorA)` sets `_baseValidatorScore[validatorA][virtualId] = _getMaxScore(virtualId) = 1`.
6. `agentNft.validatorScore(virtualId, validatorA)` now returns `_baseValidatorScore (1) + _getScoreOf (1) = 2`, while `totalProposals(virtualId)` is `1`.
7. When the protocol backend calls `AgentRewardV2.distributeRewards`, `_distributeValidatorRewards` computes:
   - `validatorRewards = (amount * votes) / totalStaked` (non-zero since `validatorA` has delegated votes).
   - `participationReward = (validatorRewards * 2) / 1 = 2 * validatorRewards`.
   - `validatorPoolRewards += validatorRewards - participationReward` reverts with arithmetic underflow (`Panic(0x11)`), permanently halting reward distribution.

### Impact assessment

- **Severity:** High
- **Technical Impact:** `validatorScore` exceeds `totalProposals`, violating the participation rate upper bound of 100%. In `AgentRewardV2._distributeValidatorRewards`, this causes an arithmetic underflow revert when subtracting `participationReward` from `validatorRewards`, resulting in a permanent Denial of Service (DoS) of reward distribution for the affected `virtualId`. Even if `validatorScore` remains `<= totalProposals` (e.g., when a delegate missed earlier proposals before calling `addValidator`), the validator can erase past non-participation penalties and claim a higher share of rewards than earned.

### Remediation

In `ValidatorRegistry.sol`, subtract the validator's already-accumulated score `_getScoreOf(virtualId, validator)` when initializing `_baseValidatorScore` in `_initValidatorScore`, and cap `validatorScore` (and `getPastValidatorScore`) at `_getMaxScore(virtualId)` to ensure the score can never exceed the total number of proposals:

```solidity
    function _initValidatorScore(uint256 virtualId, address validator) internal {
        uint256 maxScore = _getMaxScore(virtualId);
        uint256 currentScore = _getScoreOf(virtualId, validator);
        _baseValidatorScore[validator][virtualId] = maxScore > currentScore ? maxScore - currentScore : 0;
    }

    function validatorScore(uint256 virtualId, address validator) public view virtual returns (uint256) {
        uint256 score = _baseValidatorScore[validator][virtualId] + _getScoreOf(virtualId, validator);
        uint256 maxScore = _getMaxScore(virtualId);
        return score > maxScore ? maxScore : score;
    }
```

## Finding 12: Unbounded Validator Set Enumeration via `ValidatorRegistry.validatorCount` Enables DoS of Reward Distribution

### Synopsis

`ValidatorRegistry` stores validators in an append-only array `_validators[virtualId]` and exposes its unbounded length via `validatorCount(virtualId)`. Because `AgentNftV2.addValidator` is permissionless, an attacker can register an arbitrary number of validator addresses for any `virtualId`, causing `AgentRewardV2._distributeValidatorRewards` and `ValidatorRegistry.totalUptimeScore` to revert with out-of-gas.

### Technical details

In `contracts/virtualPersona/ValidatorRegistry.sol`, each registered validator for a given `virtualId` is appended to `_validators[virtualId]` via `_addValidator`, and the registry exposes the total number of validators through `validatorCount(uint256 virtualId)` (lines 52–53):

```solidity
function validatorCount(uint256 virtualId) public view returns (uint256) {
    return _validators[virtualId].length;
}
```

There is no upper bound on `_validators[virtualId].length`, no mechanism to remove inactive or zero-stake validators from `_validators[virtualId]`, and no access control on `AgentNftV2.addValidator(uint256 virtualId, address validator)` (`contracts/virtualPersona/AgentNftV2.sol`, lines 133–139):

```solidity
function addValidator(uint256 virtualId, address validator) public {
    if (isValidator(virtualId, validator)) {
        return;
    }
    _addValidator(virtualId, validator);
    _initValidatorScore(virtualId, validator);
}
```

Downstream consumers rely on `validatorCount(virtualId)` to bound loops that execute expensive storage reads and external calls for every registered validator:

1. **`AgentRewardV2._distributeValidatorRewards` (`contracts/AgentRewardV2.sol`, lines 231–249):**
   Queries `uint256 validatorCount = nft.validatorCount(virtualId);` and iterates `for (uint256 i = 0; i < validatorCount; i++)`. On each iteration, it performs multiple external calls (`nft.validatorAt`, `nft.virtualInfo`, `IERC5805(stakingAddress).getVotes`, `nft.validatorScore`) and storage writes (`_validatorRewards`, `validatorPoolRewards`).
2. **`ValidatorRegistry.totalUptimeScore` (`contracts/virtualPersona/ValidatorRegistry.sol`, lines 60–66):**
   Iterates `for (uint256 i = 0; i < validatorCount(virtualId); i++)`, calling `validatorAt` and `validatorScore` (which performs an external call to `IAgentDAO(info.dao).scoreOf`) on each iteration.

Because anyone can call `addValidator(virtualId, validator)` with arbitrary distinct `validator` addresses (even with zero stake or voting power), `_validators[virtualId].length` can be inflated until the loops bounded by `validatorCount(virtualId)` exceed the block gas limit.

### Assumptions and preconditions

1. At least one valid virtual persona (`virtualId`) has been minted in `AgentNftV2`.
2. An attacker has a funded address to pay transaction gas fees to invoke `AgentNftV2.addValidator(virtualId, address)` repeatedly (or in batches via a helper contract) with distinct addresses.

### Proof of concept

1. A valid agent with `virtualId = 1` is active and accruing rewards to be distributed via `AgentRewardV2`.
2. An unprivileged caller invokes `AgentNftV2.addValidator(1, addr_i)` for thousands of distinct addresses `addr_1, addr_2, ..., addr_N` that hold zero voting power.
3. Each call passes the `!isValidator(1, addr_i)` check, appends `addr_i` to `_validators[1]`, and increments `validatorCount(1)`.
4. When the protocol backend calls `AgentRewardV2.distributeRewards` (which invokes `_distributeAgentRewards` -> `_distributeValidatorRewards`), the loop `for (uint256 i = 0; i < validatorCount; i++)` executes multiple external calls for all `N` addresses.
5. The total gas consumed by the loop exceeds the block gas limit, causing `_distributeValidatorRewards` and `distributeRewards` to revert with out-of-gas permanently for `virtualId = 1`.

### Impact assessment

- **Severity:** High / Medium (Permanent Denial of Service of core protocol functionality).
- **Direct Technical Effect:** `AgentRewardV2._distributeValidatorRewards` and `ValidatorRegistry.totalUptimeScore` unconditionally revert due to out-of-gas once `validatorCount(virtualId)` is sufficiently large.
- **Business Implications:** Reward distribution for targeted virtual personas (and any batch `distributeRewards` call that includes an affected `virtualId`) is permanently blocked, freezing staker, validator, and contributor rewards with no on-chain recovery path to prune `_validators[virtualId]`.

### Remediation

Restrict validator registration to authorized callers (such as `VALIDATOR_ADMIN_ROLE`, the staking token contract when a minimum stake threshold is met, or the Virtual DAO), enforce a maximum active validator count per `virtualId` in `ValidatorRegistry`, or refactor reward distribution to use a pull-based or paginated accounting mechanism rather than iterating over `validatorCount(virtualId)`.

```solidity
    uint256 public constant MAX_VALIDATORS = 100;

    function _addValidator(uint256 virtualId, address validator) internal {
        require(_validators[virtualId].length < MAX_VALIDATORS, "Max validators reached");
        _validatorsMap[virtualId][validator] = true;
        _validators[virtualId].push(validator);
        emit NewValidator(virtualId, validator);
    }

    function validatorCount(uint256 virtualId) public view returns (uint256) {
        return _validators[virtualId].length;
    }
```

Additionally, restrict `AgentNftV2.addValidator` so that arbitrary accounts cannot register zero-stake validators permissionlessly:

```solidity
    function addValidator(uint256 virtualId, address validator) public onlyRole(VALIDATOR_ADMIN_ROLE) {
        if (isValidator(virtualId, validator)) {
            return;
        }
        _addValidator(virtualId, validator);
        _initValidatorScore(virtualId, validator);
    }
```
