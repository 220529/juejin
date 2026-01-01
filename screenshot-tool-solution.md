# 跨平台截图工具技术方案

## 一、市场调研

### 1.1 竞品分析

| 工具 | 平台 | 技术栈 | 特点 | 缺点 |
|------|------|--------|------|------|
| **Snipaste** | Win/Mac | Qt + C++ | 贴图功能强大、像素级标注 | 闭源、Mac 版功能少 |
| **Flameshot** | Linux/Win/Mac | Qt + C++ | 开源、标注丰富 | Mac 体验差 |
| **iShot** | Mac | Swift | 原生体验、长截图、OCR | 仅 Mac、部分收费 |
| **ShareX** | Win | C# | 功能最全、自动化工作流 | 仅 Windows |
| **Shottr** | Mac | Swift | 轻量、滚动截图 | 仅 Mac |
| **Greenshot** | Win | C# | 开源、轻量 | 仅 Windows、停更 |

### 1.2 用户核心需求

1. **快速截图** - 全局快捷键一键触发
2. **区域选择** - 自由框选截图区域
3. **标注编辑** - 箭头、矩形、文字、马赛克
4. **快速分享** - 复制到剪贴板、保存文件
5. **贴图功能** - 截图悬浮在桌面（Snipaste 特色）

### 1.3 市场空白

- **跨平台一致体验**：现有工具要么单平台，要么跨平台体验差
- **开源 + 现代技术栈**：Qt/C++ 门槛高，缺少 Rust/Web 技术的现代方案
- **轻量级**：Electron 方案太重，Tauri 是更好的选择

---

## 二、技术选型

### 2.1 框架对比

| 方案 | 包体积 | 性能 | 开发效率 | 跨平台 |
|------|--------|------|----------|--------|
| **Tauri 2.0** | ~10MB | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Win/Mac/Linux |
| Electron | ~150MB | ⭐⭐ | ⭐⭐⭐⭐⭐ | Win/Mac/Linux |
| Qt + C++ | ~30MB | ⭐⭐⭐⭐⭐ | ⭐⭐ | Win/Mac/Linux |
| Flutter | ~20MB | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Win/Mac/Linux |

**选择 Tauri 2.0**：
- 包体积小，适合工具类应用
- Rust 后端性能好，截图处理快
- 前端用 Web 技术，开发效率高
- 你已经有 Tauri 经验

### 2.2 技术栈

```
┌─────────────────────────────────────────────────────┐
│                    Frontend                          │
│         React 19 + TypeScript + Tailwind CSS        │
│         Canvas API（选区绘制、标注编辑）              │
├─────────────────────────────────────────────────────┤
│                   Tauri 2.0 IPC                      │
├─────────────────────────────────────────────────────┤
│                    Backend (Rust)                    │
│  ┌─────────────┬─────────────┬─────────────────┐   │
│  │  截图核心   │   图像处理   │    系统集成      │   │
│  │ screenshots │    image    │ global-shortcut │   │
│  │   xcap      │   imageproc │   clipboard     │   │
│  │             │             │   tray          │   │
│  └─────────────┴─────────────┴─────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 2.3 核心依赖

**Rust Crates**：

| Crate | 用途 | 说明 |
|-------|------|------|
| `xcap` | 屏幕截图 | 跨平台，支持多显示器 |
| `image` | 图像处理 | 裁剪、缩放、格式转换 |
| `arboard` | 剪贴板 | 跨平台剪贴板操作 |
| `tauri-plugin-global-shortcut` | 全局快捷键 | 官方插件 |
| `tauri-plugin-autostart` | 开机自启 | 官方插件 |

**为什么选 xcap 而不是 screenshots**：
- `xcap` 更活跃，支持 Wayland
- 支持窗口级截图
- API 更简洁

---

## 三、架构设计

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                        用户交互层                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  系统托盘   │  │  全局快捷键  │  │     设置窗口        │ │
│  │  右键菜单   │  │  Cmd+Shift+S│  │  快捷键/保存路径    │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘ │
└─────────┼────────────────┼────────────────────┼─────────────┘
          │                │                    │
          ▼                ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                        核心服务层                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   截图服务                           │   │
│  │  1. capture_all_screens() → 全屏截图                │   │
│  │  2. show_selection_window() → 显示选区窗口          │   │
│  │  3. crop_image(rect) → 裁剪选中区域                 │   │
│  │  4. save_or_copy() → 保存/复制到剪贴板              │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   标注服务                           │   │
│  │  - 矩形、椭圆、箭头、直线                            │   │
│  │  - 文字、序号                                       │   │
│  │  - 马赛克、高斯模糊                                  │   │
│  │  - 撤销/重做                                        │   │
│  └─────────────────────────────────────────────────────┘   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   贴图服务                           │   │
│  │  - 创建悬浮窗口                                      │   │
│  │  - 置顶/透明度调节                                   │   │
│  │  - 缩放/关闭                                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                        数据存储层                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  配置文件   │  │  历史记录   │  │     临时文件        │ │
│  │  JSON/TOML │  │  SQLite     │  │  截图缓存           │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 截图流程

```
用户按下快捷键 (Cmd+Shift+S)
        │
        ▼
┌───────────────────┐
│ 1. 隐藏所有窗口   │  ← 避免截到自己的窗口
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ 2. 全屏截图       │  ← xcap 截取所有显示器
│    保存到内存     │
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ 3. 创建选区窗口   │  ← 全屏无边框窗口
│    显示截图+遮罩  │     覆盖所有显示器
└────────┬──────────┘
         │
         ▼
┌───────────────────┐
│ 4. 用户拖拽选区   │  ← Canvas 绘制选区框
│    实时显示尺寸   │     显示放大镜
└────────┬──────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌───────┐
│ ESC   │ │ 确认  │
│ 取消  │ │ 选区  │
└───────┘ └───┬───┘
              │
              ▼
┌───────────────────┐
│ 5. 显示标注工具栏 │  ← 可选：直接保存或标注
└────────┬──────────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌───────┐ ┌───────┐
│ 标注  │ │ 完成  │
│ 编辑  │ │       │
└───┬───┘ └───┬───┘
    │         │
    └────┬────┘
         │
         ▼
┌───────────────────┐
│ 6. 裁剪图像       │
│    复制到剪贴板   │
│    保存到文件     │
└───────────────────┘
```

### 3.3 窗口管理

| 窗口 | 类型 | 说明 |
|------|------|------|
| 主窗口 | 隐藏 | 仅用于托盘菜单和设置 |
| 选区窗口 | 全屏无边框 | 截图时创建，完成后销毁 |
| 标注窗口 | 普通窗口 | 编辑标注时显示 |
| 贴图窗口 | 无边框置顶 | 可创建多个，独立管理 |
| 设置窗口 | 普通窗口 | 配置快捷键、保存路径等 |

---

## 四、核心功能设计

### 4.1 截图模块

```rust
// 截取所有屏幕
pub fn capture_all_screens() -> Result<Vec<ScreenCapture>> {
    let screens = xcap::Monitor::all()?;
    screens.iter().map(|s| {
        let image = s.capture_image()?;
        Ok(ScreenCapture {
            id: s.id(),
            x: s.x(),
            y: s.y(),
            width: s.width(),
            height: s.height(),
            image,
        })
    }).collect()
}

// 裁剪指定区域
pub fn crop_region(captures: &[ScreenCapture], rect: Rect) -> DynamicImage {
    // 找到对应的屏幕，裁剪指定区域
}
```

### 4.2 选区模块（前端 Canvas）

```typescript
// 选区状态
interface SelectionState {
  isSelecting: boolean;
  startPoint: Point | null;
  endPoint: Point | null;
  rect: Rect | null;
}

// Canvas 绘制
function drawSelection(ctx: CanvasRenderingContext2D, state: SelectionState) {
  // 1. 绘制半透明遮罩
  ctx.fillStyle = 'rgba(0, 0, 0, 0.5)';
  ctx.fillRect(0, 0, width, height);
  
  // 2. 清除选区部分（显示原图）
  if (state.rect) {
    ctx.clearRect(state.rect.x, state.rect.y, state.rect.width, state.rect.height);
    
    // 3. 绘制选区边框
    ctx.strokeStyle = '#00ff00';
    ctx.lineWidth = 2;
    ctx.strokeRect(state.rect.x, state.rect.y, state.rect.width, state.rect.height);
    
    // 4. 显示尺寸信息
    drawSizeInfo(ctx, state.rect);
  }
}
```

### 4.3 标注模块

```typescript
// 标注工具类型
type AnnotationTool = 
  | 'rect'      // 矩形
  | 'ellipse'   // 椭圆
  | 'arrow'     // 箭头
  | 'line'      // 直线
  | 'text'      // 文字
  | 'number'    // 序号
  | 'mosaic'    // 马赛克
  | 'blur';     // 模糊

// 标注对象
interface Annotation {
  id: string;
  tool: AnnotationTool;
  points: Point[];
  style: AnnotationStyle;
}

// 撤销/重做栈
class AnnotationHistory {
  private undoStack: Annotation[][] = [];
  private redoStack: Annotation[][] = [];
  
  push(annotations: Annotation[]) { ... }
  undo(): Annotation[] | null { ... }
  redo(): Annotation[] | null { ... }
}
```

### 4.4 贴图模块

```rust
// 创建贴图窗口
pub fn create_pin_window(app: &AppHandle, image: DynamicImage) -> Result<()> {
    let window = tauri::WebviewWindowBuilder::new(
        app,
        format!("pin-{}", uuid()),
        tauri::WebviewUrl::App("pin.html".into())
    )
    .title("")
    .decorations(false)      // 无边框
    .always_on_top(true)     // 置顶
    .transparent(true)       // 透明背景
    .resizable(true)
    .build()?;
    
    // 发送图片数据到窗口
    window.emit("set-image", image_to_base64(&image))?;
    
    Ok(())
}
```

---

## 五、平台适配

### 5.1 支持平台

| 平台 | 支持度 | 说明 |
|------|--------|------|
| **macOS** | ⭐⭐⭐⭐⭐ | 完整支持，需要屏幕录制权限 |
| **Windows** | ⭐⭐⭐⭐⭐ | 完整支持 |
| **Linux X11** | ⭐⭐⭐⭐ | 完整支持 |
| **Linux Wayland** | ⭐⭐⭐ | 部分支持，需要 xdg-desktop-portal |

### 5.2 平台特殊处理

**macOS**：
```rust
// 需要请求屏幕录制权限
#[cfg(target_os = "macos")]
pub fn request_screen_capture_permission() -> bool {
    // 使用 CGPreflightScreenCaptureAccess / CGRequestScreenCaptureAccess
}
```

**Windows**：
```rust
// 高 DPI 处理
#[cfg(target_os = "windows")]
pub fn get_scale_factor(monitor: &Monitor) -> f64 {
    // 使用 GetDpiForMonitor
}
```

**Linux Wayland**：
```rust
// 使用 xdg-desktop-portal 截图
#[cfg(all(target_os = "linux", feature = "wayland"))]
pub fn capture_wayland() -> Result<DynamicImage> {
    // D-Bus 调用 org.freedesktop.portal.Screenshot
}
```

---

## 六、性能优化

### 6.1 截图性能

| 优化点 | 方案 |
|--------|------|
| 大图传输 | 使用共享内存，避免 IPC 序列化 |
| 多显示器 | 并行截取，合并结果 |
| 高 DPI | 按需缩放，避免全分辨率处理 |

### 6.2 渲染性能

| 优化点 | 方案 |
|--------|------|
| Canvas 绘制 | 使用 OffscreenCanvas + requestAnimationFrame |
| 标注编辑 | 分层渲染（背景层 + 标注层 + 交互层） |
| 马赛克/模糊 | Rust 端处理，避免 JS 性能瓶颈 |

---

## 七、项目结构

```
screenshot-tool/
├── src/                          # 前端代码
│   ├── windows/                  # 窗口组件
│   │   ├── Selection.tsx         # 选区窗口
│   │   ├── Annotation.tsx        # 标注窗口
│   │   ├── Pin.tsx               # 贴图窗口
│   │   └── Settings.tsx          # 设置窗口
│   ├── components/               # 通用组件
│   │   ├── Canvas.tsx            # Canvas 封装
│   │   ├── Toolbar.tsx           # 工具栏
│   │   └── ColorPicker.tsx       # 颜色选择器
│   ├── hooks/                    # 自定义 Hooks
│   │   ├── useCanvas.ts          # Canvas 操作
│   │   └── useAnnotation.ts      # 标注管理
│   └── utils/                    # 工具函数
├── src-tauri/                    # Rust 后端
│   ├── src/
│   │   ├── capture/              # 截图模块
│   │   │   ├── mod.rs
│   │   │   ├── screen.rs         # 屏幕截图
│   │   │   └── window.rs         # 窗口截图
│   │   ├── image/                # 图像处理
│   │   │   ├── mod.rs
│   │   │   ├── crop.rs           # 裁剪
│   │   │   └── annotate.rs       # 标注渲染
│   │   ├── clipboard.rs          # 剪贴板
│   │   ├── shortcut.rs           # 快捷键
│   │   ├── tray.rs               # 系统托盘
│   │   └── lib.rs
│   ├── Cargo.toml
│   └── tauri.conf.json
├── package.json
└── README.md
```

---

## 八、开发计划

### Phase 1：MVP（1-2 周）
- [ ] 项目初始化
- [ ] 全屏截图
- [ ] 区域选择
- [ ] 复制到剪贴板
- [ ] 保存到文件
- [ ] 全局快捷键
- [ ] 系统托盘

### Phase 2：标注功能（1-2 周）
- [ ] 矩形、椭圆
- [ ] 箭头、直线
- [ ] 文字标注
- [ ] 马赛克
- [ ] 撤销/重做

### Phase 3：高级功能（1-2 周）
- [ ] 贴图功能
- [ ] 滚动截图
- [ ] OCR 文字识别
- [ ] 历史记录
- [ ] 云同步

---

## 九、总结

**技术选型**：Tauri 2.0 + React + Rust

**核心优势**：
1. 跨平台一致体验
2. 包体积小（~15MB）
3. 性能好（Rust 图像处理）
4. 开发效率高（Web 前端）

**与 File Toolkit 的关系**：
- 独立项目，专注截图场景
- 可共享 Tauri + React 技术栈经验
- 未来可考虑整合为工具套件

---

要开始做吗？
