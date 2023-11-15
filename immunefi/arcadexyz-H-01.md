# Users using EIP-1271 for signatures can be forced into loans as the wrong party without consent

## Details

### Protocol

[Arcade.xyz](https://immunefi.com/bounty/arcade/)

### Target

https://etherscan.io/address/0xb7bfcca7d7ff0f371867b770856fac184b185878

### Severity

High

### Bug Description

In `OriginationController.sol`, whenever a user is creating a loan, they have to provide the signature of the opposite party. Using `initializeLoan()` as an example:

[OriginationController.sol#L210-L221](https://github.com/arcadexyz/arcade-protocol/blob/main/contracts/OriginationController.sol#L210-L221)

```solidity
        // Determine if signature needs to be on the borrow or lend side
        Side neededSide = isSelfOrApproved(borrower, msg.sender) ? Side.LEND : Side.BORROW;

        (bytes32 sighash, address externalSigner) = recoverItemsSignature(
            loanTerms,
            sig,
            nonce,
            neededSide,
            encodedPredicates
        );

        _validateCounterparties(borrower, lender, msg.sender, externalSigner, sig, sighash, neededSide);
```

For instance, if `msg.sender` is the `borrower`, he would have to provide the signature of the `lender` as `neededSide` would be `Side.LEND`.

The `msg.sender`, `lender` and `borrower` are then validated in `_validateCounterparties()`:

[OriginationController.sol#L765-L782](https://github.com/arcadexyz/arcade-protocol/blob/main/contracts/OriginationController.sol#L765-L782)

```solidity
        address signingCounterparty = neededSide == Side.LEND ? lender : borrower;
        address callingCounterparty = neededSide == Side.LEND ? borrower : lender;

        // Make sure the signer recovered from the loan terms is not the caller,
        // and even if the caller is approved, the caller is not the signing counterparty
        if (caller == signer || caller == signingCounterparty) revert OC_ApprovedOwnLoan(caller);

        // Check that caller can actually call this function - neededSide assignment
        // defaults to BORROW if the signature is not approved by the borrower, but it could
        // also not be a participant
        if (!isSelfOrApproved(callingCounterparty, caller) && !isApprovedForContract(callingCounterparty, sig, sighash)) {
            revert OC_CallerNotParticipant(msg.sender);
        }

        // Check signature validity
        if (!isSelfOrApproved(signingCounterparty, signer) && !isApprovedForContract(signingCounterparty, sig, sighash)) {
            revert OC_InvalidSignature(signingCounterparty, signer);
        }
```

Where:
- `sig` is the signature provided by the caller.
- `sighash` is the signature digest, generated from hashing the loan's terms.

As seen from above, the `isApprovedForContract()` function is used to check if the signature belongs to both the `callingCounterparty` and `signingCounterparty`. However, this is incorrect as the signature should belong to only the `signingCounterparty`, and **not** the `callingCounterparty`. 

An attacker can abuse this to force a user with a `Side.BORROW` signature to become a lender:

- Assume Bob uses a smart contract wallet:
  - Bob's wallet has an existing approval of 1000 USDC to the `LoanCore` contract.
- Bob wants to borrow a loan using his Azuki NFT as collateral. He generates a signature with the following:
  - `side = Side.BORROW` 
  - `loanTerms.collateralAddress` is the Azuki address
  - `loanTerms.collateralId = 1337`
  - `loanTerms.payableCurrency` is the USDC address
  - `loanTerms.principal = 1000e6`. Note that this amount can be anything smaller than Bob's approval.
- Alice wishes to make Bob become a lender for the loan, she does the following:
  - Deploy a malicious contract that will be the `borrower` address. Whenever `isValidSignature()` is called, its selector is always returned.
  - She buys Bob's Azuki NFT from him and transfers it to the malicious contract.
  - Before Bob can invalidate his signature using `cancelNonce()` in `LoanCore`, she calls `initializeLoan()` with the following arguments:
    - `loanTerms` is set to the terms in Bob's signature.
    - `borrower` is Alice's malicious contract.
    - `lender` is Bob's wallet.
    - `sig` is set to Bob's signature.
    - `nonce` is set to the nonce in Bob's signature.
  - Note that the caller (`msg.sender`) is Alice's address.
- In `initializeLoan()`:
  - `neededSide = Side.BORROW`, as Alice's malicious contract is not approved by her address.
  - In `_validateCounterparties()`:
    - `signingCounterparty` is Alice's malicious contract.
    - `callingCounterparty` is Bob's wallet.
    - The `isApprovedForContract()` check for `callingCounterparty` passes, as Bob is the signer of the BORROW signature.
    - The `isApprovedForContract()` check for `signingCounterparty` passes, as Alice's malicious contract returns the correct magic value for any signature.
- As a result, a new loan is created with Bob's wallet as the lender.

In the scenario above, Bob's signature was used to force him to become a lender, even though his signature contained `Side.BORROW`.


Note that the likelihood of a user having an existing approval to the `LoanCore` contract is not low; the user could be waiting for another loan to be created, where he is the lender that provides currency.

The exploit shown above can be used to manipulate signatures using any function that calls `_validateCounterparties()`, including:
- `initializeLoan()`
- `initializeLoanWithItems()`
- `rolloverLoan()`
- `rolloverLoanWithItems()`

### Impact

Using a victim's signature, an attacker can force the victim to enter a loan as the wrong party. As demonstrated above, the victim was forced to be a lender against his will, even though he only consented to being a borrower via signature. 

### Recommendation

In `_validateCounterparties()`, consider removing the `isApprovedForContract()` check for `callingCounterparty`:

[OriginationController.sol#L775-L777](https://github.com/arcadexyz/arcade-protocol/blob/main/contracts/OriginationController.sol#L775-L777)

```diff
-       if (!isSelfOrApproved(callingCounterparty, caller) && !isApprovedForContract(callingCounterparty, sig, sighash)) {
+       if (!isSelfOrApproved(callingCounterparty, caller)) {
            revert OC_CallerNotParticipant(msg.sender);
        }
```

This ensures that the caller must be either the borrower or the lender.

### Proof of Concept

The following gist contains a Foundry test that demonstrates how an attacker can manipulate a user's signature meant for a BORROW and force him to become a lender:

https://gist.github.com/MiloTruck/53a5b8bf90240280a237f827dec1a074#file-h-01-t-sol

To run the test:
1. Create a Foundry Project:
```sh
forge init arcadexyz
cd arcadexyz
```

2. Copy the test into the `test/` folder.
3. Copy solmate's [`ERC721.sol`](https://github.com/transmissions11/solmate/blob/main/src/tokens/ERC721.sol) into the `src/` folder.
4. Run the test with your Mainnet RPC URL:

```sh
forge test -vvv --fork-url <MAINNET_RPC_URL> --block-number 18363316
```