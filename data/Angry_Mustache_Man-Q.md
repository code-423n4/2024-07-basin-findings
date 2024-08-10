## `calcLpTokenSupply` does not have the revertion mechanism implemented for the rare edge case that can happen . 
### Vulnerability Details
`Stable2.sol#calcLpTokenSupply` function is inspired from Curve's `get_D` function from this contract - 
https://github.com/curvefi/curve-stablecoin/blob/master/contracts/Stableswap.vy#L385C1-L418C10

Here we can see that, at the end of execution there is a revert(or raise in Vyper) implemented to prevent the rare case where convergent doesn't happen and the contract using this code becomes borked . 

Similar revertion can be seen in the calculation of Basin's Stable2.sol#calcReserve & also in Curve's Stableswap.vy
https://github.com/code-423n4/2024-07-basin/blob/main/src/functions/Stable2.sol#L142C2-L144C6  

https://github.com/curvefi/curve-stablecoin/blob/master/contracts/Stableswap.vy#L730C2-L732C10

### Mitigation Recommendation
Implement the revert in `Stable2.sol#calcLpTokenSupply` function to prevent return of unexpected values during rare edge cases.
