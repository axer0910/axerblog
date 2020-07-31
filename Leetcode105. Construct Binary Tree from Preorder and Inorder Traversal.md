---
title: Leetcode105. Construct Binary Tree from Preorder and Inorder Traversal
tag: 算法
date: 2019-03-06
updated: 2019-03-06
---


Given preorder and inorder traversal of a tree, construct the binary tree.

Note:
You may assume that duplicates do not exist in the tree.

For example, given：
preorder = [3,9,20,15,7]
inorder = [9,3,15,20,7]
Return the following binary tree:
![Alt text](./1555988196340.png)

题目的意思是根据中序遍历和层次遍历构建一个唯一的二叉树。根据层次遍历的特点，是可以知道第一个节点肯定是根。那么可以根据他是根的特点去先序遍历里面找这个根在先序遍历里的位置，按题目样例举例：
[**3**,9,20,15,7] 此时3为根，找**3**在中序遍历的位置:[9,**3**,15,20,7]。
此时3是根，根据中序遍历左中右的特点9一定是在根节点3的左树上，此时把9作为根，并且以3为查找右边界找到9在中序遍历里的位置:
[3,**9**,20,15,7] 
[**|** **9**,**|**3,15,20,7]
同样的道理把20再作为根，把3作为查找的左边界找到20在中序遍历里的位置。
[3,9,**20**,15,7] 
[9,3**|**,15,**20**,7**|**]

中序遍历里20左右分别还有15和7，一样的做法去查找并确定15和7的位置即可。实际上就是一个二分的查找思路。
具体代码：
```javascript
var buildTree = function(preorder, inorder) {
  let startPre = 0;
  function constructor(istart, iend) {
    if (istart > iend) return null;
    let mid, rootNode;
    for (let i = istart; i <= iend; i++) {
      if (preorder[startPre] === inorder[i]) {
        // 找到了根节点在中序遍历中的位置
        mid = i;
        break;
      }
    }
    startPre++; // 下一个先序遍历作为根节点
    rootNode = new TreeNode(inorder[mid]);
    rootNode.left = constructor(istart, mid - 1);
    rootNode.right = constructor(mid + 1, iend);
    return rootNode;
  }
  return constructor(0, inorder.length - 1);
};
```
