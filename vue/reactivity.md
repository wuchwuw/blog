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

    // todo

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