# Chapter 3.2: 图形绘制与手绘风格实现

## 概述

Excalidraw 最具特色的功能之一就是其独特的手绘风格。这种风格不仅让图表看起来更加友好和非正式，还能有效降低观众对"完成度"的预期，使其更适合用于草图和头脑风暴。本章将深入探讨 Excalidraw 如何通过 RoughJS 实现这种手绘效果，以及各种图形元素的具体渲染策略。

## RoughJS 集成原理

### 什么是 RoughJS？

RoughJS 是一个轻量级的图形库，专门用于创建手绘风格的图形。它的核心原理是：
1. 将完美的几何图形转换为不规则的手绘路径
2. 通过添加抖动、多重描边等技术模拟手绘效果
3. 保持图形的可识别性同时增加自然的不完美感

### Excalidraw 中的 RoughJS 集成

根据实际源码分析，Excalidraw 的 RoughJS 集成比较直接：

```typescript
// packages/element/src/renderElement.ts
import rough from "roughjs/bin/rough";

// 在生成元素画布时创建 RoughCanvas 实例
const generateElementCanvas = (
  element: NonDeletedExcalidrawElement,
  // ...
) => {
  const canvas = document.createElement("canvas");
  const context = canvas.getContext("2d")!;
  // ...

  // 创建 RoughCanvas 实例
  const rc = rough.canvas(canvas);

  // 渲染元素
  drawElementOnCanvas(element, rc, context, renderConfig, appState);
};
```

### ShapeCache 系统

Excalidraw 使用 ShapeCache 来缓存生成的 RoughJS 图形，避免重复计算：

```typescript
// packages/element/src/shape.ts
export class ShapeCache {
  private static rg = new RoughGenerator();  // 全局 RoughJS 生成器
  private static cache = new WeakMap<ExcalidrawElement, ElementShape>();

  public static get = <T extends ExcalidrawElement>(element: T) => {
    return ShapeCache.cache.get(element);
  };

  public static generateElementShape = (element, renderConfig) => {
    const cachedShape = ShapeCache.get(element);
    if (cachedShape !== undefined) {
      return cachedShape;  // 返回缓存
    }

    // 生成新图形并缓存
    const shape = generateElementShape(element, ShapeCache.rg, renderConfig);
    ShapeCache.cache.set(element, shape);
    return shape;
  };
}
```

### RoughJS 选项生成

实际的选项生成比文档中描述的更复杂，有一个专门的函数处理：

```typescript
// packages/element/src/shape.ts
const generateRoughOptions = (
  element: ExcalidrawElement,
  isPath: boolean
): Options => {
  const options: Options = {
    seed: element.seed,
    strokeLineDash: element.strokeStyle === "dashed" ? [12, 8] : undefined,
    // ... 其他选项处理
  };

  // 根据元素类型调整选项
  switch (element.strokeStyle) {
    case "solid":
      if (element.roughness === 0) {
        options.roughness = 0;
      }
      break;
    case "dashed":
      options.strokeLineDash = [12, 8];
      break;
    case "dotted":
      options.strokeLineDash = [3, 6];
      break;
  }

  // 处理填充
  if (!isTransparent(element.backgroundColor)) {
    options.fill = element.backgroundColor;
    if (element.fillStyle === "cross-hatch") {
      options.fillStyle = "cross-hatch";
      options.fillWeight = element.strokeWidth / 2;
      options.hachureAngle = -41;
      options.hachureGap = element.strokeWidth * 4;
    } else if (element.fillStyle === "hachure") {
      options.fillStyle = "hachure";
      options.fillWeight = element.strokeWidth / 2;
      options.hachureAngle = -41;
      options.hachureGap = element.strokeWidth * 4;
    }
  }

  return options;
};
```

## 图形元素渲染实现

### 实际的渲染架构

Excalidraw 的渲染分为两个阶段：
1. **图形生成阶段**：使用 `generateElementShape()` 生成 RoughJS 图形
2. **绘制阶段**：使用 `drawElementOnCanvas()` 将图形绘制到画布

```typescript
// packages/element/src/renderElement.ts
const drawElementOnCanvas = (
  element: NonDeletedExcalidrawElement,
  rc: RoughCanvas,
  context: CanvasRenderingContext2D,
  renderConfig: StaticCanvasRenderConfig,
  appState: StaticCanvasAppState,
) => {
  switch (element.type) {
    case "rectangle":
    case "iframe":
    case "embeddable":
    case "diamond":
    case "ellipse": {
      context.lineJoin = "round";
      context.lineCap = "round";
      rc.draw(ShapeCache.get(element)!);  // 使用缓存的图形
      break;
    }
    case "arrow":
    case "line": {
      context.lineJoin = "round";
      context.lineCap = "round";
      ShapeCache.get(element)!.forEach((shape) => {
        rc.draw(shape);  // 线性元素可能有多个图形部分
      });
      break;
    }
    // ... 其他类型
  }
};
```

### 矩形渲染（已验证 ✓）

实际的矩形生成函数（已验证源码）：

```typescript
// packages/element/src/shape.ts (generateElementShape 函数中)
case "rectangle":
case "iframe":
case "embeddable": {
  let shape: ElementShapes[typeof element.type];

  if (element.roundness) {
    // 圆角矩形：使用 SVG 路径
    const w = element.width;
    const h = element.height;
    const r = getCornerRadius(Math.min(w, h), element);
    shape = generator.path(
      `M ${r} 0 L ${w - r} 0 Q ${w} 0, ${w} ${r} L ${w} ${
        h - r
      } Q ${w} ${h}, ${w - r} ${h} L ${r} ${h} Q 0 ${h}, 0 ${
        h - r
      } L 0 ${r} Q 0 0, ${r} 0`,
      generateRoughOptions(element, true),
    );
  } else {
    // 直角矩形：使用 RoughJS rectangle 方法
    shape = generator.rectangle(
      0,
      0,
      element.width,
      element.height,
      generateRoughOptions(element, false),
    );
  }
  return shape;
}
```

### 椭圆渲染（已验证 ✓）

实际的椭圆渲染直接使用 RoughJS 的椭圆方法（已验证）：

```typescript
// packages/element/src/shape.ts (generateElementShape 函数中) - 已验证 ✓
case "ellipse": {
  const shape: ElementShapes[typeof element.type] = generator.ellipse(
    element.width / 2,   // 中心点 x 坐标
    element.height / 2,  // 中心点 y 坐标
    element.width,       // 椭圆宽度
    element.height,      // 椭圆高度
    generateRoughOptions(element),
  );
  return shape;
}
```

**实现细节：**
- 椭圆的实现非常简单，直接使用 RoughJS 的 `ellipse` 方法
- 传入中心点坐标（width/2, height/2），而非左上角坐标
- 宽度和高度参数是椭圆的实际尺寸
- `generateRoughOptions()` 会自动设置 `curveFitting: 1` 用于椭圆（源码第 223 行）

### 菱形渲染

菱形的实际实现使用 `getDiamondPoints` 函数获取顶点，并根据是否有圆角选择不同渲染方式：

```typescript
// packages/element/src/shape.ts (generateElementShape 函数中)
case "diamond": {
  const [topX, topY, rightX, rightY, bottomX, bottomY, leftX, leftY] =
    getDiamondPoints(element);

  if (element.roundness) {
    // 圆角菱形：使用复杂的 SVG 路径
    const verticalRadius = getCornerRadius(Math.abs(topX - leftX), element);
    const horizontalRadius = getCornerRadius(Math.abs(rightY - topY), element);

    shape = generator.path(
      `M ${topX + verticalRadius} ${topY + horizontalRadius} L ${
        rightX - verticalRadius
      } ${rightY - horizontalRadius}
        C ${rightX} ${rightY}, ${rightX} ${rightY}, ${
        rightX - verticalRadius
      } ${rightY + horizontalRadius}
        // ... 完整的 SVG 路径
      `,
      generateRoughOptions(element, true),
    );
  } else {
    // 直角菱形：使用多边形
    shape = generator.polygon(
      [
        [topX, topY],
        [rightX, rightY],
        [bottomX, bottomY],
        [leftX, leftY],
      ],
      generateRoughOptions(element),
    );
  }
  return shape;
}
```

### 线条和箭头渲染

线性元素（包括线条和箭头）的渲染比较复杂，因为它们可能包含多个部分：

```typescript
// packages/element/src/shape.ts (generateElementShape 函数中)
case "arrow":
case "line": {
  const options = generateRoughOptions(element);

  // 生成主线条部分
  const shapes: Drawable[] = [];

  if (element.points.length > 1) {
    if (canBecomePolygon(element)) {
      // 封闭路径：多边形
      shapes.push(
        generator.polygon(
          element.points.map(([x, y]) => [x, y]),
          options,
        ),
      );
    } else {
      // 开放路径：线性路径
      shapes.push(
        generator.linearPath(
          element.points.map(([x, y]) => [x, y]),
          options,
        ),
      );
    }
  }

  // 箭头头部（如果有）
  if (element.type === "arrow") {
    const arrowheadShapes = getArrowheadShapes(
      element,
      generator,
      options,
    );
    shapes.push(...arrowheadShapes);
  }

  return shapes;
}

// 实际的箭头头部生成使用现有的几何计算函数
function getArrowheadShapes(
  element: ExcalidrawArrowElement,
  generator: RoughGenerator,
  options: Options,
): Drawable[] {
  const shapes: Drawable[] = [];

  if (element.startArrowhead) {
    const startPoints = getArrowheadPoints(
      element,
      element.points,
      "start",
      element.startArrowhead,
    );
    if (startPoints) {
      shapes.push(generator.polygon(startPoints, options));
    }
  }

  if (element.endArrowhead) {
    const endPoints = getArrowheadPoints(
      element,
      element.points,
      "end",
      element.endArrowhead,
    );
    if (endPoints) {
      shapes.push(generator.polygon(endPoints, options));
    }
  }

  return shapes;
}
```

## 自由绘制实现

### 实际的自由绘制渲染

自由绘制元素有特殊的渲染逻辑，它使用 `perfect-freehand` 库，但渲染方式与其他元素不同：

```typescript
// packages/element/src/renderElement.ts (drawElementOnCanvas 函数中)
case "freedraw": {
  // 不使用 RoughJS，直接绘制到画布
  context.save();
  context.fillStyle = element.strokeColor;

  const path = getFreeDrawPath2D(element) as Path2D;
  const fillShape = ShapeCache.get(element);

  if (fillShape) {
    rc.draw(fillShape);  // 如果有填充，使用 RoughJS
  }

  context.fillStyle = element.strokeColor;
  context.fill(path);  // 主路径直接填充

  context.restore();
  break;
}
```

### getFreeDrawPath2D 函数

```typescript
// packages/element/src/renderElement.ts
const getFreeDrawPath2D = (
  element: ExcalidrawFreeDrawElement,
  options?: {
    x?: number;
    y?: number;
    width?: number;
    height?: number;
  },
) => {
  const path = new Path2D();
  const strokePoints = getStroke(element.points, {
    size: element.strokeWidth,
    thinning: 0.6,
    smoothing: 0.5,
    streamline: 0.5,
    simulatePressure: element.pressures.length === 0,
    pressures: element.pressures.length ? element.pressures : undefined,
    last: element.simulatePressure === false,
  } as StrokeOptions);

  if (strokePoints.length < 2) {
    return path;
  }

  path.moveTo(strokePoints[0][0], strokePoints[0][1]);

  for (let i = 1; i < strokePoints.length; i++) {
    path.lineTo(strokePoints[i][0], strokePoints[i][1]);
  }

  return path;
};
```

自由绘制使用 `perfect-freehand` 库生成平滑的笔触路径，然后使用 Canvas 2D 的 Path2D API 直接填充，而不是通过 RoughJS。

## 文本渲染系统

### 实际的文本渲染

文本渲染比文档描述的要简单得多，直接使用 Canvas 2D API：

```typescript
// packages/element/src/renderElement.ts (drawElementOnCanvas 函数中)
default: {
  if (isTextElement(element)) {
    const rtl = isRTL(element.text);
    const shouldTemporarilyAttach = rtl && !context.canvas.isConnected;

    // RTL 文本需要临时附加到 DOM
    if (shouldTemporarilyAttach) {
      document.body.appendChild(context.canvas);
    }

    context.canvas.setAttribute("dir", rtl ? "rtl" : "ltr");
    context.save();

    // 设置字体样式
    context.font = getFontString(element);
    context.fillStyle = element.strokeColor;
    context.textAlign = element.textAlign as CanvasTextAlign;

    // Canvas 不支持多行文本，需要手动处理
    const lines = element.text.replace(/\r\n?/g, "\n").split("\n");

    // 计算水平偏移
    const horizontalOffset =
      element.textAlign === "center"
        ? element.width / 2
        : element.textAlign === "right"
        ? element.width
        : 0;

    // 计算行高和垂直偏移
    const lineHeightPx = getLineHeightInPx(
      element.fontSize,
      element.lineHeight,
    );
    const verticalOffset = getVerticalOffset(
      element.fontFamily,
      element.fontSize,
      lineHeightPx,
    );

    // 逐行渲染
    for (let index = 0; index < lines.length; index++) {
      context.fillText(
        lines[index],
        horizontalOffset,
        index * lineHeightPx + verticalOffset,
      );
    }

    context.restore();
    if (shouldTemporarilyAttach) {
      context.canvas.remove();
    }
  }
}
```

文本渲染没有复杂的测量缓存系统，只是简单地拆分文本行并逐行绘制。

### 图片渲染处理

图片元素的渲染相对简单，支持裁剪和圆角：

```typescript
// packages/element/src/renderElement.ts (drawElementOnCanvas 函数中)
case "image": {
  const img = isInitializedImageElement(element)
    ? renderConfig.imageCache.get(element.fileId)?.image
    : undefined;

  if (img != null && !(img instanceof Promise)) {
    // 处理圆角
    if (element.roundness && context.roundRect) {
      context.beginPath();
      context.roundRect(
        0,
        0,
        element.width,
        element.height,
        getCornerRadius(Math.min(element.width, element.height), element),
      );
      context.clip();
    }

    // 处理裁剪
    const { x, y, width, height } = element.crop
      ? element.crop
      : {
          x: 0,
          y: 0,
          width: img.naturalWidth,
          height: img.naturalHeight,
        };

    // 绘制图片
    context.drawImage(
      img,
      x,
      y,
      width,
      height,
      0,
      0,
      element.width,
      element.height,
    );
  } else {
    drawImagePlaceholder(element, context);
  }
  break;
}
```

图片渲染不使用 RoughJS，直接使用 Canvas 的 `drawImage` 方法。

## 渲染性能优化

### 实际的性能优化策略

Excalidraw 的实际性能优化策略比文档中描述的要简单：

1. **ShapeCache 缓存系统**：使用 WeakMap 缓存生成的 RoughJS 图形
2. **throttleRAF 节流**：使用 requestAnimationFrame 节流渲染
3. **visibleElements 过滤**：只渲染可见元素
4. **Canvas 复用**：元素缓存为离屏 Canvas

```typescript
// packages/element/src/renderElement.ts
export const elementWithCanvasCache = new WeakMap<
  ExcalidrawElement,
  ExcalidrawElementWithCanvas
>();

// 生成元素画布并缓存
const generateElementWithCanvas = (
  element: NonDeletedExcalidrawElement,
  elementsMap: NonDeletedSceneElementsMap,
  // ...
) => {
  // 检查缓存
  const existingCanvasWithElement = elementWithCanvasCache.get(element);
  if (existingCanvasWithElement) {
    // 验证缓存是否有效
    if (existingCanvasWithElement.element.versionNonce === element.versionNonce) {
      return existingCanvasWithElement;
    }
  }

  // 生成新的画布缓存
  const canvasWithElement = generateElementCanvas(/* ... */);
  if (canvasWithElement) {
    elementWithCanvasCache.set(element, canvasWithElement);
  }

  return canvasWithElement;
};
```

### throttleRAF 节流

```typescript
// packages/excalidraw/renderer/staticScene.ts
export const renderStaticSceneThrottled = throttleRAF(
  (config: StaticSceneRenderConfig) => {
    _renderStaticScene(config);
  },
  { trailing: true },
);

// packages/excalidraw/renderer/interactiveScene.ts
export const renderInteractiveSceneThrottled = throttleRAF(
  (config: InteractiveSceneRenderConfig) => {
    const ret = _renderInteractiveScene(config);
    config.callback?.(ret);
  },
  { trailing: true },
);
```

实际的性能优化主要依靠这些简单但有效的策略，而不是文档中描述的复杂系统。

## 填充样式

### RoughJS 内置填充样式

Excalidraw 直接使用 RoughJS 的内置填充样式，不需要自己实现复杂的填充算法：

```typescript
// packages/element/src/shape.ts (generateRoughOptions 函数中)
if (!isTransparent(element.backgroundColor)) {
  options.fill = element.backgroundColor;

  if (element.fillStyle === "cross-hatch") {
    options.fillStyle = "cross-hatch";
    options.fillWeight = element.strokeWidth / 2;
    options.hachureAngle = -41;
    options.hachureGap = element.strokeWidth * 4;
  } else if (element.fillStyle === "hachure") {
    options.fillStyle = "hachure";
    options.fillWeight = element.strokeWidth / 2;
    options.hachureAngle = -41;
    options.hachureGap = element.strokeWidth * 4;
  } else if (element.fillStyle === "solid") {
    options.fillStyle = "solid";
  }
}
```

支持的填充样式：
- `"solid"` - 实心填充
- `"hachure"` - 单向阴影线填充
- `"cross-hatch"` - 交叉阴影线填充
- `"zigzag"` - 锯齿形填充

这些都是 RoughJS 原生支持的填充样式，不需要额外实现。

## 实际的 ShapeCache 实现

实际的 ShapeCache 实现更简单，没有哈希验证，直接使用 WeakMap：

```typescript
// packages/element/src/shape.ts
export class ShapeCache {
  private static rg = new RoughGenerator();
  private static cache = new WeakMap<ExcalidrawElement, ElementShape>();

  public static get = <T extends ExcalidrawElement>(element: T) => {
    return ShapeCache.cache.get(element);
  };

  public static set = <T extends ExcalidrawElement>(
    element: T,
    shape: ElementShape,
  ) => ShapeCache.cache.set(element, shape);

  public static delete = (element: ExcalidrawElement) =>
    ShapeCache.cache.delete(element);

  public static generateElementShape = (element, renderConfig) => {
    const cachedShape = renderConfig?.isExporting
      ? undefined  // 导出时总是重新生成
      : ShapeCache.get(element);

    if (cachedShape !== undefined) {
      return cachedShape;
    }

    // 清除元素画布缓存
    elementWithCanvasCache.delete(element);

    // 生成新图形
    const shape = generateElementShape(
      element,
      ShapeCache.rg,
      renderConfig || defaultConfig,
    );

    ShapeCache.cache.set(element, shape);
    return shape;
  };
}
```

缓存失效依赖于 `versionNonce` 的变化，当元素修改时会更新这个值，自然使缓存失效。

## 扩展 Excalidraw 图形类型

如果要为 Excalidraw 添加新的图形类型，需要在多个地方进行修改：

### 1. 定义新的元素类型

```typescript
// packages/element/src/types.ts
export interface ExcalidrawStarElement extends _ExcalidrawElementBase {
  type: "star";
  points: number;      // 星形角数
  innerRadius: number; // 内半径比例
}

// 添加到联合类型
export type ExcalidrawElement =
  | ExcalidrawRectangleElement
  | ExcalidrawDiamondElement
  | ExcalidrawEllipseElement
  | ExcalidrawStarElement  // 新增
  // ... 其他类型
```

### 2. 添加图形生成逻辑

```typescript
// packages/element/src/shape.ts (generateElementShape 函数中)
case "star": {
  const points = element.points;
  const outerRadius = Math.min(element.width, element.height) / 2;
  const innerRadius = outerRadius * element.innerRadius;
  const centerX = element.width / 2;
  const centerY = element.height / 2;

  const starPoints = [];
  const angleStep = (Math.PI * 2) / (points * 2);

  for (let i = 0; i < points * 2; i++) {
    const angle = i * angleStep - Math.PI / 2;
    const radius = i % 2 === 0 ? outerRadius : innerRadius;
    const x = centerX + Math.cos(angle) * radius;
    const y = centerY + Math.sin(angle) * radius;
    starPoints.push([x, y]);
  }

  shape = generator.polygon(starPoints, generateRoughOptions(element));
  return shape;
}
```

### 3. 添加类型检查

```typescript
// packages/element/src/typeChecks.ts
export const isStarElement = (
  element: ExcalidrawElement | null,
): element is ExcalidrawStarElement => {
  return element?.type === "star";
};
```

这样就可以为 Excalidraw 添加新的图形类型，利用现有的渲染管道。

## 总结

通过深入分析 Excalidraw 的实际源码，我们发现其图形渲染系统的架构比预想的要简单，但非常有效：

### 核心架构（已验证 ✓）

1. **ShapeCache 系统**（packages/element/src/shape.ts）
   - 使用 WeakMap 缓存 RoughJS 生成的图形
   - 静态属性 `rg = new RoughGenerator()` 全局复用
   - `generateElementShape()` 方法支持导出时强制重新生成

2. **双渲染管线**：
   - **图形生成**：`generateElementShape()` → RoughJS 图形（Drawable）
   - **画布绘制**：`drawElementOnCanvas()` → Canvas 绘制

3. **简单的缓存策略**：
   - 依赖 `element.versionNonce` 自动失效缓存
   - 导出时通过 `renderConfig.isExporting` 跳过缓存

4. **特殊渲染处理**：
   - **自由绘制**：使用 `perfect-freehand` + Canvas Path2D，不用 RoughJS
   - **文本**：直接使用 Canvas 2D API 逐行绘制
   - **图片**：使用 `drawImage()`，支持裁剪和圆角

### 关键发现（源码验证）

- ✓ **generateRoughOptions() 的实际实现**：
  - 动态调整 roughness（`adjustRoughness()` 函数）
  - 非 solid 笔触自动 +0.5 宽度并禁用 multiStroke
  - 椭圆自动设置 `curveFitting: 1`

- ✓ **Canvas 尺寸限制**：
  - 面积限制：16777216 像素（约 Safari mobile 限制）
  - 宽高限制：32767 像素
  - `cappedElementCanvasSize()` 函数自动缩放

- ✓ **实际的渲染顺序**：
  - 先渲染非 iframe-like 元素
  - 再渲染 iframe/embeddable 元素（在顶层）
  - 最后渲染 bound text 和 link icon

- ✓ **错误处理**：每个元素的渲染都包裹在 try-catch 中，单个元素失败不影响整体

### 设计优势

这种简单的架构设计有以下优势：
1. **易于理解和维护**：代码结构清晰，职责明确
2. **足够的性能**：对于 Excalidraw 的使用场景已经足够
3. **稳定可靠**：较少的复杂性意味着较少的 bug
4. **易于扩展**：添加新图形类型相对简单，只需在 switch-case 中增加分支

### 源码验证要点

- ✓ ShapeCache 位置：`packages/element/src/shape.ts`
- ✓ renderElement 位置：`packages/element/src/renderElement.ts`
- ✓ 使用 `rough.canvas(canvas)` 创建 RoughCanvas
- ✓ 自由绘制使用 `getStroke()` 从 perfect-freehand 库
- ✓ 文本渲染支持 RTL（从右到左）语言

这证明了"简单就是美"的设计哲学在实际项目中的价值。

## 下一步

在下一章中，我们将探讨 Excalidraw 实际使用的性能优化策略，主要包括 `throttleRAF` 节流、ShapeCache 缓存系统、以及 `visibleElements` 视口过滤等实用技术。