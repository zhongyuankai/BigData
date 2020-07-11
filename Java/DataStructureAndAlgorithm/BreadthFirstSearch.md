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

