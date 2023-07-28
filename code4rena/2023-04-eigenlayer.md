# EigenLayer

The code under review can be found in [2023-04-eigenlayer](https://github.com/code-423n4/2023-04-eigenlayer).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [H-01](#h-01-queued-withdrawals-with-malicious-strategies-cannot-be-slashed-due-to-error-in-slashqueuedwithdrawal) | Queued withdrawals with malicious strategies cannot be slashed due to error in `slashQueuedWithdrawal()` | High |
| [M-01](#m-01-attacker-can-grief-withdrawals-by-forcing-_completequeuedwithdrawal-to-revert) | Attacker can grief withdrawals by forcing `_completeQueuedWithdrawal()` to revert | Medium |
| [L-01](#l-01-lack-of-validation-allows-users-to-queue-withdrawals-that-cannot-be-completed) | Lack of validation allows users to queue withdrawals that cannot be completed | Low |

## [H-01] Queued withdrawals with malicious strategies cannot be slashed due to error in `slashQueuedWithdrawal()`

### Bug Description

In `StrategyManager.sol`, the owner can call `slashQueuedWithdrawal()` to slash an existing queued withdrawal that is delegated to a "frozen" operator. 

If the withdrawal contains a "malicious" strategy (ie. reverts when `withdraw()` is called) in its `strategies` array, the owner can include its index in `indicesToSkip` to skip it. However, this is implemented incorrectly in `slashQueuedWithdrawal()`:

[StrategyManager.sol#L559-L578](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L559-L578)

```solidity
uint256 strategiesLength = queuedWithdrawal.strategies.length;
for (uint256 i = 0; i < strategiesLength;) {
    // check if the index i matches one of the indices specified in the `indicesToSkip` array
    if (indicesToSkipIndex < indicesToSkip.length && indicesToSkip[indicesToSkipIndex] == i) {
        unchecked {
            ++indicesToSkipIndex;
        }
    } else {
        // Some code to slash queuedWithdrawal here...

        unchecked {
            ++i;
        }
    }
}
```

In the code above, `i` is only incremented in the `else` body. Therefore, if `indicesToSkip[indicesToSkipIndex]` matches `i`, only `indicesToSkipIndex` is incremented, and `i` remains the same for the next iteration. This makes `indicesToSkip` redundant as the malicious strategy's index is never actually skipped.

### Impact

If a queued withdrawal:
1. Is delegated to a "frozen" operator 
2. Contains a malicious strategy 

its shares can never be withdrawn, causing assets to be permanently stuck in their respecptive strategies. This is because:

1. `slashQueuedWithdrawal()` cannot skip malicious strategies due to the bug described above, and will therefore revert.
2. `completeQueuedWithdrawal()` only works for queued withdrawal delegated to non-"frozen" operators.

### Recommendation

Always increment `i` for every iteration in `slashQueuedWithdrawal()`:

[StrategyManager.sol#L559-L578](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L559-L578)

```diff
        uint256 strategiesLength = queuedWithdrawal.strategies.length;
        for (uint256 i = 0; i < strategiesLength;) {
            // check if the index i matches one of the indices specified in the `indicesToSkip` array
            if (indicesToSkipIndex < indicesToSkip.length && indicesToSkip[indicesToSkipIndex] == i) {
                unchecked {
                    ++indicesToSkipIndex;
                }
            } else {
                if (queuedWithdrawal.strategies[i] == beaconChainETHStrategy){
                     //withdraw the beaconChainETH to the recipient
                    _withdrawBeaconChainETH(queuedWithdrawal.depositor, recipient, queuedWithdrawal.shares[i]);
                } else {
                    // tell the strategy to send the appropriate amount of funds to the recipient
                    queuedWithdrawal.strategies[i].withdraw(recipient, tokens[i], queuedWithdrawal.shares[i]);
                }
-               unchecked {
-                   ++i;
-               }
           }
+          unchecked {
+              ++i;
+          }
        }
```


## [M-01] Attacker can grief withdrawals by forcing `_completeQueuedWithdrawal()` to revert

### Bug Description

In `StrategyManager.sol`, if users want to withdraw their tokens from strategies, they have to queue up withdrawals using `queueWithdrawal()`. 

After `withdrawalDelayBlocks` blocks has passed, they can call `completeQueuedWithdrawal()` to complete their queued withdrawal. This withdraws tokens from strategies in a loop:

[StrategyManager.sol#L779-L794](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L779-L794)

```solidity
// actually withdraw the funds
for (uint256 i = 0; i < strategiesLength;) {
    if (queuedWithdrawal.strategies[i] == beaconChainETHStrategy) {

        // if the strategy is the beaconchaineth strat, then withdraw through the EigenPod flow
        _withdrawBeaconChainETH(queuedWithdrawal.depositor, msg.sender, queuedWithdrawal.shares[i]);
    } else {
        // tell the strategy to send the appropriate amount of funds to the depositor
        queuedWithdrawal.strategies[i].withdraw(
            msg.sender, tokens[i], queuedWithdrawal.shares[i]
        );
    }
    unchecked {
        ++i;
    }
}
```

As seen from above, if `withdraw()` reverts for any strategy in the `strategies` array, `_completeQueuedWithdrawal()` will also revert. 

In `StrategyBase.sol`, `withdraw()` reverts if the total amount of shares is below `MIN_NONZERO_TOTAL_SHARES` after the withdrawal:

[StrategyBase.sol#L138-L140](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/strategies/StrategyBase.sol#L138-L140)

```solidity
// check to avoid edge case where share rate can be massively inflated as a 'griefing' sort of attack
require(updatedTotalShares >= MIN_NONZERO_TOTAL_SHARES || updatedTotalShares == 0,
    "StrategyBase.withdraw: updated totalShares amount would be nonzero but below MIN_NONZERO_TOTAL_SHARES");
```

By manipulating the total number of shares in a strategy, an attacker can force the check shown above to revert when a user calls `completeQueuedWithdrawal()`, making him unable to withdraw his tokens. 

Note that `withdraw()` also reverts if withdrawals are paused by a strategy's pauser. 

### Impact

An attacker can manipulate a strategy's total shares to make a user unable to withdraw their shares for tokens. 

If the user still wants withdraw his tokens from strategies, his only option is to complete the queued withdrawal by calling `completeQueuedWithdrawal()` with `receiveAsTokens = false`, and then creating a new queued withdrawal.

### Proof of Concept

Consider the following scenario:
- Bob has some tokens deposited in strategy A.
- There is a newly created strategy with USDC as its underlying token. We call this strategy B.
- Bob calls `deposit()` to deposit 1000 USDC into strategy B, gaining `1e9` shares.
- Some time later, Bob wants to withdraw all his tokens. Therefore:
  - He calls `queueWithdrawal()` with both strategies A and B.
  - `withdrawalDelayBlocks` blocks passes.
  - Bob calls `completeQueuedWithdrawal()` with the queued withdrawal to withdraw his shares for tokens.
- Alice front-runs his transaction, calling `deposit()` to deposit 1 USDC into strategy B. She gains `1e6` shares as a result.
- Now, when Bob's transaction executes, calling `completeQueuedWithdrawal()`:
  - For strategy B, `withdraw()` reverts as the total amount of shares is `1e6` after the withdrawal, which is less than `MIN_NONZERO_TOTAL_SHARES`.
  - `completeQueuedWithdrawal()` reverts.
- Alice has successfully prevented Bob from withdrawing his tokens from both strategies.

### Recommendation

In `completeQueuedWithdrawal()`, allow users to specify which strategies to skip, similar to the `strategyIndexes` array in `slashQueuedWithdrawal()`. Strategies that are skipped should have their shares added again using `_addShares()`.

## [L-01] Lack of validation allows users to queue withdrawals that cannot be completed

### Bug Description

In `StrategyManager.sol`, queued withdrawals cannot be completed if their `delegatedAddress` is "frozen" due to the `onlyNotFrozen()` modifier on `_completeQueuedWithdrawal()`:

[StrategyManager.sol#L745](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L745)

```solidity
function _completeQueuedWithdrawal(
    QueuedWithdrawal calldata queuedWithdrawal,
    IERC20[] calldata tokens,
    uint256 middlewareTimesIndex,
    bool receiveAsTokens
) internal onlyNotFrozen(queuedWithdrawal.delegatedAddress) {
```

However, `queueWithdrawal()` does not ensure that `delegatedAddress` is not "frozen" before storing the queued withdrawal. This allows users to queue withdrawals with a `delegatedAddress` that is already "frozen".

### Impact

Users can queue withdrawals with a "frozen" `delegatedAddress`, even though it cannot be completed. The shares from these queued withdrawals will be stuck as they cannot be withdrawn, therefore the corresponding assets will also be stuck in their respective strategies. 

Although owners can "rescue" these shares using `slashQueuedWithdrawal()`, this is not a sufficient mitigation as it relies on the owner, thereby creating centralization risk.

### Recommendation

In `queueWithdrawal()`, ensure that a user's `delegatedAddress` is not already "frozen":

[StrategyManager.sol#L384-L387](https://github.com/code-423n4/2023-04-eigenlayer/blob/main/src/contracts/core/StrategyManager.sol#L384-L387)

```diff
        // fetch the address that the `msg.sender` is delegated to
        address delegatedAddress = delegation.delegatedTo(msg.sender);
+       require(!slasher.isFrozen(delegatedAddress), "delegatedAddress is frozen");

        QueuedWithdrawal memory queuedWithdrawal;
```


