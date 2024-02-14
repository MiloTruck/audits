# Morpho Blue

The code under review can be found in [morpho-blue](https://github.com/morpho-org/morpho-blue/tree/f463e40f776acd0f26d0d380b51cfd02949c8c23).

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [H-01](#h-01-markets-can-be-created-for-tokens-without-code) | Markets can be created for tokens without code | High |
| [M-01](#m-01-deviation-in-oracle-price-could-lead-to-arbitrage-in-high-lltv-markets) | Deviation in oracle price could lead to arbitrage in high LLTV markets | Medium |
| [M-02](#m-02-users-can-take-advantage-of-low-liquidity-markets-to-inflate-the-interest-rate) | Users can take advantage of low liquidity markets to inflate the interest rate | Medium |
| [L-01](#l-01-positions-might-not-have-sufficient-liquidation-incentive-in-high-lltv-markets) | Positions might not have sufficient liquidation incentive in high LLTV markets | Low |
| [I-01](#i-01-markets-can-be-dosed-upon-creation-by-inflating-totalborrowshares) | Markets can be DOSed upon creation by inflating `totalBorrowShares` | Informational |
| [I-02](#i-02-max_liquidation_incentive_factor-is-incorrect-in-documentation) | `MAX_LIQUIDATION_INCENTIVE_FACTOR` is incorrect in documentation | Informational |

### [H-01] Markets can be created for tokens without code

**Context:**
- [SafeTransferLib.sol](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/libraries/SafeTransferLib.sol)
- [Morpho.sol#L150-L161](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/Morpho.sol#L150-L161)

**Description:** 

Morpho Blue uses the `SafeTransferLib` library to perform token transfers, more specifically, to call `transfer()` and `transferFrom()`.

Since `SafeTransferLib` performs these calls using low-level `.call()`, both `safeTransfer()` and `safeTransferFrom()` will not revert if the `token` address is a contract with no code.

This responsibility is delegated to the market creator:

```solidity
/// @dev It is the responsibility of the market creator to make sure that the address of the token has non-zero code.
library SafeTransferLib {
```

However, this allows an attacker to create fake balances for not-yet-existing ERC20 tokens. 

Some protocols deploy their token across multiple networks, and when they do so, a common practice is to deploy the token contract from the same deployer address and with the same nonce so that the token address can be the same for all the networks.

For example: 
- 1INCH uses the same token address for both [Ethereum](https://etherscan.io/token/0x111111111117dC0aa78b770fA6A738034120C302) and [BSC](https://bscscan.com/address/0x111111111117dc0aa78b770fa6a738034120c302)
- Gelato's GEL token uses the same token address for [Ethereum, Fantom and Polygon](https://docs.gelato.network/gelato-dao/gel-token-contracts).


An attacker can exploit this to set traps and potentially steal funds from unsuspecting users:

- 1INCH wants to deploy their token on Polygon.
- Before the token is deployed, Alice does the following in Morpho Blue on Polygon:
  - Call `createMarket()` with `0x111111111117dC0aa78b770fA6A738034120C302` as `collateralToken`, which is the 1INCH token address.
  - Call `supplyCollateral()` to give herself a large collateral balance.
- Afterwards, the 1INCH token is deployed.
- Bob wants to use his 1INCH tokens as collateral in Morpho Blue.
  - He calls `supplyCollateral()` and deposits his tokens into the market created by Alice.
- Alice can now call `withdrawCollateral()` to steal Bob's tokens.

Apart from tokens on multiple chains, another form of tokens that have pre-determined addresses are tokens created from factory contracts. 

For example, the addresses of all Uniswap V2 LP tokens are known before deployment, since [`UniswapV2Pair` is created using `CREATE2`](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Factory.sol#L28-L32) in `UniswapV2Factory`. Since their addresses are pre-determined, an attacker can perform the same attack mentioned above before the `UniswapV2Pair` is deployed to potentially steal funds.

The following POC demonstrates how an attacker can create a market and give himself unlimited supply or collateral for non-existing tokens:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.0;

import "test/forge/BaseTest.sol";

contract EOATokenTest is BaseTest {
    using MarketParamsLib for MarketParams;

    function test_canCreateMarketWithEOAToken() public {
        // Create market with loanToken and collateralToken without code
        marketParams = MarketParams({
            loanToken: address(0xc4fe),
            collateralToken: address(0xdead),
            oracle: address(oracle),
            irm: address(irm),
            lltv: DEFAULT_TEST_LLTV
        });
        id = marketParams.id();
        morpho.createMarket(marketParams);

        // Give ONBEHALF a large amount of loanToken
        morpho.supply(marketParams, 1e30, 0, ONBEHALF, hex"");
        assertEq(morpho.market(id).totalSupplyAssets, 1e30);
        
        // Give ONBEHALF a large amount of collateralTOken
        morpho.supplyCollateral(marketParams, 1e30, ONBEHALF, hex"");
        assertEq(morpho.position(id, ONBEHALF).collateral, 1e30);
    }
}
```

**Recommendation:** 

In `createMarket()`, check that `loanToken` and `collateralToken` contain code:

```diff
  function createMarket(MarketParams memory marketParams) external { 
+     require(marketParams.loanToken.length != 0, ErrorsLib.LOANTOKEN_NO_CODE);
+     require(marketParams.collateralToken.length != 0, ErrorsLib.COLLATERALTOKEN_NO_CODE);
```

### [M-01] Deviation in oracle price could lead to arbitrage in high LLTV markets

**Context:**

- [Morpho.sol#L521-L522](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/Morpho.sol#L521-L522)
- [ChainlinkOracle.sol#L116-L121](https://github.com/morpho-org/morpho-blue-oracles/blob/d351d3e59b207729d785ec568ed0d2ee24498189/src/ChainlinkOracle.sol#L116-L121)

**Description:**

In Morpho Blue, the maximum amount a user can borrow is calculated with the conversion rate between `loanToken` and `collateralToken` returned by an oracle:

```solidity
uint256 maxBorrow = uint256(position[id][borrower].collateral).mulDivDown(collateralPrice, ORACLE_PRICE_SCALE)
    .wMulDown(marketParams.lltv);
```

`collateralPrice` is fetched by calling the oracle's `price()` function. For example, the `price()` function in `ChainlinkOracle.sol` is as such.

```solidity
function price() external view returns (uint256) {
    return SCALE_FACTOR.mulDiv(
        VAULT.getAssets(VAULT_CONVERSION_SAMPLE) * BASE_FEED_1.getPrice() * BASE_FEED_2.getPrice(),
        QUOTE_FEED_1.getPrice() * QUOTE_FEED_2.getPrice()
    );
}
```

However, all price oracles are susceptible to front-running as their prices tend to lag behind an asset's real-time price. More specifically:

- Chainlink oracles are updated after the change in price crosses a deviation threshold, (eg. [2.5% in ETH / USD](https://data.chain.link/ethereum/mainnet/crypto-usd/eth-usd)), which means a price feed could return a value slightly smaller/larger than an asset's actual price under normal conditions.
- Uniwap V3 TWAP returns the average price over the past X number of blocks, which means it will always lag behind the real-time price.

An attacker could exploit the difference between the price reported by an oracle and the asset's actual price to gain a profit by front-running the oracle's price update. 

For Morpho Blue, this becomes profitable when the price deviation is sufficiently large for an attacker to open positions that become bad debt. Mathematically, arbitrage is possible when:

$$ price\ deviation \gt { 1 \over LIF } - LLTV $$

The likelihood of this condition becoming true is significantly increased when `ChainlinkOracle.sol` is used as the market's oracle with multiple Chainlink price feeds. 

As seen from above, the conversion rate between `loanToken` and `collateralToken` is calculated with multiple price feeds, with each of them having their own deviation threshold. This amplifies the maximum possible price deviation returned by `price()`. 

For example: 
- Assume a market has WBTC as `collateralToken` and FTM as `loanToken`.
- Assume the following prices:
  - 1 BTC = 40,000 USD
  - 1 FTM = 1 USD
  - 1 ETH = 2000 USD
- `ChainlinkOracle` will be set up as such:
  - `BASE_FEED_1` - [WBTC / BTC](https://data.chain.link/ethereum/mainnet/crypto-other/wbtc-btc), 2% deviation threshold
  - `BASE_FEED_2` - [BTC / USD](https://data.chain.link/ethereum/mainnet/crypto-usd/btc-usd), 0.5% deviation threshold
  - `QUOTE_FEED_1` - [FTM / ETH](https://data.chain.link/ethereum/mainnet/crypto-eth/ftm-eth), 3% deviation threshold
  - `QUOTE_FEED_2` - [ETH / USD](https://data.chain.link/ethereum/mainnet/crypto-usd/eth-usd), 0.5% deviation threshold
- Assume that all price feeds are at their deviation threshold:
  - WBTC / BTC returns 98% of `1`, which is `0.98`.
  - BTC / USD returns 99.5% of `40000`, which is `39800`.
  - FTM / ETH returns 103% of `1 / 2000`, which is `0.000515`.
  - ETH / USD returns 100.5% of `2000`, which is `2010`.
- The actual conversion rate of WBTC to FTM is:
  - `(0.98 * 39800) / (0.000515 * 2010) = 37680`
  - i.e. 1 WBTC = 37,680 FTM.
- Compared to 1 WBTC = 40,000 FTM, the maximum price deviation is 5.8%.

To demonstrate how a such a deviation in price could lead to arbitrage:
- Assume the following:
  - A market has 95% LLTV, with WBTC as collateral and FTM as `loanToken`.
  - 1 WBTC is currently worth 40,000 FTM.
- The price of WBTC drops while FTM increases in value, such that 1 WBTC = 37,680 FTM.
- All four Chainlink price feeds happen to be at their respective deviation thresholds as described above, which means the oracle's price is not updated in real time.
- An attacker sees the price discrepancy and front-runs the oracle price update to do the following:
  - Deposit 1 WBTC as collateral.
  - Borrow 38,000 FTM, which is the maximum he can borrow at 95% LLTV and 1 WBTC = 40,000 FTM conversion rate.
- Afterwards, the oracle's conversion rate is updated to 1 WBTC = 37,680 FTM:
  - Attacker's position is now unhealthy as his collateral is worth less than his loaned amount. 
- Attacker back-runs the oracle price update to liquidate himself:
  - At 95% LLTV, LIF = 100.152%.
  - To seize 1 WBTC, he repays 37,115 FTM:
    - `seizedAssets / LIF = 1 WBTC / 1.0152 = 37680 FTM / 1.0152 = 37115 FTM`
- He has gained 885 FTM worth of profit using 37,680 FTM, which is a 2.3% arbitrage opportunity.

This example proves the original condition stated above for arbitrage to occur, as:

$$ price\ deviation - ({ 1 \over LIF } - LLTV) = 5.8\% - ({ 1 \over 100.152\% } - 95\%)=\ \sim2.3\%$$

Note that all profit gained from arbitrage causes a loss of funds for lenders as the remaining bad debt is socialized by them. 

**Recommendation:**

Consider implementing a borrowing fee to mitigate against arbitrage opportunities. Ideally, this fee would be larger than the oracle's maximum price deviation so that it is not possible to profit from arbitrage.

Further possible mitigations have also been explored by other protocols:

- [Angle Protocol: Oracles and Front-Running](https://medium.com/angle-protocol/angle-research-series-part-1-oracles-and-front-running-d75184abc67)
- [Liquity: The oracle conundrum](https://www.liquity.org/blog/the-oracle-conundrum)

### [M-02] Users can take advantage of low liquidity markets to inflate the interest rate

**Context:**

- [Morpho.sol#L471-L476](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/Morpho.sol#L471-L476)
- [AdaptiveCurveIrm.sol#L117-L120](https://github.com/morpho-org/morpho-blue-irm/blob/c2b1732fc332d20a001ca505aea76bd475e95ef1/src/AdaptiveCurveIrm.sol#L117-L120)
- [Morpho.sol#L477](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/Morpho.sol#L477)
- [Morpho.sol#L180-L181](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/Morpho.sol#L180-L181)
- [SharesMathLib.sol#L15-L21](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/libraries/SharesMathLib.sol#L15-L21)

**Description:**

Morpho Blue is meant to work with stateful Interest Rate Models (IRM) - whenever `_accrueInterest()` is called, it calls `borrowRate()` of the IRM contract:

```solidity
function _accrueInterest(MarketParams memory marketParams, Id id) internal {
    uint256 elapsed = block.timestamp - market[id].lastUpdate;

    if (elapsed == 0) return;

    uint256 borrowRate = IIrm(marketParams.irm).borrowRate(marketParams, market[id]);
```

This will adjust the market's interest rate based on the current state of the market. For example, `AdaptiveCurveIrm.sol` adjusts the interest rate based on the market's current utilization rate:

```solidity
function _borrowRate(Id id, Market memory market) private view returns (uint256, int256) {
    // Safe "unchecked" cast because the utilization is smaller than 1 (scaled by WAD).
    int256 utilization =
        int256(market.totalSupplyAssets > 0 ? market.totalBorrowAssets.wDivDown(market.totalSupplyAssets) : 0);
```

However, this stateful implementation will always call `borrowRate()` and adjust the interest rate, even when it should not.

For instance, in `AdaptiveCurveIrm.sol`, an attacker can manipulate the market's utilization rate as such:

- Create market with a legitimate `loanToken`, `collateralToken`, `oracle` and the IRM as `AdaptiveCurveIrm.sol`. 
- Call `supply()` to supply 1 wei of `loanToken` to the market.
- Call `supplyCollateral()` to give himself some collateral.
- Call `borrow()` to borrow the 1 wei of `loanToken`.
- Now, the market's utilization rate is 100%. 
- Afterwards, if no one supplies any `loanToken` to the market for a long period of time, `AdaptiveCurveIrm.sol` will aggressively increase the market's interest rate.

This is problematic as Morpho Blue's interest compounds based on $e^x$:

```solidity
uint256 interest = market[id].totalBorrowAssets.wMulDown(borrowRate.wTaylorCompounded(elapsed));
```

As such, when `borrowRate` (the interest rate) increases, `interest` will grow at an exponential rate, which could cause the market's `totalSupplyAssets` and `totalBorrowAssets` to become extremely huge.

This creates a few issues:

1. The market will have a huge amount of un-clearable bad debt:

Should a large amount of interest accrue, `totalBorrowAssets` will be extremely large, even though `totalBorrowShares` is only `1e6` shares. Half of `totalBorrowAssets` would have actually accrued to the other `1e6` [virtual shares](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/libraries/SharesMathLib.sol#L15-L21).

As such, after liquidating the attacker's `1e6` shares, half of `totalBorrowAssets` will still remain in the market as un-clearable bad debt.

2. The market will permanently have a high interest rate:

As mentioned above, `AdaptiveCurveIrm.sol` aggressively increased the market's interest rate while there was only 1 wei supplied and borrowed in the market, causing utilization to be 100%. 

If other lenders decide to supply `loanToken` to the market, borrowers would still be discouraged from borrowing for an extended period of time as `AdaptiveCurveIrm.sol` would have to adjust the market's interest rate back down.

3. Users who call `supply()` with a small amount of assets might lose funds:

If `totalSupplyAssets` is sufficiently large compared to `totalSupplyShares`, the market's shares to assets ratio will be huge. This will cause the following the share calculation in `supply()` to round down to 0:

```solidity
if (assets > 0) shares = assets.toSharesDown(market[id].totalSupplyAssets, market[id].totalSupplyShares);
else assets = shares.toAssetsUp(market[id].totalSupplyAssets, market[id].totalSupplyShares);
```

Should this occur, the user will receive 0 shares when depositing assets, resulting in a loss of funds.

The following PoC demonstrates how the market's interest rate can be inflated, as described above. Note that this PoC has to be placed in the `morpho-blue-irm` repository.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "lib/forge-std/src/Test.sol";
import "src/AdaptiveCurveIrm.sol";
import {BaseTest} from "lib/morpho-blue/test/forge/BaseTest.sol";

contract CreamyInflationAttack is BaseTest {
    using MarketParamsLib for MarketParams;

    int256 constant CURVE_STEEPNESS = 4 ether;
    int256 constant ADJUSTMENT_SPEED = int256(20 ether) / 365 days;
    int256 constant TARGET_UTILIZATION = 0.9 ether; // 90%
    int256 constant INITIAL_RATE_AT_TARGET = int256(0.1 ether) / 365 days; // 10% APR

    function setUp() public override {
        super.setUp();

        // Deploy and enable AdaptiveCurveIrm
        AdaptiveCurveIrm irm = new AdaptiveCurveIrm(
            address(morpho), CURVE_STEEPNESS ,ADJUSTMENT_SPEED,TARGET_UTILIZATION, INITIAL_RATE_AT_TARGET
        );
        vm.prank(OWNER);
        morpho.enableIrm(address(irm));
        
        // Deploy market with AdaptiveCurveIrm
        marketParams = MarketParams({
            loanToken: address(loanToken), // Pretend this is USDC
            collateralToken: address(collateralToken), // Pretend this is USDT
            oracle: address(oracle),
            irm: address(irm),
            lltv: DEFAULT_TEST_LLTV
        });
        id = marketParams.id();
        morpho.createMarket(marketParams);
    }

    function testInflateInterestRateWhenLowLiquidity() public {
        // Supply and borrow 1 wei
        _supply(1);
        collateralToken.setBalance(address(this), 2);
        morpho.supplyCollateral(marketParams, 2, address(this), "");
        morpho.borrow(marketParams, 1, 0, address(this), address(this));

        // Accrue interest for 150 days
        for (uint i = 0; i < 150; i++) {
            skip(1 days);
            morpho.accrueInterest(marketParams);
        }

        // Liquidating only divides assets by 2, the other half accrues to virtual shares
        loanToken.setBalance(address(this), 2);
        morpho.liquidate(marketParams, address(this), 2, 0, "");

        // Shares to assets ratio is now insanely high
        console2.log("supplyAssets: %d, supplyShares: %d", morpho.market(id).totalSupplyAssets, morpho.market(id).totalSupplyShares);
        console2.log("borrowAssets: %d, borrowShares: %d", morpho.market(id).totalBorrowAssets, morpho.market(id).totalBorrowShares);
        
        // Supply 1M USDC, but gets no shares in return
        loanToken.setBalance(address(this), 1_000_000e6);
        morpho.supply(marketParams, 1_000_000e6, 0, SUPPLIER, "");
        assertEq(morpho.position(id, SUPPLIER).supplyShares, 0);
    }
}
```

**Recommendation:**

In `_accrueInterest()`, consider checking that `totalSupplyAssets` is sufficiently large for `IIrm.borrowRate()` to be called. 

This prevents the IRM from adjusting the interest rate when the utilization rate is "falsely" high (e.g. only 1 wei supplied and borrowed, resulting in 100% utilization rate).

### [L-01] Positions might not have sufficient liquidation incentive in high LLTV markets

**Context:** 
- [ConstantsLib.sol#L10-L11](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/libraries/ConstantsLib.sol#L10-L11)
- [Morpho.sol#L112-L120](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/Morpho.sol#L112-L120)
- [Morpho.sol#L363-L367](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/Morpho.sol#L363-L367)

**Description:** 

In Morpho Blue, markets can be deployed with less than 100% LLTV:

```solidity
require(lltv < WAD, ErrorsLib.MAX_LLTV_EXCEEDED);
```
 
The market's LLTV is used to calculate the additional percentage of collateral that liquidators earn from liquidating unhealthy positions, known as the liquidation incentive factor (LIF):

```solidity
// The liquidation incentive factor is min(maxLiquidationIncentiveFactor, 1/(1 - cursor*(1 - lltv))).
uint256 liquidationIncentiveFactor = UtilsLib.min(
    MAX_LIQUIDATION_INCENTIVE_FACTOR,
    WAD.wDivDown(WAD - LIQUIDATION_CURSOR.wMulDown(WAD - marketParams.lltv))
);
```

The calculation for LIF is shown below, where `LIQUIDATION_CURSOR` is `0.3e18` [in Morpho Blue]((https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/libraries/ConstantsLib.sol#L10-L11)):

$$ LIF = { 1 \over 1 - 0.3 (1-LLTV) }$$

However, this method of calculating LIF becomes a problem in markets with a high LLTV, since LIF will be extremely small. For example, in a market where LLTV is 95%, LIF will only be ~101.52%, which means liquidators only earn 1.52% when liquidating.

Should the 1.52% earned from liquidation be less than the gas cost of calling `liquidate()`, liquidators will no longer have any incentive to liquidate unhealthy positions. For example:
- Assume a market has a 95% LLTV.
- A borrower deposits 100 USD worth of collateral, which allows him to borrow 95 USD worth of loan tokens.
- His collateral depreciates in value, causing it to be worth only 98 USD. His position is now unhealthy.
- Assume that the current gas cost of calling `liquidate()` is 2 USD.
- With a ~101.52% LIF, a liquidator will get 99.49 USD worth of collateral in return for repaying his entire position, earning only USD 1.49.
- As such, there is no incentive for anyone to liquidate the position.

An attacker could even take advantage of this by opening multiple small positions rather than a single large one, forcing liquidators to call `liquidate()` multiple times and pay more gas for him to be entirely liquidated.

If there are many such unhealthy positions left unliquidated in a market, it could potentially harm:
- Lenders, since debts are not repaid, they will be unable to withdraw.
- Borrowers, since `totalBorrowAssets` will be larger than it should be, causing the interest rate to grow at a faster rate.

Note that markets with a smaller LLTV also face the same problem, just that the size of positions that do not have sufficient liquidation incentive is smaller. For example, at 70% LLTV, LIF will be ~109.9%, which means any position with collateral worth more than 20 USD will be profitable.

**Recommendation:** 

Consider requiring a minimum amount of collateral, which is based on the market's LLTV, for borrowers to open any position. Ideally, this lower bound should be large enough such that any unhealthy position will have sufficient liquidation incentive.   

### [I-01] Markets can be DOSed upon creation by inflating `totalBorrowShares`

**Context:** 
- [SharesMathLib.sol](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/libraries/SharesMathLib.sol)
- [Morpho.sol#L281](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/Morpho.sol#L281)

**Description:** 

When markets are created, the ratio of shares to assets is `1e6`, which means that borrowing `1` asset mints `1e6` shares. This is due to `VIRTUAL_SHARES` and `VIRTUAL_ASSETS`:

```solidity
/// @dev The number of virtual shares has been chosen low enough to prevent overflows, and high enough to ensure
/// high precision computations.
uint256 internal constant VIRTUAL_SHARES = 1e6;

/// @dev A number of virtual assets of 1 enforces a conversion rate between shares and assets when a market is
/// empty.
uint256 internal constant VIRTUAL_ASSETS = 1;
```

In `repay()`, whenever `shares` is specified, the calculation for `asset` rounds up:

```solidity
else assets = shares.toAssetsUp(market[id].totalBorrowAssets, market[id].totalBorrowShares);
```

As such, if `repay()` is called with `shares = 1`, `assets` will also round up to `1`, which means that only `1` share is burned when repaying `1` asset.

However, this does not follow the shares to assets ratio as `1` asset is actually worth `1e6` shares. An attacker can exploit this to inflate `totalBorrowShares`:

1. Assume a market starts with `1e6` shares and `1` asset.
2. Call `borrow()` with `assets = 1`, which mints `1e6` shares.
3. Call `repay()` with `shares = 1`, which burns `1` share and `1` asset.
4. Now, the market has `2e6 - 1` shares and `1` asset.
5. Therefore, the shares to assets ratio has doubled.

By repeating steps 2 and 3, the attacker can inflate `totalBorrowShares` until it is close to `uint128` max. 

This will cause `borrow()` to revert with an arithmetic overflow whenever it is called, since borrowing any substantial amount of assets will attempt to mint an amount of shares greater than `uint128` max, causing the market to be DOSed.

The following POC demonstrates that an attacker needs to repeat steps 2 and 3 exactly 107 times for `borrow()` to always revert when borrowing any amount of assets:

```solidity
// SPDX-License-Identifier: GPL-2.0-or-later
pragma solidity ^0.8.0;

import "test/forge/BaseTest.sol";

contract EOATokenTest is BaseTest {
    using MarketParamsLib for MarketParams;

    function test_canCreateMarketWithEOAToken() public {
        // Add supply and collateral for address(this)
        _supply(1e18);
        _supplyCollateralForBorrower(address(this));

        // Repeatedly repay 1 asset for burning 1 share
        for (uint256 i; i < 108; i++) {
            morpho.borrow(marketParams, 1, 0, address(this), address(this));
            morpho.repay(marketParams, 0, 1, address(this), "");
        }

        // totalBorrowShares is now larger than 1e38
        assertGt(morpho.market(id).totalBorrowShares, 1e38);

        // assets * totalShares overflows when borrowing 1 asset
        vm.expectRevert(stdError.arithmeticError);
        morpho.borrow(marketParams, 1, 0, address(this), address(this));
    }
}
```
 
**Recommendation:** 

Consider adding a lower bound for the amount of shares that is burned in `repay()`.

### [I-02] `MAX_LIQUIDATION_INCENTIVE_FACTOR` is incorrect in documentation

**Context:**

- [ConstantsLib.sol#L13-L14](https://github.com/morpho-org/morpho-blue/blob/f463e40f776acd0f26d0d380b51cfd02949c8c23/src/libraries/ConstantsLib.sol#L13-L14)

**Description:**

The [documentation](https://morpho-labs.notion.site/Morpho-Blue-Documentation-Hub-External-00ff8194791045deb522821be46abbdc) states that `MAX_LIQUIDATION_INCENTIVE_FACTOR` is 20%:

> $LI = min(M, \frac{1}{\beta*LLTV+(1-\beta)} -1)$, with $\beta = 0.3$ and $M= 0.20$ 

However, it is actually 15% in the code:

```solidity
/// @dev Max liquidation incentive factor.
uint256 constant MAX_LIQUIDATION_INCENTIVE_FACTOR = 1.15e18;
```

**Recommendation:**

Amend the documentation to state that $M = 0.15$