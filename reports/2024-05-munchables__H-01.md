# Audit report — 2024-05-munchables/H-01

## Finding 1: Inconsistent Unlock Timestamp Calculation in `LockManager.setLockDuration` Allows Players to Reduce Lockup Time and Prematurely `unlock` Funds

### Synopsis

In `LockManager.setLockDuration`, the validation guarding against lock duration reduction compares `uint32(block.timestamp) + uint32(_duration)` against `lockedToken.unlockTime`, but subsequently updates `lockedToken.unlockTime` to `lastLockTime + uint32(_duration)`. This discrepancy allows any user to shorten their existing `unlockTime` and prematurely withdraw locked tokens via `LockManager.unlock`.

### Technical details

When a player locks tokens in `LockManager._lock`, `lockedToken.lastLockTime` is set to `uint32(block.timestamp)` and `lockedToken.unlockTime` is set to `uint32(block.timestamp) + uint32(_lockDuration)`. The player is prohibited from withdrawing those tokens in `LockManager.unlock` while `lockedToken.unlockTime > uint32(block.timestamp)` (line 410).

To prevent players from shortening active locks, `LockManager.setLockDuration` checks that the new unlock timestamp is not earlier than the existing `unlockTime` (lines 256–261):

```solidity
if (
    uint32(block.timestamp) + uint32(_duration) <
    lockedTokens[msg.sender][tokenContract].unlockTime
) {
    revert LockDurationReducedError();
}

uint32 lastLockTime = lockedTokens[msg.sender][tokenContract]
    .lastLockTime;
lockedTokens[msg.sender][tokenContract].unlockTime =
    lastLockTime +
    uint32(_duration);
```

Because the validation uses `uint32(block.timestamp) + uint32(_duration)` whereas the state update computes `lastLockTime + uint32(_duration)`, any time elapsed since `lastLockTime` (`Δt = block.timestamp - lastLockTime > 0`) creates a gap between the checked timestamp and the stored `unlockTime`:

1. After half of the original lock duration has elapsed (`block.timestamp - lastLockTime >= oldDuration / 2`), calling `setLockDuration(oldDuration / 2)` passes the check (`block.timestamp + oldDuration / 2 >= lastLockTime + oldDuration`) and sets `unlockTime = lastLockTime + oldDuration / 2 <= block.timestamp`, making the funds immediately unlockable in `unlock`.
2. More generally, as soon as at least 1 second has passed since `lastLockTime` (`Δt = block.timestamp - lastLockTime >= 1`), a player can repeatedly call `setLockDuration(unlockTime - block.timestamp)`. Each call passes the check and decreases `unlockTime` by `Δt`. By iterating until `_duration == 0`, the player reduces `unlockTime` to `lastLockTime <= block.timestamp` and can immediately call `unlock` to withdraw all locked tokens.

### Assumptions and preconditions

- The contracts are not paused (`_paused == false`).
- A registered player has locked tokens in `LockManager` with an active `unlockTime` in the future.
- At least 1 second has elapsed since `lockedToken.lastLockTime` (`block.timestamp > lockedToken.lastLockTime`).

### Proof of concept

1. Alice locks `1 ETH` at timestamp `t = 1000` with `_lockDuration = 30 days` (`2,592,000` seconds).
   - `lockedToken.lastLockTime = 1000`
   - `lockedToken.unlockTime = 2,593,000`
2. At `t = 1,297,000` (15 days later, with 15 days of lockup still remaining), Alice calls `lockManager.setLockDuration(15 days)` (`1,296,000` seconds).
3. In `setLockDuration`:
   - The check evaluates `1,297,000 + 1,296,000 < 2,593,000` (`2,593,000 < 2,593,000`), which is `false` and does not revert.
   - `lockedToken.unlockTime` is updated to `lastLockTime + _duration = 1000 + 1,296,000 = 1,297,000`.
   *(Note: If Alice instead acts at `t = 1000 + 1 hours`, she can call `setLockDuration` 30 times in a batch, reducing `_duration` by `1 hours` per call down to `0`, setting `unlockTime = 1000` after just 1 hour.)*
4. Alice immediately calls `lockManager.unlock(address(0), 1 ether)` at `t = 1,297,000`.
5. In `unlock`, `lockedToken.unlockTime > uint32(block.timestamp)` evaluates `1,297,000 > 1,297,000` (`false`), allowing Alice to withdraw her `1 ETH` after only half (or a tiny fraction) of the required lock duration while retaining the minted Munchable NFTs and harvested schnibbles.

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** Any player can bypass the configured `minLockDuration` and their previously committed lockup period, reducing `lockedToken.unlockTime` to `<= block.timestamp` and withdrawing locked funds early via `LockManager.unlock`.
- **Business Implications:** Directly violates a core protocol invariant stated in the specification (*"people cannot reduce lockup times that are previously set"*). Users can mint lockdrop NFTs for free without keeping their capital locked for the required duration, undermining the Blast yield generation and point-farming economics of the protocol.

### Remediation

Ensure the validation in `setLockDuration` and the `unlockTime` assignment use the same base timestamp (`lastLockTime + uint32(_duration)`):

```solidity
            if (lockedTokens[msg.sender][tokenContract].quantity > 0) {
                uint32 lastLockTime = lockedTokens[msg.sender][tokenContract]
                    .lastLockTime;
                // check they are not setting lock time before current unlocktime
                if (
                    lastLockTime + uint32(_duration) <
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
                    revert LockDurationReducedError();
                }

                lockedTokens[msg.sender][tokenContract].unlockTime =
                    lastLockTime +
                    uint32(_duration);
            }
```

## Finding 2: Unrestricted `LockManager.lockOnBehalf` allows any caller to reset a player's lock state, wipe remainders, and lock funds indefinitely

### Synopsis

`LockManager.lockOnBehalf` lacks access control and minimum quantity validation, allowing any external account to lock zero (or dust) tokens on behalf of an arbitrary player. This resets the target player's `lastLockTime` and `unlockTime`, forces a reward harvest checkpoint, and zeroes out their accumulated NFT `remainder` outside the lockdrop window.

### Technical details

In `src/managers/LockManager.sol`, `lockOnBehalf` (lines 275–294) is an `external` function without caller restrictions that accepts an arbitrary `_onBehalfOf` address and delegates directly to `_lock(_tokenContract, _quantity, tokenOwner, lockRecipient)`:

```solidity
function lockOnBehalf(
    address _tokenContract,
    uint256 _quantity,
    address _onBehalfOf
)
    external
    payable
    notPaused
    onlyActiveToken(_tokenContract)
    onlyConfiguredToken(_tokenContract)
    nonReentrant
{
    address tokenOwner = msg.sender;
    address lockRecipient = msg.sender;
    if (_onBehalfOf != address(0)) {
        lockRecipient = _onBehalfOf;
    }

    _lock(_tokenContract, _quantity, tokenOwner, lockRecipient);
}
```

Although `lockOnBehalf` is intended to be used by `MigrationManager._migrateNFTs` to lock migrated funds on behalf of a user, any unprivileged caller can invoke it for any registered main account `_onBehalfOf` (even with `_quantity == 0` or `1 wei`).

When `_lock` executes for `_lockRecipient`:
1. **Indefinite Lock Extension (Denial of Service on `unlock`)**: Lines 381–384 unconditionally overwrite the recipient's lock timestamps:
   ```solidity
   lockedToken.lastLockTime = uint32(block.timestamp);
   lockedToken.unlockTime = uint32(block.timestamp) + uint32(_lockDuration);
   ```
   Because `unlock()` reverts with `TokenStillLockedError` whenever `lockedToken.unlockTime > uint32(block.timestamp)`, an attacker can repeatedly call `lockOnBehalf(_tokenContract, 0, victim)` before the lock expires to perpetually push `unlockTime` into the future at only the cost of gas.
2. **Cross-Deposit Remainder Wipe & Checkpoint Manipulation**: In `_lock`, `remainder` defaults to `0` (`uint256 remainder;`) and is only computed when `block.timestamp` is within `[lockdrop.start, lockdrop.end]` and `msg.sender != address(migrationManager)`. It is then written unconditionally to `lockedToken.remainder = remainder;` (line 379), wiping any previously accumulated `lockedToken.remainder` if called outside the lockdrop window, while also triggering `accountManager.forceHarvest(_lockRecipient)` (line 341).

### Assumptions and preconditions

1. The target player (`_onBehalfOf`) is a registered main account in `AccountManager` (`_player.registrationDate != 0` and `_mainAccount == _lockRecipient`).
2. `_tokenContract` is an active, configured token in `LockManager`.
3. The contract is not paused, and the target player's `lockDuration` (or `lockdrop.minLockDuration`) is valid for the current timestamp.

### Proof of concept

1. Alice (a registered player) locks `10 ETH` via `LockManager.lock{value: 10 ether}(address(0), 10 ether)` with a lock duration of `30 days`. Her `lockedTokens[alice][address(0)].unlockTime` is set to `T + 30 days`.
2. Just before `T + 30 days`, an attacker calls `LockManager.lockOnBehalf(address(0), 0, alice)` with `msg.value = 0`.
3. Inside `_lock`, `msg.value == _quantity` (`0 == 0`) succeeds, `accountManager.forceHarvest(alice)` is called, and `lockedTokens[alice][address(0)].unlockTime` is updated to `(T + 30 days) + 30 days`.
4. If the lockdrop period has ended, `remainder` remains `0` and overwrites `lockedTokens[alice][address(0)].remainder = 0`.
5. At `T + 30 days`, Alice calls `LockManager.unlock(address(0), 10 ether)`, which reverts with `TokenStillLockedError` because `unlockTime` was pushed forward by another 30 days. The attacker can repeat this indefinitely to keep Alice's funds locked forever.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: Any user's `unlockTime` and `lastLockTime` can be reset at will by an external attacker at zero token cost (using `_quantity = 0`), and any stored `lockedToken.remainder` is erased if called outside the lockdrop window.
- **Business Impact**: Directly violates the core protocol invariant that user funds cannot be locked forever. Attackers can permanently freeze all locked ETH, WETH, and USDB belonging to any player in `LockManager`.

### Remediation

Restrict `lockOnBehalf` so that only `migrationManager` is permitted to lock tokens on behalf of another user (or require explicit player authorization/opt-in before allowing third-party locks on their behalf):

```solidity
    function lockOnBehalf(
        address _tokenContract,
        uint256 _quantity,
        address _onBehalfOf
    )
        external
        payable
        notPaused
        onlyActiveToken(_tokenContract)
        onlyConfiguredToken(_tokenContract)
        nonReentrant
    {
        address tokenOwner = msg.sender;
        address lockRecipient = msg.sender;
        if (_onBehalfOf != address(0)) {
            if (
                _onBehalfOf != msg.sender &&
                msg.sender != address(migrationManager)
            ) {
                revert LockManagerInvalidCallError();
            }
            lockRecipient = _onBehalfOf;
        }

        _lock(_tokenContract, _quantity, tokenOwner, lockRecipient);
    }
```

## Finding 3: Precision Loss and Reward Griefing via Unconditional `forceHarvest` in `LockManager._lock`

### Synopsis

In `LockManager._lock`, `accountManager.forceHarvest(_lockRecipient)` is invoked unconditionally on every lock call, including zero-quantity `lockOnBehalf` calls. Because `AccountManager._harvest` truncates sub-day reward remainders via integer division while resetting `lastHarvestDate`, fragmented accrual settlements cause cumulative precision loss and enable griefing of accrued schnibbles.

### Technical details

When a player locks tokens via `LockManager.lock` or `LockManager.lockOnBehalf`, the internal function `LockManager._lock` executes the reward checkpoint at line 341:

```solidity
// they will receive schnibbles at the new rate since last harvest if not for force harvest
accountManager.forceHarvest(_lockRecipient);
```

`AccountManager.forceHarvest(_lockRecipient)` invokes `AccountManager._harvest(_caller)` (lines 372–387 of `src/managers/AccountManager.sol`), which computes and settles accrued schnibbles as follows:

```solidity
(uint256 dailySchnibbles, uint256 bonus) = getDailySchnibbles(_caller);
dailySchnibbles += ((dailySchnibbles * bonus) / 1e18);

uint256 secondsToClaim = block.timestamp -
    players[_caller].lastHarvestDate;
uint256 harvestedSchnibbles = (dailySchnibbles * secondsToClaim) /
    1 days;

players[_caller].unfedSchnibbles += harvestedSchnibbles;
players[_caller].lastHarvestDate = uint32(block.timestamp);
```

This checkpoint mechanism suffers from two compounding flaws when triggered from `LockManager._lock`:

1. **Unconditional checkpoint reset with integer truncation:** Every call to `forceHarvest(_lockRecipient)` divides `dailySchnibbles * secondsToClaim` by `1 days` (`86,400` seconds) and unconditionally updates `players[_lockRecipient].lastHarvestDate = uint32(block.timestamp)`. Any remainder `(dailySchnibbles * secondsToClaim) % 1 days` is permanently discarded instead of being carried forward. If `dailySchnibbles * secondsToClaim < 1 days`, `harvestedSchnibbles` rounds down to `0` while `lastHarvestDate` is still advanced to `block.timestamp`, erasing all accrual for that interval.
2. **Lack of non-zero quantity validation on `_lock` / `lockOnBehalf`:** `LockManager._lock` does not require `_quantity > 0` before calling `accountManager.forceHarvest(_lockRecipient)`. For any active token (including native ETH where `_tokenContract == address(0)` and `msg.value == 0` matches `_quantity == 0`), anyone can invoke `lockOnBehalf(_tokenContract, 0, _lockRecipient)` for an arbitrary registered `_lockRecipient`.

Consequently, each fragmented settlement interval $i$ of duration $\Delta t_i$ yields $\lfloor \frac{R \cdot \Delta t_i}{86400} \rfloor$ instead of settling over the full duration $\sum \Delta t_i$, causing systematic precision loss across multiple lock operations and allowing third parties to repeatedly reset a target player's `lastHarvestDate` via zero-amount `lockOnBehalf` calls.

### Assumptions and preconditions

1. The contracts are not paused (`_paused == false`), and at least one token (such as native ETH at `address(0)` or an ERC20 token) is configured and active in `LockManager`.
2. The target `_lockRecipient` is a registered main account in `AccountManager` (`_player.registrationDate != 0` and `_mainAccount == _lockRecipient`) with a valid `lockDuration` configuration when within the lockdrop window.
3. The target player has locked tokens accruing `dailySchnibbles` over time since `lastHarvestDate`.

### Proof of concept

1. A registered player (`alice`) locks tokens in `LockManager`, establishing a non-zero `dailySchnibbles` rate $R$ and initializing `players[alice].lastHarvestDate = t_0`.
2. After $\Delta t$ seconds have elapsed ($t_1 = t_0 + \Delta t$), another account (or `alice` performing multiple small locks) invokes `lockOnBehalf(address(0), 0, alice)` with `msg.value = 0`.
3. Inside `LockManager._lock`, all checks pass (`_mainAccount == alice`, `_player.registrationDate != 0`, `msg.value == _quantity == 0`), and line 341 calls `accountManager.forceHarvest(alice)`.
4. `AccountManager._harvest(alice)` computes `harvestedSchnibbles = (R * (t_1 - t_0)) / 86400`. The remainder `(R * (t_1 - t_0)) % 86400` is truncated and lost, and `players[alice].lastHarvestDate` is overwritten with `t_1`.
5. If `R * (t_1 - t_0) < 86400` (for instance, when `dailySchnibbles` is small or calls occur across short intervals), `harvestedSchnibbles` evaluates to `0` while `lastHarvestDate` advances to `t_1`, resulting in total loss of accrued rewards for the elapsed period.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Fragmented calls to `accountManager.forceHarvest(_lockRecipient)` during `_lock` strip sub-day division remainders on every checkpoint update and reset `lastHarvestDate`. Because `_quantity == 0` is permitted on `lockOnBehalf`, any external caller can trigger zero-value locks on behalf of a player to fragment their accrual intervals and erode their earned `unfedSchnibbles` (and also reset `lastLockTime` / `unlockTime`).
- **Business Impact:** Players receive fewer `unfedSchnibbles` than entitled to by their locked token value and elapsed lock time, directly degrading progression (Chonks/levels) and Munch Points generation.

### Remediation

Require a non-zero `_quantity` in `LockManager._lock` (or skip `forceHarvest` and state updates when `_quantity == 0`), and ensure that reward checkpointing only advances `lastHarvestDate` by the exact time corresponding to settled rewards or when a meaningful balance change occurs.

```solidity
        }

        if (_quantity == 0) revert InsufficientLockAmountError();

        LockedToken storage lockedToken = lockedTokens[_lockRecipient][
            _tokenContract
        ];
        ConfiguredToken storage configuredToken = configuredTokens[
            _tokenContract
        ];

        // they will receive schnibbles at the new rate since last harvest if not for force harvest
        accountManager.forceHarvest(_lockRecipient);

        // add remainder from any previous lock
        uint256 quantity = _quantity + lockedToken.remainder;
        uint256 remainder;
        uint256 numberNFTs;
        uint32 _lockDuration = playerSettings[_lockRecipient].lockDuration;
```

## Finding 4: Failure to Reset `lockedToken.remainder` in `LockManager.unlock` Allows Minting Free NFTs via Remainder Reuse

### Synopsis

`LockManager.unlock` deducts the withdrawn `_quantity` from `lockedToken.quantity` without resetting or updating `lockedToken.remainder`. A player who unlocks tokens after their lock duration expires retains their previous `remainder` balance and can reuse it in subsequent `lock` calls during an active lockdrop to mint NFTs without locking the full required token cost.

### Technical details

In `LockManager._lock` (lines 344–379), whenever a player locks tokens during an active lockdrop, the contract calculates the number of unrevealed NFTs to award by combining the newly locked `_quantity` with any leftover `lockedToken.remainder` from previous locks that did not reach a full `configuredToken.nftCost`:

```solidity
uint256 quantity = _quantity + lockedToken.remainder;
...
remainder = quantity % configuredToken.nftCost;
numberNFTs = (quantity - remainder) / configuredToken.nftCost;
...
nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
...
lockedToken.remainder = remainder;
lockedToken.quantity += _quantity;
```

However, in `LockManager.unlock` (lines 401–427), when a user withdraws their unlocked tokens:

```solidity
LockedToken storage lockedToken = lockedTokens[msg.sender][
    _tokenContract
];
if (lockedToken.quantity < _quantity)
    revert InsufficientLockAmountError();
if (lockedToken.unlockTime > uint32(block.timestamp))
    revert TokenStillLockedError();

accountManager.forceHarvest(msg.sender);

lockedToken.quantity -= _quantity;
```

`lockedToken.quantity` is decremented by `_quantity` and the underlying ETH or ERC-20 tokens are transferred back to `msg.sender`, but `lockedToken.remainder` is never cleared or adjusted. Because the tokens backing `lockedToken.remainder` have been withdrawn, the recorded residual entitlement (`lockedToken.remainder`) diverges from the actual custodied assets (`lockedToken.quantity`).

If the player locks again while a lockdrop is active, `_lock` adds the stale `lockedToken.remainder` to the new `_quantity`, crediting the player for tokens they already withdrew and minting unrevealed NFTs via `nftOverlord.addReveal` at a fraction of `configuredToken.nftCost`.

### Assumptions and preconditions

1. A token (such as ETH, WETH, or USDB) is configured and active in `LockManager` with a non-zero `nftCost` (e.g., `1 ether`).
2. A lockdrop is active whose window is longer than `minLockDuration` (or a new lockdrop is started after a user's lock expires).
3. The player is registered in `AccountManager`.

### Proof of concept

1. Assume ETH (`address(0)`) is configured with `nftCost = 1 ether`, and a 30-day lockdrop is active with `minLockDuration = 7 days`.
2. Alice calls `lock{value: 0.9 ether}(address(0), 0.9 ether)`.
   - `quantity = 0.9 ether + 0 = 0.9 ether`.
   - `numberNFTs = 0`, `remainder = 0.9 ether`.
   - `lockedToken.quantity` is set to `0.9 ether`, and `lockedToken.remainder` is set to `0.9 ether`.
3. After 7 days (`block.timestamp >= lockedToken.unlockTime`), Alice calls `unlock(address(0), 0.9 ether)`.
   - Alice receives her `0.9 ether` back.
   - `lockedToken.quantity` becomes `0`.
   - `lockedToken.remainder` remains `0.9 ether`.
4. Alice immediately calls `lock{value: 0.1 ether}(address(0), 0.1 ether)`.
   - `_lock` computes `quantity = 0.1 ether + 0.9 ether = 1.0 ether`.
   - `numberNFTs = 1`, `remainder = 0`.
   - `nftOverlord.addReveal(alice, 1)` is called, granting Alice 1 unrevealed Munchable NFT even though she only has `0.1 ether` locked in `LockManager` instead of the required `1.0 ether`.

### Impact assessment

- **Severity:** Medium / High
- **Technical Effect:** `lockedToken.remainder` persists across withdrawals, breaking accounting conservation between custodied tokens (`lockedToken.quantity`) and NFT minting progress (`lockedToken.remainder`).
- **Business Impact:** Players can repeatedly cycle the same capital across lock expirations (or across lockdrops) to mint Munchable NFTs while locking only a tiny fraction of the required `nftCost`, inflating the NFT supply and bypassing the economic requirements of the lockdrop.

### Remediation

Reset `lockedToken.remainder` to `0` (or cap/reduce it proportionally so that `lockedToken.remainder` cannot exceed the remaining `lockedToken.quantity` or is cleared upon withdrawal) in `LockManager.unlock`:

```solidity
    function unlock(
        address _tokenContract,
        uint256 _quantity
    ) external notPaused nonReentrant {
        LockedToken storage lockedToken = lockedTokens[msg.sender][
            _tokenContract
        ];
        if (lockedToken.quantity < _quantity)
            revert InsufficientLockAmountError();
        if (lockedToken.unlockTime > uint32(block.timestamp))
            revert TokenStillLockedError();

        // force harvest to make sure that they get the schnibbles that they are entitled to
        accountManager.forceHarvest(msg.sender);

        lockedToken.quantity -= _quantity;
        lockedToken.remainder = 0;

        // send token
        if (_tokenContract == address(0)) {
            payable(msg.sender).transfer(_quantity);
        } else {
            IERC20 token = IERC20(_tokenContract);
            token.transfer(msg.sender, _quantity);
        }

        emit Unlocked(msg.sender, _tokenContract, _quantity);
    }
```

## Finding 5: Unverified Token Receipt in `LockManager._lock` Credits `lockedToken.quantity` Without Checking Actual Transferred Amount

### Synopsis

In `LockManager._lock`, `lockedToken.quantity` is unconditionally incremented by the nominal `_quantity` argument without verifying the return value of `IERC20.transferFrom` or measuring the actual balance received. For tokens that return `false` on failure or charge transfer fees, positions are credited without matching asset backing, enabling pool insolvency and theft of funds.

### Technical details

When a player locks an ERC20 token via `LockManager.lock` or `LockManager.lockOnBehalf`, the internal `_lock` function executes the token transfer and updates the recipient's accounting state at lines 374–380 of `src/managers/LockManager.sol`:

```solidity
// Transfer erc tokens
if (_tokenContract != address(0)) {
    IERC20 token = IERC20(_tokenContract);
    token.transferFrom(_tokenOwner, address(this), _quantity);
}

lockedToken.remainder = remainder;
lockedToken.quantity += _quantity;
```

This settlement logic has two closely related accounting flaws:

1. **Unchecked `transferFrom` return value:** `token.transferFrom(_tokenOwner, address(this), _quantity)` is called via the raw `IERC20` interface without checking its boolean return value or using OpenZeppelin's `SafeERC20.safeTransferFrom`. Although `_lock` checks `token.allowance(_tokenOwner, address(this)) >= _quantity` at line 330, it never checks `token.balanceOf(_tokenOwner) >= _quantity`. For ERC20 tokens that return `false` instead of reverting when `_tokenOwner` has an insufficient balance, `transferFrom` silently fails while `lockedToken.quantity += _quantity` still credits the full `_quantity` to `_lockRecipient`.
2. **Disconnect between nominal `_quantity` and actual tokens received:** Even when `transferFrom` succeeds, `lockedToken.quantity += _quantity` assumes that the exact `_quantity` parameter arrived in `LockManager`. If a configured token applies a transfer fee or has nonstandard balance semantics, the contract receives fewer tokens than `_quantity`, yet credits the full `_quantity` to `lockedToken.quantity` (and uses the unadjusted `_quantity` to compute `remainder` and `numberNFTs`).

When the lock duration expires, `LockManager.unlock` allows the player to withdraw up to `lockedToken.quantity` tokens from the contract's pooled token balance (`token.transfer(msg.sender, _quantity)`), allowing the over-credited user to drain tokens deposited by other players or leaving the contract with insufficient balance to honor legitimate withdrawals.

### Assumptions and preconditions

1. The admin has configured an ERC20 token in `LockManager` via `configureToken` that either returns `false` on failed transfers instead of reverting (per the EIP-20 specification) or deducts a fee on transfer.
2. The caller (`_tokenOwner`) has registered with `AccountManager` (or specifies a registered `_onBehalfOf` main account) and has approved `LockManager` for at least `_quantity`.

### Proof of concept

1. Assume a configured ERC20 token `T` returns `false` when `transferFrom` fails due to insufficient sender balance (or charges a 5% transfer fee).
2. Honest user Alice locks `100e18` of token `T` via `LockManager.lock(address(T), 100e18)`. `LockManager` now holds `100e18` (or `95e18` in the fee-on-transfer case) of token `T`.
3. Attacker Bob holds `0` balance of token `T` (in the `returns false` token case), calls `T.approve(address(lockManager), 100e18)`, and invokes `LockManager.lock(address(T), 100e18)`.
4. Inside `_lock`:
   - `token.allowance(bob, address(lockManager))` is `100e18 >= 100e18`, passing line 330.
   - `nftOverlord.addReveal(bob, numberNFTs)` grants Bob unrevealed Munchable NFTs for `100e18` tokens.
   - `token.transferFrom(bob, address(lockManager), 100e18)` returns `false` without reverting; `0` tokens are transferred.
   - Line 380 executes `lockedToken.quantity += 100e18`, crediting Bob with `100e18` locked tokens.
5. During the lock period, Bob harvests Schnibbles via `AccountManager` based on his phantom `100e18` locked position (`getLockedWeightedValue`).
6. Once `lockedToken.unlockTime` passes, Bob calls `LockManager.unlock(address(T), 100e18)` and withdraws Alice's `100e18` tokens (or, in the fee-on-transfer case, Alice and other depositors cannot fully withdraw because total `lockedToken.quantity` exceeds `T.balanceOf(address(lockManager))`).

### Impact assessment

- **Severity:** High
- **Direct Technical Effect:** `lockedToken.quantity` becomes decoupled from the actual ERC20 token balance held by `LockManager`.
- **Business & Protocol Impact:**
  - **Theft of User Funds:** For tokens that return `false` on failure, an attacker with zero token balance can credit themselves an arbitrary `lockedToken.quantity`, mint free Munchable NFTs, farm Schnibbles, and drain all tokens of that type locked by other users upon unlock.
  - **Contract Insolvency / Frozen Funds:** For fee-on-transfer tokens, `LockManager` credits more tokens than it receives on every lock. The last users to call `unlock()` will revert due to insufficient token balance in `LockManager`, permanently locking their remaining principal.

### Remediation

Use OpenZeppelin's `SafeERC20.safeTransferFrom` to ensure failed transfers revert, and measure the contract's token balance before and after the transfer so that `lockedToken.quantity` (as well as NFT reveal calculations) only credits the exact amount of tokens received:

```solidity
        uint256 receivedQuantity = _quantity;
        if (_tokenContract != address(0)) {
            IERC20 token = IERC20(_tokenContract);
            uint256 balanceBefore = token.balanceOf(address(this));
            SafeERC20.safeTransferFrom(token, _tokenOwner, address(this), _quantity);
            receivedQuantity = token.balanceOf(address(this)) - balanceBefore;
        }

        uint256 quantity = receivedQuantity + lockedToken.remainder;
        uint256 remainder;
        uint256 numberNFTs;
        uint32 _lockDuration = playerSettings[_lockRecipient].lockDuration;

        if (_lockDuration == 0) {
            _lockDuration = lockdrop.minLockDuration;
        }
        if (
            lockdrop.start <= uint32(block.timestamp) &&
            lockdrop.end >= uint32(block.timestamp)
        ) {
            if (
                _lockDuration < lockdrop.minLockDuration ||
                _lockDuration >
                uint32(configStorage.getUint(StorageKey.MaxLockDuration))
            ) revert InvalidLockDurationError();
            if (msg.sender != address(migrationManager)) {
                remainder = quantity % configuredToken.nftCost;
                numberNFTs = (quantity - remainder) / configuredToken.nftCost;

                if (numberNFTs > type(uint16).max) revert TooManyNFTsError();

                nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
            }
        }

        lockedToken.remainder = remainder;
        lockedToken.quantity += receivedQuantity;
        lockedToken.lastLockTime = uint32(block.timestamp);
        lockedToken.unlockTime =
            uint32(block.timestamp) +
            uint32(_lockDuration);

        playerSettings[_lockRecipient].lockDuration = _lockDuration;

        emit Locked(
            _lockRecipient,
            _tokenOwner,
            _tokenContract,
            receivedQuantity,
            remainder,
            numberNFTs,
            _lockDuration
        );
```
