# 请求取消机制改进方案

## 一、为什么需要请求取消？

### 当前问题：

```javascript
// 场景 1：用户快速切换页面
用户在列表页 → 点击详情 → 立即返回 → 再次点击另一个详情
// 问题：第一个详情的请求还在进行，浪费资源

// 场景 2：搜索防抖
用户输入"项目" → 发起请求 → 继续输入"项目A" → 又发起请求
// 问题：第一个请求应该取消，但还在执行

// 场景 3：分页加载
用户快速滑动列表，触发多次加载
// 问题：前面的请求还没完成，又触发新的请求
```

### 影响：

- ❌ **浪费流量**：无用的请求消耗用户流量
- ❌ **内存泄漏**：页面已销毁，但请求回调还在执行
- ❌ **数据错乱**：后发的请求先返回，导致数据不一致
- ❌ **性能问题**：大量并发请求影响性能

---

## 二、实现方案

### 方案 1：基于 uni.request 的 requestTask（推荐）

```javascript
// src/api/core/request.js

// 存储所有进行中的请求
const requestTaskMap = new Map()

/**
 * 生成请求唯一标识
 */
const generateRequestId = ({ url, method, data }) => {
  return `${method}_${url}_${JSON.stringify(data)}`
}

/**
 * 取消指定请求
 */
export function cancelRequest(requestId) {
  const task = requestTaskMap.get(requestId)
  if (task) {
    task.abort()
    requestTaskMap.delete(requestId)
    console.log(`[请求取消] ${requestId}`)
  }
}

/**
 * 取消所有请求
 */
export function cancelAllRequests() {
  requestTaskMap.forEach((task, requestId) => {
    task.abort()
    console.log(`[请求取消] ${requestId}`)
  })
  requestTaskMap.clear()
}

/**
 * 取消指定 URL 的所有请求
 */
export function cancelRequestsByUrl(url) {
  const keysToDelete = []
  
  requestTaskMap.forEach((task, requestId) => {
    if (requestId.includes(url)) {
      task.abort()
      keysToDelete.push(requestId)
      console.log(`[请求取消] ${requestId}`)
    }
  })
  
  keysToDelete.forEach(key => requestTaskMap.delete(key))
}

/**
 * 核心请求方法（改进版）
 */
const request = (options, retryCount = 0) => {
  const MAX_RETRY_COUNT = 1
  const RETRY_DELAY = 100

  return new Promise(async (resolve, reject) => {
    try {
      const processedOptions = await requestInterceptor(options)
      const requestId = generateRequestId(processedOptions)

      // ✅ 如果设置了 cancelPrevious，取消之前的同类请求
      if (options.cancelPrevious) {
        cancelRequest(requestId)
      }

      // 防重复请求
      if (pendingRequests.has(requestId) && !options._tokenRefreshed) {
        return
      }
      pendingRequests.add(requestId)

      // Loading 处理
      if (options.loading !== false) {
        showLoading(options.loadingText)
      }

      // ✅ 创建 requestTask
      const requestTask = uni.request({
        ...processedOptions,
        success: async (response) => {
          try {
            // 清理
            requestTaskMap.delete(requestId)
            pendingRequests.delete(requestId)
            
            response.config = processedOptions
            const result = await responseInterceptor(response)

            if (result?.code === 'TOKEN_REFRESHED') {
              return handleTokenRefreshRetry(result.config, retryCount, resolve, reject)
            }

            if (result && result.code === 0) {
              resolve(result)
            } else {
              if (options.showError !== false) {
                showError(result.msg || result.message || '请求失败')
              }
              reject(result)
            }
          } catch (error) {
            if (options.showError !== false && !error.silent) {
              showError(error.message || '请求处理失败')
            }
            reject(error)
          } finally {
            if (options.loading !== false) hideLoading()
          }
        },

        fail: (error) => {
          // 清理
          requestTaskMap.delete(requestId)
          pendingRequests.delete(requestId)
          
          if (options.loading !== false) hideLoading()

          // ✅ 区分取消和失败
          if (error.errMsg && error.errMsg.includes('abort')) {
            console.log(`[请求已取消] ${requestId}`)
            reject({ code: 'REQUEST_CANCELLED', message: '请求已取消' })
          } else {
            const errorResult = createNetworkError(error)
            if (options.showError !== false) {
              showError(errorResult.msg)
            }
            reject(errorResult)
          }
        },
      })

      // ✅ 保存 requestTask
      requestTaskMap.set(requestId, requestTask)

    } catch (error) {
      const errorResult = createInterceptorError(error)
      if (options.showError !== false && !error.silent) {
        showError(errorResult.msg)
      }
      reject(errorResult)
    }
  })
}

// 便捷方法（添加取消支持）
export const http = {
  get: (url, data = {}, options = {}) => 
    request({ url, method: 'GET', data, ...options }),
  
  post: (url, data = {}, options = {}) => 
    request({ url, method: 'POST', data, ...options }),
  
  put: (url, data = {}, options = {}) => 
    request({ url, method: 'PUT', data, ...options }),
  
  delete: (url, data = {}, options = {}) => 
    request({ url, method: 'DELETE', data, ...options }),
  
  request,
  
  // ✅ 导出取消方法
  cancelRequest,
  cancelAllRequests,
  cancelRequestsByUrl,
}

export default http
```

---

## 三、使用场景

### 场景 1：页面卸载时取消请求

```vue
<!-- src/pages/project/detail.vue -->

<script setup>
import { onUnmounted } from 'vue'
import { http } from '@/api/core/request.js'
import { projectApi } from '@/api/project.js'

const projectId = ref(123)

// 获取项目详情
const getDetail = async () => {
  const res = await projectApi.getProjectDetail({ id: projectId.value })
  // 处理数据...
}

onMounted(() => {
  getDetail()
})

// ✅ 页面卸载时取消所有请求
onUnmounted(() => {
  http.cancelRequestsByUrl('/app-api/cpm/project/detail')
})
</script>
```

---

### 场景 2：搜索防抖 + 取消前一个请求

```vue
<!-- src/pages/project/index.vue -->

<script setup>
import { ref, watch } from 'vue'
import { projectApi } from '@/api/project.js'

const keyword = ref('')
const projectList = ref([])

// ✅ 搜索（自动取消前一个请求）
const searchProjects = async () => {
  const res = await projectApi.getProjectList(
    { keyword: keyword.value },
    { cancelPrevious: true }  // 取消前一个相同的请求
  )
  projectList.value = res.data.list
}

// 防抖 + 取消
let timer = null
watch(keyword, (newVal) => {
  clearTimeout(timer)
  timer = setTimeout(() => {
    searchProjects()
  }, 300)
})
</script>

<template>
  <view>
    <input v-model="keyword" placeholder="搜索项目" />
    <view v-for="item in projectList" :key="item.id">
      {{ item.name }}
    </view>
  </view>
</template>
```

---

### 场景 3：下拉刷新时取消旧请求

```vue
<!-- src/pages/project/index.vue -->

<script setup>
import { ref } from 'vue'
import { http } from '@/api/core/request.js'
import { projectApi } from '@/api/project.js'

const projectList = ref([])
const loading = ref(false)

// 加载项目列表
const loadProjects = async () => {
  loading.value = true
  try {
    const res = await projectApi.getProjectList({ page: 1 })
    projectList.value = res.data.list
  } finally {
    loading.value = false
  }
}

// ✅ 下拉刷新
const onRefresh = () => {
  // 取消正在进行的请求
  http.cancelRequestsByUrl('/app-api/cpm/project/list')
  
  // 重新加载
  loadProjects()
}

onMounted(() => {
  loadProjects()
})
</script>
```

---

### 场景 4：手动取消请求

```vue
<script setup>
import { ref } from 'vue'
import { http } from '@/api/core/request.js'

const loading = ref(false)

// 开始长时间任务
const startTask = async () => {
  loading.value = true
  try {
    const res = await http.post('/api/long-task', { data: '...' })
    console.log('任务完成', res)
  } catch (error) {
    if (error.code === 'REQUEST_CANCELLED') {
      console.log('任务已取消')
    }
  } finally {
    loading.value = false
  }
}

// ✅ 手动取消
const cancelTask = () => {
  http.cancelRequestsByUrl('/api/long-task')
  loading.value = false
}
</script>

<template>
  <view>
    <button @click="startTask" :disabled="loading">开始任务</button>
    <button @click="cancelTask" v-if="loading">取消任务</button>
  </view>
</template>
```

---

## 四、高级用法：请求分组管理

```javascript
// src/utils/requestGroup.js

import { http } from '@/api/core/request.js'

/**
 * 请求分组管理器
 * 用于管理一组相关的请求，可以批量取消
 */
class RequestGroup {
  constructor(name) {
    this.name = name
    this.requests = new Set()
  }

  /**
   * 添加请求到分组
   */
  add(requestPromise, requestId) {
    this.requests.add({ promise: requestPromise, id: requestId })
    
    // 请求完成后自动移除
    requestPromise.finally(() => {
      this.requests.delete({ promise: requestPromise, id: requestId })
    })
    
    return requestPromise
  }

  /**
   * 取消分组内所有请求
   */
  cancelAll() {
    this.requests.forEach(({ id }) => {
      http.cancelRequest(id)
    })
    this.requests.clear()
    console.log(`[请求分组] ${this.name} 已取消所有请求`)
  }

  /**
   * 获取进行中的请求数量
   */
  get size() {
    return this.requests.size
  }
}

// 使用示例
export const projectRequestGroup = new RequestGroup('项目模块')
export const orderRequestGroup = new RequestGroup('订单模块')
```

**使用：**

```vue
<script setup>
import { onUnmounted } from 'vue'
import { projectRequestGroup } from '@/utils/requestGroup.js'
import { projectApi } from '@/api/project.js'

const loadData = async () => {
  // 将请求添加到分组
  const req1 = projectApi.getProjectList({ page: 1 })
  const req2 = projectApi.getProjectDetail({ id: 123 })
  
  projectRequestGroup.add(req1, 'project-list')
  projectRequestGroup.add(req2, 'project-detail')
  
  const [list, detail] = await Promise.all([req1, req2])
  // 处理数据...
}

// ✅ 页面卸载时取消分组内所有请求
onUnmounted(() => {
  projectRequestGroup.cancelAll()
})
</script>
```

---

## 五、配置选项

### 在 API 层面配置

```javascript
// src/api/project.js

export const projectApi = {
  // 普通请求（不取消）
  getProjectList: (params) => 
    http.post('/app-api/cpm/project/list', params),

  // 搜索请求（自动取消前一个）
  searchProjects: (params) => 
    http.post('/app-api/cpm/project/search', params, {
      cancelPrevious: true,  // ✅ 自动取消前一个相同请求
    }),

  // 详情请求（页面切换时取消）
  getProjectDetail: (params, options = {}) => 
    http.post('/app-api/cpm/project/detail', params, {
      ...options,
      // 可以在调用时决定是否取消
    }),
}
```

---

## 六、注意事项

### ⚠️ 什么时候不应该取消请求？

1. **提交类请求**：如创建订单、支付等，不应该取消
2. **上传文件**：文件上传中途取消可能导致数据不完整
3. **关键业务**：如登录、Token 刷新等

### ✅ 什么时候应该取消请求？

1. **查询类请求**：列表、详情、搜索等
2. **页面切换**：用户离开页面时
3. **重复操作**：用户快速点击、输入等
4. **分页加载**：快速滑动触发多次加载

---

## 七、优势总结

### ✅ 改进后的优势：

1. **节省流量**：取消无用请求，减少流量消耗
2. **避免内存泄漏**：页面销毁时自动取消请求
3. **数据一致性**：避免旧请求覆盖新数据
4. **性能提升**：减少并发请求数量
5. **用户体验**：响应更快，操作更流畅

### 📊 对比：

| 维度 | 改进前 | 改进后 |
|------|--------|--------|
| 请求管理 | 无法取消 | 支持取消 |
| 内存泄漏 | 可能存在 | 自动清理 |
| 数据一致性 | 可能错乱 | 保证一致 |
| 流量消耗 | 浪费 | 节省 |
| 开发体验 | 需手动处理 | 自动化 |
