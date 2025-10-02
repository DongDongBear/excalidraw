# 第四章：Canvas 事件与交互

## 学习目标

- [ ] 掌握 Canvas 事件处理的基本原理
- [ ] 理解坐标转换（屏幕坐标 ↔ Canvas 坐标）
- [ ] 实现各种碰撞检测算法
- [ ] 掌握拖拽操作的实现原理
- [ ] 了解手势识别的基础技术
- [ ] 构建一个完整的交互式图形编辑器

## 1. Canvas 事件处理基础

### 1.1 Canvas 事件模型

Canvas 本身不支持对象级别的事件，所有事件都绑定在 Canvas 元素上。我们需要手动实现：
1. **坐标转换**：将鼠标坐标转换为 Canvas 坐标
2. **碰撞检测**：判断点击了哪个图形元素
3. **事件分发**：将事件传递给相应的图形对象

```javascript
class CanvasEventSystem {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.elements = [];
    this.selectedElement = null;

    this.setupEventListeners();
  }

  setupEventListeners() {
    // 鼠标事件
    this.canvas.addEventListener('mousedown', (e) => this.handleMouseDown(e));
    this.canvas.addEventListener('mousemove', (e) => this.handleMouseMove(e));
    this.canvas.addEventListener('mouseup', (e) => this.handleMouseUp(e));
    this.canvas.addEventListener('click', (e) => this.handleClick(e));
    this.canvas.addEventListener('dblclick', (e) => this.handleDoubleClick(e));

    // 触摸事件
    this.canvas.addEventListener('touchstart', (e) => this.handleTouchStart(e));
    this.canvas.addEventListener('touchmove', (e) => this.handleTouchMove(e));
    this.canvas.addEventListener('touchend', (e) => this.handleTouchEnd(e));

    // 键盘事件（需要设置 tabindex 让 Canvas 获得焦点）
    this.canvas.tabIndex = 0;
    this.canvas.addEventListener('keydown', (e) => this.handleKeyDown(e));
    this.canvas.addEventListener('keyup', (e) => this.handleKeyUp(e));

    // 滚轮事件
    this.canvas.addEventListener('wheel', (e) => this.handleWheel(e));

    // 右键菜单
    this.canvas.addEventListener('contextmenu', (e) => this.handleContextMenu(e));
  }

  // 基础事件处理方法
  handleMouseDown(event) {
    const point = this.getCanvasPoint(event);
    const element = this.getElementAtPoint(point.x, point.y);

    console.log('Mouse down at:', point, 'Element:', element);
  }

  handleMouseMove(event) {
    const point = this.getCanvasPoint(event);
    // 处理鼠标移动逻辑
  }

  handleMouseUp(event) {
    const point = this.getCanvasPoint(event);
    // 处理鼠标释放逻辑
  }

  handleClick(event) {
    const point = this.getCanvasPoint(event);
    // 处理点击逻辑
  }

  handleDoubleClick(event) {
    const point = this.getCanvasPoint(event);
    // 处理双击逻辑
  }

  handleKeyDown(event) {
    console.log('Key down:', event.key);
  }

  handleKeyUp(event) {
    console.log('Key up:', event.key);
  }

  handleWheel(event) {
    event.preventDefault();
    const point = this.getCanvasPoint(event);
    const delta = event.deltaY;
    console.log('Wheel at:', point, 'Delta:', delta);
  }

  handleContextMenu(event) {
    event.preventDefault();
    const point = this.getCanvasPoint(event);
    console.log('Right click at:', point);
  }
}
```

### 1.2 触摸事件处理

移动设备的触摸事件需要特殊处理：

```javascript
class TouchEventHandler {
  constructor(canvas) {
    this.canvas = canvas;
    this.activeTouches = new Map();
    this.gestureState = {
      isScaling: false,
      isRotating: false,
      initialDistance: 0,
      initialAngle: 0
    };
  }

  handleTouchStart(event) {
    event.preventDefault();

    Array.from(event.changedTouches).forEach(touch => {
      const point = this.getTouchPoint(touch);
      this.activeTouches.set(touch.identifier, {
        startPoint: point,
        currentPoint: point,
        startTime: Date.now()
      });
    });

    // 检测手势
    if (this.activeTouches.size === 2) {
      this.startTwoFingerGesture();
    }
  }

  handleTouchMove(event) {
    event.preventDefault();

    Array.from(event.changedTouches).forEach(touch => {
      if (this.activeTouches.has(touch.identifier)) {
        const point = this.getTouchPoint(touch);
        const touchData = this.activeTouches.get(touch.identifier);
        touchData.currentPoint = point;
      }
    });

    // 处理手势
    if (this.activeTouches.size === 2) {
      this.processTwoFingerGesture();
    } else if (this.activeTouches.size === 1) {
      this.processOneFingerGesture();
    }
  }

  handleTouchEnd(event) {
    event.preventDefault();

    Array.from(event.changedTouches).forEach(touch => {
      this.activeTouches.delete(touch.identifier);
    });

    if (this.activeTouches.size < 2) {
      this.endTwoFingerGesture();
    }
  }

  getTouchPoint(touch) {
    const rect = this.canvas.getBoundingClientRect();
    return {
      x: touch.clientX - rect.left,
      y: touch.clientY - rect.top
    };
  }

  startTwoFingerGesture() {
    const touches = Array.from(this.activeTouches.values());
    if (touches.length !== 2) return;

    const [touch1, touch2] = touches;

    // 计算初始距离（用于缩放）
    this.gestureState.initialDistance = this.calculateDistance(
      touch1.currentPoint, touch2.currentPoint
    );

    // 计算初始角度（用于旋转）
    this.gestureState.initialAngle = this.calculateAngle(
      touch1.currentPoint, touch2.currentPoint
    );

    this.gestureState.isScaling = true;
    this.gestureState.isRotating = true;
  }

  processTwoFingerGesture() {
    const touches = Array.from(this.activeTouches.values());
    if (touches.length !== 2) return;

    const [touch1, touch2] = touches;

    // 计算当前距离和角度
    const currentDistance = this.calculateDistance(
      touch1.currentPoint, touch2.currentPoint
    );
    const currentAngle = this.calculateAngle(
      touch1.currentPoint, touch2.currentPoint
    );

    // 缩放系数
    const scaleRatio = currentDistance / this.gestureState.initialDistance;

    // 旋转角度
    const rotationDelta = currentAngle - this.gestureState.initialAngle;

    // 中心点
    const centerPoint = {
      x: (touch1.currentPoint.x + touch2.currentPoint.x) / 2,
      y: (touch1.currentPoint.y + touch2.currentPoint.y) / 2
    };

    console.log('Gesture:', {
      scale: scaleRatio,
      rotation: rotationDelta * 180 / Math.PI,
      center: centerPoint
    });
  }

  endTwoFingerGesture() {
    this.gestureState.isScaling = false;
    this.gestureState.isRotating = false;
  }

  processOneFingerGesture() {
    const touch = Array.from(this.activeTouches.values())[0];
    const deltaX = touch.currentPoint.x - touch.startPoint.x;
    const deltaY = touch.currentPoint.y - touch.startPoint.y;

    console.log('Pan:', { deltaX, deltaY });
  }

  calculateDistance(point1, point2) {
    const dx = point2.x - point1.x;
    const dy = point2.y - point1.y;
    return Math.sqrt(dx * dx + dy * dy);
  }

  calculateAngle(point1, point2) {
    return Math.atan2(point2.y - point1.y, point2.x - point1.x);
  }
}
```

## 2. 坐标转换

### 2.1 基础坐标转换

```javascript
class CoordinateConverter {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.viewportTransform = {
      x: 0,
      y: 0,
      scale: 1
    };
  }

  // 屏幕坐标 → Canvas 坐标
  screenToCanvas(screenX, screenY) {
    const rect = this.canvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;

    return {
      x: (screenX - rect.left) * dpr,
      y: (screenY - rect.top) * dpr
    };
  }

  // Canvas 坐标 → 世界坐标（考虑视口变换）
  canvasToWorld(canvasX, canvasY) {
    return {
      x: (canvasX - this.viewportTransform.x) / this.viewportTransform.scale,
      y: (canvasY - this.viewportTransform.y) / this.viewportTransform.scale
    };
  }

  // 世界坐标 → Canvas 坐标
  worldToCanvas(worldX, worldY) {
    return {
      x: worldX * this.viewportTransform.scale + this.viewportTransform.x,
      y: worldY * this.viewportTransform.scale + this.viewportTransform.y
    };
  }

  // 屏幕坐标 → 世界坐标（完整转换链）
  screenToWorld(screenX, screenY) {
    const canvasPoint = this.screenToCanvas(screenX, screenY);
    return this.canvasToWorld(canvasPoint.x, canvasPoint.y);
  }

  // 考虑 Canvas 变换的坐标转换
  transformPoint(x, y, transform) {
    const [a, b, c, d, e, f] = transform;
    return {
      x: a * x + c * y + e,
      y: b * x + d * y + f
    };
  }

  // 逆向变换
  inverseTransformPoint(x, y, transform) {
    const [a, b, c, d, e, f] = transform;
    const det = a * d - b * c;

    if (Math.abs(det) < 1e-10) {
      throw new Error('Transform matrix is not invertible');
    }

    const invA = d / det;
    const invB = -b / det;
    const invC = -c / det;
    const invD = a / det;
    const invE = (c * f - d * e) / det;
    const invF = (b * e - a * f) / det;

    return {
      x: invA * x + invC * y + invE,
      y: invB * x + invD * y + invF
    };
  }
}
```

### 2.2 复杂场景的坐标处理

```javascript
class AdvancedCoordinates {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    this.transformStack = [];
  }

  // 处理嵌套变换的坐标转换
  getTransformStack() {
    // 模拟复杂的变换层次
    return [
      { type: 'viewport', x: 100, y: 50, scale: 1.5 },
      { type: 'group', x: 20, y: 30, rotation: Math.PI/4 },
      { type: 'element', x: 10, y: 15, scale: 0.8 }
    ];
  }

  // 应用变换栈
  applyTransformStack(x, y, transformStack) {
    let point = { x, y };

    for (const transform of transformStack) {
      switch (transform.type) {
        case 'viewport':
          point.x = (point.x - transform.x) / transform.scale;
          point.y = (point.y - transform.y) / transform.scale;
          break;
        case 'group':
          // 先平移到原点，旋转，再平移回去
          const cos = Math.cos(-transform.rotation);
          const sin = Math.sin(-transform.rotation);
          const tempX = point.x - transform.x;
          const tempY = point.y - transform.y;
          point.x = tempX * cos - tempY * sin;
          point.y = tempX * sin + tempY * cos;
          break;
        case 'element':
          point.x = (point.x - transform.x) / transform.scale;
          point.y = (point.y - transform.y) / transform.scale;
          break;
      }
    }

    return point;
  }

  // 处理高 DPI 屏幕的坐标
  handleHighDPI(event) {
    const rect = this.canvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;

    // CSS 像素坐标
    const cssX = event.clientX - rect.left;
    const cssY = event.clientY - rect.top;

    // 物理像素坐标
    const physicalX = cssX * dpr;
    const physicalY = cssY * dpr;

    // 考虑 Canvas 内部缩放
    const canvasScaleX = this.canvas.width / rect.width;
    const canvasScaleY = this.canvas.height / rect.height;

    return {
      css: { x: cssX, y: cssY },
      physical: { x: physicalX, y: physicalY },
      canvas: {
        x: cssX * canvasScaleX,
        y: cssY * canvasScaleY
      }
    };
  }
}
```

## 3. 碰撞检测算法

### 3.1 基础碰撞检测

```javascript
class CollisionDetection {
  // 点与矩形碰撞
  static pointInRect(px, py, rx, ry, rw, rh) {
    return px >= rx && px <= rx + rw && py >= ry && py <= ry + rh;
  }

  // 点与圆形碰撞
  static pointInCircle(px, py, cx, cy, radius) {
    const dx = px - cx;
    const dy = py - cy;
    return dx * dx + dy * dy <= radius * radius;
  }

  // 点与椭圆碰撞
  static pointInEllipse(px, py, cx, cy, rx, ry) {
    const dx = (px - cx) / rx;
    const dy = (py - cy) / ry;
    return dx * dx + dy * dy <= 1;
  }

  // 点与多边形碰撞（射线法）
  static pointInPolygon(px, py, vertices) {
    let inside = false;

    for (let i = 0, j = vertices.length - 1; i < vertices.length; j = i++) {
      const xi = vertices[i].x, yi = vertices[i].y;
      const xj = vertices[j].x, yj = vertices[j].y;

      if (((yi > py) !== (yj > py)) &&
          (px < (xj - xi) * (py - yi) / (yj - yi) + xi)) {
        inside = !inside;
      }
    }

    return inside;
  }

  // 点与旋转矩形碰撞
  static pointInRotatedRect(px, py, rect) {
    const { x, y, width, height, rotation } = rect;

    // 将点旋转到矩形的本地坐标系
    const cos = Math.cos(-rotation);
    const sin = Math.sin(-rotation);

    const localX = cos * (px - x) - sin * (py - y);
    const localY = sin * (px - x) + cos * (py - y);

    return this.pointInRect(localX, localY, -width/2, -height/2, width, height);
  }

  // 点与线段距离
  static pointToLineDistance(px, py, x1, y1, x2, y2) {
    const A = px - x1;
    const B = py - y1;
    const C = x2 - x1;
    const D = y2 - y1;

    const dot = A * C + B * D;
    const lenSq = C * C + D * D;

    if (lenSq === 0) {
      // 线段退化为点
      return Math.sqrt(A * A + B * B);
    }

    let param = dot / lenSq;
    param = Math.max(0, Math.min(1, param));

    const xx = x1 + param * C;
    const yy = y1 + param * D;

    const dx = px - xx;
    const dy = py - yy;

    return Math.sqrt(dx * dx + dy * dy);
  }

  // 点与路径碰撞（使用 Canvas API）
  static pointInPath(ctx, px, py, path) {
    return ctx.isPointInPath(path, px, py);
  }

  // 点与描边路径碰撞
  static pointInStroke(ctx, px, py, path, lineWidth) {
    return ctx.isPointInStroke(path, px, py);
  }
}
```

### 3.2 高级碰撞检测

```javascript
class AdvancedCollisionDetection {
  // 矩形与矩形碰撞（AABB）
  static rectIntersectsRect(rect1, rect2) {
    return !(rect1.x > rect2.x + rect2.width ||
             rect1.x + rect1.width < rect2.x ||
             rect1.y > rect2.y + rect2.height ||
             rect1.y + rect1.height < rect2.y);
  }

  // 圆与圆碰撞
  static circleIntersectsCircle(c1, c2) {
    const dx = c2.x - c1.x;
    const dy = c2.y - c1.y;
    const distance = Math.sqrt(dx * dx + dy * dy);
    return distance < c1.radius + c2.radius;
  }

  // 线段与线段相交
  static lineIntersectsLine(line1, line2) {
    const { x1: ax1, y1: ay1, x2: ax2, y2: ay2 } = line1;
    const { x1: bx1, y1: by1, x2: bx2, y2: by2 } = line2;

    const denom = (ax2 - ax1) * (by2 - by1) - (ay2 - ay1) * (bx2 - bx1);

    if (Math.abs(denom) < 1e-10) return false; // 平行线

    const t = ((bx1 - ax1) * (by2 - by1) - (by1 - ay1) * (bx2 - bx1)) / denom;
    const u = ((bx1 - ax1) * (ay2 - ay1) - (by1 - ay1) * (ax2 - ax1)) / denom;

    return t >= 0 && t <= 1 && u >= 0 && u <= 1;
  }

  // 分离轴定理（SAT）检测凸多边形碰撞
  static polygonIntersectsPolygon(poly1, poly2) {
    const polygons = [poly1, poly2];

    for (let i = 0; i < 2; i++) {
      const polygon = polygons[i];

      for (let j = 0; j < polygon.length; j++) {
        const vertex1 = polygon[j];
        const vertex2 = polygon[(j + 1) % polygon.length];

        // 计算法线（垂直向量）
        const normal = {
          x: vertex2.y - vertex1.y,
          y: vertex1.x - vertex2.x
        };

        // 归一化法线
        const length = Math.sqrt(normal.x * normal.x + normal.y * normal.y);
        normal.x /= length;
        normal.y /= length;

        // 投影两个多边形到法线上
        let min1 = Infinity, max1 = -Infinity;
        let min2 = Infinity, max2 = -Infinity;

        for (const vertex of poly1) {
          const projection = vertex.x * normal.x + vertex.y * normal.y;
          min1 = Math.min(min1, projection);
          max1 = Math.max(max1, projection);
        }

        for (const vertex of poly2) {
          const projection = vertex.x * normal.x + vertex.y * normal.y;
          min2 = Math.min(min2, projection);
          max2 = Math.max(max2, projection);
        }

        // 检查投影是否重叠
        if (max1 < min2 || max2 < min1) {
          return false; // 找到分离轴，不相交
        }
      }
    }

    return true; // 没有找到分离轴，相交
  }

  // 空间分割优化（四叉树）
  static createQuadTree(bounds, maxObjects = 10, maxLevels = 5, level = 0) {
    return {
      level,
      bounds,
      objects: [],
      nodes: [],

      clear() {
        this.objects = [];
        for (let i = 0; i < this.nodes.length; i++) {
          if (this.nodes[i]) {
            this.nodes[i].clear();
            this.nodes[i] = null;
          }
        }
        this.nodes = [];
      },

      split() {
        const subWidth = this.bounds.width / 2;
        const subHeight = this.bounds.height / 2;
        const x = this.bounds.x;
        const y = this.bounds.y;

        this.nodes[0] = this.constructor.createQuadTree({
          x: x + subWidth,
          y: y,
          width: subWidth,
          height: subHeight
        }, maxObjects, maxLevels, this.level + 1);

        this.nodes[1] = this.constructor.createQuadTree({
          x: x,
          y: y,
          width: subWidth,
          height: subHeight
        }, maxObjects, maxLevels, this.level + 1);

        this.nodes[2] = this.constructor.createQuadTree({
          x: x,
          y: y + subHeight,
          width: subWidth,
          height: subHeight
        }, maxObjects, maxLevels, this.level + 1);

        this.nodes[3] = this.constructor.createQuadTree({
          x: x + subWidth,
          y: y + subHeight,
          width: subWidth,
          height: subHeight
        }, maxObjects, maxLevels, this.level + 1);
      },

      getIndex(rect) {
        let index = -1;
        const verticalMidpoint = this.bounds.x + this.bounds.width / 2;
        const horizontalMidpoint = this.bounds.y + this.bounds.height / 2;

        const topQuadrant = (rect.y < horizontalMidpoint && rect.y + rect.height < horizontalMidpoint);
        const bottomQuadrant = (rect.y > horizontalMidpoint);

        if (rect.x < verticalMidpoint && rect.x + rect.width < verticalMidpoint) {
          if (topQuadrant) {
            index = 1;
          } else if (bottomQuadrant) {
            index = 2;
          }
        } else if (rect.x > verticalMidpoint) {
          if (topQuadrant) {
            index = 0;
          } else if (bottomQuadrant) {
            index = 3;
          }
        }

        return index;
      },

      insert(rect) {
        if (this.nodes.length > 0) {
          const index = this.getIndex(rect);
          if (index !== -1) {
            this.nodes[index].insert(rect);
            return;
          }
        }

        this.objects.push(rect);

        if (this.objects.length > maxObjects && this.level < maxLevels) {
          if (this.nodes.length === 0) {
            this.split();
          }

          let i = 0;
          while (i < this.objects.length) {
            const index = this.getIndex(this.objects[i]);
            if (index !== -1) {
              this.nodes[index].insert(this.objects.splice(i, 1)[0]);
            } else {
              i++;
            }
          }
        }
      },

      retrieve(rect) {
        const returnObjects = this.objects.slice();

        if (this.nodes.length > 0) {
          const index = this.getIndex(rect);
          if (index !== -1) {
            returnObjects.push(...this.nodes[index].retrieve(rect));
          } else {
            for (let i = 0; i < this.nodes.length; i++) {
              returnObjects.push(...this.nodes[i].retrieve(rect));
            }
          }
        }

        return returnObjects;
      }
    };
  }
}
```

## 4. 拖拽操作实现

### 4.1 基础拖拽

```javascript
class DragSystem {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');

    this.isDragging = false;
    this.dragStartPoint = null;
    this.dragTarget = null;
    this.dragOffset = { x: 0, y: 0 };

    this.elements = [];
    this.setupDragEvents();
  }

  setupDragEvents() {
    this.canvas.addEventListener('mousedown', (e) => this.startDrag(e));
    this.canvas.addEventListener('mousemove', (e) => this.updateDrag(e));
    this.canvas.addEventListener('mouseup', (e) => this.endDrag(e));

    // 防止图片拖拽等默认行为
    this.canvas.addEventListener('dragstart', (e) => e.preventDefault());
  }

  startDrag(event) {
    const point = this.getCanvasPoint(event);
    const element = this.getElementAtPoint(point.x, point.y);

    if (element) {
      this.isDragging = true;
      this.dragStartPoint = point;
      this.dragTarget = element;

      // 计算鼠标相对于元素的偏移
      this.dragOffset = {
        x: point.x - element.x,
        y: point.y - element.y
      };

      // 改变鼠标样式
      this.canvas.style.cursor = 'grabbing';

      // 触发拖拽开始事件
      this.onDragStart(element, point);
    }
  }

  updateDrag(event) {
    const point = this.getCanvasPoint(event);

    if (this.isDragging && this.dragTarget) {
      // 计算新位置
      const newX = point.x - this.dragOffset.x;
      const newY = point.y - this.dragOffset.y;

      // 应用约束（如网格吸附）
      const constrainedPos = this.applyConstraints(newX, newY);

      // 更新元素位置
      this.dragTarget.x = constrainedPos.x;
      this.dragTarget.y = constrainedPos.y;

      // 重绘
      this.render();

      // 触发拖拽更新事件
      this.onDragUpdate(this.dragTarget, point);
    } else {
      // 鼠标悬停检测
      const element = this.getElementAtPoint(point.x, point.y);
      this.canvas.style.cursor = element ? 'grab' : 'default';
    }
  }

  endDrag(event) {
    if (this.isDragging) {
      const point = this.getCanvasPoint(event);

      this.isDragging = false;
      this.canvas.style.cursor = 'default';

      // 触发拖拽结束事件
      this.onDragEnd(this.dragTarget, point);

      this.dragTarget = null;
      this.dragStartPoint = null;
    }
  }

  // 约束处理
  applyConstraints(x, y) {
    // 网格吸附
    const gridSize = 10;
    if (this.snapToGrid) {
      x = Math.round(x / gridSize) * gridSize;
      y = Math.round(y / gridSize) * gridSize;
    }

    // 边界限制
    if (this.constrainToBounds) {
      const bounds = this.getBounds();
      x = Math.max(bounds.left, Math.min(bounds.right - this.dragTarget.width, x));
      y = Math.max(bounds.top, Math.min(bounds.bottom - this.dragTarget.height, y));
    }

    return { x, y };
  }

  // 事件回调
  onDragStart(element, point) {
    console.log('Drag started:', element, point);
  }

  onDragUpdate(element, point) {
    console.log('Drag updated:', element, point);
  }

  onDragEnd(element, point) {
    console.log('Drag ended:', element, point);
  }

  getCanvasPoint(event) {
    const rect = this.canvas.getBoundingClientRect();
    return {
      x: event.clientX - rect.left,
      y: event.clientY - rect.top
    };
  }

  getElementAtPoint(x, y) {
    // 从后往前遍历（最上层元素优先）
    for (let i = this.elements.length - 1; i >= 0; i--) {
      const element = this.elements[i];
      if (this.pointInElement(x, y, element)) {
        return element;
      }
    }
    return null;
  }

  pointInElement(x, y, element) {
    return x >= element.x && x <= element.x + element.width &&
           y >= element.y && y <= element.y + element.height;
  }
}
```

### 4.2 多选拖拽

```javascript
class MultiSelectDragSystem extends DragSystem {
  constructor(canvas) {
    super(canvas);
    this.selectedElements = new Set();
    this.selectionBox = null;
    this.isBoxSelecting = false;
  }

  startDrag(event) {
    const point = this.getCanvasPoint(event);
    const element = this.getElementAtPoint(point.x, point.y);

    if (element) {
      // 处理多选逻辑
      if (!event.ctrlKey && !event.metaKey) {
        if (!this.selectedElements.has(element)) {
          this.selectedElements.clear();
          this.selectedElements.add(element);
        }
      } else {
        // Ctrl/Cmd 键多选
        if (this.selectedElements.has(element)) {
          this.selectedElements.delete(element);
        } else {
          this.selectedElements.add(element);
        }
      }

      if (this.selectedElements.has(element)) {
        this.startMultiDrag(point);
      }
    } else {
      // 开始框选
      this.startBoxSelection(point);
    }

    this.render();
  }

  startMultiDrag(point) {
    this.isDragging = true;
    this.dragStartPoint = point;

    // 记录所有选中元素的初始位置
    this.dragTargets = Array.from(this.selectedElements).map(element => ({
      element,
      startX: element.x,
      startY: element.y
    }));

    this.canvas.style.cursor = 'grabbing';
  }

  startBoxSelection(point) {
    this.isBoxSelecting = true;
    this.dragStartPoint = point;
    this.selectionBox = {
      x: point.x,
      y: point.y,
      width: 0,
      height: 0
    };
  }

  updateDrag(event) {
    const point = this.getCanvasPoint(event);

    if (this.isDragging && this.dragTargets) {
      // 多选拖拽
      const deltaX = point.x - this.dragStartPoint.x;
      const deltaY = point.y - this.dragStartPoint.y;

      this.dragTargets.forEach(({ element, startX, startY }) => {
        const newPos = this.applyConstraints(
          startX + deltaX,
          startY + deltaY
        );
        element.x = newPos.x;
        element.y = newPos.y;
      });

      this.render();
    } else if (this.isBoxSelecting) {
      // 框选更新
      this.updateSelectionBox(point);
      this.render();
    } else {
      // 悬停检测
      const element = this.getElementAtPoint(point.x, point.y);
      this.canvas.style.cursor = element ? 'grab' : 'default';
    }
  }

  updateSelectionBox(point) {
    if (!this.selectionBox || !this.dragStartPoint) return;

    this.selectionBox.x = Math.min(this.dragStartPoint.x, point.x);
    this.selectionBox.y = Math.min(this.dragStartPoint.y, point.y);
    this.selectionBox.width = Math.abs(point.x - this.dragStartPoint.x);
    this.selectionBox.height = Math.abs(point.y - this.dragStartPoint.y);

    // 更新选中元素
    this.updateSelection();
  }

  updateSelection() {
    if (!this.selectionBox) return;

    this.selectedElements.clear();

    this.elements.forEach(element => {
      if (this.rectIntersectsElement(this.selectionBox, element)) {
        this.selectedElements.add(element);
      }
    });
  }

  rectIntersectsElement(rect, element) {
    return !(rect.x > element.x + element.width ||
             rect.x + rect.width < element.x ||
             rect.y > element.y + element.height ||
             rect.y + rect.height < element.y);
  }

  endDrag(event) {
    if (this.isDragging) {
      this.isDragging = false;
      this.dragTargets = null;
      this.canvas.style.cursor = 'default';
    }

    if (this.isBoxSelecting) {
      this.isBoxSelecting = false;
      this.selectionBox = null;
    }

    this.render();
  }

  render() {
    const ctx = this.ctx;
    ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);

    // 绘制元素
    this.elements.forEach(element => {
      ctx.save();

      // 选中高亮
      if (this.selectedElements.has(element)) {
        ctx.strokeStyle = '#007acc';
        ctx.lineWidth = 2;
        ctx.strokeRect(element.x - 2, element.y - 2, element.width + 4, element.height + 4);
      }

      // 绘制元素
      ctx.fillStyle = element.color || '#ccc';
      ctx.fillRect(element.x, element.y, element.width, element.height);

      ctx.restore();
    });

    // 绘制选择框
    if (this.selectionBox) {
      ctx.save();
      ctx.strokeStyle = '#007acc';
      ctx.setLineDash([5, 5]);
      ctx.strokeRect(
        this.selectionBox.x,
        this.selectionBox.y,
        this.selectionBox.width,
        this.selectionBox.height
      );
      ctx.restore();
    }
  }
}
```

## 5. 手势识别基础

### 5.1 基础手势识别

```javascript
class GestureRecognizer {
  constructor(canvas) {
    this.canvas = canvas;
    this.gestures = new Map();
    this.currentGesture = null;
    this.gestureThreshold = 10; // 最小手势距离

    this.setupGestureEvents();
  }

  setupGestureEvents() {
    let startPoint = null;
    let path = [];

    this.canvas.addEventListener('mousedown', (e) => {
      startPoint = this.getCanvasPoint(e);
      path = [startPoint];
      this.currentGesture = {
        startTime: Date.now(),
        startPoint,
        path
      };
    });

    this.canvas.addEventListener('mousemove', (e) => {
      if (this.currentGesture) {
        const point = this.getCanvasPoint(e);
        path.push(point);
      }
    });

    this.canvas.addEventListener('mouseup', (e) => {
      if (this.currentGesture) {
        this.currentGesture.endTime = Date.now();
        this.currentGesture.endPoint = this.getCanvasPoint(e);
        this.recognizeGesture(this.currentGesture);
        this.currentGesture = null;
      }
    });
  }

  recognizeGesture(gesture) {
    const { startPoint, endPoint, path, startTime, endTime } = gesture;
    const duration = endTime - startTime;

    // 计算总距离
    const totalDistance = this.calculatePathDistance(path);

    // 计算直线距离
    const straightDistance = this.calculateDistance(startPoint, endPoint);

    if (totalDistance < this.gestureThreshold) {
      this.onTap(startPoint, duration);
      return;
    }

    // 速度计算
    const velocity = totalDistance / duration;

    // 方向性检测
    const direction = this.getGestureDirection(startPoint, endPoint);

    // 圆形手势检测
    if (this.isCircularGesture(path)) {
      this.onCircularGesture(path);
      return;
    }

    // 直线手势检测
    if (straightDistance / totalDistance > 0.8) {
      this.onSwipe(direction, velocity, straightDistance);
      return;
    }

    // 复杂手势检测
    const gestureType = this.classifyComplexGesture(path);
    this.onComplexGesture(gestureType, path);
  }

  calculatePathDistance(path) {
    let distance = 0;
    for (let i = 1; i < path.length; i++) {
      distance += this.calculateDistance(path[i-1], path[i]);
    }
    return distance;
  }

  calculateDistance(p1, p2) {
    const dx = p2.x - p1.x;
    const dy = p2.y - p1.y;
    return Math.sqrt(dx * dx + dy * dy);
  }

  getGestureDirection(start, end) {
    const dx = end.x - start.x;
    const dy = end.y - start.y;
    const angle = Math.atan2(dy, dx);

    // 转换为 8 方向
    const directions = ['right', 'down-right', 'down', 'down-left', 'left', 'up-left', 'up', 'up-right'];
    const index = Math.round((angle + Math.PI) / (Math.PI / 4)) % 8;
    return directions[index];
  }

  isCircularGesture(path) {
    if (path.length < 10) return false;

    // 计算路径的弯曲度
    let totalAngle = 0;
    for (let i = 2; i < path.length; i++) {
      const angle1 = Math.atan2(path[i-1].y - path[i-2].y, path[i-1].x - path[i-2].x);
      const angle2 = Math.atan2(path[i].y - path[i-1].y, path[i].x - path[i-1].x);
      let angleDiff = angle2 - angle1;

      // 规范化角度差
      while (angleDiff > Math.PI) angleDiff -= 2 * Math.PI;
      while (angleDiff < -Math.PI) angleDiff += 2 * Math.PI;

      totalAngle += angleDiff;
    }

    // 如果总角度接近 2π，则认为是圆形手势
    return Math.abs(totalAngle) > Math.PI * 1.5;
  }

  classifyComplexGesture(path) {
    // 简单的手势分类器
    const features = this.extractGestureFeatures(path);

    // 这里可以使用机器学习算法或规则引擎
    // 为简化，我们使用基础的特征匹配

    if (features.hasLoop && features.straightness < 0.3) {
      return 'spiral';
    }

    if (features.sharpTurns > 2) {
      return 'zigzag';
    }

    if (features.peakCount === 1) {
      return 'mountain';
    }

    return 'unknown';
  }

  extractGestureFeatures(path) {
    // 提取手势特征
    const features = {
      length: path.length,
      totalDistance: this.calculatePathDistance(path),
      boundingBox: this.getBoundingBox(path),
      sharpTurns: 0,
      hasLoop: false,
      straightness: 0,
      peakCount: 0
    };

    // 计算急转弯
    for (let i = 2; i < path.length; i++) {
      const angle1 = Math.atan2(path[i-1].y - path[i-2].y, path[i-1].x - path[i-2].x);
      const angle2 = Math.atan2(path[i].y - path[i-1].y, path[i].x - path[i-1].x);
      const angleDiff = Math.abs(angle2 - angle1);

      if (angleDiff > Math.PI / 3) {
        features.sharpTurns++;
      }
    }

    // 计算直线度
    if (path.length > 1) {
      const straightDistance = this.calculateDistance(path[0], path[path.length - 1]);
      features.straightness = straightDistance / features.totalDistance;
    }

    return features;
  }

  getBoundingBox(path) {
    let minX = Infinity, minY = Infinity;
    let maxX = -Infinity, maxY = -Infinity;

    path.forEach(point => {
      minX = Math.min(minX, point.x);
      minY = Math.min(minY, point.y);
      maxX = Math.max(maxX, point.x);
      maxY = Math.max(maxY, point.y);
    });

    return {
      x: minX,
      y: minY,
      width: maxX - minX,
      height: maxY - minY
    };
  }

  // 手势事件回调
  onTap(point, duration) {
    console.log('Tap gesture at:', point, 'Duration:', duration);
  }

  onSwipe(direction, velocity, distance) {
    console.log('Swipe gesture:', direction, 'Velocity:', velocity, 'Distance:', distance);
  }

  onCircularGesture(path) {
    console.log('Circular gesture detected:', path);
  }

  onComplexGesture(type, path) {
    console.log('Complex gesture:', type, path);
  }

  getCanvasPoint(event) {
    const rect = this.canvas.getBoundingClientRect();
    return {
      x: event.clientX - rect.left,
      y: event.clientY - rect.top
    };
  }
}
```

## 6. 实战：交互式图形编辑器

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>交互式 Canvas 编辑器</title>
  <style>
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Arial;
      background: #f5f5f5;
      height: 100vh;
      display: flex;
    }

    .toolbar {
      width: 60px;
      background: white;
      box-shadow: 2px 0 5px rgba(0,0,0,0.1);
      display: flex;
      flex-direction: column;
      align-items: center;
      padding: 10px 0;
    }

    .tool {
      width: 40px;
      height: 40px;
      margin: 5px;
      border: 2px solid transparent;
      background: #f8f9fa;
      border-radius: 8px;
      cursor: pointer;
      display: flex;
      align-items: center;
      justify-content: center;
      transition: all 0.3s;
    }

    .tool:hover {
      background: #e9ecef;
    }

    .tool.active {
      border-color: #007acc;
      background: #e3f2fd;
    }

    .canvas-container {
      flex: 1;
      position: relative;
      overflow: hidden;
    }

    canvas {
      background: white;
      cursor: crosshair;
      display: block;
    }

    .status-bar {
      position: absolute;
      bottom: 10px;
      left: 10px;
      background: rgba(0,0,0,0.8);
      color: white;
      padding: 5px 10px;
      border-radius: 4px;
      font-size: 12px;
      font-family: monospace;
    }

    .properties-panel {
      position: absolute;
      top: 10px;
      right: 10px;
      background: white;
      padding: 15px;
      border-radius: 8px;
      box-shadow: 0 2px 10px rgba(0,0,0,0.1);
      width: 200px;
      display: none;
    }

    .properties-panel.show {
      display: block;
    }

    .property {
      margin-bottom: 10px;
    }

    .property label {
      display: block;
      font-size: 12px;
      margin-bottom: 3px;
      color: #666;
    }

    .property input {
      width: 100%;
      padding: 5px;
      border: 1px solid #ddd;
      border-radius: 4px;
    }
  </style>
</head>
<body>
  <div class="toolbar">
    <div class="tool active" data-tool="select" title="选择">
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor">
        <path d="M3 3l7.07 16.97 2.51-7.39 7.39-2.51L3 3z"/>
      </svg>
    </div>
    <div class="tool" data-tool="rectangle" title="矩形">
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor">
        <rect x="3" y="3" width="18" height="18" rx="2" ry="2"/>
      </svg>
    </div>
    <div class="tool" data-tool="circle" title="圆形">
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor">
        <circle cx="12" cy="12" r="10"/>
      </svg>
    </div>
    <div class="tool" data-tool="line" title="直线">
      <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor">
        <line x1="5" y1="12" x2="19" y2="12"/>
      </svg>
    </div>
  </div>

  <div class="canvas-container">
    <canvas id="editor-canvas"></canvas>
    <div class="status-bar" id="status-bar">
      工具: 选择 | 坐标: (0, 0) | 元素: 0
    </div>

    <div class="properties-panel" id="properties-panel">
      <h3>属性</h3>
      <div class="property">
        <label>X 坐标</label>
        <input type="number" id="prop-x" placeholder="0">
      </div>
      <div class="property">
        <label>Y 坐标</label>
        <input type="number" id="prop-y" placeholder="0">
      </div>
      <div class="property">
        <label>宽度</label>
        <input type="number" id="prop-width" placeholder="100">
      </div>
      <div class="property">
        <label>高度</label>
        <input type="number" id="prop-height" placeholder="100">
      </div>
      <div class="property">
        <label>颜色</label>
        <input type="color" id="prop-color" value="#3498db">
      </div>
    </div>
  </div>

  <script>
    class InteractiveEditor {
      constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = this.canvas.getContext('2d');

        this.currentTool = 'select';
        this.elements = [];
        this.selectedElements = new Set();

        // 交互状态
        this.isDrawing = false;
        this.isDragging = false;
        this.isBoxSelecting = false;

        this.startPoint = null;
        this.dragOffset = null;
        this.selectionBox = null;

        this.init();
      }

      init() {
        this.setupCanvas();
        this.setupEventListeners();
        this.setupUI();
        this.render();
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
        // 鼠标事件
        this.canvas.addEventListener('mousedown', (e) => this.onMouseDown(e));
        this.canvas.addEventListener('mousemove', (e) => this.onMouseMove(e));
        this.canvas.addEventListener('mouseup', (e) => this.onMouseUp(e));
        this.canvas.addEventListener('dblclick', (e) => this.onDoubleClick(e));

        // 键盘事件
        this.canvas.tabIndex = 0;
        this.canvas.addEventListener('keydown', (e) => this.onKeyDown(e));

        // 防止右键菜单
        this.canvas.addEventListener('contextmenu', (e) => e.preventDefault());
      }

      setupUI() {
        // 工具栏
        document.querySelectorAll('.tool').forEach(tool => {
          tool.addEventListener('click', (e) => {
            document.querySelectorAll('.tool').forEach(t => t.classList.remove('active'));
            tool.classList.add('active');
            this.currentTool = tool.dataset.tool;
            this.updateStatusBar();
          });
        });

        // 属性面板
        const properties = ['x', 'y', 'width', 'height', 'color'];
        properties.forEach(prop => {
          const input = document.getElementById(`prop-${prop}`);
          input.addEventListener('change', (e) => this.updateSelectedElements(prop, e.target.value));
        });
      }

      onMouseDown(event) {
        const point = this.getCanvasPoint(event);
        this.startPoint = point;

        switch (this.currentTool) {
          case 'select':
            this.startSelection(point, event.ctrlKey || event.metaKey);
            break;
          case 'rectangle':
          case 'circle':
          case 'line':
            this.startDrawing(point);
            break;
        }
      }

      onMouseMove(event) {
        const point = this.getCanvasPoint(event);

        if (this.isDrawing) {
          this.updateDrawing(point);
        } else if (this.isDragging) {
          this.updateDragging(point);
        } else if (this.isBoxSelecting) {
          this.updateBoxSelection(point);
        } else {
          // 悬停检测
          const element = this.getElementAtPoint(point.x, point.y);
          this.canvas.style.cursor = element ? 'grab' : 'crosshair';
        }

        this.updateStatusBar(point);
        this.render();
      }

      onMouseUp(event) {
        const point = this.getCanvasPoint(event);

        if (this.isDrawing) {
          this.finishDrawing(point);
        } else if (this.isDragging) {
          this.finishDragging();
        } else if (this.isBoxSelecting) {
          this.finishBoxSelection();
        }
      }

      onDoubleClick(event) {
        const point = this.getCanvasPoint(event);
        const element = this.getElementAtPoint(point.x, point.y);

        if (element && element.type === 'text') {
          // 进入文本编辑模式
          this.startTextEditing(element);
        }
      }

      onKeyDown(event) {
        switch (event.key) {
          case 'Delete':
          case 'Backspace':
            this.deleteSelectedElements();
            break;
          case 'a':
            if (event.ctrlKey || event.metaKey) {
              event.preventDefault();
              this.selectAll();
            }
            break;
          case 'd':
            if (event.ctrlKey || event.metaKey) {
              event.preventDefault();
              this.duplicateSelectedElements();
            }
            break;
        }
      }

      startSelection(point, multiSelect) {
        const element = this.getElementAtPoint(point.x, point.y);

        if (element) {
          if (!multiSelect) {
            if (!this.selectedElements.has(element)) {
              this.selectedElements.clear();
              this.selectedElements.add(element);
            }
          } else {
            if (this.selectedElements.has(element)) {
              this.selectedElements.delete(element);
            } else {
              this.selectedElements.add(element);
            }
          }

          if (this.selectedElements.has(element)) {
            this.startDragging(point, element);
          }
        } else {
          if (!multiSelect) {
            this.selectedElements.clear();
          }
          this.startBoxSelection(point);
        }

        this.updatePropertiesPanel();
      }

      startDragging(point, element) {
        this.isDragging = true;
        this.dragOffset = {
          x: point.x - element.x,
          y: point.y - element.y
        };
        this.canvas.style.cursor = 'grabbing';
      }

      updateDragging(point) {
        const deltaX = point.x - this.startPoint.x;
        const deltaY = point.y - this.startPoint.y;

        this.selectedElements.forEach(element => {
          element.x = element.startX + deltaX;
          element.y = element.startY + deltaY;
        });
      }

      finishDragging() {
        this.isDragging = false;
        this.canvas.style.cursor = 'grab';
        this.updatePropertiesPanel();
      }

      startBoxSelection(point) {
        this.isBoxSelecting = true;
        this.selectionBox = {
          x: point.x,
          y: point.y,
          width: 0,
          height: 0
        };
      }

      updateBoxSelection(point) {
        this.selectionBox.x = Math.min(this.startPoint.x, point.x);
        this.selectionBox.y = Math.min(this.startPoint.y, point.y);
        this.selectionBox.width = Math.abs(point.x - this.startPoint.x);
        this.selectionBox.height = Math.abs(point.y - this.startPoint.y);

        // 更新选中元素
        this.selectedElements.clear();
        this.elements.forEach(element => {
          if (this.rectIntersectsElement(this.selectionBox, element)) {
            this.selectedElements.add(element);
          }
        });
      }

      finishBoxSelection() {
        this.isBoxSelecting = false;
        this.selectionBox = null;
        this.updatePropertiesPanel();
      }

      startDrawing(point) {
        this.isDrawing = true;

        const element = {
          id: Date.now(),
          type: this.currentTool,
          x: point.x,
          y: point.y,
          width: 0,
          height: 0,
          color: document.getElementById('prop-color').value,
          startX: point.x,
          startY: point.y
        };

        if (this.currentTool === 'line') {
          element.endX = point.x;
          element.endY = point.y;
        }

        this.elements.push(element);
      }

      updateDrawing(point) {
        if (this.elements.length === 0) return;

        const element = this.elements[this.elements.length - 1];

        switch (element.type) {
          case 'rectangle':
            element.x = Math.min(element.startX, point.x);
            element.y = Math.min(element.startY, point.y);
            element.width = Math.abs(point.x - element.startX);
            element.height = Math.abs(point.y - element.startY);
            break;
          case 'circle':
            const radius = Math.sqrt(
              Math.pow(point.x - element.startX, 2) +
              Math.pow(point.y - element.startY, 2)
            );
            element.x = element.startX - radius;
            element.y = element.startY - radius;
            element.width = radius * 2;
            element.height = radius * 2;
            break;
          case 'line':
            element.endX = point.x;
            element.endY = point.y;
            break;
        }
      }

      finishDrawing(point) {
        this.isDrawing = false;

        if (this.elements.length > 0) {
          const element = this.elements[this.elements.length - 1];

          // 删除太小的元素
          if ((element.type !== 'line' && (element.width < 5 || element.height < 5)) ||
              (element.type === 'line' && Math.abs(element.endX - element.startX) < 5 && Math.abs(element.endY - element.startY) < 5)) {
            this.elements.pop();
          } else {
            delete element.startX;
            delete element.startY;
          }
        }
      }

      getCanvasPoint(event) {
        const rect = this.canvas.getBoundingClientRect();
        return {
          x: event.clientX - rect.left,
          y: event.clientY - rect.top
        };
      }

      getElementAtPoint(x, y) {
        for (let i = this.elements.length - 1; i >= 0; i--) {
          const element = this.elements[i];
          if (this.pointInElement(x, y, element)) {
            return element;
          }
        }
        return null;
      }

      pointInElement(x, y, element) {
        switch (element.type) {
          case 'rectangle':
            return x >= element.x && x <= element.x + element.width &&
                   y >= element.y && y <= element.y + element.height;
          case 'circle':
            const centerX = element.x + element.width / 2;
            const centerY = element.y + element.height / 2;
            const radius = element.width / 2;
            const dx = x - centerX;
            const dy = y - centerY;
            return dx * dx + dy * dy <= radius * radius;
          case 'line':
            return this.pointToLineDistance(x, y, element.x, element.y, element.endX, element.endY) <= 5;
          default:
            return false;
        }
      }

      pointToLineDistance(px, py, x1, y1, x2, y2) {
        const A = px - x1;
        const B = py - y1;
        const C = x2 - x1;
        const D = y2 - y1;

        const dot = A * C + B * D;
        const lenSq = C * C + D * D;

        if (lenSq === 0) return Math.sqrt(A * A + B * B);

        let param = dot / lenSq;
        param = Math.max(0, Math.min(1, param));

        const xx = x1 + param * C;
        const yy = y1 + param * D;
        const dx = px - xx;
        const dy = py - yy;

        return Math.sqrt(dx * dx + dy * dy);
      }

      rectIntersectsElement(rect, element) {
        return !(rect.x > element.x + element.width ||
                 rect.x + rect.width < element.x ||
                 rect.y > element.y + element.height ||
                 rect.y + rect.height < element.y);
      }

      render() {
        const ctx = this.ctx;
        const width = this.canvas.width / (window.devicePixelRatio || 1);
        const height = this.canvas.height / (window.devicePixelRatio || 1);

        // 清空画布
        ctx.clearRect(0, 0, width, height);

        // 绘制网格
        this.drawGrid();

        // 绘制元素
        this.elements.forEach(element => {
          this.drawElement(element);
        });

        // 绘制选中状态
        this.selectedElements.forEach(element => {
          this.drawSelection(element);
        });

        // 绘制选择框
        if (this.selectionBox) {
          this.drawSelectionBox(this.selectionBox);
        }
      }

      drawGrid() {
        const ctx = this.ctx;
        const width = this.canvas.width / (window.devicePixelRatio || 1);
        const height = this.canvas.height / (window.devicePixelRatio || 1);
        const gridSize = 20;

        ctx.strokeStyle = '#f0f0f0';
        ctx.lineWidth = 1;

        for (let x = 0; x <= width; x += gridSize) {
          ctx.beginPath();
          ctx.moveTo(x, 0);
          ctx.lineTo(x, height);
          ctx.stroke();
        }

        for (let y = 0; y <= height; y += gridSize) {
          ctx.beginPath();
          ctx.moveTo(0, y);
          ctx.lineTo(width, y);
          ctx.stroke();
        }
      }

      drawElement(element) {
        const ctx = this.ctx;

        ctx.save();
        ctx.fillStyle = element.color;
        ctx.strokeStyle = element.color;
        ctx.lineWidth = 2;

        switch (element.type) {
          case 'rectangle':
            ctx.fillRect(element.x, element.y, element.width, element.height);
            break;
          case 'circle':
            ctx.beginPath();
            ctx.arc(
              element.x + element.width / 2,
              element.y + element.height / 2,
              element.width / 2,
              0,
              Math.PI * 2
            );
            ctx.fill();
            break;
          case 'line':
            ctx.beginPath();
            ctx.moveTo(element.x, element.y);
            ctx.lineTo(element.endX, element.endY);
            ctx.stroke();
            break;
        }

        ctx.restore();
      }

      drawSelection(element) {
        const ctx = this.ctx;

        ctx.save();
        ctx.strokeStyle = '#007acc';
        ctx.lineWidth = 2;
        ctx.setLineDash([5, 5]);
        ctx.strokeRect(
          element.x - 2,
          element.y - 2,
          element.width + 4,
          element.height + 4
        );
        ctx.restore();
      }

      drawSelectionBox(box) {
        const ctx = this.ctx;

        ctx.save();
        ctx.strokeStyle = '#007acc';
        ctx.fillStyle = 'rgba(0, 122, 204, 0.1)';
        ctx.setLineDash([5, 5]);
        ctx.strokeRect(box.x, box.y, box.width, box.height);
        ctx.fillRect(box.x, box.y, box.width, box.height);
        ctx.restore();
      }

      updateStatusBar(point = null) {
        const statusBar = document.getElementById('status-bar');
        const toolName = this.currentTool === 'select' ? '选择' :
                        this.currentTool === 'rectangle' ? '矩形' :
                        this.currentTool === 'circle' ? '圆形' : '直线';

        const coords = point ? `(${Math.round(point.x)}, ${Math.round(point.y)})` : '(0, 0)';

        statusBar.textContent = `工具: ${toolName} | 坐标: ${coords} | 元素: ${this.elements.length} | 选中: ${this.selectedElements.size}`;
      }

      updatePropertiesPanel() {
        const panel = document.getElementById('properties-panel');

        if (this.selectedElements.size === 1) {
          const element = Array.from(this.selectedElements)[0];
          document.getElementById('prop-x').value = Math.round(element.x);
          document.getElementById('prop-y').value = Math.round(element.y);
          document.getElementById('prop-width').value = Math.round(element.width);
          document.getElementById('prop-height').value = Math.round(element.height);
          document.getElementById('prop-color').value = element.color;
          panel.classList.add('show');
        } else {
          panel.classList.remove('show');
        }
      }

      updateSelectedElements(property, value) {
        this.selectedElements.forEach(element => {
          switch (property) {
            case 'x':
            case 'y':
            case 'width':
            case 'height':
              element[property] = parseFloat(value) || 0;
              break;
            case 'color':
              element[property] = value;
              break;
          }
        });
        this.render();
      }

      deleteSelectedElements() {
        this.selectedElements.forEach(element => {
          const index = this.elements.indexOf(element);
          if (index > -1) {
            this.elements.splice(index, 1);
          }
        });
        this.selectedElements.clear();
        this.updatePropertiesPanel();
        this.render();
      }

      selectAll() {
        this.selectedElements.clear();
        this.elements.forEach(element => this.selectedElements.add(element));
        this.updatePropertiesPanel();
        this.render();
      }

      duplicateSelectedElements() {
        const newElements = [];
        this.selectedElements.forEach(element => {
          const newElement = { ...element };
          newElement.id = Date.now() + Math.random();
          newElement.x += 20;
          newElement.y += 20;
          newElements.push(newElement);
        });

        this.elements.push(...newElements);

        this.selectedElements.clear();
        newElements.forEach(element => this.selectedElements.add(element));

        this.updatePropertiesPanel();
        this.render();
      }
    }

    // 初始化编辑器
    const editor = new InteractiveEditor('editor-canvas');
  </script>
</body>
</html>
```

## 7. Excalidraw 中的应用

### 7.1 Excalidraw 的事件系统

```typescript
// packages/excalidraw/components/App.tsx
export class App extends Component<AppProps, AppState> {
  private handleCanvasPointerDown = (event: React.PointerEvent<HTMLCanvasElement>) => {
    const pointerCoords = viewportCoordsToSceneCoords(
      { clientX: event.clientX, clientY: event.clientY },
      this.state
    );

    const hitElement = this.getElementAtPosition(
      pointerCoords.x,
      pointerCoords.y
    );

    if (hitElement && this.state.activeTool.type === "selection") {
      this.setState({ selectedElementIds: { [hitElement.id]: true } });
    }
  };

  private handleCanvasPointerMove = (event: React.PointerEvent<HTMLCanvasElement>) => {
    const pointerCoords = viewportCoordsToSceneCoords(
      { clientX: event.clientX, clientY: event.clientY },
      this.state
    );

    if (this.state.draggingElement) {
      this.updateDraggingElement(pointerCoords);
    }

    this.setState({ cursorX: pointerCoords.x, cursorY: pointerCoords.y });
  };

  private getElementAtPosition(x: number, y: number) {
    const elements = this.scene.getElements();

    for (let i = elements.length - 1; i >= 0; i--) {
      const element = elements[i];
      if (hitTest(element, x, y)) {
        return element;
      }
    }

    return null;
  }
}

// 坐标转换工具（源码: packages/common/src/utils.ts:436-456）
// Excalidraw实际使用的坐标转换函数
export const viewportCoordsToSceneCoords = (
  { clientX, clientY }: { clientX: number; clientY: number },
  {
    zoom,
    offsetLeft,
    offsetTop,
    scrollX,
    scrollY,
  }: {
    zoom: Zoom;
    offsetLeft: number;
    offsetTop: number;
    scrollX: number;
    scrollY: number;
  },
) => {
  // 关键实现细节:
  // 1. 先减去canvas的offsetLeft/offsetTop (不是getBoundingClientRect)
  // 2. 除以zoom.value进行缩放转换
  // 3. 最后减去scrollX/scrollY (而非先减)
  const x = (clientX - offsetLeft) / zoom.value - scrollX;
  const y = (clientY - offsetTop) / zoom.value - scrollY;

  return { x, y };
};
```

## 8. 练习题

### 8.1 基础练习

1. **实现精确的碰撞检测**
   - 椭圆与点的碰撞
   - 旋转矩形与点的碰撞
   - 多边形与点的碰撞

2. **优化拖拽性能**
   - 使用四叉树减少碰撞检测计算
   - 实现平滑的拖拽动画
   - 添加约束系统（网格吸附、边界限制）

3. **手势识别**
   - 实现简单的手势识别系统
   - 支持圆形、直线、波浪等基础手势
   - 添加手势快捷操作

### 8.2 进阶练习

1. **复杂选择系统**
   ```javascript
   class AdvancedSelection {
     constructor() {
       this.selectionModes = ['single', 'multi', 'lasso'];
       this.currentMode = 'single';
     }

     // 实现套索选择
     lassoSelection(path) {
       // TODO: 实现不规则路径选择
     }

     // 实现相似元素选择
     selectSimilar(element) {
       // TODO: 选择类型、颜色、大小相似的元素
     }
   }
   ```

## 9. 思考题

1. **如何处理大量元素的交互性能？**
   - 空间索引（四叉树、R树）
   - 层级细节（LOD）
   - 视口剔除

2. **移动设备上的交互有什么特殊考虑？**
   - 触摸精度问题
   - 多点触控手势
   - 性能优化

3. **如何实现撤销/重做功能？**
   - 命令模式
   - 状态快照
   - 增量记录

4. **复杂图形的碰撞检测如何优化？**
   - 边界框预筛选
   - 分层检测
   - 空间分割

## 10. 总结

### 核心要点

1. **事件处理基础**
   - Canvas 事件模型理解
   - 坐标转换的重要性
   - 触摸事件的特殊处理

2. **碰撞检测算法**
   - 基础几何碰撞检测
   - 复杂图形的处理方法
   - 性能优化策略

3. **交互系统设计**
   - 拖拽操作的完整实现
   - 多选和框选功能
   - 手势识别技术

4. **性能优化**
   - 空间索引的应用
   - 事件节流和防抖
   - 渲染优化策略

### 下一步

掌握了交互系统后，下一章将学习：
- Canvas 性能优化技术
- 脏矩形算法
- 分层渲染策略
- 内存管理技巧

## 11. 参考资源

- [MDN Touch Events](https://developer.mozilla.org/en-US/docs/Web/API/Touch_events)
- [Canvas Hit Detection](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API/Tutorial/Hit_regions_and_accessibility)
- [Collision Detection Algorithms](https://www.jeffreythompson.org/collision-detection/)
- [Spatial Indexing](https://en.wikipedia.org/wiki/Spatial_database)

---

**上一章**：[Canvas 变换与合成](./03-canvas-transform.md)
**下一章**：[Canvas 性能优化 →](./05-canvas-optimization.md)