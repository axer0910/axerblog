---
title: Leetcode 114. Flatten Binary Tree to Linked List
tag: 算法
date: 2019-04-23
updated: 2019-04-23
---

Given a binary tree, flatten it to a linked list in-place.

For example, given the following tree:
![Alt text](./1555990055968.png)

把一个二叉树按照DFS层次把所有左数上的节点全部移到右树上来。可以先分别DFS左树和右树，确定到达叶子节点后，返回根，先暂存右节点，然后把根节点右节点指向=左节点，遍历右节点右子节点直到叶子节点，叶子节点再指向刚才暂存的右节点，再把根节点的左节点设置为null即可。

```Javascript
var flatten = function(root) {
  function flatten(node) {
    // 此时node为根节点
    if (!node) return;
    if (node.left) flatten(node.left);
    if (node.right) flatten(node.right);
    // 暂存原右节点
    let tmp = node.right;
    node.right = node.left;
    node.left = null;
    while(node.right) {
      node = node.right;
    }
    node.right = tmp;
  }
  flatten(root);
};
```