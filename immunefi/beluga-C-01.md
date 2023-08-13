# Attacker can manipulate BELA's vote accounting to permanently freeze all BELA in any contract

## Details

### Protocol

Beluga Protocol

_Note: The protocol was permanently removed from Immunefi_

### Target

https://arbiscan.io/address/0x7fbdEb84D5966c1C325D8CB2E01593D74c9A41Cd

### Severity

Critical

## Bug Description

[BELA token](https://arbiscan.io/address/0x09090e22118b375f2c7b95420c04414e4bf68e1a#code) implements a Sushiswap-like vote accounting mechanism, where all user's BELA corresponds to a number of votes. This can be seen through the existence of functions such as `delegate()`, `getCurrentVotes()` and `_moveDelegates()`.

To handle the change in votes when BELA is transferred from one address to another, the contract has overriden the `_transfer` function:

```solidity
    function _transfer(address sender, address recipient, uint256 amount) internal override {
        ERC20._transfer(sender, recipient, amount);
        _moveDelegates(sender, recipient, amount);
    }
```

As seen from above, whenever BELA is transferred, votes will be moved from the `sender` address to the the `recipient` address.

However, this is incorrect as it does not handle cases where the `sender` or `recipient` address has delegated his votes to another address. For example:
- Alice has delegated her votes to Charlie.
- Bob transfers 1000 BELA to Alice.
- As Alice is the `recipient` address, she gains 1000 votes. However, the votes should have gone to Charlie, who is her delegatee.

This is a vulnerability when combined with the `delegate()` function:

```solidity
    function _delegate(address delegator, address delegatee)
        internal
    {
        address currentDelegate = _delegates[delegator];
        uint256 delegatorBalance = balanceOf(delegator); // balance of underlying FarmTokens (not scaled);
        _delegates[delegator] = delegatee;

        emit DelegateChanged(delegator, currentDelegate, delegatee);

        _moveDelegates(currentDelegate, delegatee, delegatorBalance);
    }
```

Whenever `delegate()` is called to change someone's delegatee, votes are removed from the `currentDelegate` and added to the new `delegatee`. The amount of votes transferred is equal to the `delegator`'s balance. 

An attacker can exploit this to reduce the votes of any address:
- Assume the following:
  - Alice and Bob have 1000 BELA each, which gives them 1000 votes each.
  - Charlie has 0 BELA.
  - Alice and Charlie want to reduce Bob's votes to 0.
- Charlie calls `delegate()` to set his delegatee to Bob.
- Alice calls `transfer()` to transfer 1000 BELA to Charlie:
  - Alice's 1000 votes are transferred to Charlie, not Bob.
- Charlie calls `delegate()` and sets his delegatee to himself:
  - As Bob is his delegatee, 1000 votes are subtracted from Bob and added to Charlie.
- Bob now has 0 votes, while Charlie has 2000 votes.

Even if a user has a huge amount of votes, an attacker can execute the sequence above repeatedly to reduce his votes to 0.

This is an issue as all transfers of BELA requires the `sender` to have a sufficient amount of votes to be removed. If an address attempts to transfer BELA out with 0 votes, the following line in `_moveDelegates()` will revert with an arithmetic underflow :

```solidity
                uint256 srcRepNew = srcRepOld - amount;
```

If an attacker reduces a contract's votes to 0 using the exploit demonstrated above, all the BELA in the contract can no longer be transferred out due to an insufficient amount of votes.

For instance, if an attacker does the attack on the [VeBela contract](https://arbiscan.io/address/0x7fbdEb84D5966c1C325D8CB2E01593D74c9A41Cd?utm_source=immunefi#code), which has over `8884088` BELA deposited, all depositors will be unable to withdraw their BELA from the contract forever.

## Impact

By manipulating vote accounting in the BELA token contract, an attacker can reduce the votes of **any address** to 0, causing all future transfers of BELA from that address to revert. 

This can be used to permanently prevent users who have deposited BELA into the VeBela contract from being able to withdraw any BELA, resulting in the permanent freezing of funds. 

## Recommendation

In the BELA contract, `_delegate()` should transfer votes from the `sender` and `recipient`'s delegatees instead:

```diff
    function _transfer(address sender, address recipient, uint256 amount) internal override {
        ERC20._transfer(sender, recipient, amount);
        // console2.log(sender, recipient, amount);
-       _moveDelegates(sender, recipient, amount);
+       _moveDelegates(_delegates[sender], _delegates[recipient], amount);
    }
```

## Proof of Concept

The attached gist contains a Foundry test that demonstrates how an attacker can reduce the votes of the `VeBela` contract to 0, thereby freezing all deposited BELA permanently:

https://gist.github.com/MiloTruck/2b89f8f762f1ed23eb860f96e2341fd7

To run the test:

1. Create a Foundry project:

```sh
forge init beluga
cd beluga
```

2. Copy the test into the `test/` folder.
3. Run the test with your Arbitrum RPC URL:

```sh
forge test --fork-url <ARBITRUM_RPC_URL>
```