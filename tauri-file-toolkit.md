# Tauri 2.0 实战：打造「小文喵」文件工具箱

## 前言

最近在整理电脑文件时，发现了大量重复的照片和视频，占用了几十 GB 的空间。市面上的去重工具要么收费，要么功能臃肿，于是萌生了自己造一个的想法。

正好 Tauri 2.0 正式发布，相比 Electron 有着更小的包体积和更好的性能，于是决定用 **Tauri 2.0 + React + Rust** 来实现。

做着做着，需求就多了起来：

1. **文件去重** —— 最初的需求，整理重复文件
2. **视频截取** —— 有时候只想要视频的一小段，不想装 PR
3. **格式转换** —— iPhone 录的 MOV 想转成 MP4，还有些视频想转 GIF
4. **去水印** —— 想给应用换个好看的图标，用豆包 AI 生成了一张，结果右下角有水印。去网上搜"去水印"，要么收费要么要注册，就为了去个小水印？算了，自己加一个

于是就有了「小文喵」—— 一个跨平台的文件工具箱。

> GitHub: [https://github.com/220529/file-toolkit](https://github.com/220529/file-toolkit)

## 功能展示

### 📊 文件统计

递归扫描文件夹，按类型统计文件数量和大小。

### 🔍 文件去重

这是核心功能：

- **两阶段扫描**：先按文件大小筛选，再计算哈希
- **xxHash3 快速哈希**：比 MD5 快 5-10 倍
- **并行计算**：充分利用多核 CPU
- **大文件采样**：只读头部 + 中间 + 尾部，避免全量读取
- **缩略图预览**：图片直接显示，视频用 FFmpeg 截帧

### ✂️ 视频截取

- **快速模式**：无损截取（-c copy），秒级完成
- **精确模式**：重新编码，时间精确到毫秒
- **时间轴预览**：8 帧缩略图，快速定位

### 🔄 格式转换

- 支持 MOV、MP4、GIF 互转
- 批量转换
- 画质可选（高/中/低）
- 实时进度显示

### ✨ 去水印

最后加的功能：

- **高斯模糊**：让水印区域变糊
- **颜色覆盖**：用颜色盖住水印（适合纯色背景）
- **取色器**：点击打开系统调色板，或右键图片取色
- **选区操作**：拖动移动、拖动边角缩放

## 技术选型

### 为什么选 Tauri 而不是 Electron？

| 对比项 | Electron | Tauri |
|--------|----------|-------|
| 包体积 | 150MB+ | 10MB+ |
| 内存占用 | 高（Chromium） | 低（系统 WebView） |
| 后端语言 | Node.js | Rust |
| 性能 | 一般 | 优秀 |

对于文件处理这种 CPU 密集型任务，Rust 的性能优势非常明显。

### 技术栈

```
┌─────────────────────────────────────────────────────┐
│                    Frontend                          │
│         React 19 + TypeScript + Tailwind CSS        │
├─────────────────────────────────────────────────────┤
│                   Tauri IPC                          │
├─────────────────────────────────────────────────────┤
│                    Backend                           │
│                  Rust + Tauri 2.0                   │
│  ┌──────────┬────────┬────────┬─────────┬────────┐ │
│  │file_stats│ dedup  │ video  │ convert │watermark│ │
│  │ walkdir  │xxHash3 │ FFmpeg │ FFmpeg  │ FFmpeg  │ │
│  │          │ rayon  │        │         │         │ │
│  └──────────┴────────┴────────┴─────────┴────────┘ │
└─────────────────────────────────────────────────────┘
```

## 核心实现

### 1. 文件去重算法

去重的核心是计算文件哈希，但如果对每个文件都完整计算哈希，效率会很低。我采用了两阶段策略：

**第一阶段：按文件大小筛选**

```rust
let mut size_map: HashMap<u64, Vec<String>> = HashMap::new();

for entry in WalkDir::new(&path).into_iter().filter_map(|e| e.ok()) {
    if let Ok(meta) = entry.metadata() {
        size_map.entry(meta.len()).or_default().push(entry.path().to_string_lossy().to_string());
    }
}

// 只对大小相同的文件计算哈希
let files_to_hash: Vec<_> = size_map
    .iter()
    .filter(|(_, files)| files.len() >= 2)
    .flat_map(|(size, files)| files.iter().map(move |f| (*size, f.clone())))
    .collect();
```

**第二阶段：并行计算哈希**

```rust
use rayon::prelude::*;

let results: Vec<(String, FileInfo)> = files_to_hash
    .par_iter()  // 并行迭代
    .filter_map(|(size, file_path)| {
        let hash = calculate_fast_hash(Path::new(file_path), *size).ok()?;
        Some((hash, file_info))
    })
    .collect();
```

### 2. 快速哈希算法

选择 **xxHash3**，目前最快的非加密哈希算法之一：

```rust
fn calculate_fast_hash(path: &Path, size: u64) -> Result<String, String> {
    use memmap2::Mmap;
    use xxhash_rust::xxh3::Xxh3;

    const THRESHOLD: u64 = 10 * 1024 * 1024;  // 10MB
    const SAMPLE_SIZE: usize = 1024 * 1024;   // 采样 1MB

    let file = File::open(path).map_err(|e| e.to_string())?;
    let mmap = unsafe { Mmap::map(&file) }.map_err(|e| e.to_string())?;

    if size <= THRESHOLD {
        // 小文件：完整读取
        let hash = xxhash_rust::xxh3::xxh3_64(&mmap);
        return Ok(format!("{:016x}", hash));
    }

    // 大文件：只读头部 + 中间 + 尾部
    let mut hasher = Xxh3::new();
    let len = mmap.len();
    hasher.update(&mmap[..SAMPLE_SIZE]);                         // 头部
    hasher.update(&mmap[len/2 - SAMPLE_SIZE/2..][..SAMPLE_SIZE]); // 中间
    hasher.update(&mmap[len - SAMPLE_SIZE..]);                   // 尾部
    hasher.update(&size.to_le_bytes());                          // 文件大小

    Ok(format!("{:016x}", hasher.digest()))
}
```

**优化点**：
- **xxHash3**：比 MD5 快 5-10 倍
- **memmap2**：内存映射，零拷贝读取
- **大文件采样**：只读头中尾各 1MB

### 3. 去水印实现

去水印有两种模式：

**高斯模糊**：用 FFmpeg 的 `boxblur` 滤镜

```rust
let filter = format!(
    "[0:v]crop={}:{}:{}:{}[crop];[crop]boxblur=15:3[blur];[0:v][blur]overlay={}:{}",
    width, height, x, y, x, y
);

Command::new("ffmpeg")
    .args(["-y", "-i", &input_path, "-filter_complex", &filter, "-q:v", "1", &output_path])
    .output()
```

**颜色覆盖**：用 `drawbox` 滤镜

```rust
// 注意：颜色格式要从 #ffffff 转成 0xffffff
let ffmpeg_color = if color.starts_with('#') {
    format!("0x{}", &color[1..])
} else {
    color.clone()
};

let filter = format!(
    "drawbox=x={}:y={}:w={}:h={}:color={}:t=fill",
    x, y, width, height, ffmpeg_color
);
```

### 4. 格式转换

批量转换的关键是进度反馈：

```rust
// 后端发送进度
let _ = app.emit("convert-progress", ConvertProgress {
    current_file: file_name,
    current: index + 1,
    total: files.len(),
    percent,
});
```

```typescript
// 前端监听
useEffect(() => {
  const unlisten = listen<ConvertProgress>("convert-progress", (event) => {
    setProgress(event.payload);
  });
  return () => { unlisten.then((fn) => fn()); };
}, []);
```

## 踩坑记录

开发过程中踩了不少坑，记录下来供参考。

### 1. macOS 图标白底问题

这个坑花了不少时间。用豆包生成的图标有透明背景，结果在 macOS 上显示时自动加了白底，很丑。

**原因**：macOS Big Sur 开始，所有 App 图标强制使用 squircle（圆角方形）形状。透明背景的图标会被系统自动加白底。

**解决方案**：图标设计时直接使用带背景色的 squircle 形状，不要用透明背景。让 AI 生成时就指定"squircle 形状，带背景色"。

### 2. DMG 打包失败

执行 `pnpm tauri build` 时报错：

```
failed to bundle project error running bundle_dmg.sh
```

**原因**：macOS 上打包 DMG 需要 `create-dmg` 工具。

**解决方案**：
```bash
brew install create-dmg
```

### 3. FFmpeg 滤镜语法

高斯模糊一开始用 `-vf` 参数，结果报错。

**原因**：复杂滤镜（涉及多个输入输出流）需要用 `-filter_complex`。

```bash
# 错误 ❌
ffmpeg -i input.jpg -vf "split[a][b];..." output.jpg

# 正确 ✅
ffmpeg -i input.jpg -filter_complex "[0:v]crop=...;..." output.jpg
```

### 4. 颜色格式不兼容

FFmpeg 的 `drawbox` 滤镜不认 `#ffffff` 格式。

**解决方案**：转成 `0xffffff` 格式：

```rust
let ffmpeg_color = if color.starts_with('#') {
    format!("0x{}", &color[1..])
} else {
    color.clone()
};
```

### 5. Tauri 拖拽事件是全局的

最初用 CSS `hidden` 隐藏非活动 Tab，但发现拖拽文件时所有 Tab 都会响应。

**解决方案**：给每个组件传递 `active` 属性，只有激活的组件才监听拖拽事件：

```typescript
useEffect(() => {
  if (!active) return;  // 非激活状态不监听
  const unlisten = listen("tauri://drag-drop", handler);
  return () => { unlisten.then((fn) => fn()); };
}, [active]);
```

### 6. 并行计算进度跳动

使用 rayon 并行计算时，多线程同时更新计数器，导致进度显示不连续。

**解决方案**：使用原子变量 + compare_exchange：

```rust
let progress_counter = Arc::new(AtomicUsize::new(0));
let last_reported = Arc::new(AtomicUsize::new(0));

let current = progress_counter.fetch_add(1, Ordering::Relaxed) + 1;
let last = last_reported.load(Ordering::Relaxed);

if current > last {
    if last_reported.compare_exchange(last, current, Ordering::SeqCst, Ordering::Relaxed).is_ok() {
        // 发送进度
    }
}
```

### 7. Dev 模式 Dock 显示英文名

开发模式下 macOS Dock 显示的是 Cargo 包名（英文），不是配置的中文名。

**原因**：这是正常的，Cargo 包名不支持中文。打包后会显示 `tauri.conf.json` 中配置的 `productName`。

### 8. Release 比 Debug 快很多

开发时觉得去重速度一般，打包后发现快了好几倍。

**原因**：Debug 模式无优化，Release 模式有 LTO、内联等优化。对于 CPU 密集型任务，Release 版本可能快 3-5 倍。

### 9. 打包后找不到 FFmpeg

开发模式正常，打包后报错找不到 ffmpeg。

**原因**：Tauri 的 `externalBin` 在开发模式和打包后的路径不同。开发时在 `src-tauri/binaries/`，打包后在可执行文件同目录，且文件名不带平台后缀。

**解决方案**：封装一个统一的路径查找函数：

```rust
pub fn get_ffmpeg_path(app: &AppHandle) -> String {
    // 1. 先找可执行文件同目录（打包后）
    if let Ok(exe_dir) = std::env::current_exe().and_then(|p| Ok(p.parent().unwrap().to_path_buf())) {
        let path = exe_dir.join("ffmpeg");
        if path.exists() { return path.to_string_lossy().to_string(); }
    }
    // 2. 再找 resource_dir（开发模式）
    // 3. 最后回退到系统 PATH
    "ffmpeg".to_string()
}
```

### 10. GitHub Release 显示旧版本

推送了新 tag，但 Release 页面还是显示旧版本为 "Latest"。

**原因**：workflow 配置了 `releaseDraft: true`，新版本都是草稿状态。

**解决方案**：改成 `releaseDraft: false`，或者手动去 Release 页面发布草稿。

## 打包发布

### 本地打包

```bash
# 1. 安装 DMG 打包工具（macOS）
brew install create-dmg

# 2. 下载 FFmpeg 静态版本
curl -L "https://evermeet.cx/ffmpeg/getrelease/ffmpeg/zip" -o /tmp/ffmpeg.zip
unzip /tmp/ffmpeg.zip -d src-tauri/binaries/
mv src-tauri/binaries/ffmpeg src-tauri/binaries/ffmpeg-x86_64-apple-darwin

# 3. 执行打包
pnpm tauri build
```

### GitHub Actions 多平台自动打包

本地只能打包当前平台。要支持 Windows、Linux，用 GitHub Actions：

```yaml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  build:
    strategy:
      matrix:
        include:
          - platform: macos-latest
            target: x86_64-apple-darwin
          - platform: macos-latest
            target: aarch64-apple-darwin
          - platform: windows-latest
            target: x86_64-pc-windows-msvc
          - platform: ubuntu-22.04
            target: x86_64-unknown-linux-gnu

    runs-on: ${{ matrix.platform }}
    steps:
      # ... 安装环境、下载 FFmpeg、打包
```

推送 tag 后自动触发，4 个平台并行打包，完成后上传到 Release。

### 版本管理脚本

写了个 `tag.sh` 脚本简化发版流程：

```bash
./tag.sh
# 📌 当前版本: v0.2.5
# 选择操作:
#   1) 补丁版本 v0.2.6 (bug修复)
#   2) 次版本   v0.3.0 (新功能)
#   3) 主版本   v1.0.0 (重大更新)
```

脚本会自动更新 `tauri.conf.json` 和 `package.json` 中的版本号，提交并推送 tag。

## 总结

从整理重复文件的小需求开始，一路加功能，最终做成了「小文喵」这个小工具。

这个项目让我对 Tauri 2.0 有了更深的理解：

1. **Rust 性能确实强**：文件哈希、并行计算等场景优势明显
2. **Tauri 开发体验不错**：前后端分离，IPC 通信简单
3. **包体积小**：不含 FFmpeg 只有 10MB 左右
4. **跨平台**：一套代码，多端运行

踩的坑也不少，但都是宝贵的经验。希望这篇文章对想尝试 Tauri 的同学有所帮助。

> GitHub: [https://github.com/220529/file-toolkit](https://github.com/220529/file-toolkit)
> 
> 欢迎 Star ⭐️

---

**相关文章**：
- [Tauri 官方文档](https://tauri.app/)
- [xxHash 算法介绍](https://xxhash.com/)
- [FFmpeg 滤镜文档](https://ffmpeg.org/ffmpeg-filters.html)
