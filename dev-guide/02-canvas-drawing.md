# 第二章：Canvas 图形绘制

## 学习目标

- [ ] 掌握 Canvas 路径（Path）的概念和 API
- [ ] 熟练绘制基本图形（矩形、圆形、多边形）
- [ ] 理解并应用贝塞尔曲线
- [ ] 掌握样式设置（颜色、渐变、图案、阴影）
- [ ] 实现一个功能完整的简单绘图工具

## 1. 路径（Path）概念与 API

### 1.1 什么是路径？

路径是 Canvas 中最重要的概念之一。它是一系列点和连接这些点的线段或曲线的集合。

```javascript
// 路径的基本使用流程
ctx.beginPath();      // 1. 开始新路径
ctx.moveTo(x, y);     // 2. 移动到起点
ctx.lineTo(x, y);     // 3. 绘制路径
ctx.closePath();      // 4. 闭合路径（可选）
ctx.stroke();         // 5. 描边 或 ctx.fill() 填充
```

### 1.2 路径 API 详解

```javascript
class PathDemo {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 基础路径方法
  drawBasicPath() {
    const ctx = this.ctx;

    // beginPath: 清空子路径列表，开始新路径
    ctx.beginPath();

    // moveTo: 移动画笔到指定点（不绘制）
    ctx.moveTo(50, 50);

    // lineTo: 从当前点绘制直线到指定点
    ctx.lineTo(150, 50);
    ctx.lineTo(150, 150);

    // closePath: 从当前点到起始点绘制直线，闭合路径
    ctx.closePath();

    // 设置样式
    ctx.strokeStyle = '#333';
    ctx.lineWidth = 2;
    ctx.fillStyle = 'rgba(100, 150, 200, 0.3)';

    // 绘制
    ctx.stroke();  // 描边
    ctx.fill();    // 填充
  }

  // 路径方向的重要性
  drawPathDirection() {
    const ctx = this.ctx;

    // 顺时针矩形
    ctx.beginPath();
    ctx.moveTo(50, 50);
    ctx.lineTo(150, 50);
    ctx.lineTo(150, 150);
    ctx.lineTo(50, 150);
    ctx.closePath();

    // 逆时针内部矩形（创建镂空效果）
    ctx.moveTo(100, 100);
    ctx.lineTo(100, 80);
    ctx.lineTo(120, 80);
    ctx.lineTo(120, 100);
    ctx.closePath();

    ctx.fill('evenodd'); // 使用奇偶规则填充
  }
}
```

### 1.3 路径的子路径

一个路径可以包含多个子路径：

```javascript
function drawMultipleSubPaths(ctx) {
  ctx.beginPath();

  // 第一个子路径：三角形
  ctx.moveTo(50, 50);
  ctx.lineTo(100, 100);
  ctx.lineTo(50, 100);
  ctx.closePath();

  // 第二个子路径：圆形
  ctx.moveTo(200, 75);
  ctx.arc(150, 75, 50, 0, Math.PI * 2);

  // 一次性绘制所有子路径
  ctx.stroke();
}
```

## 2. 基本图形绘制

### 2.1 矩形

Canvas 提供了专门的矩形绘制方法：

```javascript
class RectangleDrawing {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 直接绘制方法（不影响路径）
  drawDirectRectangles() {
    const ctx = this.ctx;

    // fillRect: 填充矩形
    ctx.fillStyle = 'blue';
    ctx.fillRect(50, 50, 100, 80);

    // strokeRect: 描边矩形
    ctx.strokeStyle = 'red';
    ctx.lineWidth = 3;
    ctx.strokeRect(200, 50, 100, 80);

    // clearRect: 清除矩形区域（创建透明区域）
    ctx.clearRect(75, 75, 50, 30);
  }

  // 使用路径绘制圆角矩形
  drawRoundedRect(x, y, width, height, radius) {
    const ctx = this.ctx;

    ctx.beginPath();
    ctx.moveTo(x + radius, y);
    ctx.lineTo(x + width - radius, y);
    ctx.arcTo(x + width, y, x + width, y + radius, radius);
    ctx.lineTo(x + width, y + height - radius);
    ctx.arcTo(x + width, y + height, x + width - radius, y + height, radius);
    ctx.lineTo(x + radius, y + height);
    ctx.arcTo(x, y + height, x, y + height - radius, radius);
    ctx.lineTo(x, y + radius);
    ctx.arcTo(x, y, x + radius, y, radius);
    ctx.closePath();

    return ctx;
  }

  // Excalidraw 风格的手绘矩形
  drawRoughRect(x, y, width, height, roughness = 1) {
    const ctx = this.ctx;
    const offset = () => (Math.random() - 0.5) * roughness;

    ctx.beginPath();

    // 绘制两遍，产生手绘效果
    for (let i = 0; i < 2; i++) {
      ctx.moveTo(x + offset(), y + offset());
      ctx.lineTo(x + width + offset(), y + offset());
      ctx.lineTo(x + width + offset(), y + height + offset());
      ctx.lineTo(x + offset(), y + height + offset());
      ctx.closePath();
    }

    ctx.stroke();
  }
}
```

### 2.2 圆形和椭圆

```javascript
class CircleDrawing {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 绘制圆形
  drawCircle(x, y, radius) {
    const ctx = this.ctx;

    ctx.beginPath();
    // arc(x, y, radius, startAngle, endAngle, anticlockwise)
    ctx.arc(x, y, radius, 0, Math.PI * 2);
    ctx.fill();
    ctx.stroke();
  }

  // 绘制扇形
  drawSector(x, y, radius, startAngle, endAngle) {
    const ctx = this.ctx;

    ctx.beginPath();
    ctx.moveTo(x, y);
    ctx.arc(x, y, radius, startAngle, endAngle);
    ctx.closePath();
    ctx.fill();
  }

  // 绘制椭圆
  drawEllipse(x, y, radiusX, radiusY, rotation = 0) {
    const ctx = this.ctx;

    ctx.beginPath();
    // ellipse(x, y, radiusX, radiusY, rotation, startAngle, endAngle)
    ctx.ellipse(x, y, radiusX, radiusY, rotation, 0, Math.PI * 2);
    ctx.stroke();
  }

  // 使用贝塞尔曲线绘制椭圆（兼容旧浏览器）
  drawEllipseWithBezier(x, y, radiusX, radiusY) {
    const ctx = this.ctx;
    const kappa = 0.5522848; // 4 * ((√2 - 1) / 3)
    const ox = radiusX * kappa;
    const oy = radiusY * kappa;

    ctx.beginPath();
    ctx.moveTo(x - radiusX, y);
    ctx.bezierCurveTo(x - radiusX, y - oy, x - ox, y - radiusY, x, y - radiusY);
    ctx.bezierCurveTo(x + ox, y - radiusY, x + radiusX, y - oy, x + radiusX, y);
    ctx.bezierCurveTo(x + radiusX, y + oy, x + ox, y + radiusY, x, y + radiusY);
    ctx.bezierCurveTo(x - ox, y + radiusY, x - radiusX, y + oy, x - radiusX, y);
    ctx.closePath();
    ctx.stroke();
  }
}
```

### 2.3 多边形

```javascript
class PolygonDrawing {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 绘制正多边形
  drawRegularPolygon(x, y, radius, sides, rotation = 0) {
    const ctx = this.ctx;
    const angle = (Math.PI * 2) / sides;

    ctx.beginPath();
    for (let i = 0; i < sides; i++) {
      const vertexAngle = angle * i + rotation;
      const px = x + radius * Math.cos(vertexAngle);
      const py = y + radius * Math.sin(vertexAngle);

      if (i === 0) {
        ctx.moveTo(px, py);
      } else {
        ctx.lineTo(px, py);
      }
    }
    ctx.closePath();
    ctx.stroke();
  }

  // 绘制星形
  drawStar(x, y, outerRadius, innerRadius, points) {
    const ctx = this.ctx;
    const angle = Math.PI / points;

    ctx.beginPath();
    for (let i = 0; i < points * 2; i++) {
      const radius = i % 2 === 0 ? outerRadius : innerRadius;
      const px = x + radius * Math.cos(angle * i - Math.PI / 2);
      const py = y + radius * Math.sin(angle * i - Math.PI / 2);

      if (i === 0) {
        ctx.moveTo(px, py);
      } else {
        ctx.lineTo(px, py);
      }
    }
    ctx.closePath();
    ctx.fill();
  }

  // 绘制箭头
  drawArrow(fromX, fromY, toX, toY, headSize = 10) {
    const ctx = this.ctx;
    const angle = Math.atan2(toY - fromY, toX - fromX);

    // 绘制线条
    ctx.beginPath();
    ctx.moveTo(fromX, fromY);
    ctx.lineTo(toX, toY);
    ctx.stroke();

    // 绘制箭头头部
    ctx.beginPath();
    ctx.moveTo(toX, toY);
    ctx.lineTo(
      toX - headSize * Math.cos(angle - Math.PI / 6),
      toY - headSize * Math.sin(angle - Math.PI / 6)
    );
    ctx.lineTo(
      toX - headSize * Math.cos(angle + Math.PI / 6),
      toY - headSize * Math.sin(angle + Math.PI / 6)
    );
    ctx.closePath();
    ctx.fill();
  }
}
```

## 3. 贝塞尔曲线

### 3.1 二次贝塞尔曲线

```javascript
class BezierCurves {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 二次贝塞尔曲线
  drawQuadraticCurve() {
    const ctx = this.ctx;

    ctx.beginPath();
    ctx.moveTo(50, 100);
    // quadraticCurveTo(cpx, cpy, x, y)
    // cpx, cpy: 控制点坐标
    // x, y: 终点坐标
    ctx.quadraticCurveTo(150, 50, 250, 100);
    ctx.stroke();

    // 绘制控制点（辅助理解）
    this.drawControlPoints([
      { x: 50, y: 100, label: '起点' },
      { x: 150, y: 50, label: '控制点' },
      { x: 250, y: 100, label: '终点' }
    ]);
  }

  // 三次贝塞尔曲线
  drawCubicCurve() {
    const ctx = this.ctx;

    ctx.beginPath();
    ctx.moveTo(50, 100);
    // bezierCurveTo(cp1x, cp1y, cp2x, cp2y, x, y)
    ctx.bezierCurveTo(100, 50, 200, 150, 250, 100);
    ctx.stroke();

    // 绘制控制点
    this.drawControlPoints([
      { x: 50, y: 100, label: '起点' },
      { x: 100, y: 50, label: '控制点1' },
      { x: 200, y: 150, label: '控制点2' },
      { x: 250, y: 100, label: '终点' }
    ]);
  }

  // 绘制控制点辅助
  drawControlPoints(points) {
    const ctx = this.ctx;

    ctx.save();
    ctx.fillStyle = 'red';
    ctx.strokeStyle = 'rgba(255, 0, 0, 0.3)';
    ctx.setLineDash([5, 5]);

    // 绘制控制线
    ctx.beginPath();
    points.forEach((point, i) => {
      if (i === 0) ctx.moveTo(point.x, point.y);
      else ctx.lineTo(point.x, point.y);
    });
    ctx.stroke();

    // 绘制控制点
    ctx.setLineDash([]);
    points.forEach(point => {
      ctx.beginPath();
      ctx.arc(point.x, point.y, 4, 0, Math.PI * 2);
      ctx.fill();

      // 绘制标签
      ctx.fillText(point.label, point.x + 10, point.y - 5);
    });

    ctx.restore();
  }

  // 平滑曲线（通过多个点）
  drawSmoothCurve(points) {
    const ctx = this.ctx;

    if (points.length < 2) return;

    ctx.beginPath();
    ctx.moveTo(points[0].x, points[0].y);

    if (points.length === 2) {
      ctx.lineTo(points[1].x, points[1].y);
    } else {
      // 使用 Catmull-Rom 样条曲线
      for (let i = 1; i < points.length - 1; i++) {
        const cp1x = points[i].x - (points[i + 1].x - points[i - 1].x) / 6;
        const cp1y = points[i].y - (points[i + 1].y - points[i - 1].y) / 6;
        const cp2x = points[i + 1].x + (points[i + 1].x - points[i - 1].x) / 6;
        const cp2y = points[i + 1].y + (points[i + 1].y - points[i - 1].y) / 6;

        ctx.bezierCurveTo(cp1x, cp1y, cp2x, cp2y, points[i + 1].x, points[i + 1].y);
      }
    }

    ctx.stroke();
  }
}
```

### 3.2 实现手绘效果

```javascript
// Excalidraw 风格的手绘曲线
class HandDrawnCurve {
  constructor(ctx) {
    this.ctx = ctx;
    this.roughness = 1.5;
  }

  // 手绘直线
  drawRoughLine(x1, y1, x2, y2) {
    const ctx = this.ctx;
    const offset = this.roughness;

    // 绘制两次，略有偏移
    for (let i = 0; i < 2; i++) {
      ctx.beginPath();

      const dx = x2 - x1;
      const dy = y2 - y1;
      const distance = Math.sqrt(dx * dx + dy * dy);
      const steps = Math.max(5, distance / 10);

      for (let j = 0; j <= steps; j++) {
        const t = j / steps;
        const x = x1 + dx * t + (Math.random() - 0.5) * offset;
        const y = y1 + dy * t + (Math.random() - 0.5) * offset;

        if (j === 0) {
          ctx.moveTo(x, y);
        } else {
          ctx.lineTo(x, y);
        }
      }

      ctx.stroke();
    }
  }

  // 手绘圆形
  drawRoughCircle(cx, cy, radius) {
    const ctx = this.ctx;
    const points = [];
    const numPoints = 20;

    // 生成圆上的点
    for (let i = 0; i <= numPoints; i++) {
      const angle = (i / numPoints) * Math.PI * 2;
      const offset = this.roughness * (1 + Math.random() * 0.5);
      const r = radius + (Math.random() - 0.5) * offset;
      points.push({
        x: cx + r * Math.cos(angle),
        y: cy + r * Math.sin(angle)
      });
    }

    // 绘制两次
    for (let pass = 0; pass < 2; pass++) {
      ctx.beginPath();
      points.forEach((point, i) => {
        const x = point.x + (Math.random() - 0.5) * this.roughness;
        const y = point.y + (Math.random() - 0.5) * this.roughness;

        if (i === 0) {
          ctx.moveTo(x, y);
        } else {
          ctx.lineTo(x, y);
        }
      });
      ctx.stroke();
    }
  }
}
```

## 4. 样式设置

### 4.1 颜色和透明度

```javascript
class StyleSettings {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 颜色格式
  demonstrateColors() {
    const ctx = this.ctx;

    // 十六进制颜色
    ctx.fillStyle = '#FF5733';

    // RGB
    ctx.fillStyle = 'rgb(255, 87, 51)';

    // RGBA（带透明度）
    ctx.fillStyle = 'rgba(255, 87, 51, 0.5)';

    // HSL
    ctx.fillStyle = 'hsl(9, 100%, 60%)';

    // HSLA
    ctx.fillStyle = 'hsla(9, 100%, 60%, 0.5)';

    // 预定义颜色名
    ctx.fillStyle = 'coral';

    // 全局透明度
    ctx.globalAlpha = 0.7;
  }

  // 线条样式
  demonstrateLineStyles() {
    const ctx = this.ctx;

    // 线宽
    ctx.lineWidth = 5;

    // 线端样式
    ctx.lineCap = 'round'; // 'butt' | 'round' | 'square'

    // 线连接样式
    ctx.lineJoin = 'round'; // 'miter' | 'round' | 'bevel'

    // 虚线
    ctx.setLineDash([10, 5]); // [实线长度, 间隙长度]
    ctx.lineDashOffset = 0;   // 虚线偏移

    // 斜接限制
    ctx.miterLimit = 10;
  }
}
```

### 4.2 渐变

```javascript
class GradientStyles {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 线性渐变
  createLinearGradient() {
    const ctx = this.ctx;

    // 创建渐变对象
    const gradient = ctx.createLinearGradient(0, 0, 200, 0);

    // 添加颜色停止点
    gradient.addColorStop(0, 'red');
    gradient.addColorStop(0.5, 'yellow');
    gradient.addColorStop(1, 'green');

    // 应用渐变
    ctx.fillStyle = gradient;
    ctx.fillRect(50, 50, 200, 100);
  }

  // 径向渐变
  createRadialGradient() {
    const ctx = this.ctx;

    // createRadialGradient(x0, y0, r0, x1, y1, r1)
    const gradient = ctx.createRadialGradient(150, 150, 20, 150, 150, 100);

    gradient.addColorStop(0, 'white');
    gradient.addColorStop(0.5, 'lightblue');
    gradient.addColorStop(1, 'darkblue');

    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.arc(150, 150, 100, 0, Math.PI * 2);
    ctx.fill();
  }

  // 锥形渐变（较新的 API）
  createConicGradient() {
    const ctx = this.ctx;

    if (ctx.createConicGradient) {
      // createConicGradient(startAngle, centerX, centerY)
      const gradient = ctx.createConicGradient(0, 150, 150);

      gradient.addColorStop(0, 'red');
      gradient.addColorStop(0.25, 'yellow');
      gradient.addColorStop(0.5, 'green');
      gradient.addColorStop(0.75, 'blue');
      gradient.addColorStop(1, 'red');

      ctx.fillStyle = gradient;
      ctx.beginPath();
      ctx.arc(150, 150, 100, 0, Math.PI * 2);
      ctx.fill();
    }
  }
}
```

### 4.3 图案和阴影

```javascript
class PatternAndShadow {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 创建图案
  async createPattern() {
    const ctx = this.ctx;

    // 使用图片创建图案
    const img = new Image();
    img.src = 'pattern.png';

    await new Promise(resolve => img.onload = resolve);

    // createPattern(image, repetition)
    // repetition: 'repeat' | 'repeat-x' | 'repeat-y' | 'no-repeat'
    const pattern = ctx.createPattern(img, 'repeat');
    ctx.fillStyle = pattern;
    ctx.fillRect(0, 0, 300, 300);
  }

  // 使用 Canvas 创建图案
  createCanvasPattern() {
    const ctx = this.ctx;

    // 创建小的 Canvas 作为图案
    const patternCanvas = document.createElement('canvas');
    patternCanvas.width = 20;
    patternCanvas.height = 20;
    const patternCtx = patternCanvas.getContext('2d');

    // 在图案 Canvas 上绘制
    patternCtx.fillStyle = '#ffc';
    patternCtx.fillRect(0, 0, 20, 20);
    patternCtx.fillStyle = '#f00';
    patternCtx.fillRect(0, 0, 10, 10);
    patternCtx.fillRect(10, 10, 10, 10);

    // 创建图案
    const pattern = ctx.createPattern(patternCanvas, 'repeat');
    ctx.fillStyle = pattern;
    ctx.fillRect(0, 0, 200, 200);
  }

  // 阴影效果
  drawShadow() {
    const ctx = this.ctx;

    // 设置阴影属性
    ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
    ctx.shadowBlur = 10;
    ctx.shadowOffsetX = 5;
    ctx.shadowOffsetY = 5;

    // 绘制带阴影的图形
    ctx.fillStyle = '#3498db';
    ctx.fillRect(50, 50, 100, 100);

    // 重置阴影
    ctx.shadowColor = 'transparent';
  }
}
```

## 5. 实战：简单绘图工具

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Canvas 绘图工具</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Arial;
      display: flex;
      height: 100vh;
      background: #f0f0f0;
    }

    .toolbar {
      width: 200px;
      background: white;
      padding: 20px;
      box-shadow: 2px 0 5px rgba(0,0,0,0.1);
    }

    .tool-group {
      margin-bottom: 20px;
    }

    .tool-group h3 {
      margin-bottom: 10px;
      font-size: 14px;
      color: #666;
    }

    .tools {
      display: grid;
      grid-template-columns: repeat(3, 1fr);
      gap: 5px;
    }

    .tool {
      width: 50px;
      height: 50px;
      border: 2px solid #ddd;
      background: white;
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 4px;
      transition: all 0.3s;
    }

    .tool:hover {
      background: #f0f0f0;
    }

    .tool.active {
      border-color: #3498db;
      background: #e8f4fd;
    }

    .color-picker {
      display: grid;
      grid-template-columns: repeat(5, 1fr);
      gap: 5px;
    }

    .color {
      width: 30px;
      height: 30px;
      border: 2px solid #ddd;
      border-radius: 4px;
      cursor: pointer;
    }

    .color.active {
      border-color: #333;
    }

    .canvas-container {
      flex: 1;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 20px;
    }

    canvas {
      background: white;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
      cursor: crosshair;
    }

    .controls {
      padding: 10px 0;
    }

    .control {
      margin-bottom: 10px;
    }

    label {
      display: block;
      font-size: 12px;
      margin-bottom: 5px;
      color: #666;
    }

    input[type="range"] {
      width: 100%;
    }

    button {
      width: 100%;
      padding: 8px;
      background: #3498db;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 14px;
    }

    button:hover {
      background: #2980b9;
    }
  </style>
</head>
<body>
  <div class="toolbar">
    <div class="tool-group">
      <h3>绘图工具</h3>
      <div class="tools">
        <div class="tool active" data-tool="pen">
          <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <path d="M12 19l7-7 3 3-7 7-3-3z"/>
            <path d="M18 13l-1.5-7.5L2 2l3.5 14.5L13 18l5-5z"/>
            <path d="M2 2l7.586 7.586"/>
            <circle cx="11" cy="11" r="2"/>
          </svg>
        </div>
        <div class="tool" data-tool="line">
          <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <line x1="5" y1="12" x2="19" y2="12"/>
          </svg>
        </div>
        <div class="tool" data-tool="rectangle">
          <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <rect x="3" y="3" width="18" height="18" rx="2" ry="2"/>
          </svg>
        </div>
        <div class="tool" data-tool="circle">
          <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <circle cx="12" cy="12" r="10"/>
          </svg>
        </div>
        <div class="tool" data-tool="ellipse">
          <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <ellipse cx="12" cy="12" rx="10" ry="6"/>
          </svg>
        </div>
        <div class="tool" data-tool="polygon">
          <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <polygon points="12,2 22,17 2,17"/>
          </svg>
        </div>
      </div>
    </div>

    <div class="tool-group">
      <h3>颜色</h3>
      <div class="color-picker">
        <div class="color active" style="background: #000000" data-color="#000000"></div>
        <div class="color" style="background: #FF0000" data-color="#FF0000"></div>
        <div class="color" style="background: #00FF00" data-color="#00FF00"></div>
        <div class="color" style="background: #0000FF" data-color="#0000FF"></div>
        <div class="color" style="background: #FFFF00" data-color="#FFFF00"></div>
        <div class="color" style="background: #FF00FF" data-color="#FF00FF"></div>
        <div class="color" style="background: #00FFFF" data-color="#00FFFF"></div>
        <div class="color" style="background: #FFA500" data-color="#FFA500"></div>
        <div class="color" style="background: #800080" data-color="#800080"></div>
        <div class="color" style="background: #FFC0CB" data-color="#FFC0CB"></div>
      </div>
    </div>

    <div class="tool-group">
      <h3>设置</h3>
      <div class="controls">
        <div class="control">
          <label>线宽: <span id="lineWidthValue">2</span></label>
          <input type="range" id="lineWidth" min="1" max="20" value="2">
        </div>
        <div class="control">
          <label>透明度: <span id="opacityValue">100</span>%</label>
          <input type="range" id="opacity" min="0" max="100" value="100">
        </div>
        <div class="control">
          <label>
            <input type="checkbox" id="fillShape"> 填充图形
          </label>
        </div>
      </div>
    </div>

    <div class="tool-group">
      <button id="clearCanvas">清空画布</button>
    </div>
  </div>

  <div class="canvas-container">
    <canvas id="drawingCanvas"></canvas>
  </div>

  <script>
    class DrawingApp {
      constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = this.canvas.getContext('2d');

        // 工具状态
        this.currentTool = 'pen';
        this.isDrawing = false;
        this.startPoint = null;
        this.currentPath = [];

        // 样式设置
        this.strokeColor = '#000000';
        this.fillColor = '#000000';
        this.lineWidth = 2;
        this.opacity = 1;
        this.fillShape = false;

        // 历史记录
        this.history = [];
        this.historyIndex = -1;

        this.init();
      }

      init() {
        // 设置画布尺寸
        this.resizeCanvas();
        window.addEventListener('resize', () => this.resizeCanvas());

        // 绑定工具选择
        document.querySelectorAll('.tool').forEach(tool => {
          tool.addEventListener('click', (e) => {
            document.querySelectorAll('.tool').forEach(t => t.classList.remove('active'));
            tool.classList.add('active');
            this.currentTool = tool.dataset.tool;
          });
        });

        // 绑定颜色选择
        document.querySelectorAll('.color').forEach(color => {
          color.addEventListener('click', (e) => {
            document.querySelectorAll('.color').forEach(c => c.classList.remove('active'));
            color.classList.add('active');
            this.strokeColor = color.dataset.color;
            this.fillColor = color.dataset.color;
          });
        });

        // 绑定设置控件
        document.getElementById('lineWidth').addEventListener('input', (e) => {
          this.lineWidth = e.target.value;
          document.getElementById('lineWidthValue').textContent = e.target.value;
        });

        document.getElementById('opacity').addEventListener('input', (e) => {
          this.opacity = e.target.value / 100;
          document.getElementById('opacityValue').textContent = e.target.value;
        });

        document.getElementById('fillShape').addEventListener('change', (e) => {
          this.fillShape = e.target.checked;
        });

        document.getElementById('clearCanvas').addEventListener('click', () => {
          this.clearCanvas();
        });

        // 绑定鼠标事件
        this.canvas.addEventListener('mousedown', (e) => this.onMouseDown(e));
        this.canvas.addEventListener('mousemove', (e) => this.onMouseMove(e));
        this.canvas.addEventListener('mouseup', (e) => this.onMouseUp(e));
        this.canvas.addEventListener('mouseout', (e) => this.onMouseUp(e));

        // 触摸事件支持
        this.canvas.addEventListener('touchstart', (e) => this.onTouchStart(e));
        this.canvas.addEventListener('touchmove', (e) => this.onTouchMove(e));
        this.canvas.addEventListener('touchend', (e) => this.onTouchEnd(e));

        // 键盘快捷键
        document.addEventListener('keydown', (e) => this.onKeyDown(e));
      }

      resizeCanvas() {
        const container = this.canvas.parentElement;
        const rect = container.getBoundingClientRect();
        const width = Math.min(rect.width - 40, 1000);
        const height = Math.min(rect.height - 40, 700);

        const dpr = window.devicePixelRatio || 1;
        this.canvas.width = width * dpr;
        this.canvas.height = height * dpr;
        this.canvas.style.width = width + 'px';
        this.canvas.style.height = height + 'px';

        this.ctx.scale(dpr, dpr);

        // 恢复绘制内容
        if (this.history.length > 0 && this.historyIndex >= 0) {
          this.restoreFromHistory();
        }
      }

      getMousePos(e) {
        const rect = this.canvas.getBoundingClientRect();
        return {
          x: e.clientX - rect.left,
          y: e.clientY - rect.top
        };
      }

      onMouseDown(e) {
        this.isDrawing = true;
        this.startPoint = this.getMousePos(e);
        this.currentPath = [this.startPoint];

        if (this.currentTool === 'pen') {
          this.beginPath();
        }

        // 保存画布状态（用于形状工具）
        if (this.currentTool !== 'pen') {
          this.saveCanvasState();
        }
      }

      onMouseMove(e) {
        if (!this.isDrawing) return;

        const currentPoint = this.getMousePos(e);

        switch (this.currentTool) {
          case 'pen':
            this.drawPen(currentPoint);
            break;
          case 'line':
            this.drawLine(currentPoint);
            break;
          case 'rectangle':
            this.drawRectangle(currentPoint);
            break;
          case 'circle':
            this.drawCircle(currentPoint);
            break;
          case 'ellipse':
            this.drawEllipse(currentPoint);
            break;
          case 'polygon':
            this.drawTriangle(currentPoint);
            break;
        }

        this.currentPath.push(currentPoint);
      }

      onMouseUp(e) {
        if (!this.isDrawing) return;

        this.isDrawing = false;
        this.saveToHistory();
      }

      onTouchStart(e) {
        e.preventDefault();
        const touch = e.touches[0];
        const mouseEvent = new MouseEvent('mousedown', {
          clientX: touch.clientX,
          clientY: touch.clientY
        });
        this.canvas.dispatchEvent(mouseEvent);
      }

      onTouchMove(e) {
        e.preventDefault();
        const touch = e.touches[0];
        const mouseEvent = new MouseEvent('mousemove', {
          clientX: touch.clientX,
          clientY: touch.clientY
        });
        this.canvas.dispatchEvent(mouseEvent);
      }

      onTouchEnd(e) {
        e.preventDefault();
        const mouseEvent = new MouseEvent('mouseup', {});
        this.canvas.dispatchEvent(mouseEvent);
      }

      onKeyDown(e) {
        // Ctrl+Z: 撤销
        if (e.ctrlKey && e.key === 'z') {
          e.preventDefault();
          this.undo();
        }
        // Ctrl+Y: 重做
        if (e.ctrlKey && e.key === 'y') {
          e.preventDefault();
          this.redo();
        }
      }

      beginPath() {
        this.ctx.beginPath();
        this.ctx.strokeStyle = this.strokeColor;
        this.ctx.lineWidth = this.lineWidth;
        this.ctx.lineCap = 'round';
        this.ctx.lineJoin = 'round';
        this.ctx.globalAlpha = this.opacity;
      }

      drawPen(point) {
        const ctx = this.ctx;
        ctx.lineTo(point.x, point.y);
        ctx.stroke();
      }

      drawLine(endPoint) {
        this.restoreCanvasState();
        const ctx = this.ctx;

        ctx.beginPath();
        ctx.strokeStyle = this.strokeColor;
        ctx.lineWidth = this.lineWidth;
        ctx.globalAlpha = this.opacity;
        ctx.moveTo(this.startPoint.x, this.startPoint.y);
        ctx.lineTo(endPoint.x, endPoint.y);
        ctx.stroke();
      }

      drawRectangle(endPoint) {
        this.restoreCanvasState();
        const ctx = this.ctx;

        const width = endPoint.x - this.startPoint.x;
        const height = endPoint.y - this.startPoint.y;

        ctx.beginPath();
        ctx.strokeStyle = this.strokeColor;
        ctx.fillStyle = this.fillColor;
        ctx.lineWidth = this.lineWidth;
        ctx.globalAlpha = this.opacity;
        ctx.rect(this.startPoint.x, this.startPoint.y, width, height);

        if (this.fillShape) {
          ctx.fill();
        }
        ctx.stroke();
      }

      drawCircle(endPoint) {
        this.restoreCanvasState();
        const ctx = this.ctx;

        const dx = endPoint.x - this.startPoint.x;
        const dy = endPoint.y - this.startPoint.y;
        const radius = Math.sqrt(dx * dx + dy * dy);

        ctx.beginPath();
        ctx.strokeStyle = this.strokeColor;
        ctx.fillStyle = this.fillColor;
        ctx.lineWidth = this.lineWidth;
        ctx.globalAlpha = this.opacity;
        ctx.arc(this.startPoint.x, this.startPoint.y, radius, 0, Math.PI * 2);

        if (this.fillShape) {
          ctx.fill();
        }
        ctx.stroke();
      }

      drawEllipse(endPoint) {
        this.restoreCanvasState();
        const ctx = this.ctx;

        const radiusX = Math.abs(endPoint.x - this.startPoint.x) / 2;
        const radiusY = Math.abs(endPoint.y - this.startPoint.y) / 2;
        const centerX = (this.startPoint.x + endPoint.x) / 2;
        const centerY = (this.startPoint.y + endPoint.y) / 2;

        ctx.beginPath();
        ctx.strokeStyle = this.strokeColor;
        ctx.fillStyle = this.fillColor;
        ctx.lineWidth = this.lineWidth;
        ctx.globalAlpha = this.opacity;

        if (ctx.ellipse) {
          ctx.ellipse(centerX, centerY, radiusX, radiusY, 0, 0, Math.PI * 2);
        } else {
          // Fallback for older browsers
          this.drawEllipseWithBezier(ctx, centerX, centerY, radiusX, radiusY);
        }

        if (this.fillShape) {
          ctx.fill();
        }
        ctx.stroke();
      }

      drawEllipseWithBezier(ctx, cx, cy, rx, ry) {
        const kappa = 0.5522848;
        const ox = rx * kappa;
        const oy = ry * kappa;

        ctx.moveTo(cx - rx, cy);
        ctx.bezierCurveTo(cx - rx, cy - oy, cx - ox, cy - ry, cx, cy - ry);
        ctx.bezierCurveTo(cx + ox, cy - ry, cx + rx, cy - oy, cx + rx, cy);
        ctx.bezierCurveTo(cx + rx, cy + oy, cx + ox, cy + ry, cx, cy + ry);
        ctx.bezierCurveTo(cx - ox, cy + ry, cx - rx, cy + oy, cx - rx, cy);
        ctx.closePath();
      }

      drawTriangle(endPoint) {
        this.restoreCanvasState();
        const ctx = this.ctx;

        ctx.beginPath();
        ctx.strokeStyle = this.strokeColor;
        ctx.fillStyle = this.fillColor;
        ctx.lineWidth = this.lineWidth;
        ctx.globalAlpha = this.opacity;

        // 计算三角形的三个顶点
        const centerX = (this.startPoint.x + endPoint.x) / 2;
        ctx.moveTo(centerX, this.startPoint.y);
        ctx.lineTo(endPoint.x, endPoint.y);
        ctx.lineTo(this.startPoint.x, endPoint.y);
        ctx.closePath();

        if (this.fillShape) {
          ctx.fill();
        }
        ctx.stroke();
      }

      saveCanvasState() {
        this.canvasState = this.ctx.getImageData(
          0, 0,
          this.canvas.width,
          this.canvas.height
        );
      }

      restoreCanvasState() {
        if (this.canvasState) {
          this.ctx.putImageData(this.canvasState, 0, 0);
        }
      }

      clearCanvas() {
        this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
        this.saveToHistory();
      }

      saveToHistory() {
        const imageData = this.ctx.getImageData(
          0, 0,
          this.canvas.width,
          this.canvas.height
        );

        // 删除当前索引之后的历史记录
        this.history = this.history.slice(0, this.historyIndex + 1);

        // 添加新的历史记录
        this.history.push(imageData);
        this.historyIndex++;

        // 限制历史记录数量
        if (this.history.length > 50) {
          this.history.shift();
          this.historyIndex--;
        }
      }

      restoreFromHistory() {
        if (this.historyIndex >= 0 && this.historyIndex < this.history.length) {
          this.ctx.putImageData(this.history[this.historyIndex], 0, 0);
        }
      }

      undo() {
        if (this.historyIndex > 0) {
          this.historyIndex--;
          this.restoreFromHistory();
        }
      }

      redo() {
        if (this.historyIndex < this.history.length - 1) {
          this.historyIndex++;
          this.restoreFromHistory();
        }
      }
    }

    // 初始化绘图应用
    const app = new DrawingApp('drawingCanvas');
  </script>
</body>
</html>
```

## 6. Excalidraw 中的应用

### 6.1 Excalidraw 的图形绘制系统

```typescript
// packages/excalidraw/renderer/renderElement.ts
export const renderElement = (
  element: ExcalidrawElement,
  rc: RoughCanvas,
  context: CanvasRenderingContext2D,
  renderConfig: RenderConfig
) => {
  switch (element.type) {
    case "rectangle":
      renderRectangle(element, rc, context, renderConfig);
      break;
    case "ellipse":
      renderEllipse(element, rc, context, renderConfig);
      break;
    case "arrow":
      renderArrow(element, rc, context, renderConfig);
      break;
    case "line":
      renderLine(element, rc, context, renderConfig);
      break;
    case "text":
      renderText(element, context, renderConfig);
      break;
  }
};

// 矩形绘制实现
const renderRectangle = (
  element: ExcalidrawRectangleElement,
  rc: RoughCanvas,
  context: CanvasRenderingContext2D,
  renderConfig: RenderConfig
) => {
  const { x, y, width, height, strokeColor, backgroundColor, fillStyle, roughness } = element;

  if (renderConfig.isExporting || !renderConfig.isRough) {
    // 导出模式或非手绘模式：使用标准 Canvas API
    context.strokeStyle = strokeColor;
    context.fillStyle = backgroundColor;
    context.lineWidth = element.strokeWidth;

    if (backgroundColor !== "transparent") {
      context.fillRect(x, y, width, height);
    }
    context.strokeRect(x, y, width, height);
  } else {
    // 手绘模式：使用 RoughJS
    rc.rectangle(x, y, width, height, {
      stroke: strokeColor,
      fill: backgroundColor,
      fillStyle: fillStyle,
      strokeWidth: element.strokeWidth,
      roughness: roughness,
      seed: element.seed
    });
  }
};
```

## 7. 练习题

### 7.1 基础练习

1. **实现图形绘制函数库**
   - 创建一个包含各种图形绘制的工具库
   - 支持样式配置
   - 支持链式调用

2. **贝塞尔曲线编辑器**
   - 可视化控制点
   - 实时预览曲线
   - 支持导出路径数据

3. **渐变生成器**
   - 支持线性、径向、锥形渐变
   - 颜色停止点编辑
   - 实时预览

### 7.2 进阶练习

实现一个图形样式管理器：

```javascript
class StyleManager {
  constructor() {
    this.styles = new Map();
    this.currentStyle = 'default';
  }

  // 创建样式预设
  createPreset(name, config) {
    // TODO: 实现样式预设
  }

  // 应用样式
  applyStyle(ctx, styleName) {
    // TODO: 应用指定样式
  }

  // 样式动画
  animateStyle(ctx, from, to, progress) {
    // TODO: 实现样式过渡动画
  }
}
```

## 8. 思考题

1. **为什么路径的方向很重要？**
   - 在填充规则中的作用
   - 对镂空效果的影响

2. **如何优化大量图形的绘制性能？**
   - 批量绘制策略
   - 图形缓存技术
   - 离屏渲染应用

3. **贝塞尔曲线的数学原理是什么？**
   - 参数方程推导
   - 控制点的作用
   - 曲线拟合算法

4. **如何实现平滑的手绘效果？**
   - 随机扰动算法
   - 多次绘制叠加
   - 笔触模拟

## 9. 总结

### 核心要点

1. **路径是基础**
   - 理解路径的概念和子路径
   - 掌握路径的方向性
   - 熟练使用路径 API

2. **图形绘制**
   - 基本图形的多种实现方式
   - 贝塞尔曲线的应用
   - 手绘效果的实现技巧

3. **样式系统**
   - 颜色和透明度控制
   - 渐变和图案应用
   - 阴影和特效

4. **实战技巧**
   - 工具系统设计
   - 历史记录实现
   - 性能优化策略

### 下一步

掌握了绘图基础后，下一章将学习：
- Canvas 变换系统
- 矩阵运算
- 图像合成
- 高级特效

## 10. 参考资源

- [MDN Canvas Tutorial](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API/Tutorial)
- [RoughJS Library](https://roughjs.com/)
- [Bezier Curve Primer](https://pomax.github.io/bezierinfo/)
- [Canvas Handbook](https://www.html5rocks.com/en/tutorials/canvas/performance/)

---

**上一章**：[Canvas 基础概念与 API](./01-canvas-basics.md)
**下一章**：[Canvas 变换与合成 →](./03-canvas-transform.md)