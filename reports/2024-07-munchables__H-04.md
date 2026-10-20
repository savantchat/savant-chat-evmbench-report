# Audit report — 2024-07-munchables/H-04

## Finding 1: Incorrect `StorageKey` used to configure `PRICE_PER_PLOT` in `LandManager._reconfigure` causes division-by-zero or miscalculated plot counts

### Synopsis

In `LandManager._reconfigure`, `PRICE_PER_PLOT` is loaded from `ConfigStorage` using `StorageKey.NFTOverlord` instead of a dedicated `PricePerPlot` storage key. If `uint` storage for `StorageKey.NFTOverlord` is unset (zero), `_getNumPlots` reverts with division by zero, permanently blocking staking, farming, and plot transfers.

### Technical details

In `src/managers/LandManager.sol`, the internal `_reconfigure()` function loads protocol configuration variables from `configStorage`. At lines 77–79:

```solidity
PRICE_PER_PLOT = IConfigStorage(configStorage).getUint(
    StorageKey.NFTOverlord
);
```

Instead of querying a dedicated configuration key for the plot price (e.g., `StorageKey.PricePerPlot`), `_reconfigure()` queries `StorageKey.NFTOverlord`, which is the key designated for the `NFTOverlord` contract address.

`PRICE_PER_PLOT` is subsequently used as the divisor in `_getNumPlots(address _account)` at line 345:

```solidity
function _getNumPlots(address _account) internal view returns (uint256) {
    return lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT;
}
```

Because `ConfigStorage` stores `NFTOverlord` as an address rather than a `uint256` plot price under `StorageKey.NFTOverlord` in standard configuration, `getUint(StorageKey.NFTOverlord)` returns `0` unless the admin knows about the hardcoded key reuse and manually sets a `uint256` value under `StorageKey.NFTOverlord`. When `PRICE_PER_PLOT == 0`, any call to `_getNumPlots` reverts due to division by zero (Panic `0x12`), which breaks `stakeMunchable`, `transferToUnoccupiedPlot`, and `_farmPlots` (which is also invoked by the `forceFarmPlots` modifier on `unstakeMunchable`), effectively locking staked Munchables in the contract.

### Assumptions and preconditions

1. `ConfigStorage` is configured using standard semantic keys where `StorageKey.NFTOverlord` is only populated in address storage (or `StorageKey.PricePerPlot` is populated in uint storage instead of `StorageKey.NFTOverlord`).
2. `LandManager._reconfigure()` is executed during initialization or via `configUpdated()`.

### Proof of concept

1. The protocol team configures `ConfigStorage` by setting the `NFTOverlord` contract address under `StorageKey.NFTOverlord` using `setAddress`, leaving the `uintStorage` mapping for `StorageKey.NFTOverlord` at its default value of `0`.
2. `LandManager.configUpdated()` is called, triggering `_reconfigure()`, which sets `PRICE_PER_PLOT = 0`.
3. A registered player attempts to call `stakeMunchable(landlord, tokenId, plotId)`.
4. Execution reaches `uint256 totalPlotsAvail = _getNumPlots(landlord);` at line 145, which evaluates `lockManager.getLockedWeightedValue(landlord) / 0` at line 345.
5. Alternatively, a player who already has a staked Munchable attempts to call `unstakeMunchable(tokenId)`, which triggers the `forceFarmPlots(msg.sender)` modifier, calling `_farmPlots(msg.sender)` and hitting `_getNumPlots(landlord)` at line 258.
6. The transaction reverts with a division-by-zero panic (`Panic(0x12)`), preventing staking, farming, and unstaking.

### Impact assessment

- **Severity:** High / Medium
- **Direct Technical Effect:** `PRICE_PER_PLOT` is loaded from the wrong `StorageKey` (`StorageKey.NFTOverlord`). If unset in `uint` storage, `PRICE_PER_PLOT` is `0`, causing every call to `_getNumPlots` to revert with division by zero.
- **Business Implications:** Core `LandManager` functionality (`stakeMunchable`, `unstakeMunchable`, `transferToUnoccupiedPlot`, and `farmPlots`) becomes completely inoperable, and any already-staked Munchable NFTs are frozen in `LandManager` because `unstakeMunchable` triggers `_farmPlots`.

### Remediation

Add and use the appropriate `StorageKey` (e.g., `StorageKey.PricePerPlot`) in `IConfigStorage` and `LandManager._reconfigure()`, and ensure `PRICE_PER_PLOT` cannot be configured to zero.

```solidity
        PRICE_PER_PLOT = IConfigStorage(configStorage).getUint(
            StorageKey.PricePerPlot
        );
```

## Finding 2: Updating `PRICE_PER_PLOT` in `LandManager._reconfigure` without synchronizing plot metadata timestamps causes underflow in `_farmPlots` and locks staked Munchables

### Synopsis

`LandManager._reconfigure` updates `PRICE_PER_PLOT` (and bonus parameters) via `configUpdated` without checkpointing ongoing toil accruals or updating `plotMetadata[landlord].lastUpdated`. Increasing `PRICE_PER_PLOT` reduces `_getNumPlots(landlord)` below occupied `plotId`s, causing `_farmPlots` to use a stale `plotMetadata[landlord].lastUpdated` timestamp, reverting on underflow and freezing staked NFTs.

### Technical details

In `LandManager`, the number of available plots for a landlord is dynamically calculated in `_getNumPlots` (line 345) using the state variable `PRICE_PER_PLOT`:

```solidity
function _getNumPlots(address _account) internal view returns (uint256) {
    return lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT;
}
```

When `configUpdated()` (lines 88–90) is invoked by `configStorage`, it calls `_reconfigure()`, which overwrites `PRICE_PER_PLOT`, `REALM_BONUSES`, and `RARITY_BONUSES` (lines 77–83):

```solidity
PRICE_PER_PLOT = IConfigStorage(configStorage).getUint(
    StorageKey.NFTOverlord
);
REALM_BONUSES = configStorage.getSmallIntArray(StorageKey.RealmBonuses);
RARITY_BONUSES = configStorage.getSmallUintArray(
    StorageKey.RarityBonuses
);
```

In `_farmPlots` (lines 258–261), the contract checks whether the landlord's current number of plots has fallen below the staked Munchable's `_toiler.plotId`:

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

And then calculates the accrued Schnibbles at lines 280–282:

```solidity
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
```

This logic assumes that `_getNumPlots(landlord)` can only decrease when the landlord unlocks tokens (which updates `plotMetadata[landlord].lastUpdated` to the unlock timestamp). However, when `PRICE_PER_PLOT` is increased through `_reconfigure()`, `_getNumPlots(landlord)` immediately decreases for all landlords while `plotMetadata[landlord].lastUpdated` remains unchanged at the timestamp when the landlord last updated their lock/plot metadata.

Consequently, when `_getNumPlots(landlord) < _toiler.plotId` evaluates to `true` after a `PRICE_PER_PLOT` increase:
1. **Permanent DoS / Frozen Staked Munchables:** If the renter staked or farmed at any time after `plotMetadata[landlord].lastUpdated`, then `_toiler.lastToilDate > plotMetadata[landlord].lastUpdated`. Setting `timestamp = plotMetadata[landlord].lastUpdated` causes `timestamp - _toiler.lastToilDate` at line 281 to revert with an arithmetic underflow. Because `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` all apply the `forceFarmPlots(msg.sender)` modifier, every state-changing call by the renter reverts, locking all of the renter's staked Munchable NFTs inside `LandManager`.
2. **Retroactive Loss of Accrued Rewards:** Even if `_toiler.lastToilDate <= plotMetadata[landlord].lastUpdated` (e.g., the landlord updated `plotMetadata` after `_toiler.lastToilDate`, and `PRICE_PER_PLOT` was later increased), `timestamp` is set to the historical `plotMetadata[landlord].lastUpdated` rather than the timestamp at which `PRICE_PER_PLOT` was reconfigured, retroactively erasing all Schnibbles accrued between `plotMetadata[landlord].lastUpdated` and the reconfiguration time. Furthermore, updates to `REALM_BONUSES` and `RARITY_BONUSES` in `_reconfigure()` are applied retroactively across the entire unharvested interval since `_toiler.lastToilDate`.

### Assumptions and preconditions

1. A landlord locks tokens at timestamp $t_1$, setting `plotMetadata[landlord].lastUpdated = t_1` and giving the landlord $N$ plots based on the initial `PRICE_PER_PLOT`.
2. A renter stakes a Munchable on the landlord's high-index plot (or farms existing plots) at timestamp $t_2 > t_1$, setting `toilerState[tokenId].lastToilDate = t_2`.
3. At timestamp $t_3 > t_2$, `ConfigStorage` updates configuration parameters with a higher `PRICE_PER_PLOT` and calls `LandManager.configUpdated()`, reducing `_getNumPlots(landlord)` below `_toiler.plotId`.

### Proof of concept

1. At $t_1 = 1000$, Landlord Alice locks tokens worth $3{,}000\text{ USD}$. With `PRICE_PER_PLOT` $= 30\text{ USD}$, `_getNumPlots(alice)` $= 100$, and `plotMetadata[alice].lastUpdated` $= 1000$.
2. At $t_2 = 2000$, Renter Bob stakes his Munchable NFT (`tokenId = 1`) into Alice's `plotId = 90`. This sets `toilerState[1].lastToilDate = 2000`.
3. At $t_3 = 3000$, the protocol updates `PRICE_PER_PLOT` to $50\text{ USD}$ via `ConfigStorage`, triggering `LandManager.configUpdated()` -> `_reconfigure()`.
4. Now `_getNumPlots(alice)` $= 3000 / 50 = 60$, while `plotMetadata[alice].lastUpdated` is still $1000$.
5. At $t_4 = 4000$, Bob calls `unstakeMunchable(1)` (or `farmPlots()`, `transferToUnoccupiedPlot(1, 0)`, or `stakeMunchable(...)`).
6. The `forceFarmPlots(msg.sender)` modifier executes `_farmPlots(bob)`:
   - For `tokenId = 1`, `_getNumPlots(alice)` ($60$) $< \text{\_toiler.plotId}$ ($90$) is `true`.
   - `timestamp` is set to `plotMetadata[alice].lastUpdated` $= 1000$.
   - Line 281 evaluates `timestamp - _toiler.lastToilDate` $= 1000 - 2000$, which reverts due to arithmetic underflow.
7. Bob cannot unstake, transfer, or farm any of his staked Munchables unless Alice performs an action that updates `plotMetadata[alice].lastUpdated` to a timestamp $\ge 2000$.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Arithmetic underflow in `_farmPlots` causes all calls to `farmPlots`, `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` for affected renters to revert. Even when it does not underflow, rewards accrued between `plotMetadata[landlord].lastUpdated` and the `PRICE_PER_PLOT` update are lost, and reward rate changes are applied retroactively.
- **Business Implications:** Renters' Munchable NFTs become stuck in the `LandManager` contract (directly violating the core protocol invariant against deadlocked Munchables), and users lose validly accrued Schnibbles.

### Remediation

Track historical plot-pricing epochs (or record a `pricePerPlotUpdatedTimestamp` and snapshot the `PRICE_PER_PLOT` value per landlord in `PlotMetadata` when `plotMetadata[landlord]` is initialized/updated), and ensure `_farmPlots` uses `max(plotMetadata[landlord].lastUpdated, lastPricePerPlotUpdateTimestamp)` (bounded by `max(..., _toiler.lastToilDate)`) when capping toil accrual due to plot reductions. Alternatively, snapshot `PRICE_PER_PLOT` in `plotMetadata[landlord]` so that global changes to `PRICE_PER_PLOT` only take effect for a landlord when `updatePlotMetadata` is called and `plotMetadata[landlord].lastUpdated` is refreshed:

```solidity
        uint256 newPricePerPlot = IConfigStorage(configStorage).getUint(
            StorageKey.NFTOverlord
        );
        if (PRICE_PER_PLOT != newPricePerPlot) {
            PRICE_PER_PLOT = newPricePerPlot;
            lastPricePerPlotUpdated = block.timestamp;
        }
        REALM_BONUSES = configStorage.getSmallIntArray(StorageKey.RealmBonuses);
        RARITY_BONUSES = configStorage.getSmallUintArray(
            StorageKey.RarityBonuses
        );

        __BaseBlastManagerUpgradeable_reconfigure();
    }
```

In `_farmPlots`, when `_getNumPlots(landlord) <= _toiler.plotId`, use `uint256 effectiveTimestamp = plotMetadata[landlord].lastUpdated > lastPricePerPlotUpdated ? plotMetadata[landlord].lastUpdated : lastPricePerPlotUpdated;` and clamp `timestamp = effectiveTimestamp > _toiler.lastToilDate ? effectiveTimestamp : _toiler.lastToilDate;`.

## Finding 3: Stale and Out-of-Bounds Landlord Tax Rates Persist After `LandManager.configUpdated` Updates `MIN_TAX_RATE` or `MAX_TAX_RATE`

### Synopsis

In `LandManager`, `MIN_TAX_RATE` and `MAX_TAX_RATE` are only enforced when a landlord calls `updateTaxRate`. When `configUpdated` calls `_reconfigure` to update tax rate bounds, neither `_reconfigure` nor tax-rate consumers validate or clamp existing `plotMetadata[landlord].currentTaxRate` values, allowing out-of-bounds tax rates or completely blocking `updateTaxRate`.

### Technical details

In `src/managers/LandManager.sol`, protocol tax-rate bounds (`MIN_TAX_RATE` and `MAX_TAX_RATE`) are loaded from `ConfigStorage` during `_reconfigure()`, which is invoked via `configUpdated()` (lines 88–90):

```solidity
function configUpdated() external override onlyConfigStorage {
    _reconfigure();
}

function updateTaxRate(uint256 newTaxRate) external override notPaused {
    (address landlord, ) = _getMainAccountRequireRegistered(msg.sender);
    if (newTaxRate < MIN_TAX_RATE || newTaxRate > MAX_TAX_RATE)
        revert InvalidTaxRateError();
    if (plotMetadata[landlord].lastUpdated == 0)
        revert PlotMetadataNotUpdatedError();
    uint256 oldTaxRate = plotMetadata[landlord].currentTaxRate;
    plotMetadata[landlord].currentTaxRate = newTaxRate;
```

This creates an inconsistent configuration state across `LandManager`:

1. **Bounds are only checked at write-time in `updateTaxRate` (lines 94–95):** When `ConfigStorage` updates `MIN_TAX_RATE` or `MAX_TAX_RATE` and triggers `configUpdated()` (lines 88–90), existing landlords' `plotMetadata[landlord].currentTaxRate` (and renters' cached `toilerState[tokenId].latestTaxRate`) are never re-validated or clamped to the new `[MIN_TAX_RATE, MAX_TAX_RATE]` range.
2. **Downstream operations blindly trust `plotMetadata[landlord].currentTaxRate`:**
   - `stakeMunchable` (line 166) assigns `latestTaxRate: plotMetadata[landlord].currentTaxRate` (which is even `0 < MIN_TAX_RATE` if `plotMetadata[landlord]` has never been initialized).
   - `transferToUnoccupiedPlot` (lines 213–214) assigns `toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord].currentTaxRate`.
   - `_farmPlots` (lines 287–293) calculates `schnibblesLandlord = (schnibblesTotal * _toiler.latestTaxRate) / 1e18` and refreshes `toilerState[tokenId].latestTaxRate = plotMetadata[_toiler.landlord].currentTaxRate` without clamping to `[MIN_TAX_RATE, MAX_TAX_RATE]`.
3. **Unvalidated bound ordering on intermediate updates:** Because `ConfigStorage` updates parameters individually and calls `configUpdated()` after each update, an update sequence or misconfiguration where `MIN_TAX_RATE > MAX_TAX_RATE` is accepted by `_reconfigure()`, causing `if (newTaxRate < MIN_TAX_RATE || newTaxRate > MAX_TAX_RATE)` on line 94 to revert with `InvalidTaxRateError` for every possible `newTaxRate`.

### Assumptions and preconditions

1. Landlords have initialized `plotMetadata` and set `currentTaxRate` within the initial `[MIN_TAX_RATE, MAX_TAX_RATE]` range (or a landlord has not initialized `plotMetadata`, leaving `currentTaxRate == 0`).
2. `ConfigStorage` updates `MIN_TAX_RATE` or `MAX_TAX_RATE` and invokes `LandManager.configUpdated()`.

### Proof of concept

1. Suppose initially `MIN_TAX_RATE = 0.1e18` (10%) and `MAX_TAX_RATE = 0.5e18` (50%).
2. Landlord Alice calls `updateTaxRate(0.5e18)` (lines 92–99), setting `plotMetadata[alice].currentTaxRate = 0.5e18`.
3. Landlord Bob calls `updateTaxRate(0.1e18)`, setting `plotMetadata[bob].currentTaxRate = 0.1e18`.
4. Later, the protocol updates the allowed tax range via `ConfigStorage` to `MIN_TAX_RATE = 0.2e18` (20%) and `MAX_TAX_RATE = 0.3e18` (30%), triggering `LandManager.configUpdated()` -> `_reconfigure()` (lines 88–90).
5. Any landlord who now calls `updateTaxRate` is restricted by line 94 to `[0.2e18, 0.3e18]`.
6. However, Alice and Bob simply do not call `updateTaxRate`. When renters stake (`stakeMunchable` line 166), transfer plots (`transferToUnoccupiedPlot` line 213), or farm (`_farmPlots` lines 287–293), Alice continues extracting a 50% tax rate (`> MAX_TAX_RATE`) from renters, while Bob continues offering a 10% tax rate (`< MIN_TAX_RATE`) that no other landlord is allowed to set.
7. Furthermore, if `MIN_TAX_RATE` is raised above the current `MAX_TAX_RATE` in a first configuration call before `MAX_TAX_RATE` is raised, `newTaxRate < MIN_TAX_RATE || newTaxRate > MAX_TAX_RATE` (line 94) evaluates to `true` for all `uint256` values, completely bricking `updateTaxRate`.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** `plotMetadata[landlord].currentTaxRate` and `toilerState[tokenId].latestTaxRate` bypass the protocol's `[MIN_TAX_RATE, MAX_TAX_RATE]` invariant whenever tax rate bounds are reconfigured via `configUpdated()`, or when `plotMetadata` is uninitialized (`currentTaxRate == 0`).
- **Business Impact:** Renters can be overcharged above `MAX_TAX_RATE` during `_farmPlots`, landlords with legacy rates below `MIN_TAX_RATE` gain a permanent unmatchable tax advantage, and intermediate configuration updates where `MIN_TAX_RATE > MAX_TAX_RATE` cause `updateTaxRate` to revert unconditionally.

### Remediation

Validate the tax rate bounds during `_reconfigure()` and clamp the effective landlord tax rate to `[MIN_TAX_RATE, MAX_TAX_RATE]` whenever it is read or applied.

```solidity
    function _getEffectiveTaxRate(address landlord) internal view returns (uint256) {
        if (plotMetadata[landlord].lastUpdated == 0) {
            return DEFAULT_TAX_RATE;
        }
        uint256 rate = plotMetadata[landlord].currentTaxRate;
        if (rate < MIN_TAX_RATE) return MIN_TAX_RATE;
        if (rate > MAX_TAX_RATE) return MAX_TAX_RATE;
        return rate;
    }

    function configUpdated() external override onlyConfigStorage {
        _reconfigure();
        if (
            MIN_TAX_RATE > MAX_TAX_RATE ||
            DEFAULT_TAX_RATE < MIN_TAX_RATE ||
            DEFAULT_TAX_RATE > MAX_TAX_RATE ||
            MAX_TAX_RATE > 1e18
        ) revert InvalidTaxRateError();
    }

    function updateTaxRate(uint256 newTaxRate) external override notPaused {
        (address landlord, ) = _getMainAccountRequireRegistered(msg.sender);
        if (newTaxRate < MIN_TAX_RATE || newTaxRate > MAX_TAX_RATE)
            revert InvalidTaxRateError();
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();
        uint256 oldTaxRate = _getEffectiveTaxRate(landlord);
        plotMetadata[landlord].currentTaxRate = newTaxRate;
        emit TaxRateChanged(landlord, oldTaxRate, newTaxRate);
    }
```

## Finding 4: Tax Snapshot and Effective-Time Mismatch in `LandManager.updateTaxRate` Delays or Bypasses Tax Rate Changes for Staked Munchables

### Synopsis

`LandManager.updateTaxRate` updates `plotMetadata[landlord].currentTaxRate` without checkpointing ongoing toil accruals for Munchables staked on the landlord's plots. Because `_farmPlots` applies the renter's cached `toilerState[tokenId].latestTaxRate` across the entire `[lastToilDate, timestamp]` interval before refreshing the snapshot, tax rate changes do not take effect at the time `updateTaxRate` is called.

### Technical details

In `LandManager`, each staked Munchable tracks a per-token tax snapshot in `toilerState[tokenId].latestTaxRate`, initialized at stake time (`stakeMunchable`, line 166) to `plotMetadata[landlord].currentTaxRate`.

When a landlord updates their tax rate via `updateTaxRate` (lines 92–99):
```solidity
function updateTaxRate(uint256 newTaxRate) external override notPaused {
    (address landlord, ) = _getMainAccountRequireRegistered(msg.sender);
    if (newTaxRate < MIN_TAX_RATE || newTaxRate > MAX_TAX_RATE)
        revert InvalidTaxRateError();
    if (plotMetadata[landlord].lastUpdated == 0)
        revert PlotMetadataNotUpdatedError();
    uint256 oldTaxRate = plotMetadata[landlord].currentTaxRate;
    plotMetadata[landlord].currentTaxRate = newTaxRate;
```
the function only overwrites `plotMetadata[landlord].currentTaxRate` and does not settle accrued Schnibbles for tokens currently toiling on `landlord`'s plots or record when the rate change occurred.

Later, when `_farmPlots` executes (lines 280–293):
```solidity
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
```
the entire reward accrued between `_toiler.lastToilDate` and `timestamp` is taxed at `_toiler.latestTaxRate` (the old snapshot taken at the renter's previous toil checkpoint), and `toilerState[tokenId].latestTaxRate` is only updated to `plotMetadata[landlord].currentTaxRate` at `timestamp`.

This creates a mismatch between the timestamp at which `updateTaxRate` is called ($t_1$) and the timestamp at which `newTaxRate` actually becomes effective ($t_2$, when the renter next triggers `_farmPlots`):
1. **Tax Increase Bypass:** If a landlord increases `currentTaxRate` at $t_1$, any renter who staked at $t_0 < t_1$ and waits until $t_2 > t_1$ to call `farmPlots()` or `unstakeMunchable()` pays the lower `oldTaxRate` over the post-change period $[t_1, t_2]$. If the renter calls `unstakeMunchable()` at $t_2$, `newTaxRate` is never applied to their toil at all.
2. **Tax Decrease Overcharge:** Conversely, if a landlord decreases `currentTaxRate` at $t_1$, existing renters who do not immediately call `farmPlots()` at $t_1$ continue to be charged the higher `oldTaxRate` for the entire interval $[t_1, t_2]$ until their next farm interaction.

### Assumptions and preconditions

1. A landlord has locked tokens and initialized `plotMetadata[landlord]` with an initial tax rate `oldTaxRate`.
2. A renter has staked at least one Munchable on the landlord's plot at time $t_0$.
3. The landlord calls `updateTaxRate(newTaxRate)` at time $t_1 > t_0$ with `newTaxRate != oldTaxRate`.

### Proof of concept

1. At $t_0 = 1000$, Alice (landlord) has `currentTaxRate = 0.10e18` (10%). Bob (renter) stakes `tokenId = 1` on Alice's plot 0. `toilerState[1].lastToilDate = 1000` and `toilerState[1].latestTaxRate = 0.10e18`.
2. At $t_1 = 2000$, Alice increases her tax rate by calling `updateTaxRate(0.30e18)` (30%). `plotMetadata[alice].currentTaxRate` becomes `0.30e18`.
3. Bob does not call `farmPlots()` at $t_1$ and continues toiling until $t_2 = 10000$, at which point Bob calls `unstakeMunchable(1)`.
4. `unstakeMunchable(1)` triggers `forceFarmPlots(bob)` -> `_farmPlots(bob)` at $t_2 = 10000$.
5. `_farmPlots` computes `schnibblesTotal` for the full $10000 - 1000 = 9000$ seconds, and computes `schnibblesLandlord = (schnibblesTotal * 0.10e18) / 1e18`.
6. Even though Alice's tax rate was 30% for the 8000 seconds in $[2000, 10000]$, Alice only receives 10% tax for that entire period, losing two-thirds of her expected tax revenue during $[2000, 10000]$.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** Accrual calculations across tax rate transitions apply the pre-transition `latestTaxRate` snapshot to time elapsed after `updateTaxRate` was executed, until each renter next invokes `_farmPlots`.
- **Business Impact:** Landlords lose accrued Schnibbles when increasing tax rates if renters delay farming or unstake directly, and renters are overtaxed after tax rate reductions unless they immediately checkpoint their plots when the landlord updates the tax rate.

### Remediation

Track a cumulative tax-weighted time integral (or checkpoint active toilers on the landlord's occupied plots when `updateTaxRate` is called) so that the old tax rate applies strictly to `[lastToilDate, taxChangeTimestamp]` and `newTaxRate` applies to `[taxChangeTimestamp, timestamp]`. For example, using a cumulative tax integral per landlord:

```solidity
    // Track cumulative (timestamp_delta * taxRate) per landlord
    mapping(address => uint256) public cumulativeTaxIntegral;
    mapping(address => uint256) public lastTaxIntegralTimestamp;
    mapping(uint256 => uint256) public toilerLastTaxIntegral;

    function _updateTaxIntegral(address landlord) internal returns (uint256 integral) {
        integral = cumulativeTaxIntegral[landlord];
        uint256 lastTime = lastTaxIntegralTimestamp[landlord];
        if (lastTime != 0 && block.timestamp > lastTime) {
            integral += (block.timestamp - lastTime) * plotMetadata[landlord].currentTaxRate;
            cumulativeTaxIntegral[landlord] = integral;
        }
        lastTaxIntegralTimestamp[landlord] = block.timestamp;
    }

    function updateTaxRate(uint256 newTaxRate) external override notPaused {
        (address landlord, ) = _getMainAccountRequireRegistered(msg.sender);
        if (newTaxRate < MIN_TAX_RATE || newTaxRate > MAX_TAX_RATE)
            revert InvalidTaxRateError();
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();
        _updateTaxIntegral(landlord);
        uint256 oldTaxRate = plotMetadata[landlord].currentTaxRate;
        plotMetadata[landlord].currentTaxRate = newTaxRate;
        emit TaxRateChanged(landlord, oldTaxRate, newTaxRate);
    }
```

Then in `_farmPlots`, compute `schnibblesLandlord` using the average tax rate over `[lastToilDate, timestamp]` derived from `(currentIntegral - toilerLastTaxIntegral[tokenId]) / (timestamp - _toiler.lastToilDate)`.

## Finding 5: Unsettled Accrual and Stale Per-Toiler Tax Snapshots in `LandManager.updateTaxRate` Allow Bypassing or Retroactively Misapplying Tax Rate Changes

### Synopsis

`LandManager.updateTaxRate` updates `plotMetadata[landlord].currentTaxRate` in place without settling accrued schnibbles for existing renters or checkpointing the rate change. Because `ToilerState.latestTaxRate` is only refreshed when a renter triggers `_farmPlots`, tax rate updates fail to take effect at the time of the change.

### Technical details

In `src/managers/LandManager.sol`, each landlord's current tax rate is stored in `plotMetadata[landlord].currentTaxRate`, while each staked Munchable tracks its own snapshot of the landlord's tax rate in `toilerState[tokenId].latestTaxRate` along with the last settlement timestamp in `toilerState[tokenId].lastToilDate` (defined in `ILandManager.ToilerState`).

When a landlord calls `updateTaxRate(uint256 newTaxRate)` (lines 92–101), the function validates `newTaxRate` and `plotMetadata[landlord].lastUpdated`, and then directly overwrites the shared rate:

```solidity
uint256 oldTaxRate = plotMetadata[landlord].currentTaxRate;
plotMetadata[landlord].currentTaxRate = newTaxRate;
```

However, `updateTaxRate` does not settle (`_farmPlots`) the active positions staked on the landlord's plots, nor does `LandManager` maintain historical tax-rate checkpoints or cumulative integral accumulators.

In `LandManager`, a renter's rewards and the landlord's tax share for the entire interval `[toilerState[tokenId].lastToilDate, timestamp]` are calculated inside `_farmPlots` using the per-position snapshot `toilerState[tokenId].latestTaxRate`. Only after computing and distributing schnibbles for that entire interval does `_farmPlots` update `toilerState[tokenId].latestTaxRate = plotMetadata[landlord].currentTaxRate`. Since `_farmPlots` is only invoked for `msg.sender` (the renter) when the renter calls `farmPlots`, `stakeMunchable`, or `unstakeMunchable`, changing `plotMetadata[landlord].currentTaxRate` in `updateTaxRate` creates a divergence between the landlord's active `currentTaxRate` and every existing renter's `latestTaxRate`:

1. **Tax Increase Bypass by Renters:** If a landlord increases `currentTaxRate` (e.g., from `MIN_TAX_RATE` to `MAX_TAX_RATE`), any currently staked renter who refrains from calling `farmPlots` (or staking/unstaking) continues to accrue rewards at the old, lower `latestTaxRate` across the entire post-update period until their first settlement call. When they finally call `unstakeMunchable` (or `farmPlots`), the entire elapsed duration—including all time after `updateTaxRate`—is taxed at the old `latestTaxRate`.
2. **Retroactive Overcharging on Tax Decreases:** Conversely, if a landlord lowers `currentTaxRate`, renters who did not call `farmPlots` immediately prior to or after the update continue to be charged the old, higher `latestTaxRate` for all time accrued since `lastToilDate` until they trigger `_farmPlots`. If the landlord later raises `currentTaxRate` back up before the renter calls `farmPlots`, the lower tax rate period is completely skipped for that renter.

### Assumptions and preconditions

1. Both the landlord and the renter are registered players in `AccountManager`, and the landlord has locked tokens in `LockManager` so `plotMetadata[landlord].lastUpdated != 0`.
2. At least one renter has staked a Munchable on the landlord's plot via `stakeMunchable`.
3. The landlord calls `updateTaxRate(newTaxRate)` to change their tax rate between `MIN_TAX_RATE` and `MAX_TAX_RATE`.

### Proof of concept

1. **Initial Setup ($t = t_0$):**
   - Landlord Alice locks tokens in `LockManager`, initializing `plotMetadata[Alice]` with `lastUpdated = t_0` and `currentTaxRate = DEFAULT_TAX_RATE` (e.g., 10% / `10e16`).
   - Renter Bob calls `stakeMunchable(Alice, tokenId, 0)`. This initializes `toilerState[tokenId]` with `lastToilDate = t_0` and `latestTaxRate = 10e16`.
2. **Tax Rate Update ($t = t_1$):**
   - Immediately or shortly after at $t_1$, Alice calls `updateTaxRate(50e16)` to raise her tax rate to 50%.
   - `plotMetadata[Alice].currentTaxRate` is updated to `50e16`, while Bob's `toilerState[tokenId].latestTaxRate` remains `10e16`.
3. **Continued Farming Without Settlement ($t_1 \to t_2$):**
   - Bob does not call `farmPlots`, `stakeMunchable`, or `unstakeMunchable` during the interval $[t_1, t_2]$.
4. **Settlement ($t = t_2$):**
   - At $t_2$ (e.g., weeks later), Bob calls `unstakeMunchable(tokenId)`, which invokes `_farmPlots(Bob)`.
   - `_farmPlots` calculates `totalSchnibbles` for the entire duration $t_2 - t_0$ and computes Alice's tax share using `toilerState[tokenId].latestTaxRate` (`10e16`) instead of applying `50e16` for the period $t_2 - t_1$.
   - Alice loses 80% of the tax schnibbles she was owed for the interval $[t_1, t_2]$, and Bob retains them.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** Schnibble reward distribution between landlords and renters across tax rate updates is miscalculated whenever renters do not manually settle their positions at the exact timestamp of each `updateTaxRate` transaction.
- **Business Implications:** Renters can completely bypass a landlord's tax rate increase for the remainder of their staking duration by withholding calls to `farmPlots` until they unstake. Conversely, honest renters who do not constantly monitor and settle around landlord tax rate reductions are overcharged at stale higher tax rates.

### Remediation

To ensure tax rate changes apply accurately from the exact timestamp of `updateTaxRate` without requiring unbounded loops over all renters, maintain a cumulative time-weighted tax rate integral per landlord (or settle active renters on rate changes).

Using a cumulative time-weighted tax accumulator per landlord:
1. Track `cumulativeTaxTime` and `lastTaxUpdateTime` in `PlotMetadata` (or a dedicated mapping), updating `cumulativeTaxTime += currentTaxRate * (block.timestamp - lastTaxUpdateTime)` whenever `updatePlotMetadata`, `triggerPlotMetadata`, or `updateTaxRate` is called.
2. In `_farmPlots`, compute the exact time-weighted average tax rate over `[lastToilDate, timestamp]` from the difference in the landlord's cumulative tax accumulator snapshots rather than relying on a single point-in-time `latestTaxRate`.

```solidity
    function updateTaxRate(uint256 newTaxRate) external override notPaused {
        (address landlord, ) = _getMainAccountRequireRegistered(msg.sender);
        if (newTaxRate < MIN_TAX_RATE || newTaxRate > MAX_TAX_RATE)
            revert InvalidTaxRateError();
        if (plotMetadata[landlord].lastUpdated == 0)
            revert PlotMetadataNotUpdatedError();
        _checkpointLandlordTax(landlord);
        uint256 oldTaxRate = plotMetadata[landlord].currentTaxRate;
        plotMetadata[landlord].currentTaxRate = newTaxRate;
        emit TaxRateChanged(landlord, oldTaxRate, newTaxRate);
    }
```

## Finding 6: Overwriting `plotMetadata[landlord].lastUpdated` in `LandManager.updatePlotMetadata` Allows Accruing Rewards on Invalidated Plots Without Locked Funds

### Synopsis

`LandManager.updatePlotMetadata` unconditionally overwrites `plotMetadata[landlord].lastUpdated` with `block.timestamp` on every call. When a landlord unlocks funds and later triggers `updatePlotMetadata` (e.g., via `LockManager.setLockDuration`), the historical timestamp at which plots became invalid is lost, allowing renters and landlords to farm unbacked schnibbles.

### Technical details

In `LandManager`, the number of plots available to a landlord is dynamically computed in `_getNumPlots(landlord)` as `lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT`. When a landlord's locked value decreases such that `_getNumPlots(landlord) < _toiler.plotId`, `_farmPlots` caps the schnibble accrual period for the invalidated plot using `plotMetadata[landlord].lastUpdated` and marks the toiler as `dirty`:

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

This mechanism assumes that `plotMetadata[landlord].lastUpdated` preserves the exact timestamp at which the landlord reduced their locked balance and invalidated the plot.

However, in `LandManager.updatePlotMetadata` (lines 119–129):

```solidity
if (plotMetadata[landlord].lastUpdated == 0) {
    plotMetadata[landlord] = PlotMetadata({
        lastUpdated: block.timestamp,
        currentTaxRate: DEFAULT_TAX_RATE
    });
} else {
    plotMetadata[landlord].lastUpdated = block.timestamp;
}
```

Whenever `AccountManager.forceHarvest` is called by `LockManager` (lines 151–155 of `AccountManager.sol`), it invokes `landManager.updatePlotMetadata(_player)`. In `LockManager`, `accountManager.forceHarvest(msg.sender)` is called not only during `lock` and `unlock`, but also in `setLockDuration` (line 251 of `LockManager.sol`).

Because `updatePlotMetadata` stores only a single `lastUpdated` timestamp per landlord and unconditionally overwrites `plotMetadata[landlord].lastUpdated = block.timestamp` in the `else` branch (line 125) without settling existing toilers or preserving per-plot invalidation timestamps:

1. A landlord can unlock their tokens at timestamp $T_1$, reducing `_getNumPlots(landlord)` to `0` and setting `plotMetadata[landlord].lastUpdated = T_1`.
2. Instead of farming at $T_1$, the renter leaves their Munchables staked on the now-invalid plots while the landlord holds no locked funds in the protocol.
3. At a much later timestamp $T_2 \gg T_1$, the landlord calls `LockManager.setLockDuration` (which succeeds even when the landlord has no locked tokens) or locks a dust amount of tokens.
4. `LockManager` calls `accountManager.forceHarvest(landlord)`, which calls `landManager.updatePlotMetadata(landlord)` and overwrites `plotMetadata[landlord].lastUpdated` to $T_2$.
5. When the renter subsequently calls `farmPlots()`, `_farmPlots` sees `_getNumPlots(landlord) < _toiler.plotId` and sets `timestamp = plotMetadata[landlord].lastUpdated` ($T_2$), crediting both the renter (`renterMetadata.unfedSchnibbles`) and the landlord (`landlordMetadata.unfedSchnibbles`) with full schnibble rewards for the entire $[T_1, T_2]$ period despite no capital being locked during that interval.

### Assumptions and preconditions

1. Both the landlord and renter accounts are registered in `AccountManager`, and contracts are not paused.
2. The landlord initially locks sufficient tokens in `LockManager` to provide plots (`_getNumPlots(landlord) > plotId`), and a renter stakes one or more Munchable NFTs into those plots via `stakeMunchable`.
3. The renter does not call `farmPlots()` between the time the landlord unlocks their tokens ($T_1$) and the time the landlord triggers a subsequent metadata update ($T_2$).

### Proof of concept

1. At $T_0$, Landlord locks tokens in `LockManager` sufficient to create 10 plots (`plotId` `0` through `9`). `updatePlotMetadata(landlord)` initializes `plotMetadata[landlord].lastUpdated = T_0`.
2. At $T_0$, Renter (which may be a secondary account controlled by the Landlord or a colluding player) calls `stakeMunchable(landlord, tokenId, 5)` to stake a Munchable in plot `5`. `toilerState[tokenId].lastToilDate` is set to $T_0$.
3. At $T_1$ (once the lock duration expires), Landlord unlocks all locked tokens. `_getNumPlots(landlord)` drops to `0`, and `updatePlotMetadata(landlord)` sets `plotMetadata[landlord].lastUpdated = T_1`. Landlord now has full custody of their unlocked funds.
4. Time advances to $T_2 = T_1 + 30\text{ days}$. Renter does not call `farmPlots()` during this period.
5. At $T_2$, Landlord calls `LockManager.setLockDuration(duration)`. This invokes `accountManager.forceHarvest(landlord)`, which calls `landManager.updatePlotMetadata(landlord)`. Line 125 executes `plotMetadata[landlord].lastUpdated = T_2`, erasing the $T_1$ checkpoint.
6. Immediately at $T_2$, Renter calls `LandManager.farmPlots()` (or `unstakeMunchable(tokenId)`).
7. Inside `_farmPlots`, `_getNumPlots(landlord) < _toiler.plotId` (`0 < 5`) evaluates to `true`. `timestamp` is set to `plotMetadata[landlord].lastUpdated` ($T_2$), and `schnibblesTotal` is calculated over `T_2 - T_0` instead of `T_1 - T_0`. Both Renter and Landlord receive 30 days of unbacked schnibbles.

### Impact assessment

- **Severity:** High / Medium
- **Technical Impact:** Historical plot-invalidation checkpoints in `plotMetadata[landlord].lastUpdated` are overwritten on any subsequent `LockManager` interaction, allowing `LandManager._farmPlots` to accrue schnibbles past the timestamp when the backing plots ceased to exist.
- **Business Impact:** Landlords and renters can collude (or a single user operating two accounts can self-rent) to earn full schnibble rewards and landlord taxes indefinitely without keeping any capital locked in `LockManager`, inflating reward distribution and diluting honest participants who keep their assets locked.

### Remediation

Track the list of active toilers (or occupied plots) per landlord so that when `updatePlotMetadata` is called (or when plots are invalidated), reward accrual for any out-of-bounds plots is settled or their exact invalidation timestamp is recorded before updating `plotMetadata[landlord].lastUpdated`. Alternatively, record the timestamp at which each plot count threshold was last valid (or settle all toilers on the landlord's plots whenever `updatePlotMetadata` is called).

```solidity
    function updatePlotMetadata(
        address landlord
    ) external override onlyConfiguredContract(StorageKey.AccountManager) {
        if (plotMetadata[landlord].lastUpdated == 0) {
            plotMetadata[landlord] = PlotMetadata({
                lastUpdated: block.timestamp,
                currentTaxRate: DEFAULT_TAX_RATE
            });
        } else if (_getNumPlots(landlord) > 0) {
            plotMetadata[landlord].lastUpdated = block.timestamp;
        }

        emit UpdatePlotsMeta(landlord);
    }
```

*(Note: For a complete fix across multiple partial unlocks, `LandManager` should either force-farm affected toilers when the landlord's plot count decreases or record the invalidation timestamp per plot ID when plots are removed.)*

## Finding 7: Lack of Account Canonicalization for `landlord` in `LandManager.stakeMunchable` and `LandManager.updatePlotMetadata`

### Synopsis

`LandManager.stakeMunchable` and `LandManager.updatePlotMetadata` fail to canonicalize the `landlord` address to its main account via `_getMainAccountRequireRegistered`. This allows players to bypass the self-staking restriction (`CantStakeToSelfError`) using a sub-account as the `landlord` and corrupts player state indexing during plot farming.

### Technical details

In the Munchables architecture, a registered player (`mainAccount`) can link multiple `subAccounts` in `AccountManager`. When interacting with `LandManager`, `stakeMunchable` resolves `msg.sender` to its canonical `mainAccount` at line 136:

```solidity
(address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
if (landlord == mainAccount) revert CantStakeToSelfError();
```

However, `stakeMunchable` does not resolve the `landlord` parameter to its canonical main account (nor does `updatePlotMetadata` at lines 116–129 when invoked via `AccountManager.forceHarvest`). This creates three interconnected issues:

1. **Bypass of Self-Staking Restriction**: Because `LockManager` records locked balances per `msg.sender` and `AccountManager.forceHarvest` checks registration via `getPlayer(_player)` (which succeeds for sub-accounts of registered players) before calling `landManager.updatePlotMetadata(_player)`, a sub-account can hold locked funds, valid plot metadata, and available plots (`_getNumPlots(landlord) > 0`). When the player calls `stakeMunchable` with `landlord` set to their own sub-account, `landlord == mainAccount` evaluates to `false`, bypassing `CantStakeToSelfError`.
2. **Mismatched State Indexing in `_farmPlots`**: When `_farmPlots` processes a token staked to a sub-account `landlord`, line 265 calls `_getMainAccountRequireRegistered(landlord)`, which retrieves the `Player` struct of the landlord's `mainAccount`. However, line 300 calls `accountManager.updatePlayer(landlord, landlordMetadata)` with the un-canonicalized `landlord` (sub-account) address. This writes the main account's `Player` struct into `players[subAccount]` in `AccountManager`, causing the landlord's main account to lose its accrued `schnibblesLandlord` rewards and inadvertently marking `players[subAccount].registrationDate != 0`.
3. **Asymmetric Registration Validation**: If `landlord` is an unregistered address that locked tokens prior to registering, `stakeMunchable` allows staking onto that `landlord`'s plots without verifying registration, causing `_farmPlots` to later revert with `PlayerNotRegisteredError` when `_getMainAccountRequireRegistered(landlord)` is called, blocking `forceFarmPlots` on `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot`.

### Assumptions and preconditions

- A player is registered in `AccountManager` and has linked a sub-account via `addSubAccount`, or a user locks tokens in `LockManager` from an address that is either a sub-account or not yet registered in `AccountManager`.

### Proof of concept

1. Alice registers as a player in `AccountManager` (`mainAccount = Alice`) and registers `AliceSub` as her sub-account via `AccountManager.addSubAccount(AliceSub)`.
2. Alice locks tokens in `LockManager` from `AliceSub`. `LockManager` calls `accountManager.forceHarvest(AliceSub)`.
3. In `AccountManager.forceHarvest(AliceSub)`, `this.getPlayer(AliceSub)` resolves to `(Alice, players[Alice])`, which has `registrationDate != 0`. Consequently, `landManager.updatePlotMetadata(AliceSub)` is executed, initializing `plotMetadata[AliceSub]`.
4. Alice calls `LandManager.stakeMunchable(AliceSub, tokenId, 0)` from `Alice`.
5. In `stakeMunchable`:
   - `_getMainAccountRequireRegistered(Alice)` returns `mainAccount = Alice`.
   - The check `if (landlord == mainAccount)` compares `AliceSub == Alice`, which is `false`, bypassing `CantStakeToSelfError`.
   - `_getNumPlots(AliceSub)` returns the plots corresponding to `AliceSub`'s locked tokens in `LockManager`.
   - Alice's Munchable NFT is successfully staked to her own sub-account `AliceSub`.
6. When `_farmPlots` is later triggered for Alice:
   - `_getMainAccountRequireRegistered(AliceSub)` returns `landlordMetadata = players[Alice]`.
   - `accountManager.updatePlayer(AliceSub, landlordMetadata)` writes `players[Alice]`'s state (including `registrationDate` and `schnibblesLandlord`) to `players[AliceSub]`, while `accountManager.updatePlayer(Alice, renterMetadata)` overwrites `players[Alice]` with only `renterMetadata`.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Players can circumvent the self-staking prohibition (`CantStakeToSelfError`), corrupt `AccountManager.players` storage by cloning main account `Player` structs into sub-account storage slots during `_farmPlots`, cause landlords to lose tax rewards when staked via a sub-account address, or brick farming/unstaking (`forceFarmPlots`) if staking to an unregistered landlord address that has locked tokens.

### Remediation

Canonicalize `landlord` to its main account and require it to be registered using `_getMainAccountRequireRegistered` in both `updatePlotMetadata` and `stakeMunchable`.

```solidity
    function updatePlotMetadata(
        address landlord
    ) external override onlyConfiguredContract(StorageKey.AccountManager) {
        (address landlordMainAccount, ) = _getMainAccountRequireRegistered(landlord);
        if (plotMetadata[landlordMainAccount].lastUpdated == 0) {
            plotMetadata[landlordMainAccount] = PlotMetadata({
                lastUpdated: block.timestamp,
                currentTaxRate: DEFAULT_TAX_RATE
            });
        } else {
            plotMetadata[landlordMainAccount].lastUpdated = block.timestamp;
        }

        emit UpdatePlotsMeta(landlordMainAccount);
    }

    function stakeMunchable(
        address landlord,
        uint256 tokenId,
        uint256 plotId
    ) external override forceFarmPlots(msg.sender) notPaused {
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        (address landlordMainAccount, ) = _getMainAccountRequireRegistered(landlord);
        if (landlordMainAccount == mainAccount) revert CantStakeToSelfError();
        if (plotOccupied[landlordMainAccount][plotId].occupied)
            revert OccupiedPlotError(landlordMainAccount, plotId);
```

## Finding 8: Missing `plotMetadata` Initialization Check in `LandManager.stakeMunchable` Allows Staking Against Uninitialized Landlords, Causing Zero Tax Rates and Permanent NFT Freezing

### Synopsis

`LandManager.stakeMunchable` does not verify that `plotMetadata[landlord].lastUpdated` has been initialized before allowing a renter to stake. When a landlord's `plotMetadata` is uninitialized, staked Munchables accrue rewards at a `0%` tax rate, and if the landlord's plot count drops, `_farmPlots` underflows on `timestamp = 0`, permanently freezing the renter's NFTs.

### Technical details

In `LandManager`, each landlord's plot state is tracked in `plotMetadata[landlord]`, which stores `lastUpdated` and `currentTaxRate`. This struct is initialized via `updatePlotMetadata(landlord)` (called from `AccountManager.forceHarvest` when triggered by `LockManager`) or manually by the landlord via `triggerPlotMetadata()`.

However, a registered landlord can have available plots (`_getNumPlots(landlord) > 0`) while `plotMetadata[landlord].lastUpdated == 0` and `plotMetadata[landlord].currentTaxRate == 0` in two ways:
1. **Locking prior to `LandManager` deployment:** As documented on line 103 (`// Only to be triggered by msg sender if they had locked before the land manager was deployed`), landlords who locked tokens before `LandManager` was deployed have `plotMetadata[landlord].lastUpdated == 0` until they manually invoke `triggerPlotMetadata()`.
2. **Locking prior to registration:** In `AccountManager.forceHarvest` (lines 148–156), `landManager.updatePlotMetadata(_player)` is only called inside `if (player.registrationDate != 0)`. If a user locks tokens in `LockManager` first and calls `AccountManager.register` afterward, `plotMetadata[landlord]` remains uninitialized (`lastUpdated == 0`, `currentTaxRate == 0`).

When a renter calls `LandManager.stakeMunchable(landlord, tokenId, plotId)` (lines 131–171), the function checks whether the plot is occupied and whether `plotId < _getNumPlots(landlord)`, but it never checks that `plotMetadata[landlord].lastUpdated != 0`.

This omission causes two severe failures when uninitialized metadata enters the accrual workflow:
1. **Bypass of `DEFAULT_TAX_RATE` and `MIN_TAX_RATE`:** At line 166, `toilerState[tokenId].latestTaxRate` is set to `plotMetadata[landlord].currentTaxRate`, which is `0`. Throughout subsequent `_farmPlots` executions (line 292), `latestTaxRate` continues to be updated to `0`, depriving the landlord of all tax share (`schnibblesLandlord = 0`).
2. **Arithmetic underflow in `_farmPlots` locking staked NFTs:** If the landlord's available plots decrease such that `_getNumPlots(landlord) < _toiler.plotId` while `plotMetadata[landlord].lastUpdated == 0`, `_farmPlots` (lines 258–261) sets:
   ```solidity
   timestamp = plotMetadata[landlord].lastUpdated; // 0
   ```
   Then at line 281:
   ```solidity
   schnibblesTotal = (timestamp - _toiler.lastToilDate) * BASE_SCHNIBBLE_RATE;
   ```
   Because `timestamp` is `0` and `_toiler.lastToilDate` is a non-zero block timestamp set during `stakeMunchable` (line 163), `0 - _toiler.lastToilDate` reverts with an arithmetic underflow panic in Solidity `0.8.25`. Because `unstakeMunchable`, `transferToUnoccupiedPlot`, and `stakeMunchable` all apply the `forceFarmPlots(msg.sender)` modifier, every attempt by the renter to unstake or move any of their staked Munchables will revert.

### Assumptions and preconditions

1. A landlord has locked tokens before `LandManager` was deployed (without calling `triggerPlotMetadata()`) or locked tokens before registering their account in `AccountManager`.
2. The landlord is registered in `AccountManager` and has sufficient locked value to provide at least one plot (`_getNumPlots(landlord) > plotId`).
3. A registered renter stakes a Munchable NFT onto the landlord's plot via `stakeMunchable`.

### Proof of concept

1. Landlord Alice locks tokens before registering in `AccountManager` (or locked before `LandManager` deployment and has not called `triggerPlotMetadata()`). During `LockManager` lock execution, `AccountManager.forceHarvest(Alice)` sees `player.registrationDate == 0` and skips `landManager.updatePlotMetadata(Alice)`.
2. Alice registers her account via `AccountManager.register(...)`. At this point, `plotMetadata[Alice].lastUpdated == 0` and `plotMetadata[Alice].currentTaxRate == 0`, while `_getNumPlots(Alice) >= 2`.
3. Renter Bob calls `LandManager.stakeMunchable(Alice, tokenId, 1)` at `block.timestamp = T_1 > 0`.
4. `stakeMunchable` succeeds and records `toilerState[tokenId] = { lastToilDate: T_1, plotId: 1, landlord: Alice, latestTaxRate: 0, dirty: false }`.
5. When Bob calls `farmPlots()`, `schnibblesLandlord` evaluates to `(schnibblesTotal * 0) / 1e18 = 0`, giving Alice `0` schnibbles instead of `DEFAULT_TAX_RATE`.
6. If Alice unlocks her tokens while `plotMetadata[Alice].lastUpdated` is still `0` so that `_getNumPlots(Alice)` drops to `0` (`_getNumPlots(Alice) < 1`), Bob calls `unstakeMunchable(tokenId)` to retrieve his NFT.
7. The `forceFarmPlots(msg.sender)` modifier invokes `_farmPlots(Bob)`. Inside `_farmPlots`, `_getNumPlots(Alice) < _toiler.plotId` (`0 < 1`) is true, setting `timestamp = plotMetadata[Alice].lastUpdated = 0`.
8. Line 281 computes `(0 - T_1) * BASE_SCHNIBBLE_RATE`, which reverts due to arithmetic underflow. Bob cannot unstake any of his Munchables.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  1. Renters staking with landlords whose `plotMetadata` is uninitialized pay `0` tax instead of `DEFAULT_TAX_RATE` (violating `MIN_TAX_RATE` and landlord reward accounting).
  2. If the landlord's plot count falls below `plotId` while `plotMetadata[landlord].lastUpdated == 0`, `_farmPlots` permanently reverts with an arithmetic underflow, causing a denial-of-service on `unstakeMunchable`, `transferToUnoccupiedPlot`, and `stakeMunchable` for that renter.
- **Business Implications:** Directly violates the primary protocol invariant stated in `README.md`: *"The biggest concerns are people being able to unstake a different individual's Munchables or deadlocks (ie. where an individual's Munchable is stuck)."*

### Remediation

Require `plotMetadata[landlord].lastUpdated != 0` in `stakeMunchable` (or automatically initialize `plotMetadata[landlord]` if `lastUpdated == 0`) before allowing a renter to stake on a landlord's plot.

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
```

## Finding 9: Off-by-one stake limit check in `LandManager.stakeMunchable` allows staking 11 Munchables instead of 10

### Synopsis

In `LandManager.stakeMunchable`, the maximum staked Munchables check validates `munchablesStaked[mainAccount].length > 10` before pushing the new `tokenId` to the array. This off-by-one boundary error allows players to stake up to 11 Munchables instead of the intended limit of 10, earning extra farming rewards and increasing gas consumption during `_farmPlots`.

### Technical details

When a player stakes a Munchable via `LandManager.stakeMunchable`, the contract enforces a cap on the number of Munchables a single `mainAccount` can have staked simultaneously:

```solidity
if (munchablesStaked[mainAccount].length > 10)
    revert TooManyStakedMunchiesError();
```

Because this check is evaluated prior to appending the new `tokenId` via `munchablesStaked[mainAccount].push(tokenId)` at line 159, a player who currently has 10 Munchables staked (`munchablesStaked[mainAccount].length == 10`) will pass the check (`10 > 10` is `false`). The function then pushes the 11th token into `munchablesStaked[mainAccount]`, bringing the total staked count to 11. The revert `TooManyStakedMunchiesError` is only triggered when attempting to stake a 12th Munchable (`11 > 10`).

Because `_farmPlots` is invoked via the `forceFarmPlots` modifier on every `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` call and iterates through all elements in `munchablesStaked[mainAccount]` (performing multiple external calls per staked token), bounding the array length strictly to the intended maximum is necessary both for game economy fairness (limiting concurrent toiling slots per user to 10) and for keeping loop gas costs within safe bounds.

### Assumptions and preconditions

1. A registered player (`mainAccount`) owns at least 11 Munchable NFTs and has approved `LandManager` to transfer them.
2. One or more landlords have at least 11 unoccupied plots available (`_getNumPlots(landlord)`).
3. The contracts are not paused.

### Proof of concept

1. A registered player `bob` owns 11 Munchable NFTs (`tokenId` 1 through 11) and approves `LandManager` for all of them.
2. `bob` calls `stakeMunchable(landlord, i, i - 1)` for `i` from `1` to `10`. After the 10th call, `munchablesStaked[bob].length` is `10`.
3. `bob` calls `stakeMunchable(landlord, 11, 10)`.
4. At line 140, `munchablesStaked[bob].length > 10` evaluates `10 > 10`, which is `false`, so `TooManyStakedMunchiesError` is not thrown.
5. At line 159, `tokenId = 11` is pushed to `munchablesStaked[bob]`, allowing `bob` to stake an 11th Munchable and farm schnibbles from 11 plots concurrently.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** Players can bypass the intended 10-Munchable staking limit by 1 additional NFT (10% above the cap) and increase the iteration count of `_farmPlots` on every subsequent staking lifecycle action.
- **Business Effect:** Violates the protocol's per-account staking capacity invariant, allowing users to farm 10% more Schnibbles in `LandManager` than intended by the game rules.

### Remediation

Update the boundary check in `LandManager.stakeMunchable` to revert when `munchablesStaked[mainAccount].length >= 10` (or `>= MAX_STAKED_MUNCHABLES`).

```solidity
        if (munchablesStaked[mainAccount].length >= 10)
            revert TooManyStakedMunchiesError();
```

## Finding 10: Sub-Account Identity Mismatch and Delegated Asset-Control Violation in `LandManager.stakeMunchable`

### Synopsis

In `LandManager.stakeMunchable`, `msg.sender` is resolved to `mainAccount` before checking `munchNFT` ownership and approvals and transferring the NFT. This allows a sub-account to stake `mainAccount`'s approved NFTs without ERC721 authorization for `msg.sender`, while preventing sub-accounts from staking NFTs they directly own.

### Technical details

In `LandManager.stakeMunchable` (`src/managers/LandManager.sol`), the function resolves the caller `msg.sender` to their registered `mainAccount` via `_getMainAccountRequireRegistered(msg.sender)` on line 136. However, lines 142–152 perform ERC721 ownership validation, approval checks, and token transfer using `mainAccount` instead of verifying whether `msg.sender` is the `mainAccount` or an authorized ERC721 operator/owner:

```solidity
if (munchNFT.ownerOf(tokenId) != mainAccount)
    revert InvalidOwnerError();

uint256 totalPlotsAvail = _getNumPlots(landlord);
if (plotId >= totalPlotsAvail) revert PlotTooHighError();

if (
    !munchNFT.isApprovedForAll(mainAccount, address(this)) &&
    munchNFT.getApproved(tokenId) != address(this)
) revert NotApprovedError();
munchNFT.transferFrom(mainAccount, address(this), tokenId);
```

This creates an account identity and delegation mismatch between the protocol's sub-account system and ERC721 token control:

1. **Unauthorized Staking of Main-Account Assets by Sub-Accounts:** When `mainAccount` grants approval to `LandManager` (for example, via `munchNFT.setApprovalForAll(address(landManager), true)` to avoid per-token approval transactions, or via `munchNFT.approve(address(landManager), tokenId)`), any registered sub-account of `mainAccount` can invoke `stakeMunchable` (as well as `unstakeMunchable` and then `stakeMunchable`) to pull `mainAccount`'s NFT into `LandManager` and stake it onto an arbitrary `landlord` plot (such as a landlord account controlled by the sub-account with `MAX_TAX_RATE` set), even though `mainAccount` never authorized the sub-account on `munchNFT` and sub-accounts are not intended to have custody/transfer authority over the main account's NFTs.
2. **Inability for Sub-Accounts to Stake Self-Owned NFTs:** Conversely, if a sub-account (`msg.sender != mainAccount`) holds a `MunchNFT` in its own wallet, `munchNFT.ownerOf(tokenId) != mainAccount` always reverts with `InvalidOwnerError`, making any NFT held by a sub-account impossible to stake.

### Assumptions and preconditions

1. A registered player (`mainAccount`) has configured at least one sub-account in `AccountManager`.
2. `mainAccount` owns a `MunchNFT` (`tokenId`) and has approved `LandManager` via `munchNFT.setApprovalForAll(address(landManager), true)` or `munchNFT.approve(address(landManager), tokenId)` (or the NFT is currently staked in `LandManager`, allowing the sub-account to first call `unstakeMunchable` if `setApprovalForAll` is active).
3. A registered `landlord` account with at least one available plot exists.

### Proof of concept

1. Alice (`mainAccount`) registers in `AccountManager`, delegates gameplay access to Bob as a sub-account, and calls `munchNFT.setApprovalForAll(address(landManager), true)` so she can stake her Munchables.
2. Bob registers a separate independent main account `bobLandlord`, locks tokens in `LockManager` to create plots, and sets `bobLandlord`'s tax rate to `MAX_TAX_RATE` via `updateTaxRate`.
3. Using his delegated sub-account address, Bob calls `landManager.stakeMunchable(bobLandlord, tokenId, 0)` (if `tokenId` was already staked to another landlord, Bob first calls `landManager.unstakeMunchable(tokenId)` which returns `tokenId` to Alice, and immediately calls `stakeMunchable`).
4. Inside `stakeMunchable`, `_getMainAccountRequireRegistered(msg.sender)` returns `mainAccount = Alice`.
5. `munchNFT.ownerOf(tokenId) != mainAccount` passes because Alice owns `tokenId`, and `munchNFT.isApprovedForAll(Alice, address(landManager))` passes because Alice approved `LandManager`.
6. `LandManager` executes `munchNFT.transferFrom(Alice, address(this), tokenId)` and stakes Alice's NFT onto `bobLandlord`'s plot with `MAX_TAX_RATE`, allowing Bob to siphon off up to `MAX_TAX_RATE` of the farmed Schnibbles generated by Alice's NFT.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** Sub-accounts bypass the intended delegation boundary and exercise direct staking control over `mainAccount`'s approved ERC721 tokens in `LandManager`, while any NFT held directly by a sub-account address cannot be staked at all.
- **Business Impact:** A malicious or compromised sub-account can unstake and re-stake the main account's Munchables onto a high-tax plot controlled by the attacker, diverting a large portion of the main account's earned Schnibbles to the attacker's landlord account, or lock the NFT's farming progress into an unfavorable plot.

### Remediation

Restrict `stakeMunchable` so that only the `mainAccount` itself can initiate the transfer of its own NFT into `LandManager` (or require that `msg.sender` is the token owner / an approved ERC721 operator for `mainAccount`).

Preferred fix in `src/managers/LandManager.sol`:

```solidity
        if (munchablesStaked[mainAccount].length >= 10)
            revert TooManyStakedMunchiesError();
        if (msg.sender != mainAccount || munchNFT.ownerOf(tokenId) != mainAccount)
            revert InvalidOwnerError();

        uint256 totalPlotsAvail = _getNumPlots(landlord);
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();

        if (
            !munchNFT.isApprovedForAll(mainAccount, address(this)) &&
            munchNFT.getApproved(tokenId) != address(this)
        ) revert NotApprovedError();
        munchNFT.transferFrom(mainAccount, address(this), tokenId);
```

Alternatively, if sub-accounts should only be able to stake `mainAccount`'s NFTs when explicitly approved by `mainAccount` on the `munchNFT` contract, verify `msg.sender == mainAccount || munchNFT.isApprovedForAll(mainAccount, msg.sender)`.

## Finding 11: Missing Counterparty Identity and Registration Validation for `landlord` in `LandManager.stakeMunchable`

### Synopsis

`LandManager.stakeMunchable` does not validate that the `landlord` counterparty is a registered main account before checking available plots and staking the caller's NFT. Staking to an unregistered locker freezes the renter's staked NFTs via `_farmPlots`, while staking to a sub-account corrupts player state and bypasses self-staking checks.

### Technical details

In `LandManager.stakeMunchable`, the caller's identity is resolved and verified using `_getMainAccountRequireRegistered(msg.sender)` on line 136. However, the counterparty address `landlord` is never validated through `_getMainAccountRequireRegistered(landlord)` or checked to ensure it is a registered main account rather than an unregistered address or sub-account.

Instead, `stakeMunchable` directly queries `_getNumPlots(landlord)` on line 145 (`lockManager.getLockedWeightedValue(landlord) / PRICE_PER_PLOT`), marks `plotOccupied[landlord][plotId]` as occupied on line 154, and records `landlord` in `toilerState[tokenId]` on lines 162–168.

This inconsistent counterparty identity validation creates two critical failure modes when `_farmPlots` is subsequently executed (which runs automatically via the `forceFarmPlots(msg.sender)` modifier on `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot`):

1. **Unregistered `landlord` (Permanent/Indefinite NFT Freezing):**
   `LockManager` allows any address to lock tokens before registering in `AccountManager` (as seen in `AccountManager.forceHarvest`, which explicitly checks `if (player.registrationDate != 0)` before harvesting or updating plot metadata). Consequently, an unregistered `landlord` can have `_getNumPlots(landlord) > 0`. When a renter stakes a Munchable to such a `landlord`, `stakeMunchable` succeeds. However, on any subsequent call to `unstakeMunchable`, `stakeMunchable`, `transferToUnoccupiedPlot`, or `farmPlots`, `_farmPlots` executes `_getMainAccountRequireRegistered(landlord)` on line 265, which unconditionally reverts with `PlayerNotRegisteredError()`. Because `unstakeMunchable` is gated by `forceFarmPlots(msg.sender)`, all Munchables staked by the renter become locked until `landlord` registers (or permanently if `landlord` refuses or is a contract incapable of registering).

2. **Sub-Account `landlord` (Self-Staking Bypass and Player State Corruption):**
   Because `LockManager` tracks locks per `msg.sender`, a sub-account that locks tokens has `_getNumPlots(subAccount) > 0`.
   - If the renter passes their own sub-account as `landlord`, `landlord == mainAccount` on line 137 evaluates to `false`, bypassing `CantStakeToSelfError`.
   - When `_farmPlots` runs, `_getMainAccountRequireRegistered(landlord)` on line 265 resolves `landlord` to its `mainAccount` and loads the main account's `Player` struct into `landlordMetadata`, but line 300 calls `accountManager.updatePlayer(landlord, landlordMetadata)` using the unnormalized `landlord` (sub-account) address, writing the main account's `Player` struct (including `unfedSchnibbles` and `registrationDate`) into the sub-account's record in `AccountManager`.

### Assumptions and preconditions

1. An address `landlord` has locked tokens in `LockManager` sufficient to yield at least one plot (`getLockedWeightedValue(landlord) >= PRICE_PER_PLOT`).
2. `landlord` is either not registered in `AccountManager` (`registrationDate == 0`) or is registered as a sub-account of an existing player.
3. A registered renter calls `stakeMunchable(landlord, tokenId, plotId)`.

### Proof of concept

1. An address `L` locks tokens in `LockManager` worth at least `PRICE_PER_PLOT` without registering in `AccountManager`. `AccountManager.forceHarvest(L)` sees `player.registrationDate == 0` and does not revert.
2. A registered player `R` owns `tokenId = 1`, approves `LandManager`, and calls `LandManager.stakeMunchable(L, 1, 0)`.
3. Inside `stakeMunchable`:
   - `forceFarmPlots(R)` runs `_farmPlots(R)` over `R`'s currently empty `munchablesStaked[R]` array and succeeds.
   - `_getNumPlots(L)` at line 145 returns `>= 1`.
   - `munchNFT` transfers `tokenId = 1` to `LandManager`, `munchablesStaked[R].push(1)` is executed, and `toilerState[1].landlord` is set to `L`.
4. Later, `R` attempts to unstake their NFT by calling `LandManager.unstakeMunchable(1)`.
5. The `forceFarmPlots(msg.sender)` modifier executes `_farmPlots(R)` before `unstakeMunchable` body runs.
6. In `_farmPlots(R)`, loop iteration `i = 0` processes `tokenId = 1` with `landlord = L` and calls `_getMainAccountRequireRegistered(L)` on line 265.
7. Because `L` is not registered (`registrationDate == 0`), `_getMainAccountRequireRegistered(L)` reverts with `PlayerNotRegisteredError()`.
8. `R` cannot unstake `tokenId = 1` (or any other NFTs staked by `R`).

### Impact assessment

- **Severity:** High
- **Technical Impact:** Renter NFTs staked to an unregistered `landlord` are frozen in `LandManager` because every state-changing action (`unstakeMunchable`, `stakeMunchable`, `transferToUnoccupiedPlot`, `farmPlots`) triggers `_farmPlots(msg.sender)` and reverts with `PlayerNotRegisteredError()`. Additionally, staking to a sub-account `landlord` bypasses `CantStakeToSelfError` and causes `_farmPlots` to write the main account's `Player` struct into `players[subAccount]` via `accountManager.updatePlayer(landlord, landlordMetadata)`.
- **Business Impact:** Direct loss/freezing of user Munchable NFTs and corruption of player accounting state in `AccountManager`.

### Remediation

Validate and resolve `landlord` to its registered main account via `_getMainAccountRequireRegistered` (and ensure `landlord` cannot be a sub-account or unregistered address) before checking plots and recording state in `stakeMunchable`.

```solidity
    function stakeMunchable(
        address landlord,
        uint256 tokenId,
        uint256 plotId
    ) external override forceFarmPlots(msg.sender) notPaused {
        (address mainAccount, ) = _getMainAccountRequireRegistered(msg.sender);
        (address mainLandlord, ) = _getMainAccountRequireRegistered(landlord);
        if (landlord != mainLandlord) revert PlayerNotRegisteredError();
        if (mainLandlord == mainAccount) revert CantStakeToSelfError();
        if (plotOccupied[mainLandlord][plotId].occupied)
            revert OccupiedPlotError(mainLandlord, plotId);
        if (munchablesStaked[mainAccount].length >= 10)
            revert TooManyStakedMunchiesError();
        if (munchNFT.ownerOf(tokenId) != mainAccount)
            revert InvalidOwnerError();

        uint256 totalPlotsAvail = _getNumPlots(mainLandlord);
        if (plotId >= totalPlotsAvail) revert PlotTooHighError();

        if (
            !munchNFT.isApprovedForAll(mainAccount, address(this)) &&
            munchNFT.getApproved(tokenId) != address(this)
        ) revert NotApprovedError();
        munchNFT.transferFrom(mainAccount, address(this), tokenId);

        plotOccupied[mainLandlord][plotId] = Plot({
            occupied: true,
            tokenId: tokenId
        });

        munchablesStaked[mainAccount].push(tokenId);
        munchableOwner[tokenId] = mainAccount;

        toilerState[tokenId] = ToilerState({
            lastToilDate: block.timestamp,
            plotId: plotId,
            landlord: mainLandlord,
            latestTaxRate: plotMetadata[mainLandlord].currentTaxRate,
            dirty: false
        });

        emit FarmPlotTaken(toilerState[tokenId], tokenId);
    }
```

## Finding 12: Accrual Cutoff Timestamp Underflow and Checkpoint Inconsistency in `LandManager._farmPlots` Bricks Staked NFTs and Corrupts Reward Settlement

### Synopsis

In `LandManager._farmPlots`, when a landlord's available plots drop below `_toiler.plotId`, the accrual cutoff `timestamp` is set to `plotMetadata[landlord].lastUpdated`. Because `plotMetadata[landlord].lastUpdated` can precede `_toiler.lastToilDate`, computing `timestamp - _toiler.lastToilDate` underflows and reverts, freezing all of the renter's staked Munchables.

### Technical details

In `src/managers/LandManager.sol`, `_farmPlots` iterates through all NFTs staked by `mainAccount` (`munchablesStaked[mainAccount]`) and calculates accrued schnibbles for each non-dirty token since `_toiler.lastToilDate`:

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
...
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
...
toilerState[tokenId].lastToilDate = timestamp;
```

This temporal cutoff and checkpoint mechanism has a critical consistency flaw when plot eligibility changes:

1. **Cutoff Timestamp Preceding Last Toil Checkpoint (Underflow / DoS):**
   `_getNumPlots(landlord)` is dynamically computed as `lockManager.getLockedWeightedValue(landlord) / PRICE_PER_PLOT`, whereas `plotMetadata[landlord].lastUpdated` is only updated when `triggerPlotMetadata()` or `updatePlotMetadata(landlord)` is invoked. Consequently, `_toiler.lastToilDate` will be strictly greater than `plotMetadata[landlord].lastUpdated` whenever:
   - A landlord locks tokens at $t_1$ (`plotMetadata[landlord].lastUpdated = t_1`), a renter stakes or farms a Munchable at $t_2 > t_1$ (`_toiler.lastToilDate = t_2`), and `_getNumPlots(landlord)` subsequently drops below `_toiler.plotId` without `updatePlotMetadata(landlord)` being called at or after $t_2$ (e.g., when `LockManager.unlock` reduces the landlord's locked tokens without calling `updatePlotMetadata`, or when `PRICE_PER_PLOT` or token USD prices change in `ConfigStorage`).
   - A landlord unlocks tokens at $t_1$ such that `_getNumPlots(landlord) == _toiler.plotId` (which bypasses the strict `<` check at line 258), the renter calls `farmPlots()` at $t_2 > t_1$ (updating `toilerState[tokenId].lastToilDate = t_2` at line 291), and `_getNumPlots(landlord)` later decreases by 1 due to a price/configuration update while `plotMetadata[landlord].lastUpdated` remains $t_1 < t_2$.

   When `_farmPlots` executes under these conditions, `timestamp` is set to `plotMetadata[landlord].lastUpdated < _toiler.lastToilDate`. The subtraction `timestamp - _toiler.lastToilDate` at line 281 underflows and reverts the entire transaction.

2. **Inconsistent Accrual Cutoff Across Metadata Updates and Checkpoint Updates:**
   Because `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` all apply the `forceFarmPlots(msg.sender)` modifier, an underflow at line 281 on any single staked token blocks the renter from unstaking or managing all of their staked Munchables. Furthermore, if anyone calls the external `updatePlotMetadata(landlord)` function to unblock the renter's account, `plotMetadata[landlord].lastUpdated` is overwritten with the current `block.timestamp`, causing lines 280–282 and 291 to retroactively credit rewards to both the renter and landlord across the entire ineligible period after the plot was actually lost.

### Assumptions and preconditions

- A landlord has registered and locked tokens, setting `plotMetadata[landlord].lastUpdated` at timestamp $t_1$.
- A renter stakes a Munchable on the landlord's plot (or calls `farmPlots()`) at timestamp $t_2 > t_1$, setting `toilerState[tokenId].lastToilDate = t_2`.
- The landlord's available plot count `_getNumPlots(landlord)` falls below `_toiler.plotId` while `plotMetadata[landlord].lastUpdated < t_2` (e.g., via `PRICE_PER_PLOT` or token USD price updates, or unlocking tokens without updating `plotMetadata`).

### Proof of concept

1. At $t = 1000$, Alice (landlord) locks tokens sufficient for 10 plots (`plotId` 0 through 9). `plotMetadata[alice].lastUpdated` is set to `1000`.
2. At $t = 2000$, Bob (renter) calls `stakeMunchable(alice, tokenId, 5)`. Inside `stakeMunchable`, `toilerState[tokenId].lastToilDate` is set to `2000`.
3. At $t = 3000$, Bob calls `farmPlots()`. Since `_getNumPlots(alice)` is `10` (`10 < 5` is `false`), `timestamp` is `3000`, Bob harvests schnibbles for $3000 - 2000 = 1000$ seconds, and line 291 updates `toilerState[tokenId].lastToilDate = 3000`.
4. At $t = 4000$, `_getNumPlots(alice)` drops to `4` (either because Alice unlocks tokens without calling `updatePlotMetadata`, or because `PRICE_PER_PLOT` is increased / token USD price is lowered in `ConfigStorage`), leaving `plotMetadata[alice].lastUpdated` at `1000`.
5. At $t = 5000$, Bob attempts to call `unstakeMunchable(tokenId)` (or `farmPlots()`).
6. The `forceFarmPlots(msg.sender)` modifier invokes `_farmPlots(bob)`:
   - Line 258 evaluates `_getNumPlots(alice) < _toiler.plotId` (`4 < 5`), which is `true`.
   - Line 259 sets `timestamp = plotMetadata[alice].lastUpdated = 1000`.
   - Line 281 evaluates `(timestamp - _toiler.lastToilDate) * BASE_SCHNIBBLE_RATE` = `(1000 - 3000) * BASE_SCHNIBBLE_RATE`, which reverts with an arithmetic underflow (`Panic(0x11)`).
7. Bob is unable to unstake `tokenId` or any other Munchable he has staked in `LandManager`.

### Impact assessment

**Severity: High**

- **Direct Technical Effect:** Arithmetic underflow in `_farmPlots` at `timestamp - _toiler.lastToilDate` causes unconditional reverts across all state-changing user functions in `LandManager` (`farmPlots`, `stakeMunchable`, `unstakeMunchable`, `transferToUnoccupiedPlot`).
- **Business Implications:** Renters cannot withdraw any of their staked Munchable NFTs or claim accrued schnibbles if even one staked plot becomes invalid while `plotMetadata[landlord].lastUpdated < _toiler.lastToilDate`. Conversely, if `updatePlotMetadata(landlord)` is called afterward to resolve the underflow, `plotMetadata[landlord].lastUpdated` advances to `block.timestamp`, allowing the renter and landlord to accrue unauthorized schnibbles for the entire period during which the plot was no longer backed by locked collateral.

### Remediation

Ensure that `plotMetadata[landlord].lastUpdated` is always updated whenever a landlord's locked value changes, clamp the accrual duration so that `timestamp < _toiler.lastToilDate` cannot underflow when configuration parameters change, and update `toilerState[tokenId].lastToilDate` consistently:

```solidity
            uint256 toilDuration = timestamp > _toiler.lastToilDate
                ? timestamp - _toiler.lastToilDate
                : 0;
            schnibblesTotal = toilDuration * BASE_SCHNIBBLE_RATE;
            schnibblesTotal = uint256(
                (int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus)) / 100
            );
            schnibblesLandlord =
                (schnibblesTotal * _toiler.latestTaxRate) /
                1e18;

            toilerState[tokenId].lastToilDate = block.timestamp;
```

## Finding 13: Off-by-One Plot Capacity Check in `LandManager._farmPlots` Allows Renters on Invalidated Boundary Plots to Farm Indefinitely

### Synopsis

In `LandManager._farmPlots`, the capacity boundary check uses a strict inequality (`_getNumPlots(landlord) < _toiler.plotId`) instead of `<=` for zero-indexed plot IDs. Consequently, when a landlord's plot count drops to exactly `_toiler.plotId` (including `0`), the plot is not marked `dirty` and continues accruing schnibbles indefinitely.

### Technical details

Throughout `LandManager`, plot indices (`plotId`) are zero-indexed, meaning a landlord with `N = _getNumPlots(landlord)` total plots has valid `plotId` values in the range `[0, N - 1]`. Both `stakeMunchable` (line 146) and `transferToUnoccupiedPlot` (line 211) enforce this boundary by reverting with `PlotTooHighError` when `plotId >= totalPlotsAvail`:

```solidity
uint256 totalPlotsAvail = _getNumPlots(landlord);
if (plotId >= totalPlotsAvail) revert PlotTooHighError();
```

However, in `_farmPlots` at line 258, the check determining whether a staked Munchable's `plotId` exceeds the landlord's current available plots uses `<` instead of `<=`:

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
```

Because `plotId` is zero-based, when `_getNumPlots(landlord) == _toiler.plotId`, the `plotId` is already outside the valid range `[0, _getNumPlots(landlord) - 1]`. Due to the strict `<` comparison:
1. If a renter is staked at `plotId = 0` and the landlord unlocks all of their locked tokens so that `_getNumPlots(landlord) == 0`, the condition `0 < 0` evaluates to `false`.
2. More generally, whenever a landlord unlocks tokens or token prices drop such that `_getNumPlots(landlord)` decreases to `K`, the renter occupying `plotId = K` (the `K + 1`-th plot) does not trigger the invalidation branch.

As a result, `toilerState[tokenId].dirty` is never set to `true` and `timestamp` is not capped at `plotMetadata[landlord].lastUpdated`. Both the renter and the landlord continue to accrue `unfedSchnibbles` at full rate in every subsequent `_farmPlots` execution despite the underlying plot no longer being backed by locked tokens.

### Assumptions and preconditions

1. A landlord has locked tokens providing at least `K + 1` plots (`_getNumPlots(landlord) >= K + 1`, where `K >= 0`).
2. A renter stakes a Munchable into `plotId = K` on that landlord's land.
3. The landlord's available plot count subsequently decreases to `K` (e.g., the landlord unlocks all tokens so `_getNumPlots(landlord) == 0` while `plotId == 0`, or partially unlocks tokens so `_getNumPlots(landlord) == K`).

### Proof of concept

1. Alice (landlord) locks tokens sufficient to grant 1 plot (`_getNumPlots(alice) == 1`, so `plotId = 0` is the only valid plot).
2. Bob (renter) calls `stakeMunchable(alice, tokenId, 0)` to stake his Munchable in `plotId = 0`.
3. After the lock duration expires, Alice unlocks all of her tokens. Now `_getNumPlots(alice)` returns `0`.
4. Time passes, and Bob calls `farmPlots()`.
5. Inside `_farmPlots`, line 258 checks `_getNumPlots(alice) < _toiler.plotId`, which evaluates `0 < 0` (`false`).
6. Instead of capping rewards at `plotMetadata[alice].lastUpdated` and marking `toilerState[tokenId].dirty = true`, `_farmPlots` uses `timestamp = block.timestamp` and mints new `unfedSchnibbles` to both Bob and Alice.
7. Bob can repeat step 4 indefinitely, farming rewards from `plotId = 0` even though Alice has 0 plots and 0 tokens locked.

### Impact assessment

- **Severity**: Medium / High
- **Technical Effect**: The plot invalidation mechanism fails for the boundary plot `plotId == _getNumPlots(landlord)` (and most notably for `plotId == 0` when a landlord unlocks all funds and has 0 plots remaining).
- **Business Implications**: Renters and landlords can continuously farm unbacked schnibbles without maintaining locked capital in `LockManager`. In particular, a landlord only needs to lock enough value for 1 plot (`plotId = 0`), allow a renter (or an alternate account) to stake in `plotId = 0`, and then unlock all funds once the lock period ends to earn rewards forever with zero locked TVL.

### Remediation

Update the condition in `LandManager._farmPlots` at line 258 to use `<=` (or `<= _toiler.plotId`) so that it matches the zero-indexed plot boundary enforced in `stakeMunchable` and `transferToUnoccupiedPlot`:

```solidity
            if (_getNumPlots(landlord) <= _toiler.plotId) {
                timestamp = plotMetadata[landlord].lastUpdated;
                toilerState[tokenId].dirty = true;
            }
```

## Finding 14: Unnormalized Landlord Address in `LandManager._farmPlots` Causes Player Metadata Read-Write Divergence

### Synopsis

In `LandManager._farmPlots`, the landlord's `Player` metadata is fetched via `_getMainAccountRequireRegistered(landlord)`, which normalizes sub-accounts to their main account, but the returned main account address is discarded and `accountManager.updatePlayer` is called with the raw `landlord` address, corrupting player state and reward accounting.

### Technical details

In `AccountManager`, player state is stored in `mapping(address => MunchablesCommonLib.Player) players` keyed by the player's main account address. Lookups via `AccountManager.getPlayer(address _account)` resolve `_account` through `_getMainAccount(_account)` before reading `players[_caller]`:

```solidity
function getPlayer(
    address _account
)
    external
    view
    returns (
        address _mainAccount,
        MunchablesCommonLib.Player memory _player
    )
{
    address _caller = _getMainAccount(_account);
    _player = players[_caller];
    _mainAccount = _caller;
}
```

However, `AccountManager.updatePlayer(address _account, MunchablesCommonLib.Player calldata _player)` does not normalize `_account` to its main account and instead writes directly to `players[_account]`:

```solidity
function updatePlayer(
    address _account,
    MunchablesCommonLib.Player calldata _player
) external ... {
    players[_account] = _player;
}
```

In `LandManager._farmPlots` (lines 262–265), the contract retrieves the landlord's metadata using `_getMainAccountRequireRegistered(landlord)`, which wraps `accountManager.getPlayer(landlord)`, but explicitly ignores the returned `_mainAccount` address:

```solidity
(
    ,
    MunchablesCommonLib.Player memory landlordMetadata
) = _getMainAccountRequireRegistered(landlord);
```

Then, at line 300, `_farmPlots` writes the updated `landlordMetadata` (including accrued `schnibblesLandlord` and `lastPetMunchable`) back to `AccountManager` using the unnormalized `landlord` address:

```solidity
landlordMetadata.unfedSchnibbles += schnibblesLandlord;
landlordMetadata.lastPetMunchable = uint32(timestamp);
accountManager.updatePlayer(landlord, landlordMetadata);
```

This creates direct identity inconsistencies in reward accounting:
1. **Read-Write Divergence Between Main Account and Sub-Account:** If `landlord` is a sub-account (or becomes linked to a main account), `_getMainAccountRequireRegistered(landlord)` reads `landlordMetadata` from `players[landlordMainAccount]`, while `accountManager.updatePlayer(landlord, landlordMetadata)` writes the updated struct to `players[landlord]`. As a result, `players[landlordMainAccount]` never receives `schnibblesLandlord`, and `players[landlord]` is populated with a full clone of `landlordMainAccount`'s `Player` struct (including its existing `unfedSchnibbles` and `registrationDate`), duplicating the main account's `unfedSchnibbles` balance if the sub-account is later unlinked.
2. **Aliased Renter and Landlord State Overwrite:** Because `renterMetadata` for `mainAccount` is cached in memory prior to the loop (lines 233–236) and written to `players[mainAccount]` after the loop finishes (line 309), if `landlord` and the renter resolve to the same `mainAccount` (for example, via sub-account aliasing since `stakeMunchable` only checks `landlord == mainAccount` without normalizing `landlord`), the final `accountManager.updatePlayer(mainAccount, renterMetadata)` overwrites any landlord rewards written to `players[mainAccount]`.

### Assumptions and preconditions

1. Both the renter and landlord accounts are registered players in `AccountManager`.
2. A Munchable NFT is staked in `LandManager` where `landlord` is a sub-account or resolves via `_getMainAccount(landlord)` to a main account distinct from the raw `landlord` address (or aliased with the renter's `mainAccount`).

### Proof of concept

1. Suppose Alice's main account `aliceMain` has `1,000` `unfedSchnibbles` and a sub-account `aliceSub` mapped to `aliceMain` in `AccountManager.mainAccounts`.
2. A renter Bob has a Munchable staked with `landlord = aliceSub` (or `landlord` is linked as a sub-account to `aliceMain`).
3. After time passes, Bob calls `LandManager.farmPlots()`, which executes `_farmPlots(bob)`.
4. At lines 262–265, `_getMainAccountRequireRegistered(aliceSub)` calls `accountManager.getPlayer(aliceSub)`, which resolves `aliceSub` to `aliceMain` and returns `(aliceMain, players[aliceMain])` with `unfedSchnibbles = 1,000`. The address `aliceMain` is discarded.
5. `schnibblesLandlord` (e.g., `100`) is added to `landlordMetadata.unfedSchnibbles`, making it `1,100`.
6. At line 300, `accountManager.updatePlayer(aliceSub, landlordMetadata)` writes `landlordMetadata` (with `registrationDate != 0` and `unfedSchnibbles = 1,100`) into `players[aliceSub]`.
7. `players[aliceMain].unfedSchnibbles` remains `1,000` (Alice's main account loses the `100` farmed Schnibbles), while `players[aliceSub]` now holds a duplicated `Player` struct with `1,100` `unfedSchnibbles` that becomes directly usable as an independent registered account if `aliceSub` is removed from `aliceMain`'s sub-accounts.

### Impact assessment

- **Severity:** High / Medium
- **Technical Impact:** Reading player metadata from the normalized main account while writing the updated metadata back to the unnormalized `landlord` address causes landlord farming rewards to be lost from the main account, duplicates existing `Player` state (including `unfedSchnibbles`) into the sub-account storage slot, and allows stale memory overwrites if the renter and landlord share the same main account.

### Remediation

Capture the normalized `landlordMainAccount` returned by `_getMainAccountRequireRegistered(landlord)` (and normalize `landlord` when staking), ensure `landlordMainAccount != mainAccount`, and pass `landlordMainAccount` to `accountManager.updatePlayer`:

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

Additionally, normalize `_account` using `_getMainAccount(_account)` inside `AccountManager.updatePlayer` and normalize `landlord` in `LandManager.stakeMunchable`.

## Finding 15: Incorrect `finalBonus` Percentage Calculation in `LandManager._farmPlots` Causes Negative Cast Underflow, Permanent Staking DoS, and Severe Reward Loss

### Synopsis

In `LandManager._farmPlots`, `finalBonus` is applied to `schnibblesTotal` using `(int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)) / 100`. Because `/ 100` divides the entire sum instead of only the bonus term, negative `finalBonus` values produce a negative `int256` that wraps to `~type(uint256).max` when cast to `uint256`, reverting `_farmPlots` and freezing staked NFTs, while non-negative bonuses reduce base rewards by 99%.

### Technical details

In `LandManager._farmPlots`, each staked Munchable's farming reward bonus (`finalBonus`) is computed by summing the signed realm bonus from `REALM_BONUSES` (`int16`) and the rarity bonus from `RARITY_BONUSES` (`uint8` cast to `int8` and `int16`) at lines 270–279:

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

Here, `finalBonus` represents a percentage modifier (for example, `-10` for `-10%`, `0` for `0%`, or `+20` for `+20%`). The base farming reward is then calculated and adjusted by `finalBonus` at lines 280–286:

```solidity
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
schnibblesTotal = uint256(
    (int256(schnibblesTotal) +
        (int256(schnibblesTotal) * finalBonus)) / 100
);
```

Because `int256(schnibblesTotal)` is added directly to `int256(schnibblesTotal) * finalBonus` before dividing the entire sum by `100`, the expression algebraically evaluates to:

$$\text{schnibblesTotal} = \left\lfloor \frac{\text{schnibblesTotal} \times (1 + \text{finalBonus})}{100} \right\rfloor$$

instead of the intended percentage adjustment:

$$\text{schnibblesTotal} = \text{schnibblesTotal} + \left\lfloor \frac{\text{schnibblesTotal} \times \text{finalBonus}}{100} \right\rfloor = \left\lfloor \frac{\text{schnibblesTotal} \times (100 + \text{finalBonus})}{100} \right\rfloor$$

This flaw leads to two critical failure modes:

1. **Negative `finalBonus` causes signed-to-unsigned wraparound and permanent DoS:**
   `REALM_BONUSES` contains negative values (e.g., `-10` or `-5` for incompatible Munchable and Landlord realms), while `RARITY_BONUSES` can be `0` (e.g., for Common or Primordial Munchables). Whenever `finalBonus <= -2` (for example, `finalBonus = -10`), the factor `(1 + finalBonus)` is negative (`-9`). Consequently, `(int256(schnibblesTotal) + (int256(schnibblesTotal) * finalBonus)) / 100` evaluates to a negative `int256` value (as long as `schnibblesTotal * |1 + finalBonus| >= 100`).
   Casting this negative `int256` to `uint256` wraps the value modulo $2^{256}$ to an enormous number close to `type(uint256).max`. Immediately afterward on line 288:
   ```solidity
   schnibblesLandlord = (schnibblesTotal * _toiler.latestTaxRate) / 1e18;
   ```
   Multiplying this near-maximum `uint256` by `_toiler.latestTaxRate` (which is at least `MIN_TAX_RATE`) overflows `uint256` and reverts with `Panic(0x11)`. Because `stakeMunchable`, `unstakeMunchable`, and `transferToUnoccupiedPlot` all execute the `forceFarmPlots(msg.sender)` modifier, any user who stakes a Munchable resulting in `finalBonus <= -2` is permanently unable to unstake any of their staked Munchables, stake additional tokens, transfer plots, or harvest farming rewards.

2. **Non-negative `finalBonus` severely underpays rewards:**
   When `finalBonus >= 0`, the base `schnibblesTotal` is divided by `100` instead of being preserved at `100%`. For instance:
   - If `finalBonus == 0`, the player receives `schnibblesTotal / 100` (only `1%` of the accrued base Schnibbles, losing `99%`).
   - If `finalBonus == 10` (`+10%`), the player receives `(11 * schnibblesTotal) / 100` (`11%` of the accrued base Schnibbles instead of `110%`).

### Assumptions and preconditions

- `LandManager` is initialized and configured with standard protocol parameters (`BASE_SCHNIBBLE_RATE > 0`, `MIN_TAX_RATE > 0`, `REALM_BONUSES` containing negative penalties such as `-10` or `-5` for mismatched realms, and `RARITY_BONUSES` containing `0` for Common/Primordial rarities).
- A registered renter stakes a Munchable on a registered landlord's plot.

### Proof of concept

1. Alice (Landlord) has `snuggeryRealm = Realm.Everfrost` and locked tokens providing available plots with `currentTaxRate = 20e16` (`20%`).
2. Bob (Renter) owns a Common Munchable (`tokenId = 1`) whose `immutableAttributes.realm` has a `-10` (`-10%`) realm bonus against `Realm.Everfrost` in `REALM_BONUSES` and a `0` (`0%`) rarity bonus in `RARITY_BONUSES`.
3. Bob calls `LandManager.stakeMunchable(alice, 1, 0)` at `t = 1000`.
4. At `t = 1100` (`100` seconds later), Bob calls `LandManager.unstakeMunchable(1)` (or `farmPlots()`):
   - `forceFarmPlots(bob)` triggers `_farmPlots(bob)`.
   - `finalBonus` is computed at lines 270–279 as `-10 + 0 = -10`.
   - At lines 280–282, with `BASE_SCHNIBBLE_RATE = 1e15`, `schnibblesTotal = 100 * 1e15 = 1e17`.
   - At lines 283–286:
     `(int256(1e17) + (int256(1e17) * (-10))) / 100 = (-9e17) / 100 = -9e15`.
   - Casting `-9e15` to `uint256` produces `type(uint256).max - 9e15 + 1` ($\approx 1.1579 \times 10^{77}$).
   - At line 288, `schnibblesTotal * _toiler.latestTaxRate` multiplies this value by `20e16`, overflowing `uint256` and reverting the transaction.
5. Bob can never unstake `tokenId = 1` or any other Munchables he has staked in `LandManager`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  - When `finalBonus <= -2` and time has elapsed (`timestamp > _toiler.lastToilDate`), `_farmPlots` always reverts due to `uint256` wraparound followed by multiplication overflow in `schnibblesLandlord` (or if `latestTaxRate` were `0`, corrupts `unfedSchnibbles` with `~2^256` Schnibbles).
  - Because `unstakeMunchable`, `stakeMunchable`, and `transferToUnoccupiedPlot` all depend on `forceFarmPlots(msg.sender)`, the user's staked Munchable NFTs are permanently locked in `LandManager`.
  - When `finalBonus >= 0`, both renters and landlords lose up to `99%` of their earned Schnibbles because the base amount is erroneously divided by `100`.

### Remediation

Divide only the bonus term `(int256(schnibblesTotal) * finalBonus)` by `100` before adding it to `int256(schnibblesTotal)` (or equivalently compute `int256(schnibblesTotal) * (100 + finalBonus) / 100`):

```solidity
            schnibblesTotal = uint256(
                int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus) / 100
            );
```

## Finding 16: Uncheckpointed Plot Eligibility Transitions in `LandManager._farmPlots` Allow Accrual of Unbacked Schnibbles Across Ineligible Periods

### Synopsis

In `LandManager._farmPlots`, schnibble accrual (`(timestamp - _toiler.lastToilDate) * BASE_SCHNIBBLE_RATE`) relies on the landlord's current plot count and a single mutable `plotMetadata[landlord].lastUpdated` timestamp evaluated lazily when the renter farms. If a landlord updates their plots or re-locks tokens after a plot becomes ineligible, the original cutoff timestamp is overwritten or bypassed, minting unbacked schnibbles across the ineligible period.

### Technical details

When a renter stakes a Munchable on a landlord's plot `plotId`, `LandManager._farmPlots` calculates the accrued schnibbles over the interval `_toiler.lastToilDate` to `timestamp`:

```solidity
if (_getNumPlots(landlord) < _toiler.plotId) {
    timestamp = plotMetadata[landlord].lastUpdated;
    toilerState[tokenId].dirty = true;
}
...
schnibblesTotal =
    (timestamp - _toiler.lastToilDate) *
    BASE_SCHNIBBLE_RATE;
...
toilerState[tokenId].lastToilDate = timestamp;
```

This lazy accrual cutoff mechanism assumes that `plotMetadata[landlord].lastUpdated` and `_getNumPlots(landlord)` remain frozen at the exact state of the first event that invalidated `_toiler.plotId`. However, `plotMetadata[landlord].lastUpdated` is a single mutable timestamp shared across all of a landlord's plots and is overwritten whenever `landManager.updatePlotMetadata(landlord)` is invoked—including automatically via `AccountManager.forceHarvest(landlord)` whenever the landlord locks additional tokens in `LockManager`.

This creates two temporal accrual inconsistencies in `280-291` whenever a renter does not call `farmPlots()` immediately after their plot loses eligibility at timestamp $t_1$:

1. **Overwritten cutoff timestamp across multiple plot updates:** Suppose a renter is staked in `plotId = 9` (and another in `plotId = 5`). At $t_1$, the landlord unlocks tokens so `_getNumPlots(landlord)` drops to `7`, making `plotId = 9` ineligible and setting `plotMetadata[landlord].lastUpdated = t_1`. Before the renter on `plotId = 9` farms, at $t_2 > t_1$ the landlord either unlocks more tokens (e.g., dropping `_getNumPlots(landlord)` to `4`) or locks a small amount of tokens (triggering `AccountManager.forceHarvest(landlord)` $\rightarrow$ `landManager.updatePlotMetadata(landlord)`), which overwrites `plotMetadata[landlord].lastUpdated = t_2` while `_getNumPlots(landlord)` remains below `9`. When the renter subsequently calls `farmPlots()`, `timestamp` is set to $t_2$ instead of $t_1$, and `(timestamp - _toiler.lastToilDate) * BASE_SCHNIBBLE_RATE` (`280-282`) credits both the renter and the landlord with schnibbles for the entire ineligible interval $[t_1, t_2]$.
2. **Complete bypass of eligibility loss upon re-locking:** If the landlord unlocks all tokens at $t_1$ (leaving 0 valid plots so `_toiler.plotId` is unbacked during $[t_1, t_2]$) and later re-locks tokens at $t_2$ before the renter calls `_farmPlots` such that `_getNumPlots(landlord) >= _toiler.plotId` again, the condition `_getNumPlots(landlord) < _toiler.plotId` evaluates to `false`. Consequently, `timestamp` remains `block.timestamp`, `toilerState[tokenId].dirty` is never set, and `280-291` settles full schnibble rewards across $[t_1, t_2]$ as if the landlord had never unlocked their funds.

### Assumptions and preconditions

1. A renter has staked a Munchable NFT in a landlord's plot with `_toiler.plotId` at timestamp $t_0$.
2. At $t_1 > t_0$, the landlord reduces their locked tokens so that `_getNumPlots(landlord)` drops below `_toiler.plotId`.
3. The renter does not call `farmPlots()` between $t_1$ and $t_2$.
4. At $t_2 > t_1$, the landlord updates their lock/plot metadata again (either locking tokens, which triggers `AccountManager.forceHarvest` $\rightarrow$ `LandManager.updatePlotMetadata`, or unlocking again and updating plot metadata).

### Proof of concept

1. **Initial Setup ($t_0 = 1000$):**
   - Landlord Alice locks enough tokens to provide 10 plots (`plotId` `0` through `9`).
   - Renter Bob stakes his Munchable NFT (`tokenId = 1`) into `plotId = 8`. `toilerState[1].lastToilDate = 1000`.
2. **Plot Loses Eligibility ($t_1 = 2000$):**
   - Alice unlocks her tokens so that `_getNumPlots(Alice)` drops to `0`, and `plotMetadata[Alice].lastUpdated = 2000`.
   - Bob's plot (`plotId = 8`) is now unbacked and should stop accruing schnibbles at $t_1 = 2000$. Bob does not call `farmPlots()` yet.
3. **Subsequent Landlord Update ($t_2 = 88400$, 1 day later):**
   - **Case A (Partial Lock / Second Update):** Alice locks enough tokens for 1 plot (`_getNumPlots(Alice) = 1`). `LockManager` calls `AccountManager.forceHarvest(Alice)`, which calls `landManager.updatePlotMetadata(Alice)`, overwriting `plotMetadata[Alice].lastUpdated = 88400`.
   - **Case B (Full Re-lock):** Alice re-locks enough tokens for 10 plots (`_getNumPlots(Alice) = 10`) right before Bob farms.
4. **Renter Farms ($t_3 = 88400$):**
   - Bob calls `farmPlots()`.
   - In **Case A**, `_getNumPlots(Alice) < 8` is true, so `timestamp` is set to `plotMetadata[Alice].lastUpdated = 88400` instead of `2000`. At `280-282`, `schnibblesTotal` is calculated over `88400 - 1000 = 87400` seconds instead of `2000 - 1000 = 1000` seconds.
   - In **Case B**, `_getNumPlots(Alice) < 8` is false, so `timestamp = 88400`, `dirty` remains `false`, and `schnibblesTotal` is likewise calculated over the full `87400` seconds.
   - In both cases, Bob and Alice receive 86,400 seconds (1 full day) of unbacked schnibbles during which Alice had no tokens locked to support `plotId = 8`.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** `_farmPlots` fails to enforce the accrual cutoff at the timestamp when a plot actually became ineligible, allowing both renters and landlords to mint unbacked `unfedSchnibbles` (which convert directly into NFT `chonks` and level-ups) during periods when the landlord's tokens were unlocked.
- **Business Impact:** A landlord and renter (or a single user operating two accounts) can unlock the landlord's capital for extended periods to earn yield elsewhere, then briefly re-lock (or trigger a small lock update) right before calling `farmPlots()`, capturing 100% of the land-staking schnibble rewards and landlord tax rewards without keeping capital locked in the protocol.

### Remediation

Track plot eligibility checkpoints (or settle active toilers / record the exact timestamp at which each plot threshold was first invalidated without overwriting it on subsequent updates) and cap `timestamp` at the earliest timestamp when `_toiler.plotId` lost eligibility since `_toiler.lastToilDate`:

```solidity
            uint256 effectiveTimestamp = timestamp;
            if (
                plotInvalidatedAt[landlord][_toiler.plotId] >
                _toiler.lastToilDate &&
                plotInvalidatedAt[landlord][_toiler.plotId] < effectiveTimestamp
            ) {
                effectiveTimestamp = plotInvalidatedAt[landlord][
                    _toiler.plotId
                ];
                toilerState[tokenId].dirty = true;
            }
            schnibblesTotal =
                (effectiveTimestamp - _toiler.lastToilDate) *
                BASE_SCHNIBBLE_RATE;
            schnibblesTotal = uint256(
                int256(schnibblesTotal) +
                    (int256(schnibblesTotal) * finalBonus) / 100
            );
            schnibblesLandlord =
                (schnibblesTotal * _toiler.latestTaxRate) /
                1e18;

            toilerState[tokenId].lastToilDate = effectiveTimestamp;
```

## Finding 17: Inconsistent Canonical Account Resolution in `LandManager._getMainAccountRequireRegistered` and `LandManager._getNumPlots`

### Synopsis

`LandManager._getMainAccountRequireRegistered` and `LandManager._getNumPlots` inconsistently resolve delegated subaccounts to canonical main accounts. While `_getMainAccountRequireRegistered` resolves `_account` to `_mainAccount` and checks registration on `_mainAccount`, `_getNumPlots` and landlord state persistence operate on unresolved addresses, causing identity mismatches between validation and state updates.

### Technical details

In the Munchables protocol, a registered player has a canonical `_mainAccount` and can authorize multiple delegated subaccounts in `AccountManager`. `AccountManager.getPlayer(address _account)` resolves any subaccount to its `_mainAccount` via `_getMainAccount(_account)` and returns `(_mainAccount, players[_mainAccount])`. Conversely, `AccountManager.updatePlayer(address _account, Player calldata _player)` writes directly to `players[_account]` without resolving `_account` to `_mainAccount`, and `LockManager.getLockedWeightedValue(address _player)` reads `lockedTokens[_player]` directly without canonical resolution.

1. **Registration checks bound to resolved main accounts in `_getMainAccountRequireRegistered` (lines 332–342):**
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
   ```
   When `_account` is a delegated subaccount of a registered player, `accountManager.getPlayer(_account)` returns the canonical `_mainAccount` and its `Player` struct. Line 340 validates `_player.registrationDate != 0` against `_mainAccount` rather than verifying whether `_account` itself is the canonical main account.

2. **Unresolved account lookup in `_getNumPlots` (lines 344–346):**
   ```solidity
   function _getNumPlots(address _account) internal view returns (uint256) {
       return lockManager.getLockedWeightedValue(_account) / PRICE_PER_PLOT;
   }
   ```
   Unlike `_getMainAccountRequireRegistered`, `_getNumPlots` passes `_account` directly to `lockManager.getLockedWeightedValue(_account)` without resolving `_account` to `_mainAccount`.

3. **Identity mismatch between validation and persistence across callers:**
   - In `_farmPlots` (lines 262–265 and 300), `_getMainAccountRequireRegistered(landlord)` validates registration on the resolved `_mainAccount` and retrieves `_mainAccount`'s `Player` struct as `landlordMetadata`, but discards the returned `_mainAccount` address and calls `accountManager.updatePlayer(landlord, landlordMetadata)` with the unresolved `landlord` address.
   - In `stakeMunchable` (lines 136–146), `msg.sender` is canonicalized via `_getMainAccountRequireRegistered(msg.sender)`, whereas `landlord` is not canonicalized before checking `if (landlord == mainAccount) revert CantStakeToSelfError()` or querying `_getNumPlots(landlord)`.

If a subaccount address is used as `landlord` (for instance, if tokens were locked on behalf of a subaccount address or a subaccount is passed as `landlord`), `_getMainAccountRequireRegistered(landlord)` validates `_mainAccount` and loads `players[_mainAccount]`, and `accountManager.updatePlayer(landlord, landlordMetadata)` overwrites `players[landlord]` with `_mainAccount`'s player metadata (including `registrationDate` and `unfedSchnibbles`), corrupting player state and Failing to credit `_mainAccount`.

### Assumptions and preconditions

1. A main account is registered in `AccountManager` and has linked a delegated subaccount.
2. Either tokens are locked under the subaccount address in `LockManager` (e.g., via `lockOnBehalf`) or a subaccount address is supplied where a canonical main account is expected.

### Proof of concept

1. Alice registers as a main account in `AccountManager` and adds `aliceSub` as a subaccount.
2. `aliceSub` has non-zero locked weighted value in `LockManager` (or is used as a `landlord` address).
3. Bob calls `stakeMunchable(aliceSub, tokenId, 0)`:
   - `_getMainAccountRequireRegistered(bob)` resolves Bob's main account.
   - `_getNumPlots(aliceSub)` queries `lockManager.getLockedWeightedValue(aliceSub)`.
   - `toilerState[tokenId].landlord` is recorded as `aliceSub`.
4. Later, Bob calls `farmPlots()`:
   - `_getMainAccountRequireRegistered(aliceSub)` resolves `aliceSub` to `alice`, checks `players[alice].registrationDate != 0`, and returns `players[alice]` as `landlordMetadata`.
   - `_farmPlots` ignores the resolved `alice` address and calls `accountManager.updatePlayer(aliceSub, landlordMetadata)`.
   - `AccountManager.updatePlayer` writes `players[alice]` (plus `schnibblesLandlord`) into `players[aliceSub]`, leaving `players[alice].unfedSchnibbles` uncredited and registering `aliceSub` with `alice`'s player struct in `AccountManager`.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** Validation and state retrieval in `_getMainAccountRequireRegistered` operate on the canonical `_mainAccount`, while `_getNumPlots` and `updatePlayer` operate on the unresolved `_account` / `landlord` address.
- **Business Impact:** Landlord rewards (`unfedSchnibbles`) and `lastPetMunchable` updates can be misdirected from the canonical main account to a subaccount record, duplicating `Player` metadata onto a subaccount address or preventing legitimate plot lookup via `_getNumPlots`.

### Remediation

Ensure canonical account resolution is consistently enforced in `_getNumPlots` and `_getMainAccountRequireRegistered` (or require canonical main accounts when resolving landlord accounts):

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
        (address _mainAccount, ) = _getMainAccountRequireRegistered(_account);
        return
            lockManager.getLockedWeightedValue(_mainAccount) / PRICE_PER_PLOT;
    }
```

Additionally, ensure all callers in `LandManager` (such as `stakeMunchable` and `_farmPlots`) capture and use the canonical `_mainAccount` returned by `_getMainAccountRequireRegistered(landlord)` when referencing or updating landlord state.
