# Audit report — 2024-05-munchables/H-02

## Finding 1: Missing `forceHarvest` in `LockManager.setLockDuration` allows retroactive lock bonus inflation on accrued Schnibbles

### Synopsis

`LockManager.setLockDuration` updates a player's `lockDuration` setting without first calling `accountManager.forceHarvest(msg.sender)`. Because `BonusManager.getHarvestBonus` computes the harvest multiplier from the current `playerSettings.lockDuration`, players can retroactively apply the maximum lock duration bonus to all unharvested Schnibbles accrued under a shorter lock commitment.

### Technical details

In `AccountManager._harvest`, a player's accrued Schnibbles for the entire elapsed period since `players[_caller].lastHarvestDate` are calculated using the current bonus returned by `BonusManager.getHarvestBonus(_caller)`:

```solidity
(uint256 dailySchnibbles, uint256 bonus) = getDailySchnibbles(_caller);
dailySchnibbles += ((dailySchnibbles * bonus) / 1e18);

uint256 secondsToClaim = block.timestamp - players[_caller].lastHarvestDate;
uint256 harvestedSchnibbles = (dailySchnibbles * secondsToClaim) / 1 days;
```

`BonusManager.getHarvestBonus` (and `getPetBonus`) reads `playerSettings[_caller].lockDuration` from `LockManager` and passes it to `_calculateLockBonus(_settings.lockDuration)` to determine the reward multiplier.

To prevent changes in lock parameters from retroactively affecting previously accrued rewards, `LockManager._lock` (line 341) and `LockManager.unlock` (line 414) both call `accountManager.forceHarvest` before modifying lock state.

However, `LockManager.setLockDuration` (lines 245–272) updates `playerSettings[msg.sender].lockDuration = uint32(_duration)` without calling `accountManager.forceHarvest(msg.sender)`. Furthermore, for any existing locked tokens, line 265 recalculates `unlockTime` relative to the historical `lastLockTime` (`lastLockTime + uint32(_duration)`) rather than the current timestamp.

Consequently, a player can:
1. Lock tokens with the minimum lock duration (e.g., 30 days, which yields a 0% lock bonus).
2. Wait for a period of time (e.g., 90 days) without harvesting. At this point, `lastLockTime + 90 days <= block.timestamp`, meaning their tokens would remain immediately unlockable even if `_duration` is set to 90 days.
3. Call `setLockDuration(90 days)`. The check `uint32(block.timestamp) + uint32(_duration) < unlockTime` passes, `playerSettings[msg.sender].lockDuration` becomes `90 days`, and `unlockTime` becomes `lastLockTime + 90 days <= block.timestamp`.
4. Immediately call `unlock` (which triggers `forceHarvest`) or `harvest`, claiming the full 90 days of accrued Schnibbles with the maximum 90-day lock bonus (30%) while having only committed to a 30-day lock and retaining immediate liquidity.

### Assumptions and preconditions

- The protocol is not paused (`_paused == false`).
- The player is registered in `AccountManager` and has locked configured tokens via `LockManager.lock` or `LockManager.lockOnBehalf`.
- `MaxLockDuration` in `ConfigStorage` is greater than the player's initial lock duration.

### Proof of concept

1. Alice registers as a player and locks 10 ETH at timestamp `T0` with `_lockDuration = 30 days` (0% lock bonus). Her `lastLockTime` is `T0`, `unlockTime` is `T0 + 30 days`, and `lastHarvestDate` is `T0`.
2. At `T0 + 30 days`, Alice's lock expires (`block.timestamp >= unlockTime`). She now has complete freedom to withdraw her 10 ETH at any moment, bearing zero lockup risk for any subsequent days.
3. Alice leaves her tokens in the contract until `T0 + 90 days` without calling `harvest`.
4. At `T0 + 90 days`, Alice calls `LockManager.setLockDuration(90 days)`:
   - `90 days <= MaxLockDuration` (90 days) passes.
   - `playerSettings[alice].lockDuration` is updated to `90 days`.
   - `block.timestamp + 90 days >= unlockTime` passes.
   - `lockedTokens[alice][address(0)].unlockTime` is updated to `lastLockTime + 90 days = T0 + 90 days = block.timestamp`.
5. In the same block (`T0 + 90 days`), Alice calls `LockManager.unlock(address(0), 10 ether)`:
   - `lockedToken.unlockTime > uint32(block.timestamp)` is false (`T0 + 90 days > T0 + 90 days` is false), so the unlock succeeds.
   - `accountManager.forceHarvest(alice)` executes and queries `BonusManager.getHarvestBonus(alice)`, which reads `playerSettings[alice].lockDuration == 90 days` and returns the maximum 30% lock bonus (`30e16`).
   - Alice receives a 30% bonus on all 90 days of accrued Schnibbles and immediately withdraws her 10 ETH, despite only ever locking her funds for 30 days.

### Impact assessment

- **Severity**: Medium / High.
- **Technical Impact**: Unfair and retroactive inflation of harvested Schnibbles (up to the maximum lock bonus of 30%) over arbitrary unharvested time periods without enforcing the corresponding lock duration commitment.
- **Business Impact**: Dilutes the Schnibbles and downstream Munch Points / NFT progression economy for honest players who genuinely lock their capital for the full maximum duration.

### Remediation

Call `accountManager.forceHarvest(msg.sender)` at the beginning of `setLockDuration` (when the caller is a registered player) before updating `playerSettings[msg.sender].lockDuration`, so all previously accrued Schnibbles are settled at the prior lock duration rate.

```solidity
    function setLockDuration(uint256 _duration) external notPaused {
        if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
            revert MaximumLockDurationError();

        (
            address _mainAccount,
            MunchablesCommonLib.Player memory _player
        ) = accountManager.getPlayer(msg.sender);
        if (_mainAccount == msg.sender && _player.registrationDate != 0) {
            accountManager.forceHarvest(msg.sender);
        }

        playerSettings[msg.sender].lockDuration = uint32(_duration);
        // update any existing lock
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            address tokenContract = configuredTokenContracts[i];
            if (lockedTokens[msg.sender][tokenContract].quantity > 0) {
                // check they are not setting lock time before current unlocktime
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
            }
        }

        emit LockDuration(msg.sender, _duration);
    }
```

## Finding 2: Inconsistent `unlockTime` Calculation in `LockManager.setLockDuration` Allows Players to Reduce Existing Lockup Times and Unlock Early

### Synopsis

In `LockManager.setLockDuration`, the check preventing lock duration reduction compares the current `unlockTime` against `uint32(block.timestamp) + uint32(_duration)`, whereas the state update sets the new `unlockTime` to `lastLockTime + uint32(_duration)`. Once time elapses after locking, players can supply a shorter `_duration` to reduce their existing `unlockTime` and unlock funds prematurely.

### Technical details

In `src/managers/LockManager.sol`, `setLockDuration(uint256 _duration)` is intended to allow a player to update their configured `lockDuration` and recalculate `unlockTime` for all currently locked tokens without allowing any existing lockup time to be reduced.

However, lines 256–267 use two different base timestamps for validation versus assignment:

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

When tokens are initially locked in `_lock`, `lastLockTime` is set to `uint32(block.timestamp)` and `unlockTime` is set to `uint32(block.timestamp) + uint32(_lockDuration)`.

As time progresses (`block.timestamp > lastLockTime`), a gap of `elapsed = block.timestamp - lastLockTime` develops between `block.timestamp` and `lastLockTime`:
1. The validation at line 257 only requires `block.timestamp + _duration >= unlockTime`, which is equivalent to `_duration >= unlockTime - block.timestamp` (the remaining lock time).
2. The assignment at lines 265–267 then sets `unlockTime = lastLockTime + uint32(_duration)`.

Because `lastLockTime < block.timestamp`, setting `_duration` to any value in the range `[unlockTime - block.timestamp, unlockTime - lastLockTime)` passes the `LockDurationReducedError` check while reducing `lockedTokens[msg.sender][tokenContract].unlockTime` by `block.timestamp - lastLockTime`.

Specifically, once at least half of the initial lock duration has elapsed (`block.timestamp >= lastLockTime + initialDuration / 2`), the player can set `_duration = unlockTime - block.timestamp <= block.timestamp - lastLockTime`, which sets the new `unlockTime = lastLockTime + _duration <= block.timestamp` and allows the player to immediately call `unlock()` to withdraw all locked tokens early.

In addition, because `BonusManager.getHarvestBonus` and `BonusManager.getPetBonus` compute the player's lock bonus directly from `playerSettings[_player].lockDuration`:
- A player can initially lock with `MaxLockDuration` (e.g., 90 days) to earn the maximum harvest and pet bonus rate during the first half of the lock period, then call `setLockDuration` to shrink `unlockTime` to `<= block.timestamp` and withdraw their principal early.
- Conversely, if a player keeps tokens in the contract after `lastLockTime + MaxLockDuration <= block.timestamp`, calling `setLockDuration(MaxLockDuration)` passes the check while setting `unlockTime = lastLockTime + MaxLockDuration <= block.timestamp`, granting the maximum lock bonus in `playerSettings` with zero remaining lockup commitment.

### Assumptions and preconditions

- The contract is not paused (`_paused == false`).
- A registered player has locked tokens in `LockManager` with an initial lock duration `D`.
- At least some time `t > 0` has elapsed since `lastLockTime` (`block.timestamp == lastLockTime + t`).

### Proof of concept

1. At `t = 1000`, Alice calls `setLockDuration(90 days)` and locks 10 ETH via `lock(address(0), 10 ether)`.
   - `lockedTokens[alice][address(0)].lastLockTime = 1000`
   - `lockedTokens[alice][address(0)].unlockTime = 1000 + 90 days`
   - `playerSettings[alice].lockDuration = 90 days` (Alice earns the maximum 30% lock bonus in `BonusManager`).
2. 45 days pass, so `block.timestamp = 1000 + 45 days`. Alice's tokens are still supposed to be locked for another 45 days (until `1000 + 90 days`).
3. Alice calls `setLockDuration(45 days)`:
   - In `setLockDuration`, the check on line 257 evaluates:
     `uint32(block.timestamp) + uint32(45 days) = (1000 + 45 days) + 45 days = 1000 + 90 days`.
   - Since `1000 + 90 days < 1000 + 90 days` is `false`, `LockDurationReducedError` is not triggered.
   - Lines 265–267 update `unlockTime`:
     `unlockTime = lastLockTime + 45 days = 1000 + 45 days == block.timestamp`.
4. In the same block (`block.timestamp = 1000 + 45 days`), Alice calls `unlock(address(0), 10 ether)`.
   - The check `lockedToken.unlockTime > uint32(block.timestamp)` (`1000 + 45 days > 1000 + 45 days`) is `false`.
   - Alice harvests her rewards (having accrued the 90-day bonus for the preceding 45 days) and withdraws her 10 ETH 45 days early.

### Impact assessment

- **Severity**: High
- **Direct Technical Effect**: Violates the core protocol invariant that players cannot reduce previously set lockup times (`unlockTime`). Any player can cut their effective lock duration in half (or reduce `unlockTime` by the time elapsed since `lastLockTime`).
- **Business and Economic Implications**: Players can game the reward system by committing to the maximum lock duration (`MaxLockDuration`) to farm schnibbles at the maximum lock bonus rate and then shortening their lock duration mid-way to unlock their capital early, undermining TVL retention and yield generation on Blast.

### Remediation

Ensure the validation and the `unlockTime` assignment use the same reference timestamp. To prevent reducing existing lockup times while keeping `unlockTime` anchored to `lastLockTime` (or resetting the lock from `block.timestamp` when extending), validate `lastLockTime + uint32(_duration)` against `unlockTime`, or set `unlockTime = uint32(block.timestamp) + uint32(_duration)` and update `lastLockTime = uint32(block.timestamp)`.

If `lastLockTime` is intended to remain the base timestamp for the lock:

```solidity
    function setLockDuration(uint256 _duration) external notPaused {
        if (_duration > configStorage.getUint(StorageKey.MaxLockDuration))
            revert MaximumLockDurationError();

        playerSettings[msg.sender].lockDuration = uint32(_duration);
        // update any existing lock
        uint256 configuredTokensLength = configuredTokenContracts.length;
        for (uint256 i; i < configuredTokensLength; i++) {
            address tokenContract = configuredTokenContracts[i];
            if (lockedTokens[msg.sender][tokenContract].quantity > 0) {
                uint32 lastLockTime = lockedTokens[msg.sender][tokenContract]
                    .lastLockTime;
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
        }

        emit LockDuration(msg.sender, _duration);
    }
```

Alternatively, if setting a new lock duration should apply from `block.timestamp`:

```solidity
                if (
                    uint32(block.timestamp) + uint32(_duration) <
                    lockedTokens[msg.sender][tokenContract].unlockTime
                ) {
                    revert LockDurationReducedError();
                }

                lockedTokens[msg.sender][tokenContract].lastLockTime = uint32(
                    block.timestamp
                );
                lockedTokens[msg.sender][tokenContract].unlockTime =
                    uint32(block.timestamp) +
                    uint32(_duration);
```

## Finding 3: Griefing Attack in `LockManager.lockOnBehalf` Allows Attackers to Indefinitely Extend Victim's Token Lockup

### Synopsis

In `LockManager._lock`, `lockedToken.lastLockTime` and `lockedToken.unlockTime` are unconditionally overwritten with `block.timestamp` and `block.timestamp + _lockDuration` whenever tokens are locked. Because `lockOnBehalf` is permissionless and accepts zero (or dust) quantities, any third party can repeatedly reset a victim's `unlockTime` and keep their funds locked forever.

### Technical details

The `LockManager.lockOnBehalf` function allows any caller (`msg.sender`) to lock tokens on behalf of an arbitrary registered player (`_onBehalfOf`), who becomes `_lockRecipient` in the internal `_lock` function.

At lines 381–384 of `src/managers/LockManager.sol`, `_lock` updates the recipient's lock timestamps unconditionally:

```solidity
lockedToken.lastLockTime = uint32(block.timestamp);
lockedToken.unlockTime =
    uint32(block.timestamp) +
    uint32(_lockDuration);
```

There are two critical flaws in this design:
1. `lockOnBehalf` has no access control restricting who may lock on behalf of `_onBehalfOf`, nor does it require `_quantity > 0`.
2. Whenever `_lock` executes for `_lockRecipient`, `lockedToken.unlockTime` is pushed forward to `uint32(block.timestamp) + uint32(_lockDuration)`, where `_lockDuration` is at least `lockdrop.minLockDuration` (or the player's configured `playerSettings[_lockRecipient].lockDuration`).

Consequently, a malicious actor can call `lockOnBehalf(_tokenContract, 0, victim)` (paying 0 wei or 0 ERC20 tokens) just before the victim's lock expires. This resets `lockedToken.unlockTime` into the future by `_lockDuration`. When the victim subsequently attempts to withdraw their tokens via `unlock`, the check `if (lockedToken.unlockTime > uint32(block.timestamp)) revert TokenStillLockedError();` at line 410 will revert. By repeating this call once every `_lockDuration` period, an attacker can freeze the victim's locked funds indefinitely.

### Assumptions and preconditions

- The victim is a registered player in `AccountManager` who has locked ETH or ERC20 tokens in `LockManager` with a non-zero `_lockDuration`.
- The contract is not paused and the targeted `_tokenContract` remains active.

### Proof of concept

1. Alice (a registered player) locks `10 ETH` via `LockManager.lock{value: 10 ether}(address(0), 10 ether)` at timestamp `T_0` with a lock duration of `30 days`.
2. Alice's `lockedTokens[alice][address(0)].unlockTime` is set to `T_0 + 30 days`.
3. At timestamp `T_1 = T_0 + 30 days - 1`, right before Alice's lock expires, an attacker Bob calls `LockManager.lockOnBehalf{value: 0}(address(0), 0, alice)`.
4. Inside `_lock`, `msg.value == _quantity` (`0 == 0`) passes validation, and `lockedTokens[alice][address(0)].unlockTime` is updated to `T_1 + 30 days`.
5. At timestamp `T_0 + 30 days`, Alice calls `LockManager.unlock(address(0), 10 ether)`. The transaction reverts with `TokenStillLockedError` because `unlockTime` is now `T_1 + 30 days`.
6. Bob repeats Step 3 before each new `unlockTime`, permanently freezing Alice's `10 ETH`.

### Impact assessment

- **Severity:** High / Critical
- **Direct Technical Effect:** Arbitrary third parties can reset any user's `lastLockTime` and extend their `unlockTime` at no token cost (or 1 wei if a zero-amount check were present).
- **Business Implications:** Directly violates a core protocol invariant ("funds cannot get locked forever"). All user deposits in `LockManager` can be permanently frozen by a griefing attacker for the cost of a single transaction per lock duration period.

### Remediation

Restrict `lockOnBehalf` so that only trusted protocol contracts (such as `MigrationManager`) or explicitly authorized operators for `_onBehalfOf` can lock tokens on behalf of another player, or only update `unlockTime` and `lastLockTime` when the lock is initiated by `_lockRecipient` themselves (while ensuring existing `unlockTime` is preserved or only extended when authorized by the recipient).

```solidity
        lockedToken.remainder = remainder;
        lockedToken.quantity += _quantity;
        if (_tokenOwner == _lockRecipient || lockedToken.unlockTime == 0) {
            lockedToken.lastLockTime = uint32(block.timestamp);
            lockedToken.unlockTime =
                uint32(block.timestamp) +
                uint32(_lockDuration);
        }
```

## Finding 4: Unconditional Remainder Overwrite in `LockManager._lock` Wipes Accumulated NFT Lock Remainder

### Synopsis

In `LockManager._lock`, `lockedToken.remainder` is unconditionally overwritten with the local variable `remainder`, which defaults to `0` and is only computed during an active lockdrop for non-migration calls. Consequently, locking outside a lockdrop or via `MigrationManager` resets a player's accumulated `remainder` to zero.

### Technical details

In `src/managers/LockManager.sol`, when a player locks tokens via `lock` or `lockOnBehalf`, the internal function `_lock` calculates the total eligible amount for NFT minting by adding the previous residual lock amount (`lockedToken.remainder`) to the newly locked `_quantity`:

```solidity
uint256 quantity = _quantity + lockedToken.remainder;
uint256 remainder;
uint256 numberNFTs;
```

However, the new `remainder` is only calculated if two conditions hold:
1. The current `block.timestamp` falls within an active lockdrop window (`lockdrop.start <= uint32(block.timestamp) && lockdrop.end >= uint32(block.timestamp)`), and
2. The caller is not the `migrationManager` (`msg.sender != address(migrationManager)`).

```solidity
if (
    lockdrop.start <= uint32(block.timestamp) &&
    lockdrop.end >= uint32(block.timestamp)
) {
    ...
    if (msg.sender != address(migrationManager)) {
        // calculate number of nfts
        remainder = quantity % configuredToken.nftCost;
        numberNFTs = (quantity - remainder) / configuredToken.nftCost;
        ...
    }
}
```

Despite `remainder` only being assigned inside this conditional block, line 379 unconditionally assigns:

```solidity
lockedToken.remainder = remainder;
```

When `_lock` is executed outside of an active lockdrop period, or when `migrationManager` calls `lockOnBehalf` on behalf of a player during a lockdrop, the local variable `remainder` remains at its default value of `0`. Executing `lockedToken.remainder = remainder` on line 379 overwrites the player's previously stored `lockedToken.remainder` with `0`, permanently destroying their residual credit toward future NFT reveals. Furthermore, because `lockOnBehalf` is permissionless, any third party can intentionally trigger this reset for a target player outside of a lockdrop by locking a dust amount (even `0` tokens) on their behalf.

### Assumptions and preconditions

1. A registered player has locked tokens during an active lockdrop where the locked amount was not an exact multiple of `configuredToken.nftCost`, leaving `lockedToken.remainder > 0`.
2. Either:
   - `migrationManager` calls `lockOnBehalf` for that player and token during the lockdrop, or
   - The player (or any third party via `lockOnBehalf`) locks any amount of the same token when no lockdrop is active, prior to a subsequent lockdrop being configured by the admin.

### Proof of concept

1. Assume ETH (`address(0)`) is configured with `nftCost = 2 ether`, and an active lockdrop is running.
2. Alice (a registered player) calls `lock(address(0), 3 ether)` with `3 ether`.
   - Inside `_lock`, `remainder = 3 ether % 2 ether = 1 ether` and `numberNFTs = 1`.
   - `lockedTokens[alice][address(0)].remainder` is set to `1 ether`.
3. Scenario A (Migration during lockdrop):
   - Alice triggers a migration action that causes `migrationManager` to call `lockOnBehalf(address(0), 1 ether, alice)`.
   - Inside `_lock`, `msg.sender == address(migrationManager)`, so the inner `if` block at line 361 is skipped and `remainder` remains `0`.
   - At line 379, `lockedTokens[alice][address(0)].remainder` is overwritten with `0`, losing `1 ether` of NFT progress.
4. Scenario B (Lock outside lockdrop before a future lockdrop):
   - The first lockdrop ends (`block.timestamp > lockdrop.end`).
   - Alice (or a griefer calling `lockOnBehalf(address(0), 0, alice)`) invokes `_lock`.
   - The lockdrop check at lines 352-355 evaluates to `false`, leaving `remainder = 0`.
   - At line 379, `lockedTokens[alice][address(0)].remainder` is overwritten with `0`. When the admin configures a new lockdrop later, Alice's `1 ether` remainder is gone.

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** `lockedTokens[_lockRecipient][_tokenContract].remainder` is erroneously zeroed out whenever `_lock` executes outside the NFT-calculation branch.
- **Business Impact:** Players lose their accumulated fractional NFT progress (`remainder` up to `nftCost - 1 wei`) when migrating tokens or when locking outside an active lockdrop window (or being griefed via `lockOnBehalf` with `0` value).

### Remediation

Update `lockedToken.remainder` only when `remainder` is actually computed inside the active lockdrop and non-migration conditional block (or initialize `uint256 remainder = lockedToken.remainder;` at declaration).

```solidity
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
                // calculate number of nfts
                remainder = quantity % configuredToken.nftCost;
                numberNFTs = (quantity - remainder) / configuredToken.nftCost;

                if (numberNFTs > type(uint16).max) revert TooManyNFTsError();

                lockedToken.remainder = remainder;

                // Tell nftOverlord that the player has new unopened Munchables
                nftOverlord.addReveal(_lockRecipient, uint16(numberNFTs));
            }
        }

        // Transfer erc tokens
        if (_tokenContract != address(0)) {
            IERC20 token = IERC20(_tokenContract);
            token.transferFrom(_tokenOwner, address(this), _quantity);
        }

        lockedToken.quantity += _quantity;
        lockedToken.lastLockTime = uint32(block.timestamp);
        lockedToken.unlockTime =
            uint32(block.timestamp) +
            uint32(_lockDuration);
```

## Finding 5: Unchecked ERC20 `transferFrom` in `LockManager._lock` Allows Crediting Locks Without Token Transfer

### Synopsis

In `LockManager._lock`, the ERC20 `transferFrom` call does not use `SafeERC20.safeTransferFrom` or check the returned boolean value, nor does it verify the actual balance received. This allows non-reverting failed transfers or fee-on-transfer tokens to desynchronize `lockedToken.quantity` from the contract's actual token balance, enabling theft of locked tokens and free NFT minting.

### Technical details

In ` src/managers/LockManager.sol`, both `lock()` and `lockOnBehalf()` delegate token locking to the internal `_lock()` function. When locking an ERC20 token (`_tokenContract != address(0)`), `_lock()` validates the caller's allowance on lines 328–330:

```solidity
IERC20 token = IERC20(_tokenContract);
uint256 allowance = token.allowance(_tokenOwner, address(this));
if (allowance < _quantity) revert InsufficientAllowanceError();
```

However, when executing the token transfer on lines 374–377 and updating the player's accounting on lines 379–380:

```solidity
if (_tokenContract != address(0)) {
    IERC20 token = IERC20(_tokenContract);
    token.transferFrom(_tokenOwner, address(this), _quantity);
}

lockedToken.remainder = remainder;
lockedToken.quantity += _quantity;
```

This implementation has critical token transfer integration flaws:
1. **Unchecked Return Value & Nonstandard ERC20 Incompatibility:** `token.transferFrom(_tokenOwner, address(this), _quantity)` is called via the raw `IERC20` interface without checking its `bool` return value and without using OpenZeppelin's `SafeERC20.safeTransferFrom`. Because `approve()` does not require the owner to hold any balance, an attacker with zero token balance can approve `LockManager` for `_quantity` (passing the `allowance` check on line 330). For ERC20 tokens that return `false` on failure instead of reverting when the sender has an insufficient balance, `transferFrom` silently fails while `lockedToken.quantity += _quantity` and `nftOverlord.addReveal(...)` still execute. Furthermore, tokens that do not return a boolean (such as USDT-style tokens) will cause `IERC20.transferFrom` to revert during ABI decoding of the expected `bool` return value, breaking token locking entirely for those tokens.
2. **Received-Asset and Credited-Balance Divergence:** If a configured token applies a transfer fee (fee-on-transfer / deflationary tokens), the contract receives less than `_quantity`, yet credits the full `_quantity` to `lockedToken.quantity` and computes `remainder` and `numberNFTs` from the nominal `_quantity`. When the user later calls `unlock()`, `LockManager` transfers the full `_quantity` out, drawing from tokens deposited by other users.

### Assumptions and preconditions

1. The admin has configured an ERC20 token in `LockManager` via `configureToken` that either returns `false` on failed transfers (rather than reverting), has a non-standard return payload, or charges a transfer fee.
2. The attacker (or `_lockRecipient`) is a registered main account in `AccountManager`.
3. The contract is not paused (`_paused == false`).

### Proof of concept

1. Assume an active configured ERC20 token `T` in `LockManager` returns `false` when `transferFrom` fails due to insufficient balance, and `LockManager` currently holds `100e18` of token `T` locked by honest users.
2. A registered user Alice holds `0` balance of token `T`.
3. Alice calls `T.approve(address(lockManager), 100e18)`. This succeeds because `approve` does not check token balances.
4. Alice calls `lockManager.lock(address(T), 100e18)`.
5. Inside `_lock()`:
   - `T.allowance(alice, address(lockManager))` is `100e18 >= 100e18`, passing line 330.
   - During an active lockdrop, `nftOverlord.addReveal(alice, numberNFTs)` grants Alice unrevealed NFTs for the `100e18` lock.
   - On line 376, `T.transferFrom(alice, address(this), 100e18)` fails due to Alice's `0` balance and returns `false`, which is ignored.
   - On line 380, `lockedTokens[alice][address(T)].quantity` is increased by `100e18`.
6. Once `unlockTime` passes, Alice calls `lockManager.unlock(address(T), 100e18)` and withdraws the `100e18` tokens belonging to other users.

### Impact assessment

- **Severity:** High
- **Technical Impact:** Accounting desynchronization between `lockedTokens[_lockRecipient][_tokenContract].quantity` and `IERC20(_tokenContract).balanceOf(address(this))`, or complete inability to lock non-standard tokens that omit a boolean return value.
- **Business Impact:** An attacker can mint unrevealed Munchable NFTs, farm Schnibbles based on phantom locked value in `getLockedWeightedValue()`, and drain ERC20 tokens locked by honest users upon unlock expiration, leaving the protocol insolvent and later users unable to unlock their funds.

### Remediation

Use OpenZeppelin's `SafeERC20.safeTransferFrom` to handle both tokens that return `false` on failure and tokens that do not return a boolean value. Additionally, verify the exact amount of tokens received by checking the contract's balance before and after the transfer.

```solidity
        // Transfer erc tokens
        if (_tokenContract != address(0)) {
            IERC20 token = IERC20(_tokenContract);
            uint256 balanceBefore = token.balanceOf(address(this));
            SafeERC20.safeTransferFrom(token, _tokenOwner, address(this), _quantity);
            if (token.balanceOf(address(this)) - balanceBefore != _quantity) {
                revert InsufficientLockAmountError();
            }
        }
```

## Finding 6: Failure to update or reset `lockedToken.remainder` in `LockManager.unlock` allows reusing residual lock credit to mint NFTs

### Synopsis

In `LockManager.unlock`, withdrawing locked tokens decrements `lockedToken.quantity` without clearing or adjusting `lockedToken.remainder`. Consequently, the residual NFT credit (`remainder`) persists after the underlying principal is withdrawn, allowing players to reuse that credit in subsequent `lock` calls during a lockdrop to mint unrevealed NFTs without locking the full `nftCost`.

### Technical details

When a player locks tokens via `LockManager._lock` (lines 311–398), the contract tracks any leftover token amount that did not reach a full `configuredToken.nftCost` threshold in `lockedToken.remainder`:

```solidity
// add remainder from any previous lock
uint256 quantity = _quantity + lockedToken.remainder;
...
remainder = quantity % configuredToken.nftCost;
numberNFTs = (quantity - remainder) / configuredToken.nftCost;
...
lockedToken.remainder = remainder;
lockedToken.quantity += _quantity;
```

The purpose of `lockedToken.remainder` is to credit a player's fractional progress toward an NFT across multiple deposits while those tokens remain locked in the contract.

However, in `LockManager.unlock` (lines 401–427):

```solidity
LockedToken storage lockedToken = lockedTokens[msg.sender][_tokenContract];
if (lockedToken.quantity < _quantity)
    revert InsufficientLockAmountError();
if (lockedToken.unlockTime > uint32(block.timestamp))
    revert TokenStillLockedError();

accountManager.forceHarvest(msg.sender);

lockedToken.quantity -= _quantity;
```

When a user unlocks some or all of their tokens at line 416, `lockedToken.quantity` is reduced by `_quantity`, and the corresponding tokens are transferred back to `msg.sender`, but `lockedToken.remainder` is never cleared or reduced.

As a result, `lockedToken.remainder` diverges from the actual locked principal (`lockedToken.quantity`). Even if a user withdraws 100% of their locked tokens (`lockedToken.quantity == 0`), `lockedToken.remainder` retains its previous value (up to `configuredToken.nftCost - 1`). If the user subsequently calls `lock` or `lockOnBehalf` while a lockdrop is active (either within the same lockdrop if `minLockDuration` has elapsed before `lockdrop.end`, or in a subsequent lockdrop configured later), `_lock` adds the stale `lockedToken.remainder` to the newly locked `_quantity` at line 344. The user then receives `numberNFTs` calculated from `_quantity + lockedToken.remainder`, double-counting the already-withdrawn tokens toward new NFT reveals.

### Assumptions and preconditions

1. The player is registered with `AccountManager` as a main account.
2. A lockdrop is configured and active (`lockdrop.start <= block.timestamp <= lockdrop.end`).
3. Either `lockdrop.minLockDuration` is shorter than `lockdrop.end - lockdrop.start` (allowing a lock, unlock, and re-lock within the same lockdrop), or a subsequent lockdrop period is configured after the initial lock expires.

### Proof of concept

Assume a token is configured with `nftCost = 1 ether`, and a lockdrop is active from `t = 0` to `t = 14 days` with `minLockDuration = 7 days`.

1. At `t = 0`, Alice calls `lock(address(0), 0.9 ether)` with `0.9 ether`.
   - `quantity = 0.9 ether + 0 = 0.9 ether`.
   - `remainder = 0.9 ether % 1 ether = 0.9 ether`.
   - `numberNFTs = 0`.
   - `lockedToken.quantity` becomes `0.9 ether`, `lockedToken.remainder` becomes `0.9 ether`, and `lockedToken.unlockTime` becomes `7 days`.
2. At `t = 7 days`, `unlockTime` has passed while the lockdrop is still active. Alice calls `unlock(address(0), 0.9 ether)`.
   - Alice receives her `0.9 ether` back.
   - `lockedToken.quantity` is reduced to `0`.
   - `lockedToken.remainder` remains `0.9 ether`.
3. Immediately at `t = 7 days`, Alice calls `lock(address(0), 0.1 ether)` with `0.1 ether`.
   - In `_lock`, `quantity = 0.1 ether + 0.9 ether = 1.0 ether`.
   - `remainder = 1.0 ether % 1.0 ether = 0`.
   - `numberNFTs = 1.0 ether / 1.0 ether = 1`.
   - `nftOverlord.addReveal(alice, 1)` is called, granting Alice 1 unrevealed NFT even though she only has `0.1 ether` locked instead of the required `1.0 ether`.

### Impact assessment

- **Severity**: Medium / High.
- **Technical Impact**: `lockedToken.remainder` becomes desynchronized from `lockedToken.quantity` after unlocking, allowing withdrawn principal to continue counting toward NFT minting thresholds.
- **Business Impact**: Players can mint Munchable NFTs during lockdrops at a fraction of the configured `nftCost` (down to `1 wei` of locked capital if `nftCost - 1` was previously locked and unlocked), inflating the NFT supply and bypassing the capital commitment required by the protocol.

### Remediation

Reset `lockedToken.remainder` (or reduce it proportionally / clear it when tokens are unlocked) in `LockManager.unlock`. The simplest and safest approach is to reset `lockedToken.remainder` to `0` upon unlocking (or adjust it so it never exceeds the remaining uncredited principal):

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

## Finding 7: Mandatory `forceHarvest` Call in `LockManager.unlock` Can Permanently Block Principal Withdrawals

### Synopsis

`LockManager.unlock` unconditionally calls `accountManager.forceHarvest(msg.sender)` before returning locked tokens. Because there is no `try/catch` guard or emergency withdrawal function, any revert in the cross-contract reward harvesting pipeline permanently freezes the player's locked principal (ETH, WETH, or USDB) in `LockManager`.

### Technical details

When a player attempts to withdraw their expired lock via `LockManager.unlock` (`src/managers/LockManager.sol`, lines 401–427), the contract validates `lockedToken.quantity` and `lockedToken.unlockTime`, and then executes:

```solidity
// force harvest to make sure that they get the schnibbles that they are entitled to
accountManager.forceHarvest(msg.sender);
```

This couples the withdrawal of real principal assets (ETH and ERC20 tokens) to the successful execution of the in-game reward settlement pipeline:

1. `LockManager.unlock` calls `AccountManager.forceHarvest(msg.sender)` (line 414).
2. `AccountManager.forceHarvest` validates `onlyConfiguredContract2(StorageKey.LockManager, StorageKey.MunchadexManager)` via `configStorage` and invokes `_harvest(msg.sender)` (`src/managers/AccountManager.sol`, lines 122–132).
3. `AccountManager._harvest` calls `getDailySchnibbles(msg.sender)` (line 373), which makes two external calls:
   - `lockManager.getLockedWeightedValue(msg.sender)` (`src/managers/LockManager.sol`, lines 461–487), which loops through the entire `configuredTokenContracts` array and computes `10 ** (18 - configuredTokens[configuredTokenContracts[i]].decimals)` for every active token in which the player has a positive balance.
   - `bonusManager.getHarvestBonus(msg.sender)` (line 369), which queries downstream game contracts to calculate harvest bonuses.
4. `AccountManager._harvest` then calculates `block.timestamp - players[_caller].lastHarvestDate` and updates the player's `unfedSchnibbles` and `lastHarvestDate` (lines 376–382).

Because `accountManager.forceHarvest(msg.sender)` is not wrapped in a `try/catch` block and `LockManager` provides no emergency withdrawal mechanism that bypasses harvesting, any revert across these external dependencies causes `unlock` to revert:

- If any active token in `configuredTokenContracts` held by the player is configured with `decimals > 18`, `18 - decimals` underflows in `getLockedWeightedValue`, reverting `unlock` even when the player is trying to withdraw a completely different token (such as native ETH).
- If `bonusManager.getHarvestBonus(msg.sender)` reverts due to arithmetic overflow/underflow or broken state in downstream bonus dependencies (`BonusManager`, `SnuggeryManager`, or `MigrationManager`), `forceHarvest` reverts.
- If `AccountManager` or `BonusManager` is temporarily misconfigured or impaired, `forceHarvest` reverts.

Consequently, a failure in settling non-transferable in-game points (`Schnibbles`) prevents players from recovering their locked principal funds, directly violating the core protocol invariant in `README.md` (*"The most important thing is that funds cannot get locked forever"*).

### Assumptions and preconditions

1. A registered player has locked ETH or ERC20 tokens in `LockManager`, and `block.timestamp >= lockedToken.unlockTime`.
2. Any condition that causes `accountManager.forceHarvest(msg.sender)` to revert occurs (for example, a revert in `bonusManager.getHarvestBonus(msg.sender)` or an arithmetic revert in `lockManager.getLockedWeightedValue(msg.sender)` caused by another configured token held by the player).

### Proof of concept

1. Alice registers an account and locks `2 ETH` (`_tokenContract = address(0)`) in `LockManager.lock`.
2. Alice also locks a secondary configured ERC20 token (or holds a balance in one) that has an invalid configuration or triggers a revert in `getLockedWeightedValue`, or Alice's account state triggers a revert inside `bonusManager.getHarvestBonus(alice)`.
3. After `lockedTokens[alice][address(0)].unlockTime` has passed, Alice calls `LockManager.unlock(address(0), 2 ether)` to withdraw her `2 ETH`.
4. `LockManager.unlock` passes the `quantity` and `unlockTime` checks (lines 408–411) and calls `accountManager.forceHarvest(alice)` on line 414.
5. `AccountManager.forceHarvest(alice)` calls `_harvest(alice)` -> `getDailySchnibbles(alice)`, which reverts during `lockManager.getLockedWeightedValue(alice)` or `bonusManager.getHarvestBonus(alice)`.
6. The entire `LockManager.unlock` transaction reverts, leaving Alice unable to withdraw her `2 ETH` principal even though the lock period has expired.

### Impact assessment

- **Severity:** Medium / High
- **Direct Technical Effect:** Any revert in `AccountManager.forceHarvest` or its downstream dependencies (`BonusManager`, `LockManager.getLockedWeightedValue`) blocks `LockManager.unlock` from completing.
- **Business & Protocol Implications:** User principal (ETH, WETH, USDB) becomes frozen in `LockManager` whenever reward harvesting fails for a user, violating the protocol's primary safety invariant that user funds must never be locked forever.

### Remediation

Wrap the `accountManager.forceHarvest(msg.sender)` call in `LockManager.unlock` inside a `try/catch` block so that if reward harvesting fails, the player can still withdraw their unlocked principal tokens.

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
        try accountManager.forceHarvest(msg.sender) {} catch {}

        lockedToken.quantity -= _quantity;

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

## Finding 8: Use of `transfer` in `LockManager.unlock` permanently locks native ETH for smart contract wallets

### Synopsis

`LockManager.unlock` uses `payable(msg.sender).transfer(_quantity)` to return unlocked native ETH to the player. Because `.transfer()` forwards a hardcoded 2,300 gas stipend, smart contract accounts with `receive`/`fallback` logic exceeding 2,300 gas cannot execute `unlock`, permanently trapping their locked ETH inside the `LockManager` contract.

### Technical details

In `LockManager.sol`, players can lock native ETH (`_tokenContract == address(0)`) either directly via `lock` or by having another address call `lockOnBehalf` for them. Once the lock duration (`lockedToken.unlockTime`) expires, the player must call `unlock` to retrieve their principal.

At line 420, when `_tokenContract == address(0)`, `LockManager.unlock` transfers the native ETH back to `msg.sender` using Solidity's `.transfer()`:

```solidity
if (_tokenContract == address(0)) {
    payable(msg.sender).transfer(_quantity);
}
```

The `.transfer()` primitive imposes a hardcoded 2,300 gas limit on the call to `msg.sender`. Furthermore, `unlock` sends funds exclusively to `msg.sender` (which must be the registered main account that owns the lock, as sub-accounts cannot unlock and there is no `unlockTo` or recipient parameter).

If a player is a smart contract (such as a Gnosis Safe multisig, an ERC-4337 smart account, or a proxy wallet), its `receive()` or `fallback()` function typically performs storage reads, event emissions, or delegatecalls that require more than 2,300 gas. While such a contract can seamlessly register and lock ETH (or have ETH locked on its behalf via `lockOnBehalf`), every call to `unlock` will fail with an out-of-gas revert during `payable(msg.sender).transfer(_quantity)`. Consequently, the player's ETH is permanently frozen in `LockManager`.

### Assumptions and preconditions

1. A player uses a smart contract wallet (e.g., multisig, proxy wallet, or account abstraction wallet) whose `receive()` or `fallback()` execution consumes more than 2,300 gas when receiving native ETH.
2. The player registers with `AccountManager` and locks native ETH (`_tokenContract == address(0)`) via `lock` or receives a lock via `lockOnBehalf`.
3. The lock duration (`lockedToken.unlockTime`) elapses.

### Proof of concept

1. A user deploys or uses a smart contract wallet (e.g., a multisig or proxy contract whose `receive()` function writes to storage or delegates to an implementation, costing > 2,300 gas).
2. The smart contract wallet calls `AccountManager.register` to register as a player.
3. The smart contract wallet calls `LockManager.lock{value: 1 ether}(address(0), 1 ether)` (or another user calls `LockManager.lockOnBehalf{value: 1 ether}(address(0), 1 ether, smartWallet)`). The transaction succeeds and records `lockedTokens[smartWallet][address(0)].quantity = 1 ether`.
4. After `lockedToken.unlockTime` passes, the smart contract wallet calls `LockManager.unlock(address(0), 1 ether)`.
5. Execution reaches line 420: `payable(msg.sender).transfer(_quantity)`.
6. Only 2,300 gas is forwarded to `smartWallet`. The wallet's `receive()` handler runs out of gas and reverts, causing the entire `unlock` transaction to revert and permanently locking the `1 ether` in `LockManager`.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Native ETH withdrawal in `LockManager.unlock` always reverts for any smart contract account whose fallback/receive logic requires more than 2,300 gas.
- **Business Impact:** Direct violation of the core protocol invariant that user funds cannot get locked forever. Smart contract wallets and multisigs permanently lose access to their locked ETH principal.

### Remediation

Replace `payable(msg.sender).transfer(_quantity)` with a low-level `.call{value: _quantity}("")` and verify the returned success boolean (note that `unlock` is already protected by the `nonReentrant` modifier and follows the checks-effects-interactions pattern with respect to `lockedToken.quantity`).

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

        // send token
        if (_tokenContract == address(0)) {
            (bool success, ) = payable(msg.sender).call{value: _quantity}("");
            if (!success) revert LockManagerInvalidCallError();
        } else {
            IERC20 token = IERC20(_tokenContract);
            token.transfer(msg.sender, _quantity);
        }

        emit Unlocked(msg.sender, _tokenContract, _quantity);
    }
```
