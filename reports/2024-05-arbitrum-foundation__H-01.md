# Audit report — 2024-05-arbitrum-foundation/H-01

## Finding 1: Hardcoded empty state in `RollupCore.genesisAssertionHash` returns an incorrect genesis hash for rollups initialized with a custom genesis state

### Synopsis

`RollupCore.genesisAssertionHash()` hardcodes an empty `AssertionState` to compute the genesis assertion hash, whereas `RollupAdminLogic.initialize()` computes the actual genesis assertion hash from `config.genesisAssertionState`. Whenever a rollup is initialized or upgraded with a non-empty genesis state, `genesisAssertionHash()` returns a hash that does not exist in `_assertions`.

### Technical details

During rollup initialization in `RollupAdminLogic.initialize` (lines 63–69), the genesis assertion hash (`genesisHash`) is derived from the configured `config.genesisAssertionState`:

```solidity
bytes32 parentAssertionHash = bytes32(0);
bytes32 inboxAcc = bytes32(0);
bytes32 genesisHash = RollupLib.assertionHash({
    parentAssertionHash: parentAssertionHash,
    afterStateHash: config.genesisAssertionState.hash(),
    inboxAcc: inboxAcc
});
```

This `genesisHash` is then registered in `_assertions[genesisHash]` and set as `_latestConfirmed` via `RollupCore.initializeCore` (lines 86 and 225–230). Similarly, rollups migrating to BoLD via upgrade actions or initialized from a non-zero initial execution state (`config.genesisAssertionState` and `config.genesisInboxCount`) have a non-empty `genesisAssertionState` (e.g., populated `GlobalState` or `endHistoryRoot`).

However, `RollupCore.genesisAssertionHash()` (lines 520–530) is implemented as an `external pure` function that ignores the configured genesis state and instead constructs a default zeroed `GlobalState` and `AssertionState`:

```solidity
function genesisAssertionHash() external pure returns (bytes32) {
    GlobalState memory emptyGlobalState;
    AssertionState memory emptyAssertionState = AssertionState(emptyGlobalState, MachineStatus.FINISHED, bytes32(0));
    bytes32 parentAssertionHash = bytes32(0);
    bytes32 inboxAcc = bytes32(0);
    return RollupLib.assertionHash({
        parentAssertionHash: parentAssertionHash,
        afterState: emptyAssertionState,
        inboxAcc: inboxAcc
    });
}
```

Because `genesisAssertionHash()` does not use the actual `genesisAssertionState` (or read a stored genesis hash set during `initializeCore`), any external consumer, validator, or contract querying `IRollupCore.genesisAssertionHash()` on a rollup with a non-empty `config.genesisAssertionState` will receive an invalid assertion hash that does not match the actual genesis assertion stored in `_assertions`.

### Assumptions and preconditions

1. The rollup is initialized via `RollupAdminLogic.initialize` (or upgraded) with a `config.genesisAssertionState` that differs from `AssertionState(GlobalState([bytes32(0), bytes32(0)], [uint64(0), uint64(0)]), MachineStatus.FINISHED, bytes32(0))`.
2. An external caller, off-chain validator/node, or integrated contract calls `genesisAssertionHash()` expecting the hash of the rollup's genesis assertion.

### Proof of concept

1. Deploy and initialize a rollup via `RollupCreator.createRollup` where `deployParams.config.genesisAssertionState` has a non-zero `endHistoryRoot` or non-zero `globalState` (for example, `globalState.bytes32Vals[0] = keccak256("genesis_block")` and `u64Vals[0] = 5`).
2. `RollupAdminLogic.initialize` computes `genesisHash = RollupLib.assertionHash(bytes32(0), config.genesisAssertionState.hash(), bytes32(0))` and stores the initial `AssertionNode` at `_assertions[genesisHash]`, setting `_latestConfirmed = genesisHash`.
3. Call `rollup.latestConfirmed()`. It returns `genesisHash`, and `rollup.getAssertion(genesisHash).status` is `AssertionStatus.Confirmed`.
4. Call `rollup.genesisAssertionHash()`. It computes the hash using `emptyAssertionState`, returning `emptyGenesisHash != genesisHash`.
5. Querying `rollup.getAssertion(rollup.genesisAssertionHash()).status` returns `AssertionStatus.NoAssertion`, demonstrating that `genesisAssertionHash()` does not identify the rollup's actual genesis assertion.

### Impact assessment

**Severity: Medium**

Any off-chain infrastructure, validator software, or on-chain contract relying on `IRollupCore.genesisAssertionHash()` to identify the root of the assertion chain or verify genesis lineage will obtain a non-existent assertion hash whenever `config.genesisAssertionState` is non-empty (such as chains upgraded to BoLD or initialized from a non-zero state).

### Remediation

Store the genesis assertion hash in a state variable in `RollupCore` when `initializeCore` is called (or remove `genesisAssertionHash()` if unneeded), and return the stored value instead of recomputing it from a hardcoded empty state.

```solidity
function genesisAssertionHash() external view returns (bytes32);
```

```solidity
bytes32 private _genesisAssertionHash;

function initializeCore(AssertionNode memory initialAssertion, bytes32 assertionHash) internal {
    __Pausable_init();
    initialAssertion.status = AssertionStatus.Confirmed;
    _assertions[assertionHash] = initialAssertion;
    _latestConfirmed = assertionHash;
    _genesisAssertionHash = assertionHash;
}

function genesisAssertionHash() external view returns (bytes32) {
    return _genesisAssertionHash;
}
```

## Finding 2: Decreasing `baseStake` in `RollupAdminLogic.setBaseStake` Breaks Lineage Stake Accounting and Causes Rollup Insolvency

### Synopsis

`RollupAdminLogic.setBaseStake` allows reducing `baseStake` without restriction. When `baseStake` is lowered, a staker on an assertion requiring the old higher stake can have a child assertion created under the new lower stake, freeing their higher stake to move to another branch while `loserStakeEscrow` still confiscates the old higher stake.

### Technical details

In `RollupCore` and `RollupUserLogic`, the staking mechanism relies on the invariant that every unconfirmed assertion lineage holds at least one full required stake locked at its tip, so that when a lineage forks at a parent assertion, the parent's `requiredStake` can be safely transferred to `loserStakeEscrow` while leaving enough tokens in the contract to refund the eventual winning lineage.

Specifically:
1. When an assertion is created in `RollupCore.createNewAssertion` (lines 488–497), its `configHash` snapshots the current `baseStake` as `requiredStake`, which governs how much stake is needed to create any direct child of that assertion.
2. In `RollupUserLogic.stakeOnNewAssertion` (lines 182–216), a validator creating a child of `prevAssertion` must have `amountStaked(msg.sender) >= assertion.beforeStateData.configData.requiredStake` (the `requiredStake` stored in `prevAssertion.configHash`).
3. A validator is allowed to move their stake to a new assertion on any branch if their `latestStakedAssertion` already has at least one child (`getAssertionStorage(lastAssertion).firstChildBlock > 0` at line 196). This assumes that the child's staker has locked at least as much stake on that lineage as was required to create `lastAssertion`.
4. When a second child of `prevAssertion` is created (`!getAssertionStorage(newAssertionHash).isFirstChild` at line 210), the contract immediately transfers `prevAssertion`'s `requiredStake` (`assertion.beforeStateData.configData.requiredStake`) of `stakeToken` to `loserStakeEscrow` (line 215).

However, `RollupAdminLogic.setBaseStake` (lines 223–226) permits setting `baseStake` to a value smaller than the previous `baseStake` (including `0`). If `baseStake` is reduced from $S_{\text{high}}$ to $S_{\text{low}}$ ($S_{\text{low}} < S_{\text{high}}$):
- Suppose parent assertion $A$ was created when `baseStake` was $S_{\text{high}}$, so $A.\text{requiredStake} = S_{\text{high}}$.
- Validator $V_1$ (with $S_{\text{high}}$ staked) creates child assertion $B$ under $A$. Because `baseStake` is now $S_{\text{low}}$, $B$ is created with $B.\text{requiredStake} = S_{\text{low}}$.
- Validator $V_2$ (with only $S_{\text{low}}$ staked) creates child assertion $C$ under $B$. This sets $B.\text{firstChildBlock} > 0$.
- Because $B.\text{firstChildBlock} > 0$, $V_1$ is no longer locked to $B$ and can reuse its $S_{\text{high}}$ stake to create (or stake on) a rival branch $D$ under $A$.
- Creating $D$ (the second child of $A$) transfers $A.\text{requiredStake} = S_{\text{high}}$ from the Rollup contract to `loserStakeEscrow`, even though the losing branch $B \to C$ only has $V_2$'s $S_{\text{low}}$ locked in it while $V_1$'s $S_{\text{high}}$ is now on the winning branch $D$.

This creates a token deficit of $S_{\text{high}} - S_{\text{low}}$ in the Rollup contract.

### Assumptions and preconditions

1. An assertion $A$ exists whose `configHash` commits to `requiredStake` $= S_{\text{high}}$.
2. The rollup owner/admin calls `setBaseStake(S_low)` with $S_{\text{low}} < S_{\text{high}}$ before $A$'s children are finalized.
3. Validators can stake and create assertions on the rollup.

### Proof of concept

1. Assume `baseStake` is initially `100e18` tokens, and assertion $A$ is created with $A.\text{requiredStake} = 100\text{e18}$.
2. The admin calls `setBaseStake(10e18)`, lowering `baseStake` to `10e18`.
3. An attacker using account $V_1$ stakes `100e18` tokens and calls `stakeOnNewAssertion` to create an invalid assertion $B$ as the first child of $A$ ($A.\text{requiredStake} = 100\text{e18}$). Assertion $B$ is recorded with $B.\text{requiredStake} = \text{baseStake} = 10\text{e18}$.
4. After `minimumAssertionPeriod`, the attacker using account $V_2$ stakes `10e18` tokens and calls `stakeOnNewAssertion` to create assertion $C$ as the first child of $B$ ($B.\text{requiredStake} = 10\text{e18}$). This sets $B.\text{firstChildBlock} = \text{block.number} > 0$.
5. Because $B.\text{firstChildBlock} > 0$, $V_1$ passes the check `getAssertionStorage(lastAssertion).firstChildBlock > 0` in `stakeOnNewAssertion` and calls `stakeOnNewAssertion` (without depositing any additional tokens) to create the valid assertion $D$ as the second child of $A$.
6. Since $D$ is not the first child of $A$, `stakeOnNewAssertion` transfers $A.\text{requiredStake} = 100\text{e18}$ tokens from the Rollup contract to `loserStakeEscrow`.
7. Out of the `110e18` tokens deposited by $V_1$ (`100e18`) and $V_2$ (`10e18`), `100e18` tokens have been sent to `loserStakeEscrow`, leaving only `10e18` tokens in the Rollup contract, while $V_1$ is staked on the valid assertion $D$ with `amountStaked = 100e18`.
8. Once $D$ is confirmed, $V_1$ is eligible to withdraw `100e18` tokens:
   - If other honest stakers have funds in the Rollup contract, $V_1$ withdraws `100e18` tokens, stealing `90e18` tokens from other stakers (net profit or slashing bypass of `90e18` tokens at the cost of $V_2$'s `10e18` stake).
   - If no other staker funds exist in the contract, $V_1$ (or an honest validator in an analogous scenario where $V_1$ moves onto the honest branch after $D$ is created) cannot withdraw their stake because the contract is insolvent by `90e18` tokens.

### Impact assessment

**Severity: High**

- **Direct Technical Effect:** Decreasing `baseStake` breaks the core staking invariant that a forked lineage locks at least the parent's `requiredStake` on each branch. A staker on an invalid branch can substitute their $S_{\text{high}}$ stake with a child staker's $S_{\text{low}}$ stake and migrate their $S_{\text{high}}$ stake to the valid branch, while `loserStakeEscrow` still receives $S_{\text{high}}$.
- **Business & Protocol Implications:**
  1. **Theft of Honest Stakers' Funds / Contract Insolvency:** The Rollup contract suffers a balance shortfall of $S_{\text{high}} - S_{\text{low}}$ per fork, allowing an attacker to drain other stakers' deposited tokens upon withdrawing or leaving honest validators unable to withdraw their stakes.
  2. **Slashing Bypass:** A malicious validator can propose invalid assertions under an $S_{\text{high}}$ requirement while only risking $S_{\text{low}}$ in economic penalty.

### Remediation

Prevent `setBaseStake` from decreasing `baseStake` while unconfirmed assertions or a latest confirmed assertion with a higher `requiredStake` can still be built upon, or enforce that `baseStake` can only be increased (unless all pending assertions are resolved and the latest confirmed assertion's config is updated while paused).

The simplest and safest fix in `RollupAdminLogic.setBaseStake` is to require that `newBaseStake` is not less than the current `baseStake`:

```solidity
    function setBaseStake(uint256 newBaseStake) external override {
        require(newBaseStake >= baseStake, "STAKE_CANNOT_DECREASE");
        baseStake = newBaseStake;
        emit OwnerFunctionCalled(12);
    }
```

Alternatively, if decreasing `baseStake` must be supported, `stakeOnNewAssertion` should store the stake requirement of each assertion or prevent a staker from moving off `lastAssertion` when `amountStaked(msg.sender)` exceeds the stake locked by the child assertion's staker, or `createNewAssertion` should cap the confiscated stake to the minimum `requiredStake` along the losing branch.

## Finding 3: Missing access control in `RollupAdminLogic.forceRefundStaker` allows unauthorized callers to forcefully unstake inactive validators while paused

### Synopsis

`RollupAdminLogic.forceRefundStaker` is an administrative function intended to be restricted to the rollup owner, as indicated by its placement in `IRollupAdmin` and the emission of `OwnerFunctionCalled(22)`. However, the function only applies the `whenPaused` modifier and omits the `onlyOwner` access control check, allowing unauthorized invocation when paused.

### Technical details

In `src/rollup/RollupAdminLogic.sol`, `forceRefundStaker` is defined as part of the administrative logic contract (`IRollupAdmin`):

```solidity
function forceRefundStaker(address[] calldata staker) external override whenPaused {
    require(staker.length > 0, "EMPTY_ARRAY");
    for (uint256 i = 0; i < staker.length; i++) {
        requireInactiveStaker(staker[i]);
        reduceStakeTo(staker[i], 0);
    }
    emit OwnerFunctionCalled(22);
}
```

Despite emitting `OwnerFunctionCalled(22)` and performing a privileged state modification (`reduceStakeTo(staker[i], 0)`), the function signature on line 228 only enforces the `whenPaused` modifier and lacks an explicit owner authorization modifier (`onlyOwner`).

When the rollup contract is paused (for example, during an emergency response or maintenance window prior to unpausing), any unprivileged account that can reach `RollupAdminLogic.forceRefundStaker` can pass an array of inactive validator addresses. For each address, `reduceStakeTo(staker[i], 0)` sets `_stakerMap[staker[i]].amountStaked` to `0` and moves the entire staked balance into `_withdrawableFunds[withdrawalAddress]`, while leaving `_stakerMap[staker[i]].isStaked` set to `true`. Furthermore, callers can spoof the `OwnerFunctionCalled(22)` event by repeatedly calling `forceRefundStaker` on a staker whose stake is already `0`.

### Assumptions and preconditions

1. The rollup contract is currently in a paused state (`paused() == true`).
2. The proxy/dispatch configuration allows external calls to reach `RollupAdminLogic.forceRefundStaker` without an upstream caller restriction, or the function is relied upon to enforce its own `onlyOwner` check.
3. At least one validator is currently staked (`isStaked(staker) == true`) and inactive (`latestStakedAssertion(staker) == latestConfirmed()` or its latest staked assertion already has a child).

### Proof of concept

1. The rollup has active and inactive validators staked with `baseStake`.
2. The rollup admin pauses the rollup (`_paused = true`) to investigate an incident, intending to unpause the rollup once resolved.
3. While the rollup is paused, an unprivileged caller invokes `forceRefundStaker([validator1, validator2, ...])` with all inactive validators.
4. `requireInactiveStaker` succeeds for each inactive validator, and `reduceStakeTo(validator, 0)` sets each validator's `amountStaked` to `0` and transfers their stake accounting to `_withdrawableFunds[withdrawalAddress]`.
5. When the admin later unpauses the rollup to resume normal operation, the affected validators have `amountStaked == 0` while still marked `isStaked == true`, preventing them from creating new assertions until they manually re-deposit stake and disrupting automated validator or staking-pool workflows.

### Impact assessment

**Severity: Medium**

- **Disruption of Validator Operations:** Any inactive validator can have their `amountStaked` forcefully zeroed out during a pause window. Once the rollup is unpaused, these validators cannot post assertions until they detect the state change and replenish their stake.
- **Accounting / Integration Friction:** Validators remain marked as `isStaked = true` with `amountStaked = 0`. External integrations (such as `IAssertionStakingPool`, which explicitly notes that `returnOldDeposit` reverts with a `0` balance after `forceRefundStaker`) can be forced into an unexpected state without admin action.
- **Event Spoofing:** Because `reduceStakeTo(staker, 0)` succeeds even when `amountStaked` is already `0`, an unprivileged caller can repeatedly invoke `forceRefundStaker` while paused to emit arbitrary numbers of `OwnerFunctionCalled(22)`, `UserStakeUpdated`, and `UserWithdrawableFundsUpdated` events.

### Remediation

Add the `onlyOwner` access control modifier to `forceRefundStaker` in `src/rollup/RollupAdminLogic.sol`:

```solidity
    function forceRefundStaker(address[] calldata staker) external override onlyOwner whenPaused {
        require(staker.length > 0, "EMPTY_ARRAY");
        for (uint256 i = 0; i < staker.length; i++) {
            requireInactiveStaker(staker[i]);
            reduceStakeTo(staker[i], 0);
        }
        emit OwnerFunctionCalled(22);
    }
```

## Finding 4: Inconsistent Historical vs. Live Configuration Snapshot in `RollupCore.createNewAssertion` Causes Stake Locking and Delayed Parameter Updates

### Synopsis

In `RollupCore.createNewAssertion`, `newAssertion.configHash` snapshots live security parameters (`wasmModuleRoot`, `baseStake`, `challengeManager`, `confirmPeriodBlocks`) for future child assertions while governing the newly created assertion itself with `prevAssertion`'s historical `configHash`, causing parameter mismatches, stuck excess stake when `baseStake` changes, and broken configuration updates during `forceCreateAssertion`.

### Technical details

In `src/rollup/RollupCore.sol`, `createNewAssertion` validates `assertion.beforeStateData.configData` against `prevAssertion.configHash` (line 371), and then initializes `newAssertion` at lines 488–497 by computing `newAssertion.configHash` from the contract's live state variables:

- `wasmModuleRoot` (line 491)
- `requiredStake: baseStake` (line 492)
- `challengeManager: address(challengeManager)` (line 493)
- `confirmPeriodBlocks` (line 494)
- `nextInboxPosition: uint64(nextInboxPosition)` (line 495)

Because all protocol operations acting on `newAssertion`—including stake validation and loser-stake escrow transfers in `RollupUserLogic.stakeOnNewAssertion` (lines 182, 215) and `fastConfirmNewAssertion` (line 295), one-step challenge resolution, and confirmation delay checks—read the `ConfigData` authenticated against `prevAssertion.configHash` rather than `newAssertion.configHash`, storing live security parameters into `newAssertion.configHash` creates an inconsistency between the historical configuration governing `newAssertion` and the live configuration snapshotted onto `newAssertion`:

1. **Staking and Escrow Mismatch on `baseStake` Updates / Overflow Assertions:**
   - When `baseStake` is **decreased** (e.g., from $S_1$ to $S_2 < S_1$) after `prevAssertion` is created, a validator creating `newAssertion` must still have at least `prevAssertion`'s historical `requiredStake` ($S_1$) staked (line 182). However, `createNewAssertion` records the lower live `baseStake` ($S_2$) into `newAssertion.configHash` (line 492). Because the validator is now staked on an unconfirmed assertion, they cannot call `reduceDeposit` to withdraw the excess $S_1 - S_2$. When `newAssertion` is extended and challenged by a rival child assertion, the rival validator only needs $S_2$ stake and only $S_2$ is transferred to `loserStakeEscrow` (line 215). If the first validator loses the dispute, their entire $S_1$ stake is slashed while only $S_2$ was escrowed, permanently trapping $S_1 - S_2$ tokens in the contract.
   - When `baseStake` is **increased** (e.g., from $S_1$ to $S_2 > S_1$) after `prevAssertion` is created (including when `newAssertion` is an `overflowAssertion` that has not finished consuming `prevAssertion`'s batch range), a validator with only $S_1$ staked can create `newAssertion` because line 182 checks `prevAssertion`'s historical `requiredStake` ($S_1$), but line 492 snapshots $S_2$ into `newAssertion.configHash`, preventing the validator from extending their own lineage (or completing the remaining overflow batches) without depositing additional stake.
   - Similarly, if two competing children of `prevAssertion` are created before and after a parameter change, the sibling assertions at the same lineage depth snapshot different `configHash` configurations for their respective subtrees.

2. **Ineffective Parameter Updates on `newAssertion` (Including `forceCreateAssertion`):**
   - When an administrator updates `wasmModuleRoot`, `baseStake`, `challengeManager`, or `confirmPeriodBlocks` (either while paused during emergency recovery in `RollupAdminLogic.forceCreateAssertion` at lines 242–254, or when no pending assertions exist after `latestConfirmed`), the next assertion created from `prevAssertionHash` still uses `prevAssertion`'s historical `configHash` rather than the newly updated parameters, because lines 490–496 only apply the live parameters to `newAssertion.configHash` (which governs children of `newAssertion`, not `newAssertion` itself).

### Assumptions and preconditions

1. An assertion `prevAssertion` exists on-chain with a `configHash` snapshotted at the time of its creation.
2. One or more rollup configuration parameters (`baseStake`, `wasmModuleRoot`, `confirmPeriodBlocks`, or `challengeManager`) are updated by the administrator, or an overflow/sibling assertion is created across a configuration update boundary.

### Proof of concept

1. Suppose `baseStake` is initially `100e18`, and `prevAssertion` ($A$) is created and confirmed with `A.configData.requiredStake = 100e18`.
2. The rollup administrator lowers `baseStake` to `20e18`.
3. Validator 1 stakes `100e18` tokens (required because `stakeOnNewAssertion` line 182 checks `amountStaked(msg.sender) >= assertion.beforeStateData.configData.requiredStake`, which is `A`'s `100e18`) and creates assertion $B$ (child of $A$).
4. In `RollupCore.createNewAssertion` (lines 490–496), $B$'s `configHash` is created with the live `requiredStake: baseStake` (`20e18`).
5. While $B$ is still pending, Validator 1 cannot call `reduceDeposit` to withdraw the excess `80e18` tokens because Validator 1 is actively staked on unconfirmed assertion $B$.
6. Validator 1 extends their lineage by creating assertion $C_1$ (first child of $B$).
7. Validator 2 challenges $C_1$ by staking only `20e18` tokens (since `B.configData.requiredStake == 20e18`) and creating rival assertion $C_2$ (second child of $B$). Line 215 transfers `B.configData.requiredStake` (`20e18`) to `loserStakeEscrow`.
8. When $C_1$ loses the challenge and $C_2$ is confirmed, Validator 1's entire `100e18` stake is confiscated, Validator 2 can withdraw their `20e18` stake, `loserStakeEscrow` only received `20e18`, and `80e18` of `stakeToken` remains permanently locked inside the rollup contract.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:**
  - Changing `baseStake` causes excess validator stake to become permanently stuck in the rollup contract on a lost challenge (when `baseStake` decreases) or prevents existing stakers from extending their own assertion branch / completing overflow assertions without topping up stake (when `baseStake` increases).
  - Updating `wasmModuleRoot` or `confirmPeriodBlocks` (including during `forceCreateAssertion` recovery) fails to take effect on the newly created child assertion of `prevAssertion`, because `createNewAssertion` only snapshots the updated parameters onto `newAssertion.configHash` for its future descendants while binding `newAssertion` itself to `prevAssertion.configHash`.

### Remediation

Ensure that configuration parameters snapshotted into `newAssertion.configHash` remain consistent with the historical configuration of `prevAssertion` when continuing an unfinished inbox range (`overflowAssertion`) or creating sibling assertions on an existing branch, and bind each assertion's own execution parameters (`wasmModuleRoot`, `requiredStake`, `challengeManager`, `confirmPeriodBlocks`) consistently to the assertion being created:

```solidity
        ConfigData calldata prevConfig = assertion.beforeStateData.configData;
        bool keepPrevConfig = overflowAssertion || prevAssertion.firstChildBlock != 0;
        AssertionNode memory newAssertion = AssertionNodeLib.createAssertion(
            prevAssertion.firstChildBlock == 0, // assumes block 0 is impossible
            RollupLib.configHash({
                wasmModuleRoot: keepPrevConfig ? prevConfig.wasmModuleRoot : wasmModuleRoot,
                requiredStake: keepPrevConfig ? prevConfig.requiredStake : baseStake,
                challengeManager: keepPrevConfig ? prevConfig.challengeManager : address(challengeManager),
                confirmPeriodBlocks: keepPrevConfig ? prevConfig.confirmPeriodBlocks : confirmPeriodBlocks,
                nextInboxPosition: keepPrevConfig ? prevConfig.nextInboxPosition : uint64(nextInboxPosition)
            })
        );
```

## Finding 5: Premature Inactive-Staker Classification in `RollupCore.requireInactiveStaker` Allows Stake Withdrawal on Unconfirmed and Losing Branches

### Synopsis

`RollupCore.requireInactiveStaker` classifies a staker as inactive whenever their `latestStakedAssertion` has a child (`firstChildBlock > 0`), without verifying that the assertion is confirmed. This allows stakers on unconfirmed or losing branches to prematurely withdraw or reduce locked stake via `returnOldDeposit` or `reduceDeposit`.

### Technical details

In `src/rollup/RollupCore.sol`, `requireInactiveStaker` enforces that a validator's stake cannot be withdrawn (`returnOldDeposit`), reduced (`reduceDeposit`), or force-refunded (`forceRefundStaker`) while the validator is actively staked:

```solidity
function requireInactiveStaker(address stakerAddress) internal view {
    require(isStaked(stakerAddress), "NOT_STAKED");
    bytes32 lastestAssertion = latestStakedAssertion(stakerAddress);
    bool isLatestConfirmed = lastestAssertion == latestConfirmed();
    bool haveChild = getAssertionStorage(lastestAssertion).firstChildBlock > 0;
    require(isLatestConfirmed || haveChild, "STAKE_ACTIVE");
}
```

The function assumes that if `getAssertionStorage(lastestAssertion).firstChildBlock > 0` (`haveChild == true`), the staker's stake commitment on `lastestAssertion` (and any unconfirmed ancestors) is completely replaced by the child assertion's staker, making `stakerAddress` safe to withdraw or reduce stake.

However, `firstChildBlock > 0` only records that a child assertion was created under `lastestAssertion`; it does not guarantee that `lastestAssertion` is confirmed (`AssertionStatus.Confirmed`) or that the child assertion locks an equivalent stake covering unconfirmed ancestors or penalized branches. This creates a lifecycle desynchronization between staker balances and assertion states in two ways:

1. **Unconfirmed Ancestor Stake Requirement Bypass:** As noted in `RollupUserLogic.stakeOnNewAssertion` (lines 179–181), a staker's excess stake must remain locked while they are staked on an unconfirmed ancestor that required a higher stake (`requiredStake`), and can only be removed via `reduceDeposit` once the staker is inactive. Because `requireInactiveStaker` treats `haveChild` (`firstChildBlock > 0`) as sufficient for inactivity even while `lastestAssertion` is still `AssertionStatus.Pending`, any child assertion created under a lower `requiredStake` sets `firstChildBlock > 0` on the unconfirmed parent (or allows the staker to advance to the child and have a grandchild created at the lower `requiredStake`). The staker is immediately considered inactive and can call `returnOldDeposit()` or `reduceDeposit()` to withdraw the higher stake while the ancestor assertion is still unconfirmed. If a rival sibling challenges that unconfirmed ancestor, `stakeOnNewAssertion` transfers the ancestor's higher `requiredStake` to `loserStakeEscrow`, leaving the `RollupCore` contract under-collateralized and unable to refund honest stakers.
2. **Losing Branch Excess-Stake Lock Bypass:** As noted in `RollupUserLogic.stakeOnNewAssertion` (line 214), when a second child is created under a parent assertion, only `requiredStake` is transferred to `loserStakeEscrow`, and any excess stake (`amountStaked > requiredStake`) held by the losing staker is intended to remain stuck because the losing assertion can never become `latestConfirmed()`. However, if another staker (with only `requiredStake`) creates a first child under the losing assertion (`isFirstChild == true`, which transfers nothing to `loserStakeEscrow`), `firstChildBlock > 0` is set on the losing assertion. `requireInactiveStaker` then misclassifies the losing staker on the unconfirmed/defeated branch as inactive, allowing them to call `returnOldDeposit()` or `reduceDeposit()` and withdraw their entire `amountStaked`.

### Assumptions and preconditions

- A staker (`staker1`) is staked on an unconfirmed (`Pending`) assertion `A` where either:
  - `staker1` has `amountStaked > requiredStake` on a losing assertion `A`, or
  - `A` was created under a higher `requiredStake` than the `requiredStake` configured for children of `A`.
- A child assertion `B` is created with `A` as `prevAssertion` by a second validator staking only the child's `requiredStake`.

### Proof of concept

1. Suppose parent assertion `P` is `latestConfirmed()` and the configured `requiredStake` is `100e18` tokens.
2. `Validator1` stakes `500e18` tokens (or `100e18` tokens prior to a `requiredStake` reduction to `10e18` for child assertions) on an invalid assertion `A1` (first child of `P`).
3. `Validator2` (honest) stakes `100e18` tokens on the valid sibling assertion `A2` (second child of `P`). Because `A2.isFirstChild` is `false`, `stakeOnNewAssertion` transfers `100e18` tokens (`requiredStake`) to `loserStakeEscrow`.
4. `A2` wins the challenge and is confirmed via `confirmAssertion`, updating `_latestConfirmed` to `A2`. `Validator1`'s assertion `A1` is now on a defeated, unconfirmed branch (`A1.status == AssertionStatus.Pending`), and `Validator1`'s `500e18` stake (`400e18` excess stake) is intended to remain locked.
5. An accomplice `Validator3` stakes `100e18` tokens (or `10e18` tokens if `requiredStake` was reduced) on a new assertion `B1` with `prevAssertion = A1`.
   - In `createNewAssertion`, `A1.childCreated()` sets `A1.firstChildBlock = uint64(block.number)`.
   - Because `B1.isFirstChild` is `true`, no tokens are transferred to `loserStakeEscrow`.
6. `Validator1` calls `returnOldDeposit()`:
   - `requireInactiveStaker(Validator1)` checks `getAssertionStorage(A1).firstChildBlock > 0`, which is now `true` (`haveChild == true`).
   - `requireInactiveStaker` succeeds even though `A1` is unconfirmed (`Pending`) and on a losing branch.
   - `withdrawStaker(Validator1)` credits all `500e18` tokens to `Validator1`'s `withdrawalAddress` and removes `Validator1` from `_stakerMap`.
7. In the case where `A1` had a higher `requiredStake` (`100e18`) than `B1` (`10e18`) and `Validator1` withdrew `100e18` before `A2` was created, when `Validator2` creates `A2` (which sweeps `100e18` to `loserStakeEscrow`) and confirms `A2`, the Rollup contract only holds `10e18` from `Validator3` instead of `100e18`, leaving `Validator2` unable to withdraw their `100e18` stake.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Stakers on unconfirmed or defeated branches can bypass stake lockup invariants by having a child assertion created on their unconfirmed assertion. This enables recovery of locked excess stake on losing branches and, if `requiredStake` is reduced across unconfirmed assertions, allows withdrawing higher ancestor stakes before confirmation, resulting in Rollup contract insolvency when `loserStakeEscrow` sweeps the ancestor's `requiredStake`.

### Remediation

Ensure that a staker is only considered inactive when their `latestStakedAssertion` is confirmed (`AssertionStatus.Confirmed`), or track and enforce that `lastestAssertion` (and any unconfirmed ancestors on its branch) is confirmed before permitting stake withdrawal or reduction in `requireInactiveStaker`:

```solidity
    function requireInactiveStaker(address stakerAddress) internal view {
        require(isStaked(stakerAddress), "NOT_STAKED");
        bytes32 lastestAssertion = latestStakedAssertion(stakerAddress);
        AssertionNode storage assertion = getAssertionStorage(lastestAssertion);
        bool isConfirmed = assertion.status == AssertionStatus.Confirmed;
        bool haveChild = assertion.firstChildBlock > 0 && isConfirmed;
        require( isConfirmed || haveChild, "STAKE_ACTIVE");
    }
```

## Finding 6: Insufficient Stake Enforcement Across `baseStake` Decreases in `RollupUserLogic.stakeOnNewAssertion` Allows Unlocking Higher-Staked Ancestors and Draining Rollup Stake Collateral

### Synopsis

In `RollupUserLogic.stakeOnNewAssertion`, a child assertion only requires `assertion.beforeStateData.configData.requiredStake` from its parent, while unlocking the parent staker's entire deposit once `firstChildBlock > 0`. When `baseStake` decreases, a small child stake unlocks a larger unconfirmed parent stake, causing collateral shortfalls and enabling stake drainage.

### Technical details

In `RollupCore` and `RollupUserLogic`, only one stake is held per consistent assertion lineage. Whenever an assertion `P` is created in `RollupCore.createNewAssertion` (lines 488–497), its `configHash` records the current `baseStake` as `configData.requiredStake`, which dictates the stake required to create any direct child of `P`.

In `RollupUserLogic.stakeOnNewAssertion`:
1. Line 182 checks that the caller's stake covers the parent assertion's `requiredStake`:
   ```solidity
   require(amountStaked(msg.sender) >= assertion.beforeStateData.configData.requiredStake, "INSUFFICIENT_STAKE");
   ```
2. Lines 184–198 verify that `msg.sender` is either extending their own `lastAssertion` (`lastAssertion == prevAssertion`) or that their `lastAssertion` already has a child (`getAssertionStorage(lastAssertion).firstChildBlock > 0`).
3. Inside `createNewAssertion`, `prevAssertion.childCreated()` sets `prevAssertion.firstChildBlock = uint64(block.number)`.
4. If the new assertion is not the first child of `prevAssertion`, line 215 immediately transfers `assertion.beforeStateData.configData.requiredStake` (i.e., `prevAssertion`'s `requiredStake`) to `loserStakeEscrow`.

The developer comment at lines 179–181 notes:
```solidity
// the staker may have more than enough stake, and the entire stake will be locked
// we cannot do a refund here because the staker may be staker on an unconfirmed ancestor that requires more stake
// excess stake can be removed by calling reduceDeposit when the staker is inactive
```

However, this invariant is only preserved when the **same** staker extends their own assertion. A staker becomes inactive (and able to call `returnOldDeposit()`, `reduceDeposit()`, or move their stake to another branch via `stakeOnNewAssertion` at lines 194–198) as soon as `getAssertionStorage(lastAssertion).firstChildBlock > 0` (`RollupCore.requireInactiveStaker`, lines 565–574).

When `baseStake` is lowered from $S_{\text{high}}$ to $S_{\text{low}}$ ($S_{\text{low}} < S_{\text{high}}$):
- Suppose parent assertion `P` has `P.configData.requiredStake = S_high`.
- Assertion `X` (a child of `P`) requires $S_{\text{high}}$ to create, but `X` itself is created with `X.configData.requiredStake = baseStake = S_low`.
- A second validator (or a second account controlled by the creator of `X`) can create child `Y` of `X` by staking only $S_{\text{low}}$.
- Creating `Y` sets `X.firstChildBlock > 0`, which immediately marks the staker on `X` as inactive and frees their $S_{\text{high}}$ stake while `X` is still unconfirmed.

This breaks stake accounting integrity in two ways:
1. **Unconfirmed Ancestor Under-Collateralization**: Even among honest validators, once `V1` (who staked $S_{\text{high}}$ on `X1`) withdraws or reduces their deposit after `V2` stakes $S_{\text{low}}$ on child `Y1`, the unconfirmed assertion `X1` is only backed by $S_{\text{low}}$ in the contract. If `X1` is challenged by a sibling `X2` under `P`, creating `X2` requires $S_{\text{high}}$ and transfers $S_{\text{high}}$ to `loserStakeEscrow`, leaving only $S_{\text{low}}$ in the Rollup contract to pay a winner owed $S_{\text{high}}$.
2. **Collateral Reuse / Drainage Across Competing Commitments**: When an attacker `A1` creates a competing second child `X2` under `P` with stake $S_{\text{high}}$, $S_{\text{high}}$ tokens are sent to `loserStakeEscrow`, while `_stakerMap[A1].amountStaked` remains $S_{\text{high}}$. Using a second account `A2`, the attacker stakes just $S_{\text{low}}$ to create `Y2` (the first child of `X2`, which transfers nothing to `loserStakeEscrow`). This sets `X2.firstChildBlock > 0`, unlocking `A1`'s recorded $S_{\text{high}}$ stake so `A1` can either withdraw $S_{\text{high}}$ from the Rollup contract (stealing honest validators' stake) or reuse `A1`'s recorded $S_{\text{high}}$ stake on another competing child `X3` under `P` to repeatedly sweep $S_{\text{high}}$ from the Rollup contract to `loserStakeEscrow` at a cost of only $S_{\text{low}}$.

### Assumptions and preconditions

1. An assertion `P` exists with `configData.requiredStake = S_high`, and `baseStake` is subsequently reduced by the rollup admin to `S_low < S_high` (either before `P` gets children or before a competing child of `P` is created).
2. At least two validator accounts (or any two accounts when `validatorWhitelistDisabled == true`) participate in creating assertions.

### Proof of concept

1. Assume `latestConfirmed` is assertion `P`, which was created when `baseStake = 100e18` (`P.configData.requiredStake = 100e18`).
2. Honest validator `V1` calls `newStakeOnNewAssertion` with `100e18` tokens to create `X1` (the first child of `P`). The Rollup contract holds `100e18` tokens.
3. The rollup admin lowers `baseStake` from `100e18` to `10e18`.
4. Attacker account `A1` calls `newStakeOnNewAssertion` with `100e18` tokens to create `X2` (the second child of `P`, competing with `X1`):
   - Line 182 passes since `amountStaked(A1) == 100e18 >= P.configData.requiredStake (100e18)`.
   - `createNewAssertion` records `X2.configData.requiredStake = baseStake = 10e18`.
   - Because `X2.isFirstChild == false`, line 215 transfers `100e18` tokens to `loserStakeEscrow`.
   - `receiveTokens(100e18)` pulls `100e18` tokens from `A1`. The Rollup contract balance is `100e18 + 100e18 - 100e18 = 100e18` tokens, and `amountStaked(A1)` remains `100e18` staked on `X2`.
5. Attacker account `A2` calls `newStakeOnNewAssertion` with `10e18` tokens to create an overflow assertion `Y2` (the first child of `X2`, bypassing `minimumAssertionPeriod` since `overflowAssertion == true`):
   - Line 182 passes since `amountStaked(A2) == 10e18 >= X2.configData.requiredStake (10e18)`.
   - `createNewAssertion` sets `X2.firstChildBlock = uint64(block.number)`.
   - Because `Y2.isFirstChild == true`, no tokens are sent to `loserStakeEscrow`.
   - The Rollup contract balance is now `110e18` tokens.
6. Because `X2.firstChildBlock > 0`, `A1` is now considered inactive on `X2`:
   - **Path A (Direct Theft / Insolvency)**: `A1` calls `returnOldDeposit()` and `withdrawStakerFunds()`, withdrawing `100e18` tokens from the Rollup contract. The attacker spent `110e18` (`100e18` to `loserStakeEscrow` + `10e18` from `A2`) and recovered `100e18`, net costing only `10e18` while leaving the Rollup contract with only `10e18` tokens. When `X1` is confirmed, honest validator `V1` cannot withdraw their `100e18` stake (`90e18` shortfall).
   - **Path B (Repeated Collateral Reuse)**: Instead of withdrawing, `A1` calls `stakeOnNewAssertion` with `prevAssertion = P` to create a third child `X3` under `P`. Lines 182 and 196 pass because `amountStaked(A1) == 100e18` and `getAssertionStorage(X2).firstChildBlock > 0`. Line 215 transfers the remaining `100e18` tokens in the Rollup contract to `loserStakeEscrow` without `A1` transferring any new tokens.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: The Rollup contract becomes under-collateralized / insolvent with respect to active validator stakes, causing honest validators' `withdrawStakerFunds()` calls to revert due to insufficient token balance, or allowing malicious validators to recover stake that was already confiscated to `loserStakeEscrow` at the expense of honest validators' principal.
- **Business Implications**: Honest validators can permanently lose up to `S_high - S_low` of their staked capital (or have their entire `S_high` stake locked in an insolvent contract), and attackers can launch low-cost delay/challenge attacks at a cost of `S_low` instead of `S_high`.

### Remediation

Ensure that a child assertion on an unconfirmed lineage cannot be created with a `requiredStake` lower than that of its unconfirmed ancestors (for example, by propagating `max(baseStake, assertion.beforeStateData.configData.requiredStake)` when computing the new assertion's `configHash` in `createNewAssertion`, or requiring `amountStaked(msg.sender)` to cover the maximum required stake along the unconfirmed lineage). Alternatively, enforce in `stakeOnNewAssertion` that `amountStaked(msg.sender)` is checked against both `assertion.beforeStateData.configData.requiredStake` and `baseStake`, and prevent `requiredStake` on unconfirmed children from dropping below their parent's `requiredStake`:

```solidity
        uint256 requiredStake = assertion.beforeStateData.configData.requiredStake;
        if (baseStake > requiredStake) {
            requiredStake = baseStake;
        }
        require(amountStaked(msg.sender) >= requiredStake, "INSUFFICIENT_STAKE");

        bytes32 prevAssertion = RollupLib.assertionHash(
            assertion.beforeStateData.prevPrevAssertionHash,
            assertion.beforeState,
            assertion.beforeStateData.sequencerBatchAcc
        );
        getAssertionStorage(prevAssertion).requireExists();

        // Staker can create new assertion only if
        // a) its last staked assertion is the prev; or
        // b) its last staked assertion have a child
        bytes32 lastAssertion = latestStakedAssertion(msg.sender);
```

And in `RollupCore.createNewAssertion`, set the child's `requiredStake` to `baseStake > assertion.beforeStateData.configData.requiredStake || prevAssertionHash == _latestConfirmed ? baseStake : assertion.beforeStateData.configData.requiredStake` so an unconfirmed assertion's required stake never decreases below its unconfirmed parent's required stake.

## Finding 7: Lack of `requiredStake > 0` check or staker-uniqueness per sibling assertion in `RollupUserLogic.stakeOnNewAssertion` allows a single staker to create multiple sibling assertions and drain `RollupCore` via `loserStakeEscrow`

### Synopsis

In `RollupUserLogic.stakeOnNewAssertion`, a staker whose `lastAssertion` already has a child (`firstChildBlock > 0`) can stake on a new sibling assertion of `lastAssertion` without providing additional stake, causing `RollupUserLogic` to transfer `requiredStake` to `loserStakeEscrow` out of other stakers' deposits.

### Technical details

When a validator creates a new assertion via `RollupUserLogic.stakeOnNewAssertion`, lines 182–198 check whether the caller has sufficient stake and is allowed to move their stake to the new assertion:

```solidity
require(amountStaked(msg.sender) >= assertion.beforeStateData.configData.requiredStake, "INSUFFICIENT_STAKE");

bytes32 prevAssertion = RollupLib.assertionHash(
    assertion.beforeStateData.prevPrevAssertionHash,
    assertion.beforeState,
    assertion.beforeStateData.sequencerBatchAcc
);
getAssertionStorage(prevAssertion).requireExists();

// Staker can create new assertion only if
// a) its last staked assertion is the prev; or
// b) its last staked assertion have a child
bytes32 lastAssertion = latestStakedAssertion(msg.sender);
require(
    lastAssertion == prevAssertion || getAssertionStorage(lastAssertion).firstChildBlock > 0,
    "STAKED_ON_ANOTHER_BRANCH"
);
```

And at the end of `stakeOnNewAssertion` (lines 210–216):

```solidity
if (!getAssertionStorage(newAssertionHash).isFirstChild) {
    IERC20(stakeToken).safeTransfer(loserStakeEscrow, assertion.beforeStateData.configData.requiredStake);
}
```

The staking model in `RollupCore` assumes that each diverging branch (each sibling assertion sharing the same `prevAssertion`) locks a distinct staker's stake, so that when a non-first child is created (`!isFirstChild`), one `requiredStake` can be immediately sent to `loserStakeEscrow` while keeping enough funds in the contract to refund the eventual winning staker.

However, notice how `lastAssertion` transitions when a staker creates multiple assertions:
1. Suppose assertion `P` is the latest confirmed assertion (or any existing assertion), and Validator A creates the first child `C1` with parent `P`. Now `P.firstChildBlock > 0` and `latestStakedAssertion(A) == C1`.
2. Wait—if `latestStakedAssertion(A) == P` (for example, Validator A just staked on `P` via `createNewStake`, which sets `latestStakedAssertion = _latestConfirmed = P`, or Validator A previously asserted `P`), and another validator (or `fastConfirmNewAssertion` / `forceCreateAssertion` / Validator B) creates `C1` on top of `P`, then `P.firstChildBlock > 0`.
3. Even simpler: Validator A is staked on `P` (`lastAssertion == P`). Validator A calls `stakeOnNewAssertion` with `prevAssertion == P` to create `C1` (`isFirstChild == true`). Now `latestStakedAssertion(A) = C1`.
4. Next, someone creates a child of `C1` (call it `GC1`), OR Validator B creates `C1` on `P`. If `C1` has a child `GC1` (`C1.firstChildBlock > 0`), Validator A's stake on `C1` is considered inactive (`getAssertionStorage(lastAssertion).firstChildBlock > 0` is `true`!).
5. Because `getAssertionStorage(lastAssertion).firstChildBlock > 0` is satisfied for ANY `prevAssertion`, Validator A can now call `stakeOnNewAssertion` with `prevAssertion == P` (or ANY ancestor/unrelated assertion that already has a first child) to create a second child `C2` of `P`!
6. When `C2` is created on `P`, `C2.isFirstChild` is `false`, so line 215 transfers `requiredStake` of `stakeToken` from the rollup contract to `loserStakeEscrow`.
7. Worse yet, if `P` is the parent and Validator A creates a child `C2` on `P`, and then `C2` also gets a child (or if Validator A has a second validator address B that builds on `C2`), Validator A and B can alternate:
   - Validator A stakes on `C2` (child of `P`). Since `P` already had `C1`, `requiredStake` is transferred to `loserStakeEscrow`.
   - Validator B stakes on `GC2` (child of `C2`). Since `GC2` is the first child of `C2`, no new deposit is needed by B (B's previous assertion had a child), and now `C2.firstChildBlock > 0`!
   - Because `C2.firstChildBlock > 0`, Validator A's stake is freed again! Validator A can immediately call `stakeOnNewAssertion` to create `C3` (another child of `P`), triggering another `safeTransfer(loserStakeEscrow, requiredStake)`!
   - Even better: since `C2` now has a child `GC2`, creating a second child on `C2` or `P` can be repeated in a loop by just two colluding stakers (or one staker with two accounts, or one staker calling `withdrawStaker` / `returnOldDeposit` and re-staking), draining the entire `stakeToken` balance of the rollup contract into `loserStakeEscrow` while only locking 2 stakes!

Wait: if `C2.firstChildBlock > 0`, Validator A is considered an **inactive staker** (`requireInactiveStaker` checks `getAssertionStorage(lastestAssertion).firstChildBlock > 0`). Thus Validator A can even call `returnOldDeposit()` and `withdrawStakerFunds()` to withdraw their entire stake back, even though their stake was already sent to `loserStakeEscrow` when `C2` was created!

Let's trace that single-staker / two-staker drain via `returnOldDeposit()`:
1. `P` is `latestConfirmed` (and already has a first child `C1` created by an honest validator).
2. Attacker stakes `requiredStake` via `newStakeOnNewAssertion` creating `C2` with parent `P`.
   - In `_newStake`: `latestStakedAssertion(attacker) = P`.
   - In `stakeOnNewAssertion`: `lastAssertion == P == prevAssertion` passes!
   - `C2` is created as the second child of `P` (`isFirstChild == false`).
   - `IERC20(stakeToken).safeTransfer(loserStakeEscrow, requiredStake)` sends `requiredStake` to `loserStakeEscrow`.
   - `latestStakedAssertion(attacker) = C2`.
3. Attacker (using a second validator account, or the same account if allowed, or before withdrawing) creates `GC2` as the first child of `C2` (once `minimumAssertionPeriod` passes, or immediately if `GC2` is an `overflowAssertion` or `ERRORED` assertion? Wait: if `GC2` is an `overflowAssertion`, `if (!overflowAssertion)` at line 204 is skipped! So `GC2` can be created in the **exact same block** with zero `TIME_DELTA` wait!).
4. Once `GC2` is created as the first child of `C2`, `C2.firstChildBlock > 0`!
5. Now `requireInactiveStaker(attacker)` passes for the first attacker account because `getAssertionStorage(C2).firstChildBlock > 0`!
6. The first attacker account calls `returnOldDeposit()` and `withdrawStakerFunds()`, getting their full `requiredStake` back, or simply reuses their stake on `P` to create `C3`, `C4`, etc., in the same transaction, draining all staked tokens held by the `RollupCore` contract!

### Assumptions and preconditions

1. The attacker controls at least two validator addresses (either when `validatorWhitelistDisabled == true` or as whitelisted validators), or another validator builds on the attacker's assertion.
2. At least one message batch exists in the bridge (`bridge.sequencerMessageCount() > 0`) so that assertions (including overflow assertions) can be created.
3. The Rollup contract holds `stakeToken` balances from honest stakers.

### Proof of concept

1. Honest Validator V1 is staked on `C1` (first child of `latestConfirmed` assertion `P`) with `requiredStake`. The Rollup contract holds `requiredStake` tokens from V1.
2. Attacker Account A1 calls `newStakeOnNewAssertion` with `tokenAmount = requiredStake` to create assertion `C2` as a second child of `P` (`prevAssertion = P`), making `C2` an overflow assertion (consuming only 1 message position so another overflow assertion can follow it).
   - Because `C1` already exists on `P`, `C2.isFirstChild` is `false`.
   - Line 215 executes `IERC20(stakeToken).safeTransfer(loserStakeEscrow, requiredStake)`, transferring `requiredStake` out of the contract to `loserStakeEscrow`.
   - Contract balance is now `requiredStake` (from V1 + A1 - 1 sent to escrow).
3. In the same block/transaction, Attacker Account A2 calls `newStakeOnNewAssertion` with `tokenAmount = requiredStake` to create overflow assertion `GC2` with parent `C2` (`prevAssertion = C2`).
   - Because `GC2` is an overflow assertion, line 204 (`if (!overflowAssertion)`) skips the `TIME_DELTA` check.
   - `GC2` is the first child of `C2`, so `C2.firstChildBlock` is set to `block.number` (> 0), and no stake is transferred to `loserStakeEscrow`.
4. Now `getAssertionStorage(C2).firstChildBlock > 0` is true. Attacker Account A1 calls `stakeOnNewAssertion` to create overflow assertion `GGC2` as the first child of `GC2` (`prevAssertion = GC2`).
   - `getAssertionStorage(lastAssertion).firstChildBlock > 0` is true (since `lastAssertion == C2`), so A1 is allowed to move its stake to `GGC2`!
   - `GC2.firstChildBlock` is now set to `block.number` (> 0).
5. Because both `C2.firstChildBlock > 0` and `GC2.firstChildBlock > 0`, A2's stake on `GC2` is now free! A2 calls `stakeOnNewAssertion` to create `C3` as a third child of `P` (`prevAssertion = P`).
   - `C3.isFirstChild` is `false`, so another `requiredStake` is transferred from the Rollup contract to `loserStakeEscrow`!
6. A1 and A2 repeat steps 3–5 to create `C4, C5, ...` on `P`, transferring `requiredStake` to `loserStakeEscrow` on each iteration until all honest stakers' tokens in the Rollup contract are drained, and finally A1 and A2 build one more child on their tip and call `returnOldDeposit()` + `withdrawStakerFunds()` to withdraw their own stakes (if any remain in the contract).

### Impact assessment

**Severity: High / Critical**

An attacker with two validator accounts (or any validator when the whitelist is disabled) can:
1. Repeatedly trigger transfers of `requiredStake` to `loserStakeEscrow` without locking up new capital, draining the `stakeToken` balance of the Rollup contract that backs honest validators' stakes and withdrawable funds.
2. Withdraw their own stake via `returnOldDeposit()` even after creating a conflicting sibling assertion, completely bypassing the economic slashing/locking requirement of the BoLD protocol and enabling zero-cost Sybil/delay spam of sibling assertions.

### Remediation

Prevent a staker from moving their stake to a different branch (`lastAssertion != prevAssertion`), or track staked lineages such that a staker can only advance their stake along the direct descendants of their `lastAssertion` (`prevAssertion == lastAssertion`) unless `lastAssertion` itself has been confirmed (or require `lastAssertion == prevAssertion || lastAssertion == latestConfirmed()`). Specifically, a staker's stake on an unconfirmed assertion `A` is only replaced along `A`'s lineage when a child `B` is staked on top of `A`, and even then, `A`'s staker should not be able to jump to an unrelated branch or re-stake on `A`'s parent while `A`'s branch remains unconfirmed.

## Finding 8: Malicious Validator Can Bypass `minimumAssertionPeriod` in `RollupUserLogic.stakeOnNewAssertion` by Submitting Bogus Overflow Assertions

### Synopsis

In `RollupUserLogic.stakeOnNewAssertion`, the `minimumAssertionPeriod` delay is skipped when `overflowAssertion` is true. Because `createNewAssertion` determines `overflowAssertion` purely from unverified caller-supplied `afterState` inbox positions, a malicious validator can craft fake overflow assertions to bypass `minimumAssertionPeriod`, usurp `isFirstChild` status, and spam chained assertions within a single block.

### Technical details

In `RollupUserLogic.stakeOnNewAssertion`, the minimum delay between a parent assertion and its child assertion (`minimumAssertionPeriod`) is only enforced if the new assertion is not classified as an overflow assertion:

```solidity
(bytes32 newAssertionHash, bool overflowAssertion) =
    createNewAssertion(assertion, prevAssertion, expectedAssertionHash);
_stakerMap[msg.sender].latestStakedAssertion = newAssertionHash;

if (!overflowAssertion) {
    uint256 timeSincePrev = block.number - getAssertionStorage(prevAssertion).createdAtBlock;
    // Verify that assertion meets the minimum Delta time requirement
    require(timeSincePrev >= minimumAssertionPeriod, "TIME_DELTA");
}
```

In `RollupCore.createNewAssertion` (lines 425–434), `overflowAssertion` is set to `true` whenever the caller-supplied `assertion.afterState` has `machineStatus == MachineStatus.FINISHED` and its `globalState` position is strictly greater than `beforeState.globalState` (`afterGS.comparePositions(beforeGS) > 0`) but strictly less than `beforeStateData.configData.nextInboxPosition` (`afterStateCmpMaxInbox < 0`).

Because assertion state transitions are optimistic and not cryptographically or interactively verified at the time `createNewAssertion` executes, any staked validator can arbitrarily choose `afterState.globalState` values (for example, keeping `inboxPosition` equal to `beforeState` and incrementing `positionInMessage` by `1`) so that `0 < afterGS.comparePositions(beforeGS)` and `afterStateCmpMaxInbox < 0`. `createNewAssertion` will classify this fraudulent assertion as `overflowAssertion = true`.

This flawed assumption creates two direct protocol violations:
1. **First-child priority manipulation before `minimumAssertionPeriod`:** When the inbox does not actually overflow, honest validators must wait `minimumAssertionPeriod` blocks after `prevAssertion.createdAtBlock` before calling `stakeOnNewAssertion` (because the honest assertion consumes up to `nextInboxPosition`, making `overflowAssertion == false`). A malicious validator can submit a bogus overflow assertion immediately after `prevAssertion` is created (before `minimumAssertionPeriod` elapses). The attacker's assertion is accepted and recorded with `isFirstChild = true` and sets `prevAssertion.firstChildBlock`. When the honest validator later submits the valid non-overflow assertion after `minimumAssertionPeriod`, it is forced into the `!isFirstChild` path (line 210), triggering the immediate transfer of `requiredStake` to `loserStakeEscrow` and granting first-child status to the attacker's invalid assertion.
2. **Unthrottled assertion chain spamming with a single stake:** Because a staker whose `latestStakedAssertion` equals `prevAssertion` can immediately extend their own assertion (`lastAssertion == prevAssertion` at line 196) without locking additional stake, an attacker can repeatedly call `stakeOnNewAssertion` in the same block—incrementing `positionInMessage` by `1` each time—to create an arbitrarily long branch of assertions while completely bypassing `minimumAssertionPeriod`.

### Assumptions and preconditions

1. `minimumAssertionPeriod` is configured to a non-zero block count.
2. The attacker is an authorized validator (or `validatorWhitelistDisabled` is `true`) and holds at least `requiredStake` staked in the rollup.
3. At least one message batch exists in the sequencer inbox beyond `beforeState.globalState` so that `beforeGS < afterGS < nextInboxPosition` can be satisfied (e.g., via `positionInMessage > 0`).

### Proof of concept

1. Suppose `prevAssertion` is created at block `B` with `beforeState.globalState` at `(inboxPosition = 1, positionInMessage = 0)` and `configData.nextInboxPosition = 2`. Assume `minimumAssertionPeriod = 50`.
2. An honest validator computes the true state transition consuming batch `1` up to `(inboxPosition = 2, positionInMessage = 0)`. Because `afterInboxPosition == nextInboxPosition`, `afterStateCmpMaxInbox == 0` and `overflowAssertion == false`. The honest validator cannot submit this assertion until block `B + 50` due to `require(timeSincePrev >= minimumAssertionPeriod, "TIME_DELTA")`.
3. At block `B + 1`, a malicious validator calls `stakeOnNewAssertion` building on `prevAssertion` with a fabricated `afterState` having `machineStatus = MachineStatus.FINISHED` and `globalState` at `(inboxPosition = 1, positionInMessage = 1)`.
4. Inside `createNewAssertion`:
   - `afterGS.comparePositions(beforeGS)` returns `1` (`> 0`).
   - `afterGS.comparePositionsAgainstStartOfBatch(2)` returns `-1` (`< 0`).
   - `overflowAssertion` is set to `true`.
5. Back in `stakeOnNewAssertion`:
   - Because `overflowAssertion` is `true`, the `if (!overflowAssertion)` block at lines 204–208 is skipped, bypassing `require(timeSincePrev >= minimumAssertionPeriod, "TIME_DELTA")`.
   - The attacker's fraudulent assertion becomes `isFirstChild = true` on `prevAssertion`.
6. In the same block `B + 1`, the attacker can call `stakeOnNewAssertion` again using their newly created assertion as `prevAssertion` and setting `afterState.globalState` to `(inboxPosition = 1, positionInMessage = 2)`, chaining arbitrarily many assertions with the same single stake without waiting `minimumAssertionPeriod`.

### Impact assessment

**Severity: Medium / High**

- **Bypass of protocol rate-limiting (`minimumAssertionPeriod`):** A malicious validator can create assertions at every block (or multiple times within the same block) along a branch using a single stake, defeating the minimum assertion period intended to bound assertion creation frequency and off-chain validator/watchtower load.
- **Usurpation of `isFirstChild` status:** By submitting a bogus overflow assertion during the window `[createdAtBlock, createdAtBlock + minimumAssertionPeriod)` when honest non-overflow assertions are blocked by `TIME_DELTA`, an attacker guarantees their invalid assertion is recorded as the first child (`isFirstChild = true`), while forcing the honest validator's valid assertion to be recorded as the second child.

### Remediation

Enforce `minimumAssertionPeriod` unconditionally for all assertions in `RollupUserLogic.stakeOnNewAssertion` regardless of whether `overflowAssertion` is true, or enforce that an overflow assertion still respects the minimum assertion period.

```solidity
        (bytes32 newAssertionHash, ) =
            createNewAssertion(assertion, prevAssertion, expectedAssertionHash);
        _stakerMap[msg.sender].latestStakedAssertion = newAssertionHash;

        uint256 timeSincePrev = block.number - getAssertionStorage(prevAssertion).createdAtBlock;
        // Verify that assertion meets the minimum Delta time requirement
        require(timeSincePrev >= minimumAssertionPeriod, "TIME_DELTA");
```

## Finding 9: Cross-Function Reentrancy in `RollupUserLogic.stakeOnNewAssertion` via Premature Escrow Transfer Before Token Deposit

### Synopsis

In `RollupUserLogic.stakeOnNewAssertion`, creating a sibling assertion transfers `requiredStake` to `loserStakeEscrow` via an external `safeTransfer` call. When invoked through `newStakeOnNewAssertion`, this external call executes after the caller's stake is credited in storage but before `receiveTokens` collects the deposit, enabling cross-function reentrancy to drain staked tokens.

### Technical details

In `RollupUserLogic.newStakeOnNewAssertion` (lines 331–342), a new validator stake is created and placed on a new assertion in three steps:
1. `_newStake(tokenAmount, withdrawalAddress)` (line 338) calls `createNewStake`, which immediately records `_stakerMap[msg.sender]` with `amountStaked = tokenAmount` and `isStaked = true`.
2. `stakeOnNewAssertion(assertion, expectedAssertionHash)` (line 339) creates the assertion, sets `_stakerMap[msg.sender].latestStakedAssertion = newAssertionHash` (line 202), and—if the assertion is not the first child of its parent (`!getAssertionStorage(newAssertionHash).isFirstChild` at line 210)—executes an external token transfer:
   ```solidity
   IERC20(stakeToken).safeTransfer(loserStakeEscrow, assertion.beforeStateData.configData.requiredStake);
   ```
3. `receiveTokens(tokenAmount)` (line 341) pulls `tokenAmount` from `msg.sender` via `safeTransferFrom`. The developer comment on line 340 notes `/// @dev This is an external call, safe because it's at the end of the function`, assuming no external calls occur during `_newStake` or `stakeOnNewAssertion`.

However, the `safeTransfer` call at line 215 violates the checks-effects-interactions pattern across `newStakeOnNewAssertion`:
- **Premature disbursement before deposit collection:** `safeTransfer` sends `requiredStake` from the rollup contract's existing token balance before `receiveTokens(tokenAmount)` collects the new staker's tokens.
- **Cross-function reentrancy with unbacked stake credit:** If `stakeToken` implements transfer hooks (such as ERC-777 `tokensToSend`/`tokensReceived` callbacks) or `loserStakeEscrow` triggers a callback upon token transfer, an attacker can re-enter `RollupUserLogic` during line 215 while `_stakerMap[msg.sender].amountStaked` is already credited with an arbitrary `tokenAmount` that has not yet been paid.
- **Same-block stake unlocking via overflow assertions:** During the reentrant call, an accomplice staker can call `stakeOnNewAssertion` to create an overflow child assertion on top of `newAssertionHash`. Because overflow assertions bypass the `minimumAssertionPeriod` check at lines 204–208 (`if (!overflowAssertion)`), this child assertion succeeds in the same block (`timeSincePrev == 0`), setting `firstChildBlock > 0` on `newAssertionHash`. This immediately satisfies `requireInactiveStaker` for the original caller, allowing them to call `returnOldDeposit()` (or `reduceDeposit()`) and `withdrawStakerFunds()` to withdraw the unpaid `tokenAmount` from the contract's token balance and use those very tokens to satisfy `receiveTokens(tokenAmount)` when the outer call resumes.

### Assumptions and preconditions

1. The caller has validator permissions (`isValidator[ caller ] == true` or `validatorWhitelistDisabled == true`).
2. `stakeToken` is a token contract that invokes sender or recipient hooks on transfer (e.g., ERC-777 or an ERC-20 with transfer callbacks).
3. At least one assertion already exists on `prevAssertion`, so the newly created assertion on `prevAssertion` has `isFirstChild == false`.
4. The rollup contract holds existing `stakeToken` balances from honest stakers or withdrawable funds.

### Proof of concept

1. Suppose the rollup contract currently holds `100e18` of `stakeToken` from active validators, `requiredStake` is `10e18`, and `prevAssertion` already has a first child assertion (`firstChildBlock > 0`).
2. Attacker (using Validator Account A) calls `newStakeOnNewAssertion(100e18, assertionB2, expectedHashB2, address(attacker))`, where `assertionB2` builds on `prevAssertion` as a second child (`isFirstChild == false`).
3. `_newStake` credits Account A with `amountStaked = 100e18` in `_stakerMap[Account A]`.
4. `stakeOnNewAssertion` creates `assertionB2` (`newAssertionHash`), sets `_stakerMap[Account A].latestStakedAssertion = newAssertionHash`, and calls `IERC20(stakeToken).safeTransfer(loserStakeEscrow, 10e18)` at line 215.
5. During the `safeTransfer` hook (before `receiveTokens(100e18)` at line 341 executes):
   - Validator Account B (already staked with `10e18`) calls `stakeOnNewAssertion(overflowChildAssertion, expectedChildHash)` building on `newAssertionHash`. Because `overflowChildAssertion` has `afterStateCmpMaxInbox < 0` and `machineStatus == MachineStatus.FINISHED`, `overflowAssertion` is `true`, bypassing `require(timeSincePrev >= minimumAssertionPeriod, "TIME_DELTA")` at line 207 and setting `firstChildBlock = uint64(block.number)` on `newAssertionHash`.
   - Account A now satisfies `requireInactiveStaker(Account A)` because `getAssertionStorage(newAssertionHash).firstChildBlock > 0`.
   - Account A calls `returnOldDeposit()`, which credits `_withdrawableFunds[Account A] += 100e18` and deletes Account A's staker entry.
   - Account A calls `withdrawStakerFunds()`, receiving `90e18` (or `100e18` if holding `10e18` initially) directly from the rollup contract's `stakeToken` balance.
6. The `safeTransfer` call finishes and `newStakeOnNewAssertion` calls `receiveTokens(100e18)`, which pulls `100e18` from Account A back into the rollup contract using the funds just withdrawn in step 5.
7. Net result: Account A has drained the rollup contract's preexisting staked funds without locking its own capital.

### Impact assessment

- **Severity:** High / Critical.
- **Technical Impact:** Complete drainage of `stakeToken` balances held by the rollup contract (belonging to honest staked validators and withdrawable balances), as well as corrupted staker state (`_stakerMap` is deleted during `returnOldDeposit` before `newStakeOnNewAssertion` completes).
- **Business Impact:** Honest validators cannot withdraw their stakes, and the economic security of the rollup dispute protocol is compromised.

### Remediation

Instead of performing an external token transfer during `stakeOnNewAssertion`, credit `loserStakeEscrow` using the internal `increaseWithdrawableFunds` accounting mechanism (as documented in `RollupCore.sol`), and ensure token deposits in `newStakeOnNewAssertion` are collected before state changes or external calls.

Replace line 215 in `src/rollup/RollupUserLogic.sol` with `increaseWithdrawableFunds`:

```solidity
        if (!getAssertionStorage(newAssertionHash).isFirstChild) {
            // We assume assertion.beforeStateData is valid here as it will be validated in createNewAssertion
            // only 1 of the children can be confirmed and get their stake refunded
            // so we send the other children's stake to the loserStakeEscrow
            // NOTE: if the losing staker have staked more than requiredStake, the excess stake will be stuck
            increaseWithdrawableFunds(loserStakeEscrow, assertion.beforeStateData.configData.requiredStake);
        }
```

## Finding 10: Unaccounted Stake Transfer to `loserStakeEscrow` in `RollupUserLogic.stakeOnNewAssertion` Causes Stake Ledger Divergence and Fund Insolvency

### Synopsis

In `RollupUserLogic.stakeOnNewAssertion`, when a non-first child assertion is created, `requiredStake` is transferred to `loserStakeEscrow` without reducing `_stakerMap[msg.sender].amountStaked`. This divergence between ERC20 collateral and internal stake accounting allows forfeited stake to be reused across assertion branches or withdrawn, draining other stakers' funds.

### Technical details

In `RollupUserLogic.stakeOnNewAssertion` (`src/rollup/RollupUserLogic.sol`), a validator stakes on a newly created assertion `newAssertionHash` descending from `prevAssertion`. The function checks that the caller has sufficient recorded stake in `_stakerMap`:

```solidity
require(amountStaked(msg.sender) >= assertion.beforeStateData.configData.requiredStake, "INSUFFICIENT_STAKE");
```

After creating the assertion via `createNewAssertion` and updating `_stakerMap[msg.sender].latestStakedAssertion = newAssertionHash`, lines 210–215 handle the case where `newAssertionHash` is not the first child of `prevAssertion`:

```solidity
if (!getAssertionStorage(newAssertionHash).isFirstChild) {
    IERC20(stakeToken).safeTransfer(loserStakeEscrow, assertion.beforeStateData.configData.requiredStake);
}
```

This logic contains a critical stake collateral accounting flaw:

1. **Escrow Disbursement and Stake-Ledger Divergence**: When `!getAssertionStorage(newAssertionHash).isFirstChild` holds, `assertion.beforeStateData.configData.requiredStake` tokens are immediately transferred out of the rollup contract to `loserStakeEscrow`. However, `_stakerMap[msg.sender].amountStaked` is not decremented by `requiredStake`. As a result, the sum of all recorded stakes in `_stakerMap` plus `totalWithdrawableFunds` exceeds the actual `IERC20(stakeToken).balanceOf(address(this))` by `requiredStake` for every non-first child assertion created.
2. **Collateral Reuse Across Assertion Branches**: Because `_stakerMap[msg.sender].amountStaked` remains intact after the tokens are disbursed to `loserStakeEscrow`, the staker retains full credit in the stake ledger. The staker can immediately reuse this phantom collateral to extend the branch (`lastAssertion == prevAssertion`), and once `newAssertionHash` has a child (`getAssertionStorage(lastAssertion).firstChildBlock > 0`), the staker can move this unbacked stake to other assertion branches in `stakeOnNewAssertion`—even creating additional non-first children that trigger further `safeTransfer(loserStakeEscrow, requiredStake)` payouts funded by other stakers' deposits.
3. **Withdrawal Eligibility After Stake Forfeiture**: Under `requireInactiveStaker`, a staker becomes inactive as soon as their `latestStakedAssertion` either equals `latestConfirmed()` or has a child (`firstChildBlock > 0`). Once a child assertion is built on `newAssertionHash` (or `newAssertionHash` is confirmed), the staker whose `requiredStake` was already transferred to `loserStakeEscrow` becomes eligible to call `returnOldDeposit()` or `reduceDeposit()`, crediting `_withdrawableFunds` with the unreduced `amountStaked` and enabling `withdrawStakerFunds()` to drain tokens deposited by other honest stakers.

### Assumptions and preconditions

1. At least two validators (or two sybil validator accounts when the whitelist is disabled or both are whitelisted) can call `stakeOnNewAssertion` / `newStakeOnNewAssertion`.
2. `assertion.beforeStateData.configData.requiredStake > 0`.

### Proof of concept

1. Assume `requiredStake = 10e18` and `G` is the genesis assertion (`latestConfirmed()`). Honest Validator `V1` calls `newStakeOnNewAssertion` with `10e18` `stakeToken` to create `A1` (the first child of `G`).
   - Contract `stakeToken` balance: `10e18`.
   - `_stakerMap[V1].amountStaked = 10e18`, `latestStakedAssertion(V1) = A1`.
2. Validator `V2` calls `newStakeOnNewAssertion` with `10e18` `stakeToken` to create `A2` (the second child of `G`, so `getAssertionStorage(A2).isFirstChild == false`).
   - At line 215, `10e18` `stakeToken` is transferred to `loserStakeEscrow`.
   - `V2` transfers `10e18` `stakeToken` to the rollup contract in `receiveTokens`.
   - Contract `stakeToken` balance: `10e18 + 10e18 - 10e18 = 10e18`.
   - However, `_stakerMap[V2].amountStaked` remains `10e18` alongside `_stakerMap[V1].amountStaked = 10e18` (`20e18` total recorded stake backed by only `10e18` tokens).
3. Validator `V3` calls `newStakeOnNewAssertion` with `10e18` `stakeToken` to create `B1` (the first child of `A1`).
   - Contract `stakeToken` balance: `20e18`.
   - `A1.firstChildBlock` becomes non-zero, making `V1`'s stake on `A1` inactive.
4. `V1` (or `V3`) uses its freed stake to call `stakeOnNewAssertion` with `prevAssertion = A2`, creating `B2` (the first child of `A2`).
   - Because `B2.isFirstChild == true`, no tokens are transferred to `loserStakeEscrow`.
   - `A2.firstChildBlock` becomes non-zero, unlocking `V2`'s stake on `A2` despite `V2`'s `10e18` stake having already been disbursed to `loserStakeEscrow`.
5. `V2` now has `getAssertionStorage(A2).firstChildBlock > 0` and `_stakerMap[V2].amountStaked == 10e18`:
   - **Branch reuse**: `V2` can call `stakeOnNewAssertion` with `prevAssertion = G` (or `A1`) to create another sibling assertion `A3` (`isFirstChild == false`), causing line 215 to transfer another `10e18` `stakeToken` from the rollup contract's pool to `loserStakeEscrow` without `V2` depositing any additional tokens.
   - **Withdrawal after forfeiture**: Alternatively, `V2` can call `returnOldDeposit()` and `withdrawStakerFunds()` to withdraw `10e18` `stakeToken` from the rollup contract, leaving the rollup insolvent for remaining stakers.

### Impact assessment

- **Severity**: High / Critical.
- **Technical Effect**: The rollup contract's `stakeToken` balance becomes undercollateralized relative to `_stakerMap` and `totalWithdrawableFunds`. Stakers whose stake was already sent to `loserStakeEscrow` retain their full `amountStaked` balance in `_stakerMap`, allowing them to either withdraw funds belonging to other stakers or repeatedly trigger `loserStakeEscrow` transfers on new branches to drain the rollup's `stakeToken` reserves.
- **Business Impact**: Honest validators can lose their staked collateral or face permanent denial of service when attempting to withdraw via `withdrawStakerFunds()`.

### Remediation

Deduct `requiredStake` from the staker's recorded stake when disbursing funds to `loserStakeEscrow`, or defer confiscating losing stakes until the dispute/assertion resolution determines which branch lost so that `_stakerMap` and the contract's token balance remain synchronized.

If deducting from the staker who creates the conflicting assertion at creation time:

```solidity
        if (!getAssertionStorage(newAssertionHash).isFirstChild) {
            uint256 requiredStake = assertion.beforeStateData.configData.requiredStake;
            _stakerMap[msg.sender].amountStaked -= requiredStake;
            IERC20(stakeToken).safeTransfer(loserStakeEscrow, requiredStake);
        }
```
