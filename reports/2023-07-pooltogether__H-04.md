# Audit report — 2023-07-pooltogether/H-04

## Finding 1: Unaccounted Accrued Yield Fee Shares in `Vault.totalSupply` and Exchange Rate Calculation Causes Share Accounting Desynchronization

### Synopsis

`Vault.totalSupply()` and `Vault._currentExchangeRate()` query `_totalSupply()` (minted shares in `TwabController`) instead of `_totalShares()` (minted shares plus accrued `_yieldFeeTotalSupply`). This reconciliation failure between issued and accrued-fee shares overstates the exchange rate and masks vault undercollateralization when yield fees are accrued.

### Technical details

In `Vault.sol`, user share balances and minted share supply are externalized to `_twabController`. When yield is liquidated, the liquidator's shares are immediately minted in `_twabController`, whereas the protocol's yield fee shares are accrued separately in `_yieldFeeTotalSupply` via `_increaseYieldFeeBalance()` (lines 852–854) until claimed.

To reconcile issued shares with accrued fee shares, `Vault` defines two internal helpers:
- `_totalSupply()` (lines 808–810), which returns `_twabController.totalSupply(address(this))` (only minted shares).
- `_totalShares()` (lines 818–820), which returns `_totalSupply() + _yieldFeeTotalSupply` (all shares backed by the vault's assets).

While `availableYieldBalance()` (lines 311–316) uses `_totalShares()` to compute the assets backing all allocated shares, `totalSupply()` (lines 350–352) and `_currentExchangeRate()` (lines 1168–1187) use `_totalSupply()` and ignore `_yieldFeeTotalSupply`:

1. **`Vault.totalSupply()` (lines 350–352)** returns `_totalSupply()` rather than `_totalShares()`. Meanwhile, `Vault.totalAssets()` (lines 345–347) returns `_totalAssets()`, which includes the underlying assets in `_yieldVault` that back `_yieldFeeTotalSupply`. Any external integration or ERC4626 consumer relying on `totalSupply()` and `totalAssets()` observes an understated share supply relative to the vault's managed assets.
2. **`Vault._currentExchangeRate()` (lines 1168–1187)** sets `_totalSupplyAmount = _totalSupply()` (line 1169) and computes `_totalSupplyToAssets` and the exchange rate denominator using only minted shares instead of `_totalShares()`:
   - **Undetected Undercollateralization:** If `_yieldVault` suffers a loss smaller than or equal to the assets backing `_yieldFeeTotalSupply`, `_withdrawableAssets` remains greater than or equal to `_totalSupplyToAssets`. Lines 1178–1180 cap `_withdrawableAssets` to `_totalSupplyToAssets`, causing `_currentExchangeRate()` to return `_assetUnit` (`1e18`) and `_isVaultCollateralized()` (line 1195) to falsely return `true` even though the vault lacks sufficient assets to cover both `_totalSupply()` and `_yieldFeeTotalSupply`.
   - **Inflated Exchange Rate & Insolvency on Withdrawal:** When `_withdrawableAssets < _totalSupplyToAssets`, line 1183 calculates `_withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down)` using `_totalSupply()` as the denominator instead of `_totalShares()`. Each minted share is valued too highly because the loss is not apportioned across `_yieldFeeTotalSupply`. Minted shareholders can withdraw up to 100% of `_withdrawableAssets`, leaving zero underlying assets to back the accrued `_yieldFeeTotalSupply` shares (or leaving the last withdrawers unable to redeem if `mintYieldFee` is called before withdrawals complete).

### Assumptions and preconditions

1. The vault has a non-zero `_yieldFeePercentage` configured.
2. Yield has been liquidated such that `_yieldFeeTotalSupply > 0` has been accrued via `_increaseYieldFeeBalance()`, and those fee shares have not yet been minted via `mintYieldFee()`.
3. The underlying `_yieldVault` experiences a loss (reducing `_yieldVault.maxWithdraw(address(this))` below the assets required to back `_totalShares()`), or an external caller/integrator queries `totalSupply()` alongside `totalAssets()`.

### Proof of concept

1. Alice deposits `1,000e18` underlying assets into `Vault`. `_twabController.totalSupply(address(vault))` (`_totalSupply()`) is `1,000e18`, and `_yieldVault.maxWithdraw(address(vault))` is `1,000e18`.
2. `_yieldVault` accrues `100e18` in yield (`_totalAssets() = 1,100e18`).With a 10% yield fee, a liquidator liquidates the yield: `90e18` shares are minted to the liquidator (`_totalSupply() = 1,090e18`) and `10e18` shares are added to `_yieldFeeTotalSupply` (`_totalShares() = 1,100e18`).
3. **Scenario A (Masked Undercollateralization):** `_yieldVault` loses `5e18` assets, leaving `_withdrawableAssets = 1,095e18` against `_totalShares() = 1,100e18`.
   - In `_currentExchangeRate()`, `_totalSupplyAmount = _totalSupply() = 1,090e18` and `_totalSupplyToAssets = 1,090e18`.
   - Because `_withdrawableAssets (1,095e18) > _totalSupplyToAssets (1,090e18)`, `_withdrawableAssets` is clamped to `1,090e18`, and `_currentExchangeRate()` returns `1e18` (`_assetUnit`).
   - `_isVaultCollateralized()` returns `true` despite the `5e18` deficit. `mintYieldFee(10e18, feeRecipient)` is not blocked by the collateralization check and mints `10e18` unbacked shares, or Alice and the liquidator can withdraw `1,090e18` assets at a 1:1 rate, leaving only `5e18` assets to cover `10e18` fee shares.
4. **Scenario B (Overstated Undercollateralized Exchange Rate):** `_yieldVault` loses `550e18` assets, leaving `_withdrawableAssets = 550e18` against `_totalShares() = 1,100e18` (true exchange rate should be `550e18 * 1e18 / 1,100e18 = 0.5e18`).
   - `_currentExchangeRate()` computes `550e18 * 1e18 / 1,090e18 = 504,587,155,963,302,752` (~`0.504587e18`).
   - Alice and the liquidator redeem their `1,090e18` shares and withdraw all `550e18` assets from `_yieldVault`, leaving `0` assets for the `10e18` shares in `_yieldFeeTotalSupply`.

### Impact assessment

- **Severity:** High / Medium
- **Direct Technical Effects:**
  - `Vault.totalSupply()` under-reports total allocated vault shares by `_yieldFeeTotalSupply`.
  - `Vault._currentExchangeRate()` overstates the share-to-asset exchange rate whenever `_yieldFeeTotalSupply > 0` and `_yieldVault` experiences a loss.
  - `Vault._isVaultCollateralized()` fails to detect undercollateralization when the asset deficit is less than or equal to the value of `_yieldFeeTotalSupply`.
- **Business Implications:**
  - Existing shareholders can withdraw more than their pro-rata share of remaining assets during a loss event, exhausting vault liquidity and leaving the yield fee recipient (or late withdrawers if `mintYieldFee` is executed while the deficit is masked) with unbacked shares.

### Remediation

Update `totalSupply()` and `_currentExchangeRate()` to use `_totalShares()` (which includes `_yieldFeeTotalSupply`) so that all allocated shares are consistently accounted for in supply queries and exchange rate calculations.

```solidity
  /// @inheritdoc ERC20
  function totalSupply() public view virtual override(ERC20, IERC20) returns (uint256) {
    return _totalShares();
  }
```

And in `_currentExchangeRate()`:

```solidity
  function _currentExchangeRate() internal view returns (uint256) {
    uint256 _totalSupplyAmount = _totalShares();
    uint256 _totalSupplyToAssets = _convertToAssets(
      _totalSupplyAmount,
      _lastRecordedExchangeRate,
      Math.Rounding.Down
    );

    uint256 _withdrawableAssets = _yieldVault.maxWithdraw(address(this));

    if (_withdrawableAssets > _totalSupplyToAssets) {
      _withdrawableAssets = _withdrawableAssets - (_withdrawableAssets - _totalSupplyToAssets);
    }

    if (_totalSupplyAmount != 0 && _withdrawableAssets != 0) {
      return _withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down);
    }

    return _assetUnit;
  }
```

## Finding 2: Inconsistent and Manipulable Asset Valuation in `Vault.totalAssets` and `Vault._totalAssets` Leads to Vault Undercollateralization and DoS

### Synopsis

`Vault.totalAssets()` (via `_totalAssets()`) computes managed assets as `_yieldVault.maxWithdraw(address(this)) + super.totalAssets()`, creating an asset valuation mismatch with `_currentExchangeRate()` and conflating withdrawable liquidity with total backing. Direct token transfers to `Vault` inflate `availableYieldBalance()`, and subsequent yield liquidation permanently pushes the vault into an undercollateralized state.

### Technical details

In `Vault.sol`, `totalAssets()` (`345-347`) delegates to `_totalAssets()` (`800-802`):

```solidity
function totalAssets() public view virtual override returns (uint256) {
  return _totalAssets();
}

function _totalAssets() internal view returns (uint256) {
  return _yieldVault.maxWithdraw(address(this)) + super.totalAssets();
}
```

This asset valuation design introduces two closely related flaws across the vault's accounting, yield calculation, and collateralization paths:

1. **Inconsistent Accounting of Idle vs. Invested Assets (`super.totalAssets()`):**
   - `super.totalAssets()` (`ERC4626.totalAssets()`) returns `_asset.balanceOf(address(this))`, representing idle underlying tokens held directly by the `Vault` contract.
   - `availableYieldBalance()` (`311-316`) queries `_totalAssets()`, so any idle underlying tokens held directly by `Vault` are counted as available yield (`_assets - _sharesToAssets`).
   - When this yield is liquidated, `Vault` mints new shares (`_mint` at `1122-1127`, increasing `_totalSupply()` in `_twabController`) and updates the exchange rate via `_updateExchangeRate()`, which relies on `_currentExchangeRate()` (`1168-1187`).
   - However, `_currentExchangeRate()` computes withdrawable backing solely as:
     ```solidity
     uint256 _withdrawableAssets = _yieldVault.maxWithdraw(address(this));
     ```
     It completely omits `super.totalAssets()` (`_asset.balanceOf(address(this))`).
   - Because the newly minted shares increase `_totalSupply()` (and thus `_totalSupplyToAssets`) while the idle assets remain in `Vault` rather than `_yieldVault`, `_withdrawableAssets` is strictly less than `_totalSupplyToAssets`. Consequently, `_currentExchangeRate()` computes `_withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down) < _assetUnit`.
   - This causes `_isVaultCollateralized()` (`1194-1196`) to evaluate to `false`, setting `maxMint()` (`383-385`) and `maxDeposit()` to `0`, blocking `mintYieldFee()`, and locking the vault in an undercollateralized state.

2. **Conflation of Withdrawable Liquidity (`maxWithdraw`) with Total Asset Backing:**
   - `_totalAssets()` and `_currentExchangeRate()` use `_yieldVault.maxWithdraw(address(this))` instead of the total underlying assets backing the vault's shares (`_yieldVault.convertToAssets(_yieldVault.balanceOf(address(this)))`), violating the EIP-4626 requirement that `totalAssets()` reflect total managed assets rather than instantaneous withdrawal limits.
   - If `_yieldVault` imposes withdrawal limits or experiences high utilization where `maxWithdraw(address(this))` is temporarily less than the vault's share value, `totalAssets()` and `_currentExchangeRate()` underreport the vault's backing, causing `_isVaultCollateralized()` to return `false` and recording an artificially depressed `_lastRecordedExchangeRate` when shares are burned on withdrawal.

### Assumptions and preconditions

- The `Vault` has a non-zero `totalSupply()` of shares backed by assets in `_yieldVault` (e.g., `1000e18` shares backed by `1000e18` assets in `_yieldVault`, with `_lastRecordedExchangeRate == _assetUnit`).
- An external actor transfers any non-zero amount of the underlying `asset` directly to `address(vault)` (or idle assets remain in `Vault`), and the resulting `availableYieldBalance()` is liquidated.

### Proof of concept

1. Suppose Alice deposits `1000e18` underlying assets into `Vault`.
   - `_twabController.totalSupply(address(vault))` (`totalSupply()`) is `1000e18`.
   - `_yieldVault.maxWithdraw(address(vault))` is `1000e18`.
   - `super.totalAssets()` (`_asset.balanceOf(address(vault))`) is `0`.
   - `_lastRecordedExchangeRate` is `1e18` (`_assetUnit`).
2. An attacker transfers `10e18` (or even a small dust amount) of the underlying `asset` directly to `address(vault)`.
   - `super.totalAssets()` becomes `10e18`.
   - `totalAssets()` / `_totalAssets()` returns `1000e18 + 10e18 = 1010e18`.
   - `availableYieldBalance()` returns `1010e18 - 1000e18 = 10e18`.
3. The `10e18` available yield is liquidated via `liquidate()`:
   - `Vault` mints `10e18` shares (and/or allocates yield fee shares), increasing `_totalSupply()` to `1010e18`.
   - `_mint()` calls `_updateExchangeRate()`, which invokes `_currentExchangeRate()`.
   - In `_currentExchangeRate()`:
     - `_totalSupplyAmount` is `1010e18` and `_totalSupplyToAssets` is `1010e18`.
     - `_withdrawableAssets = _yieldVault.maxWithdraw(address(this))` is still `1000e18` because the `10e18` donated tokens are sitting idle in `Vault`, not in `_yieldVault`.
     - `_currentExchangeRate()` calculates `1000e18 * 1e18 / 1010e18 = 990099009900990099` (`< 1e18`).
   - `_lastRecordedExchangeRate` is updated to `990099009900990099`.
4. Now `_isVaultCollateralized()` returns `false`:
   - `maxMint()` and `maxDeposit()` return `0`, preventing any further deposits or mints (which also prevents sweeping the idle tokens into `_yieldVault` via a deposit).
   - `mintYieldFee()` reverts with `VaultUnderCollateralized()`.
   - Existing depositors who redeem or withdraw receive a reduced exchange rate (`~0.99` assets per share) because their withdrawals only pull from `_yieldVault`.

### Impact assessment

- **Severity:** High / Medium
- **Direct Technical Effects:**
  1. **Denial of Service (DoS):** Any user can donate a small amount of underlying tokens directly to `Vault` and trigger a yield liquidation, causing `_currentExchangeRate()` to fall below `_assetUnit`. This makes `_isVaultCollateralized()` return `false`, disabling `deposit()`, `mint()`, and `mintYieldFee()`.
  2. **Loss / Freezing of Value for Depositors:** Because `_lastRecordedExchangeRate` is Ratcheted down below `_assetUnit` while the idle `super.totalAssets()` sit unaccounted for in `_currentExchangeRate()` and withdrawals only pull from `_yieldVault`, withdrawing users suffer a loss proportional to the uninvested idle balance, and the idle tokens remain stranded in `Vault`.

### Remediation

Ensure consistent accounting between `totalAssets()` / `_totalAssets()` and `_currentExchangeRate()`, and avoid treating uninvested idle tokens in `Vault` as liquidatable yield unless they are also accounted for in exchange rate and withdrawal calculations (or sweep latent balance into `_yieldVault` before accounting for yield). Additionally, use `convertToAssets(balanceOf(address(this)))` on `_yieldVault` for total asset reporting in `totalAssets()`.

Specifically, if `Vault` does not hold idle assets outside of deposits, `_totalAssets()` should not include `super.totalAssets()` unless `_currentExchangeRate()` and withdrawal flows also account for `super.totalAssets()`:

```solidity
  /// @inheritdoc ERC4626
  function totalAssets() public view virtual override returns (uint256) {
    return _totalAssets();
  }

  function _totalAssets() internal view returns (uint256) {
    return _yieldVault.convertToAssets(_yieldVault.balanceOf(address(this)));
  }
```

Alternatively, if `super.totalAssets()` is intentionally included in `_totalAssets()`, include `super.totalAssets()` in `_currentExchangeRate()`'s `_withdrawableAssets` calculation and ensure withdrawals can utilize idle assets held in `Vault`.

## Finding 3: Premature Collateralization Check and Excluded Yield Fee Liability in `Vault.mintYieldFee` Allows Minting Yield Fees into an Undercollateralized State

### Synopsis

`Vault.mintYieldFee` checks `_requireVaultCollateralized()` before minting `_shares`, while `_currentExchangeRate()` only accounts for `_totalSupply()` and excludes unminted `_yieldFeeTotalSupply` liabilities. Consequently, if the underlying yield vault suffers a loss smaller than accrued yield fees, `mintYieldFee` succeeds and leaves the vault undercollateralized, diluting existing depositors.

### Technical details

In `Vault.sol`, total share liabilities managed by the vault consist of minted shares tracked in `_twabController` (`_totalSupply()`) plus accrued but unminted yield fee shares (`_yieldFeeTotalSupply`), represented together by `_totalShares()`:

```solidity
function _totalShares() internal view returns (uint256) {
  return _totalSupply() + _yieldFeeTotalSupply;
}
```

When yield is liquidated in `liquidate()`, yield fee shares are accrued into `_yieldFeeTotalSupply` without immediately minting them into `_twabController`. Later, `mintYieldFee()` converts `_shares` from `_yieldFeeTotalSupply` into minted shares via `_mint()`:

```solidity
function mintYieldFee(uint256 _shares, address _recipient) external {
  _requireVaultCollateralized();
  if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);

  _yieldFeeTotalSupply -= _shares;
  _mint(_recipient, _shares);

  emit MintYieldFee(msg.sender, _recipient, _shares);
}
```

However, `_requireVaultCollateralized()` validates `_currentExchangeRate() >= _assetUnit`, and `_currentExchangeRate()` computes the backing ratio using only `_totalSupply()` rather than `_totalShares()`:

```solidity
function _currentExchangeRate() internal view returns (uint256) {
  uint256 _totalSupplyAmount = _totalSupply();
  uint256 _totalSupplyToAssets = _convertToAssets(
    _totalSupplyAmount,
    _lastRecordedExchangeRate,
    Math.Rounding.Down
  );

  uint256 _withdrawableAssets = _yieldVault.maxWithdraw(address(this));
  ...
  if (_totalSupplyAmount != 0 && _withdrawableAssets != 0) {
    return _withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down);
  }

  return _assetUnit;
}
```

Because `_requireVaultCollateralized()` is executed on line 395 *before* `_mint(_recipient, _shares)` on line 399—and because `_currentExchangeRate()` ignores `_yieldFeeTotalSupply`—the pre-mint check only verifies that `_withdrawableAssets` covers the existing `_totalSupply()`.

If the underlying `_yieldVault` loses assets such that:

$$\text{\_totalSupply()} \le \text{\_withdrawableAssets} < \text{\_totalSupply()} + \text{\_shares}$$

then:
1. `_requireVaultCollateralized()` on line 395 evaluates `_currentExchangeRate()` using the pre-mint `_totalSupply()` and returns `true` (`_currentExchangeRate() == _assetUnit`).
2. `_yieldFeeTotalSupply` is decremented by `_shares`, and `_mint(_recipient, _shares)` increases `_totalSupply()` by `_shares`.
3. `_mint()` calls `_updateExchangeRate()`, which recomputes `_currentExchangeRate()` with the new `_totalSupply() + _shares`. Since `_withdrawableAssets < _totalSupply() + _shares`, `_lastRecordedExchangeRate` drops below `_assetUnit`.

Thus, `mintYieldFee()` succeeds even though the vault is insolvent with respect to its total share obligations, transitioning the vault into an undercollateralized state and allowing yield fee shares to be minted and redeemed at the expense of user principal.

### Assumptions and preconditions

1. The vault has non-zero `_yieldFeeTotalSupply` accrued from prior yield liquidations.
2. The underlying `_yieldVault` experiences a loss (or reduction in `maxWithdraw(address(this))`) such that withdrawable assets are greater than or equal to `_convertToAssets(_totalSupply(), _lastRecordedExchangeRate, Math.Rounding.Down)` but strictly less than the assets required to back `_totalSupply() + _yieldFeeTotalSupply`.

### Proof of concept

1. Alice deposits `1,000e18` underlying assets into `Vault` (18 decimals, `_assetUnit = 1e18`).
   - `_totalSupply() = 1,000e18`
   - `_lastRecordedExchangeRate = 1e18`
2. The `_yieldVault` accrues `100e18` in yield (`_withdrawableAssets = 1,100e18`).With a 10% yield fee (`_yieldFeePercentage = 1e8`), a liquidator liquidates `90e18` yield for vault shares:
   - `_totalSupply() = 1,090e18`
   - `_yieldFeeTotalSupply = 10e18`
   - `_totalShares() = 1,100e18`
3. The `_yieldVault` suffers a loss of `8e18` underlying assets, leaving `_withdrawableAssets = 1,092e18`.
   - Total obligations are `1,100e18` shares against `1,092e18` assets (the vault is undercollateralized by `8e18`).
4. `mintYieldFee(10e18, recipient)` is called:
   - At line 395, `_requireVaultCollateralized()` calls `_currentExchangeRate()`.
   - `_totalSupplyAmount` is `1,090e18` and `_withdrawableAssets` is `1,092e18`.
   - `_withdrawableAssets` is capped at `_totalSupplyToAssets` (`1,090e18`), so `_currentExchangeRate()` returns `1e18` (`>= _assetUnit`). The check passes.
   - At lines 398–399, `_yieldFeeTotalSupply` becomes `0` and `_mint(recipient, 10e18)` mints `10e18` shares.
   - Inside `_mint()`, `_updateExchangeRate()` recomputes `_currentExchangeRate()` with `_totalSupplyAmount = 1,100e18` and `_withdrawableAssets = 1,092e18`, recording `_lastRecordedExchangeRate = 1,092e18 * 1e18 / 1,100e18 = 992727272727272727` (`< 1e18`).
5. The vault is now undercollateralized (`isVaultCollateralized() == false`), new deposits/mints are halted (`maxDeposit == 0`, `maxMint == 0`), and `recipient` has been minted `10e18` shares that socialize the `8e18` loss across existing depositors instead of blocking yield fee minting when undercollateralized.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** `mintYieldFee` bypasses the collateralization invariant whenever `0 < _totalShares() - _withdrawableAssets <= _yieldFeeTotalSupply`, minting unbacked fee shares and forcing `_lastRecordedExchangeRate` below `_assetUnit`.
- **Business Impact:** Existing depositors suffer direct loss of funds (principal dilution) because unbacked yield fee shares are minted and given a pro-rata claim on the remaining underlying assets, violating the protocol invariant that no yield fees can be minted when the vault cannot fully back all shares.

### Remediation

Account for total share liabilities (`_totalShares()`) in `_currentExchangeRate()` or verify collateralization after minting the yield fee shares in `mintYieldFee()` so that minting yield fees reverts if the resulting share supply is not fully collateralized.

```solidity
  function mintYieldFee(uint256 _shares, address _recipient) external {
    if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);

    _yieldFeeTotalSupply -= _shares;
    _mint(_recipient, _shares);
    _requireVaultCollateralized();

    emit MintYieldFee(msg.sender, _recipient, _shares);
  }
```

## Finding 4: Incorrect Share Limit in `Vault.maxMint` Allows Total Supply to Exceed `TwabController` Capacity and Violates EIP-4626

### Synopsis

`Vault.maxMint` returns a static per-transaction limit of `type(uint96).max` whenever the vault is collateralized, instead of accounting for already minted shares and accrued yield fee shares. This allows aggregate share supply to exceed `uint96` capacity in `TwabController`, causing silent truncation in TWAB observations, overflows, and EIP-4626 compliance violations.

### Technical details

According to EIP-4626, `maxMint(address receiver)` must return the maximum amount of shares that can be minted for `receiver` through a `mint` call without reverting or corrupting state.

In `Vault.sol`, `maxMint` is implemented as:
```solidity
function maxMint(address) public view virtual override returns (uint256) {
  return _isVaultCollateralized() ? type(uint96).max : 0;
}
```

The comment on line 381 states:
> `@dev We use type(uint96).max cause this is the type used to store balances in TwabController.`

However, `TwabController` and `TwabLib` rely on narrow integer types across both individual account records and the vault's aggregate total supply record (`totalSupplyObservations[vault]`):
1. `Vault._mint` downcasts `_shares` to `uint96` when calling `_twabController.mint(_receiver, uint96(_shares))` (line 1123).
2. In `TwabLib.increaseBalances` (used for both `userObservations` and `totalSupplyObservations`), `accountDetails.balance` and `accountDetails.delegateBalance` are stored as `uint112`.
3. Whenever an observation is recorded in `TwabLib.increaseBalances`, the cumulative/current delegated balance is downcast to `uint96`:
   ```solidity
   observation = ObservationLib.Observation({
     balance: uint96(accountDetails.delegateBalance),
     cumulativeBalance: _extrapolateFromBalance(newestObservation, currentTime),
     timestamp: currentTime
   });
   ```

Because `maxMint` returns a fixed constant `type(uint96).max` per call rather than bounding the vault's aggregate share supply (`_totalShares()`, which equals `_totalSupply() + _yieldFeeTotalSupply`), two severe issues arise:
- **Cross-width silent truncation in `TwabLib` (`uint112` to `uint96`):** After any non-zero amount of shares is minted (and not delegated to `SPONSORSHIP_ADDRESS`), minting up to `maxMint(receiver)` pushes `accountDetails.delegateBalance` in `totalSupplyObservations` (and potentially `userObservations`) above `type(uint96).max`. When recorded into `ObservationLib.Observation`, `uint96(accountDetails.delegateBalance)` silently wraps modulo $2^{96}$, corrupting user and total-supply TWAB observations used for prize distribution.
- **Advertised mint limit exceeding executable capacity (`uint112` overflow):** Once the existing balance or total supply in `TwabController` exceeds `type(uint112).max - type(uint96).max` (i.e., after 65,535 mints of `type(uint96).max`), a subsequent `mint(maxMint(receiver), receiver)` call passes `_beforeMint` validation (line 972) but reverts inside `TwabLib.increaseBalances` due to `uint112` addition overflow, directly violating the EIP-4626 specification that `mint` must not revert for amounts up to `maxMint`.

### Assumptions and preconditions

- The vault is collateralized (`_isVaultCollateralized() == true`).
- One or more depositors hold enough underlying assets to mint shares whose cumulative total exceeds `type(uint96).max` (particularly realistic for low-decimal underlying tokens such as 2-decimal GUSD or 6-decimal USDC/USDT).

### Proof of concept

1. Suppose a `Vault` is deployed for a 6-decimal token (e.g., USDC, where `type(uint96).max` shares equals $\approx 7.92 \times 10^{22}$ units).
2. Alice calls `vault.mint(1_000e6, alice)`. The vault is collateralized, `_totalSupply()` becomes `1_000e6`, and `totalSupplyObservations[vault].details.delegateBalance` is `1_000e6`.
3. Bob queries `vault.maxMint(bob)`, which returns `type(uint96).max` despite `1_000e6` shares already existing.
4. Bob calls `vault.mint(type(uint96).max, bob)`.
5. `_beforeMint` checks `_shares > maxMint(bob)`, which evaluates to `type(uint96).max > type(uint96).max` (`false`), allowing execution to proceed.
6. In `TwabLib.increaseBalances` for `totalSupplyObservations[vault]`, `accountDetails.delegateBalance` becomes `uint112(type(uint96).max) + 1_000e6`.
7. When recording the observation, `uint96(accountDetails.delegateBalance)` silently overflows and wraps around to `1_000e6 - 1`, corrupting the vault's total supply TWAB history.
8. If additional mints continue up to `type(uint112).max`, `vault.maxMint` still returns `type(uint96).max`, but calling `vault.mint(vault.maxMint(bob), bob)` reverts with an arithmetic overflow panic in `TwabLib.increaseBalances`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:**
  1. **TWAB Corruption:** Silent `uint96` truncation in `ObservationLib.Observation` corrupts user and total supply time-weighted average balances when cumulative shares exceed `type(uint96).max`, skewing prize eligibility Calculations in `PrizePool`.
  2. **EIP-4626 Non-Compliance:** `maxMint` overstates the mintable share capacity of the vault, causing `mint` calls within the advertised `maxMint` bound to either corrupt TWAB state or revert on `uint112` overflow.

### Remediation

Update `Vault.maxMint` to subtract `_totalShares()` (which includes both minted supply in `TwabController` and unminted `_yieldFeeTotalSupply`) from `type(uint96).max`, returning `0` if `_totalShares()` has already reached or exceeded `type(uint96).max`.

```solidity
  function maxMint(address) public view virtual override returns (uint256) {
    if (!_isVaultCollateralized()) return 0;
    uint256 _total = _totalShares();
    return _total >= type(uint96).max ? 0 : type(uint96).max - _total;
  }
```

## Finding 5: Missing access control in `Vault.mintYieldFee` allows any caller to steal accrued yield fee shares

### Synopsis

The `Vault.mintYieldFee` function lacks access control and recipient validation, allowing any unprivileged external caller to mint the vault's accrued `_yieldFeeTotalSupply` shares to an arbitrary `_recipient` address instead of restricting the claim to the designated `_yieldFeeRecipient`.

### Technical details

When yield is liquidated via `Vault.liquidate`, a portion of the yield (determined by `_yieldFeePercentage`) is reserved as a fee by incrementing `_yieldFeeTotalSupply` via `_increaseYieldFeeBalance`. According to the contract's state variables and documentation, these fees are accrued for the designated `_yieldFeeRecipient`:

```solidity
/// @notice Address of the yield fee recipient that receives the fee amount when yield is captured.
address private _yieldFeeRecipient;
```

However, `Vault.mintYieldFee(uint256 _shares, address _recipient)` (lines 394–402) is an `external` function that performs only two checks:
1. `_requireVaultCollateralized()` (line 395), ensuring the vault is collateralized.
2. `if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);` (line 396), ensuring requested shares do not exceed the accrued fee supply.

It never verifies that `msg.sender == _yieldFeeRecipient` (or an authorized role), nor does it enforce that `_recipient` is `_yieldFeeRecipient`. Consequently, anyone can call `mintYieldFee` with `_shares = _yieldFeeTotalSupply` and their own address as `_recipient` to mint all accrued yield fee shares to themselves.

### Assumptions and preconditions

- The vault has a non-zero `_yieldFeePercentage` configured.
- Yield has been liquidated via `Vault.liquidate`, causing `_yieldFeeTotalSupply` to be greater than zero.
- The vault is currently collateralized (`_isVaultCollateralized() == true`).

### Proof of concept

1. Users deposit underlying assets into `Vault`, and the underlying `_yieldVault` accrues yield over time.
2. A liquidation occurs through `Vault.liquidate`, which calls `_increaseYieldFeeBalance` and increases `_yieldFeeTotalSupply` by the fee share amount.
3. Before the legitimate `_yieldFeeRecipient` claims the fees, an unprivileged actor observes the positive `_yieldFeeTotalSupply`.
4. The unprivileged actor calls `vault.mintYieldFee(vault.yieldFeeTotalSupply(), attackerAddress)`.
5. Because `mintYieldFee` has no caller restriction and accepts any `_recipient`, `_yieldFeeTotalSupply` is decremented to `0` and all yield fee shares are minted to `attackerAddress`.
6. The actor can then redeem or withdraw the underlying assets backing those Vault shares, resulting in a complete theft of the protocol's/recipient's accrued yield fees.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Any external account can claim 100% of the accrued `_yieldFeeTotalSupply` at any time by specifying their own address as `_recipient`.
- **Business Implications:** Complete loss of accrued yield fees intended for `_yieldFeeRecipient`. Automated bots can monitor `liquidate` transactions and immediately call `mintYieldFee` in the same block or bundle to continuously siphon all yield fees.

### Remediation

Restrict `mintYieldFee` so that only `_yieldFeeRecipient` can call it (or mint directly to `_yieldFeeRecipient` rather than an arbitrary address if third-party callers are intended).

```solidity
  modifier onlyYieldFeeRecipient() {
    if (msg.sender != _yieldFeeRecipient) revert CallerNotYieldFeeRecipient(msg.sender, _yieldFeeRecipient);
    _;
  }

  function mintYieldFee(uint256 _shares, address _recipient) external onlyYieldFeeRecipient {
    _requireVaultCollateralized();
    if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);

    _yieldFeeTotalSupply -= _shares;
    _mint(_recipient, _shares);

    emit MintYieldFee(msg.sender, _recipient, _shares);
  }
```

Alternatively, if `mintYieldFee` is intended to be callable by anyone on behalf of the fee recipient, remove the `_recipient` parameter and always mint to `_yieldFeeRecipient`:

```solidity
  function mintYieldFee(uint256 _shares) external {
    _requireVaultCollateralized();
    if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);

    _yieldFeeTotalSupply -= _shares;
    _mint(_yieldFeeRecipient, _shares);

    emit MintYieldFee(msg.sender, _yieldFeeRecipient, _shares);
  }
```

## Finding 6: Cross-Contract Reentrancy in `Vault.liquidate` Allows Double Liquidation of Yield and Unbacked Share Minting

### Synopsis

In `Vault.liquidate`, the contract performs an external call to `_yieldVault.deposit` before minting the liquidated shares via `_mint(_account, _amountOut)`. A reentrant call during this external interaction observes a stale `_totalSupply()`, allowing an attacker to liquidate the same yield multiple times and mint unbacked vault shares.

### Technical details

In `Vault.sol`, `liquidate()` calculates the maximum liquidatable yield via `_liquidatableBalanceOf(_tokenOut)`, which derives available yield from `availableYieldBalance()`:

```solidity
function availableYieldBalance() public view returns (uint256) {
  uint256 _assets = _totalAssets();
  uint256 _sharesToAssets = _convertToAssets(_totalShares(), Math.Rounding.Down);

  return _sharesToAssets > _assets ? 0 : _assets - _sharesToAssets;
}
```

Here, `_totalShares()` is the sum of `_totalSupply()` (tracked in `_twabController`) and `_yieldFeeTotalSupply`.

1. `_prizePool.contributePrizeTokens(address(this), _amountIn)` is called at line 570.
2. If `_yieldFeePercentage != 0`, `_increaseYieldFeeBalance` increments `_yieldFeeTotalSupply` at lines 572–576.
3. If the vault holds idle underlying assets (`_vaultAssets != 0 && _amountOut >= _vaultAssets`), `_yieldVault.deposit(_vaultAssets, address(this))` is called at line 581.
4. Finally, `_mint(_account, _amountOut)` is executed at line 584, which calls `_twabController.mint` to increase `_totalSupply()` and updates `_lastRecordedExchangeRate`.

Because `_mint(_account, _amountOut)` happens after `_yieldVault.deposit(_vaultAssets, address(this))`, the vault's `_totalSupply()` is not updated prior to the external call at line 581. As noted in comments in `_deposit` (line 934), the underlying `_asset` can be an ERC777 token that triggers a `tokensToSend` hook on the sender or a hook on the recipient during `deposit`, or `_yieldVault` itself may invoke an external callback.

When control re-enters the liquidation flow (via `LiquidationPair.swapExactAmountIn` or `swapExactAmountOut` calling `Vault.liquidate`) during `_yieldVault.deposit`:
- `_totalSupply()` has not yet been incremented by `_amountOut` from the outer `liquidate` call.
- `_liquidatableBalanceOf` still includes the `_amountOut` yield that is already in the process of being liquidated.
- The reentrant `liquidate` call passes the `_amountOut > _liquidableYield` check at line 567 against stale share accounting, accruing duplicate yield fees into `_yieldFeeTotalSupply` and minting additional shares for the same underlying yield.

### Assumptions and preconditions

- The vault holds a non-zero balance of underlying `asset()` directly (`_vaultAssets != 0`), such as from direct transfers or residual balances, and `_amountOut >= _vaultAssets`.
- Either the underlying `asset()` implements callback hooks (e.g., ERC777 `tokensToSend` / `tokensReceived`) or the configured `_yieldVault` executes an external interaction during `deposit` that allows the caller/liquidator to re-enter `LiquidationPair`.
- The attacker has sufficient `prizeToken` to cover the `amountIn` required by `LiquidationPair` for the reentrant swap(s).

### Proof of concept

1. Assume the `Vault` has `1000e18` underlying assets in `_yieldVault`, `10e18` idle underlying assets held directly in `Vault` (`_vaultAssets = 10e18`), and `_totalSupply() = 1000e18` (so `availableYieldBalance() = 10e18`). Assume `_yieldFeePercentage = 0` for simplicity.
2. The attacker initiates a liquidation via `LiquidationPair` for `_amountOut = 10e18` shares, transferring the required `_amountIn` of `prizeToken` to `PrizePool`.
3. `LiquidationPair` calls `Vault.liquidate(attacker, prizeToken, _amountIn, address(vault), 10e18)`.
4. `Vault.liquidate` verifies `_amountOut (10e18) <= _liquidableYield (10e18)` and calls `_prizePool.contributePrizeTokens(address(this), _amountIn)`.
5. At line 580, `_vaultAssets` is `10e18` and `_amountOut >= _vaultAssets` (`10e18 >= 10e18`), so `Vault` calls `_yieldVault.deposit(10e18, address(this))` at line 581.
6. During `_yieldVault.deposit`, before `_mint` on line 584 is reached, execution re-enters `LiquidationPair` to liquidate another `10e18` of `Vault` shares.
7. In the reentrant `Vault.liquidate` call, `_totalSupply()` is still `1000e18` because the first `_mint` has not executed. `_liquidatableBalanceOf(address(vault))` still returns `10e18`.
8. The reentrant `Vault.liquidate` call succeeds and mints `10e18` shares to the attacker.
9. Execution returns to the outer `Vault.liquidate` call at line 584, which executes `_mint(attacker, 10e18)`, minting an additional `10e18` shares.
10. The attacker receives `20e18` shares backed by only `10e18` of actual yield, diluting existing depositors and leaving the vault undercollateralized (`_totalSupply() = 1020e18` vs `_totalAssets() = 1010e18`).

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: State inconsistency during external calls in `Vault.liquidate` allows `_liquidatableBalanceOf` to be evaluated against stale share supply, resulting in unbacked share minting and inflated `_yieldFeeTotalSupply`.
- **Business Implications**: Violates the core protocol invariant that the liquidator cannot access more than the available yield and that users' principal cannot be diluted or stolen. Existing vault depositors suffer direct loss of principal as the vault becomes undercollateralized, which also bricks subsequent vault operations that depend on `_requireVaultCollateralized()`.

### Remediation

Follow the Checks-Effects-Interactions (CEI) pattern in `Vault.liquidate` by minting the shares (and updating `_totalSupply()` and `_lastRecordedExchangeRate`) before calling `_yieldVault.deposit`, or add a `nonReentrant` guard to `liquidate`.

```solidity
    _prizePool.contributePrizeTokens(address(this), _amountIn);

    if (_yieldFeePercentage != 0) {
      _increaseYieldFeeBalance(
        (_amountOut * FEE_PRECISION) / (FEE_PRECISION - _yieldFeePercentage) - _amountOut
      );
    }

    _mint(_account, _amountOut);

    uint256 _vaultAssets = IERC20(asset()).balanceOf(address(this));

    if (_vaultAssets != 0 && _amountOut >= _vaultAssets) {
      _yieldVault.deposit(_vaultAssets, address(this));
    }
```

## Finding 7: Inconsistent Accounting of Idle Vault Assets in `Vault.liquidate` Causes Permanent Vault Undercollateralization and DoS

### Synopsis

In `Vault.liquidate`, `_liquidatableBalanceOf` includes idle underlying assets held directly by the `Vault` (`super.totalAssets()`), whereas `_currentExchangeRate` only accounts for assets in `_yieldVault` and `liquidate` only deposits idle assets when `_amountOut >= _vaultAssets`, causing liquidations with `_amountOut < _vaultAssets` to permanently undercollateralize and brick the `Vault`.

### Technical details

When `Vault.liquidate` is called, it checks that the requested share output `_amountOut` does not exceed `_liquidableYield`:

```solidity
uint256 _liquidableYield = _liquidatableBalanceOf(_tokenOut);
if (_amountOut > _liquidableYield)
  revert LiquidationAmountOutGTYield(_amountOut, _liquidableYield);
```

`_liquidatableBalanceOf` computes available yield via `availableYieldBalance()`, which calls `_totalAssets()`:

```solidity
function _totalAssets() internal view returns (uint256) {
  return _yieldVault.maxWithdraw(address(this)) + super.totalAssets();
}
```

Here, `super.totalAssets()` is `IERC20(asset()).balanceOf(address(this))` (idle underlying assets held directly in the `Vault`). Thus, any idle `_vaultAssets` in the `Vault` are counted as liquidatable yield in `_liquidableYield`.

However, later in `liquidate` (lines 578–584), the contract only deposits `_vaultAssets` into `_yieldVault` if `_amountOut >= _vaultAssets`, and then immediately mints `_amountOut` shares to `_account`:

```solidity
uint256 _vaultAssets = IERC20(asset()).balanceOf(address(this));

if (_vaultAssets != 0 && _amountOut >= _vaultAssets) {
  _yieldVault.deposit(_vaultAssets, address(this));
}

_mint(_account, _amountOut);
```

`_mint` increases `_totalSupply()` by `_amountOut` and calls `_updateExchangeRate()`, which sets `_lastRecordedExchangeRate = _currentExchangeRate()`. Crucially, `_currentExchangeRate()` (line 1176) only considers `_withdrawableAssets = _yieldVault.maxWithdraw(address(this))` and completely ignores `super.totalAssets()` (`_vaultAssets` sitting in the `Vault`).

There are two common scenarios where `_vaultAssets > 0` and `_amountOut < _vaultAssets` during a valid liquidation (`_amountOut <= _liquidableYield`):
1. **Non-zero yield fee (`_yieldFeePercentage > 0`)**: If yield is held as idle `_vaultAssets` in the `Vault` (with little or no unliquidated yield in `_yieldVault`), `_liquidableYield` is reduced by the yield fee (`_vaultAssets - (_vaultAssets * _yieldFeePercentage) / FEE_PRECISION < _vaultAssets`). Because line 567 enforces `_amountOut <= _liquidableYield`, `_amountOut < _vaultAssets` is **guaranteed** for every valid liquidation call.
2. **Partial liquidation (or intentional attacker dust liquidation)**: Even when `_yieldFeePercentage == 0`, a liquidator or attacker can liquidate a partial amount `_amountOut < _vaultAssets` (e.g., `_amountOut = 1`).

In both cases, `_amountOut >= _vaultAssets` evaluates to `false`, so `_vaultAssets` is not deposited into `_yieldVault`. `_mint(_account, _amountOut)` then increases `_totalSupply()` while `_yieldVault.maxWithdraw(address(this))` remains unchanged. When `_withdrawableAssets < _totalSupply()`, `_currentExchangeRate()` drops below `_assetUnit` and is saved to `_lastRecordedExchangeRate`.

Furthermore, because `_currentExchangeRate()` caps `_withdrawableAssets` at `_totalSupplyToAssets` (lines 1170–1180, computed using `_lastRecordedExchangeRate`), `_currentExchangeRate()` can **never** recover back to `_assetUnit`. The `Vault` is permanently stuck in an undercollateralized state (`_isVaultCollateralized() == false`), causing all future calls to `deposit`, `mint`, `liquidate`, and `mintYieldFee` to revert with `VaultUnderCollateralized`.

### Assumptions and preconditions

1. The `Vault` has non-zero `_totalSupply()` (existing user deposits in `_yieldVault`).
2. A non-zero amount of the underlying `asset()` is held directly by the `Vault` contract (`_vaultAssets = IERC20(asset()).balanceOf(address(this)) > 0`), either via direct transfer/donation to the `Vault` or left over in the contract.
3. A liquidation occurs where `_amountOut < _vaultAssets` and the unliquidated yield inside `_yieldVault` alone is less than `_amountOut`.

### Proof of concept

1. Alice deposits `1000e18` underlying assets into `Vault`. The `Vault` deposits `1000e18` into `_yieldVault` and mints `1000e18` shares to Alice (`_totalSupply() = 1000e18`, `_yieldVault.maxWithdraw(address(vault)) = 1000e18`, `_lastRecordedExchangeRate = 1e18`).
2. `_yieldFeePercentage` is set to `10%` (`1e8`).
3. `10e18` underlying assets are transferred directly to `Vault` (`_vaultAssets = 10e18`), or an attacker donates `10` wei of `asset()` to `Vault`.
4. `availableYieldBalance()` returns `1010e18 - 1000e18 = 10e18`, and `_liquidatableBalanceOf(address(vault))` at line 566 returns `9e18`.
5. A liquidator swaps prize tokens for `_amountOut = 9e18` Vault shares via `LiquidationPair`.
6. In `Vault.liquidate`:
   - Line 567: `_amountOut <= _liquidableYield` (`9e18 <= 9e18`) passes.
   - Line 573: `_yieldFeeTotalSupply` increases by `1e18`.
   - Line 578: `_vaultAssets` is `10e18`.
   - Line 580: `_amountOut >= _vaultAssets` (`9e18 >= 10e18`) is `false`, so `_yieldVault.deposit` is skipped and the `10e18` assets remain idle in `Vault`.
   - Line 584: `_mint` increases `_totalSupply()` to `1009e18` and calls `_updateExchangeRate()`.
7. Inside `_currentExchangeRate()`:
   - `_totalSupplyAmount = 1009e18`.
   - `_withdrawableAssets = _yieldVault.maxWithdraw(address(vault)) = 1000e18`.
   - `_currentExchangeRate()` returns `(1000e18 * 1e18) / 1009e18 = 991080277502477700 < 1e18` (`_assetUnit`).
   - `_lastRecordedExchangeRate` is set to `991080277502477700`.
8. `_isVaultCollateralized()` now returns `false` permanently. All subsequent calls to `deposit`, `mint`, `liquidate`, and `mintYieldFee` revert with `VaultUnderCollateralized()`.

### Impact assessment

**Severity: High**

Any idle underlying assets in the `Vault` (including a 2-wei donation of `asset()` by an attacker followed by a 1-wei liquidation, or normal liquidation of donated yield when `_yieldFeePercentage > 0`) causes `Vault.liquidate` to mint shares without depositing the idle assets into `_yieldVault`. This immediately drops `_lastRecordedExchangeRate` below `_assetUnit`, permanently bricking the `Vault` (`deposit`, `mint`, `liquidate`, `mintYieldFee` all revert) and forcing existing depositors to take a loss on withdrawal.

### Remediation

Either exclude idle `Vault` balances (`super.totalAssets()`) from `_totalAssets()` so they cannot be liquidated as yield until deposited, or deposit all idle `_vaultAssets` into `_yieldVault` unconditionally whenever `_vaultAssets != 0` in `liquidate()` before checking/minting shares:

```solidity
  function liquidate(
    address _account,
    address _tokenIn,
    uint256 _amountIn,
    address _tokenOut,
    uint256 _amountOut
  ) public virtual override returns (bool) {
    _requireVaultCollateralized();
    if (msg.sender != address(_liquidationPair))
      revert LiquidationCallerNotLP(msg.sender, address(_liquidationPair));
    if (_tokenIn != address(_prizePool.prizeToken()))
      revert LiquidationTokenInNotPrizeToken(_tokenIn, address(_prizePool.prizeToken()));
    if (_tokenOut != address(this))
      revert LiquidationTokenOutNotVaultShare(_tokenOut, address(this));
    if (_amountOut == 0) revert LiquidationAmountOutZero();

    uint256 _vaultAssets = IERC20(asset()).balanceOf(address(this));
    if (_vaultAssets != 0) {
      _yieldVault.deposit(_vaultAssets, address(this));
    }

    uint256 _liquidableYield = _liquidatableBalanceOf(_tokenOut);
    if (_amountOut > _liquidableYield)
      revert LiquidationAmountOutGTYield(_amountOut, _liquidableYield);

    _prizePool.contributePrizeTokens(address(this), _amountIn);

    if (_yieldFeePercentage != 0) {
      _increaseYieldFeeBalance(
        (_amountOut * FEE_PRECISION) / (FEE_PRECISION - _yieldFeePercentage) - _amountOut
      );
    }

    _mint(_account, _amountOut);

    return true;
  }
```

## Finding 8: Unverified Prefunded Prize Token Settlement in `Vault.liquidate` Allows Yield Theft and Cross-Vault Contribution Misattribution

### Synopsis

`Vault.liquidate` relies on `PrizePool.contributePrizeTokens` to validate receipt of `_amountIn` prize tokens instead of pulling tokens from the liquidator. Because `PrizePool` tracks a shared global `_deltaBalance` across all vaults, an attacker can exploit unattributed prize tokens in `PrizePool` via `LiquidationPair` to mint vault yield shares without paying.

### Technical details

In PoolTogether V5, `Vault` implements `ILiquidationSource` and sells accrued yield (minted as `Vault` shares, `_tokenOut`) in exchange for `prizeToken` (`_tokenIn`) via a configured `LiquidationPair`.

`LiquidationPair` implements an optimistic, UniswapV2-style swap pattern in `swapExactAmountIn` and `swapExactAmountOut`:
1. Both functions are `external` and permissionless—any caller can invoke `LiquidationPair` directly without going through `LiquidationRouter`.
2. `LiquidationPair` does not transfer `tokenIn` from the caller. Instead, it assumes `tokenIn` has already been sent to `target()` (which is `address(_prizePool)`), and invokes `source.liquidate(_account, tokenIn, _amountIn, tokenOut, _amountOut)`.

In `Vault.liquidate` (`vault/src/Vault.sol`), lines 570–584 execute the settlement:
```solidity
_prizePool.contributePrizeTokens(address(this), _amountIn);
...
_mint(_account, _amountOut);
```

And in `PrizePool.contributePrizeTokens` (`vault/lib/pt-v5-prize-pool/src/PrizePool.sol`, lines 311–315):
```solidity
function contributePrizeTokens(address _prizeVault, uint256 _amount) external returns (uint256) {
  uint256 _deltaBalance = prizeToken.balanceOf(address(this)) - _accountedBalance();
  if (_deltaBalance < _amount) {
    revert ContributionGTDeltaBalance(_amount, _deltaBalance);
  }
  ...
```

This cross-contract settlement design creates three tightly coupled integrity flaws at line 570 of `Vault.sol`:

1. **Prefunded payment attribution failure**: `Vault.liquidate` never pulls `_amountIn` from the swapper or verifies that the caller of `LiquidationPair` actually provided the tokens. It delegates payment verification entirely to `_prizePool.contributePrizeTokens(address(this), _amountIn)`.
2. **Cross-vault interference in shared settlement balances**: `PrizePool` is a singleton shared across all vaults, and `_deltaBalance` is the global difference between `prizeToken.balanceOf(address(prizePool))` and `_accountedBalance()`. If any user, sponsor, or external contract transfers `prizeToken` to `PrizePool` without atomically calling `contributePrizeTokens` in the same transaction, those tokens sit in the shared `_deltaBalance`. Any attacker can then call `LiquidationPair.swapExactAmountOut` (or `swapExactAmountIn`) on any `Vault`'s liquidation pair with `_amountIn <= _deltaBalance` without transferring a single `prizeToken`. `Vault.liquidate` will consume the shared `_deltaBalance` in `PrizePool`, credit the contribution to `address(this)` (`Vault`), and mint `_amountOut` vault shares to the attacker for free.
3. **Nominal-versus-received contribution mismatch**: `Vault.liquidate` passes the nominal `_amountIn` parameter computed by `LiquidationPair` to `contributePrizeTokens` rather than measuring or forwarding actual tokens received for this specific liquidation. If a swapper prefunds `PrizePool` with more than `_amountIn` (e.g., when interacting directly with `LiquidationPair`), only the nominal `_amountIn` is accounted for, leaving the excess unattributed in `_deltaBalance`; conversely, if `_deltaBalance` already contains unattributed tokens, a swapper can transfer less than the nominal `_amountIn` (down to zero) and still pass the `_deltaBalance < _amount` check, or if `prizeToken` charges a transfer fee, the transaction either reverts with `ContributionGTDeltaBalance` or silently consumes preexisting `_deltaBalance` to cover the shortfall.

### Assumptions and preconditions

1. A `Vault` has accrued liquidatable yield (`liquidatableBalanceOf(address(vault)) > 0`) and has a configured `_liquidationPair`.
2. Either (a) `PrizePool` holds any unattributed `prizeToken` balance (`prizeToken.balanceOf(address(prizePool)) > prizePool.accountedBalance()`) due to a direct transfer, non-atomic contribution, or excess transfer, or (b) a caller interacts directly with `LiquidationPair` rather than `LiquidationRouter`.

### Proof of concept

1. Suppose `Vault A` has accrued `100e18` of liquidatable yield, and its `LiquidationPair` requires `_amountIn = 50e18` of `prizeToken` to swap for `_amountOut = 100e18` of `Vault A` shares.
2. A sponsor, user, or non-atomic contract transfers `50e18` of `prizeToken` directly to `PrizePool` (intending to contribute it on behalf of `Vault B`, or as a direct donation to the pool) before `contributePrizeTokens` is called. At this point, `PrizePool._deltaBalance` is `50e18`.
3. An attacker calls `liquidationPairA.swapExactAmountOut(attacker, 100e18, 50e18)` directly on `Vault A`'s `LiquidationPair` without transferring any `prizeToken`.
4. `liquidationPairA` computes `amountIn = 50e18` and calls `vaultA.liquidate(attacker, prizeToken, 50e18, vaultA, 100e18)`.
5. At line 570, `vaultA` calls `_prizePool.contributePrizeTokens(address(vaultA), 50e18)`.
6. `PrizePool.contributePrizeTokens` checks `_deltaBalance = prizeToken.balanceOf(address(prizePool)) - _accountedBalance() == 50e18 >= 50e18`. The check passes, and the `50e18` contribution is credited to `vaultAccumulator[vaultA]` instead of `Vault B`.
7. At line 584, `vaultA` calls `_mint(attacker, 100e18)`, minting `100e18` `Vault A` shares (worth `100e18` underlying assets of yield) to the attacker at zero cost.

### Impact assessment

- **Severity**: Medium / High
- **Direct Technical Effect**: Any unattributed `prizeToken` balance in `PrizePool` can be hijacked by an attacker via `LiquidationPair` to mint `Vault` shares without paying any `prizeToken`, stealing the `Vault`'s accrued yield while misattributing the `PrizePool` contribution to the liquidated `Vault`.
- **Business Implications**: Loss of accrued vault yield to arbitrageurs/attackers, misattribution of prize pool contributions across vaults (corrupting vault portion calculations in `PrizePool._getVaultPortion`), and failure of the settlement invariant that a liquidator must pay `_amountIn` prize tokens to receive `_amountOut` vault shares.

### Remediation

Instead of relying on prefunded balances directly in the shared `PrizePool`, `Vault` should either act as the `targetOf` for `tokenIn` (or pull `tokenIn`), measure the actual `prizeToken` balance held by the `Vault` (or transferred to `PrizePool` via the `Vault`), transfer those tokens to `_prizePool`, and then call `_prizePool.contributePrizeTokens(address(this), _actualAmountIn)`:

```solidity
    IERC20 _prizeToken = _prizePool.prizeToken();
    uint256 _prizePoolBalanceBefore = _prizeToken.balanceOf(address(_prizePool));
    SafeERC20.safeTransfer(_prizeToken, address(_prizePool), _amountIn);
    uint256 _actualAmountIn = _prizeToken.balanceOf(address(_prizePool)) - _prizePoolBalanceBefore;
    _prizePool.contributePrizeTokens(address(this), _actualAmountIn);

    if (_yieldFeePercentage != 0) {
      _increaseYieldFeeBalance(
        (_amountOut * FEE_PRECISION) / (FEE_PRECISION - _yieldFeePercentage) - _amountOut
      );
    }

    uint256 _vaultAssets = IERC20(asset()).balanceOf(address(this));

    if (_vaultAssets != 0 && _amountOut >= _vaultAssets) {
      _yieldVault.deposit(_vaultAssets, address(this));
    }

    _mint(_account, _amountOut);
```

*(Note: With this approach, `Vault.targetOf` should return `address(this)` so `LiquidationRouter` sends `_tokenIn` to the `Vault` rather than directly to `_prizePool`, isolating each vault's settlement balance from `PrizePool`'s shared balance.)*

## Finding 9: Rounding Asymmetry in `Vault._availableYieldFeeBalance` and `Vault._liquidatableBalanceOf` Causes Over-Allocation of Yield Shares and Vault Undercollateralization

### Synopsis

In `Vault._availableYieldFeeBalance`, the yield fee is rounded down via integer division, causing `Vault._liquidatableBalanceOf` to round the liquidatable yield up. When this maximum liquidatable yield is liquidated in `Vault.liquidate`, the total shares allocated (liquidated shares plus yield fee shares) exceed the actual available yield, rendering the vault undercollateralized.

### Technical details

The `Vault` contract accrues yield from `_yieldVault` and splits the available yield (`availableYieldBalance()`) between liquidatable yield and a protocol yield fee according to `_yieldFeePercentage` (with precision `FEE_PRECISION = 1e9`).

In `Vault._availableYieldFeeBalance` (line 845), the fee portion of `_availableYield` is computed using standard integer division, which rounds down:

```solidity
function _availableYieldFeeBalance(uint256 _availableYield) internal view returns (uint256) {
  return (_availableYield * _yieldFeePercentage) / FEE_PRECISION;
}
```

In `Vault._liquidatableBalanceOf` (lines 829–837), this rounded-down fee is subtracted from `_availableYield`:

```solidity
function _liquidatableBalanceOf(address _token) internal view returns (uint256) {
  if (_token != address(this)) revert LiquidationTokenOutNotVaultShare(_token, address(this));

  uint256 _availableYield = availableYieldBalance();

  unchecked {
    return _availableYield -= _availableYieldFeeBalance(_availableYield);
  }
}
```

Because `_availableYieldFeeBalance(_availableYield)` truncates toward zero, subtracting it from `_availableYield` effectively rounds `_liquidatableBalanceOf` **up**:

$$\text{liquidableYield} = Y - \left\lfloor \frac{Y \cdot f}{P} \right\rfloor = \left\lceil \frac{Y \cdot (P - f)}{P} \right\rceil$$

where $Y = \text{\_availableYield}$, $f = \text{\_yieldFeePercentage}$, and $P = \text{FEE\_PRECISION} = 10^9$.

However, when `Vault.liquidate` is executed for `_amountOut` (up to `_liquidableYield`), it reconstructs the yield fee shares from `_amountOut` on lines 572–576:

```solidity
if (_yieldFeePercentage != 0) {
  _increaseYieldFeeBalance(
    (_amountOut * FEE_PRECISION) / (FEE_PRECISION - _yieldFeePercentage) - _amountOut
  );
}
```

The total number of shares allocated by `liquidate` (the `_amountOut` shares minted to `_account` plus the fee shares added to `_yieldFeeTotalSupply`) is:

$$\text{totalAllocatedShares} = \left\lfloor \frac{\text{amountOut} \cdot P}{P - f} \right\rfloor$$

Whenever $(Y \cdot f) \bmod P \ge P - f$, substituting $\text{amountOut} = \left\lceil \frac{Y \cdot (P - f)}{P} \right\rceil$ yields:

$$\text{totalAllocatedShares} = Y + \left\lfloor \frac{(Y \cdot f) \bmod P}{P - f} \right\rfloor > Y$$

Consequently, `liquidate` allocates more total shares (`_amountOut + feeShares`) than the underlying yield $Y$ available in the vault. Once `mintYieldFee` is called to mint the accrued `_yieldFeeTotalSupply`, `_totalSupply()` exceeds `_yieldVault.maxWithdraw(address(this))`. Inside `_mint`, `_updateExchangeRate()` records a `_lastRecordedExchangeRate` strictly less than `_assetUnit`.

Furthermore, because `_currentExchangeRate()` caps `_withdrawableAssets` to `_totalSupplyToAssets` (computed using `_lastRecordedExchangeRate`), once `_lastRecordedExchangeRate` drops below `_assetUnit`, `_currentExchangeRate()` can never recover to `_assetUnit` via subsequent yield accrual. `_isVaultCollateralized()` remains false, causing `_requireVaultCollateralized()` to revert on all future calls to `deposit`, `mint`, `sponsor`, `liquidate`, and `mintYieldFee`.

### Assumptions and preconditions

- The vault has a non-zero `_yieldFeePercentage` configured (`0 < _yieldFeePercentage < FEE_PRECISION`).
- The accrued yield `_availableYield` satisfies `(_availableYield * _yieldFeePercentage) % FEE_PRECISION >= FEE_PRECISION - _yieldFeePercentage` (for example, with a 10% fee where `_yieldFeePercentage = 1e8`, any `_availableYield` ending in `9` such as `10e18 + 9` satisfies `(9 * 1e8) % 1e9 == 9e8 >= 9e8`).
- A liquidator liquidates the full `_liquidatableBalanceOf(address(vault))` (which is the standard behavior of `LiquidationPair.maxAmountOut()`) and the yield fee is subsequently minted via `mintYieldFee`.

### Proof of concept

1. Assume the vault has `1000e18` underlying assets deposited (`_totalSupply() == 1000e18`, `_lastRecordedExchangeRate == 1e18`) and `_yieldFeePercentage = 1e8` (10%, with `FEE_PRECISION = 1e9`).
2. The yield vault accrues `19` wei of yield, so `_totalAssets() = 1000e18 + 19` and `availableYieldBalance() = 19`.
3. `_availableYieldFeeBalance(19)` computes `(19 * 1e8) / 1e9 = 1`.
4. `_liquidatableBalanceOf(address(vault))` computes `19 - 1 = 18`.
5. A user liquidates the available yield by swapping for `_amountOut = 18` vault shares through `LiquidationPair`.
6. Inside `Vault.liquidate`:
   - `_amountOut <= _liquidableYield` (`18 <= 18`) passes.
   - `_increaseYieldFeeBalance` adds `(18 * 1e9) / (1e9 - 1e8) - 18 = 20 - 18 = 2` shares to `_yieldFeeTotalSupply`.
   - `_mint` mints `18` shares to the liquidator (`_totalSupply()` becomes `1000e18 + 18`).
   - Total shares backed by the `19` wei of yield are now `18 + 2 = 20` wei (1 wei more than the vault's actual assets).
7. Any caller invokes `mintYieldFee(2, yieldFeeRecipient)`:
   - `_yieldFeeTotalSupply` decreases by `2` and `_mint(yieldFeeRecipient, 2)` increases `_totalSupply()` to `1000e18 + 20`.
   - `_updateExchangeRate()` calls `_currentExchangeRate()`. Since `_withdrawableAssets` (`1000e18 + 19`) is strictly less than `_totalSupplyAmount` (`1000e18 + 20`), `_currentExchangeRate()` rounds down to `((1000e18 + 19) * 1e18) / (1000e18 + 20) = 1e18 - 1 < _assetUnit`.
8. The vault is now undercollateralized (`_isVaultCollateralized() == false`), causing all subsequent calls to `deposit`, `mint`, `sponsor`, `liquidate`, and `mintYieldFee` to revert with `VaultUnderCollateralized()`.

### Impact assessment

**Severity: High**

The rounding asymmetry between `_availableYieldFeeBalance` / `_liquidatableBalanceOf` and `liquidate` allows normal yield liquidations to allocate more shares than available underlying assets. Once `mintYieldFee` is called, the vault's exchange rate drops below `1:1` (`_assetUnit`), putting the vault into a persistent undercollateralized state. Because `_currentExchangeRate()` caps withdrawable assets at `_totalSupplyToAssets` based on `_lastRecordedExchangeRate`, future yield cannot restore the exchange rate back to `_assetUnit`, resulting in a permanent Denial of Service on deposits, mints, sponsorships, liquidations, and fee minting, as well as a slight loss of user funds on withdrawal.

### Remediation

Round the yield fee calculation **up** in `_availableYieldFeeBalance` (or equivalently round `_liquidatableBalanceOf` down) using `Math.mulDiv` with `Math.Rounding.Up` so that `_amountOut + feeShares` in `liquidate` never exceeds `_availableYield`.

```solidity
  function _liquidatableBalanceOf(address _token) internal view returns (uint256) {
    if (_token != address(this)) revert LiquidationTokenOutNotVaultShare(_token, address(this));

    uint256 _availableYield = availableYieldBalance();

    unchecked {
      return _availableYield - _availableYieldFeeBalance(_availableYield);
    }
  }

  /**
   * @notice Available yield fee amount.
   * @param _availableYield Total amount of yield available
   * @return uint256 Available yield fee balance
   */
  function _availableYieldFeeBalance(uint256 _availableYield) internal view returns (uint256) {
    return _availableYield.mulDiv(_yieldFeePercentage, FEE_PRECISION, Math.Rounding.Up);
  }
```

## Finding 10: Inconsistent `_yieldFeeTotalSupply` Accounting in `Vault._increaseYieldFeeBalance` and `Vault._convertToShares` Distorts Exchange Rate and Collateralization Checks

### Synopsis

`Vault._increaseYieldFeeBalance` accrues unminted yield fee shares in `_yieldFeeTotalSupply`, while `Vault._convertToShares` relies on `_currentExchangeRate()`, which divides withdrawable assets (including assets backing `_yieldFeeTotalSupply`) by `_totalSupply()` instead of `_totalShares()`. This backing-versus-liability mismatch distorts share conversions and allows `mintYieldFee` to trigger undercollateralization.

### Technical details

When yield is liquidated in `Vault.liquidate()`, the protocol's yield fee share allocation is accrued via `_increaseYieldFeeBalance()` at line 853:

```solidity
_yieldFeeTotalSupply += _shares;
```

These accrued fee shares represent a claim on the underlying assets held in `_yieldVault`, and `Vault._totalShares()` (lines 818–820) defines the total share liabilities managed by the vault as:

```solidity
function _totalShares() internal view returns (uint256) {
  return _totalSupply() + _yieldFeeTotalSupply;
}
```

While `availableYieldBalance()` (lines 311–316) uses `_totalShares()` to account for both minted shares (`_totalSupply()`) and unminted yield fee shares (`_yieldFeeTotalSupply`), `Vault._convertToShares()` (lines 864–874) converts assets to shares using `_exchangeRate = _currentExchangeRate()` (line 868):

```solidity
function _convertToShares(
  uint256 _assets,
  Math.Rounding _rounding
) internal view virtual override returns (uint256) {
  uint256 _exchangeRate = _currentExchangeRate();

  return
    (_assets == 0 || _exchangeRate == 0)
      ? _assets
      : _assets.mulDiv(_assetUnit, _exchangeRate, _rounding);
}
```

Inside `_currentExchangeRate()` (lines 1168–1187), the vault computes the exchange rate using only `_totalSupplyAmount = _totalSupply()` (which queries `_twabController.totalSupply(address(this))` and excludes `_yieldFeeTotalSupply`), while `_withdrawableAssets = _yieldVault.maxWithdraw(address(this))` includes the underlying assets backing both `_totalSupply()` and `_yieldFeeTotalSupply`.

This creates two closely related accounting failures:

1. **Masked Undercollateralization and State-Transition-Dependent Dilution:**
   When `_yieldFeeTotalSupply > 0` and `_yieldVault` suffers a loss smaller than or equal to the assets backing `_yieldFeeTotalSupply`, `_withdrawableAssets` remains greater than or equal to `_totalSupplyToAssets` (which only prices `_totalSupply()`). Consequently, `_currentExchangeRate()` remains at `_assetUnit` (`10 ** decimals`), and `_isVaultCollateralized()` returns `true` even though total backing (`_withdrawableAssets`) is less than total share liabilities (`_totalShares()`).
   - Callers of `deposit()` and `withdraw()` (which invoke `_convertToShares()` at lines 864–874) transact at an overstated `1:1` exchange rate, allowing early withdrawers to exit without bearing their pro-rata share of the loss.
   - Because `_isVaultCollateralized()` is still `true`, `mintYieldFee()` passes `_requireVaultCollateralized()` (line 395), subtracts `_shares` from `_yieldFeeTotalSupply`, and calls `_mint(_recipient, _shares)` (line 399). This moves the fee shares into `_twabController.totalSupply(address(this))` without adding any assets and calls `_updateExchangeRate()`, immediately dropping `_currentExchangeRate()` and `_lastRecordedExchangeRate` below `_assetUnit`, socializing the loss onto remaining depositors and putting the vault into an undercollateralized state.

2. **Exchange-Rate Divergence Before vs. After `mintYieldFee()`:**
   If `_yieldVault` loses more than the assets backing `_yieldFeeTotalSupply`, share pricing in `_convertToShares()` depends on whether `mintYieldFee()` was called prior to the loss. If `mintYieldFee()` was called before the loss, `_totalSupply()` is `S + F` and the exchange rate is `_withdrawableAssets * _assetUnit / (S + F)`. If `_yieldFeeTotalSupply` remained unminted at line 853, `_currentExchangeRate()` divides `_withdrawableAssets` by `S` instead of `S + F`, yielding a higher exchange rate and causing `withdraw()` (via `_convertToShares(_assets, Math.Rounding.Up)`) to burn fewer shares for the same asset withdrawal.

### Assumptions and preconditions

1. The vault has a non-zero `_yieldFeePercentage` configured.
2. Yield has been liquidated via `liquidate()`, accruing a positive `_yieldFeeTotalSupply` balance at line 853.
3. `_yieldVault.maxWithdraw(address(this))` decreases due to a loss or reduced withdrawable liquidity in the underlying `_yieldVault`.

### Proof of concept

1. **Initial Deposit:** Alice deposits `1,000e18` underlying assets into `Vault`.
   - `_totalSupply() = 1,000e18`
   - `_withdrawableAssets = 1,000e18`
   - `_lastRecordedExchangeRate = 1e18`
2. **Yield Accrual and Liquidation:** `_yieldVault` accrues `100e18` in yield (`_withdrawableAssets = 1,100e18`). With `_yieldFeePercentage = 1e8` (10%), a liquidator calls `liquidate()` for `_amountOut = 90e18` shares:
   - `_increaseYieldFeeBalance(10e18)` increments `_yieldFeeTotalSupply` to `10e18` (line 853).
   - `_mint(liquidator, 90e18)` increases `_totalSupply()` to `1,090e18` and records `_lastRecordedExchangeRate = 1e18`.
   - Total share liabilities are `_totalShares() = 1,090e18 + 10e18 = 1,100e18`, backed by `1,100e18` assets in `_yieldVault`.
3. **Partial Loss in `_yieldVault`:** `_yieldVault` loses `10e18` assets, reducing `_withdrawableAssets` to `1,090e18`.
   - True backing per share across all `1,100e18` shares (`_totalShares()`) is `1,090e18 * 1e18 / 1,100e18 = 0.990909090909090909e18`.
   - However, `_currentExchangeRate()` computes `_totalSupplyAmount = _totalSupply() = 1,090e18` and `_totalSupplyToAssets = 1,090e18`. Because `_withdrawableAssets == 1,090e18`, `_currentExchangeRate()` returns `1e18`.
4. **Mispriced Withdrawal / Uncollateralized Fee Mint:**
   - The liquidator calls `withdraw(90e18, liquidator, liquidator)`. `withdraw()` calls `_convertToShares(90e18, Math.Rounding.Up)` (line 864), which uses `_exchangeRate = 1e18` (line 868) and burns `90e18` shares for `90e18` full assets, escaping the loss completely.
   - Next, `mintYieldFee(10e18, feeRecipient)` is called. Because `_currentExchangeRate()` is still `1e18` (`1,000e18` assets for `1,000e18` `_totalSupply()`), `_requireVaultCollateralized()` succeeds.
   - `_mint(feeRecipient, 10e18)` increases `_totalSupply()` to `1,010e18` while `_withdrawableAssets` is only `1,000e18`, dropping `_currentExchangeRate()` to `1,000e18 * 1e18 / 1,010e18 = 0.990099009900990099e18`. Alice bears the entire `10e18` loss on her principal, and the vault becomes undercollateralized.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** `_currentExchangeRate()` and `_convertToShares()` overstate the vault's exchange rate and collateralization status whenever `_yieldFeeTotalSupply > 0` and `_yieldVault` experiences a loss.
- **Business Impact:** Early withdrawers can withdraw full asset amounts without sharing losses, `mintYieldFee()` can push a purportedly collateralized vault into an undercollateralized state (halting deposits, mints, and liquidations), and remaining depositors suffer disproportionate principal loss.

### Remediation

Ensure that unminted yield fee shares are accounted for consistently with minted shares when computing the exchange rate and converting assets to shares—either by minting yield fee shares directly to `_yieldFeeRecipient` (or a dedicated fee escrow) during `_increaseYieldFeeBalance`, or by accounting for `_yieldFeeTotalSupply` in the exchange rate calculation used by `_convertToShares` and `_convertToAssets`.

If yield fee shares are minted directly when accrued so that `_totalSupply()` always reflects total share liabilities:

```solidity
  function _increaseYieldFeeBalance(uint256 _shares) internal {
    _mint(_yieldFeeRecipient, _shares);
  }
```

Alternatively, if `_yieldFeeTotalSupply` remains lazily minted, include `_yieldFeeTotalSupply` (via `_totalShares()`) in `_currentExchangeRate()` when determining collateralization and share valuation so that `_convertToShares()` always reflects the full share liability:

```solidity
  function _convertToShares(
    uint256 _assets,
    Math.Rounding _rounding
  ) internal view virtual override returns (uint256) {
    uint256 _exchangeRate = _currentExchangeRate();

    return
      (_assets == 0 || _exchangeRate == 0)
        ? _assets
        : _assets.mulDiv(_assetUnit, _exchangeRate, _rounding);
  }
```
(with `_currentExchangeRate()` updated to use `_totalShares()` instead of `_totalSupply()`).

## Finding 11: Incorrect Zero Exchange Rate Handling in `Vault._convertToShares` and `Vault._convertToAssets` Allows Draining Remaining Assets During Severe Undercollateralization

### Synopsis

In `Vault._convertToShares` (and `Vault._convertToAssets`), an `_exchangeRate` of `0` is treated as a 1:1 conversion rate rather than a near-zero rate. When severe undercollateralization causes `_currentExchangeRate()` to round down to `0`, a single shareholder can withdraw all remaining vault assets by burning a negligible number of shares.

### Technical details

The `Vault` contract calculates its share-to-asset exchange rate in `_currentExchangeRate()` (lines 1168–1187):

```solidity
if (_totalSupplyAmount != 0 && _withdrawableAssets != 0) {
  return _withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down);
}

return _assetUnit;
```

When the vault is empty (`_totalSupplyAmount == 0`) or completely depleted (`_withdrawableAssets == 0`), `_currentExchangeRate()` already defaults to `_assetUnit` (`10 ** decimals()`, representing a 1:1 exchange rate). Therefore, `_currentExchangeRate()` only returns `0` when both `_totalSupplyAmount != 0` and `_withdrawableAssets != 0`, but the vault is severely undercollateralized such that:

$$\text{\_withdrawableAssets} \times \text{\_assetUnit} < \text{\_totalSupplyAmount}$$

However, `_convertToShares` (lines 864–874) and `_convertToAssets` (lines 896–905) handle `_exchangeRate == 0` as if the vault had a 1:1 exchange rate:

```solidity
function _convertToShares(
  uint256 _assets,
  Math.Rounding _rounding
) internal view virtual override returns (uint256) {
  uint256 _exchangeRate = _currentExchangeRate();

  return
    (_assets == 0 || _exchangeRate == 0)
      ? _assets
      : _assets.mulDiv(_assetUnit, _exchangeRate, _rounding);
}
```

When `_exchangeRate == 0`:
1. `_convertToShares(_assets, _rounding)` discontinuously returns `_assets` instead of reverting or scaling proportionally, valuing shares at a 1:1 rate with assets.
2. Similarly, `_convertToAssets(_shares, _exchangeRate, _rounding)` returns `_shares` at a 1:1 rate.
3. In `withdraw(_assets, _receiver, _owner)` (lines 509–521) and `redeem(_shares, _receiver, _owner)` (lines 524–535), a user holding at least `_withdrawableAssets` shares (which is less than `_totalSupplyAmount / _assetUnit` of the total supply) can withdraw the entire `_withdrawableAssets` balance of the vault by burning only `_withdrawableAssets` shares at a 1:1 ratio.
4. Additionally, if `_updateExchangeRate()` (lines 1109–1112) is triggered while `_currentExchangeRate() == 0`, `_lastRecordedExchangeRate` is set to `0`. Subsequent calls to `_currentExchangeRate()` will compute `_totalSupplyToAssets = _convertToAssets(_totalSupplyAmount, 0, Math.Rounding.Down) = _totalSupplyAmount`, erasing the previously recorded undercollateralization cap.

### Assumptions and preconditions

1. The `Vault` has non-zero share supply (`_totalSupplyAmount > 0`).
2. The underlying `_yieldVault` experiences a loss (or share supply is large relative to asset decimals, e.g., a 6-decimal token like USDC where `_assetUnit = 1e6`) such that `0 < _withdrawableAssets * _assetUnit < _totalSupplyAmount`.
3. An user or attacker holds at least `_withdrawableAssets` shares.

### Proof of concept

1. Assume an underlying asset with 6 decimals (`_assetUnit = 1e6`, e.g., USDC).
2. Alice deposits `10,000,000e6` USDC (`1e13` base units) and receives `1e13` vault shares. Bob deposits `10,000e6` USDC (`1e10` base units) and receives `1e10` vault shares. Total supply is `_totalSupplyAmount = 1.001e13`.
3. The `_yieldVault` suffers a severe loss, leaving `_withdrawableAssets = 9,000` base units (`0.009` USDC) withdrawable by the `Vault`.
4. `_currentExchangeRate()` computes `9,000 * 1e6 / 1.001e13 = 0`.
5. Because `_exchangeRate == 0`, `maxWithdraw(bob)` calls `_convertToAssets(1e10, 0, Math.Rounding.Down)` which returns `1e10`.
6. Bob calls `withdraw(9,000, bob, bob)`. `withdraw` computes `_shares = _convertToShares(9,000, Math.Rounding.Up) = 9,000`.
7. Bob burns only `9,000` shares (out of his `1e10` shares) and receives all `9,000` remaining underlying asset units, leaving `0` withdrawable assets for Alice despite Alice owning 99.9% of the vault shares.

### Impact assessment

- **Severity**: Medium / High
- **Direct Technical Effect**: Discontinuous jump in share-to-asset and asset-to-share conversion rates from near-zero to 1:1 when `_currentExchangeRate()` rounds down to `0`, and corruption of `_lastRecordedExchangeRate`.
- **Business Implications**: Instead of distributing remaining assets proportionally among shareholders during an undercollateralization event (as required by the protocol's undercollateralization invariant), the first user to withdraw can drain 100% of the remaining withdrawable assets in the yield vault using a minuscule amount of shares.

### Remediation

In `_convertToShares` and `_convertToAssets`, do not fallback to a 1:1 conversion when `_exchangeRate == 0`, or calculate conversions directly from `_totalSupply()` and withdrawable assets without intermediate truncation to `_assetUnit`. If keeping the `_exchangeRate` architecture, return `0` in `_convertToAssets` when `_exchangeRate == 0` and revert (or return `0`) in `_convertToShares` when `_exchangeRate == 0`:

```solidity
  error ZeroExchangeRate();

  function _convertToShares(
    uint256 _assets,
    Math.Rounding _rounding
  ) internal view virtual override returns (uint256) {
    uint256 _exchangeRate = _currentExchangeRate();
    if (_exchangeRate == 0) revert ZeroExchangeRate();

    return
      (_assets == 0)
        ? 0
        : _assets.mulDiv(_assetUnit, _exchangeRate, _rounding);
  }
```

## Finding 12: Cumulative Rounding Drift in `Vault._updateExchangeRate` Decays the Exchange Rate Across Checkpoints

### Synopsis

In `Vault._updateExchangeRate`, checkpointing `_lastRecordedExchangeRate` to `_currentExchangeRate()` repeatedly converts the previous exchange rate down to `_totalSupplyToAssets` (`Math.Rounding.Down`) and then converts that truncated asset amount back into a new exchange rate (`Math.Rounding.Down`). Across successive checkpoints, this double-floor cycle causes cumulative downward drift in `_lastRecordedExchangeRate`, progressively reducing withdrawable user assets.

### Technical details

`Vault._updateExchangeRate()` (lines 1109–1112) is invoked on every share mint (`_mint`, line 1124) and share burn (`_burn`, line 1140) to checkpoint `_lastRecordedExchangeRate`:

```solidity
function _updateExchangeRate() internal {
  _lastRecordedExchangeRate = _currentExchangeRate();
  emit RecordedExchangeRate(_lastRecordedExchangeRate);
}
```

When `_currentExchangeRate()` (lines 1168–1187) evaluates the current rate from the checkpointed `_lastRecordedExchangeRate`, it performs two sequential floor divisions:

1. It computes the maximum backed assets `_totalSupplyToAssets` from the current `_totalSupplyAmount` ($S$) and `_lastRecordedExchangeRate` ($R_{\text{last}}$) using `Math.Rounding.Down` (lines 1170–1174):
   $$A_{\text{cap}} = \left\lfloor \frac{S \times R_{\text{last}}}{\text{\_assetUnit}} \right\rfloor$$
2. If `_withdrawableAssets` from `_yieldVault.maxWithdraw(address(this))` exceeds $A_{\text{cap}}$, `_withdrawableAssets` is clamped down to $A_{\text{cap}}$ (lines 1178–1180).
3. It then recomputes the exchange rate from `_withdrawableAssets` and $S$, again using `Math.Rounding.Down` (line 1183):
   $$R_{\text{new}} = \left\lfloor \frac{\text{\_withdrawableAssets} \times \text{\_assetUnit}}{S} \right\rfloor$$

Whenever the vault is undercollateralized ($R_{\text{last}} < \text{\_assetUnit}$) or `_withdrawableAssets` is initially slightly below $S$ (e.g., due to 1-wei ERC-4626 rounding in `_yieldVault.maxWithdraw`), $R_{\text{new}}$ is floored:
$$R_{\text{new}} = \left\lfloor \frac{A_0 \times \text{\_assetUnit}}{S} \right\rfloor \implies S \times R_{\text{new}} = A_0 \times \text{\_assetUnit} - r_1$$
Whenever $A_0 \times \text{\_assetUnit}$ is not an exact multiple of $S$ ($1 \le r_1 < S$), storing $R_{\text{new}}$ as `_lastRecordedExchangeRate` in `_updateExchangeRate()` causes the very next evaluation of `_totalSupplyToAssets` (at the same supply $S$, or after a small burn) to truncate down by at least 1 unit of assets:
$$A_1 = \left\lfloor \frac{S \times R_{\text{new}}}{\text{\_assetUnit}} \right\rfloor = \left\lfloor \frac{A_0 \times \text{\_assetUnit} - r_1}{\text{\_assetUnit}} \right\rfloor = A_0 - \left\lceil \frac{r_1}{\text{\_assetUnit}} \right\rceil \le A_0 - 1$$
And clamping `_withdrawableAssets` to $A_1$ in turn produces an even smaller exchange rate $R_2 < R_{\text{new}}$ on the next `_updateExchangeRate()` call:
$$R_2 = \left\lfloor \frac{A_1 \times \text{\_assetUnit}}{S} \right\rfloor < R_{\text{new}}$$

Because `withdraw(0, receiver, owner)` and `redeem(0, receiver, owner)` execute `_burn(owner, 0)` without reverting in `TwabController`, any caller can repeatedly trigger `_updateExchangeRate()` without holding or burning shares, ratcheting `_lastRecordedExchangeRate` downward step by step.

### Assumptions and preconditions

1. The `Vault` has non-zero `_totalSupply()` ($S > 0$) and `_withdrawableAssets > 0`.
2. `_lastRecordedExchangeRate < _assetUnit` (or `_withdrawableAssets < _totalSupplyToAssets` due to a loss or rounding in `_yieldVault`), such that `_withdrawableAssets * _assetUnit` is not an exact multiple of `_totalSupplyAmount`.

### Proof of concept

1. Consider a `Vault` with `_assetUnit = 1e18`, `_totalSupplyAmount` $S = 99$, and `_withdrawableAssets` $= 98$ after an undercollateralization event (or scale both by $10^{18}$ with a non-dividing share supply such as $S = 99 \times 10^{18} + 1$).
2. On the first checkpoint update via `_updateExchangeRate()`, `_currentExchangeRate()` computes:
   $$R_1 = \left\lfloor \frac{98 \times 10^{18}}{99} \right\rfloor = 989898989898989898$$
   and stores `_lastRecordedExchangeRate = 989898989898989898`.
3. On the next call that triggers `_updateExchangeRate()` (such as `redeem(0, user, user)` or a subsequent withdrawal), `_currentExchangeRate()` computes:
   $$\text{\_totalSupplyToAssets} = \left\lfloor \frac{99 \times 989898989898989898}{10^{18}} \right\rfloor = \left\lfloor \frac{97999999999999999902}{10^{18}} \right\rfloor = 97$$
4. Even though `_yieldVault.maxWithdraw(address(this))` is still $98$, line 1178 clamps `_withdrawableAssets` to $97$, and line 1183 computes:
   $$R_2 = \left\lfloor \frac{97 \times 10^{18}}{99} \right\rfloor = 979797979797979797$$
5. `_updateExchangeRate()` overwrites `_lastRecordedExchangeRate` with $R_2$. Each subsequent checkpoint call reduces `_totalSupplyToAssets` by at least 1 unit of underlying assets and further decays `_lastRecordedExchangeRate`.

### Impact assessment

- **Severity**: Medium / High.
- **Technical Impact**: `_lastRecordedExchangeRate` and `_totalSupplyToAssets` decay across checkpoints even when no additional loss occurs in `_yieldVault`.
- **Business Impact**: Existing shareholders' `maxWithdraw` and `previewRedeem` values are progressively reduced, locking their underlying principal in the vault (or leaving it only for the final withdrawer when `_totalSupply()` reaches zero).

### Remediation

In `_updateExchangeRate()` (and its rate calculation), avoid overwriting `_lastRecordedExchangeRate` with a doubly-floored recomputation when `_withdrawableAssets >= _totalSupplyToAssets`, or compute `_totalSupplyToAssets` using `Math.Rounding.Up` when checking whether `_withdrawableAssets` has fallen below the last recorded exchange rate so that a checkpointed rate does not decay itself.

```solidity
  /// @notice Update exchange rate with the current exchange rate.
  function _updateExchangeRate() internal {
    uint256 _totalSupplyAmount = _totalSupply();
    uint256 _withdrawableAssets = _yieldVault.maxWithdraw(address(this));

    if (_totalSupplyAmount == 0 || _withdrawableAssets == 0) {
      _lastRecordedExchangeRate = _assetUnit;
    } else {
      uint256 _totalSupplyToAssets = _convertToAssets(
        _totalSupplyAmount,
        _lastRecordedExchangeRate,
        Math.Rounding.Up
      );

      if (_withdrawableAssets < _totalSupplyToAssets) {
        _lastRecordedExchangeRate = _withdrawableAssets.mulDiv(
          _assetUnit,
          _totalSupplyAmount,
          Math.Rounding.Down
        );
      }
    }

    emit RecordedExchangeRate(_lastRecordedExchangeRate);
  }
```

## Finding 13: Unsafe Downcast of `_shares` to `uint96` in `Vault._burn` and `Vault._mint` Enables Theft of Vault Assets and Share Accounting Corruption

### Synopsis

`Vault._burn` and `Vault._mint` unsafely downcast the `uint256 _shares` parameter to `uint96` when calling `_twabController.burn` and `_twabController.mint`. Because `TwabController` stores balances as `uint112`, an account holding more than `type(uint96).max` shares can redeem or withdraw full underlying assets while burning almost zero shares, draining the vault.

### Technical details

In `Vault.sol`, share balances and total supply are tracked externally by `TwabController`. Within `TwabLib.AccountDetails`, both `balance` and `delegateBalance` are stored as `uint112` integers (`uint112 balance; uint112 delegateBalance;`), while `TwabController.mint` and `TwabController.burn` accept a `uint96 _amount` parameter.

In `Vault.sol`, `_mint` (lines 1122–1127) and `_burn` (lines 1138–1143) accept `uint256 _shares` and perform a raw typecast `uint96(_shares)` in Solidity 0.8.17, which silently truncates the higher-order bits modulo $2^{96}$ without reverting on overflow:

```solidity
function _mint(address _receiver, uint256 _shares) internal virtual override {
  _twabController.mint(_receiver, uint96(_shares));
  _updateExchangeRate();

  emit Transfer(address(0), _receiver, _shares);
}

function _burn(address _owner, uint256 _shares) internal virtual override {
  _twabController.burn(_owner, uint96(_shares));
  _updateExchangeRate();

  emit Transfer(_owner, address(0), _shares);
}
```

Although `maxDeposit` and `maxMint` return `type(uint96).max` per single deposit/mint transaction, a user can accumulate a share balance greater than `type(uint96).max` (up to `type(uint112).max`) in `TwabController` across two or more `deposit`, `mint`, or `transfer` calls.

When the user subsequently calls `redeem` or `withdraw`:
1. `Vault.redeem` checks `if (_shares > maxRedeem(_owner))` (line 529). Inherited `ERC4626.maxRedeem(_owner)` returns `balanceOf(_owner)`, which queries `_twabController.balanceOf(address(this), _owner)` and returns the user's `uint112` balance (which can exceed `type(uint96).max`).
2. `Vault.redeem` computes `_assets = _convertToAssets(_shares, Math.Rounding.Down)` (line 531) using the full untruncated `uint256 _shares`. Similarly, `Vault.withdraw` checks `maxWithdraw(_owner)` (which converts the full `uint112` balance to assets) and computes `_shares = _convertToShares(_assets, Math.Rounding.Up)` (line 517), which exceeds `type(uint96).max` when `_assets` corresponds to more than `type(uint96).max` shares.
3. `_withdraw` calls `_burn(_owner, _shares)` (line 1024), which executes `_twabController.burn(_owner, uint96(_shares))` (line 1139). Because of the unsafe cast, only `uint96(_shares)` (i.e., `_shares mod 2^96`) shares are deducted from `_owner` and total supply in `TwabController`.
4. `_withdraw` then withdraws the full untruncated `_assets` from `_yieldVault` and transfers them to `_receiver` (lines 1026–1027), and `_burn` emits a `Transfer` event with the untruncated `_shares` amount (line 1142).

Similarly, in `_mint` (line 1123), callers that do not cap `_shares` at `type(uint96).max`—such as `liquidate` (line 584), `mintYieldFee` (line 399), or `deposit` when the exchange rate is below `1` (Wait: `maxDeposit` is `0` when undercollateralized, so via `liquidate` or `mintYieldFee`)—will silently truncate the minted shares in `TwabController` to `uint96(_shares)` while deducting the full `_shares` from `_yieldFeeTotalSupply` or liquidatable yield and emitting `Transfer` with the untruncated `_shares`.

### Assumptions and preconditions

1. The underlying asset has sufficient supply and decimals (e.g., an 18-decimal token where $2^{96} \approx 7.92 \times 10^{28}$ raw units, or $\approx 7.92 \times 10^{10}$ tokens) such that `type(uint96).max + 1` shares can be obtained (including via flash loans).
2. Other users have deposited assets into the `Vault` (or yield has accrued in `_yieldVault`) that can be drained, or yield/fees exceeding `type(uint96).max` are minted via `liquidate`/`mintYieldFee`.

### Proof of concept

1. Assume the Vault uses an 18-decimal token (such as DAI or a token with 18 decimals) where the exchange rate is $1\text{e}18$ (1 share = 1 asset). Suppose the Vault holds $100\text{e}9 \times 10^{18}$ assets from honest depositors.
2. An attacker acquires $2^{96} = 79,228,162,514,264,337,593,543,950,336$ raw units ($\approx 79.23\text{ billion}$ tokens, e.g., via a flash loan or on a token with high supply/decimals) and calls `vault.deposit(type(uint96).max, attacker)` and then `vault.deposit(1, attacker)`.
3. In `TwabController`, `attacker`'s `uint112` balance is now `uint256(type(uint96).max) + 1` ($2^{96}$).
4. The attacker calls `vault.redeem(uint256(type(uint96).max) + 1, attacker, attacker)`:
   - `maxRedeem(attacker)` returns $2^{96}$, so `_shares > maxRedeem(attacker)` does not revert.
   - `_assets` is computed as $2^{96}$.
   - `_withdraw` calls `_burn(attacker, 2^{96})`.
   - `_burn` calls `_twabController.burn(attacker, uint96(2^{96}))`, which passes `0` as the burn amount.
   - `_twabController` burns `0` shares from `attacker`, leaving `attacker`'s share balance at $2^{96}$.
   - `_withdraw` withdraws $2^{96}$ underlying assets from `_yieldVault` and transfers them to `attacker`.
5. The attacker retains all $2^{96}$ Vault shares in `TwabController` AND receives back their entire $2^{96}$ underlying asset deposit.
6. The attacker repeats `vault.redeem` (or `vault.withdraw`) to drain all remaining underlying assets held by the Vault in `_yieldVault`.

### Impact assessment

- **Severity**: High / Critical
- **Technical Impact**: Silent integer truncation in `_burn` allows an attacker with $\ge 2^{96}$ shares to withdraw underlying assets without burning their shares (`uint96(2^96) == 0`), completely breaking the 1:1 asset-to-share backing invariant and enabling total drainage of the Vault's assets in `_yieldVault`. Additionally, silent truncation in `_mint` permanently destroys excess shares during `mintYieldFee` or `liquidate` if `_shares > type(uint96).max`, and causes `Transfer` events to report amounts that diverge from the actual ledger changes in `TwabController`.

### Remediation

Use OpenZeppelin's `SafeCast.toUint96` when casting `_shares` in `_mint`, `_burn`, and `_transfer` (or cap `maxWithdraw` and `maxRedeem` to `type(uint96).max` in addition to safe casting).

```solidity
  function _mint(address _receiver, uint256 _shares) internal virtual override {
    _twabController.mint(_receiver, SafeCast.toUint96(_shares));
    _updateExchangeRate();

    emit Transfer(address(0), _receiver, _shares);
  }

  function _burn(address _owner, uint256 _shares) internal virtual override {
    _twabController.burn(_owner, SafeCast.toUint96(_shares));
    _updateExchangeRate();

    emit Transfer(_owner, address(0), _shares);
  }
```

## Finding 14: Premature Exchange Rate Update in `Vault._burn` Causes Incorrect `_lastRecordedExchangeRate` Snapshot During Withdrawals

### Synopsis

In `Vault._burn`, `_updateExchangeRate()` is invoked immediately after `_twabController.burn()` reduces the share supply, but before `_yieldVault.withdraw()` reduces the underlying withdrawable assets in `_withdraw()`. This collateral-and-supply desynchronization snapshots an artificially inflated `_lastRecordedExchangeRate` during undercollateralized withdrawals.

### Technical details

The `Vault` contract tracks `_lastRecordedExchangeRate` whenever shares are minted or burned via `_updateExchangeRate()`, which stores the output of `_currentExchangeRate()`.

In `_currentExchangeRate()` (lines 1168–1187), the exchange rate is calculated from two state values:
1. `_totalSupplyAmount = _totalSupply()`, which reads `_twabController.totalSupply(address(this))`.
2. `_withdrawableAssets = _yieldVault.maxWithdraw(address(this))`, which queries the assets currently held by `_yieldVault` on behalf of the `Vault`, capped at `_totalSupplyToAssets` (computed using `_lastRecordedExchangeRate`).

When a user calls `withdraw()` or `redeem()`, execution flows into `_withdraw()` (lines 1007–1030):
```solidity
_burn(_owner, _shares);

_yieldVault.withdraw(_assets, address(this), address(this));
SafeERC20.safeTransfer(IERC20(asset()), _receiver, _assets);
```

Inside `_burn()` (lines 1138–1143):
```solidity
function _burn(address _owner, uint256 _shares) internal virtual override {
  _twabController.burn(_owner, uint96(_shares));
  _updateExchangeRate();

  emit Transfer(_owner, address(0), _shares);
}
```

When `_updateExchangeRate()` is called at line 1140:
- `_twabController.burn(_owner, uint96(_shares))` at line 1139 has **already** decremented `_totalSupply()` from $S$ to $S - \text{\_shares}$.
- `_yieldVault.withdraw(_assets, address(this), address(this))` at line 1026 has **not yet** executed, so `_yieldVault.maxWithdraw(address(this))` still returns the pre-withdrawal asset balance $A$ instead of the post-withdrawal balance $A - \text{\_assets}$.

When the vault is undercollateralized ($A < S \cdot \text{\_lastRecordedExchangeRate} / \text{\_assetUnit}$), deposits, mints, liquidations, and yield fee mints are all disabled (`_isVaultCollateralized() == false`), so withdrawals/redemptions are the primary operations that update `_lastRecordedExchangeRate` to lock in the lower exchange rate. However, because `_updateExchangeRate()` divides the **pre-withdrawal** assets $A$ by the **post-burn** supply $S - \text{\_shares}$, it computes:

$$\text{rate} = \min\left(\text{\_lastRecordedExchangeRate},\; \left\lfloor \frac{A \cdot \text{\_assetUnit}}{S - \text{\_shares}} \right\rfloor\right)$$

instead of the actual exchange rate:

$$\text{rate}_{\text{actual}} = \left\lfloor \frac{(A - \text{\_assets}) \cdot \text{\_assetUnit}}{S - \text{\_shares}} \right\rfloor \approx \left\lfloor \frac{A \cdot \text{\_assetUnit}}{S} \right\rfloor$$

For example, if the vault loses 50% of its assets in `_yieldVault` ($A = 0.5 S$) while `_lastRecordedExchangeRate == _assetUnit` ($10^{18}$) and a user holding 50% of the shares ($\text{\_shares} = 0.5 S$) withdraws their $\text{\_assets} = 0.25 S$:
- Post-burn `_totalSupplyAmount` is $0.5 S$.
- `_totalSupplyToAssets` is $0.5 S \cdot 10^{18} / 10^{18} = 0.5 S$.
- Pre-withdrawal `_withdrawableAssets` is still $A = 0.5 S$.
- `_currentExchangeRate()` evaluates $0.5 S \cdot 10^{18} / 0.5 S = 10^{18}$, recording `_lastRecordedExchangeRate = 1e18` instead of `0.5e18`.

### Assumptions and preconditions

1. The `Vault` holds deposits from one or more users ($S > 0$) invested in `_yieldVault`.
2. `_yieldVault` experiences a loss of underlying assets such that `_yieldVault.maxWithdraw(address(this))` drops below `_convertToAssets(_totalSupply(), _lastRecordedExchangeRate, Math.Rounding.Down)` (the vault is undercollateralized).
3. A user performs a partial withdrawal or full withdrawal of their own balance while other shares remain in circulation ($0 < \text{\_shares} < S$).

### Proof of concept

1. **Initial Setup**:
   - Alice deposits `10_000_000e18` underlying assets and receives `10_000_000e18` shares.
   - Bob deposits `10_000_000e18` underlying assets and receives `10_000_000e18` shares.
   - `_totalSupply()` is `20_000_000e18`, `_yieldVault.maxWithdraw(address(vault))` is `20_000_000e18`, and `_lastRecordedExchangeRate` is `1e18`.

2. **Undercollateralization Event**:
   - `_yieldVault` loses `10_000_000e18` assets (a 50% loss), leaving `_withdrawableAssets = 10_000_000e18`.
   - `vault.exchangeRate()` now returns `0.5e18` (`5e17`), and `vault.maxWithdraw(alice)` is `5_000_000e18`.
   - `_lastRecordedExchangeRate` in storage is still `1e18` because no mint or burn has occurred since the loss.

3. **Alice Withdraws**:
   - Alice calls `vault.withdraw(5_000_000e18, alice, alice)` (burning `10_000_000e18` shares).
   - `_withdraw` calls `_burn(alice, 10_000_000e18)` before `_yieldVault.withdraw(5_000_000e18, ...)`.
   - Inside `_burn`:
     - `_twabController.burn(alice, 10_000_000e18)` reduces `_totalSupply()` to `10_000_000e18` (Bob's shares).
     - `_updateExchangeRate()` calls `_currentExchangeRate()`.
     - `_totalSupplyAmount` is `10_000_000e18`, so `_totalSupplyToAssets` is `10_000_000e18 * 1e18 / 1e18 = 10_000_000e18`.
     - `_withdrawableAssets` from `_yieldVault.maxWithdraw(address(vault))` is still `10_000_000e18` because the `5_000_000e18` withdrawal from `_yieldVault` has not happened yet.
     - `_currentExchangeRate()` returns `10_000_000e18 * 1e18 / 10_000_000e18 = 1e18`.
     - `_lastRecordedExchangeRate` is set to `1e18` (emitting `RecordedExchangeRate(1e18)`) instead of `5e17`.
   - `_yieldVault.withdraw(5_000_000e18, address(vault), address(vault))` then executes, leaving `5_000_000e18` assets in `_yieldVault`.

4. **Resulting Inconsistency**:
   - `_lastRecordedExchangeRate` remains `1e18` instead of `5e17`. Any subsequent yield up to `5_000_000e18` accrued by `_yieldVault` is absorbed into `_totalSupplyToAssets` rather than being capped by the `5e17` recorded exchange rate and exposed as liquidatable yield for the `PrizePool`.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: `_lastRecordedExchangeRate` is updated while share supply and vault collateral are desynchronized, recording an inflated exchange rate on every non-empty withdrawal during undercollateralization.
- **Business Impact**: The exchange rate cap mechanism (`_lastRecordedExchangeRate`) fails to lock in the undercollateralized exchange rate when users withdraw. Subsequent yield generated by the `YieldVault` is diverted to increasing the share exchange rate back toward the stale `_lastRecordedExchangeRate` rather than being accounted as `availableYieldBalance()` to be liquidated for the `PrizePool` once collateralized at the new baseline.

### Remediation

Update the exchange rate in `_burn` before burning shares from `_twabController` when shares will remain in circulation, and reset/update it after burning when the total supply drops to zero (or ensure `_yieldVault.withdraw` is executed prior to `_updateExchangeRate()`):

```solidity
  function _burn(address _owner, uint256 _shares) internal virtual override {
    if (_totalSupply() != _shares) {
      _updateExchangeRate();
    }

    _twabController.burn(_owner, uint96(_shares));

    if (_totalSupply() == 0) {
      _updateExchangeRate();
    }

    emit Transfer(_owner, address(0), _shares);
  }
```
