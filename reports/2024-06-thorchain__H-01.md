# Audit report — 2024-06-thorchain/H-01

## Finding 1: THORChain_Router._deposit omits the payer from Deposit events, redirecting smart-contract refunds to relayers

### Synopsis

`THORChain_Router._deposit` emits deposits without the actual payer, while Bifrost derives `Sender` from the top-level transaction signer. Relayed or smart-wallet deposits are therefore attributed to the executor, allowing failed cross-chain operations to refund the full inbound amount to an unrelated relayer.

### Technical details

The `Deposit` event emitted by `_deposit` contains the vault, asset, received amount, and memo, but it does not identify `msg.sender`.

This omission creates an identity mismatch for indirect Router calls:

- For ERC-20 deposits, `safeTransferFrom` transfers tokens from `msg.sender`, which is the immediate contract calling the Router.
- For native deposits, the immediate calling contract forwards the native currency.
- Bifrost nevertheless initializes `TxInItem.Sender` from the top-level Ethereum transaction signer.
- The deposit-event parser updates `To`, `Memo`, and `Coins`, but cannot replace `Sender` because the event does not contain the immediate payer.

Consequently, for a transaction such as `relayer -> smart wallet -> Router`, Bifrost reports the relayer as the inbound sender even though the smart wallet supplied the assets. This is reachable for approved smart-contract wallets, aggregators, multisigs, and meta-transaction integrations.

If THORChain later refunds a failed inbound using the observed sender address, the refund is sent to the relayer or transaction executor rather than to the contract that paid the Router. The receipt alone cannot reconstruct the correct payer without call tracing because the required identity was omitted from the event.

### Assumptions and preconditions

- An accepted smart contract, such as a whitelisted smart wallet, multisig, or aggregator, calls the Router indirectly.
- The top-level transaction signer or executor differs from the contract supplying the assets.
- The inbound operation reaches a normal refund path, such as an invalid memo or failed swap.
- Refund routing relies on `TxInItem.Sender` or the corresponding observed source address.
- No downstream component corrects the sender using an authenticated contract-supplied refund address or execution trace.
- The relayer or executor is untrusted but can submit an otherwise valid, user-authorized contract transaction.

### Proof of concept

1. Use an accepted smart wallet or multisig contract `C` that holds a supported ERC-20 token.
2. Have `C` approve the THORChain Router to spend that token.
3. Prepare a valid smart-wallet transaction instructing `C` to call `depositWithExpiry` with a memo that will enter THORChain's refund path.
4. Give the signed smart-wallet transaction to an untrusted executor `R`.
5. Let `R` submit the top-level Ethereum transaction. The call sequence is `R -> C -> THORChain_Router`.
6. During `_deposit`, `safeTransferFrom` transfers the tokens from `C`, because `C` is the Router's `msg.sender`.
7. Inspect the resulting `Deposit` event. It contains no field identifying `C`.
8. Follow Bifrost parsing: `TxInItem.Sender` is initialized to `R`, and the deposit parser cannot replace it.
9. Process the inbound until it is refunded.
10. Observe that the refund is addressed to `R`, while `C` and its users supplied and lost the deposited assets.

### Impact assessment

**Severity: High.**

An untrusted transaction executor can receive the entire refunded value of a contract-originated inbound. The attack does not require Router administration, vault-key compromise, or token manipulation; it requires an accepted indirect-calling integration and an inbound that fails or is refunded.

The worst case is theft of the full principal of large smart-wallet, multisig, or aggregator deposits. It also makes contract-wallet support unsafe under relayed execution, creates integration liabilities, and can produce refunds that are operationally unrecoverable even without a deliberately malicious executor.

### Remediation

Include an unforgeable payer in the deposit event and make Bifrost use that value as the observed sender. The payer should default to the Router's immediate `msg.sender`; `tx.origin` must not be used.

```solidity
event Deposit(
  address indexed sender,
  address indexed to,
  address indexed asset,
  uint amount,
  string memo
);

emit Deposit(msg.sender, vault, asset, safeAmount, memo);
```

```go
type vaultDepositEvent struct {
	Sender ecommon.Address
	To     ecommon.Address
	Asset  ecommon.Address
	Amount *big.Int
	Memo   string
}

txInItem.Sender = depositEvt.Sender.String()
txInItem.To = depositEvt.To.String()
txInItem.Memo = depositEvt.Memo
```

Coordinate the Router deployment with updates to the event ABI, event-signature hash, and Bifrost parser. If backward compatibility is required, parse the event version according to the emitting Router address. For already-deployed Routers, the alternative is deterministic execution tracing to recover the immediate caller of the specific Router invocation.

## Finding 2: Unrestricted `Deposit` Event Emission in `THORChain_Router._deposit` Enables Receipt-Wide Observation Suppression and Multi-Event Interpretation Inconsistencies

### Synopsis

`THORChain_Router._deposit` permits multiple sequential invocations and arbitrary token/zero-amount `Deposit` event emissions within a single transaction. Because Bifrost's `SmartContractLogParser.GetTxInItem` validates `To` and `Memo` consistency across events before verifying token validity and aborts on mismatch, conflicting `Deposit` events suppress observation of the entire transaction receipt.

### Technical details

In `chain/ethereum/contracts/THORChain_Router.sol`, `_deposit` (lines 143–160) handles native ETH and ERC-20 token deposits and emits `Deposit(vault, asset, safeAmount, memo)` at line 159:

1. **Per-Call vs. Per-Transaction Guard (`nonReentrant`)**:
   The `nonReentrant` modifier on `_deposit` (line 148) sets `_status = _ENTERED` upon entry and resets `_status = _NOT_ENTERED` upon exit (lines 119–124). As noted in `bifrost/pkg/chainclients/shared/evm/smartcontract_log_parser.go` (line 183, `// router contract , deposit function has re-entrance protection`), the off-chain log parser assumes `_deposit` prevents multiple conflicting deposit executions in the same transaction. However, `nonReentrant` only blocks nested reentrant calls within a single `_deposit` call frame; it does not prevent multiple sequential `depositWithExpiry` calls within the same transaction (e.g., via multicall, batching contracts, or aggregator hooks executing before or after router calls), nor does it prevent calls across old and new routers during router migrations.

2. **Unvalidated Asset, Amount, and Event Emission in `_deposit`**:
   - When `asset == address(0)` (line 150), `_deposit` ignores the `amount` parameter (line 146), sets `safeAmount = msg.value` (line 151), transfers `safeAmount` to `vault`, and emits `Deposit(vault, asset, safeAmount, memo)` even if `msg.value == 0`.
   - When `asset != address(0)` (line 154), `_deposit` calls `safeTransferFrom(asset, amount)` (line 156) on any arbitrary caller-supplied contract address `asset` without verifying that `asset` is a valid token or that `safeAmount > 0`, and unconditionally emits `Deposit(vault, asset, safeAmount, memo)` (line 159).

3. **Receipt-Wide Suppression and Multi-Event Ambiguity in Bifrost**:
   When `ETHScanner.getTxInFromSmartContract` (lines 801–803) and `EVMScanner.getTxInFromSmartContract` (lines 751–754) pass `receipt.Logs` to `SmartContractLogParser.GetTxInItem` (`smartcontract_log_parser.go`, lines 182–212):
   - For every `Deposit` event with a non-zero `Amount`, `GetTxInItem` checks whether `txInItem.To` (lines 193–195) or `txInItem.Memo` (lines 196–198) differs from a previously parsed event in the same transaction **before** resolving whether `depositEvt.Asset` is a supported token via `scp.assetResolver` (lines 199–206).
   - If a valid deposit (or prior router event) sets `txInItem.To` and `txInItem.Memo`, and a subsequent `_deposit` call in the same transaction emits a `Deposit` event with a different `vault` (`to`) or `memo`—even using a worthless/unwhitelisted ERC-20 token—`GetTxInItem` returns an error (`multiple events in the same transaction... ignore`). The block scanner then discards the entire transaction receipt, causing legitimate funds transferred to the vault or router in that transaction to go unobserved.
   - Conversely, if the unwhitelisted token `Deposit` event is emitted *before* the valid `Deposit` event, `scp.assetResolver` fails and `continue`s before setting `txInItem.To` and `txInItem.Memo`, making observation outcome order-dependent. Furthermore, when multiple `_deposit` calls share the same `vault` and `memo`, `GetTxInItem` blindly appends each event to `txInItem.Coins` (lines 210–211) without deduplicating or consolidating identical assets.

### Assumptions and preconditions

- A transaction interacts with `THORChain_Router` via a smart contract (such as an aggregator, batcher, or token/callback hook executing within the same transaction before or after the router call) that results in more than one `Deposit` event being emitted in the transaction receipt.
- Bifrost observes the transaction receipt using `ETHScanner` or `EVMScanner` and parses logs via `SmartContractLogParser.GetTxInItem`.

### Proof of concept

1. A user or aggregator contract initiates a transaction that performs a legitimate deposit via `THORChain_Router.depositWithExpiry(asgardVault, whitelistedToken, 1000e18, "SWAP:BTC.BTC:bc1...", expiry)`.
2. `_deposit` executes (lines 146–160), transfers `1000e18` of `whitelistedToken` to `THORChain_Router`, credits `_vaultAllowance[asgardVault][whitelistedToken]`, and emits `Deposit(asgardVault, whitelistedToken, 1000e18, "SWAP:BTC.BTC:bc1...")` (line 159). Upon completion, `nonReentrant` resets `_status` to `_NOT_ENTERED`.
3. Within the same transaction (e.g., via a hook in an external swap route before/after the router call or a batched call), `depositWithExpiry(otherAddress, unwhitelistedToken, 1, "SPOOF", expiry)` is called on `THORChain_Router`.
4. `_deposit` executes a second time, transfers `1` unit of `unwhitelistedToken`, and emits `Deposit(otherAddress, unwhitelistedToken, 1, "SPOOF")` (line 159).
5. When Bifrost scans the transaction receipt in `GetTxInItem`:
   - The first `Deposit` event populates `txInItem.To = asgardVault`, `txInItem.Memo = "SWAP:BTC.BTC:bc1..."`, and appends `1000e18` of `whitelistedToken` to `txInItem.Coins`.
   - When processing the second `Deposit` event, line 193 (`!strings.EqualFold(txInItem.To, depositEvt.To.String())`) and line 196 (`!strings.EqualFold(txInItem.Memo, depositEvt.Memo)`) evaluate to `true` before `assetResolver` is called at line 199.
   - `GetTxInItem` returns an error, causing `getTxInFromSmartContract` to discard the entire `txInItem`. The `1000e18` `whitelistedToken` remains locked in the router without being credited or refunded on THORChain.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: Emitting an additional `Deposit` event with a mismatched `vault` or `memo` (even with an unwhitelisted token) within the same transaction causes Bifrost to reject the entire transaction receipt, while multiple same-memo deposits append unmerged coin entries.
- **Business Implications**: Legitimate user or aggregator deposits transferred to THORChain vaults or router allowances in affected transactions are not observed by Bifrost, resulting in loss/freezing of deposited user funds requiring manual intervention or governance recovery.

### Remediation

Restrict `_deposit` so that zero-value deposits revert, require non-empty memos and valid vault recipients, and prevent multiple `_deposit` executions within the same transaction (for example, by tracking a transaction-scoped guard via transient storage or block/tx-level state when not called via internal router migration, and ensuring `safeAmount > 0`):

```solidity
  function _deposit(
    address payable vault,
    address asset,
    uint amount,
    string memory memo
  ) private nonReentrant {
    require(vault != address(0), "invalid vault");
    require(bytes(memo).length > 0, "empty memo");
    uint safeAmount;
    if (asset == address(0)) {
      safeAmount = msg.value;
      require(safeAmount > 0, "zero amount");
      bool success = vault.send(safeAmount);
      require(success);
    } else {
      require(msg.value == 0, "unexpected eth"); // protect user from accidentally locking up eth
      require(amount > 0, "zero amount");
      safeAmount = safeTransferFrom(asset, amount); // Transfer asset
      require(safeAmount > 0, "zero transferred amount");
      _vaultAllowance[vault][asset] += safeAmount; // Credit to chosen vault
    }
    emit Deposit(vault, asset, safeAmount, memo);
  }
```

Additionally, in `SmartContractLogParser.GetTxInItem`, resolve and validate `depositEvt.Asset` via `scp.assetResolver` before checking `txInItem.To` and `txInItem.Memo`, and merge duplicate assets in `txInItem.Coins`.

## Finding 3: Cross-Layer Amount Normalization Mismatch and Vault-Credit Divergence in `THORChain_Router._deposit`

### Synopsis

`THORChain_Router._deposit` accepts, custodies, and credits arbitrary raw token/ETH amounts (`safeAmount`) to `_vaultAllowance[vault][asset]` without enforcing positive normalized amounts or THORChain 8-decimal precision boundaries. When Bifrost truncates `Deposit` amounts via `amtConverter`, sub-unit deposits are silently ignored and fractional remainders become permanently stranded in `_vaultAllowance` and router custody.

### Technical details

In `chain/ethereum/contracts/THORChain_Router.sol`, `_deposit` (lines 143–160) handles both native ETH (`asset == address(0)`) and ERC-20 token deposits (`asset != address(0)`):

```solidity
if (asset == address(0)) {
  safeAmount = msg.value;
  bool success = vault.send(safeAmount);
  require(success);
} else {
  require(msg.value == 0, "unexpected eth");
  safeAmount = safeTransferFrom(asset, amount);
  _vaultAllowance[vault][asset] += safeAmount;
}
emit Deposit(vault, asset, safeAmount, memo);
```

This implementation exhibits a cross-layer accounting and custody mismatch with the Bifrost observer pipeline (`ETHScanner.getTxInFromSmartContract` in `ethereum_block_scanner.go`, `EVMScanner.getTxInFromSmartContract` in `evm_block_scanner.go`, and `SmartContractLogParser.GetTxInItem` in `smartcontract_log_parser.go`):

1. **Unvalidated Zero / Sub-Unit Deposits and Silent Fund Loss**:
   `_deposit` does not require `safeAmount > 0` or enforce that `safeAmount` is at least one THORChain base unit ($10^{\max(0, \text{decimals} - 8)}$). In `SmartContractLogParser.GetTxInItem` (lines 189–211):
   ```go
   if len(depositEvt.Amount.Bits()) == 0 {
       scp.logger.Info().Msg("deposit amount is 0, ignore")
       continue
   }
   ...
   decimals := scp.decimalResolver(depositEvt.Asset.String())
   txInItem.Coins = append(txInItem.Coins,
       common.NewCoin(asset, scp.amtConverter(depositEvt.Asset.String(), depositEvt.Amount)).WithDecimals(decimals))
   ```
   When a deposit of an asset with $\text{decimals} > 8$ (such as 18-decimal ETH or ERC-20 tokens) has $0 < \text{safeAmount} < 10^{\text{decimals} - 8}$, `len(depositEvt.Amount.Bits()) == 0` evaluates to `false` because the raw `uint256` is non-zero. However, `scp.amtConverter` divides the raw amount by $10^{\text{decimals} - 8}$, truncating the result to `0`. Consequently, `txInItem.Coins.IsEmpty()` evaluates to `true` in `getTxInFromSmartContract` (line 811 of `ethereum_block_scanner.go` and line 794 of `evm_block_scanner.go`), causing Bifrost to drop the transaction (`return nil, nil`) while the tokens/ETH have already been transferred on-chain into the router or vault.

2. **Pooled Custody and Vault-Credit Divergence**:
   For ERC-20 assets with $\text{decimals} > 8$, `_deposit` pulls the full raw `safeAmount` into the router's pooled custody (`address(this)`) and credits the exact raw `safeAmount` to `_vaultAllowance[vault][asset]`. Meanwhile, THORChain's consensus state only credits the vault with the truncated 8-decimal amount $\lfloor \text{safeAmount} / 10^{\text{decimals} - 8} \rfloor$. This routinely occurs during aggregator swaps (`swapIn`) or user deposits where the amount is not an exact multiple of $10^{\text{decimals} - 8}$. When THORChain later executes outbound transfers (`transferOut`) or vault churn migrations (`transferAllowance` and `returnVaultAssets`), the instructed amounts are reconstructed from THORChain's 8-decimal accounting and are therefore always multiples of $10^{\text{decimals} - 8}$. The untruncated remainder ($\text{safeAmount} \bmod 10^{\text{decimals} - 8}$) remains permanently credited to the retiring vault's `_vaultAllowance[vault][asset]` and locked inside the `THORChain_Router` contract.

### Assumptions and preconditions

- The deposited asset is native ETH (`address(0)`, 18 decimals) or a whitelisted ERC-20 token with more than 8 decimals (e.g., 18 decimals on Ethereum).
- A user or aggregator contract calls `depositWithExpiry` with an amount that is either strictly between $0$ and $10^{\text{decimals} - 8}$ (sub-unit deposit) or not an exact multiple of $10^{\text{decimals} - 8}$ (unaligned precision deposit, such as the exact output of an AMM swap in `THORChain_Aggregator.swapIn`).

### Proof of concept

1. Consider an 18-decimal ERC-20 token (where 1 THORChain 1e8 unit equals $10^{10}$ token base units) and an active Asgard vault `VAULT_A`.
2. **Case 1 (Sub-unit deposit loss)**:
   - A caller invokes `depositWithExpiry(VAULT_A, token, 9_999_999_999, "SWAP:THOR.RUNE:...", expiration)`.
   - `THORChain_Router._deposit` transfers `9_999_999_999` base units from the caller to the router, increments `_vaultAllowance[VAULT_A][token] += 9_999_999_999`, and emits `Deposit(VAULT_A, token, 9_999_999_999, ...)`.
   - Bifrost's `GetTxInItem` sees `len(depositEvt.Amount.Bits()) > 0`, calls `amtConverter` which computes $\lfloor 9\_999\_999\_999 / 10^{10} \rfloor = 0$, and produces an empty coin set (`txInItem.Coins.IsEmpty() == true`).
   - Bifrost ignores the transaction; the caller receives neither a swap nor a refund, and the tokens remain locked in the router.
3. **Case 2 (Vault-credit and pooled-custody divergence on unaligned deposits)**:
   - An aggregator `swapIn` swaps tokens for an 18-decimal asset and calls `depositWithExpiry(VAULT_A, token, 100_000_000_123_456_789_123, "SWAP:...", expiration)`.
   - `_deposit` credits `_vaultAllowance[VAULT_A][token]` with `100_000_000_123_456_789_123`.
   - Bifrost's `amtConverter` normalizes the amount to `10_000_000_012` (in 1e8 units), truncating `3_456_789_123` base units.
   - When `VAULT_A` later churns and migrates its full balance to `VAULT_B` via `transferAllowance`, THORChain instructs a transfer of `100_000_000_120_000_000_000` base units.
   - `3_456_789_123` base units remain permanently stranded in `_vaultAllowance[VAULT_A][token]` and in the router's ERC-20 token balance.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: On-chain vault allowances (`_vaultAllowance[vault][asset]`) and router token custody diverge from THORChain's off-chain vault accounting whenever deposits contain precision below $10^{-8}$, and deposits smaller than $10^{\text{decimals} - 8}$ base units are accepted on-chain but silently dropped by Bifrost without refund.
- **Business Impact**: Permanent lockup of user funds for sub-unit deposits and progressive accumulation of unrecoverable token dust across retired vaults in `THORChain_Router` (especially from aggregator `swapIn` calls that forward raw AMM swap outputs).

### Remediation

In `THORChain_Router._deposit`, validate that `safeAmount > 0` and normalize (or require alignment of) the deposit amount to THORChain's 8-decimal precision so that only amounts representable by Bifrost/THORChain are transferred, credited to `_vaultAllowance`, and emitted in `Deposit`.

```solidity
  function _deposit(
    address payable vault,
    address asset,
    uint amount,
    string memory memo
  ) private nonReentrant {
    require(vault != address(0) && vault != address(this), "invalid vault");
    uint safeAmount;
    if (asset == address(0)) {
      safeAmount = (msg.value / 1e10) * 1e10;
      require(safeAmount > 0 && safeAmount == msg.value, "invalid eth amount");
      bool success = vault.send(safeAmount);
      require(success);
    } else {
      require(msg.value == 0, "unexpected eth"); // protect user from accidentally locking up eth
      require(amount > 0, "zero amount");
      safeAmount = safeTransferFrom(asset, amount); // Transfer asset
      require(safeAmount > 0, "zero safeAmount");
      _vaultAllowance[vault][asset] += safeAmount; // Credit to chosen vault
    }
    emit Deposit(vault, asset, safeAmount, memo);
  }
```

*(Note: For ERC-20 assets with `decimals() > 8`, `amount` / `safeAmount` can also be checked against `10 ** (decimals - 8)` prior to transfer so that unaligned fractional dust is not pulled from the sender or left stranded in `_vaultAllowance`.)*

## Finding 4: Unvalidated Pooled Balance-Delta Attribution in `THORChain_Router._deposit` Causes Vault Credit and Custody Divergence

### Synopsis

In `THORChain_Router._deposit`, ERC-20 deposit amounts (`safeAmount`) are derived solely from the router's aggregate `balanceOf(address(this))` delta across `safeTransferFrom(asset, amount)`. Because `THORChain_Router` pools custody for all vaults, transfer-time balance changes, hooks, or transfer fees misattribute credits in `_vaultAllowance` and `Deposit` events, causing vault insolvency or unauthorized credit inflation.

### Technical details

`THORChain_Router` holds ERC-20 tokens for all THORChain vaults in a single pooled balance at `address(this)`, while tracking each vault's entitlement in `_vaultAllowance[vault][asset]` and reporting inbound amounts to Bifrost via the `Deposit` event:

```solidity
    } else {
      require(msg.value == 0, "unexpected eth"); // protect user from accidentally locking up eth
      safeAmount = safeTransferFrom(asset, amount); // Transfer asset
      _vaultAllowance[vault][asset] += safeAmount; // Credit to chosen vault
    }
    emit Deposit(vault, asset, safeAmount, memo);
```

At line 156, `_deposit` delegates the transfer to `safeTransferFrom(asset, amount)` (lines 438–453), which snapshots `_startBal = iERC20(_asset).balanceOf(address(this))`, executes `transferFrom(msg.sender, address(this), _amount)`, and returns `iERC20(_asset).balanceOf(address(this)) - _startBal`.

This design introduces a root-cause accounting flaw in `_deposit` due to relying on an unvalidated, pool-wide balance delta instead of enforcing exact conservation between the sender's debit, `amount`, and the vault credit:

1. **Pool-Wide Balance-Delta Misattribution (`safeAmount > amount`)**:
   `_startBal` is snapshotted before the external call to `asset.transferFrom`. While `nonReentrant` prevents re-entering `THORChain_Router`, it does not prevent external calls during `transferFrom` (such as ERC-777 `tokensToSend` hooks on `msg.sender`, or token contracts that settle pending rebases, reflections, or reward distributions on transfer). Because `address(this)` holds the entire pooled reserve of all vaults, any positive balance adjustment, yield/reflection accrual on the router's existing holdings, or third-party token distribution to `address(this)` triggered during `transferFrom` is included in `balanceOf(address(this)) - _startBal`. `_deposit` attributes this entire aggregate delta (`safeAmount`) to `_vaultAllowance[vault][asset]` (line 157) and emits `Deposit(vault, asset, safeAmount, memo)` (line 159). Bifrost's `SmartContractLogParser.GetTxInItem` (lines 182–212) parses `depositEvt.Amount` and credits the caller's memo (e.g., `SWAP`) with the inflated `safeAmount`.

2. **Pooled Custody and Vault-Credit Divergence on Non-Exact Transfers (`safeAmount < amount`)**:
   `_deposit` accepts `safeAmount < amount` when `asset` deducts a transfer fee. However, during router upgrades (`transferAllowance` or `returnVaultAssets` when `router != address(this)`), `_routerDeposit` (lines 466–483) deducts the full `_amount` from the old router's `_vaultAllowance` and calls `depositWithExpiry` -> `_deposit` on the new router. The new router's `_deposit` credits `_vaultAllowance[_vault][_asset]` with only the post-fee `safeAmount < _amount`, while THORChain tracks the vault as holding the pre-fee amount. Subsequent `transferOut` (line 199) or `transferAllowance` (line 461) calls signed by THORChain for the vault's tracked balance revert with an arithmetic underflow on `_vaultAllowance[msg.sender][asset] -= amount`, freezing vault funds.

### Assumptions and preconditions

1. An ERC-20 token interacted with via `depositWithExpiry` either supports transfer hooks (e.g., ERC-777 `tokensToSend`), accrues/settles balance adjustments (rebases, reflections, or rewards) on `address(this)` during `transferFrom`, or applies a transfer fee (e.g., fee-capable tokens on the 1inch token list such as PAXG or upgradable/fee-toggleable tokens).
2. `THORChain_Router` holds existing pooled token balances across one or more vaults at `address(this)` (for balance-delta inflation) or executes vault migrations/outbound transfers (for fee-induced vault credit divergence).

### Proof of concept

#### Scenario 1: Balance-Delta Misattribution on Pooled Reserves
1. `THORChain_Router` holds `1,000,000` units of `asset` in pooled custody at `address(this)` credited to `AsgardVault` in `_vaultAllowance[AsgardVault][asset]`.
2. A pending reward distribution, reflection settlement, or harvestable payout of `5,000` units of `asset` is claimable for `address(this)` (or triggered upon the next token transfer involving `address(this)`).
3. An attacker calls `depositWithExpiry(AsgardVault, asset, 100, "SWAP:THOR.RUNE:thor1attacker", expiration)` from a contract.
4. Inside `_deposit` (line 156), `safeTransferFrom` records `_startBal = 1,000,000` and invokes `asset.transferFrom(msg.sender, address(this), 100)`.
5. During `transferFrom` (either via `asset`'s internal transfer settlement on `address(this)` or via `msg.sender`'s pre-transfer `tokensToSend` hook triggering the external distribution to `address(this)`), `address(this)`'s balance increases by the `5,000` pool distribution in addition to the `100` transferred from `msg.sender`.
6. `safeTransferFrom` returns `safeAmount = 1,005,100 - 1,000,000 = 5,100`.
7. `_deposit` credits `_vaultAllowance[AsgardVault][asset] += 5,100` (line 157) and emits `Deposit(AsgardVault, asset, 5100, "SWAP:THOR.RUNE:thor1attacker")` (line 159).
8. Bifrost observes the `Deposit` event for `5,100` units and THORChain executes the swap for `5,100` units in favor of `thor1attacker`, even though the attacker only contributed `100` units.

#### Scenario 2: Vault Allowance Deficit During Router Migration
1. A user deposits `1,000` units of a 1% fee-on-transfer token via `depositWithExpiry`. `_deposit` receives `safeAmount = 990`, credits `_vaultAllowance[VaultA][asset] = 990`, and emits `Deposit(VaultA, asset, 990, memo)`. THORChain records `VaultA` as holding `990` units.
2. During a router upgrade, `VaultA` calls `returnVaultAssets(newRouter, VaultB, [Coin(asset, 990)], "MIGRATE:...")`.
3. `_routerDeposit` decrements `_vaultAllowance[VaultA][asset] -= 990` on the old router and calls `newRouter.depositWithExpiry(VaultB, asset, 990, "MIGRATE:...", type(uint).max)`.
4. `newRouter._deposit` receives `safeAmount = 981` after the 1% transfer fee and credits `_vaultAllowance[VaultB][asset] = 981`, while THORChain still tracks `990` units on `VaultB`.
5. When `VaultB` attempts to `transferOut` or `transferAllowance` its `990` units, `_vaultAllowance[VaultB][asset] -= 990` reverts due to underflow (`981 < 990`), permanently locking the `981` units in `newRouter`.

### Impact assessment

- **Severity**: High
- **Technical Impact**:
  1. Unauthorized inflation of `safeAmount` in `_vaultAllowance` and `Deposit` events when pool-level balance increases occur during `transferFrom`.
  2. Permanent divergence between `_vaultAllowance[vault][asset]`, `iERC20(asset).balanceOf(address(this))`, and THORChain's internal vault accounting when `safeAmount != amount`.
- **Business Impact**: Direct theft of yield/distributions accrued to pooled router reserves via inflated `SWAP`/`ADD` deposits, or permanent freezing of vault ERC-20 reserves during router upgrades and vault migrations due to `_vaultAllowance` underflows.

### Remediation

In `THORChain_Router._deposit`, enforce that `amount > 0` and that the exact `amount` is both debited from `msg.sender` and received by `address(this)` (`safeAmount == amount`), rejecting fee-on-transfer and balance-mutating tokens at the router boundary in accordance with THORChain's token whitelist policy.

```solidity
  function _deposit(
    address payable vault,
    address asset,
    uint amount,
    string memory memo
  ) private nonReentrant {
    uint safeAmount;
    if (asset == address(0)) {
      safeAmount = msg.value;
      require(safeAmount > 0, "zero amount");
      bool success = vault.send(safeAmount);
      require(success);
    } else {
      require(msg.value == 0, "unexpected eth"); // protect user from accidentally locking up eth
      require(amount > 0, "zero amount");
      uint senderStartBal = iERC20(asset).balanceOf(msg.sender);
      safeAmount = safeTransferFrom(asset, amount); // Transfer asset
      uint senderEndBal = iERC20(asset).balanceOf(msg.sender);
      require(
        safeAmount == amount && senderStartBal - senderEndBal == amount,
        "inexact transfer amount"
      );
      _vaultAllowance[vault][asset] += safeAmount; // Credit to chosen vault
    }
    emit Deposit(vault, asset, safeAmount, memo);
  }
```

## Finding 5: Cross-Router Migration via `THORChain_Router._routerDeposit` Emits External `Deposit` Logs Instead of Source-Router Migration Events

### Synopsis

In `THORChain_Router._routerDeposit`, cross-router allowance migrations delegate to `iROUTER(_router).depositWithExpiry(...)` without emitting a migration event on the source router, causing event provenance mismatches between the transaction target (`address(this)`) and the emitted `Deposit` log (`_router`), loss of `oldVault` attribution, and broken migration observation in Bifrost.

### Technical details

When a vault migrates ERC-20 allowances or returns vault assets in `THORChain_Router`, both `transferAllowance` (lines 165–178) and `returnVaultAssets` (lines 408–426) branch on `router == address(this)`:

- When `router == address(this)`, the router adjusts internal allowances and emits `TransferAllowance(msg.sender, newVault, asset, amount, memo)` or `VaultTransfer(msg.sender, asgard, coins, memo)` directly from `address(this)`.
- When `router != address(this)` (during a router upgrade/migration), execution enters `_routerDeposit` (lines 466–483):

```solidity
function _routerDeposit(
  address _router,
  address _vault,
  address _asset,
  uint _amount,
  string memory _memo
) internal {
  _vaultAllowance[msg.sender][_asset] -= _amount;
  safeApprove(_asset, _router, _amount);

  iROUTER(_router).depositWithExpiry(
    _vault,
    _asset,
    _amount,
    _memo,
    type(uint).max
  ); // Transfer by depositing
}
```

This implementation introduces three tightly coupled migration observation integrity flaws:

1. **Cross-router event provenance confusion & missing migration recognition:**
   The transaction's top-level target (`tx.to`) is the source router (`address(this)`), which emits no event in `_routerDeposit`. Instead, the external call at line 476 causes the target `_router` contract to execute `_deposit` and emit `Deposit(_vault, _asset, safeAmount, _memo)` with `log.address == _router`. Because Bifrost's `smartcontract_log_parser` (`GetTxInItem`) strictly requires valid events to be emitted by the watched router contract itself, logs emitted by a different router address during a cross-router call fail provenance checks and are ignored on the source router (or rejected as contract-originated deposits when `tx.to != _router`), causing on-chain token migration to occur without THORNode recognizing the migration.

2. **Duplicated/fragmented migration logs in batch returns:**
   When `returnVaultAssets` is called with `router != address(this)` and multiple entries in `coins[]`, `_routerDeposit` is invoked in a loop (lines 420–422), causing `_router.depositWithExpiry` to emit a separate `Deposit` event with the same migration `_memo` for each coin instead of a single `VaultTransfer` event, violating the single-action observation semantics expected by Bifrost.

3. **Vault attribution and amount mismatches:**
   Unlike `TransferAllowance` and `VaultTransfer` (which index both `oldVault` and `newVault`), the `Deposit` event emitted by `_router.depositWithExpiry` only records `to` (`_vault`) and omits `oldVault` (`msg.sender`), while inside `_router`, the caller (`msg.sender`) is the old router contract (`address(this)`) rather than the migrating vault. Additionally, line 473 decrements `_vaultAllowance[msg.sender][_asset]` by `_amount`, whereas `_router._deposit` credits `_vault` with `safeAmount` (the post-transfer balance delta), and `_routerDeposit` neither restricts `_asset != address(0)` nor validates `_router`, allowing unprivileged callers with zero allowance (`_amount == 0`) to trigger arbitrary external calls to `_router.depositWithExpiry`.

### Assumptions and preconditions

- Two `THORChain_Router` instances (an old router and a new router) are deployed during a router upgrade/migration, or a caller invokes `transferAllowance` / `returnVaultAssets` with `router != address(this)`.
- A vault holds a non-zero ERC-20 allowance on the old router (for migration) or any caller passes `_amount == 0`.
- Bifrost observes router transactions and parses emitted contract logs via `smartcontract_log_parser`.

### Proof of concept

1. Assume `ASGARD1` holds an allowance of `250,000 TOKEN` on `ROUTER1` (`_vaultAllowance[ASGARD1][TOKEN] = 250,000`), and `ROUTER2` is deployed as the new router.
2. During router migration, `ASGARD1` calls `ROUTER1.transferAllowance(ROUTER2.address, ASGARD2, TOKEN.address, 250000, "migrate:1234")` (or `YGGDRASIL1` calls `ROUTER1.returnVaultAssets(ROUTER2.address, ASGARD1, coins, "yggdrasil-:1234")`).
3. `ROUTER1` sees `router != address(this)` and calls `_routerDeposit(ROUTER2.address, ASGARD2, TOKEN.address, 250000, "migrate:1234")`.
4. `_routerDeposit` decrements `_vaultAllowance[ASGARD1][TOKEN]` on `ROUTER1`, approves `ROUTER2`, and calls `ROUTER2.depositWithExpiry(ASGARD2, TOKEN.address, 250000, "migrate:1234", type(uint).max)`.
5. `ROUTER2` pulls the `250,000 TOKEN` from `ROUTER1`, credits `_vaultAllowance[ASGARD2][TOKEN]` on `ROUTER2`, and emits `Deposit(ASGARD2, TOKEN.address, 250000, "migrate:1234")` from `ROUTER2.address`.
6. `ROUTER1` emits no `TransferAllowance` (or `VaultTransfer`) event. When Bifrost parses the transaction targeting `ROUTER1`, the log emitted by `ROUTER2` does not match `ROUTER1.address` (and lacks `oldVault` attribution `ASGARD1`), causing the migration observation to be missed or misattributed between routers.

### Impact assessment

- **Severity:** High
- **Technical Effect:** Cross-router allowance migrations via `_routerDeposit` move ERC-20 tokens on-chain from the old router to the new router without emitting a `TransferAllowance` or `VaultTransfer` event on the called router, while emitting `Deposit` events from a different contract address (`_router`) without source vault attribution.
- **Business Impact:** During router upgrades or vault migrations across routers, Bifrost fails to properly observe and attribute migrated ERC-20 funds, leading to desynchronized vault accounting between THORNode and the EVM chain, stalled vault retirement, and solvency discrepancies requiring manual intervention.

### Remediation

Emit the appropriate migration/allowance event on the source router after verifying the actual transferred amount, validate that `_asset != address(0)` and `_amount > 0`, and ensure cross-router migration events preserve `oldVault` (`msg.sender`) and `newVault` (`_vault`) attribution.

```solidity
  function _routerDeposit(
    address _router,
    address _vault,
    address _asset,
    uint _amount,
    string memory _memo
  ) internal {
    require(_asset != address(0), "THORChain_Router: invalid asset");
    require(_amount > 0, "THORChain_Router: amount must be greater than zero");
    _vaultAllowance[msg.sender][_asset] -= _amount;
    safeApprove(_asset, _router, _amount);

    uint startBal = iERC20(_asset).balanceOf(address(this));
    iROUTER(_router).depositWithExpiry(
      _vault,
      _asset,
      _amount,
      _memo,
      type(uint).max
    ); // Transfer by depositing
    require(
      startBal - iERC20(_asset).balanceOf(address(this)) == _amount,
      "THORChain_Router: incomplete router deposit"
    );
    emit TransferAllowance(msg.sender, _vault, _asset, _amount, _memo);
  }
```

## Finding 6: False success event in THORChain_Router.transferOut leaves rejecting ETH recipients unpaid

### Synopsis

`THORChain_Router.transferOut` emits `TransferOut` even when its native-ETH payment fails and is refunded to the vault. A rejecting contract recipient can therefore produce an authoritative-looking outbound log without receiving ETH, desynchronizing protocol settlement and leaving the intended recipient unpaid.

### Technical details

The highlighted event emission at line 206 occurs unconditionally after both the ERC-20 and native-ETH branches.

In the native branch, the router attempts delivery using `send`, which forwards only 2,300 gas and returns `false` when the recipient reverts or requires more gas. Instead of reverting, the failure path refunds the ETH to `msg.sender` and continues execution. Critically, `safeAmount` remains unchanged and still contains the attempted payment amount.

Consequently, line 206 emits a `TransferOut` event containing:

- the intended recipient;
- the native asset identifier;
- the full attempted amount; and
- the outbound memo,

even though the recipient’s balance did not increase and the value was returned to the vault. The transaction receipt is successful, making the false event indistinguishable from a completed payout to an observer relying on router logs.

This creates a native-value accounting mismatch between the event and the actual value flow. It is especially reachable for smart-contract wallets and recipients whose receive logic cannot execute within the `send` gas stipend.

### Assumptions and preconditions

- An honest vault submits a nonzero native-ETH outbound through `transferOut`.
- The destination is a contract that reverts, has no payable receive function, or requires more than 2,300 gas.
- The calling vault can receive the bounced ETH; this is normally true for an EOA-based vault.
- Protocol observers treat `TransferOut` as evidence of a completed outbound or otherwise do not verify the recipient’s actual balance change.
- No privileged or administrative account behaves maliciously.

### Proof of concept

1. Create a recipient contract whose `receive` and fallback functions revert.
2. Record the recipient’s ETH balance and the vault’s ETH balance.
3. From an authorized vault with sufficient ETH, call `transferOut` with:
   - `to` set to the rejecting contract;
   - `asset` set to the zero address;
   - a nonzero attached value; and
   - an outbound memo.
4. The router’s native `send` returns `false`.
5. The router refunds the same amount to the calling vault and continues rather than reverting.
6. The transaction succeeds and line 206 emits `TransferOut` naming the rejecting recipient and the full amount.
7. Verify that the recipient’s balance is unchanged while the vault recovered the payment.
8. Decode the receipt and observe that its event nevertheless represents the outbound as having transferred the amount to the recipient.

### Impact assessment

**Severity: Medium.**

The vulnerability does not directly transfer protocol funds to an attacker because the failed payment is returned to the vault. However, it can cause a full outbound amount to be recorded as paid when the recipient received nothing.

If router events are authoritative for cross-chain settlement, the protocol may finalize the outbound and remove the recipient’s entitlement without retrying. Legitimate smart-contract wallets are particularly exposed because `send`’s 2,300-gas stipend frequently causes otherwise valid recipients to reject payment. The resulting consequences include unpaid users, reconciliation failures, inconsistent cross-chain accounting, and manual recovery requirements.

### Remediation

The preferred fix is to make native delivery failure revert the transaction. Use a full-gas call under the existing reentrancy guard and require success. Reversion automatically restores the attached ETH to the vault and prevents `TransferOut` from being emitted.

```solidity
if (asset == address(0)) {
    safeAmount = msg.value;
    (bool success, ) = to.call{value: safeAmount}("");
    require(success, "native transfer failed");
}
```

If bounce-and-continue behavior is a protocol requirement, emit a distinct failure event and do not emit `TransferOut` for the failed payment. Off-chain observers must treat that failure as retryable rather than completed.

## Finding 7: Unverified Token Balance Delta in `THORChain_Router.transferOut` Causes Vault Allowance and Settlement Event Divergence

### Synopsis

In `THORChain_Router.transferOut`, ERC-20 outbound transfers rely solely on the low-level `transfer` call return status and hardcode `safeAmount = amount` without measuring actual balance changes, causing vault allowance accounting, router custody balances, and emitted `TransferOut` settlement amounts to diverge under nonstandard token transfer semantics.

### Technical details

In `chain/ethereum/contracts/THORChain_Router.sol`, the `_deposit` function uses `safeTransferFrom(asset, amount)` (lines 156–157) to measure the exact difference in `balanceOf(address(this))` before and after the token transfer, crediting `_vaultAllowance[vault][asset]` and emitting `Deposit` with the actual `safeAmount` received.

In contrast, `transferOut` (lines 185–207) handles ERC-20 token payouts in its `else` branch (lines 199–206) as follows:

```solidity
_vaultAllowance[msg.sender][asset] -= amount; // Reduce allowance
(bool success, bytes memory data) = asset.call(
  abi.encodeWithSignature("transfer(address,uint256)", to, amount)
);
require(success && (data.length == 0 || abi.decode(data, (bool))));
safeAmount = amount;
```

This implementation exhibits a fundamental token transfer accounting flaw:

1. **Unchecked Balance Change Semantics:** Unlike `safeTransferFrom`, `transferOut` does not check `IERC20(asset).balanceOf(address(this))` (or the recipient's balance delta) before and after the low-level `asset.call`. Instead, line 204 unconditionally assigns `safeAmount = amount`.
2. **Vault Allowance and Custody Balance Divergence:** Line 199 decrements `_vaultAllowance[msg.sender][asset]` by the nominal `amount` parameter rather than the actual amount debited from the router contract.
   - If `to == address(this)` (for example, via a user-supplied refund or outbound destination address pointing to the router), `transfer(address(this), amount)` succeeds while zero tokens leave the router, yet `_vaultAllowance[msg.sender][asset]` is reduced by `amount`, permanently stranding those tokens in the router without vault allowance backing.
   - If the token applies sender-side fees on `transfer` (debiting `amount + fee` from `address(this)`) or dynamic/rebase adjustments, the router's actual token custody balance decreases by an amount different from `_vaultAllowance`, causing subsequent `transferOut` calls by vaults to either revert due to insufficient token balance or leave unspendable surplus tokens locked in the contract.
3. **Low-Level Call Success Ambiguity:** `asset.call(...)` is executed without verifying contract existence (`asset.code.length > 0`) or actual balance movement; a call to an address without code (e.g., a self-destructed or un-deployed token address) returns `(true, "")`, passing line 203 while transferring nothing.
4. **Inaccurate Settlement Events:** Line 206 emits `TransferOut(msg.sender, to, asset, safeAmount, memo)` with `safeAmount = amount` even when the actual amount debited from the router or delivered to `to` differs, misleading Bifrost observers that rely on `TransferOut` logs for outbound accounting.

### Assumptions and preconditions

1. A vault holds a positive `_vaultAllowance[vault][asset]` from prior deposits into `THORChain_Router`.
2. An outbound transaction or refund (`TxOutbound`, `TxRefund`, or `TxRagnarok`) triggers `transferOut` on `THORChain_Router`.
3. Either the ERC-20 `asset` exhibits nonstandard balance-change behavior on `transfer` (such as transfer fees, deflationary deductions, or non-reverting no-op execution), or the recipient `to` is set to `address(this)`.

### Proof of concept

1. Users deposit `1,000` units of an ERC-20 token `T` into `THORChain_Router` via `depositWithExpiry` for `VaultA`. `_vaultAllowance[VaultA][T]` is `1,000` and `T.balanceOf(address(ROUTER))` is `1,000`.
2. **Scenario A (Self-transfer destination):** An attacker initiates a swap with a refund address (or outbound destination `to`) set to `address(ROUTER)` and a malformed memo so THORChain issues a refund of `500` `T`.
   - `VaultA` calls `ROUTER.transferOut(payable(address(ROUTER)), address(T), 500, "REFUND:...")`.
   - Line 199 reduces `_vaultAllowance[VaultA][T]` from `1,000` to `500`.
   - Lines 200–203 execute `T.transfer(address(ROUTER), 500)`, which succeeds while `T.balanceOf(address(ROUTER))` remains `1,000`.
   - Line 204 sets `safeAmount = 500` and line 206 emits `TransferOut(VaultA, address(ROUTER), T, 500, "REFUND:...")`.
   - `500` tokens are now permanently locked in `ROUTER` because no vault holds the allowance to transfer them out, while Bifrost records that `500` tokens left the vault/router.
3. **Scenario B (Token with sender-side fee or balance adjustment):** Token `T` deducts a 1% sender fee on `transfer` from the sender's balance (or deducts a fee from the transferred amount).
   - `VaultA` calls `ROUTER.transferOut(recipient, address(T), 500, "OUT:...")`.
   - `_vaultAllowance[VaultA][T]` is reduced by `500` (leaving `500` allowance), while `T.balanceOf(address(ROUTER))` decreases by `505` (leaving `495` tokens in the router).
   - When `VaultA` later attempts to `transferOut` its remaining `500` allowance, the call reverts in the token contract due to insufficient router balance (`495 < 500`), freezing the remaining funds.

### Impact assessment

- **Severity:** High / Medium
- **Technical Impact:** `_vaultAllowance` diverges from the actual ERC-20 token balance held by `THORChain_Router`, and the `TransferOut` event reports an inaccurate `safeAmount`.
- **Business Impact:** Divergence between tracked vault allowances and actual router token custody can cause subsequent vault outbound transactions (`transferOut` / `batchTransferOutV5`) to revert due to insufficient token balances, stranding user funds in the router, or permanently lock tokens inside the router contract without corresponding vault allowance.

### Remediation

Validate the recipient address, verify that `asset` is a deployed contract, and measure the actual token balance delta on `THORChain_Router` (or enforce exact balance reduction) during `transferOut`:

```solidity
    } else {
      require(to != address(this), "invalid recipient");
      require(asset.code.length > 0, "asset is not a contract");
      uint startBal = iERC20(asset).balanceOf(address(this));
      (bool success, bytes memory data) = asset.call(
        abi.encodeWithSignature("transfer(address,uint256)", to, amount)
      );
      require(success && (data.length == 0 || abi.decode(data, (bool))));
      safeAmount = startBal - iERC20(asset).balanceOf(address(this));
      _vaultAllowance[msg.sender][asset] -= safeAmount; // Reduce allowance by actual debited amount
    }
    emit TransferOut(msg.sender, to, asset, safeAmount, memo);
  }
```

## Finding 8: Missing `msg.value` Validation and Native Value Accounting Mismatch in `THORChain_Router.transferOut` and `THORChain_Router._transferOutV5`

### Synopsis

In `THORChain_Router`, `transferOut` (`else` branch) and `_transferOutV5` do not validate `msg.value` against native ETH payouts. Non-zero `msg.value` sent during ERC-20 transfers or overfunded V5 calls becomes trapped in the router and can be drained by any caller via `transferOutV5` or `batchTransferOutV5`.

### Technical details

In `chain/ethereum/contracts/THORChain_Router.sol`, the router processes outbound asset transfers from vaults via `transferOut` (lines 185–207) and `_transferOutV5` (line 209, invoked by `transferOutV5` and `batchTransferOutV5`).

1. **Unchecked `msg.value` in ERC-20 `transferOut` (`199`–`207`)**:
   While `_deposit` explicitly enforces `require(msg.value == 0, "unexpected eth")` in its ERC-20 branch (line 155) to prevent callers from accidentally locking native ETH in the contract, `transferOut` is marked `payable` and omits this check in its `else` branch (`asset != address(0)`, starting at `199`). If a caller invokes `transferOut` for an ERC-20 token while supplying `msg.value > 0`, the ERC-20 allowance is decremented (`199`), the token transfer is executed (`200`–`203`), `safeAmount` is set to `amount` (`204`), and the `TransferOut` event is emitted (`206`), leaving `msg.value` permanently retained in `THORChain_Router` without being forwarded or refunded.

2. **Call-Value / Payout Mismatch and Cross-Caller Balance Exposure in `_transferOutV5` (`209`)**:
   In `_transferOutV5(TransferOutData memory transferOutPayload)` (`209`), when `transferOutPayload.asset == address(0)`, the contract sends `transferOutPayload.amount` of native ETH (`transferOutPayload.to.send(transferOutPayload.amount)`) rather than `msg.value`, and when `transferOutPayload.asset != address(0)`, it does not require `msg.value == 0`. Neither `_transferOutV5`, `transferOutV5`, nor `batchTransferOutV5` enforces conservation between `msg.value` and the total native ETH disbursed across the call or batch. Consequently:
   - Any excess `msg.value` sent in `transferOut`, `transferOutV5`, or `batchTransferOutV5` remains in `address(this).balance`.
   - Because `transferOutV5` and `batchTransferOutV5` are publicly callable and `_transferOutV5` does not verify that `msg.value` covers `transferOutPayload.amount` (nor does it check `_vaultAllowance` for `asset == address(0)`), any unprivileged external caller can invoke `transferOutV5` (or `batchTransferOutV5`) with `asset = address(0)` and `msg.value = 0` to drain any native ETH balance held by `THORChain_Router`.

### Assumptions and preconditions

- A vault or user calls `transferOut` with `asset != address(0)` and accidentally includes `msg.value > 0`, or calls `transferOutV5` / `batchTransferOutV5` with `msg.value` exceeding the sum of `transferOutPayload.amount` for native ETH items (or ETH is otherwise present in `THORChain_Router`).
- An unprivileged actor observes the non-zero ETH balance in `THORChain_Router` (or a vault loses the excess ETH in `THORChain_Router`).

### Proof of concept

1. A vault calls `transferOut(recipient, erc20Asset, 1000e18, "OUT:...")` and inadvertently attaches `1 ether` (`msg.value = 1e18`).
2. Because `asset != address(0)`, execution enters the `else` branch at `199`. The router deducts `1000e18` from `_vaultAllowance[msg.sender][erc20Asset]`, transfers `1000e18` of `erc20Asset` to `recipient`, sets `safeAmount = 1000e18`, and emits `TransferOut`.
3. The `1 ether` supplied in `msg.value` is not rejected or refunded and remains in `address(ROUTER).balance`.
4. An arbitrary external account calls `transferOutV5(TransferOutData({to: attacker, asset: address(0), amount: 1 ether, memo: ""}))` with `msg.value = 0`.
5. `_transferOutV5` (`209`) executes `transferOutPayload.to.send(1 ether)` funded from `address(ROUTER).balance`, transferring the `1 ether` to `attacker`.

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: Native ETH sent alongside ERC-20 `transferOut` calls or overfunded V5 outbound calls is silently locked in `THORChain_Router` instead of reverting, and any retained ETH balance in the router can be stolen by any unprivileged caller via `_transferOutV5`.
- **Business Impact**: Loss of vault or user native ETH funds due to lack of call-value conservation and exposure of contract-held ETH balances.

### Remediation

Enforce `require(msg.value == 0, "unexpected eth")` in the ERC-20 branch of `transferOut`, and ensure that `transferOutV5` and `batchTransferOutV5` strictly conserve `msg.value` against the native ETH amounts transferred in `_transferOutV5`.

```solidity
  function transferOut(
    address payable to,
    address asset,
    uint amount,
    string memory memo
  ) public payable nonReentrant {
    uint safeAmount;
    if (asset == address(0)) {
      safeAmount = msg.value;
      bool success = to.send(safeAmount); // Send ETH.
      if (!success) {
        payable(address(msg.sender)).transfer(safeAmount); // For failure, bounce back to vault & continue.
      }
    } else {
      require(msg.value == 0, "unexpected eth");
      _vaultAllowance[msg.sender][asset] -= amount; // Reduce allowance
      (bool success, bytes memory data) = asset.call(
        abi.encodeWithSignature("transfer(address,uint256)", to, amount)
      );
      require(success && (data.length == 0 || abi.decode(data, (bool))));
      safeAmount = amount;
    }
    emit TransferOut(msg.sender, to, asset, safeAmount, memo);
  }

  function _transferOutV5(TransferOutData memory transferOutPayload) private returns (uint ethAmount) {
    if (transferOutPayload.asset == address(0)) {
      ethAmount = transferOutPayload.amount;
      bool success = transferOutPayload.to.send(transferOutPayload.amount); // Send ETH.
      if (!success) {
        payable(address(msg.sender)).transfer(transferOutPayload.amount); // For failure, bounce back to vault & continue.
      }
    } else {
      _vaultAllowance[msg.sender][
        transferOutPayload.asset
      ] -= transferOutPayload.amount; // Reduce allowance

      (bool success, bytes memory data) = transferOutPayload.asset.call(
        abi.encodeWithSignature(
          "transfer(address,uint256)",
          transferOutPayload.to,
          transferOutPayload.amount
        )
      );

      require(success && (data.length == 0 || abi.decode(data, (bool))));
    }

    emit TransferOut(
      msg.sender,
      transferOutPayload.to,
      transferOutPayload.asset,
      transferOutPayload.amount,
      transferOutPayload.memo
    );
  }

  function transferOutV5(
    TransferOutData calldata transferOutPayload
  ) public payable nonReentrant {
    uint ethTotal = _transferOutV5(transferOutPayload);
    require(msg.value == ethTotal, "unexpected eth");
  }

  // bifrost to budget gas limits, no more than 50% than L1 gas limit
  function batchTransferOutV5(
    TransferOutData[] calldata transferOutPayload
  ) external payable nonReentrant {
    uint ethTotal = 0;
    for (uint i = 0; i < transferOutPayload.length; ++i) {
      ethTotal += _transferOutV5(transferOutPayload[i]);
    }
    require(msg.value == ethTotal, "unexpected eth");
  }
```

## Finding 9: Unvalidated `msg.value` and Nominal Amount Usage in `THORChain_Router._transferOutV5` Causes ETH Accounting Discrepancies and Fund Loss

### Synopsis

In `THORChain_Router._transferOutV5`, native ETH outbound transfers use the nominal `transferOutPayload.amount` without validating it against `msg.value` or forbidding `msg.value` on ERC20 transfers. This discrepancy between nominal parameters and realized `msg.value` traps excess ETH in the router, allows permissionless draining of router ETH, and enables spoofed `TransferOut` events.

### Technical details

In V4 `THORChain_Router.transferOut` (lines 192–197), native asset transfers (`asset == address(0)`) ignored the nominal `amount` parameter and used `safeAmount = msg.value`, ensuring that the ETH forwarded to `to` (or bounced back to `msg.sender` on failure) and recorded in the `TransferOut` event always matched the actual ETH supplied in the call.

In V5, `_transferOutV5` (lines 209–238), which backs both `transferOutV5` and `batchTransferOutV5`, replaces `msg.value` with the nominal struct field `transferOutPayload.amount`:

```solidity
if (transferOutPayload.asset == address(0)) {
  bool success = transferOutPayload.to.send(transferOutPayload.amount); // Send ETH.
  if (!success) {
    payable(address(msg.sender)).transfer(transferOutPayload.amount); // For failure, bounce back to vault & continue.
  }
} else {
  _vaultAllowance[msg.sender][
    transferOutPayload.asset
  ] -= transferOutPayload.amount; // Reduce allowance
```

Neither `_transferOutV5`, `transferOutV5`, nor `batchTransferOutV5` validates that `msg.value` equals `transferOutPayload.amount` (for a single call) or the sum of `transferOutPayload[i].amount` across all native ETH payloads (for a batch call), nor does the `else` branch enforce `msg.value == 0` (unlike `_deposit` at line 155).

This creates several critical accounting flaws:
1. **Unaccounted ETH Custody and Loss of Funds:** If a vault calls `transferOutV5` or `batchTransferOutV5` with `msg.value` exceeding the sum of `transferOutPayload.amount` for `asset == address(0)` (for instance, accidentally supplying `msg.value` alongside an ERC20 transfer or miscalculating a batch total), the excess ETH remains trapped in `THORChain_Router`. Because `_transferOutV5` pulls `transferOutPayload.amount` from `address(this).balance` without checking `msg.value`, any external caller can subsequently call `transferOutV5` with `asset = address(0)`, `to = attacker`, `amount = address(router).balance`, and `msg.value = 0` to drain all ETH held by the router.
2. **Batch Failure / Insufficient ETH Revert:** If a batch contains multiple ETH outbounds where `msg.value` is less than the sum of `transferOutPayload[i].amount`, earlier iterations consume `msg.value` (or pre-existing router ETH) and later iterations fail `.send()`, triggering `payable(address(msg.sender)).transfer(transferOutPayload.amount)` which reverts the entire batch when the router balance is exhausted.
3. **Zero-Cost Event Spoofing via Bounce-Back:** Because `transferOutV5` and `batchTransferOutV5` are public, any caller can supply `msg.value = X` and a `TransferOutData` payload with `asset = address(0)`, `amount = X`, and `to` set to a contract that rejects ETH (causing `.send()` to return `false`). Line 213 immediately refunds `X` ETH back to `msg.sender`, while line 231 still emits `TransferOut(msg.sender, to, address(0), X, memo)` (and in `batchTransferOutV5`, the same `X` ETH is not even re-deposited, or with `amount = 0` can be repeated arbitrarily inside a single batch).

### Assumptions and preconditions

- `THORChain_Router` is deployed and used by vaults to execute single or batched outbound transfers via `transferOutV5` and `batchTransferOutV5`.
- Either a caller provides `msg.value` that does not match the sum of native asset `transferOutPayload.amount` values (leaving ETH in `THORChain_Router`), ETH is sent to the router via another mechanism, or an external actor invokes `transferOutV5` / `batchTransferOutV5` directly.

### Proof of concept

1. A vault (or user) calls `transferOutV5` or `batchTransferOutV5` where `msg.value` exceeds the sum of `transferOutPayload.amount` for `asset == address(0)` by `1 ETH` (e.g., passing `msg.value = 1 ether` on an ERC20 outbound or an under-summed batch).
2. `_transferOutV5` processes the payload using `transferOutPayload.amount` and leaves `1 ETH` in `address(router).balance`.
3. An unprivileged user calls `transferOutV5(TransferOutData({to: payable(attacker), asset: address(0), amount: 1 ether, memo: ""}))` with `msg.value = 0`.
4. In `_transferOutV5`, `transferOutPayload.asset == address(0)` is true, and `transferOutPayload.to.send(1 ether)` transfers the `1 ETH` from `THORChain_Router` to `attacker`.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Discrepancy between actual `msg.value` supplied and `transferOutPayload.amount` transferred/logged; permanent exposure and permissionless theft of any ETH retained by `THORChain_Router`; potential batch outbound reverts or forged `TransferOut` events.
- **Business Impact:** Direct loss of vault/protocol ETH if any `transferOutV5` or `batchTransferOutV5` call over-supplies `msg.value`, and desynchronization between on-chain vault ETH balances and Bifrost's outbound event accounting.

### Remediation

Track the total native ETH consumed across `_transferOutV5` executions and require that the total ETH amount transferred out in `transferOutV5` and `batchTransferOutV5` exactly equals `msg.value` (which also ensures `msg.value == 0` when only ERC20 assets are transferred).

```solidity
  function _transferOutV5(
    TransferOutData memory transferOutPayload
  ) private returns (uint256 ethAmount) {
    if (transferOutPayload.asset == address(0)) {
      ethAmount = transferOutPayload.amount;
      bool success = transferOutPayload.to.send(transferOutPayload.amount); // Send ETH.
      if (!success) {
        payable(address(msg.sender)).transfer(transferOutPayload.amount); // For failure, bounce back to vault & continue.
      }
    } else {
      _vaultAllowance[msg.sender][
        transferOutPayload.asset
      ] -= transferOutPayload.amount; // Reduce allowance

      (bool success, bytes memory data) = transferOutPayload.asset.call(
        abi.encodeWithSignature(
          "transfer(address,uint256)",
          transferOutPayload.to,
          transferOutPayload.amount
        )
      );

      require(success && (data.length == 0 || abi.decode(data, (bool))));
    }

    emit TransferOut(
      msg.sender,
      transferOutPayload.to,
      transferOutPayload.asset,
      transferOutPayload.amount,
      transferOutPayload.memo
    );
  }

  function transferOutV5(
    TransferOutData calldata transferOutPayload
  ) public payable nonReentrant {
    uint256 ethTotal = _transferOutV5(transferOutPayload);
    require(ethTotal == msg.value, "THORChain_Router:ward eth amount mismatch");
  }

  // bifrost to budget gas limits, no more than 50% than L1 gas limit
  function batchTransferOutV5(
    TransferOutData[] calldata transferOutPayload
  ) external payable nonReentrant {
    uint256 ethTotal = 0;
    for (uint i = 0; i < transferOutPayload.length; ++i) {
      ethTotal += _transferOutV5(transferOutPayload[i]);
    }
    require(ethTotal == msg.value, "THORChain_Router: eth amount mismatch");
  }
```

## Finding 10: Silent native-transfer failure in THORChain_Router._transferOutV5 finalizes unpaid outbounds

### Synopsis

`THORChain_Router._transferOutV5` suppresses failed native-asset payouts by refunding the vault and continuing normally. A recipient contract can reject the gas-stipended transfer, receive no ETH, while the router still completes and records the outbound as successfully paid.

### Technical details

The native-asset branch transfers ETH using Solidity’s `send`, which forwards only the 2,300-gas stipend and returns `false` rather than reverting when the recipient rejects ETH or requires more gas.

When this failure occurs, lines 212–214 refund the nominal payout to `msg.sender` and allow execution to continue. The function subsequently emits the ordinary `TransferOut` event for the original recipient and amount. No return value, revert, or failure event distinguishes this path from a successful payment.

Consequently, on-chain value movement and the router’s settlement record diverge:

- The recipient receives no ETH.
- The calling vault receives the ETH back.
- The transaction succeeds.
- A normal `TransferOut` event identifies the unpaid recipient and full amount.
- Off-chain processing can finalize the outbound and omit any retry.

This occurs even when the call is funded with exactly the requested amount; it does not depend on any discrepancy between `msg.value` and `transferOutPayload.amount`. Batch execution further makes the silent-failure behavior difficult to detect because processing continues after the failed item.

### Assumptions and preconditions

- A vault submits a correctly funded native-asset outbound through `transferOutV5` or `batchTransferOutV5`.
- The recipient is a contract that rejects ETH or cannot execute its receive logic within the `send` stipend.
- The calling vault can accept the 2,300-gas refund, as an EOA normally can.
- THORChain’s off-chain processing treats the successful transaction and ordinary `TransferOut` event as proof that the outbound was completed.
- No separate reconciliation mechanism automatically retries the unpaid recipient.

### Proof of concept

1. Deploy a recipient contract whose receive function always reverts:

   ```solidity
   contract RejectETH {
       receive() external payable {
           revert("reject ETH");
       }
   }
   ```

2. Record the recipient’s, vault’s, and router’s ETH balances.
3. From a vault address, call `transferOutV5` with:
   - `to` set to `RejectETH`;
   - `asset` set to `address(0)`;
   - `amount` set to 1 ETH;
   - `msg.value` set to exactly 1 ETH.
4. The `send` at line 211 fails because the recipient reverts.
5. Lines 212–214 transfer the 1 ETH back to the vault.
6. Observe that the outer transaction nevertheless succeeds.
7. Observe that the recipient’s balance is unchanged and the vault recovered the nominal payout.
8. Observe that the router emits the ordinary `TransferOut` event naming `RejectETH` as recipient and 1 ETH as the transferred amount.
9. Verify that an event-driven outbound processor consequently sees an apparently successful payment despite no ETH reaching the recipient.

### Impact assessment

**Severity: Medium.**

A legitimate user can permanently miss an outbound payment while the protocol records the obligation as fulfilled. Contract recipients are especially exposed because ordinary receive logic can exceed the fixed stipend even without malicious behavior.

A malicious destination can intentionally force the discrepancy by reverting. Although the ETH is returned to the vault rather than stolen directly, the settlement record becomes inaccurate and automated retry behavior can be suppressed. At scale, this can produce unresolved customer liabilities, reconciliation failures, and loss of confidence in outbound settlement correctness.

### Remediation

The preferred solution is to treat native transfer failure as transaction failure. Use a full-gas call and require success so that the normal `TransferOut` event is reachable only after actual delivery:

```solidity
if (transferOutPayload.asset == address(0)) {
    (bool success, ) = transferOutPayload.to.call{
        value: transferOutPayload.amount
    }("");
    require(success, "THORChain_Router: native transfer failed");
}
```

Retain `nonReentrant` protection on every external wrapper because forwarding gas transfers control to the recipient.

If batch liveness requires failed items not to revert the entire batch, introduce explicit failure handling instead: emit a distinct failure event, queue the amount for pull-based withdrawal or retry, and never emit the ordinary `TransferOut` success event for an unpaid recipient.

## Finding 11: Fee-on-transfer ERC20s make THORChain_Router._transferOutV5 record more than recipients receive

### Synopsis

`THORChain_Router._transferOutV5` treats a fee-on-transfer ERC20’s requested amount as the delivered payout without checking recipient balance changes. A taxed transfer can succeed while the recipient receives less, yet the full allowance is consumed and a full-amount settlement event is emitted.

### Technical details

The ERC20 branch deducts `transferOutPayload.amount` from the caller’s vault allowance at lines 216–218, then invokes the token at line 220. Successful execution and an acceptable return value only prove that the token call did not report failure; they do not prove that the recipient’s balance increased by the requested amount.

For a fee-on-transfer token, an outbound request for `A` tokens may produce the following successful result:

- The Router’s vault allowance decreases by `A`.
- The recipient receives `A - fee`.
- The token transfers or burns `fee`.
- The Router emits `TransferOut` using the original amount `A`.

The transaction therefore finalizes despite the payment being smaller than the amount represented by Router accounting and the settlement event. Measuring fee deductions during deposits does not prevent this: even if a deposit credits only the Router’s net receipt, another transfer fee can be charged when those credited tokens are paid out.

Because THORChain relies on Router events to describe asset movements, downstream settlement can treat the full amount as paid without retrying or refunding the shortfall. Tokens that additionally debit fees on top of the requested amount can also reduce the Router’s pooled balance by more than the allowance deduction, leaving other vault allowances undercollateralized.

### Assumptions and preconditions

- The protocol supports or whitelists an ERC20 with non-1:1 transfer semantics.
- The token’s `transfer` call reports success while charging a fee.
- A vault has a positive allowance for the affected token.
- The outbound is submitted through `transferOutV5` or `batchTransferOutV5`.
- Settlement processing does not independently reconcile the recipient’s actual token balance increase before marking the outbound complete.

### Proof of concept

1. Use a fee-on-transfer ERC20 that deducts a 10% fee and returns `true`.
2. Approve the Router and deposit 100 tokens. If the deposit is also taxed, the Router receives and credits a 90-token vault allowance.
3. Record the intended recipient’s token balance.
4. From the credited vault, call `transferOutV5` with:
   - `asset` set to the taxed token;
   - `amount` set to 90;
   - `to` set to the intended recipient.
5. Observe that the transaction succeeds and the vault allowance decreases by 90.
6. Observe that the recipient’s balance increases by only 81 tokens, while 9 tokens are charged as a fee.
7. Observe that the Router’s `TransferOut` event nevertheless records an amount of 90.
8. Confirm that downstream processing has no on-chain indication that the recipient is still short by 9 tokens.

### Impact assessment

**Severity: Medium.**

Affected outbounds can be irreversibly underpaid while appearing fully settled. Users may receive less than the protocol-recorded amount, with no automatic retry or refund because the token call succeeded. Repeated occurrences can cause customer losses, accounting disputes, and divergence between on-chain balances and THORChain’s settlement records.

The attack is conditional on an accepted token having fee-bearing transfer semantics. In the worst case, a token with a very high fee can report a full payout while delivering little or nothing. Sender-surcharge variants can additionally consume pooled token balances backing other vaults’ allowances.

### Remediation

Require exact balance deltas for both the Router and recipient before accepting an ERC20 payout. Revert for tokens whose outbound transfer semantics are not one-to-one. Use a safe ERC20 call implementation for return-data and contract-code validation, but note that `SafeERC20` alone does not detect transfer fees.

```solidity
} else {
  address asset = transferOutPayload.asset;
  address to = transferOutPayload.to;
  uint256 amount = transferOutPayload.amount;

  require(asset.code.length > 0, "asset has no code");
  require(to != address(this), "invalid recipient");

  uint256 routerBalanceBefore = iERC20(asset).balanceOf(address(this));
  uint256 recipientBalanceBefore = iERC20(asset).balanceOf(to);

  _vaultAllowance[msg.sender][asset] -= amount;

  (bool success, bytes memory data) = asset.call(
    abi.encodeWithSignature("transfer(address,uint256)", to, amount)
  );

  require(
    success &&
      (data.length == 0 ||
        (data.length >= 32 && abi.decode(data, (bool)))),
    "token transfer failed"
  );

  uint256 routerBalanceAfter = iERC20(asset).balanceOf(address(this));
  uint256 recipientBalanceAfter = iERC20(asset).balanceOf(to);

  require(
    routerBalanceBefore >= routerBalanceAfter &&
      routerBalanceBefore - routerBalanceAfter == amount &&
      recipientBalanceAfter >= recipientBalanceBefore &&
      recipientBalanceAfter - recipientBalanceBefore == amount,
    "unsupported token transfer semantics"
  );
}
```

If fee-on-transfer assets must remain supported, measure the actual recipient credit and propagate that value into settlement processing rather than finalizing the requested amount. Any gross-up mechanism must also ensure the Router never spends more than the caller’s authorized allowance.

## Finding 12: Unsafe Token Approval Lifecycle in `THORChain_Router._routerDeposit` Allows Residual Spender Authorization and Breaks Nonstandard ERC20 Migrations

### Synopsis

`THORChain_Router._routerDeposit` grants ERC20 approvals from the router's shared token custody via `safeApprove(_asset, _router, _amount)` without resetting prior allowances to zero, revoking unspent allowances after `depositWithExpiry`, or checking the `approve` boolean return data, enabling residual spender authorizations and causing non-zero-to-non-zero approval reverts (e.g., with USDT).

### Technical details

In `chain/ethereum/contracts/THORChain_Router.sol`, all ERC20 tokens deposited into the protocol are held in a single shared custody contract (`address(this)`), while individual vault balances are tracked internally in the `_vaultAllowance` mapping. When migrating allowances or returning vault assets to another router (`_router != address(this)`), `transferAllowance` (line 176) and `returnVaultAssets` (line 421) invoke `_routerDeposit` (lines 466–483):

```solidity
  function _routerDeposit(
    address _router,
    address _vault,
    address _asset,
    uint _amount,
    string memory _memo
  ) internal {
    _vaultAllowance[msg.sender][_asset] -= _amount;
    safeApprove(_asset, _router, _amount);

    iROUTER(_router).depositWithExpiry(
      _vault,
      _asset,
      _amount,
      _memo,
      type(uint).max
    ); // Transfer by depositing
  }
```

This implementation has a flawed token approval lifecycle centered on line 474 (`safeApprove(_asset, _router, _amount)`):

1. **Missing Pre-Approval Zero Reset (Nonstandard Allowance-Transition Incompatibility):** `_routerDeposit` directly calls `safeApprove(_asset, _router, _amount)` without first resetting the allowance to `0`. Certain widely used ERC20 tokens supported by THORChain (such as `USDT` / `TetherToken`) revert if `approve(spender, amount)` is called with a non-zero `amount` while `allowance(address(this), spender)` is already non-zero.
2. **Missing Post-Deposit Allowance Revocation and Transfer Verification (Residual Spender Authorization & Shared-Custody Authorization Leakage):** `transferAllowance` and `returnVaultAssets` are permissionless external functions callable by any address that holds a `_vaultAllowance` (which any user can obtain by calling `depositWithExpiry` with their own address as `vault`) and accept an arbitrary `_router` address. `_routerDeposit` neither verifies that `iROUTER(_router).depositWithExpiry(...)` actually pulled `_amount` tokens from `THORChain_Router` nor resets the allowance to `0` after `depositWithExpiry` returns. If `_router` does not consume the entire `_amount` allowance during `depositWithExpiry`, `_router` retains a lingering non-zero ERC20 allowance directly against `THORChain_Router`'s shared token custody.
3. **Unchecked Boolean Return Value in `safeApprove` (Approval Outcome Misinterpretation):** Unlike `safeTransferFrom` (line 451) and `transferOut` (line 203), `safeApprove` (lines 485–494) only checks `require(success)` on the low-level `.call` and ignores the returned `bytes memory data`, failing to detect ERC20 tokens that return `false` rather than reverting on an unsuccessful approval.

### Assumptions and preconditions

- `THORChain_Router` holds ERC20 tokens for multiple vaults in shared custody at `address(this)`.
- A caller (either a vault migrating assets to a target `_router` or an external user who acquired `_vaultAllowance` via `depositWithExpiry`) invokes `transferAllowance` or `returnVaultAssets` with `_router != address(this)`.
- Either `_router` does not consume the full `_amount` approved during `depositWithExpiry` (leaving a residual allowance on `THORChain_Router`), or `_asset` enforces non-zero-to-non-zero approval restrictions (such as `USDT`) or returns `false` on failed approvals.

### Proof of concept

1. Assume `THORChain_Router` holds `1,000,000 USDT` allocated to an Asgard vault.
2. A user deposits `100 USDT` into `THORChain_Router` via `depositWithExpiry(userAddress, address(USDT), 100, "MEMO", expiration)`, crediting `_vaultAllowance[userAddress][USDT] = 100`.
3. The user deploys a contract `CustomRouter` implementing `depositWithExpiry(address,address,uint256,string,uint256)` that pulls only `40 USDT` (or `0 USDT`) instead of the full `_amount`.
4. The user calls `THORChain_Router.transferAllowance(address(CustomRouter), userAddress, address(USDT), 100, "MEMO")`:
   - `_vaultAllowance[userAddress][USDT]` is decremented by `100` to `0`.
   - `safeApprove(address(USDT), address(CustomRouter), 100)` sets `USDT.allowance(address(THORChain_Router), address(CustomRouter)) = 100`.
   - `CustomRouter.depositWithExpiry(...)` pulls `40 USDT`, leaving a residual allowance of `60 USDT` from `THORChain_Router` to `CustomRouter` over the router's shared token pool even after the transaction finishes.
5. If a subsequent `_routerDeposit` call targets the same `_router` with `USDT` while the residual `60 USDT` allowance remains unspent, `safeApprove(address(USDT), _router, newAmount)` at line 474 attempts a non-zero-to-non-zero approval (`60 -> newAmount`), which reverts inside `USDT.approve`.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Lingering ERC20 allowances remain active against `THORChain_Router`'s shared token custody whenever `_router.depositWithExpiry` does not consume the exact approved `_amount`. Any residual non-zero allowance on `_router` causes subsequent `_routerDeposit` calls for tokens like `USDT` to revert due to non-zero-to-non-zero approval restrictions, and leaves unspent spender authorization active on the shared router contract.

### Remediation

Reset the allowance to `0` prior to setting the new approval amount, verify that the expected token amount was actually transferred out during `depositWithExpiry` (or revoke any remaining allowance by resetting it to `0` immediately after `depositWithExpiry`), and validate the optional boolean return data in `safeApprove`:

```solidity
  function _routerDeposit(
    address _router,
    address _vault,
    address _asset,
    uint _amount,
    string memory _memo
  ) internal {
    _vaultAllowance[msg.sender][_asset] -= _amount;
    safeApprove(_asset, _router, 0);
    safeApprove(_asset, _router, _amount);

    uint _startBal = iERC20(_asset).balanceOf(address(this));
    iROUTER(_router).depositWithExpiry(
      _vault,
      _asset,
      _amount,
      _memo,
      type(uint).max
    ); // Transfer by depositing
    safeApprove(_asset, _router, 0);
    require(
      _startBal - iERC20(_asset).balanceOf(address(this)) == _amount,
      "THORChain_Router: incomplete router deposit"
    );
  }
```

## Finding 13: Unverified Cross-Router Token Settlement and Allowance Handling in `THORChain_Router._routerDeposit`

### Synopsis

`THORChain_Router._routerDeposit` debits the caller's vault allowance, approves an arbitrary `_router` address, and invokes `depositWithExpiry` without verifying that the expected ERC-20 tokens were actually transferred or that the approval was fully consumed, enabling source-debit and destination-credit divergence and lingering token approvals.

### Technical details

In `chain/ethereum/contracts/THORChain_Router.sol`, the internal function `_routerDeposit` (lines 466–483), reachable via `transferAllowance` and `returnVaultAssets` whenever `router != address(this)`, migrates a vault's asset allowance from the current router to another router contract:

```solidity
  function _routerDeposit(
    address _router,
    address _vault,
    address _asset,
    uint _amount,
    string memory _memo
  ) internal {
    _vaultAllowance[msg.sender][_asset] -= _amount;
    safeApprove(_asset, _router, _amount);

    iROUTER(_router).depositWithExpiry(
      _vault,
      _asset,
      _amount,
      _memo,
      type(uint).max
    ); // Transfer by depositing
  }
```

This implementation has several closely related settlement integrity flaws:

1. **Unverified Asset-Transfer Completion:** `_routerDeposit` assumes that `iROUTER(_router).depositWithExpiry(...)` pulls the exact `_amount` of `_asset` from `address(this)`. It neither checks the router's `_asset` balance before and after the external call nor verifies that the ERC-20 allowance granted to `_router` was consumed.
2. **Untrusted Destination Behavior and Residual Approvals:** Because `_router` is a caller-supplied address and is not validated against a trusted router address, a non-standard, misconfigured, or malicious `_router` implementation can return successfully from `depositWithExpiry` without calling `transferFrom` (or by pulling less than `_amount`). The source router permanently debits `_vaultAllowance[msg.sender][_asset] -= _amount`, while `_router` retains an active ERC-20 allowance on `_asset` held by the source router.
3. **Source-Debit and Destination-Credit Divergence:** While `THORChain_Router._deposit` uses `safeTransferFrom` to credit `_vaultAllowance[_vault][_asset]` based on the actual balance delta received (`iERC20(_asset).balanceOf(address(this)) - _startBal` to account for transfer fees or non-standard token semantics), `_routerDeposit` unconditionally debits the nominal `_amount` on the source router without reconciling the actual amount transferred or credited on the destination router, and without rejecting native asset (`_asset == address(0)`) calls where no `msg.value` is forwarded to `depositWithExpiry`.

### Assumptions and preconditions

- A caller invokes `transferAllowance` or `returnVaultAssets` with a `router` address different from `address(this)`.
- The target `_router` contract either fails to pull the full `_amount` during `depositWithExpiry` (e.g., due to a faulty/untrusted target contract or `address(0)` asset parameter) or the token exhibits transfer fees / non-standard allowance semantics.

### Proof of concept

1. Suppose Vault A holds an allowance of `1,000` tokens for `_asset` in `Router1` (`_vaultAllowance[VaultA][_asset] == 1000`).
2. `transferAllowance(_router, _vault, _asset, 1000, _memo)` (or `returnVaultAssets`) is called with `_router != address(Router1)`.
3. `_routerDeposit` executes:
   - `_vaultAllowance[VaultA][_asset]` is reduced by `1,000` (down to `0`).
   - `safeApprove(_asset, _router, 1000)` sets `ERC20(_asset).allowance(address(Router1), _router) = 1000`.
   - `iROUTER(_router).depositWithExpiry(_vault, _asset, 1000, _memo, type(uint).max)` is called.
4. If `_router` is a contract whose `depositWithExpiry` does not pull the `1,000` tokens (or pulls only a partial amount), the transaction still succeeds:
   - `Router1` has already debited `1,000` from `_vaultAllowance[VaultA][_asset]`.
   - `_vault` on the destination router is not credited with the `1,000` tokens.
   - `_router` retains the unspent ERC-20 approval from `Router1` for the remaining tokens still sitting in `Router1`.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Source vault allowances can be permanently debited without a matching token transfer and destination vault credit, leaving tokens stranded or exposed to lingering ERC-20 approvals on the source router and causing accounting desynchronization between THORChain vaults/Bifrost and on-chain router balances.

### Remediation

Validate that `_asset != address(0)`, reset any prior approval before setting the new approval, verify that the router's token balance decreased by the expected `_amount` across `depositWithExpiry`, and clear any remaining approval afterward.

```solidity
  function _routerDeposit(
    address _router,
    address _vault,
    address _asset,
    uint _amount,
    string memory _memo
  ) internal {
    require(_asset != address(0), "THORChain_Router: invalid asset");
    _vaultAllowance[msg.sender][_asset] -= _amount;
    safeApprove(_asset, _router, 0);
    safeApprove(_asset, _router, _amount);

    uint _startBal = iERC20(_asset).balanceOf(address(this));
    iROUTER(_router).depositWithExpiry(
      _vault,
      _asset,
      _amount,
      _memo,
      type(uint).max
    ); // Transfer by depositing
    require(
      _startBal - iERC20(_asset).balanceOf(address(this)) == _amount,
      "THORChain_Router: incomplete router deposit"
    );
  }
```

## Finding 14: Nominal source debit in THORChain_Router._routerDeposit lets sender-fee tokens consume other vaults’ pooled backing

### Synopsis

`THORChain_Router._routerDeposit` records only the nominal `_amount` against the source vault without checking the Router’s actual token-balance decrease. A sender-surcharge ERC-20 can therefore charge pooled reserves while successfully moving the full requested amount, progressively leaving other vault allowances unbacked.

### Technical details

The Router holds each asset in a shared token balance, while `_vaultAllowance` partitions that balance among vaults. Consequently, every physical token decrease must be charged to the allowance of the vault initiating it.

At line 473, `_routerDeposit` reduces the caller’s allowance by exactly `_amount`. It then approves `_amount` and asks the destination Router to pull that amount. There is no measurement of the source Router’s token balance before and after the pull.

Some fee-bearing ERC-20 implementations transfer the requested amount to the recipient while charging an additional sender-side fee. For example, `transferFrom(source, destination, 100)` may:

- consume an approval of 100;
- credit 100 to the destination;
- debit 110 from the source; and
- return `true`.

With this behavior, the destination Router legitimately receives and credits the full 100, but the source Router loses 110 while only 100 is removed from the initiating vault’s allowance. The additional 10 is therefore taken from the pooled balance backing other vaults without reducing those vaults’ recorded allowances.

This mismatch can be repeated through cross-router movements by an unprivileged address that has deposited tokens to itself as a vault. Each successful movement can preserve the attacker’s nominal allowance on the destination while creating a deficit in the source Router’s pooled reserves.

### Assumptions and preconditions

- A supported token exhibits, or can later enable, sender-side transfer fees or equivalent non-conservative transfer behavior.
- The token consumes an allowance of `_amount` but decreases the source Router’s balance by more than `_amount`.
- The source Router contains the same token backing allowances belonging to other vaults.
- The attacker can deposit the token to an attacker-controlled vault address and obtain a corresponding allowance.
- At least one compatible destination Router is available for cross-router transfers.
- No protocol owner or administrator needs to act maliciously; the condition may result from externally controlled or upgradeable token behavior.

### Proof of concept

1. Let Router A hold 1,100 tokens:
   - 100 tokens back the attacker’s vault allowance.
   - 1,000 tokens back an honest vault’s allowance.
2. Configure or use a supported token that charges a 10% sender-side fee when a Router is the token owner. A transfer request for 100 therefore debits 110 from the source but credits 100 to the recipient.
3. The attacker calls the cross-router allowance-transfer path, moving 100 tokens from Router A to an attacker-controlled vault on Router B.
4. `_routerDeposit` reduces the attacker’s Router A allowance from 100 to zero.
5. Router A approves Router B for 100, and Router B successfully pulls 100:
   - the approval is consumed by 100;
   - Router B receives and credits 100;
   - Router A’s physical balance decreases by 110.
6. Router A is left with a balance of 990 but still records an honest-vault allowance of 1,000.
7. The attacker retains a fully backed 100-token allowance on Router B, while 10 tokens of the honest vault’s backing have been consumed.
8. Repeating the process with additional attacker allowances or between funded Routers progressively increases the accounting deficit until honest outbounds fail.

### Impact assessment

This is a **Medium-severity** accounting-integrity vulnerability. Exploitation requires a supported token with sender-surcharge or equivalent non-conservative transfer behavior, but does not require control of a protocol vault or malicious protocol administration.

Successful exploitation can:

- consume assets backing unrelated vault allowances;
- make recorded allowances exceed the Router’s actual token balance;
- cause later honest outbounds or migrations to revert;
- trigger solvency failures or chain halts;
- require protocol reimbursement or manual accounting repair.

In the worst case, repeated cross-router movements can deplete all pooled reserves of the affected token while the corresponding honest-vault allowances remain recorded.

### Remediation

Enforce the documented no-fee transfer invariant by measuring the source Router’s token balance and requiring that the actual decrease equals `_amount`. Because a failed check reverts the entire transaction, sender-side surcharges cannot be paid from pooled backing.

```solidity
function _routerDeposit(
  address _router,
  address _vault,
  address _asset,
  uint _amount,
  string memory _memo
) internal {
  require(_asset != address(0), "native asset is not supported");

  uint256 sourceBalanceBefore = iERC20(_asset).balanceOf(address(this));

  _vaultAllowance[msg.sender][_asset] -= _amount;
  safeApprove(_asset, _router, _amount);

  iROUTER(_router).depositWithExpiry(
    _vault,
    _asset,
    _amount,
    _memo,
    type(uint).max
  );

  uint256 sourceBalanceAfter = iERC20(_asset).balanceOf(address(this));
  require(
    sourceBalanceBefore >= sourceBalanceAfter &&
      sourceBalanceBefore - sourceBalanceAfter == _amount,
    "unsupported token debit semantics"
  );
}
```

If sender-fee tokens must be supported, an alternative is to debit the initiating vault by the measured source-balance decrease and revert unless that vault’s allowance covers the complete decrease, including the fee.
