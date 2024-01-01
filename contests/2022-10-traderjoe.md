# Trader Joe v2

The code under review can be found in [2022-10-tradejoe](https://github.com/code-423n4/2022-10-traderjoe).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-incorrect-parameters-for-_beforetokentransfer-hook) | Incorrect parameters for `_beforeTokenTransfer()` hook | Medium |

## [M-01] Incorrect parameters for `_beforeTokenTransfer()` hook

In `LBToken.sol, the `_beforeTokenTransfer()` hook has the following parameters:
```solidity
317:    /// @param from The address of the owner of the token
318:    /// @param to The address of the recipient of the  token
319:    /// @param id The id of the token
320:    /// @param amount The amount of token of type `id`
321:    function _beforeTokenTransfer(
322:        address from,
323:        address to,
324:        uint256 id,
325:        uint256 amount
326:    ) internal virtual {}
```

Howver, in `_burn()`, it is called with the following parameters:
```solidity
src/LBToken.sol:
237:        _beforeTokenTransfer(address(0), _account, _id, _amount);
```
This is incorrect as the positions of `address(0)` and `_account` should be swapped.

Although this currently does not have any impact, it could potentially cause bugs to occur should `_beforeTokenTransfer()` be overidden in future contracts.