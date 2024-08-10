## Initialization not disabled on minimal proxy allows minimal proxy to be re-initialized and owned

## Proof of Concept

`WellUpgradeable` uses 2 proxies a ERC1967 proxy and a EIP1167 minimal proxy ,with the flow: 1967proxy -> minimal proxy -> implementation. The minimal proxy enables compatibility with protocol design pattern and upgradeability. The issue is that the `initNoWellToken` function used to initialize the minimal proxy(this is for compatibility with the Aquifer which is OOS) uses the `initializer` modifier and the `init` function used to initialize the well uses `reinitializer(2)` 
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

    /**
     * @notice `initNoWellToken` allows for the Well to be initialized without deploying a Well token.
     */
    function initNoWellToken() external initializer {} 
```

This allows the minimal proxy to be re-initialized and owned by calling `init`.

## Tools Used

Manual Review

## Recommended Mitigation Steps

It's better to use the `_disableInitializers` function for `initNoWellToken`, this still maintains compatibility with the Aquifer but prevents further initialization.
The `init` function can also use a regular initializer.

```solidity
 function init(string memory _name, string memory _symbol) external override initializer {
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

    /**
     * @notice `initNoWellToken` allows for the Well to be initialized without deploying a Well token.
     */
    function initNoWellToken() external  {
        _disableInitializers();
    }
```

## Mismatch between implementation and documentation of `notDelegatedOrIsMinimalProxy` modifier

## Proof of Concept

The documentation of `notDelegatedOrIsMinimalProxy` suggests that the modifier should allow non-delegated Calls, However the function implementation does not allow this.

```solidity
    /**
     * @notice verifies that the execution is called through an minimal proxy or is not a delegate call.
     */
    modifier notDelegatedOrIsMinimalProxy() {
        if (address(this) != ___self) {
            address aquifer = aquifer();
            address wellImplmentation = IAquifer(aquifer).wellImplementation(address(this));
            require(wellImplmentation == ___self, "Function must be called by a Well bored by an aquifer");
        } else {
            revert("UUPSUpgradeable: must not be called through delegatecall");
        }
        _;
    }
```

From the above code ,the function only allows calls from minimal proxy.

## Tools Used

Manual Review

## Recommended Mitigation Steps

```diff
    /**
     * @notice verifies that the execution is called through an minimal proxy or is not a delegate call.
     */
    modifier notDelegatedOrIsMinimalProxy() {
        if (address(this) != ___self) {
            address aquifer = aquifer();
            address wellImplmentation = IAquifer(aquifer).wellImplementation(address(this));
            require(wellImplmentation == ___self, "Function must be called by a Well bored by an aquifer");
        } else {
-           revert("UUPSUpgradeable: must not be called through delegatecall");
        }
        _;
    }
```

## Extreme price crash of pair token can Dos Season flip on beanstalk

## Proof of Concept

Based on information provided by protocol team `Stable2::calcReserveAtRatioSwap` is called [here](https://github.com/BeanstalkFarms/Beanstalk/blob/8bd11a21f753c36b7449e2189a1a84306126bbf4/protocol/contracts/libraries/Minting/LibWellMinting.sol#L182C1-L195C6) at the start of each season to determine `deltaB` - the amount of beans to buy or sell to maintain peg. The lookup table `Stable2LUT1::getRatiosFromPriceSwap` only provides values for price ratio within the range of 10 to 0.001 and reverts if the price ratio is outside of this range, DOSing the season flip.

**important things to note**
1.Stable2 is only used for bean/stableCoin pairs ,the price ratio is calculated as (price of stableCoin)/(price of beans) and the system internally assumes the price of bean as 1 , so price ratio = price of stableCoin.

2.Even though it's extremely unlikely(but not impossible) that the price of the paired stableCoin crashes below 0.001 - one precedence is LUNA. It should be avoided if possible 

## Tools Used

Manual Review

## Recommended Mitigation Steps

```diff
 function calcReserveAtRatioSwap(
        uint256[] memory reserves,
        uint256 j,
        uint256[] memory ratios,
        bytes calldata data
    ) external view returns (uint256 reserve) {
        uint256 i = j == 1 ? 0 : 1;
        // scale reserves and ratios:
        uint256[] memory decimals = decodeWellData(data);
        uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);

        PriceData memory pd;
        uint256[] memory scaledRatios = getScaledReserves(ratios, decimals);
        // calc target price with 6 decimal precision:
        pd.targetPrice = scaledRatios[i] * PRICE_PRECISION / scaledRatios[j];

        // get ratios and price from the closest highest and lowest price from targetPrice:
-       pd.lutData = ILookupTable(lookupTable).getRatiosFromPriceSwap(pd.targetPrice);
+       try ILookupTable(lookupTable).getRatiosFromPriceSwap(pd.targetPrice) returns(ILookupTable.PriceData memory lutData) {
+           pd.lutData = lutData;
+       } catch {
+           return reserves[j]; // so deltaB will be zero
+       }

        //...omitted code...
    }
```