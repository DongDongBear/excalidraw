# Chapter 3.1: 渲染引擎核心架构解析

## 概述

Excalidraw 的渲染引擎是整个应用性能的关键所在。它采用了精心设计的双 Canvas 架构，结合多层缓存机制和智能视口管理，实现了在复杂场景下依然保持 60fps 流畅度的目标。本章将深入解析这个高性能渲染引擎的实现原理。

## 渲染架构总览

### 双 Canvas 架构设计

Excalidraw 采用了创新的双 Canvas 架构，将渲染内容分为两层：

```
┌─────────────────────────────────────────────┐
│           Interactive Canvas (上层)          │
│  选择框 | 变换手柄 | 光标 | 临时UI元素        │
└─────────────────────────────────────────────┘
                      ↑
                  用户交互
                      ↓
┌─────────────────────────────────────────────┐
│             Static Canvas (下层)            │
│  图形元素 | 文本 | 图片 | 背景网格            │
└─────────────────────────────────────────────┘
```

### 为什么采用双 Canvas？

```javascript
// 传统单 Canvas 方案的问题
function renderScene(canvas) {
  // 每次交互都需要重绘所有内容
  clearCanvas();
  renderBackground();    // 背景
  renderElements();      // 所有元素
  renderSelection();     // 选择框
  renderHandles();       // 变换手柄
  // 性能开销大，特别是元素数量多时
}

// Excalidraw 的双 Canvas 方案
function renderStaticCanvas() {
  // 只在元素变化时重绘
  if (elementsChanged) {
    renderElements();  // 静态内容
  }
}

function renderInteractiveCanvas() {
  // 高频重绘，但内容少
  clearCanvas();
  renderSelection();   // 仅UI元素
  renderHandles();
}
```

## Renderer 类详解

### 核心职责

`Renderer` 类是渲染引擎的核心管理器，负责：
1. 管理可渲染元素列表
2. 视口裁剪优化
3. 缓存管理
4. 渲染状态追踪

### 源码分析

```typescript
// packages/excalidraw/scene/Renderer.ts - 实际源码
export class Renderer {
  private scene: Scene;

  constructor(scene: Scene) {
    this.scene = scene;
  }

  // 核心方法：获取可渲染元素（带缓存）
  public getRenderableElements = (() => {
    // 内部函数：获取视口内可见元素
    const getVisibleCanvasElements = ({
      elementsMap,
      zoom,
      offsetLeft,
      offsetTop,
      scrollX,
      scrollY,
      height,
      width,
    }: {
      elementsMap: NonDeletedElementsMap;
      zoom: AppState["zoom"];
      offsetLeft: AppState["offsetLeft"];
      offsetTop: AppState["offsetTop"];
      scrollX: AppState["scrollX"];
      scrollY: AppState["scrollY"];
      height: AppState["height"];
      width: AppState["width"];
    }): readonly NonDeletedExcalidrawElement[] => {
      const visibleElements: NonDeletedExcalidrawElement[] = [];

      // 视口剔除：只保留在视口内的元素
      for (const element of elementsMap.values()) {
        if (
          isElementInViewport(
            element,
            width,
            height,
            { zoom, offsetLeft, offsetTop, scrollX, scrollY },
            elementsMap,
          )
        ) {
          visibleElements.push(element);
        }
      }
      return visibleElements;
    };

    // 内部函数：过滤可渲染元素
    const getRenderableElements = ({
      elements,
      editingTextElement,
      newElementId,
    }: {
      elements: readonly NonDeletedExcalidrawElement[];
      editingTextElement: AppState["editingTextElement"];
      newElementId: ExcalidrawElement["id"] | undefined;
    }) => {
      const elementsMap = toBrandedType<RenderableElementsMap>(new Map());

      for (const element of elements) {
        // 跳过正在创建的新元素
        if (newElementId === element.id) {
          continue;
        }

        // 跳过正在编辑的文本元素（远程协作时渲染）
        // 源码注释：we don't want to render text element that's being currently edited
        //          (it's rendered on remote only)
        if (
          !editingTextElement ||
          editingTextElement.type !== "text" ||
          element.id !== editingTextElement.id
        ) {
          elementsMap.set(element.id, element);
        }
      }
      return elementsMap;
    };

    // 返回带缓存的处理函数
    return memoize(({
      zoom,
      offsetLeft,
      offsetTop,
      scrollX,
      scrollY,
      height,
      width,
      editingTextElement,
      newElementId,
      sceneNonce: _sceneNonce, // 缓存失效随机数
    }: {
      zoom: AppState["zoom"];
      offsetLeft: AppState["offsetLeft"];
      offsetTop: AppState["offsetTop"];
      scrollX: AppState["scrollX"];
      scrollY: AppState["scrollY"];
      height: AppState["height"];
      width: AppState["width"];
      editingTextElement: AppState["editingTextElement"];
      /** 源码注释：first render of newElement will always bust the cache */
      newElementId: ExcalidrawElement["id"] | undefined;
      sceneNonce: ReturnType<InstanceType<typeof Scene>["getSceneNonce"]>;
    }) => {
      // 从场景获取非删除元素
      const elements = this.scene.getNonDeletedElements();

      // 过滤可渲染元素
      const elementsMap = getRenderableElements({
        elements,
        editingTextElement,
        newElementId,
      });

      // 视口剔除
      const visibleElements = getVisibleCanvasElements({
        elementsMap,
        zoom,
        offsetLeft,
        offsetTop,
        scrollX,
        scrollY,
        height,
        width,
      });

      return { elementsMap, visibleElements };
    });
  })();

  // 销毁方法：清理缓存和取消节流
  // NOTE: Doesn't destroy everything (scene, rc, etc.) because it may not be
  //       safe to break TS contract here (for upstream cases)
  public destroy() {
    renderInteractiveSceneThrottled.cancel();
    renderStaticSceneThrottled.cancel();
    this.getRenderableElements.clear();
  }
}
```

### 缓存机制分析

```typescript
// memoize 实现原理
function memoize<T extends (...args: any[]) => any>(fn: T): T {
  const cache = new Map();

  return ((...args: Parameters<T>) => {
    // 生成缓存键（基于参数）
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      return cache.get(key);  // 命中缓存
    }

    const result = fn(...args);
    cache.set(key, result);   // 存入缓存

    return result;
  }) as T;
}

// 缓存失效条件
// 1. 视口参数变化（zoom, scroll, offset）
// 2. 元素状态变化（sceneNonce 递增）
// 3. 编辑状态变化（editingTextElement）
```

## 视口裁剪系统

### 视口坐标系

```javascript
// 三个坐标系的转换关系
// 1. 屏幕坐标（鼠标/触摸事件）
// 2. 视口坐标（Canvas 坐标）
// 3. 场景坐标（元素实际位置）

// 屏幕坐标 → 视口坐标
function screenToViewport(screenX, screenY, canvas) {
  const rect = canvas.getBoundingClientRect();
  return {
    x: screenX - rect.left,
    y: screenY - rect.top,
  };
}

// 视口坐标 → 场景坐标
function viewportToScene(viewportX, viewportY, appState) {
  const { zoom, scrollX, scrollY, offsetLeft, offsetTop } = appState;
  return {
    x: (viewportX - offsetLeft) / zoom.value - scrollX,
    y: (viewportY - offsetTop) / zoom.value - scrollY,
  };
}

// 场景坐标 → 视口坐标
function sceneToViewport(sceneX, sceneY, appState) {
  const { zoom, scrollX, scrollY, offsetLeft, offsetTop } = appState;
  return {
    x: (sceneX + scrollX) * zoom.value + offsetLeft,
    y: (sceneY + scrollY) * zoom.value + offsetTop,
  };
}
```

### 元素可见性判断

```typescript
// packages/element/src/sizeHelpers.ts
export const isElementInViewport = (
  element: ExcalidrawElement,
  width: number,
  height: number,
  viewTransformations: {
    zoom: Zoom;
    offsetLeft: number;
    offsetTop: number;
    scrollX: number;
    scrollY: number;
  },
  elementsMap: ElementsMap,
): boolean => {
  // 1. 获取元素边界（考虑旋转）
  const [x1, y1, x2, y2] = getElementBounds(element, elementsMap);

  // 2. 计算视口在场景坐标系中的边界
  const topLeftSceneCoords = viewportCoordsToSceneCoords(
    { clientX: 0, clientY: 0 },
    viewTransformations,
  );

  const bottomRightSceneCoords = viewportCoordsToSceneCoords(
    { clientX: width, clientY: height },
    viewTransformations,
  );

  // 3. AABB 碰撞检测
  return (
    topLeftSceneCoords.x <= x2 &&
    topLeftSceneCoords.y <= y2 &&
    bottomRightSceneCoords.x >= x1 &&
    bottomRightSceneCoords.y >= y1
  );
};
```

### 视口裁剪优化效果

```javascript
// 性能对比示例
class PerformanceComparison {
  // 无视口裁剪：渲染所有元素
  renderWithoutCulling(elements) {
    const startTime = performance.now();

    elements.forEach(element => {
      renderElement(element);  // 渲染每个元素
    });

    const endTime = performance.now();
    console.log(`渲染 ${elements.length} 个元素，耗时：${endTime - startTime}ms`);
    // 1000个元素：约 50ms
  }

  // 有视口裁剪：只渲染可见元素
  renderWithCulling(elements, viewport) {
    const startTime = performance.now();

    const visibleElements = elements.filter(element =>
      isElementInViewport(element, viewport)
    );

    visibleElements.forEach(element => {
      renderElement(element);
    });

    const endTime = performance.now();
    console.log(`渲染 ${visibleElements.length}/${elements.length} 个元素，耗时：${endTime - startTime}ms`);
    // 1000个元素，可见50个：约 3ms
  }
}
```

## 场景图管理

### Scene 类架构（已验证）

```typescript
// packages/element/src/Scene.ts（实际源码位置 ✓）
export class Scene {
  // 实际的数据结构（已验证 ✓）
  private callbacks: Set<SceneStateCallback> = new Set();
  private nonDeletedElements: readonly Ordered<NonDeletedExcalidrawElement>[] = [];
  private nonDeletedElementsMap = toBrandedType<NonDeletedSceneElementsMap>(new Map());
  private elements: readonly OrderedExcalidrawElement[] = [];
  private nonDeletedFramesLikes: readonly NonDeleted<ExcalidrawFrameLikeElement>[] = [];
  private frames: readonly ExcalidrawFrameLikeElement[] = [];
  private elementsMap = toBrandedType<SceneElementsMap>(new Map());

  /**
   * Random integer regenerated each scene update.
   *
   * Does not relate to elements versions, it's only a renderer
   * cache-invalidation nonce at the moment.
   */
  private sceneNonce: number | undefined;

  constructor(
    elements: ElementsMapOrArray | null = null,
    options?: { skipValidation?: true; },
  ) {
    if (elements) {
      this.replaceAllElements(elements, options);
    }
  }

  // 场景版本管理（缓存失效机制）
  getSceneNonce() {
    return this.sceneNonce;
  }

  // 元素查询方法
  getNonDeletedElements() {
    return this.nonDeletedElements;
  }

  getNonDeletedElementsMap() {
    return this.nonDeletedElementsMap;
  }

  getElement<T extends ExcalidrawElement>(id: T["id"]): T | null {
    return (this.elementsMap.get(id) as T | undefined) || null;
  }

  getNonDeletedElement(
    id: ExcalidrawElement["id"],
  ): NonDeleted<ExcalidrawElement> | null {
    const element = this.getElement(id);
    if (element && isNonDeletedElement(element)) {
      return element;
    }
    return null;
  }

  // 批量更新元素（核心方法）
  replaceAllElements(
    nextElements: ElementsMapOrArray,
    options?: { skipValidation?: true; },
  ) {
    const _nextElements = toArray(nextElements);
    const nextFrameLikes: ExcalidrawFrameLikeElement[] = [];

    // 验证分数索引（用于元素排序）
    if (!options?.skipValidation) {
      validateIndicesThrottled(_nextElements);
    }

    // 同步无效索引
    this.elements = syncInvalidIndices(_nextElements);
    this.elementsMap.clear();

    this.elements.forEach((element) => {
      if (isFrameLikeElement(element)) {
        nextFrameLikes.push(element);
      }
      this.elementsMap.set(element.id, element);
    });

    // 构建非删除元素映射
    const nonDeletedElements = getNonDeletedElements(this.elements);
    this.nonDeletedElements = nonDeletedElements.elements;
    this.nonDeletedElementsMap = nonDeletedElements.elementsMap;

    this.frames = nextFrameLikes;
    this.nonDeletedFramesLikes = getNonDeletedElements(this.frames).elements;

    // 触发更新回调（重要：这会递增 sceneNonce）
    this.triggerUpdate();
  }

  // 触发场景更新（递增 sceneNonce）
  triggerUpdate() {
    this.sceneNonce = randomInteger(); // 生成随机整数作为新的 nonce

    // 通知所有注册的回调
    for (const callback of Array.from(this.callbacks)) {
      callback();
    }
  }

  // 注册更新回调
  onUpdate(cb: SceneStateCallback): SceneStateCallbackRemover {
    if (this.callbacks.has(cb)) {
      throw new Error();
    }
    this.callbacks.add(cb);
    return () => {
      if (!this.callbacks.has(cb)) {
        throw new Error();
      }
      this.callbacks.delete(cb);
    };
  }
}
```

**关键发现：**

1. **sceneNonce 的实际类型**：`number | undefined`，而不是初始化为 0
2. **使用 `randomInteger()` 生成 nonce**：每次更新时生成随机整数，而不是简单递增
3. **更多的数据结构**：除了基本的 elements，还包括 frames、callbacks 等
4. **fractional indices 验证**：使用 `validateIndicesThrottled` 进行分数索引验证
5. **回调机制**：通过 `callbacks` Set 实现观察者模式

### 元素层级管理

```typescript
// Z-index 管理
interface LayerManagement {
  // 元素分层
  layers: {
    background: ExcalidrawElement[];   // 背景层
    main: ExcalidrawElement[];        // 主要内容层
    foreground: ExcalidrawElement[];   // 前景层
  };

  // 按 z-index 排序元素
  sortElementsByZIndex(elements: ExcalidrawElement[]) {
    return elements.sort((a, b) => {
      // 1. 先按 versionNonce（创建时间）
      if (a.versionNonce !== b.versionNonce) {
        return a.versionNonce - b.versionNonce;
      }
      // 2. 再按 id（确保稳定排序）
      return a.id < b.id ? -1 : 1;
    });
  }

  // 移动元素到顶层
  bringToFront(element: ExcalidrawElement, elements: ExcalidrawElement[]) {
    const maxVersionNonce = Math.max(...elements.map(el => el.versionNonce));
    return {
      ...element,
      versionNonce: maxVersionNonce + 1,
    };
  }
}
```

## 渲染管线实现

### 静态画布渲染流程

```typescript
// packages/excalidraw/renderer/staticScene.ts
export const renderStaticScene = ({
  canvas,
  rc,
  elements,
  visibleElements,
  appState,
  renderConfig,
}: StaticSceneRenderConfig) => {
  const context = canvas.getContext("2d")!;

  // 1. 清空画布
  context.clearRect(0, 0, canvas.width, canvas.height);

  // 2. 设置全局渲染参数
  context.save();
  context.scale(appState.zoom.value, appState.zoom.value);
  context.translate(appState.scrollX, appState.scrollY);

  // 3. 渲染背景（网格、颜色等）
  renderBackground(context, appState);

  // 4. 按顺序渲染元素
  visibleElements.forEach(element => {
    context.save();

    // 应用元素变换
    if (element.angle) {
      const [cx, cy] = getElementCenter(element);
      context.translate(cx, cy);
      context.rotate(element.angle);
      context.translate(-cx, -cy);
    }

    // 渲染元素
    renderElement(element, rc, context, renderConfig, appState);

    context.restore();
  });

  // 5. 恢复上下文状态
  context.restore();
};
```

### 交互画布渲染流程

```typescript
// packages/excalidraw/renderer/interactiveScene.ts
export const renderInteractiveScene = throttleRAF((
  config: InteractiveSceneRenderConfig,
) => {
  const { canvas, appState, elements } = config;
  const context = canvas.getContext("2d")!;

  // 1. 清空交互层
  context.clearRect(0, 0, canvas.width, canvas.height);

  // 2. 设置变换
  context.save();
  context.scale(appState.zoom.value, appState.zoom.value);
  context.translate(appState.scrollX, appState.scrollY);

  // 3. 渲染选择相关UI
  if (appState.selectedElementIds.size > 0) {
    // 渲染选择框
    renderSelectionBorder(context, getSelectedElements(elements, appState));

    // 渲染变换手柄
    if (appState.selectedElementIds.size === 1) {
      renderTransformHandles(context, appState, elements);
    }
  }

  // 4. 渲染正在绘制的元素
  if (appState.draggingElement) {
    renderDraggingElement(context, appState.draggingElement);
  }

  // 5. 渲染其他UI元素（光标、提示等）
  renderCursor(context, appState);

  context.restore();

  // 触发渲染完成回调
  config.callback?.();
}, { trailing: true });
```

## 实战示例：实现简化版渲染引擎

### 最小化渲染引擎

```javascript
class MinimalRenderEngine {
  constructor(canvas) {
    this.canvas = canvas;
    this.context = canvas.getContext('2d');
    this.elements = [];
    this.viewport = {
      zoom: 1,
      scrollX: 0,
      scrollY: 0,
    };
    this.cache = new Map();
  }

  // 添加元素
  addElement(element) {
    this.elements.push(element);
    this.invalidateCache();
  }

  // 视口变换
  setViewport(zoom, scrollX, scrollY) {
    this.viewport = { zoom, scrollX, scrollY };
    this.render();
  }

  // 视口裁剪
  getVisibleElements() {
    const { width, height } = this.canvas;
    const { zoom, scrollX, scrollY } = this.viewport;

    // 计算视口边界（场景坐标）
    const left = -scrollX / zoom;
    const top = -scrollY / zoom;
    const right = (width - scrollX) / zoom;
    const bottom = (height - scrollY) / zoom;

    // 过滤可见元素
    return this.elements.filter(element => {
      const { x, y, width: w, height: h } = element;
      return x < right && x + w > left && y < bottom && y + h > top;
    });
  }

  // 渲染元素
  renderElement(element) {
    const ctx = this.context;
    const { type, x, y, width, height, color } = element;

    ctx.fillStyle = color;
    ctx.strokeStyle = color;

    switch (type) {
      case 'rectangle':
        ctx.strokeRect(x, y, width, height);
        break;
      case 'ellipse':
        ctx.beginPath();
        ctx.ellipse(x + width/2, y + height/2, width/2, height/2, 0, 0, 2 * Math.PI);
        ctx.stroke();
        break;
    }
  }

  // 主渲染函数
  render() {
    const ctx = this.context;
    const { zoom, scrollX, scrollY } = this.viewport;

    // 清空画布
    ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // 应用视口变换
    ctx.save();
    ctx.scale(zoom, zoom);
    ctx.translate(scrollX / zoom, scrollY / zoom);

    // 获取并渲染可见元素
    const visibleElements = this.getVisibleElements();
    visibleElements.forEach(element => {
      this.renderElement(element);
    });

    ctx.restore();

    // 显示统计信息
    this.renderStats(visibleElements.length);
  }

  // 渲染统计
  renderStats(visibleCount) {
    const ctx = this.context;
    ctx.save();
    ctx.fillStyle = '#666';
    ctx.font = '12px monospace';
    ctx.fillText(`Visible: ${visibleCount}/${this.elements.length} | Zoom: ${this.viewport.zoom.toFixed(2)}`, 10, 20);
    ctx.restore();
  }

  // 缓存管理
  invalidateCache() {
    this.cache.clear();
  }
}

// 使用示例
const canvas = document.getElementById('canvas');
const engine = new MinimalRenderEngine(canvas);

// 添加测试元素
for (let i = 0; i < 100; i++) {
  engine.addElement({
    type: Math.random() > 0.5 ? 'rectangle' : 'ellipse',
    x: Math.random() * 2000,
    y: Math.random() * 2000,
    width: 50 + Math.random() * 100,
    height: 50 + Math.random() * 100,
    color: `hsl(${Math.random() * 360}, 70%, 50%)`,
  });
}

// 渲染
engine.render();

// 添加交互
let isDragging = false;
let lastX, lastY;

canvas.addEventListener('wheel', (e) => {
  e.preventDefault();
  const delta = e.deltaY > 0 ? 0.9 : 1.1;
  engine.setViewport(
    engine.viewport.zoom * delta,
    engine.viewport.scrollX,
    engine.viewport.scrollY
  );
});

canvas.addEventListener('mousedown', (e) => {
  isDragging = true;
  lastX = e.clientX;
  lastY = e.clientY;
});

canvas.addEventListener('mousemove', (e) => {
  if (isDragging) {
    const dx = e.clientX - lastX;
    const dy = e.clientY - lastY;
    engine.setViewport(
      engine.viewport.zoom,
      engine.viewport.scrollX + dx,
      engine.viewport.scrollY + dy
    );
    lastX = e.clientX;
    lastY = e.clientY;
  }
});

canvas.addEventListener('mouseup', () => {
  isDragging = false;
});
```

## 性能优化技巧

### 1. 渲染批处理

```javascript
class BatchRenderer {
  constructor() {
    this.renderQueue = [];
    this.isScheduled = false;
  }

  // 添加渲染任务
  scheduleRender(task) {
    this.renderQueue.push(task);

    if (!this.isScheduled) {
      this.isScheduled = true;
      requestAnimationFrame(() => this.flush());
    }
  }

  // 批量执行
  flush() {
    const tasks = this.renderQueue.splice(0);

    // 合并相同类型的任务
    const mergedTasks = this.mergeTasks(tasks);

    // 执行渲染
    mergedTasks.forEach(task => task());

    this.isScheduled = false;
  }

  // 任务合并
  mergeTasks(tasks) {
    const taskMap = new Map();

    tasks.forEach(task => {
      const key = task.type;
      if (!taskMap.has(key)) {
        taskMap.set(key, []);
      }
      taskMap.get(key).push(task);
    });

    // 合并同类任务
    return Array.from(taskMap.values()).map(group => {
      return () => group.forEach(task => task.execute());
    });
  }
}
```

### 2. 离屏Canvas优化

```javascript
class OffscreenRenderer {
  constructor() {
    this.cache = new Map();
  }

  // 获取或创建离屏Canvas
  getOffscreenCanvas(element) {
    const key = this.getCacheKey(element);

    if (this.cache.has(key)) {
      return this.cache.get(key);
    }

    // 创建离屏Canvas
    const offscreen = document.createElement('canvas');
    const { width, height } = element;

    // 设置合适的分辨率
    const scale = window.devicePixelRatio || 1;
    offscreen.width = width * scale;
    offscreen.height = height * scale;

    // 渲染到离屏Canvas
    const ctx = offscreen.getContext('2d');
    ctx.scale(scale, scale);
    this.renderToOffscreen(ctx, element);

    // 缓存
    this.cache.set(key, offscreen);

    return offscreen;
  }

  // 生成缓存键
  getCacheKey(element) {
    return JSON.stringify({
      id: element.id,
      width: element.width,
      height: element.height,
      color: element.color,
      versionNonce: element.versionNonce,
    });
  }

  // 渲染到离屏Canvas
  renderToOffscreen(ctx, element) {
    // 复杂渲染逻辑
    // ...
  }

  // 使用离屏Canvas
  drawWithCache(mainCtx, element) {
    const offscreen = this.getOffscreenCanvas(element);
    mainCtx.drawImage(offscreen, element.x, element.y);
  }
}
```

### 3. 分层渲染策略

```javascript
class LayeredRenderer {
  constructor(container) {
    this.layers = {
      background: this.createLayer('background', 0),
      static: this.createLayer('static', 1),
      dynamic: this.createLayer('dynamic', 2),
      overlay: this.createLayer('overlay', 3),
    };

    // 将所有层添加到容器
    Object.values(this.layers).forEach(canvas => {
      container.appendChild(canvas);
    });
  }

  // 创建图层
  createLayer(name, zIndex) {
    const canvas = document.createElement('canvas');
    canvas.className = `layer-${name}`;
    canvas.style.position = 'absolute';
    canvas.style.zIndex = zIndex;
    canvas.width = 800;
    canvas.height = 600;
    return canvas;
  }

  // 渲染背景层（很少更新）
  renderBackground() {
    const ctx = this.layers.background.getContext('2d');
    // 渲染网格、水印等
    this.drawGrid(ctx);
  }

  // 渲染静态层（偶尔更新）
  renderStatic(elements) {
    const ctx = this.layers.static.getContext('2d');
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

    elements.filter(el => !el.selected).forEach(element => {
      this.renderElement(ctx, element);
    });
  }

  // 渲染动态层（频繁更新）
  renderDynamic(elements) {
    const ctx = this.layers.dynamic.getContext('2d');
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

    elements.filter(el => el.selected).forEach(element => {
      this.renderElement(ctx, element);
    });
  }

  // 渲染覆盖层（UI元素）
  renderOverlay(ui) {
    const ctx = this.layers.overlay.getContext('2d');
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);

    // 渲染选择框、手柄等
    this.renderSelectionBox(ctx, ui.selection);
    this.renderHandles(ctx, ui.handles);
  }
}
```

## 思考题

1. **双 Canvas 架构的优缺点是什么？** 在什么场景下单 Canvas 可能更合适？

2. **视口裁剪的边界情况如何处理？** 例如部分可见的大型元素，旋转后的元素边界计算。

3. **如何实现更智能的缓存策略？** 考虑内存限制、缓存命中率、失效策略等。

4. **渲染性能瓶颈在哪里？** 使用 Performance API 分析 Excalidraw 的实际渲染性能。

5. **如何支持 10000+ 元素的场景？** 需要哪些额外的优化策略？

## 实践练习

### 练习 1：实现视口裁剪算法
实现一个完整的视口裁剪系统，支持：
- 矩形元素的快速判断
- 旋转元素的精确判断
- 部分可见元素的处理

### 练习 2：构建缓存管理器
设计并实现一个 LRU 缓存管理器：
- 自动内存限制
- 智能失效策略
- 性能统计

### 练习 3：优化渲染管线
优化渲染流程，实现：
- 脏矩形算法
- 增量渲染
- 渲染优先级队列

## 总结

通过深入分析 Excalidraw 的实际源码，我们发现其渲染引擎展示了现代 Web 图形应用的最佳实践：

### 核心架构（已验证 ✓）

1. **双 Canvas 架构**
   - `staticScene` (packages/excalidraw/renderer/staticScene.ts) - 渲染元素本体
   - `interactiveScene` (packages/excalidraw/renderer/interactiveScene.ts) - 渲染 UI 元素

2. **Renderer 类**（packages/excalidraw/scene/Renderer.ts）
   - 使用 `memoize` 进行智能缓存
   - `getRenderableElements()` 返回 { elementsMap, visibleElements }
   - 视口裁剪通过 `isElementInViewport()` 实现

3. **Scene 类**（packages/element/src/Scene.ts）
   - 使用 `randomInteger()` 生成 sceneNonce（而非简单递增）
   - 通过观察者模式（callbacks Set）通知更新
   - 支持 fractional indices 进行元素排序

### 关键性能优化策略

1. **缓存失效机制**：基于 sceneNonce 的随机整数，每次场景更新时重新生成
2. **视口裁剪**：只渲染 `isElementInViewport()` 返回 true 的元素
3. **memoize 缓存**：getRenderableElements 的结果会被缓存，直到 sceneNonce 变化
4. **throttleRAF 节流**：所有渲染函数都通过 `throttleRAF` 节流到 60fps

### 源码验证要点

- ✓ Renderer.ts 实际位置：`packages/excalidraw/scene/Renderer.ts`
- ✓ Scene.ts 实际位置：`packages/element/src/Scene.ts`（不在 excalidraw/scene 下）
- ✓ sceneNonce 使用 randomInteger() 而非递增
- ✓ destroy() 方法有明确注释说明为何不销毁所有内容
- ✓ 编辑中的文本元素会被跳过渲染（仅在远程协作时渲染）

这些设计理念和实现技术不仅适用于画板应用，也可以应用到其他需要高性能图形渲染的 Web 应用中。

## 下一步

在下一章中，我们将深入探讨 Excalidraw 如何使用 RoughJS 实现独特的手绘风格，以及各种图形元素的具体渲染实现。