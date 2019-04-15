---
title: PHP反射常用方法
tag: 
---

 通过反射方法可以动态取得类名，方法列表，构造函数等和类的许多有关信息。

反射类构造：
```
$class = new \ReflectionClass('SomeNamespace\\AClass');
```
获取类方法列表
```
$class->getMethods();
```

反射类常用方法
```
$class->isInterface() // 是否为接口类
$class->isAbstract() // 是否为抽象类
```
反射类取得构造函数：
```
$constructor = $class->getConstructor(); // 获取一个constructor对象
$constructor->getParameters(); // 获取构造参数列表
```
实例一个类
```
$args = ['some arg', 123]
$newObj = $class->newInstanceArgs($args); // 通过一个数组传入构造函数实例一个对象。
```

如何获取当前所有类名？
使用 `get_declared_classes()` 返回一个class包含命名空间的完整class列表数组

获取类的命名空间：
```
$class->getNameSpaceName()
```
