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

在Vue中，整个框架最核心的一点其实是**响应式**，整个框架模板的渲染展示都是围绕着变量，react和vue的模板渲染都是围绕这一点，一种数据状态对应一种模板渲染的状态。Compostion Api的灵感来自于React Hook，React和Vue有一个显著区别在于，React的数据更新都是需要通过一个更新函数主动push更新，产生一个新的数据状态以及相对应的渲染状态。Vue的话修改一个变量后，框架检测到这个变量的修改后，模板中找到引用依赖这个变量的地方再对这一块的渲染实施更新。所以在Api的设计上，React Hook的`useState`会多返回一个函数用来更新变量，Vue的Api则只需要直接修改变量就可达到响应更新的效果。

首先来了解一下定义响应式对象的Api：

##### reactive

```javascript
import { reactive } from 'vue'

// reactive state
const state = reactive({
  count: 0
})
```

使用`reactive`定义了一个可响应的`state`，`state`中包含了属性`count`。当对`state`的属性进行修改时，如`state.count = 1`，会通知模板在引用的地方进行更新。在Vue3.0中，实现这种响应式的机制实际上是使用了一个`Proxy`来对原始对象添加响应依赖包装后进行返回，在2.x中，实际上是调用了`Vue.observable()`在对象上每个属性设置了setter和getter。无论是`Proxy`还是`setter``getter`，都是为了保证在引用或修改属性的时候让Vue知道数据的改动，这样才能取对应地进行视图更新。

当更新了响应式对象，此时需要对DOM进行更新，或者运行其他watch这个变量的回调，这个过程可以叫做副作用（side effect），指的是对当前代码运行上下文环境外部的系统的交互或者修改。对vue而言，就是与不在vue运行过程中的那部分系统（比如DOM，或者用户自定义的代码）进行交互。在Composition Api中，`watch`和`watchEffect`可以观察产生的副作用。假设先抛开vue的模板系统，在修改上面`state`响应式对象后，使用`watchEffect`来对DOM进行更新：

```javascript
watchEffect(() => {
  document.body.innerHTML = `count is ${state.count}`
})
```

`watchEffect`接受了一个回调，在发生副作用行为的时候立即运行这个回调, `watch`函数与`watchEffect`类似，只不过是观察某一个特定的响应式属性，当这个属性产生副作用时运行回调（功能与Vue2.x一致）。


使用`reactive`定义了一个`state`，`state`中包含了属性`count`。当设置`state.count = 1`，模板也会相应进行更新。更新state也可以说是产生了一个副作用（side effect）。在composition api中也有观察这种副作用的api。

##### computed

在Vue2.x的Options Api里面，我们知道有一个选项叫`computed`，它的作用是依赖其他响应式变量并实时计算出经过转换后的值。在上个例子中，假设对响应式对象`state`要实时取得它的`count`属性乘以2的结果：

```javascript
import { reactive, computed } from 'vue'

const state = reactive({
  count: 0
})

const double = computed(() => state.count * 2)
```
这里的`double`应该是什么类型？`state.count * 2`之后的类型结果应该是数字，那么在经过`computed`返回的结果应该也是数字。如何在模板里使用`double`，并且观察它的变化呢？我们知道在模板里引用了一个对象或变量，实际上是告诉Vue被引用的对象或变量“订阅”了这个变量的变化，这个变量变化后需要让Vue重新渲染视图。如果`computed`返回的`double`类型是数字，我们知道JavaScript中原始类型（`string,number,boolean,null,undefined`）是不能给他们设置`getter`，`setter`或者`proxy`来“通知”Vue这些变量的变化关系，那怎么在模板里使用这个原始类型的值呢？既然在对象上可以设置`getter`或者`proxy`，那么只要返回一个对象，并且将它的属性放置在`value`属性里，给`value`属性设置监听器，就可以知道它的变更了。这个产生包装变量的函数叫做`ref`，它会返回一个响应式对象，里面只有一个`value`属性，并将传入的类型为原始类型的值设置到`value`上面。实际上，`computed`函数可以简化理解为类似下面代码的实现：

```javascript
function computed(getter) {
  const ref = ref(null)
  watchEffect(() => {
    ref.value = getter()
  })
  return ref
}
```

##### ref

除了在`computed`里面会返回一个经过`ref`包装后的对象，我们也可以直接使用`ref`函数：

```javascript
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

到这里可能有些读者会产生一个问题，我在模板里使用经过`ref`包装后的对象，需不需要在模板里指定引用它的value属性来取它的值呢？其实是不需要的，在模板里，Vue会自动对`Ref`包装变量进行解构，取它的`value`属性。

```javascript
import { ref, watchEffect } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}

const renderContext = {
  count,
  increment
}

watchEffect(() => {
  renderTemplate(
    `<button @click="increment">{{ count }}</button>`, // 不需要写成{{ count.value }} vue会自动引用它的value属性
    renderContext
  )
})
```

在上面代码里，使用了`count`Ref包装对象来代替第一个例子里`state`响应式对象里的`count`属性。`ref`方法提供了一种更加灵活的方式，不要求模板一定要使用某个响应式对象上的属性来形成订阅关系，只要变量最后经过`ref`来包装，就能有依赖追踪的效果。

##### 在组件里的使用

在有了`reactive`, `ref`这些方法，当需要实现一套响应式逻辑的时候，无需在组件代码里依靠`data()`，`computed()`，`watch()`这些选项实现逻辑，逻辑可以被放在任意一个文件，函数，类里面，只要在最后暴露给组件，模板引用的是响应式对象，就能动态实现数据更新后实时更新模板，提升了代码复用性，逻辑也可以被更合理的组织，不被组件代码所“绑架”。在Vue组件里，使用了`setup()`代替了之前所有生命周期以及各类选项。生命周期都被放在函数式api中指定，并在`setup()`函数里进行初始化：

```javascript
<template>
  <button @click="increment">
    Count is: {{ state.count }}, double is: {{ state.double }}
  </button>
</template>

<script>
import { reactive, computed } from 'vue'

export default {
  setup() {
    const state = reactive({
      count: 0,
      double: computed(() => state.count * 2)
    })

    function increment() {
      state.count++
    }

    return {
      state,
      increment
    }
  }
}
</script>
```

使用`onMounted`，`onCreated`, `onActivated`等函数实现组件生命周期钩子，组件将在每一个生命周期触发对应的回调：

```javascript
import { onMounted, onActivated } from 'vue'

export default {
  setup() {
    onCreated(() => {
      console.log('component is created!')
    })
    onMounted(() => {
      console.log('component is mounted!')
    })
    onActivated(() => {
      console.log('component is activated!')
    })
  }
}
```

#### 代替vuex

我们在Vue项目中使用`Vuex`，主要是解决不同层级的组件能够共享同一个数据状态，多个组件某个部分展示的状态来共同依赖同一个公共的状态。但是在`Vuex`中，需要遵循一个特定的结构，如定义`state`来定义数据状态结构，定义`mutations`来定义修改数据状态的逻辑，定义`actions`来定义异步修改数据状态的方法。在组件中如果要引用`vuex`中的数据状态，还需要通过`mapState`，`mapGetters`等函数去映射到组件中去。这个写法在项目中随着时间的推移难免会显得有些冗余。得益于Composition Api能将数据定义在任何地方，或许能够实现`vuex`本来所实现的变量提升的功能，不再使用那些冗余的Api。

Vuex强调的是一个**单一状态树**，这个状态树位于所有Vue组件之上。先参照官方文档实现一个简单的vuex结构：

```html
<div id="app">
    <my-counter></my-counter>
    <another-counter></another-counter>
    <button @click="add">increment</button>
</div>
```

```javascript
const store = new Vuex.Store({
    state: {
        count: 0
    },
    mutations: {
        ADD_COUNT(state) {
            state.count ++;
        }
    }
})
const MyCounter = {
  template: `<div>{{ count }}</div>`,
  computed: {
    count () {
      return store.state.count
    }
  }
}
const AnotherCounter = {
  template: `<div>multipled by 2：{{ count }}</div>`,
  computed: {
    count () {
      return store.state.count * 2
    }
  }
}
new Vue({
    el: '#app',
    store,
    components: { MyCounter, AnotherCounter },
    methods: {
        add() {
            this.$store.commit('ADD_COUNT');
        }
    }
})
```

我们在vuex的`state`中添加了一个`count`属性并初始化为0，并且定义了一个修改state的mutation`ADD_COUNT`方法，每调用一次就自增1。要在组件中使用，需要定义一个computed属性，返回state中的count以便组件中引用。所有相关的方法，都需要按部就班地按照设置`state`，`mutations`，并在`computed`中引用store，如果有异步接口操作还需要在`actions`中定义，如果`state`比较复杂可能还需要在`module`中进行拆分，整个结构的灵活度是比较低的。


再来看看如何用Composition Api来定义类似的store状态管理，我们可以利用`reactive`定义一个全局的状态树，并且设置一些修改状态的方法，这样就很简单地达到和vuex一样的作用。

```html
<div id="app">
    <my-counter></my-counter>
    <another-counter></another-counter>
    <button @click="addCount">increment</button>
</div>
```

```javascript
const { reactive, computed } = VueCompositionAPI
const storeState = reactive({
        count: 0
    });
const addCount = () => {
    storeState.count ++;
}
const MyCounter = {
  template: `<div>{{ storeState.count }}</div>`,
  setup() {
    return { storeState }
  }
}
const AnotherCounter = {
  template: `<div>multipled by 2：{{ count }}</div>`,
  setup() {
    const count = computed(() => storeState.count * 2);
    return { count }
  }
}
new Vue({
    el: '#app',
    components: { MyCounter, AnotherCounter },
    setup() {
        return { addCount }
    }
})
```

与上面vuex的代码相比较，使用了composition api更加符合平时编写代码的逻辑，无需再一些特定属性里面定义数据状态，修改数据状态的一些方法，也减少了固定Api的使用，可以按照自己的想法封装修改状态的函数。唯一的一问题是暂时没有办法实现vuex自带的time-travel功能（记录每次修改状态的操作）。

##### 类型推导 

在Vue3.0中一个设计主要设计目标是增强对 TypeScript 的支持，所以在3.0中的代码所有框架代码都使用了TypeScript编写。在尤大的博客中，本来是期望使用Class Api达成这个目标，但是经过原型开发阶段后发现有很多问题，就改用了函数来实现。使用class风格的api有这么几个问题：

* 将vue组件上的属性合并到this上下文
* 装饰器问题
* 打包尺寸


了解过vue2.x的class-component-api中，为了在类里其他的函数使用props以及获取类型推导，常常需要通过各类装饰器来为class中的this添加类型推导支持（有些如`this.$refs`，无法事先添加类型推导只能手动断言），实现非常绕，明显不是一个好的路线。再一个就是装饰器的问题。目前TS中的装饰器实现已经与TC39规范相差甚远，目前装饰器在JS中也只是一个stage-2的提案，未来这个语法的特性很不确定。再一个就是class对tree-shaking不是很友好，不利于优化打包尺寸。

在TypeScript中，函数是对类型推导来说相对支持最好，无论是参数，泛型，返回值都有着不错的支持，并且函数实现的代码经过编译后与TS相差不大，基本就是抹去了类型。同时基于函数的 API可以被单独作为ES Module引入，自然打包尺寸更加容易控制。



#### 封装一些组件的思路（rollupjs打包）

动态配置化，表单状态提升实现复杂联动（脱离组件层面依赖，复用表单联动逻辑）


在做偏业务管理后台的时候，经常会使用到Element UI来进行快速页面搭建，其中最常见的就是表单需求。在Element UI中，典型的表单由`el-form`组件以及`el-form-item`组件构成，官方例子如下：

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
      <el-date-picker type="date" placeholder="选择日期" v-model="form.date1" style="width: 100%;"></el-date-picker>
    </el-col>
    <el-col class="line" :span="2">-</el-col>
    <el-col :span="11">
      <el-time-picker placeholder="选择时间" v-model="form.date2" style="width: 100%;"></el-time-picker>
    </el-col>
  </el-form-item>
  <el-form-item label="即时配送">
    <el-switch v-model="form.delivery"></el-switch>
  </el-form-item>
  <el-form-item label="活动性质">
    <el-checkbox-group v-model="form.type">
      <el-checkbox label="美食/餐厅线上活动" name="type"></el-checkbox>
      <el-checkbox label="地推活动" name="type"></el-checkbox>
      <el-checkbox label="线下主题活动" name="type"></el-checkbox>
      <el-checkbox label="单纯品牌曝光" name="type"></el-checkbox>
    </el-checkbox-group>
  </el-form-item>
  <el-form-item label="特殊资源">
    <el-radio-group v-model="form.resource">
      <el-radio label="线上品牌商赞助"></el-radio>
      <el-radio label="线下场地免费"></el-radio>
    </el-radio-group>
  </el-form-item>
  <el-form-item label="活动形式">
    <el-input type="textarea" v-model="form.desc"></el-input>
  </el-form-item>
  <el-form-item>
    <el-button type="primary" @click="onSubmit">立即创建</el-button>
    <el-button>取消</el-button>
  </el-form-item>
</el-form>
<script>
  export default {
    data() {
      return {
        form: {
          name: '',
          region: '',
          date1: '',
          date2: '',
          delivery: false,
          type: [],
          resource: '',
          desc: ''
        }
      }
    },
    methods: {
      onSubmit() {
        console.log('submit!');
      }
    }
  }
</script>
```

#### 总结

缺点：可能过于灵活，缺少了约束代码会更加开放。


