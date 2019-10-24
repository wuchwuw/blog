## reactivity之reactive篇

reactive的用法大致与Vue2.x的Vue.observable()相同，都是创建一个响应式对象，不过在Vue3.x中reactive是基于Proxy实现的，通过Proxy拦截对象的访问和赋值操作来收集依赖和派发更新。

    import { reactive } from 'vue'

    // 将state转化成响应式对象
    const state = reactive({
      count: 0
    })

### 主要api

reactive: 创建响应式对象
isReactive: 判断对象是不是响应式对象
readonly: 创建只读的响应式对象
isReadonly: 判断对象是不是只读的响应式对象
toRaw: 传入响应式对象，返回它的原对象
markReadonly: 将对象标记为只读的
markNonReactive: 将对象标记为不可响应式的

### 源码

在介绍源码之前，先要了解reactive中几个重要的Map、Set和方法

    // targetMap存储着一个响应式对象的所有依赖
    // 它的存储关系为target -> key -> dep
    // 也就是说每一个响应式对象创建时都会在targetMap中创建一个KeyToDepMap
    // KeyToDepMap上的key对应着对象的key，value对应的一个用来储存该键值
    // 所有dep的set，set里面存储的类型是ReactiveEffect后面会介绍
    export type Dep = Set<ReactiveEffect>
    export type KeyToDepMap = Map<any, Dep>
    export const targetMap = new WeakMap<any, KeyToDepMap>()

    // 4个weakMap分别是原生对象以及它们的响应式对象的映射
    const rawToReactive = new WeakMap<any, any>()
    const reactiveToRaw = new WeakMap<any, any>()
    const rawToReadonly = new WeakMap<any, any>()
    const readonlyToRaw = new WeakMap<any, any>()

    // readonlyValues存储着标记为只读的对象
    const readonlyValues = new WeakSet<any>()
    // nonReactiveValues存储着标记为不可响应式的对象
    const nonReactiveValues = new WeakSet<any>()

    // 所有可以创建响应式的数据类型
    const isObservableType = /*#__PURE__*/ makeMap(
      ['Object', 'Array', 'Map', 'Set', 'WeakMap', 'WeakSet']
        .map(t => `[object ${t}]`)
        .join(',')
    )

    // 满足以下条件才可以创建响应式对象
    // 1、不是一个vue实例
    // 2、不是一个虚拟Dom节点
    // 3、原对象是['Object', 'Array', 'Map', 'Set', 'WeakMap', 'WeakSet']中的一种
    // 4、没有标记为不可响应式的对象
    const canObserve = (value: any): boolean => {
      return (
        !value._isVue &&
        !value._isVNode &&
        isObservableType(toTypeString(value)) &&
        !nonReactiveValues.has(value)
      )
    }

创建响应式对象的两个api，reactive、readonly

    export function reactive(target: object) {
      // if trying to observe a readonly proxy, return the readonly version.
      // 如果target是一个只读的响应式对象，直接返回target
      if (readonlyToRaw.has(target)) {
        return target
      }
      // target is explicitly marked as readonly by user
      // 如果target被用户手动标记为只读，则调用readonly来创建
      if (readonlyValues.has(target)) {
        return readonly(target)
      }
      return createReactiveObject(
        target,
        rawToReactive,
        reactiveToRaw,
        mutableHandlers,
        mutableCollectionHandlers
      )
    }

    export function readonly<T extends object>(
      target: T
    ): Readonly<UnwrapNestedRefs<T>> {
      // value is a mutable observable, retrieve its original and return
      // a readonly version.
      // 如果target已经是非只读的响应式对象，那么拿到它的原对象
      // 去创建它的readonly版本
      if (reactiveToRaw.has(target)) {
        target = reactiveToRaw.get(target)
      }
      return createReactiveObject(
        target,
        rawToReadonly,
        readonlyToRaw,
        readonlyHandlers,
        readonlyCollectionHandlers
      )
    }

    // 可以看到，不管是reactive或者readonly都是调用createReactiveObject来创建
    // 响应式对象，关键是传入的参数不同，因为传入的几个全局变量一开始都介绍了这里就不重复了
    // 参数这里没介绍的是baseHandlers和collectionHandlers，是创建Proxy的关键
    function createReactiveObject(
      target: unknown,
      toProxy: WeakMap<any, any>,
      toRaw: WeakMap<any, any>,
      baseHandlers: ProxyHandler<any>,
      collectionHandlers: ProxyHandler<any>
    ) {
      // 如果target吧
      if (!isObject(target)) {
        if (__DEV__) {
          console.warn(`value cannot be made reactive: ${String(target)}`)
        }
        return target
      }
      // target already has corresponding Proxy
      // 传入的target已经创建过proyx，直接返回target对应的proxy
      let observed = toProxy.get(target)
      if (observed !== void 0) {
        return observed
      }
      // target is already a Proxy
      // 传入的target本身是一个proxy对象，直接返回
      if (toRaw.has(target)) {
        return target
      }
      // only a whitelist of value types can be observed.
      // 检查target能不能创建响应式对象，这个方法之前分析过了
      if (!canObserve(target)) {
        return target
      }
      // collectionTypes => Set, Map, WeakMap, WeakSet
      // 根据target属于collectionTypes中哪的一种来决定传入的是
      // baseHandlers还是collectionHandlers
      const handlers = collectionTypes.has(target.constructor)
        ? collectionHandlers
        : baseHandlers
      observed = new Proxy(target, handlers)
      // 缓存创建完成的proxy对象
      toProxy.set(target, observed)
      toRaw.set(observed, target)
      // 为原对象创建KeyToDepMap用来保存依赖
      // 之前已经分析过targetMap这个全局变量
      if (!targetMap.has(target)) {
        targetMap.set(target, new Map())
      }
      return observed
    }

可以看到创建Proxy对象的过程是很简单的，最主要的是根据只读和非只读以及是否属于collectionTypes来传入不同的Handlers。

### reactive mutableHandlers

    // 遍历Symbol对象上所有的内置Symbol值
    const builtInSymbols = new Set(
      Object.getOwnPropertyNames(Symbol)
        .map(key => (Symbol as any)[key])
        .filter(isSymbol)
    )

    function createGetter(isReadonly: boolean) {
      return function get(target: object, key: string | symbol, receiver: object) {
        // Reflect.get 与 target[key]的作用相同
        // 它们的区别是当key设置了getter，则调用getter时绑定的this值是proxy对象
        // 而target[key]调用getter绑定的this是target
        const res = Reflect.get(target, key, receiver)
        // 如果key是内置方法则直接返回该方法
        if (isSymbol(key) && builtInSymbols.has(key)) {
          return res
        }
        // 如果res是Ref类型，则返回它的value值，这也是为什么Ref嵌套在响应式对象中是默认展开的
        if (isRef(res)) {
          return res.value
        }
        // 调用track方法收集依赖，后面effect篇会分析
        track(target, OperationTypes.GET, key)
        // 当访问的值也是一个对象时，递归的创建响应式对象，等到访问的时候才做这个判断的好处是
        // 不需要额外的逻辑来处理循环引用，因为创建过的响应式对象已经缓存在对应的全局变量中
        return isObject(res)
          ? isReadonly
            ? // need to lazy access readonly and reactive here to avoid
              // circular dependency
              readonly(res)
            : reactive(res)
          : res
      }
    }

    function set(
      target: object,
      key: string | symbol,
      value: unknown,
      receiver: object
    ): boolean {
      // 如果旧值是Ref类型，而新值不是，那么只需要更新旧值的value属性
      value = toRaw(value)
      const oldValue = (target as any)[key]
      if (isRef(oldValue) && !isRef(value)) {
        oldValue.value = value
        return true
      }
      // 用hadKey来区分这次操作是设置属性还是添加属性
      const hadKey = hasOwn(target, key)
      // Reflect.set 与 target[key] = value 的作用相同
      // 它们的区别是当key设置了setter，则调用setter时绑定的this值是proxy对象
      // 而target[key]调用setter绑定的this是target
      const result = Reflect.set(target, key, value, receiver)
      // don't trigger if target is something up in the prototype chain of original
      if (target === toRaw(receiver)) {
        /* istanbul ignore else */
        if (__DEV__) {
          const extraInfo = { oldValue, newValue: value }
          if (!hadKey) {
            trigger(target, OperationTypes.ADD, key, extraInfo)
          } else if (hasChanged(value, oldValue)) {
            trigger(target, OperationTypes.SET, key, extraInfo)
          }
        } else {
          // 根据hadKey传入不同的参数，调用trigger派发更新，后面effect篇会分析
          if (!hadKey) {
            trigger(target, OperationTypes.ADD, key)
          } else if (hasChanged(value, oldValue)) {
            trigger(target, OperationTypes.SET, key)
          }
        }
      }
      return result
    }

    function deleteProperty(target: object, key: string | symbol): boolean {
      const hadKey = hasOwn(target, key)
      const oldValue = (target as any)[key]
      // 删除key值，删除成功则返回true
      const result = Reflect.deleteProperty(target, key)
      if (result && hadKey) {
        /* istanbul ignore else */
        if (__DEV__) {
          trigger(target, OperationTypes.DELETE, key, { oldValue })
        } else {
          // 调用trigger派发更新，传入type为'delete'
          trigger(target, OperationTypes.DELETE, key)
        }
      }
      return result
    }

    function has(target: object, key: string | symbol): boolean {
      // 拦截属性查询操作
      // 调用track收集依赖
      const result = Reflect.has(target, key)
      track(target, OperationTypes.HAS, key)
      return result
    }

    function ownKeys(target: object): (string | number | symbol)[] {
      // 拦截Object.getOwnPropertyNames 和Object.getOwnPropertySymbols操作
      // 调用track收集依赖
      track(target, OperationTypes.ITERATE)
      return Reflect.ownKeys(target)
    }

### readonly readonlyHandlers

    // readonlyHandlers比较简单，可以看到get、has、ownKeys等拦截方法都跟mutableHandlers一样
    // 而set、deleteProperty多加了一层判断、通过全局变量LOCKED来决定是否可以执行set和delete操作
    // 而Vue3.x对外并没有暴露更改LOCKED变量的方法
    export const readonlyHandlers: ProxyHandler<object> = {
      get: createGetter(true),

      set(
        target: object,
        key: string | symbol,
        value: unknown,
        receiver: object
      ): boolean {
        if (LOCKED) {
          if (__DEV__) {
            console.warn(
              `Set operation on key "${String(key)}" failed: target is readonly.`,
              target
            )
          }
          return true
        } else {
          return set(target, key, value, receiver)
        }
      },

      deleteProperty(target: object, key: string | symbol): boolean {
        if (LOCKED) {
          if (__DEV__) {
            console.warn(
              `Delete operation on key "${String(
                key
              )}" failed: target is readonly.`,
              target
            )
          }
          return true
        } else {
          return deleteProperty(target, key)
        }
      },

      has,
      ownKeys
    }
