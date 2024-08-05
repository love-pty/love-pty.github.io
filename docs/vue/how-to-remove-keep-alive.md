# 手动清除keep-alive缓存的一种解决方案

`keep-alive`的作用主要是**缓存组件实例**，以提高应用的性能和用户体验。具体来说，当一个组件被`<keep-alive>`包裹时，该组件的实例在第一次渲染后会被缓存下来，而不是被销毁。这样，在需要再次显示该组件时，可以直接从缓存中读取其实例，而无需重新渲染和初始化，从而减少了资源的消耗和渲染时间。

## keep-alive组件的生命周期

当组件实例不被缓存时，组件渲染会经历必要的生命周期。但当组件被缓存后，加载时不会触发`onMounted`，也没有卸载时的钩子函数`onMounted`，随之诞生的是`onActivated`和`onDeactivated`。

- `onActivated`：在组件被激活时调用。这通常发生在通过导航到该组件的路由（无论是直接导航还是通过缓存的视图重新进入）而使得该组件变为可见时。如果你使用了 `Vue Router`的 `<keep-alive>` 包裹 `<router-view>`，那么当从其他视图切换回被 `<keep-alive>` 包裹的视图时，`onActivated` 会被调用，因为此时组件被视为“重新激活”。

- `onDeactivated`：与 `onActivated` 相对，`onDeactivated` 钩子在组件被停用时调用。这发生在路由导航导致当前组件不再可见时。如果使用了 `<keep-alive>`，那么当组件被从视图中移除但尚未销毁时（即被缓存起来），`onDeactivated` 会被调用。

## keep-alive使用示例

```vue
<script setup lang="ts">
import { RouterLink, RouterView } from 'vue-router'
</script>

<template>
    <div class="link">
      <router-link to="/">发现</router-link>
      <router-link to="/pageB">关注</router-link>
      <router-link to="/pageC">商城</router-link>
      <router-link to="/pageD">主页</router-link>
    </div>
    <div class="routerView">
      <routerView v-slot="{ Component }">
        <!-- keep-alive中只能传入一个子组件。这也包括注释 -->
        <!-- key将指定component的唯一标识符 -->
        <keep-alive>
          <component :is="Component" :key="$route.name" v-if="$route.meta.keepAlive"/>
        </keep-alive>
        <component :is="Component" :key="$route.name" v-if="!$route.meta.keepAlive"/>
      </routerView>
    </div>
</template> 
```

```typescript
const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes: [
    {
      path: '/',
      redirect: '/pageA'
    },
    {
      path: '/pageA',
      name: 'pageA',
      component: pageA,
      meta: {
        keepAlive: true
      }
    },
    {
      path: '/pageB',
      name: 'pageB',
      component: pageB,
      meta: {
        keepAlive: true
      }
    },
    {
      path: '/pageC',
      name: 'pageC',
      component: pageC,
      meta: {
        keepAlive: true
      }
    },
    {
      path: '/pageD',
      name: 'pageD',
      component: pageD,
      meta: {
        keepAlive: true
      }
    }
  ]
})
```

## 为什么要手动清除

案例一：我们知道表单可以收集数据，也可以给表单的`model`赋值并且禁用，这样可以用来展示收集好的数据。我们将编辑和查看两个功能封装在一个组件中，从编辑页离开时需要缓存，从查看页离开时不需要缓存。这时单纯的使用`keep-alive`就不能达到我们的需求。

# 如何手动清除缓存

回到我们的问题上，官方并没有直接提供`API`为`keep-alive`清除缓存。

我们唯一能操作keep-alive组件的的地方就是三个属性：

```typescript
interface KeepAliveProps {
  /**
   * 如果指定，则只有与 `include` 名称
   * 匹配的组件才会被缓存。
   */
  include?: MatchPattern
  /**
   * 任何名称与 `exclude`
   * 匹配的组件都不会被缓存。
   */
  exclude?: MatchPattern
  /**
   * 最多可以缓存多少组件实例。
   */
  max?: number | string
}

type MatchPattern = string | RegExp | (string | RegExp)[]
```

我们要使用的是`include`属性，他允许我们动态的传入一个包含`route.name`的数组，通过控制该数组可以实现对路由实例的缓存或者清除缓存。下文将用`cacheList`来表述该数组。

当清除缓存后该实例可能被卸载，因此我选择在集中状态管理工具中控制数组。

我使用的是`pina`也可以使用`Vuex`。

```typescript
import { computed, ref } from 'vue'
import { defineStore } from 'pinia'
import router from '@/router'
export const useCacheListStore = defineStore('cacheList', () => {
  const cacheList = ref<string[]>([])
  const initCacheList = () => {
    router.options.routes.length > 0 && router.options.routes.forEach(route => {
      if (route.meta && route.meta.keepAlive) {
        addCacheList(route.name as string)
      }
    })
  }
  const getKeepAliveList = computed(() => cacheList)

  const addCacheList = (name: string) => {
    if (!cacheList.value.includes(name)) {
      cacheList.value.push(name)
    }
  }
  const removeCacheList = (name: string) => {
    const index = cacheList.value.indexOf(name)
    if (index !== -1) {
      cacheList.value.splice(index, 1)
    }
  }
  const removeAllCacheList = () => {
    cacheList.value = []
  }
  return { initCacheList, getKeepAliveList, addCacheList, removeCacheList,removeAllCacheList }
})
```

在你任何需要的地方调用吧！