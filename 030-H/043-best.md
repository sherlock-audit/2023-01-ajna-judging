ctf_sec

high

# Proposer can vote multiple times within the same block by transferring the token to multiple address

## Summary

Proposer can vote multiple times within the same block by transferring the token to multiple address

## Vulnerability Detail

The implementation for voting mechanism for extraordinaryFunding is vulnerable:

```solidity
function _castVote
```

which calls:

```solidity
FundingMechanism mechanism = findMechanismOfProposal(proposalId_);

// standard funding mechanism
if (mechanism == FundingMechanism.Standard) {
    Proposal storage proposal = standardFundingProposals[proposalId_];
    QuarterlyDistribution memory currentDistribution = distributions[proposal.distributionId];
    uint256 screeningPeriodEndBlock = currentDistribution.endBlock - 72000;

    // screening stage
    if (block.number >= currentDistribution.startBlock && block.number <= screeningPeriodEndBlock) {
        uint256 votes = _getVotes(account_, block.number, bytes("Screening"));

        votesCast_ = _screeningVote(account_, proposal, votes);
    }
else {
	votesCast_ = _extraordinaryFundingVote(proposalId_, account_);
}
```

which calls:

```solidity
function _extraordinaryFundingVote(uint256 proposalId_, address account_) internal returns (uint256 votes_) {
	if (hasVotedExtraordinary[proposalId_][account_]) revert AlreadyVoted();

	ExtraordinaryFundingProposal storage proposal = extraordinaryFundingProposals[proposalId_];
	if (proposal.startBlock > block.number || proposal.endBlock < block.number || proposal.executed) {
		revert ExtraordinaryFundingProposalInactive();
	}

	// check voting power at snapshot block
	votes_ = _getVotes(account_, block.number, abi.encode(proposalId_));
	proposal.votesReceived += votes_;

	// record that voter has voted on this extraorindary funding proposal
	hasVotedExtraordinary[proposalId_][account_] = true;

	emit VoteCast(account_, proposalId_, 1, votes_, "");
}
```

mainly because counting mechanism count the voting power in the current block.number

```solidity
// check voting power at snapshot block
votes_ = _getVotes(account_, block.number, abi.encode(proposalId_));
```

## Impact

A adversary can propose a urgent propose to let the contract grant extraordinaryFunding, then purchase 10000 AJNA token,

he build a smart contract then sniper the precise block number of voting:

he can keep deploying new smart contract and transfer the same token to the smart contract and use the smart contract to vote again.

The cycle is Vote -> deploy new contract -> transfer the old token balance into new contract -> vote -> deploy new contract -> transfer the old token balanc into new contract -> vote.

Until the voting power pass the threshold then he can execute the proposal and get the granted token.

**Reference finding** (that finding is that user can vote multiple times by transfer the NFT to different address within the same block, this finding is that the user can transfer the token to multiple address and vote multiple times with the same block):

https://github.com/code-423n4/2022-09-party-findings/issues/113


## Code Snippet

https://github.com/sherlock-audit/2023-01-ajna/blob/main/ecosystem-coordination/src/grants/base/ExtraordinaryFunding.sol#L110-L136

## Tool used

Manual Review

## Recommendation

We recommend the protocol count at least the block.number - 1 snapshot

```solidity
votes_ = _getVotes(account_, block.number - 1 abi.encode(proposalId_));
```
