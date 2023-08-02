# Polynomial Protocol

The code under review can be found in [2023-03-polynomial](https://github.com/code-423n4/2023-03-polynomial).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [H-01](#h-01-short-positions-can-be-burned-while-holding-collateral) | Short positions can be burned while holding collateral | High |
| [M-01](#m-01-short-positions-with-minimum-collateral-can-be-liquidated-even-though-canliquidate-returns-false) | Short positions with minimum collateral can be liquidated even though `canLiquidate()` returns `false` | Medium |
| [M-02](#m-02-users-can-receive-less-collateral-than-expected-from-liquidations) | Users can receive less collateral than expected from liquidations | Medium |

## [H-01] Short positions can be burned while holding collateral

### Impact

Users can permanently lose a portion of their collateral due to a malicious attacker or their own mistake.

### Vulnerability Details

In the `ShortToken` contract, `adjustPosition()` is used to handle changes to a short position's short or collateral amounts. The function also handles the burning of positions with the following logic:

[ShortToken.sol#L79-L84](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L79-L84)

```solidity
position.collateralAmount = collateralAmount;
position.shortAmount = shortAmount;

if (position.shortAmount == 0) {
    _burn(positionId);
}
```

Where:
* `collateralAmount` - New amount of collateral in a position.
* `shortAmount` - New short amount of a position.
* `positionId` - ERC721 `ShortToken` of a short position.

As seen from above, if a position's `shortAmount` is set to 0, it will be burned. Furthermore, as the code does not ensure `collateralAmount` is not 0 before burning, it is possible to burn a position while it still has collateral. 

If this occurs, the position's owner will lose all remaining collateral in the position. This remaining amount will forever be stuck in the position as its corresponding `ShortToken` no longer has an owner.

### Proof of Concept

In the `Exchange` contract, users can reduce a position's `shortAmount` using [`closeTrade()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L100-L109) and [`liquidate()`](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/Exchange.sol#L140-L148). With these two functions, there are three realistic scenarios where a position with collateral could be burned.

#### 1. User reduces his position's `shortAmount` to 0

A user might call `closeTrade()` on a short position with the following parameters:
* `params.amount` - Set to the position's short amount.
* `params.collateralAmount` - Set to any amount less than the position's total collateral amount.

This would reduce his position's `shortAmount` to 0 without withdrawing all of its collateral, causing him to lose the remaining amount.

Although this could be considered a user mistake, such a scenario could occur if a user does not want to hold a short position temporarily without fully withdrawing his collateral.

#### 2. Attacker fully liquidates a short position

In certain situations, it is possible for a short position to have collateral remaining after a full liquidation (example in the coded PoC below). This collateral will be lost as full liquidations reduces a position's `shortAmount` to 0, thereby burning the position.

#### 3. Attacker frontruns a user's `closeTrade()` transaction with a liquidation

Consider the following scenario:
* Alice has an unhealthy short position that is under the liquidation ratio and can be fully liquidated.
* To bring her position back above the liquidation ratio, Alice decides to partially reduce its short amount. She calls `closeTrade()` on her position with the following parameters:
  * `params.amount` - Set to 40% of the position's short amount.
  * `params.collateralAmount` - Set to 0.
* A malicious attacker, Bob, sees her `closeTrade()` transaction in the mempool.
* Bob frontruns the transaction, calling `liquidate()` with the following parameters:
    * `positionId` - ID of Alice's position.
    * `debtRepaying` - Set to 60% of Alice's position's short amount.
* Bob's `liquidate()` transaction executes first, reducing the short amount of Alice's position to 40% of the original amount.
* Alice's `closeTrade()` transaction executes, reducing her position's short amount by 40% of the original amount, thus its `shortAmount` becomes 0.

In the scenario above, Alice loses the remaining collateral in her short position as it is burned after `closeTrade()` executes. 

Note that this attack is possible as long as an attacker can liquidate the position's remaining short amount. For example, if Alice calls `closeTrade()` with 70% of the position's short amount, Bob only has to liquidate 30% of its short amount. 

#### Coded PoC

The code below contains three tests that demonstrates the scenarios above:
1. `testCloseBurnsCollateral()`
2. `testLiquidateBurnsCollateral()`
3. `testAttackerFrontrunLiquidateBurnsCollateral()`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import {TestSystem, Exchange, ShortToken, ShortCollateral, MockERC20Fail} from "./utils/TestSystem.sol";

contract PositionWithCollateralBurned is TestSystem {
    // Protocol contracts
    Exchange private exchange;
    ShortToken private shortToken;
    ShortCollateral private shortCollateral;

    // sUSD token contract
    MockERC20Fail private SUSD;

    // Intial base asset price
    uint256 private constant initialBaseAssetPrice = 1e18;

    // Users
    address private USER = user_1;
    address private ATTACKER = user_2;

    function setUp() public {
        // Deploy contracts
        deployTestSystem();
        initPool();
        initExchange();
        preparePool();

        exchange = getExchange();
        shortToken = getShortToken();
        shortCollateral = getShortCollateral();
        SUSD = getSUSD();

        // Set initial price for base asset
        setAssetPrice(initialBaseAssetPrice);

        // Mint sUSD for USER
        SUSD.mint(USER, 1e20);

        // Mint powerPerp for ATTACKER
        vm.prank(address(exchange));
        getPowerPerp().mint(ATTACKER, 1e20);
    }

    function testCloseBurnsCollateral() public {
        // Open short position
        uint256 shortAmount = 1e18;
        uint256 collateralAmount = 1e15;
        uint256 positionId = openShort(shortAmount, collateralAmount, USER);

        // Fully close position without withdrawing any collateral
        closeShort(positionId, shortAmount, 0, USER);

        // positionId still holds 1e15 sUSD as collateral
        (,, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingCollateralAmount, collateralAmount);

        // positionId is already burned (ie. ownerOf reverts with "NOT_MINTED")
        vm.expectRevert("NOT_MINTED");
        shortToken.ownerOf(positionId);
    }

    function testLiquidateBurnsCollateral() public {
        // USER opens short position with amount = 1e18, collateral amount = 1e15
        uint256 shortAmount = 1e18;
        uint256 positionId = openShort(1e18, 1e15, USER);

        // Base asset price rises by 35%
        setAssetPrice(initialBaseAssetPrice * 135 / 100);

        // USER's entire short position is liquidatable
        assertEq(shortCollateral.maxLiquidatableDebt(positionId), shortAmount);

        // ATTACKER liquidates USER's entire short position
        vm.prank(ATTACKER);
        exchange.liquidate(positionId, shortAmount);

        // positionId has no remaining debt, but still holds some collateral
        (, uint256 remainingAmount, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingAmount, 0);
        assertGt(remainingCollateralAmount, 0);

        // positionId is already burned (ie. ownerOf reverts with "NOT_MINTED")
        vm.expectRevert("NOT_MINTED");
        shortToken.ownerOf(positionId);
    }

    function testAttackerFrontrunLiquidateBurnsCollateral() public {
        // USER opens short position with amount = 1e18, collateral amount = 1e15
        uint256 shortAmount = 1e18;
        uint256 positionId = openShort(1e18, 1e15, USER);

        // Base asset price rises by 40%
        setAssetPrice(initialBaseAssetPrice * 140 / 100);

        // USER's short position is liquidatable
        assertEq(shortCollateral.maxLiquidatableDebt(positionId), shortAmount);

        // ATTACKER frontruns USER's closeTrade() transaction, liquidating 60% of USER's amount
        vm.prank(ATTACKER);
        exchange.liquidate(positionId, shortAmount * 60 / 100);

        // USER's closeTrade() transaction executes, reducing shortAmount by the remaining 40%
        closeShort(positionId, shortAmount * 40 / 100, 0, USER);

        // positionId has no remaining debt, but still holds some collateral
        (, uint256 remainingAmount, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingAmount, 0);
        assertGt(remainingCollateralAmount, 0);

        // positionId is already burned (ie. ownerOf reverts with "NOT_MINTED")
        vm.expectRevert("NOT_MINTED");
        shortToken.ownerOf(positionId);
    }

    function openShort(
        uint256 amount,
        uint256 collateralAmount,
        address user
    ) internal returns (uint256 positionId) {
        Exchange.TradeParams memory tradeParams;
        tradeParams.amount = amount;
        tradeParams.collateral = address(SUSD);
        tradeParams.collateralAmount = collateralAmount;

        vm.startPrank(user);
        SUSD.approve(address(exchange), collateralAmount);
        (positionId, ) = exchange.openTrade(tradeParams);
        vm.stopPrank();
    }

    function closeShort(
        uint256 positionId,
        uint256 amount,
        uint256 collateralAmount,
        address user
    ) internal {
        Exchange.TradeParams memory tradeParams;
        tradeParams.amount = amount;
        tradeParams.collateral = address(SUSD);
        tradeParams.collateralAmount = collateralAmount;
        tradeParams.maxCost = 100e18;
        tradeParams.positionId = positionId;

        vm.startPrank(user);
        SUSD.approve(address(getPool()), tradeParams.maxCost);
        exchange.closeTrade(tradeParams);
        vm.stopPrank();
    }
}
```

### Recommended Mitigation

Ensure that positions cannot be burned if they have any collateral:

[ShortToken.sol#L82-L84](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortToken.sol#L82-L84)
```diff
-            if (position.shortAmount == 0) {
+            if (position.shortAmount == 0 && position.collateralAmount == 0) {
                 _burn(positionId);
             }
```

## [M-01] Short positions with minimum collateral can be liquidated even though `canLiquidate()` returns `false`

### Impact

Frontends or contracts that rely on `canLiquidate()` to determine if a position is liquidatable could be incorrect. Users could think their positions are safe from liquidation even though they are liquidatable, leading to them losing their collateral.

### Vulnerability Details

In the `ShortCollateral` contract, `canLiquidate()` determines if a short position can be liquidated using the following formula:

[ShortCollateral.sol#L207-L210](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L207-L210)

```solidity
uint256 minCollateral = markPrice.mulDivUp(position.shortAmount, collateralPrice);
minCollateral = minCollateral.mulWadDown(collateral.liqRatio);

return position.collateralAmount < minCollateral;
```

Where:
* `position.collateralAmount` - Amount of collateral in the short position.
* `minCollateral` - Minimum amount of collateral required to avoid liquidation.

From the above, a short position can be liquidated if its collateral amount is **less than** `minCollateral`. This means a short position with the minimum collateral amount (ie. `position.collateralAmount == minCollateral`)  cannot be liquidated. 

However, this is not the case in `maxLiquidatableDebt()`, which is used to determine a position's maximum liquidatable debt:

[ShortCollateral.sol#L230-L237](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L230-L237)

```solidity
uint256 safetyRatioNumerator = position.collateralAmount.mulWadDown(collateralPrice);
uint256 safetyRatioDenominator = position.shortAmount.mulWadDown(markPrice);
safetyRatioDenominator = safetyRatioDenominator.mulWadDown(collateral.liqRatio);
uint256 safetyRatio = safetyRatioNumerator.divWadDown(safetyRatioDenominator);

if (safetyRatio > 1e18) return maxDebt;

maxDebt = position.shortAmount / 2;
```

Where:
* `safetyRatio` - Equivalent to `position.collateralAmount / minCollateral`. Can be seen as a position's collateral amount against the minimum collateral required.  
* `maxDebt` - The amount of debt liquidatable. Defined as 0 at the start of the function.

As seen from the `safetyRatio > 1e18` check, a position is safe from liquidation (ie. `maxDebt = 0`) if  its `safetyRatio` is **greater than** 1. 

Therefore, as a position with the minimum collateral amount has a `safetyRatio` of 1, half its debt becomes liquidatable. This contradicts `canLiquidate()`, which returns `false` for such positions.

### Proof of Concept

The following test demonstrates how a position with minimum collateral is liquidatable even though `canLiquidate()` returns `false`:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import {TestSystem, Exchange, ShortToken, ShortCollateral, PowerPerp, MockERC20Fail} from "./utils/TestSystem.sol";

contract CanLiquidateIsInaccurate is TestSystem {
    // Protocol contracts
    Exchange private exchange;
    ShortToken private shortToken;
    ShortCollateral private shortCollateral;
    
    // sUSD token contract
    MockERC20Fail private SUSD;
    
    function setUp() public {  
        // Set liquidation ratio of sUSD to 125%  
        susdLiqRatio = 1.24e18;

        // Deploy contracts
        deployTestSystem();
        initPool();
        initExchange();
        preparePool();

        exchange = getExchange();
        shortToken = getShortToken();
        shortCollateral = getShortCollateral();
        SUSD = getSUSD();

        // Mint sUSD for user_1
        SUSD.mint(user_1, 1e20);

        // Mint powerPerp for user_2
        vm.prank(address(exchange));
        getPowerPerp().mint(user_2, 1e20);
    }

    function testCanLiquidateMightBeWrong() public {
        // Initial price of base asset is 1e18
        uint256 initialPrice = 1e18;
        setAssetPrice(initialPrice);

        // Open short position with 1e15 sUSD as collateral
        Exchange.TradeParams memory tradeParams;
        tradeParams.amount = 1e18;
        tradeParams.collateral = address(SUSD);
        tradeParams.collateralAmount = 1e15;
        tradeParams.minCost = 0;

        vm.startPrank(user_1);
        SUSD.approve(address(exchange), tradeParams.collateralAmount);
        (uint256 positionId,) = exchange.openTrade(tradeParams);
        vm.stopPrank();
       
        // Initial price of base asset increases, such that minCollateral == collateralAmount
        setAssetPrice(1270001270001905664);

        // canLiquidate() returns false
        assertFalse(shortCollateral.canLiquidate(positionId));

        // However, maxLiquidatableDebt() returns half of original amount
        assertEq(shortCollateral.maxLiquidatableDebt(positionId), tradeParams.amount / 2);

        // Other users can liquidate the short position
        vm.prank(user_2);
        exchange.liquidate(positionId, tradeParams.amount);

        // Position's shortAmount and collateral is reduced
        (, uint256 remainingAmount, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingAmount, tradeParams.amount / 2);
        assertLt(remainingCollateralAmount, tradeParams.collateralAmount);
    }
}
```

### Recommended Mitigation

Consider making short positions safe from liquidation if their `safetyRatio` equals to 1:

[ShortCollateral.sol#L235](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L235)
```diff
-        if (safetyRatio > 1e18) return maxDebt;
+        if (safetyRatio >= 1e18) return maxDebt;
```

## [M-02] Users can receive less collateral than expected from liquidations

### Impact

Users might receive very little or no collateral when liquidating extremely unhealthy short positions.

### Vulnerability Details

When users liquidate a short position, they expect to get a reasonable amount of collateral in return. The collateral amount sent to liquidators is handled by `liquidate()` in the `ShortCollateral` contract:

[ShortCollateral.sol#L137-L141](https://github.com/code-423n4/2023-03-polynomial/blob/main/src/ShortCollateral.sol#L137-L141)
```solidity
totalCollateralReturned = liqBonus + collateralClaim;
if (totalCollateralReturned > userCollateral.amount) totalCollateralReturned = userCollateral.amount;
userCollateral.amount -= totalCollateralReturned;

ERC20(userCollateral.collateral).safeTransfer(user, totalCollateralReturned);
```

Where:
* `liqBonus` - Bonus amount of collateral for liquidation.
* `collateralClaim` - Collateral amount returned, proportional to the how much debt is being liquidated.


As seen from above, if the position does not have sufficient collateral to repay the short amount being liquidated, it simply repays the liquidator with the remaining collateral amount.

This could cause liquidators to receive less collateral than expected, especially if they fully liquidate positions with high short amount to collateral ratios. In extreme cases, if a user liquidates a position with a positive short amount and no collateral (known as bad debt), they would receive no collateral at all. 

### Proof of Concept

The following test demonstrates how a user can liquidate a short position without getting any collateral in return:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.9;

import {TestSystem, Exchange, ShortToken, ShortCollateral, MockERC20Fail} from "./utils/TestSystem.sol";

contract LiquidationOverpay is TestSystem {
    // Protocol contracts
    Exchange private exchange;
    ShortToken private shortToken;
    ShortCollateral private shortCollateral;

    // sUSD token contract
    MockERC20Fail private SUSD;

    // Intial base asset price
    uint256 private constant initialBaseAssetPrice = 1e18;

    function setUp() public {
        // Deploy contracts
        deployTestSystem();
        initPool();
        initExchange();
        preparePool();

        exchange = getExchange();
        shortToken = getShortToken();
        shortCollateral = getShortCollateral();
        SUSD = getSUSD();

        // Set initial price for base asset
        setAssetPrice(initialBaseAssetPrice);

        // Mint sUSD for user_1
        SUSD.mint(user_1, 1e20);

        // Mint powerPerp for user_2 and user_3
        vm.startPrank(address(exchange));
        getPowerPerp().mint(user_2, 1e20);
        getPowerPerp().mint(user_3, 1e20);
        vm.stopPrank();
    }

    function testLiquidationReturnsLessCollateralThanExpected() public {
        // Open short position with amount = 1e18, collateral amount = 1e15 (sUSD)
        uint256 shortAmount = 1e18;
        uint256 positionId = openShort(1e18, 1e15, user_1);

        // Base asset price rises by 50%
        setAssetPrice(initialBaseAssetPrice * 150 / 100);

        // user_2 liquidates 85% USER's entire short position
        vm.prank(user_2);
        exchange.liquidate(positionId, shortAmount * 85 / 100);

        // positionId has no remaining collateral, but still has remaining 15% of debt
        (, uint256 remainingAmount, uint256 remainingCollateralAmount, ) = shortToken.shortPositions(positionId);
        assertEq(remainingAmount, shortAmount * 15 / 100);
        assertEq(remainingCollateralAmount, 0);

        // user_3 liquidates the same position
        vm.prank(user_3);
        exchange.liquidate(positionId, shortAmount);
        
        // user_3 did not get any collateral
        assertEq(SUSD.balanceOf(user_3), 0);
    }

    function openShort(
        uint256 amount,
        uint256 collateralAmount,
        address user
    ) internal returns (uint256 positionId) {
        Exchange.TradeParams memory tradeParams;
        tradeParams.amount = amount;
        tradeParams.collateral = address(SUSD);
        tradeParams.collateralAmount = collateralAmount;

        vm.startPrank(user);
        SUSD.approve(address(exchange), collateralAmount);
        (positionId, ) = exchange.openTrade(tradeParams);
        vm.stopPrank();
    }
}
```

### Recommended Mitigation

In `liquidate()` of the `Exchange` contract, consider adding a `minCollateralAmount` parameter, which represents the minimum amount of collateral a liquidator is willing to receive. . If the returned collateral amount is less than `minCollateralAmount`, the transaction should revert.