# [L-1] Missing check for 0 `lpTokenSupply` in `calcReserve` function
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L114-#L144
## Impact
The `calcReserve` function currently lacks a check to ensure that `lpTokenSupply` is not zero. Adding such a check and potentially computing `lpTokenSupply` using the `calcTokenSupply` function if it is zero can ensures that the function can handle unexpected input values. 
## Proof of concept
Here is a part of the current implementation of the `calcReserve` function
```solidity
    function calcReserve( 
        uint256[] memory reserves,
        uint256 j,
        uint256 lpTokenSupply,
        bytes memory data
    ) public view returns (uint256 reserve) {
        uint256[] memory decimals = decodeWellData(data);
        uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);

        // avoid stack too deep errors.
        (uint256 c, uint256 b) = getBandC(a * N * N, lpTokenSupply, j == 0 ? scaledReserves[1] : scaledReserves[0]); //ok
        reserve = lpTokenSupply;
        // code
    }
```

## Recommended mitigation steps
Add check for `lpTokenSupply == 0` and compute it if it's true:

```diff
    function calcReserve( 
        uint256[] memory reserves,
        uint256 j,
        uint256 lpTokenSupply,
        bytes memory data
    ) public view returns (uint256 reserve) {
        uint256[] memory decimals = decodeWellData(data);
        uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);
+       if (lpTokenSupply == 0) {
+           lpTokenSupply = calcLpTokenSupply(scaledReserves. decimals);
+       }
        // avoid stack too deep errors.
        (uint256 c, uint256 b) = getBandC(a * N * N, lpTokenSupply, j == 0 ? scaledReserves[1] : scaledReserves[0]); //ok
        reserve = lpTokenSupply;
        // code
    }
```



# [I-1] Incorrect documentation in `calcReserveAtRatioSwap` function
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L228
## Impact
The comment in the `calcReserveAtRatioSwap` function is inaccurate regarding the price threshold used for validation. The function checks if the new price is within a defined PRICE_THRESHOLD of the target price, but the comment incorrectly states that it checks if the new price is within 1 of the target price. While the functionality is not impacted, the misleading comment could cause confusion for developers/SRs reading or maintaining the code. Inaccurate documentation can lead to misunderstandings about the function's behavior.
## Proof of concept
```solidity
for (uint256 k; k < 255; k++) {
            scaledReserves[j] = updateReserve(pd, scaledReserves[j]);

            // calculate scaledReserve[i]:
            scaledReserves[i] = calcReserve(scaledReserves, i, lpTokenSupply, abi.encode(18, 18));
            // calc currentPrice:
            pd.currentPrice = _calcRate(scaledReserves, i, j, lpTokenSupply);

@>          // check if new price is within 1 of target price:
            if (pd.currentPrice > pd.targetPrice) {
                if (pd.currentPrice - pd.targetPrice <= PRICE_THRESHOLD) {
                    return scaledReserves[j] / (10 ** (18 - decimals[j]));
                }
            } else {
                if (pd.targetPrice - pd.currentPrice <= PRICE_THRESHOLD) {
                    return scaledReserves[j] / (10 ** (18 - decimals[j]));
                }
            }
        }
```

## Recommended mitigation steps
Rewrite the comment to:
// check if new price is within PRICE_THRESHOLD:
```diff
for (uint256 k; k < 255; k++) {
            scaledReserves[j] = updateReserve(pd, scaledReserves[j]);

            // calculate scaledReserve[i]:
            scaledReserves[i] = calcReserve(scaledReserves, i, lpTokenSupply, abi.encode(18, 18));
            // calc currentPrice:
            pd.currentPrice = _calcRate(scaledReserves, i, j, lpTokenSupply);

-           // check if new price is within 1 of target price:
+           // check if new price is within PRICE_THRESHOLD:
            if (pd.currentPrice > pd.targetPrice) {
                if (pd.currentPrice - pd.targetPrice <= PRICE_THRESHOLD) {
                    return scaledReserves[j] / (10 ** (18 - decimals[j]));
                }
            } else {
                if (pd.targetPrice - pd.currentPrice <= PRICE_THRESHOLD) {
                    return scaledReserves[j] / (10 ** (18 - decimals[j]));
                }
            }
        }
```

# [I-2] Unreachable code block in `calcReserveAtRatioSwap` function 
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L216
## Impact
In the `calcReserveAtRatioSwap` function, the calculation for maxStepSize includes an else block that appears to be unreachable based on the current implementation of StableLUT1. Specifically, the condition `pd.lutData.lowPriceJ > pd.lutData.highPriceJ` is always true, making the `else` block redundant. The redundancy of the else block does not affect functionality, but can contribute to unnecessary code complexity. 

## Proof of concept
The relevant section of code in the `calcReserveAtRatioSwap`:
```solidity
    // calculate max step size:
    if (pd.lutData.lowPriceJ > pd.lutData.highPriceJ) {
        pd.maxStepSize = scaledReserves[j] * (pd.lutData.lowPriceJ - pd.lutData.highPriceJ) / pd.lutData.lowPriceJ;
    } else { 
        pd.maxStepSize = scaledReserves[j] * (pd.lutData.highPriceJ - pd.lutData.lowPriceJ) / pd.lutData.highPriceJ;
    }
```
Given the current implementation of `StableLUT1`, `pd.lutData.lowPriceJ` is always greater than `pd.lutData.highPriceJ`. Thus, the else block is never executed.
## Recommended mitigation steps
Since the else block is unreachable, you can simplify the code by removing it. Alternatively, if there is a possibility that the implementation of `StableLUT1` might change in the future, it might be useful to leave the else block.

# [I-3] Unreachable code block in `calcReserveAtRatioLiquidity` function 
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L284
## Impact
In the `calcReserveAtRatioLiquidity` function, the calculation for maxStepSize includes an else block that appears to be unreachable based on the current implementation of StableLUT1. Specifically, the condition `pd.lutData.lowPriceJ > pd.lutData.highPriceJ` is always true, making the `else` block redundant. The redundancy of the else block does not affect functionality, but can contribute to unnecessary code complexity. 

## Proof of concept
The relevant section of code in the `calcReserveAtRatioLiquidity`:
```solidity
    // calculate max step size:
    if (pd.lutData.lowPriceJ > pd.lutData.highPriceJ) {
        pd.maxStepSize = scaledReserves[j] * (pd.lutData.lowPriceJ - pd.lutData.highPriceJ) / pd.lutData.lowPriceJ;
    } else { 
        pd.maxStepSize = scaledReserves[j] * (pd.lutData.highPriceJ - pd.lutData.lowPriceJ) / pd.lutData.highPriceJ;
    }
```
Given the current implementation of `StableLUT1`, `pd.lutData.lowPriceJ` is always greater than `pd.lutData.highPriceJ`. Thus, the else block is never executed.
## Recommended mitigation steps
Since the else block is unreachable, you can simplify the code by removing it. Alternatively, if there is a possibility that the implementation of `StableLUT1` might change in the future, it might be useful to leave the else block.
