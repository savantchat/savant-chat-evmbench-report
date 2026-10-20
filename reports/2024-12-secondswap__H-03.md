# Audit report — 2024-12-secondswap/H-03

## Finding 1: Stale Authority Bindings for `manager` and `tokenIssuer` in `SecondSwap_StepVesting` Break Vesting Management After Role Updates

### Synopsis

`SecondSwap_StepVesting` permanently binds the `manager` and `tokenIssuer` addresses at contract construction without providing any mechanism to update them or query `SecondSwap_VestingDeployer`. When `SecondSwap_VestingDeployer.setManager` or `setTokenOwner` updates these roles, existing `SecondSwap_StepVesting` instances retain stale authority bindings, breaking marketplace operations and leaving revoked addresses authorized.

### Technical details

In `SecondSwap_StepVesting.sol`, the authority roles `manager` (line 91) and `tokenIssuer` (line 96) are stored as state variables and initialized once in the `constructor` (lines 127–128) when `SecondSwap_VestingDeployer.deployVesting` deploys a new vesting contract (lines 119–129).

However, `SecondSwap_StepVesting` does not expose any setter functions to update `manager` or `tokenIssuer`, nor does it query `vestingDeployer` for the current `manager` or token owner:

1. **Stale `manager` binding:** `SecondSwap_VestingDeployer` implements `setManager(address _manager)` (lines 152–155) to allow `s2Admin` to rotate or upgrade the `SecondSwap_VestingManager` contract. When `manager` is updated in `SecondSwap_VestingDeployer`, all previously deployed `SecondSwap_StepVesting` contracts still hold the old `manager` address in `SecondSwap_StepVesting.manager`. Consequently:
   - Any call from the new `SecondSwap_VestingManager` to `SecondSwap_StepVesting.transferVesting` during `listVesting` (line 139), `unlistVesting` (line 151), or `completePurchase` (line 163) reverts with `"SS_StepVesting: unauthorized"` at line 218 of `SecondSwap_StepVesting.sol`.
   - The old, replaced `manager` address retains full authorization to call `transferVesting`, `createVesting`, and `createVestings` on all existing `SecondSwap_StepVesting` instances.

2. **Stale `tokenIssuer` binding:** `SecondSwap_StepVesting` permanently caches `tokenIssuer = msg.sender` at deployment. If a token's owner is rotated or reassigned via `SecondSwap_VestingDeployer.setTokenOwner` (lines 141–144):
   - The old `tokenIssuer` still passes the direct authorization checks in `SecondSwap_StepVesting.transferVesting`, `createVesting`, `createVestings`, and `SecondSwap_VestingManager.setMaxSellPercent` (line 195).
   - If the new token owner calls `SecondSwap_VestingDeployer.createVesting` (lines 176–183), execution passes the check in `SecondSwap_VestingDeployer` (`_tokenOwner[msg.sender] == address(SecondSwap_StepVesting(_stepVesting).token())`) and delegates to `SecondSwap_StepVesting._createVesting`, which then executes `token.safeTransferFrom(tokenIssuer, address(this), _totalAmount)` (line 307) against the stale `tokenIssuer` address rather than the current token owner.

### Assumptions and preconditions

1. At least one `SecondSwap_StepVesting` contract has been deployed via `SecondSwap_VestingDeployer.deployVesting`.
2. The protocol admin (`s2Admin`) updates the manager contract via `SecondSwap_VestingDeployer.setManager`, or updates the token owner via `SecondSwap_VestingDeployer.setTokenOwner`.

### Proof of concept

1. `tokenOwner` deploys a vesting contract `stepVesting` via `SecondSwap_VestingDeployer.deployVesting(...)`. At this point, `stepVesting.manager()` is set to `managerV1` and `stepVesting.tokenIssuer()` is set to `tokenOwner`.
2. A beneficiary lists part of their vesting allocation on the marketplace; `managerV1.listVesting` transfers the allocation to `address(managerV1)` inside `stepVesting`.
3. `s2Admin` deploys a new vesting manager `managerV2` and calls `SecondSwap_VestingDeployer.setManager(address(managerV2))` and `MarketplaceSetting.setManager(address(managerV2))`.
4. A buyer attempts to purchase the listed vesting or the seller attempts to unlist it through the marketplace, which calls `managerV2.completePurchase` or `managerV2.unlistVesting`.
5. `managerV2` calls `stepVesting.transferVesting(...)`.
6. Inside `stepVesting.transferVesting`, `msg.sender` is `managerV2`, which does not match `stepVesting.manager()` (`managerV1`), `stepVesting.tokenIssuer()`, or `stepVesting.vestingDeployer()`. The transaction reverts with `"SS_StepVesting: unauthorized"`, permanently freezing the listed vesting allocations and rendering `stepVesting` unusable with `managerV2`.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effects:** Updating the `manager` in `SecondSwap_VestingDeployer` causes a denial of service for all marketplace listing, unlisting, and purchasing operations on existing `SecondSwap_StepVesting` contracts, while leaving the deprecated `manager` and original `tokenIssuer` addresses permanently privileged on those contracts.
- **Business Implications:** The protocol cannot rotate or upgrade `SecondSwap_VestingManager` or change a `tokenIssuer` without breaking existing vesting plans or leaving stale privileges active.

### Remediation

Either query the current authority addresses dynamically from `vestingDeployer`, or add access-controlled setter functions on `SecondSwap_StepVesting` callable by `vestingDeployer` so role updates are synchronized.

Below is the preferred approach reading `manager` from `vestingDeployer` and allowing `vestingDeployer` to update `tokenIssuer` (or reading it from `vestingDeployer`):

```solidity
    /**
     * @notice Address of the manager contract
     */
    address public manager;

    /**
     * @notice Address of the token issuer
     */
    address public tokenIssuer;

    /**
     * @notice Updates the manager address from the vesting deployer
     * @param _manager New manager address
     */
    function setManager(address _manager) external {
        require(msg.sender == vestingDeployer, "SS_StepVesting: unauthorized");
        require(_manager != address(0), "SS_StepVesting: manager is zero");
        manager = _manager;
    }

    /**
     * @notice Updates the token issuer address from the vesting deployer
     * @param _tokenIssuer New token issuer address
     */
    function setTokenIssuer(address _tokenIssuer) external {
        require(msg.sender == vestingDeployer, "SS_StepVesting: unauthorized");
        require(_tokenIssuer != address(0), "SS_StepVesting: tokenIssuer is zero");
        tokenIssuer = _tokenIssuer;
    }
```

## Finding 2: Uncapped `stepsClaimed` update in `SecondSwap_StepVesting.claim` allows `stepsClaimed` to exceed `numOfSteps`, causing underflow DoS in subsequent vesting operations

### Synopsis

In `SecondSwap_StepVesting.claim()`, `vesting.stepsClaimed` is incremented by the uncapped `claimableSteps` returned by `claimable()`. Because `stepDuration` is truncated via integer division, `currentStep` can exceed `numOfSteps`, pushing `vesting.stepsClaimed > numOfSteps` and causing arithmetic underflow reverts in `_createVesting()` when unlisting, purchasing, or reallocating vestings.

### Technical details

In the constructor of `SecondSwap_StepVesting` (line 133), the duration of each step is computed using integer division:

```solidity
stepDuration = (_endTime - _startTime) / _numOfSteps;
```

Whenever `(_endTime - _startTime)` is not an exact multiple of `_numOfSteps`, `stepDuration` is rounded down, leaving a remainder `r = (_endTime - _startTime) % _numOfSteps`. As `block.timestamp` approaches or reaches `endTime`, `elapsedTime = currentTime - startTime` reaches `numOfSteps * stepDuration + r`. Whenever `r >= stepDuration`, the calculated step in `claimable()` (line 173):

```solidity
uint256 currentStep = elapsedTime / stepDuration;
uint256 claimableSteps = currentStep - vesting.stepsClaimed;
```

yields `currentStep > numOfSteps`, so `vesting.stepsClaimed + claimableSteps > numOfSteps`.

At the maturity boundary in `claimable()` (lines 178–182):

```solidity
if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
    claimableAmount = vesting.totalAmount - vesting.amountClaimed;
    return (claimableAmount, claimableSteps);
}
```

`claimableAmount` is capped at the remaining unclaimed allocation `vesting.totalAmount - vesting.amountClaimed`, but `claimableSteps` is returned uncapped.

When the beneficiary calls `claim()` (lines 193–203):

```solidity
(uint256 claimableAmount, uint256 claimableSteps) = claimable(msg.sender);
require(claimableAmount > 0, "SS_StepVesting: nothing to claim");

Vesting storage vesting = _vestings[msg.sender];
vesting.stepsClaimed += claimableSteps;
vesting.amountClaimed += claimableAmount;
```

line 198 updates `vesting.stepsClaimed` to `currentStep > numOfSteps`, violating the arithmetic domain invariant `vesting.stepsClaimed <= numOfSteps`.

All downstream vesting allocation and transfer logic in `_createVesting()` (lines 292, 297, and 300) assumes `numOfSteps >= stepsClaimed` and computes `numOfSteps - _vestings[_beneficiary].stepsClaimed`. Once `claim()` sets `vesting.stepsClaimed > numOfSteps`, any subsequent call for that beneficiary to:
- `VestingManager.unlistVesting()` (which calls `transferVesting(address(this), seller, amount)`),
- `VestingManager.completePurchase()` (which calls `transferVesting(address(this), buyer, amount)`),
- `SecondSwap_StepVesting.transferVesting()`, or
- `SecondSwap_StepVesting.createVesting()` / `createVestings()`

reverts with a Solidity `Panic(0x11)` arithmetic underflow when evaluating `numOfSteps - _vestings[_beneficiary].stepsClaimed`. Additionally, the truncation of `stepDuration` causes the vesting schedule to drift and reach step `numOfSteps` at `startTime + numOfSteps * stepDuration < endTime`, allowing beneficiaries to claim 100% of their vested allocation before `endTime`.

### Assumptions and preconditions

1. A `SecondSwap_StepVesting` contract is deployed with `(_endTime - _startTime) % _numOfSteps >= stepDuration` (i.e., `(_endTime - _startTime) % _numOfSteps >= (_endTime - _startTime) / _numOfSteps`).
2. A beneficiary has an active vesting allocation with unclaimed tokens (`totalAmount > amountClaimed`).
3. `block.timestamp` reaches at least `startTime + (numOfSteps + 1) * stepDuration` before the beneficiary claims their final tokens.

### Proof of concept

1. A vesting contract is deployed with `startTime = 1000`, `endTime = 2000` (duration `1000`), and `numOfSteps = 600`.
2. The constructor calculates `stepDuration = (2000 - 1000) / 600 = 1`.
3. Alice is granted `totalAmount = 600e18` tokens (`releaseRate = 1e18`).
4. At `t = 1100`, Alice lists `100e18` tokens on the marketplace via `VestingManager.listVesting()`, transferring `100e18` of her vesting to `VestingManager` and leaving `totalAmount = 500e18` in `_vestings[alice]`.
5. At `t = 1601` (or at `t = 2000`), Alice calls `claim()` to claim her remaining `500e18` vested tokens:
   - `claimable(alice)` computes `elapsedTime = 601`, `currentStep = 601 / 1 = 601`, and `claimableSteps = 601`.
   - Since `0 + 601 >= 600`, `claimableAmount = 500e18 - 0 = 500e18`, returning `(500e18, 601)`.
   - `claim()` sets `_vestings[alice].stepsClaimed = 601` and `_vestings[alice].amountClaimed = 500e18`, transferring `500e18` tokens to Alice.
6. Alice now attempts to unlist her `100e18` tokens from the marketplace (`unlistVesting`), or the token issuer attempts to allocate/reallocate tokens to Alice (`createVesting` / `transferVesting`).
7. `_createVesting(alice, ...)` executes the `else` branch because `_vestings[alice].totalAmount == 500e18 != 0`, and evaluates `numOfSteps - _vestings[alice].stepsClaimed` (`600 - 601`), which reverts due to arithmetic underflow. Alice's `100e18` listed tokens cannot be unlisted by her.

### Impact assessment

**Severity: Medium / High**

- **Permanent Locking of Listed Vesting / DoS on Reallocation:** If a user lists part of their vesting on the marketplace and claims their remaining vested tokens at a timestamp where `currentStep > numOfSteps`, their `stepsClaimed` exceeds `numOfSteps`. They can never unlist their marketplace listing (`unlistVesting` always reverts), can never purchase additional vestings from that plan (`completePurchase` always reverts), and the token issuer cannot reallocate or add vestings to or from that address (violating the core protocol invariant: *"The token issuers must always be able to reallocate vestings"*).
- **Schedule Drift / Premature Full Vesting:** Because `stepDuration` rounds down, all `numOfSteps` complete at `startTime + numOfSteps * stepDuration < endTime`, allowing 100% of tokens to be claimed via `claim()` before `endTime`.

### Remediation

Cap `vesting.stepsClaimed` at `numOfSteps` in `claim()` (and cap `currentStep` / `claimableSteps` in `claimable()` so `stepsClaimed + claimableSteps` never exceeds `numOfSteps`):

```solidity
    function claim() external {
        (uint256 claimableAmount, uint256 claimableSteps) = claimable(msg.sender);
        require(claimableAmount > 0, "SS_StepVesting: nothing to claim");

        Vesting storage vesting = _vestings[msg.sender];
        vesting.stepsClaimed = Math.min(numOfSteps, vesting.stepsClaimed + claimableSteps);
        vesting.amountClaimed += claimableAmount;

        token.safeTransfer(msg.sender, claimableAmount);
        emit Claimed(msg.sender, claimableAmount);
    }
```

## Finding 3: Incorrect `releaseRate` recalculation in `SecondSwap_StepVesting.transferVesting` allows grantors to overclaim tokens via `claim()`

### Synopsis

In `SecondSwap_StepVesting.transferVesting()`, the grantor's `releaseRate` is recalculated as `grantorVesting.totalAmount / numOfSteps`, ignoring `amountClaimed` and `stepsClaimed`. Consequently, a grantor who has already claimed steps and then transfers or lists their remaining vesting can call `claim()` to claim more tokens than their remaining `totalAmount`, draining tokens from the contract.

### Technical details

In `SecondSwap_StepVesting`, each beneficiary's vesting state is tracked by a `Vesting` struct containing `stepsClaimed`, `amountClaimed`, `releaseRate`, and `totalAmount`.

When a beneficiary claims tokens in `claim()` (lines 193–203), `claimable(msg.sender)` (lines 161–186) calculates the claimable amount for any intermediate step (`vesting.stepsClaimed + claimableSteps < numOfSteps`) as:

```solidity
claimableAmount = vesting.releaseRate * claimableSteps;
```

In `claim()`, the contract then updates the beneficiary's state and transfers the tokens without verifying whether `vesting.amountClaimed + claimableAmount` exceeds `vesting.totalAmount`:

```solidity
Vesting storage vesting = _vestings[msg.sender];
vesting.stepsClaimed += claimableSteps;
vesting.amountClaimed += claimableAmount;

token.safeTransfer(msg.sender, claimableAmount);
```

When a grantor's vesting allocation is reduced via `transferVesting()` (lines 216–235)—which is invoked either by `SecondSwap_VestingDeployer.transferVesting()` or when a user lists vested tokens for sale on the marketplace via `SecondSwap_VestingManager.listVesting()`—the grantor's `totalAmount` and `releaseRate` are updated on lines 229–230:

```solidity
grantorVesting.totalAmount -= _amount;
grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;
```

This `releaseRate` calculation is flawed whenever `grantorVesting.stepsClaimed > 0`:
1. `grantorVesting.totalAmount` after subtracting `_amount` still includes `grantorVesting.amountClaimed` (the tokens already claimed by the grantor during the first `stepsClaimed` steps).
2. The denominator uses `numOfSteps` instead of the remaining steps `numOfSteps - grantorVesting.stepsClaimed` (unlike `_createVesting()` on lines 299–300, which computes `(totalAmount - amountClaimed) / (numOfSteps - stepsClaimed)`).

Mathematically, if a grantor initially had allocation $T_0$ over $N$ steps ($R_0 = T_0 / N$), claimed at step $s > 0$ ($A = s \cdot R_0$), and then transferred $\Delta > 0$ tokens so that $T_1 = T_0 - \Delta$, the true remaining release rate over the remaining $N - s$ steps is:

$$R_{\text{correct}} = \frac{T_1 - A}{N - s} = \frac{T_0}{N} - \frac{\Delta}{N - s}$$

However, line 230 sets:

$$R_{\text{buggy}} = \frac{T_1}{N} = \frac{T_0}{N} - \frac{\Delta}{N}$$

Because $N - s < N$ for all $0 < s < N$, $R_{\text{buggy}} > R_{\text{correct}}$ holds whenever any steps have been claimed prior to transferring vesting.

In the extreme case where the grantor transfers all of their remaining unclaimed allocation ($\Delta = T_0 - A$), `grantorVesting.totalAmount` becomes equal to `grantorVesting.amountClaimed` ($A > 0$), leaving the grantor with `0` unclaimed tokens (`available(_grantor) == 0`). Yet `grantorVesting.releaseRate` is set to $A / N > 0$. On subsequent steps before `numOfSteps`, calling `claim()` (lines 193–203) allows the grantor to continue claiming $A / N$ tokens per step up to step $N - 1$, pushing `vesting.amountClaimed` above `vesting.totalAmount`, stealing tokens held by `SecondSwap_StepVesting` for other beneficiaries, and causing `available()` (line 142) and final-step `claimable()` (line 180) to revert due to arithmetic underflow.

### Assumptions and preconditions

1. A `SecondSwap_StepVesting` contract is deployed with `numOfSteps > 2`.
2. A beneficiary (`_grantor`) has a non-zero vesting allocation and has claimed tokens via `claim()` at least once after `startTime` (`0 < stepsClaimed < numOfSteps - 1` and `amountClaimed > 0`).
3. A portion or all of the grantor's remaining unclaimed allocation is transferred via `transferVesting()` (either through marketplace listing in `SecondSwap_VestingManager.listVesting()` or reallocation via `SecondSwap_VestingDeployer.transferVesting()`).

### Proof of concept

Consider a `SecondSwap_StepVesting` schedule with `numOfSteps = 10` and `stepDuration = 1 day`:

1. **Initial Allocation**: Alice is allocated `totalAmount = 1,000e18` tokens. Her initial state is:
   - `stepsClaimed = 0`
   - `amountClaimed = 0`
   - `releaseRate = 1,000e18 / 10 = 100e18`
   - `totalAmount = 1,000e18`

2. **Claim at Step 5**: After 5 days (`currentStep = 5`), Alice calls `claim()` (line 193):
   - `claimable(Alice)` returns `(500e18, 5)`.
   - Alice receives `500e18` tokens.
   - Alice's state becomes `stepsClaimed = 5`, `amountClaimed = 500e18`, `totalAmount = 1,000e18`.

3. **Transfer / List Remaining Allocation**: At step 5, Alice transfers (or lists on the marketplace via `listVesting`) her entire remaining `500e18` unclaimed allocation (`_amount = 500e18`):
   - In `transferVesting()` (line 225), `1,000e18 - 500e18 >= 500e18` passes.
   - Line 229 updates `Alice.totalAmount = 1,000e18 - 500e18 = 500e18`.
   - Line 230 updates `Alice.releaseRate = 500e18 / 10 = 50e18` (instead of `(500e18 - 500e18) / (10 - 5) = 0`).
   - Meanwhile, the recipient receives a vesting allocation of `500e18` with `stepsClaimed = 5` and `releaseRate = 500e18 / (10 - 5) = 100e18`.

4. **Overclaim at Step 9**: At day 9 (`currentStep = 9`), Alice calls `claim()` (line 193) again:
   - In `claimable(Alice)`: `claimableSteps = 9 - 5 = 4`.
   - Since `vesting.stepsClaimed + claimableSteps = 5 + 4 = 9 < 10`, line 184 computes `claimableAmount = 50e18 * 4 = 200e18`.
   - In `claim()`:
     - `Alice.stepsClaimed` becomes `9`.
     - `Alice.amountClaimed` becomes `500e18 + 200e18 = 700e18` (exceeding `Alice.totalAmount = 500e18`).
     - Line 201 transfers `200e18` tokens to Alice.

5. **Result**: Alice has claimed `700e18` tokens while the recipient of her transfer can claim `500e18` tokens, totaling `1,200e18` tokens from a `1,000e18` allocation. The extra `200e18` tokens are drained directly from other beneficiaries' locked tokens in `SecondSwap_StepVesting`, and subsequent calls to `available(Alice)` or `claimable(Alice)` at step 10 revert with underflow because `totalAmount (500e18) < amountClaimed (700e18)`.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Grantors who have claimed at least one step and subsequently transfer or list their vesting allocation have an inflated `releaseRate`, enabling them to claim more tokens via `claim()` than their `totalAmount`. This breaks the core protocol invariant that all tokens locked in a vesting plan must be allocated and claimable, causes underflows in `available()` and `claimable()`, and drains tokens belonging to other vesting beneficiaries (leaving the last beneficiaries unable to claim their tokens due to insufficient contract balance).

### Remediation

Update `transferVesting()` to compute `grantorVesting.releaseRate` using the remaining unclaimed amount `(grantorVesting.totalAmount - grantorVesting.amountClaimed)` divided by the remaining steps `(numOfSteps - grantorVesting.stepsClaimed)` (setting `releaseRate` to `0` if no steps remain):

```solidity
    function transferVesting(address _grantor, address _beneficiary, uint256 _amount) external {
        require(
            msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer,
            "SS_StepVesting: unauthorized"
        );
        require(_beneficiary != address(0), "SS_StepVesting: beneficiary is zero");
        require(_amount > 0, "SS_StepVesting: amount is zero");
        Vesting storage grantorVesting = _vestings[_grantor];
        require(
            grantorVesting.totalAmount - grantorVesting.amountClaimed >= _amount,
            "SS_StepVesting: insufficient balance"
        );

        grantorVesting.totalAmount -= _amount;
        if (numOfSteps - grantorVesting.stepsClaimed != 0) {
            grantorVesting.releaseRate =
                (grantorVesting.totalAmount - grantorVesting.amountClaimed) /
                (numOfSteps - grantorVesting.stepsClaimed);
        } else {
            grantorVesting.releaseRate = 0;
        }

        _createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);

        emit VestingTransferred(_grantor, _beneficiary, _amount);
    }
```

## Finding 4: Propagation of Grantor's `stepsClaimed` in `SecondSwap_StepVesting.transferVesting` Corrupts Marketplace Vesting State and Causes Division by Zero

### Synopsis

In `SecondSwap_StepVesting.transferVesting`, `grantorVesting.stepsClaimed` is passed to `_createVesting` when creating or updating the beneficiary's vesting schedule. When vestings are transferred through `SecondSwap_VestingManager`, this corrupts the manager's `stepsClaimed` state, allowing buyers to claim vested tokens prematurely or causing a division-by-zero panic at the end of the vesting period.

### Technical details

The `SecondSwap_StepVesting.transferVesting` function (lines 216–235) transfers `_amount` of vesting allocation from `_grantor` to `_beneficiary`. At line 232, it calls `_createVesting`:
```solidity
_createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);
```
Inside `_createVesting` (lines 279–310):
- If `_vestings[_beneficiary].totalAmount == 0` (line 288), a new `Vesting` struct is initialized for `_beneficiary` with `stepsClaimed` set to `_stepsClaimed` (i.e., `grantorVesting.stepsClaimed`) and `releaseRate` calculated as `_totalAmount / (numOfSteps - _stepsClaimed)` (line 292).
- If `_vestings[_beneficiary].totalAmount != 0` (line 295), `_vestings[_beneficiary].totalAmount` is increased by `_totalAmount`, while `_vestings[_beneficiary].stepsClaimed` remains unchanged and `_stepsClaimed` is ignored.

This design introduces severe state synchronization flaws across two primary flows:

1. **Division-by-Zero Panic at Terminal Vesting State:**
   If a grantor has already claimed all steps (`grantorVesting.stepsClaimed == numOfSteps`), or if `VestingManager` has `stepsClaimed == numOfSteps` (because the very first user to list on the marketplace had claimed `numOfSteps` before listing, or a seller lists when `stepsClaimed == numOfSteps` while `VestingManager.totalAmount == 0`), then `numOfSteps - _stepsClaimed` evaluates to `0`. When `_createVesting` executes line 292 (`releaseRate: _totalAmount / (numOfSteps - _stepsClaimed)`), it reverts with a division-by-zero panic. This permanently blocks `completePurchase`, `unlistVesting`, or `transferVesting` for new beneficiaries once `stepsClaimed == numOfSteps`.

2. **Incompatible `stepsClaimed` Aggregation in `VestingManager` and Beneficiary Accounts:**
   When sellers list vestings via `SecondSwap_VestingManager.listVesting`, `transferVesting(seller, address(VestingManager), amount)` is called.
   - The first seller (`Seller A`) to list when `_vestings[VestingManager].totalAmount == 0` sets `_vestings[VestingManager].stepsClaimed = sellerA.stepsClaimed`.
   - Because `VestingManager` never calls `claim()`, its `stepsClaimed` remains fixed at `sellerA.stepsClaimed` as long as its `totalAmount > 0`.
   - If `Seller A` had `stepsClaimed = 0` (listed at step 0), `_vestings[VestingManager].stepsClaimed` is `0`. Later, at step `k > 0`, `Seller B` claims `k` steps of their vesting (`sellerB.stepsClaimed = k`) and lists their remaining unvested tokens (`amount`) on the marketplace.
   - When `Buyer C` (who has no prior vesting, `totalAmount == 0`) purchases `Seller B`'s listing via `VestingManager.completePurchase`, `transferVesting(address(VestingManager), buyerC, amount)` is called.
   - `buyerC` receives `_stepsClaimed = _vestings[VestingManager].stepsClaimed = 0` instead of `k`.
   - Consequently, `buyerC` can immediately call `claim()` and claim `k` steps worth of tokens (`(amount / numOfSteps) * k`) that were meant to vest over the remaining `numOfSteps - k` steps, effectively double-claiming steps `0` through `k` on the same underlying allocation and accelerating the vesting schedule.
   - Conversely, if `Seller A` listed with `stepsClaimed = k`, `_vestings[VestingManager].stepsClaimed` becomes `k`. If `Seller B` (who has `stepsClaimed = 0`) later lists and unlists (or sells to a new buyer) before step `k`, the recipient receives `stepsClaimed = k`, causing an arithmetic underflow in `claimable()` (`currentStep - vesting.stepsClaimed` at line 174 where `currentStep < k`) until step `k` is reached, and compressing their release rate over `numOfSteps - k` steps.

### Assumptions and preconditions

- A `SecondSwap_StepVesting` contract is deployed and active with `numOfSteps > 0`.
- For the marketplace state corruption vector: `SecondSwap_VestingManager` is configured as the `manager` and marketplace listing is enabled (`sellable == true`).
- Users interact with `StepVesting` and `VestingManager` through normal protocol functions (`claim`, `listVesting`, `unlistVesting`, `completePurchase`, or `transferVesting`).

### Proof of concept

#### Scenario 1: Premature Vesting Claim / Double-Claiming Elapsed Steps via Marketplace

1. Assume a `SecondSwap_StepVesting` plan has `numOfSteps = 100`.
2. `Alice` is allocated `1,000` tokens (`stepsClaimed = 0`) and `Bob` is allocated `1,000` tokens (`stepsClaimed = 0`).
3. At step `0`, `Alice` lists `100` tokens on the marketplace via `listVesting`.
   - `transferVesting(Alice, VestingManager, 100)` executes.
   - Since `_vestings[VestingManager].totalAmount == 0`, `_vestings[VestingManager]` is initialized with `totalAmount = 100` and `stepsClaimed = Alice.stepsClaimed = 0`.
4. Time advances to step `50`.
5. `Bob` calls `claim()`, claiming `500` tokens for the first 50 steps. Now `Bob.stepsClaimed = 50`, `Bob.amountClaimed = 500`, and `Bob`'s remaining unvested balance is `500` tokens (which should vest linearly over steps 51 to 100).
6. `Bob` lists `200` of his remaining unvested tokens on the marketplace via `listVesting`.
   - `transferVesting(Bob, VestingManager, 200)` executes.
   - Because `_vestings[VestingManager].totalAmount` is `100` (`!= 0`), `_createVesting` takes the `else` branch (line 295). `_vestings[VestingManager].totalAmount` becomes `300`, and `_vestings[VestingManager].stepsClaimed` remains `0` (`Bob.stepsClaimed = 50` is ignored).
7. `Charlie` (a new address with `_vestings[Charlie].totalAmount == 0`) buys `Bob`'s `200` tokens via `completePurchase`.
   - `transferVesting(VestingManager, Charlie, 200)` executes.
   - `_createVesting(Charlie, 200, VestingManager.stepsClaimed, true)` initializes `_vestings[Charlie]` with `totalAmount = 200`, `stepsClaimed = 0`, and `releaseRate = 200 / (100 - 0) = 2`.
8. Immediately at step `50`, `Charlie` calls `claim()`.
   - In `claimable(Charlie)`, `currentStep = 50` and `Charlie.stepsClaimed = 0`, so `claimableSteps = 50`.
   - `Charlie` immediately claims `50 * 2 = 100` tokens at step 50, even though `Bob` already claimed the first 50 steps for that allocation and the `200` transferred tokens were strictly unvested tokens meant for steps 51–100.
   - Note that `Bob` could perform this self-purchase using a second wallet `Charlie` he controls to bypass the vesting schedule and unlock 50% of his unvested tokens immediately.

#### Scenario 2: Division-by-Zero DoS at Step `numOfSteps`

1. Time reaches `endTime` (`currentStep == numOfSteps = 100`).
2. `Alice` had previously listed `100` tokens at step `100` after claiming all her tokens, or `tokenIssuer` calls `transferVesting(Alice, Bob, amount)` where `Alice` has `stepsClaimed == 100` (e.g., `Alice` claimed at step 100 after receiving extra unclaimable or newly added tokens, or `VestingManager` was initialized when `seller.stepsClaimed == 100`).
3. In `_createVesting` (line 292), `numOfSteps - _stepsClaimed` is `100 - 100 = 0`.
4. The calculation `_totalAmount / (numOfSteps - _stepsClaimed)` reverts with a division-by-zero panic (Panic `0x12`), permanently locking the tokens inside `VestingManager` (blocking both `unlistVesting` and `completePurchase`).

### Impact assessment

- **Severity:** High
- **Technical Impact:**
  1. **Bypass of Vesting Schedule / Premature Token Release:** A seller who has already claimed up to the current step `k` can list their remaining unvested tokens and purchase them from a fresh address (or sell to a buyer) to reset `stepsClaimed` to `0` (or to whatever lower value `VestingManager.stepsClaimed` holds). By repeating this across secondary accounts, a user can immediately unlock and claim almost their entire unvested token allocation long before the vesting schedule matures.
  2. **Denial of Service / Locked Funds:** Underflow in `claimable()` (`currentStep - vesting.stepsClaimed`) if `VestingManager.stepsClaimed > currentStep`, or division-by-zero in `_createVesting()` if `_stepsClaimed == numOfSteps`, freezing tokens in `VestingManager`.
  3. **Contract Insolvency (Drainage):** When combined with how `grantorVesting` and `_beneficiary` (when `totalAmount > 0`) update their `releaseRate` and `totalAmount` across transfers with mismatched `stepsClaimed`, total claims can exceed actual allocations or lock remaining tokens.

### Remediation

Before transferring vesting between addresses, ensure that only unvested tokens are transferred (or force/account for pending claims) and synchronize `stepsClaimed` to the current active step rather than blindly copying `grantorVesting.stepsClaimed` into a shared pool like `VestingManager`. Alternatively, if `VestingManager` only ever holds unvested tokens or if transfers are synchronized to `currentStep`:

In `SecondSwap_StepVesting.sol`, update `transferVesting` and `_createVesting` so that `grantorVesting.releaseRate` and beneficiary vesting states properly account for claimed amounts and steps, and prevent division by zero when `_stepsClaimed == numOfSteps`:

```solidity
    function transferVesting(address _grantor, address _beneficiary, uint256 _amount) external {
        require(
            msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer,
            "SS_StepVesting: unauthorized"
        );
        require(_beneficiary != address(0), "SS_StepVesting: beneficiary is zero");
        require(_amount > 0, "SS_StepVesting: amount is zero");
        Vesting storage grantorVesting = _vestings[_grantor];
        require(
            grantorVesting.totalAmount - grantorVesting.amountClaimed >= _amount,
            "SS_StepVesting: insufficient balance"
        );

        grantorVesting.totalAmount -= _amount;
        if (numOfSteps - grantorVesting.stepsClaimed != 0) {
            grantorVesting.releaseRate =
                (grantorVesting.totalAmount - grantorVesting.amountClaimed) /
                (numOfSteps - grantorVesting.stepsClaimed);
        } else {
            grantorVesting.releaseRate = 0;
        }

        _createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);

        emit VestingTransferred(_grantor, _beneficiary, _amount);
    }
```

And in `_createVesting`, guard against `numOfSteps - _stepsClaimed == 0`:

```solidity
        if (_vestings[_beneficiary].totalAmount == 0) {
            uint256 remainingSteps = numOfSteps - _stepsClaimed;
            _vestings[_beneficiary] = Vesting({
                stepsClaimed: _stepsClaimed,
                amountClaimed: 0,
                releaseRate: remainingSteps != 0 ? _totalAmount / remainingSteps : 0,
                totalAmount: _totalAmount
            });
        }
```

## Finding 5: Flawed Quota Accounting in `SecondSwap_VestingManager.listVesting` and `unlistVesting` Miscalculates Sell Limits Across Listing, Unlisting, and Resale

### Synopsis

In `SecondSwap_VestingManager.listVesting` and `unlistVesting`, purchased token entitlements (`bought`) and restricted original-grant allocations are conflated into a single cumulative `sold` counter and combined with `currentAlloc`. This mismatch between historical counters and current allocations causes inaccurate `sellLimit` calculations across listing, cancellation, reallocation, and resale.

### Technical details

In `SecondSwap_VestingManager`, each user's marketplace activity for a vesting plan is tracked via `Allocation { uint256 bought; uint256 sold; }` in `allocations[seller][plan]`. The protocol intends for tokens purchased on the marketplace (`userAllocation.bought`) to be 100% resellable, while tokens from a user's original vesting grant are subject to `vestingSettings[plan].maxSellPercent`.

In `listVesting` (lines 125–140) and `unlistVesting` (lines 149–152):

```solidity
uint256 sellLimit = userAllocation.bought;
uint256 currentAlloc = SecondSwap_Vesting(plan).total(seller);

if (currentAlloc + userAllocation.sold > userAllocation.bought) {
    sellLimit +=
        ((currentAlloc + userAllocation.sold - userAllocation.bought) * vestingSettings[plan].maxSellPercent) /
        BASE;
}

userAllocation.sold += amount;

require(userAllocation.sold <= sellLimit, "SS_VestingManager: cannot list more than max sell percent");
SecondSwap_Vesting(plan).transferVesting(seller, address(this), amount);
```

This accounting mechanism has three interdependent flaws centered on lines 136–152:

1. **Purchased-entitlement and original-grant misclassification**:
   When a user lists tokens in `listVesting`, line 136 unconditionally increments `userAllocation.sold += amount` rather than distinguishing whether the listed tokens are drawn from the user's 100%-sellable `bought` entitlement or their `maxSellPercent`-restricted original grant (and `completePurchase` never decrements `sold` or separates purchased inventory from grant inventory). Similarly, `unlistVesting` (line 150) unconditionally decrements `allocations[seller][plan].sold -= amount`.

2. **Historical-versus-current allocation mismatch**:
   `userAllocation.sold` and `userAllocation.bought` are cumulative historical counters, whereas `currentAlloc` (`SecondSwap_Vesting(plan).total(seller)`) queries the user's current vesting balance. Because `SecondSwap_StepVesting.transferVesting` allows token issuers to reallocate vestings directly (a core protocol invariant), `currentAlloc` can change independently of `bought` and `sold`.

3. **Sell-quota inconsistencies across listing, cancellation, and resale**:
   - **Blocking resale of purchased tokens (DoS)**: If a user sells up to their `maxSellPercent` limit from their original grant (e.g., selling 500 of 1,000 tokens at 50% `maxSellPercent`, so `sold = 500`), and subsequently either `maxSellPercent` is lowered (e.g., to 20%) or part of the user's remaining grant is reallocated away via `transferVesting` (e.g., reducing `currentAlloc` by 300), the user's historical `sold` (`500`) exceeds the new grant sell limit. If the user then buys `100` tokens on the marketplace (`bought = 100`) and attempts to resell those `100` purchased tokens via `listVesting`, `userAllocation.sold` becomes `600` while `sellLimit` is only `100 + (500 * 20%) = 200` (or `100 + (1000 * 20%) = 300`). Line 138 (`require(userAllocation.sold <= sellLimit)`) reverts, preventing the user from reselling the tokens they purchased.
   - **Bypassing `maxSellPercent` after unlisting and reallocation**: If a user with a `1,000` original grant buys `200` tokens (`bought = 200`, `currentAlloc = 1,200`), lists and unlists, and then `200` tokens are transferred out via `transferVesting` (`currentAlloc = 1,000`), `userAllocation.bought` remains `200`. When calling `listVesting`, the transferred amount is deducted from `currentAlloc + sold - bought` (misclassifying the outflow as reducing the original grant to `800` while keeping the full `200` `bought` entitlement), giving `sellLimit = 200 + (800 * 20%) = 360` and allowing the user to sell 36% of their remaining `1,000` grant instead of 20%.

### Assumptions and preconditions

- A vesting plan is registered as `sellable` in `SecondSwap_VestingManager` with `maxSellPercent < BASE` (e.g., 20%).
- A user participates in listing, unlisting, and purchasing vesting allocations on the marketplace, and either `maxSellPercent` is adjusted or vesting allocations are transferred/reallocated by the token issuer via `transferVesting`.

### Proof of concept

1. Alice is granted `1,000e18` vesting tokens in `SecondSwap_StepVesting`. Suppose `maxSellPercent` is initially `5000` (50%) (or `2000` (20%) before a token issuer reallocation).
2. Alice lists and sells `200e18` tokens (at `maxSellPercent = 2000`):
   - `allocations[Alice][plan].sold = 200e18`
   - `allocations[Alice][plan].bought = 0`
   - `currentAlloc = 800e18`
3. The token issuer reallocates `500e18` of Alice's remaining unclaimable/vesting allocation to another beneficiary via `SecondSwap_StepVesting.transferVesting`:
   - `currentAlloc` for Alice is now `300e18`.
   - `allocations[Alice][plan].sold` remains `200e18`.
4. Alice purchases `100e18` vesting tokens on the marketplace via `spotPurchase` -> `completePurchase`:
   - `allocations[Alice][plan].bought = 100e18`
   - `currentAlloc` for Alice becomes `400e18`.
5. Alice attempts to resell the `100e18` tokens she just bought by calling `Marketplace.listVesting` for `amount = 100e18`:
   - In `SecondSwap_VestingManager.listVesting`:
     - `sellLimit` starts at `userAllocation.bought = 100e18`.
     - `currentAlloc + userAllocation.sold - userAllocation.bought` = `400e18 + 200e18 - 100e18 = 500e18`.
     - `sellLimit += (500e18 * 2000) / 10000 = 100e18`, so `sellLimit = 200e18`.
     - Line 136 updates `userAllocation.sold = 200e18 + 100e18 = 300e18`.
     - Line 138 checks `require(300e18 <= 200e18)`, which reverts with `"SS_VestingManager: cannot list more than max sell percent"`.
6. Alice's `100e18` purchased tokens are locked from being resold on the marketplace because her historical `sold` counter consumes her new `bought` quota.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: `sellLimit` in `SecondSwap_VestingManager.listVesting` either under-calculates the user's available sell quota (causing a denial of service when users attempt to resell tokens purchased on the marketplace) or over-calculates it (allowing users to bypass `maxSellPercent` restrictions when `bought` tokens are transferred out).
- **Business Impact**: Violates the marketplace's core allocation quota rules, breaking the guarantee that marketplace-purchased allocations can always be resold in full and that original grants remain bounded by `maxSellPercent`.

### Remediation

Track original-grant sales separately from purchased-token (`bought`) inventory (or consume `userAllocation.bought` first when listing and restore it appropriately on unlisting, while recording the original grant baseline separately from `bought` tokens).

```solidity
        if (amount <= userAllocation.bought) {
            userAllocation.bought -= amount;
        } else {
            uint256 grantAmount = amount - userAllocation.bought;
            userAllocation.bought = 0;

            uint256 maxGrantSell = ((currentAlloc - userAllocation.bought + userAllocation.sold) *
                vestingSettings[plan].maxSellPercent) / BASE;
            userAllocation.sold += grantAmount;
            require(userAllocation.sold <= maxGrantSell, "SS_VestingManager: cannot list more than max sell percent");
        }
        SecondSwap_Vesting(plan).transferVesting(seller, address(this), amount);
    }

    function unlistVesting(address seller, address plan, uint256 amount) external onlyMarketplace {
        Allocation storage userAllocation = allocations[seller][plan];
        if (userAllocation.sold >= amount) {
            userAllocation.sold -= amount;
        } else {
            uint256 boughtReturned = amount - userAllocation.sold;
            userAllocation.sold = 0;
            userAllocation.bought += boughtReturned;
        }
        SecondSwap_Vesting(plan).transferVesting(address(this), seller, amount);
    }
```
