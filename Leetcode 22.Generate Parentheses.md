---
title: Leetcode 22.Generate Parentheses
tag: 算法
date: 2019-03-08
updated: 2019-03-08
---

Given n pairs of parentheses, write a function to generate all combinations of well-formed parentheses.
For example, given n = 3, a solution set is:
[
  "((()))",
  "(()())",
  "(())()",
  "()(())",
  "()()()"
]
题目意思就是生成成对括号，因为是成对括号，所以有几个左括号就一定有几个右括号。可以用DFS思想解决：先产生最大深度n个左括号，再往上生成n个右括号。同时使用两个变量记录了生成了几个左括号和右括号。
需要注意的是，为了防止生成 **)(** 这样的组合，需要始终保持right变量大于left变量。

```javascript
var generateParenthesis = function(n) {
    let res = [];
    function dfs(str, left, right) {
        if (left === 0 && right === 0) {
            res.push(str);
            return;
        }
        if (left > 0) {
            dfs(str + '(', left - 1, right); // 递归出所有左括号情况
        }
        if (right > 0 && right > left) {
            dfs(str + ')', left, right - 1);
        }
    }
    dfs('', n, n);
    return res;
};
```