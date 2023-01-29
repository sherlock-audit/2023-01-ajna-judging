koxuan

medium

# unique constraint on subset erc721 pools can be bypassed

## Summary
`subsetHash_` is used as the unique identifier for the unique constraint. For example, id 1, 2, 3 are the most rare nfts in the collection and therefore the pool wants to only accept that. Another pool that also only accepts id 1, 2, 3 of the same collection with the same quote token can be created due to the way the `subsetHash_` is created and hence bypass the unique constraint. 

## Vulnerability Detail
In `deployPool` of `ERC721PoolFactory`, `getNFTSubsetHash` is used to calculated the unique subsetHash for the particular pool.

```solidity
        deployedPools[getNFTSubsetHash(tokenIds_)][collateral_][quote_] = pool_;
```

The tokenIds are `encodePacked` before hashing with `keccak256`. Continuing from the summary example, the already deployed pool deploys with [1, 2, 3] for the tokenIds parameter in `deployPool`. The same collection token id 1, 2, 3 as collateral with the same quote token should not be allowed to be deployed again. However, a pool creator can deploy with [2, 1, 3] to obtain a different subset hash with the same quote token in order to bypass the unique subset hash check in `canDeploy` modifier used in `deployPool`. 
```solidity
    function getNFTSubsetHash(uint256[] memory tokenIds_) public pure returns (bytes32) {
        if (tokenIds_.length == 0) return ERC721_NON_SUBSET_HASH;
        else return keccak256(abi.encodePacked(tokenIds_));
    }
```

## Impact
unique constraint of subset erc721 of the same collection can be bypassed.

## Code Snippet
[ERC721PoolFactory.sol#L96](https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/ERC721PoolFactory.sol#L96)
[ERC721PoolFactory.sol#L108-L111](https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/ERC721PoolFactory.sol#L108-L111)

## Tool used

Manual Review

## Recommendation

Recommend using a merkle root as the basis for verifying the accepted token ids instead of storing on chain.
