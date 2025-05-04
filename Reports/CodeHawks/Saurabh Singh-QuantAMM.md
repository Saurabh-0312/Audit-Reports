# QuantAMM - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. A malicious owner can alter the fee amount paid by updating the value of `lpTokenDepositValue` in the `afterUpdate` function of the `UpliftOnlyExample` contract.](#H-01)
- ## Medium Risk Findings
    - ### [M-01. The `updateWeightRunner` contract has two pairs of functions with the same implementation and functionality. [`setQuantAMMSwapFeeTake` -- `setQuantAMMUpliftFeeTake`] and [`getQuantAMMSwapFeeTake` -- `getQuantAMMUpliftFeeTake`]](#M-01)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: QuantAMM

### Dates: Dec 20th, 2024 - Jan 15th, 2025

[See more contest details here](https://codehawks.cyfrin.io/c/2024-12-quantamm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 1
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. A malicious owner can alter the fee amount paid by updating the value of `lpTokenDepositValue` in the `afterUpdate` function of the `UpliftOnlyExample` contract.            



## Summary

A user can change the amount of fee they pay by modifying the value of `lpTokenDepositValue` in the `afterUpdate` function. Altering the `lpTokenDepositValue` will consequently change the fee.

## Vulnerability Details

The bug occurs when the owner of the NFT transfers it to another account, but effectively to themselves.

When the ower transfer the NFT, the value of `lpTokenDepositValue`is changed to current value.

<https://github.com/Cyfrin/2024-12-quantamm/blob/a775db4273eb36e7b4536c5b60207c9f17541b92/pkg/pool-hooks/contracts/hooks-quantamm/UpliftOnlyExample.sol#L609>

```Solidity
 if (tokenIdIndexFound) {
            if (_to != address(0)) {
                // Update the deposit value to the current value of the pool in base currency (e.g. USD) and the block index to the current block number
                //vault.transferLPTokens(_from, _to, feeDataArray[i].amount);
@>>                feeDataArray[tokenIdIndex].lpTokenDepositValue = lpTokenDepositValueNow;
                feeDataArray[tokenIdIndex].blockTimestampDeposit = uint32(block.number);
                feeDataArray[tokenIdIndex].upliftFeeBps = upliftFeeBps;

```



&#x20;By checking  the value of `lpTokenDepositValue` , the owner can transfer the NFT to themself when the price is highest, or any price benifiting the owner causing them to pay the minimum fee or a significantly reduced fee.

calculation of fee in the \`onAfterRemoveLiquidity\` :-

<https://github.com/Cyfrin/2024-12-quantamm/blob/a775db4273eb36e7b4536c5b60207c9f17541b92/pkg/pool-hooks/contracts/hooks-quantamm/UpliftOnlyExample.sol#L474>

```Solidity
localData.lpTokenDepositValue = feeDataArray[i].lpTokenDepositValue;

            localData.lpTokenDepositValueChange =
@>>                (int256(localData.lpTokenDepositValueNow) - int256(localData.lpTokenDepositValue)) /
                int256(localData.lpTokenDepositValue);

            uint256 feePerLP;
            // if the pool has increased in value since the deposit, the fee is calculated based on the deposit value
@>>            if (localData.lpTokenDepositValueChange > 0) {
                feePerLP =
                    (uint256(localData.lpTokenDepositValueChange) * (uint256(feeDataArray[i].upliftFeeBps) * 1e18)) /
                    10000;
            }
            // if the pool has decreased in value since the deposit, the fee is calculated based on the base value - see wp
            else {
                //in most cases this should be a normal swap fee amount.
                //there always myst be at least the swap fee amount to avoid deposit/withdraw attack surgace.
                feePerLP = (uint256(minWithdrawalFeeBps) * 1e18) / 10000;
            }
```

## Impact

The owner of an NFT can manipulate the fee they pay significantly. If the owner sends the NFT to themselves at a very high LP token value, and the value drops slightly, they will only be charged the minimum fee.

## Tools Used

Manual review

## Recommendations

In my opinion:

&#x20;

1. Avoid updating the value of `lpTokenDepositValue`.
2. Alternatively, calculate the fee using the old value of `lpTokenDepositValue` to prevent manipulation.

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. The `updateWeightRunner` contract has two pairs of functions with the same implementation and functionality. [`setQuantAMMSwapFeeTake` -- `setQuantAMMUpliftFeeTake`] and [`getQuantAMMSwapFeeTake` -- `getQuantAMMUpliftFeeTake`]            



## Summary

The `updateWeightRunner` contract has two pairs of functions ->

 \[`setQuantAMMSwapFeeTake` -- `setQuantAMMUpliftFeeTake`]  and&#x20;

\[`getQuantAMMSwapFeeTake` -- `getQuantAMMUpliftFeeTake`]

with exact same functionality.

## Vulnerability Details

Pair 1 \[`setQuantAMMSwapFeeTake` -- `setQuantAMMUpliftFeeTake`]  \
have same implementation => They both update the same `quantAMMSwapFeeTake`variable but the name of function and the event emitted by them is different.



```Solidity
@>pair 1 function setQuantAMMSwapFeeTake(uint256 _quantAMMSwapFeeTake) external override {
        require(msg.sender == quantammAdmin, "ONLYADMIN");
        require(_quantAMMSwapFeeTake <= 1e18, "Swap fee must be less than 100%");
        uint256 oldSwapFee = quantAMMSwapFeeTake;
        quantAMMSwapFeeTake = _quantAMMSwapFeeTake;

        emit SwapFeeTakeSet(oldSwapFee, _quantAMMSwapFeeTake);
    }
    
    /// @notice Set the quantAMM uplift fee % amount allocated to the protocol for running costs
    /// @param _quantAMMUpliftFeeTake The new uplift fee % amount allocated to the protocol for running costs
 @>pair 1   function setQuantAMMUpliftFeeTake(uint256 _quantAMMUpliftFeeTake) external{
        require(msg.sender == quantammAdmin, "ONLYADMIN");
        require(_quantAMMUpliftFeeTake <= 1e18, "Uplift fee must be less than 100%");
        uint256 oldSwapFee = quantAMMSwapFeeTake;
        quantAMMSwapFeeTake = _quantAMMUpliftFeeTake;

        emit UpliftFeeTakeSet(oldSwapFee, _quantAMMUpliftFeeTake);
    }
```



pair 2 \[`getQuantAMMSwapFeeTake` -- `getQuantAMMUpliftFeeTake`] 

have different name but return same variable `quantAMMSwapFeeTake`.

```Solidity
@>pair 2   function getQuantAMMSwapFeeTake() external view override returns (uint256) {
        return quantAMMSwapFeeTake;
    }


/// @notice Get the quantAMM uplift fee % amount allocated to the protocol for running costs
@>pair 2    function getQuantAMMUpliftFeeTake() external view returns (uint256){
        return quantAMMSwapFeeTake;
    }
```

## Impact

1. If the other function in each pair performs different work, the impact will be critical, as a major functionality could be missing from the contract.
2. If both functions perform the same work with no additional functionality, it can still confuse the user or caller, and cause ambiguity with the emitted events, leading to potential issues in understanding the contract's behavior.

## Tools Used

manual review

## Recommendations

1. If the function in each pair is redundant, remove one of the functions from each pair to reduce unnecessary complexity and avoid confusion.
2. If both functions in a pair have different work, implement the missing functionality in the other function as needed, ensuring both functions contribute to the intended behavior of the contract without redundancy.





