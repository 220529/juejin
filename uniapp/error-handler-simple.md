# 错误处理优化方案（简化版）

## 一、当前问题分析

### 你的项目现状（已经做得不错）：

```javascript
// ✅ 已有的优点：
1. 错误节流（相同错误 1.5 秒内只提示一次）
2. 静默错误（401 等特殊错误不显示 toast）
3. 统一的 showError 函数
4. Loading 管理

// ❌ 真正的问题：
1. 错误提示信息不够友好（直接显示后端返回的 msg）
2. 没有错误码映射（后端返回 "操作失败" 用户不知道什么原因）
3. 业务代码中还是要写很多 try-catch
```

---

## 二、轻量级改进方案

### 只做两件事：

1. **错误码映射表**（让提示更友好）
2. **简化业务代码**（减少重复的 try-catch）

---

## 三、具体实施

### 1. 创建错误码映射表（可选）

```javascript
// src/config/errorCode.js

/**
 * 错误码映射表
 * 只映射常见的、需要特殊处理的错误码
 * 其他错误直接使用后端返回的 msg
 */
export const ERROR_CODE_MAP = {
  // 认证相关
  401: '登录已过期，请重新登录',
  403: '无权限访问',
  
  // 业务相关（根据你的后端实际错误码添加）
  10001: '项目不存在或已删除',
  10002: '材料库存不足',
  10003: '订单状态异常，无法操作',
  
  // 文件上传
  20001: '文件格式不支持',
  20002: '文件大小超过 10MB',
}

/**
 * 获取友好的错误提示
 */
export function getErrorMessage(error) {
  // 1. 优先使用映射表
  if (error.code && ERROR_CODE_MAP[error.code]) {
    return ERROR_CODE_MAP[error.code]
  }
  
  // 2. 使用后端返回的消息
  if (error.msg || error.message) {
    return error.msg || error.message
  }
  
  // 3. 默认提示
  return '操作失败，请稍后重试'
}
```

---

### 2. 修改 `request.js`（只改一行）

```javascript
// src/api/core/request.js

import { getErrorMessage } from '@/config/errorCode.js'

// 原有代码...

// ✅ 只需要改这一行
const showError = (message) => {
  const now = Date.now()
  
  // 如果传入的是 error 对象，使用映射表
  const errorMsg = typeof message === 'object' 
    ? getErrorMessage(message) 
    : message
  
  if (lastErrorMessage === errorMsg && now - lastErrorTime <= 1500) {
    return
  }

  uni.hideLoading()

  setTimeout(() => {
    uni.showToast({
      title: errorMsg,
      icon: 'error',
      duration: 3000,
      mask: true,
    })
  }, 50)

  lastErrorTime = now
  lastErrorMessage = errorMsg
}

// 然后在调用时传入完整的 error 对象
success: async (response) => {
  // ...
  if (result && result.code === 0) {
    resolve(result)
  } else {
    if (options.showError !== false) {
      showError(result)  // ✅ 传入完整对象，而不是 result.msg
    }
    reject(result)
  }
}
```

---

### 3. 业务代码简化（可选）

#### 方式 1：不需要 try-catch（推荐）

```javascript
// ❌ 之前：每个请求都要 try-catch
async function loadProjects() {
  try {
    const res = await projectApi.getProjectList({ page: 1 })
    projectList.value = res.data.list
  } catch (error) {
    // 错误已经在 request.js 中处理了，这里不需要做什么
  }
}

// ✅ 之后：直接调用，错误自动提示
async function loadProjects() {
  const res = await projectApi.getProjectList({ page: 1 })
  projectList.value = res.data.list
}
```

#### 方式 2：需要自定义处理

```javascript
// 场景：删除操作，成功后需要刷新列表
async function deleteProject(id) {
  try {
    await projectApi.deleteProject({ id })
    uni.showToast({ title: '删除成功', icon: 'success' })
    loadProjects() // 刷新列表
  } catch (error) {
    // 错误已经自动提示了，这里只需要处理业务逻辑
    console.log('删除失败，不刷新列表')
  }
}
```

#### 方式 3：需要静默处理

```javascript
// 场景：统计类请求，失败了不需要提示用户
async function reportAnalytics() {
  await http.post('/api/analytics', data, {
    showError: false,  // ✅ 不显示错误提示
    loading: false,    // 不显示 loading
  })
}
```

---

## 四、对比

### 改进前 vs 改进后

```javascript
// ==================== 改进前 ====================

// 1. 错误提示不友好
showError(result.msg)  // 显示："操作失败"（用户不知道什么原因）

// 2. 业务代码重复
try {
  const res = await api.xxx()
  // 处理数据
} catch (error) {
  // 每个地方都要写
}

// ==================== 改进后 ====================

// 1. 错误提示友好
showError(result)  // 显示："材料库存不足"（清楚明了）

// 2. 业务代码简洁
const res = await api.xxx()
// 直接处理数据，错误自动提示
```

---

## 五、是否需要错误上报？

### 我的建议：**暂时不需要**

**理由：**

1. **你没有错误收集服务**
   - 搭建错误收集服务需要额外成本
   - 小程序本身有微信后台的错误日志

2. **微信小程序自带错误监控**
   - 微信开发者工具 → 运维中心 → 错误日志
   - 可以看到线上的 JS 错误、网络错误

3. **过度设计**
   - 如果项目不大，错误上报的价值不高
   - 增加代码复杂度，维护成本高

### 什么时候需要错误上报？

- ✅ 用户量大（日活 > 1万）
- ✅ 需要实时监控线上问题
- ✅ 需要分析错误趋势
- ✅ 有专门的运维团队

### 如果以后需要，怎么加？

```javascript
// src/config/errorCode.js

/**
 * 错误上报（可选，默认关闭）
 */
export function reportError(error) {
  // 开发环境不上报
  if (import.meta.env.DEV) return
  
  // 简单的上报（发送到自己的服务器）
  uni.request({
    url: '/api/error/report',
    method: 'POST',
    data: {
      message: error.message,
      code: error.code,
      url: error.url,
      timestamp: Date.now(),
    },
  })
}

// 在 request.js 中调用
if (options.showError !== false) {
  showError(result)
  reportError(result)  // ✅ 需要时再加
}
```

---

## 六、总结

### ✅ 推荐做的（轻量级）：

1. **创建错误码映射表**（`src/config/errorCode.js`）
   - 只映射常见的、需要特殊处理的错误码
   - 其他错误直接使用后端返回的 msg

2. **修改 `showError` 函数**（`src/api/core/request.js`）
   - 支持传入 error 对象
   - 自动使用映射表

3. **简化业务代码**
   - 不需要每个地方都 try-catch
   - 错误自动提示

### ❌ 不推荐做的（过度设计）：

1. ~~错误上报服务~~（暂时不需要）
2. ~~复杂的错误分类~~（BusinessError、NetworkError 等）
3. ~~错误回调机制~~（handleError 的 callback 参数）
4. ~~错误日志系统~~（微信后台已有）

---

## 七、实施步骤

### 只需要 3 步：

```bash
# 1. 创建错误码映射表
touch src/config/errorCode.js
# 复制上面的代码

# 2. 修改 request.js
# 只改 showError 函数，支持传入 error 对象

# 3. 测试
# 触发一个错误，看提示是否友好
```

---

## 八、代码量对比

### 改进前：
- 0 行新增代码
- 业务代码中大量 try-catch

### 改进后（轻量级）：
- **约 30 行新增代码**（errorCode.js）
- **约 5 行修改代码**（request.js）
- 业务代码减少 50% 的 try-catch

### 我之前的方案（过度设计）：
- ~~约 200 行新增代码~~
- ~~复杂的错误处理逻辑~~
- ~~错误上报、分类、回调等~~

---

## 九、最终建议

### 对于你的项目：

1. **环境变量管理**：如果变量少，合并到一个文件也可以 ✅
2. **错误处理**：只做错误码映射表，不做错误上报 ✅
3. **请求取消**：这个是真正有价值的，建议做 ✅

### 优先级：

1. **请求取消机制**（最有价值，解决实际问题）
2. **错误码映射表**（轻量级，提升用户体验）
3. ~~环境变量分离~~（变量少的话不需要）

---

**总结：我之前的方案确实过度设计了，这个简化版更适合你的项目！** 🎯
