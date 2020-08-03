---
title: ElementUI基于TypeScript的二次封装小记
tag: frontend
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
```javascript
formatter: Function(row, column, cellValue, index)
```
通过这个属性可以在表格cell中自定义渲染的内容，然而如果想返回一段html，或者一个组件，那么会被以字符串的形式原样输出：
```javascript
formatter: (row, column, cellValue, index) {
   return '<p style="color:red;">一些文字' + row.someCol + '</p>';
}
```
解决的方法是使用createElement，然而`this.$createElement(tag, {attrs: style: {color: 'red'}, row.someCol})`这种写法实在不友好，这个使用可以使用jsx， `vuejs/babel-plugin-transform-vue-jsx`这个插件会自动将jsx通过正则编译成createElement函数。
```javascript
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

#### 表单组件的封装

我们的目的是用类似json配置化的格式来替代原先`el-form`与`el-form-item`互相嵌套定义表单。在Vue的表单中，表单最主要的元素即为Form配置以及Form绑定的数据，所以一个动态表单的组件，它的使用方法可以表达为：

```html
 <dynamic-form ref="dy_form" formState={formState} formModel={formModel} />
```

在组件内部，`formState`如何映射到原先template定义的表单项组件上呢？在`dynamic-form`组件中，定义一个`render`

```javascript
  <el-form ref="dy_form" {...dynamicProps}>
    {
      this.formState.map((row) => {
        return (
          <el-row class="row-bg">
            { row.map(formItem => {
              return (
                <el-col span={formItem.colSpan}>
                  { this.getDynamicComponent(formItem) }
                </el-col>
              );
              })
            }
          </el-row>
        );
      })
    }
  </el-form>
```

formState中，每个表单项formItem的结构可以定义为

```typescript
export interface FormItem {
  formLabel: string,
  modelKey: string,
  formRules?: string | Array<ValidateRule> | null,
  inputProps?: { [key: string]: any },
  inputAttrs?: { [key: string]: any },
  inputEvents?: { [key: string]: any },
  tagName: string,
  tagOption?: Array<ElOptionTag>,
  className?: string
}
```

`tagOption`中`ElOptionTag`对应的是一个`label`，`value`组成的对象，在表单进行渲染的时候会渲染成对应的组件（如`el-select`下拉组件中`el-option`组件用来表示下拉每个选项）

FormState传入表单配置的实例：
```javascript
const formState1Col: FormState = [
  [
    {
      formLabel: '输入框',
      tagName: 'el-input',
      modelKey: 'inputText1',
      inputProps: {
        clearable: true,
        placeholder: '跟进天数'
      },
      inputEvents: {},
      formRules: 'required'
    },
    {
      formLabel: '输入框2',
      tagName: 'el-input',
      modelKey: 'inputText2',
      formRules: 'required',
      tagOption: [
        {
        label: '选项1',
        value: 1
        },
        {
        label: '选项2',
        value: 2
        }
      ]
    }
  ]
];

```

FormState的结构即为由FormItem对象组成的数组。在render渲染函数中可以看到`formState`先map到`row`再map到每个表单项上进行渲染，所以`FormState`是一个二维`FormItem`数组。这样就能以行-列的格式把一个`el-form`渲染出来了。

在render函数中，`getDynamicComponent`就是负责生成每个表单项配置对应实际渲染的组件`el-form-item`，返回一个配置好的`VNode`:

```typescript
getDynamicComponent(formItem: FormItem) {
  const dyComp: DynamicCompTag = getDyCompConfig(formItem); 
 return (
    <el-form-item label={ formItem.formLabel } class={['dy-form-item', formItem.      className]} prop={formItem.modelKey}>
        <dynamic-component
          key={formItem.modelKey}
          comp={ dyComp }
        />
    </el-form-item>
    );
}
```

`getDyCompConfig`将formItem转换为一个`$createElement`函数接受传入的options:

```typescript
getDyCompConfig() {
  // ...这里先省略后续做的一些默认值适配的过程
  return {
      tagName: formItem.tagName,
      options: {
        props: { ...formItem.inputProps },
        attrs: { ...formItem.inputAttrs },
        on: { ...formItem.inputEvents }
      },
      children: mapTagOptions(formItem.tagOption) // 这个函数会将传入的ElOptionTag数组对象映射成对应传入的slot组件（如el-option）的DynamicCompTag配置对象
    };
}
```

`dynamic-component`组件内部将`tagName`和`options`通过`$createElemnt`渲染到VNode上进行挂载：

```typescript

export interface DynamicCompTag {
  tagName: string
  options?: { [key: string]: any }, // 对应$createElement里面的options（attrs, on, props等）
  children?: string | Array<DynamicCompTag> | VNodesRender
}

export type VNodesRender = (h: CreateElement) => Array<VNode>;
export type VNodeRender = (h: CreateElement) => VNode;


@Component
export default class DynamicComponent extends Vue {
  @Prop({ type: Object, default: () => {} }) comp!: any;

  _createComponent(tagName: string,
                  options: VNodeData = {},
                  textOrChildren?: Array<DynamicCompTag> | VNodesRender | string): VNode {
    // children可以是DynamicCompTag数组对象或者VNodes实例
    if (textOrChildren instanceof Array) {
      // 递归创建子节点
      const childrenVNodes = (textOrChildren as Array<DynamicCompTag>).map((child: DynamicCompTag): VNode => {
        return this._createComponent(child.tagName, child.options, child.children);
      });
      return this.$createElement(tagName, options, childrenVNodes);
    }
    // 此时textOrChildren为文字
    return this.$createElement(tagName, options, textOrChildren);
  }

  render(h: CreateElement) {
    return this._createComponent(this.comp.tagName, this.comp.options, this.comp.children);
  }
}

```

#### 动态表格组件

动态表格组件的思路和动态表单大同小异，都是传入配置最后映射成`el-table`与`el-table-column`相结合的模板配置：


官方模板配置：
```html
<el-table
  :data="tableData"
  style="width: 100%">
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
  <el-table-column
      fixed="right"
      label="操作"
      width="100">
      <template slot-scope="scope">
        <el-button type="text" size="small">编辑</el-button>
      </template>
    </el-table-column>
</el-table>
```

自定义的配置对象格式写法：
```javascript
const tableColumnConf: Array<TableColumn>= [
  {
    label: '日期',
    dataKey: 'date',
    props: {
      width: 180
    },
  },
  {
    label: '姓名',
    dataKey: 'name',
    props: {
      width: 180
    },
  },
  {
    label: '地址',
    dataKey: 'address'
  },
  {
    label: '操作',
    scopeRender: (scope: CellScope, h: CreateElement) => {
      return (
        <el-button>编辑</el-button>
      )
    }
  }
]
```

DynamicTable组件使用的方式：
```html
<dynamic-table column={tableColumn} data={tableData} />
```

实现，使用map将配置映射到`el-table-column`组件上面就可以了
```typescript
import { Prop, Vue, Component } from 'vue-property-decorator';
import { CreateElement, VNode } from 'vue';
import { ElTable } from 'element-ui/types/table';

type FormatterFn = (value: any, vm: Vue) => any

type ScopeRenderSlot = (h: CreateElement) => {[key: string]: (scope: CellScope) => VNode | string}

type ScopeRenderDefault = (scope: CellScope, h: CreateElement) => VNode | string

export interface TableColumn {
  label: string, // column标签名称
  dataKey: string, // 对应数据源的key
  props?: { [key: string]: any },
  eventOn?: { [key: string]: Function }, // 单元格事件
  scopeRender?: ScopeRenderSlot | ScopeRenderDefault, // 单元格scope slot，用法参考element ui
  formatter?: Array<string | FormatterFn>
}

export interface CellScope {
  column: {[key: string]: any},
  row: {[key: string]: any},
  $index: any
}

@Component
export default class DynamicTable extends Vue {
  @Prop({ type: Array, default: () => [] }) column!: Array<TableColumn>; // 表格列配置
  @Prop({ type: Array, default: () => [] }) data!: Array<any>; // 表格数据
  @Prop({ type: Object, default: () => { return {}; } }) tableProps!: { [key: string]: any }; // el-table的自定义props
  @Prop({ type: Object, default: () => { return {}; } }) tableEvents!: { [key: string]: Function }; // 对应触发current-change事件

  static formatters: {[key: string]: FormatterFn} = {};

  get columns() {
    return this.column.map(columnItem => {
      const item = { ...columnItem };
      // 表格的默认值设定
      const props = {
        prop: item.dataKey,
        label: item.label,
        align: 'left', // 默认左对齐
        formatter: (row) => {
          const data = this.$utils.objStrGet(row, item.dataKey);
          if (item.formatter && item.formatter.length) {
            return this.formatterPipeline(item.formatter, data);
          }
          // 无数据显示横杠
          return this.$utils.emptyableValue(data);
        }
      };
      item.props = Object.assign(props, item.props); // 自定义设定覆盖默认的表格设定
      return item;
    });
  }

  get tableOption() {
    return {
      props: {
        data: this.data,
        class: 'table',
        style: 'width: 100%',
        stripe: true,
        ...this.tableProps // 覆盖默认el-tables设定
      },
      on: this.tableEvents
    };
  }

  // 直接获取ElTable
  get elTableRef() {
    return this.$refs.multipleTable as ElTable;
  }

  render(h: CreateElement) {
    return (
      <el-row type="flex"
        class="comp-dy-table"
        justify="start"
      >
        <el-table {...this.tableOption} ref="multipleTable">
          {
            this.columns.map((columnItem, index) => {
              const props = {
                key: `cus-tab-${index}`, // 需要设置key，不然vue的原地复用策略可能会导致渲染出错
                fit: true,
                ...columnItem.props
              };
              const columnOption = {
                props,
                on: columnItem.eventOn ? columnItem.eventOn : {}
              };
              columnOption['scopedSlots'] = {
                // 对应template里的slot-scope，自定义列渲染
                default: (scope) => {
                  return columnItem.scopeRender!(scope, this.$createElement);
                }
              };
              return (
                <el-table-column {...columnOption} />
              );
            })
          }
        </el-table>
      </el-row>
    );
  }
}
```

后记：

这个库在项目里主要还是利用`class`来实现方法的封装，实际上后期使用碰到很多问题，主要有：

* 表单联动支持不够友好，联动逻辑需要与组件进行绑定
* 在输入事件上无法灵活处理
* 没有利用好TS的类型推导，很多地方不能自动跳出类型和不全

这个库在后期重新用`Composition Api`重新进行了设计，可参考https://blog.marisa6.cn/2020/07/16/Vue%20Composition%20Api%E5%B0%8F%E7%BB%93/ ，虽然这个坑还没填完（ElementUI不再维护了，想把这一套api从ElementUI中脱离出来）