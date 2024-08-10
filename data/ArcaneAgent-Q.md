## 1. Converging solution formula described in function helper is not same as implementation.

In helper string [here](https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L72), converging solution is described as following.
```
D[j+1] = (4 * A * sum(b_i) - (D[j] ** 3) / (4 * prod(b_i))) / (4 * A - 1)       [1]
``` 
Invariant condition is as following:
$$An^n\sum x_i + D = DAn^n+D^{(n+1)}/(n^n\prod x_i)$$
```
A * sum(x_i) * n**n + D = A * D * n**n + D**(n+1) / (n**n * prod(x_i))
```

@Brean shared an article how someone derived D[j+1] from above invariant condition [here](https://atulagarwal.dev/posts/curveamm/stableswap/).
$$D_{(n+1)}=D_n-f(D_n)/f'(D_n)=(AnnS+nD_p)D_n/((Ann-1)D_n+(n+1)D_p)$$
where $$S=\sum x_i$$,  $$Ann=An^n$$,       $$D_p=D^{(n+1)}/(n^n \prod x_i)$$

As $$N=2$$, above equation can be simplified as following:
```
D[j+1]=(4*A*sum(b_i)+2*D_p)*D[j] / ( (4 * A - 1)*D[j] + (2+1)*D_p )       [2]
```  
Implementation [here](https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L94) is not same as [1], but same to [2]. 

#### Recommended Mitigation Steps
```diff
     * @notice Calculate the amount of lp tokens given reserves.
     * D invariant calculation in non-overflowing integer operations iteratively
     * A * sum(x_i) * n**n + D = A * D * n**n + D**(n+1) / (n**n * prod(x_i))
     *
     * Converging solution:
-    * D[j+1] = (4 * A * sum(b_i) - (D[j] ** 3) / (4 * prod(b_i))) / (4 * A - 1)
+    * D[j+1]=(4*A*sum(b_i)+2*D_p)*D[j] / ( (4 * A - 1)*D[j] + (2+1)*D_p )
```


## 2. Ann calculation is specific case.
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L83
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L124
```
uint256 Ann = a * N * N;
```
It only works for the pool with 2 tokens (N=2), because $$Ann=An^n$$.
There should be a comment for this simplification.

## 3. decimal1 validation is missing.
https://github.com/code-423n4/2024-07-basin/blob/7d5aacbb144d0ba0bc358dfde6e0cc913d25310e/src/functions/Stable2.sol#L317

#### Recommended Mitigation Steps
```diff
        // if well data returns 0, assume 18 decimals.
        if (decimal0 == 0) {
            decimal0 = 18;
        }
-       if (decimal0 == 0) {
+       if (decimal1 == 0) {
            decimal1 = 18;
        }
```

## Tools Used
Manual review, Foundry
