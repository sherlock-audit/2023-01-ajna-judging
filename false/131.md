tsvetanovv

medium

# It is possible Reentrancy in `mint()` function

## Summary
In `PositionManager.sol`  the `mint()` function have Reentrancy vulnerability.

## Vulnerability Detail
When writing or interacting with callback functions in solidity, it's important to ensure that they can't be used to perform unexpected effects. `_safeMint` is a callback function, the recipient contract may define any arbitrary logic to be executed, including reenterring the initial mint function, thereby bypassing limits defined in the contract code.

## Impact

```solidity
contracts/src/PositionManager.sol

181: function mint( 
        MintParams calldata params_
    ) external override returns (uint256 tokenId_) {
        tokenId_ = _nextId++;

        // revert if the address is not a valid Ajna pool
        if (!_isAjnaPool(params_.pool, params_.poolSubsetHash)) revert NotAjnaPool();

        // record which pool the tokenId was minted in
        poolKey[tokenId_] = params_.pool;

        emit Mint(params_.recipient, params_.pool, tokenId_);

        _safeMint(params_.recipient, tokenId_);
    }
```

## Code Snippet
https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/PositionManager.sol#L181

## Tool used

Manual Review

## Recommendation
Add `nonReentrant` modifier to `mint()` function.