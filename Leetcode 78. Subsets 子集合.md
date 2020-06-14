---
title: Leetcode 78. Subsets 子集合
tag: 刷题
date: 2019-03-25
updated: 2019-03-25
---

Example:

Input: nums = [1,2,3]
Output:
[
  [3],
  [1],
  [2],
  [1,2,3],
  [1,3],
  [2,3],
  [1,2],
  []
]

dfs思想解决，实际上就是先取出最后一个数字，放在一个集合里，然后前面的数字与这个集合进行组合。需要注意的是所有的集合都要保留，每向前一个数字都要与所有产生过的集合进行组合。
```Javascript
var subsets = function(nums) {
  if (nums.length === 0) return [[]];
  function dfs(index) {
    if (index === nums.length - 1) return [[nums[index]]];
    let current = [[nums[index]]];
    let rs = dfs(index + 1);
    rs.forEach(item => {
      current.push([nums[index], ...item]); // 当前的数组和dfs返回的结果集进行组合
    });
    return [...rs, ...current];
  }
  let rt = dfs(0);
  rt.unshift([]);
  return rt;
};
```