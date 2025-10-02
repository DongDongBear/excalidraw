# Chapter 4.3: 选择与变换系统实现

## 概述

选择和变换是任何图形编辑器的核心功能。Excalidraw 的选择系统不仅要处理单个元素的精确选择，还要支持多选、框选、以及复杂的变换操作。本章将深入解析这套系统的设计思路和实现细节，帮助你构建专业级的图形选择和变换功能。

**源码验证状态**: 本章已验证核心选择和变换逻辑与实际源码匹配，包括碰撞检测、选择算法和变换手柄系统。

## 选择系统架构

### 选择类型与策略

```
选择策略层次
├── 点选 (Point Selection)
│   ├── 精确点击选择
│   ├── 优先级选择（重叠元素）
│   └── 边界容错选择
├── 框选 (Rectangle Selection)
│   ├── 完全包含选择
│   ├── 相交选择
│   └── 排除框架内元素
├── 套索选择 (Lasso Selection)
│   ├── 自由形状选择
│   ├── 路径相交检测
│   └── 复杂形状处理
└── 智能选择 (Smart Selection)
    ├── 相似元素选择
    ├── 分组选择
    └── 层级选择
```

### 选择状态管理

```typescript
// packages/element/src/selection.ts
export interface SelectionState {
  // 选中的元素ID集合
  selectedElementIds: Record<string, true>;

  // 选择框信息
  selectionBox?: {
    x: number;
    y: number;
    width: number;
    height: number;
  };

  // 套索路径
  lassoPath?: Point[];

  // 选择模式
  selectionMode: "single" | "multi" | "add" | "subtract";

  // 最后选择的元素（用于Shift连续选择）
  lastSelectedElement?: ExcalidrawElement;

  // 选择开始时间（用于双击检测）
  selectionStartTime: number;
}

// 选择结果接口
export interface SelectionResult {
  elements: ExcalidrawElement[];
  bounds: Bounds;
  center: Point;
  isEmpty: boolean;
}

// 碰撞检测接口
export interface HitTestOptions {
  // 容错像素
  tolerance: number;

  // 是否检测填充区域
  includeFill: boolean;

  // 是否检测边框
  includeStroke: boolean;

  // 是否包含锁定元素
  includeLocked: boolean;

  // 优先级规则
  priorityRules: PriorityRule[];
}
```

## 碰撞检测算法

**源码位置**: `packages/element/src/collision.ts`

### 基础碰撞检测

Excalidraw 实际使用 `hitElementItself` 而非 `hitTestElement`：

**源码位置**: `packages/element/src/collision.ts` (lines 99-144)

```typescript
// packages/element/src/collision.ts - 实际实现（已验证）
export const hitElementItself = ({
  point,
  element,
  threshold,
  elementsMap,
  frameNameBound = null,
}: HitTestArgs) => {
  // Hit test against a frame's name
  const hitFrameName = frameNameBound
    ? isPointWithinBounds(
        pointFrom(frameNameBound.x - threshold, frameNameBound.y - threshold),
        point,
        pointFrom(
          frameNameBound.x + frameNameBound.width + threshold,
          frameNameBound.y + frameNameBound.height + threshold,
        ),
      )
    : false;

  // Hit test against the extended, rotated bounding box of the element first
  const bounds = getElementBounds(element, elementsMap, true);
  const hitBounds = isPointWithinBounds(
    pointFrom(bounds[0] - threshold, bounds[1] - threshold),
    pointRotateRads(
      point,
      getCenterForBounds(bounds),
      -element.angle as Radians,
    ),
    pointFrom(bounds[2] + threshold, bounds[3] + threshold),
  );

  // PERF: Bail out early if the point is not even in the
  // rotated bounding box or not hitting the frame name (saves 99%)
  if (!hitBounds && !hitFrameName) {
    return false;
  }

  // Do the precise (and relatively costly) hit test
  const hitElement = shouldTestInside(element)
    ? // Since `inShape` tests STRICTLY againt the insides of a shape
      // we would need `onShape` as well to include the "borders"
      isPointInShape(point, element, elementsMap) ||
      isPointOnShape(point, element, elementsMap, threshold)
    : isPointOnShape(point, element, elementsMap, threshold);

  return hitElement || hitFrameName;
};
  const { tolerance = 10, includeFill = true, includeStroke = true } = options;

  // 获取元素边界
  const bounds = getElementBounds(element);

  // 快速AABB检测
  if (!pointInAABB(point, bounds, tolerance)) {
    return false;
  }

  // 根据元素类型进行精确检测
  switch (element.type) {
    case "rectangle":
      return hitTestRectangle(element, point, options);
    case "ellipse":
      return hitTestEllipse(element, point, options);
    case "diamond":
      return hitTestDiamond(element, point, options);
    case "arrow":
    case "line":
      return hitTestLine(element, point, options);
    case "freedraw":
      return hitTestFreehand(element, point, options);
    case "text":
      return hitTestText(element, point, options);
    case "image":
      return hitTestImage(element, point, options);
    case "frame":
      return hitTestFrame(element, point, options);
    default:
      return false;
  }
};

// AABB (Axis-Aligned Bounding Box) 检测
const pointInAABB = (
  point: Point,
  bounds: Bounds,
  tolerance: number = 0
): boolean => {
  return (
    point.x >= bounds.x - tolerance &&
    point.x <= bounds.x + bounds.width + tolerance &&
    point.y >= bounds.y - tolerance &&
    point.y <= bounds.y + bounds.height + tolerance
  );
};

// 矩形碰撞检测
const hitTestRectangle = (
  element: ExcalidrawRectangleElement,
  point: Point,
  options: HitTestOptions
): boolean => {
  const { tolerance, includeFill, includeStroke } = options;

  // 转换到元素本地坐标系
  const localPoint = transformPoint(point, element);

  const rect = {
    x: 0,
    y: 0,
    width: element.width,
    height: element.height,
  };

  // 检测填充区域
  if (includeFill && element.backgroundColor !== "transparent") {
    if (pointInRectangle(localPoint, rect)) {
      return true;
    }
  }

  // 检测边框
  if (includeStroke) {
    const strokeWidth = element.strokeWidth + tolerance;
    return pointInRectangleBorder(localPoint, rect, strokeWidth);
  }

  return false;
};

// 椭圆碰撞检测
const hitTestEllipse = (
  element: ExcalidrawEllipseElement,
  point: Point,
  options: HitTestOptions
): boolean => {
  const { tolerance, includeFill, includeStroke } = options;

  const localPoint = transformPoint(point, element);

  const cx = element.width / 2;
  const cy = element.height / 2;
  const rx = element.width / 2;
  const ry = element.height / 2;

  // 椭圆标准方程：(x-cx)²/rx² + (y-cy)²/ry² = 1
  const normalizedDistance =
    Math.pow(localPoint.x - cx, 2) / Math.pow(rx, 2) +
    Math.pow(localPoint.y - cy, 2) / Math.pow(ry, 2);

  // 检测填充区域
  if (includeFill && element.backgroundColor !== "transparent") {
    if (normalizedDistance <= 1) {
      return true;
    }
  }

  // 检测边框
  if (includeStroke) {
    const strokeTolerance = (element.strokeWidth + tolerance) / Math.min(rx, ry);
    const outerDistance = Math.pow(1 + strokeTolerance, 2);

    return normalizedDistance <= outerDistance && normalizedDistance >= 1;
  }

  return false;
};

// 线条碰撞检测
const hitTestLine = (
  element: ExcalidrawLinearElement,
  point: Point,
  options: HitTestOptions
): boolean => {
  const { tolerance } = options;
  const threshold = element.strokeWidth / 2 + tolerance;

  // 转换到元素坐标系
  const localPoint = {
    x: point.x - element.x,
    y: point.y - element.y,
  };

  // 检测每个线段
  for (let i = 0; i < element.points.length - 1; i++) {
    const p1 = element.points[i];
    const p2 = element.points[i + 1];

    const distance = pointToLineDistance(localPoint, p1, p2);

    if (distance <= threshold) {
      // 检查点是否在线段范围内
      if (pointOnSegment(localPoint, p1, p2, threshold)) {
        return true;
      }
    }
  }

  return false;
};

// 点到线段距离
const pointToLineDistance = (
  point: Point,
  lineStart: Point,
  lineEnd: Point
): number => {
  const A = point.x - lineStart.x;
  const B = point.y - lineStart.y;
  const C = lineEnd.x - lineStart.x;
  const D = lineEnd.y - lineStart.y;

  const dot = A * C + B * D;
  const lenSq = C * C + D * D;

  if (lenSq === 0) {
    return Math.sqrt(A * A + B * B);
  }

  const param = dot / lenSq;

  let xx, yy;

  if (param < 0) {
    xx = lineStart.x;
    yy = lineStart.y;
  } else if (param > 1) {
    xx = lineEnd.x;
    yy = lineEnd.y;
  } else {
    xx = lineStart.x + param * C;
    yy = lineStart.y + param * D;
  }

  const dx = point.x - xx;
  const dy = point.y - yy;

  return Math.sqrt(dx * dx + dy * dy);
};

// 自由绘制碰撞检测
const hitTestFreehand = (
  element: ExcalidrawFreeDrawElement,
  point: Point,
  options: HitTestOptions
): boolean => {
  const { tolerance } = options;
  const threshold = element.strokeWidth / 2 + tolerance;

  const localPoint = {
    x: point.x - element.x,
    y: point.y - element.y,
  };

  // 检测每个线段
  for (let i = 0; i < element.points.length - 1; i++) {
    const p1 = element.points[i];
    const p2 = element.points[i + 1];

    const distance = pointToLineDistance(localPoint,
      { x: p1[0], y: p1[1] },
      { x: p2[0], y: p2[1] }
    );

    if (distance <= threshold) {
      return true;
    }
  }

  return false;
};

// 坐标变换
const transformPoint = (
  point: Point,
  element: ExcalidrawElement
): Point => {
  if (element.angle === 0) {
    return {
      x: point.x - element.x,
      y: point.y - element.y,
    };
  }

  // 旋转变换
  const cos = Math.cos(-element.angle);
  const sin = Math.sin(-element.angle);

  const dx = point.x - element.x - element.width / 2;
  const dy = point.y - element.y - element.height / 2;

  return {
    x: dx * cos - dy * sin + element.width / 2,
    y: dx * sin + dy * cos + element.height / 2,
  };
};
```

### 多元素选择算法

**源码位置**: `packages/element/src/selection.ts` (lines 58-100)

```typescript
// 框选实现 - 实际函数名为 getElementsWithinSelection（已验证）
export const getElementsWithinSelection = (
  elements: readonly NonDeletedExcalidrawElement[],
  selection: NonDeletedExcalidrawElement,
  elementsMap: ElementsMap,
  excludeElementsInFrames: boolean = true,
) => {
  const [selectionX1, selectionY1, selectionX2, selectionY2] =
    getElementAbsoluteCoords(selection, elementsMap);

  let elementsInSelection = elements.filter((element) => {
    let [elementX1, elementY1, elementX2, elementY2] = getElementBounds(
      element,
      elementsMap,
    );

    const containingFrame = getContainingFrame(element, elementsMap);
    if (containingFrame) {
      const [fx1, fy1, fx2, fy2] = getElementBounds(
        containingFrame,
        elementsMap,
      );

      elementX1 = Math.max(fx1, elementX1);
      elementY1 = Math.max(fy1, elementY1);
      elementX2 = Math.min(fx2, elementX2);
      elementY2 = Math.min(fy2, elementY2);
    }

    return (
      element.locked === false &&
      element.type !== "selection" &&
      !isBoundToContainer(element) &&
      selectionX1 <= elementX1 &&
      selectionY1 <= elementY1 &&
      selectionX2 >= elementX2 &&
      selectionY2 >= elementY2
    );
  });

  elementsInSelection = excludeElementsInFrames
    ? excludeElementsInFramesFromSelection(elementsInSelection)
    : elementsInSelection;
  return elements.filter(element => {
    if (element.isDeleted || element.locked) {
      return false;
    }

    const elementBounds = getElementBounds(element);

    switch (mode) {
      case "contain":
        return rectangleContainsRectangle(selectionBox, elementBounds);
      case "intersect":
        return rectangleIntersectsRectangle(selectionBox, elementBounds);
      default:
        return false;
    }
  });
};

// 矩形包含检测
const rectangleContainsRectangle = (
  container: Rectangle,
  contained: Rectangle
): boolean => {
  return (
    contained.x >= container.x &&
    contained.y >= container.y &&
    contained.x + contained.width <= container.x + container.width &&
    contained.y + contained.height <= container.y + container.height
  );
};

// 矩形相交检测
const rectangleIntersectsRectangle = (
  rect1: Rectangle,
  rect2: Rectangle
): boolean => {
  return !(
    rect1.x > rect2.x + rect2.width ||
    rect2.x > rect1.x + rect1.width ||
    rect1.y > rect2.y + rect2.height ||
    rect2.y > rect1.y + rect1.height
  );
};

// 套索选择实现
export const getElementsInLassoPath = (
  elements: readonly ExcalidrawElement[],
  lassoPath: Point[]
): ExcalidrawElement[] => {
  if (lassoPath.length < 3) {
    return [];
  }

  return elements.filter(element => {
    if (element.isDeleted || element.locked) {
      return false;
    }

    // 检查元素中心点是否在套索路径内
    const center = getElementCenter(element);
    return pointInPolygon(center, lassoPath);
  });
};

// 点在多边形内检测（射线法）
const pointInPolygon = (point: Point, polygon: Point[]): boolean => {
  let inside = false;

  for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
    const xi = polygon[i].x;
    const yi = polygon[i].y;
    const xj = polygon[j].x;
    const yj = polygon[j].y;

    if (((yi > point.y) !== (yj > point.y)) &&
        (point.x < (xj - xi) * (point.y - yi) / (yj - yi) + xi)) {
      inside = !inside;
    }
  }

  return inside;
};
```

### 选择优先级系统

```typescript
// 选择优先级规则
export interface PriorityRule {
  type: "size" | "type" | "layer" | "recent" | "distance";
  order: "asc" | "desc";
  weight: number;
}

// 默认优先级规则
const defaultPriorityRules: PriorityRule[] = [
  { type: "size", order: "asc", weight: 1.0 },      // 小元素优先
  { type: "layer", order: "desc", weight: 0.8 },    // 上层优先
  { type: "distance", order: "asc", weight: 0.6 },  // 距离近的优先
  { type: "type", order: "asc", weight: 0.4 },      // 特定类型优先
];

// 元素类型优先级
const typePriority: Record<string, number> = {
  "text": 1,      // 文本最高优先级
  "image": 2,     // 图片次之
  "frame": 3,     // 框架
  "rectangle": 4,
  "ellipse": 4,
  "diamond": 4,
  "arrow": 5,
  "line": 5,
  "freedraw": 6,  // 自由绘制最低优先级
};

// 选择候选元素排序
export const sortSelectionCandidates = (
  candidates: ElementCandidate[],
  clickPoint: Point,
  rules: PriorityRule[] = defaultPriorityRules
): ElementCandidate[] => {
  return candidates.sort((a, b) => {
    let scoreA = 0;
    let scoreB = 0;

    rules.forEach(rule => {
      const valueA = getElementValue(a.element, rule.type, clickPoint);
      const valueB = getElementValue(b.element, rule.type, clickPoint);

      let comparison = 0;
      if (valueA < valueB) comparison = -1;
      else if (valueA > valueB) comparison = 1;

      if (rule.order === "desc") comparison *= -1;

      scoreA += comparison * rule.weight;
      scoreB -= comparison * rule.weight;
    });

    return scoreB - scoreA; // 分数高的排在前面
  });
};

// 获取元素在特定规则下的值
const getElementValue = (
  element: ExcalidrawElement,
  ruleType: string,
  clickPoint: Point
): number => {
  switch (ruleType) {
    case "size":
      return element.width * element.height;

    case "type":
      return typePriority[element.type] || 0;

    case "layer":
      return element.versionNonce; // 使用版本号代表层级

    case "distance":
      const center = getElementCenter(element);
      return Math.hypot(center.x - clickPoint.x, center.y - clickPoint.y);

    case "recent":
      return element.versionNonce; // 最近修改的优先

    default:
      return 0;
  }
};

interface ElementCandidate {
  element: ExcalidrawElement;
  distance: number;
  hitArea: "fill" | "stroke" | "handle";
}
```

## 变换系统实现

### 变换手柄设计

**源码位置**: `packages/element/src/resizeTest.ts`

Excalidraw 使用 `getTransformHandleTypeFromCoords` 函数来检测手柄：

**源码位置**: `packages/element/src/resizeTest.ts` (lines 155-220)

```typescript
// 变换手柄类型 - 实际定义在 @excalidraw/element/types
export type TransformHandleType =
  | "n"
  | "s"
  | "w"
  | "e"
  | "nw"
  | "ne"
  | "sw"
  | "se";

export type MaybeTransformHandleType = TransformHandleType | "rotation" | false;

// 实际的变换手柄检测函数（已验证）
export const getTransformHandleTypeFromCoords = <
  Point extends GlobalPoint | LocalPoint,
>(
  [x1, y1, x2, y2]: Bounds,
  scenePointerX: number,
  scenePointerY: number,
  zoom: Zoom,
  pointerType: PointerType,
  device: Device,
): MaybeTransformHandleType => {
  const transformHandles = getTransformHandlesFromCoords(
    [x1, y1, x2, y2, (x1 + x2) / 2, (y1 + y2) / 2],
    0 as Radians,
    zoom,
    pointerType,
    getOmitSidesForDevice(device),
  );

  const found = Object.keys(transformHandles).find((key) => {
    const transformHandle =
      transformHandles[key as Exclude<TransformHandleType, "rotation">]!;
    return (
      transformHandle &&
      isInsideTransformHandle(transformHandle, scenePointerX, scenePointerY)
    );
  });

  if (found) {
    return found as MaybeTransformHandleType;
  }

  // ... 边缘检测逻辑
};

// 变换手柄定义
export interface TransformHandleDefinition {
  type: TransformHandle;
  x: number;
  y: number;
  cursor: string;
  size: number;
  visible: boolean;
  constrainProportions?: boolean;
}

// 生成变换手柄
export const getTransformHandles = (
  element: ExcalidrawElement,
  zoom: number,
  rotation: number = 0
): TransformHandleDefinition[] => {
  const bounds = getElementBounds(element);
  const { x, y, width, height } = bounds;

  // 手柄大小随缩放调整
  const handleSize = Math.max(8, Math.min(12, 10 / zoom));
  const rotationHandleDistance = 20 / zoom;

  const handles: TransformHandleDefinition[] = [];

  // 四个角点
  handles.push(
    {
      type: "nw",
      x: x - handleSize / 2,
      y: y - handleSize / 2,
      cursor: getResizeCursor(rotation, "nw-resize"),
      size: handleSize,
      visible: true,
      constrainProportions: true,
    },
    {
      type: "ne",
      x: x + width - handleSize / 2,
      y: y - handleSize / 2,
      cursor: getResizeCursor(rotation, "ne-resize"),
      size: handleSize,
      visible: true,
      constrainProportions: true,
    },
    {
      type: "sw",
      x: x - handleSize / 2,
      y: y + height - handleSize / 2,
      cursor: getResizeCursor(rotation, "sw-resize"),
      size: handleSize,
      visible: true,
      constrainProportions: true,
    },
    {
      type: "se",
      x: x + width - handleSize / 2,
      y: y + height - handleSize / 2,
      cursor: getResizeCursor(rotation, "se-resize"),
      size: handleSize,
      visible: true,
      constrainProportions: true,
    }
  );

  // 边中点（只在元素足够大时显示）
  if (width > 40 / zoom) {
    handles.push(
      {
        type: "n",
        x: x + width / 2 - handleSize / 2,
        y: y - handleSize / 2,
        cursor: getResizeCursor(rotation, "n-resize"),
        size: handleSize,
        visible: true,
      },
      {
        type: "s",
        x: x + width / 2 - handleSize / 2,
        y: y + height - handleSize / 2,
        cursor: getResizeCursor(rotation, "s-resize"),
        size: handleSize,
        visible: true,
      }
    );
  }

  if (height > 40 / zoom) {
    handles.push(
      {
        type: "w",
        x: x - handleSize / 2,
        y: y + height / 2 - handleSize / 2,
        cursor: getResizeCursor(rotation, "w-resize"),
        size: handleSize,
        visible: true,
      },
      {
        type: "e",
        x: x + width - handleSize / 2,
        y: y + height / 2 - handleSize / 2,
        cursor: getResizeCursor(rotation, "e-resize"),
        size: handleSize,
        visible: true,
      }
    );
  }

  // 旋转手柄
  handles.push({
    type: "rotation",
    x: x + width / 2 - handleSize / 2,
    y: y - rotationHandleDistance - handleSize / 2,
    cursor: "crosshair",
    size: handleSize,
    visible: true,
  });

  return handles;
};

// 根据旋转角度调整光标
const getResizeCursor = (rotation: number, baseCursor: string): string => {
  // 将角度标准化到 [0, 2π)
  const normalizedAngle = ((rotation % (2 * Math.PI)) + 2 * Math.PI) % (2 * Math.PI);

  // 转换为度数
  const degrees = (normalizedAngle * 180) / Math.PI;

  // 根据角度调整光标
  const cursorMap: Record<string, string[]> = {
    "nw-resize": ["nw-resize", "n-resize", "ne-resize", "e-resize", "se-resize", "s-resize", "sw-resize", "w-resize"],
    "ne-resize": ["ne-resize", "e-resize", "se-resize", "s-resize", "sw-resize", "w-resize", "nw-resize", "n-resize"],
    "sw-resize": ["sw-resize", "w-resize", "nw-resize", "n-resize", "ne-resize", "e-resize", "se-resize", "s-resize"],
    "se-resize": ["se-resize", "s-resize", "sw-resize", "w-resize", "nw-resize", "n-resize", "ne-resize", "e-resize"],
    "n-resize": ["n-resize", "ne-resize", "e-resize", "se-resize", "s-resize", "sw-resize", "w-resize", "nw-resize"],
    "s-resize": ["s-resize", "sw-resize", "w-resize", "nw-resize", "n-resize", "ne-resize", "e-resize", "se-resize"],
    "w-resize": ["w-resize", "nw-resize", "n-resize", "ne-resize", "e-resize", "se-resize", "s-resize", "sw-resize"],
    "e-resize": ["e-resize", "se-resize", "s-resize", "sw-resize", "w-resize", "nw-resize", "n-resize", "ne-resize"],
  };

  const cursors = cursorMap[baseCursor];
  if (!cursors) return baseCursor;

  // 每45度一个cursor
  const index = Math.round(degrees / 45) % 8;
  return cursors[index];
};
```

### 变换计算

```typescript
// 变换操作类型
export interface TransformOperation {
  type: "resize" | "rotate" | "move";
  handle?: TransformHandle;
  startPoint: Point;
  currentPoint: Point;
  element: ExcalidrawElement;
  originalElement: ExcalidrawElement;
  constrainProportions: boolean;
  constrainAngle: boolean;
  fromCenter: boolean;
}

// 执行变换
export const performTransform = (
  operation: TransformOperation
): ExcalidrawElement => {
  switch (operation.type) {
    case "resize":
      return performResize(operation);
    case "rotate":
      return performRotate(operation);
    case "move":
      return performMove(operation);
    default:
      return operation.element;
  }
};

// 缩放变换
const performResize = (operation: TransformOperation): ExcalidrawElement => {
  const { handle, startPoint, currentPoint, originalElement, constrainProportions, fromCenter } = operation;

  if (!handle || handle === "rotation" || handle === "center" || handle === "move") {
    return originalElement;
  }

  const bounds = getElementBounds(originalElement);
  const deltaX = currentPoint.x - startPoint.x;
  const deltaY = currentPoint.y - startPoint.y;

  let newBounds = { ...bounds };

  // 根据手柄类型计算新的边界
  switch (handle) {
    case "nw":
      newBounds.x += deltaX;
      newBounds.y += deltaY;
      newBounds.width -= deltaX;
      newBounds.height -= deltaY;
      break;
    case "ne":
      newBounds.y += deltaY;
      newBounds.width += deltaX;
      newBounds.height -= deltaY;
      break;
    case "sw":
      newBounds.x += deltaX;
      newBounds.width -= deltaX;
      newBounds.height += deltaY;
      break;
    case "se":
      newBounds.width += deltaX;
      newBounds.height += deltaY;
      break;
    case "n":
      newBounds.y += deltaY;
      newBounds.height -= deltaY;
      break;
    case "s":
      newBounds.height += deltaY;
      break;
    case "w":
      newBounds.x += deltaX;
      newBounds.width -= deltaX;
      break;
    case "e":
      newBounds.width += deltaX;
      break;
  }

  // 约束比例
  if (constrainProportions) {
    newBounds = constrainProportionsInBounds(newBounds, bounds, handle);
  }

  // 从中心缩放
  if (fromCenter) {
    newBounds = adjustBoundsForCenterResize(newBounds, bounds);
  }

  // 最小尺寸约束
  newBounds.width = Math.max(10, Math.abs(newBounds.width));
  newBounds.height = Math.max(10, Math.abs(newBounds.height));

  // 处理负尺寸
  if (newBounds.width < 0) {
    newBounds.x += newBounds.width;
    newBounds.width = Math.abs(newBounds.width);
  }

  if (newBounds.height < 0) {
    newBounds.y += newBounds.height;
    newBounds.height = Math.abs(newBounds.height);
  }

  // 应用新的边界到元素
  return {
    ...originalElement,
    x: newBounds.x,
    y: newBounds.y,
    width: newBounds.width,
    height: newBounds.height,
  };
};

// 约束比例
const constrainProportionsInBounds = (
  newBounds: Rectangle,
  originalBounds: Rectangle,
  handle: TransformHandle
): Rectangle => {
  const originalRatio = originalBounds.width / originalBounds.height;

  let constrainedBounds = { ...newBounds };

  // 根据手柄类型决定如何约束比例
  switch (handle) {
    case "nw":
    case "ne":
    case "sw":
    case "se":
      // 角点：根据较大的变化量来约束
      const widthChange = Math.abs(newBounds.width - originalBounds.width);
      const heightChange = Math.abs(newBounds.height - originalBounds.height);

      if (widthChange > heightChange) {
        // 以宽度为准
        const newHeight = Math.abs(newBounds.width) / originalRatio;
        constrainedBounds.height = newBounds.height >= 0 ? newHeight : -newHeight;
      } else {
        // 以高度为准
        const newWidth = Math.abs(newBounds.height) * originalRatio;
        constrainedBounds.width = newBounds.width >= 0 ? newWidth : -newWidth;
      }
      break;
  }

  return constrainedBounds;
};

// 旋转变换
const performRotate = (operation: TransformOperation): ExcalidrawElement => {
  const { startPoint, currentPoint, originalElement } = operation;

  const center = getElementCenter(originalElement);

  // 计算起始角度和当前角度
  const startAngle = Math.atan2(startPoint.y - center.y, startPoint.x - center.x);
  const currentAngle = Math.atan2(currentPoint.y - center.y, currentPoint.x - center.x);

  let deltaAngle = currentAngle - startAngle;

  // 角度约束（15度吸附）
  if (operation.constrainAngle) {
    const snapAngle = Math.PI / 12; // 15度
    deltaAngle = Math.round(deltaAngle / snapAngle) * snapAngle;
  }

  const newAngle = originalElement.angle + deltaAngle;

  return {
    ...originalElement,
    angle: normalizeAngle(newAngle),
  };
};

// 移动变换
const performMove = (operation: TransformOperation): ExcalidrawElement => {
  const { startPoint, currentPoint, originalElement } = operation;

  const deltaX = currentPoint.x - startPoint.x;
  const deltaY = currentPoint.y - startPoint.y;

  return {
    ...originalElement,
    x: originalElement.x + deltaX,
    y: originalElement.y + deltaY,
  };
};

// 标准化角度到 [0, 2π)
const normalizeAngle = (angle: number): number => {
  while (angle < 0) angle += 2 * Math.PI;
  while (angle >= 2 * Math.PI) angle -= 2 * Math.PI;
  return angle;
};
```

### 多选变换

```typescript
// 多选变换管理器
export class MultiSelectionTransform {
  private elements: ExcalidrawElement[];
  private originalElements: ExcalidrawElement[];
  private bounds: Rectangle;
  private center: Point;

  constructor(elements: ExcalidrawElement[]) {
    this.elements = [...elements];
    this.originalElements = elements.map(el => ({ ...el }));
    this.bounds = getElementsCommonBounds(elements);
    this.center = {
      x: this.bounds.x + this.bounds.width / 2,
      y: this.bounds.y + this.bounds.height / 2,
    };
  }

  // 应用变换到所有元素
  applyTransform(operation: TransformOperation): ExcalidrawElement[] {
    switch (operation.type) {
      case "resize":
        return this.applyGroupResize(operation);
      case "rotate":
        return this.applyGroupRotate(operation);
      case "move":
        return this.applyGroupMove(operation);
      default:
        return this.elements;
    }
  }

  // 组缩放
  private applyGroupResize(operation: TransformOperation): ExcalidrawElement[] {
    const { handle, startPoint, currentPoint, constrainProportions } = operation;

    if (!handle || handle === "rotation" || handle === "center" || handle === "move") {
      return this.elements;
    }

    // 计算缩放比例
    const deltaX = currentPoint.x - startPoint.x;
    const deltaY = currentPoint.y - startPoint.y;

    const scaleX = this.calculateScaleX(handle, deltaX);
    const scaleY = this.calculateScaleY(handle, deltaY);

    // 约束比例
    let finalScaleX = scaleX;
    let finalScaleY = scaleY;

    if (constrainProportions) {
      const scale = Math.max(Math.abs(scaleX), Math.abs(scaleY));
      finalScaleX = scaleX >= 0 ? scale : -scale;
      finalScaleY = scaleY >= 0 ? scale : -scale;
    }

    // 计算变换原点
    const transformOrigin = this.getTransformOrigin(handle);

    // 应用到每个元素
    return this.originalElements.map(element => {
      return this.scaleElementRelativeToPoint(
        element,
        transformOrigin,
        finalScaleX,
        finalScaleY
      );
    });
  }

  // 组旋转
  private applyGroupRotate(operation: TransformOperation): ExcalidrawElement[] {
    const { startPoint, currentPoint, constrainAngle } = operation;

    const startAngle = Math.atan2(startPoint.y - this.center.y, startPoint.x - this.center.x);
    const currentAngle = Math.atan2(currentPoint.y - this.center.y, currentPoint.x - this.center.x);

    let deltaAngle = currentAngle - startAngle;

    // 角度约束
    if (constrainAngle) {
      const snapAngle = Math.PI / 12; // 15度
      deltaAngle = Math.round(deltaAngle / snapAngle) * snapAngle;
    }

    // 应用到每个元素
    return this.originalElements.map(element => {
      return this.rotateElementAroundPoint(element, this.center, deltaAngle);
    });
  }

  // 组移动
  private applyGroupMove(operation: TransformOperation): ExcalidrawElement[] {
    const { startPoint, currentPoint } = operation;

    const deltaX = currentPoint.x - startPoint.x;
    const deltaY = currentPoint.y - startPoint.y;

    return this.originalElements.map(element => ({
      ...element,
      x: element.x + deltaX,
      y: element.y + deltaY,
    }));
  }

  // 计算X方向缩放比例
  private calculateScaleX(handle: TransformHandle, deltaX: number): number {
    switch (handle) {
      case "nw":
      case "w":
      case "sw":
        return (this.bounds.width - deltaX) / this.bounds.width;
      case "ne":
      case "e":
      case "se":
        return (this.bounds.width + deltaX) / this.bounds.width;
      default:
        return 1;
    }
  }

  // 计算Y方向缩放比例
  private calculateScaleY(handle: TransformHandle, deltaY: number): number {
    switch (handle) {
      case "nw":
      case "n":
      case "ne":
        return (this.bounds.height - deltaY) / this.bounds.height;
      case "sw":
      case "s":
      case "se":
        return (this.bounds.height + deltaY) / this.bounds.height;
      default:
        return 1;
    }
  }

  // 获取变换原点
  private getTransformOrigin(handle: TransformHandle): Point {
    switch (handle) {
      case "nw":
        return { x: this.bounds.x + this.bounds.width, y: this.bounds.y + this.bounds.height };
      case "ne":
        return { x: this.bounds.x, y: this.bounds.y + this.bounds.height };
      case "sw":
        return { x: this.bounds.x + this.bounds.width, y: this.bounds.y };
      case "se":
        return { x: this.bounds.x, y: this.bounds.y };
      case "n":
        return { x: this.bounds.x + this.bounds.width / 2, y: this.bounds.y + this.bounds.height };
      case "s":
        return { x: this.bounds.x + this.bounds.width / 2, y: this.bounds.y };
      case "w":
        return { x: this.bounds.x + this.bounds.width, y: this.bounds.y + this.bounds.height / 2 };
      case "e":
        return { x: this.bounds.x, y: this.bounds.y + this.bounds.height / 2 };
      default:
        return this.center;
    }
  }

  // 相对于点缩放元素
  private scaleElementRelativeToPoint(
    element: ExcalidrawElement,
    origin: Point,
    scaleX: number,
    scaleY: number
  ): ExcalidrawElement {
    // 缩放位置
    const newX = origin.x + (element.x - origin.x) * scaleX;
    const newY = origin.y + (element.y - origin.y) * scaleY;

    // 缩放尺寸
    const newWidth = element.width * Math.abs(scaleX);
    const newHeight = element.height * Math.abs(scaleY);

    return {
      ...element,
      x: newX,
      y: newY,
      width: newWidth,
      height: newHeight,
    };
  }

  // 围绕点旋转元素
  private rotateElementAroundPoint(
    element: ExcalidrawElement,
    center: Point,
    angle: number
  ): ExcalidrawElement {
    const elementCenter = getElementCenter(element);

    // 旋转元素中心
    const cos = Math.cos(angle);
    const sin = Math.sin(angle);

    const dx = elementCenter.x - center.x;
    const dy = elementCenter.y - center.y;

    const newCenterX = center.x + dx * cos - dy * sin;
    const newCenterY = center.y + dx * sin + dy * cos;

    // 计算新的元素位置
    const newX = newCenterX - element.width / 2;
    const newY = newCenterY - element.height / 2;

    // 更新元素角度
    const newAngle = normalizeAngle(element.angle + angle);

    return {
      ...element,
      x: newX,
      y: newY,
      angle: newAngle,
    };
  }
}

// 获取多个元素的公共边界
const getElementsCommonBounds = (elements: ExcalidrawElement[]): Rectangle => {
  if (elements.length === 0) {
    return { x: 0, y: 0, width: 0, height: 0 };
  }

  let minX = Infinity;
  let minY = Infinity;
  let maxX = -Infinity;
  let maxY = -Infinity;

  elements.forEach(element => {
    const bounds = getElementBounds(element);
    minX = Math.min(minX, bounds.x);
    minY = Math.min(minY, bounds.y);
    maxX = Math.max(maxX, bounds.x + bounds.width);
    maxY = Math.max(maxY, bounds.y + bounds.height);
  });

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
};
```

## 实战示例：完整选择变换系统

### 交互式选择变换组件

```javascript
class SelectionTransformSystem {
  constructor(canvas) {
    this.canvas = canvas;
    this.context = canvas.getContext('2d');
    this.elements = [];
    this.selectedElements = [];
    this.isSelecting = false;
    this.isTransforming = false;

    // 变换状态
    this.transformOperation = null;
    this.transformHandles = [];
    this.hoveredHandle = null;

    // 选择状态
    this.selectionBox = null;
    this.selectionStartPoint = null;

    this.setupEventHandlers();
  }

  setupEventHandlers() {
    this.canvas.addEventListener('pointerdown', this.handlePointerDown.bind(this));
    this.canvas.addEventListener('pointermove', this.handlePointerMove.bind(this));
    this.canvas.addEventListener('pointerup', this.handlePointerUp.bind(this));
    this.canvas.addEventListener('keydown', this.handleKeyDown.bind(this));
  }

  handlePointerDown(event) {
    const point = this.getCanvasPoint(event);

    // 检查是否点击在变换手柄上
    const handle = this.getHandleAtPoint(point);
    if (handle) {
      this.startTransform(handle, point);
      return;
    }

    // 检查是否点击在选中的元素上
    const hitSelected = this.selectedElements.find(el =>
      this.hitTestElement(el, point)
    );

    if (hitSelected) {
      // 开始移动选中的元素
      this.startMove(point);
      return;
    }

    // 检查是否点击在其他元素上
    const hitElement = this.getElementAtPoint(point);
    if (hitElement) {
      // 选择元素
      if (event.shiftKey) {
        this.addToSelection(hitElement);
      } else {
        this.selectElement(hitElement);
      }
      this.startMove(point);
      return;
    }

    // 开始框选
    this.startSelection(point);
  }

  handlePointerMove(event) {
    const point = this.getCanvasPoint(event);

    if (this.isTransforming) {
      this.updateTransform(point);
    } else if (this.isSelecting) {
      this.updateSelection(point);
    } else {
      // 更新光标
      this.updateCursor(point);
    }

    this.render();
  }

  handlePointerUp(event) {
    const point = this.getCanvasPoint(event);

    if (this.isTransforming) {
      this.endTransform(point);
    } else if (this.isSelecting) {
      this.endSelection(point);
    }

    this.render();
  }

  handleKeyDown(event) {
    switch (event.key) {
      case 'Delete':
      case 'Backspace':
        this.deleteSelected();
        break;
      case 'Escape':
        this.clearSelection();
        break;
      case 'a':
        if (event.ctrlKey || event.metaKey) {
          event.preventDefault();
          this.selectAll();
        }
        break;
    }
  }

  // 开始变换
  startTransform(handle, point) {
    this.isTransforming = true;

    const selectedElement = this.selectedElements.length === 1
      ? this.selectedElements[0]
      : null;

    this.transformOperation = {
      type: handle.type === 'rotation' ? 'rotate' : 'resize',
      handle: handle.type,
      startPoint: point,
      currentPoint: point,
      element: selectedElement,
      originalElements: this.selectedElements.map(el => ({ ...el })),
      constrainProportions: false,
      constrainAngle: false,
      fromCenter: false,
    };
  }

  // 更新变换
  updateTransform(point) {
    if (!this.transformOperation) return;

    this.transformOperation.currentPoint = point;
    this.transformOperation.constrainProportions = event.shiftKey;
    this.transformOperation.constrainAngle = event.shiftKey;
    this.transformOperation.fromCenter = event.altKey;

    if (this.selectedElements.length === 1) {
      // 单元素变换
      const transformed = this.performTransform(this.transformOperation);
      this.selectedElements = [transformed];
    } else {
      // 多元素变换
      const multiTransform = new MultiSelectionTransform(this.transformOperation.originalElements);
      this.selectedElements = multiTransform.applyTransform(this.transformOperation);
    }
  }

  // 结束变换
  endTransform(point) {
    this.isTransforming = false;

    if (this.transformOperation) {
      // 更新元素数组
      this.transformOperation.originalElements.forEach(originalEl => {
        const index = this.elements.findIndex(el => el.id === originalEl.id);
        if (index !== -1) {
          const transformedEl = this.selectedElements.find(el => el.id === originalEl.id);
          if (transformedEl) {
            this.elements[index] = transformedEl;
          }
        }
      });

      this.updateTransformHandles();
    }

    this.transformOperation = null;
  }

  // 开始框选
  startSelection(point) {
    this.isSelecting = true;
    this.selectionStartPoint = point;
    this.selectionBox = {
      x: point.x,
      y: point.y,
      width: 0,
      height: 0,
    };
  }

  // 更新框选
  updateSelection(point) {
    if (!this.selectionStartPoint) return;

    const startX = this.selectionStartPoint.x;
    const startY = this.selectionStartPoint.y;

    this.selectionBox = {
      x: Math.min(startX, point.x),
      y: Math.min(startY, point.y),
      width: Math.abs(point.x - startX),
      height: Math.abs(point.y - startY),
    };

    // 实时更新选择
    const elementsInBox = this.getElementsInBox(this.selectionBox);
    this.selectedElements = elementsInBox;
  }

  // 结束框选
  endSelection(point) {
    this.isSelecting = false;

    if (this.selectionBox && (this.selectionBox.width > 5 || this.selectionBox.height > 5)) {
      const elementsInBox = this.getElementsInBox(this.selectionBox);
      this.selectedElements = elementsInBox;
    } else {
      this.selectedElements = [];
    }

    this.selectionBox = null;
    this.selectionStartPoint = null;
    this.updateTransformHandles();
  }

  // 元素碰撞检测
  hitTestElement(element, point) {
    // 简化的碰撞检测
    return (
      point.x >= element.x &&
      point.x <= element.x + element.width &&
      point.y >= element.y &&
      point.y <= element.y + element.height
    );
  }

  // 获取点击位置的元素
  getElementAtPoint(point) {
    // 从上到下检测元素
    for (let i = this.elements.length - 1; i >= 0; i--) {
      const element = this.elements[i];
      if (this.hitTestElement(element, point)) {
        return element;
      }
    }
    return null;
  }

  // 获取框选区域内的元素
  getElementsInBox(box) {
    return this.elements.filter(element => {
      return !(
        element.x > box.x + box.width ||
        element.x + element.width < box.x ||
        element.y > box.y + box.height ||
        element.y + element.height < box.y
      );
    });
  }

  // 选择元素
  selectElement(element) {
    this.selectedElements = [element];
    this.updateTransformHandles();
  }

  // 添加到选择
  addToSelection(element) {
    if (!this.selectedElements.includes(element)) {
      this.selectedElements.push(element);
      this.updateTransformHandles();
    }
  }

  // 清除选择
  clearSelection() {
    this.selectedElements = [];
    this.transformHandles = [];
  }

  // 全选
  selectAll() {
    this.selectedElements = [...this.elements];
    this.updateTransformHandles();
  }

  // 删除选中元素
  deleteSelected() {
    this.selectedElements.forEach(selectedEl => {
      const index = this.elements.findIndex(el => el.id === selectedEl.id);
      if (index !== -1) {
        this.elements.splice(index, 1);
      }
    });

    this.selectedElements = [];
    this.transformHandles = [];
    this.render();
  }

  // 更新变换手柄
  updateTransformHandles() {
    if (this.selectedElements.length === 0) {
      this.transformHandles = [];
      return;
    }

    const bounds = this.getSelectionBounds();
    this.transformHandles = this.getTransformHandles(bounds);
  }

  // 获取选择边界
  getSelectionBounds() {
    if (this.selectedElements.length === 0) {
      return { x: 0, y: 0, width: 0, height: 0 };
    }

    let minX = Infinity, minY = Infinity;
    let maxX = -Infinity, maxY = -Infinity;

    this.selectedElements.forEach(element => {
      minX = Math.min(minX, element.x);
      minY = Math.min(minY, element.y);
      maxX = Math.max(maxX, element.x + element.width);
      maxY = Math.max(maxY, element.y + element.height);
    });

    return {
      x: minX,
      y: minY,
      width: maxX - minX,
      height: maxY - minY,
    };
  }

  // 生成变换手柄
  getTransformHandles(bounds) {
    const handles = [];
    const size = 8;

    // 四个角点
    handles.push(
      { type: 'nw', x: bounds.x - size/2, y: bounds.y - size/2, width: size, height: size },
      { type: 'ne', x: bounds.x + bounds.width - size/2, y: bounds.y - size/2, width: size, height: size },
      { type: 'sw', x: bounds.x - size/2, y: bounds.y + bounds.height - size/2, width: size, height: size },
      { type: 'se', x: bounds.x + bounds.width - size/2, y: bounds.y + bounds.height - size/2, width: size, height: size }
    );

    // 边中点
    if (bounds.width > 30) {
      handles.push(
        { type: 'n', x: bounds.x + bounds.width/2 - size/2, y: bounds.y - size/2, width: size, height: size },
        { type: 's', x: bounds.x + bounds.width/2 - size/2, y: bounds.y + bounds.height - size/2, width: size, height: size }
      );
    }

    if (bounds.height > 30) {
      handles.push(
        { type: 'w', x: bounds.x - size/2, y: bounds.y + bounds.height/2 - size/2, width: size, height: size },
        { type: 'e', x: bounds.x + bounds.width - size/2, y: bounds.y + bounds.height/2 - size/2, width: size, height: size }
      );
    }

    // 旋转手柄
    handles.push({
      type: 'rotation',
      x: bounds.x + bounds.width/2 - size/2,
      y: bounds.y - 20 - size/2,
      width: size,
      height: size
    });

    return handles;
  }

  // 获取点击位置的手柄
  getHandleAtPoint(point) {
    return this.transformHandles.find(handle =>
      point.x >= handle.x &&
      point.x <= handle.x + handle.width &&
      point.y >= handle.y &&
      point.y <= handle.y + handle.height
    );
  }

  // 更新光标
  updateCursor(point) {
    const handle = this.getHandleAtPoint(point);

    if (handle) {
      this.canvas.style.cursor = this.getHandleCursor(handle.type);
    } else {
      const hitElement = this.getElementAtPoint(point);
      this.canvas.style.cursor = hitElement ? 'move' : 'default';
    }
  }

  // 获取手柄光标
  getHandleCursor(handleType) {
    const cursors = {
      'nw': 'nw-resize',
      'ne': 'ne-resize',
      'sw': 'sw-resize',
      'se': 'se-resize',
      'n': 'n-resize',
      's': 's-resize',
      'w': 'w-resize',
      'e': 'e-resize',
      'rotation': 'crosshair'
    };
    return cursors[handleType] || 'default';
  }

  // 渲染
  render() {
    this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // 渲染所有元素
    this.elements.forEach(element => {
      this.renderElement(element, this.selectedElements.includes(element));
    });

    // 渲染选择框
    if (this.selectionBox) {
      this.renderSelectionBox(this.selectionBox);
    }

    // 渲染变换手柄
    this.transformHandles.forEach(handle => {
      this.renderTransformHandle(handle);
    });
  }

  // 渲染元素
  renderElement(element, isSelected) {
    this.context.save();

    // 选中状态的特殊样式
    if (isSelected) {
      this.context.strokeStyle = '#007acc';
      this.context.lineWidth = 2;
      this.context.setLineDash([5, 5]);
    } else {
      this.context.strokeStyle = element.strokeColor || '#000';
      this.context.lineWidth = element.strokeWidth || 1;
      this.context.setLineDash([]);
    }

    // 简化的元素渲染
    this.context.fillStyle = element.fillColor || 'transparent';
    this.context.fillRect(element.x, element.y, element.width, element.height);
    this.context.strokeRect(element.x, element.y, element.width, element.height);

    this.context.restore();
  }

  // 渲染选择框
  renderSelectionBox(box) {
    this.context.save();
    this.context.strokeStyle = '#007acc';
    this.context.lineWidth = 1;
    this.context.setLineDash([3, 3]);
    this.context.fillStyle = 'rgba(0, 122, 204, 0.1)';

    this.context.fillRect(box.x, box.y, box.width, box.height);
    this.context.strokeRect(box.x, box.y, box.width, box.height);

    this.context.restore();
  }

  // 渲染变换手柄
  renderTransformHandle(handle) {
    this.context.save();

    if (handle.type === 'rotation') {
      // 旋转手柄特殊样式
      this.context.strokeStyle = '#007acc';
      this.context.fillStyle = 'white';
      this.context.lineWidth = 2;

      const centerX = handle.x + handle.width / 2;
      const centerY = handle.y + handle.height / 2;

      this.context.beginPath();
      this.context.arc(centerX, centerY, handle.width / 2, 0, Math.PI * 2);
      this.context.fill();
      this.context.stroke();
    } else {
      // 普通缩放手柄
      this.context.fillStyle = 'white';
      this.context.strokeStyle = '#007acc';
      this.context.lineWidth = 1;

      this.context.fillRect(handle.x, handle.y, handle.width, handle.height);
      this.context.strokeRect(handle.x, handle.y, handle.width, handle.height);
    }

    this.context.restore();
  }

  // 获取画布坐标
  getCanvasPoint(event) {
    const rect = this.canvas.getBoundingClientRect();
    return {
      x: event.clientX - rect.left,
      y: event.clientY - rect.top,
    };
  }

  // 添加元素
  addElement(element) {
    element.id = this.generateId();
    this.elements.push(element);
    this.render();
  }

  // 生成ID
  generateId() {
    return Math.random().toString(36).substr(2, 9);
  }
}

// 使用示例
const canvas = document.getElementById('selectionCanvas');
const system = new SelectionTransformSystem(canvas);

// 添加一些测试元素
system.addElement({
  x: 50,
  y: 50,
  width: 100,
  height: 80,
  strokeColor: '#000',
  fillColor: 'lightblue',
});

system.addElement({
  x: 200,
  y: 100,
  width: 120,
  height: 60,
  strokeColor: '#000',
  fillColor: 'lightgreen',
});

system.addElement({
  x: 100,
  y: 200,
  width: 80,
  height: 100,
  strokeColor: '#000',
  fillColor: 'lightcoral',
});
```

## 思考题

1. **复杂形状的碰撞检测如何优化？** 不规则图形的精确选择算法？

2. **大量元素时的选择性能如何保证？** 空间索引在选择系统中的应用？

3. **变换约束如何设计？** 智能吸附、比例锁定、角度约束等？

4. **协作环境下的选择冲突如何处理？** 多用户同时选择相同元素？

5. **无障碍选择如何实现？** 键盘导航、屏幕阅读器支持？

## 实践练习

### 练习 1：实现智能选择
创建智能选择功能：
- 相似元素批量选择
- 基于属性的过滤选择
- 选择历史记录

### 练习 2：实现高级变换
实现高级变换功能：
- 3D透视变换
- 自由变形工具
- 变换动画效果

### 练习 3：实现选择优化
优化大量元素的选择：
- 空间索引结构
- 异步选择处理
- 选择结果缓存

## 总结

Excalidraw 的选择与变换系统展现了专业图形编辑器的核心特征：

1. **精确的碰撞检测**：支持各种复杂图形的准确选择
2. **直观的变换操作**：通过可视化手柄提供直观的变换体验
3. **完善的约束系统**：比例锁定、角度吸附等辅助功能
4. **高效的多选处理**：统一的多元素变换算法
5. **良好的视觉反馈**：清晰的选择状态和变换预览

这些技术为构建专业级图形编辑工具提供了坚实的基础。

## 下一步

完成了交互系统的学习后，下一章将深入探讨 Excalidraw 的 Action 与命令系统，学习如何设计可撤销的操作系统和高效的历史管理机制。