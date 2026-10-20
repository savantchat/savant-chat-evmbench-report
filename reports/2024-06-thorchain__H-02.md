# Audit report — 2024-06-thorchain/H-02

## Finding 1: Premature Loop Termination and Missing Vault Validation in `SmartContractLogParser.GetTxInItem` Allow Spoofed `TransferOut` Logs to Hijack or Suppress Outbound Observations

### Synopsis

In `SmartContractLogParser.GetTxInItem`, the `transferOutEvent` handler neither verifies that `transferOutEvt.Vault` matches `txInItem.Sender` nor continues past invalid `TransferOut` logs, and immediately terminates log scanning via `earlyExit = true`. An attacker-controlled contract emitting a preceding `TransferOut` log can suppress or hijack vault outbound observations.

### Technical details

When Bifrost scans smart contract transactions in `ETHScanner.getTxInFromSmartContract` (`bifrost/pkg/chainclients/ethereum/ethereum_block_scanner.go:777`) and `EVMScanner.getTxInFromSmartContract` (`bifrost/pkg/chainclients/evm/evm_block_scanner.go:726`), it initializes `txInItem.Sender` to the transaction signer (`e.eipSigner.Sender(tx)`) and passes `receipt.Logs` and `txInItem` to `SmartContractLogParser.GetTxInItem` (`bifrost/pkg/chainclients/shared/evm/smartcontract_log_parser.go:166`).

In the THORChain Router contract, outbound execution functions (`transferOut` and `transferOutAndCall`) perform external calls to the token contract, recipient address, or aggregator (`asset.call`, `to.send`, `target.call`) **before** emitting the Vault's `TransferOut` or `TransferOutAndCall` event at the end of the function. Consequently, any Router event emitted during the external call (for example, via re-entry on a Router without `nonReentrant` on `transferOut`, or via a secondary/retired Router contract still present in `pubkeyMgr.GetContracts` that passes `scp.addressValidator`) is recorded at an earlier index in `receipt.Logs` than the Vault's legitimate `TransferOut` / `TransferOutAndCall` event. Similarly, a user-initiated transaction calling a contract can emit multiple Router events in a chosen sequence.

In `GetTxInItem`, the `transferOutEvent` branch at lines 213–243 contains three interconnected flaws in lines 227–242:

1. **Missing Vault Sender Validation and Premature Loop Exit (`earlyExit = true` at line 242):**
   Unlike `transferAllowanceEvent` (which verifies `strings.EqualFold(txInItem.Sender, transferAllowanceEvt.OldVault.String())` at line 256 and checks for a non-zero amount at line 252), `transferOutEvent` never checks whether `transferOutEvt.Vault` matches `txInItem.Sender`. When an earlier `TransferOut` log in `receipt.Logs` has a valid `OUT:` or `MIGRATE:` memo and a resolvable asset, lines 236–241 overwrite `txInItem.To`, `txInItem.Memo`, and `txInItem.Coins`, and line 242 sets `earlyExit = true`, breaking out of the `for _, item := range logs` loop at line 338 before the Vault's actual `TransferOut` (or `TransferOutAndCall`) event at a later log index is processed.

2. **Premature Abort and Partial-State Acceptance on Asset Resolution Failure (lines 229–235):**
   Unlike `depositEvent` (lines 199–206) and `transferAllowanceEvent` (lines 277–284), which `continue` to the next log when `scp.assetResolver` returns an error or an empty asset, `transferOutEvent` immediately executes `return false, fmt.Errorf(...)` on error (line 231) or `return false, nil` when `asset.IsEmpty()` (line 234).
   - In a Vault outbound transaction, an attacker-emitted `TransferOut` log with an unresolvable or empty asset causes `GetTxInItem` to return immediately at line 231 or 234, discarding the Vault's legitimate `TransferOut` log that follows.
   - In a multi-event transaction where a preceding log (`depositEvent`, `transferAllowanceEvent`, or `transferOutAndCallEvent`) already mutated `txInItem`, hitting `if asset.IsEmpty() { return false, nil }` at line 234 returns a `nil` error without clearing `txInItem`. Because the caller (`getTxInFromSmartContract`) only abandons `txInItem` when `err != nil` and otherwise checks `!txInItem.Coins.IsEmpty()`, the partially populated `txInItem` is accepted while all subsequent logs are ignored.

3. **Cross-Event Field Contamination (lines 236–241):**
   When `transferOutEvent` processes a log after a `transferOutAndCallEvent` in the same transaction, lines 236–241 overwrite only `txInItem.To`, `txInItem.Memo`, and `txInItem.Coins`, leaving `txInItem.Sender` (overwritten at line 328), `txInItem.Aggregator`, `txInItem.AggregatorTarget`, and `txInItem.AggregatorTargetLimit` (lines 329–335) populated from the earlier `transferOutAndCallEvent`.

### Assumptions and preconditions

1. An outbound transaction is executed by a THORChain Vault via `transferOut` (for an ERC20 token with transfer hooks/callbacks or recipient fallback) or `transferOutAndCall` (where `target` invokes `swapOut` with user-supplied `finalToken` and `to` parameters), or a multi-event transaction is submitted to the Router.
2. During the external call before the Vault's `TransferOut` / `TransferOutAndCall` log is emitted, the called contract invokes `transferOut` on a Router contract accepted by `scp.addressValidator` (either the same Router if `transferOut` is not protected by `nonReentrant`, or another active/retired Router address in `pubkeyMgr.GetContracts`).

### Proof of concept

1. A user initiates a swap on THORChain whose outbound executes `transferOut` (or `transferOutAndCall`) to an attacker-controlled contract or token hook.
2. The THORChain Vault signs and broadcasts the outbound transaction. In `getTxInFromSmartContract`, `txInItem.Sender` is set to the Vault's EOA address.
3. During the external call inside `transferOut` / `transferOutAndCall` (prior to the Router emitting the Vault's `TransferOut` / `TransferOutAndCall` event), the attacker's contract calls `transferOut` on a valid Router contract:
   - **Variant A (Outbound Suppression):** The attacker passes an unresolvable or empty token address and memo `"OUT:<hash>"`. `receipt.Logs[0]` is the attacker's `TransferOut` log; `receipt.Logs[1]` is the Vault's legitimate `TransferOut` log. When `GetTxInItem` processes `receipt.Logs[0]`, lines 230–235 return early (`false, err` or `false, nil`). `receipt.Logs[1]` is never processed, and Bifrost fails to observe the Vault's outbound transaction.
   - **Variant B (Outbound Hijacking & Vault Slashing):** The attacker deposits/approves a large amount of an ERC20 token (e.g., via a flash loan) to their contract on the Router and calls `transferOut(attackerAddr, tokenAddr, largeAmount, "OUT:<fake_txid>")`. `receipt.Logs[0]` is the attacker's `TransferOut` event, and `receipt.Logs[1]` is the Vault's real `TransferOut` event.
4. In Variant B, `GetTxInItem` processes `receipt.Logs[0]`, does not check `transferOutEvt.Vault == txInItem.Sender`, populates `txInItem` with `largeAmount` and `"OUT:<fake_txid>"` at lines 236–241, and sets `earlyExit = true` at line 242.
5. `GetTxInItem` breaks out of the loop at line 338 before reading `receipt.Logs[1]`.
6. Bifrost reports to THORNode that `txInItem.Sender` (the Vault) sent an unscheduled outbound of `largeAmount` with memo `"OUT:<fake_txid>"`, while the legitimate outbound memo in `receipt.Logs[1]` is never observed. THORNode slashes the Vault for the unauthorized outbound and reschedules the legitimate outbound payment.

### Impact assessment

- **Severity:** High / Critical
- **Technical Impact:** An attacker can suppress observation of legitimate Vault outbound transactions or substitute arbitrary `To`, `Memo`, and `Coins` values into an observation attributed to `txInItem.Sender` (the Vault). Additionally, mixed events in a single transaction can leave stale aggregator fields or partial state in `txInItem`.
- **Business Impact:** Unobserved Vault outbounds cause THORChain to reschedule and re-send outbound payouts (double-spending Vault funds), while spoofed `TransferOut` observations attributed to the Vault trigger unauthorized-outbound bond slashing against node operators.

### Remediation

Validate that `transferOutEvt.Vault` matches `txInItem.Sender` (and that `Amount` is non-zero), `continue` when a `TransferOut` log fails validation or asset resolution so legitimate subsequent logs are not skipped, and ensure `txInItem` state is not contaminated across incompatible event types.

```go
				if !m.IsOutbound() && !m.IsType(memo.TxMigrate) {
					scp.logger.Error().Str("memo", transferOutEvt.Memo).Msg("incorrect memo for transferOutEvent")
					continue
				}
				if len(transferOutEvt.Amount.Bits()) == 0 {
					scp.logger.Error().Msg("transfer out event with amount 0, ignore")
					continue
				}
				if len(txInItem.Sender) > 0 && !strings.EqualFold(txInItem.Sender, transferOutEvt.Vault.String()) {
					scp.logger.Error().Msg("transfer out event, vault address is not the same as sender, ignore")
					continue
				}
				asset, err := scp.assetResolver(transferOutEvt.Asset.String())
				if err != nil {
					scp.logger.Err(err).Str("address", transferOutEvt.Asset.String()).Msg("fail to get asset from token address")
					continue
				}
				if asset.IsEmpty() {
					continue
				}
				txInItem.To = transferOutEvt.To.String()
				txInItem.Memo = transferOutEvt.Memo
				txInItem.Aggregator = ""
				txInItem.AggregatorTarget = ""
				txInItem.AggregatorTargetLimit = nil
				decimals := scp.decimalResolver(transferOutEvt.Asset.String())
				txInItem.Coins = common.NewCoins(
					common.NewCoin(asset, scp.amtConverter(transferOutEvt.Asset.String(), transferOutEvt.Amount)).WithDecimals(decimals),
				)
				earlyExit = true
```

## Finding 2: Incomplete `TxInItem` state reset in `SmartContractLogParser.GetTxInItem` on `TransferOut` allows cross-event contamination from preceding `TransferOutAndCall` logs

### Synopsis

In `SmartContractLogParser.GetTxInItem`, the `transferOutEvent` handler overrides `To`, `Memo`, and `Coins` on `txInItem` when finalizing an outbound observation, but fails to reset `Sender`, `Aggregator`, `AggregatorTarget`, and `AggregatorTargetLimit`, allowing a preceding `TransferOutAndCall` log in the same transaction to contaminate the observed outbound transaction.

### Technical details

`SmartContractLogParser.GetTxInItem` in `bifrost/pkg/chainclients/shared/evm/smartcontract_log_parser.go` iterates through all logs emitted by the THORChain Router contract within a transaction and mutates a single shared `*types.TxInItem` struct in place.

As demonstrated in `TestFakeTransferOut`, when a vault executes an outbound transfer via the Router, a contract invoked during the transaction before the Router emits its final `TransferOut` event can call back into the Router to emit decoy events earlier in the transaction receipt's log list. The `transferOutEvent` branch (lines 213–243) is designed to supersede any state written to `txInItem` by earlier logs in the same transaction and immediately finalize the observation.

However, when a `transferOutAndCallEvent` log is processed earlier in the loop (lines 295–336), it mutates six fields on `txInItem`:
- `txInItem.Coins` (line 312)
- `txInItem.To` (line 326)
- `txInItem.Memo` (line 327)
- `txInItem.Sender = transferOutAndCall.Vault.String()` (line 328)
- `txInItem.Aggregator = aggregatorAddr.String()` (line 329)
- `txInItem.AggregatorTarget = aggregatorTargetAddr.String()` (line 330)
- `txInItem.AggregatorTargetLimit = &limit` (line 334)

When the subsequent `transferOutEvent` log is processed at lines 236–242:
```go
txInItem.To = transferOutEvt.To.String()
txInItem.Memo = transferOutEvt.Memo
decimals := scp.decimalResolver(transferOutEvt.Asset.String())
txInItem.Coins = common.NewCoins(
	common.NewCoin(asset, scp.amtConverter(transferOutEvt.Asset.String(), transferOutEvt.Amount)).WithDecimals(decimals),
)
earlyExit = true
```
it only overwrites `txInItem.To`, `txInItem.Memo`, and `txInItem.Coins` (and if `asset.IsEmpty()` at lines 233–235, it returns `false, nil` without clearing the fields already populated on `txInItem`). It does not clear `txInItem.Aggregator`, `txInItem.AggregatorTarget`, or `txInItem.AggregatorTargetLimit`, nor does it restore `txInItem.Sender` to `transferOutEvt.Vault.String()`.

Subsequently, in `bifrost/observer/observe.go` (lines 646–648), Bifrost copies `item.Aggregator`, `item.AggregatorTarget`, and `item.AggregatorTargetLimit` (along with the overwritten `item.Sender`) directly into the observed transaction sent to THORChain. Because THORChain matches observed outbound transactions against scheduled `TxOutItem` records using the vault sender and aggregator parameters, the contaminated fields cause the outbound observation to mismatch the scheduled vault outbound.

### Assumptions and preconditions

1. A transaction interacting with the THORChain Router emits a `TransferOutAndCall` event (or another Router event) prior to a `TransferOut` event within the same transaction receipt, and total logs do not exceed `maxLogs`.
2. The preceding `TransferOutAndCall` event contains a valid outbound memo (`OUT:<hash>`) and valid addresses so that lines 326–336 execute.

### Proof of concept

1. A vault executes an outbound transfer via the Router, or a contract interaction produces two Router logs in the same transaction receipt (within `maxLogs`):
   - Log 0: `TransferOutAndCall(vault=0xAttacker, target=0xAggregator, amount=1, finalAsset=0xToken, to=0xRecipient, amountOutMin=100, memo="OUT:<txid>")`
   - Log 1: `TransferOut(vault=0xAsgardVault, to=0xRecipient, asset=0xERC20, amount=100000000, memo="OUT:<txid>")`
2. `GetTxInItem` processes Log 0 (`transferOutAndCallEvent`), setting:
   - `txInItem.Sender = "0xAttacker"`
   - `txInItem.Aggregator = "0xAggregator"`
   - `txInItem.AggregatorTarget = "0xToken"`
   - `txInItem.AggregatorTargetLimit = 100`
   - `txInItem.Coins = [ETH: 1]`
3. `GetTxInItem` then processes Log 1 (`transferOutEvent` at lines 229–242).
   - Case A (normal asset): `txInItem.To`, `txInItem.Memo`, and `txInItem.Coins` are updated to the `TransferOut` values, but `txInItem.Sender` remains `"0xAttacker"` and `Aggregator`, `AggregatorTarget`, and `AggregatorTargetLimit` remain set to Log 0's values.
   - Case B (`asset.IsEmpty()` at line 233): `GetTxInItem` returns `false, nil`, and `getTxInFromSmartContract` accepts the `txInItem` populated by Log 0 because `err == nil` and `txInItem.Coins` is non-empty.
4. `observe.go` (lines 646–648) forwards the contaminated `TxInItem` to THORChain, where the altered `Sender` and non-empty `Aggregator` fields prevent proper matching of the vault's outbound transaction.

### Impact assessment

**Severity: High**

Contaminating a vault's `TransferOut` observation with a forged `Sender` (`transferOutAndCall.Vault`) and unexpected `Aggregator`, `AggregatorTarget`, and `AggregatorTargetLimit` fields prevents THORChain from matching the observed transaction to the scheduled `TxOutItem` (which has no aggregator fields and expects the vault's address as sender). As a result, THORChain treats the outbound transfer as unobserved/unfulfilled, slashing the vault and re-assigning the outbound transaction to be sent a second time (loss of vault funds via duplicate outbound payouts).

### Remediation

In `SmartContractLogParser.GetTxInItem`, ensure `transferOutEvent` explicitly resets aggregator fields, sets `txInItem.Sender` from the validated `transferOutEvt.Vault`, and does not leave prior partial state on `txInItem` when `asset.IsEmpty()`:

```go
				asset, err := scp.assetResolver(transferOutEvt.Asset.String())
				if err != nil {
					return false, fmt.Errorf("fail to get asset from token address: %w", err)
				}
				if asset.IsEmpty() {
					*txInItem = types.TxInItem{
						Tx:     txInItem.Tx,
						Sender: txInItem.Sender,
					}
					return false, nil
				}
				txInItem.Sender = strings.ToLower(transferOutEvt.Vault.String())
				txInItem.To = transferOutEvt.To.String()
				txInItem.Memo = transferOutEvt.Memo
				txInItem.Aggregator = ""
				txInItem.AggregatorTarget = ""
				txInItem.AggregatorTargetLimit = nil
				decimals := scp.decimalResolver(transferOutEvt.Asset.String())
				txInItem.Coins = common.NewCoins(
					common.NewCoin(asset, scp.amtConverter(transferOutEvt.Asset.String(), transferOutEvt.Amount)).WithDecimals(decimals),
				)
				earlyExit = true
```

## Finding 3: Restrictive Memo Type Check in `SmartContractLogParser.GetTxInItem` Causes Non-`TxOutbound` Aggregator Outbounds to Be Ignored

### Synopsis

In `SmartContractLogParser.GetTxInItem`, the `transferOutAndCallEvent` handler validates the event memo using `!m.IsType(memo.TxOutbound)` instead of `!m.IsOutbound()`. This causes legitimate outbound transactions carrying `TxRefund` or `TxRagnarok` memos to be ignored by Bifrost, preventing outbound confirmation on THORNode and risking repeated payouts.

### Technical details

When an EVM vault executes an outbound transaction through the THORChain Router contract, Bifrost's block scanner (`EVMScanner.getTxInFromSmartContract` and `ETHScanner.getTxInFromSmartContract`) parses the transaction receipt logs using `SmartContractLogParser.GetTxInItem` in `bifrost/pkg/chainclients/shared/evm/smartcontract_log_parser.go`.

In `x/thorchain/memo/memo.go` (lines 138–145), THORChain defines outbound memo types via `TxType.IsOutbound()`:

```go
func (tx TxType) IsOutbound() bool {
	switch tx {
	case TxOutbound, TxRefund, TxRagnarok:
		return true
	default:
		return false
	}
}
```

In `SmartContractLogParser.GetTxInItem`, standard router outbound events (`transferOutEvent`, lines 213–243) validate the memo using `m.IsOutbound()`:

```go
if !m.IsOutbound() && !m.IsType(memo.TxMigrate) {
	scp.logger.Error().Str("memo", transferOutEvt.Memo).Msg("incorrect memo for transferOutEvent")
	continue
}
```

However, in the `transferOutAndCallEvent` branch (lines 295–310), the parser checks `m.IsType(memo.TxOutbound)` instead of `m.IsOutbound()`:

```go
m, err := memo.ParseMemo(common.LatestVersion, transferOutAndCall.Memo)
if err != nil {
	scp.logger.Err(err).Msgf("fail to parse memo: %s", transferOutAndCall.Memo)
	continue
}
if !m.IsType(memo.TxOutbound) {
	scp.logger.Error().Msgf("%s is not an outbound memo", transferOutAndCall.Memo)
	continue
}
```

Although the log message on line 308 explicitly states `"%s is not an outbound memo"`, `m.IsType(memo.TxOutbound)` only matches `TxOutbound` (`OUT:<txid>`) and rejects all other outbound memo types classified by `m.IsOutbound()`, namely `TxRefund` (`REFUND:<txid>`) and `TxRagnarok` (`RAGNAROK:<block>`).

When a `TransferOutAndCall` event is emitted with a `REFUND:<txid>` or `RAGNAROK:<block>` memo:
1. `memo.ParseMemo` succeeds and returns a `Memo` with type `TxRefund` or `TxRagnarok`.
2. `!m.IsType(memo.TxOutbound)` evaluates to `true`, logging an error and skipping the log entry via `continue`.
3. `txInItem.Coins` remains empty, causing `getTxInFromSmartContract` (line 811 in `ethereum_block_scanner.go` and line 794 in `evm_block_scanner.go`) to return `nil, nil`.
4. Bifrost drops the observation and never submits an `ObservedTxOut` witness transaction to THORChain.

### Assumptions and preconditions

1. A THORChain vault executes a `transferOutAndCall` transaction on the EVM Router contract carrying an outbound memo of type `TxRefund` (`REFUND:<txid>`) or `TxRagnarok` (`RAGNAROK:<block>`).
2. The transaction succeeds on the EVM chain (`receipt.Status == 1`) and emits a `TransferOutAndCall` event from the valid router contract address.

### Proof of concept

1. A transaction on THORChain triggers a refund (or ragnarok) outbound routed through an EVM aggregator via `transferOutAndCall`, carrying memo `REFUND:<inbound_tx_hash>`.
2. The vault signs and broadcasts the `transferOutAndCall` transaction to the EVM Router contract, transferring native ETH and emitting:
   `TransferOutAndCall(vault, target, amount, finalAsset, to, amountOutMin, "REFUND:<inbound_tx_hash>")`.
3. Bifrost scans the block containing the transaction and invokes `getTxInFromSmartContract` -> `SmartContractLogParser.GetTxInItem`.
4. In `GetTxInItem` (line 302), `memo.ParseMemo(common.LatestVersion, transferOutAndCall.Memo)` parses `"REFUND:<inbound_tx_hash>"` into a `RefundMemo` where `GetType() == memo.TxRefund`.
5. At line 307, `m.IsType(memo.TxOutbound)` returns `false`. The parser logs `"REFUND:<inbound_tx_hash> is not an outbound memo"` and executes `continue` at line 309.
6. `GetTxInItem` finishes with an unpopulated `txInItem` (`txInItem.Coins.IsEmpty() == true`), causing `getTxInFromSmartContract` to return `nil, nil`.
7. Because Bifrost never observes the completed outbound transaction, THORNode considers the `TxOutItem` unfulfilled and reassigns/retries the outbound payout or slashes the vault for failing to send outbounds.

### Impact assessment

- **Severity:** Medium / High
- **Technical Effect:** Valid `TransferOutAndCall` outbound events with `TxRefund` or `TxRagnarok` memos are silently ignored by Bifrost observers.
- **Business Impact:** Unobserved outbound transactions remain pending in THORNode's outbound queue (`TxOutStorage`), resulting in stuck outbounds, vault slashing for unfulfilled signing obligations, or repeated execution of the outbound transaction (loss of vault funds through duplicate payouts) and potential false insolvency detection.

### Remediation

Replace `!m.IsType(memo.TxOutbound)` with `!m.IsOutbound()` in the `transferOutAndCallEvent` handler so that all valid outbound memo types (`TxOutbound`, `TxRefund`, `TxRagnarok`) are recognized consistently with `transferOutEvent`.

```go
			case transferOutAndCallEvent:
				transferOutAndCall, err := scp.parseTransferOutAndCall(*item)
				if err != nil {
					scp.logger.Err(err).Msg("fail to parse transferOutAndCall event")
					continue
				}
				scp.logger.Info().Msgf("transferOutAndCall: %+v", transferOutAndCall)
				m, err := memo.ParseMemo(common.LatestVersion, transferOutAndCall.Memo)
				if err != nil {
					scp.logger.Err(err).Msgf("fail to parse memo: %s", transferOutAndCall.Memo)
					continue
				}
				if !m.IsOutbound() {
					scp.logger.Error().Msgf("%s is not an outbound memo", transferOutAndCall.Memo)
					continue
				}
				decimals := scp.decimalResolver(NativeTokenAddr)
```

## Finding 4: Multi-Event Observation Inconsistency and Field Contamination in `SmartContractLogParser.GetTxInItem` (`transferOutAndCallEvent`)

### Synopsis

In `SmartContractLogParser.GetTxInItem`, the `transferOutAndCallEvent` handler neither enforces single-event finality nor validates/resets `txInItem` state across multiple router logs in a single transaction, enabling order-dependent event precedence, cross-event field contamination, inconsistent conflict handling, and stale optional-field retention.

### Technical details

`SmartContractLogParser.GetTxInItem` in `bifrost/pkg/chainclients/shared/evm/smartcontract_log_parser.go` iterates over all transaction receipt logs emitted by valid THORChain router addresses and mutates a single shared `*types.TxInItem` struct in place.

When handling `case transferOutAndCallEvent:` (lines 295–337), the parser exhibits a multi-event observation consistency flaw compared to the other event handlers (`depositEvent`, `transferOutEvent`, and `transferAllowanceEvent`):

1. **Inconsistent Rejection of Conflicting Events**:
   - `depositEvent` (lines 193–198) explicitly rejects transactions containing prior events with conflicting `To` or `Memo` values by returning an error.
   - `transferAllowanceEvent` (lines 256–267) checks `txInItem.Sender`, `txInItem.To`, and `txInItem.Memo` against existing values on `txInItem`.
   - `transferOutAndCallEvent` (lines 295–337) performs no validation against pre-existing `txInItem.To`, `txInItem.Memo`, `txInItem.Coins`, or `txInItem.Sender` (specifically, it does not verify that `txInItem.Sender` matches `transferOutAndCall.Vault.String()`). Instead, it unconditionally overwrites `txInItem.Coins`, `txInItem.To`, `txInItem.Memo`, and `txInItem.Sender` (lines 312–328).

2. **Order-Dependent Event Precedence**:
   - `transferOutEvent` sets `earlyExit = true` (line 242) to terminate log processing immediately, treating outbound events as final.
   - `transferOutAndCallEvent` does not set `earlyExit = true` (and does not reset `isVaultTransfer = false`). Consequently, the final observed `txInItem` depends on log ordering:
     - In the Router's `transferOutAndCall` implementation, the external call `target.call{value:_safeAmount}(...)` executes *before* `emit TransferOutAndCall(...)`. Any Router event emitted during the external call appears earlier in `receipt.Logs`:
       - If a `transferOutEvent` is emitted during the subcall, it sets `earlyExit = true` and causes the legitimate trailing `TransferOutAndCall` event to be ignored.
       - If a `depositEvent` or `transferAllowanceEvent` is emitted first, the trailing `TransferOutAndCall` event silently overwrites it instead of rejecting the conflicting multi-event transaction.
     - Conversely, if `transferOutAndCallEvent` is emitted *before* another event in a multi-call transaction, loop execution continues and allows subsequent `depositEvent`, `transferOutEvent`, `transferAllowanceEvent`, or `transferOutAndCallEvent` logs to mutate or overwrite `txInItem`.

3. **Cross-Event Observation Field Contamination**:
   - `transferOutAndCallEvent` mutates fields on `txInItem` that no other event handler resets:
     - `txInItem.Sender = transferOutAndCall.Vault.String()` (line 328)
     - `txInItem.Aggregator = aggregatorAddr.String()` (line 329)
     - `txInItem.AggregatorTarget = aggregatorTargetAddr.String()` (line 330)
     - `txInItem.AggregatorTargetLimit = &limit` (line 334)
   - If a subsequent `transferOutEvent` or `transferAllowanceEvent` is processed in the same transaction, it overwrites `To`, `Memo`, and `Coins`, but leaves `Sender` (now spoofed to `transferOutAndCall.Vault`), `Aggregator`, `AggregatorTarget`, and `AggregatorTargetLimit` contaminated on `txInItem`.
   - If a subsequent `depositEvent` has a matching `To` and `Memo`, it appends its `Coin` to `txInItem.Coins` (line 210) on top of the `transferOutAndCall` coin while retaining the contaminated `Sender` and aggregator fields.
   - Furthermore, if `transferOutAndCallEvent` fails address parsing at lines 315–324 (`continue`), `txInItem.Coins` (lines 312–314) has already been overwritten before the `continue`, corrupting `txInItem.Coins` for the transaction.

4. **Stale Optional-Field Retention**:
   - At lines 331–336, `txInItem.AggregatorTargetLimit` is only assigned when `transferOutAndCall.AmountOutMin != nil` and `!limit.IsZero()`, and is never cleared when `AmountOutMin` is zero or nil:
     ```go
     if transferOutAndCall.AmountOutMin != nil {
         limit := cosmos.NewUintFromBigInt(transferOutAndCall.AmountOutMin)
         if !limit.IsZero() {
             txInItem.AggregatorTargetLimit = &limit
         }
     }
     ```
   - If an earlier `transferOutAndCallEvent` in the same transaction sets a non-zero `AggregatorTargetLimit` and a later `transferOutAndCallEvent` has `AmountOutMin == 0`, the stale `AggregatorTargetLimit` pointer from the earlier event is retained on `txInItem`.

### Assumptions and preconditions

- A single EVM transaction triggers more than one THORChain Router event within `maxLogs` (either via a contract invoking multiple Router functions when `EVMDisableContractWhitelist` is enabled or via a whitelisted contract/aggregator callback during `transferOutAndCall`).
- Bifrost's `EVMScanner` or `ETHScanner` processes the transaction receipt via `getTxInFromSmartContract` -> `SmartContractLogParser.GetTxInItem`.

### Proof of concept

1. **Cross-Event Contamination & Precedence Inversion**:
   - A transaction calls a contract that first invokes `router.transferOutAndCall(target, finalAsset, toAddr, 1000, "OUT:<txhash>")` with `0` or dust native value (emitting `TransferOutAndCall` with `Vault = attackerContract`, `AmountOutMin = 1000`, `To = toAddr`, `Memo = "OUT:<txhash>"`), and then invokes `router.transferOut(...)` or a second `router.transferOutAndCall(target2, finalAsset2, toAddr, 0, "OUT:<txhash>")`.
   - When `GetTxInItem` processes the first log (`TransferOutAndCall`), it sets:
     - `txInItem.Coins`
     - `txInItem.Sender = transferOutAndCall.Vault.String()`
     - `txInItem.Aggregator`, `txInItem.AggregatorTarget`
     - `txInItem.AggregatorTargetLimit = 1000`
   - Because `earlyExit` is not set, `GetTxInItem` proceeds to the second log:
     - If the second log is `TransferOutAndCall` with `AmountOutMin = 0`, `txInItem.AggregatorTargetLimit` is not reset and retains the stale `1000` limit from the first log.
     - If the second log is `TransferOut`, `txInItem.To`, `txInItem.Memo`, and `txInItem.Coins` are updated from `TransferOut`, while `txInItem.Sender`, `txInItem.Aggregator`, `txInItem.AggregatorTarget`, and `txInItem.AggregatorTargetLimit` remain contaminated with the values from the earlier `TransferOutAndCall` log.

2. **Subcall Event Injection During Outbound `transferOutAndCall`**:
   - During a legitimate vault outbound call to `router.transferOutAndCall(...)`, `target.call` executes before the router emits `TransferOutAndCall`.
   - If `target` (or a downstream contract called during `swapOut`) calls `router.transferOut(...)` with an outbound memo, the `TransferOut` log appears *before* `TransferOutAndCall` in `receipt.Logs`.
   - `GetTxInItem` processes `TransferOut` first and sets `earlyExit = true` (line 242), breaking out of the loop and ignoring the actual `TransferOutAndCall` event emitted by the vault's transaction.

### Impact assessment

- **Severity**: High / Medium
- **Technical Impact**: Bifrost observers can construct a corrupted `TxInItem` combining fields from distinct router events in the same transaction (e.g., spoofing `txInItem.Sender`, attaching unintended `Aggregator` / `AggregatorTarget` / `AggregatorTargetLimit` metadata to non-aggregator or subsequent events, retaining a stale `AggregatorTargetLimit`, or dropping a legitimate `TransferOutAndCall` observation in favor of an earlier subcall event).
- **Business Impact**: Inconsistent or corrupted outbound/inbound transaction observations in THORNode can cause outbound accounting mismatches, failed or misattributed vault outbounds, or divergence between expected and observed aggregator swap-out parameters.

### Remediation

Validate `txInItem` consistency before mutating state in `case transferOutAndCallEvent:`, validate `aggregatorAddr` and `aggregatorTargetAddr` before mutating `txInItem.Coins`, explicitly reset `txInItem.AggregatorTargetLimit` when `AmountOutMin` is zero/nil, verify `txInItem.Sender` matches `transferOutAndCall.Vault`, reject transactions that mix conflicting router events, and set `earlyExit = true` and `isVaultTransfer = false`.

```go
			case transferOutAndCallEvent:
				if len(txInItem.To) > 0 || len(txInItem.Memo) > 0 || !txInItem.Coins.IsEmpty() {
					return false, fmt.Errorf("multiple conflicting events in the same transaction, ignore")
				}
				transferOutAndCall, err := scp.parseTransferOutAndCall(*item)
				if err != nil {
					scp.logger.Err(err).Msg("fail to parse transferOutAndCall event")
					continue
				}
				scp.logger.Info().Msgf("transferOutAndCall: %+v", transferOutAndCall)
				if len(txInItem.Sender) > 0 && !strings.EqualFold(txInItem.Sender, transferOutAndCall.Vault.String()) {
					scp.logger.Error().Msg("transferOutAndCall event, vault address is not the same as sender, ignore")
					continue
				}
				m, err := memo.ParseMemo(common.LatestVersion, transferOutAndCall.Memo)
				if err != nil {
					scp.logger.Err(err).Msgf("fail to parse memo: %s", transferOutAndCall.Memo)
					continue
				}
				if !m.IsType(memo.TxOutbound) {
					scp.logger.Error().Msgf("%s is not an outbound memo", transferOutAndCall.Memo)
					continue
				}
				aggregatorAddr, err := common.NewAddress(transferOutAndCall.Target.String())
				if err != nil {
					scp.logger.Err(err).Str("aggregator_address", transferOutAndCall.Target.String()).Msg("fail to parse aggregator address")
					continue
				}
				aggregatorTargetAddr, err := common.NewAddress(transferOutAndCall.FinalAsset.String())
				if err != nil {
					scp.logger.Err(err).Str("final_asset", transferOutAndCall.FinalAsset.String()).Msg("fail to parse aggregator target address")
					continue
				}
				decimals := scp.decimalResolver(NativeTokenAddr)
				txInItem.Coins = common.Coins{
					common.NewCoin(scp.nativeAsset, scp.amtConverter(NativeTokenAddr, transferOutAndCall.Amount)).WithDecimals(decimals),
				}
				txInItem.To = transferOutAndCall.To.String()
				txInItem.Memo = transferOutAndCall.Memo
				txInItem.Sender = transferOutAndCall.Vault.String()
				txInItem.Aggregator = aggregatorAddr.String()
				txInItem.AggregatorTarget = aggregatorTargetAddr.String()
				txInItem.AggregatorTargetLimit = nil
				if transferOutAndCall.AmountOutMin != nil {
					limit := cosmos.NewUintFromBigInt(transferOutAndCall.AmountOutMin)
					if !limit.IsZero() {
						txInItem.AggregatorTargetLimit = &limit
					}
				}
				earlyExit = true
				isVaultTransfer = false
```

## Finding 5: Unconditional `TransferOut` Event Emission on Failed ETH Send in `THORChain_Router.transferOut` Causes False Outbound Settlement Observation

### Synopsis

In `THORChain_Router.transferOut`, when a native ETH transfer via `to.send(safeAmount)` fails, the contract refunds the ETH to the calling vault (`msg.sender`) but unconditionally emits `TransferOut(msg.sender, to, asset, safeAmount, memo)`, causing THORChain observers to record a failed payout to `to` as successfully settled.

### Technical details

In `THORChain_Router.sol`, the `transferOut` function handles native ETH outbound transfers (`asset == address(0)`) as follows:

```solidity
if (asset == address(0)) {
  safeAmount = msg.value;
  bool success = to.send(safeAmount); // Send ETH.
  if (!success) {
    payable(address(msg.sender)).transfer(safeAmount); // For failure, bounce back to vault & continue.
  }
}
...
emit TransferOut(msg.sender, to, asset, safeAmount, memo);
```

This implementation introduces a critical mismatch between actual on-chain settlement and the emitted event observed by THORChain's off-chain infrastructure (Bifrost):

1. **2,300 Gas Stipend Failure**: Line 194 uses `to.send(safeAmount)`, which forwards only a 2,300 gas stipend. Any recipient `to` that is a smart contract (e.g., a multisig/smart contract wallet or a contract with state-modifying `receive()`/`fallback()` logic) or that reverts will cause `to.send(safeAmount)` to return `false`.
2. **Silent Bounce to Vault**: In lines 195–197, when `success == false`, the router sends `safeAmount` back to `payable(address(msg.sender))` (the vault) and continues execution without reverting.
3. **Mismatched Event Attribution**: At line 206, `emit TransferOut(msg.sender, to, asset, safeAmount, memo)` is executed unconditionally with `to` as the recipient and `safeAmount` as the transferred amount, even though `to` received `0` ETH and the funds were actually returned to `msg.sender`.

Because Bifrost's `smartcontract_log_parser` relies on the `TransferOut` event emitted by the Router to verify and finalize outbound transactions on THORChain, it observes a valid `TransferOut` event matching the outbound `memo`, `to`, and `safeAmount`. Consequently, THORNode marks the outbound transaction as completed and decrements the vault's tracked balance, while the recipient `to` never receives the ETH and the returned ETH sits unaccounted in the vault EOA.

### Assumptions and preconditions

1. A THORChain vault calls `transferOut` with `asset == address(0)` and `msg.value > 0` to fulfill an outbound native ETH transfer to recipient `to`.
2. The recipient address `to` is a contract whose `receive()` or `fallback()` function reverts or consumes more than the 2,300 gas stipend forwarded by `.send()`.

### Proof of concept

1. A user initiates a swap or withdrawal on THORChain specifying a smart contract wallet `to` (whose `receive()` function uses more than 2,300 gas, e.g., emits an event or updates storage) as the ETH recipient.
2. The THORChain Asgard vault calls `THORChain_Router.transferOut(to, address(0), amount, "OUT:<txhash>")` with `msg.value = amount`.
3. At line 194, `to.send(safeAmount)` runs out of gas (exceeding the 2,300 stipend) and returns `success = false`.
4. At lines 195–197, `!success` is true, so `payable(address(msg.sender)).transfer(safeAmount)` returns the `amount` of ETH back to the Asgard vault.
5. At line 206, `TransferOut(asgardVault, to, address(0), amount, "OUT:<txhash>")` is emitted.
6. Bifrost parses the `TransferOut` event log, observes that `amount` ETH was sent to `to` with memo `"OUT:<txhash>"`, and reports the outbound transfer as successfully completed to THORNode.
7. The user `to` receives `0` ETH, while THORChain records the outbound obligation as settled.

### Impact assessment

- **Severity**: High / Medium
- **Technical Impact**: Outbound ETH transfers to smart contract wallets or reverting recipients silently fail while still emitting a canonical `TransferOut` event attributing the transfer to `to`.
- **Business Impact**: Users withdrawing or swapping into ETH to smart contract wallets (or any contract requiring >2,300 gas on `receive()`) lose their outbound funds as THORChain marks the payout fulfilled without delivering ETH to the recipient, and vault accounting becomes desynchronized from actual on-chain balances.

### Remediation

Use a low-level `.call{value: safeAmount}("")` instead of `.send(safeAmount)` so smart contract recipients are not restricted to the 2,300 gas stipend, and either revert on failure or ensure any fallback path accurately reflects the actual recipient or emits a distinct failure/refund event so off-chain observers do not misattribute failed settlements as successful payouts to `to`.

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
      (bool success, ) = to.call{value: safeAmount}(""); // Send ETH.
      if (!success) {
        payable(address(msg.sender)).transfer(safeAmount); // For failure, bounce back to vault & continue.
        emit TransferOut(msg.sender, payable(address(msg.sender)), asset, safeAmount, memo);
        return;
      }
    } else {
      _vaultAllowance[msg.sender][asset] -= amount; // Reduce allowance
      (bool success, bytes memory data) = asset.call(
        abi.encodeWithSignature("transfer(address,uint256)", to, amount)
      );
      require(success && (data.length == 0 || abi.decode(data, (bool))));
      safeAmount = amount;
    }
    emit TransferOut(msg.sender, to, asset, safeAmount, memo);
  }
```

## Finding 6: Hardcoded 2300 Gas Stipend in `THORChain_Router.transferOut` Causes Native ETH Payouts and Refunds to Smart Contract Recipients to Fail

### Synopsis

`THORChain_Router.transferOut` uses Solidity's `.send()` and `.transfer()` primitives to forward native ETH to recipients and refund callers on failure. Because both primitives impose a hardcoded 2300 gas stipend, native ETH payouts to smart contract wallets fail and bounce back while still emitting `TransferOut`, or revert entirely during refund.

### Technical details

In `THORChain_Router.transferOut` (`ethereum/contracts/THORChain_Router.sol`), when `asset == address(0)`, the router processes native ETH outbound transfers as follows:

```solidity
if (asset == address(0)) {
  safeAmount = msg.value;
  bool success = to.send(safeAmount); // Send ETH.
  if (!success) {
    payable(address(msg.sender)).transfer(safeAmount); // For failure, bounce back to vault & continue.
  }
}
...
emit TransferOut(msg.sender, to, asset, safeAmount, memo);
```

Both `to.send(safeAmount)` at line 194 and `payable(address(msg.sender)).transfer(safeAmount)` at line 196 forward a fixed 2300 gas stipend to the recipient's `receive()` or `fallback()` function. Under EIP-1884 and EIP-2929 gas schedule changes, 2300 gas is insufficient to execute even a single `SLOAD`, `SSTORE`, or proxy `DELEGATECALL`.

This causes two distinct failure modes:

1. **Silent Payout Failure to Smart Contract Recipients:** When `to` is a smart contract wallet (e.g., a multisig such as Safe, an EIP-1967 proxy, or an account abstraction wallet) whose `receive()` or `fallback()` consumes more than 2300 gas (for instance, by emitting an event with indexed data, reading implementation storage in a proxy, or updating state), `to.send(safeAmount)` returns `false`. The ETH is bounced back to `msg.sender` (the vault), yet execution continues to line 206 and emits `TransferOut(msg.sender, to, address(0), safeAmount, memo)` as if the outbound payout succeeded. Off-chain observers (Bifrost) treat the outbound obligation as fulfilled even though the recipient `to` received 0 ETH.
2. **Refund Revert for Smart Contract Callers:** When `to.send(safeAmount)` returns `false` and `msg.sender` is a contract whose `receive()`/`fallback()` requires more than 2300 gas, `payable(address(msg.sender)).transfer(safeAmount)` at line 196 runs out of gas and reverts the entire outbound transaction.

### Assumptions and preconditions

- A THORChain vault (or caller) invokes `transferOut` with `asset == address(0)` and `msg.value > 0`.
- The recipient `to` (or caller `msg.sender` during a bounce refund) is a smart contract whose `receive()` or `fallback()` logic consumes more than 2300 gas (such as a proxy wallet or multisig contract).

### Proof of concept

1. A user initiates a cross-chain swap on THORChain specifying their Ethereum EIP-1967 proxy smart contract wallet (whose `fallback` performs an `SLOAD` + `DELEGATECALL` costing >2300 gas) as the recipient `to` of native ETH.
2. The THORChain vault calls `THORChain_Router.transferOut(to, address(0), amount, memo)` with `msg.value = amount`.
3. At line 194, `to.send(safeAmount)` forwards only 2300 gas to `to`. Because `to` is a proxy contract that reads its implementation slot via `SLOAD` (2100 gas cold) and delegates the call, it exceeds 2300 gas and fails with out-of-gas, returning `success = false`.
4. At line 195–196, `if (!success)` executes `payable(address(msg.sender)).transfer(safeAmount)`, returning `safeAmount` ETH back to the EOA vault.
5. At line 206, `emit TransferOut(msg.sender, to, address(0), safeAmount, memo)` is emitted with the full `safeAmount`.
6. THORChain's Bifrost parses the `TransferOut` event, marks the user's outbound ETH payout as completed, and deducts the outbound from the vault's pending obligations, while the user `to` never received their ETH.
7. Alternatively, if `msg.sender` is a contract wallet requiring >2300 gas on `receive()` and `to` rejects or runs out of gas on `.send()`, line 196 `payable(address(msg.sender)).transfer(safeAmount)` reverts the transaction.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Legitimate smart contract wallets (e.g., Safe multisigs, proxy contracts, smart accounts) cannot receive native ETH via `transferOut`. Any native ETH payout to such addresses fails the `.send()` call, returns the ETH to the vault, and still emits `TransferOut`, causing the protocol to account for the payout as completed while the user loses their outbound payout. Additionally, contract callers requiring >2300 gas on `receive()` cannot receive bounce refunds without reverting.

### Remediation

Replace `.send()` and `.transfer()` with low-level `.call{value: safeAmount}("")` (which is safe from reentrancy here because `transferOut` is protected by the `nonReentrant` modifier), and ensure refund failures are properly checked:

```solidity
    if (asset == address(0)) {
      safeAmount = msg.value;
      (bool success, ) = to.call{value: safeAmount}(""); // Send ETH.
      if (!success) {
        (bool refundSuccess, ) = payable(address(msg.sender)).call{value: safeAmount}(""); // For failure, bounce back to vault & continue.
        require(refundSuccess, "Refund failed");
      }
    } else {
```

## Finding 7: Missing `msg.value == 0` Validation in `THORChain_Router.transferOut` Token-Transfer Path Locks Native ETH in the Router

### Synopsis

`THORChain_Router.transferOut` is marked `payable` to support native ETH payouts when `asset == address(0)`, but its ERC20 token-transfer branch (`asset != address(0)`) fails to require `msg.value == 0`. Consequently, any native ETH sent alongside an ERC20 outbound transfer remains untracked in the router contract and can be drained by any caller via `transferOutV5` or `batchTransferOutV5`.

### Technical details

In `ethereum/contracts/THORChain_Router.sol`, the `transferOut` function (lines 185–207) handles both native asset (`asset == address(0)`) and ERC20 token (`asset != address(0)`) outbound transfers from vaults:

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
      _vaultAllowance[msg.sender][asset] -= amount; // Reduce allowance
      (bool success, bytes memory data) = asset.call(
        abi.encodeWithSignature("transfer(address,uint256)", to, amount)
      );
      require(success && (data.length == 0 || abi.decode(data, (bool))));
      safeAmount = amount;
    }
    emit TransferOut(msg.sender, to, asset, safeAmount, memo);
  }
```

Unlike `_deposit` (line 155: `require(msg.value == 0, "unexpected eth");`) and `AvaxRouter.deposit` (line 124: `require(msg.value == 0, "unexpected avax");`), the `else` branch in `transferOut` (lines 198–205) does not check that `msg.value == 0`.

When `transferOut` is invoked for an ERC20 token (`asset != address(0)`) with a non-zero `msg.value`:
1. The ERC20 allowance of `msg.sender` is decremented by `amount` and the ERC20 tokens are transferred to `to`.
2. The `TransferOut` event is emitted with `safeAmount = amount` for the ERC20 asset.
3. The attached native ETH (`msg.value`) is neither forwarded to `to` nor refunded to `msg.sender`, nor does the transaction revert. Instead, the ETH stays untracked in the `THORChain_Router` contract's balance.
4. Because `_transferOutV5` (lines 209–238, reachable publicly via `transferOutV5` and `batchTransferOutV5`) sends `transferOutPayload.amount` of native ETH directly from the contract's ETH balance (`transferOutPayload.to.send(transferOutPayload.amount)`) without checking that `msg.value >= transferOutPayload.amount`, any untracked ETH left in the router can immediately be swept by an unprivileged external caller.

### Assumptions and preconditions

- A vault or caller invokes `transferOut` with an ERC20 token address (`asset != address(0)`) and inadvertently includes a non-zero `msg.value` in the transaction.
- `msg.sender` holds sufficient `_vaultAllowance[msg.sender][asset]` for the ERC20 transfer to succeed.

### Proof of concept

1. A vault holds an allowance of `1,000` units of an ERC20 `TOKEN` in `THORChain_Router` (`_vaultAllowance[vault][TOKEN] == 1000`).
2. The vault calls `transferOut(recipient, TOKEN, 1000, "OUT:...")` and accidentally attaches `1 ETH` (`msg.value = 1 ether`).
3. Execution enters the `else` branch at line 198 because `asset != address(0)`:
   - `_vaultAllowance[vault][TOKEN]` is reduced by `1,000`.
   - `1,000 TOKEN` is transferred to `recipient`.
   - `1 ETH` remains locked in `address(THORChain_Router)`.
4. An unprivileged user calls `transferOutV5(TransferOutData({ to: payable(attacker), asset: address(0), amount: 1 ether, memo: "" }))` with `msg.value = 0`, draining the `1 ETH` from `THORChain_Router`.

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: Native ETH sent along with an ERC20 `transferOut` call is silently absorbed by the `THORChain_Router` contract without reverting or being accounted for.
- **Business Impact**: Permanent loss of vault/user native ETH funds, as any residual ETH balance in the router can be extracted permissionlessly via `transferOutV5` or `batchTransferOutV5`.

### Remediation

Enforce `require(msg.value == 0, "unexpected eth");` in the ERC20 branch (`else` block) of `transferOut`, matching the protection already implemented in `_deposit`.

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
```

## Finding 8: Unchecked Balance Delta in `THORChain_Router.transferOut` Causes Allowance-to-Custody Accounting Divergence for Fee-Bearing and Rebasing ERC20 Tokens

### Synopsis

In `THORChain_Router.transferOut`, ERC20 outbound transfers deduct the nominal `amount` from `_vaultAllowance` and emit `safeAmount = amount` without measuring actual balance changes. For fee-bearing or rebasing tokens, this creates a divergence between tracked vault allowances and actual Router token custody, misreporting settled amounts and locking vault funds.

### Technical details

The Router tracks ERC20 custody per vault using the `_vaultAllowance[vault][asset]` mapping. During inbound deposits in `_deposit`, the contract calls `safeTransferFrom(asset, amount)` (lines 438–453), which records the Router's token balance before and after the `transferFrom` call:

```solidity
uint _startBal = iERC20(_asset).balanceOf(address(this));
...
return (iERC20(_asset).balanceOf(address(this)) - _startBal);
```

This ensures `_vaultAllowance` is credited with the exact net tokens received by the Router.

However, in `transferOut` (lines 198–206), the ERC20 settlement logic does not measure the actual balance delta on the Router or recipient:

```solidity
} else {
  _vaultAllowance[msg.sender][asset] -= amount; // Reduce allowance
  (bool success, bytes memory data) = asset.call(
    abi.encodeWithSignature("transfer(address,uint256)", to, amount)
  );
  require(success && (data.length == 0 || abi.decode(data, (bool))));
  safeAmount = amount;
}
emit TransferOut(msg.sender, to, asset, safeAmount, memo);
```

This creates multiple settlement and accounting discrepancies:
1. **Allowance-to-Custody Divergence (Sender-Side Fee / Negative Rebase):** If an ERC20 token deducts a transfer fee from the sender (`address(this)`) on `transfer(to, amount)`, the Router's actual token balance decreases by `amount + fee`, whereas `_vaultAllowance[msg.sender][asset]` is only decremented by `amount`. Similarly, if a token undergoes a negative rebase while held by the Router, `_vaultAllowance[msg.sender][asset]` remains higher than the actual custodied balance. In multi-vault scenarios, one vault's outbound transfer consumes tokens backing another vault's allowance; for the last vault (or a single vault withdrawing its full allowance), `transferOut` reverts due to insufficient token balance in the Router.
2. **Inaccurate Outbound Settlement Reporting (Recipient-Side Fee):** Although `safeAmount` is declared to represent the actual settled amount, line 204 hardcodes `safeAmount = amount`. When a fee-on-transfer token deducts a fee from `amount`, the recipient `to` receives `amount - fee`, while the emitted `TransferOut` event reports `amount` to off-chain observers (`smartcontract_log_parser`).
3. **Non-Existent Token Code Execution:** Because `asset.call` is low-level and does not verify `asset.code.length > 0` (unlike `safeTransferFrom`, which calls `iERC20(_asset).balanceOf(address(this))` first), calling `transferOut` with `amount == 0` on an address without deployed code succeeds with `data.length == 0` and emits a `TransferOut` event.

### Assumptions and preconditions

1. Users deposit a fee-bearing or rebasing ERC20 token into the Router via `depositWithExpiry`, crediting one or more THORChain vaults in `_vaultAllowance`.
2. A vault initiates `transferOut` to settle an outbound ERC20 transfer.

### Proof of concept

1. Assume ERC20 token `T` charges a 1% sender-side fee on transfers (or experiences a balance reduction via rebase).
2. Vault A and Vault B each receive deposits resulting in `_vaultAllowance[VaultA][T] = 1000` and `_vaultAllowance[VaultB][T] = 1000`, with the Router holding `2000 T`.
3. Vault A calls `transferOut(recipientA, T, 1000, "OUT:A")`.
4. Line 199 reduces `_vaultAllowance[VaultA][T]` by `1000` (to `0`).
5. Lines 200–202 execute `T.transfer(recipientA, 1000)`. Token `T` transfers `1000 T` to `recipientA` and deducts the `10 T` fee from the Router's balance, leaving the Router with `990 T`.
6. Vault B attempts to withdraw its full allowance by calling `transferOut(recipientB, T, 1000, "OUT:B")` (or churns via `returnVaultAssets`).
7. The transaction reverts inside `T.transfer(recipientB, 1000)` because `_vaultAllowance[VaultB][T]` is `1000`, but the Router's actual custody balance is only `990 T`, permanently stranding the remaining allowance unless manually calculated and partially withdrawn.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** The invariant that the sum of `_vaultAllowance` across vaults matches the Router's actual ERC20 custody balance is violated, and `TransferOut` events emit nominal rather than settled amounts.
- **Business Impact:** Vaults cannot withdraw or churn their full recorded `_vaultAllowance` when tokens charge sender-side transfer fees or negatively rebase, leading to stuck vault funds and failed vault churns (`returnVaultAssets`), or cross-vault depletion of custodied ERC20 balances.

### Remediation

Measure the Router's token balance before and after the outbound `transfer` call (or cap the transfer amount to the available balance when necessary) and deduct the actual balance delta from `_vaultAllowance[msg.sender][asset]`, ensuring `safeAmount` reflects the true settled amount and `iERC20(asset).balanceOf` enforces that `asset` is a deployed contract.

```solidity
    } else {
      uint startBal = iERC20(asset).balanceOf(address(this));
      (bool success, bytes memory data) = asset.call(
        abi.encodeWithSignature("transfer(address,uint256)", to, amount)
      );
      require(success && (data.length == 0 || abi.decode(data, (bool))));
      safeAmount = startBal - iERC20(asset).balanceOf(address(this));
      _vaultAllowance[msg.sender][asset] -= safeAmount; // Reduce allowance by actual amount deducted
    }
    emit TransferOut(msg.sender, to, asset, safeAmount, memo);
```

## Finding 9: Unconditional `TransferOut` emission on failed ETH `.send()` in `THORChain_Router._transferOutV5` causes bounced refunds to be observed as completed settlements

### Synopsis

In `THORChain_Router._transferOutV5`, when an outbound ETH `.send()` to `transferOutPayload.to` fails, the ETH is refunded back to `msg.sender` (the vault), but `TransferOut` is still unconditionally emitted with the original recipient and amount, causing off-chain observers to misclassify failed outbounds as completed settlements.

### Technical details

In `THORChain_Router._transferOutV5` (called by `transferOutV5` and `batchTransferOutV5`), native ETH outbounds (`transferOutPayload.asset == address(0)`) are executed on lines 210–214:

```solidity
if (transferOutPayload.asset == address(0)) {
  bool success = transferOutPayload.to.send(transferOutPayload.amount); // Send ETH.
  if (!success) {
    payable(address(msg.sender)).transfer(transferOutPayload.amount); // For failure, bounce back to vault & continue.
  }
}
```

Subsequently, on lines 231–237, `_transferOutV5` unconditionally emits the `TransferOut` event using the original payload parameters:

```solidity
emit TransferOut(
  msg.sender,
  transferOutPayload.to,
  transferOutPayload.asset,
  transferOutPayload.amount,
  transferOutPayload.memo
);
```

This creates an event-driven settlement integrity flaw across three dimensions:

1. **Payment outcome and event divergence**: `transferOutPayload.to.send(transferOutPayload.amount)` forwards only a 2,300 gas stipend and returns `false` without reverting if the recipient call fails or runs out of gas (for example, when `transferOutPayload.to` is a smart contract wallet, multisig, or proxy whose `receive()`/`fallback()` logic consumes more than 2,300 gas). Even though `transferOutPayload.to` receives `0` ETH, the emitted `TransferOut` event reports that `transferOutPayload.amount` was sent to `transferOutPayload.to`.
2. **Refund versus settlement misclassification**: On `.send()` failure (`!success`), the contract refunds `transferOutPayload.amount` back to `msg.sender` (the vault) via `.transfer()`. However, `TransferOut` is emitted with `to = transferOutPayload.to` rather than reflecting the bounce/refund to `msg.sender` or emitting a distinct failure/refund event.
3. **Incomplete or ambiguous batch outbound observation**: In `batchTransferOutV5`, `_transferOutV5` is invoked in a loop over an array of `TransferOutData` structs. If a subset of ETH transfers in the batch fails `.send()` while others succeed, the failed transfers silently bounce back to `msg.sender` while emitting `TransferOut` events indistinguishable from the successful transfers in the same transaction. Furthermore, `_transferOutV5` does not reconcile `msg.value` against the sum of ETH `transferOutPayload.amount` values in the batch.

Because THORChain's off-chain observer (`smartcontract_log_parser` `GetTxInItem`) relies on `TransferOut` events emitted by `THORChain_Router` to confirm outbound execution, a failed outbound that bounced back to the vault is recorded by the protocol as a completed settlement to the user.

### Assumptions and preconditions

1. A THORChain vault executes an outbound native ETH transfer (`asset == address(0)`) via `transferOutV5` or `batchTransferOutV5`.
2. The recipient address `transferOutPayload.to` is a contract whose `receive()` or `fallback()` function consumes more than the 2,300 gas stipend provided by `.send()` (such as a multisig, proxy, or smart account) or reverts upon receiving ETH.
3. The vault (`msg.sender`) is capable of receiving the bounced ETH via `.transfer()`.

### Proof of concept

1. A user initiates a cross-chain swap or withdrawal on THORChain with an Ethereum smart contract wallet (e.g., a proxy wallet or multisig whose `receive()` function performs storage writes or emits events requiring > 2,300 gas) as the destination address `to`.
2. A THORChain vault calls `batchTransferOutV5` (or `transferOutV5`) including a `TransferOutData` entry with:
   - `to`: the user's smart contract wallet address
   - `asset`: `address(0)`
   - `amount`: `1 ether`
   - `memo`: `"OUT:<txHash>"`
3. Inside `_transferOutV5`:
   - `transferOutPayload.to.send(1 ether)` forwards only 2,300 gas, runs out of gas in the recipient's `receive()` hook, and returns `success = false`.
   - The `if (!success)` branch executes `payable(address(msg.sender)).transfer(1 ether)`, returning the `1 ether` back to the calling vault.
   - Execution proceeds to line 231 and emits `TransferOut(vault, to, address(0), 1 ether, "OUT:<txHash>")`.
4. Off-chain observers (`smartcontract_log_parser`) parse the emitted `TransferOut` event, observe that `1 ether` was transferred to `to` with memo `"OUT:<txHash>"`, and mark the outbound obligation as fulfilled on THORChain, even though the recipient received `0` ETH and the funds were refunded to the vault.

### Impact assessment

- **Severity**: High / Medium
- **Direct Technical Effect**: Outbound ETH transfers that fail due to the 2,300 gas limit of `.send()` or recipient revert are bounced back to the vault while still emitting a `TransferOut` event claiming full delivery to `transferOutPayload.to`.
- **Business & Protocol Implications**: Off-chain settlement accounting diverges from on-chain token/ETH movements. THORChain considers the user's outbound withdrawal or swap fulfilled without retrying or refunding the user's source transaction, resulting in permanent loss of user funds (stuck as unaccounted surplus in the vault) until manual intervention.

### Remediation

Update `_transferOutV5` so that either failed ETH transfers revert (ensuring atomic batch/outbound execution and preventing false `TransferOut` emissions) or the emitted event / recipient accurately reflects that the funds were bounced back to `msg.sender` rather than delivered to `transferOutPayload.to`, and validate `msg.value` consistency.

If non-reverting bounce behavior is required so a single failing recipient does not block a batch, update the logged recipient or emit a distinct bounce event instead of emitting a misleading `TransferOut` to `transferOutPayload.to`:

```solidity
  function _transferOutV5(TransferOutData memory transferOutPayload) private {
    address payable recipient = transferOutPayload.to;
    if (transferOutPayload.asset == address(0)) {
      bool success = transferOutPayload.to.send(transferOutPayload.amount); // Send ETH.
      if (!success) {
        recipient = payable(address(msg.sender));
        recipient.transfer(transferOutPayload.amount); // For failure, bounce back to vault & continue.
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
      recipient,
      transferOutPayload.asset,
      transferOutPayload.amount,
      transferOutPayload.memo
    );
  }
```

## Finding 10: Missing `msg.value` Validation in `THORChain_Router._transferOutV5` Allows Unbacked ETH Disbursements and Draining of Contract ETH Balance

### Synopsis

In `THORChain_Router._transferOutV5`, native ETH transfers use the caller-supplied `transferOutPayload.amount` instead of validating it against `msg.value`. This mismatch between caller funding and ETH disbursement allows excess ETH to be locked in the router and lets any external caller drain residual ETH held by the contract.

### Technical details

In the legacy `transferOut` function (lines 192–197), when transferring native ETH (`asset == address(0)`), the contract sets `safeAmount = msg.value` and sends `safeAmount` to the recipient. This guarantees a strict 1:1 correspondence between the ETH supplied by the caller in the transaction and the ETH disbursed by the router.

In contrast, `_transferOutV5` (lines 210–214), which is invoked by the public `transferOutV5` and external `batchTransferOutV5` functions, disburses `transferOutPayload.amount` directly:

```solidity
if (transferOutPayload.asset == address(0)) {
  bool success = transferOutPayload.to.send(transferOutPayload.amount); // Send ETH.
  if (!success) {
    payable(address(msg.sender)).transfer(transferOutPayload.amount); // For failure, bounce back to vault & continue.
  }
}
```

Neither `_transferOutV5`, `transferOutV5`, nor `batchTransferOutV5` verifies that `msg.value` equals `transferOutPayload.amount` (for single transfers) or the aggregate sum of `transferOutPayload[i].amount` across all native ETH entries (for batch transfers).

This introduces several flaws:
1. **Unfunded Withdrawals / Pooled-Balance Drainage:** Because `transferOutV5` and `batchTransferOutV5` are publicly callable by anyone, an unprivileged caller can pass `asset = address(0)` and `amount = address(this).balance` with `msg.value = 0`. `_transferOutV5` will transfer the contract's existing ETH balance to `transferOutPayload.to` (or bounce it to `msg.sender` via `.transfer` if `to` rejects ETH).
2. **Accidental Overfunding:** If a vault or user calls `transferOutV5` or `batchTransferOutV5` with `msg.value` greater than the specified `transferOutPayload.amount` sum, the difference remains trapped in `THORChain_Router` rather than being transferred or refunded, immediately becoming stealable by any caller.
3. **Spoofed `TransferOut` Events for Zero-Value or Contract-Funded Calls:** If `THORChain_Router` holds ETH (e.g., via overfunding, failed sweeps, or forced ETH via `selfdestruct`/coinbase rewards), an attacker can trigger `_transferOutV5` with `msg.value = 0` and still emit a valid `TransferOut` event on lines 231–237 with a non-zero `amount`.

### Assumptions and preconditions

- For draining ETH: `THORChain_Router` holds a non-zero ETH balance (either from a previous call where `msg.value` exceeded the sum of `transferOutPayload.amount`, or from ETH sent directly/forcibly to the contract).
- Any external account can call `transferOutV5` or `batchTransferOutV5` because both functions have `public`/`external` visibility without caller access restrictions.

### Proof of concept

1. Suppose `THORChain_Router` holds 1 ETH (for example, a vault called `batchTransferOutV5` with `msg.value = 4 ether` for three 1 ETH transfers, leaving 1 ETH in the router).
2. An unprivileged user calls `transferOutV5(TransferOutData({ to: attackerAddress, asset: address(0), amount: 1 ether, memo: "" }))` with `msg.value = 0`.
3. Inside `_transferOutV5` at line 210, `transferOutPayload.asset == address(0)` evaluates to `true`.
4. At line 211, `transferOutPayload.to.send(1 ether)` executes using the router's own ETH balance and succeeds.
5. `attackerAddress` receives the 1 ETH without having supplied any `msg.value` or holding any vault allowance.

### Impact assessment

- **Severity:** Medium / High.
- **Technical Impact:** Discrepancy between `msg.value` and disbursed ETH in `_transferOutV5`, enabling unauthorized extraction of any ETH held by the `THORChain_Router` contract and potential loss of excess ETH sent during outbound transfers.
- **Business Impact:** Loss of protocol or vault funds whenever excess ETH is present in the router contract, and violation of the router's core accounting invariant that callers can only transfer out ETH they directly fund in the transaction.

### Remediation

Track or validate the ETH spent in `_transferOutV5` against `msg.value`. In `transferOutV5`, require `msg.value == transferOutPayload.amount` when `transferOutPayload.asset == address(0)` (and `msg.value == 0` otherwise). In `batchTransferOutV5`, accumulate the total ETH `amount` across all batch elements where `asset == address(0)` and require that the total equals `msg.value`.

```solidity
  function _transferOutV5(TransferOutData memory transferOutPayload) private returns (uint ethSpent) {
    if (transferOutPayload.asset == address(0)) {
      ethSpent = transferOutPayload.amount;
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
    uint ethSpent = _transferOutV5(transferOutPayload);
    require(ethSpent == msg.value, "Incorrect msg.value");
  }

  function batchTransferOutV5(
    TransferOutData[] calldata transferOutPayload
  ) external payable nonReentrant {
    uint totalEthSpent = 0;
    for (uint i = 0; i < transferOutPayload.length; ++i) {
      totalEthSpent += _transferOutV5(transferOutPayload[i]);
    }
    require(totalEthSpent == msg.value, "Incorrect msg.value");
  }
```

## Finding 11: Unconditional `TransferOutAndCall` Event Emission After Fallback Bounce-Back in `THORChain_Router.transferOutAndCall` Causes Settlement Accounting Mismatch and Loss of User Outbound Funds

### Synopsis

In `THORChain_Router.transferOutAndCall`, when both the external aggregator swap call and the fallback ETH `.send()` to the recipient fail, the ETH is bounced back to the calling vault (`msg.sender`), yet `TransferOutAndCall` is still unconditionally emitted as if the outbound settlement succeeded.

### Technical details

In `ethereum/contracts/THORChain_Router.sol`, `transferOutAndCall` processes native ETH outbound swaps initiated by a THORChain vault (`msg.sender`):

1. On lines 269–276, the router calls `aggregator.call{value: _safeAmount}(abi.encodeWithSignature("swapOut(address,address,uint256)", finalToken, to, amountOutMin))`.
2. If `erc20Success` is `false` (for example, due to slippage limits `amountOutMin` not being met or liquidity issues), execution enters the fallback block at line 277.
3. On line 278, the contract attempts to send `_safeAmount` of native ETH directly to the recipient `to` using `payable(to).send(_safeAmount)`, which forwards only a 2,300 gas stipend.
4. If `ethSuccess` is `false` (for example, `to` is a smart contract wallet, multisig, or contract without a payable fallback or whose `receive()` logic requires more than 2,300 gas, or intentionally reverts), line 280 executes:
   ```solidity
   payable(address(msg.sender)).transfer(_safeAmount);
   ```
   This returns `_safeAmount` ETH back to the calling vault (`msg.sender`, an EOA).
5. However, on lines 284–292, `transferOutAndCall` unconditionally emits:
   ```solidity
   emit TransferOutAndCall(
     msg.sender,
     aggregator,
     _safeAmount,
     finalToken,
     to,
     amountOutMin,
     memo
   );
   ```

As documented in `docs/upgrade_router.md`, Bifrost relies on the Router contract to emit accurate events to determine what happened on-chain for inbound/outbound settlements. Because `TransferOutAndCall` is emitted with the full `_safeAmount`, `to`, and `memo` (`OUT:<txhash>`) even when `_safeAmount` was bounced back to `msg.sender` (or when the swap failed and raw ETH was sent instead of `finalToken`), Bifrost and THORNode record the outbound transaction as successfully settled and deduct `_safeAmount` from the vault's tracked ETH balance on THORChain. In reality, when `!ethSuccess` occurs, the recipient `to` receives nothing and the vault retains the `_safeAmount` ETH on-chain, creating an event-to-value-flow inconsistency between the emitted settlement event and the actual asset transfer.

### Assumptions and preconditions

1. A vault executes an outbound aggregator transaction via `THORChain_Router.transferOutAndCall` (scheduled by THORNode/Bifrost in `EVMClient.getOutboundTxData` at `bifrost/pkg/chainclients/evm/client.go:447`).
2. The call to `aggregator.swapOut(...)` fails (`erc20Success == false`), which can happen naturally due to price movement exceeding `amountOutMin` or intentionally via a tight `AggregatorTargetLimit`.
3. The recipient address `to` is a contract that either rejects direct ETH transfers or consumes more than 2,300 gas in its `receive()`/`fallback()` function, causing `payable(to).send(_safeAmount)` to return `false`.

### Proof of concept

1. A user (or smart contract wallet / Gnosis Safe with a `receive()` hook requiring > 2,300 gas) initiates a cross-chain swap on THORChain specifying an EVM aggregator and a high `amountOutMin` (`AggregatorTargetLimit`), with `to` set to the contract wallet address.
2. Bifrost signs and broadcasts the vault transaction calling `THORChain_Router.transferOutAndCall(aggregator, finalToken, to, amountOutMin, "OUT:<in_hash>")` with `msg.value = _safeAmount`.
3. Inside `transferOutAndCall`:
   - `aggregator.call{value: _safeAmount}(...)` reverts because `amountOutMin` cannot be satisfied (`erc20Success = false`).
   - `payable(to).send(_safeAmount)` fails because `to` consumes more than 2,300 gas or reverts on direct ETH receipt (`ethSuccess = false`).
   - `payable(address(msg.sender)).transfer(_safeAmount)` sends `_safeAmount` ETH back to the Asgard vault EOA.
   - `TransferOutAndCall(vault, aggregator, _safeAmount, finalToken, to, amountOutMin, "OUT:<in_hash>")` is emitted.
4. Bifrost observes the `TransferOutAndCall` event matching the outbound memo `"OUT:<in_hash>"` and reports the outbound transaction as completed to THORNode.
5. THORNode marks the `TxOutItem` as fulfilled and deducts `_safeAmount` from the vault's book balance, while the user receives neither `finalToken` nor ETH, and the vault's on-chain ETH balance diverges from THORNode's accounting (triggering solvency mismatches).

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** `TransferOutAndCall` is emitted claiming `_safeAmount` exited the vault to fulfill `memo` even when the ETH bounced back to `msg.sender`.
- **Business & Protocol Implications:**
  1. **Permanent Loss of User Outbound Funds:** Users (especially smart contract wallets such as Gnosis Safe whose `receive()` fallback exceeds the 2,300 gas stipend of `.send()`) permanently lose their outbound funds if the aggregator swap fails, because THORNode marks the outbound as completed and never reschedules or refunds it.
  2. **Vault Solvency Discrepancy:** Because the vault retains `_safeAmount` on-chain while THORNode deducts `_safeAmount` from the vault's tracked balance upon observing `TransferOutAndCall`, the vault's on-chain balance and THORNode's internal accounting become desynchronized.

### Remediation

Revert when the fallback transfer to `to` fails (or use a bounded gas `.call` to `to` and revert if `to` still cannot receive the ETH) rather than bouncing the funds back to the vault while still emitting `TransferOutAndCall`, or do not emit `TransferOutAndCall` as a successful outbound to `to` when funds are returned to the vault.

Preferred fix in `ethereum/contracts/THORChain_Router.sol`:

```solidity
    if (!erc20Success) {
      (bool ethSuccess, ) = payable(to).call{value: _safeAmount}("");
      require(ethSuccess, "Failed to send ETH to recipient");
    }

    emit TransferOutAndCall(
      msg.sender,
      aggregator,
      _safeAmount,
      finalToken,
      to,
      amountOutMin,
      memo
    );
```

## Finding 12: Unbounded Subcall Gas Forwarding and 2300-Gas Stipend Transfers in `THORChain_Router.transferOutAndCall` Cause Fallback Revert and Stuck Outbounds

### Synopsis

In `THORChain_Router.transferOutAndCall`, the fallback recovery path uses `.send()` and `.transfer()` with fixed 2,300-gas stipends after an unbounded `aggregator.call` that can consume 63/64 of transaction gas. Consequently, a failing swap can exhaust recovery gas or fail recipient/vault ETH transfers, reverting the outbound transaction.

### Technical details

In `ethereum/contracts/THORChain_Router.sol`, `transferOutAndCall` attempts an external swap via `aggregator.call{value: _safeAmount}(...)` (line 269) and implements a two-step native ETH recovery path on lines 277–282 when `erc20Success` is `false`:

```solidity
    if (!erc20Success) {
      bool ethSuccess = payable(to).send(_safeAmount); // If can't swap, just send the recipient the ETH
      if (!ethSuccess) {
        payable(address(msg.sender)).transfer(_safeAmount); // For failure, bounce back to vault & continue.
      }
    }
```

This fallback mechanism is designed to guarantee that a failed aggregator call never reverts the outer transaction, allowing the router to either deliver the native ETH to `to` or bounce it back to the calling vault (`msg.sender`) and emit `TransferOutAndCall` so Bifrost can observe the outbound completion. However, the recovery logic breaks down due to three interacting mechanics:

1. **Gas-stipend-dependent payment failure (`payable(to).send`)**:
   Line 278 uses `.send(_safeAmount)`, which restricts the callee to the hardcoded 2,300-gas stipend. If `to` is a contract wallet (e.g., Gnosis Safe, proxy, or any contract whose `receive`/`fallback` performs state reads/writes or emits events), the 2,300-gas stipend is insufficient, causing `ethSuccess` to evaluate to `false` even when the recipient is otherwise valid and payable.

2. **Callee-induced recovery gas exhaustion**:
   Line 269 invokes `aggregator.call{value: _safeAmount}(...)` without reserving gas for the fallback path. Under EIP-150, up to 63/64 of the remaining gas is forwarded to `aggregator`. If the swap fails by exhausting all forwarded gas (for example, an out-of-gas condition in the DEX route or a malicious/broken token or callback consuming all gas in the subcall), only 1/64 of the gas remains in `THORChain_Router`.
   Because the subcall reverted, `to` remains cold in the EIP-2929 access list. Executing `payable(to).send(_safeAmount)` requires at least `2,600` (cold address access) + `9,000` (non-zero value transfer) = `11,600` gas, and if that fails, `payable(address(msg.sender)).transfer(_safeAmount)` requires another `9,000`+ gas, followed by `emit TransferOutAndCall(...)`. With typical aggregator gas limits (e.g., 400,000 gas, where 1/64 is ~6,250 gas), the router runs out of gas at line 278 or line 280 and the entire transaction reverts.

3. **Refund-path failure propagation (`payable(address(msg.sender)).transfer`)**:
   Line 280 uses `.transfer(_safeAmount)`, which also enforces the 2,300-gas stipend and bubbles up a revert if the transfer fails (either because `msg.sender` is a contract requiring >2,300 gas on `receive` or because the remaining gas is insufficient for the value transfer).

Additionally, even when the fallback bounces ETH back to `msg.sender` (line 280) instead of delivering it to `to`, line 284 still emits `TransferOutAndCall` with `to` as the recipient, making the event indistinguishable from a successful swap or direct fallback to `to`.

### Assumptions and preconditions

1. A THORChain outbound transaction is scheduled using `transferOutAndCall` with native ETH (`msg.value > 0`).
2. Either:
   - The call to `aggregator.swapOut` consumes all forwarded gas (63/64 of available gas) before failing, leaving less than ~23,000 gas for the fallback transfers and event emission; or
   - The swap reverts (e.g., due to slippage `amountOutMin` not met) and `to` (and/or `msg.sender`) is a contract whose `receive()`/`fallback()` requires more than 2,300 gas.

### Proof of concept

1. A user initiates a cross-chain swap to an Ethereum target token via a whitelisted aggregator with `amountOutMin` set, specifying a smart contract wallet `to` (or a token/recipient route that runs out of gas on failure).
2. Bifrost constructs the outbound transaction calling `THORChain_Router.transferOutAndCall(aggregator, finalToken, to, amountOutMin, memo)` with `_safeAmount = msg.value` and a bounded gas limit (`maxGasUnits` / `gasLimitForAggregator`, e.g., 400,000 gas).
3. During execution, `aggregator.call{value: _safeAmount}(...)` fails after consuming all 63/64 forwarded gas (~393,750 gas), leaving ~6,250 gas in `THORChain_Router`.
4. Execution enters `if (!erc20Success)` at line 277 and attempts `payable(to).send(_safeAmount)` at line 278.
5. Because transferring non-zero ETH to a cold address `to` costs at least 11,600 gas (`2,600` cold access + `9,000` value transfer), the call immediately runs out of gas and reverts the entire `transferOutAndCall` transaction (instead of gracefully falling back and emitting `TransferOutAndCall`).
6. Alternatively, if `aggregator` reverts early with gas remaining, `payable(to).send(_safeAmount)` still fails for any smart contract wallet `to` that uses >2,300 gas in `receive()`, forcing the ETH to bounce back to `msg.sender` at line 280 while still emitting `TransferOutAndCall(msg.sender, aggregator, _safeAmount, finalToken, to, amountOutMin, memo)` as if the outbound to `to` completed.

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**:
  1. When `aggregator.call` fails by exhausting its 63/64 gas allowance, the fallback path in `transferOutAndCall` runs out of gas and reverts the entire transaction, defeating the fallback mechanism.
  2. Smart contract recipients (`to`) with `receive()` logic exceeding 2,300 gas cannot receive fallback ETH via `.send()`, causing their funds to be bounced back to the vault while `TransferOutAndCall` is still emitted as if the outbound succeeded, leading to loss of user funds (the vault retains the ETH while THORChain marks the outbound as completed).
  3. If the refund `.transfer()` at line 280 reverts, the outbound transaction fails on-chain, wasting vault gas and causing outbound queue liveness issues until manual intervention or rescheduling.

### Remediation

Reserve sufficient gas for the fallback path before calling the external aggregator, and replace `.send()` / `.transfer()` with bounded-gas `.call{value: _safeAmount, gas: ...}("")` so smart contract recipients can receive ETH without exhausting the gas needed for the vault bounce-back and event emission.

```solidity
  function transferOutAndCall(
    address payable aggregator,
    address finalToken,
    address to,
    uint256 amountOutMin,
    string memory memo
  ) public payable nonReentrant {
    uint256 _safeAmount = msg.value;
    uint256 fallbackGasReserve = 50000;
    require(gasleft() > fallbackGasReserve, "Insufficient gas for fallback");
    (bool erc20Success, ) = aggregator.call{
      value: _safeAmount,
      gas: gasleft() - fallbackGasReserve
    }(
      abi.encodeWithSignature(
        "swapOut(address,address,uint256)",
        finalToken,
        to,
        amountOutMin
      )
    );
    if (!erc20Success) {
      (bool ethSuccess, ) = payable(to).call{value: _safeAmount, gas: 20000}(""); // If can't swap, just send the recipient the ETH
      if (!ethSuccess) {
        (bool bounceSuccess, ) = payable(address(msg.sender)).call{value: _safeAmount}(""); // For failure, bounce back to vault & continue.
        require(bounceSuccess, "Failed to bounce ETH to vault");
      }
    }

    emit TransferOutAndCall(
      msg.sender,
      aggregator,
      _safeAmount,
      finalToken,
      to,
      amountOutMin,
      memo
    );
  }
```

## Finding 13: Incorrect Use of `msg.value` Instead of `fromAmount` in `THORChain_Router._transferOutAndCallV5` Breaks Native Asset Accounting and Batch Outbounds

### Synopsis

`THORChain_Router._transferOutAndCallV5` uses `msg.value` instead of `aggregationPayload.fromAmount` when forwarding ETH, handling fallback transfers, and emitting `TransferOutAndCallV5`. This breaks batch ETH outbounds in `batchTransferOutAndCallV5` and causes mismatches between the declared swap amount and actual native asset value transferred.

### Technical details

In `THORChain_Router._transferOutAndCallV5`, each outbound operation is described by a `TransferOutAndCallData` struct containing `fromAmount`, which specifies the exact amount of `fromAsset` allocated to that individual operation.

When `aggregationPayload.fromAsset == address(0)` (indicating native ETH), the function passes `aggregationPayload.fromAmount` as an argument in the calldata to `swapOutV5` (line 315), but uses the transaction-wide `msg.value` for all actual ETH transfers and event accounting:

1. **Aggregator call (lines 309–311):** `aggregationPayload.target.call{value: msg.value}(...)` forwards the entire transaction `msg.value` rather than `aggregationPayload.fromAmount`.
2. **Swap failure fallback (line 324):** `payable(aggregationPayload.target).send(msg.value)` attempts to send `msg.value` instead of `aggregationPayload.fromAmount`.
3. **Vault bounce-back fallback (line 326):** `payable(address(msg.sender)).transfer(msg.value)` attempts to refund `msg.value` instead of `aggregationPayload.fromAmount`.
4. **Event emission (lines 330–333):** `emit TransferOutAndCallV5(..., msg.value, ...)` logs `msg.value` as the outbound amount rather than `aggregationPayload.fromAmount`.

When `batchTransferOutAndCallV5` is invoked with multiple payloads where more than one payload uses native ETH (`fromAsset == address(0)`), the caller supplies the sum of all native `fromAmount` values as `msg.value`. On the first native ETH payload, `_transferOutAndCallV5` sends the entire `msg.value` to the target aggregator in a single call. Consequently:
- The first outbound operation receives the total ETH of the entire batch instead of its designated `fromAmount`.
- Every subsequent ETH outbound operation in the batch attempts to forward `msg.value` again from the Router's remaining balance, causing `swapOutV5` and the fallback `.send(msg.value)` and `.transfer(msg.value)` on line 326 to fail and revert the entire batch transaction.
- Even in a single operation or if the Router holds pre-existing ETH, any discrepancy between `msg.value` and `aggregationPayload.fromAmount` results in inconsistent accounting between the calldata parameter (`fromAmount`), the transferred ETH (`msg.value`), and the emitted `TransferOutAndCallV5` event.

### Assumptions and preconditions

- A vault calls `batchTransferOutAndCallV5` with a batch containing at least two native ETH outbound payloads (`fromAsset == address(0)`), sending total ETH equal to the sum of the payloads' `fromAmount` fields (`msg.value = fromAmount_1 + fromAmount_2 + ...`).
- Alternatively, `transferOutAndCallV5` or `batchTransferOutAndCallV5` is called where `msg.value` differs from a payload's `aggregationPayload.fromAmount`.

### Proof of concept

1. An Asgard vault constructs a `batchTransferOutAndCallV5` transaction with two ETH aggregation payloads:
   - Payload 0: `fromAsset = address(0)`, `fromAmount = 1 ether`, `recipient = UserA`.
   - Payload 1: `fromAsset = address(0)`, `fromAmount = 2 ether`, `recipient = UserB`.
2. The vault submits the transaction with `msg.value = 3 ether` (the sum of `1 ether + 2 ether`).
3. During iteration `i = 0`, `_transferOutAndCallV5(aggregationPayloads[0])` executes:
   - Line 309 calls `target.call{value: 3 ether}(swapOutV5(..., fromAmount = 1 ether, ...))`.
   - `THORChain_Aggregator.swapOutV5` swaps the full `msg.value` (`3 ether`) via `swapRouter.swapExactETHForTokens{value: msg.value}` for `UserA`, overpaying `UserA` by `2 ether`.
   - Line 330 emits `TransferOutAndCallV5` logging `3 ether` for Payload 0.
4. During iteration `i = 1`, `_transferOutAndCallV5(aggregationPayloads[1])` executes:
   - The Router now has `0 ether` balance remaining from the transaction.
   - Line 309 attempts `target.call{value: 3 ether}(...)`, which fails (`swapOutSuccess = false`) due to insufficient ETH balance in the Router.
   - Line 324 attempts `payable(target).send(3 ether)`, which also fails (`sendSuccess = false`).
   - Line 326 attempts `payable(msg.sender).transfer(3 ether)`, which reverts the entire transaction because the Router lacks `3 ether`.

### Impact assessment

- **Severity:** High
- **Technical Impact:**
  - `batchTransferOutAndCallV5` cannot process batches with multiple native ETH swaps: the first ETH payload consumes the entire batch `msg.value` (overpaying the first recipient if the transaction somehow succeeds, e.g., if the Router had residual ETH), and subsequent ETH payloads revert the entire batch on line 326.
  - Emitted `TransferOutAndCallV5` events report `msg.value` for every ETH payload in a batch instead of each payload's `fromAmount`, corrupting Bifrost outbound accounting.

### Remediation

Use `aggregationPayload.fromAmount` instead of `msg.value` for the external call value, fallback transfers, and event emission in `_transferOutAndCallV5`, and optionally track total native value spent across a batch to ensure conservation of `msg.value`.

```solidity
  function _transferOutAndCallV5(
    TransferOutAndCallData calldata aggregationPayload
  ) private {
    if (aggregationPayload.fromAsset == address(0)) {
      // call swapOutV5 with ether
      (bool swapOutSuccess, ) = aggregationPayload.target.call{
        value: aggregationPayload.fromAmount
      }(
        abi.encodeWithSignature(
          "swapOutV5(address,uint256,address,address,uint256,bytes,string)",
          aggregationPayload.fromAsset,
          aggregationPayload.fromAmount,
          aggregationPayload.toAsset,
          aggregationPayload.recipient,
          aggregationPayload.amountOutMin,
          aggregationPayload.payload,
          aggregationPayload.originAddress
        )
      );
      if (!swapOutSuccess) {
        bool sendSuccess = payable(aggregationPayload.recipient).send(
          aggregationPayload.fromAmount
        ); // If can't swap, just send the recipient the gas asset
        if (!sendSuccess) {
          payable(address(msg.sender)).transfer(aggregationPayload.fromAmount); // For failure, bounce back to vault & continue.
        }
      }

      emit TransferOutAndCallV5(
        msg.sender,
        aggregationPayload.target,
        aggregationPayload.fromAmount,
        aggregationPayload.toAsset,
        aggregationPayload.recipient,
        aggregationPayload.amountOutMin,
        aggregationPayload.memo,
        aggregationPayload.payload,
        aggregationPayload.originAddress
      );
    } else {
```

## Finding 14: Fallback Beneficiary Confusion in `THORChain_Router._transferOutAndCallV5` Sends Failed Swap ETH to the Aggregator Instead of the Recipient

### Synopsis

In `THORChain_Router._transferOutAndCallV5`, when an external `swapOutV5` call with native ETH fails, the fallback recovery logic mistakenly transfers `msg.value` to `aggregationPayload.target` (the aggregator contract) instead of `aggregationPayload.recipient`, resulting in permanent loss of user funds if the aggregator accepts native ETH transfers.

### Technical details

When `THORChain_Router._transferOutAndCallV5` processes a native gas asset outbound (`aggregationPayload.fromAsset == address(0)`), it attempts to execute `swapOutV5` on the aggregator contract (`aggregationPayload.target`) while forwarding `msg.value` (lines 309–322).

According to the protocol specification and the previous `transferOutAndCall` implementation (line 278), if the external swap call to the aggregator reverts (for instance, due to slippage breaching `amountOutMin`), the router is intended to fall back by sending the native ETH directly to the user (`aggregationPayload.recipient`), and only bounce the funds back to the vault (`msg.sender`) if sending to the recipient also fails.

However, at line 324, the fallback transfer specifies `aggregationPayload.target` instead of `aggregationPayload.recipient`:

```solidity
if (!swapOutSuccess) {
  bool sendSuccess = payable(aggregationPayload.target).send(msg.value); // If can't swap, just send the recipient the gas asset
  if (!sendSuccess) {
    payable(address(msg.sender)).transfer(msg.value); // For failure, bounce back to vault & continue.
  }
}
```

Because DEX aggregator contracts typically implement a `receive()` or `fallback()` payable function to accept native ETH from DEX pools or WETH unwrapping, `payable(aggregationPayload.target).send(msg.value)` will succeed if it executes within the 2300 gas stipend. Consequently, the user's ETH is sent directly to the aggregator contract without executing any state updates or swap logic, permanently locking or exposing the funds inside the aggregator contract rather than delivering them to `aggregationPayload.recipient` or returning them to the THORChain vault.

### Assumptions and preconditions

1. A vault calls `transferOutAndCallV5` or `batchTransferOutAndCallV5` with `aggregationPayload.fromAsset == address(0)` and `msg.value > 0`.
2. The `swapOutV5` call on `aggregationPayload.target` reverts (e.g., due to price slippage exceeding `amountOutMin`, insufficient liquidity, or an invalid payload).
3. The `aggregationPayload.target` contract has a payable `receive()` or `fallback()` function that consumes less than 2300 gas.

### Proof of concept

1. A user initiates a cross-chain swap through THORChain expecting an outbound swap via `THORChain_Router.transferOutAndCallV5` with `fromAsset = address(0)` (ETH), `msg.value = 1 ether`, `target = AGGREGATOR`, and `recipient = USER`.
2. While the outbound transaction is in flight, market prices move such that the output token amount on the target DEX is less than `aggregationPayload.amountOutMin`.
3. The vault executes `transferOutAndCallV5`. Inside `_transferOutAndCallV5`, the low-level call `aggregationPayload.target.call{value: 1 ether}(abi.encodeWithSignature("swapOutV5(...)", ...))` reverts due to the slippage check in the underlying DEX router, setting `swapOutSuccess = false`.
4. Execution enters the `if (!swapOutSuccess)` branch at line 323.
5. At line 324, `payable(aggregationPayload.target).send(1 ether)` is executed instead of sending to `USER`.
6. Because `AGGREGATOR` implements a standard `receive() external payable {}` function to accept ETH from swaps, `sendSuccess` evaluates to `true`.
7. `TransferOutAndCallV5` is emitted indicating the outbound completed, the vault considers the outbound fulfilled, `USER` receives neither tokens nor ETH, and `1 ether` remains stuck in the `AGGREGATOR` contract.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Any failed native ETH swap through `_transferOutAndCallV5` transfers the ETH to the aggregator contract (`target`) instead of the intended `recipient`.
- **Business Impact**: Direct and permanent loss of user funds on every failed ETH swap outbound where the target aggregator accepts native ETH via `receive()`/`fallback()`. Any third-party caller may also be able to sweep unallocated ETH balances left inside stateless aggregator contracts.

### Remediation

Update line 324 in `THORChain_Router._transferOutAndCallV5` to send the fallback ETH to `aggregationPayload.recipient` instead of `aggregationPayload.target`.

```solidity
      if (!swapOutSuccess) {
        bool sendSuccess = payable(aggregationPayload.recipient).send(msg.value); // If can't swap, just send the recipient the gas asset
        if (!sendSuccess) {
          payable(address(msg.sender)).transfer(msg.value); // For failure, bounce back to vault & continue.
        }
      }
```
