---
title: Leetcode 19.Remove Nth Node From End of List
tag: 算法
date: 2019-03-20
updated: 2019-03-20
---

Given linked list: 1->2->3->4->5, and n = 2.

After removing the second node from the end, the linked list becomes 1->2->3->5.

即从链表移除倒数第n个节点。
最容易想到的解法当然是遍历一遍链表然后找出对应的节点删除即可，但肯定不是最优解。
如果是可以一次One pass的解法，分别用两个指针拉开他们直接的距离等于n，然后后一个指针移动到最后即可。
```javascript
var removeNthFromEnd = function(head, n) {
  // 分别用两个指针，拉开他们距离使他们之间的距离等于n，前一个指针的next修改为它的后一个的后一个即可。
  let cur = head; // 后节点
  let pre = head; // 前节点
  for (let i = 0; i < n; i++) {
    cur = cur.next;
    if (!cur) return head.next; // 后指针超出了那么直接取头部下一个即可
  }
  // cur移动到最后, pre跟着移动
  while(cur.next) {
    cur = cur.next;
    pre = pre.next;
  }
  // 跳过被删除的节点
  pre.next = pre.next.next;
  return head;
};
```
