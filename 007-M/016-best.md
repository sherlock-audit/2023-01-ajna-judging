MalfurionWhitehat

medium

# Interest rate increase/decrease algorithm is not reciprocal

## Summary

The interest rate increase/decrease algorithm is not reciprocal, as it considers a 10% increase equal to multiplying by 1.1, and a 10% decrease equal to [multiplying by 0.9](https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/libraries/external/PoolCommons.sol#L113).

## Vulnerability Detail

An increase followed by a decrease does not get the interest back to the same value. This happens because increase and decrease are not reciprocal. A 10% increase is indeed equal to multiplying by 1.1, but a 10% decrease is equal to dividing by 1.1 (which is the same as multiplying by 0.909090...) and not 0.9.

## Impact

An increase followed by a decrease does not get the interest back to the same value. 
Decreasing yields a value _lower_ than expected (multiplies by 0.9 instead of multiplying by 0.909090...).
As a result, the interest rate algorithm shrinks faster than it grows.

## Code Snippet


```solidity
        // raise rates if 4*(tu-1.02*mau) < (tu+1.02*mau-1)^2-1
        if (4 * (tu - mau102) < ((tu + mau102 - 1e18) ** 2) / 1e18 - 1e18) {
            newInterestRate = Maths.wmul(poolState_.rate, INCREASE_COEFFICIENT);
        }
        // decrease rates if 4*(tu-mau) > 1-(tu+mau-1)^2
        else if (4 * (tu - mau) > 1e18 - ((tu + mau - 1e18) ** 2) / 1e18) {
            newInterestRate = Maths.wmul(poolState_.rate, DECREASE_COEFFICIENT);
        }
```

## Tool used

Manual Review

## Recommendation

Divide by `INCREASE_COEFFICIENT` when decreasing rates.