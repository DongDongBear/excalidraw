# 第三章：Canvas 变换与合成

## 学习目标

- [ ] 理解变换矩阵的原理和数学基础
- [ ] 掌握平移、旋转、缩放等基本变换
- [ ] 熟练使用 save() 和 restore() 进行状态管理
- [ ] 理解图像合成模式和应用场景
- [ ] 实现复杂的图形变换和动画效果

## 1. 变换矩阵原理

### 1.1 什么是变换矩阵？

变换矩阵是计算机图形学中用于描述坐标变换的数学工具。在 2D Canvas 中，我们使用 3x3 矩阵来表示仿射变换。

```javascript
// 2D 仿射变换矩阵
// [a c e]   [x]   [ax + cy + e]
// [b d f] × [y] = [bx + dy + f]
// [0 0 1]   [1]   [1]

// Canvas 变换矩阵参数：transform(a, b, c, d, e, f)
// a: 水平缩放
// b: 水平倾斜
// c: 垂直倾斜
// d: 垂直缩放
// e: 水平平移
// f: 垂直平移
```

### 1.2 变换矩阵基础

```javascript
class TransformMatrix {
  constructor() {
    // 单位矩阵（恒等变换）
    this.matrix = [
      1, 0, 0,  // [1 0 0]
      0, 1, 0,  // [0 1 0]
      0, 0, 1   // [0 0 1]
    ];
  }

  // 从 Canvas 变换参数创建矩阵
  static fromTransform(a, b, c, d, e, f) {
    return [
      a, c, e,
      b, d, f,
      0, 0, 1
    ];
  }

  // 矩阵乘法
  multiply(other) {
    const [a1, c1, e1, b1, d1, f1] = this.matrix;
    const [a2, c2, e2, b2, d2, f2] = other;

    return [
      a1 * a2 + c1 * b2,     // a
      b1 * a2 + d1 * b2,     // b
      a1 * c2 + c1 * d2,     // c
      b1 * c2 + d1 * d2,     // d
      a1 * e2 + c1 * f2 + e1, // e
      b1 * e2 + d1 * f2 + f1  // f
    ];
  }

  // 应用变换到点
  transformPoint(x, y) {
    const [a, c, e, b, d, f] = this.matrix;
    return {
      x: a * x + c * y + e,
      y: b * x + d * y + f
    };
  }

  // 获取 Canvas transform 参数
  toCanvasTransform() {
    const [a, c, e, b, d, f] = this.matrix;
    return [a, b, c, d, e, f];
  }
}
```

### 1.3 基本变换的矩阵表示

```javascript
class BasicTransforms {
  // 平移变换
  static translate(tx, ty) {
    return [
      1, 0, tx,
      0, 1, ty,
      0, 0, 1
    ];
  }

  // 缩放变换
  static scale(sx, sy = sx) {
    return [
      sx, 0,  0,
      0,  sy, 0,
      0,  0,  1
    ];
  }

  // 旋转变换
  static rotate(angle) {
    const cos = Math.cos(angle);
    const sin = Math.sin(angle);
    return [
      cos, -sin, 0,
      sin,  cos, 0,
      0,    0,   1
    ];
  }

  // 倾斜变换
  static skew(skewX, skewY) {
    return [
      1, Math.tan(skewY), 0,
      Math.tan(skewX), 1, 0,
      0, 0, 1
    ];
  }

  // 反射变换（水平翻转）
  static flipHorizontal() {
    return [
      -1, 0, 0,
       0, 1, 0,
       0, 0, 1
    ];
  }

  // 反射变换（垂直翻转）
  static flipVertical() {
    return [
      1,  0, 0,
      0, -1, 0,
      0,  0, 1
    ];
  }
}
```

## 2. Canvas 变换 API

### 2.1 基本变换方法

```javascript
class CanvasTransformDemo {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 平移演示
  demonstrateTranslate() {
    const ctx = this.ctx;

    // 绘制原始位置
    ctx.fillStyle = 'lightgray';
    ctx.fillRect(50, 50, 100, 60);

    // 平移后绘制
    ctx.translate(100, 50); // 向右平移100px，向下平移50px
    ctx.fillStyle = 'blue';
    ctx.fillRect(50, 50, 100, 60);

    // 重置变换
    ctx.resetTransform();
  }

  // 旋转演示
  demonstrateRotate() {
    const ctx = this.ctx;

    // 保存状态
    ctx.save();

    // 移到旋转中心
    ctx.translate(150, 150);

    // 绘制多个旋转的矩形
    for (let i = 0; i < 8; i++) {
      ctx.rotate(Math.PI / 4); // 旋转45度
      ctx.fillStyle = `hsl(${i * 45}, 70%, 60%)`;
      ctx.fillRect(0, -10, 80, 20);
    }

    // 恢复状态
    ctx.restore();
  }

  // 缩放演示
  demonstrateScale() {
    const ctx = this.ctx;

    ctx.save();
    ctx.translate(100, 100);

    // 绘制原始大小
    ctx.strokeStyle = 'gray';
    ctx.strokeRect(-25, -25, 50, 50);

    // 不同比例缩放
    const scales = [0.5, 1.5, 2.0];
    const colors = ['red', 'green', 'blue'];

    scales.forEach((scale, i) => {
      ctx.save();
      ctx.scale(scale, scale);
      ctx.fillStyle = colors[i];
      ctx.globalAlpha = 0.5;
      ctx.fillRect(-25, -25, 50, 50);
      ctx.restore();
    });

    ctx.restore();
  }

  // 复合变换演示
  demonstrateCompositeTransform() {
    const ctx = this.ctx;

    ctx.save();

    // 移动到中心
    ctx.translate(200, 200);

    // 旋转
    ctx.rotate(Math.PI / 6);

    // 缩放
    ctx.scale(1.5, 0.8);

    // 绘制变换后的图形
    ctx.fillStyle = 'purple';
    ctx.fillRect(-50, -30, 100, 60);

    // 绘制坐标轴（帮助理解变换）
    ctx.strokeStyle = 'red';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(-100, 0);
    ctx.lineTo(100, 0);
    ctx.moveTo(0, -60);
    ctx.lineTo(0, 60);
    ctx.stroke();

    ctx.restore();
  }
}
```

### 2.2 直接矩阵操作

```javascript
class MatrixTransformDemo {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 使用 transform() 方法
  useTransformMethod() {
    const ctx = this.ctx;

    // transform(a, b, c, d, e, f)
    // 等价于当前矩阵 × 新矩阵
    ctx.transform(1.2, 0.2, -0.1, 1.3, 50, 30);

    ctx.fillStyle = 'orange';
    ctx.fillRect(0, 0, 100, 100);
  }

  // 使用 setTransform() 方法
  useSetTransformMethod() {
    const ctx = this.ctx;

    // setTransform() 直接设置变换矩阵（替换当前矩阵）
    ctx.setTransform(1, 0.5, -0.5, 1, 100, 100);

    ctx.fillStyle = 'green';
    ctx.fillRect(0, 0, 50, 50);

    // 重置为单位矩阵
    ctx.setTransform(1, 0, 0, 1, 0, 0);
    // 或者使用 resetTransform() (较新的 API)
    // ctx.resetTransform();
  }

  // 自定义变换函数
  applyCustomTransform(matrix) {
    const ctx = this.ctx;
    const [a, b, c, d, e, f] = matrix;
    ctx.setTransform(a, b, c, d, e, f);
  }

  // 3D 透视效果（使用 2D 变换模拟）
  create3DPerspective() {
    const ctx = this.ctx;

    // 保存状态
    ctx.save();

    // 透视变换参数
    const perspective = 500; // 透视距离
    const rotationY = Math.PI / 6; // Y轴旋转角度

    // 计算透视变换矩阵
    const cos = Math.cos(rotationY);
    const sin = Math.sin(rotationY);

    // 简化的透视投影
    const scaleX = cos;
    const skewX = -sin * 0.5;

    ctx.translate(200, 200);
    ctx.transform(scaleX, 0, skewX, 1, 0, 0);

    // 绘制"3D"立方体
    this.draw3DCube();

    ctx.restore();
  }

  draw3DCube() {
    const ctx = this.ctx;
    const size = 80;

    // 前面
    ctx.fillStyle = 'rgba(255, 0, 0, 0.8)';
    ctx.fillRect(-size/2, -size/2, size, size);

    // 顶面
    ctx.fillStyle = 'rgba(255, 100, 100, 0.8)';
    ctx.beginPath();
    ctx.moveTo(-size/2, -size/2);
    ctx.lineTo(-size/2 + 20, -size/2 - 20);
    ctx.lineTo(size/2 + 20, -size/2 - 20);
    ctx.lineTo(size/2, -size/2);
    ctx.closePath();
    ctx.fill();

    // 右面
    ctx.fillStyle = 'rgba(200, 0, 0, 0.8)';
    ctx.beginPath();
    ctx.moveTo(size/2, -size/2);
    ctx.lineTo(size/2 + 20, -size/2 - 20);
    ctx.lineTo(size/2 + 20, size/2 - 20);
    ctx.lineTo(size/2, size/2);
    ctx.closePath();
    ctx.fill();
  }
}
```

## 3. 状态管理（save & restore）

### 3.1 状态栈的概念

Canvas 维护一个状态栈，`save()` 将当前状态推入栈，`restore()` 从栈中弹出状态。

```javascript
class StateManagementDemo {
  constructor(ctx) {
    this.ctx = ctx;
    this.stateStack = [];
  }

  // 基本状态管理
  basicStateManagement() {
    const ctx = this.ctx;

    // 初始状态
    ctx.fillStyle = 'black';
    ctx.lineWidth = 1;

    // 保存状态 1
    ctx.save();

    // 修改状态
    ctx.fillStyle = 'red';
    ctx.lineWidth = 5;
    ctx.translate(50, 50);

    // 绘制
    ctx.fillRect(0, 0, 50, 50);

    // 保存状态 2
    ctx.save();

    // 继续修改状态
    ctx.fillStyle = 'blue';
    ctx.rotate(Math.PI / 4);
    ctx.scale(1.5, 1.5);

    // 绘制
    ctx.fillRect(0, 0, 30, 30);

    // 恢复状态 2
    ctx.restore();

    // 此时回到红色、线宽5、平移(50,50)的状态
    ctx.fillRect(60, 0, 30, 30);

    // 恢复状态 1
    ctx.restore();

    // 此时回到初始状态
    ctx.fillRect(0, 0, 30, 30);
  }

  // 嵌套变换
  nestedTransforms() {
    const ctx = this.ctx;

    // 绘制层级结构（太阳系模拟）
    this.drawSolarSystem();
  }

  drawSolarSystem() {
    const ctx = this.ctx;
    const time = Date.now() / 1000;

    ctx.save();
    ctx.translate(200, 200);

    // 太阳
    ctx.fillStyle = 'yellow';
    ctx.beginPath();
    ctx.arc(0, 0, 30, 0, Math.PI * 2);
    ctx.fill();

    // 地球轨道
    ctx.save();
    ctx.rotate(time * 0.5); // 地球公转
    ctx.translate(100, 0);

    // 地球
    ctx.fillStyle = 'blue';
    ctx.beginPath();
    ctx.arc(0, 0, 15, 0, Math.PI * 2);
    ctx.fill();

    // 月球轨道
    ctx.save();
    ctx.rotate(time * 2); // 月球绕地球
    ctx.translate(30, 0);

    // 月球
    ctx.fillStyle = 'gray';
    ctx.beginPath();
    ctx.arc(0, 0, 5, 0, Math.PI * 2);
    ctx.fill();

    ctx.restore(); // 恢复到地球坐标系
    ctx.restore(); // 恢复到太阳坐标系
    ctx.restore(); // 恢复到画布坐标系
  }

  // 状态管理最佳实践
  stateManagementBestPractices() {
    const ctx = this.ctx;

    // 1. 使用函数封装状态管理
    const drawWithState = (drawFn, setupFn) => {
      ctx.save();
      if (setupFn) setupFn(ctx);
      drawFn(ctx);
      ctx.restore();
    };

    // 使用示例
    drawWithState(
      (ctx) => ctx.fillRect(0, 0, 50, 50),
      (ctx) => {
        ctx.fillStyle = 'red';
        ctx.translate(100, 100);
        ctx.rotate(Math.PI / 4);
      }
    );

    // 2. 状态对象封装
    class CanvasState {
      constructor(ctx) {
        this.ctx = ctx;
        this.savedStates = [];
      }

      save() {
        this.ctx.save();
        this.savedStates.push(this.getCurrentState());
        return this;
      }

      restore() {
        if (this.savedStates.length > 0) {
          this.ctx.restore();
          this.savedStates.pop();
        }
        return this;
      }

      getCurrentState() {
        return {
          fillStyle: this.ctx.fillStyle,
          strokeStyle: this.ctx.strokeStyle,
          lineWidth: this.ctx.lineWidth,
          globalAlpha: this.ctx.globalAlpha,
          // ... 其他属性
        };
      }
    }
  }
}
```

### 3.2 状态管理的性能考虑

```javascript
class OptimizedStateManagement {
  constructor(ctx) {
    this.ctx = ctx;
    this.stateCache = new Map();
  }

  // 状态缓存
  cacheState(key) {
    const ctx = this.ctx;
    this.stateCache.set(key, {
      transform: ctx.getTransform(),
      fillStyle: ctx.fillStyle,
      strokeStyle: ctx.strokeStyle,
      lineWidth: ctx.lineWidth,
      globalAlpha: ctx.globalAlpha,
      globalCompositeOperation: ctx.globalCompositeOperation,
      shadowColor: ctx.shadowColor,
      shadowBlur: ctx.shadowBlur,
      shadowOffsetX: ctx.shadowOffsetX,
      shadowOffsetY: ctx.shadowOffsetY
    });
  }

  // 恢复缓存的状态
  restoreState(key) {
    const state = this.stateCache.get(key);
    if (!state) return;

    const ctx = this.ctx;
    ctx.setTransform(state.transform);
    ctx.fillStyle = state.fillStyle;
    ctx.strokeStyle = state.strokeStyle;
    ctx.lineWidth = state.lineWidth;
    ctx.globalAlpha = state.globalAlpha;
    ctx.globalCompositeOperation = state.globalCompositeOperation;
    ctx.shadowColor = state.shadowColor;
    ctx.shadowBlur = state.shadowBlur;
    ctx.shadowOffsetX = state.shadowOffsetX;
    ctx.shadowOffsetY = state.shadowOffsetY;
  }

  // 批量状态更改
  batchStateChange(changes) {
    const ctx = this.ctx;
    Object.entries(changes).forEach(([key, value]) => {
      if (ctx[key] !== undefined) {
        ctx[key] = value;
      }
    });
  }

  // 智能状态管理：只有在需要时才保存/恢复
  smartStateManagement(drawFn, requiredChanges) {
    const ctx = this.ctx;
    let needsSave = false;

    // 检查是否需要保存状态
    for (const [key, value] of Object.entries(requiredChanges)) {
      if (ctx[key] !== value) {
        needsSave = true;
        break;
      }
    }

    if (needsSave) {
      ctx.save();
      this.batchStateChange(requiredChanges);
      drawFn();
      ctx.restore();
    } else {
      drawFn();
    }
  }
}
```

## 4. 图像合成

### 4.1 合成操作模式

```javascript
class CompositeOperationsDemo {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 演示所有合成模式
  demonstrateAllCompositeOperations() {
    const operations = [
      'source-over',      // 默认：新图形在旧图形上
      'source-in',        // 新图形只在与旧图形重叠的地方绘制
      'source-out',       // 新图形只在与旧图形不重叠的地方绘制
      'source-atop',      // 新图形只在旧图形顶部绘制

      'destination-over', // 旧图形在新图形上
      'destination-in',   // 旧图形只保留与新图形重叠的部分
      'destination-out',  // 旧图形只保留与新图形不重叠的部分
      'destination-atop', // 旧图形只在新图形顶部

      'lighter',          // 颜色相加
      'copy',             // 只显示新图形
      'xor',              // 只显示不重叠的部分

      'multiply',         // 颜色相乘
      'screen',           // 反相相乘后再反相
      'overlay',          // 综合 multiply 和 screen
      'darken',           // 保留较暗的颜色
      'lighten',          // 保留较亮的颜色
      'color-dodge',      // 颜色减淡
      'color-burn',       // 颜色加深
      'hard-light',       // 强光
      'soft-light',       // 柔光
      'difference',       // 差值
      'exclusion',        // 排除
      'hue',              // 色相
      'saturation',       // 饱和度
      'color',            // 颜色
      'luminosity'        // 明度
    ];

    const gridCols = 5;
    const cellSize = 120;

    operations.forEach((operation, index) => {
      const x = (index % gridCols) * cellSize + 10;
      const y = Math.floor(index / gridCols) * cellSize + 10;

      this.drawCompositeExample(x, y, operation);
    });
  }

  drawCompositeExample(x, y, operation) {
    const ctx = this.ctx;

    ctx.save();
    ctx.translate(x, y);

    // 绘制标题
    ctx.fillStyle = 'black';
    ctx.font = '10px Arial';
    ctx.fillText(operation, 5, 15);

    // 创建一个小的演示区域
    ctx.save();
    ctx.translate(0, 20);

    // 先绘制红色圆形
    ctx.fillStyle = 'red';
    ctx.beginPath();
    ctx.arc(30, 30, 20, 0, Math.PI * 2);
    ctx.fill();

    // 设置合成操作
    ctx.globalCompositeOperation = operation;

    // 再绘制蓝色矩形
    ctx.fillStyle = 'blue';
    ctx.fillRect(20, 20, 30, 30);

    ctx.restore();
    ctx.restore();
  }

  // 实际应用示例
  practicalCompositeExamples() {
    const ctx = this.ctx;

    // 1. 创建遮罩效果
    this.createMaskEffect();

    // 2. 创建发光效果
    this.createGlowEffect();

    // 3. 创建擦除效果
    this.createEraseEffect();
  }

  createMaskEffect() {
    const ctx = this.ctx;

    ctx.save();
    ctx.translate(50, 50);

    // 绘制背景图片（用渐变模拟）
    const gradient = ctx.createLinearGradient(0, 0, 200, 200);
    gradient.addColorStop(0, 'red');
    gradient.addColorStop(0.5, 'yellow');
    gradient.addColorStop(1, 'blue');
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, 200, 200);

    // 设置合成模式为 destination-in（遮罩）
    ctx.globalCompositeOperation = 'destination-in';

    // 绘制遮罩形状（圆形）
    ctx.fillStyle = 'black';
    ctx.beginPath();
    ctx.arc(100, 100, 80, 0, Math.PI * 2);
    ctx.fill();

    ctx.restore();
  }

  createGlowEffect() {
    const ctx = this.ctx;

    ctx.save();
    ctx.translate(300, 50);

    // 先绘制文字
    ctx.fillStyle = 'white';
    ctx.font = 'bold 40px Arial';
    ctx.textAlign = 'center';
    ctx.fillText('GLOW', 100, 120);

    // 使用 lighter 模式创建发光效果
    ctx.globalCompositeOperation = 'lighter';

    // 绘制多层发光
    for (let i = 1; i <= 10; i++) {
      ctx.shadowColor = 'cyan';
      ctx.shadowBlur = i * 3;
      ctx.fillStyle = `rgba(0, 255, 255, ${0.1})`;
      ctx.fillText('GLOW', 100, 120);
    }

    ctx.restore();
  }

  createEraseEffect() {
    const ctx = this.ctx;

    ctx.save();
    ctx.translate(50, 300);

    // 绘制背景
    ctx.fillStyle = 'purple';
    ctx.fillRect(0, 0, 200, 100);

    // 模拟擦除效果
    ctx.globalCompositeOperation = 'destination-out';

    // 绘制"擦除"的路径
    ctx.beginPath();
    ctx.arc(50, 50, 30, 0, Math.PI * 2);
    ctx.arc(150, 50, 25, 0, Math.PI * 2);
    ctx.fill();

    ctx.restore();
  }
}
```

### 4.2 透明度和混合

```javascript
class AlphaAndBlendingDemo {
  constructor(ctx) {
    this.ctx = ctx;
  }

  // 透明度演示
  demonstrateAlpha() {
    const ctx = this.ctx;

    ctx.save();
    ctx.translate(50, 50);

    // 1. globalAlpha 影响所有后续绘制
    const alphaValues = [1.0, 0.8, 0.6, 0.4, 0.2];
    const colors = ['red', 'green', 'blue', 'orange', 'purple'];

    alphaValues.forEach((alpha, i) => {
      ctx.save();
      ctx.globalAlpha = alpha;
      ctx.fillStyle = colors[i];
      ctx.fillRect(i * 30, 0, 40, 100);
      ctx.restore();
    });

    ctx.restore();

    // 2. RGBA 颜色直接设置透明度
    ctx.save();
    ctx.translate(50, 200);

    for (let i = 0; i < 5; i++) {
      const alpha = (5 - i) / 5;
      ctx.fillStyle = `rgba(255, 0, 0, ${alpha})`;
      ctx.fillRect(i * 30, 0, 40, 100);
    }

    ctx.restore();
  }

  // 混合模式实际应用
  practicalBlendingExamples() {
    // 图片混合效果
    this.createImageBlendEffect();

    // 渐变蒙版
    this.createGradientMask();

    // 彩虹效果
    this.createRainbowEffect();
  }

  createImageBlendEffect() {
    const ctx = this.ctx;

    ctx.save();
    ctx.translate(300, 50);

    // 创建两个图像（用渐变模拟）
    // 第一层
    const gradient1 = ctx.createRadialGradient(50, 50, 0, 50, 50, 50);
    gradient1.addColorStop(0, 'red');
    gradient1.addColorStop(1, 'transparent');
    ctx.fillStyle = gradient1;
    ctx.fillRect(0, 0, 100, 100);

    // 混合模式
    ctx.globalCompositeOperation = 'multiply';

    // 第二层
    const gradient2 = ctx.createRadialGradient(70, 70, 0, 70, 70, 50);
    gradient2.addColorStop(0, 'blue');
    gradient2.addColorStop(1, 'transparent');
    ctx.fillStyle = gradient2;
    ctx.fillRect(20, 20, 100, 100);

    ctx.restore();
  }

  createGradientMask() {
    const ctx = this.ctx;

    ctx.save();
    ctx.translate(450, 50);

    // 绘制背景内容
    ctx.fillStyle = 'orange';
    ctx.fillRect(0, 0, 150, 100);

    ctx.fillStyle = 'darkblue';
    ctx.font = '24px Arial';
    ctx.fillText('MASKED', 10, 50);

    // 应用渐变蒙版
    ctx.globalCompositeOperation = 'destination-in';

    const gradient = ctx.createLinearGradient(0, 0, 150, 0);
    gradient.addColorStop(0, 'black');
    gradient.addColorStop(0.5, 'transparent');
    gradient.addColorStop(1, 'black');
    ctx.fillStyle = gradient;
    ctx.fillRect(0, 0, 150, 100);

    ctx.restore();
  }

  createRainbowEffect() {
    const ctx = this.ctx;

    ctx.save();
    ctx.translate(300, 200);
    ctx.globalCompositeOperation = 'lighter';

    const colors = [
      'red', 'orange', 'yellow', 'green', 'blue', 'indigo', 'violet'
    ];

    colors.forEach((color, i) => {
      ctx.save();
      ctx.fillStyle = color;
      ctx.globalAlpha = 0.7;
      ctx.beginPath();
      ctx.arc(75 + i * 10, 50, 30, 0, Math.PI * 2);
      ctx.fill();
      ctx.restore();
    });

    ctx.restore();
  }
}
```

## 5. 实战：图形变换工具

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Canvas 变换工具</title>
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

    .controls {
      width: 300px;
      background: white;
      padding: 20px;
      box-shadow: 2px 0 5px rgba(0,0,0,0.1);
      overflow-y: auto;
    }

    .control-group {
      margin-bottom: 20px;
      padding: 15px;
      border: 1px solid #ddd;
      border-radius: 8px;
    }

    .control-group h3 {
      margin-bottom: 15px;
      color: #333;
      font-size: 14px;
      font-weight: 600;
    }

    .control {
      margin-bottom: 12px;
    }

    .control label {
      display: block;
      font-size: 12px;
      color: #666;
      margin-bottom: 4px;
    }

    .control input[type="range"] {
      width: 100%;
      margin-bottom: 4px;
    }

    .control .value {
      font-size: 11px;
      color: #999;
      text-align: right;
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
      border: 1px solid #ddd;
    }

    button {
      width: 100%;
      padding: 10px;
      background: #3498db;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 14px;
      margin-bottom: 10px;
    }

    button:hover {
      background: #2980b9;
    }

    .preset-buttons {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 10px;
    }

    .preset-buttons button {
      margin: 0;
      padding: 8px;
      font-size: 12px;
    }
  </style>
</head>
<body>
  <div class="controls">
    <div class="control-group">
      <h3>🔄 基础变换</h3>

      <div class="control">
        <label>平移 X</label>
        <input type="range" id="translateX" min="-200" max="200" value="0">
        <div class="value" id="translateXValue">0</div>
      </div>

      <div class="control">
        <label>平移 Y</label>
        <input type="range" id="translateY" min="-200" max="200" value="0">
        <div class="value" id="translateYValue">0</div>
      </div>

      <div class="control">
        <label>旋转角度</label>
        <input type="range" id="rotation" min="0" max="360" value="0">
        <div class="value" id="rotationValue">0°</div>
      </div>

      <div class="control">
        <label>缩放 X</label>
        <input type="range" id="scaleX" min="0.1" max="3" step="0.1" value="1">
        <div class="value" id="scaleXValue">1.0</div>
      </div>

      <div class="control">
        <label>缩放 Y</label>
        <input type="range" id="scaleY" min="0.1" max="3" step="0.1" value="1">
        <div class="value" id="scaleYValue">1.0</div>
      </div>
    </div>

    <div class="control-group">
      <h3>🎭 高级变换</h3>

      <div class="control">
        <label>倾斜 X</label>
        <input type="range" id="skewX" min="-45" max="45" value="0">
        <div class="value" id="skewXValue">0°</div>
      </div>

      <div class="control">
        <label>倾斜 Y</label>
        <input type="range" id="skewY" min="-45" max="45" value="0">
        <div class="value" id="skewYValue">0°</div>
      </div>

      <div class="control">
        <label>
          <input type="checkbox" id="flipH"> 水平翻转
        </label>
      </div>

      <div class="control">
        <label>
          <input type="checkbox" id="flipV"> 垂直翻转
        </label>
      </div>
    </div>

    <div class="control-group">
      <h3>🎨 混合模式</h3>

      <div class="control">
        <label>合成操作</label>
        <select id="compositeOperation">
          <option value="source-over">source-over</option>
          <option value="multiply">multiply</option>
          <option value="screen">screen</option>
          <option value="overlay">overlay</option>
          <option value="darken">darken</option>
          <option value="lighten">lighten</option>
          <option value="difference">difference</option>
          <option value="exclusion">exclusion</option>
          <option value="lighter">lighter</option>
        </select>
      </div>

      <div class="control">
        <label>透明度</label>
        <input type="range" id="alpha" min="0" max="100" value="100">
        <div class="value" id="alphaValue">100%</div>
      </div>
    </div>

    <div class="control-group">
      <h3>⚡ 预设</h3>
      <div class="preset-buttons">
        <button onclick="loadPreset('identity')">重置</button>
        <button onclick="loadPreset('rotate45')">旋转45°</button>
        <button onclick="loadPreset('scale2x')">放大2倍</button>
        <button onclick="loadPreset('flipH')">水平翻转</button>
        <button onclick="loadPreset('3d')">3D效果</button>
        <button onclick="loadPreset('crazy')">疯狂模式</button>
      </div>
    </div>

    <div class="control-group">
      <h3>📊 矩阵信息</h3>
      <textarea id="matrixInfo" readonly rows="6" style="width: 100%; font-family: monospace; font-size: 11px; resize: none;"></textarea>
    </div>
  </div>

  <div class="canvas-container">
    <canvas id="transformCanvas" width="600" height="600"></canvas>
  </div>

  <script>
    class TransformTool {
      constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = this.canvas.getContext('2d');

        this.transforms = {
          translateX: 0,
          translateY: 0,
          rotation: 0,
          scaleX: 1,
          scaleY: 1,
          skewX: 0,
          skewY: 0,
          flipH: false,
          flipV: false
        };

        this.compositeOperation = 'source-over';
        this.alpha = 1;

        this.init();
        this.setupEventListeners();
        this.render();
      }

      init() {
        // 高DPI适配
        const dpr = window.devicePixelRatio || 1;
        const rect = this.canvas.getBoundingClientRect();

        this.canvas.width = rect.width * dpr;
        this.canvas.height = rect.height * dpr;
        this.canvas.style.width = rect.width + 'px';
        this.canvas.style.height = rect.height + 'px';

        this.ctx.scale(dpr, dpr);
      }

      setupEventListeners() {
        // 基础变换控件
        const controls = ['translateX', 'translateY', 'rotation', 'scaleX', 'scaleY', 'skewX', 'skewY'];

        controls.forEach(control => {
          const input = document.getElementById(control);
          const valueDisplay = document.getElementById(control + 'Value');

          input.addEventListener('input', (e) => {
            let value = parseFloat(e.target.value);

            if (control === 'rotation' || control.includes('skew')) {
              this.transforms[control] = value * Math.PI / 180; // 转换为弧度
              valueDisplay.textContent = value + '°';
            } else {
              this.transforms[control] = value;
              valueDisplay.textContent = value;
            }

            this.render();
          });
        });

        // 翻转控件
        document.getElementById('flipH').addEventListener('change', (e) => {
          this.transforms.flipH = e.target.checked;
          this.render();
        });

        document.getElementById('flipV').addEventListener('change', (e) => {
          this.transforms.flipV = e.target.checked;
          this.render();
        });

        // 混合模式
        document.getElementById('compositeOperation').addEventListener('change', (e) => {
          this.compositeOperation = e.target.value;
          this.render();
        });

        // 透明度
        document.getElementById('alpha').addEventListener('input', (e) => {
          this.alpha = e.target.value / 100;
          document.getElementById('alphaValue').textContent = e.target.value + '%';
          this.render();
        });
      }

      calculateTransformMatrix() {
        const t = this.transforms;

        // 创建单位矩阵
        let matrix = [1, 0, 0, 1, 0, 0];

        // 平移
        matrix = this.multiplyMatrix(matrix, [1, 0, 0, 1, t.translateX, t.translateY]);

        // 旋转
        if (t.rotation !== 0) {
          const cos = Math.cos(t.rotation);
          const sin = Math.sin(t.rotation);
          matrix = this.multiplyMatrix(matrix, [cos, sin, -sin, cos, 0, 0]);
        }

        // 缩放
        if (t.scaleX !== 1 || t.scaleY !== 1) {
          matrix = this.multiplyMatrix(matrix, [t.scaleX, 0, 0, t.scaleY, 0, 0]);
        }

        // 倾斜
        if (t.skewX !== 0 || t.skewY !== 0) {
          const tanX = Math.tan(t.skewX);
          const tanY = Math.tan(t.skewY);
          matrix = this.multiplyMatrix(matrix, [1, tanY, tanX, 1, 0, 0]);
        }

        // 翻转
        let flipX = t.flipH ? -1 : 1;
        let flipY = t.flipV ? -1 : 1;
        if (flipX !== 1 || flipY !== 1) {
          matrix = this.multiplyMatrix(matrix, [flipX, 0, 0, flipY, 0, 0]);
        }

        return matrix;
      }

      multiplyMatrix(a, b) {
        return [
          a[0] * b[0] + a[2] * b[1],
          a[1] * b[0] + a[3] * b[1],
          a[0] * b[2] + a[2] * b[3],
          a[1] * b[2] + a[3] * b[3],
          a[0] * b[4] + a[2] * b[5] + a[4],
          a[1] * b[4] + a[3] * b[5] + a[5]
        ];
      }

      render() {
        const ctx = this.ctx;
        const width = this.canvas.width / (window.devicePixelRatio || 1);
        const height = this.canvas.height / (window.devicePixelRatio || 1);

        // 清空画布
        ctx.clearRect(0, 0, width, height);

        // 绘制网格背景
        this.drawGrid();

        // 绘制原始图形（参考）
        ctx.save();
        ctx.translate(width/2, height/2);
        ctx.globalAlpha = 0.3;
        ctx.strokeStyle = '#ccc';
        ctx.setLineDash([5, 5]);
        this.drawTestShape();
        ctx.restore();

        // 应用变换并绘制
        ctx.save();
        ctx.translate(width/2, height/2);

        const matrix = this.calculateTransformMatrix();
        ctx.setTransform(...matrix, width/2, height/2);

        ctx.globalCompositeOperation = this.compositeOperation;
        ctx.globalAlpha = this.alpha;

        this.drawTestShape();
        ctx.restore();

        // 更新矩阵信息
        this.updateMatrixInfo(matrix);
      }

      drawGrid() {
        const ctx = this.ctx;
        const width = this.canvas.width / (window.devicePixelRatio || 1);
        const height = this.canvas.height / (window.devicePixelRatio || 1);
        const gridSize = 20;

        ctx.strokeStyle = '#f0f0f0';
        ctx.lineWidth = 1;

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

        // 中心线
        ctx.strokeStyle = '#ddd';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.moveTo(width/2, 0);
        ctx.lineTo(width/2, height);
        ctx.moveTo(0, height/2);
        ctx.lineTo(width, height/2);
        ctx.stroke();
      }

      drawTestShape() {
        const ctx = this.ctx;

        // 绘制复合图形
        // 主矩形
        ctx.fillStyle = '#3498db';
        ctx.fillRect(-50, -30, 100, 60);

        // 小圆形
        ctx.fillStyle = '#e74c3c';
        ctx.beginPath();
        ctx.arc(-25, -15, 10, 0, Math.PI * 2);
        ctx.fill();

        // 三角形
        ctx.fillStyle = '#2ecc71';
        ctx.beginPath();
        ctx.moveTo(15, -20);
        ctx.lineTo(35, 0);
        ctx.lineTo(15, 20);
        ctx.closePath();
        ctx.fill();

        // 文字
        ctx.fillStyle = 'white';
        ctx.font = '14px Arial';
        ctx.textAlign = 'center';
        ctx.fillText('DEMO', 0, 5);

        // 边框
        ctx.strokeStyle = '#34495e';
        ctx.lineWidth = 2;
        ctx.setLineDash([]);
        ctx.strokeRect(-50, -30, 100, 60);
      }

      updateMatrixInfo(matrix) {
        const [a, b, c, d, e, f] = matrix;
        const info = `变换矩阵:
[${a.toFixed(3)} ${c.toFixed(3)} ${e.toFixed(1)}]
[${b.toFixed(3)} ${d.toFixed(3)} ${f.toFixed(1)}]
[0.000  0.000  1.000]

Canvas 参数:
transform(${a.toFixed(3)}, ${b.toFixed(3)}, ${c.toFixed(3)}, ${d.toFixed(3)}, ${e.toFixed(1)}, ${f.toFixed(1)})

行列式: ${(a * d - b * c).toFixed(3)}`;

        document.getElementById('matrixInfo').value = info;
      }
    }

    // 预设功能
    const presets = {
      identity: {
        translateX: 0, translateY: 0, rotation: 0,
        scaleX: 1, scaleY: 1, skewX: 0, skewY: 0,
        flipH: false, flipV: false
      },
      rotate45: {
        translateX: 0, translateY: 0, rotation: 45,
        scaleX: 1, scaleY: 1, skewX: 0, skewY: 0,
        flipH: false, flipV: false
      },
      scale2x: {
        translateX: 0, translateY: 0, rotation: 0,
        scaleX: 2, scaleY: 2, skewX: 0, skewY: 0,
        flipH: false, flipV: false
      },
      flipH: {
        translateX: 0, translateY: 0, rotation: 0,
        scaleX: 1, scaleY: 1, skewX: 0, skewY: 0,
        flipH: true, flipV: false
      },
      '3d': {
        translateX: 0, translateY: -20, rotation: 0,
        scaleX: 1, scaleY: 0.8, skewX: -15, skewY: 0,
        flipH: false, flipV: false
      },
      crazy: {
        translateX: 30, translateY: -40, rotation: 30,
        scaleX: 1.5, scaleY: 0.7, skewX: 20, skewY: -10,
        flipH: false, flipV: true
      }
    };

    function loadPreset(presetName) {
      const preset = presets[presetName];
      if (!preset) return;

      // 更新控件值
      Object.entries(preset).forEach(([key, value]) => {
        if (key === 'flipH' || key === 'flipV') {
          document.getElementById(key).checked = value;
        } else {
          let displayValue = value;
          if (key === 'rotation' || key.includes('skew')) {
            displayValue = value; // 预设中已经是角度值
            document.getElementById(key).value = value;
            document.getElementById(key + 'Value').textContent = value + '°';
          } else {
            document.getElementById(key).value = value;
            document.getElementById(key + 'Value').textContent = value;
          }
        }
      });

      // 触发变化事件
      Object.keys(preset).forEach(key => {
        const element = document.getElementById(key);
        if (element) {
          element.dispatchEvent(new Event(element.type === 'checkbox' ? 'change' : 'input'));
        }
      });
    }

    // 初始化应用
    const transformTool = new TransformTool('transformCanvas');

    // 窗口大小变化时重新初始化
    window.addEventListener('resize', () => {
      transformTool.init();
      transformTool.render();
    });
  </script>
</body>
</html>
```

## 6. Excalidraw 中的应用

### 6.1 Excalidraw 的变换系统

```typescript
// packages/excalidraw/element/transform.ts
export const getTransformHandles = (
  element: ExcalidrawElement,
  zoom: number,
  pointerType: string
) => {
  const bounds = getElementAbsoluteCoords(element);
  const size = Math.max(8, Math.min(16, 20 / zoom));

  return {
    nw: { x: bounds[0] - size/2, y: bounds[1] - size/2 },
    ne: { x: bounds[2] - size/2, y: bounds[1] - size/2 },
    sw: { x: bounds[0] - size/2, y: bounds[3] - size/2 },
    se: { x: bounds[2] - size/2, y: bounds[3] - size/2 },
    n:  { x: (bounds[0] + bounds[2])/2 - size/2, y: bounds[1] - size/2 },
    s:  { x: (bounds[0] + bounds[2])/2 - size/2, y: bounds[3] - size/2 },
    w:  { x: bounds[0] - size/2, y: (bounds[1] + bounds[3])/2 - size/2 },
    e:  { x: bounds[2] - size/2, y: (bounds[1] + bounds[3])/2 - size/2 },
  };
};

export const transformElements = (
  elements: ExcalidrawElement[],
  transformHandleType: TransformHandleType,
  selectedElements: ExcalidrawElement[],
  pointerX: number,
  pointerY: number,
  originalPointerX: number,
  originalPointerY: number,
  shouldChangeBackgroundColor: boolean,
  shouldKeepSidesRatio: boolean
) => {
  // 计算变换矩阵
  const transformMatrix = calculateTransformMatrix(
    transformHandleType,
    pointerX - originalPointerX,
    pointerY - originalPointerY,
    shouldKeepSidesRatio
  );

  // 应用变换到选中元素
  return elements.map((element) => {
    if (selectedElements.includes(element)) {
      return applyTransformToElement(element, transformMatrix);
    }
    return element;
  });
};

// 元素变换应用
const applyTransformToElement = (
  element: ExcalidrawElement,
  transform: TransformMatrix
): ExcalidrawElement => {
  const [a, b, c, d, e, f] = transform;

  // 变换元素的位置和大小
  const newX = a * element.x + c * element.y + e;
  const newY = b * element.x + d * element.y + f;
  const newWidth = Math.abs(a * element.width + c * element.height);
  const newHeight = Math.abs(b * element.width + d * element.height);

  return {
    ...element,
    x: newX,
    y: newY,
    width: newWidth,
    height: newHeight,
  };
};
```

## 7. 练习题

### 7.1 基础练习

1. **矩阵计算器**
   - 实现矩阵乘法函数
   - 计算逆矩阵
   - 矩阵分解（提取平移、旋转、缩放）

2. **变换动画**
   - 从一个变换状态平滑过渡到另一个状态
   - 实现弹性动画效果
   - 支持缓动函数

3. **3D 透视模拟**
   - 使用 2D 变换模拟 3D 效果
   - 实现简单的 3D 立方体旋转

### 7.2 进阶练习

实现一个完整的变换编辑器：

```javascript
class TransformEditor {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.selectedElement = null;
    this.transformHandles = [];
  }

  // 实现变换手柄显示和交互
  drawTransformHandles() {
    // TODO: 绘制 8 个变换手柄
  }

  // 实现拖拽变换
  handleMouseDrag(startPos, currentPos, handleType) {
    // TODO: 根据手柄类型计算变换
  }

  // 实现约束变换（等比缩放、45度旋转等）
  applyConstraints(transform, constraints) {
    // TODO: 应用变换约束
  }
}
```

## 8. 思考题

1. **为什么使用矩阵表示变换？**
   - 数学上的优势
   - 复合变换的简化
   - 硬件加速支持

2. **变换顺序为什么重要？**
   - 矩阵乘法的不可交换性
   - 实际应用中的影响

3. **如何优化大量元素的变换计算？**
   - 矩阵缓存策略
   - 增量更新方案
   - GPU 加速可能性

4. **状态管理的最佳实践是什么？**
   - 何时使用 save/restore
   - 如何避免状态泄漏
   - 性能考虑因素

## 9. 总结

### 核心要点

1. **变换矩阵**
   - 理解仿射变换的数学原理
   - 掌握基本变换的矩阵表示
   - 熟练进行矩阵运算

2. **Canvas 变换 API**
   - 基本变换方法的使用
   - transform 和 setTransform 的区别
   - 直接矩阵操作技巧

3. **状态管理**
   - save/restore 机制
   - 状态栈的理解和应用
   - 嵌套变换的处理

4. **图像合成**
   - 各种混合模式的效果和用途
   - 透明度的控制和应用
   - 实际场景中的应用技巧

### 下一步

在掌握了变换与合成后，下一章将学习：
- Canvas 事件处理系统
- 坐标转换和碰撞检测
- 复杂交互的实现
- 手势识别技术

## 10. 参考资源

- [MDN Transform Functions](https://developer.mozilla.org/en-US/docs/Web/CSS/transform-function)
- [Canvas State Management](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/save)
- [Composite Operations](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/globalCompositeOperation)
- [Matrix Math for Graphics](https://www.mathworks.com/help/phased/ug/introduction-to-2-d-transforms.html)

---

**上一章**：[Canvas 图形绘制](./02-canvas-drawing.md)
**下一章**：[Canvas 事件与交互 →](./04-canvas-interaction.md)