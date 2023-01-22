ctf_sec

medium

# ERC20PoolFactory and ERC721PoolFactory deployment is vulnerable to front-running with sub-optimal interest rate

## Summary

ERC20Pool and ERC721Pool deployment is vulnerable to front-running

## Vulnerability Detail

Below is the current implementation of the ERC20PoolFactory deployment code:

```solidity
function deployPool(
	address collateral_, address quote_, uint256 interestRate_
) external canDeploy(ERC20_NON_SUBSET_HASH, collateral_, quote_, interestRate_) returns (address pool_) {
	uint256 quoteTokenScale = 10 ** (18 - IERC20Token(quote_).decimals());
	uint256 collateralScale = 10 ** (18 - IERC20Token(collateral_).decimals());

	bytes memory data = abi.encodePacked(
		PoolType.ERC20,
		ajna,
		collateral_,
		quote_,
		quoteTokenScale,
		collateralScale
	);

	ERC20Pool pool = ERC20Pool(address(implementation).clone(data));

	pool_ = address(pool);

	// Track the newly deployed pool
	deployedPools[ERC20_NON_SUBSET_HASH][collateral_][quote_] = pool_;
	deployedPoolsList.push(pool_);

	emit PoolCreated(pool_);

	pool.initialize(interestRate_);
}
```

There could be only one pool for collateral + quote token.

Such validation is done the canDeploy modifier

```solidity
/**
 * @notice Ensures that pools are deployed according to specifications.
 * @dev    Used by both ERC20, and ERC721 pool factory types.
 */
modifier canDeploy(bytes32 subsetHash_, address collateral_, address quote_, uint256 interestRate_) {
	if (collateral_ == address(0) || quote_ == address(0))              revert IPoolFactory.DeployWithZeroAddress();
	if (deployedPools[subsetHash_][collateral_][quote_] != address(0)) revert IPoolFactory.PoolAlreadyExists();
	if (MIN_RATE >= interestRate_ || interestRate_ >= MAX_RATE)         revert IPoolFactory.PoolInterestRateInvalid();
	_;
}
```

if the collateral address + quote address pool is already deployed, the deployment will revert:

```solidity
	if (deployedPools[subsetHash_][collateral_][quote_] != address(0)) revert IPoolFactory.PoolAlreadyExists();
```

When the pool is initially deployed, the initial interest rate is set:

```solidity
pool.initialize(interestRate_);
```

However, the intererestRate is not related to the encoded data, this open door for frontrunning.

```solidity
bytes memory data = abi.encodePacked(
	PoolType.ERC20,
	ajna,
	collateral_,
	quote_,
	quoteTokenScale,
	collateralScale
);

ERC20Pool pool = ERC20Pool(address(implementation).clone(data));
```

## Impact

According to the on-chain context, the protocol wants to deploy the code to multi-chain, let us assume the blockchain deployed is ethereum

```solidity
DEPLOYMENT: Ethereum mainnet, Arbitrum, Optimism, Binance Smart Chain, Polygon, Fantom, Tron, Avalanche
ERC20:  any - ERC20's are used in fungible, collection and subset pool types
ERC721: any - ERC721's are used in collection and subset pool types
ERC777: none
FEE-ON-TRANSFER: none
REBASING TOKENS: none
ADMIN: N/A
```

First, lender A wants to create a Pending pool for ETH / DAI pair, lender can supply DAI as quote token and borrower can use ETH as collateral to borrower DAI.

The lender A submit a transaction set the interest to 7%.

A borrower monitor the mempool and detect the transaction, he front-run the lender's pool creation transaction and creating the same ETH / DAI lendg pool, but setting the interest rate to as low as 1%.

The borrower's transaction go through, the lender's transaction revert.

The lender has to add quoted token in a sub-optimal interest rate and the fund generates less interest because the borrower's front-running transaction of the pool creation.

Same front-running issue exists in ERC721PoolFactory because the deployment process is the same.

## Code Snippet

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/ERC20PoolFactory.sol#L49-L76

https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/ERC721PoolFactory.sol#L90-L103

## Tool used

Manual Review

## Recommendation

We recommend the protocol encode the initial interest rate into the data when cloning the contract, also, can verify the signature of the deployer match the msg.sender and let the first deployer add some token to a given bucket.


