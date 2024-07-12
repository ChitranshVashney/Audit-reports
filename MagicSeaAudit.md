# [H1]: Lock Time Manipulation Vulnerability in Booster::deposit Function of Booster Contract

## Summary
In the `Booster::deposit` function of `magicsea-staking/src/booster/Booster.sol`, an attacker can manipulate the `user.lockedAt` while depositing tokens. This vulnerability allows an attacker to reduce the lock time for future deposits, potentially converting their LUM to Magic LUM tokens in a significantly shorter period.
## Vulnerability Detail
**Issue:** Manipulation of Lock Time Calculation
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/booster/Booster.sol#L389-L404

The vulnerability lies in the calculation of the average lock time using the formula:
```diff
uint256 amount_old = amount;

uint256 userAmount = amount + _amount;
uint256 _time_locked = time_locked; // gas savings
- uint256 lockedFor = timeToUnlock(msg.sender);

-   lockedFor = (lockedFor * amount_old + _time_locked * _amount) / userAmount;
```

When `timeToUnlock(msg.sender)` returns 0, the attacker can manipulate the lock time for their subsequent deposits. This happens because lockedFor can be zero if timeToUnlock(msg.sender) returns 0, which is possible if the required lock time has already elapsed.

https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/booster/Booster.sol#L791-L796

By making a large deposit, the attacker can significantly influence the lockedFor value. This allows the attacker to reduce the lock time for future deposits, enabling them to convert their LUM to Magic LUM tokens much faster than intended.

**Detailed Explanation of Vulnerability:**
- Initial Deposit: The attacker makes an initial deposit. When the lock time elapses, `timeToUnlock(msg.sender)` will return 0.
- Subsequent Deposits: When the attacker makes subsequent deposits, the calculation `(lockedFor * amount_old + _time_locked * _amount) / userAmount` can be manipulated. If lockedFor is 0, the formula simplifies to (_time_locked * _amount) / userAmount, which reduces the average lock time for the new deposit.
- Manipulation: By strategically timing their deposits and leveraging the above calculation, the attacker can continuously reduce their lock time, converting their tokens faster than intended and gaining an unfair advantage.
Example:
Assume the attacker initially deposits 10000 tokens with a lock time of 72hr. After 72hr, `timeToUnlock` returns 0. If the attacker then deposits an additional 100 tokens, the new lock time calculation is skewed due to the zero value of lockedFor, significantly reducing the lock period by 99%.


## Impact
**High -** This vulnerability allows an attacker to reduce the lock time for their staked tokens, enabling them to convert their LUM to Magic LUM tokens much faster than intended. This undermines the staking mechanism and can lead to an unfair advantage and potential exploitation of the reward system.
## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/booster/Booster.sol#L389-L404

## Tool used

Manual Review

## Recommendation
To mitigate this vulnerability, add a condition to ensure `lockedFor` is not set to 0. If `timeToUnlock(msg.sender)` returns 0, set `lockedFor` to `_time_locked`. Here's the updated code:
```diff
function deposit(uint256 _amount) external nonReentrant onlyAllowed {
    UserInfo storage user = userInfo[msg.sender];

    if (!user.initialized) {
        users.push(msg.sender);
        user.initialized = true;
    }
    _updatePool();

    (uint256 rewards, uint256 amount, uint256 rewardDebt) = _computeSnapshotRewards(user);

    // we save the rewards (if any so far) for harvesting
    user.rewardClaim = user.rewardClaim + rewards;

    uint256 amount_old = amount;

    uint256 userAmount = amount + _amount;

    uint256 _burnAfterTime = burnAfterTime; // gas savings
    uint256 _stakedSupply = stakedSupply + _amount;

    totalBurnMultiplier = (_stakedSupply * PRECISION_FACTOR) / (_burnAfterTime);
    totalBurnUntil = block.timestamp + _burnAfterTime;
    stakedSupply = _stakedSupply;

    // set new locked amount based on average locking window
    uint256 _time_locked = time_locked; // gas savings
    uint256 lockedFor = timeToUnlock(msg.sender);
-    // avg lockedFor: (lockedFor * amount_old + blocks_locked * _amount) / user.amount
-   lockedFor = (lockedFor * amount_old + _time_locked * _amount) / userAmount;

+  // Ensure lockedFor is not manipulated
+    if (lockedFor != 0) {
+       lockedFor = (lockedFor * amount_old + _time_locked * _amount) / userAmount;
+    } else {
+       lockedFor = _time_locked;
+    }

    // set new locked at
    user.lockedAt = block.timestamp - (_time_locked - lockedFor);

    user.amount = userAmount;

    user.burnRound = lastBurnRound();

    user.rewardDebt = rewardDebt + ((_amount * accTokenPerShare) / PRECISION_FACTOR);

    // get tokens from user
    stakedToken.safeTransferFrom(address(msg.sender), address(this), _amount);

    emit DepositPosition(msg.sender, _amount);
}
```
By adding this condition, the lock time cannot be reduced to zero, preventing the attacker from manipulating the lock time to their advantage. This ensures that the staking mechanism remains fair and robust, maintaining the integrity of the reward distribution system.

# [M1]: Integration of Dynamic Month Duration Calculation in `Booster` Smart Contract

## Summary
The proposed integration of the `NextMonthDays` library into the **`Booster`** smart contract aims to replace the static definition of month duration (`SEC_PER_MONTH`) with a dynamic calculation based on the actual number of days in the next calendar month. This enhancement provides users with accurate information regarding the end time of reward periods, aligning with business model requirements for clarity and precision.
## Vulnerability Detail
The current implementation of SEC_PER_MONTH in the Booster smart contract uses a fixed value of 2,628,000 seconds, equating to approximately 30 days. However, this static definition does not account for variations in the number of days per month due to leap years and calendar irregularities. This could lead to discrepancies between the expected and actual end times of reward periods, potentially confusing users and affecting the accuracy of financial projections.
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/booster/Booster.sol#L111
By integrating the NextMonthDays library, the contract dynamically calculates the duration of the next month based on the current timestamp. This approach mitigates the risk of inaccuracies stemming from static month duration assumptions, ensuring that reward end times are reliably aligned with calendar months.
## Impact
**Medium:** The impact of inaccuracies in reward period calculations can lead to user confusion and misaligned expectations regarding the availability of rewards. In financial applications such as staking contracts, where timing precision is crucial, discrepancies in reward period end times could undermine user trust and satisfaction.
## Code Snippet
```diff
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./NextMonthDays.sol";

contract Booster {
-    uint256 public constant SEC_PER_MONTH = 2628000; // Financial month ~ 30 days
+    uint256 public constant SEC_PER_MONTH = 2628000; // Financial month ~ 30 days
    
    // Other contract code...

+    function getDynamicSecPerMonth() external view returns (uint256) {
+       return NextMonthDays.getNextMonthDays() * 1 days; // Convert days to seconds
    }

    // Other contract functions...
}
```
```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

library NextMonthDays {
    function getNextMonthDays() public view returns (uint8) {
        uint256 currentTimestamp = block.timestamp;
        uint8[12] memory daysInMonth = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];

        // Extract the current year and month from the timestamp
        (uint16 currentYear, uint8 currentMonth,) = _timestampToDate(currentTimestamp);

        // Determine the next month and year
        uint8 nextMonth = currentMonth == 12 ? 1 : currentMonth + 1;
        uint16 nextYear = currentMonth == 12 ? currentYear + 1 : currentYear;

        // Check for leap year if February is the next month
        if (nextMonth == 2 && _isLeapYear(nextYear)) {
            return 29;
        } else {
            return daysInMonth[nextMonth - 1];
        }
    }

    function _timestampToDate(uint256 timestamp) private pure returns (uint16 year, uint8 month, uint8 day) {
        uint256 z = timestamp / 86400 + 719468;
        uint256 era = (z >= 0 ? z : z - 146096) / 146097;
        uint256 doe = z - era * 146097;
        uint256 yoe = (doe - doe / 1460 + doe / 36524 - doe / 146096) / 365;
        uint256 y = yoe + era * 400;
        uint256 doy = doe - (365 * yoe + yoe / 4 - yoe / 100);
        uint256 mp = (5 * doy + 2) / 153;
        uint256 d = doy - (153 * mp + 2) / 5 + 1;
        uint256 m = mp < 10 ? mp + 3 : mp - 9;

        year = uint16(y + (m <= 2 ? 1 : 0));
        month = uint8(m);
        day = uint8(d);
    }

    function _isLeapYear(uint16 year) private pure returns (bool) {
        if (year % 4 == 0) {
            if (year % 100 == 0) {
                return year % 400 == 0;
            } else {
                return true;
            }
        } else {
            return false;
        }
    }
}

}
```
## Tool used

Manual Review

## Recommendation
To enhance the robustness and usability of the `Booster` smart contract, consider integrating the `NextMonthDays` library as proposed. This integration ensures that reward period end times accurately reflect the actual number of days in the next calendar month, thereby providing users with transparent and predictable reward scheduling. Additionally, document the integration thoroughly to facilitate understanding and maintenance by future developers.

# [L1]: Analysis of allowAccounts and disallowAccounts Functions

## Summary
This report audits the `allowAccounts` and `disallowAccounts` functions in the Booster contract. These functions enable the contract owner to bulk update the allow list of addresses. However, the current implementation has some vulnerabilities and inefficiencies that need to be addressed.
## Vulnerability Detail
### Function:  `allowAccounts` and `disallowAccounts`
If any address in the input array is invalid, the entire transaction is reverted. This can be costly in terms of gas fees, especially if the input array is large. Additionally, the transaction reverts after processing all valid addresses up to the invalid one, resulting in wasted computation.


## Impact
Low- These vulnerabilities and inefficiencies can lead to unnecessary gas costs and failed transactions, especially when dealing with large arrays of addresses. This can make the contract less user-friendly and more costly to operate.
## Code Snippet
https://github.com/sherlock-audit/2024-06-magicsea/blob/main/magicsea-staking/src/booster/Booster.sol#L280-L304
## Tool used

Manual Review

## Recommendation
- Address Checksum Validation: Validate addresses **off-chain** before submitting transactions to ensure they follow the correct checksum format. This can prevent transactions from being reverted due to invalid addresses.

- Remove Redundant Null Address Check: Consider removing the redundant check for null addresses in the disallowAccounts function to simplify the code and improve readability.
