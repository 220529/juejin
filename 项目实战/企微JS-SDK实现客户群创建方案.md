# 企微 JS-SDK 实现客户群创建方案：从调研到落地

## 背景

在 ToB 业务场景中，客户到店或下单后，需要将客户与服务人员拉到同一个企业微信群中，便于后续沟通与服务跟进。

但现实中存在一个痛点：**员工通常添加的是客户的个人微信，拉的是普通微信群**。一旦员工离职，很容易导致客户流失。

核心诉求：**一客一群（或一单一群），群归企业，员工离职不丢客**。

## 调研过程

### 服务端 API 的局限性

最初的想法是通过业务系统调用服务端接口，自动创建客户群并邀请成员。但调研后发现几个关键限制：

| 限制项 | 说明 |
|--------|------|
| 无创建客户群 API | 官方只有创建内部群的接口（`appchat`），客户属于外部人员，无法加入 |
| 无法服务端拉人 | 服务端不支持直接将外部联系人加入任何群（无论客户群还是内部群） |
| 只能扫码进群 | 外部联系人只能通过"进群方式"（二维码/小程序按钮）加入客户群 |
| 客户群需预建 | 创建客户群必须在企微客户端由具备"客户联系"权限的员工执行 |

### 备选方案：客户群进群方式（服务端）

官方支持的标准路径是 [客户群进群方式](https://developer.work.weixin.qq.com/document/path/92229#19330)：

**核心思路**：预建客户群 + 进群二维码

**步骤**：
1. 在企微客户端由群主（如服务人员/主管）创建该客户的"客户群"（首次人工）
2. 在业务系统登记"客户/订单 ↔ chat_id"的映射
3. 调用 `add_join_way`（`chat_id_list=[客户专属 chat_id]`，可加 `state=订单号`）
4. 调用 `get_join_way` 获取 `qr_code`，返回前端展示给客户扫码
5. 客户扫码后入群，后续复用已生成的 `config_id/qr_code` 即可

**关键接口**：
```
创建进群方式：POST /externalcontact/groupchat/add_join_way
获取进群方式：POST /externalcontact/groupchat/get_join_way
```

**关键字段**：
- `scene=2`（小程序场景）
- `chat_id_list`（必填，最多5个群）
- `auto_create_room`（群满自动建新群）
- `room_base_name/room_base_id`（新群命名规则）
- `state`（可用于标识订单号等）

**优化建议**：建少量"空壳客户群"作为缓冲池，业务系统自动分配未占用的 `chat_id` 给新客户；余量低于阈值时自动提醒补建。

**方案局限**：
- 首次建群必须人工操作
- 客户必须扫码进群
- 一客一群会导致群数量≈客户数量

### 关于多客户进群的疑问

> 假如 A 员工服务于客户 A，B 员工服务于客户 B，如果用同一个 `chat_id_list` 生成二维码，A 客户和 B 客户扫码后会进入同一个群吗？

**答案**：是的，会进入同一个群。`chat_id_list` 指定的是目标群，所有扫该二维码的人都会进入这些群。

如果要实现"一客一群"的隔离，需要：
- 每个客户/订单对应独立的 `chat_id`
- 每个客户生成独立的进群二维码

### JS-SDK 方案（推荐）

继续调研发现，企微 JS-SDK 提供了 `openEnterpriseChat` 接口，可以在企微客户端内**直接创建**包含内部成员和外部客户的群聊，无需预建群。

关键文档：
- [JS-SDK 接口文档](https://developer.work.weixin.qq.com/document/path/92525)
- [客户端 API - 获取客户列表](https://developer.work.weixin.qq.com/document/path/100746)

## 最终方案

通过企微内置浏览器访问 H5 页面，加载 JS-SDK，调用 `openEnterpriseChat` 创建客户群。

**技术栈**：前端 Vue 3 + TypeScript，后端 Node.js（Egg.js）

**核心流程**：企微扫码访问 H5 → 加载 SDK → 点击按钮 → 选择联系人 → 创建客户群

### 架构设计

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   企微客户端     │────▶│    H5 页面      │────▶│    后端服务      │
│  (内置浏览器)    │     │  (Vue 组件)     │     │  (签名接口)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │                       │
        │                       │  1. 请求签名           │
        │                       │──────────────────────▶│
        │                       │                       │
        │                       │  2. 返回签名配置       │
        │                       │◀──────────────────────│
        │                       │                       │
        │  3. 调用 JS-SDK       │                       │
        │◀──────────────────────│                       │
        │                       │                       │
        │  4. 创建客户群        │                       │
        │                       │                       │
```

### 前端实现

核心代码（Vue 3）：

```typescript
// 1. 加载 JS-SDK 脚本
const loadSdkScript = async () => {
  await new Promise<void>((resolve, reject) => {
    const script = document.createElement('script')
    script.src = 'https://wwcdn.weixin.qq.com/node/open/js/wecom-jssdk-2.3.2.js'
    script.onload = () => resolve()
    script.onerror = () => reject(new Error('脚本加载失败'))
    document.head.appendChild(script)
  })
}

// 2. 注册 SDK（新版使用 ww.register）
const registerWecomSDK = async () => {
  const ww = (window as any).ww
  
  ww.register({
    corpId: 'your_corp_id',
    agentId: your_agent_id,
    jsApiList: ['selectExternalContact', 'openEnterpriseChat', 'getContext'],
    // 企业签名回调
    async getConfigSignature(url: string) {
      return await getSignature(url, 'corp')
    },
    // 应用签名回调
    async getAgentConfigSignature(url: string) {
      return await getSignature(url, 'agent')
    },
  })
}

// 3. 创建客户群
const handleCreateCustomerGroup = async () => {
  const ww = (window as any).ww
  
  // 获取当前用户 ID（群主）
  const userRes = await ww.getContext()
  const currentUserId = userRes?.userId
  
  // 从后端获取客户列表（可按手机号过滤）
  const response = await api.getCustomers({ 
    userId: currentUserId,
    mobile: customerMobile  // 可选：按手机号匹配特定客户
  })
  const externalUserIds = response.externalUserIds
  
  // 调用 JS-SDK 创建群
  const res = await ww.openEnterpriseChat({
    groupName: '客户服务群',
    userIds: [currentUserId, ...colleagueUserIds],  // 内部成员（当前用户即群主）
    externalUserIds: externalUserIds,                // 外部客户
  })
  
  if (res?.errMsg === 'openEnterpriseChat:ok') {
    console.log('创建成功，chatId:', res?.chatId)
  }
}
```

### 后端签名实现

签名算法参考 [官方文档](https://developer.work.weixin.qq.com/document/path/100746)：

```javascript
// 入参：url（当前页面完整URL，必填）
const urlToSign = url.split('#')[0]  // 去掉 # 及其后面的部分

// 1. 获取 jsapi_ticket（有效期7200秒，建议缓存）
const jsapi_ticket = await getJSSDKTicket()

// 2. 生成签名参数
const noncestr = generateRandomString(32)
const timestamp = Math.floor(Date.now() / 1000)

// 3. 按字典序拼接后 SHA1
const signString = `jsapi_ticket=${jsapi_ticket}&noncestr=${noncestr}&timestamp=${timestamp}&url=${urlToSign}`
const signature = crypto.createHash('sha1').update(signString).digest('hex')

// 4. 返回签名配置
return {
  appId: corpid,
  timestamp,
  nonceStr: noncestr,
  signature,
}
```

**注意事项**：
- URL 必须去掉 `#` 及其后面的部分，且必须包含协议（`http://` 或 `https://`）
- `jsapi_ticket` 有效期 7200 秒，建议缓存
- 第三方应用需要使用 `suite_access_token` 获取 ticket
- 企微要求 URL 必须完全匹配，包括末尾斜杠

### 实施要点

**企微管理后台配置**：
1. 开通"客户联系"功能
2. 将应用加入"客户联系-可调用接口的应用"
3. 配置应用可信域名（H5 页面的域名）
4. 确保群主具备客户联系权限

**业务系统侧**：
1. 建立"客户/订单 ↔ chat_id"映射表（如需后续管理）
2. 提供客户列表查询接口（支持按手机号过滤）
3. 生成二维码流程（如使用服务端方案）：`add_join_way → get_join_way → 落库`
4. 幂等处理：有 `qr_code` 复用；仅 `config_id` 则 `get_join_way` 拉取；都无才 `add_join_way`

## 踩坑记录

### 1. 签名失败（40093）

原因：企业签名和应用签名混淆。新版 JS-SDK 需要同时提供两种签名：
- `getConfigSignature`：企业签名，使用企业 `jsapi_ticket`
- `getAgentConfigSignature`：应用签名，使用应用 `jsapi_ticket`

建议：后端同时返回 `corpSignature` 和 `agentSignature`，前端根据回调类型选用。

### 2. 可信域名错误（80001）

原因：H5 页面域名未配置为应用可信域名。

解决：在企微管理后台 → 我的应用 → 设置应用可信域名。注意内网 IP 地址可能无法配置，建议使用域名。

### 3. localhost 无法访问

企微客户端无法访问 `localhost` 或 `127.0.0.1`，开发时需要：
- 使用内网 IP（如 `192.168.x.x`）
- 或使用 ngrok 等内网穿透工具

### 4. URL 签名不匹配

签名时的 URL 必须与实际访问的 URL 完全一致，包括：
- 协议（http/https）
- 域名
- 端口（非默认端口）
- 路径
- 末尾斜杠

建议前端传入 `window.location.href.split('#')[0]` 作为签名 URL。

## 方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 服务端进群方式 | 自动化程度高，可后台静默操作 | 需预建客户群，客户需扫码 | 批量运营、自动化流程 |
| JS-SDK 方案 | 一步到位创建群，无需预建 | 需在企微客户端内操作 | 一客一群、即时创建 |

对于"一客一群"的场景，JS-SDK 方案更简洁直接。

## 总结

通过企微 JS-SDK 的 `openEnterpriseChat` 接口，可以在企微客户端内一键创建包含内部成员和外部客户的群聊，实现了：

- ✅ 一客一群，群归企业
- ✅ 员工离职不丢客
- ✅ 操作简单，无需预建群
- ✅ 数据可沉淀（`chatId` 可落库关联业务）

如果业务场景需要更高的自动化程度（如客户无需操作即可进群），可以考虑"服务端进群方式"作为补充方案。

---

参考文档：
- [企业微信 JS-SDK 文档](https://developer.work.weixin.qq.com/document/path/92525)
- [JS-SDK 签名算法](https://developer.work.weixin.qq.com/document/path/100746)
- [客户群进群方式](https://developer.work.weixin.qq.com/document/path/92229)
- [获取客户列表](https://developer.work.weixin.qq.com/document/path/92113)
