# 7.3 可扩展性核心：插件化架构设计

本节深入探讨 Excalidraw 的可扩展性设计，展示如何构建一个支持插件、主题、工具扩展的灵活架构。

## 插件化架构核心

### 1. 插件系统基础

```typescript
// 插件基础接口
interface ExcalidrawPlugin {
  id: string;
  name: string;
  version: string;

  // 插件生命周期
  onInstall?(api: ExcalidrawAPI): Promise<void>;
  onUninstall?(api: ExcalidrawAPI): Promise<void>;
  onActivate?(api: ExcalidrawAPI): Promise<void>;
  onDeactivate?(api: ExcalidrawAPI): Promise<void>;

  // 功能扩展点
  tools?: ToolExtension[];
  actions?: ActionExtension[];
  components?: ComponentExtension[];
  exporters?: ExportExtension[];
}

// 插件管理器
class PluginManager {
  private plugins = new Map<string, ExcalidrawPlugin>();
  private activePlugins = new Set<string>();
  private api: ExcalidrawAPI;

  constructor(api: ExcalidrawAPI) {
    this.api = api;
  }

  async installPlugin(plugin: ExcalidrawPlugin): Promise<void> {
    // 验证插件
    this.validatePlugin(plugin);

    // 检查依赖冲突
    await this.checkDependencies(plugin);

    // 执行安装钩子
    if (plugin.onInstall) {
      await plugin.onInstall(this.api);
    }

    // 注册插件
    this.plugins.set(plugin.id, plugin);

    // 注册插件扩展
    this.registerExtensions(plugin);

    console.log(`Plugin ${plugin.name} installed successfully`);
  }

  async activatePlugin(pluginId: string): Promise<void> {
    const plugin = this.plugins.get(pluginId);
    if (!plugin) {
      throw new Error(`Plugin ${pluginId} not found`);
    }

    if (this.activePlugins.has(pluginId)) {
      return; // Already active
    }

    // 执行激活钩子
    if (plugin.onActivate) {
      await plugin.onActivate(this.api);
    }

    this.activePlugins.add(pluginId);
    this.api.notifyPluginActivated(pluginId);
  }

  private registerExtensions(plugin: ExcalidrawPlugin): void {
    // 注册工具扩展
    if (plugin.tools) {
      plugin.tools.forEach(tool => {
        this.api.registerTool(tool);
      });
    }

    // 注册动作扩展
    if (plugin.actions) {
      plugin.actions.forEach(action => {
        this.api.registerAction(action);
      });
    }

    // 注册组件扩展
    if (plugin.components) {
      plugin.components.forEach(component => {
        this.api.registerComponent(component);
      });
    }

    // 注册导出器扩展
    if (plugin.exporters) {
      plugin.exporters.forEach(exporter => {
        this.api.registerExporter(exporter);
      });
    }
  }
}
```

### 2. API 设计核心

```typescript
// 插件 API 核心
class ExcalidrawAPI {
  private eventBus: EventBus;
  private stateManager: StateManager;
  private extensionRegistry: ExtensionRegistry;

  constructor(
    eventBus: EventBus,
    stateManager: StateManager,
    extensionRegistry: ExtensionRegistry
  ) {
    this.eventBus = eventBus;
    this.stateManager = stateManager;
    this.extensionRegistry = extensionRegistry;
  }

  // 状态访问 API
  getAppState(): Readonly<AppState> {
    return this.stateManager.getAppState();
  }

  getElements(): readonly ExcalidrawElement[] {
    return this.stateManager.getElements();
  }

  getSelectedElements(): readonly ExcalidrawElement[] {
    const state = this.getAppState();
    const elements = this.getElements();

    return elements.filter(element =>
      state.selectedElementIds[element.id]
    );
  }

  // 状态修改 API（受限）
  updateElements(elements: ExcalidrawElement[]): void {
    // 安全性检查
    this.validateElements(elements);

    this.stateManager.updateElements(elements);
  }

  updateAppState(appState: Partial<AppState>): void {
    // 安全性检查：只允许特定属性
    const allowedProperties = [
      'currentItemStrokeColor',
      'currentItemBackgroundColor',
      'currentItemFillStyle',
      'currentItemStrokeWidth'
    ];

    const filteredState = this.filterProperties(appState, allowedProperties);
    this.stateManager.updateAppState(filteredState);
  }

  // 事件系统 API
  on<T = any>(event: string, callback: (data: T) => void): () => void {
    return this.eventBus.on(event, callback);
  }

  emit<T = any>(event: string, data: T): void {
    // 插件事件必须有前缀以避免冲突
    const pluginEvent = this.addPluginPrefix(event);
    this.eventBus.emit(pluginEvent, data);
  }

  // 工具注册 API
  registerTool(tool: ToolExtension): void {
    this.validateTool(tool);
    this.extensionRegistry.registerTool(tool);
  }

  // 动作注册 API
  registerAction(action: ActionExtension): void {
    this.validateAction(action);
    this.extensionRegistry.registerAction(action);
  }

  // UI 扩展 API
  registerComponent(component: ComponentExtension): void {
    this.validateComponent(component);
    this.extensionRegistry.registerComponent(component);
  }

  // 安全性验证
  private validateElements(elements: ExcalidrawElement[]): void {
    elements.forEach(element => {
      if (!element.id || typeof element.id !== 'string') {
        throw new Error('Invalid element ID');
      }

      if (!element.type || typeof element.type !== 'string') {
        throw new Error('Invalid element type');
      }

      // 检查是否尝试修改不允许的属性
      const forbiddenProperties = ['seed', 'versionNonce'];
      forbiddenProperties.forEach(prop => {
        if (prop in element) {
          throw new Error(`Modifying ${prop} is not allowed`);
        }
      });
    });
  }
}
```

## 工具扩展系统

### 1. 自定义工具接口

```typescript
interface ToolExtension {
  id: string;
  name: string;
  icon: string | React.ComponentType;
  keyBinding?: string;

  // 工具行为定义
  onPointerDown?(
    event: PointerEvent,
    appState: AppState,
    api: ExcalidrawAPI
  ): ToolResponse;

  onPointerMove?(
    event: PointerEvent,
    appState: AppState,
    api: ExcalidrawAPI
  ): ToolResponse;

  onPointerUp?(
    event: PointerEvent,
    appState: AppState,
    api: ExcalidrawAPI
  ): ToolResponse;

  // 工具选项配置
  getToolOptions?(): ToolOption[];

  // 工具栏集成
  renderInToolbar?(api: ExcalidrawAPI): React.ReactElement;
}

// 工具响应类型
type ToolResponse = {
  elements?: ExcalidrawElement[];
  appState?: Partial<AppState>;
  commitToHistory?: boolean;
};

// 自定义工具示例：圆角矩形工具
class RoundedRectangleTool implements ToolExtension {
  id = 'rounded-rectangle';
  name = 'Rounded Rectangle';
  icon = '▢';
  keyBinding = 'R';

  private isDrawing = false;
  private startPoint: Point | null = null;
  private currentElement: ExcalidrawElement | null = null;

  onPointerDown(
    event: PointerEvent,
    appState: AppState,
    api: ExcalidrawAPI
  ): ToolResponse {
    this.isDrawing = true;
    this.startPoint = { x: event.clientX, y: event.clientY };

    this.currentElement = {
      id: generateId(),
      type: 'rectangle', // 基于现有类型扩展
      x: this.startPoint.x,
      y: this.startPoint.y,
      width: 0,
      height: 0,
      strokeColor: appState.currentItemStrokeColor,
      backgroundColor: appState.currentItemBackgroundColor,
      fillStyle: appState.currentItemFillStyle,
      strokeWidth: appState.currentItemStrokeWidth,
      roughness: appState.currentItemRoughness,
      opacity: appState.currentItemOpacity,
      // 自定义属性：圆角半径
      customData: {
        cornerRadius: 10
      }
    };

    return {
      appState: {
        activeTool: { type: this.id },
        draggingElement: this.currentElement
      }
    };
  }

  onPointerMove(
    event: PointerEvent,
    appState: AppState,
    api: ExcalidrawAPI
  ): ToolResponse {
    if (!this.isDrawing || !this.startPoint || !this.currentElement) {
      return {};
    }

    const updatedElement = {
      ...this.currentElement,
      width: event.clientX - this.startPoint.x,
      height: event.clientY - this.startPoint.y
    };

    return {
      appState: {
        draggingElement: updatedElement
      }
    };
  }

  onPointerUp(
    event: PointerEvent,
    appState: AppState,
    api: ExcalidrawAPI
  ): ToolResponse {
    if (!this.currentElement) {
      return {};
    }

    const finalElement = { ...this.currentElement };

    // 重置状态
    this.isDrawing = false;
    this.startPoint = null;
    this.currentElement = null;

    return {
      elements: [...api.getElements(), finalElement],
      appState: {
        draggingElement: null,
        activeTool: { type: 'selection' }
      },
      commitToHistory: true
    };
  }

  getToolOptions(): ToolOption[] {
    return [
      {
        type: 'slider',
        label: 'Corner Radius',
        property: 'cornerRadius',
        min: 0,
        max: 50,
        defaultValue: 10
      }
    ];
  }
}
```

### 2. 动作扩展系统

```typescript
interface ActionExtension {
  id: string;
  name: string;
  icon?: string | React.ComponentType;
  keyBinding?: string;
  contextMenuLabel?: string;

  // 动作执行
  perform(
    elements: readonly ExcalidrawElement[],
    appState: Readonly<AppState>,
    api: ExcalidrawAPI
  ): ActionResult;

  // 动作可用性检查
  predicate?(
    elements: readonly ExcalidrawElement[],
    appState: Readonly<AppState>,
    api: ExcalidrawAPI
  ): boolean;
}

type ActionResult = {
  elements?: ExcalidrawElement[];
  appState?: Partial<AppState>;
  commitToHistory?: boolean;
};

// 自定义动作示例：对齐工具
class AlignElementsAction implements ActionExtension {
  id = 'align-elements';
  name = 'Align Elements';
  keyBinding = 'Ctrl+L';

  perform(
    elements: readonly ExcalidrawElement[],
    appState: Readonly<AppState>,
    api: ExcalidrawAPI
  ): ActionResult {
    const selectedElements = api.getSelectedElements();

    if (selectedElements.length < 2) {
      return {}; // 需要至少两个元素
    }

    // 计算对齐基准
    const bounds = this.calculateBounds(selectedElements);

    // 执行左对齐
    const alignedElements = selectedElements.map(element => ({
      ...element,
      x: bounds.minX
    }));

    // 更新元素数组
    const newElements = elements.map(element => {
      const aligned = alignedElements.find(ae => ae.id === element.id);
      return aligned || element;
    });

    return {
      elements: newElements as ExcalidrawElement[],
      commitToHistory: true
    };
  }

  predicate(
    elements: readonly ExcalidrawElement[],
    appState: Readonly<AppState>,
    api: ExcalidrawAPI
  ): boolean {
    return api.getSelectedElements().length >= 2;
  }

  private calculateBounds(elements: readonly ExcalidrawElement[]) {
    let minX = Infinity, minY = Infinity;
    let maxX = -Infinity, maxY = -Infinity;

    elements.forEach(element => {
      minX = Math.min(minX, element.x);
      minY = Math.min(minY, element.y);
      maxX = Math.max(maxX, element.x + element.width);
      maxY = Math.max(maxY, element.y + element.height);
    });

    return { minX, minY, maxX, maxY };
  }
}
```

## 主题系统

### 1. 主题架构设计

```typescript
interface ExcalidrawTheme {
  id: string;
  name: string;
  author?: string;

  // 颜色系统
  colors: {
    // 应用颜色
    background: string;
    surface: string;
    primary: string;
    secondary: string;
    accent: string;

    // 文本颜色
    text: string;
    textSecondary: string;
    textDisabled: string;

    // 状态颜色
    success: string;
    warning: string;
    error: string;

    // 工具栏颜色
    toolbar: string;
    toolbarText: string;
    toolbarHover: string;

    // 画布颜色
    canvasBackground: string;
    gridColor: string;

    // 元素默认颜色
    elementStroke: string;
    elementBackground: string;
  };

  // 样式配置
  styles: {
    borderRadius: number;
    shadows: boolean;
    animations: boolean;
    toolbar: {
      height: number;
      padding: number;
    };
  };

  // CSS 变量定义
  cssVariables: Record<string, string>;
}

class ThemeManager {
  private themes = new Map<string, ExcalidrawTheme>();
  private currentTheme: ExcalidrawTheme;
  private styleElement: HTMLStyleElement;

  constructor() {
    this.styleElement = document.createElement('style');
    document.head.appendChild(this.styleElement);

    // 注册默认主题
    this.registerDefaultThemes();
  }

  registerTheme(theme: ExcalidrawTheme): void {
    this.themes.set(theme.id, theme);
  }

  applyTheme(themeId: string): void {
    const theme = this.themes.get(themeId);
    if (!theme) {
      throw new Error(`Theme ${themeId} not found`);
    }

    this.currentTheme = theme;

    // 生成并应用 CSS
    const css = this.generateThemeCSS(theme);
    this.styleElement.textContent = css;

    // 更新 CSS 自定义属性
    this.applyCustomProperties(theme);

    // 触发主题变更事件
    this.emitThemeChanged(theme);
  }

  private generateThemeCSS(theme: ExcalidrawTheme): string {
    return `
      :root {
        --color-background: ${theme.colors.background};
        --color-surface: ${theme.colors.surface};
        --color-primary: ${theme.colors.primary};
        --color-secondary: ${theme.colors.secondary};
        --color-accent: ${theme.colors.accent};

        --color-text: ${theme.colors.text};
        --color-text-secondary: ${theme.colors.textSecondary};
        --color-text-disabled: ${theme.colors.textDisabled};

        --color-success: ${theme.colors.success};
        --color-warning: ${theme.colors.warning};
        --color-error: ${theme.colors.error};

        --color-toolbar: ${theme.colors.toolbar};
        --color-toolbar-text: ${theme.colors.toolbarText};
        --color-toolbar-hover: ${theme.colors.toolbarHover};

        --color-canvas-background: ${theme.colors.canvasBackground};
        --color-grid: ${theme.colors.gridColor};

        --border-radius: ${theme.styles.borderRadius}px;
        --toolbar-height: ${theme.styles.toolbar.height}px;
        --toolbar-padding: ${theme.styles.toolbar.padding}px;

        ${Object.entries(theme.cssVariables)
          .map(([key, value]) => `--${key}: ${value}`)
          .join(';\n        ')};
      }

      .excalidraw {
        background-color: var(--color-background);
        color: var(--color-text);
      }

      .excalidraw__canvas {
        background-color: var(--color-canvas-background);
      }

      .excalidraw__toolbar {
        background-color: var(--color-toolbar);
        color: var(--color-toolbar-text);
        height: var(--toolbar-height);
        padding: var(--toolbar-padding);
        border-radius: var(--border-radius);
      }

      .excalidraw__toolbar button {
        color: var(--color-toolbar-text);
        border-radius: var(--border-radius);
      }

      .excalidraw__toolbar button:hover {
        background-color: var(--color-toolbar-hover);
      }

      ${theme.styles.animations ? '' : '* { animation: none !important; }'}
    `;
  }

  private registerDefaultThemes(): void {
    // 浅色主题
    this.registerTheme({
      id: 'light',
      name: 'Light',
      colors: {
        background: '#ffffff',
        surface: '#f8f9fa',
        primary: '#1971c2',
        secondary: '#495057',
        accent: '#e64980',

        text: '#212529',
        textSecondary: '#6c757d',
        textDisabled: '#adb5bd',

        success: '#51cf66',
        warning: '#ffd43b',
        error: '#ff6b6b',

        toolbar: '#ffffff',
        toolbarText: '#495057',
        toolbarHover: '#e9ecef',

        canvasBackground: '#ffffff',
        gridColor: '#e9ecef',

        elementStroke: '#000000',
        elementBackground: 'transparent'
      },
      styles: {
        borderRadius: 4,
        shadows: true,
        animations: true,
        toolbar: {
          height: 40,
          padding: 8
        }
      },
      cssVariables: {}
    });

    // 深色主题
    this.registerTheme({
      id: 'dark',
      name: 'Dark',
      colors: {
        background: '#1a1a1a',
        surface: '#2c2c2c',
        primary: '#339af0',
        secondary: '#adb5bd',
        accent: '#f783ac',

        text: '#ffffff',
        textSecondary: '#ced4da',
        textDisabled: '#6c757d',

        success: '#51cf66',
        warning: '#ffd43b',
        error: '#ff6b6b',

        toolbar: '#2c2c2c',
        toolbarText: '#ffffff',
        toolbarHover: '#404040',

        canvasBackground: '#1a1a1a',
        gridColor: '#404040',

        elementStroke: '#ffffff',
        elementBackground: 'transparent'
      },
      styles: {
        borderRadius: 4,
        shadows: true,
        animations: true,
        toolbar: {
          height: 40,
          padding: 8
        }
      },
      cssVariables: {}
    });
  }
}
```

## 组件扩展系统

### 1. UI 组件扩展

```typescript
interface ComponentExtension {
  id: string;
  name: string;
  position: 'toolbar' | 'sidebar' | 'modal' | 'context-menu';

  // 组件渲染
  render(props: ComponentProps): React.ReactElement;

  // 组件可见性控制
  shouldRender?(
    appState: Readonly<AppState>,
    api: ExcalidrawAPI
  ): boolean;
}

interface ComponentProps {
  appState: Readonly<AppState>;
  elements: readonly ExcalidrawElement[];
  api: ExcalidrawAPI;
}

// 自定义组件示例：元素统计面板
class ElementStatsPanel implements ComponentExtension {
  id = 'element-stats';
  name = 'Element Statistics';
  position = 'sidebar' as const;

  render(props: ComponentProps): React.ReactElement {
    const { elements } = props;

    const stats = this.calculateStats(elements);

    return React.createElement('div', {
      className: 'element-stats-panel',
      style: {
        padding: '16px',
        backgroundColor: 'var(--color-surface)',
        borderRadius: 'var(--border-radius)',
        border: '1px solid var(--color-secondary)'
      }
    }, [
      React.createElement('h3', { key: 'title' }, 'Statistics'),
      React.createElement('div', { key: 'stats' }, [
        React.createElement('p', { key: 'total' }, `Total Elements: ${stats.total}`),
        React.createElement('p', { key: 'shapes' }, `Shapes: ${stats.shapes}`),
        React.createElement('p', { key: 'text' }, `Text Elements: ${stats.text}`),
        React.createElement('p', { key: 'arrows' }, `Arrows: ${stats.arrows}`)
      ])
    ]);
  }

  shouldRender(appState: Readonly<AppState>): boolean {
    return appState.openSidebar?.name === 'stats';
  }

  private calculateStats(elements: readonly ExcalidrawElement[]) {
    return {
      total: elements.length,
      shapes: elements.filter(e => ['rectangle', 'ellipse', 'diamond'].includes(e.type)).length,
      text: elements.filter(e => e.type === 'text').length,
      arrows: elements.filter(e => e.type === 'arrow').length
    };
  }
}
```

这个可扩展性核心为 Excalidraw 提供了强大的扩展能力，允许开发者创建自定义工具、动作、主题和 UI 组件，同时保持核心系统的稳定性和安全性。通过良好的 API 设计和插件管理，Excalidraw 可以支持丰富的生态系统扩展。