# 《从零构建 Excalidraw：现代画板应用的完整实现》
## 大学级别的系统性技术课程设计

---

## 📚 课程概述

### 课程目标
通过 15 周的系统学习，让零基础学员完全掌握如何从头实现一个类似 Excalidraw 的现代化协作画板应用，理解其背后的核心技术原理和架构设计思想。

### 学习成果
- 深入理解 Canvas 2D 图形编程
- 掌握现代前端架构设计模式
- 学会高性能图形应用的优化技巧
- 理解实时协作系统的实现原理
- 具备独立开发复杂前端应用的能力

### 前置知识
- 基础的 HTML/CSS/JavaScript 知识
- 对 React 有初步了解
- 基本的数学概念（坐标系、向量等）

---

## 🎯 课程结构设计

### 第一阶段：基础技术栈（第 1-5 周）
**目标：掌握构建画板应用的底层技术**

### 第二阶段：核心系统实现（第 6-10 周）  
**目标：实现完整的画板核心功能**

### 第三阶段：高级特性与优化（第 11-15 周）
**目标：添加协作、性能优化等高级功能**

---

## 📖 详细课程大纲

### 第一周：Web 图形学基础 - Canvas API 深入探索

**学习目标**
- 理解浏览器图形渲染原理
- 掌握 Canvas 2D Context 的完整 API
- 理解即时模式 vs 保留模式的差异

**核心内容**
```javascript
// 1.1 Canvas 基础概念
const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');

// 1.2 坐标系统与变换
ctx.translate(100, 100);
ctx.rotate(Math.PI / 4);
ctx.scale(2, 2);

// 1.3 路径绘制系统
ctx.beginPath();
ctx.moveTo(0, 0);
ctx.lineTo(100, 100);
ctx.stroke();

// 1.4 样式系统详解
ctx.strokeStyle = '#ff0000';
ctx.fillStyle = 'rgba(0, 255, 0, 0.5)';
ctx.lineWidth = 5;
ctx.lineCap = 'round';
```

**实践项目**：实现一个基础的绘图板
- 支持鼠标绘制直线
- 支持颜色和线宽调整
- 理解 Canvas 的即时模式特性

**技术深度**
- Canvas 与 SVG 的性能对比分析
- Canvas 在不同浏览器中的实现差异
- 高 DPI 屏幕的适配处理

### 第二周：数学基础与几何算法

**学习目标**
- 掌握 2D 图形学的数学基础
- 理解几何变换的数学原理
- 学会常用的几何算法

**核心内容**
```javascript
// 2.1 向量数学
class Vector2 {
    constructor(x, y) {
        this.x = x;
        this.y = y;
    }
    
    dot(other) {
        return this.x * other.x + this.y * other.y;
    }
    
    cross(other) {
        return this.x * other.y - this.y * other.x;
    }
    
    normalize() {
        const length = Math.sqrt(this.x * this.x + this.y * this.y);
        return new Vector2(this.x / length, this.y / length);
    }
}

// 2.2 矩阵变换
class Transform2D {
    constructor() {
        this.matrix = [1, 0, 0, 1, 0, 0]; // [a, b, c, d, tx, ty]
    }
    
    translate(tx, ty) {
        this.matrix[4] += this.matrix[0] * tx + this.matrix[2] * ty;
        this.matrix[5] += this.matrix[1] * tx + this.matrix[3] * ty;
    }
    
    rotate(angle) {
        const cos = Math.cos(angle);
        const sin = Math.sin(angle);
        // 矩阵乘法实现旋转
    }
    
    transformPoint(x, y) {
        return {
            x: this.matrix[0] * x + this.matrix[2] * y + this.matrix[4],
            y: this.matrix[1] * x + this.matrix[3] * y + this.matrix[5]
        };
    }
}

// 2.3 碰撞检测算法
function pointInRect(px, py, rect) {
    return px >= rect.x && px <= rect.x + rect.width &&
           py >= rect.y && py <= rect.y + rect.height;
}

function pointToLineDistance(px, py, x1, y1, x2, y2) {
    // 点到线段最短距离的计算
    const A = px - x1;
    const B = py - y1;
    const C = x2 - x1;
    const D = y2 - y1;
    
    const dot = A * C + B * D;
    const lenSq = C * C + D * D;
    let param = -1;
    if (lenSq !== 0) param = dot / lenSq;
    
    let xx, yy;
    if (param < 0) {
        xx = x1; yy = y1;
    } else if (param > 1) {
        xx = x2; yy = y2;
    } else {
        xx = x1 + param * C;
        yy = y1 + param * D;
    }
    
    const dx = px - xx;
    const dy = py - yy;
    return Math.sqrt(dx * dx + dy * dy);
}
```

**实践项目**：几何计算器
- 实现各种几何形状的绘制
- 添加旋转、缩放功能
- 实现精确的点击检测

### 第三周：事件系统与交互设计

**学习目标**
- 掌握现代 Web 事件系统
- 理解指针事件的统一处理
- 学会设计流畅的交互体验

**核心内容**
```javascript
// 3.1 统一的指针事件处理
class PointerEventManager {
    constructor(canvas) {
        this.canvas = canvas;
        this.pointers = new Map(); // 支持多点触控
        this.setupEventListeners();
    }
    
    setupEventListeners() {
        // 统一处理鼠标和触摸事件
        this.canvas.addEventListener('pointerdown', this.handlePointerDown.bind(this));
        this.canvas.addEventListener('pointermove', this.handlePointerMove.bind(this));
        this.canvas.addEventListener('pointerup', this.handlePointerUp.bind(this));
        
        // 阻止默认的触摸行为
        this.canvas.addEventListener('touchstart', e => e.preventDefault());
        this.canvas.addEventListener('touchmove', e => e.preventDefault());
    }
    
    handlePointerDown(event) {
        this.pointers.set(event.pointerId, {
            x: event.offsetX,
            y: event.offsetY,
            pressure: event.pressure,
            pointerType: event.pointerType
        });
        
        this.onPointerDown?.(event);
    }
}

// 3.2 手势识别系统
class GestureRecognizer {
    constructor() {
        this.gestures = [];
        this.currentGesture = null;
    }
    
    // 识别双击
    recognizeDoubleClick(events) {
        const timeDiff = events[1].timeStamp - events[0].timeStamp;
        const distance = Math.hypot(
            events[1].x - events[0].x,
            events[1].y - events[0].y
        );
        return timeDiff < 500 && distance < 10;
    }
    
    // 识别缩放手势
    recognizePinch(pointers) {
        if (pointers.size !== 2) return null;
        
        const [p1, p2] = Array.from(pointers.values());
        const distance = Math.hypot(p2.x - p1.x, p2.y - p1.y);
        const center = {
            x: (p1.x + p2.x) / 2,
            y: (p1.y + p2.y) / 2
        };
        
        return { distance, center };
    }
}

// 3.3 平滑的动画系统
class AnimationSystem {
    constructor() {
        this.animations = [];
        this.running = false;
    }
    
    animate(from, to, duration, easing = 'easeInOut') {
        return new Promise(resolve => {
            const animation = {
                from, to, duration,
                startTime: performance.now(),
                easing: this.getEasingFunction(easing),
                resolve
            };
            
            this.animations.push(animation);
            if (!this.running) this.start();
        });
    }
    
    getEasingFunction(name) {
        const easings = {
            linear: t => t,
            easeInOut: t => t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t,
            elastic: t => Math.sin(13 * Math.PI / 2 * t) * Math.pow(2, 10 * (t - 1))
        };
        return easings[name] || easings.easeInOut;
    }
}
```

**实践项目**：交互原型
- 实现多种手势识别
- 添加平滑的动画效果
- 支持键盘快捷键

### 第四周：状态管理架构设计

**学习目标**
- 理解复杂应用的状态管理需求
- 学会 Jotai 原子化状态管理
- 掌握状态持久化和同步

**核心内容**
```javascript
// 4.1 Jotai 原子化状态管理
import { atom, useAtom, useAtomValue } from 'jotai';

// 定义原子状态
export const elementsAtom = atom([]);
export const selectedElementIdsAtom = atom(new Set());
export const currentToolAtom = atom('selection');
export const canvasStateAtom = atom({
    zoom: 1,
    scrollX: 0,
    scrollY: 0
});

// 派生状态
export const selectedElementsAtom = atom((get) => {
    const elements = get(elementsAtom);
    const selectedIds = get(selectedElementIdsAtom);
    return elements.filter(el => selectedIds.has(el.id));
});

// 4.2 状态持久化
class StateManager {
    constructor() {
        this.storage = new Map();
        this.subscribers = [];
    }
    
    // 自动保存到 localStorage
    persist(key, atom) {
        const value = JSON.stringify(atom.get());
        localStorage.setItem(key, value);
    }
    
    // 从 localStorage 恢复
    restore(key) {
        const value = localStorage.getItem(key);
        return value ? JSON.parse(value) : null;
    }
    
    // 实现撤销/重做
    createHistoryAtom(atom) {
        return atom({
            history: [],
            currentIndex: -1,
            
            record(value) {
                this.history = this.history.slice(0, this.currentIndex + 1);
                this.history.push(JSON.parse(JSON.stringify(value)));
                this.currentIndex++;
                
                if (this.history.length > 100) {
                    this.history.shift();
                    this.currentIndex--;
                }
            },
            
            undo() {
                if (this.currentIndex > 0) {
                    this.currentIndex--;
                    return this.history[this.currentIndex];
                }
                return null;
            },
            
            redo() {
                if (this.currentIndex < this.history.length - 1) {
                    this.currentIndex++;
                    return this.history[this.currentIndex];
                }
                return null;
            }
        });
    }
}

// 4.3 状态同步与冲突解决
class StateSync {
    constructor() {
        this.localVersion = 0;
        this.remoteVersion = 0;
        this.conflictResolver = new ConflictResolver();
    }
    
    // 应用远程更新
    applyRemoteUpdate(update) {
        if (update.version > this.remoteVersion) {
            // 检查是否有冲突
            if (this.hasConflict(update)) {
                return this.conflictResolver.resolve(update, this.getLocalState());
            }
            
            // 无冲突，直接应用
            this.applyUpdate(update);
            this.remoteVersion = update.version;
        }
    }
    
    // 广播本地更新
    broadcastUpdate(update) {
        update.version = ++this.localVersion;
        update.clientId = this.clientId;
        this.socket.emit('update', update);
    }
}
```

**实践项目**：状态管理系统
- 实现完整的状态管理
- 添加撤销/重做功能
- 实现状态的持久化

### 第五周：React 集成与组件设计

**学习目标**
- 学会将 Canvas 集成到 React 生态
- 掌握高性能 React 组件设计
- 理解 React 和 Canvas 的协作模式

**核心内容**
```typescript
// 5.1 Canvas React Hook
import { useRef, useEffect, useCallback } from 'react';

function useCanvas() {
    const canvasRef = useRef<HTMLCanvasElement>(null);
    const ctxRef = useRef<CanvasRenderingContext2D | null>(null);
    
    useEffect(() => {
        const canvas = canvasRef.current;
        if (!canvas) return;
        
        const ctx = canvas.getContext('2d');
        if (!ctx) return;
        
        // 高DPI适配
        const dpr = window.devicePixelRatio || 1;
        const rect = canvas.getBoundingClientRect();
        
        canvas.width = rect.width * dpr;
        canvas.height = rect.height * dpr;
        canvas.style.width = rect.width + 'px';
        canvas.style.height = rect.height + 'px';
        
        ctx.scale(dpr, dpr);
        ctxRef.current = ctx;
    }, []);
    
    return { canvasRef, ctx: ctxRef.current };
}

// 5.2 高性能 Canvas 组件
const CanvasRenderer = React.memo(({ elements, viewState }) => {
    const { canvasRef, ctx } = useCanvas();
    const lastRenderTime = useRef(0);
    
    // 使用 RAF 节流渲染
    const render = useCallback(() => {
        if (!ctx) return;
        
        const now = performance.now();
        if (now - lastRenderTime.current < 16) return; // 60FPS
        
        // 清空画布
        ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
        
        // 应用视口变换
        ctx.save();
        ctx.translate(viewState.scrollX, viewState.scrollY);
        ctx.scale(viewState.zoom, viewState.zoom);
        
        // 渲染所有元素
        elements.forEach(element => {
            renderElement(element, ctx);
        });
        
        ctx.restore();
        lastRenderTime.current = now;
    }, [ctx, elements, viewState]);
    
    useEffect(() => {
        render();
    }, [render]);
    
    return <canvas ref={canvasRef} className="excalidraw-canvas" />;
});

// 5.3 组件架构设计
const ExcalidrawApp: React.FC = () => {
    return (
        <Provider store={jotaiStore}>
            <div className="excalidraw-wrapper">
                <Toolbar />
                <div className="excalidraw-main">
                    <Sidebar />
                    <div className="excalidraw-canvas-container">
                        <CanvasRenderer />
                        <SelectionUI />
                        <CursorLayer />
                    </div>
                </div>
                <StatusBar />
            </div>
        </Provider>
    );
};

// 5.4 性能优化策略
const OptimizedComponent = React.memo(({ data }) => {
    // 使用 useMemo 缓存复杂计算
    const processedData = useMemo(() => {
        return expensiveProcessing(data);
    }, [data]);
    
    // 使用 useCallback 缓存事件处理器
    const handleClick = useCallback((event) => {
        // 处理逻辑
    }, []);
    
    return <div onClick={handleClick}>{processedData}</div>;
});
```

**实践项目**：React Canvas 应用
- 构建完整的 React 应用架构
- 实现工具栏和侧边栏
- 优化渲染性能

---

### 第六周：元素系统架构设计

**学习目标**
- 设计灵活可扩展的元素系统
- 理解面向对象设计在图形系统中的应用
- 掌握元素序列化和反序列化

**核心内容**
```typescript
// 6.1 元素基类设计
interface BaseElement {
    id: string;
    type: ElementType;
    x: number;
    y: number;
    width: number;
    height: number;
    angle: number;
    
    // 样式属性
    strokeColor: string;
    backgroundColor: string;
    strokeWidth: number;
    roughness: number;
    opacity: number;
    
    // 版本控制
    version: number;
    versionNonce: number;
    
    // 关系
    boundElements?: BoundElement[];
    groupIds: string[];
    
    // 状态
    isDeleted: boolean;
    locked: boolean;
    
    // 扩展数据
    customData?: Record<string, any>;
}

// 6.2 具体元素类型
interface RectangleElement extends BaseElement {
    type: 'rectangle';
}

interface EllipseElement extends BaseElement {
    type: 'ellipse';
}

interface LineElement extends BaseElement {
    type: 'line' | 'arrow' | 'freedraw';
    points: readonly Point[];
    lastCommittedPoint?: Point;
    startBinding?: PointBinding;
    endBinding?: PointBinding;
    startArrowhead?: Arrowhead;
    endArrowhead?: Arrowhead;
}

interface TextElement extends BaseElement {
    type: 'text';
    text: string;
    fontSize: number;
    fontFamily: FontFamily;
    textAlign: TextAlign;
    verticalAlign: VerticalAlign;
    containerId?: string;
    originalText: string;
    lineHeight: number;
}

// 6.3 元素工厂模式
class ElementFactory {
    private static creators = new Map<ElementType, ElementCreator>();
    
    static register(type: ElementType, creator: ElementCreator) {
        this.creators.set(type, creator);
    }
    
    static create(type: ElementType, props: Partial<BaseElement>): BaseElement {
        const creator = this.creators.get(type);
        if (!creator) {
            throw new Error(`Unknown element type: ${type}`);
        }
        
        return creator({
            id: nanoid(),
            version: 1,
            versionNonce: Math.random(),
            isDeleted: false,
            locked: false,
            groupIds: [],
            ...props
        });
    }
}

// 6.4 元素操作系统
class ElementOperations {
    // 变换操作
    static transform(element: BaseElement, transform: Transform2D): BaseElement {
        const bounds = this.getBounds(element);
        const newBounds = transform.transformRect(bounds);
        
        return {
            ...element,
            ...newBounds,
            version: element.version + 1,
            versionNonce: Math.random()
        };
    }
    
    // 获取边界框
    static getBounds(element: BaseElement): Bounds {
        switch (element.type) {
            case 'rectangle':
            case 'ellipse':
            case 'text':
                return {
                    x: element.x,
                    y: element.y,
                    width: element.width,
                    height: element.height
                };
                
            case 'line':
            case 'arrow':
            case 'freedraw':
                return this.getLinearElementBounds(element as LineElement);
                
            default:
                throw new Error(`Unknown element type: ${element.type}`);
        }
    }
    
    // 碰撞检测
    static hitTest(element: BaseElement, point: Point): boolean {
        const bounds = this.getBounds(element);
        
        if (element.angle !== 0) {
            // 旋转后的碰撞检测
            const center = {
                x: bounds.x + bounds.width / 2,
                y: bounds.y + bounds.height / 2
            };
            
            const rotatedPoint = rotatePoint(point, center, -element.angle);
            return pointInRect(rotatedPoint, bounds);
        }
        
        return pointInRect(point, bounds);
    }
    
    // 序列化
    static serialize(element: BaseElement): SerializedElement {
        return {
            ...element,
            // 移除运行时属性
            isSelected: undefined,
            isDragging: undefined
        };
    }
    
    // 反序列化
    static deserialize(data: SerializedElement): BaseElement {
        // 验证数据完整性
        if (!data.id || !data.type) {
            throw new Error('Invalid element data');
        }
        
        // 迁移旧版本数据
        return this.migrate(data);
    }
}
```

**实践项目**：元素系统实现
- 实现完整的元素类型系统
- 添加元素序列化功能
- 支持元素的各种变换操作

### 第七周：渲染引擎与性能优化

**学习目标**
- 构建高性能的渲染引擎
- 理解分层渲染和脏矩形优化
- 掌握大规模图形的渲染技巧

**核心内容**
```javascript
// 7.1 分层渲染系统
class LayeredRenderer {
    constructor(container) {
        this.layers = {
            background: this.createCanvas(container, 0),
            static: this.createCanvas(container, 1),
            dynamic: this.createCanvas(container, 2),
            ui: this.createCanvas(container, 3)
        };
        
        this.dirtyLayers = new Set();
        this.renderScheduled = false;
    }
    
    createCanvas(container, zIndex) {
        const canvas = document.createElement('canvas');
        canvas.style.position = 'absolute';
        canvas.style.zIndex = zIndex;
        container.appendChild(canvas);
        return canvas;
    }
    
    markLayerDirty(layerName) {
        this.dirtyLayers.add(layerName);
        this.scheduleRender();
    }
    
    scheduleRender() {
        if (this.renderScheduled) return;
        this.renderScheduled = true;
        
        requestAnimationFrame(() => {
            this.renderDirtyLayers();
            this.renderScheduled = false;
        });
    }
    
    renderDirtyLayers() {
        this.dirtyLayers.forEach(layerName => {
            this.renderLayer(layerName);
        });
        this.dirtyLayers.clear();
    }
    
    renderLayer(layerName) {
        const canvas = this.layers[layerName];
        const ctx = canvas.getContext('2d');
        
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        
        switch (layerName) {
            case 'background':
                this.renderBackground(ctx);
                break;
            case 'static':
                this.renderStaticElements(ctx);
                break;
            case 'dynamic':
                this.renderDynamicElements(ctx);
                break;
            case 'ui':
                this.renderUI(ctx);
                break;
        }
    }
}

// 7.2 视口剔除
class ViewportCuller {
    constructor() {
        this.viewport = { x: 0, y: 0, width: 0, height: 0 };
    }
    
    updateViewport(scrollX, scrollY, width, height, zoom) {
        this.viewport = {
            x: scrollX,
            y: scrollY,
            width: width / zoom,
            height: height / zoom
        };
    }
    
    cullElements(elements) {
        return elements.filter(element => {
            const bounds = ElementOperations.getBounds(element);
            return this.intersects(bounds, this.viewport);
        });
    }
    
    intersects(rect1, rect2) {
        return !(rect1.x > rect2.x + rect2.width ||
                rect1.x + rect1.width < rect2.x ||
                rect1.y > rect2.y + rect2.height ||
                rect1.y + rect1.height < rect2.y);
    }
}

// 7.3 批量渲染优化
class BatchRenderer {
    constructor() {
        this.batches = new Map();
    }
    
    addElement(element) {
        const batchKey = this.getBatchKey(element);
        if (!this.batches.has(batchKey)) {
            this.batches.set(batchKey, []);
        }
        this.batches.get(batchKey).push(element);
    }
    
    getBatchKey(element) {
        // 相同样式的元素可以批量渲染
        return `${element.strokeColor}_${element.strokeWidth}_${element.fillStyle}`;
    }
    
    render(ctx) {
        this.batches.forEach((elements, batchKey) => {
            this.renderBatch(ctx, elements, batchKey);
        });
        this.batches.clear();
    }
    
    renderBatch(ctx, elements, batchKey) {
        const [strokeColor, strokeWidth, fillStyle] = batchKey.split('_');
        
        // 设置批量样式
        ctx.strokeStyle = strokeColor;
        ctx.lineWidth = parseInt(strokeWidth);
        ctx.fillStyle = fillStyle;
        
        // 批量绘制
        ctx.beginPath();
        elements.forEach(element => {
            this.addElementToPath(ctx, element);
        });
        ctx.stroke();
        
        if (fillStyle !== 'transparent') {
            ctx.fill();
        }
    }
}

// 7.4 LOD (Level of Detail) 系统
class LODRenderer {
    constructor() {
        this.lodLevels = [
            { minZoom: 0, maxZoom: 0.25, detail: 'lowest' },
            { minZoom: 0.25, maxZoom: 0.5, detail: 'low' },
            { minZoom: 0.5, maxZoom: 1, detail: 'medium' },
            { minZoom: 1, maxZoom: Infinity, detail: 'high' }
        ];
    }
    
    getLODLevel(zoom) {
        return this.lodLevels.find(level => 
            zoom >= level.minZoom && zoom < level.maxZoom
        );
    }
    
    renderElement(ctx, element, zoom) {
        const lodLevel = this.getLODLevel(zoom);
        
        switch (lodLevel.detail) {
            case 'lowest':
                // 只绘制简单的矩形轮廓
                ctx.strokeRect(element.x, element.y, element.width, element.height);
                break;
            case 'low':
                // 绘制基本形状，不绘制文字
                this.renderBasicShape(ctx, element);
                break;
            case 'medium':
                // 绘制完整形状，简化文字
                this.renderMediumDetail(ctx, element);
                break;
            case 'high':
                // 完整渲染
                this.renderFullDetail(ctx, element);
                break;
        }
    }
}
```

**实践项目**：高性能渲染器
- 实现分层渲染系统
- 添加视口剔除优化
- 实现 LOD 系统

### 第八周：工具系统与状态机

**学习目标**
- 设计可扩展的工具系统架构
- 理解有限状态机在交互设计中的应用
- 掌握复杂工具的实现技巧

**核心内容**
```javascript
// 8.1 工具基类设计
class BaseTool {
    constructor(name, options = {}) {
        this.name = name;
        this.options = options;
        this.state = 'idle';
        this.cursor = 'default';
    }
    
    // 生命周期方法
    onActivate() {}
    onDeactivate() {}
    
    // 事件处理方法
    onPointerDown(event) {}
    onPointerMove(event) {}
    onPointerUp(event) {}
    onKeyDown(event) {}
    onKeyUp(event) {}
    
    // 渲染方法
    renderOverlay(ctx) {}
    
    // 状态转换
    setState(newState) {
        const oldState = this.state;
        this.state = newState;
        this.onStateChange?.(oldState, newState);
    }
}

// 8.2 选择工具实现
class SelectionTool extends BaseTool {
    constructor() {
        super('selection');
        this.selectedElements = [];
        this.dragStart = null;
        this.resizeHandle = null;
        this.selectionBox = null;
    }
    
    onPointerDown(event) {
        const hitElement = this.hitTest(event.x, event.y);
        
        if (hitElement) {
            this.handleElementSelect(hitElement, event);
        } else {
            this.handleSelectionBoxStart(event);
        }
    }
    
    handleElementSelect(element, event) {
        if (!event.shiftKey) {
            this.clearSelection();
        }
        
        this.addToSelection(element);
        
        // 检查是否点击了调整大小的手柄
        this.resizeHandle = this.getResizeHandle(event.x, event.y);
        
        if (this.resizeHandle) {
            this.setState('resizing');
        } else {
            this.setState('dragging');
            this.dragStart = { x: event.x, y: event.y };
        }
    }
    
    handleSelectionBoxStart(event) {
        this.setState('selecting');
        this.selectionBox = {
            startX: event.x,
            startY: event.y,
            endX: event.x,
            endY: event.y
        };
    }
    
    onPointerMove(event) {
        switch (this.state) {
            case 'dragging':
                this.handleDrag(event);
                break;
            case 'resizing':
                this.handleResize(event);
                break;
            case 'selecting':
                this.handleSelectionBox(event);
                break;
            default:
                this.updateCursor(event);
        }
    }
    
    onPointerUp(event) {
        switch (this.state) {
            case 'selecting':
                this.finalizeSelection();
                break;
            case 'dragging':
                this.finalizeDrag();
                break;
            case 'resizing':
                this.finalizeResize();
                break;
        }
        
        this.setState('idle');
    }
}

// 8.3 绘制工具实现
class DrawingTool extends BaseTool {
    constructor(elementType) {
        super(`draw-${elementType}`);
        this.elementType = elementType;
        this.currentElement = null;
        this.previewPath = [];
    }
    
    onPointerDown(event) {
        this.setState('drawing');
        this.currentElement = ElementFactory.create(this.elementType, {
            x: event.x,
            y: event.y,
            width: 0,
            height: 0,
            strokeColor: AppState.currentStrokeColor,
            backgroundColor: AppState.currentBackgroundColor
        });
        
        if (this.elementType === 'freedraw') {
            this.currentElement.points = [{ x: event.x, y: event.y }];
        }
        
        this.addElementToScene(this.currentElement);
    }
    
    onPointerMove(event) {
        if (this.state !== 'drawing') return;
        
        switch (this.elementType) {
            case 'rectangle':
            case 'ellipse':
                this.updateRectangularElement(event);
                break;
            case 'line':
            case 'arrow':
                this.updateLinearElement(event);
                break;
            case 'freedraw':
                this.updateFreeDrawElement(event);
                break;
        }
    }
    
    updateFreeDrawElement(event) {
        // 使用 perfect-freehand 库优化自由绘制
        this.currentElement.points.push({
            x: event.x,
            y: event.y,
            pressure: event.pressure || 0.5
        });
        
        // 平滑处理
        if (this.currentElement.points.length > 2) {
            const points = this.currentElement.points;
            const lastThree = points.slice(-3);
            
            // 使用三点平滑
            const smoothed = this.smoothPoints(lastThree);
            this.currentElement.points.splice(-3, 3, ...smoothed);
        }
    }
    
    smoothPoints(points) {
        if (points.length < 3) return points;
        
        const [p0, p1, p2] = points;
        return [
            p0,
            {
                x: (p0.x + p1.x) / 2,
                y: (p0.y + p1.y) / 2,
                pressure: (p0.pressure + p1.pressure) / 2
            },
            {
                x: (p1.x + p2.x) / 2,
                y: (p1.y + p2.y) / 2,
                pressure: (p1.pressure + p2.pressure) / 2
            },
            p2
        ];
    }
}

// 8.4 工具管理器
class ToolManager {
    constructor() {
        this.tools = new Map();
        this.currentTool = null;
        this.defaultTool = 'selection';
    }
    
    registerTool(tool) {
        this.tools.set(tool.name, tool);
    }
    
    setTool(toolName) {
        if (this.currentTool) {
            this.currentTool.onDeactivate();
        }
        
        this.currentTool = this.tools.get(toolName) || this.tools.get(this.defaultTool);
        this.currentTool.onActivate();
        
        // 更新光标
        document.body.style.cursor = this.currentTool.cursor;
    }
    
    handlePointerEvent(eventType, event) {
        if (!this.currentTool) return;
        
        const method = `on${eventType.charAt(0).toUpperCase()}${eventType.slice(1)}`;
        this.currentTool[method]?.(event);
    }
}
```

**实践项目**：完整工具系统
- 实现多种绘图工具
- 添加工具间的平滑切换
- 支持工具的配置和扩展

### 第九周：手绘风格与 RoughJS 集成

**学习目标**
- 理解手绘风格的设计价值
- 掌握 RoughJS 库的深度应用
- 学会自定义手绘效果算法

**核心内容**
```javascript
// 9.1 RoughJS 深度应用
import rough from 'roughjs';

class RoughRenderer {
    constructor(canvas) {
        this.canvas = canvas;
        this.rc = rough.canvas(canvas);
        this.seedManager = new SeedManager();
    }
    
    renderElement(element) {
        // 使用元素的种子确保一致性
        const seed = this.seedManager.getSeed(element.id);
        
        const options = {
            roughness: element.roughness,
            stroke: element.strokeColor,
            strokeWidth: element.strokeWidth,
            fill: element.backgroundColor,
            fillStyle: element.fillStyle,
            seed: seed,
            
            // 高级选项
            bowing: this.calculateBowing(element),
            maxRandomnessOffset: element.roughness * 2,
            curveStepCount: this.getCurveStepCount(element),
            curveFitting: 0.95
        };
        
        switch (element.type) {
            case 'rectangle':
                this.rc.rectangle(element.x, element.y, element.width, element.height, options);
                break;
            case 'ellipse':
                this.rc.ellipse(
                    element.x + element.width / 2,
                    element.y + element.height / 2,
                    element.width,
                    element.height,
                    options
                );
                break;
            case 'line':
                this.renderRoughLine(element, options);
                break;
        }
    }
    
    renderRoughLine(element, options) {
        if (element.points.length < 2) return;
        
        // 为长线条创建更自然的分段
        const segments = this.segmentLine(element.points);
        
        segments.forEach(segment => {
            this.rc.line(
                segment.start.x, segment.start.y,
                segment.end.x, segment.end.y,
                options
            );
        });
    }
    
    segmentLine(points) {
        // 将长线条分解为更短的段，增加自然感
        const segments = [];
        const maxSegmentLength = 100;
        
        for (let i = 0; i < points.length - 1; i++) {
            const start = points[i];
            const end = points[i + 1];
            const distance = Math.hypot(end.x - start.x, end.y - start.y);
            
            if (distance <= maxSegmentLength) {
                segments.push({ start, end });
            } else {
                // 分割长线段
                const numSegments = Math.ceil(distance / maxSegmentLength);
                for (let j = 0; j < numSegments; j++) {
                    const t1 = j / numSegments;
                    const t2 = (j + 1) / numSegments;
                    
                    segments.push({
                        start: this.interpolatePoint(start, end, t1),
                        end: this.interpolatePoint(start, end, t2)
                    });
                }
            }
        }
        
        return segments;
    }
}

// 9.2 自定义手绘算法
class HandDrawnRenderer {
    constructor() {
        this.roughnessVariations = new Map();
    }
    
    drawRoughRectangle(ctx, x, y, width, height, options = {}) {
        const { roughness = 1, iterations = 2 } = options;
        
        // 生成变化的角点位置
        const corners = this.generateRoughCorners(x, y, width, height, roughness);
        
        // 多次描边增加手绘感
        for (let i = 0; i < iterations; i++) {
            ctx.beginPath();
            
            // 使用贝塞尔曲线连接角点
            this.drawRoughPath(ctx, corners, roughness * (1 - i * 0.3));
            
            ctx.stroke();
        }
    }
    
    generateRoughCorners(x, y, width, height, roughness) {
        const offset = roughness * 4;
        
        return [
            {
                x: x + this.randomOffset(offset),
                y: y + this.randomOffset(offset)
            },
            {
                x: x + width + this.randomOffset(offset),
                y: y + this.randomOffset(offset)
            },
            {
                x: x + width + this.randomOffset(offset),
                y: y + height + this.randomOffset(offset)
            },
            {
                x: x + this.randomOffset(offset),
                y: y + height + this.randomOffset(offset)
            }
        ];
    }
    
    drawRoughPath(ctx, points, roughness) {
        if (points.length < 2) return;
        
        ctx.moveTo(points[0].x, points[0].y);
        
        for (let i = 1; i < points.length; i++) {
            const current = points[i];
            const previous = points[i - 1];
            
            // 添加控制点创建略微弯曲的线条
            const controlX = (previous.x + current.x) / 2 + this.randomOffset(roughness);
            const controlY = (previous.y + current.y) / 2 + this.randomOffset(roughness);
            
            ctx.quadraticCurveTo(controlX, controlY, current.x, current.y);
        }
        
        // 闭合路径
        const first = points[0];
        const last = points[points.length - 1];
        const controlX = (last.x + first.x) / 2 + this.randomOffset(roughness);
        const controlY = (last.y + first.y) / 2 + this.randomOffset(roughness);
        
        ctx.quadraticCurveTo(controlX, controlY, first.x, first.y);
        ctx.closePath();
    }
    
    randomOffset(maxOffset) {
        return (Math.random() - 0.5) * maxOffset * 2;
    }
}

// 9.3 种子管理器（确保一致性）
class SeedManager {
    constructor() {
        this.seeds = new Map();
        this.baseSeeds = new Map();
    }
    
    getSeed(elementId) {
        if (!this.seeds.has(elementId)) {
            // 基于元素ID生成确定性的种子
            this.seeds.set(elementId, this.generateSeed(elementId));
        }
        return this.seeds.get(elementId);
    }
    
    generateSeed(input) {
        // 简单的字符串哈希函数
        let hash = 0;
        for (let i = 0; i < input.length; i++) {
            const char = input.charCodeAt(i);
            hash = ((hash << 5) - hash) + char;
            hash = hash & hash; // 转换为32位整数
        }
        return Math.abs(hash) % 1000000;
    }
    
    // 为特定属性生成种子变体
    getPropertySeed(elementId, property) {
        const baseSeed = this.getSeed(elementId);
        const propertyHash = this.generateSeed(property);
        return (baseSeed + propertyHash) % 1000000;
    }
}

// 9.4 填充样式实现
class RoughFillRenderer {
    constructor() {
        this.patterns = {
            'hachure': this.renderHachureFill,
            'cross-hatch': this.renderCrossHatchFill,
            'dots': this.renderDotsFill,
            'zigzag': this.renderZigzagFill
        };
    }
    
    renderFill(ctx, element, fillStyle) {
        const pattern = this.patterns[fillStyle];
        if (pattern) {
            pattern.call(this, ctx, element);
        }
    }
    
    renderHachureFill(ctx, element) {
        const { x, y, width, height } = element;
        const spacing = 8 + element.roughness * 2;
        const angle = Math.PI / 6; // 30度
        
        ctx.save();
        ctx.clip(); // 限制在元素边界内
        
        // 绘制斜线填充
        const cos = Math.cos(angle);
        const sin = Math.sin(angle);
        const diagonal = width + height;
        
        for (let i = -diagonal; i < diagonal; i += spacing) {
            const startX = x + i * cos;
            const startY = y + i * sin;
            const endX = startX + height * sin;
            const endY = startY - height * cos;
            
            // 添加随机偏移
            const offset = element.roughness;
            
            ctx.beginPath();
            ctx.moveTo(
                startX + this.randomOffset(offset),
                startY + this.randomOffset(offset)
            );
            ctx.lineTo(
                endX + this.randomOffset(offset),
                endY + this.randomOffset(offset)
            );
            ctx.stroke();
        }
        
        ctx.restore();
    }
    
    renderCrossHatchFill(ctx, element) {
        // 先绘制一个方向的线条
        this.renderHachureFill(ctx, element);
        
        // 再绘制垂直方向的线条
        const originalRoughness = element.roughness;
        element.roughness *= 0.8; // 第二层稍微规整一些
        
        ctx.save();
        ctx.rotate(Math.PI / 2);
        this.renderHachureFill(ctx, element);
        ctx.restore();
        
        element.roughness = originalRoughness;
    }
}
```

**实践项目**：手绘风格渲染器
- 集成 RoughJS 渲染系统
- 实现多种填充样式
- 优化手绘效果的性能

### 第十周：文件格式与数据持久化

**学习目标**
- 设计完整的文件格式规范
- 实现多格式的导入导出功能
- 掌握数据压缩和优化技术

**核心内容**
```typescript
// 10.1 Excalidraw 文件格式设计
interface ExcalidrawFile {
    type: 'excalidraw';
    version: 2;
    source: string;
    elements: ExcalidrawElement[];
    appState: AppState;
    files: Record<FileId, BinaryFileData>;
}

interface BinaryFileData {
    mimeType: string;
    id: FileId;
    dataURL: DataURL;
    created: number;
    lastRetrieved?: number;
}

// 10.2 序列化系统
class SerializationManager {
    // 序列化为 JSON
    static serializeToJSON(scene) {
        const serialized = {
            type: 'excalidraw',
            version: 2,
            source: window.location.origin,
            elements: scene.elements.map(this.serializeElement),
            appState: this.serializeAppState(scene.appState),
            files: this.serializeFiles(scene.files)
        };
        
        return JSON.stringify(serialized, null, 2);
    }
    
    static serializeElement(element) {
        const {
            // 排除运行时属性
            isSelected,
            isDragging,
            isResizing,
            ...serializable
        } = element;
        
        return serializable;
    }
    
    // 压缩序列化（用于网络传输）
    static serializeCompressed(scene) {
        const json = this.serializeToJSON(scene);
        return this.compressString(json);
    }
    
    static compressString(str) {
        // 使用 pako 进行 gzip 压缩
        const compressed = pako.deflate(str, { to: 'string' });
        return btoa(compressed); // base64 编码
    }
    
    static decompressString(compressed) {
        const binaryString = atob(compressed);
        const decompressed = pako.inflate(binaryString, { to: 'string' });
        return decompressed;
    }
}

// 10.3 导出系统
class ExportManager {
    constructor() {
        this.exporters = {
            png: new PNGExporter(),
            svg: new SVGExporter(),
            json: new JSONExporter(),
            pdf: new PDFExporter()
        };
    }
    
    async export(scene, format, options = {}) {
        const exporter = this.exporters[format];
        if (!exporter) {
            throw new Error(`Unsupported export format: ${format}`);
        }
        
        return exporter.export(scene, options);
    }
}

// PNG 导出器
class PNGExporter {
    async export(scene, options = {}) {
        const {
            scale = 2,
            background = true,
            padding = 10,
            darkMode = false
        } = options;
        
        // 计算场景边界
        const bounds = this.getSceneBounds(scene.elements);
        const exportBounds = {
            x: bounds.minX - padding,
            y: bounds.minY - padding,
            width: bounds.maxX - bounds.minX + padding * 2,
            height: bounds.maxY - bounds.minY + padding * 2
        };
        
        // 创建离屏 Canvas
        const canvas = document.createElement('canvas');
        canvas.width = exportBounds.width * scale;
        canvas.height = exportBounds.height * scale;
        
        const ctx = canvas.getContext('2d');
        ctx.scale(scale, scale);
        ctx.translate(-exportBounds.x, -exportBounds.y);
        
        // 设置背景
        if (background) {
            ctx.fillStyle = darkMode ? '#121212' : '#ffffff';
            ctx.fillRect(
                exportBounds.x,
                exportBounds.y,
                exportBounds.width,
                exportBounds.height
            );
        }
        
        // 渲染所有元素
        const renderer = new ExportRenderer(ctx, { darkMode });
        scene.elements.forEach(element => {
            if (!element.isDeleted) {
                renderer.renderElement(element);
            }
        });
        
        return canvas.toDataURL('image/png');
    }
    
    getSceneBounds(elements) {
        if (elements.length === 0) {
            return { minX: 0, minY: 0, maxX: 100, maxY: 100 };
        }
        
        let minX = Infinity, minY = Infinity;
        let maxX = -Infinity, maxY = -Infinity;
        
        elements.forEach(element => {
            if (element.isDeleted) return;
            
            const bounds = ElementOperations.getBounds(element);
            minX = Math.min(minX, bounds.x);
            minY = Math.min(minY, bounds.y);
            maxX = Math.max(maxX, bounds.x + bounds.width);
            maxY = Math.max(maxY, bounds.y + bounds.height);
        });
        
        return { minX, minY, maxX, maxY };
    }
}

// SVG 导出器
class SVGExporter {
    export(scene, options = {}) {
        const { darkMode = false } = options;
        
        const bounds = new PNGExporter().getSceneBounds(scene.elements);
        const padding = 10;
        
        const svgContent = [
            `<?xml version="1.0" encoding="UTF-8"?>`,
            `<svg`,
            `  width="${bounds.maxX - bounds.minX + padding * 2}"`,
            `  height="${bounds.maxY - bounds.minY + padding * 2}"`,
            `  viewBox="${bounds.minX - padding} ${bounds.minY - padding} ${bounds.maxX - bounds.minX + padding * 2} ${bounds.maxY - bounds.minY + padding * 2}"`,
            `  xmlns="http://www.w3.org/2000/svg"`,
            `  xmlns:xlink="http://www.w3.org/1999/xlink">`,
            
            // 定义样式
            this.generateSVGStyles(darkMode),
            
            // 渲染元素
            ...scene.elements
                .filter(el => !el.isDeleted)
                .map(el => this.elementToSVG(el)),
            
            `</svg>`
        ];
        
        return svgContent.join('\n');
    }
    
    elementToSVG(element) {
        const commonAttrs = [
            `stroke="${element.strokeColor}"`,
            `stroke-width="${element.strokeWidth}"`,
            `fill="${element.backgroundColor === 'transparent' ? 'none' : element.backgroundColor}"`,
            `opacity="${element.opacity / 100}"`
        ];
        
        if (element.angle !== 0) {
            const centerX = element.x + element.width / 2;
            const centerY = element.y + element.height / 2;
            commonAttrs.push(`transform="rotate(${element.angle * 180 / Math.PI} ${centerX} ${centerY})"`);
        }
        
        switch (element.type) {
            case 'rectangle':
                return `<rect x="${element.x}" y="${element.y}" width="${element.width}" height="${element.height}" ${commonAttrs.join(' ')} />`;
                
            case 'ellipse':
                return `<ellipse cx="${element.x + element.width / 2}" cy="${element.y + element.height / 2}" rx="${element.width / 2}" ry="${element.height / 2}" ${commonAttrs.join(' ')} />`;
                
            case 'line':
            case 'arrow':
                return this.linearElementToSVG(element, commonAttrs);
                
            case 'text':
                return this.textElementToSVG(element, commonAttrs);
                
            default:
                return '';
        }
    }
    
    linearElementToSVG(element, commonAttrs) {
        if (element.points.length < 2) return '';
        
        const pathData = element.points
            .map((point, index) => `${index === 0 ? 'M' : 'L'} ${point.x} ${point.y}`)
            .join(' ');
        
        let svg = `<path d="${pathData}" ${commonAttrs.join(' ')} />`;
        
        // 添加箭头
        if (element.endArrowhead) {
            svg += this.generateArrowhead(element, 'end');
        }
        if (element.startArrowhead) {
            svg += this.generateArrowhead(element, 'start');
        }
        
        return svg;
    }
}

// 10.4 自动保存系统
class AutoSaveManager {
    constructor() {
        this.saveInterval = 30000; // 30秒
        this.maxBackups = 10;
        this.isEnabled = true;
        this.lastSaveTime = 0;
        this.changesSinceLastSave = false;
        
        this.setupAutoSave();
    }
    
    setupAutoSave() {
        // 监听数据变化
        this.observeChanges();
        
        // 定期保存
        setInterval(() => {
            if (this.isEnabled && this.changesSinceLastSave) {
                this.performAutoSave();
            }
        }, this.saveInterval);
        
        // 页面关闭前保存
        window.addEventListener('beforeunload', () => {
            if (this.changesSinceLastSave) {
                this.performAutoSave();
            }
        });
    }
    
    observeChanges() {
        // 使用 Proxy 监听状态变化
        const originalElementsAtom = elementsAtom;
        
        elementsAtom.onMount = () => {
            this.markChanged();
        };
    }
    
    async performAutoSave() {
        try {
            const scene = this.getCurrentScene();
            const serialized = SerializationManager.serializeToJSON(scene);
            
            // 保存到 IndexedDB
            await this.saveToIndexedDB(serialized);
            
            // 管理备份数量
            await this.cleanupOldBackups();
            
            this.changesSinceLastSave = false;
            this.lastSaveTime = Date.now();
            
            console.log('Auto-save completed');
        } catch (error) {
            console.error('Auto-save failed:', error);
        }
    }
    
    async saveToIndexedDB(data) {
        const db = await this.openDB();
        const transaction = db.transaction(['autosaves'], 'readwrite');
        const store = transaction.objectStore('autosaves');
        
        await store.put({
            id: Date.now(),
            timestamp: new Date().toISOString(),
            data: data
        });
    }
    
    async openDB() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open('ExcalidrawAutoSave', 1);
            
            request.onerror = () => reject(request.error);
            request.onsuccess = () => resolve(request.result);
            
            request.onupgradeneeded = (event) => {
                const db = event.target.result;
                db.createObjectStore('autosaves', { keyPath: 'id' });
            };
        });
    }
}
```

**实践项目**：完整的数据管理系统
- 实现多格式导出功能
- 添加自动保存机制
- 支持文件恢复功能

---

### 第十一周：实时协作系统

**学习目标**
- 理解实时协作的技术挑战
- 掌握 WebSocket 和 WebRTC 的应用
- 学会实现冲突解决算法

**核心内容**
```javascript
// 11.1 协作客户端
class CollaborationClient {
    constructor(roomId, userId) {
        this.roomId = roomId;
        this.userId = userId;
        this.socket = null;
        this.isConnected = false;
        
        // 协作状态
        this.collaborators = new Map();
        this.pendingUpdates = [];
        this.operationQueue = new OperationQueue();
        
        this.setupConnection();
    }
    
    async setupConnection() {
        try {
            // 建立 WebSocket 连接
            this.socket = io(`${SERVER_URL}/collaboration`);
            
            this.socket.on('connect', this.handleConnect.bind(this));
            this.socket.on('disconnect', this.handleDisconnect.bind(this));
            this.socket.on('room-joined', this.handleRoomJoined.bind(this));
            this.socket.on('element-update', this.handleElementUpdate.bind(this));
            this.socket.on('cursor-update', this.handleCursorUpdate.bind(this));
            this.socket.on('user-joined', this.handleUserJoined.bind(this));
            this.socket.on('user-left', this.handleUserLeft.bind(this));
            
            // 加入房间
            await this.joinRoom();
            
        } catch (error) {
            console.error('Failed to setup collaboration:', error);
            this.handleConnectionError(error);
        }
    }
    
    async joinRoom() {
        return new Promise((resolve, reject) => {
            this.socket.emit('join-room', {
                roomId: this.roomId,
                userId: this.userId,
                userInfo: {
                    name: this.getUserName(),
                    color: this.getUserColor()
                }
            }, (response) => {
                if (response.success) {
                    resolve(response.data);
                } else {
                    reject(new Error(response.error));
                }
            });
        });
    }
    
    // 广播元素更新
    broadcastElementUpdate(element, operation = 'update') {
        const update = {
            type: 'element-update',
            operation,
            element: SerializationManager.serializeElement(element),
            timestamp: Date.now(),
            userId: this.userId,
            version: element.version
        };
        
        // 添加到操作队列
        this.operationQueue.enqueue(update);
        
        // 立即发送
        this.socket.emit('element-update', update);
    }
    
    // 处理远程元素更新
    handleElementUpdate(update) {
        if (update.userId === this.userId) return;
        
        try {
            // 应用操作变换
            const transformedUpdate = this.operationQueue.transform(update);
            
            // 更新本地状态
            this.applyRemoteUpdate(transformedUpdate);
            
        } catch (error) {
            console.error('Failed to apply remote update:', error);
            this.requestFullSync();
        }
    }
    
    applyRemoteUpdate(update) {
        const { operation, element } = update;
        
        switch (operation) {
            case 'create':
                this.createElement(element);
                break;
            case 'update':
                this.updateElement(element);
                break;
            case 'delete':
                this.deleteElement(element.id);
                break;
            case 'move':
                this.moveElement(element.id, element.x, element.y);
                break;
        }
        
        // 触发重新渲染
        this.notifyStateChange();
    }
}

// 11.2 操作变换系统
class OperationQueue {
    constructor() {
        this.localOps = [];
        this.remoteOps = [];
        this.acknowledged = [];
    }
    
    // 添加本地操作
    enqueue(operation) {
        operation.id = this.generateOperationId();
        operation.clientSequence = this.localOps.length;
        this.localOps.push(operation);
    }
    
    // 变换远程操作
    transform(remoteOp) {
        // 找到需要变换的本地操作
        const unacknowledgedOps = this.localOps.filter(op => 
            !this.acknowledged.includes(op.id) && 
            op.timestamp < remoteOp.timestamp
        );
        
        // 应用操作变换
        let transformedOp = remoteOp;
        
        unacknowledgedOps.forEach(localOp => {
            transformedOp = this.transformOperation(transformedOp, localOp);
        });
        
        return transformedOp;
    }
    
    // 核心变换逻辑
    transformOperation(op1, op2) {
        // 如果操作作用于不同元素，无需变换
        if (op1.element?.id !== op2.element?.id) {
            return op1;
        }
        
        const element1 = op1.element;
        const element2 = op2.element;
        
        // 位置变换
        if (op1.operation === 'move' && op2.operation === 'move') {
            // 两个移动操作：保持相对偏移
            return {
                ...op1,
                element: {
                    ...element1,
                    x: element1.x + (element2.x - element2.prevX || 0),
                    y: element1.y + (element2.y - element2.prevY || 0)
                }
            };
        }
        
        // 删除操作的变换
        if (op2.operation === 'delete') {
            // 如果元素已被删除，忽略其他操作
            return null;
        }
        
        // 属性更新的变换
        if (op1.operation === 'update' && op2.operation === 'update') {
            // 合并属性更新（后写者胜利 + 时间戳优先级）
            const mergedElement = {
                ...element2,
                ...element1,
                version: Math.max(element1.version, element2.version) + 1
            };
            
            return {
                ...op1,
                element: mergedElement
            };
        }
        
        return op1;
    }
    
    // 确认操作
    acknowledge(operationId) {
        this.acknowledged.push(operationId);
        
        // 清理已确认的操作
        this.cleanup();
    }
    
    cleanup() {
        // 保留最近的一些操作用于变换
        const keepCount = 100;
        if (this.acknowledged.length > keepCount) {
            this.acknowledged = this.acknowledged.slice(-keepCount);
        }
    }
}

// 11.3 冲突解决器
class ConflictResolver {
    constructor() {
        this.strategies = {
            'last-write-wins': this.lastWriteWins,
            'semantic-merge': this.semanticMerge,
            'user-priority': this.userPriority
        };
    }
    
    resolve(conflict, strategy = 'semantic-merge') {
        const resolver = this.strategies[strategy];
        if (!resolver) {
            throw new Error(`Unknown conflict resolution strategy: ${strategy}`);
        }
        
        return resolver.call(this, conflict);
    }
    
    // 后写者胜利策略
    lastWriteWins(conflict) {
        const { local, remote } = conflict;
        
        if (remote.timestamp > local.timestamp) {
            return { winner: remote, resolution: 'accept-remote' };
        } else if (local.timestamp > remote.timestamp) {
            return { winner: local, resolution: 'keep-local' };
        } else {
            // 时间戳相同，比较用户ID
            return local.userId < remote.userId ? 
                { winner: local, resolution: 'keep-local' } :
                { winner: remote, resolution: 'accept-remote' };
        }
    }
    
    // 语义合并策略
    semanticMerge(conflict) {
        const { local, remote } = conflict;
        
        // 尝试智能合并不同属性的更改
        const merged = this.mergeElementChanges(local.element, remote.element);
        
        if (merged) {
            return { 
                winner: merged, 
                resolution: 'merged',
                conflicts: this.detectRemainingConflicts(local.element, remote.element, merged)
            };
        }
        
        // 无法合并，回退到后写者胜利
        return this.lastWriteWins(conflict);
    }
    
    mergeElementChanges(localElement, remoteElement) {
        // 找出变更的属性
        const localChanges = this.getChangedProperties(localElement);
        const remoteChanges = this.getChangedProperties(remoteElement);
        
        // 检查冲突
        const conflictingProps = localChanges.filter(prop => 
            remoteChanges.includes(prop)
        );
        
        if (conflictingProps.length === 0) {
            // 无冲突，可以安全合并
            return {
                ...localElement,
                ...this.extractChanges(remoteElement, remoteChanges),
                version: Math.max(localElement.version, remoteElement.version) + 1
            };
        }
        
        // 有冲突，需要进一步处理
        return this.resolvePropertyConflicts(localElement, remoteElement, conflictingProps);
    }
    
    resolvePropertyConflicts(local, remote, conflictingProps) {
        const resolved = { ...local };
        
        conflictingProps.forEach(prop => {
            switch (prop) {
                case 'x':
                case 'y':
                    // 位置冲突：取平均值
                    resolved[prop] = (local[prop] + remote[prop]) / 2;
                    break;
                    
                case 'strokeColor':
                case 'backgroundColor':
                    // 颜色冲突：优先保留非默认颜色
                    if (remote[prop] !== '#000000' && remote[prop] !== 'transparent') {
                        resolved[prop] = remote[prop];
                    }
                    break;
                    
                case 'text':
                    // 文本冲突：标记冲突供用户解决
                    resolved[prop] = `${local[prop]} [CONFLICT] ${remote[prop]}`;
                    break;
                    
                default:
                    // 默认：后写者胜利
                    if (remote.timestamp > local.timestamp) {
                        resolved[prop] = remote[prop];
                    }
            }
        });
        
        return resolved;
    }
}

// 11.4 协作用户界面
class CollaborationUI {
    constructor(collaborationClient) {
        this.client = collaborationClient;
        this.cursors = new Map();
        this.userList = new Map();
        
        this.setupUI();
    }
    
    setupUI() {
        // 创建协作者列表
        this.userListElement = document.createElement('div');
        this.userListElement.className = 'collaboration-users';
        document.body.appendChild(this.userListElement);
        
        // 创建光标容器
        this.cursorsContainer = document.createElement('div');
        this.cursorsContainer.className = 'collaboration-cursors';
        document.body.appendChild(this.cursorsContainer);
        
        // 监听协作事件
        this.client.on('user-joined', this.handleUserJoined.bind(this));
        this.client.on('user-left', this.handleUserLeft.bind(this));
        this.client.on('cursor-update', this.updateUserCursor.bind(this));
    }
    
    handleUserJoined(user) {
        this.userList.set(user.id, user);
        this.updateUserList();
        
        // 显示加入通知
        this.showNotification(`${user.name} joined the session`);
    }
    
    updateUserCursor(data) {
        const { userId, x, y, pointer } = data;
        
        let cursor = this.cursors.get(userId);
        if (!cursor) {
            cursor = this.createCursor(userId);
            this.cursors.set(userId, cursor);
        }
        
        // 更新光标位置
        cursor.style.transform = `translate(${x}px, ${y}px)`;
        cursor.style.display = 'block';
        
        // 更新光标状态
        cursor.classList.toggle('drawing', pointer.isDrawing);
        cursor.classList.toggle('selecting', pointer.isSelecting);
        
        // 隐藏非活跃光标
        clearTimeout(cursor.hideTimer);
        cursor.hideTimer = setTimeout(() => {
            cursor.style.display = 'none';
        }, 3000);
    }
    
    createCursor(userId) {
        const user = this.userList.get(userId);
        const cursor = document.createElement('div');
        cursor.className = 'collaboration-cursor';
        cursor.style.borderColor = user.color;
        
        // 添加用户名标签
        const label = document.createElement('div');
        label.className = 'cursor-label';
        label.textContent = user.name;
        label.style.backgroundColor = user.color;
        cursor.appendChild(label);
        
        this.cursorsContainer.appendChild(cursor);
        return cursor;
    }
    
    // 广播本地光标位置
    broadcastCursor(x, y, pointer = {}) {
        this.client.socket.emit('cursor-update', {
            x, y, pointer,
            userId: this.client.userId,
            timestamp: Date.now()
        });
    }
}
```

**实践项目**：实时协作画板
- 实现多人实时协作
- 添加冲突解决机制
- 支持协作者光标显示

### 第十二周：性能监控与优化

**学习目标**
- 学会性能分析和监控技术
- 掌握内存优化策略
- 理解渲染性能优化原理

**核心内容**
```javascript
// 12.1 性能监控系统
class PerformanceMonitor {
    constructor() {
        this.metrics = {
            fps: new FPSCounter(),
            memory: new MemoryMonitor(),
            render: new RenderTimeTracker(),
            interactions: new InteractionTracker()
        };
        
        this.thresholds = {
            fps: { warning: 45, critical: 30 },
            memory: { warning: 50 * 1024 * 1024, critical: 100 * 1024 * 1024 },
            renderTime: { warning: 16, critical: 33 }
        };
        
        this.setupMonitoring();
    }
    
    setupMonitoring() {
        // 定期收集指标
        setInterval(() => {
            this.collectMetrics();
            this.checkThresholds();
        }, 1000);
        
        // 监控 DOM 变化
        this.observeDOM();
        
        // 监控内存使用
        if (performance.memory) {
            this.monitorMemoryUsage();
        }
    }
    
    collectMetrics() {
        const currentMetrics = {
            timestamp: Date.now(),
            fps: this.metrics.fps.getFPS(),
            memory: this.metrics.memory.getUsage(),
            renderTime: this.metrics.render.getAverageTime(),
            elementCount: this.getElementCount(),
            canvasSize: this.getCanvasSize()
        };
        
        this.logMetrics(currentMetrics);
        return currentMetrics;
    }
    
    checkThresholds() {
        const current = this.collectMetrics();
        
        Object.entries(this.thresholds).forEach(([metric, thresholds]) => {
            const value = current[metric];
            
            if (value > thresholds.critical) {
                this.onCriticalThreshold(metric, value);
            } else if (value > thresholds.warning) {
                this.onWarningThreshold(metric, value);
            }
        });
    }
    
    onCriticalThreshold(metric, value) {
        console.warn(`Critical performance issue: ${metric} = ${value}`);
        
        // 自动应用优化策略
        this.applyEmergencyOptimizations(metric);
        
        // 通知用户
        this.notifyUser(`Performance warning: ${metric} is critically high`);
    }
    
    applyEmergencyOptimizations(metric) {
        switch (metric) {
            case 'renderTime':
                // 降低渲染质量
                this.enableLowQualityMode();
                break;
            case 'memory':
                // 清理缓存
                this.clearNonEssentialCaches();
                break;
            case 'fps':
                // 减少渲染频率
                this.throttleRendering();
                break;
        }
    }
}

// 12.2 内存优化管理
class MemoryManager {
    constructor() {
        this.caches = new Map();
        this.objectPools = new Map();
        this.weakRefs = new Set();
        
        this.setupMemoryOptimizations();
    }
    
    // 对象池模式
    createObjectPool(type, factory, resetFn) {
        const pool = {
            objects: [],
            factory,
            reset: resetFn,
            acquired: 0,
            created: 0
        };
        
        this.objectPools.set(type, pool);
        return pool;
    }
    
    acquire(type) {
        const pool = this.objectPools.get(type);
        if (!pool) return null;
        
        let obj;
        if (pool.objects.length > 0) {
            obj = pool.objects.pop();
            pool.reset(obj);
        } else {
            obj = pool.factory();
            pool.created++;
        }
        
        pool.acquired++;
        return obj;
    }
    
    release(type, obj) {
        const pool = this.objectPools.get(type);
        if (!pool) return;
        
        pool.objects.push(obj);
        pool.acquired--;
    }
    
    // 智能缓存管理
    createSmartCache(name, maxSize = 100, ttl = 5 * 60 * 1000) {
        const cache = {
            data: new Map(),
            accessTimes: new Map(),
            maxSize,
            ttl,
            hits: 0,
            misses: 0
        };
        
        this.caches.set(name, cache);
        return cache;
    }
    
    get(cacheName, key) {
        const cache = this.caches.get(cacheName);
        if (!cache) return undefined;
        
        const entry = cache.data.get(key);
        if (!entry) {
            cache.misses++;
            return undefined;
        }
        
        // 检查 TTL
        if (Date.now() - entry.timestamp > cache.ttl) {
            cache.data.delete(key);
            cache.accessTimes.delete(key);
            cache.misses++;
            return undefined;
        }
        
        // 更新访问时间
        cache.accessTimes.set(key, Date.now());
        cache.hits++;
        
        return entry.value;
    }
    
    set(cacheName, key, value) {
        const cache = this.caches.get(cacheName);
        if (!cache) return;
        
        // LRU 清理
        if (cache.data.size >= cache.maxSize) {
            this.evictLRU(cache);
        }
        
        cache.data.set(key, {
            value,
            timestamp: Date.now()
        });
        cache.accessTimes.set(key, Date.now());
    }
    
    evictLRU(cache) {
        let oldestKey = null;
        let oldestTime = Date.now();
        
        cache.accessTimes.forEach((time, key) => {
            if (time < oldestTime) {
                oldestTime = time;
                oldestKey = key;
            }
        });
        
        if (oldestKey) {
            cache.data.delete(oldestKey);
            cache.accessTimes.delete(oldestKey);
        }
    }
    
    // 内存泄漏检测
    detectMemoryLeaks() {
        const report = {
            objectPools: {},
            caches: {},
            weakRefs: this.weakRefs.size,
            recommendations: []
        };
        
        // 检查对象池
        this.objectPools.forEach((pool, type) => {
            report.objectPools[type] = {
                created: pool.created,
                pooled: pool.objects.length,
                acquired: pool.acquired,
                efficiency: pool.objects.length / pool.created
            };
            
            if (pool.efficiency < 0.5) {
                report.recommendations.push(
                    `Object pool '${type}' has low efficiency (${(pool.efficiency * 100).toFixed(1)}%)`
                );
            }
        });
        
        // 检查缓存
        this.caches.forEach((cache, name) => {
            report.caches[name] = {
                size: cache.data.size,
                hitRate: cache.hits / (cache.hits + cache.misses),
                maxSize: cache.maxSize
            };
            
            if (cache.data.size === cache.maxSize) {
                report.recommendations.push(
                    `Cache '${name}' is at maximum capacity. Consider increasing size or decreasing TTL.`
                );
            }
        });
        
        return report;
    }
}

// 12.3 渲染性能优化
class RenderOptimizer {
    constructor(renderer) {
        this.renderer = renderer;
        this.optimizations = {
            culling: new ViewportCuller(),
            batching: new BatchRenderer(),
            lod: new LODRenderer(),
            caching: new RenderCache()
        };
        
        this.performanceMode = 'balanced'; // auto, balanced, quality, performance
        this.adaptiveOptimization = true;
    }
    
    optimizeRender(elements, viewport, options = {}) {
        const startTime = performance.now();
        
        // 1. 视口剔除
        let visibleElements = this.optimizations.culling.cull(elements, viewport);
        
        // 2. LOD 处理
        if (viewport.zoom < 0.5) {
            visibleElements = this.optimizations.lod.simplify(visibleElements, viewport.zoom);
        }
        
        // 3. 批量渲染优化
        const batches = this.optimizations.batching.createBatches(visibleElements);
        
        // 4. 缓存检查
        const cacheKey = this.generateCacheKey(visibleElements, viewport);
        const cached = this.optimizations.caching.get(cacheKey);
        
        if (cached && !this.hasChanges(cached.timestamp)) {
            return this.renderCached(cached);
        }
        
        // 5. 实际渲染
        const result = this.performRender(batches, viewport, options);
        
        // 6. 缓存结果
        this.optimizations.caching.set(cacheKey, {
            result,
            timestamp: Date.now(),
            elements: visibleElements.map(el => el.id)
        });
        
        const renderTime = performance.now() - startTime;
        this.trackRenderTime(renderTime);
        
        // 7. 自适应优化
        if (this.adaptiveOptimization) {
            this.adjustOptimizations(renderTime);
        }
        
        return result;
    }
    
    adjustOptimizations(renderTime) {
        if (renderTime > 33) { // 30 FPS
            this.enablePerformanceMode();
        } else if (renderTime < 8 && this.performanceMode === 'performance') { // 120+ FPS
            this.enableBalancedMode();
        }
    }
    
    enablePerformanceMode() {
        this.performanceMode = 'performance';
        
        // 激进的优化设置
        this.optimizations.culling.setMargin(0); // 严格剔除
        this.optimizations.lod.setAggressiveSimplification(true);
        this.optimizations.batching.setMaxBatchSize(1000);
        
        console.log('Switched to performance mode due to low FPS');
    }
    
    enableBalancedMode() {
        this.performanceMode = 'balanced';
        
        // 平衡的优化设置
        this.optimizations.culling.setMargin(50);
        this.optimizations.lod.setAggressiveSimplification(false);
        this.optimizations.batching.setMaxBatchSize(500);
        
        console.log('Switched to balanced mode');
    }
}

// 12.4 实时性能面板
class PerformancePanel {
    constructor() {
        this.panel = null;
        this.isVisible = false;
        this.charts = {};
        
        this.createPanel();
        this.setupCharts();
    }
    
    createPanel() {
        this.panel = document.createElement('div');
        this.panel.className = 'performance-panel';
        this.panel.style.cssText = `
            position: fixed;
            top: 10px;
            right: 10px;
            width: 300px;
            background: rgba(0, 0, 0, 0.9);
            color: white;
            padding: 10px;
            border-radius: 5px;
            font-family: monospace;
            font-size: 12px;
            z-index: 10000;
            display: none;
        `;
        
        document.body.appendChild(this.panel);
    }
    
    setupCharts() {
        // FPS 图表
        this.charts.fps = new MiniChart(50, 30);
        this.charts.fps.setRange(0, 60);
        this.charts.fps.setColor('#00ff00');
        
        // 内存图表
        this.charts.memory = new MiniChart(50, 30);
        this.charts.memory.setColor('#ff9900');
        
        // 渲染时间图表
        this.charts.renderTime = new MiniChart(50, 30);
        this.charts.renderTime.setRange(0, 33);
        this.charts.renderTime.setColor('#0099ff');
    }
    
    update(metrics) {
        if (!this.isVisible) return;
        
        // 更新图表
        this.charts.fps.addDataPoint(metrics.fps);
        this.charts.memory.addDataPoint(metrics.memory / 1024 / 1024); // MB
        this.charts.renderTime.addDataPoint(metrics.renderTime);
        
        // 更新面板内容
        this.panel.innerHTML = `
            <div><strong>Performance Monitor</strong></div>
            <div>FPS: ${metrics.fps.toFixed(1)} ${this.charts.fps.render()}</div>
            <div>Memory: ${(metrics.memory / 1024 / 1024).toFixed(1)}MB ${this.charts.memory.render()}</div>
            <div>Render: ${metrics.renderTime.toFixed(1)}ms ${this.charts.renderTime.render()}</div>
            <div>Elements: ${metrics.elementCount}</div>
            <div>Canvas: ${metrics.canvasSize}</div>
        `;
    }
    
    toggle() {
        this.isVisible = !this.isVisible;
        this.panel.style.display = this.isVisible ? 'block' : 'none';
    }
}

// 简单的迷你图表类
class MiniChart {
    constructor(width, height, maxPoints = 30) {
        this.width = width;
        this.height = height;
        this.maxPoints = maxPoints;
        this.data = [];
        this.min = 0;
        this.max = 100;
        this.color = '#ffffff';
    }
    
    setRange(min, max) {
        this.min = min;
        this.max = max;
    }
    
    setColor(color) {
        this.color = color;
    }
    
    addDataPoint(value) {
        this.data.push(value);
        if (this.data.length > this.maxPoints) {
            this.data.shift();
        }
    }
    
    render() {
        if (this.data.length < 2) return '';
        
        const canvas = document.createElement('canvas');
        canvas.width = this.width;
        canvas.height = this.height;
        canvas.style.verticalAlign = 'middle';
        
        const ctx = canvas.getContext('2d');
        ctx.strokeStyle = this.color;
        ctx.lineWidth = 1;
        
        ctx.beginPath();
        this.data.forEach((value, index) => {
            const x = (index / (this.data.length - 1)) * this.width;
            const y = this.height - ((value - this.min) / (this.max - this.min)) * this.height;
            
            if (index === 0) {
                ctx.moveTo(x, y);
            } else {
                ctx.lineTo(x, y);
            }
        });
        ctx.stroke();
        
        return canvas.outerHTML;
    }
}
```

**实践项目**：性能监控系统
- 实现完整的性能监控
- 添加自适应优化机制
- 构建实时性能面板

---

### 第十三周：移动端适配与触控优化

**学习目标**
- 掌握移动端触控事件处理
- 学会响应式设计在画板中的应用
- 理解移动端性能优化策略

### 第十四周：插件系统与扩展性设计

**学习目标**
- 设计可扩展的插件架构
- 学会 API 设计原则
- 掌握第三方集成技术

### 第十五周：项目部署与工程化

**学习目标**
- 掌握现代前端工程化流程
- 学会性能监控和错误追踪
- 理解 CDN 和缓存策略

---

## 🛠 实践项目规划

### 阶段一项目：Mini Canvas Editor (第1-5周)
构建一个基础的 Canvas 编辑器，包含基本绘图功能。

### 阶段二项目：Excalidraw Clone (第6-10周)
实现一个功能完整的 Excalidraw 克隆版本。

### 阶段三项目：协作画板应用 (第11-15周)
添加实时协作、性能优化等高级功能。

---

## 📊 学习评估体系

### 理论考核 (30%)
- 每周小测验
- 中期技术原理考试
- 期末综合测试

### 实践考核 (50%)
- 阶段性项目实现
- 代码质量评估
- 功能完整性检查

### 创新项目 (20%)
- 自定义功能扩展
- 性能优化方案
- 技术文档撰写

---

## 🎓 课程特色

1. **渐进式学习**：从简单到复杂，循序渐进
2. **理论与实践结合**：每个概念都有对应的代码实现
3. **真实项目驱动**：基于 Excalidraw 真实源码学习
4. **现代技术栈**：使用最新的前端技术和最佳实践
5. **可扩展架构**：学会设计大型前端应用

这门课程将让学员从零开始，逐步掌握构建现代化协作画板应用的完整技能体系，不仅学会技术实现，更重要的是理解背后的架构设计思想和工程实践。