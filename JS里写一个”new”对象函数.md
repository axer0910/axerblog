---
title: JS里写一个"new"对象函数
tag: frontend
date: 2019-01-13
updated: 2019-01-13
---


**创建JS对象主要四步骤**
* 创建新对象
* 新对象的原型链连接到函数的prototype
* 执行构造函数
* 返回新创建的对象

```javascript
	// "new"对象函数

  function newObject(fn, constructorArgs) {
    // 创建新对象，设置原型链，调用一下构造函数
    let instanced = Object.create(fn.prototype); // 创建新对象，并且链接原型链到函数prototype
    fn.call(instanced, ...constructorArgs); // 调用构造函数
    return instanced;
  }

  function MyTest(arg, arg2) {
    this.aaa = arg;
    this.bbb = arg2;
  }

  MyTest.prototype.getAaa = function () {
    return this.aaa;
  };

  MyTest.prototype.getBbb = function () {
    return this.bbb;
  };

  let myTest = newObject(MyTest, [111, 222]);
  console.log(myTest);

  console.log(myTest.getAaa()); // 111
  console.log(myTest.getBbb()); // 222
console.log(myTest instanceof MyTest); // true，判断Object.getPrototypeOf(myTest) === MyTest.prototype。
```