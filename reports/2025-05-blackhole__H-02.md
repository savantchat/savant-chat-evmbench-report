# Audit report — 2025-05-blackhole/H-02

## Finding 1: Unprotected `GaugeFactoryCL.createGauge` Exposes `createEternalFarming` to Confused-Deputy Incentive Creation and Bootstrap Fund Drainage

### Synopsis

`GaugeFactoryCL.createGauge` lacks access control, allowing any external caller to invoke the internal `createEternalFarming` routine with arbitrary parameters. An attacker can abuse `GaugeFactoryCL`'s privileges and token balances to drain bootstrap reward funds, approve arbitrary spenders, and register unauthorized incentives on `AlgebraEternalFarming` to DoS legitimate pools.

### Technical details

In `contracts/AlgebraCLVe33/GaugeFactoryCL.sol`, `createGauge` (lines 59–67) is declared `external` without any access control modifier restricting callers to `GaugeManager` or authorized administrators. `createGauge` directly invokes `createEternalFarming(_pool, farmingParam.algebraEternalFarming, _rewardToken, _bonusRewardToken)` (lines 69–83) and appends the newly deployed `GaugeCL` to `__gauges`.

Inside `createEternalFarming`:
1. **Caller-Controlled Privileged Execution Targets**: The function accepts caller-supplied `_pool`, `_algebraEternalFarming`, `_rewardToken`, and `_bonusRewardToken` directly from `createGauge` without validation.
2. **Unauthorized Consumption of Shared Bootstrap Funds**: Because `AlgebraEternalFarming.createEternalFarming` requires a non-zero initial reward (`if (params.reward == 0) revert zeroRewardAmount()`), `GaugeFactoryCL` holds pre-funded `rewardToken` (`base` / `BLACK`) balances and sets `uint128 reward = 1e10` (line 75). It then executes `IERC20(_rewardToken).safeApprove(_algebraEternalFarming, reward)` (line 78) and calls `IAlgebraEternalFarmingCustom(_algebraEternalFarming).createEternalFarming(...)` (line 82). By passing a malicious contract as `farmingParam.algebraEternalFarming` (and mocking `_pool`), an attacker can repeatedly approve and pull `1e10` increments of `_rewardToken` held by `GaugeFactoryCL`, or exhaust the bootstrap funds via dummy pools so legitimate gauge creations revert.
3. **Confused-Deputy Incentive Creation**: `GaugeFactoryCL` holds the privileged `INCENTIVE_MAKER_ROLE` in `AlgebraEternalFarming`. An attacker can call `GaugeFactoryCL.createGauge` targeting a legitimate Algebra CL pool before `GaugeManager.createGauge` is called for it, passing arbitrary `_rewardToken` or `_bonusRewardToken` addresses alongside the legitimate `algebraEternalFarming` address. `AlgebraEternalFarming.createEternalFarming` connects a new `EternalVirtualPool` to the pool's plugin (`IFarmingPlugin(connectedPlugin).setIncentive(virtualPool)`). Because `AlgebraEternalFarming` enforces `if (IFarmingPlugin(connectedPlugin).incentive() != address(0)) revert anotherFarmingIsActive()`, any subsequent attempt by `GaugeManager` to create a legitimate gauge for that pool will permanently revert unless an administrator manually deactivates the spoofed incentive. Furthermore, pushing spoofed gauges into `__gauges` corrupts `veNFTAPIV1._getRewardsForNft`.

### Assumptions and preconditions

- `GaugeFactoryCL` has been deployed, initialized, granted `INCENTIVE_MAKER_ROLE` on `AlgebraEternalFarming`, and funded with `rewardToken` (`BLACK`) to cover the `1e10` initial farming reward per CL gauge.
- For pool DoS via confused-deputy incentive creation: a target Algebra CL pool has been deployed with a farming plugin, and a gauge has not yet been created for it via `GaugeManager`.

### Proof of concept

1. **Bootstrap Fund Drainage**:
   - The attacker deploys a helper contract implementing `tickSpacing()`, `plugin()`, `numOfIncentives()`, and `createEternalFarming(...)`.
   - In the helper's `createEternalFarming` implementation, it calls `IERC20(rewardToken).transferFrom(address(gaugeFactoryCL), attacker, 1e10)`.
   - The attacker calls `GaugeFactoryCL.createGauge(rewardToken, address(1), helper, address(1), address(1), address(1), true, FarmingParam(address(1), helper, address(1)), address(0))`.
   - `GaugeFactoryCL.createEternalFarming` approves `1e10` of `rewardToken` to `helper` and calls `helper.createEternalFarming(...)`, transferring the bootstrap funds to the attacker. Repeating this call drains `GaugeFactoryCL`'s `rewardToken` balance, causing future `GaugeManager.createGauge` calls for CL pools to revert with `STF` (`SafeTransferFrom` failure).

2. **Confused-Deputy Pool DoS**:
   - A legitimate Algebra CL pool `pool` is deployed via `CustomPoolDeployer.createCustomPool`.
   - Before `GaugeManager.createGauge(pool, 1)` is executed, the attacker calls `GaugeFactoryCL.createGauge(rewardToken, ve, pool, attacker, attacker, attacker, true, FarmingParam(farmingCenter, algebraEternalFarming, nfpm), worthlessToken)`.
   - `GaugeFactoryCL` uses its `INCENTIVE_MAKER_ROLE` to register an eternal farming incentive on `algebraEternalFarming` and bind a virtual pool to `pool.plugin()`.
   - When a user or admin subsequently calls `GaugeManager.createGauge(pool, 1)`, the call reverts in `AlgebraEternalFarming.createEternalFarming` with `anotherFarmingIsActive()`.

### Impact assessment

**Severity: High**

An unprivileged attacker can:
1. Steal or exhaust the `rewardToken` bootstrap balance held by `GaugeFactoryCL`, halting the creation of all new Concentrated Liquidity (CL) gauges across the protocol.
2. Abuse `GaugeFactoryCL`'s `INCENTIVE_MAKER_ROLE` to attach unauthorized farming incentives to legitimate Algebra CL pools, blocking `GaugeManager` from creating official gauges for those pools.
3. Pollute the `__gauges` array in `GaugeFactoryCL` with arbitrary `GaugeCL` instances, causing view functions like `veNFTAPIV1._getRewardsForNft` to revert or return corrupted data.

### Remediation

Restrict `GaugeFactoryCL.createGauge` so that only the authorized `GaugeManager` (or authorized admin) can invoke it.

```solidity
    address public gaugeManager;

    modifier onlyGaugeManager() {
        require(msg.sender == gaugeManager, "ERR: NOT_GAUGE_MANAGER");
        _;
    }

    function setGaugeManager(address _gaugeManager) external {
        require(owner() == msg.sender, "not owner");
        require(_gaugeManager != address(0), "zero address");
        gaugeManager = _gaugeManager;
    }

    function createGauge(
        address _rewardToken,
        address _ve,
        address _pool,
        address _distribution,
        address _internal_bribe,
        address _external_bribe,
        bool _isPair,
        IGaugeManager.FarmingParam memory farmingParam,
        address _bonusRewardToken
    ) external onlyGaugeManager returns (address) {
        createEternalFarming(_pool, farmingParam.algebraEternalFarming, _rewardToken, _bonusRewardToken);
        last_gauge = address(
            new GaugeCL(
                _rewardToken,
                _ve,
                _pool,
                _distribution,
                _internal_bribe,
                _external_bribe,
                _isPair,
                farmingParam,
                _bonusRewardToken,
                address(this)
            )
        );
        __gauges.push(last_gauge);
        return last_gauge;
    }
```

## Finding 2: Missing Pool-Plugin Association Verification in `GaugeFactoryCL.createEternalFarming` AllowsHijacking Pool Plugins and Permanently DoSing Legitimate CL Gauges

### Synopsis

`GaugeFactoryCL.createEternalFarming` fails to verify that `_pool` is a legitimate Algebra pool and matches `IFarmingPlugin(pluginAddress).getPool()`. Because `GaugeManager._createGauge` skips factory validation for CL gauges (`_gaugeType == 1`), an attacker can pass a counterfeit pool returning a legitimate pool's plugin, permanently bricking gauge creation and farming for the legitimate pool.

### Technical details

When creating a concentrated liquidity (CL) gauge (`_gaugeType == 1`) via the permissionless `GaugeManager.createGauge` function, `GaugeManager._createGauge` does not verify that `_pool` was deployed by the Algebra factory (lines 202–206 set `isPair = true` unconditionally after the factory check was commented out). It then invokes `GaugeFactoryCL.createGauge`, which calls `GaugeFactoryCL.createEternalFarming(_pool, farmingParam.algebraEternalFarming, _rewardToken, _bonusRewardToken)`.

In `GaugeFactoryCL.createEternalFarming` (`contracts/AlgebraCLVe33/GaugeFactoryCL.sol`):
1. Line 72 queries `address pluginAddress = algebraPool.plugin();` from the user-supplied `_pool`.
2. Line 73 constructs `incentivekey` with `key.pool = IAlgebraPool(_pool)`.
3. Line 79 queries `address customDeployer = IAlgebraPoolAPIStorage(algebraPoolAPIStorage).pairToDeployer(_pool);` (which returns `address(0)` for any unmapped address, matching the `address(0)` deployer used by standard Algebra pools).
4. Line 82 calls `IAlgebraEternalFarmingCustom(_algebraEternalFarming).createEternalFarming(incentivekey, incentiveParams, pluginAddress, customDeployer)`.

Inside `AlgebraEternalFarming.createEternalFarming`:
- It checks `key.pool.plugin() == plugin` (which calls `_pool.plugin()` again) and verifies `IFarmingPlugin(connectedPlugin).incentive() == address(0)`.
- It then calls `FarmingCenter.connectVirtualPoolToPlugin(virtualPool, IFarmingPlugin(connectedPlugin), ...)`.
- However, `FarmingCenter._checkParamsForVirtualPoolToggle` derives the pool address to validate from `plugin.getPool()`, **not** from `key.pool` (`_pool`).

Because `plugin.getPool()` returns the legitimate `AlgebraPool` associated with `pluginAddress`, `FarmingCenter` validates the legitimate pool's deterministic address, attaches the newly created `virtualPool` to the legitimate pool's plugin (`plugin.setIncentive(newVirtualPool)`), and sets `virtualPoolAddresses[address(realPool)] = newVirtualPool`. Meanwhile, `AlgebraEternalFarming` records `incentiveKeys[address(_pool)] = key` under the counterfeit `_pool`.

This creates a critical state corruption:
1. Any subsequent attempt to create a gauge for the legitimate `realPool` reverts in `AlgebraEternalFarming.createEternalFarming` with `anotherFarmingIsActive()` because `IFarmingPlugin(connectedPlugin).incentive()` is already non-zero.
2. Users cannot stake NFT positions of `realPool` into the counterfeit gauge either, because `AlgebraEternalFarming._enterFarming` checks `if (pool != key.pool) revert invalidPool()`, where `pool` is `realPool` and `key.pool` is the counterfeit `_pool`.
3. Each counterfeit gauge creation also consumes `1e10` of `rewardToken` from `GaugeFactoryCL` (approved at line 78) and registers a bogus active gauge in `GaugeManager` that can receive protocol emissions.

### Assumptions and preconditions

1. A legitimate Algebra CL pool (`realPool`) has been deployed with an associated `IFarmingPlugin` (`realPlugin`), and a CL gauge has not yet been created for `realPool` (so `realPlugin.incentive() == address(0)`).
2. At least two tokens are whitelisted in `TokenHandler`, with at least one being a connector token (standard protocol operating state).
3. `GaugeFactoryCL` holds at least `1e10` wei of `rewardToken` (`BLACK`) to initialize eternal farming incentives.

### Proof of concept

1. A legitimate Algebra pool `realPool` is deployed with `realPlugin = realPool.plugin()`, where `realPlugin.getPool() == realPool` and `realPlugin.incentive() == address(0)`.
2. An attacker deploys a minimal contract `FakePool` implementing:
   - `token0()` and `token1()`: returning a whitelisted token and a connector token.
   - `tickSpacing()`: returning `60`.
   - `plugin()`: returning `realPlugin`.
   - `communityVault()`: returning a dummy contract implementing `changeCommunityFeeReceiver(address)`.
3. The attacker calls `GaugeManager.createGauge(address(FakePool), 1)`.
4. `GaugeManager._createGauge` sets `isPair = true` and calls `GaugeFactoryCL.createGauge`, which executes `GaugeFactoryCL.createEternalFarming(address(FakePool), ...)`.
5. `GaugeFactoryCL.createEternalFarming` reads `pluginAddress = FakePool.plugin()` (`realPlugin`) and `customDeployer = pairToDeployer(address(FakePool))` (`address(0)`), approves `1e10` `rewardToken`, and calls `AlgebraEternalFarming.createEternalFarming` with `key.pool = FakePool` and `plugin = realPlugin`.
6. `FarmingCenter` validates `realPlugin.getPool()` (`realPool`), which succeeds, and binds `realPlugin.setIncentive(virtualPool)`.
7. When a legitimate user or admin later calls `GaugeManager.createGauge(realPool, 1)`, `AlgebraEternalFarming.createEternalFarming` checks `IFarmingPlugin(realPlugin).incentive() != address(0)` and reverts with `anotherFarmingIsActive()`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** An attacker can permissionlessly hijack the `IFarmingPlugin` of any legitimate Algebra pool before its gauge is created, permanently blocking `GaugeManager.createGauge` for that pool (`anotherFarmingIsActive()`) and rendering its farming functionality completely unusable (`invalidPool()`). Additionally, the attacker registers a fake gauge in `GaugeManager` and consumes `1e10` `rewardToken` from `GaugeFactoryCL` per call.
- **Business Implications:** Newly deployed Algebra CL pools can be front-run and permanently denied gauge creation and farming incentives unless an administrator manually intervenes to deactivate the maliciously registered incentive on `AlgebraEternalFarming`.

### Remediation

In `GaugeFactoryCL.createEternalFarming`, verify that the plugin returned by `_pool.plugin()` is genuinely bound to `_pool` by checking `IFarmingPlugin(pluginAddress).getPool() == _pool` before approving rewards and calling `createEternalFarming`.

```solidity
    function createEternalFarming(address _pool, address _algebraEternalFarming, address _rewardToken, address _bonusRewardToken) internal {
        IAlgebraPool algebraPool = IAlgebraPool(_pool);
        uint24 tickSpacing = uint24(algebraPool.tickSpacing());
        address pluginAddress = algebraPool.plugin();
        require(IFarmingPlugin(pluginAddress).getPool() == _pool, "Invalid pool plugin");
        IncentiveKey memory incentivekey = getIncentiveKey(_rewardToken, _bonusRewardToken, _pool, _algebraEternalFarming);
        uint256 remainingTimeInCurrentEpoch = BlackTimeLibrary.epochNext(block.timestamp) - block.timestamp;
        uint128 reward = 1e10;
        uint128 rewardRate = uint128(reward/remainingTimeInCurrentEpoch);
        
        IERC20(_rewardToken).safeApprove(_algebraEternalFarming, reward);
        address customDeployer = IAlgebraPoolAPIStorage(algebraPoolAPIStorage).pairToDeployer(_pool);
        IAlgebraEternalFarming.IncentiveParams memory incentiveParams = 
            IAlgebraEternalFarming.IncentiveParams(reward, 0, rewardRate, 0, tickSpacing);
        IAlgebraEternalFarmingCustom(_algebraEternalFarming).createEternalFarming(incentivekey, incentiveParams, pluginAddress, customDeployer);
    }
```
