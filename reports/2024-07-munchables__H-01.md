# Audit report — 2024-07-munchables/H-01

## Finding 1: Missing Landlord Registration and `plotMetadata` Initialization Check in `LandManager.stakeMunchable` Causes Permanent Freezing of Staked NFTs

### Synopsis

`LandManager.stakeMunchable` does not verify that the target `landlord` is a registered main account with initialized `plotMetadata`. Staking onto an unregistered or uninitialized landlord's plot causes subsequent `_farmPlots` executions to revert, permanently locking the renter's staked Munchable NFTs in `LandManager`.

### Technical details

In `LandManager.stakeMunchable`, a renter stakes a Munchable NFT onto a `landlord`'s plot after checking plot occupancy (`plotOccupied[landlord][plotId].occupied`) and plot availability (`_getNumPlots(landlord)`):

```solidity
if (plotOccupied[landlord][plotId].occupied)
    revert OccupiedPlotError(landlord, plotId);
if (munchablesStaked[mainAccount].length > 10)
    revert TooManyStakedMunchiesError();
if (munchNFT.ownerOf(tokenId) != mainAccount)
    revert InvalidOwnerError();

uint256 totalPlotsAvail = _getNumPlots(landlord);
if (plotId >= totalPlotsAvail) revert PlotTooHighError();
```

`_getNumPlots(landlord)` queries `lockManager.getLockedWeightedValue(landlord) / PRICE_PER_PLOT`. However, `LockManager` allows users to lock tokens before registering in `AccountManager`. When `AccountManager` is notified by `LockManager` during a lock operation (lines 148–156 of `AccountManager.sol`), it only calls `landManager.updatePlotMetadata(_player)` if `player.registrationDate != 0`.

Consequently, an address can have non-zero `_getNumPlots(landlord)` while:
1. Not being registered in `AccountManager` (`registrationDate == 0`), and/or
2. Having uninitialized `plotMetadata[landlord]` (`plotMetadata[landlord].lastUpdated == 0` and `currentTaxRate == 0`).

Because `stakeMunchable` does not validate that `plotMetadata[landlord].lastUpdated != 0` (nor that `landlord` is a registered main account), staking succeeds and records the token in `munchablesStaked[mainAccount]` and `toilerState[tokenId]`.

Every subsequent state-changing action by the renter—including `unstakeMunchable`, `transferToUnoccupiedPlot`, `stakeMunchable`, and `farmPlots`—invokes `_farmPlots(msg.sender)` via the `forceFarmPlots` modifier. Inside `_farmPlots`:
- Line 265 calls `_getMainAccountRequireRegistered(landlord)`, which reverts with `PlayerNotRegisteredError` if `landlord` is unregistered.
- Even if `landlord` registered after locking tokens (so `registrationDate != 0`, while `plotMetadata[landlord].lastUpdated` remains `0`) and later reduces their locked value below `_toiler.plotId`, line 259 sets `timestamp = plotMetadata[landlord].lastUpdated` (`0`), causing line 281 (`timestamp - _toiler.lastToilDate`) to revert due to arithmetic underflow.

In both scenarios, `_farmPlots` always reverts, making it impossible for the renter to call `unstakeMunchable` for any of their staked NFTs.

### Assumptions and preconditions

1. An address (`landlord`) has locked tokens in `LockManager` sufficient to yield at least one plot (`_getNumPlots(landlord) > 0`) prior to registering in `AccountManager`, leaving `plotMetadata[landlord].lastUpdated == 0`.
2. A registered renter calls `stakeMunchable(landlord, tokenId, plotId)` with a valid `tokenId` and `plotId < _getNumPlots(landlord)`.

### Proof of concept

1. Alice locks 1 ETH in `LockManager` without registering in `AccountManager`. Because `players[alice].registrationDate == 0`, `landManager.updatePlotMetadata(alice)` is not called, leaving `plotMetadata[alice].lastUpdated = 0`, while `_getNumPlots(alice) > 0`.
2. Bob (a registered player owning `tokenId = 1`) approves `LandManager` and calls `stakeMunchable(alice, 1, 0)`.
3. `_farmPlots(bob)` runs first (with 0 staked tokens), all checks in `stakeMunchable` pass since `_getNumPlots(alice) > 0`, and `tokenId = 1` is transferred to `LandManager`.
4. Bob subsequently calls `unstakeMunchable(1)` to retrieve his NFT.
5. The `forceFarmPlots(msg.sender)` modifier triggers `_farmPlots(bob)`, which loops over `munchablesStaked[bob]` (`tokenId = 1`) and calls `_getMainAccountRequireRegistered(alice)` at line 265.
6. Because Alice is not registered (`registrationDate == 0`), `_getMainAccountRequireRegistered(alice)` reverts with `PlayerNotRegisteredError`. Bob's Munchable NFT (as well as any other Munchables Bob staked previously) is permanently frozen in `LandManager`.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Complete denial of service on `_farmPlots`, `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` for the affected renter, resulting in permanent freezing of the renter's Munchable NFTs held by `LandManager` unless the landlord registers and initializes their plot metadata.
- **Business Impact**: Violates the core protocol invariant against deadlocks where a user's Munchable NFT becomes permanently stuck in the contract.

### Remediation

Validate in `stakeMunchable` that the landlord's `plotMetadata` has been initialized (`plotMetadata[landlord].lastUpdated != 0`), which is only possible when the landlord is a registered main account that has either had `updatePlotMetadata` called via `AccountManager` or invoked `triggerPlotMetadata()`.

```solidity
        if (plotOccupied[landlord][plotId].occupied)
            revert OccupiedPlotError(landlord, plotId);
        if (munchablesStaked[mainAccount].length > 10)
            revert TooManyStakedMunchiesError();
        if (munchNFT.ownerOf(tokenId) != mainAccount)
            revert InvalidOwnerError();
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();

        uint256 totalPlotsAvail = _getNumPlots(landlord);
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();
```

## Finding 2: Off-by-One Stake Limit Check in `LandManager.stakeMunchable` Allows Players to Stake 11 Munchables Instead of 10

### Synopsis

An off-by-one error in the stake-count boundary check of `LandManager.stakeMunchable` uses `> 10` instead of `>= 10` before pushing the newly staked token to `munchablesStaked[mainAccount]`. This allows any registered player to stake up to 11 Munchables simultaneously instead of the intended cap of 10, earning extra schnibbles rewards.

### Technical details

In `LandManager.stakeMunchable`, the contract enforces a maximum limit on the number of Munchables a single main account can stake at once:

```solidity
if (munchablesStaked[mainAccount].length > 10)
    revert TooManyStakedMunchiesError();
```

This validation is performed at lines 140–141, prior to appending the new `tokenId` to the `munchablesStaked[mainAccount]` array at line 159:

```solidity
munchablesStaked[mainAccount].push(tokenId);
```

When a player has already staked 10 Munchables, `munchablesStaked[mainAccount].length` is `10`. If the player calls `stakeMunchable` for an 11th Munchable:
1. The condition `munchablesStaked[mainAccount].length > 10` evaluates `10 > 10`, which is `false`.
2. The transaction does not revert with `TooManyStakedMunchiesError`.
3. Execution proceeds to line 159, pushing the 11th `tokenId` into `munchablesStaked[mainAccount]` and increasing its length to `11`.

Only when the player attempts to stake a 12th Munchable does `munchablesStaked[mainAccount].length > 10` evaluate to `true` (`11 > 10`). Consequently, the maximum staked Munchables cap is off by one, permitting 11 staked NFTs per account instead of 10.

### Assumptions and preconditions

1. A registered player (`mainAccount`) owns at least 11 Munchable NFTs and has approved `LandManager` to transfer them.
2. One or more registered landlords have at least 11 unoccupied plots available (`_getNumPlots(landlord)` is sufficient for the chosen `plotId`s).
3. The `LandManager` contract is not paused.

### Proof of concept

1. A registered player (`bob`) owns 11 Munchable NFTs with `tokenId`s `1` through `11` and approves `LandManager` for all 11 tokens.
2. A landlord (`alice`) locks enough tokens in `LockManager` to have at least 11 available plots (`plotId` `0` through `10`).
3. `bob` calls `stakeMunchable(alice, i, i - 1)` for `i` from `1` to `10`. After the 10th call, `munchablesStaked[bob].length` is `10`.
4. `bob` calls `stakeMunchable(alice, 11, 10)`.
5. At line 140, `munchablesStaked[bob].length > 10` evaluates `10 > 10` (`false`), bypassing the `TooManyStakedMunchiesError` revert.
6. At line 159, `munchablesStaked[bob].push(11)` executes, resulting in `munchablesStaked[bob].length == 11`.
7. Subsequent calls to `farmPlots()` iterate over all 11 staked Munchables in `munchablesStaked[bob]`, accruing schnibbles for the extra 11th Munchable.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** The per-account staking capacity invariant is violated, allowing 11 Munchables to be staked in `munchablesStaked[mainAccount]` instead of the intended maximum of 10.
- **Business Impact:** Players can stake 10% more Munchables than intended by the game economy design, generating additional schnibbles for both the renter and the landlord on the 11th staked NFT. Additionally, the loop in `_farmPlots` iterates over 11 elements instead of 10 on every state-changing call decorated with `forceFarmPlots`.

### Remediation

Update the boundary check in `LandManager.stakeMunchable` to revert when `munchablesStaked[mainAccount].length >= 10`:

```solidity
        if (plotOccupied[landlord][plotId].occupied)
            revert OccupiedPlotError(landlord, plotId);
        if (munchablesStaked[mainAccount].length >= 10)
            revert TooManyStakedMunchiesError();
        if (munchNFT.ownerOf(tokenId) != mainAccount)
            revert InvalidOwnerError();
```

## Finding 3: Incomplete `toilerState` Update in `LandManager.transferToUnoccupiedPlot` Causes Plot State Corruption and Permanently Halts Reward Accrual

### Synopsis

In `LandManager.transferToUnoccupiedPlot`, transferring a staked Munchable to a new plot updates `plotOccupied` and `latestTaxRate` but fails to update `toilerState[tokenId].plotId`, reset `dirty`, or sync `lastToilDate`. This corrupts plot occupancy tracking, breaks capacity-based reward eligibility checks, and permanently halts Schnibble farming for relocated tokens.

### Technical details

According to `ILandManager`, `transferToUnoccupiedPlot(uint256 tokenId, uint256 plotId)` is intended to allow a renter whose plot became invalid (after the landlord unlocked funds and reduced `_getNumPlots(landlord)`) to move their staked Munchable to a valid, unoccupied `plotId`.

However, `transferToUnoccupiedPlot` (lines 199–226 in `src/managers/LandManager.sol`) only updates `toilerState[tokenId].latestTaxRate` (lines 213–214) and `plotOccupied` (lines 215–222), while omitting essential updates to `toilerState[tokenId]`:

1. **Missing `toilerState[tokenId].plotId = plotId` update:**
   `toilerState[tokenId].plotId` remains equal to `oldPlotId`. Consequently:
   - During subsequent `_farmPlots` executions, the capacity eligibility check `_getNumPlots(landlord) < _toiler.plotId` (line 258) evaluates against `oldPlotId` instead of the new `plotId`.
   - When the renter later calls `unstakeMunchable(tokenId)` (line 181) or calls `transferToUnoccupiedPlot` again (line 215), the contract vacates `plotOccupied[_toiler.landlord][_toiler.plotId]` (`oldPlotId`) instead of the actual `plotId`. This leaves `plotOccupied[_toiler.landlord][plotId].occupied = true` permanently stuck as occupied even after the token is unstaked or moved again, and can overwrite a new occupant at `oldPlotId`.
   - The `FarmPlotTaken` event emitted at line 225 logs the stale `oldPlotId`.

2. **Missing `dirty` reset and `lastToilDate` checkpoint update:**
   `transferToUnoccupiedPlot` executes the `forceFarmPlots(msg.sender)` modifier before the function body. When a renter moves a Munchable out of an invalidated plot where `_getNumPlots(landlord) < _toiler.plotId`, `_farmPlots` sets `toilerState[tokenId].dirty = true` (line 260) and sets `toilerState[tokenId].lastToilDate = plotMetadata[landlord].lastUpdated` (lines 259, 291). Because `transferToUnoccupiedPlot` never resets `toilerState[tokenId].dirty = false` and never updates `toilerState[tokenId].lastToilDate = block.timestamp`, all future calls to `_farmPlots` hit `if (_toiler.dirty) continue;` (line 251), permanently preventing the staked Munchable from accruing any further Schnibble rewards.

### Assumptions and preconditions

- A registered renter has staked a Munchable NFT on a registered landlord's plot via `LandManager.stakeMunchable`.
- The landlord has at least one other unoccupied valid plot (`plotId < _getNumPlots(landlord)`).
- Either the renter's original plot became invalid due to the landlord unlocking tokens (setting `dirty = true` during `_farmPlots`), or the renter transfers their Munchable to another unoccupied plot via `transferToUnoccupiedPlot`.

### Proof of concept

1. Alice (landlord) locks tokens sufficient for 10 plots (`plotId` 0 through 9).
2. Bob (renter) stakes `tokenId = 1` onto Alice's `plotId = 9` via `stakeMunchable(alice, 1, 9)`. At this point, `toilerState[1].plotId == 9` and `toilerState[1].dirty == false`.
3. Alice unlocks part of her tokens so `_getNumPlots(alice)` drops to 5 (`plotId` 0 through 4). `plotMetadata[alice].lastUpdated` is updated to the unlock timestamp `t_unlock`.
4. Later at `t_move > t_unlock`, Bob calls `transferToUnoccupiedPlot(1, 2)` to move `tokenId = 1` to valid `plotId = 2`:
   - The `forceFarmPlots(msg.sender)` modifier runs `_farmPlots(bob)`, which sees `_getNumPlots(alice) < 9`, farms rewards up to `t_unlock`, sets `toilerState[1].lastToilDate = t_unlock`, and sets `toilerState[1].dirty = true`.
   - `transferToUnoccupiedPlot` sets `plotOccupied[alice][9].occupied = false` and `plotOccupied[alice][2] = Plot({occupied: true, tokenId: 1})`.
   - However, `toilerState[1].plotId` remains `9`, `toilerState[1].dirty` remains `true`, and `toilerState[1].lastToilDate` remains `t_unlock`.
5. Bob waits 1 day and calls `farmPlots()`. In `_farmPlots`, `_toiler.dirty` is `true` (and `_toiler.plotId` is still `9`), so the loop executes `if (_toiler.dirty) continue;`. Bob and Alice receive 0 Schnibbles despite `tokenId = 1` occupying valid `plotId = 2`.
6. Bob calls `unstakeMunchable(1)`. The function clears `plotOccupied[alice][9]` (from `_toiler.plotId`) instead of `plotOccupied[alice][2]`. Alice's `plotId = 2` remains permanently marked as `occupied: true` with no staked Munchable able to use it.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  - Relocated Munchables fail to update `toilerState[tokenId].plotId`, `dirty`, and `lastToilDate`.
  - Any Munchable moved from an invalidated plot via `transferToUnoccupiedPlot` remains permanently `dirty` and accrues zero Schnibble rewards for both the renter and the landlord.
  - Unstaking or re-transferring a relocated Munchable clears `oldPlotId` instead of the current `plotId`, permanently bricking the new plot (`occupied: true`) and potentially corrupting occupancy state at `oldPlotId` if another token staked there in the meantime.

### Remediation

In `LandManager.transferToUnoccupiedPlot`, update `toilerState[tokenId].plotId` to the new `plotId`, reset `toilerState[tokenId].dirty` to `false`, and update `toilerState[tokenId].lastToilDate` to `block.timestamp`.

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

## Finding 4: Arithmetic Underflow in `LandManager._farmPlots` Permanently Locks Staked Munchables

### Synopsis

In `LandManager._farmPlots`, when a landlord's available plot count falls below a staked token's `plotId`, the accrual `timestamp` is set to `plotMetadata[landlord].lastUpdated`. If `plotMetadata[landlord].lastUpdated` is earlier than `_toiler.lastToilDate` (or uninitialized at `0`), `timestamp - _toiler.lastToilDate` underflows and reverts, permanently locking the renter's staked NFTs.

### Technical details

When a renter calls `farmPlots()`, `stakeMunchable()`, `unstakeMunchable()`, or `transferToUnoccupiedPlot()`, `LandManager._farmPlots(address _sender)` iterates over all tokens in `munchablesStaked[mainAccount]`.

In each iteration, `timestamp` is initialized to `block.timestamp` (line 248). However, if the landlord's current number of plots is less than `_toiler.plotId` (lines 258–261), `_farmPlots` overwrites `timestamp` with `plotMetadata[landlord].lastUpdated`:

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

Subsequently, at lines 280–282, the contract computes the accrued schnibbles using unchecked-assumed subtraction in Solidity 0.8.25:

```solidity
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
```

`plotMetadata[landlord].lastUpdated` can be strictly less than `_toiler.lastToilDate` in several realistic scenarios:
1. **Uninitialized `plotMetadata`**: `stakeMunchable` does not verify that `plotMetadata[landlord].lastUpdated != 0`. If a landlord locked tokens prior to `LandManager` deployment (or without `plotMetadata` being initialized) and never called `triggerPlotMetadata()`, `plotMetadata[landlord].lastUpdated` remains `0` while `_toiler.lastToilDate` is the staking timestamp (`> 0`).
2. **Plot Reduction Without `plotMetadata` Update**: `_getNumPlots(landlord)` is dynamically computed as `lockManager.getLockedWeightedValue(landlord) / PRICE_PER_PLOT`. When the USD price of the landlord's locked tokens decreases in `LockManager`, or when `PRICE_PER_PLOT` increases via configuration, or when `LockManager` unlocks tokens without updating `plotMetadata` (note that `updatePlotMetadata` is restricted to `onlyConfiguredContract(StorageKey.AccountManager)`), `_getNumPlots(landlord)` decreases while `plotMetadata[landlord].lastUpdated` remains at an old timestamp prior to the renter's latest `_toiler.lastToilDate`.

Additionally, the boundary check `_getNumPlots(landlord) < _toiler.plotId` at line 258 uses `<` instead of `<=`, whereas valid `plotId` values are `0`-indexed (`plotId < totalPlotsAvail` in `stakeMunchable` at line 146).

When `timestamp < _toiler.lastToilDate`, the subtraction `timestamp - _toiler.lastToilDate` reverts with an arithmetic underflow. Because `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` all apply the `forceFarmPlots(msg.sender)` modifier, any attempt by the renter to unstake or move any of their staked Munchables will unconditionally revert.

### Assumptions and preconditions

- A landlord has sufficient locked weighted value in `LockManager` to provide plots (`_getNumPlots(landlord) > plotId`).
- A renter stakes a Munchable into `plotId` (setting `_toiler.lastToilDate = block.timestamp`) or farms plots at a timestamp later than `plotMetadata[landlord].lastUpdated`.
- The landlord's `_getNumPlots(landlord)` subsequently drops below `_toiler.plotId` (e.g., via token unlocking or token USD price reduction) while `plotMetadata[landlord].lastUpdated < _toiler.lastToilDate`.

### Proof of concept

1. Alice (landlord) locks tokens at $t = 1000$, giving her 10 plots (`plotId` `0` through `9`). Suppose `plotMetadata[alice].lastUpdated` is `1000` (or `0` if Alice locked before `LandManager` deployment and did not call `triggerPlotMetadata()`).
2. At $t = 2000$, Bob (renter) calls `stakeMunchable(alice, tokenId, 5)`. This sets `toilerState[tokenId].lastToilDate = 2000`.
3. At $t = 3000$, Alice's available plots drop to `4` (either because the locked token's `usdPrice` decreases or Alice unlocks tokens without `plotMetadata[alice].lastUpdated` being updated).
4. At $t = 4000$, Bob calls `unstakeMunchable(tokenId)` to withdraw his NFT.
5. The `forceFarmPlots(msg.sender)` modifier executes `_farmPlots(bob)`:
   - `_getNumPlots(alice)` is `4`, which is `< _toiler.plotId` (`5`).
   - `timestamp` is set to `plotMetadata[alice].lastUpdated` (`1000` or `0`).
   - Line 281 evaluates `timestamp - _toiler.lastToilDate` (`1000 - 2000`), which reverts due to arithmetic underflow.
6. Bob cannot call `unstakeMunchable`, `transferToUnoccupiedPlot`, `stakeMunchable`, or `farmPlots`. All of Bob's staked Munchables in `LandManager` are permanently frozen.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Denial of Service on `_farmPlots`, `farmPlots`, `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` for the affected renter.
- **Business Impact**: Permanent freezing (deadlock) of the renter's staked Munchable NFTs inside `LandManager` and loss of all accrued schnibbles, directly violating the core protocol invariant against stuck Munchables.

### Remediation

Ensure that `plotMetadata[landlord]` is initialized before staking, fix the out-of-bounds plot comparison to `<=`, ensure `LockManager` updates `plotMetadata` when unlocking, and guard the `timestamp` in `_farmPlots` so it is never less than `_toiler.lastToilDate`:

```solidity
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated > _toiler.lastToilDate
                    ? plotMetadata[landlord].lastUpdated
                    : _toiler.lastToilDate;
                toilerState[tokenId].dirty = true;
            }
```

## Finding 5: Stale `renterMetadata` Snapshot and Canonical-Account Mismatch in `LandManager._farmPlots` Causes Lost Schnibble and Metadata Updates

### Synopsis

In `LandManager._farmPlots`, the renter's `Player` struct (`renterMetadata`) is snapshotted into memory once before the farming loop and written back after the loop, while `landlordMetadata` is read from `landlord`'s canonical main account and written to `players[landlord]` inside each iteration, causing lost updates and cross-account state corruption when accounts alias or use sub-accounts.

### Technical details

In `src/managers/LandManager.sol`, `_farmPlots(address _sender)` manages Schnibble distribution between the renter (`_sender`) and the `landlord` of each staked Munchable:

```solidity
    function _farmPlots(address _sender) internal {
        (
            address mainAccount,
            MunchablesCommonLib.Player memory renterMetadata
        ) = _getMainAccountRequireRegistered(_sender);

        uint256[] memory staked = munchablesStaked[mainAccount];
        ...
        for (uint8 i = 0; i < staked.length; i++) {
            timestamp = block.timestamp;
            ...
            (
                ,
                MunchablesCommonLib.Player memory landlordMetadata
            ) = _getMainAccountRequireRegistered(landlord);
            ...
            renterMetadata.unfedSchnibbles += (schnibblesTotal -
                schnibblesLandlord);

            landlordMetadata.unfedSchnibbles += schnibblesLandlord;
            landlordMetadata.lastPetMunchable = uint32(timestamp);
            accountManager.updatePlayer(landlord, landlordMetadata);
            ...
        }
        accountManager.updatePlayer(mainAccount, renterMetadata);
    }
```

This accounting flow introduces two intertwined cross-account consistency flaws:

1. **Pre-loop `renterMetadata` snapshot and lost updates on role overlap / aliasing:**
   At lines 233–236, `_getMainAccountRequireRegistered(_sender)` loads `players[mainAccount]` into the in-memory `renterMetadata` struct once before the `for` loop (line 247). Inside the loop, `renterMetadata.unfedSchnibbles` is accumulated only in memory (lines 295–296), whereas `landlordMetadata` is fetched from `accountManager.getPlayer(landlord)` (lines 262–265), modified with `schnibblesLandlord` and `lastPetMunchable = uint32(timestamp)`, and written directly to `AccountManager` storage via `accountManager.updatePlayer(landlord, landlordMetadata)` (line 300). After the loop completes, `accountManager.updatePlayer(mainAccount, renterMetadata)` (line 309) writes the in-memory `renterMetadata` struct back to `players[mainAccount]`. If the renter's `mainAccount` and the landlord's account refer to the same canonical player record, the in-loop `landlordMetadata` read does not see the in-memory `renterMetadata` changes, and the post-loop write of `renterMetadata` at line 309 overwrites `players[mainAccount]` with the stale snapshot taken at lines 233–236—erasing all `schnibblesLandlord` and `lastPetMunchable` updates persisted during the loop.

2. **Canonical-account read/write mismatch for `landlord`:**
   In `stakeMunchable`, `if (landlord == mainAccount) revert CantStakeToSelfError()` only compares `mainAccount` against the raw `landlord` address without resolving `landlord` to its canonical main account via `_getMainAccountRequireRegistered(landlord)`. In `_farmPlots`, line 265 calls `_getMainAccountRequireRegistered(landlord)`—which resolves `landlord` via `accountManager.getPlayer(landlord)` to `players[_getMainAccount(landlord)]`—but discards the returned canonical `_mainAccount` address (`( , MunchablesCommonLib.Player memory landlordMetadata)`). Line 300 then calls `accountManager.updatePlayer(landlord, landlordMetadata)`, which writes `players[landlord] = landlordMetadata` directly to the raw `landlord` key rather than the canonical main account. On subsequent loop iterations for the same landlord, `_getMainAccountRequireRegistered(landlord)` again reads from the unchanged canonical main account snapshot, losing prior iterations' `schnibblesLandlord` increments and duplicating main-account player state into the sub-account slot.

### Assumptions and preconditions

1. `LandManager` is properly configured and authorized in `AccountManager` to call `updatePlayer`.
2. A player has one or more Munchable NFTs staked in `LandManager` and triggers `_farmPlots` (either directly via `farmPlots()` or via the `forceFarmPlots` modifier on `stakeMunchable`, `unstakeMunchable`, or `transferToUnoccupiedPlot`).
3. Either the `landlord` address aliases / resolves to the same canonical `mainAccount` as the renter, or `landlord` is a non-canonical (sub-account) address of a registered main account across one or more staked plots.

### Proof of concept

1. Suppose player `A` is a registered main account with `players[A].unfedSchnibbles = 1000` and `players[A].lastPetMunchable = 100`.
2. A Munchable is staked to a plot where the landlord resolves to `A` (or a sub-account `S` whose `mainAccounts[S] = A`).
3. After time passes, `_farmPlots` is invoked for `A`:
   - At lines 233–236, `mainAccount = A` and `renterMetadata` is loaded into memory with `unfedSchnibbles = 1000` and `lastPetMunchable = 100`.
   - Inside the loop (line 247), suppose `schnibblesTotal = 500` and `schnibblesLandlord = 100`.
   - At lines 262–265, `_getMainAccountRequireRegistered(landlord)` resolves `landlord` to `A` and loads `landlordMetadata` with `unfedSchnibbles = 1000`.
   - At lines 295–296, `renterMetadata.unfedSchnibbles` in memory becomes `1000 + 400 = 1400` (with `renterMetadata.lastPetMunchable` still `100`).
   - At lines 298–300, `landlordMetadata.unfedSchnibbles` becomes `1000 + 100 = 1100`, `landlordMetadata.lastPetMunchable` becomes `uint32(block.timestamp)`, and `accountManager.updatePlayer(landlord, landlordMetadata)` writes this struct.
   - If `landlord == A`, `players[A]` is updated to `unfedSchnibbles = 1100`, `lastPetMunchable = uint32(block.timestamp)`, and then at line 309 `accountManager.updatePlayer(A, renterMetadata)` overwrites `players[A]` with `unfedSchnibbles = 1400` and `lastPetMunchable = 100`, losing the `100` landlord Schnibbles and the `lastPetMunchable` update.
   - If `landlord == S` (a sub-account of `A` or of another landlord `L`), line 300 writes `L`'s `Player` struct into `players[S]` (registering `S` as a duplicate player in storage) while leaving `players[L].unfedSchnibbles` and `players[L].lastPetMunchable` completely uncredited; if multiple Munchables are staked to `S`, each loop iteration re-reads the unchanged `players[L]` and overwrites `players[S]`, losing all prior iterations' landlord Schnibbles.

### Impact assessment

**Severity: Medium / High**

- **Direct Technical Effect:** Inconsistent cross-account `Player` struct reads and writes in `_farmPlots` cause landlord Schnibbles (`schnibblesLandlord`) and `lastPetMunchable` timestamps to be overwritten or written to the wrong storage key in `AccountManager.players`.
- **Business Impact:** Accrued farming rewards (`unfedSchnibbles`, which are used to feed Munchables in `SnuggeryManager.feed` and `PrimordialManager.feedPrimordial`) are permanently lost for affected accounts, and sub-account entries in `AccountManager.players` can be corrupted with cloned main-account metadata.

### Remediation

Resolve `landlord` to its canonical `landlordMainAccount`, persist or synchronize `renterMetadata` and `landlordMetadata` consistently through canonical main accounts on each iteration (or reload/merge `renterMetadata` when `landlordMainAccount == mainAccount`), and ensure `accountManager.updatePlayer` is always called with the canonical main account address:

```solidity
    function _farmPlots(address _sender) internal {
        (
            address mainAccount,
            MunchablesCommonLib.Player memory renterMetadata
        ) = _getMainAccountRequireRegistered(_sender);

        uint256[] memory staked = munchablesStaked[mainAccount];
        MunchablesCommonLib.NFTImmutableAttributes memory immutableAttributes;
        ToilerState memory _toiler;
        uint256 timestamp;
        address landlord;
        address landlordMainAccount;
        uint256 tokenId;
        int256 finalBonus;
        uint256 schnibblesTotal;
        uint256 schnibblesLandlord;
        for (uint8 i = 0; i < staked.length; i++) {
            timestamp = block.timestamp;
            tokenId = staked[i];
            _toiler = toilerState[tokenId];
            if (_toiler.dirty) continue;
            landlord = _toiler.landlord;
            if (_getNumPlots(landlord) < _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated;
                toilerState[tokenId].dirty = true;
            }
            (
                landlordMainAccount,
                MunchablesCommonLib.Player memory landlordMetadata
            ) = _getMainAccountRequireRegistered(landlord);

            immutableAttributes = nftAttributesManager.getImmutableAttributes(
                tokenId
            );
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
            schnibblesTotal =
                (timestamp - _toiler.lastToilDate) *
                BASE_SCHNIBBLE_RATE;
            schnibblesTotal = uint256(
                (int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus)) / 100
            );
            schnibblesLandlord =
                (schnibblesTotal * _toiler.latestTaxRate) /
                1e18;

            toilerState[tokenId].lastToilDate = timestamp;
            toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord]
                .currentTaxRate;

            if (landlordMainAccount == mainAccount) {
                renterMetadata.unfedSchnibbles += schnibblesTotal;
                renterMetadata.lastPetMunchable = uint32(timestamp);
            } else {
                renterMetadata.unfedSchnibbles += (schnibblesTotal -
                    schnibblesLandlord);
                landlordMetadata.unfedSchnibbles += schnibblesLandlord;
                landlordMetadata.lastPetMunchable = uint32(timestamp);
                accountManager.updatePlayer(
                    landlordMainAccount,
                    landlordMetadata
                );
            }

            emit FarmedSchnibbles(
                _toiler.landlord,
                tokenId,
                _toiler.plotId,
                schnibblesTotal - schnibblesLandlord,
                schnibblesLandlord
            );
        }
        accountManager.updatePlayer(mainAccount, renterMetadata);
    }
```

## Finding 6: Incorrect operator precedence in `LandManager._farmPlots` divides base schnibbles by 100 and underflows on negative `finalBonus`

### Synopsis

In `LandManager._farmPlots`, `schnibblesTotal` is calculated as `uint256((int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)) / 100)` instead of dividing only the bonus term by `100`. This reduces base farming rewards by 100x and causes signed-to-unsigned wraparound or overflow reverts whenever `finalBonus` is negative.

### Technical details

In `src/managers/LandManager.sol`, `_farmPlots` computes the realm and rarity bonus for each staked Munchable and applies it to `schnibblesTotal`:

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
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
schnibblesTotal = uint256(
    (int256(schnibblesTotal) +
        (int256(schnibblesTotal) * finalBonus)) / 100
);
```

`finalBonus` represents an integer percentage bonus (for example, `10` for `+10%` or `-10` for `-10%`, matching `BonusManager.getFeedBonus` where `sumBonuses` is scaled by `1e16`, i.e., `1e18 / 100`). To apply `finalBonus` as a percentage modifier to `schnibblesTotal`, the intended formula is:

$$\text{schnibblesTotal} = \text{schnibblesTotal} + \frac{\text{schnibblesTotal} \times \text{finalBonus}}{100}$$

However, because the addition `int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)` is wrapped in parentheses before dividing by `100`, the code instead evaluates:

$$\text{schnibblesTotal} = \frac{\text{schnibblesTotal} \times (1 + \text{finalBonus})}{100}$$

This flawed scaling has two severe consequences:
1. **Incorrect reward scaling when `finalBonus >= 0`:** The base `schnibblesTotal` is itself divided by `100`. For example, if `finalBonus == 0`, the user receives only `1%` (`schnibblesTotal / 100`) of their earned schnibbles; if `finalBonus == 10` (`+10%`), the user receives `11%` (`11 * schnibblesTotal / 100`) instead of `110%`.
2. **Signedness underflow / revert when `finalBonus < -1`:** When the realm penalty exceeds the rarity bonus such that `finalBonus <= -2` (e.g., `-5` or `-10`), `int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)` evaluates to a negative `int256`. Casting this negative value to `uint256` wraps it to a massive value near `type(uint256).max`. On the next line (`(schnibblesTotal * _toiler.latestTaxRate) / 1e18`), multiplying this huge `uint256` by `_toiler.latestTaxRate` overflows and reverts with a panic (`0x11`), permanently blocking `_farmPlots` (and all functions using `forceFarmPlots`: `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot`).

### Assumptions and preconditions

- A registered renter has staked at least one Munchable NFT in `LandManager`.
- Time has elapsed since `_toiler.lastToilDate` (`timestamp > _toiler.lastToilDate`).
- For the negative-bonus DoS/underflow case, the staked Munchable's realm and the landlord's `snuggeryRealm` produce a negative `finalBonus` (e.g., `-5` or `-10`).

### Proof of concept

1. Alice (renter) stakes a Munchable `tokenId` on Bob's (landlord's) plot at `t = 1000`, with `BASE_SCHNIBBLE_RATE = 1e15` and `latestTaxRate = 0.2e18` (20%).
2. Case A (`finalBonus = 10`, i.e., +10% bonus):
   - At `t = 2000` (`1000` seconds later), base `schnibblesTotal = 1000 * 1e15 = 1e18`.
   - Expected total with +10% bonus: `1e18 + (1e18 * 10) / 100 = 1.1e18`.
   - Actual computation in `_farmPlots`: `uint256((1e18 + (1e18 * 10)) / 100) = 0.11e18` (10x lower than intended).
3. Case B (`finalBonus = -10`, i.e., -10% penalty):
   - At `t = 2000`, base `schnibblesTotal = 1e18`.
   - Expected total with -10% penalty: `1e18 + (1e18 * -10) / 100 = 0.9e18`.
   - Actual computation in `_farmPlots`: `(1e18 + (1e18 * -10)) / 100 = -9e16`.
   - Casting `int256(-9e16)` to `uint256` yields `type(uint256).max - 9e16 + 1`.
   - Line 288 computes `(schnibblesTotal * _toiler.latestTaxRate) / 1e18`, which overflows `uint256` and reverts, locking Alice's staked Munchable in `LandManager` because `unstakeMunchable` triggers `forceFarmPlots(msg.sender)`.

### Impact assessment

- **Severity:** High.
- **Technical & Business Impact:**
  - All positive or zero bonus farming rewards in `LandManager` are miscalculated (base schnibbles are divided by 100).
  - Any renter who stakes a Munchable with a net negative `finalBonus` (`<= -2`) suffers a permanent denial of service on `_farmPlots` once time elapses, making it impossible for the user to call `farmPlots`, `stakeMunchable`, `unstakeMunchable`, or `transferToUnoccupiedPlot`, permanently freezing all of their staked Munchable NFTs in `LandManager`.

### Remediation

Divide only the bonus term `(int256(schnibblesTotal) * finalBonus)` by `100` before adding it to `int256(schnibblesTotal)`:

```solidity
            schnibblesTotal = uint256(
                int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus) / 100
            );
```

## Finding 7: Off-by-One Plot Count Check in `LandManager._farmPlots` Allows Farming on Invalidated Boundary Plot

### Synopsis

In `LandManager._farmPlots`, the check determining whether a staked Munchable's plot has become invalid compares the 0-indexed `_toiler.plotId` against `_getNumPlots(landlord)` using `<` instead of `<=`. Consequently, a Munchable staked in the boundary plot (`plotId == _getNumPlots(landlord)`) is never marked dirty and continues earning schnibbles indefinitely.

### Technical details

Plot identifiers in `LandManager` are zero-indexed. During plot admission in `stakeMunchable` (line 146) and `transferToUnoccupiedPlot` (line 211), the contract enforces that a valid `plotId` must be strictly less than the total number of available plots:

```solidity
uint256 totalPlotsAvail = _getNumPlots(landlord);
if (plotId >= totalPlotsAvail) revert PlotTooHighError();
```

Thus, when a landlord has `N` plots (`_getNumPlots(landlord) == N`), the valid `plotId` values are `0` through `N - 1`, and any plot with `plotId >= N` is out of bounds and invalid.

However, during reward settlement in `_farmPlots` (lines 258–260), the contract checks whether the plot is out of bounds using a strict inequality (`<`):

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

If a landlord reduces their locked token value such that `_getNumPlots(landlord)` becomes equal to `_toiler.plotId` (for example, a landlord with 1 plot at `plotId = 0` unlocks all funds so `_getNumPlots(landlord)` drops to `0`, or drops from `N` plots to `N - 1` plots while a Munchable occupies plot `N - 1`), the condition `_getNumPlots(landlord) < _toiler.plotId` evaluates to `false`.

As a result:
1. `timestamp` remains `block.timestamp` rather than being capped at `plotMetadata[landlord].lastUpdated`.
2. `toilerState[tokenId].dirty` is never set to `true`.
3. On that call and all subsequent calls to `_farmPlots`, both the renter and the landlord continue to accrue `unfedSchnibbles` from a plot that no longer exists.

### Assumptions and preconditions

1. Both the landlord and the renter are registered players in `AccountManager`.
2. The landlord initially has `N >= 1` available plots (`_getNumPlots(landlord) == N`).
3. A renter stakes a Munchable in plot `plotId = N - 1` (or the landlord later unlocks enough tokens so that `_getNumPlots(landlord)` decreases to `plotId`).

### Proof of concept

1. Alice (landlord) locks tokens worth `1 * PRICE_PER_PLOT`, giving her `_getNumPlots(alice) == 1` (valid plot index: `plotId = 0`).
2. Bob (renter) calls `stakeMunchable(alice, tokenId, 0)`. Since `0 < 1`, the stake succeeds and `toilerState[tokenId].plotId` is `0`.
3. After the lock duration expires, Alice unlocks all of her tokens. Now `_getNumPlots(alice)` returns `0`, and `plotMetadata[alice].lastUpdated` is updated to the unlock timestamp.
4. Time passes well beyond the unlock timestamp.
5. Bob calls `farmPlots()`. In `_farmPlots`, `_getNumPlots(alice) < _toiler.plotId` evaluates `0 < 0`, which is `false`.
6. `toilerState[tokenId].dirty` remains `false` and `timestamp` remains `block.timestamp`. Bob and Alice receive schnibbles for the entire elapsed period despite Alice having 0 plots available, and they can repeat this indefinitely.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** The dirty-flag invalidation mechanism and `lastUpdated` timestamp cap are bypassed for plot `plotId == _getNumPlots(landlord)`, allowing unauthorized minting/accrual of `unfedSchnibbles` for both the renter and the landlord.
- **Business Impact:** Renters and landlords can earn farming rewards without maintaining the required locked capital in `LockManager` (e.g., unlocking 100% of tokens after a renter stakes in `plotId = 0`), inflating schnibble and Munch Point distributions at the expense of honest participants.

### Remediation

Update the boundary condition in `LandManager._farmPlots` to use `<=` so that a plot is invalidated whenever `_getNumPlots(landlord) <= _toiler.plotId`:

```solidity
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated;
                toilerState[tokenId].dirty = true;
            }
```

## Finding 8: Inconsistent Landlord Account Resolution and Registration Validation in `LandManager._getMainAccountRequireRegistered` and `LandManager._getNumPlots` Enables Permanent Staked NFT Lockup and Sub-Account Alias Abuse

### Synopsis

In `LandManager`, `_getNumPlots` queries locked token values on raw addresses without verifying registration or resolving canonical main accounts, while `_getMainAccountRequireRegistered` reverts on unregistered accounts and resolves sub-accounts to their main account. This mismatch lets unregistered or sub-account landlords accept staked NFTs and permanently DoS `unstakeMunchable` or corrupt player state.

### Technical details

In the Munchables architecture, player records in `AccountManager` are keyed by a canonical `mainAccount`, while up to 5 unregistered addresses can be linked as `subAccounts` (`mainAccounts[subAccount] = mainAccount`). Additionally, accounts can lock tokens in `LockManager` prior to registering in `AccountManager` (which is why `AccountManager.forceHarvest` explicitly checks `if (player.registrationDate != 0)` before harvesting or updating plot metadata).

In `src/managers/LandManager.sol`, account identity and registration checks across `_getMainAccountRequireRegistered` (lines 332–342) and `_getNumPlots` (lines 344–345) fail to enforce consistent canonical main-account registration for landlords:

```solidity
function _getMainAccountRequireRegistered(
    address _account
) internal view returns (address, MunchablesCommonLib.Player memory) {
    (
        address _mainAccount,
        MunchablesCommonLib.Player memory _player
    ) = accountManager.getPlayer(_account);

    if (_player.registrationDate == 0) revert PlayerNotRegisteredError();
    return (_mainAccount, _player);
}

function _getNumPlots(address _account) internal view returns (uint256) {
    return lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT;
}
```

This creates two closely coupled flaws in how `LandManager` validates and resolves `landlord` addresses:

1. **Missing Registration Prerequisite in `_getNumPlots` vs. `_getMainAccountRequireRegistered` (Permanent Staking DoS):**
   When a renter calls `stakeMunchable(address landlord, uint256 tokenId, uint256 plotId)`, `LandManager` checks plot availability via `_getNumPlots(landlord)` (line 145 -> lines 344–345), which directly computes `lockManager.getLockedWeightedValue(landlord) / PRICE_PER_PLOT` without invoking `_getMainAccountRequireRegistered(landlord)`.
   - If `landlord` has locked tokens in `LockManager` but has not registered in `AccountManager` (`players[landlord].registrationDate == 0`), `_getNumPlots(landlord)` returns a non-zero plot count and `stakeMunchable` succeeds.
   - Alternatively, an unregistered address `L` with locked tokens can be added as a sub-account of a registered player `M` via `AccountManager.addSubAccount(L)` (which requires `onlyUnregistered(L)`), and `M` can later unlink `L` via `AccountManager.removeSubAccount(L)`.
   - Once a renter's Munchable NFT is staked to an unregistered `landlord`, every subsequent call by the renter to `unstakeMunchable`, `farmPlots`, `stakeMunchable`, or `transferToUnoccupiedPlot` executes `_farmPlots(msg.sender)`.
   - Inside `_farmPlots` (line 265), `LandManager` calls `_getMainAccountRequireRegistered(landlord)` (lines 332–342). Because `_player.registrationDate == 0`, line 340 reverts with `PlayerNotRegisteredError()`. Since `unstakeMunchable` has the `forceFarmPlots(msg.sender)` modifier, the renter's staked Munchable NFTs are permanently frozen in `LandManager`.

2. **Canonical-vs.-Alias Mismatch When Resolving Sub-Accounts in `_getMainAccountRequireRegistered` and `_getNumPlots`:**
   - When a registered player `M` locks tokens under `M` and a caller passes `M`'s sub-account `S` to `_getNumPlots(S)`, `lockManager.getLockedWeightedValue(S)` queries `S` instead of resolving `S` to `M` via `_getMainAccountRequireRegistered(S)`.
   - Conversely, if an unregistered address `S` locks tokens first and is then added as a sub-account of `M` (`mainAccounts[S] = M`), `M` can call `stakeMunchable(S, tokenId, plotId)`. Because `S` is not canonicalized to `M`, `if (landlord == mainAccount) revert CantStakeToSelfError()` is bypassed (`S != M`). Then, during `_farmPlots`, `_getMainAccountRequireRegistered(S)` (lines 335–338) returns `(M, players[M])`, and line 300 calls `accountManager.updatePlayer(S, landlordMetadata)`, writing `M`'s `Player` struct (including `M`'s `unfedSchnibbles` and `registrationDate`) into `players[S]` and duplicating `M`'s schnibbles when `S` is subsequently removed as a sub-account.

### Assumptions and preconditions

- The protocol contracts (`LandManager`, `AccountManager`, `LockManager`, `MunchNFT`) are deployed, initialized, and unpaused.
- An address `L` has locked tokens in `LockManager` sufficient to provide at least one plot (`getLockedWeightedValue(L) >= PRICE_PER_PLOT`), either without registering in `AccountManager` or before being linked/unlinked as a sub-account in `AccountManager`.
- A registered player (renter) owns at least one Munchable NFT and stakes it onto `L`'s plot via `LandManager.stakeMunchable`.

### Proof of concept

#### Scenario 1: Permanent Lockup of Renter's Staked Munchable NFTs
1. Address `L` locks tokens in `LockManager` worth at least `PRICE_PER_PLOT` without calling `AccountManager.register` (or `L` locks tokens while unregistered, is added as a sub-account of a registered player `M` via `AccountManager.addSubAccount(L)`, and `M` removes `L` via `AccountManager.removeSubAccount(L)` after step 2).
2. A registered renter `R` approves `LandManager` for `tokenId` and calls `LandManager.stakeMunchable(L, tokenId, 0)`.
   - `forceFarmPlots(R)` succeeds because `R` has no previously staked tokens.
   - `_getMainAccountRequireRegistered(R)` succeeds because `R` is registered.
   - `_getNumPlots(L)` (lines 344–345) returns `lockManager.getLockedWeightedValue(L) / PRICE_PER_PLOT >= 1` because it does not check whether `L` is registered.
   - `tokenId` is transferred from `R` to `LandManager` and recorded in `munchablesStaked[R]` with `toilerState[tokenId].landlord = L`.
3. `R` later attempts to unstake `tokenId` by calling `LandManager.unstakeMunchable(tokenId)` (or calls `farmPlots()`, `transferToUnoccupiedPlot()`, or `stakeMunchable()` for another NFT).
4. The `forceFarmPlots(msg.sender)` modifier invokes `_farmPlots(R)`, which iterates over `munchablesStaked[R]` and calls `_getMainAccountRequireRegistered(L)` (lines 332–342).
5. Because `L` is unregistered (`_player.registrationDate == 0`), line 340 reverts with `PlayerNotRegisteredError()`. `R` can never unstake `tokenId` (or any other NFTs staked by `R`).

#### Scenario 2: Self-Staking Restriction Bypass and Schnibbles Duplication via Sub-Account Landlord
1. Registered player `M` has `X` `unfedSchnibbles` in `players[M]`.
2. Unregistered address `S` locks tokens in `LockManager` so `_getNumPlots(S) >= 1`.
3. `M` links `S` as a sub-account via `AccountManager.addSubAccount(S)` (`mainAccounts[S] = M`).
4. `M` calls `LandManager.stakeMunchable(S, tokenId, 0)`. Because `landlord` (`S`) is not resolved to its canonical `mainAccount` (`M`), `if (landlord == mainAccount)` (`S == M`) evaluates to `false`, bypassing `CantStakeToSelfError`.
5. `M` calls `LandManager.farmPlots()`. In `_farmPlots`:
   - `renterMetadata` is loaded from `players[M]` (`unfedSchnibbles = X`).
   - `_getMainAccountRequireRegistered(S)` (lines 332–342) resolves `S` to `M` via `accountManager.getPlayer(S)` and returns `players[M]` (`unfedSchnibbles = X`, `registrationDate != 0`) as `landlordMetadata`.
   - `accountManager.updatePlayer(S, landlordMetadata)` (line 300) writes `landlordMetadata` to `players[S]`, giving `S` `X + schnibblesLandlord` schnibbles and a non-zero `registrationDate`.
   - `accountManager.updatePlayer(M, renterMetadata)` (line 309) writes `renterMetadata` to `players[M]`, retaining `X + schnibblesRenter` schnibbles on `M`.
6. `M` calls `AccountManager.removeSubAccount(S)`, leaving both `M` and `S` as independent registered main accounts, each holding `X` schnibbles.

### Impact assessment

- **Severity:** High
- **Technical Impact:**
  1. Any renter who stakes a Munchable NFT to an unregistered landlord (or to a sub-account landlord that is subsequently unlinked by its main account) suffers a permanent denial of service on `_farmPlots`, making it impossible to unstake any of their staked Munchable NFTs from `LandManager`.
  2. A player using a sub-account with locked tokens as a landlord can bypass `CantStakeToSelfError` and overwrite the sub-account's `Player` storage record with the main account's `Player` struct during `_farmPlots`, duplicating `unfedSchnibbles`.
- **Business Impact:** Direct violation of the protocol's core invariant against deadlocks where a user's Munchable NFT becomes permanently stuck in `LandManager`, alongside unauthorized inflation of in-game rewards (`unfedSchnibbles`).

### Remediation

Ensure that `_getNumPlots` and `_getMainAccountRequireRegistered` enforce that any account acting as a landlord or queried for plots is a registered main account (canonical account), and resolve `landlord` to its canonical `mainAccount`:

```solidity
    function _getMainAccountRequireRegistered(
        address _account
    ) internal view returns (address, MunchablesCommonLib.Player memory) {
        (
            address _mainAccount,
            MunchablesCommonLib.Player memory _player
        ) = accountManager.getPlayer(_account);

        if (_player.registrationDate == 0) revert PlayerNotRegisteredError();
        return (_mainAccount, _player);
    }

    function _getNumPlots(address _account) internal view returns (uint256) {
        (address mainAccount, ) = _getMainAccountRequireRegistered(_account);
        if (mainAccount != _account) revert InvalidOwnerError();
        return lockManager.getLockedWeightedValue(mainAccount) / PRICE_PER_PLOT;
    }
```

Additionally, in `stakeMunchable` and `_farmPlots`, canonicalize `landlord` via `(landlord, ) = _getMainAccountRequireRegistered(landlord);` (and use the returned canonical `_mainAccount` when calling `accountManager.updatePlayer`).
