---
title: Leetcode 46. Permutation
tag: 算法
date: 2019-03-06
updated: 2019-03-06
---

列出所有的全排列，有两种解法。

**DFS解法**
第一种就是通过DFS遍历出所有的组合，如下图所示，从第一个格子开始依次填空，第一个空可以填1或2或3，当填完第一个格子后，第二个格子只剩下两种选择，当填完第二个格子后，第三个格子只剩下一种选择。

代码里需要注意的地方是需要一个变量标记每一层递归访问过的数字（图里就是第二层的1被访问过，第二层就不能再从1进行DFS而需要从2开始），同时每一层需要一个新变量去保存当前遍历的结果。

![Alt text](./1551866218683.png)

代码：
```javascript
var permute = function(nums) {
    let rs = [];
    // visited记录当前组合遍历过的数组
    function dfs(currentP, visited) {
        for (let i = 0; i < nums.length; i++) {
            if (visited[i]) continue;
            visited[i] = true;
            currentP.push(nums[i]); // 加入当前组合，同时向下一层遍历
            dfs([...currentP], visited);
            currentP.pop(); // 退出当前组合，给下一个循环加入新的组合
            visited[i] = false;
        }
        if (currentP.length === nums.length)
          rs.push(currentP);
    }
    dfs([], {});
    return rs;
};
```
**插入法**
1，2，3的全排列：
从最后一位开始取排列组合，然后用前一位不停地插入已经生成过的组合序列当中：
长度位1的组合：
3
长度为2的组合，用2分别插入到第一位和最后一位
2  3
3  2
长度为3的组合，分别用1插入到2，3和3，2的组合当中
第一组：1,2,3
               2,1,3
               2,3,1
第二组：1,3,2
			   3,1,2
			   3,2,1
可以用递归实现：
```javascript
var permute = function(nums) {
  let rs = [];
  let lastIndex = nums.length - 1;
  function insert(startIndex) {
    let tmpRs = []; // 当前长度排列结果
    if (startIndex === lastIndex) {
      return [[nums[startIndex]]];
    }
    let insertedArr = insert(startIndex + 1);
    for (let inserted of insertedArr) {
      for (let spliceIndex = 0; spliceIndex <= inserted.length; spliceIndex++) {
        inserted.splice(spliceIndex, 0, nums[startIndex]);
        tmpRs.push([...inserted]);
        inserted.splice(spliceIndex, 1);
      }
    }
    return tmpRs;
  }
  
  return insert(0);
};
```