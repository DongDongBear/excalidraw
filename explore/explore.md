# Excalidraw 项目深度探索与二次开发指南

## 目录

1. [项目架构概览](#项目架构概览)
2. [核心包体系](#核心包体系)
3. [最小核心实现](#最小核心实现)
4. [开发环境搭建](#开发环境搭建)
5. [关键组件与API](#关键组件与api)
6. [二次开发策略](#二次开发策略)
7. [定制化开发建议](#定制化开发建议)

## 项目架构概览

### 1.1 Monorepo 结构

Excalidraw 采用 **Monorepo** 架构，使用 Yarn Workspaces 管理多个包：

```
excalidraw/
├── packages/               # 核心包集合
│   ├── excalidraw/        # 主要 React 组件库 (npm: @excalidraw/excalidraw)
│   ├── common/            # 公共工具、常量、类型定义
│   ├── element/           # 元素相关逻辑
│   ├── math/              # 数学计算工具
│   └── utils/             # 工具函数库
├── excalidraw-app/        # 完整的 Web 应用 (excalidraw.com)
├── examples/              # 使用示例
│   ├── with-nextjs/       # Next.js 集成示例
│   └── with-script-in-browser/  # 浏览器脚本示例
└── scripts/               # 构建和开发脚本
```

### 1.2 技术栈

- **核心框架**: React 19.0 + TypeScript 4.9
- **状态管理**: Jotai (原子化状态管理)
- **构建工具**: 
  - ESBuild (用于包构建)
  - Vite (用于应用开发)
- **绘图引擎**: 
  - RoughJS (手绘风格渲染)
  - Canvas API (底层绘制)
- **关键依赖**:
  - `perfect-freehand`: 平滑手绘线条
  - `fractional-indexing`: 协作时的元素排序
  - `pako`: 数据压缩

## 核心包体系

### 2.1 包依赖关系

```mermaid
graph TD
    A[@excalidraw/excalidraw] --> B[@excalidraw/common]
    A --> C[@excalidraw/element]
    A --> D[@excalidraw/math]
    C --> B
    C --> D
    E[@excalidraw/utils] --> B
```

### 2.2 各包职责

#### @excalidraw/common (基础层)
- **作用**: 提供基础常量、类型定义、工具函数
- **无依赖**: 不依赖其他内部包
- **关键内容**:
  ```typescript
  - 常量定义 (KEYS, CODES, CURSOR_TYPE, THEME等)
  - 基础工具函数 (throttle, debounce, isInputLike等)
  - 类型定义 (基础类型，不包含元素类型)
  ```

#### @excalidraw/math (数学层)
- **作用**: 提供2D几何计算
- **依赖**: @excalidraw/common
- **关键功能**:
  ```typescript
  - 向量运算 (vector, vectorScale, vectorDot等)
  - 点操作 (pointFrom, pointDistance, pointRotateRads等)
  - 几何计算 (角度、距离、碰撞检测等)
  ```

#### @excalidraw/element (元素层)
- **作用**: 定义和操作画布元素
- **依赖**: @excalidraw/common, @excalidraw/math
- **核心功能**:
  ```typescript
  - 元素类型定义 (ExcalidrawElement及其子类型)
  - 元素创建 (newElement, newTextElement, newArrowElement等)
  - 元素操作 (duplicateElements, deepCopyElement, bindOrUnbindLinearElements等)
  - 元素工具 (LinearElementEditor, refreshTextDimensions等)
  ```

#### @excalidraw/excalidraw (组件层)
- **作用**: React 组件和应用逻辑
- **依赖**: 所有其他核心包
- **主要组件**:
  ```typescript
  - <Excalidraw />: 主组件
  - <App />: 应用核心逻辑
  - UI组件: MainMenu, Sidebar, Actions, Footer等
  - 渲染器: staticScene, interactiveScene
  ```

## 最小核心实现

### 3.1 最简使用示例

```typescript
// 最小化实现只需要三步
import React from "react";
import { Excalidraw } from "@excalidraw/excalidraw";
import "@excalidraw/excalidraw/index.css";

function MinimalExcalidraw() {
  return (
    <div style={{ height: "100vh" }}>
      <Excalidraw />
    </div>
  );
}
```

### 3.2 核心功能模块

最小核心包含以下必要功能：

1. **画布管理**
   - Canvas 渲染上下文
   - 视口控制 (缩放、平移)
   - 场景坐标系统

2. **元素系统**
   - 基础图形 (矩形、圆形、菱形、线条、箭头)
   - 自由绘制
   - 文本元素

3. **交互系统**
   - 工具选择
   - 元素选择和变换
   - 撤销/重做

4. **渲染引擎**
   - RoughJS 集成
   - 静态场景渲染
   - 交互场景渲染

### 3.3 可选功能模块

这些功能可以根据需求选择性启用：

- **协作功能**: 实时多人协作
- **图片支持**: 导入和显示图片
- **库功能**: 元素库管理
- **导出功能**: PNG/SVG/JSON导出
- **嵌入元素**: iframe、嵌入式内容
- **AI功能**: 图表转代码、魔法框架

## 开发环境搭建

### 4.1 环境要求

```json
{
  "engines": {
    "node": "18.0.0 - 22.x.x"
  }
}
```

### 4.2 开发命令

```bash
# 安装依赖
yarn install

# 启动开发环境 (运行 excalidraw-app)
yarn start

# 构建核心包
yarn build:packages

# 运行测试
yarn test
yarn test:typecheck
yarn test:update  # 更新快照

# 代码格式化
yarn fix
```

### 4.3 开发流程

1. **包开发**: 在 `packages/*` 中开发核心功能
2. **应用开发**: 在 `excalidraw-app/` 中开发应用特性
3. **测试验证**: 使用 Vitest 进行单元测试
4. **类型检查**: 确保 TypeScript 类型正确

## 关键组件与API

### 5.1 主要导出组件

```typescript
// 核心组件
export { Excalidraw };           // 主画板组件
export { MainMenu };             // 主菜单
export { Footer };               // 页脚
export { Sidebar };              // 侧边栏
export { LiveCollaborationTrigger }; // 协作触发器
export { WelcomeScreen };        // 欢迎界面

// UI组件
export { Button };
export { DefaultSidebar };
export { Stats };
export { TTDDialog, TTDDialogTrigger };
```

### 5.2 核心API

```typescript
// 数据操作
export { serializeAsJSON, serializeLibraryAsJSON };
export { convertToExcalidrawElements };
export { reconcileElements };

// 元素操作
export { newElement, newTextElement, newArrowElement };
export { duplicateElements };
export { isLinearElement };

// 工具函数
export { getCommonBounds };
export { zoomToFitBounds };
export { getDataURL };
```

### 5.3 Props 接口

```typescript
interface ExcalidrawProps {
  // 基础配置
  initialData?: ImportedDataState;
  excalidrawAPI?: ExcalidrawAPIRefValue;
  
  // UI配置
  UIOptions?: UIOptions;
  theme?: Theme;
  langCode?: string;
  viewModeEnabled?: boolean;
  zenModeEnabled?: boolean;
  gridModeEnabled?: boolean;
  
  // 回调函数
  onChange?: (elements: ExcalidrawElement[], appState: AppState) => void;
  onPointerUpdate?: (payload: {pointer: Point, button: "down"|"up", pointersMap: Map}) => void;
  onPaste?: (data: ClipboardData, event: ClipboardEvent) => boolean;
  onLibraryChange?: (items: LibraryItems) => void;
  
  // 自定义渲染
  renderTopRightUI?: (isMobile: boolean, appState: AppState) => JSX.Element;
  renderCustomStats?: (elements: ExcalidrawElement[], appState: AppState) => JSX.Element;
  
  // 其他
  autoFocus?: boolean;
  detectScroll?: boolean;
  handleKeyboardGlobally?: boolean;
}
```

## 二次开发策略

### 6.1 开发方式选择

#### 方式一：直接使用 NPM 包（推荐用于快速集成）

**优点**：
- 快速集成，开箱即用
- 自动获得官方更新
- 无需维护构建流程

**缺点**：
- 定制化能力有限
- 依赖官方发布节奏

**适用场景**：
- 需要标准画板功能
- 定制需求较少
- 快速原型开发

```typescript
// 安装
npm install @excalidraw/excalidraw

// 使用
import { Excalidraw } from "@excalidraw/excalidraw";
```

#### 方式二：Fork 源码（推荐用于深度定制）

**优点**：
- 完全控制代码
- 可深度定制任何功能
- 可以贡献回上游

**缺点**：
- 需要维护分支
- 合并上游更新较复杂

**适用场景**：
- 需要修改核心功能
- 有特殊业务需求
- 长期维护的产品

### 6.2 定制化开发路径

#### 初级定制（使用 NPM 包）

1. **UI 定制**
   ```typescript
   <Excalidraw
     UIOptions={{
       canvasActions: {
         export: false,  // 隐藏导出
         saveAsScene: false,
       }
     }}
     renderTopRightUI={(isMobile, appState) => (
       <CustomToolbar />
     )}
   />
   ```

2. **主题定制**
   ```css
   /* 覆盖 CSS 变量 */
   :root {
     --color-primary: #your-color;
     --color-primary-darker: #your-darker-color;
   }
   ```

3. **事件监听**
   ```typescript
   const handleChange = (elements, appState) => {
     // 监听画布变化
     console.log("Elements changed:", elements);
   };
   
   <Excalidraw onChange={handleChange} />
   ```

#### 中级定制（修改部分源码）

1. **创建自定义包装器**
   ```typescript
   // CustomExcalidraw.tsx
   import { Excalidraw } from "@excalidraw/excalidraw";
   
   export function CustomExcalidraw(props) {
     // 添加自定义逻辑
     const customAPI = useCustomFeatures();
     
     return (
       <div className="custom-wrapper">
         <CustomToolbar />
         <Excalidraw {...props} />
         <CustomPanel />
       </div>
     );
   }
   ```

2. **扩展工具栏**
   ```typescript
   // 添加自定义工具
   const customTools = [
     {
       icon: CustomIcon,
       name: "customTool",
       action: (elements, appState) => {
         // 自定义工具逻辑
       }
     }
   ];
   ```

3. **自定义元素类型**
   ```typescript
   // 通过 convertToExcalidrawElements 转换自定义数据
   const customElements = convertToExcalidrawElements([
     {
       type: "rectangle",
       x: 100,
       y: 100,
       width: 200,
       height: 100,
       customData: { myField: "value" }
     }
   ]);
   ```

#### 高级定制（Fork 源码）

1. **修改核心渲染逻辑**
   - 位置：`packages/excalidraw/renderer/`
   - 可以修改 RoughJS 参数实现不同绘制风格
   
2. **添加新元素类型**
   - 修改 `packages/element/src/types.ts`
   - 实现对应的创建和渲染逻辑
   
3. **集成自定义后端**
   - 修改 `excalidraw-app/` 中的协作逻辑
   - 实现自定义存储和同步

### 6.3 最小化构建策略

如果你想要一个更轻量的版本，可以：

1. **移除不需要的功能**
   ```typescript
   // 在构建配置中排除模块
   {
     external: [
       "@excalidraw/mermaid-to-excalidraw",  // 移除 Mermaid 支持
       "@excalidraw/laser-pointer",          // 移除激光指针
     ]
   }
   ```

2. **创建精简版组件**
   ```typescript
   // MinimalExcalidraw.tsx
   import { App } from "./components/App";
   
   export function MinimalExcalidraw() {
     return (
       <App
         // 只保留核心功能
         UIOptions={{
           canvasActions: {
             export: false,
             saveAsScene: false,
             theme: false,
           }
         }}
       />
     );
   }
   ```

3. **优化打包体积**
   - 使用 tree-shaking 移除未使用代码
   - 懒加载非核心功能
   - 使用 dynamic import 分割代码

## 定制化开发建议

### 7.1 架构设计建议

1. **分层架构**
   ```
   你的应用
   ├── presentation/   # UI层
   ├── business/       # 业务逻辑
   ├── integration/    # Excalidraw集成层
   └── data/          # 数据层
   ```

2. **适配器模式**
   - 创建适配器层隔离 Excalidraw API
   - 便于未来升级或替换

3. **插件化设计**
   - 将自定义功能设计为插件
   - 保持核心功能独立

### 7.2 性能优化

1. **按需加载**
   ```typescript
   const Excalidraw = lazy(() => import("@excalidraw/excalidraw"));
   ```

2. **虚拟化大量元素**
   - 实现视口裁剪
   - 只渲染可见元素

3. **优化重渲染**
   - 使用 React.memo
   - 合理使用 useCallback/useMemo

### 7.3 开发技巧

1. **调试技巧**
   ```typescript
   // 启用开发模式日志
   if (import.meta.env.DEV) {
     window.EXCALIDRAW_DEBUG = true;
   }
   ```

2. **测试策略**
   - 单元测试：测试工具函数
   - 集成测试：测试组件交互
   - E2E测试：测试完整流程

3. **版本管理**
   - 锁定 Excalidraw 版本避免breaking changes
   - 定期评估和计划升级

### 7.4 常见定制场景

#### 场景1：企业协作白板

**需求**：
- 用户权限管理
- 实时协作
- 云端存储

**方案**：
```typescript
// 1. 集成WebSocket
const collaborationBackend = new WebSocketBackend();

// 2. 添加权限控制
const PermissionWrapper = ({ children }) => {
  const canEdit = usePermission();
  return (
    <Excalidraw
      viewModeEnabled={!canEdit}
      onChange={canEdit ? handleChange : undefined}
    />
  );
};

// 3. 云端同步
const CloudSync = {
  save: async (elements) => {
    await api.saveToCloud(elements);
  },
  load: async () => {
    return await api.loadFromCloud();
  }
};
```

#### 场景2：教育绘图工具

**需求**：
- 数学公式支持
- 预设图形库
- 作业提交

**方案**：
```typescript
// 1. 集成数学公式
import { MathJaxRenderer } from "./math";

// 2. 自定义图形库
const educationLibrary = [
  { type: "triangle", ... },
  { type: "coordinate", ... }
];

// 3. 作业功能
const HomeworkMode = {
  submit: (elements) => {
    // 提交逻辑
  },
  review: (homework) => {
    // 批改逻辑
  }
};
```

#### 场景3：流程图设计器

**需求**：
- 连接点吸附
- 自动布局
- 导出标准格式

**方案**：
```typescript
// 1. 增强连接功能
const FlowchartMode = {
  enableSnapping: true,
  connectionPoints: true,
  
  // 2. 自动布局
  autoLayout: (elements) => {
    return layoutAlgorithm(elements);
  },
  
  // 3. 格式转换
  exportToMermaid: (elements) => {
    return convertToMermaid(elements);
  }
};
```

### 7.5 注意事项

1. **License 合规**
   - Excalidraw 使用 MIT License
   - 可商用但需保留版权声明

2. **API 稳定性**
   - 主版本可能有 breaking changes
   - 建议固定版本并定期评估升级

3. **浏览器兼容性**
   - 需要现代浏览器支持
   - 不支持 IE11

4. **移动端适配**
   - 触摸事件已支持
   - 但UI可能需要调整

## 总结

Excalidraw 是一个架构优秀、功能完整的开源白板项目。通过本文档的深入分析，你应该能够：

1. **理解项目架构**：掌握 Monorepo 结构和包依赖关系
2. **快速上手开发**：知道如何搭建环境和进行开发
3. **灵活定制功能**：根据需求选择合适的定制方案
4. **高效二次开发**：基于最佳实践进行定制开发

### 推荐的开发路径

1. **第一步**：使用 NPM 包快速集成，熟悉基本 API
2. **第二步**：通过 Props 和 CSS 进行初级定制
3. **第三步**：根据需求决定是否需要 Fork 源码
4. **第四步**：逐步实现深度定制功能

### 核心要点

- **最小核心**：一个 `<Excalidraw />` 组件即可实现基本画板
- **渐进增强**：根据需求逐步添加功能
- **模块化设计**：各包职责清晰，便于理解和修改
- **生态完整**：从基础绘图到协作、导出一应俱全

希望这份探索文档能帮助你快速掌握 Excalidraw，并成功实现你的画板功能需求！