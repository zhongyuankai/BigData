# 动态规划解题分析

[TOC]

## 1、买卖股票的最佳时机 II-122.

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

## 2、丑数 II-264

编写一个程序，找出第 n 个丑数。丑数就是只包含质因数 2, 3, 5 的正整数。

 示例:输入: n = 10

```
输出: 12
解释: 1, 2, 3, 4, 5, 6, 8, 9, 10, 12 是前 10 个丑数。
说明: 1 是丑数。n 不超过1690。
```

**方法一：堆**

1. 弹出堆中最小的数字 k 并添加到数组 nums 中;
2. 若 2k，3k，5k 不存在在哈希表中，则将其添加到栈中并更新哈希表。

```java
class Solution {
    public int nthUglyNumber(int n) {
        long[] nums = new long[1690];
        PriorityQueue<Long> heap = new PriorityQueue<>();
        HashSet<Long> set = new HashSet<>();
        heap.offer(1L);
        set.add(1L);       
        int[] items = {2, 3, 5};
        for (int i = 0; i < 1690; i++) {
            long min = heap.poll();
            nums[i] = min;
            for (int item: items) {
                long temp = min * item;
                if (!set.contains(temp)) {
                    heap.offer(temp);
                    set.add(temp);
                }
            }
        }
        return (int)nums[n-1];
    }
}
```

**方法二： 动态规划**

1. 初始化数组 nums 和三个指针 i2，i3，i5 ;
2. 在 nums[i2] * 2，nums[i3] * 3 和 nums[i5] * 5 选出最小的数字添加到数组 nums 中。
3. 将该数字对应的因子指针向前移动一步。

```java
class Solution {
    public int nthUglyNumber(int n) {
        int[] nums = new int[n];
        nums[0] = 1;
        int p2 = 0, p3 = 0, p5 = 0;
        for (int i = 1; i < n; i++) {
            int min = Math.min(nums[p2] * 2, Math.min(nums[p3] * 3, nums[p5] * 5));
            if (min == nums[p2] * 2)  p2++;
            if (min == nums[p3] * 3) p3++;
            if (min == nums[p5] * 5) p5++;
            nums[i] = min;
        }
        return nums[n-1];
    }
}
```

## 3、区域和检索 - 数组不可变-303

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

## 4、使用最小花费爬楼梯-746

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

## 5、三步问题-面试题 08.01

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

## 6、按摩师-面试题 17.16

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

## 7、不同的二叉搜索树-96

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

[详解](https://leetcode-cn.com/problems/unique-binary-search-trees/solution/bu-tong-de-er-cha-sou-suo-shu-by-leetcode/)

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

## 8、最大正方形-221

在一个由 0 和 1 组成的二维矩阵内，找到只包含 1 的最大正方形，并返回其面积。

```
示例:
输入:
1 0 1 0 0
1 0 1 1 1
1 1 1 1 1
1 0 0 1 0
输出: 4
```

**思路分析：**

- 1. 我们用 0 初始化另一个矩阵 dp，维数和原始矩阵维数相同；
  2. dp(i,j) 表示的是由 1 组成的最大正方形的边长；
  3. 从 (0,0)开始，对原始矩阵中的每一个 1，我们将当前元素的值更新为dp(i, j)=min(dp(i−1, j), dp(i−1, j−1), dp(i, j−1))+1；
  4. 我们还用一个变量记录当前出现的最大边长，这样遍历一次，找到最大的正方形边长；

**代码：**

```java
class Solution {
    public int maximalSquare(char[][] matrix) {
        // dp(i,j) = max(dp(i,j-1), dp(i-1, j), dp(i-1,j-1)) + 1;
        int X = matrix.length;
        int Y = X > 0 ? matrix[0].length : 0;
        int[][] dp = new int[X+1][Y+1];
        int ans = 0;
        for (int i = 1; i <= X; i++) {
            for (int j = 1; j <= Y; j++) {
                if (matrix[i-1][j-1] == '1') {
                    dp[i][j] = Math.min(dp[i][j-1], Math.min(dp[i-1][j], dp[i-1][j-1])) + 1;
                    ans = Math.max(ans, dp[i][j]);
                }
            }
        }
        return ans * ans;
    }
}
```

## 9、打家劫舍-leetcode-198

你是一个专业的小偷，计划偷窃沿街的房屋。每间房内都藏有一定的现金，影响你偷窃的唯一制约因素就是相邻的房屋装有相互连通的防盗系统，如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警。

给定一个代表每个房屋存放金额的非负整数数组，计算你在不触动警报装置的情况下，能够偷窃到的最高金额。

**示例：**

```
输入: [2,7,9,3,1]
输出: 12
解释: 偷窃 1 号房屋 (金额 = 2), 偷窃 3 号房屋 (金额 = 9)，接着偷窃 5 号房屋 (金额 = 1)。
偷窃到的最高金额 = 2 + 9 + 1 = 12 。
```

**思路分析：**

状态转移方程 `dp[i] = Math.max(num[i] + dp[i-2], dp[i-1])`;

由于每次计算只需要前两个值，所有可以不用数组，用两个变量就足够用了。

**代码：**

```java
public int rob(int[] num) {
    int prevMax = 0;
    int currMax = 0;
    for (int x : num) {
        int temp = currMax;
        currMax = Math.max(prevMax + x, currMax);
        prevMax = temp;
    }
    return currMax;
}
```

## 10、打家劫舍 II-213

你是一个专业的小偷，计划偷窃沿街的房屋，每间房内都藏有一定的现金。这个地方所有的房屋都围成一圈，这意味着第一个房屋和最后一个房屋是紧挨着的。同时，相邻的房屋装有相互连通的防盗系统，如果两间相邻的房屋在同一晚上被小偷闯入，系统会自动报警。

给定一个代表每个房屋存放金额的非负整数数组，计算你在不触动警报装置的情况下，能够偷窃到的最高金额。

**示例：**

```
输入: [1,2,3,1]
输出: 4
解释: 你可以先偷窃 1 号房屋（金额 = 1），然后偷窃 3 号房屋（金额 = 3）。
偷窃到的最高金额 = 1 + 3 = 4 。
```

**思路分析：**

环状排列意味着第一个房子和最后一个房子中只能选择一个偷窃，因此可以把此环状排列房间问题约化为两个单排排列房间子问题：

1. 在不偷窃第一个房子的情况下（即  `nums[1:]`），最大金额是 `p1` ；
2. 在不偷窃最后一个房子的情况下（即 `nums[:n−1]`），最大金额是 `p2`；
3. 综合偷窃最大金额： 为以上两种情况的较大值，即 `max(p1,p2)` ；

**代码：**

```java
class Solution {
    public int rob(int[] nums) {
        if (nums.length == 0) return 0;
        if (nums.length == 1) return nums[0];
        return Math.max(myRob(Arrays.copyOfRange(nums, 0, nums.length - 1)),
                        myRob(Arrays.copyOfRange(nums, 1, nums.length)));
    }
    public int myRob(int[] nums) {
        int cur = nums[0];
        int pre = 0;
        for (int i = 1; i < nums.length; i++) {
            int tmp = Math.max(cur, pre + nums[i]);
            pre = cur;
            cur = tmp;
        }
        return cur;
    }
}
```

## 11、乘积最大子数组-152

给你一个整数数组 nums ，请你找出数组中乘积最大的连续子数组（该子数组中至少包含一个数字）。

```
示例 1:
输入: [2,3,-2,4]
输出: 6
解释: 子数组 [2,3] 有最大乘积 6。
```

**思路分析: **

1. 当前最大值为 `imax = max(imax * nums[i], nums[i])`；
2. 由于存在负数，那么会导致最大的变最小的，最小的变最大的。因此还需要维护当前最小值，`imin = min(imin * nums[i], nums[i])]`；
3. 当负数出现时则`imax`与`imin`进行交换再进行下一步计算；

**代码：**

```java
class Solution {
    public int maxProduct(int[] nums) {
        int max = Integer.MIN_VALUE;
        int imax = 1;
        int imin = 1;
        for (int num: nums) {
            // 如果当前数为负数，当前最大值会变最小，最小值会变最大，所以交换最大最小值。
            if (num < 0) {
                int tmp = imax;
                imax = imin;
                imin = tmp;
            }
            imax = Math.max(imax * num, num);
            imin = Math.min(imin * num, num);
            max = Math.max(max, imax);
        }
        return max;
    }
}
```

## 12、单词拆分-139

给定一个非空字符串 s 和一个包含非空单词列表的字典 `wordDict`，判定 s 是否可以被空格拆分为一个或多个在字典中出现的单词。

说明：拆分时可以重复使用字典中的单词，你可以假设字典中没有重复的单词。

```
输入: s = "applepenapple", wordDict = ["apple", "pen"]
输出: true
解释: 返回 true 因为 "applepenapple" 可以被拆分成 "apple pen apple"。
注意你可以重复使用字典中的单词。
```

**方法 1：递归和回溯**

检查字典单词中每一个单词的可能前缀，**如果在字典中出现过，那么去掉这个前缀后剩余部分回归调用**。

**代码：**

 ```java
public boolean wordBreak(String s, List<String> wordDict) {
    return word_Break(s, new HashSet(wordDict), 0);
}
public boolean word_Break(String s, Set<String> wordDict, int start) {
    if (start == s.length()) {
        return true;
    }
    for (int end = start + 1; end <= s.length(); end++) {
        if (wordDict.contains(s.substring(start, end)) && word_Break(s, wordDict, end)) {
            return true;
        }
    }
    return false;
}
 ```

**方法 2：记忆化回溯**

 在先前的方法中，我们看到许多函数调用都是冗余的，也就是我们会对相同的字符串调用多次回溯函数。

为了避免这种情况，我们可以使用记忆化的方法，其中一个 memo 数组会被用来保存子问题的结果。每当访问到已经访问过的后缀串，直接用 memo 数组中的值返回而不需要继续调用函数。

通过记忆化，许多冗余的子问题可以极大被优化，回溯树得到了剪枝，因此极大减小了时间复杂度。

代码：

```java
public boolean wordBreak(String s, List<String> wordDict) {
    return word_Break(s, new HashSet(wordDict), 0, new Boolean[s.length()]);
}
public boolean word_Break(String s, Set<String> wordDict, int start, Boolean[] memo) {
    if (start == s.length()) {
        return true;
    }
    if (memo[start] != null) {
        return memo[start];
    }
    for (int end = start + 1; end <= s.length(); end++) {
        if (wordDict.contains(s.substring(start, end)) && word_Break(s, wordDict, end, memo)) {
            return memo[start] = true;
        }
    }
    return memo[start] = false;
}
```

## 13、三角形最小路径和-120

给定一个三角形，找出自顶向下的最小路径和。每一步只能移动到下一行中相邻的结点上。

例如，给定三角形：

```java
[
[2],
[3,4],
[6,5,7],
[4,1,8,3]
]
```

自顶向下的最小路径和为 11（即，2 + 3 + 5 + 1 = 11）。

说明：

如果你可以只使用 O(n) 的额外空间（n 为三角形的总行数）来解决这个问题，那么你的算法会很加分。

**思路分析：**

1. 状态转换方程： `dp[i][j]=min(dp[i-1][j],dp[i-1][j-1])+triangle[i][j]`；

2. `triangle[i][0]`没有左上角 只能从`triangle[i-1][j]`经过；
3. `triangle[i][row[0].length]`没有上面 只能从`triangle[i-1][j-1]`经过 ；

**代码：**

```java
public int minimumTotal(List<List<Integer>> triangle) {
    // 特判
    if (triangle == null || triangle.size() == 0) {
        return 0;
    }
    int row = triangle.size();
    int column = triangle.get(row - 1).size();
    int[][] dp = new int[row][column];
    dp[0][0] = triangle.get(0).get(0);
    int res = Integer.MAX_VALUE;
    for (int i = 1; i < row; i++) {
        //对每一行的元素进行推导
        for (int j = 0; j <= i; j++) {
            if (j == 0) {
                // 最左端特殊处理
                dp[i][j] = dp[i - 1][j] + triangle.get(i).get(j);
            } else if (j == i) {
                // 最右端特殊处理
                dp[i][j] = dp[i - 1][j - 1] + triangle.get(i).get(j);
            } else {
                dp[i][j] = Math.min(dp[i - 1][j], dp[i - 1][j - 1]) + triangle.get(i).get(j);
            }
        }
    }
    // dp最后一行记录了最小路径
    for (int i = 0; i < column; i++) {
        res = Math.min(res, dp[row - 1][i]);
    }
    return res;
}
```

**优化：**

因为只需要第`i-1`行的`dp[i - 1][j]`和`dp[i - 1][j - 1]`元素即可，可以使用两个变量暂存。一维的dp数组只存储第`i`行的最小路径和。

```java
public int minimumTotal(List<List<Integer>> triangle) {
    // 特判
    if (triangle == null || triangle.size() == 0) {
        return 0;
    }
    // dp最大长度==triangle底边长度
    // 题意：只使用 O(n) 的额外空间（n 为三角形的总行数）
    int[] dp = new int[triangle.size()];
    dp[0] = triangle.get(0).get(0);

    // prev暂存dp[i-1][j-1],cur暂存dp[i-1][j]
    int prev = 0, cur;
    for (int i = 1; i < triangle.size(); i++) {
        //对每一行的元素进行推导
        List<Integer> rows = triangle.get(i);
        for (int j = 0; j <= i; j++) {
            cur = dp[j];
            if (j == 0) {
                // 最左端特殊处理
                dp[j] = cur + rows.get(j);
            } else if (j == i) {
                // 最右端特殊处理
                dp[j] = prev + rows.get(j);
            } else {
                dp[j] = Math.min(cur, prev) + rows.get(j);
            }
            prev = cur;
        }
    }

    int res = Integer.MAX_VALUE;
    // dp最后一行记录了最小路径
    for (int i = 0; i < triangle.size(); i++) {
        res = Math.min(res, dp[i]);
    }
    return res;
}
```



- 