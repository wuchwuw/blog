## reactivity之effect篇

前两篇文章都是在介绍响应式对象创建相关的东西。那么作为一个响应式对象，它应该拥有收集依赖以及派发更新的能力，这里可以回顾一下前面的文章，ref、reactive的源码中都调用了track和trigger方法来收集依赖和派发更新，这也是本文介绍的重点。在介绍这两个方法之前，我们先了解一下effect，它是响应式数据实现依赖收集和派发更新的媒介，也是watch、cpmputed、模板更新的底层实现。

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
      // 如果传入的函数已经是一个effect
      // 那么获取raw上保存的原函数，重新创建effect
      if (isEffect(fn)) {
        fn = fn.raw
      }
      // 调用createReactive创建effect
      const effect = createReactiveEffect(fn, options)
      // 如果没有设置lazy为true，则默认先执行一次effect
      if (!options.lazy) {
        effect()
      }
      return effect
    }

    // createReactiveEffect也很简单，当调用effect时返回了run方法的执行结果
    // 并且在effect上挂载了之前介绍的一些属性
    function createReactiveEffect<T = any>(
      fn: () => T,
      options: ReactiveEffectOptions
    ): ReactiveEffect<T> {
      const effect = function reactiveEffect(...args: unknown[]): unknown {
        return run(effect, fn, args)
      } as ReactiveEffect
      effect._isEffect = true
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

分析到这里，我们介绍了effect的类型以及它的创建过程，其实还是很简单的也没有什么难点，接下来我们通过一个例子继续分析effect与响应式数据之间的联系，这里涉及到之前一直没有介绍的track和trigger方法。在分析的过程中会有几个比较绕的点，需要配合effect的test case来分析：

    const a = reactive({ count: 0 })
    let dummy
    effect(() => {
      dummy = a.count
    })
    a.count = 2

根据我们上面分析过的，当调用effect api来创建一个effect时，如果lazy不为true，那么它会先执行一次，也就是执行在createReactiveEffect中创建的effect方法。

    const effect = function reactiveEffect(...args: unknown[]): unknown {
      return run(effect, fn, args)
    } as ReactiveEffect

    
    function run(effect: ReactiveEffect, fn: Function, args: unknown[]): unknown {
      // 当effect的激活状态为false时，直接返回我们传入函数的执行结果
      if (!effect.active) {
        return fn(...args)
      }
      // effectStack是一个全局的effect栈
      // 判断当前的effect是否在栈中，如果存在就什么都不做
      // 有一种情况就是在fn中修改了依赖的响应式对象时，会出现循环，所以每次执行前先判断一下
      // 可以避免这种情况
      if (!effectStack.includes(effect)) {
        // 每次执行传入的函数fn前，先实行cleanup方法清除依赖，后面会分析它的作用
        cleanup(effect)
        try {
          // 将effect push进栈中并执行fn
          effectStack.push(effect)
          return fn(...args)
        } finally {
          // 执行完毕后将当前的effect出栈
          effectStack.pop()
        }
      }
    }

在执行传入的函数fn时，会访问到的函数里的响应式对象，在例子中为a.count，并触发Proxy的get方法，之前get方法我们已经在reactive篇分析过了，这里主要分析执行get方法时调用的track方法来收集依赖。

    export function track(target: object, type: OperationTypes, key?: unknown) {
      if (!shouldTrack || effectStack.length === 0) {
        return
      }
      // 拿到effectStack栈中最后一个effect，那么这里拿到的就是执行fn之前
      // push到栈中的effect，也就是我们执行fn之前添加的effect
      const effect = effectStack[effectStack.length - 1]
      if (type === OperationTypes.ITERATE) {
        key = ITERATE_KEY
      }
      // 在全局的targetMap中拿到当前响应式对象的依赖
      let depsMap = targetMap.get(target)
      if (depsMap === void 0) {
        targetMap.set(target, (depsMap = new Map()))
      }
      // 拿到当前key值的依赖，这里的dep是一个set，之前我们已经分析过了
      let dep = depsMap.get(key!)
      if (dep === void 0) {
        depsMap.set(key!, (dep = new Set()))
      }
      // 如果当前dep中不存在effect，则把effect添加到dep中
      // 并且effect上的deps属性也会添加当前的dep，这样做是为了方便做清除操作，后面我们会分析
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

也就是说，当effect第一次执行时，会将依赖也就是effect添加到响应式对象的dep中，此时dummy被赋值为0，
接下来我们更改a.count的值，触发Proxy的set方法中的trigger来派发更新。

    // 根据effect上的computed属性来决定将effct添加到哪个set中
    function addRunners(
      effects: Set<ReactiveEffect>,
      computedRunners: Set<ReactiveEffect>,
      effectsToAdd: Set<ReactiveEffect> | undefined
    ) {
      if (effectsToAdd !== void 0) {
        effectsToAdd.forEach(effect => {
          if (effect.computed) {
            computedRunners.add(effect)
          } else {
            effects.add(effect)
          }
        })
      }
    }

    // 如果effect上有scheduler则执行scheduler，否则执行effect本身
    function scheduleRun(
      effect: ReactiveEffect,
      target: object,
      type: OperationTypes,
      key: unknown,
      extraInfo?: DebuggerEventExtraInfo
    ) {
      if (__DEV__ && effect.onTrigger) {
        const event: DebuggerEvent = {
          effect,
          target,
          key,
          type
        }
        effect.onTrigger(extraInfo ? extend(event, extraInfo) : event)
      }
      if (effect.scheduler !== void 0) {
        effect.scheduler(effect)
      } else {
        effect()
      }
    }

    export function trigger(
      target: object,
      type: OperationTypes,
      key?: unknown,
      extraInfo?: DebuggerEventExtraInfo
    ) {
      // 从targetMap中拿到所有依赖
      const depsMap = targetMap.get(target)
      if (depsMap === void 0) {
        // never been tracked
        return
      }
      // 创建effects和computedRunners来保存即将重新执行的effect
      // 这里computedRunners存的是通过计算属性computed创建的effect
      const effects = new Set<ReactiveEffect>()
      const computedRunners = new Set<ReactiveEffect>()
      if (type === OperationTypes.CLEAR) {
        // collection being cleared, trigger all effects for target
        // 当传入的type为清除操作时，对象上的所有依赖都要添加到set中
        depsMap.forEach(dep => {
          addRunners(effects, computedRunners, dep)
        })
      } else {
        // schedule runs for SET | ADD | DELETE
        // 如果key值存在，则添加当前key值的依赖
        if (key !== void 0) {
          addRunners(effects, computedRunners, depsMap.get(key))
        }
        // also run for iteration key on ADD | DELETE
        // 如果触发set的操作是添加或者删除一个key时，依赖了长度或者迭代器相关的key值的依赖也要更新
        if (type === OperationTypes.ADD || type === OperationTypes.DELETE) {
          const iterationKey = Array.isArray(target) ? 'length' : ITERATE_KEY
          addRunners(effects, computedRunners, depsMap.get(iterationKey))
        }
      }
      const run = (effect: ReactiveEffect) => {
        scheduleRun(effect, target, type, key, extraInfo)
      }
      // Important: computed effects must be run first so that computed getters
      // can be invalidated before any normal effects that depend on them are run.
      // 遍历set中的所有effect，调用run方法，run方法中执行的是scheduleRun
      // 这里computedRunners中的effect会先执行，因为computed也是通过effect实现的，它也有可能作为
      // 普通effect的一个依赖值，后面compued篇我们再详细解释。
      computedRunners.forEach(run)
      effects.forEach(run)
    }

拿之前的例子来说，当前触发set的key值为count，这里在depsMap中拿到了之前在track中添加的effect，因为这个effect创建的时候没有传入computed属性，所以它被添加到effects变量中，并且它也没有scheduler方法，所以在遍历执行scheduleRun的时候会重新执行effect本身，也就是会重新执行一遍fn，之前例子的dummy值也会被赋值为2。这里涉及到一部分计算属性computed的逻辑，不过没有关系，因为计算属性的实现也是通过创建一个effect，后面computed篇我们会分析到。

到这里我们就分析完整个effect的实现了，不过这里想提一下effect的几个实现细节:

1、cleanup(effect)

    // 每次执行传入的fn前，把当前effect从所有依赖此effect的dep上删除，并清空本身的deps属性上的值
    // 这样做是为了实时更新依赖，来看看下面的test case 
    function cleanup(effect: ReactiveEffect) {
      const { deps } = effect
      if (deps.length) {
        for (let i = 0; i < deps.length; i++) {
          deps[i].delete(effect)
        }
        deps.length = 0
      }
    }

    // 当出现effect传入的函数里有分支这种情况时，我们需是要根据obj.run的值来决定是否要依赖obj.prop的
    // 当obj.run为false时，我们并不关心obj.prop的变化，此时是不需要依赖obj.prop的
    // 但是有可能obj.prop之前已经将effect添加到依赖中，所以每次重新执行effect传入的fn前，
    // 要先清空所有依赖，在调用fn的时候再根据当前fn里面需要访问的响应式的对象来重新添加依赖。
    // 其实在模板上会经常出现这种情况，例如
    // <div v-if="obj.run">
    //    <div>{{obj.prop}}</div>
    // </div>
    // <div v-else>other</div>

    it('should discover new branches while running automatically', () => {
      let dummy
      const obj = reactive({ prop: 'value', run: false })

      const conditionalSpy = jest.fn(() => {
        dummy = obj.run ? obj.prop : 'other'
      })
      effect(conditionalSpy)

      expect(dummy).toBe('other')
      expect(conditionalSpy).toHaveBeenCalledTimes(1)
      obj.prop = 'Hi'
      expect(dummy).toBe('other')
      expect(conditionalSpy).toHaveBeenCalledTimes(1)
      obj.run = true
      expect(dummy).toBe('Hi')
      expect(conditionalSpy).toHaveBeenCalledTimes(2)
      obj.prop = 'World'
      expect(dummy).toBe('World')
      expect(conditionalSpy).toHaveBeenCalledTimes(3)
    })

下一篇：reactive之computed篇