# 第一章：Canvas 基础概念与 API

## 学习目标

- [ ] 理解 Canvas 的本质和应用场景
- [ ] 掌握 Canvas 与 SVG、DOM 的区别和选择依据
- [ ] 熟悉 Canvas 2D 上下文的获取和基本设置
- [ ] 理解坐标系统和像素操作原理
- [ ] 掌握设备像素比的处理方法
- [ ] 能够创建一个响应式的基础画布

## 1. Canvas 是什么？

### 1.1 定义与本质

Canvas 是 HTML5 提供的一个**位图画布**元素，它本质上是一个**像素级别的绘图 API**。

```html
<canvas id="myCanvas" width="800" height="600">
  您的浏览器不支持 Canvas
</canvas>
```

关键特性：
- **即时模式（Immediate Mode）**：绘制后立即生成位图，没有对象模型
- **像素操作**：可以直接操作每个像素
- **低级 API**：需要手动管理绘制和重绘
- **高性能**：适合复杂动画和游戏

### 1.2 Canvas vs SVG vs DOM

| 特性 | Canvas | SVG | DOM |
|------|--------|-----|-----|
| **渲染方式** | 位图（栅格） | 矢量 | 文档对象 |
| **性能** | 大量对象时性能好 | 少量复杂图形好 | 简单交互好 |
| **内存占用** | 固定（宽×高） | 随元素增加 | 随元素增加 |
| **事件处理** | 需手动实现 | 原生支持 | 原生支持 |
| **缩放** | 失真 | 不失真 | 不失真 |
| **适用场景** | 游戏、图像编辑、数据可视化 | 图标、图表、插画 | 网页布局、UI |

**Excalidraw 为什么选择 Canvas？**
```javascript
// Excalidraw 需要处理大量图形元素的实时渲染
// Canvas 在这种场景下性能优势明显
const reasons = {
  performance: "可以渲染数千个元素保持流畅",
  flexibility: "完全控制渲染过程",
  effects: "易于实现手绘效果",
  export: "直接导出为图片"
};
```

## 2. Canvas 2D 上下文详解

### 2.1 获取上下文

```javascript
// 获取 Canvas 元素
const canvas = document.getElementById('myCanvas');

// 获取 2D 绘图上下文
const ctx = canvas.getContext('2d', {
  alpha: true,           // 是否包含透明通道
  desynchronized: false, // 是否与事件循环异步
  colorSpace: 'srgb',   // 颜色空间
  willReadFrequently: false // 是否频繁读取像素
});

// 检查是否支持
if (!ctx) {
  console.error('浏览器不支持 Canvas 2D');
}
```

### 2.2 上下文的核心属性

```javascript
// 样式属性
ctx.fillStyle = '#FF0000';      // 填充颜色
ctx.strokeStyle = '#000000';    // 描边颜色
ctx.lineWidth = 2;              // 线宽
ctx.lineCap = 'round';          // 线端样式: butt, round, square
ctx.lineJoin = 'miter';         // 线连接样式: miter, round, bevel
ctx.globalAlpha = 0.8;          // 全局透明度
ctx.globalCompositeOperation = 'source-over'; // 混合模式

// 文本属性
ctx.font = '16px Arial';        // 字体
ctx.textAlign = 'left';         // 水平对齐: left, right, center, start, end
ctx.textBaseline = 'top';       // 垂直对齐: top, middle, bottom, alphabetic

// 阴影属性
ctx.shadowColor = 'rgba(0,0,0,0.5)';
ctx.shadowBlur = 10;
ctx.shadowOffsetX = 5;
ctx.shadowOffsetY = 5;
```

## 3. 坐标系统与像素操作

### 3.1 坐标系统

Canvas 使用**左手坐标系**：
- 原点 (0,0) 在左上角
- X 轴向右为正
- Y 轴向下为正

```javascript
// 坐标系示例
function drawCoordinateSystem(ctx) {
  const width = ctx.canvas.width;
  const height = ctx.canvas.height;

  // 清空画布
  ctx.clearRect(0, 0, width, height);

  // 绘制坐标轴
  ctx.strokeStyle = '#666';
  ctx.lineWidth = 1;

  // X 轴
  ctx.beginPath();
  ctx.moveTo(0, height/2);
  ctx.lineTo(width, height/2);
  ctx.stroke();

  // Y 轴
  ctx.beginPath();
  ctx.moveTo(width/2, 0);
  ctx.lineTo(width/2, height);
  ctx.stroke();

  // 标注
  ctx.fillStyle = '#333';
  ctx.font = '14px monospace';
  ctx.fillText('(0,0)', 5, 15);
  ctx.fillText(`(${width},0)`, width - 60, 15);
  ctx.fillText(`(0,${height})`, 5, height - 5);
  ctx.fillText(`(${width},${height})`, width - 80, height - 5);
}
```

### 3.2 像素操作

```javascript
// 获取像素数据
const imageData = ctx.getImageData(x, y, width, height);
const data = imageData.data; // Uint8ClampedArray

// 像素数据结构：[R, G, B, A, R, G, B, A, ...]
// 每个像素占 4 个字节

// 遍历所有像素
for (let i = 0; i < data.length; i += 4) {
  const red = data[i];
  const green = data[i + 1];
  const blue = data[i + 2];
  const alpha = data[i + 3];

  // 修改像素（例如：灰度化）
  const gray = red * 0.299 + green * 0.587 + blue * 0.114;
  data[i] = gray;
  data[i + 1] = gray;
  data[i + 2] = gray;
}

// 写回像素数据
ctx.putImageData(imageData, x, y);
```

## 4. 设备像素比（DPR）处理

### 4.1 什么是设备像素比？

设备像素比（Device Pixel Ratio）是**物理像素**与**CSS 像素**的比率。

```javascript
const dpr = window.devicePixelRatio || 1;
console.log(`设备像素比: ${dpr}`);
// 普通屏幕: 1
// Retina 屏幕: 2 或 3
```

### 4.2 高清屏幕适配

不处理 DPR 会导致在高清屏幕上图像模糊：

```javascript
// ❌ 错误方式：图像会模糊
const canvas = document.getElementById('canvas');
canvas.width = 800;
canvas.height = 600;
```

正确的处理方式：

```javascript
// ✅ 正确方式：高清适配
function setupCanvas(canvas, width, height) {
  const dpr = window.devicePixelRatio || 1;

  // 设置实际渲染尺寸（物理像素）
  canvas.width = width * dpr;
  canvas.height = height * dpr;

  // 设置显示尺寸（CSS 像素）
  canvas.style.width = width + 'px';
  canvas.style.height = height + 'px';

  // 缩放上下文以匹配设备像素比
  const ctx = canvas.getContext('2d');
  ctx.scale(dpr, dpr);

  return ctx;
}

// 使用示例
const canvas = document.getElementById('myCanvas');
const ctx = setupCanvas(canvas, 800, 600);
```

### 4.3 坐标转换

处理了 DPR 后，需要正确转换坐标：

```javascript
class CanvasCoordinate {
  constructor(canvas) {
    this.canvas = canvas;
    this.dpr = window.devicePixelRatio || 1;
  }

  // 将鼠标事件坐标转换为 Canvas 坐标
  getCanvasPoint(event) {
    const rect = this.canvas.getBoundingClientRect();
    return {
      x: event.clientX - rect.left,
      y: event.clientY - rect.top
    };
  }

  // 将 Canvas 坐标转换为实际像素坐标
  toPixelPoint(canvasPoint) {
    return {
      x: canvasPoint.x * this.dpr,
      y: canvasPoint.y * this.dpr
    };
  }
}
```

## 5. 实战：创建响应式画布

### 5.1 完整示例

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>响应式 Canvas 画布</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial;
      background: #f5f5f5;
    }

    #canvas-container {
      width: 100%;
      height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      padding: 20px;
    }

    #myCanvas {
      background: white;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
      border-radius: 8px;
      cursor: crosshair;
    }

    .info {
      position: fixed;
      top: 20px;
      left: 20px;
      background: white;
      padding: 15px;
      border-radius: 8px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
      font-size: 14px;
      font-family: monospace;
    }
  </style>
</head>
<body>
  <div id="canvas-container">
    <canvas id="myCanvas"></canvas>
  </div>

  <div class="info">
    <div>DPR: <span id="dpr">1</span></div>
    <div>Canvas Size: <span id="size">0x0</span></div>
    <div>Mouse: <span id="mouse">0, 0</span></div>
    <div>FPS: <span id="fps">0</span></div>
  </div>

  <script>
    class ResponsiveCanvas {
      constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = null;
        this.dpr = window.devicePixelRatio || 1;
        this.animationId = null;
        this.lastTime = 0;
        this.fps = 0;

        // 绘制相关
        this.particles = [];
        this.mousePos = { x: 0, y: 0 };

        this.init();
      }

      init() {
        // 设置画布尺寸
        this.resize();

        // 监听窗口大小变化
        window.addEventListener('resize', () => this.resize());

        // 监听鼠标移动
        this.canvas.addEventListener('mousemove', (e) => this.handleMouseMove(e));

        // 初始化粒子
        this.initParticles();

        // 开始动画
        this.animate();

        // 更新信息显示
        this.updateInfo();
      }

      resize() {
        // 计算画布尺寸（留出边距）
        const containerWidth = window.innerWidth - 40;
        const containerHeight = window.innerHeight - 40;

        // 限制最大尺寸
        const width = Math.min(containerWidth, 1200);
        const height = Math.min(containerHeight, 800);

        // 设置物理像素尺寸
        this.canvas.width = width * this.dpr;
        this.canvas.height = height * this.dpr;

        // 设置 CSS 显示尺寸
        this.canvas.style.width = width + 'px';
        this.canvas.style.height = height + 'px';

        // 获取上下文并缩放
        this.ctx = this.canvas.getContext('2d');
        this.ctx.scale(this.dpr, this.dpr);

        // 设置默认样式
        this.ctx.lineCap = 'round';
        this.ctx.lineJoin = 'round';
      }

      handleMouseMove(event) {
        const rect = this.canvas.getBoundingClientRect();
        this.mousePos = {
          x: event.clientX - rect.left,
          y: event.clientY - rect.top
        };

        document.getElementById('mouse').textContent =
          `${Math.round(this.mousePos.x)}, ${Math.round(this.mousePos.y)}`;
      }

      initParticles() {
        this.particles = [];
        const count = 50;

        for (let i = 0; i < count; i++) {
          this.particles.push({
            x: Math.random() * (this.canvas.width / this.dpr),
            y: Math.random() * (this.canvas.height / this.dpr),
            vx: (Math.random() - 0.5) * 2,
            vy: (Math.random() - 0.5) * 2,
            radius: Math.random() * 3 + 1,
            color: `hsl(${Math.random() * 360}, 70%, 60%)`
          });
        }
      }

      updateParticles() {
        const width = this.canvas.width / this.dpr;
        const height = this.canvas.height / this.dpr;

        this.particles.forEach(p => {
          // 更新位置
          p.x += p.vx;
          p.y += p.vy;

          // 边界检测
          if (p.x < 0 || p.x > width) p.vx *= -1;
          if (p.y < 0 || p.y > height) p.vy *= -1;

          // 鼠标吸引
          const dx = this.mousePos.x - p.x;
          const dy = this.mousePos.y - p.y;
          const dist = Math.sqrt(dx * dx + dy * dy);

          if (dist < 100) {
            p.vx += dx * 0.001;
            p.vy += dy * 0.001;
          }

          // 速度衰减
          p.vx *= 0.999;
          p.vy *= 0.999;
        });
      }

      draw() {
        const ctx = this.ctx;
        const width = this.canvas.width / this.dpr;
        const height = this.canvas.height / this.dpr;

        // 清空画布（半透明，产生拖尾效果）
        ctx.fillStyle = 'rgba(255, 255, 255, 0.1)';
        ctx.fillRect(0, 0, width, height);

        // 绘制网格
        this.drawGrid();

        // 绘制粒子
        this.particles.forEach(p => {
          ctx.beginPath();
          ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
          ctx.fillStyle = p.color;
          ctx.fill();
        });

        // 绘制连线
        this.drawConnections();
      }

      drawGrid() {
        const ctx = this.ctx;
        const width = this.canvas.width / this.dpr;
        const height = this.canvas.height / this.dpr;
        const gridSize = 50;

        ctx.strokeStyle = 'rgba(0, 0, 0, 0.05)';
        ctx.lineWidth = 0.5;

        // 垂直线
        for (let x = 0; x <= width; x += gridSize) {
          ctx.beginPath();
          ctx.moveTo(x, 0);
          ctx.lineTo(x, height);
          ctx.stroke();
        }

        // 水平线
        for (let y = 0; y <= height; y += gridSize) {
          ctx.beginPath();
          ctx.moveTo(0, y);
          ctx.lineTo(width, y);
          ctx.stroke();
        }
      }

      drawConnections() {
        const ctx = this.ctx;

        this.particles.forEach((p1, i) => {
          this.particles.slice(i + 1).forEach(p2 => {
            const dx = p2.x - p1.x;
            const dy = p2.y - p1.y;
            const dist = Math.sqrt(dx * dx + dy * dy);

            if (dist < 80) {
              ctx.beginPath();
              ctx.moveTo(p1.x, p1.y);
              ctx.lineTo(p2.x, p2.y);
              ctx.strokeStyle = `rgba(0, 0, 0, ${0.2 * (1 - dist / 80)})`;
              ctx.lineWidth = 0.5;
              ctx.stroke();
            }
          });
        });
      }

      animate(currentTime = 0) {
        // 计算 FPS
        const deltaTime = currentTime - this.lastTime;
        if (deltaTime > 0) {
          this.fps = Math.round(1000 / deltaTime);
        }
        this.lastTime = currentTime;

        // 更新粒子
        this.updateParticles();

        // 绘制
        this.draw();

        // 继续动画
        this.animationId = requestAnimationFrame((time) => this.animate(time));
      }

      updateInfo() {
        document.getElementById('dpr').textContent = this.dpr.toFixed(1);
        document.getElementById('size').textContent =
          `${this.canvas.width}x${this.canvas.height} (物理) / ${this.canvas.style.width}x${this.canvas.style.height} (CSS)`;

        setInterval(() => {
          document.getElementById('fps').textContent = this.fps;
        }, 100);
      }

      destroy() {
        if (this.animationId) {
          cancelAnimationFrame(this.animationId);
        }
      }
    }

    // 初始化画布
    const canvas = new ResponsiveCanvas('myCanvas');
  </script>
</body>
</html>
```

### 5.2 关键技术点解析

```javascript
// 1. 响应式尺寸计算
function calculateCanvasSize() {
  // 获取容器尺寸
  const container = canvas.parentElement;
  const rect = container.getBoundingClientRect();

  // 考虑边距和最大尺寸
  const padding = 40;
  const maxWidth = 1200;
  const maxHeight = 800;

  return {
    width: Math.min(rect.width - padding, maxWidth),
    height: Math.min(rect.height - padding, maxHeight)
  };
}

// 2. 性能优化：节流 resize 事件
function throttle(func, limit) {
  let inThrottle;
  return function() {
    const args = arguments;
    const context = this;
    if (!inThrottle) {
      func.apply(context, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

window.addEventListener('resize', throttle(() => {
  canvas.resize();
}, 100));

// 3. 内存管理
class CanvasManager {
  constructor() {
    this.resources = new Set();
  }

  createImageBitmap(blob) {
    return createImageBitmap(blob).then(bitmap => {
      this.resources.add(bitmap);
      return bitmap;
    });
  }

  cleanup() {
    this.resources.forEach(resource => {
      if (resource.close) resource.close();
    });
    this.resources.clear();
  }
}
```

## 6. Excalidraw 中的应用

让我们看看 Excalidraw 是如何处理 Canvas 基础设置的：

```typescript
// packages/excalidraw/scene/Scene.tsx
export class Scene {
  private canvas: HTMLCanvasElement;
  private rc: RoughCanvas;
  private contextSettings = {
    alpha: true,
    desynchronized: true,  // 性能优化：异步渲染
    colorSpace: "srgb",
    willReadFrequently: false
  };

  constructor(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.setupCanvas();
  }

  private setupCanvas() {
    const context = this.canvas.getContext("2d", this.contextSettings);

    // 设置默认样式
    context.imageSmoothingEnabled = true;
    context.imageSmoothingQuality = "high";

    // 处理高 DPI
    this.handleHighDPI();

    // 初始化 RoughJS
    this.rc = rough.canvas(this.canvas);
  }

  private handleHighDPI() {
    const dpr = window.devicePixelRatio || 1;
    const rect = this.canvas.getBoundingClientRect();

    // 设置实际尺寸
    this.canvas.width = rect.width * dpr;
    this.canvas.height = rect.height * dpr;

    // 缩放上下文
    const context = this.canvas.getContext("2d");
    context.scale(dpr, dpr);

    // 设置 CSS 尺寸
    this.canvas.style.width = rect.width + "px";
    this.canvas.style.height = rect.height + "px";
  }
}
```

## 7. 练习题

### 7.1 基础练习

1. **创建自适应画布**
   - 创建一个全屏 Canvas
   - 处理窗口 resize 事件
   - 正确处理高 DPI 显示

2. **坐标系统可视化**
   - 绘制坐标轴和网格
   - 显示鼠标当前坐标
   - 实现坐标变换演示

3. **像素操作**
   - 实现图片灰度化
   - 实现马赛克效果
   - 实现简单的滤镜

### 7.2 进阶练习

1. **性能测试工具**
   ```javascript
   class PerformanceMonitor {
     constructor(canvas) {
       this.canvas = canvas;
       this.ctx = canvas.getContext('2d');
       this.metrics = {
         fps: 0,
         drawCalls: 0,
         renderTime: 0
       };
     }

     // 实现 FPS 监控
     // 实现绘制调用计数
     // 实现渲染时间统计
   }
   ```

2. **Canvas 管理器**
   ```javascript
   class CanvasLayerManager {
     constructor(container) {
       this.layers = new Map();
       this.container = container;
     }

     // 实现多层 Canvas 管理
     // 实现层的添加、删除、排序
     // 实现层的显示/隐藏
   }
   ```

## 8. 思考题

1. **为什么 Excalidraw 选择 Canvas 而不是 SVG？**
   - 考虑性能因素
   - 考虑功能需求
   - 考虑导出需求

2. **如何在 Canvas 中实现事件系统？**
   - Canvas 本身不支持对象级事件
   - 需要手动实现碰撞检测
   - 需要维护元素的空间索引

3. **设备像素比（DPR）为什么重要？**
   - 对渲染质量的影响
   - 对性能的影响
   - 对内存使用的影响

4. **Canvas 的性能瓶颈在哪里？**
   - 绘制调用次数
   - 状态切换成本
   - 像素填充率

5. **如何选择合适的 Canvas 尺寸？**
   - 平衡质量和性能
   - 考虑目标设备
   - 动态调整策略

## 9. 总结

### 核心要点

1. **Canvas 基础**
   - Canvas 是位图绘制 API，适合复杂动画和大量图形
   - 与 SVG 和 DOM 相比，Canvas 在特定场景下性能更好
   - 理解坐标系统和像素操作是基础

2. **高清适配**
   - 必须处理设备像素比（DPR）
   - 物理像素 vs CSS 像素的区别
   - 正确的缩放和坐标转换

3. **性能考虑**
   - 选择合适的上下文选项
   - 合理设置 Canvas 尺寸
   - 注意内存管理和资源清理

4. **实战技巧**
   - 响应式设计的实现
   - 事件处理的坐标转换
   - 多层渲染的管理

### 下一步

在掌握了 Canvas 基础后，下一章我们将学习：
- Canvas 的各种绘图 API
- 路径的概念和使用
- 样式和效果的应用
- 实现一个简单的绘图工具

## 10. 参考资源

- [MDN Canvas API](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API)
- [Canvas Deep Dive](https://joshondesign.com/p/books/canvasdeepdive/)
- [Excalidraw 源码](https://github.com/excalidraw/excalidraw)
- [HTML5 Canvas Tutorials](https://www.html5canvastutorials.com/)

---

**下一章**：[Canvas 图形绘制 →](./02-canvas-drawing.md)