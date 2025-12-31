# 照片直播小程序技术方案

> 类似"喔图"的照片直播平台，支持活动现场实时上传、用户端瀑布流浏览、新图片推送提醒

## 一、产品功能

### 用户端（小程序）
- 瀑布流浏览活动照片
- 实时接收新图片推送，显示"有N张新图片"提示
- 点击提示后滚动到顶部，展示新图
- 长按/点击下载原图到相册
- 分享活动给好友

### 摄影师端（H5/小程序）
- 创建活动，生成分享码
- 批量上传照片
- 管理已上传照片（删除、排序）

### 管理后台（Web）
- 活动管理
- 数据统计
- 用户管理

## 二、技术架构

```
┌─────────────────────────────────────────────────────────────┐
│                        微信生态                              │
│  ┌──────────────┐              ┌──────────────┐             │
│  │  用户小程序   │              │ 摄影师小程序  │             │
│  │  (浏览下载)   │              │  (上传管理)   │             │
│  └──────┬───────┘              └──────┬───────┘             │
└─────────┼────────────────────────────┼──────────────────────┘
          │                            │
          ▼                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     微信云开发                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   云函数      │  │  实时数据库   │  │   云存储      │      │
│  │  (业务逻辑)   │  │  (数据+推送)  │  │  (图片CDN)   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### 技术选型

| 模块 | 技术方案 | 说明 |
|------|---------|------|
| 用户端 | 微信小程序原生 | 性能好，生态契合 |
| 摄影师端 | 微信小程序 / H5 | 小程序方便，H5 批量上传更灵活 |
| 后端 | 微信云开发 | 免运维，自带实时推送 |
| 图片存储 | 云存储 | 自带 CDN 加速 |
| 实时推送 | 实时数据库 watch | 替代 WebSocket |

## 三、数据库设计

### events 活动表
```javascript
{
  _id: "event_001",
  title: "天童幼儿园元旦演出",      // 活动标题
  coverUrl: "cloud://xxx",         // 封面图
  shareCode: "ABC123",             // 6位分享码
  creatorId: "user_001",           // 创建者openid
  photoCount: 156,                 // 照片数量（冗余）
  viewCount: 892,                  // 浏览次数
  status: 1,                       // 0-关闭 1-进行中 2-已结束
  createdAt: Date,
  updatedAt: Date
}
```

### photos 照片表
```javascript
{
  _id: "photo_001",
  eventId: "event_001",            // 关联活动
  fileId: "cloud://xxx",           // 云存储文件ID
  thumbFileId: "cloud://xxx",      // 缩略图文件ID
  width: 1920,                     // 原图宽度
  height: 1280,                    // 原图高度
  size: 2048000,                   // 文件大小(bytes)
  uploaderId: "user_002",          // 上传者
  createdAt: Date
}

// 索引
// eventId + createdAt 复合索引（按活动查询，按时间倒序）
```

### users 用户表
```javascript
{
  _id: "user_001",
  openid: "oXXXX",
  nickName: "张三",
  avatarUrl: "https://xxx",
  role: "photographer",            // user/photographer/admin
  createdAt: Date
}
```

## 四、核心功能实现

### 4.1 实时推送新图片

利用云开发实时数据库的 `watch` 能力：

```javascript
// pages/live/live.js
Page({
  data: {
    photos: [],
    newPhotos: [],
    newCount: 0
  },

  onLoad(options) {
    this.eventId = options.eventId;
    this.loadPhotos();
    this.startWatch();
  },

  // 监听新图片
  startWatch() {
    const db = wx.cloud.database();
    this.watcher = db.collection('photos')
      .where({ eventId: this.eventId })
      .orderBy('createdAt', 'desc')
      .limit(1)
      .watch({
        onChange: (snapshot) => {
          if (snapshot.type === 'init') return;
          
          const newDocs = snapshot.docChanges.filter(c => c.queueType === 'enqueue');
          if (newDocs.length > 0) {
            this.setData({
              newPhotos: [...newDocs.map(d => d.doc), ...this.data.newPhotos],
              newCount: this.data.newCount + newDocs.length
            });
          }
        },
        onError: (err) => console.error(err)
      });
  },

  // 点击"有N张新图片"
  loadNewPhotos() {
    this.setData({
      photos: [...this.data.newPhotos, ...this.data.photos],
      newPhotos: [],
      newCount: 0
    });
    
    // 滚动到顶部
    wx.pageScrollTo({ scrollTop: 0, duration: 300 });
  },

  onUnload() {
    this.watcher?.close();
  }
});
```

### 4.2 瀑布流布局

小程序实现双列瀑布流：

```javascript
// utils/waterfall.js
function splitToColumns(photos, columnCount = 2, columnWidth = 375) {
  const columns = Array.from({ length: columnCount }, () => ({ items: [], height: 0 }));
  
  photos.forEach(photo => {
    // 计算图片在列中的实际高度
    const itemHeight = (photo.height / photo.width) * columnWidth + 8; // 8是间距
    
    // 找到最短的列
    const shortestCol = columns.reduce((min, col, idx) => 
      col.height < columns[min].height ? idx : min, 0);
    
    columns[shortestCol].items.push(photo);
    columns[shortestCol].height += itemHeight;
  });
  
  return columns.map(col => col.items);
}

module.exports = { splitToColumns };
```

```html
<!-- pages/live/live.wxml -->
<view class="new-tip" wx:if="{{newCount > 0}}" bindtap="loadNewPhotos">
  有 {{newCount}} 张新图片，点击查看
</view>

<view class="waterfall">
  <view class="column" wx:for="{{columns}}" wx:key="index" wx:for-item="column">
    <view class="photo-item" wx:for="{{column}}" wx:key="_id" wx:for-item="photo">
      <image 
        src="{{photo.thumbFileId}}" 
        mode="widthFix"
        lazy-load
        bindtap="previewPhoto"
        data-photo="{{photo}}"
      />
    </view>
  </view>
</view>
```

```css
/* pages/live/live.wxss */
.new-tip {
  position: fixed;
  top: 20rpx;
  left: 50%;
  transform: translateX(-50%);
  background: #07c160;
  color: #fff;
  padding: 16rpx 32rpx;
  border-radius: 32rpx;
  font-size: 28rpx;
  z-index: 100;
  box-shadow: 0 4rpx 12rpx rgba(0,0,0,0.15);
}

.waterfall {
  display: flex;
  padding: 8rpx;
}

.column {
  flex: 1;
  padding: 0 4rpx;
}

.photo-item {
  margin-bottom: 8rpx;
  border-radius: 8rpx;
  overflow: hidden;
}

.photo-item image {
  width: 100%;
  display: block;
}
```

### 4.3 图片上传（摄影师端）

```javascript
// 云函数：uploadPhoto
const cloud = require('wx-server-sdk');
cloud.init();

exports.main = async (event, context) => {
  const { eventId, fileId, width, height, size } = event;
  const { OPENID } = cloud.getWXContext();
  
  const db = cloud.database();
  
  // 1. 生成缩略图（可选，用云函数处理）
  const thumbFileId = await generateThumb(fileId);
  
  // 2. 写入数据库
  const result = await db.collection('photos').add({
    data: {
      eventId,
      fileId,
      thumbFileId,
      width,
      height,
      size,
      uploaderId: OPENID,
      createdAt: db.serverDate()
    }
  });
  
  // 3. 更新活动照片数量
  await db.collection('events').doc(eventId).update({
    data: {
      photoCount: db.command.inc(1),
      updatedAt: db.serverDate()
    }
  });
  
  return { success: true, photoId: result._id };
};
```

### 4.4 图片下载

```javascript
// 保存图片到相册
async savePhoto(e) {
  const { fileId } = e.currentTarget.dataset;
  
  wx.showLoading({ title: '保存中...' });
  
  try {
    // 1. 获取临时路径
    const { tempFilePath } = await wx.cloud.downloadFile({ fileID: fileId });
    
    // 2. 保存到相册
    await wx.saveImageToPhotosAlbum({ filePath: tempFilePath });
    
    wx.showToast({ title: '已保存到相册', icon: 'success' });
  } catch (err) {
    if (err.errMsg.includes('auth deny')) {
      // 引导用户开启权限
      wx.showModal({
        title: '提示',
        content: '需要您授权保存图片到相册',
        success: (res) => {
          if (res.confirm) wx.openSetting();
        }
      });
    }
  } finally {
    wx.hideLoading();
  }
}
```

## 五、项目结构

```
miniprogram/
├── pages/
│   ├── index/              # 首页（输入分享码/扫码进入）
│   ├── live/               # 照片直播页（瀑布流）
│   ├── upload/             # 上传页（摄影师）
│   └── my/                 # 我的（历史活动）
├── components/
│   ├── waterfall/          # 瀑布流组件
│   ├── photo-item/         # 图片卡片
│   └── new-tip/            # 新图片提示
├── utils/
│   ├── waterfall.js        # 瀑布流算法
│   └── cloud.js            # 云开发封装
└── app.js

cloudfunctions/
├── uploadPhoto/            # 上传照片
├── getPhotos/              # 获取照片列表
├── createEvent/            # 创建活动
└── generateThumb/          # 生成缩略图（可选）
```

## 六、性能优化

### 6.1 图片加载优化
- 列表使用缩略图（宽度750px以内）
- 预览/下载时才加载原图
- 开启 `lazy-load` 懒加载

### 6.2 分页加载
```javascript
async loadMore() {
  if (this.loading || !this.hasMore) return;
  this.loading = true;
  
  const db = wx.cloud.database();
  const res = await db.collection('photos')
    .where({ eventId: this.eventId })
    .orderBy('createdAt', 'desc')
    .skip(this.data.photos.length)
    .limit(20)
    .get();
  
  this.hasMore = res.data.length === 20;
  this.setData({
    photos: [...this.data.photos, ...res.data]
  });
  this.loading = false;
}
```

### 6.3 缩略图生成
上传时通过云函数生成缩略图，或使用云存储的图片处理能力：
```javascript
// 云存储图片处理，获取宽度400的缩略图
const thumbUrl = fileId + '?imageMogr2/thumbnail/400x';
```

## 七、成本估算

以一场 500 人观看、200 张照片的活动为例：

| 资源 | 用量 | 费用 |
|------|------|------|
| 云存储 | 200张 × 3MB = 600MB | 免费额度内 |
| 数据库读 | 500人 × 20次 = 1万次 | 免费额度内 |
| 云函数 | 约 5000 次调用 | 免费额度内 |
| CDN流量 | 约 2GB | 约 1 元 |

**结论**：小型活动基本在免费额度内，大型活动成本也很低。

## 八、开发计划

| 阶段 | 内容 | 时间 |
|------|------|------|
| P0 | 基础框架 + 瀑布流展示 + 图片上传 | 2天 |
| P1 | 实时推送 + 新图片提示 | 1天 |
| P2 | 图片下载 + 分享功能 | 1天 |
| P3 | 摄影师端完善 + 活动管理 | 2天 |
| P4 | 优化 + 测试 + 上线 | 2天 |

**总计：约 8 天可完成 MVP 版本**

## 九、后续扩展

- [ ] AI 智能修图（接入腾讯云图像处理）
- [ ] 人脸识别找照片（找到有"我"的照片）
- [ ] 视频直播支持
- [ ] 付费下载高清原图
- [ ] 活动数据统计面板
