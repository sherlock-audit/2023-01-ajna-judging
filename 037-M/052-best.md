ctf_sec

high

# Hardcoded AJNA token address is not compatible with the multi-chain setting

## Summary

hardcoded AJNA token address is not compatible with the multi-chain setting.

## Vulnerability Detail

According to the on-chain context of the protocol, the contract plans to deploy the protocol to multi-chain context.

```solidity
DEPLOYMENT: Ethereum mainnet, Arbitrum, Optimism, Binance Smart Chain, Polygon, Fantom, Tron, Avalanche
ERC20:  any - ERC20's are used in fungible, collection and subset pool types
ERC721: any - ERC721's are used in collection and subset pool types
ERC777: none
FEE-ON-TRANSFER: none
REBASING TOKENS: none
ADMIN: N/A
```

However, the hardcoded AJNA token address is not compatible with the multi-chain setting.

Note the implementation for RewardsManager.sol, the ajnaToken is not hardcoded is feed into the constructor.

```solidity
constructor(address ajnaToken_, IPositionManager positionManager_) {
	ajnaToken = ajnaToken_;
	positionManager = positionManager_;
}
```

However, AJNA token address is hardcoded in the grant governance contract.

The Grant Contract inherits from StandardFunding.sol and ExtraordinaryFunding.sol

```solidity
contract GrantFund is IGrantFund, ExtraordinaryFunding, StandardFunding {
```

and both StandardFunding and ExtraordinaryFunding inherits from the funding contract:

```solidity
abstract contract StandardFunding is Funding, IStandardFunding {
```

and

```solidity
abstract contract ExtraordinaryFunding is Funding, IExtraordinaryFunding {
```

In the funding contract, the AJNA token is hardcoded to the token address in ethereum mainnet:

```solidity
// address of the ajna token used in grant coordination
address public ajnaTokenAddress = 0x9a96ec9B57Fb64FbC60B423d1f4da7691Bd35079;
```

## Impact

While the contract is the correct AJNA token in ethereum mainnet

https://etherscan.io/address/0x9a96ec9B57Fb64FbC60B423d1f4da7691Bd35079

The contract does not exists in other blockchain:

https://blockscan.com/address/0x9a96ec9B57Fb64FbC60B423d1f4da7691Bd35079

for example:

https://bscscan.com/address/0x9a96ec9B57Fb64FbC60B423d1f4da7691Bd35079

This hardcoded address is used when validating the calldata:

```solidity
function _validateCallDatas(address[] memory targets_,
	uint256[] memory values_,
	bytes[] memory calldatas_) internal view returns (uint256 tokensRequested_) {

	for (uint256 i = 0; i < targets_.length;) {

		// check  targets and values are valid
		if (targets_[i] != ajnaTokenAddress) revert InvalidTarget();
		if (values_[i] != 0) revert InvalidValues();
```

If the protocol choose to deploy the AJNA token to other blockchain, the _validateCallData will not work because if code only allows that the proposal sets the target address to an non-existing address, the proposal such as AJNA token granting and transfer will not work.

## Code Snippet

https://github.com/sherlock-audit/2023-01-ajna/blob/main/ecosystem-coordination/src/grants/base/Funding.sol#L61

https://github.com/sherlock-audit/2023-01-ajna/blob/main/ecosystem-coordination/src/grants/base/Funding.sol#L98-l111

## Tool used

Manual Review

## Recommendation

We recommend the protocol set the AJNA token to the constructor of the grant governance contract instead of hardcode the token address.
