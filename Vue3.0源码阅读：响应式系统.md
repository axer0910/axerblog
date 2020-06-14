---
title: Vue3.0源码阅读：响应式系统
tag: 源码分析
date: 2020-04-23
updated: 2020-04-23
---

在上一篇文章中，分析了一个vue3.0的app根组件从设置选项加载到mount的过程，进入到首次patch的流程，从template生成render函数，以及运行它的setup。这篇文章主要是开始分析组件是如何调用编译好的render一步步去渲染dom以及子组件，以及如何与视图绑定响应式变量，修改一个响应式变量的时候如何更新视图。

vue3.0的响应式系统是独立的，在`@vue/reactivity`的readme中，清楚地表明可以独立拿出来使用。我们将从几个单元测试的例子详解这套响应式基本原理以及实现，后面再来分析在组件代码里应用。

## 从基本的测试例子开始 ##

我们先来看一下这套源代码里面一个简单的ref与effect的单元测试实现基本的响应式效果：

```
it('should be reactive', () => {
    const a = ref(1) // 通过ref包装数值1并作为响应式变量赋值给a，读写a的值需要通过a.value实现
    let dummy
    effect(() => {
      dummy = a.value // 将a的值赋值给dummy，如果a的值改变了，dummy的值会被同步赋值更新
    })
    expect(dummy).toBe(1) // dummy的值是初始的1
    a.value = 2 // 将a的值从1改成2
    expect(dummy).toBe(2) // dummy同步被更新，dummy = a.value语句再次被运行，从1变成了2
  })
```
在这个单元测试中，通过`ref()`将a`定义为一个响应式变量，ref() 返回的是一个 value reference （包装对象）。一个包装对象只有一个属性：.value ，该属性指向内部被包装的值。这个单元测试里，a是一个数值的响应式包装，修改a的值需要修改它的value。修改它的value将会通过effect传入的回调同步给dummy这个变量。

### effect函数 ###
effect可以理解为在观测的对象发生变化以后，执行的回调。Vue里所有用到的更新操作最后都会通过`effect`去实际运行。

来看一下effect函数的定义：
reactivity/src/effect.ts
```
// 接受一个回调并创建一个effect
export function effect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions = EMPTY_OBJ
): ReactiveEffect<T> {
  if (isEffect(fn)) {
    fn = fn.raw
  }
  // 创建一个ReactiveEffect
  const effect = createReactiveEffect(fn, options)
  if (!options.lazy) {
    effect() // 运行fn，并且将当前activeEffect设置为effect
  }
  return effect
}
```
effect将通过createReactiveEffect这个工厂函数创建一个响应式回调，
```
const effectStack: ReactiveEffect[] = []
export let activeEffect: ReactiveEffect | undefined // 这个变量很重要，表明当前需要被deps收集的回调

function createReactiveEffect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
  // 创建一个ReactiveEffect
  // 返回的effect首先是返回调用fn的一个函数
  // 同时还包含着deps依赖收集器，选项，状态等
  const effect = function reactiveEffect(...args: unknown[]): unknown {
    return run(effect, fn, args) // 将effect设置为activeEffect并且立即执行effect
  } as ReactiveEffect
  effect._isEffect = true
  effect.active = true
  effect.raw = fn
  effect.deps = []
  effect.options = options
  return effect
}

function run(effect: ReactiveEffect, fn: Function, args: unknown[]): unknown {
  if (!effect.active) {
    // 默认走不到这里来
    return fn(...args)
  }
  if (!effectStack.includes(effect)) { // 看看栈里有没有当前想运行的effect
    cleanup(effect)
    try {
      enableTracking()
      effectStack.push(effect) // 把要运行的effect推到当前正在执行的栈里面
      activeEffect = effect // 设置当前正在运行的effect，后面依赖收集会用到
      return fn(...args) // 运行函数并返回结果
    } finally { // 清理操作
      effectStack.pop()
      resetTracking()
      activeEffect = effectStack[effectStack.length - 1]
    }
  }
}
```
effect是对传入函数进行的一层包装，在effec函数的定义当中，通过闭包引用传入的fn交给run实际去运行函数。在run函数里面，除了运行fn以外，还有**标注当前依赖收集阶段执行的回调(ref，reactive等将会用到)**。`effect`对象里，除了保存函数，还将定义这个函数对应的依赖收集器（也就是deps），以及选项等。通过`createReactiveEffect`将一个函数构造为`effect`后，将会立即执行一次`effect`。



### 变量追踪 ###

在运行ref(1)的过程中发生了什么呢？
来看看ref函数定义：
```
function createRef(value: unknown, shallow = false) {
  if (isRef(value)) {
    return value
  }
  if (!shallow) {
    value = convert(value)
  }
  const r = {
    _isRef: true,
    get value() {
      // 获取value时，对这个对象value的修改依赖进行跟踪
      track(r, TrackOpTypes.GET, 'value')
      return value
    },
    set value(newVal) {
      // convert函数如果newVal是一个对象，那么通过reactive转换为响应式对象
      value = shallow ? newVal : convert(newVal)
      // 触发依赖这个对象的更新回调
      trigger(
        r,
        TriggerOpTypes.SET,
        'value',
        __DEV__ ? { newValue: newVal } : void 0
      )
    }
  }
  // 返回一个含有value属性的对象，value中含有getter和setter
  return r
}
```
ref主要是对原始值类型如 string 和 number 这种只有值的变量实现追踪响应，因为在js里面，只有一个值没有办法给它设置额外的属性，所以通过将它包装为对象的方法再包装一层，里面加上value对象以及getter和setter，实现依赖追踪各种需要触发的操作。

ref中，访问`value`属性的时候，会将被访问对象自身添加到依赖追踪的集合中，具体实现方法:
```
const targetMap = new WeakMap<any, KeyToDepMap>() // targetMap中保存着目标变量的引用->键值关系
const effectStack: ReactiveEffect[] = []
export let activeEffect: ReactiveEffect | undefined
export function track(target: object, type: TrackOpTypes, key: unknown) {
// activeEffect没有值的话略过追踪（因为没有改变变量后要调用的回调）
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  // void 0相当于undefined，防止undefined被重写不能正确判断
  let depsMap = targetMap.get(target)
  if (depsMap === void 0) {
    targetMap.set(target, (depsMap = new Map())) // 保存target与depsMap的关系
  }
  let dep = depsMap.get(key) // depsMap对应着对象里这个key所有的effect回调集合
  if (dep === void 0) {
    depsMap.set(key, (dep = new Set())) // 保存对应key下面所有的effect
  }
  // activeEffect就是当前effect函数中传入的响应式回调
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect) // 把当前effect响应式回调添加到dep依赖收集中
    activeEffect.deps.push(dep) // effect的deps里添加这个依赖
  }
}
```
在track过程中，有三个关键变量：`activeEffect`，`targetMap`，`depsMap`。`activeEffect`就是上文中`effect`传入的回调函数，这个回调函数在加入一些属性后（deps等）将会被立即执行（如果`lazy`是`false`），立即执行之前会将它设置为`activeEffect`。
当在运行响应式回调的过程中，遇到了由`ref`函数生成的变量，`track`函数会先在`targetMap`中建立**响应式对象的引用**与**键值（`ref`函数生成的对象键值就是`value`）**对应的`depsMap`集合映射关系，再把`activeEffect`推入到`depsMap`集合当中。如果响应式对象在多个`effect`回调中被引用，那么deps集合里面就会有多个`effect`回调。有了这个deps集合，当修改响应式变量的时候就可以一次性全部运行deps里面的回调，实现依赖追踪的效果。
// todo 画个图
###变量更新
修改一个由ref对象包装的值，需要直接对它的`value`属性进行修改，重新赋值。修改`value`时候，将会触发`value`上的`getter`：
```
set value(newVal) {
     
      value = shallow ? newVal : convert(newVal) // 如果newVal是一个对象，那么现将它转换为响应式对象再赋值
      // 更新订阅这个变量的依赖项
      trigger(
        r,
        TriggerOpTypes.SET,
        'value',
        void 0
      )
    }
```
convert函数里实际上就是将newVal转为reactive对象（后面会再专门分析转换为reactive对象的过程）,然后就开始调用`trigger`函数：
```
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target) // 从WeakMap获取对象的dep收集器
  if (depsMap === void 0) {
    // never been tracked
    return
  }
  const effects = new Set<ReactiveEffect>()
  const computedRunners = new Set<ReactiveEffect>()
  if (type === TriggerOpTypes.CLEAR) {
    // collection being cleared
    // trigger all effects for target
    // 清除跟踪，运行所有effect
    depsMap.forEach(dep => {
      addRunners(effects, computedRunners, dep)
    })
  } else if (key === 'length' && isArray(target)) {
    // 数组情况，当key为length
    depsMap.forEach((dep, key) => {
      if (key === 'length' || key >= (newValue as number)) {
        addRunners(effects, computedRunners, dep)
      }
    })
  } else {
    // schedule runs for SET | ADD | DELETE
    // 如果操作是修改变量，添加，删除
    if (key !== void 0) {
      addRunners(effects, computedRunners, depsMap.get(key))
    }
    // also run for iteration key on ADD | DELETE | Map.SET
    if (
      type === TriggerOpTypes.ADD ||
      type === TriggerOpTypes.DELETE ||
      (type === TriggerOpTypes.SET && target instanceof Map)
    ) {
      const iterationKey = isArray(target) ? 'length' : ITERATE_KEY
      addRunners(effects, computedRunners, depsMap.get(iterationKey))
    }
  }
  const run = (effect: ReactiveEffect) => {
    scheduleRun(
      effect,
      target,
      type,
      key,
      undefined
    )
  }
  // Important: computed effects must be run first so that computed getters
  // can be invalidated before any normal effects that depend on them are run.
   // 优先运行effect里面有依赖computed对象的那些effect
  // 这样其他普通effect里面依赖computed的对象可以得出正确结果
  computedRunners.forEach(run)
  effects.forEach(run)
}
```
这里面有一个需要注意的地方，就是定义了两个Set： `effects`和`computedRunners`。`effects`好理解，就是调用track时候设置到`deps`里面的`effects`，触发更新后会运行这些`effects`函数来达到去更新vue视图或组件状态或者其他的一些事情。`computedRunners`从名字上可以看出是和`computed`属性有关，主要是处理惰性求值的情况（后面会分析computed和惰性求值的过程）。

在`trigger`函数定义中，可以看到主要有三个触发类型：`CLEAR`,`数组length的修改`,`ADD,DELETE,SET`。`addRunners`函数判断`effectsToAdd`的类型，如果是`computed`生成的`effect`就添加到`computedRunners`，否则就添加到普通`effects`集合中。

```
function addRunners(
  effects: Set<ReactiveEffect>,
  computedRunners: Set<ReactiveEffect>,
  effectsToAdd: Set<ReactiveEffect> | undefined
) {
  if (effectsToAdd !== void 0) {
    effectsToAdd.forEach(effect => {
      if (effect !== activeEffect) {
        // 只有effect不是当前activeEffect才添加到runners队列里
        if (effect.options.computed) {
          computedRunners.add(effect)
        } else {
          effects.add(effect)
        }
      } else {
        // 避免执行effect过程中修改响应式对象自身(foo.value++)造成死循环
        // the effect mutated its own dependency during its execution.
        // this can be caused by operations like foo.value++
        // do not trigger or we end in an infinite loop
      }
    })
  }
}
```
执行`effect`
```

function scheduleRun(
  effect: ReactiveEffect,
  target: object,
  type: TriggerOpTypes,
  key: unknown,
  extraInfo?: DebuggerEventExtraInfo
) {
  if (effect.options.scheduler !== void 0) {
    effect.options.scheduler(effect)
  } else {
    effect()
  }
}
```
如果有`scheduler`（调度器）给`scheduler`去调用`effect`，否则就直接运行`effect`。至此，一个ref由依赖追踪到修改ref的value触发更新运行effect回调的过程就分析完毕了。

### Computed ###
`computed(getter)`方法可以传入一个getter回调，并且返回一个`ref`响应式对象。（getter只有需要使用的时候才会被运行）每当getter回调里的响应式对象更新了，在其他effect里面有依赖computed对象话这个effect会被重新运行来更新计算后的值。

简单的单元测试示例：
```
const refVal = ref(1);
const cValue = computed(() => refVal.value) // cValue也是一个ref对象，此时() => refVal.value还没有被运行
let dummy
effect(() => {
  dummy = cValue.value // 运行() => refVal.value并且将值赋给dummy（1）
})
expect(dummy).toBe(1)
refVal.value = 10 // 将refVal改为10，这个时候上面effect里的回调会再次调用，更新dummy的值
expect(dummy).toBe(10)
```
computed实现：
```
export function computed<T>(
  getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>
) {
  let getter: ComputedGetter<T>
  let setter: ComputedSetter<T>

  if (isFunction(getterOrOptions)) {
    // 当传入的为函数（一般情况下用法）
    getter = getterOrOptions
    setter = __DEV__
      ? () => {
          console.warn('Write operation failed: computed value is readonly')
        }
      : NOOP
  } else {
    getter = getterOrOptions.get
    setter = getterOrOptions.set
  }

  let dirty = true // 标记dirty为true，当dirty是true的时候，运行getter重新获取value，否则直接返回value（上次计算的值）
  let value: T
  let computed: ComputedRef<T>

  const runner = effect(getter, {
    lazy: true,
    // mark effect as computed so that it gets priority during trigger
    computed: true,
    scheduler: () => {
      if (!dirty) {
        dirty = true // 标记dirty为true，下次再访问computed的value重新运行runner获取最新值（惰性求值）
        trigger(computed, TriggerOpTypes.SET, 'value')
      }
    }
  })
  computed = {
    _isRef: true,
    // expose effect so computed can be stopped
    effect: runner,
    get value() {
      if (dirty) {
        value = runner() // 从传入的函数改变为effect后运行求值
        dirty = false
      }
      track(computed, TrackOpTypes.GET, 'value') // 与调用computed的对象建立依赖关系
      return value
    },
    set value(newValue: T) {
      setter(newValue) // 只有定义了setter才能set，否则报错（计算属性不应该被修改）
    }
  } as any

  return computed
}
```
在`computed`函数里，有几个主要的点：
* 传入的getter函数也会处理成一个`effect`，但是这个`effect`是惰性的（不会被立即执行）
* computed的`effect`中有一个`scheduler`，每当computed需要计算时（getter里面的响应式对象发生变化），会先将dirty改为true（下次访问值的时候运行getter获取最新值），然后再通知依赖computed的effect进行更新
* 在effect的`trigger`中，运行在effect中包含`computed`对象的更新回调会优先运行，这样其他普通effect里面依赖computed的值可以得出正确结果（上文`trigger`函数可以再回顾一下）

### reactive对象的创建 ###
刚才分析了通过ref方法对基本变量的跟踪及相应过程。`reactive`函数中，实际上本质是将对象中的每一个键值对应的值都转换为类似`ref`函数中跟踪基本类型用到的`track`和`trigger`，也是一个惰性依赖的过程（访问某个对象的键才会对其进行依赖跟踪）。Vue3.0中改用了`Proxy`作为劫持对象访问及修改其属性的代理对象。相对于vue2.x时代用的`defineproperty`，我认为最重要的一个好处是给一个对象新增属性的时候无需再调用`Vue.set`方法去生命一个响应式对象属性，并且`Proxy`还能代理数组。（为啥Vue2不直接用Proxy？因为要照顾低版本IE，Proxy对象的一些特性没有办法在低版本IE中模拟）

// todo reactvie里面自动unwrap ref分析？

#### Proxy简单用法 ####
`let proxy = new Proxy(target, handler)`
传入的构造参数中，`target`是Proxy需要进行代理的对象。任何需要访问target的操作，都将经过Proxy处理后再返回结果。第二个参数`handler`就是定义具体要代理的行为（最常用的就是定义getter和setter）。

MDN的例子：
```
const handler = {
    get: function(obj, prop) { // 定义一个getter，prop键存在返回prop对应的值，否则返回数值37
        return prop in obj ? obj[prop] : 37;
    }
};

const p = new Proxy({}, handler);
p.a = 1;
p.b = undefined; // b不存在，访问属性b将返回37

console.log(p.a, p.b);      // 1, undefined
console.log('c' in p, p.c); // false, 37
```

#### 有关Reflect对象 ####
Vue在创建`reactive`的过程中，还用到了`Reflect`这个es6的特性。`Reflect`实际上是其他很多语言都有的反射特性，反射的作用最主要就是在程序运行的过程中，能动态获取当前对象的信息。在没有`Reflect`之前，想要获取对象上拥有的属性可以用`for(…in…)`,`Object.prototype.hasOwnProperty`等等方法去取得。为什么还需要`Reflect`呢？简单来讲就是让js更加强大，将所有用到的反射有关操作全都放到`Reflect`这个对象上去。这样就不需要再调用`Object.prototype.hasOwnProperty`等等之类对象上到原型方法，统一使用`Reflect`去操作获取。

先简单看一下在`vue3.0`中用到的`Proxy`常用属性：
* `Reflect.get()` 获取对象身上某个属性的值，类似于 target[name]。. 
* `Reflect.has()` 判断一个对象是否存在某个属性，和 in 运算符 的功能完全相同。
* `Reflect.set()`将值分配给属性的函数。返回一个Boolean，如果更新成功，则返回true。
* `Reflect.deleteProperty()`作为函数的delete操作符，相当于执行 delete target[name]。

**Reactive实现过程**
```
// 入口函数，处理target是只读，或者已经是ref对象的情况
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  if (readonlyToRaw.has(target)) {
    return target
  }
  // target is explicitly marked as readonly by user
  if (readonlyValues.has(target)) {
    return readonly(target)
  }
  if (isRef(target)) {
    return target
  }
  return createReactiveObject(
    target, // 普通对象
    rawToReactive, // 通过原始对象去获取对应的响应式对象
    reactiveToRaw, // 通过响应式对象去获取对应的原始对象
    mutableHandlers, // proxy用来进行操作的handlers，追踪依赖响应的过程都在这里面
    mutableCollectionHandlers // 这个不太常用，暂时不分析
  )
}

// 创建响应式对象函数
function createReactiveObject(
  target: unknown,
  toProxy: WeakMap<any, any>,
  toRaw: WeakMap<any, any>,
  baseHandlers: ProxyHandler<any>,
  collectionHandlers: ProxyHandler<any>
) {
  if (!isObject(target)) {
    if (__DEV__) {
      console.warn(`value cannot be made reactive: ${String(target)}`)
    }
    return target
  }
  // targe已经有定义过Proxy了
  let observed = toProxy.get(target)
  if (observed !== void 0) {
    return observed
  }
  // 传入的target是响应式对象并且有对应的原始对象
  if (toRaw.has(target)) {
    return target
  }
  // only a whitelist of value types can be observed.
  if (!canObserve(target)) {
    return target
  }
  const handlers = collectionTypes.has(target.constructor)
    ? collectionHandlers
    : baseHandlers
  observed = new Proxy(target, handlers)
  toProxy.set(target, observed)
  toRaw.set(observed, target)
  return observed
}
```
主要过程很简单，就是将target传入后将其和对应的handlers设置为Proxy并且返回。一开始两个if的作用主要是防止重复定义响应式对象（比如一个对象的reactive是readonly，再传入reactive防止再次生成响应式）。在这个过程中，没有递归，就是简单的定义一个Proxy。
那么具体的修改对象和追踪依赖的过程在哪呢？就是在这个`mutableHandlers`里面：
```
export const mutableHandlers: ProxyHandler<object> = {
  get, // 调用createGetter
  set, // 调用createSetter
  deleteProperty,
  has,
  ownKeys
}
```
get和set的方法是由`createGetter()`和`createSetter()`生成
```
const get = /*#__PURE__*/ createGetter()
const set = /*#__PURE__*/ createSetter()
```
#### `getter`的实现 ####

```
// 创建proxy对象的getter
function createGetter(isReadonly = false, shallow = false) {
  return function get(target: object, key: string | symbol, receiver: object) {
    // 当target是数组，并且操作'includes', 'indexOf', 'lastIndexOf'这三个方法时候，将会遍历数组内所有元素
    // 所以需要对每个元素设置响应式追踪
    if (isArray(target) && hasOwn(arrayInstrumentations, key)) {
      return Reflect.get(arrayInstrumentations, key, receiver)
    }
    const res = Reflect.get(target, key, receiver) // 先获取target中key的值
    if (isSymbol(key) && builtInSymbols.has(key)) {
      return res
    }
    if (shallow) {
      track(target, TrackOpTypes.GET, key)
      // TODO strict mode that returns a shallow-readonly version of the value
      return res
    }
    // ref unwrapping, only for Objects, not for Arrays.
    if (isRef(res) && !isArray(target)) {
      return res.value
    }
    track(target, TrackOpTypes.GET, key) // 将当前访问的键值设置响应追踪
    // 如果target的key对应值为对象，则对其设置响应式对象
    // 否则就直接返回
    return isObject(res)
      ? isReadonly
        ? // need to lazy access readonly and reactive here to avoid
          // circular dependency
          readonly(res)
        : reactive(res)
      : res
  }
}

const arrayInstrumentations: Record<string, Function> = {}
;['includes', 'indexOf', 'lastIndexOf'].forEach(key => {
  arrayInstrumentations[key] = function(...args: any[]): any {
    const arr = toRaw(this) as any // 获取原始数组
    for (let i = 0, l = (this as any).length; i < l; i++) {
      track(arr, TrackOpTypes.GET, i + '')
    }
    return arr[key](...args.map(toRaw)) // 当在数组上'includes', 'indexOf', 'lastIndexOf'这三个方法时，操作先获取原始数组，再对数组每个元素进行track
    // 最后一行return中args就是'includes', 'indexOf'传入的参数，key是'includes', 'indexOf', 'lastIndexOf'其中之一。这行相当于运行: 数组.key方法(参数)   args.map(toRaw)的作用就是：如果参数是响应式对象，先转换为普通值
  }
})
```
在`Proxy`的`getter`中，步骤可以规纳如下：
* 先对`target`对应的`key`取值并暂存为`res`
* 将`target`和`key`设置响应追踪（`track`函数里的追踪流程和上文中分析`ref`里的一致）
* 返回`target[key]`取值结果`res`，如果`res`是对象，那么也将其设置为`响应式`
* 数组设置的方法也是这个流程，如果target是数组并且操作数组上`'includes', 'indexOf', 'lastIndexOf'`这三个方法中一个，那么先遍历一遍数组并且将每个元素都进行track，再调用目标函数。（当数组被Proxy代理的时候，如果运行`'includes', 'indexOf', 'lastIndexOf'`，Proxy会调用getter给每个元素取值并且返回给方法进行查找）

#### `setter`的实现 ####
```
function createSetter(isReadonly = false, shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    if (isReadonly && LOCKED) {
    // 只读的话抛出警告并直接返回
      return true
    }

    const oldValue = (target as any)[key]
    if (!shallow) {
      value = toRaw(value) 
      if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
    } else {
      // in shallow mode, objects are set as-is regardless of reactive or not
    }

    const hadKey = hasOwn(target, key)
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        // 新增一个属性
        // 这里相比vue2.x，可以监听到对象上属性的增加从而不用再调用this.$set方法设置响应式
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}
```
总体流程也是先用`Reflect.set`设置值，然后调用`trigger`方法更新去通知响应式更新。这里有一个判断，`target === toRaw(receiver)`的作用？
从注释上来看似乎和原型链有关系，来看一个例子：[这里参考了这篇博文](https://bengbu-yuezhang.github.io/2019/11/25/vue%E6%BA%90%E7%A0%81/)
```
const childProxy = new Proxy({
  name: 'child proxy'
}, {
  set(target, key, value, receiver) {
    console.log('childProxy上的set', target, receiver)
    return Reflect.set(target, key, value, receiver)
  }
})

const parentProxy = new Proxy({
  name: 'parent proxy '
}, {
  set(target, key, value, receiver) {
    console.log('parentProxy的set', target, receiver)
    return Reflect.set(target, key, value, receiver)
  }
})

Object.setPrototypeOf(childProxy, parentProxy)
childProxy.newName = 'test2'
// 控制台输出：
// childProxy上的set {name: "test"} Proxy {name: "test"}
// parentProxy的set {name: "parent proxy "} Proxy {name: "test"}
```
从输出可以发现，如果一个对象是原型链，并且它的原型链上也有一个proxy，在进行属性添加的时候，会调用两次set方法。但是这两次set调用中`receiver`的对象是相同的。所以在这种情况下，`target === toRaw(receiver)`的作用就是防止这种情况发生。

### Vue3.0组件源码里的应用 ###
看完了上面的响应式分析，再来回到3.0的组件源码部分。`SetupRenderEffect`这个函数将在完成运行组件`setup`调用后运行。这个`effect`对组件来说非常关键，组件进行初始化，或者需要更新的时候，都会执行这个`effect`，让我们来看一下这个函数：
renderer.ts 1074行左右，effect回调里分为两部分：组件的子组件渲染初始化以及子组件更新。首先来看一下这个函数的主要部分：
```
const setupRenderEffect: SetupRenderEffectFn<HostNode, HostElement> = (
    instance,
    initialVNode,
    container,
    anchor,
    parentSuspense,
    isSVG
  ) => {
    // effect函数将创建响应式回调，依赖收集器，并且立即执行（如果没有传入lazy选项）
    // 并且放入update中，如果有依赖更新了调用update函数
    // 后面会分析update什么时候进行调用
    instance.update = effect(function componentEffect() {
      if (!instance.isMounted) {
       // 组件还没有被挂载，进行初始化
       // 对子组件进行首次patch的相关代码
        instance.isMounted = true
      } else {
        // 组件已被初始化，修改响应式对象后调用effect进行更新与patch的流程
        // 分析组件更新流程的时候再详细解释
         // 对子组件进行更新的相关代码
      }
    }, __DEV__ ? createDevEffectOptions(instance) : prodEffectOptions)
  }
```


再来回到`effect`传入的函数中，看一下第一次加载渲染根组件调用`render`函数的过程：
```
instance.update = effect(function componentEffect() {
      if (!instance.isMounted) {
        // renderComponentRoot将会调用组件上编译好的render函数
        // 调用render期间有依赖响应式对象则会建立依赖关系
        // activeEffect指向当前这个函数回调
        // 创建相应式变量的时候会将activeEffect加入到deps
        const subTree = (instance.subTree = renderComponentRoot(instance)) // subTree就是渲染好的VNode
        // subTree是运行render函数以后返回的vnode
        // beforeMount hook
        if (instance.bm !== null) {
          invokeHooks(instance.bm)
        }
        // 组件还没有被mount，进行第一次patch（映射到dom上）
          patch(
            null,
            subTree,
            container!, // container is only null during hydration
            anchor,
            instance,
            parentSuspense,
            isSVG
          )
          initialVNode.el = subTree.el
         // 一些生命周期的方法...
        instance.isMounted = true
      } else {
        // 组件已被初始化，修改响应式对象后调用effect进行更新。。。
      }
    }, __DEV__ ? createDevEffectOptions(instance) : prodEffectOptions
  }
```
首先第一步就是调用`renderComponentRoot(instance)`，去调用这个组件的`render`函数创建vnode，并且最后返回一个`vnode`作为虚拟dom的根节点（**这也是为什么写模板的时候只允许有一个根节点，在vue内部每个组件只返回一个vnode作为虚拟dom树的根节点**）

为了更好地展示更新流程，这里放一个例子：
```
 
  const myComp = {
    name: 'myCustomComp',
    props: {
      propVal: Object
    },
    setup(props) {
      return () => {
        return h('div', {
          class: 'test'
        }, [
          h('p', 'prop val: ' + JSON.stringify(props.propVal))
        ]);
      }
    }
  };

  createApp({
    setup() {
      let reactiveObj = ref(reactive({
        val2: 'my reactive object'
      }));
      setTimeout(() => {
      // 2000毫秒后更新propVal
        reactiveObj.value = reactive({
          val2: 'another reactive object'
        });
      }, 2000);
      return () => {
        return h('div', {
          class: ['foo', 'bar']
        }, h(myComp, {
          propVal: reactiveObj.value
        }))
      }
    }
  }).mount('#app')
```
为了更清晰展示整个组件更新的流程，这里直接写了render函数。在这个例子里，2000毫秒后将会更新子组件`myCustomComp`的`propVal`，`reactiveObj.value`将会调用内置的`trigger`。首先父组件会再次运行`render`函数，在`propVal`发生变化后，`patch`函数将会调用子组件的`instance.update()`调用`effect`更新子组件状态。
```
instance.update = effect(function componentEffect() {
      if (!instance.isMounted) {
        // 初始化方法
      } else {
        // 组件已被初始化，修改响应式对象后调用effect进行更新。。。
        // This is triggered by mutation of component's own state (next: null)
        // OR parent calling processComponent (next: HostVNode)
        const { next } = instance // next即为父组件进行renderComponentRoot后传入patch的nextTree
        // 如果next是null说明是组件自身的变量修改触发导致的更新
        // 不是null说明是其他组件的调用
        if (__DEV__) {
          pushWarningContext(next || instance.vnode)
        }

        if (next !== null) {
          // 将组件内部实例instance中的vnode进行更新为新的vnode
          updateComponentPreRender(instance, next)
        }
        const nextTree = renderComponentRoot(instance) // 新的VNode树
        const prevTree = instance.subTree // 老的Vnode树
        instance.subTree = nextTree // 更新VNode树
        // beforeUpdate hook
        if (instance.bu !== null) {
          invokeHooks(instance.bu)
        }
        // reset refs
        // only needed if previous patch had refs
        if (instance.refs !== EMPTY_OBJ) {
          instance.refs = {}
        }
        patch( // 如果传入子组件的prop或者class，style等发生变化，将调用子组件内部实例上的update()方法
          prevTree, // 上一次渲染的vnode树
          nextTree, // 当前更新后渲染的vnode树(如果子组件的props有更新，patch处理processComponent的时候会进行normal update（一般情况下）)
          // parent may have changed if it's in a portal
          hostParentNode(prevTree.el as HostNode) as HostElement,
          // anchor may have changed if it's in a fragment
          getNextHostNode(prevTree),
          instance,
          parentSuspense,
          isSVG
        )
        instance.vnode.el = nextTree.el
        if (next === null) {
          // self-triggered update. In case of HOC, update parent component
          // vnode el. HOC is indicated by parent instance's subTree pointing
          // to child component's vnode
          updateHOCHostEl(instance, nextTree.el)
        }
        // updated hook
        if (instance.u !== null) {
          queuePostRenderEffect(instance.u, parentSuspense)
        }

        if (__DEV__) {
          popWarningContext()
        }
      }
    }, __DEV__ ? createDevEffectOptions(instance) : prodEffectOptions
  }
```
