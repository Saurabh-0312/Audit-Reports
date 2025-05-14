# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. There is a calculation error in the `SecondSwap_StepVesting::transferVesting` contract that allows the grantor to claim more tokens than the amount vested to them.](#H-01)
- ## Medium Risk Findings
  - ### [M-01. A user can list its vesting for sell more than the `maxSellPercentage` in the `SecondSwap_VestingManager::listVesting` contract.](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: SecondSwap

### Dates: Dec 10th, 2024 - Dec 20th, 2024

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 1
- Low: 0

# <a id='H-01'></a>H-01. There is a calculation error in the `SecondSwap_StepVesting::transferVesting` contract that allows the grantor to claim more tokens than the amount vested to them.

## **Finding description and impact**

### Description

The vulnerability arises when a user claims a certain number of steps and then transfers some steps or all the remaining steps (amount) via listing.
the user will be able to claim more tokens from the upcoming steps than intended.

If the user claims a certain number of steps and then transfers all the remaining steps then also the user able to claim the upcoming steps.

This is due to incorrect calculation in the `SecondSwap_StepVesting::transferVesting` contract as given below :-

This recalculation of releaseRate is causing the vulnerability

```solidity
    grantorVesting.totalAmount -= _amount;
    grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;
```

### Impact

User will be able to claim more tokens then its initial vesting.

After transferring vested tokens to others, a user will still be able to claim more than intended tokens from upcoming vesting steps.

### Proof of Concept

#### Example 1:

User A has a vesting of 5000 Tokens for 5 steps.

- amountPerStep = 1000
- After first 2 steps he claimed 2000
- claimedAmount = 2000
- totalAmount = 5000

Now the User A Lists the remaining vesting of 3000 and this `transferVesting` function is called.

```solidity
require(
    grantorVesting.totalAmount - grantorVesting.amountClaimed >= _amount,
    "SS_StepVesting: insufficient balance"
);
```

After listing:

```solidity
grantorVesting.totalAmount -= _amount; // = 2000
grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps; // 2000 / 3 = 666 (should be 0)
```

#### Example 2:

User A has a vesting of 5000 Tokens for 5 steps.

- amountPerStep = 1000
- claimedAmount = 2000
- transfers 2000 tokens

```solidity
grantorVesting.totalAmount -= _amount; // = 3000
grantorVesting.releaseRate = 3000 / 3 // = 1000 (should be 333)
```

## **Recommended mitigation steps**

```diff
 function transferVesting(address _grantor, address _beneficiary, uint256 _amount) external {
     require(
         msg.sender == tokenIssuer || msg.sender == manager || msg.sender == vestingDeployer,
         "SS_StepVesting: unauthorized"
     );
     require(_beneficiary != address(0), "SS_StepVesting: beneficiary is zero");
     require(_amount > 0, "SS_StepVesting: amount is zero");
     Vesting storage grantorVesting = _vestings[_grantor];
     require(
         grantorVesting.totalAmount - grantorVesting.amountClaimed >= _amount,
         "SS_StepVesting: insufficient balance"
     ); // 3.8. Claimed amount not checked in transferVesting function

-    grantorVesting.totalAmount -= _amount;
+    grantorVesting.totalAmount -= (grantorVesting.amountClaimed + _amount);
     grantorVesting.releaseRate = grantorVesting.totalAmount / numOfSteps;

     _createVesting(_beneficiary, _amount, grantorVesting.stepsClaimed, true);

     emit VestingTransferred(_grantor, _beneficiary, _amount);
 }
```

# <a id='M-01'></a>M-01. A user can list its vesting for sell more than the `maxSellPercentage` in the `SecondSwap_VestingManager::listVesting` contract.

## **Finding description and impact**

### Description

The sellLimit for a user in the `SecondSwap_VestingManager::listVesting` function is calculated incorrectly, allowing them to list more tokens above sell limit.

Each time a user lists their tokens for sale in the vesting contract, the sellLimit increases, which is the opposite of what should happen.

### Impact

The user will be able to sell more tokens than they have and above their sellLimit.
As the maxSellPercentage increases more token can be listed by the user.

### Proof of Concept

#### Case 1:

```
currentAlloc = 1000 Tokens
userAllocation.sold = 0
userAllocation.bought = 0
maxSellPercent = 2000 (20%)

sellLimit = ((1000 + 0 - 0) * 2000) / 10000 = 200
```

- User lists 200 tokens — OK

#### Case 2:

```
currentAlloc = 1000 Tokens
userAllocation.sold = 200
userAllocation.bought = 0
maxSellPercent = 2000 (20%)

sellLimit = ((1000 + 200 - 0) * 2000) / 10000 = 240
userAllocation.sold += 40 → 240 (exceeds 20%)
```

### Recommended mitigation steps

Use one of the following options to correctly limit the sellable amount:

```diff
 function listVesting(address seller, address plan, uint256 amount) external onlyMarketplace {
     require(vestingSettings[plan].sellable, "vesting not sellable");
     require(SecondSwap_Vesting(plan).available(seller) >= amount, "SS_VestingManager: insufficient availablility");

     Allocation storage userAllocation = allocations[seller][plan];

-    uint256 sellLimit = userAllocation.bought;
-    uint256 currentAlloc = SecondSwap_Vesting(plan).total(seller);
-
-    if (currentAlloc + userAllocation.sold > userAllocation.bought) {
-        sellLimit +=
-            ((currentAlloc + userAllocation.sold - userAllocation.bought) * vestingSettings[plan].maxSellPercent) /
-            BASE;
-    }

 // Use one of the mitigation given below

 // Option 1: maxSellPercent of (currentAlloc + userAllocation.bought)
+    uint256 sellLimit;
+    uint256 currentAlloc = SecondSwap_Vesting(plan).total(seller);
+    sellLimit = (( currentAlloc + userAllocation.bought ) * vestingSettings[plan].maxSellPercent)/ BASE;

 // Option 2: maxSellPercent of (currentAlloc) plus full bought
+    uint256 sellLimit = userAllocation.bought;
+    uint256 currentAlloc = SecondSwap_Vesting(plan).total(seller);
+    sellLimit += (currentAlloc * vestingSettings[plan].maxSellPercent) / BASE;

     userAllocation.sold += amount;

     require(userAllocation.sold <= sellLimit, "SS_VestingManager: cannot list more than max sell percent");
     SecondSwap_Vesting(plan).transferVesting(seller, address(this), amount);
 }
```
