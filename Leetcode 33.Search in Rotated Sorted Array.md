---
title: Leetcode.33 Search in Rotated Sorted Array
tag: 算法
date: 2019-03-10
updated: 2019-03-10
---

Suppose an array sorted in ascending order is rotated at some pivot unknown to you beforehand.

(i.e., [0,1,2,4,5,6,7] might become [4,5,6,7,0,1,2]).

You are given a target value to search. If found in the array return its index, otherwise return -1.

**Example 1**:

Input: nums = [4,5,6,7,0,1,2], target = 0
Output: 4
**Example 2**:

Input: nums = [4,5,6,7,0,1,2], target = 3
Output: -1

题目的意思是在一个先降后升的序列里找到target位置，并且复杂度要求是O(log n)。第一个想到的解法是把Pivot的位置找出来再二分查找即可。虽然写出来的代码能AC但是复杂度其实不是O(log n)的，因为有扫描找Pivot的过程：
```
var search = function(nums, target) {
    function binarySearch(start, end) {
        if (start > end) return -1;
        let middlePos = parseInt((end - start) / 2) + start;
        if (target > nums[middlePos]) {
            return binarySearch(middlePos + 1, end);
        }
        else if (target < nums[middlePos]) {
            return binarySearch(start, middlePos - 1);
        }
        return middlePos;
    }
    function findPivot() {
       // 找从降序到升序的那个转折点位置
        let i = 0;
        let j = 1;
        while (nums[j] > nums[i]) {
            i++;
            j++;
        }
        return j;
    }
    let pivotIndex = findPivot();
    if (nums[pivotIndex] === target) return pivotIndex;
    if (target <= nums[pivotIndex - 1] && target >= nums[0]) {
        return binarySearch(0, pivotIndex - 1);
    } else {
        return binarySearch(pivotIndex + 1, nums.length - 1);
    }
};
```
如果是O(log n)的解法那么就只能用二分，难点在于如何判断应该在中点的左边寻找还是右边寻找。为了保证能用二分法找到位置，首先得保证