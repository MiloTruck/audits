## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [L-01](#l-01-reentrancy-risk-in-_transfer-due-to-lsp1-hook-in-_burn) | Reentrancy risk in `_transfer()` due to LSP1 hook in `_burn()` | Low |
| [L-02](#l-02-contract-deployment-can-be-forced-to-revert-by-initial-receivers) | Contract deployment can be forced to revert by initial receivers | Low |
| [L-03](#l-03-transfers-with-small-token-amounts-will-revert) | Transfers with small token amounts will revert | Low |
| [I-01](#i-01-use-lsp2utilsgeneratearrayelementkeyatindex-to-compute-lsp4-array-keys) | Use `LSP2Utils.generateArrayElementKeyAtIndex()` to compute LSP4 array keys | Informational |


## [L-01] Reentrancy risk in `_transfer()` due to LSP1 hook in `_burn()`

### Description

When `_transfer()` is called while the burn fee is set, `_burn()` is called to burn a percentage from the sender's balance:

[UpTurnOs.sol#L295-L299](https://github.com/ledfut/upturnOScontract-2.0/blob/4efed8061a40ec0144fec5c5bc429c53c21823de/src/UpTurnOs.sol#L295-L299)

```solidity
uint16 burnFeePercentage = _burnFeePercentage;
if (burnFeePercentage > 0) {
    burnAmount = _calculatePercentage(amount, burnFeePercentage);
    _burn(from, burnAmount, "");
}
```

Afterwards, the amount transferred is then subtracted from the sender's balance:

[UpTurnOs.sol#L321-L325](https://github.com/ledfut/upturnOScontract-2.0/blob/4efed8061a40ec0144fec5c5bc429c53c21823de/src/UpTurnOs.sol#L321-L325)

```solidity
uint256 transferAmount = amount - (burnAmount + creatorAmount);

// BurnAmount is already deducted from the balance in _burn function
_tokenOwnerBalances[from] -= (amount - burnAmount);
_tokenOwnerBalances[to] += transferAmount;
```

However, such an implementation violates the [Checks-Effects-Interactions](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html) pattern - the LSP1 callback in `_burn()` to the sender is performed before his balance in `_tokenOwnerBalances` is updated:

[LSP7DigitalAssetCore.sol#L533](https://github.com/lukso-network/lsp-smart-contracts/blob/882da18470045fd26973cc46268fa94c0f6f8e52/packages/lsp7-contracts/contracts/LSP7DigitalAssetCore.sol#L533)

```solidity
_notifyTokenSender(from, lsp1Data);
```

This exposes the contract to reentrancy risk - in the LSP1 callback from `_burn()`, a sender could perform other calls before his token balance is subtracted.

### Recommendation

Instead of calling `_burn()`, subtract `burnAmount` from `_existingTokens` and `_tokenOwnerBalances` directly:

```diff
  uint16 burnFeePercentage = _burnFeePercentage;
  if (burnFeePercentage > 0) {
      burnAmount = _calculatePercentage(amount, burnFeePercentage);
-     _burn(from, burnAmount, "");
+     _existingTokens -= burnAmount;
+     _tokenOwnerBalances[from] -= burnAmount;
+
+      emit Transfer({
+          operator: msg.sender,
+          from: from,
+          to: address(0),
+          amount: burnAmount,
+          force: force,
+          data: data
+      });
  }
```

This removes the LSP1 callback that the sender receives in `_burn()` and ensures that he only receives one callback when `_transfer()` is called, which is at the end of the function.

Alternatively, call  `_burn()` after sender and receiver balances in `_tokenOwnerBalances` have been updated:

```diff
  if (amount != 0) {
  
      uint16 burnFeePercentage = _burnFeePercentage;
      if (burnFeePercentage > 0) {
          burnAmount = _calculatePercentage(amount, burnFeePercentage);
-         _burn(from, burnAmount, "");
      }
  
      // ...
  }
  
  uint256 transferAmount = amount - (burnAmount + creatorAmount);
  
  // BurnAmount is already deducted from the balance in _burn function
  _tokenOwnerBalances[from] -= (amount - burnAmount);
  _tokenOwnerBalances[to] += transferAmount;

  emit Transfer({
      // ...
  });

+ if (burnAmount != 0) _burn(from, burnAmount, "");
```

### Mitigation Review

**UpTurnOs:** Fixed in commit [a63ea7a](https://github.com/ledfut/upturnOScontract-2.0/commit/a63ea7a180f577eb4e05b7498beb908b4cd75193).

**MiloTruck:** Verified, `_burn()` is now called after `_tokenOwnerBalances` has been updated as recommended.

## [L-02] Contract deployment can be forced to revert by initial receivers

### Description

In the constructor, `_mint()` is called to mint an initial amount tokens to addresses specified in the `initialReceivers` array:

[UpTurnOs.sol#L95-L105](https://github.com/ledfut/upturnOScontract-2.0/blob/4efed8061a40ec0144fec5c5bc429c53c21823de/src/UpTurnOs.sol#L95-L105)

```solidity
// Mint tokens to specified receivers
if (params.initialAmountToReceive.length > 0) {
    for (uint256 i = 0; i < params.initialReceivers.length; i++) {
        _mint(
            params.initialReceivers[i],
            params.initialAmountToReceive[i],
            true,
            ""
        );
    }
}
```

However, `_mint()` provides an an LSP1 callback to the receiver:

[LSP7DigitalAssetCore.sol#L467](https://github.com/lukso-network/lsp-smart-contracts/blob/882da18470045fd26973cc46268fa94c0f6f8e52/packages/lsp7-contracts/contracts/LSP7DigitalAssetCore.sol#L467)

```solidity
_notifyTokenReceiver(to, force, lsp1Data);
```

As such, if any initial receiver implements a LSP1 hook that simply reverts, the constructor of `UpTurnOs` will also revert.

However, note that initial receivers do not have any incentive to force deployment to revert, so it is unlikely for this to occur.

### Recommendation

Consider documenting that addresses in `initialReceivers` should not implement a `universalReceiver` hook that reverts when receiving tokens.

### Mitigation Review

**UpTurnOs:** Documented in the natspec in commit [a63ea7a](https://github.com/ledfut/upturnOScontract-2.0/commit/a63ea7a180f577eb4e05b7498beb908b4cd75193).

**MiloTruck:** Verified.

## [L-03] Transfers with small token amounts will revert

### Description

If `_burnFeePercentage` or `_creatorFeePercentage` is set to a non-zero value (ie. they are enabled), when `_transfer()` is called, both fees are calculated with `_calculatePercentage()`:

[UpTurnOs.sol#L295-L306](https://github.com/ledfut/upturnOScontract-2.0/blob/4efed8061a40ec0144fec5c5bc429c53c21823de/src/UpTurnOs.sol#L295-L306)

```solidity
uint16 burnFeePercentage = _burnFeePercentage;
if (burnFeePercentage > 0) {
    burnAmount = _calculatePercentage(amount, burnFeePercentage);
    _burn(from, burnAmount, "");
}

uint16 creatorFeePercentage = _creatorFeePercentage;
if (creatorFeePercentage > 0) {
    creatorAmount = _calculatePercentage(
        amount,
        creatorFeePercentage
    );
```

`_calculatePercentage()` reverts if `amount` multiplied by the fee is less than `10_000`:

[UpTurnOs.sol#L396-L402](https://github.com/ledfut/upturnOScontract-2.0/blob/4efed8061a40ec0144fec5c5bc429c53c21823de/src/UpTurnOs.sol#L396-L402)

```solidity
function _calculatePercentage(
    uint256 amount,
    uint256 bps
) public pure returns (uint256) {
    require((amount * bps) >= 10_000, "Amount not enough to be divisible");
    return (amount * bps) / 10_000;
}
```

As such, if `_transfer()` is called to transfer a small amount of tokens, this check could cause the transfer to revert. For example:

- `_burnFeePercentage = 50`, which is 0.5%.
- `_transfer()` is called with `amount = 100`.
- `amount * bps = 100 * 50 = 5000`, which is smaller than `10_000`, hence the check reverts.

This does not have any direct impact on the `UpTurnOs` contract, but could cause issues with external integrations that do not expect `transfer()` to revert when transferring an extremely small amount of tokens.

### Recommendation

Consider removing the `amount * bps >= 10_000` check:

```diff
  function _calculatePercentage(
      uint256 amount,
      uint256 bps
  ) public pure returns (uint256) {
-     require((amount * bps) >= 10_000, "Amount not enough to be divisible");
      return (amount * bps) / 10_000;
  }
```

Since the `UpTurnOs` contract has 18 decimals, any loss of fees due to `(amount * bps) / 10_000` rounding down to `0` would be extremely minimal.

Otherwise, document that small transfers will revert when the burn or creator fee is enabled.

### Mitigation Review

**UpTurnOs:** Fixed in commit [a63ea7a](https://github.com/ledfut/upturnOScontract-2.0/commit/a63ea7a180f577eb4e05b7498beb908b4cd75193).

**MiloTruck:** Verified, the check has been removed as recommended.

## [I-01] Use `LSP2Utils.generateArrayElementKeyAtIndex()` to compute LSP4 array keys

### Description:

In the constructor, the LSP4 key for `LSP4_CREATORS_ARRAY` at index 0 is computed as such:

[UpTurnOs.sol#L115-L120](https://github.com/ledfut/upturnOScontract-2.0/blob/4efed8061a40ec0144fec5c5bc429c53c21823de/src/UpTurnOs.sol#L115-L120)

```solidity
_setData(
    bytes32(
        abi.encodePacked(bytes16(_LSP4_CREATORS_ARRAY_KEY), bytes16(0))
    ),
    abi.encodePacked(owner_)
);
```

Consider calling `generateArrayElementKeyAtIndex()` instead, which is the designated function for computing LSP4 array keys:

[LSP2Utils.sol#L75-L84](https://github.com/lukso-network/lsp-smart-contracts/blob/882da18470045fd26973cc46268fa94c0f6f8e52/packages/lsp2-contracts/contracts/LSP2Utils.sol#L75-L84)

```solidity
function generateArrayElementKeyAtIndex(
    bytes32 arrayKey,
    uint128 index
) internal pure returns (bytes32) {
    bytes memory elementInArray = bytes.concat(
        bytes16(arrayKey),
        bytes16(index)
    );
    return bytes32(elementInArray);
}
```

### Recommendation:

```diff
  _setData(
-     bytes32(
-         abi.encodePacked(bytes16(_LSP4_CREATORS_ARRAY_KEY), bytes16(0))
-     ),
+     LSP2Utils.generateArrayElementKeyAtIndex(_LSP4_CREATORS_ARRAY_KEY, 0),
      abi.encodePacked(owner_)
  );
```

### Mitigation Review

**UpTurnOs:** Fixed in commit [a63ea7a](https://github.com/ledfut/upturnOScontract-2.0/commit/a63ea7a180f577eb4e05b7498beb908b4cd75193).

**MiloTruck:** Verified, the recommended fix was implemented.