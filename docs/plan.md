# 刷题记录

## 动态规划

- [hdu 2084 数塔](http://acm.hdu.edu.cn/showproblem.php?pid=2084) 可以从上往下递推节省空间，也可以从下往上递推，代码更好写：$dp[i-1][j] = cur + max(dp[i][j],dp[i][j+1])$

- [hdu 2018 母牛的故事](http://acm.hdu.edu.cn/showproblem.php?pid=2018) 简单递推计数: $dp[i] = dp[i-1]+dp[i-3]$, $dp[1] = 1,dp[2]=2,dp[3] =3$

- [hdu 2044 一只小蜜蜂...](http://acm.hdu.edu.cn/showproblem.php?pid=2044) 简单递推计数(Fibonacci): $dp[i][j] = dp[i][j-1]+dp[i][j-2]$, $dp[i][i+1] = 1, dp[i][i+2] = 2$

- [hdu 2041 超级楼梯](http://acm.hdu.edu.cn/showproblem.php?pid=2041) Fibonacci : 

- [53. 最大子数组和 - 力扣](https://leetcode.cn/problems/maximum-subarray/) MIS

- [152. 乘积最大子数组- 力扣](https://leetcode.cn/problems/maximum-product-subarray/solution/) 类似MIS，但是需要注意转移方程的不同

- [1567. 乘积为正数的最长子数组长度- 力扣](https://leetcode.cn/problems/maximum-length-of-subarray-with-positive-product/solution/) 类似MIS，但是需要注意转移方程的不同+分类讨论

- [918. 环形子数组的最大和 - 力扣 ](https://leetcode.cn/problems/maximum-sum-circular-subarray/) MIS的变种
  考虑两种情况，第一种情况是最大连续子数组在数组中间，这种类似于53求最大连续子数组；
  第二种情况是最大连续子数组在数组两边，这就需要求出最小连续子数组，然后用数组和减去最小连续子数组；
  如果说这数组的所有数都是负数——

- [213. 打家劫舍 II - 力扣](https://leetcode.cn/problems/house-robber-ii/) 考虑是否偷当前物品$\Longrightarrow$ 不偷转移到$f(k-1)$，偷转移到$f(k-2)$，对于环形数组分情况讨论

- [740. 删除并获得点数 - 力扣](https://leetcode.cn/problems/delete-and-earn/) 打家劫舍的变体，通过一些技巧转化为打家劫舍

- [121. 买卖股票的最佳时机 - 力扣](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock/) 1维dp，$f(k) = max(f(k-1),v_k - v_{pre\_min})$

- [122. 买卖股票的最佳时机 II - 力扣](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-ii/submissions/) 2维dp：
  $f(k,0) = \max\{f(k-1,0),f(k-1,1)+v_k\}$

  $f(k,1) = \max\{f(k-1,1),f(k-1,0)-v_k\}$ 

  递推边界条件：$f(0,0) = 0,f(0,1) = -v_0$

- [309. 最佳买卖股票时机含冷冻期 - 力扣](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-with-cooldown/) 

- [714. 买卖股票的最佳时机含手续费 - 力扣](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-with-transaction-fee/) 与122相同，多了手续费，对递推关系稍加修改即可

**背包：**

- [416. 分割等和子集](https://leetcode.cn/problems/partition-equal-subset-sum/)；0-1背包恰好装满
- [474. 一和零](https://leetcode-cn.com/problems/ones-and-zeroes)；0-1背包最大价值3维
- [494. 目标和 ](https://leetcode.cn/problems/target-sum/)；
- [879. 盈利计划 ](https://leetcode.cn/problems/profitable-schemes/)；
- [1049. 最后一块石头的重量 II](https://leetcode.cn/problems/last-stone-weight-ii/)；转化为0-1背包尽可能多装的问题
- [1230. 抛掷硬币](https://leetcode.cn/problems/toss-strange-coins/)；
- 
- [322. 零钱兑换](https://leetcode-cn.com/problems/coin-change)；完全背包恰好装满
- [518. 零钱兑换 II ](https://leetcode-cn.com/problems/coin-change-2)；完全背包输出方案总数
- [1449. 数位成本和为目标值的最大数字](https://leetcode.cn/problems/form-largest-integer-with-digits-that-add-up-to-target/)；
- [279. 完全平方数](https://leetcode.cn/problems/perfect-squares/)；完全背包



