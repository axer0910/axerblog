---
title: ElementUI基于TypeScript的二次封装部分小结
tag: php
date: 2019-12-06
updated: 2019-12-06
---

背景：最近的项目基于Element UI搭建后台数据管理系统。在项目中出现的展示主体最多的为搜索筛选框+表格+分页和表单提交数据。写过ElementUI的同学知道如果需要调用表格和表单基本上是下面这种写法：

表格：
```html
<el-table :data="tableData" style="width: 100%">
	<el-table-column
        prop="date"
        label="日期"
        width="180">
      </el-table-column>
      <el-table-column
        prop="name"
        label="姓名"
        width="180">
      </el-table-column>
      <el-table-column
        prop="address"
        label="地址">
      </el-table-column>
</el-table>
```
表单：
```html
<el-form ref="form" :model="form" label-width="80px">
  <el-form-item label="活动名称">
    <el-input v-model="form.name"></el-input>
  </el-form-item>
  <el-form-item label="活动区域">
    <el-select v-model="form.region" placeholder="请选择活动区域">
      <el-option label="区域一" value="shanghai"></el-option>
      <el-option label="区域二" value="beijing"></el-option>
    </el-select>
  </el-form-item>
  <el-form-item label="活动时间">
    <el-col :span="11">
      <el-time-picker placeholder="选择时间" v-model="form.date2" style="width: 100%;"></el-time-picker>
    </el-col>
  </el-form-item>
  <el-form-item label="即时配送">
    <el-switch v-model="form.delivery"></el-switch>
  </el-form-item>
  ...
</el-form>
```
这样写难免有些冗余，要不停地手动重复各种el-xxx组件来定义表单或者表格列。冗余是一个问题，操作表单也特别麻烦，需要根据条件在模板中写v-if控制组件的渲染。表单中如果需要根据条件控制的逻辑复杂，或者表格中有很多展示逻辑，难免会造成模板维护不便，于是就花了点时间对他进行了二次封装。

#### 使用TypeScript
由于是做二次封装，方法一般都是定义组件和各类对象格式。如果使用了TypeScript来定义对象格式和接口，就可以对传入的对象格式进行约束，一方面是增加了代码可读性，另一方面可以增加代码健壮性和扩展性，尤其在TS中利用接口和抽象类的特性可以方便地使用常用的面向对象设计模式写出扩展性更高的代码。

<!-- more -->

#### Vue中使用TypeScript
现在Vue使用TS的例子已经很多了，ElementUI本身也有d.ts文件对TS进行支持。由于现在Vue3.0现在整体来说仍然处于预览阶段，加上项目本身是8月份启动，用vue-cli 3.0搭建的，暂时不考虑迁移到3.0上面。如果使用了vue-cli 搭建的项目，要想在项目中引入TypeScript，只需要一行命令：
```
vue add typescript
```
这会自动添加并安装和TS有关的依赖。[相关文档](https://github.com/vuejs/vue-cli/tree/dev/packages/%40vue/cli-plugin-typescript)

为了更好地使用TS的特性，插件引入了`vue-class-component`这个库。这个库官方出品对TS的支持，使用了TS中的装饰器特性对组件内的各种hook进行了支持。典型的TS组件这么写：
```javascript
<template>
  <div>
    <input v-model="msg">
    <p>prop: {{propMessage}}</p>
    <p>msg: {{msg}}</p>
    <p>helloMsg: {{helloMsg}}</p>
    <p>computed msg: {{computedMsg}}</p>
    <button @click="greet">Greet</button>
  </div>
</template>

<script>
import Vue from 'vue'
import Component from 'vue-class-component'

Component
export default class App extends Vue {
  // 初始值，相当于定义在data()中的值
  msg = 123

  // 相当于定义prop，把helloMsg "装饰为"一个prop 
  @Prop helloMsg = 'Hello, ' + this.propMessage

  // 和mounted声明周期使用方法一样
  mounted () {
    this.greet()
  }

  // 使用get作为计算属性
  get computedMsg () {
    return 'computed ' + this.msg
  }

  // 普通methods定义
  greet () {
    alert('greeting: ' + this.msg)
  }
}
</script>
```
主要通过装饰器，get，class的属性值对vue自带的声明周期和属性定义进行支持。

#### 使用JSX
大家都知道vue的模板到最后都会编译成虚拟DOM，也就是一个个由VNode组成的对象。相比写vue模板语法最后编译成vnode结构，jsx可以带来更加灵活的渲染视图方法，直接使用正则编译成createElement函数创建VNode对象。相比模板语法，这就意味着有更加灵活的方法去进行OO（面向对象）的编写或者函数式组件的编写，也可以拥有更灵活的方法控制props的传递。
在element ui里，表格列配置有这么一个属性：
```
formatter: Function(row, column, cellValue, index)
```
通过这个属性可以在表格cell中自定义渲染的内容，然而如果想返回一段html，或者一个组件，那么会被以字符串的形式原样输出：
```javascript
formatter: (row, column, cellValue, index) {
   return '<p style="color:red;">一些文字' + row.someCol + '</p>';
}
```
解决的方法是使用createElement，然而`this.$createElement(tag, {attrs: style: {color: 'red'}, row.someCol})`这种写法实在不友好，这个使用可以使用jsx， `vuejs/babel-plugin-transform-vue-jsx`这个插件会自动将jsx通过正则编译成createElement函数。
```
formatter: (row, column, cellValue, index) {
   return (<p style="color:red;">一些文字{row.someCol}</p>);
}
```
所以在思考封装结构的时候，直接使用了jsx语法生成虚拟dom和在组件中传递，可以直接在定义配置结构中传入jsx：

```javascript
{
    labelName: '测试单选组件',
    tagName: 'el-radio-group', // 组件标签名称
    modelName: 'radioValue', // 绑定的数据字段key
    props: {},
    options: (h: CreateElement): Array<VNode> => {
      // 使用jsx
      return [
        (<el-radio label="radio选项1" />),
        (<el-radio label="radio选项2" />),
        (<el-radio label="radio选项3" />)
      ];
    }
  }
```
在vue组件中，使用render函数替代<template>标签：
```javascript
render(h: CreateElement) {
	return (<my-comp someProp= onClick={this.myMethod} />)
}
```
注意在ts中写jsx文件名要保存为tsx。
相比react，由于vue2.x版本源码中很多地方使用了动态构建对象的方式，相对于react，缺少对prop的类型检测支持，暂时没有办法像react那样组件有类型提示和检测报错。同时还有一些坑，由于vue的createElement函数不像react所有的props都是顶级属性，vue的createElement为props区分了`on attrs props`三个类型，分别对应事件，dom属性，和组件动态属性props。直接在jsx上面写属性，babel插件会通过正则进行分类，然而这个分类有时候会出问题，说一个实际碰到的问题：

```javascript
<el-form ref="dy_form" model={this.localModel} rules={this.formRules}>
	...          
</el-form>
```

运行的时候，发现model对应的对象并没有传进el-form当中，在vue调试工具中发现model里面被分到了attrs属性中去了。最后定义了一个对象：

```javascript
const dynamicProps = {
   props: {
     rules: this.formRules,
     model: this.localModel
   }
};
return (
<el-form ref="dy_form" {...dynamicProps}>
	...          
</el-form>
);
```
显示手动进行分类。相比react来说的确不是很优雅。
如果组件里定义事件最好不要以on开头，如果组件里定义了自定义事件onChange，写jsx的时候得这么写：

```html
<my-comp onOnChange={this.someMetod} />
```
要写两次on，一次小写一次大写，因为正则匹配把on开头的名称把小写开头on去掉，并且将剩余的名称转为小驼峰在放进on对象里面。为了避免这种奇怪的写法定义事件的时候不要以on开头。

踩坑的时候参考了[在Vue中使用JSX的正确姿势](https://zhuanlan.zhihu.com/p/37920151)。

todo未完待续..

后记：

这个库在项目里主要还是利用`class`来实现方法的封装，实际上后期使用碰到很多问题，主要有：

* 表单联动支持不够友好
* 在输入事件上无法灵活处理
* 没有利用好TS的类型推导，很多地方不能自动跳出类型和不全
* 表单组件过于配置化，有时候表单写模板更加符合直觉（当然配置文件更有利于做配置变换）
* 设计略显冗余，用了函数方法去更新状态而不是用函数（当时同时在研究flutter的设计影响了，实际上vue就应该直接操作数据，对其他同事来讲这样更利于理解）

这个库在后期重新用`Composition Api`重新进行了设计，可参考https://blog.marisa6.cn/2020/07/16/Vue%20Composition%20Api%E5%B0%8F%E7%BB%93/ ，虽然这个坑还没填完（ElementUI不再维护了，想把这一套api从ElementUI中脱离出来）