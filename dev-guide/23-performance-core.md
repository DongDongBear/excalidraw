# 7.2 性能优化核心：构建高效绘图引擎

在最小核心基础上，本节将重点介绍 Excalidraw 的性能优化核心技术，帮你构建一个高效的绘图引擎。

## 性能优化策略概览

### 1. 渲染优化核心

```typescript
interface RenderOptimization {
  // 脏区域跟踪
  dirtyRegions: Set<BoundingBox>;

  // 视口剔除
  viewportBounds: BoundingBox;

  // 渲染缓存
  renderCache: Map<string, ImageData>;

  // 批量渲染
  renderBatch: ExcalidrawElement[];
}

class OptimizedRenderer {
  private dirtyRegions = new Set<BoundingBox>();
  private renderCache = new Map<string, OffscreenCanvas>();
  private lastViewport: BoundingBox | null = null;

  // 智能重绘：只重绘变化区域
  render(elements: ExcalidrawElement[], viewport: BoundingBox): void {
    // 检查视口是否变化
    const viewportChanged = !this.lastViewport ||
      !this.boundingBoxEquals(viewport, this.lastViewport);

    if (viewportChanged) {
      // 视口变化，需要全量重绘
      this.renderFullViewport(elements, viewport);
      this.lastViewport = { ...viewport };
      this.dirtyRegions.clear();
      return;
    }

    // 增量渲染：只重绘脏区域
    if (this.dirtyRegions.size > 0) {
      this.renderDirtyRegions(elements, viewport);
      this.dirtyRegions.clear();
    }
  }

  // 标记脏区域
  markDirty(element: ExcalidrawElement): void {
    const bounds = this.getElementBounds(element);
    this.dirtyRegions.add(bounds);

    // 合并相邻的脏区域以减少重绘次数
    this.coalesceDirtyRegions();
  }
}
```

### 2. 视口剔除优化

```typescript
class ViewportCuller {
  // 高效的视口剔除算法
  cullElements(elements: ExcalidrawElement[], viewport: BoundingBox): ExcalidrawElement[] {
    const visibleElements: ExcalidrawElement[] = [];
    const expandedViewport = this.expandBounds(viewport, 100); // 100px 缓冲区

    for (const element of elements) {
      if (this.isElementInViewport(element, expandedViewport)) {
        visibleElements.push(element);
      }
    }

    return visibleElements;
  }

  // 快速边界框相交检测
  private isElementInViewport(element: ExcalidrawElement, viewport: BoundingBox): boolean {
    const elementBounds = this.getElementBounds(element);

    return !(
      elementBounds.x > viewport.x + viewport.width ||
      elementBounds.x + elementBounds.width < viewport.x ||
      elementBounds.y > viewport.y + viewport.height ||
      elementBounds.y + elementBounds.height < viewport.y
    );
  }

  // 扩展边界框以支持缓冲区
  private expandBounds(bounds: BoundingBox, buffer: number): BoundingBox {
    return {
      x: bounds.x - buffer,
      y: bounds.y - buffer,
      width: bounds.width + 2 * buffer,
      height: bounds.height + 2 * buffer
    };
  }
}
```

## 高级缓存系统

### 1. 多层次缓存架构

```typescript
class MultiLevelCache {
  // L1: 元素级缓存（最快）
  private elementCache = new Map<string, OffscreenCanvas>();

  // L2: 区域级缓存（中等速度）
  private regionCache = new Map<string, OffscreenCanvas>();

  // L3: 全局缓存（最慢，但覆盖面大）
  private globalCache: OffscreenCanvas | null = null;

  // 获取缓存的渲染结果
  getCachedRender(
    element: ExcalidrawElement,
    renderingContext: RenderingContext
  ): OffscreenCanvas | null {
    const cacheKey = this.generateCacheKey(element, renderingContext);

    // 尝试从 L1 缓存获取
    const l1Cache = this.elementCache.get(cacheKey);
    if (l1Cache) {
      this.updateCacheStats('l1', 'hit');
      return l1Cache;
    }

    // 尝试从 L2 缓存获取
    const regionKey = this.getRegionKey(element);
    const l2Cache = this.regionCache.get(regionKey);
    if (l2Cache) {
      this.updateCacheStats('l2', 'hit');
      return this.extractFromRegionCache(l2Cache, element);
    }

    this.updateCacheStats('miss');
    return null;
  }

  // 缓存渲染结果
  cacheRender(
    element: ExcalidrawElement,
    renderingContext: RenderingContext,
    renderedCanvas: OffscreenCanvas
  ): void {
    const cacheKey = this.generateCacheKey(element, renderingContext);

    // 根据元素大小决定缓存策略
    const elementSize = element.width * element.height;

    if (elementSize < 1000) {
      // 小元素：缓存到 L1
      this.elementCache.set(cacheKey, renderedCanvas);
    } else if (elementSize < 10000) {
      // 中等元素：缓存到 L2
      const regionKey = this.getRegionKey(element);
      this.cacheToRegion(regionKey, element, renderedCanvas);
    }
    // 大元素：不缓存，避免内存溢出

    // 内存管理
    this.manageMemoryUsage();
  }

  private manageMemoryUsage(): void {
    const maxCacheSize = 100 * 1024 * 1024; // 100MB
    let currentSize = this.calculateCacheSize();

    if (currentSize > maxCacheSize) {
      // LRU 淘汰策略
      this.evictLeastRecentlyUsed();
    }
  }
}
```

### 2. 智能预渲染

```typescript
class PreRenderer {
  private renderQueue: Array<{
    element: ExcalidrawElement;
    priority: number;
    estimatedCost: number;
  }> = [];

  private isPreRendering = false;

  // 预渲染调度器
  schedulePreRender(elements: ExcalidrawElement[], viewport: BoundingBox): void {
    // 清空旧的预渲染队列
    this.renderQueue = [];

    // 分析元素渲染优先级
    elements.forEach(element => {
      const priority = this.calculatePriority(element, viewport);
      const cost = this.estimateRenderCost(element);

      if (priority > 0) {
        this.renderQueue.push({
          element,
          priority,
          estimatedCost: cost
        });
      }
    });

    // 按优先级排序
    this.renderQueue.sort((a, b) => b.priority - a.priority);

    // 开始预渲染
    if (!this.isPreRendering) {
      this.startPreRendering();
    }
  }

  private async startPreRendering(): Promise<void> {
    this.isPreRendering = true;

    while (this.renderQueue.length > 0) {
      const task = this.renderQueue.shift()!;

      // 使用 RequestIdleCallback 在空闲时预渲染
      await this.waitForIdleTime();

      // 检查元素是否仍然需要预渲染
      if (this.shouldPreRender(task.element)) {
        await this.preRenderElement(task.element);
      }
    }

    this.isPreRendering = false;
  }

  private waitForIdleTime(): Promise<void> {
    return new Promise(resolve => {
      if ('requestIdleCallback' in window) {
        requestIdleCallback(() => resolve());
      } else {
        // 降级到 setTimeout
        setTimeout(() => resolve(), 0);
      }
    });
  }
}
```

## 事件处理优化

### 1. 高效事件分发

```typescript
class OptimizedEventHandler {
  private eventQueue: Array<{
    event: PointerEvent;
    timestamp: number;
    processed: boolean;
  }> = [];

  private throttledHandlers = new Map<string, {
    handler: Function;
    lastCall: number;
    throttleMs: number;
  }>();

  // 事件节流处理
  handlePointerEvent(event: PointerEvent): void {
    const eventType = event.type;

    // 对高频事件进行节流
    if (this.isHighFrequencyEvent(eventType)) {
      this.throttleEvent(eventType, event, () => {
        this.processEvent(event);
      });
    } else {
      this.processEvent(event);
    }
  }

  private throttleEvent(
    eventType: string,
    event: PointerEvent,
    handler: () => void
  ): void {
    const throttleConfig = this.getThrottleConfig(eventType);
    const now = performance.now();

    const existing = this.throttledHandlers.get(eventType);
    if (!existing || now - existing.lastCall >= throttleConfig.throttleMs) {
      handler();

      this.throttledHandlers.set(eventType, {
        handler,
        lastCall: now,
        throttleMs: throttleConfig.throttleMs
      });
    }
  }

  private getThrottleConfig(eventType: string): { throttleMs: number } {
    switch (eventType) {
      case 'pointermove':
        return { throttleMs: 16 }; // ~60fps
      case 'wheel':
        return { throttleMs: 8 }; // ~120fps for smooth scrolling
      default:
        return { throttleMs: 0 }; // No throttling
    }
  }
}
```

### 2. 批量状态更新

```typescript
class BatchedStateUpdater {
  private pendingUpdates: Array<{
    elements?: ExcalidrawElement[];
    appState?: Partial<AppState>;
    priority: 'low' | 'medium' | 'high';
  }> = [];

  private updateScheduled = false;

  // 批量更新入口
  queueUpdate(
    elements?: ExcalidrawElement[],
    appState?: Partial<AppState>,
    priority: 'low' | 'medium' | 'high' = 'medium'
  ): void {
    this.pendingUpdates.push({
      elements,
      appState,
      priority
    });

    if (!this.updateScheduled) {
      this.scheduleUpdate();
    }
  }

  private scheduleUpdate(): void {
    this.updateScheduled = true;

    // 根据优先级决定调度策略
    const hasHighPriority = this.pendingUpdates.some(u => u.priority === 'high');

    if (hasHighPriority) {
      // 高优先级：立即执行
      Promise.resolve().then(() => this.flushUpdates());
    } else {
      // 低优先级：下一个动画帧执行
      requestAnimationFrame(() => this.flushUpdates());
    }
  }

  private flushUpdates(): void {
    if (this.pendingUpdates.length === 0) {
      this.updateScheduled = false;
      return;
    }

    // 合并所有待处理的更新
    const mergedUpdate = this.mergeUpdates(this.pendingUpdates);

    // 清空队列
    this.pendingUpdates = [];
    this.updateScheduled = false;

    // 执行合并后的更新
    this.executeUpdate(mergedUpdate);
  }

  private mergeUpdates(
    updates: Array<{
      elements?: ExcalidrawElement[];
      appState?: Partial<AppState>;
      priority: 'low' | 'medium' | 'high';
    }>
  ): { elements?: ExcalidrawElement[]; appState?: Partial<AppState> } {
    let mergedElements: ExcalidrawElement[] | undefined;
    let mergedAppState: Partial<AppState> = {};

    // 按优先级和时间顺序合并更新
    const sortedUpdates = updates.sort((a, b) => {
      const priorityWeight = { high: 3, medium: 2, low: 1 };
      return priorityWeight[b.priority] - priorityWeight[a.priority];
    });

    for (const update of sortedUpdates) {
      if (update.elements) {
        mergedElements = update.elements; // 最新的元素状态覆盖旧的
      }

      if (update.appState) {
        mergedAppState = { ...mergedAppState, ...update.appState };
      }
    }

    return {
      elements: mergedElements,
      appState: mergedAppState
    };
  }
}
```

## 内存管理优化

### 1. 对象池模式

```typescript
class ObjectPool<T> {
  private pool: T[] = [];
  private createFn: () => T;
  private resetFn: (obj: T) => void;
  private maxSize: number;

  constructor(
    createFn: () => T,
    resetFn: (obj: T) => void,
    maxSize: number = 100
  ) {
    this.createFn = createFn;
    this.resetFn = resetFn;
    this.maxSize = maxSize;
  }

  acquire(): T {
    if (this.pool.length > 0) {
      return this.pool.pop()!;
    }
    return this.createFn();
  }

  release(obj: T): void {
    if (this.pool.length < this.maxSize) {
      this.resetFn(obj);
      this.pool.push(obj);
    }
  }
}

// 使用对象池管理临时对象
class PerformanceOptimizedExcalidraw {
  private pointPool = new ObjectPool<Point>(
    () => ({ x: 0, y: 0 }),
    (point) => { point.x = 0; point.y = 0; },
    1000
  );

  private boundingBoxPool = new ObjectPool<BoundingBox>(
    () => ({ x: 0, y: 0, width: 0, height: 0 }),
    (box) => { box.x = 0; box.y = 0; box.width = 0; box.height = 0; },
    500
  );

  // 使用对象池避免频繁的内存分配
  private calculateBounds(elements: ExcalidrawElement[]): BoundingBox {
    const bounds = this.boundingBoxPool.acquire();

    try {
      // 计算边界框逻辑...
      this.computeBoundingBox(elements, bounds);

      // 创建返回值的副本
      return { ...bounds };
    } finally {
      // 归还到对象池
      this.boundingBoxPool.release(bounds);
    }
  }
}
```

### 2. 智能垃圾回收

```typescript
class MemoryManager {
  private memoryUsage = {
    elements: 0,
    cache: 0,
    textures: 0,
    total: 0
  };

  private gcThreshold = 50 * 1024 * 1024; // 50MB

  // 监控内存使用
  trackMemoryUsage(): void {
    if ('memory' in performance) {
      const memInfo = (performance as any).memory;
      this.memoryUsage.total = memInfo.usedJSHeapSize;

      // 当内存使用超过阈值时触发清理
      if (this.memoryUsage.total > this.gcThreshold) {
        this.performMemoryCleanup();
      }
    }
  }

  private performMemoryCleanup(): void {
    // 清理缓存
    this.cleanupCache();

    // 清理未使用的纹理
    this.cleanupTextures();

    // 建议浏览器进行垃圾回收
    if ('gc' in window && typeof (window as any).gc === 'function') {
      (window as any).gc();
    }
  }

  private cleanupCache(): void {
    // 实现 LRU 缓存清理逻辑
    const cacheEntries = Array.from(this.cache.entries())
      .sort((a, b) => a[1].lastAccess - b[1].lastAccess);

    // 清理最旧的 25% 缓存条目
    const entriesToRemove = Math.floor(cacheEntries.length * 0.25);

    for (let i = 0; i < entriesToRemove; i++) {
      this.cache.delete(cacheEntries[i][0]);
    }
  }
}
```

## 性能监控和调试

### 1. 性能指标收集

```typescript
class PerformanceMonitor {
  private metrics = {
    renderTime: new Array<number>(),
    eventProcessingTime: new Array<number>(),
    memoryUsage: new Array<number>(),
    fps: new Array<number>()
  };

  private frameCount = 0;
  private lastFrameTime = 0;

  // 渲染性能测量
  measureRenderPerformance<T>(renderFn: () => T): T {
    const startTime = performance.now();
    const result = renderFn();
    const endTime = performance.now();

    const renderTime = endTime - startTime;
    this.metrics.renderTime.push(renderTime);

    // 保持最近 100 个样本
    if (this.metrics.renderTime.length > 100) {
      this.metrics.renderTime.shift();
    }

    // 计算 FPS
    this.calculateFPS(endTime);

    return result;
  }

  private calculateFPS(currentTime: number): void {
    this.frameCount++;

    if (this.lastFrameTime === 0) {
      this.lastFrameTime = currentTime;
      return;
    }

    const deltaTime = currentTime - this.lastFrameTime;

    if (deltaTime >= 1000) { // 每秒计算一次
      const fps = (this.frameCount * 1000) / deltaTime;
      this.metrics.fps.push(fps);

      // 重置计数器
      this.frameCount = 0;
      this.lastFrameTime = currentTime;

      // 保持最近 60 个样本
      if (this.metrics.fps.length > 60) {
        this.metrics.fps.shift();
      }
    }
  }

  // 获取性能报告
  getPerformanceReport(): PerformanceReport {
    return {
      averageRenderTime: this.getAverage(this.metrics.renderTime),
      averageFPS: this.getAverage(this.metrics.fps),
      p95RenderTime: this.getPercentile(this.metrics.renderTime, 95),
      memoryTrend: this.getMemoryTrend()
    };
  }
}
```

这些性能优化技术是 Excalidraw 能够处理大型画布和复杂图形的关键。通过智能缓存、视口剔除、批量更新和内存管理，我们构建了一个高效的绘图引擎核心。