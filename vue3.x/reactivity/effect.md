## reactivity之effect篇

前两篇文章都是在介绍响应式对象创建相关的东西。那么作为一个响应式对象，它应该拥有
收集依赖以及派发更新的能力，这里可以回顾一下前面的文章，ref、reactive的源码中
都调用了track和trigger方法来收集依赖和派发更新，这也是本文介绍的重点。在介绍
这两个方法之前，我们先了解一下effect，它是响应式数据实现依赖收集和派发更新的媒介，也是watch、cpmputed、模板更新的底层实现。

### effect

    const a = reactive({ count: 0 })
    let dummy
    effect(() => {
      dummy = a.count
    })

effect可以单纯的作为一个api来使用，当创建effect时传入的函数中的响应式数据改变时，传入的函数会再次执行，这里先不考虑它是怎么实现的，我们先从effect的类型定义来看看它是什么东西。

    // effect api返回的是ReactiveEffect类型
    export interface ReactiveEffect<T = any> {
      // 可以看到返回的effect本身是一个函数
      // 并且函数上面挂载了很多的属性
      (): T
      // 通过symbol表名它是一个effect
      [effectSymbol]: true
      // 一个可用的effect
      active: boolean
      // raw保存着传入effect()时的函数
      raw: () => T
      // 用来存放依赖
      deps: Array<Dep>
      // 是否为一个计算属性的effect
      computed?: boolean
      // 在派发更新时如果scheduler存在，优先执行scheduler
      scheduler?: (run: Function) => void
      // dev相关
      onTrack?: (event: DebuggerEvent) => void
      // dev相关
      onTrigger?: (event: DebuggerEvent) => void
      // 当effect被停止时调用此方法
      onStop?: () => void
    }

    // 创建effect时可传入的配置
    // 这里主要看lazy，它决定在创建时是否立即执行一次effect
    export interface ReactiveEffectOptions {
      lazy?: boolean
      computed?: boolean
      scheduler?: (run: Function) => void
      onTrack?: (event: DebuggerEvent) => void
      onTrigger?: (event: DebuggerEvent) => void
      onStop?: () => void
    }

    export function effect<T = any>(
      fn: () => T,
      options: ReactiveEffectOptions = EMPTY_OBJ
    ): ReactiveEffect<T> {
      if (isEffect(fn)) {
        fn = fn.raw
      }
      const effect = createReactiveEffect(fn, options)
      if (!options.lazy) {
        effect()
      }
      return effect
    }