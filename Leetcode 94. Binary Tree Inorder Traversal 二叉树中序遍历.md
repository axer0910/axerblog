---
title: Leetcode 94. Binary Tree Inorder Traversal 二叉树中序遍历
tag: 算法
date: 2019-03-20
updated: 2019-03-20
---

Given a binary tree, return the inorder traversal of its nodes' values.

Input: [1,null,2,3]
   1
    \
     2
    /
   3

Output: [1,3,2]

中序遍历即遍历顺序为左，根，右。按照这个顺序很容易写出递归代码：
```javascript
var inorderTraversal = function(root) {
  let rs = [];
  function traversal(node) {
    if (node.left) traversal(node.left);
    rs.push(node.val);
    if (node.right) traversal(node.right);
  }
  if (!root) return [];
  traversal(root);
  return rs;
};
```

但是题目要求最好不使用递归，那么也可以用栈的方式进行递归。思路是先把所有的左节点push到stack里面，然后判断左节点是否存在，若不存在，此时出栈，节点即为根节点，再指向他的右节点。如果stack长度为0，那么遍历结束。
```javascript
var inorderTraversal = function(root) {
  if (!root) return [];
  let curr = root;
  let stack = [];
  stack.push(curr);
  curr = curr.left;
  let rs = [];
  while(curr || stack.length > 0) {
    // 有左节点就进行入栈，否则就出栈并读取当前值
    if (curr) {
      stack.push(curr);
      curr = curr.left;
    } else {
      let poped = stack.pop();
      rs.push(poped.val);
      curr = poped.right;
    }
  }
  return rs;
};
```