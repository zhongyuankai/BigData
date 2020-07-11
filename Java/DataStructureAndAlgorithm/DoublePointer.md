# 双指针

[TOC]

## 1、替换后的最长重复字符_424

给你一个仅由大写英文字母组成的字符串，你可以将任意位置上的字符替换成另外的字符，总共可最多替换 *k* 次。在执行上述操作后，找到包含重复字母的最长子串的长度。

**注意:**
字符串长度 和 *k* 不会超过 104。

**示例 1:**

```
输入:
s = "ABAB", k = 2

输出:
4

解释:
用两个'A'替换为两个'B',反之亦然。
```

**示例 2:**

```
输入:
s = "AABABBA", k = 1

输出:
4

解释:
将中间的一个'A'替换为'B',字符串变为 "AABBBBA"。
子串 "BBBB" 有最长重复字母, 答案为 4。
```

思路分析：

用双指针来维护一个移动窗口。

1. 当窗口的长度（right - left + 1）大于最大的串的长度（max + k）时，将窗口右移。
2. 当窗口的长度小于或等于最大串的长度时，将窗口向右扩大。

代码：

```java
class Solution {
    public int characterReplacement(String s, int k) {
        char[] chars = s.toCharArray();
        int[] map = new int[26];    // 记录窗口中字符出现的次数
        int left = 0;
        int max = 0;    // 记录窗口中出现字符次数最多的字符
        for (int right = 0; right < chars.length; right++) {
            int index = chars[right] - 'A';
            map[index]++;
            max = Math.max(max, map[index]);
            // 如果当前窗口已经大于当前最大的串，则将窗口右移
            if (right - left + 1 > max + k) {
                map[chars[left] - 'A']--;
                left++;
            }
        }
        return chars.length - left;
    }
}
```

类似题：[1004. 最大连续1的个数 III](https://leetcode-cn.com/problems/max-consecutive-ones-iii/)

## 2、乘积小于K的子数组_713

给定一个正整数数组 nums。

找出该数组内乘积小于 k 的连续的子数组的个数。

**示例 1:**

```
输入: nums = [10,5,2,6], k = 100
输出: 8
解释: 8个乘积小于100的子数组分别为: [10], [5], [2], [6], [10,5], [5,2], [2,6], [5,2,6]。
需要注意的是 [10,5,2] 并不是乘积小于100的子数组。
```

**说明:**

```
0 < nums.length <= 50000
0 < nums[i] < 1000
0 <= k < 10^6
```

**思路分析：**

我们使用一重循环枚举 `right`，同时设置 `left` 的初始值为 `0`。在循环的每一步中，表示 `right` 向右移动了一位，将乘积乘以 `nums[right]`。此时我们需要向右移动 `left`，直到满足乘积小于 `k`的条件。在每次移动时，需要将乘积除以 `[left]`。当 `left` 移动完成后，对于当前的 `right`，就包含了 `right−left+1` 个乘积小于 `k` 的连续子数组。

代码：

```java
class Solution {
    public int numSubarrayProductLessThanK(int[] nums, int k) {
        if (k <= 1) return 0;
        int prod = 1, ans = 0, left = 0;
        for (int right = 0; right < nums.length; right++) {
            prod *= nums[right];
            while (prod >= k) prod /= nums[left++];
            ans += right - left + 1;
        }
        return ans;
    }
}
```