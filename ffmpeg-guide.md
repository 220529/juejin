# FFmpeg 入门指南

## 一、FFmpeg 是什么？

FFmpeg 是一个**开源的音视频处理工具**，被称为"音视频处理的瑞士军刀"。

几乎所有你能想到的音视频操作，它都能做：
- 视频格式转换（MP4 → AVI、MKV → MP4）
- 视频截取、裁剪、合并
- 视频压缩、调整分辨率
- 提取音频、添加字幕
- 直播推流、录屏
- ...

**谁在用？**
- YouTube、Netflix、Bilibili 等视频平台
- 微信、抖音等 App 的视频处理
- OBS、剪映等视频软件底层
- 几乎所有涉及音视频的软件

## 二、核心组件

FFmpeg 包含三个命令行工具：

| 工具 | 作用 |
|------|------|
| `ffmpeg` | 音视频转码、处理 |
| `ffprobe` | 查看音视频信息 |
| `ffplay` | 播放音视频 |

## 三、安装

### macOS
```bash
brew install ffmpeg
```

### Windows
1. 下载：https://ffmpeg.org/download.html
2. 解压到某个目录（如 `C:\ffmpeg`）
3. 添加 `C:\ffmpeg\bin` 到系统 PATH

### Linux
```bash
# Ubuntu/Debian
sudo apt install ffmpeg

# CentOS/RHEL
sudo yum install ffmpeg
```

### 验证安装
```bash
ffmpeg -version
```

## 四、常用命令

### 4.1 查看视频信息

```bash
ffprobe -v error -show_format -show_streams video.mp4
```

简化版（只看时长）：
```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 video.mp4
```

### 4.2 格式转换

```bash
# MP4 转 AVI
ffmpeg -i input.mp4 output.avi

# MKV 转 MP4
ffmpeg -i input.mkv -c copy output.mp4
```

### 4.3 视频截取

```bash
# 从第 10 秒开始，截取 30 秒
ffmpeg -i input.mp4 -ss 10 -t 30 -c copy output.mp4

# 从 00:01:30 到 00:02:45
ffmpeg -i input.mp4 -ss 00:01:30 -to 00:02:45 -c copy output.mp4
```

**参数说明**：
- `-ss`：开始时间
- `-t`：持续时长
- `-to`：结束时间
- `-c copy`：不重新编码，直接复制（快但不精确）

**精确截取**（重新编码，慢但精确）：
```bash
ffmpeg -i input.mp4 -ss 00:01:30 -to 00:02:45 -c:v libx264 -c:a aac output.mp4
```

### 4.4 视频压缩

```bash
# 通过 CRF 控制质量（18-28，越小质量越高）
ffmpeg -i input.mp4 -c:v libx264 -crf 23 output.mp4

# 指定码率
ffmpeg -i input.mp4 -b:v 1M output.mp4
```

### 4.5 调整分辨率

```bash
# 缩放到 1280x720
ffmpeg -i input.mp4 -vf scale=1280:720 output.mp4

# 保持比例，宽度 1280
ffmpeg -i input.mp4 -vf scale=1280:-1 output.mp4
```

### 4.6 提取音频

```bash
# 提取为 MP3
ffmpeg -i video.mp4 -vn -acodec mp3 audio.mp3

# 提取为 AAC
ffmpeg -i video.mp4 -vn -acodec copy audio.aac
```

### 4.7 视频合并

先创建文件列表 `list.txt`：
```
file 'video1.mp4'
file 'video2.mp4'
file 'video3.mp4'
```

然后合并：
```bash
ffmpeg -f concat -safe 0 -i list.txt -c copy output.mp4
```

### 4.8 添加水印

```bash
# 图片水印（右下角）
ffmpeg -i input.mp4 -i logo.png -filter_complex "overlay=W-w-10:H-h-10" output.mp4

# 文字水印
ffmpeg -i input.mp4 -vf "drawtext=text='Hello':fontsize=24:fontcolor=white:x=10:y=10" output.mp4
```

### 4.9 视频转 GIF

```bash
# 基础转换
ffmpeg -i input.mp4 -vf "fps=10,scale=320:-1" output.gif

# 高质量（先生成调色板）
ffmpeg -i input.mp4 -vf "fps=10,scale=320:-1:flags=lanczos,palettegen" palette.png
ffmpeg -i input.mp4 -i palette.png -filter_complex "fps=10,scale=320:-1:flags=lanczos[x];[x][1:v]paletteuse" output.gif
```

### 4.10 截取封面图

```bash
# 截取第 5 秒的画面
ffmpeg -i input.mp4 -ss 5 -vframes 1 cover.jpg

# 每秒截取一帧
ffmpeg -i input.mp4 -vf fps=1 frame_%04d.jpg
```

## 五、常用参数速查

| 参数 | 说明 |
|------|------|
| `-i` | 输入文件 |
| `-c copy` | 不重新编码，直接复制流 |
| `-c:v` | 视频编码器（libx264, libx265, copy） |
| `-c:a` | 音频编码器（aac, mp3, copy） |
| `-ss` | 开始时间 |
| `-t` | 持续时长 |
| `-to` | 结束时间 |
| `-vf` | 视频滤镜 |
| `-af` | 音频滤镜 |
| `-b:v` | 视频码率 |
| `-b:a` | 音频码率 |
| `-crf` | 质量（0-51，越小越好） |
| `-r` | 帧率 |
| `-s` | 分辨率（如 1920x1080） |
| `-vn` | 去掉视频 |
| `-an` | 去掉音频 |
| `-y` | 覆盖输出文件 |

## 六、在代码中使用

### Node.js
```javascript
const { exec } = require('child_process');

exec('ffmpeg -i input.mp4 -ss 10 -t 30 -c copy output.mp4', (err, stdout, stderr) => {
  if (err) {
    console.error('Error:', stderr);
    return;
  }
  console.log('Done!');
});
```

### Python
```python
import subprocess

subprocess.run([
    'ffmpeg', '-i', 'input.mp4',
    '-ss', '10', '-t', '30',
    '-c', 'copy', 'output.mp4'
])
```

### Rust
```rust
use std::process::Command;

let status = Command::new("ffmpeg")
    .args(["-i", "input.mp4", "-ss", "10", "-t", "30", "-c", "copy", "output.mp4"])
    .status()
    .expect("failed to execute ffmpeg");
```

## 七、性能优化

### 硬件加速

```bash
# macOS (VideoToolbox)
ffmpeg -i input.mp4 -c:v h264_videotoolbox output.mp4

# NVIDIA GPU (NVENC)
ffmpeg -i input.mp4 -c:v h264_nvenc output.mp4

# Intel (QSV)
ffmpeg -i input.mp4 -c:v h264_qsv output.mp4
```

### 多线程

```bash
ffmpeg -i input.mp4 -threads 4 -c:v libx264 output.mp4
```

## 八、常见问题

### Q: `-c copy` 截取不精确？
A: `-c copy` 只能在关键帧处切割。如需精确，去掉 `-c copy` 让它重新编码。

### Q: 输出文件比原文件大？
A: 可能是编码参数问题，尝试加 `-crf 23` 或指定码率。

### Q: 中文路径/文件名报错？
A: Windows 下可能有编码问题，尽量用英文路径。

### Q: 如何查看支持的编码器？
```bash
ffmpeg -encoders
ffmpeg -decoders
```

## 九、学习资源

- 官方文档：https://ffmpeg.org/documentation.html
- FFmpeg Wiki：https://trac.ffmpeg.org/wiki
- 在线命令生成器：https://ffmpeg.guide/
