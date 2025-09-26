# Chapter 3.2: 图形绘制与手绘风格实现

## 概述

Excalidraw 最具特色的功能之一就是其独特的手绘风格。这种风格不仅让图表看起来更加友好和非正式，还能有效降低观众对"完成度"的预期，使其更适合用于草图和头脑风暴。本章将深入探讨 Excalidraw 如何通过 RoughJS 实现这种手绘效果，以及各种图形元素的具体渲染策略。

## RoughJS 集成原理

### 什么是 RoughJS？

RoughJS 是一个轻量级的图形库，专门用于创建手绘风格的图形。它的核心原理是：
1. 将完美的几何图形转换为不规则的手绘路径
2. 通过添加抖动、多重描边等技术模拟手绘效果
3. 保持图形的可识别性同时增加自然的不完美感

### Excalidraw 中的 RoughJS 初始化

```typescript
// packages/excalidraw/renderer/renderElement.ts
import rough from "roughjs/bin/rough";

// 创建 RoughCanvas 实例
export const getRoughCanvas = (canvas: HTMLCanvasElement) => {
  return rough.canvas(canvas, {
    // RoughJS 配置选项
    options: {
      // 异步绘制，提高性能
      async: false,
      // 自定义随机数生成器，确保可重现性
      randomizer: seededRandomizer,
    },
  });
};

// 种子随机数生成器，确保相同的元素每次渲染结果一致
class SeededRandomizer {
  constructor(private seed: number) {}

  next() {
    // 使用 LCG 算法生成伪随机数
    this.seed = (this.seed * 9301 + 49297) % 233280;
    return this.seed / 233280;
  }
}
```

### RoughJS 渲染选项详解

```typescript
interface RoughOptions {
  // 粗糙度：控制线条的抖动程度（0 = 完美直线）
  roughness: number;

  // 线条宽度
  strokeWidth: number;

  // 填充样式
  fillStyle: 'hachure' | 'solid' | 'zigzag' | 'cross-hatch' | 'dots' | 'dashed' | 'zigzag-line';

  // 填充线条密度
  fillWeight: number;

  // 填充线条间隙
  hachureGap: number;

  // 填充线条角度
  hachureAngle: number;

  // 弯曲度：控制曲线的弯曲程度
  bowing: number;

  // 描边次数：多次描边创造素描效果
  stroke: string;

  // 填充颜色
  fill: string;

  // 种子值：用于生成可重现的随机效果
  seed: number;
}

// Excalidraw 元素到 RoughJS 选项的映射
function elementToRoughOptions(element: ExcalidrawElement): RoughOptions {
  return {
    roughness: element.roughness,
    strokeWidth: element.strokeWidth,
    stroke: element.strokeColor,
    fill: element.backgroundColor,
    fillStyle: element.fillStyle,
    fillWeight: element.strokeWidth / 2,
    hachureGap: element.strokeWidth * 4,
    hachureAngle: 60,
    bowing: element.roughness,
    seed: element.seed,
  };
}
```

## 图形元素渲染实现

### 矩形渲染

```typescript
// packages/element/src/shape.ts
export const generateRectangleShape = (
  element: ExcalidrawRectangleElement,
  generator: RoughGenerator,
): Drawable => {
  const { x, y, width, height, strokeSharpness } = element;

  // 根据锐度选择渲染方式
  if (strokeSharpness === "sharp") {
    // 锐利边角：使用标准矩形
    return generator.rectangle(x, y, width, height, {
      ...elementToRoughOptions(element),
      // 禁用圆角
      disableMultiStroke: true,
    });
  } else {
    // 圆润边角：使用带圆角的路径
    const radius = getCornerRadius(Math.min(width, height), element);

    return generator.path(
      getRoundedRectanglePath(x, y, width, height, radius),
      elementToRoughOptions(element)
    );
  }
};

// 生成圆角矩形路径
function getRoundedRectanglePath(
  x: number,
  y: number,
  width: number,
  height: number,
  radius: number,
): string {
  // 使用 SVG 路径语法
  return `
    M ${x + radius} ${y}
    L ${x + width - radius} ${y}
    Q ${x + width} ${y} ${x + width} ${y + radius}
    L ${x + width} ${y + height - radius}
    Q ${x + width} ${y + height} ${x + width - radius} ${y + height}
    L ${x + radius} ${y + height}
    Q ${x} ${y + height} ${x} ${y + height - radius}
    L ${x} ${y + radius}
    Q ${x} ${y} ${x + radius} ${y}
    Z
  `;
}
```

### 椭圆渲染

```typescript
export const generateEllipseShape = (
  element: ExcalidrawEllipseElement,
  generator: RoughGenerator,
): Drawable => {
  const { x, y, width, height } = element;

  // 椭圆中心和半径
  const cx = x + width / 2;
  const cy = y + height / 2;
  const rx = width / 2;
  const ry = height / 2;

  return generator.ellipse(cx, cy, width, height, {
    ...elementToRoughOptions(element),
    // 椭圆特定选项
    curveFitting: 0.95,  // 曲线拟合精度
    curveStepCount: 9,   // 曲线分段数
  });
};

// 圆形作为椭圆的特例
export const generateCircleShape = (
  element: ExcalidrawEllipseElement,
  generator: RoughGenerator,
): Drawable => {
  const size = Math.min(element.width, element.height);
  const adjustedElement = {
    ...element,
    width: size,
    height: size,
  };

  return generateEllipseShape(adjustedElement, generator);
};
```

### 菱形渲染

```typescript
export const generateDiamondShape = (
  element: ExcalidrawDiamondElement,
  generator: RoughGenerator,
): Drawable => {
  const { x, y, width, height } = element;

  // 菱形的四个顶点
  const points: Point[] = [
    [x + width / 2, y],                // 顶部
    [x + width, y + height / 2],       // 右侧
    [x + width / 2, y + height],        // 底部
    [x, y + height / 2],                // 左侧
  ];

  // 使用多边形生成器
  return generator.polygon(points, elementToRoughOptions(element));
};
```

### 线条和箭头渲染

```typescript
// 线条渲染
export const generateLineShape = (
  element: ExcalidrawLinearElement,
  generator: RoughGenerator,
): Drawable => {
  const { points } = element;

  if (points.length < 2) {
    return { sets: [] };
  }

  // 根据点的数量选择渲染方式
  if (element.strokeSharpness === "sharp") {
    // 锐利线条：使用直线段
    return generator.linearPath(
      points.map(p => [p.x, p.y] as Point),
      elementToRoughOptions(element)
    );
  } else {
    // 平滑线条：使用曲线
    return generator.curve(
      points.map(p => [p.x, p.y] as Point),
      elementToRoughOptions(element)
    );
  }
};

// 箭头渲染
export const generateArrowShape = (
  element: ExcalidrawArrowElement,
  generator: RoughGenerator,
): ArrowShape => {
  const lineShape = generateLineShape(element, generator);
  const arrowheads = generateArrowheads(element);

  return {
    line: lineShape,
    arrowheads,
  };
};

// 生成箭头头部
function generateArrowheads(element: ExcalidrawArrowElement): Arrowhead[] {
  const arrowheads: Arrowhead[] = [];
  const { points, startArrowhead, endArrowhead } = element;

  if (startArrowhead && startArrowhead !== "none") {
    arrowheads.push(
      createArrowhead(
        points[0],
        points[1],
        startArrowhead,
        element
      )
    );
  }

  if (endArrowhead && endArrowhead !== "none") {
    const lastPoint = points[points.length - 1];
    const secondLastPoint = points[points.length - 2];

    arrowheads.push(
      createArrowhead(
        lastPoint,
        secondLastPoint,
        endArrowhead,
        element
      )
    );
  }

  return arrowheads;
}

// 创建箭头头部形状
function createArrowhead(
  tip: Point,
  base: Point,
  type: ArrowheadType,
  element: ExcalidrawArrowElement
): Arrowhead {
  const angle = Math.atan2(tip.y - base.y, tip.x - base.x);
  const size = getArrowheadSize(element);

  switch (type) {
    case "arrow":
      // 标准箭头
      return {
        type: "arrow",
        points: [
          rotatePoint([-size, -size / 2], angle, tip),
          tip,
          rotatePoint([-size, size / 2], angle, tip),
        ],
      };

    case "dot":
      // 圆点箭头
      return {
        type: "dot",
        center: tip,
        radius: size / 2,
      };

    case "bar":
      // 竖线箭头
      return {
        type: "bar",
        points: [
          rotatePoint([0, -size / 2], angle, tip),
          rotatePoint([0, size / 2], angle, tip),
        ],
      };

    default:
      return { type: "none" };
  }
}
```

## 自由绘制实现

### FreeHand 绘制算法

```typescript
// packages/element/src/freehand.ts
export const generateFreeDrawShape = (
  element: ExcalidrawFreeDrawElement,
): FreeDrawShape => {
  const { points, pressures } = element;

  // 使用 perfect-freehand 库生成平滑路径
  const strokePoints = getStroke(points, {
    size: element.strokeWidth,
    thinning: 0.5,
    smoothing: 0.5,
    streamline: 0.5,
    simulatePressure: pressures === null,
    pressures: pressures || undefined,
  });

  // 将点集转换为 SVG 路径
  const pathData = getSvgPathFromStroke(strokePoints);

  return {
    path: pathData,
    fillStyle: "solid",
  };
};

// 使用 Catmull-Rom 样条曲线平滑路径
function smoothPath(points: Point[]): Point[] {
  if (points.length < 3) {
    return points;
  }

  const smoothed: Point[] = [points[0]];

  for (let i = 1; i < points.length - 1; i++) {
    const prev = points[i - 1];
    const curr = points[i];
    const next = points[i + 1];

    // Catmull-Rom 插值
    for (let t = 0; t <= 1; t += 0.1) {
      const t2 = t * t;
      const t3 = t2 * t;

      const x =
        0.5 * (
          2 * curr.x +
          (-prev.x + next.x) * t +
          (2 * prev.x - 5 * curr.x + 4 * next.x - next.x) * t2 +
          (-prev.x + 3 * curr.x - 3 * next.x + next.x) * t3
        );

      const y =
        0.5 * (
          2 * curr.y +
          (-prev.y + next.y) * t +
          (2 * prev.y - 5 * curr.y + 4 * next.y - next.y) * t2 +
          (-prev.y + 3 * curr.y - 3 * next.y + next.y) * t3
        );

      smoothed.push({ x, y });
    }
  }

  smoothed.push(points[points.length - 1]);
  return smoothed;
}
```

## 文本渲染系统

### 文本测量与布局

```typescript
// packages/element/src/textElement.ts
export const measureText = (
  text: string,
  font: FontString,
  maxWidth?: number
): TextMetrics => {
  // 获取或创建测量用的 Canvas
  const canvas = document.createElement("canvas");
  const context = canvas.getContext("2d")!;

  context.font = font;

  // 分行处理
  const lines = text.split("\n");
  let width = 0;
  let height = 0;

  const processedLines = lines.map(line => {
    if (maxWidth) {
      // 自动换行
      return wrapText(context, line, maxWidth);
    }
    return [line];
  }).flat();

  // 测量每行
  processedLines.forEach(line => {
    const metrics = context.measureText(line);
    width = Math.max(width, metrics.width);
    height += getLineHeight(font);
  });

  return {
    width,
    height,
    lines: processedLines,
  };
};

// 文本自动换行
function wrapText(
  context: CanvasRenderingContext2D,
  text: string,
  maxWidth: number
): string[] {
  const words = text.split(" ");
  const lines: string[] = [];
  let currentLine = "";

  words.forEach(word => {
    const testLine = currentLine ? `${currentLine} ${word}` : word;
    const metrics = context.measureText(testLine);

    if (metrics.width > maxWidth && currentLine) {
      lines.push(currentLine);
      currentLine = word;
    } else {
      currentLine = testLine;
    }
  });

  if (currentLine) {
    lines.push(currentLine);
  }

  return lines;
}
```

### 文本渲染实现

```typescript
export const renderTextElement = (
  element: ExcalidrawTextElement,
  context: CanvasRenderingContext2D,
  renderConfig: RenderConfig
) => {
  const { x, y, width, height, text, fontSize, fontFamily, textAlign } = element;

  // 设置文本样式
  context.font = `${fontSize}px ${fontFamily}`;
  context.fillStyle = element.strokeColor;
  context.textAlign = textAlign;
  context.textBaseline = "top";

  // 获取文本度量
  const metrics = measureText(text, context.font, width);

  // 计算文本位置
  let offsetX = x;
  if (textAlign === "center") {
    offsetX = x + width / 2;
  } else if (textAlign === "right") {
    offsetX = x + width;
  }

  // 渲染每行文本
  metrics.lines.forEach((line, index) => {
    const lineY = y + index * getLineHeight(context.font);

    // 文本阴影（可选）
    if (renderConfig.enableShadow) {
      context.shadowColor = "rgba(0, 0, 0, 0.1)";
      context.shadowBlur = 2;
      context.shadowOffsetX = 1;
      context.shadowOffsetY = 1;
    }

    context.fillText(line, offsetX, lineY);
  });

  // 渲染文本装饰（下划线、删除线等）
  if (element.textDecoration) {
    renderTextDecoration(context, element, metrics);
  }
};

// 文本装饰渲染
function renderTextDecoration(
  context: CanvasRenderingContext2D,
  element: ExcalidrawTextElement,
  metrics: TextMetrics
) {
  const { textDecoration, x, y, strokeColor, strokeWidth } = element;

  context.strokeStyle = strokeColor;
  context.lineWidth = strokeWidth;

  metrics.lines.forEach((line, index) => {
    const lineY = y + index * getLineHeight(context.font);
    const lineWidth = context.measureText(line).width;

    switch (textDecoration) {
      case "underline":
        context.beginPath();
        context.moveTo(x, lineY + metrics.height * 0.9);
        context.lineTo(x + lineWidth, lineY + metrics.height * 0.9);
        context.stroke();
        break;

      case "line-through":
        context.beginPath();
        context.moveTo(x, lineY + metrics.height * 0.5);
        context.lineTo(x + lineWidth, lineY + metrics.height * 0.5);
        context.stroke();
        break;
    }
  });
}
```

## 图片渲染处理

### 图片加载与缓存

```typescript
// packages/excalidraw/data/image.ts
export class ImageCache {
  private static cache = new Map<string, HTMLImageElement | Promise<HTMLImageElement>>();

  static async loadImage(src: string): Promise<HTMLImageElement> {
    // 检查缓存
    const cached = this.cache.get(src);
    if (cached) {
      return cached instanceof Promise ? await cached : cached;
    }

    // 创建加载 Promise
    const loadPromise = new Promise<HTMLImageElement>((resolve, reject) => {
      const img = new Image();

      img.onload = () => {
        // 替换 Promise 为实际图片
        this.cache.set(src, img);
        resolve(img);
      };

      img.onerror = (error) => {
        this.cache.delete(src);
        reject(error);
      };

      img.src = src;
    });

    // 缓存 Promise
    this.cache.set(src, loadPromise);

    return loadPromise;
  }

  // 预加载图片
  static preload(sources: string[]) {
    return Promise.all(sources.map(src => this.loadImage(src)));
  }

  // 清理缓存
  static clear(src?: string) {
    if (src) {
      this.cache.delete(src);
    } else {
      this.cache.clear();
    }
  }
}
```

### 图片渲染与变换

```typescript
export const renderImageElement = async (
  element: ExcalidrawImageElement,
  context: CanvasRenderingContext2D,
  renderConfig: RenderConfig
) => {
  const { x, y, width, height, angle, scale, imageUrl } = element;

  try {
    // 加载图片
    const image = await ImageCache.loadImage(imageUrl);

    context.save();

    // 应用变换
    if (angle) {
      const centerX = x + width / 2;
      const centerY = y + height / 2;
      context.translate(centerX, centerY);
      context.rotate(angle);
      context.translate(-centerX, -centerY);
    }

    // 应用缩放
    if (scale && scale !== 1) {
      context.scale(scale, scale);
    }

    // 设置透明度
    context.globalAlpha = element.opacity;

    // 图片圆角处理
    if (element.borderRadius) {
      context.beginPath();
      context.roundRect(x, y, width, height, element.borderRadius);
      context.clip();
    }

    // 渲染图片
    context.drawImage(image, x, y, width, height);

    // 渲染边框
    if (element.strokeWidth > 0) {
      context.strokeStyle = element.strokeColor;
      context.lineWidth = element.strokeWidth;
      context.strokeRect(x, y, width, height);
    }

    context.restore();
  } catch (error) {
    // 渲染占位符
    renderImagePlaceholder(context, element);
  }
};

// 图片加载失败时的占位符
function renderImagePlaceholder(
  context: CanvasRenderingContext2D,
  element: ExcalidrawImageElement
) {
  const { x, y, width, height } = element;

  // 绘制虚线边框
  context.save();
  context.strokeStyle = "#999";
  context.lineWidth = 2;
  context.setLineDash([5, 5]);
  context.strokeRect(x, y, width, height);

  // 绘制图标
  context.fillStyle = "#999";
  context.font = "20px sans-serif";
  context.textAlign = "center";
  context.textBaseline = "middle";
  context.fillText("🖼️", x + width / 2, y + height / 2);

  context.restore();
}
```

## 填充样式实现

### Hachure（阴影线）填充

```typescript
// Hachure 填充算法
export const generateHachureFill = (
  shape: Shape,
  options: HachureOptions
): Path[] => {
  const { hachureGap, hachureAngle, strokeWidth } = options;
  const paths: Path[] = [];

  // 获取形状边界
  const bounds = getShapeBounds(shape);

  // 计算填充线条
  const angleRad = (hachureAngle * Math.PI) / 180;
  const gap = hachureGap || strokeWidth * 4;

  // 生成平行线
  const lines = generateParallelLines(bounds, angleRad, gap);

  // 裁剪到形状内部
  lines.forEach(line => {
    const clipped = clipLineToShape(line, shape);
    if (clipped) {
      // 添加手绘效果
      const roughLine = addRoughness(clipped, options.roughness);
      paths.push(roughLine);
    }
  });

  return paths;
};

// 生成平行线
function generateParallelLines(
  bounds: Bounds,
  angle: number,
  gap: number
): Line[] {
  const lines: Line[] = [];
  const { x, y, width, height } = bounds;

  // 计算对角线长度
  const diagonal = Math.sqrt(width * width + height * height);

  // 根据角度调整起始位置
  const cos = Math.cos(angle);
  const sin = Math.sin(angle);

  for (let d = -diagonal; d < diagonal; d += gap) {
    // 计算线的两个端点
    const x1 = x + width / 2 + d * cos - diagonal * sin;
    const y1 = y + height / 2 + d * sin + diagonal * cos;
    const x2 = x + width / 2 + d * cos + diagonal * sin;
    const y2 = y + height / 2 + d * sin - diagonal * cos;

    lines.push({ start: { x: x1, y: y1 }, end: { x: x2, y: y2 } });
  }

  return lines;
}
```

### 其他填充样式

```typescript
// Cross-Hatch（交叉阴影线）填充
export const generateCrossHatchFill = (
  shape: Shape,
  options: HachureOptions
): Path[] => {
  // 生成两个方向的阴影线
  const paths1 = generateHachureFill(shape, options);
  const paths2 = generateHachureFill(shape, {
    ...options,
    hachureAngle: options.hachureAngle + 90,
  });

  return [...paths1, ...paths2];
};

// Solid（实心）填充
export const generateSolidFill = (
  shape: Shape,
  options: FillOptions
): Path => {
  // 直接返回形状路径
  return shape.toPath();
};

// Dots（点状）填充
export const generateDotsFill = (
  shape: Shape,
  options: DotsOptions
): Dot[] => {
  const dots: Dot[] = [];
  const { dotSize, dotGap } = options;
  const bounds = getShapeBounds(shape);

  for (let x = bounds.x; x < bounds.x + bounds.width; x += dotGap) {
    for (let y = bounds.y; y < bounds.y + bounds.height; y += dotGap) {
      if (isPointInShape({ x, y }, shape)) {
        dots.push({
          x: x + Math.random() * 2 - 1,  // 添加随机偏移
          y: y + Math.random() * 2 - 1,
          radius: dotSize * (0.8 + Math.random() * 0.4),  // 随机大小变化
        });
      }
    }
  }

  return dots;
};
```

## ShapeCache 缓存机制

### 缓存架构

```typescript
// packages/element/src/shape.ts
export class ShapeCache {
  // 使用 WeakMap 自动管理内存
  private static cache = new WeakMap<
    ExcalidrawElement,
    {
      shape: ElementShape;
      hash: string;
    }
  >();

  // 生成元素哈希值
  private static generateHash(element: ExcalidrawElement): string {
    // 包含所有影响渲染的属性
    return JSON.stringify({
      type: element.type,
      width: element.width,
      height: element.height,
      roughness: element.roughness,
      strokeWidth: element.strokeWidth,
      fillStyle: element.fillStyle,
      seed: element.seed,
      versionNonce: element.versionNonce,
    });
  }

  // 获取缓存的形状
  public static get(element: ExcalidrawElement): ElementShape | null {
    const cached = this.cache.get(element);

    if (cached) {
      // 验证缓存是否有效
      const currentHash = this.generateHash(element);
      if (cached.hash === currentHash) {
        return cached.shape;
      }
    }

    return null;
  }

  // 设置缓存
  public static set(element: ExcalidrawElement, shape: ElementShape) {
    this.cache.set(element, {
      shape,
      hash: this.generateHash(element),
    });
  }

  // 生成并缓存形状
  public static generateElementShape(
    element: ExcalidrawElement,
    renderConfig: RenderConfig
  ): ElementShape {
    // 尝试从缓存获取
    let shape = this.get(element);

    if (!shape) {
      // 生成新形状
      const generator = rough.generator();

      switch (element.type) {
        case "rectangle":
          shape = generateRectangleShape(element, generator);
          break;
        case "ellipse":
          shape = generateEllipseShape(element, generator);
          break;
        case "diamond":
          shape = generateDiamondShape(element, generator);
          break;
        // ... 其他类型
      }

      // 缓存形状
      if (shape) {
        this.set(element, shape);
      }
    }

    return shape;
  }

  // 清理特定元素的缓存
  public static delete(element: ExcalidrawElement) {
    this.cache.delete(element);
  }
}
```

## 实战示例：实现自定义图形渲染

### 创建星形图形

```javascript
class StarShape {
  constructor(cx, cy, outerRadius, innerRadius, points) {
    this.cx = cx;
    this.cy = cy;
    this.outerRadius = outerRadius;
    this.innerRadius = innerRadius;
    this.points = points;
  }

  // 生成星形路径
  generatePath() {
    const angleStep = (Math.PI * 2) / (this.points * 2);
    const path = [];

    for (let i = 0; i < this.points * 2; i++) {
      const angle = i * angleStep - Math.PI / 2;
      const radius = i % 2 === 0 ? this.outerRadius : this.innerRadius;

      const x = this.cx + Math.cos(angle) * radius;
      const y = this.cy + Math.sin(angle) * radius;

      if (i === 0) {
        path.push(`M ${x} ${y}`);
      } else {
        path.push(`L ${x} ${y}`);
      }
    }

    path.push('Z');
    return path.join(' ');
  }

  // 渲染到 Canvas
  render(ctx, options = {}) {
    const path = new Path2D(this.generatePath());

    // 填充
    if (options.fill) {
      ctx.fillStyle = options.fill;
      ctx.fill(path);
    }

    // 描边
    if (options.stroke) {
      ctx.strokeStyle = options.stroke;
      ctx.lineWidth = options.strokeWidth || 2;
      ctx.stroke(path);
    }
  }

  // 添加手绘效果
  renderRough(rc, options = {}) {
    const path = this.generatePath();

    return rc.path(path, {
      roughness: options.roughness || 1,
      bowing: options.bowing || 1,
      stroke: options.stroke || '#000',
      fill: options.fill || 'transparent',
      fillStyle: options.fillStyle || 'hachure',
      strokeWidth: options.strokeWidth || 2,
      seed: options.seed || Math.random() * 100000,
    });
  }
}

// 使用示例
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const rc = rough.canvas(canvas);

// 创建星形
const star = new StarShape(200, 200, 80, 40, 5);

// 标准渲染
star.render(ctx, {
  fill: '#FFD700',
  stroke: '#FF6347',
  strokeWidth: 3,
});

// 手绘风格渲染
star.renderRough(rc, {
  roughness: 2,
  bowing: 0.5,
  stroke: '#333',
  fill: '#FFD700',
  fillStyle: 'cross-hatch',
});
```

### 实现渐变填充

```javascript
class GradientRenderer {
  // 线性渐变
  createLinearGradient(ctx, x1, y1, x2, y2, colors) {
    const gradient = ctx.createLinearGradient(x1, y1, x2, y2);

    colors.forEach((color, index) => {
      const offset = index / (colors.length - 1);
      gradient.addColorStop(offset, color);
    });

    return gradient;
  }

  // 径向渐变
  createRadialGradient(ctx, cx, cy, r, colors) {
    const gradient = ctx.createRadialGradient(cx, cy, 0, cx, cy, r);

    colors.forEach((color, index) => {
      const offset = index / (colors.length - 1);
      gradient.addColorStop(offset, color);
    });

    return gradient;
  }

  // 渲染渐变矩形
  renderGradientRect(ctx, x, y, width, height, gradientType, colors) {
    let gradient;

    if (gradientType === 'linear') {
      gradient = this.createLinearGradient(ctx, x, y, x + width, y + height, colors);
    } else {
      const cx = x + width / 2;
      const cy = y + height / 2;
      const r = Math.max(width, height) / 2;
      gradient = this.createRadialGradient(ctx, cx, cy, r, colors);
    }

    ctx.fillStyle = gradient;
    ctx.fillRect(x, y, width, height);
  }

  // 手绘风格的渐变效果（通过密度变化模拟）
  renderRoughGradient(rc, x, y, width, height, startColor, endColor) {
    const steps = 10;
    const stepHeight = height / steps;

    for (let i = 0; i < steps; i++) {
      const ratio = i / steps;
      const opacity = 1 - ratio * 0.7;  // 渐变透明度
      const gap = 4 + ratio * 8;        // 渐变密度

      rc.rectangle(x, y + i * stepHeight, width, stepHeight, {
        roughness: 1,
        fill: startColor,
        fillStyle: 'hachure',
        hachureGap: gap,
        stroke: 'none',
        fillWeight: 1,
      });
    }
  }
}
```

## 思考题

1. **手绘风格的性能影响如何评估？** RoughJS 生成的复杂路径对渲染性能有何影响？

2. **如何实现自定义填充样式？** 设计一个新的填充样式（如波浪纹、网格等）。

3. **文本渲染的国际化问题如何解决？** 处理不同语言、字体和书写方向。

4. **图片渲染的内存管理策略？** 大量图片时如何避免内存溢出？

5. **如何支持矢量图形（SVG）的导入和渲染？** 将 SVG 转换为 Excalidraw 元素。

## 实践练习

### 练习 1：实现自定义图形
创建一个新的图形类型（如心形、多边形、齿轮等）：
- 定义图形数据结构
- 实现渲染函数
- 添加手绘风格支持

### 练习 2：优化文本渲染
改进文本渲染系统：
- 实现文本选中高亮
- 添加富文本支持
- 优化大量文本的渲染性能

### 练习 3：图片滤镜效果
为图片元素添加滤镜：
- 模糊、锐化、灰度等基本滤镜
- 保持手绘风格一致性
- 考虑性能影响

## 总结

Excalidraw 的图形渲染系统展示了如何在 Web 平台上实现独特的视觉风格：

1. **RoughJS 集成**：巧妙利用第三方库实现手绘效果，同时保持可控性
2. **多样化图形支持**：从基础形状到自由绘制，提供丰富的表达方式
3. **高效缓存策略**：通过 ShapeCache 避免重复计算，提升渲染性能
4. **灵活的填充系统**：多种填充样式满足不同的视觉需求
5. **完善的文本和图片处理**：提供专业级的文本排版和图片渲染能力

这些技术不仅适用于画板应用，也为其他需要独特视觉风格的 Web 应用提供了参考。

## 下一步

在下一章中，我们将深入探讨 Excalidraw 的渲染优化策略，包括分层渲染、虚拟化、脏矩形等高级技术，以及如何在保持 60fps 的同时处理数千个元素的复杂场景。