### 找零钱问题
如果用穷举法来解题的话，本质上它就是一个多叉树的遍历  
假设总金额是amount，零钱coins用vector来表示，示例amount=11, vector = {1,2,5}  
那么问题dp(11)的解其实就是1 + min(dp(10), dp(9), dp(6))  
我们不用去考虑dp(10),dp(9),dp(6)是多少，怎么算的，计算机会帮你去最终计算出来，我们现在就是假设这个值是已知的  
那么上面的步骤的写成代码就是  
```
coins = [1,2,5]
def dp(n):
    res = 99999
    for coin in coins:
        res = min(res, 1 + dp(n-coin)) #res就相当于dp(10), dp(9), dp(6)
    return res
```
这类问题是具有最优子结构的，也就是说上述的dp(10), dp(9), dp(6)都是当n分别为10,9,6时的最优解。

