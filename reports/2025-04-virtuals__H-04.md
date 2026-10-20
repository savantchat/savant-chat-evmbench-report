# Audit report — 2025-04-virtuals/H-04

## Finding 1: Missing `datasetId` Validation in `ContributionNft.mint` Allows Hijacking Dataset Impact and Stealing Agent Tokens

### Synopsis

`ContributionNft.mint` accepts an arbitrary `datasetId` when minting a model contribution without validating that `datasetId` is a valid dataset belonging to the same virtual agent. A proposer can link a model contribution to another contribution's ID, overwriting its `ServiceNft` impact and diverting minted `AgentToken` rewards.

### Technical details

In `ContributionNft.mint` (lines 75-91), when a user mints a model contribution (`isModel_ == true`), the contract records `modelDatasets[proposalId] = datasetId` at line 91 without performing any validation on `datasetId`. Specifically, `ContributionNft.mint` fails to verify that:
1. `datasetId` corresponds to an existing contribution NFT (or is `0` when no dataset is used).
2. `datasetId` belongs to the same `virtualId` (`_contributionVirtualId[datasetId] == virtualId`).
3. `datasetId` belongs to the same `coreId` (`_cores[datasetId] == coreId`).
4. `datasetId` is actually a dataset contribution rather than another model contribution (`!modelContributions[datasetId]`).
5. `datasetId` is not equal to `proposalId` (`datasetId != proposalId`).

Because `modelDatasets[proposalId]` is exposed via `getDatasetId(proposalId)`, downstream contracts trust this association during proposal execution and token minting:

1. **Impact Overwrite and Corruption in `ServiceNft`:**
   When the model proposal is executed by the `AgentDAO`, `ServiceNft.mint` calls `ServiceNft.updateImpact(virtualId, proposalId)` (lines 90-108). If `datasetId > 0`, `updateImpact` executes:
   ```solidity
   _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
   _impacts[proposalId] = rawImpact - _impacts[datasetId];
   _maturities[datasetId] = _maturities[proposalId];
   ```
   Because `datasetId` was never validated in `ContributionNft.mint`, a model contribution can point to:
   - An existing model or dataset `ServiceNft` (even from a different `virtualId` or `coreId`), overwriting its `_impacts[datasetId]` and `_maturities[datasetId]`. If the targeted `datasetId` had a high impact and its owner had not yet called `Minter.mint(datasetId)` (or before `AgentRewardV2._distributeContributorRewards` snapshotted `serviceReward.impact`), its impact is overwritten with a smaller or zero value.
   - `datasetId == proposalId`, which causes `_impacts[proposalId]` to be set to `rawImpact - _impacts[proposalId]`, while `Minter.mint` subsequently pays out both `amount` (for `nftId`) and `dataAmount` (for `datasetId == nftId`), allowing the model owner to claim both the model share and the dataset share without any actual dataset contribution.

2. **Cross-Agent Token Minting in `Minter.mint`:**
   In `Minter.mint(uint256 nftId)` (lines 134-179), `tokenAddress` is fetched from `virtualId = IContributionNft(contributionNft).tokenVirtualId(nftId)`. If `datasetId != 0`, `Minter.mint` reads `IServiceNft(serviceNft).getImpact(datasetId)` and transfers `dataAmount` of `virtualId`'s `AgentToken` to `IERC721(contributionNft).ownerOf(datasetId)`. By referencing a `datasetId` owned by the attacker (such as a contribution NFT minted under a completely different `virtualId` where the attacker controls the DAO or a low-value agent, or `datasetId == proposalId`), the attacker receives `dataAmount` of `virtualId`'s `AgentToken` without having an accepted dataset in `virtualId`.

### Assumptions and preconditions

1. A proposer creates a proposal in `AgentDAO` (either by holding sufficient voting power above `proposalThreshold()` or via the admin account) and calls `ContributionNft.mint` with `isModel_ = true`.
2. For reward/token extraction or impact overwrite, the proposal in `AgentDAO` succeeds and is executed to mint the `ServiceNft`, or `datasetId` references an already-minted `ServiceNft`.

### Proof of concept

#### Scenario 1: Self-referential `datasetId == proposalId` to claim dataset rewards without a dataset
1. Proposer creates a model contribution proposal with ID `P` in `AgentDAO` for `virtualId = 1`.
2. Proposer calls `ContributionNft.mint(proposer, 1, coreId, uri, P, 0, true, P)`, setting `datasetId = P`.
3. Line 91 sets `modelDatasets[P] = P`.
4. The proposal `P` passes voting and is executed, calling `ServiceNft.mint(1, descHash)`.
5. `ServiceNft.updateImpact(1, P)` runs with `rawImpact = 100` and `datasetImpactWeight = 7000` (70%):
   - `_impacts[P]` is first set to `(100 * 7000) / 10000 = 70`.
   - `_impacts[P]` is then updated to `100 - _impacts[P] = 30`.
6. When `Minter.mint(P)` is called, `amount` (30% of impact) and `dataAmount` (30% of impact, since `datasetId == P`) are both transferred to `proposer`, even though no dataset contribution was ever created or approved.

#### Scenario 2: Overwriting another model contribution's impact before `Minter.mint`
1. Contributor A has an approved and executed model contribution `M1` with `_impacts[M1] = 215`, and has not yet called `Minter.mint(M1)`.
2. Proposer B submits a new model contribution `M2` (even on a different `virtualId` or `coreId` where Proposer B controls the votes) and calls `ContributionNft.mint(B, virtualId2, coreId, uri, M2, 0, true, M1)`, setting `datasetId = M1`.
3. When `M2` is executed with `rawImpact = 0` (e.g., maturity did not increase), `ServiceNft.updateImpact` sets `_impacts[M1] = (0 * datasetImpactWeight) / 10000 = 0`.
4. When Contributor A calls `Minter.mint(M1)`, `IServiceNft(serviceNft).getImpact(M1)` returns `0`, permanently denying Contributor A their 215,000 `AgentToken` reward.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:** Arbitrary corruption of `_impacts` and `_maturities` mappings in `ServiceNft` for any existing `tokenId`, and unauthorized attribution of dataset rewards in `Minter` and `AgentRewardV2`.
- **Business Implications:** Attackers can grief legitimate model and dataset contributors by zeroing out or reducing their earned impact scores prior to token minting or reward distribution, or siphon `AgentToken` emissions using cross-agent or self-referential dataset IDs.

### Remediation

In `ContributionNft.mint`, validate that if `isModel_` is `true` and `datasetId != 0`, the referenced `datasetId` is an existing dataset contribution (not a model) that belongs to the same `virtualId` and `coreId` (and is accepted by the DAO). Additionally, ensure `parentId` (when non-zero) exists and belongs to the same `virtualId`.

```solidity
        IGovernor personaDAO = getAgentDAO(virtualId);
        require(
            msg.sender == personaDAO.proposalProposer(proposalId),
            "Only proposal proposer can mint Contribution NFT"
        );
        require(parentId != proposalId, "Cannot be parent of itself");
        if (parentId != 0) {
            require(_ownerOf(parentId) != address(0), "Parent does not exist");
            require(_contributionVirtualId[parentId] == virtualId, "Invalid parent virtualId");
        }

        _mint(to, proposalId);
        _setTokenURI(proposalId, newTokenURI);
        _contributionVirtualId[proposalId] = virtualId;
        _parents[proposalId] = parentId;
        _children[parentId].push(proposalId);
        _cores[proposalId] = coreId;

        if (isModel_) {
            if (datasetId != 0) {
                require(_ownerOf(datasetId) != address(0) && datasetId != proposalId, "Invalid datasetId");
                require(!modelContributions[datasetId], "Dataset cannot be a model");
                require(_contributionVirtualId[datasetId] == virtualId, "Dataset virtualId mismatch");
                require(_cores[datasetId] == coreId, "Dataset coreId mismatch");
            }
            modelContributions[proposalId] = true;
            modelDatasets[proposalId] = datasetId;
        }
```

## Finding 2: Lack of Proposal Binding and Cross-DAO Proposal ID Isolation in `ContributionNft.mint` Enables Contribution Hijacking and Unauthorized Token Minting

### Synopsis

`ContributionNft.mint` authenticates callers solely by checking `msg.sender == personaDAO.proposalProposer(proposalId)` for a user-supplied `virtualId`. Because OpenZeppelin Governor `proposalId` hashes exclude the governor address and `mint` neither verifies proposal calldata/state nor isolates token IDs per DAO, attackers can hijack contribution NFTs and mint unauthorized agent tokens.

### Technical details

In `ContributionNft.sol`, `mint` is intended to mint a `ContributionNft` corresponding to a contribution proposal submitted to a specific Virtual Persona's `AgentDAO`:

```solidity
IGovernor personaDAO = getAgentDAO(virtualId);
require(
    msg.sender == personaDAO.proposalProposer(proposalId),
    "Only proposal proposer can mint Contribution NFT"
);
require(parentId != proposalId, "Cannot be parent of itself");

_mint(to, proposalId);
_setTokenURI(proposalId, newTokenURI);
_contributionVirtualId[proposalId] = virtualId;
```

This authorization and identity-binding logic has three closely coupled flaws rooted in the check at lines 75–79:

1. **Lack of Cross-DAO Proposal Identifier Isolation:**
   Each `virtualId` has its own `AgentDAO` (`GovernorUpgradeable` clone), whereas `ContributionNft` and `ServiceNft` are global singletons that key ERC-721 token IDs directly by `proposalId`. In `GovernorUpgradeable.hashProposal(targets, values, calldatas, descriptionHash)`, neither the governor contract address nor `virtualId` is included in the hash computation. Consequently, submitting the identical `(targets, values, calldatas, description)` tuple to two different `AgentDAO` instances produces the exact same `proposalId`.

2. **Lack of Proposal-to-Contribution Semantic Binding:**
   `ContributionNft.mint` does not verify that `proposalId` actually encodes a `ServiceNft.mint(virtualId, descHash)` call for the supplied `virtualId` (or that `parentId` and `datasetId` belong to the same `virtualId`). A proposal created on `virtualId = 1`'s DAOwhose calldata targets `ServiceNft.mint(2, descHash)` still records `proposalProposer(proposalId)` on `virtualId = 1`'s DAO, allowing the proposer to bind `_contributionVirtualId[proposalId] = 1` in `ContributionNft` while executing `ServiceNft.mint(2, descHash)` on `virtualId = 2`'s DAO.

3. **Lack of Lifecycle State Validation:**
   `ContributionNft.mint` only checks `personaDAO.proposalProposer(proposalId)` and never checks `personaDAO.state(proposalId)`. Even a Pending, Defeated, or Canceled proposal that never passes governance in `virtualId = 1`'s DAO remains permanently eligible to mint `ContributionNft(proposalId)`.

As a result, an attacker who controls or holds proposal power in a secondary agent DAO (`virtualId = 2`) can:
- **Hijack or DoS legitimate contributions on `virtualId = 1`:** By re-proposing the same `(targets, values, calldatas, description)` on `virtualId = 2`'s DAO (or front-running `ContributionNft.mint`), the attacker becomes `proposalProposer(proposalId)` on `dao2` and calls `ContributionNft.mint(attacker, 2, ..., proposalId, ...)`. This blocks the legitimate proposer on `virtualId = 1` from minting `ContributionNft(proposalId)` and makes the attacker the `ownerOf(proposalId)` when `AgentRewardV2._claimServiceRewards(proposalId)` checks `IERC721(contributionNft).ownerOf(nftId) == account`.
- **Mint valuable `AgentToken`s from `Minter.mint` using self-approved impact:** By submitting an unpassed proposal on `dao1` (`virtualId = 1`) with calldata `ServiceNft.mint(2, descHash)`, minting `ContributionNft` with `_contributionVirtualId[proposalId] = 1`, and then proposing and executing `ServiceNft.mint(2, descHash)` on their own `dao2` (`virtualId = 2`), the attacker sets `_impacts[proposalId]` in `ServiceNft`. Calling `Minter.mint(proposalId)` then reads `tokenVirtualId(proposalId) == 1` from `ContributionNft` and `getImpact(proposalId)` from `ServiceNft`, transferring `virtualId = 1`'s `AgentToken` to the attacker.

### Assumptions and preconditions

- At least two Virtual Personas exist (`virtualId = 1` and `virtualId = 2`), where the attacker has sufficient voting/proposing power in `virtualId = 2`'s `AgentDAO` (or also holds at least `proposalThreshold` in `virtualId = 1`'s `AgentDAO` for the cross-DAO `Minter.mint` attack).
- For the front-running / hijacking vector, the legitimate proposer on `virtualId = 1` submits `AgentDAO.propose` before calling `ContributionNft.mint` (or in separate transactions, as shown in `README.md` and `test/contribution.js`).

### Proof of concept

#### Scenario 1: Cross-DAO Contribution NFT Hijacking & Reward Theft

1. A legitimate contributor submits a proposal on `dao1` (`virtualId = 1`) calling `ServiceNft.mint(1, descHash)` with description `"Model v1"`, producing `proposalId`.
2. Before the contributor's `ContributionNft.mint` transaction is executed, an attacker who holds `proposalThreshold` on `dao2` (`virtualId = 2`) calls `dao2.propose([serviceNft], [0], [abi.encodeWithSignature("mint(uint256,bytes32)", 1, descHash)], "Model v1")`.
3. Because `hashProposal` is identical across governors, `dao2` records `dao2.proposalProposer(proposalId) = attacker`.
4. The attacker calls `ContributionNft.mint(attacker, 2, coreId, uri, proposalId, 0, true, 0)`. The check `msg.sender == getAgentDAO(2).proposalProposer(proposalId)` succeeds, and `ContributionNft` mints `tokenId = proposalId` to `attacker`.
5. The legitimate contributor's `ContributionNft.mint` call reverts with `ERC721InvalidSender(address(0))` because `proposalId` is already minted.
6. If `dao1` executes the proposal, `ServiceNft.mint(1, descHash)` mints the service NFT for `proposalId` and `AgentRewardV2` allocates contributor rewards to `_serviceRewards[proposalId]`, which only `attacker` (as `ContributionNft.ownerOf(proposalId)`) can claim via `_claimServiceRewards(proposalId)`.

#### Scenario 2: Unauthorized `AgentToken` Minting via `Minter.mint`

1. The attacker (holding `proposalThreshold` in `dao1`, or using a defeated/unvoted proposal on `dao1`) calls `dao1.propose([serviceNft], [0], [abi.encodeWithSignature("mint(uint256,bytes32)", 2, descHash)], "Attack")`, generating `proposalId`.
2. The attacker calls `ContributionNft.mint(attacker, 1, coreId, uri, proposalId, 0, true, 0)`. This sets `_contributionVirtualId[proposalId] = 1` and `modelContributions[proposalId] = true`.
3. On `dao2` (`virtualId = 2`, where the attacker controls governance votes), the attacker proposes, votes for, and executes the exact same proposal `([serviceNft], [0], [abi.encodeWithSignature("mint(uint256,bytes32)", 2, descHash)], "Attack")`.
4. `ServiceNft.mint(2, descHash)` executes from `dao2`, minting `ServiceNft(proposalId)` and recording a positive `_impacts[proposalId]`.
5. The attacker calls `Minter.mint(proposalId)`. `Minter` reads `virtualId = IContributionNft(contributionNft).tokenVirtualId(proposalId)` (`1`) and `impact = IServiceNft(serviceNft).getImpact(proposalId)` (from `virtualId = 2`), transferring `virtualId = 1`'s `AgentToken` to the attacker without `dao1`'s approval.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:** Attackers can pre-emptively mint or hijack `ContributionNft` token IDs belonging to other DAOs, permanently DoS legitimate contribution NFT mints, claim contributor service rewards in `AgentRewardV2`, and drain `AgentToken`s from `Minter` for target virtual agents using impacts generated on attacker-controlled virtual agents.

### Remediation

Bind `proposalId` strictly to the target `virtualId` and require caller-provided `descHash` so `ContributionNft.mint` deterministically reconstructs and verifies the exact `ServiceNft.mint(virtualId, descHash)` proposal hash on `personaDAO`, and validate that the proposal is currently active/pending and that any referenced `parentId` or `datasetId` belongs to the same `virtualId`:

```solidity
    function mint(
        address to,
        uint256 virtualId,
        uint8 coreId,
        string memory newTokenURI,
        uint256 proposalId,
        uint256 parentId,
        bool isModel_,
        uint256 datasetId,
        bytes32 descHash
    ) external returns (uint256) {
        IGovernor personaDAO = getAgentDAO(virtualId);
        address serviceNft = IAgentNft(personaNft).getServiceNft();

        address[] memory targets = new address[](1);
        targets[0] = serviceNft;
        uint256[] memory values = new uint256[](1);
        values[0] = 0;
        bytes[] memory calldatas = new bytes[](1);
        calldatas[0] = abi.encodeWithSignature("mint(uint256,bytes32)", virtualId, descHash);

        uint256 expectedProposalId = personaDAO.hashProposal(targets, values, calldatas, descHash);
        require(proposalId == expectedProposalId, "Invalid proposalId for virtualId");
        require(
            msg.sender == personaDAO.proposalProposer(proposalId),
            "Only proposal proposer can mint Contribution NFT"
        );
        IGovernor.ProposalState currentState = personaDAO.state(proposalId);
        require(
            currentState == IGovernor.ProposalState.Pending ||
                currentState == IGovernor.ProposalState.Active ||
                currentState == IGovernor.ProposalState.Succeeded,
            "Invalid proposal state"
        );
        require(parentId != proposalId, "Cannot be parent of itself");
        if (parentId != 0) {
            require(_contributionVirtualId[parentId] == virtualId, "Invalid parent virtualId");
        }

        _mint(to, proposalId);
        _setTokenURI(proposalId, newTokenURI);
        _contributionVirtualId[proposalId] = virtualId;
        _parents[proposalId] = parentId;
        _children[parentId].push(proposalId);
        _cores[proposalId] = coreId;

        if (isModel_) {
            if (datasetId != 0) {
                require(
                    _contributionVirtualId[datasetId] == virtualId && !modelContributions[datasetId],
                    "Invalid datasetId"
                );
            }
            modelContributions[proposalId] = true;
            modelDatasets[proposalId] = datasetId;
        }

        emit NewContribution(proposalId, virtualId, parentId, datasetId);

        return proposalId;
    }
```

## Finding 3: Unbounded `_children` array growth in `ContributionNft.mint` enables permanent Denial of Service on `AgentRewardV2` reward claiming and querying

### Synopsis

`ContributionNft.mint` appends every newly minted `proposalId` to `_children[parentId]` without bounding fan-out or validating proposal acceptance or parent eligibility. A proposer can repeatedly attach child contributions to a target `parentId`, bloating `_children[parentId]` until `AgentRewardV2._claimServiceRewards` and `AgentRewardV2.getChildrenRewards` revert from gas exhaustion.

### Technical details

In `ContributionNft.mint` (`contracts/contribution/ContributionNft.sol`), lines 75–87:

```solidity
IGovernor personaDAO = getAgentDAO(virtualId);
require(
    msg.sender == personaDAO.proposalProposer(proposalId),
    "Only proposal proposer can mint Contribution NFT"
);
require(parentId != proposalId, "Cannot be parent of itself");

_mint(to, proposalId);
_setTokenURI(proposalId, newTokenURI);
_contributionVirtualId[proposalId] = virtualId;
_parents[proposalId] = parentId;
_children[parentId].push(proposalId);
_cores[proposalId] = coreId;
```

Whenever `mint` is called for a `proposalId`, it unconditionally pushes `proposalId` into the storage array `_children[parentId]` (line 86). There are several compounding weaknesses in this design:

1. **Unbounded Fan-Out on `_children[parentId]`:** There is no upper bound on the number of children that can reference a single `parentId`, nor any pagination mechanism when reading `_children`.
2. **No Validation of Proposal State or Parent Relationship:** `mint` only verifies that `msg.sender == personaDAO.proposalProposer(proposalId)` and `parentId != proposalId`. It does not require the proposal to have succeeded (unlike `ServiceNft.mint`, which is only called by the DAO upon proposal execution), nor does it require `parentId` to belong to the same `virtualId` or even be an accepted contribution. Any account capable of creating proposals in any `AgentDAO` (including the DAO of a low-cost or attacker-created virtual persona) can generate an arbitrary number of proposals with distinct descriptions and call `ContributionNft.mint` pointing `parentId` to a victim's `ContributionNft` token ID.
3. **Full-Array Memory Copy and Linear Iteration in `AgentRewardV2`:**
   - `ContributionNft.getChildren(uint256 tokenId)` copies the entire `_children[tokenId]` storage array into memory and returns it across an external call.
   - `AgentRewardV2._claimServiceRewards(uint256 nftId)` (lines 462–476) and `AgentRewardV2.getChildrenRewards(uint256 nftId)` (lines 380–390) fetch the full `uint256[] memory children = IContributionNft(contributionNft).getChildren(nftId)` array and iterate through every element `0 .. children.length - 1`, performing storage reads (`_serviceRewards[children[i]]`) on each iteration—even for child IDs that never became accepted `ServiceNft`s and have zero rewards.

Once `_children[nftId]` grows sufficiently large (either through adversarial spamming of proposals referencing `nftId` as `parentId` or organic high fan-out over time), the gas required to ABI-encode/decode `getChildren(nftId)` and iterate over the array exceeds the block gas limit. Consequently, the owner of `nftId` is permanently unable to execute `claimAllRewards` / `_claimServiceRewards` to withdraw either their own `serviceReward.amount` or any legitimate `childrenAmount`.

### Assumptions and preconditions

1. A target `ContributionNft` (`nftId`) exists and has accrued (or will accrue) service rewards or parent rewards in `AgentRewardV2`.
2. Either an attacker holds enough voting power (`>= proposalThreshold()`) in any `AgentDAO` instance (for example, as founder/staker of any virtual persona) to submit proposals, or a popular contribution organically accumulates a large number of child contributions over the protocol's lifetime.

### Proof of concept

1. Victim owns a valid `ContributionNft` with token ID `victimNftId` that has accrued service rewards in `AgentRewardV2._serviceRewards[victimNftId].amount`.
2. Attacker holds voting power above `proposalThreshold()` in an `AgentDAO` for `virtualId` (which can be any virtual persona's DAO, as `ContributionNft.mint` does not enforce `_contributionVirtualId[parentId] == virtualId`).
3. In a loop (batched across transactions), the attacker calls `agentDAO.propose(targets, values, calldatas, description_i)` with unique descriptions `description_i` to produce unique `proposalId_i` values.
4. For each `proposalId_i`, the attacker calls `ContributionNft.mint(attacker, virtualId, coreId, "", proposalId_i, victimNftId, false, 0)`.
5. Each call passes the `msg.sender == personaDAO.proposalProposer(proposalId_i)` check (line 77) and appends `proposalId_i` to `_children[victimNftId]` (line 86). None of these proposals need to pass or be voted on.
6. When the victim calls `AgentRewardV2.claimAllRewards([], [victimNftId])`, execution enters `_claimServiceRewards(victimNftId)`, which calls `IContributionNft(contributionNft).getChildren(victimNftId)` and loops over `_children[victimNftId]`.
7. Because `_children[victimNftId]` contains thousands of entries, memory expansion, ABI encoding/decoding, and per-element `SLOAD` operations in the loop exhaust the block gas limit, causing the transaction to revert and permanently locking the victim's service rewards.

### Impact assessment

- **Severity:** High / Medium (Permanent Denial of Service on reward claiming).
- **Direct Technical Effect:** Calling `AgentRewardV2.claimAllRewards` (for `victimNftId`), `AgentRewardV2.getTotalClaimableRewards`, and `AgentRewardV2.getChildrenRewards(victimNftId)` reverts due to Out-Of-Gas.
- **Business Implications:** Contributors cannot claim their earned `rewardToken` payouts for affected `ContributionNft` tokens (both direct service rewards and parent share rewards are locked together in `_claimServiceRewards` because direct rewards cannot be claimed without also executing the children loop).

### Remediation

Restrict child registration so that a contribution is only added to `_children[parentId]` when it is an accepted contribution belonging to the same `virtualId`, and bound the maximum number of children per parent (or replace pull-enumeration over `_children` with push-based parent reward accounting in `AgentRewardV2._distributeContributorRewards` using `getParentId(serviceId)` directly).

Within `ContributionNft.mint`, validate `parentId` and enforce a maximum child fan-out cap:

```solidity
    uint256 public constant MAX_CHILDREN = 100;

    function mint(
        address to,
        uint256 virtualId,
        uint8 coreId,
        string memory newTokenURI,
        uint256 proposalId,
        uint256 parentId,
        bool isModel_,
        uint256 datasetId
    ) external returns (uint256) {
        IGovernor personaDAO = getAgentDAO(virtualId);
        require(
            msg.sender == personaDAO.proposalProposer(proposalId),
            "Only proposal proposer can mint Contribution NFT"
        );
        require(parentId != proposalId, "Cannot be parent of itself");

        if (parentId != 0) {
            require(_ownerOf(parentId) != address(0), "Parent does not exist");
            require(_contributionVirtualId[parentId] == virtualId, "Parent virtualId mismatch");
            require(isAccepted(parentId), "Parent contribution not accepted");
            require(_children[parentId].length < MAX_CHILDREN, "Max children exceeded");
            _parents[proposalId] = parentId;
            _children[parentId].push(proposalId);
        }

        _mint(to, proposalId);
        _setTokenURI(proposalId, newTokenURI);
        _contributionVirtualId[proposalId] = virtualId;
        _cores[proposalId] = coreId;

        if (isModel_) {
            modelContributions[proposalId] = true;
            modelDatasets[proposalId] = datasetId;
        }

        emit NewContribution(proposalId, virtualId, parentId, datasetId);

        return proposalId;
    }
```

## Finding 4: Missing `parentId` Validation in `ContributionNft.mint` Breaks Lineage Integrity and Locks Parent Contributor Rewards

### Synopsis

`ContributionNft.mint` only checks `parentId != proposalId` and fails to validate that a non-zero `parentId` is an existing, accepted contribution belonging to the same virtual agent and core. This breaks parent referential integrity and lineage acyclicity, corrupts cross-agent parent-child mappings, and permanently locks `parentAmount` rewards in `AgentRewardV2`.

### Technical details

In `ContributionNft.mint` (lines 75–91), a proposer mints a `ContributionNft` corresponding to a governance proposal `proposalId` in `personaDAO`:

```solidity
require(parentId != proposalId, "Cannot be parent of itself");

_mint(to, proposalId);
_setTokenURI(proposalId, newTokenURI);
_contributionVirtualId[proposalId] = virtualId;
_parents[proposalId] = parentId;
_children[parentId].push(proposalId);
_cores[proposalId] = coreId;
```

The function performs only a trivial self-equality check (`parentId != proposalId`) and unconditionally records `_parents[proposalId] = parentId` and `_children[parentId].push(proposalId)` without validating `parentId` when `parentId != 0`. Specifically:

1. **Lack of Existence and Acceptance Validation (Reward Locking):**
   In `AgentRewardV2._distributeContributorRewards` (lines 296–301), parent rewards are separated whenever `getParentId(serviceId) != 0`:
   ```solidity
   parentAmount = contributionNftContract.getParentId(serviceId) == 0
       ? 0
       : ((impactAmount * parentShares) / DENOMINATOR);

   serviceReward.amount += impactAmount - parentAmount;
   serviceReward.parentAmount += parentAmount;
   ```
   To claim `serviceReward.parentAmount`, the owner of `parentId` must call `AgentRewardV2._claimServiceRewards(parentId)` (line 454), which requires `IERC721(contributionNft).ownerOf(parentId) == account`. If `parentId` is a non-existent token ID (e.g., an unminted ID or a typo), `ownerOf(parentId)` reverts with `ERC721NonexistentToken`. Consequently, `serviceReward.parentAmount` is deducted from the child contribution's rewards and permanently locked in `AgentRewardV2`.

2. **Bypass of Lineage Acyclicity:**
   Because `proposalId` in `GovernorUpgradeable` is deterministically computed via `hashProposal(targets, values, calldatas, descriptionHash)` and `mint` does not require `parentId` to already be minted (`_ownerOf(parentId) != address(0)`), a user can create two proposals `proposalA` and `proposalB` in `AgentDAO`, then call `ContributionNft.mint` for `proposalA` with `parentId = proposalB` and for `proposalB` with `parentId = proposalA`. Both calls pass `require(parentId != proposalId)`, creating a cyclic lineage (`proposalA <-> proposalB`).

3. **Missing Cross-Agent and Cross-Core Isolation and Unbounded `_children` Pollution:**
   `ContributionNft.mint` does not verify that `_contributionVirtualId[parentId] == virtualId` or `_cores[parentId] == coreId`. A proposer in any virtual agent DAO can attach their proposal as a child of an arbitrary `parentId` belonging to another virtual agent or core, polluting `_children[parentId]`. Because `AgentRewardV2.getChildrenRewards` (line 382) and `AgentRewardV2._claimServiceRewards` (line 462) iterate through the entire `getChildren(nftId)` array, repeatedly appending unaccepted or cross-agent proposals to a victim's `_children[nftId]` array increases gas consumption and can cause reward queries and claims for `nftId` to run out of gas. Additionally, when `parentId == 0`, `_children[0].push(proposalId)` needlessly appends every root contribution to `_children[0]`.

### Assumptions and preconditions

- A proposer has sufficient voting power (at least `proposalThreshold()`) or is the `ContributionNft` admin in any `AgentDAO` instance to create a proposal and become `personaDAO.proposalProposer(proposalId)`.
- For reward locking in `AgentRewardV2`, `settings.parentShares > 0` and the child contribution proposal succeeds and is executed to mint a `ServiceNft` with non-zero impact.

### Proof of concept

#### Scenario 1: Creating a Cyclic Parent-Child Lineage and Locking Parent Rewards

1. A proposer submits two proposals `proposalA` and `proposalB` to `AgentDAO.propose(...)`. Both `proposalA` and `proposalB` are now registered in `AgentDAO` with `proposalProposer(proposalA) == proposer` and `proposalProposer(proposalB) == proposer`.
2. The proposer calls `ContributionNft.mint(proposer, virtualId, coreId, uriA, proposalA, proposalB, true, 0)`.
   - `proposalB != proposalA` holds, so `require(parentId != proposalId)` passes even though `proposalB` has not been minted yet.
   - `_parents[proposalA]` is set to `proposalB`, and `proposalA` is pushed to `_children[proposalB]`.
3. The proposer calls `ContributionNft.mint(proposer, virtualId, coreId, uriB, proposalB, proposalA, true, 0)`.
   - `proposalA != proposalB` holds, so `require(parentId != proposalId)` passes.
   - `_parents[proposalB]` is set to `proposalA`, and `proposalB` is pushed to `_children[proposalA]`, forming a 2-node cycle (`proposalA <-> proposalB`).
4. Alternatively, if `proposalA` is minted with a non-existent `parentId = 999999` (and `proposalA` is later accepted and executed into `ServiceNft`), `AgentRewardV2._distributeContributorRewards` deducts `parentAmount = (impactAmount * parentShares) / DENOMINATOR` from ` _serviceRewards[proposalA].amount` and credits `_serviceRewards[proposalA].parentAmount`.
5. Because token `999999` does not exist in `ContributionNft`, any call to `_claimServiceRewards(999999)` reverts at `IERC721(contributionNft).ownerOf(999999)`, permanently locking `parentAmount` in `AgentRewardV2`.

#### Scenario 2: Cross-Agent `_children` Array Pollution

1. Victim owns a valid `ContributionNft` with ID `victimNftId` in `virtualId = 1`.
2. An attacker who controls staking tokens in a separate agent (`virtualId = 2`) creates proposals `p_1, p_2, ..., p_n` in `virtualId = 2`'s `AgentDAO`.
3. For each proposal `p_i`, the attacker calls `ContributionNft.mint(attacker, 2, coreId, "", p_i, victimNftId, false, 0)`.
4. Each call appends `p_i` to `_children[victimNftId]`, linking contributions from `virtualId = 2` to `victimNftId` in `virtualId = 1` and inflating the loop in `AgentRewardV2._claimServiceRewards(victimNftId)` and `AgentRewardV2.getChildrenRewards(victimNftId)`.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:**
  1. Non-existent or unminted `parentId` values cause `parentAmount` shares in `AgentRewardV2` to be deducted from contributors and permanently locked in the reward contract.
  2. Proposers can construct cyclic contribution graphs (`A -> B -> A`), violating the acyclicity invariant intended by `require(parentId != proposalId)`.
  3. Cross-agent and unaccepted proposals can be attached to any `parentId`, corrupting lineage metadata and enabling gas griefing on `getChildrenRewards` and `_claimServiceRewards`.

### Remediation

Validate that any non-zero `parentId` already exists (which also strictly prevents cycles since `proposalId` is newly minted in the transaction), belongs to the same `virtualId` and `coreId`, and has been accepted as a valid service (or minted in `ServiceNft`). Additionally, only push to `_children[parentId]` when `parentId != 0`.

```solidity
        IGovernor personaDAO = getAgentDAO(virtualId);
        require(
            msg.sender == personaDAO.proposalProposer(proposalId),
            "Only proposal proposer can mint Contribution NFT"
        );
        require(parentId != proposalId, "Cannot be parent of itself");

        if (parentId != 0) {
            require(_ownerOf(parentId) != address(0), "Parent contribution does not exist");
            require(_contributionVirtualId[parentId] == virtualId, "Parent belongs to different virtualId");
            require(_cores[parentId] == coreId, "Parent belongs to different coreId");
            require(isAccepted(parentId), "Parent contribution is not accepted");
            _parents[proposalId] = parentId;
            _children[parentId].push(proposalId);
        }

        _mint(to, proposalId);
        _setTokenURI(proposalId, newTokenURI);
        _contributionVirtualId[proposalId] = virtualId;
        _cores[proposalId] = coreId;

        if (isModel_) {
            modelContributions[proposalId] = true;
            modelDatasets[proposalId] = datasetId;
        }
```

## Finding 5: Missing Cross-Contract Identity Validation in `ServiceNft.mint` Allows Cross-Agent Reward Minting and Unauthorized Service Registration

### Synopsis

`ServiceNft.mint` derives `proposalId` via `personaDAO.hashProposal` and queries `ContributionNft` and `AgentDAO` without verifying that the `ContributionNft` exists and belongs to `virtualId` (`tokenVirtualId(proposalId) == virtualId`) or that `proposalId` is the proposal actually executed by `personaDAO`, enabling unauthorized `AgentToken` minting and state corruption.

### Technical details

In `contracts/contribution/ServiceNft.sol`, `mint(uint256 virtualId, bytes32 descHash)` restricts the caller to `info.dao` (line 60) and reconstructs `proposalId` by hashing a single-action call to `mint(virtualId, descHash)` with `descHash` as the proposal description hash (`70`). It then immediately mints the `ServiceNft` (`71`), assigns `_cores[proposalId]` (`72`), fetches `_maturities[proposalId]` (`74`), checks `isModel(proposalId)` (`76`), and updates impact and core service/dataset mappings (`77-83`).

This logic lacks cross-contract identity integrity checks across `AgentDAO`, `ContributionNft`, and `ServiceNft`:

1. **Missing `ContributionNft` Agent Binding and Existence Check (`tokenVirtualId(proposalId) == virtualId`)**:
   `GovernorUpgradeable.hashProposal` computes `uint256(keccak256(abi.encode(targets, values, calldatas, descriptionHash)))` without including the governor contract address (`info.dao`). Because `ServiceNft`, `ContributionNft`, and `Minter` are shared across all virtual personas (`virtualId`s), submitting a proposal with calldata `ServiceNft.mint(virtualIdB, descHash)` and description `desc` to both Victim Agent A's DAO (`virtualIdA`) and Attacker Agent B's DAO (`virtualIdB`) produces the exact same `proposalId`.
   - The proposer can mint the `ContributionNft` for `proposalId` under `virtualIdA` (since they are the proposer of `proposalId` in `virtualIdA`'s DAO), setting `IContributionNft(contributionNft).tokenVirtualId(proposalId) = virtualIdA`.
   - Without `virtualIdA`'s DAO ever voting for or executing the proposal, the proposer can vote for and execute `proposalId` in `virtualIdB`'s DAO, which invokes `ServiceNft.mint(virtualIdB, descHash)`.
   - Because `ServiceNft.mint` (`70-83`) never verifies `IContributionNft(contributionNft).tokenVirtualId(proposalId) == virtualId`, it mints the `ServiceNft` for `proposalId` and records `_maturities[proposalId]` and `_impacts[proposalId]` using `virtualIdB`'s DAO.
   - Subsequently, `Minter.mint(proposalId)` reads `virtualId = IContributionNft(contributionNft).tokenVirtualId(proposalId)` (which resolves to `virtualIdA`) and `impact = IServiceNft(serviceNft).getImpact(proposalId)` (which was set by `virtualIdB`'s DAO), transferring `virtualIdA`'s `AgentToken` to the `ContributionNft` owner.
   - Conversely, a malicious actor can front-run a legitimate proposer's `ContributionNft.mint` on `virtualIdA` by proposing the identical parameters on `virtualIdB`'s DAO and minting `ContributionNft(proposalId)` under `virtualIdB`, hijacking ownership and virtual persona binding of `proposalId` before `virtualIdA`'s DAO executes `ServiceNft.mint(virtualIdA, descHash)`.

2. **Missing Executed Proposal Verification (`personaDAO.state(proposalId)`) and Linked Asset Validation**:
   `ServiceNft.mint` assumes `descHash` passed in its calldata matches the `descriptionHash` of the currently executing proposal in `personaDAO`. However, `GovernorUpgradeable.propose` hashes `keccak256(bytes(description))` independently of the `descHash` argument encoded inside `calldatas[0]`. A proposal executed in `personaDAO` with outer description `D_outer` (or a multi-action proposal) can invoke `ServiceNft.mint(virtualId, descHash_target)` where `descHash_target` corresponds to a different, defeated or unexecuted proposal `proposalId_target`. Because `AgentDAO.getMaturity(proposalId_target)` (`contracts/virtualPersona/AgentDAO.sol:195-198`) divides `_proposalMaturities[proposalId_target]` only by `forVotes` without checking `againstVotes` or proposal state, `ServiceNft.mint` will mint the service and compute impact for the defeated proposal `proposalId_target`. Furthermore, if `isModel` is true and `getDatasetId(proposalId)` is non-zero, `ServiceNft.mint` (`79`) invokes `updateImpact` without verifying that the linked `datasetId` is an existing dataset `ServiceNft` belonging to the same `virtualId` and core.

### Assumptions and preconditions

- At least two virtual personas exist (e.g., a target agent `virtualIdA` and an attacker-controlled or separate agent `virtualIdB`), or a proposal is executed on `info.dao` whose outer proposal parameters differ from the single-call `(virtualId, descHash)` assumed by `ServiceNft.mint`.
- The proposer has sufficient voting power to submit a proposal (`propose`) on `virtualIdA`'s `AgentDAO` (to satisfy the proposer check when minting `ContributionNft`), without needing `virtualIdA`'s validators to approve or execute the proposal.

### Proof of concept

1. Consider Victim Agent `virtualId = 1` (with `DAO_1` and `AgentToken_1`) and Agent `virtualId = 2` (with `DAO_2` where a user controls governance voting).
2. The user constructs `mintCalldata = abi.encodeWithSignature("mint(uint256,bytes32)", 2, descHash)` where `descHash = keccak256(bytes(desc))`.
3. The user calls `DAO_1.propose([address(serviceNft)], [0], [mintCalldata], desc)` and `DAO_2.propose([address(serviceNft)], [0], [mintCalldata], desc)`. Because `hashProposal` only hashes `(targets, values, calldatas, descHash)`, both `DAO_1` and `DAO_2` return the exact same `proposalId`.
4. The user mints the `ContributionNft` for `proposalId` associated with `virtualId = 1` (`isModel = true`). Since the user is the proposer of `proposalId` in `DAO_1`, `ContributionNft` records `_contributionVirtualId[proposalId] = 1` and `modelContributions[proposalId] = true`.
5. In `DAO_1`, the proposal is never voted on or is defeated. In `DAO_2`, the user casts a `For` vote with battle results `[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]` and executes the proposal after the voting period.
6. `DAO_2` calls `ServiceNft.mint(2, descHash)`. Inside `ServiceNft.mint`:
   - `require(_msgSender() == info.dao)` passes because `virtualId` is `2` and `_msgSender()` is `DAO_2`.
   - Line 70 computes `proposalId`, which matches the `ContributionNft` registered under `virtualId = 1`.
   - Lines 71-80 mint the `ServiceNft` for `proposalId`, set `_maturities[proposalId]` from `DAO_2`, and compute `_impacts[proposalId]` via `updateImpact(2, proposalId)`.
7. The user invokes `Minter.mint(proposalId)`:
   - `Minter.mint` queries `IContributionNft(contributionNft).tokenVirtualId(proposalId)`, receiving `virtualId = 1`.
   - `Minter.mint` queries `IServiceNft(serviceNft).getImpact(proposalId)`, receiving the impact set via `DAO_2`.
   - `Minter.mint` transfers `AgentToken_1` (belonging to `virtualId = 1`) to the user without `DAO_1` ever approving the contribution.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Cross-agent identity confusion between `ContributionNft`, `AgentDAO`, and `ServiceNft`, allowing `ServiceNft` records and impact scores generated in one DAO (or from a mismatched/unapproved proposal ID) to be claimed against another agent's `ContributionNft` and token reserves.
- **Business Impact**: Unauthorized minting of `AgentToken` rewards from `Minter`, corruption of `_coreServices` and `_coreDatasets` state in `ServiceNft`, and distortion of contributor reward distributions in `AgentRewardV2`.

### Remediation

In `ServiceNft.mint`, validate that:
1. `proposalId` is currently in the `Executed` state in `personaDAO` (since OpenZeppelin's `GovernorUpgradeable.execute` sets `executed = true` prior to `_executeOperations`).
2. The `ContributionNft` for `proposalId` exists and is bound to `virtualId` (`IContributionNft(contributionNft).tokenVirtualId(proposalId) == virtualId`).
3. If `isModel` is true and a `datasetId` is linked, the `datasetId` belongs to the same `virtualId`, has the same core type, is not a model, and has already been minted as a `ServiceNft`.

```solidity
        uint256 proposalId = personaDAO.hashProposal(targets, values, calldatas, descHash);
        require(personaDAO.state(proposalId) == IGovernor.ProposalState.Executed, "Proposal not executed");
        require(
            IContributionNft(contributionNft).tokenVirtualId(proposalId) == virtualId,
            "Invalid contribution virtualId"
        );
        _mint(info.tba, proposalId);
        uint8 core = IContributionNft(contributionNft).getCore(proposalId);
        _cores[proposalId] = core;
        // Calculate maturity
        _maturities[proposalId] = IAgentDAO(info.dao).getMaturity(proposalId);

        bool isModel = IContributionNft(contributionNft).isModel(proposalId);
        if (isModel) {
            uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);
            if (datasetId > 0) {
                _requireOwned(datasetId);
                require(
                    IContributionNft(contributionNft).tokenVirtualId(datasetId) == virtualId &&
                        !IContributionNft(contributionNft).isModel(datasetId) &&
                        _cores[datasetId] == core,
                    "Invalid dataset"
                );
            }
            emit CoreServiceUpdated(virtualId, core, proposalId);
            updateImpact(virtualId, proposalId);
            _coreServices[virtualId][core] = proposalId;
        } else {
            _coreDatasets[virtualId][core].push(proposalId);
        }

        emit NewService(proposalId, core, _maturities[proposalId], _impacts[proposalId], isModel);
```

## Finding 6: Missing Cross-Contract Existence, Type, and Scope Validation for Linked Dataset and Core References in `ServiceNft.mint`

### Synopsis

During model service minting, `ServiceNft.mint` invokes `updateImpact` and records core assignments without validating that the linked `datasetId` exists as a minted dataset `ServiceNft` within the same `virtualId` and `coreId`, enabling cross-agent/cross-core state corruption, reward misdirection, and `Minter.mint` denial of service.

### Technical details

When `ServiceNft.mint(uint256 virtualId, bytes32 descHash)` executes for a model contribution (`isModel == true` at `76`), it assigns `_cores[proposalId]` (`72`), `_maturities[proposalId]` (`74`), and calls `updateImpact(virtualId, proposalId)` (`79`) before updating `_coreServices[virtualId][_cores[proposalId]]` (`80`).

Inside `updateImpact(virtualId, proposalId)`, the contract queries `uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId)` (line 97) and, if `datasetId > 0`, splits `rawImpact` between `proposalId` and `datasetId`, directly mutating `_impacts[datasetId]` and `_maturities[datasetId]` (lines 100–105). However, `ServiceNft.mint` and `updateImpact` fail to enforce cross-contract identity and linkage invariants on `datasetId` and `_cores[proposalId]`:

1. **Missing `ServiceNft` Existence Guarantee (`_ownerOf(datasetId) != address(0)`)**:
   Neither `ServiceNft.mint` nor `updateImpact` verifies that `datasetId` has been approved and minted as a `ServiceNft`.
   - If `datasetId` is not a minted `ServiceNft`, `updateImpact` still deducts `_impacts[datasetId]` from `_impacts[proposalId]` and writes to `_impacts[datasetId]`. When `Minter.mint(proposalId)` is subsequently called, `IServiceNft(serviceNft).getImpact(datasetId)` (line 159 of `Minter.sol`) executes `_requireOwned(datasetId)` (line 121 of `ServiceNft.sol`), reverting with `ERC721NonexistentToken(datasetId)` and blocking token minting for the model contributor and IP vault.
   - If the dataset `ServiceNft` is minted *after* the model `ServiceNft`, `ServiceNft.mint` executes `_maturities[proposalId] = IAgentDAO(info.dao).getMaturity(proposalId)` (`74`), overwriting the dataset's previously assigned `_maturities[datasetId]` with `0` (because `AgentDAO._updateMaturity` ignores non-model proposals).

2. **Missing Asset Type and Single-Use Guarantee (`!isModel(datasetId)`)**:
   There is no check ensuring `datasetId` represents a dataset rather than a model (`!IContributionNft(contributionNft).isModel(datasetId)`), nor that `datasetId` has not already been consumed by a prior model.
   - If `datasetId` points to an existing model `ServiceNft`, `updateImpact` overwrites that model's `_impacts[datasetId]` and `_maturities[datasetId]`, corrupting its ELO baseline or its pending reward calculations in `Minter.mint` and `AgentRewardV2`.
   - If multiple models reference the same `datasetId`, `Minter.mint` (which only tracks `_mintedNfts[nftId]` for the model ID) repeatedly mints dataset rewards for the same `datasetId`.

3. **Cross-Agent and Cross-Core Reference Contamination**:
   Neither `ServiceNft.mint` nor `updateImpact` verifies that `IContributionNft(contributionNft).tokenVirtualId(datasetId) == virtualId`, that `_cores[datasetId] == _cores[proposalId]`, or that `_cores[proposalId]` belongs to `info.coreTypes`.
   - Referencing a `datasetId` from another agent (`virtualId`) or core contaminates `_impacts[datasetId]` across agents/cores. In `Minter.mint`, Agent A's tokens are paid to Agent B's dataset owner; in `AgentRewardV2._distributeContributorRewards`, Agent B's service list includes `datasetId` and distributes Agent B's rewards using the impact assigned by Agent A (or skews another core's `_rewardImpacts` denominator).

### Assumptions and preconditions

- A model contribution proposal is created, linked to a `datasetId` in `ContributionNft`, and executed by the agent's `personaDAO` via `ServiceNft.mint`.
- The linked `datasetId` is either not yet minted in `ServiceNft`, minted after the model in `ServiceNft`, a model token ID (`isModel(datasetId) == true`), already used by another model, or belongs to a different `virtualId` or `coreId`.

### Proof of concept

1. **Cross-Agent / Cross-Type Contamination**:
   - Suppose Dataset $D_B$ (or Model $M_B$) is an already-minted `ServiceNft` belonging to Agent 2 (`virtualId = 2`) or Core 1.
   - A contributor submits Model $M_A$ on Agent 1 (`virtualId = 1`, Core 0) with `datasetId = D_B` (or $M_B$).
   - When Agent 1's DAO executes `ServiceNft.mint(1, descHash)`, `79` calls `updateImpact(1, M_A)` with `rawImpact = 200` and `datasetImpactWeight = 7000` (70%).
   - `updateImpact` overwrites `_impacts[D_B] = 140` and `_maturities[D_B] = _maturities[M_A]` even though $D_B$ belongs to Agent 2.
   - Calling `Minter.mint(M_A)` mints Agent 1's tokens to the owner of $D_B$, while `AgentRewardV2._distributeContributorRewards` for Agent 2 reads `serviceNftContract.getImpact(D_B) == 140` and allocates Agent 2's contributor rewards to $D_B$ based on Agent 1's model impact.

2. **Unminted Dataset DoS in `Minter.mint` and Maturity Overwrite**:
   - A contributor creates Dataset $D_1$ (whose DAO proposal is not yet executed) and Model $M_1$ referencing `datasetId = D_1`.
   - $M_1$'s proposal executes `ServiceNft.mint`, which calls `updateImpact` (`79`) and sets `_impacts[D_1] = 140`, `_impacts[M_1] = 60`, and `_maturities[D_1] = 100`.
   - Calling `Minter.mint(M_1)` reverts at `IServiceNft(serviceNft).getImpact(D_1)` with `ERC721NonexistentToken(D_1)` because $D_1$ has not been minted in `ServiceNft`.
   - If $D_1$'s proposal is subsequently executed via `ServiceNft.mint`, line `74` sets `_maturities[D_1] = IAgentDAO(info.dao).getMaturity(D_1) = 0`, wiping out the maturity score assigned during `updateImpact`.

### Impact assessment

- **Severity**: High.
- **Technical Impact**: Arbitrary corruption of `_impacts` and `_maturities` for existing `ServiceNft` tokens across agents and cores, permanent or temporary revert (`ERC721NonexistentToken`) in `Minter.mint` for models linked to unminted datasets, and loss of dataset maturity when datasets are minted after models.
- **Business Impact**: Misallocation of `AgentToken` emissions in `Minter.mint` and ongoing reward distributions in `AgentRewardV2._distributeContributorRewards`, including double-minting of dataset rewards when multiple models link the same `datasetId`.

### Remediation

Validate in `ServiceNft.mint` (and restrict `updateImpact` to internal calls) that `_cores[proposalId]` belongs to `info.coreTypes`, and that any non-zero `datasetId` linked to a model contribution is already minted as a `ServiceNft`, is not a model (`!isModel(datasetId)`), belongs to the same `virtualId` and `coreId`, and has not already had its impact assigned. Additionally, only query `IAgentDAO(info.dao).getMaturity(proposalId)` when `isModel` is true.

```solidity
        uint256 proposalId = personaDAO.hashProposal(targets, values, calldatas, descHash);
        _mint(info.tba, proposalId);
        uint8 coreId = IContributionNft(contributionNft).getCore(proposalId);
        bool validCore = false;
        for (uint256 i = 0; i < info.coreTypes.length; i++) {
            if (info.coreTypes[i] == coreId) {
                validCore = true;
                break;
            }
        }
        require(validCore, "Invalid core type");
        _cores[proposalId] = coreId;

        bool isModel = IContributionNft(contributionNft).isModel(proposalId);
        if (isModel) {
            _maturities[proposalId] = IAgentDAO(info.dao).getMaturity(proposalId);
            uint256 datasetId = IContributionNft(contributionNft).getDatasetId(proposalId);
            if (datasetId > 0) {
                _requireOwned(datasetId);
                require(!IContributionNft(contributionNft).isModel(datasetId), "Dataset cannot be a model");
                require(IContributionNft(contributionNft).tokenVirtualId(datasetId) == virtualId, "Dataset virtualId mismatch");
                require(_cores[datasetId] == coreId, "Dataset core mismatch");
                require(_impacts[datasetId] == 0 && _maturities[datasetId] == 0, "Dataset already used");
            }
            emit CoreServiceUpdated(virtualId, coreId, proposalId);
            updateImpact(virtualId, proposalId);
            _coreServices[virtualId][coreId] = proposalId;
        } else {
            _coreDatasets[virtualId][coreId].push(proposalId);
        }

        emit NewService(proposalId, coreId, _maturities[proposalId], _impacts[proposalId], isModel);
```

## Finding 7: Unconditional Core Service Overwrite in `ServiceNft.mint` Allows Repeated Impact Harvesting Across Non-Monotonic Model Maturities

### Synopsis

In `ServiceNft.mint`, `_coreServices[virtualId][_cores[proposalId]]` is unconditionally overwritten whenever a model `ServiceNft` is minted, even if the new model's maturity is lower than the current core service's maturity. This lowers the maturity baseline and enables repeated impact harvesting over the same maturity range, inflating token minting and reward distribution.

### Technical details

In `ServiceNft.sol`, each agent core tracks its active benchmark model in `_coreServices[virtualId][coreId]`. When a model contribution proposal is executed by the `AgentDAO`, `ServiceNft.mint` computes the proposal's maturity and updates the service impact and core service pointer (lines 74–80):

```solidity
_maturities[proposalId] = IAgentDAO(info.dao).getMaturity(proposalId);

bool isModel = IContributionNft(contributionNft).isModel(proposalId);
if (isModel) {
    emit CoreServiceUpdated(virtualId, _cores[proposalId], proposalId);
    updateImpact(virtualId, proposalId);
    _coreServices[virtualId][_cores[proposalId]] = proposalId;
}
```

Inside `updateImpact(virtualId, proposalId)` (lines 93–96), the marginal impact of the new model is calculated against `prevServiceId = _coreServices[virtualId][_cores[proposalId]]`:

```solidity
uint256 prevServiceId = _coreServices[virtualId][_cores[proposalId]];
uint256 rawImpact = (_maturities[proposalId] > _maturities[prevServiceId])
    ? _maturities[proposalId] - _maturities[prevServiceId]
    : 0;
```

While `rawImpact` is clamped to `0` when `_maturities[proposalId] <= _maturities[prevServiceId]`, line 80 of `ServiceNft.mint` still unconditionally overwrites `_coreServices[virtualId][_cores[proposalId]]` with `proposalId`.

A new model proposal can have `_maturities[proposalId] < _maturities[prevServiceId]` in two scenarios:
1. **Concurrent proposals:** Two model proposals `P1` and `P2` for the same `(virtualId, core)` are voted on while the active `coreService` has baseline maturity `M0` (e.g., `100`). `P1` receives high ELO ratings (e.g., maturity `315`) and `P2` receives lower ELO ratings (e.g., maturity `115`). If `P1` executes first (`impact = 315`, `_coreServices = P1`) and `P2` executes second (`impact = 0`), `_coreServices` is overwritten with `P2` (maturity `115`).
2. **Negative ELO delta in `EloCalculator.battleElo`:** In `AgentDAO._calcMaturity`, `battleElo(maturity, votes)` returns `currentRating + eloA - 1000`. If validators submit battle results where the candidate model loses more battles than it wins (`eloA < 1000`) while casting `For` votes (`support = 1`), the resulting maturity is strictly less than the previous `coreService` maturity.

Once `_coreServices[virtualId][core]` is overwritten with a lower-maturity service `P2`, both `ServiceNft.updateImpact` and `AgentDAO._calcMaturity` use `P2`'s lower maturity as the baseline for the next model `P3`. When `P3` achieves maturity `315`, `updateImpact` awards `315 - 115 = 200` impact again for the already-rewarded `[115, 315]` maturity interval. This breaks impact conservation ($\sum \text{impact} \le \max \text{maturity}$), allowing the same maturity gains to be minted repeatedly via `Minter.mint` and counted repeatedly in `AgentRewardV2._distributeContributorRewards`.

### Assumptions and preconditions

1. An agent (`virtualId`) and its corresponding `AgentDAO` are deployed and active.
2. At least two model contribution proposals for the same `(virtualId, core)` are either created/voted concurrently with different maturities or voted such that a subsequent proposal has a lower maturity than the currently active `_coreServices[virtualId][core]`.
3. The proposals pass governance voting and are executed via `AgentDAO.execute`, calling `ServiceNft.mint`.

### Proof of concept

1. **Initial Model (`P1`) Execution:**
   - For `virtualId = 1`, `core = 0`, initially `_coreServices[1][0] == 0`.
   - Model proposal `P1` is proposed and voted on as the first service (`_calcMaturity` returns `100`).
   - `P1` is executed (`ServiceNft.mint`): `_maturities[P1] = 100`, `_impacts[P1] = 100 - 0 = 100`, and `_coreServices[1][0] = P1`.
2. **Concurrent or Subsequent Model Proposals (`P2` and `P3`):**
   - While `_coreServices[1][0] == P1` (maturity `100`), two model proposals `P2` and `P3` are submitted and voted on:
     - `P2` wins all 10 battles (`votes = [1,1,1,1,1,1,1,1,1,1]`), so `battleElo(100, votes)` yields `_maturities[P2] = 315`.
     - `P3` ties its battles (`votes = [3,3,3,3,3,3,3,3,3,3]`), so `battleElo(100, votes)` yields `_maturities[P3] = 100`.
3. **Executing `P2` then `P3` Resets the Maturity Baseline:**
   - `P2` is executed first: `updateImpact` computes `rawImpact = _maturities[P2] - _maturities[P1] = 315 - 100 = 215`. `_coreServices[1][0]` is updated to `P2` (maturity `315`). `Minter.mint(P2)` mints `215 * finalImpactMultiplier` tokens.
   - `P3` is executed second: `updateImpact` compares `_maturities[P3]` (`100`) against `_maturities[P2]` (`315`), setting `_impacts[P3] = 0`. However, line 80 of `ServiceNft.mint` overwrites `_coreServices[1][0] = P3` (maturity `100`).
4. **Re-harvesting Impact (`P4`):**
   - Model proposal `P4` is now submitted and voted on with all wins against `coreService = P3` (maturity `100`), achieving `_maturities[P4] = 315`.
   - When `P4` is executed, `updateImpact` computes `rawImpact = _maturities[P4] - _maturities[P3] = 315 - 100 = 215`.
   - `Minter.mint(P4)` mints another `215 * finalImpactMultiplier` tokens and `AgentRewardV2._distributeContributorRewards` allocates another `215` impact points to `P4`, even though the core's peak maturity never exceeded `315`.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Breaks the monotonicity of `_coreServices` maturity and violates impact conservation. The cumulative impact across model services for a core can grow without bound even though `AgentDAO.getMaturity` caps individual service maturity at `10,000`.
- **Business Impact:** Unbacked inflation of the agent's ERC-20 token supply via `Minter.mint` (draining the `Minter` contract's token reserves) and dilution of honest contributors' ongoing reward shares in `AgentRewardV2._distributeContributorRewards`.

### Remediation

Update `_coreServices[virtualId][_cores[proposalId]]` (and emit `CoreServiceUpdated`) only when the new model strictly improves upon the current core service's maturity, or revert in `ServiceNft.mint` if a model contribution does not increase maturity.

```solidity
        uint256 proposalId = personaDAO.hashProposal(targets, values, calldatas, descHash);
        _mint(info.tba, proposalId);
        _cores[proposalId] = IContributionNft(contributionNft).getCore(proposalId);
        // Calculate maturity
        _maturities[proposalId] = IAgentDAO(info.dao).getMaturity(proposalId);

        bool isModel = IContributionNft(contributionNft).isModel(proposalId);
        if (isModel) {
            uint8 core = _cores[proposalId];
            uint256 prevServiceId = _coreServices[virtualId][core];
            require(_maturities[proposalId] > _maturities[prevServiceId], "Maturity not improved");
            emit CoreServiceUpdated(virtualId, core, proposalId);
            updateImpact(virtualId, proposalId);
            _coreServices[virtualId][core] = proposalId;
        } else {
            _coreDatasets[virtualId][_cores[proposalId]].push(proposalId);
        }

        emit NewService(proposalId, _cores[proposalId], _maturities[proposalId], _impacts[proposalId], isModel);
```

Also ensure `updateImpact` is marked `internal` or `private` so it cannot be invoked externally outside of `mint`.

## Finding 8: Missing Access Control in `ServiceNft.updateImpact` Allows Arbitrary Impact Manipulation

### Synopsis

`ServiceNft.updateImpact` is declared `public` without access control instead of `internal` or `private`. Any external caller can invoke it with arbitrary `virtualId` and `proposalId` parameters to zero out or artificially inflate service and dataset impact scores, corrupting token minting in `Minter.mint` and reward distribution in `AgentRewardV2`.

### Technical details

In `contracts/contribution/ServiceNft.sol`, `mint` is gated to the Virtual Persona's DAO (`require(_msgSender() == info.dao, "Caller is not VIRTUAL DAO")`). When a model contribution proposal is executed by the DAO, `mint` calculates the new service's impact by calling `updateImpact(virtualId, proposalId)` at line 79 before updating `_coreServices[virtualId][_cores[proposalId]] = proposalId` at line 80.

However, `updateImpact` at line 90 is declared `public` with no access control or validation that `proposalId` belongs to `virtualId`:

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

This creates two primary attack vectors for any unprivileged caller:

1. **Inflating Impact via Cross-Agent `virtualId` Spoofing:**
   Normally, `rawImpact` is the incremental maturity gain over the previous model service for that virtual agent and core (`_maturities[proposalId] - _maturities[prevServiceId]`). By calling `updateImpact(uninitializedVirtualId, proposalId)` where `_coreServices[uninitializedVirtualId][_cores[proposalId]] == 0`, `prevServiceId` resolves to `0` (with `_maturities[0] == 0`). Consequently, `rawImpact` is set to the full cumulative `_maturities[proposalId]` rather than the marginal delta. When `Minter.mint(proposalId)` (lines 153-160 of `Minter.sol`) or `AgentRewardV2._distributeContributorRewards` (lines 271-280 of `AgentRewardV2.sol`) reads `IServiceNft(serviceNft).getImpact(...)`, the model and dataset contributors receive inflated `AgentToken` mints and contributor reward shares.

2. **Zeroing Out Impact via Post-Mint Re-execution:**
   Immediately after `ServiceNft.mint` completes, `_coreServices[virtualId][_cores[proposalId]]` equals `proposalId` (and later equals newer service IDs with even higher maturity). Calling `updateImpact(virtualId, proposalId)` causes `_maturities[proposalId] > _maturities[prevServiceId]` to evaluate to `false`, setting `rawImpact = 0`. This overwrites `_impacts[proposalId]` and `_impacts[datasetId]` to `0` prior to `Minter.mint(proposalId)` or `AgentRewardV2.distributeRewardsForAgents`, denying the model and dataset contributors their minted tokens and rewards. Also, if multiple models reference the same dataset or if `updateImpact` is called with an older `proposalId` that used the same `datasetId`, `_impacts[datasetId]` and `_maturities[datasetId]` are overwritten.

### Assumptions and preconditions

- At least one model `ServiceNft` has been minted via `ServiceNft.mint` (so `_maturities[proposalId] > 0`).
- `Minter.mint(proposalId)` and/or `AgentRewardV2._distributeContributorRewards` has not yet been executed for `proposalId` (Wait: even after `AgentRewardV2` records `serviceReward.impact` once, any unrecorded service or unminted model NFT in `Minter.mint` remains vulnerable).

### Proof of concept

1. Suppose Virtual Agent `virtualId = 1` has an existing core `0` service `serviceId_1` with `_maturities[serviceId_1] = 300`.
2. A new model contribution `proposalId_2` is approved and executed by the DAO for `virtualId = 1`, core `0`, achieving `_maturities[proposalId_2] = 350`.
3. During `ServiceNft.mint(1, descHash)`:
   - `updateImpact(1, proposalId_2)` sets `rawImpact = 350 - 300 = 50`.
   - `_coreServices[1][0]` is updated to `proposalId_2`.
4. **Scenario A (Griefing / Denial of Rewards):**
   - Before `Minter.mint(proposalId_2)` is called, an arbitrary user calls `ServiceNft.updateImpact(1, proposalId_2)`.
   - `prevServiceId = _coreServices[1][0] = proposalId_2`.
   - Since `_maturities[proposalId_2] > _maturities[proposalId_2]` is `false`, `rawImpact` becomes `0`, setting `_impacts[proposalId_2] = 0` (and `_impacts[datasetId] = 0`).
   - When `Minter.mint(proposalId_2)` is called, `amount` and `dataAmount` are `0`, permanently burning the contributor's minting right (`_mintedNfts[proposalId_2] = true`).
5. **Scenario B (Reward & Mint Inflation):**
   - The contributor of `proposalId_2` calls `ServiceNft.updateImpact(999, proposalId_2)` where `virtualId = 999` has `_coreServices[999][0] == 0`.
   - `prevServiceId` is `0` and `_maturities[0]` is `0`.
   - `rawImpact` is calculated as `350 - 0 = 350` (7x the legitimate impact of `50`).
   - The contributor then calls `Minter.mint(proposalId_2)`, receiving 7x the intended amount of `AgentToken`, and receives a 7x larger weight in `AgentRewardV2`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:** Unauthorized modification of `_impacts` and `_maturities` mappings in `ServiceNft`.
- **Business Implications:**
  - Attackers can inflate their own model and dataset impact scores up to the full cumulative maturity value (up to 10,000), minting excess `AgentToken` from `Minter` and stealing a disproportionate share of contributor rewards in `AgentRewardV2` at the expense of other contributors.
  - Attackers can front-run `Minter.mint` or `AgentRewardV2` distributions to reset legitimate contributors' impacts to `0`, permanently depriving them of minted `AgentToken` and ongoing service rewards.

### Remediation

Change the visibility of `updateImpact` from `public` to `internal` (or `private`) so it can only be invoked during the DAO-authorized `ServiceNft.mint` execution flow before `_coreServices[virtualId][_cores[proposalId]]` is updated.

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

## Finding 9: Overwriting Dataset Impact in `ServiceNft.updateImpact` Corrupts Dataset Reward Accounting and Token Minting

### Synopsis

In `ServiceNft.updateImpact`, `_impacts[datasetId]` is overwritten via assignment (`=`) whenever a model referencing `datasetId` is minted. When multiple models reference the same dataset, this destructive update corrupts dataset impact accounting in `Minter.mint` and causes a divergence between cached `serviceReward.impact` and live `_rewardImpacts` totals in `AgentRewardV2._distributeContributorRewards`.

### Technical details

When a model contribution proposal is executed, `ServiceNft.mint` calls `ServiceNft.updateImpact(virtualId, proposalId)` (lines 90–108). If the model contribution is linked to a dataset (`datasetId > 0`), `updateImpact` assigns the dataset's impact directly:

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

Neither `ContributionNft` nor `ServiceNft` restricts multiple model contributions from referencing the same `datasetId`. Because line 101 uses destructive assignment (`=`) rather than accumulating (`+=`) or preventing dataset reuse, minting a subsequent model that references an existing `datasetId` overwrites `_impacts[datasetId]`.

This breaks accounting across two downstream components that consume `ServiceNft.getImpact(datasetId)`:

1. **Divergence between cached and live reward weights in `AgentRewardV2._distributeContributorRewards` (lines 269–302):**
   In the first loop (lines 269–281), `AgentRewardV2` queries the live `impact = serviceNftContract.getImpact(serviceId)`. If `impact == 0`, the loop skips `serviceId`. Otherwise, it caches `serviceReward.impact = impact` only once (when `serviceReward.impact == 0` at line 277) while adding the live `impact` to the denominator `_rewardImpacts[reward.id][core]` at line 280.
   In the second loop (lines 287–302), rewards are calculated using the cached `serviceReward.impact` in the numerator and the live sum `_rewardImpacts[reward.id][core]` in the denominator:
   ```solidity
   impactAmount =
       (reward.coreAmount * serviceReward.impact) /
       _rewardImpacts[reward.id][serviceNftContract.getCore(serviceId)];
   ```
   If Model 1 sets `_impacts[datasetId] = 70` and a reward distribution caches `serviceReward.impact = 70`, and subsequently Model 2 (referencing the same `datasetId`) has a lower `rawImpact` (e.g., `rawImpact = 0` when `_maturities[proposalId] <= _maturities[prevServiceId]`), `_impacts[datasetId]` is overwritten to a smaller value or `0`. On subsequent reward distributions:
   - The denominator `_rewardImpacts[reward.id][core]` reflects the reduced/zero live `_impacts[datasetId]`, whereas the numerator `serviceReward.impact` retains the larger cached `70`.
   - Consequently, the sum of `impactAmount` across services in that core exceeds `reward.coreAmount`, over-allocating rewards to `datasetId` (and potentially reverting with division-by-zero if `_rewardImpacts[reward.id][core]` is `0`).
   - Conversely, if no reward distribution occurred between Model 1 and Model 2 being minted, the dataset contributor permanently loses the impact earned from Model 1 if Model 2 has a lower `rawImpact`.

2. **Corrupted token minting in `Minter.mint` (lines 152–173):**
   `Minter.mint(nftId)` computes `dataAmount` for the dataset owner using the live `IServiceNft(serviceNft).getImpact(datasetId)`. If Model 2 (referencing the same `datasetId`) is minted in `ServiceNft` before `Minter.mint(model1)` is called, `Minter.mint(model1)` will read the overwritten `_impacts[datasetId]` from Model 2 instead of the dataset impact attributable to Model 1. Furthermore, calling `Minter.mint` for both Model 1 and Model 2 pays out `datasetId`'s current `_impacts[datasetId]` twice.

### Assumptions and preconditions

1. A dataset contribution (`datasetId`) has been minted as a `ServiceNft`.
2. Two model contribution proposals reference the same `datasetId` and are executed via `AgentDAO`, invoking `ServiceNft.mint`.

### Proof of concept

1. Contributor A submits a dataset contribution `D` (with `proposalId = D`) on Core 0. The DAO votes and executes `ServiceNft.mint`, minting `D` as a dataset service.
2. Contributor B submits Model 1 (`M1`) on Core 0 referencing `datasetId = D`. Validators vote such that `_maturities[M1] = 200`. Upon execution of `ServiceNft.mint` for `M1` (with `datasetImpactWeight = 7000`, i.e., 70%):
   - `rawImpact = 200 - 0 = 200`.
   - `_impacts[D] = (200 * 7000) / 10000 = 140`.
   - `_impacts[M1] = 200 - 140 = 60`.
3. Governance runs `AgentRewardV2.distributeRewards` and `distributeRewardsForAgents`. During `_distributeContributorRewards`:
   - `_serviceRewards[D].impact` is cached as `140`.
   - `_serviceRewards[M1].impact` is cached as `60`.
   - `_rewardImpacts[reward.id][0] = 140 + 60 = 200`.
4. A second model `M2` on Core 0 also references `datasetId = D`, and is executed with `_maturities[M2] = 180` (which is `<= _maturities[M1] = 200`):
   - `rawImpact = 0`.
   - Line 101 overwrites `_impacts[D] = 0`.
   - `_impacts[M2] = 0`.
5. On the next `AgentRewardV2.distributeRewardsForAgents` call (assume `reward.coreAmount = 300`):
   - In the first loop, `getImpact(D)` is `0` (skipped) and `getImpact(M1)` is `60`. Thus `_rewardImpacts[reward.id][0] = 60`.
   - In the second loop, `_serviceRewards[D].impact` is still the cached `140`, and `_serviceRewards[M1].impact` is `60`.
   - `D` receives `impactAmount = (300 * 140) / 60 = 700`, and `M1` receives `impactAmount = (300 * 60) / 60 = 300`, totaling `1000` instead of `300` (a 3.33x over-allocation that drains reward tokens held for other agents/contributors).
   - Furthermore, if `Minter.mint(M1)` had not yet been called prior to step 4, calling `Minter.mint(M1)` now reads `getImpact(D) == 0` and mints `0` tokens to the dataset owner instead of the expected reward for `140` impact.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Overwriting `_impacts[datasetId]` causes cached `serviceReward.impact` values in `AgentRewardV2` to exceed the live `_rewardImpacts` denominator (or causes a division-by-zero revert if all other services in that core also have 0 live impact), resulting in severe reward over-distribution or DoS of reward distribution, as well as incorrect token minting in `Minter.mint`.
- **Business Impact:** Dataset contributors can lose their earned AgentToken mint allocation or have their ongoing VIRTUAL rewards miscalculated, and `AgentRewardV2` reward pools can be drained via inflated `impactAmount` payouts.

### Remediation

Accumulate the dataset impact (`+=`) in `ServiceNft.updateImpact` so that multiple models referencing the same dataset add to its total impact rather than overwriting it (and ensure `AgentRewardV2` updates `serviceReward.impact = impact` on every distribution rather than only when `serviceReward.impact == 0`). Within the highlighted `ServiceNft.sol` section, accumulate `_impacts[datasetId]` and compute `_impacts[proposalId]` from the current model's dataset share:

```solidity
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
            uint256 datasetImpact = (rawImpact * datasetImpactWeight) / 10000;
            _impacts[datasetId] += datasetImpact;
            _impacts[proposalId] = rawImpact - datasetImpact;
```

## Finding 10: Lack of `datasetId` Validation in `ServiceNft.updateImpact` Allows Corrupting Arbitrary Service Impacts and Maturities

### Synopsis

In `ServiceNft.updateImpact`, the linked `datasetId` fetched from `ContributionNft` is not validated to be a dataset belonging to the same agent (`virtualId`), core type, or distinct from `proposalId` and other model services. This allows a model proposal to overwrite `_impacts` and `_maturities` of arbitrary services.

### Technical details

When a model contribution proposal is executed by an `AgentDAO`, `ServiceNft.mint` calls `ServiceNft.updateImpact(virtualId, proposalId)` (lines 90–108 of `contracts/contribution/ServiceNft.sol`):

```solidity
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
```

During contribution creation, the proposer supplies `datasetId` to `ContributionNft.mint`. However, `ServiceNft.updateImpact` performs no relationship or type integrity checks on `datasetId` before mutating `_impacts[datasetId]` and `_maturities[datasetId]`:

1. **Aliasing (`datasetId == proposalId`)**: If a model contribution specifies `datasetId == proposalId`, `_impacts[proposalId]` is set to `rawImpact - ((rawImpact * datasetImpactWeight) / 10000)`. When `Minter.mint(proposalId)` is subsequently executed, it reads `getImpact(nftId)` for `amount` and `getImpact(datasetId)` for `dataAmount` from the same storage slot `_impacts[proposalId]`. Whenever `datasetImpactWeight < 5000`, the model owner receives `2 * (10000 - datasetImpactWeight) / 10000` of the intended reward, minting excess `AgentToken`s out of thin air.
2. **Model / Dataset Type Confusion and Cross-Agent / Cross-Core Overwrites**: `datasetId` is not checked via `IContributionNft(contributionNft).isModel(datasetId)`, `IContributionNft(contributionNft).tokenVirtualId(datasetId) == virtualId`, or `_cores[datasetId] == _cores[proposalId]`. A model contribution can reference any existing `ServiceNft` token ID (including another model service, a previously rewarded dataset, or the active `_coreServices` of another `virtualId` or core) as its `datasetId`.
3. **Destructive Overwrite of Impact and Maturity**: Because `_impacts[datasetId]` and `_maturities[datasetId]` are overwritten using `=` rather than validated or accumulated, referencing an existing model or dataset service with a lower or zero `rawImpact` wipes out the victim service's `_impacts[datasetId]` prior to `Minter.mint` or `AgentRewardV2._distributeContributorRewards`, and alters `_maturities[datasetId]` (which corrupts subsequent `rawImpact` calculations if `datasetId` is an active core service).

### Assumptions and preconditions

1. A proposer creates a model contribution proposal (`isModel == true`) and mints the corresponding `ContributionNft` with a chosen `datasetId > 0`.
2. The proposal passes DAO voting and is executed, invoking `ServiceNft.mint` and `ServiceNft.updateImpact`.

### Proof of concept

1. **Scenario A (Self-Aliasing Double Mint)**:
   - Assume `datasetImpactWeight` is configured to `3000` (30%).
   - An attacker creates a model proposal `P` and calls `ContributionNft.mint` with `isModel = true` and `datasetId = P`.
   - When `P` is executed via `ServiceNft.mint`, `updateImpact` computes `rawImpact = 100`.
   - Line 101 sets `_impacts[P] = (100 * 3000) / 10000 = 30`.
   - Line 102 overwrites `_impacts[P] = 100 - 30 = 70`.
   - The attacker calls `Minter.mint(P)`. `Minter.mint` computes `amount` using `getImpact(P) = 70` and `dataAmount` using `getImpact(datasetId) = getImpact(P) = 70`, minting `140` impact units worth of `AgentToken` to the attacker instead of `100`.

2. **Scenario B (Overwriting Another Service's Impact and Maturity)**:
   - Contributor A has a model or dataset service `S` minted in `ServiceNft` with `_impacts[S] = 215` and `_maturities[S] = 315`, and has not yet called `Minter.mint(S)` (or `AgentRewardV2` has not yet snapshotted `S`).
   - Contributor B submits a model proposal `P` (on the same or a different `virtualId`) with `datasetId = S` that achieves `rawImpact = 0` (e.g., maturity did not improve over `prevServiceId`).
   - Upon execution of `P`, `updateImpact` sets `_impacts[S] = 0` and overwrites `_maturities[S] = _maturities[P]`.
   - When `Minter.mint(S)` or `AgentRewardV2._distributeContributorRewards` runs, Contributor A receives `0` tokens/rewards for `S`, and if `S` was the active `_coreServices` for its core, its baseline maturity is corrupted.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Arbitrary corruption of `_impacts` and `_maturities` mappings in `ServiceNft` across agents, cores, and service types, as well as double-counting of impact when `datasetId == proposalId`.
- **Business Impact**: Contributors can have their earned `AgentToken` mint allocations and ongoing `AgentRewardV2` contributor rewards zeroed out or stolen, core maturity baselines for other agents can be manipulated, and attackers can mint more `AgentToken` than allowed by the ELO impact calculation.

### Remediation

Validate in `ServiceNft.updateImpact` (or during `ServiceNft.mint`) that `datasetId` is a minted `ServiceNft` belonging to the same `virtualId` and core, is not a model contribution (which also guarantees `datasetId != proposalId`), and accumulate rather than overwrite dataset impact if a dataset can be reused across multiple models.

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
            _requireOwned(datasetId);
            require(!IContributionNft(contributionNft).isModel(datasetId), "Invalid dataset service");
            require(
                IContributionNft(contributionNft).tokenVirtualId(datasetId) == virtualId &&
                    _cores[datasetId] == _cores[proposalId],
                "Dataset virtualId or core mismatch"
            );
            require(_impacts[datasetId] == 0, "Dataset impact already set");

            _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
            _impacts[proposalId] = rawImpact - _impacts[datasetId];
            emit SetServiceScore(datasetId, _maturities[proposalId], _impacts[datasetId]);
            _maturities[datasetId] = _maturities[proposalId];
        }

        emit SetServiceScore(proposalId, _maturities[proposalId], _impacts[proposalId]);
    }
```

## Finding 11: Unvalidated Dataset Reference in `Minter.mint` Enables Cross-Agent Reward Manipulation and Minting DoS

### Synopsis

In `Minter.mint`, `datasetId` is retrieved from `ContributionNft.getDatasetId(nftId)` without verifying that `datasetId` belongs to the same `virtualId`, represents a dataset contribution rather than a model, and exists as an accepted `ServiceNft`. This allows cross-agent impact manipulation, unauthorized reward distribution, or permanent denial of service during reward minting.

### Technical details

When a contribution proposal is registered via `ContributionNft.mint`, the proposer supplies an arbitrary `datasetId` parameter that is stored in `modelDatasets[proposalId]` whenever `isModel_` is `true`:

```solidity
if (isModel_) {
    modelContributions[proposalId] = true;
    modelDatasets[proposalId] = datasetId;
}
```

Neither `ContributionNft.mint` nor `ServiceNft.mint` validates that a non-zero `datasetId` is a valid dataset contribution belonging to the same agent (`virtualId`) or that it has been accepted and minted in `ServiceNft`.

Subsequently, in `Minter.mint(uint256 nftId)` (lines 147–160 of `contracts/token/Minter.sol`), the contract validates that `nftId` belongs to a valid `virtualId` and is a model contribution (`require(contribution.isModel(nftId), "Not a model contribution")`), and then fetches `datasetId` at line 152:

```solidity
uint256 datasetId = contribution.getDatasetId(nftId);
uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
```

When `datasetId > 0`, `Minter.mint` queries `IServiceNft(serviceNft).getImpact(datasetId)` and transfers `dataAmount` of `virtualId`'s `AgentToken` (`tokenAddress`) to `IERC721(contributionNft).ownerOf(datasetId)`. Because `Minter.mint` performs no cross-contract integrity checks on `datasetId`, three integrity failures arise:

1. **Agent-to-asset binding inconsistency (`tokenVirtualId(datasetId) != virtualId`):** A model contribution for one agent (`virtualId`) can reference a `datasetId` belonging to a different agent, or a model proposal executed in a second agent DAO (where a malicious founder/validator controls voting and maturity scores) can reference a `datasetId` used by the first agent. When `ServiceNft.updateImpact` runs for the second agent, it overwrites `_impacts[datasetId]` in `ServiceNft`. Calling `Minter.mint(nftId)` for the first agent then uses the overwritten `_impacts[datasetId]` to distribute the first agent's tokens (or zeroes out a legitimate dataset's reward before it is minted).
2. **Invalid model-to-dataset association (`contribution.isModel(datasetId) == true` or `datasetId == nftId`):** `datasetId` can reference another model NFT (or `nftId` itself), allowing `ServiceNft.updateImpact` to overwrite a model's `_impacts` entry or allowing a model NFT to be rewarded a second time as a dataset.
3. **Cross-registry existence mismatch:** `ServiceNft.updateImpact` writes directly to `_impacts[datasetId]` and deducts the dataset share from `_impacts[nftId]` without verifying that `datasetId` was minted in `ServiceNft`. If `datasetId` was never executed as a `ServiceNft` (e.g., its proposal was defeated, canceled, or never existed), `ServiceNft.getImpact(datasetId)` in `Minter.mint` calls `_requireOwned(datasetId)` and reverts with `ERC721NonexistentToken(datasetId)`, permanently blocking `Minter.mint(nftId)` from minting rewards for the accepted model.

### Assumptions and preconditions

1. Core protocol contracts (`AgentNft`, `ContributionNft`, `ServiceNft`, `Minter`) are deployed and initialized with valid parameters.
2. For the denial-of-service scenario, a model contribution proposal is created and executed with a non-zero `datasetId` whose corresponding `ServiceNft` has not been minted (e.g., rejected or unexecuted dataset proposal).
3. For the cross-agent impact overwrite scenario, an unprivileged actor creates or controls a second virtual agent DAO (or submits a competing contribution proposal) referencing the target `datasetId` or `nftId`.

### Proof of concept

#### Scenario A: Cross-Agent Impact Overwrite and Token Drain / Griefing
1. In Agent 1 (`virtualId = 1`), Dataset `D1` and Model `M1` (referencing `datasetId = D1`) are proposed, approved by Agent 1's DAO, and executed via `ServiceNft.mint`, setting `_impacts[D1]` and `_impacts[M1]`.
2. Before `Minter.mint(M1)` is called, an attacker who controls Validator votes in Agent 2 (`virtualId = 2`) creates and executes a model contribution `M2` in Agent 2 with `datasetId = D1` (or `datasetId = M1`).
3. During `ServiceNft.mint` for `M2`, `ServiceNft.updateImpact(2, M2)` reads `datasetId = D1` and overwrites `_impacts[D1] = (rawImpact_M2 * datasetImpactWeight) / 10000`.
   - If `rawImpact_M2 == 0`, `_impacts[D1]` is overwritten to `0`, depriving `D1`'s owner of their Agent 1 token rewards when `Minter.mint(M1)` is called.
   - If the attacker owns `D1` and inflates `rawImpact_M2` in Agent 2's DAO, `_impacts[D1]` is overwritten to the inflated value; calling `Minter.mint(M1)` then transfers the inflated `dataAmount` of Agent 1's `AgentToken` to the attacker.

#### Scenario B: Permanent Denial of Service on Model Reward Minting
1. Contributor A submits dataset proposal `D1` and mints `ContributionNft(D1)`, while Contributor B submits model proposal `M1` referencing `datasetId = D1` and mints `ContributionNft(M1)`.
2. Proposal `M1` succeeds and is executed via `ServiceNft.mint`, which deducts the dataset share from `_impacts[M1]` and writes `_impacts[D1]`.
3. Proposal `D1` is defeated or canceled in `AgentDAO`, so `ServiceNft(D1)` is never minted.
4. When Contributor B calls `Minter.mint(M1)`, `IServiceNft(serviceNft).getImpact(D1)` reverts with `ERC721NonexistentToken(D1)` because `_requireOwned(D1)` fails in `ServiceNft`, permanently locking Contributor B's model reward and the IP vault share.

### Impact assessment

- **Severity:** High
- **Technical Impact:**
  - Unauthorized manipulation of `_impacts[datasetId]` across different agents (`virtualId`s) or across model/dataset types, leading to inflated `AgentToken` payouts from `Minter` or zeroing out of legitimate contributor rewards.
  - Permanent denial of service in `Minter.mint(nftId)` for any accepted model contribution whose referenced `datasetId` was not minted in `ServiceNft`.
- **Business Impact:** Loss of `AgentToken` funds held by `Minter`, corrupted reward accounting for model and dataset contributors, and permanently frozen contributor and IP vault rewards.

### Remediation

Validate `datasetId` in `Minter.mint` (and enforce corresponding checks in `ContributionNft.mint` / `ServiceNft.mint`) to ensure that any non-zero `datasetId` belongs to the same `virtualId`, is not a model contribution, and exists as an owned `ServiceNft` before computing rewards. If `datasetId` is not a valid minted `ServiceNft` of the same agent, either revert during contribution/service minting or ignore invalid dataset associations in `Minter.mint`.

```solidity
        address tokenAddress = IAgentNft(agentNft).virtualInfo(virtualId).token;
        IContributionNft contribution = IContributionNft(contributionNft);
        require(contribution.isModel(nftId), "Not a model contribution");

        uint256 finalImpactMultiplier = _getImpactMultiplier(virtualId);
        uint256 datasetId = contribution.getDatasetId(nftId);
        if (datasetId != 0) {
            require(
                contribution.tokenVirtualId(datasetId) == virtualId &&
                    !contribution.isModel(datasetId) &&
                    contribution.getCore(datasetId) == contribution.getCore(nftId) &&
                    IERC721(serviceNft).ownerOf(datasetId) != address(0),
                "Invalid dataset"
            );
        }
        uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
        if (impact > maxImpact) {
            impact = maxImpact;
        }
        uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
```

## Finding 12: Shared Dataset Impact State Reuse and Overwrite Across Model Claims in `Minter.mint`

### Synopsis

`Minter.mint` resolves a model contribution's dataset reward via `contribution.getDatasetId(nftId)` and settles tokens using the shared `IServiceNft(serviceNft).getImpact(datasetId)` slot. Because multiple model contributions can reference the same `datasetId`, subsequent models overwrite `_impacts[datasetId]` before earlier claims settle, inflating or zeroing dataset and IP rewards.

### Technical details

When a model contribution is registered in `ContributionNft.mint`, its associated `datasetId` is stored in `modelDatasets[proposalId]` without restricting whether that `datasetId` has already been linked to another model contribution.

When the model proposal succeeds and `ServiceNft.mint` executes, `ServiceNft.updateImpact(virtualId, proposalId)` computes `rawImpact` for the model and splits it between the model and its linked dataset:

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

In `Minter.mint(uint256 nftId)`, the contract records `_mintedNfts[nftId] = true` only for the model `nftId`, retrieves `uint256 datasetId = contribution.getDatasetId(nftId);` (line 152), and calculates `dataAmount` by reading `IServiceNft(serviceNft).getImpact(datasetId)` (lines 158–160).

This creates a shared mutable reward-state flaw across claims:
1. **Overwrite of unsettled dataset rewards:** `_impacts[datasetId]` is keyed solely by `datasetId` rather than by `(modelId, datasetId)`. If Model A references `datasetId` (having `datasetImpactWeight` deducted from `_impacts[modelA]`), and Model B also references the same `datasetId` and is executed in `ServiceNft.mint` before `Minter.mint(modelA)` is invoked, `_impacts[datasetId]` is overwritten with Model B's dataset share.
   - If Model B has `rawImpact == 0` (e.g., its maturity does not exceed the current core service maturity), `_impacts[datasetId]` is overwritten to `0`. When `Minter.mint(modelA)` is later called, the dataset owner receives `0` tokens even though `modelA` already had `datasetImpactWeight` deducted from `_impacts[modelA]`.
   - If Model B has a larger `rawImpact` than Model A while `Minter.mint(modelA)` is still unsettled, `_impacts[datasetId]` is overwritten to the higher value. Calling `Minter.mint(modelA)` and `Minter.mint(modelB)` then settles both claims against the higher `_impacts[datasetId]` value, minting excess `AgentToken` to `datasetOwner` and `ipVault`.
2. **Missing per-claim dataset settlement tracking:** `Minter.mint` does not mark `datasetId` in `_mintedNfts` or isolate dataset impact per model proposal, allowing the same shared `_impacts[datasetId]` state to be consumed across multiple `Minter.mint` executions.

### Assumptions and preconditions

1. A dataset contribution `datasetId` exists for an agent.
2. Two model contributions (`modelA` and `modelB`) both reference `datasetId` and pass DAO voting to execute `ServiceNft.mint`.
3. `Minter.mint(modelA)` is called after `ServiceNft.mint` has executed for `modelB`.

### Proof of concept

1. Suppose `datasetImpactWeight = 7000` (70%), `impactMultiplier = 10000`, and `ipShare = 1000` (10%).
2. Contributor 1 submits a dataset contribution `datasetId = 10`.
3. Contributor 2 submits Model A (`modelA = 11`) referencing `datasetId = 10`. The DAO votes and executes `ServiceNft.mint` for `modelA` with `rawImpact = 100`:
   - `_impacts[10]` (`datasetId`) is set to `70`.
   - `_impacts[11]` (`modelA`) is set to `30`.
4. Before `Minter.mint(11)` is called, Model B (`modelB = 12`) is submitted also referencing `datasetId = 10`:
   - **Scenario A (Reward Inflation):** Model B achieves `rawImpact = 200`. `ServiceNft.mint` updates `_impacts[10] = 140` and `_impacts[12] = 60`. Now, when `Minter.mint(11)` and `Minter.mint(12)` are called, `IServiceNft(serviceNft).getImpact(10)` returns `140` for both calls. The dataset owner is paid `140e18 + 140e18 = 280e18` tokens instead of `70e18 + 140e18 = 210e18` tokens (and `ipVault` is similarly overpaid).
   - **Scenario B (Reward Loss):** Model B achieves a maturity less than or equal to `modelA`'s maturity, resulting in `rawImpact = 0`. `ServiceNft.mint` overwrites `_impacts[10] = 0` and sets `_impacts[12] = 0`. When `Minter.mint(11)` is called, `modelA`'s owner only receives `30e18` tokens (as `70` was already deducted from `_impacts[11]`), while `datasetOwner` receives `0` instead of `70e18` tokens.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** `Minter.mint` settles dataset and IP token rewards against a shared, mutable `_impacts[datasetId]` value that is overwritten whenever another model contribution references the same `datasetId`.
- **Business Impact:** Dataset contributors and the IP vault can either lose legitimately earned `AgentToken` rewards (despite the model contributor already having that portion deducted from their model impact) or receive inflated duplicate payouts across multiple model claims.

### Remediation

Store the dataset impact attributable to each model contribution independently (or prevent `datasetId` reuse and record dataset mint status in `Minter.mint`), and cap total model-plus-dataset impact properly. If a dataset can only be rewarded once, mark `datasetId` as minted in `Minter.mint`; if a dataset can be reused across models, compute or store the dataset impact per `nftId` so that subsequent models cannot overwrite unsettled dataset rewards:

```solidity
        address tokenAddress = IAgentNft(agentNft).virtualInfo(virtualId).token;
        IContributionNft contribution = IContributionNft(contributionNft);
        require(contribution.isModel(nftId), "Not a model contribution");

        uint256 finalImpactMultiplier = _getImpactMultiplier(virtualId);
        uint256 datasetId = contribution.getDatasetId(nftId);
        if (datasetId != 0) {
            require(!_mintedNfts[datasetId], "Dataset already minted");
            _mintedNfts[datasetId] = true;
        }
        uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
        if (impact > maxImpact) {
            impact = maxImpact;
        }
        uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
```

## Finding 13: Uncapped Dataset Impact and Inconsistent `maxImpact` Enforcement in `Minter.mint`

### Synopsis

In `Minter.mint`, the `maxImpact` cap is applied exclusively to the model's post-split impact value while the associated dataset's impact is left completely uncapped. This inconsistent cap enforcement allows contributions with a dataset to bypass `maxImpact` and mint excessive `AgentToken` rewards.

### Technical details

When a model contribution proposal is executed in `ServiceNft.updateImpact` (lines 90–108), the total maturity increase (`rawImpact`) is split between the model contribution (`proposalId`) and its associated dataset contribution (`datasetId`):

```solidity
_impacts[proposalId] = rawImpact;
if (datasetId > 0) {
    _impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
    _impacts[proposalId] = rawImpact - _impacts[datasetId];
    ...
}
```

As shown in the test suite (`test/contribution.js`), `datasetImpactWeight` can represent the majority of the total impact (e.g., 70% to the dataset and 30% to the model).

However, when `Minter.mint` calculates the reward amounts to distribute (`contracts/token/Minter.sol`, lines 153–160):

```solidity
uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
if (impact > maxImpact) {
    impact = maxImpact;
}
uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
uint256 dataAmount = datasetId > 0
    ? (IServiceNft(serviceNft).getImpact(datasetId) * finalImpactMultiplier * 10 ** 18) / DENOM
    : 0;
```

There are two intertwined flaws in this `maxImpact` check:
1. **Uncapped dataset impact:** `IServiceNft(serviceNft).getImpact(datasetId)` is used directly in the calculation of `dataAmount` without any `maxImpact` bound.
2. **Post-split model cap instead of total impact cap:** Because `IServiceNft(serviceNft).getImpact(nftId)` already represents only the model's share `(10000 - datasetImpactWeight) / 10000` of `rawImpact`, comparing only `nftId`'s impact against `maxImpact` allows the model's share alone to reach `maxImpact` while `datasetId`'s share remains completely unbounded. Consequently, `amount + dataAmount` (and the derived `ipAmount`) can far exceed the maximum intended emission per contribution.

### Assumptions and preconditions

1. `maxImpact` is configured to a finite bound to limit the maximum token emission per contribution.
2. A model contribution is submitted referencing a valid `datasetId` (`datasetId > 0`).
3. The contribution achieves a `rawImpact` in `ServiceNft` that exceeds `maxImpact`.

### Proof of concept

Consider the configuration from `test/contribution.js` where `datasetImpactWeight = 7000` (70% dataset, 30% model), `finalImpactMultiplier = 10_000_000` (1000 tokens per unit of impact), `ipShare = 1000` (10%), and assume `maxImpact = 300`:

1. **Contribution without a dataset (`datasetId = 0`) and `rawImpact = 1000`:**
   - In `ServiceNft.updateImpact`, `_impacts[nftId] = 1000`.
   - In `Minter.mint`, `impact` (1000) is capped to `maxImpact` (300).
   - `amount = 300 * 1000e18 = 300,000e18`, `dataAmount = 0`, `ipAmount = 30,000e18`.
   - Total minted is capped at `330,000e18` tokens.

2. **Contribution with a dataset (`datasetId > 0`) and the exact same `rawImpact = 1000`:**
   - In `ServiceNft.updateImpact`, `_impacts[datasetId] = (1000 * 7000) / 10000 = 700`, and `_impacts[nftId] = 1000 - 700 = 300`.
   - In `Minter.mint`:
     - `impact` is `300`, which is `<= maxImpact` (300), so `amount = 300,000e18`.
     - `datasetId` impact is `700` (uncapped), so `dataAmount = 700,000e18`.
     - `ipAmount = (300,000e18 + 700,000e18) * 10% = 100,000e18`.
   - Total distributed is `1,100,000e18` tokens—completely bypassing the `maxImpact = 300` cap and distributing full rewards for all `1000` impact units.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** The `maxImpact` safeguard fails whenever a contribution includes a dataset. Both `dataAmount` and `ipAmount` (as well as the combined `amount + dataAmount`) can exceed the protocol's configured maximum impact limit.
- **Business Impact:** Excessive `AgentToken` rewards are transferred from the `Minter` contract to the model owner, dataset owner, and IP vault, prematurely depleting the `Minter` contract's token reserves and inflating circulating token supply beyond intended bounds.

### Remediation

Cap the combined `rawImpact` (model impact plus dataset impact) at `maxImpact` and preserve the configured split between the model and dataset, or cap `rawImpact` in `ServiceNft.updateImpact` before splitting. Within `Minter.mint`, this can be fixed as follows:

```solidity
        uint256 finalImpactMultiplier = _getImpactMultiplier(virtualId);
        uint256 datasetId = contribution.getDatasetId(nftId);
        uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
        uint256 dataImpact = datasetId > 0 ? IServiceNft(serviceNft).getImpact(datasetId) : 0;
        uint256 totalImpact = impact + dataImpact;
        if (totalImpact > maxImpact) {
            impact = (impact * maxImpact) / totalImpact;
            dataImpact = maxImpact - impact;
        }
        uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 dataAmount = (dataImpact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
```

## Finding 14: Unvalidated `datasetId` entitlement in `Minter.mint` allows unauthorized or manipulated dataset reward minting

### Synopsis

In `Minter.mint`, dataset rewards (`dataAmount`) are calculated from `IServiceNft(serviceNft).getImpact(datasetId)` and transferred to `IERC721(contributionNft).ownerOf(datasetId)` without validating that `datasetId` is an unrewarded dataset contribution belonging to the same agent (`virtualId`), allowing unauthorized reward extraction or denial of dataset rewards.

### Technical details

When a model contribution NFT (`nftId`) is minted in `ContributionNft.mint`, the proposer can supply any arbitrary `datasetId`, which is stored in `modelDatasets[proposalId]` (line 91 of `ContributionNft.sol`) without verifying that `datasetId` is an existing dataset contribution for the same `virtualId` and `coreId`.

Subsequently, when `Minter.mint(uint256 nftId)` is called to distribute `AgentToken` rewards for the model contribution (`nftId`), the contract validates `nftId` (checking `!_mintedNfts[nftId]`, `virtualId != 0`, and `contribution.isModel(nftId)`), but performs no entitlement or state validation on `datasetId` at lines 158–171:

```solidity
uint256 dataAmount = datasetId > 0
    ? (IServiceNft(serviceNft).getImpact(datasetId) * finalImpactMultiplier * 10 ** 18) / DENOM
    : 0;
uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
...
// Mint to Dataset owner
if (datasetId != 0) {
    address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
    IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount);
}
```

Specifically:
1. `Minter.mint` does not verify that `datasetId` belongs to the same agent (`IContributionNft(contributionNft).tokenVirtualId(datasetId) == virtualId`) or that `datasetId` is actually a dataset contribution (`!IContributionNft(contributionNft).isModel(datasetId)`).
2. `Minter.mint` does not record or check `_mintedNfts[datasetId]`, nor does it cap `IServiceNft(serviceNft).getImpact(datasetId)` with `maxImpact`. A single dataset NFT (or even a model NFT from the same or another agent) can be referenced as `datasetId` across multiple model contributions to repeatedly claim `dataAmount` in `Minter.mint`.
3. Because `ServiceNft.getImpact(datasetId)` reads the mutable `_impacts[datasetId]` storage slot written by `ServiceNft.updateImpact(virtualId, proposalId)` (which is `public`), any proposal referencing `datasetId` (including proposals on a different `virtualId` with a higher maturity score or an unapproved proposal with `0` maturity) can overwrite `_impacts[datasetId]` before `Minter.mint(nftId)` executes, either inflating `dataAmount` to drain `AgentToken` from `Minter` or zeroing out `dataAmount` to deny rewards to the dataset owner.
4. If `datasetId` was minted in `ContributionNft` (or set to a non-zero ID) but never approved/minted in `ServiceNft`, `ServiceNft.updateImpact` still deducts `_impacts[datasetId]` from `_impacts[nftId]`, while `IServiceNft(serviceNft).getImpact(datasetId)` at line 159 reverts with `ERC721NonexistentToken(datasetId)` due to `_requireOwned(datasetId)`, permanently blocking `Minter.mint(nftId)`.

### Assumptions and preconditions

1. An agent (`virtualId`) is deployed and `Minter` holds `AgentToken` balances for reward distribution.
2. A model contribution (`nftId`) is proposed and approved by the Agent DAO, or an attacker interacts with `ContributionNft` / `ServiceNft` using a shared `datasetId`.

### Proof of concept

1. Dataset contributor creates a dataset contribution `D` on Agent 1 (`virtualId = 1`), which is approved and minted in `ServiceNft`.
2. Model contributor creates a model contribution `M1` on Agent 1 referencing `datasetId = D`. When `M1` is approved with `rawImpact = 100` (and `datasetImpactWeight = 7000`), `ServiceNft.updateImpact` sets `_impacts[M1] = 30` and `_impacts[D] = 70`.
3. `Minter.mint(M1)` is executed:
   - Line 159 reads `IServiceNft(serviceNft).getImpact(D) == 70` and computes `dataAmount`.
   - Line 171 fetches `datasetOwner = IERC721(contributionNft).ownerOf(D)` and transfers `dataAmount` of Agent 1's `AgentToken` to `datasetOwner`.
   - `_mintedNfts[M1]` is set to `true`, while `_mintedNfts[D]` remains `false`.
4. A second model contribution `M2` (or a model contribution on another agent whose `ServiceNft` impact for `D` was updated) also sets `datasetId = D` (or reuses an existing `ServiceNft` ID `D` that already has non-zero impact).
5. When `Minter.mint(M2)` is called, `Minter.mint` again computes `dataAmount` from `IServiceNft(serviceNft).getImpact(D)` at lines 158–160 and transfers `dataAmount` to `datasetOwner` at lines 170–172, paying out dataset rewards multiple times or across agents using unvalidated `datasetId` associations.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Unauthorized minting/transfer of `AgentToken` rewards from `Minter`, manipulation of `dataAmount` and `ipAmount`, or permanent denial of service of `Minter.mint(nftId)` if `datasetId` is not minted in `ServiceNft`.
- **Business Impact**: Depletion of agent token reward reserves in `Minter` and unfair reward distribution among model contributors, dataset contributors, and the IP vault.

### Remediation

Validate in `Minter.mint` that `datasetId` belongs to the same `virtualId`, is not a model contribution, has not already been minted/rewarded, exists in `ServiceNft`, and has its impact bounded by `maxImpact`.

```solidity
        uint256 dataAmount = 0;
        if (datasetId > 0) {
            require(!_mintedNfts[datasetId], "Dataset already minted");
            require(
                contribution.tokenVirtualId(datasetId) == virtualId && !contribution.isModel(datasetId),
                "Invalid datasetId"
            );
            _mintedNfts[datasetId] = true;
            uint256 dataImpact = IServiceNft(serviceNft).getImpact(datasetId);
            if (impact + dataImpact > maxImpact) {
                dataImpact = maxImpact > impact ? maxImpact - impact : 0;
            }
            dataAmount = (dataImpact * finalImpactMultiplier * 10 ** 18) / DENOM;
        }
        uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;

        // Mint to model owner
        if (amount > 0) {
            address modelOwner = IERC721(contributionNft).ownerOf(nftId);
            IAgentToken(tokenAddress).transfer(modelOwner, amount);
        }

        // Mint to Dataset owner
        if (dataAmount > 0) {
            address datasetOwner = IERC721(contributionNft).ownerOf(datasetId);
            IAgentToken(tokenAddress).transfer(datasetOwner, dataAmount);
        }
```

## Finding 15: Shared Dataset Impact State in `Minter.mint` Causes Incorrect Dataset and IP Token Rewards

### Synopsis

In `Minter.mint`, dataset reward entitlements (`dataAmount`) are calculated dynamically at claim time by querying `IServiceNft(serviceNft).getImpact(datasetId)`. Because `ServiceNft` overwrites `_impacts[datasetId]` whenever another model uses the same dataset or `updateImpact` is called, deferred minting causes reward drift, double-minting of high dataset impacts, or zeroed dataset payouts.

### Technical details

When a model contribution proposal is executed via `ServiceNft.mint`, `ServiceNft.updateImpact(virtualId, proposalId)` computes `rawImpact` for the model (`proposalId`). If the model references a dataset (`datasetId = IContributionNft(contributionNft).getDatasetId(proposalId) > 0`), `ServiceNft.updateImpact` splits `rawImpact` between the dataset and the model:

```solidity
_impacts[datasetId] = (rawImpact * datasetImpactWeight) / 10000;
_impacts[proposalId] = rawImpact - _impacts[datasetId];
```

While `_impacts[proposalId]` is uniquely keyed to the specific model contribution, `_impacts[datasetId]` is keyed only by `datasetId` and is shared across all model contributions that reference the same dataset.

Token settlement in `Minter.mint(uint256 nftId)` is decoupled from `ServiceNft.mint` and can be invoked at any time after the service is minted. At lines 158–161 of `Minter.sol`, `Minter.mint` calculates `dataAmount` and `ipAmount` as follows:

```solidity
uint256 dataAmount = datasetId > 0
    ? (IServiceNft(serviceNft).getImpact(datasetId) * finalImpactMultiplier * 10 ** 18) / DENOM
    : 0;
uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
```

This creates a deferred settlement inconsistency due to shared reward-state aliasing:

1. **Overwriting by Subsequent Model Contributions:** Multiple model contributions can reference the same `datasetId` in `ContributionNft`. When a second model contribution (`nftId_2`) using the same `datasetId` is minted in `ServiceNft`, `updateImpact` overwrites `_impacts[datasetId]` with the dataset share of `nftId_2`'s `rawImpact`.
   - If `Minter.mint(nftId_1)` is called after `nftId_2` is minted in `ServiceNft`, `Minter.mint(nftId_1)` reads `nftId_2`'s dataset impact instead of `nftId_1`'s dataset impact.
   - If `nftId_1` had a low impact and `nftId_2` had a high impact, calling `Minter.mint(nftId_1)` and `Minter.mint(nftId_2)` after `nftId_2` is approved pays the high dataset reward twice, over-distributing `AgentToken` to `datasetOwner` and `ipVault`.
   - Conversely, if `nftId_1` had a high impact and `nftId_2` had a lower (or zero) impact, delaying `Minter.mint(nftId_1)` until after `nftId_2` is minted reduces or zeroes out the dataset owner's reward and the corresponding `ipAmount` for `nftId_1`.

2. **Unrestricted `ServiceNft.updateImpact` Calls:** `ServiceNft.updateImpact(uint256 virtualId, uint256 proposalId)` is `public` and lacks access control. Once `_coreServices[virtualId][core]` has updated to `proposalId` (or a newer service with higher maturity), anyone can call `ServiceNft.updateImpact(virtualId, proposalId)` directly. Because `_maturities[proposalId] <= _maturities[prevServiceId]`, `rawImpact` evaluates to `0`, overwriting `_impacts[datasetId]` to `0` before `Minter.mint(proposalId)` is called.

3. **Uncapped Dataset Impact:** While `impact` for the model is capped at `maxImpact` (lines 154–156), `IServiceNft(serviceNft).getImpact(datasetId)` at line 159 is not capped or derived proportionally from the capped model impact.

### Assumptions and preconditions

- At least one dataset contribution (`datasetId > 0`) exists for the virtual agent.
- Either multiple model contributions reference the same `datasetId` before `Minter.mint` is called for the earlier model(s), or `ServiceNft.updateImpact` is invoked directly before `Minter.mint` settles the model's rewards.

### Proof of concept

1. Contributor A submits a dataset contribution `D` (`datasetId = D`).
2. Contributor B submits Model 1 referencing dataset `D`. Assume Model 1 achieves `rawImpact = 100` and `datasetImpactWeight = 7000` (70%). When `ServiceNft.mint` executes for Model 1:
   - `_impacts[Model_1] = 30`
   - `_impacts[D] = 70`
   - `Minter.mint(Model_1)` is not called yet.
3. Contributor C submits Model 2 also referencing dataset `D`. Assume Model 2 achieves `rawImpact = 500`. When `ServiceNft.mint` executes for Model 2:
   - `_impacts[Model_2] = 150`
   - `_impacts[D]` is overwritten from `70` to `350` (`500 * 7000 / 10000`).
4. `Minter.mint(Model_1)` and `Minter.mint(Model_2)` are now called:
   - In `Minter.mint(Model_1)`, `IServiceNft(serviceNft).getImpact(D)` returns `350` instead of `70`.
   - In `Minter.mint(Model_2)`, `IServiceNft(serviceNft).getImpact(D)` also returns `350`.
   - Contributor A receives dataset rewards corresponding to `350 + 350 = 700` impact instead of the intended `70 + 350 = 420` impact, and `ipVault` similarly receives inflated `ipAmount` tokens.
5. Alternatively, after step 2, any user calls `ServiceNft.updateImpact(virtualId, Model_1)` directly. Because `_coreServices[virtualId][core]` is now `Model_1`, `_maturities[Model_1] > _maturities[Model_1]` is `false`, setting `rawImpact = 0` and overwriting `_impacts[D] = 0` and `_impacts[Model_1] = 0`. When `Minter.mint(Model_1)` is subsequently called, both the model owner and dataset owner receive `0` tokens.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:** `Minter.mint` computes corrupted `dataAmount` and `ipAmount` values whenever `_impacts[datasetId]` is modified between `ServiceNft.mint` and `Minter.mint`.
- **Business Implications:** Dataset owners and the IP vault can either be deprived of their earned `AgentToken` rewards (reduced to zero or a smaller impact) or be overpaid multiple times for a single high-impact model contribution, draining `AgentToken` reserves held by the `Minter` contract.

### Remediation

Record the dataset impact associated with each specific model contribution (or store the model's total raw impact per `proposalId` and compute the model/dataset split and `maxImpact` cap per model), and restrict `ServiceNft.updateImpact` so it cannot be called externally to overwrite settled impacts.

In `Minter.sol`, ensure the total impact for `nftId` (including its dataset share) is bounded and tied to `nftId` rather than reading a mutable shared `_impacts[datasetId]` slot:

```solidity
        uint256 datasetId = contribution.getDatasetId(nftId);
        uint256 impact = IServiceNft(serviceNft).getImpact(nftId);
        uint256 dataImpact = datasetId > 0 ? IServiceNft(serviceNft).getDatasetImpact(nftId) : 0;
        uint256 totalImpact = impact + dataImpact;
        if (totalImpact > maxImpact) {
            impact = (impact * maxImpact) / totalImpact;
            dataImpact = maxImpact - impact;
        }
        uint256 amount = (impact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 dataAmount = (dataImpact * finalImpactMultiplier * 10 ** 18) / DENOM;
        uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
```

Where `ServiceNft` tracks the per-model dataset impact (e.g., `_modelDatasetImpacts[proposalId]`) during `ServiceNft.mint` and restricts `updateImpact` to `internal` visibility.

## Finding 16: Ignored `ipShareOverrides` in `Minter.mint` causes per-agent IP share configurations to be bypassed

### Synopsis

In `Minter.mint`, the calculation of `ipAmount` uses the global `ipShare` state variable directly instead of checking the per-agent `ipShareOverrides` mapping. Consequently, any custom IP share configured by an Agent DAO via `setIPShareOverride` is completely ignored during reward minting, resulting in incorrect token allocations to `ipVault`.

### Technical details

The `Minter` contract supports both global and per-agent configurations for reward calculation parameters:
1. Impact Multiplier: Stored globally in `impactMultiplier` (line 24) and overridable per agent in `impactMulOverrides` (line 32). In `mint`, the contract retrieves the effective multiplier via `_getImpactMultiplier(virtualId)` (line 151), which checks `impactMulOverrides[virtualId]` before falling back to `impactMultiplier`.
2. IP Share: Stored globally in `ipShare` (line 23) and overridable per agent in `ipShareOverrides` (line 33). An Agent DAO can update its specific override by calling `setIPShareOverride(uint256 virtualId, uint256 _ipShare)` (lines 99–102).

However, in `Minter.mint` (line 161), the `ipAmount` calculation directly references the global `ipShare` variable:

```solidity
uint256 ipAmount = ((amount + dataAmount) * ipShare) / DENOM;
```

The `ipShareOverrides[virtualId]` mapping is never read anywhere in the contract. There is no helper function (analogous to `_getImpactMultiplier`) to resolve the effective IP share for `virtualId`. As a result, any per-agent IP share policy set by an Agent DAO has no effect on the tokens transferred to `ipVault` (lines 176–178).

### Assumptions and preconditions

1. An Agent DAO calls `Minter.setIPShareOverride(virtualId, customIpShare)` to set a custom IP share (`customIpShare != ipShare`).
2. A model contribution NFT for `virtualId` is minted and `Minter.mint(nftId)` is invoked with a non-zero reward amount (`amount + dataAmount > 0`).

### Proof of concept

1. Suppose `Minter` is initialized with a global `ipShare = 1000` (10%, given `DENOM = 10000`).
2. The Agent DAO for `virtualId = 1` passes a governance proposal calling `Minter.setIPShareOverride(1, 2000)` to increase the IP share for its agent to 20%.
3. A contributor submits a model contribution (`nftId`) for `virtualId = 1` that achieves an impact resulting in `amount + dataAmount = 100,000 * 10**18` agent tokens.
4. When `Minter.mint(nftId)` is called, line 161 computes:
   `ipAmount = (100,000 * 10**18 * 1000) / 10000 = 10,000 * 10**18`
   using the global `ipShare` (1000) instead of `ipShareOverrides[1]` (2000).
5. `ipVault` receives `10,000 * 10**18` tokens instead of the intended `20,000 * 10**18` tokens (or receives more tokens than intended if the Agent DAO had lowered the override below the global `ipShare`).

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: Per-agent governance over IP share distribution (`setIPShareOverride`) is completely non-functional.
- **Business Impact**: Depending on whether an Agent DAO set its override higher or lower than the global `ipShare`, `ipVault` will systematically receive fewer or more agent tokens than dictated by the agent's governance policy on every model contribution mint.

### Remediation

Introduce an internal helper function `_getIPShare(uint256 virtualId)` analogous to `_getImpactMultiplier(uint256 virtualId)`, and use it when calculating `ipAmount` in `Minter.mint`.

```solidity
    function _getIPShare(uint256 virtualId) internal view returns (uint256) {
        uint256 share = ipShareOverrides[virtualId];
        if (share == 0) {
            share = ipShare;
        }
        return share;
    }
```

And in `Minter.mint`:

```solidity
        uint256 finalIPShare = _getIPShare(virtualId);
        uint256 ipAmount = ((amount + dataAmount) * finalIPShare) / DENOM;
```

## Finding 17: Missing `votes` Array Length Validation in `AgentDAO._updateMaturity` and `AgentDAO._calcMaturity` Allows Validators to Inflate Maturity and Impact Scores

### Synopsis

`AgentDAO._updateMaturity` and `AgentDAO._calcMaturity` decode the voter-supplied `params` bytes into a `uint8[] memory votes` array without validating its length. A validator can supply more than the intended 10 battle outcomes to `EloCalculator.battleElo`, inflating a proposal's maturity score, `ServiceNft` impact, and resulting token rewards.

### Technical details

When a validator casts a `For` vote (`support == 1`) with non-empty `params` on an `AgentDAO` proposal, `_castVote` invokes `_updateMaturity(account, proposalId, weight, params)` (line 139).

In `AgentDAO._updateMaturity` (lines 171–176):
```solidity
uint8[] memory votes = abi.decode(params, (uint8[]));
uint256 maturity = _calcMaturity(proposalId, votes);

_proposalMaturities[proposalId] += (maturity * weight);

emit ValidatorEloRating(proposalId, account, maturity, votes);
```
`_calcMaturity` (lines 179–193) retrieves the current `coreService` maturity and passes the unvalidated `votes` array directly to `EloCalculator.battleElo(maturity, votes)` (line 189):
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

The protocol's Elo scoring model is designed around a 10-match evaluation (`votes.length == 10`), where a completely one-sided vote (`[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]`) yields a maximum Elo rating increase of `215` (`eloA = 1215`, `eloB = 785`).

However, neither `_updateMaturity`, `_calcMaturity`, nor `EloCalculator.battleElo` enforces `votes.length == 10` (or `votes.length <= 10`). Furthermore, in `Elo.ratingChange(eloB, eloA, result, k)`, `ratingA` is `eloB` and `ratingB` is `eloA`. When `eloA > eloB`, `_negative` (`ratingB < ratingA`) is `false`, so only `require(ratingDiff < 1126, "Rating difference too large")` applies, and each additional win (`1`) increases `change` up to `30` Elo points per step.

Consequently, a voter can pass up to 22–23 consecutive `1`s in `votes` before `ratingDiff` reaches `1126`, increasing `eloA` from `1000` to over `1550` (`+550` maturity increase in a single contribution instead of the intended maximum of `+215`). When the proposal executes and calls `ServiceNft.mint`, `ServiceNft.updateImpact` computes `rawImpact = _maturities[proposalId] - _maturities[prevServiceId]`, directly recording the inflated maturity and impact on the minted `ServiceNft`.

### Assumptions and preconditions

1. An active virtual persona has an existing core `ServiceNft` (`coreService > 0`) for the target core type (e.g., after the initial contribution is accepted with base maturity `100`).
2. A model contribution proposal (`isModel(proposalId) == true`) is created and active on `AgentDAO`.
3. A voter with non-zero voting weight casts a `For` vote (`support == 1`) using `castVoteWithReasonAndParams`.

### Proof of concept

1. Assume the first contribution for core `1` has been accepted and minted a `ServiceNft` with initial maturity `100`.
2. A second model contribution proposal (`proposalId`) for core `1` is submitted to `AgentDAO`.
3. Instead of passing a 10-element array `[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]` (which produces `maturity = 315` and `impact = 215`), a validator encodes a 22-element array of `1`s:
   ```solidity
   uint8[] memory votes = new uint8[](22);
   for (uint256 i = 0; i < 22; i++) {
       votes[i] = 1;
   }
   bytes memory params = abi.encode(votes);
   agentDAO.castVoteWithReasonAndParams(proposalId, 1, "vote", params);
   ```
4. During `_calcMaturity(proposalId, votes)`:
   - Iteration 1 to 10 raises `eloA` from `1000` to `1215` (`eloB = 785`, `ratingDiff = 430`).
   - Iterations 11 to 22 continue executing `Elo.ratingChange(eloB, eloA, 100, 30)` without reverting because `ratingDiff` remains below `1126`.
   - After 22 iterations, `eloA` reaches `1555` (`eloB = 445`, `ratingDiff = 1110 < 1126`).
   - `battleElo(100, votes)` returns `100 + 1555 - 1000 = 655`.
5. `_proposalMaturities[proposalId]` accumulates `655 * weight`.
6. Upon execution of `proposalId`, `ServiceNft.mint` sets `_maturities[proposalId] = 655` and `_impacts[proposalId] = 655 - 100 = 555`, more than 2.5x the intended maximum impact of `215`.

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: Validators can bypass the intended `[0, 215]` per-proposal Elo rating gain cap and inflate `_proposalMaturities[proposalId]` up to `+555` per proposal (bounded only by `ratingDiff < 1126` in `Elo.sol` and the `10000` cap in `getMaturity`).
- **Business Impact**: Because `ServiceNft` impact scores determine contribution value and downstream token reward minting for contributors and datasets, inflating maturity and impact scores allows malicious or colluding validators/contributors to extract excess protocol rewards and corrupt the Elo progression of virtual agent cores.

### Remediation

Validate that the decoded `votes` array length matches the expected number of battle rounds (e.g., `10`) in `AgentDAO._updateMaturity` or `AgentDAO._calcMaturity`.

```solidity
    function _updateMaturity(address account, uint256 proposalId, uint256 weight, bytes memory params) internal {
        // Check is this a contribution proposal
        address contributionNft = IAgentNft(_agentNft).getContributionNft();
        address owner = IERC721(contributionNft).ownerOf(proposalId);
        if (owner == address(0)) {
            return;
        }

        bool isModel = IContributionNft(contributionNft).isModel(proposalId);
        if (!isModel) {
            return;
        }

        uint8[] memory votes = abi.decode(params, (uint8[]));
        require(votes.length == 10, "Invalid votes length");
        uint256 maturity = _calcMaturity(proposalId, votes);

        _proposalMaturities[proposalId] += (maturity * weight);

        emit ValidatorEloRating(proposalId, account, maturity, votes);
    }
```

## Finding 18: Unverified Contribution-to-Agent Association in `AgentDAO._calcMaturity` Allows Cross-DAO Maturity Manipulation

### Synopsis

`AgentDAO._calcMaturity` retrieves `virtualId` from `ContributionNft.tokenVirtualId(proposalId)` without verifying that the contribution's `virtualId` belongs to the executing `AgentDAO`. Because OpenZeppelin `Governor` proposal IDs do not include the governor address, an attacker can bind a shared `proposalId` to another agent's `virtualId` and manipulate maturity and impact scores.

### Technical details

In `AgentDAO`, each virtual persona (agent) has its own `AgentDAO` governor instance and an associated `virtualId`. When a contribution proposal is voted on with `support == 1` and non-empty `params`, `_castVote` invokes `_updateMaturity`, which calls `_calcMaturity(proposalId, votes)` at line 172 and accumulates `_proposalMaturities[proposalId] += (maturity * weight)` at line 174.

In `_calcMaturity` (lines 179–193):
```solidity
function _calcMaturity(uint256 proposalId, uint8[] memory votes) internal view returns (uint256) {
    address contributionNft = IAgentNft(_agentNft).getContributionNft();
    address serviceNft = IAgentNft(_agentNft).getServiceNft();
    uint256 virtualId = IContributionNft(contributionNft).tokenVirtualId(proposalId);
    uint8 core = IContributionNft(contributionNft).getCore(proposalId);
    uint256 coreService = IServiceNft(serviceNft).getCoreService(virtualId, core);
    ...
```
The function queries `virtualId` directly from `IContributionNft(contributionNft).tokenVirtualId(proposalId)` (line 182) and uses it to look up `IServiceNft(serviceNft).getCoreService(virtualId, core)` (line 184) and its baseline maturity `IServiceNft(serviceNft).getMaturity(coreService)` (line 188).

However, `GovernorUpgradeable.hashProposal` computes `proposalId` solely as:
```solidity
uint256(keccak256(abi.encode(targets, values, calldatas, descriptionHash)))
```
Neither the `AgentDAO` contract address nor the agent's `virtualId` is enforced by `hashProposal`. Consequently, the exact same `(targets, values, calldatas, description)` tuple submitted to two different `AgentDAO` instances (`DAO_A` for `virtualId_A` and `DAO_B` for `virtualId_B`) produces the identical `proposalId`.

Because neither `_updateMaturity` nor `_calcMaturity` verifies that `IContributionNft(contributionNft).getAgentDAO(virtualId) == address(this)` (or derives `virtualId` from the DAO's own voting token via `IAgentNft(_agentNft).stakingTokenToVirtualId(address(token()))`), a `proposalId` registered in `ContributionNft` under `virtualId_B` will still pass the checks in `DAO_A._updateMaturity` (`ownerOf(proposalId) != address(0)` and `isModel(proposalId) == true`). When validators vote on `DAO_A`, `_calcMaturity` fetches `virtualId_B` at line 182 and calculates the new maturity using `virtualId_B`'s active `coreService` maturity instead of `virtualId_A`'s.

When `DAO_A` subsequently executes the proposal calling `ServiceNft.mint(virtualId_A, descHash)`, `ServiceNft.mint` pulls the inflated `_maturities[proposalId]` from `DAO_A.getMaturity(proposalId)` (line 74) and calls `ServiceNft.updateImpact(virtualId_A, proposalId)` (line 79), subtracting `virtualId_A`'s lower previous service maturity from `virtualId_B`'s higher maturity to record an artificially inflated `_impacts[proposalId]` and `_maturities[proposalId]` for `virtualId_A`.

### Assumptions and preconditions

1. Two virtual agents exist: Agent A (`virtualId_A`, governed by `DAO_A`) with a low-maturity or initial (`coreService == 0`, maturity 100) core service, and Agent B (`virtualId_B`, governed by `DAO_B`) with a higher-maturity core service for the same `core` type (or vice versa to suppress maturity).
2. A proposer with sufficient proposal threshold (or admin/founder privileges on `DAO_B`) submits the identical proposal payload `(targets, values, calldatas, description)` to both `DAO_A` and `DAO_B`, and mints the `ContributionNft` for `proposalId` associated with `virtualId_B`.

### Proof of concept

1. Assume Agent A (`virtualId = 1`, `DAO_A`) has no prior model service for `core = 0` (`getCoreService(1, 0) == 0`, so its initial maturity should be locked at `100`).
2. Assume Agent B (`virtualId = 2`, `DAO_B`) has an existing model service for `core = 0` with maturity `1000`.
3. An attacker constructs a proposal with `targets = [serviceNft]`, `values = [0]`, `calldatas = [abi.encodeWithSignature("mint(uint256,bytes32)", 1, descHash)]`, and `description = desc`.
4. The attacker submits this proposal to `DAO_B` and `DAO_A`. Both calls return the exact same `proposalId` because `hashProposal` is independent of the governor address.
5. The attacker mints the `ContributionNft` for `proposalId` under `virtualId = 2` (`_contributionVirtualId[proposalId] = 2`, `isModel = true`, `core = 0`).
6. Validators cast votes on `DAO_A` with `support = 1` and battle results `votes = [1, 1, 1, 1, 1, 1, 1, 1, 1, 1]`.
7. During `DAO_A._castVote`, `_updateMaturity` calls `_calcMaturity(proposalId, votes)` (line 172).
8. At line 182, `_calcMaturity` reads `virtualId = IContributionNft(contributionNft).tokenVirtualId(proposalId)`, which returns `2` instead of `1`.
9. `_calcMaturity` reads `coreService = IServiceNft(serviceNft).getCoreService(2, 0)` (which has maturity `1000`) and computes `battleElo(1000, votes) = 1215`, instead of returning `100` for Agent A's first service.
10. When the proposal on `DAO_A` succeeds and executes `ServiceNft.mint(1, descHash)`, `ServiceNft` sets `_maturities[proposalId] = 1215` and `_impacts[proposalId] = 1215 - 0 = 1215` for Agent A (`virtualId = 1`), corrupting Agent A's service maturity and impact accounting.

### Impact assessment

**Severity: High**

- **Direct Technical Effect**: `AgentDAO._calcMaturity` uses the `coreService` maturity of an arbitrary foreign `virtualId` rather than the `virtualId` governed by the `AgentDAO` instance, corrupting `_proposalMaturities[proposalId]`.
- **Business and Protocol Impact**: When `ServiceNft.mint` executes on the target `AgentDAO`, the resulting `ServiceNft` receives an artificially inflated (or deflated) maturity and impact score. Because contribution rewards (`AgentToken` minting) and subsequent ELO calculations depend directly on service maturity and impact, an attacker can inflate impact scores to mint excess reward tokens or corrupt an agent's ELO progression.

### Remediation

Verify in `_updateMaturity` or `_calcMaturity` that the contribution's `virtualId` belongs to the current `AgentDAO` instance (`IContributionNft(contributionNft).getAgentDAO(virtualId) == address(this)`), or resolve `virtualId` directly from the DAO's own voting token via `IAgentNft(_agentNft).stakingTokenToVirtualId(address(token()))`.

```solidity
    function _updateMaturity(address account, uint256 proposalId, uint256 weight, bytes memory params) internal {
        // Check is this a contribution proposal
        address contributionNft = IAgentNft(_agentNft).getContributionNft();
        address owner = IERC721(contributionNft).ownerOf(proposalId);
        if (owner == address(0)) {
            return;
        }

        uint256 virtualId = IContributionNft(contributionNft).tokenVirtualId(proposalId);
        if (address(IContributionNft(contributionNft).getAgentDAO(virtualId)) != address(this)) {
            return;
        }

        bool isModel = IContributionNft(contributionNft).isModel(proposalId);
        if (!isModel) {
            return;
        }

        uint8[] memory votes = abi.decode(params, (uint8[]));
        uint256 maturity = _calcMaturity(proposalId, votes);

        _proposalMaturities[proposalId] += (maturity * weight);

        emit ValidatorEloRating(proposalId, account, maturity, votes);
    }

    function _calcMaturity(uint256 proposalId, uint8[] memory votes) internal view returns (uint256) {
        address contributionNft = IAgentNft(_agentNft).getContributionNft();
        address serviceNft = IAgentNft(_agentNft).getServiceNft();
        uint256 virtualId = IContributionNft(contributionNft).tokenVirtualId(proposalId);
        require(address(IContributionNft(contributionNft).getAgentDAO(virtualId)) == address(this), "Invalid proposal virtualId");
        uint8 core = IContributionNft(contributionNft).getCore(proposalId);
        uint256 coreService = IServiceNft(serviceNft).getCoreService(virtualId, core);
        // All services start with 100 maturity
        uint256 maturity = 100;
        if (coreService > 0) {
            maturity = IServiceNft(serviceNft).getMaturity(coreService);
            maturity = IEloCalculator(IAgentNft(_agentNft).getEloCalculator()).battleElo(maturity, votes);
        }

        return maturity;
    }
```

## Finding 19: Live `coreService` Lookup During Vote Casting in `AgentDAO._calcMaturity` Causes Inconsistent Proposal Maturity and Impact Calculation

### Synopsis

`AgentDAO._calcMaturity` reads the live `coreService` from `ServiceNft.getCoreService(virtualId, core)` upon every vote instead of using a fixed baseline per proposal. If another proposal for the same virtual persona and core executes during voting, baseline maturity drifts mid-vote, corrupting proposal maturity scores and reward minting.

### Technical details

When a validator casts a supporting vote with parameters on a model contribution proposal in `AgentDAO._castVote`, `_updateMaturity` invokes `_calcMaturity(proposalId, votes)` and accumulates `maturity * weight` into `_proposalMaturities[proposalId]`:

```solidity
uint256 coreService = IServiceNft(serviceNft).getCoreService(virtualId, core);
// All services start with 100 maturity
uint256 maturity = 100;
if (coreService > 0) {
    maturity = IServiceNft(serviceNft).getMaturity(coreService);
    maturity = IEloCalculator(IAgentNft(_agentNft).getEloCalculator()).battleElo(maturity, votes);
}
```

In `ServiceNft.mint`, executing a model contribution proposal updates `_coreServices[virtualId][_cores[proposalId]] = proposalId` (which can also happen immediately during voting via `_tryAutoExecute`). Because `_calcMaturity` queries `IServiceNft(serviceNft).getCoreService(virtualId, core)` dynamically at the time each individual vote is cast rather than referencing a fixed baseline (such as the contribution's parent ID or a snapshot taken at proposal creation):

1. **Mid-Proposal Baseline Drift Across Votes:** Early voters on Proposal B have their ELO calculations anchored to the old `coreService` (or `100` if `coreService == 0`), while voters casting votes on Proposal B after Proposal A executes have their ELO calculations anchored to Proposal A's newly minted `ServiceNft` maturity. `getMaturity(proposalId)` then averages these incompatible baselines (`_proposalMaturities[proposalId] / forVotes`).
2. **Baseline Interference on Execution (`ServiceNft.updateImpact`):** When Proposal B is finally executed in `ServiceNft.mint`, `updateImpact` computes `rawImpact = _maturities[proposalId] - _maturities[prevServiceId]` using the latest `_coreServices[virtualId][core]` (Proposal A). If Proposal B's votes were cast before Proposal A executed, Proposal B's `_maturities[proposalId]` was calculated against the pre-Proposal-A baseline, resulting in an understated or zero `rawImpact` and depriving the model and dataset contributors of their `AgentToken` rewards in `Minter.mint`. Conversely, if Proposal A had lower maturity than the prior service, Proposal B's `rawImpact` becomes artificially inflated.

### Assumptions and preconditions

1. Two model contribution proposals (Proposal A and Proposal B) targeting the same `virtualId` and `core` exist concurrently, or Proposal A is awaiting execution while Proposal B is being voted on.
2. Proposal A is executed (either manually via `execute` or automatically via `_tryAutoExecute` when quorum is reached) before Proposal B finishes voting or before Proposal B is executed.

### Proof of concept

1. Assume `virtualId = 1`, `core = 0`, and initially no core service exists (`getCoreService(1, 0) == 0`), or the initial service has `maturity = 200`.
2. Proposal A and Proposal B are created for `virtualId = 1, core = 0` as model contributions.
3. Validator 1 (weight 100) votes on Proposal B with `[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]` (+215 ELO). In `_calcMaturity`, `coreService` has maturity `200`, so `_calcMaturity` returns `200 + 215 = 415`. `_proposalMaturities[proposalB]` becomes `415 * 100 = 41,500`.
4. Proposal A receives enough votes and executes (e.g., via `_tryAutoExecute`). Proposal A's maturity is `415`, so `ServiceNft.mint` sets `_coreServices[1][0] = proposalA` with `_maturities[proposalA] = 415`.
5. Validator 2 (weight 100) now votes on Proposal B with the exact same battle results (`+215` ELO). Because `getCoreService(1, 0)` now returns `proposalA` (`maturity = 415`), `_calcMaturity` returns `415 + 215 = 630`. `_proposalMaturities[proposalB]` becomes `41,500 + 63,000 = 104,500`.
6. When Proposal B executes, `getMaturity(proposalB)` returns `104,500 / 200 = 522` (a hybrid of two different baselines), and `ServiceNft.updateImpact` subtracts `_maturities[proposalA]` (`415`), yielding an impact of `522 - 415 = 107` instead of `215` (or `0` if all of Proposal B's votes were cast prior to step 4).

### Impact assessment

- **Severity:** Medium / High.
- **Technical Impact:** The maturity score of a proposal becomes dependent on transaction ordering of other proposals' executions relative to individual votes, corrupting both `_proposalMaturities[proposalId]` and the resulting `_impacts[proposalId]` / `_impacts[datasetId]`.
- **Business Impact:** Contributors receive inaccurate (under-minted or over-minted) `AgentToken` payouts in `Minter.mint`, and the ELO progression of core services in `ServiceNft` is distorted.

### Remediation

Anchor the baseline service for a contribution proposal to a fixed reference determined when the contribution is created (such as `IContributionNft(contributionNft).getParentId(proposalId)` or snapshotting `coreService` on the first vote / proposal creation) and ensure `coreService > 0` is consistent with the initial `100` maturity battle calculation.

```solidity
    function _calcMaturity(uint256 proposalId, uint8[] memory votes) internal view returns (uint256) {
        address contributionNft = IAgentNft(_agentNft).getContributionNft();
        address serviceNft = IAgentNft(_agentNft).getServiceNft();
        uint256 coreService = IContributionNft(contributionNft).getParentId(proposalId);
        // All services start with 100 maturity
        uint256 maturity = 100;
        if (coreService > 0) {
            maturity = IServiceNft(serviceNft).getMaturity(coreService);
            maturity = IEloCalculator(IAgentNft(_agentNft).getEloCalculator()).battleElo(maturity, votes);
        }

        return maturity;
    }
```

## Finding 20: Arithmetic Underflow in `AgentDAO._calcMaturity` via `EloCalculator.battleElo` When Evaluating Lower-Rated Model Contributions

### Synopsis

In `AgentDAO._calcMaturity`, services start with an initial maturity of `100`, which is later passed as `currentRating` to `EloCalculator.battleElo`. Because `battleElo` computes `currentRating + eloA - 1000` where `eloA` can drop by up to `215` points over 10 battles, unfavorable battle votes cause an arithmetic underflow and revert vote casting.

### Technical details

When the first model contribution for a `(virtualId, core)` pair is proposed and approved, `AgentDAO._calcMaturity` (line 186) assigns it a baseline maturity score of `100`:

```solidity
uint256 coreService = IServiceNft(serviceNft).getCoreService(virtualId, core);
// All services start with 100 maturity
uint256 maturity = 100;
if (coreService > 0) {
    maturity = IServiceNft(serviceNft).getMaturity(coreService);
    maturity = IEloCalculator(IAgentNft(_agentNft).getEloCalculator()).battleElo(maturity, votes);
}
```

Once that first `ServiceNft` is minted, `_coreServices[virtualId][core]` is nonzero and its stored maturity in `ServiceNft._maturities` is `100`.

When a subsequent model contribution for the same `virtualId` and `core` is voted on with `support == 1`, `_updateMaturity` calls `_calcMaturity(proposalId, votes)`. Since `coreService > 0`, line 188 reads `maturity = 100` and line 189 invokes `IEloCalculator.battleElo(100, votes)`.

In `EloCalculator.battleElo` (lines 39–55):

```solidity
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
```

`eloA` and `eloB` start at `1000`. Over a standard 10-battle sequence, if the new model loses battles against the current core service (`negative == true`), `eloA` decreases by up to `215` points (reaching `785` for 10 losses, symmetric to the `+215` gain for 10 wins tested in `test/elo.js`). At line 54, `battleElo` computes:

$$\text{currentRating} + \text{eloA} - 1000$$

Whenever the net ELO loss `1000 - eloA` exceeds `currentRating` (which is `100` for any core whose current service is at the initial maturity, or any service with maturity `< 215`), `currentRating + eloA < 1000`. Under Solidity `^0.8.20` checked arithmetic, the subtraction `- 1000` reverts with an arithmetic underflow (`Panic(0x11)`). Furthermore, if repeated losses or larger battle arrays drive `eloB - eloA >= 800` (or `>= 1126`), `Elo.ratingChange` itself reverts on its rating-difference domain checks (`require(ratingDiff < 800)` / `require(ratingDiff < 1126)`).

### Assumptions and preconditions

1. A `VirtualPersona` agent has at least one minted `ServiceNft` for a given `core`, whose maturity is at or near the initial value of `100` (or `< 215`).
2. A new model contribution proposal is created for the same `virtualId` and `core`.
3. A validator casts a vote in support (`support == 1`) with `params` encoding battle outcomes (`votes`) in which the candidate model loses enough battles that `1000 - eloA > maturity` (e.g., losing 6 or more out of 10 battles when `maturity == 100`).

### Proof of concept

1. Proposal 1 (the first model contribution for `virtualId = 1, core = 0`) is proposed, voted on by a validator, and executed. Because `coreService == 0` during voting, `_calcMaturity` returns `100`, and `ServiceNft.mint` records `_maturities[proposalId_1] = 100` and `_coreServices[1][0] = proposalId_1`.
2. Proposal 2 (a second model contribution for `virtualId = 1, core = 0`) is submitted.
3. A validator calls `castVoteWithReasonAndParams(proposalId_2, 1, "evaluated", abi.encode(votes))` where `votes` represents 10 losses for the new model against the existing core service.
4. `_castVote` calls `_updateMaturity`, which calls `_calcMaturity(proposalId_2, votes)`.
5. `_calcMaturity` reads `coreService = proposalId_1` and `maturity = IServiceNft(serviceNft).getMaturity(proposalId_1) = 100`, then calls `battleElo(100, votes)`.
6. Inside `battleElo`, `eloA` drops from `1000` to `785` (`-215` ELO).
7. Line 54 evaluates `100 + 785 - 1000 = 885 - 1000`, which reverts with `Panic(0x11)` (arithmetic underflow), blocking the validator from recording a valid evaluation vote with params.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** Any `for` vote carrying battle evaluation parameters where the ELO delta is negative by more than the current service's `maturity` (such as `-101` to `-215` against an initial `100`-maturity service) reverts with an arithmetic underflow.
- **Business Implications:** Validators cannot submit honest battle evaluations when a proposed model performs worse than an initial or low-maturity core service by more than its current maturity score, breaking the rating computation integrity and governance voting workflow for model contributions.

### Remediation

Ensure the initial maturity and rating floor in `AgentDAO._calcMaturity` (or the subtraction in `battleElo`) cannot underflow when `eloA < 1000`. For example, initialize base maturity at `1000` to match `EloCalculator`'s initial `1000` rating domain, or clamp the updated maturity to a minimum floor so that negative ELO deltas do not underflow:

```solidity
    function _calcMaturity(uint256 proposalId, uint8[] memory votes) internal view returns (uint256) {
        address contributionNft = IAgentNft(_agentNft).getContributionNft();
        address serviceNft = IAgentNft(_agentNft).getServiceNft();
        uint256 virtualId = IContributionNft(contributionNft).tokenVirtualId(proposalId);
        uint8 core = IContributionNft(contributionNft).getCore(proposalId);
        uint256 coreService = IServiceNft(serviceNft).getCoreService(virtualId, core);
        // Start at 1000 (or offset by 1000 before calling battleElo) so negative ELO changes cannot underflow
        uint256 maturity = 100;
        if (coreService > 0) {
            maturity = IServiceNft(serviceNft).getMaturity(coreService);
            uint256 rawMaturity = IEloCalculator(IAgentNft(_agentNft).getEloCalculator()).battleElo(
                maturity + 1000,
                votes
            );
            maturity = rawMaturity > 1000 ? rawMaturity - 1000 : 0;
        }

        return maturity;
    }
```

## Finding 21: Unbounded `votes` array length in `AgentDAO._calcMaturity` allows validators to inflate model maturity and token rewards

### Synopsis

In `AgentDAO._calcMaturity`, the voter-supplied `votes` array is forwarded to `IEloCalculator.battleElo` without validating its length. Because the ELO calculation iterates over the entire `votes` array, a voter can submit more than the expected 10 battle outcomes to artificially inflate a model contribution's maturity score, resulting in excessive impact and reward token minting.

### Technical details

When a validator votes in favor of a model contribution proposal via `castVoteWithReasonAndParams`, `AgentDAO._castVote` invokes `_updateMaturity`, which ABI-decodes the arbitrary `params` bytes into `uint8[] memory votes` and passes it to `_calcMaturity(proposalId, votes)` (lines 171–172).

In `AgentDAO._calcMaturity` (lines 184–193), if a previous `coreService` exists for the virtual agent's core (`coreService > 0`), the function fetches the current service maturity and computes the new maturity via:

```solidity
maturity = IEloCalculator(IAgentNft(_agentNft).getEloCalculator()).battleElo(maturity, votes);
```

As demonstrated in the project's test suite (`"should get max 215 elo rating for one sided votes"`), a validator evaluation is intended to consist of 10 battle results (`votes.length == 10`), which caps the maximum ELO increase from a single one-sided evaluation at `215` (raising `eloA` from `1000` to `1215`).

However, neither `AgentDAO._calcMaturity` nor `EloCalculator.battleElo` enforces any constraint on `votes.length`. `EloCalculator.battleElo` simply loops over `0 .. battles.length - 1`, adjusting `eloA` and `eloB` on every element until `Elo.ratingChange` reverts when `ratingDiff >= 800` (i.e., when `eloA - eloB >= 800`, or `eloA >= 1400`). By passing a longer array of wins (`1`s), a voter can increase `eloA` up to nearly `1400` (+399 maturity instead of the intended maximum of +215).

Because `ServiceNft.mint` records `_maturities[proposalId] = IAgentDAO(info.dao).getMaturity(proposalId)` and `ServiceNft.updateImpact` computes `rawImpact = _maturities[proposalId] - _maturities[prevServiceId]`, this inflated maturity directly translates into a larger impact score for the service NFT and consequently inflates the amount of `AgentToken` minted to the model owner, dataset owner, and IP vault in `Minter.mint`.

### Assumptions and preconditions

1. The virtual persona already has an initial model service NFT for the target `core` (`coreService > 0`).
2. A model contribution proposal is active in `AgentDAO`.
3. A voter with voting power casts a `For` vote (`support == 1`) via `castVoteWithReasonAndParams` or `castVoteWithReasonAndParamsBySig` with `params` encoding a `uint8[]` array containing more than 10 elements.

### Proof of concept

1. Assume `coreService` for `virtualId = 1, core = 0` currently has a maturity of `100`.
2. A new model contribution proposal `proposalId` is created for `virtualId = 1, core = 0`.
3. Instead of submitting the standard 10 battle outcomes `[1, 1, 1, 1, 1, 1, 1, 1, 1, 1]` (which produces `eloA = 1215`, `maturity = 315`, and `impact = 215`), a validator encodes an array of 50 wins (`uint8[]` of fifty `1`s) into `params` and calls `castVoteWithReasonAndParams(proposalId, 1, "", params)`.
4. During `battleElo(100, votes)`, each iteration increases `eloA` and decreases `eloB` by the rounded-up ELO delta (at least `1` per iteration as `eloA - eloB` approaches `800`), driving `eloA` up to `1399` (`ratingDiff = 798 < 800`).
5. `_calcMaturity` returns `100 + 1399 - 1000 = 499` instead of the intended maximum `315`.
6. When the proposal is executed and `Minter.mint(proposalId)` is called, the service impact is recorded as `499 - 100 = 399` (instead of `215`), minting ~85.5% more `AgentToken` rewards than the intended maximum for a single upgrade step.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Validators can bypass the intended maximum ELO rating change per contribution proposal (+215) and inflate `_proposalMaturities[proposalId]` up to +399 per proposal.
- **Business Impact:** Direct inflation of `ServiceNft` maturity and impact scores, leading to unauthorized excess minting of `AgentToken` rewards in `Minter.mint` for model contributors, dataset contributors, and the IP vault.

### Remediation

Validate that the `votes` array length matches the expected number of evaluation battles (e.g., `10`) before computing the ELO rating in `AgentDAO._calcMaturity`.

```solidity
    function _calcMaturity(uint256 proposalId, uint8[] memory votes) internal view returns (uint256) {
        address contributionNft = IAgentNft(_agentNft).getContributionNft();
        address serviceNft = IAgentNft(_agentNft).getServiceNft();
        uint256 virtualId = IContributionNft(contributionNft).tokenVirtualId(proposalId);
        uint8 core = IContributionNft(contributionNft).getCore(proposalId);
        uint256 coreService = IServiceNft(serviceNft).getCoreService(virtualId, core);
        // All services start with 100 maturity
        uint256 maturity = 100;
        if (coreService > 0) {
            require(votes.length == 10, "Invalid votes length");
            maturity = IServiceNft(serviceNft).getMaturity(coreService);
            maturity = IEloCalculator(IAgentNft(_agentNft).getEloCalculator()).battleElo(maturity, votes);
        }

        return maturity;
    }
```

## Finding 22: Mismatched Scoring and Voting Populations in `AgentDAO.getMaturity` Dilutes Model Maturity and Token Rewards

### Synopsis

`AgentDAO.getMaturity` normalizes `_proposalMaturities[proposalId]` using the total `forVotes` of the proposal rather than the total voting weight of voters who actually submitted maturity ratings. Consequently, any `For` vote cast without `params` increases the `forVotes` denominator without incrementing `_proposalMaturities`, artificially lowering the service's maturity, impact score, and minted token rewards.

### Technical details

In `AgentDAO._castVote`, when a voter casts a supporting vote (`support == 1`), `super._castVote` records the voter's `weight` in the proposal's `forVotes` tally. However, `_updateMaturity` is only invoked conditionally when `params.length > 0` (line 138):

```solidity
if (params.length > 0 && support == 1) {
    _updateMaturity(account, proposalId, weight, params);
}
```

Furthermore, within `_updateMaturity`, `_proposalMaturities[proposalId]` is only incremented by `maturity * weight` if the `proposalId` already exists in `ContributionNft` as a model contribution (`isModel(proposalId) == true`).

When the proposal is executed and `ServiceNft.mint` queries `IAgentDAO(info.dao).getMaturity(proposalId)` at line 74, `AgentDAO.getMaturity` computes the weighted average maturity at lines 195–197 as:

```solidity
function getMaturity(uint256 proposalId) public view returns (uint256) {
    (, uint256 forVotes, ) = proposalVotes(proposalId);
    return Math.min(10000, _proposalMaturities[proposalId] / forVotes);
}
```

Because `GovernorUpgradeable` publicly exposes `castVote` and `castVoteWithReason` (which pass empty `params`), as well as `castVoteWithReasonAndParams`, any validator who votes `For` with empty `params` (or before the `ContributionNft` is minted) contributes their `weight` to `forVotes` while contributing `0` to `_proposalMaturities[proposalId]`. In addition, if `getMaturity` is called when `forVotes == 0`, the division reverts due to a division-by-zero panic.

Because `ServiceNft.updateImpact` calculates `rawImpact` as `_maturities[proposalId] - _maturities[prevServiceId]` (or `0` if `_maturities[proposalId] <= _maturities[prevServiceId]`), diluting the resulting maturity directly reduces or zeroes out the impact score of the new service NFT and the corresponding `AgentToken` rewards minted in `Minter.mint`.

### Assumptions and preconditions

1. A model contribution proposal is created in `AgentDAO` to call `ServiceNft.mint`.
2. At least one voter with non-zero voting weight casts a `For` vote (`support == 1`) without `params` (e.g., via `castVote(proposalId, 1)` or `castVoteWithReason(proposalId, 1, reason)`), or votes before the `ContributionNft` is minted.

### Proof of concept

1. Suppose a Virtual Persona currently has a core service with maturity `315`.
2. A contributor submits a new model contribution proposal (`proposalId`) and mints the corresponding `ContributionNft`.
3. Validator A (with voting weight `100,000`) votes in favor via `castVoteWithReasonAndParams(proposalId, 1, "", encodedVotes)` with all-win battle results, yielding a calculated maturity of `315 + 215 = 530`.
   - `_proposalMaturities[proposalId]` becomes `530 * 100,000 = 53,000,000`.
   - `forVotes` becomes `100,000`.
4. Validator B (also with voting weight `100,000`) votes in favor via `castVote(proposalId, 1)` (so `params` is empty).
   - `_updateMaturity` is skipped because `params.length == 0`.
   - `_proposalMaturities[proposalId]` remains `53,000,000`.
   - `forVotes` increases to `200,000`.
5. When the proposal executes `ServiceNft.mint`, `AgentDAO.getMaturity(proposalId)` computes:
   - `53,000,000 / 200,000 = 265`.
6. In `ServiceNft.updateImpact`, because the new maturity (`265`) is less than the previous service maturity (`315`), `rawImpact` is set to `0`, and `coreServices` is updated to the new service with maturity `265`.
7. When `Minter.mint(proposalId)` is called, the model contributor and dataset contributor receive `0` `AgentToken` rewards instead of the expected reward for a `215` impact improvement, and the core service's baseline maturity is degraded from `315` to `265`.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: `getMaturity` under-calculates the weighted average maturity whenever any `support == 1` vote does not trigger `_updateMaturity` (and reverts with division by zero when `forVotes == 0`).
- **Business Implications**: Model and dataset contributors lose part or all of their `AgentToken` rewards in `Minter.mint`, and the Virtual Persona's active core service maturity is corrupted downward, affecting future contribution evaluations.

### Remediation

Track the cumulative voting weight of validators who actually record a maturity score for each proposal (or require all `support == 1` votes on model proposals to include valid maturity parameters), and normalize `_proposalMaturities[proposalId]` by that scored weight while handling the zero-weight case safely.

```solidity
    mapping(uint256 => uint256) private _proposalMaturityVotes;

    function _updateMaturity(address account, uint256 proposalId, uint256 weight, bytes memory params) internal {
        // Check is this a contribution proposal
        address contributionNft = IAgentNft(_agentNft).getContributionNft();
        address owner = IERC721(contributionNft).ownerOf(proposalId);
        if (owner == address(0)) {
            return;
        }

        bool isModel = IContributionNft(contributionNft).isModel(proposalId);
        if (!isModel) {
            return;
        }

        uint8[] memory votes = abi.decode(params, (uint8[]));
        uint256 maturity = _calcMaturity(proposalId, votes);

        _proposalMaturities[proposalId] += (maturity * weight);
        _proposalMaturityVotes[proposalId] += weight;

        emit ValidatorEloRating(proposalId, account, maturity, votes);
    }

    function getMaturity(uint256 proposalId) public view returns (uint256) {
        uint256 scoredVotes = _proposalMaturityVotes[proposalId];
        if (scoredVotes == 0) {
            return 0;
        }
        return Math.min(10000, _proposalMaturities[proposalId] / scoredVotes);
    }
```
