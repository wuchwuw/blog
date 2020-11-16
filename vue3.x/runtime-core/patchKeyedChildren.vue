## 在Vue3.x中，带key值的子节点是如何更新的

在Vue中，我们经常使用v-for指令来渲染列表，并且官方推荐我们在列表上绑定唯一的key值，以便于更新时Vue可以
根据key值来进行优化。本文将基于Vue3.x的源码，分析在Vue3.x的版本中，对于带key值的子节点是如何更新的。
在阅读本文前，先简单了解一下本文涉及到的几个方法:

isSameType: 传入两个VNode，如果它们的类型相同并且key值相等，则返回true

patch(vnode1, vnode2, ...): 
创建或者更新VNode，如果传入的vnode1为null，代表当前patch为创建一个节点，根据
vnode2的类型来创建真实的DOM节点并且递归的创建它的子节点，如果vnode1不为null，
代表当前patch为更新一个节点，根据新的VNode也就是传入的vnode2来更新DOM节点并递
归的更新它的子节点。

unmount(vnode, ...): 销毁一个节点，并将它创建的DOM节点从父节点中删除

以上简单的介绍仅仅是为了方便理解本文的内容，实际上在Vue新版本中，对模板的编译和更新下了很多的功夫，就拿
不过这

    const list = [
      { key: 'a', value: 'a'},
      { key: 'b', value: 'b'},
      { key: 'c', value: 'c'},
      { key: 'd', value: 'd'},
      { key: 'e', value: 'e'},
    ]

假设我们当前有列表数据list，它是一个带有key值和value值的对象数组，
我们使用v-for渲染这样一个列表，并将key属性作为列表项的唯一的key值。一般来说列表的插入、删除或者列表项顺序的改变
都会导致列表的重新渲染，在Vue3.x中将分为以下几个步骤来对比新旧列表，并根据它们之间的差异来进行更新：

    let i = 0             // 注意这里i为全局变量
    const l2 = c2.length 
    let e1 = c1.length - 1 // 新列表最后一位的index值
    let e2 = l2 - 1 // 旧列表最后一位的index值

1、从头开始遍历，直到第一个不相同的节点

    // c1 [d, b, c, a, b]
    // c2 [d, c, d, a, b]
    // 此步骤将通过patch方法先更新在新旧列表中相同的节点d
    // 并递增i的值，它代表着当前前进的位置，此刻i = 1
    while (i <= e1 && i <= e2) {
      const n1 = c1[i]
      const n2 = optimized
        ? (c2[i] as HostVNode)
        : (c2[i] = normalizeVNode(c2[i]))
      if (isSameType(n1, n2)) {
        patch(
          n1,
          n2,
          container,
          parentAnchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        )
      } else {
        break
      }
      i++
    }

2、从尾部开始遍历，直到第一个不相同的节点

    // 相同的例子
    // c1 [d, b, c, a, b]
    // c2 [d, c, d, a, b]
    // 在步骤1中，我们已经更新了头部相同的节点d
    // 步骤2改为从尾部开始遍历，寻找相同的节点并更新
    // 那么在这个例子中，步骤2将通过patch方法更新a,b节点
    // 并通过e1,e2的值来记录当前的位置
    // 此刻i=1, e1=2, e2=2
    while (i <= e1 && i <= e2) {
      const n1 = c1[e1]
      const n2 = optimized
        ? (c2[i] as HostVNode)
        : (c2[e2] = normalizeVNode(c2[e2]))
      if (isSameType(n1, n2)) {
        patch(
          n1,
          n2,
          container,
          parentAnchor,
          parentComponent,
          parentSuspense,
          isSVG,
          optimized
        )
      } else {
        break
      }
      e1--
      e2--
    }

可以看出，通过步骤1和2先更新了新旧列表中头尾相同的节点，并更新变量i、e1、e2来记录当前更新的位置。


3、 在更新头尾相同的节点后，根据i、e1、e2的值来决定剩下的节点如何更新，分为以下几种情况：

    // 假设
    // c1 [a, b, c, d]
    // c2 [a, b, e, c, d]
    // 进行完步骤1、2后，此时 i=2, e1=1, e2=2
    // 那么当i > e1,并且i <= e2时，也就是执行完步骤1，2后c1中已经没有剩余待更新的节点
    // 并且c2中还有未更新的节点，代表着c2中剩下的为新插入的节点，则调用patch方法去创建
    // c2中剩余的节点
    if (i > e1) {
      if (i <= e2) {
        const nextPos = e2 + 1
        const anchor =
          nextPos < l2 ? (c2[nextPos] as HostVNode).el : parentAnchor
        while (i <= e2) {
          patch(
            null,
            optimized ? (c2[i] as HostVNode) : (c2[i] = normalizeVNode(c2[i])),
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG
          )
          i++
        }
      }
    }

    // 同理， 如果i > e2，也就是说c2中已经没有剩余待更新的节点
    // 并且c1中还有未更新的节点，代表着当前c1中剩下的为已经删除的节点，则调用unmount方法去删除
    // c1中剩余的节点
    else if (i > e2) {
      while (i <= e1) {
        unmount(c1[i], parentComponent, parentSuspense, true)
        i++
      }
    }

    // 当c1、c2中均剩余节点时，进入以下分支

    else {
      const s1 = i // prev starting index
      const s2 = i // next starting index

      // 5.1 build key:index map for newChildren
      const keyToNewIndexMap: Map<string | number, number> = new Map()
      for (i = s2; i <= e2; i++) {
        const nextChild = optimized
          ? (c2[i] as HostVNode)
          : (c2[i] = normalizeVNode(c2[i]))
        if (nextChild.key != null) {
          if (__DEV__ && keyToNewIndexMap.has(nextChild.key)) {
            warn(
              `Duplicate keys found during update:`,
              JSON.stringify(nextChild.key),
              `Make sure keys are unique.`
            )
          }
          keyToNewIndexMap.set(nextChild.key, i)
        }
      }

      // 5.2 loop through old children left to be patched and try to patch
      // matching nodes & remove nodes that are no longer present
      let j
      let patched = 0
      const toBePatched = e2 - s2 + 1
      let moved = false
      // used to track whether any node has moved
      let maxNewIndexSoFar = 0
      // works as Map<newIndex, oldIndex>
      // Note that oldIndex is offset by +1
      // and oldIndex = 0 is a special value indicating the new node has
      // no corresponding old node.
      // used for determining longest stable subsequence
      const newIndexToOldIndexMap = new Array(toBePatched)
      for (i = 0; i < toBePatched; i++) newIndexToOldIndexMap[i] = 0

      for (i = s1; i <= e1; i++) {
        const prevChild = c1[i]
        if (patched >= toBePatched) {
          // all new children have been patched so this can only be a removal
          unmount(prevChild, parentComponent, parentSuspense, true)
          continue
        }
        let newIndex
        if (prevChild.key != null) {
          newIndex = keyToNewIndexMap.get(prevChild.key)
        } else {
          // key-less node, try to locate a key-less node of the same type
          for (j = s2; j <= e2; j++) {
            if (
              newIndexToOldIndexMap[j - s2] === 0 &&
              isSameType(prevChild, c2[j] as HostVNode)
            ) {
              newIndex = j
              break
            }
          }
        }
        if (newIndex === undefined) {
          unmount(prevChild, parentComponent, parentSuspense, true)
        } else {
          newIndexToOldIndexMap[newIndex - s2] = i + 1
          if (newIndex >= maxNewIndexSoFar) {
            maxNewIndexSoFar = newIndex
          } else {
            moved = true
          }
          patch(
            prevChild,
            c2[newIndex] as HostVNode,
            container,
            null,
            parentComponent,
            parentSuspense,
            isSVG,
            optimized
          )
          patched++
        }
      }

      // 5.3 move and mount
      // generate longest stable subsequence only when nodes have moved
      const increasingNewIndexSequence = moved
        ? getSequence(newIndexToOldIndexMap)
        : EMPTY_ARR
      j = increasingNewIndexSequence.length - 1
      // looping backwards so that we can use last patched node as anchor
      for (i = toBePatched - 1; i >= 0; i--) {
        const nextIndex = s2 + i
        const nextChild = c2[nextIndex] as HostVNode
        const anchor =
          nextIndex + 1 < l2
            ? (c2[nextIndex + 1] as HostVNode).el
            : parentAnchor
        if (newIndexToOldIndexMap[i] === 0) {
          // mount new
          patch(
            null,
            nextChild,
            container,
            anchor,
            parentComponent,
            parentSuspense,
            isSVG
          )
        } else if (moved) {
          // move if:
          // There is no stable subsequence (e.g. a reverse)
          // OR current node is not among the stable sequence
          if (j < 0 || i !== increasingNewIndexSequence[j]) {
            move(nextChild, container, anchor)
          } else {
            j--
          }
        }
      }
    }