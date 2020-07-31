---
title: Leetcode 20. Valid Parentheses
tag: 算法
date: 2019-03-08
updated: 2019-03-08
---

Example 1:

Input: "()"
Output: true
Example 2:

Input: "()[]{}"
Output: true
Example 3:

Input: "(]"
Output: false
Example 4:

Input: "([)]"
Output: false
Example 5:

Input: "{[]}"
Output: true

验证是否是有效的成对括号。解决方法就是使用一个栈，如果碰到( [ { 就入栈，如果碰到) ] }就出栈。需要注意的是因为是成对出现，所以出栈的时候要验证是否和入栈的符号相匹配（遇到**}**符号那么出栈的内容就一定要是**{**）

```javascript
/**
 * @param {string} s
 * @return {boolean}
 */
var isValid = function(s) {
  if (!s) return true;
  s = s.split('');
  let stack = [];
  let hasPushed = false;
  let dict = {
    '(': ')',
    '{': '}',
    '[': ']'
  }
  
  for (let i = 0; i < s.length; i++) {
    let char = s[i];
    if (s[i] === '(' || s[i] === '[' || s[i] === '{') {
      stack.push(s[i]);
      if (!hasPushed) hasPushed = true;
    }
    else if (dict[stack[stack.length - 1]] === s[i]) {
      stack.pop();
    }
    else {
      return false;
    }
  }
  if (!hasPushed) return false;
  if (stack.length === 0){
    return true
  }
   return false;
};
```