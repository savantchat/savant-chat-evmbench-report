# Audit report — 2025-02-thorwallet/H-01

## Finding 1: Unweighted `claimableTitn` Accounting in `MergeTgt.onTokenTransfer` and `MergeTgt.claimTitn` Enables Last-Minute Residual Pool Sniping and Strands Unclaimed `TITN`

### Synopsis

In `MergeTgt.onTokenTransfer` and `MergeTgt.claimTitn`, `claimableTitnPerUser` and `totalTitnClaimable` track only currently unclaimed `TITN` without time-weighting or deposit-window restrictions. Consequently, early claimants forfeit residual `TITN` (stranding the pool if all claim), while last-second depositors before `360 days` can snipe the 1-year residual `TITN` allocation.

### Technical details

`MergeTgt` is designed to allow `TGT` holders to merge their tokens for `TITN` over a `360 days` window and, after `360 days`, distribute any remaining unallocated `TITN` (`unclaimedTitn`) proportionally via `withdrawRemainingTitn()` to users who left their `TITN` in the contract for the 12-month period.

However, the way `claimableTitnPerUser` and `totalTitnClaimable` are updated in `onTokenTransfer` (lines 81–83) and `claimTitn` (lines 96–109) creates a severe claim-timing vulnerability for residual allocations:

1. **Pre-expiry claims reduce the residual allocation denominator (`claimTitn`, lines 103–107):**
   When users follow the documented merge workflow (`transferAndCall` followed by `claimTitn` before `360 days`), `claimTitn` decrements `claimableTitnPerUser[msg.sender]` (line 104) and `totalTitnClaimable` (line 107). Even though `claimedTitnPerUser` and `totalTitnClaimed` are incremented (lines 103, 106), they are never used in `withdrawRemainingTitn()`.
   - If all users who merged `TGT` claim their `TITN` via `claimTitn` before `360 days`, `totalTitnClaimable` and every user's `claimableTitnPerUser` become `0`. At `>= 360 days`, every call to `withdrawRemainingTitn()` reverts with `"No claimable TITN"` (and `initialTotalClaimable` would be `0`), stranding all unallocated `TITN` in the contract.

2. **Late deposits accrue into the same `claimableTitnPerUser` / `totalTitnClaimable` pool right up to expiry (`onTokenTransfer`, lines 81–83):**
   In `onTokenTransfer`, any deposit made before `360 days` increments `claimableTitnPerUser[from] += titnOut` and `totalTitnClaimable += titnOut` without recording when the deposit was made or requiring the tokens to be held in the contract for 12 months.
   - At `launchTime + 360 days - 1`, `totalTitnClaimable` and the contract's remaining `TITN` balance are publicly visible on-chain.
   - If all prior depositors have claimed their `TITN` via `claimTitn` (`totalTitnClaimable == 0`), an attacker can deposit a tiny amount of `TGT` at `launchTime + 360 days - 1` (just enough for `quoteTitn(amount) >= 1 wei`) and, 1 second later at `launchTime + 360 days`, call `withdrawRemainingTitn()` to drain 100% of `unclaimedTitn`.
   - Even if honest users left their `TITN` unclaimed from Day 1, an attacker can deposit right before the 90-day cliff (receiving the full undecayed rate while locking for 90 fewer days) or right before `360 days` (whenever `unclaimedTitn / totalTitnClaimable` exceeds the linear decay factor) to dilute honest 12-month depositors and capture a disproportionate share of the residual `TITN` pool.

### Assumptions and preconditions

1. `MergeTgt` has been initialized by the owner with `launchTime > 0`, `lockedStatus != LockedStatus.Locked`, and funded with `TITN_ARB` (`173,700,000e18` `TITN`).
2. Fewer than `TGT_TO_EXCHANGE` (`579,000,000e18` `TGT`) are merged at the full rate during the 360-day period, leaving a positive unallocated `unclaimedTitn` balance at `launchTime + 360 days`.

### Proof of concept

Consider the following scenario:

1. During the first 90 days, honest users merge `100,000,000e18` `TGT` via `onTokenTransfer`, accruing `30,000,000e18` `TITN` in `claimableTitnPerUser` and `totalTitnClaimable`.
2. Following the documented user flow, those users call `claimTitn` before `launchTime + 360 days` to receive their `30,000,000e18` `TITN` and bridge it to Base. After their claims:
   - `totalTitnClaimed = 30,000,000e18`
   - `totalTitnClaimable = 0`
   - `titn.balanceOf(address(mergeTgt)) = 143,700,000e18`
3. At `block.timestamp = launchTime + 360 days - 1` (`remainingtime = 1`), an attacker merges `100e18` `TGT` via `onTokenTransfer` (lines 81–83):
   - `titnOut = (100e18 * 173_700_000e18 * 1) / (579_000_000e18 * 270 days) = 1,286,931,599,229` wei (`~1.28e-6` `TITN`).
   - `claimableTitnPerUser[attacker] = 1,286,931,599,229`
   - `totalTitnClaimable = 1,286,931,599,229`
4. One second later, at `block.timestamp = launchTime + 360 days`, the attacker calls `withdrawRemainingTitn()`:
   - `remainingTitnAfter1Year = 143,700,000e18`
   - `initialTotalClaimable = 1,286,931,599,229`
   - `unclaimedTitn = 143,700,000e18 - 1,286,931,599,229`
   - `userProportionalShare = (initialTotalClaimable * unclaimedTitn) / initialTotalClaimable = unclaimedTitn`
   - `titnOut = 143,700,000e18`
5. The attacker receives the entire `143,700,000` remaining `TITN` by locking `100` `TGT` for 1 second.
6. Alternatively, if the honest users in step 1 had left `300e18` `TITN` unclaimed for the full 360 days (`totalTitnClaimable = 300e18`), the attacker could deposit `270,000e18` `TGT` at `launchTime + 359 days` (`titnOut = 300e18`) and 1 day later claim 50% (`71,849,850` `TITN`) of the residual pool, diluting the 12-month depositors by half while locking tokens for only 1 day.

### Impact assessment

- **Severity:** High / Medium
- **Direct Technical Effect:** The residual `TITN` distribution at `360 days` depends purely on who holds unclaimed `claimableTitnPerUser` at the exact `360 days` boundary, regardless of when the deposit was made or whether other depositors claimed their base allocation earlier.
- **Business Implications:**
  - Honest users who claim their base `TITN` during the year forfeit all residual rewards, and if all users claim before `360 days`, the residual `TITN` becomes unclaimable by users.
  - Opportunistic actors can wait until the end of the merge window to inspect on-chain state and deposit `TGT` right before `360 days` to capture most or all of the remaining `TITN` at the expense of long-term participants.

### Remediation

Track each user's eligible share for the 1-year residual distribution separately (e.g., either only credit residual-pool shares for deposits made during the initial 90-day window that remain unclaimed for the full duration, or weight shares by lock duration, or base residual distribution on total merged `claimedTitnPerUser[user] + claimableTitnPerUser[user]` relative to `totalTitnClaimed + totalTitnClaimable`).

If residual `TITN` is only intended for users who deposit during the initial 90-day window and leave their allocation unclaimed until `360 days`, restrict `claimTitn` or track residual-eligible balances separately; alternatively, if all merged users should share residual `TITN` proportionally to their merged allocation regardless of when they claimed their base `TITN`, keep their accrued share in `totalTitnClaimable` / `claimableTitnPerUser` for residual accounting or disallow last-minute residual dilution:

```solidity
        uint256 titnOut = quoteTitn(amount);
        if (titnOut == 0) {
            revert ZeroAmount();
        }
        claimableTitnPerUser[from] += titnOut;
        totalTitnClaimable += titnOut;

        emit ClaimableTitnUpdated(from, titnOut);
    }

    function tgtBalance() external view returns (uint256) {
        return tgt.balanceOf(address(this));
    }

    function titnBalance() external view returns (uint256) {
        return titn.balanceOf(address(this));
    }

    function claimTitn(uint256 amount) external nonReentrant {
        require(amount <= claimableTitnPerUser[msg.sender], "Not enough claimable titn");

        if (block.timestamp - launchTime >= 360 days) {
            revert TooLateToClaimRemainingTitn();
        }

        // Prevent partial claims that leave dust to snipe residual allocations,
        // and require users who want the 1-year residual share to lock during the initial window
        claimedTitnPerUser[msg.sender] += amount;
        claimableTitnPerUser[msg.sender] -= amount;

        totalTitnClaimed += amount;
        totalTitnClaimable -= amount;

        titn.safeTransfer(msg.sender, amount);
```

## Finding 2: Missing TGT Token Pull in `MergeTgt.onTokenTransfer` Allows Draining `TITN` Reserves via `claimTitn`

### Synopsis

In `MergeTgt.onTokenTransfer`, the contract credits `claimableTitnPerUser[from]` and `totalTitnClaimable` without pulling the `TGT` tokens from `from` via `tgt.safeTransferFrom(from, address(this), amount)`. Consequently, users can repeatedly invoke the callback and call `MergeTgt.claimTitn` to drain all `TITN` tokens from the contract without surrendering `TGT`.

### Technical details

The `MergeTgt` contract is designed to exchange `TGT` tokens for `TITN` tokens on a two-step flow documented in `README.md`:
1. The user approves `MergeTgt` to spend their `TGT` (`tgt.approve(MERGE_TGT_ADDRESS, amountToDeposit)`).
2. The user calls `tgt.transferAndCall(MERGE_TGT_ADDRESS, amountToDeposit, 0x)`, which invokes `MergeTgt.onTokenTransfer(address from, uint256 amount, bytes calldata extraData)`.
3. The user calls `MergeTgt.claimTitn(claimableAmount)` to withdraw their credited `TITN`.

In THORWallet's `TGT` token implementation, `transferAndCall` invokes `ITokenReceiver(to).onTokenTransfer(msg.sender, value, data)` without moving balances directly, relying on the receiver contract to pull the pre-approved `TGT` tokens via `transferFrom` (which is why both `README.md` and `test/hardhat/MergeTgt.test.ts` explicitly call `tgt.approve(mergeTgt.address, amount)` prior to `tgt.transferAndCall(...)`).

However, in `MergeTgt.onTokenTransfer` (`contracts/MergeTgt.sol`), right below the `// tgt in, titn out` comment on line 79, the `"tgt in"` token pull (`tgt.safeTransferFrom(from, address(this), amount)`) is omitted:

```solidity
        // tgt in, titn out

        uint256 titnOut = quoteTitn(amount);
        claimableTitnPerUser[from] += titnOut;
        totalTitnClaimable += titnOut;

        emit ClaimableTitnUpdated(from, titnOut);
```

Because `onTokenTransfer` credits `claimableTitnPerUser[from]` and `totalTitnClaimable` (lines 81–83) without transferring `amount` of `tgt` into `MergeTgt` or verifying actual token receipt, an attacker can obtain arbitrary `claimableTitnPerUser` credit without spending `TGT`, and then call `claimTitn` (lines 96–109) to transfer `TITN` out via `titn.safeTransfer(msg.sender, amount)` (line 109).

### Assumptions and preconditions

1. `MergeTgt` has been initialized by the owner with `launchTime > 0`, `lockedStatus != LockedStatus.Locked`, and funded with `173,700,000 * 10 ** 18` `TITN` via `deposit()`.
2. The merger period is active (`block.timestamp - launchTime < 360 days`).
3. The `TGT` token callback (`onTokenTransfer`) is invoked without transferring `TGT` inside `TGT.transferAndCall` itself, expecting `MergeTgt` to pull the approved `amount` from `from`.

### Proof of concept

1. Assume `MergeTgt` holds `173,700,000e18` `TITN` within the first 90 days after `launchTime`.
2. An attacker calls `tgt.transferAndCall( address(mergeTgt), 579_000_000e18, "0x" )`.
3. `tgt` calls `MergeTgt.onTokenTransfer(attacker, 579_000_000e18, "0x")`.
4. `MergeTgt.onTokenTransfer` validates `msg.sender == address(tgt)` and computes `titnOut = quoteTitn(579_000_000e18) = 173_700_000e18` at line 81.
5. `MergeTgt.onTokenTransfer` increments `claimableTitnPerUser[attacker]` and `totalTitnClaimable` by `173_700_000e18` at lines 82–83 without pulling any `TGT` from `attacker`.
6. The attacker calls `MergeTgt.claimTitn(173_700_000e18)` (line 96), which transfers the entire `173,700,000e18` `TITN` balance of `MergeTgt` to the attacker at line 109 (`titn.safeTransfer(msg.sender, amount)`), while the attacker retains 100% of their `TGT`.

### Impact assessment

- **Severity:** Critical / High
- **Direct Technical Impact:** `claimableTitnPerUser` and `totalTitnClaimable` can be inflated without corresponding `TGT` deposits into `MergeTgt`, allowing all `173,700,000` `TITN` tokens in `MergeTgt` to be drained via `claimTitn` (or causing `withdrawRemainingTitn` to underflow when `initialTotalClaimable > remainingTitnAfter1Year`).
- **Business Impact:** Total loss of the `173.7M TITN` reserve allocated to the Arbitrum merge contract and complete insolvency of the token merge mechanism.

### Remediation

Pull the `TGT` tokens from `from` using `tgt.safeTransferFrom` (or verify the exact balance delta received by `MergeTgt`) in `onTokenTransfer` before crediting `claimableTitnPerUser[from]` and `totalTitnClaimable`, and ensure `totalTitnClaimed + totalTitnClaimable` cannot exceed `TITN_ARB`.

```solidity
    function onTokenTransfer(address from, uint256 amount, bytes calldata extraData) external nonReentrant {
        if (msg.sender != address(tgt)) {
            revert InvalidTokenReceived();
        }
        if (lockedStatus == LockedStatus.Locked || launchTime == 0) {
            revert MergeLocked();
        }
        if (amount == 0) {
            revert ZeroAmount();
        }
        if (block.timestamp - launchTime > 360 days) {
            revert MergeEnded();
        }

        // tgt in, titn out
        tgt.safeTransferFrom(from, address(this), amount);

        uint256 titnOut = quoteTitn(amount);
        claimableTitnPerUser[from] += titnOut;
        totalTitnClaimable += titnOut;

        emit ClaimableTitnUpdated(from, titnOut);
    }
```

## Finding 3: Unbounded entitlement issuance in `MergeTgt.onTokenTransfer` causes contract insolvency and locks remaining TITN after 1 year

### Synopsis

`MergeTgt.onTokenTransfer` issues `TITN` entitlements without verifying that cumulative claimable and claimed `TITN` remain within the contract's `173,700,000 TITN` reserve (`TITN_ARB`, corresponding to `57.9%` of `MAX_TGT`). Depositing more than `579,000,000 TGT` creates unbacked liabilities, causing `claimTitn` to fail for late claimants and `withdrawRemainingTitn` to permanently revert via underflow.

### Technical details

In `MergeTgt.sol`, the protocol parameters define:
- `TGT_TO_EXCHANGE = 579_000_000 * 10 ** 18` (`57.9%` of the `1,000,000,000` `MAX_TGT` supply)
- `TITN_ARB = 173_700_000 * 10 ** 18` (`17.37%` of `MAX_TITN`)

The owner funds the contract via `deposit()`, which strictly requires `amount == TITN_ARB` (`173,700,000e18` `TITN`). During the first 90 days after `launchTime`, `quoteTitn(tgtAmount)` computes:

```solidity
titnAmount = (tgtAmount * TITN_ARB) / TGT_TO_EXCHANGE;
```

This conversion rate (`0.3 TITN` per `1 TGT`) exhausts the entire `TITN_ARB` reserve once `579,000,000 TGT` (`TGT_TO_EXCHANGE`) has been deposited. However, because `TGT_TO_EXCHANGE` represents only `57.9%` of the total `TGT` supply (`1,000,000,000 TGT`), up to `421,000,000` additional `TGT` tokens exist in circulation that can also be sent to `MergeTgt` via `transferAndCall`.

In `onTokenTransfer` (lines 81–85):

```solidity
uint256 titnOut = quoteTitn(amount);
claimableTitnPerUser[from] += titnOut;
totalTitnClaimable += titnOut;

emit ClaimableTitnUpdated(from, titnOut);
```

There is no check ensuring that the cumulative issued `TITN` entitlements (`totalTitnClaimed + totalTitnClaimable`) do not exceed `TITN_ARB` (or `titn.balanceOf(address(this))`), nor is there a cap restricting cumulative `TGT` deposits to `TGT_TO_EXCHANGE`.

When cumulative issued `TITN` exceeds `TITN_ARB` (`totalTitnClaimed + totalTitnClaimable > TITN_ARB`), two critical failures occur:

1. **Insolvency during the 360-day claim window (`claimTitn`):**
   As users call `claimTitn(amount)`, `titn.safeTransfer(msg.sender, amount)` (line 109) transfers `TITN` out of the contract, reducing `titn.balanceOf(address(this))` to `TITN_ARB - totalTitnClaimed`. Once `totalTitnClaimed` approaches or reaches `TITN_ARB`, remaining users with positive `claimableTitnPerUser` balances cannot claim their tokens because `titn.safeTransfer` reverts due to insufficient token balance.

2. **Permanent freeze of remaining funds after 360 days (`withdrawRemainingTitn`):**
   At any point where `totalTitnClaimed + totalTitnClaimable > TITN_ARB`, the contract's actual `TITN` balance (`currentRemainingTitn = TITN_ARB - totalTitnClaimed`) is strictly less than `totalTitnClaimable`. When the 360-day period ends and any user calls `withdrawRemainingTitn()`, the contract initializes:
   ```solidity
   remainingTitnAfter1Year = currentRemainingTitn;
   initialTotalClaimable = totalTitnClaimable;
   ```
   and immediately computes (line 135):
   ```solidity
   uint256 unclaimedTitn = remainingTitnAfter1Year - initialTotalClaimable;
   ```
   Because `remainingTitnAfter1Year < initialTotalClaimable`, this subtraction reverts with a Solidity 0.8 arithmetic underflow panic (`0x11`). Consequently, `withdrawRemainingTitn()` permanently reverts for all users, locking all remaining `TITN` in the contract forever while users' deposited `TGT` has already been transferred in.

### Assumptions and preconditions

1. `MergeTgt` is deployed, initialized (`launchTime` set, `lockedStatus` set to `OneWay` or `TwoWay`), and funded with `TITN_ARB` (`173,700,000e18` `TITN`).
2. Circulating `TGT` holders collectively deposit an amount of `TGT` such that the cumulative `titnOut` quoted across all deposits exceeds `TITN_ARB` (`173,700,000e18` `TITN`)—for example, depositing more than `579,000,000 TGT` within the first 90 days (or a correspondingly larger amount during the linear decay period between day 90 and day 360).

### Proof of concept

1. The admin deploys `MergeTgt`, sets `launchTime`, sets `lockedStatus = LockedStatus.OneWay`, and deposits `173,700,000e18` `TITN` (`TITN_ARB`).
2. Within the first 90 days, User A deposits `500,000,000e18` `TGT` via `tgt.transferAndCall(mergeTgt, 500_000_000e18, "0x")`.
   - `quoteTitn(500_000_000e18)` returns `150,000,000e18` `TITN`.
   - `claimableTitnPerUser[UserA] = 150,000,000e18`, `totalTitnClaimable = 150,000,000e18`.
3. User A immediately calls `claimTitn(150,000,000e18)`.
   - `totalTitnClaimed = 150,000,000e18`, `totalTitnClaimable = 0`.
   - `titn.balanceOf(mergeTgt)` is now `23,700,000e18` `TITN`.
4. Still within the first 90 days, User B deposits `100,000,000e18` `TGT` via `tgt.transferAndCall(mergeTgt, 100_000_000e18, "0x")`.
   - `quoteTitn(100_000_000e18)` returns `30,000,000e18` `TITN`.
   - `onTokenTransfer` succeeds without checking reserve bounds: `claimableTitnPerUser[UserB] = 30,000,000e18` and `totalTitnClaimable = 30,000,000e18`.
5. If User B calls `claimTitn(30,000,000e18)` before 360 days:
   - `titn.safeTransfer(UserB, 30_000,000e18)` on line 109 reverts because `mergeTgt` only holds `23,700,000e18` `TITN`.
6. If User B (or any other user who deposited and waited 12 months) waits until `360 days` have elapsed and calls `withdrawRemainingTitn()`:
   - `remainingTitnAfter1Year` is set to `currentRemainingTitn = 23,700,000e18`.
   - `initialTotalClaimable` is set to `totalTitnClaimable = 30,000,000e18`.
   - Line 135 executes `23,700,000e18 - 30,000,000e18`, reverting with an arithmetic underflow and permanently locking the remaining `23,700,000e18` `TITN` in the contract.

### Impact assessment

**Severity: High**

- **Direct Loss of Funds & Insolvency:** Users who deposit `TGT` once the `TITN_ARB` allocation is exceeded have their `TGT` tokens permanently taken by the contract while receiving unbacked `claimableTitnPerUser` entitlements that cannot be fully redeemed.
- **Permanent Freezing of Remaining Pool After 1 Year:** Any over-issuance of `totalTitnClaimable` relative to the contract's `TITN` balance causes `withdrawRemainingTitn()` to revert due to arithmetic underflow (`remainingTitnAfter1Year - initialTotalClaimable`), preventing all users who waited 12 months from withdrawing either their base claimable `TITN` or their proportional share of unclaimed `TITN`.

### Remediation

Enforce in `onTokenTransfer` that `titnOut > 0` and that cumulative `TITN` obligations (`totalTitnClaimable + titnOut`) never exceed the contract's current `TITN` balance (or equivalently that `totalTitnClaimed + totalTitnClaimable + titnOut <= TITN_ARB`).

```solidity
    function onTokenTransfer(address from, uint256 amount, bytes calldata extraData) external nonReentrant {
        if (msg.sender != address(tgt)) {
            revert InvalidTokenReceived();
        }
        if (lockedStatus == LockedStatus.Locked || launchTime == 0) {
            revert MergeLocked();
        }
        if (amount == 0) {
            revert ZeroAmount();
        }
        if (block.timestamp - launchTime > 360 days) {
            revert MergeEnded();
        }

        // tgt in, titn out

        uint256 titnOut = quoteTitn(amount);
        if (totalTitnClaimable + titnOut > titn.balanceOf(address(this))) {
            revert InvalidAmountReceived();
        }
        claimableTitnPerUser[from] += titnOut;
        totalTitnClaimable += titnOut;

        emit ClaimableTitnUpdated(from, titnOut);
    }
```

## Finding 4: Arithmetic Underflow in `MergeTgt.withdrawRemainingTitn` Locks Remaining Funds When Total Claimable Exceeds Remaining Balance

### Synopsis

In `MergeTgt.withdrawRemainingTitn`, the surplus calculation `unclaimedTitn = remainingTitnAfter1Year - initialTotalClaimable` assumes `remainingTitnAfter1Year >= initialTotalClaimable`. Because `TGT_TO_EXCHANGE` represents only 57.9% of `MAX_TGT` and deposits are uncapped, `initialTotalClaimable` can exceed `remainingTitnAfter1Year`, causing an arithmetic underflow that permanently blocks all post-1-year withdrawals.

### Technical details

The `MergeTgt` contract is funded with `TITN_ARB = 173_700_000 * 10 ** 18` TITN and calculates user exchange quotes in `quoteTitn()` using `TGT_TO_EXCHANGE = 579_000_000 * 10 ** 18`, which is only 57.9% of the 1,000,000,000 `MAX_TGT` total supply (`0.3 TITN` per `1 TGT` during the first 90 days).

However, `onTokenTransfer()` does not enforce an upper bound on the cumulative TGT deposited or the cumulative TITN allocated (`totalTitnClaimed + totalTitnClaimable <= TITN_ARB`). If more than `579_000_000 * 10 ** 18` TGT is deposited (or an equivalent amount across the 360-day window such that cumulative quoted TITN exceeds `TITN_ARB`), the total TITN obligations (`totalTitnClaimed + totalTitnClaimable`) exceed the `173_700_000 * 10 ** 18` TITN held by the contract.

When the 360-day merge period ends and users who left their TITN in the contract call `withdrawRemainingTitn()`:

```solidity
if (remainingTitnAfter1Year == 0) {
    remainingTitnAfter1Year = currentRemainingTitn;
    initialTotalClaimable = totalTitnClaimable;
}

uint256 claimableTitn = claimableTitnPerUser[msg.sender];
require(claimableTitn > 0, "No claimable TITN");

// Calculate proportional remaining TITN for the user
uint256 unclaimedTitn = remainingTitnAfter1Year - initialTotalClaimable;
uint256 userProportionalShare = (claimableTitn * unclaimedTitn) / initialTotalClaimable;

uint256 titnOut = claimableTitn + userProportionalShare;
```

Because `totalTitnClaimed + totalTitnClaimable > TITN_ARB`, the remaining contract balance `remainingTitnAfter1Year` (`TITN_ARB - totalTitnClaimed`) is strictly less than `initialTotalClaimable` (`totalTitnClaimable`). Consequently, line 135 (`remainingTitnAfter1Year - initialTotalClaimable`) reverts with a Solidity arithmetic underflow (`Panic(0x11)`). Every call to `withdrawRemainingTitn()` reverts, preventing all users who waited 12 months from withdrawing both their surplus and their base `claimableTitn`.

### Assumptions and preconditions

1. The `MergeTgt` contract is deployed, initialized with `launchTime`, unlocked, and funded with `TITN_ARB` (`173_700_000 * 10 ** 18` TITN).
2. Users deposit enough `TGT` during the 360-day merge window such that the total credited `TITN` (`totalTitnClaimed + totalTitnClaimable`) exceeds `TITN_ARB` (or `titn.balanceOf(address(this))` at 360 days is less than `totalTitnClaimable`).
3. At least 360 days have elapsed since `launchTime`, so `claimTitn` is disabled and users must call `withdrawRemainingTitn`.

### Proof of concept

1. During the first 90 days (`timeSinceLaunch < 90 days`), users deposit `600,000,000 * 10 ** 18` TGT (out of the `1,000,000,000 * 10 ** 18` TGT supply) via `tgt.transferAndCall(mergeTgt, amount, "0x")`.
2. `quoteTitn` credits a total of `(600,000,000e18 * 173,700,000e18) / 579,000,000e18 = 180,000,000 * 10 ** 18` TITN across depositors.
3. Suppose users claim `10,000,000 * 10 ** 18` TITN prior to 360 days via `claimTitn`. At 360 days:
   - `titn.balanceOf(address(mergeTgt))` = `163,700,000 * 10 ** 18`.
   - `totalTitnClaimable` = `170,000,000 * 10 ** 18`.
4. After 360 days, any remaining user calls `withdrawRemainingTitn()`:
   - `remainingTitnAfter1Year` is set to `163,700,000 * 10 ** 18`.
   - `initialTotalClaimable` is set to `170,000,000 * 10 ** 18`.
   - Line 135 evaluates `163,700,000e18 - 170,000,000e18`, reverting with arithmetic underflow (`Panic(0x11)`).
5. No user can withdraw any of the remaining `163,700,000 * 10 ** 18` TITN via `withdrawRemainingTitn()`.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Complete denial of service of `withdrawRemainingTitn()` due to arithmetic underflow on line 135 whenever `initialTotalClaimable > remainingTitnAfter1Year`.
- **Business Impact**: All users who waited 360 days to withdraw their TITN are unable to claim either their base entitlement or any remaining TITN, freezing up to `173,700,000 * 10 ** 18` TITN in the contract because `claimTitn()` also reverts once `block.timestamp - launchTime >= 360 days`.

### Remediation

Compute `titnOut` directly as the user's proportional share of `remainingTitnAfter1Year`:
$$\text{titnOut} = \frac{\text{claimableTitn} \times \text{remainingTitnAfter1Year}}{\text{initialTotalClaimable}}$$
Algebraically, when `remainingTitnAfter1Year >= initialTotalClaimable`, this is equivalent to `claimableTitn + (claimableTitn * (remainingTitnAfter1Year - initialTotalClaimable)) / initialTotalClaimable`, while also gracefully scaling payouts without underflowing or exceeding the contract's available balance if `remainingTitnAfter1Year < initialTotalClaimable`.

```solidity
        uint256 currentRemainingTitn = titn.balanceOf(address(this));

        if (remainingTitnAfter1Year == 0) {
            // Initialize remainingTitnAfter1Year to the current balance of TITN
            remainingTitnAfter1Year = currentRemainingTitn;

            // Capture the total claimable TITN at the time of the first claim
            initialTotalClaimable = totalTitnClaimable;
        }

        uint256 claimableTitn = claimableTitnPerUser[msg.sender];
        require(claimableTitn > 0, "No claimable TITN");

        // Calculate proportional remaining TITN for the user without underflowing when over-subscribed
        uint256 titnOut = (claimableTitn * remainingTitnAfter1Year) / initialTotalClaimable;

        // Update state variables
```

## Finding 5: Uncapped TITN Quote Calculation in `MergeTgt.quoteTitn` Allows Total Claimable Liabilities to Exceed `TITN_ARB` Reserves, Causing Contract Insolvency and Locking Remaining Withdrawals

### Synopsis

`MergeTgt.quoteTitn` calculates TITN payouts using `TGT_TO_EXCHANGE` (57.9% of `MAX_TGT`) without capping cumulative allocated TITN against the fixed `TITN_ARB` reserve. If more than 579M TGT is merged, `totalTitnClaimable` exceeds the contract's TITN balance, causing insolvency and permanent DoS in `withdrawRemainingTitn` and `claimTitn`.

### Technical details

In `MergeTgt.sol`, the contract is funded via `deposit()` with a strictly enforced, fixed reserve of `TITN_ARB = 173_700_000 * 10 ** 18` TITN (line 50). The exchange rate in `quoteTitn` (lines 153–165) is parameterized against `TGT_TO_EXCHANGE = 579_000_000 * 10 ** 18`, which represents only 57.9% of the 1,000,000,000 `MAX_TGT` supply (giving an initial rate of `0.3` TITN per TGT during the first 90 days):

```solidity
if (timeSinceLaunch < 90 days) {
    titnAmount = (tgtAmount * TITN_ARB) / TGT_TO_EXCHANGE;
} else if (timeSinceLaunch < 360 days) {
    uint256 remainingtime = 360 days - timeSinceLaunch;
    titnAmount = (tgtAmount * TITN_ARB * remainingtime) / (TGT_TO_EXCHANGE * 270 days); //270 days = 9 months
}
```

When users deposit TGT via `onTokenTransfer`, `quoteTitn(amount)` is called to determine `titnOut`, which is directly added to `claimableTitnPerUser[from]` and `totalTitnClaimable` (lines 81–83). However, `quoteTitn` neither checks nor caps `titnAmount` against the remaining unallocated TITN reserve (`TITN_ARB - (totalTitnClaimed + totalTitnClaimable)`).

Because the total supply of TGT is `1_000_000_000 * 10 ** 18` (421,000,000 TGT more than `TGT_TO_EXCHANGE`), users can deposit more than `579_000_000 * 10 ** 18` TGT into `MergeTgt`. When this happens, cumulative TITN liabilities (`totalTitnClaimed + totalTitnClaimable`) exceed the `173_700_000 * 10 ** 18` `TITN_ARB` reserve held by the contract (up to `300_000_000 * 10 ** 18` TITN if 1B TGT is deposited in the first 90 days).

This reserve-to-liability inconsistency has two critical consequences for settlement and solvency:
1. **Insolvency during `withdrawRemainingTitn` (lines 114–151):** Users who deposit TGT and leave their claimable TITN in the contract for 12 months call `withdrawRemainingTitn()`. If `totalTitnClaimed + totalTitnClaimable > TITN_ARB`, then after 360 days `remainingTitnAfter1Year` (`titn.balanceOf(address(this)) = TITN_ARB - totalTitnClaimed`) is strictly less than `initialTotalClaimable` (`totalTitnClaimable`). Consequently, `uint256 unclaimedTitn = remainingTitnAfter1Year - initialTotalClaimable` (line 135) reverts with an arithmetic underflow, and `titn.safeTransfer(msg.sender, titnOut)` (line 148) cannot be reached or fulfilled, permanently locking all remaining TITN in the contract.
2. **Insolvency during `claimTitn`:** Once `totalTitnClaimed` reaches `TITN_ARB`, remaining users with non-zero `claimableTitnPerUser` cannot claim their owed TITN because the contract's TITN balance is exhausted.

### Assumptions and preconditions

1. `MergeTgt` is deployed, initialized with `TITN_ARB` (`173_700_000 * 10 ** 18` TITN), and unlocked (`lockedStatus != LockedStatus.Locked`, `launchTime > 0`).
2. The circulating supply of TGT available to be merged exceeds `579_000_000 * 10 ** 18` (out of the `1_000_000_000 * 10 ** 18` total TGT supply), or enough TGT is merged across the 360-day window such that cumulative quoted TITN exceeds `173_700_000 * 10 ** 18`.

### Proof of concept

1. The owner deploys `MergeTgt`, sets `launchTime`, unlocks the contract, and deposits `TITN_ARB = 173,700,000e18` TITN.
2. During the first 90 days (`timeSinceLaunch < 90 days`), multiple TGT holders merge a combined total of `600,000,000e18` TGT via `tgt.transferAndCall(mergeTgt, amount, "0x")`.
3. `quoteTitn` (line 158) calculates total `titnAmount`:
   $$\frac{600,000,000 \times 10^{18} \times 173,700,000 \times 10^{18}}{579,000,000 \times 10^{18}} = 180,000,000 \times 10^{18}\text{ TITN}$$
4. `totalTitnClaimable` becomes `180,000,000e18`, while `titn.balanceOf(address(mergeTgt))` is only `173,700,000e18`.
5. Suppose users holding `100,000,000e18` of `claimableTitnPerUser` call `claimTitn()` before 360 days:
   - `totalTitnClaimed = 100,000,000e18`
   - `totalTitnClaimable = 80,000,000e18`
   - `titn.balanceOf(address(mergeTgt)) = 73,700,000e18`
6. The remaining users (who hold `80,000,000e18` of `claimableTitnPerUser`) wait until 360 days have elapsed and call `withdrawRemainingTitn()`:
   - `remainingTitnAfter1Year` is set to `73,700,000e18`.
   - `initialTotalClaimable` is set to `80,000,000e18`.
   - Line 135 computes `remainingTitnAfter1Year - initialTotalClaimable` (`73,700,000e18 - 80,000,000e18`), which reverts with panic `0x11` (arithmetic underflow).
   - None of the remaining users can execute lines 141–151 to withdraw their TITN, permanently freezing all `73,700,000e18` TITN inside `MergeTgt`.

### Impact assessment

**Severity: High**

- **Direct Technical Impact:** Total credited `claimableTitnPerUser` (`totalTitnClaimable + totalTitnClaimed`) can exceed the fixed `TITN_ARB` reserve by up to `126,300,000e18` TITN. Once total liabilities exceed `TITN_ARB`, `withdrawRemainingTitn()` permanently reverts due to underflow when computing `unclaimedTitn`, and `claimTitn()` reverts for late claimers once the `TITN_ARB` balance is depleted.
- **Business Impact:** Users who deposit their TGT (which is permanently locked in the merge contract) and wait 12 months as incentivized by the protocol are unable to withdraw any of their TITN, and all remaining TITN in the contract becomes frozen unless rescued by the owner.

### Remediation

Cap the quoted `titnAmount` in `quoteTitn` so that the cumulative allocated TITN (`totalTitnClaimed + totalTitnClaimable + titnAmount`) cannot exceed `TITN_ARB` (or revert if the cap is exceeded). Alternatively, if all `1_000_000_000 * 10 ** 18` `MAX_TGT` is intended to be eligible for merging without a first-come-first-served cap, adjust the denominator or reserve accordingly.

To enforce the `TITN_ARB` reserve cap in `quoteTitn`:

```solidity
    function quoteTitn(uint256 tgtAmount) public view returns (uint256 titnAmount) {
        require(launchTime > 0, "Launch time not set");

        uint256 timeSinceLaunch = (block.timestamp - launchTime);
        if (timeSinceLaunch < 90 days) {
            titnAmount = (tgtAmount * TITN_ARB) / TGT_TO_EXCHANGE;
        } else if (timeSinceLaunch < 360 days) {
            uint256 remainingtime = 360 days - timeSinceLaunch;
            titnAmount = (tgtAmount * TITN_ARB * remainingtime) / (TGT_TO_EXCHANGE * 270 days); //270 days = 9 months
        } else {
            titnAmount = 0;
        }

        uint256 totalAllocated = totalTitnClaimed + totalTitnClaimable;
        uint256 availableTitn = TITN_ARB > totalAllocated ? TITN_ARB - totalAllocated : 0;
        if (titnAmount > availableTitn) {
            titnAmount = availableTitn;
        }
    }
```

## Finding 6: Unprotected Time-Dependent Quote Decay and Inconsistent Phase Boundaries in `MergeTgt.quoteTitn` Lead to Zero or Reduced TITN Allocation

### Synopsis

In `MergeTgt.quoteTitn`, token conversion pricing after 90 days decays every second based on `block.timestamp` and drops to zero at `360 days`, while `onTokenTransfer` allows deposits up to and including `360 days` without minimum-output or non-zero validation, causing users to receive fewer or zero TITN tokens for permanently locked TGT.

### Technical details

The `MergeTgt` contract relies on `quoteTitn(uint256 tgtAmount)` (lines 153–165) to determine how much `TITN` a user receives when transferring `TGT` via `onTokenTransfer` (lines 65–86):

```solidity
uint256 timeSinceLaunch = (block.timestamp - launchTime);
if (timeSinceLaunch < 90 days) {
    titnAmount = (tgtAmount * TITN_ARB) / TGT_TO_EXCHANGE;
} else if (timeSinceLaunch < 360 days) {
    uint256 remainingtime = 360 days - timeSinceLaunch;
    titnAmount = (tgtAmount * TITN_ARB * remainingtime) / (TGT_TO_EXCHANGE * 270 days); //270 days = 9 months
} else {
    titnAmount = 0;
}
```

This temporal pricing and phase-transition logic introduces inconsistent conversion and settlement boundaries and quote-to-execution rate drift:

1. **Inconsistent Conversion, Pricing, and Settlement Phase Boundaries (`360 days`):**
   - `onTokenTransfer` (line 75) checks `if (block.timestamp - launchTime > 360 days) revert MergeEnded();`, permitting `TGT` deposits when `block.timestamp - launchTime == 360 days`.
   - `quoteTitn` (line 159) uses a strict inequality `else if (timeSinceLaunch < 360 days)`. At `timeSinceLaunch == 360 days` (or when `remainingtime` in line 160–161 is small enough that integer division truncates `(tgtAmount * TITN_ARB * remainingtime) / (TGT_TO_EXCHANGE * 270 days)` to `0`), `quoteTitn` returns `titnAmount = 0`.
   - Because `onTokenTransfer` does not require `titnOut > 0`, the user's `TGT` is transferred into `MergeTgt` and permanently locked while `claimableTitnPerUser[from]` increases by `0`. If the user had no prior claimable balance, they cannot participate in `withdrawRemainingTitn()` either (which requires `claimableTitn > 0` at line 132).
   - Meanwhile, `claimTitn` (line 99) reverts when `block.timestamp - launchTime >= 360 days`, so any user whose `onTokenTransfer` executes right before `360 days` cannot complete the standard `claimTitn` step once the timestamp reaches `360 days`.

2. **Continuous Per-Second Quote-to-Execution Rate Drift Without Slippage Protection:**
   - During the decay phase (`90 days <= timeSinceLaunch < 360 days`, lines 159–161), `remainingtime = 360 days - timeSinceLaunch` decreases every second as `block.timestamp` advances.
   - A user following the documented flow calls `quoteTitn(amount)` off-chain and then submits `tgt.transferAndCall(mergeTgt, amount, "0x")`. By the time the transaction is mined in a later block, `timeSinceLaunch` is strictly larger, so `titnOut` is always lower than quoted—or `0` if executed at the `360 days` boundary—with no `minTitnOut` validation or revert on zero output.

### Assumptions and preconditions

1. `MergeTgt` has been initialized by the owner with `launchTime > 0` and `lockedStatus != LockedStatus.Locked`.
2. `block.timestamp - launchTime` is in the decay window (`>= 90 days`) or at the `360 days` phase boundary.
3. A user initiates a `TGT` merge via `tgt.transferAndCall( address(mergeTgt), amount, extraData )`.

### Proof of concept

1. Assume `launchTime = 1_000_000`.
2. At `block.timestamp = launchTime + 360 days - 12` (12 seconds before the 360-day mark), User A calls `quoteTitn(1000e18)` and observes a non-zero quote:
   $$\text{titnAmount} = \frac{1000 \times 10^{18} \times 173{,}700{,}000 \times 10^{18} \times 12}{579{,}000{,}000 \times 10^{18} \times 23{,}328{,}000} \approx 1.5435 \times 10^{14} \text{ wei}$$
3. User A submits `tgt.transferAndCall(address(mergeTgt), 1000e18, "")`, which is included in a block at `block.timestamp = launchTime + 360 days`.
4. `MergeTgt.onTokenTransfer` executes:
   - `block.timestamp - launchTime > 360 days` evaluates to `360 days > 360 days` (`false`), so it does not revert.
   - `quoteTitn(1000e18)` evaluates `timeSinceLaunch < 360 days` (`false`) and returns `titnAmount = 0`.
   - `claimableTitnPerUser[userA]` remains `0`, while `1000e18` `TGT` is transferred to `MergeTgt`.
5. User A cannot call `claimTitn` (reverts with `TooLateToClaimRemainingTitn`) and cannot call `withdrawRemainingTitn` (reverts with `"No claimable TITN"` because `claimableTitnPerUser[userA] == 0`), resulting in complete loss of the deposited `TGT`.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** Users converting `TGT` during the linear decay phase receive fewer `TITN` tokens than quoted due to block timestamp progression, and users whose transactions land at the `360 days` boundary (or whose decayed quote truncates to `0`) permanently lose their `TGT` tokens while receiving `0` claimable `TITN`.
- **Business Implications:** Permanent loss of user funds (`TGT`) without any `TITN` allocation and inconsistent phase transition behavior at the end of the merge window.

### Remediation

Align the phase boundary across `onTokenTransfer`, `claimTitn`, `withdrawRemainingTitn`, and `quoteTitn` (reversing in `quoteTitn` or `onTokenTransfer` when `timeSinceLaunch >= 360 days` or when `titnAmount == 0`), and decode an optional `minTitnOut` slippage parameter from `extraData` in `onTokenTransfer`.

In `quoteTitn`:

```solidity
    function quoteTitn(uint256 tgtAmount) public view returns (uint256 titnAmount) {
        require(launchTime > 0, "Launch time not set");

        uint256 timeSinceLaunch = (block.timestamp - launchTime);
        if (timeSinceLaunch < 90 days) {
            titnAmount = (tgtAmount * TITN_ARB) / TGT_TO_EXCHANGE;
        } else if (timeSinceLaunch < 360 days) {
            uint256 remainingtime = 360 days - timeSinceLaunch;
            titnAmount = (tgtAmount * TITN_ARB * remainingtime) / (TGT_TO_EXCHANGE * 270 days); //270 days = 9 months
        } else {
            revert MergeEnded();
        }

        if (titnAmount == 0) {
            revert ZeroAmount();
        }
    }
```
