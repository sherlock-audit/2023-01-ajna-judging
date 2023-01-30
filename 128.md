ck

high

# `_bucketCollateralDust` calculation is not normalized

## Summary

`_bucketCollateralDust` calculation for tokens with low decimals e.g `USDC` results in an inflated dust amount. For tokens with high decimals the dust amount becomes very low. 

## Vulnerability Detail

`_bucketCollateralDust` is calculated as:

```solidity
    function _bucketCollateralDust(uint256 bucketIndex) internal pure returns (uint256) {
        // price precision adjustment will always be 0 for encumbered collateral
        uint256 pricePrecisionAdjustment = _getCollateralDustPricePrecisionAdjustment(bucketIndex);
        // difference between the normalized scale and the collateral token's scale
        return Maths.max(_getArgUint256(COLLATERAL_SCALE), 10 ** pricePrecisionAdjustment);
    } 
 ```

Let's take two tokens `USDC` and `DAI`

`COLLATERAL_SCALE` for `USDC` will be `10**(18 - 6) = 10**12)`
`COLLATERAL_SCALE` for `DAI` will be `10**(18 - 18) = 10**0 = 1)`

Taking into account their decimals (USDC - 6) and (DAI - 18), it can be seen the collateral dust for `DAI` will be very low while that of `USDC` will be very high.

To add `USDC` as collateral a minimum of `1,000,000` will be needed i.e `1,000,000 * 10**6 = 10**12` while it will be very low for `DAI`.
 
Very low dust amounts will also lead to possible `HTP` manipulation.

## Impact

Collateral dust is used as a check in multiple functions and this will break intended functionality and could lead to loss of funds in cases where the minimum collateral dust required is either too low or too high.

## Code Snippet

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/ERC20Pool.sol#L524-L529

## Tool used

Manual Review

## Recommendation

The `_bucketCollateralDust` should use the unscaled token's decimals in the function rather than the `COLLATERAL_SCALE`. 