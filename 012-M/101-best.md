hyh

medium

# Flashloan end result isn't controlled

## Summary

FlashLoan logic do not control the end result of transferring tokens out and back in. Given that protocol aims to support arbitrary non-rebasing / without fee on transfer / decimals in `[1, 18]` fungible tokens to be quote and collateral of a pool, this includes any exotic types of behavior, for example, reporting a successful transfer, but not performing internal accounting update for any reason.

As an example, this can be a kind of wide blacklisting mechanics introduction (i.e. allow this white list of accounts, freeze everyone else type of logic).

## Vulnerability Detail

Now there is no control of the resulting balance, and any token that successfully performs safeTransfer, but for any reason withholds an update of token internal accounting, can successfully steal the whole pool's balance of any Ajna pool. This can be initiated by an attacker unrelated to token itself as griefing.

As many core token contracts are upgradable (USDC, USDT and so forth), such behaviour can be not in place right now, but can be introduced in the future.

## Impact

Some fungible tokens that qualify for Ajna pools (including not imposing any fee on transfers) may not return the whole amount back, but will report successful safeTransfer(), i.e. up to the whole balance of Ajna pool for such ERC20 token can be stolen.

This can take place in a situation when a popular token was upgraded and the consequences of the internal logic change weren't fully understood by wide market initially and most depositors remained in the corresponding Ajna pool, then someone calls a flash loan as a griefing attack that will result in the token freezing the balancer of the pool. Or it was understood, but the griefer was quicker.

As the probability of such internal mechanics introduction is low, but the impact is up to full loss of user's funds, setting the severity to be medium.

## Code Snippet

Flash loan functions do not employ any checks after ERC20 token was received back:

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/ERC20Pool.sol#L236-L256

```solidity
    /// @inheritdoc FlashloanablePool
    function flashLoan(
        IERC3156FlashBorrower receiver_,
        address token_,
        uint256 amount_,
        bytes calldata data_
    ) external override(IERC3156FlashLender, FlashloanablePool) nonReentrant returns (bool) {
        if (token_ == _getArgAddress(QUOTE_ADDRESS)) return _flashLoanQuoteToken(receiver_, token_, amount_, data_);

        if (token_ == _getArgAddress(COLLATERAL_ADDRESS)) {
            _transferCollateral(address(receiver_), amount_);

            if (receiver_.onFlashLoan(msg.sender, token_, amount_, 0, data_) !=
                keccak256("ERC3156FlashBorrower.onFlashLoan")) revert FlashloanCallbackFailed();

            _transferCollateralFrom(address(receiver_), amount_);
            return true;
        }

        revert FlashloanUnavailableForToken();
    }
```

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/base/FlashloanablePool.sol#L33-L45

```solidity
    function _flashLoanQuoteToken(IERC3156FlashBorrower receiver_,
        address token_,
        uint256 amount_,
        bytes calldata data_
    ) internal returns (bool) {
        _transferQuoteToken(address(receiver_), amount_);
        
        if (receiver_.onFlashLoan(msg.sender, token_, amount_, 0, data_) != 
            keccak256("ERC3156FlashBorrower.onFlashLoan")) revert FlashloanCallbackFailed();

        _transferQuoteTokenFrom(address(receiver_), amount_);
        return true;
    }
```

Flash loan safety is now controlled by safeTransfer() only, which internal mechanics can vary between ERC20 tokens:

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/base/Pool.sol#L502-L508

```solidity
    function _transferQuoteTokenFrom(address from_, uint256 amount_) internal {
        IERC20(_getArgAddress(QUOTE_ADDRESS)).safeTransferFrom(from_, address(this), amount_ / _getArgUint256(QUOTE_SCALE));
    }

    function _transferQuoteToken(address to_, uint256 amount_) internal {
        IERC20(_getArgAddress(QUOTE_ADDRESS)).safeTransfer(to_, amount_ / _getArgUint256(QUOTE_SCALE));
    }
```

## Tool used

Manual Review

## Recommendation

Consider adding a balance control check to ensure that flash loan invariant remains: record contract balance before `receiver_.onFlashLoan(...)` callback and record it after `_transferQuoteTokenFrom(address(receiver_), amount_)`, require that resulting token balance ends up being not less than initial.

This applies both to ERC20Pool's flashLoan() and FlashloanablePool's _flashLoanQuoteToken().