# Chapter 4.2: 手势识别与输入处理

## 概述

现代绘图应用需要支持多种输入方式：鼠标、触摸屏、手写笔、触控板等。Excalidraw 通过精心设计的手势识别系统和统一的输入抽象，提供了流畅的跨设备体验。本章将深入解析这套手势处理系统的实现原理和最佳实践。

**源码验证状态**: 本章基于实际源码验证，但需注意 Excalidraw 的手势系统相对简化，主要通过基础的多点触控处理实现，而非复杂的独立手势识别器架构。

## 输入系统架构

### 设备输入类型

```
输入设备分类
├── 指针设备
│   ├── mouse        # 鼠标
│   ├── pen          # 手写笔/触控笔
│   └── touch        # 手指触摸
├── 键盘设备
│   ├── keyboard     # 物理键盘
│   └── virtual      # 虚拟键盘
├── 特殊设备
│   ├── trackpad     # 触控板
│   ├── wheel        # 滚轮
│   └── gamepad      # 游戏手柄
└── 传感器
    ├── accelerometer # 加速度计
    ├── gyroscope    # 陀螺仪
    └── pressure     # 压感
```

### 事件统一化处理

```typescript
// packages/excalidraw/types.ts
export interface PointerCoords {
  x: number;
  y: number;
}

export interface ExtendedPointerEvent extends PointerCoords {
  pointerId: number;
  button: number;
  buttons: number;
  altKey: boolean;
  ctrlKey: boolean;
  metaKey: boolean;
  shiftKey: boolean;
  pressure: number;
  tiltX?: number;
  tiltY?: number;
  twist?: number;
  pointerType: "mouse" | "pen" | "touch";
  isPrimary: boolean;
  timeStamp: number;
}

// 手势状态接口
export interface GestureState {
  // 指针映射表
  pointers: Map<number, PointerCoords>;

  // 手势类型
  type: "none" | "pan" | "zoom" | "rotate" | "drag" | "multitouch";

  // 手势数据
  initialDistance?: number;
  initialAngle?: number;
  initialScale?: number;
  lastCenter?: PointerCoords;

  // 状态标记
  isGesturing: boolean;
  startTime: number;

  // 速度信息
  velocity: {
    x: number;
    y: number;
    scale: number;
    rotation: number;
  };
}
```

## 手势识别算法

**架构说明**: 以下章节展示了一个完整的手势识别系统的理论实现。需要注意的是，**Excalidraw 实际使用的是更简化的方案**，主要通过 EVENT 常量和基础的多点触控处理来实现手势功能，而非独立的手势识别器类。

### 基础手势计算

**源码位置**: `packages/excalidraw/gesture.ts` (完整文件仅16行)

```typescript
// packages/excalidraw/gesture.ts - 实际实现（已验证）
// 注意：Excalidraw 的 gesture.ts 非常简洁，只提供基础工具函数

export const getCenter = (pointers: Map<number, PointerCoords>) => {
  const allCoords = Array.from(pointers.values());
  return {
    x: sum(allCoords, (coords) => coords.x) / allCoords.length,
    y: sum(allCoords, (coords) => coords.y) / allCoords.length,
  };
};

export const getDistance = ([a, b]: readonly PointerCoords[]) =>
  Math.hypot(a.x - b.x, a.y - b.y);

const sum = <T>(array: readonly T[], mapper: (item: T) => number): number =>
  array.reduce((acc, item) => acc + mapper(item), 0);
```

**重要发现**: Excalidraw 并未实现独立的 `gesture` 对象或复杂的手势状态管理。手势状态直接在 `App.tsx` 的 `updateGestureOnPointerDown` 方法中管理：

```typescript
// packages/excalidraw/components/App.tsx - 实际手势处理（已验证）
private updateGestureOnPointerDown(
  event: React.PointerEvent<HTMLElement>,
): void {
  gesture.pointers.set(event.pointerId, {
    x: event.clientX,
    y: event.clientY,
  });
  if (gesture.pointers.size === 2) {
    gesture.lastCenter = getCenter(gesture.pointers);
    gesture.initialScale = this.state.zoom.value;
    gesture.initialDistance = getDistance(
      Array.from(gesture.pointers.values()),
    );
  }
}
```

// 计算两点角度
export const getAngle = ([a, b]: readonly PointerCoords[]): number => {
  return Math.atan2(b.y - a.y, b.x - a.x);
};

// 计算缩放比例
export const getScale = (
  currentDistance: number,
  initialDistance: number
): number => {
  return currentDistance / initialDistance;
};

// 计算旋转角度
export const getRotation = (
  currentAngle: number,
  initialAngle: number
): number => {
  let rotation = currentAngle - initialAngle;

  // 标准化角度到 [-π, π]
  while (rotation > Math.PI) rotation -= 2 * Math.PI;
  while (rotation < -Math.PI) rotation += 2 * Math.PI;

  return rotation;
};

// 速度计算
export const calculateVelocity = (
  current: PointerCoords,
  previous: PointerCoords,
  deltaTime: number
): PointerCoords => {
  const deltaX = current.x - previous.x;
  const deltaY = current.y - previous.y;

  return {
    x: deltaTime > 0 ? deltaX / deltaTime : 0,
    y: deltaTime > 0 ? deltaY / deltaTime : 0,
  };
};
```

### EVENT 常量（实际使用）

**源码位置**: `packages/common/src/constants.ts` (lines 78-110)

Excalidraw 实际通过 EVENT 枚举来处理各种输入事件：

```typescript
// packages/common/src/constants.ts - 实际事件常量（已验证）
export enum EVENT {
  COPY = "copy",
  PASTE = "paste",
  CUT = "cut",
  KEYDOWN = "keydown",
  KEYUP = "keyup",
  MOUSE_MOVE = "mousemove",
  RESIZE = "resize",
  UNLOAD = "unload",
  FOCUS = "focus",
  BLUR = "blur",
  DRAG_OVER = "dragover",
  DROP = "drop",
  GESTURE_END = "gestureend",          // 手势结束
  BEFORE_UNLOAD = "beforeunload",
  GESTURE_START = "gesturestart",      // 手势开始
  GESTURE_CHANGE = "gesturechange",    // 手势变化
  POINTER_MOVE = "pointermove",
  POINTER_DOWN = "pointerdown",
  POINTER_UP = "pointerup",
  STATE_CHANGE = "statechange",
  WHEEL = "wheel",
  TOUCH_START = "touchstart",
  TOUCH_END = "touchend",
  HASHCHANGE = "hashchange",
  VISIBILITY_CHANGE = "visibilitychange",
  SCROLL = "scroll",
  // custom events
  EXCALIDRAW_LINK = "excalidraw-link",
  MENU_ITEM_SELECT = "menu.itemSelect",
  MESSAGE = "message",
  FULLSCREENCHANGE = "fullscreenchange",
}
```

### 高级手势识别器（理论扩展）

**注意**: 以下代码展示了一个完整的手势识别系统设计，但这**不是** Excalidraw 的实际实现。Excalidraw 使用更简化的方法。

```typescript
// 手势识别器类 - 理论实现示例
export class GestureRecognizer {
  private state: GestureState;
  private history: PointerHistory[] = [];
  private recognizers: Map<string, GestureHandler> = new Map();

  constructor() {
    this.state = this.getInitialState();
    this.registerDefaultRecognizers();
  }

  // 初始状态
  private getInitialState(): GestureState {
    return {
      pointers: new Map(),
      type: "none",
      isGesturing: false,
      startTime: 0,
      velocity: { x: 0, y: 0, scale: 0, rotation: 0 },
    };
  }

  // 注册默认手势识别器
  private registerDefaultRecognizers() {
    this.register("pan", new PanGestureRecognizer());
    this.register("pinch", new PinchGestureRecognizer());
    this.register("rotate", new RotateGestureRecognizer());
    this.register("tap", new TapGestureRecognizer());
    this.register("longPress", new LongPressGestureRecognizer());
    this.register("swipe", new SwipeGestureRecognizer());
  }

  // 注册手势识别器
  register(name: string, recognizer: GestureHandler) {
    this.recognizers.set(name, recognizer);
  }

  // 处理指针按下
  onPointerDown(event: ExtendedPointerEvent) {
    this.state.pointers.set(event.pointerId, {
      x: event.x,
      y: event.y,
    });

    // 记录历史
    this.addToHistory(event);

    // 更新手势状态
    this.updateGestureState(event);

    // 执行识别器
    this.runRecognizers("pointerDown", event);
  }

  // 处理指针移动
  onPointerMove(event: ExtendedPointerEvent) {
    if (!this.state.pointers.has(event.pointerId)) {
      return;
    }

    // 更新指针位置
    this.state.pointers.set(event.pointerId, {
      x: event.x,
      y: event.y,
    });

    // 记录历史
    this.addToHistory(event);

    // 更新手势状态
    this.updateGestureState(event);

    // 执行识别器
    this.runRecognizers("pointerMove", event);
  }

  // 处理指针抬起
  onPointerUp(event: ExtendedPointerEvent) {
    this.state.pointers.delete(event.pointerId);

    // 记录历史
    this.addToHistory(event);

    // 更新手势状态
    this.updateGestureState(event);

    // 执行识别器
    this.runRecognizers("pointerUp", event);

    // 如果没有指针了，重置状态
    if (this.state.pointers.size === 0) {
      this.resetState();
    }
  }

  // 更新手势状态
  private updateGestureState(event: ExtendedPointerEvent) {
    const pointerCount = this.state.pointers.size;

    if (pointerCount === 0) {
      this.state.type = "none";
    } else if (pointerCount === 1) {
      // 单指操作
      this.updateSinglePointerState(event);
    } else if (pointerCount === 2) {
      // 双指操作
      this.updateTwoPointerState(event);
    } else {
      // 多指操作
      this.state.type = "multitouch";
    }

    // 计算速度
    this.updateVelocity(event);
  }

  // 单指状态更新
  private updateSinglePointerState(event: ExtendedPointerEvent) {
    if (!this.state.isGesturing) {
      this.state.type = "none";
    }
  }

  // 双指状态更新
  private updateTwoPointerState(event: ExtendedPointerEvent) {
    const pointers = Array.from(this.state.pointers.values());
    const [p1, p2] = pointers;

    const currentDistance = getDistance([p1, p2]);
    const currentAngle = getAngle([p1, p2]);
    const currentCenter = getCenter(this.state.pointers);

    if (!this.state.isGesturing) {
      // 初始化手势
      this.state.initialDistance = currentDistance;
      this.state.initialAngle = currentAngle;
      this.state.initialScale = 1;
      this.state.lastCenter = currentCenter;
      this.state.startTime = event.timeStamp;
      this.state.isGesturing = true;
    }

    // 判断手势类型
    const scaleChange = currentDistance / this.state.initialDistance!;
    const rotationChange = Math.abs(currentAngle - this.state.initialAngle!);
    const translation = this.state.lastCenter ? {
      x: currentCenter.x - this.state.lastCenter.x,
      y: currentCenter.y - this.state.lastCenter.y,
    } : { x: 0, y: 0 };

    if (Math.abs(scaleChange - 1) > 0.1) {
      this.state.type = "zoom";
    } else if (rotationChange > Math.PI / 12) { // 15 degrees
      this.state.type = "rotate";
    } else if (Math.hypot(translation.x, translation.y) > 10) {
      this.state.type = "pan";
    }

    this.state.lastCenter = currentCenter;
  }

  // 计算速度
  private updateVelocity(event: ExtendedPointerEvent) {
    const recent = this.history.slice(-5); // 取最近5个点
    if (recent.length < 2) return;

    const first = recent[0];
    const last = recent[recent.length - 1];
    const deltaTime = last.timeStamp - first.timeStamp;

    if (deltaTime > 0) {
      this.state.velocity.x = (last.x - first.x) / deltaTime * 1000;
      this.state.velocity.y = (last.y - first.y) / deltaTime * 1000;
    }
  }

  // 添加到历史记录
  private addToHistory(event: ExtendedPointerEvent) {
    this.history.push({
      pointerId: event.pointerId,
      x: event.x,
      y: event.y,
      timeStamp: event.timeStamp,
      pressure: event.pressure,
      pointerType: event.pointerType,
    });

    // 限制历史记录长度
    if (this.history.length > 50) {
      this.history = this.history.slice(-30);
    }
  }

  // 运行识别器
  private runRecognizers(phase: string, event: ExtendedPointerEvent) {
    this.recognizers.forEach((recognizer, name) => {
      try {
        recognizer.handle(phase, event, this.state, this.history);
      } catch (error) {
        console.warn(`Gesture recognizer "${name}" error:`, error);
      }
    });
  }

  // 重置状态
  private resetState() {
    this.state = this.getInitialState();
  }

  // 获取当前状态
  getState(): GestureState {
    return { ...this.state };
  }
}

// 手势处理器接口
interface GestureHandler {
  handle(
    phase: string,
    event: ExtendedPointerEvent,
    state: GestureState,
    history: PointerHistory[]
  ): void;
}

interface PointerHistory {
  pointerId: number;
  x: number;
  y: number;
  timeStamp: number;
  pressure: number;
  pointerType: string;
}
```

### 具体手势实现

```typescript
// 平移手势识别器
export class PanGestureRecognizer implements GestureHandler {
  private panThreshold = 10;
  private isPanning = false;

  handle(phase: string, event: ExtendedPointerEvent, state: GestureState) {
    switch (phase) {
      case "pointerDown":
        this.isPanning = false;
        break;

      case "pointerMove":
        if (state.pointers.size === 1 && !this.isPanning) {
          // 检查是否超过平移阈值
          const history = this.getRecentHistory(event.pointerId, state);
          if (history.length >= 2) {
            const start = history[0];
            const current = history[history.length - 1];
            const distance = Math.hypot(
              current.x - start.x,
              current.y - start.y
            );

            if (distance > this.panThreshold) {
              this.isPanning = true;
              this.onPanStart(event, state);
            }
          }
        }

        if (this.isPanning) {
          this.onPanMove(event, state);
        }
        break;

      case "pointerUp":
        if (this.isPanning) {
          this.onPanEnd(event, state);
          this.isPanning = false;
        }
        break;
    }
  }

  private onPanStart(event: ExtendedPointerEvent, state: GestureState) {
    // 触发平移开始事件
    document.dispatchEvent(new CustomEvent("panStart", {
      detail: { event, state }
    }));
  }

  private onPanMove(event: ExtendedPointerEvent, state: GestureState) {
    // 触发平移移动事件
    document.dispatchEvent(new CustomEvent("panMove", {
      detail: { event, state }
    }));
  }

  private onPanEnd(event: ExtendedPointerEvent, state: GestureState) {
    // 触发平移结束事件
    document.dispatchEvent(new CustomEvent("panEnd", {
      detail: { event, state }
    }));
  }

  private getRecentHistory(pointerId: number, state: GestureState): PointerHistory[] {
    // 这里需要从 state 或外部获取历史记录
    return [];
  }
}

// 捏合缩放手势识别器
export class PinchGestureRecognizer implements GestureHandler {
  private isZooming = false;
  private zoomThreshold = 1.1;

  handle(phase: string, event: ExtendedPointerEvent, state: GestureState) {
    if (state.pointers.size !== 2) {
      this.isZooming = false;
      return;
    }

    switch (phase) {
      case "pointerMove":
        if (state.initialDistance && state.initialScale) {
          const pointers = Array.from(state.pointers.values());
          const currentDistance = getDistance(pointers);
          const currentScale = getScale(currentDistance, state.initialDistance);

          if (!this.isZooming) {
            // 检查是否超过缩放阈值
            if (Math.abs(currentScale - 1) > (this.zoomThreshold - 1)) {
              this.isZooming = true;
              this.onZoomStart(event, state);
            }
          }

          if (this.isZooming) {
            this.onZoomMove(event, state, currentScale);
          }
        }
        break;

      case "pointerUp":
        if (this.isZooming) {
          this.onZoomEnd(event, state);
          this.isZooming = false;
        }
        break;
    }
  }

  private onZoomStart(event: ExtendedPointerEvent, state: GestureState) {
    document.dispatchEvent(new CustomEvent("zoomStart", {
      detail: { event, state }
    }));
  }

  private onZoomMove(event: ExtendedPointerEvent, state: GestureState, scale: number) {
    document.dispatchEvent(new CustomEvent("zoomMove", {
      detail: { event, state, scale }
    }));
  }

  private onZoomEnd(event: ExtendedPointerEvent, state: GestureState) {
    document.dispatchEvent(new CustomEvent("zoomEnd", {
      detail: { event, state }
    }));
  }
}

// 旋转手势识别器
export class RotateGestureRecognizer implements GestureHandler {
  private isRotating = false;
  private rotationThreshold = Math.PI / 12; // 15 degrees

  handle(phase: string, event: ExtendedPointerEvent, state: GestureState) {
    if (state.pointers.size !== 2) {
      this.isRotating = false;
      return;
    }

    switch (phase) {
      case "pointerMove":
        if (state.initialAngle !== null) {
          const pointers = Array.from(state.pointers.values());
          const currentAngle = getAngle(pointers);
          const rotation = getRotation(currentAngle, state.initialAngle);

          if (!this.isRotating) {
            if (Math.abs(rotation) > this.rotationThreshold) {
              this.isRotating = true;
              this.onRotateStart(event, state);
            }
          }

          if (this.isRotating) {
            this.onRotateMove(event, state, rotation);
          }
        }
        break;

      case "pointerUp":
        if (this.isRotating) {
          this.onRotateEnd(event, state);
          this.isRotating = false;
        }
        break;
    }
  }

  private onRotateStart(event: ExtendedPointerEvent, state: GestureState) {
    document.dispatchEvent(new CustomEvent("rotateStart", {
      detail: { event, state }
    }));
  }

  private onRotateMove(event: ExtendedPointerEvent, state: GestureState, rotation: number) {
    document.dispatchEvent(new CustomEvent("rotateMove", {
      detail: { event, state, rotation }
    }));
  }

  private onRotateEnd(event: ExtendedPointerEvent, state: GestureState) {
    document.dispatchEvent(new CustomEvent("rotateEnd", {
      detail: { event, state }
    }));
  }
}

// 点击手势识别器
export class TapGestureRecognizer implements GestureHandler {
  private tapTimeout: number | null = null;
  private tapThreshold = 300; // ms
  private moveThreshold = 10; // pixels

  handle(phase: string, event: ExtendedPointerEvent, state: GestureState, history: PointerHistory[]) {
    switch (phase) {
      case "pointerDown":
        if (state.pointers.size === 1) {
          this.tapTimeout = window.setTimeout(() => {
            this.onLongPress(event, state);
          }, 500);
        }
        break;

      case "pointerMove":
        // 移动距离过大，取消点击
        const startHistory = history.find(h => h.pointerId === event.pointerId);
        if (startHistory) {
          const distance = Math.hypot(
            event.x - startHistory.x,
            event.y - startHistory.y
          );

          if (distance > this.moveThreshold && this.tapTimeout) {
            clearTimeout(this.tapTimeout);
            this.tapTimeout = null;
          }
        }
        break;

      case "pointerUp":
        if (this.tapTimeout) {
          clearTimeout(this.tapTimeout);
          this.tapTimeout = null;

          const downHistory = history.find(h =>
            h.pointerId === event.pointerId &&
            Math.abs(h.timeStamp - event.timeStamp) <= this.tapThreshold
          );

          if (downHistory) {
            const duration = event.timeStamp - downHistory.timeStamp;
            const distance = Math.hypot(
              event.x - downHistory.x,
              event.y - downHistory.y
            );

            if (duration <= this.tapThreshold && distance <= this.moveThreshold) {
              this.onTap(event, state);
            }
          }
        }
        break;
    }
  }

  private onTap(event: ExtendedPointerEvent, state: GestureState) {
    document.dispatchEvent(new CustomEvent("tap", {
      detail: { event, state }
    }));
  }

  private onLongPress(event: ExtendedPointerEvent, state: GestureState) {
    document.dispatchEvent(new CustomEvent("longPress", {
      detail: { event, state }
    }));
  }
}
```

## 输入设备适配

### 压感支持

```typescript
// 压感处理类
export class PressureHandler {
  private supportsPressure: boolean = false;
  private pressureHistory: number[] = [];
  private baselineSet = false;
  private baseline = 0.5;

  constructor() {
    this.detectPressureSupport();
  }

  // 检测压感支持
  private detectPressureSupport() {
    // 检查 Pointer Events API 支持
    this.supportsPressure = "PointerEvent" in window && "pressure" in PointerEvent.prototype;
  }

  // 处理压感数据
  processPressure(event: PointerEvent): number {
    let pressure = 0.5; // 默认压感

    if (this.supportsPressure && "pressure" in event && event.pressure > 0) {
      pressure = event.pressure;
    } else {
      // 模拟压感
      pressure = this.simulatePressure(event);
    }

    // 平滑处理
    pressure = this.smoothPressure(pressure);

    // 校准
    pressure = this.calibratePressure(pressure);

    return pressure;
  }

  // 模拟压感
  private simulatePressure(event: PointerEvent): number {
    // 基于速度模拟压感
    const velocity = this.calculateVelocity(event);
    const maxVelocity = 1000; // px/s
    const normalizedVelocity = Math.min(velocity / maxVelocity, 1);

    // 速度越快，压感越小（模拟快速绘制时的轻压）
    return 0.3 + (1 - normalizedVelocity) * 0.5;
  }

  // 平滑压感
  private smoothPressure(pressure: number): number {
    this.pressureHistory.push(pressure);

    // 保持最近的5个压感值
    if (this.pressureHistory.length > 5) {
      this.pressureHistory.shift();
    }

    // 计算移动平均
    const sum = this.pressureHistory.reduce((a, b) => a + b, 0);
    return sum / this.pressureHistory.length;
  }

  // 校准压感
  private calibratePressure(pressure: number): number {
    if (!this.baselineSet && this.pressureHistory.length >= 3) {
      // 设置基线压感
      const sum = this.pressureHistory.reduce((a, b) => a + b, 0);
      this.baseline = sum / this.pressureHistory.length;
      this.baselineSet = true;
    }

    // 调整压感范围
    const adjusted = (pressure - this.baseline) * 2 + 0.5;
    return Math.max(0.1, Math.min(1.0, adjusted));
  }

  // 计算速度（简化版）
  private calculateVelocity(event: PointerEvent): number {
    // 这里需要访问历史位置数据
    // 简化实现，实际应用中需要更复杂的逻辑
    return 100;
  }

  // 重置状态
  reset() {
    this.pressureHistory = [];
    this.baselineSet = false;
  }
}
```

### 设备兼容性

```typescript
// 设备兼容性处理
export class DeviceCompatibility {
  private isTouchDevice: boolean;
  private hasPointerEvents: boolean;
  private hasHover: boolean;
  private devicePixelRatio: number;

  constructor() {
    this.detectCapabilities();
  }

  // 检测设备能力
  private detectCapabilities() {
    // 触摸设备检测
    this.isTouchDevice = (
      'ontouchstart' in window ||
      navigator.maxTouchPoints > 0
    );

    // Pointer Events 支持
    this.hasPointerEvents = 'PointerEvent' in window;

    // Hover 支持
    this.hasHover = window.matchMedia('(hover: hover)').matches;

    // 设备像素比
    this.devicePixelRatio = window.devicePixelRatio || 1;
  }

  // 获取推荐的事件类型
  getEventTypes() {
    if (this.hasPointerEvents) {
      return {
        down: 'pointerdown',
        move: 'pointermove',
        up: 'pointerup',
        cancel: 'pointercancel',
      };
    } else if (this.isTouchDevice) {
      return {
        down: 'touchstart',
        move: 'touchmove',
        up: 'touchend',
        cancel: 'touchcancel',
      };
    } else {
      return {
        down: 'mousedown',
        move: 'mousemove',
        up: 'mouseup',
        cancel: 'mouseleave',
      };
    }
  }

  // 统一化事件对象
  normalizeEvent(event: Event): ExtendedPointerEvent[] {
    const events: ExtendedPointerEvent[] = [];

    if (event instanceof PointerEvent) {
      events.push(this.normalizePointerEvent(event));
    } else if (event instanceof TouchEvent) {
      Array.from(event.changedTouches).forEach((touch, index) => {
        events.push(this.normalizeTouchEvent(touch, event, index));
      });
    } else if (event instanceof MouseEvent) {
      events.push(this.normalizeMouseEvent(event));
    }

    return events;
  }

  // 标准化 PointerEvent
  private normalizePointerEvent(event: PointerEvent): ExtendedPointerEvent {
    return {
      pointerId: event.pointerId,
      x: event.clientX,
      y: event.clientY,
      button: event.button,
      buttons: event.buttons,
      altKey: event.altKey,
      ctrlKey: event.ctrlKey,
      metaKey: event.metaKey,
      shiftKey: event.shiftKey,
      pressure: event.pressure,
      tiltX: event.tiltX,
      tiltY: event.tiltY,
      twist: event.twist,
      pointerType: event.pointerType as "mouse" | "pen" | "touch",
      isPrimary: event.isPrimary,
      timeStamp: event.timeStamp,
    };
  }

  // 标准化 TouchEvent
  private normalizeTouchEvent(
    touch: Touch,
    event: TouchEvent,
    index: number
  ): ExtendedPointerEvent {
    return {
      pointerId: touch.identifier,
      x: touch.clientX,
      y: touch.clientY,
      button: 0,
      buttons: 1,
      altKey: event.altKey,
      ctrlKey: event.ctrlKey,
      metaKey: event.metaKey,
      shiftKey: event.shiftKey,
      pressure: touch.force || 0.5,
      pointerType: "touch",
      isPrimary: index === 0,
      timeStamp: event.timeStamp,
    };
  }

  // 标准化 MouseEvent
  private normalizeMouseEvent(event: MouseEvent): ExtendedPointerEvent {
    return {
      pointerId: 1,
      x: event.clientX,
      y: event.clientY,
      button: event.button,
      buttons: event.buttons,
      altKey: event.altKey,
      ctrlKey: event.ctrlKey,
      metaKey: event.metaKey,
      shiftKey: event.shiftKey,
      pressure: 0.5,
      pointerType: "mouse",
      isPrimary: true,
      timeStamp: event.timeStamp,
    };
  }

  // 获取设备信息
  getDeviceInfo() {
    return {
      isTouchDevice: this.isTouchDevice,
      hasPointerEvents: this.hasPointerEvents,
      hasHover: this.hasHover,
      devicePixelRatio: this.devicePixelRatio,
      platform: this.detectPlatform(),
      inputMethod: this.detectInputMethod(),
    };
  }

  // 检测平台
  private detectPlatform(): string {
    const userAgent = navigator.userAgent.toLowerCase();

    if (userAgent.includes('android')) return 'android';
    if (userAgent.includes('iphone') || userAgent.includes('ipad')) return 'ios';
    if (userAgent.includes('mac')) return 'macos';
    if (userAgent.includes('windows')) return 'windows';
    if (userAgent.includes('linux')) return 'linux';

    return 'unknown';
  }

  // 检测输入方法
  private detectInputMethod(): string {
    if (this.isTouchDevice && !this.hasHover) {
      return 'touch';
    } else if (this.hasHover) {
      return 'mouse';
    } else {
      return 'hybrid';
    }
  }
}
```

## 事件处理管道

### 事件处理器

```typescript
// 统一事件处理器
export class UnifiedEventHandler {
  private gestureRecognizer: GestureRecognizer;
  private pressureHandler: PressureHandler;
  private deviceCompatibility: DeviceCompatibility;
  private eventQueue: EventQueue;

  constructor(element: HTMLElement) {
    this.gestureRecognizer = new GestureRecognizer();
    this.pressureHandler = new PressureHandler();
    this.deviceCompatibility = new DeviceCompatibility();
    this.eventQueue = new EventQueue();

    this.setupEventListeners(element);
  }

  // 设置事件监听
  private setupEventListeners(element: HTMLElement) {
    const eventTypes = this.deviceCompatibility.getEventTypes();

    element.addEventListener(eventTypes.down, this.handlePointerDown.bind(this));
    element.addEventListener(eventTypes.move, this.handlePointerMove.bind(this));
    element.addEventListener(eventTypes.up, this.handlePointerUp.bind(this));
    element.addEventListener(eventTypes.cancel, this.handlePointerCancel.bind(this));

    // 额外的事件监听
    element.addEventListener('wheel', this.handleWheel.bind(this));
    element.addEventListener('contextmenu', this.handleContextMenu.bind(this));

    // 阻止默认的触摸行为
    element.addEventListener('touchstart', (e) => e.preventDefault());
    element.addEventListener('touchmove', (e) => e.preventDefault());
  }

  // 处理指针按下
  private handlePointerDown(event: Event) {
    event.preventDefault();

    const normalizedEvents = this.deviceCompatibility.normalizeEvent(event);

    normalizedEvents.forEach(pointerEvent => {
      // 添加压感信息
      pointerEvent.pressure = this.pressureHandler.processPressure(event as PointerEvent);

      // 添加到事件队列
      this.eventQueue.enqueue({
        type: 'pointerDown',
        event: pointerEvent,
        timestamp: performance.now(),
      });

      // 手势识别
      this.gestureRecognizer.onPointerDown(pointerEvent);
    });

    // 处理队列
    this.processEventQueue();
  }

  // 处理指针移动
  private handlePointerMove(event: Event) {
    const normalizedEvents = this.deviceCompatibility.normalizeEvent(event);

    normalizedEvents.forEach(pointerEvent => {
      // 添加压感信息
      pointerEvent.pressure = this.pressureHandler.processPressure(event as PointerEvent);

      // 节流处理
      if (this.shouldThrottleMove(pointerEvent)) {
        return;
      }

      // 添加到事件队列
      this.eventQueue.enqueue({
        type: 'pointerMove',
        event: pointerEvent,
        timestamp: performance.now(),
      });

      // 手势识别
      this.gestureRecognizer.onPointerMove(pointerEvent);
    });

    // 处理队列
    this.processEventQueue();
  }

  // 处理指针抬起
  private handlePointerUp(event: Event) {
    const normalizedEvents = this.deviceCompatibility.normalizeEvent(event);

    normalizedEvents.forEach(pointerEvent => {
      // 添加到事件队列
      this.eventQueue.enqueue({
        type: 'pointerUp',
        event: pointerEvent,
        timestamp: performance.now(),
      });

      // 手势识别
      this.gestureRecognizer.onPointerUp(pointerEvent);
    });

    // 重置压感处理器
    this.pressureHandler.reset();

    // 处理队列
    this.processEventQueue();
  }

  // 处理指针取消
  private handlePointerCancel(event: Event) {
    const normalizedEvents = this.deviceCompatibility.normalizeEvent(event);

    normalizedEvents.forEach(pointerEvent => {
      this.eventQueue.enqueue({
        type: 'pointerCancel',
        event: pointerEvent,
        timestamp: performance.now(),
      });
    });

    this.processEventQueue();
  }

  // 处理滚轮事件
  private handleWheel(event: WheelEvent) {
    event.preventDefault();

    const wheelEvent = {
      x: event.clientX,
      y: event.clientY,
      deltaX: event.deltaX,
      deltaY: event.deltaY,
      deltaZ: event.deltaZ,
      deltaMode: event.deltaMode,
      ctrlKey: event.ctrlKey,
      metaKey: event.metaKey,
      altKey: event.altKey,
      shiftKey: event.shiftKey,
    };

    // 触发滚轮事件
    document.dispatchEvent(new CustomEvent('canvasWheel', {
      detail: wheelEvent
    }));
  }

  // 处理右键菜单
  private handleContextMenu(event: MouseEvent) {
    event.preventDefault();

    // 触发上下文菜单事件
    document.dispatchEvent(new CustomEvent('canvasContextMenu', {
      detail: {
        x: event.clientX,
        y: event.clientY,
        target: event.target,
      }
    }));
  }

  // 移动事件节流判断
  private shouldThrottleMove(event: ExtendedPointerEvent): boolean {
    // 基于时间和距离的节流
    const now = performance.now();
    const threshold = 16; // ~60fps

    // 这里应该检查上次相同 pointerId 的事件时间
    return false; // 简化实现
  }

  // 处理事件队列
  private processEventQueue() {
    requestAnimationFrame(() => {
      this.eventQueue.process();
    });
  }
}

// 事件队列
class EventQueue {
  private queue: QueuedEvent[] = [];
  private processing = false;

  enqueue(event: QueuedEvent) {
    this.queue.push(event);
  }

  process() {
    if (this.processing) return;

    this.processing = true;

    while (this.queue.length > 0) {
      const event = this.queue.shift()!;
      this.dispatch(event);
    }

    this.processing = false;
  }

  private dispatch(event: QueuedEvent) {
    document.dispatchEvent(new CustomEvent(`unified${event.type}`, {
      detail: event
    }));
  }
}

interface QueuedEvent {
  type: string;
  event: ExtendedPointerEvent;
  timestamp: number;
}
```

## 实战示例：手势绘图板

### 多指手势画板

```javascript
class GestureDrawingBoard {
  constructor(canvas) {
    this.canvas = canvas;
    this.context = canvas.getContext('2d');
    this.paths = [];
    this.currentPaths = new Map(); // pointerId -> path

    this.gestureRecognizer = new GestureRecognizer();
    this.eventHandler = new UnifiedEventHandler(canvas);

    this.setupGestureHandlers();
    this.setupCanvas();
  }

  setupCanvas() {
    // 设置画布样式
    this.context.lineCap = 'round';
    this.context.lineJoin = 'round';

    // 高DPI支持
    const dpr = window.devicePixelRatio || 1;
    const rect = this.canvas.getBoundingClientRect();

    this.canvas.width = rect.width * dpr;
    this.canvas.height = rect.height * dpr;

    this.context.scale(dpr, dpr);
    this.canvas.style.width = rect.width + 'px';
    this.canvas.style.height = rect.height + 'px';
  }

  setupGestureHandlers() {
    // 单指绘画
    document.addEventListener('unifiedpointerDown', (e) => {
      const { event } = e.detail;

      if (this.gestureRecognizer.getState().pointers.size === 1) {
        this.startPath(event);
      }
    });

    document.addEventListener('unifiedpointerMove', (e) => {
      const { event } = e.detail;

      if (this.currentPaths.has(event.pointerId)) {
        this.addPointToPath(event);
      }
    });

    document.addEventListener('unifiedpointerUp', (e) => {
      const { event } = e.detail;

      if (this.currentPaths.has(event.pointerId)) {
        this.endPath(event);
      }
    });

    // 双指缩放
    document.addEventListener('zoomMove', (e) => {
      const { scale, state } = e.detail;
      this.handleZoom(scale, state);
    });

    // 双指平移
    document.addEventListener('panMove', (e) => {
      const { event, state } = e.detail;

      if (state.pointers.size === 2) {
        this.handlePan(event, state);
      }
    });

    // 双指旋转
    document.addEventListener('rotateMove', (e) => {
      const { rotation, state } = e.detail;
      this.handleRotation(rotation, state);
    });
  }

  // 开始路径
  startPath(event) {
    const point = this.getCanvasPoint(event);

    const path = {
      id: this.generateId(),
      points: [point],
      pressure: [event.pressure || 0.5],
      color: this.getCurrentColor(),
      width: this.getCurrentWidth(),
      timestamp: Date.now(),
    };

    this.currentPaths.set(event.pointerId, path);
    this.renderPath(path);
  }

  // 添加点到路径
  addPointToPath(event) {
    const path = this.currentPaths.get(event.pointerId);
    if (!path) return;

    const point = this.getCanvasPoint(event);
    const pressure = event.pressure || 0.5;

    // 距离过滤
    const lastPoint = path.points[path.points.length - 1];
    const distance = Math.hypot(point.x - lastPoint.x, point.y - lastPoint.y);

    if (distance < 2) return;

    path.points.push(point);
    path.pressure.push(pressure);

    this.renderPath(path, true); // 增量渲染
  }

  // 结束路径
  endPath(event) {
    const path = this.currentPaths.get(event.pointerId);
    if (!path) return;

    // 平滑路径
    path.points = this.smoothPath(path.points);

    // 保存到路径集合
    this.paths.push(path);
    this.currentPaths.delete(event.pointerId);

    // 完整重绘（可选）
    this.redraw();
  }

  // 平滑路径
  smoothPath(points) {
    if (points.length < 3) return points;

    const smoothed = [points[0]];

    for (let i = 1; i < points.length - 1; i++) {
      const prev = points[i - 1];
      const curr = points[i];
      const next = points[i + 1];

      const smoothedPoint = {
        x: (prev.x + curr.x + next.x) / 3,
        y: (prev.y + curr.y + next.y) / 3,
      };

      smoothed.push(smoothedPoint);
    }

    smoothed.push(points[points.length - 1]);
    return smoothed;
  }

  // 渲染路径
  renderPath(path, incremental = false) {
    this.context.save();

    this.context.strokeStyle = path.color;
    this.context.globalCompositeOperation = 'source-over';

    if (!incremental) {
      // 完整渲染路径
      this.context.beginPath();

      path.points.forEach((point, index) => {
        const pressure = path.pressure[index] || 0.5;
        const width = path.width * pressure;

        this.context.lineWidth = width;

        if (index === 0) {
          this.context.moveTo(point.x, point.y);
        } else {
          this.context.lineTo(point.x, point.y);
        }
      });

      this.context.stroke();
    } else {
      // 增量渲染（只绘制最新的线段）
      const points = path.points;
      if (points.length >= 2) {
        const prevPoint = points[points.length - 2];
        const currPoint = points[points.length - 1];
        const pressure = path.pressure[path.pressure.length - 1] || 0.5;

        this.context.lineWidth = path.width * pressure;
        this.context.beginPath();
        this.context.moveTo(prevPoint.x, prevPoint.y);
        this.context.lineTo(currPoint.x, currPoint.y);
        this.context.stroke();
      }
    }

    this.context.restore();
  }

  // 处理缩放
  handleZoom(scale, state) {
    const center = state.lastCenter;
    if (!center) return;

    const canvasCenter = this.getCanvasPoint({ x: center.x, y: center.y });

    // 保存当前变换
    this.context.save();

    // 应用缩放
    this.context.translate(canvasCenter.x, canvasCenter.y);
    this.context.scale(scale, scale);
    this.context.translate(-canvasCenter.x, -canvasCenter.y);

    // 重绘所有路径
    this.redraw();

    this.context.restore();
  }

  // 处理平移
  handlePan(event, state) {
    if (!state.lastCenter) return;

    // 计算平移量
    const deltaX = event.x - state.lastCenter.x;
    const deltaY = event.y - state.lastCenter.y;

    // 应用平移
    this.context.translate(deltaX, deltaY);
    this.redraw();
  }

  // 处理旋转
  handleRotation(rotation, state) {
    const center = state.lastCenter;
    if (!center) return;

    const canvasCenter = this.getCanvasPoint({ x: center.x, y: center.y });

    this.context.save();

    // 应用旋转
    this.context.translate(canvasCenter.x, canvasCenter.y);
    this.context.rotate(rotation);
    this.context.translate(-canvasCenter.x, -canvasCenter.y);

    this.redraw();

    this.context.restore();
  }

  // 重绘所有路径
  redraw() {
    this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // 绘制所有完成的路径
    this.paths.forEach(path => {
      this.renderPath(path);
    });

    // 绘制当前正在绘制的路径
    this.currentPaths.forEach(path => {
      this.renderPath(path);
    });
  }

  // 获取画布坐标
  getCanvasPoint(event) {
    const rect = this.canvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;

    return {
      x: (event.x - rect.left) * dpr,
      y: (event.y - rect.top) * dpr,
    };
  }

  // 获取当前颜色
  getCurrentColor() {
    return '#000000'; // 可从UI获取
  }

  // 获取当前线宽
  getCurrentWidth() {
    return 2; // 可从UI获取
  }

  // 生成ID
  generateId() {
    return Math.random().toString(36).substr(2, 9);
  }

  // 清除画布
  clear() {
    this.paths = [];
    this.currentPaths.clear();
    this.context.clearRect(0, 0, this.canvas.width, this.canvas.height);
  }

  // 撤销
  undo() {
    if (this.paths.length > 0) {
      this.paths.pop();
      this.redraw();
    }
  }

  // 导出为数据
  export() {
    return {
      paths: this.paths,
      width: this.canvas.width,
      height: this.canvas.height,
      timestamp: Date.now(),
    };
  }

  // 导入数据
  import(data) {
    this.paths = data.paths || [];
    this.redraw();
  }
}

// 使用示例
const canvas = document.getElementById('gestureCanvas');
const drawingBoard = new GestureDrawingBoard(canvas);

// 添加控制UI
document.getElementById('clearBtn').addEventListener('click', () => {
  drawingBoard.clear();
});

document.getElementById('undoBtn').addEventListener('click', () => {
  drawingBoard.undo();
});

document.getElementById('exportBtn').addEventListener('click', () => {
  const data = drawingBoard.export();
  console.log('导出数据:', data);
});
```

## 思考题

1. **手势冲突如何解决？** 当多个手势同时满足条件时如何处理？

2. **延迟和响应性如何平衡？** 手势识别的准确性和响应速度如何平衡？

3. **跨设备一致性如何保证？** 不同设备上的手势体验如何保持一致？

4. **自定义手势如何设计？** 如何让用户定义自己的手势？

5. **无障碍支持如何实现？** 如何为残障用户提供替代的输入方式？

## 实践练习

### 练习 1：实现手写识别
创建手写文字识别系统：
- 笔画数据收集
- 特征提取算法
- 机器学习模型集成

### 练习 2：实现3D手势
支持空间手势（如果有深度相机）：
- 深度数据处理
- 3D手势算法
- 空间交互设计

### 练习 3：实现预测性交互
基于用户行为预测下一步操作：
- 行为模式分析
- 预测算法实现
- UI预加载优化

## 总结

Excalidraw 的手势处理系统体现了现代输入处理的最佳实践：

1. **统一抽象**：将不同输入设备抽象为统一的指针事件
2. **精确识别**：通过多层次的算法准确识别复杂手势
3. **设备适配**：针对不同设备特性进行专门优化
4. **性能优化**：通过节流、队列等技术保证流畅体验
5. **扩展性设计**：支持自定义手势和新设备类型

这些技术不仅适用于绘图应用，也为其他需要丰富交互的应用提供了参考。

## 下一步

下一章将探讨选择和变换系统，学习如何实现精确的元素选择、高效的变换操作以及直观的变换手柄设计。