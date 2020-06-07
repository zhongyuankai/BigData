# 动态规划解题分析

### 买卖股票的最佳时机 II-122.

给定一个数组，它的第 i 个元素是一支给定股票第 i 天的价格。

设计一个算法来计算你所能获取的最大利润。你可以尽可能地完成更多的交易（多次买卖一支股票）。

注意：你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票）。

示例 1:

```
输入: [7,1,5,3,6,4]
输出: 7
解释: 在第 2 天（股票价格 = 1）的时候买入，在第 3 天（股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5-1 = 4 。
随后，在第 4 天（股票价格 = 3）的时候买入，在第 5 天（股票价格 = 6）的时候卖出, 这笔交易所能获得利润 = 6-3 = 3 。
```

示例 2:

```
输入: [1,2,3,4,5]
输出: 4
解释: 在第 1 天（股票价格 = 1）的时候买入，在第 5 天 （股票价格 = 5）的时候卖出, 这笔交易所能获得利润 = 5-1 = 4 。
注意你不能在第 1 天和第 2 天接连购买股票，之后再将它们卖出。
因为这样属于同时参与了多笔交易，你必须在再次购买前出售掉之前的股票。
```

示例 3:

```
输入: [7,6,4,3,1]
输出: 0
解释: 在这种情况下, 没有交易完成, 所以最大利润为 0。
```

思路分析：

**方法一：递归回溯**

1. 每天都存在持有现金和持有股票两种状态。
2. 持有现金时我们可以选择不操作和买入股票。
3. 持有股票时也可以选择不操作和卖出股票。
4. 递归出所有的方案，选出利润最大的。

代码实现：

```java
class Solution_122 {
   private int ans;
    public int maxProfit(int[] prices) {
        // 暴力法：对于每一天都有三种操作：不操作、买入、卖出。所以可以列出所有的情况选出最大的利润
       if (prices.length < 2) {
           return 0;
       }
       dfs(prices, 0, 0, 0);
       return ans;
    }

    /**
     * @param prices 每天的股价
     * @param n 代表第几天
     * @param flag 0表示持有现金，1表示持有股票
     * @param profit 利润
     */
   private void dfs(int[] prices, int n, int flag, int profit) {
       // 当n=最后一天，递归结束
       if (n == prices.length) {
          this.ans = Math.max(profit, ans);
          return;
       }
       //  不操作
       dfs(prices, n+1, flag, profit);
       if (flag == 0) { // 持有现金可以选择买入
          dfs(prices, n+1, 1, profit - prices[n]);
       } else {
          dfs(prices, n+1, 0, profit + prices[n]);
       }
   }
}
```

**方法二：贪心算法**

从第 `i` 天（这里 `i >= 1`）开始，与第 `i - 1` 的股价进行比较，如果股价有上升（严格上升），就将升高的股价（ `prices[i] - prices[i- 1]` ）记入总利润，按照这种算法，得到的结果就是符合题意的最大利润。

代码实现：

```java
public int maxProfit2(int[] prices) {
   if (prices.length < 2) {
       return 0;
   }
   int ans = 0;
   for (int i = 1; i < prices.length; i++) {
       int profit = prices[i] - prices[i-1];
       if (profit >  0) {
          ans += profit;
       }
   }
   return ans;
}
```

**方法三：动态规划**

1. 计算出每天持有现金的最大利润和持有股票的最大利润，后面的天都基于前面的天计算最大的利润。
2. `dp[i][j]`  i维表示那一天能获得的最大利润，j维表示该天持有现金还是股票。
3. 起始状态 `dp[0][0]=0, dp[0][1] = -prices[0]`
4. 状态转移方程：`dp[i][0] = Math.max(dp[i-1][0], dp[i-1][1]+prices[i])`
        `dp[i][1] = Math.max(dp[i-1][1], dp[i-1][0]-prices[i])`

代码实现：

```java
public int maxProfit3(int[] prices) {
   if (prices.length < 2) {
       return 0;
   }
   int[][] dp = new int[prices.length][2];
   dp[0][0] = 0;
   dp[0][1] = -prices[0];
   for (int i = 1; i < prices.length; i++) {
       dp[i][0] = Math.max(dp[i-1][0], dp[i-1][1]+prices[i]);
       dp[i][1] = Math.max(dp[i-1][1], dp[i-1][0]-prices[i]);
   }
   return dp[prices.length-1][0];
}
```

### 区域和检索 - 数组不可变-303

给定一个整数数组  nums，求出数组从索引 i 到 j  (i ≤ j) 范围内元素的总和，包含 i,  j 两点。

示例：

```
给定 nums = [-2, 0, 3, -5, 2, -1]，求和函数为 sumRange()
sumRange(0, 2) -> 1
sumRange(2, 5) -> -1
sumRange(0, 5) -> -3
说明:你可以假设数组不可变。会多次调用 sumRange 方法。
```

 思路分析：

1. 先计算了从数字 0 到 k 的累积和，`sum[k]`定义为 `nums[0⋯k−1]`的累积和。
2. `sumrange（i，j）=sum[j+1]−sum[i]`。

代码实现：

```java
class Solution {
	private int[] sum;
	
	public NumArray(int[] nums) {
		sum = new int[nums.length + 1];
		for (int i = 0; i < nums.length; i++) {
			sum[i + 1] = sum[i] + nums[i];
		}
	}
	
	public int sumRange(int i, int j) {
		return sum[j + 1] - sum[i];
	}
}
```

### 使用最小花费爬楼梯-746

数组的每个索引做为一个阶梯，第 i个阶梯对应着一个非负数的体力花费值 `cost[i]`(索引从0开始)。每当你爬上一个阶梯你都要花费对应的体力花费值，然后你可以选择继续爬一个阶梯或者爬两个阶梯。您需要找到达到楼层顶部的最低花费。在开始时，你可以选择从索引为 0 或 1 的元素作为初始阶梯。

示例 1:

```
输入: cost = [10, 15, 20]
输出: 15
解释: 最低花费是从cost[1]开始，然后走两步即可到阶梯顶，一共花费15。
```

示例 2:

```
输入: cost = [1, 100, 1, 1, 1, 100, 1, 1, 100, 1]
输出: 6
解释: 最低花费方式是从cost[0]开始，逐个经过那些1，跳过cost[3]，一共花费6。
```

注意：
1. cost 的长度将会在 [2, 1000]。
2. 每一个 cost[i] 将会是一个Integer类型，范围为 [0, 999]。

思路分析：
1. 第 i 级阶梯的总花费 = 第 i 级的cost + 前两级阶梯的总花费的较小者
2. 状态转移方程：`f[i] = cost[i] + min( f[i-1] , f[i-2] )`

代码实现：

```Java
class Solution {
    public int minCostClimbingStairs(int[] cost) {
        int f1 = 0;
        int f2 = 0;
        for (int i = 0; i < cost.length; i++) {
            int cur = cost[i] + Math.min(f1, f2);
            f1 = f2;
            f2 = cur;
        }
        return Math.min(f1, f2);
    }
}
```

### 三步问题-面试题 08.01

有个小孩正在上楼梯，楼梯有n阶台阶，小孩一次可以上1阶、2阶或3阶。实现一种方法，计算小孩有多少种上楼梯的方式。结果可能很大，你需要对结果模1000000007。

示例1:

```
输入：n = 3 
输出：4
说明: 有四种走法
```

示例2:

```
输入：n = 5
输出：13
提示:n范围在[1, 1000000]之间
```

思路分析：
1. `f(1) = 1`
2. `f(2) = f(1) + 1`          2
3. `f(3) = f(1) + f(2) + 1 `      4
4. `f(4) = f(1) + f(2) + f(3)`
5. `f(n) = f(n-1) + f(n-2) + f(n-3)`   找到递归式

```java
class Solution {
    public int waysToStep(int n) {
        if (n == 1) {
            return 1;
        } else if (n == 2) {
            return 2;
        } else if (n == 3) {
            return 4;
        }
        // 由于只用到前三项的数，所以没必要搞个数组，三个变量解决
        long d1 = 1;
        long d2 = 2;
        long d3 = 4;
        for (int i = 4; i <= n; i++) {
            long temp = (d1 + d2 + d3) % 1000000007;
            d1 = d2;
            d2 = d3;
            d3 = temp;
        }
        return (int)d3;
    }
}
```

### 按摩师-面试题 17.16

一个有名的按摩师会收到源源不断的预约请求，每个预约都可以选择接或不接。在每次预约服务之间要有休息时间，因此她不能接受相邻的预约。给定一个预约请求序列，替按摩师找到最优的预约集合（总预约时间最长），返回总的分钟数。
注意：本题相对原题稍作改动 

示例 1：

```
输入： [1,2,3,1]
输出： 4
解释： 选择 1 号预约和 3 号预约，总时长 = 1 + 3 = 4。
```

示例 2：

```
输入： [2,7,9,3,1]
输出： 12
解释： 选择 1 号预约、 3 号预约和 5 号预约，总时长 = 2 + 9 + 1 = 12。
```

示例 3：

```
输入： [2,1,4,5,3,1,1,3]
输出： 12
解释： 选择 1 号预约、 3 号预约、 5 号预约和 8 号预约，总时长 = 2 + 4 + 3 + 3 = 12。
```

思路分析：

这题的关键找出动态转移方程；状态转移方程：`dp[i]=max(dp[i-1],dp[i-2]+nums[i])`

```java

class Solution {
    public int massage(int[] nums) {  
        if(nums.length == 0) return 0;
        if(nums.length == 1) return nums[0];
        if(nums.length == 2){
            return Math.max(nums[0],nums[1]);
        }
        int [] dp = new int[nums.length];
        dp[0] = nums[0];
        dp[1] = tworet;
        for(int i = 2; i < nums.length; i++){
            dp[i] = Math.max(dp[i - 2] + nums[i], dp[i - 1]);    // 核心式子
        }
        return dp[nums.length - 1];
    }
}
```

#### ### 不同的二叉搜索树-96

给定一个整数 n，求以 1 ... n 为节点组成的二叉搜索树有多少种？

示例:

```
输入: 3
输出: 5
解释:
给定 n = 3, 一共有 5 种不同结构的二叉搜索树:

   1         3     3      2      1
    \       /     /      / \      \
     3     2     1      1   3      2
    /     /       \                 \
   2     1         2                 3
```

思路分析：

给定一个有序序列 1 ... n，为了根据序列构建一棵二叉搜索树。我们可以遍历每个数字 i，将该数字作为树根，1 ... (i-1) 序列将成为左子树，(i+1) ... n 序列将成为右子树。

详解：https://leetcode-cn.com/problems/unique-binary-search-trees/solution/bu-tong-de-er-cha-sou-suo-shu-by-leetcode/

代码实现：

```java
public class Solution {
  public int numTrees(int n) {
    int[] G = new int[n + 1];
    G[0] = 1;
    G[1] = 1;

    for (int i = 2; i <= n; ++i) {
      for (int j = 1; j <= i; ++j) {
        G[i] += G[j - 1] * G[i - j];
      }
    }
    return G[n];
  }
}
```

