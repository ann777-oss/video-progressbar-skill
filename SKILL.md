---
name: "video-progressbar"
description: "AI-generates a video progress bar overlay based on script semantics and burns it into the video via ffmpeg. Invoke when the user provides a local video file and a plain-text script, and wants AI to auto-segment the video and add a progress bar overlay."
---

# 视频进度条生成 Skill

为本地视频生成 AI 驱动的进度条覆盖层，并烧录到视频中。

## 功能边界

- **输入**：本地视频文件路径（ffmpeg 支持的格式，如 MP4）+ 纯文本脚本（无时间戳/章节标记，由 AI 解析语义分段）
- **最终输出**：ffmpeg 烧录覆盖层后的 `<原文件名>_progressbar.mp4`，输出到原视频同目录
- **中间产物**：分段方案 JSON、HTML 预览页、视频帧图片、overlay.png
- **进度条显示内容**：动态填充 + 分段名称标签
- **不做的事**：不下载视频/脚本、不修改原视频文件、不生成视频片段预览（仅 HTML 静态预览）、不显示顶部章节标题

## 依赖

执行前必须验证以下工具可用，缺失则提示用户安装并加入 PATH：

- `ffmpeg` / `ffprobe`（视频时长读取、提取帧、烧录覆盖层、动态绘制）
- Python + Pillow (PIL)（生成进度条静态覆盖层 PNG）

验证命令：
```bash
ffmpeg -version
ffprobe -version
python -c "import PIL"
```

**PATH 刷新问题**：winget 安装后 PATH 在已运行进程的子终端不自动刷新。需用完整路径调用，或在新终端中执行。ffmpeg 默认路径示例：
`C:\Users\<用户名>\AppData\Local\Microsoft\WinGet\Packages\Gyan.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe\ffmpeg-8.1.2-full_build\bin`

## 2 个预设模板

每个模板可配置参数：`color`（主色调，默认 `#3B82F6`）、`position`（top/bottom，默认 bottom）、`show_segment_name`（true/false，默认 true）。

### 模板 A：底部线性进度条
- 位置：紧靠视频边缘（bottom 时紧靠底部，top 时紧靠顶部）
- 进度条：高度 8px，横跨视频宽度
- 分段刻度：每段边界处 2px 宽竖线，高度 12px
- 分段名标签：进度条内侧，每段中心，白色带阴影
- 动态填充：geq 按时间增长宽度（t=0 无填充，t=duration 全填充）

### 模板 B：底部块状分段条
- 位置：紧靠视频边缘
- 每分段一个色块，高度 16px，宽度 = 段占比 × 视频宽度
- 已播放段实色填充；当前段加高亮边框；未播放段半透明
- 分段名：色块内居中，白色
- 动态填充：geq 按时间增长宽度（t=0 无填充，t=duration 全填充）

## 工作流程

严格按以下顺序执行，每个步骤完成后向用户报告进展。

### 步骤 1：获取视频时长与元信息

```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "<视频路径>"
```

同时获取视频分辨率：
```bash
ffprobe -v error -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0 "<视频路径>"
```

### 步骤 2：询问用户是否已分段

**必须先询问用户**："你是否已经自己分好了视频段落？"

- **用户已分段**：让用户输入每段标题+时间戳，格式如下（每行一段）：
  ```
  开场介绍,0,4
  变量,4,10
  条件判断,10,16
  ```
  AI 解析用户输入，直接生成分段方案 JSON（跳过 AI 语义分段）
- **用户未分段**：AI 解析脚本语义分段（进入步骤 3）

### 步骤 3：AI 解析脚本语义，划分段落

读取纯文本脚本，根据脚本内容的主题、转折、章节语义，将视频时长划分为 N 段（N 由脚本内容决定，通常 3-8 段）。为每段生成简短名称（取自脚本语义，不超过 8 个字）。

输出分段方案 JSON（文件名 `分段方案.json`，保存到原视频同目录）：

```json
{
  "video": "<视频绝对路径>",
  "duration": 123.45,
  "width": 1920,
  "height": 1080,
  "segments": [
    {"name": "开场介绍", "start": 0.0, "end": 15.0},
    {"name": "核心讲解", "start": 15.0, "end": 60.0}
  ]
}
```

约束：
- `start` / `end` 单位为秒，浮点数
- 第一个段 `start` 必须为 0，最后一个段 `end` 必须等于 `duration`
- 相邻段 `end` == 下一段 `start`，不允许间隙或重叠

**分段完成后，必须停下来进入步骤4让用户确认，不得直接进入步骤5。**

### 步骤 4：用户确认分段（强制环节，不可跳过）

**AI 生成或接收分段方案后，必须在此步骤停下来，向用户展示完整分段方案并等待用户回复。禁止自动进入下一步。**

展示格式（在对话中直接输出，让用户一目了然）：

```
分段方案已生成，请确认：

1. [0.0s - 4.0s] 开场介绍
2. [4.0s - 10.0s] 变量
3. [10.0s - 16.0s] 条件判断
4. [16.0s - 21.0s] 循环
5. [21.0s - 26.0s] 函数
6. [26.0s - 30.0s] 课程小结

请回复：
- 「确认」/「可以了」/「没问题」→ 进入下一步
- 或直接说出修改意见，例如：
  - "把第2段拆成两段，在7秒处分"
  - "第3段名字改成'分支结构'"
  - "最后一段结束时间改成28秒"
  - "合并第4和第5段"
```

**处理用户回复**：

- **用户确认**（"确认"、"可以了"、"没问题"、"继续"等肯定词）→ 校验 JSON 格式与时间约束，通过后进入步骤5
- **用户提出修改意见** → AI 根据自然语言描述更新 `分段方案.json`，重新展示分段方案，再次请用户确认（循环直到用户确认）
- **用户未回复** → 保持等待，不得自行推进

调整后必须重新验证 JSON 格式与时间约束：
- 第一个段 `start` 必须为 0
- 最后一个段 `end` 必须等于 `duration`
- 相邻段 `end` == 下一段 `start`，无间隙无重叠
- 每段 `end` > `start`

不合法则报错并提示用户修正。**只有用户明确确认后，才能进入步骤5。**

**适用场景**：
- 步骤2 用户选择"AI 分段"时：AI 分完后在步骤4 等待确认
- 步骤2 用户选择"自己分段"时：AI 解析用户输入生成 JSON 后，也在步骤4 等待确认（用户可能输入有误，需复核）

### 步骤 5：提取视频预览帧

取视频中点帧作为 HTML 预览背景：
```bash
ffmpeg -ss "<duration/2>" -i "<视频路径>" -frames:v 1 -y "预览帧.jpg"
```

### 步骤 6：生成 HTML 预览页面（交互式 + 样式确认）

读取分段方案 JSON，生成 `预览.html`，要求：

**交互控件**（顶部控制面板）：
- 模板选择（A/B 单选按钮）
- 主色调颜色选择器（`<input type="color">`，默认 #3B82F6）
- 进度条位置选择（top/bottom 下拉）
- 是否显示分段名（checkbox）

**预览区**：
- 内嵌 `预览帧.jpg` 作为背景
- 用 CSS+JS 渲染选定模板，固定显示 50% 进度状态
- 参数变化时实时更新预览效果（onchange 触发）
- 页面显示当前参数摘要（如"模板: B | 颜色: #FF0000 | 位置: bottom | 显示分段名: 是"）

**样式确认功能**（核心，必须在 HTML 内实现）：
- 在预览区上方放置"确认使用此样式"按钮
- 点击按钮后，用 Blob 下载 `selected_style.json` 文件，内容为当前选择的参数：
  ```json
  {
    "template": "A",
    "color": "#3B82F6",
    "position": "bottom",
    "show_segment_name": true,
    "confirmed": true
  }
  ```
- 点击后页面显示成功提示："✓ 样式已保存！请将下载的 selected_style.json 放到视频所在目录，然后告诉 AI「样式已确认」继续。"
- 按钮点击逻辑（JS 实现）：
  ```javascript
  function confirmStyle() {
    const styleData = {
      template: <当前选中模板>,
      color: <当前选中颜色>,
      position: <当前选中位置>,
      show_segment_name: <当前 checkbox 状态>,
      confirmed: true
    };
    const blob = new Blob([JSON.stringify(styleData, null, 2)], {type: 'application/json'});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = 'selected_style.json';
    document.body.appendChild(a); a.click(); document.body.removeChild(a);
    URL.revokeObjectURL(url);
    // 显示成功提示
  }
  ```

**技术实现**：
- 用 JS 根据模板选择动态生成进度条 DOM
- 颜色用 `style.background = color` 实时应用
- 位置用 `style.bottom` 或 `style.top` 控制
- 分段名根据 checkbox 状态显示/隐藏
- 确认按钮通过 Blob + `<a download>` 触发文件下载（浏览器原生能力，无需额外依赖）

### 步骤 7：用户确认样式

向用户报告 `预览.html` 路径，提示用浏览器打开查看并调整参数。**用户通过 HTML 中的「确认使用此样式」按钮确定最终样式，而非通过对话反馈参数。**

流程：
1. 用户在浏览器中调整参数（模板、颜色、位置、分段名）
2. 满意后点击「确认使用此样式」按钮
3. 浏览器下载 `selected_style.json`
4. 用户将文件放到视频同目录（如已在同目录可跳过）
5. 用户在对话中说「样式已确认」或「选好了」
6. **AI 读取 `selected_style.json` 获取最终参数**，按此执行后续步骤

**读取样式 JSON**：
```bash
# AI 应读取原视频同目录下的 selected_style.json
# 文件不存在时提示用户先在 HTML 中点击确认按钮
```

读取后必须校验：
- `confirmed` 字段为 `true`
- `template` 为 "A" 或 "B"
- `color` 为合法十六进制色值（如 #3B82F6）
- `position` 为 "top" 或 "bottom"
- `show_segment_name` 为布尔值

校验失败时提示用户重新在 HTML 中确认。**禁止通过对话让用户口述参数**（如"你想用什么颜色"），参数选择必须通过 HTML 确认按钮完成。

### 步骤 8：PIL 生成进度条静态覆盖层 PNG

根据步骤7读取的 `selected_style.json` 中的模板与参数，用 Pillow 绘制进度条的**静态部分**，输出 `overlay.png`（与原视频同尺寸，RGBA 透明背景）：

**参数来源**：从 `selected_style.json` 读取 `template`、`color`、`position`、`show_segment_name`

**只画**：
- 进度条上下边框线（轨道轮廓）
- 段之间的白色分隔线
- 分段名称标签文字（白色带阴影，`show_segment_name=false` 时不画）

**不画**（由 ffmpeg geq 动态绘制）：
- ~~半透明色块~~（删除，否则会覆盖动态填充导致看不出进度变化）
- ~~动态填充~~（由 ffmpeg geq 按时间计算）

**进度条位置**：紧靠视频边缘（根据 `position` 参数）
- bottom：`bar_y = H - bar_h`（如 720-16=704）
- top：`bar_y = 0`

PNG 尺寸必须等于视频分辨率。

### 步骤 9：ffmpeg 烧录覆盖层 + 动态绘制

使用 `filter_complex` 组合两个操作：

1. `geq`：在原视频上动态绘制进度填充（按时间从左到右推进）
2. `overlay`：将 `overlay.png` 叠加到视频（分段名、分隔线、边框画在填充之上，文字始终可见）

**关键技术：必须用 geq，不能用 drawbox**
- `drawbox` 的 `w` 参数**不是动态的**，只在初始化时求值一次（`w='W*t/duration'` 在 t=0 时就被固定为某个值）
- `crop` / `scale` 的 `t`/`n` 表达式会报错 "Expressions with frame variables 'n', 't', 'pos' are not valid in init eval_mode"
- `geq` filter 支持 `T`（时间秒）、`W`（宽度）、`X`/`Y`（像素坐标）、`p(X,Y)`（原像素值）变量，且**逐像素逐帧动态求值**

**geq 表达式**：在进度条区域（`Y` 在 `bar_y` 到 `bar_y+bar_h-1`）且 `X < W*T/duration` 时填充指定颜色，否则保留原像素：

```
geq=r='if(gte(Y\,<bar_y>)*lte(Y\,<bar_y+bar_h-1>)*lt(X\,W*T/<duration>)\,<R>\,p(X\,Y))':
    g='if(gte(Y\,<bar_y>)*lte(Y\,<bar_y+bar_h-1>)*lt(X\,W*T/<duration>)\,<G>\,p(X\,Y))':
    b='if(gte(Y\,<bar_y>)*lte(Y\,<bar_y+bar_h-1>)*lt(X\,W*T/<duration>)\,<B>\,p(X\,Y))'
```

其中 `<R>`, `<G>`, `<B>` 是颜色的十进制 RGB 值（如蓝色 #3B82F6 → R=59, G=130, B=246）。

**filter_complex 结构**（geq 先执行，overlay 后执行）：
```bash
ffmpeg -i "<视频>" -i "overlay.png" \
  -filter_complex "[0:v]geq=r='<expr>':g='<expr>':b='<expr>'[bg];[bg][1:v]overlay=0:0" \
  -c:a copy "<原文件名>_progressbar.mp4"
```

注意：
- geq 中 filter_complex 的逗号必须转义为 `\,`
- geq 表达式中用单引号 `'` 包围
- `gte`/`lte`/`lt` 是比较函数（>=、<=、<），`*` 作逻辑与
- `p(X,Y)` 获取原像素值（当前通道）
- `T` 是当前时间（秒），`W` 是视频宽度
- overlay 在 geq 之后，确保分段名文字画在蓝色填充之上
- 音频流用 `-c:a copy` 直接复制

**验证动态效果**（必须执行）：提取 t=0、t=duration/2、t=duration-0.5 的帧，检查：
- t=0：进度条区域无填充色（0% 覆盖）
- t=duration/2：约 50% 覆盖
- t=duration-0.5：接近 100% 覆盖

### 步骤 10：验证输出

检查输出文件 `<原文件名>_progressbar.mp4` 存在且可播放：
```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "<原文件名>_progressbar.mp4"
```

输出时长应等于原视频时长。提取 3 个时间点截图验证动态效果：
```bash
ffmpeg -ss 5 -i "<输出文件>" -frames:v 1 -y "验证_5s.jpg"
ffmpeg -ss 15 -i "<输出文件>" -frames:v 1 -y "验证_15s.jpg"
ffmpeg -ss 25 -i "<输出文件>" -frames:v 1 -y "验证_25s.jpg"
```

向用户报告最终文件路径与验证帧。

## 默认值汇总

| 参数 | 默认值 |
|------|--------|
| 进度条位置 | bottom（紧靠底部） |
| 主色调 | #3B82F6 |
| 显示分段名 | true |
| 输出文件名 | `<原文件名>_progressbar.mp4` |
| 输出目录 | 原视频同目录 |
| 中间文件目录 | 原视频同目录 |

## 错误处理

- `ffmpeg` / `ffprobe` 不在 PATH：提示用户安装 ffmpeg，或使用完整路径调用
- `PIL` 未安装：提示用户执行 `pip install Pillow`
- 视频文件不存在或无法读取：报错停止，不继续后续步骤
- 脚本为空或无法识别分段：提示用户检查脚本内容
- ffmpeg geq 表达式语法错误：报告完整 ffmpeg 命令与错误日志

## 文件清单（执行后产物）

原视频同目录下生成：
- `分段方案.json`
- `预览帧.jpg`
- `预览.html`（交互式 + 样式确认按钮）
- `selected_style.json`（用户在 HTML 中点击确认按钮后生成，步骤8/9 的参数来源）
- `overlay.png`
- `<原文件名>_progressbar.mp4`（最终产物）

除最终视频外，中间文件保留供用户检查与调整，不自动清理。

## 关键约束（防止回退）

1. **overlay.png 不画半透明色块**：否则 geq 动态填充看不出来（问题4修复）
2. **进度条紧靠视频边缘**：bar_y = H - bar_h，不留间距（问题5修复）
3. **不画 drawtext 章节标题**：已从所有模板移除（问题2修复）
4. **只保留模板 A 和 B**：模板 C/D 已删除（问题2修复）
5. **步骤2 必须先询问用户是否已分段**：不臆测用户需要 AI 分段（问题1修复）
6. **HTML 预览页必须交互式**：支持参数实时调整（问题3修复）
7. **动态填充必须用 geq，禁止用 drawbox**：drawbox 的 w 参数只在初始化时求值一次，不随时间变化；geq 逐像素逐帧动态求值，t=0 时无填充，随时间推进，t=duration 时全覆盖（问题4核心修复）
8. **样式选择必须通过 HTML 确认按钮完成，禁止对话口述参数**：HTML 必须包含「确认使用此样式」按钮，点击后下载 `selected_style.json`；步骤8/9 的所有参数（模板、颜色、位置、分段名）必须从 `selected_style.json` 读取，禁止在对话中询问用户"你想用什么颜色"等参数问题
9. **分段方案必须经用户确认，禁止 AI 分完直接进入下一步**：步骤4 是强制环节，AI 生成分段方案后必须停下来展示完整分段列表（段名+时间范围），等待用户回复"确认"或提出修改意见；用户提出修改意见时 AI 调整后重新展示并再次确认，循环直到用户明确确认才能进入步骤5；步骤2 中用户自己分段时也必须经步骤4 复核确认
