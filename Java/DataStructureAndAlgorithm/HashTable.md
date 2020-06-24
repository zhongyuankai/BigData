# 哈希表

## 面试题 16.24. 数对和

设计一个算法，找出数组中两数之和为指定值的所有整数对。一个数只能属于一个数对。

示例 1:

```
输入: nums = [5,6,5], target = 11
输出: [[5,6]]
```

示例 2:

```
输入: nums = [5,6,5,6], target = 11
输出: [[5,6],[5,6]]
```

提示：

    nums.length <= 100000

方法一：哈希表辅助

```java
class Solution {
    public List<List<Integer>> pairSums(int[] nums, int target) {
        //key:数组的元素;value:该元素出现的次数
        Map<Integer, Integer> map = new HashMap<>();
        
        List<List<Integer>> ans = new ArrayList<>();
        for (int num : nums) {
            Integer count = map.get(target - num);
            if (count != null) {
                ans.add(Arrays.asList(num, target - num));
                if (count == 1)
                    map.remove(target - num);
                else
                    map.put(target - num, --count);
            } else 
                map.put(num, map.getOrDefault(num, 0) + 1);
        }
        return ans;
    }
}
```

方法二：双指针实现，要先对数组进行排序处理。

```java
class Solution {
    public List<List<Integer>> pairSums(int[] nums, int target) {
        //对数组进行排序
        Arrays.sort(nums);
        
        List<List<Integer>> ans = new LinkedList<>();
        int left = 0, right = nums.length - 1;
        while (left < right) {
            //两个指针所指的两个元素和
            int sum = nums[left] + nums[right];
            //如果两个的和小于目标值，那么left指针向右走一步继续寻找
            if (sum < target)
                ++left;
            //如果两个的和大于目标值，那么right指针向左走一步继续寻找
            else if (sum > target)
                --right;
            //如果刚好等于要找的target值，那么加入结果集中，并且left指针和right指针分别向右和向左走一步(因为一个数只能属于一个数对)
            else 
                ans.add(Arrays.asList(nums[left++], nums[right--]));
        }
        return ans;
    }
}
```

## 面试题48. 最长不含重复字符的子字符串

请从字符串中找出一个最长的不包含重复字符的子字符串，计算该最长子字符串的长度。

示例 1:

```
输入: "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
```

示例 2:

```
输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
```

示例 3:

```
输入: "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
```

提示：

    s.length <= 40000

思路分析：**双指针 + 哈希表**

1. 遍历字符串，用map存储字符所在的位置；
2. 如果存在了，则更新左指针，`i = Math.max(i, map.get(s.charAt(j)))`;（需要求最大值的原因是防止左指针之前的字符由于已经存在map中而造成干扰）；
3. 计算当前最大长度 `res = Math.max(res, j - 1)`;

代码：

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> dic = new HashMap<>();
        int i = -1, res = 0;
        for(int j = 0; j < s.length(); j++) {
            if(dic.containsKey(s.charAt(j)))
                i = Math.max(i, dic.get(s.charAt(j))); // 更新左指针 i
            dic.put(s.charAt(j), j); // 哈希表记录
            res = Math.max(res, j - i); // 更新结果
        }
        return res;
    }
} 
```

