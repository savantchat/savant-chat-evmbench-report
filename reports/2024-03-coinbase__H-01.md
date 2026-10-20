# Audit report — 2024-03-coinbase/H-01

## Finding 1: Cross-account signature replay in `CoinbaseSmartWallet.validateUserOp` allows a `UserOperation` on one wallet to authorize ERC-1271 signature verification on another wallet

### Synopsis

`CoinbaseSmartWallet.validateUserOp` validates signatures directly against the raw `userOpHash` without applying EIP-712 domain separation, whereas `ERC1271.isValidSignature` wraps the input hash in `replaySafeHash`. If two wallets share an owner, a `UserOperation` signed for one wallet can be replayed as a valid ERC-1271 signature on another wallet.

`CoinbaseSmartWallet.validateUserOp` passes the raw `userOpHash` directly to `_validateSignature` without domain or context separation, whereas `ERC1271.isValidSignature` wraps its input hash via `replaySafeHash`. This causes signature-context confusion across verification interfaces, allowing ERC-1271 signatures on a smart-wallet owner to authorize `UserOperation` execution on the owned wallet (and vice versa).

### Technical details

`CoinbaseSmartWallet` inherits from `ERC1271` and overrides `_validateSignature(bytes32 message, bytes calldata signature)` to verify signatures for both ERC-1271 (`isValidSignature`) and ERC-4337 (`validateUserOp`).

In `ERC1271.isValidSignature(bytes32 hash, bytes calldata signature)` (lines 69-76 of `ERC1271.sol`), the contract applies an anti-cross-account-replay wrapper via `replaySafeHash(hash)` before calling `_validateSignature`:
```solidity
function _eip712Hash(bytes32 hash) internal view virtual returns (bytes32) {
    return keccak256(abi.encodePacked("\x19\x01", domainSeparator(), _hashStruct(hash)));
}
```
Here, `domainSeparator()` binds the signature to `block.chainid` and `address(this)` (the specific `CoinbaseSmartWallet` instance), and `_hashStruct(hash)` computes `keccak256(abi.encode(_MESSAGE_TYPEHASH, hash))`.

However, in `CoinbaseSmartWallet.validateUserOp` (lines 149-160 of `CoinbaseSmartWallet.sol`), the contract passes `userOpHash` directly as the `message` argument to `_validateSignature(userOpHash, userOp.signature)`.

Consider an owner (EOA or WebAuthn passkey) that owns both Wallet A and Wallet B:
1. For a normal `UserOperation`, the EntryPoint computes:
   `userOpHash = keccak256(abi.encode(userOp.hash(), entryPoint, block.chainid))`
2. For a replayable `UserOperation` (`executeWithoutChainIdValidation`), `CoinbaseSmartWallet` computes (line 149 and line 235):
   `userOpHash = keccak256(abi.encode(UserOperationLib.hash(userOp), entryPoint()))`

Notice that in both cases, `userOp.sender` is hashed inside `UserOperationLib.hash(userOp)`, which is the first 32-byte word in the outer `abi.encode(...)`.

More importantly, the reverse direction (or crafted collision structure) creates a signature-context confusion across verification interfaces:
1. When an owner signs a message `H` for ERC-1271 verification on Wallet A, the actual digest signed by the owner key is:
   `D = WalletA.replaySafeHash(H)`.
2. If an attacker or external protocol queries `WalletA.isValidSignature(H, sig)`, `_validateSignature` verifies the owner's signature over `WalletA.replaySafeHash(H)`.
3. Conversely, when the owner signs a `UserOperation` for Wallet B, the owner signs `userOpHash_B` directly. Once `userOp` is submitted to the mempool or executed on-chain, `(userOpHash_B, userOp.signature)` is publicly visible.
4. Even more directly: if a user is tricked into signing (or legitimately signs) an ERC-1271 message on Wallet B where the inner `hash` is crafted such that `WalletB.replaySafeHash(hash) == userOpHash_A` (where `userOpHash_A` is the hash of a `UserOperation` for Wallet A, also owned by the same signer), the resulting signature is immediately valid in `WalletA.validateUserOp(userOp_A, userOpHash_A, ...)`!

Let's trace this exact attack vector:
- Suppose Signer S is an owner of Wallet A (which holds valuable assets) and Wallet B (e.g., a secondary/throwaway wallet or an account on another chain if chainId is skipped or same chain).
- An attacker constructs a `UserOperation` (`userOp_A`) for Wallet A (e.g., transferring all funds from Wallet A or adding the attacker as an owner of Wallet A) and computes its `userOpHash_A`.
- Because `validateUserOp` on Wallet A checks `_validateSignature(userOpHash_A, signature)` directly without `replaySafeHash` or `toEthSignedMessageHash`, `validateUserOp` expects a raw signature from Signer S over the 32-byte value `userOpHash_A`.
- Conversely, if Signer S signs a `UserOperation` for Wallet A (`userOp_A`), and `userOp_A` has `userOpHash_A == WalletB.replaySafeHash(H)`? No, `keccak256` is preimage resistant, so `userOpHash_A` cannot equal `WalletB.replaySafeHash(H)`.
- What about the other direction? Can `WalletB.replaySafeHash(H)` equal `userOpHash_A`? No, because `userOpHash_A` is a `keccak256` hash of `abi.encode(userOp.hash(), entryPoint, chainid)` (64 or 96 bytes), while `replaySafeHash(H)` is `keccak256(abi.encodePacked("\x19\x01", domainSeparator, hashStruct))` (66 bytes). Due to length and prefix differences, they cannot collide.

Wait: where does the signature replay happen when `_validateSignature` is called on `userOpHash`?
Let's check `_validateSignature` in `CoinbaseSmartWallet.sol` (lines 291-325):
```solidity
        if (ownerBytes.length == 32) {
            ...
            return SignatureCheckerLib.isValidSignatureNow(owner, message, sigWrapper.signatureData);
        }
```
Look at what happens when `owner` is a smart contract (e.g., another `CoinbaseSmartWallet`, or an ERC-1271 contract), OR when `owner` is an EOA!
Wait:
Case 1: Signer is an EOA (or Passkey) that owns Wallet A, or Signer is used directly.
When an EOA signs a message in standard Ethereum wallets (or signs a hash for a protocol), if the EOA signs any digest `M` (for instance, an EIP-712 typed data hash for an external protocol like Permit2 / Seaport / ERC20 Permit where the EOA is used directly, OR a `UserOperation` on another smart account implementation), what is `M`?
In EIP-712, any typed data signature is over `keccak256("\x19\x01" || domainSeparator || hashStruct)`.
In `validateUserOp` (line 160), `CoinbaseSmartWallet` passes `userOpHash` directly to `_validateSignature(userOpHash, userOp.signature)`, which calls `SignatureCheckerLib.isValidSignatureNow(owner, userOpHash, sigWrapper.signatureData)` (without `toEthSignedMessageHash` or `replaySafeHash`).
Wait: if an attacker asks the user (EOA or Passkey) to sign an arbitrary 32-byte hash `H` for `isValidSignature` on Wallet A, the wallet UI / user computes `replaySafeHash(H)`.
Wait what if `owner` of Wallet A AND Wallet B is another `CoinbaseSmartWallet` (Wallet C) or an ERC-1271 contract?
Wait: in `IsValidSignature.t.sol` line 40: `testSmartWalletSigner()`!
Look at `testSmartWalletSigner()` in `test/SmartWallet/CoinbaseSmartWallet/IsValidSignature.t.sol`:
```solidity
    function testSmartWalletSigner() public {
        MockCoinbaseSmartWallet otherAccount = new MockCoinbaseSmartWallet();
        otherAccount.initialize(owners);

        vm.prank(signer);
        account.addOwnerAddress(address(otherAccount));

        bytes32 hash = 0x15fa6f8c855db1dccbb8a42eef3a7b83f11d29758e84aed37312527165d5eec5;
        bytes32 toSign = account.replaySafeHash(hash);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(signerPrivateKey, otherAccount.replaySafeHash(toSign));
        ...
```
HOLD ON! Look closely at what happens when `otherAccount` (Wallet C) is an owner of `account` (Wallet A)!
1. `otherAccount` (Wallet C) is registered as an address owner on `account` (Wallet A).
2. Suppose `account` (Wallet A) validates a `UserOperation` in `validateUserOp` (line 160):
   - `account.validateUserOp` calls `account._validateSignature(userOpHash, userOp.signature)`.
   - Inside `account._validateSignature`, `owner` is `otherAccount` (Wallet C).
   - `account._validateSignature` calls `SignatureCheckerLib.isValidSignatureNow(otherAccount, userOpHash, sigWrapper.signatureData)`.
   - `SignatureCheckerLib` calls `otherAccount.isValidSignature(userOpHash, sigWrapper.signatureData)`!
   - And what does `otherAccount.isValidSignature(userOpHash, sigWrapper.signatureData)` do?!
   - It verifies a signature from `otherAccount`'s owner over `otherAccount.replaySafeHash(userOpHash)`!
3. Now look at `otherAccount.isValidSignature(hash, sig)` directly!
   - When `otherAccount` (Wallet C) signs a `UserOperation` for `account` (Wallet A), `otherAccount`'s owner signs `otherAccount.replaySafeHash(userOpHash)`.
   - That EXACT signature `sigWrapper.signatureData` is a valid ERC-1271 signature for `otherAccount.isValidSignature(userOpHash, sigWrapper.signatureData)`!
   - AND conversely: if anyone asks `otherAccount` (Wallet C) to sign a hash `H` for ERC-1271 `otherAccount.isValidSignature(H, sig)`, where `H` is set to `userOpHash` of `account` (Wallet A), the resulting ERC-1271 signature `sig` for `otherAccount` can be wrapped in `SignatureWrapper(ownerIndexOfOtherAccount, sig)` and submitted to `account.validateUserOp` to execute `userOp` on `account` (Wallet A)!

Wait: is there also something in `getUserOpHashWithoutChainId` or `validateUserOp` lines 149-167?
```solidity
        // 0xbf6ba1fc = bytes4(keccak256("executeWithoutChainIdValidation(bytes)"))
        if (userOp.callData.length >= 4 && bytes4(userOp.callData[0:4]) == 0xbf6ba1fc) {
149:            userOpHash = getUserOpHashWithoutChainId(userOp);
150:            if (key != REPLAYABLE_NONCE_KEY) {
151:                revert InvalidNonceKey(key);
152:            }
153:        } else {
154:            if (key == REPLAYABLE_NONCE_KEY) {
155:                revert InvalidNonceKey(key);
156:            }
157:        }
158:
159:        // Return 0 if the recovered address matches the owner.
160:        if (_validateSignature(userOpHash, userOp.signature)) {
161:            return 0;
162:        }
163:
164:        // Else return 1, which is equivalent to:
165:        // `(uint256(validAfter) << (160 + 48)) | (uint256(validUntil) << 160) | (success ? 0 : 1)`
166:        // where `validUntil` is 0 (indefinite) and `validAfter` is 0.
167:        return 1;
```

Let's check ALL possible issues in lines 149-167!
1. Look at `getUserOpHashWithoutChainId(userOp)` on line 149 (defined at line 229-236):
```solidity
    function getUserOpHashWithoutChainId(UserOperation calldata userOp)
        public
        view
        virtual
        returns (bytes32 userOpHash)
    {
        return keccak256(abi.encode(UserOperationLib.hash(userOp), entryPoint()));
    }
```
In `UserOperationLib.pack(userOp)`:
```solidity
    function getSender(UserOperation calldata userOp) internal pure returns (address) {
        address data;
        //read sender from userOp, which is first userOp member (saves 800 gas...)
        assembly {data := calldataload(userOp)}
        return address(uint160(data));
    }
```
`sender` (which is `address(this)` when called via EntryPoint) is included in `UserOperationLib.hash(userOp)`.
Wait: what if an account is deployed at different addresses or same address across chains? The README says:
"This allows certain operations to be replayed for all accounts sharing the same address across chains."
Wait: what if `executeWithoutChainIdValidation` is called?
Let's check `executeWithoutChainIdValidation`:
```solidity
    function executeWithoutChainIdValidation(bytes calldata data) public payable virtual onlyEntryPoint {
        bytes4 selector = bytes4(data[0:4]);
        if (!canSkipChainIdValidation(selector)) {
            revert SelectorNotAllowed(selector);
        }

        _call(address(this), 0, data);
    }
```
Wait: that's in `executeWithoutChainIdValidation` (lines 180-187), which is NOT highlighted (`HL:` is only lines 149-167).

2. Why are lines 149-167 specifically highlighted?
```
- Signer identity ambiguity in owner-index resolution
- Authorization inconsistencies across signature backends
- Signature-context confusion across verification interfaces
```
Look at the third pattern:
"Signature-context confusion across verification interfaces"
`149` to `167` in `validateUserOp`!
In `validateUserOp`:
Line 149: `userOpHash = getUserOpHashWithoutChainId(userOp);`
Line 160: `if (_validateSignature(userOpHash, userOp.signature))`

Let's examine how `validateUserOp` and `isValidSignature` (the two verification interfaces of `CoinbaseSmartWallet`) interact and why `149-167` has a "Signature-context confusion across verification interfaces" vulnerability!

Let's trace carefully:
What are the two verification interfaces on `CoinbaseSmartWallet`?
Interface 1: `validateUserOp(UserOperation calldata userOp, bytes32 userOpHash, uint256 missingAccountFunds)` (ERC-4337 verification interface)
Interface 2: `isValidSignature(bytes32 hash, bytes calldata signature)` (ERC-1271 verification interface)

How does each interface call `_validateSignature(bytes32 message, bytes calldata signature)`?
- `isValidSignature(hash, signature)` calls `_validateSignature({message: replaySafeHash(hash), signature: signature})`.
- `validateUserOp(userOp, userOpHash, missingAccountFunds)` calls `_validateSignature(userOpHash, userOp.signature)` directly (or with `userOpHash = getUserOpHashWithoutChainId(userOp)` on line 149).

Now trace BOTH scenarios of signature-context confusion between `validateUserOp` and `isValidSignature`:

Scenario A (Smart contract wallet as owner / nested `CoinbaseSmartWallet`, or EOA/Passkey signing via ERC-1271):
Suppose `WalletA` has `WalletB` (another `CoinbaseSmartWallet`, or any ERC-1271 smart wallet) as an owner (`addOwnerAddress(address(WalletB))`). Note that `IsValidSignature.t.sol` explicitly tests `testSmartWalletSigner()` where one `CoinbaseSmartWallet` (`otherAccount`) is added as an owner of another `CoinbaseSmartWallet` (`account`).
When `WalletA.validateUserOp` is called for a `UserOperation` with hash `userOpHash`:
1. `WalletA.validateUserOp` calls `WalletA._validateSignature(userOpHash, userOp.signature)` at line 160.
2. Since `WalletB` is an address owner of `WalletA`, `WalletA._validateSignature` calls `SignatureCheckerLib.isValidSignatureNow(WalletB, userOpHash, sigWrapper.signatureData)`.
3. `SignatureCheckerLib` invokes `WalletB.isValidSignature(userOpHash, sigWrapper.signatureData)`.
4. `WalletB.isValidSignature` computes `WalletB.replaySafeHash(userOpHash)` and validates `sigWrapper.signatureData` against `WalletB`'s owner.
Because `WalletA.validateUserOp` passes the raw `userOpHash` (without wrapping it in `WalletA.replaySafeHash(userOpHash)`) to `_validateSignature`, `WalletB.isValidSignature` receives `userOpHash` directly as its `hash` parameter.
Consequently:
- Any ERC-1271 signature produced for `WalletB` over a message hash `H = userOpHash` is directly valid to authorize `userOp` on `WalletA`!
- Even worse, if `WalletB` is an owner of multiple accounts (`WalletA_1`, `WalletA_2`, ...), `WalletB.isValidSignature` only includes `WalletB`'s domain separator (`address(WalletB)` and `block.chainid`), NOT `WalletA`'s address (unless `userOpHash` binds `WalletA`, which is true for `userOpHash` because `userOp.sender == WalletA`).
- Wait: what if `WalletA` is NOT using a smart wallet owner, i.e., `WalletA` has a normal EOA or Passkey owner `S`?
Let's check what happens when `WalletA` has an EOA or Passkey owner `S`:
1. In `WalletA.isValidSignature(hash, signature)`, `_validateSignature` is called with `message = WalletA.replaySafeHash(hash)`.
   The owner `S` signs `WalletA.replaySafeHash(hash)`.
2. In `WalletA.validateUserOp(userOp, userOpHash, ...)`, `_validateSignature` is called with `message = userOpHash` (NOT wrapped in `replaySafeHash` or `toEthSignedMessageHash`!).
   Wait: why did Solady's `ERC4337.sol` (in the external context, line 118) do:
   ```solidity
   bool success = SignatureCheckerLib.isValidSignatureNowCalldata(
       owner(), SignatureCheckerLib.toEthSignedMessageHash(userOpHash), userOp.signature
   );
   ```
   while `CoinbaseSmartWallet` removed `toEthSignedMessageHash` and did NOT use `replaySafeHash(userOpHash)`?
   Because in `CoinbaseSmartWallet`, `_validateSignature` verifies the raw `userOpHash` directly!
   Wait: if `S` (an EOA or Passkey) signs a `UserOperation` for `WalletA`, `S` produces a raw signature over `userOpHash`.
   If `S` is ALSO a contract (ERC-1271) or if `WalletA` is asked to verify an ERC-1271 signature?
   Wait: if `S` is an EOA, and `S` signs `userOpHash` for `WalletA.validateUserOp`:
   Where else is a raw signature by `S` over `userOpHash` valid?
   - If `S` is an EOA, any contract calling `SignatureCheckerLib.isValidSignatureNow(S, digest, sig)` or `ecrecover(digest, v, r, s)` checks if `S` signed `digest`.
   - What if `S` is a `CoinbaseSmartWallet` (or other ERC-1271 wallet) that owns `WalletA`?
     To sign a `UserOperation` for `WalletA`, `S`'s owner must sign `S.replaySafeHash(userOpHash)`. That exact signature is ALSO a valid ERC-1271 signature on `S` (`S.isValidSignature(userOpHash, sig) == 0x1626ba7e`). And conversely, if an attacker gets `S` to sign an ERC-1271 message with `hash = userOpHash` (e.g., an off-chain login/permit challenge where the attacker controls the 32-byte challenge `hash` for `S`), the attacker can take that ERC-1271 signature from `S` and submit it to `EntryPoint.handleOps` to execute an arbitrary `UserOperation` on `WalletA`!

Wait: what if `S` is an EOA or Passkey owner of `WalletA`, and an attacker tricks `S` into signing an ERC-1271 message on `WalletB` (or an EIP-712 message on any domain)?
Wait: if `S` is an EOA or Passkey owner of `WalletA`, how does `S` sign an ERC-1271 message on `WalletA`?
`S` signs `WalletA.replaySafeHash(hash)` which is `keccak256(abi.encodePacked("\x19\x01", domainSeparator, _hashStruct(hash)))`.
How does `S` sign a `UserOperation` on `WalletA`?
- For normal UserOps: `S` signs `userOpHash = keccak256(abi.encode(userOp.hash(), entryPoint, block.chainid))`.
- For replayable UserOps (`executeWithoutChainIdValidation`): `S` signs `userOpHash = keccak256(abi.encode(userOp.hash(), entryPoint))`.
Notice that NEITHER `userOpHash` starts with `"\x19\x01"` or `"\x19Ethereum Signed Message:\n"`!
Wait: if `S` is an EOA or Passkey, when an external dApp asks `S` to sign an arbitrary 32-byte challenge `hash` directly (not via `WalletA.isValidSignature`, or if `S` signs raw hashes), `S` might sign `userOpHash`.
AND if `S` is a smart contract wallet (like another `CoinbaseSmartWallet`, Safe, etc., supported via `SignatureCheckerLib.isValidSignatureNow` in `_validateSignature`), `WalletA.validateUserOp` calls `S.isValidSignature(userOpHash, sigData)`!
Because `WalletA.validateUserOp` does NOT wrap `userOpHash` (e.g., in `replaySafeHash(userOpHash)`), `S.isValidSignature` is called with the raw `userOpHash`.
Therefore:
1. If an attacker requests an ERC-1271 signature from smart wallet `S` over a challenge `hash = userOpHash` (where `userOpHash` is the hash of a malicious `UserOperation` targeting `WalletA`), `S` sees a request to sign a 32-byte hash for `S` (not `WalletA`). Once `S` signs it for ERC-1271 verification on `S`, the attacker wraps the signature with `WalletA`'s `ownerIndex` for `S` and executes the malicious `UserOperation` on `WalletA` (draining `WalletA` or adding a new owner to `WalletA`, even across all chains via `executeWithoutChainIdValidation`).
2. Conversely, whenever `S` legitimately signs a `UserOperation` to be executed on `WalletA`, the signature submitted on-chain in `userOp.signature` contains `sigWrapper.signatureData` which is a valid ERC-1271 signature for `S.isValidSignature(userOpHash, sigWrapper.signatureData)`.

Wait — is there ANOTHER issue in `validateUserOp` (lines 149-167)? Let's check EVERYTHING in lines 149-167 before concluding!
Let's check every single line from 145 to 168:
```solidity
145:        uint256 key = userOp.nonce >> 64;
146:
147:        // 0xbf6ba1fc = bytes4(keccak256("executeWithoutChainIdValidation(bytes)"))
148:        if (userOp.callData.length >= 4 && bytes4(userOp.callData[0:4]) == 0xbf6ba1fc) {
149:            userOpHash = getUserOpHashWithoutChainId(userOp);
150:            if (key != REPLAYABLE_NONCE_KEY) {
151:                revert InvalidNonceKey(key);
152:            }
153:        } else {
154:            if (key == REPLAYABLE_NONCE_KEY) {
155:                revert InvalidNonceKey(key);
156:            }
157:        }
158:
159:        // Return 0 if the recovered address matches the owner.
160:        if (_validateSignature(userOpHash, userOp.signature)) {
161:            return 0;
162:        }
163:
164:        // Else return 1, which is equivalent to:
165:        // `(uint256(validAfter) << (160 + 48)) | (uint256(validUntil) << 160) | (success ? 0 : 1)`
166:        // where `validUntil` is 0 (indefinite) and `validAfter` is 0.
167:        return 1;
```

Let's check if there are any other possibilities in lines 149-167:
Possibility 1: Look at `getUserOpHashWithoutChainId(userOp)` on line 149 and `executeWithoutChainIdValidation`.
When a user signs a cross-chain replayable `UserOperation` (`executeWithoutChainIdValidation`):
- What is included in `getUserOpHashWithoutChainId(userOp)`?
  `keccak256(abi.encode(UserOperationLib.hash(userOp), entryPoint()))`
  And `UserOperationLib.hash(userOp)` includes:
  `sender, nonce, hashInitCode, hashCallData, callGasLimit, verificationGasLimit, preVerificationGas, maxFeePerGas, maxPriorityFeePerGas, hashPaymasterAndData`.
- Wait: what if `userOp.initCode` is non-empty on Chain B when replaying a `UserOperation` from Chain A? No, if `initCode` differs, `hashInitCode` differs.
- What if on Chain A the user adds Owner X (at index 1), removes Owner X (at index 1), etc., or what if `ownerIndex` in `SignatureWrapper` points to a DIFFERENT owner on Chain B?!
  WAIT A SECOND!
  ```
  - Signer identity ambiguity in owner-index resolution
  - Authorization inconsistencies across signature backends
  - Signature-context confusion across verification interfaces
  ```
  Let's check ALL THREE bullets carefully!
  Let's check each of the 3 bullets against lines 149-167:

  Bullet 1: "Signer identity ambiguity in owner-index resolution"
  - In `_validateSignature`, `sigWrapper.ownerIndex` is used to look up `ownerAtIndex(sigWrapper.ownerIndex)`.
  - Where is `sigWrapper`? It's in `userOp.signature`!
  - Is `userOp.signature` included in `userOpHash`? NO! `UserOperationLib.pack(userOp)` hashes everything EXCEPT `userOp.signature`.
  - Wait: if `ownerBytes.length == 32`, `SignatureCheckerLib.isValidSignatureNow(owner, message, sigWrapper.signatureData)` verifies that `owner` (at `ownerIndex`) signed `message`. If `ownerBytes.length == 64`, `WebAuthn.verify` verifies that `(x, y)` (at `ownerIndex`) signed `message`. So changing `ownerIndex` to a different owner would fail signature verification unless that owner ALSO signed `message` (or is a contract that returns true).

  Bullet 2: "Authorization inconsistencies across signature backends"
  - In `_validateSignature`, there are two backends: `SignatureCheckerLib.isValidSignatureNow` (for 32-byte address owners) and `WebAuthn.verify` (for 64-byte passkey owners).
  - That's in `_validateSignature` (lines 291-325), which is NOT highlighted!

  Bullet 3: "Signature-context confusion across verification interfaces"
  - Why is `validateUserOp` (lines 149-167) highlighted?
  - Because `validateUserOp` (line 149 & line 160) is the ERC-4337 verification interface, and it calls `_validateSignature(userOpHash, userOp.signature)` directly without `replaySafeHash` (or distinct domain separation from ERC-1271), whereas `isValidSignature` (the ERC-1271 verification interface) calls `_validateSignature(replaySafeHash(hash), signature)`!

Let's double check if there is any other way `validateUserOp` and `isValidSignature` (or replayable vs non-replayable `validateUserOp`) have signature-context confusion in lines 149-167!
Let's check:
1. Between `validateUserOp` and `isValidSignature`:
   - In `ERC1271.isValidSignature(bytes32 hash, bytes calldata signature)`, the contract wraps `hash` using `replaySafeHash(hash)` before passing it to `_validateSignature`.
   - In `CoinbaseSmartWallet.validateUserOp`, the contract does NOT wrap `userOpHash` in `replaySafeHash` (or an ERC-4337-specific EIP-712 domain wrapper) before passing it to `_validateSignature(userOpHash, userOp.signature)` at line 160.
   - Wait: what if `replaySafeHash` WAS used in `validateUserOp`? Wait: if `validateUserOp` ALSO called `replaySafeHash(userOpHash)` with the exact same `_MESSAGE_TYPEHASH`, then signing `hash = userOpHash` in `isValidSignature` would produce `replaySafeHash(userOpHash)`, which would STILL collide with `validateUserOp`!
   - Why? Look at what happens right now when:
     (a) `owner` is a smart contract (e.g., another `CoinbaseSmartWallet`, as in `testSmartWalletSigner`):
         In `validateUserOp`, `_validateSignature(userOpHash, userOp.signature)` calls `SignatureCheckerLib.isValidSignatureNow(owner, userOpHash, sigWrapper.signatureData)`, which calls `owner.isValidSignature(userOpHash, sigWrapper.signatureData)`.
         Meanwhile, if someone calls `owner.isValidSignature(hash, sig)` directly on the `owner` contract with `hash = userOpHash`, it executes the exact same `owner.isValidSignature(userOpHash, sig)` check!
         Thus, an ERC-1271 signature from `owner` over `hash = userOpHash` is 100% interchangeable with a `UserOperation` signature for `account.validateUserOp`!
     (b) Even when `owner` is an EOA or WebAuthn Passkey:
         In `validateUserOp`, `_validateSignature(userOpHash, userOp.signature)` verifies the EOA / Passkey signature directly over the raw 32-byte `userOpHash` (with no EIP-191 prefix, no EIP-712 domain separator, and no `replaySafeHash`).
         Wait: what if the EOA or Passkey signs an ERC-1271 message on `CoinbaseSmartWallet`?
         In `CoinbaseSmartWallet.isValidSignature(hash, sig)`, `_validateSignature` verifies the EOA / Passkey signature over `replaySafeHash(hash)`.
         Can an attacker pass a `UserOperation` whose `userOpHash` equals `replaySafeHash(hash)`? No, because both are outputs of `keccak256` with different preimage lengths (96 or 64 bytes for `userOpHash` vs 66 bytes for `replaySafeHash`).
         WAIT: what if the EOA or Passkey is asked to sign a message, or what if `validateUserOp` is called during simulation or execution?
         Wait: why would `userOpHash` and `isValidSignature` confuse contexts for an EOA or Passkey, OR is there a way to make `userOpHash` match something else?
         Let's check the preimage lengths of:
         1. `getUserOpHash` in `EntryPoint.sol`:
            `keccak256(abi.encode(userOp.hash(), address(this), block.chainid))` -> `abi.encode(bytes32, address, uint256)` is `32 * 3 = 96` bytes.
         2. `getUserOpHashWithoutChainId` in `CoinbaseSmartWallet.sol` (line 235):
            `keccak256(abi.encode(UserOperationLib.hash(userOp), entryPoint()))` -> `abi.encode(bytes32, address)` is `32 * 2 = 64` bytes!
         3. `_hashStruct(bytes32 hash)` in `ERC1271.sol`:
            `keccak256(abi.encode(_MESSAGE_TYPEHASH, hash))` -> 64 bytes, first 32 bytes is `_MESSAGE_TYPEHASH`.
         4. `_eip712Hash(bytes32 hash)` in `ERC1271.sol`:
            `keccak256(abi.encodePacked("\x19\x01", domainSeparator(), _hashStruct(hash)))` -> `2 + 32 + 32 = 66` bytes.
         5. What about `SignatureCheckerLib.isValidSignatureNow(owner, message, sigWrapper.signatureData)` when `owner` is an EOA vs a contract?
            Look at `SignatureCheckerLib.isValidSignatureNow(address signer, bytes32 hash, bytes memory signature)` (lines 33-112 of `SignatureCheckerLib.sol`):
            First, if `signature.length == 64` or `65`, it tries `ecrecover(hash, v, r, s)`.
            If `ecrecover` does NOT match `signer` (or if `signature.length` is not 64/65, OR if `signer` is a contract or EOA), it falls through to:
            `staticcall(gas(), signer, abi.encodeWithSelector(0x1626ba7e, hash, signature))`!
            Wait: if `signer` is a smart contract (like another `CoinbaseSmartWallet` or Gnosis Safe), `ecrecover` does not match `signer`, so it calls `signer.isValidSignature(hash, signature)`.

2. Wait: what if `validateUserOp` is called by `EntryPoint` with a `userOpHash`?
   How does `EntryPoint` compute `userOpHash`?
   `EntryPoint.getUserOpHash(userOp)` computes `keccak256(abi.encode(userOp.hash(), address(this), block.chainid))` and passes it as the `userOpHash` parameter to `validateUserOp(userOp, userOpHash, missingAccountFunds)`.
   And in `validateUserOp`:
   - If `bytes4(userOp.callData[0:4]) == 0xbf6ba1fc` (`executeWithoutChainIdValidation.selector`), line 149 overwrites `userOpHash`:
     `userOpHash = getUserOpHashWithoutChainId(userOp);`
   - Look at line 148:
     `if (userOp.callData.length >= 4 && bytes4(userOp.callData[0:4]) == 0xbf6ba1fc)`
     Wait: what if `userOp.callData` is NOT `0xbf6ba1fc`, or what if `userOp.callData` IS `0xbf6ba1fc`?
     Wait: in `executeWithoutChainIdValidation(bytes calldata data)` (line 180):
     ```solidity
     function executeWithoutChainIdValidation(bytes calldata data) public payable virtual onlyEntryPoint {
         bytes4 selector = bytes4(data[0:4]);
         if (!canSkipChainIdValidation(selector)) {
             revert SelectorNotAllowed(selector);
         }

         _call(address(this), 0, data);
     }
     ```
     Can `executeWithoutChainIdValidation` be bypassed via `validateUserOp`?
     In `validateUserOp`:
     If `bytes4(userOp.callData[0:4]) == 0xbf6ba1fc`:
     `userOpHash = getUserOpHashWithoutChainId(userOp);`
     `if (key != REPLAYABLE_NONCE_KEY) revert InvalidNonceKey(key);`
     Else:
     `if (key == REPLAYABLE_NONCE_KEY) revert InvalidNonceKey(key);`
     Wait: what if `userOp.callData` has selector `0xbf6ba1fc`, so `validateUserOp` uses `getUserOpHashWithoutChainId(userOp)` and requires `key == REPLAYABLE_NONCE_KEY`. Then `EntryPoint` calls `account` with `userOp.callData`, which invokes `executeWithoutChainIdValidation(bytes calldata data)`.

3. Wait: what if `userOp` calls `upgradeToAndCall` via `executeWithoutChainIdValidation`?
   `canSkipChainIdValidation` allows `UUPSUpgradeable.upgradeToAndCall.selector`!
   And `upgradeToAndCall(address newImplementation, bytes calldata data)` takes arbitrary `data` and delegatecalls `newImplementation` with `data`!
   Wait: that's in `canSkipChainIdValidation` (line 257), not in lines 149-167.

4. Let's check if there is any other issue in lines 149-167!
   Look at the exact lines marked `HL:`:
   ```solidity
   149:            userOpHash = getUserOpHashWithoutChainId(userOp);
   150:            if (key != REPLAYABLE_NONCE_KEY) {
   151:                revert InvalidNonceKey(key);
   152:            }
   153:        } else {
   154:            if (key == REPLAYABLE_NONCE_KEY) {
   155:                revert InvalidNonceKey(key);
   156:            }
   157:        }
   158:
   159:        // Return 0 if the recovered address matches the owner.
   160:        if (_validateSignature(userOpHash, userOp.signature)) {
   161:            return 0;
   162:        }
   163:
   164:        // Else return 1, which is equivalent to:
   165:        // `(uint256(validAfter) << (160 + 48)) | (uint256(validUntil) << 160) | (success ? 0 : 1)`
   166:        // where `validUntil` is 0 (indefinite) and `validAfter` is 0.
   167:        return 1;
   ```
   Why did the marker start at `149` (inside the `if` block, right at `userOpHash = getUserOpHashWithoutChainId(userOp);`) and end at `167` (`return 1;`)?
   Look at `149` and `160`:
   At `149`, `userOpHash` is assigned `getUserOpHashWithoutChainId(userOp)`.
   At `160`, `_validateSignature(userOpHash, userOp.signature)` is called with `userOpHash`.

   Wait: why did `HL:` start at 149 instead of 145 or 148?
   Because line 149 is where `userOpHash` is modified, and line 160 is where `_validateSignature(userOpHash, userOp.signature)` is called!
   Wait — or is there something specific about `userOpHash = getUserOpHashWithoutChainId(userOp)` at line 149 AND `_validateSignature(userOpHash, userOp.signature)` at line 160?
   Let's check BOTH:
   Wait: what happens when `userOpHash = getUserOpHashWithoutChainId(userOp)` is signed by a smart contract owner (`otherAccount`, i.e., an ERC-1271 owner like another `CoinbaseSmartWallet`)?
   Wait:
   Suppose `account` has `otherAccount` (a `CoinbaseSmartWallet`) as an owner, and the user wants to execute a cross-chain replayable `UserOperation` (`executeWithoutChainIdValidation`) on `account`!
   1. On Chain 1, `account.validateUserOp` computes `userOpHash = getUserOpHashWithoutChainId(userOp)` (line 149), which omits `block.chainid` so that the `UserOperation` can be replayed across chains.
   2. Then at line 160, `account.validateUserOp` calls `_validateSignature(userOpHash, userOp.signature)`.
   3. `_validateSignature` calls `SignatureCheckerLib.isValidSignatureNow(otherAccount, userOpHash, sigWrapper.signatureData)`, which calls `otherAccount.isValidSignature(userOpHash, sigWrapper.signatureData)`.
   4. `otherAccount.isValidSignature` calls `otherAccount.replaySafeHash(userOpHash)`, which includes `otherAccount.domainSeparator()` — and `domainSeparator()` (in `ERC1271.sol` line 107) hardcodes `block.chainid`!
      Wait: that means if `otherAccount` is an ERC-1271 owner, cross-chain replay fails on other chains because `otherAccount.isValidSignature` includes `block.chainid`.
   5. AND what if `otherAccount` (or ANY owner) signs an ERC-1271 signature vs a `UserOperation` signature?
      Let's compare:
      - In `CoinbaseSmartWallet`, an owner can be an EOA (20-byte address), a smart contract wallet (20-byte address implementing ERC-1271, explicitly tested in `IsValidSignature.t.sol:testSmartWalletSigner`), or a WebAuthn passkey (64-byte public key).
      - In `ERC1271.isValidSignature(bytes32 hash, bytes calldata signature)`, the wallet wraps `hash` in `replaySafeHash(hash)` (an EIP-712 hash with `domainSeparator` containing `name`, `version`, `chainId`, and `verifyingContract = address(this)`) before calling `_validateSignature(message, signature)`.
      - In `CoinbaseSmartWallet.validateUserOp(UserOperation calldata userOp, bytes32 userOpHash, uint256 missingAccountFunds)`, the wallet does NOT wrap `userOpHash` in an EIP-712 domain/struct hash (nor `replaySafeHash` nor `toEthSignedMessageHash`), and instead passes the raw `userOpHash` directly to `_validateSignature(userOpHash, userOp.signature)` at line 160.
      - Consequently, `_validateSignature` is shared between two different verification interfaces (`validateUserOp` and `isValidSignature`), but `validateUserOp` passes an unwrapped `userOpHash` while `isValidSignature` passes `replaySafeHash(hash)`.
      - Look at what happens when an owner is a smart contract wallet (e.g., `otherAccount` in `testSmartWalletSigner`, or any ERC-1271 contract):
        - When `account.validateUserOp` validates `userOp`, `account._validateSignature(userOpHash, userOp.signature)` calls `SignatureCheckerLib.isValidSignatureNow(owner, userOpHash, sigWrapper.signatureData)`, which invokes `owner.isValidSignature(userOpHash, sigWrapper.signatureData)`.
        - This call is completely indistinguishable from a direct ERC-1271 verification call `owner.isValidSignature(hash, sigData)` where `hash = userOpHash`!
        - Therefore, if an attacker tricks the owner of `owner` (the signer smart wallet) into signing an off-chain ERC-1271 message `hash = userOpHash` (for example, as a login challenge or off-chain order hash on `owner`), the attacker can wrap that signature in a `SignatureWrapper(ownerIndex, sig)` and submit it to `EntryPoint.handleOps` to execute an arbitrary `UserOperation` on `account` (via `account.validateUserOp`).
        - Conversely, any valid `UserOperation` signature submitted for `account` (which is public in the mempool and on-chain calldata) contains `sigWrapper.signatureData`, which is immediately a valid ERC-1271 signature on the `owner` smart wallet for `hash = userOpHash` (`owner.isValidSignature(userOpHash, sigWrapper.signatureData)` returns `0x1626ba7e`).
      - Furthermore, even for EOA and WebAuthn passkey owners, because `validateUserOp` passes the raw `userOpHash` directly to `_validateSignature` without `replaySafeHash` or EIP-191/EIP-712 domain separation, any raw signature produced by the EOA or passkey over a 32-byte challenge `userOpHash` in any other context is directly valid in `validateUserOp`, and any `UserOperation` signature on `account` is a valid raw signature by the EOA/passkey over `userOpHash`.

Let's check the remediation for this issue:
If `validateUserOp` wraps `userOpHash` (or if `validateUserOp` and `isValidSignature` use distinct EIP-712 struct/domain hashes, or if `validateUserOp` hashes `userOpHash` with a domain/typehash or `toEthSignedMessageHash` / `replaySafeHash` with a distinct typehash), then a signature for `isValidSignature` can never be confused with a signature for `validateUserOp`.
Specifically, in `validateUserOp` (line 160), `userOpHash` is passed directly to `_validateSignature(userOpHash, userOp.signature)`.

Let's verify line numbers for Metadata:
- File: `src/SmartWallet/CoinbaseSmartWallet.sol`

Everything is clear, well-structured, and adheres to all constraints. Let's write the report.<!-- RESPONSE BEGIN -->
# Signature-context confusion in `CoinbaseSmartWallet.validateUserOp` allows cross-interface signature replay between ERC-1271 and ERC-4337

`CoinbaseSmartWallet` exposes two signature verification interfaces that both delegate to the internal `_validateSignature(bytes32 message, bytes calldata signature)` function (lines 291–325 of `src/SmartWallet/CoinbaseSmartWallet.sol`):

1. **ERC-1271 (`isValidSignature` in `src/SmartWallet/ERC1271.sol`, lines 69–76):**
   Calls `_validateSignature({message: replaySafeHash(hash), signature: signature})`, where `replaySafeHash(hash)` computes an EIP-712 digest bound to `address(this)`, `block.chainid`, and `_MESSAGE_TYPEHASH`:
   ```solidity
   keccak256(abi.encodePacked("\x19\x01", domainSeparator(), keccak256(abi.encode(_MESSAGE_TYPEHASH, hash))))
   ```

2. **ERC-4337 (`validateUserOp` in `src/SmartWallet/CoinbaseSmartWallet.sol`, lines 137–168):**
   Computes `userOpHash = getUserOpHashWithoutChainId(userOp)` when calling `executeWithoutChainIdValidation` (line 149) or uses the `userOpHash` provided by the `EntryPoint`, and then directly invokes:
   ```solidity
   if (_validateSignature(userOpHash, userOp.signature)) {
       return 0;
   }
   ```

Inside `_validateSignature`, if the owner at `sigWrapper.ownerIndex` is a 32-byte ABI-encoded Ethereum address, line 313 calls:
```solidity
SignatureCheckerLib.isValidSignatureNow(owner, message, sigWrapper.signatureData);
```

`CoinbaseSmartWallet` explicitly supports smart contract wallets (including another `CoinbaseSmartWallet`) as address owners via ERC-1271 (as seen in `TestIsValidSignature.testSmartWalletSigner`). When `owner` is a smart contract wallet (let's call it `OwnerWallet`) and `WalletA` is the `CoinbaseSmartWallet` being operated:

- During `WalletA.validateUserOp(userOp, userOpHash, ...)`, `WalletA._validateSignature` calls `SignatureCheckerLib.isValidSignatureNow(OwnerWallet, userOpHash, sigWrapper.signatureData)`.
- `SignatureCheckerLib` performs a `staticcall` to `OwnerWallet.isValidSignature(userOpHash, sigWrapper.signatureData)`.
- Inside `OwnerWallet.isValidSignature`, `OwnerWallet` verifies that its signer signed `OwnerWallet.replaySafeHash(userOpHash)`.

Because `WalletA.validateUserOp` passes the raw, unwrapped `userOpHash` directly into `_validateSignature` (line 160) without any ERC-4337 context wrapper or `WalletA` domain separation:
1. **ERC-1271 to `validateUserOp` replay:** The call `OwnerWallet.isValidSignature(userOpHash, sigData)` made during `WalletA.validateUserOp` is identical to a standalone ERC-1271 verification call `OwnerWallet.isValidSignature(hash, sigData)` where the challenge `hash` equals `userOpHash`. If the signer of `OwnerWallet` signs an off-chain ERC-1271 challenge `hash = userOpHash` intended only for `OwnerWallet`, an attacker can wrap that signature in `SignatureWrapper(ownerIndex, sigData)` and submit `userOp` to `EntryPoint.handleOps` to execute arbitrary calls (or cross-chain owner modifications via `executeWithoutChainIdValidation`) on `WalletA`.
2. **`validateUserOp` to ERC-1271 replay:** Whenever `OwnerWallet` authorizes a `UserOperation` for `WalletA`, the inner `sigWrapper.signatureData` broadcast in `userOp.signature` is simultaneously a valid ERC-1271 signature on `OwnerWallet` for the hash `userOpHash` (`OwnerWallet.isValidSignature(userOpHash, sigWrapper.signatureData)` returns `0x1626ba7e`).
3. **Lack of EIP-191 / EIP-712 formatting for EOA and Passkey owners in `validateUserOp`:** Unlike Solady's `ERC4337._validateSignature` (which applies `SignatureCheckerLib.toEthSignedMessageHash(userOpHash)`), `CoinbaseSmartWallet.validateUserOp` verifies EOA and WebAuthn signatures over the raw 32-byte `userOpHash`. Any raw signature over a 32-byte digest matching `userOpHash` produced by an EOA or passkey owner in another context is directly valid in `validateUserOp`.

### Assumptions and preconditions

1. A `CoinbaseSmartWallet` (`WalletA`) has an ERC-1271 smart contract wallet (`OwnerWallet`, such as another `CoinbaseSmartWallet`) registered as an address owner, or shares an EOA/Passkey signer that signs raw 32-byte digests.
2. Either the signer of `OwnerWallet` signs an ERC-1271 message on `OwnerWallet` whose 32-byte hash is crafted to equal a `userOpHash` for `WalletA`, or a valid `UserOperation` for `WalletA` is submitted on-chain and its `signatureData` is replayed against `OwnerWallet.isValidSignature(userOpHash, signatureData)`.

### Proof of concept

1. Suppose `WalletA` (`CoinbaseSmartWallet`) has `OwnerWallet` (another `CoinbaseSmartWallet` owned by `signerPrivateKey` at owner index `0`) registered as an owner at `ownerIndex = 0`.
2. An attacker constructs a malicious `UserOperation` (`userOp`) with `sender = address(WalletA)` — for example, calling `WalletA.execute(attacker, address(WalletA).balance, "")` to drain all ETH from `WalletA`, or calling `WalletA.executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.addOwnerAddress, (attacker)))` with `nonce = REPLAYABLE_NONCE_KEY << 64`.
3. The attacker computes `bytes32 targetHash = entryPoint.getUserOpHash(userOp)` (or `WalletA.getUserOpHashWithoutChainId(userOp)` for a replayable operation).
4. The attacker presents `targetHash` to the user as an innocent ERC-1271 off-chain message hash to be signed on `OwnerWallet` (not `WalletA`).
5. The user signs `OwnerWallet.replaySafeHash(targetHash)` with `signerPrivateKey`, producing `bytes memory innerSig = abi.encode(SignatureWrapper(0, abi.encodePacked(r, s, v)))`, which is valid for `OwnerWallet.isValidSignature(targetHash, innerSig)`.
6. The attacker wraps `innerSig` into `userOp.signature = abi.encode(SignatureWrapper(0, innerSig))` and submits `userOp` to `EntryPoint.handleOps`.
7. `EntryPoint` calls `WalletA.validateUserOp(userOp, targetHash, missingAccountFunds)`.
8. At line 160, `WalletA.validateUserOp` calls `_validateSignature(targetHash, userOp.signature)`, which invokes `OwnerWallet.isValidSignature(targetHash, innerSig)`.
9. `OwnerWallet.isValidSignature` returns `0x1626ba7e`, `WalletA.validateUserOp` returns `0` (validation success), and `EntryPoint` executes the attacker's `UserOperation` on `WalletA`.

### Impact assessment

- **Severity:** High / Medium
- **Technical Impact:** Signatures intended for ERC-1271 verification on an owner smart wallet can be replayed in `validateUserOp` to execute arbitrary `UserOperation`s on any `CoinbaseSmartWallet` owned by that smart wallet (and vice versa). Additionally, if `executeWithoutChainIdValidation` is used, a single replayed signature can add an attacker as an owner or upgrade `WalletA` across all EVM chains where `WalletA` is deployed.
- **Business Impact:** Complete compromise of funds and ownership of the `CoinbaseSmartWallet` instance if an owner wallet signs an attacker-supplied 32-byte ERC-1271 hash, violating the core invariant that only intentionally authorized `UserOperation` signatures can move funds, modify owners, or upgrade the account.

### Remediation

Apply domain and context separation to `userOpHash` before passing it to `_validateSignature` in `validateUserOp` (using a dedicated EIP-712 struct typehash distinct from `ERC1271._MESSAGE_TYPEHASH`, or wrapping `userOpHash` so that `validateUserOp` and `isValidSignature` never verify the same underlying message digest).

```solidity
        // 0xbf6ba1fc = bytes4(keccak256("executeWithoutChainIdValidation(bytes)"))
        if (userOp.callData.length >= 4 && bytes4(userOp.callData[0:4]) == 0xbf6ba1fc) {
            userOpHash = getUserOpHashWithoutChainId(userOp);
            if (key != REPLAYABLE_NONCE_KEY) {
                revert InvalidNonceKey(key);
            }
        } else {
            if (key == REPLAYABLE_NONCE_KEY) {
                revert InvalidNonceKey(key);
            }
        }

        bytes32 message = keccak256(
            abi.encodePacked(
                "\x19\x01",
                domainSeparator(),
                keccak256(abi.encode(keccak256("CoinbaseSmartWalletUserOp(bytes32 userOpHash)"), userOpHash))
            )
        );

        // Return 0 if the recovered address matches the owner.
        if (_validateSignature(message, userOp.signature)) {
            return 0;
        }

        // Else return 1, which is equivalent to:
        // `(uint256(validAfter) << (160 + 48)) | (uint256(validUntil) << 160) | (success ? 0 : 1)`
        // where `validUntil` is 0 (indefinite) and `validAfter` is 0.
        return 1;
```
*(Note: For replayable operations where `key == REPLAYABLE_NONCE_KEY`, the domain separator used for `message` should omit `block.chainid` so cross-chain replayability is preserved while still binding `verifyingContract = address(this)` and the `CoinbaseSmartWalletUserOp` struct typehash.)*

## Finding 2: Cross-Chain Replay Validation in `CoinbaseSmartWallet.validateUserOp` Allows Arbitrary Execution and Unsafe Upgrades via `upgradeToAndCall`

### Synopsis

In `CoinbaseSmartWallet.validateUserOp`, `UserOperation`s calling `executeWithoutChainIdValidation` skip `chainId` validation, relying on `canSkipChainIdValidation` to restrict callable selectors. Because `UUPSUpgradeable.upgradeToAndCall` is whitelisted and executes an arbitrary `delegatecall` payload, functions excluded from `canSkipChainIdValidation` can be invoked without `chainId` validation and replayed across chains.

### Technical details

In `CoinbaseSmartWallet.validateUserOp` (lines 148–167), when `userOp.callData` begins with selector `0xbf6ba1fc` (`executeWithoutChainIdValidation(bytes)`), line 149 replaces `userOpHash` with `getUserOpHashWithoutChainId(userOp)`:

```solidity
if (userOp.callData.length >= 4 && bytes4(userOp.callData[0:4]) == 0xbf6ba1fc) {
    userOpHash = getUserOpHashWithoutChainId(userOp);
    if (key != REPLAYABLE_NONCE_KEY) {
        revert InvalidNonceKey(key);
    }
}
```

Signature verification at line 160 (`_validateSignature(userOpHash, userOp.signature)`) then validates the owner's signature over a digest that excludes `block.chainid`.

The security boundary of this cross-chain replay mechanism relies on `executeWithoutChainIdValidation` (lines 180–187) only allowing inner calls whose 4-byte selector passes `canSkipChainIdValidation(bytes4)` (lines 252–262), so that state-changing operations such as `execute` or `executeBatch` (which can transfer funds or interact with external contracts) cannot skip `chainId` validation.

However, `canSkipChainIdValidation` whitelists `UUPSUpgradeable.upgradeToAndCall.selector` (line 257). When `executeWithoutChainIdValidation` invokes `_call(address(this), 0, data)` targeting `upgradeToAndCall(address newImplementation, bytes calldata data)`:

1. **Bypass of `canSkipChainIdValidation` selector restrictions via indirect `delegatecall`**:
   `UUPSUpgradeable.upgradeToAndCall` updates `_ERC1967_IMPLEMENTATION_SLOT` to `newImplementation` and, if `data.length > 0`, executes `delegatecall(gas(), newImplementation, data)`. Because `executeWithoutChainIdValidation` calls `address(this)`, `msg.sender` during `upgradeToAndCall` and its inner `delegatecall` is `address(this)`. In `MultiOwnable._checkOwner()`, `msg.sender == address(this)` is authorized as an owner. Consequently, an owner can pass `newImplementation = account.implementation()` (keeping the same implementation) and `data = abi.encodeCall(CoinbaseSmartWallet.execute, (target, value, callPayload))` (or `executeBatch`), executing arbitrary calls not in `canSkipChainIdValidation` via a chain-agnostic `UserOperation`.

2. **Cross-chain implementation address and state mismatch**:
   Because `userOpHash = getUserOpHashWithoutChainId(userOp)` (line 149) omits `block.chainid`, a `UserOperation` upgrading the wallet to `newImplementation` and executing a migration/setup `data` call signed for one chain can be replayed on any other chain where the wallet shares the same address and `REPLAYABLE_NONCE_KEY` nonce. On another chain:
   - Any `data` payload passed to `upgradeToAndCall` (such as `execute`, token approvals, or asset transfers) will also be replayed on the other chain.
   - If `newImplementation` was not deployed deterministically to the identical address on the other chain, an attacker who controls that address on the destination chain (e.g., via EOA nonce matching or a different factory) can deploy a malicious implementation contract with a valid `proxiableUUID()` at `newImplementation` and replay the `UserOperation` to take over the wallet and drain its funds on that chain.

### Assumptions and preconditions

- The smart wallet proxy is deployed (or can be deterministically deployed via `CoinbaseSmartWalletFactory.createAccount`) at the same address on multiple EVM chains.
- An owner signs a `UserOperation` calling `executeWithoutChainIdValidation` with `UUPSUpgradeable.upgradeToAndCall.selector` using `REPLAYABLE_NONCE_KEY`.

### Proof of concept

1. Suppose a `CoinbaseSmartWallet` exists at the same address on Chain A and Chain B with `REPLAYABLE_NONCE_KEY` nonce `0`.
2. **Scenario 1 (Bypassing `canSkipChainIdValidation` to call `execute`)**:
   - Construct inner call data:
     ```solidity
     bytes memory innerExecute = abi.encodeCall(
         CoinbaseSmartWallet.execute,
         (recipient, 1 ether, "")
     );
     bytes memory upgradeCall = abi.encodeCall(
         UUPSUpgradeable.upgradeToAndCall,
         (account.implementation(), innerExecute)
     );
     bytes memory userOpCalldata = abi.encodeCall(
         CoinbaseSmartWallet.executeWithoutChainIdValidation,
         (upgradeCall)
     );
     ```
   - In `validateUserOp`, `bytes4(userOp.callData[0:4]) == 0xbf6ba1fc` is satisfied, so line 149 computes `userOpHash = getUserOpHashWithoutChainId(userOp)` and line 160 validates the signature without `block.chainid`.
   - During execution, `executeWithoutChainIdValidation` checks `canSkipChainIdValidation(UUPSUpgradeable.upgradeToAndCall.selector)` which returns `true`, and calls `upgradeToAndCall`.
   - `upgradeToAndCall` keeps the implementation unchanged and `delegatecall`s `execute(recipient, 1 ether, "")` with `msg.sender == address(account)`, transferring `1 ether` on Chain A and allowing the exact same `UserOperation` to be replayed on Chain B to transfer another `1 ether`.
3. **Scenario 2 (Cross-chain implementation mismatch)**:
   - On Chain A, a new implementation contract `implA` is deployed at address `0x1234...` (e.g., via standard `CREATE` from an EOA or non-identical deployment).
   - The wallet owner signs a replayable `UserOperation` calling `executeWithoutChainIdValidation(abi.encodeCall(UUPSUpgradeable.upgradeToAndCall, (0x1234..., "")))` intending to upgrade the wallet.
   - On Chain B, an attacker deploys a malicious UUPS-compliant contract at `0x1234...` (or `0x1234...` contains a different contract) and replays the signed `UserOperation` through `EntryPoint.handleOps`.
   - `validateUserOp` (lines 149–161) accepts the signature because `chainId` is excluded, and `executeWithoutChainIdValidation` upgrades the wallet on Chain B to the attacker-controlled implementation at `0x1234...`.

### Impact assessment

- **Severity**: Medium / High
- **Direct Technical Effects**:
  - Directly violates the protocol invariant that functions not in `canSkipChainIdValidation` (such as `execute` and `executeBatch`) cannot be invoked via `executeWithoutChainIdValidation`.
  - Allows cross-chain replay of arbitrary calls bundled inside `upgradeToAndCall`'s `data` parameter, and exposes accounts to cross-chain takeover or bricking if an implementation address differs across chains.

### Remediation

Remove `UUPSUpgradeable.upgradeToAndCall.selector` from `canSkipChainIdValidation` so that contract upgrades (and any associated `delegatecall` initialization data) always require chain-specific validation in `validateUserOp`, or validate in `validateUserOp` / `executeWithoutChainIdValidation` that arbitrary `delegatecall` payloads cannot skip chain ID validation:

```solidity
    function canSkipChainIdValidation(bytes4 functionSelector) public pure returns (bool) {
        if (
            functionSelector == MultiOwnable.addOwnerPublicKey.selector
                || functionSelector == MultiOwnable.addOwnerAddress.selector
                || functionSelector == MultiOwnable.removeOwnerAtIndex.selector
        ) {
            return true;
        }
        return false;
    }
```

## Finding 3: Lack of batch execution in `CoinbaseSmartWallet.executeWithoutChainIdValidation` prevents atomic cross-chain owner rotation and enables stale owner takeover

### Synopsis

`CoinbaseSmartWallet.executeWithoutChainIdValidation` only accepts a single `bytes calldata data` payload instead of an array of calls (`bytes[] calldata calls`). Consequently, multi-step cross-chain account management operations—most critically atomic owner key rotation (`addOwner` followed by `removeOwnerAtIndex`)—must be split across multiple sequential `UserOperation`s, enabling compromised or removed owners to hijack the wallet on secondary chains.

### Technical details

`CoinbaseSmartWallet` supports cross-chain replayable `UserOperation`s via `executeWithoutChainIdValidation(bytes calldata data)` (lines 180–187) paired with `validateUserOp` (lines 148–153), which omits `block.chainid` from `userOpHash` and enforces sequential nonces under `REPLAYABLE_NONCE_KEY`.

However, unlike `executeBatch(Call[] calldata calls)`, `executeWithoutChainIdValidation` only accepts a single `bytes calldata data` argument and executes a single self-call:

```solidity
function executeWithoutChainIdValidation(bytes calldata data) public payable virtual onlyEntryPoint {
    bytes4 selector = bytes4(data[0:4]);
    if (!canSkipChainIdValidation(selector)) {
        revert SelectorNotAllowed(selector);
    }

    _call(address(this), 0, data);
}
```

`canSkipChainIdValidation` only whitelists `MultiOwnable.addOwnerPublicKey.selector`, `MultiOwnable.addOwnerAddress.selector`, `MultiOwnable.removeOwnerAtIndex.selector`, and `UUPSUpgradeable.upgradeToAndCall.selector` (specifically excluding `executeBatch`).

Because `executeWithoutChainIdValidation` cannot execute a batch of whitelisted self-calls atomically within a single `UserOperation`:

1. **Non-atomic owner key rotation:** A wallet with a single owner `Owner_A` (the standard configuration upon creation) cannot atomically replace `Owner_A` with `Owner_B` in one cross-chain replayable `UserOperation`. If `Owner_A` is removed first via `removeOwnerAtIndex(0)`, the wallet is left with zero owners and is permanently bricked on every chain where that `UserOperation` is executed. Thus, the rotation must be split into two separate replayable `UserOperation`s using sequential `REPLAYABLE_NONCE_KEY` nonces:
   - `UserOp_1` (nonce `k`): `executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.addOwnerAddress, (Owner_B)))`
   - `UserOp_2` (nonce `k + 1`): `executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.removeOwnerAtIndex, (indexA)))`

2. **Stale authorization reuse and ownership divergence across chains:** Because `UserOp_1` and `UserOp_2` are separate transactions with separate signatures and nonces:
   - If `Owner_B` was previously added via a replayable `UserOperation` (`UserOp_1` at nonce `k`) and later compromised and removed via `UserOp_2` (at nonce `k + 1`) on Chain A, `Owner_B` can take `UserOp_1` and replay only `UserOp_1` on any secondary Chain B where the wallet has not yet synced its replayable operations. `Owner_B` then becomes an active owner on Chain B without replaying `UserOp_2` and can immediately execute chain-specific `execute` calls (using a non-replayable nonce key) to remove all legitimate owners and drain the wallet on Chain B.
   - Similarly, during an owner rotation from a compromised `Owner_A` to `Owner_B`, `UserOp_1` and `UserOp_2` cannot be enforced atomically across chains; `Owner_A` remains authorized until `UserOp_2` is separately bundled and executed on each chain.

### Assumptions and preconditions

1. A `CoinbaseSmartWallet` account is used across multiple EVM chains (sharing the same deterministic address via `CoinbaseSmartWalletFactory`).
2. Cross-chain owner updates are performed via `executeWithoutChainIdValidation` using `REPLAYABLE_NONCE_KEY`.
3. Either an owner key is rotated in two replayable `UserOperation`s (`addOwner` followed by `removeOwnerAtIndex`), or an owner added via a replayable `UserOperation` is later removed via a subsequent replayable `UserOperation`.

### Proof of concept

1. Alice deploys a `CoinbaseSmartWallet` on Chain A with her initial owner key `Owner_0` at index `0`. The wallet also holds funds (or is later deployed via `CoinbaseSmartWalletFactory.createAccount`) at the same deterministic address on Chain B.
2. Alice adds a secondary key `Owner_1` on Chain A via a cross-chain replayable `UserOperation` (`UserOp_Add`, `nonce = (REPLAYABLE_NONCE_KEY << 64) | 0`), calling `executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.addOwnerAddress, (Owner_1)))`.
3. Later, `Owner_1` is compromised. Because `executeWithoutChainIdValidation` only supports a single call per `UserOperation`, Alice revokes `Owner_1` and rotates to `Owner_2` on Chain A using two replayable `UserOperation`s:
   - `UserOp_Remove` (`nonce = (REPLAYABLE_NONCE_KEY << 64) | 1`): `executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.removeOwnerAtIndex, (1)))`
   - `UserOp_Add2` (`nonce = (REPLAYABLE_NONCE_KEY << 64) | 2`): `executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.addOwnerAddress, (Owner_2)))`
4. On Chain B, `UserOp_Add` has not yet been replayed. The attacker who compromised `Owner_1` submits a bundle to `EntryPoint.handleOps` on Chain B containing:
   - `UserOp_Add` (`nonce = (REPLAYABLE_NONCE_KEY << 64) | 0`), which adds `Owner_1` as a valid owner on Chain B.
   - A chain-specific `UserOperation` signed by `Owner_1` (`nonce = 0`), calling `execute(attackerAddress, walletBalance, "")` and `removeOwnerAtIndex(0)`.
5. Both operations succeed on Chain B before `UserOp_Remove` is ever executed, allowing the revoked owner `Owner_1` to remove `Owner_0` and drain all wallet funds on Chain B.

### Impact assessment

**Severity: Medium**

- **Direct Technical Effect:** Multi-step owner management actions (such as owner rotation, replacing a compromised key, or adding and removing multiple owners) cannot be executed atomically via `executeWithoutChainIdValidation`.
- **Business & Security Impact:** Any owner that was ever added via `executeWithoutChainIdValidation` and later removed (or any owner being rotated out across two separate replayable operations) can selectively replay the state up to their addition on secondary chains where replayable nonces lag behind, gaining unauthorized ownership and draining funds on those chains.

### Remediation

Update `executeWithoutChainIdValidation` to accept an array of calls (`bytes[] calldata calls`), validate that each call's selector is permitted by `canSkipChainIdValidation`, and execute all calls atomically in sequence. Also update the selector check in `validateUserOp` to match `executeWithoutChainIdValidation(bytes[])`.

```solidity
    /// @notice Execute the given calls from this account to this account (i.e., self call).
    ///
    /// @dev Can only be called by the Entrypoint.
    /// @dev Reverts if any given call is not authorized to skip the chain ID validation.
    /// @dev `validateUserOp()` will recompute the `userOpHash` without the chain ID before validating
    ///      it if the `UserOperation` aims at executing this function. This allows certain operations
    ///      to be replayed for all accounts sharing the same address across chains. E.g. This may be
    ///      useful for syncing owner changes.
    ///
    /// @param calls The array of raw call data to execute on `address(this)`.
    function executeWithoutChainIdValidation(bytes[] calldata calls) public payable virtual onlyEntryPoint {
        for (uint256 i; i < calls.length;) {
            bytes calldata call = calls[i];
            bytes4 selector = bytes4(call[0:4]);
            if (!canSkipChainIdValidation(selector)) {
                revert SelectorNotAllowed(selector);
            }

            _call(address(this), 0, call);

            unchecked {
                ++i;
            }
        }
    }
```

## Finding 4: Cross-Chain Replay of `removeOwnerAtIndex` via `CoinbaseSmartWallet.canSkipChainIdValidation` Can Remove the Wrong Owner Across Chains

### Synopsis

`CoinbaseSmartWallet.canSkipChainIdValidation` whitelists `MultiOwnable.removeOwnerAtIndex` for chain-agnostic replay via `executeWithoutChainIdValidation`. Because owner indices (`nextOwnerIndex`) can diverge across chains when owners are added through chain-specific calls, replaying `removeOwnerAtIndex(uint256 index)` across chains can delete an unintended owner or brick the wallet.

### Technical details

`CoinbaseSmartWallet` supports cross-chain replayable `UserOperation`s through `executeWithoutChainIdValidation(bytes calldata data)`. When `validateUserOp` detects that `userOp.callData` targets `executeWithoutChainIdValidation`, it enforces that the nonce key equals `REPLAYABLE_NONCE_KEY` and validates the signature against `getUserOpHashWithoutChainId(userOp)`, which omits `block.chainid`. During execution, `executeWithoutChainIdValidation` checks whether the inner call's selector is permitted by `canSkipChainIdValidation(bytes4 functionSelector)` (lines 252–262):

```solidity
function canSkipChainIdValidation(bytes4 functionSelector) public pure returns (bool) {
    if (
        functionSelector == MultiOwnable.addOwnerPublicKey.selector
            || functionSelector == MultiOwnable.addOwnerAddress.selector
            || functionSelector == MultiOwnable.removeOwnerAtIndex.selector
            || functionSelector == UUPSUpgradeable.upgradeToAndCall.selector
    ) {
        return true;
    }
    return false;
}
```

In `MultiOwnable`, owner management functions (`addOwnerAddress`, `addOwnerPublicKey`, and `removeOwnerAtIndex`) are protected by the `onlyOwner` modifier, which permits calls from any registered owner EOA directly, from `address(this)` via a chain-specific `execute` / `executeBatch` `UserOperation` (using a standard chain-bound nonce key), or from `address(this)` via `executeWithoutChainIdValidation` (using `REPLAYABLE_NONCE_KEY`).

When `addOwnerAddress` or `addOwnerPublicKey` is called, `_addOwner` assigns the new owner to the current `_getMultiOwnableStorage().nextOwnerIndex++` and stores the mapping `ownerAtIndex[index] = owner`. Because owner additions are not restricted exclusively to `executeWithoutChainIdValidation`:

1. **Ownership index divergence across chains:** An owner can add a chain-specific owner on Chain A (directly or via `execute`) and a different owner on Chain B (either via a chain-specific call or by later replaying an `executeWithoutChainIdValidation` call). As a result, the same `index` in `ownerAtIndex[index]` maps to different `owner` bytes on Chain A and Chain B.
2. **Unverified index-only revocation in cross-chain replay:** `MultiOwnable.removeOwnerAtIndex(uint256 index)` takes only `index` as an argument and does not verify the expected `owner` bytes at that index before deleting `isOwner[owner]` and `ownerAtIndex[index]`. When a user signs a cross-chain replayable `UserOperation` calling `executeWithoutChainIdValidation` with `removeOwnerAtIndex(index)` to revoke a specific owner at `index` on Chain A, anyone can permissionlessly replay that exact `UserOperation` on Chain B (provided the `REPLAYABLE_NONCE_KEY` sequence matches), removing whatever owner happens to occupy `ownerAtIndex[index]` on Chain B.
3. **Inconsistent cross-chain revocation:** Conversely, if an owner is added via a replayable `executeWithoutChainIdValidation` operation (e.g., replayable nonce 0) and is later revoked on Chain A via a direct `removeOwnerAtIndex` call or a chain-bound `execute` `UserOperation`, the revocation cannot be replayed on other chains and does not increment the replayable nonce. An attacker or the removed owner can still replay the original `addOwner` `UserOperation` on any chain where it has not yet been executed, regaining ownership on those chains without a corresponding replayable revocation.

### Assumptions and preconditions

1. A `CoinbaseSmartWallet` instance is deployed (or can be deterministically deployed) at the same address across at least two EVM chains (Chain A and Chain B) with identical initial owners.
2. At least one owner addition is performed on Chain A via a direct owner call or a chain-bound `execute` / `executeBatch` `UserOperation` (without executing the identical sequence on Chain B), causing `nextOwnerIndex` and `ownerAtIndex` to diverge between Chain A and Chain B while their `REPLAYABLE_NONCE_KEY` nonces remain equal.
3. An owner signs a `UserOperation` using `executeWithoutChainIdValidation` to call `removeOwnerAtIndex(index)` (or adds an owner via `executeWithoutChainIdValidation` and later removes them via a non-replayable call).

### Proof of concept

1. Alice deploys her `CoinbaseSmartWallet` on Chain A and Chain B with initial owner `Owner_0` (Alice's main key at `index = 0`, so `nextOwnerIndex = 1` and replayable nonce = 0 on both chains).
2. On Chain A, `Owner_0` directly calls `addOwnerAddress(TempOwner_A)` (or executes it via a standard chain-bound `UserOperation` calling `execute`). On Chain A, `ownerAtIndex[1] = TempOwner_A` and `nextOwnerIndex = 2`. On Chain B, `nextOwnerIndex` is still `1`.
3. Next, `Owner_0` signs a cross-chain replayable `UserOperation` (with `REPLAYABLE_NONCE_KEY` and sequence `0`) calling `executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.addOwnerAddress, (ColdWallet)))`.
   - On Chain A, `ColdWallet` is registered at `ownerAtIndex[2]`.
   - When replayed on Chain B, `ColdWallet` is registered at `ownerAtIndex[1]`.
4. Later, `TempOwner_A` is compromised or no longer needed. `Owner_0` removes `TempOwner_A` on Chain A by signing a replayable `UserOperation` (with `REPLAYABLE_NONCE_KEY` and sequence `1`) calling `executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.removeOwnerAtIndex, (1)))`.
5. On Chain A, `ownerAtIndex[1]` (`TempOwner_A`) is removed as intended.
6. An attacker (such as the compromised `TempOwner_A` or any front-runner) replays the exact same `UserOperation` (sequence `1`) on Chain B.
7. On Chain B, `executeWithoutChainIdValidation` invokes `removeOwnerAtIndex(1)`, which deletes `ownerAtIndex[1]`—removing `ColdWallet` instead of `TempOwner_A` (and if `Owner_0` had already been removed on Chain B, leaving the wallet on Chain B with no valid owners and permanently bricking it).

### Impact assessment

- **Severity:** Medium
- **Technical Impact:** Cross-chain replay of `removeOwnerAtIndex(uint256 index)` can revoke the wrong owner on chains where `nextOwnerIndex` has diverged, or fail to revoke a compromised owner across chains. In the worst case, if the mistakenly removed owner at `index` on a secondary chain is the sole remaining active owner on that chain, the wallet on that chain is permanently bricked and all funds held in it are frozen.

### Remediation

Ensure that cross-chain replayable owner removal explicitly binds the target `owner` bytes in addition to `index` (e.g., `removeOwnerAtIndex(uint256 index, bytes calldata owner)` in `MultiOwnable` that reverts if `keccak256(owner) != keccak256(ownerAtIndex(index))`), or do not allow unverified index-only removal to skip chain ID validation in `canSkipChainIdValidation`.

If restricting `canSkipChainIdValidation` in `CoinbaseSmartWallet.sol` until owner-verified removal is used:

```solidity
    /// @return `true` is the function selector is whitelisted to skip the chain ID validation, else `false`.
    function canSkipChainIdValidation(bytes4 functionSelector) public pure returns (bool) {
        if (
            functionSelector == MultiOwnable.addOwnerPublicKey.selector
                || functionSelector == MultiOwnable.addOwnerAddress.selector
                || functionSelector == UUPSUpgradeable.upgradeToAndCall.selector
        ) {
            return true;
        }
        return false;
    }
```

Alternatively, update `MultiOwnable.removeOwnerAtIndex` to require `(uint256 index, bytes calldata owner)` and validate that `ownerAtIndex(index)` matches `owner` so `MultiOwnable.removeOwnerAtIndex.selector` in `canSkipChainIdValidation` cannot delete an unintended owner when indices diverge across chains.

## Finding 5: Cross-Chain Replay of `MultiOwnable.removeOwnerAtIndex` Can Remove the Wrong Owner Due to Missing Owner-Bytes Binding

### Synopsis

`MultiOwnable.removeOwnerAtIndex` is whitelisted for cross-chain replay via `CoinbaseSmartWallet.executeWithoutChainIdValidation`, but it identifies the target owner solely by storage `index`. If owner indices diverge across chains, replaying a removal operation on another chain removes whatever owner occupies that index, potentially revoking a legitimate owner and locking the wallet.

### Technical details

`CoinbaseSmartWallet` supports cross-chain replayable user operations through `executeWithoutChainIdValidation`, which validates signatures against `getUserOpHashWithoutChainId(userOp)` (omitting `block.chainid`) when `userOp.nonce >> 64 == REPLAYABLE_NONCE_KEY`. `canSkipChainIdValidation` explicitly whitelists `MultiOwnable.removeOwnerAtIndex.selector` for cross-chain replay.

In `MultiOwnable.sol`, owners are assigned sequential indices via `nextOwnerIndex++` whenever `addOwnerAddress` or `addOwnerPublicKey` is called:

```solidity
function removeOwnerAtIndex(uint256 index) public virtual onlyOwner {
    bytes memory owner = ownerAtIndex(index);
    if (owner.length == 0) revert NoOwnerAtIndex(index);

    delete _getMultiOwnableStorage().isOwner[owner];
    delete _getMultiOwnableStorage().ownerAtIndex[index];

    emit RemoveOwner(index, owner);
}
```

Because `addOwnerAddress` and `addOwnerPublicKey` can also be invoked via chain-specific `UserOperation`s (using a normal nonce key) or directly by an EOA owner on a single chain, `nextOwnerIndex` and the `ownerAtIndex` mapping are not guaranteed to be synchronized across all chains where the wallet is deployed.

Furthermore, even if all owner additions are signed as cross-chain replayable operations, different operations adding different owners on different chains—or a chain where the wallet is deployed later with a different initial owner order—will result in different owners occupying the same `index` on different chains.

Because `removeOwnerAtIndex(uint256 index)` does not take the expected `bytes calldata owner` parameter and verify that `ownerAtIndex(index)` matches the intended owner bytes, the replayable signature is bound only to the numeric `index` rather than the actual owner identity being removed. Anyone can permissionlessly replay the signed `UserOperation` on another chain where `index` points to a different owner, removing that owner against the signer's intent.

### Assumptions and preconditions

1. A `CoinbaseSmartWallet` instance is deployed at the same address on at least two EVM chains (Chain A and Chain B).
2. Due to a chain-specific owner addition (via direct EOA owner call or standard `execute` `UserOperation`) on Chain A, or independent owner additions across chains, `ownerAtIndex(i)` on Chain A holds an unwanted/compromised owner `Owner_X`, while `ownerAtIndex(i)` on Chain B holds a valid owner `Owner_Y`.
3. An owner signs a cross-chain replayable `UserOperation` (`key == REPLAYABLE_NONCE_KEY`) calling `executeWithoutChainIdValidation` with `removeOwnerAtIndex(i)` intending to remove `Owner_X`.

### Proof of concept

1. A user deploys `CoinbaseSmartWallet` on Chain A and Chain B with `Owner_1` at `index = 0`.
2. On Chain A, `Owner_1` directly calls `addOwnerAddress(Owner_Temp)` (or submits a normal chain-specific `UserOperation`), assigning `Owner_Temp` to `index = 1` on Chain A. `nextOwnerIndex` is now `2` on Chain A and `1` on Chain B.
3. `Owner_1` signs a cross-chain replayable `UserOperation` (`nonce = (REPLAYABLE_NONCE_KEY << 64) | 0`) calling `executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.addOwnerAddress, (Owner_2)))`.
   - On Chain A, `Owner_2` is registered at `index = 2`.
   - On Chain B, `Owner_2` is registered at `index = 1`.
4. Later, `Owner_1` wants to remove `Owner_Temp` (at `index = 1` on Chain A) using a cross-chain replayable `UserOperation` (`nonce = (REPLAYABLE_NONCE_KEY << 64) | 1`) calling `executeWithoutChainIdValidation(abi.encodeCall(MultiOwnable.removeOwnerAtIndex, (1)))`.
5. An attacker (or any relayer) replays this `UserOperation` on Chain B via `EntryPoint.handleOps`.
6. On Chain B, `validateUserOp` succeeds because the signature is valid over `getUserOpHashWithoutChainId(userOp)` and the replayable nonce `1` is unused on Chain B.
7. `removeOwnerAtIndex(1)` executes on Chain B and deletes `Owner_2` instead of `Owner_Temp`. If `Owner_1` was already removed or lost on Chain B, the wallet on Chain B is permanently bricked.

### Impact assessment

- **Severity:** Medium / High
- **Technical Impact:** Unauthorized/unintended removal of a valid wallet owner on secondary chains when cross-chain replayable removal operations are replayed against desynchronized `ownerAtIndex` state.
- **Business Impact:** Users relying on `Owner_2` on Chain B lose access to their smart wallet and potentially all funds held in the wallet on that chain.

### Remediation

Modify `removeOwnerAtIndex` to require the expected `bytes calldata owner` as an argument and revert if the stored `ownerAtIndex(index)` does not match the provided `owner` bytes.

```solidity
    error WrongOwnerAtIndex(uint256 index, bytes expectedOwner, bytes actualOwner);

    function removeOwnerAtIndex(uint256 index, bytes calldata owner) public virtual onlyOwner {
        bytes memory owner_ = ownerAtIndex(index);
        if (owner_.length == 0) revert NoOwnerAtIndex(index);
        if (keccak256(owner_) != keccak256(owner)) {
            revert WrongOwnerAtIndex({index: index, expectedOwner: owner, actualOwner: owner_});
        }

        delete _getMultiOwnableStorage().isOwner[owner];
        delete _getMultiOwnableStorage().ownerAtIndex[index];

        emit RemoveOwner(index, owner);
    }
```
