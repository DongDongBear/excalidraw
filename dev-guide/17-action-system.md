# Chapter 5.1: Action 与命令模式实现

## 概述

Action系统是Excalidraw架构的重要组成部分，它实现了命令模式（Command Pattern），将所有用户操作封装为可执行、可撤销的命令。这种设计不仅支持了强大的撤销/重做功能，还为宏录制、协作同步、操作回放等高级功能奠定了基础。本章将深入解析这套Action系统的设计与实现。

## Action系统架构

### 核心概念

```
Action系统层次结构
├── Action接口层
│   ├── ActionName        # 动作名称枚举
│   ├── ActionFunction    # 动作执行函数
│   └── ActionManager     # 动作管理器
├── History管理层
│   ├── HistoryStack      # 历史栈
│   ├── Snapshot         # 状态快照
│   └── HistoryEntry     # 历史条目
├── 撤销重做层
│   ├── UndoRedoManager   # 撤销重做管理
│   ├── StateComparator   # 状态比较器
│   └── DiffCalculator    # 差异计算器
└── 协作同步层
    ├── ActionBroadcast   # 动作广播
    ├── ConflictResolver  # 冲突解决
    └── SyncManager       # 同步管理
```

### Action接口定义

```typescript
// packages/excalidraw/actions/types.ts - 实际源码接口
export interface Action {
  name: ActionName;

  // 动态标签（支持函数式计算）
  label: string | ((
    elements: readonly ExcalidrawElement[],
    appState: Readonly<AppState>,
    app: AppClassProperties,
  ) => string);

  // 搜索关键词
  keywords?: string[];

  // 动态图标（支持React组件）
  icon?: React.ReactNode | ((
    appState: UIAppState,
    elements: readonly ExcalidrawElement[],
  ) => React.ReactNode);

  // UI组件（用于自定义面板）
  PanelComponent?: React.FC<PanelComponentProps>;

  // 核心执行函数
  perform: ActionFn;

  // 按键优先级（处理冲突）
  keyPriority?: number;

  // 键盘快捷键测试
  keyTest?: (
    event: React.KeyboardEvent | KeyboardEvent,
    appState: AppState,
    elements: readonly ExcalidrawElement[],
    app: AppClassProperties,
  ) => boolean;

  // 可用性条件判断
  predicate?: (
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    props: any,
    app: AppClassProperties,
  ) => boolean;
}

// Action执行函数类型
type ActionFn = (
  elements: readonly OrderedExcalidrawElement[],
  appState: Readonly<AppState>,
  formData: any,
  app: AppClassProperties,
) => ActionResult | Promise<ActionResult>;

// Action执行结果类型
export type ActionResult =
  | {
      elements?: readonly ExcalidrawElement[] | null;
      appState?: Partial<AppState> | null;
      files?: BinaryFiles | null;
      captureUpdate: CaptureUpdateActionType;  // 历史记录捕获类型
      replaceFiles?: boolean;
    }
  | false; // false表示阻止执行

// 历史记录捕获类型
export type CaptureUpdateActionType =
  | "never"          // 不记录到历史
  | "always"         // 总是记录
  | "skipIfEmpty"    // 空变更时跳过
  | "incremental";   // 增量记录

export type ActionName =
  | "copy"
  | "paste"
  | "copyAsPng"
  | "copyAsSvg"
  | "sendBackward"
  | "bringForward"
  | "sendToBack"
  | "bringToFront"
  | "copyStyles"
  | "pasteStyles"
  | "changeStrokeColor"
  | "changeBackgroundColor"
  | "changeFillStyle"
  | "changeStrokeWidth"
  | "changeStrokeStyle"
  | "changeArrowhead"
  | "changeOpacity"
  | "changeFontSize"
  | "changeFontFamily"
  | "changeTextAlign"
  | "changeVerticalAlign"
  | "toggleCanvasMenu"
  | "toggleEditMenu"
  | "undo"
  | "redo"
  | "finalize"
  | "changeProjectName"
  | "changeExportBackground"
  | "changeExportEmbedScene"
  | "changeShouldAddWatermark"
  | "saveToDisk"
  | "loadFromJSON"
  | "duplicateSelection"
  | "deleteSelectedElements"
  | "changeViewBackgroundColor"
  | "clearCanvas"
  | "zoomIn"
  | "zoomOut"
  | "resetZoom"
  | "zoomToFit"
  | "zoomToSelection"
  | "changeImageFilter"
  | "createFrameFromSelection"
  | "removeFrameFromSelection";
```

## Action管理器实现

### ActionManager核心类

```typescript
// packages/excalidraw/actions/manager.tsx
export class ActionManager {
  private actions = new Map<ActionName, ExcalidrawAction>();
  private historyManager: HistoryManager;
  private broadcastManager: BroadcastManager;

  constructor() {
    this.historyManager = new HistoryManager();
    this.broadcastManager = new BroadcastManager();
    this.registerDefaultActions();
  }

  // 注册动作
  registerAction(action: ExcalidrawAction) {
    this.actions.set(action.name, action);
  }

  // 注册默认动作
  private registerDefaultActions() {
    // 基础编辑动作
    this.registerAction(actionCopy);
    this.registerAction(actionPaste);
    this.registerAction(actionDuplicate);
    this.registerAction(actionDeleteSelected);

    // 撤销重做
    this.registerAction(actionUndo);
    this.registerAction(actionRedo);

    // 样式动作
    this.registerAction(actionChangeStrokeColor);
    this.registerAction(actionChangeBackgroundColor);
    this.registerAction(actionChangeFillStyle);

    // 层级动作
    this.registerAction(actionSendBackward);
    this.registerAction(actionBringForward);
    this.registerAction(actionSendToBack);
    this.registerAction(actionBringToFront);

    // 视图动作
    this.registerAction(actionZoomIn);
    this.registerAction(actionZoomOut);
    this.registerAction(actionResetZoom);
    this.registerAction(actionZoomToFit);
  }

  // 执行动作
  async executeAction(
    actionName: ActionName,
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    formData?: any,
    app?: AppClassProperties
  ): Promise<ActionResult | null> {
    const action = this.actions.get(actionName);

    if (!action) {
      console.warn(`Action "${actionName}" not found`);
      return null;
    }

    // 检查前置条件
    if (action.predicate && !action.predicate(elements, appState, formData)) {
      return null;
    }

    // 创建历史快照
    const shouldCommitToHistory = action.commitToHistory !== false;
    if (shouldCommitToHistory) {
      this.historyManager.createSnapshot(elements, appState);
    }

    try {
      // 执行动作
      const result = await action.perform(elements, appState, formData, app!);

      // 处理结果
      const finalResult = this.processActionResult(result, elements, appState);

      // 提交到历史
      if (shouldCommitToHistory && finalResult.commitToHistory !== false) {
        this.historyManager.commitSnapshot(
          finalResult.elements || elements,
          { ...appState, ...finalResult.appState }
        );
      }

      // 广播动作（协作模式）
      if (finalResult.syncableElements) {
        this.broadcastManager.broadcastAction(actionName, finalResult);
      }

      return finalResult;
    } catch (error) {
      console.error(`Action "${actionName}" execution failed:`, error);

      // 回滚历史快照
      if (shouldCommitToHistory) {
        this.historyManager.rollbackSnapshot();
      }

      return null;
    }
  }

  // 处理动作结果
  private processActionResult(
    result: ActionResult,
    originalElements: readonly ExcalidrawElement[],
    originalAppState: AppState
  ): ActionResult {
    return {
      elements: result.elements ?? originalElements,
      appState: result.appState ? { ...originalAppState, ...result.appState } : originalAppState,
      files: result.files,
      commitToHistory: result.commitToHistory,
      syncableElements: result.syncableElements || result.elements,
    };
  }

  // 获取可用动作
  getAvailableActions(
    elements: readonly ExcalidrawElement[],
    appState: AppState
  ): ExcalidrawAction[] {
    return Array.from(this.actions.values()).filter(action =>
      !action.predicate || action.predicate(elements, appState, {})
    );
  }

  // 检查动作是否可用
  isActionAvailable(
    actionName: ActionName,
    elements: readonly ExcalidrawElement[],
    appState: AppState
  ): boolean {
    const action = this.actions.get(actionName);
    return action ? (!action.predicate || action.predicate(elements, appState, {})) : false;
  }

  // 获取动作
  getAction(actionName: ActionName): ExcalidrawAction | undefined {
    return this.actions.get(actionName);
  }
}
```

## 具体Action实现

### 基础编辑Action

```typescript
// packages/excalidraw/actions/actionDeleteSelection.tsx
export const actionDeleteSelected: ExcalidrawAction = {
  name: "deleteSelectedElements",
  icon: TrashIcon,
  keywords: ["delete", "remove", "trash", "bin"],
  perform: (elements, appState) => {
    const selectedElements = getSelectedElements(elements, appState);

    if (selectedElements.length === 0) {
      return { elements, appState };
    }

    const selectedElementIds = new Set(selectedElements.map(el => el.id));

    // 删除选中元素
    const newElements = elements.filter(el => !selectedElementIds.has(el.id));

    // 清除选择状态
    const newAppState = {
      ...appState,
      selectedElementIds: {},
      selectedGroupIds: {},
    };

    return {
      elements: newElements,
      appState: newAppState,
      commitToHistory: true,
    };
  },
  predicate: (elements, appState) => {
    return getSelectedElements(elements, appState).length > 0;
  },
  keyTest: (event) => event.key === "Delete" || event.key === "Backspace",
  contextItemLabel: "Delete",
};

// packages/excalidraw/actions/actionDuplicateSelection.tsx
export const actionDuplicate: ExcalidrawAction = {
  name: "duplicateSelection",
  icon: DuplicateIcon,
  keywords: ["duplicate", "copy", "clone"],
  perform: (elements, appState) => {
    const selectedElements = getSelectedElements(elements, appState);

    if (selectedElements.length === 0) {
      return { elements, appState };
    }

    // 复制元素
    const duplicatedElements = selectedElements.map(element => {
      const newElement = duplicateElement(
        null,
        new Map(),
        element,
        {
          x: element.x + 10,
          y: element.y + 10,
        }
      );

      return newElement;
    });

    // 添加到元素列表
    const newElements = [...elements, ...duplicatedElements];

    // 选择复制的元素
    const newSelectedElementIds: Record<string, true> = {};
    duplicatedElements.forEach(element => {
      newSelectedElementIds[element.id] = true;
    });

    const newAppState = {
      ...appState,
      selectedElementIds: newSelectedElementIds,
    };

    return {
      elements: newElements,
      appState: newAppState,
      commitToHistory: true,
    };
  },
  predicate: (elements, appState) => {
    return getSelectedElements(elements, appState).length > 0;
  },
  keyTest: (event) => {
    return event.ctrlKey && event.key === "d" && !event.shiftKey;
  },
  contextItemLabel: "Duplicate",
};
```

### 样式修改Action

```typescript
// packages/excalidraw/actions/actionProperties.tsx
const createChangePropertyAction = <T extends keyof ExcalidrawElement>(
  property: T,
  value: ExcalidrawElement[T],
  commitToHistory: boolean = true
): ExcalidrawAction => ({
  name: `change${capitalize(property)}` as ActionName,
  perform: (elements, appState) => {
    const selectedElements = getSelectedElements(elements, appState);

    if (selectedElements.length === 0) {
      return { elements, appState };
    }

    const newElements = elements.map(element => {
      if (appState.selectedElementIds[element.id]) {
        return {
          ...element,
          [property]: value,
          versionNonce: randomInteger(),
        };
      }
      return element;
    });

    return {
      elements: newElements,
      appState,
      commitToHistory,
    };
  },
  predicate: (elements, appState) => {
    return getSelectedElements(elements, appState).length > 0;
  },
});

export const actionChangeStrokeColor: ExcalidrawAction = {
  name: "changeStrokeColor",
  perform: (elements, appState, formData) => {
    const selectedElements = getSelectedElements(elements, appState);

    if (selectedElements.length === 0) {
      return { elements, appState };
    }

    const strokeColor = formData?.strokeColor || appState.currentItemStrokeColor;

    const newElements = elements.map(element => {
      if (appState.selectedElementIds[element.id]) {
        return {
          ...element,
          strokeColor,
          versionNonce: randomInteger(),
        };
      }
      return element;
    });

    return {
      elements: newElements,
      appState: {
        ...appState,
        currentItemStrokeColor: strokeColor,
      },
      commitToHistory: true,
    };
  },
  predicate: (elements, appState) => {
    return getSelectedElements(elements, appState).length > 0;
  },
};
```

### 层级管理Action

```typescript
// packages/excalidraw/actions/actionZindex.tsx
export const actionBringToFront: ExcalidrawAction = {
  name: "bringToFront",
  icon: BringToFrontIcon,
  keywords: ["front", "forward", "top"],
  perform: (elements, appState) => {
    const selectedElements = getSelectedElements(elements, appState);

    if (selectedElements.length === 0) {
      return { elements, appState };
    }

    const selectedElementIds = new Set(selectedElements.map(el => el.id));

    // 分离选中和未选中的元素
    const nonSelectedElements = elements.filter(el => !selectedElementIds.has(el.id));

    // 将选中元素放到最前面
    const newElements = [...nonSelectedElements, ...selectedElements];

    return {
      elements: newElements,
      appState,
      commitToHistory: true,
    };
  },
  predicate: (elements, appState) => {
    const selectedElements = getSelectedElements(elements, appState);

    if (selectedElements.length === 0) {
      return false;
    }

    // 检查是否已经在最前面
    const lastElements = elements.slice(-selectedElements.length);
    const selectedIds = new Set(selectedElements.map(el => el.id));

    return !lastElements.every(el => selectedIds.has(el.id));
  },
  keyTest: (event) => {
    return event.ctrlKey && event.shiftKey && event.key === "]";
  },
  contextItemLabel: "Bring to front",
};

export const actionSendToBack: ExcalidrawAction = {
  name: "sendToBack",
  icon: SendToBackIcon,
  keywords: ["back", "backward", "bottom"],
  perform: (elements, appState) => {
    const selectedElements = getSelectedElements(elements, appState);

    if (selectedElements.length === 0) {
      return { elements, appState };
    }

    const selectedElementIds = new Set(selectedElements.map(el => el.id));

    // 分离选中和未选中的元素
    const nonSelectedElements = elements.filter(el => !selectedElementIds.has(el.id));

    // 将选中元素放到最后面
    const newElements = [...selectedElements, ...nonSelectedElements];

    return {
      elements: newElements,
      appState,
      commitToHistory: true,
    };
  },
  predicate: (elements, appState) => {
    const selectedElements = getSelectedElements(elements, appState);

    if (selectedElements.length === 0) {
      return false;
    }

    // 检查是否已经在最后面
    const firstElements = elements.slice(0, selectedElements.length);
    const selectedIds = new Set(selectedElements.map(el => el.id));

    return !firstElements.every(el => selectedIds.has(el.id));
  },
  keyTest: (event) => {
    return event.ctrlKey && event.shiftKey && event.key === "[";
  },
  contextItemLabel: "Send to back",
};
```

## 历史管理系统

### History管理器

```typescript
// packages/excalidraw/history.ts
export interface HistoryEntry {
  elements: readonly ExcalidrawElement[];
  appState: AppState;
  timestamp: number;
  actionName?: ActionName;
}

export class HistoryManager {
  private undoStack: HistoryEntry[] = [];
  private redoStack: HistoryEntry[] = [];
  private maxHistorySize: number = 100;
  private currentSnapshot: HistoryEntry | null = null;

  // 创建快照
  createSnapshot(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    actionName?: ActionName
  ) {
    this.currentSnapshot = {
      elements: deepCloneElements(elements),
      appState: deepCloneAppState(appState),
      timestamp: Date.now(),
      actionName,
    };
  }

  // 提交快照
  commitSnapshot(
    elements: readonly ExcalidrawElement[],
    appState: AppState
  ) {
    if (!this.currentSnapshot) {
      return;
    }

    // 检查是否有实际变化
    if (this.hasChanges(this.currentSnapshot, elements, appState)) {
      this.undoStack.push(this.currentSnapshot);

      // 限制历史栈大小
      if (this.undoStack.length > this.maxHistorySize) {
        this.undoStack.shift();
      }

      // 清空重做栈
      this.redoStack = [];
    }

    this.currentSnapshot = null;
  }

  // 回滚快照
  rollbackSnapshot() {
    this.currentSnapshot = null;
  }

  // 撤销
  undo(): HistoryEntry | null {
    if (this.undoStack.length === 0) {
      return null;
    }

    const entry = this.undoStack.pop()!;
    return entry;
  }

  // 重做
  redo(): HistoryEntry | null {
    if (this.redoStack.length === 0) {
      return null;
    }

    const entry = this.redoStack.pop()!;
    return entry;
  }

  // 推入重做栈
  pushToRedoStack(entry: HistoryEntry) {
    this.redoStack.push(entry);
  }

  // 检查是否有变化
  private hasChanges(
    snapshot: HistoryEntry,
    elements: readonly ExcalidrawElement[],
    appState: AppState
  ): boolean {
    // 元素数量变化
    if (snapshot.elements.length !== elements.length) {
      return true;
    }

    // 元素内容变化
    for (let i = 0; i < elements.length; i++) {
      if (!this.elementsEqual(snapshot.elements[i], elements[i])) {
        return true;
      }
    }

    // 关键状态变化
    const keyAppStateFields: (keyof AppState)[] = [
      'selectedElementIds',
      'viewBackgroundColor',
      'exportBackground',
      'gridSize',
      'theme',
    ];

    for (const field of keyAppStateFields) {
      if (!this.deepEqual(snapshot.appState[field], appState[field])) {
        return true;
      }
    }

    return false;
  }

  // 比较元素是否相等
  private elementsEqual(a: ExcalidrawElement, b: ExcalidrawElement): boolean {
    // 关键属性比较
    const keyFields: (keyof ExcalidrawElement)[] = [
      'id', 'x', 'y', 'width', 'height', 'angle',
      'strokeColor', 'backgroundColor', 'fillStyle',
      'strokeWidth', 'roughness', 'opacity', 'isDeleted'
    ];

    for (const field of keyFields) {
      if (a[field] !== b[field]) {
        return false;
      }
    }

    // 类型特定属性比较
    if (a.type !== b.type) {
      return false;
    }

    switch (a.type) {
      case 'text':
        return (a as ExcalidrawTextElement).text === (b as ExcalidrawTextElement).text;
      case 'freedraw':
        return this.deepEqual(
          (a as ExcalidrawFreeDrawElement).points,
          (b as ExcalidrawFreeDrawElement).points
        );
      case 'line':
      case 'arrow':
        return this.deepEqual(
          (a as ExcalidrawLinearElement).points,
          (b as ExcalidrawLinearElement).points
        );
      default:
        return true;
    }
  }

  // 深度比较
  private deepEqual(a: any, b: any): boolean {
    if (a === b) return true;
    if (a == null || b == null) return false;
    if (Array.isArray(a) && Array.isArray(b)) {
      if (a.length !== b.length) return false;
      for (let i = 0; i < a.length; i++) {
        if (!this.deepEqual(a[i], b[i])) return false;
      }
      return true;
    }
    if (typeof a === 'object' && typeof b === 'object') {
      const keysA = Object.keys(a);
      const keysB = Object.keys(b);
      if (keysA.length !== keysB.length) return false;
      for (const key of keysA) {
        if (!keysB.includes(key) || !this.deepEqual(a[key], b[key])) {
          return false;
        }
      }
      return true;
    }
    return false;
  }

  // 获取历史统计
  getStats() {
    return {
      undoCount: this.undoStack.length,
      redoCount: this.redoStack.length,
      maxSize: this.maxHistorySize,
      currentSnapshot: this.currentSnapshot !== null,
    };
  }

  // 清空历史
  clear() {
    this.undoStack = [];
    this.redoStack = [];
    this.currentSnapshot = null;
  }

  // 设置最大历史大小
  setMaxHistorySize(size: number) {
    this.maxHistorySize = size;

    // 截断现有历史
    if (this.undoStack.length > size) {
      this.undoStack = this.undoStack.slice(-size);
    }
    if (this.redoStack.length > size) {
      this.redoStack = this.redoStack.slice(-size);
    }
  }
}

// 深度克隆元素
function deepCloneElements(elements: readonly ExcalidrawElement[]): ExcalidrawElement[] {
  return elements.map(element => ({ ...element }));
}

// 深度克隆应用状态
function deepCloneAppState(appState: AppState): AppState {
  return {
    ...appState,
    selectedElementIds: { ...appState.selectedElementIds },
    selectedGroupIds: { ...appState.selectedGroupIds },
  };
}
```

### 撤销重做Action

```typescript
// packages/excalidraw/actions/actionHistory.tsx
export const actionUndo: ExcalidrawAction = {
  name: "undo",
  icon: UndoIcon,
  keywords: ["undo", "revert"],
  perform: (elements, appState, formData, app) => {
    const historyManager = app.historyManager;

    if (!historyManager) {
      return { elements, appState };
    }

    // 创建当前状态的快照用于重做
    const currentSnapshot: HistoryEntry = {
      elements: deepCloneElements(elements),
      appState: deepCloneAppState(appState),
      timestamp: Date.now(),
    };

    // 执行撤销
    const undoEntry = historyManager.undo();

    if (!undoEntry) {
      return { elements, appState };
    }

    // 将当前状态推入重做栈
    historyManager.pushToRedoStack(currentSnapshot);

    return {
      elements: undoEntry.elements,
      appState: undoEntry.appState,
      commitToHistory: false, // 不提交到历史，避免循环
    };
  },
  predicate: (elements, appState, props, app) => {
    return app?.historyManager?.getStats().undoCount > 0;
  },
  keyTest: (event) => {
    return (event.ctrlKey || event.metaKey) && event.key === "z" && !event.shiftKey;
  },
  contextItemLabel: "Undo",
};

export const actionRedo: ExcalidrawAction = {
  name: "redo",
  icon: RedoIcon,
  keywords: ["redo", "repeat"],
  perform: (elements, appState, formData, app) => {
    const historyManager = app.historyManager;

    if (!historyManager) {
      return { elements, appState };
    }

    // 执行重做
    const redoEntry = historyManager.redo();

    if (!redoEntry) {
      return { elements, appState };
    }

    // 将当前状态推入撤销栈
    historyManager.createSnapshot(elements, appState);
    historyManager.commitSnapshot(redoEntry.elements, redoEntry.appState);

    return {
      elements: redoEntry.elements,
      appState: redoEntry.appState,
      commitToHistory: false,
    };
  },
  predicate: (elements, appState, props, app) => {
    return app?.historyManager?.getStats().redoCount > 0;
  },
  keyTest: (event) => {
    return (
      ((event.ctrlKey || event.metaKey) && event.shiftKey && event.key === "Z") ||
      ((event.ctrlKey || event.metaKey) && event.key === "y")
    );
  },
  contextItemLabel: "Redo",
};
```

## 宏录制与回放

### 宏管理器

```typescript
// 宏定义
export interface Macro {
  id: string;
  name: string;
  actions: RecordedAction[];
  createdAt: number;
  description?: string;
}

export interface RecordedAction {
  actionName: ActionName;
  timestamp: number;
  formData?: any;
  elementContext?: {
    selectedElementIds: string[];
    elementCount: number;
  };
}

export class MacroManager {
  private macros: Map<string, Macro> = new Map();
  private isRecording = false;
  private currentRecording: RecordedAction[] = [];
  private recordingStartTime = 0;
  private actionManager: ActionManager;

  constructor(actionManager: ActionManager) {
    this.actionManager = actionManager;
  }

  // 开始录制宏
  startRecording(name: string, description?: string) {
    if (this.isRecording) {
      this.stopRecording();
    }

    this.isRecording = true;
    this.currentRecording = [];
    this.recordingStartTime = Date.now();

    console.log(`Started recording macro: ${name}`);
  }

  // 停止录制宏
  stopRecording(name?: string): Macro | null {
    if (!this.isRecording) {
      return null;
    }

    this.isRecording = false;

    if (this.currentRecording.length === 0) {
      return null;
    }

    const macro: Macro = {
      id: this.generateId(),
      name: name || `Macro ${Date.now()}`,
      actions: [...this.currentRecording],
      createdAt: Date.now(),
    };

    this.macros.set(macro.id, macro);
    this.currentRecording = [];

    console.log(`Stopped recording macro: ${macro.name}`, macro);
    return macro;
  }

  // 记录动作
  recordAction(
    actionName: ActionName,
    formData?: any,
    elements?: readonly ExcalidrawElement[],
    appState?: AppState
  ) {
    if (!this.isRecording) {
      return;
    }

    const recordedAction: RecordedAction = {
      actionName,
      timestamp: Date.now() - this.recordingStartTime,
      formData,
    };

    // 记录元素上下文
    if (elements && appState) {
      recordedAction.elementContext = {
        selectedElementIds: Object.keys(appState.selectedElementIds),
        elementCount: elements.length,
      };
    }

    this.currentRecording.push(recordedAction);
  }

  // 播放宏
  async playMacro(
    macroId: string,
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    app: AppClassProperties,
    options: {
      speed?: number;          // 播放速度倍率
      pauseBetweenActions?: number; // 动作间暂停时间（ms）
      skipTimings?: boolean;   // 跳过时间间隔
    } = {}
  ): Promise<ActionResult | null> {
    const macro = this.macros.get(macroId);
    if (!macro) {
      console.warn(`Macro ${macroId} not found`);
      return null;
    }

    const { speed = 1, pauseBetweenActions = 0, skipTimings = false } = options;

    let currentElements = elements;
    let currentAppState = appState;

    console.log(`Playing macro: ${macro.name}`);

    for (let i = 0; i < macro.actions.length; i++) {
      const recordedAction = macro.actions[i];

      // 计算等待时间
      if (!skipTimings && i > 0) {
        const prevTimestamp = macro.actions[i - 1].timestamp;
        const currentTimestamp = recordedAction.timestamp;
        const waitTime = (currentTimestamp - prevTimestamp) / speed;

        if (waitTime > 0) {
          await this.sleep(waitTime);
        }
      }

      // 执行动作
      try {
        const result = await this.actionManager.executeAction(
          recordedAction.actionName,
          currentElements,
          currentAppState,
          recordedAction.formData,
          app
        );

        if (result) {
          currentElements = result.elements || currentElements;
          currentAppState = { ...currentAppState, ...result.appState };
        }

        console.log(`Executed action: ${recordedAction.actionName}`);
      } catch (error) {
        console.error(`Failed to execute action: ${recordedAction.actionName}`, error);
        break;
      }

      // 动作间暂停
      if (pauseBetweenActions > 0) {
        await this.sleep(pauseBetweenActions);
      }
    }

    console.log(`Finished playing macro: ${macro.name}`);

    return {
      elements: currentElements,
      appState: currentAppState,
      commitToHistory: true,
    };
  }

  // 获取所有宏
  getMacros(): Macro[] {
    return Array.from(this.macros.values()).sort((a, b) => b.createdAt - a.createdAt);
  }

  // 删除宏
  deleteMacro(macroId: string): boolean {
    return this.macros.delete(macroId);
  }

  // 重命名宏
  renameMacro(macroId: string, newName: string): boolean {
    const macro = this.macros.get(macroId);
    if (macro) {
      macro.name = newName;
      return true;
    }
    return false;
  }

  // 导出宏
  exportMacro(macroId: string): string | null {
    const macro = this.macros.get(macroId);
    return macro ? JSON.stringify(macro, null, 2) : null;
  }

  // 导入宏
  importMacro(macroData: string): boolean {
    try {
      const macro: Macro = JSON.parse(macroData);

      // 验证宏格式
      if (!macro.id || !macro.name || !Array.isArray(macro.actions)) {
        throw new Error("Invalid macro format");
      }

      // 生成新ID避免冲突
      macro.id = this.generateId();

      this.macros.set(macro.id, macro);
      return true;
    } catch (error) {
      console.error("Failed to import macro:", error);
      return false;
    }
  }

  // 获取录制状态
  getRecordingStatus() {
    return {
      isRecording: this.isRecording,
      actionCount: this.currentRecording.length,
      duration: this.isRecording ? Date.now() - this.recordingStartTime : 0,
    };
  }

  // 工具方法
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  private generateId(): string {
    return Math.random().toString(36).substr(2, 9);
  }
}
```

## 思考题

1. **Action粒度如何设计？** 原子操作与复合操作的平衡？

2. **历史压缩如何实现？** 如何减少历史栈的内存占用？

3. **协作冲突如何解决？** 多用户同时执行Action时的冲突处理？

4. **性能优化策略？** 大量Action执行时的性能保证？

5. **Action扩展机制？** 如何支持插件开发者添加自定义Action？

## 实践练习

### 练习 1：实现批量操作Action
创建批量处理系统：
- 批量样式修改
- 批量变换操作
- 操作进度显示

### 练习 2：实现条件Action
实现基于条件的Action系统：
- 条件判断逻辑
- 分支执行流程
- 错误处理机制

### 练习 3：实现Action性能监控
添加性能监控功能：
- 执行时间统计
- 内存使用跟踪
- 性能报告生成

## 总结

Excalidraw的Action系统展现了企业级应用架构的特点：

1. **命令模式应用**：将所有操作封装为可执行、可撤销的命令
2. **历史管理**：完善的撤销重做机制支持复杂的编辑工作流
3. **可扩展架构**：支持动态注册新Action和自定义操作
4. **状态管理**：智能的状态比较和差异检测
5. **协作支持**：为实时协作提供了Action广播基础

这套系统为构建专业级编辑器提供了可靠的操作管理基础。

## 下一步

下一章将探讨历史管理的高级特性，包括状态压缩、增量存储、以及协作环境下的历史同步机制。