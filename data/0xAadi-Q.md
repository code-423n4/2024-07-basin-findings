# [L-01] Revert if Convergence is Not Found

## Summary

The contract performs iterative calculations to find convergence in functions like `calcLpTokenSupply`, `calcReserveAtRatioSwap`, and `calcReserveAtRatioLiquidity`. However, there is no explicit revert if convergence is not achieved within the defined iteration limit.

## Vulnerability Detail

Convergence is typically expected within a few iterations. And it will be safe to revert the transaction if convergence is not found.

## Code Snippet

```solidity
src/functions/Stable2.sol:
   88:         for (uint256 i = 0; i < 255; i++) {
   89              uint256 dP = lpTokenSupply;

  220:         for (uint256 k; k < 255; k++) {
  221              scaledReserves[j] = updateReserve(pd, scaledReserves[j]);

  288:         for (uint256 k; k < 255; k++) {
  289              scaledReserves[j] = updateReserve(pd, scaledReserves[j]);
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L102
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L238
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L303

## Recommendation

Introduce a revert statement if convergence is not achieved within the allowed iterations to prevent potential issues and ensure predictable behavior.

# [L-02] Validate `j` Parameter

## Summary

The parameter `j` in functions like `calcReserve()` and `calcReserveAtRatioSwap()` is used to index reserves and ratios. There is no validation to ensure that `j` is within the expected range (0 to 1).

## Vulnerability Detail

If `j` is not properly validated, it could lead to out-of-bounds access or unexpected behavior.

## Impact

Unvalidated parameters could cause out-of-bounds errors or incorrect calculations, potentially leading to security issues or inaccurate results.

## Code Snippet

calcReserve()
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L116

calcReserveAtRatioSwap()
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L175

## Recommendation

Add checks to ensure `j` is less than `N` to prevent invalid access and ensure correct parameter values.

```solidity
require(j < N, "Invalid index for reserve or ratio");
```

# [L-03] Minimum Scaled Reserves Requirement Not Enforced

## Summary

The function `calcRate()` mentions that a minimum scaled reserve of `1e12` is required, but this requirement is not enforced in the implementation.

## Vulnerability Detail

Lack of enforcement for minimum scaled reserves can lead to incorrect calculations or unexpected results when reserves are below the required threshold.

## Impact

Could cause incorrect rate calculations if reserves are too small, potentially leading to unexpected behavior or inaccuracies.

## Code Snippet

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L149

## Recommendation

Implement a check to enforce the minimum scaled reserve requirement before performing calculations.

# [L-04] Validate `i` and `j` Parameters in `_calcRate()`

## Summary

The parameters `i` and `j` in the `_calcRate()` function should be validated to ensure they are within the range of valid indices.

## Vulnerability Detail

Unvalidated `i` and `j` parameters could lead to out-of-bounds errors or incorrect calculations.

## Impact

Potential for out-of-bounds access or incorrect calculations if `i` and `j` are not properly validated.

## Code Snippet

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L219

## Recommendation

Ensure that `i` and `j` are validated to be within the range of valid indices (`0` and `1`) and both should not be the same.

# [L-05] Validate Length of Reserves and Ratios in `calcReserveAtRatioSwap`

## Summary

The `calcReserveAtRatioSwap` function performs operations on `reserves` and `ratios` arrays but does not validate their lengths.

## Vulnerability Detail

Lack of validation for the length of `reserves` and `ratios` arrays can lead to out-of-bounds errors or incorrect calculations if these arrays are not properly sized.

## Impact

Incorrect array lengths can cause out-of-bounds errors or unexpected behavior, leading to potential security issues or inaccurate results.

## Code Snippet

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L174C3-L176C33

## Recommendation

Add validation to ensure that `reserves` and `ratios` arrays have the expected length before performing operations.

```solidity
require(reserves.length == N && ratios.length == N, "Invalid length of reserves or ratios");
```

# [L-06] Avoid Division Before Multiplication for More Accurate Values

## Summary

The calculation in `calcRate()` performs division before multiplication, which can lead to less accurate results due to rounding errors.

## Vulnerability Detail

Performing division before multiplication can lead to precision loss and inaccurate results, especially in calculations involving large numbers.

## Code Snippet

```solidity
src/functions/Stable2.sol:
   380:         c = lpTokenSupply * lpTokenSupply / (reserves * N) * lpTokenSupply * A_PRECISION / (Ann * N);
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L380

## Recommendation

Rearrange the operations to perform multiplication before division to improve accuracy.

```solidity
c = (lpTokenSupply * lpTokenSupply * lpTokenSupply * A_PRECISION) / (reserves * N * Ann * N);
```

# [NC-01] Remove Unnecessary `IERC20` Import

## Summary

The contract imports `IERC20` from the `forge-std` library, but it is not used anywhere in the contract.

## Issue Detail

An unused import does not affect the functionality of the contract but can lead to unnecessary code clutter and potentially increase compilation time.

## Impact

Removing the unused import will clean up the codebase and improve readability without affecting the contract's functionality.

## Code Snippet

```solidity
import {IERC20} from "forge-std/interfaces/IERC20.sol";
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L8

## Recommendation

Remove the `IERC20` import if it is not used in the contract.

```solidity
// Remove this line
import {IERC20} from "forge-std/interfaces/IERC20.sol";
```

# [NC-02] Remove Redundant Check for Zero Reserves

## Summary

The code contains a check for `sumReserves` being zero, which is redundant given that an earlier check already ensures that both `reserves[0]` and `reserves[1]` are zero.

## Issue Detail

Redundant checks can lead to unnecessary code execution

## Impact

While this check is unlikely to cause major issues, it can be optimized for better code clarity and efficiency.

## Code Snippet

```solidity
// Redundant check
if (sumReserves == 0) return 0;
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L86

```solidity
// Early check for zero reserves
if (reserves[0] == 0 && reserves[1] == 0) return 0;
```

## Recommendation

Remove the redundant `sumReserves` check and rely on the earlier `reserves` check to handle the case where both reserves are zero.

```solidity
// Remove the redundant check
if (sumReserves == 0) return 0;
```
