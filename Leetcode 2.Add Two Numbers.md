---
title: Leetcode 2.Add Two Numbers 
tag: 算法
date: 2019-02-19
updated: 2019-02-19
---

 * Input: (2 -> 4 -> 3) + (5 -> 6 -> 4)
 * Output: 7 -> 0 -> 8
 * Explanation: 342 + 465 = 807.
 
 第一次写法直接想到先把链表转成数字，然后相加再拆回链表。提交的时候大部分case可以过，然而一旦链表超出31位js就会因为精度问题溢出变成科学计数法相加结果就不对了。
 ```
 var addTwoNumbers = function(l1, l2) {
    function getNumber(linkedList) {
        let numberArr = [];
        for (let currentLink = linkedList; ; currentLink = currentLink.next) {
            numberArr.push(currentLink.val);
            if (!currentLink.next) break;
        }
        return new Number(numberArr.reverse().join(''));
    }
    let strNumber = (getNumber(l1) + getNumber(l2)).toString();
    let digits = [...strNumber];
    console.log('add is', getNumber(l1).toLocaleString());
    digits = digits.reverse();
    let rsLinkedList = {
        val: digits[0]
    };
    let head = rsLinkedList;
    for (let i = 1; i < digits.length; i++) {
        head.next = {
            val: digits[i]
        }
        head = head.next;
    }
    return rsLinkedList;
};
 ```
 正确的做法是每个链表按位相加添加到新链表节点上，并且设置进位，如果进位是1的时候强制添加一个新的链表节点。（比如5+5结果10，需要两个链表节点）
 ```
 var addTwoNumbers = function(l1, l2) {
    let head1 = l1;
    let head2 = l2;
    let head3 = new ListNode(0);
    let rsHead = head3;
    let carry = 0; // 进位标识，0或者1
    while(head1 || head2) {
        if (head1) {
            carry += head1.val;
            head1 = head1.next;
        }
        if (head2) {
            carry += head2.val;
            head2 = head2.next;
        }
        head3.val = carry % 10;
        carry = parseInt(carry / 10);
        if (head1 || head2) {
            head3.next = new ListNode(carry);
            head3 = head3.next;
        }   
    }
    // 如果最后一次相加需要进位，新增一个节点
    if (carry === 1) {
        head3.next = new ListNode(carry);
    }
    return rsHead;
};
 ```
 
