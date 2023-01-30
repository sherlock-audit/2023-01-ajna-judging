hyh

high

# removeCollateral miss bankrupcy logic and can make future LPs sharing losses with the current ones

## Summary

LenderActions' removeCollateral() do not checks for bucket solvency after it has removed a collateral from there. This can lead to losses for future depositors of the bucket.

## Vulnerability Detail

Bankrupcy check logic now exist in all asset removing functions. That prevent a situation when a bucket defaults, but next LP deposit makes in solvent again and next LP shared losses with the old ones this way without having such intent.

For example, mergeOrRemoveCollateral() calls _removeMaxCollateral() that do check affected bucket for bankrupcy. removeCollateral() do not check for that despite insolvency situation for a bucket can occur after collateral was removed.

## Impact

When bucket defaults, but no bankrupcy is checked and no such flag is set, the next LP depositors have to bail out previous, i.e. have to share their losses.

That's a loss for next LPs by unconditional transfer from them to the previous ones.

As removeCollateral() is a part of base functionality that to be used frequently and bucket defaults can routinely happen, so there is no low probability prerequisites, and given the loss for future bucket depositors, setting the severity to be high.

## Code Snippet

There is no bucket bankrupcy logic in removeCollateral(), i.e. when there is no quote tokens in the bucket, `lpAmount_ < bucketLPs`, but `bucketCollateral <= collateralAmount_`, bucket de facto defaults, but no such flag is set:

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/libraries/external/LenderActions.sol#L379-L414

```solidity
    function removeCollateral(
        ...
    ) external returns (uint256 lpAmount_) {
        ...

        Lender storage lender = bucket.lenders[msg.sender];

        uint256 lenderLpBalance;
        if (bucket.bankruptcyTime < lender.depositTime) lenderLpBalance = lender.lps;
        if (lenderLpBalance == 0 || lpAmount_ > lenderLpBalance) revert InsufficientLPs();

        // update lender LPs balance
        lender.lps -= lpAmount_;

        // update bucket LPs and collateral balance
        bucket.lps        -= Maths.min(bucketLPs, lpAmount_);
        bucket.collateral -= Maths.min(bucketCollateral, amount_);
    }
```

The check is present in _removeMaxCollateral():

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/libraries/external/LenderActions.sol#L619-L630

```solidity
        // update bucket LPs and collateral balance
        bucketLPs         -= Maths.min(bucketLPs, lpAmount_);
        bucketCollateral  -= Maths.min(bucketCollateral, collateralAmount_);
        bucket.collateral  = bucketCollateral;
        if (bucketCollateral == 0 && bucketDeposit == 0 && bucketLPs != 0) {
            emit BucketBankruptcy(index_, bucketLPs);
            bucket.lps            = 0;
            bucket.bankruptcyTime = block.timestamp;
        } else {
            bucket.lps = bucketLPs;
        }
    }
```

And removeQuoteToken():

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/libraries/external/LenderActions.sol#L310-L368

```solidity
    function removeQuoteToken(
        mapping(uint256 => Bucket) storage buckets_,
        DepositsState storage deposits_,
        PoolState calldata poolState_,
        RemoveQuoteParams calldata params_
    ) external returns (uint256 removedAmount_, uint256 redeemedLPs_, uint256 lup_) {
        ...

        // update lender and bucket LPs balances
        lender.lps -= redeemedLPs_;

        uint256 lpsRemaining = removeParams.bucketLPs - redeemedLPs_;

        if (removeParams.bucketCollateral == 0 && unscaledRemaining == 0 && lpsRemaining != 0) {
            emit BucketBankruptcy(params_.index, lpsRemaining);
            bucket.lps            = 0;
            bucket.bankruptcyTime = block.timestamp;
        } else {
            bucket.lps = lpsRemaining;
        }

        emit RemoveQuoteToken(msg.sender, params_.index, removedAmount_, redeemedLPs_, lup_);
    }
```

## Tool used

Manual Review

## Recommendation

Consider adding the bankruptcy check similarly to other asset removal functions:

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/libraries/external/LenderActions.sol#L379-L414

```solidity
    function removeCollateral(
        ...
    ) external returns (uint256 lpAmount_) {
        ...

        Lender storage lender = bucket.lenders[msg.sender];

        uint256 lenderLpBalance;
        if (bucket.bankruptcyTime < lender.depositTime) lenderLpBalance = lender.lps;
        if (lenderLpBalance == 0 || lpAmount_ > lenderLpBalance) revert InsufficientLPs();

        // update lender LPs balance
        lender.lps -= lpAmount_;

        // update bucket LPs and collateral balance
-       bucket.lps        -= Maths.min(bucketLPs, lpAmount_);
-       bucket.collateral -= Maths.min(bucketCollateral, amount_);
+       uint256 bucketLPs = bucket.lps - Maths.min(bucketLPs, lpAmount_);
+       uint256 bucketCollateral = bucket.collateral - Maths.min(bucketCollateral, amount_);
+       uint256 bucketDeposit = Deposits.valueAt(deposits_, index_);

+       if (bucketCollateral == 0 && bucketDeposit == 0 && bucketLPs != 0) {
+           emit BucketBankruptcy(index_, bucketLPs);
+           bucket.lps = 0;
+           bucket.bankruptcyTime = block.timestamp;
+       } else {
+           bucket.lps = bucketLPs;
+       }
+       bucket.collateral = bucketCollateral;
+   }
```