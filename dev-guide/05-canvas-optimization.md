# 第五章：Canvas 性能优化

## 学习目标

- [ ] 理解 Canvas 性能瓶颈和优化原理
- [ ] 掌握脏矩形算法和增量渲染
- [ ] 实现离屏 Canvas 和分层渲染
- [ ] 学会使用 requestAnimationFrame 优化动画
- [ ] 掌握内存管理和垃圾回收优化
- [ ] 构建一个高性能的画板应用

## 1. Canvas 性能基础

### 1.1 性能瓶颈分析

Canvas 应用的主要性能瓶颈：

```javascript
class PerformanceAnalyzer {
  constructor() {
    this.metrics = {
      renderTime: [],
      drawCalls: 0,
      memoryUsage: 0,
      fps: 0
    };

    this.startTime = 0;
    this.frameCount = 0;
    this.lastFrameTime = performance.now();
  }

  // 开始性能监测
  startFrame() {
    this.startTime = performance.now();
    this.drawCalls = 0;
  }

  // 记录绘制调用
  recordDrawCall() {
    this.drawCalls++;
  }

  // 结束性能监测
  endFrame() {
    const endTime = performance.now();
    const renderTime = endTime - this.startTime;

    this.metrics.renderTime.push(renderTime);

    // 保留最近100帧的数据
    if (this.metrics.renderTime.length > 100) {
      this.metrics.renderTime.shift();
    }

    // 计算FPS
    this.frameCount++;
    if (endTime - this.lastFrameTime >= 1000) {
      this.metrics.fps = this.frameCount;
      this.frameCount = 0;
      this.lastFrameTime = endTime;
    }

    // 内存使用情况
    if (performance.memory) {
      this.metrics.memoryUsage = performance.memory.usedJSHeapSize;
    }
  }

  // 获取性能报告
  getReport() {
    const renderTimes = this.metrics.renderTime;
    const avgRenderTime = renderTimes.reduce((a, b) => a + b, 0) / renderTimes.length;
    const maxRenderTime = Math.max(...renderTimes);
    const minRenderTime = Math.min(...renderTimes);

    return {
      fps: this.metrics.fps,
      averageRenderTime: avgRenderTime.toFixed(2),
      maxRenderTime: maxRenderTime.toFixed(2),
      minRenderTime: minRenderTime.toFixed(2),
      drawCalls: this.drawCalls,
      memoryUsage: (this.metrics.memoryUsage / 1024 / 1024).toFixed(2) + ' MB'
    };
  }

  // 性能瓶颈诊断
  diagnose() {
    const report = this.getReport();
    const issues = [];

    if (report.fps < 30) {
      issues.push('FPS过低，需要优化渲染性能');
    }

    if (parseFloat(report.averageRenderTime) > 16.67) {
      issues.push('平均渲染时间超过16.67ms，可能导致掉帧');
    }

    if (this.drawCalls > 1000) {
      issues.push('绘制调用次数过多，考虑批量渲染');
    }

    if (parseFloat(report.memoryUsage) > 100) {
      issues.push('内存使用量过高，检查是否存在内存泄漏');
    }

    return {
      report,
      issues,
      recommendations: this.getRecommendations(issues)
    };
  }

  getRecommendations(issues) {
    const recommendations = [];

    if (issues.some(issue => issue.includes('FPS'))) {
      recommendations.push('实现脏矩形更新');
      recommendations.push('使用离屏Canvas缓存');
      recommendations.push('减少不必要的重绘');
    }

    if (issues.some(issue => issue.includes('渲染时间'))) {
      recommendations.push('优化绘制算法');
      recommendations.push('使用分层渲染');
      recommendations.push('实现视口裁剪');
    }

    if (issues.some(issue => issue.includes('绘制调用'))) {
      recommendations.push('批量绘制相同类型的元素');
      recommendations.push('使用Canvas状态缓存');
      recommendations.push('减少上下文切换');
    }

    if (issues.some(issue => issue.includes('内存'))) {
      recommendations.push('及时清理未使用的资源');
      recommendations.push('使用对象池模式');
      recommendations.push('避免创建大量临时对象');
    }

    return recommendations;
  }
}
```

### 1.2 性能测试工具

```javascript
class CanvasPerformanceTester {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.analyzer = new PerformanceAnalyzer();
  }

  // 测试基本绘制性能
  testBasicDrawing() {
    const iterations = 10000;

    console.log('开始基本绘制性能测试...');

    // 测试矩形绘制
    const rectStart = performance.now();
    for (let i = 0; i < iterations; i++) {
      this.ctx.fillRect(
        Math.random() * this.canvas.width,
        Math.random() * this.canvas.height,
        10, 10
      );
    }
    const rectTime = performance.now() - rectStart;

    // 清空画布
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // 测试圆形绘制
    const circleStart = performance.now();
    for (let i = 0; i < iterations; i++) {
      this.ctx.beginPath();
      this.ctx.arc(
        Math.random() * this.canvas.width,
        Math.random() * this.canvas.height,
        5, 0, Math.PI * 2
      );
      this.ctx.fill();
    }
    const circleTime = performance.now() - circleStart;

    return {
      rectangles: `${iterations}个矩形: ${rectTime.toFixed(2)}ms`,
      circles: `${iterations}个圆形: ${circleTime.toFixed(2)}ms`,
      rectPerSecond: Math.round(iterations * 1000 / rectTime),
      circlePerSecond: Math.round(iterations * 1000 / circleTime)
    };
  }

  // 测试大量元素渲染性能
  testMassRendering() {
    const elementCounts = [100, 500, 1000, 2000, 5000];
    const results = [];

    elementCounts.forEach(count => {
      // 生成测试元素
      const elements = this.generateTestElements(count);

      const startTime = performance.now();
      this.renderElements(elements);
      const endTime = performance.now();

      results.push({
        elementCount: count,
        renderTime: endTime - startTime,
        elementsPerSecond: Math.round(count * 1000 / (endTime - startTime))
      });

      // 清空画布
      this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    });

    return results;
  }

  generateTestElements(count) {
    const elements = [];
    const types = ['rect', 'circle', 'line'];

    for (let i = 0; i < count; i++) {
      elements.push({
        type: types[Math.floor(Math.random() * types.length)],
        x: Math.random() * this.canvas.width,
        y: Math.random() * this.canvas.height,
        width: 10 + Math.random() * 40,
        height: 10 + Math.random() * 40,
        color: `hsl(${Math.random() * 360}, 70%, 60%)`
      });
    }

    return elements;
  }

  renderElements(elements) {
    elements.forEach(element => {
      this.ctx.fillStyle = element.color;

      switch (element.type) {
        case 'rect':
          this.ctx.fillRect(element.x, element.y, element.width, element.height);
          break;
        case 'circle':
          this.ctx.beginPath();
          this.ctx.arc(
            element.x + element.width / 2,
            element.y + element.height / 2,
            element.width / 2,
            0, Math.PI * 2
          );
          this.ctx.fill();
          break;
        case 'line':
          this.ctx.beginPath();
          this.ctx.moveTo(element.x, element.y);
          this.ctx.lineTo(element.x + element.width, element.y + element.height);
          this.ctx.stroke();
          break;
      }
    });
  }

  // 运行完整性能测试套件
  runFullTest() {
    console.log('=== Canvas 性能测试报告 ===');

    const basicTest = this.testBasicDrawing();
    console.log('基本绘制性能:', basicTest);

    const massTest = this.testMassRendering();
    console.log('大量元素渲染性能:', massTest);

    return {
      basic: basicTest,
      mass: massTest
    };
  }
}
```

## 2. 脏矩形算法

### 2.1 基础脏矩形实现

```javascript
class DirtyRectManager {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.dirtyRegions = [];
    this.mergeThreshold = 0.5; // 合并阈值
  }

  // 添加脏矩形区域
  addDirtyRegion(x, y, width, height) {
    const newRegion = {
      x: Math.max(0, Math.floor(x)),
      y: Math.max(0, Math.floor(y)),
      width: Math.min(this.canvas.width - Math.floor(x), Math.ceil(width)),
      height: Math.min(this.canvas.height - Math.floor(y), Math.ceil(height))
    };

    // 过滤无效区域
    if (newRegion.width <= 0 || newRegion.height <= 0) {
      return;
    }

    this.dirtyRegions.push(newRegion);
  }

  // 根据元素变化添加脏矩形
  addElementDirtyRegion(oldElement, newElement) {
    if (oldElement) {
      // 添加旧位置的脏矩形
      this.addDirtyRegion(
        oldElement.x - 2,
        oldElement.y - 2,
        oldElement.width + 4,
        oldElement.height + 4
      );
    }

    if (newElement) {
      // 添加新位置的脏矩形
      this.addDirtyRegion(
        newElement.x - 2,
        newElement.y - 2,
        newElement.width + 4,
        newElement.height + 4
      );
    }
  }

  // 合并重叠的脏矩形
  mergeDirtyRegions() {
    if (this.dirtyRegions.length <= 1) return;

    const merged = [];
    const used = new Set();

    for (let i = 0; i < this.dirtyRegions.length; i++) {
      if (used.has(i)) continue;

      let currentRegion = { ...this.dirtyRegions[i] };
      used.add(i);

      // 查找可以合并的区域
      let foundMerge = true;
      while (foundMerge) {
        foundMerge = false;

        for (let j = i + 1; j < this.dirtyRegions.length; j++) {
          if (used.has(j)) continue;

          const otherRegion = this.dirtyRegions[j];

          if (this.shouldMergeRegions(currentRegion, otherRegion)) {
            currentRegion = this.mergeRegions(currentRegion, otherRegion);
            used.add(j);
            foundMerge = true;
          }
        }
      }

      merged.push(currentRegion);
    }

    this.dirtyRegions = merged;
  }

  // 判断是否应该合并两个区域
  shouldMergeRegions(region1, region2) {
    // 计算两个区域的联合区域
    const unionX = Math.min(region1.x, region2.x);
    const unionY = Math.min(region1.y, region2.y);
    const unionRight = Math.max(region1.x + region1.width, region2.x + region2.width);
    const unionBottom = Math.max(region1.y + region1.height, region2.y + region2.height);
    const unionArea = (unionRight - unionX) * (unionBottom - unionY);

    // 计算两个区域的总面积
    const totalArea = region1.width * region1.height + region2.width * region2.height;

    // 如果联合区域面积不超过总面积的阈值倍数，则合并
    return unionArea <= totalArea / this.mergeThreshold;
  }

  // 合并两个区域
  mergeRegions(region1, region2) {
    const x = Math.min(region1.x, region2.x);
    const y = Math.min(region1.y, region2.y);
    const right = Math.max(region1.x + region1.width, region2.x + region2.width);
    const bottom = Math.max(region1.y + region1.height, region2.y + region2.height);

    return {
      x,
      y,
      width: right - x,
      height: bottom - y
    };
  }

  // 清除指定区域
  clearRegion(region) {
    this.ctx.clearRect(region.x, region.y, region.width, region.height);
  }

  // 应用脏矩形更新
  applyDirtyRegions(renderCallback) {
    if (this.dirtyRegions.length === 0) return;

    // 合并重叠区域
    this.mergeDirtyRegions();

    // 保存当前上下文状态
    this.ctx.save();

    this.dirtyRegions.forEach(region => {
      // 设置裁剪区域
      this.ctx.beginPath();
      this.ctx.rect(region.x, region.y, region.width, region.height);
      this.ctx.clip();

      // 清除区域
      this.clearRegion(region);

      // 重新绘制该区域
      renderCallback(region);

      // 恢复上下文状态
      this.ctx.restore();
      this.ctx.save();
    });

    this.ctx.restore();

    // 清空脏矩形列表
    this.dirtyRegions = [];
  }

  // 获取脏矩形统计信息
  getStats() {
    if (this.dirtyRegions.length === 0) {
      return { regionCount: 0, totalArea: 0, coverage: 0 };
    }

    const totalArea = this.dirtyRegions.reduce(
      (sum, region) => sum + region.width * region.height,
      0
    );

    const canvasArea = this.canvas.width * this.canvas.height;
    const coverage = (totalArea / canvasArea * 100).toFixed(2);

    return {
      regionCount: this.dirtyRegions.length,
      totalArea,
      coverage: coverage + '%'
    };
  }

  // 可视化脏矩形（调试用）
  visualizeDirtyRegions() {
    this.ctx.save();
    this.ctx.strokeStyle = 'red';
    this.ctx.lineWidth = 2;
    this.ctx.setLineDash([5, 5]);

    this.dirtyRegions.forEach((region, index) => {
      this.ctx.strokeRect(region.x, region.y, region.width, region.height);

      // 显示区域编号
      this.ctx.fillStyle = 'red';
      this.ctx.font = '12px monospace';
      this.ctx.fillText(`${index}`, region.x + 5, region.y + 15);
    });

    this.ctx.restore();
  }
}
```

### 2.2 增量渲染系统

```javascript
class IncrementalRenderer {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.dirtyManager = new DirtyRectManager(canvas);

    this.elements = [];
    this.elementCache = new Map(); // 元素渲染缓存
    this.lastRenderState = new Map(); // 上次渲染状态
  }

  // 添加元素
  addElement(element) {
    this.elements.push(element);
    this.invalidateElement(element);
  }

  // 更新元素
  updateElement(elementId, updates) {
    const element = this.elements.find(el => el.id === elementId);
    if (!element) return;

    const oldState = { ...element };
    Object.assign(element, updates);

    // 添加脏矩形区域
    this.dirtyManager.addElementDirtyRegion(oldState, element);

    // 清除缓存
    this.elementCache.delete(elementId);
  }

  // 删除元素
  removeElement(elementId) {
    const elementIndex = this.elements.findIndex(el => el.id === elementId);
    if (elementIndex === -1) return;

    const element = this.elements[elementIndex];

    // 添加脏矩形
    this.dirtyManager.addElementDirtyRegion(element, null);

    // 删除元素和缓存
    this.elements.splice(elementIndex, 1);
    this.elementCache.delete(elementId);
  }

  // 标记元素无效
  invalidateElement(element) {
    this.dirtyManager.addElementDirtyRegion(null, element);
    this.elementCache.delete(element.id);
  }

  // 增量渲染
  render() {
    this.dirtyManager.applyDirtyRegions((region) => {
      this.renderRegion(region);
    });
  }

  // 渲染指定区域
  renderRegion(region) {
    // 找到与该区域相交的元素
    const intersectingElements = this.getElementsInRegion(region);

    // 按z-index排序
    intersectingElements.sort((a, b) => (a.zIndex || 0) - (b.zIndex || 0));

    // 渲染元素
    intersectingElements.forEach(element => {
      this.renderElement(element);
    });
  }

  // 获取区域内的元素
  getElementsInRegion(region) {
    return this.elements.filter(element => {
      return this.isElementInRegion(element, region);
    });
  }

  // 判断元素是否在区域内
  isElementInRegion(element, region) {
    return !(element.x > region.x + region.width ||
             element.x + element.width < region.x ||
             element.y > region.y + region.height ||
             element.y + element.height < region.y);
  }

  // 渲染单个元素
  renderElement(element) {
    // 检查缓存
    if (this.elementCache.has(element.id)) {
      const cached = this.elementCache.get(element.id);
      this.ctx.drawImage(cached.canvas, cached.x, cached.y);
      return;
    }

    // 渲染元素
    this.ctx.save();

    switch (element.type) {
      case 'rectangle':
        this.renderRectangle(element);
        break;
      case 'circle':
        this.renderCircle(element);
        break;
      case 'line':
        this.renderLine(element);
        break;
      case 'text':
        this.renderText(element);
        break;
    }

    this.ctx.restore();

    // 缓存复杂元素
    if (this.shouldCacheElement(element)) {
      this.cacheElement(element);
    }
  }

  renderRectangle(element) {
    this.ctx.fillStyle = element.fillColor || '#000';
    this.ctx.strokeStyle = element.strokeColor || '#000';
    this.ctx.lineWidth = element.strokeWidth || 1;

    if (element.fillColor) {
      this.ctx.fillRect(element.x, element.y, element.width, element.height);
    }
    if (element.strokeColor) {
      this.ctx.strokeRect(element.x, element.y, element.width, element.height);
    }
  }

  renderCircle(element) {
    this.ctx.fillStyle = element.fillColor || '#000';
    this.ctx.strokeStyle = element.strokeColor || '#000';
    this.ctx.lineWidth = element.strokeWidth || 1;

    this.ctx.beginPath();
    this.ctx.arc(
      element.x + element.width / 2,
      element.y + element.height / 2,
      element.width / 2,
      0, Math.PI * 2
    );

    if (element.fillColor) {
      this.ctx.fill();
    }
    if (element.strokeColor) {
      this.ctx.stroke();
    }
  }

  renderLine(element) {
    this.ctx.strokeStyle = element.strokeColor || '#000';
    this.ctx.lineWidth = element.strokeWidth || 1;

    this.ctx.beginPath();
    this.ctx.moveTo(element.x, element.y);
    this.ctx.lineTo(element.endX || element.x + element.width,
                    element.endY || element.y + element.height);
    this.ctx.stroke();
  }

  renderText(element) {
    this.ctx.fillStyle = element.fillColor || '#000';
    this.ctx.font = `${element.fontSize || 16}px ${element.fontFamily || 'Arial'}`;
    this.ctx.textAlign = element.textAlign || 'left';
    this.ctx.textBaseline = element.textBaseline || 'top';

    this.ctx.fillText(element.text || '', element.x, element.y);
  }

  // 判断是否应该缓存元素
  shouldCacheElement(element) {
    // 复杂元素或大型元素适合缓存
    return element.type === 'text' ||
           (element.width * element.height > 10000) ||
           element.hasComplexStyles;
  }

  // 缓存元素
  cacheElement(element) {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');

    canvas.width = Math.ceil(element.width + 10);
    canvas.height = Math.ceil(element.height + 10);

    // 在缓存画布上绘制元素
    const originalCtx = this.ctx;
    this.ctx = ctx;

    const tempElement = { ...element, x: 5, y: 5 };
    this.renderElement(tempElement);

    this.ctx = originalCtx;

    // 存储缓存
    this.elementCache.set(element.id, {
      canvas,
      x: element.x - 5,
      y: element.y - 5
    });
  }

  // 获取渲染统计
  getRenderStats() {
    const stats = this.dirtyManager.getStats();
    return {
      ...stats,
      elementCount: this.elements.length,
      cachedElements: this.elementCache.size,
      cacheHitRate: this.calculateCacheHitRate()
    };
  }

  calculateCacheHitRate() {
    // 这里应该记录缓存命中次数
    // 简化实现
    return ((this.elementCache.size / Math.max(this.elements.length, 1)) * 100).toFixed(2) + '%';
  }
}
```

## 3. 离屏 Canvas 和分层渲染

### 3.1 离屏 Canvas 技术

```javascript
class OffscreenCanvasManager {
  constructor(mainCanvas) {
    this.mainCanvas = mainCanvas;
    this.mainCtx = mainCanvas.getContext('2d');
    this.offscreenCanvases = new Map();
    this.layers = new Map();
  }

  // 创建离屏Canvas
  createOffscreenCanvas(id, width, height) {
    let canvas;
    let ctx;

    if (typeof OffscreenCanvas !== 'undefined') {
      // 使用OffscreenCanvas API（支持Worker）
      canvas = new OffscreenCanvas(width, height);
      ctx = canvas.getContext('2d');
    } else {
      // 降级到普通Canvas
      canvas = document.createElement('canvas');
      canvas.width = width;
      canvas.height = height;
      ctx = canvas.getContext('2d');
    }

    this.offscreenCanvases.set(id, { canvas, ctx });
    return { canvas, ctx };
  }

  // 获取离屏Canvas
  getOffscreenCanvas(id) {
    return this.offscreenCanvases.get(id);
  }

  // 删除离屏Canvas
  removeOffscreenCanvas(id) {
    const offscreen = this.offscreenCanvases.get(id);
    if (offscreen) {
      // 清理资源
      if (offscreen.canvas.transferToImageBitmap) {
        // OffscreenCanvas的清理
        offscreen.canvas.width = 0;
        offscreen.canvas.height = 0;
      }
      this.offscreenCanvases.delete(id);
    }
  }

  // 将离屏Canvas内容绘制到主Canvas
  drawOffscreenCanvas(id, x = 0, y = 0, sx = 0, sy = 0, sw, sh, dx, dy, dw, dh) {
    const offscreen = this.offscreenCanvases.get(id);
    if (!offscreen) return;

    if (arguments.length === 3) {
      // 简单绘制
      this.mainCtx.drawImage(offscreen.canvas, x, y);
    } else {
      // 复杂绘制（裁剪、缩放）
      this.mainCtx.drawImage(
        offscreen.canvas,
        sx, sy, sw || offscreen.canvas.width, sh || offscreen.canvas.height,
        dx || x, dy || y, dw || sw, dh || sh
      );
    }
  }

  // 批量管理多个离屏Canvas
  createCanvasGroup(groupId, canvasConfigs) {
    const group = new Map();

    canvasConfigs.forEach(config => {
      const { canvas, ctx } = this.createOffscreenCanvas(
        `${groupId}_${config.id}`,
        config.width,
        config.height
      );
      group.set(config.id, { canvas, ctx, ...config });
    });

    return group;
  }
}
```

### 3.2 分层渲染系统

```javascript
class LayeredRenderer {
  constructor(canvas) {
    this.mainCanvas = canvas;
    this.mainCtx = canvas.getContext('2d');
    this.offscreenManager = new OffscreenCanvasManager(canvas);

    this.layers = new Map();
    this.layerOrder = [];
    this.dirtyLayers = new Set();
  }

  // 创建图层
  createLayer(id, zIndex = 0, options = {}) {
    const layer = {
      id,
      zIndex,
      elements: [],
      visible: true,
      opacity: 1,
      blendMode: 'source-over',
      ...options,
      isDirty: true
    };

    // 创建离屏Canvas
    const { canvas, ctx } = this.offscreenManager.createOffscreenCanvas(
      id,
      this.mainCanvas.width,
      this.mainCanvas.height
    );

    layer.canvas = canvas;
    layer.ctx = ctx;

    this.layers.set(id, layer);
    this.updateLayerOrder();

    return layer;
  }

  // 更新图层顺序
  updateLayerOrder() {
    this.layerOrder = Array.from(this.layers.values())
      .sort((a, b) => a.zIndex - b.zIndex)
      .map(layer => layer.id);
  }

  // 添加元素到图层
  addElementToLayer(layerId, element) {
    const layer = this.layers.get(layerId);
    if (!layer) return;

    layer.elements.push(element);
    this.markLayerDirty(layerId);
  }

  // 从图层移除元素
  removeElementFromLayer(layerId, elementId) {
    const layer = this.layers.get(layerId);
    if (!layer) return;

    const index = layer.elements.findIndex(el => el.id === elementId);
    if (index > -1) {
      layer.elements.splice(index, 1);
      this.markLayerDirty(layerId);
    }
  }

  // 标记图层为脏
  markLayerDirty(layerId) {
    const layer = this.layers.get(layerId);
    if (layer) {
      layer.isDirty = true;
      this.dirtyLayers.add(layerId);
    }
  }

  // 渲染单个图层
  renderLayer(layerId) {
    const layer = this.layers.get(layerId);
    if (!layer || !layer.isDirty) return;

    // 清空图层画布
    layer.ctx.clearRect(0, 0, layer.canvas.width, layer.canvas.height);

    // 设置图层样式
    layer.ctx.save();
    layer.ctx.globalAlpha = layer.opacity;
    layer.ctx.globalCompositeOperation = layer.blendMode;

    // 渲染图层中的所有元素
    layer.elements.forEach(element => {
      this.renderElement(layer.ctx, element);
    });

    layer.ctx.restore();
    layer.isDirty = false;
  }

  // 渲染所有图层到主Canvas
  render() {
    // 只渲染脏图层
    this.dirtyLayers.forEach(layerId => {
      this.renderLayer(layerId);
    });
    this.dirtyLayers.clear();

    // 清空主Canvas
    this.mainCtx.clearRect(0, 0, this.mainCanvas.width, this.mainCanvas.height);

    // 按顺序合成所有可见图层
    this.layerOrder.forEach(layerId => {
      const layer = this.layers.get(layerId);
      if (layer && layer.visible) {
        this.mainCtx.save();
        this.mainCtx.globalAlpha = layer.opacity;
        this.mainCtx.globalCompositeOperation = layer.blendMode;
        this.mainCtx.drawImage(layer.canvas, 0, 0);
        this.mainCtx.restore();
      }
    });
  }

  // 渲染元素
  renderElement(ctx, element) {
    ctx.save();

    switch (element.type) {
      case 'rectangle':
        ctx.fillStyle = element.color || '#000';
        ctx.fillRect(element.x, element.y, element.width, element.height);
        break;
      case 'circle':
        ctx.fillStyle = element.color || '#000';
        ctx.beginPath();
        ctx.arc(
          element.x + element.width / 2,
          element.y + element.height / 2,
          element.width / 2,
          0, Math.PI * 2
        );
        ctx.fill();
        break;
      case 'text':
        ctx.fillStyle = element.color || '#000';
        ctx.font = `${element.fontSize || 16}px ${element.fontFamily || 'Arial'}`;
        ctx.fillText(element.text || '', element.x, element.y);
        break;
    }

    ctx.restore();
  }

  // 图层管理
  setLayerVisibility(layerId, visible) {
    const layer = this.layers.get(layerId);
    if (layer) {
      layer.visible = visible;
    }
  }

  setLayerOpacity(layerId, opacity) {
    const layer = this.layers.get(layerId);
    if (layer) {
      layer.opacity = Math.max(0, Math.min(1, opacity));
    }
  }

  setLayerBlendMode(layerId, blendMode) {
    const layer = this.layers.get(layerId);
    if (layer) {
      layer.blendMode = blendMode;
    }
  }

  moveLayer(layerId, newZIndex) {
    const layer = this.layers.get(layerId);
    if (layer) {
      layer.zIndex = newZIndex;
      this.updateLayerOrder();
    }
  }

  // 清理图层
  removeLayer(layerId) {
    const layer = this.layers.get(layerId);
    if (layer) {
      this.offscreenManager.removeOffscreenCanvas(layerId);
      this.layers.delete(layerId);
      this.dirtyLayers.delete(layerId);
      this.updateLayerOrder();
    }
  }

  // 获取图层信息
  getLayerInfo() {
    return this.layerOrder.map(layerId => {
      const layer = this.layers.get(layerId);
      return {
        id: layer.id,
        zIndex: layer.zIndex,
        visible: layer.visible,
        opacity: layer.opacity,
        elementCount: layer.elements.length,
        isDirty: layer.isDirty
      };
    });
  }
}
```

## 4. RequestAnimationFrame 优化

### 4.1 动画循环管理

```javascript
class AnimationManager {
  constructor() {
    this.animations = new Map();
    this.isRunning = false;
    this.lastFrameTime = 0;
    this.targetFPS = 60;
    this.frameInterval = 1000 / this.targetFPS;

    this.stats = {
      frameCount: 0,
      droppedFrames: 0,
      averageFPS: 0
    };
  }

  // 添加动画
  addAnimation(id, animationFunction, options = {}) {
    const animation = {
      id,
      func: animationFunction,
      priority: options.priority || 0,
      startTime: performance.now(),
      duration: options.duration || Infinity,
      enabled: true,
      ...options
    };

    this.animations.set(id, animation);

    if (!this.isRunning) {
      this.start();
    }
  }

  // 移除动画
  removeAnimation(id) {
    this.animations.delete(id);

    if (this.animations.size === 0) {
      this.stop();
    }
  }

  // 启动动画循环
  start() {
    if (this.isRunning) return;

    this.isRunning = true;
    this.lastFrameTime = performance.now();
    this.animationLoop();
  }

  // 停止动画循环
  stop() {
    this.isRunning = false;
  }

  // 主动画循环
  animationLoop = (currentTime) => {
    if (!this.isRunning) return;

    const deltaTime = currentTime - this.lastFrameTime;

    // 帧率控制
    if (deltaTime >= this.frameInterval) {
      const actualDelta = Math.min(deltaTime, this.frameInterval * 2);

      this.runAnimations(currentTime, actualDelta);

      this.lastFrameTime = currentTime - (deltaTime % this.frameInterval);
      this.updateStats(currentTime, deltaTime);
    }

    requestAnimationFrame(this.animationLoop);
  };

  // 执行所有动画
  runAnimations(currentTime, deltaTime) {
    // 按优先级排序
    const sortedAnimations = Array.from(this.animations.values())
      .filter(animation => animation.enabled)
      .sort((a, b) => b.priority - a.priority);

    const completedAnimations = [];

    for (const animation of sortedAnimations) {
      const elapsed = currentTime - animation.startTime;

      // 检查动画是否完成
      if (elapsed >= animation.duration) {
        completedAnimations.push(animation.id);
        continue;
      }

      // 执行动画函数
      try {
        const progress = animation.duration === Infinity ? 0 : elapsed / animation.duration;
        animation.func(currentTime, deltaTime, progress);
      } catch (error) {
        console.error(`Animation ${animation.id} error:`, error);
        completedAnimations.push(animation.id);
      }
    }

    // 清理完成的动画
    completedAnimations.forEach(id => this.removeAnimation(id));
  }

  // 更新统计信息
  updateStats(currentTime, deltaTime) {
    this.stats.frameCount++;

    // 检测掉帧
    if (deltaTime > this.frameInterval * 1.5) {
      this.stats.droppedFrames++;
    }

    // 计算平均FPS（每秒更新一次）
    if (this.stats.frameCount % 60 === 0) {
      this.stats.averageFPS = Math.round(1000 / deltaTime);
    }
  }

  // 暂停/恢复动画
  pauseAnimation(id) {
    const animation = this.animations.get(id);
    if (animation) {
      animation.enabled = false;
    }
  }

  resumeAnimation(id) {
    const animation = this.animations.get(id);
    if (animation) {
      animation.enabled = true;
      animation.startTime = performance.now(); // 重置开始时间
    }
  }

  // 设置目标帧率
  setTargetFPS(fps) {
    this.targetFPS = Math.max(1, Math.min(120, fps));
    this.frameInterval = 1000 / this.targetFPS;
  }

  // 获取统计信息
  getStats() {
    return {
      ...this.stats,
      activeAnimations: this.animations.size,
      targetFPS: this.targetFPS,
      dropRate: this.stats.frameCount > 0 ?
        (this.stats.droppedFrames / this.stats.frameCount * 100).toFixed(2) + '%' : '0%'
    };
  }
}
```

### 4.2 智能渲染调度

```javascript
class SmartRenderScheduler {
  constructor(renderer) {
    this.renderer = renderer;
    this.animationManager = new AnimationManager();

    this.renderQueue = [];
    this.isRendering = false;
    this.lastRenderTime = 0;
    this.renderThrottle = 16; // 最小渲染间隔(ms)

    this.adaptiveQuality = {
      enabled: true,
      currentLevel: 1.0,
      targetFPS: 60,
      minQuality: 0.3,
      maxQuality: 1.0
    };
  }

  // 请求渲染
  requestRender(priority = 0, region = null) {
    const renderRequest = {
      id: Date.now() + Math.random(),
      priority,
      region,
      timestamp: performance.now()
    };

    // 插入到正确的位置（按优先级排序）
    let insertIndex = this.renderQueue.length;
    for (let i = 0; i < this.renderQueue.length; i++) {
      if (this.renderQueue[i].priority < priority) {
        insertIndex = i;
        break;
      }
    }

    this.renderQueue.splice(insertIndex, 0, renderRequest);

    this.scheduleRender();
  }

  // 调度渲染
  scheduleRender() {
    if (this.isRendering) return;

    const now = performance.now();
    const timeSinceLastRender = now - this.lastRenderTime;

    if (timeSinceLastRender >= this.renderThrottle) {
      this.performRender();
    } else {
      // 延迟渲染
      setTimeout(() => {
        this.performRender();
      }, this.renderThrottle - timeSinceLastRender);
    }
  }

  // 执行渲染
  performRender() {
    if (this.renderQueue.length === 0) return;

    this.isRendering = true;
    const startTime = performance.now();

    // 合并渲染请求
    const mergedRequests = this.mergeRenderRequests();

    // 自适应质量调整
    if (this.adaptiveQuality.enabled) {
      this.adjustRenderQuality();
    }

    // 执行渲染
    mergedRequests.forEach(request => {
      if (request.region) {
        this.renderer.renderRegion(request.region);
      } else {
        this.renderer.render();
      }
    });

    const renderTime = performance.now() - startTime;
    this.updateRenderStats(renderTime);

    this.renderQueue = [];
    this.isRendering = false;
    this.lastRenderTime = performance.now();
  }

  // 合并渲染请求
  mergeRenderRequests() {
    if (this.renderQueue.length <= 1) return this.renderQueue;

    const regions = [];
    const fullRenders = [];

    this.renderQueue.forEach(request => {
      if (request.region) {
        regions.push(request);
      } else {
        fullRenders.push(request);
      }
    });

    // 如果有全屏渲染请求，忽略区域渲染
    if (fullRenders.length > 0) {
      return [fullRenders[0]]; // 只需要一次全屏渲染
    }

    // 合并相邻的区域
    return this.mergeRegions(regions);
  }

  // 合并相邻区域
  mergeRegions(regionRequests) {
    // 简化实现：使用联合边界框
    if (regionRequests.length <= 1) return regionRequests;

    let minX = Infinity, minY = Infinity;
    let maxX = -Infinity, maxY = -Infinity;

    regionRequests.forEach(request => {
      const region = request.region;
      minX = Math.min(minX, region.x);
      minY = Math.min(minY, region.y);
      maxX = Math.max(maxX, region.x + region.width);
      maxY = Math.max(maxY, region.y + region.height);
    });

    return [{
      id: 'merged',
      priority: Math.max(...regionRequests.map(r => r.priority)),
      region: {
        x: minX,
        y: minY,
        width: maxX - minX,
        height: maxY - minY
      },
      timestamp: performance.now()
    }];
  }

  // 自适应质量调整
  adjustRenderQuality() {
    const stats = this.animationManager.getStats();
    const currentFPS = stats.averageFPS;

    if (currentFPS < this.adaptiveQuality.targetFPS * 0.8) {
      // FPS过低，降低质量
      this.adaptiveQuality.currentLevel = Math.max(
        this.adaptiveQuality.minQuality,
        this.adaptiveQuality.currentLevel - 0.1
      );
    } else if (currentFPS > this.adaptiveQuality.targetFPS * 0.95) {
      // FPS良好，提高质量
      this.adaptiveQuality.currentLevel = Math.min(
        this.adaptiveQuality.maxQuality,
        this.adaptiveQuality.currentLevel + 0.05
      );
    }

    // 应用质量设置
    this.applyQualitySettings(this.adaptiveQuality.currentLevel);
  }

  // 应用质量设置
  applyQualitySettings(quality) {
    if (this.renderer.setQuality) {
      this.renderer.setQuality(quality);
    }

    // 调整渲染参数
    if (quality < 0.7) {
      this.renderThrottle = 32; // 降低帧率
    } else if (quality > 0.9) {
      this.renderThrottle = 16; // 恢复正常帧率
    }
  }

  // 更新渲染统计
  updateRenderStats(renderTime) {
    // 记录渲染时间等统计信息
    console.debug(`Render time: ${renderTime.toFixed(2)}ms, Quality: ${this.adaptiveQuality.currentLevel.toFixed(2)}`);
  }

  // 获取调度器状态
  getSchedulerState() {
    return {
      queueLength: this.renderQueue.length,
      isRendering: this.isRendering,
      currentQuality: this.adaptiveQuality.currentLevel,
      renderThrottle: this.renderThrottle,
      animationStats: this.animationManager.getStats()
    };
  }
}
```

## 5. 内存管理与优化

### 5.1 对象池模式

```javascript
class ObjectPool {
  constructor(createFn, resetFn, maxSize = 100) {
    this.createFn = createFn;
    this.resetFn = resetFn;
    this.maxSize = maxSize;
    this.pool = [];
    this.inUse = new Set();
  }

  // 获取对象
  acquire() {
    let obj;

    if (this.pool.length > 0) {
      obj = this.pool.pop();
    } else {
      obj = this.createFn();
    }

    this.inUse.add(obj);
    return obj;
  }

  // 释放对象
  release(obj) {
    if (!this.inUse.has(obj)) return;

    this.inUse.delete(obj);

    if (this.pool.length < this.maxSize) {
      this.resetFn(obj);
      this.pool.push(obj);
    }
  }

  // 批量释放
  releaseAll() {
    this.inUse.forEach(obj => {
      if (this.pool.length < this.maxSize) {
        this.resetFn(obj);
        this.pool.push(obj);
      }
    });
    this.inUse.clear();
  }

  // 清空池
  clear() {
    this.pool = [];
    this.inUse.clear();
  }

  // 获取统计信息
  getStats() {
    return {
      poolSize: this.pool.length,
      inUse: this.inUse.size,
      totalCreated: this.pool.length + this.inUse.size
    };
  }
}

// Canvas 专用对象池管理器
class CanvasObjectPoolManager {
  constructor() {
    this.pools = new Map();
    this.setupDefaultPools();
  }

  setupDefaultPools() {
    // Path2D对象池
    this.pools.set('path', new ObjectPool(
      () => new Path2D(),
      (path) => { /* Path2D没有reset方法，创建新的 */ },
      50
    ));

    // 图像数据池
    this.pools.set('imageData', new ObjectPool(
      () => null, // 延迟创建
      (imageData) => {
        // 清空像素数据
        if (imageData && imageData.data) {
          imageData.data.fill(0);
        }
      },
      10
    ));

    // 临时Canvas池
    this.pools.set('canvas', new ObjectPool(
      () => {
        const canvas = document.createElement('canvas');
        return { canvas, ctx: canvas.getContext('2d') };
      },
      (canvasObj) => {
        canvasObj.canvas.width = 1;
        canvasObj.canvas.height = 1;
      },
      5
    ));

    // 几何对象池
    this.pools.set('rect', new ObjectPool(
      () => ({ x: 0, y: 0, width: 0, height: 0 }),
      (rect) => {
        rect.x = rect.y = rect.width = rect.height = 0;
      },
      100
    ));

    this.pools.set('point', new ObjectPool(
      () => ({ x: 0, y: 0 }),
      (point) => {
        point.x = point.y = 0;
      },
      200
    ));
  }

  // 获取指定类型的对象
  acquire(type, ...args) {
    const pool = this.pools.get(type);
    if (!pool) {
      throw new Error(`Unknown object pool type: ${type}`);
    }

    const obj = pool.acquire();

    // 特殊处理某些类型
    if (type === 'imageData' && args.length >= 2) {
      const [width, height] = args;
      if (!obj || obj.width !== width || obj.height !== height) {
        // 创建新的ImageData
        return new ImageData(width, height);
      }
    } else if (type === 'canvas' && args.length >= 2) {
      const [width, height] = args;
      obj.canvas.width = width;
      obj.canvas.height = height;
    }

    return obj;
  }

  // 释放对象
  release(type, obj) {
    const pool = this.pools.get(type);
    if (pool) {
      pool.release(obj);
    }
  }

  // 批量释放
  releaseAll(type) {
    const pool = this.pools.get(type);
    if (pool) {
      pool.releaseAll();
    }
  }

  // 获取所有池的统计信息
  getAllStats() {
    const stats = {};
    this.pools.forEach((pool, type) => {
      stats[type] = pool.getStats();
    });
    return stats;
  }

  // 清理所有池
  clearAll() {
    this.pools.forEach(pool => pool.clear());
  }
}
```

### 5.2 内存泄漏检测

```javascript
class MemoryLeakDetector {
  constructor() {
    this.snapshots = [];
    this.references = new Map();
    this.intervalId = null;
    this.isMonitoring = false;
  }

  // 开始监控
  startMonitoring(interval = 5000) {
    if (this.isMonitoring) return;

    this.isMonitoring = true;
    this.intervalId = setInterval(() => {
      this.takeSnapshot();
    }, interval);
  }

  // 停止监控
  stopMonitoring() {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
    this.isMonitoring = false;
  }

  // 拍摄内存快照
  takeSnapshot() {
    if (!performance.memory) {
      console.warn('Performance.memory API not available');
      return;
    }

    const snapshot = {
      timestamp: Date.now(),
      usedJSHeapSize: performance.memory.usedJSHeapSize,
      totalJSHeapSize: performance.memory.totalJSHeapSize,
      jsHeapSizeLimit: performance.memory.jsHeapSizeLimit,
      references: this.countReferences()
    };

    this.snapshots.push(snapshot);

    // 保留最近100个快照
    if (this.snapshots.length > 100) {
      this.snapshots.shift();
    }

    this.analyzeMemoryTrend();
  }

  // 计算引用数量
  countReferences() {
    return {
      total: this.references.size,
      byType: this.groupReferencesByType()
    };
  }

  // 按类型分组引用
  groupReferencesByType() {
    const groups = {};
    this.references.forEach((ref, obj) => {
      const type = ref.type || 'unknown';
      groups[type] = (groups[type] || 0) + 1;
    });
    return groups;
  }

  // 分析内存趋势
  analyzeMemoryTrend() {
    if (this.snapshots.length < 10) return;

    const recentSnapshots = this.snapshots.slice(-10);
    const memoryGrowth = this.calculateMemoryGrowth(recentSnapshots);

    if (memoryGrowth > 10 * 1024 * 1024) { // 10MB增长
      console.warn('Potential memory leak detected:', {
        growth: `${(memoryGrowth / 1024 / 1024).toFixed(2)} MB`,
        snapshots: recentSnapshots.length
      });

      this.generateLeakReport();
    }
  }

  // 计算内存增长
  calculateMemoryGrowth(snapshots) {
    if (snapshots.length < 2) return 0;

    const first = snapshots[0];
    const last = snapshots[snapshots.length - 1];

    return last.usedJSHeapSize - first.usedJSHeapSize;
  }

  // 生成泄漏报告
  generateLeakReport() {
    const latest = this.snapshots[this.snapshots.length - 1];
    const previous = this.snapshots[this.snapshots.length - 10] || this.snapshots[0];

    const report = {
      timespan: latest.timestamp - previous.timestamp,
      memoryGrowth: latest.usedJSHeapSize - previous.usedJSHeapSize,
      referenceGrowth: latest.references.total - previous.references.total,
      suspiciousTypes: this.findSuspiciousTypes(latest, previous)
    };

    console.warn('Memory leak report:', report);
    return report;
  }

  // 找出可疑的类型
  findSuspiciousTypes(latest, previous) {
    const suspicious = [];
    const latestTypes = latest.references.byType;
    const previousTypes = previous.references.byType;

    Object.keys(latestTypes).forEach(type => {
      const currentCount = latestTypes[type];
      const previousCount = previousTypes[type] || 0;
      const growth = currentCount - previousCount;

      if (growth > 10) { // 增长超过10个引用
        suspicious.push({
          type,
          growth,
          currentCount,
          previousCount
        });
      }
    });

    return suspicious;
  }

  // 注册引用
  registerReference(obj, type, description = '') {
    this.references.set(obj, {
      type,
      description,
      timestamp: Date.now()
    });
  }

  // 注销引用
  unregisterReference(obj) {
    this.references.delete(obj);
  }

  // 获取内存统计
  getMemoryStats() {
    if (this.snapshots.length === 0) return null;

    const latest = this.snapshots[this.snapshots.length - 1];
    const growth = this.snapshots.length > 1 ?
      latest.usedJSHeapSize - this.snapshots[0].usedJSHeapSize : 0;

    return {
      current: {
        used: (latest.usedJSHeapSize / 1024 / 1024).toFixed(2) + ' MB',
        total: (latest.totalJSHeapSize / 1024 / 1024).toFixed(2) + ' MB',
        limit: (latest.jsHeapSizeLimit / 1024 / 1024).toFixed(2) + ' MB'
      },
      growth: (growth / 1024 / 1024).toFixed(2) + ' MB',
      references: latest.references.total,
      isMonitoring: this.isMonitoring
    };
  }
}

// Canvas 特定的内存管理器
class CanvasMemoryManager {
  constructor() {
    this.objectPool = new CanvasObjectPoolManager();
    this.leakDetector = new MemoryLeakDetector();
    this.cleanupTasks = new Set();
    this.autoCleanup = true;
  }

  // 开始内存监控
  startMonitoring() {
    this.leakDetector.startMonitoring();

    // 定期清理
    if (this.autoCleanup) {
      this.scheduleCleanup();
    }
  }

  // 定期清理任务
  scheduleCleanup() {
    setInterval(() => {
      this.performCleanup();
    }, 30000); // 每30秒清理一次
  }

  // 执行清理
  performCleanup() {
    // 执行所有清理任务
    this.cleanupTasks.forEach(task => {
      try {
        task();
      } catch (error) {
        console.error('Cleanup task failed:', error);
      }
    });

    // 建议垃圾回收
    if (window.gc) {
      window.gc();
    }
  }

  // 注册清理任务
  registerCleanupTask(task) {
    this.cleanupTasks.add(task);
  }

  // 注销清理任务
  unregisterCleanupTask(task) {
    this.cleanupTasks.delete(task);
  }

  // 获取综合内存状态
  getMemoryStatus() {
    return {
      heap: this.leakDetector.getMemoryStats(),
      pools: this.objectPool.getAllStats(),
      cleanupTasks: this.cleanupTasks.size
    };
  }
}
```

## 6. 实战：高性能画板应用

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>高性能 Canvas 画板</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Arial;
      background: #f0f0f0;
      height: 100vh;
      display: flex;
    }

    .sidebar {
      width: 250px;
      background: white;
      padding: 15px;
      box-shadow: 2px 0 5px rgba(0,0,0,0.1);
      overflow-y: auto;
    }

    .control-group {
      margin-bottom: 20px;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 8px;
    }

    .control-group h3 {
      margin-bottom: 10px;
      font-size: 14px;
      color: #333;
    }

    .control {
      margin-bottom: 8px;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .control label {
      font-size: 12px;
      color: #666;
    }

    .control input, .control select {
      width: 100px;
      padding: 4px;
      border: 1px solid #ddd;
      border-radius: 4px;
      font-size: 12px;
    }

    .canvas-container {
      flex: 1;
      position: relative;
      overflow: hidden;
    }

    canvas {
      background: white;
      cursor: crosshair;
    }

    .performance-panel {
      position: absolute;
      top: 10px;
      right: 10px;
      background: rgba(0,0,0,0.8);
      color: white;
      padding: 10px;
      border-radius: 8px;
      font-family: monospace;
      font-size: 12px;
      min-width: 200px;
    }

    .perf-item {
      display: flex;
      justify-content: space-between;
      margin-bottom: 4px;
    }

    .button-group {
      display: flex;
      gap: 5px;
      margin-top: 10px;
    }

    button {
      padding: 6px 12px;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 12px;
    }

    .btn-primary {
      background: #007acc;
      color: white;
    }

    .btn-secondary {
      background: #6c757d;
      color: white;
    }

    .btn-danger {
      background: #dc3545;
      color: white;
    }
  </style>
</head>
<body>
  <div class="sidebar">
    <div class="control-group">
      <h3>🎨 绘图工具</h3>
      <div class="control">
        <label>工具类型</label>
        <select id="toolType">
          <option value="brush">画笔</option>
          <option value="eraser">橡皮擦</option>
          <option value="shape">图形</option>
        </select>
      </div>
      <div class="control">
        <label>画笔大小</label>
        <input type="range" id="brushSize" min="1" max="50" value="5">
      </div>
      <div class="control">
        <label>颜色</label>
        <input type="color" id="brushColor" value="#000000">
      </div>
    </div>

    <div class="control-group">
      <h3>⚡ 性能优化</h3>
      <div class="control">
        <label>脏矩形</label>
        <input type="checkbox" id="dirtyRect" checked>
      </div>
      <div class="control">
        <label>分层渲染</label>
        <input type="checkbox" id="layeredRender" checked>
      </div>
      <div class="control">
        <label>自适应质量</label>
        <input type="checkbox" id="adaptiveQuality" checked>
      </div>
      <div class="control">
        <label>目标FPS</label>
        <input type="number" id="targetFPS" min="30" max="120" value="60">
      </div>
    </div>

    <div class="control-group">
      <h3>📊 测试</h3>
      <div class="control">
        <label>元素数量</label>
        <input type="number" id="elementCount" min="100" max="10000" value="1000">
      </div>
      <div class="button-group">
        <button class="btn-primary" onclick="generateTestElements()">生成测试</button>
        <button class="btn-secondary" onclick="runPerformanceTest()">性能测试</button>
        <button class="btn-danger" onclick="clearCanvas()">清空</button>
      </div>
    </div>

    <div class="control-group">
      <h3>🧹 内存管理</h3>
      <div class="button-group">
        <button class="btn-primary" onclick="toggleMemoryMonitoring()">内存监控</button>
        <button class="btn-secondary" onclick="performCleanup()">手动清理</button>
        <button class="btn-secondary" onclick="showMemoryReport()">内存报告</button>
      </div>
    </div>
  </div>

  <div class="canvas-container">
    <canvas id="canvas"></canvas>

    <div class="performance-panel" id="perfPanel">
      <div class="perf-item">
        <span>FPS:</span>
        <span id="fpsValue">0</span>
      </div>
      <div class="perf-item">
        <span>渲染时间:</span>
        <span id="renderTime">0ms</span>
      </div>
      <div class="perf-item">
        <span>元素数量:</span>
        <span id="elementCount">0</span>
      </div>
      <div class="perf-item">
        <span>脏矩形:</span>
        <span id="dirtyRegions">0</span>
      </div>
      <div class="perf-item">
        <span>内存使用:</span>
        <span id="memoryUsage">0MB</span>
      </div>
      <div class="perf-item">
        <span>质量级别:</span>
        <span id="qualityLevel">100%</span>
      </div>
    </div>
  </div>

  <script>
    class HighPerformanceDrawingApp {
      constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = this.canvas.getContext('2d');

        // 性能组件
        this.incrementalRenderer = new IncrementalRenderer(this.canvas);
        this.layeredRenderer = new LayeredRenderer(this.canvas);
        this.scheduler = new SmartRenderScheduler(this.incrementalRenderer);
        this.memoryManager = new CanvasMemoryManager();
        this.analyzer = new PerformanceAnalyzer();

        // 绘图状态
        this.isDrawing = false;
        this.currentStroke = null;
        this.elements = [];

        // 配置
        this.config = {
          useDirtyRect: true,
          useLayeredRender: true,
          useAdaptiveQuality: true,
          targetFPS: 60
        };

        this.init();
      }

      init() {
        this.setupCanvas();
        this.setupEventListeners();
        this.setupUI();
        this.startPerformanceMonitoring();

        // 创建默认图层
        this.backgroundLayer = this.layeredRenderer.createLayer('background', 0);
        this.drawingLayer = this.layeredRenderer.createLayer('drawing', 1);
        this.uiLayer = this.layeredRenderer.createLayer('ui', 2);
      }

      setupCanvas() {
        const container = this.canvas.parentElement;
        const resizeCanvas = () => {
          const rect = container.getBoundingClientRect();
          const dpr = window.devicePixelRatio || 1;

          this.canvas.width = rect.width * dpr;
          this.canvas.height = rect.height * dpr;
          this.canvas.style.width = rect.width + 'px';
          this.canvas.style.height = rect.height + 'px';

          this.ctx.scale(dpr, dpr);
          this.render();
        };

        resizeCanvas();
        window.addEventListener('resize', resizeCanvas);
      }

      setupEventListeners() {
        this.canvas.addEventListener('mousedown', (e) => this.startDrawing(e));
        this.canvas.addEventListener('mousemove', (e) => this.updateDrawing(e));
        this.canvas.addEventListener('mouseup', (e) => this.endDrawing(e));
        this.canvas.addEventListener('mouseout', (e) => this.endDrawing(e));
      }

      setupUI() {
        // 绑定控件事件
        document.getElementById('dirtyRect').addEventListener('change', (e) => {
          this.config.useDirtyRect = e.target.checked;
        });

        document.getElementById('layeredRender').addEventListener('change', (e) => {
          this.config.useLayeredRender = e.target.checked;
        });

        document.getElementById('adaptiveQuality').addEventListener('change', (e) => {
          this.config.useAdaptiveQuality = e.target.checked;
          this.scheduler.adaptiveQuality.enabled = e.target.checked;
        });

        document.getElementById('targetFPS').addEventListener('change', (e) => {
          this.config.targetFPS = parseInt(e.target.value);
          this.scheduler.animationManager.setTargetFPS(this.config.targetFPS);
        });
      }

      startPerformanceMonitoring() {
        this.memoryManager.startMonitoring();

        // 定期更新性能面板
        setInterval(() => {
          this.updatePerformancePanel();
        }, 500);
      }

      startDrawing(event) {
        this.isDrawing = true;
        const point = this.getCanvasPoint(event);

        this.currentStroke = {
          id: Date.now() + Math.random(),
          points: [point],
          color: document.getElementById('brushColor').value,
          size: parseInt(document.getElementById('brushSize').value),
          tool: document.getElementById('toolType').value
        };

        this.analyzer.startFrame();
      }

      updateDrawing(event) {
        if (!this.isDrawing || !this.currentStroke) return;

        const point = this.getCanvasPoint(event);
        this.currentStroke.points.push(point);

        // 增量渲染当前笔画
        if (this.config.useDirtyRect) {
          const lastPoint = this.currentStroke.points[this.currentStroke.points.length - 2];
          if (lastPoint) {
            this.incrementalRenderer.dirtyManager.addDirtyRegion(
              Math.min(lastPoint.x, point.x) - this.currentStroke.size,
              Math.min(lastPoint.y, point.y) - this.currentStroke.size,
              Math.abs(point.x - lastPoint.x) + this.currentStroke.size * 2,
              Math.abs(point.y - lastPoint.y) + this.currentStroke.size * 2
            );
          }
        }

        this.render();
        this.analyzer.recordDrawCall();
      }

      endDrawing(event) {
        if (!this.isDrawing) return;

        this.isDrawing = false;

        if (this.currentStroke && this.currentStroke.points.length > 1) {
          this.elements.push(this.currentStroke);

          if (this.config.useLayeredRender) {
            this.layeredRenderer.addElementToLayer('drawing', this.currentStroke);
          } else {
            this.incrementalRenderer.addElement(this.currentStroke);
          }
        }

        this.currentStroke = null;
        this.analyzer.endFrame();
      }

      render() {
        const startTime = performance.now();

        if (this.config.useLayeredRender) {
          this.layeredRenderer.render();
        } else if (this.config.useDirtyRect) {
          this.incrementalRenderer.render();
        } else {
          this.fullRender();
        }

        // 绘制当前笔画
        if (this.currentStroke) {
          this.drawStroke(this.ctx, this.currentStroke);
        }

        const renderTime = performance.now() - startTime;
        document.getElementById('renderTime').textContent = renderTime.toFixed(2) + 'ms';
      }

      fullRender() {
        this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

        this.elements.forEach(element => {
          this.drawStroke(this.ctx, element);
        });
      }

      drawStroke(ctx, stroke) {
        if (stroke.points.length < 2) return;

        ctx.save();
        ctx.strokeStyle = stroke.color;
        ctx.lineWidth = stroke.size;
        ctx.lineCap = 'round';
        ctx.lineJoin = 'round';

        if (stroke.tool === 'eraser') {
          ctx.globalCompositeOperation = 'destination-out';
        }

        ctx.beginPath();
        ctx.moveTo(stroke.points[0].x, stroke.points[0].y);

        for (let i = 1; i < stroke.points.length; i++) {
          ctx.lineTo(stroke.points[i].x, stroke.points[i].y);
        }

        ctx.stroke();
        ctx.restore();
      }

      getCanvasPoint(event) {
        const rect = this.canvas.getBoundingClientRect();
        return {
          x: event.clientX - rect.left,
          y: event.clientY - rect.top
        };
      }

      updatePerformancePanel() {
        const stats = this.analyzer.getReport();
        const memoryStats = this.memoryManager.getMemoryStatus();
        const schedulerState = this.scheduler.getSchedulerState();

        document.getElementById('fpsValue').textContent = stats.fps;
        document.getElementById('elementCount').textContent = this.elements.length;
        document.getElementById('dirtyRegions').textContent =
          this.incrementalRenderer.dirtyManager.dirtyRegions.length;

        if (memoryStats.heap) {
          document.getElementById('memoryUsage').textContent = memoryStats.heap.current.used;
        }

        document.getElementById('qualityLevel').textContent =
          Math.round(schedulerState.currentQuality * 100) + '%';
      }

      // 全局函数接口
      generateTestElements() {
        const count = parseInt(document.getElementById('elementCount').value);
        console.log(`Generating ${count} test elements...`);

        for (let i = 0; i < count; i++) {
          const element = {
            id: Date.now() + i,
            points: this.generateRandomPath(),
            color: `hsl(${Math.random() * 360}, 70%, 60%)`,
            size: Math.random() * 10 + 2,
            tool: 'brush'
          };

          this.elements.push(element);

          if (this.config.useLayeredRender) {
            this.layeredRenderer.addElementToLayer('drawing', element);
          } else {
            this.incrementalRenderer.addElement(element);
          }
        }

        this.render();
      }

      generateRandomPath() {
        const points = [];
        const centerX = Math.random() * this.canvas.width;
        const centerY = Math.random() * this.canvas.height;
        const pointCount = 5 + Math.random() * 20;

        for (let i = 0; i < pointCount; i++) {
          points.push({
            x: centerX + (Math.random() - 0.5) * 100,
            y: centerY + (Math.random() - 0.5) * 100
          });
        }

        return points;
      }

      runPerformanceTest() {
        const tester = new CanvasPerformanceTester(this.canvas);
        const results = tester.runFullTest();
        console.log('Performance test results:', results);
        alert('性能测试完成，请查看控制台输出');
      }

      clearCanvas() {
        this.elements = [];
        this.incrementalRenderer.elements = [];
        this.layeredRenderer.layers.forEach(layer => {
          layer.elements = [];
          layer.isDirty = true;
        });
        this.render();
      }

      toggleMemoryMonitoring() {
        if (this.memoryManager.leakDetector.isMonitoring) {
          this.memoryManager.leakDetector.stopMonitoring();
        } else {
          this.memoryManager.leakDetector.startMonitoring();
        }
      }

      performCleanup() {
        this.memoryManager.performCleanup();
        console.log('Manual cleanup performed');
      }

      showMemoryReport() {
        const status = this.memoryManager.getMemoryStatus();
        console.log('Memory report:', status);
        alert('内存报告已输出到控制台');
      }
    }

    // 全局函数
    function generateTestElements() {
      app.generateTestElements();
    }

    function runPerformanceTest() {
      app.runPerformanceTest();
    }

    function clearCanvas() {
      app.clearCanvas();
    }

    function toggleMemoryMonitoring() {
      app.toggleMemoryMonitoring();
    }

    function performCleanup() {
      app.performCleanup();
    }

    function showMemoryReport() {
      app.showMemoryReport();
    }

    // 初始化应用
    const app = new HighPerformanceDrawingApp('canvas');

    console.log('高性能Canvas画板应用已启动');
    console.log('- 支持脏矩形更新');
    console.log('- 支持分层渲染');
    console.log('- 支持自适应质量');
    console.log('- 内置性能监控');
    console.log('- 内存泄漏检测');
  </script>
</body>
</html>
```

## 7. Excalidraw 中的性能优化

### 7.1 Excalidraw 的渲染优化策略

```typescript
// packages/excalidraw/renderer/index.ts
export const renderScene = (
  elements: readonly ExcalidrawElement[],
  appState: AppState,
  canvas: HTMLCanvasElement
) => {
  const context = canvas.getContext('2d')!;
  const renderConfig = {
    viewBackgroundColor: appState.viewBackgroundColor,
    zoom: appState.zoom,
    scrollX: appState.scrollX,
    scrollY: appState.scrollY,
    isExporting: false
  };

  // 视口裁剪优化
  const visibleElements = getVisibleElements(
    elements,
    canvas.width,
    canvas.height,
    appState
  );

  // 分层渲染
  const staticElements = visibleElements.filter(el => !el.isBeingEdited);
  const dynamicElements = visibleElements.filter(el => el.isBeingEdited);

  // 静态层缓存
  if (shouldUpdateStaticLayer(staticElements, appState)) {
    renderStaticLayer(staticElements, context, renderConfig);
  }

  // 动态层实时渲染
  renderDynamicLayer(dynamicElements, context, renderConfig);
};

// 视口裁剪
const getVisibleElements = (
  elements: readonly ExcalidrawElement[],
  canvasWidth: number,
  canvasHeight: number,
  appState: AppState
): ExcalidrawElement[] => {
  const viewportBounds = getViewportBounds(canvasWidth, canvasHeight, appState);

  return elements.filter(element => {
    const elementBounds = getElementBounds(element);
    return boundsIntersect(viewportBounds, elementBounds);
  });
};

// 静态层缓存策略
let staticLayerCache: {
  canvas: HTMLCanvasElement;
  elements: ExcalidrawElement[];
  appState: Partial<AppState>;
} | null = null;

const shouldUpdateStaticLayer = (
  elements: ExcalidrawElement[],
  appState: AppState
): boolean => {
  if (!staticLayerCache) return true;

  // 检查元素是否发生变化
  if (elements.length !== staticLayerCache.elements.length) {
    return true;
  }

  // 检查视图状态是否变化
  const relevantAppState = {
    zoom: appState.zoom,
    scrollX: appState.scrollX,
    scrollY: appState.scrollY
  };

  return !isEqual(relevantAppState, staticLayerCache.appState);
};
```

## 8. 练习题

### 8.1 基础练习

1. **实现基础性能监控器**
   - 监控 FPS、渲染时间、内存使用
   - 实现性能警告系统
   - 生成性能报告

2. **脏矩形算法优化**
   - 实现更智能的区域合并算法
   - 支持不规则形状的脏区域
   - 添加区域优先级机制

3. **离屏Canvas实践**
   - 实现图形缓存系统
   - 支持动态缓存策略
   - 内存使用优化

### 8.2 进阶练习

1. **完整的分层渲染系统**
   ```javascript
   class AdvancedLayerSystem {
     constructor() {
       this.layers = new Map();
       this.renderOrder = [];
       this.optimizer = new LayerOptimizer();
     }

     // 实现智能图层合并
     optimizeLayers() {
       // TODO: 分析图层使用情况，自动合并相似图层
     }

     // 实现条件渲染
     conditionalRender(viewport) {
       // TODO: 根据视口和性能状况选择渲染策略
     }
   }
   ```

## 9. 思考题

1. **什么情况下脏矩形算法反而会降低性能？**
   - 大量小区域更新
   - 区域合并开销
   - 裁剪区域设置成本

2. **如何平衡渲染质量和性能？**
   - 自适应质量系统
   - 用户偏好设置
   - 设备性能检测

3. **移动设备上的性能优化有什么特殊考虑？**
   - 内存限制更严格
   - GPU性能差异大
   - 电池消耗考虑

4. **Canvas 性能 vs SVG 性能的权衡？**
   - 元素数量临界点
   - 交互复杂度影响
   - 动画性能对比

## 10. 总结

### 核心要点

1. **性能监控**
   - 建立完善的性能指标体系
   - 实时监控和预警机制
   - 数据驱动的优化决策

2. **渲染优化**
   - 脏矩形算法减少重绘
   - 分层渲染提高效率
   - 离屏Canvas缓存复杂图形

3. **动画优化**
   - RequestAnimationFrame最佳实践
   - 帧率控制和自适应质量
   - 智能渲染调度

4. **内存管理**
   - 对象池模式减少GC压力
   - 内存泄漏检测和预防
   - 资源及时清理

### 性能优化清单

- [ ] 实现性能监控系统
- [ ] 应用脏矩形算法
- [ ] 使用分层渲染策略
- [ ] 优化动画循环
- [ ] 建立内存管理机制
- [ ] 添加自适应质量控制
- [ ] 实现视口裁剪
- [ ] 使用对象池模式

## 11. 参考资源

- [Canvas Performance Tips](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Optimizing_canvas)
- [RequestAnimationFrame Guide](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame)
- [JavaScript Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)
- [OffscreenCanvas API](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)

---

**上一章**：[Canvas 事件与交互](./04-canvas-interaction.md)
**下一章**：[Excalidraw 项目架构 →](./06-excalidraw-structure.md)