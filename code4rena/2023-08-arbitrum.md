# Arbitrum Security Council Election System

The code under review can be found in [2023-08-arbitrum](https://github.com/code-423n4/2023-08-arbitrum).

## Findings Summary

| ID | Description | Severity |
| :-: | - | :-: |
| [H-01](#h-01-signatures-can-be-replayed-in-castvotewithreasonandparamsbysig-to-use-up-more-votes-than-a-user-intended) | Signatures can be replayed in `castVoteWithReasonAndParamsBySig()` to use up more votes than a user intended | High |
| [M-01](#m-01-missing-__governor_init-call-in-securitycouncilmemberremovalgovernors-initialize-function) | Missing `__Governor_init()` call in `SecurityCouncilMemberRemovalGovernor`'s `initialize()` function | Medium |
| [M-02](#m-02-electiontotimestamp-might-return-incorrect-timestamps-depending-on-the-day-of-the-first-election) | `electionToTimestamp()` might return incorrect timestamps depending on the day of the first election | Medium |
| [M-03](#m-03-setfullweightduration-can-be-called-while-a-member-election-is-ongoing) | `setFullWeightDuration()` can be called while a member election is ongoing | Medium |
| [L-01](#l-01-securitycouncilmanagers-initialize-function-contains-a-gas-bomb) | `SecurityCouncilManager`'s `initialize()` function contains a gas bomb | Low |
| [L-02](#l-02-governance-could-accidentally-dos-member-elections-by-setting-_votingperiod-less-than-fullweightduration) | Governance could accidentally DOS member elections by setting `_votingPeriod` less than `fullWeightDuration` | Low |
| [L-03](#l-03-consider-checking-that-msgvalue-is-0-in-_execute-of-governor-contracts) | Consider checking that `msg.value` is 0 in `_execute()` of governor contracts | Low |
| [L-04](#l-04-governor-contracts-should-prevent-users-from-directly-transferring-eth-or-tokens) | Governor contracts should prevent users from directly transferring ETH or tokens | Low |
| [L-05](#l-05-governance-can-dos-elections-by-setting-votingdelay-or-votingperiod-more-than-typeuint64max) | Governance can DOS elections by setting `votingDelay` or `votingPeriod` more than `type(uint64).max` | Low |
| [L-06](#l-06-areaddressarraysequal-isnt-foolproof-when-both-arrays-have-duplicate-elements) | `areAddressArraysEqual()` isn't foolproof when both arrays have duplicate elements | Low |
| [L-07](#l-07-missing-duplicate-checks-in-l2securitycouncilmgmtfactorys-deploy) | Missing duplicate checks in `L2SecurityCouncilMgmtFactory`'s `deploy()` | Low |
| [L-08](#l-08-topnominees-could-consume-too-much-gas) | `topNominees()` could consume too much gas | Low |
| [L-09](#l-09-nominees-excluded-using-excludenominee-cannot-be-added-back-using-includenominee) | Nominees excluded using `excludeNominee()` cannot be added back using `includeNominee()` | Low |
| [N-01](#n-01-check-that-_addresstoremove-and-_addresstoadd-are-not-equal-in-_swapmembers) | Check that `_addressToRemove` and `_addressToAdd` are not equal in `_swapMembers()` | Non-Critical |
| [N-02](#n-02-document-how-ties-are-handled-for-member-elections) | Document how ties are handled for member elections | Non-Critical |
| [N-03](#n-03-relay-is-not-declared-as-payable) | `relay()` is not declared as `payable` | Non-Critical |

## [H-01] Signatures can be replayed in `castVoteWithReasonAndParamsBySig()` to use up more votes than a user intended

### Bug Description

In the `SecurityCouncilNomineeElectionGovernor` and `SecurityCouncilMemberElectionGovernor` contracts, users can provide a signature to allow someone else to vote on their behalf using the `castVoteWithReasonAndParamsBySig()` function, which is in Openzeppelin's [`GovernorUpgradeable`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol):

[GovernorUpgradeable.sol#L480-L495](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L480-L495)

```solidity
        address voter = ECDSAUpgradeable.recover(
            _hashTypedDataV4(
                keccak256(
                    abi.encode(
                        EXTENDED_BALLOT_TYPEHASH,
                        proposalId,
                        support,
                        keccak256(bytes(reason)),
                        keccak256(params)
                    )
                )
            ),
            v,
            r,
            s
        );
```

As seen from above, the signature provided does not include a nonce. This becomes an issue in nominee and member elections, as users can choose not to use all of their votes in a single call, allowing them split their voting power amongst contenders/nominees:

[Nominee Election Specification](https://forum.arbitrum.foundation/t/proposal-security-council-elections-proposed-implementation-spec/15425#h-1-nominee-selection-7-days-10)

>  A single delegate can split their vote across multiple candidates.

[Member Election Specification](https://forum.arbitrum.foundation/t/proposal-security-council-elections-proposed-implementation-spec/15425#h-3-member-election-21-days-14)

> Additionally, delegates can cast votes for more than one nominee:
> * Split voting. delegates can split their tokens across multiple nominees, with 1 token representing 1 vote.

Due to the lack of a nonce, `castVoteWithReasonAndParamsBySig()` can be called multiple times with the same signature. 

Therefore, if a user provides a signature to use a portion of his votes, an attacker can repeatedly call `castVoteWithReasonAndParamsBySig()` with the same signature to use up more votes than the user originally intended.

### Impact

Due to the lack of signature replay protection in `castVoteWithReasonAndParamsBySig()`, during nominee or member elections, an attacker can force a voter to use more votes on a contender/nominee than intended by replaying his signature multiple times.

### Proof of Concept

Assume that a nominee election is currently ongoing:
* Bob has 1000 votes, he wants to split his votes between contender A and B:
  * He signs one signature to give 500 votes to contender A.
  * He signs a second signature to allocate 500 votes to contender B.
* `castVoteWithReasonAndParamsBySig()` is called to submit Bob's first signature:
  * This gives contender A 500 votes.
* After the transaction is executed, Alice sees Bob's signature in the transaction.
* As Alice wants contender A to be elected, she calls `castVoteWithReasonAndParamsBySig()` with Bob's first signature again:
  * Due to a lack of a nonce, the transaction is executed successfully, giving contender A another 500 votes.
* Now, when `castVoteWithReasonAndParamsBySig()` is called with Bob's second signature, it reverts as all his 1000 votes are already allocated to contender A.

In the scenario above, Alice has managed to allocate all of Bob's votes to contender A against his will. Note that this can also occur in member elections, where split voting is also allowed.

#### Coded Proof

The following test demonstrates how signatures can be replayed multiple times when calling `castVoteWithReasonAndParamsBySig()`:

```solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity 0.8.16;

import "@openzeppelin/contracts-upgradeable/utils/cryptography/ECDSAUpgradeable.sol";
import "test/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.t.sol";

contract SignatureReplay is SecurityCouncilNomineeElectionGovernorTest {
    bytes32 constant _TYPE_HASH = keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");
    bytes32 constant EXTENDED_BALLOT_TYPEHASH = keccak256("ExtendedBallot(uint256 proposalId,uint8 support,string reason,bytes params)");

    function test_castVoteWithReasonAndParamsBySigReplay() public {
        // Setup voter
        (address voter, uint256 privateKey) = makeAddrAndKey("Voter");

        // Create proposal
        uint256 proposalId = _propose();

        // Mock some votes for voter
        uint256 voteCount = governor.quorum(proposalId);
        _mockGetPastVotes(voter, voteCount);

        // Contender adds himself
        address contender = _contender(0);
        _addContender(proposalId, contender);

        // Voter casts vote through castVoteWithReasonAndParamsBySig()
        uint256 votesToUse = voteCount / 2;
        (uint8 v, bytes32 r, bytes32 s) = getDigest(
            privateKey,
            proposalId,
            1,
            "",
            abi.encode(contender, votesToUse)
        );
        governor.castVoteWithReasonAndParamsBySig(
            proposalId,
            1,
            "",
            abi.encode(contender, votesToUse),
            v, r, s
        );

        // Voter used half his votes for contender
        assertEq(governor.votesUsed(proposalId, voter), votesToUse);
        assertEq(governor.votesReceived(proposalId, contender), votesToUse);

        // The same signature is replayed
        governor.castVoteWithReasonAndParamsBySig(
            proposalId,
            1,
            "",
            abi.encode(contender, votesToUse),
            v, r, s
        );        

        // All of his votes are allocated to contender
        assertEq(governor.votesUsed(proposalId, voter), voteCount);
        assertEq(governor.votesReceived(proposalId, contender), voteCount);
    }

    function getDigest(
        uint256 privateKey,
        uint256 proposalId, 
        uint8 support, 
        string memory reason, 
        bytes memory params
    ) internal returns (uint8, bytes32, bytes32) {
        bytes32 hashedName = keccak256(bytes(governor.name()));
        bytes32 hashedVersion = keccak256(bytes(governor.version()));
        bytes32 domainSeperator = keccak256(abi.encode(
            _TYPE_HASH,
            hashedName, 
            hashedVersion, 
            block.chainid, 
            address(governor)
        ));

        bytes32 structHash = keccak256(abi.encode(
            EXTENDED_BALLOT_TYPEHASH,
            proposalId,
            support,
            keccak256(bytes(reason)),
            keccak256(params)
        ));
        
        bytes32 digest = ECDSAUpgradeable.toTypedDataHash(domainSeperator, structHash);

        return vm.sign(privateKey, digest);
    }
}
```

### Recommended Mitigation

Consider adding some form of signature replay protection in the `SecurityCouncilNomineeElectionGovernor` and `SecurityCouncilMemberElectionGovernor` contracts.

One way of achieving this is to override the `castVoteWithReasonAndParamsBySig()` function to include a nonce in the signature, which would protect against signature replay.

## [M-01] Missing `__Governor_init()` call in `SecurityCouncilMemberRemovalGovernor`'s `initialize()` function

### Bug Description

The `SecurityCouncilMemberRemovalGovernor` contract inherits Openzeppelin's `GovernorUpgradeable`:

[SecurityCouncilMemberRemovalGovernor.sol#L17-L19](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberRemovalGovernor.sol#L17-L19)

```solidity
contract SecurityCouncilMemberRemovalGovernor is
    Initializable,
    GovernorUpgradeable,
```

However, in its [`initialize()`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberRemovalGovernor.sol#L58-L84) function, `__Governor_init()` is never called. `__Governor_init()` is used to initialize the `GovernorUpgradeable` contract, which sets `_name`, `_HASHED_NAME` and  `_HASHED_VERSION`:

[GovernorUpgradeable.sol#L79-L82](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L79-L82)

```solidity
    function __Governor_init(string memory name_) internal onlyInitializing {
        __EIP712_init_unchained(name_, version());
        __Governor_init_unchained(name_);
    }
```

[GovernorUpgradeable.sol#L84-L86](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L84-L86)

```solidity
    function __Governor_init_unchained(string memory name_) internal onlyInitializing {
        _name = name_;
    }
```

[draft-EIP712Upgradeable.sol#L54-L59](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/utils/cryptography/draft-EIP712Upgradeable.sol#L54-L59)

```solidity
    function __EIP712_init_unchained(string memory name, string memory version) internal onlyInitializing {
        bytes32 hashedName = keccak256(bytes(name));
        bytes32 hashedVersion = keccak256(bytes(version));
        _HASHED_NAME = hashedName;
        _HASHED_VERSION = hashedVersion;
    }
```

As such, after `SecurityCouncilMemberRemovalGovernor` is deployed and initialized, `_HASHED_NAME` and `_HASHED_VERSION` will still be `bytes32(0)`. These values are used to build the domain separator when verifying signatures:

[draft-EIP712Upgradeable.sol#L64-L66](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/utils/cryptography/draft-EIP712Upgradeable.sol#L64-L66)

```solidity
    function _domainSeparatorV4() internal view returns (bytes32) {
        return _buildDomainSeparator(_TYPE_HASH, _EIP712NameHash(), _EIP712VersionHash());
    }
```

[draft-EIP712Upgradeable.sol#L101-L103](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/utils/cryptography/draft-EIP712Upgradeable.sol#L101-L103)

```solidity
    function _EIP712NameHash() internal virtual view returns (bytes32) {
        return _HASHED_NAME;
    }
```

[draft-EIP712Upgradeable.sol#L111-L113](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/utils/cryptography/draft-EIP712Upgradeable.sol#L111-L113)

```solidity
    function _EIP712VersionHash() internal virtual view returns (bytes32) {
        return _HASHED_VERSION;
    }
```

This becomes an problematic when calling [`castVoteBySig()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L449-L466) and [`castVoteWithReasonAndParamsBySig()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L468-L498), which relies on signatures for voting.


### Impact

Since `__Governor_init()` is never called, [`castVoteBySig()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L449-L466) and [`castVoteWithReasonAndParamsBySig()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L468-L498) will revert for signatures generated using the [`name()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L112-L117) and [`version()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L119-L124) functions. This could break the functionality of frontends/contracts that rely on these functions to integrate with the protocol.

Note that this also violates the [EIP-712](https://eips.ethereum.org/EIPS/eip-712) standard as the name and version parameters are incorrectly set to `bytes32(0)` when verifying signatures.

### Recommended Mitigation

Call `__Governor__init()` in the contract's `initialize()` function:

[SecurityCouncilMemberRemovalGovernor#L69-L76](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberRemovalGovernor.sol#L70-L76)

```diff
    ) public initializer {
+       __Governor_init("SecurityCouncilMemberRemovalGovernor");
        __GovernorSettings_init(_votingDelay, _votingPeriod, _proposalThreshold);
        __GovernorCountingSimple_init();
        __GovernorVotes_init(_token);
        __ArbitrumGovernorVotesQuorumFraction_init(_quorumNumerator);
        __GovernorPreventLateQuorum_init(_minPeriodAfterQuorum);
        __ArbitrumGovernorProposalExpirationUpgradeable_init(_proposalExpirationBlocks);
        _transferOwnership(_owner);
```

## [M-02] `electionToTimestamp()` might return incorrect timestamps depending on the day of the first election

### Bug Description

For nominee elections, election dates are determined using the the `electionToTimestamp()` function in the `SecurityCouncilNomineeElectionGovernorTiming` module. 

When `SecurityCouncilNomineeElectionGovernor` is initialized after deployment, the first election date is stored through [`__SecurityCouncilNomineeElectionGovernorTiming_init()`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilNomineeElectionGovernorTiming.sol#L26-L66). Afterwards, `electionToTimestamp()` will provide the next election timestamp based on the number of elections passed:

[SecurityCouncilNomineeElectionGovernorTiming.sol#L75-L94](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilNomineeElectionGovernorTiming.sol#L73-L94)

```solidity
    function electionToTimestamp(uint256 electionIndex) public view returns (uint256) {
        // subtract one to make month 0 indexed
        uint256 month = firstNominationStartDate.month - 1;

        month += 6 * electionIndex;
        uint256 year = firstNominationStartDate.year + month / 12;
        month = month % 12;

        // add one to make month 1 indexed
        month += 1;

        return DateTimeLib.dateTimeToTimestamp({
            year: year,
            month: month,
            day: firstNominationStartDate.day,
            hour: firstNominationStartDate.hour,
            minute: 0,
            second: 0
        });
    }
```

As seen from above, `electionToTimestamp()` works by adding 6 months for every passed election, and then converting the date to a timestamp through Solady's `dateTimeToTimestamp()`.

However, this approach does not account for months that have less days than others. 

For example, if the first election was scheduled on 31 August, the next election would be 31 February according to the formula above. However, as February doesn't have 31 days, the `day` parameter is outside the range supported by `dateTimeToTimestamp()`, resulting in undefined behavior:

[DateTimeLib.sol#L131-L133](https://github.com/Vectorized/solady/blob/main/src/utils/DateTimeLib.sol#L131-L133)

```solidity
    /// Note: Inputs outside the supported ranges result in undefined behavior.
    /// Use {isSupportedDateTime} to check if the inputs are supported.
    function dateTimeToTimestamp(
```

Therefore, `dateTimeToTimestamp()` will return an incorrect timestamp.

### Impact

If the the first election starts on the 29th to 31st day of the month, `dateTimeToTimestamp()` could potentially return an incorrect timestamp for subsequent elections.

### Proof of Concept

Assume that the first election is brought forward from 15 September 2023 to 31 August 2023. Every alternate election will now be held in February, which creates two problems:

#### 1. The election date for one cohort will not be fixed

If the current year is a leap year, the election that was supposed to be held on February will be one day earlier than a non-leap year. For example:
* Since 2024 is a leap year, the second election will be on 2 March 2024.
* Since 2025 is not a leap year, the fourth election will be on 3 March 2025.

This becomes a problem as the [Arbitrum Constitution](https://docs.arbitrum.foundation/dao-constitution) states a specific date for the two elections in a year, which is not possible if the scenario above occurs.

#### 2. One term is a few days shorter for a cohort

As mentioned above, if the start date was 31 August 2023, the fourth election will be on 3 March 2025. However, if the first election was on 3 September 2023, the fourth election would still be on 3 March 2025.

This means that the election starts three days later for the scenario above, making the term for one cohort a few days longer than intended.

#### Coded Proof

The following test demonstrates how leap years affects `electionToTimestamp()`:

```solidity
// SPDX-License-Identifier: Apache-2.0
pragma solidity 0.8.16;

import "forge-std/Test.sol";
import "src/security-council-mgmt/governors/modules/SecurityCouncilNomineeElectionGovernorTiming.sol";

contract ElectionDates is Test, SecurityCouncilNomineeElectionGovernorTiming {
    function setUp() public initializer {
        // Set first election date to 31 August 2023, 12:00
        __SecurityCouncilNomineeElectionGovernorTiming_init(
            Date({
                year: 2023,
                month: 8,
                day: 31,
                hour: 12
            }),
            0
        );
    }

    function test_electionToTimestampIncorrect() public {
        // First election is on 31 August 2023
        assertEq(electionToTimestamp(0), 1693483200);
        
        // Second election is on 2 March 2024
        assertEq(electionToTimestamp(1), 1709380800);
        
        // Fourth election is on 3 March 2025
        assertEq(electionToTimestamp(3), 1741003200);
    }

    // Required override functions
    function COUNTING_MODE() public pure override returns (string memory) {}
    function votingDelay() public view override returns (uint256) {}
    function votingPeriod() public view override returns (uint256) {}
    function quorum(uint256) public view override returns (uint256) {}
    function hasVoted(uint256, address) public view override returns (bool) {}
    function _quorumReached(uint256) internal view override returns (bool) {}
    function _voteSucceeded(uint256) internal view override returns (bool) {}
    function _getVotes(address, uint256, bytes memory) internal view override returns (uint256) {}
    function _countVote(uint256, address, uint8, uint256, bytes memory) internal override {}
}
```

### Recommended Mitigation

Ensure that `_firstNominationStartDate.day` is never above 28:

[SecurityCouncilNomineeElectionGovernorTiming.sol#L41-L48](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilNomineeElectionGovernorTiming.sol#L41-L48)

```diff
-       if (!isSupportedDateTime) {
+       if (!isSupportedDateTime || _firstNominationStartDate.day > 28) {
            revert InvalidStartDate(
                _firstNominationStartDate.year,
                _firstNominationStartDate.month,
                _firstNominationStartDate.day,
                _firstNominationStartDate.hour
            );
        }
```

Additionally, consider storing `startTimestamp` instead of `firstNominationStartDate`. With the first election's timestamp, subsequent election timestamps can be calculated using `DateTimeLib.addMonths()`  instead:

```solidity
    function electionToTimestamp(uint256 electionIndex) public view returns (uint256) { 
        return DateTimeLib.addMonths(startTimestamp, electionIndex * 6);
    }
```

Using `addMonths()` ensures that election dates are always fixed, even in the scenario mentioned above.

## [M-03] `setFullWeightDuration()` can be called while a member election is ongoing

### Bug Description

In `SecurityCouncilMemberElectionGovernorCountingUpgradeable`, `fullWeightDuration` (which is the duration where a user's votes has weight 1) can be set using `setFullWeightDuration()`:

[SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L77-L84](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L77-L84)

```solidity
    function setFullWeightDuration(uint256 newFullWeightDuration) public onlyGovernance {
        if (newFullWeightDuration > votingPeriod()) {
            revert FullWeightDurationGreaterThanVotingPeriod(newFullWeightDuration, votingPeriod());
        }

        fullWeightDuration = newFullWeightDuration;
        emit FullWeightDurationSet(newFullWeightDuration);
    }
```

`fullWeightDuration` is then used to calculate the weightage of a user's votes in `votesToWeight()`:

[SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L247-L255](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L247-L255)

```solidity
        // Between the fullWeightVotingDeadline and the proposalDeadline each vote will have weight linearly decreased by time since fullWeightVotingDeadline
        // slope denominator
        uint256 decreasingWeightDuration = endBlock - fullWeightVotingDeadline_;
        // slope numerator is -votes, slope denominator is decreasingWeightDuration, delta x is blockNumber - fullWeightVotingDeadline_
        // y intercept is votes
        uint256 decreaseAmount =
            votes * (blockNumber - fullWeightVotingDeadline_) / decreasingWeightDuration;
        // subtract the decreased amount to get the remaining weight
        return _downCast(votes - decreaseAmount);
```

Where:
- `fullWeightVotingDeadline_` is equal to `startBlock + fullWeightDuration`.

However, as there is no restriction on when `setFullWeightDuration()` can be called, it could potentially be unfair for voters if `fullWeightDuration` is increased or decreased while an election is ongoing. For example:

- Assume the following:
  - `startBlock = 1000000`, `endBlock = 2000000`, `fullWeightDuration = 500000`
  - `block.timestamp` is currently `1625000`
- Alice calls `castVote()` to vote for a nominee:
  - As a quarter of `decreasingWeightDuration` has passed, her votes have a 75% weightage.
- Governance suddenly calls `setFullWeightDuration()` to set `fullWeightDuration` to `750000`.
- `block.timestamp` is now within the full weight duration, meaning that users who vote now will have a 100% weightage. However, Alice has already voted and cannot undo her votes.

In the scenario above, future voters will have a larger weight than Alice for the same amount of votes, making it unfair for her.

### Impact

If `setFullWeightDuration()` is called during an election, users who have already voted will be unfairly affected as their votes will have more/less weight than they should.

Given that `setFullWeightDuration()` is called by governance, which has to be scheduled through timelocks, it might be possible for governance to accidentally schedule a call that updates `fullWeightDuration` while an election is ongoing.

### Recommended Mitigation

Consider allowing `setFullWeightDuration()` to be called only when there isn't an election ongoing. 

One way of knowing when an election is ongoing is to track when the `proposeFromNomineeElectionGovernor()` and `_execute()` functions are called, which mark the start and end of an election respectively.

## [L-01] `SecurityCouncilManager`'s `initialize()` function contains a gas bomb

### Bug Description

In `SecurityCouncilManager.sol`, the `initialize()` function calls `_addSecurityCouncil()` in a loop to add security councils individually: 

[SecurityCouncilManager.sol#L118-L120](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/SecurityCouncilManager.sol#L118-L120)

```solidity
        for (uint256 i = 0; i < _securityCouncils.length; i++) {
            _addSecurityCouncil(_securityCouncils[i]);
        }
```

`_addSecurityCouncil()` performs checks, which includes ensuring the new security council (`_securityCouncilData`) isn't already added, before adding it to the `securityCouncils` array: 

[SecurityCouncilManager.sol#L251-L262](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/SecurityCouncilManager.sol#L251-L262)

```solidity
        for (uint256 i = 0; i < securityCouncils.length; i++) {
            SecurityCouncilData storage existantSecurityCouncil = securityCouncils[i];

            if (
                existantSecurityCouncil.chainId == _securityCouncilData.chainId
                    && existantSecurityCouncil.securityCouncil == _securityCouncilData.securityCouncil
            ) {
                revert SecurityCouncilAlreadyInRouter(_securityCouncilData);
            }
        }

        securityCouncils.push(_securityCouncilData);
```

However, as the duplicate check loops over all elements in the `securityCouncils` storage array, `_addSecurityCouncil()` will consume a lot of gas whenever it is called. 

As it is called repeatedly in `initialize()`, there is a significant chance that `initialize()` might consume too much gas when called, making it revert due to an out-of-gas error.

The contract declares a maximum limit of 500 security councils to mitigate this:

[SecurityCouncilManager.sol#L67](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/SecurityCouncilManager.sol#L67)

```solidity
    uint256 public immutable MAX_SECURITY_COUNCILS = 500;
```

However, this is insufficient as calling `initialize()` with 500 security councils will still read from storage 125,250 times, which will still consume a huge amount of gas.

### Impact

If `SecurityCouncilManager` is initialized with a large number of security councils, the `initialize()` function might not be executable due to consuming too much gas.

### Recommended Mitigation

Consider performing all checks in `initialize()` and pushing to the `securityCouncils` array directly, instead of calling `_addSecurityCouncil`. This might reduce gas consumption significantly as the iteration is performed over the array stored in memory, thereby avoiding reading from storage.

## [L-02] Governance could accidentally DOS member elections by setting `_votingPeriod` less than `fullWeightDuration`

### Bug Description

In `SecurityCouncilMemberElectionGovernorCountingUpgradeable`, `setFullWeightDuration()` has a check to ensure that `fullWeightDuration` is more than the voting period:

[SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L77-L84](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L77-L84)

```solidity
    function setFullWeightDuration(uint256 newFullWeightDuration) public onlyGovernance {
        if (newFullWeightDuration > votingPeriod()) {
            revert FullWeightDurationGreaterThanVotingPeriod(newFullWeightDuration, votingPeriod());
        }

        fullWeightDuration = newFullWeightDuration;
        emit FullWeightDurationSet(newFullWeightDuration);
    }
```

However, the [`setVotingPeriod()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/extensions/GovernorSettingsUpgradeable.sol#L74-L81) function in Openzeppelin's [`GovernorSettingsUpgradeable`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/extensions/GovernorSettingsUpgradeable.sol) module doesn't ensure that `_votingPeriod` is above `fullWeightDuration`. This means that governance could accidentally cause `fullWeightDuration` to be greater than `_votingPeriod` by calling `setVotingPeriod()` to decrease the voting period.

Should this occur, `votesToWeight()` will revert due to an arithmetic underflow when performing the following calculation:

[SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L241-L249](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L241-L249)

```solidity
        // Between proposalSnapshot and fullWeightVotingDeadline all votes will have 100% weight - each vote has weight 1
        uint256 fullWeightVotingDeadline_ = fullWeightVotingDeadline(proposalId);
        if (blockNumber <= fullWeightVotingDeadline_) {
            return _downCast(votes);
        }

        // Between the fullWeightVotingDeadline and the proposalDeadline each vote will have weight linearly decreased by time since fullWeightVotingDeadline
        // slope denominator
        uint256 decreasingWeightDuration = endBlock - fullWeightVotingDeadline_;
```

Where:
- `endBlock` is equal to `startBlock + _votingPeriod`.
- `fullWeightVotingDeadline_` is equal to `startBlock + fullWeightDuration`.

Since `votesToWeight()` is used to determine the weightage of votes in [`_countVote()`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L86-L140), all voting functions (eg. [`castVote()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L416-L422)) will always revert once `fullWeightVotingDeadline_` has passed, causing all voting to be DOSed.

### Impact

Governance could accidentally DOS voting for member elections by reducing the voting period below `fullWeightDuration` using `setVotingPeriod()`. 

This could occur if `fullWeightDuration` is initially equal to `votingPeriod` (votes have 100% weightage during the entire voting period), and governance decides to reduce the voting period to a shorter duration.

Given that `setFullWeightDuration()` is also called by governance, and has to be scheduled through timelocks, it might not be possible for governance to call `setFullWeightDuration()` to reduce `fullWeightDuration` in time after realizing the DOS has occurred.

### Recommended Mitigation

In the [`SecurityCouncilMemberElectionGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberElectionGovernor.sol) contract, consider overriding the `setVotingPeriod()` function to ensure that the new voting period is always greater than `fullWeightDuration`. For example:

```solidity
    function setVotingPeriod(uint256 newVotingPeriod) public override onlyGovernance {
        if (newVotingPeriod <= fullWeightDuration) {
            revert NewVotingPeriodLessThanFullWeightDuration();
        }

        _setVotingPeriod(newVotingPeriod);
    }
```

## [L-03] Consider checking that `msg.value` is 0 in `_execute()` of governor contracts

In Openzeppelin's `GovernorUpgradeable`, `execute()` is declared as `payable`:

[GovernorUpgradeable.sol#L295-L300](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L295-L300)

```solidity
    function execute(
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        bytes32 descriptionHash
    ) public payable virtual override returns (uint256) {
```

This makes it possible for users to accidentally transfer ETH to the governor contracts when calling `execute()`.

### Recommendation

In [`SecurityCouncilNomineeElectionGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.sol), [`SecurityCouncilMemberRemovalGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberRemovalGovernor.sol) and [`SecurityCouncilMemberElectionGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberElectionGovernor.sol), consider overriding `_execute()` and reverting if `msg.value` is not 0. This ensures that users cannot accidentally lose their ETH while calling `execute()`.

## [L-04] Governor contracts should prevent users from directly transferring ETH or tokens

Openzeppelin's `GovernorUpgradeable` contract contains [`receive()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L88-L93), [`onERC721Received()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L568-L575), [`onERC1155Received()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L580-L588) and [`onERC1155BatchReceived()`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L590-L609) to allow inheriting contracts to receive ETH and tokens. 

However, this allows users to accidentally transfer their ETH/tokens to the governor contracts, which will then remain stuck until they are rescued by governance.

### Recommendation

In [`SecurityCouncilNomineeElectionGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.sol), [`SecurityCouncilMemberRemovalGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberRemovalGovernor.sol) and [`SecurityCouncilMemberElectionGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberElectionGovernor.sol), consider overriding these functions and making them revert. This prevents users from accidentally transferring ETH/tokens to the contracts.

## [L-05] Governance can DOS elections by setting `votingDelay` or `votingPeriod` more than `type(uint64).max`

In the `propose()` function of Openzeppelin's `GovernorUpgradeable` contract, `votingDelay` and `votingPeriod` are cast from `uint256` to `uint64` safely:

[GovernorUpgradeable.sol#L271-L272](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/release-v4.7/contracts/governance/GovernorUpgradeable.sol#L271-L272)

```solidity
        uint64 snapshot = block.number.toUint64() + votingDelay().toUint64();
        uint64 deadline = snapshot + votingPeriod().toUint64();
```

Therefore, if either of these values are set to above `type(uint64).max`, `propose()` will revert.


### Recommendation

In [`SecurityCouncilNomineeElectionGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.sol), [`SecurityCouncilMemberRemovalGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberRemovalGovernor.sol) and [`SecurityCouncilMemberElectionGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberElectionGovernor.sol), consider overriding the `setVotingDelay()` and `setVotingPeriod()` functions to check that `votingDelay` and `votingPeriod` are not set to values above `type(uint64).max`.


## [L-06] `areAddressArraysEqual()` isn't foolproof when both arrays have duplicate elements

The `areAddressArraysEqual()` function is used to check if `array1` and `array2` contain the same elements. It does so by checking that each element in `array1` exists in `array2`, and vice versa:

[SecurityCouncilMgmtUpgradeLib.sol#L61-L85](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/gov-action-contracts/AIPs/SecurityCouncilMgmt/SecurityCouncilMgmtUpgradeLib.sol#L61-L85)

```solidity
        for (uint256 i = 0; i < array1.length; i++) {
            bool found = false;
            for (uint256 j = 0; j < array2.length; j++) {
                if (array1[i] == array2[j]) {
                    found = true;
                    break;
                }
            }
            if (!found) {
                return false;
            }
        }

        for (uint256 i = 0; i < array2.length; i++) {
            bool found = false;
            for (uint256 j = 0; j < array1.length; j++) {
                if (array2[i] == array1[j]) {
                    found = true;
                    break;
                }
            }
            if (!found) {
                return false;
            }
        }
```

However, this method isn't foolproof when both `array1` and `array2` contain duplicate elements. For example:
- `array1 = [1, 1, 2]`
- `array2 = [1, 2, 2]` 

Even though both arrays are not equal, `areAddressArraysEqual()` will return true as they have the same length and all elements in one array exist in the other.

### Recommendation

Consider checking that both arrays do not contain duplicate elements.

## [L-07] Missing duplicate checks in `L2SecurityCouncilMgmtFactory`'s `deploy()`

In `L2SecurityCouncilMgmtFactory.sol`, the `deploy()` function only checks that every address in every cohort is an owner in `govChainEmergencySCSafe`:

[L2SecurityCouncilMgmtFactory.sol#L111-L121](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/factories/L2SecurityCouncilMgmtFactory.sol#L111-L121)

```solidity
        for (uint256 i = 0; i < dp.firstCohort.length; i++) {
            if (!govChainEmergencySCSafe.isOwner(dp.firstCohort[i])) {
                revert AddressNotInCouncil(owners, dp.firstCohort[i]);
            }
        }

        for (uint256 i = 0; i < dp.secondCohort.length; i++) {
            if (!govChainEmergencySCSafe.isOwner(dp.secondCohort[i])) {
                revert AddressNotInCouncil(owners, dp.secondCohort[i]);
            }
        }
```

However, there is no check to ensure that `firstCohort` and `secondCohort` do not contain any duplicates, or that any address in one cohort is not in the other. This makes it possible for the `SecurityCouncilManager` contract to be deployed with incorrect cohorts.

### Recommendation

Consider checking the following:
- `firstCohort` and `secondCohort` do not contain any duplicates
- An address that is in `firstCohort` must not be in `secondCohort`, and vice versa.

## [L-08] `topNominees()` could consume too much gas

In `SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol`, the `topNominees()` function is extremely gas-intensive due to the following reasons:
- `_compliantNominees()` copies the entire nominees array  the storage of the `SecurityCouncilNomineeElectionGovernor` contract:

[SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L178](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L178)

```solidity
        address[] memory nominees = _compliantNominees(proposalId);
```

- `selectTopNominees()` iterates over all nominees and in the worst-case scenario, calls `LibSort.insertionSort()` in each iteration:

[SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L205-L212](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L205-L212)

```solidity
        for (uint16 i = 0; i < nominees.length; i++) {
            uint256 packed = (uint256(weights[i]) << 16) | i;

            if (topNomineesPacked[0] < packed) {
                topNomineesPacked[0] = packed;
                LibSort.insertionSort(topNomineesPacked);
            }
        }
```

If the number of nominees is too large for an election, there is a significant chance that the `topNominees()` function will consume too much gas and revert due to an out-of-gas error. 

If this occurs, member elections will be stuck permanently as proposals cannot be executed. This is because [`_execute()`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberElectionGovernor.sol#L112-L133) calls `topNominees()` to select the top nominees to replace the cohort in `SecurityCouncilManager`. 

The number of nominees for an election is implicitly limited by the percentage of votes a contender needs to become a nominee. Currently, this is set to 0.2% which makes the maximum number of nominees 500. However, this also means that number of nominees could increase significantly should the percentage be decreased in the future. 

## [L-09] Nominees excluded using `excludeNominee()` cannot be added back using `includeNominee()`

In `SecurityCouncilNomineeElectionGovernor`, once nominees are excluded from the election by the nominee vetter using [`excludeNominee()`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.sol#L263-L283), they cannot be added back using the [`includeNominee()`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.sol#L285-L317).

This is because excluded nominees are not removed from the array of nominees, but are simply marked as excluded in the `isExcluded` mapping:

[SecurityCouncilNomineeElectionGovernor.sol#L279-L280](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.sol#L279-L280)

```solidity
        election.isExcluded[nominee] = true;
        election.excludedNomineeCount++;
```

Therefore, following check in `includeNominee()` will still fail when it is called for excluded nominees:

[SecurityCouncilNomineeElectionGovernor.sol#L296-L298](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.sol#L296-L298)

```solidity
        if (isNominee(proposalId, account)) {
            revert NomineeAlreadyAdded(account);
        }
```

This could become a problem if the nominee vetter accidentally calls `excludeNominee()` on the wrong nominee, or if there is some other legitimate reason a previously excluded nominee needs to be added back to the election.

## [N-01] Check that `_addressToRemove` and `_addressToAdd` are not equal in `_swapMembers()`

In `_swapMembers()`, consider checking that `_addressToRemove` and `_addressToAdd` are not the same address:

[SecurityCouncilManager.sol#L218-L229](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/SecurityCouncilManager.sol#L218-L229)

```diff
    function _swapMembers(address _addressToRemove, address _addressToAdd)
        internal
        returns (Cohort)
    {
        if (_addressToRemove == address(0) || _addressToAdd == address(0)) {
            revert ZeroAddress();
        }
+       if (_addressToRemove == _addressToAdd) {
+           revert CannotSwapSameMembers();
+       }    
        Cohort cohort = _removeMemberFromCohortArray(_addressToRemove);
        _addMemberToCohortArray(_addressToAdd, cohort);
        _scheduleUpdate();
        return cohort;
    }
```

This would prevent scheduling unnecessary updates as there no changes to the security council members. 

## [N-02] Document how ties are handled for member elections

In the [Arbitrum Constitution](https://docs.arbitrum.foundation/dao-constitution), there is no specification on how members are chosen in the event nominees are tied for votes.

Currently, `selectTopNominees()` simply picks the first 6 nominees after `LibSort.insertionSort()` is called, which means the nominee selected is random in the event they tie:

[SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L203-L217](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/modules/SecurityCouncilMemberElectionGovernorCountingUpgradeable.sol#L203-L217)

```solidity
        uint256[] memory topNomineesPacked = new uint256[](k);

        for (uint16 i = 0; i < nominees.length; i++) {
            uint256 packed = (uint256(weights[i]) << 16) | i;

            if (topNomineesPacked[0] < packed) {
                topNomineesPacked[0] = packed;
                LibSort.insertionSort(topNomineesPacked);
            }
        }

        address[] memory topNomineesAddresses = new address[](k);
        for (uint16 i = 0; i < k; i++) {
            topNomineesAddresses[i] = nominees[uint16(topNomineesPacked[i])];
        }
```

This could be confusing for users who expect tiebreaks to be handled in a deterministic manner (eg. whoever got the number of votes first).

### Recommendation

Consider documenting how voting ties are handled in the [Arbitrum Constitution](https://docs.arbitrum.foundation/dao-constitution) to prevent confusion.

## [N-03] `relay()` is not declared as `payable`

In `SecurityCouncilNomineeElectionGovernor`, although `relay()` makes calls with `AddressUpgradeable.functionCallWithValue()`, it is not declared as `payable`:

[SecurityCouncilNomineeElectionGovernor.sol#L254-L261](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.sol#L254-L261)

```solidity
    function relay(address target, uint256 value, bytes calldata data)
        external
        virtual
        override
        onlyOwner
    {
        AddressUpgradeable.functionCallWithValue(target, data, value);
    }
```

This limits the functionality of `relay()`, as governance will not be able to send ETH to this contract and transfer the ETH to `target` in a single call to `relay()`.

### Recommendation

Consider declaring `relay()` as `payable`:

[SecurityCouncilNomineeElectionGovernor.sol#L254-L261](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilNomineeElectionGovernor.sol#L254-L261)

```solidity
    function relay(address target, uint256 value, bytes calldata data)
        external
+       payable
        virtual
        override
        onlyOwner
    {
        AddressUpgradeable.functionCallWithValue(target, data, value);
    }
```

This applies to the `relay()` function in [`SecurityCouncilMemberElectionGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberElectionGovernor.sol) and [`SecurityCouncilMemberRemovalGovernor`](https://github.com/ArbitrumFoundation/governance/blob/c18de53820c505fc459f766c1b224810eaeaabc5/src/security-council-mgmt/governors/SecurityCouncilMemberRemovalGovernor.sol) as well.