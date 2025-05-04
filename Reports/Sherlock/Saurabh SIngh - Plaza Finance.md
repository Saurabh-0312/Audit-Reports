# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. The fee calculations in the pool contract are incorrect, with different total fee amounts calculated during user creation/redeem and when the feeBeneficiary claims.](#H-01)
- ## Medium Risk Findings
  - ### [M-01. An attacker can intentionally prevent the auction from succeeding by placing a bid that makes the totalSellReserveAmount exceed the sale limit just before bidding ends.](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Plaze Finance

### Dates: Jan 14th, 2025 - Jan 23rd, 2025

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 1
- Low: 0

# <a id='H-01'></a>H-01. The fee calculations in the pool contract are incorrect, with different total fee amounts calculated during user creation/redeem and when the feeBeneficiary claims.

The fee calculations in the pool contract are incorrect, with different total fee amounts calculated during user creation/redeem and when the feeBeneficiary claims.

## **Summary**

The fee calculated during create and redeem differs from the fee transferred to the feeBeneficiary due to the use of block.timestamp. Different amounts are deducted while create/redeem, and a separate amount is transferred to the feeBeneficiary.

## **Vulnerability Details**

The fee calculation in the simulateCreate and simulateRedeem is given below ->  
https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Pool.sol#L433

The fee is time based means more time passes more will be the fee amount is deducated from the poolReserves.  
Same is implemented in the getFeeAmount which is called from claimFees function in pool contract ->  
https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Pool.sol#L719

When a user creates or redeems tokens, the fee is deducted from the poolReserves. However, when the feeBeneficiary calls claimFees, a different fee amount is calculated due to the use of block.timestamp. This discrepancy results in an incorrect amount of poolReserves being distributed to the feeBeneficiary, differing from the amount calculated during the user's action.

## **Impact**

A different fee is transferred to the feeBeneficiary than the one calculated during the user's create/redeem due to the use of block.timestamp.  
If block.timestamp becomes large enough that after sending the reserve amount to the Auction, the remaining balance in the pool is less than the fee, the feeBeneficiary will be unable to withdraw the fee.

## **Recommendations**

1. Calculate the fee based on time when the user creates or redeems, but do not recalculate the fee when the feeBeneficiary wants to withdraw.
2. Store the fee calculated when the user creates or redeems in a variable, and then send the stored amount to the feeBeneficiary.

# <a id='M-01'></a>M-01. An attacker can intentionally prevent the auction from succeeding by placing a bid that makes the totalSellReserveAmount exceed the sale limit just before bidding ends.

An attacker can intentionally prevent the auction from succeeding by placing a bid that makes the totalSellReserveAmount exceed the sale limit just before bidding ends.

## **Summary**

A malicious bidder can intentionally bid an amount that causes the totalSellReserveAmount to exceed the sale limit, leading to the auction failure (FAILED_POOL_SALE_LIMIT).

## **Vulnerability Details**

An attacker can intentionally prevent the auction from succeeding by placing a bid that causes the totalSellReserveAmount to exceed the pool's sale limit just before the auction ends.

There is no check in the bidding function to prevent users from placing a bid that causes the totalSellReserveAmount to exceed the pool's sale limit. An attacker could exploit this by bidding just before the auction ends, causing it to enter the FAILED_POOL_SALE_LIMIT state every time.

https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Auction.sol#L336

```solidity
function endAuction() external auctionExpired whenNotPaused {
        if (state != State.BIDDING) revert AuctionAlreadyEnded();

        if (currentCouponAmount < totalBuyCouponAmount) {
            state = State.FAILED_UNDERSOLD;
        } else if (totalSellReserveAmount >= (IERC20(sellReserveToken).balanceOf(pool) * poolSaleLimit) / 100) {
            state = State.FAILED_POOL_SALE_LIMIT;
        } else {
            state = State.SUCCEEDED;
            Pool(pool).transferReserveToAuction(totalSellReserveAmount);
            IERC20(buyCouponToken).safeTransfer(beneficiary, IERC20(buyCouponToken).balanceOf(address(this)));
        }

        emit AuctionEnded(state, totalSellReserveAmount, totalBuyCouponAmount);
    }
```

## **Impact**

Preventing the auction from succeeding each time.

## **Recommendations**

Place a check in the bidding function or in the removeExcessBids function to ensure that the bid, when added to the totalSellReserveAmount, does not exceed the pool's sale limit. This will prevent an attacker from exploiting the vulnerability.
