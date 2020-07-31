---
title: Leetcode 3Sum
tag: 算法
date: 2019-03-02
updated: 2019-03-02
---

2Sum的升级版，需要列出数组中三个数相加为0的所有情况。解题方法是可以先固定数组中的一个数字，这样就可以变成2sum的问题。直接固定一个数字并且套用2sum那种hashmap的办法会有重复结果出现，并且做去重处理会超时。再次观察其实就是求固定的那个数字的相反数，如果把数组给排序了，并且用两个指针使用互相逼近的方法去找三者和为0的情况，那么就可以以线性复杂度找到三个数字。
```javascript
var threeSum = function(nums) {
  // 排序后双指针找target相加为0即可。
  nums = nums.sort((a, b) => a - b);
  let rs = [];
  let find = function(start, end, target) {
    let l = start, r = end;
    let m = new Map();
    while(1) {
      if (l >= r) break;
      if (nums[l] + nums[r] + target === 0) {
        m.set(nums[l], nums[r]); // 去除重复，像[0,0,0,0,0]这种条件下一次寻找会多次满足三者相加为0的情况，只需要保留一个。
        r --;
        l ++;
      }
      else if (nums[l] + nums[r] + target < 0) {
        l ++;
      } else {
        r --;
      }
    }
    m.forEach((item, index) => {
      rs.push([item, index, target]);
    });
  }
  for (let i = 0; i < nums.length - 2; i ++) {
  // 数组倒数第一和第二个就不需要放到target了
    if (nums[i] === nums[i - 1]) continue;
    find(i + 1, nums.length - 1, nums[i]);
  }
  return rs;
}; 
```

