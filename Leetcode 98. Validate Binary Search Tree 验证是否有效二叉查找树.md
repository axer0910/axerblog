---
title: Leetcode 98. Validate Binary Search Tree 验证是否有效二叉查找树
tag: 算法
date: 2019-03-20
updated: 2019-03-20
---

Given a binary tree, determine if it is a valid binary search tree (BST).

Assume a BST is defined as follows:

The left subtree of a node contains only nodes with keys less than the node's key.
The right subtree of a node contains only nodes with keys greater than the node's key.
Both the left and right subtrees must also be binary search trees.
因为这道题设定为一般情况左<根<右，那么就可以用中序遍历来做。因为如果不去掉左=根这个条件的话，那么下边两个数用中序遍历无法区分：

中序遍历一遍，存在一个数字判断是否为递增序列即可
```javascript
var isValidBST = function(root) {
  if (!root) return true;
  let stack = [];
  function check(node) {
    let leftRs = true;
    let rightRs = true;
    if (node.left) {
      leftRs = check(node.left);
    }
    // 取值
    stack.push(node.val);
    if (stack.length > 1 && stack[stack.length - 1] <= stack[stack.length - 2]) return false;
    if (node.right) {
      rightRs = check(node.right);
    }
    return leftRs && rightRs;
  }
  return check(root);
};
```