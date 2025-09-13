# Canvas 基础与 Web 图形学
## 从零开始构建画板的技术基础

---

## 🎯 学习目标

通过本章学习，你将完全掌握：
- Canvas API 的所有核心概念和用法
- 浏览器图形渲染的底层机制
- 高 DPI 屏幕适配的完整方案
- Canvas 与其他图形技术的对比分析
- 性能优化的基础技巧

---

## 1. Canvas 的本质理解

### 1.1 什么是 Canvas？

Canvas 是 HTML5 提供的一个绘图元素，它提供了一个可以通过脚本绘制图形的画布。

```html
<!-- 最基础的 Canvas 元素 -->
<canvas id="myCanvas" width="800" height="600">
  Your browser does not support the canvas element.
</canvas>
```

**关键理解**：Canvas 本身只是一个容器，真正的绘图能力来自于它的**绘图上下文（Context）**。

### 1.2 即时模式 vs 保留模式

这是理解 Canvas 最重要的概念：

#### 即时模式（Canvas）
```javascript
const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');

// 画一个矩形
ctx.fillRect(10, 10, 100, 100);

// 矩形已经变成像素了！
// 你无法再"访问"这个矩形对象
// 想要修改？只能重新绘制整个画布
```

**特点**：
- 绘制后立即转为像素
- 没有对象概念，只有像素
- 性能高，内存占用低
- 需要手动管理"对象"状态

#### 保留模式（SVG/DOM）
```javascript
// SVG 方式
const rect = document.createElementNS('http://www.w3.org/2000/svg', 'rect');
rect.setAttribute('x', 10);
rect.setAttribute('y', 10);
rect.setAttribute('width', 100);
rect.setAttribute('height', 100);

// 矩形是一个真实的 DOM 对象！
// 可以随时修改属性
rect.setAttribute('x', 20); // 矩形移动了！
```

**特点**：
- 每个图形都是对象
- 可以直接操作对象
- 浏览器自动处理重绘
- 内存占用高，性能相对较低

### 1.3 为什么 Excalidraw 选择 Canvas？

**性能考量**：
- 大量图形元素时性能更好
- 可以精确控制渲染时机
- 支持复杂的图形操作

**灵活性考量**：
- 可以实现任意复杂的视觉效果
- 精确控制每一个像素
- 不受 DOM 限制

---

## 2. Canvas API 完全指南

### 2.1 获取绘图上下文

```javascript
const canvas = document.getElementById('myCanvas');

// 2D 上下文
const ctx = canvas.getContext('2d');

// WebGL 上下文（3D）
const gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');

// 检查支持性
if (!ctx) {
    console.error('Canvas 2D context not supported');
}
```

### 2.2 坐标系统详解

Canvas 使用左上角为原点的坐标系统：

```javascript
// Canvas 坐标系统
//  (0,0) ------> X 轴
//    |
//    |
//    |
//    ↓
//   Y 轴

// 基本绘制
ctx.fillRect(0, 0, 50, 50);      // 左上角 50x50 的矩形
ctx.fillRect(100, 100, 50, 50);  // (100,100) 位置的矩形
```

### 2.3 基础绘制方法

#### 2.3.1 矩形绘制
```javascript
// 填充矩形
ctx.fillStyle = 'blue';
ctx.fillRect(x, y, width, height);

// 描边矩形
ctx.strokeStyle = 'red';
ctx.lineWidth = 2;
ctx.strokeRect(x, y, width, height);

// 清除矩形区域
ctx.clearRect(x, y, width, height);
```

#### 2.3.2 路径绘制系统
```javascript
// Canvas 的路径系统是核心
ctx.beginPath();        // 开始新路径
ctx.moveTo(50, 50);     // 移动到起点
ctx.lineTo(150, 50);    // 画线到终点
ctx.lineTo(100, 150);   // 继续画线
ctx.closePath();        // 闭合路径
ctx.fill();            // 填充路径
// 或者
ctx.stroke();          // 描边路径
```

#### 2.3.3 圆弧和圆形
```javascript
// 绘制圆形
ctx.beginPath();
ctx.arc(100, 100, 50, 0, 2 * Math.PI);  // 圆心(100,100)，半径50
ctx.fill();

// 绘制扇形
ctx.beginPath();
ctx.arc(100, 100, 50, 0, Math.PI);      // 半圆
ctx.fill();

// 椭圆（现代浏览器）
ctx.beginPath();
ctx.ellipse(100, 100, 50, 30, 0, 0, 2 * Math.PI);
ctx.fill();
```

#### 2.3.4 贝塞尔曲线
```javascript
// 二次贝塞尔曲线
ctx.beginPath();
ctx.moveTo(50, 50);
ctx.quadraticCurveTo(100, 25, 150, 50);  // 控制点(100,25)，终点(150,50)
ctx.stroke();

// 三次贝塞尔曲线
ctx.beginPath();
ctx.moveTo(50, 50);
ctx.bezierCurveTo(70, 25, 130, 25, 150, 50);  // 两个控制点
ctx.stroke();
```

### 2.4 样式系统详解

#### 2.4.1 颜色和填充
```javascript
// 纯色填充
ctx.fillStyle = 'red';
ctx.fillStyle = '#ff0000';
ctx.fillStyle = 'rgb(255, 0, 0)';
ctx.fillStyle = 'rgba(255, 0, 0, 0.5)';  // 半透明

// 渐变填充
const gradient = ctx.createLinearGradient(0, 0, 200, 0);
gradient.addColorStop(0, 'red');
gradient.addColorStop(0.5, 'yellow');
gradient.addColorStop(1, 'blue');
ctx.fillStyle = gradient;

// 图案填充
const img = new Image();
img.onload = function() {
    const pattern = ctx.createPattern(img, 'repeat');
    ctx.fillStyle = pattern;
};
img.src = 'pattern.png';
```

#### 2.4.2 线条样式
```javascript
// 线条宽度
ctx.lineWidth = 5;

// 线条端点样式
ctx.lineCap = 'butt';    // 默认，方形端点
ctx.lineCap = 'round';   // 圆形端点
ctx.lineCap = 'square';  // 方形端点，但延伸半个线宽

// 线条连接样式
ctx.lineJoin = 'miter';  // 默认，尖角连接
ctx.lineJoin = 'round';  // 圆角连接
ctx.lineJoin = 'bevel';  // 斜角连接

// 虚线
ctx.setLineDash([5, 5]);      // 5像素实线，5像素空白
ctx.setLineDash([10, 5, 2]);  // 复杂虚线模式
ctx.lineDashOffset = 2;       // 虚线偏移
```

#### 2.4.3 阴影效果
```javascript
ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
ctx.shadowOffsetX = 5;
ctx.shadowOffsetY = 5;
ctx.shadowBlur = 10;

ctx.fillRect(50, 50, 100, 100);  // 带阴影的矩形
```

### 2.5 变换系统

这是 Canvas 最强大的功能之一：

#### 2.5.1 基础变换
```javascript
// 平移
ctx.translate(100, 100);  // 将坐标系移动到(100,100)

// 旋转
ctx.rotate(Math.PI / 4);  // 旋转45度（注意：弧度制）

// 缩放
ctx.scale(2, 1.5);        // X方向放大2倍，Y方向放大1.5倍

// 应用变换后绘制
ctx.fillRect(0, 0, 50, 50);  // 这个矩形会受到上述变换影响
```

#### 2.5.2 变换矩阵
```javascript
// 直接设置变换矩阵
ctx.setTransform(a, b, c, d, e, f);
// a, d: 缩放
// b, c: 倾斜
// e, f: 平移

// 示例：倾斜效果
ctx.setTransform(1, 0.5, 0, 1, 0, 0);  // X方向倾斜
```

#### 2.5.3 状态保存与恢复
```javascript
// 保存当前状态
ctx.save();

// 做一些变换和绘制
ctx.translate(100, 100);
ctx.rotate(Math.PI / 4);
ctx.fillRect(0, 0, 50, 50);

// 恢复到保存的状态
ctx.restore();

// 现在又回到了原始状态
ctx.fillRect(0, 0, 50, 50);  // 这个矩形不会受到前面变换的影响
```

### 2.6 文本绘制

#### 2.6.1 基础文本
```javascript
// 设置字体
ctx.font = '30px Arial';
ctx.fillStyle = 'black';
ctx.textAlign = 'left';      // left, right, center, start, end
ctx.textBaseline = 'top';    // top, middle, bottom, alphabetic, hanging

// 绘制文本
ctx.fillText('Hello World', 50, 50);
ctx.strokeText('Hello World', 50, 100);  // 描边文本
```

#### 2.6.2 文本测量
```javascript
const text = 'Hello World';
const metrics = ctx.measureText(text);

console.log('文本宽度:', metrics.width);
console.log('文本高度:', metrics.actualBoundingBoxAscent + metrics.actualBoundingBoxDescent);
```

#### 2.6.3 多行文本处理
```javascript
function drawMultilineText(ctx, text, x, y, lineHeight) {
    const lines = text.split('\n');
    lines.forEach((line, index) => {
        ctx.fillText(line, x, y + index * lineHeight);
    });
}

drawMultilineText(ctx, 'Line 1\nLine 2\nLine 3', 50, 50, 30);
```

---

## 3. 高 DPI 屏幕适配

这是现代 Web 应用必须解决的问题：

### 3.1 问题分析

```javascript
// 在高 DPI 屏幕上，这样的代码会导致模糊
const canvas = document.getElementById('myCanvas');
canvas.width = 800;
canvas.height = 600;
canvas.style.width = '800px';
canvas.style.height = '600px';
```

**问题原因**：
- CSS 像素 ≠ 设备像素
- Retina 屏幕：1个 CSS 像素 = 4个设备像素
- Canvas 按设备像素绘制，但按 CSS 像素显示

### 3.2 完整解决方案

```javascript
function setupHighDPICanvas(canvas) {
    // 获取设备像素比
    const dpr = window.devicePixelRatio || 1;
    
    // 获取 Canvas 的 CSS 尺寸
    const rect = canvas.getBoundingClientRect();
    const cssWidth = rect.width;
    const cssHeight = rect.height;
    
    // 设置 Canvas 的实际尺寸（设备像素）
    canvas.width = cssWidth * dpr;
    canvas.height = cssHeight * dpr;
    
    // 设置 Canvas 的显示尺寸（CSS 像素）
    canvas.style.width = cssWidth + 'px';
    canvas.style.height = cssHeight + 'px';
    
    // 缩放绘图上下文
    const ctx = canvas.getContext('2d');
    ctx.scale(dpr, dpr);
    
    return ctx;
}

// 使用
const canvas = document.getElementById('myCanvas');
const ctx = setupHighDPICanvas(canvas);

// 现在绘制的内容在高 DPI 屏幕上会很清晰
ctx.fillRect(0, 0, 100, 100);
```

### 3.3 响应式 Canvas

```javascript
class ResponsiveCanvas {
    constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = this.canvas.getContext('2d');
        this.setupCanvas();
        
        // 监听窗口大小变化
        window.addEventListener('resize', () => this.handleResize());
    }
    
    setupCanvas() {
        const container = this.canvas.parentElement;
        const rect = container.getBoundingClientRect();
        
        this.resize(rect.width, rect.height);
    }
    
    resize(width, height) {
        const dpr = window.devicePixelRatio || 1;
        
        // 设置实际尺寸
        this.canvas.width = width * dpr;
        this.canvas.height = height * dpr;
        
        // 设置显示尺寸
        this.canvas.style.width = width + 'px';
        this.canvas.style.height = height + 'px';
        
        // 缩放上下文
        this.ctx.scale(dpr, dpr);
        
        // 触发重绘事件
        this.onResize?.(width, height);
    }
    
    handleResize() {
        const container = this.canvas.parentElement;
        const rect = container.getBoundingClientRect();
        this.resize(rect.width, rect.height);
    }
}
```

---

## 4. Canvas vs 其他图形技术

### 4.1 Canvas vs SVG

| 特性 | Canvas | SVG |
|------|--------|-----|
| 渲染模式 | 即时模式（像素） | 保留模式（矢量） |
| 性能 | 大量元素时更好 | 少量元素时更好 |
| 交互性 | 需要手动实现 | 原生 DOM 事件 |
| 可访问性 | 较差 | 很好 |
| 动画 | 需要 JS | CSS + JS |
| 文件大小 | 固定（基于分辨率） | 基于复杂度 |

**选择建议**：
- **Canvas**：游戏、数据可视化、图像编辑器
- **SVG**：图标、简单图形、需要缩放的图形

### 4.2 Canvas vs WebGL

| 特性 | Canvas 2D | WebGL |
|------|-----------|-------|
| 学习曲线 | 简单 | 复杂 |
| 性能 | CPU 渲染 | GPU 渲染 |
| 3D 支持 | 无 | 原生支持 |
| 复杂效果 | 有限 | 无限可能 |

### 4.3 Canvas vs CSS

对于某些效果，CSS 可能是更好的选择：

```css
/* CSS 实现的一些效果可能比 Canvas 更高效 */
.rounded-rect {
    width: 100px;
    height: 100px;
    background: blue;
    border-radius: 10px;
    transform: rotate(45deg);
}
```

---

## 5. 性能基础

### 5.1 渲染性能优化

```javascript
// ❌ 低效：每次都重绘整个画布
function animateCircle() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);  // 清空整个画布
    ctx.fillRect(x, y, 50, 50);  // 重绘
    requestAnimationFrame(animateCircle);
}

// ✅ 高效：只清理和重绘变化的区域
function animateCircleOptimized() {
    // 清理旧位置
    ctx.clearRect(oldX, oldY, 50, 50);
    
    // 绘制新位置
    ctx.fillRect(x, y, 50, 50);
    
    oldX = x;
    oldY = y;
    
    requestAnimationFrame(animateCircleOptimized);
}
```

### 5.2 避免频繁的状态改变

```javascript
// ❌ 低效：频繁改变状态
elements.forEach(element => {
    ctx.fillStyle = element.color;
    ctx.fillRect(element.x, element.y, element.width, element.height);
});

// ✅ 高效：按颜色分组绘制
const elementsByColor = groupBy(elements, 'color');
Object.entries(elementsByColor).forEach(([color, elements]) => {
    ctx.fillStyle = color;  // 只设置一次
    elements.forEach(element => {
        ctx.fillRect(element.x, element.y, element.width, element.height);
    });
});
```

### 5.3 离屏 Canvas

```javascript
// 预渲染复杂图形到离屏 Canvas
function createComplexShape() {
    const offscreenCanvas = document.createElement('canvas');
    offscreenCanvas.width = 100;
    offscreenCanvas.height = 100;
    const offCtx = offscreenCanvas.getContext('2d');
    
    // 绘制复杂图形
    offCtx.beginPath();
    for (let i = 0; i < 100; i++) {
        // 复杂的绘制逻辑
        offCtx.lineTo(Math.random() * 100, Math.random() * 100);
    }
    offCtx.stroke();
    
    return offscreenCanvas;
}

// 使用离屏 Canvas
const complexShape = createComplexShape();

// 快速绘制到主 Canvas
ctx.drawImage(complexShape, x, y);
```

---

## 6. 实战：构建基础绘图板

让我们构建一个功能完整的基础绘图板：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>基础绘图板</title>
    <style>
        body {
            margin: 0;
            padding: 20px;
            font-family: Arial, sans-serif;
        }
        
        #canvas {
            border: 2px solid #ccc;
            cursor: crosshair;
        }
        
        .toolbar {
            margin-bottom: 10px;
        }
        
        .toolbar button {
            margin-right: 10px;
            padding: 8px 16px;
        }
        
        .toolbar button.active {
            background: #007bff;
            color: white;
        }
        
        .color-picker {
            margin-left: 20px;
        }
    </style>
</head>
<body>
    <div class="toolbar">
        <button id="pen" class="active">画笔</button>
        <button id="eraser">橡皮擦</button>
        <button id="clear">清空</button>
        
        <span class="color-picker">
            颜色: <input type="color" id="colorPicker" value="#000000">
        </span>
        
        <span>
            画笔大小: <input type="range" id="brushSize" min="1" max="50" value="5">
            <span id="sizeDisplay">5</span>px
        </span>
    </div>
    
    <canvas id="canvas" width="800" height="600"></canvas>

    <script>
        class DrawingBoard {
            constructor(canvasId) {
                this.canvas = document.getElementById(canvasId);
                this.ctx = this.setupHighDPICanvas(this.canvas);
                
                // 绘制状态
                this.isDrawing = false;
                this.currentTool = 'pen';
                this.currentColor = '#000000';
                this.currentSize = 5;
                this.lastX = 0;
                this.lastY = 0;
                
                this.setupEventListeners();
                this.setupToolbar();
            }
            
            setupHighDPICanvas(canvas) {
                const dpr = window.devicePixelRatio || 1;
                const rect = canvas.getBoundingClientRect();
                
                canvas.width = rect.width * dpr;
                canvas.height = rect.height * dpr;
                canvas.style.width = rect.width + 'px';
                canvas.style.height = rect.height + 'px';
                
                const ctx = canvas.getContext('2d');
                ctx.scale(dpr, dpr);
                
                return ctx;
            }
            
            setupEventListeners() {
                // 鼠标事件
                this.canvas.addEventListener('mousedown', this.startDrawing.bind(this));
                this.canvas.addEventListener('mousemove', this.draw.bind(this));
                this.canvas.addEventListener('mouseup', this.stopDrawing.bind(this));
                this.canvas.addEventListener('mouseout', this.stopDrawing.bind(this));
                
                // 触摸事件（移动设备支持）
                this.canvas.addEventListener('touchstart', this.handleTouch.bind(this));
                this.canvas.addEventListener('touchmove', this.handleTouch.bind(this));
                this.canvas.addEventListener('touchend', this.stopDrawing.bind(this));
            }
            
            setupToolbar() {
                // 工具切换
                document.getElementById('pen').addEventListener('click', () => {
                    this.setTool('pen');
                });
                
                document.getElementById('eraser').addEventListener('click', () => {
                    this.setTool('eraser');
                });
                
                document.getElementById('clear').addEventListener('click', () => {
                    this.clearCanvas();
                });
                
                // 颜色选择
                document.getElementById('colorPicker').addEventListener('change', (e) => {
                    this.currentColor = e.target.value;
                });
                
                // 画笔大小
                const brushSize = document.getElementById('brushSize');
                const sizeDisplay = document.getElementById('sizeDisplay');
                
                brushSize.addEventListener('input', (e) => {
                    this.currentSize = e.target.value;
                    sizeDisplay.textContent = e.target.value;
                });
            }
            
            setTool(tool) {
                this.currentTool = tool;
                
                // 更新 UI
                document.querySelectorAll('.toolbar button').forEach(btn => {
                    btn.classList.remove('active');
                });
                document.getElementById(tool).classList.add('active');
                
                // 更新光标
                this.canvas.style.cursor = tool === 'eraser' ? 'grab' : 'crosshair';
            }
            
            getMousePos(e) {
                const rect = this.canvas.getBoundingClientRect();
                return {
                    x: e.clientX - rect.left,
                    y: e.clientY - rect.top
                };
            }
            
            startDrawing(e) {
                this.isDrawing = true;
                const pos = this.getMousePos(e);
                this.lastX = pos.x;
                this.lastY = pos.y;
                
                // 设置绘制样式
                this.ctx.lineCap = 'round';
                this.ctx.lineJoin = 'round';
                this.ctx.lineWidth = this.currentSize;
                
                if (this.currentTool === 'pen') {
                    this.ctx.globalCompositeOperation = 'source-over';
                    this.ctx.strokeStyle = this.currentColor;
                } else if (this.currentTool === 'eraser') {
                    this.ctx.globalCompositeOperation = 'destination-out';
                }
            }
            
            draw(e) {
                if (!this.isDrawing) return;
                
                const pos = this.getMousePos(e);
                
                this.ctx.beginPath();
                this.ctx.moveTo(this.lastX, this.lastY);
                this.ctx.lineTo(pos.x, pos.y);
                this.ctx.stroke();
                
                this.lastX = pos.x;
                this.lastY = pos.y;
            }
            
            stopDrawing() {
                this.isDrawing = false;
            }
            
            handleTouch(e) {
                e.preventDefault();
                const touch = e.touches[0];
                const mouseEvent = new MouseEvent(e.type === 'touchstart' ? 'mousedown' : 
                                                 e.type === 'touchmove' ? 'mousemove' : 'mouseup', {
                    clientX: touch.clientX,
                    clientY: touch.clientY
                });
                this.canvas.dispatchEvent(mouseEvent);
            }
            
            clearCanvas() {
                this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
            }
        }
        
        // 初始化绘图板
        const drawingBoard = new DrawingBoard('canvas');
    </script>
</body>
</html>
```

---

## 7. 常见问题和解决方案

### 7.1 Canvas 模糊问题
```javascript
// 问题：线条或文字模糊
// 原因：坐标不是整数
// 解决：使用 Math.floor() 或添加 0.5 偏移
ctx.moveTo(Math.floor(x) + 0.5, Math.floor(y) + 0.5);
```

### 7.2 内存泄漏
```javascript
// 问题：Canvas 占用内存过多
// 原因：没有及时清理
// 解决：定期清理和重用
ctx.clearRect(0, 0, canvas.width, canvas.height);

// 或重置 Canvas
canvas.width = canvas.width;  // 这会清空 Canvas 并重置所有状态
```

### 7.3 事件坐标转换
```javascript
// 问题：鼠标坐标与 Canvas 坐标不匹配
// 原因：Canvas 有 CSS 缩放或定位
// 解决：正确计算坐标
function getCanvasCoordinates(canvas, event) {
    const rect = canvas.getBoundingClientRect();
    const scaleX = canvas.width / rect.width;
    const scaleY = canvas.height / rect.height;
    
    return {
        x: (event.clientX - rect.left) * scaleX,
        y: (event.clientY - rect.top) * scaleY
    };
}
```

---

## 8. 进阶主题预告

在接下来的章节中，我们将基于这些 Canvas 基础知识构建更复杂的功能：

1. **数学几何基础** - 向量运算、变换矩阵、碰撞检测
2. **事件系统** - 统一的指针事件、手势识别
3. **状态管理** - 大型应用的状态架构
4. **元素系统** - 可编辑图形对象的设计
5. **渲染引擎** - 高性能渲染和优化

---

## 🎯 本章总结

通过本章学习，你应该掌握了：

✅ **Canvas 核心概念**
- 即时模式 vs 保留模式的差异
- 为什么选择 Canvas 构建图形编辑器

✅ **Canvas API 完全掌握**
- 所有基础绘制方法
- 变换系统的使用
- 样式和文本处理

✅ **高 DPI 适配方案**
- 设备像素比的处理
- 响应式 Canvas 的实现

✅ **性能优化基础**
- 避免不必要的重绘
- 状态管理优化
- 离屏渲染技巧

✅ **完整的实战项目**
- 功能完整的绘图板
- 多工具支持
- 移动端兼容

现在你已经具备了构建复杂画板应用的基础能力！下一章我们将深入数学几何，学习如何处理复杂的图形变换和碰撞检测。