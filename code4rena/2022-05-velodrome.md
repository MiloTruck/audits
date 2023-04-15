# Velodrome Finance

The code under review can be found in [2022-05-velodrome](https://github.com/code-423n4/2022-05-velodrome).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| M-01 | `Bribe.sol` is not meant to handle fee-on-transfer tokens | Medium |

# [M-01] `Bribe.sol` is not meant to handle fee-on-transfer tokens

## Impact
Should a fee-on-transfer token be added as a reward token and deposited, the tokens will be locked in the `Bribe` contract. Voters will be unable to withdraw their rewards.

## Vulnerability Details
Tokens are deposited into the `Bribe` contract using `notifyRewardAmount()`, where `amount` of tokens are transferred, then added directly to `tokenRewardsPerEpoch[token][adjustedTstamp]`:
```js
  _safeTransferFrom(token, msg.sender, address(this), amount);
  tokenRewardsPerEpoch[token][adjustedTstamp] = epochRewards + amount;
```

Tokens are transferred out of the `Bribe` contract using `deliverReward()`, which attempts to transfer `tokenRewardsPerEpoch[token][epochStart]` amount of tokens out.
```js
function deliverReward(address token, uint epochStart) external lock returns (uint) {
  require(msg.sender == gauge);
  uint rewardPerEpoch = tokenRewardsPerEpoch[token][epochStart];
  if (rewardPerEpoch > 0) {
      _safeTransfer(token, address(gauge), rewardPerEpoch);
  }
  return rewardPerEpoch;
}
```

If `token` happens to be a fee-on-transfer token, `deliverReward()` will always fail. For example:
* User calls `notifyRewardAmount()`, with `token` as token that charges a 2% fee upon any transfer, and `amount = 100`:
  * `_safeTransferFrom()` only transfers 98 tokens to the contract due to the 2% fee
  * Assuming `epochRewards = 0`, `tokenRewardsPerEpoch[token][adjustedTstamp]` becomes `100`
* Later on, when `deliverReward()` is called with the same `token` and `epochStart`:
  * `rewardPerEpoch = tokenRewardsPerEpoch[token][epochStart] = 100`
  * `_safeTransfer` attempts to transfer 100 tokens out of the contract
  * However, the contract only contains 98 tokens
  * `deliverReward()` reverts

## Proof of Concept

The following test, which implements a [MockERC20 with fee-on-transfer](https://gist.github.com/MiloTruck/6fe0a13c4d08689b8be8a55b9b14e7e1), demonstrates this: 
```js
// Note that the following test was adapted from Bribes.t.sol
function testFailFeeOnTransferToken() public {
  // Deploy ERC20 token with fee-on-transfer
  MockERC20Fee FEE_TOKEN = new MockERC20Fee("FEE", "FEE", 18);

  // Mint FEE token for address(this)
  FEE_TOKEN.mint(address(this), 1e25);

  // vote
  VELO.approve(address(escrow), TOKEN_1);
  escrow.create_lock(TOKEN_1, 4 * 365 * 86400);
  vm.warp(block.timestamp + 1);

  address[] memory pools = new address[](1);
  pools[0] = address(pair);
  uint256[] memory weights = new uint256[](1);
  weights[0] = 10000;
  voter.vote(1, pools, weights);

  // and deposit into the gauge!
  pair.approve(address(gauge), 1e9);
  gauge.deposit(1e9, 1);

  vm.warp(block.timestamp + 12 hours); // still prior to epoch start
  vm.roll(block.number + 1);
  assertEq(uint(gauge.getVotingStage(block.timestamp)), uint(Gauge.VotingStage.BribesPhase));

  vm.warp(block.timestamp + 12 hours); // start of epoch
  vm.roll(block.number + 1);
  assertEq(uint(gauge.getVotingStage(block.timestamp)), uint(Gauge.VotingStage.VotesPhase));

  vm.warp(block.timestamp + 5 days); // votes period over
  vm.roll(block.number + 1);

  vm.warp(2 weeks + 1); // emissions start
  vm.roll(block.number + 1);

  minter.update_period();
  distributor.claim(1); // yay this works

  vm.warp(block.timestamp + 1 days); // next votes period start
  vm.roll(block.number + 1);

  // get a bribe
  owner.approve(address(FEE_TOKEN), address(bribe), TOKEN_1);
  bribe.notifyRewardAmount(address(FEE_TOKEN), TOKEN_1);

  vm.warp(block.timestamp + 5 days); // votes period over
  vm.roll(block.number + 1);

  // Atttempt to claim tokens will revert
  voter.distro(); // bribe gets deposited in the gauge
}
```

## Additional Impact
On a larger scale, a malicious attacker could temporarily DOS any `Gauge` contract. This can be done by:
1. Depositing a fee-on-transfer token into its respective `Bribe` contract, using `notifyRewardAmount()`, and adding it as a reward token.
2. This would cause `deliverBribes()` to fail whenever it is called, thus no one would be able to withdraw any reward tokens from the `Gauge` contract.

The only way to undo the DOS would be to call `swapOutBribeRewardToken()` and swap out the fee-on-transfer token for another valid token.

## Recommended Mitigation
* The amount of tokens received should be added to `epochRewards` and stored in `tokenRewardsPerEpoch[token][adjustedTstamp]`, instead of the amount stated for transfer. For example:
```js
  uint256 _before = IERC20(token).balanceOf(address(this));
  _safeTransferFrom(token, msg.sender, address(this), amount);
  uint256 _after = IERC20(token).balanceOf(address(this));

  tokenRewardsPerEpoch[token][adjustedTstamp] = epochRewards + (_after - _before);
```
* Alternatively, disallow tokens with fee-on-transfer mechanics to be added as reward tokens.