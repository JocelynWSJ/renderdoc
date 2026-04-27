# renderdoc-analysis

## 简介

**RenderDoc 帧分析 Skill**，通过 RenderDoc MCP 工具对 `.rdc` 捕获文件进行系统化分析，并生成带可视化图例的专业 HTML/PDF 报告。

支持两种核心模式：
- **模式 A：单帧分析** — 对单个 `.rdc` 文件进行全面的渲染管线分析
- **模式 B：双帧对比** — 对两个 `.rdc` 文件进行差异对比分析

---

## 触发条件

当用户提出以下请求时使用本 Skill：

- "分析这个 rdc 文件"
- "打开并分析这一帧"
- "对比两个 rdc 文件"
- "生成帧分析报告"
- "帮我看一下这个 capture"

---

## 分析工作流

### 第一步：加载 Capture

```
1. 调用 renderdoc_open_capture(capture_path) 加载文件
2. 确认返回 success:true 及 api 类型
3. 若为对比模式，依次加载两个文件分别采集数据（切换时需重新 open）
```

### 第二步：并行采集帧基础数据

同时调用以下工具（可并行）：

| 工具 | 获取内容 |
|------|---------|
| `renderdoc_get_frame_summary` | 总 Action 数、DrawCall 数、Pass 统计、资源计数 |
| `renderdoc_get_draw_calls` | 完整 Pass 树（include_children=true） |

**必须提取的关键指标：**
- 总 Action 数、DrawCall 数、Dispatch 数、Present 数
- Marker 数（Depth Pass 数 + Colour Pass 数）
- 纹理资源数、Buffer 资源数
- 顶级 Pass 名称列表及各 Pass child_count
- Command Buffer 数（从 draw calls 树中统计 vkBeginCommandBuffer 出现次数）

### 第三步：识别渲染管线结构

分析 Pass 树，识别以下阶段（按 Pass 名称 + child_count 判断）：

| 阶段类型 | 识别特征 | 颜色标记 |
|---------|---------|---------|
| **Z-Prepass** | `Depth-only Pass`，PS 无纹理绑定 | 紫色 `#6366f1` |
| **主场景前向渲染** | `Colour Pass (1 Targets + Depth)`，child_count ≥ 10 | 蓝色 `#0ea5e9` |
| **透明/补充几何** | `Colour Pass (1 Targets + Depth)`，C=Load，D=Load | 蓝色 `#0ea5e9` |
| **后处理链** | 连续多个 `Colour Pass (1 Targets)`，child_count=3，单 DrawCall | 绿色 `#10b981` |
| **UI / Overlay** | `Colour Pass`，DS=Clear，多个小 DrawCall（6 indices）| 橙色 `#f59e0b` |
| **最终合成** | 末尾 Pass，`vkCmdDraw 4 verts`，无 CB | 红色 `#ef4444` |

**判断双视角渲染：** 若帧内出现两段结构完全对称的 Pass 序列（相同 DrawCall 数 + 索引数），则标注为双视角渲染（VR/双摄像机）。

### 第四步：并行采集代表性 Pass 的管线状态

针对以下代表性 event_id 调用 `renderdoc_get_pipeline_state`（可并行）：

| 目标 | 选取策略 |
|------|---------|
| Z-Prepass 第一个 DrawCall | Depth-only Pass #1 的第一个 Drawcall event_id |
| 主场景第一个 DrawCall | Colour Pass #1 (Targets+Depth) 的第一个 Drawcall |
| 后处理第一个 Pass | Colour Pass #3（第一个单 DrawCall 全屏 Pass）|
| UI Pass 中段 DrawCall | UI Pass 中索引数最大的 DrawCall |
| 最终合成 DrawCall | 最后一个 vkCmdDraw(4 verts) |

**必须从 pipeline_state 中提取：**
- 各阶段 Shader resource_id + entry_point
- PS 纹理绑定（slot、resource_id、尺寸、格式）→ 推断用途
- PS/VS Constant Buffer 列表（slot、大小）→ 推断语义
- 颜色/深度输出 resource_id → 追踪 RT 链路

**纹理用途推断规则：**

| 尺寸 | 格式 | 推断用途 |
|------|------|---------|
| 256×256, 512×512 | R8G8B8A8_SRGB | Albedo 漫反射 |
| 256×256 | R8G8B8A8_UNORM | 法线贴图 |
| 2048×2048 | D16 / D24 | Shadow Map |
| 128×128 | R8G8B8A8_UNORM | 细节贴图/LUT |
| 渲染分辨率 | R16G16B16A16_FLOAT | HDR 颜色缓冲 |
| 渲染分辨率 | R16_FLOAT | 亮度/曝光缓冲 |
| 2048×2048 | R8G8B8A8_UNORM (UI Pass) | UI Sprite Atlas |
| 2048×2048 | R8_UNORM (UI Pass) | SDF 字体图集 |
| 输出分辨率 | R8G8B8A8_UNORM (最终) | 已合成帧缓冲 |

### 第五步（可选）：Shader 反汇编分析

若用户要求深入分析 Shader，对关键 event_id 调用 `renderdoc_get_shader_info`：

从 SPIR-V 反汇编中识别：
- Shader 编译器来源（ANGLE → OpenGL ES，DXC → HLSL，glslang → GLSL）
- 光照模型（Phong / Blinn-Phong / PBR GGX / 自定义）
- 阴影技术（PCF / PCSS / VSM → 从 `ImageSampleDrefExplicitLod` 判断）
- 是否含骨骼动画（大型 CB 含 float4 数组 ≥ 64 项 → 推断为骨骼矩阵）
- 动态分支（光源循环 `while(true)` → 统计最大光源数）
- Specialization Constants → 调试模式支持

### 第六步：生成分析报告

---

## 输出报告规范

### 文件产物

| 文件 | 说明 |
|------|------|
| `{name}_analysis.html` | 带图例的完整 HTML 报告（内联所有资源）|
| `{name}_analysis.pdf` | Chrome 无头打印 PDF |

输出路径默认为 `.rdc` 文件所在目录。若用户未指定报告名称，使用 `.rdc` 文件名作为前缀。

### PDF 生成命令

```powershell
# 下载 Chart.js 并内联（避免 headless 无网络问题）
python -c "import urllib.request,pathlib; pathlib.Path('chartjs.min.js').write_bytes(urllib.request.urlopen('https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js',timeout=15).read())"

# 内联后打印 PDF
& "C:\Program Files\Google\Chrome\Application\chrome.exe" `
  --headless --disable-gpu `
  --run-all-compositor-stages-before-draw `
  --no-pdf-header-footer `
  --print-to-pdf="output.pdf" `
  "input.html"
```

---

## 视觉设计规范

### 调色板

```
蓝色（3.rdc / 主要数据）:   #3b82f6  (bg: #eff6ff)
橙色（4.rdc / 对比数据）:   #f59e0b  (bg: #fffbeb)
绿色（正向变化 / 优化）:    #10b981  (bg: #d1fae5)
红色（负向变化 / 警告）:    #ef4444  (bg: #fee2e2)
紫色（深度 Pass）:          #6366f1
青色（颜色 Pass）:          #0ea5e9
背景色:                     #f8fafc
卡片色:                     #ffffff
边框色:                     #e2e8f0
```

### 封面设计

```css
/* 深色渐变背景 */
background: linear-gradient(135deg, #0f172a 0%, #1e3a5f 50%, #0f172a 100%);
/* 右上角光晕装饰 */
radial-gradient(circle, rgba(59,130,246,.25) 0%, transparent 70%)
```

封面必须包含：
- 徽章标签（如 `RENDERDOC · FRAME ANALYSIS REPORT`）
- 标题（大字体，蓝色高亮关键词）
- 副标题（游戏名 · API · 平台）
- 元数据行（分析日期、GPU API、分辨率）
- 对比模式：左右两个 Pill 卡片展示文件名 + VS 分隔符

### 章节结构（单帧分析）

每章节必须有编号 Icon（渐变背景圆角方块）：

```
01  帧总览（4格统计卡 + 柱状图 + DrawCall 分类表）
02  渲染管线结构图（色块宽度 = DC 数，双视角对称标注）
03  Z-Prepass 几何分析（横向条形图 + 表格）
04  主场景前向渲染（纹理绑定表 + CB 布局）
05  后处理链（Pass 序列图 + 推断功能表）
06  UI Overlay（Atlas 信息 + DrawCall 碎片化分析）
07  Shader 分析（SPIR-V 关键特性提取）
08  关键技术总结（技术点汇总表）
09  优化建议（优先级排序表）
```

### 章节结构（双帧对比）

```
01  帧总览对比（4格 Stat 卡 + 对比柱状图）
02  渲染 Pass 结构对比（左右并排管线图，差异项橙色边框高亮）
03  Z-Prepass 几何体对比（横向条形排名图）
04  透明/补充 Pass 对比（DrawCall 分布直方图 + 深度策略对比）
05  Command Buffer 拓扑对比（左右对比表，差异行高亮 #fffbeb）
06  关键差异汇总（影响等级标签：高/中/低 + 颜色编码）
07  优化建议（针对差异的具体优化项）
```

### 图表类型选择

| 数据 | 图表类型 | 颜色 |
|------|---------|------|
| 总览指标对比 | 分组柱状图 (Chart.js bar) | 蓝 vs 橙 |
| 索引总量对比 | 分组柱状图 | 蓝 vs 橙 |
| DrawCall 分布 | 直方图（按索引区间分桶） | 单色（各自颜色） |
| Pass 结构 | 横向色块图（CSS div，宽度∝DC数）| 按阶段类型上色 |
| 几何复杂度排名 | 横向渐变条形图 | 蓝/橙渐变 |

### 差异标签规范（对比报告）

```html
<!-- 增加（正方向关注）-->
<span class="badge-up">+N ▲</span>  /* bg:#fef3c7, color:#92400e */

<!-- 减少（通常是优化）-->
<span class="badge-dn">-N ▼</span>  /* bg:#dcfce7, color:#14532d */

<!-- 不变 -->
<span class="badge-eq">不变</span>   /* bg:#f1f5f9, color:#475569 */

<!-- 高影响警告 -->
<span class="badge-warn">高 ▲</span> /* bg:#fee2e2, color:#991b1b */

<!-- 正向优化 -->
<span class="badge-ok">优化</span>   /* bg:#d1fae5, color:#065f46 */
```

### Callout 提示框

```html
<!-- 警告 -->
<div class="callout callout-warn">⚠️ 标题 + 说明文字</div>
/* border: #fcd34d, background: #fffbeb */

<!-- 信息 -->
<div class="callout callout-info">ℹ️ 标题 + 说明文字</div>
/* border: #93c5fd, background: #eff6ff */

<!-- 成功 -->
<div class="callout callout-ok">✅ 标题 + 说明文字</div>
/* border: #6ee7b7, background: #f0fdf4 */
```

### 管线结构图实现

```javascript
// Pass 色块宽度 = DrawCall 数 / MAX_DRAWS * 100%
// MAX_DRAWS = 帧内单 Pass 最大 DrawCall 数（通常取 45）
const pct = Math.max(Math.round(draws / MAX_DRAWS * 100), 6);  // 最小 6%

// 差异项：橙色描边
const hl = isDiff ? 'border:2px solid #f59e0b;' : '';
```

### 字体与间距

```css
font-family: "Microsoft YaHei", "Segoe UI", sans-serif;
body: font-size 14px, line-height 1.6
h1: 42px (封面) / 18px (章节标题)
h3: 1.2em, color #444
table th: 11px uppercase, letter-spacing .06em, background #f1f5f9
card padding: 20px
section margin-bottom: 48px
```

---

## 输出内容规范

### 必须输出的分析项

**单帧必须包含：**
1. 基本信息表（API、Shader 编译器、分辨率、CB 数）
2. 渲染管线总体结构图（文字 ASCII 或 HTML 色块图）
3. 各阶段代表性 DrawCall 详情（Pass 名 + event_id + 纹理绑定表）
4. 技术总结表（技术点 + 实现方案 + 备注）
5. 优化建议（至少 3 条，含优先级）

**双帧对比必须包含：**
1. 量化对比表（≥ 8 个指标，含变化量和百分比）
2. 逐 Pass 结构对比（标注新增/删除/修改）
3. 关键差异汇总（影响等级分类）
4. 针对差异的优化建议

### 优化建议格式

```markdown
| # | 优化项（具体可操作描述）| 针对帧 | 预期收益 |
```

优先级排序：DrawCall 合批 → 深度策略 → CB 合并 → Shader 复杂度 → LOD/剔除

### 不确定内容的处理

- 无法直接判断功能时，用「推断为 XXX」+ 依据说明
- 对 Shader 功能不确定时，描述观察到的关键操作（如 `ImageSampleDrefExplicitLod` → PCF 阴影）
- 对 CB 用途不确定时，通过字段数量和大小推断（如 115×float4 → 骨骼矩阵）

---

## 注意事项

1. **MCP 超时处理**：若工具调用超时，提示用户确认 RenderDoc 桌面程序已启动
2. **大文件处理**：draw_calls 返回数据较大时，提取关键 Pass 结构即可，无需逐 DrawCall 分析
3. **并行调用**：所有无数据依赖的工具调用必须并行执行，加快采集速度
4. **中文路径**：Windows 路径含中文时，Python 脚本需确保 `encoding='utf-8'`
5. **Chart.js 内联**：生成 HTML 时，必须将 Chart.js 下载后内联，确保 headless PDF 中图表正常渲染
6. **临时文件清理**：报告生成完毕后，删除中间临时脚本和 JS 库文件
