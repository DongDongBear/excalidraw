# Chapter 6.2: 插件系统与可扩展架构

## 概述

插件系统是现代应用程序的重要特性，它允许第三方开发者扩展应用功能，而无需修改核心代码。本章将探讨如何设计和实现一个强大的插件架构，以Excalidraw的扩展需求为例，展示插件系统的设计原理和最佳实践。

## 插件架构设计

### 整体架构

```
插件系统架构层次
├── 核心层 (Core Layer)
│   ├── PluginManager      # 插件管理器
│   ├── EventSystem        # 事件系统
│   ├── HookSystem         # 钩子系统
│   └── SecurityManager    # 安全管理
├── API层 (API Layer)
│   ├── DrawingAPI         # 绘图API
│   ├── ElementAPI         # 元素操作API
│   ├── StateAPI           # 状态管理API
│   └── UIAPI              # 界面API
├── 插件层 (Plugin Layer)
│   ├── BuiltinPlugins     # 内置插件
│   ├── ThirdPartyPlugins  # 第三方插件
│   └── UserScripts        # 用户脚本
└── 沙箱层 (Sandbox Layer)
    ├── IsolatedContext    # 隔离上下文
    ├── PermissionSystem   # 权限系统
    └── ResourceLimiter    # 资源限制
```

### 插件接口定义

```typescript
// 插件基础接口
export interface ExcalidrawPlugin {
  // 插件元信息
  readonly id: string;
  readonly name: string;
  readonly version: string;
  readonly description?: string;
  readonly author?: string;
  readonly homepage?: string;
  readonly repository?: string;

  // 兼容性信息
  readonly minVersion?: string;  // 最小兼容版本
  readonly maxVersion?: string;  // 最大兼容版本
  readonly dependencies?: string[];  // 依赖的插件ID

  // 权限声明
  readonly permissions?: PluginPermission[];

  // 插件配置
  readonly config?: PluginConfig;

  // 生命周期方法
  onLoad?(api: PluginAPI): Promise<void> | void;
  onUnload?(): Promise<void> | void;
  onEnable?(): Promise<void> | void;
  onDisable?(): Promise<void> | void;

  // 事件处理
  onElementsChange?(elements: readonly ExcalidrawElement[]): void;
  onAppStateChange?(appState: AppState): void;
  onSelectionChange?(selectedElements: ExcalidrawElement[]): void;
}

// 插件权限枚举
export enum PluginPermission {
  READ_ELEMENTS = 'read:elements',
  WRITE_ELEMENTS = 'write:elements',
  READ_APPSTATE = 'read:appstate',
  WRITE_APPSTATE = 'write:appstate',
  CREATE_UI = 'create:ui',
  ACCESS_FILESYSTEM = 'access:filesystem',
  NETWORK_REQUEST = 'network:request',
  REGISTER_ACTIONS = 'register:actions',
  REGISTER_TOOLS = 'register:tools',
}

// 插件配置
export interface PluginConfig {
  [key: string]: {
    type: 'string' | 'number' | 'boolean' | 'select' | 'color';
    default: any;
    label?: string;
    description?: string;
    options?: string[];  // for select type
    min?: number;        // for number type
    max?: number;        // for number type
  };
}

// 插件API接口
export interface PluginAPI {
  // 元素操作
  elements: ElementAPI;

  // 状态管理
  appState: AppStateAPI;

  // 绘图操作
  canvas: CanvasAPI;

  // UI操作
  ui: UIAPI;

  // 动作系统
  actions: ActionAPI;

  // 工具系统
  tools: ToolAPI;

  // 事件系统
  events: EventAPI;

  // 配置管理
  config: ConfigAPI;

  // 存储系统
  storage: StorageAPI;

  // 网络请求
  http?: HttpAPI;  // 需要网络权限

  // 文件系统
  fs?: FileSystemAPI;  // 需要文件系统权限
}
```

## 插件管理器实现

### PluginManager核心类

```typescript
// packages/excalidraw/plugins/PluginManager.ts
export class PluginManager {
  private plugins: Map<string, LoadedPlugin> = new Map();
  private pluginConfigs: Map<string, any> = new Map();
  private eventSystem: EventSystem;
  private securityManager: SecurityManager;
  private hookSystem: HookSystem;

  // 插件状态
  private enabledPlugins: Set<string> = new Set();
  private loadingPlugins: Set<string> = new Set();

  // API实例
  private pluginAPIs: Map<string, PluginAPI> = new Map();

  constructor() {
    this.eventSystem = new EventSystem();
    this.securityManager = new SecurityManager();
    this.hookSystem = new HookSystem();

    this.setupInternalEvents();
    this.loadPluginConfigs();
  }

  // 注册插件
  async registerPlugin(plugin: ExcalidrawPlugin): Promise<boolean> {
    try {
      // 验证插件
      this.validatePlugin(plugin);

      // 检查依赖
      const missingDeps = this.checkDependencies(plugin);
      if (missingDeps.length > 0) {
        throw new Error(`Missing dependencies: ${missingDeps.join(', ')}`);
      }

      // 检查权限
      this.securityManager.validatePermissions(plugin.permissions || []);

      // 创建沙箱环境
      const sandbox = this.createPluginSandbox(plugin);

      // 创建插件API
      const api = this.createPluginAPI(plugin, sandbox);

      // 包装插件
      const loadedPlugin: LoadedPlugin = {
        plugin,
        sandbox,
        api,
        state: PluginState.REGISTERED,
        loadedAt: Date.now(),
        config: this.getPluginConfig(plugin.id),
      };

      this.plugins.set(plugin.id, loadedPlugin);
      this.pluginAPIs.set(plugin.id, api);

      console.log(`Plugin registered: ${plugin.name} (${plugin.id})`);

      // 触发事件
      this.eventSystem.emit('plugin:registered', { plugin });

      return true;
    } catch (error) {
      console.error(`Failed to register plugin ${plugin.id}:`, error);
      return false;
    }
  }

  // 加载插件
  async loadPlugin(pluginId: string): Promise<boolean> {
    const loadedPlugin = this.plugins.get(pluginId);
    if (!loadedPlugin) {
      console.error(`Plugin ${pluginId} not found`);
      return false;
    }

    if (loadedPlugin.state === PluginState.LOADED) {
      return true;
    }

    this.loadingPlugins.add(pluginId);

    try {
      // 执行加载钩子
      if (loadedPlugin.plugin.onLoad) {
        await this.executeSafely(
          () => loadedPlugin.plugin.onLoad!(loadedPlugin.api),
          `${pluginId}.onLoad`
        );
      }

      loadedPlugin.state = PluginState.LOADED;
      this.loadingPlugins.delete(pluginId);

      console.log(`Plugin loaded: ${loadedPlugin.plugin.name}`);

      // 触发事件
      this.eventSystem.emit('plugin:loaded', { plugin: loadedPlugin.plugin });

      // 如果插件应该启用，则启用它
      if (this.shouldAutoEnable(pluginId)) {
        await this.enablePlugin(pluginId);
      }

      return true;
    } catch (error) {
      console.error(`Failed to load plugin ${pluginId}:`, error);
      loadedPlugin.state = PluginState.ERROR;
      loadedPlugin.error = error instanceof Error ? error.message : String(error);
      this.loadingPlugins.delete(pluginId);
      return false;
    }
  }

  // 启用插件
  async enablePlugin(pluginId: string): Promise<boolean> {
    const loadedPlugin = this.plugins.get(pluginId);
    if (!loadedPlugin || loadedPlugin.state !== PluginState.LOADED) {
      return false;
    }

    if (this.enabledPlugins.has(pluginId)) {
      return true;
    }

    try {
      // 执行启用钩子
      if (loadedPlugin.plugin.onEnable) {
        await this.executeSafely(
          () => loadedPlugin.plugin.onEnable!(),
          `${pluginId}.onEnable`
        );
      }

      this.enabledPlugins.add(pluginId);
      loadedPlugin.state = PluginState.ENABLED;

      console.log(`Plugin enabled: ${loadedPlugin.plugin.name}`);

      // 注册插件提供的功能
      await this.registerPluginFeatures(loadedPlugin);

      // 触发事件
      this.eventSystem.emit('plugin:enabled', { plugin: loadedPlugin.plugin });

      return true;
    } catch (error) {
      console.error(`Failed to enable plugin ${pluginId}:`, error);
      return false;
    }
  }

  // 禁用插件
  async disablePlugin(pluginId: string): Promise<boolean> {
    const loadedPlugin = this.plugins.get(pluginId);
    if (!loadedPlugin || !this.enabledPlugins.has(pluginId)) {
      return false;
    }

    try {
      // 注销插件功能
      await this.unregisterPluginFeatures(loadedPlugin);

      // 执行禁用钩子
      if (loadedPlugin.plugin.onDisable) {
        await this.executeSafely(
          () => loadedPlugin.plugin.onDisable!(),
          `${pluginId}.onDisable`
        );
      }

      this.enabledPlugins.delete(pluginId);
      loadedPlugin.state = PluginState.LOADED;

      console.log(`Plugin disabled: ${loadedPlugin.plugin.name}`);

      // 触发事件
      this.eventSystem.emit('plugin:disabled', { plugin: loadedPlugin.plugin });

      return true;
    } catch (error) {
      console.error(`Failed to disable plugin ${pluginId}:`, error);
      return false;
    }
  }

  // 卸载插件
  async unloadPlugin(pluginId: string): Promise<boolean> {
    const loadedPlugin = this.plugins.get(pluginId);
    if (!loadedPlugin) {
      return false;
    }

    // 先禁用插件
    if (this.enabledPlugins.has(pluginId)) {
      await this.disablePlugin(pluginId);
    }

    try {
      // 执行卸载钩子
      if (loadedPlugin.plugin.onUnload) {
        await this.executeSafely(
          () => loadedPlugin.plugin.onUnload!(),
          `${pluginId}.onUnload`
        );
      }

      // 清理资源
      this.pluginAPIs.delete(pluginId);
      this.plugins.delete(pluginId);

      console.log(`Plugin unloaded: ${loadedPlugin.plugin.name}`);

      // 触发事件
      this.eventSystem.emit('plugin:unloaded', { plugin: loadedPlugin.plugin });

      return true;
    } catch (error) {
      console.error(`Failed to unload plugin ${pluginId}:`, error);
      return false;
    }
  }

  // 创建插件沙箱
  private createPluginSandbox(plugin: ExcalidrawPlugin): PluginSandbox {
    const sandbox = new PluginSandbox(plugin.id, plugin.permissions || []);

    // 配置沙箱限制
    sandbox.setTimeoutLimit(30000);  // 30秒超时
    sandbox.setMemoryLimit(100 * 1024 * 1024);  // 100MB内存限制
    sandbox.setCPULimit(1000);  // 1秒CPU时间限制

    return sandbox;
  }

  // 创建插件API
  private createPluginAPI(plugin: ExcalidrawPlugin, sandbox: PluginSandbox): PluginAPI {
    const permissions = new Set(plugin.permissions || []);

    return {
      elements: new ElementAPI(permissions, sandbox),
      appState: new AppStateAPI(permissions, sandbox),
      canvas: new CanvasAPI(permissions, sandbox),
      ui: new UIAPI(permissions, sandbox),
      actions: new ActionAPI(permissions, sandbox),
      tools: new ToolAPI(permissions, sandbox),
      events: this.eventSystem,
      config: new ConfigAPI(plugin.id, this.pluginConfigs),
      storage: new StorageAPI(plugin.id, permissions),
      http: permissions.has(PluginPermission.NETWORK_REQUEST)
        ? new HttpAPI(sandbox) : undefined,
      fs: permissions.has(PluginPermission.ACCESS_FILESYSTEM)
        ? new FileSystemAPI(sandbox) : undefined,
    };
  }

  // 验证插件
  private validatePlugin(plugin: ExcalidrawPlugin): void {
    if (!plugin.id || typeof plugin.id !== 'string') {
      throw new Error('Plugin ID is required and must be a string');
    }

    if (!plugin.name || typeof plugin.name !== 'string') {
      throw new Error('Plugin name is required and must be a string');
    }

    if (!plugin.version || typeof plugin.version !== 'string') {
      throw new Error('Plugin version is required and must be a string');
    }

    if (this.plugins.has(plugin.id)) {
      throw new Error(`Plugin with ID ${plugin.id} already exists`);
    }

    // 验证版本格式
    if (!/^\d+\.\d+\.\d+/.test(plugin.version)) {
      throw new Error('Plugin version must be in semver format (x.y.z)');
    }
  }

  // 检查依赖
  private checkDependencies(plugin: ExcalidrawPlugin): string[] {
    if (!plugin.dependencies) {
      return [];
    }

    const missing: string[] = [];

    for (const depId of plugin.dependencies) {
      if (!this.plugins.has(depId)) {
        missing.push(depId);
      }
    }

    return missing;
  }

  // 注册插件功能
  private async registerPluginFeatures(loadedPlugin: LoadedPlugin): Promise<void> {
    // 注册插件提供的Actions、Tools等
    // 这里需要与ActionManager、ToolManager等系统集成
  }

  // 注销插件功能
  private async unregisterPluginFeatures(loadedPlugin: LoadedPlugin): Promise<void> {
    // 清理插件注册的功能
  }

  // 安全执行插件代码
  private async executeSafely<T>(
    fn: () => T | Promise<T>,
    context: string
  ): Promise<T | null> {
    try {
      const result = await Promise.resolve(fn());
      return result;
    } catch (error) {
      console.error(`Error in ${context}:`, error);
      this.eventSystem.emit('plugin:error', { context, error });
      return null;
    }
  }

  // 其他辅助方法
  private shouldAutoEnable(pluginId: string): boolean {
    const config = this.getPluginConfig(pluginId);
    return config?.autoEnable !== false;
  }

  private getPluginConfig(pluginId: string): any {
    return this.pluginConfigs.get(pluginId) || {};
  }

  private setupInternalEvents(): void {
    // 监听应用事件并转发给插件
    this.eventSystem.on('elements:changed', (elements) => {
      for (const [pluginId, loadedPlugin] of this.plugins) {
        if (this.enabledPlugins.has(pluginId) && loadedPlugin.plugin.onElementsChange) {
          this.executeSafely(
            () => loadedPlugin.plugin.onElementsChange!(elements),
            `${pluginId}.onElementsChange`
          );
        }
      }
    });
  }

  private loadPluginConfigs(): void {
    // 从localStorage加载插件配置
    try {
      const configs = localStorage.getItem('excalidraw-plugin-configs');
      if (configs) {
        const parsed = JSON.parse(configs);
        this.pluginConfigs = new Map(Object.entries(parsed));
      }
    } catch (error) {
      console.error('Failed to load plugin configs:', error);
    }
  }

  // 公共API
  getPlugin(pluginId: string): ExcalidrawPlugin | null {
    return this.plugins.get(pluginId)?.plugin || null;
  }

  getLoadedPlugins(): ExcalidrawPlugin[] {
    return Array.from(this.plugins.values()).map(p => p.plugin);
  }

  getEnabledPlugins(): ExcalidrawPlugin[] {
    return Array.from(this.plugins.values())
      .filter(p => this.enabledPlugins.has(p.plugin.id))
      .map(p => p.plugin);
  }

  isPluginEnabled(pluginId: string): boolean {
    return this.enabledPlugins.has(pluginId);
  }
}

// 插件状态枚举
enum PluginState {
  REGISTERED = 'registered',
  LOADED = 'loaded',
  ENABLED = 'enabled',
  DISABLED = 'disabled',
  ERROR = 'error',
}

// 加载的插件接口
interface LoadedPlugin {
  plugin: ExcalidrawPlugin;
  sandbox: PluginSandbox;
  api: PluginAPI;
  state: PluginState;
  loadedAt: number;
  config: any;
  error?: string;
}
```

### 插件沙箱实现

```typescript
// packages/excalidraw/plugins/PluginSandbox.ts
export class PluginSandbox {
  private pluginId: string;
  private permissions: Set<PluginPermission>;
  private context: any;

  // 资源限制
  private timeoutLimit = 30000;  // 30秒
  private memoryLimit = 100 * 1024 * 1024;  // 100MB
  private cpuLimit = 1000;  // 1秒

  // 资源监控
  private startTime = 0;
  private initialMemory = 0;

  constructor(pluginId: string, permissions: PluginPermission[]) {
    this.pluginId = pluginId;
    this.permissions = new Set(permissions);
    this.createSandboxContext();
  }

  // 创建沙箱上下文
  private createSandboxContext(): void {
    this.context = {
      // 安全的全局对象
      console: this.createSecureConsole(),
      setTimeout: this.createSecureSetTimeout(),
      setInterval: this.createSecureSetInterval(),
      clearTimeout: clearTimeout,
      clearInterval: clearInterval,

      // 限制的API
      fetch: this.permissions.has(PluginPermission.NETWORK_REQUEST)
        ? this.createSecureFetch() : undefined,

      // 禁用危险的API
      eval: undefined,
      Function: undefined,
      document: undefined,  // 插件不能直接访问DOM
      window: undefined,
      global: undefined,
      process: undefined,
    };
  }

  // 创建安全的console
  private createSecureConsole(): Console {
    const originalConsole = console;

    return {
      ...originalConsole,
      log: (...args) => originalConsole.log(`[Plugin:${this.pluginId}]`, ...args),
      error: (...args) => originalConsole.error(`[Plugin:${this.pluginId}]`, ...args),
      warn: (...args) => originalConsole.warn(`[Plugin:${this.pluginId}]`, ...args),
      info: (...args) => originalConsole.info(`[Plugin:${this.pluginId}]`, ...args),
    };
  }

  // 创建安全的setTimeout
  private createSecureSetTimeout(): typeof setTimeout {
    return (callback: Function, delay: number, ...args: any[]) => {
      // 限制延迟时间
      const safeDelay = Math.min(delay, 60000);  // 最大1分钟

      return setTimeout(() => {
        this.executeInSandbox(() => callback(...args));
      }, safeDelay);
    };
  }

  // 创建安全的setInterval
  private createSecureSetInterval(): typeof setInterval {
    return (callback: Function, interval: number, ...args: any[]) => {
      // 限制间隔时间
      const safeInterval = Math.max(interval, 100);  // 最小100ms

      return setInterval(() => {
        this.executeInSandbox(() => callback(...args));
      }, safeInterval);
    };
  }

  // 创建安全的fetch
  private createSecureFetch(): typeof fetch {
    return async (input: RequestInfo, init?: RequestInit) => {
      // 检查URL是否安全
      const url = typeof input === 'string' ? input : input.url;

      if (!this.isUrlAllowed(url)) {
        throw new Error(`URL not allowed: ${url}`);
      }

      // 限制请求大小
      const maxSize = 10 * 1024 * 1024;  // 10MB

      return fetch(input, {
        ...init,
        // 添加安全头
        headers: {
          ...init?.headers,
          'X-Requested-By': `excalidraw-plugin-${this.pluginId}`,
        },
      }).then(response => {
        // 检查响应大小
        const contentLength = response.headers.get('content-length');
        if (contentLength && parseInt(contentLength) > maxSize) {
          throw new Error('Response too large');
        }

        return response;
      });
    };
  }

  // 在沙箱中执行代码
  public executeInSandbox<T>(fn: () => T): T {
    this.startResourceMonitoring();

    try {
      // 设置超时
      const timeoutId = setTimeout(() => {
        throw new Error(`Plugin ${this.pluginId} execution timeout`);
      }, this.timeoutLimit);

      // 在受限上下文中执行
      const result = this.runWithContext(fn);

      clearTimeout(timeoutId);
      return result;
    } finally {
      this.endResourceMonitoring();
    }
  }

  // 在受限上下文中运行
  private runWithContext<T>(fn: () => T): T {
    // 保存原始全局对象
    const originalGlobals: any = {};
    const globalKeys = Object.keys(this.context);

    globalKeys.forEach(key => {
      if (key in globalThis) {
        originalGlobals[key] = (globalThis as any)[key];
      }
    });

    try {
      // 设置沙箱上下文
      globalKeys.forEach(key => {
        (globalThis as any)[key] = this.context[key];
      });

      return fn();
    } finally {
      // 恢复原始全局对象
      globalKeys.forEach(key => {
        if (originalGlobals[key] !== undefined) {
          (globalThis as any)[key] = originalGlobals[key];
        } else {
          delete (globalThis as any)[key];
        }
      });
    }
  }

  // 开始资源监控
  private startResourceMonitoring(): void {
    this.startTime = performance.now();

    // 简化的内存监控（实际实现需要更复杂的逻辑）
    if ('memory' in performance) {
      this.initialMemory = (performance as any).memory.usedJSHeapSize;
    }
  }

  // 结束资源监控
  private endResourceMonitoring(): void {
    const duration = performance.now() - this.startTime;

    if (duration > this.cpuLimit) {
      console.warn(`Plugin ${this.pluginId} exceeded CPU limit: ${duration}ms`);
    }

    // 内存检查
    if ('memory' in performance) {
      const currentMemory = (performance as any).memory.usedJSHeapSize;
      const memoryUsed = currentMemory - this.initialMemory;

      if (memoryUsed > this.memoryLimit) {
        console.warn(`Plugin ${this.pluginId} exceeded memory limit: ${memoryUsed} bytes`);
      }
    }
  }

  // 检查URL是否被允许
  private isUrlAllowed(url: string): boolean {
    try {
      const parsedUrl = new URL(url);

      // 禁止本地网络访问
      const hostname = parsedUrl.hostname;
      if (hostname === 'localhost' ||
          hostname === '127.0.0.1' ||
          hostname.startsWith('192.168.') ||
          hostname.startsWith('10.') ||
          hostname.startsWith('172.')) {
        return false;
      }

      // 只允许HTTPS（除了开发环境）
      if (parsedUrl.protocol !== 'https:' && parsedUrl.protocol !== 'http:') {
        return false;
      }

      return true;
    } catch {
      return false;
    }
  }

  // 设置资源限制
  setTimeoutLimit(ms: number): void {
    this.timeoutLimit = ms;
  }

  setMemoryLimit(bytes: number): void {
    this.memoryLimit = bytes;
  }

  setCPULimit(ms: number): void {
    this.cpuLimit = ms;
  }

  // 检查权限
  hasPermission(permission: PluginPermission): boolean {
    return this.permissions.has(permission);
  }
}
```

## 插件API实现

### ElementAPI实现

```typescript
// packages/excalidraw/plugins/api/ElementAPI.ts
export class ElementAPI {
  private permissions: Set<PluginPermission>;
  private sandbox: PluginSandbox;

  constructor(permissions: Set<PluginPermission>, sandbox: PluginSandbox) {
    this.permissions = permissions;
    this.sandbox = sandbox;
  }

  // 获取所有元素
  getAll(): readonly ExcalidrawElement[] {
    this.checkPermission(PluginPermission.READ_ELEMENTS);

    // 返回元素的只读副本
    return this.getElements().map(el => Object.freeze({ ...el }));
  }

  // 根据ID获取元素
  getById(id: string): ExcalidrawElement | null {
    this.checkPermission(PluginPermission.READ_ELEMENTS);

    const elements = this.getElements();
    return elements.find(el => el.id === id) || null;
  }

  // 获取选中的元素
  getSelected(): readonly ExcalidrawElement[] {
    this.checkPermission(PluginPermission.READ_ELEMENTS);
    this.checkPermission(PluginPermission.READ_APPSTATE);

    const elements = this.getElements();
    const selectedIds = this.getSelectedElementIds();

    return elements.filter(el => selectedIds[el.id]);
  }

  // 创建新元素
  create(elementData: Partial<ExcalidrawElement>): ExcalidrawElement {
    this.checkPermission(PluginPermission.WRITE_ELEMENTS);

    // 验证元素数据
    this.validateElementData(elementData);

    // 生成新元素
    const element = this.createElementFromData(elementData);

    // 添加到画布
    this.addElementToCanvas(element);

    return element;
  }

  // 更新元素
  update(id: string, updates: Partial<ExcalidrawElement>): boolean {
    this.checkPermission(PluginPermission.WRITE_ELEMENTS);

    const elements = this.getElements();
    const elementIndex = elements.findIndex(el => el.id === id);

    if (elementIndex === -1) {
      return false;
    }

    // 验证更新数据
    this.validateElementData(updates);

    // 创建更新后的元素
    const updatedElement = {
      ...elements[elementIndex],
      ...updates,
      versionNonce: this.generateVersionNonce(),
    };

    // 更新画布
    this.updateElementOnCanvas(elementIndex, updatedElement);

    return true;
  }

  // 删除元素
  delete(id: string): boolean {
    this.checkPermission(PluginPermission.WRITE_ELEMENTS);

    const elements = this.getElements();
    const elementIndex = elements.findIndex(el => el.id === id);

    if (elementIndex === -1) {
      return false;
    }

    // 从画布移除
    this.removeElementFromCanvas(elementIndex);

    return true;
  }

  // 批量操作
  batch(operations: ElementOperation[]): boolean {
    this.checkPermission(PluginPermission.WRITE_ELEMENTS);

    try {
      // 在单个事务中执行所有操作
      for (const op of operations) {
        switch (op.type) {
          case 'create':
            this.create(op.data);
            break;
          case 'update':
            this.update(op.id, op.data);
            break;
          case 'delete':
            this.delete(op.id);
            break;
        }
      }
      return true;
    } catch (error) {
      console.error('Batch operation failed:', error);
      return false;
    }
  }

  // 查询元素
  query(predicate: (element: ExcalidrawElement) => boolean): readonly ExcalidrawElement[] {
    this.checkPermission(PluginPermission.READ_ELEMENTS);

    const elements = this.getElements();

    return this.sandbox.executeInSandbox(() => {
      return elements.filter(predicate);
    });
  }

  // 私有方法
  private checkPermission(permission: PluginPermission): void {
    if (!this.permissions.has(permission)) {
      throw new Error(`Plugin does not have permission: ${permission}`);
    }
  }

  private validateElementData(data: Partial<ExcalidrawElement>): void {
    // 验证必要字段和数据类型
    if ('type' in data && !this.isValidElementType(data.type)) {
      throw new Error(`Invalid element type: ${data.type}`);
    }

    if ('x' in data && (typeof data.x !== 'number' || !isFinite(data.x))) {
      throw new Error('Invalid x coordinate');
    }

    if ('y' in data && (typeof data.y !== 'number' || !isFinite(data.y))) {
      throw new Error('Invalid y coordinate');
    }

    // 更多验证...
  }

  private isValidElementType(type: any): boolean {
    const validTypes = ['rectangle', 'ellipse', 'diamond', 'arrow', 'line', 'freedraw', 'text', 'image'];
    return validTypes.includes(type);
  }

  private createElementFromData(data: Partial<ExcalidrawElement>): ExcalidrawElement {
    // 创建完整的元素对象
    return {
      id: data.id || this.generateElementId(),
      type: data.type || 'rectangle',
      x: data.x || 0,
      y: data.y || 0,
      width: data.width || 100,
      height: data.height || 100,
      angle: data.angle || 0,
      strokeColor: data.strokeColor || '#000000',
      backgroundColor: data.backgroundColor || 'transparent',
      fillStyle: data.fillStyle || 'hachure',
      strokeWidth: data.strokeWidth || 1,
      strokeStyle: data.strokeStyle || 'solid',
      roughness: data.roughness || 1,
      opacity: data.opacity || 100,
      versionNonce: this.generateVersionNonce(),
      isDeleted: false,
      groupIds: data.groupIds || [],
      frameId: data.frameId || null,
      index: data.index || this.generateIndex(),
      roundness: data.roundness || null,
      seed: data.seed || Math.floor(Math.random() * 2 ** 31),
      ...data,
    } as ExcalidrawElement;
  }

  // 这些方法需要与Excalidraw的核心系统集成
  private getElements(): readonly ExcalidrawElement[] {
    // 从Excalidraw获取当前元素
    return [];
  }

  private getSelectedElementIds(): Record<string, true> {
    // 从AppState获取选中的元素ID
    return {};
  }

  private addElementToCanvas(element: ExcalidrawElement): void {
    // 将元素添加到画布
  }

  private updateElementOnCanvas(index: number, element: ExcalidrawElement): void {
    // 更新画布上的元素
  }

  private removeElementFromCanvas(index: number): void {
    // 从画布移除元素
  }

  private generateElementId(): string {
    return Math.random().toString(36).substr(2, 9);
  }

  private generateVersionNonce(): number {
    return Math.floor(Math.random() * 2 ** 31);
  }

  private generateIndex(): string {
    return Math.random().toString(36);
  }
}

// 元素操作接口
interface ElementOperation {
  type: 'create' | 'update' | 'delete';
  id?: string;
  data: Partial<ExcalidrawElement>;
}
```

## 实战示例：开发一个插件

### 网格对齐插件

```javascript
// GridSnapPlugin.js
class GridSnapPlugin {
  constructor() {
    this.id = 'grid-snap-plugin';
    this.name = 'Grid Snap';
    this.version = '1.0.0';
    this.description = 'Snap elements to grid';
    this.author = 'Excalidraw Team';

    this.permissions = [
      'read:elements',
      'write:elements',
      'read:appstate',
      'write:appstate',
      'create:ui',
      'register:actions'
    ];

    this.config = {
      gridSize: {
        type: 'number',
        default: 20,
        label: 'Grid Size',
        min: 5,
        max: 100
      },
      enabled: {
        type: 'boolean',
        default: true,
        label: 'Enable Grid Snap'
      },
      snapToGrid: {
        type: 'boolean',
        default: true,
        label: 'Snap to Grid'
      }
    };
  }

  // 插件加载
  async onLoad(api) {
    this.api = api;

    console.log('Grid Snap Plugin loaded');

    // 注册动作
    this.registerActions();

    // 监听元素变化
    this.api.events.on('elements:changed', this.handleElementsChanged.bind(this));

    // 创建UI
    this.createGridOverlay();
  }

  // 插件启用
  async onEnable() {
    console.log('Grid Snap Plugin enabled');

    // 显示网格
    this.showGrid();
  }

  // 插件禁用
  async onDisable() {
    console.log('Grid Snap Plugin disabled');

    // 隐藏网格
    this.hideGrid();
  }

  // 注册动作
  registerActions() {
    // 切换网格显示
    this.api.actions.register({
      id: 'toggle-grid',
      name: 'Toggle Grid',
      icon: '⊞',
      keywords: ['grid', 'snap'],
      perform: () => {
        const config = this.api.config.get('enabled');
        this.api.config.set('enabled', !config);

        if (config) {
          this.hideGrid();
        } else {
          this.showGrid();
        }

        return { commitToHistory: false };
      },
      keyTest: (event) => {
        return event.ctrlKey && event.key === 'g';
      }
    });

    // 对齐到网格
    this.api.actions.register({
      id: 'align-to-grid',
      name: 'Align to Grid',
      icon: '⊡',
      keywords: ['align', 'grid', 'snap'],
      perform: () => {
        const selectedElements = this.api.elements.getSelected();

        if (selectedElements.length === 0) {
          return { commitToHistory: false };
        }

        const gridSize = this.api.config.get('gridSize');

        // 对齐选中元素到网格
        selectedElements.forEach(element => {
          const snappedX = Math.round(element.x / gridSize) * gridSize;
          const snappedY = Math.round(element.y / gridSize) * gridSize;

          this.api.elements.update(element.id, {
            x: snappedX,
            y: snappedY
          });
        });

        return { commitToHistory: true };
      },
      predicate: () => {
        return this.api.elements.getSelected().length > 0;
      }
    });
  }

  // 处理元素变化
  handleElementsChanged(elements) {
    if (!this.api.config.get('snapToGrid')) {
      return;
    }

    const gridSize = this.api.config.get('gridSize');

    // 自动对齐新创建的元素
    elements.forEach(element => {
      if (this.isNewElement(element)) {
        const snappedX = Math.round(element.x / gridSize) * gridSize;
        const snappedY = Math.round(element.y / gridSize) * gridSize;

        if (snappedX !== element.x || snappedY !== element.y) {
          this.api.elements.update(element.id, {
            x: snappedX,
            y: snappedY
          });
        }
      }
    });
  }

  // 创建网格覆盖层
  createGridOverlay() {
    const overlay = this.api.ui.createElement('div', {
      id: 'grid-overlay',
      className: 'grid-overlay',
      style: {
        position: 'absolute',
        top: 0,
        left: 0,
        width: '100%',
        height: '100%',
        pointerEvents: 'none',
        opacity: 0.1,
        background: this.createGridPattern()
      }
    });

    this.api.ui.appendToCanvas(overlay);
  }

  // 创建网格图案
  createGridPattern() {
    const gridSize = this.api.config.get('gridSize');

    const svg = `
      <svg width="${gridSize}" height="${gridSize}" xmlns="http://www.w3.org/2000/svg">
        <defs>
          <pattern id="grid" width="${gridSize}" height="${gridSize}" patternUnits="userSpaceOnUse">
            <path d="M ${gridSize} 0 L 0 0 0 ${gridSize}" fill="none" stroke="#666" stroke-width="1"/>
          </pattern>
        </defs>
        <rect width="100%" height="100%" fill="url(#grid)" />
      </svg>
    `;

    return `url("data:image/svg+xml,${encodeURIComponent(svg)}")`;
  }

  // 显示网格
  showGrid() {
    const overlay = this.api.ui.getElementById('grid-overlay');
    if (overlay) {
      overlay.style.display = 'block';
    }
  }

  // 隐藏网格
  hideGrid() {
    const overlay = this.api.ui.getElementById('grid-overlay');
    if (overlay) {
      overlay.style.display = 'none';
    }
  }

  // 检查是否是新元素
  isNewElement(element) {
    // 简化的新元素检测逻辑
    return Date.now() - element.versionNonce < 1000;
  }
}

// 注册插件
if (typeof window !== 'undefined' && window.excalidrawPluginManager) {
  window.excalidrawPluginManager.registerPlugin(new GridSnapPlugin());
}
```

## 思考题

1. **插件安全如何保证？** 恶意插件的防护措施？

2. **插件性能如何优化？** 避免插件影响主应用性能？

3. **插件生态如何建设？** 插件商店、版本管理、依赖解析？

4. **跨平台兼容如何处理？** 不同环境下的插件适配？

5. **插件调试如何支持？** 开发者工具和调试接口？

## 总结

插件系统为Excalidraw提供了强大的扩展能力：

1. **安全沙箱**：保护主应用免受恶意插件影响
2. **丰富API**：提供完整的功能访问接口
3. **权限控制**：细粒度的权限管理机制
4. **资源限制**：防止插件过度占用系统资源
5. **开发友好**：简单易用的插件开发接口

这套插件架构为构建可扩展的应用提供了完整的解决方案。

## 下一步

下一章将探讨导入导出系统的实现，了解如何处理各种文件格式和数据转换。