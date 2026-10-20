# Audit report — 2024-07-munchables/H-03

## Finding 1: Mandatory `_farmPlots` Settlement in `LandManager.unstakeMunchable` Causes Permanent Locking of Staked NFTs

### Synopsis

`LandManager.unstakeMunchable` enforces the `forceFarmPlots(msg.sender)` modifier before allowing a renter to withdraw a staked NFT. Because `_farmPlots` iterates over all of the caller's staked positions and reverts under reachable counterparty states (such as underflow of `timestamp - _toiler.lastToilDate` or unregistered landlord accounts), all staked NFTs of the renter can become permanently locked.

### Technical details

In `LandManager.sol`, `unstakeMunchable` is decorated with the `forceFarmPlots(msg.sender)` modifier on line 175:

```solidity
function unstakeMunchable(
    uint256 tokenId
) external override forceFarmPlots(msg.sender) notPaused {
```

Before `unstakeMunchable` can execute its withdrawal logic, `forceFarmPlots(msg.sender)` invokes `_farmPlots(msg.sender)`, which loops through every token in `munchablesStaked[mainAccount]` and performs reward settlement against each token's `toilerState[tokenId]`.

This mandatory, coupled settlement introduces a critical exit liveness failure whenever `_farmPlots` reverts for any single staked token owned by `mainAccount`:

1. **Underflow via `plotMetadata[landlord].lastUpdated`:** When a renter stakes a Munchable in `stakeMunchable`, `toilerState[tokenId].lastToilDate` is set to `block.timestamp` (line 163), while `plotMetadata[landlord].lastUpdated` may be an earlier timestamp (or `0` if `plotMetadata` was never initialized for that landlord). If `_getNumPlots(landlord) < _toiler.plotId` subsequently evaluates to `true` (lines 258–261)—for example, if the landlord's locked weighted value drops due to a token `usdPrice` reduction or `PRICE_PER_PLOT` increase without updating `plotMetadata[landlord].lastUpdated`, or if `plotMetadata[landlord].lastUpdated` predates `_toiler.lastToilDate`—`timestamp` is set to `plotMetadata[landlord].lastUpdated`. The calculation `(timestamp - _toiler.lastToilDate) * BASE_SCHNIBBLE_RATE` on line 281 then underflows and reverts in Solidity 0.8.25.
2. **Unvalidated `landlord` in `stakeMunchable`:** `stakeMunchable` stores `landlord` directly in `toilerState[tokenId].landlord` (line 165) without verifying that `landlord` is a registered main account or that `plotMetadata[landlord].lastUpdated != 0`. If `landlord` is an unregistered account that happens to have locked value (or if `finalBonus` / tax calculation fails), `_getMainAccountRequireRegistered(landlord)` in `_farmPlots` (line 265) reverts with `PlayerNotRegisteredError`.
3. **Cross-Position Coupling:** Because `_farmPlots` iterates over the entire `munchablesStaked[mainAccount]` array, a failure in settling even one staked token causes `forceFarmPlots(msg.sender)` to revert for every call to `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot`, freezing all of the user's staked Munchables in `LandManager` with no fallback withdrawal function.

### Assumptions and preconditions

1. A registered renter stakes one or more Munchable NFTs into `LandManager` via `stakeMunchable`.
2. For at least one of the renter's staked NFTs, the landlord's available plots fall below `_toiler.plotId` while `plotMetadata[landlord].lastUpdated < _toiler.lastToilDate` (e.g., `plotMetadata` was never triggered prior to locking, or the landlord's weighted locked value decreased due to price/configuration updates after the renter staked).

### Proof of concept

1. Landlord `Alice` locks tokens at `t = 100`, setting `plotMetadata[Alice].lastUpdated = 100` and giving `Alice` 100 plots (`plotId` 0 to 99).
2. At `t = 200`, Renter `Bob` stakes `tokenId = 1` on `Alice`'s `plotId = 50` (and `tokenId = 2` on another valid landlord's plot). `toilerState[1].lastToilDate` is recorded as `200`.
3. At `t = 300`, `Alice`'s `_getNumPlots(Alice)` drops to `40` (for example, due to a decrease in the locked token's `usdPrice` or an increase in `PRICE_PER_PLOT`, which does not update `plotMetadata[Alice].lastUpdated`), or `plotMetadata[Alice].lastUpdated` remains `100` (`< 200`).
4. `Bob` calls `unstakeMunchable(1)` (or `unstakeMunchable(2)` to withdraw his other NFT).
5. The `forceFarmPlots(msg.sender)` modifier invokes `_farmPlots(Bob)`.
6. When the loop processes `tokenId = 1`, `_getNumPlots(Alice) < _toiler.plotId` (`40 < 50`) is `true`, so `timestamp` is set to `plotMetadata[Alice].lastUpdated` (`100`).
7. On line 281, `timestamp - _toiler.lastToilDate` evaluates `100 - 200`, which reverts with an arithmetic underflow panic.
8. Both `tokenId = 1` and `tokenId = 2` are permanently trapped in `LandManager` because `Bob` cannot call `unstakeMunchable` or `transferToUnoccupiedPlot` without triggering `_farmPlots(Bob)`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** `unstakeMunchable` (as well as `transferToUnoccupiedPlot` and `stakeMunchable`) reverts unconditionally for the affected renter, causing a permanent denial of service on NFT withdrawals.
- **Business Implications:** Core protocol invariant violation—renters' Munchable NFTs are permanently locked in `LandManager`, preventing them from recovering their assets or participating in the rest of the game.

### Remediation

Decouple unstaking from mandatory global settlement or provide an emergency unstake path, and ensure in `stakeMunchable` and `_farmPlots` that `plotMetadata[landlord].lastUpdated` is initialized and clamped so that `timestamp` cannot be less than `_toiler.lastToilDate`. Additionally, allow users to unstake a specific `tokenId` without forcing settlement across unrelated positions if settlement cannot be completed.

```solidity
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();

        toilerState[tokenId] = ToilerState({
            lastToilDate: block.timestamp,
            plotId: plotId,
            landlord: landlord,
            latestTaxRate: plotMetadata[landlord].currentTaxRate,
            dirty: false
        });

        emit FarmPlotTaken(toilerState[tokenId], tokenId);
    }

    function unstakeMunchable(
        uint256 tokenId
    ) external override notPaused {
        _farmPlots(msg.sender);
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        ToilerState memory _toiler = toilerState[tokenId];
        if (_toiler.landlord == address(0)) revert NotStakedError();
        if (munchableOwner[tokenId] != mainAccount) revert InvalidOwnerError();

        plotOccupied[_toiler.landlord][_toiler.plotId] = Plot({
            occupied: false,
            tokenId: 0
        });
        toilerState[tokenId] = ToilerState({
            lastToilDate: 0,
            plotId: 0,
            landlord: address(0),
            latestTaxRate: 0,
            dirty: false
        });
        munchableOwner[tokenId] = address(0);
        _removeTokenIdFromStakedList(mainAccount, tokenId);

        munchNFT.transferFrom(address(this), mainAccount, tokenId);
        emit FarmPlotLeave(_toiler.landlord, tokenId, _toiler.plotId);
    }
```

In `_farmPlots`, also guard `timestamp` when `_getNumPlots(landlord) <= _toiler.plotId` so that `timestamp = Math.max(plotMetadata[landlord].lastUpdated, _toiler.lastToilDate)` to prevent arithmetic underflow from blocking withdrawals.

## Finding 2: Missing `plotMetadata` Initialization Check in `LandManager.stakeMunchable` Allows Staking With Uninitialized Tax Rate and Causes Underflow in `_farmPlots`

### Synopsis

`LandManager.stakeMunchable` does not verify that `plotMetadata[landlord].lastUpdated != 0` before staking. A renter can stake onto a landlord whose `plotMetadata` has not been initialized, snapshotting a `0` tax rate instead of `DEFAULT_TAX_RATE` and risking arithmetic underflow in `_farmPlots` that blocks `unstakeMunchable`.

### Technical details

In `LandManager`, a player's available plots are dynamically calculated in `_getNumPlots(address _account)` via `lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT`. However, a landlord's `plotMetadata[landlord]` (`lastUpdated` and `currentTaxRate`) is only initialized when `updatePlotMetadata` or `triggerPlotMetadata` is executed. For players who locked tokens prior to `LandManager` deployment (or when `LockManager` did not notify `LandManager`), `_getNumPlots(landlord)` is non-zero while `plotMetadata[landlord].lastUpdated == 0` and `plotMetadata[landlord].currentTaxRate == 0`.

While `updateTaxRate` explicitly checks `if (plotMetadata[landlord].lastUpdated == 0) revert PlotMetadataNotUpdatedError();` (lines 96–97), `stakeMunchable` omits this validation and directly snapshots the landlord's tax rate at line 166:

```solidity
toilerState[tokenId] = ToilerState({
    lastToilDate: block.timestamp,
    plotId: plotId,
    landlord: landlord,
    latestTaxRate: plotMetadata[landlord].currentTaxRate,
    dirty: false
});
```

This creates two interconnected issues:

1. **Zero Tax Rate Snapshot:** `toilerState[tokenId].latestTaxRate` is set to `0` (and continues to be refreshed to `0` in `_farmPlots` at line 292) instead of `DEFAULT_TAX_RATE`, allowing the renter to farm 100% of the generated schnibbles while depriving the landlord of tax rewards.
2. **Settlement Underflow and Staked NFT Deadlock:** In `_farmPlots`, if the landlord's plot count falls below `_toiler.plotId` (line 258: `if (_getNumPlots(landlord) < _toiler.plotId)`), `timestamp` is set to `plotMetadata[landlord].lastUpdated` (line 259). If `plotMetadata[landlord].lastUpdated` is `0` (or less than `_toiler.lastToilDate`), the reward calculation at line 281:
   ```solidity
   schnibblesTotal = (timestamp - _toiler.lastToilDate) * BASE_SCHNIBBLE_RATE;
   ```
   reverts due to arithmetic underflow (`0 - _toiler.lastToilDate`). Because `unstakeMunchable` (line 175), `stakeMunchable` (line 135), and `transferToUnoccupiedPlot` (line 202) all execute the `forceFarmPlots(msg.sender)` modifier, any revert in `_farmPlots` prevents the renter from unstaking any of their Munchables.

### Assumptions and preconditions

1. A landlord has locked tokens in `LockManager` prior to `LandManager` being configured/triggered (as tested in `triggerPlotMetadata.test.ts`), so `_getNumPlots(landlord) > 0` while `plotMetadata[landlord].lastUpdated == 0` and `plotMetadata[landlord].currentTaxRate == 0`.
2. A registered renter owns at least one Munchable NFT and stakes it onto the landlord's plot via `stakeMunchable`.

### Proof of concept

1. Alice locks tokens in `LockManager` prior to `LandManager` notification, giving her `_getNumPlots(alice) = 100`, while `plotMetadata[alice]` remains `{ lastUpdated: 0, currentTaxRate: 0 }`.
2. Bob calls `stakeMunchable(alice, tokenId, 50)`.
3. `stakeMunchable` succeeds and records `toilerState[tokenId]` with `lastToilDate = block.timestamp`, `plotId = 50`, `landlord = alice`, and `latestTaxRate = 0`.
4. When Bob calls `farmPlots()`, `schnibblesLandlord = (schnibblesTotal * 0) / 1e18 = 0`. Bob receives 100% of `schnibblesTotal` and Alice receives `0` instead of `DEFAULT_TAX_RATE`.
5. Furthermore, if `lockManager.getLockedWeightedValue(alice)` decreases (e.g., via token price reconfiguration) such that `_getNumPlots(alice) < 50` while `plotMetadata[alice].lastUpdated == 0`, any call by Bob to `unstakeMunchable(tokenId)` invokes `forceFarmPlots(msg.sender)`, which sets `timestamp = plotMetadata[alice].lastUpdated = 0` and reverts with panic `0x11` (arithmetic underflow) at `timestamp - _toiler.lastToilDate`, locking Bob's staked NFT in `LandManager`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:** Renters can bypass the landlord tax rate by staking into plots of landlords whose `plotMetadata` has not yet been initialized (`currentTaxRate == 0`). Additionally, if the landlord's plot count drops below `plotId` while `plotMetadata[landlord].lastUpdated < _toiler.lastToilDate`, `_farmPlots` reverts with an underflow, causing a denial of service on `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` for the renter.
- **Business Implications:** Landlords lose their expected share of farmed schnibbles (`DEFAULT_TAX_RATE`), and renters risk having their Munchable NFTs frozen in the contract.

### Remediation

Require that the landlord's `plotMetadata` has been initialized (`plotMetadata[landlord].lastUpdated != 0`) before allowing a Munchable to be staked on their land in `stakeMunchable`:

```solidity
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();

        toilerState[tokenId] = ToilerState({
            lastToilDate: block.timestamp,
            plotId: plotId,
            landlord: landlord,
            latestTaxRate: plotMetadata[landlord].currentTaxRate,
            dirty: false
        });
```

## Finding 3: Missing `toilerState[tokenId].plotId` Update in `LandManager.transferToUnoccupiedPlot` Causes `unstakeMunchable` to Corrupt `plotOccupied` State

### Synopsis

`LandManager.transferToUnoccupiedPlot` moves a staked Munchable to a new plot in `plotOccupied` without updating `toilerState[tokenId].plotId`. Consequently, when `unstakeMunchable` later frees `plotOccupied[_toiler.landlord][_toiler.plotId]`, it clears the stale `oldPlotId` instead of the new `plotId`, permanently bricking the new plot and overwriting any new occupant on the old plot.

### Technical details

In `LandManager`, two storage mappings track plot occupancy and staking metadata for a Munchable NFT:
- `plotOccupied[landlord][plotId]` stores a `Plot` struct (`occupied`, `tokenId`).
- `toilerState[tokenId]` stores a `ToilerState` struct (`lastToilDate`, `plotId`, `landlord`, `latestTaxRate`, `dirty`).

When a Munchable is initially staked in `stakeMunchable` (lines 154–168), both mappings are synchronized: `plotOccupied[landlord][plotId]` is marked occupied for `tokenId`, and `toilerState[tokenId].plotId` is set to `plotId`.

When a renter relocates their staked Munchable via `transferToUnoccupiedPlot(uint256 tokenId, uint256 plotId)` (lines 199–226):
1. The contract reads `uint256 oldPlotId = _toiler.plotId;` (line 205).
2. It updates `toilerState[tokenId].latestTaxRate` (lines 213–214).
3. It marks `plotOccupied[_toiler.landlord][oldPlotId]` as unoccupied (lines 215–218) and marks `plotOccupied[_toiler.landlord][plotId]` as occupied by `tokenId` (lines 219–222).

However, `transferToUnoccupiedPlot` never updates `toilerState[tokenId].plotId = plotId`. The `toilerState[tokenId].plotId` field retains the stale `oldPlotId`.

This cross-mapping state desynchronization directly breaks `unstakeMunchable(uint256 tokenId)` (lines 173–197):
- Line 177 loads `ToilerState memory _toiler = toilerState[tokenId];`, where `_toiler.plotId` is still `oldPlotId`.
- Lines 181–184 clear `plotOccupied[_toiler.landlord][_toiler.plotId]` (`oldPlotId`) instead of the actual `plotId` that `tokenId` currently occupies:
  ```solidity
  plotOccupied[_toiler.landlord][_toiler.plotId] = Plot({
      occupied: false,
      tokenId: 0
  });
  ```
- Furthermore, during `_farmPlots` (called via the `forceFarmPlots` modifier), line 258 checks `if (_getNumPlots(landlord) < _toiler.plotId)` against the stale `oldPlotId` rather than the new `plotId`, and line 196 in `unstakeMunchable` emits `FarmPlotLeave` with the wrong plot ID.

### Assumptions and preconditions

- A landlord has locked tokens in `LockManager` providing at least two available plots (`_getNumPlots(landlord) >= 2`).
- A registered renter stakes a Munchable on one of the landlord's plots (`oldPlotId`) and subsequently transfers it to another unoccupied plot (`plotId`) via `transferToUnoccupiedPlot`.

### Proof of concept

1. Landlord Alice locks funds sufficient for at least 2 plots (Plot `0` and Plot `1`).
2. Renter Bob calls `stakeMunchable(alice, tokenIdA, 0)`.
   - `plotOccupied[alice][0]` is `{occupied: true, tokenId: tokenIdA}`.
   - `toilerState[tokenIdA].plotId` is `0`.
3. Renter Bob calls `transferToUnoccupiedPlot(tokenIdA, 1)` to move `tokenIdA` to Plot `1`.
   - `plotOccupied[alice][0]` becomes `{occupied: false, tokenId: 0}`.
   - `plotOccupied[alice][1]` becomes `{occupied: true, tokenId: tokenIdA}`.
   - `toilerState[tokenIdA].plotId` remains `0` (not updated).
4. Renter Jirard calls `stakeMunchable(alice, tokenIdB, 0)` to stake on the now-unoccupied Plot `0`.
   - `plotOccupied[alice][0]` becomes `{occupied: true, tokenId: tokenIdB}`.
5. Renter Bob calls `unstakeMunchable(tokenIdA)`.
   - `unstakeMunchable` reads `_toiler.plotId == 0` on line 177 and resets `plotOccupied[alice][0]` to `{occupied: false, tokenId: 0}` on lines 181–184.
   - Plot `1` (`plotOccupied[alice][1]`) remains `{occupied: true, tokenId: tokenIdA}` forever even though `tokenIdA` has been unstaked and returned to Bob, permanently bricking Plot `1`.
   - Plot `0` (`plotOccupied[alice][0]`) is erroneously marked unoccupied while Jirard's `tokenIdB` is still staked there, allowing another renter to stake on Plot `0` and overwrite Jirard's plot occupancy.

### Impact assessment

- **Severity**: High / Medium
- **Technical Impact**:
  1. **Permanent Plot Deadlock**: Every time a Munchable is moved via `transferToUnoccupiedPlot` and later unstaked via `unstakeMunchable`, the destination plot (`plotId`) remains permanently marked `occupied: true` with no staked token able to clear it, permanently reducing the landlord's usable plot capacity.
  2. **Cross-Renter State Corruption**: If another renter stakes on `oldPlotId` after the first renter transfers out, the first renter's subsequent call to `unstakeMunchable` wipes `plotOccupied[landlord][oldPlotId]`, allowing multiple Munchables to occupy the same plot ID and corrupting plot accounting.
  3. **Incorrect Farming Validation**: If a renter transfers from a high plot ID (which became invalid after a landlord's partial unlock) to a valid low plot ID, `_farmPlots` (line 258) still checks `_getNumPlots(landlord) < _toiler.plotId` using the stale high `oldPlotId`, prematurely marking the Munchable `dirty = true` and halting its Schnibbles farming rewards.

### Remediation

Update `toilerState[tokenId].plotId` to the new `plotId` inside `transferToUnoccupiedPlot` before emitting `FarmPlotTaken`.

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

## Finding 4: Arithmetic Underflow in `LandManager._farmPlots` When `plotMetadata.lastUpdated` Precedes `_toiler.lastToilDate` Permanently Locks Staked Munchables

### Synopsis

In `LandManager._farmPlots`, when a landlord's available plot count falls below a staked Munchable's `plotId`, the accrual end `timestamp` is set to `plotMetadata[landlord].lastUpdated`. If `plotMetadata[landlord].lastUpdated` is earlier than `_toiler.lastToilDate`, computing `timestamp - _toiler.lastToilDate` underflows and reverts, permanently freezing all of the renter's staked Munchables.

### Technical details

In `LandManager._farmPlots`, lines 258–261 check whether the landlord still has enough plots to accommodate the staked Munchable at `_toiler.plotId`:

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

Subsequently, at lines 280–282, `_farmPlots` calculates the base Schnibbles accrued since `_toiler.lastToilDate`:

```solidity
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
```

This subtraction assumes that `timestamp >= _toiler.lastToilDate` always holds (monotonic accrual boundaries). However, `plotMetadata[landlord].lastUpdated` can be strictly less than `_toiler.lastToilDate` in several realistic scenarios:

1. **Token USD price decrease or token deactivation in `LockManager`:** `_getNumPlots(landlord)` dynamically queries `lockManager.getLockedWeightedValue(landlord) / PRICE_PER_PLOT`, which depends on `configuredTokens[token].usdPrice` and `configuredTokens[token].active`, as well as `PRICE_PER_PLOT` in `LandManager`. A landlord can lock tokens at $t_1$ (setting `plotMetadata[landlord].lastUpdated = t_1`), a renter can stake or farm at $t_2 > t_1$ (setting `_toiler.lastToilDate = t_2`), and a subsequent reduction in token `usdPrice`, token deactivation, or increase in `PRICE_PER_PLOT` at $t_3 > t_2$ can reduce `_getNumPlots(landlord)` below `_toiler.plotId` without updating `plotMetadata[landlord].lastUpdated` (which remains $t_1 < t_2$).
2. **`updatePlotMetadata` not invoked on unlock:** `LandManager.updatePlotMetadata` is restricted by `onlyConfiguredContract(StorageKey.AccountManager)` rather than `LockManager`, so `plotMetadata[landlord].lastUpdated` is not updated when a landlord unlocks tokens after a renter has staked or farmed at a later timestamp.

When `plotMetadata[landlord].lastUpdated < _toiler.lastToilDate`, the subtraction `timestamp - _toiler.lastToilDate` at line 281 reverts due to arithmetic underflow in Solidity 0.8.25.

Because `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` all execute the `forceFarmPlots(msg.sender)` modifier—which loops over every token in `munchablesStaked[mainAccount]` and runs `_farmPlots`—any revert in `_farmPlots` for a single staked token blocks the renter from calling `unstakeMunchable`, `transferToUnoccupiedPlot`, `stakeMunchable`, or `farmPlots` for **all** of their staked Munchables.

### Assumptions and preconditions

1. A landlord has locked tokens and initialized `plotMetadata[landlord].lastUpdated` at timestamp $t_1$.
2. A renter stakes a Munchable on the landlord's `plotId` (or farms their existing staked Munchable) at timestamp $t_2 > t_1$, setting `toilerState[tokenId].lastToilDate = t_2`.
3. `_getNumPlots(landlord)` subsequently drops below `_toiler.plotId` while `plotMetadata[landlord].lastUpdated` remains $t_1 < t_2$ (e.g., due to a token `usdPrice` reduction or `PRICE_PER_PLOT` update, or an unlock that does not update `plotMetadata[landlord].lastUpdated`).

### Proof of concept

1. At $t = 1000$, Alice (landlord) locks ETH in `LockManager`, which sets `plotMetadata[alice].lastUpdated = 1000` and gives Alice 100 available plots (`plotId` 0 to 99).
2. At $t = 2000$, Bob (renter) calls `stakeMunchable(alice, tokenId, 80)`. `toilerState[tokenId].lastToilDate` is recorded as `2000` and `toilerState[tokenId].plotId` is `80`.
3. At $t = 3000$, the USD price of ETH in `LockManager` decreases by 30% (or `PRICE_PER_PLOT` is increased), reducing `_getNumPlots(alice)` from `100` to `70`. `plotMetadata[alice].lastUpdated` remains `1000`.
4. At $t = 4000$, Bob attempts to call `unstakeMunchable(tokenId)` (or `transferToUnoccupiedPlot` or `farmPlots`).
5. The `forceFarmPlots(msg.sender)` modifier invokes `_farmPlots(bob)`.
6. For `tokenId`, `_getNumPlots(alice) < _toiler.plotId` (`70 < 80`) evaluates to `true`, so `timestamp` is set to `plotMetadata[alice].lastUpdated = 1000`.
7. At line 281, `timestamp - _toiler.lastToilDate` evaluates `1000 - 2000`, which reverts with an arithmetic underflow panic (`0x11`).
8. Bob cannot unstake `tokenId` or any other Munchable he has staked in `LandManager`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** `_farmPlots` reverts with an arithmetic underflow whenever `plotMetadata[landlord].lastUpdated < _toiler.lastToilDate` and `_getNumPlots(landlord) < _toiler.plotId`.
- **Business / Protocol Impact:** Because `unstakeMunchable`, `transferToUnoccupiedPlot`, and `stakeMunchable` all require `_farmPlots` to succeed via the `forceFarmPlots` modifier, all Munchable NFTs staked by the affected renter are permanently locked in `LandManager` and all unclaimed Schnibbles cannot be harvested.

### Remediation

Ensure that the effective accrual timestamp in `_farmPlots` is never earlier than `_toiler.lastToilDate` (for example, by clamping `timestamp` to `max(plotMetadata[landlord].lastUpdated, _toiler.lastToilDate)` when a plot becomes invalid, or skipping positive reward accrual when `timestamp <= _toiler.lastToilDate`), and ensure `plotMetadata[landlord].lastUpdated` is updated whenever plot availability changes.

```solidity
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated > _toiler.lastToilDate
                    ? plotMetadata[landlord].lastUpdated
                    : _toiler.lastToilDate;
                toilerState[tokenId].dirty = true;
            }
            (
                ,
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
```

## Finding 5: Off-By-One Plot Eligibility Check in `LandManager._farmPlots` Allows Indefinite Schnibble Accrual on Invalidated Boundary Plots

### Synopsis

In `LandManager._farmPlots`, the plot eligibility cutoff check uses strict inequality (`_getNumPlots(landlord) < _toiler.plotId`) instead of `<=`. Because `plotId` is zero-indexed, a Munchable staked in plot `_getNumPlots(landlord)` bypasses the eligibility cutoff timestamp and `dirty` flag, allowing indefinite reward accrual on an unbacked plot.

### Technical details

In `LandManager`, land plots are zero-indexed from `0` to `_getNumPlots(landlord) - 1`. Both `stakeMunchable` (line 146) and `transferToUnoccupiedPlot` (line 211) enforce this upper bound by reverting when `plotId >= totalPlotsAvail`:

```solidity
uint256 totalPlotsAvail = _getNumPlots(landlord);
if (plotId >= totalPlotsAvail) revert PlotTooHighError();
```

When a landlord unlocks tokens (or the locked token USD value decreases) such that `_getNumPlots(landlord)` drops, any staked Munchable occupying a `plotId` that is now greater than or equal to `_getNumPlots(landlord)` is no longer backed by a valid plot. In `_farmPlots`, the contract attempts to stop reward accrual for invalidated plots by capping `timestamp` at `plotMetadata[landlord].lastUpdated` and setting `toilerState[tokenId].dirty = true` (lines 258–261):

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

However, line 258 uses the strict inequality `<` rather than `<=`. If a landlord's available plot count decreases to exactly `_toiler.plotId` (for example, the landlord initially had `N` plots with indices `0` through `N - 1`, a renter staked in the last plot `plotId = N - 1`, and the landlord subsequently unlocked 1 plot's worth of tokens so `_getNumPlots(landlord) == N - 1`), the check `(N - 1) < (N - 1)` evaluates to `false`.

Consequently:
1. `timestamp` is not capped to the eligibility cutoff `plotMetadata[landlord].lastUpdated` and instead remains `block.timestamp` (line 248).
2. `toilerState[tokenId].dirty` remains `false`.
3. At lines 280–282:
   ```solidity
   schnibblesTotal =
       (timestamp - _toiler.lastToilDate) *
       BASE_SCHNIBBLE_RATE;
   ```
   `schnibblesTotal` is computed up to `block.timestamp` rather than the cutoff timestamp, and `toilerState[tokenId].lastToilDate` is updated to `block.timestamp` (line 291). Because `dirty` is never set to `true`, the renter and landlord continue to farm unbacked schnibbles on every subsequent `_farmPlots` call indefinitely.

### Assumptions and preconditions

1. A landlord has `N >= 1` available plots (`_getNumPlots(landlord) == N`), and a renter stakes a Munchable into the boundary plot `plotId = N - 1` (or any plot `plotId = K` where the landlord later reduces their available plots to `K`).
2. The landlord's available plot count `_getNumPlots(landlord)` decreases to `plotId` (e.g., `N - 1`) due to a partial token unlock or a token price reduction.

### Proof of concept

1. Alice (landlord) locks tokens sufficient to provide `_getNumPlots(alice) == 10` plots (valid `plotId` values are `0` through `9`).
2. Bob (renter) calls `stakeMunchable(alice, tokenId, 9)` at timestamp $t_0$. Since `plotId = 9 < 10`, staking succeeds and `toilerState[tokenId].plotId` is set to `9`.
3. At timestamp $t_1 > t_0$, Alice unlocks enough tokens to reduce `_getNumPlots(alice)` from `10` to `9` (so only plots `0` through `8` remain valid, and plot `9` is now unbacked). `plotMetadata[alice].lastUpdated` is updated to $t_1$.
4. At timestamp $t_2 \gg t_1$, Bob calls `farmPlots()`.
5. Inside `_farmPlots(bob)`, `_getNumPlots(alice) < _toiler.plotId` evaluates `9 < 9`, which is `false`.
6. `timestamp` remains `block.timestamp` ($t_2$) instead of being capped at $t_1$, and `toilerState[tokenId].dirty` remains `false`.
7. At lines 280–282, `schnibblesTotal = (t_2 - t_0) * BASE_SCHNIBBLE_RATE` is calculated over the entire period $[t_0, t_2]$, and Bob and Alice continue accruing schnibbles for plot `9` on all future `farmPlots()` calls despite Alice only having 9 plots (`0`–`8`).

### Impact assessment

**Severity: Medium**

- **Direct Technical Effect**: Staked Munchables occupying plot index `_getNumPlots(landlord)` bypass plot invalidation in `_farmPlots`, failing to cap reward accrual at `plotMetadata[landlord].lastUpdated` and failing to set `dirty = true`.
- **Business Implications**: Renters and landlords can mint unbacked schnibbles (and downstream Chonks / Munch Points) indefinitely for one extra plot per landlord without maintaining the required locked token value for that plot.

### Remediation

Update the plot count comparison in `LandManager._farmPlots` (line 258) from `<` to `<=` so that any zero-indexed `_toiler.plotId` greater than or equal to `_getNumPlots(landlord)` triggers the eligibility cutoff and sets `dirty = true`:

```solidity
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated;
                toilerState[tokenId].dirty = true;
            }
```

## Finding 6: Misplaced Parentheses in `LandManager._farmPlots` Bonus Calculation Causes Incorrect Rewards and Negative Cast Underflow

### Synopsis

In `LandManager._farmPlots`, misplaced parentheses in the `schnibblesTotal` bonus calculation divide the base `schnibblesTotal` by 100 alongside the bonus term. This reduces rewards for non-negative bonuses and produces a negative `int256` when `finalBonus < -1`, which wraps to near `type(uint256).max` upon `uint256` casting.

### Technical details

In `LandManager._farmPlots` (lines 270–286), `finalBonus` is computed as the sum of the percentage bonus from `REALM_BONUSES` and `RARITY_BONUSES`, where `1 = 1%`. As documented in `CONSTANTS.md`, `REALM_BONUSES` can be negative when the staked Munchable's realm does not match the landlord's `snuggeryRealm`.

After computing the base `schnibblesTotal = (timestamp - _toiler.lastToilDate) * BASE_SCHNIBBLE_RATE`, the contract applies `finalBonus` on lines 283–286:

```solidity
schnibblesTotal = uint256(
    (int256(schnibblesTotal) +
        (int256(schnibblesTotal) * finalBonus)) / 100
);
```

Instead of dividing only the bonus term `(int256(schnibblesTotal) * finalBonus)` by `100` and adding it to `int256(schnibblesTotal)`, the outer parentheses group `int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)` together before dividing by `100`. Algebraically, this computes:

$$\text{schnibblesTotal} = \frac{\text{schnibblesTotal} \times (1 + \text{finalBonus})}{100}$$

instead of the intended:

$$\text{schnibblesTotal} = \text{schnibblesTotal} + \frac{\text{schnibblesTotal} \times \text{finalBonus}}{100} = \frac{\text{schnibblesTotal} \times (100 + \text{finalBonus})}{100}$$

This flawed calculation causes two severe issues:
1. **Drastic underpayment for non-negative bonuses:** When `finalBonus >= 0`, the base `schnibblesTotal` is divided by `100`. For example, if `finalBonus == 0`, the user receives only `1%` (`schnibblesTotal / 100`) of the intended base schnibbles; if `finalBonus == 10` (+10%), the user receives `11%` (`11 * schnibblesTotal / 100`) instead of `110%`.
2. **Two's complement wrap-around and DoS / infinite mint when `finalBonus < -1`:** When `finalBonus` is negative (e.g., `-5` or `-10` due to an unfavorable realm pairing and low rarity bonus), `int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)` is negative whenever `timestamp > _toiler.lastToilDate`. Dividing by `100` preserves the negative sign, and casting the negative `int256` to `uint256` wraps the value in two's complement to an astronomical number near `type(uint256).max`.
   - When `_toiler.latestTaxRate > 1`, the subsequent multiplication `schnibblesTotal * _toiler.latestTaxRate` on line 288 reverts with an arithmetic overflow panic. Because `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` all apply the `forceFarmPlots(msg.sender)` modifier, any revert in `_farmPlots` permanently freezes the renter's staked Munchable NFTs inside `LandManager`.
   - If `_toiler.latestTaxRate == 0` (e.g., when a landlord's `plotMetadata` has not yet been initialized), the multiplication on line 288 does not overflow, crediting the renter with near-`type(uint256).max` `unfedSchnibbles`.

### Assumptions and preconditions

1. A registered renter stakes a Munchable NFT on a registered landlord's plot via `LandManager.stakeMunchable`.
2. At least 1 second passes (`block.timestamp > _toiler.lastToilDate`) so that base `schnibblesTotal > 0`.
3. For the underflow/revert scenario, the staked Munchable's realm and rarity relative to the landlord's `snuggeryRealm` yield `finalBonus < -1` (e.g., `REALM_BONUSES` is `-10` or `-5` and `RARITY_BONUSES` is `0`).

### Proof of concept

1. Alice (landlord) registers with `snuggeryRealm = 0` and locks tokens to create available plots with `DEFAULT_TAX_RATE` (e.g., `25e16`).
2. Bob (renter) owns a Common Munchable (`rarity = 0`, `RARITY_BONUSES[0] = 0`) whose `realm` gives a `-10` (i.e., `-10%`) realm bonus against `snuggeryRealm = 0`, resulting in `finalBonus = -10`.
3. Bob stakes this Munchable into Alice's plot 0 via `LandManager.stakeMunchable(alice, tokenId, 0)`.
4. After 100 seconds pass (`timestamp - _toiler.lastToilDate = 100`), Bob calls `unstakeMunchable(tokenId)` or `farmPlots()`.
5. Inside `_farmPlots`:
   - Base `schnibblesTotal = 100 * BASE_SCHNIBBLE_RATE` (e.g., `100 * 1e15 = 1e17`).
   - Line 284–285 computes `(1e17 + (1e17 * -10)) / 100 = -9e17 / 100 = -9e15`.
   - Line 283 casts `-9e15` to `uint256`, yielding `type(uint256).max - 9e15 + 1`.
   - Line 288 computes `(schnibblesTotal * 25e16) / 1e18`, which overflows `uint256` and reverts.
6. Bob's call to `unstakeMunchable(tokenId)` reverts, permanently locking Bob's Munchable NFT inside `LandManager`.

### Impact assessment

- **Severity:** High / Critical
- **Direct Technical Effects:**
  - Every staked Munchable with `finalBonus < -1` causes `_farmPlots` to revert once time elapses, permanently locking all of the renter's staked Munchable NFTs in `LandManager` (since `unstakeMunchable` calls `_farmPlots` via `forceFarmPlots`).
  - If `_toiler.latestTaxRate == 0` when `finalBonus < -1`, the renter receives near-`type(uint256).max` `unfedSchnibbles`.
  - For all other staked Munchables (`finalBonus >= 0`), base farming rewards are erroneously divided by 100, reducing base rewards by 99%.

### Remediation

Fix the parenthesization in `LandManager._farmPlots` so that only `int256(schnibblesTotal) * finalBonus` is divided by `100` before being added to `int256(schnibblesTotal)`:

```solidity
            schnibblesTotal = uint256(
                int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus) / 100
            );
```
