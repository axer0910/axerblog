---
title: Vue组件使用自定义v-model
tag: 源码分析
date: 2018-12-29
updated: 2018-12-29
---

v-model实际上是个语法糖，结合了props属性和emit事件

`<input type="text" v-model="somevalue">`
实际上和
`<input type="text" :value="somevalue" @input="somevalue = $event.target.value">`
相等

v-model通过把传进来的外部变量与元素的value相互绑定，当绑定的元素value变化时候通知修改外部变量从而达到了双向绑定。

用到自定义组件上的话：

>要让组件的 v-model 生效，它应该 (在 2.2.0+ 这是可配置的)：
>接受一个 value 属性
>在有新的值时触发 input 事件

包装一个自定义checkbox组件，
```javascript
<my-comp v-model="foo"></my-comp>

Vue.component('my-comp', {
  tempalte: `<input 
               type="checkbox"
               @change="$emit('input', $event.target.checked)"
               :checked="value"
             />`
  props: ['value'],
})
```



Vue2.2以后的版本可以自定义支持v-model的prop和emit名称：
```javascript
<my-comp v-model="foo"></my-comp>

Vue.component('my-comp', {
  tempalte: `<input 
               type="checkbox"
               @change="$emit('emit-vmodel', $event.target.checked)"
               :checked="checked"
             />`
  props: ['checked'],
  model: {
    prop: 'checked',
    event: 'emit-vmodel'
  },
})
```
