# 深度优先搜索

[TOC]

## 1. 从前序与中序遍历序列构造二叉树-105

根据一棵树的前序遍历与中序遍历构造二叉树。

注意:
你可以假设树中没有重复的元素。

例如，给出

```
前序遍历 preorder = [3,9,20,15,7]
中序遍历 inorder = [9,3,15,20,7]
```

返回如下的二叉树：

```
    3
   / \
  9  20
    /  \
   15   7
```

思路：

递归的去构建左右子树。

代码：

```java
class Solution {
    private Map<Integer, Integer> indexMap;

    public TreeNode buildTree(int[] preorder, int[] inorder, int pleft, int pright, int ileft, int iright) {
        if (pleft > pright) {
            return null;
        }
        int iroot = indexMap.get(preorder[pleft]);
        TreeNode root = new TreeNode(preorder[pleft]);
        int size = iroot - ileft;
        root.left = buildTree(preorder, inorder, pleft + 1, pleft + size, ileft, iroot - 1);
        root.right = buildTree(preorder, inorder, pleft + size + 1, pright, iroot + 1, iright);
        return root;
    }

    public TreeNode buildTree(int[] preorder, int[] inorder) {
        int n = preorder.length;
        // 构造哈希映射，帮助我们快速定位根节点
        indexMap = new HashMap<Integer, Integer>();
        for (int i = 0; i < n; i++) {
            indexMap.put(inorder[i], i);
        }
        return buildTree(preorder, inorder, 0, n - 1, 0, n - 1);
    }
}
```

类似题：[106. 从中序与后序遍历序列构造二叉树](https://leetcode-cn.com/problems/construct-binary-tree-from-inorder-and-postorder-traversal/)

## 2. 在二叉树中分配硬币-979

给定一个有 N 个结点的二叉树的根结点 root，树中的每个结点上都对应有 node.val 枚硬币，并且总共有 N 枚硬币。

在一次移动中，我们可以选择两个相邻的结点，然后将一枚硬币从其中一个结点移动到另一个结点。(移动可以是从父结点到子结点，或者从子结点移动到父结点。)。

返回使每个结点上只有一枚硬币所需的移动次数。

示例：

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2019/01/19/tree2.png)

```
输入：[0,3,0]
输出：3
解释：从根结点的左子结点开始，我们将两枚硬币移到根结点上 [移动两次]。然后，我们把一枚硬币从根结点移到右子结点上。
```

思路：

定义 `dfs(node)` 为这个节点所在的子树中金币的 过载量，也就是这个子树中金币的数量减去这个子树中节点的数量。接着，我们可以计算出这个节点与它的子节点之间需要移动金币的数量为 `abs(dfs(node.left)) + abs(dfs(node.right))`，这个节点金币的过载量为 `node.val + dfs(node.left) + dfs(node.right) - 1`。

代码：

```java
class Solution {
    int ans;
    public int distributeCoins(TreeNode root) {
        ans = 0;
        dfs(root);
        return ans;
    }

    public int dfs(TreeNode node) {
        if (node == null) return 0;
        int L = dfs(node.left);
        int R = dfs(node.right);
        ans += Math.abs(L) + Math.abs(R);
        return node.val + L + R - 1;
    }
}
```

