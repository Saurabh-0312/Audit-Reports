# Core Contracts - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect calculation in the `Auction::buy` function leads to an inaccurate cost determination.](#H-01)
    - ### [H-02. "The `_getBaseWeight` function will return incorrect weights."](#H-02)
    - ### [H-03. Incorrect calculations in the `LendingPool` contract for checking whether a user has enough collateral.](#H-03)
    - ### [H-04. The NFT cannot be liquidated in the `NFTLiquidator` contract.](#H-04)
- ## Medium Risk Findings
    - ### [M-01. "The `lastClaimTime` mapping is not updated anywhere in the `FeeCollector` contract."](#M-01)
    - ### [M-02. Incorrect scaling of total supply in the `DebtToken::totalSupply` function.](#M-02)
    - ### [M-03. Incorrect `totalBorrowed` amount is used in the `RAACMinter::getUtilizationRate` function.](#M-03)
- ## Low Risk Findings
    - ### [L-01. Incorrect assignment of `boostState.minBoost` in the `BaseGauge` contract constructor.](#L-01)
    - ### [L-02. A user can vote on cancelled proposal in the `Governance` contract.](#L-02)
    - ### [L-03. The `RToken::mint` function returns an incorrect value.](#L-03)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Regnum Aurum Acquisition Corp

### Dates: Feb 3rd, 2025 - Feb 24th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2025-02-raac)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 4
- Medium: 3
- Low: 3


# High Risk Findings

## <a id='H-01'></a>H-01. Incorrect calculation in the `Auction::buy` function leads to an inaccurate cost determination.            



## Summary

The `Auction::buy` function calculates the cost incorrectly.

## Vulnerability Details

The calculation for cost in the `buy` function does not include decimal precision, leading to an incorrect cost calculation.

the `getPrice()`function will return price of zeno  ( which will let say follows 6 or 8 decimals).

since the zeno are ERC20 tokens they follows 18 decimals. 

the cost = price \* amount;

             = price in 8 decimals \* amount in 18 decimlas

Now the usdc follows 6 decimals , The cost calculated will be way greater than intended, hence user will send more used than needed.

```Solidity
 function buy(uint256 amount) external whenActive {
        require(amount <= state.totalRemaining, "Not enough ZENO remaining");
@>>        uint256 price = getPrice();
@>>        uint256 cost = price * amount;
@>>       require(usdc.transferFrom(msg.sender, businessAddress, cost), "Transfer failed");

        bidAmounts[msg.sender] += amount;
        state.totalRemaining -= amount;
        state.lastBidTime = block.timestamp;
        state.lastBidder = msg.sender;

        zeno.mint(msg.sender, amount);
        emit ZENOPurchased(msg.sender, amount, price);
    }
```

## Proof of Code 

Lets take an example for that :- 

```diff
amount = 2500e18
price = 50e8 // if it has 8 decimals

cost = 2500e18 * 50e8 
     =125000e26
     
the usdc follows 6 decimals
the amount will become 125000e26 which is wrong and very high
```

## Impact

Due to the incorrect cost calculation, the user **spends more USDC than intended** and **receives fewer Zeno tokens** than they should. This happens because the decimal misalignment inflates the cost, leading to an unfair exchange.

## Recommendations

To normalize the **cost** and ensure it's in **6 decimals (USDC precision)**, divide by the appropriate decimal factor.

## <a id='H-02'></a>H-02. "The `_getBaseWeight` function will return incorrect weights."            



## Summary

"An incorrect argument is passed to `getGaugeWeight` to retrieve the user’s weight."

## Vulnerability Details

The `_getBaseWeight` function is used to get the base weight of a user account, but in the implementation below, the contract's address (`this`) is incorrectly passed instead of the user's account address.

```Solidity
function _getBaseWeight(address account) internal view virtual returns (uint256) {
        
        return IGaugeController(controller).getGaugeWeight(address(this)); //- incorrectvalue passed in the getGaugeWeight
    
```

## Impact

"The `getBaseWeight` function will return the weight of `this` address instead of the user's base weight."

## Recommendations

Implementation following change :-

```diff
function _getBaseWeight(address account) internal view virtual returns (uint256) {
        
-        return IGaugeController(controller).getGaugeWeight(address(this));
+        return IGaugeController(controller).getGaugeWeight(account);
    
```

## <a id='H-03'></a>H-03. Incorrect calculations in the `LendingPool` contract for checking whether a user has enough collateral.            



## Summary

The `withdrawNFT` functions have incorrect calculations in the `if` condition for collateral.

## Vulnerability Details

withdrawNFT function :-

In this function, `userDebt.percentMul(liquidationThreshold)` is compared to `collateralValue - nftValue`, which is incorrect.

By withdrawing the NFT, the collateral value decreases. The debt should be lower than `liquidationThreshold` % of the new collateral value after withdrawing the NFT.

```Solidity
@>> if (collateralValue - nftValue < userDebt.percentMul(liquidationThreshold)) {
            revert WithdrawalWouldLeaveUserUnderCollateralized();
        }
```

<https://github.com/Cyfrin/2025-02-raac/blob/89ccb062e2b175374d40d824263a4c0b601bcb7f/contracts/core/pools/LendingPool/LendingPool.sol#L302>

## Proof of code :-

lets take an example 

liquidationThreshold = 80 %

`collateralValue`= 100

`userDebt`=70 

Initially, the user has enough collateral for the debt.

Now follow the calculation for , `nftValue`=40

`collateralValue - nftValue`=60 and`userDebt.percentMul(liquidationThreshold))` =80% \*70 =56

the new colletral amount = 60 , and the debt amount is `userDebt`=70 

60 < 56 will be `false `, but the user has 70 debt and colletral = 60 , it should be `UnderCollateralized`.

## Impact

The user cannot fully utilize the collateral value because `userDebt.percentMul(liquidationThreshold)` will be much lower than the actual `liquidationThreshold` of the collateral value.

The value of debt will be greater than the `liquidationThreshold`, which is incorrect.

## Recommendations

withdrawNFT function :-

```diff
-if (collateralValue - nftValue < userDebt.percentMul(liquidationThreshold)) {
+if ((collateralValue - nftValue).percentMul(liquidationThreshold) < userDebt) {            
            revert WithdrawalWouldLeaveUserUnderCollateralized();
        }
```

## <a id='H-04'></a>H-04. The NFT cannot be liquidated in the `NFTLiquidator` contract.            



## Summary

An NFT cannot be liquidated because of the check `(msg.sender != stabilityPool)` in the `NFTLiquidator` contract.

&#x20;

## Vulnerability Details

The `if` condition checks whether the caller of the `liquidateNFT` function is `stabilityPool` or not. If it is not `stabilityPool`, it will revert. Hence, only the `stabilityPool` can call this function to liquidate NFTs.

But in the `StabilityPool` contract, there is no function call made to the `NFTLiquidator` contract for the `liquidateNFT` function.

Because of the `if` condition, no one other than `stabilityPool` can call this function.

&#x20;

Since `StabilityPool` does not call `liquidateNFT`, the NFT **cannot be liquidated**.

```Solidity
function liquidateNFT(uint256 tokenId, uint256 debt) external {
@>>     if (msg.sender != stabilityPool) revert OnlyStabilityPool();
        
        nftContract.transferFrom(msg.sender, address(this), tokenId);
        
        tokenData[tokenId] = TokenData({
            debt: debt,
            auctionEndTime: block.timestamp + 3 days,
            highestBid: 0,
            highestBidder: address(0)
        });

        indexToken.mint(stabilityPool, debt);

        emit NFTLiquidated(tokenId, debt);
        emit AuctionStarted(tokenId, debt, tokenData[tokenId].auctionEndTime);
    }
```

## Impact

The user under debt cannot liquidate their NFT, nor can the contract liquidate the user's NFT to prevent over-indebtedness.

## Recommendations

Call the `liquidateNFT` function in the `StabilityPool` where it is needed.

or remove the if condition to call the function from other contracts, or directly by user.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. "The `lastClaimTime` mapping is not updated anywhere in the `FeeCollector` contract."            



## Summary

The function `_updateLastClaimTime`, which updates the `lastClaimTime` mapping, is not called anywhere.

## Vulnerability Details

The `_updateLastClaimTime` function updates the `lastClaimTime` mapping and should be called when the user claims rewards by calling the `claimRewards` function.

The `claimRewards` function should call `_updateLastClaimTime` to update the mapping whenever the user claims their reward.

```Solidity
function _updateLastClaimTime(address user) internal {
        lastClaimTime[user] = block.timestamp;
    }
```

```Solidity
function claimRewards(address user) external override nonReentrant whenNotPaused returns (uint256) {
        if (user == address(0)) revert InvalidAddress();
        
        uint256 pendingReward = _calculatePendingRewards(user);
        if (pendingReward == 0) revert InsufficientBalance();
        
        // Reset user rewards before transfer
        userRewards[user] = totalDistributed;
        
        // Transfer rewards
        raacToken.safeTransfer(user, pendingReward);
        
        emit RewardClaimed(user, pendingReward);
        return pendingReward;
    }
```

## Impact

The contract will lose track of users' claim times.

## Recommendations

1. Update the lastClaimTime while user claims the rewards.

```diff
function claimRewards(address user) external override nonReentrant whenNotPaused returns (uint256) {
        if (user == address(0)) revert InvalidAddress();
        
        uint256 pendingReward = _calculatePendingRewards(user);
        if (pendingReward == 0) revert InsufficientBalance();
+      _updateLastClaimTime(user);
        
        // Reset user rewards before transfer
        userRewards[user] = totalDistributed;
        
        // Transfer rewards
        raacToken.safeTransfer(user, pendingReward);
        
        emit RewardClaimed(user, pendingReward);
        return pendingReward;
    }
```

## <a id='M-02'></a>M-02. Incorrect scaling of total supply in the `DebtToken::totalSupply` function.            



## Summary

The `DebtToken::totalSupply` function has incorrect logic for scaling the total supply.

## Vulnerability Details

The `rayDiv` is used to scale the `totalSupply` value, which is incorrect. The `rayMul` should be used instead; otherwise, it will return an incorrect value.

as done in the `balanceOf`function.

```Solidity
function balanceOf(address account) public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledBalance = super.balanceOf(account);
@>>        return scaledBalance.rayMul(ILendingPool(_reservePool).getNormalizedDebt());
    }
```

```Solidity
function totalSupply() public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledSupply = super.totalSupply();
@>>        return scaledSupply.rayDiv(ILendingPool(_reservePool).getNormalizedDebt());
  }
```

## Impact

The `totalSupply` will return an incorrect value if `rayDiv` is used instead of `rayMul`.

## Recommendations

```diff
function totalSupply() public view override(ERC20, IERC20) returns (uint256) {
        uint256 scaledSupply = super.totalSupply();
-       return scaledSupply.rayDiv(ILendingPool(_reservePool).getNormalizedDebt());
+       return scaledSupply.rayMul(ILendingPool(_reservePool).getNormalizedDebt());
  }
```

## <a id='M-03'></a>M-03. Incorrect `totalBorrowed` amount is used in the `RAACMinter::getUtilizationRate` function.            



## Summary

`totalBorrowed` is incorrect because the `getNormalizedDebt` function returns an index value, not totalBorrowed amount.

## Vulnerability Details

The `getNormalizedDebt` function returns an index, not the `totalBorrowed` amount, as implemented in the `LiquidityPool` contract.

Using the index value instead of the `totalBorrowed` amount will cause incorrect calculations in `getUtilizationRate`, as this function should return the utilization of deposits. The index value will lead to incorrect results.

```Solidity
    /**
     * @notice Gets the reserve's normalized debt
@>>     * @return The normalized debt (usage index)
     */
function getNormalizedDebt() external view returns (uint256) {
@>>        return reserve.usageIndex; 
    }
```

```Solidity
function getUtilizationRate() internal view returns (uint256) {
@>>     uint256 totalBorrowed = lendingPool.getNormalizedDebt();//- this is wrong in lending piil
        uint256 totalDeposits = stabilityPool.getTotalDeposits();
        if (totalDeposits == 0) return 0;
        return (totalBorrowed * 100) / totalDeposits;
    }
```

## Impact

The `totalBorrowed` value is incorrect, leading to wrong calculations in `getUtilizationRate`, which will return an incorrect utilization rate.

## Recommendations

Use the correct function to get the `totalBorrowed` value in the `getUtilizationRate` function.

Or modify the `getNormalizedDebt` function to return the `totalBorrowed` amount.


# Low Risk Findings

## <a id='L-01'></a>L-01. Incorrect assignment of `boostState.minBoost` in the `BaseGauge` contract constructor.            



## Summary

The constructor of `BaseGauge` contract assign incorrect value to the `boostState.minBoost`

## Vulnerability Details

`boostState.minBoost` is incorrectly set to `1e18`, which is not valid. The `minBoost` should be **1x or less than** **`maxBoost`**, but in the constructor, it is set far too high.

```Solidity
constructor(
        address _rewardToken,
        address _stakingToken,
        address _controller,
        uint256 _maxEmission,
        uint256 _periodDuration
    ) {
        rewardToken = IERC20(_rewardToken);
        stakingToken = IERC20(_stakingToken);
        controller = _controller;

        // Initialize roles
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(CONTROLLER_ROLE, _controller);

        // Initialize boost parameters
        boostState.maxBoost = 25000; // 2.5x
@>>     boostState.minBoost = 1e18;  // this is incorrect
        
        boostState.boostWindow = 7 days;

        uint256 currentTime = block.timestamp;
        uint256 nextPeriod = ((currentTime / _periodDuration) * _periodDuration) + _periodDuration; //q - why ther are dividing and multiplyuing with same variable.

        // Initialize period state
        periodState.periodStartTime = nextPeriod;
        periodState.emission = _maxEmission;
        TimeWeightedAverage.createPeriod(
            periodState.votingPeriod,
            nextPeriod,
            _periodDuration,
            0,
            10000 // VOTE_PRECISION
        );
    }
```

## Impact

The value of `boostState.minBoost` used in any calculation will cause incorrect results, leading to miscalculations in boost-related logic.

## Recommendations

```diff
constructor(
        address _rewardToken,
        address _stakingToken,
        address _controller,
        uint256 _maxEmission,
        uint256 _periodDuration
    ) {
        rewardToken = IERC20(_rewardToken);
        stakingToken = IERC20(_stakingToken);
        controller = _controller;

        // Initialize roles
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(CONTROLLER_ROLE, _controller);

        // Initialize boost parameters
        boostState.maxBoost = 25000; // 2.5x
-       boostState.minBoost = 1e18;  // this should be 1X
+       boostState.minBoost = 10000; // this should be 1X
        
        boostState.boostWindow = 7 days;

        uint256 currentTime = block.timestamp;
        uint256 nextPeriod = ((currentTime / _periodDuration) * _periodDuration) + _periodDuration; //q - why ther are dividing and multiplyuing with same variable.

        // Initialize period state
        periodState.periodStartTime = nextPeriod;
        periodState.emission = _maxEmission;
        TimeWeightedAverage.createPeriod(
            periodState.votingPeriod,
            nextPeriod,
            _periodDuration,
            0,
            10000 // VOTE_PRECISION
        );
    }
```

## <a id='L-02'></a>L-02. A user can vote on cancelled proposal in the `Governance` contract.            



## Summary

"Users can cast votes on proposals that are canceled."

## Vulnerability Details

In the `Governance::castVote` function, there is no check to prevent users from casting votes on canceled proposals.

Once the proposal is canceled by calling `cancel`, the `canceled` state is set to `true`, as shown below.

```solidity
proposal.canceled = true;
```

But the `Governance::castVote` function does not check this state of the proposal to verify whether it is canceled or not. Hence, a user can cast a vote on canceled proposals.

## Impact

A user can cast votes on canceled proposals since the `Governance::castVote` function lacks a check for canceled proposals.

## Recommendations

```diff
function castVote(uint256 proposalId, bool support) external override returns (uint256) {
        ProposalCore storage proposal = _proposals[proposalId];
        if (proposal.startTime == 0) revert ProposalDoesNotExist(proposalId);
        if (block.timestamp < proposal.startTime) {
            revert VotingNotStarted(proposalId, proposal.startTime, block.timestamp);
        }
        if (block.timestamp > proposal.endTime) {
            revert VotingEnded(proposalId, proposal.endTime, block.timestamp);
        }
+      require(!proposal.cancelled, "proposal cancelled");

        ProposalVote storage proposalVote = _proposalVotes[proposalId];
        if (proposalVote.hasVoted[msg.sender]) {
            revert AlreadyVoted(proposalId, msg.sender, block.timestamp);
        }

        uint256 weight = _veToken.getVotingPower(msg.sender);
        if (weight == 0) {
            revert NoVotingPower(msg.sender, block.number);
        }

        proposalVote.hasVoted[msg.sender] = true;

        if (support) {
            proposalVote.forVotes += weight;
        } else {
            proposalVote.againstVotes += weight;
        }

        emit VoteCast(msg.sender, proposalId, support, weight, "");
        return weight;
    }
```

## <a id='L-03'></a>L-03. The `RToken::mint` function returns an incorrect value.            



## Summary

The `RToken::mint` function return incorrect value when `amountToMint == 0`

## Vulnerability Details

As stated in the comments above, the function's third return parameter should be:

**"The new total supply after minting."**

If no amount is minted, the total supply **won't** become zero as returned in the function; it will remain `totalSupply()`.

``````Solidity
  * @return A tuple containing:
     *         - bool: True if this is the first mint for the recipient, false otherwise
     *         - uint256: The amount of scaled tokens minted
@>   *         - uint256: The new total supply after minting
     *         - uint256: The amount of underlying tokens minted
     */
    function mint(
        address caller,
        address onBehalfOf,
        uint256 amountToMint,
        uint256 index
    ) external override onlyReservePool returns (bool, uint256, uint256, uint256) {
@>>        if (amountToMint == 0) {
@>>            return (false, 0, 0, 0);
          
          `````
          `````
          `````
    }      
``````

## Impact

The return value used in any function will be inconsistent.

## Recommendations

``````diff
  * @return A tuple containing:
     *         - bool: True if this is the first mint for the recipient, false otherwise
     *         - uint256: The amount of scaled tokens minted
     *         - uint256: The new total supply after minting
     *         - uint256: The amount of underlying tokens minted
     */
    function mint(
        address caller,
        address onBehalfOf,
        uint256 amountToMint,
        uint256 index
    ) external override onlyReservePool returns (bool, uint256, uint256, uint256) {
          if (amountToMint == 0) {
-            return (false, 0, 0, 0);
+            return (false, 0, totalSupply(), 0);
          
          `````
          `````
          `````
    }   
``````



