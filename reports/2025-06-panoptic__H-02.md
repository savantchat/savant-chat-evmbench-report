# Audit report — 2025-06-panoptic/H-02

## Finding 1: Unsettled and Uncollectible Short Premium Included in `PanopticVaultAccountant.computeNAV`

### Synopsis

`PanopticVaultAccountant.computeNAV` calls `PanopticPool.getAccumulatedFeesAndPositionsData` with `includePendingPremium = true`. This causes the vault's Net Asset Value (NAV) to include gross accrued short premia rather than only settled, available premia, inflating NAV with uncollectible or haircutted premia and enabling share price manipulation during deposits and withdrawals.

### Technical details

In `PanopticVaultAccountant.sol`, `computeNAV` queries the vault's accumulated option premia and position balances for each configured `PanopticPool`:

```solidity
(shortPremium, longPremium, positionBalanceArray) = pools[i]
    .pool
    .getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i]);
```

The second argument to `PanopticPool.getAccumulatedFeesAndPositionsData` is `includePendingPremium`, which is forwarded to `PanopticPool._calculateAccumulatedPremia`:

- When `includePendingPremium == false` (which `PanopticPool` itself uses via `ONLY_AVAILABLE_PREMIUM` during solvency checks in `_checkSolvencyAtTicks`), short leg premia (`tokenId.isLong(leg) == 0`) are passed through `_getAvailablePremium`. This function prorates the gross `premiaByLeg[leg]` by the ratio of actually settled tokens (`s_settledTokens[chunkKey]`) to the total gross premium accumulated on that chunk (`((premiumAccumulators - grossPremiumLast) * totalLiquidity) / 2 ** 64`), capped at 100% of `premiumOwed`.
- When `includePendingPremium == true`, `_calculateAccumulatedPremia` bypasses `_getAvailablePremium` and adds the raw gross `premiaByLeg[leg]` (computed from `SFPM.getAccountPremium`'s `s_accountPremiumGross` accumulator) directly to `shortPremium`.

Including pending (unsettled) short premium in `computeNAV` introduces a critical accounting divergence between reported NAV and realizable vault assets:

1. **Uncollectible / Haircutted Premia Recognition:** Long option buyers only settle premia when they close/burn their position, are liquidated, or have `settleLongPremium` called against them. If a buyer becomes deeply insolvent and is liquidated, `PanopticMath.haircutPremia` socialization haircuts the unpaid premium against `s_settledTokens`, meaning sellers will never receive those tokens. However, the SFPM's `s_accountPremiumGross` accumulator is never decremented upon a liquidation haircut. Consequently, calling `getAccumulatedFeesAndPositionsData` with `includePendingPremium = true` permanently counts haircutted, uncollectible premia as part of the vault's NAV until the vault closes its short position.
2. **Gross Accrual vs. Available Entitlement Divergence:** Even when buyers are solvent, a short option seller closing a position in `PanopticPool` can only collect the currently available settled premium (`_getAvailablePremium`), forfeiting any unsettled pending premium. If the vault closes a short position to service withdrawals or rebalance before all buyers settle, the pending premium counted in `computeNAV` cannot be realized.
3. **Value Extraction Across Settlement Boundaries:** Because `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals` rely on `accountant.computeNAV` to price shares, an inflated NAV caused by unsettled or haircutted short premia allows withdrawing users to redeem shares for more underlying assets than the vault can actually collect (leaving remaining shareholders with the shortfall when the vault closes its short position or after a counterparty liquidation), while diluting new depositors who receive fewer shares than warranted.

### Assumptions and preconditions

1. The `HypoVault` holds at least one open short option leg (`isLong == 0`) in a `PanopticPool` tracked by `PanopticVaultAccountant`.
2. A counterparty holds a corresponding long option leg in the same chunk and accrues owed premium over time without settling it (or is liquidated with a premium haircut).
3. The vault manager fulfills deposits (`fulfillDeposits`) or withdrawals (`fulfillWithdrawals`) while the unsettled or haircutted premium is present on the vault's short position.

### Proof of concept

1. **Initial State:** `HypoVault` has 1,000,000 shares and 1,000e18 `underlyingToken` in assets. User A and User B each own 50% of the shares (500,000 shares each).
2. **Short Position & Unsettled Accrual:** The vault sells a short option position in `PanopticPool` on a liquidity chunk. A buyer opens a long option position on the same chunk. Over time, 200e18 worth of gross short premium accrues to the vault's short leg, but the buyer has not settled their long premium (`s_settledTokens` for this premium is 0), or the buyer becomes insolvent and is liquidated such that the 200e18 premium is haircutted and never added to `s_settledTokens`.
3. **Withdrawal Fulfillment at Inflated NAV:** User A requests a withdrawal of all 500,000 shares. The manager calls `fulfillWithdrawals`.
4. **Mispricing Calculation:** `PanopticVaultAccountant.computeNAV` calls `getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i])`, which includes the 200e18 unsettled/haircutted short premium. `computeNAV` reports 1,200e18 instead of the true realizable NAV of 1,000e18.
5. **Loss to Remaining Users:** User A's 50% share withdrawal is fulfilled for `500,000 * 1,200e18 / 1,000,000 = 600e18` underlying assets. When the vault closes its short position in `PanopticPool`, it receives 0 premium (because `s_settledTokens` is 0), leaving only 400e18 assets for User B (a 100e18 loss to User B and a 100e18 windfall to User A).

### Impact assessment

- **Severity:** High / Medium.
- **Technical Effect:** `PanopticVaultAccountant.computeNAV` overstates the vault's net asset value by the full amount of unsettled or haircutted short premia.
- **Business Impact:** Exiting shareholders can extract unrealized or permanently uncollectible option premia at the expense of remaining shareholders, and new depositors are diluted by receiving fewer shares for their deposited assets. Furthermore, higher reported withdrawal payouts inflate the `performanceFee` charged to withdrawing users in `HypoVault.executeWithdrawal`.

### Remediation

Pass `false` for the `includePendingPremium` parameter when calling `getAccumulatedFeesAndPositionsData` in `PanopticVaultAccountant.computeNAV` so that only settled, available short premia are included in the vault's NAV calculation (or alternatively, ensures `settleLongPremium` is called on solvent buyers prior to `computeNAV` if pending premia are to be realized):

```solidity
                (shortPremium, longPremium, positionBalanceArray) = pools[i]
                    .pool
                    .getAccumulatedFeesAndPositionsData(_vault, false, tokenIds[i]);
```

## Finding 2: Inverted `shortPremium` and `longPremium` subtraction in `PanopticVaultAccountant.computeNAV` miscalculates `token1` pool exposure

### Synopsis

In `PanopticVaultAccountant.computeNAV`, `poolExposure1` is initialized by subtracting `shortPremium.leftSlot()` from `longPremium.leftSlot()`, which is the exact opposite of how option premia affect Net Asset Value (NAV). This treats `token1` premium liabilities as assets and `token1` premium assets as liabilities, corrupting vault share pricing.

### Technical details

When computing the Net Asset Value (NAV) of a `HypoVault` in `PanopticVaultAccountant.computeNAV`, the accountant queries each `PanopticPool` via `getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i])` (lines 127–129).

According to `PanopticPool.getAccumulatedFeesAndPositionsData` and `PanopticPool._calculateAccumulatedPremia`:
- The first return value, `shortPremium` (`LeftRightUnsigned`), represents the total premium owed **to** the user's short legs (an asset/receivable for the vault), with `currency0` (`token0`) in the `rightSlot()` and `currency1` (`token1`) in the `leftSlot()`.
- The second return value, `longPremium` (`LeftRightUnsigned`), represents the total premium owed **by** the user's long legs (a liability/payable for the vault), with `currency0` (`token0`) in the `rightSlot()` and `currency1` (`token1`) in the `leftSlot()`.

For `token0` (`poolExposure0`), lines 131–133 properly add the receivable `shortPremium` and subtract the payable `longPremium`:
```solidity
poolExposure0 =
    int256(uint256(shortPremium.rightSlot())) -
    int256(uint256(longPremium.rightSlot()));
```

However, for `token1` (`poolExposure1`), lines 134–136 invert the operands:
```solidity
poolExposure1 =
    int256(uint256(longPremium.leftSlot())) -
    int256(uint256(shortPremium.leftSlot()));
```

Because `longPremium.leftSlot()` is added and `shortPremium.leftSlot()` is subtracted:
1. Any `token1` premium earned by the vault on short positions reduces `poolExposure1` instead of increasing it.
2. Any `token1` premium owed by the vault on long positions increases `poolExposure1` instead of reducing it.

This corrupted `poolExposure1` flows directly into `nav` at line 250 (`nav += uint256(Math.max(poolExposure0 + poolExposure1, 0))`), resulting in an overstated or understated NAV during `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals`.

### Assumptions and preconditions

1. A `HypoVault` is configured to use `PanopticVaultAccountant` as its `accountant`.
2. The vault holds one or more active option positions in a `PanopticPool` that have accrued non-zero `token1` (`leftSlot`) short or long premia where `shortPremium.leftSlot() != longPremium.leftSlot()`.

### Proof of concept

1. Suppose `HypoVault` has `underlyingToken = token1` (e.g., USDC), holds 1,000 USDC in collateral/idle balances, and has an open option position in `PanopticPool`.
2. Over time, the vault's short leg earns `100e6` USDC (`shortPremium.leftSlot() = 100e6`) and its long leg owes `0` USDC (`longPremium.leftSlot() = 0`).
3. The true net premia value for `token1` is `+100e6` USDC, so the vault's NAV should reflect `+100e6` USDC from accumulated premia.
4. When the manager calls `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals`, `PanopticVaultAccountant.computeNAV` executes lines 134–136:
   ```solidity
   poolExposure1 = int256(uint256(0)) - int256(uint256(100e6)) = -100e6;
   ```
5. Instead of adding `100e6` USDC to `poolExposure1`, `100e6` USDC is subtracted, undervaluing the vault's NAV by `200e6` USDC (`2 * (shortPremium.leftSlot() - longPremium.leftSlot())`).
6. Conversely, if the vault has a long leg that owes `100e6` USDC (`longPremium.leftSlot() = 100e6`, `shortPremium.leftSlot() = 0`), `poolExposure1` starts at `+100e6` instead of `-100e6`, overvaluing the vault's NAV by `200e6` USDC.

### Impact assessment

**Severity: High**

- **Direct Technical Effect:** `computeNAV` miscalculates `poolExposure1` by `2 * (longPremium.leftSlot() - shortPremium.leftSlot())`.
- **Business Impact:**
  - **Understated NAV (when `shortPremium.leftSlot() > longPremium.leftSlot()`):** New depositors calling `requestDeposit` receive more shares than they should when `fulfillDeposits` is executed, diluting existing shareholders, while withdrawing users receive fewer assets than their fair share.
  - **Overstated NAV (when `longPremium.leftSlot() > shortPremium.leftSlot()`):** Withdrawing users receive excess assets at the expense of remaining shareholders, inflated performance fees may be charged on phantom profits in `executeWithdrawal`, and new depositors receive fewer shares than their deposit is worth.

### Remediation

Update `poolExposure1` in `PanopticVaultAccountant.computeNAV` so that `longPremium.leftSlot()` is subtracted from `shortPremium.leftSlot()`, matching the sign convention used for `poolExposure0`.

```solidity
                poolExposure0 =
                    int256(uint256(shortPremium.rightSlot())) -
                    int256(uint256(longPremium.rightSlot()));
                poolExposure1 =
                    int256(uint256(shortPremium.leftSlot())) -
                    int256(uint256(longPremium.leftSlot()));
```

## Finding 3: Missing Duplicate Check on `tokenIds` in `PanopticVaultAccountant.computeNAV` Allows Portfolio Misrepresentation and NAV Manipulation

### Synopsis

In `PanopticVaultAccountant.computeNAV`, the completeness of the manager-supplied `tokenIds` array for each pool is validated solely by comparing the sum of legs (`numLegs`) against `pools[i].pool.numberOfLegs(_vault)`. Because duplicate `TokenId` entries are not rejected, a vault manager can repeat one open position and omit another to manipulate the reported NAV.

### Technical details

When `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals` is called, the vault queries `PanopticVaultAccountant.computeNAV` using `managerInput`, which decodes into `(ManagerPrices[] memory managerPrices, PoolInfo[] memory pools, TokenId[][] memory tokenIds)`.

For each pool `i`, `computeNAV` queries `pools[i].pool.getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i])` and iterates over `tokenIds[i]` (lines 140–173):
1. It verifies that each supplied `TokenId` represents an active position held by the vault via `if (positionBalanceArray[j][1] == 0) revert IncorrectPositionList();` (line 141).
2. It accumulates the net liquidity exposure and exercised amounts for `tokenIds[i][j]` into `poolExposure0` and `poolExposure1`, and increments `numLegs` by `tokenIds[i][j].countLegs()` (lines 142–172).
3. After the loop, it checks that the total leg count matches the pool's recorded leg count for the vault:
   ```solidity
   if (numLegs != pools[i].pool.numberOfLegs(_vault)) revert IncorrectPositionList();
   ```
   (line 175).

However, `computeNAV` never enforces that the entries in `tokenIds[i]` are unique. Because `getAccumulatedFeesAndPositionsData` simply loops over the calldata array `positionIdList` and reads `s_positionBalance[_vault][tokenId]` for each element independently, passing the same active `TokenId` multiple times succeeds and returns non-zero balances and premia for each duplicate entry.

Consequently, whenever the vault holds at least two open positions in a pool whose leg counts can be substituted (for example, two 1-leg positions $A$ and $B$, or one 2-leg position $C$ and one 1-leg position $A$ where $[A, A]$ replaces $[C]$), the manager can supply duplicate entries (e.g., `[A, A]` instead of `[A, B]`). Both `positionBalanceArray[j][1] != 0` and `numLegs == pools[i].pool.numberOfLegs(_vault)` pass, causing `computeNAV` to double-count position $A$ (including its accumulated premia in `getAccumulatedFeesAndPositionsData`, its liquidity chunks, and its exercised amounts) while completely ignoring position $B$.

### Assumptions and preconditions

1. The `HypoVault` has at least two active option positions in a `PanopticPool` (e.g., Position $A$ and Position $B$ with equal leg counts, or positions whose leg counts allow a linear combination of one subset to equal the total leg count of all open positions).
2. Position $A$ and Position $B$ have different net valuations (exposure + premia).
3. The vault manager invokes `fulfillDeposits` or `fulfillWithdrawals` with crafted `managerInput`.

### Proof of concept

Consider a `HypoVault` with `underlyingToken` = Token1 (e.g., USDC) that holds two 1-leg positions in `pools[0]`:
- **Position A**: A profitable 1-leg long option currently worth `+50e18` in net exposure.
- **Position B**: An out-of-the-money or losing 1-leg short option currently representing `-30e18` in net exposure.
- On-chain in `pools[0].pool`, `numberOfLegs(vault)` is `2`.

1. **Legitimate NAV Calculation**:
   - If the manager passes `tokenIds[0] = [Position A, Position B]`, the combined position exposure is `+50e18 - 30e18 = +20e18`, and `numLegs = 1 + 1 = 2`.
2. **Exploit Execution**:
   - During `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals`, the manager passes `tokenIds[0] = [Position A, Position A]`.
   - In `getAccumulatedFeesAndPositionsData`, both entries look up `s_positionBalance[vault][Position A]`, returning a non-zero balance for `j = 0` and `j = 1` and summing Position A's premia twice.
   - In the loop at lines 140–173 of `PanopticVaultAccountant.sol`:
     - For `j = 0`: `positionBalanceArray[0][1] != 0` passes; Position A's exposure (`+50e18`) is added; `numLegs` becomes `1`.
     - For `j = 1`: `positionBalanceArray[1][1] != 0` passes; Position A's exposure (`+50e18`) is added again; `numLegs` becomes `2`.
   - At line 175, `numLegs (2) != pools[0].pool.numberOfLegs(vault) (2)` is `false`, so the check passes without reverting.
3. **Result**:
   - The reported exposure from positions is `+100e18` instead of `+20e18` (or if `[Position B, Position B]` is passed, `-60e18` instead of `+20e18`), distorting the NAV used to mint shares or distribute withdrawal assets.

### Impact assessment

- **Severity**: High.
- **Technical Impact**: `PanopticVaultAccountant.computeNAV` returns an artificially inflated or deflated Net Asset Value for the vault.
- **Business Impact**:
  - **Inflated NAV**: During `fulfillDeposits`, depositors receive fewer shares than they are entitled to (diluting new depositors and unjustly enriching existing shareholders / performance fee recipients via an inflated share price). During `fulfillWithdrawals`, withdrawing users receive more underlying assets per share than the vault truly backs, draining the vault at the expense of remaining shareholders.
  - **Deflated NAV**: During `fulfillDeposits`, depositors are minted an excessive number of shares at a discounted price, diluting existing vault shareholders. During `fulfillWithdrawals`, withdrawing users receive fewer assets than their fair share.

### Remediation

Enforce that all `TokenId` entries in `tokenIds[i]` for a given pool are strictly unique by checking for duplicates in the `tokenIds[i]` loop.

```solidity
            uint256 numLegs;
            for (uint256 j = 0; j < tokenIds[i].length; j++) {
                for (uint256 k = j + 1; k < tokenIds[i].length; k++) {
                    if (TokenId.unwrap(tokenIds[i][j]) == TokenId.unwrap(tokenIds[i][k])) {
                        revert IncorrectPositionList();
                    }
                }
                if (positionBalanceArray[j][1] == 0) revert IncorrectPositionList();
                uint256 positionLegs = tokenIds[i][j].countLegs();
                for (uint256 k = 0; k < positionLegs; k++) {
                    (uint256 amount0, uint256 amount1) = Math.getAmountsForLiquidity(
                        managerPrices[i].poolPrice,
                        PanopticMath.getLiquidityChunk(
                            tokenIds[i][j],
                            k,
                            uint128(positionBalanceArray[j][1])
                        )
                    );

                    if (tokenIds[i][j].isLong(k) == 0) {
                        unchecked {
                            poolExposure0 += int256(amount0);
                            poolExposure1 += int256(amount1);
                        }
                    } else {
                        unchecked {
                            poolExposure0 -= int256(amount0);
                            poolExposure1 -= int256(amount1);
                        }
                    }
                }

                (LeftRightSigned longAmounts, LeftRightSigned shortAmounts) = PanopticMath
                    .computeExercisedAmounts(tokenIds[i][j], uint128(positionBalanceArray[j][1]));

                poolExposure0 += int256(longAmounts.rightSlot()) - int256(shortAmounts.rightSlot());
                poolExposure1 += int256(longAmounts.leftSlot()) - int256(shortAmounts.leftSlot());

                numLegs += positionLegs;
            }

            if (numLegs != pools[i].pool.numberOfLegs(_vault)) revert IncorrectPositionList();
```

## Finding 4: Inverted Exercised Notional Sign Convention in `PanopticVaultAccountant.computeNAV` Distorts Vault NAV

### Synopsis

In `PanopticVaultAccountant.computeNAV`, the exercised notional amounts (`longAmounts` and `shortAmounts`) returned by `PanopticMath.computeExercisedAmounts` are applied with inverted signs when updating `poolExposure0` and `poolExposure1`. This causes open long and short option positions to be misvalued, distorting share pricing during deposit and withdrawal fulfillment in `HypoVault`.

### Technical details

`PanopticVaultAccountant.computeNAV` calculates the Net Asset Value (NAV) of a `HypoVault` across its configured `PanopticPool` instances. For each open position `tokenIds[i][j]` held by the vault:

1. Lines 143–164 iterate through each leg `k` of the position, compute the current token amounts `(amount0, amount1)` represented by the leg's liquidity chunk via `Math.getAmountsForLiquidity`, and update `poolExposure0` and `poolExposure1`:
   - For short legs (`tokenIds[i][j].isLong(k) == 0`), `amount0` and `amount1` are added (`+=`).
   - For long legs (`tokenIds[i][j].isLong(k) == 1`), `amount0` and `amount1` are subtracted (`-=`).

2. Lines 166–167 call `PanopticMath.computeExercisedAmounts(tokenIds[i][j], uint128(positionBalanceArray[j][1]))`, which iterates through all legs and calls `_calculateIOAmounts`. Both `longAmounts` and `shortAmounts` are returned as positive `int128` values packed into `LeftRightSigned` words (`rightSlot()` for `token0` and `leftSlot()` for `token1`).

3. At lines 169–170, `computeNAV` adjusts `poolExposure0` and `poolExposure1` using:
   ```solidity
   poolExposure0 += int256(longAmounts.rightSlot()) - int256(shortAmounts.rightSlot());
   poolExposure1 += int256(longAmounts.leftSlot()) - int256(shortAmounts.leftSlot());
   ```

In `CollateralTracker.takeCommissionAddData` (line 1085) and `CollateralTracker.exercise` (lines 1111–1113 and 1133), the net AMM exposure delta from exercised notional amounts is tracked as `shortAmount - longAmount` (`tokenToPay = int256(swappedAmount) - (longAmount - shortAmount) - realizedPremium`, i.e. `swappedAmount + shortAmount - longAmount - realizedPremium`). When `PanopticVaultAccountant.computeNAV` applies `longAmounts - shortAmounts` (instead of `shortAmounts - longAmounts`, or with mismatched asset-liability sign convention relative to the liquidity chunk and collateral settlement accounting), the exercised notional delta diverges from the settlement accounting, overstating or understating `poolExposure0` and `poolExposure1` by `2 * (longAmounts - shortAmounts)`.

Because `HypoVault.fulfillDeposits` (line 480) and `HypoVault.fulfillWithdrawals` (line 522) rely directly on `accountant.computeNAV` to determine `totalAssets`, any miscalculation in `poolExposure0` and `poolExposure1` directly corrupts the exchange rate between vault shares and underlying assets.

### Assumptions and preconditions

1. The `HypoVault` uses `PanopticVaultAccountant` as its configured `accountant`.
2. The vault holds at least one open option position (long or short leg) in a `PanopticPool` when `computeNAV` is called during `fulfillDeposits` or `fulfillWithdrawals`.

### Proof of concept

1. A `HypoVault` holds collateral in `PanopticPool` and opens a position with `positionSize > 0` where `PanopticMath.computeExercisedAmounts` returns non-zero `longAmounts` or `shortAmounts` (for example, `shortAmounts.rightSlot() = 100e18` or `longAmounts.rightSlot() = 100e18`).
2. The vault manager calls `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals`, which invokes `PanopticVaultAccountant.computeNAV`.
3. At lines 169–170 of `PanopticVaultAccountant.sol`, `computeNAV` executes:
   ```solidity
   poolExposure0 += int256(longAmounts.rightSlot()) - int256(shortAmounts.rightSlot());
   poolExposure1 += int256(longAmounts.leftSlot()) - int256(shortAmounts.leftSlot());
   ```
4. The resulting `poolExposure0` and `poolExposure1` are miscalculated by `2 * (longAmounts - shortAmounts)` (or truncated to `0` at line 250 via `Math.max(poolExposure0 + poolExposure1, 0)` if driven negative).
5. `HypoVault` mints an incorrect number of shares to depositors in `fulfillDeposits` or distributes an incorrect amount of underlying assets to withdrawers in `fulfillWithdrawals`.

### Impact assessment

- **Severity**: High.
- **Technical Impact**: `PanopticVaultAccountant.computeNAV` returns an inaccurate NAV whenever the vault holds open option positions in a `PanopticPool`.
- **Business Impact**: Depositors receive too many or too few shares in `HypoVault.fulfillDeposits`, and withdrawers receive too many or too few underlying tokens in `HypoVault.fulfillWithdrawals`, leading to direct dilution of vault shares and loss of user funds.

### Remediation

Update lines 169–170 in `PanopticVaultAccountant.computeNAV` so that the exercised notional amounts use the consistent sign convention:

```solidity
                poolExposure0 += int256(shortAmounts.rightSlot()) - int256(longAmounts.rightSlot());
                poolExposure1 += int256(shortAmounts.leftSlot()) - int256(longAmounts.leftSlot());
```

## Finding 5: Inconsistent Native Token Normalization in `PanopticVaultAccountant.computeNAV` Causes Misaccounting or Revert

### Synopsis

In `PanopticVaultAccountant.computeNAV`, native token normalization from `address(0)` to `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` is performed only on `pools[i].token0`. If `pools[i].token1` or `underlyingToken` represents the native token via `address(0)` or `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`, identifier comparisons, deduplication in `underlyingTokens`, and balance lookups fail, corrupting NAV or reverting epoch fulfillment.

### Technical details

`PanopticVaultAccountant.computeNAV` tracks which pool tokens have already had their direct vault balances counted by storing visited token addresses in a fixed-size memory array `underlyingTokens = new address[](pools.length * 2)`. Because unpopulated entries in `underlyingTokens` default to `address(0)`, the loop uses `underlyingTokens[j] == address(0)` (line 188) as the sentinel for the first free slot in the array.

To prevent native ETH (`address(0)`) from colliding with empty array slots, lines 177–178 rewrite `pools[i].token0`:

```solidity
if (address(pools[i].token0) == address(0))
    pools[i].token0 = IERC20Partial(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE);
```

And lines 197–201 special-case `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` for `token0` to read `address(_vault).balance` instead of calling `IERC20Partial.balanceOf(_vault)`.

However, this normalization and native-balance handling is inconsistent across the function:

1. **`pools[i].token1` is not normalized or handled as native ETH:**
   - If `pools[i].token1` is `address(0)`, lines 177–178 do not rewrite it to `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`. When the deduplication loop runs at line 186 (`if (underlyingTokens[j] == address(pools[i].token1)) skipToken1 = true;`), `pools[i].token1` (`address(0)`) matches the first unpopulated slot in `underlyingTokens` (`address(0)`), setting `skipToken1 = true` on the first pool that uses it. Consequently, `token1` is never recorded in `underlyingTokens` and its balance is completely omitted at line 202.
   - If `pools[i].token1` is `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` (for instance, when the sentinel address `0xEeee...` is used in `PoolInfo` and ordered after a `token0` with an address `< 0xEeee...`), line 202 (`if (!skipToken1) poolExposure1 += int256(pools[i].token1.balanceOf(_vault));`) attempts an external `balanceOf(_vault)` call on `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` instead of reading `address(_vault).balance`, reverting the transaction because `0xEeee...` has no code.

2. **`underlyingToken` is not normalized or handled as native ETH:**
   - If `underlyingToken` is `address(0)`, rewriting `pools[i].token0` from `address(0)` to `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` at line 178 causes `address(pools[i].token0) != underlyingToken` at line 215 to evaluate to `true` even though both represent native ETH. `computeNAV` then erroneously converts `poolExposure0` through `pools[i].oracle0`.
   - At lines 255–258, `underlyingTokens[i] == underlyingToken` compares `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` against `address(0)` (either failing to match if `underlyingTokens` is full and reverting on `IERC20Partial(address(0)).balanceOf(_vault)`, or falsely matching an unpopulated `address(0)` slot when tokens are shared across pools), and if `underlyingToken` is `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` and not present in any pool, line 258 reverts by calling `IERC20Partial(0xEeee...).balanceOf(_vault)` instead of `address(_vault).balance`.

### Assumptions and preconditions

1. A vault tracked by `PanopticVaultAccountant` is configured with a `PoolInfo` entry where `token1` is `address(0)` or `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`, or `computeNAV` is called with `underlyingToken` set to `address(0)` or `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`.
2. The manager calls `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals`, invoking `PanopticVaultAccountant.computeNAV`.

### Proof of concept

1. Suppose a `PoolInfo` entry is configured with a token pair where `token0` is an ERC20 at `0x1111111111111111111111111111111111111111` and `token1` represents native ETH using `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` (or `address(0)`).
2. The vault holds `10 ether` in native ETH (`address(vault).balance == 10 ether`).
3. When `computeNAV` executes lines 177–183:
   - **Case A (`token1 == 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`):** `skipToken1` remains `false`. At line 202, `pools[i].token1.balanceOf(_vault)` performs a `STATICCALL` to `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`, which returns empty data and causes ABI decoding of the `uint256` return value to revert.
   - **Case B (`token1 == address(0)`):** Lines 177–178 do not normalize `token1`. At line 186, on `j = 0`, `underlyingTokens[0]` is `address(0)`, so `underlyingTokens[0] == address(pools[0].token1)` evaluates to `true`, setting `skipToken1 = true`. Line 202 is skipped, and the vault's `10 ether` native balance is omitted from `poolExposure1` and `nav`.
4. Similarly, if `underlyingToken == address(0)` and `pools[0].token0 == address(0)`, line 178 rewrites `pools[0].token0` to `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`. Line 215 then evaluates `address(pools[0].token0) != underlyingToken` as `true` (`0xEeee... != address(0)`), erroneously applying `oracle0` conversion to `poolExposure0`.

### Impact assessment

**Severity: Medium**

- **Direct Technical Effect:** `computeNAV` either reverts unconditionally (blocking `fulfillDeposits` and `fulfillWithdrawals` in `HypoVault`), omits the vault's native ETH balance from NAV, or applies an erroneous oracle conversion when `underlyingToken` is `address(0)`.
- **Business Impact:** Incorrect NAV calculation leads to mispriced vault shares during deposit and withdrawal epoch fulfillment, or a denial of service of vault accounting until the pool configuration is updated (which is permanent if `vaultLocked[vault]` is `true`).

### Remediation

Normalize `pools[i].token0`, `pools[i].token1`, and `underlyingToken` consistently from `address(0)` to `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`, and check for `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` whenever querying `token1` or `underlyingToken` balances:

```solidity
            if (address(pools[i].token0) == address(0))
                pools[i].token0 = IERC20Partial(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE);
            if (address(pools[i].token1) == address(0))
                pools[i].token1 = IERC20Partial(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE);
            if (underlyingToken == address(0))
                underlyingToken = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;

            bool skipToken0 = false;
            bool skipToken1 = false;
```

Additionally, use `address(_vault).balance` when `address(pools[i].token1)` or `underlyingToken` equals `0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE`.

## Finding 6: Unencumbered Vault Token Balances Added to Per-Pool Exposure in `PanopticVaultAccountant.computeNAV` Are Incorrectly Offset by Non-Recourse Pool Debt

### Synopsis

In `PanopticVaultAccountant.computeNAV`, the vault's direct wallet balances of `token0` and `token1` are attributed to the first pool containing each token and added to `poolExposure0` and `poolExposure1` before clamping negative pool exposure to zero, causing non-recourse pool debt to erase unencumbered vault assets and understate NAV.

### Technical details

Each `PanopticPool` maintains isolated margin through its `collateralToken0` and `collateralToken1` contracts. As noted at `src/accountants/PanopticVaultAccountant.sol:249` (`// debt in pools with negative exposure does not need to be paid back`), if a vault's net position and collateral value within a `PanopticPool` is negative, that shortfall is non-recourse debt and cannot claim tokens held outside that `PanopticPool` (such as tokens sitting idle in the `HypoVault` contract or allocated to other pools). Accordingly, line 250 clamps each pool's net contribution to NAV at zero:

```solidity
nav += uint256(Math.max(poolExposure0 + poolExposure1, 0));
```

However, at lines 180–202, `computeNAV` deduplicates pool tokens via `skipToken0`, `skipToken1`, and the `underlyingTokens` array, and adds the vault's entire unencumbered wallet balance of `pools[i].token0` and `pools[i].token1` (`address(_vault).balance` or `IERC20Partial.balanceOf(_vault)`) directly into `poolExposure0` and `poolExposure1` of the first pool `i` that references each token—prior to applying `Math.max(poolExposure0 + poolExposure1, 0)` at line 250.

This creates a flawed aggregation when pool `i` has negative net exposure from its Panoptic positions and collateral:

1. **Direct wallet balances are offset against non-recourse pool debt:** The vault's idle balance of `token0` or `token1` is added to pool `i`'s negative `poolExposure0` / `poolExposure1` before clamping to `0`, reducing or completely eliminating those wallet balances from the resulting `nav`.
2. **Cross-pool shared-balance misattribution:** When multiple pools share a token, `skipToken0` / `skipToken1` attributes 100% of the vault's wallet balance of that token to the first pool in `pools` that contains it. If that first pool has negative exposure, the balance is absorbed by the first pool's non-recourse debt and skipped in all subsequent pools (`skipToken0 == true` / `skipToken1 == true`).
3. **Omission of `underlyingToken` balance and accounting failure in `HypoVault`:** When `underlyingToken` is `token0` or `token1` of a pool with negative exposure, `underlyingTokens` records `underlyingToken`, setting `skipUnderlying = true` at line 256 so line 258 (`if (!skipUnderlying) nav += IERC20Partial(underlyingToken).balanceOf(_vault);`) is skipped. Because `underlyingToken.balanceOf(_vault)` holds queued `assetsDeposited` and `reservedWithdrawalAssets`, having it absorbed by a pool's negative exposure causes `HypoVault.fulfillDeposits` (lines 480–483) and `HypoVault.fulfillWithdrawals` (lines 522–525) to either revert from arithmetic underflow or compute an understated `totalAssets`.

### Assumptions and preconditions

1. The `HypoVault` is configured with `PanopticVaultAccountant` and at least one Panoptic pool in `pools`.
2. At least one pool `pools[i]` (specifically the first pool in `pools` referencing `token0` or `token1`) has a negative net position and collateral exposure (`poolExposure0 + poolExposure1 < 0` prior to adding wallet balances).
3. The `HypoVault` holds a non-zero direct balance of `pools[i].token0` or `pools[i].token1` (for example, idle `underlyingToken`, queued deposit assets, or reserved withdrawal assets).

### Proof of concept

Consider a `HypoVault` whose `underlyingToken` is `USDC`, trading on a single `WETH/USDC` Panoptic pool (`pools[0]`, where `token0 = WETH` and `token1 = USDC` with `1 WETH = 2000 USDC`):

1. Users have requested deposits totaling `1,000 USDC` (`depositEpochState[0].assetsDeposited = 1000e6`), so `USDC.balanceOf(vault) = 1000e6`. The vault also holds `1 WETH` (`2,000 USDC` value) directly in its wallet (`WETH.balanceOf(vault) = 1e18`).
2. Inside `pools[0].pool`, the vault's options position is deeply underwater: its net position and collateral exposure is `-2,500 USDC` (`poolExposure0 + poolExposure1 = -2500e6` in `USDC` terms). Because Panoptic pool debt is non-recourse, the true value of the vault's position in `pools[0].pool` is `max(-2500e6, 0) = 0`, and the true NAV of the vault is `2000e6 (WETH in wallet) + 1000e6 (USDC in wallet) = 3000e6`.
3. When `computeNAV` executes for `i = 0`:
   - `skipToken0` and `skipToken1` are both `false` (lines 180–195), and `underlyingTokens` records both `WETH` and `USDC`.
   - Lines 197–202 add `WETH.balanceOf(vault)` (`1e18`) to `poolExposure0` and `USDC.balanceOf(vault)` (`1000e6`) to `poolExposure1`.
   - After converting `poolExposure0` to `USDC` (lines 215–230), `poolExposure0 + poolExposure1 = -2500e6 + 2000e6 + 1000e6 = +500e6`.
   - Line 250 sets `nav = uint256(Math.max(500e6, 0)) = 500e6`.
   - At lines 254–258, `skipUnderlying` is `true` because `USDC` is in `underlyingTokens`, so `USDC.balanceOf(vault)` is not added.
4. `computeNAV` returns `500e6` instead of `3000e6`.
5. When the manager calls `HypoVault.fulfillDeposits(1000e6, managerInput)` (`src/HypoVault.sol:480-483`), the calculation:
   ```solidity
   uint256 totalAssets = 500e6 + 1 - 1000e6 - 0;
   ```
   reverts due to arithmetic underflow, blocking deposit fulfillment (or, if the remaining NAV exceeds `1000e6`, fulfills deposits and withdrawals at a corrupted share price).

### Impact assessment

**Severity: High**

- **Direct Technical Effect:** `PanopticVaultAccountant.computeNAV` deducts non-recourse Panoptic pool debt from the vault's unencumbered ERC20/ETH wallet balances, returning an understated NAV.
- **Business and Protocol Impact:**
  1. **Share Price Corruption (Loss of Funds / Dilution):** During `fulfillDeposits`, an understated `totalAssets` mints an inflated number of shares to new depositors, diluting existing vault shareholders. During `fulfillWithdrawals`, an understated `totalAssets` pays withdrawing users fewer underlying assets than their shares are entitled to, leaving the difference to remaining shareholders.
  2. **Denial of Service on Epoch Fulfillment:** If the negative pool exposure reduces `nav` below `epochState.assetsDeposited + reservedWithdrawalAssets - 1`, both `fulfillDeposits` and `fulfillWithdrawals` revert with an arithmetic underflow until the underwater position is closed or additional funds are added.

### Remediation

Separate the clamp of each pool's Panoptic exposure (`poolExposure0 + poolExposure1`) from the valuation of the vault's direct wallet balances (`token0`, `token1`, and native ETH), so that non-recourse negative exposure in a `PanopticPool` is floored at zero before adding the vault's unencumbered wallet balances.

```solidity
                poolExposure0 += int256(longAmounts.rightSlot()) - int256(shortAmounts.rightSlot());
                poolExposure1 += int256(longAmounts.leftSlot()) - int256(shortAmounts.leftSlot());

                numLegs += positionLegs;
            }

            if (numLegs != pools[i].pool.numberOfLegs(_vault)) revert IncorrectPositionList();

            if (address(pools[i].token0) == address(0))
                pools[i].token0 = IERC20Partial(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE);

            bool skipToken0 = false;
            bool skipToken1 = false;

            // optimized for small number of pools
            for (uint256 j = 0; j < underlyingTokens.length; j++) {
                if (underlyingTokens[j] == address(pools[i].token0)) skipToken0 = true;
                if (underlyingTokens[j] == address(pools[i].token1)) skipToken1 = true;

                if (underlyingTokens[j] == address(0)) {
                    if (!skipToken0) underlyingTokens[j] = address(pools[i].token0);
                    // ensure a gap is not created in the underlyingTokens array
                    if (!skipToken1)
                        underlyingTokens[j + (skipToken0 ? 0 : 1)] = address(pools[i].token1);
                    break;
                }
            }

            int256 walletBalance0 = !skipToken0
                ? (
                    address(pools[i].token0) ==
                        address(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE)
                        ? int256(address(_vault).balance)
                        : int256(pools[i].token0.balanceOf(_vault))
                )
                : int256(0);
            int256 walletBalance1 = !skipToken1
                ? int256(pools[i].token1.balanceOf(_vault))
                : int256(0);

            uint256 collateralBalance = pools[i].pool.collateralToken0().balanceOf(_vault);
            poolExposure0 += int256(
                pools[i].pool.collateralToken0().previewRedeem(collateralBalance)
            );

            collateralBalance = pools[i].pool.collateralToken1().balanceOf(_vault);
            poolExposure1 += int256(
                pools[i].pool.collateralToken1().previewRedeem(collateralBalance)
            );

            // convert position values and wallet balances to underlying
            if (address(pools[i].token0) != underlyingToken) {
                int24 conversionTick = PanopticMath.twapFilter(
                    pools[i].oracle0,
                    pools[i].twapWindow
                );
                if (
                    Math.abs(conversionTick - managerPrices[i].token0Price) >
                    pools[i].maxPriceDeviation
                ) revert StaleOraclePrice();

                uint160 conversionPrice = Math.getSqrtRatioAtTick(
                    pools[i].isUnderlyingToken0InOracle0 ? -conversionTick : conversionTick
                );

                poolExposure0 = PanopticMath.convert0to1(poolExposure0, conversionPrice);
                walletBalance0 = PanopticMath.convert0to1(walletBalance0, conversionPrice);
            }

            if (address(pools[i].token1) != underlyingToken) {
                int24 conversionTick = PanopticMath.twapFilter(
                    pools[i].oracle1,
                    pools[i].twapWindow
                );
                if (
                    Math.abs(conversionTick - managerPrices[i].token1Price) >
                    pools[i].maxPriceDeviation
                ) revert StaleOraclePrice();

                uint160 conversionPrice = Math.getSqrtRatioAtTick(
                    pools[i].isUnderlyingToken0InOracle1 ? conversionTick : -conversionTick
                );

                poolExposure1 = PanopticMath.convert1to0(poolExposure1, conversionPrice);
                walletBalance1 = PanopticMath.convert1to0(walletBalance1, conversionPrice);
            }

            // debt in pools with negative exposure does not need to be paid back
            nav +=
                uint256(Math.max(poolExposure0 + poolExposure1, 0)) +
                uint256(walletBalance0 + walletBalance1);
        }
```

## Finding 7: Flash-deposit share inflation in `PanopticVaultAccountant.computeNAV` via `CollateralTracker.previewRedeem` undervalues vault collateral

### Synopsis

`PanopticVaultAccountant.computeNAV` values the vault's Panoptic collateral shares using `CollateralTracker.previewRedeem`. Because `CollateralTracker.deposit` charges an MEV commission fee (`COMMISSION_FEE`) that mints fewer shares than proportional while adding the full deposit to `s_poolAssets`, an attacker can flash-deposit into `CollateralTracker` to inflate `totalAssets() / totalSupply` and manipulate the vault's NAV.

### Technical details

In `PanopticVaultAccountant.computeNAV`, lines 204–212 compute the value of the vault's collateral shares in `collateralToken0` and `collateralToken1`:

```solidity
uint256 collateralBalance = pools[i].pool.collateralToken0().balanceOf(_vault);
poolExposure0 += int256(
    pools[i].pool.collateralToken0().previewRedeem(collateralBalance)
);
```

In `CollateralTracker` (`lib/panoptic-v1.1/contracts/CollateralTracker.sol`), `previewRedeem(shares)` delegates directly to `convertToAssets(shares)`:

```solidity
function convertToAssets(uint256 shares) public view returns (uint256 assets) {
    return Math.mulDiv(shares, totalAssets(), totalSupply);
}
```

where `totalAssets()` is `uint256(s_poolAssets) + s_inAMM`.

However, when any user deposits `assets` into `CollateralTracker.deposit(assets, receiver)`, an MEV tax (`COMMISSION_FEE`) is deducted from the minted shares while the full `assets` amount is added to `s_poolAssets`:

```solidity
function previewDeposit(uint256 assets) public view returns (uint256 shares) {
    unchecked {
        shares = Math.mulDiv(
            assets * (DECIMALS - COMMISSION_FEE),
            totalSupply,
            totalAssets() * DECIMALS
        );
    }
}
```

And upon redemption (`redeem` / `withdraw`), if the user has no open positions, `CollateralTracker` burns their shares and returns `convertToAssets(shares)` without charging an exit fee, meaning the user only loses a fraction of `COMMISSION_FEE` proportional to the pre-existing pool share, while while the deposit is active, the share price `totalAssets() / totalSupply` is instantaneously inflated by up to `COMMISSION_FEE / DECIMALS` (as the deposit size grows relative to existing `totalAssets()`).

Conversely, or during deposit/withdrawal fulfillment in `HypoVault`, the instantaneous exchange rate reported by `previewRedeem(collateralBalance)` can be manipulated within a single transaction or across blocks around `fulfillDeposits` / `fulfillWithdrawals`:
1. An attacker deposits a large amount of assets into `CollateralTracker` right before `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals`.
2. Because `previewDeposit` mints `assets * (DECIMALS - COMMISSION_FEE) / DECIMALS` worth of shares while `s_poolAssets` increases by the full `assets`, the ratio `totalAssets() / totalSupply` in `CollateralTracker` increases, inflating `previewRedeem(collateralBalance)` for `_vault`.
3. Additionally, `previewRedeem` values all `collateralBalance` using `totalAssets()` (which includes `s_inAMM`, i.e. assets locked in Uniswap V4 that cannot be redeemed above `maxRedeem`), whereas realizable redemption is capped and affected by pool utilization and commission fee mechanics.
4. When `computeNAV` is called, the vault's `collateralToken0` (and `collateralToken1`) balance is overvalued, distorting `totalAssets` in `HypoVault` so withdrawing users receive more assets per share (or depositing users receive fewer shares per asset).

### Assumptions and preconditions

1. The `HypoVault` holds a non-zero balance of `collateralToken0` or `collateralToken1` in a configured `PanopticPool`.
2. `COMMISSION_FEE` in the `CollateralTracker` is non-zero (standard in Panoptic pools).
3. An attacker has access to capital (e.g., via a flash loan or personal funds) up to the `maxDeposit` limit (`type(uint104).max`) to deposit into the `CollateralTracker` before the manager calls `fulfillDeposits` or `fulfillWithdrawals`.

### Proof of concept

1. Suppose `CollateralTracker` (`collateralToken0`) has `totalAssets = 100e18`, `totalSupply = 100e18`, and `COMMISSION_FEE = 100` (1%). The `HypoVault` holds nearly all shares (`collateralBalance = 100e18`), so `previewRedeem(collateralBalance) = 100e18`.
2. An attacker requests a withdrawal of their shares from `HypoVault`.
3. Before the manager calls `fulfillWithdrawals`, the attacker deposits `9,900e18` of `token0` into `CollateralTracker`.
4. `CollateralTracker.deposit` mints `9,900e18 * (10000 - 100) / 10000 = 9,801e18` shares to the attacker and increases `s_poolAssets` by `9,900e18`.
5. Now `totalAssets = 10,000e18` and `totalSupply = 9,901e18`.
6. When `HypoVault.fulfillWithdrawals` calls `accountant.computeNAV`, `collateralToken0().previewRedeem(100e18)` returns `100e18 * 10,000e18 / 9,901e18 ≈ 100.9998e18` (inflated by nearly `1e18`).
7. The attacker then redeems their `9,801e18` shares from `CollateralTracker`, receiving `9,801e18 * 10,000e18 / 9,901e18 ≈ 9,899.0001e18` (paying only the ~1e18 donation to the pool, most of which is captured by the vault's inflated NAV calculation or shared across the vault's leveraged exposure).

### Impact assessment

**Severity: Medium**

The share-to-asset exchange rate in `CollateralTracker` (`previewRedeem`) is manipulable via deposits due to the asymmetric commission fee (`previewDeposit` vs `previewRedeem`) and does not reflect the post-commission or realizable value of the collateral shares. This allows third parties to distort `computeNAV` during epoch fulfillment, skewing share minting and asset redemption prices in `HypoVault`.

### Remediation

Ensure that collateral share valuation in `PanopticVaultAccountant.computeNAV` cannot be manipulated by transient deposits into `CollateralTracker` or, where appropriate, account for the commission/realizable redemption constraints and validate that the pool's collateral share price has not been abnormally skewed within the block.

```solidity
            uint256 collateralBalance = pools[i].pool.collateralToken0().balanceOf(_vault);
            poolExposure0 += int256(
                pools[i].pool.collateralToken0().convertToAssets(collateralBalance)
            );
```

*(Note: To fully prevent instant rate manipulation on `CollateralTracker`, the accountant or pool integration should check that the collateral tracker share price has not deviated or been flash-manipulated in the same block prior to `computeNAV`.)*

## Finding 8: Unadjusted Book-Value Valuation via `previewRedeem` in `PanopticVaultAccountant.computeNAV` Overstates Recoverable Collateral

### Synopsis

`PanopticVaultAccountant.computeNAV` values the vault's Panoptic `CollateralTracker` shares using `previewRedeem(balanceOf(_vault))`. Because `previewRedeem` prices shares against total book assets (`s_poolAssets + s_inAMM`) while ignoring `maxRedeem`/`maxWithdraw` liquidity and position-lock restrictions, `computeNAV` overstates recoverable NAV, distorting share pricing in `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals`.

### Technical details

In `src/accountants/PanopticVaultAccountant.sol`, `computeNAV` calculates the vault's exposure in each configured `PanopticPool` by adding the value of the vault's `collateralToken0` and `collateralToken1` share balances (lines 204–212):

```solidity
uint256 collateralBalance = pools[i].pool.collateralToken0().balanceOf(_vault);
poolExposure0 += int256(
    pools[i].pool.collateralToken0().previewRedeem(collateralBalance)
);

collateralBalance = pools[i].pool.collateralToken1().balanceOf(_vault);
poolExposure1 += int256(
    pools[i].pool.collateralToken1().previewRedeem(collateralBalance)
);
```

In `lib/panoptic-v1.1/contracts/CollateralTracker.sol`, `previewRedeem(uint256 shares)` delegates directly to `convertToAssets(shares)` (lines 651–653), which computes:

$$\text{assets} = \left\lfloor \frac{\text{shares} \times \text{totalAssets}()}{\text{totalSupply}} \right\rfloor, \quad \text{where } \text{totalAssets}() = \text{s\_poolAssets} + \text{s\_inAMM}$$

However, `CollateralTracker` enforces strict redemption and withdrawal limits in `redeem` (line 666) and `withdraw` (line 564) via `maxRedeem` (lines 639–646) and `maxWithdraw` (lines 534–541):

1. **AMM-Locked Liquidity (`s_inAMM`) and Minimum Pool Reserve:** `totalAssets()` includes `s_inAMM`, which represents assets moved out of the `PanopticPool` into the Uniswap V4 AMM to back active option liquidity chunks, as well as the `1` wei minimum reserve in `s_poolAssets`. Both `maxWithdraw` and `maxRedeem` cap the maximum withdrawable assets at `available = poolAssets > 0 ? poolAssets - 1 : 0`. Whenever `convertToAssets(collateralBalance) > s_poolAssets - 1` (for instance, when pool utilization moves assets into `s_inAMM`, or even when the vault is the sole LP due to the `poolAssets - 1` cap), `previewRedeem(collateralBalance)` returns a value that cannot be redeemed without reverting with `Errors.ExceedsMaximumRedemption()`.
2. **Active Position Lock (`numberOfLegs(_vault) != 0`):** When `_vault` has any open option legs (`_panopticPool().numberOfLegs(_vault) != 0`), `maxRedeem(_vault)` and `maxWithdraw(_vault)` return `0`, and `CollateralTracker.transfer` / `transferFrom` revert with `Errors.PositionCountNotZero()` (lines 312, 331).

Consequently, `computeNAV` treats illiquid or unrecoverable book-value claims on `CollateralTracker` as immediately realizable assets when reporting NAV to `HypoVault.fulfillDeposits` (line 480) and `HypoVault.fulfillWithdrawals` (line 522).

### Assumptions and preconditions

1. A `HypoVault` instance is configured with `PanopticVaultAccountant` as its `accountant` and holds collateral shares in a Panoptic `CollateralTracker` (`collateralToken0` and/or `collateralToken1`).
2. Either a portion of the `CollateralTracker`'s assets is deployed into the AMM (`s_inAMM > 0` such that `convertToAssets(balanceOf(_vault)) > s_poolAssets - 1`) or the vault holds open option legs (`numberOfLegs(_vault) > 0`).

### Proof of concept

1. Suppose `HypoVault` (`_vault`) has `totalSupply = 1,000e18` shares and holds `1,000e18` shares of `collateralToken1` (representing 100% of `collateralToken1.totalSupply`).
2. Initially, `s_poolAssets = 1,000e18` and `s_inAMM = 0` in `collateralToken1`, with no other assets or positions in `_vault`.
3. Option traders mint positions in `PanopticPool` that move `600e18` of `token1` into the Uniswap V4 AMM, updating `collateralToken1` state to `s_poolAssets = 400e18` and `s_inAMM = 600e18`.
4. A vault user requests a withdrawal of `500e18` vault shares (50% of `HypoVault.totalSupply`).
5. When the manager calls `HypoVault.fulfillWithdrawals(500e18, ...)`, `PanopticVaultAccountant.computeNAV` executes lines 209–212:
   - `collateralBalance = collateralToken1.balanceOf(_vault) = 1,000e18`.
   - `collateralToken1.previewRedeem(1,000e18)` computes `1,000e18 * (400e18 + 600e18) / 1,000e18 = 1,000e18`.
6. However, `collateralToken1.maxWithdraw(_vault)` is only `400e18 - 1` (`s_poolAssets - 1`), and `collateralToken1.maxRedeem(_vault)` is `400e18 - 1` shares.
7. Because `computeNAV` reports `1,000e18` instead of the recoverable collateral (`400e18 - 1`), `fulfillWithdrawals` values the `500e18` shares at `500e18` underlying assets—more than the entire liquid balance (`400e18 - 1`) that `_vault` can actually withdraw from `collateralToken1` (any attempt by the manager to redeem `500e18` shares from `collateralToken1` reverts with `Errors.ExceedsMaximumRedemption()`).

### Impact assessment

- **Severity:** Medium / High
- **Technical Effect:** `PanopticVaultAccountant.computeNAV` overstates the vault's net asset value relative to the collateral that can actually be redeemed or withdrawn from `CollateralTracker`.
- **Business Impact:**
  - **Withdrawal Over-Allocation / Insolvency Risk:** Withdrawals fulfilled via `HypoVault.fulfillWithdrawals` while pool liquidity is utilized in `s_inAMM` credit withdrawing users with full book value, either blocking the manager from sourcing the required liquidity from `CollateralTracker` (reverting with `ExceedsMaximumRedemption`) or exhausting the vault's liquid reserves at the expense of remaining shareholders.
  - **Share Dilution on Deposit:** Deposits fulfilled via `HypoVault.fulfillDeposits` mint fewer shares to new depositors because `totalAssets` is inflated by illiquid/unrecoverable `CollateralTracker` book value.

### Remediation

Cap the collateral valuation in `PanopticVaultAccountant.computeNAV` to the actually redeemable shares (or withdrawable assets) when valuing liquid collateral, or bound the redeemable share balance by the available pool assets in `CollateralTracker`:

```solidity
            uint256 collateralBalance = Math.min(
                pools[i].pool.collateralToken0().balanceOf(_vault),
                pools[i].pool.collateralToken0().maxRedeem(_vault)
            );
            poolExposure0 += int256(
                pools[i].pool.collateralToken0().previewRedeem(collateralBalance)
            );

            collateralBalance = Math.min(
                pools[i].pool.collateralToken1().balanceOf(_vault),
                pools[i].pool.collateralToken1().maxRedeem(_vault)
            );
            poolExposure1 += int256(
                pools[i].pool.collateralToken1().previewRedeem(collateralBalance)
            );
```

Alternatively, if collateral backing active vault positions (`numberOfLegs(_vault) != 0`) is intended to be valued up to the pool's available liquidity regardless of the position-count transfer lock, cap the valued assets at the pool's available liquidity (`maxWithdraw` without the `numberOfLegs` restriction, i.e., `Math.min(collateralToken.previewRedeem(collateralBalance), availablePoolAssets)`).

## Finding 9: Inconsistent Pricing in `PanopticVaultAccountant.computeNAV` Uses TWAP Tick Instead of Manager Price for Token Conversion

### Synopsis

In `PanopticVaultAccountant.computeNAV`, option liquidity chunks are valued using `managerPrices[i].poolPrice`, but token exposures (`poolExposure0` and `poolExposure1`) are converted to `underlyingToken` using the lagging TWAP tick (`conversionTick`) instead of the validated `managerPrices[i].token0Price` and `managerPrices[i].token1Price`, causing inconsistent portfolio valuation and mispriced vault shares.

### Technical details

`PanopticVaultAccountant.computeNAV` accepts `ManagerPrices[] memory managerPrices` from the vault manager, containing `poolPrice`, `token0Price`, and `token1Price` for each configured pool. Because TWAP oracles lag behind real-time market prices, the intended design validates each manager-supplied price against the TWAP oracle (`PanopticMath.twapFilter`) within `pools[i].maxPriceDeviation` and then uses the manager-supplied spot prices to value the vault's portfolio consistently.

For option position liquidity chunks (lines 113–118 and 144–151), `computeNAV` validates `managerPrices[i].poolPrice` against `PanopticMath.twapFilter(pools[i].poolOracle, pools[i].twapWindow)` and passes `managerPrices[i].poolPrice` to `Math.getAmountsForLiquidity`.

However, when converting `poolExposure0` and `poolExposure1` to `underlyingToken` (lines 215–247):

```solidity
if (address(pools[i].token0) != underlyingToken) {
    int24 conversionTick = PanopticMath.twapFilter(
        pools[i].oracle0,
        pools[i].twapWindow
    );
    if (
        Math.abs(conversionTick - managerPrices[i].token0Price) >
        pools[i].maxPriceDeviation
    ) revert StaleOraclePrice();

    uint160 conversionPrice = Math.getSqrtRatioAtTick(
        pools[i].isUnderlyingToken0InOracle0 ? -conversionTick : conversionTick
    );

    poolExposure0 = PanopticMath.convert0to1(poolExposure0, conversionPrice);
}
```

Although `managerPrices[i].token0Price` and `managerPrices[i].token1Price` are validated against `conversionTick` (lines 220–223 and 237–240), they are never used in the conversion calculation. Instead, `Math.getSqrtRatioAtTick` uses `conversionTick` (the TWAP tick) rather than `managerPrices[i].token0Price` and `managerPrices[i].token1Price`.

This creates a pricing mismatch across portfolio components within the same pool:
1. Option legs (`getAmountsForLiquidity` and `computeExercisedAmounts`) decompose positions into `token0` and `token1` exposures based on `managerPrices[i].poolPrice`.
2. Those resulting `poolExposure0` and `poolExposure1` balances (including the option delta/notional amounts and collateral) are then converted into `underlyingToken` using the lagging TWAP `conversionTick`.

When the market price moves within `maxPriceDeviation` of the TWAP, `poolPrice` and `conversionTick` diverge, creating artificial gains or losses on option positions and miscalculating the vault's Net Asset Value (NAV).

### Assumptions and preconditions

1. A `HypoVault` is configured with `PanopticVaultAccountant` and trades in at least one `PanopticPool` where `token0` or `token1` differs from `underlyingToken`.
2. The current market price (`poolPrice` / `token0Price` / `token1Price`) differs from the lagging TWAP tick (`conversionTick`) within the allowed `maxPriceDeviation` band.

### Proof of concept

1. Assume `underlyingToken` is USDC (`token1`) and the vault trades in an ETH/USDC `PanopticPool` (`token0` = ETH, `token1` = USDC).
2. Suppose the 10-minute TWAP tick (`conversionTick`) corresponds to an ETH price of 2,000 USDC, while the current spot price (`managerPrices[0].poolPrice` and `managerPrices[0].token0Price`) corresponds to 2,100 USDC (within `maxPriceDeviation`).
3. The vault holds a long call option leg on 1 ETH with strike 2,050 USDC (`tokenType = 0`, `isLong = 1`).
4. During `computeNAV`:
   - At `managerPrices[0].poolPrice` (2,100 USDC), the liquidity chunk is above the strike range and consists entirely of `token1`: `getAmountsForLiquidity` returns `amount0 = 0` and `amount1 = 2,050 USDC`. Since the leg is long, `poolExposure1` is reduced by `2,050 USDC`.
   - `computeExercisedAmounts` credits `longAmounts` with `+1 ETH` to `poolExposure0`.
5. In lines 215–230, `poolExposure0` (`+1 ETH`) is converted to USDC (`underlyingToken`) using `conversionTick` (2,000 USDC) instead of `managerPrices[0].token0Price` (2,100 USDC), yielding `+2,000 USDC`.
6. Instead of valuing the in-the-money long call at `2,100 - 2,050 = +50 USDC`, `computeNAV` values the position at `2,000 - 2,050 = -50 USDC`, understating the vault's NAV by 100 USDC per ETH of notional exposure.

### Impact assessment

**Severity: Medium**

During `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals`, `computeNAV` determines the share price for all deposits and withdrawals in the epoch. Using the lagging TWAP `conversionTick` instead of `managerPrices[i].token0Price` and `managerPrices[i].token1Price` causes:
- Inconsistent valuation between option liquidity chunks (valued at `poolPrice`) and token conversions (valued at `conversionTick`).
- Stale conversion pricing for non-underlying token exposures, resulting in too many or too few vault shares minted on deposits and incorrect asset payouts on withdrawals.

### Remediation

Use `managerPrices[i].token0Price` and `managerPrices[i].token1Price` (after validating that they are within `maxPriceDeviation` of `conversionTick`) when computing `conversionPrice`:

```solidity
            // convert position values to underlying
            if (address(pools[i].token0) != underlyingToken) {
                int24 conversionTick = PanopticMath.twapFilter(
                    pools[i].oracle0,
                    pools[i].twapWindow
                );
                if (
                    Math.abs(conversionTick - managerPrices[i].token0Price) >
                    pools[i].maxPriceDeviation
                ) revert StaleOraclePrice();

                uint160 conversionPrice = Math.getSqrtRatioAtTick(
                    pools[i].isUnderlyingToken0InOracle0
                        ? -managerPrices[i].token0Price
                        : managerPrices[i].token0Price
                );

                poolExposure0 = PanopticMath.convert0to1(poolExposure0, conversionPrice);
            }

            if (address(pools[i].token1) != underlyingToken) {
                int24 conversionTick = PanopticMath.twapFilter(
                    pools[i].oracle1,
                    pools[i].twapWindow
                );
                if (
                    Math.abs(conversionTick - managerPrices[i].token1Price) >
                    pools[i].maxPriceDeviation
                ) revert StaleOraclePrice();

                uint160 conversionPrice = Math.getSqrtRatioAtTick(
                    pools[i].isUnderlyingToken0InOracle1
                        ? managerPrices[i].token1Price
                        : -managerPrices[i].token1Price
                );

                poolExposure1 = PanopticMath.convert1to0(poolExposure1, conversionPrice);
            }
```

## Finding 10: Incorrect Oracle Price Conversion Direction in `PanopticVaultAccountant.computeNAV` Causes Miscomputed Vault NAV

### Synopsis

In `PanopticVaultAccountant.computeNAV`, converting `poolExposure0` and `poolExposure1` into `underlyingToken` applies the wrong price conversion direction at line 229 (`PanopticMath.convert0to1`) and line 243 (inverted `conversionTick` ternary), causing non-underlying pool exposures to be valued using the reciprocal exchange rate and distorting the vault's NAV.

### Technical details

In `src/accountants/PanopticVaultAccountant.sol`, `computeNAV` calculates the net exposure of a vault across each configured `PanopticPool` in terms of `token0` (`poolExposure0`) and `token1` (`poolExposure1`), and then converts any exposure not already denominated in `underlyingToken` into `underlyingToken` using `oracle0` and `oracle1` (lines 215–247):

```solidity
if (address(pools[i].token0) != underlyingToken) {
    int24 conversionTick = PanopticMath.twapFilter(
        pools[i].oracle0,
        pools[i].twapWindow
    );
    if (
        Math.abs(conversionTick - managerPrices[i].token0Price) >
        pools[i].maxPriceDeviation
    ) revert StaleOraclePrice();

    uint160 conversionPrice = Math.getSqrtRatioAtTick(
        pools[i].isUnderlyingToken0InOracle0 ? -conversionTick : conversionTick
    );

    poolExposure0 = PanopticMath.convert0to1(poolExposure0, conversionPrice);
}

if (address(pools[i].token1) != underlyingToken) {
    int24 conversionTick = PanopticMath.twapFilter(
        pools[i].oracle1,
        pools[i].twapWindow
    );
    if (
        Math.abs(conversionTick - managerPrices[i].token1Price) >
        pools[i].maxPriceDeviation
    ) revert StaleOraclePrice();

    uint160 conversionPrice = Math.getSqrtRatioAtTick(
        pools[i].isUnderlyingToken0InOracle1 ? conversionTick : -conversionTick
    );

    poolExposure1 = PanopticMath.convert1to0(poolExposure1, conversionPrice);
}
```

Here, `isUnderlyingToken0InOracle0` and `isUnderlyingToken0InOracle1` indicate whether `underlyingToken` is `token0` in the respective reference oracle (`oracle0` or `oracle1`), while `PanopticMath.convert0to1` multiplies by `conversionPrice^2 / 2^192` and `PanopticMath.convert1to0` divides by `conversionPrice^2 / 2^192`:

1. For `poolExposure0` (lines 225–229): `conversionPrice` is computed using `pools[i].isUnderlyingToken0InOracle0 ? -conversionTick : conversionTick`, which orientates the tick such that `underlyingToken` is treated as `token1` and `pools[i].token0` as `token0`. However, line 229 calls `PanopticMath.convert0to1(poolExposure0, conversionPrice)` instead of `PanopticMath.convert1to0(poolExposure0, conversionPrice)` (when `conversionTick` represents `token0 / token1` in the oracle's quote convention), applying the reciprocal conversion factor.
2. For `poolExposure1` (lines 242–246): although line 246 calls `PanopticMath.convert1to0(poolExposure1, conversionPrice)`, line 243 flips the ternary sign to `pools[i].isUnderlyingToken0InOracle1 ? conversionTick : -conversionTick` instead of `pools[i].isUnderlyingToken0InOracle1 ? -conversionTick : conversionTick`. Flipping both the tick sign at line 243 and the conversion helper at line 246 cancels out the inversion and produces the same inverted conversion factor as `poolExposure0`.

As a result, whenever `pools[i].token0 != underlyingToken` or `pools[i].token1 != underlyingToken` and the oracle price differs from `1:1` (`conversionTick != 0`), `computeNAV` multiplies by the reciprocal price instead of the true conversion price to `underlyingToken`. Because `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals` rely directly on `accountant.computeNAV` to determine `totalAssets`, share minting and asset redemption amounts are miscalculated.

### Assumptions and preconditions

1. A `HypoVault` is configured to use `PanopticVaultAccountant` as its `accountant`.
2. At least one pool in `PoolInfo[]` has `token0 != underlyingToken` or `token1 != underlyingToken`.
3. The conversion oracle (`oracle0` or `oracle1`) has a non-zero TWAP tick (`conversionTick != 0`), meaning the price between the pool token and `underlyingToken` is not 1:1.

### Proof of concept

1. Consider a `HypoVault` whose `underlyingToken` is USDC and which holds a position or collateral in a pool where `token0` is WETH and `token1` is USDC (`token0 != underlyingToken`, `token1 == underlyingToken`).
2. Suppose `isUnderlyingToken0InOracle0 = false` (so `token0 = WETH` and `token1 = USDC` in `oracle0`) and the oracle tick reflects the price ratio between WETH and USDC (`conversionTick != 0`).
3. When `computeNAV` is called during `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals`, `poolExposure0` (denominated in WETH) is converted at line 229 via `PanopticMath.convert0to1(poolExposure0, Math.getSqrtRatioAtTick(conversionTick))` (and correspondingly if `token1 != underlyingToken`, `poolExposure1` is converted via `Math.getSqrtRatioAtTick(-conversionTick)` and `PanopticMath.convert1to0` at lines 242–246).
4. Because the reciprocal price ratio is applied, `poolExposure0` (and/or `poolExposure1`) is scaled in the opposite direction of the actual exchange rate relative to `underlyingToken`.
5. `computeNAV` returns an overstated or understated `nav`, causing `HypoVault.fulfillDeposits` to mint an incorrect number of shares to depositors and `HypoVault.fulfillWithdrawals` to allocate an incorrect amount of underlying assets to withdrawing users.

### Impact assessment

**Severity: High**

- **Direct Technical Effect:** `PanopticVaultAccountant.computeNAV` miscalculates the `underlyingToken` value of all non-underlying token exposures (`poolExposure0` and `poolExposure1`) by applying the reciprocal oracle price.
- **Business Implications:** Because `HypoVault` uses `computeNAV` to price shares during deposit and withdrawal fulfillment, an understated NAV dilutes existing shareholders (minting excessive shares to new depositors) and underpays withdrawing users, while an overstated NAV overcharges new depositors and allows withdrawing users to drain excess underlying assets from the vault at the expense of remaining LPs.

### Remediation

Update line 229 to use `PanopticMath.convert1to0` and update line 243 so that `isUnderlyingToken0InOracle1` negates `conversionTick` consistently with `isUnderlyingToken0InOracle0`:

```solidity
            // convert position values to underlying
            if (address(pools[i].token0) != underlyingToken) {
                int24 conversionTick = PanopticMath.twapFilter(
                    pools[i].oracle0,
                    pools[i].twapWindow
                );
                if (
                    Math.abs(conversionTick - managerPrices[i].token0Price) >
                    pools[i].maxPriceDeviation
                ) revert StaleOraclePrice();

                uint160 conversionPrice = Math.getSqrtRatioAtTick(
                    pools[i].isUnderlyingToken0InOracle0 ? -conversionTick : conversionTick
                );

                poolExposure0 = PanopticMath.convert1to0(poolExposure0, conversionPrice);
            }

            if (address(pools[i].token1) != underlyingToken) {
                int24 conversionTick = PanopticMath.twapFilter(
                    pools[i].oracle1,
                    pools[i].twapWindow
                );
                if (
                    Math.abs(conversionTick - managerPrices[i].token1Price) >
                    pools[i].maxPriceDeviation
                ) revert StaleOraclePrice();

                uint160 conversionPrice = Math.getSqrtRatioAtTick(
                    pools[i].isUnderlyingToken0InOracle1 ? -conversionTick : conversionTick
                );

                poolExposure1 = PanopticMath.convert1to0(poolExposure1, conversionPrice);
            }
```

## Finding 11: Stale Sampling Window and Misaligned Interval Division in `PanopticMath.twapFilter` Distort Oracle Conversion Ticks in `PanopticVaultAccountant.computeNAV`

### Synopsis

In `PanopticVaultAccountant.computeNAV`, token exposure conversions rely on `PanopticMath.twapFilter`, which constructs observation timestamps starting at `twapWindow / 20` instead of `0` and divides cumulative tick deltas by `twapWindow / 20` rather than the actual interval duration, omitting the freshest time window and distorting TWAP ticks.

### Technical details

When `PanopticVaultAccountant.computeNAV` converts `poolExposure0` and `poolExposure1` into `underlyingToken` (lines 215–247), it queries `conversionTick` via `PanopticMath.twapFilter(pools[i].oracle0, pools[i].twapWindow)` and `PanopticMath.twapFilter(pools[i].oracle1, pools[i].twapWindow)` (lines 216–219 and 233–236).

In `PanopticMath.twapFilter` (`lib/panoptic-v1.1/contracts/libraries/PanopticMath.sol`, lines 340–369):

```solidity
for (uint256 i = 0; i < 20; ++i) {
    secondsAgos[i] = uint32(((i + 1) * twapWindow) / 20);
}

(int56[] memory tickCumulatives, ) = oracleContract.observe(secondsAgos);

for (uint256 i = 0; i < 19; ++i) {
    twapMeasurement[i] = int24(
        (tickCumulatives[i] - tickCumulatives[i + 1]) / int56(uint56(twapWindow / 20))
    );
}
```

This TWAP calculation has two sampling and arithmetic defects that compromise oracle price integrity:

1. **Omission of the freshest observation window (`[0, twapWindow / 20]`)**:
   Because the loop computes `secondsAgos[i] = uint32(((i + 1) * twapWindow) / 20)` for `i = 0` to `19`, the most recent observation requested from `oracleContract.observe` is `secondsAgos[0] = twapWindow / 20` (e.g., 30 seconds ago for a 600-second window) rather than `0` (the current block timestamp). As documented in `IV3CompatibleOracle.observe`, obtaining a TWAP up to the current timestamp requires passing `0` as the end of the period. Consequently, the entire most recent `[0, twapWindow / 20]` window is never sampled, and only 19 historical windows covering `[twapWindow / 20, twapWindow]` are measured.

2. **Mismatched interval divisor when `twapWindow` is not divisible by 20**:
   In `PanopticVaultAccountant`, `pools[i].twapWindow` is an arbitrary configurable `uint32` per pool. For each window `i`, the cumulative tick difference `tickCumulatives[i] - tickCumulatives[i + 1]` is accumulated over an actual time delta of `secondsAgos[i + 1] - secondsAgos[i] = ((i + 2) * twapWindow) / 20 - ((i + 1) * twapWindow) / 20` seconds. However, line 360 divides every window's cumulative tick delta by the truncated integer `int56(uint56(twapWindow / 20))`. Whenever `twapWindow % 20 != 0`, `(twapWindow % 20)` of the sub-intervals span `twapWindow / 20 + 1` seconds while still being divided by `twapWindow / 20`, systematically inflating the magnitude of the computed average ticks by a factor of `(twapWindow / 20 + 1) / (twapWindow / 20)` (and reverting with a division-by-zero panic if `twapWindow < 20`).

### Assumptions and preconditions

1. A vault uses `PanopticVaultAccountant` with at least one pool where `token0 != underlyingToken` or `token1 != underlyingToken`.
2. Either a price movement occurs within the most recent `twapWindow / 20` seconds before `fulfillDeposits` / `fulfillWithdrawals`, or the pool is configured with a `twapWindow` that is not a multiple of 20.

### Proof of concept

1. Suppose a pool in `PanopticVaultAccountant` is configured with `token1 != underlyingToken` and `twapWindow = 610` (10 minutes 10 seconds), where the steady-state tick of `oracle1` is `2000`.
2. When `computeNAV` executes lines 233–236:
   - `PanopticMath.twapFilter(pools[i].oracle1, 610)` builds `secondsAgos[i] = ((i + 1) * 610) / 20`.
   - Half of the 19 intervals have duration `secondsAgos[i + 1] - secondsAgos[i] = 31` seconds, while `twapWindow / 20 = 30`.
   - For each 31-second interval, `tickCumulatives[i] - tickCumulatives[i + 1] = 2000 * 31 = 62000`.
   - Line 360 divides `62000` by `30` instead of `31`, producing `twapMeasurement[i] = 2066` (a 66-tick error, or ~0.66% price distortion) even though the oracle tick never moved from `2000`.
3. If `maxPriceDeviation` is `50` ticks and the manager passes the true tick `managerPrices[i].token1Price = 2000`, `Math.abs(2066 - 2000) = 66 > 50`, causing `computeNAV` to revert with `StaleOraclePrice()` and blocking `fulfillDeposits` and `fulfillWithdrawals`.
4. Even when `twapWindow = 600` (a multiple of 20), because `secondsAgos[0] = 30`, any price change in `oracle1` during the last 30 seconds `[0, 30]` is completely ignored by `conversionTick`. If the spot and manager price move by 40 ticks in the last 30 seconds, `conversionTick` remains purely historical, mispricing `poolExposure1` (or reverting if the recent move exceeds `maxPriceDeviation`).

### Impact assessment

- **Severity**: Medium
- **Direct Technical Effect**: `PanopticMath.twapFilter` ignores the most recent `twapWindow / 20` seconds of oracle observations and miscalculates interval tick averages whenever `twapWindow` is not divisible by 20.
- **Business Implications**: `PanopticVaultAccountant.computeNAV` either computes an inaccurate `conversionPrice` for `poolExposure0` and `poolExposure1`—skewing share minting in `HypoVault.fulfillDeposits` and asset payouts in `HypoVault.fulfillWithdrawals`—or unexpectedly reverts with `StaleOraclePrice()`.

### Remediation

Update `PanopticMath.twapFilter` to include `secondsAgos[0] = 0` across 20 intervals (21 observation points) up to the current block timestamp and divide each cumulative tick delta by the exact duration `secondsAgos[i + 1] - secondsAgos[i]` (rounding down toward negative infinity for negative tick deltas):

```solidity
    function twapFilter(
        IV3CompatibleOracle oracleContract,
        uint32 twapWindow
    ) external view returns (int24) {
        uint32[] memory secondsAgos = new uint32[](20);

        int256[] memory twapMeasurement = new int256[](19);

        unchecked {
            // construct the time slots starting from 0 (current timestamp)
            for (uint256 i = 0; i < 20; ++i) {
                secondsAgos[i] = uint32((i * twapWindow) / 19);
            }

            // observe the tickCumulative at the 20 pre-defined time slots
            (int56[] memory tickCumulatives, ) = oracleContract.observe(secondsAgos);

            // compute the average tick per window using the exact interval length
            for (uint256 i = 0; i < 19; ++i) {
                int56 tickDelta = tickCumulatives[i] - tickCumulatives[i + 1];
                int56 timeDelta = int56(uint56(secondsAgos[i + 1] - secondsAgos[i]));
                int24 tick = int24(tickDelta / timeDelta);
                if (tickDelta < 0 && (tickDelta % timeDelta != 0)) --tick;
                twapMeasurement[i] = tick;
            }

            // sort the tick measurements
            int256[] memory sortedTicks = Math.sort(twapMeasurement);

            // Get the median value
            return int24(sortedTicks[9]);
        }
    }
```

## Finding 12: Incorrect Token Conversion in `PanopticVaultAccountant.computeNAV` Corrupts Vault NAV Calculation

### Synopsis

In `PanopticVaultAccountant.computeNAV`, `poolExposure1` is converted to `underlyingToken` using `PanopticMath.convert1to0` with an inconsistent oracle quote-direction relative to `conversionPrice`. This miscalculates the `token1` exposure value in `underlyingToken` terms, resulting in an inaccurate Net Asset Value (`nav`) and incorrect share/asset pricing during vault deposit and withdrawal fulfillments.

### Technical details

In `src/accountants/PanopticVaultAccountant.sol`, `computeNAV` calculates the vault's net exposure in each configured `PanopticPool` across both pool tokens (`poolExposure0` in `pools[i].token0` and `poolExposure1` in `pools[i].token1`), including accumulated premia, active option liquidity chunks, exercised amounts, idle token balances, and `CollateralTracker` shares.

Before aggregating the pool's net value into `nav` at line 250 (`nav += uint256(Math.max(poolExposure0 + poolExposure1, 0))`), both `poolExposure0` and `poolExposure1` must be converted into `underlyingToken` units whenever the respective pool token differs from `underlyingToken`:

1. For `token0` (lines 215–230):
   ```solidity
   uint160 conversionPrice = Math.getSqrtRatioAtTick(
       pools[i].isUnderlyingToken0InOracle0 ? -conversionTick : conversionTick
   );

   poolExposure0 = PanopticMath.convert0to1(poolExposure0, conversionPrice);
   ```
   Here, `conversionTick` is the TWAP tick from `oracle0`, and the tick sign is oriented according to `isUnderlyingToken0InOracle0` so that `conversionPrice` represents the price of `token0` denominated in `underlyingToken` (`sqrt(underlyingToken / token0) * 2^96`), which is then multiplied with `poolExposure0` via `PanopticMath.convert0to1`.

2. For `token1` (lines 232–247):
   ```solidity
   uint160 conversionPrice = Math.getSqrtRatioAtTick(
       pools[i].isUnderlyingToken0InOracle1 ? conversionTick : -conversionTick
   );

   poolExposure1 = PanopticMath.convert1to0(poolExposure1, conversionPrice);
   ```
   In contrast to `token0`, `poolExposure1` is merely the vault's exposure in the second token of `pools[i]` (`pools[i].token1`), not necessarily `token1` of `oracle1`—its position within `oracle1` is already fully specified by the `isUnderlyingToken0InOracle1` flag. Because line 243 flips the ternary sign (`conversionTick : -conversionTick`) while line 246 also switches the conversion function from `convert0to1` to `convert1to0` (which divides by `(conversionPrice / 2^96)^2` instead of multiplying), any oracle configuration or quote convention that expects `conversionPrice` to be applied in the same directional convention as `oracle0`—or where `isUnderlyingToken0InOracle1` is set under the convention that `conversionPrice` is the `token1 -> underlyingToken` ratio—suffers from an oracle quote-direction mismatch at line 246.

Furthermore, when `conversionTick` is negative and `isUnderlyingToken0InOracle1` is `false` (or positive when `true`), `convert1to0` performs a division (`Math.mulDiv(Math.absUint(amount), 2 ** 192, uint256(sqrtPriceX96) ** 2)`) rather than the multiplication in `convert0to1`, introducing asymmetric truncation and inverse scaling whenever the oracle quote direction and conversion helper are mismatched. The corrupted `poolExposure1` is then summed directly with `poolExposure0` at line 250:
```solidity
nav += uint256(Math.max(poolExposure0 + poolExposure1, 0));
```
This directly distorts the `nav` returned to `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals`.

### Assumptions and preconditions

1. A `HypoVault` is configured with `PanopticVaultAccountant` as its `accountant`.
2. At least one pool in `pools` has `address(pools[i].token1) != underlyingToken` and a non-zero `poolExposure1` (from collateral shares, option positions, premia, or idle `token1` balances).
3. `oracle1` has a non-zero TWAP tick (`conversionTick != 0`), so the price ratio between `token1` and `underlyingToken` is not `1:1`.

### Proof of concept

1. Suppose a `HypoVault` uses `USDC` as `underlyingToken` and trades in a `PanopticPool` where `token0 = USDC` and `token1 = WETH`.
2. The vault holds `100 WETH` (`poolExposure1 = 100e18`) in `token1` exposure and `0 USDC` in `token0` exposure.
3. Suppose `oracle1` quotes the conversion price between `WETH` and `USDC` such that `conversionPrice` at line 242 already represents the `USDC / WETH` price (e.g., `2000 USDC` per `WETH`, so `(conversionPrice / 2^96)^2 = 2000 * 1e6 / 1e18`).
4. At line 246, `PanopticVaultAccountant.computeNAV` executes:
   ```solidity
   poolExposure1 = PanopticMath.convert1to0(poolExposure1, conversionPrice);
   ```
   Instead of multiplying `poolExposure1` by the price ratio (`convert0to1`) to obtain `200,000 USDC` (`200_000e6`), `convert1to0` divides `poolExposure1` by `(conversionPrice / 2^96)^2`, yielding `100e18 * (1e18 / (2000 * 1e6)) = 5e26` (or, in the inverse case, dividing by `2000` to yield `0.05` instead of `200,000`).
5. At line 250, `nav += uint256(Math.max(poolExposure0 + poolExposure1, 0))` records the corrupted `poolExposure1` into `nav`.
6. When the manager calls `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals`, `totalAssets` is computed from `accountant.computeNAV(...)`, minting a severely inflated or deflated number of shares to depositors or transferring an incorrect amount of underlying assets to withdrawing users.

### Impact assessment

- **Severity**: High.
- **Technical Impact**: `PanopticVaultAccountant.computeNAV` computes an erroneous `poolExposure1` value in `underlyingToken` terms, directly corrupting the reported `nav`.
- **Business Impact**: Because `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals` rely directly on `accountant.computeNAV` to price vault shares, an over-reported `nav` causes withdrawing users to drain excess underlying assets from the vault and dilutes new depositors, while an under-reported `nav` mints excessive shares to new depositors at the expense of existing vault shareholders (or causes `totalAssets` calculation in `HypoVault` to underflow and revert).

### Remediation

Ensure that the oracle tick orientation and conversion function for `poolExposure1` match the convention used for `poolExposure0` (converting from the pool token into `underlyingToken` consistently):

```solidity
                uint160 conversionPrice = Math.getSqrtRatioAtTick(
                    pools[i].isUnderlyingToken0InOracle1 ? -conversionTick : conversionTick
                );

                poolExposure1 = PanopticMath.convert0to1(poolExposure1, conversionPrice);
```
