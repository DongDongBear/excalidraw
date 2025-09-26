# Chapter 4.1: 工具系统设计与实现

## 概述

Excalidraw 的工具系统是其交互体验的核心。一个良好设计的工具系统不仅要支持多样化的绘图功能，还要提供直观的操作体验和可扩展的架构。本章将深入探讨 Excalidraw 工具系统的设计哲学、实现机制以及如何构建自己的工具系统。

## 工具系统架构总览

### 工具分类体系

```
Excalidraw 工具分类
├── 核心交互工具
│   ├── selection    # 选择工具
│   ├── hand         # 平移工具
│   └── laser        # 激光笔工具
├── 绘图工具
│   ├── rectangle    # 矩形
│   ├── diamond      # 菱形
│   ├── ellipse      # 椭圆
│   ├── arrow        # 箭头
│   ├── line         # 线条
│   ├── freedraw     # 自由绘制
│   └── text         # 文本
├── 高级工具
│   ├── eraser       # 橡皮擦
│   ├── frame        # 框架
│   └── image        # 图片
└── 扩展工具
    └── custom       # 自定义工具
```

### 工具状态管理

```typescript
// packages/excalidraw/types.ts
export type ToolType =
  | "selection"
  | "hand"
  | "rectangle"
  | "diamond"
  | "ellipse"
  | "arrow"
  | "line"
  | "freedraw"
  | "text"
  | "image"
  | "eraser"
  | "laser"
  | "frame";

export type ActiveTool =
  | {
      type: ToolType;
      customType: null;
    }
  | {
      type: "custom";
      customType: string;
    };

// 工具状态接口
interface ToolState {
  // 当前激活的工具
  activeTool: ActiveTool;

  // 工具锁定状态（绘制后是否保持当前工具）
  locked: boolean;

  // 工具相关的临时数据
  dragging: ExcalidrawElement | null;
  editing: ExcalidrawTextElement | null;
  penMode: boolean;

  // 工具配置选项
  strokeColor: string;
  backgroundColor: string;
  fillStyle: string;
  strokeWidth: number;
  roughness: number;
}
```

## 工具抽象与接口设计

### 工具接口定义

```typescript
// 工具基础接口
interface Tool {
  readonly type: ToolType;
  readonly icon: string;
  readonly cursor: string;
  readonly shortcut?: string;

  // 生命周期方法
  onActivate?(appState: AppState): Partial<AppState>;
  onDeactivate?(appState: AppState): Partial<AppState>;

  // 事件处理方法
  onPointerDown?(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ): {
    elements: ExcalidrawElement[];
    appState: Partial<AppState>;
  } | null;

  onPointerMove?(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ): {
    elements: ExcalidrawElement[];
    appState: Partial<AppState>;
  } | null;

  onPointerUp?(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ): {
    elements: ExcalidrawElement[];
    appState: Partial<AppState>;
  } | null;

  onKeyDown?(
    event: KeyboardEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ): {
    elements: ExcalidrawElement[];
    appState: Partial<AppState>;
  } | null;
}

// 绘图工具特定接口
interface DrawingTool extends Tool {
  createElement(
    startPoint: Point,
    endPoint: Point,
    appState: AppState
  ): ExcalidrawElement;

  updateElement(
    element: ExcalidrawElement,
    startPoint: Point,
    endPoint: Point,
    appState: AppState
  ): ExcalidrawElement;

  isComplete(element: ExcalidrawElement): boolean;
}
```

### 工具管理器实现

```typescript
// packages/excalidraw/toolManager.ts
export class ToolManager {
  private tools: Map<ToolType, Tool> = new Map();
  private currentTool: Tool | null = null;
  private appState: AppState;

  constructor(appState: AppState) {
    this.appState = appState;
    this.registerDefaultTools();
  }

  // 注册默认工具
  private registerDefaultTools() {
    this.register(new SelectionTool());
    this.register(new RectangleTool());
    this.register(new EllipseTool());
    this.register(new ArrowTool());
    this.register(new LineTool());
    this.register(new FreedrawTool());
    this.register(new TextTool());
    this.register(new EraserTool());
  }

  // 注册工具
  register(tool: Tool) {
    this.tools.set(tool.type, tool);
  }

  // 激活工具
  setActiveTool(
    toolType: ToolType,
    options?: { locked?: boolean }
  ): Partial<AppState> {
    const tool = this.tools.get(toolType);
    if (!tool) {
      console.warn(`Tool "${toolType}" not found`);
      return {};
    }

    // 停用当前工具
    let stateUpdates: Partial<AppState> = {};
    if (this.currentTool) {
      const deactivateUpdates = this.currentTool.onDeactivate?.(this.appState);
      if (deactivateUpdates) {
        stateUpdates = { ...stateUpdates, ...deactivateUpdates };
      }
    }

    // 激活新工具
    this.currentTool = tool;
    const activateUpdates = tool.onActivate?.(this.appState);
    if (activateUpdates) {
      stateUpdates = { ...stateUpdates, ...activateUpdates };
    }

    // 更新工具状态
    stateUpdates = {
      ...stateUpdates,
      activeTool: { type: toolType, customType: null },
      locked: options?.locked || false,
    };

    return stateUpdates;
  }

  // 事件委托
  handlePointerDown(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    if (!this.currentTool?.onPointerDown) {
      return null;
    }

    return this.currentTool.onPointerDown(event, appState, elements);
  }

  handlePointerMove(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    if (!this.currentTool?.onPointerMove) {
      return null;
    }

    return this.currentTool.onPointerMove(event, appState, elements);
  }

  handlePointerUp(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    if (!this.currentTool?.onPointerUp) {
      return null;
    }

    return this.currentTool.onPointerUp(event, appState, elements);
  }

  handleKeyDown(
    event: KeyboardEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    if (!this.currentTool?.onKeyDown) {
      return null;
    }

    return this.currentTool.onKeyDown(event, appState, elements);
  }

  // 获取当前工具
  getCurrentTool(): Tool | null {
    return this.currentTool;
  }

  // 获取所有已注册的工具
  getTools(): Tool[] {
    return Array.from(this.tools.values());
  }
}
```

## 具体工具实现

### 选择工具 (Selection Tool)

```typescript
export class SelectionTool implements Tool {
  readonly type = "selection" as const;
  readonly icon = "🖱️";
  readonly cursor = "default";
  readonly shortcut = "v";

  onActivate(appState: AppState): Partial<AppState> {
    return {
      cursor: "default",
      dragging: null,
    };
  }

  onPointerDown(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    const { clientX, clientY } = event;
    const scenePointer = viewportCoordsToSceneCoords(
      { clientX, clientY },
      appState
    );

    // 检查是否点击在元素上
    const hitElement = getElementAtPosition(
      elements,
      appState,
      scenePointer.x,
      scenePointer.y
    );

    if (hitElement) {
      // 点击在元素上 - 开始拖拽或选择
      if (!appState.selectedElementIds[hitElement.id]) {
        // 选择新元素
        return {
          elements,
          appState: {
            ...appState,
            selectedElementIds: { [hitElement.id]: true },
            dragging: null,
          },
        };
      } else {
        // 开始拖拽已选择的元素
        return {
          elements,
          appState: {
            ...appState,
            dragging: {
              ...appState.dragging,
              element: hitElement,
              offset: {
                x: scenePointer.x - hitElement.x,
                y: scenePointer.y - hitElement.y,
              },
            },
          },
        };
      }
    } else {
      // 点击在空白处 - 开始框选或清除选择
      return {
        elements,
        appState: {
          ...appState,
          selectedElementIds: {},
          dragging: {
            ...appState.dragging,
            selectionRect: {
              x: scenePointer.x,
              y: scenePointer.y,
              width: 0,
              height: 0,
            },
          },
        },
      };
    }
  }

  onPointerMove(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    const { clientX, clientY } = event;
    const scenePointer = viewportCoordsToSceneCoords(
      { clientX, clientY },
      appState
    );

    if (appState.dragging?.element) {
      // 拖拽元素
      const draggedElement = appState.dragging.element;
      const offset = appState.dragging.offset;

      const updatedElement = {
        ...draggedElement,
        x: scenePointer.x - offset.x,
        y: scenePointer.y - offset.y,
      };

      const updatedElements = elements.map(el =>
        el.id === updatedElement.id ? updatedElement : el
      );

      return {
        elements: updatedElements,
        appState,
      };
    } else if (appState.dragging?.selectionRect) {
      // 更新框选矩形
      const startX = appState.dragging.selectionRect.x;
      const startY = appState.dragging.selectionRect.y;

      const updatedSelectionRect = {
        x: Math.min(startX, scenePointer.x),
        y: Math.min(startY, scenePointer.y),
        width: Math.abs(scenePointer.x - startX),
        height: Math.abs(scenePointer.y - startY),
      };

      // 计算框选范围内的元素
      const selectedElements = getElementsInSelection(
        elements,
        updatedSelectionRect
      );

      const selectedElementIds: Record<string, true> = {};
      selectedElements.forEach(element => {
        selectedElementIds[element.id] = true;
      });

      return {
        elements,
        appState: {
          ...appState,
          selectedElementIds,
          dragging: {
            ...appState.dragging,
            selectionRect: updatedSelectionRect,
          },
        },
      };
    }

    return null;
  }

  onPointerUp(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    // 结束拖拽
    return {
      elements,
      appState: {
        ...appState,
        dragging: null,
      },
    };
  }
}
```

### 矩形工具 (Rectangle Tool)

```typescript
export class RectangleTool implements DrawingTool {
  readonly type = "rectangle" as const;
  readonly icon = "⬜";
  readonly cursor = "crosshair";
  readonly shortcut = "r";

  createElement(
    startPoint: Point,
    endPoint: Point,
    appState: AppState
  ): ExcalidrawRectangleElement {
    const width = endPoint.x - startPoint.x;
    const height = endPoint.y - startPoint.y;

    return {
      id: nanoid(),
      type: "rectangle",
      x: Math.min(startPoint.x, endPoint.x),
      y: Math.min(startPoint.y, endPoint.y),
      width: Math.abs(width),
      height: Math.abs(height),
      angle: 0,
      strokeColor: appState.strokeColor,
      backgroundColor: appState.backgroundColor,
      fillStyle: appState.fillStyle,
      strokeWidth: appState.strokeWidth,
      strokeStyle: appState.strokeStyle,
      roughness: appState.roughness,
      opacity: appState.opacity,
      strokeSharpness: appState.strokeSharpness,
      seed: Math.floor(Math.random() * 2 ** 31),
      versionNonce: getRandomInteger(),
      isDeleted: false,
      groupIds: [],
      frameId: null,
      index: generateFractionalIndex(
        null,
        getMaximumIndex(elements) || "a0"
      ),
    };
  }

  updateElement(
    element: ExcalidrawRectangleElement,
    startPoint: Point,
    endPoint: Point,
    appState: AppState
  ): ExcalidrawRectangleElement {
    const width = endPoint.x - startPoint.x;
    const height = endPoint.y - startPoint.y;

    return {
      ...element,
      x: Math.min(startPoint.x, endPoint.x),
      y: Math.min(startPoint.y, endPoint.y),
      width: Math.abs(width),
      height: Math.abs(height),
    };
  }

  isComplete(element: ExcalidrawRectangleElement): boolean {
    return element.width > 0 && element.height > 0;
  }

  onPointerDown(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    const { clientX, clientY } = event;
    const scenePointer = viewportCoordsToSceneCoords(
      { clientX, clientY },
      appState
    );

    // 创建新的矩形元素
    const newElement = this.createElement(
      scenePointer,
      scenePointer,
      appState
    );

    return {
      elements: [...elements, newElement],
      appState: {
        ...appState,
        dragging: {
          element: newElement,
          startPoint: scenePointer,
        },
      },
    };
  }

  onPointerMove(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    if (!appState.dragging?.element) {
      return null;
    }

    const { clientX, clientY } = event;
    const scenePointer = viewportCoordsToSceneCoords(
      { clientX, clientY },
      appState
    );

    const startPoint = appState.dragging.startPoint;
    const draggedElement = appState.dragging.element;

    // 更新矩形尺寸
    const updatedElement = this.updateElement(
      draggedElement,
      startPoint,
      scenePointer,
      appState
    );

    const updatedElements = elements.map(el =>
      el.id === updatedElement.id ? updatedElement : el
    );

    return {
      elements: updatedElements,
      appState,
    };
  }

  onPointerUp(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    if (!appState.dragging?.element) {
      return null;
    }

    const element = appState.dragging.element;

    // 检查元素是否有效（有尺寸）
    if (!this.isComplete(element)) {
      // 移除无效元素
      const filteredElements = elements.filter(el => el.id !== element.id);
      return {
        elements: filteredElements,
        appState: {
          ...appState,
          dragging: null,
        },
      };
    }

    // 完成绘制
    const nextAppState: Partial<AppState> = {
      dragging: null,
    };

    // 如果工具未锁定，切换回选择工具
    if (!appState.locked) {
      nextAppState.activeTool = { type: "selection", customType: null };
    }

    return {
      elements,
      appState: {
        ...appState,
        ...nextAppState,
      },
    };
  }
}
```

### 自由绘制工具 (Freedraw Tool)

```typescript
export class FreedrawTool implements Tool {
  readonly type = "freedraw" as const;
  readonly icon = "✏️";
  readonly cursor = "crosshair";
  readonly shortcut = "p";

  onActivate(appState: AppState): Partial<AppState> {
    return {
      cursor: "crosshair",
      penMode: true,
    };
  }

  onPointerDown(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    // 检查是否支持压感
    const pressure = "pressure" in event ? event.pressure : 0.5;

    const { clientX, clientY } = event;
    const scenePointer = viewportCoordsToSceneCoords(
      { clientX, clientY },
      appState
    );

    // 创建自由绘制元素
    const newElement: ExcalidrawFreeDrawElement = {
      id: nanoid(),
      type: "freedraw",
      x: scenePointer.x,
      y: scenePointer.y,
      width: 0,
      height: 0,
      angle: 0,
      strokeColor: appState.strokeColor,
      backgroundColor: "transparent",
      fillStyle: "hachure",
      strokeWidth: appState.strokeWidth,
      strokeStyle: appState.strokeStyle,
      roughness: 0,
      opacity: appState.opacity,
      points: [[0, 0]],
      pressures: [pressure],
      simulatePressure: !("pressure" in event),
      seed: Math.floor(Math.random() * 2 ** 31),
      versionNonce: getRandomInteger(),
      isDeleted: false,
      groupIds: [],
      frameId: null,
      index: generateFractionalIndex(
        null,
        getMaximumIndex(elements) || "a0"
      ),
    };

    return {
      elements: [...elements, newElement],
      appState: {
        ...appState,
        dragging: {
          element: newElement,
          lastPoint: scenePointer,
        },
      },
    };
  }

  onPointerMove(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    if (!appState.dragging?.element) {
      return null;
    }

    const { clientX, clientY } = event;
    const scenePointer = viewportCoordsToSceneCoords(
      { clientX, clientY },
      appState
    );

    const element = appState.dragging.element as ExcalidrawFreeDrawElement;
    const lastPoint = appState.dragging.lastPoint;

    // 计算距离，避免添加过于密集的点
    const distance = Math.hypot(
      scenePointer.x - lastPoint.x,
      scenePointer.y - lastPoint.y
    );

    const MIN_DISTANCE = 3;
    if (distance < MIN_DISTANCE) {
      return null;
    }

    // 添加新点
    const pressure = "pressure" in event ? event.pressure : 0.5;
    const relativeX = scenePointer.x - element.x;
    const relativeY = scenePointer.y - element.y;

    const updatedElement = {
      ...element,
      points: [...element.points, [relativeX, relativeY]],
      pressures: element.pressures ? [...element.pressures, pressure] : null,
      // 更新边界
      width: Math.max(element.width, Math.abs(relativeX)),
      height: Math.max(element.height, Math.abs(relativeY)),
    };

    // 更新最小边界
    const minX = Math.min(element.x, scenePointer.x);
    const minY = Math.min(element.y, scenePointer.y);

    if (minX !== element.x || minY !== element.y) {
      // 调整坐标原点
      updatedElement.x = minX;
      updatedElement.y = minY;
      updatedElement.points = updatedElement.points.map(([x, y]) => [
        x + (element.x - minX),
        y + (element.y - minY),
      ]);
    }

    const updatedElements = elements.map(el =>
      el.id === updatedElement.id ? updatedElement : el
    );

    return {
      elements: updatedElements,
      appState: {
        ...appState,
        dragging: {
          ...appState.dragging,
          lastPoint: scenePointer,
        },
      },
    };
  }

  onPointerUp(
    event: PointerEvent,
    appState: AppState,
    elements: ExcalidrawElement[]
  ) {
    if (!appState.dragging?.element) {
      return null;
    }

    const element = appState.dragging.element as ExcalidrawFreeDrawElement;

    // 检查是否有足够的点
    if (element.points.length < 2) {
      // 移除无效的绘制
      const filteredElements = elements.filter(el => el.id !== element.id);
      return {
        elements: filteredElements,
        appState: {
          ...appState,
          dragging: null,
        },
      };
    }

    // 平滑路径
    const smoothedElement = this.smoothPath(element);

    const updatedElements = elements.map(el =>
      el.id === element.id ? smoothedElement : el
    );

    return {
      elements: updatedElements,
      appState: {
        ...appState,
        dragging: null,
      },
    };
  }

  // 路径平滑算法
  private smoothPath(element: ExcalidrawFreeDrawElement): ExcalidrawFreeDrawElement {
    if (element.points.length < 3) {
      return element;
    }

    const smoothedPoints: [number, number][] = [element.points[0]];

    for (let i = 1; i < element.points.length - 1; i++) {
      const prev = element.points[i - 1];
      const curr = element.points[i];
      const next = element.points[i + 1];

      // 简单的平均平滑
      const smoothed: [number, number] = [
        (prev[0] + curr[0] + next[0]) / 3,
        (prev[1] + curr[1] + next[1]) / 3,
      ];

      smoothedPoints.push(smoothed);
    }

    smoothedPoints.push(element.points[element.points.length - 1]);

    return {
      ...element,
      points: smoothedPoints,
    };
  }
}
```

## 工具状态机

### 状态转换管理

```typescript
// 工具状态枚举
enum ToolState {
  IDLE = "idle",
  DRAWING = "drawing",
  DRAGGING = "dragging",
  RESIZING = "resizing",
  EDITING = "editing",
}

// 状态转换映射
const stateTransitions: Record<ToolState, Record<string, ToolState>> = {
  [ToolState.IDLE]: {
    pointerDown: ToolState.DRAWING,
    selectElement: ToolState.DRAGGING,
    doubleClick: ToolState.EDITING,
  },
  [ToolState.DRAWING]: {
    pointerUp: ToolState.IDLE,
    escape: ToolState.IDLE,
  },
  [ToolState.DRAGGING]: {
    pointerUp: ToolState.IDLE,
    escape: ToolState.IDLE,
    startResize: ToolState.RESIZING,
  },
  [ToolState.RESIZING]: {
    pointerUp: ToolState.IDLE,
    escape: ToolState.IDLE,
  },
  [ToolState.EDITING]: {
    clickOutside: ToolState.IDLE,
    enter: ToolState.IDLE,
    escape: ToolState.IDLE,
  },
};

// 工具状态机
export class ToolStateMachine {
  private currentState: ToolState = ToolState.IDLE;
  private listeners: Map<string, Set<(state: ToolState) => void>> = new Map();

  // 状态转换
  transition(event: string): boolean {
    const transitions = stateTransitions[this.currentState];
    const nextState = transitions[event];

    if (nextState) {
      const prevState = this.currentState;
      this.currentState = nextState;
      this.emit("stateChange", this.currentState, prevState);
      return true;
    }

    return false;
  }

  // 获取当前状态
  getState(): ToolState {
    return this.currentState;
  }

  // 监听状态变化
  on(event: string, listener: (state: ToolState, prevState?: ToolState) => void) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(listener);
  }

  // 触发事件
  private emit(event: string, ...args: any[]) {
    const eventListeners = this.listeners.get(event);
    if (eventListeners) {
      eventListeners.forEach(listener => listener(...args));
    }
  }

  // 检查是否可以执行操作
  canExecute(action: string): boolean {
    const transitions = stateTransitions[this.currentState];
    return action in transitions;
  }
}
```

## 快捷键系统

### 快捷键绑定

```typescript
// packages/excalidraw/shortcuts.ts
export interface ShortcutBinding {
  key: string;
  modifiers?: {
    ctrl?: boolean;
    alt?: boolean;
    shift?: boolean;
    meta?: boolean;
  };
  action: string;
  tool?: ToolType;
  preventDefault?: boolean;
}

// 默认快捷键配置
export const defaultShortcuts: ShortcutBinding[] = [
  // 工具切换
  { key: "v", action: "setTool", tool: "selection" },
  { key: "r", action: "setTool", tool: "rectangle" },
  { key: "d", action: "setTool", tool: "diamond" },
  { key: "o", action: "setTool", tool: "ellipse" },
  { key: "a", action: "setTool", tool: "arrow" },
  { key: "l", action: "setTool", tool: "line" },
  { key: "p", action: "setTool", tool: "freedraw" },
  { key: "t", action: "setTool", tool: "text" },
  { key: "h", action: "setTool", tool: "hand" },
  { key: "e", action: "setTool", tool: "eraser" },

  // 操作快捷键
  { key: "Delete", action: "deleteSelected" },
  { key: "Backspace", action: "deleteSelected" },
  {
    key: "z",
    modifiers: { ctrl: true },
    action: "undo",
    preventDefault: true
  },
  {
    key: "y",
    modifiers: { ctrl: true },
    action: "redo",
    preventDefault: true
  },
  {
    key: "z",
    modifiers: { ctrl: true, shift: true },
    action: "redo",
    preventDefault: true
  },
  {
    key: "a",
    modifiers: { ctrl: true },
    action: "selectAll",
    preventDefault: true
  },
  {
    key: "d",
    modifiers: { ctrl: true },
    action: "duplicate",
    preventDefault: true
  },

  // 视图控制
  { key: "1", action: "zoomToFit" },
  { key: "2", action: "zoomToSelection" },
  { key: "0", action: "resetZoom" },
  { key: "+", action: "zoomIn" },
  { key: "-", action: "zoomOut" },

  // 图层操作
  {
    key: "]",
    modifiers: { ctrl: true },
    action: "bringForward"
  },
  {
    key: "[",
    modifiers: { ctrl: true },
    action: "sendBackward"
  },
  {
    key: "]",
    modifiers: { ctrl: true, shift: true },
    action: "bringToFront"
  },
  {
    key: "[",
    modifiers: { ctrl: true, shift: true },
    action: "sendToBack"
  },
];

// 快捷键管理器
export class ShortcutManager {
  private bindings: Map<string, ShortcutBinding> = new Map();
  private actionHandlers: Map<string, (binding: ShortcutBinding) => void> = new Map();

  constructor() {
    this.registerDefaultShortcuts();
    this.setupEventListeners();
  }

  // 注册默认快捷键
  private registerDefaultShortcuts() {
    defaultShortcuts.forEach(binding => {
      this.register(binding);
    });
  }

  // 注册快捷键
  register(binding: ShortcutBinding) {
    const key = this.getBindingKey(binding);
    this.bindings.set(key, binding);
  }

  // 注销快捷键
  unregister(binding: ShortcutBinding) {
    const key = this.getBindingKey(binding);
    this.bindings.delete(key);
  }

  // 注册动作处理器
  registerActionHandler(action: string, handler: (binding: ShortcutBinding) => void) {
    this.actionHandlers.set(action, handler);
  }

  // 生成绑定键
  private getBindingKey(binding: ShortcutBinding): string {
    const modifiers = binding.modifiers || {};
    const parts: string[] = [];

    if (modifiers.ctrl) parts.push("ctrl");
    if (modifiers.alt) parts.push("alt");
    if (modifiers.shift) parts.push("shift");
    if (modifiers.meta) parts.push("meta");

    parts.push(binding.key.toLowerCase());

    return parts.join("+");
  }

  // 处理键盘事件
  private handleKeyDown = (event: KeyboardEvent) => {
    // 忽略在输入框中的按键
    if (this.isInputElement(event.target)) {
      return;
    }

    const key = this.getEventKey(event);
    const binding = this.bindings.get(key);

    if (binding) {
      if (binding.preventDefault) {
        event.preventDefault();
      }

      const handler = this.actionHandlers.get(binding.action);
      if (handler) {
        handler(binding);
      }
    }
  };

  // 从键盘事件生成键
  private getEventKey(event: KeyboardEvent): string {
    const parts: string[] = [];

    if (event.ctrlKey || event.metaKey) parts.push("ctrl");
    if (event.altKey) parts.push("alt");
    if (event.shiftKey) parts.push("shift");
    if (event.metaKey && !event.ctrlKey) parts.push("meta");

    parts.push(event.key.toLowerCase());

    return parts.join("+");
  }

  // 检查是否为输入元素
  private isInputElement(target: EventTarget | null): boolean {
    if (!target) return false;

    const element = target as HTMLElement;
    const tagName = element.tagName.toLowerCase();

    return (
      tagName === "input" ||
      tagName === "textarea" ||
      element.contentEditable === "true"
    );
  }

  // 设置事件监听
  private setupEventListeners() {
    document.addEventListener("keydown", this.handleKeyDown);
  }

  // 销毁
  destroy() {
    document.removeEventListener("keydown", this.handleKeyDown);
    this.bindings.clear();
    this.actionHandlers.clear();
  }

  // 获取工具的快捷键
  getToolShortcut(tool: ToolType): string | null {
    for (const [key, binding] of this.bindings) {
      if (binding.tool === tool) {
        return binding.key.toUpperCase();
      }
    }
    return null;
  }
}
```

## 实战示例：构建自定义工具

### 星形工具实现

```javascript
// 自定义星形工具
class StarTool {
  constructor() {
    this.type = "star";
    this.icon = "⭐";
    this.cursor = "crosshair";
    this.shortcut = "s";
  }

  // 创建星形元素
  createElement(startPoint, endPoint, appState) {
    const width = endPoint.x - startPoint.x;
    const height = endPoint.y - startPoint.y;
    const size = Math.max(Math.abs(width), Math.abs(height));

    return {
      id: this.generateId(),
      type: "star",
      x: startPoint.x,
      y: startPoint.y,
      width: size,
      height: size,
      angle: 0,
      strokeColor: appState.strokeColor,
      backgroundColor: appState.backgroundColor,
      fillStyle: appState.fillStyle,
      strokeWidth: appState.strokeWidth,
      roughness: appState.roughness,
      opacity: appState.opacity,
      // 星形特有属性
      points: 5,          // 星形角数
      innerRadius: 0.4,   // 内半径比例
      seed: Math.floor(Math.random() * 2 ** 31),
    };
  }

  // 更新星形尺寸
  updateElement(element, startPoint, endPoint) {
    const width = endPoint.x - startPoint.x;
    const height = endPoint.y - startPoint.y;
    const size = Math.max(Math.abs(width), Math.abs(height));

    return {
      ...element,
      width: size,
      height: size,
    };
  }

  // 生成星形路径
  generatePath(element) {
    const { x, y, width, height, points, innerRadius } = element;
    const cx = x + width / 2;
    const cy = y + height / 2;
    const outerRadius = Math.min(width, height) / 2;
    const innerR = outerRadius * innerRadius;

    const path = [];
    const angleStep = (Math.PI * 2) / (points * 2);

    for (let i = 0; i < points * 2; i++) {
      const angle = i * angleStep - Math.PI / 2;
      const radius = i % 2 === 0 ? outerRadius : innerR;

      const px = cx + Math.cos(angle) * radius;
      const py = cy + Math.sin(angle) * radius;

      if (i === 0) {
        path.push(`M ${px} ${py}`);
      } else {
        path.push(`L ${px} ${py}`);
      }
    }

    path.push('Z');
    return path.join(' ');
  }

  // 渲染星形
  render(element, ctx, renderConfig) {
    const path = new Path2D(this.generatePath(element));

    // 设置样式
    ctx.globalAlpha = element.opacity;
    ctx.strokeStyle = element.strokeColor;
    ctx.lineWidth = element.strokeWidth;

    if (element.backgroundColor !== "transparent") {
      ctx.fillStyle = element.backgroundColor;
      ctx.fill(path);
    }

    ctx.stroke(path);
  }

  // 事件处理
  onPointerDown(event, appState, elements) {
    const scenePointer = this.viewportToScene(event, appState);

    const newElement = this.createElement(
      scenePointer,
      scenePointer,
      appState
    );

    return {
      elements: [...elements, newElement],
      appState: {
        ...appState,
        dragging: {
          element: newElement,
          startPoint: scenePointer,
        },
      },
    };
  }

  onPointerMove(event, appState, elements) {
    if (!appState.dragging?.element) {
      return null;
    }

    const scenePointer = this.viewportToScene(event, appState);
    const startPoint = appState.dragging.startPoint;
    const element = appState.dragging.element;

    const updatedElement = this.updateElement(
      element,
      startPoint,
      scenePointer
    );

    const updatedElements = elements.map(el =>
      el.id === updatedElement.id ? updatedElement : el
    );

    return {
      elements: updatedElements,
      appState,
    };
  }

  onPointerUp(event, appState, elements) {
    if (!appState.dragging?.element) {
      return null;
    }

    const element = appState.dragging.element;

    // 检查尺寸有效性
    if (element.width < 10 || element.height < 10) {
      const filteredElements = elements.filter(el => el.id !== element.id);
      return {
        elements: filteredElements,
        appState: {
          ...appState,
          dragging: null,
        },
      };
    }

    return {
      elements,
      appState: {
        ...appState,
        dragging: null,
        activeTool: appState.locked
          ? appState.activeTool
          : { type: "selection", customType: null },
      },
    };
  }

  // 工具属性面板
  renderToolOptions(element, onUpdate) {
    return `
      <div class="tool-options">
        <label>
          角数:
          <input
            type="range"
            min="3"
            max="12"
            value="${element.points}"
            oninput="onUpdate({ points: parseInt(this.value) })"
          />
          <span>${element.points}</span>
        </label>

        <label>
          内半径:
          <input
            type="range"
            min="0.2"
            max="0.8"
            step="0.1"
            value="${element.innerRadius}"
            oninput="onUpdate({ innerRadius: parseFloat(this.value) })"
          />
          <span>${(element.innerRadius * 100).toFixed(0)}%</span>
        </label>
      </div>
    `;
  }

  // 辅助方法
  generateId() {
    return Math.random().toString(36).substr(2, 9);
  }

  viewportToScene(event, appState) {
    const canvas = event.target;
    const rect = canvas.getBoundingClientRect();
    const viewportX = event.clientX - rect.left;
    const viewportY = event.clientY - rect.top;

    return {
      x: (viewportX - appState.offsetLeft) / appState.zoom.value - appState.scrollX,
      y: (viewportY - appState.offsetTop) / appState.zoom.value - appState.scrollY,
    };
  }
}

// 注册自定义工具
const toolManager = new ToolManager(appState);
toolManager.register(new StarTool());

// 使用示例
const starTool = new StarTool();

// 模拟绘制星形
let appState = {
  strokeColor: "#000000",
  backgroundColor: "#ffff00",
  fillStyle: "solid",
  strokeWidth: 2,
  roughness: 1,
  opacity: 1,
  dragging: null,
  locked: false,
  activeTool: { type: "star", customType: null },
};

let elements = [];

// 开始绘制
const startResult = starTool.onPointerDown(
  { clientX: 100, clientY: 100, target: canvas },
  appState,
  elements
);

elements = startResult.elements;
appState = { ...appState, ...startResult.appState };

// 拖拽绘制
const moveResult = starTool.onPointerMove(
  { clientX: 200, clientY: 200, target: canvas },
  appState,
  elements
);

elements = moveResult.elements;
appState = { ...appState, ...moveResult.appState };

// 结束绘制
const endResult = starTool.onPointerUp(
  { clientX: 200, clientY: 200, target: canvas },
  appState,
  elements
);

elements = endResult.elements;
appState = { ...appState, ...endResult.appState };

console.log("绘制完成，创建了星形:", elements[0]);
```

### 多点绘制工具

```javascript
// 多点绘制工具（如多边形）
class PolygonTool {
  constructor() {
    this.type = "polygon";
    this.icon = "⬡";
    this.cursor = "crosshair";
    this.shortcut = "g";
  }

  onPointerDown(event, appState, elements) {
    const scenePointer = this.viewportToScene(event, appState);

    if (!appState.dragging?.element) {
      // 开始新的多边形
      const newElement = {
        id: this.generateId(),
        type: "polygon",
        x: scenePointer.x,
        y: scenePointer.y,
        width: 0,
        height: 0,
        angle: 0,
        points: [[0, 0]],  // 相对坐标点数组
        strokeColor: appState.strokeColor,
        backgroundColor: appState.backgroundColor,
        fillStyle: appState.fillStyle,
        strokeWidth: appState.strokeWidth,
        roughness: appState.roughness,
        opacity: appState.opacity,
        isCompleted: false,
      };

      return {
        elements: [...elements, newElement],
        appState: {
          ...appState,
          dragging: {
            element: newElement,
            isMultiPoint: true,
          },
        },
      };
    } else {
      // 添加新点到现有多边形
      const element = appState.dragging.element;
      const relativeX = scenePointer.x - element.x;
      const relativeY = scenePointer.y - element.y;

      // 检查是否点击在起始点附近（完成多边形）
      const firstPoint = element.points[0];
      const firstPointAbsolute = {
        x: element.x + firstPoint[0],
        y: element.y + firstPoint[1],
      };

      const distanceToFirst = Math.hypot(
        scenePointer.x - firstPointAbsolute.x,
        scenePointer.y - firstPointAbsolute.y
      );

      if (distanceToFirst < 15 && element.points.length >= 3) {
        // 完成多边形
        const completedElement = {
          ...element,
          isCompleted: true,
        };

        const updatedElements = elements.map(el =>
          el.id === element.id ? completedElement : el
        );

        return {
          elements: updatedElements,
          appState: {
            ...appState,
            dragging: null,
            activeTool: { type: "selection", customType: null },
          },
        };
      } else {
        // 添加新点
        const updatedElement = {
          ...element,
          points: [...element.points, [relativeX, relativeY]],
          width: Math.max(element.width, Math.abs(relativeX)),
          height: Math.max(element.height, Math.abs(relativeY)),
        };

        const updatedElements = elements.map(el =>
          el.id === element.id ? updatedElement : el
        );

        return {
          elements: updatedElements,
          appState,
        };
      }
    }
  }

  onPointerMove(event, appState, elements) {
    if (!appState.dragging?.element || !appState.dragging.isMultiPoint) {
      return null;
    }

    const scenePointer = this.viewportToScene(event, appState);
    const element = appState.dragging.element;

    // 更新预览点
    const relativeX = scenePointer.x - element.x;
    const relativeY = scenePointer.y - element.y;

    const updatedElement = {
      ...element,
      previewPoint: [relativeX, relativeY],
    };

    const updatedElements = elements.map(el =>
      el.id === element.id ? updatedElement : el
    );

    return {
      elements: updatedElements,
      appState,
    };
  }

  onKeyDown(event, appState, elements) {
    if (event.key === "Escape" && appState.dragging?.isMultiPoint) {
      // 取消多边形绘制
      const element = appState.dragging.element;
      const filteredElements = elements.filter(el => el.id !== element.id);

      return {
        elements: filteredElements,
        appState: {
          ...appState,
          dragging: null,
        },
      };
    }

    if (event.key === "Enter" && appState.dragging?.isMultiPoint) {
      // 完成多边形
      const element = appState.dragging.element;

      if (element.points.length >= 3) {
        const completedElement = {
          ...element,
          isCompleted: true,
        };

        const updatedElements = elements.map(el =>
          el.id === element.id ? completedElement : el
        );

        return {
          elements: updatedElements,
          appState: {
            ...appState,
            dragging: null,
          },
        };
      }
    }

    return null;
  }

  // 渲染多边形
  render(element, ctx, renderConfig) {
    if (element.points.length < 2) return;

    ctx.save();
    ctx.globalAlpha = element.opacity;
    ctx.strokeStyle = element.strokeColor;
    ctx.lineWidth = element.strokeWidth;

    // 绘制多边形路径
    ctx.beginPath();

    element.points.forEach((point, index) => {
      const x = element.x + point[0];
      const y = element.y + point[1];

      if (index === 0) {
        ctx.moveTo(x, y);
      } else {
        ctx.lineTo(x, y);
      }
    });

    // 绘制预览线
    if (element.previewPoint && !element.isCompleted) {
      const previewX = element.x + element.previewPoint[0];
      const previewY = element.y + element.previewPoint[1];

      ctx.lineTo(previewX, previewY);

      // 绘制到起始点的预览线
      if (element.points.length >= 3) {
        const firstX = element.x + element.points[0][0];
        const firstY = element.y + element.points[0][1];
        ctx.lineTo(firstX, firstY);
      }
    }

    if (element.isCompleted) {
      ctx.closePath();
    }

    // 填充
    if (element.backgroundColor !== "transparent") {
      ctx.fillStyle = element.backgroundColor;
      ctx.fill();
    }

    // 描边
    ctx.stroke();

    // 绘制控制点
    if (!element.isCompleted && renderConfig.showControls) {
      ctx.fillStyle = "#4285f4";
      element.points.forEach(point => {
        const x = element.x + point[0];
        const y = element.y + point[1];
        ctx.beginPath();
        ctx.arc(x, y, 4, 0, 2 * Math.PI);
        ctx.fill();
      });

      // 起始点特殊标记
      if (element.points.length >= 3) {
        const firstPoint = element.points[0];
        const firstX = element.x + firstPoint[0];
        const firstY = element.y + firstPoint[1];

        ctx.strokeStyle = "#4285f4";
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.arc(firstX, firstY, 8, 0, 2 * Math.PI);
        ctx.stroke();
      }
    }

    ctx.restore();
  }
}
```

## 思考题

1. **工具扩展性如何设计？** 如何支持插件式工具开发？

2. **工具间的数据共享如何处理？** 不同工具如何共享状态和配置？

3. **复杂工具的状态管理？** 如何处理多步骤的工具操作？

4. **工具性能优化？** 如何避免工具切换时的性能损失？

5. **移动端适配？** 触摸操作下的工具体验如何优化？

## 实践练习

### 练习 1：实现图章工具
创建一个图章工具，支持：
- 自定义图章图案
- 图案库管理
- 大小和旋转调整

### 练习 2：实现测量工具
实现测量距离和角度的工具：
- 实时显示测量结果
- 支持不同单位
- 临时标注功能

### 练习 3：实现智能连线工具
创建智能连接工具：
- 自动吸附到元素边缘
- 支持不同连接点
- 连接关系维护

## 总结

Excalidraw 的工具系统展示了优秀交互设计的核心要素：

1. **清晰的抽象层次**：Tool 接口提供了统一的工具协议
2. **灵活的状态管理**：支持复杂的工具状态转换
3. **可扩展的架构**：便于添加新工具和自定义功能
4. **直观的操作体验**：快捷键和状态反馈提升效率
5. **完善的事件处理**：覆盖各种输入设备和交互场景

这些设计原则不仅适用于绘图工具，也可以应用到其他需要复杂交互的应用中。

## 下一步

下一章我们将深入探讨手势处理和输入系统，了解 Excalidraw 如何处理复杂的多点触控、手势识别以及各种输入设备的适配。