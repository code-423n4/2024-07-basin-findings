# QA for Basin
## Table of Contents

| Issue ID | Description |
| -------- | ----------- |
| [QA-01](#qa-01-potential-non-convergence-in-calclptokensupply-function-could-lead-to-silent-failures) | Potential non-convergence in `calcLpTokenSupply` function could lead to silent failures |
| [QA-02](#qa-02-non-adherence-to-eip-1822-standard-in-wellupgradeablesol) | Non-adherence to EIP-1822 standard in `WellUpgradeable.sol` |
| [QA-03](#qa-03-misspelled-variable-name-in-upgrade-authorization-logic) | Misspelled variable name in Upgrade Authorization Logic |
| [QA-04](#qa-04-parity-reserve-calculation-in-calcreserveatratioswap-could-lead-to-suboptimal-pricing-in-some-edge-cases) | Parity reserve calculation in `calcReserveAtRatioSwap` could lead to suboptimal pricing in some edge cases |
| [QA-05](#qa-05-lack-of-check-for-identical-implementation-during-upgrade-in-_authorizeupgrade-function) | Lack of check for identical implementation during upgrade in `_authorizeUpgrade` function |

## [QA-01] Potential non-convergence in `calcLpTokenSupply` function could lead to silent failures

### Impact
The `calcLpTokenSupply` function may not always converge within the 255 iterations limit. If it doesn't converge, the function will silently exit the loop without returning a value or throwing an error. 

### Proof of Concept
Take a look at the `calcLpTokenSupply` function: https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L74-L103

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
```

The issue here is that if the loop completes without finding a convergent solution, the function will exit without returning a value, leading to silent failures.

### Recommended Mitigation Steps
Consider adding a revert statement after the loop to ensure that if the calculation doesn't converge within 255 iterations, the function will revert with a clear error message. Here is the recommended change in diff format:

```diff
        if (lpTokenSupply > prevReserves) {
            if (lpTokenSupply - prevReserves <= 1) return lpTokenSupply;
        } else {
            if (prevReserves - lpTokenSupply <= 1) return lpTokenSupply;
        }
    }
+   revert("LP token supply calculation did not converge");
}
```



## [QA-02] Non-adherence to EIP-1822 standard in `WellUpgradeable.sol`

### Impact
The `WellUpgradeable` contract fails to meet the EIP-1822 standard, potentially causing issues with upgradeability and compatibility with proxy contracts.

It is explicitly mentioned in the [ReadMe](https://github.com/code-423n4/2024-07-basin#eip-compliance) that `src/WellUpgradeable.sol` should adhere to EIP-1822.

### Proof of Concept

#### Issue 1: Absence of `updateCodeAddress` Functionality

According to the EIP-1822 standard:
> ⚠️ Warning: `updateCodeAddress` and `proxiable` must be present in the Logic Contract. Failure to include these may prevent upgrades and could allow the Proxy Contract to become entirely unusable. See below [Restricting dangerous functions](https://github.com/eth-infinitism/ERCs/blob/0a3011040045ea36b551dd287d895a1187204b96/ERCS/erc-1822.md#restricting-dangerous-functions).

`WellUpgradeable.sol`, which serves as a logic contract, lacks the `updateCodeAddress` function, which is necessary to store the Logic Contract's address at the specified storage position.

#### Issue 2: Incorrect `proxiableUUID` Implementation

EIP-1822 mandates that the `proxiableUUID` function should return a specific `bytes32` value derived from `keccak256("PROXIABLE")`. However, `WellUpgradeable` returns `_IMPLEMENTATION_SLOT` instead.

Here is the correct implementation as per the EIP-1822 Standard:

```solidity
function proxiableUUID() public pure returns (bytes32) {
    return 0xc5f16f0fcc639fa48a6947836d9850f504798523bf8c9a3a87d5876cf622bcf7;
}
```

And here is the incorrect implementation in `WellUpgradeable`:

```solidity
function proxiableUUID() external view override notDelegatedOrIsMinimalProxy returns (bytes32) {
    return _IMPLEMENTATION_SLOT;
}
```

#### Issue 3: Deviation from Constructor Pattern

EIP-1822 recommends a flexible constructor pattern for initializing the logic contract. `WellUpgradeable` does not follow this pattern.

Standard excerpt:

```solidity
constructor(bytes memory constructData, address contractLogic) public {
    // Initialization logic
}
```

But in `WellUpgradeable`:

```solidity
function init(string memory _name, string memory _symbol) external override reinitializer(2) {
    // Initialization logic
}
```

### Recommended Mitigation Steps
1. Implement the `updateCodeAddress` function to store the Logic Contract's address at the specified storage position.
2. Modify the `proxiableUUID` function to return the specific `bytes32` value as required by EIP-1822, ensuring compliance with the standard's expectations for identifying proxiable contracts.
3. Adjust the initialization process of `WellUpgradeable` to align with the constructor pattern recommended by EIP-1822.




## [QA-03] Misspelled variable name in Upgrade Authorization Logic

### Impact
The misspelling of the variable `newImplementation` as `newImplmentation` in the `_authorizeUpgrade` function prevents the contract from compiling successfully. This issue impedes the upgrade mechanism of the contract, potentially locking it in an outdated state.

### Proof of Concept
In the `_authorizeUpgrade` function, the following line attempts to verify that the new implementation address is a well implementation bored by an aquifer: https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L76

```solidity
require(
    IAquifer(aquifer).wellImplementation(newImplmentation) != address(0),
    "New implementation must be a well implmentation"
);
```

However, the variable `newImplmentation` is misspelled. The correct variable name, as passed to the function, is `newImplementation`.

### Recommended Mitigation Steps
Correct the spelling of the variable `newImplementation` in the `_authorizeUpgrade` function to match the parameter name and also correct all the instances in the contract.





## [QA-04] Parity reserve calculation in `calcReserveAtRatioSwap` could lead to suboptimal pricing in some edge cases

### Impact
The `calcReserveAtRatioSwap` function uses a simplified calculation for the parity reserve (`lpTokenSupply / 2`). This assumption may not hold true for stableswap pools, especially when the tokens have different precisions or when the pool is imbalanced. This can lead to suboptimal pricing calculations, particularly in edge cases or when the pool is far from balanced.

### Proof of Concept

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L173-L239

```solidity
// calculate lp token supply:
uint256 lpTokenSupply = calcLpTokenSupply(scaledReserves, abi.encode(18, 18));

// lpTokenSupply / 2 gives the reserves at parity:
uint256 parityReserve = lpTokenSupply / 2;

// update `scaledReserves` based on whether targetPrice is closer to low or high price:
if (pd.lutData.highPrice - pd.targetPrice > pd.targetPrice - pd.lutData.lowPrice) {
    // targetPrice is closer to lowPrice.
    scaledReserves[i] = parityReserve * pd.lutData.lowPriceI / pd.lutData.precision;
    scaledReserves[j] = parityReserve * pd.lutData.lowPriceJ / pd.lutData.precision;
    // initialize currentPrice:
    pd.currentPrice = pd.lutData.lowPrice;
} else {
    // targetPrice is closer to highPrice.
    scaledReserves[i] = parityReserve * pd.lutData.highPriceI / pd.lutData.precision;
    scaledReserves[j] = parityReserve * pd.lutData.highPriceJ / pd.lutData.precision;
    // initialize currentPrice:
    pd.currentPrice = pd.lutData.highPrice;
}
```

### Recommended Mitigation Steps
Replace the simplified calculation of `parityReserve` with a more accurate calculation that considers the amplification factor (`A`) and the specific dynamics of the pool. Implement a function to calculate the true parity reserve based on the current state of the pool and the amplification factor, and use this function in place of `lpTokenSupply / 2`.





## [QA-05] Lack of check for identical implementation during upgrade in `_authorizeUpgrade` function

### Impact
The absence of a check to ensure that the new implementation is different from the current implementation could lead to unnecessary upgrades. This might result in wasted gas fees and potential confusion during the upgrade process.

### Proof of Concept
The `_authorizeUpgrade` function currently verifies that the new implementation is a valid well implementation bored by an aquifer and a valid ERC-1967 implementation. However, it does not check if the new implementation is different from the current implementation.
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L65-L85

```solidity
function _authorizeUpgrade(address newImplementation) internal view override {
    // verify the function is called through a delegatecall.
    require(address(this) != ___self, "Function must be called through delegatecall");

    // verify the function is called through an active proxy bored by an aquifer.
    address aquifer = aquifer();
    address activeProxy = IAquifer(aquifer).wellImplementation(_getImplementation());
    require(activeProxy == ___self, "Function must be called through active proxy bored by an aquifer");

    // verify the new implementation is a well bored by an aquifer.
    require(
        IAquifer(aquifer).wellImplementation(newImplementation) != address(0),
        "New implementation must be a well implementation"
    );

    // verify the new implementation is a valid ERC-1967 implementation.
    require(
        UUPSUpgradeable(newImplementation).proxiableUUID() == _IMPLEMENTATION_SLOT,
        "New implementation must be a valid ERC-1967 implementation"
    );
}
```

### Recommended Mitigation Steps
Add a check to ensure that the new implementation is different from the current implementation. 

