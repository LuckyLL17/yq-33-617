# Canvas 七层叠加渲染管线数据流图

## 渲染管线总览

本项目采用 **单 Canvas 七层顺序叠加** 的渲染架构，所有图层绘制在同一个 `CanvasRenderingContext2D` 上，按从底到顶的顺序依次绘制，后绘制的图层自然覆盖先绘制的图层，形成完整的纸面效果。

### 七层渲染顺序

```
┌─────────────────────────────────────────────────────────┐
│                    最终合成画面                            │
├─────────────────────────────────────────────────────────┤
│  Layer 7  ▲  装饰层 (Decoration)                         │
│  Layer 6  │  批注层 (Annotation)                          │
│  Layer 5  │  印章层 (Stamp)                               │
│  Layer 4  │  签名层 (Signature)                           │
│  Layer 3  │  滤镜层 (Filter)                              │
│  Layer 2  │  手写文本层 (Handwriting / Markdown)          │
│  Layer 1  ▼  纸张背景层 (Paper)                           │
└─────────────────────────────────────────────────────────┘
```

---

## 完整数据流图

```
                              ┌──────────────────┐
                              │  useWorkspaceStore│
                              │   (Zustand 全局)  │
                              └────────┬─────────┘
                                       │
           ┌───────────────────────────┼───────────────────────────┐
           │                           │                           │
           ▼                           ▼                           ▼
  ┌────────────────┐       ┌──────────────────┐       ┌──────────────────┐
  │  RenderState   │       │  Placement 数据   │       │  Image 缓存      │
  │  (useMemo)     │       │  (签名/印章/装饰)  │       │  (异步加载)       │
  └───────┬────────┘       └────────┬─────────┘       └────────┬─────────┘
          │                         │                          │
          ▼                         ▼                          ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │                     renderToCanvas(canvas, pageIdx)               │
  │                                                                   │
  │  ① canvas.width = PAGE_WIDTH × DPR                               │
  │    canvas.height = PAGE_HEIGHT × DPR                              │
  │    ctx.setTransform(DPR, 0, 0, DPR, 0, 0)                       │
  │                                                                   │
  │  ┌─────────────────────────────────────────────────────────────┐  │
  │  │ Layer 1: drawPaper(ctx, renderState)                        │  │
  │  │   ├─ 背景色填充 / 牛皮纸渐变                                 │  │
  │  │   ├─ 纸张线条 (横线/网格/点阵)                               │  │
  │  │   ├─ 装订线 + 装订孔                                        │  │
  │  │   └─ 纸面噪声纹理叠加                                        │  │
  │  └─────────────────────────────────────────────────────────────┘  │
  │                          ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐  │
  │  │ Layer 2: 文本渲染 (二选一)                                   │  │
  │  │                                                              │  │
  │  │  [Markdown 模式]                                             │  │
  │  │  ├─ parseMarkdown(text) → MarkdownBlock[]                   │  │
  │  │  ├─ layoutMarkdown() → RenderLine[]                         │  │
  │  │  ├─ paginateMarkdownLines() → RenderLine[][]                │  │
  │  │  └─ drawMarkdownPage(ctx, pageLines, pageIdx, options)      │  │
  │  │      ├─ 绘制代码块/引用块背景                                │  │
  │  │      ├─ 绘制引用块左边框                                     │  │
  │  │      └─ 逐字符绘制 (含 jitter 偏移/旋转/墨色变化)            │  │
  │  │                                                              │  │
  │  │  [纯文本模式]                                                │  │
  │  │  ├─ breakTextIntoLines() → {text, isParagraphEnd}[]        │  │
  │  │  ├─ paginate() → pages[][]                                  │  │
  │  │  └─ drawHandwrittenPage(ctx, pageLines, rs, pageIdx)        │  │
  │  │      ├─ 逐行计算 lineDrift / lineTilt                       │  │
  │  │      ├─ 逐字符计算 CharDrawInfo (12维 jitter)               │  │
  │  │      ├─ 绘制主文字 fillText                                  │  │
  │  │      ├─ 绘制光晕 (halo) 效果                                 │  │
  │  │      └─ 绘制干笔 (dryBrush) 效果                             │  │
  │  └─────────────────────────────────────────────────────────────┘  │
  │                          ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐  │
  │  │ Layer 3: applyFilter(ctx, activeFilter, intensity, inkColor)│  │
  │  │   ├─ getImageData() 获取全部像素                             │  │
  │  │   ├─ 像素级变换 (9种滤镜算法)                                │  │
  │  │   │   ├─ inkBleed     墨迹晕染                               │  │
  │  │   │   ├─ pencilSketch 铅笔素描 (Sobel 边缘检测)             │  │
  │  │   │   ├─ penStroke    钢笔笔触                               │  │
  │  │   │   ├─ brushStroke  毛笔笔触                               │  │
  │  │   │   ├─ watercolor   水彩渲染                               │  │
  │  │   │   ├─ carbonCopy   复写纸                                 │  │
  │  │   │   ├─ fountainPen  钢笔墨痕                               │  │
  │  │   │   ├─ crayon       蜡笔涂鸦                               │  │
  │  │   │   └─ marker       马克笔                                 │  │
  │  │   ├─ putImageData() 写回像素                                 │  │
  │  │   └─ ctx.setTransform(DPR, ...) 重置变换矩阵                 │  │
  │  └─────────────────────────────────────────────────────────────┘  │
  │                          ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐  │
  │  │ Layer 4: drawSignatures(ctx, currentPageIdx)                │  │
  │  │   ├─ 过滤当前页的 signaturePlacements                       │  │
  │  │   ├─ 查找对应的 Signature 数据和签名图片                     │  │
  │  │   ├─ drawSignatureOnCanvas()                                 │  │
  │  │   │   ├─ [可选] drawSignatureBackground() 绘制签名背景纸     │  │
  │  │   │   └─ ctx.drawImage(img, x, y, w, h)                    │  │
  │  │   └─ 签名图片来源: loadSignatureImages() → HTMLImageElement │  │
  │  └─────────────────────────────────────────────────────────────┘  │
  │                          ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐  │
  │  │ Layer 5: drawStamps(ctx, currentPageIdx)                    │  │
  │  │   ├─ 过滤当前页的 stampPlacements                           │  │
  │  │   ├─ 查找对应的 Stamp 数据和印章图片                         │  │
  │  │   ├─ drawStampOnCanvas()                                     │  │
  │  │   │   ├─ ctx.globalAlpha = 0.92                             │  │
  │  │   │   ├─ ctx.globalCompositeOperation = 'multiply'          │  │
  │  │   │   └─ ctx.drawImage(img, x, y, w, h)                    │  │
  │  │   └─ 印章图片来源: loadStampImages() → HTMLImageElement    │  │
  │  └─────────────────────────────────────────────────────────────┘  │
  │                          ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐  │
  │  │ Layer 6: drawAnnotations(ctx, currentPageIdx)               │  │
  │  │   ├─ 过滤当前页的 annotations                               │  │
  │  │   ├─ 逐条绘制，按类型分发:                                   │  │
  │  │   │   ├─ path/highlight → drawPath()   自由路径/荧光笔      │  │
  │  │   │   ├─ circle/rect    → drawShape()  圆/矩形              │  │
  │  │   │   ├─ line/arrow     → drawLine()   直线/箭头            │  │
  │  │   │   ├─ underline/wavy → drawUnderline() 下划线/波浪线     │  │
  │  │   │   └─ text           → drawText()   文字批注             │  │
  │  │   └─ 每条批注独立应用 AnnotationStyle (颜色/线宽/透明度)     │  │
  │  └─────────────────────────────────────────────────────────────┘  │
  │                          ▼                                        │
  │  ┌─────────────────────────────────────────────────────────────┐  │
  │  │ Layer 7: drawDecorations(ctx, currentPageIdx)               │  │
  │  │   ├─ 过滤当前页的 decorationPlacements                      │  │
  │  │   ├─ 查找对应的 DecorationPreset                             │  │
  │  │   ├─ 逐个绘制:                                              │  │
  │  │   │   ├─ ctx.globalAlpha = placement.opacity                │  │
  │  │   │   ├─ ctx.translate + ctx.rotate (位置/旋转)             │  │
  │  │   │   ├─ [优先] decorationImages 缓存绘制                   │  │
  │  │   │   ├─ [备选] getOrLoadImage() 重试加载                   │  │
  │  │   │   └─ [兜底] 绘制占位矩形                                │  │
  │  │   └─ 装饰图片来源: loadDecorationImages() → SVG→Image缓存  │  │
  │  └─────────────────────────────────────────────────────────────┘  │
  │                                                                   │
  └───────────────────────────────────────────────────────────────────┘
                          │
                          ▼
               ┌─────────────────────┐
               │  Canvas 最终画面输出  │
               │  (794×1123 @2x DPR) │
               └─────────────────────┘
```

---

## 各层详细解析

### Layer 1: 纸张背景层 — `drawPaper()`

**源码位置**: [useHandwritingRender.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Mercury/src/hooks/useHandwritingRender.ts#L63-L157)

**数据输入**:
| 数据 | 来源 | 说明 |
|------|------|------|
| `paperType` | `resolvePaperType(selectedPaperId)` | blank / line / grid / kraft / dotted |
| `paperBgColor` | Store → `paperBgColor` | 纸张背景色 |
| `paperLineColor` | Store → `paperLineColor` | 线条颜色 |
| `paperLineSpacing` | Store → `paperLineSpacing` | 线条间距 |
| `showMargin` | Store → `showBindingLine` | 是否显示装订线 |
| `marginTop/Right/Bottom/Left` | Store | 页边距 |

**渲染子步骤**:
1. **背景填充**: 纯色填充或牛皮纸渐变 (`createLinearGradient`)
2. **纸张线条**: 根据 `paperType` 绘制横线/网格/点阵
3. **装订线**: 红色竖线 + 3个装订孔 (20%/50%/80% 高度处)
4. **纸面噪声**: 叠加半透明棕色渐变模拟纸张老化

---

### Layer 2: 手写文本层 — `drawHandwrittenPage()` / `drawMarkdownPage()`

**源码位置**:
- 纯文本: [useHandwritingRender.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Mercury/src/hooks/useHandwritingRender.ts#L245-L398)
- Markdown: [markdownRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Mercury/src/utils/markdown/markdownRenderer.ts#L314-L493)

**数据输入**:
| 数据 | 来源 | 说明 |
|------|------|------|
| `rawText` | Store | 用户输入的原始文本 |
| `fontFamily` | `buildFontFamily(selectedFontId)` | 字体族 |
| `fontSize` | Store | 字号 |
| `inkColor` | Store | 墨色 |
| `jitter` (12维) | Store | 抖动参数组 |
| `letterSpacing` | Store | 字间距 |
| `lineHeight` | Store | 行高 |
| `paragraphSpacing` | Store | 段间距 |

**文本处理管线**:
```
rawText
  │
  ├─ [纯文本模式]
  │   ├─ breakTextIntoLines()   逐字符测量宽度，按 maxWidth 换行
  │   ├─ paginate()             按可用高度分页
  │   └─ drawHandwrittenPage()  逐字符绘制
  │
  └─ [Markdown 模式]
      ├─ parseMarkdown()        解析为 MarkdownBlock[]
      ├─ layoutMarkdown()       布局计算 → RenderLine[]
      ├─ paginateMarkdownLines() 分页
      └─ drawMarkdownPage()     逐字符绘制
```

**Jitter 12维参数体系** (`CharDrawInfo`):
| 参数 | 作用 | 计算公式 |
|------|------|----------|
| `positionX` | 水平偏移 | `(rand - 0.5) × 2.5 × amount × positionX × fontSize × 0.2` |
| `positionY` | 垂直偏移 | `(rand - 0.5) × 2.0 × amount × positionY × fontSize × 0.15` |
| `size` | 字号变化 | `fontSize × (1 + (rand - 0.5) × 0.12 × amount × size)` |
| `rotation` | 旋转角度 | `(rand - 0.5) × 0.07 × amount × rotation` |
| `baseline` | 基线偏移 | `(rand - 0.5) × 2.0 × amount × baseline × fontSize × 0.15` |
| `inkDensity` | 墨迹浓度 | `0.82 + rand × 0.18 × inkDensity × amount` |
| `inkColor` | 墨色变化 | `1 + (rand - 0.5) × 0.15 × inkColor × amount` |
| `spacing` | 间距抖动 | `(rand - 0.5) × 1.8 × amount × spacing` |
| `lineDrift` | 行漂移 | `(rand - 0.5) × 2.0 × amount × lineDrift × fontSize × 0.3` |
| `lineTilt` | 行倾斜 | `(rand - 0.5) × 0.008 × amount × lineTilt` |
| `halo` | 光晕效果 | 偏移重绘半透明文字 |
| `dryBrush` | 干笔效果 | 缩小字号偏移重绘 |

**确定性随机**: 使用 `seededRandom(seed)` 确保相同输入产生相同的抖动效果，seed = `pageIndex × 100001 + lineIndex × 9973 + charIndex × 137`

---

### Layer 3: 滤镜层 — `applyFilter()`

**源码位置**: [filterEffects.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Mercury/src/utils/filterEffects.ts#L4-L49)

**数据输入**:
| 数据 | 来源 | 说明 |
|------|------|------|
| `activeFilter` | Store | 滤镜类型 (none / 9种) |
| `filterIntensity` | Store | 滤镜强度 [0, 1] |
| `inkColor` | Store | 墨色 (用于墨迹检测) |

**处理流程**:
```
ctx.getImageData(0, 0, w, h)
        │
        ▼
  Uint8ClampedArray (像素数据)
        │
        ▼
  ┌─────────────────────────────┐
  │ isInkPixel() 墨迹像素检测    │
  │ dist = √((r-inkR)² + ...)  │
  │ dist < 100 || (r<150 && ..) │
  └─────────────────────────────┘
        │
        ▼
  switch(activeFilter) → 像素级变换
        │
        ▼
  ctx.putImageData(imageData, 0, 0)
  ctx.setTransform(DPR, 0, 0, DPR, 0, 0)  ← 重置变换矩阵
```

**9种滤镜算法核心**:
| 滤镜 | 核心算法 | 作用域 |
|------|----------|--------|
| `inkBleed` | 墨迹像素邻域扩散 | 墨迹像素 + 邻域 |
| `pencilSketch` | Sobel 边缘检测 + 灰度化 + 颗粒噪声 | 全像素 |
| `penStroke` | 墨迹加深 + 邻域墨迹扩散 | 墨迹像素 + 邻域 |
| `brushStroke` | 圆形邻域墨迹扩散 | 墨迹像素 + 邻域 |
| `watercolor` | 随机半径 RGB 通道分离扩散 | 墨迹像素 + 邻域 |
| `carbonCopy` | 灰度化 + 蓝色偏移 | 全像素 |
| `fountainPen` | 笔压模拟 + 墨迹羽化 | 墨迹像素 + 邻域 |
| `crayon` | 蜡质纹理随机覆盖 + 纸面颗粒 | 全像素 |
| `marker` | 曼哈顿距离扩散 + 颜色增艳 | 墨迹像素 + 邻域 |

---

### Layer 4: 签名层 — `drawSignaturesForPage()`

**源码位置**: [signatureRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Mercury/src/utils/signatureRenderer.ts#L167-L187)

**数据输入**:
| 数据 | 来源 | 说明 |
|------|------|------|
| `signatures` | Store | 签名数据数组 (含 dataUrl) |
| `signaturePlacements` | Store | 签名放置位置 |
| `signatureImages` | `loadSignatureImages()` | 异步加载的签名图片缓存 |

**渲染流程**:
```
signaturePlacements.filter(p => p.pageIndex === pageIdx)
        │
        ▼
  查找 Signature + HTMLImageElement
        │
        ▼
  drawSignatureOnCanvas()
    ├─ [bgOpacity > 0] drawSignatureBackground()
    │   ├─ 纸张背景色/渐变
    │   └─ 纸张线条 (line/grid/dotted)
    └─ ctx.drawImage(img, x, y, w, h)
```

**签名图片加载管线**:
```
Signature.dataUrl (PNG base64)
    │
    ▼
  new Image() → img.src = dataUrl
    │
    ▼
  loadSignatureImages() → Record<sigId, HTMLImageElement>
    │
    ▼
  useState 缓存 → setSignatureImages()
```

---

### Layer 5: 印章层 — `drawStampsForPage()`

**源码位置**: [stampRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Mercury/src/utils/stampRenderer.ts#L416-L436)

**数据输入**:
| 数据 | 来源 | 说明 |
|------|------|------|
| `stamps` | Store | 印章数据数组 (含 dataUrl + config) |
| `stampPlacements` | Store | 印章放置位置 |
| `stampImages` | `loadStampImages()` | 异步加载的印章图片缓存 |

**渲染流程**:
```
stampPlacements.filter(p => p.pageIndex === pageIdx)
        │
        ▼
  查找 Stamp + HTMLImageElement
        │
        ▼
  drawStampOnCanvas()
    ├─ ctx.globalAlpha = 0.92
    ├─ ctx.globalCompositeOperation = 'multiply'  ← 关键: 正片叠底混合
    └─ ctx.drawImage(img, x, y, w, h)
```

**印章预渲染管线** (在 Stamp 创建时):
```
StampConfig (shape/topText/centerText/bottomText/...)
        │
        ▼
  renderStampToCanvas()
    ├─ drawBorder()         圆形/方形/椭圆边框 (solid/double/dashed)
    ├─ drawArcTextEllipse() 弧形文字 (上/下弧)
    ├─ drawCenterText()     居中文字
    ├─ drawStar()           五角星装饰
    └─ drawStampNoise()     印章磨损噪声 (像素级)
        │
        ▼
  Stamp.dataUrl (PNG base64)
```

> **关键设计**: 印章使用 `multiply` 混合模式，模拟真实印章盖在纸上的正片叠底效果。

---

### Layer 6: 批注层 — `drawAnnotationsForPage()`

**源码位置**: [annotationRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Mercury/src/utils/annotationRenderer.ts#L162-L169)

**数据输入**:
| 数据 | 来源 | 说明 |
|------|------|------|
| `annotations` | Store | 批注数据数组 |
| `annotation.style` | 每条批注自带 | 颜色/线宽/透明度/填充色 |

**批注类型分发**:
```
annotations.filter(a => a.pageIndex === pageIdx)
        │
        ▼
  drawAnnotation() → switch(type)
    ├─ 'path'      → drawPath()      自由绘制路径
    ├─ 'highlight' → drawPath()      荧光笔 (高透明度 + 粗线宽)
    ├─ 'circle'    → drawShape()     椭圆 (ctx.ellipse)
    ├─ 'rect'      → drawShape()     矩形 (ctx.rect)
    ├─ 'line'      → drawLine()      直线
    ├─ 'arrow'     → drawLine()      箭头 (三角形箭头)
    ├─ 'underline' → drawUnderline() 直线下划线
    ├─ 'wavy'      → drawUnderline() 波浪线 (quadraticCurveTo)
    └─ 'text'      → drawText()      文字批注
```

---

### Layer 7: 装饰层 — `drawDecorationsForPage()`

**源码位置**: [decorationRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Mercury/src/utils/decorationRenderer.ts#L61-L106)

**数据输入**:
| 数据 | 来源 | 说明 |
|------|------|------|
| `decorationPlacements` | Store | 装饰放置位置 (含 opacity/rotation) |
| `decorationImages` | `loadDecorationImages()` | SVG→Image 缓存 |

**渲染流程**:
```
decorationPlacements.filter(d => d.pageIndex === pageIdx)
        │
        ▼
  查找 DecorationPreset
        │
        ▼
  ctx.save()
  ctx.globalAlpha = placement.opacity
  ctx.translate(placement.x, placement.y)
  ctx.rotate(placement.rotation × π / 180)
        │
        ▼
  三级绘制策略:
    ├─ [1级] decorationImages 缓存命中 → ctx.drawImage()
    ├─ [2级] getOrLoadImage() 重试加载 → ctx.drawImage()
    └─ [3级] 兜底占位矩形 (半透明棕色)
        │
        ▼
  ctx.restore()
```

**装饰图片加载管线**:
```
DecorationPreset.svgContent (SVG 字符串)
        │
        ▼
  new Blob([svgContent], {type: 'image/svg+xml'})
  URL.createObjectURL(blob)
        │
        ▼
  new Image() → img.src = blobURL
        │
        ▼
  loadDecorationImages() → DecorationImageCache
```

---

## 数据依赖关系图

```
                         ┌────────────────────┐
                         │  useWorkspaceStore  │
                         │    (Zustand Store)  │
                         └─────────┬──────────┘
                                   │
        ┌──────────────┬───────────┼───────────┬──────────────┐
        │              │           │           │              │
        ▼              ▼           ▼           ▼              ▼
  ┌───────────┐ ┌───────────┐ ┌────────┐ ┌──────────┐ ┌──────────┐
  │ 文本相关   │ │ 纸张相关   │ │ 滤镜   │ │ 放置相关  │ │ 样式相关  │
  │ rawText   │ │ paperId   │ │ filter │ │ sigPlace │ │ fontId   │
  │ fontSize  │ │ bgColor   │ │ intens │ │ stpPlace │ │ inkColor │
  │ inkColor  │ │ lineColor │ │        │ │ decPlace │ │ jitter*  │
  │ jitter*   │ │ lineSpace │ │        │ │          │ │ layout*  │
  │ layout*   │ │ margin*   │ │        │ │          │ │          │
  └─────┬─────┘ └─────┬─────┘ └───┬────┘ └────┬─────┘ └────┬─────┘
        │             │           │           │             │
        ▼             ▼           ▼           ▼             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                    useHandwritingRender                       │
  │                                                              │
  │  ┌─────────────┐    ┌──────────────┐    ┌────────────────┐  │
  │  │ RenderState  │    │ Image 缓存    │    │ computePages   │  │
  │  │ (useMemo)    │    │ (useState)    │    │ (useCallback)  │  │
  │  └──────┬──────┘    └──────┬───────┘    └───────┬────────┘  │
  │         │                  │                     │           │
  │         └──────────────────┼─────────────────────┘           │
  │                            │                                 │
  │                            ▼                                 │
  │                   renderToCanvas()                           │
  │                   renderAllCanvases()                        │
  └──────────────────────────────────────────────────────────────┘
                            │
                            ▼
                   ┌─────────────────┐
                   │  Canvas 输出     │
                   │  794×1123 @2x   │
                   └─────────────────┘
```

---

## Canvas 初始化与 DPR 适配

```
canvas.width  = PAGE_WIDTH  × DPR   (794 × 2 = 1588 物理像素)
canvas.height = PAGE_HEIGHT × DPR   (1123 × 2 = 2246 物理像素)
canvas.style.width  = PAGE_WIDTH + 'px'   (794 CSS 像素)
canvas.style.height = PAGE_HEIGHT + 'px'  (1123 CSS 像素)
ctx.setTransform(DPR, 0, 0, DPR, 0, 0)   (2倍缩放变换)
```

- `PAGE_WIDTH = 794` (A4 宽度 @96dpi)
- `PAGE_HEIGHT = 1123` (A4 高度 @96dpi)
- `DPR = 2` (固定2倍设备像素比，保证 Retina 屏清晰度)

> **注意**: 滤镜层 `applyFilter()` 操作的是物理像素 (`canvas.width × canvas.height`)，处理后需要重置 `ctx.setTransform(DPR, 0, 0, DPR, 0, 0)` 以恢复逻辑坐标系。

---

## 触发重渲染的数据流

```
Store 中任意状态变化
        │
        ▼
useHandwritingRender 内的 useMemo / useCallback 依赖更新
        │
        ▼
renderState (useMemo) 重新计算
        │
        ▼
renderToCanvas (useCallback) 重新生成
        │
        ▼
useEffect([canvasRef, renderToCanvas, currentPage]) 触发
        │
        ▼
renderToCanvas(canvasRef.current, currentPage - 1)
        │
        ▼
七层依次绘制 → Canvas 画面更新
```

**异步图片加载触发链**:
```
signatures 变化 → loadSignatureImages() → setSignatureImages() → drawSignatures 回调更新 → renderToCanvas 重建
stamps 变化     → loadStampImages()     → setStampImages()     → drawStamps 回调更新     → renderToCanvas 重建
组件挂载        → loadDecorationImages() → setDecorationImages() → drawDecorations 回调更新 → renderToCanvas 重建
```

---

## 导出管线 (`renderAllCanvases`)

导出时为每一页创建独立的离屏 Canvas，按相同的七层管线渲染，最终返回 `HTMLCanvasElement[]`:

```
for (let i = 0; i < pages.length; i++) {
    const c = document.createElement('canvas')
    c.width = PAGE_WIDTH × DPR
    c.height = PAGE_HEIGHT × DPR
    const ctx = c.getContext('2d')
    ctx.setTransform(DPR, 0, 0, DPR, 0, 0)

    drawPaper(ctx, renderState)                    ← Layer 1
    drawMarkdownPage / drawHandwrittenPage(...)     ← Layer 2
    applyFilter(ctx, ...)                           ← Layer 3
    drawSignaturesForPage({...})                    ← Layer 4
    drawStampsForPage({...})                        ← Layer 5
    drawAnnotationsForPage({...})                   ← Layer 6
    drawDecorationsForPage({...})                   ← Layer 7

    canvases.push(c)
}
return canvases
```

---

## 关键设计总结

| 设计点 | 实现方式 | 优势 |
|--------|----------|------|
| **单 Canvas 叠加** | 七层共享一个 Canvas 上下文 | 避免多 Canvas 内存开销，天然支持图层混合 |
| **确定性随机** | `seededRandom(seed)` 基于页面/行/字符索引 | 相同输入始终产生相同抖动，保证预览与导出一致 |
| **DPR 适配** | 物理像素 2x + `setTransform` 缩放 | Retina 屏清晰渲染 |
| **异步图片缓存** | `useState` + `loadXxxImages()` | 避免每帧重复加载 dataUrl/SVG |
| **印章正片叠底** | `globalCompositeOperation = 'multiply'` | 模拟真实印章盖印效果 |
| **滤镜像素操作** | `getImageData` / `putImageData` | 灵活实现9种墨迹效果 |
| **装饰三级降级** | 缓存 → 重试 → 占位矩形 | 保证 SVG 加载失败时仍有视觉反馈 |
| **Markdown/纯文本双模式** | 运行时切换渲染路径 | 兼顾简单文本和富文本排版需求 |
