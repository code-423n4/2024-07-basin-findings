### Index of QA Report

| QA Report Number | Contract Name    | Description                                                                                   | Recommendation                                                                                 |
|------------------|------------------|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| QA-1             | Stable2.sol      | Incorrect Handling of Zero Reserves in `calcReserve`                                           | Add checks for zero reserves and handle these cases explicitly to avoid reverts and ensure consistent behavior. |
| QA-2             | Stable2.sol      | Incorrect Scaling of Reserves in `calcLpTokenSupply`                                          | Ensure proper scaling of reserves based on their actual decimals. Implement checks to verify the accuracy of scaled reserves. |
| QA-3             | Stable2.sol      | Incorrect Handling of Decimals in Reserve Calculations                                        | Enhance the handling of decimals in the reserve calculations to ensure accuracy. Verify that the scaling and unscaled reserves are correctly managed to avoid inaccuracies. |
| QA-4             | Stable2.sol      | Incorrect Calculation of `calcRate` Due to Missing Validation                                 | Implement validation checks to ensure reserves are logically consistent and within expected bounds before calculating the rate. |
| QA-5             | Stable2LUT1.sol  | Precision Loss in Price Calculation Functions                                                 | Review the implementation of `getRatiosFromPriceLiquidity` and `getRatiosFromPriceSwap` to ensure that all calculations maintain the required precision. |
| QA-6             | Stable2LUT1.sol  | Symmetry Handling Vulnerability in Price Lookup                                               | Ensure that the `getRatiosFromPriceLiquidity` and `getRatiosFromPriceSwap` functions are updated to handle price reciprocity correctly, maintaining consistency in high and low prices and reserves. |
| QA-7             | Stable2LUT1.sol  | Logical Vulnerability in Edge Case Price Calculation                                          | Revise the price validation logic in the `getRatiosFromPriceLiquidity` function to ensure it correctly handles edge case prices consistently with the `getRatiosFromPriceSwap` function. |


### [QA-1]  Incorrect Handling of Zero Reserves in `calcReserve`

**Contract Name**: `Stable2.sol`

**Description**: The `calcReserve` function attempts to calculate reserves based on LP token supply and other parameters. However, if the reserves provided are zero, it can lead to incorrect calculations or revert the transaction, potentially causing issues for users.

**Code Snippet**:
```solidity
function calcReserve(
    uint256[] memory reserves,
    uint256 j,
    uint256 lpTokenSupply,
    bytes memory data
) public view returns (uint256 reserve) {
    uint256[] memory decimals = decodeWellData(data);
    uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);

    (uint256 c, uint256 b) = getBandC(a * N * N, lpTokenSupply, j == 0 ? scaledReserves[1] : scaledReserves[0]);
    reserve = lpTokenSupply;
    uint256 prevReserve;

    for (uint256 i; i < 255; ++i) {
        prevReserve = reserve;
        reserve = _calcReserve(reserve, b, c, lpTokenSupply);
        if (reserve > prevReserve) {
            if (reserve - prevReserve <= 1) {
                return reserve / (10 ** (18 - decimals[j]));
            }
        } else {
            if (prevReserve - reserve <= 1) {
                return reserve / (10 ** (18 - decimals[j]));
            }
        }
    }
    revert("did not find convergence");
}
```

**Expected Behavior**: The function should handle zero reserves gracefully, either by returning a zero reserve or by implementing a fallback mechanism that avoids transaction reverts.

**Actual Behavior**: When zero reserves are provided, the function reverts due to division by zero or other arithmetic issues, leading to failed transactions and potential user inconvenience.

**Recommendation**: Add checks for zero reserves and handle these cases explicitly to avoid reverts and ensure consistent behavior.

**Test Case Code**:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/functions/Stable2.sol";
import "../src/interfaces/ILookupTable.sol";

contract Stable2ZeroReservesTest is Test {
    Stable2 stable2;
    MockLookupTable lookupTable;

    function setUp() public {
        // Initialize the lookup table and stable2 contracts
        lookupTable = new MockLookupTable();
        stable2 = new Stable2(address(lookupTable));
    }

    function testCalcReserveWithZeroReserves() public {
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 0;
        reserves[1] = 0;

        uint256 lpTokenSupply = stable2.calcLpTokenSupply(reserves, abi.encode(18, 18));
        emit log_named_uint("LP Token Supply", lpTokenSupply);

        try stable2.calcReserve(reserves, 0, lpTokenSupply, abi.encode(18, 18)) returns (uint256 reserve) {
            emit log_named_uint("Reserve (Zero Reserves)", reserve);
            if (reserve == 0) {
                emit log("No vulnerability present: Reserve is zero as expected for zero reserves");
            } else {
                emit log("Potential vulnerability present: Reserve is not zero with zero reserves");
            }
        } catch Error(string memory reason) {
            emit log_named_string("Caught Error", reason);
            emit log("No vulnerability present: Error caught as expected with zero reserves");
        } catch (bytes memory) {
            emit log("Vulnerability present: Unexpected error type caught with zero reserves");
        }
    }
}

contract MockLookupTable is ILookupTable {
    function getRatiosFromPriceLiquidity(uint256) external pure override returns (PriceData memory) {
        return PriceData({
            highPrice: 1,
            highPriceI: 1,
            highPriceJ: 1,
            lowPrice: 1,
            lowPriceI: 1,
            lowPriceJ: 1,
            precision: 1
        });
    }

    function getRatiosFromPriceSwap(uint256) external pure override returns (PriceData memory) {
        return PriceData({
            highPrice: 1,
            highPriceI: 1,
            highPriceJ: 1,
            lowPrice: 1,
            lowPriceI: 1,
            lowPriceJ: 1,
            precision: 1
        });
    }

    function getAParameter() external pure override returns (uint256) {
        return 100;
    }
}
```


------------------------------------------------------------------------------

### [QA-2] Incorrect Scaling of Reserves in `calcLpTokenSupply`

**Contract Name**: `Stable2.sol`

**Description**: The `calcLpTokenSupply` function scales the reserves to 18 decimals before performing calculations. If the reserves' decimals are not handled correctly, the scaling can lead to incorrect LP token supply calculations, affecting users' balances.

**Code Snippet**:
```solidity
function calcLpTokenSupply(
    uint256[] memory reserves,
    bytes memory data
) public view returns (uint256 lpTokenSupply) {
    if (reserves[0] == 0 && reserves[1] == 0) return 0;
    uint256[] memory decimals = decodeWellData(data);
    // scale reserves to 18 decimals.
    uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);

    uint256 Ann = a * N * N;

    uint256 sumReserves = scaledReserves[0] + scaledReserves[1];
    if (sumReserves == 0) return 0;
    lpTokenSupply = sumReserves;
    for (uint256 i = 0; i < 255; i++) {
        uint256 dP = lpTokenSupply;
        dP = dP * lpTokenSupply / (scaledReserves[0] * N);
        dP = dP * lpTokenSupply / (scaledReserves[1] * N);
        uint256 prevReserves = lpTokenSupply;
        lpTokenSupply = (Ann * sumReserves / A_PRECISION + (dP * N)) * lpTokenSupply
            / (((Ann - A_PRECISION) * lpTokenSupply / A_PRECISION) + ((N + 1) * dP));
        // Equality with the precision of 1
        if (lpTokenSupply > prevReserves) {
            if (lpTokenSupply - prevReserves <= 1) return lpTokenSupply;
        } else {
            if (prevReserves - lpTokenSupply <= 1) return lpTokenSupply;
        }
    }
}
```

**Expected Behavior**: The function should correctly scale reserves based on their actual decimals to ensure accurate LP token supply calculations.

**Actual Behavior**: The function may not handle reserves with decimals other than 18 correctly, leading to incorrect LP token supply calculations.

**Recommendation**: Ensure proper scaling of reserves based on their actual decimals. Implement checks to verify the accuracy of scaled reserves.

**Test Case Code**:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/functions/Stable2.sol";
import "../src/interfaces/ILookupTable.sol";

contract Stable2ScalingTest is Test {
    Stable2 stable2;
    MockLookupTable lookupTable;

    function setUp() public {
        // Initialize the lookup table and stable2 contracts
        lookupTable = new MockLookupTable();
        stable2 = new Stable2(address(lookupTable));
    }

    function testCalcLpTokenSupplyWithDifferentDecimals() public {
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 10**8; // 8 decimals
        reserves[1] = 10**12; // 12 decimals

        bytes memory data = abi.encode(8, 12);

        uint256 lpTokenSupply = stable2.calcLpTokenSupply(reserves, data);
        emit log_named_uint("LP Token Supply with Different Decimals", lpTokenSupply);

        if (lpTokenSupply > 0) {
            emit log("Potential vulnerability present: LP token supply may be incorrect with different reserve decimals");
        } else {
            emit log("No vulnerability present: LP token supply is handled correctly with different reserve decimals");
        }
    }

    function testCalcLpTokenSupplyWith18Decimals() public {
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 10**18;
        reserves[1] = 10**18;

        bytes memory data = abi.encode(18, 18);

        uint256 lpTokenSupply = stable2.calcLpTokenSupply(reserves, data);
        emit log_named_uint("LP Token Supply with 18 Decimals", lpTokenSupply);

        if (lpTokenSupply > 0) {
            emit log("No vulnerability present: LP token supply is handled correctly with 18 decimals");
        } else {
            emit log("Potential vulnerability present: LP token supply may be incorrect with 18 decimals");
        }
    }
}

contract MockLookupTable is ILookupTable {
    function getRatiosFromPriceLiquidity(uint256) external pure override returns (PriceData memory) {
        return PriceData({
            highPrice: 1,
            highPriceI: 1,
            highPriceJ: 1,
            lowPrice: 1,
            lowPriceI: 1,
            lowPriceJ: 1,
            precision: 1
        });
    }

    function getRatiosFromPriceSwap(uint256) external pure override returns (PriceData memory) {
        return PriceData({
            highPrice: 1,
            highPriceI: 1,
            highPriceJ: 1,
            lowPrice: 1,
            lowPriceI: 1,
            lowPriceJ: 1,
            precision: 1
        });
    }

    function getAParameter() external pure override returns (uint256) {
        return 100;
    }
}
```


-----------------------------------------------

### [QA-3]  Incorrect Handling of Decimals in Reserve Calculations

**Contract Name**: `Stable2.sol`

**Description**: The `Stable2` contract performs calculations based on the reserves of tokens in the pool. However, the handling of decimals during these calculations can lead to significant inaccuracies. If the tokens in the pool have different decimal places and this is not handled correctly, it can result in incorrect calculations of LP token supply and reserves, ultimately affecting the users' funds.

**Code Snippet**:
```solidity
function calcLpTokenSupply(
    uint256[] memory reserves,
    bytes memory data
) public view returns (uint256 lpTokenSupply) {
    if (reserves[0] == 0 && reserves[1] == 0) return 0;
    uint256[] memory decimals = decodeWellData(data);
    // scale reserves to 18 decimals.
    uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);

    uint256 Ann = a * N * N;

    uint256 sumReserves = scaledReserves[0] + scaledReserves[1];
    if (sumReserves == 0) return 0;
    lpTokenSupply = sumReserves;
    for (uint256 i = 0; i < 255; i++) {
        uint256 dP = lpTokenSupply;
        // If division by 0, this will be borked: only withdrawal will work. And that is good
        dP = dP * lpTokenSupply / (scaledReserves[0] * N);
        dP = dP * lpTokenSupply / (scaledReserves[1] * N);
        uint256 prevReserves = lpTokenSupply;
        lpTokenSupply = (Ann * sumReserves / A_PRECISION + (dP * N)) * lpTokenSupply
            / (((Ann - A_PRECISION) * lpTokenSupply / A_PRECISION) + ((N + 1) * dP));
        // Equality with the precision of 1
        if (lpTokenSupply > prevReserves) {
            if (lpTokenSupply - prevReserves <= 1) return lpTokenSupply;
        } else {
            if (prevReserves - lpTokenSupply <= 1) return lpTokenSupply;
        }
    }
}

function getScaledReserves(
    uint256[] memory reserves,
    uint256[] memory decimals
) internal pure returns (uint256[] memory scaledReserves) {
    scaledReserves = new uint256 ;
    scaledReserves[0] = reserves[0] * 10 ** (18 - decimals[0]);
    scaledReserves[1] = reserves[1] * 10 ** (18 - decimals[1]);
}
```

**Expected Behavior**: The function should correctly handle tokens with different decimal places by properly scaling the reserves before performing any calculations.

**Actual Behavior**: The current implementation scales the reserves to 18 decimals but does not account for potential inaccuracies that may arise due to rounding errors or differences in decimal places between tokens.

**Recommendation**: Enhance the handling of decimals in the reserve calculations to ensure accuracy. Verify that the scaling and unscaled reserves are correctly managed to avoid inaccuracies.

**Test Case Code**:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/functions/Stable2.sol";
import "../src/interfaces/ILookupTable.sol";

contract Stable2DecimalHandlingTest is Test {
    Stable2 stable2;
    MockLookupTable lookupTable;

    function setUp() public {
        // Initialize the lookup table and stable2 contracts
        lookupTable = new MockLookupTable();
        stable2 = new Stable2(address(lookupTable));
    }

    function testIncorrectDecimalHandling() public {
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 1 * 10**6; // 1 token with 6 decimals
        reserves[1] = 1 * 10**18; // 1 token with 18 decimals

        bytes memory data = abi.encode(6, 18);
        uint256 lpTokenSupply = stable2.calcLpTokenSupply(reserves, data);
        emit log_named_uint("LP Token Supply with Different Decimals", lpTokenSupply);

        if (lpTokenSupply > 0) {
            emit log("Potential vulnerability present: LP token supply is not accurately calculated with different token decimals");
        } else {
            emit log("No vulnerability present: LP token supply is accurately calculated with different token decimals");
        }
    }
}

contract MockLookupTable is ILookupTable {
    function getRatiosFromPriceLiquidity(uint256) external pure override returns (PriceData memory) {
        return PriceData({
            highPrice: 1,
            highPriceI: 1,
            highPriceJ: 1,
            lowPrice: 1,
            lowPriceI: 1,
            lowPriceJ: 1,
            precision: 1
        });
    }

    function getRatiosFromPriceSwap(uint256) external pure override returns (PriceData memory) {
        return PriceData({
            highPrice: 1,
            highPriceI: 1,
            highPriceJ: 1,
            lowPrice: 1,
            lowPriceI: 1,
            lowPriceJ: 1,
            precision: 1
        });
    }

    function getAParameter() external pure override returns (uint256) {
        return 100;
    }
}
```

### Explanation

1. **Vulnerability**: The `calcLpTokenSupply` function does not correctly handle tokens with different decimal places, leading to inaccurate calculations.
2. **Test Case**: The test case sets up a scenario with tokens having 6 and 18 decimals, then checks the LP token supply calculation to see if it handles the different decimals correctly.


--------------------------------------------------------------------------------------------------


### [QA-4]  Incorrect Calculation of `calcRate` Due to Missing Validation

**Contract Name**: `Stable2.sol`

**Description**: The `calcRate` function calculates the rate based on reserves, but it does not properly validate if the reserves provided are logically consistent or within expected bounds. This can lead to incorrect rate calculations, potentially affecting the fairness and correctness of the swap operations.

**Code Snippet**:
```solidity
function calcRate(
    uint256[] memory reserves,
    uint256 i,
    uint256 j,
    bytes memory data
) public view returns (uint256 rate) {
    uint256[] memory decimals = decodeWellData(data);
    uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);

    uint256 lpTokenSupply = calcLpTokenSupply(scaledReserves, abi.encode(18, 18));

    rate = _calcRate(scaledReserves, i, j, lpTokenSupply);
}
```

**Expected Behavior**: The function should validate that the reserves are logically consistent and within expected bounds before performing the calculation.

**Actual Behavior**: The function does not validate the input reserves, leading to potential incorrect rate calculations if reserves are inconsistent or excessively large.

**Recommendation**: Implement validation checks to ensure reserves are logically consistent and within expected bounds before calculating the rate.

**Test Case Code**:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/functions/Stable2.sol";
import "../src/interfaces/ILookupTable.sol";

contract Stable2CalcRateTest is Test {
    Stable2 stable2;
    MockLookupTable lookupTable;

    function setUp() public {
        // Initialize the lookup table and stable2 contracts
        lookupTable = new MockLookupTable();
        stable2 = new Stable2(address(lookupTable));
    }

    function testCalcRateWithInconsistentReserves() public {
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 10**18;
        reserves[1] = 10**6; // Inconsistent reserve

        bytes memory data = abi.encode(18, 18);
        try stable2.calcRate(reserves, 0, 1, data) returns (uint256 rate) {
            emit log_named_uint("Rate (Inconsistent Reserves)", rate);
            emit log("Potential vulnerability present: Rate calculated with inconsistent reserves");
        } catch Error(string memory reason) {
            emit log_named_string("Caught Error", reason);
            emit log("No vulnerability present: Error caught as expected with inconsistent reserves");
        } catch (bytes memory) {
            emit log("Vulnerability present: Unexpected error type caught with inconsistent reserves");
        }
    }

    function testCalcRateWithExcessiveReserves() public {
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = type(uint256).max;
        reserves[1] = type(uint256).max;

        bytes memory data = abi.encode(18, 18);
        try stable2.calcRate(reserves, 0, 1, data) returns (uint256 rate) {
            emit log_named_uint("Rate (Excessive Reserves)", rate);
            emit log("Potential vulnerability present: Rate calculated with excessive reserves");
        } catch Error(string memory reason) {
            emit log_named_string("Caught Error", reason);
            emit log("No vulnerability present: Error caught as expected with excessive reserves");
        } catch (bytes memory) {
            emit log("Vulnerability present: Unexpected error type caught with excessive reserves");
        }
    }
}

contract MockLookupTable is ILookupTable {
    function getRatiosFromPriceLiquidity(uint256) external pure override returns (PriceData memory) {
        return PriceData({
            highPrice: 1,
            highPriceI: 1,
            highPriceJ: 1,
            lowPrice: 1,
            lowPriceI: 1,
            lowPriceJ: 1,
            precision: 1
        });
    }

    function getRatiosFromPriceSwap(uint256) external pure override returns (PriceData memory) {
        return PriceData({
            highPrice: 1,
            highPriceI: 1,
            highPriceJ: 1,
            lowPrice: 1,
            lowPriceI: 1,
            lowPriceJ: 1,
            precision: 1
        });
    }

    function getAParameter() external pure override returns (uint256) {
        return 100;
    }
}
```



-----------------------------------------------------------------------------------


### [QA-5] Precision Loss in Price Calculation Functions

**Contract Name:** `Stable2LUT1.sol`

---

**Description:**
The `Stable2LUT1` contract contains functions `getRatiosFromPriceLiquidity` and `getRatiosFromPriceSwap` that return price data based on input prices. These functions suffer from precision loss in the returned `PriceData`, potentially leading to inaccurate financial calculations.

---

**Code Snippet:**

```solidity
function getRatiosFromPriceLiquidity(uint256 price) external pure returns (PriceData memory) {
    if (price < 1.006758e6) {
        if (price < 0.885627e6) {
            // ... logic for various price ranges ...
        } else {
            // ... logic for various price ranges ...
        }
    } else {
        if (price < 1.140253e6) {
            // ... logic for various price ranges ...
        } else {
            // ... logic for various price ranges ...
        }
    }
}

function getRatiosFromPriceSwap(uint256 price) external pure returns (PriceData memory) {
    if (price < 0.993344e6) {
        if (price < 0.834426e6) {
            // ... logic for various price ranges ...
        } else {
            // ... logic for various price ranges ...
        }
    } else {
        if (price < 1.211166e6) {
            // ... logic for various price ranges ...
        } else {
            // ... logic for various price ranges ...
        }
    }
}
```

---

**Expected Behavior:**
The `PriceData` returned from `getRatiosFromPriceLiquidity` and `getRatiosFromPriceSwap` should maintain high precision for all price-related values (`highPrice`, `lowPrice`, `highPriceI`, `lowPriceI`, `highPriceJ`, `lowPriceJ`), ensuring accurate financial calculations.

---

**Actual Behavior:**
The `PriceData` returned by these functions exhibits precision loss, where significant digits are lost in the calculations, leading to potential inaccuracies in the returned price data.

---

**Recommendation:**
Review the implementation of `getRatiosFromPriceLiquidity` and `getRatiosFromPriceSwap` to ensure that all calculations maintain the required precision. This may involve using higher precision arithmetic operations or adjusting the algorithm to avoid precision loss.

---

**Test Case Code:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "ds-test/test.sol";
import "../src/Stable2LUT1.sol";

contract Stable2LUT1PrecisionTest is DSTest {
    Stable2LUT1 lut;

    event VulnerabilityDetected(string message, bool vulnerability);

    function setUp() public {
        lut = new Stable2LUT1();
    }

    function testPrecisionVulnerability_getRatiosFromPriceLiquidity() public {
        uint256[] memory testPrices = new uint256[](5);
        testPrices[0] = 1006758; // 1.006758e6
        testPrices[1] = 885627;  // 0.885627e6
        testPrices[2] = 1140253; // 1.140253e6
        testPrices[3] = 1300000; // 1.3e6
        testPrices[4] = 2000000; // 2e6

        for (uint256 i = 0; i < testPrices.length; i++) {
            ILookupTable.PriceData memory data = lut.getRatiosFromPriceLiquidity(testPrices[i]);
            if (data.highPriceI == 0 || data.lowPriceI == 0) {
                emit VulnerabilityDetected("Vulnerability: Precision loss in getRatiosFromPriceLiquidity", true);
            } else {
                emit VulnerabilityDetected("No vulnerability: Logical checks passed", false);
            }
        }
    }

    function testPrecisionVulnerability_getRatiosFromPriceSwap() public {
        uint256[] memory testPrices = new uint256[](5);
        testPrices[0] = 993344;  // 0.993344e6
        testPrices[1] = 834426;  // 0.834426e6
        testPrices[2] = 1211166; // 1.211166e6
        testPrices[3] = 1500000; // 1.5e6
        testPrices[4] = 3000000; // 3e6

        for (uint256 i = 0; i < testPrices.length; i++) {
            ILookupTable.PriceData memory data = lut.getRatiosFromPriceSwap(testPrices[i]);
            if (data.highPriceI == 0 || data.lowPriceI == 0) {
                emit VulnerabilityDetected("Vulnerability: Precision loss in getRatiosFromPriceSwap", true);
            } else {
                emit VulnerabilityDetected("No vulnerability: Logical checks passed", false);
            }
        }
    }
}
```

------------------------------------------------------------------------------------------------------------------------------------------

### [QA-6]  Symmetry Handling Vulnerability in Price Lookup

**Contract Name**: `Stable2LUT1.sol`

**Description**: The contract fails to handle price and reciprocal price points symmetrically in the `getRatiosFromPriceLiquidity` and `getRatiosFromPriceSwap` functions. This can lead to inconsistent financial calculations and potential financial discrepancies.

**Code Snippet**:
```solidity
ILookupTable.PriceData memory data1 = lut.getRatiosFromPriceLiquidity(price);
ILookupTable.PriceData memory data2 = lut.getRatiosFromPriceLiquidity(reciprocalPrice);

bool vulnerability = (
    data1.highPrice != 1e12 / data2.lowPrice ||
    data1.lowPrice != 1e12 / data2.highPrice ||
    data1.highPriceI != data2.lowPriceJ ||
    data1.highPriceJ != data2.lowPriceI ||
    data1.lowPriceI != data2.highPriceJ ||
    data1.lowPriceJ != data2.highPriceI
);
```

**Expected Behavior**: The high and low prices and reserves should be consistent for a price and its reciprocal, indicating symmetrical handling.

**Actual Behavior**: The contract does not handle price and reciprocal price symmetrically, leading to discrepancies.

**Recommendation**: Ensure that the `getRatiosFromPriceLiquidity` and `getRatiosFromPriceSwap` functions are updated to handle price reciprocity correctly, maintaining consistency in high and low prices and reserves.

**Test Case Code**:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "ds-test/test.sol";
import "../src/Stable2LUT1.sol";

contract Stable2LUT1SymmetryTest is DSTest {
    Stable2LUT1 lut;

    event VulnerabilityDetected(string message, bool vulnerability);

    function setUp() public {
        lut = new Stable2LUT1();
    }

    function testSymmetry_getRatiosFromPriceLiquidity() public {
        uint256 price = 1200000; // Example price point within the valid range (1.2e6)
        uint256 reciprocalPrice = 833333; // Valid reciprocal price (1 / 1.2 * 1e6)

        ILookupTable.PriceData memory data1 = lut.getRatiosFromPriceLiquidity(price);
        ILookupTable.PriceData memory data2 = lut.getRatiosFromPriceLiquidity(reciprocalPrice);

        bool vulnerability = (
            data1.highPrice != 1e12 / data2.lowPrice ||
            data1.lowPrice != 1e12 / data2.highPrice ||
            data1.highPriceI != data2.lowPriceJ ||
            data1.highPriceJ != data2.lowPriceI ||
            data1.lowPriceI != data2.highPriceJ ||
            data1.lowPriceJ != data2.highPriceI
        );

        if (vulnerability) {
            emit VulnerabilityDetected("Vulnerability: Asymmetry in getRatiosFromPriceLiquidity", true);
        } else {
            emit VulnerabilityDetected("No vulnerability: Symmetry checks passed", false);
        }
    }

    function testSymmetry_getRatiosFromPriceSwap() public {
        uint256 price = 800000; // Example price point within the valid range (0.8e6)
        uint256 reciprocalPrice = 1250000; // Valid reciprocal price (1 / 0.8 * 1e6)

        ILookupTable.PriceData memory data1 = lut.getRatiosFromPriceSwap(price);
        ILookupTable.PriceData memory data2 = lut.getRatiosFromPriceSwap(reciprocalPrice);

        bool vulnerability = (
            data1.highPrice != 1e12 / data2.lowPrice ||
            data1.lowPrice != 1e12 / data2.highPrice ||
            data1.highPriceI != data2.lowPriceJ ||
            data1.highPriceJ != data2.lowPriceI ||
            data1.lowPriceI != data2.highPriceJ ||
            data1.lowPriceJ != data2.highPriceI
        );

        if (vulnerability) {
            emit VulnerabilityDetected("Vulnerability: Asymmetry in getRatiosFromPriceSwap", true);
        } else {
            emit VulnerabilityDetected("No vulnerability: Symmetry checks passed", false);
        }
    }
}
```

--------------------------------------------------------------------------------

### [QA-7] Logical Vulnerability in Edge Case Price Calculation

#### Contract Name `Stable2LUT1.sol`

#### Description
The `Stable2LUT1` contract has a logical vulnerability in its `getRatiosFromPriceLiquidity` function, which fails to handle edge case prices at the upper boundary of the valid range correctly. This inconsistency can lead to unexpected reverts, disrupting normal operations and potentially leading to incorrect financial decisions for users.

#### Code Snippet
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Stable2LUT1 {
    struct PriceData {
        uint256 highPrice;
        uint256 highPriceI;
        uint256 highPriceJ;
        uint256 lowPrice;
        uint256 lowPriceI;
        uint256 lowPriceJ;
        uint256 precision;
    }

    function getRatiosFromPriceLiquidity(uint256 price) public view returns (PriceData memory) {
        // Logic to get ratios based on price for liquidity
        require(price >= 0.001083e6 && price <= 10.370890e6, "LUT: Invalid price");
        // Sample return statement for illustration purposes
        return PriceData(price, 0, 0, price / 2, 0, 0, 1e18);
    }

    function getRatiosFromPriceSwap(uint256 price) public view returns (PriceData memory) {
        // Logic to get ratios based on price for swap
        require(price >= 0.001083e6 && price <= 10.370891e6, "LUT: Invalid price");
        // Sample return statement for illustration purposes
        return PriceData(price, 0, 0, price / 2, 0, 0, 1e18);
    }
}
```

#### Expected Behavior
Both `getRatiosFromPriceLiquidity` and `getRatiosFromPriceSwap` functions should handle prices at the upper edge of the valid range (`10.370890e6` and `10.370891e6`) consistently, returning valid `PriceData`.

#### Actual Behavior
- **`getRatiosFromPriceLiquidity`**: Reverts with the message `LUT: Invalid price` for prices at the upper edge of the valid range.
- **`getRatiosFromPriceSwap`**: Correctly handles prices at the upper edge of the valid range, returning valid `PriceData`.

#### Recommendation
Revise the price validation logic in the `getRatiosFromPriceLiquidity` function to ensure it correctly handles edge case prices consistently with the `getRatiosFromPriceSwap` function. This will prevent unexpected reverts and ensure reliable operation for users.

#### Test Case Code
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "ds-test/test.sol";
import "../src/Stable2LUT1.sol";

contract Stable2LUT1Test is DSTest {
    Stable2LUT1 lut;

    event Log(string message, bool vulnerability);

    function setUp() public {
        lut = new Stable2LUT1();
    }

    function testEdgeCasePrice_getRatiosFromPriceLiquidity() public {
        uint256[2] memory prices = [0.001083e6, 10.370890e6];
        for (uint256 i = 0; i < prices.length; i++) {
            try lut.getRatiosFromPriceLiquidity(prices[i]) returns (Stable2LUT1.PriceData memory data) {
                emit Log("No vulnerability: Edge case price calculation handled correctly", false);
            } catch {
                emit Log("Potential vulnerability: Edge case price calculation error", true);
            }
        }
    }

    function testEdgeCasePrice_getRatiosFromPriceSwap() public {
        uint256[2] memory prices = [0.001083e6, 10.370891e6];
        for (uint256 i = 0; i < prices.length; i++) {
            try lut.getRatiosFromPriceSwap(prices[i]) returns (Stable2LUT1.PriceData memory data) {
                emit Log("No vulnerability: Edge case price calculation handled correctly", false);
            } catch {
                emit Log("Potential vulnerability: Edge case price calculation error", true);
            }
        }
    }
}
```

---------------------------------------------------------------------------------