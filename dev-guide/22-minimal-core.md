# 7.1 最小核心拆解：理解 Excalidraw 的本质

本章节将带你深入理解 Excalidraw 的最小核心概念，从最基础的功能开始，逐步构建一个完整的绘图应用。

## 核心概念

### 1. 最小可行的 Excalidraw

一个最小的 Excalidraw 实现需要以下核心组件：

```typescript
// 最小核心接口
interface MinimalExcalidraw {
  canvas: HTMLCanvasElement;
  elements: ExcalidrawElement[];
  appState: AppState;
  render(): void;
  handleEvent(event: Event): void;
}

// 最小元素定义
interface MinimalElement {
  id: string;
  type: 'rectangle' | 'line' | 'text';
  x: number;
  y: number;
  width: number;
  height: number;
  strokeColor: string;
  backgroundColor: string;
}
```

### 2. 核心数据流

```typescript
// 单向数据流
class MinimalExcalidrawCore {
  private elements: ExcalidrawElement[] = [];
  private appState: AppState = getDefaultAppState();

  // 状态更新的唯一入口
  public updateState(
    elements?: ExcalidrawElement[],
    appState?: Partial<AppState>
  ): void {
    if (elements !== undefined) {
      this.elements = elements;
    }

    if (appState !== undefined) {
      this.appState = { ...this.appState, ...appState };
    }

    // 重新渲染
    this.render();
  }

  private render(): void {
    // 清空画布
    this.clearCanvas();

    // 渲染所有元素
    this.elements.forEach(element => {
      this.renderElement(element);
    });
  }
}
```

## 最小实现

### 1. 基础画布设置

```typescript
class MinimalCanvas {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;

  constructor(container: HTMLElement) {
    this.canvas = document.createElement('canvas');
    this.ctx = this.canvas.getContext('2d')!;

    this.setupCanvas(container);
    this.bindEvents();
  }

  private setupCanvas(container: HTMLElement): void {
    // 设置画布尺寸
    this.canvas.width = container.clientWidth;
    this.canvas.height = container.clientHeight;

    // 高DPI支持
    const dpr = window.devicePixelRatio || 1;
    this.canvas.width *= dpr;
    this.canvas.height *= dpr;
    this.ctx.scale(dpr, dpr);

    // 样式设置
    this.canvas.style.width = container.clientWidth + 'px';
    this.canvas.style.height = container.clientHeight + 'px';

    container.appendChild(this.canvas);
  }
}
```

### 2. 基础绘图工具

```typescript
class MinimalDrawingTool {
  private isDrawing = false;
  private startPoint: { x: number; y: number } | null = null;
  private currentElement: ExcalidrawElement | null = null;

  onPointerDown(event: PointerEvent, appState: AppState): void {
    this.isDrawing = true;
    this.startPoint = { x: event.clientX, y: event.clientY };

    // 创建新元素
    this.currentElement = {
      id: generateId(),
      type: appState.activeTool.type,
      x: this.startPoint.x,
      y: this.startPoint.y,
      width: 0,
      height: 0,
      strokeColor: appState.currentItemStrokeColor,
      backgroundColor: appState.currentItemBackgroundColor,
    };
  }

  onPointerMove(event: PointerEvent): ExcalidrawElement | null {
    if (!this.isDrawing || !this.startPoint || !this.currentElement) {
      return null;
    }

    // 更新元素尺寸
    this.currentElement.width = event.clientX - this.startPoint.x;
    this.currentElement.height = event.clientY - this.startPoint.y;

    return this.currentElement;
  }

  onPointerUp(): ExcalidrawElement | null {
    this.isDrawing = false;
    const element = this.currentElement;

    // 重置状态
    this.startPoint = null;
    this.currentElement = null;

    return element;
  }
}
```

### 3. 基础渲染器

```typescript
class MinimalRenderer {
  private ctx: CanvasRenderingContext2D;

  constructor(ctx: CanvasRenderingContext2D) {
    this.ctx = ctx;
  }

  renderElement(element: ExcalidrawElement): void {
    this.ctx.save();

    // 设置样式
    this.ctx.strokeStyle = element.strokeColor;
    this.ctx.fillStyle = element.backgroundColor;
    this.ctx.lineWidth = element.strokeWidth || 1;

    switch (element.type) {
      case 'rectangle':
        this.renderRectangle(element);
        break;
      case 'line':
        this.renderLine(element);
        break;
      case 'text':
        this.renderText(element);
        break;
    }

    this.ctx.restore();
  }

  private renderRectangle(element: ExcalidrawElement): void {
    const { x, y, width, height } = element;

    // 填充
    if (element.backgroundColor && element.backgroundColor !== 'transparent') {
      this.ctx.fillRect(x, y, width, height);
    }

    // 描边
    if (element.strokeColor && element.strokeColor !== 'transparent') {
      this.ctx.strokeRect(x, y, width, height);
    }
  }
}
```

## 状态管理核心

### 1. 不可变状态更新

```typescript
class StateManager {
  private state: AppState;
  private elements: ExcalidrawElement[];
  private subscribers: Array<(state: AppState, elements: ExcalidrawElement[]) => void> = [];

  constructor(initialState: AppState, initialElements: ExcalidrawElement[] = []) {
    this.state = initialState;
    this.elements = initialElements;
  }

  // 更新状态的唯一方法
  updateState(
    stateUpdater?: Partial<AppState> | ((state: AppState) => Partial<AppState>),
    elementsUpdater?: ExcalidrawElement[] | ((elements: ExcalidrawElement[]) => ExcalidrawElement[])
  ): void {
    let newState = this.state;
    let newElements = this.elements;

    // 更新状态
    if (stateUpdater) {
      const stateUpdate = typeof stateUpdater === 'function'
        ? stateUpdater(this.state)
        : stateUpdater;
      newState = { ...this.state, ...stateUpdate };
    }

    // 更新元素
    if (elementsUpdater) {
      newElements = typeof elementsUpdater === 'function'
        ? elementsUpdater(this.elements)
        : elementsUpdater;
    }

    this.state = newState;
    this.elements = newElements;

    // 通知订阅者
    this.notifySubscribers();
  }

  private notifySubscribers(): void {
    this.subscribers.forEach(callback => {
      callback(this.state, this.elements);
    });
  }

  subscribe(callback: (state: AppState, elements: ExcalidrawElement[]) => void): () => void {
    this.subscribers.push(callback);

    // 返回取消订阅函数
    return () => {
      const index = this.subscribers.indexOf(callback);
      if (index > -1) {
        this.subscribers.splice(index, 1);
      }
    };
  }
}
```

### 2. 事件处理核心

```typescript
class EventHandler {
  private stateManager: StateManager;
  private currentTool: MinimalDrawingTool;

  constructor(stateManager: StateManager) {
    this.stateManager = stateManager;
    this.currentTool = new MinimalDrawingTool();
  }

  handlePointerEvent(event: PointerEvent): void {
    const { state, elements } = this.stateManager.getState();

    switch (event.type) {
      case 'pointerdown':
        this.currentTool.onPointerDown(event, state);
        break;

      case 'pointermove':
        const updatedElement = this.currentTool.onPointerMove(event);
        if (updatedElement) {
          // 更新临时元素用于预览
          this.stateManager.updateState(
            { draggingElement: updatedElement }
          );
        }
        break;

      case 'pointerup':
        const newElement = this.currentTool.onPointerUp();
        if (newElement && this.isValidElement(newElement)) {
          // 添加新元素到画布
          this.stateManager.updateState(
            { draggingElement: null },
            [...elements, newElement]
          );
        }
        break;
    }
  }

  private isValidElement(element: ExcalidrawElement): boolean {
    // 基础验证：确保元素有有效的尺寸
    return Math.abs(element.width) > 1 || Math.abs(element.height) > 1;
  }
}
```

## 完整最小实现

```typescript
class MinimalExcalidraw {
  private canvas: HTMLCanvasElement;
  private ctx: CanvasRenderingContext2D;
  private stateManager: StateManager;
  private eventHandler: EventHandler;
  private renderer: MinimalRenderer;

  constructor(container: HTMLElement) {
    // 初始化画布
    this.setupCanvas(container);

    // 初始化核心组件
    this.stateManager = new StateManager(getDefaultAppState());
    this.eventHandler = new EventHandler(this.stateManager);
    this.renderer = new MinimalRenderer(this.ctx);

    // 绑定事件
    this.bindEvents();

    // 订阅状态变化
    this.stateManager.subscribe((state, elements) => {
      this.render(state, elements);
    });

    // 初始渲染
    this.render(this.stateManager.getState().state, this.stateManager.getState().elements);
  }

  private render(state: AppState, elements: ExcalidrawElement[]): void {
    // 清空画布
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // 渲染所有元素
    elements.forEach(element => {
      this.renderer.renderElement(element);
    });

    // 渲染拖拽中的元素
    if (state.draggingElement) {
      this.renderer.renderElement(state.draggingElement);
    }
  }

  private bindEvents(): void {
    ['pointerdown', 'pointermove', 'pointerup'].forEach(eventType => {
      this.canvas.addEventListener(eventType, (event) => {
        this.eventHandler.handlePointerEvent(event as PointerEvent);
      });
    });
  }
}

// 使用示例
const container = document.getElementById('canvas-container')!;
const excalidraw = new MinimalExcalidraw(container);
```

## 核心设计原则

### 1. 单一数据源
- 所有状态都通过 StateManager 管理
- 不可变状态更新确保数据一致性
- 单向数据流简化调试

### 2. 职责分离
- Renderer 只负责渲染
- EventHandler 只负责事件处理
- StateManager 只负责状态管理
- 每个组件都有明确的单一职责

### 3. 可扩展性
- 基础接口设计支持功能扩展
- 插件式架构便于添加新工具
- 模块化设计支持按需加载

这个最小核心实现包含了 Excalidraw 的所有基本概念，为理解复杂功能奠定了基础。在后续章节中，我们将基于这个核心逐步扩展更多高级功能。