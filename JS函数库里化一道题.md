---
title: JS函数有关库里化一道题
tag: frontend
date: 2019-03-06
updated: 2019-03-06
---

编写curry.js

实现函数的分步调用

调用过程：
```javascript
var curry = require('./curry.js');// <- this is the file you make;
 
function add2(a, b, c) {
    return a + b + c;
}
 
var curried2 = curry(add2);
console.log(curried2(1)(2)(3)); // 6

```

要实现这样调用，需要使用闭包，函数内保存最终调用的函数以及已经传进的参数列表，如果参数列表个数符合目标调用的函数就直接运行。

```javascript
function curry(_fn, _args = []) {
  let args = _args;
  let fn = _fn;
  return function (param) {
    args.push(param);
    if (args.length === fn.length) {
      return fn(...args);
    } else {
      // 再返回一个闭包
      return curry(fn, args);
    }
  }
}

let curried = curry(function (a, b) {
  return a * b;
});

console.log(curried(4)(6)); // 24
```