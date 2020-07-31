---
title: Leetcode	Container With Most Water   
tag: 算法
date: 2019-03-02
updated: 2019-03-02
---

![Alt text](./1551539203993.png)
给定 n 个非负整数 a1,a2,...,an，其中每个代表一个点坐标（i,ai）。

n 个垂直线段例如线段的两个端点在（i,ai）和（i,0）。

找到两个线段，与 x 轴形成一个容器，使其包含最多的水。

备注：你不必倾倒容器。

一开始做法是从高到底排个序把最高的和第二高的柱子的面积乘一下即可，实际上是不对的，如果是[8,6,7]这样的情况会变成计算6x1的面积，实际上最大面积是7x2...

再次观察这个算最大面积实际上就是求min(i1, i2)*(a1-a2)的最大值。要算出这个高度，必然是两个柱子x两个柱子之间的距离。如何求最大值呢。首先两个柱子距离肯定是越大越好，但是两头的柱子高度无法保证是最高的，那么此时就要想办法在缩短距离的时候尽可能提高柱子的高度。分别用两个指针变量保存前面一个柱子和后面一个柱子的位置。并且初始化一个变量存结果最大值。当左边的高度小于右边的高度左指针往右走，当右边的高度小于左边的高度右指针往左走，这个过程中乘以距离算出面积。在两个指针相互逼近的时候一定会经过最大值，此时返回最大值即可。
```javascript
var maxArea = function(heights) {
    // 左右两个指针，分别往中间扫描，不断使两边的高度相互靠近，在这过程中一定会扫描到最大的结果面积
    let left = 0;
    let right = heights.length - 1;
    let max = 0;
    while (right > left) {
        max = Math.max(max, Math.min(heights[left], heights[right]) * (right - left));
        if (heights[left] < heights[right]) {
            left ++;
        } else {
            right --;
        }
    }
    return max;
};
```