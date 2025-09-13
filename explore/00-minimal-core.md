# Excalidraw 最小核心实现

这是一个可以直接运行的最小画板实现，包含了 Excalidraw 的核心功能。你可以直接复制代码到 HTML 文件中运行。

## 完整的最小实现（300行代码）

将以下代码保存为 `mini-excalidraw.html` 并在浏览器中打开：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Mini Excalidraw</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif;
            overflow: hidden;
        }
        
        #toolbar {
            position: fixed;
            top: 10px;
            left: 10px;
            background: white;
            border-radius: 8px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.15);
            padding: 8px;
            display: flex;
            gap: 4px;
            z-index: 1000;
        }
        
        .tool-btn {
            width: 40px;
            height: 40px;
            border: 2px solid #e0e0e0;
            background: white;
            border-radius: 8px;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
            transition: all 0.2s;
        }
        
        .tool-btn:hover {
            background: #f0f0f0;
        }
        
        .tool-btn.active {
            background: #4A90E2;
            color: white;
            border-color: #4A90E2;
        }
        
        #canvas {
            cursor: crosshair;
        }
        
        #canvas.tool-selection {
            cursor: default;
        }
        
        #canvas.tool-hand {
            cursor: grab;
        }
        
        #canvas.dragging {
            cursor: grabbing;
        }
    </style>
</head>
<body>
    <div id="toolbar">
        <button class="tool-btn active" data-tool="selection" title="Selection">
            <svg width="20" height="20" viewBox="0 0 20 20" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M3 3 L12 8 L7 9 L9 15 L7 16 L5 10 L3 12 Z"/>
            </svg>
        </button>
        <button class="tool-btn" data-tool="rectangle" title="Rectangle">
            <svg width="20" height="20" viewBox="0 0 20 20" fill="none" stroke="currentColor" stroke-width="2">
                <rect x="4" y="4" width="12" height="12"/>
            </svg>
        </button>
        <button class="tool-btn" data-tool="ellipse" title="Ellipse">
            <svg width="20" height="20" viewBox="0 0 20 20" fill="none" stroke="currentColor" stroke-width="2">
                <ellipse cx="10" cy="10" rx="7" ry="5"/>
            </svg>
        </button>
        <button class="tool-btn" data-tool="line" title="Line">
            <svg width="20" height="20" viewBox="0 0 20 20" fill="none" stroke="currentColor" stroke-width="2">
                <line x1="4" y1="16" x2="16" y2="4"/>
            </svg>
        </button>
        <button class="tool-btn" data-tool="freedraw" title="Free Draw">
            <svg width="20" height="20" viewBox="0 0 20 20" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M4 10 Q6 4, 10 8 T16 10"/>
            </svg>
        </button>
        <button class="tool-btn" data-tool="text" title="Text">
            <svg width="20" height="20" viewBox="0 0 20 20" fill="currentColor">
                <text x="10" y="15" text-anchor="middle" font-size="16" font-weight="bold">T</text>
            </svg>
        </button>
        <button class="tool-btn" data-tool="hand" title="Hand (Pan)">
            <svg width="20" height="20" viewBox="0 0 20 20" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M7 16 L7 6 L6 4 L6 10 M7 6 L8 4 L8 6 M8 6 L9 4 L9 6 M9 6 L10 5 L10 14 Q10 16, 8 16"/>
            </svg>
        </button>
    </div>
    
    <canvas id="canvas"></canvas>

    <script>
        // ============ 核心数据结构 ============
        let elements = [];
        let selectedElement = null;
        let currentTool = 'selection';
        let isDrawing = false;
        let isDragging = false;
        let isPanning = false;
        let startPoint = { x: 0, y: 0 };
        let currentElement = null;
        let panOffset = { x: 0, y: 0 };
        let dragOffset = { x: 0, y: 0 };

        // ============ Canvas 设置 ============
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        
        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
            redraw();
        }
        
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        // ============ 工具栏 ============
        document.querySelectorAll('.tool-btn').forEach(btn => {
            btn.addEventListener('click', (e) => {
                document.querySelectorAll('.tool-btn').forEach(b => b.classList.remove('active'));
                btn.classList.add('active');
                currentTool = btn.dataset.tool;
                canvas.className = `tool-${currentTool}`;
                selectedElement = null;
                redraw();
            });
        });

        // ============ 元素创建 ============
        function createElement(type, x, y) {
            return {
                id: Date.now() + Math.random(),
                type: type,
                x: x,
                y: y,
                width: 0,
                height: 0,
                points: type === 'freedraw' || type === 'line' ? [] : null,
                text: '',
                strokeColor: '#000000',
                backgroundColor: 'transparent',
                strokeWidth: 2,
                fontSize: 20,
                roughness: 1,
                seed: Math.floor(Math.random() * 100000)
            };
        }

        // ============ 绘制函数 ============
        function drawElement(element, isSelected = false) {
            ctx.save();
            
            // 设置样式
            ctx.strokeStyle = element.strokeColor;
            ctx.fillStyle = element.backgroundColor;
            ctx.lineWidth = element.strokeWidth;
            ctx.font = `${element.fontSize}px Virgil, Comic Sans MS, cursive`;

            // 添加手绘效果的辅助函数
            function roughLine(x1, y1, x2, y2) {
                const offset = element.roughness * 2;
                ctx.beginPath();
                ctx.moveTo(x1 + (Math.random() - 0.5) * offset, y1 + (Math.random() - 0.5) * offset);
                
                // 添加中间点让线条更自然
                const midX = (x1 + x2) / 2;
                const midY = (y1 + y2) / 2;
                ctx.quadraticCurveTo(
                    midX + (Math.random() - 0.5) * offset,
                    midY + (Math.random() - 0.5) * offset,
                    x2 + (Math.random() - 0.5) * offset,
                    y2 + (Math.random() - 0.5) * offset
                );
                ctx.stroke();
            }

            // 根据类型绘制
            switch (element.type) {
                case 'rectangle':
                    // 手绘矩形效果
                    roughLine(element.x, element.y, element.x + element.width, element.y);
                    roughLine(element.x + element.width, element.y, element.x + element.width, element.y + element.height);
                    roughLine(element.x + element.width, element.y + element.height, element.x, element.y + element.height);
                    roughLine(element.x, element.y + element.height, element.x, element.y);
                    
                    if (element.backgroundColor !== 'transparent') {
                        ctx.fillRect(element.x, element.y, element.width, element.height);
                    }
                    break;

                case 'ellipse':
                    ctx.beginPath();
                    const centerX = element.x + element.width / 2;
                    const centerY = element.y + element.height / 2;
                    
                    // 手绘椭圆 - 使用两个略有偏移的椭圆
                    for (let i = 0; i < 2; i++) {
                        ctx.beginPath();
                        ctx.ellipse(
                            centerX + (Math.random() - 0.5) * element.roughness,
                            centerY + (Math.random() - 0.5) * element.roughness,
                            Math.abs(element.width / 2),
                            Math.abs(element.height / 2),
                            0, 0, 2 * Math.PI
                        );
                        ctx.stroke();
                    }
                    
                    if (element.backgroundColor !== 'transparent') {
                        ctx.fill();
                    }
                    break;

                case 'line':
                    if (element.points && element.points.length >= 2) {
                        roughLine(element.points[0].x, element.points[0].y, 
                                element.points[1].x, element.points[1].y);
                    }
                    break;

                case 'freedraw':
                    if (element.points && element.points.length > 0) {
                        ctx.beginPath();
                        ctx.moveTo(element.points[0].x, element.points[0].y);
                        for (let i = 1; i < element.points.length; i++) {
                            ctx.lineTo(element.points[i].x, element.points[i].y);
                        }
                        ctx.stroke();
                    }
                    break;

                case 'text':
                    if (element.text) {
                        ctx.fillStyle = element.strokeColor;
                        const lines = element.text.split('\n');
                        lines.forEach((line, index) => {
                            ctx.fillText(line, element.x, element.y + (index + 1) * element.fontSize);
                        });
                    }
                    break;
            }

            // 绘制选择框
            if (isSelected) {
                ctx.strokeStyle = '#4A90E2';
                ctx.lineWidth = 1;
                ctx.setLineDash([5, 5]);
                
                const padding = 10;
                ctx.strokeRect(
                    element.x - padding,
                    element.y - padding,
                    (element.width || 100) + padding * 2,
                    (element.height || 30) + padding * 2
                );
                ctx.setLineDash([]);
            }

            ctx.restore();
        }

        function redraw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // 应用平移
            ctx.save();
            ctx.translate(panOffset.x, panOffset.y);
            
            // 绘制所有元素
            elements.forEach(element => {
                drawElement(element, element === selectedElement);
            });
            
            // 绘制当前正在创建的元素
            if (currentElement && isDrawing) {
                drawElement(currentElement);
            }
            
            ctx.restore();
        }

        // ============ 碰撞检测 ============
        function hitTest(x, y, element) {
            if (element.type === 'freedraw' || element.type === 'line') {
                // 简化的线条碰撞检测
                if (!element.points || element.points.length < 2) return false;
                
                for (let i = 0; i < element.points.length - 1; i++) {
                    const p1 = element.points[i];
                    const p2 = element.points[i + 1];
                    const dist = pointToLineDistance(x, y, p1.x, p1.y, p2.x, p2.y);
                    if (dist < 10) return true;
                }
                return false;
            } else if (element.type === 'text') {
                const width = element.text ? element.text.length * element.fontSize * 0.6 : 100;
                const height = element.fontSize * 1.5;
                return x >= element.x && x <= element.x + width &&
                       y >= element.y - element.fontSize && y <= element.y + height;
            } else {
                return x >= element.x && x <= element.x + element.width &&
                       y >= element.y && y <= element.y + element.height;
            }
        }

        function pointToLineDistance(px, py, x1, y1, x2, y2) {
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
                xx = x1;
                yy = y1;
            } else if (param > 1) {
                xx = x2;
                yy = y2;
            } else {
                xx = x1 + param * C;
                yy = y1 + param * D;
            }
            
            const dx = px - xx;
            const dy = py - yy;
            return Math.sqrt(dx * dx + dy * dy);
        }

        // ============ 鼠标事件 ============
        canvas.addEventListener('mousedown', (e) => {
            const x = e.offsetX - panOffset.x;
            const y = e.offsetY - panOffset.y;
            
            if (currentTool === 'hand') {
                isPanning = true;
                startPoint = { x: e.offsetX, y: e.offsetY };
                canvas.classList.add('dragging');
                return;
            }
            
            if (currentTool === 'selection') {
                // 查找点击的元素
                selectedElement = null;
                for (let i = elements.length - 1; i >= 0; i--) {
                    if (hitTest(x, y, elements[i])) {
                        selectedElement = elements[i];
                        isDragging = true;
                        dragOffset = {
                            x: x - selectedElement.x,
                            y: y - selectedElement.y
                        };
                        break;
                    }
                }
                redraw();
            } else if (currentTool === 'text') {
                const text = prompt('Enter text:');
                if (text) {
                    const element = createElement('text', x, y);
                    element.text = text;
                    elements.push(element);
                    redraw();
                }
            } else {
                // 开始绘制新元素
                isDrawing = true;
                startPoint = { x, y };
                currentElement = createElement(currentTool, x, y);
                
                if (currentTool === 'freedraw') {
                    currentElement.points = [{ x, y }];
                } else if (currentTool === 'line') {
                    currentElement.points = [{ x, y }, { x, y }];
                }
            }
        });

        canvas.addEventListener('mousemove', (e) => {
            const x = e.offsetX - panOffset.x;
            const y = e.offsetY - panOffset.y;
            
            if (isPanning) {
                panOffset.x += e.offsetX - startPoint.x;
                panOffset.y += e.offsetY - startPoint.y;
                startPoint = { x: e.offsetX, y: e.offsetY };
                redraw();
                return;
            }
            
            if (isDragging && selectedElement) {
                selectedElement.x = x - dragOffset.x;
                selectedElement.y = y - dragOffset.y;
                
                // 同时移动点
                if (selectedElement.points) {
                    const dx = x - dragOffset.x - selectedElement.x;
                    const dy = y - dragOffset.y - selectedElement.y;
                    selectedElement.points = selectedElement.points.map(p => ({
                        x: p.x + dx,
                        y: p.y + dy
                    }));
                }
                
                redraw();
            } else if (isDrawing && currentElement) {
                if (currentTool === 'rectangle' || currentTool === 'ellipse') {
                    currentElement.width = x - startPoint.x;
                    currentElement.height = y - startPoint.y;
                } else if (currentTool === 'line') {
                    currentElement.points[1] = { x, y };
                } else if (currentTool === 'freedraw') {
                    currentElement.points.push({ x, y });
                }
                redraw();
            }
        });

        canvas.addEventListener('mouseup', (e) => {
            if (isPanning) {
                isPanning = false;
                canvas.classList.remove('dragging');
                return;
            }
            
            if (isDragging) {
                isDragging = false;
            } else if (isDrawing && currentElement) {
                // 完成绘制
                if (currentTool === 'rectangle' || currentTool === 'ellipse') {
                    // 处理负值（向左上拖动）
                    if (currentElement.width < 0) {
                        currentElement.x += currentElement.width;
                        currentElement.width = Math.abs(currentElement.width);
                    }
                    if (currentElement.height < 0) {
                        currentElement.y += currentElement.height;
                        currentElement.height = Math.abs(currentElement.height);
                    }
                }
                
                // 只添加有效的元素
                if ((currentElement.width && currentElement.height) || 
                    (currentElement.points && currentElement.points.length > 1)) {
                    elements.push(currentElement);
                }
                
                isDrawing = false;
                currentElement = null;
                redraw();
            }
        });

        // ============ 键盘快捷键 ============
        document.addEventListener('keydown', (e) => {
            if (e.key === 'Delete' || e.key === 'Backspace') {
                if (selectedElement) {
                    elements = elements.filter(el => el !== selectedElement);
                    selectedElement = null;
                    redraw();
                }
            }
        });

        // 初始化
        redraw();
    </script>
</body>
</html>
```

## 功能特性

这个最小实现包含了：

### 1. 核心绘图功能
- **矩形工具** - 拖动创建矩形
- **椭圆工具** - 拖动创建椭圆
- **线条工具** - 点击拖动画直线
- **自由绘制** - 随手涂鸦
- **文本工具** - 点击添加文本

### 2. 交互功能
- **选择工具** - 点击选择元素，拖动移动
- **手型工具** - 拖动画布平移视图
- **删除功能** - 选中后按 Delete 键删除

### 3. 视觉效果
- **手绘风格** - 简单的随机扰动模拟手绘
- **选择框** - 虚线框显示选中状态

## 核心概念解析

### 1. 数据结构

```javascript
// 每个元素的基本结构
{
    id: 唯一标识,
    type: 元素类型,
    x: X坐标,
    y: Y坐标,
    width: 宽度,
    height: 高度,
    points: 点集合（线条类），
    strokeColor: 描边颜色,
    backgroundColor: 背景色,
    // ... 其他属性
}
```

### 2. 渲染循环

```javascript
function redraw() {
    // 1. 清空画布
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    
    // 2. 应用变换（平移等）
    ctx.save();
    ctx.translate(panOffset.x, panOffset.y);
    
    // 3. 绘制所有元素
    elements.forEach(element => {
        drawElement(element);
    });
    
    // 4. 恢复变换
    ctx.restore();
}
```

### 3. 工具状态机

每个工具有不同的行为：
- **mousedown** - 开始操作
- **mousemove** - 更新操作
- **mouseup** - 完成操作

### 4. 碰撞检测

```javascript
function hitTest(x, y, element) {
    // 根据元素类型判断点是否在元素内
    return 点在元素边界内;
}
```

## 扩展建议

基于这个最小核心，你可以添加：

### 1. 更多工具
- 箭头工具（带箭头的线条）
- 多边形工具
- 图片插入

### 2. 样式控制
- 颜色选择器
- 线条粗细调节
- 填充样式选择

### 3. 高级功能
- 撤销/重做
- 复制/粘贴
- 图层管理
- 导出功能

### 4. 协作功能
- WebSocket 连接
- 操作同步
- 冲突解决

## 与完整 Excalidraw 的对比

| 功能 | 最小实现 | 完整 Excalidraw |
|-----|---------|----------------|
| 代码行数 | ~300 | ~50,000+ |
| 元素类型 | 6种 | 10+种 |
| 渲染层 | 单层 | 多层优化 |
| 手绘效果 | 简单随机 | RoughJS |
| 状态管理 | 全局变量 | Jotai |
| 协作 | 无 | WebSocket + CRDT |
| 导出 | 无 | PNG/SVG/JSON |
| 性能优化 | 基础 | 高度优化 |

## 学习路径

1. **理解这个最小实现** - 运行并修改代码
2. **添加一个新工具** - 比如箭头工具
3. **实现撤销功能** - 添加历史栈
4. **优化渲染** - 实现脏矩形或分层
5. **研究真实源码** - 对比学习

这个最小实现展示了 Excalidraw 的核心原理，是深入学习的最佳起点。
