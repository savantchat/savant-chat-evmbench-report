# Audit report — 2025-04-virtuals/H-03

## Finding 1: Unbounded Service Iteration and External Calls in `AgentRewardV2._distributeContributorRewards` Cause Reward Distribution DoS

### Synopsis

`AgentRewardV2._distributeContributorRewards` iterates twice over the unbounded array of all services belonging to a virtual agent without pagination, performing multiple external calls and storage updates per service. As services accumulate over time, gas consumption exceeds the block gas limit, permanently blocking reward distribution for the agent.

### Technical details

When rewards are distributed to a virtual agent via `distributeRewardsForAgents` -> `_distributeAgentRewards` -> `_distributeContributorRewards`, the contract retrieves the full list of services associated with `virtualId` by calling `nft.getAllServices(virtualId)` (line 264). In `AgentNftV2.getAllServices`, this enumerates every `ServiceNft` held by the agent's Token Bound Account (`info.tba`). Because new `ServiceNft` tokens are minted to the agent's TBA over the lifetime of the virtual persona and are never removed, the `services` array grows monotonically without an upper bound.

In `_distributeContributorRewards`, the contract processes this unbounded `services` array across two consecutive loops:

1. **Impact aggregation loop (lines 269–281):** For every `serviceId` in `services`, the function makes an external call `serviceNftContract.getImpact(serviceId)` (line 271), conditionally initializes `_serviceRewards[serviceId].impact` in storage (lines 276–279), makes a second external call `serviceNftContract.getCore(serviceId)` (line 280), and writes to the `_rewardImpacts[reward.id][core]` storage slot (line 280).
2. **Reward allocation loop (lines 287–302):** The function iterates over the entire `services` array a second time (line 287), again calling `serviceNftContract.getCore(serviceId)` (line 295) and `contributionNftContract.getParentId(serviceId)` (line 296), and updating `serviceReward.amount` and `serviceReward.parentAmount` in storage (lines 300–301) for every service with non-zero impact.

Because `_distributeAgentRewards` must complete atomically for a given `virtualId` (enforced by the double-entry check at lines 200–202) and provides no mechanism to paginate across services, once an agent accumulates enough services, the per-item external call and storage overhead causes `_distributeContributorRewards` to exceed the block gas limit. Furthermore, if `virtualId` is included in a batch call to `distributeRewardsForAgents`, the revert aborts the entire batch transaction.

### Assumptions and preconditions

1. A virtual agent (`virtualId`) is active and receives service contributions (`ServiceNft` tokens minted to its TBA) over time, or any ERC-721 `ServiceNft` tokens are transferred to `info.tba`.
2. The number of services owned by the agent's TBA grows large enough that executing `getAllServices` and the two loops in `_distributeContributorRewards` exceeds the network block gas limit.

### Proof of concept

1. A virtual agent (`virtualId = 1`) is created with its Token Bound Account (`info.tba`), and stakers stake tokens above `settings.stakeThreshold`.
2. Over time, governance proposals for contributions are approved (or `ServiceNft` tokens are transferred to `info.tba`), resulting in hundreds or thousands of `ServiceNft` tokens owned by `info.tba`.
3. Governance calls `distributeRewards(amount)` to create a new `MainReward` checkpoint, which records a `Reward` entry for `virtualId = 1`.
4. Governance calls `distributeRewardsForAgents(mainRewardIndex, [1])`.
5. `_distributeAgentRewards` invokes `_distributeContributorRewards`, which calls `nft.getAllServices(1)` and then executes the two loops at lines 269–281 and 287–302.
6. For each service, the contract executes external calls (`serviceNftContract.getImpact`, `serviceNftContract.getCore` twice, `contributionNftContract.getParentId`) and multiple `SSTORE`/`SLOAD` operations.
7. The total gas required exceeds the block gas limit, causing `distributeRewardsForAgents` to revert with an out-of-gas error and permanently locking contributor and validator rewards for `virtualId = 1`.

### Impact assessment

- **Severity:** Medium / High (Availability / Permanent Locking of Rewards)
- **Technical Impact:** `_distributeContributorRewards` and `_distributeAgentRewards` become uncallable for any virtual agent whose service count exceeds the gas limit threshold. Any batch call to `distributeRewardsForAgents` that includes the affected `virtualId` will also revert atomically.
- **Business Impact:** Reward tokens already transferred into `AgentRewardV2` during `distributeRewards` for the affected agent's stake share cannot be distributed to or claimed by its validators, stakers, or contributors, resulting in permanently frozen reward funds for the most active virtual agents.

### Remediation

Refactor the reward accounting to use a pull-based (checkpoint/index-based) model rather than pushing rewards to every service on each distribution, or maintain running cumulative core impact totals in storage when services are minted/updated so `_distributeContributorRewards` does not need to iterate over all services.

If push-based distribution must be retained, cache cumulative core impacts in `ServiceNft` and paginate service reward distribution across bounded batches:

```solidity
        // Cache coreTypes impacts in ServiceNft and distribute in bounded pages, or
        // compute per-service rewards lazily during _claimServiceRewards using checkpoints:
        for (uint256 i = 0; i < coreTypes.length; i++) {
            uint8 coreType = coreTypes[i];
            uint256 totalCoreImpact = serviceNftContract.getTotalCoreImpact(virtualId, coreType);
            if (totalCoreImpact > 0) {
                _rewardImpacts[reward.id][coreType] = totalCoreImpact;
            }
        }
```

## Finding 2: Mismatched Cached and Live Service Impact Weights in `AgentRewardV2._distributeContributorRewards` Corrupts Reward Distribution

### Synopsis

In `AgentRewardV2._distributeContributorRewards`, the total core impact denominator `_rewardImpacts` is calculated using each service's live impact from `ServiceNft.getImpact`, whereas individual service rewards are filtered and calculated using the cached `_serviceRewards[serviceId].impact` value, causing reward over-allocation, under-allocation, or division-by-zero reverts when impacts change.

### Technical details

In `contracts/AgentRewardV2.sol`, `_distributeContributorRewards` distributes contributor rewards across all services associated with a `virtualId` in two passes over the `services` array:

1. **First Pass (Lines 269–281):** For each `serviceId`, the contract queries the live impact weight via `impact = serviceNftContract.getImpact(serviceId)`. If `impact > 0`, it initializes `serviceReward.impact = impact` **only if** `serviceReward.impact == 0` (lines 277–279), and then adds the live `impact` value to the per-reward, per-core accumulator `_rewardImpacts[reward.id][serviceNftContract.getCore(serviceId)]` (line 280).

   ```solidity
   impactAmount =
       (reward.coreAmount * serviceReward.impact) /
       _rewardImpacts[reward.id][serviceNftContract.getCore(serviceId)];
   ```

Because `ServiceNft` allows a service's impact weight to be updated after minting (as evidenced by `IServiceNft.SetServiceScore(uint256 serviceId, uint256 eloRating, uint256 impact)` and `IServiceNft.DatasetImpactUpdated(uint16 weight)`), the live `serviceNftContract.getImpact(serviceId)` can diverge from the cached `serviceReward.impact` stored during the service's first reward distribution:

- **Impact Decrease:** If a service's live impact decreases, `_rewardImpacts[reward.id][core]` reflects the smaller live impact, while the numerator at line 294 uses the larger cached `serviceReward.impact`. Consequently, the sum of `impactAmount` across services for that core exceeds `reward.coreAmount`, allocating more tokens to `_serviceRewards` than were funded for the contributor pool.
- **Impact Reduced to Zero:** If a service's live impact becomes `0`, the first pass skips it (`if (impact == 0) continue;`), contributing `0` to `_rewardImpacts[reward.id][core]`. However, in the second pass, `serviceReward.impact` is still non-zero, so line 290 does not skip the service. If other services on the same core have positive live impact, this zero-impact service still receives rewards and inflates total allocations beyond `reward.coreAmount`. If no service on that core has a positive live impact, `_rewardImpacts[reward.id][core]` is `0`, and line 295 reverts with a division-by-zero panic, blocking `distributeRewardsForAgents`.
- **Impact Increase:** If a service's live impact increases, `_rewardImpacts[reward.id][core]` increases in the denominator while `serviceReward.impact` in the numerator remains at its lower cached value, resulting in under-distribution and permanently stranded reward tokens in `AgentRewardV2`.

### Assumptions and preconditions

1. At least one reward distribution has occurred for a `virtualId` where a service (`serviceId`) had a non-zero impact (`I_1 > 0`), caching `serviceReward.impact = I_1`.
2. Prior to a subsequent reward distribution for the same `virtualId`, the impact of `serviceId` on `ServiceNft` is updated to a different value `I_2 != I_1` (e.g., via score or dataset weight updates).
3. Governance calls `distributeRewardsForAgents` for a new `mainRewardIndex` including `virtualId`.

### Proof of concept

1. Suppose `virtualId = 1` has a single core type (`core = 1`) and two services (`serviceId = 10` and `serviceId = 11`) both belonging to `core = 1`, each initially having a live impact of `100` on `ServiceNft`.
2. During Reward Period 1 (`reward.id = 1`), `distributeRewardsForAgents` is called:
   - In the first loop, `_serviceRewards[10].impact` is set to `100`, `_serviceRewards[11].impact` is set to `100`, and `_rewardImpacts[1][1] = 200`.
   - In the second loop, each service legitimately receives `50%` of `reward.coreAmount`.
3. Before Reward Period 2 (`reward.id = 2`), the live impact of `serviceId = 10` is updated on `ServiceNft` from `100` to `20`, while `serviceId = 11` remains at `100`.
4. During Reward Period 2 (`reward.id = 2`, with `reward.coreAmount = 1200`), `distributeRewardsForAgents` is called:
   - In the first loop, `_rewardImpacts[2][1]` is computed from live impacts: `20 + 100 = 120`. Because `_serviceRewards[10].impact` is already `100 != 0`, it is not updated and remains `100`.
   - In the second loop (lines 288–300):
     - For `serviceId = 10`: `impactAmount = (1200 * 100) / 120 = 1000`.
     - For `serviceId = 11`: `impactAmount = (1200 * 100) / 120 = 1000`.
   - Total rewards credited to services for this core equal `2000`, exceeding the allocated `reward.coreAmount` of `1200` by `800` tokens.
5. Alternatively, if both services have their live impacts updated to `0` before Reward Period 2, the first loop skips both services leaving `_rewardImpacts[2][1] = 0`, and line 295 in the second loop executes `(reward.coreAmount * 100) / 0`, reverting the entire `distributeRewardsForAgents` transaction with a division-by-zero panic.

### Impact assessment

- **Severity:** High
- **Technical Impact:**
  - **Over-allocation / Pool Insolvency:** When service impacts decrease, contributors are credited with more rewards than allocated to `reward.coreAmount`. Claiming these inflated service rewards drains `rewardToken` balances reserved for other stakers, validators, or contributors.
  - **Under-allocation / Locked Funds:** When service impacts increase, the sum of credited `impactAmount` values is strictly less than `reward.coreAmount`, permanently locking the unallocated remainder in `AgentRewardV2`.
  - **Denial of Service:** If all services for a given core have their live impacts reduced to `0` after previously having non-zero impacts, `_rewardImpacts[reward.id][core]` remains `0` while `serviceReward.impact > 0`, causing an unrecoverable division-by-zero revert in `distributeRewardsForAgents`.

### Remediation

In `_distributeContributorRewards`, use the current live impact value from `serviceNftContract.getImpact(serviceId)` (and keep `serviceReward.impact` synchronized) when filtering services and computing `impactAmount` in the distribution loop.

```solidity
        // Distribute service rewards
        uint256 impactAmount = 0;
        uint256 parentAmount = 0;
        uint256 parentShares = uint256(settings.parentShares);
        for (uint i = 0; i < services.length; i++) {
            serviceId = services[i];
            impact = serviceNftContract.getImpact(serviceId);
            if (impact == 0) {
                continue;
            }
            ServiceReward storage serviceReward = _serviceRewards[serviceId];
            serviceReward.impact = impact;
            impactAmount =
                (reward.coreAmount * impact) /
                _rewardImpacts[reward.id][serviceNftContract.getCore(serviceId)];
            parentAmount = contributionNftContract.getParentId(serviceId) == 0
                ? 0
                : ((impactAmount * parentShares) / DENOMINATOR);

            serviceReward.amount += impactAmount - parentAmount;
            serviceReward.parentAmount += parentAmount;
        }
```

## Finding 3: Unrestricted Public Visibility of `ServiceNft.updateImpact` Allows Arbitrary Manipulation of Service and Dataset Impact Scores

### Synopsis

`ServiceNft.updateImpact` is declared `public` without access control or `virtualId` validation. Any external caller can invoke it to reset a minted service's and its dataset's impact scores to zero, or inflate a subsequent model contribution's impact score to its full maturity by passing an unrelated `virtualId`.

### Technical details

In `contracts/contribution/ServiceNft.sol`, `mint` calculates a newly accepted model service's impact score by calling `updateImpact(virtualId, proposalId)` (line 79) immediately before updating the latest core service pointer `_coreServices[virtualId][_cores[proposalId]] = proposalId` (line 80).

However, `updateImpact(uint256 virtualId, uint256 proposalId)` at line 90 is declared `public` instead of `internal` or `private`, and contains no access control or state guards:

```solidity
    function updateImpact(uint256 virtualId, uint256 proposalId) public {
        // Calculate impact
        // Get current service maturity
        uint256 prevServiceId = _coreServices[virtualId][_cores[proposalId]];
        uint256 rawImpact = (_maturities[proposalId] > _maturities[prevServiceId])
            ? _maturities[proposalId] - _maturities[prevServiceId]
            : 0;
        uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

        _impacts[proposalId] = rawImpact;
        if (datasetId > 0) {
            _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
            _impacts[proposalId] = rawImpact - _impacts[datasetId];
            emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
            _maturities[datasetId] = _maturities[proposalId];
        }

        emit SetServiceScore(proposalId, _maturities[proposalId], _impacts[proposalId]);
    }
```

Because `updateImpact` is non-idempotent and depends on the live state of `_coreServices[virtualId][_cores[proposalId]]`, exposing it publicly introduces two severe attack vectors:

1. **Zeroing out impact scores (Griefing / Denial of Rewards):**
   Once `ServiceNft.mint` completes for a model `proposalId`, `_coreServices[virtualId][_cores[proposalId]]` equals `proposalId` (and any later model upgrade for the same core will have an equal or greater maturity). If any user calls `updateImpact(virtualId, proposalId)` after `mint`, `prevServiceId` resolves to `proposalId` (or a newer service with `_maturities[prevServiceId] >= _maturities[proposalId]`). Consequently, `_maturities[proposalId] > _maturities[prevServiceId]` evaluates to `false`, `rawImpact` is computed as `0`, and both `_impacts[proposalId]` and `_impacts[datasetId]` are overwritten with `0`.
   - When `Minter.mint(proposalId)` (lines 153–161) is subsequently called, `IServiceNft(serviceNft).getImpact(nftId)` and `getImpact(datasetId)` return `0`, permanently denying the model contributor, dataset contributor, and IP vault their minted `AgentToken` rewards (since `_mintedNfts[nftId]` is marked `true` or rewards are 0).
   - When `AgentRewardV2._distributeContributorRewards` (lines 271–280) runs for a service whose impact has not yet been cached in `_serviceRewards[serviceId]`, `serviceNftContract.getImpact(serviceId)` returns `0`, permanently excluding the service from ongoing contributor rewards.

2. **Inflating impact scores via arbitrary `virtualId`:**
   `updateImpact` does not verify that `virtualId` matches `IContributionNft(contributionNft).tokenVirtualId(proposalId)`. For any second or subsequent model contribution on a core (where `_maturities[proposalId]` grows cumulatively, e.g., from `100` to `315`, so the legitimate incremental `rawImpact` is `315 - 100 = 215`), a contributor can call `updateImpact(unusedVirtualId, proposalId)` using a `virtualId` where `_coreServices[unusedVirtualId][_cores[proposalId]] == 0`. Since `_maturities[0] == 0`, `rawImpact` is recomputed as `_maturities[proposalId] - 0 = 315` instead of `215`. The contributor can then call `Minter.mint(proposalId)` to mint excess `AgentToken` rewards based on the total cumulative maturity rather than the incremental ELO improvement, and similarly obtain an inflated share of contributor rewards in `AgentRewardV2`.

### Assumptions and preconditions

- A model contribution proposal has been executed via `AgentDAO`, which calls `ServiceNft.mint(virtualId, descHash)` and sets `_maturities[proposalId]` and `_impacts[proposalId]`.
- For the reward-denial attack: the attacker calls `ServiceNft.updateImpact(virtualId, proposalId)` before `Minter.mint(proposalId)` or `AgentRewardV2.distributeRewardsForAgents` is executed.
- For the reward-inflation attack: a model contribution is at least the second model for its `(virtualId, core)` pair (so `_maturities[prevServiceId] > 0`), and the caller invokes `ServiceNft.updateImpact(0, proposalId)` (or any `virtualId` with no service for that core) before `Minter.mint(proposalId)` or `AgentRewardV2.distributeRewardsForAgents` is executed.

### Proof of concept

#### Scenario A: Zeroing Out a Contributor's Impact Score
1. `contributor1` submits a model contribution (`proposalId = P1`) with linked dataset `D1` for `virtualId = 1`, `core = 0`.
2. Voters pass the proposal and `AgentDAO` executes `ServiceNft.mint(1, descHash)`:
   - `_maturities[P1]` is set to `100`.
   - Line 79 calls `updateImpact(1, P1)`: `prevServiceId` is `0`, `rawImpact = 100 - 0 = 100`. With `datasetImpactWeight = 7000` (70%), `_impacts[D1] = 70` and `_impacts[P1] = 30`.
   - Line 80 sets `_coreServices[1][0] = P1`.
3. Before `Minter.mint(P1)` is called, an attacker calls `ServiceNft.updateImpact(1, P1)`:
   - Line 93 reads `prevServiceId = _coreServices[1][0]`, which is now `P1`.
   - Line 94 checks `_maturities[P1] > _maturities[P1]` (`100 > 100`), which is `false`, setting `rawImpact = 0`.
   - Lines 99–102 overwrite `_impacts[P1] = 0` and `_impacts[D1] = 0`.
4. When `Minter.mint(P1)` is called, `amount`, `dataAmount`, and `ipAmount` all evaluate to `0`, and `_mintedNfts[P1]` is permanently set to `true`. Neither `contributor1` nor the dataset owner receives any tokens.

#### Scenario B: Inflating Impact Score to Full Maturity
1. Following `P1` (where `_maturities[P1] = 100` and `_coreServices[1][0] = P1`), `contributor2` submits a second model contribution `P2` for `virtualId = 1`, `core = 0`.
2. `AgentDAO` executes `ServiceNft.mint(1, descHash2)`:
   - `_maturities[P2]` is calculated as `315`.
   - Line 79 calls `updateImpact(1, P2)`: `prevServiceId` is `P1` (maturity `100`), so `rawImpact = 315 - 100 = 215` and `_impacts[P2] = 215`.
   - Line 80 sets `_coreServices[1][0] = P2`.
3. `contributor2` calls `ServiceNft.updateImpact(0, P2)` (where `_coreServices[0][0] == 0`):
   - `prevServiceId` is `0`, and `_maturities[0]` is `0`.
   - `rawImpact = 315 - 0 = 315`.
   - `_impacts[P2]` is overwritten with `315` (instead of `215`).
4. `contributor2` calls `Minter.mint(P2)` and receives `315,000e18` AgentTokens instead of the intended `215,000e18` AgentTokens.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Unauthorized external callers can arbitrarily overwrite `_impacts[proposalId]` and `_impacts[datasetId]` in `ServiceNft`.
- **Business & Financial Impact:**
  - Complete denial of `AgentToken` minting rewards (in `Minter.sol`) and ongoing staking/service rewards (in `AgentRewardV2.sol`) for legitimate model and dataset contributors.
  - Unauthorized inflation of `AgentToken` minting rewards (up to `maxImpact`) and distortion of contributor reward shares in `AgentRewardV2.sol` by replacing incremental ELO gain with cumulative maturity.

### Remediation

Change the visibility of `updateImpact` from `public` to `internal` (or `private`) so that it can only be invoked during `ServiceNft.mint` before `_coreServices[virtualId][_cores[proposalId]]` is updated.

```solidity
    function updateImpact(uint256 virtualId, uint256 proposalId) internal {
        // Calculate impact
        // Get current service maturity
        uint256 prevServiceId = _coreServices[virtualId][_cores[proposalId]];
        uint256 rawImpact = (_maturities[proposalId] > _maturities[prevServiceId])
            ? _maturities[proposalId] - _maturities[prevServiceId]
            : 0;
        uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

        _impacts[proposalId] = rawImpact;
        if (datasetId > 0) {
            _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
            _impacts[proposalId] = rawImpact - _impacts[datasetId];
            emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
            _maturities[datasetId] = _maturities[proposalId];
        }

        emit SetServiceScore(proposalId, _maturities[proposalId], _impacts[proposalId]);
    }
```

## Finding 4: Unvalidated Dataset Attribution and State Overwrite in `ServiceNft.mint` and `ServiceNft.updateImpact`

### Synopsis

In `ServiceNft.mint` and `ServiceNft.updateImpact`, a model contribution's associated `datasetId` is retrieved from `ContributionNft` and its impact and maturity records are overwritten without validating that `datasetId` is an existing, distinct, same-agent, same-core dataset service, enabling cross-record score corruption, reward miscalculation, and duplicate or blocked token minting.

### Technical details

When a governance proposal for a model contribution is executed in `ServiceNft.mint`, lines 79–80 invoke `updateImpact(virtualId, proposalId)` and update `_coreServices[virtualId][_cores[proposalId]]`. Inside `ServiceNft.updateImpact` (lines 90–108), the contract fetches the linked dataset ID via `uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId)` (line 97) and, whenever `datasetId > 0`, splits `rawImpact` between `proposalId` and `datasetId`:

```solidity
_impacts[proposalId] = rawImpact;
if (datasetId > 0) {
    _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
    _impacts[proposalId] = rawImpact - _impacts[datasetId];
    emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
    _maturities[datasetId] = _maturities[proposalId];
}
```

Neither `ServiceNft.mint` nor `ServiceNft.updateImpact` validates the `datasetId` association before mutating `_impacts[datasetId]` and `_maturities[datasetId]`. Specifically:

1. **Unvalidated model-to-dataset association and cross-agent / cross-core reference confusion**: There is no check that `datasetId` has been minted as a `ServiceNft` (`_ownerOf(datasetId) != address(0)`), that it is a dataset rather than a model (`!IContributionNft(contributionNft).isModel(datasetId)`), or that it belongs to the same `virtualId` and core (`IContributionNft(contributionNft).tokenVirtualId(datasetId) == virtualId` and `_cores[datasetId] == _cores[proposalId]`).
   - If `datasetId` references an unminted or rejected dataset proposal, `_impacts[proposalId]` is reduced by `_impacts[datasetId]`, and subsequent calls to `Minter.mint(proposalId)` revert at `IServiceNft(serviceNft).getImpact(datasetId)` (line 159 of `Minter.sol`) because `ServiceNft.getImpact` enforces `_requireOwned(datasetId)`.
   - If `datasetId` references an existing model `ServiceNft` (even from another core or another `virtualId`), `updateImpact` overwrites that model's `_impacts[datasetId]` and `_maturities[datasetId]`, corrupting its baseline maturity for subsequent `rawImpact` calculations as well as its unminted `Minter` rewards.

2. **Source and destination record aliasing (`datasetId == proposalId`)**: If a model contribution specifies `datasetId == proposalId`, `updateImpact` first assigns `_impacts[proposalId] = (rawImpact * datasetImpactWeight) / 10000` and then overwrites `_impacts[proposalId]` with `rawImpact - _impacts[proposalId]`. When `Minter.mint(proposalId)` is executed, both `amount` and `dataAmount` query `getImpact(proposalId)` and transfer both shares to the model owner without any dataset `ServiceNft` existing.

3. **Shared dependency score contamination (dataset reuse across multiple models)**: Multiple model contributions can reference the same `datasetId`. Each subsequent model's `updateImpact` call overwrites `_impacts[datasetId]` and `_maturities[datasetId]`:
   - In `Minter.mint` (lines 140–173 of `Minter.sol`), `_mintedNfts` only tracks the model `nftId`, allowing `dataAmount` to be minted repeatedly for the same `datasetId` across multiple models, or causing an earlier model's unminted `dataAmount` to be overwritten by a later model's impact.
   - In `AgentRewardV2._distributeContributorRewards` (lines 271–302 of `AgentRewardV2.sol`), `_serviceRewards[serviceId].impact` is cached only once when `serviceReward.impact == 0` (line 277), whereas `_rewardImpacts[reward.id][core]` sums the live `serviceNftContract.getImpact(serviceId)` value (line 280). When a dataset's `_impacts[datasetId]` is later overwritten by a second model with a lower `rawImpact` (or overwritten by a cross-referenced model with `rawImpact == 0`), `_rewardImpacts[reward.id][core]` becomes smaller than the sum of cached `serviceReward.impact` numerators (line 294), causing reward distribution to over-allocate rewards for that core or revert on underflow/insufficient balance.

### Assumptions and preconditions

1. A model contribution proposal is created, associated with a `ContributionNft` whose `datasetId` points to an already-used dataset `ServiceNft`, an existing model `ServiceNft`, a `ServiceNft` from another agent/core, or `proposalId` itself.
2. The `AgentDAO` passes and executes the proposal, invoking `ServiceNft.mint(virtualId, descHash)`.

### Proof of concept

1. **Dataset Reuse & Reward Accounting Desynchronization**:
   - Dataset contribution `D` is minted as a `ServiceNft` for `virtualId = 1, core = 0`. Initially, `_impacts[D] = 0`.
   - Model contribution `M1` (referencing `datasetId = D`) is executed via `ServiceNft.mint` with `rawImpact = 200` and `datasetImpactWeight = 7000` (70%). `updateImpact` sets `_impacts[D] = 140` and `_impacts[M1] = 60`.
   - `Minter.mint(M1)` mints tokens for `M1` (based on impact 60) and for `D` (based on impact 140).
   - `AgentRewardV2.distributeRewardsForAgents` runs for Reward #1: `_serviceRewards[D].impact` is cached at `140` (since it was `0`), and `_serviceRewards[M1].impact` is cached at `60`.
   - A second model contribution `M2` also specifies `datasetId = D` and is executed with `rawImpact = 50`. `ServiceNft.updateImpact` overwrites `_impacts[D] = 35` and sets `_impacts[M2] = 15`.
   - `Minter.mint(M2)` executes and mints dataset tokens to the owner of `D` a second time because `_mintedNfts` only checks `M2`.
   - On the next `AgentRewardV2.distributeRewardsForAgents` call (Reward #2), the first loop computes `_rewardImpacts[reward.id][0] = getImpact(D) + getImpact(M1) + getImpact(M2) = 35 + 60 + 15 = 110`. However, the second loop uses the cached `serviceReward.impact` values (`140 + 60 + 15 = 215`), distributing `215 / 110` (nearly 200%) of `reward.coreAmount` for core `0`.

2. **Self-Aliasing (`datasetId == proposalId`)**:
   - A contributor creates model proposal `P` and mints its `ContributionNft` with `isModel = true` and `datasetId = P`.
   - Upon execution of `ServiceNft.mint`, `updateImpact` sets `_impacts[P] = (rawImpact * datasetImpactWeight) / 10000` and then `_impacts[P] = rawImpact - _impacts[P]`.
   - Calling `Minter.mint(P)` reads `getImpact(P)` for both `amount` and `dataAmount`, paying the contributor twice the model's post-deduction impact without any dataset `ServiceNft` having been approved.

### Impact assessment

- **Severity**: High.
- **Technical & Business Consequences**:
  - Overwriting `_impacts[datasetId]` after `_serviceRewards[datasetId].impact` has been initialized in `AgentRewardV2` breaks the invariant that the sum of `serviceReward.impact` for a core equals `_rewardImpacts[reward.id][core]`, leading to over-allocation of contributor rewards for one core and insolvency of the reward pool when contributors claim.
  - Reusing a `datasetId` across multiple models allows `Minter.mint` to mint dataset rewards multiple times for the same dataset, or overwrites an earlier model's unminted dataset impact.
  - Pointing `datasetId` to an existing model `ServiceNft` corrupts that model's `_maturities` and `_impacts` records, distorting subsequent ELO impact deltas and reward distributions.

### Remediation

Validate `datasetId` before attributing impact to it, ensure that a dataset `ServiceNft` is a valid, distinct dataset belonging to the same `virtualId` and core, and prevent overwriting or reusing an already-attributed dataset's impact (or accumulate impacts consistently across `ServiceNft`, `Minter`, and `AgentRewardV2`).

```solidity
    function mint(uint256 virtualId, bytes32 descHash) public returns (uint256) {
        IAgentNft.VirtualInfo memory info = IAgentNft(personaNft).virtualInfo(virtualId);
        require(_msgSender() == info.dao, "Caller is not VIRTUAL DAO");

        IGovernor personaDAO = IGovernor(info.dao);
        bytes memory mintCalldata = abi.encodeWithSignature("mint(uint256,bytes32)", virtualId, descHash);
        address[] memory targets = new address[](1);
        targets[0] = address(this);
        uint256[] memory values = new uint256[](1);
        values[0] = 0;
        bytes[] memory calldatas = new bytes[](1);
        calldatas[0] = mintCalldata;
        uint256 proposalId = personaDAO.hashProposal(targets, values, calldatas, descHash);
        _mint(info.tba, proposalId);
        _cores[proposalId] = IContributionNft(contributionNft).getCore(proposalId);
        // Calculate maturity
        _maturities[proposalId] = IAgentDAO(info.dao).getMaturity(proposalId);

        bool isModel = IContributionNft(contributionNft).isModel(proposalId);
        if (isModel) {
            emit CoreServiceUpdated(virtualId, _cores[proposalId], proposalId);
            updateImpact(virtualId, proposalId);
            _coreServices[virtualId][_cores[proposalId]] = proposalId;
        } else {
            _coreDatasets[virtualId][_cores[proposalId]].push(proposalId);
        }

        emit NewService(proposalId, _cores[proposalId], _maturities[proposalId], _impacts[proposalId], isModel);

        return proposalId;
    }

    function updateImpact(uint256 virtualId, uint256 proposalId) internal {
        // Calculate impact
        // Get current service maturity
        uint256 prevServiceId = _coreServices[virtualId][_cores[proposalId]];
        uint256 rawImpact = (_maturities[proposalId] > _maturities[prevServiceId])
            ? _maturities[proposalId] - _maturities[prevServiceId]
            : 0;
        uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

        _impacts[proposalId] = rawImpact;
        if (datasetId > 0) {
            _requireOwned(datasetId);
            require(datasetId != proposalId, "Invalid datasetId");
            require(!IContributionNft(contributionNft).isModel(datasetId), "Dataset cannot be a model");
            require(IContributionNft(contributionNft).tokenVirtualId(datasetId) == virtualId, "Mismatched virtualId");
            require(_cores[datasetId] == _cores[proposalId], "Mismatched core");
            require(_impacts[datasetId] == 0 && _maturities[datasetId] == 0, "Dataset already attributed");

            _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
            _impacts[proposalId] = rawImpact - _impacts[datasetId];
            emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
            _maturities[datasetId] = _maturities[proposalId];
        }

        emit SetServiceScore(proposalId, _maturities[proposalId], _impacts[proposalId]);
    }
```

## Finding 5: Overwriting Dataset Impact in `ServiceNft.updateImpact` Corrupts Shared Dataset Attribution and Rewards

### Synopsis

In `ServiceNft.updateImpact`, a dataset's impact score `_impacts[datasetId]` is overwritten rather than tracked per model or accumulated when multiple models reference the same dataset. Consequently, subsequent model proposals using the same dataset overwrite its prior impact, corrupting token minting in `Minter.mint` and contributor reward distributions in `AgentRewardV2`.

### Technical details

When a model contribution proposal is executed by the DAO via `ServiceNft.mint`, `ServiceNft.updateImpact(virtualId, proposalId)` is invoked to compute the model's `rawImpact` and split it between the model (`proposalId`) and its associated dataset (`datasetId`):

```solidity
uint256 rawImpact = (_maturities[proposalId] > _maturities[prevServiceId])
    ? _maturities[proposalId] - _maturities[prevServiceId]
    : 0;
uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

_impacts[proposalId] = rawImpact;
if (datasetId > 0) {
    _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
    _impacts[proposalId] = rawImpact - _impacts[datasetId];
    emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
    _maturities[datasetId] = _maturities[proposalId];
}
```

In the protocol architecture, a dataset contribution (`isModel == false`) is minted once with its own `datasetId` and can be referenced by multiple model contributions via `ContributionNft.getDatasetId(proposalId)`. However, `ServiceNft.updateImpact` stores the dataset's impact in a single scalar mapping entry `_impacts[datasetId]` and overwrites it on line 101 (`_impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000`) every time a model referencing `datasetId` is minted.

This causes three direct accounting failures across the protocol:

1. **Loss of Dataset Impact on Subsequent Model Execution:** When a first model $M_1$ referencing dataset $D$ is minted with `rawImpact` $R_1$, $M_1$'s impact is reduced by $D$'s share $I_{D,1} = \lfloor R_1 \times \text{datasetImpactWeight} / 10000 \rfloor$, and `_impacts[D]` is set to $I_{D,1}$. When a second model $M_2$ referencing the same dataset $D$ is later minted with `rawImpact` $R_2$, `_impacts[D]` is overwritten with $I_{D,2} = \lfloor R_2 \times \text{datasetImpactWeight} / 10000 \rfloor$. The initial dataset impact $I_{D,1}$ deducted from $M_1$ is permanently erased instead of being preserved. If $R_2 = 0$ (e.g., $M_2$'s maturity does not exceed the previous core service's maturity), `_impacts[D]` is overwritten to `0`, completely wiping out the dataset's unminted and future rewards.
2. **Corrupted Token Minting in `Minter.mint`:** `Minter.mint(nftId)` calculates `dataAmount` for the dataset owner by reading `IServiceNft(serviceNft).getImpact(datasetId)`. If `Minter.mint(M_1)` is called after $M_2$ has executed, `getImpact(D)` returns $I_{D,2}$ rather than $I_{D,1}$, minting an incorrect amount of agent tokens (or zero if $R_2 = 0$) to the dataset owner and the IP vault.
3. **Cached-vs-Live Impact Mismatch in `AgentRewardV2._distributeContributorRewards`:** During reward distribution, `AgentRewardV2` caches `serviceReward.impact = impact` only on the first distribution where `serviceReward.impact == 0` (lines 277–279), but adds the live `serviceNftContract.getImpact(serviceId)` to `_rewardImpacts[reward.id][core]` (line 280). When a subsequent model $M_2$ overwrites `_impacts[D]` in `ServiceNft`, the denominator `_rewardImpacts[reward.id][core]` uses the newly overwritten `_impacts[D]` while the numerator (line 294) uses the stale cached `serviceReward.impact` from the first reward epoch (or skips $D$ entirely at line 272 if `_impacts[D]` was overwritten to `0`). This breaks conservation of contributor rewards—either under-distributing rewards or reverting/over-allocating when the cached numerator exceeds the reduced denominator.

### Assumptions and preconditions

- A dataset contribution $D$ (`datasetId > 0`) has been approved and minted in `ServiceNft`.
- `datasetImpactWeight` is configured to a non-zero value (e.g., 7000 for 70%, as in the test suite).
- Two model contributions $M_1$ and $M_2$ reference the same `datasetId` $D$ and are executed via `AgentDAO`.

### Proof of concept

1. Contributor A submits dataset contribution $D$ for `virtualId = 1`, `coreId = 0`. Validators vote and execute the proposal, minting `ServiceNft` $D$.
2. Contributor B submits model contribution $M_1$ referencing `datasetId = D`. Validators vote and execute the proposal, resulting in `rawImpact = 100`.
3. `ServiceNft.updateImpact(1, M_1)` sets (`datasetImpactWeight = 7000`):
   - `_impacts[D] = (100 * 7000) / 10000 = 70`
   - `_impacts[M_1] = 100 - 70 = 30`
4. In the first reward distribution epoch, `AgentRewardV2._distributeContributorRewards` records:
   - `_serviceRewards[D].impact = 70`
   - `_serviceRewards[M_1].impact = 30`
5. Contributor C submits model contribution $M_2$ also referencing `datasetId = D`. Suppose $M_2$ achieves a maturity equal to or lower than $M_1$ (`rawImpact = 0`), or achieves a smaller maturity increase (`rawImpact = 20`).
6. When $M_2$ is executed, `ServiceNft.updateImpact(1, M_2)` overwrites `_impacts[D]`:
   - If `rawImpact == 0`: `_impacts[D]` is overwritten to `0`.
   - If `Minter.mint(M_1)` had not yet been called, calling `Minter.mint(M_1)` now reads `getImpact(D) == 0`, paying Contributor A `0` tokens instead of the `70` impact worth of tokens deducted from $M_1$.
   - If `rawImpact == 20`: `_impacts[D]` is overwritten to `14`, and `_impacts[M_2] = 6`. In the next `AgentRewardV2._distributeContributorRewards` call, `_rewardImpacts[reward.id][0]` sums the live impacts `30 (M_1) + 14 (D) + 6 (M_2) = 50`, while the distribution loop uses the cached `_serviceRewards[D].impact = 70`. The sum of numerators (`30 + 70 + 6 = 106`) exceeds the denominator (`50`), corrupting reward shares for the core.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:** Overwriting `_impacts[datasetId]` destroys historical dataset impact attribution whenever a dataset is reused by another model, causing desynchronization between `ServiceNft._impacts` and `AgentRewardV2._serviceRewards` as well as inaccurate `Minter.mint` payouts.
- **Business Implications:** Dataset contributors lose their earned token mints and ongoing contributor reward shares, or reward accounting in `AgentRewardV2` distributes miscalculated amounts that do not sum to `reward.coreAmount`.

### Remediation

Accumulate the dataset's impact in `ServiceNft.updateImpact` so earlier dataset impact contributions are preserved when multiple models use the same dataset, and record the model-specific dataset impact if needed by `Minter`:

```solidity
    function updateImpact(uint256 virtualId, uint256 proposalId) public {
        // Calculate impact
        // Get current service maturity
        uint256 prevServiceId = _coreServices[virtualId][_cores[proposalId]];
        uint256 rawImpact = (_maturities[proposalId] > _maturities[prevServiceId])
            ? _maturities[proposalId] - _maturities[prevServiceId]
            : 0;
        uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

        _impacts[proposalId] = rawImpact;
        if (datasetId > 0) {
            uint256 datasetImpact = (rawImpact * datasetImpactWeight) / 10000;
            _impacts[datasetId] += datasetImpact;
            _impacts[proposalId] = rawImpact - datasetImpact;
            emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
            _maturities[datasetId] = _maturities[proposalId];
        }

        emit SetServiceScore(proposalId, _maturities[proposalId], _impacts[proposalId]);
    }
```

Additionally, ensure `AgentRewardV2._distributeContributorRewards` updates `serviceReward.impact = impact` unconditionally rather than only when `serviceReward.impact == 0`, and that `Minter.mint` computes the dataset's mint amount from the model's own dataset impact share rather than the cumulative `_impacts[datasetId]`.

## Finding 6: Unvalidated `datasetId` Linking and Missing Dataset Claim Tracking in `Minter.mint` and `ContributionNft.mint` Allow Reward Reuse Across Linked Claims

### Synopsis

`ContributionNft.mint` and `Minter.mint` fail to validate `datasetId` or track whether a linked dataset's impact reward has already been claimed. An attacker can link a model contribution to itself, to another model contribution, or to an already-scored dataset, causing `Minter.mint` to pay out the same impact multiple times.

### Technical details

When a model contribution is registered and settled across `ContributionNft`, `ServiceNft`, and `Minter`, the reward accounting relies on `datasetId` to split and distribute `AgentToken` payouts:

1. **Unvalidated `datasetId` assignment in `ContributionNft.mint` (`contracts/contribution/ContributionNft.sol`, lines 80, 89–92):**
   ```solidity
   require(parentId != proposalId, "Cannot be parent of itself");
   ...
   if (isModel_) {
       modelContributions[proposalId] = true;
       modelDatasets[proposalId] = datasetId;
   }
   ```
   While line 80 checks `parentId != proposalId`, `ContributionNft.mint` performs no validation on `datasetId`:
   - It does not verify `datasetId != proposalId`.
   - It does not verify that `datasetId` is actually a dataset contribution rather than another model contribution (`!modelContributions[datasetId]`).
   - It does not verify that `datasetId` belongs to the same `virtualId`.
   - It does not require `ContributionNft.mint` to be called before the proposal is executed in `ServiceNft.mint`.

2. **Impact assignment in `ServiceNft.updateImpact` (`contracts/contribution/ServiceNft.sol`, lines 97–105):**
   ```solidity
   uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

   _impacts[proposalId] = rawImpact;
   if (datasetId > 0) {
       _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
       _impacts[proposalId] = rawImpact - _impacts[datasetId];
       emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
       _maturities[datasetId] = _maturities[proposalId];
   }
   ```

3. **Unchecked `datasetId` reward payout in `Minter.mint` (`contracts/token/Minter.sol`, lines 140–178):**
   ```solidity
   require(!_mintedNfts[nftId], "Already minted");
   ...
   _mintedNfts[nftId] = true;
   ...
   uint256 datasetId = contribution.getDatasetId(nftId);
   uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
   ...
   uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
   uint256 dataAmount = datasetId > 0
       ? (IServiceNft(serviceNft).getImpact(datasetId) * finalImpactMultiplier * 10 ** 18) / DENOM
       : 0;
   uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
   ```
   `Minter.mint` only checks and marks `_mintedNfts[nftId] = true`. It neither checks `!_mintedNfts[datasetId]` nor marks `_mintedNfts[datasetId] = true`, and it does not check that `datasetId` is a valid dataset (`!contribution.isModel(datasetId)`).

This enables multiple reward reuse vectors across linked claims:

- **Linking a model contribution as `datasetId` of another model (`datasetId = M1`):**
  If model proposal `M1` (`isModel == true`) is created and a second model proposal `M2` (`isModel == true`) sets `datasetId = M1`, execution of `M2` in `ServiceNft.mint` sets `_impacts[M1] = (rawImpact_M2 * datasetImpactWeight) / 10000` (e.g., 70% of `rawImpact_M2`) and `_impacts[M2] = rawImpact_M2 - _impacts[M1]` (30% of `rawImpact_M2`).
  - Calling `Minter.mint(M2)` pays 30% of `rawImpact_M2` as `amount` (lines 165–166) and 70% of `rawImpact_M2` (`_impacts[M1]`) as `dataAmount` (lines 170–173).
  - Because `_mintedNfts[M1]` remains `false` and `contribution.isModel(M1)` is `true`, calling `Minter.mint(M1)` pays out `_impacts[M1]` (70% of `rawImpact_M2`) a **second time** as `M1`'s model `amount` (lines 165–166), in addition to `M1`'s own `dataAmount` and `ipAmount` (lines 176–178).

- **Post-execution `ContributionNft.mint` reusing an existing `datasetId`:**
  If a proposer submits a proposal `P` to `AgentDAO` and waits until `P` is executed via `ServiceNft.mint` before calling `ContributionNft.mint`, `IContributionNft(contributionNft).isModel(P)` returns `false` during `ServiceNft.mint` (line 76 of `ServiceNft.sol`). `ServiceNft.mint` mints the `ServiceNft` for `P` and takes the `else` branch without calling `updateImpact`, leaving existing `_impacts` untouched. The proposer then calls `ContributionNft.mint` for `P` with `isModel_ = true` and `datasetId = D` (where `D` is any existing dataset or model with `_impacts[D] > 0`). Calling `Minter.mint(P)` passes `contribution.isModel(P) == true`, reads `IServiceNft(serviceNft).getImpact(D)`, and transfers `dataAmount` and `ipAmount` for `D` again.

- **Self-linking (`datasetId == proposalId`):**
  If a model proposal sets `datasetId = proposalId`, `ServiceNft.updateImpact` sets `_impacts[proposalId] = rawImpact - (rawImpact * datasetImpactWeight) / 10000`, and `Minter.mint(proposalId)` reads `_impacts[proposalId]` for both `amount` and `dataAmount`, paying that impact out twice in the same transaction.

### Assumptions and preconditions

1. An `AgentDAO`, `AgentToken`, `ContributionNft`, `ServiceNft`, and `Minter` are deployed and initialized for a virtual agent, and `Minter` holds `AgentToken` balances to distribute to contributors.
2. A contributor can create proposals in `AgentDAO` (or has proposals approved by DAO voters).

### Proof of concept

1. Contributor A creates dataset contribution `D1` (`isModel = false`) and model contribution `M1` (`isModel = true`, `datasetId = D1`).
2. `M1` is voted on and executed via `ServiceNft.mint` with `rawImpact = 100` and `datasetImpactWeight = 7000` (70%):
   - `_impacts[D1] = 70`
   - `_impacts[M1] = 30`
3. Before calling `Minter.mint(M1)`, Contributor A creates a second model contribution `M2` (`isModel = true`, `datasetId = M1`), linking the model NFT `M1` as `M2`'s dataset.
4. `M2` is voted on and executed via `ServiceNft.mint` with `rawImpact = 215`:
   - `ServiceNft.updateImpact` sets `_impacts[M1] = (215 * 7000) / 10000 = 150` and `_impacts[M2] = 215 - 150 = 65`.
5. Contributor A calls `Minter.mint(M2)`:
   - `Minter.mint` marks `_mintedNfts[M2] = true`.
   - `Minter.mint` transfers `amount` for `getImpact(M2) = 65` to `ownerOf(M2)` and `dataAmount` for `getImpact(M1) = 150` to `ownerOf(M1)` (total 215 impact, plus `ipAmount`).
6. Contributor A then calls `Minter.mint(M1)`:
   - `_mintedNfts[M1]` is still `false` and `contribution.isModel(M1)` is `true`.
   - `Minter.mint` transfers `amount` for `getImpact(M1) = 150` a **second time** to `ownerOf(M1)` and `dataAmount` for `getImpact(D1) = 70` to `ownerOf(D1)` (plus `ipAmount`).
7. Alternatively, after step 2 and calling `Minter.mint(M1)`, Contributor A creates a dataset proposal `P3`, lets `ServiceNft.mint` execute while `isModel(P3)` is `false` (leaving `_impacts[D1] = 70` intact), then calls `ContributionNft.mint` for `P3` with `isModel_ = true, datasetId = D1`, and finally calls `Minter.mint(P3)` to collect `D1`'s 70 impact reward again.

### Impact assessment

- **Severity:** High
- **Technical Impact:** `_impacts[datasetId]` can be claimed multiple times across `Minter.mint` executions, minting/transferring unauthorized `AgentToken` rewards to contributors and `ipVault`.
- **Business Impact:** Attackers can inflate token payouts and drain the `AgentToken` reserve held by the `Minter` contract, diluting token holders and leaving insufficient tokens for legitimate contributors.

### Remediation

1. In `ContributionNft.mint`, ensure `proposalId` has not yet been minted in `ServiceNft`, and when `isModel_` is `true` and `datasetId != 0`, validate that `datasetId` exists, belongs to the same `virtualId`, and is a dataset contribution (`!modelContributions[datasetId]` and `datasetId != proposalId`).
2. In `Minter.mint`, verify and record `_mintedNfts[datasetId]` (or validate that each dataset impact is only paid once and `datasetId` is not a model contribution).

Corrected `Minter.mint` in `contracts/token/Minter.sol`:

```solidity
    function mint(uint256 nftId) public noReentrant {
        require(!_mintedNfts[nftId], "Already minted");

        uint256 virtualId = IContributionNft(contributionNft).tokenVirtualId(nftId);
        require(virtualId != 0, "Agent not found");

        _mintedNfts[nftId] = true;

        address tokenAddress = IAgentNft(agentNft).virtualInfo(virtualId).token;
        IContributionNft contribution = IContributionNft(contributionNft);
        require(contribution.isModel(nftId), "Not a model contribution");

        uint256 finalImpactMultiplier = _getImpactMultiplier(virtualId);
        uint256 datasetId = contribution.getDatasetId(nftId);
        if (datasetId != 0) {
            require(!_mintedNfts[datasetId], "Dataset already minted");
            require(!contribution.isModel(datasetId), "Dataset cannot be a model");
            require(contribution.tokenVirtualId(datasetId) == virtualId, "Dataset virtualId mismatch");
            _mintedNfts[datasetId] = true;
        }

        uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
        if (impact > maxImpact) {
            impact = maxImpact;
        }
        uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 dataAmount = datasetId > 0
            ? (IServiceNft(serviceNft).getImpact(datasetId) * finalImpactMultiplier * 10 ** 18) / DENOM
            : 0;
        uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;

        // Mint to model owner
        if (amount > 0) {
            address modelOwner = IERC721(contributionNft).ownerOf(nftId);
            IAgentToken(tokenAddress).transfer(modelOwner, amount);
        }

        // Mint to Dataset owner
        if (datasetId != 0) {
            address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
            IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount);
        }

        // To IP vault
        if (ipAmount > 0) {
            IAgentToken(tokenAddress).transfer(ipVault, ipAmount);
        }
    }
```

## Finding 7: Unprotected `ServiceNft.updateImpact` Allows Arbitrary Manipulation of Contribution Impacts and Token Rewards in `Minter.mint`

### Synopsis

`ServiceNft.updateImpact` is declared `public` without access control or provenance validation. Any external caller can invoke it before `Minter.mint` to inflate model and dataset impacts using an arbitrary `virtualId` or zero out legitimate impacts, causing unauthorized over-minting or denial of `AgentToken` rewards.

### Technical details

In `Minter.mint(uint256 nftId)` (`contracts/token/Minter.sol:153-164`), token rewards for the model contributor, dataset contributor, and IP vault are computed directly from external impact values queried from `ServiceNft`:

```solidity
uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
if (impact > maxImpact) {
    impact = maxImpact;
}
uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
uint256 dataAmount = datasetId > 0
    ? (IServiceNft(serviceNft).getImpact(datasetId) * finalImpactMultiplier * 10 ** 18) / DENOM
    : 0;
uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
```

In `ServiceNft.sol`, `_impacts[proposalId]` and `_impacts[datasetId]` are intended to be calculated once during `ServiceNft.mint` before `_coreServices[virtualId][_cores[proposalId]]` is updated to `proposalId` (lines 79-80). However, `ServiceNft.updateImpact(uint256 virtualId, uint256 proposalId)` at line 90 is marked `public` with no access control and does not validate that `proposalId` belongs to `virtualId`:

```solidity
function updateImpact(uint256 virtualId, uint256 proposalId) public {
    uint256 prevServiceId = _coreServices[virtualId][_cores[proposalId]];
    uint256 rawImpact = (_maturities[proposalId] > _maturities[prevServiceId])
        ? _maturities[proposalId] - _maturities[prevServiceId]
        : 0;
    uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

    _impacts[proposalId] = rawImpact;
    if (datasetId > 0) {
        _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
        _impacts[proposalId] = rawImpact - _impacts[datasetId];
        emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
        _maturities[datasetId] = _maturities[proposalId];
    }

    emit SetServiceScore(proposalId, _maturities[proposalId], _impacts[proposalId]);
}
```

This flaw creates three attack vectors affecting `Minter.mint`:

1. **Reward Inflation via Cross-VirtualId Baseline Reset:** A contributor whose model `proposalId` achieved a small marginal ELO improvement over `prevServiceId` (e.g., `_maturities[proposalId] = 500`, `_maturities[prevServiceId] = 490`, `rawImpact = 10`) can call `ServiceNft.updateImpact(0, proposalId)` prior to calling `Minter.mint(proposalId)`. Because `_coreServices[0][_cores[proposalId]]` is `0` and `_maturities[0]` is `0`, `rawImpact` is recalculated as `_maturities[proposalId] - 0 = 500` (the entire cumulative maturity rather than the marginal delta). Furthermore, while `Minter.mint` caps the model's `impact` at `maxImpact` (lines 154-156), `IServiceNft(serviceNft).getImpact(datasetId)` (line 159) is not capped by `maxImpact`.
2. **Reward Griefing / Zeroing Out:** After `ServiceNft.mint` finishes executing line 80, `_coreServices[virtualId][_cores[proposalId]]` equals `proposalId`. Any user can call `ServiceNft.updateImpact(virtualId, proposalId)`, causing `prevServiceId == proposalId` and `rawImpact = 0`. This overwrites both `_impacts[proposalId]` and `_impacts[datasetId]` to `0`, preventing the model owner, dataset owner, and IP vault from receiving their rewards in `Minter.mint` (as well as in `AgentRewardV2._distributeContributorRewards`).
3. **Cross-Contribution Dataset Impact Contamination:** Because anyone can propose a contribution in `AgentDAO` and mint a `ContributionNft` referencing an arbitrary existing `datasetId` (`ContributionNft.sol:73,91`), an attacker can pass that unexecuted `proposalId` (where `_maturities[proposalId] == 0`) to `ServiceNft.updateImpact(virtualId, attackerProposalId)` to overwrite `_impacts[datasetId]` and `_maturities[datasetId]` to `0` for any target dataset before its associated model calls `Minter.mint`.

### Assumptions and preconditions

- A model contribution proposal has been executed by the `AgentDAO`, minting the corresponding `ServiceNft` (`nftId`), or is about to have its rewards claimed via `Minter.mint(nftId)`.
- `Minter` holds `AgentToken` balances to distribute to contributors and the IP vault.

### Proof of concept

1. Suppose `virtualId = 1` currently has an active core service `prevServiceId = 10` with `_maturities[10] = 315`.
2. A new model proposal `proposalId = 20` (referencing `datasetId = 15`, with `datasetImpactWeight = 7000` i.e. 70%) is executed by `AgentDAO`, calling `ServiceNft.mint(1, descHash)`.
3. Suppose the new maturity is `_maturities[20] = 325`. During `ServiceNft.mint`, `updateImpact(1, 20)` computes `rawImpact = 325 - 315 = 10`, setting `_impacts[15] = 7` and `_impacts[20] = 3`. Then `_coreServices[1][coreId]` is updated to `20`.
4. **Inflation Path:** Before `Minter.mint(20)` is called, the contributor calls `ServiceNft.updateImpact(0, 20)`.
   - `prevServiceId = _coreServices[0][coreId] = 0`, so `_maturities[0] = 0`.
   - `rawImpact = 325 - 0 = 325`.
   - `_impacts[15]` is updated to `(325 * 7000) / 10000 = 227`, and `_impacts[20]` is updated to `325 - 227 = 98`.
   - The contributor then calls `Minter.mint(20)`. `Minter.mint` reads `getImpact(20) = 98` and `getImpact(15) = 227`, minting 32.5x the intended `AgentToken` rewards.
5. **Griefing Path:** Alternatively, before `Minter.mint(20)` is called, an attacker calls `ServiceNft.updateImpact(1, 20)` (and `Minter.mint(20)` in the same transaction).
   - Because `_coreServices[1][coreId]` is already `20`, `prevServiceId = 20`.
   - `rawImpact = 0`, overwriting `_impacts[20] = 0` and `_impacts[15] = 0`.
   - `Minter.mint(20)` marks `_mintedNfts[20] = true` and transfers `0` tokens to the model owner, dataset owner, and IP vault, permanently burning their minting right.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Unauthorized external callers can arbitrarily inflate or zero out `_impacts[nftId]` and `_impacts[datasetId]` in `ServiceNft` prior to `Minter.mint(nftId)`.
- **Business Implications:** Malicious contributors can mint vastly inflated `AgentToken` rewards (draining the `Minter` contract and diluting token holders), while griefing attackers can permanently deny legitimate model contributors, dataset contributors, and the IP vault their token rewards.

### Remediation

Make `ServiceNft.updateImpact` an `internal` or `private` function so that it can only be invoked during `ServiceNft.mint` before `_coreServices[virtualId][_cores[proposalId]]` is updated. Additionally, apply `maxImpact` capping to the combined/raw impact (or to the dataset impact as well) in `Minter.mint`.

In `contracts/contribution/ServiceNft.sol`:
```solidity
    function updateImpact(uint256 virtualId, uint256 proposalId) internal {
        // Calculate impact
        // Get current service maturity
        uint256 prevServiceId = _coreServices[virtualId][_cores[proposalId]];
        uint256 rawImpact = (_maturities[proposalId] > _maturities[prevServiceId])
            ? _maturities[proposalId] - _maturities[prevServiceId]
            : 0;
        uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

        _impacts[proposalId] = rawImpact;
        if (datasetId > 0) {
            _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
            _impacts[proposalId] = rawImpact - _impacts[datasetId];
            emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
            _maturities[datasetId] = _maturities[proposalId];
        }

        emit SetServiceScore(proposalId, _maturities[proposalId], _impacts[proposalId]);
    }
```

## Finding 8: Missing `maxImpact` Cap on Dataset Impact in `Minter.mint` Allows Bypassing Maximum Reward Limits

### Synopsis

In `Minter.mint`, the `maxImpact` upper bound is applied only to the model's individual impact (`IServiceNft(serviceNft).getImpact(nftId)`) while leaving the associated dataset's impact (`IServiceNft(serviceNft).getImpact(datasetId)`) completely uncapped. Contributors can include a dataset in a model contribution to bypass `maxImpact` and mint excessive `AgentToken` rewards.

### Technical details

When a model contribution proposal is executed in `ServiceNft.mint`, `ServiceNft.updateImpact` computes the total ELO improvement `rawImpact = _maturities[proposalId] - _maturities[prevServiceId]`. If the model contribution references a dataset (`datasetId > 0`), `ServiceNft.updateImpact` splits `rawImpact` between the dataset and the model according to `datasetImpactWeight` (in basis points):

```solidity
_impacts[proposalId] = rawImpact;
if (datasetId > 0) {
    _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
    _impacts[proposalId] = rawImpact - _impacts[datasetId];
    ...
}
```

As documented in `Minter.mint` (`// 1. ELO impact amount, to be shared between model and dataset owner`), the total ELO impact of a contribution is shared between the model owner and the dataset owner, and `maxImpact` is intended to cap the ELO impact eligible for token minting.

However, in `Minter.mint` (lines 153–161):

```solidity
uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
if (impact > maxImpact) {
    impact = maxImpact;
}
uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
uint256 dataAmount = datasetId > 0
    ? (IServiceNft(serviceNft).getImpact(datasetId) * finalImpactMultiplier * 10 ** 18) / DENOM
    : 0;
uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
```

This logic is flawed in two ways:
1. **Unbounded Dataset Impact:** `IServiceNft(serviceNft).getImpact(datasetId)` at line 159 is never checked or capped against `maxImpact`.
2. **Post-Split Cap Check on Model Impact:** `impact` (`getImpact(nftId)`) has already been reduced by `_impacts[datasetId]` in `ServiceNft.updateImpact`. Comparing only the model's post-split portion against `maxImpact` means the model itself can still claim up to `maxImpact` even after `datasetImpactWeight` of the `rawImpact` was allocated to the dataset.

Consequently, when `datasetId > 0`, the combined impact used to calculate `amount + dataAmount` (and by extension `ipAmount`) is `min(rawImpact - datasetImpact, maxImpact) + datasetImpact`, which exceeds `maxImpact` whenever `rawImpact > maxImpact` and grows without bound as `rawImpact` increases.

### Assumptions and preconditions

1. `maxImpact` is configured to a value that can be exceeded by the ELO rating gain (`rawImpact`) of a contribution, or `maxImpact` is lowered by the owner via `setMaxImpact`.
2. `datasetImpactWeight` in `ServiceNft` is non-zero (e.g., `7000` for 70%, as seen in the test suite where dataset owner receives 70,000 tokens and model owner receives 30,000 tokens).
3. A model contribution is submitted referencing a valid `datasetId > 0`.

### Proof of concept

Consider the configuration from the test suite (`test/contribution.js`):
- `datasetImpactWeight = 7000` (70% to dataset, 30% to model)
- `finalImpactMultiplier = 10_000_000` (`1000 * DENOM`, i.e., 1,000 tokens per 1 unit of impact)
- `ipShare = 1000` (10%)
- Suppose `maxImpact = 100` (so the maximum intended reward for a contribution's ELO impact is `100 * 1000 = 100,000` tokens across model and dataset owners, plus `10,000` tokens to `ipVault`).

1. A contributor submits a dataset contribution `c1` (`isModel = false`) and a model contribution `nftId` (`isModel = true`, `datasetId = c1`).
2. Validators vote unanimously in favor of the model (`[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]`), resulting in `rawImpact = 215` when the proposal executes `ServiceNft.mint`.
3. `ServiceNft.updateImpact` sets:
   - `_impacts[c1] = (215 * 7000) / 10000 = 150`
   - `_impacts[nftId] = 215 - 150 = 65`
4. Anyone calls `Minter.mint(nftId)`:
   - `impact = getImpact(nftId) = 65`. Since `65 <= maxImpact (100)`, `impact` remains `65`.
   - `amount = 65 * 1000e18 = 65,000e18`.
   - `dataAmount = getImpact(c1) * 1000e18 = 150 * 1000e18 = 150,000e18` (uncapped).
   - `ipAmount = (65,000e18 + 150,000e18) * 1000 / 10000 = 21,500e18`.
5. In total, `215,000e18` tokens are distributed to the contributors (plus `21,500e18` to `ipVault`) instead of the capped `100,000e18` (plus `10,000e18` to `ipVault`), completely bypassing `maxImpact = 100`.

### Impact assessment

- **Severity:** Medium / High.
- **Technical Impact:** The `maxImpact` safeguard in `Minter.mint` is ineffective whenever a contribution includes a dataset. Both `amount + dataAmount` and `ipAmount` exceed the intended maximum payout bound.
- **Business Impact:** Over-distribution of `AgentToken` reserves held by the `Minter` contract, prematurely depleting token allocations and diluting token holders.

### Remediation

Cap the combined contribution impact (`rawImpact = modelImpact + datasetImpact`) against `maxImpact` before computing the model and dataset reward amounts, or scale down both the model and dataset impacts proportionally when their sum exceeds `maxImpact`.

```solidity
        uint256 modelImpact = IServiceNft(serviceNft).getImpact(nftId);
        uint256 datasetImpact = datasetId > 0 ? IServiceNft(serviceNft).getImpact(datasetId) : 0;
        uint256 totalImpact = modelImpact + datasetImpact;
        if (totalImpact > maxImpact) {
            modelImpact = (modelImpact * maxImpact) / totalImpact;
            datasetImpact = maxImpact - modelImpact;
        }
        uint256 amount = (modelImpact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 dataAmount = (datasetImpact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
```

## Finding 9: Cross-Contribution Contamination and Unvalidated `datasetId` Association in `Minter.mint` and `ServiceNft.updateImpact` Allow Reward Manipulation

### Synopsis

`Minter.mint` computes model and dataset rewards from `ServiceNft._impacts`, which stores impact per token ID without validating `datasetId` provenance, type, or single-use exclusivity. Legitimate `ServiceNft.mint` executions can overwrite shared `_impacts[datasetId]` entries across models or agents and bypass `maxImpact` on dataset rewards.

### Technical details

In `Minter.mint` (`contracts/token/Minter.sol` lines 152–161), token payouts for the model contributor, dataset contributor, and IP vault are derived at claim time from `IServiceNft(serviceNft).getImpact(nftId)` and `IServiceNft(serviceNft).getImpact(datasetId)`:

```solidity
uint256 datasetId = contribution.getDatasetId(nftId);
uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
if (impact > maxImpact) {
    impact = maxImpact;
}
uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
uint256 dataAmount = datasetId > 0
    ? (IServiceNft(serviceNft).getImpact(datasetId) * finalImpactMultiplier * 10 ** 18) / DENOM
    : 0;
uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
```

This reward calculation relies on external state in `ContributionNft` and `ServiceNft` that lacks provenance and isolation guarantees:

1. **Unvalidated `datasetId` provenance and type in `ContributionNft.mint` and `ServiceNft.mint`:**
   In `ContributionNft.mint` (`contracts/contribution/ContributionNft.sol` lines 89–92), a model proposal records an arbitrary `datasetId` (`modelDatasets[proposalId] = datasetId`) without verifying that `datasetId` exists, belongs to the same `virtualId` (`tokenVirtualId(datasetId) == virtualId`), or represents a dataset contribution (`!isModel(datasetId)`).

2. **Cross-contribution overwrite of shared `_impacts` state during `ServiceNft.mint`:**
   When a model proposal is executed via `ServiceNft.mint`, `updateImpact` (`contracts/contribution/ServiceNft.sol` lines 97–105) writes directly to `_impacts[datasetId]` and `_maturities[datasetId]`:
   ```solidity
   uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

   _impacts[proposalId] = rawImpact;
   if (datasetId > 0) {
       _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
       _impacts[proposalId] = rawImpact - _impacts[datasetId];
       emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
       _maturities[datasetId] = _maturities[proposalId];
   }
   ```
   Because `_impacts` is a flat mapping keyed solely by token ID (`mapping(uint256 tokenId => uint256 impact)`), any subsequent model proposal referencing the same `datasetId` overwrites `_impacts[datasetId]` before earlier models call `Minter.mint`. Furthermore, a model proposal in one agent (`virtualId`) can set `datasetId` to a model or dataset token ID belonging to a different agent, overwriting the target token's `_impacts` and `_maturities` upon proposal execution.

3. **Uncapped and repeatable dataset reward claims in `Minter.mint`:**
   `_mintedNfts[nftId]` only tracks whether a model `nftId` has been minted, not whether `datasetId` has already been rewarded. Multiple models referencing the same `datasetId` can each mint `dataAmount` for that dataset. In addition, `Minter.mint` caps `impact` to `maxImpact` at lines 154–156 AFTER `ServiceNft.updateImpact` has already split `rawImpact` into `_impacts[proposalId]` and `_impacts[datasetId]`, and `Minter.mint` never applies `maxImpact` to `IServiceNft(serviceNft).getImpact(datasetId)` at line 159.

### Assumptions and preconditions

- At least one agent (`virtualId`) is deployed and active in the protocol.
- Either multiple model contributions within the same agent reference the same `datasetId`, or a proposer/DAO in any agent creates and executes a model proposal whose `datasetId` points to an existing dataset or model `tokenId`.

### Proof of concept

1. **Scenario 1 (Cross-Contribution Overwrite & Double Dataset Payout):**
   - Contributor A submits dataset `D` for Agent 1.
   - Contributor B submits Model `M1` referencing `datasetId = D`, which passes governance with `rawImpact = 200` and `datasetImpactWeight = 7000` (70%). `ServiceNft.mint` sets `_impacts[M1] = 60` and `_impacts[D] = 140`.
   - Before `Minter.mint(M1)` is called, Contributor C submits Model `M2` also referencing `datasetId = D`, which passes governance with `rawImpact = 10` (or `0` if maturity did not increase).
   - `ServiceNft.mint` for `M2` overwrites `_impacts[D]` from `140` to `7` (or `0`).
   - When `Minter.mint(M1)` is called, `dataAmount` is calculated using the overwritten `_impacts[D] = 7` (or `0`) instead of `140`, depriving Contributor A of 95%–100% of the earned dataset reward and reducing `ipAmount` for `ipVault`. Conversely, if `M1` calls `Minter.mint(M1)` before `M2` is executed, both `Minter.mint(M1)` and `Minter.mint(M2)` pay out dataset rewards for `D`.

2. **Scenario 2 (Cross-Agent Impact Contamination):**
   - Model `M_target` in Agent 1 is minted via `ServiceNft.mint` with `_impacts[M_target] = 200`.
   - In Agent 2, a model proposal `M_attacker` is created with `datasetId = M_target` and executed by Agent 2's DAO with `rawImpact = 0`.
   - `ServiceNft.mint` executes `updateImpact`, setting `_impacts[M_target] = 0`.
   - When `Minter.mint(M_target)` is subsequently called for Agent 1, `IServiceNft(serviceNft).getImpact(M_target)` returns `0`, resulting in zero tokens minted for the model owner and IP vault.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Shared mutable `_impacts[datasetId]` state can be overwritten across contributions or across agents, and dataset rewards bypass `maxImpact` capping and single-claim accounting.
- **Business Impact:** Legitimate contributors and the IP vault can lose their entitled `AgentToken` rewards (reduced or zeroed out before claiming), or excess `AgentToken` rewards can be minted via repeated dataset claims and uncapped dataset impacts.

### Remediation

Validate in `ContributionNft.mint` (or `ServiceNft.mint`) that any non-zero `datasetId` belongs to the same `virtualId` and is a valid dataset contribution (`!isModel(datasetId)`). In `Minter.mint`, cap the total combined impact of the model and its dataset to `maxImpact` (or derive the dataset's share from the model's capped impact) and ensure a dataset's impact for a given model cannot be mutated by subsequent proposals.

In `contracts/token/Minter.sol`:

```solidity
        uint256 finalImpactMultiplier = _getImpactMultiplier(virtualId);
        uint256 datasetId = contribution.getDatasetId(nftId);
        uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
        uint256 dataImpact = 0;
        if (datasetId > 0) {
            require(
                contribution.tokenVirtualId(datasetId) == virtualId && !contribution.isModel(datasetId),
                "Invalid datasetId"
            );
            uint16 weight = ServiceNft(serviceNft).datasetImpactWeight();
            uint256 rawImpact = (impact * DENOM) / (DENOM - weight);
            if (rawImpact > maxImpact) {
                rawImpact = maxImpact;
            }
            dataImpact = (rawImpact * weight) / DENOM;
            impact = rawImpact - dataImpact;
        } else if (impact > maxImpact) {
            impact = maxImpact;
        }
        uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 dataAmount = (dataImpact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;

        // Mint to model owner
        if (amount > 0) {
```

## Finding 10: Ignored `ipShareOverrides` in `Minter.mint` causes per-agent IP share configurations to be bypassed

### Synopsis

In `Minter.mint`, the calculation of `ipAmount` directly uses the global `ipShare` state variable instead of checking `ipShareOverrides[virtualId]`. Consequently, any custom IP share configured by an Agent DAO via `setIPShareOverride` is completely ignored, resulting in inaccurate IP vault token distributions.

### Technical details

The `Minter` contract is designed to support both global reward parameters (`impactMultiplier` and `ipShare`) and per-agent overrides stored in `impactMulOverrides` and `ipShareOverrides`. Specifically, an Agent DAO can configure a custom IP share for its `virtualId` by calling `setIPShareOverride(uint256 virtualId, uint256 _ipShare)` (lines 99–102):

```solidity
function setIPShareOverride(uint256 virtualId, uint256 _ipShare) public onlyAgentDAO(virtualId) {
    ipShareOverrides[virtualId] = _ipShare;
    emit AgentIPShareUpdated(virtualId, _ipShare);
}
```

While `Minter.mint` retrieves the impact multiplier using the internal helper `_getImpactMultiplier(virtualId)` (which checks `impactMulOverrides[virtualId]` before falling back to `impactMultiplier`), it fails to perform an equivalent lookup for the IP share on line 161:

```solidity
uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
```

Because `ipShareOverrides[virtualId]` is never read in `Minter.mint` (or anywhere else in the contract), the global `ipShare` is always applied when computing `ipAmount`.

### Assumptions and preconditions

1. An Agent DAO executes a governance action calling `Minter.setIPShareOverride(virtualId, customIpShare)` to set a non-zero custom IP share that differs from the global `ipShare`.
2. A model contribution for that `virtualId` is accepted and `Minter.mint(nftId)` is called.

### Proof of concept

1. Suppose `Minter` is initialized with a global `ipShare = 1000` (10%) and `impactMultiplier = 10000` (1x).
2. The Agent DAO for `virtualId = 1` passes and executes a proposal calling `Minter.setIPShareOverride(1, 2000)` to increase the IP share for its agent to 20% (`ipShareOverrides[1] = 2000`).
3. A contributor submits a model contribution (`nftId`) for `virtualId = 1` that achieves an impact of `100` (with no dataset, so `amount = 100e18` and `dataAmount = 0`).
4. When `Minter.mint(nftId)` is invoked, line 161 computes:
   `ipAmount = ((100e18 + 0) * 1000) / 10000 = 10e18`
   instead of using `ipShareOverrides[1] = 2000` to compute `20e18`.
5. `ipVault` receives `10e18` `AgentToken` instead of the expected `20e18`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** The `ipShareOverrides` state variable is completely ineffective, breaking the per-agent IP share customization functionality exposed to Agent DAOs.
- **Business Impact:** `ipVault` receives either fewer or more `AgentToken` rewards than dictated by the Agent DAO's governance configuration whenever `ipShareOverrides[virtualId]` differs from the global `ipShare`.

### Remediation

Add an internal helper function `_getIPShare(uint256 virtualId)` analogous to `_getImpactMultiplier(uint256 virtualId)` and use it when computing `ipAmount` in `Minter.mint`.

```solidity
    function _getIPShare(uint256 virtualId) internal view returns (uint256) {
        uint256 share = ipShareOverrides[virtualId];
        if (share == 0) {
            share = ipShare;
        }
        return share;
    }
```

In `Minter.mint`:

```solidity
        uint256 finalImpactMultiplier = _getImpactMultiplier(virtualId);
        uint256 finalIPShare = _getIPShare(virtualId);
        uint256 datasetId = contribution.getDatasetId(nftId);
        uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
        if (impact > maxImpact) {
            impact = maxImpact;
        }
        uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 dataAmount = datasetId > 0
            ? (IServiceNft(serviceNft).getImpact(datasetId) * finalImpactMultiplier * 10 ** 18) / DENOM
            : 0;
        uint256 ipAmount = ((amount + dataAmount) * finalIPShare) / DENOM;
```

## Finding 11: Unrestricted Public Visibility of `ServiceNft.updateImpact` Allows Arbitrary Manipulation of Minting Rewards in `Minter.mint`

### Synopsis

`ServiceNft.updateImpact` is declared with `public` visibility and lacks access control. Any external caller can invoke it before `Minter.mint` settles rewards, either zeroing out or artificially inflating `_impacts[proposalId]` and `_impacts[datasetId]`, which directly corrupts the `AgentToken` payouts transferred in `Minter.mint`.

### Technical details

When a model contribution proposal is executed by an `AgentDAO`, `ServiceNft.mint` calculates the service's maturity and computes its impact before updating the current core service pointer:

```solidity
bool isModel = IContributionNft(contributionNft).isModel(proposalId);
if (isModel) {
    emit CoreServiceUpdated(virtualId, _cores[proposalId], proposalId);
    updateImpact(virtualId, proposalId);
    _coreServices[virtualId][_cores[proposalId]] = proposalId;
}
```

`ServiceNft.updateImpact(uint256 virtualId, uint256 proposalId)` (at `contracts/contribution/ServiceNft.sol:90`) computes `rawImpact` by comparing `_maturities[proposalId]` against `_maturities[prevServiceId]`, where `prevServiceId = _coreServices[virtualId][_cores[proposalId]]`, and writes the resulting scores to `_impacts[proposalId]` and `_impacts[datasetId]`.

Later, `Minter.mint(uint256 nftId)` settles the contribution rewards by reading `IServiceNft(serviceNft).getImpact(nftId)` and `IServiceNft(serviceNft).getImpact(datasetId)` to compute `amount`, `dataAmount`, and `ipAmount`, and transfers those `AgentToken` amounts at lines 165–178:

```solidity
if (amount > 0) {
    address modelOwner = IERC721(contributionNft).ownerOf(nftId);
    IAgentToken(tokenAddress).transfer(modelOwner, amount);
}

if (datasetId != 0) {
    address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
    IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount);
}

if (ipAmount > 0) {
    IAgentToken(tokenAddress).transfer(ipVault, ipAmount);
}
```

Because `ServiceNft.updateImpact` is `public` rather than `internal`, any unprivileged account can call it directly after `ServiceNft.mint` and prior to `Minter.mint(nftId)` with arbitrary `virtualId` and `proposalId` arguments:

1. **Reward Zeroing (Denial of Rewards):** After `ServiceNft.mint` executes line 80 (`_coreServices[virtualId][_cores[proposalId]] = proposalId`), calling `updateImpact(virtualId, proposalId)` with the model's actual `virtualId` sets `prevServiceId = proposalId`. Because `_maturities[proposalId] > _maturities[proposalId]` evaluates to `false`, `rawImpact` becomes `0`, overwriting `_impacts[proposalId]` and `_impacts[datasetId]` to `0`. Subsequent execution of `Minter.mint(proposalId)` transfers `0` tokens to `modelOwner` and `datasetOwner` (and `0` to `ipVault`), permanently burning the contributor's one-time minting right since `_mintedNfts[nftId]` is set to `true`.
2. **Reward Inflation (Excessive Minting):** Calling `updateImpact(otherVirtualId, proposalId)` with any `otherVirtualId` that has no prior service for `_cores[proposalId]` (`_coreServices[otherVirtualId][_cores[proposalId]] == 0`) sets `prevServiceId = 0` (`_maturities[0] == 0`). Consequently, `rawImpact` is overwritten with the entire `_maturities[proposalId]` value rather than the incremental improvement over the previous service (even if the new model actually had lower maturity than the previous service and initially received `0` impact). Calling `Minter.mint(proposalId)` then transfers inflated `amount`, `dataAmount`, and `ipAmount` payouts.

### Assumptions and preconditions

- A model contribution proposal has been executed via `AgentDAO`, minting the corresponding `ServiceNft` for `proposalId`.
- `Minter.mint(proposalId)` has not yet been called (or `updateImpact` is called in the same block / front-run before `Minter.mint(proposalId)`).

### Proof of concept

1. For `virtualId = 1` and `coreId = 0`, an initial model contribution is minted with maturity `300`, setting `_coreServices[1][0] = firstProposalId` and `_maturities[firstProposalId] = 300`.
2. A second model contribution (`proposalId = P2`, linked to `datasetId = D2`) is executed for `virtualId = 1` with maturity `350`. During `ServiceNft.mint`:
   - `updateImpact(1, P2)` calculates `rawImpact = 350 - 300 = 50`.
   - With `datasetImpactWeight = 7000` (70%), `_impacts[D2] = 35` and `_impacts[P2] = 15`.
   - `_coreServices[1][0]` is updated to `P2`.
3. **Scenario A (Inflating Rewards):** Before calling `Minter.mint(P2)`, the contributor calls `ServiceNft.updateImpact(999, P2)`, where `999` is an unused `virtualId` (`_coreServices[999][0] == 0`).
   - `prevServiceId` resolves to `0`, and `_maturities[0]` is `0`.
   - `rawImpact` is recalculated as `350 - 0 = 350` (7x higher than the true impact of `50`).
   - `_impacts[D2]` is overwritten to `245` and `_impacts[P2]` is overwritten to `105`.
   - The contributor calls `Minter.mint(P2)` and receives `AgentToken` transfers corresponding to `350` impact instead of `50`.
4. **Scenario B (Zeroing Rewards):** Alternatively, before an honest contributor calls `Minter.mint(P2)`, an external party calls `ServiceNft.updateImpact(1, P2)` followed by `Minter.mint(P2)`.
   - `prevServiceId` resolves to `_coreServices[1][0] == P2`.
   - `_maturities[P2] > _maturities[P2]` is `false`, setting `rawImpact = 0`, `_impacts[P2] = 0`, and `_impacts[D2] = 0`.
   - `Minter.mint(P2)` marks `_mintedNfts[P2] = true` while transferring `0` tokens to `modelOwner` and `ipVault`, and `0` `dataAmount` to `datasetOwner`.

### Impact assessment

- **Severity:** High.
- **Technical Impact:** Any external caller can arbitrarily zero out or inflate the impact scores of model and dataset `ServiceNft` tokens prior to reward settlement.
- **Business Impact:** Malicious contributors can mint up to `maxImpact` (plus uncapped dataset impact) worth of `AgentToken` for every model contribution regardless of actual ELO maturity delta (even when the model fails to improve maturity), draining `AgentToken` reserves held by `Minter`. Conversely, attackers can grief honest contributors and the IP vault by resetting their earned impact scores to zero before `Minter.mint` is called.

### Remediation

Change the visibility of `updateImpact` in `ServiceNft.sol` from `public` to `internal` so it can only be invoked during `ServiceNft.mint` before `_coreServices[virtualId][_cores[proposalId]]` is updated.

```solidity
    function updateImpact(uint256 virtualId, uint256 proposalId) internal {
        // Calculate impact
        // Get current service maturity
        uint256 prevServiceId = _coreServices[virtualId][_cores[proposalId]];
        uint256 rawImpact = (_maturities[proposalId] > _maturities[prevServiceId])
            ? _maturities[proposalId] - _maturities[prevServiceId]
            : 0;
        uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);

        _impacts[proposalId] = rawImpact;
        if (datasetId > 0) {
            _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
            _impacts[proposalId] = rawImpact - _impacts[datasetId];
            emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
            _maturities[datasetId] = _maturities[proposalId];
        }

        emit SetServiceScore(proposalId, _maturities[proposalId], _impacts[proposalId]);
    }
```

## Finding 12: Atomic Multi-Recipient Push Payout in `Minter.mint` Allows a Failing Recipient Transfer to Block Rewards for All Other Recipients

### Synopsis

`Minter.mint` atomically pushes `AgentToken` rewards to `modelOwner`, `datasetOwner`, and `ipVault` in a single transaction. If any single transfer reverts due to recipient-specific conditions or an unminted `ContributionNft` returning `address(0)`, the entire `mint` call reverts, permanently locking rewards for the other honest recipients.

### Technical details

In `contracts/token/Minter.sol`, `mint(uint256 nftId)` calculates token rewards for up to three distinct parties—the model contributor (`modelOwner`), the dataset contributor (`datasetOwner`), and the protocol IP vault (`ipVault`)—and distributes all of them sequentially via direct push transfers on lines 165–178:

```solidity
        // Mint to model owner
        if (amount > 0) {
            address modelOwner = IERC721(contributionNft).ownerOf(nftId);
            IAgentToken(tokenAddress).transfer(modelOwner, amount);
        }

        // Mint to Dataset owner
        if (datasetId != 0) {
            address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
            IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount);
        }

        // To IP vault
        if (ipAmount > 0) {
            IAgentToken(tokenAddress).transfer(ipVault, ipAmount);
        }
```

Because all three transfers are coupled within a single atomic transaction, a revert in any one recipient's `transfer` call causes the entire `Minter.mint(nftId)` execution to revert, denying payouts to the remaining parties. In this architecture, an individual transfer can revert under several reachable conditions:

1. **Unminted `ContributionNft` returning `address(0)`:** `ContributionNft.ownerOf(uint256 tokenId)` (lines 153–155 of `ContributionNft.sol`) overrides `ERC721Upgradeable.ownerOf` to return `_ownerOf(tokenId)` instead of `_requireOwned(tokenId)`, returning `address(0)` when the `ContributionNft` has not been minted. While `ServiceNft.getImpact(datasetId)` checks `_requireOwned(datasetId)` on `ServiceNft`, `ServiceNft` minting during DAO proposal execution does not require the proposer to have called `ContributionNft.mint` for `datasetId`. Consequently, `datasetOwner` on line 171 can resolve to `address(0)` (or `modelOwner` on line 165 can resolve to `address(0)` if the model's `ContributionNft` was only registered for proposal parameters without minting to a valid owner), causing `IAgentToken(tokenAddress).transfer(address(0), ...)` to revert under standard ERC-20 receiver validation.
2. **Recipient-dependent restrictions in `AgentToken`:** `AgentToken.transfer` (`contracts/virtualPersona/AgentToken.sol` line 500) invokes `_transfer(owner, to, amount, (isLiquidityPool(owner) || isLiquidityPool(to)))` and enforces token configuration limits (such as `maxTokensPerWallet` and `maxTokensPerTransaction` defined in `IAgentToken`) as well as liquidity-pool tax/autoswap logic. Because `ContributionNft` is a standard transferable ERC-721 token, `datasetOwner` (or `modelOwner`) can transfer the `ContributionNft` to an address that violates `AgentToken`'s recipient holding limits or triggers a reverting liquidity pool interaction.
3. **Unchecked zero `dataAmount` transfer:** Unlike `amount > 0` (line 164) and `ipAmount > 0` (line 176), line 170 checks `if (datasetId != 0)` rather than `if (dataAmount > 0)`, forcing an external call to `ownerOf(datasetId)` and `transfer(datasetOwner, 0)` even when the dataset's calculated token reward is zero.

As a result, an uncooperative `datasetOwner` (or an unminted dataset `ContributionNft`) can prevent `modelOwner` and `ipVault` from ever receiving their rewards for `nftId`.

### Assumptions and preconditions

1. A valid model contribution proposal `nftId` referencing a dataset `datasetId != 0` has been approved and executed by the `AgentDAO`, minting the corresponding `ServiceNft` records.
2. `Minter` holds sufficient `AgentToken` balance to distribute the rewards.
3. Either `datasetOwner` transfers the dataset `ContributionNft` to an address that cannot receive `dataAmount` of `AgentToken` (e.g., an address exceeding `maxTokensPerWallet`), or `datasetId` was never minted on `ContributionNft` so `ContributionNft.ownerOf(datasetId)` returns `address(0)`.

### Proof of concept

1. Contributor A creates a dataset proposal `datasetId` in `AgentDAO`. The proposal passes and executes `ServiceNft.mint`, minting `datasetId` on `ServiceNft`.
2. Either:
   - Contributor A never calls `ContributionNft.mint` for `datasetId`, leaving `ContributionNft.ownerOf(datasetId) == address(0)`, **or**
   - Contributor A mints `ContributionNft(datasetId)` and transfers it to an address whose `AgentToken` balance is at `maxTokensPerWallet` (so any further `AgentToken.transfer` to `datasetOwner` reverts).
3. Contributor B creates a model contribution proposal `nftId` that uses `datasetId` (`isModel_ = true`, `datasetId = datasetId`), mints `ContributionNft(nftId)`, and the proposal succeeds and executes on `ServiceNft`.
4. Contributor B (or any caller) invokes `Minter.mint(nftId)` to distribute the model, dataset, and IP vault rewards.
5. `Minter.mint(nftId)` executes `IAgentToken(tokenAddress).transfer(modelOwner, amount)` on line 166, then reaches line 171–172 (`IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount)`), which reverts.
6. The entire `Minter.mint(nftId)` transaction reverts, preventing Contributor B (`modelOwner`) and `ipVault` from receiving their `AgentToken` rewards.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Denial of Service (DoS) on `Minter.mint(nftId)` for any contribution where one of the coupled recipients (`modelOwner`, `datasetOwner`, or `ipVault`) causes `AgentToken.transfer` to revert.
- **Business Impact:** Honest contributors (`modelOwner` or `datasetOwner`) and the protocol's `ipVault` are permanently blocked from claiming their earned `AgentToken` rewards for an approved contribution if a counterparty contributor's NFT is unminted or held by a restricted address.

### Remediation

Decouple the multi-recipient payouts by using a pull-over-push reward claiming pattern (or by recording failed payouts in a `claimableRewards` mapping if a direct transfer fails or the recipient address is `address(0)`), and guard the dataset transfer with `dataAmount > 0`.

Replace lines 163–178 of `contracts/token/Minter.sol` with isolated transfer handling (accompanied by a `mapping(address => mapping(address => uint256)) public pendingRewards` state variable and a `claimPendingRewards(address tokenAddress)` function):

```solidity
        // Mint to model owner
        if (amount > 0) {
            address modelOwner = IERC721(contributionNft).ownerOf(nftId);
            _distributeOrEscrow(tokenAddress, modelOwner, amount);
        }

        // Mint to Dataset owner
        if (dataAmount > 0) {
            address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
            _distributeOrEscrow(tokenAddress, datasetOwner, dataAmount);
        }

        // To IP vault
        if (ipAmount > 0) {
            _distributeOrEscrow(tokenAddress, ipVault, ipAmount);
        }
```

Where `_distributeOrEscrow` safely attempts the transfer and credits `pendingRewards` on failure instead of reverting the entire `mint` transaction.

## Finding 13: Unvalidated Zero-Address Beneficiary Resolution in `Minter.mint` via Non-Reverting `ContributionNft.ownerOf`

### Synopsis

In `Minter.mint`, beneficiary addresses `modelOwner` and `datasetOwner` are resolved via `ContributionNft.ownerOf`, which overrides OpenZeppelin's `ERC721Upgradeable.ownerOf` to return `_ownerOf(tokenId)` instead of `_requireOwned(tokenId)`. Consequently, non-existent or unminted `ContributionNft` IDs resolve to `address(0)` without reverting, causing invalid reward transfers or permanent minting denial-of-service.

### Technical details

According to EIP-721 and OpenZeppelin's `ERC721Upgradeable.ownerOf`, querying the owner of a non-existent token must revert via `_requireOwned(tokenId)`. However, `ContributionNft.ownerOf` overrides this behavior at lines 153–155:

```solidity
function ownerOf(uint256 tokenId) public view override(IERC721, ERC721Upgradeable) returns (address) {
    return _ownerOf(tokenId);
}
```

Because `_ownerOf(tokenId)` directly reads the underlying mapping slot without checking for existence, it silently returns `address(0)` for any `tokenId` that has not been minted in `ContributionNft` (or has been burned).

In `Minter.sol` (lines 165–173), `Minter.mint` resolves reward beneficiaries directly through `IERC721(contributionNft).ownerOf(...)` without verifying that the returned beneficiary address is non-zero:

```solidity
// Mint to model owner
if (amount > 0) {
    address modelOwner = IERC721(contributionNft).ownerOf(nftId);
    IAgentToken(tokenAddress).transfer(modelOwner, amount);
}

// Mint to Dataset owner
if (datasetId != 0) {
    address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
    IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount);
}
```

In the protocol lifecycle, a dataset proposal executed by `AgentDAO` mints a `ServiceNft` in `ServiceNft.mint` even if `ContributionNft.mint` was never called for that dataset proposal (since `IContributionNft.getCore` returns `0` and `IContributionNft.isModel` returns `false` for unminted `ContributionNft` tokens). When a model contribution references such a `datasetId` (or if either `ContributionNft` is burned/unminted when `Minter.mint` is invoked), `IServiceNft(serviceNft).getImpact(datasetId)` succeeds because the `ServiceNft` exists, while `IERC721(contributionNft).ownerOf(datasetId)` returns `address(0)`. `Minter.mint` then attempts to transfer `AgentToken` rewards to `address(0)` (and executes the transfer even when `dataAmount == 0`), resulting in either token loss to `address(0)` or a revert in `IAgentToken.transfer` that permanently blocks `Minter.mint(nftId)` for the model owner and IP vault as well.

### Assumptions and preconditions

1. A dataset proposal is created and executed in `AgentDAO`, minting a `ServiceNft` for `datasetId`, while the corresponding `ContributionNft` for `datasetId` has not been minted (or has been burned).
2. A model contribution referencing `datasetId` is proposed, minted in `ContributionNft`, and executed in `AgentDAO`.
3. Any user calls `Minter.mint(nftId)` to distribute the model, dataset, and IP vault rewards.

### Proof of concept

1. A proposer submits a dataset contribution proposal (`datasetId`) to `AgentDAO`, and voters pass and execute the proposal without the proposer calling `ContributionNft.mint` for `datasetId`.
2. `ServiceNft.mint` mints the `ServiceNft` for `datasetId` to the agent's TBA and records `datasetId` in `_coreDatasets`.
3. A model contributor submits a model contribution proposal (`nftId`) referencing `datasetId`, calls `ContributionNft.mint` for `nftId`, and the DAO executes the proposal. `ServiceNft.updateImpact` assigns maturity and impact scores to both `nftId` and `datasetId`.
4. When `Minter.mint(nftId)` is called:
   - `IServiceNft(serviceNft).getImpact(datasetId)` succeeds because `datasetId` exists in `ServiceNft`.
   - At line 171, `IERC721(contributionNft).ownerOf(datasetId)` returns `address(0)` because `ContributionNft.ownerOf` uses `_ownerOf` and `datasetId` was not minted in `ContributionNft`.
   - At line 172, `IAgentToken(tokenAddress).transfer(address(0), dataAmount)` is executed with `datasetOwner == address(0)`, either reverting the entire `Minter.mint(nftId)` call (blocking rewards for `modelOwner` and `ipVault`) or misdirecting `dataAmount` tokens to `address(0)`.

### Impact assessment

- **Severity**: High / Medium
- **Technical Effect**: `Minter.mint` resolves an unminted or burned `ContributionNft` owner to `address(0)` and passes it directly to `IAgentToken(tokenAddress).transfer`. If `AgentToken` reverts on transfers to `address(0)`, the model contributor and IP vault are blocked from claiming rewards for `nftId` until the dataset proposer mints the missing `ContributionNft` (if still possible). If `AgentToken` does not revert on `address(0)` transfers, the dataset reward tokens are permanently burned/lost at `address(0)` while `_mintedNfts[nftId]` is marked `true`.

### Remediation

Validate that resolved beneficiaries are non-zero addresses before transferring tokens (and only transfer dataset rewards when `dataAmount > 0`):

```solidity
        // Mint to model owner
        if (amount > 0) {
            address modelOwner = IERC721(contributionNft).ownerOf(nftId);
            require(modelOwner != address(0), "Invalid model owner");
            IAgentToken(tokenAddress).transfer(modelOwner, amount);
        }

        // Mint to Dataset owner
        if (datasetId != 0 && dataAmount > 0) {
            address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
            require(datasetOwner != address(0), "Invalid dataset owner");
            IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount);
        }

        // To IP vault
        if (ipAmount > 0) {
            IAgentToken(tokenAddress).transfer(ipVault, ipAmount);
        }
```

## Finding 14: Unvalidated Dataset Contribution Binding in `Minter.mint` Allows Cross-Agent Reward Misdirection and Impact Manipulation

### Synopsis

In `Minter.mint`, dataset reward payouts only check `if (datasetId != 0)` before resolving `datasetOwner` and transferring agent tokens. Because `datasetId` is never validated to be a dataset contribution belonging to the same `virtualId`, unvalidated or cross-agent linked assets can misdirect payouts or corrupt reward accounting.

### Technical details

When a model contribution proposal is registered in `ContributionNft.mint`, the proposer supplies an arbitrary `datasetId` parameter:

```solidity
if (isModel_) {
    modelContributions[proposalId] = true;
    modelDatasets[proposalId] = datasetId;
}
```

Neither `ContributionNft.mint`, `ServiceNft.mint`, `ServiceNft.updateImpact`, nor `Minter.mint` validates that `datasetId` is an eligible dataset contribution belonging to the same agent (`virtualId`) and core type as the model contribution `nftId`.

In `Minter.mint(uint256 nftId)`, the contract verifies that `nftId` belongs to a valid `virtualId` and is a model contribution (`require(contribution.isModel(nftId), "Not a model contribution")`), and resolves `tokenAddress = IAgentNft(agentNft).virtualInfo(virtualId).token`. However, when distributing the dataset reward at lines 170–173:

```solidity
// Mint to Dataset owner
if (datasetId != 0) {
    address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
    IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount);
}
```

`Minter.mint` performs no eligibility or binding checks on `datasetId`:

1. **Cross-agent beneficiary misbinding**: `Minter.mint` does not verify that `IContributionNft(contributionNft).tokenVirtualId(datasetId) == virtualId`. A model contribution for Agent A can link a `datasetId` belonging to Agent B, causing `Minter.mint` to pay Agent A's `AgentToken` (`tokenAddress`) to the owner of Agent B's NFT.
2. **Unvalidated linked-asset eligibility and cross-agent impact overwrite**: `Minter.mint` does not verify `!contribution.isModel(datasetId)` or `_mintedNfts[datasetId]`. Because `ServiceNft.updateImpact` writes `_impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000` for any `datasetId > 0` without checking which `virtualId` owns `datasetId`, a model proposal executed in one agent DAO can reference a `datasetId` (or an unminted model `nftId`) from another agent. This overwrites `_impacts[datasetId]` in `ServiceNft`—either inflating `dataAmount` (and `ipAmount`) to drain the target agent's tokens when `Minter.mint` is called, or zeroing out the target contribution's impact before its owner calls `Minter.mint`.
3. **Invalid beneficiary resolution**: `ContributionNft.ownerOf` overrides `ERC721Upgradeable.ownerOf` to return `_ownerOf(tokenId)` directly without `_requireOwned(tokenId)`, returning `address(0)` if `datasetId` was not minted in `ContributionNft` (or reverting in `ServiceNft.getImpact(datasetId)` if `datasetId` was never minted as a `ServiceNft`), which permanently blocks `Minter.mint(nftId)` for the model owner and IP vault.

### Assumptions and preconditions

1. At least one target agent (`virtualId_A`) is deployed with its `AgentToken` funded in `Minter`.
2. A model contribution in `virtualId_A` references a cross-agent or ineligible `datasetId`, or an attacker controlling governance in a secondary agent (`virtualId_B`) executes a model proposal whose `datasetId` points to a contribution ID from `virtualId_A`.

### Proof of concept

1. In Agent A (`virtualId = 1`), contributor Alice creates a dataset contribution `datasetId = D1` (which is accepted and minted in `ServiceNft`) and contributor Bob creates a model contribution `M1` linked to `datasetId = D1`.
2. Before `Minter.mint(M1)` is executed (or while `M1` is being proposed), an attacker who controls Agent B (`virtualId = 2`) creates and executes a model proposal `M2` in Agent B's `AgentDAO` with `datasetId = D1` (or `datasetId = M1`).
3. When `ServiceNft.mint` executes for `M2`, `ServiceNft.updateImpact(2, M2)` overwrites `_impacts[D1]` using Agent B's maturity delta:
   - If `M2` has `rawImpact = 0`, `_impacts[D1]` is overwritten to `0`. When `Minter.mint(M1)` is subsequently called, `dataAmount` is `0`, depriving Alice of her 70% dataset reward (`70,000e18` tokens) and reducing `ipVault`'s `ipAmount`.
   - Conversely, if a model proposal in Agent A links `datasetId` belonging to Agent B, `Minter.mint` at lines 170–173 resolves `datasetOwner = IERC721(contributionNft).ownerOf(datasetId)` and transfers Agent A's `AgentToken` to Agent B's contributor.

### Impact assessment

- **Severity**: High
- **Consequences**:
  - Unauthorized misdirection of agent tokens to cross-agent or non-dataset NFT owners during `Minter.mint`.
  - Cross-agent corruption of `_impacts` in `ServiceNft`, allowing an attacker to zero out legitimate dataset/model payouts or inflate `dataAmount` and `ipAmount` in `Minter.mint`.
  - Permanent denial of service of `Minter.mint(nftId)` if `datasetId` is not a minted `ServiceNft` or `ContributionNft`.

### Remediation

Validate in `Minter.mint` (and during contribution registration/impact calculation) that `datasetId` is a valid dataset contribution (`!contribution.isModel(datasetId)`) belonging to the same `virtualId` (`contribution.tokenVirtualId(datasetId) == virtualId`) and core type before resolving `datasetOwner` and transferring `dataAmount`.

```solidity
        // Mint to model owner
        if (amount > 0) {
            address modelOwner = IERC721(contributionNft).ownerOf(nftId);
            require(modelOwner != address(0), "Invalid model owner");
            IAgentToken(tokenAddress).transfer(modelOwner, amount);
        }

        // Mint to Dataset owner
        if (datasetId != 0 && dataAmount > 0) {
            require(
                contribution.tokenVirtualId(datasetId) == virtualId &&
                    !contribution.isModel(datasetId) &&
                    contribution.getCore(datasetId) == contribution.getCore(nftId),
                "Invalid dataset contribution"
            );
            address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
            require(datasetOwner != address(0), "Invalid dataset owner");
            IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount);
        }

        // To IP vault
        if (ipAmount > 0) {
            IAgentToken(tokenAddress).transfer(ipVault, ipAmount);
        }
```
