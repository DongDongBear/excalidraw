# Chapter 3.3: 渲染性能优化策略

## 概述

在处理复杂场景时，渲染性能优化是决定用户体验的关键因素。Excalidraw 通过一系列精心设计的优化策略，实现了即使在包含数千个元素的场景中也能保持流畅的 60fps 渲染。本章将深入探讨这些优化技术的原理和实现。

## 性能优化架构总览

### 优化层次结构

```
┌─────────────────────────────────────────────┐
│           应用层优化 (Application)           │
│  批处理 | 防抖节流 | 状态管理 | 异步处理     │
└─────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│           渲染层优化 (Rendering)            │
│  视口裁剪 | 分层渲染 | 缓存策略 | 脏矩形     │
└─────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────┐
│            底层优化 (Low-level)             │
│  Canvas API | 内存管理 | GPU加速 | WebWorker│
└─────────────────────────────────────────────┘
```

## 分层渲染策略

### 层级设计原理

```typescript
// packages/excalidraw/components/canvases/CanvasLayers.tsx
interface RenderLayers {
  // 背景层：网格、背景色（很少变化）
  background: {
    updateFrequency: "rare";
    canvas: HTMLCanvasElement;
    cached: boolean;
  };

  // 静态层：未选中的元素（偶尔变化）
  static: {
    updateFrequency: "occasional";
    canvas: HTMLCanvasElement;
    cached: boolean;
  };

  // 动态层：选中的元素、正在编辑的内容（频繁变化）
  dynamic: {
    updateFrequency: "frequent";
    canvas: HTMLCanvasElement;
    cached: false;
  };

  // UI层：选择框、手柄、光标（实时变化）
  ui: {
    updateFrequency: "realtime";
    canvas: HTMLCanvasElement;
    cached: false;
  };
}

// 分层渲染管理器
class LayeredRenderer {
  private layers: Map<string, CanvasLayer> = new Map();

  constructor() {
    this.initializeLayers();
  }

  private initializeLayers() {
    // 创建各层 Canvas
    this.layers.set('background', new CanvasLayer({
      zIndex: 0,
      cacheEnabled: true,
      updateStrategy: 'lazy',
    }));

    this.layers.set('static', new CanvasLayer({
      zIndex: 1,
      cacheEnabled: true,
      updateStrategy: 'incremental',
    }));

    this.layers.set('dynamic', new CanvasLayer({
      zIndex: 2,
      cacheEnabled: false,
      updateStrategy: 'immediate',
    }));

    this.layers.set('ui', new CanvasLayer({
      zIndex: 3,
      cacheEnabled: false,
      updateStrategy: 'immediate',
    }));
  }

  // 智能更新决策
  public update(changes: SceneChanges) {
    // 根据变化类型决定更新哪些层
    if (changes.backgroundChanged) {
      this.updateLayer('background', changes);
    }

    if (changes.elementsChanged) {
      // 分离静态和动态元素
      const { staticElements, dynamicElements } = this.separateElements(changes.elements);

      if (staticElements.length > 0) {
        this.updateLayer('static', { elements: staticElements });
      }

      if (dynamicElements.length > 0) {
        this.updateLayer('dynamic', { elements: dynamicElements });
      }
    }

    if (changes.uiChanged) {
      this.updateLayer('ui', changes);
    }
  }

  // 分离静态和动态元素
  private separateElements(elements: ExcalidrawElement[]) {
    const staticElements: ExcalidrawElement[] = [];
    const dynamicElements: ExcalidrawElement[] = [];

    elements.forEach(element => {
      if (element.isSelected || element.isBeingEdited) {
        dynamicElements.push(element);
      } else {
        staticElements.push(element);
      }
    });

    return { staticElements, dynamicElements };
  }
}
```

### 层级缓存机制

```typescript
class CanvasLayer {
  private canvas: HTMLCanvasElement;
  private context: CanvasRenderingContext2D;
  private cache: HTMLCanvasElement | null = null;
  private isDirty: boolean = false;

  constructor(private config: LayerConfig) {
    this.canvas = document.createElement('canvas');
    this.context = this.canvas.getContext('2d')!;

    if (config.cacheEnabled) {
      this.cache = document.createElement('canvas');
    }
  }

  // 渲染层内容
  public render(elements: ExcalidrawElement[], renderConfig: RenderConfig) {
    if (!this.isDirty && this.cache) {
      // 使用缓存
      this.context.drawImage(this.cache, 0, 0);
      return;
    }

    // 清空并重新渲染
    this.clear();

    // 根据更新策略选择渲染方式
    switch (this.config.updateStrategy) {
      case 'lazy':
        this.lazyRender(elements, renderConfig);
        break;
      case 'incremental':
        this.incrementalRender(elements, renderConfig);
        break;
      case 'immediate':
        this.immediateRender(elements, renderConfig);
        break;
    }

    // 更新缓存
    if (this.cache) {
      const cacheContext = this.cache.getContext('2d')!;
      cacheContext.clearRect(0, 0, this.cache.width, this.cache.height);
      cacheContext.drawImage(this.canvas, 0, 0);
    }

    this.isDirty = false;
  }

  // 延迟渲染
  private lazyRender(elements: ExcalidrawElement[], config: RenderConfig) {
    requestIdleCallback(() => {
      this.doRender(elements, config);
    });
  }

  // 增量渲染
  private incrementalRender(elements: ExcalidrawElement[], config: RenderConfig) {
    const BATCH_SIZE = 50;
    let index = 0;

    const renderBatch = () => {
      const batch = elements.slice(index, index + BATCH_SIZE);
      this.doRender(batch, config);

      index += BATCH_SIZE;

      if (index < elements.length) {
        requestAnimationFrame(renderBatch);
      }
    };

    renderBatch();
  }

  // 立即渲染
  private immediateRender(elements: ExcalidrawElement[], config: RenderConfig) {
    this.doRender(elements, config);
  }

  // 标记为脏
  public markDirty() {
    this.isDirty = true;
  }
}
```

## 虚拟化渲染

### 视口虚拟化原理

```typescript
// packages/excalidraw/renderer/virtualizer.ts
export class RenderVirtualizer {
  private visibleElements: Set<string> = new Set();
  private elementBounds: Map<string, Bounds> = new Map();

  // 计算可见元素（空间索引优化）
  public getVisibleElements(
    elements: ExcalidrawElement[],
    viewport: Viewport
  ): ExcalidrawElement[] {
    // 使用四叉树或 R-tree 进行空间索引
    const spatialIndex = this.buildSpatialIndex(elements);

    // 查询视口内的元素
    const candidates = spatialIndex.query(viewport);

    // 精确过滤
    return candidates.filter(element => {
      const bounds = this.getElementBounds(element);
      return this.isInViewport(bounds, viewport);
    });
  }

  // 构建空间索引
  private buildSpatialIndex(elements: ExcalidrawElement[]): SpatialIndex {
    const index = new QuadTree({
      x: -10000,
      y: -10000,
      width: 20000,
      height: 20000,
    });

    elements.forEach(element => {
      const bounds = this.getElementBounds(element);
      index.insert({
        x: bounds.x,
        y: bounds.y,
        width: bounds.width,
        height: bounds.height,
        element,
      });
    });

    return index;
  }

  // 获取元素边界（带缓存）
  private getElementBounds(element: ExcalidrawElement): Bounds {
    const cached = this.elementBounds.get(element.id);

    if (cached && cached.versionNonce === element.versionNonce) {
      return cached;
    }

    const bounds = calculateElementBounds(element);
    this.elementBounds.set(element.id, {
      ...bounds,
      versionNonce: element.versionNonce,
    });

    return bounds;
  }

  // 级别细节（LOD）优化
  public getLevelOfDetail(element: ExcalidrawElement, zoom: number): LOD {
    const elementSize = Math.max(element.width, element.height);
    const screenSize = elementSize * zoom;

    if (screenSize < 10) {
      return LOD.MINIMAL;  // 只渲染边界框
    } else if (screenSize < 50) {
      return LOD.SIMPLIFIED;  // 简化渲染
    } else {
      return LOD.FULL;  // 完整渲染
    }
  }
}

// 四叉树实现
class QuadTree {
  private root: QuadTreeNode;

  constructor(bounds: Bounds) {
    this.root = new QuadTreeNode(bounds);
  }

  insert(item: SpatialItem) {
    this.root.insert(item);
  }

  query(range: Bounds): SpatialItem[] {
    return this.root.query(range);
  }
}

class QuadTreeNode {
  private static MAX_ITEMS = 4;
  private static MAX_DEPTH = 10;

  private items: SpatialItem[] = [];
  private children: QuadTreeNode[] | null = null;

  constructor(
    private bounds: Bounds,
    private depth: number = 0
  ) {}

  insert(item: SpatialItem): boolean {
    if (!this.contains(item)) {
      return false;
    }

    if (this.items.length < QuadTreeNode.MAX_ITEMS ||
        this.depth >= QuadTreeNode.MAX_DEPTH) {
      this.items.push(item);
      return true;
    }

    if (!this.children) {
      this.subdivide();
    }

    for (const child of this.children!) {
      if (child.insert(item)) {
        return true;
      }
    }

    return false;
  }

  query(range: Bounds): SpatialItem[] {
    const result: SpatialItem[] = [];

    if (!this.intersects(range)) {
      return result;
    }

    for (const item of this.items) {
      if (this.itemInRange(item, range)) {
        result.push(item);
      }
    }

    if (this.children) {
      for (const child of this.children) {
        result.push(...child.query(range));
      }
    }

    return result;
  }

  private subdivide() {
    const { x, y, width, height } = this.bounds;
    const halfWidth = width / 2;
    const halfHeight = height / 2;

    this.children = [
      new QuadTreeNode({ x, y, width: halfWidth, height: halfHeight }, this.depth + 1),
      new QuadTreeNode({ x: x + halfWidth, y, width: halfWidth, height: halfHeight }, this.depth + 1),
      new QuadTreeNode({ x, y: y + halfHeight, width: halfWidth, height: halfHeight }, this.depth + 1),
      new QuadTreeNode({ x: x + halfWidth, y: y + halfHeight, width: halfWidth, height: halfHeight }, this.depth + 1),
    ];

    // 重新分配已有元素
    const items = this.items;
    this.items = [];

    for (const item of items) {
      this.insert(item);
    }
  }
}
```

## 脏矩形算法

### 脏矩形追踪系统

```typescript
// packages/excalidraw/renderer/dirtyRect.ts
export class DirtyRectManager {
  private dirtyRects: Rect[] = [];
  private previousFrame: Map<string, ElementSnapshot> = new Map();

  // 计算脏矩形
  public calculateDirtyRects(
    elements: ExcalidrawElement[],
    changes: ElementChanges
  ): Rect[] {
    this.dirtyRects = [];

    // 处理修改的元素
    changes.modified.forEach(element => {
      const prevSnapshot = this.previousFrame.get(element.id);
      if (prevSnapshot) {
        // 添加元素的旧位置和新位置
        this.addDirtyRect(prevSnapshot.bounds);
        this.addDirtyRect(getElementBounds(element));
      }
    });

    // 处理新增的元素
    changes.added.forEach(element => {
      this.addDirtyRect(getElementBounds(element));
    });

    // 处理删除的元素
    changes.removed.forEach(elementId => {
      const prevSnapshot = this.previousFrame.get(elementId);
      if (prevSnapshot) {
        this.addDirtyRect(prevSnapshot.bounds);
      }
    });

    // 合并重叠的脏矩形
    this.mergeDirtyRects();

    // 更新快照
    this.updateSnapshots(elements);

    return this.dirtyRects;
  }

  // 添加脏矩形
  private addDirtyRect(rect: Rect) {
    // 扩展矩形以包含边框等效果
    const expanded = {
      x: rect.x - 5,
      y: rect.y - 5,
      width: rect.width + 10,
      height: rect.height + 10,
    };

    this.dirtyRects.push(expanded);
  }

  // 合并重叠矩形
  private mergeDirtyRects() {
    if (this.dirtyRects.length === 0) {
      return;
    }

    const merged: Rect[] = [];
    const sorted = [...this.dirtyRects].sort((a, b) => a.x - b.x);

    let current = sorted[0];

    for (let i = 1; i < sorted.length; i++) {
      const rect = sorted[i];

      if (this.rectsOverlap(current, rect)) {
        // 合并矩形
        current = this.mergeRects(current, rect);
      } else {
        merged.push(current);
        current = rect;
      }
    }

    merged.push(current);

    // 如果脏矩形覆盖超过50%的画布，直接重绘整个画布
    const totalDirtyArea = merged.reduce((sum, rect) => sum + rect.width * rect.height, 0);
    const canvasArea = this.canvasWidth * this.canvasHeight;

    if (totalDirtyArea > canvasArea * 0.5) {
      this.dirtyRects = [{
        x: 0,
        y: 0,
        width: this.canvasWidth,
        height: this.canvasHeight,
      }];
    } else {
      this.dirtyRects = merged;
    }
  }

  // 判断矩形是否重叠
  private rectsOverlap(a: Rect, b: Rect): boolean {
    return !(
      a.x + a.width < b.x ||
      b.x + b.width < a.x ||
      a.y + a.height < b.y ||
      b.y + b.height < a.y
    );
  }

  // 合并两个矩形
  private mergeRects(a: Rect, b: Rect): Rect {
    const x = Math.min(a.x, b.x);
    const y = Math.min(a.y, b.y);
    const right = Math.max(a.x + a.width, b.x + b.width);
    const bottom = Math.max(a.y + a.height, b.y + b.height);

    return {
      x,
      y,
      width: right - x,
      height: bottom - y,
    };
  }

  // 更新元素快照
  private updateSnapshots(elements: ExcalidrawElement[]) {
    this.previousFrame.clear();

    elements.forEach(element => {
      this.previousFrame.set(element.id, {
        bounds: getElementBounds(element),
        versionNonce: element.versionNonce,
      });
    });
  }
}

// 脏矩形渲染器
export class DirtyRectRenderer {
  private dirtyRectManager = new DirtyRectManager();

  public render(
    canvas: HTMLCanvasElement,
    elements: ExcalidrawElement[],
    changes: ElementChanges
  ) {
    const context = canvas.getContext('2d')!;

    // 计算脏矩形
    const dirtyRects = this.dirtyRectManager.calculateDirtyRects(elements, changes);

    if (dirtyRects.length === 0) {
      return;  // 无需重绘
    }

    // 保存上下文状态
    context.save();

    dirtyRects.forEach(rect => {
      // 设置裁剪区域
      context.beginPath();
      context.rect(rect.x, rect.y, rect.width, rect.height);
      context.clip();

      // 清空脏矩形区域
      context.clearRect(rect.x, rect.y, rect.width, rect.height);

      // 重绘脏矩形内的元素
      const elementsInRect = this.getElementsInRect(elements, rect);
      elementsInRect.forEach(element => {
        renderElement(element, context);
      });
    });

    // 恢复上下文状态
    context.restore();

    // 调试模式：绘制脏矩形边框
    if (DEBUG_MODE) {
      this.debugDrawDirtyRects(context, dirtyRects);
    }
  }

  // 获取矩形内的元素
  private getElementsInRect(elements: ExcalidrawElement[], rect: Rect): ExcalidrawElement[] {
    return elements.filter(element => {
      const bounds = getElementBounds(element);
      return this.rectIntersects(bounds, rect);
    });
  }

  // 调试：绘制脏矩形
  private debugDrawDirtyRects(context: CanvasRenderingContext2D, rects: Rect[]) {
    context.save();
    context.strokeStyle = 'red';
    context.lineWidth = 2;
    context.setLineDash([5, 5]);

    rects.forEach(rect => {
      context.strokeRect(rect.x, rect.y, rect.width, rect.height);
    });

    context.restore();
  }
}
```

## 批处理与节流

### 渲染批处理

```typescript
// packages/excalidraw/renderer/batcher.ts
export class RenderBatcher {
  private renderQueue: RenderTask[] = [];
  private isScheduled = false;
  private frameDeadline = 16;  // 16ms for 60fps

  // 添加渲染任务
  public schedule(task: RenderTask) {
    // 合并相同类型的任务
    const existingTask = this.renderQueue.find(t => t.type === task.type);

    if (existingTask) {
      existingTask.merge(task);
    } else {
      this.renderQueue.push(task);
    }

    if (!this.isScheduled) {
      this.scheduleFlush();
    }
  }

  // 调度批量执行
  private scheduleFlush() {
    this.isScheduled = true;

    requestAnimationFrame((timestamp) => {
      const deadline = timestamp + this.frameDeadline;
      this.flush(deadline);
    });
  }

  // 执行批量任务
  private flush(deadline: number) {
    const startTime = performance.now();

    // 按优先级排序任务
    this.renderQueue.sort((a, b) => b.priority - a.priority);

    // 执行任务直到超时
    while (this.renderQueue.length > 0 && performance.now() < deadline) {
      const task = this.renderQueue.shift()!;

      try {
        task.execute();
      } catch (error) {
        console.error('Render task failed:', error);
      }
    }

    // 如果还有剩余任务，继续调度
    if (this.renderQueue.length > 0) {
      this.scheduleFlush();
    } else {
      this.isScheduled = false;
    }

    // 性能监控
    const duration = performance.now() - startTime;
    if (duration > this.frameDeadline * 0.8) {
      console.warn(`Frame budget exceeded: ${duration.toFixed(2)}ms`);
    }
  }
}

// 渲染任务
class RenderTask {
  constructor(
    public type: string,
    public priority: number,
    private callback: () => void
  ) {}

  execute() {
    this.callback();
  }

  merge(other: RenderTask) {
    // 合并任务逻辑
    const originalCallback = this.callback;
    this.callback = () => {
      originalCallback();
      other.callback();
    };
  }
}
```

### 节流与防抖

```typescript
// packages/excalidraw/utils/throttle.ts
export const throttleRAF = <T extends (...args: any[]) => any>(
  fn: T,
  opts?: { trailing?: boolean }
): T => {
  let rafId: number | null = null;
  let lastArgs: Parameters<T> | null = null;

  const throttled = (...args: Parameters<T>) => {
    lastArgs = args;

    if (rafId === null) {
      rafId = requestAnimationFrame(() => {
        fn(...lastArgs!);
        rafId = null;

        // 处理尾调用
        if (opts?.trailing && lastArgs) {
          throttled(...lastArgs);
        }
      });
    }
  };

  // 添加取消方法
  (throttled as any).cancel = () => {
    if (rafId !== null) {
      cancelAnimationFrame(rafId);
      rafId = null;
    }
  };

  return throttled as T;
};

// 自适应节流
export class AdaptiveThrottle {
  private targetFPS = 60;
  private minDelay = 16;  // 60fps
  private maxDelay = 100; // 10fps
  private currentDelay = this.minDelay;
  private frameTimings: number[] = [];

  public throttle<T extends (...args: any[]) => any>(fn: T): T {
    let timeoutId: number | null = null;
    let lastCallTime = 0;

    return ((...args: Parameters<T>) => {
      const now = performance.now();
      const timeSinceLastCall = now - lastCallTime;

      // 自适应调整延迟
      this.adjustDelay();

      if (timeSinceLastCall >= this.currentDelay) {
        lastCallTime = now;
        this.measureFrame(() => fn(...args));
      } else if (!timeoutId) {
        const remainingTime = this.currentDelay - timeSinceLastCall;

        timeoutId = window.setTimeout(() => {
          lastCallTime = performance.now();
          timeoutId = null;
          this.measureFrame(() => fn(...args));
        }, remainingTime);
      }
    }) as T;
  }

  // 测量帧时间
  private measureFrame(callback: () => void) {
    const startTime = performance.now();

    callback();

    const frameTime = performance.now() - startTime;
    this.frameTimings.push(frameTime);

    // 保持最近的帧时间
    if (this.frameTimings.length > 10) {
      this.frameTimings.shift();
    }
  }

  // 自适应调整延迟
  private adjustDelay() {
    if (this.frameTimings.length === 0) {
      return;
    }

    const avgFrameTime = this.frameTimings.reduce((a, b) => a + b, 0) / this.frameTimings.length;
    const targetFrameTime = 1000 / this.targetFPS;

    if (avgFrameTime > targetFrameTime * 1.2) {
      // 性能不足，增加延迟
      this.currentDelay = Math.min(this.currentDelay * 1.5, this.maxDelay);
    } else if (avgFrameTime < targetFrameTime * 0.8) {
      // 性能充足，减少延迟
      this.currentDelay = Math.max(this.currentDelay * 0.9, this.minDelay);
    }
  }
}
```

## 内存优化策略

### 对象池

```typescript
// packages/excalidraw/utils/objectPool.ts
export class ObjectPool<T> {
  private pool: T[] = [];
  private inUse: Set<T> = new Set();
  private factory: () => T;
  private reset: (obj: T) => void;
  private maxSize: number;

  constructor(config: {
    factory: () => T;
    reset: (obj: T) => void;
    initialSize?: number;
    maxSize?: number;
  }) {
    this.factory = config.factory;
    this.reset = config.reset;
    this.maxSize = config.maxSize || 100;

    // 预创建对象
    const initialSize = config.initialSize || 10;
    for (let i = 0; i < initialSize; i++) {
      this.pool.push(this.factory());
    }
  }

  // 获取对象
  public acquire(): T {
    let obj: T;

    if (this.pool.length > 0) {
      obj = this.pool.pop()!;
    } else {
      obj = this.factory();
    }

    this.inUse.add(obj);
    return obj;
  }

  // 释放对象
  public release(obj: T) {
    if (!this.inUse.has(obj)) {
      return;
    }

    this.inUse.delete(obj);
    this.reset(obj);

    if (this.pool.length < this.maxSize) {
      this.pool.push(obj);
    }
  }

  // 批量释放
  public releaseAll() {
    this.inUse.forEach(obj => {
      this.reset(obj);
      if (this.pool.length < this.maxSize) {
        this.pool.push(obj);
      }
    });
    this.inUse.clear();
  }

  // 清空池
  public clear() {
    this.pool = [];
    this.inUse.clear();
  }

  // 获取统计信息
  public getStats() {
    return {
      poolSize: this.pool.length,
      inUseSize: this.inUse.size,
      totalCreated: this.pool.length + this.inUse.size,
    };
  }
}

// 使用示例：Path2D 对象池
const pathPool = new ObjectPool({
  factory: () => new Path2D(),
  reset: (path) => {
    // Path2D 无法重置，需要创建新的
  },
  initialSize: 20,
  maxSize: 50,
});

// 使用示例：渲染上下文状态对象池
interface RenderState {
  fillStyle: string;
  strokeStyle: string;
  lineWidth: number;
  globalAlpha: number;
}

const renderStatePool = new ObjectPool<RenderState>({
  factory: () => ({
    fillStyle: '#000000',
    strokeStyle: '#000000',
    lineWidth: 1,
    globalAlpha: 1,
  }),
  reset: (state) => {
    state.fillStyle = '#000000';
    state.strokeStyle = '#000000';
    state.lineWidth = 1;
    state.globalAlpha = 1;
  },
});
```

### 纹理图集

```typescript
// packages/excalidraw/renderer/textureAtlas.ts
export class TextureAtlas {
  private canvas: HTMLCanvasElement;
  private context: CanvasRenderingContext2D;
  private regions: Map<string, TextureRegion> = new Map();
  private packer: RectanglePacker;

  constructor(size: number = 2048) {
    this.canvas = document.createElement('canvas');
    this.canvas.width = size;
    this.canvas.height = size;
    this.context = this.canvas.getContext('2d')!;
    this.packer = new RectanglePacker(size, size);
  }

  // 添加纹理
  public addTexture(
    id: string,
    source: HTMLCanvasElement | HTMLImageElement,
    padding: number = 2
  ): TextureRegion | null {
    const width = source.width + padding * 2;
    const height = source.height + padding * 2;

    // 尝试打包
    const rect = this.packer.pack(width, height);

    if (!rect) {
      console.warn('Texture atlas is full');
      return null;
    }

    // 绘制到图集
    this.context.drawImage(
      source,
      rect.x + padding,
      rect.y + padding
    );

    // 创建纹理区域
    const region: TextureRegion = {
      id,
      x: rect.x + padding,
      y: rect.y + padding,
      width: source.width,
      height: source.height,
      u1: (rect.x + padding) / this.canvas.width,
      v1: (rect.y + padding) / this.canvas.height,
      u2: (rect.x + padding + source.width) / this.canvas.width,
      v2: (rect.y + padding + source.height) / this.canvas.height,
    };

    this.regions.set(id, region);
    return region;
  }

  // 获取纹理区域
  public getRegion(id: string): TextureRegion | undefined {
    return this.regions.get(id);
  }

  // 绘制纹理
  public drawTexture(
    context: CanvasRenderingContext2D,
    id: string,
    x: number,
    y: number,
    width?: number,
    height?: number
  ) {
    const region = this.regions.get(id);

    if (!region) {
      return;
    }

    context.drawImage(
      this.canvas,
      region.x,
      region.y,
      region.width,
      region.height,
      x,
      y,
      width || region.width,
      height || region.height
    );
  }

  // 清空图集
  public clear() {
    this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);
    this.regions.clear();
    this.packer.reset();
  }
}

// 矩形打包算法
class RectanglePacker {
  private freeRects: Rect[] = [];

  constructor(private width: number, private height: number) {
    this.reset();
  }

  pack(width: number, height: number): Rect | null {
    let bestRect: Rect | null = null;
    let bestShortSide = Number.MAX_VALUE;
    let bestLongSide = Number.MAX_VALUE;

    for (const rect of this.freeRects) {
      if (width <= rect.width && height <= rect.height) {
        const leftX = rect.width - width;
        const leftY = rect.height - height;
        const shortSide = Math.min(leftX, leftY);
        const longSide = Math.max(leftX, leftY);

        if (shortSide < bestShortSide ||
            (shortSide === bestShortSide && longSide < bestLongSide)) {
          bestRect = {
            x: rect.x,
            y: rect.y,
            width,
            height,
          };
          bestShortSide = shortSide;
          bestLongSide = longSide;
        }
      }
    }

    if (bestRect) {
      this.placeRect(bestRect);
    }

    return bestRect;
  }

  private placeRect(rect: Rect) {
    // 分割空闲矩形
    const newRects: Rect[] = [];

    for (let i = this.freeRects.length - 1; i >= 0; i--) {
      const freeRect = this.freeRects[i];

      if (this.rectsIntersect(rect, freeRect)) {
        this.freeRects.splice(i, 1);

        // 分割成最多4个新矩形
        const splits = this.splitRect(freeRect, rect);
        newRects.push(...splits);
      }
    }

    // 添加新的空闲矩形
    this.freeRects.push(...newRects);

    // 合并重叠的空闲矩形
    this.pruneRects();
  }

  reset() {
    this.freeRects = [{
      x: 0,
      y: 0,
      width: this.width,
      height: this.height,
    }];
  }
}
```

## 实战示例：高性能画板实现

### 完整的优化画板

```javascript
class OptimizedDrawingBoard {
  constructor(container) {
    this.container = container;
    this.initializeLayers();
    this.initializeOptimizers();
    this.setupEventHandlers();
  }

  initializeLayers() {
    // 创建分层 Canvas
    this.layers = {
      background: this.createCanvas('background', 0),
      static: this.createCanvas('static', 1),
      dynamic: this.createCanvas('dynamic', 2),
      ui: this.createCanvas('ui', 3),
    };

    // 获取上下文
    this.contexts = {};
    Object.keys(this.layers).forEach(key => {
      this.contexts[key] = this.layers[key].getContext('2d');
    });
  }

  initializeOptimizers() {
    // 脏矩形管理器
    this.dirtyRectManager = new DirtyRectManager();

    // 渲染批处理器
    this.renderBatcher = new RenderBatcher();

    // 虚拟化管理器
    this.virtualizer = new RenderVirtualizer();

    // 对象池
    this.pathPool = new ObjectPool({
      factory: () => new Path2D(),
      reset: () => {},
      maxSize: 100,
    });

    // 纹理图集
    this.textureAtlas = new TextureAtlas(2048);

    // 性能监控
    this.performanceMonitor = new PerformanceMonitor();
  }

  createCanvas(name, zIndex) {
    const canvas = document.createElement('canvas');
    canvas.className = `canvas-${name}`;
    canvas.style.position = 'absolute';
    canvas.style.zIndex = zIndex;
    canvas.width = 1920;
    canvas.height = 1080;
    this.container.appendChild(canvas);
    return canvas;
  }

  // 主渲染函数
  render(elements, appState, changes) {
    this.performanceMonitor.startFrame();

    // 获取视口内的可见元素
    const visibleElements = this.virtualizer.getVisibleElements(
      elements,
      appState.viewport
    );

    // 分离静态和动态元素
    const { staticElements, dynamicElements } = this.separateElements(
      visibleElements,
      appState
    );

    // 批量渲染任务
    this.renderBatcher.schedule({
      type: 'background',
      priority: 1,
      execute: () => this.renderBackground(appState),
    });

    this.renderBatcher.schedule({
      type: 'static',
      priority: 2,
      execute: () => this.renderStaticLayer(staticElements, changes),
    });

    this.renderBatcher.schedule({
      type: 'dynamic',
      priority: 3,
      execute: () => this.renderDynamicLayer(dynamicElements),
    });

    this.renderBatcher.schedule({
      type: 'ui',
      priority: 4,
      execute: () => this.renderUILayer(appState),
    });

    this.performanceMonitor.endFrame();
  }

  // 渲染静态层（使用脏矩形）
  renderStaticLayer(elements, changes) {
    const ctx = this.contexts.static;

    if (!changes || changes.fullRedraw) {
      // 完全重绘
      ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
      elements.forEach(element => this.renderElement(ctx, element));
    } else {
      // 脏矩形渲染
      const dirtyRects = this.dirtyRectManager.calculateDirtyRects(
        elements,
        changes
      );

      dirtyRects.forEach(rect => {
        ctx.save();
        ctx.beginPath();
        ctx.rect(rect.x, rect.y, rect.width, rect.height);
        ctx.clip();

        ctx.clearRect(rect.x, rect.y, rect.width, rect.height);

        const elementsInRect = this.getElementsInRect(elements, rect);
        elementsInRect.forEach(element => this.renderElement(ctx, element));

        ctx.restore();
      });
    }
  }

  // 渲染单个元素（使用缓存）
  renderElement(ctx, element) {
    // 尝试从纹理图集获取
    const textureRegion = this.textureAtlas.getRegion(element.id);

    if (textureRegion && !element.isDirty) {
      // 使用缓存的纹理
      this.textureAtlas.drawTexture(
        ctx,
        element.id,
        element.x,
        element.y
      );
    } else {
      // 渲染并缓存
      const offscreen = this.renderElementToOffscreen(element);
      this.textureAtlas.addTexture(element.id, offscreen);
      ctx.drawImage(offscreen, element.x, element.y);
    }
  }

  // 渲染到离屏 Canvas
  renderElementToOffscreen(element) {
    const offscreen = document.createElement('canvas');
    offscreen.width = element.width;
    offscreen.height = element.height;
    const ctx = offscreen.getContext('2d');

    // 使用对象池中的 Path2D
    const path = this.pathPool.acquire();

    // 绘制元素
    this.drawElementPath(path, element);
    ctx.stroke(path);

    // 释放 Path2D
    this.pathPool.release(path);

    return offscreen;
  }

  // 性能监控
  getPerformanceStats() {
    return this.performanceMonitor.getStats();
  }
}

// 性能监控器
class PerformanceMonitor {
  constructor() {
    this.frameTimings = [];
    this.maxSamples = 60;
  }

  startFrame() {
    this.frameStartTime = performance.now();
  }

  endFrame() {
    const frameTime = performance.now() - this.frameStartTime;
    this.frameTimings.push(frameTime);

    if (this.frameTimings.length > this.maxSamples) {
      this.frameTimings.shift();
    }
  }

  getStats() {
    if (this.frameTimings.length === 0) {
      return null;
    }

    const sorted = [...this.frameTimings].sort((a, b) => a - b);
    const sum = sorted.reduce((a, b) => a + b, 0);

    return {
      fps: Math.round(1000 / (sum / sorted.length)),
      avg: sum / sorted.length,
      min: sorted[0],
      max: sorted[sorted.length - 1],
      p50: sorted[Math.floor(sorted.length * 0.5)],
      p95: sorted[Math.floor(sorted.length * 0.95)],
      p99: sorted[Math.floor(sorted.length * 0.99)],
    };
  }
}

// 使用示例
const board = new OptimizedDrawingBoard(document.getElementById('container'));

// 添加大量元素进行测试
const elements = [];
for (let i = 0; i < 1000; i++) {
  elements.push({
    id: `element-${i}`,
    type: 'rectangle',
    x: Math.random() * 1920,
    y: Math.random() * 1080,
    width: 50 + Math.random() * 150,
    height: 50 + Math.random() * 150,
    color: `hsl(${Math.random() * 360}, 70%, 50%)`,
  });
}

// 渲染
board.render(elements, { viewport: { x: 0, y: 0, width: 1920, height: 1080 } });

// 显示性能统计
setInterval(() => {
  const stats = board.getPerformanceStats();
  console.log(`FPS: ${stats.fps}, Avg: ${stats.avg.toFixed(2)}ms`);
}, 1000);
```

## 思考题

1. **如何平衡渲染质量和性能？** 在不同设备上自动调整渲染策略。

2. **脏矩形算法的局限性在哪里？** 什么情况下脏矩形反而会降低性能？

3. **如何实现更智能的缓存淘汰策略？** 基于使用频率和重要性的 LRU-K 算法。

4. **WebGL 渲染的优势和挑战？** 评估迁移到 WebGL 的成本收益。

5. **如何处理极端场景？** 10000+ 元素时的优化策略。

## 实践练习

### 练习 1：实现自适应渲染
设计一个自适应渲染系统：
- 根据设备性能动态调整渲染质量
- 实时监控帧率并自动优化
- 提供用户可配置的性能选项

### 练习 2：优化移动端性能
针对移动设备优化：
- 触摸事件优化
- 降低渲染分辨率
- 简化视觉效果

### 练习 3：实现渐进式渲染
实现渐进式加载和渲染：
- 优先渲染视口中心
- 延迟加载视口外内容
- 分级细节渲染

## 总结

Excalidraw 的渲染优化策略展示了如何在 Web 平台实现高性能图形应用：

1. **分层渲染**：通过合理的层级划分减少不必要的重绘
2. **虚拟化技术**：只渲染可见内容，支持大规模场景
3. **脏矩形算法**：精确追踪变化区域，最小化重绘范围
4. **批处理机制**：合并渲染操作，减少 API 调用开销
5. **内存优化**：通过对象池、纹理图集等技术降低内存压力
6. **自适应策略**：根据运行时性能动态调整渲染策略

这些优化技术的综合运用，使得 Excalidraw 能够在各种设备和场景下都提供流畅的用户体验。

## 下一步

完成了渲染系统的学习后，下一章将深入探讨 Excalidraw 的交互系统实现，包括工具系统设计、手势处理、选择与变换等核心交互功能的实现原理。