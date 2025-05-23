
# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
- ## Medium Risk Findings
    - ### [M-01. User role can't be removed .The _setRole function cannot remove a role for a user due to incorrect implementation; it fails to remove the user's role.](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: Winnables Raffles 

### Dates: Aug 16th 2024 - Aug 20 2024

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 0
- Medium: 1
- Low: 0

# <a id='M-01'></a>M-01. User role can't be removed .The _setRole function cannot remove a role for a user due to incorrect implementation; it fails to remove the user's role.

[MEDIUM]- User role can't be removed .The _setRole function cannot remove a role for a user due to incorrect implementation; it fails to remove the user's role.

## **Summary**  
In the Roles.sol::_setRole function, once a role is assigned to a user, it cannot be removed due to a flaw in the implementation.  
The bool status parameter is not used in the current implementation.

## **Vulnerability Detail**  
The _setRole function in the contract is intended to manage user roles by either assigning or removing a specific role based on the bool status parameter. However, the current implementation ignores the status parameter, failing to remove a role from a user when status is false.  
Once a role is assigned to a user, it cannot be removed with the current implementation of the function.

## **Impact**  
The role assigned to a user cannot be removed by the admin or anyone else if they want to ensure that a user does not have a particular role in future in the protocol.  
This vulnerability could be exploited to compromise the integrity of the raffle, potentially leading to unfair results or financial loss.

**Code Snippet**  
https://github.com/sherlock-audit/2024-08-winnables-raffles/blob/main/public-contracts/contracts/Roles.sol#L29

```solidity
function _setRole(address user, uint8 role, bool status) internal virtual {
        uint256 roles = uint256(_addressRoles[user]);
        _addressRoles[user] = bytes32(roles | (1 << role));
        emit RoleUpdated(user, role, status);
    }
```

## **Tool used**  
Manual Review

## **Recommendation**  
The following implementation can be done to ensure the intended working of the _setRole function, It ensures that a role is assigned to a user if status is true and removed from a user if status is false.

```diff
function _setRole(address user, uint8 role, bool status) internal virtual {
    uint256 roles = uint256(_addressRoles[user]);
-    _addressRoles[user] = bytes32(roles | (1 << role));
+    if (status) {
+       // Assign role
+       _addressRoles[user] = bytes32(roles | (1 << role));
+   } else {
+        // Remove role
+        _addressRoles[user] = bytes32(roles & ~(1 << role));
+    }
    emit RoleUpdated(user, role, status);
}
```
