## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [H-01](#h-01-missing-msgsender-check-in-insigniabidmint) | Missing `msg.sender` check in `Insignia.bidMint()` | High | 
| [L-01](#l-01-amount-parameter-in-mint-and-bidmint-is-incorrect) | `amount` parameter in `mint()` and `bidMint()` is incorrect | Low |
| [L-02](#l-02-missing-validation-for-the-_insigniacontract-address-in-bidderplacebid) | Missing validation for the `_insigniaContract` address in `Bidder.placeBid()` | Low |
| [L-03](#l-03-missing-check-for-success-when-sending-eth-with-low-level-call) | Missing check for `success` when sending ETH with low-level call | Low |
| [L-04](#l-04-checks-effects-interactions-violation-in-mint-and-bidmint) | Checks-Effects-Interactions violation in `mint()` and `bidMint()` | Low |
| [I-01](#i-01-making-the-owner-pay-eth-when-calling-insigniamint-is-redundant) | Making the owner pay ETH when calling `Insignia.mint()` is redundant | Informational |
| [I-02](#i-02-missing-nonreentrant-modifier-in-insigniaburn-and-insigniawithdraw) | Missing `nonReentrant` modifier in `Insignia.burn()` and `Insignia.withdraw()` | Informational |
| [I-03](#i-03-minor-improvements) | Minor improvements | Informational |


## [H-01] Missing `msg.sender` check in `Insignia.bidMint()`

### Description

`Insignia.bidMint()` is meant to be called by the `Bidder` contract when the owner accepts a bid. However, `bidMint()` does not check that `msg.sender` is the `Bidder` contract:

[Insignia.sol#L104-L115](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L104-L115)

```solidity
function bidMint(address to, uint256 amount) external payable nonReentrant {
    uint256 tokenSupply = paidSupply + freeSupply;
    
    require(tokenSupply + amount <= maxSupply, "Exceeds Max supply");
    
    uint256 tokenId = ++tokenSupply;
    _setDataForTokenId( bytes32(tokenId), _LSP4_METADATA_KEY, lsp4BidMetadataUri);
    _mint(to, bytes32(tokenId), true, "");
    paidSupply = ++paidSupply;

    emit InsigniaPaidMinted(to, bytes32(tokenId));
}
```

Therefore, an attacker can directly call `bidMint()` to mint unlimited NFTs to themselves at no cost.

### Recommendation

Store the address of the `Bidder` contract in `InsigniaFactory` and pass it to the `Insignia` contract when `initialize()` is called.

In `Insignia.bidMint()`, add the following check:

```diff
  function bidMint(address to, uint256 amount) external payable nonReentrant {
      uint256 tokenSupply = paidSupply + freeSupply;
+     require(msg.sender == bidder, "Not bidder");  
```

### Mitigation

Fixed in commit [cf94322](https://github.com/bozp-pzob/contract-inkso-public/commit/cf94322f3aa7cab3eaf488328b4b8b0fdffc8749), a check has been added in `bidMint()` to ensure that `msg.sender` is the `Bidder` contract.

## [L-01] `amount` parameter in `mint()` and `bidMint()` is incorrect

### Description

In the `Insignia` contract, `mint()` and `bidMint()` have an `amount` parameter that specifies the amount of `Insignia` tokens to be minted:

[Insignia.sol#L90-L94](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L90-L94)

```solidity
function mint(address to, uint256 amount) external payable nonReentrant onlyOwner {
    uint256 tokenSupply = paidSupply + freeSupply;

    if (msg.value != pricePerToken * amount) revert InsufficientBalance();
    require(tokenSupply + amount <= maxSupply, "Exceeds Max supply");
```

[Insignia.sol#L104-L107](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L104-L107)

```solidity
function bidMint(address to, uint256 amount) external payable nonReentrant {
    uint256 tokenSupply = paidSupply + freeSupply;
    
    require(tokenSupply + amount <= maxSupply, "Exceeds Max supply");
```

As seen from above, both functions check that `tokenSupply + amount` does not exceed the `maxSupply` of tokens.

However, since `Insignia` is an LSP8 token, whenever `mint()` or `bidMint()` is called, only one token will be minted. This means that `amount` should always be `1` and any other value would be incorrect.

As such, if `mint()` is called with `amount > 1`, the owner will have to overpay to mint only one token. Additionally, `mint()` and `bidMint()` could incorrectly revert due to the `maxSupply` check.

### Recommendation

Remove the `amount` parameter from both `mint()` and `bidMint()`:

```diff
- function mint(address to, uint256 amount) external payable nonReentrant onlyOwner {
+ function mint(address to) external payable nonReentrant onlyOwner {
      uint256 tokenSupply = paidSupply + freeSupply;
  
-     if (msg.value != pricePerToken * amount) revert InsufficientBalance();
-     require(tokenSupply + amount <= maxSupply, "Exceeds Max supply");
+     if (msg.value != pricePerToken) revert InsufficientBalance();
+     require(tokenSupply + 1 <= maxSupply, "Exceeds Max supply");
```

```diff
- function bidMint(address to, uint256 amount) external payable nonReentrant {
+ function bidMint(address to) external payable nonReentrant {
      uint256 tokenSupply = paidSupply + freeSupply;
      
-     require(tokenSupply + amount <= maxSupply, "Exceeds Max supply");
+     require(tokenSupply + 1 <= maxSupply, "Exceeds Max supply");
```

### Mitigation

Fixed in commit [cf94322](https://github.com/bozp-pzob/contract-inkso-public/commit/cf94322f3aa7cab3eaf488328b4b8b0fdffc8749), the `amount` parameter has been removed from both functions.

## [L-02] Missing validation for the `_insigniaContract` address in `Bidder.placeBid()`

### Description

The `Bidder.placeBid()` function does not check that the `_insigniaContract` address passed by the caller is an `Insignia` contract deployed by `InsigniaFactory`:

[Bidder.sol#L39-L52](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Bidder.sol#L39-L52)

```solidity
function placeBid(address _insigniaContract) public payable nonReentrant {
    require(msg.value >= 0, "Bid amount must be greater than or equal to 0.");
    require(_insigniaContract != address(0), "Invalid NFT contract address.");

    bids[bidCounter] = Bid({
        bidder: msg.sender,
        amount: msg.value,
        accepted: false,
        insigniaContract: _insigniaContract
    });

    emit NewBid(bidCounter, msg.sender, msg.value, _insigniaContract);
    bidCounter++;
}
```

Therefore, an attacker could call `placeBid()` and pass a malicious contract as the `_insigniaContract` address. This could have unexpected behavior when `acceptBid()` or `withdraw()` is called.

### Recommendation

Ensure that the `_insigniaContract` address is a valid `Insignia` contract deployed from the factory. One possible implementation would be to specify the index of the `Insignia` contract in the `deployedTokens` array, instead of its address:

```diff
- function placeBid(address _insigniaContract) public payable nonReentrant {
+ function placeBid(uint256 _insigniaContractIndex) public payable nonReentrant {
+     address _insigniaContract = insigniaFactory.deployedTokens(_insigniaContractIndex);
+
      require(msg.value >= 0, "Bid amount must be greater than or equal to 0.");
      require(_insigniaContract != address(0), "Invalid NFT contract address.");
```

### Mitigation

Fixed in commit [cf94322](https://github.com/bozp-pzob/contract-inkso-public/commit/cf94322f3aa7cab3eaf488328b4b8b0fdffc8749) as recommended.

## [L-03] Missing check for `success` when sending ETH with low-level call

### Description

In the instances listed below, a low-level `.call()` is used to transfer ETH, but the `success`/`owner_success` boolean is not checked to be `true`:

[Bidder.sol#L39-L52](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Bidder.sol#L39-L52)

```solidity
(bool success, ) = inksoAddress.call{value: amount5}(
    bytes.concat(bytes4(0), bytes(unicode"sending percentage"))
);

(bool owner_success, ) = IInsignia(bid.insigniaContract).owner().call{value: amount95}(
    bytes.concat(bytes4(0), bytes(unicode"sending bid to owner"))
);
```

[InsigniaFactory.sol#L99-L101](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/InsigniaFactory.sol#L99-L101)

```solidity
(bool success, ) = msg.sender.call{value: amount}(
    bytes.concat(bytes4(0), bytes(unicode"withdrawing"))
);
```

[Insignia.sol#L120-L122](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L120-L122)

```solidity
(bool success, ) = msg.sender.call{value: amount}(
    bytes.concat(bytes4(0), bytes(unicode"withdrawing"))
);
```

As such, even if the ETH transfer failed, the functions called will not revert.

For example, in `acceptBid()`, if the ETH transfer to the owner of an `Insignia` contract failed, `acceptBid()` would still succeed. As a result, ETH for that bid will not be sent to the owner and will instead be stuck in the `Bidder` contract.

### Recommendation

In all instances listed above, check that `success`/`owner_success` is `true` as such:

```solidity
(bool success, ) = inksoAddress.call{value: amount5}(
    bytes.concat(bytes4(0), bytes(unicode"sending percentage"))
);
require(success, "ETH transfer failed");
```

### Mitigation

Fixed in commit [cf94322](https://github.com/bozp-pzob/contract-inkso-public/commit/cf94322f3aa7cab3eaf488328b4b8b0fdffc8749) as recommended.

## [L-04] Checks-Effects-Interactions violation in `mint()` and `bidMint()` 

### Description

In the `Insignia` contract, `mint()` first calls `_mint()` to mint a token to the user, followed by an increment to `freeSupply`:

[Insignia.sol#L98-L99](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L98-L99)

```solidity
_mint(to, bytes32(tokenId), true, "");
freeSupply = ++freeSupply;
```

Similarly, in `bidMint()`, `_mint()` is called before incrementing `paidSupply`:

[Insignia.sol#L111-L112](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L111-L112)

```solidity
_mint(to, bytes32(tokenId), true, "");
paidSupply = ++paidSupply;
```

However, such an implementation violates the [Checks-Effects-Interactions](https://fravoll.github.io/solidity-patterns/checks_effects_interactions.html) pattern - the LSP1 callback in `_mint()` to the user is performed before `freeSupply`/`paidSupply` is updated:

[LSP8IdentifiableDigitalAssetCore.sol#L547](https://github.com/lukso-network/lsp-smart-contracts/blob/develop/packages/lsp8-contracts/contracts/LSP8IdentifiableDigitalAssetCore.sol#L547)

```solidity
_notifyTokenReceiver(to, force, lsp1Data);
```

This exposes the contract to reentrancy risk - in the LSP1 callback from `_mint()`, a malicious user could perform other calls before `freeSupply`/`paidSupply` is updated.

### Recommendation

In both instances, increment `freeSupply`/`paidSupply` before calling `_mint()`:

[Insignia.sol#L98-L99](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L98-L99)

```diff
+ freeSupply = ++freeSupply;
  _mint(to, bytes32(tokenId), true, "");
- freeSupply = ++freeSupply;
```

[Insignia.sol#L111-L112](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L111-L112)

```diff
+ paidSupply = ++paidSupply;
  _mint(to, bytes32(tokenId), true, "");
- paidSupply = ++paidSupply;
```

This ensures that state variables are updated _before_ any external call occurs.

### Mitigation

Fixed in commit [cf94322](https://github.com/bozp-pzob/contract-inkso-public/commit/cf94322f3aa7cab3eaf488328b4b8b0fdffc8749) as recommended.

## [I-01] Making the owner pay ETH when calling `Insignia.mint()` is redundant

### Description

When the owner calls `Insignia.mint()` to mint tokens to users, he has to pay `pricePerToken` in ETH for each token minted:

[Insignia.sol#L90-L93](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L90-L93)

```solidity
function mint(address to, uint256 amount) external payable nonReentrant onlyOwner {
    uint256 tokenSupply = paidSupply + freeSupply;

    if (msg.value != pricePerToken * amount) revert InsufficientBalance();
```

The ETH paid remains in the `Insignia` contract.

However, this is redundant as the owner can call `withdraw()` to withdraw all ETH from the contract afterwards:

[Insignia.sol#L117-L123](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L117-L123)

```solidity
function withdraw(uint256 amount) external onlyOwner {
    if (amount > address(this).balance) revert InsufficientBalance();

    (bool success, ) = msg.sender.call{value: amount}(
        bytes.concat(bytes4(0), bytes(unicode"withdrawing"))
    );
}

```

As such, the `Insignia` owner should not need to pay ETH to mint tokens to user, since he is effectively paying himself.

### Recommendation

Make `mint()` non-payable and remove the `msg.value` check:

[Insignia.sol#L90-L93](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L90-L93)

```diff
- function mint(address to, uint256 amount) external payable nonReentrant onlyOwner {
+ function mint(address to, uint256 amount) external nonReentrant onlyOwner {
      uint256 tokenSupply = paidSupply + freeSupply;
  
-     if (msg.value != pricePerToken * amount) revert InsufficientBalance();
```

This allows the owner to call `mint()` without paying any ETH.

Additionally, the `pricePerToken` state variable can also be removed as it is not used anywhere else in the contract.

### Mitigation

Fixed in commit [cf94322](https://github.com/bozp-pzob/contract-inkso-public/commit/cf94322f3aa7cab3eaf488328b4b8b0fdffc8749), the ETH sent when calling `mint()` is now sent to the `InsigniaFactory` contract as fees to the protocol owner.

## [I-02] Missing `nonReentrant` modifier in `Insignia.burn()` and `Insignia.withdraw()`

### Description

Both `Insignia.withdraw()` and `Insignia.burn()` do not have the `nonReentrant` modifier:

[Insignia.sol#L117](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L117)

```solidity
function withdraw(uint256 amount) external onlyOwner {
```

[Insignia.sol#L125](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L125)

```solidity
function burn(bytes32 tokenId) external {
```

As such, both functions can be called in a reentrant call.

### Recommendation

Add the `nonReentrant` modifier to both functions:

```diff
- function withdraw(uint256 amount) external onlyOwner {
+ function withdraw(uint256 amount) external nonReentrant onlyOwner {
```

```diff
- function burn(bytes32 tokenId) external {
+ function burn(bytes32 tokenId) external nonReentrant {
```

### Mitigation

Fixed in commit [cf94322](https://github.com/bozp-pzob/contract-inkso-public/commit/cf94322f3aa7cab3eaf488328b4b8b0fdffc8749) as recommended.

## [I-03] Minor improvements

1. [Bidder.sol#L22](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Bidder.sol#L22)  - `inksoAddress` can be a constant:

```diff
- address public inksoAddress = 0x39b14cAAb195bfD15Ce91bdB483966249159c07D;
+ address public constant inksoAddress = 0x39b14cAAb195bfD15Ce91bdB483966249159c07D;
```

2. [Bidder.sol#L35-L37](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Bidder.sol#L35-L37) - This constructor is redundant and can be removed as `bidCounter` is set to `0` by default.

3. [Insignia.sol#L19-L23](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L19-L23) - The following assignments are redundant as the `Insignia` contract is cloned with OpenZeppelin's `Clones.clone()`, so the value of state variables are not carried over. Consider removing these assignments:

```diff
- uint256 public maxSupply = 0;
- uint256 public freeSupply = 0;
- uint256 public paidSupply = 0;
- uint256 public burnSupply = 0;
- uint256 public pricePerToken = 0.01 ether;
+ uint256 public maxSupply;
+ uint256 public freeSupply;
+ uint256 public paidSupply;
+ uint256 public burnSupply;
+ uint256 public pricePerToken;
```

4. [Insignia.sol#L99](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L99), [Insignia.sol#L112](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L112), [Insignia.sol#L130](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/Insignia.sol#L130) - The following state variable increments can be simplified as such:

```diff
- freeSupply = ++freeSupply;
+ ++freeSupply;
```

```diff
- paidSupply = ++paidSupply;
+ ++paidSupply;
```

```diff
- burnSupply = ++burnSupply;
+ ++burnSupply;
```

5. [InsigniaFactory.sol#L35](https://github.com/bozp-pzob/contract-inkso-public/blob/2ff0face82aa0568094638ef41886fe3897e4054/contracts/InsigniaFactory.sol#L35) - This empty constructor can be removed.

### Mitigation

Fixed in commit [cf94322](https://github.com/bozp-pzob/contract-inkso-public/commit/cf94322f3aa7cab3eaf488328b4b8b0fdffc8749).