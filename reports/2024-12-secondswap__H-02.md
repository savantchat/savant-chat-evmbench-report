# Audit report — 2024-12-secondswap/H-02

## Finding 1: Incorrect `releaseRate` Calculation in `SecondSwap_StepVesting.transferVesting` Allows Grantor to Overclaim Vested Tokens

### Synopsis

In `SecondSwap_StepVesting.transferVesting`, the grantor's `releaseRate` is recalculated as `grantorVesting.totalAmount / numOfSteps` instead of accounting for `amountClaimed` and `stepsClaimed`. After claiming partially vested tokens and transferring the remaining allocation, the grantor retains an inflated `releaseRate` and can overclaim tokens, draining the vesting contract.

### Technical details

In `SecondSwap_StepVesting`, each beneficiary's vesting schedule is tracked by the `Vesting` struct (`stepsClaimed`, `amountClaimed`, `releaseRate`, and `totalAmount`).

When `claim()` is called during the vesting period (before `numOfSteps` is reached), `claimable()` calculates the claimable tokens using `vesting.releaseRate * claimableSteps` (line 184):

```solidity
if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
    //[BUG FIX] user can buy more than they are allocated
    claimableAmount = vesting.totalAmount - vesting.amountClaimed;
    return (claimableAmount, claimableSteps);
}

claimableAmount = vesting.releaseRate * claimableSteps;
return (claimableAmount, claimableSteps);
```

Notice that for any step prior to the final step (`vesting.stepsClaimed + claimableSteps < numOfSteps`), `claimable()` relies solely on `vesting.releaseRate * claimableSteps` and does not cap `claimableAmount` at `vesting.totalAmount - vesting.amountClaimed`. Consequently, `vesting.releaseRate` must strictly equal the remaining unclaimed obligation divided by the remaining unclaimed steps:

$$\text{releaseRate} = \frac{\text{totalAmount} - \text{amountClaimed}}{\text{numOfSteps} - \text{stepsClaimed}}$$

While `_createVesting` (lines 297–303) applies this formula when updating an existing beneficiary's vesting schedule, `transferVesting` (lines 229–230) updates the grantor's schedule as follows:

```solidity
grantorVesting.totalAmount -= _amount;
grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;
```

This calculation is flawed in two ways whenever `grantorVesting.stepsClaimed > 0`:
1. `grantorVesting.totalAmount` still includes `grantorVesting.amountClaimed` (tokens the grantor has already claimed prior to the transfer).
2. The denominator is `numOfSteps` rather than the remaining steps `(numOfSteps - grantorVesting.stepsClaimed)`.

Specifically, if a grantor claims their vested tokens at step $s > 0$ (so `amountClaimed > 0`) and then transfers all of their remaining unclaimed allocation (`_amount = grantorVesting.totalAmount - grantorVesting.amountClaimed`), `grantorVesting.totalAmount` becomes equal to `grantorVesting.amountClaimed`. Although the grantor has zero unclaimed tokens remaining, `grantorVesting.releaseRate` is set to `grantorVesting.amountClaimed / numOfSteps > 0` instead of `0`. In subsequent steps prior to `numOfSteps`, the grantor can continue calling `claim()` to extract additional tokens from the contract, pushing `grantorVesting.amountClaimed` above `grantorVesting.totalAmount` and causing `available(_grantor)` and final-step `claimable(_grantor)` calls to revert with an arithmetic underflow.

### Assumptions and preconditions

1. A vesting plan is deployed with `numOfSteps > 2` and a beneficiary (`_grantor`) has a non-zero vesting allocation (either initially granted or purchased via the marketplace).
2. At least one vesting step has elapsed (`currentStep >= 1` and `currentStep < numOfSteps - 1`), and `_grantor` calls `claim()` so that `grantorVesting.stepsClaimed > 0` and `grantorVesting.amountClaimed > 0`.
3. `_grantor`'s vesting allocation is transferred via `transferVesting` (either directly via `SecondSwap_VestingDeployer.transferVesting` or by `_grantor` listing their available vesting on the marketplace via `SecondSwap_VestingManager.listVesting`).

### Proof of concept

Consider a `SecondSwap_StepVesting` contract configured with `numOfSteps = 100`:

1. Alice has a vesting allocation with `totalAmount = 1,000e18`, `stepsClaimed = 0`, `amountClaimed = 0`, and `releaseRate = 10e18` per step.
2. At step 50 (`currentStep = 50`), Alice calls `claim()`:
   - `claimable(Alice)` returns `(500e18, 50)`.
   - Alice receives `500e18` tokens.
   - Alice's state is now `totalAmount = 1,000e18`, `amountClaimed = 500e18`, `stepsClaimed = 50`. Her remaining unclaimed balance (`available(Alice)`) is `500e18`.
3. At step 50, `transferVesting(Alice, Bob, 500e18)` is executed (e.g., via marketplace listing or token issuer reallocation):
   - The check `grantorVesting.totalAmount - grantorVesting.amountClaimed >= 500e18` (`1,000e18 - 500e18 >= 500e18`) passes.
   - `grantorVesting.totalAmount` is updated to `1,000e18 - 500e18 = 500e18`.
   - Line 230 sets `grantorVesting.releaseRate = 500e18 / 100 = 5e18` (instead of `(500e18 - 500e18) / (100 - 50) = 0`).
   - Bob receives a vesting schedule for `500e18` tokens over the remaining 50 steps (`releaseRate = 10e18`).
4. Time advances to step 99 (`currentStep = 99`). Alice calls `claim()`:
   - `claimableSteps = 99 - 50 = 49`.
   - Since `vesting.stepsClaimed + claimableSteps = 50 + 49 = 99 < 100`, `claimable(Alice)` executes line 184 and returns `claimableAmount = 5e18 * 49 = 245e18`.
   - `claim()` transfers an extra `245e18` tokens to Alice, updating `grantorVesting.amountClaimed` to `745e18` (exceeding her `totalAmount` of `500e18`).
5. At step 100, Bob claims his full `500e18` tokens. Between Alice (`745e18`) and Bob (`500e18`), `1,245e18` tokens have been claimed from an initial `1,000e18` allocation, stealing `245e18` tokens from other beneficiaries in the contract.

### Impact assessment

- **Severity:** High
- **Direct Technical Impact:** A grantor who has claimed tokens in earlier steps and subsequently transfers part or all of their remaining vesting allocation retains an inflated `releaseRate`. Before the final vesting step, the grantor can claim tokens exceeding their `totalAmount`, stealing tokens held by `SecondSwap_StepVesting` for other beneficiaries. Additionally, once `amountClaimed > totalAmount`, view calls to `available(_grantor)` and `claimable(_grantor)` (at `numOfSteps`) revert due to arithmetic underflow.
- **Business Impact:** Core protocol invariant violation ("All tokens locked in a vesting plan must be allocated and claimable"). Because `SecondSwap_StepVesting` pools tokens for all beneficiaries in a single contract, overclaiming by one user directly causes insolvency for the remaining beneficiaries whose final claims will revert due to insufficient token balance.

### Remediation

Update `SecondSwap_StepVesting.transferVesting` to calculate `grantorVesting.releaseRate` using the remaining unclaimed tokens (`grantorVesting.totalAmount - grantorVesting.amountClaimed`) divided by the remaining unclaimed steps (`numOfSteps - grantorVesting.stepsClaimed`), matching the logic in `_createVesting`:

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

## Finding 2: Integer Division Truncation in `SecondSwap_StepVesting` `stepDuration` Calculation Causes Accelerated Vesting and Premature Token Release

### Synopsis

In `SecondSwap_StepVesting`, `stepDuration` is stored as an integer calculated via floor division `(_endTime - _startTime) / _numOfSteps`. When the total vesting duration is not evenly divisible by `_numOfSteps`, truncation reduces `stepDuration`, causing all vesting steps to complete and tokens to become fully claimable significantly before `endTime`.

### Technical details

The `SecondSwap_StepVesting` contract defines the vesting schedule parameters using `startTime`, `endTime`, `numOfSteps`, and `stepDuration` (lines 61–76). In the constructor (line 133), `stepDuration` is computed using integer division:

```solidity
stepDuration = (_endTime - _startTime) / _numOfSteps;
```

In `claimable(address _beneficiary)` (lines 167–185), the contract determines how many steps have elapsed by dividing `elapsedTime` by the truncated `stepDuration`:

```solidity
uint256 currentTime = Math.min(block.timestamp, endTime);
...
uint256 elapsedTime = currentTime - startTime;
uint256 currentStep = elapsedTime / stepDuration;
uint256 claimableSteps = currentStep - vesting.stepsClaimed;
```

Because Solidity integer division truncates towards zero, any remainder `R = (_endTime - _startTime) % _numOfSteps` is lost per step. This creates two distinct failure modes:

1. **Accelerated Vesting Schedule & Premature Unlock:**
   When `(_endTime - _startTime)` is not a multiple of `_numOfSteps`, `stepDuration` is smaller than the true duration of a step. Specifically, in the extreme case where `_endTime - _startTime = 2 * _numOfSteps - 1`, `stepDuration` evaluates to `1` second instead of nearly `2` seconds, cutting the effective vesting duration nearly in half. All `numOfSteps` steps will be reached at `startTime + numOfSteps * stepDuration = endTime - R`. At that point, `vesting.stepsClaimed + claimableSteps >= numOfSteps` (line 178) triggers early, allowing beneficiaries to claim 100% of their `totalAmount` well before `endTime`. Furthermore, if `currentTime` reaches `endTime` while `stepDuration` was truncated, `currentStep = (endTime - startTime) / stepDuration` can actually exceed `numOfSteps`.

2. **Division by Zero DoS:**
   If `_numOfSteps > (_endTime - _startTime)`, `stepDuration` evaluates to `0`. Consequently, every call to `claimable()` or `claim()` after `startTime` will revert due to a division by zero at `elapsedTime / stepDuration` (line 173), permanently locking all tokens in the vesting contract.

Instead of pre-calculating a truncated `stepDuration`, the number of elapsed steps should be calculated directly using `elapsedTime * numOfSteps / (endTime - startTime)` to preserve full precision across the entire schedule duration.

### Assumptions and preconditions

- A `SecondSwap_StepVesting` contract is deployed with valid parameters where `(_endTime - _startTime)` is not evenly divisible by `_numOfSteps` (or where `_numOfSteps > _endTime - _startTime`).
- Beneficiaries have active vesting allocations created via `createVesting` or `transferVesting`.

### Proof of concept

Consider a vesting plan deployed with the following parameters:
- `startTime = 1000`
- `endTime = 2999` (total intended duration = `1999` seconds)
- `numOfSteps = 1000`
- Alice is allocated `1000e18` tokens.

1. **Deployment:**
   The constructor computes:
   `stepDuration = (2999 - 1000) / 1000 = 1999 / 1000 = 1` second.

2. **Premature Full Claim:**
   At `block.timestamp = 2000` (which is only ~50% through the intended 1999-second vesting period, 999 seconds before `endTime`):
   - `currentTime = Math.min(2000, 2999) = 2000`
   - `elapsedTime = 2000 - 1000 = 1000`
   - `currentStep = 1000 / 1 = 1000`
   - `claimableSteps = 1000 - 0 = 1000`
   - The check `vesting.stepsClaimed + claimableSteps >= numOfSteps` (`1000 >= 1000`) evaluates to `true`.
   - `claimableAmount = vesting.totalAmount - vesting.amountClaimed = 1000e18`.

3. **Result:**
   Alice calls `claim()` at `t = 2000` and receives 100% of her vested tokens (`1000e18`), even though roughly half of the vesting duration remains until `endTime = 2999`.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** Rounding truncation in `stepDuration` distorts the step release schedule, causing each step to elapse faster than intended and allowing beneficiaries to claim 100% of their locked tokens prematurely (up to nearly 2x faster than the configured `endTime - startTime` duration), or causing a permanent DoS if `stepDuration == 0`.
- **Business Implications:** Violates the core vesting schedule timing guarantees relied upon by token issuers and secondary market buyers, releasing locked tokens into circulation ahead of the scheduled unlock timeline.

### Remediation

Remove `stepDuration` (or avoid using it for step calculation) and compute `currentStep` directly from the total vesting duration `endTime - startTime` using multiplication before division in `claimable()`:

```solidity
    function claimable(address _beneficiary) public view returns (uint256, uint256) {
        Vesting memory vesting = _vestings[_beneficiary];
        if (vesting.totalAmount == 0) {
            return (0, 0);
        }

        uint256 currentTime = Math.min(block.timestamp, endTime);
        if (currentTime < startTime) {
            return (0, 0);
        }

        uint256 elapsedTime = currentTime - startTime;
        uint256 totalDuration = endTime - startTime;
        uint256 currentStep = (elapsedTime * numOfSteps) / totalDuration;
        uint256 claimableSteps = currentStep - vesting.stepsClaimed;

        uint256 claimableAmount;

        if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
            claimableAmount = vesting.totalAmount - vesting.amountClaimed;
            return (claimableAmount, claimableSteps);
        }

        claimableAmount = vesting.releaseRate * claimableSteps;
        return (claimableAmount, claimableSteps);
    }
```

## Finding 3: Unchecked Token Balance Delta in `SecondSwap_StepVesting.createVesting` Causes Allocation-to-Collateral Insolvency for Fee-on-Transfer Tokens

### Synopsis

When `createVesting` or `createVestings` funds a new or existing vesting schedule via `_createVesting`, the contract credits the full nominal `_totalAmount` to the beneficiary's allocation without measuring the actual token balance received from `safeTransferFrom`, causing contract insolvency if the vested ERC20 token charges a transfer fee.

### Technical details

In `SecondSwap_StepVesting`, the external entrypoints `createVesting` (lines 244–250) and `createVestings` (lines 260–267) invoke `_createVesting(_beneficiary, _totalAmount, 0, false)` to allocate vested tokens and pull the backing collateral from `tokenIssuer`.

Inside `_createVesting` (lines 279–310):
1. The beneficiary's vesting record `_vestings[_beneficiary].totalAmount` is increased by the raw `_totalAmount` argument (lines 293 and 296), and `_vestings[_beneficiary].releaseRate` is computed directly from this nominal `_totalAmount` (lines 292 and 298–300).
2. Because `_isInternal` is `false` for `createVesting` and `createVestings`, line 307 executes:
   ```solidity
   token.safeTransferFrom(tokenIssuer, address(this), _totalAmount);
   ```

The contract never measures its `token.balanceOf(address(this))` before and after the `safeTransferFrom` call to determine how many tokens were actually received. For ERC20 tokens with nonstandard transfer semantics—such as fee-on-transfer or deflationary tokens—the actual collateral deposited into `SecondSwap_StepVesting` is strictly less than `_totalAmount`.

Consequently, the sum of beneficiaries' allocated `totalAmount` values exceeds the actual token balance held by the vesting contract. This directly violates the core protocol invariant stated in the documentation: *"All tokens locked in a vesting plan must be allocated and claimable."* As beneficiaries call `claim()` (or trade their allocations on the marketplace via `transferVesting`), early claimants withdraw at the unadjusted `releaseRate` and `totalAmount`, depleting the contract's token reserves and causing subsequent `claim()` calls by remaining beneficiaries to revert due to insufficient token balance.

### Assumptions and preconditions

- A `SecondSwap_StepVesting` contract is deployed for an ERC20 token that deducts a fee on `transferFrom` (fee-on-transfer token).
- An authorized caller (`tokenIssuer`, `manager`, or `vestingDeployer`) calls `createVesting` or `createVestings` to fund one or more vesting schedules.

### Proof of concept

1. Assume `token` is an ERC20 token that charges a 5% transfer fee, and `numOfSteps = 100`.
2. `tokenIssuer` approves `SecondSwap_StepVesting` and calls `createVesting(alice, 1000e18)` and `createVesting(bob, 1000e18)`.
3. For each call, `_createVesting` sets `totalAmount = 1000e18` and `releaseRate = 10e18`, while `token.safeTransferFrom(tokenIssuer, address(this), 1000e18)` delivers only `950e18` tokens to the vesting contract.
4. The contract now holds `1900e18` tokens in total, while the combined allocated `totalAmount` across `alice` and `bob` is `2000e18`.
5. After `endTime` is reached, `alice` calls `claim()`. `claimable(alice)` returns `1000e18`, and `token.safeTransfer(alice, 1000e18)` succeeds, leaving `900e18` tokens in the contract.
6. `bob` then calls `claim()`. `claimable(bob)` returns `1000e18`, and `token.safeTransfer(bob, 1000e18)` reverts because the contract only has `900e18` tokens remaining. `bob` is permanently unable to claim his vested allocation.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Accounting divergence between `_vestings[_beneficiary].totalAmount` and the contract's actual ERC20 token balance (`token.balanceOf(address(this))`), leading to denial of service in `claim()` for later claimants and unbacked vesting allocations that can be listed and sold on the SecondSwap marketplace.
- **Business Impact:** Beneficiaries and secondary-market buyers who purchase vesting allocations cannot claim their vested tokens at maturity because earlier claims exhaust the undercollateralized pool.

### Remediation

In `_createVesting`, execute the `safeTransferFrom` call before updating the beneficiary's vesting state and measure the exact amount of tokens received by comparing the contract's token balance before and after the transfer.

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

## Finding 4: Overbroad Authorization in `SecondSwap_StepVesting.createVesting` Allows Unauthorized Vesting Creation and Token Pulling from `tokenIssuer`

### Synopsis

The `SecondSwap_StepVesting.createVesting` function improperly grants vesting creation privileges to `manager`, alongside `tokenIssuer` and `vestingDeployer`. Because `_createVesting` pulls tokens directly from `tokenIssuer` via `safeTransferFrom`, the `manager` contract or its authorized callers can create new vestings and spend the `tokenIssuer`'s approved tokens without the issuer's consent.

### Technical details

In `contracts/SecondSwap_StepVesting.sol`, the `createVesting` function enforces access control at lines 245–248:

```solidity
require(
    msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer,
    "SS_StepVesting: unauthorized"
);
_createVesting(_beneficiary, _totalAmount, 0, false);
```

When `createVesting` calls `_createVesting(_beneficiary, _totalAmount, 0, false)`, the `_isInternal` flag is set to `false`. At lines 306–308 of `SecondSwap_StepVesting.sol`, when `!_isInternal` holds, the contract executes:

```solidity
if (!_isInternal) {
    token.safeTransferFrom(tokenIssuer, address(this), _totalAmount);
}
```

According to the protocol's design and role specifications in the documentation and `SecondSwap_VestingDeployer.sol`, only the `tokenIssuer` (either directly or routed via `SecondSwap_VestingDeployer`, which validates `_tokenOwner[msg.sender] == address(SecondSwap_StepVesting(_stepVesting).token())`) is intended to create new vesting schedules funded from the `tokenIssuer`'s balance. In contrast, the `manager` (`SecondSwap_VestingManager`) only requires permission to call `transferVesting` to reallocate existing vested balances during marketplace listing, unlisting, and purchases.

By including `msg.sender == manager` in the `require` check of `createVesting` (and similarly in `createVestings`), the contract creates a cross-contract authority scope mismatch: `manager` is granted the ability to mint new vesting schedules backed by pulling additional tokens directly from `tokenIssuer`'s wallet whenever `tokenIssuer` has an active ERC20 allowance to the `SecondSwap_StepVesting` contract.

### Assumptions and preconditions

1. A `SecondSwap_StepVesting` contract has been deployed and initialized with valid `tokenIssuer`, `manager`, and `vestingDeployer` addresses.
2. The `tokenIssuer` has granted an ERC20 token allowance to the `SecondSwap_StepVesting` contract that exceeds the currently created vestings (e.g., an upfront batch approval or residual allowance).
3. A call to `createVesting` originates from the `manager` address.

### Proof of concept

1. `tokenIssuer` approves `SecondSwap_StepVesting` to spend `10,000` tokens on its behalf via `token.approve( address(stepVesting), 10000e18 )`.
2. `tokenIssuer` creates an initial vesting of `1,000` tokens for Alice, leaving `9,000` tokens of unused allowance granted to `SecondSwap_StepVesting`.
3. The `manager` account calls `stepVesting.createVesting(beneficiary, 9000e18)`.
4. The authorization check at line 246 (`msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer`) passes because `msg.sender == manager`.
5. `_createVesting(beneficiary, 9000e18, 0, false)` executes and pulls `9,000` tokens from `tokenIssuer` into `SecondSwap_StepVesting` via `token.safeTransferFrom(tokenIssuer, address(this), 9000e18)`, allocating a new vesting schedule to `beneficiary` without the `tokenIssuer`'s authorization.

### Impact assessment

**Severity: Medium / High**

Granting `manager` permission to call `createVesting` violates the principle of least privilege and breaks the authority boundary between marketplace management (`manager`) and token issuance (`tokenIssuer` / `vestingDeployer`). If `tokenIssuer` maintains a standing or excess ERC20 allowance on `SecondSwap_StepVesting`, `manager` can unilaterally pull tokens from `tokenIssuer` and create unauthorized vesting allocations for arbitrary beneficiaries.

### Remediation

Remove `msg.sender == manager` from the authorization check in `createVesting` (and `createVestings`) so that only `tokenIssuer` or `vestingDeployer` can create new vesting schedules.

```solidity
    function createVesting(address _beneficiary, uint256 _totalAmount) external {
        require(
            msg.sender == tokenIssuer || msg.sender == vestingDeployer,
            "SS_StepVesting: unauthorized"
        );
        _createVesting(_beneficiary, _totalAmount, 0, false);
    }

    function createVestings(address[] memory _beneficiaries, uint256[] memory _totalAmounts) external {
        require(
            msg.sender == tokenIssuer || msg.sender == vestingDeployer,
            "SS_StepVesting: unauthorized"
        );
        require(_beneficiaries.length == _totalAmounts.length, "SS_StepVesting: array length mismatch");

        for (uint256 i = 0; i < _beneficiaries.length; i++) {
            _createVesting(_beneficiaries[i], _totalAmounts[i], 0, false);
        }
    }
```

## Finding 5: Incorrect Schedule Merging and Release Rate Calculation in `SecondSwap_StepVesting._createVesting` Distorts Beneficiary Claimable Entitlements

### Synopsis

In `SecondSwap_StepVesting`, `createVestings` and `_createVesting` merge new or transferred vesting allocations into a beneficiary's existing `Vesting` struct without reconciling `stepsClaimed` differences or settling prior unclaimed steps. This corrupts `releaseRate` and `stepsClaimed`, allowing premature claiming of unvested tokens or locking already-vested tokens into future steps.

### Technical details

`SecondSwap_StepVesting` stores a single `Vesting` struct (`stepsClaimed`, `amountClaimed`, `releaseRate`, `totalAmount`) per beneficiary in `_vestings[_beneficiary]`. Whenever `createVestings` (line 265), `createVesting` (line 249), or `transferVesting` (line 232) allocates tokens to `_beneficiary`, it invokes `_createVesting(_beneficiary, _totalAmount, _stepsClaimed, _isInternal)`.

In `_createVesting` (lines 288–304):

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

When `_vestings[_beneficiary].totalAmount != 0` (the `else` branch), `_createVesting` simply increments `_vestings[_beneficiary].totalAmount` by `_totalAmount` and recomputes `_vestings[_beneficiary].releaseRate` as:

$$\text{releaseRate} = \frac{\text{totalAmount} - \text{amountClaimed}}{\text{numOfSteps} - \text{stepsClaimed}}$$

This logic is flawed in multiple ways:

1. **Incoming `_stepsClaimed` is ignored when merging into an existing schedule:**
   `_createVesting` does not use `_stepsClaimed` in the `else` branch and keeps `_vestings[_beneficiary].stepsClaimed` unchanged.
   - If `_vestings[_beneficiary].stepsClaimed < _stepsClaimed` (for instance, the beneficiary holds a tiny existing allocation with `stepsClaimed == 0`, or allocations are pooled inside `SecondSwap_VestingManager` with `stepsClaimed == 0`), a transferred allocation whose early steps were already claimed by the grantor (`_stepsClaimed > 0`) has its `stepsClaimed` history erased. The beneficiary's `releaseRate` is increased across all `numOfSteps - _vestings[_beneficiary].stepsClaimed` steps, allowing the beneficiary to immediately claim `(currentStep - _vestings[_beneficiary].stepsClaimed)` steps of the newly added allocation that were only supposed to vest in future steps.
   - Conversely, if `_vestings[_beneficiary].stepsClaimed > _stepsClaimed` (for example, the beneficiary already claimed up to `currentStep` on an earlier small allocation and then receives a new allocation via `createVestings([beneficiary], [amount])` with `_stepsClaimed = 0`), `_vestings[_beneficiary].stepsClaimed` stays at `currentStep`. The beneficiary cannot claim the already-elapsed steps of the new allocation; instead, the entire new allocation is deferred into future steps `(currentStep, numOfSteps]`.

2. **Unclaimed elapsed steps on the existing schedule are not settled before updating `releaseRate`:**
   If a beneficiary has claimed `stepsClaimed < currentStep` (so `currentStep - stepsClaimed` steps have already vested at the old `releaseRate` but have not yet been claimed) and receives an additional allocation with `_stepsClaimed == currentStep`, recomputing `releaseRate` without first accruing the `currentStep - stepsClaimed` elapsed steps at the old rate and advancing `stepsClaimed` retroactively applies the new, higher `releaseRate` to those past steps `(stepsClaimed, currentStep]`.

### Assumptions and preconditions

- A `SecondSwap_StepVesting` contract is deployed with `numOfSteps > 1` and active vesting (`block.timestamp > startTime`).
- A beneficiary already has a non-zero vesting schedule (`_vestings[_beneficiary].totalAmount > 0`) or receives multiple allocations via `createVestings`, `createVesting`, or `transferVesting` at different claim states.

### Proof of concept

Consider a `SecondSwap_StepVesting` contract with `numOfSteps = 100` and `stepDuration = 1 days`:

1. **Setup:**
   - Grantor `Alice` is allocated `1,000e18` tokens (`releaseRate = 10e18` per step, `stepsClaimed = 0`).
   - Beneficiary `Bob` has a small existing allocation of `100` wei (`releaseRate = 1` wei per step, `stepsClaimed = 0`, `amountClaimed = 0`).

2. **Step 50 (`block.timestamp == startTime + 50 days`):**
   - `Alice` calls `claim()` and receives `50 * 10e18 = 500e18` tokens. Now `Alice` has `stepsClaimed = 50`, `amountClaimed = 500e18`, and `500e18` unvested tokens remaining for steps 51–100.
   - `Alice`'s remaining `500e18` vesting is transferred to `Bob` via `transferVesting(Alice, Bob, 500e18)` (or via marketplace listing/purchase).

3. **Flawed Merge in `_createVesting(Bob, 500e18, 50, true)`:**
   - Because `_vestings[Bob].totalAmount == 100 != 0`, execution enters the `else` branch (lines 295–304).
   - `_stepsClaimed = 50` is ignored; `_vestings[Bob].stepsClaimed` remains `0`.
   - `_vestings[Bob].totalAmount` becomes `500e18 + 100`.
   - `_vestings[Bob].releaseRate` becomes `(500e18 + 100 - 0) / (100 - 0) = 5e18 + 1`.

4. **Premature Claim by `Bob` at Step 50:**
   - `Bob` immediately calls `claim()` at step 50.
   - `claimable(Bob)` computes `claimableSteps = 50 - 0 = 50` and `claimableAmount = 50 * (5e18 + 1) = 250e18 + 50`.
   - Even though `Alice` already claimed all tokens for steps 1–50 on that allocation and `Bob` was only entitled to `50` wei at step 50, `Bob` immediately claims `250e18 + 50` tokens at step 50.

Conversely, if `Bob` had `totalAmount = 100e18` and already claimed at step 50 (`Bob.stepsClaimed = 50`, `Bob.amountClaimed = 50e18`), and the token issuer calls `createVestings([Bob], [1000e18])` at step 50 (`_stepsClaimed = 0`), `Bob.stepsClaimed` remains `50`. At step 50, `claimable(Bob)` returns `0` instead of the `500e18` tokens corresponding to the 50 elapsed steps of the new allocation.

### Impact assessment

- **Severity:** High / Medium.
- **Technical Impact:** Merging vesting schedules with different `stepsClaimed` values distorts `releaseRate` and `stepsClaimed`. Beneficiaries can prematurely claim tokens that are still unvested (draining tokens held for other beneficiaries in the contract), or have their already-vested tokens unfairly locked until future steps.
- **Business Impact:** Breaks core vesting schedule accounting invariants ("Is the claimable amount correct at all times? Will the claimable amount still be correct if the token allocation amount for a user is increased or decreased?").

### Remediation

Automatically claim or accrue any pending claimable tokens for `_beneficiary` before modifying their vesting schedule, or track vesting schedules such that `stepsClaimed` and `amountClaimed` are properly adjusted relative to `_stepsClaimed` when adding to an existing schedule. Specifically, if settling pending claims on top-ups/transfers:

```solidity
    function _createVesting(
        address _beneficiary,
        uint256 _totalAmount,
        uint256 _stepsClaimed,
        bool _isInternal
    ) internal {
        require(_beneficiary != address(0), "SS_StepVesting: beneficiary is zero");
        require(_totalAmount > 0, "SS_StepVesting: total amount is zero");

        if (_vestings[_beneficiary].totalAmount == 0) {
            uint256 remainingSteps = numOfSteps - _stepsClaimed;
            _vestings[_beneficiary] = Vesting({
                stepsClaimed: _stepsClaimed,
                amountClaimed: 0,
                releaseRate: remainingSteps != 0 ? _totalAmount / remainingSteps : 0,
                totalAmount: _totalAmount
            });
        } else {
            (uint256 claimableAmount, uint256 claimableSteps) = claimable(_beneficiary);
            if (claimableAmount > 0) {
                _vestings[_beneficiary].stepsClaimed += claimableSteps;
                _vestings[_beneficiary].amountClaimed += claimableAmount;
                token.safeTransfer(_beneficiary, claimableAmount);
                emit Claimed(_beneficiary, claimableAmount);
            }

            uint256 effectiveStepsClaimed = _isInternal
                ? _stepsClaimed
                : _vestings[_beneficiary].stepsClaimed;

            if (!_isInternal && _vestings[_beneficiary].stepsClaimed > 0) {
                uint256 immediateClaim = (_totalAmount * _vestings[_beneficiary].stepsClaimed) / numOfSteps;
                if (immediateClaim > 0) {
                    _vestings[_beneficiary].amountClaimed += immediateClaim;
                    token.safeTransferFrom(tokenIssuer, _beneficiary, immediateClaim);
                    emit Claimed(_beneficiary, immediateClaim);
                }
                _vestings[_beneficiary].totalAmount += _totalAmount;
                uint256 remainingSteps = numOfSteps - _vestings[_beneficiary].stepsClaimed;
                _vestings[_beneficiary].releaseRate = remainingSteps != 0
                    ? (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) / remainingSteps
                    : 0;
                if (_totalAmount > immediateClaim) {
                    token.safeTransferFrom(tokenIssuer, address(this), _totalAmount - immediateClaim);
                }
                emit VestingCreated(_beneficiary, _totalAmount);
                return;
            }

            _vestings[_beneficiary].stepsClaimed = effectiveStepsClaimed;
            _vestings[_beneficiary].totalAmount += _totalAmount;
            if (numOfSteps - _vestings[_beneficiary].stepsClaimed != 0) {
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

## Finding 6: Unchecked Division by `numOfSteps - _stepsClaimed` in `SecondSwap_StepVesting._createVesting` Causes Division-by-Zero DoS on Vesting Transfers

### Synopsis

In `SecondSwap_StepVesting._createVesting`, initializing a vesting schedule for a beneficiary with zero existing allocation divides `_totalAmount` by `numOfSteps - _stepsClaimed` without checking if all steps are already claimed, causing a division-by-zero revert that blocks vesting reallocations and marketplace transfers.

### Technical details

When `SecondSwap_StepVesting._createVesting` is called to allocate or transfer vesting tokens to `_beneficiary`, it branches on whether `_vestings[_beneficiary].totalAmount == 0` (line 288):

- In the `else` branch (lines 295–304, for beneficiaries with an existing non-zero `totalAmount`), the contract explicitly guards against `stepsClaimed` reaching `numOfSteps`:
  ```solidity
  if (numOfSteps - _vestings[_beneficiary].stepsClaimed != 0) {
      _vestings[_beneficiary].releaseRate =
          (_vestings[_beneficiary].totalAmount - _vestings[_beneficiary].amountClaimed) /
          (numOfSteps - _vestings[_beneficiary].stepsClaimed);
  } else {
      _vestings[_beneficiary].releaseRate = 0;
  }
  ```
- However, in the `if (_vestings[_beneficiary].totalAmount == 0)` branch (lines 288–294, when initializing a beneficiary with no current allocation), line 292 unconditionally computes:
  ```solidity
  releaseRate: _totalAmount / (numOfSteps - _stepsClaimed)
  ```

A grantor can legitimately hold an unclaimed balance (`grantorVesting.totalAmount - grantorVesting.amountClaimed > 0`) while `grantorVesting.stepsClaimed >= numOfSteps` in two ways:

1. **Top-up after full step completion:** A beneficiary claims their allocation once all `numOfSteps` have elapsed (setting `_vestings[grantor].stepsClaimed = numOfSteps`), and the `tokenIssuer`, `manager`, or `vestingDeployer` subsequently adds a new allocation via `createVesting` or `createVestings`. This executes the `else` branch (lines 296–303), increasing `totalAmount` while keeping `stepsClaimed == numOfSteps` and setting `releaseRate = 0`.
2. **Step duration truncation:** Because `stepDuration = (endTime - startTime) / numOfSteps` uses integer division, if `endTime - startTime` is not an exact multiple of `numOfSteps`, `currentStep = (currentTime - startTime) / stepDuration` reaches or exceeds `numOfSteps` before `endTime`.

When `transferVesting` (line 232) subsequently transfers any portion of this grantor's unclaimed balance to a `_beneficiary` whose `_vestings[_beneficiary].totalAmount == 0`, `_createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true)` enters the `if` branch at line 288 and evaluates `_totalAmount / (numOfSteps - _stepsClaimed)` at line 292. Because `_stepsClaimed >= numOfSteps`, `numOfSteps - _stepsClaimed` evaluates to `0` (or underflows), reverting the transaction with a Solidity panic (`Panic(0x12)` division by zero or `Panic(0x11)` arithmetic underflow).

This directly violates the protocol invariant stated in `README.md`: *"The token issuers must always be able to reallocate vestings."*

### Assumptions and preconditions

1. A vesting contract `SecondSwap_StepVesting` is deployed with `numOfSteps > 0`.
2. A beneficiary (`_grantor`) has claimed all `numOfSteps` steps (`_vestings[_grantor].stepsClaimed == numOfSteps`) and holds an unclaimed allocation (for instance, after receiving an additional allocation via `createVesting` or `createVestings`).
3. The `tokenIssuer`, `vestingDeployer`, or `manager` calls `transferVesting` to transfer tokens from `_grantor` to a `_beneficiary` address that currently has `_vestings[_beneficiary].totalAmount == 0`.

### Proof of concept

1. Deploy `SecondSwap_StepVesting` with `numOfSteps = 100`, `startTime = 1000`, and `endTime = 2000` (`stepDuration = 10`).
2. `tokenIssuer` calls `createVesting(Alice, 1000e18)`. Alice's vesting state is:
   - `totalAmount = 1000e18`
   - `stepsClaimed = 0`
   - `amountClaimed = 0`
   - `releaseRate = 10e18`
3. Advance `block.timestamp` to `2000` (`endTime`, so `currentStep = 100`).
4. Alice calls `claim()`. In `claimable(Alice)`, `claimableSteps = 100` and `claimableAmount = 1000e18`. Alice's state is updated to:
   - `stepsClaimed = 100` (`== numOfSteps`)
   - `amountClaimed = 1000e18`
   - `totalAmount = 1000e18`
5. `tokenIssuer` calls `createVesting(Alice, 500e18)` to grant Alice an additional allocation. Since `_vestings[Alice].totalAmount == 1000e18 != 0`, `_createVesting` takes the `else` branch (lines 295–304):
   - `Alice.totalAmount` becomes `1500e18`
   - `numOfSteps - Alice.stepsClaimed == 0`, so `Alice.releaseRate` is set to `0` (line 302)
   - `Alice.stepsClaimed` remains `100`
   - Alice now has `totalAmount - amountClaimed = 500e18` available.
6. `tokenIssuer` (or `vestingDeployer` / `manager`) calls `transferVesting(Alice, Bob, 500e18)` where Bob is a fresh recipient (`_vestings[Bob].totalAmount == 0`).
7. `transferVesting` passes the balance check (`1500e18 - 1000e18 >= 500e18`) and calls `_createVesting(Bob, 500e18, 100, true)`.
8. Because `_vestings[Bob].totalAmount == 0`, execution enters the `if` branch at line 288 and evaluates line 292:
   ```solidity
   releaseRate: 500e18 / (100 - 100)
   ```
   The call reverts with `Panic(0x12)` (division by zero), permanently blocking the transfer to `Bob`.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** Calling `transferVesting` from a grantor with `stepsClaimed >= numOfSteps` to any recipient with `_vestings[_beneficiary].totalAmount == 0` always reverts with an arithmetic panic (`Panic(0x12)` or `Panic(0x11)`).
- **Business Impact:** Token issuers cannot reallocate vesting schedules to new beneficiary wallets after `numOfSteps` have elapsed (breaking a core protocol invariant), and affected beneficiaries cannot list or transfer their vested allocations to the marketplace if the manager or buyer has no active balance in the vesting contract.

### Remediation

Check whether `numOfSteps > _stepsClaimed` before dividing when initializing a vesting schedule in `_createVesting`, setting `releaseRate` to `0` when all steps have already elapsed (matching the logic in the `else` branch):

```solidity
        if (_vestings[_beneficiary].totalAmount == 0) {
            uint256 rate = numOfSteps > _stepsClaimed
                ? _totalAmount / (numOfSteps - _stepsClaimed)
                : 0;
            _vestings[_beneficiary] = Vesting({
                stepsClaimed: _stepsClaimed,
                amountClaimed: 0,
                releaseRate: rate,
                totalAmount: _totalAmount
            });
        } else {
```

## Finding 7: Incorrect Sell-Limit Calculation in `SecondSwap_VestingManager.listVesting` Causes `maxSellPercent` Bypass and Blocks Resale of Purchased Allocations

### Synopsis

In `SecondSwap_VestingManager.listVesting`, the seller's `sellLimit` is dynamically derived from `SecondSwap_Vesting(plan).total(seller)`, `userAllocation.bought`, and `userAllocation.sold`. Because this formula conflates original, claimed, and reallocated tokens with marketplace-purchased tokens, users can bypass `maxSellPercent` to sell locked allocations or become unable to resell purchased tokens.

### Technical details

In `SecondSwap_VestingManager.sol`, `listVesting` enforces a maximum sellable limit on a user's vesting allocation:
- Tokens originally allocated to a beneficiary are restricted by `vestingSettings[plan].maxSellPercent` (e.g., 20%).
- Tokens purchased on the marketplace (tracked in `userAllocation.bought`) are intended to be 100% resellable.

To compute `sellLimit`, `listVesting` (lines 127–138) executes:
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

This calculation relies on the invariant that `currentAlloc + userAllocation.sold - userAllocation.bought` always equals the seller's original allocation and that all `userAllocation.bought` tokens remain available as purchased entitlement. This invariant breaks in multiple ways:

1. **Untracked Reallocations (`transferVesting`) Inflate or Corrupt `sellLimit`:**
   Token issuers can reallocate vestings via `SecondSwap_VestingDeployer.transferVesting` / `SecondSwap_StepVesting.transferVesting`, which directly modifies `_vestings[user].totalAmount` (`total(seller)`) without updating `allocations[user][plan]`.
   - **Blocking Resale of Purchased Tokens (DoS):** If a seller lists 20% of their original 1,000-token allocation (`userAllocation.sold = 200`, `currentAlloc = 800`) and the token issuer subsequently transfers out their remaining 800 tokens (`currentAlloc = 0`, `userAllocation.sold = 200`), the historical `sold` balance remains 200. If the user later purchases 100 tokens on the marketplace (`userAllocation.bought = 100`, `currentAlloc = 100`), `listVesting` computes:
     $$\text{sellLimit} = 100 + \frac{(100 + 200 - 100) \times 2000}{10000} = 140$$
     When the user attempts to list even `1` wei of their 100 purchased tokens, `userAllocation.sold` becomes $200 + 1 = 201 > 140$, reverting with `"SS_VestingManager: cannot list more than max sell percent"`.
   - **Bypassing `maxSellPercent` on Original Allocations:** If a seller has 100 original tokens, lists 20 tokens (reaching the 20% limit), buys 500 tokens (`userAllocation.bought = 500`, `currentAlloc = 580`), and has 500 tokens transferred out (`currentAlloc = 80`), `sellLimit` still starts at `userAllocation.bought = 500`. The seller can then list their remaining 80 original tokens (`userAllocation.sold = 100 <= 500`), selling 100% of their original allocation instead of 20%. Any subsequent original allocations up to 400 tokens can also be sold at 100%.

2. **Claiming Purchased Tokens Enables Selling Locked Original Allocations Beyond `maxSellPercent`:**
   `SecondSwap_Vesting(plan).total(seller)` returns `_vestings[seller].totalAmount`, which is never decremented when `SecondSwap_StepVesting.claim()` is called. When a user with both an original allocation and a marketplace-purchased allocation claims vested tokens, a portion of their purchased tokens is already claimed into their wallet, yet `userAllocation.bought` and `total(seller)` remain unchanged. The user retains the full `userAllocation.bought` resale quota and can apply the already-claimed portion of `userAllocation.bought` to list and sell their remaining unvested original allocation far above `maxSellPercent`.

### Assumptions and preconditions

- A vesting plan is deployed and marked `sellable` in `SecondSwap_VestingManager` with `maxSellPercent < BASE` (e.g., default `2000` = 20%).
- Either:
  - A user with an original vesting allocation also purchases a vesting allocation on the marketplace and claims vested tokens as steps elapse, or
  - A token issuer reallocates vesting tokens via `transferVesting` for a user who has previously listed or purchased vesting tokens.

### Proof of concept

#### Scenario A: Bypassing `maxSellPercent` via Claimed Purchased Tokens
1. Alice receives an original vesting allocation of `1,000e18` tokens (`maxSellPercent = 2000`, i.e., 20%). Her maximum sellable original allocation is `200e18`.
2. At 50% of the vesting schedule, Alice purchases `1,000e18` tokens from a marketplace listing via `completePurchase`.
   - `_vestings[alice].totalAmount = 2,000e18`
   - `allocations[alice][plan].bought = 1,000e18`
   - `allocations[alice][plan].sold = 0`
3. Alice calls `SecondSwap_StepVesting.claim()`, claiming 50% of her total allocation (`1,000e18` tokens: `500e18` from her original allocation and `500e18` from her purchased allocation).
   - `available(alice) = 1,000e18` (`500e18` unvested original tokens + `500e18` unvested purchased tokens).
   - `total(alice) = 2,000e18` (unchanged by `claim()`).
4. Alice calls `Marketplace.listVesting` for all `1,000e18` remaining unvested tokens:
   - `available(alice) == 1,000e18 >= 1,000e18`.
   - `sellLimit = 1,000e18 + ((2,000e18 + 0 - 1,000e18) * 2000) / 10000 = 1,200e18`.
   - `userAllocation.sold = 1,000e18 <= 1,200e18`.
5. The listing succeeds. Alice has sold all `500e18` of her remaining unvested original allocation (50% of her initial `1,000e18` original allocation and 100% of her remaining unvested original allocation), bypassing the 20% `maxSellPercent` restriction.

#### Scenario B: Permanent DoS on Reselling Purchased Tokens After Reallocation
1. Bob receives an original allocation of `1,000e18` tokens (`maxSellPercent = 20%`).
2. Bob lists `200e18` tokens on the marketplace (`currentAlloc = 800e18`, `sold = 200e18`, `bought = 0`).
3. The token issuer reallocates Bob's remaining `800e18` tokens to another address via `transferVesting` (`currentAlloc = 0`, `sold = 200e18`, `bought = 0`).
4. Bob purchases `100e18` tokens on the marketplace (`currentAlloc = 100e18`, `sold = 200e18`, `bought = 100e18`).
5. Bob attempts to list `100e18` (or even `1` wei) of his newly purchased tokens:
   - `sellLimit = 100e18 + ((100e18 + 200e18 - 100e18) * 2000) / 10000 = 140e18`.
   - `userAllocation.sold` becomes `200e18 + amount > 140e18`, reverting with `"SS_VestingManager: cannot list more than max sell percent"`.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effects:**
  1. Sellers can bypass `maxSellPercent` and liquidate up to 100% of their remaining unvested original tokens on the secondary market.
  2. Users whose original allocations were reduced by token issuer reallocations after listing can have their `userAllocation.sold` exceed their computed `sellLimit`, freezing their ability to resell tokens subsequently bought on the marketplace.
- **Business Implications:** Breaks two core protocol guarantees stated in the specification: enforcing the issuer-configured `maxSellPercent` cap on original vesting allocations, and allowing secondary-market buyers to freely resell purchased allocations even when token issuers reallocate vestings.

### Remediation

Track original allocations and purchased allocations explicitly (and update them across claims, listings, purchases, and reallocations), or base the sell limit on remaining unclaimed/available allocations while synchronizing `SecondSwap_VestingManager` during `transferVesting` and `claim`.

If computing the sell limit within `SecondSwap_VestingManager.listVesting` using available unclaimed balances and capping `bought` to currently available tokens:

```solidity
        uint256 currentAlloc = SecondSwap_Vesting(plan).available(seller);
        uint256 sellLimit = userAllocation.bought;

        if (currentAlloc + userAllocation.sold > userAllocation.bought) {
            sellLimit +=
                ((currentAlloc + userAllocation.sold - userAllocation.bought) * vestingSettings[plan].maxSellPercent) /
                BASE;
        } else {
            sellLimit = currentAlloc + userAllocation.sold;
        }

        userAllocation.sold += amount;

        require(userAllocation.sold <= sellLimit, "SS_VestingManager: cannot list more than max sell percent");
        SecondSwap_Vesting(plan).transferVesting(seller, address(this), amount);
```

Additionally, ensure `SecondSwap_VestingManager` updates `allocations` whenever `transferVesting` is invoked by the token issuer or deployer so that `sold` and `bought` quotas remain consistent with reallocated balances.

## Finding 8: Missing `sellable` Check in `SecondSwap_VestingManager.completePurchase` Allows Buying Disabled Vestings

### Synopsis

`SecondSwap_VestingManager.completePurchase` fails to verify whether the target vesting plan is currently configured as `sellable` before executing the purchase transfer. Consequently, buyers can still purchase already-listed vesting tokens even after the admin disables marketplace trading for that vesting plan via `setSellable(vesting, false)`.

### Technical details

In `SecondSwap_VestingManager`, the `vestingSettings` mapping maintains configuration parameters for each vesting contract, including a `bool sellable` flag that controls whether tokens from a given vesting plan can be traded on the marketplace. According to the protocol specification, the SecondSwap admin (`s2Admin`) can enable and disable marketplace trading for vesting plans at any time using `setSellable(address vesting, bool sellable)` (lines 173–186).

When a user lists vested tokens for sale, `listVesting` enforces this policy on line 122:
```solidity
require(vestingSettings[plan].sellable, "vesting not sellable");
```

However, when a buyer purchases a listed vesting allocation and the marketplace calls `completePurchase` (lines 161–164), no check is performed on `vestingSettings[vesting].sellable`:
```solidity
function completePurchase(address buyer, address vesting, uint256 amount) external onlyMarketplace {
    allocations[buyer][vesting].bought += amount;
    SecondSwap_Vesting(vesting).transferVesting(address(this), buyer, amount);
}
```

Because `completePurchase` does not validate `vestingSettings[vesting].sellable`, disabling a vesting plan's sellability only prevents new listings in `listVesting` while leaving all existing active listings in the marketplace fully purchasable. Any buyer can still complete purchases of those listed vestings, directly bypassing the admin's restriction.

### Assumptions and preconditions

1. A vesting contract is deployed and initially has `vestingSettings[vesting].sellable == true` (set automatically upon deployment via `VestingDeployer`).
2. A seller lists a vesting allocation on the marketplace while `sellable` is `true`.
3. `s2Admin` calls `setSellable(vesting, false)` to disable trading of the vesting plan on the marketplace.

### Proof of concept

1. A vesting plan `V` is deployed through `SecondSwap_VestingDeployer`, which calls `IVestingManager(manager).setSellable(V, true)` (line 130).
2. Alice (a beneficiary of `V`) lists 100 vested tokens on the marketplace. The marketplace calls `VestingManager.listVesting(alice, V, 100)`, which passes `require(vestingSettings[V].sellable, "vesting not sellable")` and transfers the 100 vesting tokens to `VestingManager`.
3. Later, `s2Admin` decides to halt marketplace trading for vesting plan `V` and calls `VestingManager.setSellable(V, false)`. Now `vestingSettings[V].sellable` is `false`.
4. Bob spots Alice's existing listing on the marketplace and initiates a purchase of the 100 vested tokens.
5. The marketplace contract calls `VestingManager.completePurchase(bob, V, 100)`.
6. Because `completePurchase` (lines 161–164) lacks a `require(vestingSettings[vesting].sellable, ...)` check, the call succeeds, updating `allocations[bob][V].bought` and transferring the 100 vesting allocation to Bob despite `V` being marked as non-sellable.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Inconsistent policy enforcement across the listing and purchase stages of the marketplace lifecycle. Disabling `sellable` on a vesting contract fails to prevent trades on existing listings.
- **Business Impact:** When the protocol administrator disables marketplace trading for a vesting plan (e.g., due to compliance requirements, token issuer request, or emergency suspension), active listings remain fillable, allowing unauthorized secondary market transfers of locked tokens to continue.

### Remediation

Add a `require` statement in `completePurchase` to verify that the vesting plan is currently marked as `sellable` before completing the purchase.

```solidity
    function completePurchase(address buyer, address vesting, uint256 amount) external onlyMarketplace {
        require(vestingSettings[vesting].sellable, "vesting not sellable");
        allocations[buyer][vesting].bought += amount;
        SecondSwap_Vesting(vesting).transferVesting(address(this), buyer, amount);
    }
```

## Finding 9: Ambiguous Initialization Check in `SecondSwap_VestingManager.setSellable` Resets Custom Fee and `maxSellPercent` Settings

### Synopsis

`SecondSwap_VestingManager.setSellable` relies on `vestingSetting.maxSellPercent == 0` to detect whether a vesting plan's settings are uninitialized. If the admin configures `maxSellPercent` to `0` (and/or sets custom `buyerFee` and `sellerFee`), any subsequent `setSellable(vesting, true)` call treats the plan as uninitialized and overwrites `maxSellPercent`, `buyerFee`, and `sellerFee` with default values.

### Technical details

In `contracts/SecondSwap_VestingManager.sol`, `setSellable` both toggles the `sellable` flag of a vesting plan and initializes default parameters (`maxSellPercent = 2000`, `buyerFee = -1`, `sellerFee = -1`) when the vesting plan is first registered by `SecondSwap_VestingDeployer`:

```solidity
function setSellable(address vesting, bool sellable) external {
    require(s2Admin == msg.sender || vestingDeployer == msg.sender, "SS_VestingManager: Unauthorised user");

    VestingSettings storage vestingSetting = vestingSettings[vesting];

    vestingSetting.sellable = sellable;
    if (vestingSetting.maxSellPercent == 0 && vestingSetting.sellable) {
        vestingSetting.maxSellPercent = 2000;
        vestingSetting.buyerFee = -1;
        vestingSetting.sellerFee = -1;
        emit MaxSellPercentUpdated(vesting, 2000);
    }
    emit VestingSellableUpdated(vesting, sellable);
}
```

The `VestingSettings` struct does not track an explicit `initialized` state. Instead, line 179 uses `vestingSetting.maxSellPercent == 0` to infer whether the struct has ever been initialized. This creates an initialization-state ambiguity:

1. A `maxSellPercent` of `0` is a valid policy configuration (e.g., when the admin wants to disallow original vesting recipients from selling their initial allocation unless they have purchased tokens on the secondary market, or when pre-configuring parameters before enabling trading).
2. Whenever `vestingSetting.maxSellPercent` is `0`, calling `setSellable(vesting, true)`—such as when disabling and later re-enabling sellability for the vesting plan—causes the condition `vestingSetting.maxSellPercent == 0 && vestingSetting.sellable` to evaluate to `true`.
3. Consequently, `vestingSetting.maxSellPercent` is silently reset to `2000` (20%), and any custom `buyerFee` and `sellerFee` values configured for that vesting plan are overwritten with `-1`.

### Assumptions and preconditions

1. A vesting plan is deployed and registered with `SecondSwap_VestingManager`.
2. `s2Admin` configures `maxSellPercent = 0` for the vesting plan (and optionally configures custom `buyerFee` and/or `sellerFee`).
3. `setSellable(vesting, true)` is subsequently called (e.g., after temporarily disabling marketplace listings via `setSellable(vesting, false)` and later re-enabling them).

### Proof of concept

1. `SecondSwap_VestingDeployer` deploys a new `SecondSwap_StepVesting` contract `V` and calls `VestingManager.setSellable(V, true)`. `vestingSettings[V]` is initialized with `sellable = true`, `maxSellPercent = 2000`, `buyerFee = -1`, `sellerFee = -1`.
2. `s2Admin` updates the settings for `V` to set custom fees (e.g., `buyerFee = 100`, `sellerFee = 100`) and sets `maxSellPercent = 0` so original allocations cannot be sold above 0% (only secondary-purchased allocations can be resold).
3. `s2Admin` temporarily disables marketplace listings for `V` by calling `setSellable(V, false)`.
4. `s2Admin` later re-enables marketplace listings for `V` by calling `setSellable(V, true)`.
5. Inside `setSellable(V, true)`, because `vestingSettings[V].maxSellPercent == 0`, the initialization branch executes again:
   - `vestingSettings[V].maxSellPercent` is reset from `0` to `2000`.
   - `vestingSettings[V].buyerFee` is reset from `100` to `-1`.
   - `vestingSettings[V].sellerFee` is reset from `100` to `-1`.
6. Users can now list up to 20% of their original vesting allocations and trade under default fees instead of the admin-configured fees.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: Re-enabling sellability on a vesting plan where `maxSellPercent == 0` silently corrupts `VestingSettings` state by resetting `maxSellPercent` to `2000` and wiping custom `buyerFee` and `sellerFee` configurations back to `-1`.
- **Business Impact**: Violates marketplace sell limits by allowing holders to sell up to 20% of locked tokens when `maxSellPercent` was intentionally restricted to `0`, and causes the protocol/token issuer to charge incorrect marketplace fees.

### Remediation

Track initialization explicitly in `VestingSettings` (or only initialize default settings when called by `vestingDeployer` during initial deployment) so that toggling `sellable` does not overwrite existing configurations.

```solidity
    struct VestingSettings {
        bool initialized;
        bool sellable;
        uint256 maxSellPercent;
        int256 buyerFee;
        int256 sellerFee;
    }
```

```solidity
    function setSellable(address vesting, bool sellable) external {
        require(s2Admin == msg.sender || vestingDeployer == msg.sender, "SS_VestingManager: Unauthorised user");

        VestingSettings storage vestingSetting = vestingSettings[vesting];

        vestingSetting.sellable = sellable;
        if (!vestingSetting.initialized && vestingSetting.sellable) {
            vestingSetting.initialized = true;
            vestingSetting.maxSellPercent = 2000;
            vestingSetting.buyerFee = -1;
            vestingSetting.sellerFee = -1;
            emit MaxSellPercentUpdated(vesting, 2000);
        }
        emit VestingSellableUpdated(vesting, sellable);
    }
```
