---
title: Vue3.0源码阅读（一）
tag: 源码分析
date: 2020-04-16
updated: 2020-04-16
---

在开始之前，让我们先了解一下vue3.0的主要变化
- 编译器（Compiler）
	- 使用模块化架构
	- 优化 "Block tree"
	- 更激进的 static tree hoisting 功能 （检测静态语法，进行提升）
	- 支持 Source map
	- 内置标识符前缀（又名"stripWith"）
	- 内置整齐打印（pretty-printing）功能
	- 移除 Source map 和标识符前缀功能后，使用 Brotli 压缩的浏览器版本精简了大约10KB

- 运行时（Runtime）
	- 速度显著提升
	- 同时支持 Composition API 和 Options API，以及 typings
	- 基于 Proxy 实现的数据变更检测
	- 支持 Fragments (允许组件有从多个根结点)
	- 支持 Portals (允许在DOM的其它位置进行渲染)
	- 支持 Suspense w/ async setup()

对于平时使用vue的我们来说，这其中变化最大的应该新增的**Composition API与Typescipt的支持** , Composition的Api最显著的特点就是移除了各种生命周期的api，转而由一个setup()的函数完成整个组件的初始化过程。在setup函数中，使用**ref, reactive**定义响应式数据对象，使用**computed**定义一个需要经过映射的值，使用**watchEffect**来观察组件状态的变化并且定义改变组件的行为。让我们来看一个来自官方RFC的例子：
组件代码:
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

模板代码：
```
<div id="app">
    <button @click="enableMousePosition">点我实时显示鼠标位置</button>
    <p v-if="isShowMousePos">鼠标x: {{x}} 鼠标: {{y}}</p>
</div>
```
在上面的代码里面，不同于vue2.x的代码，没有使用`this`去引用实例修改某个值，而是改为通过ref定义一个相应式的对象，要修改它只需要简单的对这个对象的value进行重新赋值。组件中只需要简单地通过函数引入响应式变量，无需知晓里面发生的具体逻辑。
相对于原来的optional方式的写法，`useMousePosition`里面用于更新坐标的逻辑无需与组件本身的生命周期钩子（mounted）进行强制绑定，转而由实现方法的函数内导入单独的onMounted方法进行处理，Composition API的写法可以将具体更新坐标的逻辑与组件本身相分离，从而可以是组件本身的代码更加精简。
**与react hook的区别**

熟悉react的同学也发现了Composition API整体风格非常像react hooks。与react hooks，vue的Composition API还有着如下特点：
* Vue通过ref创建的响应式对象是mutable（可修改）的，react通过语句`const [count, setCount] = useState(0);`同时创建响应式对象和修改器 ，通过修改器来push改变对象的方法来修改响应式对象，vue则直接修改响应式对象的value即可。当然，react hooks这种用法是与react本身immutable理念保持一致，不能算作是缺点。vue修改对象用法更加符合写自然代码的直觉。
* Vue的`setup`只会在组件创建的时候运行一次，与react的hooks相比，定义的hooks将会在组件每次重新渲染的时候调用，这就导致如果在条件语句或循环里面定义hooks，可能会导致很多意料之外的问题。vue的setup由于只调用一次不会有这方面的问题。初次之外，还要关心各种调用顺序，渲染顺序。
* 生命周期方面：react hooks弱化了生命周期的概念，废弃了很多如`componentWillMount、 componentWillUpdate、 componentWillReceiveProp`这类方法，Composition API则基本保留了2.x时代所有的生命周期方法。

### 组件如何被setup以及只被运行一次的？###

vue3.0的初始化组件从`new vue(option)`改为了`createApp(option)`这种形式。让我们从`createApp`这个函数开始看起。

vue/src/index.ts

```javascript
// 省略了一些导入的代码
// compileToFunction的作用是导入dom环境模板编译器，将目标template元素最终编译成render函数并且返回
function compileToFunction(
  template: string | HTMLElement,
  options?: CompilerOptions
): RenderFunction {
  const { code } = compile(template, {
    hoistStatic: true,
    onError(err: CompilerError) {
      ...
    },
    ...options
  })
  ...
  const render = (__GLOBAL__
    ? new Function(code)()
    : new Function('Vue', code)(runtimeDom)) as RenderFunction
  // compileCache的key是模板字符串
  return (compileCache[key] = render) // 放进编译缓存并且返回render
}

// 注册运行时编译器
registerRuntimeCompiler(compileToFunction)

export { compileToFunction as compile }
export * from '@vue/runtime-dom' // 从这里导出createApp, ref, 创建应用函数

import './devCheck'

```
在入口文件，指定并注册了模板编译到render函数的方法（后面setup的过程会调用render），随后将runtime-dom包将相关api进行导出供代码使用。

随后在runtime-dom中index.ts有一个 `createApp`的方法，这里的`createApp`方法实际上是将真正的`createApp`做了一层包装和指定mount需要编译目标template：
```typescript
export const createApp = ((...args) => {
 // args
  const app = ensureRenderer().createApp(...args) // ensureRenderer将会把renderer.ts中的render和createApp方法进行导出来给到这里的createApp进行调用以及提供后面的mount方法调用render

  const { mount } = app
  app.mount = (containerOrSelector: Element | string): any => {
    const container = normalizeContainer(containerOrSelector)
    if (!container) return
    const component = app._component
    if (
      __RUNTIME_COMPILE__ &&
      !isFunction(component) &&
      !component.render &&
      !component.template
    ) {
      component.template = container.innerHTML // 将模板信息放入component
    }
    // clear content before mounting
    container.innerHTML = ''
    return mount(container) // 指定mount的container
  }

  return app
}) as CreateAppFunction<Element>
```
真正的`createApp`函数，在`runtime-core/src/apiCreateApp.ts` 专门的一个文件来维护，先从返回的createApp函数app对象类型定义看起：
```javascript
export interface App<HostElement = any> {
  config: AppConfig
  use(plugin: Plugin, ...options: any[]): this
  mixin(mixin: ComponentOptions): this
  component(name: string): Component | undefined
  component(name: string, component: Component): this
  directive(name: string): Directive | undefined
  directive(name: string, directive: Directive): this
  mount(
    rootContainer: HostElement | string,
    isHydrate?: boolean
  ): ComponentPublicInstance
  unmount(rootContainer: HostElement | string): void
  provide<T>(key: InjectionKey<T> | string, value: T): this

  // internal. We need to expose these for the server-renderer
  _component: Component
  _props: Data | null
  _container: HostElement | null
  _context: AppContext
}

```
app返回了很多2.x熟悉的一些api：
* use 安装插件
* mixin 支持组件混入
* component 注册自定义组件
* directive 注册指令
* mount 将createApp后返回的app进行挂载
* unmount 卸载组件
这些函数功能与vue2.x总体来说是差不多的，先来看看craeteApp中的`mount`函数中做了些什么：
```javascript
export function createAppAPI<HostNode, HostElement>(
  render: RootRenderFunction<HostNode, HostElement>,
  hydrate?: (vnode: VNode, container: Element) => void
): CreateAppFunction<HostElement> {
  return function createApp(rootComponent: Component, rootProps = null) {
    // 第一个rootComponent就是用户编写的组件配置（template配置，setup()或者传统的mounted()等）
    // 第二个就是组件props配置
    if (rootProps != null && !isObject(rootProps)) {
      __DEV__ && warn(`root props passed to app.mount() must be an object.`)
      rootProps = null
    }

    const context = createAppContext()
    const installedPlugins = new Set()

    let isMounted = false

    // 先省略mixin，use，directive等插件有关函数
    // 这里先看mount函数
    mount(rootContainer: HostElement, isHydrate?: boolean): any {
      if (!isMounted) {
        // 这里的vnode包含着选项配置，模板字符串等初始信息
        // 后面才进行初始化
        const vnode = createVNode(rootComponent, rootProps)
        // store app context on the root VNode.
        // this will be set on the root instance on initial mount.
        vnode.appContext = context

        // 省略一些HMR和SSR有关的代码...
        render(vnode, rootContainer) // 渲染vnode
        isMounted = true // 标记为已mount
        app._container = rootContainer
        return vnode.component!.proxy
      } else if (__DEV__) {
        warn(
          `App has already been mounted. Create a new app instance instead.`
        )
      }
    }
  }
}
```
为什么是createApp这个函数是由createAppAPI这个函数返回的？在哪里调用的？
来看到runtime/src/renderer.ts最后几行：
```
return {
    render,
    hydrate,
    createApp: createAppAPI(render, hydrate) as CreateAppFunction<HostElement> // 将renderer.ts里的render通过闭包的方式给到createApp函数里面使用
  }
```
renderer.ts这个文件很长，大约2000左右，里面包含了vue核心的虚拟dom的生成，更新算法，组件的挂载与状态更新，响应式等非常多的内容，后面篇幅会逐渐详细解析里面每一块的作用。
在mount里面有一个createVNode的函数，这个即为vue里面非常核心的生成虚拟dom节点的方法，先来看一下vnode的定义：
runtime/src/vnode.ts 这里面都是关于vnode的有关的定义和方法，VNode有关的定义：
```javascript
export interface VNode<HostNode = any, HostElement = any> {
  _isVNode: true
  type: VNodeTypes
  props: VNodeProps | null
  key: string | number | null
  ref: string | Ref | ((ref: object | null) => void) | null
  scopeId: string | null // SFC only
  children: VNodeNormalizedChildren<HostNode, HostElement>
  component: ComponentInternalInstance | null
  suspense: SuspenseBoundary<HostNode, HostElement> | null
  dirs: DirectiveBinding[] | null
  transition: TransitionHooks | null

  // DOM
  el: HostNode | null
  anchor: HostNode | null // fragment anchor
  target: HostElement | null // portal target

  // optimization only
  shapeFlag: number
  patchFlag: number
  dynamicProps: string[] | null
  dynamicChildren: VNode[] | null

  // application root node only
  appContext: AppContext | null
}
```
属性还是挺多的，但是了解过虚拟的dom的同学都知道，虚拟dom无非就是对dom节点的一种抽象描述，里面最主要包括的就是`节点类型`，`节点内容`，`子节点`，`目标节点`，`组件信息`。VNode是一种对真实渲染的节点的抽象，大家都知道操作dom是一种非常费时的操作，有了虚拟dom的抽象，在合适的更新策略下，能很大程度提高渲染到真实dom（或者是其他目标平台）的速度。
这里的虚拟dom的定义和2.0不一样的有一个地方是type，这里的type不仅仅是字符串(2.0对应的是一个tag，一般是一个字符)，这里的type还可以是组件的配置对象。

来看到createVNode的方法实现：
```javascript
export function createVNode(
  type: VNodeTypes, // 一般情况下是字符串，也可以是组件选项对象
  props: (Data & VNodeProps) | null = null,
  children: unknown = null,
  patchFlag: number = 0,
  dynamicProps: string[] | null = null
): VNode {
  // class & style normalization. 规范化clas和style对象，并且合并进props里
  if (props !== null) {
    // for reactive or proxy objects, we need to clone it to enable mutation.
    if (isReactive(props) || SetupProxySymbol in props) {
      props = extend({}, props)
    }
    let { class: klass, style } = props
    if (klass != null && !isString(klass)) {
      props.class = normalizeClass(klass)
    }
    if (isObject(style)) {
      // reactive state objects need to be cloned since they are likely to be
      // mutated
      if (isReactive(style) && !isArray(style)) {
        style = extend({}, style)
      }
      props.style = normalizeStyle(style)
    }
  }

  // encode the vnode type information into a bitmap
  // 定义shapeFlag，后面patch函数会根据这个判断类型
  const shapeFlag = isString(type)
    ? ShapeFlags.ELEMENT
    : __FEATURE_SUSPENSE__ && isSuspense(type)
      ? ShapeFlags.SUSPENSE
      : isPortal(type)
        ? ShapeFlags.PORTAL
        : isObject(type)
          ? ShapeFlags.STATEFUL_COMPONENT // STATEFUL_COMPONENT就是对象创建组件
          : isFunction(type)
            ? ShapeFlags.FUNCTIONAL_COMPONENT
            : 0
 // 创建的vnode主要先定义type，shapeFlag以及props
  const vnode: VNode = {
    _isVNode: true,
    type, // 这里type第一次从mount调用的时候就是createApp传入的配置对象
    props,
    key: (props !== null && props.key) || null,
    ref: (props !== null && props.ref) || null,
    scopeId: currentScopeId,
    children: null,
    component: null,
    suspense: null,
    dirs: null,
    transition: null,
    el: null,
    anchor: null,
    target: null,
    shapeFlag,
    patchFlag,
    dynamicProps,
    dynamicChildren: null,
    appContext: null
  }

  normalizeChildren(vnode, children) // 规范化子节点
  /*
  后面一些处理暂时省略
  */	
  return vnode
}
```
从`createApp(option)`到`mount`后，主要步骤可以概括为：
* 定义模板信息，
* 初始化上下文的环境，
* 规范化参数(class以及style)
* 建立根节点VNode
* 渲染根节点VNode

在这些事情完成了以后，接下来就开始调用render渲染根节点VNode了。运行setup，模板，组件的解析，响应式对象的创建和绑定都将在render这个函数中完成。

上面代码有一个`shapeFlag`的枚举定义，我们先来看一下这个枚举的定义，后面的代码里有多处用到这个枚举：
```javascript
export const enum ShapeFlags {
  ELEMENT = 1,
  FUNCTIONAL_COMPONENT = 1 << 1, // 1
  STATEFUL_COMPONENT = 1 << 2, // 二进制为10，十进制为2
  TEXT_CHILDREN = 1 << 3, // 二进制为100，十进制就是8
  ARRAY_CHILDREN = 1 << 4, // 16
  SLOTS_CHILDREN = 1 << 5, // 32
  PORTAL = 1 << 6, 
  SUSPENSE = 1 << 7,
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8,
  COMPONENT_KEPT_ALIVE = 1 << 9,
  COMPONENT = ShapeFlags.STATEFUL_COMPONENT | ShapeFlags.FUNCTIONAL_COMPONENT
}
```
这里枚举的值使用了位运算定义。<< 操作符就是将当前数的二进制形式向左移 n个 比特位，右边用0填充。为什么用位运算呢？从后面的代码来看，这样位运算类型可以包含多种类型枚举信息，可以简单的用与，或运算进行多种组件类型的判断。上面的枚举定义最后一个COMPONENT类型定义：
`COMPONENT = ShapeFlags.STATEFUL_COMPONENT | ShapeFlags.FUNCTIONAL_COMPONENT`
`ShapeFlags.STATEFUL_COMPONENT`的值是2（10），`ShapeFlags.FUNCTIONAL_COMPONENT`的值是1（1）他们之间进行或运算二进制就是11(十进制为3)。

再来看到`normalizeChildren`函数：
```javascript
// 设置传入children的type
export function normalizeChildren(vnode: VNode, children: unknown) {
  let type = 0
  if (children == null) {
    children = null
  } else if (isArray(children)) {
    type = ShapeFlags.ARRAY_CHILDREN
  } else if (typeof children === 'object') {
    type = ShapeFlags.SLOTS_CHILDREN
  } else if (isFunction(children)) {
    children = { default: children }
    type = ShapeFlags.SLOTS_CHILDREN
  } else {
    children = String(children)
    type = ShapeFlags.TEXT_CHILDREN
  }
  vnode.children = children as VNodeNormalizedChildren
  vnode.shapeFlag |= type // 将slot的type通过与运算合并到vnode上到shapeFlag
}
```
这个函数的作用主要就是将slot的类型枚举“合并到”`vnode.shapeFlag`上。如果组件类型是`STATEFUL_COMPONENT`，他的二进制值是10，如果这个组件的children是`TEXT_CHILDREN`，二进制值是100，那么经过或运算后，`shapeFlag`的值就是110。这就意味着这个枚举值的类型同时包含了`STATEFUL_COMPONENT`类型和`TEXT_CHILDREN`类型。这样在之后的代码里，通过`shapeFlag & ShapeFlags.TEXT_CHILDREN` 或者`shapeFlag & ShapeFlags.STATEFUL_COMPONENT`都将为true（&是与运算，110 & 100结果为1，100 & 10 也是1）这样一个枚举信息里面就能包含并判断多个类型信息。

####render以及patch节点
在renderer.ts中1870行左右是mount中用到的render函数的定义
```javascript
// 渲染vnode
  const render: RootRenderFunction<HostNode, HostElement> = (
    vnode, // 要解析的vnode
    container: HostRootElement // 解析完最终要插入的父节点
  ) => {
    if (vnode == null) {
      if (container._vnode) {
        unmount(container._vnode, null, null, true)
      }
    } else {
      // 解析并patch vnode
      // 从根节点进入到渲染流程的时候container是包含template模板的根节点
      // 这里传入的第一个参数结果是null
      patch(container._vnode || null, vnode, container)
    }
    flushPostFlushCbs()
    container._vnode = vnode
  }
```
来看到patch函数的签名部分：
```javascript
const patch: PatchFn<HostNode, HostElement> = (
    n1, // 旧vnode节点，没渲染过之前一定是null
    n2, // 新的等待渲染vnode节点
    container,
    anchor = null,
    parentComponent = null,
    parentSuspense = null,
    isSVG = false,
    optimized = false
  ) => {
   ...各种vnode类型的判断，进入到相应的处理流程
  }
```
了解过一点vue原理的同学应该之前有了解到vue更新组件的机制是比较新旧两颗树，取出其中差异的部分进行更新。n1和n2这个两个参数实际上就是旧节点和新节点，container就是包含渲染后结果的容器元素。这里我们先暂时不了解diff以及更新的过程，来看一下作为一个根组件是如何被初始化，运行setup以及建立响应关系最后插入到dom中的。

从上面mount的过程中可以了解到，n2的类型是一个组件，shapeFlag是STATEFUL_COMPONENT，vnode的type里面包含着一开始就传进来的包含setup()的组件选项。有了这些，patch函数中内部将直接跳转到`processComponent`这个函数：
```javascript
const processComponent = (
    n1: HostVNode | null, // n1是旧vnode节点状态
    n2: HostVNode, // n2是当前vnode节点状态
    container: HostElement,
    anchor: HostNode | null,
    parentComponent: ComponentInternalInstance | null,
    parentSuspense: HostSuspenseBoundary | null,
    isSVG: boolean,
    optimized: boolean
  ) => {
    if (n1 == null) {
	  // n1是null的时候，说明此时组件还没有被mount（或者是keepAlive的暂时被隐藏）
      // 如果是keepAlive的情况，调用active从内部实例取出之前mount的结果
      if (n2.shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE) {
        ;(parentComponent!.sink as KeepAliveSink).activate(
          n2,
          container,
          anchor
        )
      } else {
        // mount组件
        mountComponent(
          n2,
          container,
          anchor,
          parentComponent,
          parentSuspense,
          isSVG
        )
      }
    } else {
	    /*更新组件的流程，后面再展开*/
    }
  }
```
来重点看一下`mountComponent`中发生的事情：
```typescript
const mountComponent: MountComponentFn<HostNode, HostElement> = (
    initialVNode,
    container, // only null during hydration
    anchor,
    parentComponent,
    parentSuspense,
    isSVG
  ) => {
    // 创建一个内部实例，并且将它保存在VNode的component属性上
    const instance: ComponentInternalInstance = (initialVNode.component = createComponentInstance(
      initialVNode,
      parentComponent
    ))

    if (__HMR__ && instance.type.__hmrId != null) {
      registerHMR(instance)
    }

    if (__DEV__) {
      pushWarningContext(initialVNode)
    }

    // inject renderer internals for keepAlive
    if (isKeepAlive(initialVNode)) {
      const sink = instance.sink as KeepAliveSink
      sink.renderer = internals
      sink.parentSuspense = parentSuspense
    }

    // resolve props and slots for setup context
    // 调用setup函数
    setupComponent(instance, parentSuspense)
    // setup() is async. This component relies on async logic to be resolved
    // before proceeding
    if (__FEATURE_SUSPENSE__ && instance.asyncDep) {
      if (!parentSuspense) {
        if (__DEV__) warn('async setup() is used without a suspense boundary!')
        return
      }

      parentSuspense.registerDep(instance, setupRenderEffect)

      // Give it a placeholder if this is not hydration
      const placeholder = (instance.subTree = createVNode(Comment))
      processCommentNode(null, placeholder, container!, anchor)
      initialVNode.el = placeholder.el
      return
    }

    // 组件设置响应式回调
    setupRenderEffect(
      instance,
      initialVNode,
      container,
      anchor,
      parentSuspense,
      isSVG
    )

    if (__DEV__) {
      popWarningContext()
    }
  }
```
上面代码主要做了这些事情：
* 设置并创建内部实例
* 调用setupComponent()初始化组件
* setupRenderEffect()设置相应式副作用

首先来看一下内部实例是如何创建的：
```javascript
export function createComponentInstance(
  vnode: VNode,
  parent: ComponentInternalInstance | null
) {
  // inherit parent app context - or - if root, adopt from root vnode
  const appContext =
    (parent ? parent.appContext : vnode.appContext) || emptyAppContext
  const instance: ComponentInternalInstance = {
    vnode,
    parent,
    appContext,
    type: vnode.type as Component,
    root: null!, // set later so it can point to itself
    next: null,
    subTree: null!, // will be set synchronously right after creation
    update: null!, // will be set synchronously right after creation
    render: null,
    proxy: null,
    withProxy: null,
    propsProxy: null,
    setupContext: null,
    effects: null,
    provides: parent ? parent.provides : Object.create(appContext.provides),
    accessCache: null!,
    renderCache: [],

    // setup context properties
    renderContext: EMPTY_OBJ,
    data: EMPTY_OBJ,
    props: EMPTY_OBJ,
    attrs: EMPTY_OBJ,
    vnodeHooks: EMPTY_OBJ,
    slots: EMPTY_OBJ,
    refs: EMPTY_OBJ,

    // per-instance asset storage (mutable during options resolution)
    components: Object.create(appContext.components),
    directives: Object.create(appContext.directives),

    // async dependency management
    asyncDep: null,
    asyncResult: null,
    asyncResolved: false,

    // user namespace for storing whatever the user assigns to `this`
    // can also be used as a wildcard storage for ad-hoc injections internally
    sink: {},

    // lifecycle hooks
    // not using enums here because it results in computed properties
    isMounted: false,
    isUnmounted: false,
    isDeactivated: false,
    bc: null,
    c: null,
    bm: null,
    m: null,
    bu: null,
    u: null,
    um: null,
    bum: null,
    da: null,
    a: null,
    rtg: null,
    rtc: null,
    ec: null,

    emit: (event, ...args): any[] => {
      const props = instance.vnode.props || EMPTY_OBJ
      let handler = props[`on${event}`] || props[`on${capitalize(event)}`]
      if (!handler && event.indexOf('update:') === 0) {
        event = hyphenate(event)
        handler = props[`on${event}`] || props[`on${capitalize(event)}`]
      }
      if (handler) {
        const res = callWithAsyncErrorHandling(
          handler,
          instance,
          ErrorCodes.COMPONENT_EVENT_HANDLER,
          args
        )
        return isArray(res) ? res : [res]
      } else {
        return []
      }
    }
  }

  instance.root = parent ? parent.root : instance
  return instance
}
```
上面代码中内部实例里最主要保存的是组件的上下文关系，当前对应的组件VNode，组件选项，还有各种生命周期钩子的初始化，以及一个emit函数。emit函数很简单，就是组件内调用emit的时候处理好大小写直接携带参数调用props上面on开头的函数。
这里的EMPTY_OBJ的定义实际上就是返回一个Object.freeze()创建的不可修改的空对象。这些对象不能新建或者修改属性，只能将它替换为其他的对象。

设置完内部实例后再来到setupComponent：
```typescript
export function setupComponent(
  instance: ComponentInternalInstance,
  parentSuspense: SuspenseBoundary | null,
  isSSR = false
) {
  isInSSRComponentSetup = isSSR
  const propsOptions = instance.type.props
  const { props, children, shapeFlag } = instance.vnode
  resolveProps(instance, props, propsOptions) // 处理并保存props到内部实例instance.props上，和设置propsProxy
  resolveSlots(instance, children) // 处理并保存slots到内部实例instance.slots上

  // setup stateful logic
  let setupResult
  if (shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
    setupResult = setupStatefulComponent(instance, parentSuspense)
  }
  isInSSRComponentSetup = false
  return setupResult
}
```
首先先处理props和组件的slots（插槽）将它们保存到内部实例相应的属性中（暂时跳过SSR的一些代码），然后调用setupStatefulComponent初始化组件，在这一步将会调用组件定义时选项中的setup函数：
```javascript
function setupStatefulComponent(
  instance: ComponentInternalInstance,
  parentSuspense: SuspenseBoundary | null
) {
  const Component = instance.type as ComponentOptions

  if (__DEV__) {
    /*这里都是开发模式验证组件或指令名称是否有效的一些方法，暂且忽略*/
  }
  
  // 0. create render proxy property access cache
  // 设置内部实例cache
  instance.accessCache = {}
  // 1. create public instance / render proxy 设置内部实例的proxy对象
  instance.proxy = new Proxy(instance, PublicInstanceProxyHandlers)
  // 2. create props proxy 设置props的proxy
  // the propsProxy is a reactive AND readonly proxy to the actual props.
  // it will be updated in resolveProps() on updates before render
  const propsProxy = (instance.propsProxy = isInSSRComponentSetup
    ? instance.props
    : shallowReadonly(instance.props))
  // 3. call setup()
  // 调用setup函数进行初始化
  // 设置setup函数上下文
  const { setup } = Component
  if (setup) {
    const setupContext = (instance.setupContext =
      setup.length > 1 ? createSetupContext(instance) : null)

    currentInstance = instance
    currentSuspense = parentSuspense
    pauseTracking()
    // 将props的proxy传入setup，以供setup内的代码使用props传入的组件属性
    // setup的写法：setup(props) { ...一些使用到props的代码 }
    // props是一个响应式对象，同时应该是只读的（props和2.0一样不应该被修改）
    const setupResult = callWithErrorHandling(
      setup,
      instance,
      ErrorCodes.SETUP_FUNCTION,
      [propsProxy, setupContext]
    )
    resetTracking()
    currentInstance = null
    currentSuspense = null

    if (isPromise(setupResult)) {
      /**setupResult只有是SUSPNESE或者在SSR环境里才可能返回promise，这里先跳过*/
    } else {
      // 处理setup函数返回的结果
      handleSetupResult(instance, setupResult, parentSuspense)
    }
  } else {
    // 没有setup函数直接调用完成setup后处理的函数
    finishComponentSetup(instance, parentSuspense)
  }
}
```
后面就是setup函数返回的结果处理了，再看之前，先了解一下官方setup对象可以返回什么：
* 响应式对象
* render函数

如果是响应式对象的话，定义的模板里将直接使用这里返回的响应式对象，用来实现视图的渲染以及更新（作用相当于vue2.0的data返回的对象）
如果是render函数的话（通常情况下就是jsx），将略过模板编译，直接使用setup返回的render函数。

handleSetupResult函数：
```javascript
export function handleSetupResult(
  instance: ComponentInternalInstance,
  setupResult: unknown,
  parentSuspense: SuspenseBoundary | null
) {
  console.warn('setup result', setupResult)
  if (isFunction(setupResult)) {
    // 返回的内联render函数
    instance.render = setupResult as RenderFunction
  } else if (isObject(setupResult)) {
    // setup函数不能直接返回vnodes
    if (__DEV__ && isVNode(setupResult)) {
      warn(
        `setup() should not return VNodes directly - ` +
          `return a render function instead.`
      )
    }
    // setup returned bindings.
    // assuming a render function compiled from template is present.
    // 这里假设模板已编译。如果setup返回的是对象，将返回的对象设置为响应式对象并且设置于渲染上下文上（即传入render函数上的入参，这样就可以在render里面使用到setup函数内设置的响应式对象）
    instance.renderContext = reactive(setupResult) // 将返回的对象设置为响应式
  } else if (__DEV__ && setupResult !== undefined) {
    warn(
      `setup() should return an object. Received: ${
        setupResult === null ? 'null' : typeof setupResult
      }`
    )
  }
  finishComponentSetup(instance, parentSuspense)
}
```
setup中返回对象的例子：
```javascript
const MyComponent = {
  props: {
    name: String
  },
  setup(props) {
    return {
      msg: `hello ${props.name}!`
    }
  },
  template: `<div>{{ msg }}</div>`
}
```
返回的对象上的属性将会被暴露给模版的渲染上下文，msg 可以在模版中被直接使用，也可以直接被模版中的内联函数修改，模板都会同步得到更新。
上面代码中，如果有render函数直接设置了render函数，如果没有render函数，意味着template还没有编译成模板，接着往下看finishComponentSetup
```typescript
// 完成setup调用
// 如果没有render函数，则尝试编译template模板
function finishComponentSetup(
  instance: ComponentInternalInstance,
  parentSuspense: SuspenseBoundary | null
) {
  const Component = instance.type as ComponentOptions
  if (!instance.render) {
    if (__RUNTIME_COMPILE__ && Component.template && !Component.render) {
      // __RUNTIME_COMPILE__ ensures `compile` is provided
      // 还记得入口文件注册的compiler吗？这里的compile就是入口文件注册的compileToFunction方法
      // 会通过compile这个方法，尝试将模板编译成包含createVNode的render函数
      Component.render = compile!(Component.template, {
        isCustomElement: instance.appContext.config.isCustomElement || NO
      })
      // mark the function as runtime compiled
      ;(Component.render as RenderFunction).isRuntimeCompiled = true
    }

    if (__DEV__ && !Component.render && !Component.ssrRender) {
      /* istanbul ignore if */
      if (!__RUNTIME_COMPILE__ && Component.template) {
        warn(
          `Component provides template but the build of Vue you are running ` +
            `does not support runtime template compilation. Either use the ` +
            `full build or pre-compile the template using Vue CLI.`
        )
      } else {
        warn(
          `Component is missing${
            __RUNTIME_COMPILE__ ? ` template or` : ``
          } render function.`
        )
      }
    }

    // 将编译好的render函数（或者用户定义的render函数）设置到内部实例上的render属性
    instance.render = (Component.render || NOOP) as RenderFunction
    // for runtime-compiled render functions using `with` blocks, the render
    // proxy used needs a different `has` handler which is more performant and
    // also only allows a whitelist of globals to fallthrough.
    if (__RUNTIME_COMPILE__ && instance.render.isRuntimeCompiled) {
      instance.withProxy = new Proxy(
        instance,
        runtimeCompiledRenderProxyHandlers
      )
    }
  }

  // support for 2.x options
  if (__FEATURE_OPTIONS__) {
    currentInstance = instance
    currentSuspense = parentSuspense
    applyOptions(instance, Component)
    currentInstance = null
    currentSuspense = null
  }
  if (instance.renderContext === EMPTY_OBJ) {
    instance.renderContext = {}
  }
}
```
到这个函数为止，一个组件的从设置模板，到运行setup，设置上下文基本上是完成了。但是到这个阶段位置，只是确定了render和props，slots和一些context，还没有真正调用编译后的render（或者jsx），以及建立响应式关系。在上面的mountComponent函数中setupComponent之后，后面将开始调用setupRenderEffect调用render以及建立各种响应式回调。后面会继续分析响应式系统这块的实现。
