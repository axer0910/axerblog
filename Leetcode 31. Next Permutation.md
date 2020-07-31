---
title: Leetcode 31. Next Permutation
tag: 算法
date: 2019-03-08
updated: 2019-03-08
---

碰到这类题目，可以先写几个例子：
7 2 5 **2** 3 1
7 2 5 3 **1** 2
7 **2** 5 3 2 1
7 3 1 2 **2** 5

从最后往前开，一开始都是升序的到了加粗的地方就开始降序。在Permutation这个问题里，如果序列里某一段都是降序排列那么这一段就是已经排序完成了的。如7 **2** 5 3 2 1，5，3，2，1即是这个序列里4位排序的最大值。四位的最大值是5，3，2，1，那么五位呢？这时不得不改变第5位的2来增加数值。如何改变？为了使增量最小，在前4位中比第5位大的数(5, 3)中找一个最小的数，即数字3。用3替换2，而剩下5, 2, 2, 1四个数字要组成最低4位。由于第5位已经从2增加到3，同样为了使增量最小，我们希望剩下的4位数尽可能小。所以将他们从低到高位降序排列即可。
总结一下：最后往前找升序序列的最后一个的后一个数字，再找到这个数字后面比它大的数字（7 **2** 5 3 2 1中的比2大的3），交换一下（7 **3** 5 **2** 2 1），最后3后面的序列转置一下即可（7 3 1 2 **2** 5）。
代码：
```javascript
var nextPermutation = function(nums) {
  let j = nums.length - 1;
  function swap(ia, ib) {
    let t = nums[ia];
    nums[ia] = nums[ib];
    nums[ib] = t;
  }
  function reverse(start, end) {
    while (start < end) {
      swap(start, end);
      start++;
      end--;
    }
  }
  for (let i = nums.length - 2; i >= 0; i--) {
    if (nums[i] >= nums[j]) {
      j --;
      continue;
    } else {
      j = nums.length - 1;
      while (j >= 0) {
        if (nums[j] > nums[i]) {
          swap(j, i);
          reverse(i + 1, nums.length - 1);
          return;
        } else {
          j --;
        }
      }
      break; 
    }
  }
  // 已经是最后一个排列，直接颠倒数组
  reverse(0, nums.length - 1);
};
```