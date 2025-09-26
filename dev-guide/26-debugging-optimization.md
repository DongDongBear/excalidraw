# 8.2 调试与性能优化实践

本节提供 Excalidraw 开发过程中的调试技巧、性能优化方法和问题排查策略。

## 调试工具与技术

### 1. 浏览器开发者工具使用

```typescript
// 调试辅助工具类
class ExcalidrawDebugger {
  private static instance: ExcalidrawDebugger;
  private debugPanel: HTMLElement | null = null;
  private metrics: DebugMetrics = {
    renderCount: 0,
    lastRenderTime: 0,
    elementCount: 0,
    selectedElementCount: 0
  };

  static getInstance(): ExcalidrawDebugger {
    if (!ExcalidrawDebugger.instance) {
      ExcalidrawDebugger.instance = new ExcalidrawDebugger();
    }
    return ExcalidrawDebugger.instance;
  }

  // 启用调试模式
  enableDebugMode(): void {
    if (process.env.NODE_ENV !== 'development') {
      console.warn('Debug mode only available in development');
      return;
    }

    this.createDebugPanel();
    this.setupPerformanceMonitoring();
    this.exposeGlobalAPI();

    console.log('🎨 Excalidraw Debug Mode Enabled');
    console.log('Use window.__excalidraw for debugging APIs');
  }

  // 创建调试面板
  private createDebugPanel(): void {
    this.debugPanel = document.createElement('div');
    this.debugPanel.id = 'excalidraw-debug-panel';
    this.debugPanel.innerHTML = `
      <div style="
        position: fixed;
        top: 10px;
        right: 10px;
        background: rgba(0, 0, 0, 0.8);
        color: white;
        padding: 12px;
        border-radius: 4px;
        font-family: monospace;
        font-size: 12px;
        z-index: 10000;
        min-width: 200px;
      ">
        <h4 style="margin: 0 0 8px 0;">🎨 Excalidraw Debug</h4>
        <div id="debug-metrics"></div>
        <div style="margin-top: 8px;">
          <button id="clear-canvas">Clear Canvas</button>
          <button id="toggle-grid">Toggle Grid</button>
          <button id="export-state">Export State</button>
        </div>
      </div>
    `;

    document.body.appendChild(this.debugPanel);
    this.bindDebugPanelEvents();
    this.updateDebugMetrics();
  }

  // 性能监控
  private setupPerformanceMonitoring(): void {
    // 监控渲染性能
    const originalRender = HTMLCanvasElement.prototype.getContext;
    HTMLCanvasElement.prototype.getContext = function(contextId: string, options?: any) {
      const context = originalRender.call(this, contextId, options);

      if (contextId === '2d' && context) {
        const debugger = ExcalidrawDebugger.getInstance();
        debugger.wrapCanvasContext(context as CanvasRenderingContext2D);
      }

      return context;
    };

    // 监控内存使用
    if ('memory' in performance) {
      setInterval(() => {
        const memory = (performance as any).memory;
        console.log(`Memory: ${Math.round(memory.usedJSHeapSize / 1024 / 1024)}MB`);
      }, 5000);
    }
  }

  // 包装 Canvas 上下文以监控绘制调用
  private wrapCanvasContext(ctx: CanvasRenderingContext2D): void {
    const originalMethods = [
      'fillRect', 'strokeRect', 'clearRect',
      'fillText', 'strokeText',
      'drawImage', 'putImageData'
    ];

    originalMethods.forEach(method => {
      const original = ctx[method as keyof CanvasRenderingContext2D] as Function;

      (ctx as any)[method] = (...args: any[]) => {
        this.metrics.renderCount++;
        const startTime = performance.now();

        const result = original.apply(ctx, args);

        this.metrics.lastRenderTime = performance.now() - startTime;
        return result;
      };
    });
  }

  // 暴露全局调试 API
  private exposeGlobalAPI(): void {
    (window as any).__excalidraw = {
      // 获取当前状态
      getState: () => ({
        elements: window.__excalidrawAPI?.getSceneElements(),
        appState: window.__excalidrawAPI?.getAppState()
      }),

      // 性能分析
      startProfiling: () => {
        console.profile('Excalidraw Performance');
      },

      stopProfiling: () => {
        console.profileEnd('Excalidraw Performance');
      },

      // 内存快照
      takeMemorySnapshot: () => {
        if ('memory' in performance) {
          const memory = (performance as any).memory;
          console.table({
            'Used JS Heap': `${Math.round(memory.usedJSHeapSize / 1024 / 1024)}MB`,
            'Total JS Heap': `${Math.round(memory.totalJSHeapSize / 1024 / 1024)}MB`,
            'JS Heap Limit': `${Math.round(memory.jsHeapSizeLimit / 1024 / 1024)}MB`
          });
        }
      },

      // 导出调试信息
      exportDebugInfo: () => {
        const debugInfo = {
          timestamp: new Date().toISOString(),
          userAgent: navigator.userAgent,
          viewport: {
            width: window.innerWidth,
            height: window.innerHeight,
            devicePixelRatio: window.devicePixelRatio
          },
          elements: window.__excalidrawAPI?.getSceneElements()?.length || 0,
          appState: window.__excalidrawAPI?.getAppState(),
          metrics: this.metrics
        };

        console.log('Debug Info:', debugInfo);
        return debugInfo;
      }
    };
  }

  updateDebugMetrics(): void {
    if (!this.debugPanel) return;

    const metricsElement = this.debugPanel.querySelector('#debug-metrics');
    if (metricsElement) {
      metricsElement.innerHTML = `
        <div>Renders: ${this.metrics.renderCount}</div>
        <div>Last Render: ${this.metrics.lastRenderTime.toFixed(2)}ms</div>
        <div>Elements: ${this.metrics.elementCount}</div>
        <div>Selected: ${this.metrics.selectedElementCount}</div>
      `;
    }

    requestAnimationFrame(() => this.updateDebugMetrics());
  }
}

// 自动启用调试模式（仅开发环境）
if (process.env.NODE_ENV === 'development') {
  ExcalidrawDebugger.getInstance().enableDebugMode();
}
```

### 2. React DevTools 集成

```typescript
// React DevTools 专用钩子
export const useExcalidrawDevtools = (
  elements: ExcalidrawElement[],
  appState: AppState
) => {
  const devtoolsData = useMemo(() => ({
    elementCount: elements.length,
    selectedCount: Object.keys(appState.selectedElementIds).length,
    activeTool: appState.activeTool.type,
    viewState: {
      zoom: appState.zoom,
      scrollX: appState.scrollX,
      scrollY: appState.scrollY
    },
    performance: {
      lastRenderTime: performance.now()
    }
  }), [elements, appState]);

  // 在 React DevTools 中显示组件信息
  useDebugValue(devtoolsData);

  // 发送数据到 React DevTools Profiler
  React.unstable_useOpaqueIdentifier && React.unstable_useOpaqueIdentifier();

  return devtoolsData;
};

// 用法示例
const ExcalidrawApp = () => {
  const [elements, setElements] = useState<ExcalidrawElement[]>([]);
  const [appState, setAppState] = useState<AppState>(getDefaultAppState());

  // 开发工具集成
  const devtoolsData = useExcalidrawDevtools(elements, appState);

  return (
    <div className="excalidraw-app">
      <Excalidraw
        initialData={{ elements, appState }}
        onChange={(elements, appState) => {
          setElements(elements);
          setAppState(appState);
        }}
      />
    </div>
  );
};
```

## 性能分析与优化

### 1. 渲染性能监控

```typescript
class RenderingProfiler {
  private frameId: number = 0;
  private frameTimes: number[] = [];
  private isRecording: boolean = false;

  // 开始性能记录
  startRecording(): void {
    this.isRecording = true;
    this.frameTimes = [];
    this.recordFrame();
  }

  // 停止记录并生成报告
  stopRecording(): PerformanceReport {
    this.isRecording = false;
    cancelAnimationFrame(this.frameId);

    return this.generateReport();
  }

  private recordFrame(): void {
    if (!this.isRecording) return;

    const frameStart = performance.now();

    this.frameId = requestAnimationFrame(() => {
      const frameTime = performance.now() - frameStart;
      this.frameTimes.push(frameTime);

      // 保持最近 1000 帧的数据
      if (this.frameTimes.length > 1000) {
        this.frameTimes.shift();
      }

      this.recordFrame();
    });
  }

  private generateReport(): PerformanceReport {
    if (this.frameTimes.length === 0) {
      return { avgFrameTime: 0, fps: 0, dropCount: 0 };
    }

    const avgFrameTime = this.frameTimes.reduce((a, b) => a + b) / this.frameTimes.length;
    const fps = 1000 / avgFrameTime;

    // 计算掉帧数（超过 16.67ms 的帧）
    const dropCount = this.frameTimes.filter(time => time > 16.67).length;

    const p95 = this.calculatePercentile(this.frameTimes, 95);
    const p99 = this.calculatePercentile(this.frameTimes, 99);

    return {
      avgFrameTime,
      fps,
      dropCount,
      dropRate: dropCount / this.frameTimes.length,
      p95FrameTime: p95,
      p99FrameTime: p99,
      recommendations: this.generateRecommendations(avgFrameTime, dropCount)
    };
  }

  private calculatePercentile(values: number[], percentile: number): number {
    const sorted = [...values].sort((a, b) => a - b);
    const index = Math.ceil(sorted.length * (percentile / 100)) - 1;
    return sorted[index];
  }

  private generateRecommendations(avgFrameTime: number, dropCount: number): string[] {
    const recommendations: string[] = [];

    if (avgFrameTime > 16.67) {
      recommendations.push('Frame time is high - consider optimizing render operations');
    }

    if (dropCount > 0) {
      recommendations.push('Experiencing frame drops - check for expensive operations in render loop');
    }

    return recommendations;
  }
}

// 使用示例
const profiler = new RenderingProfiler();

// 在需要分析的代码段周围使用
profiler.startRecording();

// ... 执行需要分析的渲染代码 ...

setTimeout(() => {
  const report = profiler.stopRecording();
  console.table(report);
}, 10000); // 记录 10 秒
```

### 2. 内存泄漏检测

```typescript
class MemoryLeakDetector {
  private snapshots: Array<{
    timestamp: number;
    heapUsed: number;
    elementsCount: number;
    eventListenersCount: number;
  }> = [];

  private intervalId: number | null = null;

  startMonitoring(): void {
    this.intervalId = window.setInterval(() => {
      this.takeSnapshot();
    }, 5000); // 每 5 秒一次快照
  }

  stopMonitoring(): void {
    if (this.intervalId) {
      clearInterval(this.intervalId);
      this.intervalId = null;
    }
  }

  private takeSnapshot(): void {
    const snapshot = {
      timestamp: Date.now(),
      heapUsed: this.getHeapSize(),
      elementsCount: document.querySelectorAll('*').length,
      eventListenersCount: this.getEventListenerCount()
    };

    this.snapshots.push(snapshot);

    // 保持最近 50 个快照
    if (this.snapshots.length > 50) {
      this.snapshots.shift();
    }

    this.analyzeMemoryTrend();
  }

  private getHeapSize(): number {
    if ('memory' in performance) {
      return (performance as any).memory.usedJSHeapSize;
    }
    return 0;
  }

  private getEventListenerCount(): number {
    // 这是一个简化的实现，实际中可能需要更复杂的检测
    return (window as any).__eventListenerCount || 0;
  }

  private analyzeMemoryTrend(): void {
    if (this.snapshots.length < 10) return;

    const recent = this.snapshots.slice(-10);
    const first = recent[0];
    const last = recent[recent.length - 1];

    const memoryGrowth = last.heapUsed - first.heapUsed;
    const timeSpan = last.timestamp - first.timestamp;
    const growthRate = memoryGrowth / timeSpan; // bytes per ms

    // 如果内存增长率超过阈值，发出警告
    if (growthRate > 1000) { // 1KB per second
      console.warn('🚨 Potential memory leak detected!', {
        growthRate: `${Math.round(growthRate * 1000)}KB/s`,
        currentHeap: `${Math.round(last.heapUsed / 1024 / 1024)}MB`
      });

      this.generateMemoryReport();
    }
  }

  private generateMemoryReport(): void {
    const report = {
      snapshots: this.snapshots,
      summary: {
        totalGrowth: this.snapshots[this.snapshots.length - 1]?.heapUsed -
                    this.snapshots[0]?.heapUsed,
        avgHeapSize: this.snapshots.reduce((sum, s) => sum + s.heapUsed, 0) /
                    this.snapshots.length,
        peakHeapSize: Math.max(...this.snapshots.map(s => s.heapUsed))
      },
      recommendations: [
        'Check for unremoved event listeners',
        'Verify Canvas contexts are properly disposed',
        'Check for circular references in objects',
        'Monitor large object allocations'
      ]
    };

    console.log('Memory Report:', report);
  }
}
```

### 3. 代码分割和懒加载

```typescript
// 动态导入和代码分割
const LazyExportDialog = React.lazy(() =>
  import('./components/ExportDialog').then(module => ({
    default: module.ExportDialog
  }))
);

const LazyCollaborationPanel = React.lazy(() =>
  import('./components/CollaborationPanel')
);

// 智能预加载
class PreloadManager {
  private preloadedModules = new Set<string>();

  // 预加载核心模块
  preloadCoreModules(): void {
    const coreModules = [
      () => import('./renderer/RoughRenderer'),
      () => import('./tools/SelectionTool'),
      () => import('./actions/ActionManager')
    ];

    // 在空闲时间预加载
    if ('requestIdleCallback' in window) {
      coreModules.forEach(moduleLoader => {
        requestIdleCallback(() => {
          moduleLoader().catch(error => {
            console.warn('Failed to preload module:', error);
          });
        });
      });
    }
  }

  // 基于用户行为的智能预加载
  preloadBasedOnUserAction(action: string): void {
    const preloadMap: Record<string, () => Promise<any>> = {
      'export-hovered': () => import('./exporters/ImageExporter'),
      'collaboration-clicked': () => import('./collab/CollaborationManager'),
      'library-opened': () => import('./library/LibraryManager')
    };

    const moduleLoader = preloadMap[action];
    if (moduleLoader && !this.preloadedModules.has(action)) {
      this.preloadedModules.add(action);
      moduleLoader().catch(error => {
        console.warn(`Failed to preload ${action} module:`, error);
        this.preloadedModules.delete(action);
      });
    }
  }
}

// Suspense 封装组件
const SuspenseWrapper: React.FC<{
  children: React.ReactNode;
  fallback?: React.ReactNode;
}> = ({ children, fallback }) => {
  const defaultFallback = (
    <div className="loading-spinner">
      <div>Loading...</div>
    </div>
  );

  return (
    <React.Suspense fallback={fallback || defaultFallback}>
      <ErrorBoundary>
        {children}
      </ErrorBoundary>
    </React.Suspense>
  );
};
```

## 错误处理与日志

### 1. 错误边界和错误报告

```typescript
class ExcalidrawErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean; errorInfo?: any }
> {
  constructor(props: { children: React.ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    this.setState({ errorInfo });

    // 记录错误详情
    this.logError(error, errorInfo);

    // 发送错误报告
    this.reportError(error, errorInfo);
  }

  private logError(error: Error, errorInfo: React.ErrorInfo): void {
    const errorDetails = {
      message: error.message,
      stack: error.stack,
      componentStack: errorInfo.componentStack,
      timestamp: new Date().toISOString(),
      userAgent: navigator.userAgent,
      url: window.location.href,
      userId: this.getUserId(), // 如果有用户ID
      sessionId: this.getSessionId()
    };

    console.group('🚨 Excalidraw Error');
    console.error('Error:', error);
    console.error('Error Info:', errorInfo);
    console.error('Details:', errorDetails);
    console.groupEnd();

    // 保存到本地存储用于离线分析
    this.saveErrorToLocalStorage(errorDetails);
  }

  private reportError(error: Error, errorInfo: React.ErrorInfo): void {
    if (process.env.NODE_ENV === 'production') {
      // 发送到错误监控服务
      this.sendToErrorService({
        error: {
          message: error.message,
          stack: error.stack
        },
        context: {
          componentStack: errorInfo.componentStack,
          timestamp: Date.now(),
          userAgent: navigator.userAgent
        }
      });
    }
  }

  private sendToErrorService(errorData: any): void {
    // 实际项目中应该集成 Sentry、Bugsnag 等服务
    fetch('/api/errors', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(errorData)
    }).catch(err => {
      console.warn('Failed to report error:', err);
    });
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary">
          <h2>Something went wrong with Excalidraw</h2>
          <details>
            <summary>Error details</summary>
            <pre>{JSON.stringify(this.state.errorInfo, null, 2)}</pre>
          </details>
          <button onClick={() => window.location.reload()}>
            Reload Page
          </button>
          <button onClick={this.handleReportBug}>
            Report Bug
          </button>
        </div>
      );
    }

    return this.props.children;
  }

  private handleReportBug = (): void => {
    const errorReport = this.generateErrorReport();
    const bugReportUrl = `https://github.com/excalidraw/excalidraw/issues/new?template=bug_report.md&body=${encodeURIComponent(errorReport)}`;
    window.open(bugReportUrl, '_blank');
  };
}
```

### 2. 日志系统

```typescript
enum LogLevel {
  DEBUG = 0,
  INFO = 1,
  WARN = 2,
  ERROR = 3
}

class Logger {
  private static instance: Logger;
  private logLevel: LogLevel = LogLevel.INFO;
  private logs: Array<{
    level: LogLevel;
    message: string;
    timestamp: number;
    context?: any;
  }> = [];

  static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger();
    }
    return Logger.instance;
  }

  setLogLevel(level: LogLevel): void {
    this.logLevel = level;
  }

  debug(message: string, context?: any): void {
    this.log(LogLevel.DEBUG, message, context);
  }

  info(message: string, context?: any): void {
    this.log(LogLevel.INFO, message, context);
  }

  warn(message: string, context?: any): void {
    this.log(LogLevel.WARN, message, context);
  }

  error(message: string, context?: any): void {
    this.log(LogLevel.ERROR, message, context);
  }

  private log(level: LogLevel, message: string, context?: any): void {
    if (level < this.logLevel) return;

    const logEntry = {
      level,
      message,
      timestamp: Date.now(),
      context
    };

    this.logs.push(logEntry);

    // 保持最近 1000 条日志
    if (this.logs.length > 1000) {
      this.logs.shift();
    }

    // 输出到控制台
    this.outputToConsole(logEntry);
  }

  private outputToConsole(logEntry: any): void {
    const { level, message, context } = logEntry;
    const timestamp = new Date(logEntry.timestamp).toISOString();

    const logMethods = {
      [LogLevel.DEBUG]: console.debug,
      [LogLevel.INFO]: console.info,
      [LogLevel.WARN]: console.warn,
      [LogLevel.ERROR]: console.error
    };

    const method = logMethods[level];

    if (context) {
      method(`[${timestamp}] ${message}`, context);
    } else {
      method(`[${timestamp}] ${message}`);
    }
  }

  // 导出日志用于分析
  exportLogs(): string {
    return JSON.stringify(this.logs, null, 2);
  }

  // 清理旧日志
  clearLogs(): void {
    this.logs = [];
  }
}

// 全局日志实例
const logger = Logger.getInstance();

// 使用示例
logger.info('Excalidraw initialized', { version: '1.0.0' });
logger.warn('Large number of elements detected', { count: 1000 });
logger.error('Failed to save drawing', { error: 'Network error' });
```

通过这些调试和性能优化实践，开发者可以更好地诊断问题、优化性能并确保 Excalidraw 的稳定运行。这些工具和技术为维护高质量的代码提供了坚实的基础。