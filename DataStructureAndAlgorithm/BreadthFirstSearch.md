# 广度优先搜索

[TOC]

## 1、从上到下打印二叉树 III

请实现一个函数按照之字形顺序打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右到左的顺序打印，第三行再按照从左到右的顺序打印，其他行以此类推。

例如:
给定二叉树: [3,9,20,null,null,15,7],

```
  3
 / \
9  20
  /  \
 15   7
```


返回其层次遍历结果：

```
[
  [3],
  [20,9],
  [15,7]
]
```


提示：

1. 节点总数 <= 1000

**思路分析：层序遍历 + 双端队列**

利用双端队列的两端皆可添加元素的特性，设打印列表（双端队列） `tmp` ，并规定：

- 奇数层 则添加至 `tmp` **尾部** ，
- 偶数层 则添加至 `tmp` **头部** 。

**代码：**

```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        Queue<TreeNode> queue = new LinkedList<>();
        List<List<Integer>> res = new ArrayList<>();
        if(root != null) queue.add(root);
        while(!queue.isEmpty()) {
            LinkedList<Integer> tmp = new LinkedList<>();
            for(int i = queue.size(); i > 0; i--) {
                TreeNode node = queue.poll();
                if(res.size() % 2 == 0) tmp.addLast(node.val); // 偶数层 -> 队列头部
                else tmp.addFirst(node.val); // 奇数层 -> 队列尾部
                if(node.left != null) queue.add(node.left);
                if(node.right != null) queue.add(node.right);
            }
            res.add(tmp);
        }
        return res;
    }
}
```

## 2、获取你好友已观看的视频_1311

有 n 个人，每个人都有一个  0 到 n-1 的唯一 id 。

给你数组 watchedVideos  和 friends ，其中 watchedVideos[i]  和 friends[i] 分别表示 id = i 的人观看过的视频列表和他的好友列表。

Level 1 的视频包含所有你好友观看过的视频，level 2 的视频包含所有你好友的好友观看过的视频，以此类推。一般的，Level 为 k 的视频包含所有从你出发，最短距离为 k 的好友观看过的视频。

给定你的 id  和一个 level 值，请你找出所有指定 level 的视频，并将它们按观看频率升序返回。如果有频率相同的视频，请将它们按字母顺序从小到大排列。

**示例 1：**

 ![](/Users/didi/Desktop/zyk/Java_BigData/Java/DataStructureAndAlgorithm/img/1311_0.png)

```
输入：watchedVideos = [["A","B"],["C"],["B","C"],["D"]], friends = [[1,2],[0,3],[0,3],[1,2]], id = 0, level = 1
输出：["B","C"] 
解释：
你的 id 为 0（绿色），你的朋友包括（黄色）：
id 为 1 -> watchedVideos = ["C"] 
id 为 2 -> watchedVideos = ["B","C"] 
你朋友观看过视频的频率为：
B -> 1 
C -> 2
```

**思路分析：**

1. 使用广度优先搜索找到level层的朋友。
2. 统计level层的朋友看过的电影的数量。
3. 根据数量进行排序。

**代码：**

```java
class Solution {
   public List<String> watchedVideosByFriends(List<List<String>> watchedVideos, int[][] friends, int id, int level) {
        Map<String, Integer> map = new HashMap<>();
        boolean[] visited = new boolean[friends.length];
        Queue<Integer> queue  = new LinkedList<>();
        visited[id] = true;
        queue.offer(id);
        int current = -1;
        while (!queue.isEmpty()) {
            int count = queue.size();
            current++;
            for (int i = 0; i < count; i++) { // 遍历当前层所有的朋友
                Integer num = queue.poll();
                if (level == current) {  // 已经是level层的朋友，统计该层朋友观看电影的数量
                    List<String> list = watchedVideos.get(num);
                    for (String video : list) {
                        int c = map.getOrDefault(video, 0);
                        map.put(video, c + 1);
                    }
                    continue;
                }
                for (int sub: friends[num]) {
                    if (visited[sub]) {
                        continue;
                    }
                    visited[sub] = true;
                    queue.offer(sub);
                }
            }
        }
        // 排序
        List<Node> nodes = new ArrayList<>();
        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            nodes.add(new Node(entry.getKey(), entry.getValue()));
        }
        return nodes.stream()
                .sorted((n1, n2) -> {
                    if (n1.count == n2.count) {
                        return n1.videos.compareTo(n2.videos);
                    }
                    return n1.count - n2.count;
                })
                .map(node -> node.videos)
                .collect(Collectors.toList());
    }

    class Node {
        String videos;
        Integer count;
        public Node(String videos, Integer count) {
            this.videos = videos;
            this.count = count;
        }
    }
}
```

## 3. 腐烂的橘子-994

在给定的网格中，每个单元格可以有以下三个值之一：

值 0 代表空单元格；
值 1 代表新鲜橘子；
值 2 代表腐烂的橘子。
每分钟，任何与腐烂的橘子（在 4 个正方向上）相邻的新鲜橘子都会腐烂。

返回直到单元格中没有新鲜橘子为止所必须经过的最小分钟数。如果不可能，返回 -1。

思路：

从腐烂的橘子出发层次遍历，并且记录当前腐烂橘子的层数。

代码：

```java
class Solution {
    int[] dx = {-1, 0, 1, 0};
    int[] dy = {0, 1, 0, -1};

    public int orangesRotting(int[][] grid) {
        int X = grid.length, Y = grid[0].length;
        Queue<Integer> queue = new LinkedList<>();
        Map<Integer, Integer> deep = new HashMap<>();
      // 腐烂的橘子入队，并记录当前层数为0
        for (int x = 0; x < X; x++) {
            for (int y = 0; y < Y; y++) {
                if (grid[x][y] == 2) {
                    int pos = x * Y + y;
                    queue.offer(pos);
                    deep.put(pos, 0);
                }
            }
        }
      // 广度优先搜索
        int ans = 0;
        while (!queue.isEmpty()) {
            int pos = queue.poll();
            int x = pos / Y, y = pos % Y;
            for (int i = 0; i < 4; i++) {
                int nx = x + dx[i];
                int ny = y + dy[i];
                if (nx >= 0 && nx < X && ny >= 0 && ny < Y && grid[nx][ny] == 1) {
                    grid[nx][ny] = 2;
                    int npos = nx * Y + ny;
                    queue.offer(npos);
                    deep.put(npos, deep.get(pos) + 1);
                    ans = deep.get(npos);
                }
            }
        }
        for (int[] row: grid) {
            for (int o: row) {
                if (o == 1) {
                    return -1;
                }
            }
        }
        return ans;
    }
}
```

## 4. 找树左下角的值-513

给定一个二叉树，在树的最后一行找到最左边的值。

示例 1:

```
输入:
    2
   / \
  1   3
输出:
1
```


示例 2:

```
输入:
        1
       / \
      2   3
     /   / \
    4   5   6
       /
      7
输出:
7
```

**注意:** 您可以假设树（即给定的根节点）不为 **NULL**。

**思路：方法一：广度优先搜索**

遍历每一层的数据，记录该层的第一个

代码：

```java
public int findBottomLeftValue(TreeNode root) {
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    int ans = root.val;
    while(!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            TreeNode node = queue.poll();
            if (i == 0) {
                ans = node.val;
            }
            if (node.left != null) {
                queue.offer(node.left);
            }
            if (node.right != null) {
                queue.offer(node.right);
            }
        }
    }
    return ans;
}
```

类似的题：[199. 二叉树的右视图](https://leetcode-cn.com/problems/binary-tree-right-side-view/)

方法二：深度优先搜索**

中序遍历，记录第一次到达最大层的值。

代码：

```java
class Solution {
    int maxDeep = -1;
    int ans;

    public int findBottomLeftValue(TreeNode root) {
        midOrder(root, 0);
        return ans;
    }

    public void midOrder(TreeNode root, int deep) {
        if (root == null) return ;
        midOrder(root.left, deep + 1);
        if (deep > maxDeep) {
            maxDeep = deep;
            ans = root.val;
        }
        midOrder(root.right, deep + 1);
    }
}
```

