# Nouns Builder

The code under review can be found in [2022-09-nouns-builder](https://github.com/code-423n4/2022-09-nouns-builder).

## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](https://github.com/MiloTruck/audits/blob/main/code4rena/2022-09-nouns-builder.md#m-01-possible-for-funds-to-get-stuck-in-the-auction-contract) | Possible for funds to get stuck in the Auction contract | Medium |
| [M-02](https://github.com/MiloTruck/audits/blob/main/code4rena/2022-09-nouns-builder.md#m-02-founder-percentage-larger-than-255-will-break-minting-of-tokens) | Founder percentage larger than 255 will break minting of tokens | Medium |
| [M-03](https://github.com/MiloTruck/audits/blob/main/code4rena/2022-09-nouns-builder.md#m-03-founders-might-receive-less-tokens-than-their-specified-percentage-of-token-ownership) | Founders might receive less tokens than their specified percentage of token ownership | Medium |

# [M-01] Possible for funds to get stuck in the Auction contract

## Impact
For projects with a 0% founder percentage ownership, it is possible for the highest bidder of the first auction to not receive the minted token, and funds from the first auction to be stuck in the contract.

## Vulnerability Details

In `Auction.sol`, the contract determines if the first auction has been created using the statement `auction.tokenId == 0`, as shown below:
```solidity
244:    function unpause() external onlyOwner {
245:        _unpause();
246:
247:        // If this is the first auction:
248:        if (auction.tokenId == 0) {
249:            // Transfer ownership of the contract to the DAO
250:            transferOwnership(settings.treasury);
251:
252:            // Start the first auction
253:            _createAuction();
254:        }
255:        // Else if the contract was paused and the previous auction was settled:
256:        else if (auction.settled) {
257:            // Start the next auction
258:            _createAuction();
259:        }
260:    }
```

If the check at line 248 passes, the `unpause()` function will directly call `_createAuction()` without checking that the current auction has been settled.

Thus, in a situation where the `tokenId` of the first auction is `0`, the owner could potentially call `unpause()` again without settling the first auction, causing the current auction to be overwritten due to the call to `_createAuction()`.

This would cause the following:
* Funds deposited by the highest bidder from the first auction would become stuck in the contract
* The highest bidder for the first auction will never receive the minted token

For example:
* A project is deployed with a total percentage ownership of 0% from founders
* The founder calls `unpause()` to start the first auction
* At line 250, ownership is transferred to the treasury
* The first auction is started via `_createAuction()` at line 253
* As founders have no ownership percentage of the minted tokens, the first token minted (`tokenId = 0`), will be used as the token for the first auction
* The DAO pauses the auction for some reason via `pause()`
* If the DAO calls `unpause()` without first calling `settleAuction()`
* A new auction with `tokenId = 1` would be created, overwriting the first auction

In this scenario, funds from the first auction would be unretrievable as the `Auction` contract does not have a function to send any amount of ETH to an address. Furthermore, as `settleAuction()` was not called, the highest bidder would not receive the token of the first auction.

## Proof of Concept

The test in [this gist](https://gist.github.com/MiloTruck/f0329c57864951cda55ffff1bda380fc) demonstrates the above.

## Recommended Mitigation Steps
At line 248, instead of checking `auction.tokenId == 0` to determine if the first auction has been created, check if `auction.startTime == 0` instead.

# [M-02] Founder percentage larger than 255 will break minting of tokens

## Impact
If the `Token` contract is deployed with a founder's ownership percentage larger than 255, the minting of tokens will break.

## Vulnerability Details
In `Token.sol:80-120`, the function `_addFounders()` handles the addition of each founder as shown below:
```solidity
 80:    for (uint256 i; i < numFounders; ++i) {
 81:        // Cache the percent ownership
 82:        uint256 founderPct = _founders[i].ownershipPct;
 83:
 84:        // Continue if no ownership is specified
 85:        if (founderPct == 0) continue;
 86:
 87:        // Update the total ownership and ensure it's valid
 88:        if ((totalOwnership += uint8(founderPct)) > 100) revert INVALID_FOUNDER_OWNERSHIP();
 89:                                                                                                 
 90:        // Compute the founder's id
 91:        uint256 founderId = settings.numFounders++;
 92:
 93:        // Get the pointer to store the founder
 94:        Founder storage newFounder = founder[founderId];
 95:
 96:        // Store the founder's vesting details
 97:        newFounder.wallet = _founders[i].wallet;
 98:        newFounder.vestExpiry = uint32(_founders[i].vestExpiry);
 99:        newFounder.ownershipPct = uint8(founderPct);
100:
101:        // Compute the vesting schedule
102:        uint256 schedule = 100 / founderPct;
103:
104:        // Used to store the base token id the founder will recieve
105:        uint256 baseTokenId;
106:
107:        // For each token to vest:
108:        for (uint256 j; j < founderPct; ++j) {
109:            // Get the available token id
110:            baseTokenId = _getNextTokenId(baseTokenId);
111:
112:            // Store the founder as the recipient
113:            tokenRecipient[baseTokenId] = newFounder;
114:
115:            emit MintScheduled(baseTokenId, founderId, newFounder);
116:
117:            // Update the base token id
118:            (baseTokenId += schedule) % 100;
119:        }
120:    }
```

At lines 88 and 99, `founderPct` is downcasted to `uint8` before its value is checked or stored. However, at lines 102 and 108, the `founderPct` is used directly to handle the logic of allocating tokens to founders.

As seen from [here](https://ethereum.stackexchange.com/questions/100029/how-is-uint8-calculated-from-a-uint256-conversion-in-solidity), downcasting to `uint8` takes the last 2 characters of the `uint256` hexadecimal:
```solidity
uint256 a = 0x100;
uint8 b = uint8(a); // b equals to 0x00
```

Thus, if a founder is provided with `ownershipPct` set to larger than `0xff`, such as `0x100`, the following occurs:
* At line 88, the check passes:
  * `uint8(founderPct) = uint8(0x100) = 0`, which is less than 100
  * `totalOwnership` is set to `0`
* At line 99, `newFounder.ownershipPct` is also set to `0`
* At line 102, `schedule` is set to `0` as `100 / founderPct = 100 / 0x100`, which rounds down to `0`
* At line 108, the `for` statement loops `0x100` instead of `0` times
  * Since `schedule` is `0`, `tokenRecipient[]` from `0` to `255` is set to the founder

In `Token.sol:152-157`, the `mint()` function uses a `do-while` loop to find the next `tokenId` not allocated to a founder:
```solidity
152:    do {
153:        // Get the next token to mint
154:        tokenId = settings.totalSupply++;
155:
156:        // Lookup whether the token is for a founder, and mint accordingly if so
157:    } while (_isForFounder(tokenId));
```

Since `_isForFounder()` will always return true for any `tokenId`, the `do-while` loop will continue on forever, thus breaking the minting of tokens. However, this is not obvious to users as calling `totalFounderOwnership()` would return `0`.

Additionally, note that setting `ownershipPct` to `100` will also break minting, but `totalFounderOwnership()` would return `100` as downcasting to `uint8` would not change the value.

## Proof of Concept

This [test](https://gist.github.com/MiloTruck/7f9368bc1dd274ba278e1c6113e70a28) demonstrates the above.

## Recommended Mitigation Steps
Store the value of `uint8(founderPct)` in a local variable, and use that variable throughout the entire `_addFounders()` function. Alternatively, use `SafeCast.toUint8()` instead of `uint8()`.

# [M-03] Founders might receive less tokens than their specified percentage of token ownership

## Impact
Due to how the allocation of tokens to founders is handled in `Token.sol`, projects with a certain sequence of founder percentage ownership may cause founders to receive less tokens than intended.

## Vulnerability Details
In the `Token` contract, before any token is minted, its `_tokenId` is passed into `_isForFounder()` to determine if the token belongs to a founder, as shown below:
```solidity
177:    function _isForFounder(uint256 _tokenId) private returns (bool) {
178:        // Get the base token id
179:        uint256 baseTokenId = _tokenId % 100;
180:
181:        // If there is no scheduled recipient:
182:        if (tokenRecipient[baseTokenId].wallet == address(0)) {
183:            return false;
184:
185:            // Else if the founder is still vesting:
186:        } else if (block.timestamp < tokenRecipient[baseTokenId].vestExpiry) {
187:            // Mint the token to the founder
188:            _mint(tokenRecipient[baseTokenId].wallet, _tokenId);
189:
190:            return true;
191:
192:            // Else the founder has finished vesting:
193:        } else {
194:            // Remove them from future lookups
195:            delete tokenRecipient[baseTokenId];
196:
197:            return false;
198:        }
199:    }
```

At line 179, `baseTokenId` is derived by using `% 100`. `baseTokenId` is then used to check if `tokenRecipient[baseTokenId]` corresponds to a founder, and if so, the token is minted directly to the founder's wallet.

Since `baseTokenId` will always be smaller than `100` in `_isForFounder()`, the valid index range for `tokenRecipient[]` is from `0` to `99` inclusive. This means that assigning a founder to anything above `99`, such as `tokenRecipient[100]`, would not give tokens to the founder.

The logic that assigns founders to `tokenRecipient[]` based on their percentage ownership is in `_addFounders()`:
```solidity
101:    // Compute the vesting schedule
102:    uint256 schedule = 100 / founderPct;
103:
104:    // Used to store the base token id the founder will recieve
105:    uint256 baseTokenId;
106:
107:    // For each token to vest:
108:    for (uint256 j; j < founderPct; ++j) {
109:        // Get the available token id
110:        baseTokenId = _getNextTokenId(baseTokenId);
111:
112:        // Store the founder as the recipient
113:        tokenRecipient[baseTokenId] = newFounder;
114:
115:        emit MintScheduled(baseTokenId, founderId, newFounder);
116:
117:        // Update the base token id
118:        (baseTokenId += schedule) % 100;
119:    }
```

For each founder, the code above is executed. It can be seen that `baseTokenId` is either incremented by `schedule`, or set using `_getNextTokenId()`, as shown below:
```solidity
130:    function _getNextTokenId(uint256 _tokenId) internal view returns (uint256) {
131:        unchecked {
132:            while (tokenRecipient[_tokenId].wallet != address(0)) ++_tokenId;
133:
134:            return _tokenId;
135:        }
136:    }
```

From the logic presented above, it is possible for a founder to be assigned to an index larger than `99` in `tokenRecipient[]`, due to the following two issues:
1. At line 118, the `% 100` statement used to ensure `baseTokenId` is smaller than `100` is never applied
2. `_getNextTokenId()` will always return the next `_tokenId` available

### Issue 1
Due to the syntax error at line 118, `baseTokenId` is never reset to below `100` with the `%` operator. Thus, if enough founders with a percentage ownership are added, `baseTokenId` would eventually exceed `99`.

For example, if founders are added in sequence with the following percentages:
```solidity
1, 1, 1, 1, 1, 20
```
The last founder would receive less tokens than intended, as he is assigned to `tokenRecipient[100]`.

`test_Issue1()` in [this gist](https://gist.github.com/MiloTruck/57ffb59b61365826c2f2fd591e50b2ee) demonstrates this.

### Issue 2
`_getNextTokenId()` does not contain any checks to ensure that the `_tokenId` returned is less than `100`. Thus, if `baseTokenId` equals to `99`, but `tokenRecipient[99]` already corresponds to another founder, `_getNextTokenId()` would automatically return `100`, which is invalid.

The following sequence of percentages demonstrates this:
```solidity
1, 1, 1, 1, 1, 1, 1, 1, 1, 10, 11
```
The last founder would receive less tokens than intended, as he is assigned to `tokenRecipient[100]`.

`test_Issue2()` in [this gist](https://gist.github.com/MiloTruck/57ffb59b61365826c2f2fd591e50b2ee) demonstrates this.

## Recommended Mitigation Steps
For issue 1, rewrite line 118 in `Token.sol` to the following:
```solidity
baseTokenId = (baseTokenId + schedule) % 100;
```

For issue 2, consider adding a check in either `_getNextTokenId()` or `_addFounders()` to ensure `baseTokenid` is always smaller than `100`.
