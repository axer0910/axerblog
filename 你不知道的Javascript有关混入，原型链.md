---
title: 你不知道的Javascript有关混入，原型链
tag: frontend
date: 2019-01-13
updated: 2019-01-13
---

#### 混入
在js中实现继承或者实例化的时候，并没有类似于传统的面向对象语言自动执行**复制**的行为。js中只有对象。对象和对象之间都是通过原型链关联起来。

js中，如果要实现继承，可以自己实现一个方法，复制一个对象到另外一个对象，这个过程可以成为`mixin(..)` （混入）

##### 显式混入：
```javascript
// 非常简单的mixin(..) 例子:
function mixin(sourceObj, targetObj) {
    for (var key in sourceObj) {
        // 跳过目标对象已经存在的key
        if (!(key in targetObj)) {
            targetObj[key] = sourceObj[key];
        }
    }
    return targetObj;
}
var Vehicle = {
    engines: 1,
    ignition: function () {
        console.log("Turning on my engine.");
    },
    drive: function () {
        this.ignition();
        console.log("Steering and moving forward!");
    }
};
var Car = mixin(Vehicle, {
    wheels: 4,
    drive: function () {
        Vehicle.drive.call(this); // 调用一下“父类”，需要把Car的上下文给到“父类”的drive方法，才能取得Car里的wheels属性。
        console.log(
            "Rolling on all " + this.wheels + " wheels!"
        );
    }
});
```
Car对象"继承了"Vehicle对象。在Car对象里的`Vehicle.drive.call(this);`这句话就是显式多态。JS里并没有多态的机制，必须通过call方法显示调用指定对象的方法（这里是Vehicle里的drive方法）。 

<!-- more -->

隐式混入：
```javascript
var Something = {
    cool: function () {
        this.greeting = "Hello World";
        this.count = this.count ? this.count + 1 : 1;
    }
};
Something.cool();
Something.greeting; // "Hello World"
Something.count; // 1
var Another = {
    cool: function () {
        // 隐式把Something 混入Another
        Something.cool.call(this);
    }
};
Another.cool();
Another.greeting; // "Hello World"
Another.count; // 1 （count 不是共享状态）
```
通过在构造函数调用或者方法调用中使用Something.cool.call(this)，我们实际上“借用”了函数Something.cool()并在Another的上下文中调用了它（通过this绑定）。最终的结果是Something.cool()中的赋值操作都会应用在Another对象上而不是Something对象上。虽然借用了this的重新绑定功能，但是但是Something.cool.call(this)仍然无法变成相对（并且更灵活的）引用，因此不推荐使用这类方法。

#### 原型
JS中的时候new 一个对象的时候，里面关键一部是把新对象的原型链连接到函数的prototype属性上。JS中在对象上查找一个属性的时候，如果这个属性不在当前对象里，引擎会尝试在该对象的[[Prototype]]里面查找是否包含该属性。**这样做的好处是让n个对象可以共享一系列属性和方法。**

```javascript
// 假设有多个对象，都需要拥有foo方法，bar方法

let commonFuns = {
    foo: function () {
      console.log('this is foo function');
    },
    bar: function () {
      console.log('this is bar function');
    }
  };

// 创建两个对象，他们的原型关联到commonFuns
let obj1 = Object.create(commonFuns);
obj1.obj1Fun = function () {
  console.log('this is obj1Fun');
};`	
let obj2 = Object.create(commonFuns);
obj2.obj2Fun = function () {
  console.log('this is obj2Fun');
};

// 不需要复制commonFuns里的foo方法和bar方法分别到obj1和obj2，只需使用Object.create(commonFuns) 定义它们的原型即可。

obj1.foo(); // 'this is foo function'
obj2.bar(); // 'this is bar function'
```
#### 属性屏蔽

如果一个属性名即存在于某个对象的[[Prototype]]链上也存在于对象本身上，那么就会发生屏蔽。对象本身的属性将会屏蔽它原型链上的同名属性。在取对象的某个属性的时候总会取原型链当中最底层的那个。
```javascript
let prototype = {
    a: 111
  };

let obj = Object.create(prototype);
obj.a = 222;
console.log(obj.a); // 222 屏蔽了原型链上的a: 111
```
如果原型链上定义了只读的属性（writable: false），那么没办法修改已有属性，并且在对象上再创建一个同名属性会静默失败。
```javascript
let prototype = {};

Object.defineProperty(prototype, 'a', {
  writable: false,
  value: 111
});

let obj = Object.create(prototype);
obj.a = 222;
console.log(obj.a); // 111，原型链上的a属性被定义为只读，这时候obj上再次定义a将会静默失败
```
如果原型链上的某个属性定义了setter，那么在对象上再创建一个同名属性一定会调用这个setter(而不是创建一个新属性覆盖原型链上的属性)。
```javascript
let prototype = {
    b: 0
  };

Object.defineProperty(prototype, 'a', {
  set(val) {
    this.b = val * 2;
  },
  get() {
    return this.b;
  }
});

let obj = Object.create(prototype);
obj.a = 222; // a不会再次创建而是调用原型链上的a（setter）
console.log(obj.a); // 输出444，调用a的getter
console.log(obj.b); // 输出444， 通过a setter属性设置
```
#### JS里的类和继承

在Javascript里面，类无法描述对象的行为（没有类这种设计），所以需要对象自己定义自己的行为。**JS里只有对象**。
要模拟类的行为，可以利用函数的一个特殊属性叫做prototype。**所有的函数都拥有prototype属性**，它会指向一个对象。
```javascript
function Foo(){
	//....
}

var a = new Foo();
Object.getPrototypeOf(a) === Foo.prototype; // true
```
使用new操作符，创建JS对象主要四步骤
* 创建JS对象主要四步骤
* **新对象的原型链连接到函数的prototype**
* 执行构造函数
* 返回新创建的对象

上面的代码里，通过new操作符生成了新对象a以后，a对象的[[Prototype]]链就指向了Foo的prototype属性，也就是说这个时候a的原型链代理了所有Foo.prototype所有的方法，**访问a原型链中的方法将会委托到Foo.prototype上**。

>*委托*即是javascript中对象的关联机制
>在JavaScript中，我们并不会将一个对象（“类”）复制到另一个对象（“实例”），只是将它们关联起来。（原型继承）
>这样一个对象就可以通过委托访问另一个对象的属性和函数

**构造函数**

```javascript
function Foo(){
	//....
}

Foo.prototype.constructor === Foo; //true

var a = new Foo();
a.constructor === Foo; // true
```
JS函数protoype属性里的constructor实际上就是指向函数本身。上面这段代码里Foo显然不是什么特殊的构造函数，但是使用new关键符调用Foo的时候，调用Foo的时候会使它变成一个构造函数调用(此时函数里的this指向新对象)。

上面代码里**a.constructor只是通过默认的[[Prototype]]委托指向Foo**，a本身没有constructor属性。Foo.prototype里面有constructor属性。
```javascript
function foo() {
 console.log(this);
}
let f = new foo();
console.log(f.constructor === foo.prototype.constructor);
```
要注意，.constructor是可以被修改的。如果一个函数的prototype不是指向默认的prototype对象而是指向别的对象（可能是其他函数的prototype），那么实例化以后对象中的consturctro并不会指向生成它的函数。
```javascript
function Foo(){ /*...*/ }

Foo.prototype = { /*...*/ }; //创建一个新原型对象(不包含constructor属性)

var a1 = new Foo();

a1.constructor === Foo; // false

a1.constructor === Object; // true
```
a1对象在Foo的prototype里面找不到consutructor，那么它会继续查找prototype的原型也就是Object的consturctor。那么a1此时的constructor并不是指向生成它的函数Foo。所以在对象里的constructor引用通常不是很可靠和安全，需要尽量避免。

**原型继承**
既然JS没有复制实现继承，但是可以通过关联原型使用委托来实现“继承”。
```javascript
 // Bar作为Foo子类，拥有Foo类所有属性和功能，并且拥有子类的方法。
    function Foo(name) {
      this.name = name;
    }
    Foo.prototype.outputName = function () {
      console.log('outputName：', this.name);
    };
    
    /*
    Bar“继承”Foo，可以再设置一个属性label
     */
    function Bar(name, labelName) {
      // 调用Foo构造，将Foo构造里的this替换为Bar构造的this。
      Foo.call(this, name);
      this.labelName = labelName;
    }
    Bar.prototype = Object.create(Foo.prototype); // Bar的prototype原型链设置为Foo的prototype，使得Bar可以找到Foo的方法。

    Bar.prototype.outputLabel = function () {
      console.log('outputLabel：', this.labelName);
    };
    

    let bar = new Bar('my test name', 'my label');
    bar.outputName(); // my test name
    bar.outputLabel(); // my label
```
**检查类关系**
如果有对象a，怎么样才能寻找a委托的对象（如果存在）呢？如何判断一个对象的原型链和某个函数的prototype是否有关联呢？可以使用instanceof
```javascript
function Foo(){
	//....
}

Foo.prototype.x = ....;

var a = new Foo();

a instanceof Foo; //true

// instanceof回答的问题是：在a的整条 [[Prototype]] 链中是否有指向 Foo.prototype 的对象？

// 处理对象(a)和函数(带 .prototype 引用的Foo)之间的关系

// 如果想判断两个对象（比如a和b）之间是否通过 [[Prototype]] 链关联，instanceof 无法实现
```

