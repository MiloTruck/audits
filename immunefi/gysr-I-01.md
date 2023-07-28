# Precision loss in `unstake()` allows an attacker to steal other staker's tokens

## Details

### Protocol

[GYSR](https://immunefi.com/bounty/gysr/)

### Target

https://etherscan.io/address/0xaef2a7E4CbEB875475cFb1924867B9374569D894

### Severity

Informational

## Report

### Bug description

In the `stake()` function of the `ERC20StakingModule` contract, the amount of shares minted to users when they stake is calculated as follows:

https://github.com/gysr-io/core/blob/master/contracts/ERC20StakingModule.sol#L102-L112

```solidity
// transfer
uint256 total = _token.balanceOf(address(this));
_token.safeTransferFrom(user, address(this), amount);
uint256 actual = _token.balanceOf(address(this)) - total;

// mint staking shares at current rate
uint256 minted =
    (totalShares > 0)
        ? (totalShares * actual) / total
        : actual * INITIAL_SHARES_PER_TOKEN;
require(minted > 0, "sm2");
```

As seen from above, if the total number of shares if non-zero, the number of shares minted will follow the formula below:

```
totalShares * tokens staked / total tokens staked
```

However, as the contract uses `_token.balanceOf(address(this))` to determine the amount of tokens staked, this calculation can be manipulated. When `totalShares` is small, an attacker can directly transfer staking tokens to this contract. As the amount of tokens per share increases, future calls to `stake()` will mint less shares as a result.

This is exploitable due to the `unstake()` function, which calculates the amount of shares to burn as such:

https://github.com/gysr-io/core/blob/master/contracts/ERC20StakingModule.sol#L130-L138

```solidity
// validate and get shares
uint256 burned = _shares(user, amount);

// burn shares
totalShares -= burned;
shares[user] -= burned;
```

`amount` represents the amount of tokens a user wishes to unstake. The `_shares()` function calculates the amount of shares to burn as such:

https://github.com/gysr-io/core/blob/master/contracts/ERC20StakingModule.sol#L197-L200

```solidity
// convert token amount to shares
shares_ = (totalShares * amount) / _token.balanceOf(address(this));

require(shares_ > 0, "sm5");
```

When the amount of tokens per share is large, this calculation is susceptible to precision loss. For example:

* Assume `totalShares = 1e6` and `token.balanceOf(address(this)) = 1e18`.
* The amount of tokens per share is `1e18 / 1e6 = 1e12`.
* If a user calls `unstake()` with `amount = 2e12 - 1`, `shares_` will round down to 1.
* Even though the user only burns 1 share, he withdraws `2e12 - 1` worth of tokens, which is 100% more than intended.

When `totalShares` is still 0, an attacker can abuse this to steal staked tokens from other users:

* Assume the staking token used has 18 decimals.
* Attacker calls `stake()` with `amount = 1`.
    * As `totalShares` is 0, he receives `1e6` shares:

```
actual * INITIAL_SHARES_PER_TOKEN = 1 * 1e6 = 1e6
```

* Attacker directly transfers 1000 tokens (`amount = 1000 * 1e18`) to the contract.
* Afterwards, some users stake a significant amount of tokens, such as 10000 tokens. They call `stake()` with `amount = 10000 * 1e18`.
    * As `totalShares` is now non-zero, they receive `1e7` shares:

```
(totalShares * actual) / total = (1e6 * 10000 * 1e18) / (1000 * 1e18) = 1e7
```

* Now, `totalShares` and the total amount of tokens is as such:
  * `totalShares = 1e7 + 1e6 = 11000000`.
  * `token.balanceOf(address(this)) = 1000 * 1e18 + 10000 * 1e18 = 11e21`
* One share is worth `11e21 / 11000000 = 1e15` tokens.
* The attacker repeatedly calls `unstake()` to profit from precision loss:
    * The formula for `amount` is `token.balanceOf(address(this)) / totalShares * 2 - 1`.
    * For the first call, `amount = 11e21 / 11000000 * 2 - 1 = 1999999999999999`.
    * `shares_` will round down to 1:

```
totalShares * amount / token.balanceOf(address(this)) = 11000000 * 1999999999999999 / 11e21 = 1
```

* Thus, even though 1 share is only worth `1e15` tokens, the attacker can unstake `1999999999999999` tokens. He is able to unstake 100% more tokens than he should be able to with 1 share.
* By repeatedly calling `unstake()` to burn 1 share with `amount` as large as possible, the attacker will receive much more tokens than he originally staked, causing a loss of staked funds for other stakers.

### Impact

In a staking module, whenever the total number of shares is extremely low or 0, an attacker can manipulate the contract's accounting to directly steal funds from future stakers.

As this protocol allows regular users to create their own staking pools, a scenario where a newly created staking module has 0 total shares is not unlikely. Therefore, the likelihood of such an attack occuring is increased.

### Recommendation

In the `stake()` function, avoid using `token.balanceOf(address(this))` to determine the amount of tokens staked in the contract. Instead, use a storage variable to keep track of the amount of tokens staked whenever a user calls `stake()` or `unstake()`.

### Proof of Concept

The attached gist contains a Foundry test that demonstrates how an attacker can steal funds from stakers as described above.

https://gist.github.com/MiloTruck/3df554474999b3d48f823434744b1cc2

To run the test:

1. Create a Foundry project:

```
forge init gysr
cd gysr
```

2. Copy the following Solidity files into the `src` directory:
   * `IPool.sol` from https://github.com/gysr-io/core/blob/master/contracts/interfaces/IPool.sol
   * `ERC20.sol` from https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC20.sol
3. Run the test using your mainnet RPC URL:

```
forge test --fork-url <MAINNET_RPC_URL>
```

Note that the test might take a while to complete due to the repeated calls to `unstake()`.