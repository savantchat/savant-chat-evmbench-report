# Audit report — 2024-07-munchables/H-02

## Finding 1: Missing `plotMetadata` Initialization Check in `LandManager.stakeMunchable` Allows Zero-Tax Farming and Underflow DoS

### Synopsis

`LandManager.stakeMunchable` fails to verify that `plotMetadata[landlord].lastUpdated != 0` before allowing a renter to stake on a landlord's plot. If a landlord locked tokens prior to `LandManager` deployment and has not yet called `triggerPlotMetadata()`, renters can stake with a `0%` tax rate and risk permanent staking DoS via timestamp underflow in `_farmPlots`.

### Technical details

In `LandManager`, plot availability (`_getNumPlots(landlord)`) is calculated dynamically from `lockManager.getLockedWeightedValue(landlord) / PRICE_PER_PLOT`, whereas per-landlord tax rate and plot update timestamps are stored separately in `plotMetadata[landlord]`.

For landlords who locked tokens in `LockManager` before `LandManager` was deployed (or before `AccountManager` was configured to notify `LandManager`), `_getNumPlots(landlord)` is non-zero, but `plotMetadata[landlord]` remains uninitialized (`lastUpdated == 0` and `currentTaxRate == 0`) until the landlord explicitly calls `triggerPlotMetadata()` or performs a new lock action that invokes `updatePlotMetadata(landlord)`.

While `updateTaxRate` explicitly guards against uninitialized metadata on lines 96–97:
```solidity
if (plotMetadata[landlord].lastUpdated == 0)
    revert PlotMetadataNotUpdatedError();
```
`stakeMunchable` (lines 131–144) performs no such check and does not initialize `plotMetadata[landlord]`. Consequently, staking on a plot belonging to a landlord whose `plotMetadata[landlord].lastUpdated == 0` causes two issues:

1. **Zero-Tax Farming (Loss of Landlord Yield):** On line 166, `toilerState[tokenId].latestTaxRate` is set to `plotMetadata[landlord].currentTaxRate`, which is `0` instead of `DEFAULT_TAX_RATE` (and below `MIN_TAX_RATE`). When `_farmPlots` is later executed, `schnibblesLandlord` evaluates to `0` (lines 287–289), allowing the renter to keep 100% of farmed Schnibbles and depriving the landlord of their tax share.
2. **Underflow DoS in `_farmPlots`:** If the landlord's plot count (`_getNumPlots(landlord)`) drops below `_toiler.plotId` while `plotMetadata[landlord].lastUpdated == 0` (e.g., due to an increase in `PRICE_PER_PLOT` or a decrease in locked token `usdPrice` in `LockManager`), `_farmPlots` sets `timestamp = plotMetadata[landlord].lastUpdated` (`0`) on line 259. Line 281 then computes `(timestamp - _toiler.lastToilDate) * BASE_SCHNIBBLE_RATE`, which reverts due to arithmetic underflow (`0 - _toiler.lastToilDate`). Because `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` all apply the `forceFarmPlots(msg.sender)` modifier, all staking operations for the renter revert until `plotMetadata[landlord]` is initialized.

### Assumptions and preconditions

1. A registered landlord has locked tokens in `LockManager` prior to `LandManager` being deployed/configured (so `_getNumPlots(landlord) > 0`), and has not yet called `triggerPlotMetadata()` or locked additional tokens.
2. A registered renter owns a Munchable NFT and calls `stakeMunchable` targeting one of the landlord's plots.

### Proof of concept

1. Alice (landlord) registers and locks ETH in `LockManager` before `LandManager` is deployed, giving her `totalPlotsAvail = 10` once `LandManager` is deployed. At this point, `plotMetadata[alice].lastUpdated == 0` and `plotMetadata[alice].currentTaxRate == 0`.
2. Bob (renter) approves his Munchable (`tokenId = 1`) and calls `stakeMunchable(alice, 1, 5)`.
3. `stakeMunchable` succeeds because `_getNumPlots(alice)` is `10 > 5`, and sets `toilerState[1].latestTaxRate = plotMetadata[alice].currentTaxRate = 0` and `toilerState[1].lastToilDate = block.timestamp`.
4. After time passes, Bob calls `farmPlots()`. In `_farmPlots()`, `schnibblesLandlord = (schnibblesTotal * 0) / 1e18 = 0`. Bob receives 100% of `schnibblesTotal` while Alice receives `0` Schnibbles instead of `DEFAULT_TAX_RATE`.
5. Furthermore, if `PRICE_PER_PLOT` is increased or the locked token's `usdPrice` is reduced such that `_getNumPlots(alice) < 5` before Alice initializes `plotMetadata[alice]`, Bob's calls to `unstakeMunchable(1)` or `farmPlots()` will execute `timestamp = plotMetadata[alice].lastUpdated = 0` and revert with panic `0x11` (arithmetic underflow) at `timestamp - _toiler.lastToilDate`.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effects:** Renters can bypass the `DEFAULT_TAX_RATE` (and `MIN_TAX_RATE`) by staking on plots of landlords who have not yet triggered `plotMetadata`, and their staked NFTs can become temporarily frozen if `_getNumPlots(landlord)` falls below `plotId` while `plotMetadata[landlord].lastUpdated == 0`.
- **Business Implications:** Landlords lose earned Schnibbles yield from renters, and renters face denial of service on `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot`.

### Remediation

In `LandManager.stakeMunchable`, revert with `PlotMetadataNotUpdatedError()` if `plotMetadata[landlord].lastUpdated == 0` (consistent with `updateTaxRate`), or initialize `plotMetadata[landlord]` to `DEFAULT_TAX_RATE` and `block.timestamp` if uninitialized.

```solidity
    function stakeMunchable(
        address landlord,
        uint256 tokenId,
        uint256 plotId
    ) external override forceFarmPlots(msg.sender) notPaused {
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        if (landlord == mainAccount) revert CantStakeToSelfError();
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();
        if (plotOccupied[landlord][plotId].occupied)
            revert OccupiedPlotError(landlord, plotId);
        if (munchablesStaked[mainAccount].length > 10)
            revert TooManyStakedMunchiesError();
        if (munchNFT.ownerOf(tokenId) != mainAccount)
            revert InvalidOwnerError();
```

## Finding 2: Missing Main-Account Canonicalization of `landlord` in `LandManager.stakeMunchable` Allows Self-Staking Bypass and Attribution Mismatches

### Synopsis

In `LandManager.stakeMunchable`, `msg.sender` is resolved to its canonical `mainAccount` via `_getMainAccountRequireRegistered`, but the `landlord` parameter is never validated or canonicalized to its main account. This allows bypassing `CantStakeToSelfError` via a sub-account, staking to unregistered landlords (locking NFTs), and corrupting player state during farming.

### Technical details

The Munchables protocol supports main accounts and delegated sub-accounts managed through `AccountManager.getPlayer(address _account)`, which resolves any sub-account to its canonical `(_mainAccount, _player)` record.

In `LandManager.stakeMunchable` (`src/managers/LandManager.sol:131-144`), the caller is canonicalized to `mainAccount`, and the self-staking check compares the raw `landlord` parameter against `mainAccount`:

```solidity
(address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
if (landlord == mainAccount) revert CantStakeToSelfError();
if (plotOccupied[landlord][plotId].occupied)
    revert OccupiedPlotError(landlord, plotId);
```

Because `_getMainAccountRequireRegistered(landlord)` is never called in `stakeMunchable` to validate and canonicalize `landlord` to its main account:

1. **Self-Staking Restriction Bypass & Schnibbles Duplication**: A player (`mainAccount`) can register a sub-account (`subAccount`) and pass `landlord = subAccount` to `stakeMunchable`. Since `subAccount != mainAccount`, `if (landlord == mainAccount) revert CantStakeToSelfError()` is bypassed. Later, when `_farmPlots` executes (`src/managers/LandManager.sol:262-300`), `_getMainAccountRequireRegistered(landlord)` resolves `subAccount` to `mainAccount`'s `Player` struct (`landlordMetadata`), adds `schnibblesLandlord`, and then calls `accountManager.updatePlayer(landlord, landlordMetadata)` using the uncanonicalized `subAccount` address. This writes a full copy of `mainAccount`'s `Player` struct (including all previously accumulated `unfedSchnibbles`) into `players[subAccount]`, effectively duplicating the player's `unfedSchnibbles` and promoting `subAccount` to a standalone main account.
2. **Staking to Unregistered Landlords / Sub-Accounts**: If a renter stakes a Munchable to a `landlord` address that has locked tokens in `LockManager` (so `_getNumPlots(landlord) > 0`) but is not registered in `AccountManager`, `stakeMunchable` succeeds because it skips `_getMainAccountRequireRegistered(landlord)`. Subsequently, any call to `farmPlots`, `unstakeMunchable`, `transferToUnoccupiedPlot`, or `stakeMunchable` triggers `_farmPlots`, which calls `_getMainAccountRequireRegistered(landlord)` on line 265 and reverts with `PlayerNotRegisteredError()`, trapping the renter's staked Munchable NFT. Similarly, if a renter passes a landlord's sub-account instead of their main account, `_farmPlots` fetches the landlord's main account metadata via `_getMainAccountRequireRegistered(landlord)` on line 265 but writes the updated metadata to `players[subAccount]` on line 300 instead of the landlord's main account.

### Assumptions and preconditions

- The protocol is deployed and unpaused (`_paused == false`).
- For the self-staking / state duplication vector: a registered player adds a sub-account and has at least one available plot on the `landlord` address (`_getNumPlots(landlord) > plotId`).
- For the unregistered landlord NFT lockup vector: an unregistered address has locked tokens in `LockManager` such that `_getNumPlots(landlord) > 0`.

### Proof of concept

#### Scenario 1: Bypassing `CantStakeToSelfError` via Sub-Account
1. Alice registers as a player (`mainAccount = Alice`) and registers `AliceSub` as her sub-account in `AccountManager`.
2. `AliceSub` locks tokens in `LockManager` so that `_getNumPlots(AliceSub) >= 1`.
3. Alice calls `LandManager.stakeMunchable(AliceSub, tokenId, 0)`.
4. In `stakeMunchable`, `_getMainAccountRequireRegistered(msg.sender)` returns `mainAccount = Alice`.
5. The check `if (landlord == mainAccount)` evaluates `AliceSub == Alice` (`false`), bypassing `CantStakeToSelfError`.
6. When `farmPlots()` is subsequently called by Alice:
   - `_getMainAccountRequireRegistered(Alice)` loads Alice's `Player` struct into `renterMetadata`.
   - `_getMainAccountRequireRegistered(AliceSub)` resolves `AliceSub` to Alice's main account and loads Alice's `Player` struct into `landlordMetadata`.
   - `accountManager.updatePlayer(AliceSub, landlordMetadata)` writes Alice's entire `Player` struct (including all of Alice's preexisting `unfedSchnibbles` plus `schnibblesLandlord`) to `players[AliceSub]`.
   - `accountManager.updatePlayer(Alice, renterMetadata)` updates `players[Alice]` with `renterMetadata`. Alice has both farmed her own land and duplicated her `unfedSchnibbles` balance into `AliceSub`.

#### Scenario 2: Staking to an Unregistered Landlord Locks the Renter's NFT
1. Bob locks tokens in `LockManager` (yielding `_getNumPlots(Bob) >= 1`) before registering in `AccountManager`.
2. Renter Jirard calls `LandManager.stakeMunchable(Bob, tokenId, 0)`. Because `stakeMunchable` does not call `_getMainAccountRequireRegistered(Bob)`, the transaction succeeds and transfers Jirard's `tokenId` to `LandManager`.
3. Jirard attempts to call `unstakeMunchable(tokenId)`. The `forceFarmPlots(msg.sender)` modifier runs `_farmPlots(Jirard)`, which calls `_getMainAccountRequireRegistered(Bob)` at line 265 and reverts with `PlayerNotRegisteredError()`. Jirard's NFT is locked until Bob registers.

### Impact assessment

- **Severity**: High
- **Technical Impact**:
  1. Bypass of the self-staking prohibition (`CantStakeToSelfError`).
  2. Duplication of `unfedSchnibbles` and corruption of sub-account/main-account state in `AccountManager` when `landlord` is a sub-account.
  3. Denial of service (frozen Munchable NFTs and inability to call `farmPlots`, `stakeMunchable`, `unstakeMunchable`, or `transferToUnoccupiedPlot`) if a renter stakes to an unregistered address that has locked tokens.

### Remediation

Canonicalize `landlord` to its main account using `_getMainAccountRequireRegistered(landlord)` in `stakeMunchable` before checking `CantStakeToSelfError` and recording plot/toiler state:

```solidity
    function stakeMunchable(
        address landlord,
        uint256 tokenId,
        uint256 plotId
    ) external override forceFarmPlots(msg.sender) notPaused {
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        (landlord, ) = _getMainAccountRequireRegistered(landlord);
        if (landlord == mainAccount) revert CantStakeToSelfError();
        if (plotOccupied[landlord][plotId].occupied)
            revert OccupiedPlotError(landlord, plotId);
        if (munchablesStaked[mainAccount].length > 10)
            revert TooManyStakedMunchiesError();
        if (munchNFT.ownerOf(tokenId) != mainAccount)
            revert InvalidOwnerError();
```

## Finding 3: Underflow in `LandManager._farmPlots` When `plotMetadata[landlord].lastUpdated` Precedes `lastToilDate` Permanently Locks Staked Munchables

### Synopsis

`LandManager.stakeMunchable` sets `toilerState[tokenId].lastToilDate` to `block.timestamp` without ensuring `plotMetadata[landlord].lastUpdated` is initialized or kept synchronized when `_getNumPlots(landlord)` decreases via price or configuration updates. Consequently, `timestamp - _toiler.lastToilDate` in `LandManager._farmPlots` underflows, reverting `unstakeMunchable` and freezing the renter's staked NFTs.

### Technical details

In `LandManager.stakeMunchable`, the available plot count for `landlord` is dynamically calculated via `_getNumPlots(landlord)` (line 145), which divides `lockManager.getLockedWeightedValue(landlord)` by `PRICE_PER_PLOT` (line 345). When a renter stakes a Munchable NFT, `toilerState[tokenId]` is recorded with `lastToilDate: block.timestamp` (line 163), and whenever the renter subsequently farms, `toilerState[tokenId].lastToilDate` is updated to `block.timestamp` (line 291).

In `LandManager._farmPlots`, if a landlord's available plot count drops below the staked `plotId`, the contract attempts to cap farming rewards at the timestamp when the landlord's plot metadata was last updated:

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

It then calculates the accrued Schnibbles on line 280–282:

```solidity
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
```

This logic assumes that `_getNumPlots(landlord)` can only decrease when the landlord unlocks tokens through `LockManager`, which triggers `AccountManager` to call `landManager.updatePlotMetadata(landlord)` (setting `plotMetadata[landlord].lastUpdated = block.timestamp`), and that `plotMetadata[landlord].lastUpdated` is always greater than or equal to `_toiler.lastToilDate`.

This assumption fails in two ways:

1. **Dynamic plot capacity reduction without `updatePlotMetadata`**:
   A landlord locks tokens at $T_0$, setting `plotMetadata[landlord].lastUpdated = T_0`. A renter stakes a Munchable in `stakeMunchable` (line 163) or farms plots (line 291) at $T_1 > T_0$, setting `_toiler.lastToilDate = T_1`. Later at $T_2 > T_1$, `_getNumPlots(landlord)` decreases without the landlord calling `LockManager`—either because `PRICE_PER_PLOT` is increased in `ConfigStorage` (updated in `LandManager._reconfigure` at line 77) or because a locked token's `usdPrice` is lowered or `active` status is disabled in `LockManager.getLockedWeightedValue` (lines 478, 489). Because `updatePlotMetadata(landlord)` is not called during these updates, `plotMetadata[landlord].lastUpdated` remains at $T_0 < T_1$.

2. **Staking to a landlord with uninitialized `plotMetadata`**:
   If a landlord locked tokens prior to `LandManager` deployment and has not called `triggerPlotMetadata()` (lines 104–114), `plotMetadata[landlord].lastUpdated` is `0` and `plotMetadata[landlord].currentTaxRate` is `0`. Unlike `updateTaxRate` (line 96), `stakeMunchable` (lines 145–164) does not revert when `plotMetadata[landlord].lastUpdated == 0`. A renter staking to this landlord receives `latestTaxRate = 0` (depriving the landlord of `DEFAULT_TAX_RATE`) and `lastToilDate = block.timestamp > 0`. Any reduction in `_getNumPlots(landlord)` causes `timestamp` to be set to `0` in `_farmPlots`.

In both scenarios, `timestamp` (`plotMetadata[landlord].lastUpdated`) is strictly less than `_toiler.lastToilDate`. Executing `timestamp - _toiler.lastToilDate` on line 281 reverts with an arithmetic underflow (Panic `0x11`). Because `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` all apply the `forceFarmPlots(msg.sender)` modifier, every attempt by the renter to unstake or move any of their staked Munchables reverts.

### Assumptions and preconditions

- A landlord has locked tokens in `LockManager` providing at least `plotId + 1` plots at time $T_0$ (with `plotMetadata[landlord].lastUpdated == T_0`, or `0` if locked before `LandManager` deployment).
- A renter stakes a Munchable on `plotId` via `stakeMunchable` at time $T_1 > T_0$, setting `toilerState[tokenId].lastToilDate = T_1`.
- Subsequently, `_getNumPlots(landlord)` drops below `plotId` due to an increase in `PRICE_PER_PLOT` or a decrease in `getLockedWeightedValue(landlord)` (via token `usdPrice` reduction or token deactivation) before `plotMetadata[landlord].lastUpdated` is updated past $T_1$.

### Proof of concept

1. At $T_0 = 1000$, Landlord `alice` locks tokens worth $3,000 USD in `LockManager`. `AccountManager` invokes `landManager.updatePlotMetadata(alice)`, setting `plotMetadata[alice].lastUpdated = 1000`. With `PRICE_PER_PLOT = 30e18`, `_getNumPlots(alice) = 100`.
2. At $T_1 = 2000$, Renter `bob` calls `landManager.stakeMunchable(alice, 1, 90)`.
   - `_getNumPlots(alice)` is `100`, so `plotId = 90 < 100` passes line 146.
   - `toilerState[1].lastToilDate` is set to `2000` (line 163), while `plotMetadata[alice].lastUpdated` remains `1000`.
3. At $T_2 = 3000$, the protocol updates the USD price of `alice`'s locked token in `LockManager` (or increases `PRICE_PER_PLOT` in `ConfigStorage`), reducing `_getNumPlots(alice)` from `100` to `80`. `plotMetadata[alice].lastUpdated` is still `1000`.
4. At $T_3 = 4000$, `bob` calls `landManager.unstakeMunchable(1)` to retrieve his Munchable NFT.
5. The `forceFarmPlots(msg.sender)` modifier executes `_farmPlots(bob)`:
   - For `tokenId = 1`, `_getNumPlots(alice)` (`80`) `< _toiler.plotId` (`90`) evaluates to `true` on line 258.
   - `timestamp` is set to `plotMetadata[alice].lastUpdated` (`1000`) on line 259.
   - Line 281 computes `timestamp - _toiler.lastToilDate` = `1000 - 2000`, which reverts with an arithmetic underflow.
6. `bob` is unable to call `unstakeMunchable`, `transferToUnoccupiedPlot`, `stakeMunchable`, or `farmPlots`. All of `bob`'s staked Munchable NFTs are stuck in `LandManager`.

### Impact assessment

**Severity: High**

When a landlord's plot count decreases due to token price adjustments, token deactivation, or `PRICE_PER_PLOT` updates (or if the landlord never called `triggerPlotMetadata()`), any renter whose `lastToilDate` is newer than `plotMetadata[landlord].lastUpdated` suffers a complete denial of service across all functions guarded by `forceFarmPlots`. Because `_farmPlots` iterates over the renter's entire `munchablesStaked[mainAccount]` array, a single affected plot freezes all of the renter's staked Munchable NFTs and unclaimed Schnibbles in `LandManager` unless the landlord performs a lock/unlock transaction to refresh `plotMetadata[landlord].lastUpdated`.

### Remediation

1. In `LandManager.stakeMunchable`, require that the landlord's `plotMetadata` has been initialized (`plotMetadata[landlord].lastUpdated != 0`).
2. In `LandManager._farmPlots`, ensure `timestamp` cannot be set to a value earlier than `_toiler.lastToilDate` when a plot becomes invalid (e.g., by bounding `timestamp` with `Math.max(plotMetadata[landlord].lastUpdated, _toiler.lastToilDate)`).

```solidity
    function stakeMunchable(
        address landlord,
        uint256 tokenId,
        uint256 plotId
    ) external override forceFarmPlots(msg.sender) notPaused {
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        if (landlord == mainAccount) revert CantStakeToSelfError();
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();
        if (plotOccupied[landlord][plotId].occupied)
            revert OccupiedPlotError(landlord, plotId);
        if (munchablesStaked[mainAccount].length > 10)
            revert TooManyStakedMunchiesError();
        if (munchNFT.ownerOf(tokenId) != mainAccount)
            revert InvalidOwnerError();

        uint256 totalPlotsAvail = _getNumPlots(landlord);
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();

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
    }
```

And in `_farmPlots`:

```solidity
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated > _toiler.lastToilDate
                    ? plotMetadata[landlord].lastUpdated
                    : _toiler.lastToilDate;
                toilerState[tokenId].dirty = true;
            }
```

## Finding 4: Missing Landlord Registration and `plotMetadata` Initialization Checks in `LandManager.stakeMunchable` Cause Permanent Staked NFT Lockup

### Synopsis

`LandManager.stakeMunchable` allows renters to stake NFTs onto any `landlord` address that has locked tokens (`_getNumPlots(landlord) > plotId`), without verifying that `landlord` is a registered main account or that `plotMetadata[landlord]` has been initialized. This causes `_farmPlots` and `unstakeMunchable` to revert, locking the renter's staked NFTs.

### Technical details

In `LandManager.stakeMunchable`, the contract validates the caller (`msg.sender`) via `_getMainAccountRequireRegistered(msg.sender)` and checks whether the target `landlord` has enough available plots via `_getNumPlots(landlord)`:

```solidity
uint256 totalPlotsAvail = _getNumPlots(landlord);
if (plotId >= totalPlotsAvail) revert PlotTooHighError();
```

However, `stakeMunchable` never verifies that:
1. `landlord` is a registered player (and resolves to its canonical main account via `_getMainAccountRequireRegistered(landlord)`), and
2. `plotMetadata[landlord].lastUpdated != 0` (or initializes `plotMetadata[landlord]` if needed).

In `AccountManager.forceHarvest`, `landManager.updatePlotMetadata(_player)` is only invoked when `LockManager` triggers a harvest for an already-registered player (`player.registrationDate != 0`):

```solidity
(, MunchablesCommonLib.Player memory player) = this.getPlayer(_player);
if (player.registrationDate != 0) {
    _harvest(_player);
    if (
        msg.sender == configStorage.getAddress(StorageKey.LockManager)
    ) {
        landManager.updatePlotMetadata(_player);
    }
}
```

This creates two inconsistent states where `_getNumPlots(landlord) > 0` holds in `stakeMunchable`, but subsequent operations in `_farmPlots` fail or misbehave:

1. **Unregistered `landlord` (`registrationDate == 0`):** A user can lock tokens in `LockManager` before registering in `AccountManager` (or without ever registering). Because `_getNumPlots(landlord)` queries `lockManager.getLockedWeightedValue(landlord) / PRICE_PER_PLOT`, `totalPlotsAvail` is non-zero and `stakeMunchable` succeeds. However, once the token is staked, every call to `_farmPlots(msg.sender)` executes `_getMainAccountRequireRegistered(landlord)` at line 265, which reverts with `PlayerNotRegisteredError()`. Because `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` all carry the `forceFarmPlots(msg.sender)` modifier, the renter's staked Munchables are completely frozen and cannot be unstaked unless `landlord` registers.
2. **Registered `landlord` with uninitialized `plotMetadata` (`plotMetadata[landlord].lastUpdated == 0`):** If a landlord locks tokens first and registers afterward (or locked tokens before `LandManager` was deployed and has not called `triggerPlotMetadata()`), `plotMetadata[landlord]` remains uninitialized (`lastUpdated == 0`, `currentTaxRate == 0`). When a renter stakes to this landlord:
   - `toilerState[tokenId].latestTaxRate` is set to `0` instead of `DEFAULT_TAX_RATE`, depriving the landlord of tax rewards.
   - If the landlord later unlocks tokens (or `PRICE_PER_PLOT` increases) such that `_getNumPlots(landlord) < _toiler.plotId` before `plotMetadata[landlord]` is updated, `_farmPlots` sets `timestamp = plotMetadata[landlord].lastUpdated` (`0`) at line 259 and then evaluates `(timestamp - _toiler.lastToilDate)` at line 281. This underflows (`0 - lastToilDate`) and reverts with an arithmetic panic, permanently preventing the renter from calling `unstakeMunchable`, `farmPlots`, `stakeMunchable`, or `transferToUnoccupiedPlot`.

### Assumptions and preconditions

- A `landlord` address has locked tokens in `LockManager` such that `_getNumPlots(landlord) > 0`, either before registering in `AccountManager` or before `LandManager` was deployed (without calling `triggerPlotMetadata()`).
- A registered renter calls `stakeMunchable(landlord, tokenId, plotId)` with a valid `plotId < _getNumPlots(landlord)`.

### Proof of concept

1. Alice locks tokens in `LockManager` without registering in `AccountManager` (or locks tokens first and then registers in `AccountManager` without locking again, leaving `plotMetadata[alice].lastUpdated == 0`).
2. Because `lockManager.getLockedWeightedValue(alice) >= PRICE_PER_PLOT`, `_getNumPlots(alice)` returns at least `1`.
3. Bob (a registered player) approves `LandManager` for `tokenId = 1` and calls `stakeMunchable(alice, 1, 0)`.
4. `stakeMunchable` succeeds and transfers Bob's NFT (`tokenId = 1`) to `LandManager`.
5. **Scenario A (Alice is unregistered):** Bob attempts to call `farmPlots()` or `unstakeMunchable(1)`. The `forceFarmPlots(msg.sender)` modifier executes `_farmPlots(bob)`, which calls `_getMainAccountRequireRegistered(alice)` at line 265 and reverts with `PlayerNotRegisteredError()`. Bob cannot unstake his NFT.
6. **Scenario B (Alice registered after locking, so `plotMetadata[alice].lastUpdated == 0`):** While `plotMetadata[alice].lastUpdated == 0`, `PRICE_PER_PLOT` increases (or Alice's locked weighted value drops) so `_getNumPlots(alice) < _toiler.plotId`. When Bob calls `unstakeMunchable(1)`, `_farmPlots(bob)` sets `timestamp = plotMetadata[alice].lastUpdated = 0` at line 259 and reverts due to arithmetic underflow on `timestamp - _toiler.lastToilDate` at line 281. Bob's NFT is permanently stuck in `LandManager`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Renters who stake a Munchable to an unregistered landlord or a landlord whose `plotMetadata` has not been initialized suffer a denial of service in `_farmPlots`, which blocks `unstakeMunchable`, `stakeMunchable`, `transferToUnoccupiedPlot`, and `farmPlots`.
- **Business Implications:** Directly violates a primary protocol invariant highlighted in the audit scope ("deadlocks (ie. where an individual's Munchable is stuck)"), trapping user NFTs inside `LandManager` and halting all farming rewards for affected renters.

### Remediation

In `LandManager.stakeMunchable`, resolve and validate that `landlord` is a registered main account via `_getMainAccountRequireRegistered(landlord)` and ensure `plotMetadata[landlord].lastUpdated != 0` (either by reverting if uninitialized or by initializing `plotMetadata[landlord]` when `lastUpdated == 0`).

```solidity
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        (address landlordMainAccount, ) = _getMainAccountRequireRegistered(landlord);
        landlord = landlordMainAccount;
        if (landlord == mainAccount) revert CantStakeToSelfError();
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();
        if (plotOccupied[landlord][plotId].occupied)
            revert OccupiedPlotError(landlord, plotId);
        if (munchablesStaked[mainAccount].length > 10)
            revert TooManyStakedMunchiesError();
        if (munchNFT.ownerOf(tokenId) != mainAccount)
            revert InvalidOwnerError();

        uint256 totalPlotsAvail = _getNumPlots(landlord);
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();
```

## Finding 5: Off-by-one Plot Capacity Check in `LandManager._farmPlots` Allows Renters on Invalidated Plots to Farm Indefinitely

### Synopsis

In `LandManager._farmPlots`, the boundary check comparing a landlord's available plots against a staked Munchable's `plotId` uses strict inequality (`<`) instead of `<=`. Because `plotId` is zero-indexed, a renter staked on the boundary plot (`plotId == _getNumPlots(landlord)`) bypasses invalidation and farms Schnibbles indefinitely.

### Technical details

In `LandManager`, plots are zero-indexed: a landlord with `totalPlotsAvail = _getNumPlots(landlord)` has valid plot IDs in the range `[0, totalPlotsAvail - 1]`. Both `stakeMunchable` (line 146) and `transferToUnoccupiedPlot` (line 211) enforce this invariant by reverting with `PlotTooHighError` when `plotId >= totalPlotsAvail`.

When a landlord unlocks tokens in `LockManager`, their locked weighted value decreases, which reduces `_getNumPlots(landlord)` and updates `plotMetadata[landlord].lastUpdated`. In `LandManager._farmPlots`, the contract attempts to detect whether the plot occupied by a staked Munchable is no longer backed by the landlord's locked funds (lines 258–260):

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

Because `_getNumPlots(landlord)` returns the count of valid plots while `_toiler.plotId` is a zero-based index, a plot becomes invalid as soon as `_getNumPlots(landlord) <= _toiler.plotId`. Using `<` instead of `<=` causes an off-by-one error when `_getNumPlots(landlord) == _toiler.plotId`.

Specifically:
1. If a landlord has `N` plots (valid indices `0` through `N - 1`) and a renter stakes on the last plot (`plotId = N - 1`), and the landlord subsequently unlocks enough funds to reduce their plot count to `N - 1` (for example, a landlord with 1 plot at `plotId = 0` unlocking all funds so `_getNumPlots(landlord) == 0`), `_getNumPlots(landlord) < _toiler.plotId` evaluates to `false` (`N - 1 < N - 1`).
2. `toilerState[tokenId].dirty` is never set to `true`, and `timestamp` remains `block.timestamp` instead of being capped at `plotMetadata[landlord].lastUpdated`.
3. Both the renter and the landlord continue to accrue `unfedSchnibbles` on every subsequent `_farmPlots` call for a plot that no longer exists.

### Assumptions and preconditions

1. A landlord has locked funds providing `N >= 1` plots.
2. A renter stakes a Munchable on `plotId = K` (where `0 <= K < N`).
3. The landlord unlocks tokens such that their remaining locked value yields exactly `K` plots (`_getNumPlots(landlord) == K`), such as unlocking all funds when `K == 0`.

### Proof of concept

1. Alice (landlord) locks tokens sufficient for 1 plot (`_getNumPlots(alice) == 1`). The only valid plot ID is `0`.
2. Bob (renter) calls `stakeMunchable(alice, tokenId, 0)`. Because `0 < 1`, the check `if (plotId >= totalPlotsAvail)` passes, and `toilerState[tokenId].plotId` is set to `0`.
3. Alice unlocks all of her locked tokens. `plotMetadata[alice].lastUpdated` is updated to the unlock timestamp, and `_getNumPlots(alice)` becomes `0`.
4. Later, Bob calls `farmPlots()`. In `_farmPlots`:
   - `_getNumPlots(alice)` returns `0`.
   - `_toiler.plotId` is `0`.
   - The condition `_getNumPlots(landlord) < _toiler.plotId` (`0 < 0`) evaluates to `false`.
5. `toilerState[tokenId].dirty` remains `false` and `timestamp` remains `block.timestamp`. Bob and Alice receive Schnibbles for the full elapsed time despite Alice having `0` plots, and they can repeat this indefinitely.

### Impact assessment

**Severity: Medium**

Renters staked on a plot whose index equals the landlord's new plot count (most notably `plotId = 0` when a landlord withdraws all locked funds) are never marked `dirty`. This allows both the renter and the landlord to mint unbacked `unfedSchnibbles` indefinitely without any locked capital backing the plot, undermining the core game reward economy.

### Remediation

Update the boundary condition in `LandManager._farmPlots` to use `<=` instead of `<` so that any `plotId` greater than or equal to `_getNumPlots(landlord)` is treated as out of bounds:

```solidity
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated;
                toilerState[tokenId].dirty = true;
            }
```

## Finding 6: Discarded Canonical Landlord Address in `LandManager._farmPlots` Causes Player State Read/Write Key Mismatch

### Synopsis

In `LandManager._farmPlots`, the call to `_getMainAccountRequireRegistered(landlord)` discards the resolved canonical main account address and later passes the raw `landlord` address to `accountManager.updatePlayer`. Because `AccountManager.getPlayer` resolves sub-accounts while `AccountManager.updatePlayer` writes directly to `players[_account]`, landlord rewards are written to the wrong storage key and lost.

### Technical details

In `AccountManager`, player records are indexed in the `players` mapping by the player's canonical `mainAccount`. However, `AccountManager.getPlayer` and `AccountManager.updatePlayer` handle account resolution asymmetrically:

1. `AccountManager.getPlayer(address _account)` resolves `_account` via `_getMainAccount(_account)` (lines 405–407) and returns `(_mainAccount, players[_mainAccount])`.
2. `AccountManager.updatePlayer(address _account, MunchablesCommonLib.Player calldata _player)` does not call `_getMainAccount(_account)`; instead, it writes directly to `players[_account] = _player` (line 364).

In `LandManager._farmPlots` (lines 262–265), the contract retrieves the landlord's `Player` struct using `_getMainAccountRequireRegistered(landlord)`:

```solidity
(
    ,
    MunchablesCommonLib.Player memory landlordMetadata
) = _getMainAccountRequireRegistered(landlord);
```

Here, the first return value—the resolved `_mainAccount` for `landlord`—is discarded. After adding `schnibblesLandlord` and updating `lastPetMunchable` on `landlordMetadata` (lines 298–299), `_farmPlots` persists the updated struct by calling:

```solidity
accountManager.updatePlayer(landlord, landlordMetadata);
```

When `landlord` is a sub-account associated with a main account (either because a sub-account address was used when staking or because `landlord` was linked as a sub-account in `AccountManager`), `_getMainAccountRequireRegistered(landlord)` reads `landlordMetadata` from `players[landlordMainAccount]`, whereas `accountManager.updatePlayer(landlord, landlordMetadata)` writes the updated struct to `players[landlord]`.

Consequently:
- `players[landlordMainAccount]` is never updated with the farmed `schnibblesLandlord` or `lastPetMunchable`.
- Across multiple staked tokens in the same `_farmPlots` loop for the same `landlord`, each iteration re-reads the stale `players[landlordMainAccount]` record and overwrites `players[landlord]`, discarding the landlord schnibbles accrued in previous iterations.
- All subsequent reads via `accountManager.getPlayer(landlord)` or `accountManager.getPlayer(landlordMainAccount)` resolve to `players[landlordMainAccount]`, meaning the landlord cannot access the schnibbles written to `players[landlord]`.

### Assumptions and preconditions

- A Munchable is staked to a `landlord` address that resolves to a different `mainAccount` in `AccountManager` (e.g., `landlord` is a sub-account of a registered player, or becomes a sub-account prior to `_farmPlots` being executed).
- `_farmPlots` is triggered directly via `farmPlots()` or via the `forceFarmPlots` modifier on `stakeMunchable`, `unstakeMunchable`, or `transferToUnoccupiedPlot`.

### Proof of concept

1. Alice is a registered main account in `AccountManager` (`players[alice].registrationDate != 0`, `players[alice].unfedSchnibbles == 0`) and has `aliceSub` registered as a sub-account (`mainAccounts[aliceSub] == alice`), with locked funds providing available plots on `aliceSub` (or `alice` links `aliceSub` as a sub-account after plot setup).
2. Bob (a registered renter) stakes two Munchables (`tokenId1` and `tokenId2`) onto `aliceSub`'s plots via `LandManager.stakeMunchable(aliceSub, ...)`.
3. After time passes, Bob calls `LandManager.farmPlots()`, which invokes `_farmPlots(bob)`.
4. In iteration `i = 0` (`tokenId1`):
   - `_getMainAccountRequireRegistered(aliceSub)` calls `accountManager.getPlayer(aliceSub)`, which resolves `_getMainAccount(aliceSub) -> alice` and returns `(alice, players[alice])`.
   - `LandManager._farmPlots` discards the returned `alice` address and stores `players[alice]` in `landlordMetadata`.
   - `landlordMetadata.unfedSchnibbles` is incremented by `schnibblesLandlord_1`.
   - `accountManager.updatePlayer(aliceSub, landlordMetadata)` writes `landlordMetadata` to `players[aliceSub]` instead of `players[alice]`.
5. In iteration `i = 1` (`tokenId2`):
   - `_getMainAccountRequireRegistered(aliceSub)` again reads `players[alice]`, where `unfedSchnibbles` is still `0`.
   - `accountManager.updatePlayer(aliceSub, landlordMetadata)` overwrites `players[aliceSub]` with only `schnibblesLandlord_2`.
6. When Alice queries `accountManager.getPlayer(alice)` or `accountManager.getPlayer(aliceSub)`, `AccountManager` reads `players[alice]`, which still has `unfedSchnibbles == 0`. Both `schnibblesLandlord_1` and `schnibblesLandlord_2` are inaccessible to Alice's main account.

### Impact assessment

- **Severity**: Medium / High.
- **Technical Effect**: State updates for `landlordMetadata` in `LandManager._farmPlots` are written to `players[landlord]` instead of `players[landlordMainAccount]`, desynchronizing `AccountManager` storage between sub-accounts and main accounts.
- **Business Impact**: Landlords whose plots are associated with a sub-account permanently lose their tax share of farmed schnibbles (`schnibblesLandlord`) and `lastPetMunchable` timestamp updates during plot farming.

### Remediation

Capture the canonical `landlordMainAccount` returned by `_getMainAccountRequireRegistered(landlord)` in `LandManager._farmPlots` and pass `landlordMainAccount` to `accountManager.updatePlayer` (and ensure `landlord` is canonicalized when staking).

```solidity
            (
                address landlordMainAccount,
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

            renterMetadata.unfedSchnibbles += (schnibblesTotal -
                schnibblesLandlord);

            landlordMetadata.unfedSchnibbles += schnibblesLandlord;
            landlordMetadata.lastPetMunchable = uint32(timestamp);
            accountManager.updatePlayer(landlordMainAccount, landlordMetadata);
```

## Finding 7: Unvalidated Attribute and Bonus Table Lookups in `LandManager._farmPlots` Cause Out-of-Bounds Reverts or Corrupted Farming Bonuses

### Synopsis

In `LandManager._farmPlots`, `finalBonus` is computed by directly indexing `REALM_BONUSES` and `RARITY_BONUSES` using unvalidated `immutableAttributes` and `snuggeryRealm` enum values. Mismatches between enum domains (`Realm`/`Rarity`, which include `Invalid`/`Primordial`), flattened 5x5 table indexing, or incomplete attribute records cause corrupted bonuses or out-of-bounds reverts that freeze staked Munchables.

### Technical details

In `src/managers/LandManager.sol`, `_farmPlots` retrieves the staked token's immutable attributes and computes `finalBonus` on lines 267–279:

```solidity
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
```

This lookup has three interconnected integrity flaws:

1. **Enum domain incompatibility with configured bonus tables:**
   In `MunchablesCommonLib.sol`, `Realm` defines 6 variants (`Everfrost`, `Drench`, `Moltania`, `Arridia`, `Verdentis`, `Invalid` = `0..5`) and `Rarity` defines 7 variants (`Primordial`, `Common`, `Rare`, `Epic`, `Legendary`, `Mythic`, `Invalid` = `0..6`). However, `REALM_BONUSES` is configured as a flattened 5x5 table of 25 elements (for the 5 valid realms `0..4`) and `RARITY_BONUSES` is configured for valid rarities (`0..5`, or `1..5` for non-primordial Munchables). Neither `stakeMunchable` nor `_farmPlots` validates that `immutableAttributes.realm < Realm.Invalid` or `immutableAttributes.rarity < Rarity.Invalid` (and `< RARITY_BONUSES.length`). If a staked token has `Rarity.Invalid` (`6`) or `Realm.Invalid` (`5`), indexing `RARITY_BONUSES` or `REALM_BONUSES` triggers an out-of-bounds panic (`Panic(0x32)`).

2. **Flattened 2D table row wrap-around / dimension mismatch:**
   The flattened index `(uint256(immutableAttributes.realm) * 5) + uint256(landlordMetadata.snuggeryRealm)` assumes both dimensions are strictly `< 5`. If `landlordMetadata.snuggeryRealm` is `Realm.Invalid` (`5`) while `immutableAttributes.realm < 4`, the index evaluates to `(realm + 1) * 5 + 0 < 25`, silently wrapping into the next row of `REALM_BONUSES` and applying the wrong realm bonus instead of reverting.

3. **Unvalidated / incomplete attribute records from `getImmutableAttributes`:**
   `NFTAttributeManagerV1.getImmutableAttributes(tokenId)` directly returns `immutableAttributes[_tokenId]` from storage without verifying that the record was initialized. For a token with default or incomplete attributes (`hatchedDate == 0`, `rarity == Rarity.Primordial (0)`, `realm == Realm.Everfrost (0)`), `_farmPlots` silently uses index `0` (`Primordial` / `Everfrost`) to calculate `finalBonus`.

Because `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` all execute the `forceFarmPlots(msg.sender)` modifier, any out-of-bounds panic in `_farmPlots` permanently blocks the user from unstaking any of their staked Munchables.

### Assumptions and preconditions

1. A Munchable NFT staked in `LandManager` has an uninitialized, incomplete, or `Invalid`-valued `NFTImmutableAttributes` record in `NFTAttributesManager` (or the configured `REALM_BONUSES` / `RARITY_BONUSES` arrays have lengths smaller than the enum domains).
2. A player stakes this NFT via `stakeMunchable` and subsequently triggers `_farmPlots` (directly via `farmPlots()` or indirectly via `unstakeMunchable`, `stakeMunchable`, or `transferToUnoccupiedPlot`).

### Proof of concept

1. Suppose `RARITY_BONUSES` has length 6 (indices `0..5`) and `REALM_BONUSES` has length 25 (5x5 grid, indices `0..24`).
2. A renter owns `tokenId` whose `immutableAttributes` in `NFTAttributesManager` either:
   - Have not been populated yet (defaulting to `realm = Realm.Everfrost (0)`, `rarity = Rarity.Primordial (0)`, `hatchedDate = 0`), causing `_farmPlots` to compute `finalBonus` using `Everfrost` and `Primordial` bonuses for an uninitialized token; or
   - Have `realm = Realm.Invalid (5)` or `rarity = Rarity.Invalid (6)`.
3. The renter stakes `tokenId` via `stakeMunchable(landlord, tokenId, plotId)`, which succeeds because `stakeMunchable` does not validate `immutableAttributes`.
4. When the renter later calls `unstakeMunchable(tokenId)`, the `forceFarmPlots(msg.sender)` modifier executes `_farmPlots(msg.sender)`.
5. On lines 270–279, `(uint256(immutableAttributes.realm) * 5) + uint256(landlordMetadata.snuggeryRealm)` evaluates to `25 + uint256(landlordMetadata.snuggeryRealm) >= 25` (or `uint256(immutableAttributes.rarity)` evaluates to `6 >= RARITY_BONUSES.length`), reverting with an out-of-bounds array panic (`Panic(0x32)`).
6. All of the renter's staked Munchables (up to 10 NFTs) are permanently locked in `LandManager` because every call to `unstakeMunchable` must pass through `_farmPlots`.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Out-of-bounds array access in `REALM_BONUSES` or `RARITY_BONUSES` causes `_farmPlots` to revert with `Panic(0x32)`, while out-of-domain `snuggeryRealm` or default `immutableAttributes` silently select the wrong bonus entry.
- **Business Impact:** Any revert in `_farmPlots` causes a permanent denial of service on `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` for the affected player, locking all of their staked Munchable NFTs in `LandManager`. Unvalidated default attributes also distort Schnibble reward emissions for both the renter and the landlord.

### Remediation

Validate the retrieved `immutableAttributes` and `snuggeryRealm` enum values against the valid domain (`< Realm.Invalid`, `> Rarity.Primordial` / `< Rarity.Invalid`) and the configured array lengths before indexing `REALM_BONUSES` and `RARITY_BONUSES`:

```solidity
            immutableAttributes = nftAttributesManager.getImmutableAttributes(
                tokenId
            );
            if (
                immutableAttributes.rarity == MunchablesCommonLib.Rarity.Primordial ||
                immutableAttributes.rarity >= MunchablesCommonLib.Rarity.Invalid ||
                uint256(immutableAttributes.rarity) >= RARITY_BONUSES.length
            ) revert InvalidRarityError(uint256(immutableAttributes.rarity));

            if (
                immutableAttributes.realm >= MunchablesCommonLib.Realm.Invalid ||
                landlordMetadata.snuggeryRealm >= MunchablesCommonLib.Realm.Invalid
            ) revert InvalidRealmError();

            uint256 realmIndex = (uint256(immutableAttributes.realm) * 5) +
                uint256(landlordMetadata.snuggeryRealm);
            if (realmIndex >= REALM_BONUSES.length)
                revert InvalidRealmBonus(realmIndex);

            finalBonus =
                int16(REALM_BONUSES[realmIndex]) +
                int16(
                    int8(RARITY_BONUSES[uint256(immutableAttributes.rarity)])
                );
```

## Finding 8: Unsafe `uint8` to `int8` cast in `LandManager._farmPlots` turns rarity bonuses above 127 into negative penalties

### Synopsis

In `LandManager._farmPlots`, the `uint8` value read from `RARITY_BONUSES` is cast to `int8` before being widened to `int16` and `int256`. If any rarity bonus is configured in the range `[128, 255]`, the narrowing conversion wraps it into a negative value, penalizing farming rewards instead of boosting them.

### Technical details

In `LandManager`, `RARITY_BONUSES` is declared as a `uint8[]` array (line 18) loaded from `ConfigStorage` via `getSmallUintArray(StorageKey.RarityBonuses)` (lines 81–83), allowing unsigned bonus percentages in the range `0` to `255`.

However, when `LandManager._farmPlots` calculates `finalBonus` (which has type `int256`) on lines 270–279, it performs an intermediate cast through `int8`:

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

Because `int8` is a signed 8-bit integer with a range of `[-128, 127]`, casting a `uint8` value in `[128, 255]` to `int8` reinterprets the most significant bit as a sign bit (`value - 256`), yielding a negative value in `[-128, -1]`. The subsequent cast to `int16` and implicit widening to `int256` sign-extend this negative value.

Consequently, when `schnibblesTotal` is calculated on lines 283–286:

```solidity
schnibblesTotal = uint256(
    (int256(schnibblesTotal) +
        (int256(schnibblesTotal) * finalBonus)) / 100
);
```

1. A high-rarity Munchable with a rarity bonus between `128` and `255` receives a negative `finalBonus` contribution instead of a positive boost, drastically reducing farmed schnibbles for both the renter and the landlord.
2. If the negative wrapped rarity bonus combined with the realm bonus results in `finalBonus < -100`, the signed expression `int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)` becomes negative, and casting it back to `uint256` wraps around to an astronomical value (`~2^256 - 1`), causing the subsequent multiplication `schnibblesTotal * _toiler.latestTaxRate` on line 288 to revert with an arithmetic overflow and permanently blocking `_farmPlots` (and any function with the `forceFarmPlots` modifier, including `unstakeMunchable`).

### Assumptions and preconditions

1. A rarity tier in `RARITY_BONUSES` is configured with a `uint8` value greater than `127` (i.e., `128` to `255`).
2. A player stakes a Munchable of that rarity tier in `LandManager` and triggers `_farmPlots` (directly via `farmPlots` or indirectly via `stakeMunchable`, `unstakeMunchable`, or `transferToUnoccupiedPlot`).

### Proof of concept

1. Suppose `RARITY_BONUSES` for `Rarity.Mythic` is configured to `150` (a `150%` bonus, which fits within `uint8`), and the matching `REALM_BONUSES` entry is `0`.
2. A renter stakes a `Mythic` Munchable on a landlord's plot and waits for `1000` seconds with `BASE_SCHNIBBLE_RATE = 1e15`.
3. The renter calls `farmPlots()`. Inside `_farmPlots`:
   - Base `schnibblesTotal = 1000 * 1e15 = 1e18`.
   - `int8(uint8(150))` evaluates to `-106`.
   - `finalBonus = 0 + int16(-106) = -106`.
   - On lines 283–286, `(int256(1e18) + (int256(1e18) * -106)) / 100 = -1.05e18 / 100 = -1.05e16`.
   - Casting `-1.05e16` to `uint256` wraps to `type(uint256).max - 1.05e16 + 1`.
   - On line 288, `schnibblesTotal * _toiler.latestTaxRate` overflows `uint256` and reverts, locking the staked Munchable in `LandManager` because `unstakeMunchable` also executes `forceFarmPlots(msg.sender)`.
4. If the `REALM_BONUSES` entry is `+10` instead of `0`, `finalBonus = 10 + (-106) = -96`. Instead of receiving `2.6e18` schnibbles (`+160%` bonus), the renter and landlord receive only `0.04e18` schnibbles (`-96%` penalty).

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: Loss of farming rewards due to sign corruption on `uint8` rarity bonuses `>= 128`, or denial of service (revert due to underflow/wrap-around and subsequent multiplication overflow) in `_farmPlots`, which also prevents unstaking via `unstakeMunchable`.
- **Business Impact**: Players holding the highest-rarity NFTs suffer severe reward reductions or have their staked NFTs frozen in `LandManager` if any rarity bonus is set above `127`.

### Remediation

Widen the `uint8` rarity bonus to `uint16` (or `uint256`) before converting it to a signed integer (`int16` or `int256`) so that all `uint8` values in `[0, 255]` remain positive.

```solidity
            finalBonus =
                int256(
                    REALM_BONUSES[
                        (uint256(immutableAttributes.realm) * 5) +
                            uint256(landlordMetadata.snuggeryRealm)
                    ]
                ) +
                int256(
                    uint256(RARITY_BONUSES[uint256(immutableAttributes.rarity)])
                );
```

## Finding 9: Misplaced Parentheses in `LandManager._farmPlots` Bonus Calculation Causes Incorrect Reward Scaling and Integer Wrap-Around

### Synopsis

In `LandManager._farmPlots`, misplaced parentheses in the `schnibblesTotal` bonus calculation divide the base schnibbles amount by `100` along with the percentage bonus term. This scales rewards to a fraction of the intended amount for non-negative bonuses and causes signed-to-unsigned wrap-around when `finalBonus` is negative, reverting `_farmPlots` and freezing staked NFTs.

### Technical details

In `src/managers/LandManager.sol`, `_farmPlots` calculates the total schnibbles earned by a staked Munchable and adjusts it by `finalBonus`, which is the sum of `REALM_BONUSES` and `RARITY_BONUSES` expressed in whole percentage points (`1 = 1%`, with possible negative values down to `-20` when the Munchable's realm mismatches the landlord's snuggery realm):

```solidity
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
schnibblesTotal = uint256(
    (int256(schnibblesTotal) +
        (int256(schnibblesTotal) * finalBonus)) / 100
);
```

The intended formula to apply a percentage bonus `finalBonus` to `schnibblesTotal` is:

$$\text{schnibblesTotal} + \frac{\text{schnibblesTotal} \times \text{finalBonus}}{100} = \frac{\text{schnibblesTotal} \times (100 + \text{finalBonus})}{100}$$

However, because `(int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus))` is wrapped in parentheses before dividing by `100`, the code actually computes:

$$\frac{\text{schnibblesTotal} \times (1 + \text{finalBonus})}{100}$$

This introduces two severe failure modes:

1. **Severe under-rewarding when `finalBonus >= 0`:**
   - When `finalBonus == 0`, the user receives `schnibblesTotal / 100` (1% of the base reward) instead of `100%`.
   - When `finalBonus == 10` (+10% bonus), the user receives `11%` of `schnibblesTotal` instead of `110%`.

2. **Signed-to-unsigned wrap-around and DoS when `finalBonus < -1`:**
   - Because `REALM_BONUSES` contains negative values (e.g., `-10` or `-20`) and `RARITY_BONUSES` starts at `0` for common Munchables, `finalBonus` can be negative (e.g., `-10`).
   - When `finalBonus < -1` and `timestamp > _toiler.lastToilDate`, the expression `(int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)) / 100` evaluates to a negative `int256` value.
   - Casting this negative `int256` to `uint256` on line 283 wraps around in two's complement to an astronomically large value near `type(uint256).max`.
   - On line 288, multiplying this wrapped `schnibblesTotal` by `_toiler.latestTaxRate` (which is at least `MIN_TAX_RATE`) overflows `uint256` and reverts with a panic (`0x11`).
   - Because `stakeMunchable`, `unstakeMunchable`, `transferToUnoccupiedPlot`, and `farmPlots` all invoke `_farmPlots`, any player who stakes a Munchable resulting in a negative `finalBonus` will be permanently unable to farm or unstake any of their staked Munchables once at least 1 second has elapsed.

### Assumptions and preconditions

- The protocol is configured with standard `REALM_BONUSES` (which include negative values such as `-10` or `-20` for mismatched realms) and `RARITY_BONUSES` (which include `0` or small positive percentages).
- A registered renter stakes a Munchable on a registered landlord's plot, and time advances by at least 1 second (`block.timestamp > _toiler.lastToilDate`).

### Proof of concept

1. Alice (landlord) registers with `snuggeryRealm = 0` and locks tokens to create available plots with `currentTaxRate = DEFAULT_TAX_RATE` (e.g., `25e16`).
2. Bob (renter) owns a Munchable (`tokenId = 1`) whose immutable attributes give a negative `REALM_BONUSES` value (e.g., `-10`) and a `RARITY_BONUSES` value of `0`, resulting in `finalBonus = -10`.
3. Bob calls `LandManager.stakeMunchable(alice, 1, 0)` at `t = 1000`.
4. At `t = 1100` (`100` seconds later), Bob calls `LandManager.unstakeMunchable(1)` (or `farmPlots()`).
5. Inside `_farmPlots(bob)`:
   - `schnibblesTotal = (1100 - 1000) * BASE_SCHNIBBLE_RATE` (e.g., `100 * 1e15 = 1e17`).
   - `(int256(1e17) + (int256(1e17) * -10)) / 100 = -9e17 / 100 = -9e15`.
   - `uint256(-9e15)` wraps to `type(uint256).max - 9e15 + 1`.
   - Line 288 computes `(schnibblesTotal * 25e16) / 1e18`, which overflows `uint256` and reverts the transaction.
6. Bob's Munchable (along with any other Munchables Bob has staked in `LandManager`) is permanently frozen in the contract and cannot be unstaked.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  1. When `finalBonus < -1`, `_farmPlots` reverts due to `uint256` wrap-around and subsequent multiplication overflow, permanently locking all of the renter's staked Munchable NFTs in `LandManager` (`unstakeMunchable` always calls `_farmPlots` via `forceFarmPlots`).
  2. When `finalBonus >= 0`, both renters and landlords receive erroneous schnibble rewards scaled to `(1 + finalBonus)%` instead of `(100 + finalBonus)%` of the base amount.

### Remediation

Divide only the bonus component `(int256(schnibblesTotal) * finalBonus)` by `100` before adding it to `int256(schnibblesTotal)`:

```solidity
            schnibblesTotal =
                (timestamp - _toiler.lastToilDate) *
                BASE_SCHNIBBLE_RATE;
            schnibblesTotal = uint256(
                int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus) / 100
            );
```

## Finding 10: Unconditional Reset of `lockedToken.remainder` in `LockManager.unlock` Causes Loss of Uncredited NFT Lock Remainder on Partial Withdrawals

### Synopsis

In `LockManager.unlock`, `lockedToken.remainder` is unconditionally reset to `0` regardless of the unlocked `_quantity`. When a player performs a partial withdrawal (or a zero-amount unlock), any remaining uncredited token balance stored in `lockedToken.remainder` is permanently erased, preventing those still-locked tokens from counting toward future NFT mints.

### Technical details

When a player locks tokens during an active lockdrop via `LockManager._lock` (`src/managers/LockManager.sol`), the contract calculates the number of Munchable NFTs to award based on the sum of the newly locked `_quantity` and any previously uncredited `lockedToken.remainder`:

```solidity
uint256 quantity = _quantity + lockedToken.remainder;
...
remainder = quantity % configuredToken.nftCost;
numberNFTs = (quantity - remainder) / configuredToken.nftCost;
...
nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
lockedToken.remainder = remainder;
```

Here, `lockedToken.remainder` tracks the residual portion of `lockedToken.quantity` that is less than `configuredToken.nftCost` and has not yet been used to mint an NFT.

However, in `LockManager.unlock` (`src/managers/LockManager.sol`, lines 422–423):

```solidity
lockedToken.quantity -= _quantity;
lockedToken.remainder = 0;
```

Whenever a player calls `unlock(_tokenContract, _quantity)` for any `_quantity` (even when `_quantity < lockedToken.remainder` or `_quantity == 0`), `lockedToken.remainder` is unconditionally overwritten with `0`.

Consequently:
1. If a player has an accumulated `lockedToken.remainder` (e.g., `0.8e18` when `nftCost = 1e18`) and unlocks a smaller amount (e.g., `0.2e18`) after `unlockTime` expires, `0.6e18` of uncredited tokens remain locked in `LockManager`, but `lockedToken.remainder` is wiped to `0`.
2. If the player subsequently locks additional tokens during a lockdrop (e.g., `0.4e18`), `_lock` computes `quantity = 0.4e18 + 0 = 0.4e18` instead of `0.4e18 + 0.6e18 = 1.0e18`, failing to grant the NFT reveal the player is entitled to and extending `unlockTime` across the entire `lockedToken.quantity`.

### Assumptions and preconditions

1. A player has locked tokens during a lockdrop such that `quantity % configuredToken.nftCost != 0`, resulting in `lockedToken.remainder > 0`.
2. The lock period has expired (`block.timestamp >= lockedToken.unlockTime`).
3. The player calls `unlock(_tokenContract, _quantity)` with `_quantity < lockedToken.remainder` (or `_quantity < lockedToken.quantity` where uncredited remainder still remains locked).

### Proof of concept

1. Assume ETH (`address(0)`) is configured with `nftCost = 1 ether`.
2. During an active lockdrop, Alice calls `lock(address(0), 0.9 ether)` with `{value: 0.9 ether}`.
   - `numberNFTs = 0`
   - `lockedTokens[alice][address(0)].quantity = 0.9 ether`
   - `lockedTokens[alice][address(0)].remainder = 0.9 ether`
3. Once `lockedTokens[alice][address(0)].unlockTime` passes, Alice calls `unlock(address(0), 0.1 ether)` to withdraw `0.1 ether`.
   - `lockedTokens[alice][address(0)].quantity` is updated to `0.8 ether`.
   - `lockedTokens[alice][address(0)].remainder` is unconditionally set to `0` at line 423 (losing the remaining `0.8 ether` of uncredited remainder).
4. While a lockdrop is active, Alice calls `lock(address(0), 0.2 ether)` with `{value: 0.2 ether}` expecting her `0.8 ether` remainder plus `0.2 ether` new deposit to reach `1 ether` (`nftCost`) and mint 1 NFT.
5. In `_lock`, `quantity = 0.2 ether + 0 = 0.2 ether`. Alice receives `0` NFTs (`remainder = 0.2 ether`), despite having `1.0 ether` locked in `LockManager` and having never minted an NFT. Furthermore, her entire `1.0 ether` balance is now locked until the new `unlockTime`.

### Impact assessment

- **Severity**: Medium
- **Technical Effect**: Partial (or zero-value) withdrawals in `LockManager.unlock` corrupt residual lock accounting by erasing `lockedToken.remainder` even when uncredited tokens remain locked in the contract.
- **Business Impact**: Players who partially unlock tokens lose their accumulated progress (`remainder`) toward minting Munchable NFTs, resulting in fewer NFT reveals than warranted by their net locked tokens and locking their remaining funds for a new lock duration if they top up.

### Remediation

Update `lockedToken.remainder` proportionally or deduct `_quantity` from `lockedToken.remainder` (saturating at `0`) instead of unconditionally resetting it to `0`, and disallow zero-quantity unlocks.

```solidity
        // force harvest to make sure that they get the schnibbles that they are entitled to
        accountManager.forceHarvest(msg.sender);

        lockedToken.quantity -= _quantity;
        if (_quantity >= lockedToken.remainder) {
            lockedToken.remainder = 0;
        } else {
            lockedToken.remainder -= _quantity;
        }

        // send token
        if (_tokenContract == address(0)) {
            (bool success, ) = payable(msg.sender).call{value: _quantity}("");
            if (!success) revert FailedTransferError();
        } else {
            IERC20 token = IERC20(_tokenContract);
            token.transfer(msg.sender, _quantity);
        }

        emit Unlocked(msg.sender, _tokenContract, _quantity);
```

## Finding 11: Unchecked `IERC20.transfer` in `LockManager.unlock` Causes Permanent Token Lockup or Loss of Funds

### Synopsis

In `LockManager.unlock`, ERC20 tokens are transferred using raw `IERC20.transfer` instead of `SafeERC20.safeTransfer`, whereas `_lock` uses `SafeERC20.safeTransferFrom`. For tokens with nonstandard transfer semantics (returning `false` on failure or returning no value), this causes silent settlement failure with accounting divergence or permanent token lockup.

### Technical details

In `src/managers/LockManager.sol`, when a user locks ERC20 tokens via `_lock` (lines 376–384), the contract uses OpenZeppelin's `SafeERC20.safeTransferFrom`:

```solidity
if (_tokenContract != address(0)) {
    IERC20 token = IERC20(_tokenContract);
    SafeERC20.safeTransferFrom(
        token,
        _tokenOwner,
        address(this),
        _quantity
    );
}
```

However, when the user unlocks their tokens in `unlock` (lines 426–432), the contract directly invokes `IERC20.transfer` without checking the return value or using `SafeERC20.safeTransfer`:

```solidity
if (_tokenContract == address(0)) {
    (bool success, ) = payable(msg.sender).call{value: _quantity}("");
    if (!success) revert FailedTransferError();
} else {
    IERC20 token = IERC20(_tokenContract);
    token.transfer(msg.sender, _quantity);
}
```

This creates two distinct failure modes for nonstandard ERC20 tokens:

1. **Tokens that return `false` on failure instead of reverting (e.g., ZRX, EURS):** If `token.transfer(msg.sender, _quantity)` fails and returns `false`, the return value is ignored and execution continues. `lockedToken.quantity` is permanently reduced by `_quantity` (line 422), `lockedToken.remainder` is cleared to `0` (line 423), and the `Unlocked` event is emitted (line 434), even though no tokens were transferred to `msg.sender`.
2. **Tokens that do not return a boolean value on `transfer` (e.g., USDT-style tokens):** Because `_lock` uses `SafeERC20.safeTransferFrom` (which handles empty return data), users can successfully lock such tokens into `LockManager`. However, when `unlock` calls `token.transfer(msg.sender, _quantity)` through the `IERC20` interface (which expects `returns (bool)`), Solidity 0.8.25's ABI decoder reverts on the empty return data. As a result, any locked tokens of this type are permanently frozen in `LockManager`.

### Assumptions and preconditions

1. A configured ERC20 token in `LockManager` either returns `false` on transfer failure instead of reverting, or does not return a boolean value in its `transfer` implementation.
2. A registered player has locked a positive quantity of this ERC20 token via `lock` / `lockOnBehalf`, and the `unlockTime` has elapsed (`lockedToken.unlockTime <= uint32(block.timestamp)`).

### Proof of concept

#### Scenario A: Token returns `false` on transfer failure
1. A registered player Alice locks `1000e18` of a configured ERC20 token that returns `false` on failure. `_lock` succeeds via `SafeERC20.safeTransferFrom` and sets `lockedTokens[alice][token].quantity = 1000e18`.
2. After `unlockTime` passes, Alice calls `LockManager.unlock(token, 1000e18)`.
3. `LockManager.unlock` decrements `lockedTokens[alice][token].quantity` to `0` and sets `lockedTokens[alice][token].remainder` to `0`.
4. `token.transfer(alice, 1000e18)` fails (e.g., due to a token pause, blacklist/freeze on Alice's address, or insufficient balance in `LockManager`) and returns `false`.
5. `LockManager.unlock` ignores the `false` return value, emits `Unlocked(alice, token, 1000e18)`, and completes without reverting. Alice loses her `1000e18` locked balance in `LockManager` without receiving any tokens.

#### Scenario B: Token returns no value on `transfer` (e.g., USDT)
1. Alice locks `1000e6` of a no-return ERC20 token. `_lock` uses `SafeERC20.safeTransferFrom`, which accepts empty return data, so the lock succeeds.
2. After `unlockTime` passes, Alice calls `LockManager.unlock(token, 1000e6)`.
3. `token.transfer(alice, 1000e6)` succeeds at the token level with `0` bytes of return data, but the high-level Solidity call to `IERC20.transfer` reverts during ABI decoding because it expects 32 bytes for the `bool` return value.
4. Alice's tokens are permanently stuck in `LockManager`.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Accounting divergence between `lockedTokens[msg.sender][_tokenContract].quantity` and actual token balances when `transfer` returns `false`, or permanent denial of withdrawal when `transfer` returns no data.
- **Business Impact:** Direct loss or permanent lockup of user funds stored in `LockManager`.

### Remediation

Use `SafeERC20.safeTransfer` instead of `token.transfer` in `LockManager.unlock` to safely handle both missing return values and `false` return values.

```solidity
        // force harvest to make sure that they get the schnibbles that they are entitled to
        accountManager.forceHarvest(msg.sender);

        lockedToken.quantity -= _quantity;
        lockedToken.remainder = 0;

        // send token
        if (_tokenContract == address(0)) {
            (bool success, ) = payable(msg.sender).call{value: _quantity}("");
            if (!success) revert FailedTransferError();
        } else {
            IERC20 token = IERC20(_tokenContract);
            SafeERC20.safeTransfer(token, msg.sender, _quantity);
        }

        emit Unlocked(msg.sender, _tokenContract, _quantity);
```

## Finding 12: Unbounded iteration over `configuredTokenContracts` in `LockManager.getLockedWeightedValue` can cause gas exhaustion and lock user funds

### Synopsis

`LockManager.getLockedWeightedValue` iterates over the entire `configuredTokenContracts` array on every invocation. Because `configuredTokenContracts` is append-only and `getLockedWeightedValue` is invoked multiple times during mandatory accounting in `lock`, `unlock`, and `setLockDuration`, growth of the token registry can cause out-of-gas reverts and trap user funds.

### Technical details

In `LockManager.configureToken` (lines 115–127), whenever a new token is configured (`configuredTokens[_tokenContract].nftCost == 0`), its address is appended to the `configuredTokenContracts` array. There is no mechanism to remove addresses from `configuredTokenContracts`, even when a token is later marked as inactive (`active = false`), meaning the array is strictly append-only over the lifetime of the protocol.

In `LockManager.getLockedWeightedValue` (lines 469–495), the function computes a player's total USD-weighted locked value by iterating through every element of `configuredTokenContracts`:

```solidity
uint256 configuredTokensLength = configuredTokenContracts.length;
for (uint256 i; i < configuredTokensLength; i++) {
    if (
        lockedTokens[_player][configuredTokenContracts[i]].quantity > 0 &&
        configuredTokens[configuredTokenContracts[i]].active
    ) {
        ...
    }
}
```

Each iteration performs multiple storage reads (`configuredTokenContracts[i]`, `lockedTokens[_player][...]`, and `configuredTokens[...]`) regardless of whether the player has ever locked that token or whether the token is still active.

Crucially, `getLockedWeightedValue` sits on the critical path of core state-changing operations via `accountManager.forceHarvest`:
1. `LockManager.unlock`, `LockManager._lock`, and `LockManager.setLockDuration` all call `accountManager.forceHarvest`.
2. `AccountManager.forceHarvest` executes `_harvest(_player)`, which calls `AccountManager.getDailySchnibbles(_player)`:
   - `getDailySchnibbles` calls `lockManager.getLockedWeightedValue(_caller)` (line 432).
   - `getDailySchnibbles` also calls `bonusManager.getHarvestBonus(_caller)` (line 435), which again calls `_lockManager.getLockedWeightedValue(_caller)` (`BonusManager.sol` line 128).
3. When called from `LockManager`, `AccountManager.forceHarvest` also calls `landManager.updatePlotMetadata(_player)` (line 154), which calls `_getNumPlots` and triggers `lockManager.getLockedWeightedValue(_account)` a third time (`LandManager.sol` line 345).

Because a single user action (`unlock`, `lock`, or `setLockDuration`) traverses `configuredTokenContracts` at least three times (and `setLockDuration` and `getLocked` add additional full-array loops), the per-transaction gas cost scales linearly with the total number of tokens ever configured in `LockManager`. If enough tokens are added over time, these calls will exceed the block gas limit and revert, preventing users from unlocking their tokens.

### Assumptions and preconditions

- The admin configures new tokens over time via `LockManager.configureToken` (e.g., adding new supported assets or rotating/deactivating old tokens across seasons), causing `configuredTokenContracts.length` to grow.
- A player has locked ETH or ERC20 tokens in `LockManager`.

### Proof of concept

1. A player registers an account and locks ETH or ERC20 tokens via `LockManager.lock(...)`.
2. Over time, the admin configures additional tokens via `LockManager.configureToken(...)`. Even if older tokens are deactivated (`active = false`), their addresses remain in `configuredTokenContracts`.
3. Once the player's `unlockTime` is reached, the player calls `LockManager.unlock(_tokenContract, _quantity)` to withdraw their locked assets.
4. `LockManager.unlock` calls `accountManager.forceHarvest(msg.sender)`.
5. `forceHarvest` triggers `_harvest` (which invokes `LockManager.getLockedWeightedValue` twice via `getDailySchnibbles` and `BonusManager.getHarvestBonus`) and `landManager.updatePlotMetadata` (which invokes `LockManager.getLockedWeightedValue` a third time via `_getNumPlots`).
6. Because each call to `getLockedWeightedValue` iterates over the entire `configuredTokenContracts` array, the cumulative gas cost exceeds the transaction or block gas limit, causing `unlock` to revert with out-of-gas.

### Impact assessment

- **Severity**: Medium / High (Availability / Permanent Freezing of Funds).
- **Direct Technical Effect**: Calls to `getLockedWeightedValue` and all dependent state-changing functions (`unlock`, `lock`, `lockOnBehalf`, `setLockDuration`, `forceHarvest`) revert due to gas exhaustion when `configuredTokenContracts` grows too large.
- **Business Implications**: Players may be unable to execute `LockManager.unlock`, resulting in their locked ETH and ERC20 tokens being permanently stuck in the `LockManager` contract, violating a core protocol invariant against deadlocks of user assets.

### Remediation

Track only the tokens actively locked by each player (for example, using an `EnumerableSet.AddressSet` per player or maintaining and caching the player's weighted value / active token list when locking, unlocking, or updating prices), and provide an admin function to prune deactivated tokens from `configuredTokenContracts` or bound the maximum number of configured tokens.

In `getLockedWeightedValue`, iterate only over the specific tokens held by `_player` (or cache the player's weighted locked balance on state changes) rather than scanning the entire global registry:

```solidity
    function getLockedWeightedValue(
        address _player
    ) external view returns (uint256 _lockedWeightedValue) {
        uint256 lockedWeighted = 0;
        address[] memory playerTokens = playerLockedTokenContracts[_player];
        uint256 playerTokensLength = playerTokens.length;
        for (uint256 i; i < playerTokensLength; i++) {
            address tokenContract = playerTokens[i];
            uint256 quantity = lockedTokens[_player][tokenContract].quantity;
            ConfiguredToken memory configuredToken = configuredTokens[tokenContract];
            if (quantity > 0 && configuredToken.active) {
                // We are assuming all tokens have a maximum of 18 decimals and that USD Price is denoted in 1e18
                uint256 deltaDecimal = 10 ** (18 - configuredToken.decimals);
                lockedWeighted +=
                    (deltaDecimal * quantity * configuredToken.usdPrice) /
                    1e18;
            }
        }

        _lockedWeightedValue = lockedWeighted;
    }
```

## Finding 13: Arithmetic Underflow in `LockManager.getLockedWeightedValue` When Token Decimals Exceed 18 Causes Denial of Service

### Synopsis

`LockManager.getLockedWeightedValue` assumes all configured tokens have at most 18 decimals and computes `10 ** (18 - decimals)`. If a configured token has more than 18 decimals, this subtraction underflows and reverts, permanently blocking `unlock`, `lock`, `setLockDuration`, and harvesting for affected players.

### Technical details

In `LockManager.getLockedWeightedValue`, the contract normalizes each locked token's balance to 18 decimals before multiplying by `usdPrice`:

```solidity
// We are assuming all tokens have a maximum of 18 decimals and that USD Price is denoted in 1e18
uint256 deltaDecimal = 10 **
    (18 -
        configuredTokens[configuredTokenContracts[i]].decimals);
```

Because the contract is compiled with Solidity `0.8.25`, checked arithmetic is enabled by default. The `decimals` field in `ConfiguredToken` is of type `uint8`, and `configureToken` does not enforce `_tokenData.decimals <= 18`.

If an ERC20 token with `decimals > 18` (for example, a token with 24 decimals) is configured via `configureToken`—or if a token's configuration is updated with `decimals > 18`—then for any player who holds a non-zero locked balance (`quantity > 0`) of that active token, the expression `18 - configuredTokens[configuredTokenContracts[i]].decimals` underflows and reverts with `Panic(0x11)`.

Furthermore, `getLockedWeightedValue` is invoked along critical state-changing paths across the protocol:
- `LockManager.unlock` -> `AccountManager.forceHarvest` -> `_harvest` -> `getDailySchnibbles` / `BonusManager.getHarvestBonus` / `LandManager.updatePlotMetadata` -> `LockManager.getLockedWeightedValue`.
- `LockManager.lock` and `LockManager.lockOnBehalf` -> `AccountManager.forceHarvest` -> `LockManager.getLockedWeightedValue`.
- `LockManager.setLockDuration` -> `AccountManager.forceHarvest` -> `LockManager.getLockedWeightedValue`.

As a result, once a player locks any amount of an active token configured with `decimals > 18`, every subsequent call to `unlock`, `lock`, or `setLockDuration` for that player reverts, trapping all of the player's locked tokens (including ETH and other standard ERC20 tokens) in `LockManager`.

### Assumptions and preconditions

1. A token with `decimals > 18` is configured and active in `LockManager` via `configureToken`.
2. A registered player locks a non-zero quantity of that token (or already has a non-zero locked quantity when the token is configured with `decimals > 18`).

### Proof of concept

1. A token with `decimals = 24` (`active = true`, `nftCost = 1e24`, `usdPrice = 1e18`) is added via `LockManager.configureToken(tokenAddress, tokenData)`.
2. A registered player (Alice), who has not locked this token yet (`lockedTokens[alice][tokenAddress].quantity == 0`), calls `LockManager.lock(tokenAddress, 1e24)`.
   - During `_lock`, `accountManager.forceHarvest(alice)` calls `getLockedWeightedValue(alice)`.
   - Because `lockedTokens[alice][tokenAddress].quantity` is still `0` before line 386, the `if` branch at lines 475–479 is skipped, so the initial `lock` transaction succeeds and sets `lockedTokens[alice][tokenAddress].quantity = 1e24`.
3. After the lock duration expires, Alice calls `LockManager.unlock(tokenAddress, 1e24)` (or attempts to unlock any other token she previously locked, such as ETH).
4. `unlock` calls `accountManager.forceHarvest(alice)`, which calls `lockManager.getLockedWeightedValue(alice)`.
5. Inside the loop in `getLockedWeightedValue`, `lockedTokens[alice][tokenAddress].quantity > 0` and `configuredTokens[tokenAddress].active` are both `true`.
6. Line 482 evaluates `18 - 24`, which underflows `uint8` and reverts the entire transaction with `Panic(0x11)`. Alice cannot unlock any of her funds.

### Impact assessment

- **Severity:** Medium / High (Denial of Service / Locked Funds).
- **Technical Impact:** `getLockedWeightedValue` unconditionally reverts for any player holding a positive locked balance of an active token with `decimals > 18`.
- **Business Impact:** Players who lock such a token cannot unlock any of their locked assets (even across other tokens), update their lock duration, lock additional tokens, or harvest rewards until an admin deactivates or reconfigures the token.

### Remediation

Either validate in `configureToken` that `_tokenData.decimals <= 18`, or support tokens with `decimals > 18` directly in `getLockedWeightedValue` by dividing by `10 ** (decimals - 18)` when `decimals > 18`.

#### Recommended Fix in `LockManager.getLockedWeightedValue`

```solidity
    function getLockedWeightedValue(
        address _player
    ) external view returns (uint256 _lockedWeightedValue) {
        uint256 lockedWeighted = 0;
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            address tokenContract = configuredTokenContracts[i];
            if (
                lockedTokens[_player][tokenContract].quantity > 0 &&
                configuredTokens[tokenContract].active
            ) {
                uint8 decimals = configuredTokens[tokenContract].decimals;
                uint256 quantity = lockedTokens[_player][tokenContract].quantity;
                uint256 usdPrice = configuredTokens[tokenContract].usdPrice;

                if (decimals <= 18) {
                    uint256 deltaDecimal = 10 ** (18 - decimals);
                    lockedWeighted += (deltaDecimal * quantity * usdPrice) / 1e18;
                } else {
                    uint256 deltaDecimal = 10 ** (decimals - 18);
                    lockedWeighted += (quantity * usdPrice) / (deltaDecimal * 1e18);
                }
            }
        }

        _lockedWeightedValue = lockedWeighted;
    }
```

## Finding 14: Deactivating a Token in `LockManager.getLockedWeightedValue` Excludes Existing Locked Positions and Retroactively Zeroes Unsettled Rewards

### Synopsis

`LockManager.getLockedWeightedValue` only includes tokens where `configuredTokens[token].active` is `true`. Because `active` controls whether a token is currently eligible for new locks, deactivating a token immediately zeroes the weighted USD valuation of existing locked positions, retroactively wiping unsettled Schnibbles and invalidating landlords' plots while tokens remain locked.

### Technical details

In `ILockManager.sol`, the `ConfiguredToken.active` field is documented as *"Boolean indicating if the token is currently active for locking"*, and `LockManager` defines two distinct modifiers: `onlyActiveToken` (used on `lock` and `lockOnBehalf` to restrict new locks) and `onlyConfiguredToken` (whose NatSpec explicitly notes *"Token supplied must be configured but can be inactive"*).

However, in `LockManager.getLockedWeightedValue` (`src/managers/LockManager.sol`, lines 472–495):

```solidity
if (
    lockedTokens[_player][configuredTokenContracts[i]].quantity >
    0 &&
    configuredTokens[configuredTokenContracts[i]].active
) {
```

Line 478 gates the valuation of a player's already-locked balance on `configuredTokens[configuredTokenContracts[i]].active`. This creates a valuation state inconsistency across the protocol whenever `configureToken` sets `active = false` for a previously active token:

1. **Loss of valuation while funds remain locked:** Players who already locked the token cannot withdraw their principal until `lockedTokens[_player][token].unlockTime` expires (`TokenStillLockedError`), yet their locked balance immediately contributes `0` to `getLockedWeightedValue(_player)`.
2. **Retroactive forfeiture of unsettled rewards:** `AccountManager` accrues Schnibbles lazily between harvests based on `getDailySchnibbles(_caller)`, which computes `lockManager.getLockedWeightedValue(_caller) / 10`, and `BonusManager.getHarvestBonus(_caller)`, which computes the migration bonus from `getLockedWeightedValue(_caller)`. Because `configureToken` cannot checkpoint (`forceHarvest`) all affected users, setting `active = false` retroactively zeroes the reward rate applied to the entire unharvested period since each player's last harvest.
3. **Immediate plot invalidation in `LandManager`:** `LandManager._getNumPlots(_account)` computes available plots as `lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT`. Dropping the token's weighted value to `0` immediately reduces the landlord's plot count even though their tokens are still locked in `LockManager`.

### Assumptions and preconditions

1. A token has been configured as active (`active = true`, `nftCost > 0`, `usdPrice > 0`) via `LockManager.configureToken`.
2. One or more players have locked a non-zero quantity of this token via `lock` or `lockOnBehalf`.
3. The admin subsequently updates the token configuration via `configureToken` with `active = false` to pause or discontinue new locks for that asset.

### Proof of concept

1. Admin configures token `T` with `usdPrice = 100e18`, `nftCost = 1e18`, `decimals = 18`, and `active = true`.
2. Alice locks `100e18` of token `T` at timestamp `t0` with a 30-day lock duration (`unlockTime = t0 + 30 days`). At this point, `getLockedWeightedValue(alice)` returns `10,000e18`, giving Alice `1,000e18` daily Schnibbles and `10,000e18 / PRICE_PER_PLOT` plots in `LandManager`.
3. Alice accrues rewards for 10 days without calling `harvest`.
4. At `t0 + 10 days`, Admin calls `configureToken(T, { usdPrice: 100e18, nftCost: 1e18, decimals: 18, active: false })` so no new locks can be made in token `T`.
5. `getLockedWeightedValue(alice)` now returns `0` because `configuredTokens[T].active` is `false`.
6. When Alice's account is harvested (or when her plots are evaluated in `LandManager`), her daily Schnibbles rate is calculated as `0` for the entire 10-day unharvested period, her available plots drop to `0`, and she still cannot call `unlock(T, 100e18)` for another 20 days until `unlockTime`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Deactivating a token for new locks causes `getLockedWeightedValue` to return `0` for all existing locked positions of that token. This retroactively erases unharvested Schnibbles accrued prior to deactivation, eliminates ongoing Schnibble generation and migration bonuses for the remainder of the mandatory lock period, and immediately reduces landlords' valid plot counts in `LandManager`.
- **Business Impact:** Players suffer permanent loss of earned game rewards (Schnibbles/Munch Points) and disruption of their Land/Snuggery positions while their capital remains time-locked in the contract.

### Remediation

Remove the `configuredTokens[configuredTokenContracts[i]].active` check in `getLockedWeightedValue` so that any positive locked balance of a configured token continues to be valued until the player unlocks it (while `onlyActiveToken` on `lock` and `lockOnBehalf` continues to prevent new locks of inactive tokens).

```solidity
    /// @inheritdoc ILockManager
    function getLockedWeightedValue(
        address _player
    ) external view returns (uint256 _lockedWeightedValue) {
        uint256 lockedWeighted = 0;
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            if (
                lockedTokens[_player][configuredTokenContracts[i]].quantity > 0
            ) {
                // We are assuming all tokens have a maximum of 18 decimals and that USD Price is denoted in 1e18
                uint256 deltaDecimal = 10 **
                    (18 -
                        configuredTokens[configuredTokenContracts[i]].decimals);
                lockedWeighted +=
                    (deltaDecimal *
                        lockedTokens[_player][configuredTokenContracts[i]]
                            .quantity *
                        configuredTokens[configuredTokenContracts[i]]
                            .usdPrice) /
                    1e18;
            }
        }

        _lockedWeightedValue = lockedWeighted;
    }
```
