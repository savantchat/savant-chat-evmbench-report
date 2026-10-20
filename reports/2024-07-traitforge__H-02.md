# Audit report — 2024-07-traitforge/H-02

## Finding 1: Use of `tx.origin` in `Airdrop.startAirdrop` Prevents Multisig Owner from Funding and Starting the Airdrop

### Synopsis

`Airdrop.startAirdrop` uses `tx.origin` as the token payer in `traitToken.transferFrom(tx.origin, address(this), amount)`. Because `TraitForgeNft` is owned by a multisig contract and owns `Airdrop`, `tx.origin` resolves to the executing EOA rather than the multisig, causing `startAirdrop` to fail or pull funds from the wrong account.

### Technical details

In the protocol architecture, ownership of the `Airdrop` contract is transferred to the `TraitForgeNft` contract during deployment (`airdrop.transferOwnership(await nft.getAddress())`), while `TraitForgeNft` itself is owned by the protocol's multisig wallet.

To initiate the airdrop, the multisig owner calls `TraitForgeNft.startAirdrop(uint256 amount)` (lines 96–98 of `TraitForgeNft.sol`), which forwards the call to `Airdrop.startAirdrop(uint256 amount)` (lines 24–32 of `Airdrop.sol`):

```solidity
function startAirdrop(
  uint256 amount
) external whenNotPaused nonReentrant onlyOwner {
  require(!started, 'Already started');
  require(amount > 0, 'Invalid amount');
  traitToken.transferFrom(tx.origin, address(this), amount);
  started = true;
  totalTokenAmount = amount;
}
```

Because `Airdrop.onlyOwner` restricts the direct caller (`msg.sender`) to `TraitForgeNft`, the developer used `tx.origin` on line 29 to bypass the intermediate `TraitForgeNft` contract and pull `traitToken` from the initiator of the transaction. However, this assumes the owner of `TraitForgeNft` is an Externally Owned Account (EOA).

When the owner of `TraitForgeNft` is a multisig smart contract (or any smart contract wallet / account abstraction wallet):
1. The multisig holds the `traitToken` treasury and approves `Airdrop` to spend `amount`.
2. When the multisig executes `TraitForgeNft.startAirdrop(amount)`, `msg.sender` in `TraitForgeNft` is the multisig address, `msg.sender` in `Airdrop` is `TraitForgeNft`, and `tx.origin` is the EOA of the final multisig signer or relayer who broadcasted the outer transaction.
3. `traitToken.transferFrom(tx.origin, address(this), amount)` attempts to transfer tokens from the EOA (`tx.origin`) instead of the multisig contract. Unless the individual signer/relayer EOA personally holds the full `amount` of `traitToken` and has personally approved `Airdrop`, the call reverts with `ERC20: insufficient allowance` or `ERC20: transfer amount exceeds balance`.

Furthermore, using `tx.origin` for token transfers is a well-known anti-pattern that breaks smart contract composability and exposes any EOA that has approved `Airdrop` to phishing if induced to execute a transaction while `TraitForgeNft` is invoked.

### Assumptions and preconditions

1. `Airdrop` is owned by `TraitForgeNft`, as configured in the deployment scripts.
2. `TraitForgeNft` is owned by a multisig contract (or smart contract wallet), as specified by the protocol team.
3. The `traitToken` allocation intended for the airdrop is held by and approved from the multisig wallet rather than an individual signer's EOA.

### Proof of concept

1. The protocol is deployed via `deploy-all`: `Airdrop` is owned by `TraitForgeNft`, and `TraitForgeNft` ownership is held by the protocol's `Multisig` contract.
2. `Multisig` holds `1,000,000 TRAIT` tokens and executes `traitToken.approve(airdropAddress, 1000000e18)`.
3. The multisig signers sign a transaction for `Multisig` to call `TraitForgeNft.startAirdrop(1000000e18)`.
4. Signer A (an EOA with `0 TRAIT` tokens and `0` allowance to `Airdrop`) broadcasts the multisig execution transaction.
5. `Multisig` calls `TraitForgeNft.startAirdrop(1000000e18)`, which calls `Airdrop.startAirdrop(1000000e18)`.
6. Inside `Airdrop.startAirdrop`, `traitToken.transferFrom(tx.origin, address(this), 1000000e18)` is called where `tx.origin` is Signer A's EOA address.
7. Because Signer A has not approved `Airdrop` (and does not hold the tokens), `ERC20._spendAllowance(SignerA, Airdrop, 1000000e18)` reverts with `"ERC20: insufficient allowance"`. The multisig cannot start the airdrop using its own funds.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: `Airdrop.startAirdrop` reverts when called by a multisig or smart contract owner whose treasury holds and approves the tokens, breaking smart contract composability between `TraitForgeNft` and `Airdrop`.
- **Business Impact**: The protocol multisig cannot directly fund and start the airdrop from its treasury; it would be forced to transfer the entire airdrop token supply to an untrusted or single-point-of-failure EOA (the transaction broadcaster) to complete `startAirdrop`, defeating the security guarantees of using a multisig.

### Remediation

Pass the payer address (the `msg.sender` of `TraitForgeNft.startAirdrop`) explicitly from `TraitForgeNft` to `Airdrop.startAirdrop`, or transfer the tokens via `msg.sender` so that `TraitForgeNft` (or `Airdrop` directly using the passed owner address) pulls the tokens from the multisig rather than `tx.origin`. Alternatively, pull tokens from `Ownable(msg.sender).owner()` inside `Airdrop.startAirdrop` so the interface does not need to change:

```solidity
  function startAirdrop(
    uint256 amount
  ) external whenNotPaused nonReentrant onlyOwner {
    require(!started, 'Already started');
    require(amount > 0, 'Invalid amount');
    traitToken.transferFrom(Ownable(msg.sender).owner(), address(this), amount);
    started = true;
    totalTokenAmount = amount;
  }
```

## Finding 2: Unreachable `Airdrop.allowDaoFund` under `TraitForgeNft` ownership permanently blocks DAO fund distribution in `NukeFund`

### Synopsis

`Airdrop` ownership is transferred to `TraitForgeNft` upon deployment so `TraitForgeNft` can record user entropy and start the airdrop. However, `Airdrop.allowDaoFund()` is restricted to `onlyOwner` while `TraitForgeNft` lacks any function to invoke it, permanently preventing `daoAllowed` from becoming `true` and blocking `NukeFund` distributions to `daoAddress`.

### Technical details

The `Airdrop` contract inherits from OpenZeppelin's `Ownable` and restricts administrative and accounting functions using the `onlyOwner` modifier:
- `setTraitToken(address)` (`Airdrop.sol:20`)
- `startAirdrop(uint256)` (`Airdrop.sol:24`)
- `allowDaoFund()` (`Airdrop.sol:38`)
- `addUserAmount(address, uint256)` (`Airdrop.sol:48`)
- `subUserAmount(address, uint256)` (`Airdrop.sol:57`)

During protocol deployment (`scripts/deployTasks.ts`), ownership of the `Airdrop` contract is transferred to the `TraitForgeNft` contract (`airdrop.transferOwnership(await nft.getAddress())`). This is required because `TraitForgeNft` directly calls `airdropContract.addUserAmount(...)` during `_mintInternal` (`TraitForgeNft.sol:297`) and `_mintNewEntity` (`TraitForgeNft.sol:338`), and calls `airdropContract.subUserAmount(...)` during `burn` (`TraitForgeNft.sol:148`).

To allow the protocol owner to initiate the airdrop after ownership of `Airdrop` has been transferred to `TraitForgeNft`, `TraitForgeNft` implements a pass-through wrapper (`TraitForgeNft.sol:96-98`):

```solidity
function startAirdrop(uint256 amount) external whenNotPaused onlyOwner {
  airdropContract.startAirdrop(amount);
}
```

However, `TraitForgeNft` neither implements a wrapper function to call `airdropContract.allowDaoFund()` nor provides any function to transfer `Airdrop` ownership to another address. Additionally, the deployer cannot call `airdrop.allowDaoFund()` prior to transferring ownership of `Airdrop` to `TraitForgeNft`, because `allowDaoFund()` enforces `require(started, 'Not started')` (`Airdrop.sol:39`), and `started` cannot be set to `true` before NFTs are minted and user airdrop shares are accumulated (`addUserAmount` and `subUserAmount` both require `!started`).

As a result, `Airdrop.allowDaoFund()` is completely unreachable once the contracts are deployed and configured. `daoAllowed` (`airdropContract.daoFundAllowed()`) remains permanently `false`, which causes `NukeFund.receive()` (`NukeFund.sol:50-57`) to always take the `else if (!airdropContract.daoFundAllowed())` branch after the airdrop starts and never forward `devShare` to `daoAddress`.

### Assumptions and preconditions

- The protocol is deployed and configured as specified in `scripts/deployTasks.ts` and the test suite (`NukeFund.test.ts`, `TraitForgeNft.test.ts`), where ownership of `Airdrop` is transferred to `TraitForgeNft`.
- The airdrop has not been started prior to transferring `Airdrop` ownership to `TraitForgeNft` (which is required so users can mint NFTs and accumulate `userInfo` balances before the airdrop starts).

### Proof of concept

1. The deployer deploys `TraitForgeNft`, `Airdrop`, and `NukeFund` according to `scripts/deployTasks.ts`.
2. The deployer calls `airdrop.setTraitToken(token)` and `airdrop.transferOwnership(address(nft))` so `TraitForgeNft` can record user entropy via `addUserAmount` and `subUserAmount` during minting, forging, and burning.
3. Users mint entities, and `TraitForgeNft` calls `airdropContract.addUserAmount(to, entropyValue)`.
4. The owner of `TraitForgeNft` calls `nft.startAirdrop(amount)`, which calls `airdropContract.startAirdrop(amount)` as the owner of `Airdrop`, setting `started = true`.
5. The protocol owner attempts to enable DAO funding by calling `airdrop.allowDaoFund()`. The call reverts with `"Ownable: caller is not the owner"` because `airdrop.owner()` is `address(nft)`.
6. Because `TraitForgeNft` has no function that calls `airdropContract.allowDaoFund()`, `airdrop.daoFundAllowed()` remains `false` forever, and all subsequent ETH sent to `NukeFund` sends `devShare` to `NukeFund.owner()` instead of `daoAddress`.

### Impact assessment

- **Severity**: Medium / High
- **Technical Effect**: `Airdrop.allowDaoFund()` is permanently uncallable in production, and `Airdrop.daoFundAllowed()` is permanently stuck at `false`.
- **Business Impact**: The third phase of fee distribution in `NukeFund.receive()`—routing the 10% tax share (`devShare`) to `daoAddress` (`DAOFund`) after DAO funding is enabled—can never be activated. The DAO fund is completely starved of its intended ETH revenue from `NukeFund`.

### Remediation

Add a wrapper function in `TraitForgeNft` restricted to `onlyOwner` that forwards the call to `airdropContract.allowDaoFund()`, matching the pattern used for `startAirdrop`, or separate the `TraitForgeNft` role from the `Ownable` owner in `Airdrop`.

In `TraitForgeNft.sol`:
```solidity
  function allowDaoFund() external whenNotPaused onlyOwner {
    airdropContract.allowDaoFund();
  }
```

Alternatively, in `Airdrop.sol`, keep the protocol admin as `owner()` and introduce a dedicated modifier/address for `TraitForgeNft` to call `addUserAmount` and `subUserAmount`:
```solidity
  address public nftContract;

  modifier onlyNft() {
    require(msg.sender == nftContract, 'Unauthorized');
    _;
  }

  function setNftContract(address _nftContract) external onlyOwner {
    nftContract = _nftContract;
  }
```

## Finding 3: Entitlement Attribution Mismatch Across Minting, Transfers, and Burns in `Airdrop` and `TraitForgeNft`

### Synopsis

`Airdrop.addUserAmount` and `Airdrop.subUserAmount` track per-user airdrop weights (`userInfo`) and `totalValue` for `Airdrop.claim`, but `TraitForgeNft` attributes mint weights to `initialOwners[tokenId]` without updating entitlements on token transfers, causing `TraitForgeNft.burn` to deduct airdrop weight from the original minter when a subsequent owner burns the token.

### Technical details

In `Airdrop.sol`, user airdrop entitlements and total weight are managed through `addUserAmount` (lines 48–55) and `subUserAmount` (lines 57–65), and later used in `claim` (lines 67–71) to calculate a user's claimable share:

```solidity
uint256 amount = (totalTokenAmount * userInfo[msg.sender]) / totalValue;
```

`Airdrop` is owned exclusively by `TraitForgeNft`, which calls `addUserAmount` and `subUserAmount` during the NFT lifecycle before the airdrop starts (`!airdropContract.airdropStarted()`):

1. **Minting (`_mintInternal` and `_mintNewEntity`):** When a token is minted or forged, `TraitForgeNft` records `initialOwners[tokenId] = to` (lines 294 and 329) and credits the initial owner's airdrop weight via `airdropContract.addUserAmount(to, entropyValue)` (lines 297 and 338), increasing `userInfo[to]` and `totalValue` in `Airdrop.sol` (lines 53–54).
2. **Transfers (`_beforeTokenTransfer`):** When a token is transferred to a new owner (lines 367–395), `TraitForgeNft` does not transfer the token's `entropy` weight in `Airdrop` from `from` to `to`, nor does `Airdrop` track ownership changes per token.
3. **Burning (`burn`):** When `TraitForgeNft.burn(tokenId)` (lines 141–151) is called by the current token owner or an approved address while `!airdropContract.airdropStarted()`, it executes:
   ```solidity
   uint256 entropy = getTokenEntropy(tokenId);
   airdropContract.subUserAmount(initialOwners[tokenId], entropy);
   ```
   This invokes `Airdrop.subUserAmount` (lines 57–65), which subtracts `entropy` from `userInfo[initialOwners[tokenId]]` (line 63) and `totalValue` (line 64).

This creates a direct accounting and entitlement attribution mismatch across minting, transfers, and burns:
- If airdrop entitlements are meant to belong to the current NFT holder (which is why burning an NFT before the airdrop starts revokes its `entropy` weight from `Airdrop`), transferring an NFT fails to move the `entropy` weight from the sender to the recipient, and burning the transferred NFT penalizes `initialOwners[tokenId]` instead of the burner (`ownerOf(tokenId)`).
- If airdrop entitlements are meant to permanently reward the initial minter (`initialOwners[tokenId]`), allowing any subsequent buyer or transferee to call `burn(tokenId)` and trigger `Airdrop.subUserAmount(initialOwners[tokenId], entropy)` enables the new owner to arbitrarily destroy the original minter's `userInfo` balance before the airdrop starts.

### Assumptions and preconditions

1. The airdrop has not started yet (`airdropContract.airdropStarted() == false`).
2. User A mints or forges a `TraitForgeNft` token, receiving `entropy` weight in `Airdrop.userInfo[User A]`.
3. User A transfers or sells the NFT to User B before the airdrop starts.
4. User B calls `TraitForgeNft.burn(tokenId)` before the airdrop starts.

### Proof of concept

1. While `started == false` on `Airdrop`, User A mints `tokenId = 1` with `entropy = 500,000` via `TraitForgeNft.mintToken`.
   - `TraitForgeNft.initialOwners[1]` is set to `User A`.
   - `Airdrop.addUserAmount(User A, 500,000)` sets `userInfo[User A] = 500,000` and increases `totalValue` by `500,000`.
2. User A sells or transfers `tokenId = 1` to User B.
   - Ownership of `tokenId = 1` is now `User B`, while `initialOwners[1]` remains `User A` and `userInfo[User A]` remains `500,000`.
3. Before `startAirdrop` is called, User B calls `TraitForgeNft.burn(1)`.
   - `burn(1)` checks `isApprovedOrOwner(User B, 1)` (which passes) and calls `airdropContract.subUserAmount(initialOwners[1], 500,000)`.
   - `Airdrop.subUserAmount` executes `userInfo[User A] -= 500,000` and `totalValue -= 500,000`.
4. When the airdrop starts and User A attempts to call `Airdrop.claim()`, `userInfo[User A]` is `0`, so the call reverts with `'Not eligible'`, depriving User A of the airdrop entitlement despite having sold the token rather than burning it.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** `Airdrop.userInfo` and `totalValue` become desynchronized from actual token ownership across transfers and burns. A subsequent token owner who burns an NFT reduces the `userInfo` balance of the token's `initialOwners[tokenId]` rather than their own weight, or conversely, token buyers do not receive the `Airdrop` weight associated with the NFTs they hold unless entitlements are transferred on `_beforeTokenTransfer`.
- **Business Impact:** Original minters who sell or transfer their NFTs on the secondary market (`EntityTrading`) can have their accrued airdrop rewards wiped out if a subsequent holder burns the NFT prior to `startAirdrop`, leading to unfair loss of `traitToken` rewards during `Airdrop.claim()`.

### Remediation

Ensure entitlement attribution is consistent across minting, transfers, and burns. Specifically:
- If airdrop weight belongs to the current token holder so that burning removes weight, transfer the token's `entropy` weight from `from` to `to` during token transfers (by moving `entropy` between `userInfo[from]` and `userInfo[to]`) and deduct from the current token owner on `burn`.
- Alternatively, if airdrop weight is strictly a non-transferable minting reward for `initialOwners[tokenId]`, do not call `subUserAmount(initialOwners[tokenId], entropy)` when a token is burned, or only allow `subUserAmount` when burned by `initialOwners[tokenId]`.

To keep `userInfo` aligned with token ownership across transfers and burns:

```solidity
  function addUserAmount(
    address user,
    uint256 amount
  ) external whenNotPaused nonReentrant onlyOwner {
    require(!started, 'Already started');
    userInfo[user] += amount;
    totalValue += amount;
  }

  function subUserAmount(
    address user,
    uint256 amount
  ) external whenNotPaused nonReentrant onlyOwner {
    require(!started, 'Already started');
    require(userInfo[user] >= amount, 'Invalid amount');
    userInfo[user] -= amount;
    totalValue -= amount;
  }

  function transferUserAmount(
    address from,
    address to,
    uint256 amount
  ) external whenNotPaused nonReentrant onlyOwner {
    require(!started, 'Already started');
    require(userInfo[from] >= amount, 'Invalid amount');
    userInfo[from] -= amount;
    userInfo[to] += amount;
  }

  function claim() external whenNotPaused nonReentrant {
    require(started, 'Not started');
    require(userInfo[msg.sender] > 0, 'Not eligible');

    uint256 amount = (totalTokenAmount * userInfo[msg.sender]) / totalValue;
    userInfo[msg.sender] = 0;
    traitToken.transfer(msg.sender, amount);
  }
```

And in `TraitForgeNft.burn`, deduct from the current token owner (`ownerOf(tokenId)`) while moving weight on non-mint/non-burn transfers in `_beforeTokenTransfer`.

## Finding 4: Inconsistent Token Maturity and Age Tracking in `NukeFund.nuke` Causes Mismatched Holding-Period Eligibility and Nuke Factor Accrual

### Synopsis

In `NukeFund.nuke`, token maturity verification (`canTokenBeNuked`) and reward calculation (`calculateNukeFactor` via `calculateAge`) rely on inconsistent lifecycle timestamps (`getTokenLastTransferredTimestamp` versus `getTokenCreationTimestamp`) and approval-based execution checks, causing mismatches between an owner's actual holding period, nuke eligibility, and accrued `finalNukeFactor` claim amounts.

### Technical details

When a user invokes `NukeFund.nuke(uint256 tokenId)` (`contracts/NukeFund/NukeFund.sol`, lines 153–166), the contract performs two maturity-dependent operations to gate execution and compute the ETH payout from `fund`:

1. **Maturity Gate (`163`)**:
   ```solidity
   require(canTokenBeNuked(tokenId), 'Token is not mature yet');
   ```
   `canTokenBeNuked(tokenId)` (lines 184–194) computes the holding duration using `nftContract.getTokenLastTransferredTimestamp(tokenId)`:
   ```solidity
   uint256 tokenAgeInSeconds = block.timestamp -
     nftContract.getTokenLastTransferredTimestamp(tokenId);
   return tokenAgeInSeconds >= minimumDaysHeld;
   ```

2. **Nuke Factor & Claim Calculation (`165–166`)**:
   ```solidity
   uint256 finalNukeFactor = calculateNukeFactor(tokenId);
   uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR;
   ```
   `calculateNukeFactor(tokenId)` (lines 136–151) delegates to `calculateAge(tokenId)` (lines 118–133), which computes token age using `nftContract.getTokenCreationTimestamp(tokenId)`:
   ```solidity
   uint256 daysOld = (block.timestamp -
     nftContract.getTokenCreationTimestamp(tokenId)) /
     60 /
     60 /
     24;
   ```

This creates a state integrity mismatch across the token lifecycle:
- **Holding-Period vs. Accrual-Age Mismatch on Ownership Transition**: The protocol intends for `nukeFactor` to reward the owner based on the time they held the NFT, and `canTokenBeNuked` resets eligibility upon transfer via `lastTokenTransferredTimestamp`. However, `calculateAge` uses `getTokenCreationTimestamp(tokenId)` rather than `getTokenLastTransferredTimestamp(tokenId)`. A user who acquires an old NFT only waits `minimumDaysHeld` (default 3 days) to pass `canTokenBeNuked(tokenId)` at line 163, but at lines 165–166 claims a `finalNukeFactor` and `potentialClaimAmount` accrued over the entire time since the token's original creation timestamp.
- **Approval-Based Ownership Transition Bypass (`154–163`)**: Because `nuke` permits any approved caller (`nftContract.isApprovedOrOwner(msg.sender, tokenId)` and `nftContract.isApprovedForAll(msg.sender, address(this))`) to burn the NFT and receive the `claimAmount` directly (`payable(msg.sender).call{ value: claimAmount }('')`), a token holder can transfer effective nuke rights to a buyer via ERC721 approval without executing an ERC721 transfer. Because `lastTokenTransferredTimestamp` is not updated on approval, `canTokenBeNuked(tokenId)` at line 163 is immediately satisfied without enforcing `minimumDaysHeld` for the new beneficiary.
- **Mint Lifecycle Timestamp Initialization**: In `TraitForgeNft._mintInternal` (lines 280–309) and `TraitForgeNft._mintNewEntity` (lines 311–343), `tokenCreationTimestamps[tokenId] = block.timestamp` is explicitly initialized, whereas `lastTokenTransferredTimestamp[tokenId]` is not set in the mint functions themselves. If `lastTokenTransferredTimestamp[tokenId]` remains `0` on mint, `block.timestamp - 0 >= minimumDaysHeld` in `canTokenBeNuked(tokenId)` evaluates to `true` immediately at mint time, bypassing `minimumDaysHeld` at line 163.

### Assumptions and preconditions

- `NukeFund` and `TraitForgeNft` are deployed and unpaused, and `NukeFund` holds a non-zero ETH balance in `fund`.
- `minimumDaysHeld` is configured to a non-zero duration (e.g., default `3 days`) and `ageMultiplier` is configured so tokens accrue age over time.
- A token exists with an older `tokenCreationTimestamps[tokenId]` than its current holder's holding duration, or is nuked via an approved caller / immediately after minting.

### Proof of concept

1. Alice mints `tokenId = 1` at `t = 0` and holds it for 180 days (`tokenCreationTimestamps[1] = t_0`).
2.Option A (Direct Transfer): At `t = 180 days`, Alice transfers `tokenId = 1` to Bob. `lastTokenTransferredTimestamp[1]` updates to `t = 180 days`. Bob waits only `minimumDaysHeld` (3 days, until `t = 183 days`). When Bob calls `nukeFund.nuke(1)`:
   - `canTokenBeNuked(1)` at line 163 checks `block.timestamp - lastTokenTransferredTimestamp[1] == 3 days >= minimumDaysHeld` and returns `true`.
   - `calculateNukeFactor(1)` at line 165 calls `calculateAge(1)`, which computes `daysOld` using `tokenCreationTimestamps[1]` (`183 days` instead of Bob's `3 days` holding period), granting Bob the full 183-day `potentialClaimAmount` at line 166.
3. Option B (Approval Bypass): At `t = 180 days`, instead of transferring `tokenId = 1`, Alice approves Bob (`nftContract.approve(bob, 1)`), and Bob calls `nftContract.setApprovalForAll(address(nukeFund), true)` and `nukeFund.nuke(1)` in the same block:
   - Line 155 `nftContract.isApprovedOrOwner(bob, 1)` passes.
   - Line 160 `nftContract.isApprovedForAll(bob, address(nukeFund))` passes.
   - Line 163 `canTokenBeNuked(1)` passes immediately with zero holding period for Bob because `lastTokenTransferredTimestamp[1]` was never updated, and Bob receives the full `claimAmount`.

### Impact assessment

- **Severity**: Medium / High
- **Technical Impact**: The holding-period lock (`minimumDaysHeld`) and holding-duration reward accrual in `NukeFund` are desynchronized and can be bypassed. New owners can claim full creation-age rewards after only `minimumDaysHeld`, or bypass `minimumDaysHeld` entirely via approval-based nuking.
- **Business Impact**: Premature and inflated ETH withdrawals from `NukeFund` deplete the honeypot pool at the expense of long-term NFT holders.

### Remediation

Ensure that `nuke` enforces ownership and maturity consistently:
1. Require the token owner to approve `NukeFund` (checking `isApprovedForAll(nftContract.ownerOf(tokenId), address(this))`) and send payouts or enforce holding periods strictly with respect to the token owner's actual holding timestamp.
2. Use consistent timestamp tracking (`getTokenLastTransferredTimestamp(tokenId)`) for both maturity eligibility (`canTokenBeNuked`) and age accrual (`calculateAge`), while ensuring `lastTokenTransferredTimestamp` is always initialized at mint time.

```solidity
  function calculateAge(uint256 tokenId) public view returns (uint256) {
    require(nftContract.ownerOf(tokenId) != address(0), 'Token does not exist');

    uint256 daysOld = (block.timestamp -
      nftContract.getTokenLastTransferredTimestamp(tokenId)) /
      60 /
      60 /
      24;
    uint256 perfomanceFactor = nftContract.getTokenEntropy(tokenId) % 10;

    uint256 age = (daysOld *
      perfomanceFactor *
      MAX_DENOMINATOR *
      ageMultiplier) / 365; // add 5 digits for decimals
    return age;
  }

  function nuke(uint256 tokenId) public whenNotPaused nonReentrant {
    address tokenOwner = nftContract.ownerOf(tokenId);
    require(
      nftContract.isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    require(
      nftContract.getApproved(tokenId) == address(this) ||
        nftContract.isApprovedForAll(tokenOwner, address(this)),
      'Contract must be approved to transfer the NFT.'
    );
    require(canTokenBeNuked(tokenId), 'Token is not mature yet');

    uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
    uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
```

## Finding 5: Broken Delegated Approval Validation and Operator Authority Conflation in `NukeFund.nuke`

### Synopsis

In `NukeFund.nuke`, the authorization checks conflate the caller (`msg.sender`) with the token owner (`ownerOf(tokenId)`). Specifically, `nuke` checks `nftContract.isApprovedForAll(msg.sender, address(this))` instead of checking the token owner's approval, creating a cross-contract approval mismatch with `TraitForgeNft.burn` and mismanaging delegated redemption authority.

### Technical details

In `NukeFund.sol`, `nuke(uint256 tokenId)` validates caller and contract permissions across lines 154–162 before burning the token via `nftContract.burn(tokenId)` at line 176 and transferring the ETH claim proceeds to `payable(msg.sender)` at line 177:

```solidity
  function nuke(uint256 tokenId) public whenNotPaused nonReentrant {
    require(
      nftContract.isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    require(
      nftContract.getApproved(tokenId) == address(this) ||
        nftContract.isApprovedForAll(msg.sender, address(this)),
      'Contract must be approved to transfer the NFT.'
    );
```

This authorization logic is flawed due to a mismatch between ERC721's approval model and `NukeFund`'s checks:

1. **Owner vs. Caller Approval Scope Conflation (Line 160):**
   Line 160 checks `nftContract.isApprovedForAll(msg.sender, address(this))`. However, when `nuke` subsequently calls `nftContract.burn(tokenId)` at line 176, `TraitForgeNft.burn` executes `require(isApprovedOrOwner(msg.sender, tokenId))` where `msg.sender` is `address(nukeFund)`. Inside OpenZeppelin's `ERC721._isApprovedOrOwner(spender, tokenId)`, the check evaluates:
   ```solidity
   address owner = ERC721.ownerOf(tokenId);
   return (spender == owner || isApprovedForAll(owner, spender) || getApproved(tokenId) == spender);
   ```
   `TraitForgeNft.burn` therefore requires `isApprovedForAll(owner, address(nukeFund))` or `getApproved(tokenId) == address(nukeFund)`, whereas `NukeFund.nuke` checks `isApprovedForAll(msg.sender, address(nukeFund))`.

2. **Single-Slot Token Approval Conflict & Delegated Execution Failure:**
   In ERC721, `getApproved(tokenId)` stores only a single address per token.
   - If a token owner approves an address `spender` via `approve(spender, tokenId)`, `getApproved(tokenId)` becomes `spender` (overwriting any prior approval for `address(nukeFund)`). When `spender` calls `nuke(tokenId)`,even if `spender` sets `setApprovalForAll(address(nukeFund), true)` to pass line 160, the call to `nftContract.burn(tokenId)` will revert unless the token owner has separately granted global `setApprovalForAll(address(nukeFund), true)`.
   - Conversely, if the token owner has granted `isApprovedForAll(owner, address(nukeFund))` (e.g., to interact with `NukeFund`) or `approve(address(nukeFund), tokenId)`, any operator or single-token approved spender (`msg.sender != owner`) can bypass line 160 simply by calling `nftContract.setApprovalForAll(address(nukeFund), true)` on their own account, burn the owner's NFT, and divert the ETH payout to `msg.sender` (line 177).

### Assumptions and preconditions

- A valid token `tokenId` exists in `TraitForgeNft` and satisfies `canTokenBeNuked(tokenId)` (held for at least `minimumDaysHeld`).
- Neither `NukeFund` nor `TraitForgeNft` is paused.
- Either:
  - An approved spender/operator attempts to nuke a token on behalf of the owner when the owner has not called `setApprovalForAll(address(nukeFund), true)` (resulting in an unexpected revert in `TraitForgeNft.burn`), or
  - The token owner has approved `NukeFund` (`approve(address(nukeFund), tokenId)` or `setApprovalForAll(address(nukeFund), true)`) and has an approved operator/spender who calls `nuke(tokenId)` to claim the ETH proceeds for themselves.

### Proof of concept

#### Scenario 1: Delegated Spender/Operator Blocked by Approval Scope Mismatch
1. `alice` owns `tokenId = 1` and approves `bob` to manage `tokenId = 1` via `nftContract.approve(bob, 1)` (or `nftContract.setApprovalForAll(bob, true)`).
2. `bob` attempts to nuke `tokenId = 1` by calling `nukeFund.nuke(1)`.
3. Even if `bob` calls `nftContract.setApprovalForAll(address(nukeFund), true)` so that `nftContract.isApprovedForAll(bob, address(nukeFund))` is `true`:
   - `nftContract.isApprovedOrOwner(bob, 1)` returns `true`.
   - `nftContract.isApprovedForAll(bob, address(nukeFund))` returns `true` (passing line 160).
   - `nftContract.burn(1)` is called by `nukeFund`.
   - Inside `TraitForgeNft.burn(1)`, `isApprovedOrOwner(address(nukeFund), 1)` checks `isApprovedForAll(alice, address(nukeFund))` and `getApproved(1) == address(nukeFund)`. Both are `false` (since `getApproved(1)` is `bob`, and `alice` cannot simultaneously set `getApproved(1)` to both `bob` and `nukeFund`).
   - The transaction reverts with `'ERC721: caller is not token owner or approved'`.

#### Scenario 2: Operator/Spender Bypasses Contract Approval Check and Claims Owner's Nuke Payout
1. `alice` owns `tokenId = 1` and `tokenId = 2`. `alice` calls `nftContract.setApprovalForAll(address(nukeFund), true)` so she can nuke her tokens, or `nftContract.approve(address(nukeFund), 1)`.
2. `alice` also approves `bob` on `tokenId = 2` via `nftContract.approve(bob, 2)` (e.g., for a third-party marketplace or lending contract) or as an operator.
3. `bob` calls `nftContract.setApprovalForAll(address(nukeFund), true)` from his own address (`msg.sender = bob`).
4. `bob` calls `nukeFund.nuke(2)`:
   - Line 155 passes because `bob` is approved for `tokenId = 2`.
   - Line 160 passes because `nftContract.isApprovedForAll(bob, address(nukeFund))` is `true`.
   - Line 176 burns `alice`'s `tokenId = 2` (succeeding because `alice` had `setApprovalForAll(address(nukeFund), true)`).
   - Line 177 transfers the full ETH `claimAmount` to `bob` (`msg.sender`) instead of `alice`.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** `NukeFund.nuke` checks `isApprovedForAll(msg.sender, address(this))` instead of verifying the token owner's approval or restricting redemption appropriately. Delegated single-token approvals (`approve(spender, tokenId)`) cannot be used to nuke a token without global owner `setApprovalForAll(nukeFund, true)`, and when `nukeFund` is approved by the owner, any approved spender/operator can satisfy line 160 using their own `setApprovalForAll` state, burn the owner's NFT, and extract the ETH payout to `msg.sender`.

### Remediation

Check the token owner's approval in `isApprovedForAll` (using `nftContract.ownerOf(tokenId)`) rather than `msg.sender`, or require `msg.sender` to be the token owner directly if delegated callers should not receive the owner's nuke payout:

```solidity
  function nuke(uint256 tokenId) public whenNotPaused nonReentrant {
    address tokenOwner = nftContract.ownerOf(tokenId);
    require(
      nftContract.isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    require(
      nftContract.getApproved(tokenId) == address(this) ||
        nftContract.isApprovedForAll(tokenOwner, address(this)),
      'Contract must be approved to transfer the NFT.'
    );
    require(canTokenBeNuked(tokenId), 'Token is not mature yet');

    uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
    uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
    uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

    // Directly assign the value to claimAmount based on the condition, removing the redeclaration
    uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam
      ? maxAllowedClaimAmount
      : potentialClaimAmount;

    fund -= claimAmount; // Deduct the claim amount from the fund

    nftContract.burn(tokenId); // Burn the token
    (bool success, ) = payable(msg.sender).call{ value: claimAmount }('');
    require(success, 'Failed to send Ether');

    emit Nuked(msg.sender, tokenId, claimAmount); // Emit the event with the actual claim amount
    emit FundBalanceUpdated(fund); // Update the fund balance
  }
```

## Finding 6: Desynchronized Cap Parameters in `NukeFund.nuke` Cause Payout Limit Bypass and Discontinuous Claim Amounts

### Synopsis

In `NukeFund.nuke`, the maximum claim cap check compares `finalNukeFactor` against `nukeFactorMaxParam`, while calculating the capped payout using a separate configurable variable, `maxAllowedClaimDivisor`. When either parameter is updated independently, `claimAmount` can exceed `maxAllowedClaimAmount`, experience a sharp discontinuity at the threshold, or revert from arithmetic underflow.

### Technical details

In `contracts/NukeFund/NukeFund.sol`, the `nuke` function calculates a user's ETH payout from `fund` when burning an NFT:

```solidity
uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR;
uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor;

uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam
  ? maxAllowedClaimAmount
  : potentialClaimAmount;

fund -= claimAmount;
```

The contract defines two separate, independently configurable state variables intended to control the redemption cap:
1. `maxAllowedClaimDivisor` (default `2`, updated via `setMaxAllowedClaimDivisor`), which defines `maxAllowedClaimAmount = fund / maxAllowedClaimDivisor`.
2. `nukeFactorMaxParam` (default `MAX_DENOMINATOR / 2 = 50000`, updated via `setNukeFactorMaxParam`), which acts as the threshold on `finalNukeFactor` for switching from `potentialClaimAmount` to `maxAllowedClaimAmount`.

Because `potentialClaimAmount` is `(fund * finalNukeFactor) / MAX_DENOMINATOR` and `maxAllowedClaimAmount` is `fund / maxAllowedClaimDivisor`, the threshold and the cap only align when `nukeFactorMaxParam == MAX_DENOMINATOR / maxAllowedClaimDivisor`. However, both parameters have independent setter functions (`setMaxAllowedClaimDivisor` and `setNukeFactorMaxParam`) and neither updates or validates the other. Furthermore, line 170 checks `finalNukeFactor > nukeFactorMaxParam` rather than comparing `potentialClaimAmount > maxAllowedClaimAmount` (or `Math.min(potentialClaimAmount, maxAllowedClaimAmount)`).

This creates three distinct failure modes whenever the two parameters are not kept in exact inverse proportion (`nukeFactorMaxParam * maxAllowedClaimDivisor == MAX_DENOMINATOR`):

1. **Bypass of `maxAllowedClaimAmount` and negative payout cliff (`nukeFactorMaxParam > MAX_DENOMINATOR / maxAllowedClaimDivisor`):**
   If the owner increases `maxAllowedClaimDivisor` (e.g., from `2` to `4` to cap maximum claims at 25% of `fund`) without updating `nukeFactorMaxParam` (still `50000`), any token with `25000 < finalNukeFactor <= 50000` satisfies `finalNukeFactor <= nukeFactorMaxParam`. The ternary expression selects `potentialClaimAmount`, allowing the caller to claim up to 50% of `fund` and completely bypassing `maxAllowedClaimAmount` (25% of `fund`). Once `finalNukeFactor` exceeds `50000`, the payout suddenly drops from 50% down to 25% of `fund`, penalizing holders for holding longer.
2. **Instantaneous payout jump (`nukeFactorMaxParam < MAX_DENOMINATOR / maxAllowedClaimDivisor`):**
   If the owner lowers `nukeFactorMaxParam` (e.g., to `25000`) without updating `maxAllowedClaimDivisor` (still `2`), a token whose `finalNukeFactor` increases from `25000` to `25001` immediately jumps from claiming 25% of `fund` (`potentialClaimAmount`) to claiming 50% of `fund` (`maxAllowedClaimAmount`).
3. **Reserve bound violation / underflow (`nukeFactorMaxParam > MAX_DENOMINATOR`):**
   If `nukeFactorMaxParam` is configured above `MAX_DENOMINATOR` (`100000`), a token with `MAX_DENOMINATOR < finalNukeFactor <= nukeFactorMaxParam` yields `potentialClaimAmount > fund`, causing `fund -= claimAmount` at line 174 to revert with an arithmetic underflow and permanently preventing mature high-factor tokens from being nuked until they pass `nukeFactorMaxParam`.

### Assumptions and preconditions

1. The `NukeFund` contract holds a non-zero `fund` balance.
2. Either `setMaxAllowedClaimDivisor` or `setNukeFactorMaxParam` is called by the owner to adjust the claim limit (as intended by the presence of both setter functions in `INukeFund` and `NukeFund`) without manually recalculating and updating the other parameter in tandem.
3. A user holds a mature NFT (`canTokenBeNuked(tokenId) == true`) and calls `nuke(tokenId)`.

### Proof of concept

1. `NukeFund` is deployed with default parameters:
   - `MAX_DENOMINATOR = 100000`
   - `maxAllowedClaimDivisor = 2`
   - `nukeFactorMaxParam = 50000`
   - `fund = 100 ether`
2. To reduce the maximum single-nuke drain from 50% to 20% of the pool, the owner calls `setMaxAllowedClaimDivisor(5)`. Now `maxAllowedClaimAmount = 100 ether / 5 = 20 ether`.
3. A user holds a token whose `finalNukeFactor` has grown to `45000` (45%).
4. The user calls `nuke(tokenId)`:
   - `potentialClaimAmount = (100 ether * 45000) / 100000 = 45 ether`.
   - `maxAllowedClaimAmount = 100 ether / 5 = 20 ether`.
   - The condition `finalNukeFactor > nukeFactorMaxParam` evaluates `45000 > 50000`, which is `false`.
   - `claimAmount` is assigned `potentialClaimAmount = 45 ether`.
5. The user receives `45 ether` instead of the capped `20 ether`, draining more than twice the configured maximum allowed claim from `NukeFund`.
6. Conversely, if another user holds a token whose `finalNukeFactor` reaches `50001`, `finalNukeFactor > nukeFactorMaxParam` is `true`, and that user only receives `maxAllowedClaimAmount = 20 ether`—less than half of what the lower-factor token claimed.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** `claimAmount` is not bounded by `maxAllowedClaimAmount`. Depending on how `maxAllowedClaimDivisor` and `nukeFactorMaxParam` are configured, users can bypass the intended maximum claim limit (`maxAllowedClaimAmount`), receive artificially inflated payouts upon crossing `nukeFactorMaxParam`, or suffer a sharp reduction in payout as their token matures past `nukeFactorMaxParam`.
- **Business Implications:** The NukeFund honeypot is the central economic reserve of the TraitForge protocol. Draining a larger share of `fund` than permitted by `maxAllowedClaimAmount` directly depletes rewards belonging to all other NFT holders and breaks the core game-theoretic invariant that older/higher-nuke-factor tokens yield greater or equal rewards up to a hard cap.

### Remediation

Cap `potentialClaimAmount` directly against `maxAllowedClaimAmount` (or ensure the condition checks `potentialClaimAmount > maxAllowedClaimAmount`) so that `claimAmount` can never exceed `maxAllowedClaimAmount` and transitions continuously at the cap:

```solidity
    uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
    uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
    uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

    uint256 claimAmount = potentialClaimAmount > maxAllowedClaimAmount ||
      finalNukeFactor > nukeFactorMaxParam
      ? maxAllowedClaimAmount
      : potentialClaimAmount;

    fund -= claimAmount; // Deduct the claim amount from the fund

    nftContract.burn(tokenId); // Burn the token
    (bool success, ) = payable(msg.sender).call{ value: claimAmount }('');
```

Alternatively, remove `nukeFactorMaxParam` altogether and rely solely on `potentialClaimAmount > maxAllowedClaimAmount ? maxAllowedClaimAmount : potentialClaimAmount`, or derive `maxAllowedClaimAmount` directly from `nukeFactorMaxParam` (`(fund * nukeFactorMaxParam) / MAX_DENOMINATOR` with `require(nukeFactorMaxParam <= MAX_DENOMINATOR)`).

## Finding 7: Payout Sent to `msg.sender` Instead of Token Owner in `NukeFund.nuke` Allows Approved Operators to Steal Nuke Rewards

### Synopsis

In `NukeFund.nuke`, the ETH reward from burning an NFT is transferred to `payable(msg.sender)` rather than the actual owner of the NFT (`nftContract.ownerOf(tokenId)`). Because `nuke` permits approved addresses and operators to invoke the function, an approved party can nuke the token and divert the ETH payout to themselves.

### Technical details

In `contracts/NukeFund/NukeFund.sol`, `nuke(uint256 tokenId)` checks whether `msg.sender` is the owner or an approved party for `tokenId`:

```solidity
require(
  nftContract.isApprovedOrOwner(msg.sender, tokenId),
  'ERC721: caller is not token owner or approved'
);
require(
  nftContract.getApproved(tokenId) == address(this) ||
    nftContract.isApprovedForAll(msg.sender, address(this)),
  'Contract must be approved to transfer the NFT.'
);
```

When `nftContract.burn(tokenId)` is subsequently executed on line 176, `TraitForgeNft.burn` checks `isApprovedOrOwner(msg.sender, tokenId)` where `msg.sender` is the `NukeFund` contract.

On line 177, `NukeFund.nuke` transfers the resulting `claimAmount` of ETH directly to `payable(msg.sender)`:

```solidity
nftContract.burn(tokenId); // Burn the token
(bool success, ) = payable(msg.sender).call{ value: claimAmount }('');
```

This creates a payout recipient entitlement mismatch and an owner-operator privilege separation failure:
1. If a token owner has approved `NukeFund` (either via `approve(address(nukeFund), tokenId)` or `setApprovalForAll(address(nukeFund), true)`), any operator approved by the owner (`isApprovedForAll(owner, operator)`)—or any address specifically approved for `tokenId` while `isApprovedForAll(owner, address(nukeFund))` is active (after calling `nftContract.setApprovalForAll(address(nukeFund), true)` for itself)—passes both `require` checks in `nuke`.
2. When that approved address calls `nuke(tokenId)`, the owner's NFT is permanently burned, and the entire ETH `claimAmount` is paid out to `msg.sender` (the operator/approved address) instead of the NFT owner.

### Assumptions and preconditions

- A token owner owns a mature `TraitForgeNft` token (`canTokenBeNuked(tokenId) == true`).
- `NukeFund` is authorized to burn the token on the owner's behalf (either via `approve(address(nukeFund), tokenId)` before the owner sets an operator via `setApprovalForAll(operator, true)`, or via `setApprovalForAll(address(nukeFund), true)` while another address is approved via `approve(operator, tokenId)` or `setApprovalForAll(operator, true)`).

### Proof of concept

1. Alice owns a mature `tokenId` and has granted `NukeFund` approval via `nftContract.setApprovalForAll(address(nukeFund), true)` (or `nftContract.approve(address(nukeFund), tokenId)`).
2. Alice also approves an operator or third-party contract `Bob` (e.g., via `nftContract.setApprovalForAll(Bob, true)` or `nftContract.approve(Bob, tokenId)`).
3. `Bob` calls `nftContract.setApprovalForAll(address(nukeFund), true)` so that `nftContract.isApprovedForAll(Bob, address(nukeFund))` evaluates to `true`.
4. `Bob` calls `nukeFund.nuke(tokenId)`.
5. Inside `nuke`:
   - `nftContract.isApprovedOrOwner(Bob, tokenId)` returns `true`.
   - `nftContract.getApproved(tokenId) == address(nukeFund) || nftContract.isApprovedForAll(Bob, address(nukeFund))` returns `true`.
   - `nftContract.burn(tokenId)` burns Alice's NFT because `NukeFund` is approved by Alice.
   - `(bool success, ) = payable(msg.sender).call{ value: claimAmount }('')` sends the ETH payout to `Bob` instead of Alice.

### Impact assessment

- **Severity**: Medium / High
- **Direct Technical Effect**: An approved operator or approved spender can permanently destroy the owner's NFT and redirect 100% of the ETH nuke payout to their own address.
- **Business Implications**: Users who grant approvals to operators, marketplaces, or helper contracts risk losing both their NFT and the underlying ETH reward from `NukeFund`.

### Remediation

Fetch the token owner (`address tokenOwner = nftContract.ownerOf(tokenId);`) before burning the token, and send the `claimAmount` to `payable(tokenOwner)` instead of `payable(msg.sender)` (and check `isApprovedForAll(tokenOwner, address(this))` on line 160). Alternatively, restrict `nuke` so that only the token owner can call it.

```solidity
    address tokenOwner = nftContract.ownerOf(tokenId);

    uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
    uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
    uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

    // Directly assign the value to claimAmount based on the condition, removing the redeclaration
    uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam
      ? maxAllowedClaimAmount
      : potentialClaimAmount;

    fund -= claimAmount; // Deduct the claim amount from the fund

    nftContract.burn(tokenId); // Burn the token
    (bool success, ) = payable(tokenOwner).call{ value: claimAmount }('');
    require(success, 'Failed to send Ether');

    emit Nuked(tokenOwner, tokenId, claimAmount); // Emit the event with the actual claim amount
    emit FundBalanceUpdated(fund); // Update the fund balance
```

## Finding 8: Resetting `generationMintCounts` in `TraitForgeNft._incrementGeneration` erases forged token counts and allows generations to exceed `maxTokensPerGen`

### Synopsis

`TraitForgeNft` tracks per-generation supply across both direct minting (`_mintInternal`) and forging (`_mintNewEntity`) using the shared `generationMintCounts` mapping. However, `_incrementGeneration()` resets `generationMintCounts[currentGeneration] = 0` when advancing to the next generation, erasing the count of already-forged entities and allowing subsequent generations to exceed `maxTokensPerGen`.

### Technical details

In `TraitForgeNft`, each generation is intended to have a strict maximum supply of `maxTokensPerGen` (10,000 entities), counted across both direct mints and forged entities.

1. When two parent entities of generation `g` are forged via `forge()`, `_mintNewEntity()` is called to mint a child entity of generation `gen = g + 1`.
2. `_mintNewEntity()` enforces the per-generation cap via:
   ```solidity
   require(
     generationMintCounts[gen] < maxTokensPerGen,
     'Exceeds maxTokensPerGen'
   );
   ```
   and increments `generationMintCounts[gen]++` (line 328). Because forging of generation `1` entities occurs while `currentGeneration == 1`, `generationMintCounts[2]` (and potentially even higher generations `3, 4, ...` forged from those children) accumulates a non-zero count before `currentGeneration` ever reaches `2`.
3. Meanwhile, `_mintInternal()` mints entities in `currentGeneration` and increments `generationMintCounts[currentGeneration]++` (line 293).
4. Once `generationMintCounts[currentGeneration]` reaches `maxTokensPerGen` (either at the start of the next `_mintInternal()` call at lines 281–283, or inside `_mintNewEntity()` at lines 331–335), `_incrementGeneration()` is invoked:
   ```solidity
   function _incrementGeneration() private {
     require(
       generationMintCounts[currentGeneration] >= maxTokensPerGen,
       'Generation limit not yet reached'
     );
     currentGeneration++;
     generationMintCounts[currentGeneration] = 0;
     priceIncrement = priceIncrement + priceIncrementByGen;
     entropyGenerator.initializeAlphaIndices();
     emit GenerationIncremented(currentGeneration);
   }
   ```
5. Line 351 unconditionally sets `generationMintCounts[currentGeneration] = 0` for the newly entered generation. This overwrites and discards the count of all entities that were already forged into that generation while `currentGeneration` was at an earlier generation.

As a result:
- The `require(generationMintCounts[gen] < maxTokensPerGen, 'Exceeds maxTokensPerGen')` check in `_mintNewEntity` and the generation-transition check in `_mintInternal` operate on a reset counter, allowing up to `maxTokensPerGen` (10,000) *additional* tokens to be minted/forged in generation `g + 1` on top of the tokens already forged prior to the transition.
- Furthermore, because `calculateMintPrice()` computes the mint price using `generationMintCounts[currentGeneration]`, resetting the counter to `0` also resets the mint price progression for the new generation despite tokens of that generation already existing.

### Assumptions and preconditions

- Users mint entities in `currentGeneration` (e.g., Generation 1) and forge them via `EntityForging.forgeWithListed()` before all `maxTokensPerGen` (10,000) entities of `currentGeneration` have been minted.
- `currentGeneration` subsequently reaches `maxTokensPerGen` and transitions to `currentGeneration + 1` via `_incrementGeneration()`.

### Proof of concept

1. Initially, `currentGeneration = 1` and `generationMintCounts[1] = 0`, `generationMintCounts[2] = 0`.
2. Users mint 2,000 Generation 1 NFTs via `mintToken()`. Now `generationMintCounts[1] = 2000`.
3. Users forge pairs of Generation 1 NFTs via `EntityForging.forgeWithListed()` to create 500 Generation 2 NFTs. `_mintNewEntity()` increments `generationMintCounts[2]` to `500`.
4. Users mint the remaining 8,000 Generation 1 NFTs via `mintToken()`, bringing `generationMintCounts[1]` to `10000` (`maxTokensPerGen`).
5. On the next `mintToken()` call, `_mintInternal()` sees `generationMintCounts[1] >= 10000` and calls `_incrementGeneration()`.
6. `_incrementGeneration()` increments `currentGeneration` from `1` to `2` and executes `generationMintCounts[2] = 0`, erasing the record of the 500 already-minted Generation 2 NFTs.
7. Users can now mint or forge 10,000 additional Generation 2 NFTs, resulting in 10,500 Generation 2 NFTs in total and violating the `maxTokensPerGen` (10,000) supply cap.

### Impact assessment

- **Severity**: High / Medium
- **Technical Impact**: The `maxTokensPerGen` supply invariant (10,000 entities per generation, 100,000 total across 10 generations) is broken for every generation `>= 2`. Up to `2 * maxTokensPerGen - 1` tokens can be created in a single generation if tokens are forged before the previous generation completes minting.
- **Economic Impact**: Excess NFT supply dilutes the scarcity of entities and distorts `calculateMintPrice()`, which relies on `generationMintCounts[currentGeneration]` to determine the bonding curve price.

### Remediation

Do not reset `generationMintCounts[currentGeneration]` to `0` in `_incrementGeneration()`. Additionally, if the newly entered generation has already reached `maxTokensPerGen` through prior forging, ensure the generation transition handles already-full generations appropriately and respects `maxGeneration`.

```solidity
  function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    require(currentGeneration < maxGeneration, 'Max generation reached');
    currentGeneration++;
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
  }
```

## Finding 9: Incorrect Airdrop Entitlement Attribution via `initialOwners` in `TraitForgeNft._mintInternal` Causes Mismatched Reward Accounting on Token Transfer and Burn

### Synopsis

In `TraitForgeNft._mintInternal`, airdrop entitlement is statically attributed to `initialOwners[newItemId]` via `airdropContract.addUserAmount`. Because entitlement is never updated on token transfers while `burn` deducts `entropy` from `initialOwners[tokenId]`, subsequent holders who burn or nuke a transferred NFT strip the original minter's airdrop rewards.

### Technical details

When a token is minted in `TraitForgeNft._mintInternal` (lines 294–298), the contract records the recipient in `initialOwners[newItemId]` and increases their airdrop weight in `Airdrop` by the token's `entropyValue`:

```solidity
initialOwners[newItemId] = to;

if (!airdropContract.airdropStarted()) {
  airdropContract.addUserAmount(to, entropyValue);
}
```

Similarly, `_mintNewEntity` sets `initialOwners[newTokenId] = newOwner` and credits `newOwner` via `airdropContract.addUserAmount(newOwner, entropy)`.

When a token is later burned in `TraitForgeNft.burn(uint256 tokenId)` (either directly by its current owner or via `NukeFund.nuke(uint256 tokenId)` when claiming ETH from the nuke fund), the contract revokes the token's airdrop weight before the airdrop has started:

```solidity
if (!airdropContract.airdropStarted()) {
  uint256 entropy = getTokenEntropy(tokenId);
  airdropContract.subUserAmount(initialOwners[tokenId], entropy);
}
```

However, `TraitForgeNft._beforeTokenTransfer` does not move the token's `entropy` weight in `airdropContract` from `from` to `to` when an NFT is transferred (for example, via `transferFrom`, `safeTransferFrom`, or a sale through `EntityTrading`). This creates a direct entitlement attribution mismatch:

1. **If airdrop weight is intended to belong to the active NFT holder (hence revoked on `burn`/`nuke`):** Selling or transferring an NFT leaves the airdrop entitlement with `initialOwners[tokenId]` instead of transferring it to the new owner `to`.
2. **If airdrop weight is intended to reward the minter (`initialOwners[tokenId]`):** When the minter sells or transfers the NFT to a second user and that second user subsequently calls `NukeFund.nuke(tokenId)` (or `TraitForgeNft.burn(tokenId)`) before `airdropStarted()` is `true`, `burn()` calls `airdropContract.subUserAmount(initialOwners[tokenId], entropy)`. The new owner receives the ETH payout from `NukeFund`, while the original minter (`initialOwners[tokenId]`) loses their `userInfo` airdrop balance.

### Assumptions and preconditions

1. The airdrop has not yet started (`airdropContract.airdropStarted() == false`).
2. A user (User A) mints an NFT via `mintToken` or `mintWithBudget` (or forges one via `forge`), receiving `entropyValue` weight in `Airdrop.userInfo[User A]`.
3. User A transfers or sells the NFT (e.g., via `EntityTrading`) to another user (User B).
4. User B burns the NFT via `TraitForgeNft.burn(tokenId)` or nukes it via `NukeFund.nuke(tokenId)` before the airdrop starts.

### Proof of concept

1. While `airdropContract.airdropStarted()` is `false`, User A calls `TraitForgeNft.mintToken` and mints `tokenId = 1` with `entropyValue = 500000`.
   - `initialOwners[1]` is set to `User A`.
   - `airdropContract.userInfo(User A)` becomes `500000` and `airdropContract.totalValue()` increases by `500000`.
2. User A lists `tokenId = 1` on `EntityTrading` and User B purchases it (or User A transfers `tokenId = 1` to User B via `transferFrom`).
   - `ownerOf(1)` is now `User B`.
   - `initialOwners[1]` remains `User A`, and `airdropContract.userInfo(User B)` remains `0`.
3. After `minimumDaysHeld` has elapsed, User B approves `NukeFund` and calls `NukeFund.nuke(1)` while `airdropContract.airdropStarted()` is still `false`.
4. `NukeFund.nuke(1)` sends the ETH claim amount to `User B` and calls `TraitForgeNft.burn(1)`.
5. `TraitForgeNft.burn(1)` executes `airdropContract.subUserAmount(initialOwners[1], 500000)`.
6. `airdropContract.userInfo(User A)` is reduced from `500000` to `0`, depriving User A of their airdrop allocation due to User B nuking the token, while User B never held the corresponding airdrop weight.

### Impact assessment

**Severity: Medium**

- **Direct Technical Effect:** Airdrop entitlement accounting in `Airdrop.userInfo` becomes desynchronized from token ownership whenever NFTs are transferred prior to the start of the airdrop. Burning or nuking a transferred NFT deducts airdrop points from the original minter (`initialOwners[tokenId]`) rather than the burning holder.
- **Business Implications:** Minters who sell their NFTs on the secondary market (`EntityTrading`) have their accrued airdrop rewards unexpectedly wiped out when buyers later nuke or burn those NFTs. Conversely, secondary market buyers who acquire NFTs before the airdrop starts do not receive the airdrop weight associated with the NFTs they hold.

### Remediation

Ensure airdrop entitlement accounting is consistent across minting, transfers, and burns:

- **Preferred Solution (Holder-Based Attribution):** Attribute airdrop weight to the current NFT owner rather than tracking `initialOwners`. In `_beforeTokenTransfer`, when `from != address(0)` and `to != address(0)` and `!airdropContract.airdropStarted()`, transfer the token's `tokenEntropy[firstTokenId]` weight from `from` to `to` via `subUserAmount` and `addUserAmount` (or update `initialOwners[firstTokenId] = to`), and in `burn`, deduct `entropy` from `ownerOf(tokenId)`.
- **Alternative Solution (Minter-Based Attribution):** If airdrop rewards are strictly meant for minters who paid the protocol minting fee, remove `initialOwners` and do not call `airdropContract.subUserAmount` in `burn()`.

Corrected `_mintInternal` and `_beforeTokenTransfer` / `burn` alignment (removing redundant `initialOwners` assignment in `_mintInternal` when using `ownerOf(tokenId)` and transferring entitlement on token transfer):

```solidity
    if (!airdropContract.airdropStarted()) {
      airdropContract.addUserAmount(to, entropyValue);
    }

    emit Minted(
      msg.sender,
      newItemId,
      currentGeneration,
      entropyValue,
      mintPrice
    );

    _distributeFunds(mintPrice);
```

## Finding 10: Unfair Airdrop Entitlement Inflation and Economic Manipulation via Forging in `TraitForgeNft._mintNewEntity`

### Synopsis

In `TraitForgeNft._mintNewEntity`, forging a new entity credits `newOwner` with the child's full `entropy` in `Airdrop` via `airdropContract.addUserAmount(newOwner, entropy)`. Attackers using two wallets can self-forge high-entropy tokens across generations at minimal net cost, inflating their airdrop share and diluting legitimate minters.

### Technical details

The `Airdrop` contract distributes `totalTokenAmount` of `traitToken` proportionally to each user's accumulated entropy weight:

$$\text{claimable} = \frac{\text{totalTokenAmount} \times \text{userInfo}[\text{user}]}{\text{totalValue}}$$

Entropy weights are added to `Airdrop` in two places inside `TraitForgeNft`:
1. `_mintInternal` (lines 296–298): When minting a Generation 1 token, the minter pays the full `mintPrice` (100% of which is sent to `NukeFund` via `_distributeFunds`), and receives a random `entropyValue` from `entropyGenerator.getNextEntropy()`.
2. `_mintNewEntity` (lines 326–338): When forging a new token via `EntityForging.forgeWithListed` -> `TraitForgeNft.forge`, `_mintNewEntity` sets `tokenEntropy[newTokenId] = entropy` (line 326) and `initialOwners[newTokenId] = newOwner` (line 329), and credits `newOwner` with the full child `entropy` before the airdrop starts (lines 337–338):

```solidity
if (!airdropContract.airdropStarted()) {
  airdropContract.addUserAmount(newOwner, entropy);
}
```

This creates a severe economic and accounting imbalance in `Airdrop` entitlements:
- **Deterministic High Entropy via Forging:** Unlike `_mintInternal`, where entropy is pseudo-randomly drawn, `TraitForgeNft.forge` computes `newEntropy = (forgerEntropy + mergerEntropy) / 2` (line 173). By pairing two high-entropy parent tokens (e.g., near `999,999`), the resulting child token deterministically receives a near-maximum `entropy` value and high `forgePotential` (5th digit).
- **Minimal Net Cost via Two-Wallet Self-Forging:** In `EntityForging.forgeWithListed` (lines 146–159), `msg.sender` (the `mergerTokenId` owner, who becomes `newOwner`) pays `forgingFee`. However, `forgerShare = forgingFee - (forgingFee / taxCut)` (90% of the fee) is refunded directly to `forgerOwner`, and only `devFee = forgingFee / taxCut` (10% of the fee) is sent to `nukeFundAddress`. An attacker holding a forger token in Wallet A and a merger token in Wallet B can list at the minimum fee and forge between their own accounts, paying only `devFee` while crediting Wallet B (`newOwner`) with near-maximum `entropy` in `Airdrop`.
- **Compounding Across Generations:** Because the forged Gen 2 entities can immediately be used to forge Gen 3, Gen 4, ..., up to `maxGeneration` (Gen 10) before `airdropStarted()` becomes `true`, an attacker can recursively multiply their `userInfo` entitlement in `Airdrop` at negligible cost, capturing an outsized portion of `totalTokenAmount` at the expense of honest Gen 1 minters. Furthermore, in legitimate two-party forging, `forgerOwner` contributes one of the two parent tokens but receives zero airdrop credit while `newOwner` receives 100% of the child's `entropy`.

### Assumptions and preconditions

- The airdrop has not yet started (`airdropContract.airdropStarted() == false`).
- A user controls two addresses (Wallet A and Wallet B) holding a high-entropy forger token and a high-entropy merger token of the same generation with non-zero `forgePotential`.

### Proof of concept

1. Honest users mint Generation 1 tokens via `mintToken`, paying the rising `mintPrice` (100% to `NukeFund`) and receiving average random entropy (~`500,000` each) in `Airdrop.userInfo`.
2. An attacker holds a Forger token (`forgerEntropy = 990,090`, `forgePotential = 9`) in Wallet A and a Merger token (`mergerEntropy = 990,092`, `forgePotential = 9`) in Wallet B.
3. Wallet A lists the Forger token for the minimum forging fee (`0.01 ETH`), and Wallet B calls `EntityForging.forgeWithListed(forgerTokenId, mergerTokenId)`.
4. Wallet A receives `0.009 ETH` back (`forgerShare`), so the attacker's net cost is only `0.001 ETH` (`devFee`).
5. `TraitForgeNft._mintNewEntity` mints a Gen 2 token to Wallet B with `entropy = 990,091` (`forgePotential = 9`) and calls `airdropContract.addUserAmount(Wallet B, 990091)` at line 338.
6. The attacker repeats this process up to `forgePotential` (9 times) for the Gen 1 pair, and then uses the newly forged Gen 2 tokens (transferred between Wallet A and Wallet B) to forge Gen 3 through Gen 10 tokens before the airdrop begins.
7. Once `startAirdrop` is called, the attacker invokes `Airdrop.claim()` and drains the majority of `totalTokenAmount` due to their heavily inflated `userInfo` balance.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:** `userInfo[newOwner]` and `totalValue` in `Airdrop` can be artificially inflated at a small fraction of the standard minting cost by recursively forging high-entropy entities before the airdrop starts.
- **Business Implications:** Honest users who paid full mint prices into `NukeFund` suffer severe dilution of their `traitToken` airdrop rewards, undermining the protocol's tokenomics and incentive structure.

### Remediation

Do not grant full entropy airdrop allocations to newly forged tokens in `_mintNewEntity` (or restrict airdrop eligibility strictly to Gen 1 mints in `_mintInternal` and correspondingly only deduct Gen 1 tokens in `burn`).

If forged tokens should not accrue airdrop entitlements, update `_mintNewEntity` and `burn` in `contracts/TraitForgeNft/TraitForgeNft.sol`:

```solidity
  function burn(uint256 tokenId) external whenNotPaused nonReentrant {
    require(
      isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    if (!airdropContract.airdropStarted() && tokenGenerations[tokenId] == 1) {
      uint256 entropy = getTokenEntropy(tokenId);
      airdropContract.subUserAmount(initialOwners[tokenId], entropy);
    }
    _burn(tokenId);
  }
```

```solidity
  function _mintNewEntity(
    address newOwner,
    uint256 entropy,
    uint256 gen
  ) private returns (uint256) {
    require(
      generationMintCounts[gen] < maxTokensPerGen,
      'Exceeds maxTokensPerGen'
    );

    _tokenIds++;
    uint256 newTokenId = _tokenIds;
    _mint(newOwner, newTokenId);

    tokenCreationTimestamps[newTokenId] = block.timestamp;
    tokenEntropy[newTokenId] = entropy;
    tokenGenerations[newTokenId] = gen;
    generationMintCounts[gen]++;
    initialOwners[newTokenId] = newOwner;

    if (
      generationMintCounts[gen] >= maxTokensPerGen && gen == currentGeneration
    ) {
      _incrementGeneration();
    }

    emit NewEntityMinted(newOwner, newTokenId, gen, entropy);
    return newTokenId;
  }
```

Alternatively, if forged entities are intended to participate in the airdrop, weight `addUserAmount` by the actual fee contributed to `NukeFund` (`devFee`) or split the parent/child entitlement proportionally between `forgerOwner` and `newOwner`.

## Finding 11: Missing `maxGeneration` Terminal Check in `TraitForgeNft._mintNewEntity` and `_incrementGeneration` Allows Minting Beyond Generation 10

### Synopsis

In `TraitForgeNft._mintNewEntity`, reaching `maxTokensPerGen` when `gen == currentGeneration` unconditionally calls `_incrementGeneration()` without checking if `currentGeneration` has reached `maxGeneration`. This advances `currentGeneration` beyond the terminal generation cap (`10`), allowing `mintToken` to mint NFTs in Generation 11 and beyond.

### Technical details

The `TraitForgeNft` protocol defines a hard upper bound of 10 generations via `uint256 public maxGeneration = 10` (line 35), with up to `maxTokensPerGen = 10000` (line 22) entities per generation (100,000 entities total).

In `TraitForgeNft.forge` (lines 163–166), this generation cap is enforced for forged entities:

```solidity
uint256 newGeneration = getTokenGeneration(parent1Id) + 1;
require(newGeneration <= maxGeneration, "can't be over max generation");
```

However, when `TraitForgeNft._mintNewEntity` mints a forged token of generation `gen` (lines 328–335):

```solidity
generationMintCounts[gen]++;
initialOwners[newTokenId] = newOwner;

if (
  generationMintCounts[gen] >= maxTokensPerGen && gen == currentGeneration
) {
  _incrementGeneration();
}
```

Neither the rollover condition at lines 331–335 in `_mintNewEntity` (nor `_incrementGeneration` at lines 345–355 or `_mintInternal` at lines 281–283) checks whether `currentGeneration` has reached the terminal generation `maxGeneration`.

Consequently, when `currentGeneration == maxGeneration` (`10`) and the 10,000th Generation 10 entity is forged from two Generation 9 parents (`gen == 10 == currentGeneration`), `generationMintCounts[10] >= maxTokensPerGen && gen == currentGeneration` evaluates to `true` and `_mintNewEntity` calls `_incrementGeneration()`. Inside `_incrementGeneration()` (line 350), `currentGeneration++` increments `currentGeneration` to `11`, resets `generationMintCounts[11] = 0`, and updates `priceIncrement`.

Because `mintToken` and `_mintInternal` do not validate `currentGeneration <= maxGeneration` and assign `tokenGenerations[newItemId] = currentGeneration` (line 292), advancing `currentGeneration` past `maxGeneration` allows users to continue minting Generation 11 (and subsequent generations) via `mintToken`, violating both the `maxGeneration = 10` terminal cap and the 100,000 total token cap. Furthermore, if `_incrementGeneration()` simply reverted at `maxGeneration`, calling `_incrementGeneration()` inside `_mintNewEntity` upon forging the 10,000th Generation 10 token would cause that final valid forge transaction to revert unless `_mintNewEntity` guards the rollover with `gen < maxGeneration`.

### Assumptions and preconditions

1. The protocol progresses to `currentGeneration == maxGeneration` (`10`).
2. `generationMintCounts[10]` reaches `maxTokensPerGen` (`10000`) when a Generation 10 entity is forged via `EntityForging.forgeWithListed` -> `TraitForgeNft.forge` -> `TraitForgeNft._mintNewEntity` (or when `mintToken` triggers `_incrementGeneration()` after 10,000 Generation 10 tokens).

### Proof of concept

1. The protocol reaches Generation 10 (`currentGeneration = 10`, `maxGeneration = 10`), and 9,999 Generation 10 tokens have been minted (`generationMintCounts[10] = 9999`).
2. A user calls `EntityForging.forgeWithListed(forgerTokenId, mergerTokenId)` using two Generation 9 parent tokens.
3. `TraitForgeNft.forge` computes `newGeneration = 9 + 1 = 10`, which passes `require(newGeneration <= maxGeneration)` (line 166), and calls `_mintNewEntity(newOwner, newEntropy, 10)`.
4. In `_mintNewEntity`, line 328 increments `generationMintCounts[10]` to `10000` (`maxTokensPerGen`).
5. At lines 331–335, `generationMintCounts[10] >= 10000 && 10 == 10` is `true`, so `_incrementGeneration()` is called.
6. `_incrementGeneration()` increments `currentGeneration` to `11` (`> maxGeneration`) and sets `generationMintCounts[11] = 0`.
7. Any user can now call `TraitForgeNft.mintToken` to mint Generation 11 tokens (`tokenGenerations[newItemId] = 11`), bypassing the `maxGeneration = 10` limit.

### Impact assessment

- **Severity**: Medium
- **Technical Impact**: `currentGeneration` advances beyond `maxGeneration` (`10`), allowing tokens with generation `> 10` to be minted via `mintToken` and breaking the maximum generation invariant and total supply limit of 100,000 NFTs.
- **Business Impact**: Dilutes the scarcity and economic value of existing NFTs and the NukeFund pool by permitting unbounded generation rollover and token inflation past the terminal generation.

### Remediation

In `TraitForgeNft._mintNewEntity`, only trigger `_incrementGeneration()` when `gen < maxGeneration`. Additionally, enforce `currentGeneration <= maxGeneration` in `_mintInternal` and `currentGeneration < maxGeneration` in `_incrementGeneration()`.

```solidity
    tokenEntropy[newTokenId] = entropy;
    tokenGenerations[newTokenId] = gen;
    generationMintCounts[gen]++;
    initialOwners[newTokenId] = newOwner;

    if (
      generationMintCounts[gen] >= maxTokensPerGen &&
      gen == currentGeneration &&
      gen < maxGeneration
    ) {
      _incrementGeneration();
    }

    if (!airdropContract.airdropStarted()) {
      airdropContract.addUserAmount(newOwner, entropy);
    }
```
