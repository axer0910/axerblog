---
title: Vue Composition Api使用小结
tag: 源码分析
date: 2020-07-16
updated: 2020-07-16
---

Vue3.0 beta发布也有一段时间了，当时在阅读了Vue3.0响应式部分的源代码过后，觉得这种类似React Hook的设计的确是可以解决一些平时开发上遇到的一些痛点的，简单看一下Composition Api主要特点：

* 函数式的Api，以及更好的TypeScript类型推导支持
* 脱离组件生命周期，可以存在于单独的文件中
* 更好的逻辑组合以及复用

先来简单看一下使用这个Api的Demo：

```javascript
const { ref, onMounted, onUnmounted, createApp } = Vue;

  /*
  实时返回鼠标位置的函数
   */
  function useMousePosition() {
    // ref的作用：将一个基本类型对象定义为响应式对象
    const x = ref(0) // 创建响应式变量x与y并初始化为0
    const y = ref(0)

    function update(e) {
      x.value = e.pageX // 修改一个ref创建的对象的值直接修改他的value
      y.value = e.pageY
    }

    onMounted(() => {
      window.addEventListener('mousemove', update) // 挂载组件时将鼠标移动事件与update方法进行绑定
    })

    onUnmounted(() => {
      window.removeEventListener('mousemove', update)
    })

    return { x, y }
  }

  createApp({
    setup() {
      // 组件初始化阶段setup钩子，通过工厂函数引入响应式变量的定义
      const isShowMousePos = ref(false)

      function enableMousePosition() {
        isShowMousePos.value = true
      }

      const { x, y } = useMousePosition() // 一行代码导入x和y两个响应式坐标值，这两个值将随着鼠标位置的改变实时进行变化
      return {
        x, y, enableMousePosition, isShowMousePos
      }
    }
  }).mount('#app')
```

模板：
```html
<div id="app">
    <button @click="enableMousePosition">点我实时显示鼠标位置</button>
    <p v-if="isShowMousePos">鼠标x: {{x}} 鼠标: {{y}}</p>
</div>
```
在这个实时显示鼠标位置的Demo中，使用了`useMousePosition`这个工厂函数，定义了`X，Y`这两个可响应的变量，并且在`onMounted`这个钩子中将鼠标移动事件与响应式变量的更新关联上。组件中只要模板订阅了这两个响应式变量，即可得到鼠标位置移动时实时位置的更新展示。

`useMousePosition`从形式上来看只是一个普通函数，这个普通函数可以被定义在项目源码中任意一个位置，不再依赖组件中`computed`, `mounted`, `data`等生命周期以及选项函数的依赖。这意味着这个工厂函数定义的逻辑可以在任意组件中使用。试想一下在传统的`Options api`风格中，要定义这样的逻辑，不论是Mixin还是HOC（高阶组件，可以理解为组件套组件）还是scoped slot这种方式，至少需要额外定义一个组件，从而产生额外的代码，性能开销等。而且在`Options Api`里，生命周期一定程度上可以说是“绑架”了代码编写的逻辑。在使用了函数式api的这种方式下，各个逻辑可以被拆散到“函数”的唯独下，不再与生命周期的钩子强绑定，再利用工厂函数的这种方式通过闭包快捷定义一些初始化的参数，可以最大程度灵活组织代码逻辑。


#### 使用细节

在Vue中，整个框架最核心的一点其实是**响应式**，整个框架模板的渲染展示都是围绕着变量，react和vue的模板渲染都是围绕这一点，一种数据状态对应一种模板渲染的状态。react和vue的区别在于，react的数据更新都是需要通过一个更新函数主动push去更新，产生一个新的数据状态以及相对应的渲染状态。vue的话修改一个变量后，框架检测到这个变量的修改后，模板中找到引用依赖这个变量的地方再对这一块的渲染实施更新。

围绕着可响应这一点，先从如何定义响应式变量开始。

##### reactive

```javascript
import { reactive } from 'vue'

// reactive state
const state = reactive({
  count: 0
})
```
模板：
```html
<template>
  <p>count is {{ state.count }}</p>
</template>
```
使用`reactive`定义了一个`state`，`state`中包含了属性`count`。当设置`state.count = 1`，模板也会相应进行更新。更新state也可以说是产生了一个副作用（side effect）。在composition api中也有观察这种副作用的api。

##### ref



#### 封装一些组件的思路（rollupjs打包）

#### 代替vuex

,



