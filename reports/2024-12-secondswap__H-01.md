# Audit report — 2024-12-secondswap/H-01

## Finding 1: Unvalidated External `decimals()` Call in `SecondSwap_Marketplace.listVesting` Causes Listing and Purchase DoS or Valuation Mismatch

### Synopsis

`SecondSwap_Marketplace.listVesting` casts the vesting plan's token to `IERC20Extended` and calls `decimals()` without validating return data encoding, bounding the returned exponent, or caching the result. Non-standard ERC-20 metadata responses, high decimals values, or mutable metadata responses cause unexpected reverts or inconsistent `baseAmount` calculations.

### Technical details

In `SecondSwap_Marketplace.listVesting`, the contract computes `baseAmount` to verify that the listed token amount and price do not round down to zero:

```solidity
uint256 baseAmount = (_amount * _price) /
    uint256(
        10 **
            (
                IERC20Extended(
                    address(
                        IVestingManager(IMarketplaceSetting(marketplaceSetting).vestingManager())
                            .getVestingTokenAddress(_vestingPlan)
                    )
                ).decimals()
            )
    );
require(baseAmount > 0, "SS_Marketplace: Cannot list amount it is too little");
```

This external metadata call introduces multiple trust and validation flaws:

1. **`decimals()` is optional in ERC-20 and `doesFunctionExist` is an insufficient check:**
   `decimals()` is part of the optional `IERC20Metadata` extension rather than the core EIP-20 specification. Although line 265 calls `doesFunctionExist(token, "decimals()")`, `doesFunctionExist` only executes `(bool success, ) = target.staticcall(abi.encodeWithSelector(selector))` and returns `success`. It does not verify that `returndata.length >= 32` or that the returned value fits within `uint8`. If a token does not implement `decimals()` (rendering the vesting plan unusable on the marketplace) or implements a non-reverting `fallback()` function that returns empty data, `doesFunctionExist` returns `true` while `IERC20Extended(...).decimals()` on line 285 reverts during ABI decoding.

2. **Missing upper-bound validation on `decimals()`:**
   Unlike the commented-out validation in `addCoin` (`require(decimals <= 18)`), `listVesting` applies no upper bound to the `uint8` value returned by `decimals()`. If a token returns `decimals() > 77`, the exponentiation `10 ** decimals` overflows `uint256` and reverts with an arithmetic panic. Even for values between `19` and `77`, the huge denominator `10 ** decimals` causes `(_amount * _price) / (10 ** decimals)` to truncate to `0` for normal prices and amounts, permanently blocking listings via `require(baseAmount > 0)`.

3. **Repeated uncached external metadata reads across lifecycle functions:**
   `listVesting` queries `decimals()` on line 285 to validate the listing, and `_handleTransfers` re-queries `decimals()` on line 468 during `spotPurchase` to compute the settlement `baseAmount`. Because the token's decimals value is not validated and snapshotted when the listing (or vesting plan) is created, any change in the external contract's `decimals()` response (e.g., via a proxy upgrade or dynamic wrapper/vault offset) between listing and purchase alters the denominator `10 ** decimals`, invalidating the seller's valuation assumptions or causing `spotPurchase` to revert.

### Assumptions and preconditions

1. A vesting plan is registered for an ERC-20 token whose `decimals()` function is either missing, handled by a fallback function, returns a value greater than `18` (or `> 77`), or is upgradeable/mutable over the lifetime of an active listing.
2. A seller attempts to list their vesting allocation via `listVesting`, or a buyer attempts to purchase from an active listing via `spotPurchase`.

### Proof of concept

1. Suppose a vesting plan uses an ERC-20 token contract that implements a silent `fallback()` (returning 0 bytes) without implementing `decimals()`, or returns a `uint256` value outside the `uint8` / `<= 77` range.
2. When a seller calls `listVesting`, `doesFunctionExist(tokenAddress, "decimals()")` performs `target.staticcall(abi.encodeWithSelector(0x313ce567))`, which succeeds (`success == true`).
3. Execution proceeds to lines 280–285: `IERC20Extended(tokenAddress).decimals()`.
4. If `returndata` is empty (from a fallback) or exceeds `type(uint8).max`, the Solidity ABI decoder reverts the transaction; if `decimals` is between `78` and `255`, `10 ** decimals` reverts with Panic `0x11` (arithmetic overflow).
5. Alternatively, if the token is behind an upgradeable proxy or dynamic wrapper whose `decimals()` changes from `18` to `6` after `listVesting` succeeds, a subsequent `spotPurchase` call computes `baseAmount = (_amount * discountedPrice) / (10 ** 6)` instead of `/ (10 ** 18)`, charging the buyer $10^{12}$ times the intended currency amount.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Denial of Service in `listVesting` and `spotPurchase` for tokens with non-standard or out-of-range `decimals()` implementations, or corrupted payment calculations (`baseAmount`) if `decimals()` changes between listing creation and purchase execution.
- **Business Impact:** Legitimate vesting plans backed by valid ERC-20 tokens that do not strictly conform to `IERC20Metadata` with `decimals <= 18` cannot be traded on the marketplace, and dynamic metadata responses can lead to severe mispricing during settlement.

### Remediation

Validate `decimals()` safely using a low-level `staticcall` with return-data length and bounds checking (or a `try/catch` block that defaults to `18` when `decimals()` is not supported, depending on protocol requirements), enforce a strict upper bound (such as `decimals <= 18`), and cache the validated decimals or unit scale in the `Listing` struct so that `_handleTransfers` uses the same immutable denominator established at listing time.

```solidity
        address vestingToken = address(
            IVestingManager(IMarketplaceSetting(marketplaceSetting).vestingManager())
                .getVestingTokenAddress(_vestingPlan)
        );
        uint8 tokenDecimals;
        try IERC20Extended(vestingToken).decimals() returns (uint8 dec) {
            require(dec > 0 && dec <= 18, "SS_Marketplace: Invalid token decimals");
            tokenDecimals = dec;
        } catch {
            revert("SS_Marketplace: No valid decimals function");
        }

        uint256 baseAmount = (_amount * _price) / (10 ** uint256(tokenDecimals));
        require(baseAmount > 0, "SS_Marketplace: Cannot list amount it is too little");
```

## Finding 2: Uncapped `claimableSteps` in `SecondSwap_StepVesting.claimable` pushes `stepsClaimed` above `numOfSteps` and causes underflow DoS

### Synopsis

In `SecondSwap_StepVesting.claimable`, when `currentStep` exceeds `numOfSteps` due to integer truncation of `stepDuration`, `claimableSteps` is returned uncapped. Calling `claim()` sets `vesting.stepsClaimed > numOfSteps`, causing arithmetic underflow in `_createVesting` and permanently blocking the beneficiary from unlisting or receiving vestings.

### Technical details

In `SecondSwap_StepVesting.constructor` (line 133), the duration of each step is computed using integer division:

```solidity
stepDuration = (_endTime - _startTime) / _numOfSteps;
```

Whenever `(_endTime - _startTime)` is not an exact multiple of `_numOfSteps`, `stepDuration` is truncated downward such that `numOfSteps * stepDuration < endTime - startTime`. When `(endTime - startTime) % numOfSteps >= stepDuration`, as `block.timestamp` approaches or reaches `endTime`, `currentStep` in `claimable()` (line 173) evaluates to a value strictly greater than `numOfSteps`:

```solidity
uint256 elapsedTime = currentTime - startTime;
uint256 currentStep = elapsedTime / stepDuration;
uint256 claimableSteps = currentStep - vesting.stepsClaimed;

uint256 claimableAmount;

if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
    //[BUG FIX] user can buy more than they are allocated
    claimableAmount = vesting.totalAmount - vesting.amountClaimed;
    return (claimableAmount, claimableSteps);
}
```

In the terminal branch (`if (vesting.stepsClaimed + claimableSteps >= numOfSteps)` at lines 178–182), `claimableAmount` is capped at `vesting.totalAmount - vesting.amountClaimed`, but `claimableSteps` is returned uncapped at line 181 instead of being bounded to `numOfSteps - vesting.stepsClaimed`.

When the beneficiary subsequently executes `claim()` (lines 193–203):

```solidity
(uint256 claimableAmount, uint256 claimableSteps) = claimable(msg.sender);
require(claimableAmount > 0, "SS_StepVesting: nothing to claim");

Vesting storage vesting = _vestings[msg.sender];
vesting.stepsClaimed += claimableSteps;
```

`vesting.stepsClaimed` is incremented to `currentStep > numOfSteps`.

However, `_createVesting` (lines 297–303) assumes the invariant `vesting.stepsClaimed <= numOfSteps`:

```solidity
if (numOfSteps - _vestings[_beneficiary].stepsClaimed != 0) {
    _vestings[_beneficiary].releaseRate =
        (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
        (numOfSteps - _vestings[_beneficiary].stepsClaimed);
} else {
    _vestings[_beneficiary].releaseRate = 0;
}
```

Once `_vestings[_beneficiary].stepsClaimed > numOfSteps`, any call to `_createVesting` for that beneficiary—including `SecondSwap_VestingManager.unlistVesting`, `SecondSwap_VestingManager.completePurchase`, `transferVesting`, or `createVesting`—reverts with a Solidity `Panic(0x11)` arithmetic underflow at `numOfSteps - _vestings[_beneficiary].stepsClaimed`.

### Assumptions and preconditions

1. A `SecondSwap_StepVesting` contract is deployed where `(endTime - startTime) % numOfSteps >= stepDuration` (for example, `endTime - startTime = 1000`, `numOfSteps = 300`, yielding `stepDuration = 3`).
2. A beneficiary has an active vesting schedule (`totalAmount > 0`) and claims tokens via `claim()` at a timestamp where `elapsedTime / stepDuration > numOfSteps` (e.g., at `block.timestamp >= endTime`).
3. The beneficiary later attempts to unlist a previously listed vesting via `SecondSwap_VestingManager.unlistVesting`, purchase a vesting via `completePurchase`, or receive an allocation via `transferVesting` / `createVesting`.

### Proof of concept

1. A vesting plan is deployed with `startTime = 1000`, `endTime = 2000`, and `numOfSteps = 300`. The constructor computes `stepDuration = (2000 - 1000) / 300 = 3`.
2. Alice is granted `1000e18` tokens via `createVesting(alice, 1000e18)`.
3. Alice lists `200e18` tokens on the marketplace via `listVesting(alice, plan, 200e18)`. Alice's `totalAmount` becomes `800e18`, and `SecondSwap_VestingManager` holds `200e18`.
4. Time advances to `block.timestamp = 2000` (`endTime`).
5. Alice calls `claim()`:
   - In `claimable(alice)`, `elapsedTime = 2000 - 1000 = 1000` and `currentStep = 1000 / 3 = 333`.
   - `claimableSteps = 333 - 0 = 333`.
   - Since `0 + 333 >= 300`, `claimableAmount = 800e18 - 0 = 800e18`, and `claimable(alice)` returns `(800e18, 333)`.
   - `claim()` sets `alice.stepsClaimed = 333` and `alice.amountClaimed = 800e18`, transferring `800e18` tokens to Alice.
6. Alice attempts to unlist her `200e18` tokens from the marketplace, which calls `SecondSwap_VestingManager.unlistVesting(alice, plan, 200e18)` -> `SecondSwap_StepVesting.transferVesting(manager, alice, 200e18)` -> `_createVesting(alice, 200e18, 0, true)`.
7. Because `_vestings[alice].totalAmount` is `800e18 != 0`, `_createVesting` enters the `else` branch and evaluates `numOfSteps - _vestings[alice].stepsClaimed` (`300 - 333`), reverting with `Panic(0x11)` arithmetic underflow. Alice's `200e18` tokens are permanently locked in `SecondSwap_VestingManager`.

### Impact assessment

**Severity: High**

- **Permanent Freezing of Funds:** Any beneficiary who lists a portion of their vesting on the marketplace and claims their remaining vested tokens near or after `endTime` will have `stepsClaimed > numOfSteps`. They can never unlist their listed tokens from `SecondSwap_VestingManager`, permanently locking those tokens in the manager contract.
- **Violation of Core Protocol Invariants:** Token issuers can no longer reallocate or add vestings to the affected beneficiary via `transferVesting` or `createVesting`, directly violating the stated protocol invariants that *"All tokens locked in a vesting plan must be allocated and claimable"* and *"The token issuers must always be able to reallocate vestings."*

### Remediation

Cap `claimableSteps` to `numOfSteps - vesting.stepsClaimed` inside the terminal branch of `claimable()` so that `vesting.stepsClaimed` never exceeds `numOfSteps`:

```solidity
        uint256 elapsedTime = currentTime - startTime;
        uint256 currentStep = elapsedTime / stepDuration;
        uint256 claimableSteps = currentStep - vesting.stepsClaimed;

        uint256 claimableAmount;

        if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
            //[BUG FIX] user can buy more than they are allocated
            claimableSteps = numOfSteps - vesting.stepsClaimed;
            claimableAmount = vesting.totalAmount - vesting.amountClaimed;
            return (claimableAmount, claimableSteps);
        }

        claimableAmount = vesting.releaseRate * claimableSteps;
        return (claimableAmount, claimableSteps);
```

## Finding 3: Incorrect `releaseRate` Recalculation in `SecondSwap_StepVesting.transferVesting` Allows Grantors to Overclaim Tokens

### Synopsis

In `SecondSwap_StepVesting.transferVesting`, the grantor's `releaseRate` is recalculated as `grantorVesting.totalAmount / numOfSteps` without deducting `grantorVesting.amountClaimed` or accounting for `grantorVesting.stepsClaimed`. After claiming some steps and transferring or listing remaining tokens, a grantor can continue claiming unallocated tokens in subsequent steps, draining the vesting contract.

### Technical details

When a vesting allocation is transferred in `SecondSwap_StepVesting.transferVesting` (either directly by `tokenIssuer` / `vestingDeployer` or when a user lists their vesting on the marketplace via `SecondSwap_VestingManager.listVesting`), the grantor's `totalAmount` and `releaseRate` are updated on lines 229–230:

```solidity
grantorVesting.totalAmount -= _amount;
grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;
```

This calculation is flawed in two ways whenever the grantor has previously claimed tokens (`grantorVesting.stepsClaimed > 0` and `grantorVesting.amountClaimed > 0`):

1. **Numerator includes already-claimed tokens:** `grantorVesting.totalAmount` after subtracting `_amount` still includes `grantorVesting.amountClaimed`. Even if the grantor transfers 100% of their remaining unclaimed allocation (`_amount == grantorVesting.totalAmount - grantorVesting.amountClaimed`), `grantorVesting.totalAmount` remains equal to `grantorVesting.amountClaimed` (which is non-zero), resulting in a positive `grantorVesting.releaseRate`.
2. **Denominator ignores already-claimed steps:** Dividing by `numOfSteps` instead of the remaining steps `(numOfSteps - grantorVesting.stepsClaimed)` further distorts the per-step release rate for any remaining unclaimed tokens.

In `claimable(address _beneficiary)` (lines 178–185), for any step prior to the final step (`vesting.stepsClaimed + claimableSteps < numOfSteps`), the claimable amount is computed solely from `vesting.releaseRate * claimableSteps` without capping it at `vesting.totalAmount - vesting.amountClaimed`:

```solidity
if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
    claimableAmount = vesting.totalAmount - vesting.amountClaimed;
    return (claimableAmount, claimableSteps);
}

claimableAmount = vesting.releaseRate * claimableSteps;
return (claimableAmount, claimableSteps);
```

Consequently:
- **Overclaiming / Draining funds:** If a user claims at an early step, transfers (or lists on the marketplace) their entire remaining unclaimed balance (`totalAmount - amountClaimed`), their remaining available balance is `0`, yet `grantorVesting.totalAmount` equals `amountClaimed > 0` and `grantorVesting.releaseRate = amountClaimed / numOfSteps > 0`. In any subsequent step before `numOfSteps`, the user can call `claim()` to steal `releaseRate * claimableSteps` additional tokens from the contract.
- **Underclaiming prior to final step:** Conversely, if a user has claimed very little or nothing as `currentStep` approaches `numOfSteps`, or if `(totalAmount - amountClaimed) / (numOfSteps - stepsClaimed)` should be larger than `totalAmount / numOfSteps`, dividing by the full `numOfSteps` reduces their `releaseRate` below the intended rate over the remaining steps.

Compare this with `_createVesting` (lines 297–303), which properly calculates `releaseRate` as:
```solidity
(_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
    (numOfSteps - _vestings[_beneficiary].stepsClaimed)
```

### Assumptions and preconditions

1. A vesting schedule exists in `SecondSwap_StepVesting` with `numOfSteps > 1` and holds tokens for beneficiaries.
2. At least one step has elapsed (`currentStep >= 1` and `currentStep < numOfSteps`), and the grantor claims their vested tokens so that `grantorVesting.amountClaimed > 0` and `grantorVesting.stepsClaimed > 0`.
3. `transferVesting` is invoked for the grantor (either via marketplace listing through `SecondSwap_VestingManager` or reallocation by `tokenIssuer`/`vestingDeployer`).

### Proof of concept

Consider a `SecondSwap_StepVesting` contract deployed with `numOfSteps = 10` and `stepDuration = 1 days`:

1. **Initial Allocation:** Alice is allocated `1,000e18` tokens (`totalAmount = 1,000e18`, `releaseRate = 100e18`, `stepsClaimed = 0`, `amountClaimed = 0`).
2. **Claim at Step 5:** After 5 days (`currentStep = 5`), Alice calls `claim()`.
   - `claimableSteps = 5`, `claimableAmount = 500e18`.
   - Alice receives `500e18` tokens.
   - Alice's state is now: `totalAmount = 1,000e18`, `amountClaimed = 500e18`, `stepsClaimed = 5`, `releaseRate = 100e18`.
   - Alice has `available = 1,000e18 - 500e18 = 500e18` unclaimed tokens remaining.
3. **Transfer of Remaining Allocation:** At Step 5, Alice's remaining `500e18` tokens are transferred to Bob (or listed on the marketplace via `listVesting`, which calls `transferVesting(alice, manager, 500e18)`):
   - Requirement `grantorVesting.totalAmount - grantorVesting.amountClaimed >= 500e18` passes (`1,000e18 - 500e18 >= 500e18`).
   - `grantorVesting.totalAmount` is reduced to `1,000e18 - 500e18 = 500e18`.
   - `grantorVesting.releaseRate` is updated to `500e18 / 10 = 50e18` (instead of `(500e18 - 500e18) / (10 - 5) = 0`).
   - Bob (or the manager) receives a vesting allocation of `500e18` tokens with `stepsClaimed = 5` and `releaseRate = 500e18 / (10 - 5) = 100e18`.
4. **Overclaiming at Step 9:** At Step 9 (`currentStep = 9`), `vesting.stepsClaimed + claimableSteps = 5 + 4 = 9 < 10`:
   - Alice calls `claim()`.
   - `claimable(alice)` returns `claimableAmount = 50e18 * 4 = 200e18` and `claimableSteps = 4`.
   - Alice receives an extra `200e18` tokens even though she already claimed `500e18` and transferred the remaining `500e18`.
   - Alice's `amountClaimed` becomes `700e18` while her `totalAmount` is `500e18`, stealing `200e18` tokens belonging to other beneficiaries in the vesting contract.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Grantors who have claimed tokens and subsequently have their remaining vesting transferred (or listed on the marketplace) retain a non-zero `releaseRate` based on already-claimed amounts, enabling them to claim more tokens than their `totalAmount`.
- **Business Impact:** Direct theft of locked vesting tokens from the `SecondSwap_StepVesting` contract, leaving the contract insolvent and causing other beneficiaries' `claim()` calls at the end of the vesting schedule to revert due to insufficient token balance.

### Remediation

Update `SecondSwap_StepVesting.transferVesting` to calculate `grantorVesting.releaseRate` based on the remaining unclaimed amount `(grantorVesting.totalAmount - grantorVesting.amountClaimed)` divided by the remaining unclaimed steps `(numOfSteps - grantorVesting.stepsClaimed)`, handling the case where `numOfSteps == grantorVesting.stepsClaimed`:

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
        ); // 3.8. Claimed amount not checked in transferVesting function

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

## Finding 4: Improper `stepsClaimed` Handling and Schedule Aggregation in `SecondSwap_StepVesting._createVesting` Allows Premature Claiming and Draining of Vested Tokens

### Synopsis

`SecondSwap_StepVesting._createVesting` improperly inherits and aggregates vesting schedules with divergent `stepsClaimed` histories under a single beneficiary record. Through direct transfers or `SecondSwap_VestingManager` marketplace listings, users can reset or reduce `stepsClaimed` on transferred allocations, enabling premature claiming of unvested tokens and draining the vesting contract.

### Technical details

In `SecondSwap_StepVesting.sol`, each beneficiary has a single `Vesting` struct stored in `_vestings[_beneficiary]` tracking `stepsClaimed`, `amountClaimed`, `releaseRate`, and `totalAmount`.

When `transferVesting` is called (either directly by an authorized caller or via `SecondSwap_VestingManager` during `listVesting`, `unlistVesting`, and `completePurchase`), it deducts `_amount` from `grantorVesting.totalAmount` and invokes `_createVesting`:

```solidity
_createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);
```

Inside `_createVesting` (lines 279–304):
1. **When `_vestings[_beneficiary].totalAmount == 0` (lines 288–294):**
   The beneficiary's `stepsClaimed` is initialized to the incoming `_stepsClaimed`, and `releaseRate` is computed as `_totalAmount / (numOfSteps - _stepsClaimed)`.
2. **When `_vestings[_beneficiary].totalAmount != 0` (lines 295–304):**
   The incoming `_stepsClaimed` parameter is completely ignored. `_totalAmount` is added to `_vestings[_beneficiary].totalAmount`, and `releaseRate` is recomputed over the beneficiary's existing remaining steps:
   ```solidity
   _vestings[_beneficiary].releaseRate =
       (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
       (numOfSteps - _vestings[_beneficiary].stepsClaimed);
   ```

This design creates critical state transition flaws across two vectors:

1. **Direct / Buyer Schedule Aggregation Across Divergent Claim Histories:**
   If a beneficiary (such as a marketplace buyer) already holds a small vesting allocation with `stepsClaimed = 0` (unclaimed) and receives a transferred allocation from a grantor who has already claimed `k` steps (`_stepsClaimed = k`), `_createVesting` takes the `else` branch. The recipient's `stepsClaimed` remains `0`, and the newly transferred `_totalAmount` is distributed across all `numOfSteps` instead of `numOfSteps - k`. Consequently, `claimable()` immediately credits the recipient with `k` elapsed steps on the newly added tokens, allowing those steps to be claimed a second time before they are due.

2. **Shared Escrow Aggregation in `SecondSwap_VestingManager`:**
   `SecondSwap_VestingManager` holds all active marketplace listings inside a single `_vestings[manager]` record:
   - When the first seller lists tokens (`_vestings[manager].totalAmount == 0`), `_vestings[manager].stepsClaimed` is set to that seller's `stepsClaimed`.
   - Any subsequent listings from other sellers hit the `else` branch in `_createVesting`, discarding each seller's `stepsClaimed`.
   - If the first listing came from an account with `stepsClaimed = 0`, `_vestings[manager].stepsClaimed` stays `0`. Another seller who has already claimed `k` steps can list their remaining unvested tokens and immediately unlist them (or buy them from another address with `totalAmount == 0`), receiving the allocation back from `manager` with `stepsClaimed = 0`. They can then immediately call `claim()` to collect `k` steps of tokens again.
   - Conversely, if the first listing came from an account with a high `stepsClaimed = k` (e.g., `k = numOfSteps - 1`), `_vestings[manager].stepsClaimed` is set to `k`. A seller with `stepsClaimed = 0` who lists and unlists (or a buyer who purchases) when their own `totalAmount == 0` will inherit `stepsClaimed = k` and `releaseRate = _amount / (numOfSteps - k)`, accelerating their entire vesting schedule so that almost all unvested tokens become immediately claimable at step `k + 1`.

### Assumptions and preconditions

- A `SecondSwap_StepVesting` contract is deployed with `numOfSteps > 1` and funded with vesting tokens for multiple beneficiaries (or a single beneficiary using two accounts).
- At least one vesting step has elapsed (`block.timestamp >= startTime + stepDuration`).
- Either the marketplace (`SecondSwap_VestingManager`) is enabled for the vesting plan, or `transferVesting` is invoked to a beneficiary with an existing vesting allocation.

### Proof of concept

Consider a `SecondSwap_StepVesting` plan with `numOfSteps = 10` and `stepDuration = 1 day`:
1. Alice is allocated `1,000` tokens (`stepsClaimed = 0`, `releaseRate = 100`).
2. Bob is allocated `10` tokens (`stepsClaimed = 0`, `releaseRate = 1`).
3. At Step 5 (`5 days` after `startTime`):
   - Bob lists `10` tokens on the marketplace via `listVesting`. Since `_vestings[manager].totalAmount == 0`, `_createVesting` sets `_vestings[manager].stepsClaimed = 0` and `_vestings[manager].totalAmount = 10`.
   - Alice calls `claim()` and receives `5 * 100 = 500` tokens. Alice's state is now `stepsClaimed = 5`, `amountClaimed = 500`, `totalAmount = 1,000`.
4. Alice lists her remaining `500` unvested tokens via `listVesting`:
   - `transferVesting(alice, manager, 500)` reduces `alice.totalAmount` to `500` (equal to `alice.amountClaimed`) and calls `_createVesting(manager, 500, 5, true)`.
   - Because `_vestings[manager].totalAmount == 10 != 0`, the `else` branch executes: `_vestings[manager].stepsClaimed` remains `0` and `_vestings[manager].totalAmount` becomes `510`.
5. Alice transfers/buys the `500` tokens into a fresh address `Alice2` (where `_vestings[Alice2].totalAmount == 0`) via `completePurchase`:
   - `transferVesting(manager, Alice2, 500)` calls `_createVesting(Alice2, 500, 0, true)` using `manager`'s `stepsClaimed = 0`.
   - `Alice2` receives `totalAmount = 500`, `stepsClaimed = 0`, `releaseRate = 500 / 10 = 50`.
6. Still at Step 5, `Alice2` immediately calls `claim()`:
   - `claimableSteps = 5 - 0 = 5`.
   - `Alice2` claims `5 * 50 = 250` additional tokens immediately, even though Alice had already claimed all tokens for Steps 1–5.
7. By repeating the list-and-buy (or list-and-unlist into an account with `totalAmount == 0`) cycle, the attacker can continue claiming unvested tokens prematurely until the `SecondSwap_StepVesting` contract's token balance is drained.

### Impact assessment

- **Severity:** High
- **Direct Technical Impact:** Beneficiaries can bypass the vesting schedule to claim unvested tokens immediately and claim more tokens than entitled at the current step.
- **Business Impact:** Because attackers extract tokens belonging to other beneficiaries before the vesting schedule completes, the contract becomes insolvent and honest beneficiaries experience permanent loss of funds when attempting to claim their vested tokens.

### Remediation

When creating or updating a vesting schedule in `_createVesting`, do not allow a beneficiary's `stepsClaimed` to mismatch the incoming allocation or pool multiple sellers' schedules into a single `Vesting` struct on `VestingManager` without accounting for already-elapsed steps.

Specifically, when adding to an existing vesting schedule or transferring vesting allocations:
1. Automatically claim any currently claimable tokens for both the grantor and the beneficiary before modifying their vesting schedules so that both accounts are synchronized to `currentStep` (except for `manager`, which should not distort `stepsClaimed`).
2. Or adjust `amountClaimed` and `totalAmount` proportionally so that the effective `releaseRate` over the remaining steps `(numOfSteps - currentStep)` preserves the exact unclaimed entitlement without resetting `stepsClaimed`:

```solidity
    function _createVesting(
        address _beneficiary,
        uint256 _totalAmount,
        uint256 _stepsClaimed,
        bool _isInternal
    ) internal {
        require(_beneficiary != address(0), "SS_StepVesting: beneficiary is zero");
        require(_totalAmount > 0, "SS_StepVesting: total amount is zero");

        uint256 currentTime = Math.min(block.timestamp, endTime);
        uint256 currentStep = currentTime > startTime ? (currentTime - startTime) / stepDuration : 0;

        if (_beneficiary != manager) {
            (uint256 claimableAmount, uint256 claimableSteps) = claimable(_beneficiary);
            if (claimableAmount > 0) {
                _vestings[_beneficiary].stepsClaimed += claimableSteps;
                _vestings[_beneficiary].amountClaimed += claimableAmount;
                token.safeTransfer(_beneficiary, claimableAmount);
                emit Claimed(_beneficiary, claimableAmount);
            }
        }

        if (_vestings[_beneficiary].totalAmount == 0) {
            uint256 effectiveStepsClaimed = _isInternal ? _stepsClaimed : 0;
            _vestings[_beneficiary] = Vesting({
                stepsClaimed: effectiveStepsClaimed,
                amountClaimed: 0,
                releaseRate: numOfSteps > effectiveStepsClaimed
                    ? _totalAmount / (numOfSteps - effectiveStepsClaimed)
                    : 0,
                totalAmount: _totalAmount
            });
        } else {
            _vestings[_beneficiary].totalAmount += _totalAmount;
            if (_isInternal && _stepsClaimed > _vestings[_beneficiary].stepsClaimed) {
                _vestings[_beneficiary].stepsClaimed = currentStep;
            }
            if (numOfSteps > _vestings[_beneficiary].stepsClaimed) {
                _vestings[_beneficiary].releaseRate =
                    (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
                    (numOfSteps - _vestings[_beneficiary].stepsClaimed);
            } else {
                _vestings[_beneficiary].releaseRate = 0;
            }
        }

        if (!_isInternal) {
            token.safeTransferFrom(tokenIssuer, address(this), _totalAmount);
        }
        emit VestingCreated(_beneficiary, _totalAmount);
    }
```

## Finding 5: Incorrect `releaseRate` Calculation in `SecondSwap_StepVesting.transferVesting` Allows Grantors to Overclaim Vested Tokens

### Synopsis

In `SecondSwap_StepVesting.transferVesting`, the grantor's `releaseRate` is recalculated as `grantorVesting.totalAmount / numOfSteps` instead of dividing the remaining unclaimed allocation by the remaining vesting steps. A grantor who has already claimed tokens and transfers or lists their remaining allocation retains a non-zero `releaseRate` and can overclaim tokens.

### Technical details

In `SecondSwap_StepVesting`, each beneficiary's vesting state is tracked by the `Vesting` struct (`stepsClaimed`, `amountClaimed`, `releaseRate`, and `totalAmount`).

When an existing beneficiary's allocation is increased in `_createVesting` (lines 296–303), `releaseRate` is updated by spreading the remaining unclaimed tokens across the remaining unclaimed steps:

```solidity
_vestings[_beneficiary].totalAmount += _totalAmount;
if (numOfSteps - _vestings[_beneficiary].stepsClaimed != 0) {
    _vestings[_beneficiary].releaseRate =
        (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
        (numOfSteps - _vestings[_beneficiary].stepsClaimed);
} else {
    _vestings[_beneficiary].releaseRate = 0;
}
```

However, when a grantor's allocation is reduced in `transferVesting` (lines 229–232), `grantorVesting.releaseRate` is computed as:

```solidity
grantorVesting.totalAmount -= _amount;
grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;

_createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);
```

This calculation is flawed for two reasons:
1. `grantorVesting.totalAmount` after subtracting `_amount` still includes `grantorVesting.amountClaimed` (the tokens the grantor has already claimed in prior steps).
2. The denominator uses the total `numOfSteps` rather than the remaining steps `numOfSteps - grantorVesting.stepsClaimed`.

Meanwhile, `claimable(address _beneficiary)` (lines 178–185) only caps `claimableAmount` to `vesting.totalAmount - vesting.amountClaimed` when `vesting.stepsClaimed + claimableSteps >= numOfSteps`:

```solidity
if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
    //[BUG FIX] user can buy more than they are allocated
    claimableAmount = vesting.totalAmount - vesting.amountClaimed;
    return (claimableAmount, claimableSteps);
}

claimableAmount = vesting.releaseRate * claimableSteps;
return (claimableAmount, claimableSteps);
```

For any step prior to the final step (`currentStep < numOfSteps`), `claimable()` simply returns `vesting.releaseRate * claimableSteps` without checking whether `vesting.amountClaimed + claimableAmount > vesting.totalAmount`. Consequently, if a grantor claims their vested tokens partway through the schedule and then transfers their remaining unclaimed allocation (either via `SecondSwap_VestingManager.listVesting` when listing on the marketplace or via `SecondSwap_VestingDeployer.transferVesting`), `grantorVesting.totalAmount` remains equal to (or greater than) `grantorVesting.amountClaimed > 0`. This leaves `grantorVesting.releaseRate > 0`, allowing the grantor to continue claiming tokens on subsequent steps up to `numOfSteps - 1` and receive more tokens than their `totalAmount`.

### Assumptions and preconditions

1. A vesting plan (`SecondSwap_StepVesting`) is deployed with `numOfSteps > 2` and funded with tokens for beneficiaries.
2. A beneficiary (`_grantor`) has a vesting allocation and has claimed tokens after at least one step (`stepsClaimed > 0` and `amountClaimed > 0`).
3. Before `numOfSteps - 1` steps have elapsed, the grantor's allocation is partially or fully transferred via `transferVesting` (for example, by the user listing their vested tokens on the marketplace via `listVesting`, or by the `tokenIssuer` reallocating vesting).

### Proof of concept

Consider a `SecondSwap_StepVesting` contract with `numOfSteps = 100` and a beneficiary Alice with `totalAmount = 1,000e18` tokens (`releaseRate = 10e18` per step):

1. **Initial Claim at Step 50:**
   - When `currentStep == 50`, Alice calls `claim()`.
   - `claimable(alice)` returns `(500e18, 50)`.
   - Alice receives `500e18` tokens. Her state is now:
     - `stepsClaimed = 50`
     - `amountClaimed = 500e18`
     - `totalAmount = 1,000e18`
     - `available(alice) = 500e18`

2. **Transfer of Remaining Unclaimed Allocation at Step 50:**
   - Alice's remaining `500e18` unclaimed tokens are transferred via `transferVesting(alice, beneficiary, 500e18)` (or Alice lists `200e18` on the marketplace via `listVesting`).
   - Using `_amount = 500e18`:
     - `grantorVesting.totalAmount` becomes `1,000e18 - 500e18 = 500e18`.
     - `grantorVesting.releaseRate` is set to `500e18 / 100 = 5e18` (instead of `(500e18 - 500e18) / (100 - 50) = 0`).
     - `_beneficiary` receives a vesting schedule for the full `500e18` tokens.

3. **Overclaiming at Step 99:**
   - When `currentStep == 99`, Alice calls `claim()`.
   - In `claimable(alice)`:
     - `claimableSteps = 99 - 50 = 49`.
     - `vesting.stepsClaimed + claimableSteps = 50 + 49 = 99 < 100` (`numOfSteps`), so the maturity check at line 178 is not triggered.
     - `claimableAmount = 5e18 * 49 = 245e18`.
   - `claim()` transfers an extra `245e18` tokens to Alice and updates `alice.amountClaimed` to `745e18`, exceeding her `totalAmount` of `500e18`.

### Impact assessment

**Severity: High**

- **Direct Loss of Funds:** A grantor who has claimed tokens and subsequently transfers or lists their remaining vesting allocation can claim more tokens than their `totalAmount`. Because all beneficiaries' tokens are held in the same `SecondSwap_StepVesting` contract, the excess tokens claimed by the grantor directly drain tokens allocated to other beneficiaries.
- **Protocol Insolvency & DoS at Maturity:** Once one or more grantors overclaim tokens from `SecondSwap_StepVesting`, the contract's token balance becomes insufficient to honor other beneficiaries' legitimate `claim()` calls, causing their transactions to revert. Additionally, once `amountClaimed > totalAmount`, calling `available(_grantor)` or `claimable(_grantor)` at `currentStep == numOfSteps` reverts with an arithmetic underflow.

### Remediation

Update `transferVesting` in `SecondSwap_StepVesting.sol` to calculate `grantorVesting.releaseRate` using the remaining unclaimed amount `(grantorVesting.totalAmount - grantorVesting.amountClaimed)` divided by the remaining steps `(numOfSteps - grantorVesting.stepsClaimed)`, matching the logic in `_createVesting`:

```solidity
        grantorVesting.totalAmount -= _amount;
        if (numOfSteps - grantorVesting.stepsClaimed != 0) {
            grantorVesting.releaseRate =
                (grantorVesting.totalAmount - grantorVesting.amountClaimed) /
                (numOfSteps - grantorVesting.stepsClaimed);
        } else {
            grantorVesting.releaseRate = 0;
        }

        _createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);
```

## Finding 6: Improper Authorization in `SecondSwap_StepVesting.createVesting` Allows Unauthorized Vesting Creation and Spending of `tokenIssuer` Allowance

### Synopsis

`SecondSwap_StepVesting.createVesting` grants `vestingDeployer` and `manager` the same authorization as `tokenIssuer`, while `_createVesting` always pulls tokens from `tokenIssuer`. Due to token-level rather than instance-level validation in `SecondSwap_VestingDeployer`, another token owner for the same token can spend `tokenIssuer`'s approved tokens to create arbitrary vestings.

### Technical details

In `SecondSwap_StepVesting.sol`, the `createVesting` function restricts access using the following check (lines 244–249):

```solidity
function createVesting(address _beneficiary, uint256 _totalAmount) external {
    require(
        msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer,
        "SS_StepVesting: unauthorized"
    );
    _createVesting(_beneficiary, _totalAmount, 0, false);
}
```

When `_createVesting` is called with `_isInternal = false`, it transfers `_totalAmount` of `token` directly from `tokenIssuer` into the vesting contract (lines 306–308):

```solidity
if (!_isInternal) {
    token.safeTransferFrom(tokenIssuer, address(this), _totalAmount);
}
```

This creates a delegated authorization mismatch across three boundaries:

1. **Caller Authority Mismatch with Issuer-Funded Spending:** `createVesting` allows `manager` and `vestingDeployer` to initiate non-internal vesting creation, yet `_createVesting` unconditionally pulls the underlying ERC20 tokens from `tokenIssuer` via `safeTransferFrom(tokenIssuer, address(this), _totalAmount)`. `manager` never needs to call `createVesting` (it only uses `transferVesting`), and allowing non-issuer callers to trigger spending from `tokenIssuer` breaks the authority boundary over `tokenIssuer`'s token allowance.
2. **Token-Level vs. Vesting-Instance Authority Mismatch:** In `SecondSwap_VestingDeployer.sol`, `setTokenOwner(address token, address _owner)` maps `_tokenOwner[_owner] = token` (lines 141–144), allowing multiple `_owner` addresses to be associated with the same `token`. When `SecondSwap_VestingDeployer.createVesting` is called (lines 176–183), it only verifies token-level ownership:
   ```solidity
   require(
       _tokenOwner[msg.sender] == address(SecondSwap_StepVesting(_stepVesting).token()),
       "SS_VestingDeployer: caller is not the token owner"
   );
   ```
   It does not verify that `msg.sender == SecondSwap_StepVesting(_stepVesting).tokenIssuer()`.
3. **Confused-Deputy Privileged Forwarding:** Because `SecondSwap_StepVesting.createVesting` trusts `msg.sender == vestingDeployer`, `SecondSwap_VestingDeployer` acts as a confused deputy. Any other address registered in `_tokenOwner` for the same ERC20 token can call `SecondSwap_VestingDeployer.createVesting(_beneficiary, _totalAmount, _stepVesting)` targeting Victim `tokenIssuer`'s `_stepVesting` instance, creating vesting allocations for arbitrary beneficiaries funded by Victim `tokenIssuer`'s ERC20 allowance.

### Assumptions and preconditions

1. A `SecondSwap_StepVesting` contract (`vestingA`) has been deployed for `token` with `issuerA` as its `tokenIssuer`.
2. `issuerA` has approved `vestingA` to spend `token` (either with a remaining unused allowance, a pre-approval before calling `createVesting`, or an infinite approval).
3. Either another address `issuerB` is also registered in `SecondSwap_VestingDeployer._tokenOwner` for `token` (`_tokenOwner[issuerB] == address(token)`), or `vestingDeployer` / `manager` forwards a call to `vestingA.createVesting`.

### Proof of concept

1. Admin registers `issuerA` and `issuerB` as token owners for `token` via `vestingDeployer.setTokenOwner(address(token), issuerA)` and `vestingDeployer.setTokenOwner(address(token), issuerB)` (e.g., two organizational wallets or sub-issuers conducting separate vesting campaigns for the same token).
2. `issuerA` deploys a vesting plan `vestingA` via `vestingDeployer.deployVesting(address(token), startTime, endTime, steps, "PlanA")`. In `vestingA`, `tokenIssuer` is set to `issuerA`.
3. `issuerA` approves `vestingA` for `100,000e18` of `token` in preparation for creating vesting schedules (or leaves a remaining approval on `vestingA`).
4. `issuerB` calls `vestingDeployer.createVesting(issuerB, 100_000e18, address(vestingA))`.
5. `SecondSwap_VestingDeployer.createVesting` checks `_tokenOwner[issuerB] == address(vestingA.token())`, which passes because both equal `address(token)`.
6. `vestingDeployer` calls `vestingA.createVesting(issuerB, 100_000e18)`.
7. `vestingA.createVesting` checks `msg.sender == vestingDeployer`, which passes, and invokes `_createVesting(issuerB, 100_000e18, 0, false)`.
8. `_createVesting` executes `token.safeTransferFrom(issuerA, address(vestingA), 100_000e18)`, pulling `100,000e18` tokens from `issuerA` and crediting the vesting schedule to `issuerB`.
9. `issuerB` claims the vested tokens from `vestingA` as steps elapse, stealing `issuerA`'s tokens.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Unauthorized callers (such as another token owner for the same token via `vestingDeployer`, or `manager`) can invoke `createVesting` on a `SecondSwap_StepVesting` instance and force it to pull ERC20 tokens from `tokenIssuer` into arbitrary beneficiary vesting schedules. Existing beneficiaries' `releaseRate` values can also be manipulated or diluted via the `else` branch of `_createVesting` due to integer division truncation when adding small amounts.
- **Business Implications:** Direct theft of `tokenIssuer` funds up to their ERC20 allowance granted to the `SecondSwap_StepVesting` contract, and corruption of vesting schedules.

### Remediation

Remove `manager` from the authorization check in `SecondSwap_StepVesting.createVesting` (as `manager` only transfers existing vestings and never creates new ones), and ensure that `SecondSwap_VestingDeployer` verifies `msg.sender == SecondSwap_StepVesting(_stepVesting).tokenIssuer()` (or restrict `SecondSwap_StepVesting.createVesting` to only `tokenIssuer` or a verified `vestingDeployer` call from `tokenIssuer`).

In `contracts/SecondSwap_StepVesting.sol`:

```solidity
    function createVesting(address _beneficiary, uint256 _totalAmount) external {
        require(
            msg.sender == tokenIssuer || msg.sender == vestingDeployer,
            "SS_StepVesting: unauthorized"
        );
        _createVesting(_beneficiary, _totalAmount, 0, false);
    }
```

Additionally, in `contracts/SecondSwap_VestingDeployer.sol`, verify that `msg.sender` is the `tokenIssuer` of `_stepVesting`:

```solidity
    function createVesting(address _beneficiary, uint256 _totalAmount, address _stepVesting) external {
        require(
            msg.sender == SecondSwap_StepVesting(_stepVesting).tokenIssuer() &&
                _tokenOwner[msg.sender] == address(SecondSwap_StepVesting(_stepVesting).token()),
            "SS_VestingDeployer: caller is not the token owner"
        );
        SecondSwap_StepVesting(_stepVesting).createVesting(_beneficiary, _totalAmount);
        emit VestingCreated(_beneficiary, _totalAmount, _stepVesting);
    }
```

## Finding 7: Accounting Discrepancy With Fee-On-Transfer Tokens in `SecondSwap_StepVesting._createVesting` Causes Insolvency and Unclaimable Allocations

### Synopsis

In `SecondSwap_StepVesting._createVesting` (invoked by `createVesting` and `createVestings`), the beneficiary's vesting allocation is credited with the nominal `_totalAmount` rather than the actual tokens received via `safeTransferFrom`. For fee-on-transfer or deflationary tokens, this overstates vested balances, causing contract insolvency and reverting late claims.

### Technical details

When new vesting schedules are created via `SecondSwap_StepVesting.createVesting` (line 244) or `SecondSwap_StepVesting.createVestings` (lines 260–266), each call delegates to the internal function `_createVesting(_beneficiary, _totalAmount, 0, false)` (lines 279–310).

Inside `_createVesting`, the contract performs the following operations:
1. Updates the beneficiary's vesting record (`_vestings[_beneficiary].totalAmount` and `releaseRate`) using the raw input parameter `_totalAmount` (lines 288–304).
2. Pulls tokens from `tokenIssuer` into the vesting contract via `token.safeTransferFrom(tokenIssuer, address(this), _totalAmount)` when `!_isInternal` is true (lines 306–308).

Furthermore, in `createVestings` (lines 264–266), `safeTransferFrom` is executed inside the loop on every iteration via `_createVesting` without measuring the actual balance delta on `token.balanceOf(address(this))`.

If `token` is a fee-on-transfer (or deflationary/tax) ERC20 token, the actual amount of tokens credited to `address(this)` during `safeTransferFrom` is less than `_totalAmount`. Because the contract records the nominal `_totalAmount` in `_vestings[_beneficiary].totalAmount` and calculates `releaseRate` based on `_totalAmount`, the sum of allocated tokens across beneficiaries exceeds the actual token balance held by `SecondSwap_StepVesting`.

This directly violates the core protocol invariant stated in the documentation:
> *"All tokens locked in a vesting plan must be allocated and claimable (with the exception for tokens being transferred in directly)."*

When beneficiaries subsequently call `claim()` (line 193) or when all steps have elapsed (lines 178–181: `claimableAmount = vesting.totalAmount - vesting.amountClaimed`), the contract attempts to transfer more tokens than it actually holds, causing `token.safeTransfer` to revert due to insufficient token balance.

### Assumptions and preconditions

1. A token with a transfer fee (fee-on-transfer) is onboarded and deployed with a `SecondSwap_StepVesting` contract.
2. An authorized caller (`tokenIssuer`, `manager`, or `vestingDeployer`) calls `createVesting` or `createVestings` to fund vesting schedules for one or more beneficiaries.
3. The token deducts a fee on `transferFrom` calls when transferring from `tokenIssuer` to the `SecondSwap_StepVesting` contract.

### Proof of concept

1. Assume `token` applies a 5% fee on transfers and `numOfSteps = 100`.
2. `tokenIssuer` approves `SecondSwap_StepVesting` and calls `createVestings([alice, bob], [1000e18, 1000e18])`.
3. In `_createVesting`, `alice` is assigned `totalAmount = 1000e18` (`releaseRate = 10e18`) and `bob` is assigned `totalAmount = 1000e18` (`releaseRate = 10e18`). Total allocated obligations equal `2000e18`.
4. Each `token.safeTransferFrom(tokenIssuer, address(this), 1000e18)` deducts a 5% fee (`50e18`), so `SecondSwap_StepVesting` only receives `950e18 + 950e18 = 1900e18` tokens in total.
5. After `endTime` is reached (`currentStep == 100`), `alice` calls `claim()` and receives her full `1000e18` tokens, leaving `900e18` tokens in the contract.
6. When `bob` calls `claim()`, `claimable(bob)` returns `1000e18`, and `token.safeTransfer(bob, 1000e18)` reverts because the contract only holds `900e18` tokens. `bob` is permanently unable to claim his vested tokens.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: The internal accounting (`_vestings[_beneficiary].totalAmount` and `releaseRate`) becomes desynchronized from the actual ERC20 balance held by `SecondSwap_StepVesting`.
- **Business Impact**: The vesting contract becomes insolvent. Early claimants or marketplace buyers who claim first drain the tokens needed to cover later claimants, resulting in a permanent denial of service (revert on `claim()`) and loss of vested funds for the remaining beneficiaries. Additionally, `createVestings` performs redundant external `safeTransferFrom` calls on every loop iteration rather than a single aggregated transfer.

### Remediation

Measure the actual amount of tokens received by checking the contract's token balance before and after `safeTransferFrom` when `!_isInternal` is true, and use the actual received amount to update `_vestings[_beneficiary]`.

```solidity
    function _createVesting(
        address _beneficiary,
        uint256 _totalAmount,
        uint256 _stepsClaimed,
        bool _isInternal
    ) internal {
        require(_beneficiary != address(0), "SS_StepVesting: beneficiary is zero");
        require(_totalAmount > 0, "SS_StepVesting: total amount is zero");

        if (!_isInternal) {
            uint256 balanceBefore = token.balanceOf(address(this));
            token.safeTransferFrom(tokenIssuer, address(this), _totalAmount);
            _totalAmount = token.balanceOf(address(this)) - balanceBefore;
            require(_totalAmount > 0, "SS_StepVesting: zero amount received");
        }

        if (_vestings[_beneficiary].totalAmount == 0) {
            _vestings[_beneficiary] = Vesting({
                stepsClaimed: _stepsClaimed,
                amountClaimed: 0,
                releaseRate: _totalAmount / (numOfSteps - _stepsClaimed),
                totalAmount: _totalAmount
            });
        } else {
            _vestings[_beneficiary].totalAmount += _totalAmount;
            if (numOfSteps - _vestings[_beneficiary].stepsClaimed != 0) {
                _vestings[_beneficiary].releaseRate =
                    (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
                    (numOfSteps - _vestings[_beneficiary].stepsClaimed);
            } else {
                _vestings[_beneficiary].releaseRate = 0;
            }
        }

        emit VestingCreated(_beneficiary, _totalAmount);
    }
```

## Finding 8: Incorrect Release Rate and Step Accounting When Updating Allocations in `SecondSwap_StepVesting.createVestings`

### Synopsis

In `SecondSwap_StepVesting.createVestings`, batch-creating vesting allocations via `_createVesting(_beneficiaries[i], _totalAmounts[i], 0, false)` merges new tokens into existing beneficiary schedules or mid-schedule allocations without accounting for elapsed unclaimed steps (`currentStep` vs. `stepsClaimed`), distorting `releaseRate` and immediately claimable token entitlements.

### Technical details

`SecondSwap_StepVesting.createVestings` (lines 260–266) iterates over `_beneficiaries` and `_totalAmounts` and invokes `_createVesting(_beneficiaries[i], _totalAmounts[i], 0, false)` at line 265.

Inside `_createVesting` (lines 279–310), vesting entitlement accounting relies on two branches:

1. **New Beneficiary (`_vestings[_beneficiary].totalAmount == 0`, lines 288–294):**
   `stepsClaimed` is initialized to `_stepsClaimed` (`0` when called from `createVestings`), and `releaseRate` is computed as `_totalAmount / (numOfSteps - _stepsClaimed)`.
2. **Existing Beneficiary (`_vestings[_beneficiary].totalAmount != 0`, lines 295–304):**
   `totalAmount` is increased by `_totalAmount`, the passed `_stepsClaimed` argument is ignored, `_vestings[_beneficiary].stepsClaimed` is left unchanged, and `releaseRate` is recomputed as:
   ```solidity
   _vestings[_beneficiary].releaseRate =
       (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
       (numOfSteps - _vestings[_beneficiary].stepsClaimed);
   ```

Meanwhile, `claimable(address _beneficiary)` (lines 161–186) computes:
```solidity
uint256 elapsedTime = currentTime - startTime;
uint256 currentStep = elapsedTime / stepDuration;
uint256 claimableSteps = currentStep - vesting.stepsClaimed;
...
claimableAmount = vesting.releaseRate * claimableSteps;
```

This creates a fundamental accounting flaw whenever `createVestings` adds allocations after `startTime` (`currentStep > 0`) or across beneficiaries with different claim histories (`stepsClaimed`):

- **Time-Dependent Release-Rate Recalculation Across Claim Histories:** Because `_vestings[_beneficiary].stepsClaimed` only updates when the beneficiary calls `claim()`, two beneficiaries with identical vesting allocations added at the same timestamp receive different `releaseRate` values and different immediately claimable amounts depending on when they last claimed:
  - If a beneficiary claimed right before `createVestings` at `currentStep` (`stepsClaimed == currentStep`), `claimableSteps` is `0` and the newly added `_totalAmount` is spread strictly over the remaining `numOfSteps - currentStep` steps (`0` tokens from the new allocation are immediately claimable).
  - If a beneficiary has not claimed since an earlier step `stepsClaimed < currentStep` (or `stepsClaimed == 0`), `_createVesting` recalculates `releaseRate` over `numOfSteps - stepsClaimed` without settling the `currentStep - stepsClaimed` already-elapsed steps at the prior `releaseRate`. When the beneficiary subsequently calls `claim()`, the newly inflated `releaseRate` is multiplied by the entire `claimableSteps = currentStep - stepsClaimed`, retroactively unlocking `(currentStep - stepsClaimed) / (numOfSteps - stepsClaimed)` of the newly added `_totalAmount` immediately.
- **Retroactive Unlocking on Mid-Schedule Batch Creation:** Because line 265 hardcodes `_stepsClaimed = 0` when calling `_createVesting`, any new beneficiary added via `createVestings` after `startTime` (`currentStep > 0`) starts with `stepsClaimed = 0` instead of `currentStep`, allowing them to immediately claim `currentStep` steps of the newly created vesting schedule.

### Assumptions and preconditions

1. A `SecondSwap_StepVesting` contract is deployed with `startTime < endTime` and `numOfSteps > 1`.
2. The current block timestamp is past `startTime + stepDuration` (`currentStep >= 1`).
3. An authorized caller (`tokenIssuer`, `manager`, or `vestingDeployer`) calls `createVestings` to allocate tokens to beneficiaries.

### Proof of concept

Consider a vesting schedule with:
- `startTime = 1000`, `endTime = 2000`, `numOfSteps = 100` (`stepDuration = 10`).
- At `t = 1000` (`currentStep = 0`), Alice and Bob are each granted `1,000` tokens (`releaseRate = 10` tokens/step).

1. At `t = 1100` (`currentStep = 10`), Alice calls `claim()` and receives `100` tokens (`stepsClaimed = 10`, `amountClaimed = 100`).
2. Time advances to `t = 1500` (`currentStep = 50`).
   - Bob calls `claim()` at `t = 1500`, receiving `500` tokens (`bob.stepsClaimed = 50`, `bob.amountClaimed = 500`).
   - Alice has not called `claim()` since `t = 1100`, so `alice.stepsClaimed = 10` and `alice.amountClaimed = 100` (with `400` tokens currently claimable for steps 11–50).
3. At `t = 1500` (`currentStep = 50`), the token issuer calls `createVestings([alice, bob], [900, 900])`:
   - **For Bob (`stepsClaimed = 50`, `amountClaimed = 500`):**
     - `bob.totalAmount = 1000 + 900 = 1900`.
     - `bob.releaseRate = (1900 - 500) / (100 - 50) = 1400 / 50 = 28` tokens/step.
     - At `t = 1500`, `claimable(bob)` returns `(0, 0)`. Bob has claimed `500` tokens total at step 50, and his new `900` tokens vest strictly over steps 51–100.
   - **For Alice (`stepsClaimed = 10`, `amountClaimed = 100`):**
     - `alice.totalAmount = 1000 + 900 = 1900`.
     - `alice.releaseRate = (1900 - 100) / (100 - 10) = 1800 / 90 = 20` tokens/step.
     - Immediately at `t = 1500`, Alice calls `claim()`:
       - `claimableSteps = 50 - 10 = 40`.
       - `claimableAmount = 20 * 40 = 800` tokens.
     - Including the `100` tokens claimed at step 10, Alice has now claimed `900` tokens at step 50 (`500` from her original allocation plus `400` of the newly added `900` tokens immediately unlocked), whereas Bob could only claim `500` tokens at step 50 despite having the exact same allocations granted at the exact same timestamps.

### Impact assessment

- **Severity:** Medium / High.
- **Technical Impact:** The `releaseRate` and `claimable` calculations become inconsistent whenever allocations are added via `createVestings` to users with different `stepsClaimed` values or when unclaimed steps exist (`currentStep > stepsClaimed`).
- **Business Impact:** Beneficiaries who refrain from claiming (or who have not claimed recent steps) before a batch vesting top-up can immediately claim a large fraction of newly added vesting tokens that should remain locked for future steps, while beneficiaries who claimed up to `currentStep` before `createVestings` have their new allocation locked over the remaining steps (or penalized if the new allocation was intended to vest from `startTime`).

### Remediation

Before updating an existing beneficiary's allocation or `releaseRate`, settle or checkpoint the beneficiary's accrued entitlement up to `currentStep` (or pass the appropriate starting step when creating mid-schedule vestings) so that `releaseRate` recalculation only applies to future steps and does not retroactively alter already-elapsed steps.

```solidity
    function createVestings(address[] memory _beneficiaries, uint256[] memory _totalAmounts) external {
        require(msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer, "unauthorized");
        require(_beneficiaries.length == _totalAmounts.length, "SS_StepVesting: array length mismatch");

        uint256 currentTime = Math.min(block.timestamp, endTime);
        uint256 currentStep = currentTime > startTime ? (currentTime - startTime) / stepDuration : 0;

        for (uint256 i = 0; i < _beneficiaries.length; i++) {
            address beneficiary = _beneficiaries[i];
            if (_vestings[beneficiary].totalAmount != 0) {
                (uint256 claimableAmount, uint256 claimableSteps) = claimable(beneficiary);
                if (claimableAmount > 0) {
                    _vestings[beneficiary].stepsClaimed += claimableSteps;
                    _vestings[beneficiary].amountClaimed += claimableAmount;
                    token.safeTransfer(beneficiary, claimableAmount);
                    emit Claimed(beneficiary, claimableAmount);
                }
            }
            _createVesting(beneficiary, _totalAmounts[i], currentStep, false);
        }
    }
```

## Finding 9: Overly Permissive Authorization in `SecondSwap_StepVesting.createVestings` Enables Confused-Deputy Token Drainage from `tokenIssuer`

### Synopsis

`SecondSwap_StepVesting.createVestings` authorizes calls from `vestingDeployer` and `manager` in addition to `tokenIssuer`, while always pulling vested funds from `tokenIssuer`. Because `SecondSwap_VestingDeployer` only validates token-level ownership rather than schedule-level `tokenIssuer` identity, another issuer for the same token can create arbitrary vestings funded by `tokenIssuer`'s allowance.

### Technical details

In `contracts/SecondSwap_StepVesting.sol`, `createVestings` checks caller authorization on line 261:

```solidity
function createVestings(address[] memory _beneficiaries, uint256[] memory _totalAmounts) external {
    require(msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer, "unauthorized");
    require(_beneficiaries.length == _totalAmounts.length, "SS_StepVesting: array length mismatch");

    for (uint256 i = 0; i < _beneficiaries.length; i++) {
        _createVesting(_beneficiaries[i], _totalAmounts[i], 0, false);
    }
}
```

For each entry in the batch, `_createVesting` is invoked with `_isInternal = false`. Inside `_createVesting` (lines 306-308), the contract transfers `_totalAmount` of `token` directly from the schedule's `tokenIssuer`:

```solidity
if (!_isInternal) {
    token.safeTransferFrom(tokenIssuer, address(this), _totalAmount);
}
```

However, `SecondSwap_VestingDeployer` enforces only token-level authorization rather than schedule-level authorization:
1. `SecondSwap_VestingDeployer.setTokenOwner(address token, address _owner)` only checks `require(_tokenOwner[_owner] == address(0))`, allowing multiple `_owner` addresses to be mapped to the same `token`.
2. `SecondSwap_VestingDeployer.createVestings` only verifies that `_tokenOwner[msg.sender] == address(SecondSwap_StepVesting(_stepVesting).token())` before forwarding the call to `SecondSwap_StepVesting(_stepVesting).createVestings(...)`.

Because `SecondSwap_StepVesting.createVestings` trusts `vestingDeployer` (and `manager`) as an authorized `msg.sender` without verifying that the initiator is the specific `tokenIssuer` of this vesting schedule, `SecondSwap_VestingDeployer` acts as a confused deputy. Any other address registered as a `_tokenOwner` for the same ERC20 token can call `SecondSwap_VestingDeployer.createVestings` targeting victim `tokenIssuer`'s `SecondSwap_StepVesting` instance. The target vesting contract accepts the call from `vestingDeployer` and pulls tokens from `tokenIssuer`'s ERC20 allowance to fund vestings for attacker-chosen beneficiaries.

### Assumptions and preconditions

1. A `SecondSwap_StepVesting` contract has been deployed for a token with `tokenIssuer` (Issuer A), and Issuer A has granted an ERC20 allowance to the `SecondSwap_StepVesting` contract.
2. Another account (Issuer B) is also registered in `SecondSwap_VestingDeployer._tokenOwner` for the same token (e.g., when multiple issuers or vesting programs exist for the same token).

### Proof of concept

1. Admin registers Issuer A and Issuer B as token owners for `TokenX` via `SecondSwap_VestingDeployer.setTokenOwner(address(TokenX), issuerA)` and `setTokenOwner(address(TokenX), issuerB)`.
2. Issuer A deploys a `SecondSwap_StepVesting` instance (`vestingA`) via `SecondSwap_VestingDeployer.deployVesting`, setting `vestingA.tokenIssuer = issuerA`.
3. Issuer A approves `vestingA` to spend `1,000,000 TokenX` so that vesting schedules can be funded.
4. Issuer B calls `SecondSwap_VestingDeployer.createVestings([attackerBeneficiary], [1_000_000e18], address(vestingA))`.
5. `SecondSwap_VestingDeployer.createVestings` checks `_tokenOwner[issuerB] == address(vestingA.token())`, which evaluates to `true`, and calls `vestingA.createVestings([attackerBeneficiary], [1_000_000e18])`.
6. `vestingA.createVestings` checks `msg.sender == vestingDeployer` (line 261), which passes, and calls `_createVesting(attackerBeneficiary, 1_000_000e18, 0, false)`.
7. `_createVesting` executes `token.safeTransferFrom(issuerA, address(vestingA), 1_000_000e18)`, draining Issuer A's approved tokens and allocating the vesting schedule to `attackerBeneficiary`.

### Impact assessment

- **Severity**: High
- **Technical Impact**: Unauthorized consumption of `tokenIssuer`'s ERC20 allowance and creation of unauthorized vesting schedules in `SecondSwap_StepVesting`.
- **Business Impact**: Direct theft/misappropriation of tokens from a `tokenIssuer`'s approved balance by another token issuer of the same token, corrupting the vesting plan's accounting and draining issuer funds.

### Remediation

Restrict `createVestings` in `SecondSwap_StepVesting` so that only `tokenIssuer` can directly fund and create new vestings, or ensure that any forwarded call through `vestingDeployer` strictly authenticates that the caller is the schedule's `tokenIssuer` (and remove `manager` from `createVestings` authorization since `manager` only transfers existing vestings via `transferVesting`).

```solidity
    function createVestings(address[] memory _beneficiaries, uint256[] memory _totalAmounts) external {
        require(msg.sender == tokenIssuer, "SS_StepVesting: unauthorized");
        require(_beneficiaries.length == _totalAmounts.length, "SS_StepVesting: array length mismatch");

        for (uint256 i = 0; i < _beneficiaries.length; i++) {
            _createVesting(_beneficiaries[i], _totalAmounts[i], 0, false);
        }
    }
```

## Finding 10: Unchecked Division by Zero in `SecondSwap_StepVesting._createVesting` Reverts Transfers from Exhausted Vesting Schedules

### Synopsis

In `SecondSwap_StepVesting._createVesting`, the new-beneficiary initialization branch calculates `releaseRate` as `_totalAmount / (numOfSteps - _stepsClaimed)` without checking whether `numOfSteps - _stepsClaimed` is non-zero. When a grantor with `stepsClaimed == numOfSteps` transfers vesting to a beneficiary with no existing allocation, the transaction reverts with division by zero.

### Technical details

In `contracts/SecondSwap_StepVesting.sol`, `_createVesting` handles two cases depending on whether `_vestings[_beneficiary].totalAmount == 0`:

```solidity
        if (_vestings[_beneficiary].totalAmount == 0) {
            _vestings[_beneficiary] = Vesting({
                stepsClaimed: _stepsClaimed,
                amountClaimed: 0,
                releaseRate: _totalAmount / (numOfSteps - _stepsClaimed),
                totalAmount: _totalAmount
            });
        } else {
            _vestings[_beneficiary].totalAmount += _totalAmount;
            if (numOfSteps - _vestings[_beneficiary].stepsClaimed != 0) {
                _vestings[_beneficiary].releaseRate =
                    (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
                    (numOfSteps - _vestings[_beneficiary].stepsClaimed);
            } else {
                _vestings[_beneficiary].releaseRate = 0;
            }
        }
```

In the `else` branch (line 297), the contract explicitly checks `if (numOfSteps - _vestings[_beneficiary].stepsClaimed != 0)` and sets `releaseRate = 0` when all `numOfSteps` have already been claimed. However, in the `if (_vestings[_beneficiary].totalAmount == 0)` branch (line 292), the contract directly divides `_totalAmount` by `(numOfSteps - _stepsClaimed)` without guarding against `_stepsClaimed == numOfSteps`.

A grantor legitimately reaches a state where `grantorVesting.stepsClaimed == numOfSteps` while still holding transferable, unclaimed tokens (`grantorVesting.totalAmount - grantorVesting.amountClaimed > 0`) through standard protocol operations:

1. A beneficiary (`grantor`) claims their vested tokens once `block.timestamp >= endTime` (or once all steps have elapsed). In `claim()`, `grantorVesting.stepsClaimed` becomes `numOfSteps` and `grantorVesting.amountClaimed` equals `grantorVesting.totalAmount` (which remains non-zero).
2. Subsequently, the `grantor` receives additional vesting tokens—either because the `tokenIssuer` calls `createVesting`, the `grantor` unlists a previously listed marketplace offer via `SecondSwap_VestingManager.unlistVesting`, or the `grantor` purchases tokens via `completePurchase`.
3. Because `_vestings[grantor].totalAmount != 0`, `_createVesting` executes the `else` branch: `_vestings[grantor].totalAmount` increases by `_totalAmount`, `numOfSteps - _vestings[grantor].stepsClaimed == 0` causes `releaseRate` to be set to `0`, and `_vestings[grantor].stepsClaimed` remains `numOfSteps`.
4. When `transferVesting` is subsequently called to transfer any amount from `grantor` to a `_beneficiary` who has `_vestings[_beneficiary].totalAmount == 0` (such as when the `tokenIssuer` reallocates the vesting via `SecondSwap_VestingDeployer.transferVesting`, or when `grantor` lists the vesting on a `SecondSwap_VestingManager` with zero existing allocation), `transferVesting` invokes `_createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true)` with `_stepsClaimed = numOfSteps`.
5. Execution enters the `if (_vestings[_beneficiary].totalAmount == 0)` branch at line 288 and evaluates `_totalAmount / (numOfSteps - _stepsClaimed)` at line 292, dividing by `numOfSteps - numOfSteps = 0` and reverting with a Solidity division-by-zero panic (`Panic(0x12)`).

Furthermore, if `(_endTime - _startTime)` is not an exact multiple of `_numOfSteps`, integer truncation in `stepDuration = (_endTime - _startTime) / _numOfSteps` allows `currentStep = elapsedTime / stepDuration` at `endTime` to exceed `numOfSteps`. Once a user claims at `endTime`, their `stepsClaimed` becomes `currentStep > numOfSteps`, causing `numOfSteps - _stepsClaimed` at line 292 (and line 297) to revert with arithmetic underflow (`Panic(0x11)`).

### Assumptions and preconditions

1. A `SecondSwap_StepVesting` contract is deployed and reaches `endTime` (all `numOfSteps` have elapsed).
2. A beneficiary (`grantor`) claims their vested allocation at or after `endTime`, setting `_vestings[grantor].stepsClaimed = numOfSteps`.
3. The `grantor` receives additional vesting allocation via `createVesting`, `unlistVesting`, or `completePurchase`.
4. An authorized caller (`tokenIssuer`, `vestingDeployer`, or `manager`) calls `transferVesting` from `grantor` to a `_beneficiary` whose `_vestings[_beneficiary].totalAmount == 0`.

### Proof of concept

1. Deploy `SecondSwap_StepVesting` with `startTime = 1000`, `endTime = 2000`, and `numOfSteps = 10` (`stepDuration = 100`).
2. `tokenIssuer` calls `createVesting(alice, 1000e18)` at `block.timestamp = 1000`, and Alice lists `200e18` on the marketplace via `listVesting(alice, vesting, 200e18)` (leaving Alice with `totalAmount = 800e18`).
3. Advance `block.timestamp` to `2000` (`endTime`).
4. Alice calls `claim()`. In `claimable(alice)`, `currentStep = (2000 - 1000) / 100 = 10`, `claimableSteps = 10`, and `claimableAmount = 800e18`. Alice's state becomes:
   - `stepsClaimed = 10` (`== numOfSteps`)
   - `amountClaimed = 800e18`
   - `totalAmount = 800e18`
5. Alice unlists her `200e18` listing via `unlistVesting(alice, vesting, 200e18)` (or `tokenIssuer` grants Alice `200e18` via `createVesting(alice, 200e18)`).
6. `_createVesting` executes the `else` branch because `_vestings[alice].totalAmount == 800e18 != 0`:
   - `_vestings[alice].totalAmount` becomes `1000e18`
   - `numOfSteps - _vestings[alice].stepsClaimed == 10 - 10 == 0`, so `_vestings[alice].releaseRate = 0`
   - `_vestings[alice].stepsClaimed` remains `10`
7. `tokenIssuer` attempts to reallocate `200e18` from Alice to Bob (where `_vestings[bob].totalAmount == 0`) by calling `transferVesting(alice, bob, 200e18)`.
8. `transferVesting` calls `_createVesting(bob, 200e18, 10, true)`.
9. Because `_vestings[bob].totalAmount == 0`, execution enters the `if` branch at line 288 and computes `releaseRate: 200e18 / (10 - 10)` at line 292, reverting with `Panic(0x12)` (division by zero).

### Impact assessment

- **Severity**: Medium
- **Technical Effect**: Calling `transferVesting` from any grantor with `stepsClaimed == numOfSteps` to a beneficiary with `totalAmount == 0` always reverts with a division-by-zero panic.
- **Business Impact**: Directly violates the protocol's core invariant stated in `README.md` (*"The token issuers must always be able to reallocate vestings"*), permanently blocking token issuers from reallocating such vestings to new beneficiaries and preventing marketplace transfers to fresh accounts.

### Remediation

In `SecondSwap_StepVesting._createVesting`, guard the `releaseRate` calculation in the `if (_vestings[_beneficiary].totalAmount == 0)` branch against `_stepsClaimed >= numOfSteps`, matching the check performed in the `else` branch:

```solidity
        if (_vestings[_beneficiary].totalAmount == 0) {
            uint256 releaseRate = numOfSteps > _stepsClaimed
                ? _totalAmount / (numOfSteps - _stepsClaimed)
                : 0;
            _vestings[_beneficiary] = Vesting({
                stepsClaimed: _stepsClaimed,
                amountClaimed: 0,
                releaseRate: releaseRate,
                totalAmount: _totalAmount
            });
        } else {
            _vestings[_beneficiary].totalAmount += _totalAmount;
            if (numOfSteps > _vestings[_beneficiary].stepsClaimed) {
                _vestings[_beneficiary].releaseRate =
                    (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
                    (numOfSteps - _vestings[_beneficiary].stepsClaimed);
            } else {
                _vestings[_beneficiary].releaseRate = 0;
            }
        }
```

## Finding 11: Flawed `sellLimit` Accounting in `SecondSwap_VestingManager.listVesting` Allows Bypassing `maxSellPercent`

### Synopsis

In `SecondSwap_VestingManager.listVesting`, `sellLimit` is calculated by combining cumulative lifetime marketplace counters (`userAllocation.bought` and `userAllocation.sold`) with the user's current total vesting balance (`SecondSwap_Vesting(plan).total(seller)`), misclassifying purchased versus granted allocations after claims or off-market reallocations and allowing sellers to bypass `maxSellPercent`.

### Technical details

In `SecondSwap_VestingManager.sol`, `listVesting` enforces the maximum sellable allocation using the following calculation (lines 127–138):

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
```

This logic assumes that `currentAlloc + userAllocation.sold` always equals the user's lifetime total allocation, that all `userAllocation.bought` tokens are still currently held and unclaimed by `seller`, and that `currentAlloc + userAllocation.sold - userAllocation.bought` accurately represents the seller's granted (non-purchased) allocation.

This assumption breaks down in two ways:

1. **Historical `bought` credit is not reduced when purchased tokens are claimed (`total(seller)` vs. `available(seller)`):**
   `SecondSwap_StepVesting.total(seller)` returns `_vestings[seller].totalAmount`, which is not reduced when `seller` claims vested tokens via `SecondSwap_StepVesting.claim()`. Likewise, `userAllocation.bought` tracks the cumulative historical amount bought and is never reduced when purchased tokens vest and are claimed. When a user holds both granted and purchased allocations and claims vested tokens, the claimed portion of the purchased tokens is still counted in `sellLimit = userAllocation.bought + ...`. The seller can then apply the unused `bought` sale quota from already-claimed purchased tokens to list and sell their remaining unvested *granted* tokens well beyond `maxSellPercent`.

2. **Untracked off-market allocation changes via `transferVesting`:**
   Per the protocol design, `tokenIssuer` can reallocate vestings off-market via `SecondSwap_StepVesting.transferVesting(_grantor, _beneficiary, _amount)`, which updates `_vestings[user].totalAmount` without updating `allocations[user][plan]` in `SecondSwap_VestingManager`.
   - If a user purchases tokens (`userAllocation.bought > 0`) and those tokens (or part of their allocation) are later transferred out via `transferVesting`, `currentAlloc` decreases while `userAllocation.bought` remains unchanged. The formula deducts the off-market transfer from the inferred granted allocation `(currentAlloc + userAllocation.sold - userAllocation.bought)` first and keeps `sellLimit >= userAllocation.bought`. The user retains their historical `userAllocation.bought` quota credit and can reuse it to sell 100% of remaining or newly granted tokens, bypassing `maxSellPercent`.
   - Conversely, if a user lists `maxSellPercent` of their granted allocation (`userAllocation.sold` increases) and the remaining granted allocation is transferred off-market to another address, the recipient has `userAllocation.sold == 0` and can sell an additional `maxSellPercent` of the same granted tokens, while the original user retains an inflated `userAllocation.sold` counter that blocks future listings if tokens are reallocated back to them.

### Assumptions and preconditions

1. A vesting plan is deployed and marked `sellable` in `SecondSwap_VestingManager` with `maxSellPercent < BASE` (e.g., the default `2000` = 20%).
2. Either:
   - A user with a granted vesting allocation also purchases tokens on the marketplace and claims vested tokens once one or more vesting steps have elapsed; or
   - A token issuer reallocates vesting balances via `SecondSwap_StepVesting.transferVesting`.

### Proof of concept

#### Scenario 1: Bypassing `maxSellPercent` via claiming purchased tokens
1. `maxSellPercent` is set to `2000` (20%). Alice receives a granted vesting allocation of `1,000` tokens (`total = 1,000`, `bought = 0`, `sold = 0`). Alice is only supposed to be able to sell at most `200` of her granted tokens.
2. Alice purchases `600` tokens of the same vesting plan on the marketplace (`completePurchase`). Now `total(Alice) = 1,600`, `available(Alice) = 1,600`, `userAllocation.bought = 600`, `userAllocation.sold = 0`.
3. Half (50%) of the vesting steps elapse. Alice calls `SecondSwap_StepVesting.claim()`, claiming 50% of her `1,600` total allocation = `800` tokens (`500` from her granted allocation and `300` from her purchased allocation).
4. Alice now has `available(Alice) = 800` unvested tokens remaining (`500` unvested granted tokens and `300` unvested purchased tokens).
5. Alice calls `listVesting` for all `800` remaining tokens:
   - `available(Alice) >= 800` passes (`800 >= 800`).
   - `currentAlloc = total(Alice) = 1,600`.
   - `sellLimit = 600 + ((1,600 + 0 - 600) * 2000) / 10000 = 600 + 200 = 800`.
   - `userAllocation.sold` becomes `800 <= 800`, so the transaction succeeds.
6. Alice has sold all `500` of her remaining unvested granted tokens (50% of her granted allocation instead of the 20% cap) by reusing the `300` `bought` credit of the purchased tokens she already claimed.

#### Scenario 2: Reusing `bought` quota credit after off-market reallocation
1. Alice has `100` granted tokens (`maxSellPercent = 20%`, so at most `20` should be sellable) and lists `20` tokens (`sold = 20`, `currentAlloc = 80`).
2. Alice buys `500` tokens on the marketplace (`bought = 500`, `sold = 20`, `currentAlloc = 580`).
3. The token issuer transfers `500` tokens from Alice to Bob via `transferVesting(Alice, Bob, 500)`. Alice now holds only her `80` remaining granted tokens (`currentAlloc = 80`, `bought = 500`, `sold = 20`).
4. When Alice calls `listVesting` for `80` tokens, `currentAlloc + userAllocation.sold > userAllocation.bought` (`80 + 20 > 500`) is false, but `sellLimit` is initialized to `userAllocation.bought = 500`.
5. `userAllocation.sold` becomes `20 + 80 = 100 <= 500`, allowing Alice to sell 100% of her `100` granted allocation (and up to `400` more if granted additional tokens later).

### Impact assessment

- **Severity:** Medium / High
- **Technical Effect:** `sellLimit` in `SecondSwap_VestingManager.listVesting` overstates a seller's permissible sale quota whenever purchased tokens are claimed or transferred off-market, and fails to synchronize sale quotas across off-market `transferVesting` reallocations.
- **Business Impact:** Beneficiaries can bypass the token issuer's `maxSellPercent` restriction and liquidate up to 100% of their unvested granted token allocations on the secondary market, defeating a core control mechanism promised to token issuers.

### Remediation

Track granted and purchased allocations (along with their respective claimed and transferred portions) separately in `SecondSwap_StepVesting` and `SecondSwap_VestingManager`, or update `SecondSwap_VestingManager` hooks on both `claim` and off-market `transferVesting` so that `sellLimit` only credits currently available (unclaimed and untransferred) purchased tokens and accurately tracks the granted allocation:

```solidity
        Allocation storage userAllocation = allocations[seller][plan];

        uint256 availableAlloc = SecondSwap_Vesting(plan).available(seller);
        uint256 currentAlloc = SecondSwap_Vesting(plan).total(seller);
        uint256 claimedAlloc = currentAlloc - availableAlloc;

        uint256 effectiveBought = userAllocation.bought > claimedAlloc
            ? Math.min(availableAlloc, userAllocation.bought - claimedAlloc)
            : 0;

        uint256 sellLimit = effectiveBought;
        uint256 grantedBase = availableAlloc > effectiveBought
            ? (availableAlloc - effectiveBought) + claimedAlloc + userAllocation.sold
            : claimedAlloc + userAllocation.sold;

        sellLimit += (grantedBase * vestingSettings[plan].maxSellPercent) / BASE;

        userAllocation.sold += amount;

        require(userAllocation.sold <= sellLimit, "SS_VestingManager: cannot list more than max sell percent");
        SecondSwap_Vesting(plan).transferVesting(seller, address(this), amount);
```

## Finding 12: Broken Vesting Custody Transfer and Pooled Escrow Accounting in `SecondSwap_VestingManager.listVesting`

### Synopsis

In `SecondSwap_VestingManager.listVesting`, transferring listed allocations to `address(this)` via `SecondSwap_Vesting(plan).transferVesting(seller, address(this), amount)` corrupts per-user claim history, distorts the seller's `releaseRate`, merges heterogeneous seller schedules into a single pooled escrow record, and reverts with division by zero when `stepsClaimed == numOfSteps`.

### Technical details

At line 139 of `SecondSwap_VestingManager.sol`, `listVesting` escrows the seller's listed vesting allocation by transferring it to the `SecondSwap_VestingManager` contract (`address(this)`):

```solidity
SecondSwap_Vesting(plan).transferVesting(seller, address(this), amount);
```

In `SecondSwap_StepVesting.transferVesting` (lines 229–232) and `_createVesting` (lines 288–304), vesting state is tracked per address in `_vestings[beneficiary]` using a single `Vesting` struct (`stepsClaimed`, `amountClaimed`, `releaseRate`, `totalAmount`). Using `transferVesting(seller, address(this), amount)` to move tokens into and out of the `SecondSwap_VestingManager` pooled escrow introduces critical accounting flaws:

1. **Distortion of Seller `releaseRate` and Over-Claiming / Underflow:**
   When `transferVesting(seller, address(this), amount)` executes, the seller's `totalAmount` is decremented by `amount` and their `releaseRate` is recalculated as:
   ```solidity
   grantorVesting.totalAmount -= _amount;
   grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;
   ```
   Because `grantorVesting.totalAmount` still includes `grantorVesting.amountClaimed`, dividing the entire `grantorVesting.totalAmount` by `numOfSteps` (rather than dividing the remaining unclaimed entitlement `grantorVesting.totalAmount - grantorVesting.amountClaimed` by the remaining steps `numOfSteps - grantorVesting.stepsClaimed`) inflates the seller's `releaseRate` whenever `seller` has previously claimed tokens (`stepsClaimed > 0`). In subsequent steps before `numOfSteps`, the seller claims at this inflated `releaseRate`, allowing `amountClaimed` to exceed `totalAmount` (draining tokens allocated to other users) and causing `available(seller)` and the final step's `claimable(seller)` (`vesting.totalAmount - vesting.amountClaimed`) to revert with arithmetic underflow.

2. **Cross-User Claim-History Interference in Pooled Escrow (`address(this)`):**
   All active listings for a `plan` are pooled into a single `_vestings[address(this)]` struct in `SecondSwap_StepVesting`.
   - When the first seller lists (`_vestings[address(this)].totalAmount == 0`), `_vestings[address(this)].stepsClaimed` is initialized to that seller's `stepsClaimed`.
   - When subsequent sellers list while `_vestings[address(this)].totalAmount > 0`, `_createVesting` enters the `else` branch, adding `amount` to `_vestings[address(this)].totalAmount` while ignoring the new seller's `stepsClaimed`.
   - When a listing is later purchased (`completePurchase`) or unlisted (`unlistVesting`), `transferVesting(address(this), recipient, amount)` passes `_vestings[address(this)].stepsClaimed` to `_createVesting`. Any recipient with `totalAmount == 0` inherits the pooled escrow's `stepsClaimed` instead of the listing seller's `stepsClaimed`. If the escrow's `stepsClaimed` is lower than the seller's `stepsClaimed`, a fresh buyer account can immediately claim already-claimed steps ahead of schedule; if higher, a buyer loses immediate claimability for already-elapsed steps.

3. **Division-by-Zero Revert When `stepsClaimed == numOfSteps`:**
   If a user claims all `numOfSteps` at or after `endTime` (`stepsClaimed == numOfSteps`) and subsequently acquires additional vesting allocation (e.g., via `completePurchase` or `transferVesting` while their `totalAmount > 0`), their `stepsClaimed` remains equal to `numOfSteps` while `available(seller) > 0`. When this seller calls `listVesting` while `_vestings[address(this)].totalAmount == 0`, `_createVesting(address(this), amount, numOfSteps, true)` evaluates `_totalAmount / (numOfSteps - _stepsClaimed)` with `numOfSteps - _stepsClaimed == 0`, reverting with a division-by-zero panic (`Panic(0x12)`).

### Assumptions and preconditions

- A vesting plan (`SecondSwap_StepVesting`) is deployed and marked `sellable = true` in `SecondSwap_VestingManager`.
- At least one vesting step has elapsed (`block.timestamp >= startTime + stepDuration`), allowing beneficiaries to have nonzero `stepsClaimed` and `amountClaimed`.

### Proof of concept

#### Scenario 1: Seller Over-Claiming and Arithmetic Underflow After Listing
1. Suppose `numOfSteps = 10` and `seller` is allocated `totalAmount = 1,000` tokens (`releaseRate = 100` per step).
2. At step 5, `seller` calls `claim()`, receiving `500` tokens (`stepsClaimed = 5`, `amountClaimed = 500`, `available = 500`).
3. At step 5, `seller` lists `200` tokens via `Marketplace.listVesting` -> `VestingManager.listVesting(seller, plan, 200)`.
4. `SecondSwap_Vesting(plan).transferVesting(seller, address(this), 200)` sets:
   - `seller.totalAmount = 1,000 - 200 = 800`
   - `seller.releaseRate = 800 / 10 = 80` (instead of `(800 - 500) / (10 - 5) = 60`).
5. At step 9 (4 steps later), `seller` calls `claim()`. Because `stepsClaimed + claimableSteps = 5 + 4 = 9 < 10`, `claimableAmount = 80 * 4 = 320`.
6. `seller` receives `320` tokens, bringing `seller.amountClaimed` to `500 + 320 = 820 > seller.totalAmount (800)`. The seller has claimed `20` more tokens than their remaining allocation (while the `200` listed tokens can still be bought and claimed in full by a buyer, resulting in `1,020` total tokens claimed from a `1,000` token allocation).
7. At step 10, `claimable(seller)` evaluates `vesting.totalAmount - vesting.amountClaimed` (`800 - 820`), reverting with arithmetic underflow.

#### Scenario 2: Cross-User `stepsClaimed` Contamination in Pooled Escrow
1. At step 5 (of 10), `seller1` (who has never called `claim()`, so `seller1.stepsClaimed = 0`) lists `1` token. `_vestings[vestingManager].stepsClaimed` is set to `0` and `_vestings[vestingManager].totalAmount = 1`.
2. At step 5, `seller2` calls `claim()` (`seller2.stepsClaimed = 5`, `seller2.amountClaimed = 500`) and then lists `200` tokens. Because `_vestings[vestingManager].totalAmount > 0`, `_vestings[vestingManager].stepsClaimed` remains `0`.
3. A fresh `buyer` account (`_vestings[buyer].totalAmount == 0`) purchases `seller2`'s `200` tokens via `spotPurchase`.
4. `completePurchase` calls `transferVesting(vestingManager, buyer, 200)`, which initializes `_vestings[buyer]` with `stepsClaimed = _vestings[vestingManager].stepsClaimed = 0` and `releaseRate = 200 / 10 = 20`.
5. `buyer` immediately calls `claim()` at step 5 and claims `5 * 20 = 100` tokens that were only supposed to vest over steps 6–10.

### Impact assessment

- **Severity:** High
- **Technical Impact:**
  - Violation of token conservation: sellers who list after claiming can over-claim tokens from `SecondSwap_StepVesting`, insolvencing the vesting contract for final claimants.
  - Permanent underflow DoS on `available(seller)` and `claimable(seller)` once `amountClaimed > totalAmount`, or division-by-zero DoS in `listVesting` when `stepsClaimed == numOfSteps`.
  - Distortion of buyer/seller vesting schedules due to shared `stepsClaimed` state in `SecondSwap_VestingManager`'s pooled escrow.

### Remediation

Avoid pooling heterogeneous vesting schedules into a single `_vestings[address(this)]` beneficiary record and ensure `releaseRate` and `stepsClaimed` preserve each beneficiary's unclaimed entitlement across transfers. In `SecondSwap_VestingManager.listVesting`, ensure the underlying vesting contract tracks escrowed allocations per listing or normalizes claimed steps before transferring custody:

```solidity
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

Specifically, in the `transferVesting` implementation invoked by `SecondSwap_VestingManager`:
1. Update the grantor's `releaseRate` using remaining unclaimed tokens and remaining steps:
   ```solidity
   grantorVesting.totalAmount -= _amount;
   uint256 remainingSteps = numOfSteps - grantorVesting.stepsClaimed;
   grantorVesting.releaseRate = remainingSteps > 0
       ? (grantorVesting.totalAmount - grantorVesting.amountClaimed) / remainingSteps
       : 0;
   ```
2. Guard against `numOfSteps - _stepsClaimed == 0` when initializing a beneficiary's `Vesting` struct:
   ```solidity
   uint256 remainingSteps = numOfSteps - _stepsClaimed;
   _vestings[_beneficiary] = Vesting({
       stepsClaimed: _stepsClaimed,
       amountClaimed: 0,
       releaseRate: remainingSteps > 0 ? _totalAmount / remainingSteps : 0,
       totalAmount: _totalAmount
   });
   ```
3. Track escrowed vesting balances per listing (or transfer directly between `seller` and `buyer` upon purchase while locking `listedAmount` in `SecondSwap_VestingManager`) so that `address(this)` does not overwrite or conflate `stepsClaimed` across different sellers.

## Finding 13: Unreconciled Purchase Credits in `SecondSwap_VestingManager.completePurchase` Distort Resale Limits and Bypass `maxSellPercent`

### Synopsis

In `SecondSwap_VestingManager.completePurchase`, `allocations[buyer][vesting].bought` permanently accumulates purchased amounts as unconditional resale credits without reconciling against claimed or reallocated vesting entitlements. Across the allocation lifecycle, this divergence inflates `sellLimit` in `listVesting`, allowing beneficiaries to bypass `maxSellPercent` and sell up to 100% of their locked original allocations.

### Technical details

When a purchase is finalized on the marketplace, `SecondSwap_VestingManager.completePurchase` (lines 161–164) records the purchased amount in `allocations[buyer][vesting].bought`:

```solidity
function completePurchase(address buyer, address vesting, uint256 amount) external onlyMarketplace {
    allocations[buyer][vesting].bought += amount;
    SecondSwap_Vesting(vesting).transferVesting(address(this), buyer, amount);
}
```

In `SecondSwap_VestingManager.listVesting` (lines 125–138), `allocations[seller][plan].bought` is used as a 100%-resellable base allowance (`uint256 sellLimit = userAllocation.bought`) on top of the `maxSellPercent` cap applied to the seller's original allocation:

```solidity
Allocation storage userAllocation = allocations[seller][plan];

uint256 sellLimit = userAllocation.bought;
uint256 currentAlloc = SecondSwap_Vesting(plan).total(seller);

if (currentAlloc + userAllocation.sold > userAllocation.bought) {
    sellLimit +=
        ((currentAlloc + userAllocation.sold - userAllocation.bought) * vestingSettings[plan].maxSellPercent) /
        BASE;
}

userAllocation.sold += amount;

require(userAllocation.sold <= sellLimit, "SS_VestingManager: cannot list more than max sell percent");
```

This accounting model assumes that every token added to `allocations[buyer][vesting].bought` in `completePurchase` remains permanently backed 1-to-1 by resellable purchased tokens in the user's vesting balance, and that any remaining balance belongs to the user's restricted original allocation. However, `allocations[buyer][vesting].bought` is a monotonically increasing counter that is never reduced or capped by the user's actual remaining purchased entitlement across the allocation lifecycle:

1. **Divergence after token claims (`claim()`):** When a beneficiary claims vested tokens in `SecondSwap_StepVesting.claim()`, `_vestings[user].amountClaimed` increases while `_vestings[user].totalAmount` (`SecondSwap_Vesting(plan).total(user)`) and `allocations[user][vesting].bought` remain unchanged. Because `completePurchase` merges purchased tokens into the single `_vestings[buyer]` balance while adding permanent resale credits to `allocations[buyer][vesting].bought`, claiming `X` vested tokens leaves the full `userAllocation.bought` credit intact. The user can then apply that `userAllocation.bought` credit to list and sell `X` additional tokens from their unvested original allocation—exceeding `maxSellPercent`.
2. **Divergence after entitlement transfers (`transferVesting()`):** When a token issuer reallocates `X` tokens out of a user's vesting balance via `SecondSwap_StepVesting.transferVesting(user, recipient, X)`, `_vestings[user].totalAmount` decreases by `X`, but the purchase credit recorded in `completePurchase` (`allocations[user][vesting].bought`) is not reduced. When the user subsequently calls `listVesting`, `sellLimit` still starts at the full `userAllocation.bought` even when `currentAlloc + userAllocation.sold <= userAllocation.bought`, enabling the user to sell their remaining restricted original allocation up to `userAllocation.bought` in full violation of `maxSellPercent`.

### Assumptions and preconditions

1. A vesting plan is deployed with `sellable = true` and a restrictive `maxSellPercent` (e.g., the default `2000` basis points = 20%).
2. A beneficiary holds an original vesting allocation subject to `maxSellPercent` and also purchases vesting tokens on the marketplace (triggering `completePurchase`).
3. Either vested tokens are claimed by the beneficiary via `SecondSwap_StepVesting.claim()`, or tokens are reallocated out of the beneficiary's schedule by the token issuer via `SecondSwap_StepVesting.transferVesting()`.

### Proof of concept

#### Scenario 1: Bypassing `maxSellPercent` via Purchase and Claim

1. **Initial State:** Alice receives an initial vesting allocation of `1,000` tokens (`_vestings[Alice].totalAmount = 1000`, `amountClaimed = 0`). The plan has `maxSellPercent = 2000` (20%), meaning Alice is only permitted to sell at most `200` tokens on the marketplace and must keep at least `800` tokens subject to the vesting schedule.
2. **Marketplace Purchase:** Alice buys `800` vested tokens on the marketplace. `SecondSwap_VestingManager.completePurchase(Alice, vesting, 800)` executes:
   - `allocations[Alice][vesting].bought` becomes `800`.
   - `_vestings[Alice].totalAmount` becomes `1000 + 800 = 1800`.
3. **Claiming Vested Tokens:** Once `800` tokens have vested across Alice's combined schedule, Alice calls `vesting.claim()` and receives `800` ERC20 tokens:
   - `_vestings[Alice].amountClaimed` becomes `800`.
   - `_vestings[Alice].totalAmount` (`total(Alice)`) remains `1800`.
   - `SecondSwap_Vesting(vesting).available(Alice)` is `1800 - 800 = 1000` (the exact size of her original locked allocation).
4. **Resale Limit Bypass:** Alice calls `listVesting` for all `1,000` remaining unvested tokens:
   - `available(Alice)` is `1000 >= 1000` (passes).
   - `sellLimit = userAllocation.bought (800) + ((1800 + 0 - 800) * 2000) / 10000 = 800 + 200 = 1000`.
   - `userAllocation.sold` becomes `1000 <= 1000` (passes).
   - Alice successfully sells all `1,000` of her remaining unvested tokens (`100%` of her original allocation instead of the `20%` maximum).

#### Scenario 2: Bypassing `maxSellPercent` after Entitlement Reallocation

1. **Initial State:** Alice holds `100` original vesting tokens (`maxSellPercent = 2000` = 20%) and lists her maximum allowed `20` tokens (`allocations[Alice][vesting].sold = 20`, `currentAlloc = 80`). Her remaining `80` original tokens are locked from being listed.
2. **Marketplace Purchase:** Alice purchases `500` tokens on the marketplace. `completePurchase(Alice, vesting, 500)` sets `allocations[Alice][vesting].bought = 500` and increases `currentAlloc` to `580`.
3. **Reallocation:** The token issuer transfers `500` tokens from Alice to Bob via `SecondSwap_StepVesting.transferVesting(Alice, Bob, 500)`. Alice's `currentAlloc` returns to `80` (her locked original tokens), but `allocations[Alice][vesting].bought` remains `500`.
4. **Resale Limit Bypass:** Alice calls `listVesting` for her remaining `80` locked original tokens:
   - `sellLimit` is initialized to `userAllocation.bought = 500`.
   - `currentAlloc + userAllocation.sold` (`80 + 20 = 100`) is less than `userAllocation.bought` (`500`), so `sellLimit` remains `500`.
   - `userAllocation.sold` becomes `20 + 80 = 100 <= 500`, allowing Alice to list and sell the remaining `80%` of her original allocation.

### Impact assessment

- **Severity:** Medium / High
- **Technical Effect:** `allocations[buyer][vesting].bought` diverges from the buyer's actual held/unclaimed purchased tokens, inflating `sellLimit` in `listVesting`.
- **Business Impact:** Core protocol invariant violation ("Is the maximum sellable allocation enforced correctly?"). Beneficiaries who are supposed to be restricted to selling at most `maxSellPercent` (e.g., 20%) of their original token grant can liquidate up to 100% of their unvested original allocation on the secondary marketplace.

### Remediation

Reconcile `allocations[buyer][vesting].bought` with the buyer's actual unclaimed and held vesting balance before crediting new purchases (and when evaluating resale limits), so that claimed or transferred-out tokens cannot remain as phantom resale credits in `allocations[buyer][vesting].bought`:

```solidity
    function completePurchase(address buyer, address vesting, uint256 amount) external onlyMarketplace {
        Allocation storage buyerAllocation = allocations[buyer][vesting];
        uint256 availableBefore = SecondSwap_Vesting(vesting).available(buyer);
        if (buyerAllocation.bought > availableBefore + buyerAllocation.sold) {
            buyerAllocation.bought = availableBefore + buyerAllocation.sold;
        }
        buyerAllocation.bought += amount;
        SecondSwap_Vesting(vesting).transferVesting(address(this), buyer, amount);
    }
```

## Finding 14: Missing `sellable` Check in `SecondSwap_VestingManager.completePurchase` Allows Purchases of Disabled Vesting Plans

### Synopsis

`SecondSwap_VestingManager.completePurchase` does not verify that `vestingSettings[vesting].sellable` is `true` before settling a marketplace trade. Consequently, when `s2Admin` disables a vesting plan via `setSellable(vesting, false)`, existing active listings in `SecondSwap_Marketplace` can still be purchased via `spotPurchase`, bypassing the trading restriction.

### Technical details

In `SecondSwap_VestingManager`, each vesting plan's trading eligibility is controlled by `vestingSettings[plan].sellable`, which can be toggled by `s2Admin` through `setSellable(address vesting, bool sellable)` (lines 173–186).

When a seller creates a new listing via `SecondSwap_Marketplace.listVesting`, the marketplace delegates validation and custody transfer to `SecondSwap_VestingManager.listVesting` (lines 121–140), which enforces trading eligibility at line 122:

```solidity
require(vestingSettings[plan].sellable, "vesting not sellable");
```

However, when a buyer executes a purchase on an existing listing via `SecondSwap_Marketplace.spotPurchase` (lines 505–551), neither `spotPurchase` nor `SecondSwap_VestingManager.completePurchase` (lines 161–164) checks `vestingSettings[vesting].sellable`:

```solidity
function completePurchase(address buyer, address vesting, uint256 amount) external onlyMarketplace {
    allocations[buyer][vesting].bought += amount;
    SecondSwap_Vesting(vesting).transferVesting(address(this), buyer, amount);
}
```

Because `setSellable(vesting, false)` only flips the boolean flag in `vestingSettings[vesting]` without cancelling pre-existing listings in `SecondSwap_Marketplace`, any listing created while `sellable` was `true` remains in `Status.LIST`. Buyers can continue calling `spotPurchase` on those listings, and `completePurchase` will credit `allocations[buyer][vesting].bought` and transfer the vested tokens from `SecondSwap_VestingManager` to the buyer even though the vesting plan is no longer sellable.

### Assumptions and preconditions

1. A vesting plan has been deployed and enabled for marketplace trading (`vestingSettings[vesting].sellable == true`).
2. At least one seller has created an active listing (`Status.LIST`) for this vesting plan via `SecondSwap_Marketplace.listVesting`.
3. `s2Admin` subsequently disables trading for the vesting plan by calling `SecondSwap_VestingManager.setSellable(vesting, false)`.

### Proof of concept

1. A vesting plan `vestingPlan` is deployed with `vestingSettings[vestingPlan].sellable == true`.
2. Seller Alice calls `SecondSwap_Marketplace.listVesting` to list `100e18` vested tokens from `vestingPlan` as `listingId = 0`. `SecondSwap_VestingManager.listVesting` succeeds because `sellable` is `true`, and `100e18` vesting allocation is transferred to `SecondSwap_VestingManager`.
3. `s2Admin` calls `SecondSwap_VestingManager.setSellable(vestingPlan, false)` to halt secondary trading of `vestingPlan`.
4. Buyer Bob calls `SecondSwap_Marketplace.spotPurchase(vestingPlan, 0, 100e18, address(0))`.
5. `spotPurchase` processes payment transfers and calls `SecondSwap_VestingManager.completePurchase(bob, vestingPlan, 100e18)`.
6. Because `completePurchase` does not check `vestingSettings[vestingPlan].sellable`, the call succeeds and transfers the `100e18` vesting allocation to Bob despite `vestingPlan` being disabled for sale.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** Vesting allocations in a disabled vesting plan can still be bought and transferred through pre-existing marketplace listings after `s2Admin` sets `sellable` to `false`.
- **Business Implications:** Violates the protocol's administrative control invariant whereby `s2Admin` can disable marketplace trading for specific vesting plans without pausing the entire marketplace for all other tokens.

### Remediation

Enforce the `sellable` check in `SecondSwap_VestingManager.completePurchase` before updating allocations and transferring the vesting schedule.

```solidity
    function completePurchase(address buyer, address vesting, uint256 amount) external onlyMarketplace {
        require(vestingSettings[vesting].sellable, "vesting not sellable");
        allocations[buyer][vesting].bought += amount;
        SecondSwap_Vesting(vesting).transferVesting(address(this), buyer, amount);
    }
```
