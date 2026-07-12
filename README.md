# Video Progressbar Skill

AI 驱动的视频进度条生成工具，为 TRAE IDE 设计的 Skill。根据视频脚本语义自动分段，生成交互式预览页面，最终用 ffmpeg 将进度条烧录到视频中。

## 效果演示

进度条随视频播放从左到右动态填充：

| t=0s（无填充） | t=50%（半填充） | t=结束（全覆盖） |
|:---:|:---:|:---:|
| 空进度条 | 蓝色填充至一半 | 蓝色填满整条 |

## 功能特性

- **AI 语义分段**：根据纯文本脚本内容自动划分视频段落，也支持用户手动分段
- **交互式预览**：在 HTML 页面中实时调整颜色、位置、分段名显示，所见即所得
- **样式确认按钮**：在预览页中调整满意后点击「确认使用此样式」，自动下载 `selected_style.json`，AI 据此执行后续步骤，无需对话描述参数
- **2 个预设模板**：
  - 模板 A：底部线性进度条 + 分段刻度
  - 模板 B：底部块状分段条
- **动态进度填充**：基于 ffmpeg `geq` filter 逐像素逐帧动态绘制，t=0 无填充，随播放推进，t=结束全覆盖
- **紧靠视频边缘**：进度条紧贴视频底部，不留间距

## 工作流程

1. **获取视频元信息** — ffprobe 读取时长和分辨率
2. **询问用户是否已分段** — 支持用户手动输入或 AI 语义分段
3. **AI 解析脚本分段** — 基于脚本语义划分 3-8 段
4. **用户调整分段** — 可编辑 JSON 或对话反馈
5. **提取预览帧** — ffmpeg 取中点帧
6. **生成 HTML 预览页** — 交互式控件 + 样式确认按钮
7. **用户确认样式** — 在 HTML 中点击确认，下载 `selected_style.json`
8. **生成 overlay.png** — Pillow 绘制静态部分（边框、分隔线、分段名）
9. **ffmpeg 烧录** — `geq` 动态填充 + `overlay` 叠加静态层
10. **验证输出** — 提取关键帧确认动态效果

## 安装

### 方式一：全局安装（所有 TRAE 项目可用）

将 `SKILL.md` 放到 TRAE 全局 skills 目录：

```
C:\Users\<用户名>\.trae-cn\skills\video-progressbar\SKILL.md
```

### 方式二：项目级安装

将 `SKILL.md` 放到项目内：

```
<项目目录>\.trae\skills\video-progressbar\SKILL.md
```

### 依赖

- **ffmpeg / ffprobe**：视频处理与动态绘制
- **Python + Pillow**：生成进度条静态覆盖层 PNG

```bash
# Windows 安装 ffmpeg
winget install --id Gyan.FFmpeg

# 安装 Pillow
pip install Pillow
```

## 使用方法

1. 准备一个本地视频文件（MP4 等 ffmpeg 支持的格式）
2. 准备纯文本脚本（无时间戳，由 AI 解析语义分段）
3. 在 TRAE 中对 AI 说："用这个视频生成进度条"，并提供视频和脚本路径
4. AI 会引导你完成分段、预览、样式确认、烧录全流程
5. 最终输出 `<原文件名>_progressbar.mp4`

## 关键技术：为什么用 geq 而不是 drawbox

进度条动态填充必须用 ffmpeg 的 `geq` filter，不能用 `drawbox`：

| Filter | 动态性 | 原因 |
|--------|--------|------|
| `drawbox` | ❌ 不动态 | `w` 参数只在初始化时求值一次，不随时间变化 |
| `crop` / `scale` | ❌ 报错 | t/n 表达式报 "not valid in init eval_mode" |
| `geq` | ✅ 动态 | 逐像素逐帧求值，支持 T（时间）、W（宽度）、p(X,Y) 原像素 |

**geq 表达式示例**：

```bash
ffmpeg -i video.mp4 -i overlay.png \
  -filter_complex "[0:v]geq=r='if(gte(Y\,704)*lte(Y\,719)*lt(X\,W*T/30.0)\,59\,p(X\,Y))':g='...':b='...'[bg];[bg][1:v]overlay=0:0" \
  -c:a copy output_progressbar.mp4
```

- `gte(Y,704)*lte(Y,719)`：限定进度条 Y 坐标范围
- `lt(X,W*T/duration)`：限定 X 在已播放区域内
- `p(X,Y)`：未填充区域保留原像素
- `T`：当前时间秒，`W`：视频宽度

## 关键约束（防回退）

1. overlay.png 不画半透明色块（否则看不到动态填充）
2. 进度条紧靠视频边缘，不留间距
3. 不画 drawtext 章节标题
4. 只保留模板 A 和 B
5. 步骤2 必须先询问用户是否已分段
6. HTML 预览页必须交互式
7. 动态填充必须用 geq，禁止用 drawbox
8. 样式选择必须通过 HTML 确认按钮完成，禁止对话口述参数

## 文件产物

执行后在原视频同目录生成：

| 文件 | 说明 |
|------|------|
| `分段方案.json` | AI 生成的分段方案 |
| `预览帧.jpg` | 视频中点帧，HTML 预览背景 |
| `预览.html` | 交互式预览页 + 样式确认按钮 |
| `selected_style.json` | 用户确认的样式参数 |
| `overlay.png` | 进度条静态覆盖层 |
| `<原文件名>_progressbar.mp4` | 最终产物 |

## 仓库

- **GitHub**：https://github.com/ann777-oss/video-progressbar-skill
- **License**：MIT
