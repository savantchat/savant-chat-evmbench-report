# Audit report — 2024-02-althea-liquid-infrastructure/H-01

## Finding 1: Lack of Uniqueness and Custody Synchronization in `LiquidInfrastructureERC20.ManagedNFTs` Breaks Revenue Withdrawals

### Synopsis

`LiquidInfrastructureERC20` tracks managed revenue-generating NFTs in an unvalidated dynamic array (`ManagedNFTs`) without enforcing membership uniqueness or custody invariants across initialization, registration, and release. Duplicate or unowned entries in `ManagedNFTs` desynchronize the registry from actual NFT custody, causing `withdrawFromManagedNFTs` to revert and blocking revenue collection.

### Technical details

In `LiquidInfrastructureERC20.sol`, the contract maintains its registry of managed `LiquidInfrastructureNFT` contracts in the state variable `ManagedNFTs` (line 60):

```solidity
address[] public ManagedNFTs;
```

Instead of enforcing set semantics (e.g., via OpenZeppelin's `EnumerableSet.AddressSet` or a dedicated membership mapping), `ManagedNFTs` is a raw `address[]` array whose lifecycle operations fail to preserve registry-to-custody integrity:

1. **Unchecked Duplicate Registration in `addManagedNFT`:**
   `addManagedNFT(address nftContract)` (lines 394–403) only checks that `nft.ownerOf(nft.AccountId()) == address(this)` before executing `ManagedNFTs.push(nftContract)` (line 401). It does not verify whether `nftContract` is already present in `ManagedNFTs`. Because `LiquidInfrastructureERC20` remains the owner of the NFT while holding it, `addManagedNFT` can be called multiple times for the same `nftContract` (or called for an NFT already populated via the constructor), inserting duplicate entries into `ManagedNFTs`.

2. **Unvalidated Initialization in `constructor`:**
   The constructor (line 464) directly assigns `ManagedNFTs = _managedNFTs;` without verifying that `_managedNFTs` contains unique addresses or that `address(this)` actually owns the corresponding `AccountId` token of each NFT.

3. **Incomplete Removal and Broken Validation in `releaseManagedNFT`:**
   When `releaseManagedNFT(address nftContract, address to)` (lines 413–434) is called, it transfers custody of the NFT out of the contract (`nft.transferFrom(address(this), to, nft.AccountId())` at line 418) and iterates over `ManagedNFTs` to swap-and-pop the released address, terminating immediately upon the first match (`break;` at line 427). Moreover, line 431 contains a tautological check `require(true, "unable to find released NFT in ManagedNFTs");` that never reverts if the NFT was not found. If `nftContract` was registered more than once in `ManagedNFTs`, at least one entry for `nftContract` remains in `ManagedNFTs` after custody of the NFT has already been transferred to `to`. Additionally, mutating `ManagedNFTs` via swap-and-pop while a batched withdrawal is in progress (`nextWithdrawal > 0`) shifts unvisited entries or leaves `nextWithdrawal` out of sync with `ManagedNFTs.length`.

4. **Denial of Service in `withdrawFromManagedNFTs`:**
   `withdrawFromManagedNFTs` (lines 359–386) and `withdrawFromAllManagedNFTs` (lines 351–353) iterate through `ManagedNFTs` and call `withdrawFrom.withdrawBalancesTo(withdrawERC20s, address(this))` (line 377). Because `LiquidInfrastructureNFT.withdrawBalancesTo` is restricted to the current owner of `AccountId`, any stale or unowned entry remaining in `ManagedNFTs` causes `withdrawBalancesTo` to revert, preventing `LiquidInfrastructureERC20` from completing withdrawals from subsequent managed NFTs.

### Assumptions and preconditions

1. `LiquidInfrastructureERC20` is deployed and holds at least one `LiquidInfrastructureNFT` (or is initialized with `_managedNFTs` in the constructor).
2. Either an NFT address is present more than once in `ManagedNFTs` (via `_managedNFTs` and/or `addManagedNFT`) and later released via `releaseManagedNFT`, or an unowned NFT address is included in `_managedNFTs` at construction, or `releaseManagedNFT` is called while a partial `withdrawFromManagedNFTs` traversal is active (`nextWithdrawal > 0`).

### Proof of concept

1. The owner transfers custody of `nft1` (`AccountId = 1`) to `LiquidInfrastructureERC20`.
2. `addManagedNFT(address(nft1))` is called twice (or `nft1` is passed in `_managedNFTs` and then `addManagedNFT(address(nft1))` is called once custody is transferred). Both calls succeed because `nft1.ownerOf(1) == address(liquidERC20)`, resulting in `ManagedNFTs = [address(nft1), address(nft1)]`.
3. Later, the owner calls `releaseManagedNFT(address(nft1), newOwner)`.
   - `nft1.transferFrom(address(this), newOwner, 1)` transfers ownership of `nft1` to `newOwner`.
   - The loop in `releaseManagedNFT` removes `ManagedNFTs[0]` by copying `ManagedNFTs[1]` into `ManagedNFTs[0]`, popping the last element, and executing `break;`.
   - `ManagedNFTs` still contains `[address(nft1)]`, even though `newOwner` now owns `nft1`.
4. When `withdrawFromAllManagedNFTs()` or `withdrawFromManagedNFTs(...)` is called to collect revenue from the protocol's managed NFTs, it invokes `nft1.withdrawBalancesTo(withdrawERC20s, address(this))`.
5. Because `LiquidInfrastructureERC20` is no longer the owner of `nft1`, `withdrawBalancesTo` reverts, blocking withdrawals across the contract. Furthermore, because `LiquidInfrastructureERC20` no longer owns `nft1`, calling `releaseManagedNFT(address(nft1), ...)` again to purge the leftover entry also reverts at `nft.transferFrom(address(this), to, nft.AccountId())`, leaving the stale entry permanently stuck in `ManagedNFTs`.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** `ManagedNFTs` can permanently retain addresses of NFTs that `LiquidInfrastructureERC20` does not own (and cannot remove via `releaseManagedNFT` because `transferFrom` will fail once custody is lost), or corrupt the `nextWithdrawal` traversal cursor.
- **Business Impact:** Revenue collection via `withdrawFromManagedNFTs` and `withdrawFromAllManagedNFTs` is permanently bricked whenever a stale/unowned NFT entry is encountered, preventing `LiquidInfrastructureERC20` from pulling accrued ERC20 revenue from its valid `LiquidInfrastructureNFT` contracts for distribution to token holders.

### Remediation

Track managed NFTs using a mapping (or OpenZeppelin's `EnumerableSet.AddressSet`) to enforce uniqueness in `ManagedNFTs`, validate that removed NFTs actually exist in `ManagedNFTs`, disallow or adjust `nextWithdrawal` when mutating `ManagedNFTs`, and avoid populating unowned NFTs in the constructor.

```solidity
    mapping(address => bool) public isManagedNFT;
    address[] public ManagedNFTs;
```

And update `constructor`, `addManagedNFT`, and `releaseManagedNFT` to maintain the registry invariant:

```solidity
    function addManagedNFT(address nftContract) public onlyOwner {
        require(!LockedForDistribution, "cannot add NFT during distribution");
        require(nextWithdrawal == 0, "cannot add NFT during withdrawal");
        require(!isManagedNFT[nftContract], "NFT already managed");
        LiquidInfrastructureNFT nft = LiquidInfrastructureNFT(nftContract);
        address nftOwner = nft.ownerOf(nft.AccountId());
        require(
            nftOwner == address(this),
            "this contract does not own the new ManagedNFT"
        );
        isManagedNFT[nftContract] = true;
        ManagedNFTs.push(nftContract);
        emit AddManagedNFT(nftContract);
    }

    function releaseManagedNFT(
        address nftContract,
        address to
    ) public onlyOwner nonReentrant {
        require(!LockedForDistribution, "cannot release NFT during distribution");
        require(nextWithdrawal == 0, "cannot release NFT during withdrawal");
        require(isManagedNFT[nftContract], "unable to find released NFT in ManagedNFTs");

        isManagedNFT[nftContract] = false;
        for (uint i = 0; i < ManagedNFTs.length; i++) {
            if (ManagedNFTs[i] == nftContract) {
                ManagedNFTs[i] = ManagedNFTs[ManagedNFTs.length - 1];
                ManagedNFTs.pop();
                break;
            }
        }

        LiquidInfrastructureNFT nft = LiquidInfrastructureNFT(nftContract);
        nft.transferFrom(address(this), to, nft.AccountId());

        emit ReleaseManagedNFT(nftContract, to);
    }
```

## Finding 2: Duplicate entries in `holders` via zero-amount or burn transfers in `LiquidInfrastructureERC20._beforeTokenTransfer` cause unbounded array growth and DoS

### Synopsis

`LiquidInfrastructureERC20._beforeTokenTransfer` unconditionally appends `to` to the `holders` array whenever `this.balanceOf(to) == 0`, without verifying that `amount > 0` or `to != address(0)`. An approved holder can repeatedly execute zero-amount transfers (or burns) to bloat `holders` with duplicate entries, causing out-of-gas DoS across transfers and distributions.

### Technical details

In `LiquidInfrastructureERC20._beforeTokenTransfer` (lines 127–146), the contract tracks active token holders by checking whether the recipient `to` currently has a zero balance before the transfer occurs:

```solidity
bool exists = (this.balanceOf(to) != 0);
if (!exists) {
    holders.push(to);
}
```

And in `_afterTokenTransfer` (lines 164–179), if the sender `from` has a zero balance after the transfer, `from` is removed from `holders`:

```solidity
bool stillHolding = (this.balanceOf(from) != 0);
if (!stillHolding) {
    for (uint i = 0; i < holders.length; i++) {
        if (holders[i] == from) {
            holders[i] = holders[holders.length - 1];
            holders.pop();
        }
    }
}
```

This holder-tracking logic contains flaws in `_beforeTokenTransfer`:

1. **Zero-amount transfers to an approved account with zero balance:**
   If an approved account `A` has `balanceOf(A) == 0`, any caller (including `A` itself, or another holder `B` who has a non-zero balance) can call `transfer(A, 0)`.
   - In `_beforeTokenTransfer(from, A, 0)`, `this.balanceOf(A)` is `0`, so `exists` is `false` and `A` is pushed to `holders` (line 144).
   - If `from` has a non-zero balance (`balanceOf(from) > 0`), then in `_afterTokenTransfer(from, A, 0)`, `stillHolding` (`this.balanceOf(from) != 0`) evaluates to `true`, so nothing is removed from `holders`.
   - Even after `A` is already in `holders`, `A`'s balance remains `0`. Subsequent calls to `transfer(A, 0)` from an account with a non-zero balance will repeatedly see `this.balanceOf(A) == 0` and push `A` into `holders` again and again.
   - Furthermore, once `A` eventually receives a positive token balance (`balanceOf(A) > 0`), `A` will appear multiple times in `holders`! Wait—even worse, during `distribute()`, `A` would receive distributions multiple times per epoch (draining funds), or if `holders` is bloated with thousands of entries, `distribute`, `distributeToAllHolders`, and `_afterTokenTransfer` (which loops through the entire `holders` array on full balance transfers/burns) will revert with Out-Of-Gas.

2. **Burns (`to == address(0)`):**
   When any holder burns tokens (via `burn` or `burnFrom`), `_burn` invokes `_beforeTokenTransfer(account, address(0), amount)`.
   - Because `this.balanceOf(address(0))` is always `0`, `exists` is `false`, so `address(0)` is pushed to `holders` on every single burn operation (line 144).
   - If the burning account only burns a partial balance (`stillHolding == true`), `_afterTokenTransfer` does not even iterate over `holders`, and even if it did, it only removes `from`, never `address(0)`. Thus `address(0)` accumulates permanently in `holders` on every burn.

Because `holders` can be arbitrarily inflated by any user (via repeated `transfer(approvedZeroBalanceAccount, 0)` from an account with `balance > 0`, or partial burns `burn(0)` / `burn(1)`), the `holders` array grows without bound.

### Assumptions and preconditions

- At least one account is approved in `HolderAllowlist` (e.g., via `approveHolder` or constructor `_approvedHolders`).
- Either an account holds at least 1 wei of `LiquidInfrastructureERC20` (to call `burn(0)` or `transfer(approvedAccountWithZeroBalance, 0)` without triggering removal in `_afterTokenTransfer`), or a holder performs partial burns.

### Proof of concept

1. Suppose `Holder1` is an approved holder with a balance of `100` tokens, and `Holder2` is an approved holder with a balance of `0` tokens (or `Holder1` simply calls `burn(0)`).
2. **Vector A (via `burn(0)`):**
   - `Holder1` calls `burn(0)` repeatedly in a loop (e.g., 500 times in a single transaction).
   - In each call, `_burn(Holder1, 0)` calls `_beforeTokenTransfer(Holder1, address(0), 0)`.
   - Line 133 `if (!(to == address(0)))` skips the `isApprovedHolder` check.
   - Line 142 checks `this.balanceOf(address(0)) != 0`, which is `false`.
   - Line 144 executes `holders.push(address(0))`.
   - In `_afterTokenTransfer(Holder1, address(0), 0)`, `this.balanceOf(Holder1)` is `100 != 0`, so `stillHolding` is `true` and the loop is skipped.
   - `address(0)` is appended to `holders` 500 times per transaction.
3. **Vector B (via `transfer(Holder2, 0)`):**
   - `Holder1` calls `transfer(Holder2, 0)` N times.
   - In each call, `_beforeTokenTransfer(Holder1, Holder2, 0)` sees `this.balanceOf(Holder2) == 0` and pushes `Holder2` into `holders`.
   - In `_afterTokenTransfer(Holder1, Holder2, 0)`, `this.balanceOf(Holder1) == 100 != 0`, so nothing is popped.
   - `holders` now contains `Holder2` N times.
4. **Consequences:**
   - When `Holder1` (or any other holder) later tries to transfer their full balance or burn their remaining tokens, `_afterTokenTransfer` loops through the entire bloated `holders` array (`for (uint i = 0; i < holders.length; i++)`), reverting with Out-Of-Gas and permanently locking users from transferring their full balance or burning.
   - `distributeToAllHolders()`, `mintAndDistribute()`, `burnAndDistribute()`, and `burnFromAndDistribute()` revert with Out-Of-Gas because iterating through `holders` exceeds the block gas limit.
   - Additionally, in Vector B, if `Holder1` transfers `100` tokens to `Holder2`, `Holder2` is in `holders` N+1 times with `balanceOf(Holder2) == 100`, allowing `Holder2` to receive N+1 times their entitlement during `distribute()`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  1. Unbounded storage growth of the `holders` array via zero-amount transfers (`transfer(to, 0)`) or burns (`burn(0)` / partial burns).
  2. Denial of Service (Out-Of-Gas) on `_afterTokenTransfer` whenever any holder attempts to transfer or burn their entire balance, as well as on `distributeToAllHolders()`, `mintAndDistribute()`, `burnAndDistribute()`, and `burnFromAndDistribute()`.
  3. Duplicate entries for a valid holder in `holders` lead to multiple reward payouts to the same holder in `distribute()`, stealing yield from other holders.

### Remediation

In `_beforeTokenTransfer`, only add `to` to `holders` if `to != address(0)` and `amount > 0` (and `this.balanceOf(to) == 0`). Additionally, consider using OpenZeppelin's `EnumerableSet.AddressSet` for `holders` to guarantee uniqueness and $O(1)$ additions and removals.

Minimal fix in `_beforeTokenTransfer`:

```solidity
    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        require(!LockedForDistribution, "distribution in progress");
        if (!(to == address(0))) {
            require(
                isApprovedHolder(to),
                "receiver not approved to hold the token"
            );
        }
        if (from == address(0) || to == address(0)) {
            _beforeMintOrBurn();
        }
        if (to != address(0) && amount > 0 && this.balanceOf(to) == 0) {
            holders.push(to);
        }
    }
```

## Finding 3: Mint and Burn Lock in `LiquidInfrastructureERC20._beforeMintOrBurn` Causes Post-Initialization and Zero-Period Supply Deadlocks

### Synopsis

`LiquidInfrastructureERC20._beforeTokenTransfer` invokes `_beforeMintOrBurn`, which requires `!_isPastMinDistributionPeriod()`. Because `_isPastMinDistributionPeriod` bypasses the period check when `totalSupply() == 0` without updating `LastDistribution` upon the first mint and uses a non-strict `>=` boundary comparison, transitioning from empty supply locks subsequent mints/burns and permanently deadlocks supply changes when `MinDistributionPeriod == 0`.

### Technical details

In `LiquidInfrastructureERC20`, minting and burning operations invoke `_beforeTokenTransfer` (lines 127–146), which calls `_beforeMintOrBurn()` whenever `from == address(0) || to == address(0)` (lines 139–141):

```solidity
if (from == address(0) || to == address(0)) {
    _beforeMintOrBurn();
}
```

`_beforeMintOrBurn()` (lines 151–155) enforces that a distribution is not currently due before allowing any supply modification:

```solidity
function _beforeMintOrBurn() internal view {
    require(
        !_isPastMinDistributionPeriod(),
        "must distribute before minting or burning"
    );
}
```

`_isPastMinDistributionPeriod()` (lines 243–250) is defined as:

```solidity
function _isPastMinDistributionPeriod() internal view returns (bool) {
    // Do not force a distribution with no holders or supply
    if (totalSupply() == 0 || holders.length == 0) {
        return false;
    }

    return (block.number - LastDistribution) >= MinDistributionPeriod;
}
```

This lifecycle design creates two closely related failure modes at the distribution-period boundary and empty-supply transition:

1. **Permanent Mint and Burn Deadlock When `MinDistributionPeriod == 0`**:
   When `MinDistributionPeriod` is configured as `0` (or when `> 0` is not enforced), the first `mint` succeeds because `totalSupply() == 0` causes `_isPastMinDistributionPeriod()` to return `false`. Once that first mint completes, `totalSupply() > 0` and `holders.length > 0`. From that point onward, `(block.number - LastDistribution) >= 0` always evaluates to `true`—even in the exact same block in which `_endDistribution()` sets `LastDistribution = block.number` (line 293). Consequently, calling `distributeToAllHolders()`, `mintAndDistribute()`, `burnAndDistribute()`, or `burnFromAndDistribute()` cannot clear the lock: `distributeToAllHolders()` finishes and sets `LastDistribution = block.number`, and the immediately following `mint()` or `burn()` call in the same transaction checks `_beforeMintOrBurn()`, evaluates `0 >= 0` as `true`, and reverts with `"must distribute before minting or burning"`. All future mints and burns are permanently bricked.

2. **Empty-Supply Transition Lockout After `MinDistributionPeriod` Blocks**:
   `LastDistribution` is initialized to `block.number` in the constructor (line 465) and is only updated in `_endDistribution()` (line 293). While `totalSupply() == 0` (either after deployment before initial minting, or after all existing tokens have been burned), `_isPastMinDistributionPeriod()` returns `false`. During this empty-supply period:
   - `distribute()` cannot be called to refresh `LastDistribution` because line 202 requires `_isPastMinDistributionPeriod()` to be `true`.
   - Once `block.number - LastDistribution >= MinDistributionPeriod` has elapsed while `totalSupply() == 0`, the first `mint()` call succeeds (because `totalSupply() == 0` in `_beforeTokenTransfer`), transitioning `totalSupply()` to a non-zero value without updating `LastDistribution`.
   - Immediately after this first mint, `_isPastMinDistributionPeriod()` flips to `true` in the very same block. Any subsequent `mint()` (such as batch-minting initial balances to multiple approved holders, as done in `transferERC20ToHolders` in `scripts/contract-deployer.ts`) or `burn()` immediately reverts with `"must distribute before minting or burning"`, forcing an unnecessary distribution cycle right after the first holder is minted tokens.

### Assumptions and preconditions

- The contract is deployed with `MinDistributionPeriod == 0`, or at least `MinDistributionPeriod` blocks elapse while `totalSupply() == 0` (for example, during initial setup of `ManagedNFTs` and `approveHolder` calls as in `scripts/contract-deployer.ts`, or after all tokens are burned).
- At least one approved holder is minted a non-zero amount of `LiquidInfrastructureERC20` tokens, transitioning `totalSupply()` from `0` to `> 0`.

### Proof of concept

#### Case 1: Permanent Mint/Burn Deadlock (`MinDistributionPeriod == 0`)
1. The owner deploys `LiquidInfrastructureERC20` with `_minDistributionPeriod = 0` and approves `holder1` and `holder2`.
2. The owner calls `mint(holder1, 100)`.
   - Inside `_beforeTokenTransfer(address(0), holder1, 100)`, `_beforeMintOrBurn()` calls `_isPastMinDistributionPeriod()`.
   - Because `totalSupply()` is `0` prior to state update, `_isPastMinDistributionPeriod()` returns `false`.
   - The mint succeeds; `totalSupply()` is now `100` and `holders` is `[holder1]`.
3. The owner calls `mintAndDistribute(holder2, 100)` (or `holder1` calls `burnAndDistribute(50)`):
   - `_isPastMinDistributionPeriod()` checks `(block.number - LastDistribution) >= 0`, which is `true`.
   - `distributeToAllHolders()` executes a full distribution and calls `_endDistribution()`, setting `LastDistribution = block.number`.
   - Next, `mint(holder2, 100)` (or `burn(50)`) executes in the same transaction.
   - `_beforeMintOrBurn()` calls `_isPastMinDistributionPeriod()`, which evaluates `(block.number - block.number) >= 0` (`0 >= 0`), returning `true`.
   - `_beforeMintOrBurn()` reverts with `"must distribute before minting or burning"`.
4. Neither `mint`, `burn`, `mintAndDistribute`, `burnAndDistribute`, nor `burnFromAndDistribute` can ever succeed again.

#### Case 2: Empty-Supply Transition Failure (`MinDistributionPeriod > 0`)
1. `LiquidInfrastructureERC20` is deployed at block `100` with `MinDistributionPeriod = 10`, so `LastDistribution = 100`.
2. Setup transactions (approving holders, configuring NFTs) advance the chain to block `115`. `totalSupply()` is still `0`.
3. The owner executes a batch mint to approved holders `holder1` and `holder2`:
   - `mint(holder1, 100)` at block `115` checks `_isPastMinDistributionPeriod()`. Because `totalSupply() == 0`, it returns `false` and the mint succeeds (`totalSupply() = 100`, `LastDistribution` remains `100`).
   - `mint(holder2, 100)` at block `115` checks `_isPastMinDistributionPeriod()`. Now `totalSupply() == 100 > 0` and `115 - 100 = 15 >= 10`, so `_isPastMinDistributionPeriod()` returns `true` and the transaction reverts with `"must distribute before minting or burning"`.

### Impact assessment

- **Severity**: Medium / High.
- **Technical Impact**:
  - When `MinDistributionPeriod == 0`, all minting and burning functionality (including `mintAndDistribute`, `burnAndDistribute`, and `burnFromAndDistribute`) is permanently deadlocked after the first mint, freezing total supply and preventing holders from burning tokens.
  - When `MinDistributionPeriod > 0`, transitioning from `totalSupply() == 0` to `totalSupply() > 0` after `MinDistributionPeriod` blocks have elapsed immediately locks the contract against further mints or burns until a distribution is executed over the single initial holder, breaking batch initialization workflows (`transferERC20ToHolders`).

### Remediation

Reset `LastDistribution = block.number` when transitioning from an empty supply (`totalSupply() == 0`) during a mint in `_beforeTokenTransfer`, and ensure `MinDistributionPeriod` is strictly positive (or use a strict inequality `>` when `MinDistributionPeriod == 0` is permitted so same-block post-distribution mints and burns can proceed).

```solidity
    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        require(!LockedForDistribution, "distribution in progress");
        if (!(to == address(0))) {
            require(
                isApprovedHolder(to),
                "receiver not approved to hold the token"
            );
        }
        if (from == address(0) && totalSupply() == 0 && amount > 0) {
            LastDistribution = block.number;
        }
        if (from == address(0) || to == address(0)) {
            _beforeMintOrBurn();
        }
        if (to != address(0)) {
            bool exists = (this.balanceOf(to) != 0);
            if (!exists) {
                holders.push(to);
            }
        }
    }

    /**
     * Implements an additional lock on minting and burning, ensuring that supply changes happen after any potential distributions
     */
    function _beforeMintOrBurn() internal view {
        require(
            !_isPastMinDistributionPeriod() || LastDistribution == block.number,
            "must distribute before minting or burning"
        );
    }
```

## Finding 4: Unbounded `holders` Loop in `LiquidInfrastructureERC20._afterTokenTransfer` Enables Gas Exhaustion Denial of Service

### Synopsis

`LiquidInfrastructureERC20._afterTokenTransfer` iterates over the entire `holders` array whenever `from` has a zero balance after a token transfer, mint, or burn. Because `holders` can grow unboundedly (including via adversarial bloating), this unbounded loop can exceed the block gas limit, causing a denial of service on token operations.

### Technical details

In `LiquidInfrastructureERC20`, active token holders are tracked in the dynamic storage array `holders`. When tokens are transferred, minted, or burned, OpenZeppelin's `ERC20` implementation invokes `_beforeTokenTransfer` followed by `_afterTokenTransfer`.

In `_afterTokenTransfer` (lines 164–179):
```solidity
bool stillHolding = (this.balanceOf(from) != 0);
if (!stillHolding) {
    for (uint i = 0; i < holders.length; i++) {
        if (holders[i] == from) {
            // Remove the element at i by copying the last one into its place and removing the last element
            holders[i] = holders[holders.length - 1];
            holders.pop();
        }
    }
}
```

This implementation introduces a gas-driven Denial of Service (DoS) vulnerability due to unbounded holder-dependent execution costs:

1. **Unbounded Linear Scan on Every Mint, Full Transfer, and Full Burn:** Whenever `this.balanceOf(from) == 0`, `_afterTokenTransfer` iterates through the entire `holders` array from `0` to `holders.length` without early termination (`break`). Notably, during `_mint`, `from` is `address(0)` whose balance is always `0`; thus, every `mint` call executes a full scan of the `holders` array even though `address(0)` is not a valid holder. Similarly, whenever a holder transfers or burns their entire balance, the full array is scanned.
2. **Adversarial Registry Growth:** In `_beforeTokenTransfer` (lines 142–145), `to` is pushed to `holders` whenever `this.balanceOf(to) == 0`. A user holding at least 1 wei of the token (`this.balanceOf(from) != 0`) can repeatedly call `transfer(to, 0)` where `to` is any approved holder with a `0` balance, or call `burn(0)` (which pushes `address(0)` to `holders`). Because `this.balanceOf(from) != 0`, `_afterTokenTransfer` skips the removal loop during the bloating calls, allowing the attacker to cheaply append thousands of duplicate entries to `holders`.
3. **Operation Lockouts:** Once `holders.length` grows sufficiently large (either organically or via adversarial registry growth), any call that enters the `if (!stillHolding)` loop in `_afterTokenTransfer`—or batch functions like `distributeToAllHolders()` (line 184) used by `mintAndDistribute`, `burnAndDistribute`, and `burnFromAndDistribute`—will revert with an out-of-gas error, locking out minting, full-balance transfers, and burning.

### Assumptions and preconditions

1. `LiquidInfrastructureERC20` is deployed and active.
2. Either the number of holders grows large over time, or an account with a non-zero balance (or any account calling `burn(0)`) repeatedly triggers `_beforeTokenTransfer` with a zero-balance `to` address to bloat the `holders` array.

### Proof of concept

1. Assume `Alice` is an approved holder with a balance of `100` tokens, and `Bob` is an approved holder with a balance of `0` tokens (or `Alice` uses `burn(0)` where `to == address(0)`).
2. `Alice` repeatedly calls `transfer(Bob, 0)` (or `burn(0)`).
3. In each call, `_beforeTokenTransfer` checks `this.balanceOf(to) != 0`, which evaluates to `false`, and executes `holders.push(to)` in $O(1)$ gas.
4. In each call, `_afterTokenTransfer` checks `this.balanceOf(Alice) != 0`, which evaluates to `true` (`stillHolding == true`), bypassing the `for` loop in $O(1)$ gas.
5. After `holders` is bloated with thousands of entries:
   - When the owner calls `mint(recipient, amount)`, `_afterTokenTransfer(address(0), recipient, amount)` checks `this.balanceOf(address(0)) != 0` (`false`) and iterates over the entire `holders` array, running out of gas and reverting.
   - When any holder attempts to transfer or burn their entire token balance, `this.balanceOf(from)` becomes `0`, triggering the unbounded loop in `_afterTokenTransfer` and reverting due to gas exhaustion.
   - Calls to `distributeToAllHolders()` (and `mintAndDistribute`, `burnAndDistribute`, `burnFromAndDistribute`) also revert due to gas exhaustion over `holders.length`.

### Impact assessment

**Severity: High (Medium/High DoS)**

- **Technical Impact:** Execution gas costs in `_afterTokenTransfer` and `distributeToAllHolders` scale linearly with `holders.length`. When `holders.length` becomes large enough that iterating through it exceeds the block gas limit, all `mint` operations, full-balance transfers, full burns, and `distributeToAllHolders` calls permanently revert with out-of-gas errors.
- **Business Impact:** Core lifecycle operations of `LiquidInfrastructureERC20`—specifically minting new shares to investors, exiting positions completely via transfer or burn, and executing automated batch distributions—are rendered unusable.

### Remediation

Replace the raw `address[] private holders` array and linear loop in `_afterTokenTransfer` with OpenZeppelin's `EnumerableSet.AddressSet` (or track each holder's 1-based index in a mapping `mapping(address => uint256) private holderIndex`) so that holder addition and removal in `_afterTokenTransfer` execute in $O(1)$ constant time, and skip `from == address(0)` on mints.

Using `EnumerableSet.AddressSet` (or index mapping):

```solidity
    function _afterTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        if (from != address(0) && this.balanceOf(from) == 0) {
            // Using OpenZeppelin EnumerableSet.AddressSet for O(1) removal:
            // holdersSet.remove(from);
            // Or if maintaining a 1-based index mapping `holderIndex`:
            uint256 index = holderIndex[from];
            if (index != 0) {
                uint256 lastIndex = holders.length;
                address lastHolder = holders[lastIndex - 1];
                holders[index - 1] = lastHolder;
                holderIndex[lastHolder] = index;
                holders.pop();
                delete holderIndex[from];
            }
        }
    }
```

Additionally, ensure `_beforeTokenTransfer` only adds `to` when `to != address(0)` and `amount > 0` (or checks `this.balanceOf(to) > 0` in `_afterTokenTransfer` using the $O(1)$ set/mapping) so zero-value transfers and burns cannot add duplicate entries to `holders`.

## Finding 5: Improper Holder Removal and Unchecked Zero-Balance Entries in `LiquidInfrastructureERC20._afterTokenTransfer` Corrupt the `holders` Registry

### Synopsis

`LiquidInfrastructureERC20._afterTokenTransfer` fails to maintain synchronization between token balances and the `holders` array. It only checks `from`'s post-transfer balance (ignoring zero-balance `to` entries created by zero-amount transfers or burns) and mutates `holders` via swap-and-pop during forward iteration without re-inspecting the swapped index, leaving duplicate and stale entries that corrupt reward distributions.

### Technical details

`LiquidInfrastructureERC20` tracks active token holders in the `holders` storage array so `distribute()` can iterate over all holders and transfer their pro-rata share of `distributableERC20s`.

In `_beforeTokenTransfer` (lines 142–145), whenever `this.balanceOf(to) == 0`, `to` is unconditionally pushed onto `holders`:

```solidity
bool exists = (this.balanceOf(to) != 0);
if (!exists) {
    holders.push(to);
}
```

`_afterTokenTransfer` (lines 164–179) is responsible for cleaning up `holders` after a transfer, mint, or burn so that only accounts with non-zero balances remain registered:

```solidity
function _afterTokenTransfer(
    address from,
    address to,
    uint256 amount
) internal virtual override {
    bool stillHolding = (this.balanceOf(from) != 0);
    if (!stillHolding) {
        for (uint i = 0; i < holders.length; i++) {
            if (holders[i] == from) {
                // Remove the element at i by copying the last one into its place and removing the last element
                holders[i] = holders[holders.length - 1];
                holders.pop();
            }
        }
    }
}
```

This implementation has two interconnected flaws in how it synchronizes `holders` with actual token balances:

1. **Failure to reconcile `to`'s post-transfer balance:**
   - When a zero-amount transfer (`transfer(to, 0)`) is made to an approved holder `to` whose balance is currently `0`, `_beforeTokenTransfer` pushes `to` onto `holders`. After the transfer completes, `this.balanceOf(to)` is still `0`, but `_afterTokenTransfer` only checks `this.balanceOf(from) != 0`. As a result, `to` remains in `holders` with a `0` balance. Repeated zero-amount transfers to `to` push multiple duplicate entries of `to` into `holders`.
   - Similarly, on any `burn` or `burnFrom` call (`to == address(0)`), `_beforeTokenTransfer` sees `this.balanceOf(address(0)) == 0` and pushes `address(0)` into `holders`. Because `_afterTokenTransfer` only checks `from`, `address(0)` is left behind in `holders`.

2. **Collection mutation during forward iteration skips swapped elements:**
   - When `from`'s balance reaches `0`, `_afterTokenTransfer` iterates forward with `for (uint i = 0; i < holders.length; i++)` and removes matching entries by overwriting `holders[i]` with `holders[holders.length - 1]` and calling `holders.pop()`.
   - Because the loop immediately increments `i` without re-evaluating the new element placed at `holders[i]`, if `holders[holders.length - 1]` is also equal to `from` (or if duplicate entries of `from` are adjacent), the swapped-in `from` entry is skipped and remains in `holders` even after `from`'s balance is `0`.

Consequently, an approved holder can accumulate multiple entries in `holders`, acquire tokens prior to a distribution, and receive multiple payouts of `erc20EntitlementPerUnit[j] * this.balanceOf(recipient)` in `distribute()`.

### Assumptions and preconditions

- At least one address `to` is approved in `HolderAllowlist` (or tokens are burned so `to == address(0)`).
- `LockedForDistribution` is currently `false`, allowing token transfers.
- To extract excess rewards, an approved holder has `0` balance at the time zero-amount transfers are made to it (either before receiving its initial tokens or after temporarily transferring its balance to another approved account) and subsequently holds a positive balance when `distribute()` is called.

### Proof of concept

1. Suppose `Alice` and `Bob` are approved holders (`HolderAllowlist[Alice] = true`, `HolderAllowlist[Bob] = true`). `Alice` holds `100` tokens (`holders = [Alice]`), and `Bob` currently holds `0` tokens.
2. Any account (e.g., `Bob` or an unapproved external address with `0` balance) calls `transfer(Bob, 0)` three times:
   - On each call, `_beforeTokenTransfer` checks `this.balanceOf(Bob) != 0` (`false`) and pushes `Bob` to `holders`.
   - `_afterTokenTransfer` checks `this.balanceOf(from) != 0` and only removes `from` if `from` is not `Bob`.
   - `holders` is now `[Alice, Bob, Bob, Bob]` even though `balanceOf(Bob) == 0`.
3. Even if `Bob` calls `transfer(Alice, 0)` (so `from == Bob` and `this.balanceOf(Bob) == 0`):
   - In `_afterTokenTransfer`:
     - `i = 0`: `holders[0]` is `Alice` (not `Bob`).
     - `i = 1`: `holders[1]` is `Bob`. It sets `holders[1] = holders[3]` (`Bob`) and pops index `3`. `holders` is now `[Alice, Bob, Bob]`.
     - `i = 2`: `holders[2]` is `Bob`. It sets `holders[2] = holders[2]` (`Bob`) and pops index `2`. `holders` is now `[Alice, Bob]`.
     - `i = 3`: `3 < holders.length` (`3 < 2`) is `false`, so the loop exits.
   - `Bob` still remains in `holders` with a `0` balance because the swapped element at `i = 1` was never re-checked.
4. `Alice` transfers `50` tokens to `Bob`. In `_beforeTokenTransfer`, `this.balanceOf(Bob)` is still `0`, so `Bob` is pushed again: `holders` becomes `[Alice, Bob, Bob]`, with `balanceOf(Alice) = 50` and `balanceOf(Bob) = 50` (`totalSupply = 100`).
5. When `distribute()` is called to distribute `100` reward tokens (`erc20EntitlementPerUnit = 100 / 100 = 1`):
   - `i = 0` (`Alice`): receives `50` reward tokens.
   - `i = 1` (`Bob`): receives `50` reward tokens.
   - `i = 2` (`Bob`): `toDistribute.transfer(Bob, 50)` either transfers another `50` tokens (if excess rewards exist in the contract) or reverts due to insufficient reward balance, permanently blocking `_endDistribution()` and leaving `LockedForDistribution = true`.

### Impact assessment

- **Severity:** High
- **Direct Technical Effects:**
  1. **Reward Theft / Over-distribution:** A holder appearing multiple times in `holders` receives their full entitlement multiple times per distribution epoch at the expense of other holders or undistributed balances.
  2. **Permanent Denial of Service (Distribution Lockup):** Because `erc20EntitlementPerUnit` is computed as `balance / totalSupply`, paying a duplicated holder multiple times causes the total transferred rewards to exceed the contract's `distributableERC20s` balance. When `toDistribute.transfer()` reverts on the final holders, `_endDistribution()` can never be reached, leaving `LockedForDistribution == true` and permanently freezing all transfers, mints, burns, and NFT withdrawals.
  3. **Gas Griefing:** Pushing arbitrary numbers of stale zero-balance entries or `address(0)` entries into `holders` bloats the array, causing `distributeToAllHolders()` and `_afterTokenTransfer()` to run out of gas.

### Remediation

In `_afterTokenTransfer`, decrement `i` (or avoid incrementing `i`) after swapping and popping so the newly swapped element at `holders[i]` is inspected, and also ensure that `to` is removed from `holders` if its post-transfer balance is `0` (such as on zero-amount transfers or burns to `address(0)`). Alternatively, use OpenZeppelin's `EnumerableSet.AddressSet` to guarantee O(1) deduplicated insertion and removal.

```solidity
    function _afterTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal virtual override {
        if (this.balanceOf(from) == 0) {
            for (uint i = 0; i < holders.length; ) {
                if (holders[i] == from) {
                    holders[i] = holders[holders.length - 1];
                    holders.pop();
                } else {
                    i++;
                }
            }
        }
        if (this.balanceOf(to) == 0) {
            for (uint i = 0; i < holders.length; ) {
                if (holders[i] == to) {
                    holders[i] = holders[holders.length - 1];
                    holders.pop();
                } else {
                    i++;
                }
            }
        }
    }
```

## Finding 6: Unchecked Direct `IERC20.transfer` Call in `LiquidInfrastructureERC20.distribute` Causes Distribution DoS With Non-Standard or Reverting ERC20 Tokens

### Synopsis

In `LiquidInfrastructureERC20.distribute`, reward payouts invoke `toDistribute.transfer(recipient, entitlement)` directly inside an `if` condition. This fails to support non-standard ERC20 tokens with missing return values (e.g., USDT) and fails to catch reverting transfers, causing `distribute` to revert and locking the contract in distribution mode.

### Technical details

During revenue distribution, `LiquidInfrastructureERC20.distribute` iterates through the `holders` array and, for each approved holder, loops over `distributableERC20s` to transfer their calculated `entitlement`:

```solidity
for (uint j = 0; j < distributableERC20s.length; j++) {
    IERC20 toDistribute = IERC20(distributableERC20s[j]);
    uint256 entitlement = erc20EntitlementPerUnit[j] *
        this.balanceOf(recipient);
    if (toDistribute.transfer(recipient, entitlement)) {
        receipts[j] = entitlement;
    }
}
```

The `if (toDistribute.transfer(recipient, entitlement))` statement assumes that `transfer` always conforms strictly to the `IERC20` interface (returning a `bool`) and that a failed transfer returns `false` rather than reverting, allowing the loop to skip failed payouts and record `0` in `receipts[j]`.

This creates a critical interoperability flaw in two ways:
1. **Non-standard ERC20 return semantics:** Widely used tokens such as USDT do not return a boolean from `transfer(address,uint256)`. Because `toDistribute` is typed as `IERC20`, Solidity's ABI decoder attempts to decode a 32-byte `bool` from empty return data (`returndatasize() == 0`), which immediately reverts the entire transaction.
2. **Transfer failure and settlement divergence:** Standard ERC20 implementations (including OpenZeppelin's `ERC20`) revert on transfer failure instead of returning `false`. Similarly, tokens with transfer restrictions (such as USDC/USDT blacklists, pausable tokens, or tokens that revert on zero-value transfers when `entitlement == 0`) revert rather than returning `false`. Because `toDistribute.transfer` is not wrapped in a `try/catch` or low-level call, any single reverting transfer aborts the entire `distribute()` call.

Once `_beginDistribution()` has been executed (or if a multi-batch distribution is already underway), `LockedForDistribution` is set to `true`. If `distribute()` subsequently reverts on a token transfer, `_endDistribution()` cannot be reached, leaving `LockedForDistribution` stuck at `true` and blocking all token transfers, mints, burns, and NFT withdrawals.

### Assumptions and preconditions

1. `LiquidInfrastructureERC20` holds at least one distributable ERC20 token in `distributableERC20s` and has at least one approved holder in `holders` with a non-zero balance.
2. Either:
   - One of the tokens in `distributableERC20s` is a non-standard ERC20 that does not return a boolean from `transfer` (such as USDT), or
   - A transfer to one of the recipients reverts (e.g., the recipient address is blacklisted by the reward token, or the token reverts on zero-value transfers when `entitlement == 0`).

### Proof of concept

1. The contract is configured with `distributableERC20s` containing a token with non-standard return semantics (e.g., USDT, which has `function transfer(address to, uint value) public` with no return value) or a token with blacklist functionality (e.g., USDC).
2. Approved holders `Alice` and `Bob` are minted `LiquidInfrastructureERC20` tokens, and revenue is withdrawn into `LiquidInfrastructureERC20`.
3. `MinDistributionPeriod` blocks elapse.
4. Batch 1 of distribution is executed via `distribute(1)`, paying `Alice` (or `_beginDistribution()` starts inside `distributeToAllHolders()`). `LockedForDistribution` is now `true`.
5. When processing `Bob` (or on the first holder if the token is USDT):
   - In the USDT case, `toDistribute.transfer(recipient, entitlement)` succeeds inside USDT, but the caller reverts when attempting to ABI-decode a `bool` from empty return data.
   - In the blacklisted recipient case, `toDistribute.transfer(Bob, entitlement)` reverts instead of returning `false`, contrary to the `if (...)` branch's expectation.
6. The `distribute` transaction reverts. `nextDistributionRecipient` cannot advance to `holders.length`, `_endDistribution()` is never called, and `LockedForDistribution` remains `true`, locking all transfers, mints, burns, and withdrawals from managed NFTs.

### Impact assessment

- **Severity:** Medium / High (DoS of core distribution and token lifecycle).
- **Technical Impact:** `distribute()` and `distributeToAllHolders()` revert whenever a distributable token lacks a boolean return value or reverts on transfer to a recipient, preventing reward distribution to all remaining holders and locking the contract in `LockedForDistribution = true` state.
- **Business Impact:** Revenue accrued from managed infrastructure NFTs becomes stuck in the contract, and investors are unable to receive payouts or transfer, mint, or burn `LiquidInfrastructureERC20` shares.

### Remediation

Perform a low-level call to `transfer` so that both reverting transfers and non-standard tokens (which return no data or return a boolean) are handled safely without reverting the entire distribution batch, and skip zero-entitlement transfers:

```solidity
        for (i = nextDistributionRecipient; i < limit; i++) {
            address recipient = holders[i];
            if (isApprovedHolder(recipient)) {
                uint256[] memory receipts = new uint256[](
                    distributableERC20s.length
                );
                for (uint j = 0; j < distributableERC20s.length; j++) {
                    address toDistribute = distributableERC20s[j];
                    uint256 entitlement = erc20EntitlementPerUnit[j] *
                        this.balanceOf(recipient);
                    if (entitlement > 0) {
                        (bool success, bytes memory data) = toDistribute.call(
                            abi.encodeWithSelector(
                                IERC20.transfer.selector,
                                recipient,
                                entitlement
                            )
                        );
                        if (
                            success &&
                            (data.length == 0 || abi.decode(data, (bool)))
                        ) {
                            receipts[j] = entitlement;
                        }
                    }
                }

                emit Distribution(recipient, distributableERC20s, receipts);
            }
        }
```

## Finding 7: Skipping Disapproved Holders in `LiquidInfrastructureERC20.distribute` Leaves Their Entitled Share of Rewards Undistributed

### Synopsis

In `LiquidInfrastructureERC20.distribute()`, checking `isApprovedHolder(recipient)` skips reward payouts to disapproved holders even though `_beginDistribution()` includes their token balances in `totalSupply()` when calculating `erc20EntitlementPerUnit`. Consequently, the disapproved holders' share of rewards is neither paid to them nor distributed to approved holders, leaving funds stuck in the contract.

### Technical details

In `LiquidInfrastructureERC20.sol`, token holders must be allowlisted via `approveHolder()` to receive `LiquidInfrastructureERC20` tokens (enforced in `_beforeTokenTransfer` at lines 133–138). If the owner later calls `disapproveHolder(holder)` (lines 116–119), `HolderAllowlist[holder]` is set to `false`. As documented in the NatSpec for `disapproveHolder` and verified in the test suite (`basicErc20HolderTests`), disapproving an existing holder only prevents them from receiving *additional* `LiquidInfrastructureERC20` tokens; they continue to hold their existing token balance in `holders` and can reduce it via `transfer()`, `burn()`, or `burnAndDistribute()`.

When a revenue distribution begins in `_beginDistribution()` (lines 270–277), the per-unit entitlement for each distributable ERC20 is computed using `this.totalSupply()`:

```solidity
uint256 supply = this.totalSupply();
for (uint i = 0; i < distributableERC20s.length; i++) {
    uint256 balance = IERC20(distributableERC20s[i]).balanceOf(
        address(this)
    );
    uint256 entitlement = balance / supply;
    erc20EntitlementPerUnit.push(entitlement);
}
```

Because `this.totalSupply()` includes the balances of all current token holders—both approved and disapproved—each `erc20EntitlementPerUnit[j]` allocates a proportional fraction of the reward pool to the tokens held by disapproved holders.

However, when `distribute()` iterates through `holders` (lines 214–231), line 216 gates the payout on `isApprovedHolder(recipient)`:

```solidity
for (i = nextDistributionRecipient; i < limit; i++) {
    address recipient = holders[i];
    if (isApprovedHolder(recipient)) {
        ...
        for (uint j = 0; j < distributableERC20s.length; j++) {
            IERC20 toDistribute = IERC20(distributableERC20s[j]);
            uint256 entitlement = erc20EntitlementPerUnit[j] *
                this.balanceOf(recipient);
            if (toDistribute.transfer(recipient, entitlement)) {
                receipts[j] = entitlement;
            }
        }
        emit Distribution(recipient, distributableERC20s, receipts);
    }
}
```

This creates an accounting mismatch between `_beginDistribution()` and `distribute()`:
1. **Omitted Beneficiary / Undistributed Rewards**: When a holder in `holders` is not approved (`isApprovedHolder(recipient) == false`), `distribute()` skips transferring `erc20EntitlementPerUnit[j] * this.balanceOf(recipient)`. Because their balance was already included in the `supply` denominator in `_beginDistribution()`, approved holders only receive their own fraction of `totalSupply()`, while the disapproved holder's portion (`balance * balanceOf(disapprovedHolder) / totalSupply`) remains undistributed in the contract.
2. **Forced Forfeiture on `burnAndDistribute()`**: If `MinDistributionPeriod` has elapsed and a disapproved holder calls `burnAndDistribute()` to exit their position, `distributeToAllHolders()` executes first while they still hold their tokens (diluting `erc20EntitlementPerUnit` for all approved holders while paying the disapproved holder `0`), and then burns their tokens.
3. **Trivial Bypass via Transfer**: Because `_beforeTokenTransfer()` only checks `isApprovedHolder(to)` and allows a disapproved holder (`from`) to transfer tokens to any approved holder (`to`), a disapproved holder can bypass line 216 prior to `distribute()` by transferring their balance to an approved address.

### Assumptions and preconditions

1. An account `holder` was previously approved via `approveHolder(holder)` (or in the constructor) and acquired a non-zero balance of `LiquidInfrastructureERC20`.
2. The contract owner calls `disapproveHolder(holder)` while `holder` still holds a non-zero balance of `LiquidInfrastructureERC20`.
3. `MinDistributionPeriod` blocks elapse and `distribute()` (or `distributeToAllHolders()` / `burnAndDistribute()`) is invoked to distribute accrued `distributableERC20s`.

### Proof of concept

1. Assume `holder1` and `holder2` are both approved holders, and each is minted `500` `LiquidInfrastructureERC20` tokens (`totalSupply() == 1000`).
2. `1,000,000` units of reward token `erc20a` are withdrawn from a managed NFT into `LiquidInfrastructureERC20`.
3. The owner calls `disapproveHolder(holder2)`. `holder2` still holds `500` tokens (`balanceOf(holder2) == 500`, `totalSupply() == 1000`).
4. After `MinDistributionPeriod` blocks pass, `distributeToAllHolders()` (or `holder2.burnAndDistribute(500)`) is called:
   - `_beginDistribution()` computes `supply = 1000` and `erc20EntitlementPerUnit[0] = 1,000,000 / 1000 = 1,000`.
   - In `distribute()`, for `i = 0` (`recipient = holder1`), `isApprovedHolder(holder1)` is `true`, so `holder1` receives `1,000 * 500 = 500,000` `erc20a`.
   - For `i = 1` (`recipient = holder2`), `isApprovedHolder(holder2)` at line 216 is `false`, so `holder2` is skipped and receives `0` `erc20a`.
5. Distribution finishes (`_endDistribution()`). Out of the `1,000,000` `erc20a` reward pool, only `500,000` was distributed to `holder1`, `holder2` received `0`, and `500,000` `erc20a` remains stuck in `LiquidInfrastructureERC20` until a future distribution cycle.

### Impact assessment

- **Severity**: Medium
- **Technical Effect**: Whenever a disapproved holder retains a non-zero balance during a distribution, their proportional share of every token in `distributableERC20s` is deducted from the per-unit entitlement of approved holders (via `this.totalSupply()`) but never transferred out in `distribute()`.
- **Business Impact**: Approved holders receive less than 100% of the distributed revenue pool in that epoch (their payouts are diluted by the disapproved holder's token balance), while the disapproved holder receives nothing even when forced to distribute via `burnAndDistribute()` before burning their tokens.

### Remediation

Remove the `isApprovedHolder(recipient)` check in `distribute()` so that all current holders included in `totalSupply()` receive their proportional distribution before they burn or transfer their tokens (since `HolderAllowlist` already restricts who can receive `LiquidInfrastructureERC20` tokens in `_beforeTokenTransfer`). Alternatively, if disapproved holders must be excluded from distributions, track the total supply held by disapproved holders and subtract it from `this.totalSupply()` when calculating `erc20EntitlementPerUnit` in `_beginDistribution()`.

```solidity
        for (i = nextDistributionRecipient; i < limit; i++) {
            address recipient = holders[i];
            uint256[] memory receipts = new uint256[](
                distributableERC20s.length
            );
            for (uint j = 0; j < distributableERC20s.length; j++) {
                IERC20 toDistribute = IERC20(distributableERC20s[j]);
                uint256 entitlement = erc20EntitlementPerUnit[j] *
                    this.balanceOf(recipient);
                if (toDistribute.transfer(recipient, entitlement)) {
                    receipts[j] = entitlement;
                }
            }

            emit Distribution(recipient, distributableERC20s, receipts);
        }
```

## Finding 8: Division Before Multiplication in `LiquidInfrastructureERC20.distribute` Causes Severe Precision Loss and Zero Reward Payouts

### Synopsis

`LiquidInfrastructureERC20` calculates holder reward payouts by first dividing the contract's ERC20 balance by `totalSupply()` in `_beginDistribution()` and then multiplying the truncated quotient `erc20EntitlementPerUnit[j]` by `this.balanceOf(recipient)` in `distribute()`. This division-before-multiplication truncates entitlements, resulting in zero payouts whenever `balance < totalSupply()` and leaving substantial rewards undistributed.

### Technical details

In `LiquidInfrastructureERC20`, revenue distributions occur in two steps:

1. When a distribution begins in `_beginDistribution()` (lines 270–277), the contract computes the per-unit entitlement for each reward token `distributableERC20s[i]` using unscaled integer division:
   ```solidity
   uint256 supply = this.totalSupply();
   for (uint i = 0; i < distributableERC20s.length; i++) {
       uint256 balance = IERC20(distributableERC20s[i]).balanceOf(
           address(this)
       );
       uint256 entitlement = balance / supply;
       erc20EntitlementPerUnit.push(entitlement);
   }
   ```
2. During `distribute()` (lines 220–227), each approved holder's payout is calculated by multiplying the pre-divided `erc20EntitlementPerUnit[j]` by the holder's token balance:
   ```solidity
   for (uint j = 0; j < distributableERC20s.length; j++) {
       IERC20 toDistribute = IERC20(distributableERC20s[j]);
       uint256 entitlement = erc20EntitlementPerUnit[j] *
           this.balanceOf(recipient);
       if (toDistribute.transfer(recipient, entitlement)) {
           receipts[j] = entitlement;
       }
   }
   ```

Because integer division `balance / supply` is performed before multiplying by `this.balanceOf(recipient)`—instead of computing `(balance * this.balanceOf(recipient)) / supply` or scaling `erc20EntitlementPerUnit` by a high-precision multiplier—severe truncation occurs in two common scenarios:

- **Reward balance smaller than `totalSupply()` (`balance < supply`):** `LiquidInfrastructureERC20` inherits `decimals() = 18` from OpenZeppelin's `ERC20`. If `totalSupply()` is `1,000e18` (`1,000` tokens), any reward token with fewer decimals (e.g., USDC or USDT with 6 decimals, WBTC with 8 decimals) will practically always have `balance < supply` (for example, `1,000,000e6` USDC is `1e12 < 1e21`). Similarly, even for an 18-decimal reward token, if the revenue collected for the period is `500e18` (`0.5` reward tokens per share token), `balance / supply` evaluates to `500e18 / 1,000e18 = 0`. Consequently, `erc20EntitlementPerUnit[j]` is `0`, and `entitlement` at lines 222–223 evaluates to `0` for every holder.
- **Reward balance not an exact multiple of `totalSupply()` (`balance % supply != 0`):** Even when `balance >= supply`, up to `supply - 1` wei of each reward token is discarded during the integer division. For example, if `supply = 1,000e18` and `balance = 1,999e18`, `erc20EntitlementPerUnit[j]` is `1`, so only `1,000e18` is distributed across all holders while `999e18` (nearly half of the accrued revenue) remains stuck in the contract until a future distribution.

### Assumptions and preconditions

- `LiquidInfrastructureERC20` has non-zero `totalSupply()` held by one or more approved holders.
- Either the contract's balance of a `distributableERC20s` token is not an exact multiple of `totalSupply()`, or `balance < totalSupply()` (which occurs naturally for 6- or 8-decimal reward tokens like USDC/USDT/WBTC when `LiquidInfrastructureERC20` is minted in standard 18-decimal units, or whenever epoch revenue is less than 1 unit per unit of `totalSupply()`).

### Proof of concept

1. The owner deploys `LiquidInfrastructureERC20` with 18 decimals and mints `500e18` tokens each to `HolderA` and `HolderB`, resulting in `totalSupply() = 1,000e18`.
2. `distributableERC20s[0]` is a 6-decimal USDC token (or an 18-decimal DAI token).
3. Managed NFTs generate `100,000e6` USDC ($100,000) and `900e18` DAI ($900) in revenue, which are withdrawn into `LiquidInfrastructureERC20` via `withdrawFromAllManagedNFTs()`.
4. After `MinDistributionPeriod` blocks elapse, `distributeToAllHolders()` is called:
   - `_beginDistribution()` computes:
     - For USDC: `erc20EntitlementPerUnit[0] = 100,000e6 / 1,000e18 = 1e11 / 1e21 = 0`.
     - For DAI: `erc20EntitlementPerUnit[1] = 900e18 / 1,000e18 = 0`.
   - In `distribute()` (lines 222–223), for both `HolderA` and `HolderB`:
     - `entitlement = erc20EntitlementPerUnit[j] * 500e18 = 0`.
   - `toDistribute.transfer(recipient, 0)` transfers `0` USDC and `0` DAI to both holders, even though each holder is entitled to `50,000e6` USDC and `450e18` DAI.
5. `_endDistribution()` completes the distribution and resets `LastDistribution = block.number`, leaving all `100,000e6` USDC and `900e18` DAI locked in the contract.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Holders receive zero rewards whenever a distributable token's balance in the contract is less than `totalSupply()`, and lose up to `totalSupply() - 1` smallest units of rewards to truncation in every distribution even when `balance >= totalSupply()`.
- **Business Implications:** Reward tokens with fewer than 18 decimals (such as USDC, USDT, or WBTC) cannot be distributed at all when `LiquidInfrastructureERC20` uses standard 18-decimal balances, and even 18-decimal reward tokens fail to distribute whenever period revenue is below 1:1 with `totalSupply()`. If a holder burns or transfers their tokens after a truncated distribution, they permanently lose their unclaimed share of the stuck revenue.

### Remediation

Store the snapshot of each distributable token's balance at the beginning of the distribution (or scale `erc20EntitlementPerUnit` by a precision multiplier such as `1e18`), and perform the multiplication by `this.balanceOf(recipient)` before dividing by `totalSupply()` in `distribute()`.

In `_beginDistribution()` and `distribute()`:

```solidity
    function distribute(uint256 numDistributions) public nonReentrant {
        require(numDistributions > 0, "must process at least 1 distribution");
        if (!LockedForDistribution) {
            require(
                _isPastMinDistributionPeriod(),
                "MinDistributionPeriod not met"
            );
            _beginDistribution();
        }

        uint256 limit = Math.min(
            nextDistributionRecipient + numDistributions,
            holders.length
        );

        uint256 supply = this.totalSupply();
        uint i;
        for (i = nextDistributionRecipient; i < limit; i++) {
            address recipient = holders[i];
            if (isApprovedHolder(recipient)) {
                uint256[] memory receipts = new uint256[](
                    distributableERC20s.length
                );
                uint256 recipientBalance = this.balanceOf(recipient);
                for (uint j = 0; j < distributableERC20s.length; j++) {
                    IERC20 toDistribute = IERC20(distributableERC20s[j]);
                    uint256 entitlement = (erc20EntitlementPerUnit[j] *
                        recipientBalance) / supply;
                    if (toDistribute.transfer(recipient, entitlement)) {
                        receipts[j] = entitlement;
                    }
                }

                emit Distribution(recipient, distributableERC20s, receipts);
            }
        }
        nextDistributionRecipient = i;

        if (nextDistributionRecipient == holders.length) {
            _endDistribution();
        }
    }

    function _beginDistribution() internal {
        require(
            !LockedForDistribution,
            "cannot begin distribution when already locked"
        );
        LockedForDistribution = true;

        if (erc20EntitlementPerUnit.length > 0) {
            delete erc20EntitlementPerUnit;
        }

        for (uint i = 0; i < distributableERC20s.length; i++) {
            uint256 balance = IERC20(distributableERC20s[i]).balanceOf(
                address(this)
            );
            erc20EntitlementPerUnit.push(balance);
        }

        nextDistributionRecipient = 0;
        emit DistributionStarted();
    }
```

## Finding 9: Eligibility-Dependent Entitlement Mismatch in `LiquidInfrastructureERC20.distribute` Leaves Disapproved Holders' Rewards Undistributed and Misallocated

### Synopsis

In `LiquidInfrastructureERC20.distribute`, reward payouts are restricted to approved holders (`isApprovedHolder(recipient)`), whereas `_beginDistribution` computes `erc20EntitlementPerUnit` using `totalSupply()`, which includes balances of disapproved holders. When `distribute` skips disapproved holders and finalizes via `_endDistribution`, their reward share remains stranded and is misallocated in subsequent distributions.

### Technical details

`LiquidInfrastructureERC20` maintains a KYC allowlist (`HolderAllowlist`) queried via `isApprovedHolder(address)`. An account that is initially approved via `approveHolder` can receive `LiquidInfrastructureERC20` tokens (increasing `totalSupply()` and adding the account to the `holders` array) and later be disapproved via `disapproveHolder`. Disapproving an account sets `HolderAllowlist[holder] = false`, which prevents the account from receiving additional `LiquidInfrastructureERC20` tokens in `_beforeTokenTransfer` (lines 133–138), but leaves the holder's existing token balance intact in `totalSupply()` and keeps the address in `holders`.

When a distribution starts in `_beginDistribution()` (lines 257–281), the reward entitlement per unit of `LiquidInfrastructureERC20` is calculated across the entire `totalSupply()`:

```solidity
uint256 supply = this.totalSupply();
for (uint i = 0; i < distributableERC20s.length; i++) {
    uint256 balance = IERC20(distributableERC20s[i]).balanceOf(
        address(this)
    );
    uint256 entitlement = balance / supply;
    erc20EntitlementPerUnit.push(entitlement);
}
```

However, in `distribute(uint256 numDistributions)` (lines 214–236), the contract only transfers entitlements to holders that currently pass `isApprovedHolder(recipient)`:

```solidity
for (i = nextDistributionRecipient; i < limit; i++) {
    address recipient = holders[i];
    if (isApprovedHolder(recipient)) {
        ...
        emit Distribution(recipient, distributableERC20s, receipts);
    }
}
nextDistributionRecipient = i;

if (nextDistributionRecipient == holders.length) {
    _endDistribution();
}
```

When `isApprovedHolder(recipient)` evaluates to `false`:
1. The loop body is skipped (closing at line 230) without transferring the `entitlement` reserved for `recipient`'s balance during `_beginDistribution()`.
2. `nextDistributionRecipient` still advances past `recipient` at line 232.
3. Once `nextDistributionRecipient == holders.length` at line 234, `_endDistribution()` is called at line 235, deleting `erc20EntitlementPerUnit`, unlocking the contract (`LockedForDistribution = false`), and updating `LastDistribution = block.number`.

This causes two payout accounting failures:
- **If disapproved holders are still entitled to revenue accrued by their existing shares** (as implied by `disapproveHolder`'s NatSpec: *"Marks `holder` as NOT approved to hold the token, preventing them from receiving any more of the underlying ERC20"*): skipping them in `distribute()` while advancing `nextDistributionRecipient` and calling `_endDistribution()` permanently deprives them of that epoch's distribution. Their unpaid share remains in the contract and is redistributed to other holders in subsequent epochs (for instance, after the disapproved holder burns their tokens via `burn()` or after new tokens are minted to other holders).
- **If disapproved holders are intended to be excluded from reward distributions**: including their token balances in `this.totalSupply()` during `_beginDistribution()` dilutes `erc20EntitlementPerUnit` for all approved holders. Approved holders receive less than 100% of the distributable revenue for the epoch, leaving the disapproved holders' proportional share stranded in the contract at `_endDistribution()`.

### Assumptions and preconditions

1. At least one approved holder holds `LiquidInfrastructureERC20` tokens.
2. Another holder who holds `LiquidInfrastructureERC20` tokens is disapproved via `disapproveHolder(holder)` prior to or before their index is processed in `distribute()`.
3. The contract holds a non-zero balance of at least one token in `distributableERC20s`, and `MinDistributionPeriod` blocks have elapsed since `LastDistribution`.

### Proof of concept

1. Suppose `HolderA` and `HolderB` are both approved via `approveHolder`, and each is minted `500` `LiquidInfrastructureERC20` tokens (`totalSupply() == 1000`).
2. `1,000e18` of a distributable reward token `ERC20A` is withdrawn from a managed NFT into `LiquidInfrastructureERC20`.
3. `HolderB` fails KYC renewal and the owner calls `disapproveHolder(HolderB)`. `HolderB` still holds `500` `LiquidInfrastructureERC20` tokens.
4. After `MinDistributionPeriod` blocks elapse, `distributeToAllHolders()` (calling `distribute(2)`) is executed:
   - `_beginDistribution()` computes `erc20EntitlementPerUnit[0] = 1,000e18 / 1,000 = 1e18` per token.
   - For `i = 0` (`HolderA`), `isApprovedHolder(HolderA)` is `true`, so `HolderA` receives `500 * 1e18 = 500e18` `ERC20A`.
   - For `i = 1` (`HolderB`), `isApprovedHolder(HolderB)` is `false`, so the `if` block (lines 216–230) is skipped and `HolderB` receives `0`.
   - At lines 232–235, `nextDistributionRecipient` becomes `2 == holders.length`, and `_endDistribution()` finishes the distribution with `500e18` `ERC20A` still remaining in `LiquidInfrastructureERC20`.
5. `HolderB` subsequently burns their `500` tokens (or gets re-approved after the distribution has already ended). In the next distribution after `MinDistributionPeriod` blocks, `HolderA` (now holding 100% of `totalSupply()` or sharing with newly minted holders) receives the `500e18` `ERC20A` that was withheld from `HolderB` in step 4.

### Impact assessment

- **Severity**: Medium / High
- **Technical Effect**: Entitlement calculation in `_beginDistribution` and payout execution/finalization in `distribute` are inconsistent whenever a holder in `holders` is disapproved. Either disapproved holders lose rewards allocated to their token balance (which are subsequently captured by other holders), or approved holders are underpaid in each distribution epoch because disapproved balances dilute `totalSupply()`.
- **Business Impact**: Directly violates the protocol's proportional revenue distribution invariant across token holders, leading to misallocation of real-world asset revenue among investors.

### Remediation

If `disapproveHolder` is only intended to prevent an account from receiving additional `LiquidInfrastructureERC20` transfers (while still paying out revenue on existing shares until burned or transferred out), remove the `isApprovedHolder(recipient)` check in `distribute()`.

Alternatively, if disapproved holders must not receive distributions, exclude disapproved holders' balances from the denominator in `_beginDistribution()` (by summing only approved holders' balances instead of `this.totalSupply()`) so that 100% of the distributable rewards are distributed among eligible approved holders.

#### Option 1: Pay existing holders proportionally regardless of subsequent disapproval

```solidity
        uint i;
        for (i = nextDistributionRecipient; i < limit; i++) {
            address recipient = holders[i];
            uint256[] memory receipts = new uint256[](
                distributableERC20s.length
            );
            for (uint j = 0; j < distributableERC20s.length; j++) {
                IERC20 toDistribute = IERC20(distributableERC20s[j]);
                uint256 entitlement = erc20EntitlementPerUnit[j] *
                    this.balanceOf(recipient);
                if (toDistribute.transfer(recipient, entitlement)) {
                    receipts[j] = entitlement;
                }
            }

            emit Distribution(recipient, distributableERC20s, receipts);
        }
        nextDistributionRecipient = i;

        if (nextDistributionRecipient == holders.length) {
            _endDistribution();
        }
```

## Finding 10: Unhandled External Token Failure and Lack of Recovery in `LiquidInfrastructureERC20.distribute` Causes Permanent Distribution Lockout

### Synopsis

In `LiquidInfrastructureERC20.distribute`, reward distributions iterate sequentially over holders and external ERC20 tokens without isolating external call reverts, bounding gas per recipient, or providing an emergency unlock mechanism. A single reverting or gas-exhausting transfer prevents `nextDistributionRecipient` from reaching `holders.length` and calling `_endDistribution()`, permanently locking the contract.

### Technical details

`LiquidInfrastructureERC20.distribute(uint256 numDistributions)` begins a distribution cycle by calling `_beginDistribution()`, which sets `LockedForDistribution = true` (line 262). While `LockedForDistribution` is `true`:
- All `LiquidInfrastructureERC20` transfers, mints, and burns revert in `_beforeTokenTransfer` (line 132: `require(!LockedForDistribution, "distribution in progress")`).
- All withdrawals from managed NFTs revert in `withdrawFromManagedNFTs` (line 360: `require(!LockedForDistribution, "cannot withdraw during distribution")`).

The contract only exits this locked state when `distribute()` finishes iterating through all `holders`, updates `nextDistributionRecipient = i` (line 232), and satisfies `if (nextDistributionRecipient == holders.length)` (line 234) to invoke `_endDistribution()` (line 235).

Within the distribution loop (lines 214–231), the contract attempts to transfer each distributable token to `recipient`:
```solidity
if (toDistribute.transfer(recipient, entitlement)) {
    receipts[j] = entitlement;
}
```
Although the `if` statement anticipates that a failed transfer returns `false` and allows execution to continue to `emit Distribution(...)` (line 229), advance `nextDistributionRecipient` (line 232), and eventually call `_endDistribution()` (line 235), this fails in practice for several reasons:

1. **External token failure propagation:** Most ERC20 implementations (including OpenZeppelin's `ERC20`) revert on failure instead of returning `false`. A revert can be triggered if a `recipient` is blacklisted on a reward token (e.g., USDC/USDT), if a token contract is paused, if a token reverts on zero-value transfers when `entitlement == 0`, or if the contract's token balance is insufficient (e.g., if `holders` contains duplicate entries or `setDistributableERC20s` was modified mid-distribution).
2. **Unbounded per-recipient processing costs:** If any token in `distributableERC20s` implements receiver callbacks (such as ERC777 `tokensReceived` hooks), a malicious or broken contract holder can consume all available gas or revert unconditionally during `toDistribute.transfer(recipient, entitlement)`.
3. **Unrecoverable distribution stalls and lockouts:** Because `distribute()` processes recipients strictly in index order (`i = nextDistributionRecipient; i < limit; i++`) and has no `try/catch` isolation, pull-based fallback, or administrative function to cancel/abort a stalled distribution, any revert during the loop aborts the transaction before `nextDistributionRecipient = i` (line 232) and `_endDistribution()` (line 235) can execute. If a multi-batch distribution is already in progress (`LockedForDistribution == true`), the contract remains permanently locked. Even if the revert occurs on the first batch, once `_isPastMinDistributionPeriod()` is `true`, all mints and burns are permanently blocked because a distribution cannot be completed.

### Assumptions and preconditions

1. `LiquidInfrastructureERC20` has at least one holder in `holders` and at least one token in `distributableERC20s`, and `MinDistributionPeriod` blocks have elapsed since `LastDistribution`.
2. At least one `toDistribute.transfer(recipient, entitlement)` call reverts or exhausts gas (for example, because `recipient` is blacklisted by a distributable ERC20 token, `recipient` is a contract that reverts in an ERC777 token hook, a distributable ERC20 is paused or reverts on zero-amount transfers, or the contract has insufficient token balance).

### Proof of concept

1. Suppose `holders` contains `[holderA, holderB, holderC]` and `distributableERC20s` contains `USDC`.
2. After `MinDistributionPeriod` blocks elapse, a caller invokes `distribute(1)`.
   - `_beginDistribution()` executes, setting `LockedForDistribution = true` and calculating `erc20EntitlementPerUnit`.
   - `holderA` (index 0) receives their entitlement, `nextDistributionRecipient` is updated to `1` at line 232, and the transaction succeeds. The contract remains locked (`LockedForDistribution == true`) because `nextDistributionRecipient (1) != holders.length (3)`.
3. Before or during the next step, `holderB` (index 1) is blacklisted by `USDC` (or `USDC` is paused, or `holderB` reverts in a token callback hook).
4. Any subsequent call to `distribute(1)` or `distributeToAllHolders()` attempts to process `holderB` at `i = 1`, calls `toDistribute.transfer(holderB, entitlement)` at line 224, and reverts.
5. Because `nextDistributionRecipient` cannot advance to `3` at line 232 and `_endDistribution()` at line 235 can never be reached, `LockedForDistribution` remains `true` permanently (or if the owner is unavailable to disapprove `holderB` / if the failure is at the token level rather than holder level, no recovery path exists), freezing all token transfers, mints, burns, and NFT withdrawals.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Permanent denial of service (liveness failure) of the `distribute()` state machine. Once a distribution stalls, `LockedForDistribution` cannot be cleared via `_endDistribution()`, permanently disabling `transfer`, `transferFrom`, `mint`, `burn`, `burnFrom`, and `withdrawFromManagedNFTs`.
- **Business Impact:** Legitimate token holders cannot transfer or burn their `LiquidInfrastructureERC20` shares, nor can they receive revenue distributions from the protocol's managed NFTs.

### Remediation

Isolate external token transfer failures using a low-level call with a bounded gas stipend (or `try/catch` with gas limits) and skip zero-amount transfers so that a single reverting token or recipient cannot block `nextDistributionRecipient` from advancing and reaching `_endDistribution()`:

```solidity
        uint i;
        for (i = nextDistributionRecipient; i < limit; i++) {
            address recipient = holders[i];
            if (isApprovedHolder(recipient)) {
                uint256[] memory receipts = new uint256[](
                    distributableERC20s.length
                );
                for (uint j = 0; j < distributableERC20s.length; j++) {
                    address toDistribute = distributableERC20s[j];
                    uint256 entitlement = erc20EntitlementPerUnit[j] *
                        this.balanceOf(recipient);
                    if (entitlement > 0) {
                        (bool success, bytes memory data) = toDistribute.call{
                            gas: 100000
                        }(
                            abi.encodeWithSelector(
                                IERC20.transfer.selector,
                                recipient,
                                entitlement
                            )
                        );
                        if (
                            success &&
                            (data.length == 0 || abi.decode(data, (bool)))
                        ) {
                            receipts[j] = entitlement;
                        }
                    }
                }

                emit Distribution(recipient, distributableERC20s, receipts);
            }
        }
        nextDistributionRecipient = i;

        if (nextDistributionRecipient == holders.length) {
            _endDistribution();
        }
```

Alternatively, adopt a pull-based reward claiming pattern so individual recipient transfer failures are isolated to the claiming user.
