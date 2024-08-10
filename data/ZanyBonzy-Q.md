
### 1. Slight deviations in PriceData returned in LookUpTable.sol

Links to affected code *

#### Impact

Some of the values in LookupTable offer slight deviations upon testing. 

For ease, i have rearranged the table [here](https://gist.github.com/ZanyBonzy/b8999be4e8526ed5845ac5a88454d06b);
And by adding the test below to Stable2.t.sol and running it.
It handles the case below, 

```solidity
(price < 0.98038e6) {return PriceData(0.98038e6, 0, 1.061208000000000151e18, 0.973914e6, 0, 1.082432159999999977e18, 1e18);}
```

For high values, we use `0.98038e6, 0, 1.061208000000000151e18`
```solidity
    function test_calcRateStableA() public { 
        _data = abi.encode(18, 18); 
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 1e18; //in place of 0
        reserves[1] = 1.061208000000000151e18;
        assertEq(_function.calcRate(reserves, 0, 1, _data), 0.98038e6); 
    } 
```

For low values we use `0.973914e6, 0, 1.082432159999999977e18,` 
```solidity
    function test_calcRateStableB() public { 
        _data = abi.encode(18, 18); 
        uint256[] memory reserves = new uint256[](2);
        reserves[0] = 1e18; //in place of 0
        reserves[1] = 1.082432159999999977e18;
        assertEq(_function.calcRate(reserves, 0, 1, _data), 0.973914e6); 
    } 
```

We can see that the `test_calcRateStableA` reverts as the returned rate slightly differs from the expected value. `test_calcRateStableB` passes instead.

```md
  [97479] Stable2Test::test_calcRateStableA()
    ├─ [20943] Stable2::calcRate([1000000000000000000 [1e18], 1061208000000000151 [1.061e18]], 0, 1, 0x00000000000000000000000000000000000000000000000000000000000000120000000000000000000000000000000000000000000000000000000000000012) [staticcall]
    │   └─ ← [Return] 980381 [9.803e5]
    ├─ [0] VM::assertEq(980381 [9.803e5], 980380 [9.803e5]) [staticcall]
    │   └─ ← [Revert] assertion failed: 980381 != 980380
    └─ ← [Revert] assertion failed: 980381 != 980380

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.38ms (203.83µs CPU time)

Ran 1 test suite in 154.86ms (1.38ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/functions/Stable2.t.sol:Stable2Test
[FAIL. Reason: assertion failed: 980381 != 980380] test_calcRateStableA() (gas: 97479)
```
#### Recommended Mitigation Steps

I'd recommend more recallibrations of the returned values. As the deviations seem to be slight, it may not pose a lot of problems. Just thought it was worth mentioning.

***


### 2. Redundant `sumReserves == 0` check in `calcLpTokenSupply`

Links to affected code *

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L86

#### Impact

The check for reserves in `calcLpTokenSupply` ensures that if `reserve[0]` and `reserve[1]` are both 0, the function safely returns 0. The function then checks again if sum of reserves, `sumReserves` == 0. Since the `reserve` parameters are uint256, and scaling doesn't multiply by 0, the `sumReserves == 0` check can be safely removed.

```solidity
        if (reserves[0] == 0 && reserves[1] == 0) return 0;
        uint256[] memory decimals = decodeWellData(data);
        // scale reserves to 18 decimals.
        uint256[] memory scaledReserves = getScaledReserves(reserves, decimals);

        uint256 Ann = a * N * N;

        uint256 sumReserves = scaledReserves[0] + scaledReserves[1];
        if (sumReserves == 0) return 0;
        lpTokenSupply = sumReserves;
```


#### Recommended Mitigation Steps

Recommend removing the check since its not needed.

***

### 3. Wrong calculation used for `Ann` in comparison to Stableswap implementations

Links to affected code *

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L83

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L123-L124

#### Impact

The `Ann` parameter is wrongly calculated as a * N * N, when it should be A * N

```solidity
uint256 Ann = a * N * N;
```

```solidity
  (uint256 c, uint256 b) = getBandC(a * N * N, lpTokenSupply, j == 0 ? scaledReserves[1] : scaledReserves[0]);
```

#### Recommended Mitigation Steps

See the implemetation in in [Stableswap](https://github.com/curvefi/curve-stablecoin/blob/176e637a4b82874c49c90883ee1afc7e317ca051/contracts/Stableswap.vy#L404)

***

### 4. Operator comparison can use == rather than <=

Links to affected code *

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L97-99

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L133-135

#### Impact

In `calcLpTokenSupply`, the comparsion between `lpTokenSupply` and `prevReserves` is made, before checking if the difference is <= 1. Since the comparison uses the greater than operator, it can be assumed that the difference will not be 0, rather always greater than 0. So the check for difference being less than or equal to one can be safely set as equal to 1 instead.

```solidity
            if (lpTokenSupply > prevReserves) {
                if (lpTokenSupply - prevReserves <= 1) return lpTokenSupply;
            } 
```

The same can be observed in `calcReserve` where `reserve` is compared to `prevReserve`. As can be seen, since `reserve` is greater than `prevReserve`, the difference can't be 0, therefore, the difference can never be less than 1, so it can the equal to operator can be used safely.

```solidity
            if (reserve > prevReserve) {
                if (reserve - prevReserve <= 1) {
                    return reserve / (10 ** (18 - decimals[j]));
                }
```

#### Recommended Mitigation Steps

Recommend these changes

```diff
            if (lpTokenSupply > prevReserves) {
-                if (lpTokenSupply - prevReserves <= 1) return lpTokenSupply;
+                if (lpTokenSupply - prevReserves == 1) return lpTokenSupply;
            } 
```

```diff
            if (reserve > prevReserve) {
-                if (reserve - prevReserve <= 1) {
+                if (reserve - prevReserve == 1) {
                    return reserve / (10 ** (18 - decimals[j]));
                }
```
***

### 5. Condition that is never hit due to LUT can be removed

Links to affected code *

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L214-L218

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L280-L286

#### Impact

In `calcReserveAtRatioSwap` this condition exists to calculate `maxStepSize`. It makes comparisons between `pd.lutData.lowPriceJ` and `pd.lutData.highPriceJ` prices. But by going through the LUT and the `getRatiosFromPriceSwap` function (use the link [here](https://gist.github.com/ZanyBonzy/b8999be4e8526ed5845ac5a88454d06b) for a more convenient view), we can see that `pd.lutData.lowPriceJ` is always greater than `pd.lutData.highPriceJ`, never equal to and never less. So the else component is never hit, as a result can be safely removed.

```solidity
        if (pd.lutData.lowPriceJ > pd.lutData.highPriceJ) {
            pd.maxStepSize = scaledReserves[j] * (pd.lutData.lowPriceJ - pd.lutData.highPriceJ) / pd.lutData.lowPriceJ;
        } else {
            pd.maxStepSize = scaledReserves[j] * (pd.lutData.highPriceJ - pd.lutData.lowPriceJ) / pd.lutData.highPriceJ;
        }
```

The same can be observed in `calcReserveAtRatioLiquidity` and by going through the `getRatiosFromPriceLiquidity` in the LUT, the else condition is never hit and can also be safely removed.

```solidity

        // calculate max step size:
        if (pd.lutData.lowPriceJ > pd.lutData.highPriceJ) {
            pd.maxStepSize = scaledReserves[j] * (pd.lutData.lowPriceJ - pd.lutData.highPriceJ) / pd.lutData.lowPriceJ;
        } else {
            pd.maxStepSize = scaledReserves[j] * (pd.lutData.highPriceJ - pd.lutData.lowPriceJ) / pd.lutData.highPriceJ;
        }
```

#### Recommended Mitigation Steps

The if else check can be removed.

```diff
-        if (pd.lutData.lowPriceJ > pd.lutData.highPriceJ) {
            pd.maxStepSize = scaledReserves[j] * (pd.lutData.lowPriceJ - pd.lutData.highPriceJ) / pd.lutData.lowPriceJ;
-        } else {
-            pd.maxStepSize = scaledReserves[j] * (pd.lutData.highPriceJ - pd.lutData.lowPriceJ) / pd.lutData.highPriceJ;
-        }
```

```diff
-        if (pd.lutData.lowPriceJ > pd.lutData.highPriceJ) {
            pd.maxStepSize = scaledReserves[j] * (pd.lutData.lowPriceJ - pd.lutData.highPriceJ) / pd.lutData.lowPriceJ;
-        } else {
-            pd.maxStepSize = scaledReserves[j] * (pd.lutData.highPriceJ - pd.lutData.lowPriceJ) / pd.lutData.highPriceJ;
-        }
```

***

### 6. Amplification coefficient is hardcoded and set as immutable and therefore cannot be changed.

Links to affected code *


https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L47

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/StableLUT/Stable2LUT1.sol#L20


#### Impact

The amplification coefficient is hardcoded as 100 in Stable2LUT1.sol
```solidity
    function getAParameter() external pure returns (uint256) {
        return 100;
    }
```
And is also set as immutable in Stable2.sol, meaning it's set in the constructor and cannot be changed.
```solidity

    uint256 immutable a;
```

In StableSwap implementations, the amplification coefficient helps to handle a pool’s slippage and tolerance for imbalance between the assets within it. It can help liquidity providers to deposit assets in the pool by adjusting the trading fees based on the pool's liquidity. 

Not being able to change the coefficient can leave the pool unable to adapt to drastic changes in market conditions inclding increased volatility and variations in trading volume making the pool more unstale. The amplification coefficient helps protect against situations like that, and therefore being able to adjust it to accumulate for these changes is quite important.

#### Recommended Mitigation Steps

I'd recommend introducing an admin function to change the amplification coefficient.

***


***

### 7. Potential for precision loss in `getBandC` duue to division before multiplications

Links to affected code *

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L380-L381

#### Impact

Solidity's integer division truncates, so performing division before multiplication can lead to precision loss. This is important as we're dealing with aprroximate values which need to be as close as possible to the real values.

```solidity
        c = lpTokenSupply * lpTokenSupply / (reserves * N) * lpTokenSupply * A_PRECISION / (Ann * N);
        b = reserves + (lpTokenSupply * A_PRECISION / Ann);
```
***


***

### 8. Unused inheritances can be removed

Links to affected code *

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L37-L38

#### Impact

The contract doesn't use any nonReentrant or onlyOwner modifiers. They can removed.
```solidity
    function init(string memory _name, string memory _symbol) external override reinitializer(2) {
//...
        __ReentrancyGuard_init();
//...
        __Ownable_init();
```

***


***

### 9. Contract expects ETH but has no way to recover it.

Links to affected code *

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L92-L97

https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/WellUpgradeable.sol#L107-L107
#### Impact

`upgradeToAndCall` function is payable, but doesn't seem to transfer ETH anywhere in any of the functions it calls. The contract also has no way to retrieve excess ETH sent, or potentially process refunds. Recommend reanalyzing if this is expected and adding functions to recover ETH to the contract.

```solidity
    function upgradeToAndCall(address newImplementation, bytes memory data) public payable override {
        _authorizeUpgrade(newImplementation);
        _upgradeToAndCallUUPS(newImplementation, data, true);
    }
```

For extra context, `upgradeTo` has nearly the same functionality, and is not marked payable, so if ETH is desired, no way to send it.



```solidity
     */
    function upgradeTo(address newImplementation) public override { //upgrades ito new impmentation //@note @qa should be marked payableexects eth???
        _authorizeUpgrade(newImplementation);
        _upgradeToAndCallUUPS(newImplementation, new bytes(0), false);
    }

```

***
