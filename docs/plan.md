# 刷题记录

## 动态规划

### 1维dp

#### 复杂度$O(n)$

这类题的当前状态由之前的常数个状态转移而来，因此时间复杂度在$O(n)$，典型的题即

!!! example
	- 最大字段和MIS: 第i个状态由第i-1个状态转移来
	- 图像压缩: 第i个状态至多由前256个状态转移而来

以[53. 最大子数组和 - 力扣](https://leetcode.cn/problems/maximum-subarray/) MIS为例，实际代码如下：

```C++
int maxSubArray(vector<int>& nums) {
    // dp[i] = if(dp[i-1]<0){ nums[i] } else{ dp[i-1]+nums[i] }
    int n = nums.size();
    int pre_sum{-1}, res{INT32_MIN};
    for(int i=0;i<n;i++){
        pre_sum = pre_sum < 0 ? nums[i] : pre_sum + nums[i];
        res = max(res, pre_sum);
    }
    return res;
}
```




- [509. 斐波那契数 - 力扣](https://leetcode.cn/problems/fibonacci-number/) $dp[i] = dp[i-1]+dp[i-2]$；

- [70. 爬楼梯 - 力扣](https://leetcode.cn/problems/climbing-stairs/) 与斐波那契相同，$dp[i] = dp[i-1]+dp[i-2]$；

- [746. 使用最小花费爬楼梯 - 力扣](https://leetcode.cn/problems/min-cost-climbing-stairs/) $dp[i] = min(dp[i-1], dp[i-2])$；

- [53. 最大子数组和 - 力扣](https://leetcode.cn/problems/maximum-subarray/) MIS，$dp[i] = \texttt{if}(dp[i-1]<0)\quad\{nums[i]\}\quad\texttt{else}\quad\{dp[i-1]+nums[i] \}$

- [152. 乘积最大子数组- 力扣](https://leetcode.cn/problems/maximum-product-subarray/solution/) 类似MIS，但是需要注意转移方程的不同:

  ```python
  dp_max[i] = max(dp_max[i-1]*nums[i], dp_min[i-1]*nums[i], nums[i])
  dp_min[i] = min(dp_max[i-1]*nums[i], dp_min[i-1]*nums[i], nums[i])
  ```

- [1567. 乘积为正数的最长子数组长度- 力扣](https://leetcode.cn/problems/maximum-length-of-subarray-with-positive-product/solution/) 类似MIS，但是需要这题求长度，注意dp数组含义的改变以及转移方程相应调整

  ```python
  dp_pos[i] = if(nums[i]==0)  { 0 }
              elif(nums[i]<0) { dp_neg[i-1]!=0 ? dp_neg[i-1]+1 : 0 }
              elif(nums[i]>0) { dp_pos[i-1]+1 }
  dp_neg[i] = if(nums[i]==0)  { 0 }
              elif(nums[i]<0) { dp_neg[i-1]+1 }
              elif(nums[i]>0) { dp_neg[i-1]!=0 ? dp_neg[i-1]+1 : 0}
  ```

- [918. 环形子数组的最大和 - 力扣 ](https://leetcode.cn/problems/maximum-sum-circular-subarray/) MIS的变种
  考虑两种情况，第一种情况是最大连续子数组在数组中间，这种类似于53求最大连续子数组；
  第二种情况是最大连续子数组在数组两边，这就需要求出最小连续子数组，然后用数组和减去最小连续子数组；
  
- [198. 打家劫舍 - 力扣](https://leetcode.cn/problems/house-robber/) $dp[i] = max(dp[i-1], dp[i-2]+val[i-2])$
  
- [213. 打家劫舍 II - 力扣](https://leetcode.cn/problems/house-robber-ii/) 对于环形数组分情况讨论（偷第1个 or 不偷第1个）

- [740. 删除并获得点数 - 力扣](https://leetcode.cn/problems/delete-and-earn/) 打家劫舍的变体，通过一些技巧转化为打家劫舍

- [121. 买卖股票的最佳时机 - 力扣](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock/) 1维dp，$f(k) = max(f(k-1),v_k - v_{pre\_min})$

- [122. 买卖股票的最佳时机 II - 力扣](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-ii/submissions/) 2维dp：
  $f(k,0) = \max\{f(k-1,0),f(k-1,1)+v_k\}$

  $f(k,1) = \max\{f(k-1,1),f(k-1,0)-v_k\}$ 

  递推边界条件：$f(0,0) = 0,f(0,1) = -v_0$

- [309. 最佳买卖股票时机含冷冻期 - 力扣](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-with-cooldown/) 

- [714. 买卖股票的最佳时机含手续费 - 力扣](https://leetcode.cn/problems/best-time-to-buy-and-sell-stock-with-transaction-fee/) 与122相同，多了手续费，对递推关系稍加修改即可

### 2维dp

#### 背包：

这里把背包作为单独一种类型拉出来，这类题有固定套路：



- [416. 分割等和子集](https://leetcode.cn/problems/partition-equal-subset-sum/)；0-1背包恰好装满
- [474. 一和零](https://leetcode-cn.com/problems/ones-and-zeroes)；0-1背包最大价值3维
- [494. 目标和 ](https://leetcode.cn/problems/target-sum/)；
- [879. 盈利计划 ](https://leetcode.cn/problems/profitable-schemes/)；
- [1049. 最后一块石头的重量 II](https://leetcode.cn/problems/last-stone-weight-ii/)；转化为0-1背包尽可能多装的问题
- [1230. 抛掷硬币](https://leetcode.cn/problems/toss-strange-coins/)；
- [322. 零钱兑换](https://leetcode-cn.com/problems/coin-change)；完全背包恰好装满
- [518. 零钱兑换 II ](https://leetcode-cn.com/problems/coin-change-2)；完全背包输出方案总数
- [1449. 数位成本和为目标值的最大数字](https://leetcode.cn/problems/form-largest-integer-with-digits-that-add-up-to-target/)；
- [279. 完全平方数](https://leetcode.cn/problems/perfect-squares/)；完全背包



