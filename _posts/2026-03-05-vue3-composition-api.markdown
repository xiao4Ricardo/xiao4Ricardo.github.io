---
layout:     post
title:      "Vue 3 Composition API 完全指南"
subtitle:   "Mastering Reactivity, Composables, and Modern Vue Patterns"
date:       2026-03-05 11:00:00
author:     "Tony L."
header-img: "img/post-bg-re-vs-ng2.jpg"
tags:
    - Vue
    - TypeScript
    - Frontend
    - Software Engineering
---

## 为什么选择 Composition API？

Vue 3 的 Composition API 是对 Options API 的全面升级。它解决了 Options API 在大型组件中的核心痛点：

**Options API 的问题：**
- 相关逻辑分散在 `data`、`computed`、`methods`、`watch` 等选项中
- 组件越大，代码越难维护（"功能碎片化"）
- 逻辑复用依赖 Mixins，容易命名冲突和来源不明

**Composition API 的优势：**
- 按功能组织代码，相关逻辑放在一起
- 通过 Composable 函数实现优雅的逻辑复用
- 更好的 TypeScript 类型推导
- 更灵活的代码组织方式

## script setup 基础

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

// 响应式状态
const count = ref(0)
const message = ref('Hello')

// 计算属性
const doubleCount = computed(() => count.value * 2)

// 方法
function increment() {
  count.value++
}

// 生命周期
onMounted(() => {
  console.log('Component mounted')
})
</script>

<template>
  <div>
    <p>{{ message }}</p>
    <p>Count: {{ count }}, Double: {{ doubleCount }}</p>
    <button @click="increment">+1</button>
  </div>
</template>
```

`<script setup>` 是 Composition API 的语法糖，所有顶层变量和函数自动暴露给模板。

## 响应式系统深入

### ref vs reactive

```typescript
import { ref, reactive } from 'vue'

// ref：适合基本类型和需要整体替换的值
const count = ref(0)
const user = ref<User | null>(null)
count.value++        // 需要 .value
user.value = newUser // 可以整体替换

// reactive：适合对象，无需 .value
const state = reactive({
  count: 0,
  items: [] as string[]
})
state.count++        // 直接访问
state.items.push('new')
```

**经验法则**：大多数情况下用 `ref`。它更一致（总是用 `.value`），在模板中自动解包，且可以整体替换引用。

### watch vs watchEffect

```typescript
import { ref, watch, watchEffect } from 'vue'

const searchQuery = ref('')
const page = ref(1)

// watch：明确指定观察目标，可以获取新旧值
watch(searchQuery, (newVal, oldVal) => {
  console.log(`Search changed from "${oldVal}" to "${newVal}"`)
  page.value = 1 // 重置页码
})

// 观察多个源
watch([searchQuery, page], ([newQuery, newPage]) => {
  fetchResults(newQuery, newPage)
})

// watchEffect：自动追踪依赖，立即执行
watchEffect(() => {
  // 自动追踪 searchQuery 和 page 的变化
  fetchResults(searchQuery.value, page.value)
})
```

### shallowRef 和 triggerRef

对于大型对象（如表格数据），使用 `shallowRef` 避免深层响应式的性能开销：

```typescript
import { shallowRef, triggerRef } from 'vue'

const tableData = shallowRef<Row[]>([])

// 直接修改不会触发更新
tableData.value.push(newRow) // 不触发

// 需要手动触发或整体替换
tableData.value = [...tableData.value, newRow] // 触发
// 或
triggerRef(tableData) // 强制触发
```

## Composables：逻辑复用的最佳实践

Composable 是以 `use` 开头的函数，封装和复用有状态的逻辑。

### 基础示例：useFetch

```typescript
// composables/useFetch.ts
import { ref, watchEffect, type Ref } from 'vue'

interface UseFetchReturn<T> {
  data: Ref<T | null>
  error: Ref<string | null>
  loading: Ref<boolean>
  refetch: () => Promise<void>
}

export function useFetch<T>(url: Ref<string> | string): UseFetchReturn<T> {
  const data = ref<T | null>(null) as Ref<T | null>
  const error = ref<string | null>(null)
  const loading = ref(false)

  async function fetchData() {
    loading.value = true
    error.value = null
    try {
      const urlValue = typeof url === 'string' ? url : url.value
      const response = await fetch(urlValue)
      if (!response.ok) throw new Error(`HTTP ${response.status}`)
      data.value = await response.json()
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Unknown error'
    } finally {
      loading.value = false
    }
  }

  watchEffect(() => {
    fetchData()
  })

  return { data, error, loading, refetch: fetchData }
}
```

使用：

```vue
<script setup lang="ts">
import { useFetch } from '@/composables/useFetch'

interface User {
  id: number
  name: string
  email: string
}

const { data: users, loading, error } = useFetch<User[]>('/api/users')
</script>

<template>
  <div v-if="loading">Loading...</div>
  <div v-else-if="error">Error: {{ error }}</div>
  <ul v-else>
    <li v-for="user in users" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

### 实用 Composable：useDebounce

```typescript
// composables/useDebounce.ts
import { ref, watch, type Ref } from 'vue'

export function useDebounce<T>(source: Ref<T>, delay = 300): Ref<T> {
  const debounced = ref(source.value) as Ref<T>
  let timeout: ReturnType<typeof setTimeout>

  watch(source, (newVal) => {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      debounced.value = newVal
    }, delay)
  })

  return debounced
}
```

### Composable 设计原则

1. **命名以 `use` 开头**：`useFetch`, `useAuth`, `useDarkMode`
2. **参数接受 ref 或普通值**：使用 `toValue()` 或 `unref()` 兼容两者
3. **返回 ref 而非 reactive**：让调用者可以解构
4. **处理清理逻辑**：在 `onUnmounted` 中清理定时器、事件监听等
5. **保持单一职责**：一个 composable 只做一件事

## TypeScript 集成

### Props 类型定义

```typescript
// 使用泛型 defineProps
interface Props {
  title: string
  count?: number
  items: string[]
  status: 'active' | 'inactive'
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
  status: 'active'
})
```

### Emit 类型定义

```typescript
const emit = defineEmits<{
  'update:modelValue': [value: string]
  'submit': [data: FormData]
  'delete': [id: string]
}>()
```

### 模板 Ref 类型

```typescript
import { ref, type ComponentPublicInstance } from 'vue'

const inputRef = ref<HTMLInputElement | null>(null)
const childRef = ref<InstanceType<typeof ChildComponent> | null>(null)

onMounted(() => {
  inputRef.value?.focus()
})
```

## Pinia 状态管理

Pinia 是 Vue 3 官方推荐的状态管理库：

```typescript
// stores/useUserStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import { userApi } from '@/api/user'

export const useUserStore = defineStore('user', () => {
  // State
  const user = ref<User | null>(null)
  const token = ref(localStorage.getItem('token'))

  // Getters
  const isAuthenticated = computed(() => !!token.value)
  const isOwner = computed(() => user.value?.role === 'OWNER')

  // Actions
  async function login(credentials: LoginRequest) {
    const response = await userApi.login(credentials)
    token.value = response.data.token
    localStorage.setItem('token', response.data.token)
    user.value = response.data.user
  }

  function logout() {
    token.value = null
    user.value = null
    localStorage.removeItem('token')
  }

  return { user, token, isAuthenticated, isOwner, login, logout }
})
```

## 性能优化

### v-memo

缓存模板子树，只在依赖变化时重新渲染：

```vue
<div v-for="item in list" :key="item.id" v-memo="[item.id === selected]">
  <!-- 只有当 item 被选中/取消选中时才重新渲染 -->
  <ExpensiveComponent :data="item" :selected="item.id === selected" />
</div>
```

### defineAsyncComponent

按需加载组件：

```typescript
import { defineAsyncComponent } from 'vue'

const HeavyChart = defineAsyncComponent(() =>
  import('@/components/HeavyChart.vue')
)
```

## 总结

Composition API + TypeScript + Pinia 是 Vue 3 开发的黄金组合。Composable 提供了优雅的逻辑复用方式，TypeScript 保证了类型安全，Pinia 简化了状态管理。掌握这些工具，你就能构建出可维护、可扩展的现代前端应用。

## References

1. Vue 3 Official Documentation. https://vuejs.org
2. Pinia Documentation. https://pinia.vuejs.org
3. VueUse Collection. https://vueuse.org
