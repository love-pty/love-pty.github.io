# 为什么不用index做key？

## 虚拟DOM

我们知道，Vue 不可以直接操作 DOM 结构，而是通过数据驱动、指令等机制来间接操作 DOM 结构。当我们修改模版中的数据时，Vue 会触发重新渲染过程，调用`render`函数，它会返回一个 **虚拟 DOM 树**，它描述了整个组件模版的结构。

举个例子：

```vue
<template>
  <ul class="list">
    <li v-for="item in list" :key="item.index" class="item">{{ item }}</li>
  </ul>
</template>

<script setup>
import { ref } from 'vue';
const list = ref(['html', 'css', 'js'])
</script>
```

Vue 在渲染这个列表时，就会调用`render`函数，它会返回一个类似下面这个虚拟 DOM 树。

```javascript
let VDom = {
    tagName: 'ul',
    props: {
        class: 'list'
    },
    chilren: [
        {
            tagName: 'li',
            props: {
                class: 'item'
            },
            chilren: ['html']
        },
        {
            tagName: 'li',
            props: {
                class: 'item'
            },
            chilren: ['css']
        },
        {
            tagName: 'li',
            props: {
                class: 'item'
            },
            chilren: ['js']
        }
    ]
}
```

虚拟 DOM 的每个节点对应于真实 DOM 树中的一个节点。

当我们修改数据时，Vue 又会触发重新渲染的过程。

```javascript
const list = ref(['html', 'css', 'vue']) //修改列表第三项'js'->'vue'
```

Vue 又会生成一个新的虚拟DOM树：

```javascript
let VDom = {
    tagName: 'ul',
    props: {
        class: 'list'
    },
    chilren: [
        {
            tagName: 'li',
            props: {
                class: 'item'
            },
            chilren: ['html']
        },
        {
            tagName: 'li',
            props: {
                class: 'item'
            },
            chilren: ['css']
        },
        {
            tagName: 'li',
            props: {
                class: 'item'
            },
            chilren: ['vue']
        }
    ]
}
```

注意观察，这里最后一个节点的子节点为`'vue'`，发生了数据变化，Vue内部又会返回一个新的虚拟 DOM。那么 Vue 是如何将这个变化响应给页面的呢？

## 摆在面前的有两条路

要么重新渲染这个新的虚拟 DOM ，要么只新旧虚拟 DOM 之间改变的地方。

显而易见，只渲染修改了的地方是不是会更节省性能。

巧了，尤雨溪也是这样想的，于是便有了“ Diff 算法 ”。

## Diff 算法

Vue 将新生成的新虚拟 DOM 与上一次渲染时生成的旧虚拟 DOM 进行比较，对比出是哪个虚拟节点更改了，找出这个虚拟节点，并只更新这个虚拟节点所对应的**真实节点**，而不用更新其他数据没发生改变的节点。

我自己总结了一下Diff算法的过程，由于代码过多，就不在此展示了：

> 1. 新旧虚拟DOM对比的时候，Diff 算法比较只会在同层级进行，不会跨层级比较。
> 2. 首先比较两个节点的类型，如果类型不同，则废弃旧节点并用新节点替代。
> 3. 对于相同类型的节点，进一步比较它们的属性。记录属性差异，以便生成相应的补丁。
> 4. 如果两个节点相同，继续递归比较它们的子节点，直到遍历完整个树。
> 5. 如果节点有唯一标识，可以通过这些标识来快速定位相同标识的节点。
> 6. 如果节点的相同，只是顺序变化，不会执行不必要的操作。

## 为什么不用 index 做 key？

平常`v-for`循环渲染的时候，为什么不建议用 `index` 作为循环项的 `key` 呢？

举个栗子：

```vue
<div id="app">
    <ul>
        <li v-for="item in list" :key="item.index">{{item}}</li>
    </ul>
    <button @click="add">添加</button>
</div>
<script>
    const { createApp, ref } = Vue
    createApp({
        setup() {
            const list = ref(['html', 'css', 'js']);
            const add=()=> {
                list.value.unshift('阳阳羊');
            }
            return {
                list,
                add
            }
        }
    }).mount('#app')
</script>
```

这里用 `index`为 `key`渲染这个列表，我们通过 `add` 方法在列表的前面添加一项。

我们发现添加操作导致的整个列表的重新渲染，按道理来说，Diff 算法会复用后面的三项，因为它们只是位置发生了变化，内容并没有改变。但是我们回过头来发现，我们在前面添加了一项，导致后面三项的 `index` 变化，从而导致 `key` 值发生变化。Diff 算法失效了？

那我们可以怎么解决呢？其实我们只要使用一个独一无二的值来当做`key`就行了

```vue
<div id="app">
    <ul>
        <li v-for="item in list" :key="item.id">{{item.name}}</li>
    </ul>
    <button @click="add">添加</button>
</div>
<script>
    const { createApp, ref } = Vue
    createApp({
        setup() {
            const list = ref(
            [
                { name: "html", id: 1 }, 
                { name: "css", id: 2 }, 
                { name: "js", id: 3 }, 
            ]);
            const add=()=> {
                list.value.unshift({ name: '阳阳羊', id: 4 });
            }
            return {
                list,
                add
            }
        }
    }).mount('#app')
</script>
```

这样，`key`就是永远不变的，更新前后都是一样的，并且又由于节点的内容本来就没变，所以 Diff 算法完美生效，只需将新节点添加到真实 DOM 就行了。