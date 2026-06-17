# Canvas 七层叠加渲染机制深度解析

## 概述

手写字模拟器采用**七层Canvas叠加渲染架构**，通过分层渲染实现高度真实的手写效果、丰富的滤镜特效、以及灵活的装饰元素。每一层各司其职，按固定顺序叠加到最终画布上，形成完整的渲染输出。

核心渲染入口位于 [useHandwritingRender.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/hooks/useHandwritingRender.ts) 的 `renderToCanvas` 函数。

---

## 七层渲染管线总览

```
┌──────────────────────────────────────────────────────────────┐
│                     第 7 层：装饰层 (Decoration)              │
│              SVG矢量装饰图案 · 旋转/缩放/透明度               │
├──────────────────────────────────────────────────────────────┤
│                     第 6 层：批注层 (Annotation)              │
│        自由路径 · 形状 · 线条箭头 · 下划线 · 文字批注          │
├──────────────────────────────────────────────────────────────┤
│                      第 5 层：印章层 (Stamp)                  │
│           圆形/椭圆/方形印章 · 弧形文字 · multiply混合        │
├──────────────────────────────────────────────────────────────┤
│                    第 4 层：签名层 (Signature)                │
│                手写签名图片 · 可选纸张背景                    │
├──────────────────────────────────────────────────────────────┤
│                      第 3 层：滤镜层 (Filter)                 │
│  墨水晕染 · 铅笔素描 · 钢笔 · 毛笔 · 水彩 · 蜡笔 · 马克笔...  │
├──────────────────────────────────────────────────────────────┤
│                     第 2 层：文字层 (Text)                    │
│         手写抖动 · 墨水变化 · 光晕 · 干笔刷 · Markdown        │
├──────────────────────────────────────────────────────────────┤
│                     第 1 层：纸张层 (Paper)                   │
│      背景色 · 线条/网格/点阵 · 装订线 · 打孔 · 纸张纹理       │
└──────────────────────────────────────────────────────────────┘
```

---

## 渲染数据流图

```
  useWorkspaceStore (状态源)
         │
         ▼
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │  纸张配置    │────▶│  字体配置    │────▶│  抖动参数    │
  └─────────────┘     └─────────────┘     └─────────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             ▼
                    ┌─────────────────┐
                    │   RenderState   │ 渲染状态聚合
                    └─────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
  ┌───────────┐       ┌───────────┐       ┌───────────┐
  │ 签名数据  │       │ 印章数据  │       │ 批注数据  │ ...
  └───────────┘       └───────────┘       └───────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ▼
                   ┌──────────────────┐
                   │ renderToCanvas() │  核心渲染函数
                   └──────────────────┘
                            │
            ┌───────────────┴───────────────┐
            ▼                               ▼
   setTransform(DPR)              按顺序绘制七层
                                            │
                        ┌───────────────────┼───────────────────┐
                        ▼                   ▼                   ▼
                 ① drawPaper()       ② drawText()        ③ applyFilter()
                        │                   │                   │
                        └───────────────────┼───────────────────┘
                                            ▼
                        ┌───────────────────┼───────────────────┐
                        ▼                   ▼                   ▼
                ④ drawSignatures()  ⑤ drawStamps()    ⑥ drawAnnotations()
                        │                   │                   │
                        └───────────────────┼───────────────────┘
                                            ▼
                                   ⑦ drawDecorations()
                                            │
                                            ▼
                                   最终 Canvas 输出
```

---

## 各层详细解析

### 第 1 层：纸张层 (Paper Layer)

**渲染函数**：[drawPaper()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/hooks/useHandwritingRender.ts#L63-L157)

**职责**：构建最底层的纸张背景，提供书写载体的视觉基础。

**子模块**：

| 子模块 | 描述 | 关键实现 |
|--------|------|----------|
| 背景填充 | 纯色或渐变背景 | `ctx.fillRect()` + 线性渐变 (kraft纸) |
| 纸张纹理 | 横线 / 网格 / 点阵 / 空白 | switch 分支绘制不同类型 |
| 装订线 | 左侧红色装订线 + 打孔 | `showMargin` 控制，绘制三个打孔 |
| 噪点纹理 | 纸张质感噪点叠加 | 线性渐变模拟的微弱噪点 |

**支持的纸张类型**：
- `blank` - 空白纸
- `line` - 横线纸
- `grid` - 网格纸
- `dotted` - 点阵纸
- `kraft` - 牛皮纸（带渐变）

**关键参数**：
- `paperBgColor` - 背景色
- `paperLineColor` - 线条颜色
- `paperLineSpacing` - 行间距
- `showMargin` - 是否显示装订线
- `marginTop/Right/Bottom/Left` - 四边边距

---

### 第 2 层：文字层 (Text Layer)

**渲染函数**：
- 普通模式：[drawHandwrittenPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/hooks/useHandwritingRender.ts#L245-L398)
- Markdown模式：[drawMarkdownPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/markdown/markdownRenderer.ts#L314-L493)

**职责**：渲染手写文字内容，是整个渲染系统的核心层，实现手写质感的关键。

**文字渲染流水线**：

```
原始文本
   │
   ▼
┌──────────────┐
│  文本分行    │ breakTextIntoLines()
│  (测量宽度)  │
└──────────────┘
   │
   ▼
┌──────────────┐
│  分页处理    │ paginate()
└──────────────┘
   │
   ▼
┌──────────────┐
│  逐字抖动计算 │ CharDrawInfo
│  (13种抖动)  │
└──────────────┘
   │
   ▼
┌────────────────────────────┐
│  逐字绘制                    │
│  ├─ 位置/大小/旋转抖动       │
│  ├─ 墨水密度/颜色变化        │
│  ├─ 字距抖动                 │
│  ├─ 光晕效果 (halo)         │
│  └─ 干笔刷效果 (dryBrush)   │
└────────────────────────────┘
```

**抖动参数详解**（13种抖动维度）：

| 参数 | 作用 | 代码位置 |
|------|------|----------|
| `positionX` | 字符水平位置抖动 | `jitter.positionX` |
| `positionY` | 字符垂直位置抖动 | `jitter.positionY` |
| `size` | 字符大小抖动 | `jitter.size` |
| `rotation` | 字符旋转抖动 | `jitter.rotation` |
| `baseline` | 基线偏移抖动 | `jitter.baseline` |
| `inkDensity` | 墨水浓度变化 | `jitter.inkDensity` |
| `inkColor` | 墨水颜色深浅变化 | `jitter.inkColor` |
| `spacing` | 字距抖动 | `jitter.spacing` |
| `lineDrift` | 整行垂直漂移 | `jitter.lineDrift` |
| `lineTilt` | 整行倾斜 | `jitter.lineTilt` |
| `halo` | 墨韵光晕效果 | `jitter.halo` |
| `dryBrush` | 干笔刷飞白效果 | `jitter.dryBrush` |
| `amount` | 全局抖动强度系数 | `jitter.amount` |

**Markdown 渲染扩展**：
- 支持标题、加粗、斜体、删除线、代码、引用、列表等
- 不同块级元素有独立的样式配置
- 支持代码块背景、引用块左侧边框等块级装饰

---

### 第 3 层：滤镜层 (Filter Layer)

**渲染函数**：[applyFilter()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/filterEffects.ts#L4-L49)

**职责**：对前两层（纸张+文字）进行像素级后处理，模拟不同书写工具的材质效果。

**技术特点**：
- 基于 `getImageData` / `putImageData` 的像素级操作
- 直接操作 `Uint8ClampedArray` 像素数组
- 所有滤镜都是纯算法实现，无需外部资源

**支持的滤镜类型**：

| 滤镜 | 效果描述 | 核心算法 |
|------|----------|----------|
| `inkBleed` | 墨水晕染 | 像素膨胀扩散 + 随机抖动 |
| `pencilSketch` | 铅笔素描 | Sobel边缘检测 + 灰度噪点 |
| `penStroke` | 钢笔笔触 | 压力变化 + 边缘飞墨 |
| `brushStroke` | 毛笔笔触 | 笔刷半径扩散 + 干湿变化 |
| `watercolor` | 水彩效果 | 三色通道独立扩散 + 湿润感 |
| `carbonCopy` | 复写纸 | 蓝色偏移 + 褪色效果 |
| `fountainPen` | 钢笔尖 | 起始流量变化 + 羽毛状边缘 |
| `crayon` | 蜡笔 | 蜡质纹理 + 纸张颗粒感 |
| `marker` | 马克笔 | 曼哈顿距离膨胀 + 色彩增强 |

**滤镜执行流程**：

```
  画布已有内容
      │
      ▼
  getImageData() ──▶ Uint8ClampedArray
      │
      ▼
  按滤镜类型处理像素:
  ┌──────────────────────────┐
  │  遍历每个像素            │
  │  ├─ 判断是否为墨水像素   │
  │  ├─ 应用对应滤镜算法     │
  │  └─ 更新像素值           │
  └──────────────────────────┘
      │
      ▼
  putImageData() ──▶ 写回画布
      │
      ▼
  重置 DPR 变换 (滤镜会破坏变换矩阵)
```

> **注意**：滤镜层在签名、印章、批注、装饰**之前**执行，意味着滤镜只作用于纸张和文字，不影响后续叠加层。这是设计上的重要决策。

---

### 第 4 层：签名层 (Signature Layer)

**渲染函数**：[drawSignaturesForPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/signatureRenderer.ts#L167-L187)

**职责**：在页面上放置手写签名图片，支持多个签名、不同位置和缩放。

**签名数据结构**：
- `id` - 签名唯一标识
- `dataUrl` - 签名图片数据（PNG，透明背景）
- `width` / `height` - 签名原始尺寸
- `paperBgColor` / `paperType` - 可选的签名背景纸配置
- `bgOpacity` - 背景透明度

**放置数据** (`signaturePlacements`)：
- `signatureId` - 关联的签名ID
- `pageIndex` - 所在页码
- `x` / `y` - 位置坐标（中心点）
- `scale` - 缩放比例

**渲染流程**：

```
  签名放置数据
      │
      ▼
  筛选当前页的放置
      │
      ▼
  遍历每个放置:
  ┌──────────────────────────┐
  │  1. 查找签名配置和图片    │
  │  2. 计算绘制尺寸(缩放)   │
  │  3. 可选: 绘制纸张背景   │
  │  4. drawImage() 绘制签名 │
  └──────────────────────────┘
```

---

### 第 5 层：印章层 (Stamp Layer)

**渲染函数**：[drawStampsForPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/stampRenderer.ts#L416-L436)

**职责**：渲染各类印章效果，包括公章、私章等，具有真实的印章质感。

**印章形状**：
- `circle` - 圆形印章
- `ellipse` - 椭圆形印章
- `square` - 方形印章（圆角）

**印章组成元素**：

```
            ┌──────────────┐
            │   顶部弧形文字 │  topText
            └──────────────┘
          ╱                  ╲
         ╱   ┌──────────┐     ╲
        │    │  ★ 五角星 │     │  showStar
        │    │ 中心文字  │     │  centerText
         ╲   └──────────┘    ╱
          ╲                  ╱
            └──────────────┘
            │   底部弧形文字 │  bottomText
            └──────────────┘
```

**印章特效**：
- `borderStyle` - 边框样式（none/solid/dashed/double）
- `borderWidth` - 边框宽度
- `innerPadding` - 内边距
- 印章噪点 - 模拟印章盖印不均匀的质感
- `multiply` 混合模式 - 模拟红色印章叠加效果

**渲染流程**：

```
  印章配置 (StampConfig)
      │
      ▼
  renderStampToCanvas()
      │
      ├─ 1. 设置画布尺寸和DPR
      ├─ 2. 绘制边框
      ├─ 3. 绘制顶部弧形文字
      ├─ 4. 绘制底部弧形文字
      ├─ 5. 绘制中心五角星 (可选)
      ├─ 6. 绘制中心文字
      └─ 7. 应用印章噪点
      │
      ▼
  生成 dataUrl 缓存
      │
      ▼
  放置到主画布 (multiply 混合)
```

---

### 第 6 层：批注层 (Annotation Layer)

**渲染函数**：[drawAnnotationsForPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/annotationRenderer.ts#L162-L169)

**职责**：提供各种批注标记工具，用于在文档上进行标注和评论。

**批注类型**：

| 类型 | 描述 | 实现函数 |
|------|------|----------|
| `path` | 自由画笔路径 | `drawPath()` |
| `highlight` | 高亮标记 | `drawPath()` (同路径) |
| `rect` | 矩形框 | `drawShape()` |
| `circle` | 圆形/椭圆 | `drawShape()` |
| `line` | 直线 | `drawLine()` |
| `arrow` | 箭头 | `drawLine()` (带箭头) |
| `underline` | 下划线 | `drawUnderline()` |
| `wavy` | 波浪线 | `drawUnderline()` (波浪) |
| `text` | 文字批注 | `drawText()` |

**批注样式属性**：
- `color` - 颜色
- `strokeWidth` - 线条宽度
- `opacity` - 透明度
- `fillColor` - 填充色（形状）
- `fontSize` / `fontFamily` - 文字样式（文字批注）

---

### 第 7 层：装饰层 (Decoration Layer)

**渲染函数**：[drawDecorationsForPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/decorationRenderer.ts#L61-L106)

**职责**：添加各种装饰性的SVG矢量图案，如花草、边框、角落装饰等，增强视觉效果。

**装饰元素特点**：
- 基于 SVG 矢量图形，缩放不失真
- 支持旋转、缩放、透明度调整
- 预设多种装饰分类（角落、边框、花草等）

**放置属性** (`decorationPlacements`)：
- `decorationId` - 装饰图案ID
- `pageIndex` - 所在页码
- `x` / `y` - 位置坐标
- `width` / `height` - 尺寸
- `rotation` - 旋转角度
- `opacity` - 透明度

**渲染降级策略**：

```
  尝试渲染装饰
      │
      ├─▶ 优先使用预加载缓存图片 ✓
      │
      ├─▶ 失败则尝试即时加载 SVG
      │
      └─▶ 仍失败则绘制占位矩形
```

---

## 完整渲染时序与数据流

### 单次渲染完整流程

```
renderToCanvas(canvas, pageIdx)
    │
    ├─ 1. 初始化画布
    │   ├─ 设置 canvas.width/height (DPR)
    │   ├─ 设置 canvas.style.width/height (CSS)
    │   └─ ctx.setTransform(DPR, 0, 0, DPR, 0, 0)
    │
    ├─ 2. 第1层: 绘制纸张
    │   └─ drawPaper(ctx, renderState)
    │
    ├─ 3. 第2层: 绘制文字
    │   ├─ [Markdown模式] drawMarkdownPage()
    │   └─ [普通模式] drawHandwrittenPage()
    │
    ├─ 4. 第3层: 应用滤镜
    │   ├─ if (activeFilter !== 'none')
    │   ├─ applyFilter(ctx, filter, intensity, inkColor)
    │   └─ 重置 DPR 变换矩阵
    │
    ├─ 5. 第4层: 绘制签名
    │   └─ drawSignatures(ctx, currentPageIdx)
    │
    ├─ 6. 第5层: 绘制印章
    │   └─ drawStamps(ctx, currentPageIdx)
    │
    ├─ 7. 第6层: 绘制批注
    │   └─ drawAnnotations(ctx, currentPageIdx)
    │
    └─ 8. 第7层: 绘制装饰
        └─ drawDecorations(ctx, currentPageIdx)
```

### 数据流向图

```
┌─────────────────────────────────────────────────────────────────┐
│                     useWorkspaceStore (Zustand)                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │ 纸张配置 │ │ 字体配置 │ │ 文本内容 │ │ 签名印章 │ │ 批注装饰 │  │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                    ┌────────────────┐
                    │  useMemo 聚合  │
                    │  RenderState   │
                    └────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │ 图片预加载│      │ 文本排版  │      │ 页面计算  │
    │(签名/印章/│      │(分行/分页)│      │(totalPages)│
    │ 装饰)    │      └──────────┘      └──────────┘
    └──────────┘           │                   │
          │                └─────────┬─────────┘
          │                          ▼
          │                ┌────────────────┐
          └───────────────▶│ renderToCanvas │◀───────────┐
                           └────────────────┘            │
                                    │                    │
                                    ▼                    │
                           七层叠加渲染输出               │
                                    │                    │
                                    ▼                    │
                           ┌────────────────┐            │
                           │  最终 Canvas   │            │
                           └────────────────┘            │
                                    │                    │
                    ┌───────────────┴───────────────┐    │
                    ▼                               ▼    │
              实时预览显示                      导出PNG/PDF
                                                    │
                                          renderAllCanvases()
```

---

## 关键技术设计

### 1. DPR 高清晰度渲染

```
物理像素 = CSS像素 × DPR (DPR=2)
```

- 画布实际尺寸 = 794×1123 × 2 = 1588×2246 像素
- CSS显示尺寸 = 794×1123 像素
- 通过 `ctx.setTransform(DPR, 0, 0, DPR, 0, 0)` 统一坐标系

### 2. 种子随机数 (Seeded Random)

使用 `seededRandom()` 实现确定性抖动：
- 每页使用独立种子 (`pageIndex * 100001 + 17`)
- 每行、每字符都有独立的子种子
- 保证同一页面每次渲染效果完全一致

### 3. 滤镜层的特殊位置

滤镜层在**文字层之后、签名层之前**，这个设计很关键：
- 滤镜只作用于"手写内容"（纸张+文字）
- 签名、印章、批注等"叠加元素"不受滤镜影响
- 模拟真实场景：先写字，再加滤镜，最后盖章签名

### 4. 状态驱动的响应式渲染

基于 Zustand 状态管理 + React hooks：
- 所有配置变化自动触发重渲染
- `useMemo` 优化 RenderState 计算
- `useEffect` 监听依赖变化自动调用 `renderToCanvas`

### 5. 离屏渲染与批量导出

`renderAllCanvases()` 函数支持批量导出：
- 创建离屏 canvas 元素
- 逐页渲染所有七层
- 返回 canvas 数组供导出使用
- 与实时预览共享同一套渲染逻辑

---

## 相关核心文件索引

| 文件 | 职责 |
|------|------|
| [useHandwritingRender.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/hooks/useHandwritingRender.ts) | 核心渲染Hook，七层调度 |
| [canvasUtils.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/canvasUtils.ts) | Canvas工具函数（随机数、颜色、纸张绘制） |
| [filterEffects.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/filterEffects.ts) | 9种像素级滤镜效果 |
| [signatureRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/signatureRenderer.ts) | 签名渲染器 |
| [stampRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/stampRenderer.ts) | 印章渲染器 |
| [annotationRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/annotationRenderer.ts) | 批注渲染器 |
| [decorationRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/decorationRenderer.ts) | 装饰渲染器 |
| [markdownRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/utils/markdown/markdownRenderer.ts) | Markdown排版与渲染 |
| [useExport.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/hooks/useExport.ts) | 导出功能（PNG/PDF） |
| [HandwritingPreview.tsx](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Earth/src/components/preview/HandwritingPreview.tsx) | 预览组件 |

---

## 总结

Canvas七层叠加渲染架构是一个精心设计的分层系统，每一层都有明确的职责边界：

1. **纸张层** - 提供基础载体
2. **文字层** - 核心内容与手写质感
3. **滤镜层** - 像素级后处理增强
4. **签名层** - 手写签名叠加
5. **印章层** - 公章印章效果
6. **批注层** - 标注工具
7. **装饰层** - 视觉装饰元素

这种分层设计的优势：
- ✅ **职责清晰** - 每层独立，易于维护和扩展
- ✅ **顺序可控** - 叠加顺序决定视觉层级关系
- ✅ **性能优化** - 可针对单层独立优化
- ✅ **灵活组合** - 可根据需求开启/关闭特定层
- ✅ **易于测试** - 每层可单独测试验证
