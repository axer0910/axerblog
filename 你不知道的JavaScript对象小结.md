---
title: 你不知道的JavaScript对象小结
tag: frontend
date: 2019-01-06
updated: 2019-01-06
---


js中有这么几种主要类型：
* string
* number
* boolean
* null
* undefined
* object
* symbol（es6新增）

>使用typeof null测试null类型对象会返回object类型，实际上这是语言的bug，null的类型就是null。

### 内置对象
JavaScript里面还有一些对象子类型，通常叫做内置对象，有些内置对象和基本类型看起来名字差不多（和Java有点像），但实际上他们其实是一些内置函数，通过函数构造一个（new操作符）对应子类的新类型对象来使用。

```javascript
var testStr = "hello world";
console.log(typeof testStr); // string
var anotherStr = new String('new hello world');
console.log(typeof anotherStr); // object
console.log(anotherStr instanceof String); // true
console.log(testStr instanceof String); // false
console.log(testStr.length); // 输出11，自动把testStr转换成String对象，String对象里面有length属性
```
testStr本身是个字面量，为了使用length属性获取字符串长度，引擎自动将其转成了String对象来获取length长度。

同样的也会发生在数值字面量上面，如果使用42.233.toFixed(2)这样的方法，会自动调用`new Number(42.233)` 随后输出对象的toFixed(2)的值。
### 对象内容

在对象中，属性名永远都是**字符串**（感觉类似PHP关联数组）。如果使用string（字面量）以外的其他值作为属性名，那么它首先会被转换成一个字符串。即使数字也不例外，虽然数组里面使用下标的确是数字，当时在对象里面使用数字是会被转换成字符串的。

```javascript
let arr = [];
arr[3] = 'bar';
console.log(arr[3]); // bar
console.log(arr['3']); // bar
console.log(arr instanceof Array); // true
console.log(arr instanceof Object); // true

let obj = {};
obj.myKey = 'hello';
console.log(obj['myKey']); // hello
obj[String] = 'test string'; // 尝试使用一个对象当作键
console.log(obj[String]); // test string 
console.log(String.toString()); // function String() { [native code] }
console.log(obj['function String() { [native code] }']); // 结果和obj[String]一样，会把String对象转换成字符串作为键再设置内容。
```
es6支持属性名计算，可以在[]里面放一个表达式：
```javascript
let perfix= 'my';
let key ='test';
let key2 = 'another';
let obj = {};
obj[perfix + key] = 'hello test';
obj[perfix + key2] = 'hello another';
console.log(obj[perfix + key]); // hello test
console.log(obj[perfix + key2]); // hello another
```
#### 属性描述符
从es5开始，所有的属性都具有一个属性描述符，来表述对象的某个属性是否可配置，可枚举，是否只读。
```javascript
let myObj = {
    a: 2
  };
console.log(Object.getOwnPropertyDescriptor(myObj, 'a'));
// configurable: true // 可配置
// enumerable: true // 可枚举
// value: 2
// writable: true // 可写
```
定义一个新属性的时候，除了直接在对象上使用.xxx创建，也可以使用Object.defineProperty方法来定义。
```javascript
let myObj2 = {};
 Object.defineProperty(myObj2, 'b', {
   value: '222',
   writable: false, // 不可写
   enumerable: false, // 不可枚举
   configurable: false // 不可配置
 })
```
这样在myObj2上面创建了一个不可写，不可枚举，不可配置的b属性。
```javascript
myObj2.b = '333';
console.log(myObj2.b); // 222
console.log(Object.keys(myObj2)); // 由于是不可枚举属性，取得所有对象所有键的时候不会被列举出来。输出[]。
delete myObj2.b; // 尝试删除b属性
console.log(myObj2.b); // 由于b属性的不可配置，b属性还在，输出222
```

尝试操作这些不可写，不可配置的属性的时候，js不会报错，而是忽略对这个属性进行的操作（静默失败）。
不可配置的属性上，如果使用defineProperty尝试重新设置属性也会静默失败。
如果设置了writable为false，那么没办法再改回true。
如果想在对象里面创建一个常量属性，可以在对象里面定义一个writable: false 和configurable: false的属性，这样这个属性就会不可删除，不可重定义，不可修改。

如果想让某个对象不可扩展，可以使用Object.preventExtensions(...)。
```javascript
let myObj3 = {};
myObj3.a = '123';
Object.preventExtensions(myObj3);
myObj3.b = '456';
console.log(myObj3.b); // undefined
```

还有一些方法比如密封：`Object.seal(...)`，会禁止目标对象属性的增加和配置，删除。
`Object.freeze(...)`：除了调用`Object.seal(...)`，还会将所有属性的writable设置false，真正的冻结一个对象的修改。

#### 存在性
`in` 检查对象里是否存在这个属性和它的原型链；
`Object.hasOwnProperty(...)` 只检查对象本身，不会去原型链找。
`Object.keys(obj)` 返回包含当前对象所有**可枚举**属性的键的数组。
`Object.getOwnPropertyNames(...)`返回包含当前对象所有属性的键的数组，无论是否可枚举。

#### 遍历
`for ... in ...` 遍历对象所有可枚举属性，包括原型链上面。
`Array.some(function)`：遍历数组传到回调函数里，一直运行到回调函数返回true。
 `Array.every(function)`遍历数组传到回调函数里，如果回调函数返回false停止遍历。
 `for..of..` 遍历数组和有迭代器的对象。
 
 实现一个对象迭代器：
```javascript
let obj = {
      a: 222,
      b: 333
    };
    // 注册一个obj迭代器
    Object.defineProperty(obj, Symbol.iterator, {
      enumerable: false,
      writable: false,
      configurable: true,
      value() {
        let i = 0;
        let currentObjKeys = Object.keys(obj);
        // 返回一个含有next属性的对象
        return {
          next: () => {
            // next函数返回一个对象，value是值,done是否完成，true继续迭代否则停止迭代
            // this对象就是当前要实现迭代器的对象
            return {
              value: this[currentObjKeys[i++]],
              done: i > currentObjKeys.length
            }
          }
        }
      }
    });

for (let item of obj) {
 console.log(item); // 222 333
}

// 手动遍历（其实就是手动直接调用定义的Symbol.iterator中的next方法）
let it = obj[Symbol.iterator]();
console.log(it.next()); // {value: 222, done: false}
console.log(it.next()); // {value: 333, done: false}
console.log(it.next()); // {value: undefined, done: true}
```