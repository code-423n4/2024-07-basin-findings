# Non Critical Impact Issues

## 1: Spelling Errors

Vulnerability details

## Proof of Concept

> ***File: WellUpgradeable.sol***

implmentation can be changed to implementation in the following comments.

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L57

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L59

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L63

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L74

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L77

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L80


aquifier can be changed to aquifer in the following comments.

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L63 

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L69 


delgate can be changed to delegate in the following comment.

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L115 

> ***File: Stable2.sol***

prohibtive can be changed to prohibitive in the following comment.

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L151 


### Tools Used

Manual Analysis


## 2: FLOATING PRAGMA SHOULD BE AVOIDED

Vulnerability details

### Context:

### FLOATING PRAGMA SHOULD BE AVOIDED

### Proof of Concept

> ***3 Files, 3 Instances*** 

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L3
```solidity
	pragma solidity ^0.8.20;
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L3
```solidity
	pragma solidity ^0.8.20;
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/StableLUT/Stable2LUT1.sol#L3
```solidity
	pragma solidity ^0.8.20;
```

### Tools Used

Manual Analysis

### Recommended Mitigation Steps

Lock solidity pragmas

## 3: Revert statements within external and public functions can be used to perform DOS attacks

Vulnerability details

### Context:

In Solidity, 'revert' statements are used to undo changes and throw an exception when certain conditions are not met. However, in public and external functions, improper use of `revert` can be exploited for Denial of Service (DoS) attacks. An attacker can intentionally trigger these 'revert' conditions, causing legitimate transactions to consistently fail. For example, if a function relies on specific conditions from user input or contract state, an attacker could manipulate these to continually force reverts, blocking the function's execution. Therefore, it's crucial to design contract logic to handle exceptions properly and avoid scenarios where `revert` can be predictably triggered by malicious actors. This includes careful input validation and considering alternative design patterns that are less susceptible to such abuses.

### Proof of Concept

### Contracts

> ***2 Files, 3 Instances*** 

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L33-L49
```solidity
   function init(string memory _name, string memory _symbol) external override reinitializer(2) {
       __ERC20Permit_init(_name);
       __ERC20_init(_name, _symbol);
       __ReentrancyGuard_init();
       __UUPSUpgradeable_init();
       __Ownable_init();


       IERC20[] memory _tokens = tokens();
       uint256 tokensLength = _tokens.length;
       for (uint256 i; i < tokensLength - 1; ++i) {
           for (uint256 j = i + 1; j < tokensLength; ++j) {
               if (_tokens[i] == _tokens[j]) {
                   revert DuplicateTokens(_tokens[i]); // found
               }
           }
       }
   }
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L310-L325
```solidity
   function decodeWellData(bytes memory data) public view virtual returns (uint256[] memory decimals) {
       (uint256 decimal0, uint256 decimal1) = abi.decode(data, (uint256, uint256));


       // if well data returns 0, assume 18 decimals.
       if (decimal0 == 0) {
           decimal0 = 18;
       }
       if (decimal0 == 0) {
           decimal1 = 18;
       }
       if (decimal0 > 18 || decimal1 > 18) revert InvalidTokenDecimals(); // Found


       decimals = new uint256[](2);
       decimals[0] = decimal0;
       decimals[1] = decimal1;
   }
```
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L114-L144
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
       (uint256 c, uint256 b) = getBandC(a * N * N, lpTokenSupply, j == 0 ? scaledReserves[1] : scaledReserves[0]);
       reserve = lpTokenSupply;
       uint256 prevReserve;


       for (uint256 i; i < 255; ++i) {
           prevReserve = reserve;
           reserve = _calcReserve(reserve, b, c, lpTokenSupply);
           // Equality with the precision of 1
           // scale reserve down to original precision
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
       revert("did not find convergence"); // Found
   }
```

### Tools Used

Manual Analysis

## 4: Functions which are either private or internal should have a preceding _ in their name

Vulnerability details

### Context:

Add a preceding underscore to the function name, and take care to refactor where there functions are called

### Proof of Concept

> ***1 File, 3 Instances*** 

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L357-L360

```
   function getScaledReserves(
       uint256[] memory reserves,
       uint256[] memory decimals
   ) internal pure returns (uint256[] memory scaledReserves) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L375-L379
```
   function getBandC(
       uint256 Ann,
       uint256 lpTokenSupply,
       uint256 reserves
   ) private pure returns (uint256 c, uint256 b) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L387
```
   function updateReserve(PriceData memory pd, uint256 reserve) internal pure returns (uint256) {
```

### Tools Used

Manual Analysis


## 5: Contracts should have all public/external functions exposed by interfaces

### Resolution 
Contracts should expose all public and external functions through interfaces. This practice ensures a clear and consistent definition of how the contract can be interacted with, promoting better transparency and integration.

### Proof of Concept

> ***3 Files, 19 instances***

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L93
```solidity
	   function upgradeTo(address newImplementation) public override {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L104
```
   function upgradeToAndCall(address newImplementation, bytes memory data) public payable override {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L33
```
	function init(string memory _name, string memory _symbol) external override reinitializer(2) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L54
```
	function initNoWellToken() external initializer {}
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L118

```
	function proxiableUUID() external view override notDelegatedOrIsMinimalProxy returns (bytes32) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L122
```
	function getImplementation() external view returns (address) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L126
```
	function getVersion() external pure virtual returns (uint256) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L130
```
	function getInitializerVersion() external view returns (uint256) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L173-L179
```solidity
	function calcReserveAtRatioSwap(
    	uint256[] memory reserves,
    	uint256 j,
    	uint256[] memory ratios,
    	bytes calldata data
	) external view returns (uint256 reserve) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L246-L251
```
	function calcReserveAtRatioLiquidity(
    	uint256[] calldata reserves,
    	uint256 j,
    	uint256[] calldata ratios,
    	bytes calldata data
	) external view returns (uint256 reserve) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L327
```
function name() external pure returns (string memory) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L331
```
	function symbol() external pure returns (string memory) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L74-L77
```
	function calcLpTokenSupply(
    	uint256[] memory reserves,
    	bytes memory data
	) public view returns (uint256 lpTokenSupply) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L114-L119
```
	function calcReserve(
    	uint256[] memory reserves,
    	uint256 j,
    	uint256 lpTokenSupply,
    	bytes memory data
	) public view returns (uint256 reserve) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L153-L158
```
	function calcRate(
    	uint256[] memory reserves,
    	uint256 i,
    	uint256 j,
    	bytes memory data
	) public view returns (uint256 rate) {
```
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L310
```
	function decodeWellData(bytes memory data) public view virtual returns (uint256[] memory decimals) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/StableLUT/Stable2LUT1.sol#L19
```
	function getAParameter() external pure returns (uint256) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/StableLUT/Stable2LUT1.sol#L27
```
	function getRatiosFromPriceLiquidity(uint256 price) external pure returns (PriceData memory) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/StableLUT/Stable2LUT1.sol#L740
```
	function getRatiosFromPriceSwap(uint256 price) external pure returns (PriceData memory) {
```

### Tools Used

Manual Analysis

## 6: Functions within contracts are not ordered according to the solidity style guide

Vulnerability details

### Context:

The following order should be used within contracts

constructor

receive function (if exists)

fallback function (if exists)

external

public

internal

private

Rearrange the contract functions to fit this ordering

### Proof of Concept

> ***2 Files*** 

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L25

```
contract Stable2 is ProportionalLPToken2, IBeanstalkWellFunction {internal pure returns (uint256[] memory scaledReserves) {
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L16 
```
contract WellUpgradeable is Well, UUPSUpgradeable, OwnableUpgradeable {
```

### Tools Used

Manual Analysis

## 7: Initialize functions do not emit an event

Vulnerability details

### Context:

Emitting an event within initializer functions in Solidity is a best practice for providing transparency and traceability. Initializer functions set the initial state and values of an upgradeable contract. Emitting an event during initialization allows anyone to verify and audit the initial state of the contract via the transaction logs. This can be particularly useful for verifying the parameters set during initialization, tracking the contract's deployment, and troubleshooting or debugging. Therefore, developers should include an event emission in their initializer functions, providing a clear record of the contract's initialization and enhancing the contract's transparency and security.


### Proof of Concept

> ***Num of Instances: 1*** 

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L33
```solidity
		function init(string memory _name, string memory _symbol) external override reinitializer(2) {
    	__ERC20Permit_init(_name);
    	__ERC20_init(_name, _symbol);
    	__ReentrancyGuard_init();
    	__UUPSUpgradeable_init();
    	__Ownable_init();

    	IERC20[] memory _tokens = tokens();
    	uint256 tokensLength = _tokens.length;
    	for (uint256 i; i < tokensLength - 1; ++i) {
        	for (uint256 j = i + 1; j < tokensLength; ++j) {
            	if (_tokens[i] == _tokens[j]) {
                	revert DuplicateTokens(_tokens[i]);
            	}
        	}
    	}
	}
```

### Tools Used

Manual Analysis

## 8: Empty bytes check is missing

Vulnerability details

### Context:

When developing smart contracts in Solidity, it's crucial to validate the inputs of your functions. This includes ensuring that the bytes parameters are not empty, especially when they represent crucial data such as addresses, identifiers, or raw data that the contract needs to process.

Missing empty bytes checks can lead to unexpected behavior in your contract. For instance, certain operations might fail, produce incorrect results, or consume unnecessary gas when performed with empty bytes. Moreover, missing input validation can potentially expose your contract to malicious activity, including exploitation of unhandled edge cases.


### Proof of Concept

> ***Num of Instances: 1*** 

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L310C1-L325C6 
```solidity
	function decodeWellData(bytes memory data) public view virtual returns (uint256[] memory decimals) {
    	(uint256 decimal0, uint256 decimal1) = abi.decode(data, (uint256, uint256));

    	// if well data returns 0, assume 18 decimals.
    	if (decimal0 == 0) {
        	decimal0 = 18;
    	}
    	if (decimal0 == 0) {
        	decimal1 = 18;
    	}
    	if (decimal0 > 18 || decimal1 > 18) revert InvalidTokenDecimals();

    	decimals = new uint256[](2);
    	decimals[0] = decimal0;
    	decimals[1] = decimal1;
	}
```

### Tools Used

Manual Analysis

### Recommended Mitigation

To mitigate these issues, always validate that bytes parameters are not empty when the logic of your contract requires it.


# Low Impact Vulnerabilities

## 1: Missing checks for address(0x0) when updating address state variables 

Num of instances: 2

### Findings 


<details><summary>Click to show findings</summary>

['[93](https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L93-L96)']
```solidity
           function upgradeTo(address newImplementation) public override {
       _authorizeUpgrade(newImplementation);
       _upgradeToAndCallUUPS(newImplementation, new bytes(0), false);
   }

```
[‘[104](https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L104-L107)’]

```solidity
   function upgradeToAndCall(address newImplementation, bytes memory data) public payable override {
       _authorizeUpgrade(newImplementation);
       _upgradeToAndCallUUPS(newImplementation, data, true);
   }
```


</details>

## 2: Solidity version 0.8.23 won't work on all chains due to MCOPY

Vulnerability details

### Context:

Solidity version 0.8.23 introduces the MCOPY opcode, this may not be implemented on all chains and L2 thus reducing the portability and compatibility of the code. Consider using an earlier solidity version.

### Proof of Concept

> ***3 Files, 3 Instances*** 

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L3
```solidity
	pragma solidity ^0.8.20;
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L3
```solidity
	pragma solidity ^0.8.20;
```

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/StableLUT/Stable2LUT1.sol#L3
```solidity
	pragma solidity ^0.8.20;
```

### Tools Used

Manual Analysis

### Recommended Mitigation Steps

Consider using an earlier solidity version.

