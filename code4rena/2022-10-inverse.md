# Inverse Finance

The code under review can be found in [2022-10-inverse](https://github.com/code-423n4/2022-10-inverse).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-users-can-avoid-paying-their-dbr-deficit) | Users can avoid paying their DBR deficit | Medium |

## [M-01] Users can avoid paying their DBR deficit

### Impact
In both `DolaBorrowingRights` and `Market` contracts, the function `forceReplenish()` in `Market` is the only way to ensure a user pays his DBR deficit. Thus, if users repay their debt before `forceReplenish()` is ever called, they will avoid paying their DBR deficit.

### Vulnerability Details

The function `forceReplenish()` in the `Market` contract is as shown:
```solidity
src/Market.sol:
499:        function forceReplenish(address user, uint amount) public {
500:            uint deficit = dbr.deficitOf(user);
501:            require(deficit > 0, "No DBR deficit");
502:            require(deficit >= amount, "Amount > deficit");
503:            uint replenishmentCost = amount * dbr.replenishmentPriceBps() / 10000;
504:            uint replenisherReward = replenishmentCost * replenishmentIncentiveBps / 10000;
505:            debts[user] += replenishmentCost;
506:            uint collateralValue = getCollateralValueInternal(user);
507:            require(collateralValue >= debts[user], "Exceeded collateral value");
508:            totalDebt += replenishmentCost;
509:            dbr.onForceReplenish(user, amount);
510:            dola.transfer(msg.sender, replenisherReward);
511:            emit ForceReplenish(user, msg.sender, amount, replenishmentCost, replenisherReward);
512:        }
```

The function  mints DBR for the user to restore their DBR deficit to 0, while adding this cost (known as `replenishmentCost`) to the users DOLA debt (line 505):
```solidity
debts[user] += replenishmentCost;
```

However, as the contract relies on providing incentives for other user to call this function, it is not guranteed that this function will ever be called before a user fully repays their debt and withdraws all their collateral. After that, calling `forceReplenish()` to impose DOLA debt would have no effect on the user as no collateral is at stake for him.

### Recommended Mitigation Steps
In the `withdrawInternal()` function, call `forceReplenish()` before calculating the amount of collateral users are able to withdraw. This would force users to pay their DBR deficit before withdrawing their collateral.