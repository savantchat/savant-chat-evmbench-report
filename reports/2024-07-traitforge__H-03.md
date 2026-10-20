# Audit report — 2024-07-traitforge/H-03

## Finding 1: Premature Initialization of `lastForgeResetTimestamp` in `EntityForging._resetForgingCountIfNeeded` Allows Back-to-Back Annual Forging Quotas

### Synopsis

`EntityForging._resetForgingCountIfNeeded` initializes `lastForgeResetTimestamp[tokenId]` on the first listing or first forge rather than when a token's forging quota is exhausted. An owner can start the one-year reset timer early (e.g., by listing and cancelling) and later execute two full annual forging quotas back-to-back across the boundary.

### Technical details

In `EntityForging.sol`, each entity's annual forging count is tracked via `forgingCounts` (`22`) and its reset window is tracked via `lastForgeResetTimestamp` (`23`):

```solidity
mapping(uint256 => uint8) public forgingCounts; // track forgePotential
mapping(uint256 => uint256) private lastForgeResetTimestamp;
```

The helper function `_resetForgingCountIfNeeded` (lines 199–207) manages the one-year reset period (`oneYearInDays`, initialized to `365 days`):

```solidity
function _resetForgingCountIfNeeded(uint256 tokenId) private {
  uint256 oneYear = oneYearInDays;
  if (lastForgeResetTimestamp[tokenId] == 0) {
    lastForgeResetTimestamp[tokenId] = block.timestamp;
  } else if (block.timestamp >= lastForgeResetTimestamp[tokenId] + oneYear) {
    forgingCounts[tokenId] = 0; // Reset to the forge potential
    lastForgeResetTimestamp[tokenId] = block.timestamp;
  }
}
```

This design has two interconnected flaws in how the reset window is tracked across state transitions:

1. **Timer starts on `listForForging` before any forge occurs:** `listForForging` calls `_resetForgingCountIfNeeded(tokenId)` at line 83 prior to any forging activity. When `lastForgeResetTimestamp[tokenId] == 0`, simply listing a Forger token (even if immediately cancelled via `cancelListingForForging`) sets `lastForgeResetTimestamp[tokenId] = block.timestamp` while `forgingCounts[tokenId]` remains `0`.
2. **Fixed anchor instead of cooldown after quota exhaustion or rolling window:** Because `lastForgeResetTimestamp[tokenId]` is anchored to the timestamp of the very first call to `_resetForgingCountIfNeeded(tokenId)`, all subsequent forges within that 365-day window do not update `lastForgeResetTimestamp[tokenId]`. Consequently, a token owner can wait until just before `lastForgeResetTimestamp[tokenId] + oneYear` to consume the entire first year's `forgePotential` quota, and immediately after `lastForgeResetTimestamp[tokenId] + oneYear` passes, `_resetForgingCountIfNeeded` resets `forgingCounts[tokenId]` to `0`, allowing the token to consume a second full annual `forgePotential` quota within minutes.

### Assumptions and preconditions

1. A user owns a Forger NFT (with `forgePotential > 0` and `entropy % 3 == 0`) or a Merger NFT (with `mergerForgePotential > 0` and `entropy % 3 != 0`).
2. The `EntityForging` contract is unpaused.

### Proof of concept

1. At `t = T_0`, Alice mints a Forger NFT (`tokenId = 1`) with `forgePotential = 5`.
2. Immediately at `t = T_0`, Alice calls `listForForging(1, minimumListFee)` and then `cancelListingForForging(1)`.
   - Inside `listForForging`, `_resetForgingCountIfNeeded(1)` sees `lastForgeResetTimestamp[1] == 0` and sets `lastForgeResetTimestamp[1] = T_0`.
   - `forgingCounts[1]` is still `0`.
3. At `t = T_0 + 365 days - 10 minutes`, Alice lists and forges `tokenId = 1` across 5 transactions.
   - Since `block.timestamp < T_0 + 365 days`, `_resetForgingCountIfNeeded(1)` does not reset the counter, and `forgingCounts[1]` reaches its annual limit.
4. At `t = T_0 + 365 days` (10 minutes later), Alice calls `listForForging(1, minimumListFee)`.
   - Inside `_resetForgingCountIfNeeded(1)`, `block.timestamp >= lastForgeResetTimestamp[1] + 365 days` evaluates to `true`.
   - `forgingCounts[1]` is reset to `0` and `lastForgeResetTimestamp[1]` is updated to `T_0 + 365 days`.
5. Alice immediately performs another full set of 5 forges with `tokenId = 1`, completing two years' worth of forging quotas within a 10-minute window.

### Impact assessment

- **Severity:** Medium
- **Technical Effect:** The one-year forging cooldown can be pre-started without forging (via `listForForging`) and resets 365 days after initialization regardless of when the quota was actually consumed, allowing back-to-back bursts of `2 * forgePotential` forges.
- **Business Impact:** Distorts generation progression and entity supply control by allowing players to pre-age forging reset timers and batch two annual forging quotas into a single short window.

### Remediation

Do not initialize `lastForgeResetTimestamp` during `listForForging` when `forgingCounts[tokenId] == 0`, and update `lastForgeResetTimestamp[tokenId]` when a forge occurs (or when the entity reaches its `forgePotential` limit) so that the one-year cooldown is enforced after the quota is used:

```solidity
  function listForForging(
    uint256 tokenId,
    uint256 fee
  ) public whenNotPaused nonReentrant {
    Listing memory _listingInfo = listings[listedTokenIds[tokenId]];

    require(!_listingInfo.isListed, 'Token is already listed for forging');
    require(
      nftContract.ownerOf(tokenId) == msg.sender,
      'Caller must own the token'
    );
    require(
      fee >= minimumListFee,
      'Fee should be higher than minimum listing fee'
    );

    if (lastForgeResetTimestamp[tokenId] != 0) {
      _resetForgingCountIfNeeded(tokenId);
    }

    uint256 entropy = nftContract.getTokenEntropy(tokenId); // Retrieve entropy for tokenId
    uint8 forgePotential = uint8((entropy / 10) % 10); // Extract the 5th digit from the entropy
    require(
      forgePotential > 0 && forgingCounts[tokenId] < forgePotential,
      'Entity has reached its forging limit'
    );

    bool isForger = (entropy % 3) == 0; // Determine if the token is a forger based on entropy
    require(isForger, 'Only forgers can list for forging');

    ++listingCount;
    listings[listingCount] = Listing(msg.sender, tokenId, true, fee);
    listedTokenIds[tokenId] = listingCount;

    emit ListedForForging(tokenId, fee);
  }
```

And in `_resetForgingCountIfNeeded`:

```solidity
  function _resetForgingCountIfNeeded(uint256 tokenId) private {
    uint256 oneYear = oneYearInDays;
    if (
      lastForgeResetTimestamp[tokenId] != 0 &&
      block.timestamp >= lastForgeResetTimestamp[tokenId] + oneYear
    ) {
      forgingCounts[tokenId] = 0;
    }
    lastForgeResetTimestamp[tokenId] = block.timestamp;
  }
```

## Finding 2: Off-by-One Quota Check in `EntityForging.listForForging` Allows Forger NFTs to Exceed Their `forgePotential`

### Synopsis

In `EntityForging.listForForging`, the forging quota check validates `forgingCounts[tokenId] <= forgePotential` before `forgingCounts` is incremented during `forgeWithListed`. Because `forgeWithListed` increments `forgingCounts[forgerTokenId]` without re-validating against `forgePotential`, any Forger NFT can be listed and forged `forgePotential + 1` times per year instead of `forgePotential`.

### Technical details

Each NFT entity has a `forgePotential` value (extracted as the 5th digit of its 6-digit entropy, `uint8((entropy / 10) % 10)`) that defines the maximum number of times the entity is allowed to forge within a one-year window, tracked in `EntityForging.forgingCounts` (line 22).

For a Merger NFT, `EntityForging.forgeWithListed` increments `forgingCounts[mergerTokenId]` first and then enforces the upper bound (lines 139–144):

```solidity
forgingCounts[mergerTokenId]++;
require(
  mergerForgePotential > 0 &&
    forgingCounts[mergerTokenId] <= mergerForgePotential,
  'forgePotential insufficient'
);
```

This ensures a Merger NFT can only participate in forging `mergerForgePotential` times per year (as `forgingCounts` transitions `0 -> 1`, ..., `mergerForgePotential - 1 -> mergerForgePotential`).

However, for a Forger NFT, the quota check is performed in `EntityForging.listForForging` (lines 87–90) prior to incrementing `forgingCounts[tokenId]`:

```solidity
require(
  forgePotential > 0 && forgingCounts[tokenId] <= forgePotential,
  'Entity has reached its forging limit'
);
```

Later, when `EntityForging.forgeWithListed` is executed, `forgingCounts[forgerTokenId]` is incremented without any further check against `forgePotential` (lines 131–133):

```solidity
// Check forger's breed count increment but do not check forge potential here
// as it is already checked in listForForging for the forger
forgingCounts[forgerTokenId]++;
```

Because `listForForging` uses `<=` instead of `<` before the counter is incremented, a Forger NFT that has already forged `forgePotential` times (`forgingCounts[tokenId] == forgePotential`) still passes the `forgingCounts[tokenId] <= forgePotential` check in `listForForging`. It can then be forged one additional time in `forgeWithListed`, bringing `forgingCounts[forgerTokenId]` to `forgePotential + 1`.

### Assumptions and preconditions

1. A user owns a Forger NFT (`entropy % 3 == 0`) with `forgePotential > 0`.
2. The `EntityForging` and `TraitForgeNft` contracts are unpaused.

### Proof of concept

1. Suppose Alice owns a Generation 1 Forger NFT (`forgerTokenId`) whose entropy yields `forgePotential = 1`. Initially, `forgingCounts[forgerTokenId] == 0`.
2. **First Forge:**
   - Alice calls `listForForging(forgerTokenId, minimumListFee)`. At line 88, `forgePotential > 0 && 0 <= 1` evaluates to `true`, and the token is listed.
   - Bob calls `forgeWithListed(forgerTokenId, mergerTokenId1)` with a valid Merger NFT. At line 133, `forgingCounts[forgerTokenId]` is incremented from `0` to `1`, a new Generation 2 entity is minted, and the listing is cancelled.
3. **Second Forge (Exceeding `forgePotential`):**
   - Even though `forgerTokenId` has `forgePotential = 1` and has already forged once (`forgingCounts[forgerTokenId] == 1`), Alice calls `listForForging(forgerTokenId, minimumListFee)` again.
   - At line 88, `forgePotential > 0 && 1 <= 1` still evaluates to `true`, allowing `forgerTokenId` to be listed a second time.
   - Bob calls `forgeWithListed(forgerTokenId, mergerTokenId2)`. At line 133, `forgingCounts[forgerTokenId]` is incremented from `1` to `2`, and a second Generation 2 entity is minted using `forgerTokenId`.

### Impact assessment

**Severity: Medium**

- **Direct Technical Effect:** Every Forger NFT with `forgePotential >= 1` can forge `forgePotential + 1` times per year instead of `forgePotential` times.
- **Business and Economic Impact:** Forger NFT owners can generate an extra next-generation NFT and collect an extra forging fee every year per Forger NFT (up to a 100% increase in forging capacity for entities with `forgePotential == 1`), inflating the supply of forged entities, skewing pre-airdrop entropy allocations in `Airdrop`, and accelerating generation increments.

### Remediation

Use a strict inequality (`<`) when checking `forgingCounts[tokenId]` against `forgePotential` in `EntityForging.listForForging`.

```solidity
    require(
      forgePotential > 0 && forgingCounts[tokenId] < forgePotential,
      'Entity has reached its forging limit'
    );
```

## Finding 3: Unrefunded Excess ETH in `EntityForging.forgeWithListed` Permanently Locks Surplus Payments

### Synopsis

In `EntityForging.forgeWithListed`, the payment validation checks `msg.value >= forgingFee` instead of requiring an exact match or refunding the surplus. Because fee distribution only forwards `forgingFee` and the contract lacks an ETH withdrawal mechanism, any excess ETH sent by the caller is permanently locked in the contract.

### Technical details

In `contracts/EntityForging/EntityForging.sol`, `forgeWithListed` validates the caller's payment on lines 125–126:

```solidity
uint256 forgingFee = _forgerListingInfo.fee;
require(msg.value >= forgingFee, 'Insufficient fee for forging');
```

Subsequently, on lines 146–159, the contract calculates and distributes the fee shares strictly based on `forgingFee` rather than `msg.value`:

```solidity
uint256 devFee = forgingFee / taxCut;
uint256 forgerShare = forgingFee - devFee;
...
(bool success, ) = nukeFundAddress.call{ value: devFee }('');
require(success, 'Failed to send to NukeFund');
(bool success_forge, ) = forgerOwner.call{ value: forgerShare }('');
require(success_forge, 'Failed to send to Forge Owner');
```

Because `devFee + forgerShare == forgingFee`, any surplus amount (`msg.value - forgingFee`) remains in the `EntityForging` contract balance. Unlike `EntityTrading.buyNFT` (which enforces `msg.value == listing.price`), `EntityForging.forgeWithListed` allows overpayments without refunding the difference (`msg.value - forgingFee`) to `msg.sender`. Furthermore, `EntityForging` contains no function to sweep or recover trapped ETH, resulting in permanent loss of the excess funds.

### Assumptions and preconditions

1. A valid forger token is listed in `EntityForging` with a fee of `forgingFee`.
2. A user calls `forgeWithListed` with `msg.value > forgingFee` (either due to user/frontend error or because the forger token owner cancelled and relisted the token at a lower fee before the user's transaction was mined).

### Proof of concept

1. User A owns a valid forger token (`forgerTokenId`) and lists it via `listForForging(forgerTokenId, 0.5 ether)`.
2. User B owns a compatible merger token (`mergerTokenId`) of the same generation and calls `forgeWithListed(forgerTokenId, mergerTokenId)` while supplying `msg.value = 1.0 ether` (e.g., anticipating a higher fee or after User A updated the listing fee down from `1.0 ether` to `0.5 ether`).
3. Line 126 checks `1.0 ether >= 0.5 ether`, which passes.
4. Lines 146–159 distribute `0.05 ether` (`devFee`) to `nukeFundAddress` and `0.45 ether` (`forgerShare`) to User A.
5. The remaining `0.5 ether` (`msg.value - forgingFee`) stays in `address(entityForging)` with no way for User B or the contract owner to recover it.

### Impact assessment

- **Severity:** Medium
- **Direct Technical Effect:** Any ETH sent in excess of `forgingFee` during `forgeWithListed` is uncollected by recipients and unrefunded to the payer, remaining permanently frozen in the `EntityForging` contract.
- **Business Implications:** Users who accidentally overpay or whose transactions execute after a listing fee reduction suffer a direct, irreversible loss of funds.

### Remediation

Enforce an exact payment check (`msg.value == forgingFee`) to match the pattern used in `EntityTrading.buyNFT`, or refund any excess ETH to `msg.sender`.

Preferred fix in `contracts/EntityForging/EntityForging.sol`:

```solidity
    uint256 forgingFee = _forgerListingInfo.fee;
    require(msg.value == forgingFee, 'Insufficient fee for forging');
```

## Finding 4: Unsafe External Payout Settlement and CEI Violation in `EntityForging.forgeWithListed` Enables Payout DoS and Cross-Contract Reentrancy-Guard Conflicts

### Synopsis

In `EntityForging.forgeWithListed`, ETH payouts to `nukeFundAddress` and the untrusted `forgerOwner` are executed via low-level `.call` after minting the forged NFT (`nftContract.forge`) and before cancelling the forging listing (`_cancelListingForForging`), exposing an incomplete state transition, causing cross-contract `nonReentrant` conflicts, and enabling recipient-dependent DoS.

### Technical details

In `contracts/EntityForging/EntityForging.sol`, `forgeWithListed` performs external interactions before finalizing its internal listing state and relies on synchronous push-payment calls to settle fees:

```solidity
    address payable forgerOwner = payable(nftContract.ownerOf(forgerTokenId));

    uint256 newTokenId = nftContract.forge(
      msg.sender,
      forgerTokenId,
      mergerTokenId,
      ''
    );
    (bool success, ) = nukeFundAddress.call{ value: devFee }('');
    require(success, 'Failed to send to NukeFund');
    (bool success_forge, ) = forgerOwner.call{ value: forgerShare }('');
    require(success_forge, 'Failed to send to Forge Owner');

    // Cancel listed forger nft
    _cancelListingForForging(forgerTokenId);
```

This settlement order introduces a critical external-call safety flaw at line 158 (`forgerOwner.call{ value: forgerShare }('')`):

1. **Callback Exposure of Incomplete State Transitions & Cross-Contract Reentrancy-Guard Conflict:**
   When `forgerOwner.call{ value: forgerShare }('')` transfers ETH on line 158, control is handed to `forgerOwner`'s `receive()`/`fallback()` hook before `_cancelListingForForging(forgerTokenId)` is called on line 162. At this point, `listings[listedTokenIds[forgerTokenId]].isListed` is still `true`, while `EntityForging`'s `ReentrancyGuard` is in the `_ENTERED` state (and `TraitForgeNft`'s `ReentrancyGuard` has already exited `_ENTERED` after `nftContract.forge` returned).
   If `forgerOwner` (such as a smart contract wallet or automated vault reacting to the fee payout) transfers `forgerTokenId` or lists it via `EntityTrading.listNFTForSale`, `TraitForgeNft._beforeTokenTransfer` (lines 381–392) inspects `entityForgingContract.getListings(listedId)`, sees `listing.isListed == true`, and calls `entityForgingContract.cancelListingForForging(forgerTokenId)`. Because `EntityForging.cancelListingForForging` is marked `nonReentrant` (line 179), the cross-contract call reverts with `"ReentrancyGuard: reentrant call"`.

2. **Recipient-Dependent Payout Denial of Service & Post-Mint Selective Revert:**
   Because `forgerOwner` is an arbitrary NFT owner and `forgeWithListed` enforces `require(success_forge, 'Failed to send to Forge Owner')` after `nftContract.forge(...)` has already minted `newTokenId`:
   - Any contract `forgerOwner` that lacks a payable `receive()`/`fallback()` (or exceeds gas/reverts on ETH receipt) can list a token via `listForForging`, creating a listing that permanently reverts whenever any user attempts to forge with it.
   - A malicious contract `forgerOwner` receives control in `receive()` after `nftContract.forge` has already minted `newTokenId` and recorded its entropy and generation state in `TraitForgeNft`, allowing `forgerOwner` to inspect the post-mint state (or caller identity) and selectively revert the transaction to grief callers or manipulate forging outcomes.

### Assumptions and preconditions

1. `forgerTokenId` is owned by a smart contract `forgerOwner` and is listed via `EntityForging.listForForging`.
2. A user holding a compatible `mergerTokenId` calls `EntityForging.forgeWithListed(forgerTokenId, mergerTokenId)` with `msg.value >= forgingFee`.
3. Either `forgerOwner`'s `receive()` hook attempts to transfer/list `forgerTokenId` upon receiving the payout, or `forgerOwner` reverts (intentionally or due to lacking a payable fallback) when receiving ETH at line 158.

### Proof of concept

1. A contract `ForgerVault` owns `forgerTokenId` and calls `entityForging.listForForging(forgerTokenId, 1 ether)`.
2. `listings[listedTokenIds[forgerTokenId]].isListed` is set to `true`.
3. A user owning `mergerTokenId` calls `entityForging.forgeWithListed{value: 1 ether}(forgerTokenId, mergerTokenId)`.
4. `EntityForging.forgeWithListed` enters the `nonReentrant` lock (`_status = _ENTERED`), increments forging counts, calls `nftContract.forge(...)` (which mints `newTokenId` and exits `TraitForgeNft`'s `nonReentrant` lock), and sends `devFee` to `nukeFundAddress`.
5. At line 158, `EntityForging` executes `forgerOwner.call{ value: forgerShare }('')`, invoking `ForgerVault.receive()` while `listings[listedTokenIds[forgerTokenId]].isListed` is still `true`.
6. **Scenario A (Cross-Contract Reentrancy-Guard Conflict):** Inside `ForgerVault.receive()`, the vault transfers `forgerTokenId` (e.g., `nftContract.transferFrom(address(this), treasury, forgerTokenId)` or `entityTrading.listNFTForSale(forgerTokenId, price)`). `TraitForgeNft._beforeTokenTransfer` sees `listing.isListed == true` and calls `entityForging.cancelListingForForging(forgerTokenId)`, which reverts with `"ReentrancyGuard: reentrant call"`, causing `forgerOwner.call` to return `success_forge = false` and reverting `forgeWithListed`.
7. **Scenario B (Recipient-Dependent DoS / Selective Revert):** Inside `ForgerVault.receive()`, the contract inspects the newly minted `newTokenId` on `nftContract` (or has no payable `receive` function) and reverts, causing line 159 `require(success_forge, 'Failed to send to Forge Owner')` to revert the entire `forgeWithListed` transaction.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Incomplete state transitions are exposed during external ETH settlement callbacks, causing legitimate contract-owned forger workflows that move or delist the NFT upon payout to fail with a cross-contract `ReentrancyGuard` revert, and allowing untrusted or malicious `forgerOwner` contracts to DoS `forgeWithListed` or selectively revert unfavorable forging results after `nftContract.forge` has executed.
- **Business Impact:** Disrupts the core forging marketplace, wastes caller gas, allows poisoned listings to block forging, and enables post-mint outcome manipulation.

### Remediation

Follow the Checks-Effects-Interactions (CEI) pattern by cancelling the forging listing (`_cancelListingForForging(forgerTokenId)`) before making external calls, and isolate or defer untrusted recipient payouts (pull-over-push pattern) so `forgerOwner` cannot cause `forgeWithListed` to revert:

```solidity
    forgingCounts[mergerTokenId]++;
    require(
      mergerForgePotential > 0 &&
        forgingCounts[mergerTokenId] <= mergerForgePotential,
      'forgePotential insufficient'
    );

    uint256 devFee = forgingFee / taxCut;
    uint256 forgerShare = forgingFee - devFee;
    address payable forgerOwner = payable(nftContract.ownerOf(forgerTokenId));

    // Cancel listed forger nft before external interactions (CEI)
    _cancelListingForForging(forgerTokenId);

    uint256 newTokenId = nftContract.forge(
      msg.sender,
      forgerTokenId,
      mergerTokenId,
      ''
    );

    uint256 newEntropy = nftContract.getTokenEntropy(newTokenId);

    (bool success, ) = nukeFundAddress.call{ value: devFee }('');
    require(success, 'Failed to send to NukeFund');
    (bool success_forge, ) = forgerOwner.call{ value: forgerShare }('');
    require(success_forge, 'Failed to send to Forge Owner');

    emit EntityForged(
      newTokenId,
      forgerTokenId,
      mergerTokenId,
      newEntropy,
      forgingFee
    );

    return newTokenId;
```

## Finding 5: Push-payment denial of service in EntityForging.forgeWithListed lets a reverting forger owner block settlement

### Synopsis

`EntityForging.forgeWithListed` synchronously pushes proceeds to an owner-controlled address. A contract owner can reject ETH, reverting the entire forging transaction and leaving its listing active, thereby making an apparently valid listing unusable while repeatedly causing prospective mergers to waste gas.

### Technical details

The forger payment recipient is obtained from `nftContract.ownerOf(forgerTokenId)` and therefore may be an arbitrary smart contract. After minting the new entity, `forgeWithListed` sends `forgerShare` to that address using a low-level call.

If the recipient’s `receive` or fallback function reverts, exhausts the forwarded gas, or otherwise rejects the payment, the call reports failure. The subsequent success requirement reverts the entire forging transaction.

Because EVM reverts are atomic:

- The newly forged NFT is not retained.
- Both forging-counter updates are rolled back.
- The earlier fee transfer is rolled back.
- The forger’s listing remains active because cancellation occurs later.
- Every subsequent attempt against the same listing can be forced to revert identically.

The `nonReentrant` modifier does not mitigate this attack because the recipient does not need to reenter; it only needs to reject ETH. The listing owner controls whether settlement succeeds, while the merger pays the reverted transaction’s gas cost.

### Assumptions and preconditions

- The attacker controls a contract capable of owning and listing a valid forger NFT.
- The attacker contract rejects ETH received from `EntityForging`.
- The listing satisfies the generation, entropy, fee, and quota requirements.
- A victim owns an otherwise valid merger NFT and submits the required forging fee.
- The NFT forging operation and other protocol-controlled operations would otherwise succeed.
- The attacker leaves the malicious listing active.

### Proof of concept

1. Deploy a contract that can list an NFT but rejects all native-currency payments:

```solidity
contract RejectingForgerOwner {
    EntityForging public immutable forging;

    constructor(EntityForging forging_) {
        forging = forging_;
    }

    function list(uint256 tokenId, uint256 fee) external {
        forging.listForForging(tokenId, fee);
    }

    receive() external payable {
        revert("payment rejected");
    }
}
```

2. Transfer a valid forger NFT to this contract.
3. Call `list` through the attacker contract, creating an active forging listing.
4. Have a victim who owns a valid same-generation merger call `forgeWithListed` with sufficient ETH.
5. Observe that the NFT forging call is initially reached, but settlement reverts when the forger proceeds are sent to the attacker contract.
6. Verify that the victim receives no new NFT, the forging counters remain unchanged, and the listing remains active.
7. Repeat the victim transaction and observe the same revert and additional gas loss.

### Impact assessment

**Severity: Medium.**

A malicious listing owner can make its listing indefinitely unforgeable while preserving the appearance of an active, valid listing. Prospective mergers lose transaction fees each time they attempt to use it. The same defect also makes forging incompatible with legitimate smart-contract wallets that cannot receive ETH through an unrestricted fallback call.

A single listing does not necessarily halt all protocol forging, but widespread malicious listings or commonly used non-payable custody wallets can materially degrade marketplace availability, user trust, and protocol usability.

### Remediation

Use pull-based payments. Credit the forger owner during settlement and allow that owner to withdraw later to a chosen payable recipient. Failure to receive proceeds must affect only the withdrawal, not forging.

```solidity
mapping(address => uint256) public pendingForgerProceeds;
```

```solidity
pendingForgerProceeds[forgerOwner] += forgerShare;
```

```solidity
function withdrawForgerProceeds(
    address payable recipient
) external nonReentrant {
    require(recipient != address(0), "Invalid recipient");

    uint256 amount = pendingForgerProceeds[msg.sender];
    require(amount != 0, "No proceeds");

    pendingForgerProceeds[msg.sender] = 0;

    (bool success, ) = recipient.call{value: amount}("");
    require(success, "Withdrawal failed");
}
```

As an alternative, failed push payments can be converted into escrowed balances rather than reverting settlement. Merely limiting call gas or replacing the low-level call with `transfer` does not fix the denial of service.

## Finding 6: Synchronous payouts in NukeFund.receive can deny all fee deposits

### Synopsis

`NukeFund.receive` synchronously forwards every deposit’s fee share to a lifecycle-selected beneficiary and requires that callback to succeed. Any legitimate recipient or downstream integration revert therefore rolls back the deposit, blocking fund growth and upstream minting, trading, or forging operations that route ETH through `NukeFund`.

### Technical details

Depending on the airdrop lifecycle, `receive()` forwards `devShare` to `devAddress`, `owner()`, or `daoAddress`. Each transfer executes the recipient’s payable callback with all remaining gas, and each returned `success` value is immediately required to be true.

Consequently, receiving ETH is atomically coupled to arbitrary recipient logic. A recipient does not need to be malicious: it may reject zero-value payments, be temporarily paused, lack a payable callback, or depend on another integration that reverts. Any such failure causes the corresponding `require(success)` to revert the entire `NukeFund.receive()` execution.

The revert rolls back the preceding `fund` increase and propagates to the upstream operation sending ETH. This affects protocol operations such as marketplace purchases and forging because their transfers to `NukeFund` also require success.

A concrete failure exists when `daoAddress` is the provided `DAOFund` contract. `DAOFund.receive()` rejects `msg.value == 0`, while `NukeFund` can calculate `devShare == 0` through integer division and nevertheless call `daoAddress` with zero value. Once DAO funding is enabled, that payment deterministically reverts.

The root cause is the mandatory synchronous push payment, not the correctness or honesty of the configured beneficiary.

### Assumptions and preconditions

- The applicable lifecycle branch selects a contract beneficiary rather than an always-payable EOA.
- The correctly configured beneficiary has a legitimate reachable revert condition, such as rejecting zero-value transfers or depending on a temporarily failing downstream integration.
- For the deterministic zero-value case, DAO funding is enabled, `daoAddress` is the provided `DAOFund`, and the calculated `devShare` is zero.
- A user can directly send ETH to `NukeFund` or invoke an upstream protocol operation that routes ETH to it.
- No malicious or dishonest owner action is required.

### Proof of concept

1. Configure the protocol normally with the provided `DAOFund` as `NukeFund.daoAddress`.
2. Progress the airdrop lifecycle until both `airdropStarted()` and `daoFundAllowed()` return true.
3. Retain the default `taxCut` of `10`.
4. Send `1 wei` with empty calldata to `NukeFund`.
5. `NukeFund` calculates `devShare = 1 / 10 = 0`.
6. The DAO branch calls `daoAddress` with zero ETH.
7. `DAOFund.receive()` reverts because it requires `msg.value > 0`.
8. The low-level call returns `success == false`, and the highlighted success check reverts `NukeFund.receive()`.
9. Observe that the deposit, the tentative `fund` increase, and all events are rolled back.
10. The same propagation can be observed through an upstream operation that sends a zero or sufficiently small contribution to `NukeFund`; the entire purchase, forge, or similar operation reverts.

A recipient failure at larger values—such as a downstream swap failure—produces the same result for every protocol payment while that failure persists.

### Impact assessment

**Severity: Medium.**

The vulnerability permits denial of service against ETH deposits and every atomic protocol action that must contribute ETH to `NukeFund`. Users can lose transaction fees, NFT purchases or forging operations can become unavailable, and the honeypot fund can stop growing.

The deterministic zero-share case affects low-value payments once DAO funding is active. More broadly, a persistent failure in a configured recipient’s callback or downstream integration can halt all affected-value deposits until the recipient recovers or the owner changes its address. There is no direct theft, but the availability impact can affect core economic functionality.

### Remediation

Replace synchronous beneficiary pushes with accrued pull payments. `receive()` should only account for the beneficiary’s share and complete the deposit. A separate function can attempt payment without making future deposits dependent on recipient behavior. The release function may be permissionless because funds are always sent to the recorded beneficiary.

```solidity
mapping(address => uint256) public pendingShares;

event ShareAccrued(address indexed beneficiary, uint256 amount);

receive() external payable {
    uint256 devShare = msg.value / taxCut;
    uint256 remainingFund = msg.value - devShare;

    fund += remainingFund;

    address beneficiary;
    if (!airdropContract.airdropStarted()) {
        beneficiary = devAddress;
    } else if (!airdropContract.daoFundAllowed()) {
        beneficiary = owner();
    } else {
        beneficiary = daoAddress;
    }

    if (devShare != 0) {
        pendingShares[beneficiary] += devShare;
        emit ShareAccrued(beneficiary, devShare);
    }

    emit FundReceived(msg.sender, msg.value);
    emit FundBalanceUpdated(fund);
}

function releaseShare(address payable beneficiary) external nonReentrant {
    uint256 amount = pendingShares[beneficiary];
    require(amount != 0, "No pending share");

    pendingShares[beneficiary] = 0;

    (bool success, ) = beneficiary.call{value: amount}("");
    require(success, "ETH send failed");

    emit DevShareDistributed(amount);
}
```

As an alternative, failed push payments can be caught and converted into pending balances, but the beneficiary’s share must remain reserved separately from `fund` so NFT claims cannot consume it.
