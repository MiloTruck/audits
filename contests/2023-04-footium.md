# Footium

The code under view can be found in [2023-04-footium](https://github.com/sherlock-audit/2023-04-footium/tree/main).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [H-01](#h-01-a-previous-club-owner-can-abuse-approvals-to-steal-assets-from-the-clubs-escrow) | A previous club owner can abuse approvals to steal assets from the club's escrow | High |
| [M-01](#m-01-users-might-lose-funds-as-claimerc20prize-doesnt-revert-for-no-revert-on-transfer-tokens) | Users might lose funds as `claimERC20Prize()` doesn't revert for no-revert-on-transfer tokens | Medium |
| [L-01](#l-01-footiumescrow-has-no-function-for-individual-nft-approvals) | `FootiumEscrow` has no function for individual NFT approvals | Low |

## [H-01] A previous club owner can abuse approvals to steal assets from the club's escrow

### Summary

As approvals are not cleared when `FootiumClub` NFTs are transferred, a previous owner can possibly steal assets from the club's escrow after selling the club NFT.

### Vulnerability Detail

In the protocol, each club is represented by a `FootiumClub` NFT, which is a regular ERC721 token. 

In `FootiumEscrow.sol`, the owner of the club NFT has ownership of all assets held in the club's escrow, as seen by the `onlyClubOwner()` modifier:

```solidity
modifier onlyClubOwner() {
        if (msg.sender != IERC721(footiumClubAddress).ownerOf(clubId)) {
            revert NotClubOwner(clubId, msg.sender);
    }
    _;
}
```

From conversing with the sponsor, it is confirmed that club NFTs are meant to be traded:
> From **Jordan Lord#1581**:  
> 
> _2. Yes club NFTs can be traded as well_

The club's owner can approve ERC20 and ERC721 token transfers from the `FootiumEscrow` contract using [`setApprovalForERC20()`](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumEscrow.sol#L75C14-L81) and [`setApprovalForERC721()`](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumEscrow.sol#L90C14-L96) respectively. 

Therefore, if the club NFT is ever sold, the club's owner can steal assets from the NFT buyer by doing the following:
1. Approve the transfer of all assets in the club's escrow to himself using `setApprovalForERC20()` and `setApprovalForERC721()`.
2. Sell the club - transfer the club NFT to the buyer.
3. Transfer all assets out of the escrow using the approvals.

### Impact

A previous owner can steal all assets in the escrow from the current club owner.

### Recommendation

Consider revoking all existing approvals from the club's escrow whenever a club NFT is transferred. 

Alternatively, create a new escrow for the club NFT whenever it is transferred, and transfer all assets from the old escrow to the new one.

## [M-01] Users might lose funds as `claimERC20Prize()` doesn't revert for no-revert-on-transfer tokens

### Summary

Users can call `claimERC20Prize()` without actually receiving tokens if a no-revert-on-failure token is used, causing a portion of their claimable tokens to become unclaimable.

### Vulnerability Detail

In the `FootiumPrizeDistributor` contract, whitelisted users can call `claimERC20Prize()` to claim ERC20 tokens. The function adds the amount of tokens claimed to the user's total claim amount, and then transfers the tokens to the user:

[FootiumPrizeDistributor.sol#L128-L131](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumPrizeDistributor.sol#L128-L131)

```solidity
if (value > 0) {
        totalERC20Claimed[_token][_to] += value;
    _token.transfer(_to, value);
}
```

As the the return value from `transfer()` is not checked, `claimERC20Prize()` does not revert even when the transfer of tokens to the user fails.

This could potentially cause users to lose assets when:
1. `_token` is a no-revert-on-failure token.
2. The user calls `claimERC20Prize()` with `value` higher than the contract's token balance.

As the contract has an insufficient balance, `transfer()` will revert and the user receives no tokens. However, as `claimERC20Prize()` succeeds, `totalERC20Claimed` is permanently increased for the user, thus the user cannot claim these tokens again.

### Impact

Users can call `claimERC20Prize()` without receiving the token amount specified. These tokens become permanently unclaimable for the user, leading to a loss of funds.

### Recommendation

Use `safeTransfer()` from Openzeppelin's [SafeERC20](https://docs.openzeppelin.com/contracts/2.x/api/token/erc20#SafeERC20) to transfer ERC20 tokens. Note that [`transferERC20()`](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumEscrow.sol#L105-L111) in `FootiumEscrow.sol` also uses `transfer()` and is susceptible to the same vulnerability.

## [L-01] `FootiumEscrow` has no function for individual NFT approvals

### Summary

The `FootiumEscrow` contract has a function to approve all NFTs for an address, but not for individual NFTs. This could lead to NFTs being stolen when interacting with external exchanges.

### Vulnerability Detail

The [technical documentation](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/technical-docs/General.md#escrow-contracts) states that `FootiumEscrow` is meant to interact with NFT marketplaces to trade NFTs:
> Ultimately, this allows our escrows to interact use NFT marketplaces like the Rarible Protocol

On most NFT marketplaces, the seller has to to approve NFT to sell before placing a sell order. This allows the marketplace contract to transfer the NFT to the buyer. 

As `FootiumEscrow` does not have a function for individual NFT approvals, this can only be done through the [`setApprovalForERC721()`](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumEscrow.sol#L90-L96) function, which uses `setApprovalForAll()`.

However, such an implementation is dangerous as it allows the approved address to transfer **all** NFTs in the escrow. If the NFT marketplace contract ever becomes malicious or hacked, an attacker could transfer all NFTs in the escrow contract to himself using the approval.

### Impact

Users might lose all their NFTs if they sell NFTs from their club's escrow on external exchanges.

### Recommendation

Consider adding a function for individual NFT approvals:

```solidity
function approveERC721(
        IERC721 erc721Contract,
    address to,
    uint256 tokenId
) external onlyClubOwner {
        erc721Contract.approve(to, tokenId);
}
```
