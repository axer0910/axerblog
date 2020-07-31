---
title: Leetcode.34 Find First and Last Position of Element in Sorted Array
tag: 算法
date: 2019-03-09
updated: 2019-03-09
---

Given an array of integers nums sorted in ascending order, find the starting and ending position of a given target value.

Your algorithm's runtime complexity must be in the order of O(log n).

If the target is not found in the array, return [-1, -1].
**Example 1:**

Input: nums = [5,7,7,8,8,10], target = 8
Output: [3,4]
**Example 2:**

Input: nums = [5,7,7,8,8,10], target = 6
Output: [-1,-1]

题目要求了O(log n)复杂度，那么肯定是要用到二分的思想来结局。第一个解法想法比较简单，就是先按照正常的二分找到数字，如果找到了target的位置再分别寻找target两头的边界：
```
var searchRange = function(nums, target) {
    function findMaxMinPos(index) {
        let l = index;
        let r = index;
        let i = index;
        while (nums[l] === nums[i]) {
            l--;
        }
        while (nums[r] === nums[i]) {
            r++;
        }
        return [l + 1, r - 1];
    }
    function search(start, end) {
        if (start > end) return [-1, -1];
        let middle = parseInt((end - start) / 2) + start;
        // 相等的情况
        if (target === nums[middle]) {
            return findMaxMinPos(middle);
        }
        let pos = findMaxMinPos(middle);
        if (target < nums[middle]) {
            // 右端要移到相同的nums[middle]最左端
            return search(start, pos[0] - 1);
        } else {
            return search(pos[1] + 1, end);
        }
    }
    return search(0, nums.length - 1);
};
```
但是这个解法不是完全的O(log n)复杂度，因为找到targe位置后还有寻找边界的过程，最坏情况是会变成O(logn)（如果数组从头到尾都是target）。如果只用二分法来解决这道题目也是可以的，那就是用两次二分法，先找到左边界再找到右边界即可：
```
var searchRange = function(nums, target) {
    // 先二分找到左端点，然后二分再找到右端点
  let low = 0;
  let high = nums.length - 1;
  while (low < high) {
    let mid = parseInt((high - low) / 2) + low;
    if (nums[mid] < target) {
      low = mid + 1;
    } else {
      high = mid;
    }
  }
  let left = low;
  if (nums[left] !== target) return [-1, -1];
  high = nums.length; // 不是nums.length-1是因为如果low正好在nums.length - 1的位置会进不了后面的循环（比如[9]这样的情况）。
  while (low < high) {
    let mid = parseInt((high - low) / 2) + low;
    if (nums[mid] <= target) {
      low = mid + 1;
    } else {
      high = mid;
    }
  }
  return [left, high - 1]; // high-1因为上面多了1所以结果要减去。
};
```