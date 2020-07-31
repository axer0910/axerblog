---
title: 你不知道的Javascript闭包小结
tag: frontend
date: 2019-01-06
updated: 2019-01-06
---


### 闭包的定义
很简单：
>当函数可以记住并且访问所在的词法作用域时候，就产生了闭包，即使函数是在当前词法作用域域之外执行。

一段简单的代码
```javascript
	function foo() {
		var a = 2;
		function bar() {
			console.log(a);
		}
		return bar;
	}
	var baz = foo()
	bar(); // 2
```

baz得到了foo内部的函数bar的引用。bar内部的a，由于根据作用域查找规则，找到了位于foo函数体内的变量a，所以运行bar()时候var a = 2不会被回收，输出了2。


### 循环和闭包

使用for循环定义一个延时器：
```javascript
for (var i = 0; i <5; i ++) {
  setTimeout(function () {
    console.log(i);
  }, 500);
}
```
这段代码不会输出0到4而是输出5个5。实际上因为js的代码作用域问题，i的作用域不是在每个循环体里面而是全局作用域。这段代码个人理解其实类似于：
```javascript
var tmp = 0;
do {
  var i = tmp;
  setTimeout(function () {
    console.log(i);
  }, 500);
  i += 1;
  tmp = i;
} while (tmp < 5);
```
var i的作用域和tmp实际上同级的，声明都会被提升到上层。在最后一次运行循环的时候，i的值被改成5，tmp的值被改成5，这个时候tmp<5不满足退出循环。此时500毫秒后运行setTimeout，函数体内的i将向外寻找i的定义，发现是5，随后输出5个5。

如果想按照预期让for循环输出0~4，需要在每次循环里实现一个单独的作用域：
```javascript
for (var i = 0; i < 5; i++) {
    (function (j) {
      setTimeout(function () {
        console.log(j);
      }, 500);
    })(i)
  }
```
每执行一次循环，都会在单独的一个作用域里面创建一个j变量。
如果使用let语法，效果和上面相同，将会在每个循环块里面生成一个单独的作用域。
```javascript
for (let i = 0; i < 5; i++) {
    setTimeout(function () {
      console.log(i);
    }, 500);
  }
```
### 闭包应用
实现一个简单模块管理：
```javascript
function myModules() {
    let modules = [];
    function register(name, deps, moduleFun) {
      // 读取数组里面依赖，传给modules
      let depFuns = [];
      for(let dep of deps) {
        depFuns.push(modules[dep]);
      }
      modules[name] = moduleFun.apply(null, depFuns); // 序列号modules传进函数
      // modules[name] = moduleFun(...modules); // es6写法
    }
    function getModule(name) {
      return modules[name];
    }
    return {register, getModule};
}

let Modules = myModules();

Modules.register('ModuleFun1', [], function() {
   let myStrs = '';
   function setMyVarible(someString) {
      myStrs = someString;
   }
   function getMyVarible() {
     console.log(myStrs);
     return myStrs;
   }
   function echoMyVarible() {
     console.log('my varible: ' + myStrs);
   }

   return {
    setMyVarible, echoMyVarible, getMyVarible
   }

});

Modules.register('ModuleFun2', ['ModuleFun1'], function(ModuleFun1) {
  function setMyVarible(someString) {
    ModuleFun1.setMyVarible(someString);
  }
  function echoMyVaribleUppercase() {
    console.log('my varible with uppercase: ' + ModuleFun1.getMyVarible().toUpperCase());
  }
  return {
    setMyVarible, echoMyVaribleUppercase
  }
});

let moduleFun2 = Modules.getModule('ModuleFun2');
moduleFun2.setMyVarible('hello world');
moduleFun2.echoMyVaribleUppercase(); // HELLO WORLD
```

这个实例里面，是一个很典型的暴露器模式，将函数引用返回出来来达到读取作用域内部的变量。myModules函数里面实现了一个register方法注册moduleFun保存到modules变量里面。在register函数里面读取deps数组声明的依赖选项，从已经注册的modules读出传递给模块函数使用从而实现了依赖加载。