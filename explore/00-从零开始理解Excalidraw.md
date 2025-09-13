# 从零开始理解 Excalidraw - 画板的本质

这份文档将从最基础的概念开始，一步步构建出一个完整的画板系统，让你真正理解 Excalidraw 的实现原理。

## 第一章：画板的本质是什么？

### 1.1 最原始的画板 - 一个 Canvas

让我们从最简单的开始：在浏览器上画一条线。

```html
<!DOCTYPE html>
<html>
<body>
  <canvas id="canvas" width="800" height="600"></canvas>
  <script>
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    
    // 画一条线
    ctx.beginPath();
    ctx.moveTo(100, 100);
    ctx.lineTo(200, 200);
    ctx.stroke();
  </script>
</body>
</html>
```

这就是一切的开始。Canvas 提供了一个 2D 绘图上下文，我们可以在上面画任何东西。

### 1.2 让用户能画 - 监听鼠标事件

```javascript
let isDrawing = false;
let lastX = 0;
let lastY = 0;

canvas.addEventListener('mousedown', (e) => {
  isDrawing = true;
  lastX = e.offsetX;
  lastY = e.offsetY;
});

canvas.addEventListener('mousemove', (e) => {
  if (!isDrawing) return;
  
  ctx.beginPath();
  ctx.moveTo(lastX, lastY);
  ctx.lineTo(e.offsetX, e.offsetY);
  ctx.stroke();
  
  lastX = e.offsetX;
  lastY = e.offsetY;
});

canvas.addEventListener('mouseup', () => {
  isDrawing = false;
});
```

现在用户可以自由绘制了！但这只是涂鸦，不是真正的画板。

### 1.3 核心问题：如何存储和重绘？

Canvas 是"即时模式"渲染，一旦画上去就变成像素了，无法修改。要实现真正的画板，我们需要：

1. **数据结构**：存储所有图形的信息
2. **重绘机制**：根据数据重新渲染整个画布

```javascript
// 这是关键转变 - 从直接绘制到数据驱动
const elements = []; // 存储所有元素

// 添加元素
function addElement(element) {
  elements.push(element);
  redraw(); // 重新绘制整个画布
}

// 重绘函数
function redraw() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  elements.forEach(element => {
    drawElement(element);
  });
}
```

## 第二章：构建数据模型

### 2.1 定义元素结构

每个图形元素需要什么信息？

```typescript
interface Element {
  id: string;        // 唯一标识
  type: string;      // 类型：rectangle, ellipse, line, etc
  x: number;         // 位置
  y: number;
  width: number;     // 尺寸
  height: number;
  strokeColor: string;   // 样式
  backgroundColor: string;
  strokeWidth: number;
}
```

### 2.2 不同类型的元素

```typescript
// 矩形
interface RectangleElement extends Element {
  type: 'rectangle';
}

// 线条 - 需要点集合
interface LineElement extends Element {
  type: 'line';
  points: Array<{x: number, y: number}>;
}

// 文本 - 需要文字属性
interface TextElement extends Element {
  type: 'text';
  text: string;
  fontSize: number;
  fontFamily: string;
}
```

### 2.3 渲染不同类型的元素

```javascript
function drawElement(element) {
  switch(element.type) {
    case 'rectangle':
      ctx.strokeRect(element.x, element.y, element.width, element.height);
      break;
      
    case 'ellipse':
      ctx.beginPath();
      ctx.ellipse(
        element.x + element.width/2, 
        element.y + element.height/2,
        element.width/2, 
        element.height/2, 
        0, 0, 2 * Math.PI
      );
      ctx.stroke();
      break;
      
    case 'line':
      ctx.beginPath();
      element.points.forEach((point, index) => {
        if (index === 0) {
          ctx.moveTo(point.x, point.y);
        } else {
          ctx.lineTo(point.x, point.y);
        }
      });
      ctx.stroke();
      break;
  }
}
```

## 第三章：交互系统

### 3.1 工具状态机

用户在不同工具之间切换，每个工具有不同的行为：

```javascript
class ToolStateMachine {
  constructor() {
    this.currentTool = 'selection';
    this.tools = {
      selection: new SelectionTool(),
      rectangle: new RectangleTool(),
      line: new LineTool(),
      text: new TextTool()
    };
  }
  
  setTool(toolName) {
    this.currentTool = toolName;
  }
  
  handleMouseDown(e) {
    this.tools[this.currentTool].onMouseDown(e);
  }
  
  handleMouseMove(e) {
    this.tools[this.currentTool].onMouseMove(e);
  }
  
  handleMouseUp(e) {
    this.tools[this.currentTool].onMouseUp(e);
  }
}
```

### 3.2 选择工具实现

```javascript
class SelectionTool {
  constructor() {
    this.selectedElement = null;
    this.isDragging = false;
    this.dragOffset = {x: 0, y: 0};
  }
  
  onMouseDown(e) {
    // 点击检测 - 找到被点击的元素
    this.selectedElement = findElementAtPosition(e.x, e.y);
    if (this.selectedElement) {
      this.isDragging = true;
      this.dragOffset = {
        x: e.x - this.selectedElement.x,
        y: e.y - this.selectedElement.y
      };
    }
  }
  
  onMouseMove(e) {
    if (this.isDragging && this.selectedElement) {
      // 更新元素位置
      this.selectedElement.x = e.x - this.dragOffset.x;
      this.selectedElement.y = e.y - this.dragOffset.y;
      redraw();
    }
  }
  
  onMouseUp(e) {
    this.isDragging = false;
  }
}
```

### 3.3 绘制工具实现

```javascript
class RectangleTool {
  constructor() {
    this.isDrawing = false;
    this.startPoint = null;
    this.currentElement = null;
  }
  
  onMouseDown(e) {
    this.isDrawing = true;
    this.startPoint = {x: e.x, y: e.y};
    
    // 创建新元素
    this.currentElement = {
      id: generateId(),
      type: 'rectangle',
      x: e.x,
      y: e.y,
      width: 0,
      height: 0,
      strokeColor: '#000000',
      backgroundColor: 'transparent'
    };
    
    elements.push(this.currentElement);
  }
  
  onMouseMove(e) {
    if (!this.isDrawing) return;
    
    // 更新元素尺寸
    this.currentElement.width = e.x - this.startPoint.x;
    this.currentElement.height = e.y - this.startPoint.y;
    
    // 处理负值（向左上拖动）
    if (this.currentElement.width < 0) {
      this.currentElement.x = e.x;
      this.currentElement.width = Math.abs(this.currentElement.width);
    }
    if (this.currentElement.height < 0) {
      this.currentElement.y = e.y;
      this.currentElement.height = Math.abs(this.currentElement.height);
    }
    
    redraw();
  }
  
  onMouseUp(e) {
    this.isDrawing = false;
    this.currentElement = null;
  }
}
```

## 第四章：性能优化

### 4.1 问题：每次改动都重绘所有元素

当元素很多时，性能会很差。

### 4.2 解决方案1：脏矩形（Dirty Rectangle）

只重绘改变的区域：

```javascript
class DirtyRectManager {
  constructor() {
    this.dirtyRects = [];
  }
  
  addDirtyRect(x, y, width, height) {
    this.dirtyRects.push({x, y, width, height});
  }
  
  redraw() {
    // 合并重叠的脏矩形
    const mergedRect = this.mergeDirtyRects();
    
    // 只清除和重绘脏矩形区域
    ctx.clearRect(mergedRect.x, mergedRect.y, mergedRect.width, mergedRect.height);
    
    // 只重绘与脏矩形相交的元素
    const elementsToRedraw = elements.filter(element => 
      isIntersecting(element, mergedRect)
    );
    
    elementsToRedraw.forEach(element => drawElement(element));
    
    this.dirtyRects = [];
  }
}
```

### 4.3 解决方案2：分层渲染

Excalidraw 的关键优化 - 多层 Canvas：

```javascript
class LayeredRenderer {
  constructor() {
    // 静态层 - 不经常变化的元素
    this.staticCanvas = document.createElement('canvas');
    this.staticCtx = this.staticCanvas.getContext('2d');
    this.staticDirty = true;
    
    // 交互层 - 选择框、拖动预览等
    this.interactiveCanvas = document.createElement('canvas');
    this.interactiveCtx = this.interactiveCanvas.getContext('2d');
  }
  
  renderStaticLayer() {
    if (!this.staticDirty) return;
    
    this.staticCtx.clearRect(0, 0, width, height);
    elements.forEach(element => {
      if (!element.isSelected && !element.isBeingDragged) {
        drawElement(this.staticCtx, element);
      }
    });
    
    this.staticDirty = false;
  }
  
  renderInteractiveLayer() {
    this.interactiveCtx.clearRect(0, 0, width, height);
    
    // 只绘制正在交互的元素
    elements.forEach(element => {
      if (element.isSelected || element.isBeingDragged) {
        drawElement(this.interactiveCtx, element);
      }
    });
    
    // 绘制选择框等UI元素
    if (selectionBox) {
      drawSelectionBox(this.interactiveCtx, selectionBox);
    }
  }
}
```

## 第五章：手绘风格 - RoughJS 集成

### 5.1 为什么要手绘风格？

手绘风格让图形看起来更友好、更有创意，这是 Excalidraw 的标志性特征。

### 5.2 RoughJS 的原理

RoughJS 通过添加随机扰动来模拟手绘效果：

```javascript
// 简化的手绘线条实现
function drawRoughLine(ctx, x1, y1, x2, y2, roughness = 1) {
  const offset = roughness * 2;
  
  // 画两条略有偏移的线来模拟手绘
  for (let i = 0; i < 2; i++) {
    ctx.beginPath();
    
    // 起点添加随机偏移
    ctx.moveTo(
      x1 + (Math.random() - 0.5) * offset,
      y1 + (Math.random() - 0.5) * offset
    );
    
    // 在线条中间添加一些随机点
    const steps = 3;
    for (let j = 1; j < steps; j++) {
      const t = j / steps;
      const x = x1 + (x2 - x1) * t;
      const y = y1 + (y2 - y1) * t;
      ctx.lineTo(
        x + (Math.random() - 0.5) * offset,
        y + (Math.random() - 0.5) * offset
      );
    }
    
    // 终点
    ctx.lineTo(
      x2 + (Math.random() - 0.5) * offset,
      y2 + (Math.random() - 0.5) * offset
    );
    
    ctx.stroke();
  }
}
```

### 5.3 使用种子保证一致性

```javascript
class SeededRandom {
  constructor(seed) {
    this.seed = seed;
  }
  
  // 简单的伪随机数生成器
  next() {
    this.seed = (this.seed * 9301 + 49297) % 233280;
    return this.seed / 233280;
  }
}

// 每个元素都有固定的种子
function drawRoughElement(element) {
  const random = new SeededRandom(element.seed);
  // 使用 random.next() 而不是 Math.random()
  // 这样每次重绘都是一样的效果
}
```

## 第六章：状态管理

### 6.1 应用状态 vs 元素状态

```typescript
// 元素状态 - 持久化的数据
interface ElementState {
  elements: Element[];
}

// 应用状态 - 临时的UI状态
interface AppState {
  selectedElementIds: Set<string>;
  currentTool: string;
  zoom: number;
  scrollX: number;
  scrollY: number;
  cursorPosition: {x: number, y: number};
  isDragging: boolean;
  isResizing: boolean;
}
```

### 6.2 撤销/重做系统

```javascript
class History {
  constructor() {
    this.stack = [];
    this.pointer = -1;
  }
  
  push(snapshot) {
    // 删除当前指针之后的历史
    this.stack = this.stack.slice(0, this.pointer + 1);
    // 添加新快照
    this.stack.push(snapshot);
    this.pointer++;
    
    // 限制历史大小
    if (this.stack.length > 100) {
      this.stack.shift();
      this.pointer--;
    }
  }
  
  undo() {
    if (this.pointer > 0) {
      this.pointer--;
      return this.stack[this.pointer];
    }
    return null;
  }
  
  redo() {
    if (this.pointer < this.stack.length - 1) {
      this.pointer++;
      return this.stack[this.pointer];
    }
    return null;
  }
}

// 使用
const history = new History();

function commitToHistory() {
  const snapshot = {
    elements: JSON.parse(JSON.stringify(elements)),
    appState: {...appState}
  };
  history.push(snapshot);
}
```

## 第七章：协作功能

### 7.1 协作的核心：CRDT

CRDT (Conflict-free Replicated Data Type) 允许多人同时编辑而不冲突。

### 7.2 简化的协作实现

```javascript
// 每个操作都是一个事件
class CollaborationEngine {
  constructor() {
    this.localVersion = 0;
    this.remoteVersions = new Map();
  }
  
  // 本地操作
  applyLocalOperation(operation) {
    this.localVersion++;
    operation.version = this.localVersion;
    operation.clientId = this.clientId;
    
    // 应用到本地
    this.applyOperation(operation);
    
    // 广播给其他客户端
    this.broadcast(operation);
  }
  
  // 接收远程操作
  receiveRemoteOperation(operation) {
    const lastVersion = this.remoteVersions.get(operation.clientId) || 0;
    
    // 检查是否按顺序
    if (operation.version === lastVersion + 1) {
      this.applyOperation(operation);
      this.remoteVersions.set(operation.clientId, operation.version);
    } else {
      // 处理乱序或冲突
      this.resolveConflict(operation);
    }
  }
  
  // 冲突解决 - 使用 clientId 作为优先级
  resolveConflict(operation) {
    // 简单策略：clientId 较小的优先
    if (operation.clientId < this.clientId) {
      // 撤销本地操作，应用远程操作，重新应用本地操作
      this.rebase(operation);
    }
  }
}
```

## 第八章：导出和序列化

### 8.1 数据格式设计

```typescript
interface ExcalidrawData {
  type: "excalidraw";
  version: 2;
  source: string;
  elements: Element[];
  appState: Partial<AppState>;
  files: Record<string, BinaryFile>; // 图片等二进制文件
}
```

### 8.2 导出为图片

```javascript
function exportToImage(format = 'png') {
  // 创建离屏 Canvas
  const exportCanvas = document.createElement('canvas');
  const exportCtx = exportCanvas.getContext('2d');
  
  // 计算边界框
  const bounds = calculateBounds(elements);
  exportCanvas.width = bounds.width;
  exportCanvas.height = bounds.height;
  
  // 渲染所有元素
  exportCtx.translate(-bounds.x, -bounds.y);
  elements.forEach(element => {
    drawElement(exportCtx, element);
  });
  
  // 导出
  if (format === 'png') {
    return exportCanvas.toDataURL('image/png');
  } else if (format === 'svg') {
    return convertToSVG(elements, bounds);
  }
}
```

## 总结：从零到一的关键步骤

1. **Canvas 基础** - 理解 Canvas API 的即时模式渲染
2. **数据驱动** - 从直接绘制转向数据结构 + 重绘
3. **元素系统** - 设计通用的元素数据结构
4. **工具系统** - 状态机模式处理不同工具
5. **性能优化** - 分层渲染、脏矩形等技术
6. **手绘风格** - 随机扰动 + 种子保证一致性
7. **状态管理** - 分离持久状态和临时状态
8. **协作支持** - CRDT 或操作转换实现冲突解决

这就是 Excalidraw 的核心原理。从一个简单的 Canvas 开始，逐步添加功能，最终构建出一个完整的协作画板应用。

## 下一步

现在你已经理解了原理，可以：

1. 查看 [最小核心实现](./00-minimal-core.md) - 一个可运行的最小画板
2. 深入 [架构详解](./01-架构详解.md) - 了解实际的代码组织
3. 学习 [元素系统](./02-元素系统.md) - 掌握 Excalidraw 的数据模型
