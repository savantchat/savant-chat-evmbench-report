# Audit report — 2023-07-pooltogether/H-02

## Finding 1: Premature Collateralization Check in `Vault.mintYieldFee` Allows Minting Unbacked Fee Shares and Inducing Vault Undercollateralization

### Synopsis

`Vault.mintYieldFee` checks `_requireVaultCollateralized()` before minting `_shares`, while `_currentExchangeRate()` excludes `_yieldFeeTotalSupply` from the share supply. When the vault has suffered a loss smaller than `_yieldFeeTotalSupply`, `mintYieldFee` passes the pre-mint check and mints unbacked shares, forcing the vault into an undercollateralized state.

### Technical details

In `Vault.sol`, yield fees are accrued during `liquidate()` by incrementing `_yieldFeeTotalSupply` without immediately minting Vault shares in `TwabController`. As documented on `yieldFeeTotalSupply()` (line 734):

> `@dev If the vault becomes undercollateralized, this total yield fee can be used to collateralize it.`

When `mintYieldFee(uint256 _shares, address _recipient)` is called (lines 394–401), it executes the following steps:

1. Calls `_requireVaultCollateralized()` (line 395), which checks `_currentExchangeRate() >= _assetUnit` (lines 1194–1201).
2. Verifies `_shares <= _yieldFeeTotalSupply` and decrements `_yieldFeeTotalSupply` by `_shares` (lines 396–398).
3. Calls `_mint(_recipient, _shares)` (line 399), which mints `uint96(_shares)` in `_twabController` (increasing `_totalSupply()`) and calls `_updateExchangeRate()` (lines 1122–1125).

The flaw lies in the interaction between `_requireVaultCollateralized()` at line 395 and `_currentExchangeRate()` at lines 1168–1187:

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

Because `_currentExchangeRate()` uses `_totalSupply()` (circulating shares in `TwabController`) rather than `_totalShares()` (`_totalSupply() + _yieldFeeTotalSupply`), `_isVaultCollateralized()` prior to `_mint()` only checks whether `_withdrawableAssets` is sufficient to back the existing circulating `_totalSupply()` at `_lastRecordedExchangeRate`, completely ignoring the `_shares` about to be minted from `_yieldFeeTotalSupply`.

Consequently, if the underlying `_yieldVault` experiences a loss that is less than or equal to `_yieldFeeTotalSupply` (such that `_totalSupplyToAssets <= _withdrawableAssets < _totalSupplyToAssets + _convertToAssets(_shares, ...)`):
- `_requireVaultCollateralized()` at line 395 evaluates to `true` before `_mint()`.
- `_mint(_recipient, _shares)` at line 399 increases `_totalSupply()` by `_shares` and updates `_lastRecordedExchangeRate` via `_updateExchangeRate()`.
- With the increased `_totalSupply()`, `_withdrawableAssets < _totalSupplyToAssets`, causing `_currentExchangeRate()` to drop below `_assetUnit`.

Instead of `_yieldFeeTotalSupply` absorbing the loss to keep the vault collateralized, the unbacked fee shares are minted into circulation, diluting all existing Vault share holders and putting the Vault into an undercollateralized state where `deposit`, `mint`, and `liquidate` are disabled.

### Assumptions and preconditions

1. The Vault has a non-zero `_yieldFeePercentage` and has accrued `_yieldFeeTotalSupply > 0` through one or more yield liquidations.
2. The underlying `_yieldVault` loses an amount of withdrawable assets such that the remaining `_withdrawableAssets` still covers the circulating `_totalSupply()`, but is insufficient to cover `_totalSupply() + _shares` (where `_shares <= _yieldFeeTotalSupply`).

### Proof of concept

1. Alice deposits `1,000e18` underlying assets into `Vault`. `_totalSupply()` is `1,000e18` shares, and `_lastRecordedExchangeRate` is `1e18` (`_assetUnit`).
2. `YieldVault` accrues `100e18` in yield (`_withdrawableAssets = 1,100e18`). With `_yieldFeePercentage = 1e8` (10%), a liquidator liquidates `90e18` yield shares.
   - Circulating `_totalSupply()` is now `1,090e18`.
   - Accrued `_yieldFeeTotalSupply` is `10e18`.
   - `_withdrawableAssets` in `YieldVault` is `1,100e18`.
3. `YieldVault` suffers a loss of `8e18` underlying assets, reducing `_withdrawableAssets` to `1,092e18`.
4. `mintYieldFee(10e18, yieldFeeRecipient)` is called:
   - `_requireVaultCollateralized()` at line 395 calls `_currentExchangeRate()`. Since `_totalSupplyAmount = 1,090e18` and `_withdrawableAssets = 1,092e18 > 1,090e18`, `_currentExchangeRate()` returns `1e18 >= _assetUnit`. The check succeeds.
   - `_yieldFeeTotalSupply` is reduced from `10e18` to `0`.
   - `_mint(yieldFeeRecipient, 10e18)` mints `10e18` shares in `TwabController`, increasing `_totalSupply()` to `1,100e18`, and calls `_updateExchangeRate()`.
5. After `mintYieldFee` completes:
   - `_totalSupply()` is `1,100e18`, backed by only `1,092e18` withdrawable assets.
   - `_currentExchangeRate()` is now `1,092e18 * 1e18 / 1,100e18 = 992727272727272727 < 1e18`.
   - `isVaultCollateralized()` is now `false`, `maxDeposit` and `maxMint` return `0`, `liquidate` reverts with `VaultUnderCollateralized`, and the `8e18` loss is socialized across all share holders instead of being absorbed by the `10e18` yield fee reserve.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:** `mintYieldFee` bypasses the collateralization invariant when losses are within the accrued yield fee buffer, minting unbacked shares and transitioning a healthy, collateralized Vault into an undercollateralized state.
- **Business Implications:**
  1. **Theft/Dilution of Depositor Principal:** Losses that were explicitly meant to be absorbed by `yieldFeeTotalSupply` are instead socialized across all depositors while the fee recipient extracts the full unreduced fee share amount.
  2. **Protocol DoS:** Because the Vault becomes undercollateralized after the mint, `maxDeposit` and `maxMint` drop to `0` and `liquidate` reverts with `VaultUnderCollateralized`, halting all new deposits, mints, and yield liquidations for the Vault.

### Remediation

Validate that the Vault remains collateralized *after* minting the yield fee shares (and updating the exchange rate), rather than only before the mint.

```solidity
  function mintYieldFee(uint256 _shares, address _recipient) external {
    if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);

    _yieldFeeTotalSupply -= _shares;
    _mint(_recipient, _shares);

    _requireVaultCollateralized();

    emit MintYieldFee(msg.sender, _recipient, _shares);
  }
```

## Finding 2: Lack of Access Control in `Vault.mintYieldFee` Allows Any Caller to Steal Accrued Yield Fees

### Synopsis

`Vault.mintYieldFee` is an external function that lacks caller authorization and does not enforce that shares are minted to the configured `_yieldFeeRecipient`. Consequently, any unprivileged address can call `mintYieldFee` with its own address as `_recipient` and steal the entire accrued `_yieldFeeTotalSupply`.

### Technical details

In `Vault.sol`, the contract maintains a dedicated state variable `_yieldFeeRecipient` (configured in the constructor at line 276 and updatable by the owner via `setYieldFeeRecipient` at line 704) intended to designate the account entitled to receive protocol yield fees.

When yield is liquidated via `liquidate` (lines 550–587), a portion of the liquidated yield is reserved as yield fee shares by incrementing `_yieldFeeTotalSupply` via `_increaseYieldFeeBalance` (lines 572–576, 852–854).

To claim these accrued fee shares, the contract exposes `mintYieldFee` at lines 394–401:

```solidity
function mintYieldFee(uint256 _shares, address _recipient) external {
  _requireVaultCollateralized();
  if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);

  _yieldFeeTotalSupply -= _shares;
  _mint(_recipient, _shares);

  emit MintYieldFee(msg.sender, _recipient, _shares);
}
```

However, `mintYieldFee` contains two critical flaws:
1. It has no access control modifier or check verifying that `msg.sender == _yieldFeeRecipient`.
2. It accepts an arbitrary `_recipient` address parameter instead of minting exclusively to `_yieldFeeRecipient`.

Because `_yieldFeeRecipient` is never referenced during `mintYieldFee`, any external actor can monitor `_yieldFeeTotalSupply` and invoke `mintYieldFee(_yieldFeeTotalSupply, attackerAddress)` whenever `_yieldFeeTotalSupply > 0` and the vault is collateralized, minting the accrued yield fee shares to themselves and later redeeming them for underlying assets.

### Assumptions and preconditions

1. The vault is collateralized (`_isVaultCollateralized()` returns `true`).
2. `_yieldFeePercentage` is non-zero and at least one liquidation has occurred, resulting in `_yieldFeeTotalSupply > 0`.

### Proof of concept

1. The `Vault` is deployed with `_yieldFeePercentage = 100000000` (10%) and `_yieldFeeRecipient = feeRecipient`.
2. Users deposit underlying assets into the `Vault`, and the underlying `_yieldVault` accrues 10e18 of yield.
3. A liquidator swaps prize tokens for 9e18 Vault shares via `Vault.liquidate`. During this call, `_increaseYieldFeeBalance(1e18)` increases `_yieldFeeTotalSupply` to `1e18`.
4. An unprivileged attacker observes that `vault.yieldFeeTotalSupply() == 1e18`.
5. The attacker calls `vault.mintYieldFee(1e18, attacker)`.
6. `_requireVaultCollateralized()` passes, `_shares <= _yieldFeeTotalSupply` passes, `_yieldFeeTotalSupply` is reduced to `0`, and `1e18` Vault shares are minted to `attacker`.
7. The attacker calls `vault.redeem(1e18, attacker, attacker)` to withdraw the underlying assets, permanently depriving `feeRecipient` of the accrued yield fees.

### Impact assessment

- **Severity**: High
- **Technical Effect**: Unauthorized minting of all accrued yield fee shares (`_yieldFeeTotalSupply`) to an arbitrary address controlled by an attacker.
- **Business Impact**: 100% loss of accrued yield fees intended for the protocol or vault fee recipient (`_yieldFeeRecipient`), directly transferable into underlying assets via `redeem`/`withdraw`.

### Remediation

Restrict `mintYieldFee` so that only `_yieldFeeRecipient` can call it (or mint strictly to `_yieldFeeRecipient` if anyone is allowed to trigger the mint). The preferred approach is to check that `msg.sender` is `_yieldFeeRecipient`:

```solidity
  error CallerNotYieldFeeRecipient(address caller, address yieldFeeRecipient);

  function mintYieldFee(uint256 _shares, address _recipient) external {
    _requireVaultCollateralized();
    if (msg.sender != _yieldFeeRecipient)
      revert CallerNotYieldFeeRecipient(msg.sender, _yieldFeeRecipient);
    if (_shares > _yieldFeeTotalSupply) revert YieldFeeGTAvailable(_shares, _yieldFeeTotalSupply);

    _yieldFeeTotalSupply -= _shares;
    _mint(_recipient, _shares);

    emit MintYieldFee(msg.sender, _recipient, _shares);
  }
```

## Finding 3: Unsafe Downcast from `uint256` to `uint96` in `Vault._mint` Silently Truncates Minted Shares

### Synopsis

`Vault._mint` unsafely downcasts the `uint256 _shares` parameter to `uint96` when calling `_twabController.mint`. When `_shares` exceeds `type(uint96).max` (such as via `liquidate` or `mintYieldFee`), the share amount silently wraps modulo $2^{96}$, causing a permanent loss of shares and desynchronizing vault accounting from `TwabController`.

### Technical details

`Vault` overrides `ERC20._mint(address _receiver, uint256 _shares)` at line 1122 and delegates balance tracking to `TwabController.mint(address _to, uint96 _amount)`. At line 1123, `_shares` is converted from `uint256` to `uint96` using a raw typecast:

```solidity
function _mint(address _receiver, uint256 _shares) internal virtual override {
  _twabController.mint(_receiver, uint96(_shares));
  _updateExchangeRate();

  emit Transfer(address(0), _receiver, _shares);
}
```

In Solidity `0.8.x`, explicit integer downcasting (`uint96(_shares)`) does not revert on overflow; instead, it silently truncates the higher-order bits above bit 95 (`_shares % 2**96`).

While `Vault.deposit` and `Vault.mint` bound single-call inputs via `maxDeposit` and `maxMint` (which return `type(uint96).max` when collateralized), other entry points that invoke `_mint` do not enforce a `uint96` upper bound on `_shares`:

1. **`mintYieldFee(uint256 _shares, address _recipient)` (lines 394–402):** Checks only `_shares <= _yieldFeeTotalSupply`. Over multiple liquidations or high-yield periods (especially for high-decimal underlying tokens), `_yieldFeeTotalSupply` can exceed `type(uint96).max`. When `mintYieldFee` is called with `_shares > type(uint96).max`, `_yieldFeeTotalSupply` is decremented by the full `_shares` amount (line 398), yet `_twabController.mint` only credits `uint96(_shares)` to `_recipient`.
2. **`liquidate(...)` (lines 550–587):** Checks only `_amountOut <= _liquidableYield`. `_liquidableYield` is computed in `uint256` from `availableYieldBalance()`. If accrued yield exceeds `type(uint96).max` and a liquidator liquidates `_amountOut > type(uint96).max`, `_prizePool.contributePrizeTokens` collects the full `_amountIn` of prize tokens (line 570) and `_increaseYieldFeeBalance` accrues fee shares on the full `_amountOut` (lines 572–576), but `_mint(_account, _amountOut)` (line 584) credits only `uint96(_amountOut)` shares to `_account`.

In addition, `emit Transfer(address(0), _receiver, _shares)` at line 1126 emits the un-truncated `uint256 _shares`, violating ERC20 event integrity relative to the actual balance increase in `TwabController`.

### Assumptions and preconditions

1. The vault manages an underlying asset (e.g., an 18-decimal token) where either accrued liquidatable yield (`_liquidableYield`) or accumulated yield fee supply (`_yieldFeeTotalSupply`) reaches an amount greater than `type(uint96).max` ($\approx 7.92 \times 10^{28}$ raw units, or $\approx 7.92 \times 10^{10}$ tokens with 18 decimals).
2. A caller invokes `mintYieldFee` or `liquidate` with a share amount `_shares > type(uint96).max`.

### Proof of concept

1. Suppose the underlying token has 18 decimals and deposits in `Vault` generate yield over time, or yield fees accumulate in `_yieldFeeTotalSupply` across multiple liquidations until `_yieldFeeTotalSupply = type(uint96).max + 1` ($2^{96}$).
2. A caller invokes `mintYieldFee(2**96, yieldFeeRecipient)`.
3. In `mintYieldFee`:
   - `_requireVaultCollateralized()` passes.
   - `_shares > _yieldFeeTotalSupply` ($2^{96} > 2^{96}$) is false.
   - `_yieldFeeTotalSupply` is reduced by $2^{96}$ to `0`.
   - `_mint(yieldFeeRecipient, 2**96)` is called.
4. In `_mint`:
   - `uint96(2**96)` evaluates to `0`.
   - `_twabController.mint(yieldFeeRecipient, 0)` mints `0` shares to `yieldFeeRecipient` and increases `_twabController.totalSupply(address(this))` by `0`.
   - `_updateExchangeRate()` records the exchange rate based on `0` newly minted shares.
   - `Transfer(address(0), yieldFeeRecipient, 2**96)` is emitted.
5. `yieldFeeRecipient` receives `0` shares, while `_yieldFeeTotalSupply` has been completely zeroed out, permanently losing all $2^{96}$ yield fee shares (the underlying assets remain in the yield vault as unbacked surplus yield).
6. Similarly, if `liquidate` is called with `_amountOut = 2**96` when `_liquidableYield >= 2**96`, the caller transfers the full `_amountIn` of prize tokens to the PrizePool, `_yieldFeeTotalSupply` is increased for $2^{96}$ shares, and `_mint(_account, 2**96)` mints `0` shares to `_account`.

### Impact assessment

- **Severity:** Medium / High (Loss of funds / accounting corruption when values exceed `uint96`).
- **Direct Technical Effect:** Silent integer truncation in `_mint` causes `TwabController` to mint `_shares mod 2^96` shares while callers (`mintYieldFee`, `liquidate`) deduct or charge for the full `uint256 _shares` amount and `Transfer` events log the un-truncated `uint256` amount.
- **Business Impact:** Liquidators can lose up to `type(uint96).max + 1` worth of vault shares while still paying the full prize token input and inflating `_yieldFeeTotalSupply` (which can even push the vault into undercollateralization because `_totalShares()` includes the un-truncated fee shares while `_totalSupply()` does not). Yield fee recipients can permanently lose `type(uint96).max + 1` of accrued fee shares upon calling `mintYieldFee`.

### Remediation

Use OpenZeppelin's `SafeCast.toUint96` when casting `_shares` in `_mint` (and similarly in `_burn` and `_transfer`) so that any attempt to mint more than `type(uint96).max` shares reverts safely instead of silently truncating.

```solidity
  function _updateExchangeRate() internal {
    _lastRecordedExchangeRate = _currentExchangeRate();
    emit RecordedExchangeRate(_lastRecordedExchangeRate);
  }

  /**
   * @notice Creates `_shares` tokens and assigns them to `_receiver`, increasing the total supply.
   * @param _receiver Address that will receive the minted shares
   * @param _shares Shares to mint
   * @dev Emits a {Transfer} event with `from` set to the zero address.
   * @dev `_receiver` cannot be the zero address.
   * @dev Updates the exchange rate.
   */
  function _mint(address _receiver, uint256 _shares) internal virtual override {
    _twabController.mint(_receiver, SafeCast.toUint96(_shares));
    _updateExchangeRate();

    emit Transfer(address(0), _receiver, _shares);
  }
```

## Finding 4: Missing Zero-Address Check in `Vault._mint` Causes Silent Asset Loss on Minting to `address(0)`

### Synopsis

`Vault._mint` overrides `ERC20._mint` without validating that `_receiver != address(0)`. Because `TwabController.mint` treats a mint to `address(0)` as a no-op self-transfer from `address(0)` to `address(0)`, calling `deposit`, `mint`, `mintYieldFee`, or `liquidate` with `_receiver = address(0)` consumes the caller's assets or fee allocation without minting any shares.

### Technical details

In OpenZeppelin's standard `ERC20._mint`, minting to `address(0)` reverts via `require(account != address(0), "ERC20: mint to the zero address")`. `Vault` inherits from `ERC4626` (and `ERC20`), and overrides `_mint(address _receiver, uint256 _shares)` at lines 1122–1126 to delegate balance and supply tracking to `_twabController`:

```solidity
function _mint(address _receiver, uint256 _shares) internal virtual override {
  _twabController.mint(_receiver, uint96(_shares));
  _updateExchangeRate();

  emit Transfer(address(0), _receiver, _shares);
}
```

Although the NatSpec on line 1119 explicitly specifies `@dev _receiver cannot be the zero address`, neither `Vault._mint` nor `TwabController.mint` enforces this invariant.

When `_twabController.mint(_receiver, uint96(_shares))` is invoked, `TwabController` executes `_transferBalance(msg.sender, address(0), _to, _amount)`:

```solidity
function _transferBalance(address _vault, address _from, address _to, uint96 _amount) internal {
  if (_from == _to) {
    return;
  }
  ...
```

When `_receiver` (`_to`) is `address(0)`, the condition `_from == _to` (`address(0) == address(0)`) evaluates to `true`, causing `_transferBalance` to return immediately without reverting and without increasing any account balance or the vault's `totalSupply`.

Consequently, across all entry points that rely on `_mint`:
1. **`deposit(_assets, address(0))` and `mint(_shares, address(0))`**: `_deposit` pulls `_assets` from `msg.sender` and deposits them into `_yieldVault`, then calls `_mint(address(0), _shares)`. The call succeeds, emits a `Transfer(address(0), address(0), _shares)` event, and increases `_totalAssets()` without increasing `_totalSupply()`. The deposited assets are permanently lost by the caller and become unbacked yield in the vault, which can then be liquidated by third parties via `liquidate()`.
2. **`mintYieldFee(_shares, address(0))`**: Decrements `_yieldFeeTotalSupply` by `_shares` without minting any shares, permanently destroying the accrued yield fee shares.
3. **`liquidate(address(0), ...)`**: Pulls `_amountIn` of prize tokens from the caller to the prize pool without minting the corresponding `_amountOut` vault shares.

### Assumptions and preconditions

- The Vault and its associated `TwabController` and `_yieldVault` are deployed and collateralized.
- A user or integrator calls `deposit`, `mint`, `mintYieldFee`, or `liquidate` with the recipient address set to `address(0)` (e.g., due to an uninitialized recipient variable in an integrating contract or frontend).

### Proof of concept

1. Suppose the `Vault` is initialized with `totalSupply = 1000e18` and `_totalAssets = 1000e18` (exchange rate is `1e18`).
2. Alice approves the `Vault` to spend `100e18` underlying assets and calls `vault.deposit(100e18, address(0))`.
3. Inside `Vault.deposit`:
   - `_shares` is calculated as `100e18`.
   - `_deposit(alice, address(0), 100e18, 100e18)` is executed.
4. Inside `Vault._deposit`:
   - `100e18` underlying assets are transferred from Alice to the `Vault` and deposited into `_yieldVault`.
   - `_mint(address(0), 100e18)` is called.
5. Inside `Vault._mint`:
   - `_twabController.mint(address(0), uint96(100e18))` calls `_transferBalance(address(vault), address(0), address(0), 100e18)`.
   - Because `_from == _to == address(0)`, `_transferBalance` returns early without modifying any state.
   - `_updateExchangeRate()` records the exchange rate.
   - `Transfer(address(0), address(0), 100e18)` and `Deposit(alice, address(0), 100e18, 100e18)` are emitted.
6. After the transaction completes:
   - Alice's underlying asset balance has decreased by `100e18`.
   - `vault.totalSupply()` remains `1000e18` (no shares were created).
   - `vault.availableYieldBalance()` increases by `100e18`, allowing any liquidator to capture Alice's deposited principal through `liquidate()`.

### Impact assessment

- **Severity**: Medium / High
- **Technical Effect**: Violation of the ERC20/ERC4626 minting invariant (`_receiver != address(0)`), divergence between emitted `Transfer`/`Deposit` events and actual ledger state in `TwabController`, and silent failure to increase `totalSupply` upon asset deposit.
- **Business Impact**: Permanent loss of user funds (converted into liquidatable yield for arbitrary liquidators) or permanent loss of accrued protocol yield fees if `address(0)` is passed as the recipient.

### Remediation

Add a zero-address check in `Vault._mint` (or in `TwabController.mint`) to revert when `_receiver == address(0)`.

```solidity
  error MintToZeroAddress();

  function _mint(address _receiver, uint256 _shares) internal virtual override {
    if (_receiver == address(0)) revert MintToZeroAddress();

    _twabController.mint(_receiver, uint96(_shares));
    _updateExchangeRate();

    emit Transfer(address(0), _receiver, _shares);
  }
```

## Finding 5: Unauthorized Delegation Override in `Vault._sponsor` Allows Any Caller to Revoke Any User's Prize Eligibility

### Synopsis

`Vault._sponsor` accepts an arbitrary `_receiver` address without verifying that `_receiver == msg.sender`, and invokes `_twabController.sponsor(_receiver)`. Consequently, any attacker can call `sponsor` with a victim's address as `_receiver` (even with `0` assets) to forcibly delegate the victim's entire balance to `SPONSORSHIP_ADDRESS`, stripping their prize eligibility.

### Technical details

In PoolTogether V5, users' share balances and prize-winning eligibility (`delegateBalance`) are tracked in `TwabController`. Users who wish to deposit into the `Vault` to contribute yield without participating in prize draws can sponsor the vault via `Vault.sponsor` or `Vault.sponsorWithPermit`, both of which delegate to the internal function `Vault._sponsor(uint256 _assets, address _receiver)` (lines 982–994).

In `TwabController`, the `sponsor(address _from)` function is designed to be called by the `Vault` (`msg.sender` in `TwabController` is the vault address) and delegates `_from`'s entire vault share balance to `SPONSORSHIP_ADDRESS` (`address(1)`):

```solidity
function sponsor(address _from) external {
  _delegate(msg.sender, _from, SPONSORSHIP_ADDRESS);
}
```

When a user's delegate is set to `SPONSORSHIP_ADDRESS`, `TwabController._transferDelegateBalance` subtracts the user's entire balance from their current delegate's `delegateBalance` and from the vault's `totalSupplyDelegateBalance`, effectively revoking all chances for that balance to win prizes.

However, in `Vault._sponsor`:

```solidity
function _sponsor(uint256 _assets, address _receiver) internal returns (uint256) {
  uint256 _shares = deposit(_assets, _receiver);

  if (
    _twabController.delegateOf(address(this), _receiver) != _twabController.SPONSORSHIP_ADDRESS()
  ) {
    _twabController.sponsor(_receiver);
  }

  emit Sponsor(msg.sender, _receiver, _assets, _shares);

  return _shares;
}
```

`Vault._sponsor` does not restrict `_receiver` to `msg.sender`. Because `TwabController.sponsor` trusts `msg.sender` (the `Vault`) to enforce authorization over `_from`, `Vault` acts as a confused deputy: any caller can pass an arbitrary victim's address as `_receiver` to `Vault.sponsor(0, victim)` (or with dust `_assets`) and force `TwabController` to overwrite `delegates[address(this)][victim]` with `SPONSORSHIP_ADDRESS`.

### Assumptions and preconditions

- A victim holds a non-zero balance of `Vault` shares and has their balance delegated to themselves or another prize-eligible address (i.e., not `SPONSORSHIP_ADDRESS`).
- The `Vault` is collateralized (`maxDeposit` returns `type(uint96).max`).

### Proof of concept

1. Alice deposits `1,000e18` underlying assets into `Vault` via `vault.deposit(1000e18, alice)`.
   - `twabController.balanceOf(address(vault), alice)` is `1000e18`.
   - `twabController.delegateBalanceOf(address(vault), alice)` is `1000e18`.
   - `twabController.totalSupplyDelegateBalance(address(vault))` includes Alice's `1000e18` shares.
2. An attacker (Bob) calls `vault.sponsor(0, alice)`.
3. Inside `Vault._sponsor(0, alice)`:
   - `deposit(0, alice)` executes and mints `0` shares to Alice.
   - `_twabController.delegateOf(address(vault), alice)` is `alice` (which is not `SPONSORSHIP_ADDRESS`).
   - `_twabController.sponsor(alice)` is called by `Vault`.
4. `TwabController._delegate(address(vault), alice, SPONSORSHIP_ADDRESS)` executes:
   - `delegates[address(vault)][alice]` is set to `SPONSORSHIP_ADDRESS`.
   - Alice's entire `1000e18` `delegateBalance` is removed (`twabController.delegateBalanceOf(address(vault), alice)` becomes `0`).
5. Alice no longer accumulates TWAB for prize draws and cannot win any prizes, while her deposited assets continue generating yield for the pool (increasing the winning odds of other participants, including the attacker).

### Impact assessment

**Severity: High**

An attacker can arbitrarily and repeatedly grief any user (or all other depositors in the `Vault`) at zero token cost (`_assets = 0`), stripping victims of their `delegateBalance` andTWAB accumulation so they cannot win prizes. Furthermore, because `TwabController` uses `totalSupplyDelegateBalance` to compute prize odds or because the victim's yield still contributes to the prize pool while the victim's `delegateBalance` is zeroed out, an attacker who holds vault shares can zero out all other depositors' `delegateBalance` to monopolize prize draws.

### Remediation

Remove the `_receiver` parameter from `sponsor` / `sponsorWithPermit` / `_sponsor` and always use `msg.sender` as the receiver and sponsored account.

```solidity
  function _sponsor(uint256 _assets) internal returns (uint256) {
    address _receiver = msg.sender;
    uint256 _shares = deposit(_assets, _receiver);

    if (
      _twabController.delegateOf(address(this), _receiver) != _twabController.SPONSORSHIP_ADDRESS()
    ) {
      _twabController.sponsor(_receiver);
    }

    emit Sponsor(_receiver, _receiver, _assets, _shares);

    return _shares;
  }
```

## Finding 6: Precision Loss and Insolvency Mispricing in `Vault.withdraw` and `Vault.redeem` Share-Asset Conversions

### Synopsis

`Vault.withdraw` and `Vault.redeem` convert between shares and assets via an intermediate `_currentExchangeRate()` truncated to `_assetUnit` precision and falling back to a 1:1 ratio when the rate reaches zero, causing amplified rounding losses during redemptions and allowing early redeemers to drain remaining collateral under severe undercollateralization.

### Technical details

In `Vault.sol`, both `withdraw` (lines 509–521) and `redeem` (lines 524–535) rely on `maxWithdraw`, `_convertToShares` (line 517), and `_convertToAssets` (line 531) to value user shares against the underlying assets withdrawable from `_yieldVault`.

Rather than converting directly between `_shares` and `_withdrawableAssets` in a single `mulDiv` operation (`_shares.mulDiv(_withdrawableAssets, _totalSupply, rounding)`), `Vault` performs a two-step calculation using `_currentExchangeRate()` (lines 1168–1187) scaled only to `_assetUnit` (`10 ** decimals`):

1. **Intermediate Exchange-Rate Truncation and Rounding Bias (`withdraw` / `redeem`):**
   In `_currentExchangeRate()` (line 1183), the exchange rate is computed as:
   ```solidity
   return _withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down);
   ```
   Then `_convertToAssets` (line 904, invoked by `redeem` at line 531 and `maxWithdraw` at line 514) computes:
   ```solidity
   _shares.mulDiv(_exchangeRate, _assetUnit, _rounding)
   ```
   And `_convertToShares` (line 873, invoked by `withdraw` at line 517) computes:
   ```solidity
   _assets.mulDiv(_assetUnit, _exchangeRate, _rounding)
   ```
   Whenever `_totalSupplyAmount > _assetUnit` and the vault is undercollateralized—even by 1 wei (for instance, due to standard ERC-4626 rounding in `_yieldVault.maxWithdraw(address(this))`)—the intermediate division in `_currentExchangeRate()` truncates `_exchangeRate` down to the nearest multiple of `1 / _assetUnit`. When `_convertToAssets` subsequently multiplies by `_shares` and divides by `_assetUnit`, this truncation error is amplified by a factor of `_shares / _assetUnit`.
   
   For example, in a USDC vault (`_assetUnit = 1e6`) with `10,000,000e6` ($10M) total supply, a 1-wei deficit in `_withdrawableAssets` (`10,000,000e6 - 1`) reduces `_exchangeRate` from `1,000,000` to `999,999`. A user redeeming half the pool (`5,000,000e6` shares) via `redeem` (line 531) or `withdraw` (line 514) receives only `4,999,995e6` USDC, losing `5e6` ($5 USDC) to rounding truncation. Because those `5e6 - 1` assets remain in `_yieldVault` after the first user's shares are burned, the remaining shareholders become overcollateralized Again and capture the first user's lost principal (or it is liquidated as yield).

2. **Boundary-State Insolvency Mispricing (`_exchangeRate == 0` and `_withdrawableAssets == 0`):**
   When undercollateralization is severe enough that `_withdrawableAssets * _assetUnit < _totalSupplyAmount` (with `_withdrawableAssets > 0`), `_currentExchangeRate()` rounds down to `0`. However, `_convertToShares` (line 871) and `_convertToAssets` (line 902) treat `_exchangeRate == 0` as a 1:1 exchange rate:
   ```solidity
   return
     (_shares == 0 || _exchangeRate == 0)
       ? _shares
       : _shares.mulDiv(_exchangeRate, _assetUnit, _rounding);
   ```
   Consequently, in `withdraw` (line 517) and `redeem` (line 531), a user holding only `_shares = _withdrawableAssets` (a tiny fraction of `_totalSupplyAmount`) can redeem at a 1:1 rate and withdraw 100% of the vault's remaining `_withdrawableAssets`, leaving nothing for other shareholders. Similarly, if `_withdrawableAssets == 0` while `_totalSupplyAmount != 0`, `_currentExchangeRate()` (line 1182–1186) bypasses the division and returns `_assetUnit` (1:1).

### Assumptions and preconditions

- The vault holds deposits from one or more users such that `_totalSupply() > _assetUnit` (or the vault experiences severe undercollateralization where `_withdrawableAssets * _assetUnit < _totalSupply()`).
- `_yieldVault.maxWithdraw(address(this))` is less than `_totalSupplyToAssets` (either due to a loss in the underlying yield source or a 1-wei rounding difference in `_yieldVault`).

### Proof of concept

#### Scenario 1: Amplified Precision Loss on `redeem` / `withdraw`
1. Assume a 6-decimal underlying asset such as USDC (`_assetUnit = 1e6`).
2. Alice deposits `5,000,000e6` USDC and Bob deposits `5,000,000e6` USDC (`_totalSupply() = 10,000,000e6`, `_lastRecordedExchangeRate = 1e6`).
3. `_yieldVault.maxWithdraw(address(vault))` decreases by 1 wei to `9,999,999,999,999` (`10,000,000e6 - 1`) due to rounding in the yield vault or a minor loss.
4. `_currentExchangeRate()` computes:
   `mulDiv(9_999_999_999_999, 1e6, 10_000_000e6, Down) = 999_999`.
5. Alice calls `redeem(5_000_000e6, alice, alice)` (or `withdraw(maxWithdraw(alice), alice, alice)`):
   - Line 531 computes `_assets = mulDiv(5_000_000e6, 999_999, 1e6, Down) = 4_999_995e6`.
   - Alice burns all `5,000,000e6` shares and receives `4,999_995e6` USDC, losing `5e6` (5 USDC) instead of her pro-rata 0.5 wei.
6. `_yieldVault` still holds `5,000,004,999,999` wei (`5,000,005e6 - 1` USDC) for Bob's `5,000,000e6` shares. Bob can now withdraw his full `5,000,000e6` USDC while the remaining `4,999,999` wei of Alice's principal is captured as yield.

#### Scenario 2: Complete Collateral Drain When `_exchangeRate == 0`
1. In the same vault (`_totalSupply() = 10,000,000e6`, `_assetUnit = 1e6`), `_yieldVault` suffers a severe exploit/loss leaving `_withdrawableAssets = 9,000` (0.009 USDC).
2. `_currentExchangeRate()` computes `mulDiv(9_000, 1e6, 10_000_000e6, Down) = 0`.
3. A user holding just `9,000` shares (0.00009% of the pool) calls `redeem(9_000, receiver, owner)` (or `withdraw(9_000, receiver, owner)`):
   - Because `_exchangeRate == 0`, `_convertToAssets(9_000, 0, Down)` (line 902, called at line 531) returns `9,000` assets (1:1).
   - The user withdraws all `9,000` remaining assets in `_yieldVault`, leaving `0` assets for the holders of the remaining `9,999,999,991,000` shares.

### Impact assessment

- **Severity:** High / Medium
- **Direct Technical Effects:**
  - Inaccurate share-to-asset and asset-to-share conversions in `withdraw` and `redeem` due to low intermediate exchange-rate precision (`_assetUnit`).
  - Unfair redistribution of user principal to later withdrawers or liquidators when a user withdraws while `_currentExchangeRate()` suffers truncation.
  - Complete drain of remaining collateral by a minority shareholder when `_exchangeRate` rounds down to `0` (or false collateralization reporting when `_withdrawableAssets == 0`).

### Remediation

Scale the exchange rate using a higher precision constant (e.g., `1e18` or `1e36` depending on asset decimals) or perform direct single-step `mulDiv` conversions between shares and withdrawable assets in `withdraw` and `redeem`, and do not fall back to a 1:1 ratio when `_exchangeRate == 0` (or return `_assetUnit` when `_withdrawableAssets == 0` and `_totalSupplyAmount != 0`).

In `withdraw` and `redeem` (and their underlying conversion helpers), ensure conversions do not revert to 1:1 on zero exchange rate and avoid intermediate truncation:

```solidity
  function withdraw(
    uint256 _assets,
    address _receiver,
    address _owner
  ) public virtual override returns (uint256) {
    uint256 _maxWithdraw = maxWithdraw(_owner);
    if (_assets > _maxWithdraw)
      revert WithdrawMoreThanMax(_owner, _assets, _maxWithdraw);

    uint256 _shares = _convertToShares(_assets, Math.Rounding.Up);
    _withdraw(msg.sender, _receiver, _owner, _assets, _shares);

    return _shares;
  }

  function redeem(
    uint256 _shares,
    address _receiver,
    address _owner
  ) public virtual override returns (uint256) {
    uint256 _maxRedeem = maxRedeem(_owner);
    if (_shares > _maxRedeem) revert RedeemMoreThanMax(_owner, _shares, _maxRedeem);

    uint256 _assets = _convertToAssets(_shares, Math.Rounding.Down);
    _withdraw(msg.sender, _receiver, _owner, _assets, _shares);

    return _assets;
  }
```

And in `_convertToShares`, `_convertToAssets`, and `_currentExchangeRate`:

```solidity
  function _convertToShares(
    uint256 _assets,
    Math.Rounding _rounding
  ) internal view virtual override returns (uint256) {
    uint256 _exchangeRate = _currentExchangeRate();

    return
      (_assets == 0 || _exchangeRate == 0)
        ? 0
        : _assets.mulDiv(_assetUnit, _exchangeRate, _rounding);
  }

  function _convertToAssets(
    uint256 _shares,
    uint256 _exchangeRate,
    Math.Rounding _rounding
  ) internal view returns (uint256) {
    return
      (_shares == 0 || _exchangeRate == 0)
        ? 0
        : _shares.mulDiv(_exchangeRate, _assetUnit, _rounding);
  }

  function _currentExchangeRate() internal view returns (uint256) {
    uint256 _totalSupplyAmount = _totalSupply();
    if (_totalSupplyAmount == 0) {
      return _assetUnit;
    }

    uint256 _withdrawableAssets = _yieldVault.maxWithdraw(address(this));
    if (_withdrawableAssets == 0) {
      return 0;
    }

    uint256 _totalSupplyToAssets = _convertToAssets(
      _totalSupplyAmount,
      _lastRecordedExchangeRate,
      Math.Rounding.Down
    );

    if (_withdrawableAssets > _totalSupplyToAssets) {
      _withdrawableAssets = _totalSupplyToAssets;
    }

    return _withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down);
  }
```

## Finding 7: Premature `_updateExchangeRate()` call before `_yieldVault.withdraw()` in `Vault._withdraw` records an inflated `_lastRecordedExchangeRate` during `Vault.withdraw` and `Vault.redeem`

### Synopsis

During `Vault.withdraw` and `Vault.redeem`, `_withdraw` invokes `_burn` before withdrawing assets from `_yieldVault`. Because `_burn` immediately calls `_updateExchangeRate()`, `_currentExchangeRate()` is evaluated using the post-burn share supply against the pre-withdrawal asset balance, recording an inflated `_lastRecordedExchangeRate` checkpoint whenever the vault is undercollateralized.

### Technical details

`Vault` maintains a checkpoint state variable `_lastRecordedExchangeRate` (updated via `_updateExchangeRate()`) to cap the maximum exchange rate in `_currentExchangeRate()` at the most recent rate observed during share minting or burning:

```solidity
  function _currentExchangeRate() internal view returns (uint256) {
    uint256 _totalSupplyAmount = _totalSupply();
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

In `_deposit` (lines 959–960), the contract first updates the asset balance in `_yieldVault` via `_yieldVault.deposit(_assets, address(this))` and then calls `_mint(_receiver, _shares)`, which updates the share supply in `_twabController` and calls `_updateExchangeRate()`. Thus, both `_yieldVault.maxWithdraw(address(this))` and `_totalSupply()` reflect the post-deposit state when `_lastRecordedExchangeRate` is recorded.

However, in `withdraw` (lines 509–521) and `redeem` (lines 524–535), the execution delegates to `_withdraw` (lines 1007–1030):

```solidity
    _burn(_owner, _shares);

    _yieldVault.withdraw(_assets, address(this), address(this));
    SafeERC20.safeTransfer(IERC20(asset()), _receiver, _assets);
```

And `_burn` (lines 1138–1143) executes:

```solidity
  function _burn(address _owner, uint256 _shares) internal virtual override {
    _twabController.burn(_owner, uint96(_shares));
    _updateExchangeRate();

    emit Transfer(_owner, address(0), _shares);
  }
```

Because `_burn(_owner, _shares)` is executed at line 1024 before `_yieldVault.withdraw(_assets, address(this), address(this))` at line 1026, `_updateExchangeRate()` at line 1140 evaluates `_currentExchangeRate()` in an inconsistent intermediate state:
1. `_totalSupply()` has already been reduced by `_shares` (`S_after = S_before - _shares`).
2. `_yieldVault.maxWithdraw(address(this))` has **not** yet been reduced by `_assets` (`W_before` instead of `W_after = W_before - _assets`).
3. `_totalSupplyToAssets` is computed using `S_after` and the previous `_lastRecordedExchangeRate` (`R_old`).

When the vault has experienced a loss in `_yieldVault` (`W_before < S_before * R_old / _assetUnit`), the actual exchange rate at which `withdraw` (line 517) and `redeem` (line 531) value the redemption is:
$$R_{\text{actual}} = \left\lfloor \frac{W_{\text{before}} \cdot \text{assetUnit}}{S_{\text{before}}} \right\rfloor < R_{\text{old}}$$
Yet during `_burn`, `_currentExchangeRate()` divides the unreduced `W_before` (capped only at `S_after * R_old / _assetUnit`) by the reduced supply `S_after`, recording:
$$R_{\text{recorded}} = \min\left(R_{\text{old}}, \left\lfloor \frac{W_{\text{before}} \cdot \text{assetUnit}}{S_{\text{before}} - \text{shares}} \right\rfloor\right) > R_{\text{actual}}$$

If a user withdraws at least the fraction of shares corresponding to the loss (e.g., withdrawing `>= 50%` of total supply after a `50%` asset loss), `W_before >= S_after * R_old / _assetUnit`, so `_lastRecordedExchangeRate` remains stuck at `R_old` (`_assetUnit`) instead of updating to `R_actual`.

### Assumptions and preconditions

1. Multiple users hold shares in the `Vault` (`_totalSupply() > 0`), and `_lastRecordedExchangeRate` is at its collateralized rate (e.g., `_assetUnit = 1e18`).
2. The underlying `_yieldVault` experiences a loss of assets such that `_yieldVault.maxWithdraw(address(this))` falls below `_convertToAssets(_totalSupply(), _lastRecordedExchangeRate, Math.Rounding.Down)`.
3. A user calls `withdraw` or `redeem` before any zero-value deposit or mint checkpoints the new lower exchange rate.

### Proof of concept

1. Initial state (`_assetUnit = 1e18`):
   - Alice deposits `15,000,000e18` assets and receives `15,000,000e18` shares.
   - Bob deposits `5,000,000e18` assets and receives `5,000,000e18` shares.
   - `_totalSupply() = 20,000,000e18`, `_yieldVault.maxWithdraw(address(vault)) = 20,000,000e18`, and `_lastRecordedExchangeRate = 1e18`.
2. `_yieldVault` loses `10,000,000e18` assets, leaving `_yieldVault.maxWithdraw(address(vault)) = 10,000,000e18`.
   - `_currentExchangeRate()` now returns `10,000,000e18 * 1e18 / 20,000,000e18 = 0.5e18`.
3. Alice calls `vault.withdraw(7_500_000e18, alice, alice)` (or `vault.redeem(15_000_000e18, alice, alice)`):
   - `withdraw` computes `_shares = 15,000,000e18` at the `0.5e18` exchange rate and calls `_withdraw`.
   - `_withdraw` calls `_burn(alice, 15_000_000e18)`.
   - `_twabController.burn` reduces `_totalSupply()` from `20,000,000e18` to `5,000,000e18`.
   - `_updateExchangeRate()` calls `_currentExchangeRate()` while `_yieldVault.maxWithdraw(address(vault))` is still `10,000,000e18`.
   - Inside `_currentExchangeRate()`, `_totalSupplyAmount = 5,000,000e18` and `_totalSupplyToAssets = 5,000,000e18 * 1e18 / 1e18 = 5,000,000e18`.
   - Since `_withdrawableAssets` (`10,000,000e18`) > `_totalSupplyToAssets` (`5,000,000e18`), `_withdrawableAssets` is capped to `5,000,000e18`.
   - `_currentExchangeRate()` returns `5,000,000e18 * 1e18 / 5,000,000e18 = 1e18`, setting `_lastRecordedExchangeRate = 1e18` and emitting `RecordedExchangeRate(1e18)` instead of `5e17`.
   - `_yieldVault.withdraw(7_500_000e18, address(vault), address(vault))` then executes, leaving `2,500,000e18` assets in `_yieldVault` for Bob's `5,000,000e18` shares.
4. Subsequently, if `_yieldVault` accrues `1,000,000e18` in yield (bringing `_withdrawableAssets` to `3,500,000e18`), `_totalSupplyToAssets` is evaluated using the stale `_lastRecordedExchangeRate = 1e18` (`5,000,000e18`) instead of `0.5e18` (`2,500,000e18`). The `1,000,000e18` yield is absorbed into Bob's withdrawable share value (`_currentExchangeRate()` rises to `0.7e18`) rather than being accounted as `availableYieldBalance()` above the `0.5e18` checkpoint.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** `_lastRecordedExchangeRate` is corrupted whenever `withdraw` or `redeem` is called after a vault loss, emitting an inaccurate `RecordedExchangeRate` event and failing to checkpoint the undercollateralized exchange rate ceiling.
- **Business Impact:** Subsequent yield accrued by `_yieldVault` after an undercollateralized withdrawal is not capped at the post-loss exchange rate checkpoint and is instead divertible to remaining shareholders upon withdrawal rather than being isolated as liquidatable prize yield for the protocol.

### Remediation

Withdraw the underlying assets from `_yieldVault` into `address(this)` before calling `_burn(_owner, _shares)` in `_withdraw` (while keeping `SafeERC20.safeTransfer` to `_receiver` after `_burn` to preserve ERC777 reentrancy safety), so that `_updateExchangeRate()` inside `_burn` observes both the post-withdrawal `_yieldVault` balance and the post-burn share supply:

```solidity
  function _withdraw(
    address _caller,
    address _receiver,
    address _owner,
    uint256 _assets,
    uint256 _shares
  ) internal virtual override {
    if (_caller != _owner) {
      _spendAllowance(_owner, _caller, _shares);
    }

    _yieldVault.withdraw(_assets, address(this), address(this));
    _burn(_owner, _shares);
    SafeERC20.safeTransfer(IERC20(asset()), _receiver, _assets);

    emit Withdraw(_caller, _receiver, _owner, _assets, _shares);
  }
```

## Finding 8: Cross-Contract Reentrancy and Share-Asset Accounting Desynchronization in `Vault.withdraw` and `Vault.redeem`

### Synopsis

`Vault.withdraw` and `Vault.redeem` execute multi-step withdrawal settlement without reentrancy protection, burning shares before withdrawing assets from `_yieldVault` into `Vault` and forwarding them to `_receiver`. Callbacks during settlement expose desynchronized share and asset accounting, enabling cross-function reentrancy and corrupted intermediate exchange rate and yield valuations.

### Technical details

In `Vault.sol`, `withdraw` (lines 509–521) and `redeem` (lines 524–535) compute the share/asset conversion using `_convertToShares` / `_convertToAssets` and delegate settlement to `_withdraw` (lines 518 and 532) without any reentrancy guard:

```solidity
function withdraw(
  uint256 _assets,
  address _receiver,
  address _owner
) public virtual override returns (uint256) {
  if (_assets > maxWithdraw(_owner))
    revert WithdrawMoreThanMax(_owner, _assets, maxWithdraw(_owner));

  uint256 _shares = _convertToShares(_assets, Math.Rounding.Up);
  _withdraw(msg.sender, _receiver, _owner, _assets, _shares);

  return _shares;
}
```

Unlike a standalone `ERC4626` contract that holds its underlying assets directly, `Vault` delegates share tracking to `_twabController` and asset custody to `_yieldVault`. When `withdraw` or `redeem` invokes `_withdraw` (lines 1007–1030), settlement occurs across three distinct steps:

1. `_burn(_owner, _shares)` (line 1024) calls `_twabController.burn(_owner, uint96(_shares))`—immediately reducing `_totalSupply()` and `_totalShares()`—and then calls `_updateExchangeRate()` (line 1140) *before* `_assets` are withdrawn from `_yieldVault`.
2. `_yieldVault.withdraw(_assets, address(this), address(this))` (line 1026) withdraws `_assets` from `_yieldVault` to `address(this)` (`Vault`), temporarily increasing `IERC20(asset()).balanceOf(address(this))` (`super.totalAssets()`).
3. `SafeERC20.safeTransfer(IERC20(asset()), _receiver, _assets)` (line 1027) transfers `_assets` from `address(this)` to `_receiver`.

Because `withdraw` and `redeem` do not enforce a reentrancy lock and do not synchronize share burning with asset outflow, two accounting invariants are violated during withdrawal settlement:

1. **Callback-Induced Share and Asset Desynchronization & Read-Only Reentrancy**:
   - When `_burn` runs (and during `_yieldVault.withdraw` before assets leave `_yieldVault`), `_totalSupply()` has already been reduced by `_shares`, while `_yieldVault.maxWithdraw(address(this))` still includes `_assets`. Consequently, `_currentExchangeRate()` (line 1168) evaluates `_withdrawableAssets` against the reduced `_totalSupplyAmount`. When the vault is undercollateralized (`_currentExchangeRate() < _assetUnit`), this premature evaluation artificially inflates `_currentExchangeRate()` and records an overstated `_lastRecordedExchangeRate`.
   - Furthermore, after `_yieldVault.withdraw` transfers `_assets` into `address(this)` and before `SafeERC20.safeTransfer` completes (e.g., during a `_yieldVault` callback or an ERC777 `tokensToSend` hook), `_totalAssets()` (`_yieldVault.maxWithdraw(address(this)) + super.totalAssets()`, line 801) still includes the `_assets` sitting in `address(this)`, whereas `_totalShares()` has already been reduced by `_shares`. As a result, `availableYieldBalance()` (line 311) and `_liquidatableBalanceOf()` (line 829) treat the withdrawing user's `_assets` as unbacked excess yield.

2. **Cross-Function Reentrancy via Transient Vault Balance**:
   - While `_assets` are transiently held by `address(this)` during withdrawal settlement, `_deposit` (lines 932–959) and `liquidate` (lines 578–584) inspect `uint256 _vaultAssets = _asset.balanceOf(address(this))`.
   - In `_deposit`, if a reentrant call requests `_assets <= _vaultAssets`, the condition `if (_assets > _vaultAssets)` (line 942) is false. `_deposit` skips `SafeERC20.safeTransferFrom`, deposits the `Vault`'s transient `_vaultAssets` back into `_yieldVault`, and mints new shares to the caller without pulling any underlying tokens, or `liquidate` mints yield shares against the inflated `availableYieldBalance()`.

### Assumptions and preconditions

1. The `Vault` holds deposits from one or more users in `_yieldVault`.
2. Either `_yieldVault` or the underlying `asset()` triggers an external callback during withdrawal settlement (such as an ERC777 hook or a yield vault withdrawal hook), or the vault is in an undercollateralized state where `_burn` updates `_lastRecordedExchangeRate` before `_yieldVault.withdraw` reduces withdrawable assets.

### Proof of concept

1. **Initial State**:
   - Alice and Bob each deposit `1,000e18` underlying assets into `Vault` (`_totalSupply() = 2,000e18`, `_yieldVault.maxWithdraw(address(vault)) = 2,000e18`, `_lastRecordedExchangeRate = 1e18`).
2. **Scenario A — Intermediate Valuation & Cross-Function Reentrancy During Withdrawal**:
   - Alice calls `vault.withdraw(1000e18, alice, alice)` (line 509) or `vault.redeem(1000e18, alice, alice)` (line 524).
   - `_withdraw` calls `_burn(alice, 1000e18)` (line 1024). `_totalSupply()` drops to `1,000e18`.
   - `_withdraw` calls `_yieldVault.withdraw(1000e18, address(vault), address(vault))` (line 1026).
   - During the transfer of `1,000e18` assets from `_yieldVault` to `Vault` (or before `SafeERC20.safeTransfer` to `alice` completes), `_yieldVault.maxWithdraw(address(vault)) + super.totalAssets()` is still `2,000e18`, while `_totalShares()` is already `1,000e18`.
   - A callback reentering `availableYieldBalance()` / `liquidate()` observes `availableYieldBalance() = 2,000e18 - 1,000e18 = 1,000e18` of phantom yield and liquidates it via `liquidate()`, minting unbacked Vault shares against Alice's withdrawing principal.
3. **Scenario B — Premature Exchange Rate Update Under Loss**:
   - Suppose `_yieldVault` suffers a `1,000e18` loss (`_yieldVault.maxWithdraw(address(vault)) = 1,000e18` for `2,000e18` shares, so the true exchange rate is `0.5e18`, while `_lastRecordedExchangeRate` is still `1e18`).
   - Alice calls `vault.withdraw(500e18, alice, alice)` (her `maxWithdraw(alice)`).
   - `_burn(alice, 1000e18)` burns Alice's `1,000e18` shares first, reducing `_totalSupply()` to `1,000e18`, and immediately invokes `_updateExchangeRate()`.
   - Inside `_currentExchangeRate()`, `_totalSupplyAmount` is `1,000e18`, `_totalSupplyToAssets` is `1,000e18`, and `_withdrawableAssets` is still `1,000e18` (since `500e18` has not yet been withdrawn from `_yieldVault`).
   - `_currentExchangeRate()` evaluates to `1,000e18 * 1e18 / 1,000e18 = 1e18`, recording `_lastRecordedExchangeRate = 1e18` instead of `0.5e18`, before `_yieldVault.withdraw(500e18, ...)` reduces the remaining assets to `500e18`.

### Impact assessment

- **Severity**: High.
- **Technical Impact**: Share burning and exchange rate recording occur out of order with respect to `_yieldVault` asset withdrawal, and external calls during `withdraw` / `redeem` are unprotected against cross-function and read-only reentrancy.
- **Business Impact**: Attackers can exploit the intermediate state during withdrawal callbacks to mint unbacked shares via `liquidate` or `deposit` or corrupt `_lastRecordedExchangeRate` and external share valuations, leading to direct loss of user principal and vault undercollateralization.

### Remediation

Add a `nonReentrant` guard (e.g., OpenZeppelin's `ReentrancyGuard`) to `withdraw` and `redeem` (as well as other state-changing entry points `deposit`, `mint`, `sponsor`, `liquidate`, and `mintYieldFee`), and ensure `_yieldVault` withdrawals settle directly to `_receiver` before `_updateExchangeRate()` records the post-withdrawal exchange rate:

```solidity
  /// @inheritdoc ERC4626
  function withdraw(
    uint256 _assets,
    address _receiver,
    address _owner
  ) public virtual override nonReentrant returns (uint256) {
    if (_assets > maxWithdraw(_owner))
      revert WithdrawMoreThanMax(_owner, _assets, maxWithdraw(_owner));

    uint256 _shares = _convertToShares(_assets, Math.Rounding.Up);
    _withdraw(msg.sender, _receiver, _owner, _assets, _shares);

    return _shares;
  }

  /// @inheritdoc ERC4626
  function redeem(
    uint256 _shares,
    address _receiver,
    address _owner
  ) public virtual override nonReentrant returns (uint256) {
    if (_shares > maxRedeem(_owner)) revert RedeemMoreThanMax(_owner, _shares, maxRedeem(_owner));

    uint256 _assets = _convertToAssets(_shares, Math.Rounding.Down);
    _withdraw(msg.sender, _receiver, _owner, _assets, _shares);

    return _assets;
  }
```

## Finding 9: Rounding Down in `_yieldVault` During `Vault._deposit` Causes `Vault._mint` to Record an Undercollateralized Exchange Rate and Permanently DoS the Vault

### Synopsis

In `Vault._deposit` and `Vault._beforeMint`, `Vault` assumes a 1:1 conversion between deposited `_assets` and withdrawable assets in `_yieldVault`. Because ERC-4626 `_yieldVault.deposit` and `_yieldVault.maxWithdraw` both round down, `_mint` can record `_lastRecordedExchangeRate < _assetUnit`, permanently placing the `Vault` into an undercollateralized state and halting deposits, mints, and liquidations.

### Technical details

When a user calls `Vault.mint` (or `Vault.mintWithPermit` / `Vault.deposit`), `Vault._beforeMint` (lines 971–974) checks `_shares > maxMint(_receiver)` and computes the required `_assets` via `_convertToAssets(_shares, Math.Rounding.Up)`.

When the vault is collateralized (`_isVaultCollateralized() == true`), `_currentExchangeRate()` is equal to `_assetUnit` (`10 ** decimals`), because `_currentExchangeRate()` caps `_withdrawableAssets` at `_totalSupplyToAssets` (lines 1178–1180) and can never exceed `_assetUnit`. Consequently, for any collateralized mint or deposit, `_assets == _shares`.

Next, `Vault._deposit` (lines 925–963) transfers `_assets` into the vault, deposits `_assets` into the underlying ERC-4626 `_yieldVault`, and mints `_shares` to `_receiver`:

```solidity
_yieldVault.deposit(_assets, address(this));
_mint(_receiver, _shares);
```

Inside `_mint` (lines 1122–1127), `_twabController.mint(_receiver, uint96(_shares))` increases `_totalSupply()` by `_shares`, and `_updateExchangeRate()` immediately sets `_lastRecordedExchangeRate = _currentExchangeRate()`.

Inside `_currentExchangeRate()` (lines 1168–1187):
1. `_totalSupplyAmount` is the new total supply (increased by `_shares`, which equals `_assets`).
2. `_withdrawableAssets` is queried from `_yieldVault.maxWithdraw(address(this))`.
3. Under the ERC-4626 standard, `_yieldVault.deposit(_assets, address(this))` computes yield vault shares using `previewDeposit(_assets)`, which rounds **down**, and `_yieldVault.maxWithdraw(address(this))` converts the vault's yield vault shares back to assets using `_convertToAssets(..., Math.Rounding.Down)`, which also rounds **down**.

Whenever `_yieldVault` has a non-1:1 share-to-asset exchange rate (which is normal for any yield-bearing ERC-4626 vault) and `Vault` does not have an unliquidated yield buffer (e.g., when deploying a `Vault` on top of an existing `_yieldVault`, or immediately after `liquidate()` + `mintYieldFee()` has converted accrued yield into `Vault` shares), depositing `_assets` into `_yieldVault` can increase `_yieldVault.maxWithdraw(address(this))` by `_assets - 1` (1 wei less than `_shares`).

When this 1-wei truncation occurs:
1. `_withdrawableAssets` becomes `_totalSupplyAmount - 1`.
2. `_currentExchangeRate()` computes `_withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down)`, which evaluates to `_assetUnit - 1` (or lower).
3. `_mint` calls `_updateExchangeRate()`, recording `_lastRecordedExchangeRate = _assetUnit - 1`.
4. `_isVaultCollateralized()` (`_currentExchangeRate() >= _assetUnit`) becomes `false`.
5. Because `_currentExchangeRate()` caps `_withdrawableAssets` at `_totalSupplyToAssets = _convertToAssets(_totalSupplyAmount, _lastRecordedExchangeRate, Math.Rounding.Down)`, `_currentExchangeRate()` is bounded above by `_lastRecordedExchangeRate` (`< _assetUnit`). Even if `_yieldVault` subsequently accrues significant yield, `_currentExchangeRate()` can never recover to `_assetUnit` until 100% of `Vault` shares are burned (`_totalSupply() == 0`).

### Assumptions and preconditions

1. `_yieldVault` is a standard ERC-4626 vault whose share price is not an exact integer divisor of `_assets` (e.g., `_yieldVault` already holds assets and accrued yield prior to `Vault` deposit, or `Vault` has just had its yield liquidated and yield fees minted).
2. No excess unliquidated yield buffer is currently sitting in `_yieldVault` for `Vault` (or the rounding loss exceeds the remaining dust buffer).
3. Any user calls `mint`, `mintWithPermit`, or `deposit` with an amount that triggers a 1-wei downward rounding difference between `_assets` and the increase in `_yieldVault.maxWithdraw(address(this))`.

### Proof of concept

1. Suppose `_yieldVault` is an ERC-4626 vault with `100e18` total shares and `150e18` total assets (exchange rate 1.5 assets per share), and `Vault` initially has `_totalSupply() == 0` and `_assetUnit == 1e18`.
2. A user calls `vault.mint(100e18 + 1, user)` (or `vault.deposit(100e18 + 1, user)`):
   - `_beforeMint(100e18 + 1, user)` returns `_assets = 100e18 + 1` because `_currentExchangeRate()` is `1e18`.
3. Inside `Vault._deposit`:
   - `_yieldVault.deposit(100e18 + 1, address(vault))` is executed.
   - `_yieldVault` mints `floor((100e18 + 1) * 100e18 / 150e18) = 66666666666666666667` yield vault shares to `Vault`.
   - `_yieldVault.maxWithdraw(address(vault))` is now `floor(66666666666666666667 * 150e18 / 100e18) = 100e18` (1 wei less than `100e18 + 1`).
4. Still inside `Vault._deposit`, line 960 calls `_mint(user, 100e18 + 1)`:
   - `_twabController.mint(user, 100e18 + 1)` sets `_totalSupply()` to `100e18 + 1`.
   - `_updateExchangeRate()` calls `_currentExchangeRate()`.
   - In `_currentExchangeRate()`, `_withdrawableAssets = 100e18` and `_totalSupplyAmount = 100e18 + 1`.
   - `_withdrawableAssets.mulDiv(1e18, 100e18 + 1, Math.Rounding.Down)` returns `999999999999999999` (`1e18 - 1`).
   - `_lastRecordedExchangeRate` is set to `1e18 - 1`.
5. `vault.isVaultCollateralized()` is now `false` (`999999999999999999 < 1e18`):
   - `vault.maxDeposit(...)` and `vault.maxMint(...)` return `0`, reverting all future deposits and mints.
   - Even after `_yieldVault` accrues `10e18` in yield so `_withdrawableAssets` becomes `110e18`, `_currentExchangeRate()` caps `_withdrawableAssets` at `_totalSupplyToAssets = floor((100e18 + 1) * (1e18 - 1) / 1e18) = 100e18`, so `_currentExchangeRate()` remains `1e18 - 1` forever.
   - `vault.liquidate(...)` and `vault.mintYieldFee(...)` revert with `VaultUnderCollateralized()`.

### Impact assessment

- **Severity:** High
- **Technical Impact:** A 1-wei ERC-4626 rounding discrepancy during `Vault._deposit` / `Vault._mint` causes `_lastRecordedExchangeRate` to drop below `_assetUnit`. Because `_lastRecordedExchangeRate` acts as a permanent ceiling on `_currentExchangeRate()`, the `Vault` cannot recover its collateralization status through subsequent yield accrual.
- **Business Impact:** Permanent Denial of Service (DoS) of core vault operations—all new deposits, share mints, yield liquidations (`liquidate`), and fee claims (`mintYieldFee`) are blocked until every existing depositor withdraws 100% of their shares from the vault.

### Remediation

Ensure that `Vault._deposit` and `Vault._beforeMint` account for the exact assets/shares required by `_yieldVault` so that `_yieldVault.maxWithdraw(address(this))` does not fall below the minted `Vault` shares, or mint `Vault` shares corresponding to the actual withdrawable assets added to `_yieldVault` (and in `mint`, deposit enough assets via `_yieldVault.previewMint` / rounding buffer so withdrawable assets cover `_shares`):

```solidity
    uint256 _priorWithdrawableAssets = _yieldVault.maxWithdraw(address(this));
    _yieldVault.deposit(_assets, address(this));
    uint256 _withdrawableAssetsDiff = _yieldVault.maxWithdraw(address(this)) - _priorWithdrawableAssets;
    if (_withdrawableAssetsDiff < _shares) {
      _shares = _withdrawableAssetsDiff;
    }
    _mint(_receiver, _shares);

    emit Deposit(_caller, _receiver, _assets, _shares);
  }

  /**
   * @notice Compute the amount of assets to deposit before minting `_shares`.
   * @param _shares Amount of shares to mint
   * @param _receiver Address of the receiver of the vault shares
   * @return uint256 Amount of assets to deposit.
   */
  function _beforeMint(uint256 _shares, address _receiver) internal view returns (uint256) {
    if (_shares > maxMint(_receiver)) revert MintMoreThanMax(_receiver, _shares, maxMint(_receiver));
    uint256 _assets = _convertToAssets(_shares, Math.Rounding.Up);
    uint256 _yieldVaultShares = _yieldVault.previewDeposit(_assets);
    while (_yieldVault.convertToAssets(_yieldVaultShares) < _assets) {
      unchecked {
        ++_assets;
      }
      _yieldVaultShares = _yieldVault.previewDeposit(_assets);
    }
    return _assets;
  }
```

## Finding 10: Cross-Function and Read-Only Reentrancy in `Vault._withdraw` via Intermediate Accounting State and Transient Vault Asset Balance

### Synopsis

In `Vault._withdraw`, burning shares before calling `_yieldVault.withdraw(_assets, address(this), address(this))` creates an inconsistent intermediate state where `_totalShares()` is reduced while `_totalAssets()` and `asset().balanceOf(address(this))` still include the withdrawn `_assets`, enabling cross-function reentrancy into `deposit`/`mint`/`liquidate` to mint unbacked shares and steal funds.

### Technical details

In `Vault._withdraw` (lines 1014–1027), share redemptions and withdrawals are executed in the following sequence:

```solidity
_burn(_owner, _shares);

_yieldVault.withdraw(_assets, address(this), address(this));
SafeERC20.safeTransfer(IERC20(asset()), _receiver, _assets);
```

This execution order creates two compounding vulnerabilities during the external call `_yieldVault.withdraw(_assets, address(this), address(this))` and prior to the completion of `SafeERC20.safeTransfer`:

1. **Premature Share Burn and Exchange Rate Update (`_burn` at line 1024):**
   `_burn(_owner, _shares)` calls `_twabController.burn(_owner, uint96(_shares))`—immediately reducing `_totalSupply()` and `_totalShares()` by `_shares`—and then calls `_updateExchangeRate()` while `_yieldVault.maxWithdraw(address(this))` still holds the unwithdrawn `_assets`.
   - If the vault was undercollateralized (with a stale `_lastRecordedExchangeRate` higher than the true rate), `_currentExchangeRate()` evaluates `_withdrawableAssets` (not yet reduced by `_assets`) against the already-reduced `_totalSupplyAmount`, recording an artificially inflated `_lastRecordedExchangeRate`.
   - Furthermore, because `_totalShares()` has been reduced by `_shares` while `_totalAssets()` (`_yieldVault.maxWithdraw(address(this)) + super.totalAssets()`) still contains `_assets`, `availableYieldBalance()` (`_totalAssets() - _convertToAssets(_totalShares(), Math.Rounding.Down)`) is artificially inflated by `_assets`.

2. **Transient Asset Balance Held in `Vault` (`_yieldVault.withdraw` at line 1026):**
   Instead of instructing `_yieldVault` to send the withdrawn assets directly to `_receiver`, line 1026 withdraws `_assets` to `address(this)` (`Vault`) before forwarding them on line 1027.
   - Meanwhile, `Vault._deposit` (lines 932–957) checks `uint256 _vaultAssets = _asset.balanceOf(address(this))` and assumes any underlying tokens sitting in `Vault` are unallocated assets that can be used to fulfill a new deposit without pulling tokens from `_caller`:
     ```solidity
     if (_assets > _vaultAssets) {
       ...
       SafeERC20.safeTransferFrom(_asset, _caller, address(this), _assetsDeposit != 0 ? _assetsDeposit : _assets);
     }
     _yieldVault.deposit(_assets, address(this));
     _mint(_receiver, _shares);
     ```
   - Similarly, `liquidate` (lines 566–584) relies on `availableYieldBalance()` (which is inflated by `_assets`) and checks `uint256 _vaultAssets = IERC20(asset()).balanceOf(address(this))`.

If execution re-enters `Vault` during `_yieldVault.withdraw(_assets, address(this), address(this))` (e.g., via a callback or token hook triggered when `_yieldVault` retrieves or transfers the underlying asset):
- **Via `liquidate()`:** The attacker can liquidate up to `_assets` of phantom yield reported by the inflated `availableYieldBalance()`, minting unbacked Vault shares (and yield fee shares) while `_withdraw` still transfers `_assets` out to `_receiver`.
- **Via `deposit()` / `mint()`:** Once `_yieldVault` transfers `_assets` to `address(this)` during `_yieldVault.withdraw`, an attacker re-entering `deposit()` or `mint()` can use `Vault`'s transient `_vaultAssets` balance to mint new shares without transferring any underlying tokens of their own.

### Assumptions and preconditions

- The underlying `_yieldVault` or underlying `_asset` token allows execution control to pass to an external contract or hook during `_yieldVault.withdraw(_assets, address(this), address(this))` (for example, an ERC-4626 yield vault interacting with an external strategy/token with callbacks, or a token with pre/post-transfer hooks).
- The attacker holds Vault shares to initiate `withdraw` or `redeem`.

### Proof of concept

1. Assume `Vault` has `2,000` shares (`1,000` owned by Alice, `1,000` owned by the attacker) backed by `2,000` underlying assets in `_yieldVault`, with `_lastRecordedExchangeRate = 1e18` and `availableYieldBalance() = 0`.
2. The attacker calls `vault.withdraw(1000, attacker, attacker)`.
3. Inside `_withdraw`:
   - Line 1024: `_burn(attacker, 1000)` burns `1,000` shares in `_twabController`. `_totalSupply()` drops from `2,000` to `1,000`, while `_yieldVault.maxWithdraw(address(vault))` is still `2,000`.
   - Line 1026: `_yieldVault.withdraw(1000, address(vault), address(vault))` is invoked.
4. During `_yieldVault.withdraw`:
   - **Attack Path A (`liquidate` reentrancy):** Before the `1,000` assets leave `Vault`, `_totalAssets()` is `2,000` (either in `_yieldVault` or in `address(vault)`) while `_totalShares()` is `1,000`. `availableYieldBalance()` therefore reports `2,000 - 1,000 = 1,000` of available yield even though zero yield has accrued. The attacker re-enters `liquidate` (via the `LiquidationPair`) to mint up to `1,000` unbacked Vault shares in exchange for a small amount of prize tokens.
   - **Attack Path B (`deposit`/`mint` reentrancy):** If the callback occurs after `_yieldVault` transfers the `1,000` underlying assets to `address(vault)` (Plus any existing latent balance in `Vault`), `asset.balanceOf(address(vault))` is `1,000`. Re-entering `vault.deposit(1000, attacker)` sees `_vaultAssets = 1000`, skips `SafeERC20.safeTransferFrom`, re-deposits the `1,000` assets into `_yieldVault`, and mints `1,000` shares to the attacker without transferring any tokens from the attacker.
5. When `_yieldVault.withdraw` and `SafeERC20.safeTransfer` complete (in Path A, or if `Vault` held prior latent assets in Path B), the attacker receives the `1,000` withdrawn underlying assets in addition to the newly minted Vault shares, leaving the Vault undercollateralized at the expense of Alice's principal.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Accounting invariants (`_totalAssets() >= _convertToAssets(_totalShares())` and accurate `_lastRecordedExchangeRate`) are violated during `_withdraw`, allowing unbacked share minting via `liquidate` or `deposit`/`mint`.
- **Business Implications:** Loss of user principal in the Vault, violating the core protocol invariant that users cannot access each other's principal and liquidators cannot extract more than actual accrued yield.

### Remediation

Add a `nonReentrant` guard to state-changing Vault operations, withdraw assets directly from `_yieldVault` to `_receiver` instead of routing them through `address(this)`, and ensure `_updateExchangeRate()` is called only after both the shares are burned and the assets have been withdrawn from `_yieldVault`.

```solidity
  function _withdraw(
    address _caller,
    address _receiver,
    address _owner,
    uint256 _assets,
    uint256 _shares
  ) internal virtual override {
    if (_caller != _owner) {
      _spendAllowance(_owner, _caller, _shares);
    }

    _twabController.burn(_owner, uint96(_shares));
    emit Transfer(_owner, address(0), _shares);

    _yieldVault.withdraw(_assets, _receiver, address(this));

    _updateExchangeRate();

    emit Withdraw(_caller, _receiver, _owner, _assets, _shares);
  }
```

## Finding 11: Premature Exchange Rate Update in `Vault._mint` and `Vault._updateExchangeRate` During Yield Liquidation Causes False Undercollateralization and Vault DoS

### Synopsis

When `Vault.liquidate` mints shares against idle assets held in the `Vault` (`_amountOut < _vaultAssets`), `Vault._mint` increases `_totalSupply` and invokes `Vault._updateExchangeRate` while those assets remain outside `_yieldVault`. Because `_currentExchangeRate` ignores idle vault assets, `_lastRecordedExchangeRate` drops below `_assetUnit`, bricking the vault.

### Technical details

In `Vault.sol`, total managed assets are defined in `_totalAssets()` (lines 800–802) as the sum of withdrawable assets in `_yieldVault` and idle underlying assets held directly by the `Vault` (`super.totalAssets()`):

```solidity
function _totalAssets() internal view returns (uint256) {
  return _yieldVault.maxWithdraw(address(this)) + super.totalAssets();
}
```

`availableYieldBalance()` (lines 311–316) and `_liquidatableBalanceOf()` (lines 829–837) use `_totalAssets()` to determine how many Vault shares can be minted via `liquidate()`. Any underlying tokens held directly in `address(this)` (for example, from direct transfers, donations, or residual balances) are therefore included in `_liquidableYield`.

However, in `liquidate()` (lines 578–584):

```solidity
uint256 _vaultAssets = IERC20(asset()).balanceOf(address(this));

if (_vaultAssets != 0 && _amountOut >= _vaultAssets) {
  _yieldVault.deposit(_vaultAssets, address(this));
}

_mint(_account, _amountOut);
```

Whenever `_vaultAssets != 0` and `_amountOut < _vaultAssets`, the idle `_vaultAssets` are **not** deposited into `_yieldVault` before `_mint(_account, _amountOut)` (`1122–1126`) is executed. Notably, if `_vaultAssets` is the sole or primary source of available yield and `_yieldFeePercentage > 0`, `_liquidableYield` is `_vaultAssets * (FEE_PRECISION - _yieldFeePercentage) / FEE_PRECISION < _vaultAssets`, meaning `_amountOut < _vaultAssets` **always** holds (and even when `_yieldFeePercentage == 0`, any partial liquidation satisfies `_amountOut < _vaultAssets`).

When `_mint` (`1122–1126`) executes:
1. `_twabController.mint(_receiver, uint96(_shares))` (`1123`) increases `_totalSupply()` by `_amountOut`.
2. `_updateExchangeRate()` (`1124` -> `1109–1112`) immediately updates `_lastRecordedExchangeRate = _currentExchangeRate()` (`1110`).
3. `_currentExchangeRate()` (lines 1168–1187) measures asset backing exclusively via `_withdrawableAssets = _yieldVault.maxWithdraw(address(this))`, ignoring `super.totalAssets()` sitting in the `Vault`.

Because `_totalSupply()` has increased by `_amountOut` in `1123` without the corresponding `_vaultAssets` having been deposited into `_yieldVault` prior to `_updateExchangeRate()` in `1124`, `_withdrawableAssets < _totalSupplyToAssets`. Consequently, `_updateExchangeRate()` (`1110`) records `_lastRecordedExchangeRate < _assetUnit`.

Once `_lastRecordedExchangeRate < _assetUnit` is checkpointed:
- `_isVaultCollateralized()` (lines 1194–1196) returns `false`.
- `maxDeposit()` and `maxMint()` return `0`, causing all subsequent `deposit()` and `mint()` calls to revert.
- `liquidate()` and `mintYieldFee()` revert with `VaultUnderCollateralized()`.
- Depositors redeeming or withdrawing shares suffer a haircut proportional to `_amountOut / _totalSupply()`, while `_vaultAssets` remain stranded in the `Vault` because `_withdraw()` (lines 1026–1027) only pulls funds from `_yieldVault`.

### Assumptions and preconditions

1. The `Vault` has non-zero `_totalSupply()` from user deposits (`_lastRecordedExchangeRate == _assetUnit`).
2. A non-zero balance of underlying `asset()` exists directly in the `Vault` (`_vaultAssets > 0`), e.g., via a direct token transfer/donation by any user or attacker.
3. A liquidation call executes through `_liquidationPair` with `_amountOut < _vaultAssets` (which is mandatory whenever `_yieldFeePercentage > 0` and accrued yield in `_yieldVault` is less than the yield fee portion of `_vaultAssets`, or via any partial liquidation).

### Proof of concept

1. **Initial State**: Alice deposits `1,000e18` underlying assets into `Vault`.
   - `_yieldVault.maxWithdraw(address(vault)) = 1,000e18`
   - `_totalSupply() = 1,000e18`
   - `_lastRecordedExchangeRate = 1e18` (`_assetUnit`)
   - `_yieldFeePercentage = 1e8` (10%)
2. **Idle Assets Added to Vault**: `100e18` underlying assets are transferred directly to `address(vault)` (`_vaultAssets = 100e18`).
   - `_totalAssets()` is now `1,000e18 + 100e18 = 1,100e18`.
   - `availableYieldBalance()` returns `1,100e18 - 1,000e18 = 100e18`.
   - `_liquidatableBalanceOf(address(vault))` returns `100e18 - 10e18 = 90e18`.
3. **Liquidation Triggers `_mint` and `_updateExchangeRate`**: A user liquidates `_amountOut = 90e18` Vault shares via the `LiquidationPair`:
   - In `liquidate()`, `_vaultAssets = 100e18` and `_amountOut = 90e18`.
   - The check `if (_vaultAssets != 0 && _amountOut >= _vaultAssets)` (`90e18 >= 100e18`) evaluates to `false`, so the `100e18` idle assets are **not** deposited into `_yieldVault`.
   - `liquidate()` calls `_mint(_account, 90e18)` (`1122`).
   - `1123` mints `90e18` shares in `_twabController`, increasing `_totalSupply()` from `1,000e18` to `1,090e18`.
   - `1124` calls `_updateExchangeRate()` (`1109–1112`).
   - Inside `_currentExchangeRate()`, `_totalSupplyAmount = 1,090e18`, `_totalSupplyToAssets = 1,090e18`, and `_withdrawableAssets = _yieldVault.maxWithdraw(address(vault)) = 1,000e18`.
   - `_currentExchangeRate()` returns `1,000e18 * 1e18 / 1,090e18 = 917431192660550458` (`~0.917e18 < 1e18`).
   - `1110` sets `_lastRecordedExchangeRate = 917431192660550458`.
4. **Result**:
   - `_isVaultCollateralized()` is now `false` (`917431192660550458 < 1e18`).
   - All future calls to `deposit()`, `mint()`, `liquidate()`, and `mintYieldFee()` revert.
   - Alice can only withdraw `~917.43e18` assets instead of her `1,000e18` deposit, while the `100e18` idle assets remain locked in `address(vault)` because no new `deposit()` can be made to sweep them into `_yieldVault`.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Mismatch between the assets counted as liquidatable yield (`_totalAssets()`, which includes idle `Vault` balances) and the assets counted in `_currentExchangeRate()` when `_updateExchangeRate()` is called in `_mint()` corrupts `_lastRecordedExchangeRate` to a value below `_assetUnit`.
- **Business Impact**:
  1. **Permanent Denial of Service (DoS)**: Once `_lastRecordedExchangeRate < _assetUnit`, `_isVaultCollateralized()` becomes `false`, disabling `deposit()`, `mint()`, `liquidate()`, and `mintYieldFee()`.
  2. **Loss of User Funds / Stranded Assets**: Because `deposit()` and `liquidate()` are disabled when undercollateralized and `_withdraw()` only withdraws from `_yieldVault`, the idle `_vaultAssets` in `Vault` cannot be swept into `_yieldVault` or withdrawn, forcing existing depositors to absorb an immediate haircut equal to the minted liquidation shares.

### Remediation

Ensure that before `_updateExchangeRate()` checkpoints `_lastRecordedExchangeRate` upon minting shares in `_mint()` (or inside `_updateExchangeRate()`), any idle underlying assets held by the `Vault` needed to back the newly minted shares are deposited into `_yieldVault`:

```solidity
  function _updateExchangeRate() internal {
    uint256 _vaultAssets = IERC20(asset()).balanceOf(address(this));
    if (_vaultAssets != 0 && _vaultAssets <= _yieldVault.maxDeposit(address(this))) {
      _yieldVault.deposit(_vaultAssets, address(this));
    }

    _lastRecordedExchangeRate = _currentExchangeRate();
    emit RecordedExchangeRate(_lastRecordedExchangeRate);
  }
```

## Finding 12: Compounded Rounding Drift in `Vault._currentExchangeRate` Progressively Devalues Shares During Undercollateralization

### Synopsis

In `Vault._currentExchangeRate`, converting `_totalSupplyAmount` to `_totalSupplyToAssets` using `_lastRecordedExchangeRate` with downward rounding and then converting back to an exchange rate with downward rounding creates a lossy round-trip conversion. Each state update (`_mint` or `_burn`) compounds this truncation error, progressively ratcheting `_lastRecordedExchangeRate` downward and locking underlying assets.

### Technical details

`Vault._currentExchangeRate` (starting at line 1168) calculates the current exchange rate between the Vault's shares (`_totalSupplyAmount = _totalSupply()`) and the withdrawable underlying assets in `_yieldVault`:

1. At lines 1170–1174, it computes the maximum underlying assets backing the current share supply at the previously recorded exchange rate:
   ```solidity
   uint256 _totalSupplyToAssets = _convertToAssets(
     _totalSupplyAmount,
     _lastRecordedExchangeRate,
     Math.Rounding.Down
   );
   ```
   When `_totalSupplyAmount != 0` and `_lastRecordedExchangeRate != 0`, `_convertToAssets` computes:
   $$\text{\_totalSupplyToAssets} = \left\lfloor \frac{\text{\_totalSupplyAmount} \times \text{\_lastRecordedExchangeRate}}{\text{\_assetUnit}} \right\rfloor$$

2. At lines 1178–1180, if `_withdrawableAssets > _totalSupplyToAssets`, `_withdrawableAssets` is capped to `_totalSupplyToAssets`.

3. At lines 1182–1184, `_withdrawableAssets` is converted back into an exchange rate using `Math.Rounding.Down`:
   $$\text{rate}_{\text{new}} = \left\lfloor \frac{\text{\_withdrawableAssets} \times \text{\_assetUnit}}{\text{\_totalSupplyAmount}} \right\rfloor$$

Whenever the vault is undercollateralized (`0 < _lastRecordedExchangeRate < _assetUnit`) and $\text{\_totalSupplyAmount} \times \text{\_lastRecordedExchangeRate}$ is not an exact multiple of $\text{\_assetUnit}$:

$$\text{\_totalSupplyToAssets} \times \text{\_assetUnit} = \text{\_totalSupplyAmount} \times \text{\_lastRecordedExchangeRate} - \text{rem}, \quad \text{where } 1 \le \text{rem} < \text{\_assetUnit}$$

Even when no further loss has occurred in `_yieldVault` (so `_withdrawableAssets >= _totalSupplyToAssets` and `_withdrawableAssets` is capped to `_totalSupplyToAssets`), dividing by `_totalSupplyAmount` and flooring yields:

$$\text{rate}_{\text{new}} = \left\lfloor \frac{\text{\_totalSupplyToAssets} \times \text{\_assetUnit}}{\text{\_totalSupplyAmount}} \right\rfloor = \left\lfloor \text{\_lastRecordedExchangeRate} - \frac{\text{rem}}{\text{\_totalSupplyAmount}} \right\rfloor \le \text{\_lastRecordedExchangeRate} - 1$$

Because `_mint` (line 1124) and `_burn` (line 1140) invoke `_updateExchangeRate()` (line 1110), which sets `_lastRecordedExchangeRate = _currentExchangeRate()`, every call to `_mint` or `_burn` records this lower exchange rate into storage. While undercollateralized, `maxDeposit` and `maxMint` return `0`, which still permits zero-amount `deposit(0, _receiver)` and `mint(0, _receiver)` calls (as well as `redeem` and `withdraw` calls). Each such call invokes `_updateExchangeRate()` and compounds the downward rounding drift without any actual loss of underlying assets.

### Assumptions and preconditions

1. The `Vault` has non-zero share supply (`_totalSupply() > 0`) and experiences an undercollateralization event in `_yieldVault` such that `_withdrawableAssets < _totalSupplyToAssets` and `_lastRecordedExchangeRate < _assetUnit`.
2. `(_totalSupply() * _lastRecordedExchangeRate) % _assetUnit != 0` (for example, `_totalSupply()` is coprime to `_assetUnit = 10 ** decimals()`, or `_totalSupply() < _assetUnit`).

### Proof of concept

Consider a `Vault` with an 18-decimal underlying asset (`_assetUnit = 1e18`):

1. **Setup & Undercollateralization:**
   - Total supply is `_totalSupplyAmount = 3e18 - 1` (`2999999999999999999` shares), and `_yieldVault` holds `_withdrawableAssets = 2e18 - 1` (`1999999999999999999` assets) after a loss.
   - Before `_lastRecordedExchangeRate` is updated from `1e18`, `_currentExchangeRate()` computes:
     $$\left\lfloor \frac{(2 \times 10^{18} - 1) \times 10^{18}}{3 \times 10^{18} - 1} \right\rfloor = 666666666666666666$$

2. **First Exchange Rate Update:**
   - A user calls `vault.deposit(0, receiver)` (or withdraws/redeems). `_mint` calls `_updateExchangeRate()`, setting `_lastRecordedExchangeRate = 666666666666666666`.

3. **Compounded Drift on Subsequent Calls:**
   - On the next `vault.deposit(0, receiver)` call:
     - `_totalSupplyToAssets = floor((3e18 - 1) * 666666666666666666 / 1e18) = 1999999999999999997` (2 wei less than actual `_withdrawableAssets`).
     - `_withdrawableAssets` is capped to `1999999999999999997`.
     - `_currentExchangeRate()` returns `floor(1999999999999999997 * 1e18 / (3e18 - 1)) = 666666666666666665`.
     - `_lastRecordedExchangeRate` is updated to `666666666666666665`.
   - On the next `vault.deposit(0, receiver)` call:
     - `_totalSupplyToAssets = floor((3e18 - 1) * 666666666666666665 / 1e18) = 1999999999999999994`.
     - `_lastRecordedExchangeRate` is updated to `floor(1999999999999999994 * 1e18 / (3e18 - 1)) = 666666666666666664`.
   - When `_totalSupplyAmount < _assetUnit` (e.g., `_totalSupplyAmount = 3e17` and `_withdrawableAssets = 2e17`), the decay is much steeper:
     - Update 1: `_lastRecordedExchangeRate = 666666666666666666`.
     - Update 2: `_totalSupplyToAssets = 199999999999999999`, `_lastRecordedExchangeRate = 666666666666666663`.
     - Update 3: `_totalSupplyToAssets = 199999999999999998`, `_lastRecordedExchangeRate = 666666666666666660`.
     - Each call reduces `_totalSupplyToAssets` by 1 wei and `_lastRecordedExchangeRate` by $\approx \lfloor \text{\_assetUnit} / \text{\_totalSupplyAmount} \rfloor$ until all withdrawable value is eroded.

### Impact assessment

- **Severity:** Medium / High.
- **Direct Technical Effect:** `_lastRecordedExchangeRate` and `_currentExchangeRate()` decay monotonically across state updates while the vault is undercollateralized, even when `_yieldVault` suffers no further losses or accrues positive yield.
- **Business Implications:** Depositors receive progressively fewer underlying assets upon calling `withdraw` or `redeem`, and the difference becomes permanently unwithdrawable by share holders in `_yieldVault`. An attacker or griefing actor can intentionally accelerate this decay by repeatedly calling `deposit(0, receiver)` or `mint(0, receiver)`.

### Remediation

When `_withdrawableAssets` is greater than or equal to `_totalSupplyToAssets` (and `_lastRecordedExchangeRate != 0`), `_currentExchangeRate()` should return `_lastRecordedExchangeRate` directly instead of capping `_withdrawableAssets` to the rounded-down `_totalSupplyToAssets` and dividing by `_totalSupplyAmount` a second time. Additionally, round up when computing `_totalSupplyToAssets` so that normal partial withdrawals do not trigger a downward rate step.

```solidity
  function _currentExchangeRate() internal view returns (uint256) {
    uint256 _totalSupplyAmount = _totalSupply();
    if (_totalSupplyAmount == 0) {
      return _assetUnit;
    }

    uint256 _withdrawableAssets = _yieldVault.maxWithdraw(address(this));
    if (_withdrawableAssets == 0) {
      return 0;
    }

    if (_lastRecordedExchangeRate != 0) {
      uint256 _totalSupplyToAssets = _convertToAssets(
        _totalSupplyAmount,
        _lastRecordedExchangeRate,
        Math.Rounding.Up
      );

      if (_withdrawableAssets >= _totalSupplyToAssets) {
        return _lastRecordedExchangeRate;
      }
    }

    return _withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down);
  }
```

## Finding 13: Downward-Only Ratchet in `Vault._updateExchangeRate` and `Vault._mint` Permanently Bricks Vault Collateralization

### Synopsis

In `Vault._updateExchangeRate` (invoked by `Vault._mint`), `_lastRecordedExchangeRate` is updated to `_currentExchangeRate()`. Because `_currentExchangeRate()` caps withdrawable assets using `_lastRecordedExchangeRate` with downward rounding, recording any rate below `_assetUnit` creates an irreversible downward ratchet that permanently locks the vault in an undercollateralized state.

### Technical details

The `Vault` contract tracks share backing through `_lastRecordedExchangeRate`, which is checkpointed in `_updateExchangeRate()` whenever shares are minted (`_mint` at line 1124) or burned (`_burn` at line 1140):

```solidity
function _updateExchangeRate() internal {
  _lastRecordedExchangeRate = _currentExchangeRate();
  emit RecordedExchangeRate(_lastRecordedExchangeRate);
}
```

When `_currentExchangeRate()` (lines 1168–1187) computes the current exchange rate, it calculates the maximum assets attributable to the current share supply (`_totalSupplyToAssets`) using `_lastRecordedExchangeRate`, and caps `_withdrawableAssets` (`_yieldVault.maxWithdraw(address(this))`) to `_totalSupplyToAssets`:

```solidity
uint256 _totalSupplyAmount = _totalSupply();
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
```

This checkpointing mechanism has a critical accounting flaw when `_mint` calls `_updateExchangeRate()`:

1. **Irreversible Ceiling on Exchange Rate (`_lastRecordedExchangeRate`):** Because lines 1178–1180 cap `_withdrawableAssets` at `_totalSupplyToAssets = floor(_totalSupplyAmount * _lastRecordedExchangeRate / _assetUnit)`, line 1183 computes:
   $$\text{exchangeRate} = \left\lfloor \frac{\min(\text{withdrawableAssets}, \text{totalSupplyToAssets}) \times \text{assetUnit}}{\text{totalSupplyAmount}} \right\rfloor \le \text{lastRecordedExchangeRate}$$
   As long as `_totalSupplyAmount != 0`, `_currentExchangeRate()` can never exceed `_lastRecordedExchangeRate`.

2. **Triggering `_lastRecordedExchangeRate < _assetUnit` via `_mint`:**
   - **Zero-value deposits during temporary undercollateralization:** When `_withdrawableAssets < _totalSupplyToAssets`, `maxDeposit` returns `0`. However, calling `deposit(0, receiver)` or `mint(0, receiver)` passes the `if (_assets > maxDeposit(_receiver))` check (`0 > 0` is false) and calls `_mint(receiver, 0)`, which executes `_updateExchangeRate()` and locks in the undercollateralized `_lastRecordedExchangeRate`.
   - **Partial liquidation when vault holds raw assets:** In `liquidate()` (lines 578–584), `availableYieldBalance()` includes `super.totalAssets()` (`_vaultAssets`), but `_vaultAssets` are only deposited into `_yieldVault` if `_amountOut >= _vaultAssets`. When `_vaultAssets != 0 && _amountOut < _vaultAssets` (which always occurs if `_yieldFeePercentage > 0` and yield resides in the vault, or on partial liquidations of donated/held assets), `_mint(_account, _amountOut)` increases `_totalSupply()` via `_twabController.mint` without increasing `_yieldVault.maxWithdraw(address(this))`, causing `_updateExchangeRate()` to immediately record `_lastRecordedExchangeRate < _assetUnit`.
   - **ERC4626 rounding in `_deposit`:** In `_deposit()` (lines 959–960), `_yieldVault.deposit(_assets, address(this))` is followed by `_mint(_receiver, _shares)`. If `_yieldVault.maxWithdraw(address(this))` rounds down by even 1 wei relative to `_assets` when there is no unliquidated yield buffer, `_mint` increases `_totalSupply()` by `_shares` and `_updateExchangeRate()` records `_lastRecordedExchangeRate < _assetUnit`.

3. **Permanent Undercollateralization and Precision Decay:** Once `_lastRecordedExchangeRate < _assetUnit`, `_isVaultCollateralized()` (`_currentExchangeRate() >= _assetUnit`) permanently returns `false` (unless 100% of shares are burned to zero, which any holder of 1 wei of shares can prevent). Even if `_yieldVault` recovers from a loss or accrues yield exceeding the deficit, the excess assets are stripped out at line 1179. Furthermore, each subsequent `_updateExchangeRate()` call compounds downward rounding in `_convertToAssets(..., Math.Rounding.Down)` and `mulDiv(..., Math.Rounding.Down)`, ratcheting `_lastRecordedExchangeRate` strictly downward.

### Assumptions and preconditions

- The vault has a non-zero share supply (`_totalSupply() > 0`).
- Either `_yieldVault.maxWithdraw(address(this))` temporarily dips below `_totalSupplyToAssets` (e.g., due to a temporary yield vault loss or 1-wei ERC4626 rounding on deposit) and `_mint` (such as via `deposit(0, receiver)`) or `_burn` is called, or `liquidate` is called when `_vaultAssets > _amountOut`.

### Proof of concept

1. Alice deposits `1,000e18` underlying assets into `Vault`.
   - `_totalSupply() = 1,000e18`
   - `_yieldVault.maxWithdraw(address(vault)) = 1,000e18`
   - `_lastRecordedExchangeRate = 1e18` (`_assetUnit`)
2. Scenario A (Temporary loss / rounding dip): `_yieldVault` experiences a temporary loss (or rounding discrepancy) of `100e18`, reducing `_withdrawableAssets` to `900e18`.
   - Anyone calls `vault.deposit(0, alice)`. Because `maxDeposit(alice) == 0`, `0 > 0` is false and the call succeeds.
   - `_deposit` calls `_mint(alice, 0)` (line 1122), which calls `_updateExchangeRate()` (line 1124 -> line 1110).
   - `_lastRecordedExchangeRate` is updated to `0.9e18`.
3. Scenario B (Liquidation with assets in `Vault`): Alternatively, `100e18` assets are sent to `Vault` (`_vaultAssets = 100e18`) and `liquidate()` is called with `_amountOut = 50e18`.
   - Because `_amountOut < _vaultAssets`, line 580 skips `_yieldVault.deposit`.
   - Line 584 calls `_mint(_account, 50e18)`.
   - `_twabController.mint` increases `_totalSupply()` to `1,050e18`, while `_yieldVault.maxWithdraw(address(vault))` remains `1,000e18`.
   - `_updateExchangeRate()` sets `_lastRecordedExchangeRate = (1,000e18 * 1e18) / 1,050e18 = 952380952380952380 < 1e18`.
4. Subsequently, `_yieldVault` accrues `500e18` in yield (or recovers its loss), bringing `_withdrawableAssets` to `1,500e18` (well above the total share supply).
5. In `_currentExchangeRate()`, `_totalSupplyToAssets` is computed using the reduced `_lastRecordedExchangeRate` (`900e18` in Scenario A or `1,000e18` in Scenario B), and line 1179 caps `_withdrawableAssets` to `_totalSupplyToAssets`.
6. `_currentExchangeRate()` remains permanently stuck below `1e18` (`_assetUnit`). `_isVaultCollateralized()` returns `false` forever:
   - New deposits and mints revert (`maxDeposit == 0`, `maxMint == 0`).
   - Yield liquidations (`liquidate`) and fee mints (`mintYieldFee`) revert with `VaultUnderCollateralized()`.
   - The `500e18` accrued/recovered assets cannot be withdrawn by shareholders or liquidated for the prize pool, permanently freezing them.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Permanent Denial of Service of core vault operations (`deposit`, `mint`, `liquidate`, `mintYieldFee`), permanent freezing of recovered principal and future yield in `_yieldVault`, and irreversible dilution of existing shareholders' withdrawable backing.

### Remediation

Remove the cached `_lastRecordedExchangeRate` checkpointing from `_mint` / `_burn` (`_updateExchangeRate`), and compute the exchange rate in `_currentExchangeRate()` directly against `_totalShares()` (at `1:1` with `_assetUnit`) and total backing assets so that the vault can naturally recover collateralization when backing assets increase:

```solidity
  function _mint(address _receiver, uint256 _shares) internal virtual override {
    _twabController.mint(_receiver, uint96(_shares));

    emit Transfer(address(0), _receiver, _shares);
  }
```

And in `_currentExchangeRate()`, compute `_totalSupplyToAssets` directly from `_totalSupplyAmount` (or `_totalShares()`) without capping at a stale `_lastRecordedExchangeRate`:

```solidity
  function _currentExchangeRate() internal view returns (uint256) {
    uint256 _totalSupplyAmount = _totalSupply();
    uint256 _withdrawableAssets = _yieldVault.maxWithdraw(address(this));

    if (_withdrawableAssets > _totalSupplyAmount) {
      _withdrawableAssets = _totalSupplyAmount;
    }

    if (_totalSupplyAmount != 0 && _withdrawableAssets != 0) {
      return _withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down);
    }

    return _assetUnit;
  }
```

## Finding 14: Unsafe downcast to `uint96` in `Vault._burn` and `Vault._transfer` allows burning zero shares during withdrawals and forged transfers

### Synopsis

`Vault._burn` and `Vault._transfer` unsafely downcast the `uint256 _shares` parameter to `uint96` when calling `_twabController.burn` and `_twabController.transfer`. Quantities `>= 2^96` silently truncate modulo `2^96`, allowing a holder of `>= 2^96` shares to withdraw underlying assets while burning zero shares, or any caller to emit forged `Transfer` events.

### Technical details

In `Vault.sol`, user share balances and total supply are tracked by `TwabController`, where `TwabLib.AccountDetails` stores `balance` and `delegateBalance` as `uint112`:

```solidity
struct AccountDetails {
  uint112 balance;
  uint112 delegateBalance;
  uint16 nextObservationIndex;
  uint16 cardinality;
}
```

Although `Vault.maxDeposit` and `Vault.maxMint` cap a single deposit or mint call at `type(uint96).max`, they do not cap an account's cumulative balance. Because `AccountDetails.balance` is a `uint112`, an account can accumulate a share balance `>= 2^96` (up to `type(uint112).max`) across multiple deposits, mints, or transfers.

However, `Vault._burn` (line 1139) and `Vault._transfer` (line 1155) perform an unchecked narrowing cast `uint96(_shares)` before forwarding the share amount to `_twabController`:

```solidity
function _burn(address _owner, uint256 _shares) internal virtual override {
  _twabController.burn(_owner, uint96(_shares));
  _updateExchangeRate();

  emit Transfer(_owner, address(0), _shares);
}

function _transfer(address _from, address _to, uint256 _shares) internal virtual override {
  _twabController.transfer(_from, _to, uint96(_shares));

  emit Transfer(_from, _to, _shares);
}
```

This causes two critical accounting failures when `_shares > type(uint96).max`:

1. **Unbacked withdrawals / vault drainage via `_burn`**: In `Vault.withdraw` and `Vault.redeem`, `maxWithdraw(_owner)` and `maxRedeem(_owner)` are derived from `balanceOf(_owner)`, which returns the full `uint112` balance from `TwabController`. If `_owner` holds at least `2^96` shares and calls `redeem(2^96, receiver, owner)` (or `withdraw` for the equivalent asset amount), the `maxRedeem` / `maxWithdraw` check passes. `_withdraw` then calls `_burn(_owner, 2^96)`, which truncates `uint96(2^96)` to `0` and calls `_twabController.burn(_owner, 0)`. Zero shares are deducted from `_owner` and total supply, while `_withdraw` proceeds to withdraw the full `_assets` corresponding to `2^96` shares from `_yieldVault` and transfer them to `_receiver`.
2. **Truncated and forged transfers via `_transfer`**: `ERC20.transfer` and `ERC20.transferFrom` delegate directly to the overridden `Vault._transfer` without checking balances in `ERC20`. If any account (even with a `0` balance) calls `transfer(to, 2^96)`, `uint96(2^96)` truncates to `0`, `_twabController.transfer(_from, _to, 0)` succeeds without reverting, and `Vault` emits `Transfer(_from, _to, 2^96)`. Similarly, transferring `2^96 + x` shares from an account with `>= 2^96 + x` shares only moves `x` shares in `TwabController` while spending `2^96 + x` allowance and emitting `Transfer(_from, _to, 2^96 + x)`.

### Assumptions and preconditions

1. For the `_burn` exploitation path, theunderlying asset has sufficient supply/decimals (or a low unit value) for an actor to accumulate at least `2^96` vault shares across multiple deposits/mints.
2. For the `_transfer` exploitation path, no preconditions or token balances are required; any address can call `transfer(to, k * 2^96)`.

### Proof of concept

1. Assume the exchange rate is `1:1` (`1e18`). An attacker deposits `2^95` underlying assets twice into `Vault` (each deposit is `<= type(uint96).max`, so `maxDeposit` succeeds).
2. The attacker's balance in `TwabController` is now `2^95 + 2^95 = 2^96` shares (stored in `uint112 AccountDetails.balance`).
3. The attacker calls `vault.redeem(2^96, attacker, attacker)`.
4. `Vault.redeem` checks `2^96 <= maxRedeem(attacker)` (`2^96 <= 2^96`), which succeeds, and computes `_assets = 2^96`.
5. `Vault._withdraw` calls `_burn(attacker, 2^96)`.
6. Inside `Vault._burn`, `uint96(2^96)` evaluates to `0`, so `_twabController.burn(attacker, 0)` is executed, burning `0` shares from the attacker and `0` from total supply.
7. `Vault._withdraw` withdraws `2^96` underlying assets from `_yieldVault` and transfers them to `attacker`.
8. The attacker still holds `2^96` shares and repeats `vault.redeem(2^96, attacker, attacker)` to drain all remaining underlying assets held in `_yieldVault`.

### Impact assessment

- **Severity**: High / Critical.
- **Direct Technical Effect**: Silent integer truncation on `uint256` to `uint96` casts in `_burn` and `_transfer` breaks share accounting invariants, permits burning fewer shares than redeemed for assets, and allows emitting arbitrary `Transfer` events of multiples of `2^96` with zero balance.
- **Business Implications**: Total loss of funds in the vault if an attacker can acquire `2^96` shares (for example, with high-decimal or low-value tokens, or via flash loans), and corruption of off-chain indexers or integrators relying on `Transfer` events or `transferFrom` allowance consumption.

### Remediation

Use OpenZeppelin's `SafeCast.toUint96` (or an explicit `require(_shares <= type(uint96).max)` check) when converting `_shares` to `uint96` in `_burn` and `_transfer` (as well as `_mint`).

```solidity
  function _burn(address _owner, uint256 _shares) internal virtual override {
    _twabController.burn(_owner, SafeCast.toUint96(_shares));
    _updateExchangeRate();

    emit Transfer(_owner, address(0), _shares);
  }

  function _transfer(address _from, address _to, uint256 _shares) internal virtual override {
    _twabController.transfer(_from, _to, SafeCast.toUint96(_shares));

    emit Transfer(_from, _to, _shares);
  }
```

## Finding 15: Incorrect Zero-Asset Handling in `Vault._currentExchangeRate` Treats Completely Undercollateralized Vault as Collateralized

### Synopsis

In `Vault._currentExchangeRate`, when `_totalSupplyAmount != 0` and `_withdrawableAssets == 0`, the function falls through to return `_assetUnit` instead of `0`. This zero-value discontinuity makes a completely undercollateralized or paused vault appear fully collateralized, enabling deposits, mints, and yield-fee minting that dilute new depositors' funds.

### Technical details

`Vault._currentExchangeRate` (lines 1168–1187) calculates the exchange rate between the Vault's minted shares (`_totalSupply()`) and the underlying assets withdrawable from `_yieldVault` (`_yieldVault.maxWithdraw(address(this))`):

```solidity
function _currentExchangeRate() internal view returns (uint256) {
  uint256 _totalSupplyAmount = _totalSupply();
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

The initial exchange rate of `_assetUnit` (`10 ** decimals()`) is only valid when no shares exist (`_totalSupplyAmount == 0`). However, line 1182 combines the non-empty supply check (`_totalSupplyAmount != 0`) with the non-zero asset check (`_withdrawableAssets != 0`), falling through to `return _assetUnit` whenever either value is zero.

When shares have been minted (`_totalSupplyAmount != 0`) and `_withdrawableAssets == 0`—which occurs if `_yieldVault` suffers a total loss of assets, if `_yieldVault.maxWithdraw(address(this))` returns `0` due to withdrawal limits/pauses on the underlying ERC-4626 yield vault, or if `_totalSupplyToAssets` rounds down to `0`—`_currentExchangeRate()` discontinuously jumps to `_assetUnit` (1:1 collateralization) instead of returning `0`.

This breaks core collateralization invariants across `Vault`:

1. **`_isVaultCollateralized()` returns `true` instead of `false`**: Line 1195 checks `_currentExchangeRate() >= _assetUnit`. When `_withdrawableAssets == 0` and `_totalSupplyAmount > 0`, `_isVaultCollateralized()` evaluates to `true`.
2. **`maxDeposit` and `maxMint` do not halt deposits**: Lines 376 and 384 return `type(uint96).max` instead of `0`. If a new user deposits `_assets` while `_withdrawableAssets == 0` (after a loss in `_yieldVault`), `_convertToShares` mints shares at the `1:1` rate (`_assetUnit`). Once `_yieldVault.deposit(_assets, address(this))` executes in `_deposit` (line 959), `_withdrawableAssets` becomes `_assets > 0`. The subsequent `_mint` call invokes `_updateExchangeRate()`, where `_currentExchangeRate()` now enters the `if (_totalSupplyAmount != 0 && _withdrawableAssets != 0)` branch and recalculates the rate as `_assets * _assetUnit / (_totalSupplyAmount + _shares)`. The new depositor's assets are immediately diluted across all pre-existing unbacked shares (`_totalSupplyAmount`), allowing existing shareholders to withdraw the new depositor's funds.
3. **`mintYieldFee` bypasses `_requireVaultCollateralized()`**: Line 395 fails to revert when `_withdrawableAssets == 0`, allowing unbacked yield fee shares to be minted.

Additionally, in `_convertToShares` (line 871) and `_convertToAssets` (line 902), a zero `_exchangeRate` is treated as a 1:1 conversion (`(_shares == 0 || _exchangeRate == 0) ? _shares : ...`), which similarly misvalues shares whenever the exchange rate is `0`.

### Assumptions and preconditions

1. The Vault has a non-zero share supply (`_totalSupply() > 0`).
2. `_yieldVault.maxWithdraw(address(this))` returns `0` (e.g., due to a total loss of underlying assets in `_yieldVault`, or `_yieldVault` pausing/limiting withdrawals per EIP-4626) or `_totalSupplyToAssets` rounds down to `0`.

### Proof of concept

1. Alice deposits `1,000e18` underlying assets into `Vault`.
   - `_totalSupply()` is `1,000e18`.
   - `_lastRecordedExchangeRate` is `1e18` (`_assetUnit`).
   - `_yieldVault.maxWithdraw(address(vault))` is `1,000e18`.
2. `_yieldVault` loses all `1,000e18` underlying assets (analogous to `Undercollateralization.t.sol`, but for the full balance instead of half), so `_yieldVault.maxWithdraw(address(vault))` is now `0`.
3. `vault.isVaultCollateralized()` is called:
   - `_currentExchangeRate()` sees `_totalSupplyAmount = 1,000e18` and `_withdrawableAssets = 0`.
   - Line 1182 `if (_totalSupplyAmount != 0 && _withdrawableAssets != 0)` is `false`.
   - `_currentExchangeRate()` returns `1e18` (`_assetUnit`).
   - `vault.isVaultCollateralized()` returns `true`, and `vault.maxDeposit(bob)` returns `type(uint96).max`.
4. Bob deposits `1,000e18` underlying assets via `vault.deposit(1,000e18, bob)`:
   - `_shares = _convertToShares(1,000e18, Math.Rounding.Down) = 1,000e18` because `_currentExchangeRate()` returned `1e18`.
   - `_deposit` transfers `1,000e18` assets from Bob and deposits them into `_yieldVault`. Now `_yieldVault.maxWithdraw(address(vault))` is `1,000e18`.
   - `_mint(bob, 1,000e18)` increases `_totalSupply()` to `2,000e18` and calls `_updateExchangeRate()`.
   - Now both `_totalSupplyAmount` (`2,000e18`) and `_withdrawableAssets` (`1,000e18`) are non-zero, so `_currentExchangeRate()` returns `1,000e18 * 1e18 / 2,000e18 = 0.5e18`.
5. Alice immediately calls `vault.withdraw(vault.maxWithdraw(alice), alice, alice)` and withdraws `500e18` of Bob's just-deposited assets, causing an instant 50% loss to Bob.

### Impact assessment

- **Severity**: High / Medium
- **Technical Impact**: Complete bypass of the Vault's undercollateralization protections (`_isVaultCollateralized()`, `_requireVaultCollateralized()`, `maxDeposit()`, `maxMint()`) whenever `_withdrawableAssets` is `0`.
- **Business Impact**: New depositors can deposit into an insolvent or paused vault at an artificial 1:1 exchange rate and immediately lose up to almost 100% of their deposited assets to existing holders of unbacked shares. Unbacked yield fee shares can also be minted via `mintYieldFee`.

### Remediation

In `_currentExchangeRate()`, return `_assetUnit` only when `_totalSupplyAmount == 0`. When `_totalSupplyAmount != 0`, return `0` if `_withdrawableAssets == 0` (or let `mulDiv` compute `0`). Additionally, ensure `_convertToAssets` and `_convertToShares` do not treat `_exchangeRate == 0` as a 1:1 exchange rate.

```solidity
  function _currentExchangeRate() internal view returns (uint256) {
    uint256 _totalSupplyAmount = _totalSupply();
    if (_totalSupplyAmount == 0) {
      return _assetUnit;
    }

    uint256 _withdrawableAssets = _yieldVault.maxWithdraw(address(this));
    if (_withdrawableAssets == 0) {
      return 0;
    }

    uint256 _totalSupplyToAssets = _convertToAssets(
      _totalSupplyAmount,
      _lastRecordedExchangeRate,
      Math.Rounding.Down
    );

    if (_withdrawableAssets > _totalSupplyToAssets) {
      _withdrawableAssets = _totalSupplyToAssets;
    }

    return _withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down);
  }
```

## Finding 16: Unclaimed Yield Fee Shares Excluded From `Vault._currentExchangeRate` Causes Solvency Mismatch and Unbacked Fee Minting

### Synopsis

`Vault._currentExchangeRate` computes the exchange rate using `_totalSupply()` instead of `_totalShares()`, excluding accrued `_yieldFeeTotalSupply` liabilities. This asset–liability scope mismatch delays undercollateralization detection during losses and allows `mintYieldFee` to mint unbacked fee shares that dilute depositors and force the vault into undercollateralization.

### Technical details

In `Vault.sol`, total share liabilities consist of minted shares tracked in `_twabController` (`_totalSupply()`) plus accrued-but-unminted yield fee shares (`_yieldFeeTotalSupply`), represented together by `_totalShares()`:

```solidity
function _totalShares() internal view returns (uint256) {
  return _totalSupply() + _yieldFeeTotalSupply;
}
```

When yield is liquidated in `liquidate()`, `_amountOut` shares are minted to the liquidator (increasing `_totalSupply()`), while the protocol fee portion is credited to `_yieldFeeTotalSupply` via `_increaseYieldFeeBalance()`. The underlying assets backing both `_amountOut` and `_yieldFeeTotalSupply` remain in `_yieldVault`. Accordingly, `availableYieldBalance()` uses `_totalShares()` to compute the vault's total liabilities.

However, `_currentExchangeRate()` at line 1169 only queries `_totalSupply()`:

```solidity
function _currentExchangeRate() internal view returns (uint256) {
  uint256 _totalSupplyAmount = _totalSupply();
  uint256 _totalSupplyToAssets = _convertToAssets(
    _totalSupplyAmount,
    _lastRecordedExchangeRate,
    Math.Rounding.Down
  );
```

Because `_yieldFeeTotalSupply` is omitted from `_totalSupplyAmount`, `_currentExchangeRate()` understates the vault's share liabilities whenever `_yieldFeeTotalSupply > 0`. This creates two solvency accounting flaws:

1. **Unbacked `mintYieldFee` execution when losses do not exceed `_yieldFeeTotalSupply`**: If `_yieldVault` suffers a loss such that `_withdrawableAssets` falls below `_convertToAssets(_totalShares(), _lastRecordedExchangeRate, Math.Rounding.Down)` but remains greater than or equal to `_totalSupplyToAssets`, the vault is already undercollateralized relative to its total obligations. Yet `_currentExchangeRate()` still returns `_assetUnit`, so `_isVaultCollateralized()` returns `true`. Any caller can then invoke `mintYieldFee()`, which passes `_requireVaultCollateralized()`, converts `_yieldFeeTotalSupply` into minted `_totalSupply()`, and immediately pushes the vault into an undercollateralized state by socializing the loss onto depositors.
2. **Dangling `_yieldFeeTotalSupply` after undercollateralized withdrawals**: When a larger loss occurs (`_withdrawableAssets < _totalSupplyToAssets`), using `_totalSupply()` as the denominator distributes the assets that were backing `_yieldFeeTotalSupply` pro-rata to existing shareholders upon withdrawal, but `_yieldFeeTotalSupply` is never reduced. Once existing shares are redeemed (resetting `_currentExchangeRate()` to `_assetUnit`) and new users deposit, `_isVaultCollateralized()` is `true` again while `_yieldFeeTotalSupply` remains non-zero, allowing `mintYieldFee()` to mint unbacked shares against new depositors' principal.

### Assumptions and preconditions

1. The vault has a non-zero `_yieldFeePercentage` configured, and at least one yield liquidation has occurred, resulting in `_yieldFeeTotalSupply > 0`.
2. The underlying `_yieldVault` experiences a loss of assets (e.g., a loss smaller than or equal to the assets backing `_yieldFeeTotalSupply`, or a larger loss followed by full withdrawal and subsequent deposits).

### Proof of concept

1. Alice deposits `1,000e18` underlying assets into `Vault`. `_totalSupply()` is `1,000e18`, `_withdrawableAssets` is `1,000e18`, and `_lastRecordedExchangeRate` is `1e18`.
2. `_yieldVault` accrues `100e18` in yield (`_withdrawableAssets = 1,100e18`). With `_yieldFeePercentage = 10%` (`1e8`), `liquidate()` is called for `90e18` shares:
   - `_totalSupply()` becomes `1,090e18`.
   - `_yieldFeeTotalSupply` becomes `10e18`.
   - `_totalShares()` is `1,100e18`, backed by `1,100e18` withdrawable assets.
3. `_yieldVault` loses `10e18` assets, reducing `_withdrawableAssets` to `1,090e18`.
4. `_currentExchangeRate()` computes `_totalSupplyAmount = _totalSupply() = 1,090e18` and `_totalSupplyToAssets = 1,090e18`. Because `_withdrawableAssets (1,090e18) == _totalSupplyToAssets (1,090e18)`, `_currentExchangeRate()` returns `1e18` and `isVaultCollateralized()` returns `true`.
5. `mintYieldFee(10e18, yieldFeeRecipient)` is called:
   - `_requireVaultCollateralized()` succeeds because `_currentExchangeRate()` is `1e18`.
   - `10e18` shares are minted to `yieldFeeRecipient`, raising `_totalSupply()` to `1,100e18` while `_withdrawableAssets` is only `1,090e18`.
   - `_updateExchangeRate()` updates `_lastRecordedExchangeRate` to `1,090e18 * 1e18 / 1,100e18 ≈ 0.9909e18`, making the vault undercollateralized and forcing depositors to absorb the loss that should have been borne by the unminted yield fee.

### Impact assessment

- **Severity**: Medium / High
- **Prerequisites**: `_yieldFeeTotalSupply > 0` from prior yield liquidations and a loss event in `_yieldVault`.
- **Consequences**: `isVaultCollateralized()` and `exchangeRate()` overstate vault collateralization when unclaimed yield fees exist. Unbacked yield fee shares can be minted via `mintYieldFee()` during or after loss events, diluting user deposits and improperly rendering the vault undercollateralized.

### Remediation

Include `_yieldFeeTotalSupply` in the liability calculation of `_currentExchangeRate()` by using `_totalShares()` instead of `_totalSupply()`:

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

## Finding 17: Reliance on `_yieldVault.maxWithdraw` in `Vault._currentExchangeRate` Causes Liquidity-Dependent Solvency Misclassification and Permanent Principal Loss

### Synopsis

`Vault._currentExchangeRate` queries `_yieldVault.maxWithdraw(address(this))` to value the vault's holdings and determine collateralization. Because ERC-4626 `maxWithdraw` reflects immediate withdrawal liquidity rather than total share value, temporary illiquidity in the underlying yield vault misclassifies the `Vault` as undercollateralized and permanently ratchets down `_lastRecordedExchangeRate`.

### Technical details

In `Vault.sol`, `_currentExchangeRate()` computes the exchange rate between the vault's minted shares and the underlying assets held in `_yieldVault`:

```solidity
function _currentExchangeRate() internal view returns (uint256) {
  uint256 _totalSupplyAmount = _totalSupply();
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

Per the EIP-4626 specification, `IERC4626.maxWithdraw(address owner)` returns the maximum amount of underlying assets that can be immediately withdrawn by `owner`, accounting for current liquidity constraints, withdrawal limits, or timelocks in the underlying protocol. In lending-backed ERC-4626 vaults (such as Aave, Compound, or Morpho vaults), when market utilization is high, a large portion of the supplied assets is borrowed out; consequently, `_yieldVault.maxWithdraw(address(this))` drops below `_yieldVault.convertToAssets(_yieldVault.balanceOf(address(this)))` even though the vault has not suffered any loss or bad debt.

This design flaw creates two compounding failures:

1. **False Undercollateralization (DoS):** When `_yieldVault.maxWithdraw(address(this))` temporarily falls below `_totalSupplyToAssets`, `_currentExchangeRate()` returns a value strictly less than `_assetUnit` (provided `_withdrawableAssets != 0`). Consequently, `_isVaultCollateralized()` (`_currentExchangeRate() >= _assetUnit`) evaluates to `false`. This immediately disables deposits (`maxDeposit` returns `0`), mints (`maxMint` returns `0`), yield fee minting (`mintYieldFee` reverts with `VaultUnderCollateralized`), and yield liquidations (`liquidate` reverts with `VaultUnderCollateralized`).
2. **Irreversible Downward Exchange Rate Ratchet:** `_lastRecordedExchangeRate` is updated via `_updateExchangeRate()` on every `_mint` and `_burn` operation. While the vault is collateralized or even after `_isVaultCollateralized()` becomes `false`, an attacker or regular user can trigger `_updateExchangeRate()` by calling `deposit(0, receiver)`, `mint(0, receiver)`, or `withdraw(...)` / `redeem(...)`. This overwrites `_lastRecordedExchangeRate` with the depressed exchange rate. Because lines 1178–1180 cap `_withdrawableAssets` at `_totalSupplyToAssets` (which is derived from `_lastRecordedExchangeRate`), `_currentExchangeRate()` can never increase once `_lastRecordedExchangeRate` has been lowered. Even after borrowers repay or liquidity returns to `_yieldVault`, the `Vault` remains permanently stuck in an undercollateralized state, and the difference between depositors' original principal and the depressed `_totalSupplyToAssets` is treated as excess assets (`availableYieldBalance()`).

### Assumptions and preconditions

1. The `Vault` has non-zero deposits (`_totalSupply() > 0`) deployed into `_yieldVault`.
2. `_yieldVault` is a standard ERC-4626 vault whose `maxWithdraw` is bounded by available protocol liquidity (e.g., a lending market where utilization can exceed the unallocated cash reserve) or withdrawal limits.

### Proof of concept

1. Alice deposits `10,000e18` underlying assets into `Vault`. `Vault` deposits `10,000e18` into `_yieldVault` and mints `10,000e18` shares to Alice. `_lastRecordedExchangeRate` is `1e18`.
2. Borrowers in the underlying lending protocol borrow `4,000e18` of the underlying asset from `_yieldVault` (60% utilization). The true value of `Vault`'s shares in `_yieldVault` is still `10,000e18`, but `_yieldVault.maxWithdraw(address(vault))` now returns `6,000e18` (the available cash in the lending pool).
3. `_currentExchangeRate()` now computes `6,000e18 * 1e18 / 10,000e18 = 0.6e18`.
4. `isVaultCollateralized()` returns `false` (`0.6e18 < 1e18`), halting `deposit`, `mint`, `liquidate`, and `mintYieldFee`.
5. Anyone calls `vault.deposit(0, alice)` (which passes `0 <= maxDeposit(alice)` since `0 <= 0`) or a user withdraws a small amount. Inside `_mint` / `_burn`, `_updateExchangeRate()` sets `_lastRecordedExchangeRate = 0.6e18`.
6. Borrowers repay their loans in full, restoring `_yieldVault.maxWithdraw(address(vault))` to `10,000e18`.
7. Despite full liquidity restoration, `_totalSupplyToAssets` in `_currentExchangeRate()` is now `10,000e18 * 0.6e18 / 1e18 = 6,000e18`. Line 1179 caps `_withdrawableAssets` at `6,000e18`, locking `_currentExchangeRate()` at `0.6e18`.
8. Alice can only withdraw `6,000e18` (`maxWithdraw(alice) == 6,000e18`), permanently losing 40% (`4,000e18`) of her principal while the vault remains bricked until all shares are burned.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Temporary illiquidity in the external `_yieldVault` causes `_currentExchangeRate()` to misclassify the `Vault` as undercollateralized, and any state-updating mint/burn (including a zero-value deposit) permanently locks in the reduced exchange rate.
- **Business Impact:** Permanent loss of depositor principal (up to nearly 100% of temporarily illiquid funds) and prolonged or permanent denial of service for vault deposits, mints, yield liquidations, and fee claims.

### Remediation

Distinguish between the accounting valuation of the `Vault`'s shares in `_yieldVault` and the immediate withdrawal liquidity. Use `_yieldVault.convertToAssets(_yieldVault.balanceOf(address(this)))` (plus any idle assets accounted for by the vault if applicable) to determine the vault's solvency and exchange rate in `_currentExchangeRate()`, rather than `_yieldVault.maxWithdraw(address(this))`:

```solidity
  function _currentExchangeRate() internal view returns (uint256) {
    uint256 _totalSupplyAmount = _totalSupply();
    uint256 _totalSupplyToAssets = _convertToAssets(
      _totalSupplyAmount,
      _lastRecordedExchangeRate,
      Math.Rounding.Down
    );

    uint256 _withdrawableAssets = _yieldVault.convertToAssets(_yieldVault.balanceOf(address(this)));

    if (_withdrawableAssets > _totalSupplyToAssets) {
      _withdrawableAssets = _withdrawableAssets - (_withdrawableAssets - _totalSupplyToAssets);
    }

    if (_totalSupplyAmount != 0 && _withdrawableAssets != 0) {
      return _withdrawableAssets.mulDiv(_assetUnit, _totalSupplyAmount, Math.Rounding.Down);
    }

    return _assetUnit;
  }
```
