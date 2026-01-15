# 跨平台文件工具箱技术方案

> 可视化文件管理工具，支持 Mac/Windows，文件统计、去重、视频处理等功能

## 一、需求说明

### 跨平台要求
- ✅ 支持 macOS
- ✅ 支持 Windows
- ✅ 支持 Linux（可选）

### 核心功能

#### 1. 文件查看（统计）
- 选择/拖入一个文件夹
- **递归遍历**所有子文件夹
- 输出：各文件格式、对应个数、总大小

#### 2. 文件去重
- 选择一个文件夹，递归扫描所有文件
- 找出重复文件（基于内容 MD5）
- **标出重复文件，展示给用户确认**
- 用户勾选后点击"删除"，才执行删除操作
- 支持预览重复文件

#### 3. 视频截取
- 选择视频文件
- 设置开始时间、结束时间（如 00:01:30 到 00:02:45）
- 导出截取后的视频片段

#### 4. 视频清晰度还原（AI 超分）
- 选择模糊/低分辨率视频
- AI 处理，输出高清版本
- 支持 2x / 4x 放大

## 二、技术选型

### 为什么选 Tauri？

| 对比项 | Electron | Tauri | 纯 Web |
|--------|----------|-------|--------|
| 包体大小 | 150MB+ | 5-10MB | 0 |
| 内存占用 | 高 | 低 | - |
| 文件系统访问 | ✅ | ✅ | ⚠️ 受限 |
| 视频处理 | ✅ 调用 FFmpeg | ✅ 调用 FFmpeg | ❌ |
| 跨平台 | ✅ | ✅ | ✅ |
| 开发语言 | JS | Rust + JS | JS |

**结论**：Tauri 包体小、性能好，适合这类工具型应用。

### 技术栈

```
前端：React/Vue + TypeScript + TailwindCSS
后端：Tauri (Rust)
视频处理：FFmpeg（内置或系统调用）
文件哈希：Rust 原生实现（性能好）
AI 超分：Real-ESRGAN / Topaz Video AI API
```

## 三、架构设计

```
┌─────────────────────────────────────────────────────────┐
│                      Tauri 应用                          │
│  ┌───────────────────────────────────────────────────┐  │
│  │                   前端 (WebView)                    │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │  │
│  │  │文件统计  │ │文件去重  │ │视频处理  │ │ 设置    │  │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘  │  │
│  └───────────────────────┬───────────────────────────┘  │
│                          │ Tauri IPC                     │
│  ┌───────────────────────▼───────────────────────────┐  │
│  │                   Rust 后端                         │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │  │
│  │  │文件遍历  │ │MD5计算   │ │FFmpeg   │ │AI超分   │  │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## 四、核心功能实现

### 4.1 文件统计

**Rust 后端**（高性能递归遍历）：
```rust
// src-tauri/src/commands/file_stats.rs
use std::collections::HashMap;
use std::path::Path;
use walkdir::WalkDir;
use serde::Serialize;

#[derive(Serialize)]
pub struct FileStats {
    pub extension: String,
    pub count: u64,
    pub total_size: u64,
}

#[tauri::command]
pub fn scan_directory(path: String) -> Result<Vec<FileStats>, String> {
    let mut stats: HashMap<String, (u64, u64)> = HashMap::new();
    
    for entry in WalkDir::new(&path).into_iter().filter_map(|e| e.ok()) {
        if entry.file_type().is_file() {
            let ext = entry.path()
                .extension()
                .map(|e| e.to_string_lossy().to_lowercase())
                .unwrap_or_default();
            
            let size = entry.metadata().map(|m| m.len()).unwrap_or(0);
            
            let stat = stats.entry(ext).or_insert((0, 0));
            stat.0 += 1;
            stat.1 += size;
        }
    }
    
    let mut result: Vec<FileStats> = stats
        .into_iter()
        .map(|(ext, (count, size))| FileStats {
            extension: if ext.is_empty() { "(无扩展名)".into() } else { format!(".{}", ext) },
            count,
            total_size: size,
        })
        .collect();
    
    result.sort_by(|a, b| b.count.cmp(&a.count));
    Ok(result)
}
```

**前端调用**：
```typescript
// src/pages/FileStats.tsx
import { invoke } from '@tauri-apps/api/tauri';
import { open } from '@tauri-apps/api/dialog';

interface FileStats {
  extension: string;
  count: number;
  total_size: number;
}

async function selectAndScan() {
  const selected = await open({ directory: true });
  if (selected) {
    const stats = await invoke<FileStats[]>('scan_directory', { path: selected });
    setStats(stats);
  }
}
```

### 4.2 文件去重（带用户确认）

**Rust 后端**：
```rust
// src-tauri/src/commands/dedup.rs
use md5::{Md5, Digest};
use std::collections::HashMap;
use std::fs::{self, File};
use std::io::Read;
use std::path::Path;
use walkdir::WalkDir;
use serde::Serialize;

#[derive(Serialize, Clone)]
pub struct FileInfo {
    pub path: String,
    pub name: String,
    pub size: u64,
    pub modified: u64,  // 修改时间戳
}

#[derive(Serialize)]
pub struct DuplicateGroup {
    pub hash: String,
    pub size: u64,
    pub files: Vec<FileInfo>,  // 重复的文件列表
}

/// 第一步：扫描并返回重复文件（不删除，等用户确认）
#[tauri::command]
pub fn find_duplicates(path: String) -> Result<Vec<DuplicateGroup>, String> {
    let mut size_map: HashMap<u64, Vec<String>> = HashMap::new();
    
    // 1. 先按文件大小分组（快速筛选）
    for entry in WalkDir::new(&path).into_iter().filter_map(|e| e.ok()) {
        if entry.file_type().is_file() {
            if let Ok(meta) = entry.metadata() {
                let size = meta.len();
                if size > 0 {  // 忽略空文件
                    size_map.entry(size).or_default()
                        .push(entry.path().to_string_lossy().to_string());
                }
            }
        }
    }
    
    // 2. 只对大小相同的文件计算 MD5
    let mut hash_map: HashMap<String, Vec<FileInfo>> = HashMap::new();
    
    for (size, files) in size_map {
        if files.len() < 2 { continue; }  // 大小唯一，不可能重复
        
        for file_path in files {
            if let Ok(hash) = calculate_md5(Path::new(&file_path)) {
                let meta = fs::metadata(&file_path).ok();
                let file_info = FileInfo {
                    name: Path::new(&file_path)
                        .file_name()
                        .map(|n| n.to_string_lossy().to_string())
                        .unwrap_or_default(),
                    path: file_path,
                    size,
                    modified: meta.and_then(|m| m.modified().ok())
                        .and_then(|t| t.duration_since(std::time::UNIX_EPOCH).ok())
                        .map(|d| d.as_secs())
                        .unwrap_or(0),
                };
                hash_map.entry(hash).or_default().push(file_info);
            }
        }
    }
    
    // 3. 只返回有重复的组
    let duplicates: Vec<DuplicateGroup> = hash_map
        .into_iter()
        .filter(|(_, files)| files.len() > 1)
        .map(|(hash, files)| DuplicateGroup {
            hash,
            size: files[0].size,
            files,
        })
        .collect();
    
    Ok(duplicates)
}

/// 第二步：用户确认后，删除指定文件
#[tauri::command]
pub fn delete_files(paths: Vec<String>) -> Result<u32, String> {
    let mut deleted = 0;
    for path in paths {
        if fs::remove_file(&path).is_ok() {
            deleted += 1;
        }
    }
    Ok(deleted)
}

fn calculate_md5(path: &Path) -> Result<String, String> {
    let mut file = File::open(path).map_err(|e| e.to_string())?;
    let mut hasher = Md5::new();
    let mut buffer = [0u8; 65536];  // 64KB buffer
    
    loop {
        let bytes_read = file.read(&mut buffer).map_err(|e| e.to_string())?;
        if bytes_read == 0 { break; }
        hasher.update(&buffer[..bytes_read]);
    }
    
    Ok(format!("{:x}", hasher.finalize()))
}
```

**前端交互流程**：
```typescript
// src/pages/Dedup.tsx
import { useState } from 'react';
import { invoke } from '@tauri-apps/api/tauri';
import { open, confirm } from '@tauri-apps/api/dialog';

interface FileInfo {
  path: string;
  name: string;
  size: number;
  modified: number;
}

interface DuplicateGroup {
  hash: string;
  size: number;
  files: FileInfo[];
}

export default function Dedup() {
  const [groups, setGroups] = useState<DuplicateGroup[]>([]);
  const [selected, setSelected] = useState<Set<string>>(new Set());
  const [scanning, setScanning] = useState(false);

  // 1. 选择文件夹并扫描
  async function scanFolder() {
    const folder = await open({ directory: true, title: '选择要扫描的文件夹' });
    if (!folder) return;

    setScanning(true);
    try {
      const result = await invoke<DuplicateGroup[]>('find_duplicates', { path: folder });
      setGroups(result);
      setSelected(new Set());
    } finally {
      setScanning(false);
    }
  }

  // 2. 切换选中状态
  function toggleSelect(path: string) {
    const newSelected = new Set(selected);
    if (newSelected.has(path)) {
      newSelected.delete(path);
    } else {
      newSelected.add(path);
    }
    setSelected(newSelected);
  }

  // 3. 智能选择：每组保留最早的，选中其余的
  function autoSelect() {
    const toDelete = new Set<string>();
    groups.forEach(group => {
      const sorted = [...group.files].sort((a, b) => a.modified - b.modified);
      sorted.slice(1).forEach(f => toDelete.add(f.path));  // 保留第一个（最早）
    });
    setSelected(toDelete);
  }

  // 4. 确认删除
  async function deleteSelected() {
    if (selected.size === 0) return;

    const confirmed = await confirm(
      `确定要删除选中的 ${selected.size} 个文件吗？\n此操作不可恢复！`,
      { title: '确认删除', type: 'warning' }
    );

    if (confirmed) {
      const deleted = await invoke<number>('delete_files', { paths: Array.from(selected) });
      alert(`成功删除 ${deleted} 个文件`);
      // 重新扫描
      scanFolder();
    }
  }

  return (
    <div className="dedup-page">
      <div className="toolbar">
        <button onClick={scanFolder} disabled={scanning}>
          {scanning ? '扫描中...' : '选择文件夹'}
        </button>
        <button onClick={autoSelect} disabled={groups.length === 0}>
          智能选择（保留最早）
        </button>
        <button onClick={deleteSelected} disabled={selected.size === 0} className="danger">
          删除选中 ({selected.size})
        </button>
      </div>

      {groups.length > 0 && (
        <div className="summary">
          发现 {groups.length} 组重复文件，共 {groups.reduce((sum, g) => sum + g.files.length, 0)} 个文件
        </div>
      )}

      <div className="duplicate-list">
        {groups.map((group, idx) => (
          <div key={group.hash} className="duplicate-group">
            <div className="group-header">
              第 {idx + 1} 组 | {formatSize(group.size)} | {group.files.length} 个文件
            </div>
            {group.files.map(file => (
              <div 
                key={file.path} 
                className={`file-item ${selected.has(file.path) ? 'selected' : ''}`}
                onClick={() => toggleSelect(file.path)}
              >
                <input 
                  type="checkbox" 
                  checked={selected.has(file.path)}
                  onChange={() => toggleSelect(file.path)}
                />
                <span className="file-name">{file.name}</span>
                <span className="file-path">{file.path}</span>
              </div>
            ))}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 4.3 视频截取（指定时间段）

**Rust 后端**：
```rust
// src-tauri/src/commands/video.rs
use std::process::Command;
use tauri::Window;

/// 解析时间字符串为秒数，支持多种格式
/// "90" -> 90秒
/// "1:30" -> 90秒  
/// "00:01:30" -> 90秒
fn parse_time(time_str: &str) -> Result<f64, String> {
    let parts: Vec<&str> = time_str.split(':').collect();
    match parts.len() {
        1 => parts[0].parse::<f64>().map_err(|e| e.to_string()),
        2 => {
            let mins: f64 = parts[0].parse().map_err(|e: std::num::ParseFloatError| e.to_string())?;
            let secs: f64 = parts[1].parse().map_err(|e: std::num::ParseFloatError| e.to_string())?;
            Ok(mins * 60.0 + secs)
        }
        3 => {
            let hours: f64 = parts[0].parse().map_err(|e: std::num::ParseFloatError| e.to_string())?;
            let mins: f64 = parts[1].parse().map_err(|e: std::num::ParseFloatError| e.to_string())?;
            let secs: f64 = parts[2].parse().map_err(|e: std::num::ParseFloatError| e.to_string())?;
            Ok(hours * 3600.0 + mins * 60.0 + secs)
        }
        _ => Err("时间格式错误".into()),
    }
}

#[tauri::command]
pub async fn cut_video(
    input: String,
    output: String,
    start_time: String,  // "00:01:30" 或 "90"
    end_time: String,    // "00:02:45" 或 "165"
    window: Window,
) -> Result<String, String> {
    let start_secs = parse_time(&start_time)?;
    let end_secs = parse_time(&end_time)?;
    
    if end_secs <= start_secs {
        return Err("结束时间必须大于开始时间".into());
    }
    
    let duration = end_secs - start_secs;
    
    // 使用 -ss 在 -i 前面可以快速定位（seek）
    let status = Command::new("ffmpeg")
        .args([
            "-y",  // 覆盖输出文件
            "-ss", &format!("{}", start_secs),
            "-i", &input,
            "-t", &format!("{}", duration),
            "-c", "copy",  // 无损截取，速度快
            "-avoid_negative_ts", "make_zero",
            &output
        ])
        .status()
        .map_err(|e| format!("FFmpeg 执行失败: {}", e))?;
    
    if status.success() {
        Ok(output)
    } else {
        Err("视频截取失败".into())
    }
}

/// 获取视频时长（用于前端显示）
#[tauri::command]
pub fn get_video_duration(path: String) -> Result<f64, String> {
    let output = Command::new("ffprobe")
        .args([
            "-v", "error",
            "-show_entries", "format=duration",
            "-of", "default=noprint_wrappers=1:nokey=1",
            &path
        ])
        .output()
        .map_err(|e| e.to_string())?;
    
    let duration_str = String::from_utf8_lossy(&output.stdout);
    duration_str.trim().parse::<f64>().map_err(|e| e.to_string())
}
```

**前端界面**：
```typescript
// src/pages/VideoCut.tsx
import { useState } from 'react';
import { invoke } from '@tauri-apps/api/tauri';
import { open, save } from '@tauri-apps/api/dialog';

export default function VideoCut() {
  const [videoPath, setVideoPath] = useState('');
  const [duration, setDuration] = useState(0);
  const [startTime, setStartTime] = useState('00:00:00');
  const [endTime, setEndTime] = useState('00:00:00');
  const [processing, setProcessing] = useState(false);

  async function selectVideo() {
    const file = await open({
      filters: [{ name: '视频文件', extensions: ['mp4', 'mov', 'avi', 'mkv', 'wmv', 'flv'] }]
    });
    if (file && typeof file === 'string') {
      setVideoPath(file);
      const dur = await invoke<number>('get_video_duration', { path: file });
      setDuration(dur);
      setEndTime(formatTime(dur));
    }
  }

  async function cutVideo() {
    const outputPath = await save({
      defaultPath: 'output.mp4',
      filters: [{ name: '视频文件', extensions: ['mp4'] }]
    });
    if (!outputPath) return;

    setProcessing(true);
    try {
      await invoke('cut_video', {
        input: videoPath,
        output: outputPath,
        startTime,
        endTime,
      });
      alert('截取完成！');
    } catch (e) {
      alert('截取失败: ' + e);
    } finally {
      setProcessing(false);
    }
  }

  return (
    <div className="video-cut-page">
      <button onClick={selectVideo}>选择视频</button>
      
      {videoPath && (
        <>
          <video src={`file://${videoPath}`} controls style={{ maxWidth: '100%' }} />
          <p>视频时长: {formatTime(duration)}</p>
          
          <div className="time-inputs">
            <label>
              开始时间:
              <input value={startTime} onChange={e => setStartTime(e.target.value)} placeholder="00:01:30" />
            </label>
            <label>
              结束时间:
              <input value={endTime} onChange={e => setEndTime(e.target.value)} placeholder="00:02:45" />
            </label>
          </div>
          
          <button onClick={cutVideo} disabled={processing}>
            {processing ? '处理中...' : '开始截取'}
          </button>
        </>
      )}
    </div>
  );
}

function formatTime(seconds: number): string {
  const h = Math.floor(seconds / 3600);
  const m = Math.floor((seconds % 3600) / 60);
  const s = Math.floor(seconds % 60);
  return `${h.toString().padStart(2, '0')}:${m.toString().padStart(2, '0')}:${s.toString().padStart(2, '0')}`;
}
```

### 4.4 视频清晰度还原（AI 超分辨率）

有两种实现方案：

#### 方案 A：本地 Real-ESRGAN（免费，需要 GPU）

```rust
// src-tauri/src/commands/upscale.rs
use std::process::Command;
use std::path::Path;
use std::fs;

/// 视频超分辨率处理流程：
/// 1. 视频拆帧 -> 临时图片序列
/// 2. 逐帧 AI 超分
/// 3. 图片序列合成视频
/// 4. 合并原音轨
#[tauri::command]
pub async fn upscale_video(
    input: String,
    output: String,
    scale: u32,  // 2 或 4
    window: tauri::Window,
) -> Result<String, String> {
    let temp_dir = std::env::temp_dir().join("video_upscale");
    let frames_dir = temp_dir.join("frames");
    let upscaled_dir = temp_dir.join("upscaled");
    
    // 清理并创建临时目录
    let _ = fs::remove_dir_all(&temp_dir);
    fs::create_dir_all(&frames_dir).map_err(|e| e.to_string())?;
    fs::create_dir_all(&upscaled_dir).map_err(|e| e.to_string())?;
    
    // 1. 提取帧
    window.emit("upscale_progress", "正在提取视频帧...").ok();
    let status = Command::new("ffmpeg")
        .args([
            "-i", &input,
            "-qscale:v", "2",
            &format!("{}/frame_%06d.png", frames_dir.display())
        ])
        .status()
        .map_err(|e| e.to_string())?;
    
    if !status.success() {
        return Err("提取帧失败".into());
    }
    
    // 2. AI 超分（使用 Real-ESRGAN）
    window.emit("upscale_progress", "正在 AI 超分处理...").ok();
    let realesrgan_path = get_realesrgan_path();
    let status = Command::new(&realesrgan_path)
        .args([
            "-i", &frames_dir.to_string_lossy(),
            "-o", &upscaled_dir.to_string_lossy(),
            "-s", &scale.to_string(),
            "-n", "realesrgan-x4plus",  // 模型名称
        ])
        .status()
        .map_err(|e| e.to_string())?;
    
    if !status.success() {
        return Err("AI 超分处理失败".into());
    }
    
    // 3. 获取原视频帧率
    let fps = get_video_fps(&input)?;
    
    // 4. 合成视频（不含音频）
    window.emit("upscale_progress", "正在合成视频...").ok();
    let temp_video = temp_dir.join("temp_no_audio.mp4");
    let status = Command::new("ffmpeg")
        .args([
            "-framerate", &fps.to_string(),
            "-i", &format!("{}/frame_%06d.png", upscaled_dir.display()),
            "-c:v", "libx264",
            "-crf", "18",
            "-pix_fmt", "yuv420p",
            &temp_video.to_string_lossy()
        ])
        .status()
        .map_err(|e| e.to_string())?;
    
    if !status.success() {
        return Err("合成视频失败".into());
    }
    
    // 5. 合并原音轨
    window.emit("upscale_progress", "正在合并音轨...").ok();
    let status = Command::new("ffmpeg")
        .args([
            "-i", &temp_video.to_string_lossy(),
            "-i", &input,
            "-c:v", "copy",
            "-c:a", "aac",
            "-map", "0:v:0",
            "-map", "1:a:0?",
            "-y",
            &output
        ])
        .status()
        .map_err(|e| e.to_string())?;
    
    // 清理临时文件
    let _ = fs::remove_dir_all(&temp_dir);
    
    if status.success() {
        Ok(output)
    } else {
        Err("合并音轨失败".into())
    }
}

fn get_video_fps(path: &str) -> Result<f64, String> {
    let output = Command::new("ffprobe")
        .args([
            "-v", "error",
            "-select_streams", "v:0",
            "-show_entries", "stream=r_frame_rate",
            "-of", "default=noprint_wrappers=1:nokey=1",
            path
        ])
        .output()
        .map_err(|e| e.to_string())?;
    
    let fps_str = String::from_utf8_lossy(&output.stdout);
    // 格式可能是 "30/1" 或 "29.97"
    let fps_str = fps_str.trim();
    if fps_str.contains('/') {
        let parts: Vec<&str> = fps_str.split('/').collect();
        let num: f64 = parts[0].parse().unwrap_or(30.0);
        let den: f64 = parts[1].parse().unwrap_or(1.0);
        Ok(num / den)
    } else {
        fps_str.parse().map_err(|e: std::num::ParseFloatError| e.to_string())
    }
}

#[cfg(target_os = "windows")]
fn get_realesrgan_path() -> String {
    "resources/realesrgan-ncnn-vulkan.exe".into()
}

#[cfg(not(target_os = "windows"))]
fn get_realesrgan_path() -> String {
    "resources/realesrgan-ncnn-vulkan".into()
}
```

#### 方案 B：调用云端 API（推荐普通用户）

```rust
// 调用第三方 API，如：
// - 腾讯云视频处理
// - 阿里云视频增强
// - Topaz Video AI API

#[tauri::command]
pub async fn upscale_video_cloud(
    input: String,
    api_key: String,
) -> Result<String, String> {
    // 1. 上传视频到云端
    // 2. 调用 API 处理
    // 3. 轮询等待完成
    // 4. 下载处理后的视频
    todo!()
}
```

**前端界面**：
```typescript
// src/pages/VideoUpscale.tsx
import { useState, useEffect } from 'react';
import { invoke } from '@tauri-apps/api/tauri';
import { open, save } from '@tauri-apps/api/dialog';
import { listen } from '@tauri-apps/api/event';

export default function VideoUpscale() {
  const [videoPath, setVideoPath] = useState('');
  const [scale, setScale] = useState(2);
  const [processing, setProcessing] = useState(false);
  const [progress, setProgress] = useState('');

  useEffect(() => {
    const unlisten = listen('upscale_progress', (event) => {
      setProgress(event.payload as string);
    });
    return () => { unlisten.then(fn => fn()); };
  }, []);

  async function selectVideo() {
    const file = await open({
      filters: [{ name: '视频文件', extensions: ['mp4', 'mov', 'avi', 'mkv'] }]
    });
    if (file && typeof file === 'string') {
      setVideoPath(file);
    }
  }

  async function startUpscale() {
    const outputPath = await save({
      defaultPath: 'upscaled.mp4',
      filters: [{ name: '视频文件', extensions: ['mp4'] }]
    });
    if (!outputPath) return;

    setProcessing(true);
    setProgress('准备中...');
    try {
      await invoke('upscale_video', {
        input: videoPath,
        output: outputPath,
        scale,
      });
      alert('处理完成！');
    } catch (e) {
      alert('处理失败: ' + e);
    } finally {
      setProcessing(false);
      setProgress('');
    }
  }

  return (
    <div className="upscale-page">
      <h2>视频清晰度还原</h2>
      
      <button onClick={selectVideo}>选择视频</button>
      {videoPath && <p>已选择: {videoPath}</p>}
      
      <div className="scale-options">
        <label>
          <input type="radio" value={2} checked={scale === 2} onChange={() => setScale(2)} />
          2x 放大
        </label>
        <label>
          <input type="radio" value={4} checked={scale === 4} onChange={() => setScale(4)} />
          4x 放大（更清晰，更慢）
        </label>
      </div>
      
      <button onClick={startUpscale} disabled={!videoPath || processing}>
        {processing ? progress : '开始处理'}
      </button>
      
      <p className="tip">
        提示：视频超分处理较慢，1分钟视频可能需要 10-30 分钟处理时间
      </p>
    </div>
  );
}
```

## 五、UI 设计

```
┌─────────────────────────────────────────────────────────┐
│  📁 文件工具箱                              ─  □  ×    │
├─────────────────────────────────────────────────────────┤
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐           │
│  │📊 统计  │ │🔍 去重  │ │✂️ 截取  │ │✨ 超分  │           │
│  └────────┘ └────────┘ └────────┘ └────────┘           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  【文件去重页面示例】                                    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  📁 选择文件夹    🔄 智能选择    �️  删除选中(3)  │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  发现 5 组重复文件，共 12 个文件，可释放 156 MB         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 第 1 组 | 2.3 MB | 3 个文件                      │   │
│  │ ┌─────────────────────────────────────────────┐ │   │
│  │ │ ☐ photo_001.jpg                             │ │   │
│  │ │   /Users/xxx/Downloads/photo_001.jpg        │ │   │
│  │ ├─────────────────────────────────────────────┤ │   │
│  │ │ ☑ photo_001(1).jpg                          │ │   │
│  │ │   /Users/xxx/Desktop/photo_001(1).jpg       │ │   │
│  │ ├─────────────────────────────────────────────┤ │   │
│  │ │ ☑ 副本_photo.jpg                            │ │   │
│  │ │   /Users/xxx/Documents/副本_photo.jpg       │ │   │
│  │ └─────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 第 2 组 | 15.6 MB | 2 个文件                     │   │
│  │ ...                                             │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

## 六、项目结构

```
file-toolkit/
├── src/                          # 前端代码
│   ├── components/
│   │   ├── DropZone.tsx          # 拖拽区域
│   │   ├── StatsTable.tsx        # 统计表格
│   │   ├── DuplicateList.tsx     # 重复文件列表
│   │   └── VideoEditor.tsx       # 视频编辑器
│   ├── pages/
│   │   ├── FileStats.tsx         # 文件统计页
│   │   ├── Dedup.tsx             # 去重页
│   │   ├── Rename.tsx            # 重命名页
│   │   └── Video.tsx             # 视频处理页
│   ├── App.tsx
│   └── main.tsx
├── src-tauri/                    # Rust 后端
│   ├── src/
│   │   ├── commands/
│   │   │   ├── file_stats.rs
│   │   │   ├── dedup.rs
│   │   │   ├── rename.rs
│   │   │   └── video.rs
│   │   ├── lib.rs
│   │   └── main.rs
│   ├── Cargo.toml
│   └── tauri.conf.json
├── package.json
└── README.md
```

## 七、FFmpeg 打包策略

### 方案 A：内置 FFmpeg（推荐）
- 打包时将 FFmpeg 二进制文件放入 resources
- 优点：开箱即用
- 缺点：包体增大 ~80MB

### 方案 B：系统 FFmpeg
- 检测系统是否安装 FFmpeg
- 未安装则引导用户安装
- 优点：包体小
- 缺点：用户体验差

```rust
// 检测 FFmpeg
fn check_ffmpeg() -> bool {
    Command::new("ffmpeg").arg("-version").output().is_ok()
}
```

## 八、开发计划

| 阶段 | 功能 | 时间 |
|------|------|------|
| P0.1 | 项目搭建 + 文件统计（递归遍历） | 2天 |
| P0.2 | 文件去重（MD5 + 用户确认删除） | 3天 |
| P1.1 | 视频截取（时间段选择） | 2天 |
| P1.2 | 视频清晰度还原（AI 超分） | 4天 |
| P2 | UI 优化 + 测试 + 打包发布 | 3天 |

**总计：约 14 天**

## 九、注意事项

### 文件去重安全性
- 删除前必须用户确认
- 提供"智能选择"但不自动删除
- 考虑加入"移动到回收站"而非直接删除

### 视频超分性能
- 本地处理需要 GPU，否则很慢
- 建议提供云端 API 选项
- 显示预估处理时间

### 跨平台兼容
- FFmpeg 需要分别打包 Win/Mac 版本
- Real-ESRGAN 同样需要多平台二进制
- 文件路径处理注意 `/` 和 `\` 差异

## 十、后续扩展

- [ ] 批量水印添加
- [ ] 图片 EXIF 信息查看/编辑
- [ ] 文件加密/解密
- [ ] 云存储同步（OSS/S3）
- [ ] 插件系统（支持自定义处理脚本）
