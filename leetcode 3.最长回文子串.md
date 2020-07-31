---
title: leetcode 3.最长回文子串
tag: 算法
date: 2019-02-27
updated: 2019-02-27
---


经典题目，求一个字符串里面最长的回文，babad里的bab或者cbbd里面的bb。写了两个解法，第一个是使用动态规划，第二个是马拉车算法。
	  **动态规划**
	动态规划解法的思想是从长度1开始增量，判断内部的字串是不是回文并且保存在一个二维数组里。举例来说，判断abba，增量是1的时候，abba每个字符都是回文，增量从2开始的时候，先判断他们的首尾，首位相等的时候再判断里面的部分是否是回文。因为里面的部分即当前增量-1已经判断过了，所以直接读取结果即可。abba这个例子当增量是4的时候，即判断abba是否回文，因为头部a和尾部a相等，所以判断bb是否回文。bb是否回文已经在增量是2的时候判断过并且是真，所以abba也是回文。

```javascript
var longestPalindrome = function(s) {
    let str = s.split('');
    let dp = new Array(str.length);
    let rsStrPos = 0;
    let maxLen = 0
    for (let i = 0;i < dp.length; i++) {
        dp[i] = new Array(str.length);
    }
    // sLen是判断回文的增量（长度0和1一定是回文，2如果两边相等那么也是回文）
    for (let sLen = 0; sLen <= str.length; sLen++) {
         for (let charIndex = 0; charIndex < str.length; charIndex++) {
            if (sLen === 0 || sLen === 1) {
                dp[charIndex][sLen] = true;
            }
            // 头尾相等，判断中间是否已经存在过判断回文的结果
            else if (str[charIndex] === str[charIndex + sLen - 1]) {
                // abba这种情况，当前增量是4，需要判断bb这个字符串起点当前+1，当前增量-2的情况
                dp[charIndex][sLen] = dp[charIndex + 1][sLen - 2];
            }
            else {
                dp[charIndex][sLen] = false;
            }
            if (dp[charIndex][sLen] && sLen > maxLen) {
                // 记录回文最大值位置
                maxLen = sLen;
                rsStrPos = charIndex;
            }
         }
    }
    return s.substr(rsStrPos, maxLen);
};
```

**马拉车算法**
这个算法比较难，也是一部分动态规划的思想。有待整理