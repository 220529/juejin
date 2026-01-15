# 统一错误处理机制改进方案

## 一、创建错误处理工具

### 1. 错误码映射表

```javascript
// src/utils/errorHandler.js

/**
 * 业务错误码映射表
 * 根据后端返回的 code 映射为用户友好的提示
 */
export const ERROR_CODE_MAP = {
  // 通用错误 1xxx
  1000: '系统繁忙，请稍后重试',
  1001: '参数错误',
  1002: '数据不存在',
  1003: '操作失败',
  
  // 认证相关 2xxx
  2001: '登录已过期，请重新登录',
  2002: '无权限访问',
  2003: '账号已被禁用',
  
  // 业务相关 3xxx
  3001: '项目不存在',
  3002: '材料库存不足',
  3003: '订单状态异常',
  
  // 文件上传 4xxx
  4001: '文件格式不支持',
  4002: '文件大小超过限制',
  4003: '上传失败，请重试',
}

/**
 * HTTP 状态码映射
 */
export const HTTP_STATUS_MAP = {
  400: '请求参数错误',
  401: '未授权，请重新登录',
  403: '拒绝访问',
  404: '请求的资源不存在',
  500: '服务器内部错误',
  502: '网关错误',
  503: '服务暂时不可用',
  504: '网关超时',
}

/**
 * 获取错误提示信息
 */
export function getErrorMessage(error) {
  // 优先使用业务错误码
  if (error.code && ERROR_CODE_MAP[error.code]) {
    return ERROR_CODE_MAP[error.code]
  }
  
  // 其次使用 HTTP 状态码
  if (error.statusCode && HTTP_STATUS_MAP[error.statusCode]) {
    return HTTP_STATUS_MAP[error.statusCode]
  }
  
  // 最后使用后端返回的消息
  return error.msg || error.message || '操作失败，请稍后重试'
}

/**
 * 统一错误处理函数
 */
export function handleError(error, options = {}) {
  const {
    silent = false,        // 是否静默（不显示提示）
    toast = true,          // 是否显示 toast
    modal = false,         // 是否显示 modal
    callback = null,       // 错误处理回调
    report = true,         // 是否上报错误
  } = options
  
  // 静默错误直接返回
  if (silent || error.silent) {
    return
  }
  
  const message = getErrorMessage(error)
  
  // 显示 Modal
  if (modal) {
    uni.showModal({
      title: '提示',
      content: message,
      showCancel: false,
      confirmText: '我知道了',
    })
  }
  // 显示 Toast
  else if (toast) {
    uni.showToast({
      title: message,
      icon: 'none',
      duration: 3000,
      mask: true,
    })
  }
  
  // 执行回调
  if (callback && typeof callback === 'function') {
    callback(error)
  }
  
  // 上报错误（可选）
  if (report && shouldReportError(error)) {
    reportError(error)
  }
}

/**
 * 判断是否需要上报错误
 */
function shouldReportError(error) {
  // 401、403 等认证错误不上报
  if ([401, 403].includes(error.statusCode)) {
    return false
  }
  
  // 业务错误码 < 3000 不上报（认证、参数等常规错误）
  if (error.code && error.code < 3000) {
    return false
  }
  
  return true
}

/**
 * 错误上报（可接入第三方监控平台）
 */
function reportError(error) {
  // 开发环境只打印，不上报
  if (import.meta.env.MODE === 'development') {
    console.error('[错误上报]', error)
    return
  }
  
  // 生产环境上报到监控平台
  try {
    // 示例：上报到自己的服务器
    uni.request({
      url: '/api/error/report',
      method: 'POST',
      data: {
        message: error.message,
        code: error.code,
        stack: error.stack,
        url: error.url,
        timestamp: Date.now(),
        userAgent: uni.getSystemInfoSync(),
      },
    })
    
    // 或者接入第三方监控（如 Sentry）
    // Sentry.captureException(error)
  } catch (e) {
    console.error('错误上报失败:', e)
  }
}

/**
 * 创建业务错误
 */
export class BusinessError extends Error {
  constructor(code, message, data = null) {
    super(message)
    this.name = 'BusinessError'
    this.code = code
    this.data = data
  }
}

/**
 * 创建网络错误
 */
export class NetworkError extends Error {
  constructor(message, statusCode = 0) {
    super(message)
    this.name = 'NetworkError'
    this.statusCode = statusCode
  }
}
```

---

## 二、集成到请求拦截器

### 修改 `src/api/core/request.js`

```javascript
import { handleError, NetworkError, BusinessError } from '@/utils/errorHandler.js'

// 原有代码...

const request = (options, retryCount = 0) => {
  return new Promise(async (resolve, reject) => {
    try {
      // ... 原有逻辑

      uni.request({
        ...processedOptions,
        success: async (response) => {
          try {
            response.config = processedOptions
            const result = await responseInterceptor(response)

            if (result?.code === 'TOKEN_REFRESHED') {
              return handleTokenRefreshRetry(result.config, retryCount, resolve, reject)
            }

            if (result && result.code === 0) {
              resolve(result)
            } else {
              // ✅ 使用统一错误处理
              const error = new BusinessError(result.code, result.msg || result.message, result.data)
              
              if (options.showError !== false) {
                handleError(error, {
                  toast: true,
                  report: true,
                })
              }
              
              reject(error)
            }
          } catch (error) {
            // ✅ 使用统一错误处理
            if (options.showError !== false && !error.silent) {
              handleError(error, {
                toast: true,
                report: true,
              })
            }
            reject(error)
          } finally {
            pendingRequests.delete(requestKey)
            if (options.loading !== false) hideLoading()
          }
        },

        fail: (error) => {
          pendingRequests.delete(requestKey)
          if (options.loading !== false) hideLoading()

          // ✅ 使用统一错误处理
          const networkError = new NetworkError('网络连接失败', 0)
          
          if (options.showError !== false) {
            handleError(networkError, {
              toast: true,
              report: true,
            })
          }
          
          reject(networkError)
        },
      })
    } catch (error) {
      // ✅ 使用统一错误处理
      if (options.showError !== false && !error.silent) {
        handleError(error, {
          toast: true,
          report: false, // 拦截器错误不上报
        })
      }
      reject(error)
    }
  })
}
```

---

## 三、业务代码中使用

### 示例 1：普通请求

```javascript
// 之前
try {
  const res = await projectApi.getProjectList({ page: 1 })
  // 处理数据
} catch (error) {
  uni.showToast({ title: '获取项目列表失败', icon: 'error' })
}

// 之后（自动处理错误，无需手动 catch）
const res = await projectApi.getProjectList({ page: 1 })
// 直接处理数据，错误会自动提示
```

### 示例 2：需要自定义错误处理

```javascript
import { handleError } from '@/utils/errorHandler.js'

try {
  const res = await projectApi.deleteProject({ id: 123 })
  uni.showToast({ title: '删除成功', icon: 'success' })
} catch (error) {
  // 自定义错误处理
  handleError(error, {
    modal: true,  // 使用 modal 而不是 toast
    callback: (err) => {
      // 删除失败后的额外处理
      console.log('删除失败，回滚操作')
    }
  })
}
```

### 示例 3：静默请求（不显示错误）

```javascript
// 统计类请求，失败了不需要提示用户
const res = await http.post('/api/analytics', data, {
  showError: false,  // 不显示错误提示
  loading: false,    // 不显示 loading
})
```

---

## 四、特殊场景处理

### 1. 表单验证错误

```javascript
// src/utils/errorHandler.js 中添加

/**
 * 处理表单验证错误
 */
export function handleValidationError(errors) {
  if (!errors || errors.length === 0) return
  
  // 只显示第一个错误
  const firstError = errors[0]
  uni.showToast({
    title: firstError.message,
    icon: 'none',
    duration: 2000,
  })
}

// 使用
import { handleValidationError } from '@/utils/errorHandler.js'

const errors = []
if (!form.name) errors.push({ field: 'name', message: '请输入项目名称' })
if (!form.address) errors.push({ field: 'address', message: '请输入项目地址' })

if (errors.length > 0) {
  handleValidationError(errors)
  return
}
```

### 2. 文件上传错误

```javascript
// src/api/upload.js

import { handleError, BusinessError } from '@/utils/errorHandler.js'

export const uploadImage = (filePath) => {
  return new Promise((resolve, reject) => {
    uni.uploadFile({
      url: '/api/upload/image',
      filePath,
      name: 'file',
      success: (res) => {
        try {
          const data = JSON.parse(res.data)
          if (data.code === 0) {
            resolve(data.data)
          } else {
            const error = new BusinessError(data.code, data.msg)
            handleError(error, { toast: true })
            reject(error)
          }
        } catch (e) {
          const error = new BusinessError(4003, '上传失败，请重试')
          handleError(error, { toast: true })
          reject(error)
        }
      },
      fail: (error) => {
        const uploadError = new BusinessError(4003, '上传失败，请检查网络')
        handleError(uploadError, { toast: true })
        reject(uploadError)
      },
    })
  })
}
```

---

## 五、优势总结

### ✅ 改进后的优势：

1. **统一管理**：所有错误码和提示信息集中管理，易于维护
2. **用户友好**：根据错误类型显示合适的提示（toast/modal）
3. **开发便捷**：业务代码无需手动处理错误，自动提示
4. **可追溯**：错误自动上报，方便排查线上问题
5. **灵活配置**：支持静默、自定义回调等多种场景

### 📊 对比：

| 维度 | 改进前 | 改进后 |
|------|--------|--------|
| 错误提示 | 分散在各处，不统一 | 集中管理，统一风格 |
| 错误码 | 硬编码在代码中 | 统一映射表 |
| 错误上报 | 无 | 自动上报 |
| 代码量 | 每个请求都要写 catch | 自动处理，代码更简洁 |
| 可维护性 | 低 | 高 |
