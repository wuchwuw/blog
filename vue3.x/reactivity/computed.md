## reactivity之computed篇

在Vue3.x中computed也是基于effect来实现的，在effect篇中，我们介绍了响应式数据是如何与effect配合来实现收集依赖和派发更新的，在分析派发更新相关源码的时候也设计到了一点computed的内容，接下来我们通过computed的源码来看看普通的effect和计算属性的effect有什么不同。

    // computed的返回类型可以是WritableComputedRef或者是ComputedRef
    // 这两者都是Ref类型，区别就是value值是不是只读
    export interface ComputedRef<T> extends WritableComputedRef<T> {
      readonly value: UnwrapRef<T>
    }

    export interface WritableComputedRef<T> extends Ref<T> {
      readonly effect: ReactiveEffect<T>
    }

    // 传入computed参数的类型，可以是一个函数，也可以是一个包含get、set方法的对象
    export type ComputedGetter<T> = () => T
    export type ComputedSetter<T> = (v: T) => void

    export interface WritableComputedOptions<T> {
      get: ComputedGetter<T>
      set: ComputedSetter<T>
    }

    export function computed<T>(getter: ComputedGetter<T>): ComputedRef<T>
    export function computed<T>(
      options: WritableComputedOptions<T>
    ): WritableComputedRef<T>
    export function computed<T>(
      getterOrOptions: ComputedGetter<T> | WritableComputedOptions<T>
    ) {
      let getter: ComputedGetter<T>
      let setter: ComputedSetter<T>
      
      // 如果传入的参数是一个函数，那么直接赋值给getter
      // 否则拿get、set赋值给getter、setter
      if (isFunction(getterOrOptions)) {
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
      // 初始化变量dirty用来控制是否重新计算value的值
      let dirty = true
      let value: T

      // 创建一个effect，并把getter当做参数传入，注意这里传入的options
      // lazy为ture，那么创建effect并不会立马执行
      // computed为true，那么在派发更新时effect会被添加到computedRunners这个set中
      // 并且这里传入了scheduler，在computedRunners遍历执行的时候会调用scheduler而不是effect本身
      // 也就是说当getter里面的响应式数据变化时，scheduler会重新执行将dirty设置为true
      const runner = effect(getter, {
        lazy: true,
        // mark effect as computed so that it gets priority during trigger
        computed: true,
        scheduler: () => {
          dirty = true
        }
      })
      // 返回一个Ref类型的对象
      return {
        _isRef: true,
        // expose effect so computed can be stopped
        effect: runner,
        get value() {
          // 当访问计算属性时根据dirty来决定是否重新计算value
          // 这里的runner也就是effect，执行runner就是重新执行我们传入的getter
          if (dirty) {
            value = runner()
            dirty = false
          }
          // When computed effects are accessed in a parent effect, the parent
          // should track all the dependencies the computed property has tracked.
          // This should also apply for chained computed properties.
          // trackChildRun方法这里源码有一段注释，简单来说就是当computed嵌套在computed或者effect中时，
          // computed也需要收集父effect的依赖，我们后面会再详细解释
          trackChildRun(runner)
          return value
        },
        set value(newValue: T) {
          setter(newValue)
        }
      }
    }

可以看到整个computed的实现非常简单，前提是你已经理解了effect的实现原理，那么这里computed有几个小的细节

1、返回值是一个Ref类型的对象

2、可以看到computed的value值并没有一开始就计算，而是访问computed的值的时候才计算，初始化时dirty是为true的，
所以第一次访问computed的值会执行runner计算value的值，并将dirty设置为false，而runner本身是一个effect函数，执行runner的时候就会访问到getter里面的响应式对象，触发Proxy get并进行依赖收集，当响应式的值更改时，触发Proxy set派发更新，派发更新时，重新执行的并不是computed effect本身，而是effect上的scheduler方法，也就是说当响应式的值改变时，computed effect只是把dirty设置为true，等到下一次访问computed的值时，才会重新计算value的值。

3、trackChildRun

试想一下，计算属性本身就是一个Ref类型的对象，它也是响应式的，所以它也可以嵌套在其他computed或者effect中，那么当访问嵌套在effect中的计算属性时，同样也需要进行依赖收集，这就是trackChildRun的作用：

    // 这里传入的childRunner是computed effect
    // 通过effectStack拿到父effect，遍历computed effect的deps属性
    // 把父effect添加到computed effect的每一个dep中
    // 这样computed里面的响应式对象的dep中就同时添加了computed effect和父effect
    // 这里可能有点绕，我们通过下面一个例子来看
    function trackChildRun(childRunner: ReactiveEffect) {
      if (effectStack.length === 0) {
        return
      }
      const parentRunner = effectStack[effectStack.length - 1]
      for (let i = 0; i < childRunner.deps.length; i++) {
        const dep = childRunner.deps[i]
        if (!dep.has(parentRunner)) {
          dep.add(parentRunner)
          parentRunner.deps.push(dep)
        }
      }
    }

例子：

    const state = reactive({ count : 0 })
    const cState = computed(() => state.count)
    let dummy
    effect(() => {
      dummy = cState.value
    })
    console.log(dummy)
    state.count++
    console.log(dummy)

例子中我们将计算属性嵌套在一个effect中，当访问cState这个计算属性时，触发计算属性的get方法来计算value的值，这里就会调用到trackChildRun方法，并且计算属性是在一个effect中访问的，在执行传入effect的方法之前已经将外层的effect push到effectStack中，那么此时在trackChildRun方法中拿到的parentRunner就是外层的effect，所以count这个key的depMap中会同时添加computed effect和外层的effect。也就是说，当计算属性嵌套在effect中时，state.count修改需要同时更新计算属性，也要重新执行effect，这也是为什么在trigger方法中需要先遍历computedRunners后遍历effects，因为在这种情况下，必须先执行computed effect将dirty设置为true，当computed effect的父effect访问它的时候，才能保证拿到的计算属性的值是最新的。