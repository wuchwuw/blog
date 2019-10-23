## reactivity之ref篇

Ref是在Vue3.x中提出的一种新的数据类型，把Ref放在最前面介绍是因为，Ref贯穿着整个Vue3.x的响应式，是reactivity中很重要的一个概念。本文将从源码出发，分析Ref的实现原理，以及它在Vue3.x中存在的意义。

### test case

在分析源码之前，我们首先应该知道在Vue3.x中Ref到底是怎么用的。而了解一个api的最好的方法就是看它的test case，所以我们先从test case出发，来看看Ref相关的Api的用法以及特性。

    describe('reactivity/ref', () => {
      // 通过ref api创建的Ref类型的数据上有一个value属性
      it('should hold a value', () => {
        const a = ref(1)
        expect(a.value).toBe(1)
        a.value = 2
        expect(a.value).toBe(2)
      })

      // 通过ref api创建的数据一定是响应式的
      it('should be reactive', () => {
        const a = ref(1)
        let dummy
        effect(() => {
          dummy = a.value
        })
        expect(dummy).toBe(1)
        a.value = 2
        expect(dummy).toBe(2)
      })
      
      // 如果传入ref的是一个对象，那么传入的对象也会变成响应式对象
      it('should make nested properties reactive', () => {
        const a = ref({
          count: 1
        })
        let dummy
        effect(() => {
          dummy = a.value.count
        })
        expect(dummy).toBe(1)
        a.value.count = 2
        expect(dummy).toBe(2)
      })

      // 在其他响应式对象中嵌套Ref类型的数据时，可以直接通过变量访问
      // Ref类型的值，而不需要访问Ref类型上的value属性
      it('should work like a normal property when nested in a reactive object', () => {
        const a = ref(1)
        const obj = reactive({
          a,
          b: {
            c: a,
            d: [a]
          }
        })
        let dummy1
        let dummy2
        let dummy3
        effect(() => {
          dummy1 = obj.a
          dummy2 = obj.b.c
          dummy3 = obj.b.d[0]
        })
        expect(dummy1).toBe(1)
        expect(dummy2).toBe(1)
        expect(dummy3).toBe(1)
        a.value++
        expect(dummy1).toBe(2)
        expect(dummy2).toBe(2)
        expect(dummy3).toBe(2)
        obj.a++
        expect(dummy1).toBe(3)
        expect(dummy2).toBe(3)
        expect(dummy3).toBe(3)
      })

      // 当Ref类型的数据嵌套在Ref类型的数据中时，它也是默认展开的
      // 与上面的test case 同理
      it('should unwrap nested ref in types', () => {
        const a = ref(0)
        const b = ref(a)

        expect(typeof (b.value + 1)).toBe('number')
      })

      // 当Ref类型的数据嵌套在Ref类型的数据中时，它也是默认展开的
      // 与上面的test case 同理
      it('should unwrap nested values in types', () => {
        const a = {
          b: ref(0)
        }

        const c = ref(a)

        expect(typeof (c.value.b + 1)).toBe('number')
      })
      
      // 当Ref类型的数据嵌套在Ref类型的数组中时，它也是默认展开的
      // 与上面的test case 同理
      it('should properly unwrap ref types nested inside arrays', () => {
        const arr = ref([1, ref(1)]).value
        // should unwrap to number[]
        arr[0]++
        arr[1]++

        const arr2 = ref([1, new Map<string, any>(), ref('1')]).value
        const value = arr2[0]
        if (typeof value === 'string') {
          value + 'foo'
        } else if (typeof value === 'number') {
          value + 1
        } else {
          // should narrow down to Map type
          // and not contain any Ref type
          value.has('foo')
        }
      })

      // 通过isRef api来判断一个对象是不是Ref类型
      test('isRef', () => {
        expect(isRef(ref(1))).toBe(true)
        expect(isRef(computed(() => 1))).toBe(true)

        expect(isRef(0)).toBe(false)
        expect(isRef(1)).toBe(false)
        // an object that looks like a ref isn't necessarily a ref
        expect(isRef({ value: 0 })).toBe(false)
      })

      // 通过toRefs api 创建一个新对象，将对象上的所有属性转化成Ref类型
      // 从下面的例子可以看出，在新创建的对象和旧的对象之间有一层代理
      // 修改旧对象上的属性，新对象上Ref类型的value值也会改变，反之也一样
      test('toRefs', () => {
        const a = reactive({
          x: 1,
          y: 2
        })

        const { x, y } = toRefs(a)

        expect(isRef(x)).toBe(true)
        expect(isRef(y)).toBe(true)
        expect(x.value).toBe(1)
        expect(y.value).toBe(2)

        // source -> proxy
        a.x = 2
        a.y = 3
        expect(x.value).toBe(2)
        expect(y.value).toBe(3)

        // proxy -> source
        x.value = 3
        y.value = 4
        expect(a.x).toBe(3)
        expect(a.y).toBe(4)

        // reactivity
        let dummyX, dummyY
        effect(() => {
          dummyX = x.value
          dummyY = y.value
        })
        expect(dummyX).toBe(x.value)
        expect(dummyY).toBe(y.value)

        // mutating source should trigger effect using the proxy refs
        a.x = 4
        a.y = 5
        expect(dummyX).toBe(4)
        expect(dummyY).toBe(5)
      })
    })

那么分析完ref有关的所有test case，我们可以对ref相关api做一个总结：

1、 ref 创建了一个Ref类型的对象，并将值保存在对象的value属性上，并且它的value上的值是响应式的。
2、 isRef 判断一个数据是不是Ref类型
3、 toRefs 通过传入的对象创建一个新的对象，新对象上的属性都是Ref类型的。
4、 当Ref类型的的数据嵌套在其他Ref类型或者是响应式对象中时，它们是默认展开的

### Ref类型

前面的篇幅通过ref的test case介绍了ref相关api的用法，也多次提到了Ref这种类型的数据，接下来我们通过Ref的类型定义来看看Ref类型究竟长什么样子：

    export interface Ref<T = any> {
      _isRef: true         // Ref类型的标志位
      value: UnwrapRef<T>  // 存放值的value属性，类型是UnwrapRef
    }

    // Recursively unwraps nested value bindings.
    // 递归的解开嵌套的Ref类型
    export type UnwrapRef<T> = {
      cRef: T extends ComputedRef<infer V> ? UnwrapRef<V> : T
      ref: T extends Ref<infer V> ? UnwrapRef<V> : T
      array: T extends Array<infer V> ? Array<UnwrapRef<V>> : T
      object: { [K in keyof T]: UnwrapRef<T[K]> }
    }[T extends ComputedRef<any>
      ? 'cRef'
      : T extends Ref
        ? 'ref'
        : T extends Array<any>
          ? 'array'
          : T extends Function | CollectionTypes
            ? 'ref' // bail out on types that shouldn't be unwrapped
            : T extends object ? 'object' : 'ref']

这里稍微难理解的地方应该是value值的UnwrapRef<T>类型，这需要你对typescript的infer关键字有一定的了解，简单来说UnwrapRef<T>就是对传入的类型递归的进行解套的过程，举一个例子：

    // 假设我们创建一个Ref a，里面嵌套着其他Ref类型
    const a = ref({
      b: ref(1),
      c: [ref(2), ref({
        d: ref(3)
      })]
    })
    
    // 那么此时a.value的类型为
    {
      b: number;
      c: (number | {
          d: number;
      })[];
    }

### ref api的源码

#### isRef

    // 判断传入对象上_isRef属性是不是true
    export function isRef(r: any): r is Ref {
      return r ? r._isRef === true : false
    }

#### ref

    export function ref(raw: unknown) {
      // 如果传入的参数已经是Ref类型，直接返回
      if (isRef(raw)) {
        return raw
      }
      // 如果传入传入的参数是一个对象，那么通过reactive方法创建
      // 响应式对象，关于reactive后面的文章会介绍
      // 这里也印证了之前的test case
      raw = convert(raw)
      // 返回一个对象设置_isRef为true
      // 设置value值的getter、setter
      // track和trigger方法是Ref类型数据拥有响应式特性的关键
      // 后面介绍effect时会着重介绍
      const r = {
        _isRef: true,
        get value() {
          // 访问value值时，通过track方法收集依赖
          track(r, OperationTypes.GET, '')
          return raw
        },
        set value(newVal) {
          raw = convert(newVal)
          // 设置value值时，通过trigger方法派发更新
          trigger(r, OperationTypes.SET, '')
        }
      }
      return r as Ref
    }

    const convert = <T extends unknown>(val: T): T =>
      isObject(val) ? reactive(val) : val

#### toRefs

    // 遍历传入的对象的每个key，调用toProxyRef为每个key创建Ref
    export function toRefs<T extends object>(
      object: T
    ): { [K in keyof T]: Ref<T[K]> } {
      const ret: any = {}
      for (const key in object) {
        ret[key] = toProxyRef(object, key)
      }
      return ret
    }

    // 简单的返回一个Ref类型的对象
    function toProxyRef<T extends object, K extends keyof T>(
      object: T,
      key: K
    ): Ref<T[K]> {
      return {
        _isRef: true,
        get value(): any {
          return object[key]
        },
        set value(newVal) {
          object[key] = newVal
        }
      }
    }

这里有一个细节，就是ref和toProxyRef同样是返回一个Ref类型的对象，但是ref多了track和trigger来收集依赖和派发更新，并且会调用convert方法创建响应式对象，而toProxyRef只是对原对象做了一层代理，并且直接返回原对象上的值。

### Ref类型存在的意义

我们知道在js中引用类型的值传递是通过传递引用，旧值的修改会影响旧值，而普通类型的传递是对值的复制，旧值的修改不会影响新值。因为有这样的语言特性，在Vue的响应式中会引发一些小问题：

    // 例如，我们通过reactive创建了一个响应式对象state
    // 并且通过computed创建了一个计算属性double
    import { reactive, computed } from 'vue'

    const state = reactive({
      count: 0
    })

    const double = computed(() => state.count * 2)

    // 假设computed的简单实现是这样的
    // 每次重新计算value的值并直接返回value
    function computed(getter) {
      let value
      watch(() => {
        value = getter()
      })
      return value
    }

从我们对js的了解来看，这样是行不通的，因为如果computed返回的是一个基础类型的值，那么当它创建赋值给double时，它们的联系就断开了，也就是说computed内部value的修改并不会对外部的double有什么影响，计算属性也就无法更新，所以正确的方式是将value值包装在对象中，通过引用来传递：

    function computed(getter) {
      const ref = {
        value: null
      }
      watch(() => {
        ref.value = getter()
      })
      return ref
    }

所以说computed应该返回一个Ref类型的对象，而不是一个基础类型，后面分析computed源码的时候也可以证实这一点。
我们再举一个例子：

    // 在useMousePosition方法中创建一个响应式对象，并返回
    function useMousePosition() {
      const pos = reactive({
        x: 0,
        y: 0
      })
      return pos
    }

    // 在setup方法中通过解构复制给变量x，y
    // 并在setup方法中返回
    export default {
      setup() {
        const { x, y } = useMousePosition()
        // 丢失响应性！
        return {
          x,
          y
        }
      }
    }

也就是说响应式对象在解构赋值的过程中会失去他们响应式的特性，那么有两种方法可以解决：

    // 方法一
    // 不使用解构复制，直接返回整个对象
    // 那么相应的在模板中可能就要写{{pos.x}}{{pos.y}}
    const pos = useMousePosition()
    return {
      pos
    }
    
    // 方法二
    // 通过toRefs创建一个每个属性都是Ref类型对象的对象
    // 通过上面分析toRefs的源码我们也可以知道，这里
    // toRefs是对原对象的访问做了一层代理，这样x，y属性都是对象，解构
    // 之后不会失去响应式的特性。
    function useMousePosition() {
      const pos = reactive({
        x: 0,
        y: 0
      })
      return toRefs(pos)
    }
    const { x, y } = useMousePosition()
    // 保持响应性！
    return {
      x,
      y
    }

通过上面两个例子可以看出，Ref类型存在的意义在于：

1、对于基础类型的值来说，需要将他们包装在对象中，以便能够追踪它们值的变化

2、在响应式对象赋值的过程中，维持属性响应式的特性

下一篇: [reactivity之reactive篇]()