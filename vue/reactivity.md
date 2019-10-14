## reactivity

在Vue3.x中，负责响应式的代码主要放在reactivity包中。本文主要是介绍reactivity暴露的几个主要的接口，并从源码的角度来分析Vue3.x的响应式原理。

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

在Vue3.x中，当某些state的值依赖于其他state时，可以使用computed API来创建一个计算属性。

    import { reactive, computed } from 'vue'

    const state = reactive({
      count: 0
    })

    const double = computed(() => state.count * 2)

