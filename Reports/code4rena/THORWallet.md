# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. The max `TGT_TO_EXCHANGE` can be surpassed, and the user will be unable to receive `TITN` tokens.](#H-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: THORWallet

### Dates: Feb 21st, 2025 - Feb 27th, 2025

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 1
- Medium: 0
- Low: 0

# <a id='H-01'></a>H-01. The max `TGT_TO_EXCHANGE` can be surpassed, and the user will be unable to receive `TITN` tokens.

## **Summary**

In the `MergeTgt` contract, the `TGT_TO_EXCHANGE` limit can be surpassed. Users can deposit more TGT tokens than `TGT_TO_EXCHANGE` and will not be able to receive TITN tokens.

## **Vulnerability Details**

Users can send TGT tokens to the `MergeTgt` contract and receive TITN tokens in return.

However, the contract has a limited supply of TITN tokens, defined by `TITN_ARB`. There is no check in the contract to prevent users from sending more tokens than `TGT_TO_EXCHANGE`.

Users collectively or a malicious attacker can cause `TGT_TO_EXCHANGE` to be surpassed. Since there are only limited `TITN` tokens, some users will be unable to receive their `TITN` tokens, and their `TGT` tokens will be stuck in the contract.

### Example Code (quoteTitn function)

```solidity
function quoteTitn(uint256 tgtAmount) public view returns (uint256 titnAmount) {
    require(launchTime > 0, "Launch time not set");

    uint256 timeSinceLaunch = (block.timestamp - launchTime);
    if (timeSinceLaunch < 90 days) {
        titnAmount = (tgtAmount * TITN_ARB) / TGT_TO_EXCHANGE;
    } else if (timeSinceLaunch < 360 days) {
        uint256 remainingtime = 360 days - timeSinceLaunch;
        titnAmount = (tgtAmount * TITN_ARB * remainingtime) / (TGT_TO_EXCHANGE * 270 days);
    } else {
        titnAmount = 0;
    }
}
```

Additionally, if the user does not claim TITN tokens completely until 360 days and tries to use `withdrawRemainingTitn`, the function will revert due to `remainingTitnAfter1Year < initialTotalClaimable`:

The line uint256 unclaimedTitn = remainingTitnAfter1Year - initialTotalClaimable; will revert

``````solidity
function withdrawRemainingTitn() external nonReentrant {
        require(launchTime > 0, "Launch time not set");

        if (block.timestamp - launchTime < 360 days) {
            revert TooEarlyToClaimRemainingTitn();
        }

        uint256 currentRemainingTitn = titn.balanceOf(address(this));

        if (remainingTitnAfter1Year == 0) {
            // Initialize remainingTitnAfter1Year to the current balance of TITN
@>>            remainingTitnAfter1Year = currentRemainingTitn;

            // Capture the total claimable TITN at the time of the first claim
@>>            initialTotalClaimable = totalTitnClaimable;
        }

        uint256 claimableTitn = claimableTitnPerUser[msg.sender];
        require(claimableTitn > 0, "No claimable TITN");

        // Calculate proportional remaining TITN for the user
@>>        uint256 unclaimedTitn = remainingTitnAfter1Year - initialTotalClaimable;
          `````
          `````
          `````
``````

### Proof of Concept

```
TGT_TO_EXCHANGE = 579_000_000 * 10 ** 18
TITN_ARB = 173_700_000 * 10 ** 18

Lets take the calculation of timesiceLaunch < 90 days
titnAmount = (tgtAmount * TITN_ARB) / TGT_TO_EXCHANGE;

lets say 10000 user deposited the TGT_TO_EXCHANGE in 80 days
totalTitnClaimable = TITN_ARB

Now,Since there is no check for the max TGT_TO_EXCHANGE, other users can deposit more TGT tokens to exchange them for TITN_ARB.

These users will have entries in the mapping claimableTitnPerUser and totalTitnClaimable, but they will not be able to withdraw. The claimTitn function will revert due to insufficient funds.
```

i have taken the example of
timesiceLaunch < 90 days

But since the max limit can be exceeded by a malicious attacker, other calculations for `titnOut` can be manipulated.

## **Impact**

- Users who deposit after the total TGT_TO_EXCHANGE limit has been reached will not receive TITN tokens.
  Additionally, the TGT tokens sent by them will be stuck in the contract.

- if the users do not claim `TITN` tokens completely till `360 days` and try to use,`withdrawRemainingTitn` function
  `withdrawRemainingTitn` function will not work since,
  will revert [remainingTitnAfter1Year < initialTotalClaimable]
  because [ titn.balanceOf(address(this)) < totalTitnClaimable]
  uint256 unclaimedTitn = remainingTitnAfter1Year - initialTotalClaimable;

## **Recommendations**

Add a new global storage variable and update logic to cap `TGT_TO_EXCHANGE` usage properly:

```diff
+ uint256 public totalTGTDeposited;

function quoteTitn(uint256 tgtAmount) public view returns (uint256 titnAmount) {
    require(launchTime > 0, "Launch time not set");
+   require(totalTGTDeposited + tgtAmount < TGT_TO_EXCHANGE, "Max cap reached");
+   totalTGTDeposited += tgtAmount;

    uint256 timeSinceLaunch = (block.timestamp - launchTime);
    if (timeSinceLaunch < 90 days) {
        titnAmount = (tgtAmount * TITN_ARB) / TGT_TO_EXCHANGE;
    } else if (timeSinceLaunch < 360 days) {
        uint256 remainingtime = 360 days - timeSinceLaunch;
        titnAmount = (tgtAmount * TITN_ARB * remainingtime) / (TGT_TO_EXCHANGE * 270 days);
    } else {
        titnAmount = 0;
    }
}
```
