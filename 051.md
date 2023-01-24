ck

high

# Division before multiplication will amplify rounding down issues

## Summary

The `_roundToScale` does a division before multplication operation which will ampilfy solidity's default rounding down behavior.

## Vulnerability Detail

The `_roundToScale` calculates the `scaledAmount_` as follows:

```solidity
    function _roundToScale(
        uint256 amount_,
        uint256 tokenScale_
    ) pure returns (uint256 scaledAmount_) {
        scaledAmount_ = (amount_ / tokenScale_) * tokenScale_;
    }
```

Solidity rounds down by default and a division before multiplication will amplify the effect.

`_roundToScale` is used severally in `ERC20Pool` to perform calculations of various amounts. 

Let's take the example below to show how the rounding errors could be amplified:

```solidity
uint256 a = 1.1e18
uint256 b = 1e18
```

Multiplication before division - `(a * b) / b = 1.1e18`
Division before multiplication - `(a / b) * b = 1e18`

As can be seen the effect is quite considerable.
 
## Impact

This could lead to accounting errors and loss of funds e.g in the calculation: `maxQuoteTokenAmountToRepay_ = _roundToScale(maxQuoteTokenAmountToRepay_, _getArgUint256(QUOTE_SCALE));`

## Code Snippet

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/libraries/helpers/PoolHelper.sol#L234-L239

## Tool used

Manual Review

## Recommendation

Perform the division after the multiplication:

```solidity
 scaledAmount_ = (amount_ * tokenScale_) / tokenScale_;
```