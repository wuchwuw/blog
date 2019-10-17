## reactivity

在Vue3.x中，负责响应式的代码主要放在reactivity包中。本文主要是介绍reactivity暴露的几个主要的API，并从源码的角度来分析Vue3.x的响应式原理。

### reactivity的主要API

从reactivity包的入口来看，reactivity暴露的主要api有reactive、computed、ref。

#### reactive

reactive的用法大致与Vue2.x的Vue.observable()相同，通过reactive将普通的对象转化为Vue中的响应式对象，只不过在Vue3.x中响应式对象是基于Proxy来实现的。

    import { reactive } from 'vue'

    // 将state转化成响应式对象
    const state = reactive({
      count: 0
    })

#### computed

在Vue3.x中，当某些state的值依赖于其他state时，可以使用computed API来创建一个计算属性，前提是计算属性依赖的值必须是响应式的。

    import { reactive, computed } from 'vue'

    const state = reactive({
      count: 0
    })

    const double = computed(() => state.count * 2)

#### ref

ref是Vue3.x中引入的新概念，通过ref API同样可以创建一个响应式的数据。那么它跟reactive有什么区别呢？我们先看一下Ref的类型定义:

    export interface Ref<T = any> {
      [refSymbol]: true
      value: UnwrapRef<T>
    }

    export type UnwrapRef<T> = {
      ref: T extends Ref<infer V> ? UnwrapRef<V> : T
      array: T extends Array<infer V> ? Array<UnwrapRef<V>> : T
      object: { [K in keyof T]: UnwrapRef<T[K]> }
      stop: T
    }[T extends Ref
      ? 'ref'
      : T extends Array<any>
        ? 'array'
        : T extends BailTypes
          ? 'stop' // bail out on types that shouldn't be unwrapped
          : T extends object ? 'object' : 'stop']

很明显Ref类型的数据就是一个包含了一个Symbol值和value属性的对象。同时value的值是UnwrapRef类型，从UnwrapRef的类型定义来看，value值是可以嵌套其他Ref类型的值的。也就说通过ref()创建的响应式数据是这样的：

    const aRef = ref(0)

    aRef  => { refSymbol: true, value: 0 }

相比于reactive，ref这个API更多用来将一个基础值转化成响应式数据，ref会将传入的基础值包装成一个Ref类型的对象，并把值存到对象的value属性上。但是如果仅仅是用来转化基础类型，我们同样也可以简单的通过reactive来达到同样的目的:

    const a = reactive({ value: 0 })

事实上在阅读了reactivity的源码和官方对Vue Composition API的介绍后，可以看出Ref类型的存在的意义在于，在响应式数据赋值传递的过程中，维持它响应式的特性。例如：

    // 通过computed创建一个计算属性
    let a = computed(() => state.b * 2)

    const a = reactive({ b: 1 })
    
    // 失去响应式特性
    const { b } = a
    // 保持响应式特性
    const { b } = toRefs(a)

从上面的例子可以看到，假如创建的计算属性仅仅是a = state.b * 2，a只是一个基础类型number，那么当state.b更新时，a没办法根据它的依赖更新自身的值。另一方面对象解构赋值的时候也会失去引用，导致b失去响应式数据的特性，这里官方也建议通过toRefs将a中的所有属性转化为Ref类型。所以为了维持响应式，需要有Ref类型的存在，因为在js中对象的赋值是通过引用来传递的。

    // 从源码来看computed函数的返回类型就是Ref
    export function computed<T>(getter: () => T): ComputedRef<T>

来看看另一个例子:

    // todo

### 源码

#### reactive

    // packages/reactivity/reactive.ts

    export function reactive(target: object) {
      // 如果传入的target已经是响应式的，并且是readonly的直接返回
      if (readonlyToRaw.has(target)) {
        return target
      }
      // 如果传入的target已经被标记为readonly，直接返回
      if (readonlyValues.has(target)) {
        return readonly(target)
      }
      return createReactiveObject(
        target,
        rawToReactive,     // 已创建的proxy对象的缓存
        reactiveToRaw,     // 已创建的reactive对象的缓存
        mutableHandlers,   // 创建普通proxy对象所需要的handlers
        mutableCollectionHandlers // 如果target是 Set, Map, WeakMap, WeakSet中的一种，使用collectionHandlers
      )
    }

    function createReactiveObject(
      target: any,
      toProxy: WeakMap<any, any>,   // 原生对象为key，proxy对象为value的weakmap
      toRaw: WeakMap<any, any>,     // 原生对象为proxy，原生对象为value的weakmap
      baseHandlers: ProxyHandler<any>,
      collectionHandlers: ProxyHandler<any>
    ) {
      if (!isObject(target)) {
        if (__DEV__) {
          console.warn(`value cannot be made reactive: ${String(target)}`)
        }
        return target
      }
      // 传入的target已经创建过proyx，直接返回target对应的proxy
      let observed = toProxy.get(target)
      if (observed !== void 0) {
        return observed
      }
      // 传入的target已经是一个proxy对象，直接返回
      if (toRaw.has(target)) {
        return target
      }
      // 检查target能不能成为响应式对象，如果target是vue实例或者是vnode节点都是不能的。
      if (!canObserve(target)) {
        return target
      }

      // 这里判断target是不是Set, Map, WeakMap, WeakSet中的一种如果是，使用collectionHandlers
      // 否则使用baseHandlers
      const handlers = collectionTypes.has(target.constructor)
        ? collectionHandlers
        : baseHandlers
      // 根据handlers创建proxy对象
      observed = new Proxy(target, handlers)

      // 创建后缓存到rawToReactive，reactiveToRaw中，避免重复创建
      toProxy.set(target, observed)
      toRaw.set(observed, target)

      // 为每一个响应式对象创建一个map，用来保存依赖，之后会分析
      if (!targetMap.has(target)) {
        targetMap.set(target, new Map())
      }
      return observed
    }

以上是创建响应式对象的过程，其实很简单，就是为传入的对象创建一个Proxy，并且添加到缓存中。接下来我们看一下创建Proxy的handlers，这里主要分析普通对象的handers。

    // packages/reactivity/baseHandlers.ts

    export const mutableHandlers: ProxyHandler<any> = {
      get: createGetter(false),
      set,
      deleteProperty,
      has,
      ownKeys
    }

    function createGetter(isReadonly: boolean) {
      return function get(target: any, key: string | symbol, receiver: any) {
        // 通过Reflect.get拿到对应key的值
        const res = Reflect.get(target, key, receiver)
        // 如果key是一个symbol，则直接返回res
        if (isSymbol(key) && builtInSymbols.has(key)) {
          return res
        }
        // 如果res是Ref类型，则直接返回它的value值
        if (isRef(res)) {
          return res.value
        }
        // 通过track方法为当前的key值收集依赖
        track(target, OperationTypes.GET, key)
        // 如果res还是一个对象，则递归创建res的响应式对象，这里通过传入的isReadonly
        // 来决定是调用readonly还是reactive，这两个都是创建响应式对象的方法
        // 通过readonly创建的响应式对象不能修改它自身的属性
        return isObject(res)
          ? isReadonly
            ? readonly(res)
            : reactive(res)
          : res
      }
    }

    function set(
      target: any,
      key: string | symbol,
      value: any,
      receiver: any
    ): boolean {
      // 如果新设置的值是一个响应式对象，通过toRaw方法拿到缓存中的原生对象
      value = toRaw(value)
      // 拿到旧的value
      const oldValue = target[key]
      // 如果旧值是Ref类型并且新值不是，则更新旧值的value值。
      if (isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
      // 通过hadKey来判断这次set操作是添加新属性还是更改已有的属性
      const hadKey = hasOwn(target, key)
      // 通过Reflect.set来赋值
      const result = Reflect.set(target, key, value, receiver)
      // don't trigger if target is something up in the prototype chain of original
      if (target === toRaw(receiver)) {
        /* istanbul ignore else */

        // 调用trigger方法，更新所有依赖
        if (__DEV__) {
          const extraInfo = { oldValue, newValue: value }
          if (!hadKey) {
            trigger(target, OperationTypes.ADD, key, extraInfo)
          } else if (value !== oldValue) {
            trigger(target, OperationTypes.SET, key, extraInfo)
          }
        } else {
          if (!hadKey) {
            trigger(target, OperationTypes.ADD, key)
          } else if (value !== oldValue) {
            trigger(target, OperationTypes.SET, key)
          }
        }
      }
      return result
    }

响应式对象最重要的功能就是在访问它的时候能收集依赖，在更改它的值的时候能对所有收集到依赖进行派发更新。
可以看到，在访问响应式对象的时候会触发Proxy的get方法，并调用track方法收集依赖。

    export function track(
      target: any,
      type: OperationTypes,
      key?: string | symbol
    ) {
      if (!shouldTrack) {
        return
      }
      // 拿到栈中的最后一个effect，可以简单的理解为当前key的访问发生在哪个环境中
      // 比如在执行计算属性传入的回调前，会将创建计算属性生成的effect添加到栈中
      // 那么在执行回调的过程中，会对依赖的响应式对象进行访问，此时在这个地方拿到的
      // 就是computed生成的effetc，所以当响应式对象更改时，会对computed进行更新
      // 后面介绍computed会再详细介绍
      const effect = activeReactiveEffectStack[activeReactiveEffectStack.length - 1]
      if (!effect) {
        return
      }
      if (type === OperationTypes.ITERATE) {
        key = ITERATE_KEY
      }
      // 创建depsMap，并保存到全局的targetMap中,
      // map的key对应响应式对象的key，
      // map的value是一个set，用来保存对应key依赖的所有effect
      // 这个set也就是Vue2.x中的Dep
      let depsMap = targetMap.get(target)
      if (depsMap === void 0) {
        targetMap.set(target, (depsMap = new Map()))
      }
      let dep = depsMap.get(key!)
      if (dep === void 0) {
        depsMap.set(key!, (dep = new Set()))
      }
      // 判断set中是否已经存在当前effect
      // 如果不存在，则添加进set中
      // 并把set保存到effect的deps属性上
      if (!dep.has(effect)) {
        dep.add(effect)
        effect.deps.push(dep)
        if (__DEV__ && effect.onTrack) {
          effect.onTrack({
            effect,
            target,
            type,
            key
          })
        }
      }
    }

在更改响应式对象时，会触发Proxy的set方法，执行set的过程中调用trigger方法来派发更新。

    export function trigger(
      target: any,
      type: OperationTypes,
      key?: string | symbol,
      extraInfo?: any
    ) {
      // 通过targetMap拿到当前响应式对象依赖的effect
      const depsMap = targetMap.get(target)
      if (depsMap === void 0) {
        // never been tracked
        return
      }
      const effects = new Set<ReactiveEffect>()
      const computedRunners = new Set<ReactiveEffect>()

      // 根据不同的操作，来决定哪些effect需要派发更新
      // 通过addRunners方法区分computedEffect或者普通effect
      // 并添加到创建好的set中
      if (type === OperationTypes.CLEAR) {
        // 如果是clear操作，则拿到所有的effect
        depsMap.forEach(dep => {
          addRunners(effects, computedRunners, dep)
        })
      } else {
        // 如果key值存在，则拿到当前key的所有deps
        if (key !== void 0) {
          addRunners(effects, computedRunners, depsMap.get(key))
        }
        // 如果当前对象是可迭代的，那么在进行添加或者删除操作的时候，会改变可迭代对象
        // 的长度，这时也应该对依赖数组的length属性，或者是可迭代对象的Symbol('iterate')属性
        // 的effect进行更新
        if (type === OperationTypes.ADD || type === OperationTypes.DELETE) {
          const iterationKey = Array.isArray(target) ? 'length' : ITERATE_KEY
          addRunners(effects, computedRunners, depsMap.get(iterationKey))
        }
      }
      // 创建迭代方法，并执行所有已添加的effect
      const run = (effect: ReactiveEffect) => {
        scheduleRun(effect, target, type, key, extraInfo)
      }
      // Important: computed effects must be run first so that computed getters
      // can be invalidated before any normal effects that depend on them are run.
      computedRunners.forEach(run)
      effects.forEach(run)
    }

    function scheduleRun(
      effect: ReactiveEffect,
      target: any,
      type: OperationTypes,
      key: string | symbol | undefined,
      extraInfo: any
    ) {
      if (__DEV__ && effect.onTrigger) {
        effect.onTrigger(
          extend(
            {
              effect,
              target,
              key,
              type
            },
            extraInfo
          )
        )
      }
      // 如果effect上的scheduler存在，
      // 则执行scheduler方法，否则直接执行effect
      if (effect.scheduler !== void 0) {
        effect.scheduler(effect)
      } else {
        effect()
      }
    }

在Vue3.x中，effect是整个响应式的核心，不管是computed、watch、或者模板的更新，都跟effect有关，接下来介绍的计算属性中，会涉及effect的创建过程，以及计算属性是如何在它的依赖值改变时进行更新。

#### computed

    export function computed<T>(
      getterOrOptions: (() => T) | WritableComputedOptions<T>
    ): any {
      // 判断传入的参数是不是函数，如果是则直接赋值给getter，
      // 否则把参数的get方法赋值给getter，setter同理
      const isReadonly = isFunction(getterOrOptions)
      const getter = isReadonly
        ? (getterOrOptions as (() => T))
        : (getterOrOptions as WritableComputedOptions<T>).get
      const setter = isReadonly
        ? __DEV__
          ? () => {
              console.warn('Write operation failed: computed value is readonly')
            }
          : NOOP
        : (getterOrOptions as WritableComputedOptions<T>).set

      // 通过dirty值来决定是否要更新value的值
      let dirty = true
      let value: T

      // 创建effect对象，传入getter函数和options
      const runner = effect(getter, {
        lazy: true,
        // mark effect as computed so that it gets priority during trigger
        computed: true,
        scheduler: () => {
          dirty = true
        }
      })
      // 返回ComputedRef类型的对象
      return {
        [refSymbol]: true,
        // expose effect so computed can be stopped
        effect: runner,
        // 访问computed的值时，根据dirty来决定是否执行runner重新计算value
        get value() {
          if (dirty) {
            value = runner()
            dirty = false
          }
          // When computed effects are accessed in a parent effect, the parent
          // should track all the dependencies the computed property has tracked.
          // This should also apply for chained computed properties.
          trackChildRun(runner)
          return value
        },
        set value(newValue: T) {
          setter(newValue)
        }
      }
    }

创建computed的过程也很简单，可以看出computed函数最后返回的是一个ComputedRef类型的对象，并且将值存储在value属性上，通过get、set方法来控制value属性的值。这样有一个好处就是computed的值并不需要在创建或者依赖更新的时候立马计算，而是可以等到需要访问computed的值的时候，再通过dirty来决定是不是重新计算computed的值，接下来继续分析runner的创建。

    export function effect<T = any>(
      fn: () => T,
      options: ReactiveEffectOptions = EMPTY_OBJ
    ): ReactiveEffect<T> {
      if (isEffect(fn)) {
        fn = fn.raw
      }
      // 通过之前传入的getter，以及options来创建effect
      const effect = createReactiveEffect(fn, options)
      if (!options.lazy) {
        effect()
      }
      return effect
    }

    function createReactiveEffect<T = any>(
      fn: () => T,
      options: ReactiveEffectOptions
    ): ReactiveEffect<T> {
      // 创建effect，可以看到effect其实是一个函数，返回了run函数的执行结果
      // 并且在effect上挂载了一些属性和方法
      const effect = function reactiveEffect(...args: any[]): any {
        return run(effect, fn, args)
      } as ReactiveEffect
      effect[effectSymbol] = true
      effect.active = true
      effect.raw = fn
      effect.scheduler = options.scheduler
      effect.onTrack = options.onTrack
      effect.onTrigger = options.onTrigger
      effect.onStop = options.onStop
      effect.computed = options.computed
      effect.deps = []
      return effect
    }

分析完computed的创建过程，本文将通过以下的例子，来介绍计算属性更新的流程:

    import { computed, reactive } from 'vue'

    const state = reacvite({ count: 1 })
    const computedState = computed(() => state.count * 2)

    console.log(computedState.vaule)  // 1
    state = 3                         // 2
    console.log(computedState.vaule)  // 3

在创建完计算属性computedState，我们在步骤1访问它的value值，此时会触发get方法

    get value() {
      if (dirty) {
        value = runner()
        dirty = false
      }
      ...
    }

因为dirty的初始值为true，所以会执行runner函数计算value的值，也就是执行effect函数，而在effect函数里，返回的是run函数的执行结果

    const effect = function reactiveEffect(...args: any[]): any {
      return run(effect, fn, args)
    } as ReactiveEffect

    function run(effect: ReactiveEffect, fn: Function, args: any[]): any {
    if (!effect.active) {
      return fn(...args)
    }
    if (!activeReactiveEffectStack.includes(effect)) {
      cleanup(effect)
      try {
        // 在执行 () => state.count * 2 之前将当前effect
        // push到activeReactiveEffectStack中
        activeReactiveEffectStack.push(effect)
        return fn(...args)
      } finally {
        activeReactiveEffectStack.pop()
      }
    }
  }

执行fn(...args)也就是传入的() => state.count * 2函数时，会访问state.count的值，此时会触发Proyx上的
get方法，根据我们之前对track方法的分析:

    // 这里拿到的就是计算属性的effect，并保存到key值为count的dep中，完成依赖收集
    // 并且返回state.count * 2的计算结果
    const effect = activeReactiveEffectStack[activeReactiveEffectStack.length - 1]

步骤2中，我们通过修改state.count的值触发Proxy的set方法，并调用trigger方法来派发更新，在trigger中
拿到key值为count的dep，并通过scheduleRun来执行dep上保存的effect函数，由于我们创建computedEffect的时候，传入的options上的存在scheduler，所以会执行effect上的scheduler方法，来更改dirty的值。

    // 创建computed effect
    const runner = effect(getter, {
      lazy: true,
      // mark effect as computed so that it gets priority during trigger
      computed: true,
      scheduler: () => {
        dirty = true
      }
    })


    // scheduleRun方法中执行
    if (effect.scheduler !== void 0) {
      effect.scheduler(effect)
    } else {
      effect()
    }

这里也可以印证我们之前提到的细节，就是更改computed的依赖值并不会马上重新计算，而是将dirty设置为true。
等待computed被再次访问。