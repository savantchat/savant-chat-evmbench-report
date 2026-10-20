# Audit report — 2025-06-panoptic/H-01

## Finding 1: Aggregate vs. Per-User Rollover and Share Accounting Divergence in `HypoVault.fulfillDeposits`

### Synopsis

In `HypoVault.fulfillDeposits`, aggregate deposit rollover (`assetsRemaining`) and share issuance (`sharesReceived`) are computed at the epoch level, whereas `executeDeposit` computes fulfilled assets and shares per user with downward rounding. During partial fulfillments, this divergence desynchronizes `depositEpochState.assetsDeposited`, `totalSupply`, `totalAssets`, and per-user entitlements across epochs.

### Technical details

When deposits are fulfilled in `HypoVault.fulfillDeposits`, the contract performs aggregate accounting for `currentEpoch` and immediately initializes the next epoch's state:

```solidity
uint256 totalAssets = accountant.computeNAV(address(this), underlyingToken, managerInput) +
    1 -
    epochState.assetsDeposited -
    reservedWithdrawalAssets;

uint256 _totalSupply = totalSupply;

uint256 sharesReceived = Math.mulDiv(assetsToFulfill, _totalSupply, totalAssets);

uint256 assetsRemaining = epochState.assetsDeposited - assetsToFulfill;
```

It then sets `depositEpochState[currentEpoch + 1].assetsDeposited = uint128(assetsRemaining)` (line 501) and adds the full aggregate `sharesReceived` to `totalSupply` (line 506).

However, individual users claim their shares and roll over unfulfilled deposits asynchronously in `HypoVault.executeDeposit`:

```solidity
uint256 userAssetsDeposited = Math.mulDiv(
    queuedDepositAmount,
    _depositEpochState.assetsFulfilled,
    _depositEpochState.assetsDeposited
);

uint256 sharesReceived = Math.mulDiv(
    userAssetsDeposited,
    _depositEpochState.sharesReceived,
    _depositEpochState.assetsFulfilled
);

_mintVirtual(user, sharesReceived);
userBasis[user] += userAssetsDeposited;
queuedDeposit[user][epoch] = 0;

uint256 assetsRemaining = queuedDepositAmount - userAssetsDeposited;
if (assetsRemaining > 0) queuedDeposit[user][epoch + 1] += uint128(assetsRemaining);
```

Because `userAssetsDeposited` is rounded down via `Math.mulDiv` for each user $i$:
$$\text{userAssetsDeposited}_i = \left\lfloor \frac{\text{queuedDepositAmount}_i \times \text{assetsFulfilled}}{\text{assetsDeposited}} \right\rfloor$$

the per-user unfulfilled remainder rolled into `epoch + 1`:
$$\text{assetsRemaining}_i = \text{queuedDepositAmount}_i - \text{userAssetsDeposited}_i$$
is effectively rounded up.

When an epoch is partially fulfilled (`0 < assetsToFulfill < epochState.assetsDeposited`), this creates three compounding accounting inconsistencies:

1. **Understated Next-Epoch `assetsDeposited` vs. Sum of Rolled-Over `queuedDeposit`**:
   The sum of per-user rolled-over deposits satisfies:
   $$\sum_i \text{queuedDeposit}[\text{user}_i][\text{epoch} + 1] = \text{epochState.assetsDeposited} - \sum_i \text{userAssetsDeposited}_i > \text{epochState.assetsDeposited} - \text{assetsToFulfill}$$
   Thus, `depositEpochState[epoch + 1].assetsDeposited` is strictly less than the sum of `queuedDeposit[user][epoch + 1]` and strictly less than the actual unfulfilled deposit assets remaining in the vault.

2. **Phantom `totalSupply` Creation and Shareholder Dilution**:
   `fulfillDeposits` immediately increments `totalSupply` by the aggregate `sharesReceived` computed from `assetsToFulfill`. Because $\sum_i \text{userAssetsDeposited}_i < \text{assetsToFulfill}$ and the second `Math.mulDiv` for per-user `sharesReceived` also truncates, the total shares minted to users via `_mintVirtual` in `epoch` is strictly less than `depositEpochState[epoch].sharesReceived`. The unminted difference remains in `totalSupply` without belonging to any user's `balanceOf`, diluting existing shareholders during subsequent share-to-asset conversions in `fulfillWithdrawals`.

3. **NAV Distortion and Over-Allocation in Subsequent Epochs**:
   In `epoch + 1`, `fulfillDeposits` computes `totalAssets` at lines 480–483 by subtracting `epochState.assetsDeposited` from `accountant.computeNAV(...)`. Because `depositEpochState[epoch + 1].assetsDeposited` under-reports the unfulfilled deposits still held in the vault, `totalAssets` is artificially inflated by the untracked remainder (treating unfulfilled user principal as vault yield). Subsequently, when `epoch + 1` is fulfilled (for instance, with `assetsToFulfill = depositEpochState[epoch + 1].assetsDeposited`), `executeDeposit(user, epoch + 1)` divides each user's `queuedDeposit[user][epoch + 1]` by the understated `_depositEpochState.assetsDeposited`, crediting users with more `userAssetsDeposited`, `userBasis`, and `sharesReceived` than `depositEpochState[epoch + 1].assetsFulfilled` and `depositEpochState[epoch + 1].sharesReceived`.

### Assumptions and preconditions

1. One or more users have active deposit requests in `currentEpoch` (or an attacker splits a deposit across multiple accounts).
2. The manager partially fulfills the deposit epoch (`0 < assetsToFulfill < epochState.assetsDeposited`).

### Proof of concept

Consider the following numerical walkthrough:

1. **Initial State (Epoch 0)**:
   - Assume the vault has `totalSupply = 100e18` shares and `100e18` underlying assets already invested (`reservedWithdrawalAssets = 0`).
   - Honest user Alice requests a deposit of `50` underlying units (`queuedDeposit[Alice][0] = 50`).
   - User Bob (or an actor using 50 accounts `B_1 ... B_50`) requests a deposit of `1` underlying unit per account (`queuedDeposit[B_i][0] = 1` for $i \in [1, 50]$).
   - Aggregate `depositEpochState[0].assetsDeposited = 100`.

2. **Partial Fulfillment of Epoch 0**:
   - The manager calls `fulfillDeposits(50, managerInput)` to fulfill 50% of Epoch 0 (`assetsToFulfill = 50`).
   - `computeNAV` returns `200e18` (prior `100e18` + `100` deposited; for simplicity in unit scale, suppose 1 share = 1 asset so `sharesReceived = 50`).
   - `depositEpochState[0]` records `assetsDeposited = 100`, `assetsFulfilled = 50`, `sharesReceived = 50`.
   - `assetsRemaining = 100 - 50 = 50`.
   - `depositEpoch` advances to `1`, and `depositEpochState[1].assetsDeposited` is set to `50`.
   - `totalSupply` increases by `50`.

3. **Execution of Epoch 0 Deposits**:
   - Alice calls `executeDeposit(Alice, 0)`:
     - `userAssetsDeposited = Math.mulDiv(50, 50, 100) = 25`.
     - `sharesReceived = Math.mulDiv(25, 50, 50) = 25`.
     - Alice receives `25` shares, `userBasis[Alice] = 25`, and `assetsRemaining = 50 - 25 = 25` is added to `queuedDeposit[Alice][1]`.
   - Each account `B_i` calls `executeDeposit(B_i, 0)`:
     - `userAssetsDeposited = Math.mulDiv(1, 50, 100) = 0`.
     - `sharesReceived = 0`.
     - `assetsRemaining = 1 - 0 = 1` is added to `queuedDeposit[B_i][1]`.

4. **Resulting State Desynchronization in Epoch 1**:
   - In Epoch 0, `totalSupply` increased by `50`, but only `25` shares were minted to users (`25` phantom shares exist in `totalSupply` with no owner).
   - Only `25` of the `50` fulfilled assets were deducted from user deposit queues; `75` unfulfilled deposit assets remain queued (`queuedDeposit[Alice][1] = 25` and $\sum_{i=1}^{50} \text{queuedDeposit}[B_i][1] = 50$).
   - However, `depositEpochState[1].assetsDeposited` is only `50`.
   - When the manager fulfills Epoch 1 with `assetsToFulfill = 50` (100% of `depositEpochState[1].assetsDeposited`):
     - `totalAssets` in `fulfillDeposits` subtracts `epochState.assetsDeposited = 50` instead of `75`, overstating `totalAssets` by `25`.
     - Because `_depositEpochState.assetsFulfilled == _depositEpochState.assetsDeposited == 50` in Epoch 1, calling `executeDeposit` for Epoch 1 gives Alice `userAssetsDeposited = 25` and gives `B_1 ... B_50` a combined `userAssetsDeposited = 50`, totaling `75` fulfilled assets (`userBasis` sum increases by `75`) and minting `1.5x` the `sharesReceived` allocated to Epoch 1.

### Impact assessment

- **Severity**: Medium / High.
- **Technical Impact**:
  - `depositEpochState[epoch].assetsDeposited` becomes permanently desynchronized from the sum of `queuedDeposit[user][epoch]` after partial fulfillments.
  - `totalSupply` becomes desynchronized from the sum of user `balanceOf` mappings, violating the `ERC20Minimal` invariant (`sum(balanceOf) == totalSupply`).
  - `totalAssets` calculation in `fulfillDeposits` and `fulfillWithdrawals` is distorted because `depositEpochState[depositEpoch].assetsDeposited` does not match the actual unfulfilled deposit assets in the vault.
  - Users splitting deposits across accounts during partial fulfillments can avoid fulfillment in unfavorable epochs while shifting dilution onto other depositors and claiming a disproportionate share of subsequent epoch fulfillments.

### Remediation

To prevent divergence between aggregate epoch state and per-user balances, either:
1. Track user deposits via epoch share/receipt tokens or cumulative epoch multipliers so that partial fulfillment rollovers do not require per-user integer division before rolling into the next epoch; or
2. Account for the exact rolled-over amounts when `executeDeposit` is called (requiring prior epochs to be executed before rolling into active deposits, or tracking unexecuted rollover shares separately from current-epoch deposits) and mint/update `totalSupply` based on the actual shares minted in `executeDeposit` (while reserving unminted shares in a pending supply variable used in `fulfillDeposits` and `fulfillWithdrawals`).

For example, tracking `pendingShares` separately or updating `depositEpochState` and `totalSupply` consistently with executed amounts ensures that `totalSupply` and `depositEpochState.assetsDeposited` cannot diverge from per-user entitlements:

```solidity
        uint256 totalAssets = accountant.computeNAV(address(this), underlyingToken, managerInput) +
            1 -
            epochState.assetsDeposited -
            reservedWithdrawalAssets;

        uint256 _totalSupply = totalSupply;

        uint256 sharesReceived = Math.mulDiv(assetsToFulfill, _totalSupply, totalAssets);

        if (assetsToFulfill == 0 || (assetsToFulfill < epochState.assetsDeposited && sharesReceived == 0)) {
            revert DepositNotFulfillable();
        }

        uint256 assetsRemaining = epochState.assetsDeposited - assetsToFulfill;

        depositEpochState[currentEpoch] = DepositEpochState({
            assetsDeposited: uint128(epochState.assetsDeposited),
            sharesReceived: uint128(sharesReceived),
            assetsFulfilled: uint128(assetsToFulfill)
        });

        currentEpoch++;
        depositEpoch = uint128(currentEpoch);
```

*(Note: Full architectural remediation requires computing a user's multi-epoch deposit entitlement via a cumulative unfulfilled ratio product across epochs—similar to a MasterChef/index accumulator—rather than mutating and re-dividing per-epoch `queuedDeposit` balances on each partial rollover.)*

## Finding 2: Unsafe Downcast of `sharesReceived` to `uint128` in `HypoVault.fulfillDeposits` Truncates Allocated Shares While Inflating `totalSupply`

### Synopsis

In `HypoVault.fulfillDeposits`, the calculated `uint256 sharesReceived` is unsafely downcast to `uint128` when stored in `depositEpochState[currentEpoch].sharesReceived`, while the untruncated `uint256` value is added to `totalSupply`. When `sharesReceived` exceeds `type(uint128).max`, silent truncation permanently locks user funds by minting phantom supply.

### Technical details

In `HypoVault.fulfillDeposits` (`src/HypoVault.sol`), the shares minted for the fulfilled deposit amount are calculated as a `uint256`:

```solidity
uint256 totalAssets = accountant.computeNAV(address(this), underlyingToken, managerInput) +
    1 -
    epochState.assetsDeposited -
    reservedWithdrawalAssets;

uint256 _totalSupply = totalSupply;

uint256 sharesReceived = Math.mulDiv(assetsToFulfill, _totalSupply, totalAssets);
```

At line 493, `sharesReceived` is narrowed to `uint128` using an unsafe cast when writing `depositEpochState[currentEpoch]`:

```solidity
depositEpochState[currentEpoch] = DepositEpochState({
    assetsDeposited: uint128(epochState.assetsDeposited),
    sharesReceived: uint128(sharesReceived),
    assetsFulfilled: uint128(assetsToFulfill)
});
```

However, at line 506, `totalSupply` is incremented by the full, untruncated `uint256` value of `sharesReceived`:

```solidity
totalSupply = _totalSupply + sharesReceived;
```

In Solidity `^0.8.24`, explicit type conversions from `uint256` to `uint128` do not revert on overflow; instead, they silently discard the upper 128 bits (`sharesReceived mod 2^128`).

Crucially, `sharesReceived` can exceed `type(uint128).max` ($\approx 3.40 \times 10^{38}$) even with small, normal token amounts well within the protocol's $2^{126} - 1$ token bound:
1. At deployment, `totalSupply` starts at $10^6$ virtual shares and active vault assets are $0$, so `totalAssets` evaluates to $1$ on the initial deposit fulfillment. Fulfilling an initial deposit of $10\text{ ether}$ ($10^{19}$ wei) mints $10^{19} \times 10^6 / 1 = 10^{25}$ shares, bringing `totalSupply` to $\approx 10^{25}$.
2. If the vault's active positions subsequently suffer a loss (or expire worthless) such that active net assets drop to $0$ (making `totalAssets == 1` again), fulfilling a subsequent deposit of just $10\text{ ether}$ ($10^{19}$ wei) computes:
   $$\text{sharesReceived} = \frac{10^{19} \times 10^{25}}{1} = 10^{44} > 2^{128} - 1$$
3. Additionally, even on the very first deposit epoch where `totalAssets == 1` and `_totalSupply == 10^6`, any deposit fulfillment with `assetsToFulfill` $\ge 3.41 \times 10^{32}$ wei (which is $< 2^{109}$ and well below the $2^{126} - 1$ bound) produces `sharesReceived` $> 2^{128} - 1$.

When `sharesReceived > type(uint128).max`:
- `depositEpochState[currentEpoch].sharesReceived` stores only `uint128(sharesReceived)`.
- When depositors call `executeDeposit`, the shares minted to their balances via `_mintVirtual` are bounded by `depositEpochState[currentEpoch].sharesReceived` ($\le 2^{128} - 1$).
- Meanwhile, `totalSupply` increases by the full `uint256` `sharesReceived`. The difference `sharesReceived - uint128(sharesReceived)` (a multiple of $2^{128}$) becomes unowned phantom supply in `totalSupply`.
- Because `fulfillWithdrawals` prices redemptions as `Math.mulDiv(sharesToFulfill, totalAssets, totalSupply)`, the phantom `totalSupply` permanently locks the vast majority of the newly deposited assets in the vault.

### Assumptions and preconditions

- The vault fulfills deposits when the ratio of `_totalSupply` to `totalAssets` causes `assetsToFulfill * _totalSupply / totalAssets > type(uint128).max` (for instance, after an initial deposit establishes $\text{totalSupply} \ge 10^{25}$ and subsequent vault trading losses reduce active net assets near zero so `totalAssets` is small, or when a large deposit $< 2^{126} - 1$ is fulfilled against `totalAssets == 1`).

### Proof of concept

1. **Initial Deposit (Epoch 0):**
   - The vault starts with `totalSupply = 1_000_000` ($10^6$) and $0$ assets.
   - Alice calls `requestDeposit(10e18)` ($10\text{ ether}$).
   - The manager calls `fulfillDeposits(10e18, managerInput)` with `NAV = 10e18`.
   - `totalAssets = 10e18 + 1 - 10e18 - 0 = 1`.
   - `sharesReceived = (10e18 * 1e6) / 1 = 1e25`.
   - `totalSupply` becomes $10^{25} + 10^6$. Alice calls `executeDeposit(Alice, 0)` and receives $10^{25}$ shares.

2. **Vault Loss:**
   - The manager deploys the $10\text{ ether}$ into Panoptic options that expire out-of-the-money / lose value, leaving active vault assets at $0$.

3. **Second Deposit (Epoch 1):**
   - Bob calls `requestDeposit(10e18)` ($10\text{ ether}$).
   - The manager calls `fulfillDeposits(10e18, managerInput)` with `NAV = 10e18` (reflecting Bob's deposited tokens).
   - `totalAssets = 10e18 + 1 - 10e18 - 0 = 1`.
   - `sharesReceived = (10e18 * (1e25 + 1e6)) / 1 = 10^44 + 10^25`.
   - At line 493, `uint128(sharesReceived)` silently truncates `10^44 + 10^25` modulo $2^{128}$ to `2,661,564,242,916,331,941,866,620,800,950,935,552` ($\approx 2.66 \times 10^{36}$).
   - At line 506, `totalSupply` becomes $10^{44} + 2 \times 10^{25} + 10^6$.

4. **Loss of Funds on Execution:**
   - Bob calls `executeDeposit(Bob, 1)` and receives $\approx 2.66 \times 10^{36}$ shares.
   - Bob now holds only $\frac{2.66 \times 10^{36}}{10^{44}} \approx 0.00000266\%$ of `totalSupply`.
   - Over $99.999997\%$ of Bob's $10\text{ ether}$ deposit is permanently locked in the vault because the remaining $9.999997 \times 10^{43}$ shares in `totalSupply` do not belong to any user.

### Impact assessment

**Severity: High**

The silent truncation of `sharesReceived` to `uint128` combined with updating `totalSupply` using the untruncated `uint256` value breaks the fundamental accounting invariant that `totalSupply` equals the sum of virtual shares, user balances, and claimable epoch shares. Depositors in the affected epoch lose nearly 100% of their deposited assets, which become permanently stuck in the vault.

### Remediation

Use `Math.toUint128` to safely downcast `sharesReceived` (and `assetsToFulfill`) so that the transaction reverts instead of silently truncating accounting state if `sharesReceived` exceeds `type(uint128).max`:

```solidity
        depositEpochState[currentEpoch] = DepositEpochState({
            assetsDeposited: uint128(epochState.assetsDeposited),
            sharesReceived: Math.toUint128(sharesReceived),
            assetsFulfilled: Math.toUint128(assetsToFulfill)
        });
```

## Finding 3: Zero-Asset Fulfillment in `HypoVault.fulfillDeposits` Permanently Locks Queued Deposits and Corrupts Epoch Accounting

### Synopsis

In `HypoVault.fulfillDeposits`, advancing `depositEpoch` when `assetsToFulfill == 0` sets `depositEpochState[currentEpoch].assetsFulfilled` to `0` while rolling `epochState.assetsDeposited` into the next epoch. Subsequent calls to `executeDeposit` for that epoch revert with division by zero, permanently locking user deposits and desynchronizing aggregate epoch accounting.

### Technical details

In `HypoVault`, users request deposits in the active `depositEpoch` via `requestDeposit(assets)`, which increments both `queuedDeposit[msg.sender][currentEpoch]` and `depositEpochState[currentEpoch].assetsDeposited`.

When the manager calls `fulfillDeposits(assetsToFulfill, managerInput)` (`src/HypoVault.sol:472-509`), the function:
1. Computes `sharesReceived = Math.mulDiv(assetsToFulfill, _totalSupply, totalAssets)` (line 487).
2. Computes `assetsRemaining = epochState.assetsDeposited - assetsToFulfill` (line 489).
3. Records `depositEpochState[currentEpoch]` with `assetsFulfilled: uint128(assetsToFulfill)` and `sharesReceived: uint128(sharesReceived)` (lines 491-495).
4. Increments `currentEpoch` and updates `depositEpoch = uint128(currentEpoch)` (lines 497-498).
5. Initializes `depositEpochState[currentEpoch].assetsDeposited = uint128(assetsRemaining)` for the new epoch (lines 500-504).

However, `fulfillDeposits` permits `assetsToFulfill == 0` (a zero-fulfillment epoch transition, as exercised in `test_epoch_transitions_deposits` and `test_onlyManager_functions`). If `fulfillDeposits(0, managerInput)` is called when `epochState.assetsDeposited > 0`:
- `depositEpochState[currentEpoch].assetsFulfilled` is set to `0` (line 494).
- `depositEpoch` advances to `currentEpoch + 1` (lines 497-498).
- All unfulfilled deposits (`assetsRemaining = epochState.assetsDeposited`) are rolled over in aggregate into `depositEpochState[currentEpoch + 1].assetsDeposited` (line 501).

For any user with `queuedDeposit[user][currentEpoch] > 0`, their deposit is only rolled over to `queuedDeposit[user][currentEpoch + 1]` when `executeDeposit(user, currentEpoch)` is called (`src/HypoVault.sol:310-343`). However, `executeDeposit` computes:

```solidity
uint256 userAssetsDeposited = Math.mulDiv(
    queuedDepositAmount,
    _depositEpochState.assetsFulfilled,
    _depositEpochState.assetsDeposited
);

uint256 sharesReceived = Math.mulDiv(
    userAssetsDeposited,
    _depositEpochState.sharesReceived,
    _depositEpochState.assetsFulfilled
);
```

Because `_depositEpochState.assetsFulfilled` is `0`, the second `Math.mulDiv` call at line 324 divides by `_depositEpochState.assetsFulfilled == 0` and unconditionally reverts at `require(denominator > 0)` in `Math.mulDiv` (`lib/panoptic-v1.1/contracts/libraries/Math.sol:464`).

Consequently:
1. `executeDeposit(user, currentEpoch)` always reverts, so users can never roll their `queuedDeposit[user][currentEpoch]` over into `queuedDeposit[user][currentEpoch + 1]`.
2. `cancelDeposit(user)` cannot rescue the funds because it only cancels deposits in the latest `depositEpoch` (`currentEpoch + 1`), where `queuedDeposit[user][currentEpoch + 1]` is still `0`.
3. `depositEpochState[currentEpoch + 1].assetsDeposited` already includes `assetsRemaining` from `currentEpoch`. When the manager subsequently fulfills `currentEpoch + 1`, `totalSupply` is increased by the shares corresponding to those trapped assets, permanently inflating `totalSupply` with phantom shares that no user can ever claim.

### Assumptions and preconditions

1. At least one user has an active deposit in `currentEpoch` (either via `requestDeposit` in `currentEpoch` or rolled over from a prior epoch), so `depositEpochState[currentEpoch].assetsDeposited > 0`.
2. The manager calls `fulfillDeposits(0, managerInput)` to advance the deposit epoch without fulfilling deposits in that epoch.

### Proof of concept

1. In `depositEpoch = 0`, Alice calls `requestDeposit(100e18)`.
   - `queuedDeposit[Alice][0] = 100e18`
   - `depositEpochState[0].assetsDeposited = 100e18`
2. The manager calls `fulfillDeposits(0, managerInput)` to advance the epoch without fulfilling deposits.
   - `depositEpochState[0]` is stored as `{ assetsDeposited: 100e18, sharesReceived: 0, assetsFulfilled: 0 }`.
   - `depositEpoch` becomes `1`.
   - `depositEpochState[1]` is stored as `{ assetsDeposited: 100e18, sharesReceived: 0, assetsFulfilled: 0 }`.
3. Alice (or the manager) calls `executeDeposit(Alice, 0)` so Alice's unfulfilled `100e18` deposit can roll over to `queuedDeposit[Alice][1]`.
   - `userAssetsDeposited = Math.mulDiv(100e18, 0, 100e18) = 0`.
   - `sharesReceived = Math.mulDiv(0, 0, 0)` reverts in `Math.mulDiv` due to division by zero (`_depositEpochState.assetsFulfilled == 0`).
4. If the manager calls `cancelDeposit(Alice)` in epoch 1, `queuedDeposit[Alice][1]` is `0`, so `0` tokens are returned to Alice.
5. Alice's `100e18` underlying tokens are permanently locked in the contract, while `depositEpochState[1].assetsDeposited` remains inflated by `100e18`.

### Impact assessment

**Severity: High**

- **Permanent Freezing of User Funds:** All user deposits queued in an epoch that is transitioned with `assetsToFulfill == 0` become permanently unclaimable and un-cancellable.
- **Accounting & Total Supply Desynchronization:** Because `fulfillDeposits` rolls the aggregate `assetsRemaining` into `depositEpochState[currentEpoch + 1].assetsDeposited` while per-user `queuedDeposit` entries cannot roll over, fulfilling subsequent epochs mints shares into `totalSupply` for the trapped assets that can never be minted to any user via `_mintVirtual`.

### Remediation

Do not advance `depositEpoch` or record a fulfilled epoch when `assetsToFulfill == 0` (or revert if `assetsToFulfill == 0` in `fulfillDeposits`):

```solidity
    function fulfillDeposits(
        uint256 assetsToFulfill,
        bytes memory managerInput
    ) external onlyManager {
        if (assetsToFulfill == 0) return;

        uint256 currentEpoch = depositEpoch;

        DepositEpochState memory epochState = depositEpochState[currentEpoch];

        uint256 totalAssets = accountant.computeNAV(address(this), underlyingToken, managerInput) +
            1 -
            epochState.assetsDeposited -
            reservedWithdrawalAssets;

        uint256 _totalSupply = totalSupply;

        uint256 sharesReceived = Math.mulDiv(assetsToFulfill, _totalSupply, totalAssets);

        uint256 assetsRemaining = epochState.assetsDeposited - assetsToFulfill;

        depositEpochState[currentEpoch] = DepositEpochState({
            assetsDeposited: uint128(epochState.assetsDeposited),
            sharesReceived: uint128(sharesReceived),
            assetsFulfilled: uint128(assetsToFulfill)
        });

        currentEpoch++;
        depositEpoch = uint128(currentEpoch);

        depositEpochState[currentEpoch] = DepositEpochState({
            assetsDeposited: uint128(assetsRemaining),
            sharesReceived: 0,
            assetsFulfilled: 0
        });

        totalSupply = _totalSupply + sharesReceived;

        emit DepositsFulfilled(currentEpoch, assetsToFulfill, sharesReceived);
    }
```

## Finding 4: Desynchronization Between Aggregate Epoch Rollover in `HypoVault.fulfillDeposits` and Per-User Rollover in `HypoVault.executeDeposit` Breaks Epoch Accounting

### Synopsis

In `HypoVault.fulfillDeposits`, unfulfilled aggregate deposits (`assetsRemaining`) are eagerly rolled into the next epoch's `depositEpochState[currentEpoch].assetsDeposited`, whereas per-user unfulfilled deposits are lazily rolled over in `executeDeposit` using floor-divided fulfilled amounts. This desynchronizes aggregate and per-user liabilities, causing underflow reverts in `cancelDeposit` and virtual share supply mismatches.

### Technical details

When deposits in an epoch are partially fulfilled via `HypoVault.fulfillDeposits` (`0 < assetsToFulfill < epochState.assetsDeposited`), the contract updates the aggregate deposit liability for the next epoch (`currentEpoch = depositEpoch + 1`) at lines 489 and 500–504:

```solidity
uint256 assetsRemaining = epochState.assetsDeposited - assetsToFulfill;
...
currentEpoch++;
depositEpoch = uint128(currentEpoch);

depositEpochState[currentEpoch] = DepositEpochState({
    assetsDeposited: uint128(assetsRemaining),
    sharesReceived: 0,
    assetsFulfilled: 0
});

totalSupply = _totalSupply + sharesReceived;
```

However, individual users' unfulfilled deposits are rolled over into `queuedDeposit[user][epoch + 1]` asynchronously when `executeDeposit(user, epoch)` is called (lines 318–340):

```solidity
uint256 userAssetsDeposited = Math.mulDiv(
    queuedDepositAmount,
    _depositEpochState.assetsFulfilled,
    _depositEpochState.assetsDeposited
);

uint256 sharesReceived = Math.mulDiv(
    userAssetsDeposited,
    _depositEpochState.sharesReceived,
    _depositEpochState.assetsFulfilled
);

_mintVirtual(user, sharesReceived);

userBasis[user] += userAssetsDeposited;

queuedDeposit[user][epoch] = 0;

uint256 assetsRemaining = queuedDepositAmount - userAssetsDeposited;

if (assetsRemaining > 0) queuedDeposit[user][epoch + 1] += uint128(assetsRemaining);
```

This creates two compounding accounting inconsistencies between `depositEpochState[currentEpoch].assetsDeposited` (aggregate liability) and $\sum_{\text{user}} \text{queuedDeposit}[\text{user}][\text{currentEpoch}]$ (per-user liabilities):

1. **Cross-Epoch Rollover Conservation Failure (Rounding Direction Mismatch):**
   In `executeDeposit`, `userAssetsDeposited` is rounded down via `Math.mulDiv`. Consequently, each user's rolled-over remainder `assetsRemaining = queuedDepositAmount - userAssetsDeposited` is rounded **up**. Across $N$ users (or $N$ accounts controlled by an attacker), the sum of fulfilled amounts is $\sum f_i = F_e - \Delta$ (for $0 \le \Delta < N$), so the sum of rolled-over per-user deposits in `epoch + 1` is:
   $$\sum_{i=1}^N \text{queuedDeposit}[\text{user}_i][e + 1] = \sum_{i=1}^N (d_i - f_i) = (D_e - F_e) + \Delta = \text{depositEpochState}[e + 1].\text{assetsDeposited} + \Delta$$
   Thus, the sum of per-user queued deposits in `epoch + 1` exceeds `depositEpochState[epoch + 1].assetsDeposited`.

2. **Settlement-Order-Dependent State Inconsistency (Single-User Out-of-Order Execution):**
   Because `depositEpochState[currentEpoch].assetsDeposited` is updated eagerly in `fulfillDeposits` while `queuedDeposit[user][epoch + 1]` is updated lazily in `executeDeposit`, a user who has both a partially fulfilled deposit in `epoch` (with unfulfilled remainder $a$) and a new deposit $b$ in `epoch + 1` can wait until `epoch + 1` is also partially fulfilled and then call:
   - `executeDeposit(user, epoch + 1)` first (settling $b$ alone and setting `queuedDeposit[user][epoch + 1] = 0`),
   - `executeDeposit(user, epoch)` second (rolling $a$ into `queuedDeposit[user][epoch + 1]`), and
   - `executeDeposit(user, epoch + 1)` a second time (settling $a$).
   Because $\lfloor \frac{a \cdot F_{e+1}}{D_{e+1}} \rfloor + \lfloor \frac{b \cdot F_{e+1}}{D_{e+1}} \rfloor \le \lfloor \frac{(a + b) \cdot F_{e+1}}{D_{e+1}} \rfloor$, executing `epoch + 1` in two parts rounds `userAssetsDeposited` down twice and rounds `assetsRemaining` up twice, causing `queuedDeposit[user][epoch + 2] > depositEpochState[epoch + 2].assetsDeposited` even when there is only a single depositor in the vault.

Once $\sum_{\text{user}} \text{queuedDeposit}[\text{user}][\text{currentEpoch}] > \text{depositEpochState}[\text{currentEpoch}].\text{assetsDeposited}$:
- **Underflow in `cancelDeposit`:** Calling `cancelDeposit` for all users in `currentEpoch` (or for the single user in the out-of-order execution case) attempts to compute `depositEpochState[currentEpoch].assetsDeposited -= uint128(queuedDepositAmount)` (line 279), which reverts with an arithmetic underflow panic.
- **Virtual Share Supply and Deferred-Claim Mismatch:** When `currentEpoch` is subsequently fulfilled (`assetsFulfilled == assetsDeposited`), `fulfillDeposits` (line 506) only increments `totalSupply` by `sharesReceived` corresponding to `depositEpochState[currentEpoch].assetsDeposited`. However, when users call `executeDeposit` for `currentEpoch`, each user receives `userAssetsDeposited = queuedDepositAmount` and proportional shares. Because $\sum \text{queuedDepositAmount} > \text{depositEpochState}[\text{currentEpoch}].\text{assetsDeposited}$, the total `userBasis` credited and virtual shares minted via `_mintVirtual` in `currentEpoch` exceed the epoch's `assetsFulfilled` and `sharesReceived`. If the vault's share price decreased between `epoch` and `currentEpoch`, the excess shares minted in `currentEpoch` exceed the unminted rounding dust from `epoch`, resulting in $\sum_{\text{user}} \text{balanceOf}[\text{user}] > \text{totalSupply}$.

### Assumptions and preconditions

- At least one deposit epoch is partially fulfilled by the manager (`0 < assetsToFulfill < epochState.assetsDeposited`).
- Either two or more deposits exist in the partially fulfilled epoch (or a single user deposits in two consecutive partially fulfilled epochs and executes the later epoch before the earlier epoch) such that `Math.mulDiv(queuedDepositAmount, assetsFulfilled, assetsDeposited)` truncates a non-zero remainder.

### Proof of concept

#### Scenario A: Two Depositors in a Partially Fulfilled Epoch

1. **Epoch 0 Deposits:**
   - Alice calls `requestDeposit(100)`.
   - Bob calls `requestDeposit(100)`.
   - State: `queuedDeposit[Alice][0] = 100`, `queuedDeposit[Bob][0] = 100`, `depositEpochState[0].assetsDeposited = 200`.

2. **Epoch 0 Partial Fulfillment:**
   - Manager calls `fulfillDeposits(101, managerInput)`.
   - `assetsRemaining = 200 - 101 = 99`.
   - `depositEpoch` advances to `1`, and `depositEpochState[1].assetsDeposited` is set to `99` (line 501).

3. **Epoch 0 Execution (Rollover to Epoch 1):**
   - Alice calls `executeDeposit(Alice, 0)`:
     - `userAssetsDeposited = Math.mulDiv(100, 101, 200) = 50`.
     - `assetsRemaining = 100 - 50 = 50`.
     - `queuedDeposit[Alice][1] = 50`.
   - Bob calls `executeDeposit(Bob, 0)`:
     - `userAssetsDeposited = Math.mulDiv(100, 101, 200) = 50`.
     - `assetsRemaining = 100 - 50 = 50`.
     - `queuedDeposit[Bob][1] = 50`.
   - State in Epoch 1: `queuedDeposit[Alice][1] + queuedDeposit[Bob][1] = 100`, while `depositEpochState[1].assetsDeposited = 99`.

4. **Impact 1 — `cancelDeposit` Reverts:**
   - If the manager cancels Epoch 1 deposits via `cancelDeposit(Alice)` (reducing `depositEpochState[1].assetsDeposited` from `99` to `49`) and `cancelDeposit(Bob)`, the second call attempts `49 - 50` at line 279 and reverts with an arithmetic underflow.

5. **Impact 2 — Virtual Share Supply Desynchronization:**
   - Suppose instead the vault experiences a loss before Epoch 1 fulfillment so that 1 asset mints 2,000 shares in Epoch 1 (compared to 1,000 shares in Epoch 0).
   - In Epoch 0, `totalSupply` increased by `101 * 1,000 = 101,000` shares, while Alice and Bob received `50,000 + 50,000 = 100,000` shares.
   - In Epoch 1, the manager fulfills all `99` remaining assets via `fulfillDeposits(99, managerInput)`. `depositEpochState[1].sharesReceived = 99 * 2,000 = 198,000`, and `totalSupply` increases by `198,000` (net `totalSupply` increase across both epochs: `299,000` shares).
   - Alice and Bob call `executeDeposit(Alice, 1)` and `executeDeposit(Bob, 1)`. Because `assetsFulfilled == assetsDeposited == 99`, each receives `userAssetsDeposited = 50 * 99 / 99 = 50` and `sharesReceived = 50 * 198,000 / 99 = 100,000` shares (`200,000` shares total in Epoch 1).
   - Total shares minted to Alice and Bob across both epochs is `100,000 + 200,000 = 300,000`, which exceeds the `299,000` shares added to `totalSupply`.

#### Scenario B: Single Depositor Out-of-Order Execution

1. Alice deposits `100` in Epoch 0. Manager fulfills `50` in Epoch 0 (`depositEpochState[1].assetsDeposited = 50`).
2. Without calling `executeDeposit(Alice, 0)`, Alice deposits `50` in Epoch 1 (`queuedDeposit[Alice][1] = 50`, `depositEpochState[1].assetsDeposited = 100`).
3. Manager fulfills `51` in Epoch 1 (`depositEpoch = 2`, `depositEpochState[2].assetsDeposited = 49`).
4. In Epoch 2, Alice calls `executeDeposit(Alice, 1)` (`userAssetsDeposited = floor(50 * 51 / 100) = 25`, rolling `25` into `queuedDeposit[Alice][2]`), then calls `executeDeposit(Alice, 0)` (rolling `50` into `queuedDeposit[Alice][1]`), and then calls `executeDeposit(Alice, 1)` again (`userAssetsDeposited = floor(50 * 51 / 100) = 25`, rolling another `25` into `queuedDeposit[Alice][2]`).
5. Alice now has `queuedDeposit[Alice][2] = 50`, while `depositEpochState[2].assetsDeposited = 49`. Calling `cancelDeposit(Alice)` in Epoch 2 immediately reverts with underflow.

### Impact assessment

- **Severity:** High / Medium
- **Direct Technical Effects:**
  - `depositEpochState[currentEpoch].assetsDeposited` becomes smaller than the sum of `queuedDeposit[user][currentEpoch]`.
  - `cancelDeposit` reverts with an arithmetic underflow when attempting to cancel remaining deposits in subsequent epochs.
  - `executeDeposit` can mint more virtual shares (`balanceOf`) and credit more `userBasis` in subsequent epochs than the aggregate `sharesReceived` and `assetsFulfilled` accounted for in `fulfillDeposits`, breaking the invariant that total user share balances do not exceed `totalSupply` and causing downstream withdrawal fulfillment underflows.

### Remediation

Ensure that cross-epoch rollovers conserve the exact amounts between aggregate epoch state and per-user state, and enforce sequential epoch execution per user:
1. Round `userAssetsDeposited` **up** (or round the unfulfilled rollover `assetsRemaining` **down**: `assetsRemaining = Math.mulDiv(queuedDepositAmount, _depositEpochState.assetsDeposited - _depositEpochState.assetsFulfilled, _depositEpochState.assetsDeposited)`) so that the sum of rolled-over user deposits never exceeds `depositEpochState[currentEpoch].assetsDeposited`.
2. Alternatively, update `depositEpochState[epoch + 1].assetsDeposited` when `executeDeposit` rolls over `assetsRemaining` (or account for rounding dust in `fulfillDeposits` / `executeDeposit`) and require users to execute earlier epochs before executing later epochs.

```solidity
        currentEpoch++;
        depositEpoch = uint128(currentEpoch);

        depositEpochState[currentEpoch] = DepositEpochState({
            assetsDeposited: uint128(assetsRemaining),
            sharesReceived: 0,
            assetsFulfilled: 0
        });

        totalSupply = _totalSupply + sharesReceived;

        emit DepositsFulfilled(currentEpoch, assetsToFulfill, sharesReceived);
```

In `executeDeposit`, compute `assetsRemaining` using floor division so that per-user rollovers cannot exceed the aggregate `assetsRemaining` recorded in `depositEpochState[currentEpoch]`:

```solidity
        uint256 assetsRemaining = Math.mulDiv(
            queuedDepositAmount,
            _depositEpochState.assetsDeposited - _depositEpochState.assetsFulfilled,
            _depositEpochState.assetsDeposited
        );
        uint256 userAssetsDeposited = queuedDepositAmount - assetsRemaining;
```

## Finding 5: Inverted Token1 Premium Accounting in `PanopticVaultAccountant.computeNAV` Corrupts `HypoVault.fulfillWithdrawals` Valuation

### Synopsis

`HypoVault.fulfillWithdrawals` relies on `PanopticVaultAccountant.computeNAV` to determine `totalAssets` and calculate `assetsReceived`. However, `computeNAV` inverts `shortPremium` and `longPremium` when computing `poolExposure1`, treating token1 premium liabilities as assets and premium receivables as liabilities, which misprices withdrawals or causes `fulfillWithdrawals` to revert.

### Technical details

In `HypoVault.fulfillWithdrawals` (`src/HypoVault.sol`, lines 522–533), the vault queries the accountant to value the portfolio and determine the underlying assets owed for the fulfilled withdrawal shares:

```solidity
uint256 totalAssets = accountant.computeNAV(address(this), underlyingToken, managerInput) +
    1 -
    depositEpochState[depositEpoch].assetsDeposited -
    _reservedWithdrawalAssets;
...
uint256 assetsReceived = Math.mulDiv(sharesToFulfill, totalAssets, _totalSupply);
```

When `PanopticVaultAccountant.computeNAV` (`src/accountants/PanopticVaultAccountant.sol`, lines 127–136) queries `pools[i].pool.getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i])`, the Panoptic pool returns `(shortPremium, longPremium, positionBalanceArray)`:
- `shortPremium` represents the total premium owed **to** the vault's short legs (an asset/receivable for the vault), packed with `token0` in `rightSlot()` and `token1` in `leftSlot()`.
- `longPremium` represents the total premium owed **by** the vault's long legs (a liability/payable for the vault), packed with `token0` in `rightSlot()` and `token1` in `leftSlot()`.

For `token0` (`rightSlot`), `computeNAV` rightly subtracts the long premium owed by the vault from the short premium owed to the vault (lines 131–133):

```solidity
poolExposure0 =
    int256(uint256(shortPremium.rightSlot())) -
    int256(uint256(longPremium.rightSlot()));
```

However, for `token1` (`leftSlot`), the operands are reversed (lines 134–136):

```solidity
poolExposure1 =
    int256(uint256(longPremium.leftSlot())) -
    int256(uint256(shortPremium.leftSlot()));
```

Because `longPremium.leftSlot()` and `shortPremium.leftSlot()` are inverted:
1. Premium owed **by** the vault on long legs in `token1` increases `poolExposure1` and inflates `nav`.
2. Premium owed **to** the vault on short legs in `token1` decreases `poolExposure1` and deflates `nav`.

This introduces an accounting error of `2 * (longPremium.leftSlot() - shortPremium.leftSlot())` in `token1` exposure (prior to conversion and clamping at line 250). When `fulfillWithdrawals` executes, this directly corrupts `totalAssets` (line 522) and `assetsReceived` (line 533).

### Assumptions and preconditions

1. The `HypoVault` is configured with `PanopticVaultAccountant` as its `accountant`.
2. The vault holds active option positions in a `PanopticPool` that have non-zero accumulated `shortPremium` or `longPremium` in `token1` (`leftSlot()`), such that `shortPremium.leftSlot() != longPremium.leftSlot()`.
3. Users have requested withdrawals and the manager calls `HypoVault.fulfillWithdrawals`.

### Proof of concept

Consider a vault where `underlyingToken` is `token1` (and `token0` exposure is `0`), with `totalSupply = 1000e18` shares, `assetsDeposited = 0`, `reservedWithdrawalAssets = 0`, and total value in `token1` collateral and option liquidity equal to `1000e18`.

**Case 1: Vault has net payable `longPremium` in `token1` (NAV Overstatement)**
1. The vault holds long option positions in `pools[0]` that owe `longPremium.leftSlot() = 50e18` in `token1` and have `shortPremium.leftSlot() = 0`.
2. The true NAV of the vault is `1000e18 - 50e18 = 950e18`.
3. During `HypoVault.fulfillWithdrawals(100e18, maxAssetsReceived, managerInput)`, `accountant.computeNAV` computes `poolExposure1` starting at `+50e18` (lines 134–136) instead of `-50e18`, returning `nav = 1050e18`.
4. `fulfillWithdrawals` computes `totalAssets = 1050e18 + 1` (line 522) and allocates `assetsReceived = 105e18` (line 533) instead of `95e18`, overpaying withdrawing shareholders by `10e18` at the expense of remaining users.

**Case 2: Vault has net receivable `shortPremium` in `token1` (NAV Understatement)**
1. The vault holds short option positions in `pools[0]` that have earned `shortPremium.leftSlot() = 50e18` in `token1` and have `longPremium.leftSlot() = 0`.
2. The true NAV of the vault is `1000e18 + 50e18 = 1050e18`.
3. During `HypoVault.fulfillWithdrawals(100e18, maxAssetsReceived, managerInput)`, `accountant.computeNAV` computes `poolExposure1` starting at `-50e18` (lines 134–136) instead of `+50e18`, returning `nav = 950e18`.
4. `fulfillWithdrawals` computes `totalAssets = 950e18 + 1` and allocates `assetsReceived = 95e18` instead of `105e18`, shortchanging withdrawing users by `10e18` (or causing line 522–525 to underflow if `assetsDeposited + _reservedWithdrawalAssets` exceeds the artificially reduced `nav + 1`).

### Impact assessment

**Severity: High**

The inverted premium calculation directly corrupts the vault's Net Asset Value during normal operations without requiring any malicious input:
- **Direct loss of funds for users or vault:** Withdrawing users receive fewer assets than entitled when the vault has net positive `token1` short premium, or receive excess assets drained from remaining shareholders when the vault has net `token1` long premium debt.
- **Miscalculated performance fees:** Because `executeWithdrawal` computes performance fees from `assetsToWithdraw - withdrawnBasis`, inflated `assetsReceived` also causes inaccurate performance fee deductions.
- **Denial of Service on withdrawals:** Deflated `nav` can cause `accountant.computeNAV(...) + 1 - depositEpochState[depositEpoch].assetsDeposited - _reservedWithdrawalAssets` at lines 522–525 to underflow and revert.

### Remediation

Update `PanopticVaultAccountant.computeNAV` (`src/accountants/PanopticVaultAccountant.sol`, lines 134–136) so that `poolExposure1` subtracts `longPremium.leftSlot()` from `shortPremium.leftSlot()`, matching the sign convention used for `poolExposure0`:

```solidity
                poolExposure0 =
                    int256(uint256(shortPremium.rightSlot())) -
                    int256(uint256(longPremium.rightSlot()));
                poolExposure1 =
                    int256(uint256(shortPremium.leftSlot())) -
                    int256(uint256(longPremium.leftSlot()));
```

## Finding 6: Unchecked Net-Asset Subtraction Underflow in `HypoVault.fulfillWithdrawals` Blocks Withdrawals and Distorts Share Valuation

### Synopsis

In `HypoVault.fulfillWithdrawals`, `totalAssets` is computed by subtracting `depositEpochState[depositEpoch].assetsDeposited` and `reservedWithdrawalAssets` directly from `accountant.computeNAV(...) + 1`. When vault losses or negative pool exposure cause `computeNAV` to fall below reserved and deposited assets, the subtraction underflows or misprices redemptions, blocking withdrawal fulfillment.

### Technical details

In `src/HypoVault.sol`, `fulfillWithdrawals` computes the net `totalAssets` available to back outstanding shares on lines 521–525:

```solidity
uint256 _reservedWithdrawalAssets = reservedWithdrawalAssets;
uint256 totalAssets = accountant.computeNAV(address(this), underlyingToken, managerInput) +
    1 -
    depositEpochState[depositEpoch].assetsDeposited -
    _reservedWithdrawalAssets;
```

This arithmetic assumes that `accountant.computeNAV(address(this), underlyingToken, managerInput)` is always greater than or equal to `depositEpochState[depositEpoch].assetsDeposited + _reservedWithdrawalAssets`. However, this invariant breaks down in two ways:

1. **Negative Pool Exposure Absorbing Queued Deposits and Reserved Withdrawals:**
   In `PanopticVaultAccountant.computeNAV` (lines 184–202 and 250–258), if any configured pool includes `underlyingToken` as `token0` or `token1`, the vault's entire `underlyingToken.balanceOf(_vault)`—which holds both `depositEpochState[depositEpoch].assetsDeposited` and `reservedWithdrawalAssets`—is added directly into that pool's `poolExposure0` or `poolExposure1`, and `skipUnderlying` is set to `true` (line 256). At line 250, the pool's contribution to NAV is clamped via `uint256(Math.max(poolExposure0 + poolExposure1, 0))`. If that pool has negative option/premium exposure, part or all of `underlyingToken.balanceOf(_vault)` is offset against the negative exposure before clamping at `0`. Consequently:
   - Calling `requestDeposit(assets)` increases `depositEpochState[depositEpoch].assetsDeposited` by the full `assets`, while `computeNAV` increases by less than `assets` (or by `0` if net exposure remains non-positive).
   - Subtracting the full `depositEpochState[depositEpoch].assetsDeposited` and `_reservedWithdrawalAssets` at lines 524–525 causes `totalAssets` to either revert with an arithmetic underflow (`Panic(0x11)`) or be artificially depressed by the absorbed deposit amount, reducing `assetsReceived` at line 533 for all withdrawing users in the epoch.

2. **Portfolio Depreciation After Reserving Withdrawals or Deploying Capital:**
   When a withdrawal epoch is fulfilled, `reservedWithdrawalAssets` is incremented by `assetsReceived` (line 557). If the vault's remaining positions subsequently depreciate (or if assets are deployed via `manage` into instruments not fully captured by `computeNAV` or subject to conversion/redemption rounding in `previewRedeem` / `convert0to1` / `convert1to0`) before users execute their withdrawals, `computeNAV` can fall below `depositEpochState[depositEpoch].assetsDeposited + _reservedWithdrawalAssets`. Instead of flooring net assets at `0` (plus the `+ 1` virtual asset offset), lines 522–525 revert with an underflow (or evaluate `totalAssets` to `0` when `computeNAV + 1 == assetsDeposited + _reservedWithdrawalAssets`), rendering `fulfillWithdrawals` unusable.

### Assumptions and preconditions

- The vault has active withdrawal requests in the current `withdrawalEpoch` that the manager attempts to fulfill via `fulfillWithdrawals`.
- Either:
  - One of the vault's configured Panoptic pools containing `underlyingToken` has negative exposure (e.g., owed long premium or negative option value exceeding collateral in that pool), and a user has queued a deposit via `requestDeposit` or the vault holds `reservedWithdrawalAssets`; or
  - The vault's NAV reported by `accountant.computeNAV` has dropped below `depositEpochState[depositEpoch].assetsDeposited + reservedWithdrawalAssets` due to portfolio losses after previous withdrawals were reserved.

### Proof of concept

1. Assume `HypoVault` has two pools or positions: Pool A (which includes `underlyingToken` and currently has a net option/premium exposure of `-50e18` with `0` idle `underlyingToken` in the vault) and Pool B (which holds `200e18` worth of collateral). `accountant.computeNAV` returns `Math.max(-50e18, 0) + 200e18 = 200e18`.
2. Users hold vault shares and request withdrawals in epoch `0`.
3. Before the manager calls `fulfillWithdrawals`, an untrusted user calls `requestDeposit(50e18)`:
   - `underlyingToken.balanceOf(address(vault))` becomes `50e18`.
   - `depositEpochState[depositEpoch].assetsDeposited` becomes `50e18`.
   - In `PanopticVaultAccountant.computeNAV`, Pool A's exposure becomes `-50e18 + 50e18 = 0`, and `skipUnderlying` is `true`, so `computeNAV` still returns `200e18`.
4. When the manager calls `fulfillWithdrawals(sharesToFulfill, maxAssetsReceived, managerInput)`:
   - Line 522–525 computes `totalAssets = 200e18 + 1 - 50e18 - 0 = 150e18 + 1` instead of `200e18 + 1`, slashing `assetsReceived` (line 533) for all withdrawing users in epoch `0` by 25%.
5. Alternatively, if Pool B had `0` value (so `computeNAV` was `0`), calling `requestDeposit(50e18)` keeps `computeNAV` at `0` while `assetsDeposited` is `50e18`. Calling `fulfillWithdrawals` then computes `0 + 1 - 50e18`, reverting with arithmetic underflow (`Panic(0x11)`) and permanently blocking withdrawal fulfillment until the pool exposure recovers or the deposit is cancelled.

### Impact assessment

- **Severity:** Medium / High.
- **Technical Impact:**
  - Unchecked unsigned subtraction on lines 522–525 reverts with `Panic(0x11)` whenever `computeNAV + 1 < depositEpochState[depositEpoch].assetsDeposited + _reservedWithdrawalAssets`, causing a denial of service on `fulfillWithdrawals`.
  - When a pool containing `underlyingToken` has negative exposure, any queued deposit reduces `totalAssets` in `fulfillWithdrawals` dollar-for-dollar up to the negative exposure, permanently underpaying withdrawing shareholders (`assetsReceived` at line 533) when their shares are burned at line 555.

### Remediation

Floor the net asset calculation (`computeNAV - assetsDeposited - _reservedWithdrawalAssets`) at `0` before adding the `+ 1` virtual asset offset so that `totalAssets` cannot underflow below `1`:

```solidity
    function fulfillWithdrawals(
        uint256 sharesToFulfill,
        uint256 maxAssetsReceived,
        bytes memory managerInput
    ) external onlyManager {
        uint256 _reservedWithdrawalAssets = reservedWithdrawalAssets;
        uint256 nav = accountant.computeNAV(address(this), underlyingToken, managerInput);
        uint256 pendingAndReservedAssets = depositEpochState[depositEpoch].assetsDeposited +
            _reservedWithdrawalAssets;
        uint256 totalAssets = (
            nav > pendingAndReservedAssets ? nav - pendingAndReservedAssets : 0
        ) + 1;

        uint256 currentEpoch = withdrawalEpoch;

        WithdrawalEpochState memory epochState = withdrawalEpochState[currentEpoch];

        uint256 _totalSupply = totalSupply;

        uint256 assetsReceived = Math.mulDiv(sharesToFulfill, totalAssets, _totalSupply);
```

## Finding 7: Missing Duplicate Check on `tokenIds` in `PanopticVaultAccountant.computeNAV` Allows Manipulation of Vault NAV

### Synopsis

`PanopticVaultAccountant.computeNAV` validates the manager-supplied `tokenIds` list only by checking that each position has a non-zero balance and that the sum of legs equals `pool.numberOfLegs(_vault)`. Because duplicate `TokenId` entries are not rejected, a manager can duplicate positions and omit liabilities to arbitrarily manipulate NAV.

### Technical details

In `PanopticVaultAccountant.computeNAV` (`src/accountants/PanopticVaultAccountant.sol`), the vault manager provides `managerInput`, which decodes into `(ManagerPrices[] memory managerPrices, PoolInfo[] memory pools, TokenId[][] memory tokenIds)`. For each pool `i`, `computeNAV` values the vault's open Panoptic positions as follows:

1. At lines 127–129, `computeNAV` calls `pools[i].pool.getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i])`.
2. Inside `PanopticPool._calculateAccumulatedPremia`, the pool iterates through `positionIdList` (`tokenIds[i]`) without checking for duplicate `TokenId`s. For each element, it looks up `s_positionBalance[_vault][tokenId]`, appends it to `positionBalanceArray`, and adds the position's premia to `shortPremium` or `longPremium`.
3. At lines 131–136, `computeNAV` initializes `poolExposure0` and `poolExposure1` using the aggregated `shortPremium` and `longPremium`.
4. At lines 140–173, `computeNAV` loops through `tokenIds[i]`, verifies only that the position is active (`if (positionBalanceArray[j][1] == 0) revert IncorrectPositionList();` at line 141), aggregates each position's liquidity and exercised amounts into `poolExposure0` and `poolExposure1`, and accumulates the leg count `numLegs += positionLegs` (line 172).
5. At line 175, `computeNAV` checks `if (numLegs != pools[i].pool.numberOfLegs(_vault)) revert IncorrectPositionList();`.

Because neither `PanopticVaultAccountant.computeNAV` nor `PanopticPool.getAccumulatedFeesAndPositionsData` verifies that the `TokenId` elements in `tokenIds[i]` are unique, the aggregate leg-count check `numLegs == pools[i].pool.numberOfLegs(_vault)` does not guarantee position-set integrity.

If the vault holds multiple positions in `pools[i]`, the manager can repeat a valid `TokenId` multiple times in `tokenIds[i]` while omitting other open positions whose leg counts sum to the same number of legs. This results in:
- **Duplicate-position aggregation:** The duplicated position's premia, liquidity amounts, and exercised amounts are counted multiple times in `shortPremium`, `longPremium`, `poolExposure0`, and `poolExposure1`.
- **Incomplete liability coverage:** Omitted positions—such as short legs that are in-the-money (representing vault debt/liabilities) or long legs owing significant `longPremium`—are completely excluded from the NAV calculation.

In `HypoVault.fulfillDeposits` (line 480) and `HypoVault.fulfillWithdrawals` (line 522), `accountant.computeNAV` directly determines `totalAssets`, which governs the exchange rate between shares and underlying assets.

### Assumptions and preconditions

- The vault has at least two open positions in a Panoptic pool `pools[i]` (for example, two positions each having the same number of legs, or a combination of positions where a subset of leg counts can be summed using duplicates to match `numberOfLegs(_vault)`).
- The vault manager calls `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals` with a crafted `tokenIds[i]` array containing duplicate `TokenId` entries.

### Proof of concept

1. Suppose `_vault` has two open 1-leg positions in `pools[0]`:
   - Position $A$ (`tokenIdA`, 1 leg): A position with positive net exposure of $+100\text{e18}$ underlying tokens (e.g., an in-the-money long leg or a short leg with accumulated `shortPremium`).
   - Position $B$ (`tokenIdB`, 1 leg): A position with negative net exposure of $-80\text{e18}$ underlying tokens (e.g., a short leg liability or a long leg owing `longPremium`).
2. `pools[0].pool.numberOfLegs(_vault)` returns $2$. The true net exposure of the positions in `pools[0]` is $+100\text{e18} - 80\text{e18} = +20\text{e18}$.
3. When calling `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals`, the manager passes `tokenIds[0] = [tokenIdA, tokenIdA]` in `managerInput`.
4. In `PanopticVaultAccountant.computeNAV`:
   - `getAccumulatedFeesAndPositionsData(_vault, true, [tokenIdA, tokenIdA])` processes `tokenIdA` twice, doubling its premia and returning `positionBalanceArray` with two active entries for `tokenIdA`.
   - In the `for` loop (lines 140–173), `positionBalanceArray[0][1]` and `positionBalanceArray[1][1]` are both non-zero, so line 141 does not revert.
   - `tokenIdA`'s liquidity and exercised amounts are added twice to `poolExposure0` / `poolExposure1`, while `tokenIdB`'s $-80\text{e18}$ liability is completely omitted.
   - `numLegs` is computed as $1 + 1 = 2$, which matches `pools[0].pool.numberOfLegs(_vault) == 2` at line 175.
5. `computeNAV` reports a position exposure of $+200\text{e18}$ instead of $+20\text{e18}$ (or, conversely, if `tokenIds[0] = [tokenIdB, tokenIdB]` is passed, reports $-160\text{e18}$ instead of $+20\text{e18}$), corrupting `totalAssets` during deposit or withdrawal fulfillment.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** `PanopticVaultAccountant.computeNAV` computes an incorrect `nav` by double-counting duplicated positions and ignoring omitted positions (including liabilities).
- **Business Implications:** During `fulfillDeposits` and `fulfillWithdrawals` in `HypoVault`, an inflated `nav` mints fewer shares to depositors than they are entitled to and distributes excess underlying assets to withdrawing users (draining the vault at the expense of remaining shareholders), whereas a deflated `nav` dilutes existing shareholders by minting excess shares to depositors and underpays withdrawing users.

### Remediation

Validate that all `TokenId` entries in `tokenIds[i]` are strictly unique (for example, by checking for duplicates in the loop or requiring `tokenIds[i]` to be sorted in strictly ascending order) before aggregating their premia and exposures.

```solidity
            {
                LeftRightUnsigned shortPremium;
                LeftRightUnsigned longPremium;

                (shortPremium, longPremium, positionBalanceArray) = pools[i]
                    .pool
                    .getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i]);

                poolExposure0 =
                    int256(uint256(shortPremium.rightSlot())) -
                    int256(uint256(longPremium.rightSlot()));
                poolExposure1 =
                    int256(uint256(longPremium.leftSlot())) -
                    int256(uint256(shortPremium.leftSlot()));
            }

            uint256 numLegs;
            for (uint256 j = 0; j < tokenIds[i].length; j++) {
                for (uint256 m = j + 1; m < tokenIds[i].length; m++) {
                    if (TokenId.unwrap(tokenIds[i][j]) == TokenId.unwrap(tokenIds[i][m])) {
                        revert IncorrectPositionList();
                    }
                }
                if (positionBalanceArray[j][1] == 0) revert IncorrectPositionList();
```

## Finding 8: Inverted `shortPremium` and `longPremium` signs for `token1` in `PanopticVaultAccountant.computeNAV` miscalculates vault NAV

### Synopsis

In `PanopticVaultAccountant.computeNAV`, `poolExposure1` is computed as `longPremium.leftSlot() - shortPremium.leftSlot()` instead of `shortPremium.leftSlot() - longPremium.leftSlot()`. This inverts the accounting of `token1` option premia—treating premia owed by the vault as assets and premia owed to the vault as liabilities—leading to inaccurate NAV and mispriced vault shares.

### Technical details

In `PanopticVaultAccountant.sol`, `computeNAV` queries `pools[i].pool.getAccumulatedFeesAndPositionsData(_vault, true, tokenIds[i])` to retrieve the accumulated option premia and position data for `_vault`:

- `shortPremium` (`LeftRightUnsigned`): The total amount of premium owed *to* the vault's short legs (`rightSlot` = `currency0`, `leftSlot` = `currency1`). This represents value receivable by the vault and must be added to the vault's exposure.
- `longPremium` (`LeftRightUnsigned`): The total amount of premium owed *by* the vault's long legs (`rightSlot` = `currency0`, `leftSlot` = `currency1`). This represents a liability payable by the vault and must be subtracted from the vault's exposure.

For `poolExposure0` (`currency0`, stored in `rightSlot`), lines 131–133 properly add `shortPremium` and subtract `longPremium`:
```solidity
poolExposure0 =
    int256(uint256(shortPremium.rightSlot())) -
    int256(uint256(longPremium.rightSlot()));
```

However, for `poolExposure1` (`currency1`, stored in `leftSlot`), lines 134–136 reverse the operands:
```solidity
poolExposure1 =
    int256(uint256(longPremium.leftSlot())) -
    int256(uint256(shortPremium.leftSlot()));
```

Because `longPremium.leftSlot()` and `shortPremium.leftSlot()` are swapped, any `token1` premium owed by the vault increases `poolExposure1`, while any `token1` premium owed to the vault decreases `poolExposure1`. Consequently, `computeNAV` returns an over- or under-stated Net Asset Value whenever the vault has non-zero net `token1` premia, directly corrupting share minting in `HypoVault.fulfillDeposits` and asset payouts in `HypoVault.fulfillWithdrawals`.

### Assumptions and preconditions

1. A `HypoVault` uses `PanopticVaultAccountant` as its configured `accountant`.
2. The vault holds at least one open Panoptic position in a configured pool that has accumulated `token1` (`leftSlot`) short or long premia where `shortPremium.leftSlot() != longPremium.leftSlot()`.

### Proof of concept

1. Suppose a `HypoVault` with `underlyingToken = token1` (and `1000e18` collateral in `token1`) opens a long position in a Panoptic pool that accumulates `100e18` of `token1` premium owed *by* the vault (`longPremium.leftSlot() = 100e18`, `shortPremium.leftSlot() = 0`).
2. The true value of the vault's `token1` exposure from collateral and premia is `1000e18 - 100e18 = 900e18`.
3. When the manager calls `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals`, `PanopticVaultAccountant.computeNAV` executes lines 134–136:
   ```solidity
   poolExposure1 = int256(uint256(100e18)) - int256(uint256(0)) = +100e18;
   ```
4. Adding the `1000e18` collateral balance results in `poolExposure1 = 1100e18` instead of `900e18`, overstating the vault's NAV by `200e18` (`2 * longPremium.leftSlot()`).
5. Conversely, if the vault had a short position with `shortPremium.leftSlot() = 100e18` owed *to* the vault and `longPremium.leftSlot() = 0`, `poolExposure1` starts at `-100e18`, understating the vault's NAV by `200e18` (`2 * shortPremium.leftSlot()`).

### Impact assessment

**Severity: High**

The vault's NAV calculation is off by `2 * (longPremium.leftSlot() - shortPremium.leftSlot())` (converted to `underlyingToken`) for every pool.
- **When `longPremium.leftSlot() > shortPremium.leftSlot()`**: NAV is artificially inflated.Withdrawing users receive more underlying assets than their fair share at the expense of remaining vault shareholders, and new depositors receive fewer shares than they should.
- **When `shortPremium.leftSlot() > longPremium.leftSlot()`**: NAV is artificially depressed. New depositors mint an excessive number of shares (diluting existing shareholders), and withdrawing users receive fewer underlying assets than they are entitled to.

### Remediation

Swap `shortPremium.leftSlot()` and `longPremium.leftSlot()` in `PanopticVaultAccountant.computeNAV` so that `shortPremium` is added and `longPremium` is subtracted for `poolExposure1`:

```solidity
                poolExposure0 =
                    int256(uint256(shortPremium.rightSlot())) -
                    int256(uint256(longPremium.rightSlot()));
                poolExposure1 =
                    int256(uint256(shortPremium.leftSlot())) -
                    int256(uint256(longPremium.leftSlot()));
```

## Finding 9: Incorrect Oracle Tick Direction and Token Conversion in `PanopticVaultAccountant.computeNAV` Causes Mispricing of `token1` Exposure

### Synopsis

In `PanopticVaultAccountant.computeNAV`, converting `poolExposure1` into `underlyingToken` inconsistently binds the `oracle1` tick direction (`isUnderlyingToken0InOracle1`) and conversion helper (`convert1to0`), inverting the price conversion for `token1` exposures relative to the expected oracle configuration and miscalculating the vault's Net Asset Value (NAV).

### Technical details

In `PanopticVaultAccountant.computeNAV`, each pool's net exposure is tracked separately in `poolExposure0` (denominated in `pools[i].token0`) and `poolExposure1` (denominated in `pools[i].token1`). When either token is not the vault's `underlyingToken`, its exposure must be converted into `underlyingToken` using the configured TWAP oracle (`oracle0` for `token0`, `oracle1` for `token1`).

Both `oracle0` and `oracle1` are standalone `IV3CompatibleOracle` pools pairing the respective pool token against `underlyingToken`. Because the token ordering within `oracle0` or `oracle1` depends solely on the addresses of the paired tokens (and not on whether the asset was `token0` or `token1` in the `PanopticPool`), `PoolInfo` includes boolean flags `isUnderlyingToken0InOracle0` and `isUnderlyingToken0InOracle1` to indicate whether `underlyingToken` is `token0` in that oracle.

For `token0` (lines 225–229), `computeNAV` negates `conversionTick` when `isUnderlyingToken0InOracle0` is `true` and converts `poolExposure0` using `PanopticMath.convert0to1`:
```solidity
uint160 conversionPrice = Math.getSqrtRatioAtTick(
    pools[i].isUnderlyingToken0InOracle0 ? -conversionTick : conversionTick
);

poolExposure0 = PanopticMath.convert0to1(poolExposure0, conversionPrice);
```

However, for `token1` (lines 242–246), `computeNAV` flips the ternary condition on `isUnderlyingToken0InOracle1` (`conversionTick : -conversionTick`) and calls `PanopticMath.convert1to0`:
```solidity
uint160 conversionPrice = Math.getSqrtRatioAtTick(
    pools[i].isUnderlyingToken0InOracle1 ? conversionTick : -conversionTick
);

poolExposure1 = PanopticMath.convert1to0(poolExposure1, conversionPrice);
```

Because `oracle1` is an independent oracle pool between `pools[i].token1` and `underlyingToken` (and in `PanopticVaultAccountant` configurations/tests `isUnderlyingToken0InOracle1` is set under the assumption that `convert1to0` converts `token1` to `token0` without also inverting the tick ternary, or vice versa), flipping one without matching the oracle's configured token-to-underlying binding inverts the effective exchange rate applied to `poolExposure1` (multiplying by `token1 / underlyingToken` instead of `underlyingToken / token1`). Additionally, in line 238, `conversionTick` is validated against `managerPrices[i].token1Price` prior to applying the directional sign from `isUnderlyingToken0InOracle1`.

As a result, `poolExposure1` is valued at the inverse price when converted to `underlyingToken`, corrupting the `nav` returned to `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals`.

### Assumptions and preconditions

1. A `HypoVault` uses `PanopticVaultAccountant` to compute its NAV during `fulfillDeposits` or `fulfillWithdrawals`.
2. At least one pool in `pools` has `address(pools[i].token1) != underlyingToken` and a non-zero `poolExposure1` (from idle `token1` balances, `collateralToken1` shares, option positions, or accumulated premia).
3. The price between `pools[i].token1` and `underlyingToken` in `oracle1` is not 1:1 (`conversionTick != 0`).

### Proof of concept

1. Consider a vault where `underlyingToken` is USDC and a configured pool has `token0 = underlyingToken` (USDC) and `token1 = WETH`, with `oracle1` configured as in `test_computeNAV_exactCalculation_withExactPremiums` (`isUnderlyingToken0InOracle1 = false` while `underlyingToken` is `token0` in `oracle1`, or where the caller expects `convert1to0` to invert the conversion without flipping the ternary on `isUnderlyingToken0InOracle1`).
2. Suppose `conversionTick` from `oracle1` is `+2000` (representing a price ratio of `1.0001^2000 ≈ 1.2214`).
3. The vault holds `poolExposure1 = 100e18` of `token1`.
4. During `computeNAV`, line 243 evaluates `pools[i].isUnderlyingToken0InOracle1 ? conversionTick : -conversionTick` to `-2000`, and line 246 executes `PanopticMath.convert1to0(100e18, Math.getSqrtRatioAtTick(-2000))`, dividing by `1.0001^(-2000)` (i.e., multiplying by `1.2214` instead of dividing by `1.2214`), producing `~122.14e18` instead of `~81.87e18`.
5. When `HypoVault.fulfillDeposits` or `HypoVault.fulfillWithdrawals` is called, the over- or under-stated `totalAssets` causes an incorrect number of shares to be minted to depositors or an incorrect amount of underlying assets to be paid out to withdrawing users.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: `computeNAV` computes an erroneous `poolExposure1` value in `underlyingToken` whenever `pools[i].token1 != underlyingToken` and `conversionTick != 0`.
- **Business Implications**: Because `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals` directly rely on `accountant.computeNAV` to price vault shares, an inaccurate NAV causes depositors to receive too few or too many shares and withdrawers to receive an incorrect amount of `underlyingToken`, leading to direct loss of user or vault funds.

### Remediation

Ensure the oracle tick direction and conversion function for `token1` are consistent with `token0` and the definition of `isUnderlyingToken0InOracle1`:

```solidity
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

                poolExposure1 = PanopticMath.convert0to1(poolExposure1, conversionPrice);
            }
```

## Finding 10: Misclassification of Unrestricted Vault Balances as Loss-Bearing Pool Collateral in `PanopticVaultAccountant.computeNAV` Understates NAV and Breaks Epoch Fulfillment

### Synopsis

In `PanopticVaultAccountant.computeNAV`, the vault's unrestricted token balances are aggregated with Panoptic pool exposures before applying the limited-liability `Math.max(..., 0)` floor. Consequently, negative exposure in a Panoptic pool offsets the vault's idle token and underlying balances, understating NAV, distorting share pricing, and causing `fulfillDeposits`/`fulfillWithdrawals` to underflow.

### Technical details

In Panoptic, each `PanopticPool` operates with isolated collateral tracked by its `collateralToken0` and `collateralToken1` (`CollateralTracker`) contracts. As indicated by the comment at line 249 of `src/accountants/PanopticVaultAccountant.sol`:

```solidity
// debt in pools with negative exposure does not need to be paid back
nav += uint256(Math.max(poolExposure0 + poolExposure1, 0));
```

When a vault's option positions in a `PanopticPool` incur losses exceeding the collateral deposited into that pool's `CollateralTracker` contracts, the pool's net exposure becomes negative. Because liability in a `PanopticPool` is limited to the collateral deposited into that pool, negative exposure in a pool cannot claim unrestricted tokens held directly in the vault's wallet (`_vault`).

However, `PanopticVaultAccountant.computeNAV` adds the vault's direct wallet balances of `token0` and `token1` (lines 197–202) directly into `poolExposure0` and `poolExposure1` before converting them to `underlyingToken` (lines 215–247) and before applying the `Math.max(poolExposure0 + poolExposure1, 0)` floor at line 250:

```solidity
if (!skipToken0)
    poolExposure0 += address(pools[i].token0) ==
        address(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE)
        ? int256(address(_vault).balance)
        : int256(pools[i].token0.balanceOf(_vault));
if (!skipToken1) poolExposure1 += int256(pools[i].token1.balanceOf(_vault));
```

Furthermore, whenever `underlyingToken` is `token0` or `token1` in any configured pool, `underlyingToken` is recorded in `underlyingTokens` (lines 184–195), which sets `skipUnderlying = true` at lines 254–258:

```solidity
bool skipUnderlying = false;
for (uint256 i = 0; i < underlyingTokens.length; i++) {
    if (underlyingTokens[i] == underlyingToken) skipUnderlying = true;
}
if (!skipUnderlying) nav += IERC20Partial(underlyingToken).balanceOf(_vault);
```

This creates two severe accounting flaws when `pools[i]` has negative net pool exposure (i.e., when option obligations exceed `collateralToken0` and `collateralToken1` redemptions):

1. **Unrestricted Vault Assets Absorbed by Pool Bad Debt:** The vault's idle ERC20/native balances (`token0.balanceOf(_vault)`, `token1.balanceOf(_vault)`, `address(_vault).balance`) are netted against the negative Panoptic pool exposure prior to `Math.max(poolExposure0 + poolExposure1, 0)` at line 250. These unrestricted balances are treated as if they were loss-bearing collateral pledged to `pools[i]`, artificially reducing `nav`.
2. **Underlying Token & Queued Deposit/Withdrawal Reserves Deducted from NAV:** When `underlyingToken` is one of the tokens in `pools[i]`, `IERC20Partial(underlyingToken).balanceOf(_vault)` is added into `poolExposure0` or `poolExposure1` (and skipped at line 258 because `skipUnderlying` is `true`). Note that `underlyingToken.balanceOf(_vault)` holds uninvested vault funds as well as `epochState.assetsDeposited` (pending deposits) and `reservedWithdrawalAssets` (fulfilled withdrawals awaiting execution). When negative pool exposure reduces or zeroes out `poolExposure0 + poolExposure1` at line 250:
   - `computeNAV` understates the vault's actual `underlyingToken` holdings.
   - In `HypoVault.fulfillDeposits` (lines 480–483) and `HypoVault.fulfillWithdrawals` (lines 522–525), the vault computes:
     ```solidity
     uint256 totalAssets = accountant.computeNAV(address(this), underlyingToken, managerInput) +
         1 -
         epochState.assetsDeposited -
         reservedWithdrawalAssets;
     ```
     If the negative pool exposure consumes part of `assetsDeposited + reservedWithdrawalAssets` inside `computeNAV`, `computeNAV(...) + 1` becomes less than `epochState.assetsDeposited + reservedWithdrawalAssets`, causing an arithmetic underflow that reverts `fulfillDeposits` and `fulfillWithdrawals`. Even if it does not underflow, `totalAssets` is understated, minting excessive shares to incoming depositors and underpaying withdrawing users.

### Assumptions and preconditions

1. The vault has at least one configured `PanopticPool` in `PanopticVaultAccountant` where the vault holds option positions that become underwater by more than the collateral deposited in that pool's `CollateralTracker` contracts (creating negative net pool exposure).
2. The vault holds a non-zero direct balance of `token0`, `token1`, or `underlyingToken` (for example, unallocated assets, pending `assetsDeposited`, or `reservedWithdrawalAssets`).

### Proof of concept

Consider a `HypoVault` with `underlyingToken = USDC` and a single configured Panoptic pool `pools[0]` for `USDC / WETH` (`token0 = USDC`, `token1 = WETH`), where 1 WETH = 2,000 USDC:

1. **Initial State:**
   - The vault has `totalSupply = 1,000e6` shares.
   - The vault holds `1,000e6` USDC idle in the vault contract (`USDC.balanceOf(vault) = 1,000e6`).
   - Users request deposits of `500e6` USDC in the current deposit epoch (`epochState.assetsDeposited = 500e6`), bringing `USDC.balanceOf(vault)` to `1,500e6`.
   - In `pools[0]`, the vault deposited `200e6` USDC into `collateralToken0`, and sold options that are now underwater by `-1,400e6` USDC (so the pool's true net exposure is `200e6 - 1,400e6 = -1,200e6` USDC, which is floored at `0` under limited liability).

2. **Expected NAV Calculation:**
   - Pool 0 exposure: `max(200e6 - 1,400e6, 0) = 0`.
   - Vault direct USDC balance: `1,500e6` USDC.
   - Expected `nav = 1,500e6`, and in `fulfillDeposits`, `totalAssets = 1,500e6 + 1 - 500e6 - 0 = 1,000e6 + 1`.

3. **Actual NAV Calculation in `computeNAV`:**
   - `poolExposure0` starts at `-1,400e6` (from the short option obligations).
   - At line 201, `USDC.balanceOf(vault) = 1,500e6` is added to `poolExposure0`, making `poolExposure0 = +100e6`.
   - At line 205, `collateralToken0` (`200e6`) is added, making `poolExposure0 = +300e6`.
   - At line 250, `nav += uint256(Math.max(300e6 + 0, 0)) = 300e6`.
   - At lines 254–258, `skipUnderlying` is `true` because `USDC` is in `underlyingTokens[0]`, so `USDC.balanceOf(vault)` is not added at line 258.
   - `computeNAV` returns `300e6` instead of `1,500e6`.

4. **Impact on `fulfillDeposits` / `fulfillWithdrawals`:**
   - When the manager calls `fulfillDeposits`, `totalAssets` is computed as:
     `300e6 + 1 - 500e6 - 0`, which reverts with an arithmetic underflow panic, blocking deposit and withdrawal fulfillments.
   - If `epochState.assetsDeposited` were `200e6` instead (so `USDC.balanceOf(vault) = 1,200e6` and `computeNAV = 0` or `200e6`), `totalAssets` would be understated by up to `1,000e6`, allowing new depositors to mint vastly inflated shares at the expense of existing vault shareholders and slashing withdrawing shareholders.

### Impact assessment

- **Severity:** High
- **Technical Impact:** `PanopticVaultAccountant.computeNAV` misclassifies the vault's unrestricted ERC20/native balances (including `underlyingToken`) as collateral subject to Panoptic pool losses. This understates `nav` whenever any pool has negative exposure and causes arithmetic underflow reverts in `HypoVault.fulfillDeposits` and `HypoVault.fulfillWithdrawals` when the understated `nav` falls below `assetsDeposited + reservedWithdrawalAssets`.
- **Business Impact:** Existing LPs suffer severe dilution during `fulfillDeposits` or loss of funds during `fulfillWithdrawals` due to artificially depressed share price, or vault operations (deposits and withdrawals) are completely frozen due to underflow in `totalAssets`.

### Remediation

Track and convert the vault's unrestricted token balances (`token0.balanceOf(_vault)`, `token1.balanceOf(_vault)`, and `address(_vault).balance`) separately from the Panoptic pool's position and collateral exposures (`poolExposure0` and `poolExposure1`). Apply `Math.max(poolExposure0 + poolExposure1, 0)` strictly to the pool's exposure, and add the converted unrestricted wallet balances to `nav` unconditionally.

```solidity
            int256 tokenBalance0;
            int256 tokenBalance1;
            if (!skipToken0)
                tokenBalance0 = address(pools[i].token0) ==
                    address(0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE)
                    ? int256(address(_vault).balance)
                    : int256(pools[i].token0.balanceOf(_vault));
            if (!skipToken1) tokenBalance1 = int256(pools[i].token1.balanceOf(_vault));

            uint256 collateralBalance = pools[i].pool.collateralToken0().balanceOf(_vault);
            poolExposure0 += int256(
                pools[i].pool.collateralToken0().previewRedeem(collateralBalance)
            );

            collateralBalance = pools[i].pool.collateralToken1().balanceOf(_vault);
            poolExposure1 += int256(
                pools[i].pool.collateralToken1().previewRedeem(collateralBalance)
            );

            // convert position and token values to underlying
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
                tokenBalance0 = PanopticMath.convert0to1(tokenBalance0, conversionPrice);
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
                tokenBalance1 = PanopticMath.convert1to0(tokenBalance1, conversionPrice);
            }

            // debt in pools with negative exposure does not need to be paid back
            nav +=
                uint256(Math.max(poolExposure0 + poolExposure1, 0)) +
                uint256(tokenBalance0 + tokenBalance1);
        }

        // underlying cannot be native (0x000/0xeee)
        bool skipUnderlying = false;
        for (uint256 i = 0; i < underlyingTokens.length; i++) {
            if (underlyingTokens[i] == underlyingToken) skipUnderlying = true;
        }
        if (!skipUnderlying) nav += IERC20Partial(underlyingToken).balanceOf(_vault);
```
