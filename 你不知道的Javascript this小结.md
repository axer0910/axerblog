---
title: 你不知道的Javascript this小结
tag: frontend
date: 2019-01-06
updated: 2019-01-06
---


什么是this
>this在javascript是**运行时**进行绑定的，并不是在编写时候进行绑定。它的上下文取决于函数调用时候的各种条件。this的绑定和函数声明没有任何关系，只取决于函数的调用方式。

总之分析this对象是什么的时候一定要分析调用它的位置以及调用它的方式。

### 默认绑定
```javascript
function foo() {
	console.log(this.a);
}

var a = 2;

foo(); // 2
```
foo()调用的环境是全局，这个时候函数里面的this就默认绑定到了全局对象。如果在foo里面使用严格模式（添加"use strict"）那么默认绑定不能绑定全局对象，此时this变成了undefined

### 隐式绑定
```javascript
var a = 3;
function fun(){
	console.log(this.a);
}

var obj = {
	a: 2,
	foo: fun
};

obj.foo(); // 2
```
这里输出2而不是3，因为应用了隐式绑定规则。当foo()被调用时候，加上了obj的引用，当这个函数有调用上下文对象的时候，隐式绑定规则会把this绑定到调用的上下文对象，也就是示例代码中的obj对象。虽然fun的定义在全局环境中，但由于obj中的foo属性属于obj环境，这个时候运行obj.foo()，fun函数里面的this会绑定到obj对象。

常见的丢失this问题：
```javascript
function foo() {
    console.log(this.a);
  }
function doFoo(fn) {
  fn(); // 这里运行fn，fn函数体内的this将是全局对象
}
var obj = {
  a: 2,
  foo: foo()
}

var a = 'oops, global!';
doFoo(obj.foo); // oops, global!
```
使用this一定要注意掉起函数的位置。


### 显示绑定
如果不想在某个对象内部包含函数调用，而是要在某个对象上强制调用函数，那么可以使用`call(...)和apply(...)`方法。这两个方法第一个参数可以传入一个对象，接着在调用函数的时候将这个对象绑定到this，这个就是显示绑定。
```javascript
function foo(){
	console.log(this.a);
}

var obj = {
	a : 2
};

foo.call(obj); // 使用obj对象作为foo里面的this，这个时候this.a就是obj.a，输出2。
```

写一个bind辅助函数，用来绑定指定的this对象与函数对象，并且返回绑定this后的新函数。
```javascript
function bind(fn, obj) {
    return function () {
      return fn.apply(obj, arguments);
    }
  }

 function foo(something) {
   console.log(this.a, something);
 }

 var context = {a: 123};

 var newfoo = bind(foo, context);
 newfoo('hello'); // 123 "hello"
```
### new绑定

JS中的new操作符和其他语言很不一样，实际上调用new的时候构造函数只是一个普通的函数，并不是属于某个类，也不会有什么实例化的动作。

>使用new来调用函数，或者说发生构造函数调用时，会自动执行下面的操作：

>创建（构造）一个全新的对象
>这个新对象会被执行[[Prototype]]连接
>这个新对象会被绑定到函数调用的this
>如果函数没有返回其他对象，那么new表达式中的函数调用会自动返回这个新对象

执行Prototype连接会在原型链章详细说明，有关this的是1，3，4步骤

```javascript
function foo(a) {
	this.a = a;
}

var bar = new foo(2);

console.log(bar.a); //2

// 使用 new 来调用 foo(..) 时，先构造一个空的bar对象，然后将foo中的this替换为bar，再运行this.a = a，最后返回这个初始化好的bar对象。
```
this的几个判断规则总结

>函数是否在new中被调用（new绑定）？如果是的话this绑定的是新创建的对象
>函数是否通过call、apply（显式绑定）或者bind（硬绑定）调用？如果是的话this绑定的是指定的对象
>函数是否在某个上下文对象中调用（隐式绑定）？如果是的话this绑定的是那个上下文对象
>如果都不是的话，使用默认绑定。如果在严格模式下，就绑定到undefined，否则绑定到全局对象
###绑定例外
使用apply，call，bind函数绑定`null`，`undefined`作为this的话，会应用this的默认绑定规则。

```javascript
function foo(a, b){
    console.log("a : " + a + ", b : " + b);
}
foo.apply(undefined, [4, 5]); // a : 4, b : 5 相当于使用es6展开符，foo(...[4, 5])
var bar = foo.bind(null, 2); // 实现柯里化，预先添加2传入到foo的a
bar(3); // a : 2, b : 3 
```

### 词法绑定（es6）
es6里面规定了一种特殊的函数写法叫做箭头函数（=>），使用这个函数的不使用上面几种定义的this规则，而是根据外层（函数或者全局）作用域来定义this

```javascript
function foo(){
//返回一个箭头函数
	return (a) => {
		//this继承自foo()
		console.log(this.a);
	}
}

var obj1 = {
	a: 2
}

var obj2 = {
	a: 3
}

var bar = foo.call(obj1);

bar.call(obj2); // 2，不是3！,使用了箭头函数不适用call替换this规则，仍然使用foo.call(obj1)替换的this环境。
```