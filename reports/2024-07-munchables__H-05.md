# Audit report — 2024-07-munchables/H-05

## Finding 1: Missing `plotMetadata[landlord]` Initialization Check in `LandManager.stakeMunchable` Causes Zero Tax Rate and Potential Permanent Lock of Staked NFTs

### Synopsis

`LandManager.stakeMunchable` fails to verify that `plotMetadata[landlord].lastUpdated != 0` before staking a Munchable NFT and recording `plotMetadata[landlord].currentTaxRate`. Staking onto a landlord with uninitialized plot metadata assigns a `0` tax rate instead of `DEFAULT_TAX_RATE` and can cause `_farmPlots` and `unstakeMunchable` to revert, locking the renter's staked NFTs.

### Technical details

In `LandManager`, a landlord's `plotMetadata[landlord]` struct (`lastUpdated` and `currentTaxRate`) is only initialized in two places:
1. When `AccountManager.forceHarvest` calls `landManager.updatePlotMetadata(_player)` upon locking tokens in `LockManager` — which only executes if the player is already registered (`if (player.registrationDate != 0)` at `AccountManager.sol:149`).
2. When a registered landlord manually calls `LandManager.triggerPlotMetadata()` (`LandManager.sol:104`).

Consequently, a landlord can have available plots (`_getNumPlots(landlord) > 0`) while `plotMetadata[landlord].lastUpdated == 0` and `plotMetadata[landlord].currentTaxRate == 0` in the following scenarios:
- The landlord locked tokens in `LockManager` before `LandManager` was deployed and has not yet called `triggerPlotMetadata()`.
- The landlord locked tokens in `LockManager` before registering in `AccountManager` (since `AccountManager.forceHarvest` skips `updatePlotMetadata` when `registrationDate == 0`, and `AccountManager.register` does not initialize `plotMetadata`).

While `LandManager.updateTaxRate` explicitly checks `if (plotMetadata[landlord].lastUpdated == 0) revert PlotMetadataNotUpdatedError();` (`LandManager.sol:96-97`), `LandManager.stakeMunchable` performs no such validation. When a renter stakes a `tokenId` onto such a `landlord`:

1. At `LandManager.sol:162-168`, `toilerState[tokenId]` is recorded with `latestTaxRate: plotMetadata[landlord].currentTaxRate`, which evaluates to `0` instead of `DEFAULT_TAX_RATE`. When `_farmPlots` is later executed, `schnibblesLandlord` is calculated as `0` (`LandManager.sol:287-289`), depriving the landlord of their tax share.
2. If the landlord has locked tokens in `LockManager` but has not registered in `AccountManager`, `stakeMunchable` succeeds because it never validates that `landlord` is registered or has initialized `plotMetadata`. However, every subsequent call by the renter to `unstakeMunchable`, `stakeMunchable`, `transferToUnoccupiedPlot`, or `farmPlots` invokes `_farmPlots(msg.sender)`, which calls `_getMainAccountRequireRegistered(landlord)` (`LandManager.sol:265`) and reverts with `PlayerNotRegisteredError()`.
3. Similarly, if `plotMetadata[landlord].lastUpdated` remains `0` and the landlord's available plot count drops below `_toiler.plotId` (for instance, if `PRICE_PER_PLOT` is increased via configuration update), `_farmPlots` sets `timestamp = plotMetadata[landlord].lastUpdated = 0` (`LandManager.sol:259`), causing the subtraction `(timestamp - _toiler.lastToilDate)` at `LandManager.sol:281` to underflow and revert.

Because `unstakeMunchable` is gated by the `forceFarmPlots(msg.sender)` modifier (`LandManager.sol:175`), any revert in `_farmPlots` prevents the renter from unstaking any of their staked Munchable NFTs.

### Assumptions and preconditions

1. A landlord account has locked tokens in `LockManager` sufficient to provide at least one plot (`_getNumPlots(landlord) > 0`), either before `LandManager` was deployed or prior to registering in `AccountManager`.
2. `plotMetadata[landlord].lastUpdated` is `0` because neither `updatePlotMetadata(landlord)` nor `triggerPlotMetadata()` has been executed for `landlord`.
3. A registered renter owns a Munchable NFT, approves `LandManager`, and calls `stakeMunchable(landlord, tokenId, plotId)`.

### Proof of concept

1. Alice locks tokens in `LockManager` before registering in `AccountManager` (or before `LandManager` deployment without calling `triggerPlotMetadata()`). Because Alice is not registered at the time of `lock()`, `AccountManager.forceHarvest(alice)` skips `landManager.updatePlotMetadata(alice)`.
2. Alice now has `_getNumPlots(alice) > 0`, while `plotMetadata[alice].lastUpdated == 0` and `plotMetadata[alice].currentTaxRate == 0`.
3. Bob (a registered player) approves `LandManager` for `tokenId = 1` and calls `LandManager.stakeMunchable(alice, 1, 0)`.
4. `stakeMunchable` transfers `tokenId = 1` to `LandManager` (`line 152`), marks plot `0` as occupied (`lines 154-157`), adds `1` to `munchablesStaked[bob]` (`line 159`), and sets `toilerState[1].latestTaxRate = 0` (`line 166`).
5. **Case A (Alice has not registered):** When Bob attempts to call `unstakeMunchable(1)` or `farmPlots()`, the `forceFarmPlots(msg.sender)` modifier runs `_farmPlots(bob)`, which calls `_getMainAccountRequireRegistered(alice)` at `line 265` and reverts with `PlayerNotRegisteredError()`. Bob's NFT (along with any other NFTs Bob staked) is frozen in `LandManager`.
6. **Case B (Alice registered after locking without calling `triggerPlotMetadata()`):** `plotMetadata[alice].lastUpdated` and `plotMetadata[alice].currentTaxRate` remain `0`. When Bob farms plots, Alice receives `0` tax instead of `DEFAULT_TAX_RATE`. Furthermore, if `_getNumPlots(alice)` drops below `plotId`, `_farmPlots` sets `timestamp = plotMetadata[alice].lastUpdated = 0` (`line 259`) and reverts due to arithmetic underflow at `timestamp - _toiler.lastToilDate` (`line 281`), permanently locking Bob's staked NFTs in `LandManager`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  1. **Permanent/Indefinite Freezing of Staked NFTs:** A renter who stakes onto a landlord with uninitialized `plotMetadata` (either an unregistered token locker or a landlord whose `plotMetadata[landlord].lastUpdated == 0` when plot capacity decreases) cannot execute `_farmPlots` or `unstakeMunchable`, trapping all of the renter's staked Munchable NFTs inside `LandManager`.
  2. **Loss of Yield / Tax Revenue:** Landlords whose `plotMetadata` was not initialized prior to a renter staking have `toilerState[tokenId].latestTaxRate` initialized to `0` instead of `DEFAULT_TAX_RATE`, losing all tax schnibbles until the plot is farmed or updated.

### Remediation

Require `plotMetadata[landlord].lastUpdated != 0` in `stakeMunchable` (reverting with `PlotMetadataNotUpdatedError()` if uninitialized) before transferring the NFT and recording `toilerState[tokenId]`.

```solidity
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();
        if (
            !munchNFT.isApprovedForAll(mainAccount, address(this)) &&
            munchNFT.getApproved(tokenId) != address(this)
        ) revert NotApprovedError();
        munchNFT.transferFrom(mainAccount, address(this), tokenId);

        plotOccupied[landlord][plotId] = Plot({
            occupied: true,
            tokenId: tokenId
        });

        munchablesStaked[mainAccount].push(tokenId);
        munchableOwner[tokenId] = mainAccount;

        toilerState[tokenId] = ToilerState({
            lastToilDate: block.timestamp,
            plotId: plotId,
            landlord: landlord,
            latestTaxRate: plotMetadata[landlord].currentTaxRate,
            dirty: false
        });

        emit FarmPlotTaken(toilerState[tokenId], tokenId);
```

## Finding 2: Reentrancy and CEI Violation in `LandManager.stakeMunchable` Allows Bypassing Plot Occupancy and Staking Limits

### Synopsis

`LandManager.stakeMunchable` performs an external `munchNFT.transferFrom` call before updating `plotOccupied`, `munchablesStaked`, `munchableOwner`, and `toilerState`. This Checks-Effects-Interactions (CEI) violation exposes inconsistent intermediate state during the asset transfer, enabling transfer-hook reentrancy to bypass the per-account staking limit and single-plot occupancy checks.

### Technical details

In `src/managers/LandManager.sol`, `stakeMunchable` validates plot availability (`plotOccupied[landlord][plotId].occupied`), the caller's staked token count (`munchablesStaked[mainAccount].length > 10`), and token ownership/approvals before transferring the NFT into `LandManager` at line 152:

```solidity
if (
    !munchNFT.isApprovedForAll(mainAccount, address(this)) &&
    munchNFT.getApproved(tokenId) != address(this)
) revert NotApprovedError();
munchNFT.transferFrom(mainAccount, address(this), tokenId);

plotOccupied[landlord][plotId] = Plot({
    occupied: true,
    tokenId: tokenId
});

munchablesStaked[mainAccount].push(tokenId);
munchableOwner[tokenId] = mainAccount;

toilerState[tokenId] = ToilerState({
    lastToilDate: block.timestamp,
    plotId: plotId,
    landlord: landlord,
    latestTaxRate: plotMetadata[landlord].currentTaxRate,
    dirty: false
});
```

Because `stakeMunchable` lacks a `nonReentrant` modifier and executes the external call `munchNFT.transferFrom(mainAccount, address(this), tokenId)` (line 152) prior to updating internal storage (lines 154–168), `LandManager` is left in an inconsistent intermediate state during the transfer call:

1. **Staking Cap Bypass (`munchablesStaked[mainAccount].length > 10`):** The length check at line 140 reads `munchablesStaked[mainAccount].length`, which is only incremented at line 159 after `transferFrom` returns. Reentering `stakeMunchable` during the transfer hook allows `mainAccount` to pass the length check repeatedly before any `push(tokenId)` executes, staking an arbitrary number of NFTs beyond the cap.
2. **Plot Occupancy Collision (`plotOccupied[landlord][plotId].occupied`):** The occupancy check at line 138 reads `plotOccupied[landlord][plotId].occupied`, which is only set to `true` at line 154. Reentering `stakeMunchable` with a different `tokenId` and the same `(landlord, plotId)` passes the check and assigns multiple tokens' `toilerState` to the same plot. Both tokens accrue schnibbles on a single landlord plot, and unstaking one resets `plotOccupied[landlord][plotId]` to unoccupied while the other remains actively farming on that plot.
3. **Inconsistent Settlement State in `_farmPlots`:** The `forceFarmPlots(msg.sender)` modifier runs at function entry and iterates over `munchablesStaked[mainAccount]`. During a reentrant call, the outer call's token has already been transferred to `LandManager` but is not yet registered in `munchablesStaked` or `toilerState`.

### Assumptions and preconditions

1. The caller (or associated `mainAccount`) is a registered player in `AccountManager` and owns multiple Munchable NFTs approved for `LandManager`.
2. A valid registered `landlord` has at least one available plot (`_getNumPlots(landlord) > plotId`).
3. Execution of `munchNFT.transferFrom` invokes a hook or callback capable of reentering `LandManager.stakeMunchable` (or inspecting `LandManager` state) before `transferFrom` finishes.

### Proof of concept

1. A registered player `mainAccount` holds 15 Munchable NFTs (`tokenId` 1 through 15) and approves `LandManager` for all of them.
2. `mainAccount` calls `LandManager.stakeMunchable(landlord, 1, 0)`.
3. All checks at lines 136–151 pass (`plotOccupied[landlord][0].occupied == false` and `munchablesStaked[mainAccount].length == 0`).
4. At line 152, `munchNFT.transferFrom(mainAccount, address(this), 1)` is called before `plotOccupied` or `munchablesStaked` is updated.
5. During the transfer callback for `tokenId = 1`, `mainAccount` reenters `stakeMunchable(landlord, 2, 0)` (and recursively for `tokenId` 3 through 15).
6. In each nested call, `plotOccupied[landlord][0].occupied` is still `false` and `munchablesStaked[mainAccount].length` is still `0`, so every call passes validation.
7. As the call stack unwinds, lines 154–168 execute for all 15 tokens: `munchablesStaked[mainAccount]` receives all 15 token IDs (bypassing the limit), and all 15 tokens have `toilerState[tokenId]` set to `plotId = 0` on `landlord`, allowing all 15 tokens to farm schnibbles simultaneously on a single plot in `_farmPlots`.

### Impact assessment

- **Severity:** Medium / High.
- **Technical Impact:** Violates the protocol invariants that at most one Munchable can occupy a given `(landlord, plotId)` at a time and that a player cannot stake more than the maximum allowed number of Munchables in `LandManager`.
- **Business Impact:** Attackers can multiply their schnibble farming rewards by stacking multiple Munchables onto a single landlord plot and exceeding the per-player staking limit, inflating reward generation at the expense of other players and corrupting plot occupancy tracking.

### Remediation

Follow the Checks-Effects-Interactions (CEI) pattern in `LandManager.stakeMunchable` by updating `plotOccupied`, `munchablesStaked`, `munchableOwner`, and `toilerState` before calling `munchNFT.transferFrom`, and consider adding a `nonReentrant` modifier to state-mutating functions in `LandManager`.

```solidity
        if (
            !munchNFT.isApprovedForAll(mainAccount, address(this)) &&
            munchNFT.getApproved(tokenId) != address(this)
        ) revert NotApprovedError();

        plotOccupied[landlord][plotId] = Plot({
            occupied: true,
            tokenId: tokenId
        });

        munchablesStaked[mainAccount].push(tokenId);
        munchableOwner[tokenId] = mainAccount;

        toilerState[tokenId] = ToilerState({
            lastToilDate: block.timestamp,
            plotId: plotId,
            landlord: landlord,
            latestTaxRate: plotMetadata[landlord].currentTaxRate,
            dirty: false
        });

        munchNFT.transferFrom(mainAccount, address(this), tokenId);

        emit FarmPlotTaken(toilerState[tokenId], tokenId);
    }
```

## Finding 3: Missing `dirty` Flag Reset and `lastToilDate` Update in `LandManager.transferToUnoccupiedPlot` Permanently Halts Farming After Plot Relocation

### Synopsis

In `LandManager.transferToUnoccupiedPlot`, relocating a Munchable from an invalidated plot to a valid unoccupied plot fails to reset `toilerState[tokenId].dirty` to `false` and update `toilerState[tokenId].lastToilDate` to `block.timestamp`. Consequently, the relocated Munchable remains marked `dirty` and is permanently skipped during future `_farmPlots` executions.

### Technical details

According to `ILandManager.sol`, `transferToUnoccupiedPlot` is intended to allow a renter whose plot became unavailable (because the landlord unlocked funds and reduced their total plots) to move their staked Munchable to a valid unoccupied plot and resume toiling.

When a landlord's plot count drops below a staked token's `plotId` (`_getNumPlots(landlord) < _toiler.plotId`), `_farmPlots` (which also executes via the `forceFarmPlots(msg.sender)` modifier at the start of `transferToUnoccupiedPlot`) caps the farming reward timestamp at `plotMetadata[landlord].lastUpdated` and marks the token as inactive by setting:
- `toilerState[tokenId].dirty = true` (line 260)
- `toilerState[tokenId].lastToilDate = plotMetadata[landlord].lastUpdated` (line 291)

On any subsequent invocation of `_farmPlots`, tokens with `_toiler.dirty == true` are skipped at line 251 (`if (_toiler.dirty) continue;`).

However, in `LandManager.transferToUnoccupiedPlot` (lines 203–219), the contract only updates `toilerState[tokenId].latestTaxRate` (lines 213–214) and `plotOccupied` (lines 215–222). It fails to:
1. Reset `toilerState[tokenId].dirty = false` so the token can resume farming on the new valid plot.
2. Update `toilerState[tokenId].lastToilDate = block.timestamp` so farming rewards on the new plot accrue only from the moment of relocation rather than retroactively from `plotMetadata[landlord].lastUpdated`.

Because `toilerState[tokenId].dirty` remains `true` after `transferToUnoccupiedPlot` completes, `_farmPlots` continues to skip the token indefinitely, defeating the core purpose of `transferToUnoccupiedPlot` and forcing the user to fully unstake and re-stake the Munchable to earn rewards again.

### Assumptions and preconditions

1. A renter has staked a Munchable (`tokenId`) onto a landlord's plot with `plotId`.
2. The landlord unlocks locked tokens such that `_getNumPlots(landlord) < plotId`, while at least one lower-indexed plot (`newPlotId < _getNumPlots(landlord)`) remains unoccupied.
3. The renter calls `transferToUnoccupiedPlot(tokenId, newPlotId)` to move their Munchable to the valid unoccupied plot.

### Proof of concept

1. Alice (landlord) locks tokens sufficient for 10 plots (`plotId` 0 through 9).
2. Bob (renter) stakes his Munchable (`tokenId = 1`) into Alice's `plotId = 9` via `stakeMunchable(alice, 1, 9)`.
3. At timestamp $T_1$, Alice unlocks a portion of her tokens so that `_getNumPlots(alice)` decreases to 5 (`plotId` 0 through 4 are valid; `plotId = 9 > 5`). `plotMetadata[alice].lastUpdated` is updated to $T_1$.
4. At timestamp $T_2 > T_1$, Bob calls `transferToUnoccupiedPlot(1, 0)` to move `tokenId = 1` to valid, unoccupied `plotId = 0`.
5. The `forceFarmPlots(msg.sender)` modifier runs `_farmPlots(bob)` before the function body:
   - `_getNumPlots(alice) < _toiler.plotId` (`5 < 9`) evaluates to `true`.
   - `toilerState[1].dirty` is set to `true`.
   - `toilerState[1].lastToilDate` is set to $T_1$.
6. The body of `transferToUnoccupiedPlot` executes and updates `toilerState[1].latestTaxRate` and `plotOccupied`, but leaves `toilerState[1].dirty == true` and `toilerState[1].lastToilDate == T_1`.
7. At timestamp $T_3 > T_2$, Bob calls `farmPlots()`. In `_farmPlots(bob)`, `if (_toiler.dirty) continue;` (line 251) triggers for `tokenId = 1`, accruing 0 schnibbles despite the Munchable having been relocated to valid `plotId = 0`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Relocated Munchables that were marked `dirty` due to a landlord's plot reduction remain permanently unable to farm schnibbles in `LandManager._farmPlots` after `transferToUnoccupiedPlot` is called. Furthermore, if `dirty` were cleared without updating `lastToilDate` to `block.timestamp`, the renter and landlord would retroactively farm schnibbles for the unplotted downtime between `plotMetadata[landlord].lastUpdated` and `block.timestamp`.
- **Business Impact:** Renters who use `transferToUnoccupiedPlot` as designed silently earn zero rewards for their staked Munchables unless they execute a full `unstakeMunchable` and `stakeMunchable` cycle.

### Remediation

In `LandManager.transferToUnoccupiedPlot`, reset `toilerState[tokenId].dirty` to `false` and update `toilerState[tokenId].lastToilDate` to `block.timestamp` (in addition to updating `toilerState[tokenId].plotId`).

```solidity
    function transferToUnoccupiedPlot(
        uint256 tokenId,
        uint256 plotId
    ) external override forceFarmPlots(msg.sender) notPaused {
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        ToilerState memory _toiler = toilerState[tokenId];
        uint256 oldPlotId = _toiler.plotId;
        uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
        if (_toiler.landlord == address(0)) revert NotStakedError();
        if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
        if (plotOccupied[_toiler.landlord][plotId].occupied)
            revert OccupiedPlotError(_toiler.landlord, plotId);
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();

        toilerState[tokenId].lastToilDate = block.timestamp;
        toilerState[tokenId].plotId = plotId;
        toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]
            .currentTaxRate;
        toilerState[tokenId].dirty = false;
        plotOccupied[_toiler.landlord][oldPlotId] = Plot({
            occupied: false,
            tokenId: 0
        });
        plotOccupied[_toiler.landlord][plotId] = Plot({
            occupied: true,
            tokenId: tokenId
        });

        emit FarmPlotLeave(_toiler.landlord, tokenId, oldPlotId);
        emit FarmPlotTaken(toilerState[tokenId], tokenId);
    }
```

## Finding 4: Incomplete `toilerState` update in `LandManager.transferToUnoccupiedPlot` prevents reward accrual recovery and corrupts plot state

### Synopsis

In `LandManager.transferToUnoccupiedPlot`, moving a staked Munchable to a new plot only updates `toilerState[tokenId].latestTaxRate` and fails to update `plotId`, reset `dirty` to `false`, or update `lastToilDate`. Consequently, a recovered Munchable never resumes earning schnibbles and permanently locks the newly occupied plot upon unstaking.

### Technical details

According to `ILandManager`, `transferToUnoccupiedPlot` is intended to allow a renter whose plot became invalid after the landlord unlocked tokens (`_getNumPlots(landlord) < _toiler.plotId`) to move their staked Munchable to a valid unoccupied plot.

When a landlord's available plots drop below `_toiler.plotId`, `_farmPlots` settles rewards up to `plotMetadata[landlord].lastUpdated` and marks the position as inactive by setting `toilerState[tokenId].dirty = true` (line 260). On any subsequent `_farmPlots` execution, `if (_toiler.dirty) continue;` (line 251) skips reward accrual for that token.

When the renter calls `transferToUnoccupiedPlot(tokenId, plotId)` (lines 199–226):
1. The `forceFarmPlots(msg.sender)` modifier runs `_farmPlots(msg.sender)`, which either transitions the out-of-bounds token to `dirty = true` (setting `lastToilDate = plotMetadata[landlord].lastUpdated`) or skips it if `dirty` was already `true`.
2. At lines 213–214, the function only updates `toilerState[tokenId].latestTaxRate`:
```solidity
toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]
    .currentTaxRate;
```
It omits updating the remaining state fields of `toilerState[tokenId]`:
- `toilerState[tokenId].dirty` is not reset to `false` (and `lastToilDate` is not set to `block.timestamp` when recovering from a `dirty` state, which would otherwise over-accrue from `plotMetadata[landlord].lastUpdated`).
- `toilerState[tokenId].plotId` is not updated to `plotId` and remains `oldPlotId`.

As a result:
- Because `dirty` remains `true` and `plotId` remains `oldPlotId`, `_farmPlots` continues to skip `tokenId` on all future calls (`if (_toiler.dirty) continue;`), so the Munchable never accrues schnibbles again unless completely unstaked and restaked.
- Because `toilerState[tokenId].plotId` still holds `oldPlotId`, calling `unstakeMunchable(tokenId)` (line 181) or calling `transferToUnoccupiedPlot` again (line 215) clears `plotOccupied[_toiler.landlord][oldPlotId]` instead of `plotOccupied[_toiler.landlord][plotId]`, leaving `plotOccupied[_toiler.landlord][plotId].occupied = true` permanently and bricking that plot for the landlord.

### Assumptions and preconditions

- A renter has staked a Munchable (`tokenId`) onto a landlord's plot (`oldPlotId`).
- Either the landlord reduces their locked tokens so `oldPlotId` becomes out-of-bounds (marking `toilerState[tokenId].dirty = true`), or the renter calls `transferToUnoccupiedPlot(tokenId, plotId)` to move to another valid unoccupied `plotId`.

### Proof of concept

1. Alice (landlord) locks enough tokens to have 10 plots (`plotId` 0 through 9).
2. Bob (renter) stakes `tokenId = 1` into Alice's plot `8` via `stakeMunchable(alice, 1, 8)`. `toilerState[1]` has `plotId = 8` and `dirty = false`.
3. Alice unlocks part of her tokens so `_getNumPlots(alice)` drops to `5` (`plotMetadata[alice].lastUpdated` is updated).
4. Later, Bob calls `transferToUnoccupiedPlot(1, 2)` to move `tokenId = 1` to valid unoccupied plot `2`:
   - `forceFarmPlots(bob)` runs `_farmPlots(bob)`. Since `_getNumPlots(alice) (5) < 8`, `toilerState[1].dirty` is set to `true` and `toilerState[1].lastToilDate` is set to `plotMetadata[alice].lastUpdated`.
   - `transferToUnoccupiedPlot` sets `plotOccupied[alice][8].occupied = false` and `plotOccupied[alice][2] = Plot({ occupied: true, tokenId: 1 })`.
   - However, `toilerState[1].plotId` remains `8`, `toilerState[1].dirty` remains `true`, and `toilerState[1].lastToilDate` remains `plotMetadata[alice].lastUpdated`.
5. Bob later calls `farmPlots()`. In `_farmPlots`, `_toiler.dirty` is `true`, so `tokenId = 1` is skipped and Bob earns `0` schnibbles for the time spent in plot `2`.
6. Bob calls `unstakeMunchable(1)`. It resets `plotOccupied[alice][8]` (since `_toiler.plotId == 8`) instead of `plotOccupied[alice][2]`. Plot `2` remains permanently marked as `occupied: true` even though no Munchable is staked in it.

### Impact assessment

- **Severity**: High
- **Direct Technical Effects**:
  1. **Permanent stop of reward accrual**: Any `dirty` Munchable moved via `transferToUnoccupiedPlot` remains `dirty = true` (and retains its out-of-bounds `plotId`), completely failing to resume schnibble farming.
  2. **Permanent plot DoS**: Moving a Munchable via `transferToUnoccupiedPlot` and subsequently unstaking (or moving it again) leaves `plotOccupied[landlord][plotId].occupied = true` permanently, making the landlord's plot unusable by any renter.

### Remediation

In `LandManager.transferToUnoccupiedPlot`, update `toilerState[tokenId].plotId` to `plotId`, and if the token was marked `dirty`, reset `dirty` to `false` and update `lastToilDate` to `block.timestamp` (since `_farmPlots` already settled rewards up to `plotMetadata[landlord].lastUpdated` when marking it `dirty`).

```solidity
    function transferToUnoccupiedPlot(
        uint256 tokenId,
        uint256 plotId
    ) external override forceFarmPlots(msg.sender) notPaused {
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        ToilerState memory _toiler = toilerState[tokenId];
        uint256 oldPlotId = _toiler.plotId;
        uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
        if (_toiler.landlord == address(0)) revert NotStakedError();
        if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
        if (plotOccupied[_toiler.landlord][plotId].occupied)
            revert OccupiedPlotError(_toiler.landlord, plotId);
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();

        if (_toiler.dirty) {
            toilerState[tokenId].lastToilDate = block.timestamp;
            toilerState[tokenId].dirty = false;
        }
        toilerState[tokenId].plotId = plotId;
        toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]
            .currentTaxRate;
        plotOccupied[_toiler.landlord][oldPlotId] = Plot({
            occupied: false,
            tokenId: 0
        });
        plotOccupied[_toiler.landlord][plotId] = Plot({
            occupied: true,
            tokenId: tokenId
        });

        emit FarmPlotLeave(_toiler.landlord, tokenId, oldPlotId);
        emit FarmPlotTaken(toilerState[tokenId], tokenId);
    }
```

## Finding 5: Missing `toilerState[tokenId].plotId` Update in `LandManager.transferToUnoccupiedPlot` Causes Plot Occupancy State Corruption

### Synopsis

In `LandManager.transferToUnoccupiedPlot`, when a staked Munchable is moved to a new plot, the contract updates `plotOccupied` for both the old and new plots but fails to update `toilerState[tokenId].plotId`. This state divergence allows a single staked token to permanently brick all of a landlord's plots or overwrite other renters' plot occupancy records.

### Technical details

The `LandManager` contract tracks plot assignments using two coupled mappings:
1. `plotOccupied[landlord][plotId]`: Maps a landlord's plot ID to a `Plot` struct (`occupied`, `tokenId`).
2. `toilerState[tokenId]`: Maps a staked `tokenId` to a `ToilerState` struct containing `plotId`, `landlord`, `latestTaxRate`, `lastToilDate`, and `dirty`.

In `LandManager.transferToUnoccupiedPlot(uint256 tokenId, uint256 plotId)` (lines 203–222), the function reads the current state into memory (`ToilerState memory _toiler = toilerState[tokenId]`) and caches `uint256 oldPlotId = _toiler.plotId;`. After validating the caller and target `plotId`, it updates `toilerState[tokenId].latestTaxRate` (lines 213–214), clears `plotOccupied[_toiler.landlord][oldPlotId]` (lines 215–218), and marks `plotOccupied[_toiler.landlord][plotId]` as occupied by `tokenId` (lines 219–222).

However, `transferToUnoccupiedPlot` never updates `toilerState[tokenId].plotId = plotId`. Consequently, `toilerState[tokenId].plotId` retains the stale `oldPlotId` value while `plotOccupied[_toiler.landlord][plotId]` marks the new plot as occupied.

This desynchronization between `toilerState` and `plotOccupied` corrupts subsequent state transitions:
- **Plot Exhaustion (DoS):** If the renter calls `transferToUnoccupiedPlot(tokenId, plotId_2)` a second time, `oldPlotId` is still read as the initial plot (`plotId_0`). The contract frees `plotId_0` again and marks `plotId_2` as occupied, while `plotId_1` remains permanently marked as `occupied: true` in `plotOccupied`. By repeating this across all available plots, a renter with a single staked NFT can occupy every plot belonging to the landlord.
- **Clobbering Another Renter's Occupancy on Unstake or Transfer:** Once `oldPlotId` is freed by the first transfer, another renter can stake their Munchable into `oldPlotId`. When the first renter subsequently calls `unstakeMunchable(tokenId)` (or `transferToUnoccupiedPlot` again), `LandManager` resets `plotOccupied[_toiler.landlord][_toiler.plotId]` (`oldPlotId`) to `occupied: false, tokenId: 0`, erasing the new renter's plot occupancy record while leaving `plotId` permanently occupied and unfreeable.
- **Incorrect Farming Calculations:** In `_farmPlots`, the check `_getNumPlots(landlord) < _toiler.plotId` and the emitted `FarmedSchnibbles` event continue to evaluate against the stale `oldPlotId` instead of the new `plotId`.

### Assumptions and preconditions

1. A landlord has locked tokens providing at least two available plots (`_getNumPlots(landlord) >= 2`).
2. A registered renter owns at least one Munchable NFT and stakes it into one of the landlord's unoccupied plots (`plotId = 0`).

### Proof of concept

1. **Initial Stake:** Renter Alice calls `stakeMunchable(landlord, tokenId, 0)`.
   - `plotOccupied[landlord][0]` is `{ occupied: true, tokenId: tokenId }`.
   - `toilerState[tokenId].plotId` is `0`.
2. **First Relocation:** Alice calls `transferToUnoccupiedPlot(tokenId, 1)`.
   - `oldPlotId` is read from `toilerState[tokenId].plotId` (`0`).
   - `plotOccupied[landlord][0]` is set to `{ occupied: false, tokenId: 0 }`.
   - `plotOccupied[landlord][1]` is set to `{ occupied: true, tokenId: tokenId }`.
   - `toilerState[tokenId].plotId` remains `0` because it is not updated.
3. **Second Relocation / DoS:** Alice calls `transferToUnoccupiedPlot(tokenId, 2)` (or `unstakeMunchable(tokenId)`).
   - `oldPlotId` is again read as `0` from `toilerState[tokenId].plotId`.
   - `plotOccupied[landlord][0]` is set to `{ occupied: false, tokenId: 0 }` (which wipes out another user's plot record if someone staked in plot `0` in the interim).
   - `plotOccupied[landlord][2]` is set to `{ occupied: true, tokenId: tokenId }`.
   - `plotOccupied[landlord][1]` remains `{ occupied: true, tokenId: tokenId }` permanently, and no function can ever set `plotOccupied[landlord][1].occupied` back to `false`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** `toilerState[tokenId].plotId` and `plotOccupied[landlord][plotId]` permanently diverge after any call to `transferToUnoccupiedPlot`.
- **Business Implications:** Any renter with a single Munchable NFT can permanently brick all unoccupied plots of any landlord by sequentially transferring across them, rendering those plots unusable (`OccupiedPlotError`) for the lifetime of the contract. Furthermore, if another user stakes into the vacated `oldPlotId`, their `plotOccupied` entry will be overwritten as unoccupied when the original token is transferred again or unstaked, breaking exclusive plot allocation.

### Remediation

Update `toilerState[tokenId].plotId` to the new `plotId` in `transferToUnoccupiedPlot`.

```solidity
    function transferToUnoccupiedPlot(
        uint256 tokenId,
        uint256 plotId
    ) external override forceFarmPlots(msg.sender) notPaused {
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        ToilerState memory _toiler = toilerState[tokenId];
        uint256 oldPlotId = _toiler.plotId;
        uint256 totalPlotsAvail = _getNumPlots(_toiler.landlord);
        if (_toiler.landlord == address(0)) revert NotStakedError();
        if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();
        if (plotOccupied[_toiler.landlord][plotId].occupied)
            revert OccupiedPlotError(_toiler.landlord, plotId);
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();

        toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]
            .currentTaxRate;
        toilerState[tokenId].plotId = plotId;
        plotOccupied[_toiler.landlord][oldPlotId] = Plot({
            occupied: false,
            tokenId: 0
        });
        plotOccupied[_toiler.landlord][plotId] = Plot({
            occupied: true,
            tokenId: tokenId
        });

        emit FarmPlotLeave(_toiler.landlord, tokenId, oldPlotId);
        emit FarmPlotTaken(toilerState[tokenId], tokenId);
    }
```

## Finding 6: Incomplete Plot Invalidation and Reactivation Transition in `LandManager._farmPlots` Permanently Halts Rewards for Occupied Plots

### Synopsis

In `LandManager._farmPlots`, `toilerState[tokenId].dirty` transitions one-way to `true` when a landlord's plot capacity drops, and `if (_toiler.dirty) continue;` unconditionally skips the token thereafter. When plot capacity recovers, the plot remains marked occupied while the staked Munchable stays permanently invalidated and earns zero Schnibbles.

### Technical details

In `LandManager._farmPlots`, lines 251–260 manage plot eligibility when a landlord's available plot count (`_getNumPlots(landlord)`) changes relative to a staked Munchable's `_toiler.plotId`:

1. When `_getNumPlots(landlord)` falls below `_toiler.plotId` (e.g., because the landlord unlocks tokens or token USD prices decrease), line 259 sets `timestamp = plotMetadata[landlord].lastUpdated` and line 260 marks `toilerState[tokenId].dirty = true`.
2. However, marking `toilerState[tokenId].dirty = true` does not release `plotOccupied[landlord][_toiler.plotId]` (which remains `{ occupied: true, tokenId: tokenId }`), nor does `_farmPlots` provide a reactivation transition if `_getNumPlots(landlord)` subsequently increases back above `_toiler.plotId` (e.g., when the landlord locks additional tokens or token USD value recovers).
3. On every subsequent invocation of `_farmPlots`, line 251 (`if (_toiler.dirty) continue;`) executes before `_getNumPlots(landlord)` is checked and unconditionally skips the token.

As a result, occupancy and eligibility state become desynchronized: `plotOccupied[landlord][_toiler.plotId].occupied` remains `true` (preventing any other renter from staking on that plot), while `toilerState[tokenId].dirty` remains stuck at `true` (preventing the staked Munchable from ever accruing Schnibbles for the renter or landlord again unless the renter manually unstakes and re-stakes).

### Assumptions and preconditions

1. A registered renter has staked a Munchable (`tokenId`) on a registered landlord's plot (`plotId`).
2. The landlord's `_getNumPlots(landlord)` temporarily drops below `plotId` (via token unlock or token price reduction), and `_farmPlots` is triggered for the renter (directly via `farmPlots()` or via the `forceFarmPlots` modifier).
3. The landlord's `_getNumPlots(landlord)` subsequently increases so that `plotId` is within the landlord's available plot capacity again.

### Proof of concept

1. A landlord locks tokens sufficient for 5 plots (`_getNumPlots(landlord) == 5`), and a renter calls `stakeMunchable(landlord, tokenId, 4)` at $t = 100$. At this point, `plotOccupied[landlord][4]` is `{ occupied: true, tokenId: tokenId }` and `toilerState[tokenId].dirty` is `false`.
2. At $t = 200$, the landlord unlocks part of their tokens so `_getNumPlots(landlord)` drops to `2`, updating `plotMetadata[landlord].lastUpdated = 200`.
3. At $t = 300$, the renter calls `farmPlots()`. In `_farmPlots`, `_getNumPlots(landlord) < _toiler.plotId` (`2 < 4`) is true at line 258. Line 259 sets `timestamp = 200` and line 260 sets `toilerState[tokenId].dirty = true`. Schnibbles are accrued for $[100, 200]$.
4. At $t = 400$, the landlord locks more tokens so `_getNumPlots(landlord)` returns to `5` (`plotMetadata[landlord].lastUpdated = 400`). Plot `4` is now within capacity and is still occupied by `tokenId` (`plotOccupied[landlord][4].occupied == true`).
5. At $t = 500$, the renter calls `farmPlots()`. At line 251, `if (_toiler.dirty) continue;` skips `tokenId`. Neither the renter nor the landlord earns any Schnibbles for the `[400, 500]` period during which plot `4` was valid and occupied.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: Once a plot is marked `dirty` in `_farmPlots`, the state transition is irreversible without a full unstake/re-stake cycle. The plot remains marked as `occupied` in `plotOccupied` while `toilerState[tokenId].dirty` permanently blocks reward accrual in `_farmPlots`.
- **Business Impact**: Renters and landlords lose Schnibbles rewards on valid, occupied plots after temporary capacity reductions, and landlords suffer dead plot capacity that cannot be rented by other players while the `dirty` Munchable remains staked.

### Remediation

In `LandManager._farmPlots`, check current plot eligibility before skipping a `dirty` toiler so that if the plot is still invalid it is skipped, and if plot capacity has been restored, the toiler is reactivated (`toilerState[tokenId].dirty = false`) with `lastToilDate` synchronized to `plotMetadata[landlord].lastUpdated`.

```solidity
            landlord = _toiler.landlord;
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                if (_toiler.dirty) continue;
                timestamp = plotMetadata[landlord].lastUpdated;
                toilerState[tokenId].dirty = true;
            } else if (_toiler.dirty) {
                toilerState[tokenId].dirty = false;
                _toiler.lastToilDate = plotMetadata[landlord].lastUpdated;
            }
```

## Finding 7: Flawed Plot Invalidation Timestamp and Checkpoint Accounting in `LandManager._farmPlots` Causes Reward Miscalculation and Underflow DoS

### Synopsis

In `LandManager._farmPlots`, out-of-bounds plot detection relies on an unsynchronized single timestamp (`plotMetadata[landlord].lastUpdated`), an off-by-one capacity check (`<` instead of `<=`), and an irreversible `dirty` flag, causing claim-timing-dependent reward miscalculations, permanent loss of reward accrual, or arithmetic underflow that locks staked Munchables.

### Technical details

When a renter stakes a Munchable on a landlord's plot, `LandManager._farmPlots` calculates accrued Schnibbles between `_toiler.lastToilDate` and `timestamp`. Lines 251–260 attempt to cap reward accrual when a landlord's available plot capacity (`_getNumPlots(landlord)`) falls below the staked `_toiler.plotId`:

```solidity
if (_toiler.dirty) continue;
landlord = _toiler.landlord;
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

This temporal reward accounting mechanism is fundamentally flawed due to unsynchronized capacity and settlement checkpoints:

1. **Non-Monotonic Cutoff Timestamp and Underflow DoS (`timestamp < _toiler.lastToilDate`):**
   `plotMetadata[landlord].lastUpdated` is not guaranteed to be greater than or equal to `_toiler.lastToilDate`. If a renter stakes or farms at timestamp $T_2 > \text{plotMetadata[landlord].lastUpdated}$ ($T_1$), `_toiler.lastToilDate` becomes $T_2$. If `_getNumPlots(landlord)` subsequently drops below `_toiler.plotId` without `plotMetadata[landlord].lastUpdated` being updated past $T_2$ (e.g., when token USD prices or `PRICE_PER_PLOT` change, or when tokens are unlocked without updating `plotMetadata`), line 259 sets `timestamp = T_1 < _toiler.lastToilDate`. The subsequent calculation on line 281:
   ```solidity
   schnibblesTotal = (timestamp - _toiler.lastToilDate) * BASE_SCHNIBBLE_RATE;
   ```
   reverts due to arithmetic underflow. Because `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` all apply the `forceFarmPlots(msg.sender)` modifier, any revert in `_farmPlots` permanently freezes all Munchables staked by the renter.

2. **Claim-Timing-Dependent Entitlement and Overwritten Metadata Timestamps:**
   `plotMetadata[landlord].lastUpdated` stores only a single timestamp per landlord rather than tracking the exact timestamp at which each individual plot became unbacked. As noted in the inline comments (lines 255–257), if a landlord reduces plot capacity at $T_a$ and later updates `plotMetadata` again at $T_b > T_a$ (while `_getNumPlots(landlord)` remains below `_toiler.plotId`) before the renter calls `farmPlots()`, `plotMetadata[landlord].lastUpdated` is overwritten with $T_b$. When the renter claims at $T_c > T_b$, rewards are accrued up to $T_b$ instead of $T_a$, granting unbacked rewards for $[T_a, T_b]$. Similarly, if the landlord unlocks at $T_a$ and re-locks sufficient capacity at $T_b$ before the renter settles, `_getNumPlots(landlord) < _toiler.plotId` evaluates to `false` at claim time, accruing full rewards over the unbacked interval $[T_a, T_b]$.

3. **Off-by-One Plot Capacity Boundary Check:**
   Plot IDs are zero-indexed in the range $[0, \text{totalPlotsAvail} - 1]$, as enforced in `stakeMunchable` (line 146) and `transferToUnoccupiedPlot` (line 211) via `if (plotId >= totalPlotsAvail) revert PlotTooHighError();`. However, line 258 checks `_getNumPlots(landlord) < _toiler.plotId` instead of `_getNumPlots(landlord) <= _toiler.plotId`. Consequently, a toiler staked on plot index `N` when `_getNumPlots(landlord)` drops to `N` is never detected as occupying an invalid plot and continues accruing full rewards indefinitely.

4. **Irreversible `dirty` Flag Across State Updates:**
   Once `toilerState[tokenId].dirty` is set to `true` on line 260, line 251 (`if (_toiler.dirty) continue;`) permanently skips the token in future `_farmPlots` calls. Even if the landlord re-locks enough tokens to restore the plot's validity, or if the renter calls `transferToUnoccupiedPlot` (which neither resets `dirty` nor updates `toilerState[tokenId].plotId`), the token never accrues rewards again unless unstaked and re-staked.

### Assumptions and preconditions

- A landlord has locked tokens in `LockManager`, providing initial plot capacity.
- A registered renter stakes a Munchable on one of the landlord's plots via `stakeMunchable`.
- The landlord's effective plot count (`_getNumPlots(landlord)`) changes over time due to unlocking tokens, additional lock operations, or USD price / `PRICE_PER_PLOT` adjustments.

### Proof of concept

#### Scenario 1: Arithmetic Underflow Freezing Staked Munchables
1. At $t = 100$, Landlord locks tokens yielding 10 plots (`plotId` 0 through 9). `plotMetadata[landlord].lastUpdated` is set to `100`.
2. At $t = 200$, Renter stakes `tokenId = 1` on `plotId = 8`. `toilerState[1].lastToilDate` is set to `200`.
3. At $t = 300$, the USD price of the landlord's locked token drops (or `PRICE_PER_PLOT` is increased, or the landlord unlocks tokens without advancing `plotMetadata[landlord].lastUpdated`), reducing `_getNumPlots(landlord)` from `10` to `5`. `plotMetadata[landlord].lastUpdated` remains `100`.
4. At $t = 400$, Renter calls `unstakeMunchable(1)` (or `farmPlots()`).
5. The `forceFarmPlots` modifier invokes `_farmPlots`. For `tokenId = 1`, `_getNumPlots(landlord) < _toiler.plotId` (`5 < 8`) is `true`.
6. Line 259 sets `timestamp = plotMetadata[landlord].lastUpdated = 100`.
7. Line 281 computes `timestamp - _toiler.lastToilDate = 100 - 200`, which reverts with an arithmetic underflow. Renter cannot farm, transfer plots, or unstake `tokenId = 1`.

#### Scenario 2: Overwritten Cutoff Timestamp Across Multiple Landlord Updates
1. At $t = 100$, Landlord has 10 plots (`lastUpdated = 100`) and Renter stakes `tokenId = 1` on `plotId = 8` (`lastToilDate = 100`).
2. At $t = 200$, Landlord reduces locked tokens so `_getNumPlots(landlord)` becomes `2`, and `plotMetadata[landlord].lastUpdated` becomes `200`. Renter does not call `farmPlots()`.
3. At $t = 500$, Landlord locks a small additional amount of tokens (still fewer than 8 plots, e.g., 3 plots). `AccountManager.forceHarvest` calls `LandManager.updatePlotMetadata(landlord)`, overwriting `plotMetadata[landlord].lastUpdated` to `500`.
4. At $t = 600$, Renter calls `farmPlots()`. Line 259 sets `timestamp = 500`, rewarding Renter and Landlord for the entire `[200, 500]` period during which `plotId = 8` was unbacked.

### Impact assessment

- **Severity:** High
- **Technical Impact:**
  - **Permanent Freezing of Assets (DoS):** Whenever `plotMetadata[landlord].lastUpdated < _toiler.lastToilDate` for an invalidated plot, `_farmPlots` reverts due to underflow, blocking `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` for all Munchables staked by that renter.
  - **Incorrect Reward Accounting:** Renters and landlords accrue unauthorized Schnibbles across unbacked time intervals when `plotMetadata[landlord].lastUpdated` is overwritten by subsequent updates, when capacity is restored before settlement, or when `_getNumPlots(landlord) == _toiler.plotId` (due to the `<` off-by-one error).

### Remediation

Fix the boundary comparison from `<` to `<=`, clamp the cutoff `timestamp` so it can never be less than `_toiler.lastToilDate`, and track per-plot invalidation checkpoints (or settle active toilers upon plot capacity reductions) as well as resetting `dirty` and updating `plotId` in `transferToUnoccupiedPlot`. Within `_farmPlots`:

```solidity
            if (_toiler.dirty) continue;
            landlord = _toiler.landlord;
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated > _toiler.lastToilDate
                    ? plotMetadata[landlord].lastUpdated
                    : _toiler.lastToilDate;
                toilerState[tokenId].dirty = true;
            }
```

Additionally, ensure `updatePlotMetadata` is invoked whenever a landlord unlocks tokens in `LockManager`, and in `transferToUnoccupiedPlot`, update `toilerState[tokenId].plotId = plotId`, `toilerState[tokenId].lastToilDate = block.timestamp`, and `toilerState[tokenId].dirty = false`.

## Finding 8: Off-by-One Plot Boundary Check in `LandManager._farmPlots` Allows Farming on Invalidated Plots

### Synopsis

In `LandManager._farmPlots`, the plot validity check uses a strict inequality (`_getNumPlots(landlord) < _toiler.plotId`) instead of `<=`. Because plot IDs are zero-indexed, a toiler staked on the boundary plot index equal to `_getNumPlots(landlord)` (including `plotId == 0` when available plots drop to `0`) bypasses invalidation and continues earning Schnibbles indefinitely.

### Technical details

In `LandManager`, plots are zero-indexed from `0` to `totalPlotsAvail - 1`, where `totalPlotsAvail` is returned by `_getNumPlots(landlord)`. This zero-indexed boundary is enforced during plot assignment in both `stakeMunchable` (line 146) and `transferToUnoccupiedPlot` (line 211):

```solidity
uint256 totalPlotsAvail = _getNumPlots(landlord);
if (plotId >= totalPlotsAvail) revert PlotTooHighError();
```

Accordingly, a plot with index `_toiler.plotId` becomes invalid whenever `_toiler.plotId >= _getNumPlots(landlord)` (or equivalently, `_getNumPlots(landlord) <= _toiler.plotId`).

However, when `_farmPlots` checks whether a landlord's plot capacity has fallen below the staked Munchable's `plotId` (line 258), it uses `<` instead of `<=`:

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

Due to this off-by-one discrepancy:
1. When a landlord's available plots decrease to `N` while a renter is staked in `plotId == N` (the `N + 1`-th plot), `_getNumPlots(landlord) < _toiler.plotId` evaluates to `N < N` (`false`).
2. Most critically, if a renter is staked in `plotId == 0` and the landlord unlocks all of their locked tokens so that `_getNumPlots(landlord)` becomes `0`, the check `0 < 0` always evaluates to `false`.

Consequently, `timestamp` is not capped to `plotMetadata[landlord].lastUpdated` and `toilerState[tokenId].dirty` is never set to `true`. Both the renter and the landlord continue to accrue `unfedSchnibbles` at the full `block.timestamp` indefinitely on an unbacked plot.

### Assumptions and preconditions

1. A landlord has locked tokens providing at least `N + 1` plots (`_getNumPlots(landlord) >= N + 1`, where `N >= 0`).
2. A renter stakes a Munchable on `plotId = N` (e.g., `plotId = 0`).
3. The landlord unlocks tokens (or token USD prices drop) such that `_getNumPlots(landlord)` decreases to `N` (e.g., `0` plots available).

### Proof of concept

1. A landlord locks enough tokens in `LockManager` to have 1 available plot (`_getNumPlots(landlord) == 1`). The only valid plot index is `plotId = 0` (since `plotId >= 1` reverts with `PlotTooHighError`).
2. A renter calls `stakeMunchable(landlord, tokenId, 0)` to stake their Munchable into `plotId = 0`.
3. After the lock duration expires, the landlord unlocks all their tokens, reducing `_getNumPlots(landlord)` to `0`.
4. Later, the renter calls `farmPlots()`.
5. Inside `_farmPlots`, line 258 evaluates `_getNumPlots(landlord) < _toiler.plotId`, which is `0 < 0` (`false`).
6. Instead of capping rewards at `plotMetadata[landlord].lastUpdated` and marking `toilerState[tokenId].dirty = true`, `_farmPlots` uses `timestamp = block.timestamp` and mints Schnibbles to both the renter and the landlord as if the plot were still validly backed by locked tokens. The renter can repeat this indefinitely.

### Impact assessment

- **Severity:** Medium / High
- **Technical Effect:** Renters staked on `plotId == _getNumPlots(landlord)` (and always on `plotId == 0` even when the landlord has 0 plots) are never marked `dirty` and never have their farming timestamp capped to `plotMetadata[landlord].lastUpdated`.
- **Business Impact:** Renters and landlords can farm illegitimate Schnibbles (and consequently Munch Points) indefinitely without keeping any capital locked in the protocol, undermining the core token-locking incentive model of `LandManager`.

### Remediation

Update the boundary condition in `LandManager._farmPlots` to use `<=` instead of `<` so that it matches the zero-indexed plot bounds enforced across the contract:

```solidity
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated;
                toilerState[tokenId].dirty = true;
            }
```

## Finding 9: Lossy integer narrowing of `RARITY_BONUSES` in `LandManager._farmPlots` corrupts reward bonus calculation

### Synopsis

In `LandManager._farmPlots`, the `uint8` rarity bonus retrieved from `RARITY_BONUSES` is narrowed to `int8` before being widened to `int16` when computing `finalBonus`. Any configured `uint8` rarity bonus in the range `[128, 255]` wraps around to a negative value, penalizing farming rewards instead of boosting them.

### Technical details

In `LandManager`, `RARITY_BONUSES` is declared as a dynamic array of unsigned 8-bit integers (`uint8[] RARITY_BONUSES;`, loaded via `configStorage.getSmallUintArray(StorageKey.RarityBonuses)`), which supports non-negative bonus percentages up to `255`.

However, in `LandManager._farmPlots` (lines 270–279), `finalBonus` is computed as follows:

```solidity
finalBonus =
    int16(
        REALM_BONUSES[
            (uint256(immutableAttributes.realm) * 5) +
                uint256(landlordMetadata.snuggeryRealm)
        ]
    ) +
    int16(
        int8(RARITY_BONUSES[uint256(immutableAttributes.rarity)])
    );
```

And similarly in `BonusManager.getFeedBonus`, the `uint8` value is cast through `int8` before being widened to `int16`.

Because `RARITY_BONUSES[uint256(immutableAttributes.rarity)]` is of type `uint8`, casting it first to `int8` reinterprets the most significant bit as a sign bit whenever the configured rarity bonus is greater than `127` (i.e., in the valid `uint8` range `[128, 255]`). When this `int8` value is subsequently sign-extended to `int16`, it remains negative (in the range `[-128, -1]`). In addition, `finalBonus` in `_farmPlots` does not enforce the `[-20%, +100%]` aggregate bonus clamping boundaries that `BonusManager.getFeedBonus` applies.

Directly converting `uint8` to `uint16` before casting to `int16` (`int16(uint16(RARITY_BONUSES[...]))`) preserves the full unsigned `[0, 255]` value range without sign-bit corruption.

### Assumptions and preconditions

1. `ConfigStorage` has `StorageKey.RarityBonuses` configured with a `uint8` value greater than `127` (e.g., `150` for a high-tier rarity), or `finalBonus` exceeds expected multiplier bounds.
2. A registered renter stakes a Munchable of that rarity in `LandManager` and triggers `_farmPlots` (directly via `farmPlots()` or via the `forceFarmPlots` modifier on `stakeMunchable`, `unstakeMunchable`, or `transferToUnoccupiedPlot`).

### Proof of concept

1. Suppose `RARITY_BONUSES` for `Rarity.Mythic` (index 5) is configured in `ConfigStorage` to `130` (a valid `uint8` value representing a `130%` bonus), and the corresponding `REALM_BONUSES` entry is `10`.
2. A player stakes a `Mythic` Munchable on a landlord's plot and later calls `farmPlots()`.
3. During `_farmPlots`, `RARITY_BONUSES[5]` (`uint8(130)`, binary `0x82`) is cast via `int8(130)`, which evaluates to `-126`.
4. Widening to `int16` sign-extends the value to `int16(-126)`.
5. `finalBonus` is computed as `int16(10) + int16(-126) = -116` instead of `10 + 130 = +140`.
6. When applied to `schnibblesTotal` in line 283, `int256(schnibblesTotal) + (int256(schnibblesTotal) * (-116))` becomes negative, which subsequently wraps around on the `uint256(...)` cast and causes `_farmPlots` (and `unstakeMunchable`) to revert due to arithmetic overflow/underflow when updating `unfedSchnibbles` or computing `schnibblesLandlord`.

### Impact assessment

- **Severity**: Medium
- **Technical Effect**: Lossy narrowing from `uint8` to `int8` flips positive rarity bonuses `>= 128` into negative penalties.
- **Business Impact**: High-rarity Munchables with large configured bonuses suffer severe reward reduction or cause a denial of service in `_farmPlots`, locking staked NFTs in `LandManager` since `unstakeMunchable` executes `_farmPlots` via the `forceFarmPlots` modifier.

### Remediation

Widen the `uint8` element from `RARITY_BONUSES` to `uint16` before casting it to `int16` so that all `uint8` values remain positive.

```solidity
            finalBonus =
                int16(
                    REALM_BONUSES[
                        (uint256(immutableAttributes.realm) * 5) +
                            uint256(landlordMetadata.snuggeryRealm)
                    ]
                ) +
                int16(
                    uint16(RARITY_BONUSES[uint256(immutableAttributes.rarity)])
                );
```

## Finding 10: Flawed Bonus Calculation in `LandManager._farmPlots` Causes Underflow Revert and Incorrect Schnibbles Accrual

### Synopsis

In `LandManager._farmPlots`, misplaced parentheses in the `schnibblesTotal` bonus calculation divide the base `schnibblesTotal` by `100`. This drastically underpays rewards when `finalBonus >= 0` and wraps around to near `type(uint256).max` when `finalBonus < 0`, causing an overflow revert that permanently locks staked Munchables.

### Technical details

In `LandManager._farmPlots`, a staked Munchable accrues base schnibbles over time on lines 280–282:

```solidity
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
```

Next, lines 270–279 compute `finalBonus` by summing `REALM_BONUSES` (which contains values in `[-10, -5, 0, 5, 10]`) and `RARITY_BONUSES` (which contains values in `[0, 0, 10, 20, 30, 50]`). Each unit of `finalBonus` represents a `1%` bonus or penalty (spanning from `-10%` to `+60%`).

However, lines 283–286 apply `finalBonus` to `schnibblesTotal` as follows:

```solidity
schnibblesTotal = uint256(
    (int256(schnibblesTotal) +
        (int256(schnibblesTotal) * finalBonus)) / 100
);
```

Because the outer parentheses enclose the addition of `int256(schnibblesTotal)` and `int256(schnibblesTotal) * finalBonus` before dividing by `100`, the expression evaluates to:

$$\text{schnibblesTotal} = \left\lfloor \frac{\text{schnibblesTotal} \times (1 + \text{finalBonus})}{100} \right\rfloor$$

instead of the intended:

$$\text{schnibblesTotal} = \text{schnibblesTotal} + \left\lfloor \frac{\text{schnibblesTotal} \times \text{finalBonus}}{100} \right\rfloor$$

This introduces two critical failure modes:

1. **Negative `finalBonus` causes `uint256` underflow and permanent DoS:** When a Munchable has `Primordial` or `Common` rarity (`RARITY_BONUSES` of `0`) and a realm mismatch with the landlord's `snuggeryRealm` that yields a `REALM_BONUSES` of `-5` or `-10`, `finalBonus` is `-5` or `-10`. For any non-zero farming duration (`schnibblesTotal >= 100`), `(int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)) / 100` evaluates to a negative `int256` value (`-4 * schnibblesTotal / 100` or `-9 * schnibblesTotal / 100`). Casting this negative `int256` to `uint256` on line 283 wraps the value to near `type(uint256).max`. Immediately afterward, line 288 computes `(schnibblesTotal * _toiler.latestTaxRate) / 1e18`, which reverts due to arithmetic multiplication overflow. Because `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` all execute `_farmPlots(msg.sender)` via the `forceFarmPlots` modifier, the renter's staked Munchables are permanently frozen in `LandManager`.
2. **Non-negative `finalBonus` severely underpays rewards:** When `finalBonus >= 0`, dividing the base `int256(schnibblesTotal)` by `100` reduces the base reward by 99% (for example, `finalBonus == 0` yields `1%` of base schnibbles instead of `100%`, and `finalBonus == 10` yields `11%` instead of `110%`), shortchanging both `renterMetadata.unfedSchnibbles` (line 295) and `landlordMetadata.unfedSchnibbles` (line 298).

### Assumptions and preconditions

- `REALM_BONUSES` and `RARITY_BONUSES` are configured with their standard percentage values (where `REALM_BONUSES` includes `-10` and `-5`, and `RARITY_BONUSES` includes `0` for `Primordial` and `Common`).
- A registered renter stakes a Munchable on a registered landlord's plot, and at least one second elapses (`timestamp > _toiler.lastToilDate`).

### Proof of concept

1. A landlord registers with `snuggeryRealm = Realm.Drench` (`1`) and locks tokens to provide available plots with `DEFAULT_TAX_RATE = 25e16`.
2. A renter owns a `Common` (`Rarity.Common = 1`, `RARITY_BONUSES[1] = 0`) Munchable from `Realm.Everfrost` (`0`).
3. The renter calls `stakeMunchable(landlord, tokenId, 0)` at `t = 1000`.
4. At `t = 1100` (100 seconds later), the renter calls `unstakeMunchable(tokenId)` (or `farmPlots()`):
   - `forceFarmPlots` triggers `_farmPlots(renter)`.
   - `finalBonus` is computed as `REALM_BONUSES[(0 * 5) + 1] + RARITY_BONUSES[1] = -10 + 0 = -10`.
   - Base `schnibblesTotal` on lines 280–282 is `100 * 1e15 = 1e17`.
   - On lines 283–286, `(int256(1e17) + (int256(1e17) * (-10))) / 100 = -9e15`.
   - Casting `-9e15` to `uint256` produces `type(uint256).max - 9e15 + 1`.
   - On line 288, multiplying this value by `_toiler.latestTaxRate` (`25e16`) overflows `uint256` and reverts with panic `0x11`.
5. The renter can never unstake `tokenId` or any other Munchable they have staked in `LandManager`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  - Arithmetic underflow when casting a negative `int256` to `uint256` (when `finalBonus < 0`), followed by arithmetic overflow revert on tax calculation.
  - Miscalculation of `schnibblesTotal` (reduced by 99% of the base amount) for all `finalBonus >= 0`.
- **Business & Protocol Implications:**
  - Any renter who stakes a Munchable with a net negative bonus (`finalBonus < 0`) experiences a permanent denial of service on `_farmPlots`, freezing all of their staked Munchable NFTs inside `LandManager` since `unstakeMunchable` cannot execute without `_farmPlots` succeeding.
  - All other stakers and landlords lose the vast majority of their base Schnibbles yield.

### Remediation

Divide only the bonus term `(int256(schnibblesTotal) * finalBonus)` by `100` before adding it to `int256(schnibblesTotal)`:

```solidity
            schnibblesTotal =
                (timestamp - _toiler.lastToilDate) *
                BASE_SCHNIBBLE_RATE;
            schnibblesTotal = uint256(
                int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus) / 100
            );
            schnibblesLandlord =
                (schnibblesTotal * _toiler.latestTaxRate) /
                1e18;
```
