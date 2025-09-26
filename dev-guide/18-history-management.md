# Chapter 5.2: 历史管理与状态快照

## 概述

历史管理是现代编辑器的基础功能，用户期望能够无缝地撤销和重做操作。Excalidraw的历史管理系统不仅要处理简单的状态回滚，还要解决大型场景的内存优化、协作环境的状态同步、以及复杂操作的原子性保证。本章将深入探讨这套高效的历史管理机制。

## 历史管理架构设计

### 分层存储策略

```
历史存储层次
├── 内存层 (Memory Layer)
│   ├── 热数据缓存      # 最近的20个快照
│   ├── 快照索引       # 快速访问索引
│   └── 状态diff       # 增量差异数据
├── 本地存储层 (Local Storage)
│   ├── IndexedDB      # 历史快照持久化
│   ├── 压缩存储       # LZ4压缩算法
│   └── 自动清理       # 过期数据清理
└── 云端同步层 (Cloud Sync)
    ├── 关键节点同步    # 重要快照云端备份
    ├── 冲突解决       # 多设备历史冲突
    └── 版本分支       # 历史分支管理
```

### 核心数据结构

```typescript
// 历史快照接口
export interface HistorySnapshot {
  id: string;
  timestamp: number;
  elements: readonly ExcalidrawElement[];
  appState: AppState;
  files?: BinaryFiles;

  // 元数据
  metadata: {
    actionName?: ActionName;
    userInitiated: boolean;
    autoSave: boolean;
    description?: string;
    tags?: string[];
  };

  // 优化字段
  optimizations?: {
    compressed: boolean;
    deltaFrom?: string;  // 基于哪个快照的增量
    size: number;        // 原始大小
    compressedSize?: number;
  };
}

// 增量差异
export interface HistoryDelta {
  baseSnapshotId: string;
  operations: DeltaOperation[];
  timestamp: number;
}

export interface DeltaOperation {
  type: 'add' | 'remove' | 'modify' | 'move';
  elementId?: string;
  data?: Partial<ExcalidrawElement>;
  path?: string;        // 状态路径，如 'appState.zoom'
  oldValue?: any;
  newValue?: any;
}

// 历史统计信息
export interface HistoryStats {
  totalSnapshots: number;
  memoryUsage: number;      // 字节
  storageUsage: number;     // 字节
  compressionRatio: number; // 压缩比
  oldestSnapshot: number;   // 时间戳
  averageSnapshotSize: number;
}
```

## 高级历史管理器

### 智能历史管理器实现

```typescript
// packages/excalidraw/history/AdvancedHistoryManager.ts
export class AdvancedHistoryManager {
  private snapshots: Map<string, HistorySnapshot> = new Map();
  private undoStack: string[] = [];
  private redoStack: string[] = [];
  private currentSnapshotId: string | null = null;

  // 配置选项
  private options: {
    maxMemorySnapshots: number;      // 内存中最大快照数
    maxTotalSnapshots: number;       // 总共最大快照数
    compressionThreshold: number;    // 压缩阈值（字节）
    deltaThreshold: number;          // 增量存储阈值
    autoSaveInterval: number;        // 自动保存间隔（ms）
    enablePersistence: boolean;      // 启用持久化
  };

  // 组件
  private compressor: SnapshotCompressor;
  private deltaCalculator: DeltaCalculator;
  private persistenceManager: PersistenceManager;
  private conflictResolver: ConflictResolver;

  constructor(options: Partial<typeof this.options> = {}) {
    this.options = {
      maxMemorySnapshots: 20,
      maxTotalSnapshots: 100,
      compressionThreshold: 1024 * 100, // 100KB
      deltaThreshold: 0.3,              // 30%相似度
      autoSaveInterval: 30000,          // 30s
      enablePersistence: true,
      ...options,
    };

    this.compressor = new SnapshotCompressor();
    this.deltaCalculator = new DeltaCalculator();
    this.persistenceManager = new PersistenceManager();
    this.conflictResolver = new ConflictResolver();

    this.initializeAutoSave();
    this.loadPersistedHistory();
  }

  // 创建快照
  async createSnapshot(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    files?: BinaryFiles,
    metadata: Partial<HistorySnapshot['metadata']> = {}
  ): Promise<string> {
    const snapshotId = this.generateId();

    // 创建基础快照
    const snapshot: HistorySnapshot = {
      id: snapshotId,
      timestamp: Date.now(),
      elements: this.deepClone(elements),
      appState: this.deepClone(appState),
      files: files ? this.deepClone(files) : undefined,
      metadata: {
        userInitiated: true,
        autoSave: false,
        ...metadata,
      },
    };

    // 计算快照大小
    const size = this.calculateSnapshotSize(snapshot);

    // 决定存储策略
    const storageStrategy = await this.determineStorageStrategy(snapshot, size);

    switch (storageStrategy.type) {
      case 'full':
        await this.storeFullSnapshot(snapshot, size);
        break;
      case 'delta':
        await this.storeDeltaSnapshot(snapshot, storageStrategy.baseId!, size);
        break;
      case 'compressed':
        await this.storeCompressedSnapshot(snapshot, size);
        break;
    }

    // 添加到撤销栈
    this.undoStack.push(snapshotId);
    this.clearRedoStack();

    // 更新当前快照
    this.currentSnapshotId = snapshotId;

    // 内存管理
    await this.performMemoryManagement();

    return snapshotId;
  }

  // 决定存储策略
  private async determineStorageStrategy(
    snapshot: HistorySnapshot,
    size: number
  ): Promise<{ type: 'full' | 'delta' | 'compressed'; baseId?: string }> {
    // 如果是第一个快照，使用完整存储
    if (this.snapshots.size === 0) {
      return { type: 'full' };
    }

    // 如果大于压缩阈值，考虑压缩
    if (size > this.options.compressionThreshold) {
      // 寻找相似的基础快照
      const baseSnapshot = await this.findSimilarSnapshot(snapshot);

      if (baseSnapshot) {
        const similarity = this.deltaCalculator.calculateSimilarity(
          baseSnapshot,
          snapshot
        );

        // 如果相似度高，使用增量存储
        if (similarity > this.options.deltaThreshold) {
          return { type: 'delta', baseId: baseSnapshot.id };
        }
      }

      // 否则使用压缩存储
      return { type: 'compressed' };
    }

    return { type: 'full' };
  }

  // 存储完整快照
  private async storeFullSnapshot(snapshot: HistorySnapshot, size: number) {
    snapshot.optimizations = {
      compressed: false,
      size,
    };

    this.snapshots.set(snapshot.id, snapshot);

    if (this.options.enablePersistence) {
      await this.persistenceManager.storeSnapshot(snapshot);
    }
  }

  // 存储增量快照
  private async storeDeltaSnapshot(
    snapshot: HistorySnapshot,
    baseSnapshotId: string,
    size: number
  ) {
    const baseSnapshot = this.snapshots.get(baseSnapshotId)!;
    const delta = this.deltaCalculator.calculateDelta(baseSnapshot, snapshot);

    // 创建增量快照
    const deltaSnapshot: HistorySnapshot = {
      ...snapshot,
      elements: [] as any,  // 空数组，实际数据在delta中
      appState: {} as any,  // 空对象，实际数据在delta中
      optimizations: {
        compressed: false,
        deltaFrom: baseSnapshotId,
        size,
        compressedSize: this.calculateDeltaSize(delta),
      },
    };

    this.snapshots.set(snapshot.id, deltaSnapshot);

    // 存储增量数据
    if (this.options.enablePersistence) {
      await this.persistenceManager.storeDelta(snapshot.id, delta);
    }
  }

  // 存储压缩快照
  private async storeCompressedSnapshot(snapshot: HistorySnapshot, size: number) {
    const compressed = await this.compressor.compress(snapshot);

    snapshot.optimizations = {
      compressed: true,
      size,
      compressedSize: compressed.size,
    };

    this.snapshots.set(snapshot.id, snapshot);

    if (this.options.enablePersistence) {
      await this.persistenceManager.storeCompressedSnapshot(compressed);
    }
  }

  // 恢复快照
  async restoreSnapshot(snapshotId: string): Promise<HistorySnapshot | null> {
    let snapshot = this.snapshots.get(snapshotId);

    if (!snapshot) {
      // 尝试从持久化存储加载
      snapshot = await this.persistenceManager.loadSnapshot(snapshotId);
      if (snapshot) {
        this.snapshots.set(snapshotId, snapshot);
      }
    }

    if (!snapshot) {
      return null;
    }

    // 如果是增量快照，需要重构完整数据
    if (snapshot.optimizations?.deltaFrom) {
      snapshot = await this.reconstructFromDelta(snapshot);
    }

    // 如果是压缩快照，需要解压
    if (snapshot.optimizations?.compressed) {
      snapshot = await this.compressor.decompress(snapshot);
    }

    return snapshot;
  }

  // 从增量重构快照
  private async reconstructFromDelta(deltaSnapshot: HistorySnapshot): Promise<HistorySnapshot> {
    const baseSnapshotId = deltaSnapshot.optimizations!.deltaFrom!;
    const baseSnapshot = await this.restoreSnapshot(baseSnapshotId);

    if (!baseSnapshot) {
      throw new Error(`Base snapshot ${baseSnapshotId} not found`);
    }

    // 加载增量数据
    const delta = await this.persistenceManager.loadDelta(deltaSnapshot.id);

    // 应用增量到基础快照
    const reconstructed = this.deltaCalculator.applyDelta(baseSnapshot, delta);

    return {
      ...deltaSnapshot,
      elements: reconstructed.elements,
      appState: reconstructed.appState,
      files: reconstructed.files,
    };
  }

  // 执行撤销
  async undo(): Promise<HistorySnapshot | null> {
    if (this.undoStack.length <= 1) {
      return null;
    }

    // 移动当前快照到重做栈
    const currentId = this.undoStack.pop()!;
    this.redoStack.push(currentId);

    // 获取上一个快照
    const prevId = this.undoStack[this.undoStack.length - 1];
    const snapshot = await this.restoreSnapshot(prevId);

    this.currentSnapshotId = prevId;

    return snapshot;
  }

  // 执行重做
  async redo(): Promise<HistorySnapshot | null> {
    if (this.redoStack.length === 0) {
      return null;
    }

    const nextId = this.redoStack.pop()!;
    const snapshot = await this.restoreSnapshot(nextId);

    this.undoStack.push(nextId);
    this.currentSnapshotId = nextId;

    return snapshot;
  }

  // 内存管理
  private async performMemoryManagement() {
    // 清理超出限制的内存快照
    if (this.snapshots.size > this.options.maxMemorySnapshots) {
      await this.evictOldSnapshots();
    }

    // 清理超出限制的总快照
    if (this.undoStack.length > this.options.maxTotalSnapshots) {
      await this.pruneOldHistory();
    }
  }

  // 驱逐旧快照
  private async evictOldSnapshots() {
    const sortedSnapshots = Array.from(this.snapshots.values())
      .sort((a, b) => a.timestamp - b.timestamp);

    const toEvict = sortedSnapshots.slice(0, -this.options.maxMemorySnapshots);

    for (const snapshot of toEvict) {
      // 如果在撤销栈中，移动到持久化存储
      if (this.undoStack.includes(snapshot.id) || this.redoStack.includes(snapshot.id)) {
        if (!snapshot.optimizations?.compressed) {
          await this.persistenceManager.storeSnapshot(snapshot);
        }
      }

      this.snapshots.delete(snapshot.id);
    }
  }

  // 修剪旧历史
  private async pruneOldHistory() {
    const excessCount = this.undoStack.length - this.options.maxTotalSnapshots;
    const toRemove = this.undoStack.splice(0, excessCount);

    for (const snapshotId of toRemove) {
      await this.persistenceManager.removeSnapshot(snapshotId);
      this.snapshots.delete(snapshotId);
    }
  }

  // 寻找相似快照
  private async findSimilarSnapshot(
    targetSnapshot: HistorySnapshot
  ): Promise<HistorySnapshot | null> {
    const recentSnapshots = Array.from(this.snapshots.values())
      .sort((a, b) => b.timestamp - a.timestamp)
      .slice(0, 10);  // 只检查最近10个快照

    let bestMatch: HistorySnapshot | null = null;
    let bestSimilarity = 0;

    for (const snapshot of recentSnapshots) {
      const similarity = this.deltaCalculator.calculateSimilarity(
        snapshot,
        targetSnapshot
      );

      if (similarity > bestSimilarity) {
        bestMatch = snapshot;
        bestSimilarity = similarity;
      }
    }

    return bestMatch;
  }

  // 自动保存
  private initializeAutoSave() {
    if (this.options.autoSaveInterval > 0) {
      setInterval(() => {
        this.performAutoSave();
      }, this.options.autoSaveInterval);
    }
  }

  private async performAutoSave() {
    if (this.currentSnapshotId) {
      const snapshot = this.snapshots.get(this.currentSnapshotId);
      if (snapshot && !snapshot.metadata.autoSave) {
        // 标记为自动保存并持久化
        snapshot.metadata.autoSave = true;
        await this.persistenceManager.storeSnapshot(snapshot);
      }
    }
  }

  // 获取统计信息
  getStats(): HistoryStats {
    const snapshots = Array.from(this.snapshots.values());
    const totalSize = snapshots.reduce((sum, s) => sum + (s.optimizations?.size || 0), 0);
    const compressedSize = snapshots.reduce((sum, s) => sum + (s.optimizations?.compressedSize || s.optimizations?.size || 0), 0);

    return {
      totalSnapshots: this.snapshots.size,
      memoryUsage: this.calculateMemoryUsage(),
      storageUsage: totalSize,
      compressionRatio: totalSize > 0 ? compressedSize / totalSize : 1,
      oldestSnapshot: Math.min(...snapshots.map(s => s.timestamp)),
      averageSnapshotSize: snapshots.length > 0 ? totalSize / snapshots.length : 0,
    };
  }

  // 工具方法
  private generateId(): string {
    return Date.now().toString(36) + Math.random().toString(36).substr(2);
  }

  private calculateSnapshotSize(snapshot: HistorySnapshot): number {
    return JSON.stringify(snapshot).length * 2; // 估算字节大小
  }

  private calculateDeltaSize(delta: HistoryDelta): number {
    return JSON.stringify(delta).length * 2;
  }

  private calculateMemoryUsage(): number {
    return Array.from(this.snapshots.values()).reduce((total, snapshot) => {
      return total + this.calculateSnapshotSize(snapshot);
    }, 0);
  }

  private deepClone<T>(obj: T): T {
    return JSON.parse(JSON.stringify(obj));
  }

  private clearRedoStack() {
    // 清理重做栈中的快照
    for (const snapshotId of this.redoStack) {
      if (!this.undoStack.includes(snapshotId)) {
        this.snapshots.delete(snapshotId);
        this.persistenceManager.removeSnapshot(snapshotId);
      }
    }
    this.redoStack = [];
  }

  private async loadPersistedHistory() {
    if (this.options.enablePersistence) {
      const persistedSnapshots = await this.persistenceManager.loadAllSnapshots();
      for (const snapshot of persistedSnapshots) {
        this.snapshots.set(snapshot.id, snapshot);
      }
    }
  }
}
```

### 增量计算器

```typescript
// packages/excalidraw/history/DeltaCalculator.ts
export class DeltaCalculator {
  // 计算两个快照的相似度
  calculateSimilarity(
    snapshot1: HistorySnapshot,
    snapshot2: HistorySnapshot
  ): number {
    const elements1 = snapshot1.elements;
    const elements2 = snapshot2.elements;

    // 基于元素数量的初步相似度
    const countSimilarity = Math.min(elements1.length, elements2.length) /
                           Math.max(elements1.length, elements2.length);

    if (countSimilarity < 0.5) {
      return countSimilarity;
    }

    // 基于元素ID的匹配度
    const ids1 = new Set(elements1.map(el => el.id));
    const ids2 = new Set(elements2.map(el => el.id));
    const intersection = new Set([...ids1].filter(id => ids2.has(id)));

    const idSimilarity = intersection.size / Math.max(ids1.size, ids2.size);

    // 基于应用状态的相似度
    const stateSimilarity = this.calculateAppStateSimilarity(
      snapshot1.appState,
      snapshot2.appState
    );

    // 加权平均
    return (countSimilarity * 0.3 + idSimilarity * 0.5 + stateSimilarity * 0.2);
  }

  // 计算增量
  calculateDelta(
    baseSnapshot: HistorySnapshot,
    targetSnapshot: HistorySnapshot
  ): HistoryDelta {
    const operations: DeltaOperation[] = [];

    // 计算元素变化
    const elementOps = this.calculateElementDelta(
      baseSnapshot.elements,
      targetSnapshot.elements
    );
    operations.push(...elementOps);

    // 计算状态变化
    const stateOps = this.calculateAppStateDelta(
      baseSnapshot.appState,
      targetSnapshot.appState
    );
    operations.push(...stateOps);

    return {
      baseSnapshotId: baseSnapshot.id,
      operations,
      timestamp: targetSnapshot.timestamp,
    };
  }

  // 应用增量
  applyDelta(
    baseSnapshot: HistorySnapshot,
    delta: HistoryDelta
  ): HistorySnapshot {
    const elements = [...baseSnapshot.elements];
    let appState = { ...baseSnapshot.appState };

    for (const operation of delta.operations) {
      switch (operation.type) {
        case 'add':
          if (operation.data) {
            elements.push(operation.data as ExcalidrawElement);
          }
          break;

        case 'remove':
          if (operation.elementId) {
            const index = elements.findIndex(el => el.id === operation.elementId);
            if (index !== -1) {
              elements.splice(index, 1);
            }
          }
          break;

        case 'modify':
          if (operation.elementId && operation.data) {
            const index = elements.findIndex(el => el.id === operation.elementId);
            if (index !== -1) {
              elements[index] = { ...elements[index], ...operation.data };
            }
          }
          break;

        case 'move':
          if (operation.path && operation.newValue !== undefined) {
            appState = this.setDeepValue(appState, operation.path, operation.newValue);
          }
          break;
      }
    }

    return {
      ...baseSnapshot,
      elements,
      appState,
      timestamp: delta.timestamp,
    };
  }

  // 计算元素增量
  private calculateElementDelta(
    baseElements: readonly ExcalidrawElement[],
    targetElements: readonly ExcalidrawElement[]
  ): DeltaOperation[] {
    const operations: DeltaOperation[] = [];
    const baseMap = new Map(baseElements.map(el => [el.id, el]));
    const targetMap = new Map(targetElements.map(el => [el.id, el]));

    // 检测新增元素
    for (const [id, element] of targetMap) {
      if (!baseMap.has(id)) {
        operations.push({
          type: 'add',
          elementId: id,
          data: element,
        });
      }
    }

    // 检测删除元素
    for (const [id] of baseMap) {
      if (!targetMap.has(id)) {
        operations.push({
          type: 'remove',
          elementId: id,
        });
      }
    }

    // 检测修改元素
    for (const [id, targetElement] of targetMap) {
      const baseElement = baseMap.get(id);
      if (baseElement) {
        const changes = this.calculateElementChanges(baseElement, targetElement);
        if (Object.keys(changes).length > 0) {
          operations.push({
            type: 'modify',
            elementId: id,
            data: changes,
          });
        }
      }
    }

    return operations;
  }

  // 计算单个元素的变化
  private calculateElementChanges(
    baseElement: ExcalidrawElement,
    targetElement: ExcalidrawElement
  ): Partial<ExcalidrawElement> {
    const changes: Partial<ExcalidrawElement> = {};

    // 检查基础属性
    const checkProps: (keyof ExcalidrawElement)[] = [
      'x', 'y', 'width', 'height', 'angle',
      'strokeColor', 'backgroundColor', 'fillStyle',
      'strokeWidth', 'roughness', 'opacity',
    ];

    for (const prop of checkProps) {
      if (baseElement[prop] !== targetElement[prop]) {
        (changes as any)[prop] = targetElement[prop];
      }
    }

    // 检查复杂属性
    if (baseElement.type === 'text' && targetElement.type === 'text') {
      if (baseElement.text !== targetElement.text) {
        (changes as any).text = targetElement.text;
      }
    }

    if ('points' in baseElement && 'points' in targetElement) {
      if (!this.arraysEqual(baseElement.points, targetElement.points)) {
        (changes as any).points = targetElement.points;
      }
    }

    return changes;
  }

  // 计算应用状态增量
  private calculateAppStateDelta(
    baseState: AppState,
    targetState: AppState
  ): DeltaOperation[] {
    const operations: DeltaOperation[] = [];

    // 检查关键状态字段
    const checkFields: (keyof AppState)[] = [
      'selectedElementIds',
      'zoom',
      'scrollX',
      'scrollY',
      'currentItemStrokeColor',
      'currentItemBackgroundColor',
      'viewBackgroundColor',
      'gridSize',
    ];

    for (const field of checkFields) {
      if (!this.deepEqual(baseState[field], targetState[field])) {
        operations.push({
          type: 'move',
          path: `appState.${field}`,
          oldValue: baseState[field],
          newValue: targetState[field],
        });
      }
    }

    return operations;
  }

  // 计算应用状态相似度
  private calculateAppStateSimilarity(
    state1: AppState,
    state2: AppState
  ): number {
    const keyFields: (keyof AppState)[] = [
      'currentItemStrokeColor',
      'currentItemBackgroundColor',
      'currentItemFillStyle',
      'currentItemStrokeWidth',
      'currentItemRoughness',
    ];

    let matches = 0;
    for (const field of keyFields) {
      if (this.deepEqual(state1[field], state2[field])) {
        matches++;
      }
    }

    return matches / keyFields.length;
  }

  // 工具方法
  private deepEqual(a: any, b: any): boolean {
    if (a === b) return true;
    if (a == null || b == null) return false;
    if (Array.isArray(a) && Array.isArray(b)) {
      return this.arraysEqual(a, b);
    }
    if (typeof a === 'object' && typeof b === 'object') {
      const keysA = Object.keys(a);
      const keysB = Object.keys(b);
      if (keysA.length !== keysB.length) return false;
      return keysA.every(key => this.deepEqual(a[key], b[key]));
    }
    return false;
  }

  private arraysEqual(a: any[], b: any[]): boolean {
    if (a.length !== b.length) return false;
    return a.every((val, index) => this.deepEqual(val, b[index]));
  }

  private setDeepValue(obj: any, path: string, value: any): any {
    const keys = path.split('.');
    const result = { ...obj };
    let current = result;

    for (let i = 0; i < keys.length - 1; i++) {
      const key = keys[i];
      current[key] = { ...current[key] };
      current = current[key];
    }

    current[keys[keys.length - 1]] = value;
    return result;
  }
}
```

### 快照压缩器

```typescript
// packages/excalidraw/history/SnapshotCompressor.ts
export class SnapshotCompressor {
  // 压缩快照
  async compress(snapshot: HistorySnapshot): Promise<CompressedSnapshot> {
    // 分离数据
    const { elements, appState, files, ...metadata } = snapshot;

    // 压缩元素数据
    const compressedElements = await this.compressElements(elements);

    // 压缩应用状态
    const compressedAppState = await this.compressAppState(appState);

    // 压缩文件数据
    const compressedFiles = files ? await this.compressFiles(files) : undefined;

    return {
      ...metadata,
      compressedData: {
        elements: compressedElements,
        appState: compressedAppState,
        files: compressedFiles,
      },
      originalSize: this.calculateSize(snapshot),
      compressedSize: 0, // 将在存储时计算
    };
  }

  // 解压快照
  async decompress(compressedSnapshot: CompressedSnapshot | HistorySnapshot): Promise<HistorySnapshot> {
    if (!('compressedData' in compressedSnapshot)) {
      return compressedSnapshot as HistorySnapshot;
    }

    const { compressedData, ...metadata } = compressedSnapshot;

    // 解压各部分数据
    const elements = await this.decompressElements(compressedData.elements);
    const appState = await this.decompressAppState(compressedData.appState);
    const files = compressedData.files ?
      await this.decompressFiles(compressedData.files) : undefined;

    return {
      ...metadata,
      elements,
      appState,
      files,
    } as HistorySnapshot;
  }

  // 压缩元素数组
  private async compressElements(elements: readonly ExcalidrawElement[]): Promise<CompressedData> {
    // 1. 提取公共属性
    const commonProps = this.extractCommonProperties(elements);

    // 2. 创建元素字典
    const elementDict = this.createElementDictionary(elements);

    // 3. 压缩几何数据
    const compressedGeometry = this.compressGeometry(elements);

    // 4. LZ4压缩
    const jsonData = JSON.stringify({
      commonProps,
      elementDict,
      compressedGeometry,
    });

    const compressed = await this.lz4Compress(jsonData);

    return {
      type: 'elements',
      data: compressed,
      originalSize: jsonData.length * 2,
      compressedSize: compressed.length,
    };
  }

  // 提取公共属性
  private extractCommonProperties(elements: readonly ExcalidrawElement[]): Record<string, any> {
    if (elements.length === 0) return {};

    const common: Record<string, any> = {};
    const firstElement = elements[0];

    const checkProps = ['strokeColor', 'backgroundColor', 'fillStyle', 'strokeWidth', 'roughness'];

    for (const prop of checkProps) {
      const value = (firstElement as any)[prop];
      if (elements.every(el => (el as any)[prop] === value)) {
        common[prop] = value;
      }
    }

    return common;
  }

  // 创建元素字典
  private createElementDictionary(elements: readonly ExcalidrawElement[]): Record<string, any> {
    // 按类型分组
    const byType: Record<string, ExcalidrawElement[]> = {};
    elements.forEach(el => {
      if (!byType[el.type]) byType[el.type] = [];
      byType[el.type].push(el);
    });

    // 为每种类型创建模板
    const templates: Record<string, any> = {};
    for (const [type, typeElements] of Object.entries(byType)) {
      if (typeElements.length > 1) {
        templates[type] = this.createTypeTemplate(typeElements);
      }
    }

    return templates;
  }

  // 压缩几何数据
  private compressGeometry(elements: readonly ExcalidrawElement[]): CompressedGeometry {
    const positions: number[] = [];
    const sizes: number[] = [];
    const angles: number[] = [];

    elements.forEach(el => {
      positions.push(el.x, el.y);
      sizes.push(el.width, el.height);
      angles.push(el.angle);
    });

    // 量化坐标（减少精度但保持视觉效果）
    const quantizedPositions = this.quantizeArray(positions, 0.1);
    const quantizedSizes = this.quantizeArray(sizes, 0.1);
    const quantizedAngles = this.quantizeArray(angles, 0.001);

    return {
      positions: quantizedPositions,
      sizes: quantizedSizes,
      angles: quantizedAngles,
    };
  }

  // 数组量化
  private quantizeArray(array: number[], quantum: number): number[] {
    return array.map(val => Math.round(val / quantum) * quantum);
  }

  // LZ4压缩（简化实现）
  private async lz4Compress(data: string): Promise<Uint8Array> {
    // 这里应该使用真正的LZ4压缩库
    // 为了示例，我们使用简单的编码
    const encoder = new TextEncoder();
    const bytes = encoder.encode(data);

    // 简单的重复字符压缩
    const compressed = this.simpleCompress(bytes);

    return compressed;
  }

  // 简单压缩算法
  private simpleCompress(data: Uint8Array): Uint8Array {
    const result: number[] = [];
    let i = 0;

    while (i < data.length) {
      const current = data[i];
      let count = 1;

      // 计算重复次数
      while (i + count < data.length && data[i + count] === current && count < 255) {
        count++;
      }

      if (count > 3) {
        // 压缩重复
        result.push(0xFF, current, count);
      } else {
        // 直接存储
        for (let j = 0; j < count; j++) {
          result.push(current);
        }
      }

      i += count;
    }

    return new Uint8Array(result);
  }

  // 解压
  private async lz4Decompress(compressed: Uint8Array): Promise<string> {
    const decompressed = this.simpleDecompress(compressed);
    const decoder = new TextDecoder();
    return decoder.decode(decompressed);
  }

  // 简单解压算法
  private simpleDecompress(compressed: Uint8Array): Uint8Array {
    const result: number[] = [];
    let i = 0;

    while (i < compressed.length) {
      if (compressed[i] === 0xFF && i + 2 < compressed.length) {
        // 解压重复
        const value = compressed[i + 1];
        const count = compressed[i + 2];

        for (let j = 0; j < count; j++) {
          result.push(value);
        }

        i += 3;
      } else {
        // 直接复制
        result.push(compressed[i]);
        i++;
      }
    }

    return new Uint8Array(result);
  }

  private calculateSize(obj: any): number {
    return JSON.stringify(obj).length * 2;
  }

  private createTypeTemplate(elements: ExcalidrawElement[]): any {
    // 创建类型模板（简化实现）
    return {
      count: elements.length,
      commonProps: this.extractCommonProperties(elements),
    };
  }

  private async compressAppState(appState: AppState): Promise<CompressedData> {
    // 简化的状态压缩
    const jsonData = JSON.stringify(appState);
    const compressed = await this.lz4Compress(jsonData);

    return {
      type: 'appState',
      data: compressed,
      originalSize: jsonData.length * 2,
      compressedSize: compressed.length,
    };
  }

  private async decompressAppState(compressed: CompressedData): Promise<AppState> {
    const jsonData = await this.lz4Decompress(compressed.data);
    return JSON.parse(jsonData);
  }

  private async compressFiles(files: BinaryFiles): Promise<CompressedData> {
    // 文件压缩（简化实现）
    const jsonData = JSON.stringify(files);
    const compressed = await this.lz4Compress(jsonData);

    return {
      type: 'files',
      data: compressed,
      originalSize: jsonData.length * 2,
      compressedSize: compressed.length,
    };
  }

  private async decompressFiles(compressed: CompressedData): Promise<BinaryFiles> {
    const jsonData = await this.lz4Decompress(compressed.data);
    return JSON.parse(jsonData);
  }

  private async decompressElements(compressed: CompressedData): Promise<readonly ExcalidrawElement[]> {
    const jsonData = await this.lz4Decompress(compressed.data);
    const { commonProps, elementDict, compressedGeometry } = JSON.parse(jsonData);

    // 重构元素数组（简化实现）
    return this.reconstructElements(commonProps, elementDict, compressedGeometry);
  }

  private reconstructElements(
    commonProps: Record<string, any>,
    elementDict: Record<string, any>,
    geometry: CompressedGeometry
  ): ExcalidrawElement[] {
    // 简化的重构实现
    const elements: ExcalidrawElement[] = [];
    const posCount = geometry.positions.length / 2;

    for (let i = 0; i < posCount; i++) {
      const element = {
        id: `element-${i}`,
        type: 'rectangle' as const,
        x: geometry.positions[i * 2],
        y: geometry.positions[i * 2 + 1],
        width: geometry.sizes[i * 2],
        height: geometry.sizes[i * 2 + 1],
        angle: geometry.angles[i],
        ...commonProps,
      } as ExcalidrawElement;

      elements.push(element);
    }

    return elements;
  }
}

// 接口定义
interface CompressedSnapshot {
  id: string;
  timestamp: number;
  metadata: HistorySnapshot['metadata'];
  compressedData: {
    elements: CompressedData;
    appState: CompressedData;
    files?: CompressedData;
  };
  originalSize: number;
  compressedSize: number;
}

interface CompressedData {
  type: string;
  data: Uint8Array;
  originalSize: number;
  compressedSize: number;
}

interface CompressedGeometry {
  positions: number[];
  sizes: number[];
  angles: number[];
}
```

## 思考题

1. **历史压缩的效果如何评估？** 压缩比和性能的平衡点在哪里？

2. **协作环境下的历史同步如何处理？** 多用户历史的合并策略？

3. **大文件的历史管理？** 包含大量图片时的优化方案？

4. **历史数据的安全性？** 如何防止历史数据泄露敏感信息？

5. **跨会话的历史恢复？** 浏览器崩溃后的历史恢复策略？

## 实践练习

### 练习 1：实现自定义压缩算法
设计针对Excalidraw数据特点的压缩算法：
- 几何数据的特殊编码
- 颜色调色板压缩
- 重复模式识别

### 练习 2：实现历史分析工具
创建历史数据分析功能：
- 操作频率统计
- 用户行为模式分析
- 性能瓶颈识别

### 练习 3：实现协作历史同步
设计多用户历史同步机制：
- 冲突检测算法
- 历史分支管理
- 操作时序同步

## 总结

Excalidraw的历史管理系统展现了现代编辑器的技术水准：

1. **智能存储策略**：根据数据特点选择最优的存储方式
2. **高效压缩算法**：专门针对图形数据的压缩优化
3. **增量式管理**：通过差异计算减少存储开销
4. **内存优化**：多层次的缓存和清理策略
5. **扩展性设计**：支持协作和云端同步的架构

这套系统为构建企业级编辑器提供了可靠的历史管理基础。

## 下一步

下一章将探讨Excalidraw的高级功能实现，包括协作系统、插件架构、以及导入导出机制等。