# Canvas 七层叠加渲染管线数据流图

## 1. 概述

本文档深入解析手写字模拟器项目中的 Canvas 七层叠加渲染机制。渲染引擎采用**自底向上**的分层叠加策略，通过单一 Canvas 2D 上下文逐层绘制，最终合成完整的手写效果页面。

**核心渲染入口**：[useHandwritingRender.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/hooks/useHandwritingRender.ts) 中的 `renderToCanvas` 函数（第611-658行）

---

## 2. 七层渲染管线总览

### 2.1 层级结构图（从下到上）

```mermaid
graph TD
    subgraph "Canvas 2D Context 单一画布"
        L7["第7层：装饰层 Decoration Layer"]
        L6["第6层：批注层 Annotation Layer"]
        L5["第5层：印章层 Stamp Layer"]
        L4["第4层：签名层 Signature Layer"]
        L3["第3层：滤镜效果层 Filter Layer"]
        L2["第2层：手写文字层 Handwriting Layer"]
        L1["第1层：纸张背景层 Paper Layer"]
    end

    L1 -->|背景基底| L2
    L2 -->|像素级后处理| L3
    L3 -->|叠加签名| L4
    L4 -->|正片叠底印章| L5
    L5 -->|叠加批注| L6
    L6 -->|叠加装饰| L7
```

### 2.2 渲染执行顺序表

| 层级 | 层名称 | 绘制函数 | 核心文件 | 绘制时机 | 混合模式 |
|------|--------|----------|----------|----------|----------|
| 1 | 纸张背景层 | `drawPaper()` | [useHandwritingRender.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/hooks/useHandwritingRender.ts#L63-L157) | 最先绘制 | source-over |
| 2 | 手写文字层 | `drawHandwrittenPage()` / `drawMarkdownPage()` | [useHandwritingRender.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/hooks/useHandwritingRender.ts#L245-L398) / [markdownRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/markdown/markdownRenderer.ts) | 纸张之上 | source-over |
| 3 | 滤镜效果层 | `applyFilter()` | [filterEffects.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/filterEffects.ts) | 文字层之后（像素级处理） | 直接修改 ImageData |
| 4 | 签名层 | `drawSignaturesForPage()` | [signatureRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/signatureRenderer.ts) | 滤镜之后 | source-over |
| 5 | 印章层 | `drawStampsForPage()` | [stampRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/stampRenderer.ts) | 签名之上 | multiply（正片叠底） |
| 6 | 批注层 | `drawAnnotationsForPage()` | [annotationRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/annotationRenderer.ts) | 印章之上 | source-over |
| 7 | 装饰层 | `drawDecorationsForPage()` | [decorationRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/decorationRenderer.ts) | 最后绘制 | source-over |

---

## 3. 渲染管线数据流图

### 3.1 完整数据流图

```mermaid
flowchart TB
    subgraph "输入层 Input Layer"
        IN1["rawText 原始文本"]
        IN2["WorkspaceStore 全局状态"]
        IN3["signatures[] 签名数据"]
        IN4["stamps[] 印章数据"]
        IN5["annotations[] 批注数据"]
        IN6["decorationPlacements[] 装饰布局"]
    end

    subgraph "预处理层 Preprocessing"
        PP1["breakTextIntoLines()<br/>文本分行"]
        PP2["paginate()<br/>文本分页"]
        PP3["layoutMarkdown()<br/>Markdown排版"]
        PP4["loadSignatureImages()<br/>签名图片加载"]
        PP5["loadStampImages()<br/>印章图片加载"]
        PP6["loadDecorationImages()<br/>装饰SVG加载"]
    end

    subgraph "渲染管线 Render Pipeline"
        direction TB
        R1["drawPaper()<br/>第1层：纸张背景"]
        R2["drawHandwrittenPage() / drawMarkdownPage()<br/>第2层：手写文字"]
        R3["applyFilter()<br/>第3层：滤镜效果"]
        R4["drawSignaturesForPage()<br/>第4层：签名"]
        R5["drawStampsForPage()<br/>第5层：印章"]
        R6["drawAnnotationsForPage()<br/>第6层：批注"]
        R7["drawDecorationsForPage()<br/>第7层：装饰"]
    end

    subgraph "输出层 Output"
        OUT1["Canvas HTMLCanvasElement"]
        OUT2["PNG / PDF 导出"]
    end

    IN1 --> PP1
    IN1 --> PP3
    IN2 --> R1
    IN2 --> R2
    IN2 --> R3

    PP1 --> PP2
    PP2 --> R2
    PP3 --> R2

    IN3 --> PP4
    IN4 --> PP5
    IN6 --> PP6

    PP4 --> R4
    PP5 --> R5
    IN5 --> R6
    PP6 --> R7

    R1 --> R2
    R2 --> R3
    R3 --> R4
    R4 --> R5
    R5 --> R6
    R6 --> R7

    R7 --> OUT1
    OUT1 --> OUT2
```

---

## 4. 各层详细解析

### 4.1 第1层：纸张背景层 (Paper Layer)

**核心函数**：[drawPaper()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/hooks/useHandwritingRender.ts#L63-L157)

**职责**：构建页面最底层的纸张视觉效果

**绘制子流程**：

```mermaid
flowchart LR
    A["开始绘制纸张"] --> B["填充背景色<br/>纯色 / 牛皮纸渐变"]
    B --> C["绘制信纸纹理"]
    C --> D{"paperType?"}
    D -->|blank| E["无纹理"]
    D -->|line| F["绘制横线"]
    D -->|grid| G["绘制网格"]
    D -->|dotted| H["绘制点阵"]
    D -->|kraft| I["牛皮纸渐变"]
    E --> J
    F --> J
    G --> J
    H --> J
    I --> J
    J{"showMargin?"}
    J -->|true| K["绘制装订线 + 打孔"]
    J -->|false| L["跳过"]
    K --> M["添加纸张噪点纹理"]
    L --> M
    M --> N["完成"]
```

**关键技术点**：
- 牛皮纸效果使用 `createLinearGradient` 创建多色渐变
- 横线/网格/点阵使用 `stroke()` 和 `arc()` 逐点绘制
- 装订线包含红色竖线 + 三个圆形打孔（白色填充 + 浅棕描边）
- 最后叠加一层半透明渐变噪点模拟纸张质感
- 所有坐标基于 `marginTop/marginBottom/marginLeft/marginRight` 计算

---

### 4.2 第2层：手写文字层 (Handwriting Layer)

**核心函数**：[drawHandwrittenPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/hooks/useHandwritingRender.ts#L245-L398)

**职责**：在纸张背景上逐字绘制带有手写抖动效果的文字

**绘制子流程**：

```mermaid
flowchart TB
    A["开始绘制手写页"] --> B["初始化渲染状态"]
    B --> C["遍历每一行"]
    C --> D["生成行级随机抖动<br/>lineDrift + lineTilt"]
    D --> E["遍历每行每个字符"]
    E --> F["生成字符级抖动参数"]
    
    subgraph "字符抖动参数 CharDrawInfo"
        F1["sizeVar - 字号波动<br/>±12%"]
        F2["dx/dy - 位置偏移<br/>X: ±25% Y: ±15%"]
        F3["rot - 旋转角度<br/>±0.07 rad"]
        F4["alpha - 墨迹浓度<br/>0.82 ~ 1.0"]
        F5["inkR/G/B - 墨色波动<br/>±15%"]
        F6["baseline - 基线偏移<br/>±15%"]
        F7["spacingJitter - 字距波动"]
        F8["haloDx/Dy - 墨迹晕染偏移"]
        F9["drySize - 干笔效果尺寸"]
    end
    
    F --> F1 & F2 & F3 & F4 & F5 & F6 & F7 & F8 & F9
    F1 & F2 & F3 & F4 & F5 & F6 & F7 & F8 & F9 --> G["计算总宽度 + 挤压系数"]
    G --> H["逐字绘制 fillText"]
    H --> I{"halo 效果启用?"}
    I -->|true| J["绘制墨迹晕染重影"]
    I -->|false| K
    J --> K{"dryBrush 启用?"}
    K -->|true| L["绘制干笔飞白效果"]
    K -->|false| M["下一个字符"]
    L --> M
    M --> E
    E -->|"行结束"| N["更新 y 坐标 + 行高"]
    N --> C
    C -->|"页结束"| O["完成"]
```

**关键技术点**：
- 使用 `seededRandom()` 种子随机数确保同一页每次渲染结果一致
- 每个字符独立变换：`translate → rotate → fillText`
- 支持 Markdown 模式，通过 `drawMarkdownPage()` 实现富文本排版
- 文字超宽时自动 `squeeze` 压缩字间距
- 支持 **halo（墨迹晕染）** 和 **dryBrush（干笔飞白）** 两种高级效果

---

### 4.3 第3层：滤镜效果层 (Filter Layer)

**核心函数**：[applyFilter()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/filterEffects.ts#L4-L49)

**职责**：对前两层合成结果进行像素级后处理，模拟不同书写工具效果

**处理方式**：`getImageData() → 像素操作 → putImageData()`

**滤镜类型与数据流**：

```mermaid
flowchart TB
    A["获取 ImageData"] --> B{"filter 类型?"}
    
    B -->|inkBleed| C["墨迹晕染<br/>墨水向周围扩散 + 随机抖动"]
    B -->|pencilSketch| D["铅笔素描<br/>Sobel边缘检测 + 灰度 + 颗粒"]
    B -->|penStroke| E["钢笔笔触<br/>压力变化 + 边缘飞溅"]
    B -->|brushStroke| F["毛笔笔触<br/>笔刷半径扩散 + 干笔效果"]
    B -->|watercolor| G["水彩渲染<br/>RGB分色扩散 + 湿边效果"]
    B -->|carbonCopy| H["复写纸<br/>蓝色偏移 + 褪色"]
    B -->|fountainPen| I["钢笔墨痕<br/>起笔流量变化 + 飞白"]
    B -->|crayon| J["蜡笔涂鸦<br/>蜡质纹理 + 纸张颗粒"]
    B -->|marker| K["马克笔<br/>曼哈顿距离扩散 + 鲜艳增强"]
    
    C --> L["写回 ImageData"]
    D --> L
    E --> L
    F --> L
    G --> L
    H --> L
    I --> L
    J --> L
    K --> L
    
    L --> M["重置 DPR 变换矩阵"]
    M --> N["后续层级继续绘制"]
```

**关键技术点**：
- 所有滤镜操作在**像素级别**完成，直接修改 `Uint8ClampedArray`
- 通过 `isInkPixel()` 判断墨迹像素（基于颜色距离阈值）
- 滤镜强度 `intensity` 参数控制效果程度（0~1）
- 滤镜应用后需调用 `ctx.setTransform(DPR, 0, 0, DPR, 0, 0)` 重置变换矩阵
- **重要**：滤镜只作用于第1、2层，第4-7层不受滤镜影响

---

### 4.4 第4层：签名层 (Signature Layer)

**核心函数**：[drawSignaturesForPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/signatureRenderer.ts#L167-L187)

**职责**：在页面指定位置放置手写签名图片

**绘制流程**：

```mermaid
flowchart TB
    A["获取当前页签名布局"] --> B["遍历每个签名放置"]
    B --> C["查找签名对象 + 图片"]
    C --> D{"bgOpacity > 0?"}
    D -->|true| E["绘制签名背景纸<br/>背景色 + 信纸纹理"]
    D -->|false| F["跳过背景"]
    E --> G["ctx.drawImage() 绘制签名"]
    F --> G
    G --> H["下一个签名"]
    H --> B
    B -->|完成| I["结束"]
```

**关键技术点**：
- 签名图片预加载并存入 `signatureImages` 状态缓存
- 支持自定义背景纸（与第1层纸张效果一致）
- 放置位置基于中心点坐标 `(x, y)` + `scale` 缩放
- 使用 `drawImage()` 直接绘制，支持透明背景
- 每个签名关联 `pageIndex`，只在对应页绘制

---

### 4.5 第5层：印章层 (Stamp Layer)

**核心函数**：[drawStampsForPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/stampRenderer.ts#L416-L436)

**职责**：在页面上以正片叠底模式叠加印章效果

**绘制流程**：

```mermaid
flowchart TB
    A["获取当前页印章布局"] --> B["遍历每个印章放置"]
    B --> C["查找印章对象 + 图片"]
    C --> D["设置混合模式"]
    D --> D1["globalAlpha = 0.92"]
    D --> D2["globalCompositeOperation = 'multiply'"]
    D1 & D2 --> E["ctx.drawImage() 绘制印章"]
    E --> F["恢复混合模式"]
    F --> G["下一个印章"]
    G --> B
    B -->|完成| H["结束"]
```

**印章内部结构（渲染时）**：

```mermaid
flowchart LR
    A["印章配置 StampConfig"] --> B["drawBorder()<br/>边框：实线/双线/虚线"]
    A --> C["drawArcTextEllipse()<br/>顶部弧形文字"]
    A --> D["drawArcTextEllipse()<br/>底部弧形文字"]
    A --> E["drawStar()<br/>中心五角星"]
    A --> F["drawCenterText()<br/>中心文字"]
    B & C & D & E & F --> G["drawStampNoise()<br/>印章噪点纹理"]
    G --> H["生成 dataUrl"]
```

**关键技术点**：
- 使用 `globalCompositeOperation = 'multiply'` 正片叠底模拟真实印章盖印效果
- `globalAlpha = 0.92` 模拟印泥不饱满的真实感
- 印章渲染时内置噪点纹理（缺墨、斑驳效果）
- 支持圆形、方形、椭圆形三种印章形状
- 支持顶部弧形文字、底部弧形文字、中心五角星、中心文字等元素
- 印章图片预生成并缓存为 DataURL

---

### 4.6 第6层：批注层 (Annotation Layer)

**核心函数**：[drawAnnotationsForPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/annotationRenderer.ts#L162-L169)

**职责**：在页面上绘制各类批注标记（笔迹、图形、文字等）

**批注类型与绘制方法**：

```mermaid
flowchart TB
    A["获取当前页批注"] --> B["遍历每个批注"]
    B --> C{"annotation.type?"}
    
    C -->|path / highlight| D["drawPath()<br/>路径/荧光笔"]
    C -->|circle / rect| E["drawShape()<br/>圆形/矩形"]
    C -->|line / arrow| F["drawLine()<br/>直线/箭头"]
    C -->|underline / wavy| G["drawUnderline()<br/>下划线/波浪线"]
    C -->|text| H["drawText()<br/>文字批注"]
    
    D --> I["下一个批注"]
    E --> I
    F --> I
    G --> I
    H --> I
    
    I --> B
    B -->|完成| J["结束"]
```

**关键技术点**：
- 每种批注类型独立绘制函数，统一通过 `drawAnnotation()` 分发
- 样式统一通过 `applyStyle()` 设置：颜色、线宽、透明度、lineCap、lineJoin
- 荧光笔使用 `path` 类型 + 半透明颜色模拟
- 箭头使用 `Math.atan2()` 计算角度 + 三角形填充
- 波浪线使用 `quadraticCurveTo()` 绘制正弦曲线
- 所有批注关联 `pageIndex`，支持多页批注

---

### 4.7 第7层：装饰层 (Decoration Layer)

**核心函数**：[drawDecorationsForPage()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/decorationRenderer.ts#L61-L106)

**职责**：在页面最顶层添加装饰元素（胶带、贴纸、花朵等）

**绘制流程**：

```mermaid
flowchart TB
    A["获取当前页装饰布局"] --> B["遍历每个装饰放置"]
    B --> C["查找装饰预设 preset"]
    C --> D["设置绘制状态"]
    D --> D1["globalAlpha = opacity"]
    D --> D2["translate(x, y)"]
    D --> D3["rotate(rotation)"]
    
    D1 & D2 & D3 --> E{"图片加载成功?"}
    E -->|是| F["ctx.drawImage() 绘制SVG"]
    E -->|否| G["重试加载 getOrLoadImage()"]
    G --> H{"重试成功?"}
    H -->|是| F
    H -->|否| I["绘制占位矩形"]
    
    F --> J["恢复状态"]
    I --> J
    J --> K["下一个装饰"]
    K --> B
    B -->|完成| L["结束"]
```

**装饰分类**：

| 类别 | 说明 | 示例 |
|------|------|------|
| tape | 胶带 | 透明胶带、纸胶带 |
| sticker | 贴纸 | 卡通贴纸、标签贴纸 |
| flower | 花朵 | 干花、植物装饰 |
| stamp | 邮戳 | 复古邮戳、纪念章 |
| corner | 角贴 | 照片角贴、装饰角 |
| ribbon | 丝带 | 缎带、蝴蝶结 |

**关键技术点**：
- 装饰元素以 SVG 格式存储，通过 `Blob + URL.createObjectURL` 转成图片绘制
- 支持三级降级：缓存图片 → 重试加载 → 占位矩形
- 每个装饰独立变换：`translate → rotate → drawImage`
- 支持透明度 `opacity` 和旋转角度 `rotation`
- 装饰放置数据包含：位置、尺寸、旋转、透明度、关联页面

---

## 5. 核心渲染函数调用链

### 5.1 renderToCanvas 调用链

```mermaid
sequenceDiagram
    participant Store as useWorkspaceStore
    participant Hook as useHandwritingRender
    participant Paper as drawPaper
    participant Text as drawHandwrittenPage
    participant Filter as applyFilter
    participant Sig as drawSignaturesForPage
    participant Stamp as drawStampsForPage
    participant Ann as drawAnnotationsForPage
    participant Dec as drawDecorationsForPage

    Store->>Hook: 状态变更触发重渲染
    Hook->>Hook: renderToCanvas(canvas, pageIdx)
    Hook->>Hook: 设置Canvas尺寸 + DPR
    
    Note over Hook: 第1层：纸张背景
    Hook->>Paper: drawPaper(ctx, renderState)
    Paper-->>Hook: 返回
    
    Note over Hook: 第2层：手写文字
    Hook->>Text: drawHandwrittenPage(ctx, pages, rs, idx)
    Text-->>Hook: 返回
    
    Note over Hook: 第3层：滤镜效果
    Hook->>Filter: applyFilter(ctx, filter, intensity, inkColor)
    Filter->>Filter: getImageData + 像素处理
    Filter->>Filter: putImageData
    Filter->>Hook: setTransform(DPR)
    Filter-->>Hook: 返回
    
    Note over Hook: 第4层：签名
    Hook->>Sig: drawSignaturesForPage(ctx, pageIdx, ...)
    Sig-->>Hook: 返回
    
    Note over Hook: 第5层：印章
    Hook->>Stamp: drawStampsForPage(ctx, pageIdx, ...)
    Stamp->>Stamp: globalCompositeOperation = multiply
    Stamp-->>Hook: 返回
    
    Note over Hook: 第6层：批注
    Hook->>Ann: drawAnnotationsForPage(ctx, pageIdx, ...)
    Ann-->>Hook: 返回
    
    Note over Hook: 第7层：装饰
    Hook->>Dec: drawDecorationsForPage(ctx, pageIdx, ...)
    Dec-->>Hook: 返回
```

---

## 6. 状态驱动与响应式渲染

### 6.1 状态依赖关系图

```mermaid
flowchart LR
    subgraph "Zustand Store 状态源"
        S1["rawText"]
        S2["字体配置<br/>fontId, fontSize, inkColor, jitter..."]
        S3["纸张配置<br/>paperId, bgColor, lineColor..."]
        S4["排版配置<br/>letterSpacing, lineHeight, margins..."]
        S5["signatures + placements"]
        S6["stamps + placements"]
        S7["annotations"]
        S8["decorationPlacements"]
        S9["activeFilter + filterIntensity"]
        S10["currentPage"]
    end

    subgraph "useMemo 计算状态"
        M1["renderState<br/>渲染状态聚合"]
        M2["totalPages<br/>总页数计算"]
    end

    subgraph "useEffect 渲染触发"
        E1["字体加载 effect"]
        E2["签名图片加载 effect"]
        E3["印章图片加载 effect"]
        E4["装饰图片加载 effect"]
        E5["总页数同步 effect"]
        E6["Canvas 渲染 effect"]
    end

    S1 & S2 & S3 & S4 & S9 --> M1
    M1 --> M2
    S1 --> M2

    S2 --> E1
    S5 --> E2
    S6 --> E3
    S8 --> E4
    M2 --> E5
    
    M1 & M2 & E1 & E2 & E3 & E4 & S5 & S6 & S7 & S8 & S10 --> E6
    E6 --> Out["Canvas 绘制"]
```

### 6.2 性能优化策略

| 优化手段 | 实现位置 | 说明 |
|----------|----------|------|
| `useMemo` 缓存渲染状态 | [useHandwritingRender.ts:442](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/hooks/useHandwritingRender.ts#L442-L487) | 避免重复构建 renderState |
| `useCallback` 缓存绘制函数 | 多处 | 减少函数重建 |
| 图片预加载缓存 | signatureImages / stampImages / decorationImages | 避免重复加载图片 |
| 种子随机数 | `seededRandom()` | 同页渲染结果一致，减少重绘差异 |
| DPR 预计算 | `ctx.setTransform(DPR, 0, 0, DPR, 0, 0)` | 一次性设置像素比，避免逐点计算 |
| 只渲染当前页 | `renderToCanvas(canvas, currentPage - 1)` | 不渲染不可见页面 |

---

## 7. 导出管线（全页渲染）

### 7.1 renderAllCanvases 多页导出流程

```mermaid
flowchart TB
    A["开始导出"] --> B["计算所有页 pages[]"]
    B --> C["预加载所有资源<br/>签名/印章/装饰图片"]
    C --> D["遍历每一页"]
    D --> E["创建临时 Canvas"]
    E --> F["七层绘制（同预览）"]
    F --> G["Canvas 加入数组"]
    G --> D
    D -->|全部完成| H["返回 Canvas[]"]
    H --> I{"导出格式?"}
    I -->|PNG| J["逐页 toBlob() 下载"]
    I -->|PDF| K["jsPDF + addImage() 逐页加入"]
    J --> L["完成"]
    K --> L
```

**核心函数**：[renderAllCanvases()](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/hooks/useHandwritingRender.ts#L660-L734)

---

## 8. 总结

### 8.1 设计亮点

1. **分层清晰**：七层各司其职，从背景到前景逻辑分明
2. **像素级控制**：滤镜层直接操作 ImageData，效果丰富
3. **状态驱动**：Zustand + useMemo + useEffect 响应式渲染
4. **性能优化**：资源缓存、按需渲染、种子随机数
5. **可扩展性**：各层独立模块，新增效果不影响其他层
6. **单一 Canvas**：使用单一画布 + 分层绘制，避免多 Canvas 合成开销

### 8.2 关键文件索引

| 模块 | 文件路径 | 核心功能 |
|------|----------|----------|
| 渲染主 Hook | [useHandwritingRender.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/hooks/useHandwritingRender.ts) | 七层渲染调度核心 |
| 纸张绘制 | 同上 drawPaper() | 第1层：纸张背景 |
| 手写文字绘制 | 同上 drawHandwrittenPage() | 第2层：手写文字 |
| 滤镜效果 | [filterEffects.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/filterEffects.ts) | 第3层：9种滤镜效果 |
| 签名渲染 | [signatureRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/signatureRenderer.ts) | 第4层：签名叠加 |
| 印章渲染 | [stampRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/stampRenderer.ts) | 第5层：印章叠加 |
| 批注渲染 | [annotationRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/annotationRenderer.ts) | 第6层：批注绘制 |
| 装饰渲染 | [decorationRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/decorationRenderer.ts) | 第7层：装饰叠加 |
| Canvas 工具 | [canvasUtils.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/canvasUtils.ts) | 种子随机数、颜色转换等 |
| 状态管理 | [useWorkspaceStore.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/store/useWorkspaceStore.ts) | 全局状态管理 |
| Markdown 渲染 | [markdownRenderer.ts](file:///Volumes/ExMac/traeProject/0617GSB/yq-33/yq-33-617_Saturn/src/utils/markdown/markdownRenderer.ts) | 第2层 Markdown 模式 |
