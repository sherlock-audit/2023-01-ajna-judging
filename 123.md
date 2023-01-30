berndartmueller

medium

# ERC-20 tokens with more than 18 decimals are not supported

## Summary

Calculating the quote and/or collateral token scale when deploying a new pool by subtracting the token decimals from 18 will revert if the token has more than 18 decimals.

## Vulnerability Detail

The ERC-721 and ERC-20 pool factories calculate the quote and/or collateral token scale by subtracting the token decimals from 18. However, if the token has more than 18 decimals (see [Weird ERC-20 tokens](https://github.com/d-xo/weird-erc20#high-decimals)), the calculation will revert due to underflow.

## Impact

ERC-20 tokens with more than 18 decimals are not supported and will revert when used to deploy an ERC-721 or ERC-20 pool.

## Code Snippet

[contracts/src/ERC721PoolFactory.sol#L57](https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/ERC721PoolFactory.sol#L57)

```solidity
54: function deployPool(
55:     address collateral_, address quote_, uint256[] memory tokenIds_, uint256 interestRate_
56: ) external canDeploy(getNFTSubsetHash(tokenIds_), collateral_, quote_, interestRate_) returns (address pool_) {
57:     uint256 quoteTokenScale = 10**(18 - IERC20Token(quote_).decimals());
```

[contracts/src/ERC20PoolFactory.sol#L53-L54](https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/ERC20PoolFactory.sol#L53-L54)

```solidity
50: function deployPool(
51:     address collateral_, address quote_, uint256 interestRate_
52: ) external canDeploy(ERC20_NON_SUBSET_HASH, collateral_, quote_, interestRate_) returns (address pool_) {
53:     uint256 quoteTokenScale = 10 ** (18 - IERC20Token(quote_).decimals());
54:     uint256 collateralScale = 10 ** (18 - IERC20Token(collateral_).decimals());
```

## Tool used

Manual Review

## Recommendation

Consider handling the case where the token has more than 18 decimals.
