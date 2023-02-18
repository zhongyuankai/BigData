# 二分法

[TOC]

## 1、有序矩阵中第K小的元素-378.

给定一个 *`n x n`* 矩阵，其中每行和每列元素均按升序排序，找到矩阵中第 `k` 小的元素。
 请注意，它是排序后的第 `k` 小元素，而不是第 `k` 个不同的元素。

示例：

```
matrix = [
   [ 1,  5,  9],
   [10, 11, 13],
   [12, 13, 15]
],
k = 8,

返回 13。
```

提示：
你可以假设 k 的值永远是有效的，1 ≤ k ≤ n2 。

思路分析：

- 第k小的数字意味着小于等于它的元素一共有k个，大于它的数字有n*n-k个。假设某个数为mid；
- 如果小于等于mid的元素个数小于k，说明mid不是第k小的数，比mid小的数就更不可能是了。所以第k小的数至少是mid + 1。
- 如果小于等于mid的元素个数大于等于k，说明mid可能为第k小的数，比它小的数也有可能是答案。

```java
public int kthSmallest2(int[][] matrix, int k) {
    int n = matrix.length - 1;
    int left = matrix[0][0], right = matrix[n][n];
    while(left < right){
        int mid = left + (right - left) / 2;
        int count = countNotMoreThanMid(matrix, mid, n);
        if(count < k)
            left = mid + 1;
        else
            right = mid;
    }
    return left;
}

private int countNotMoreThanMid(int[][] matrix, int mid, int n){
    int count = 0;
    int x = 0, y = n;
    while(x <= n && y >= 0){
        if(matrix[y][x] <= mid){
            count += y + 1;
            x++;
        }else{
            y--;
        }
    }
    return count;
}
```

