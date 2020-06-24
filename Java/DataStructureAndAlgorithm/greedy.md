# 贪心算法解题分析

[TOC]

### 用户分组-1282

有 n 位用户参加活动，他们的 ID 从 0 到 n - 1，每位用户都 恰好 属于某一用户组。给你一个长度为 n 的数组 `groupSizes`，其中包含每位用户所处的用户组的大小，请你返回用户分组情况（存在的用户组以及每个组中用户的 ID）。

你可以任何顺序返回解决方案，ID 的顺序也不受限制。此外，题目给出的数据保证至少存在一种解决方案。

示例 1：

```
输入：groupSizes = [3,3,3,3,3,1,3]
输出：[[5],[0,1,2],[3,4,6]]
解释： 
其他可能的解决方案有 [[2,1,6],[5],[0,4,3]] 和 [[5],[0,6,2],[4,3,1]]。
```

示例 2：

```
输入：groupSizes = [2,1,3,3,3,2]
输出：[[1],[0,5],[2,3,4]]
```

提示：

    groupSizes.length == n
    1 <= n <= 500
    1 <= groupSizes[i] <= n

思路分析：

根据`groupSize`进行分组，使用`HashMap`存储。遍历`HashMap`进行分组，组元素的个数不超过`groupSize`；

代码实现：

```java
class Solution {
    public List<List<Integer>> groupThePeople(int[] groupSizes) {
        Map<Integer, List<Integer>> map = new HashMap<>();
        for (int i = 0; i < groupSizes.length; i++) {
            if (!map.containsKey(groupSizes[i])){
                map.put(groupSizes[i], new ArrayList<Integer>());
            } 
            map.get(groupSizes[i]).add(i);
        }
        List<List<Integer>> ans = new ArrayList<>();
        for (Map.Entry<Integer, List<Integer>> kv: map.entrySet()) {
            int gsize = kv.getKey();
            List<Integer> list = kv.getValue();
            for (int i = 0; i < list.size(); i+=gsize) {
                ans.add(list.subList(i, gsize+i));
            }
        }
        return ans;
    }
}
```

###  跳跃游戏-55

给定一个非负整数数组，你最初位于数组的第一个位置。

数组中的每个元素代表你在该位置可以跳跃的最大长度。

判断你是否能够到达最后一个位置。

示例 1:

```
输出: true
解释: 我们可以先跳 1 步，从位置 0 到达 位置 1, 然后再从位置 1 跳 3 步到达最后一个位置。
```

示例 2:

```
输入: [3,2,1,0,4]
输出: false
解释: 无论怎样，你总会到达索引为 3 的位置。但该位置的最大跳跃长度是 0 ， 所以你永远不可能到达最后一个位置。
```

思路分析：

遍历数组，计算能跳跃的最大距离，如果最大距离大于数组的长度，则位`true`；

代码实现：

```java
public class Solution {
    public boolean canJump(int[] nums) {
        int n = nums.length;
        int rightmost = 0;
        for (int i = 0; i < n; ++i) {
            if (i <= rightmost) {
                rightmost = Math.max(rightmost, i + nums[i]);
                if (rightmost >= n - 1) {
                    return true;
                }
            }
        }
        return false;
    }
}
```

###  划分数组为连续数字的集合-1296

给你一个整数数组 `nums` 和一个正整数 k，请你判断是否可以把这个数组划分成一些由 k 个连续数字组成的集合。
如果可以，请返回 True；否则，返回 False。

示例 1：

```
输入：nums = [1,2,3,3,4,4,5,6], k = 4
输出：true
解释：数组可以分成 [1,2,3,4] 和 [3,4,5,6]。
```

示例 2：

```
输入：nums = [3,2,1,2,3,4,3,4,5,9,10,11], k = 3
输出：true
解释：数组可以分成 [1,2,3] , [2,3,4] , [3,4,5] 和 [9,10,11]。
```

示例 3：

```
输入：nums = [1,2,3,4], k = 3
输出：false
解释：数组不能分成几个大小为 3 的子数组。
```

提示：

    1 <= nums.length <= 10^5
    1 <= nums[i] <= 10^9
    1 <= k <= nums.length

思路分析：

可以采用优先队列来实现，将所有数据加入队列中，将第一元素出队，然后以此元素为基准依次递增的删除，直到k，中间如果删除返回`false`，则不能划分，开始第二轮，最后队列为空返回`true`；

代码实现：

```java
class Solution {
    public boolean isPossibleDivide(int[] nums, int k) {
        if (nums.length % k != 0) {
            return false;
        }
        PriorityQueue<Integer> queue = new PriorityQueue<>();
        for (int num: nums) {
            queue.offer(num);
        }

        while (!queue.isEmpty()) {
            int top = queue.poll();
            for (int i = 1; i < k; i++) {
                if (!queue.remove(top + i)) {
                    return false;
                }
            }
        }
        return true;
    }
}
```

### 拼车-1094

假设你是一位顺风车司机，车上最初有 `capacity` 个空座位可以用来载客。由于道路的限制，车 只能 向一个方向行驶（也就是说，不允许掉头或改变方向，你可以将其想象为一个向量）。

这儿有一份行程计划表 trips[][]，其中 `trips[i] = [num_passengers, start_location, end_location]` 包含了你的第 i 次行程信息：

- 必须接送的乘客数量；
- 乘客的上车地点；
- 以及乘客的下车地点。

这些给出的地点位置是从你的 初始 出发位置向前行驶到这些地点所需的距离（它们一定在你的行驶方向上）。

请你根据给出的行程计划表和车子的座位数，来判断你的车是否可以顺利完成接送所用乘客的任务（当且仅当你可以在所有给定的行程中接送所有乘客时，返回 `true`，否则请返回 `false`）。

示例 1：

```
输入：trips = [[2,1,5],[3,3,7]], capacity = 4
输出：false
```

示例 2：

```
输入：trips = [[2,1,5],[3,3,7]], capacity = 5
输出：true
```

示例 3：

```
输入：trips = [[2,1,5],[3,5,7]], capacity = 3
输出：true
```

示例 4：

```
输入：trips = [[3,2,7],[3,7,9],[8,3,9]], capacity = 11
输出：true
```

提示：

1. 你可以假设乘客会自觉遵守 “先下后上” 的良好素质
2. `trips.length <= 1000`
3. `trips[i].length == 3`
4. `1 <= trips[i][0] <= 100`
5. `0 <= trips[i][1] < trips[i][2] <= 1000`
6. `1 <= capacity <= 100000`

思路分析：

1. 先计算出每个地点上车或下车的人数；
2. 依次遍历所有的地点，如果当前的乘客数大于`capacity`，则返回`false`；

代码实现：

```java
class Solution {
    public boolean carPooling(int[][] trips, int capacity) {
        int[] changers = new int[1001];
        for (int[] trip: trips) {
            changers[trip[1]] += trip[0];
            changers[trip[2]] -= trip[0];
        }
        int num = 0;
        for (int changer: changers) {
            num += changer;
            if (num > capacity) {
                return false;
            }
        }
        return true;
    }
}
```

### 任务调度器-621

给定一个用字符数组表示的 CPU 需要执行的任务列表。其中包含使用大写的 A - Z 字母表示的26 种不同种类的任务。任务可以以任意顺序执行，并且每个任务都可以在 1 个单位时间内执行完。CPU 在任何一个单位时间内都可以执行一个任务，或者在待命状态。

然而，两个相同种类的任务之间必须有长度为 n 的冷却时间，因此至少有连续 n 个单位时间内 CPU 在执行不同的任务，或者在待命状态。

你需要计算完成所有任务所需要的最短时间。 

示例 ：

```
输入：tasks = ["A","A","A","B","B","B"], n = 2
输出：8
解释：A -> B -> (待命) -> A -> B -> (待命) -> A -> B.
     在本示例中，两个相同类型任务之间必须间隔长度为 n = 2 的冷却时间，而执行一个任务只需要一个单位时间，所以中间出现了（待命）状态。 
```

提示：

1. 任务的总个数为 [1, 10000]。
2. n 的取值范围为 [0, 100]。

 思路分析：

1. 计算每个种类的任务的数量，根据数量进行排序；
2. 在n个单位时间为一个批次处理任务，从任务数量最多的开始，依次递减；
3. 处理完一批次后，从重新排序；
4. 重复2~3；

代码实现：

```java
public class Solution {
    public int leastInterval(char[] tasks, int n) {
        int[] map = new int[26];
        for (char c: tasks)
            map[c - 'A']++;
        Arrays.sort(map);
        int time = 0;
        while (map[25] > 0) {
            int i = 0;
            while (i <= n) {
                if (map[25] == 0)
                    break;
                if (i < 26 && map[25 - i] > 0)
                    map[25 - i]--;
                time++;
                i++;
            }
            Arrays.sort(map);
        }
        return time;
    }
}
```

### Dota2 参议院-649

`Dota2` 的世界里有两个阵营：`Radiant`(天辉)和 `Dire`(夜魇)

`Dota2` 参议院由来自两派的参议员组成。现在参议院希望对一个 Dota2 游戏里的改变作出决定。他们以一个基于轮为过程的投票进行。在每一轮中，每一位参议员都可以行使两项权利中的一项：

1. 禁止一名参议员的权利：

   参议员可以让另一位参议员在这一轮和随后的几轮中丧失所有的权利。

2. 宣布胜利：

   如果参议员发现有权利投票的参议员都是同一个阵营的，他可以宣布胜利并决定在游戏中的有关变化。


给定一个字符串代表每个参议员的阵营。字母 “`R`” 和 “`D`” 分别代表了 Radiant（天辉）和 Dire（夜魇）。然后，如果有 n 个参议员，给定字符串的大小将是 n。

以轮为基础的过程从给定顺序的第一个参议员开始到最后一个参议员结束。这一过程将持续到投票结束。所有失去权利的参议员将在过程中被跳过。

假设每一位参议员都足够聪明，会为自己的政党做出最好的策略，你需要预测哪一方最终会宣布胜利并在 Dota2 游戏中决定改变。输出应该是 Radiant 或 Dire。

 

示例 1:

```
输入: "RD"
输出: "Radiant"
解释:  第一个参议员来自  Radiant 阵营并且他可以使用第一项权利让第二个参议员失去权力，因此第二个参议员将被跳过因为他没有任何权利。然后在第二轮的时候，第一个参议员可以宣布胜利，因为他是唯一一个有投票权的人
```

示例 2:

```
输入: "RDD"
输出: "Dire"
解释: 
第一轮中,第一个来自 Radiant 阵营的参议员可以使用第一项权利禁止第二个参议员的权利
第二个来自 Dire 阵营的参议员会被跳过因为他的权利被禁止
第三个来自 Dire 阵营的参议员可以使用他的第一项权利禁止第一个参议员的权利
因此在第二轮只剩下第三个参议员拥有投票的权利,于是他可以宣布胜利
```

注意:

1. 给定字符串的长度在 [1, 10,000] 之间.

思路分析：

​	为了使己方得到胜利，在使用权力的时候需要把敌方淘汰了，所以成员执行权力的先后顺序决定着输赢。

就比如`DDRRR`，虽然`R`方人比较多，但是`D`方先使用权力，最后胜利的`D`方的。

在解决这道题的时候有一个小技巧，就是当遍历到`R`的时候，不要马上去找`D`把它淘汰掉，而是先记着，等遍历到`D`的时候再把它淘汰掉。

代码实现：

```java
class Solution {
    public String predictPartyVictory(String senate) {
        Queue<Integer> queue = new LinkedList();
        int[] people = new int[]{0, 0};
        int[] bans = new int[]{0, 0};

        for (char person: senate.toCharArray()) {
            int x = person == 'R' ? 1 : 0;
            people[x]++;
            queue.add(x);
        }

        while (people[0] > 0 && people[1] > 0) {
            int x = queue.poll();
            if (bans[x] > 0) {
                bans[x]--;
                people[x]--;
            } else {
                bans[x^1]++;
                queue.add(x);
            }
        }

        return people[1] > 0 ? "Radiant" : "Dire";
    }
}
```

### 加油站-134.

在一条环路上有 N 个加油站，其中第 i 个加油站有汽油 gas[i] 升。

你有一辆油箱容量无限的的汽车，从第 i 个加油站开往第 i+1 个加油站需要消耗汽油 cost[i] 升。你从其中的一个加油站出发，开始时油箱为空。

如果你可以绕环路行驶一周，则返回出发时加油站的编号，否则返回 -1。

说明: 

- 如果题目有解，该答案即为唯一答案。
- 输入数组均为非空数组，且长度相同。
- 输入数组中的元素均为非负数。

示例 1:

```
输入: 
gas  = [1,2,3,4,5]
cost = [3,4,5,1,2]

输出: 3

解释:
从 3 号加油站(索引为 3 处)出发，可获得 4 升汽油。此时油箱有 = 0 + 4 = 4 升汽油
开往 4 号加油站，此时油箱有 4 - 1 + 5 = 8 升汽油
开往 0 号加油站，此时油箱有 8 - 2 + 1 = 7 升汽油
开往 1 号加油站，此时油箱有 7 - 3 + 2 = 6 升汽油
开往 2 号加油站，此时油箱有 6 - 4 + 3 = 5 升汽油
开往 3 号加油站，你需要消耗 5 升汽油，正好足够你返回到 3 号加油站。
因此，3 可为起始索引。
```

示例 2:

```java
输入: 
gas  = [2,3,4]
cost = [3,4,3]

输出: -1

解释:
你不能从 0 号或 1 号加油站出发，因为没有足够的汽油可以让你行驶到下一个加油站。
我们从 2 号加油站出发，可以获得 4 升汽油。 此时油箱有 = 0 + 4 = 4 升汽油
开往 0 号加油站，此时油箱有 4 - 3 + 2 = 3 升汽油
开往 1 号加油站，此时油箱有 3 - 3 + 3 = 3 升汽油
你无法返回 2 号加油站，因为返程需要消耗 4 升汽油，但是你的油箱只有 3 升汽油。
因此，无论怎样，你都不可能绕环路行驶一周。
```

思路分析：

1. 初始化 `total_tank` 和 `curr_tank` 为 0 ，并且选择 0 号加油站为起点。

2. 遍历所有的加油站：

   - 每一步中，都通过加上 `gas[i]` 和减去 `cost[i]` 来更新 `total_tank` 和 `curr_tank` 。

   - 如果在 `i + 1` 号加油站， `curr_tank < 0` ，将 `i + 1` 号加油站作为新的起点，同时重置 `curr_tank = 0` ，让油箱也清空。

3. 如果 `total_tank < 0` ，返回 `-1` ，否则返回 `starting station`。

具体参考：https://leetcode-cn.com/problems/gas-station/solution/jia-you-zhan-by-leetcode/

代码实现：

```java
class Solution {
  public int canCompleteCircuit(int[] gas, int[] cost) {
    int n = gas.length;

    int total_tank = 0;
    int curr_tank = 0;
    int starting_station = 0;
    for (int i = 0; i < n; ++i) {
      total_tank += gas[i] - cost[i];
      curr_tank += gas[i] - cost[i];
      // If one couldn't get here,
      if (curr_tank < 0) {
        // Pick up the next station as the starting one.
        starting_station = i + 1;
        // Start with an empty tank.
        curr_tank = 0;
      }
    }
    return total_tank >= 0 ? starting_station : -1;
  }
}
```

### 分割数组为连续子序列-659

输入一个按升序排序的整数数组（可能包含重复数字），你需要将它们分割成几个子序列，其中每个子序列至少包含三个连续整数。返回你是否能做出这样的分割？

示例 1：

```java
输入: [1,2,3,3,4,5]
输出: True
解释:
你可以分割出这样两个连续子序列 : 
1, 2, 3
3, 4, 5
```

示例 2：

```java
输入: [1,2,3,3,4,4,5,5]
输出: True
解释:
你可以分割出这样两个连续子序列 : 
1, 2, 3, 4, 5
3, 4, 5
```

示例 3：

```java
输入: [1,2,3,4,4,5]
输出: False
```

提示：

1. 输入的数组长度范围为 [1, 10000]

思路分析：

我们将每个数字的出现次数统计好，记 tails[x] 是恰好在 x 之前结束的链的数目。

现在我们逐一考虑每个数字，如果有一个链恰好在 x 之前结束，我们将 x 加入此链中。否则，如果我们可以新建立一条链就新建。

代码实现：

```java
class Solution {
    public boolean isPossible(int[] nums) {
        Counter count = new Counter();
        Counter tails = new Counter();
        for (int x: nums) count.add(x, 1);

        for (int x: nums) {
            if (count.get(x) == 0) {
                continue;
            } else if (tails.get(x) > 0) {
                tails.add(x, -1);
                tails.add(x+1, 1);
            } else if (count.get(x+1) > 0 && count.get(x+2) > 0) {
                count.add(x+1, -1);
                count.add(x+2, -1);
                tails.add(x+3, 1);
            } else {
                return false;
            }
            count.add(x, -1);
        }
        return true;
    }
}

class Counter extends HashMap<Integer, Integer> {
    public int get(int k) {
        return containsKey(k) ? super.get(k) : 0;
    }

    public void add(int k, int v) {
        put(k, get(k) + v);
    }
}
```

### 摆动序列376

如果连续数字之间的差严格地在正数和负数之间交替，则数字序列称为摆动序列。第一个差（如果存在的话）可能是正数或负数。少于两个元素的序列也是摆动序列。

例如，` [1,7,4,9,2,5] `是一个摆动序列，因为差值 `(6,-3,5,-7,3) `是正负交替出现的。相反,` [1,4,7,2,5]` 和` [1,7,4,5,5] `不是摆动序列，第一个序列是因为它的前两个差值都是正数，第二个序列是因为它的最后一个差值为零。

给定一个整数序列，返回作为摆动序列的最长子序列的长度。 通过从原始序列中删除一些（也可以不删除）元素来获得子序列，剩下的元素保持其原始顺序。

示例 1:

```
输入: [1,7,4,9,2,5]
输出: 6 
解释: 整个序列均为摆动序列。
```

示例 2:

```
输入: [1,17,5,10,13,15,10,5,16,8]
输出: 7
解释: 这个序列包含几个长度为 7 摆动序列，其中一个可为[1,17,10,13,10,16,8]。
```

示例 3:

```
输入: [1,2,3,4,5,6,7,8,9]
输出: 2
```

**方法一：暴力**

思路：我们去找所有可能摆动子序列的长度并找到它们中的最大值。`boolean` 变量 `isU `记录的是现在要找的是上升元素还是下降元素。

代码：

```java
public class Solution {
    private int calculate(int[] nums, int index, boolean isUp) {
        int maxcount = 0;
        for (int i = index + 1; i < nums.length; i++) {
            if ((isUp && nums[i] > nums[index]) || (!isUp && nums[i] < nums[index]))
                maxcount = Math.max(maxcount, 1 + calculate(nums, i, !isUp));
        }
        return maxcount;
    }

    public int wiggleMaxLength(int[] nums) {
        if (nums.length < 2)
            return nums.length;
        return 1 + Math.max(calculate(nums, 0, true), calculate(nums, 0, false));
    }
}
```

**方法二：动态规划**

思路：

1. 用两个数组来 `dp` ，分别记作 `upupup` 和 `downdowndown` ；
2. up[i] 存的是目前为止最长的以第 i个元素结尾的上升摆动序列的长度。
3. 当找到将第 i 个元素作为上升摆动序列的尾部的时候就更新 up[i]，如何更新 up[i]，我们需要考虑前面所有的降序结尾摆动序列，也就是找到 down[j]，满足`i>j`，`nums[i] > nums[j]`，`up[i] = Math.max(up[i], down[i] + 1)`；

代码：

```java
public class Solution {
    public int wiggleMaxLength(int[] nums) {
        if (nums.length < 2)
            return nums.length;
        int[] up = new int[nums.length];
        int[] down = new int[nums.length];
        for (int i = 1; i < nums.length; i++) {
            for(int j = 0; j < i; j++) {
                if (nums[i] > nums[j]) {
                    up[i] = Math.max(up[i],down[j] + 1);
                } else if (nums[i] < nums[j]) {
                    down[i] = Math.max(down[i],up[j] + 1);
                }
            }
        }
        return 1 + Math.max(down[nums.length - 1], up[nums.length - 1]);
    }
}
```





















