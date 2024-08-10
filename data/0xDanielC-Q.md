Incorrect decimals return value in decodeWellData function. 

https://github.com/code-423n4/2024-07-basin/blob/main/src/functions/Stable2.sol#L317

```decima1``` is given the value ```18``` if ```decimal0 == 0```, this could lead to an incorrect amount of decimals being returned, for example, decimals would equal 0 for a token that has 6 decimals due to the incorrect if statement. 

This could lead to later negative downstream effects such as incorrect rate calculations. 