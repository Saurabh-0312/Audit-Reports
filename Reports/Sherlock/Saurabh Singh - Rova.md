
# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Incorrect if check condition in the updateParticipation function for verifying theminTokenAmountPerUser for a user](#H-01)
- ## Medium Risk Findings

# <a id='contest-summary'></a>Contest Summary

### Sponsor: [Sponsor Name]

### Dates: Feb 14th, 2025 - Feb 19th, 2025

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 0

# <a id='H-01'></a>H-01. Incorrect if check condition in the updateParticipation function for verifying theminTokenAmountPerUser for a user

## **High**

Incorrect if check condition in the updateParticipation function for verifying theminTokenAmountPerUser for a user  
## **Summary**  
The updateParticipation function contains an incorrect if conditions to check for theminTokenAmountPerUser and maxTokenAmountPerUser for a user.

## **Vulnerability Details**  
theminTokenAmountPerUser :-  
https://github.com/sherlock-audit/2025-02-rova/blob/main/rova-contracts/src/Launch.sol#L355

@>  
```solidity
if (userTokenAmount - refundCurrencyAmount < settings.minTokenAmountPerUser) {
    revert MinUserTokenAllocationNotReached(
        request.launchGroupId, request.userId, userTokenAmount, request.tokenAmount
    );
}
```

maxTokenAmountPerUser:-  
https://github.com/sherlock-audit/2025-02-rova/blob/main/rova-contracts/src/Launch.sol#L368

```solidity
if (userTokenAmount + additionalCurrencyAmount > settings.maxTokenAmountPerUser) {
    revert MaxUserTokenAllocationReached(
        request.launchGroupId, request.userId, userTokenAmount, request.tokenAmount
    );
}
```

1. The first if condition ensures whether the token amount for a user is greater than the minimum threshold, i.e., settings.minTokenAmountPerUser. If the user has less than that, it reverts. 

But the given condition in the contract is incorrect because it checks `(userTokenAmount - refundCurrencyAmount < settings.minTokenAmountPerUser)`, which is wrong. Subtracting the refundCurrencyAmount from userTokenAmount is incorrect.

userTokenAmount is the amount of tokens the user allocated before the update, but it is incorrectly subtracted by refundCurrencyAmount. refundCurrencyAmount represents the equivalent amount in currency, not the number of tokens to refund.  
Instead of directly subtracting refundCurrencyAmount they should convert it to token amount and then subtract it.

2. Similarly with the 2nd if condition ensures that the userTokenAmount does not exceed upper limit i.e settings.maxTokenAmountPerUser.  
Directly add the additionalCurrencyAmount to user token is incorrect: `userTokenAmount + additionalCurrencyAmount > settings.maxTokenAmountPerUser`.

## **Impact**  
The incorrect condition will revert on the wrong amount and also pass an incorrect token amount, causing contract malfunction.

1. Since refundCurrencyAmount represents the currency amount to refund, not the tokens to be taken back from the user, the condition leads to incorrect validation logic. 

2. Similarly additionalCurrencyAmount represent the currency amount not the token amount.

## **Recommendations**  
1. convert the refundCurrencyAmount to token amount and then subtract.  
2. convert the additionalCurrencyAmount to token amount and then subtract.
