# Chapter 6.1: 实时协作系统实现

## 概述

实时协作是现代在线工具的核心功能之一。Excalidraw的协作系统支持多用户同时编辑，实时同步操作，解决编辑冲突，并提供流畅的协作体验。本章将深入解析这套协作系统的技术架构和实现细节。

## 协作系统架构

### 整体架构设计

```
协作系统分层架构
├── 客户端层 (Client Layer)
│   ├── CollabManager     # 协作管理器
│   ├── OperationBuffer   # 操作缓冲区
│   ├── ConflictResolver  # 冲突解决器
│   └── PresenceTracker   # 在线状态追踪
├── 传输层 (Transport Layer)
│   ├── WebSocket         # 实时通信
│   ├── MessageQueue      # 消息队列
│   ├── Retry机制         # 重试机制
│   └── 连接管理          # 连接状态管理
├── 同步层 (Sync Layer)
│   ├── OperationalTransform # 操作变换
│   ├── VectorClock       # 向量时钟
│   ├── CRDT算法          # 无冲突复制数据类型
│   └── StateSync         # 状态同步
└── 服务端层 (Server Layer)
    ├── RoomManager       # 房间管理
    ├── UserManager       # 用户管理
    ├── MessageBroker     # 消息代理
    └── PersistenceLayer  # 持久化层
```

### 核心数据结构

```typescript
// 协作相关的类型定义
export interface CollaboratorPointer {
  x: number;
  y: number;
  tool: "pointer" | ToolType;
}

export interface Collaborator {
  socketId: string;
  id?: string;
  username?: string;
  avatarUrl?: string;
  color?: {
    background: string;
    stroke: string;
  };
  pointer?: CollaboratorPointer;
  button?: "down" | "up";
  selectedElementIds?: AppState["selectedElementIds"];
  userState?: UserIdleState;
  isCurrentUser?: boolean;
  isInCall?: boolean;
}

// 协作操作类型
export interface CollabOperation {
  id: string;
  type: 'element-add' | 'element-update' | 'element-delete' | 'state-update';
  timestamp: number;
  authorId: string;
  elementId?: string;
  data?: any;
  dependencies?: string[];  // 依赖的操作ID
  vectorClock?: VectorClock;
}

// 向量时钟
export interface VectorClock {
  [userId: string]: number;
}

// 协作房间状态
export interface CollabRoomState {
  roomId: string;
  elements: readonly ExcalidrawElement[];
  collaborators: Map<string, Collaborator>;
  operationLog: CollabOperation[];
  vectorClock: VectorClock;
  version: number;
}
```

## 协作管理器实现

### CollabManager核心类

```typescript
// packages/excalidraw/collab/CollabManager.ts
export class CollabManager {
  private ws: WebSocket | null = null;
  private roomId: string;
  private userId: string;
  private collaborators = new Map<string, Collaborator>();
  private operationBuffer: OperationBuffer;
  private conflictResolver: ConflictResolver;
  private presenceTracker: PresenceTracker;

  // 状态管理
  private isConnected = false;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;

  // 回调函数
  private onElementsChange?: (elements: readonly ExcalidrawElement[]) => void;
  private onCollaboratorsChange?: (collaborators: Map<string, Collaborator>) => void;
  private onConnectionChange?: (connected: boolean) => void;

  constructor(roomId: string, userId: string) {
    this.roomId = roomId;
    this.userId = userId;
    this.operationBuffer = new OperationBuffer();
    this.conflictResolver = new ConflictResolver();
    this.presenceTracker = new PresenceTracker();

    this.setupEventHandlers();
  }

  // 连接到协作服务器
  async connect(serverUrl: string): Promise<boolean> {
    try {
      this.ws = new WebSocket(`${serverUrl}/collab/${this.roomId}`);

      this.ws.onopen = this.handleOpen.bind(this);
      this.ws.onmessage = this.handleMessage.bind(this);
      this.ws.onclose = this.handleClose.bind(this);
      this.ws.onerror = this.handleError.bind(this);

      return new Promise((resolve) => {
        const timeout = setTimeout(() => resolve(false), 5000);

        this.ws!.onopen = () => {
          clearTimeout(timeout);
          this.handleOpen();
          resolve(true);
        };
      });
    } catch (error) {
      console.error('Failed to connect to collaboration server:', error);
      return false;
    }
  }

  // 断开连接
  disconnect() {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
    this.isConnected = false;
    this.onConnectionChange?.(false);
  }

  // 发送元素操作
  sendElementOperation(
    type: CollabOperation['type'],
    elementId: string,
    data: any
  ) {
    const operation: CollabOperation = {
      id: this.generateOperationId(),
      type,
      timestamp: Date.now(),
      authorId: this.userId,
      elementId,
      data,
      vectorClock: this.presenceTracker.getVectorClock(),
    };

    // 添加到缓冲区
    this.operationBuffer.add(operation);

    // 发送到服务器
    this.sendMessage({
      type: 'operation',
      operation,
    });
  }

  // 发送状态更新
  sendStateUpdate(stateUpdates: Partial<AppState>) {
    const operation: CollabOperation = {
      id: this.generateOperationId(),
      type: 'state-update',
      timestamp: Date.now(),
      authorId: this.userId,
      data: stateUpdates,
      vectorClock: this.presenceTracker.getVectorClock(),
    };

    this.sendMessage({
      type: 'state-update',
      operation,
    });
  }

  // 发送指针位置
  sendPointerUpdate(pointer: CollaboratorPointer) {
    this.sendMessage({
      type: 'pointer-update',
      pointer,
    });
  }

  // 处理连接建立
  private handleOpen() {
    console.log('Connected to collaboration server');
    this.isConnected = true;
    this.reconnectAttempts = 0;

    // 发送初始化消息
    this.sendMessage({
      type: 'join-room',
      roomId: this.roomId,
      userId: this.userId,
    });

    this.onConnectionChange?.(true);
  }

  // 处理消息接收
  private handleMessage(event: MessageEvent) {
    try {
      const message = JSON.parse(event.data);

      switch (message.type) {
        case 'room-joined':
          this.handleRoomJoined(message);
          break;
        case 'operation':
          this.handleOperationReceived(message.operation);
          break;
        case 'collaborator-joined':
          this.handleCollaboratorJoined(message.collaborator);
          break;
        case 'collaborator-left':
          this.handleCollaboratorLeft(message.userId);
          break;
        case 'pointer-update':
          this.handlePointerUpdate(message.userId, message.pointer);
          break;
        case 'state-sync':
          this.handleStateSync(message.state);
          break;
        case 'error':
          this.handleServerError(message.error);
          break;
      }
    } catch (error) {
      console.error('Failed to parse collaboration message:', error);
    }
  }

  // 处理连接关闭
  private handleClose() {
    console.log('Disconnected from collaboration server');
    this.isConnected = false;
    this.onConnectionChange?.(false);

    // 尝试重连
    this.attemptReconnect();
  }

  // 处理连接错误
  private handleError(error: Event) {
    console.error('Collaboration WebSocket error:', error);
  }

  // 尝试重连
  private attemptReconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnection attempts reached');
      return;
    }

    this.reconnectAttempts++;
    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);

    setTimeout(() => {
      console.log(`Attempting to reconnect (${this.reconnectAttempts}/${this.maxReconnectAttempts})`);
      this.connect(this.getServerUrl());
    }, delay);
  }

  // 处理房间加入
  private handleRoomJoined(message: any) {
    const { collaborators, state } = message;

    // 更新协作者列表
    this.collaborators.clear();
    collaborators.forEach((collab: Collaborator) => {
      this.collaborators.set(collab.socketId, collab);
    });

    // 同步初始状态
    if (state) {
      this.handleStateSync(state);
    }

    this.onCollaboratorsChange?.(new Map(this.collaborators));
  }

  // 处理操作接收
  private handleOperationReceived(operation: CollabOperation) {
    // 检查操作是否来自当前用户
    if (operation.authorId === this.userId) {
      // 确认操作，从缓冲区移除
      this.operationBuffer.confirm(operation.id);
      return;
    }

    // 应用操作变换
    const transformedOperation = this.conflictResolver.transformOperation(
      operation,
      this.operationBuffer.getPendingOperations()
    );

    // 应用操作
    this.applyOperation(transformedOperation);

    // 更新向量时钟
    if (operation.vectorClock) {
      this.presenceTracker.updateVectorClock(operation.vectorClock);
    }
  }

  // 应用操作
  private applyOperation(operation: CollabOperation) {
    // 这里需要根据操作类型应用到本地状态
    // 具体实现取决于应用的状态管理方式
    console.log('Applying operation:', operation);

    // 触发元素变化回调
    // this.onElementsChange?.(updatedElements);
  }

  // 处理协作者加入
  private handleCollaboratorJoined(collaborator: Collaborator) {
    this.collaborators.set(collaborator.socketId, collaborator);
    this.onCollaboratorsChange?.(new Map(this.collaborators));
  }

  // 处理协作者离开
  private handleCollaboratorLeft(userId: string) {
    for (const [socketId, collaborator] of this.collaborators) {
      if (collaborator.id === userId) {
        this.collaborators.delete(socketId);
        break;
      }
    }
    this.onCollaboratorsChange?.(new Map(this.collaborators));
  }

  // 处理指针更新
  private handlePointerUpdate(userId: string, pointer: CollaboratorPointer) {
    for (const [socketId, collaborator] of this.collaborators) {
      if (collaborator.id === userId) {
        collaborator.pointer = pointer;
        break;
      }
    }
    this.onCollaboratorsChange?.(new Map(this.collaborators));
  }

  // 处理状态同步
  private handleStateSync(state: any) {
    // 同步完整状态
    console.log('Syncing state:', state);
    // 这里需要合并远程状态到本地
  }

  // 发送消息到服务器
  private sendMessage(message: any) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    } else {
      console.warn('WebSocket not connected, message queued');
      // 这里可以添加消息队列机制
    }
  }

  // 设置事件处理器
  private setupEventHandlers() {
    // 心跳保活
    setInterval(() => {
      if (this.isConnected) {
        this.sendMessage({ type: 'ping' });
      }
    }, 30000);
  }

  // 生成操作ID
  private generateOperationId(): string {
    return `${this.userId}-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  private getServerUrl(): string {
    // 返回服务器URL，这里应该从配置获取
    return 'wss://collab.excalidraw.com';
  }

  // 公共API
  setOnElementsChange(callback: (elements: readonly ExcalidrawElement[]) => void) {
    this.onElementsChange = callback;
  }

  setOnCollaboratorsChange(callback: (collaborators: Map<string, Collaborator>) => void) {
    this.onCollaboratorsChange = callback;
  }

  setOnConnectionChange(callback: (connected: boolean) => void) {
    this.onConnectionChange = callback;
  }

  getCollaborators(): Map<string, Collaborator> {
    return new Map(this.collaborators);
  }

  isConnectedToServer(): boolean {
    return this.isConnected;
  }
}
```

### 操作变换算法

```typescript
// packages/excalidraw/collab/OperationalTransform.ts
export class ConflictResolver {
  // 操作变换主函数
  transformOperation(
    serverOp: CollabOperation,
    clientOps: CollabOperation[]
  ): CollabOperation {
    let transformedOp = serverOp;

    // 按时间戳排序客户端操作
    const sortedClientOps = clientOps.sort((a, b) => a.timestamp - b.timestamp);

    // 对每个客户端操作进行变换
    for (const clientOp of sortedClientOps) {
      transformedOp = this.transformPair(transformedOp, clientOp);
    }

    return transformedOp;
  }

  // 变换一对操作
  private transformPair(
    op1: CollabOperation,
    op2: CollabOperation
  ): CollabOperation {
    // 如果操作针对不同元素，无需变换
    if (op1.elementId !== op2.elementId) {
      return op1;
    }

    // 根据操作类型进行具体变换
    switch (`${op1.type}-${op2.type}`) {
      case 'element-update-element-update':
        return this.transformUpdateUpdate(op1, op2);

      case 'element-update-element-delete':
        return this.transformUpdateDelete(op1, op2);

      case 'element-delete-element-update':
        return this.transformDeleteUpdate(op1, op2);

      case 'element-delete-element-delete':
        return this.transformDeleteDelete(op1, op2);

      case 'element-add-element-add':
        return this.transformAddAdd(op1, op2);

      default:
        return op1;
    }
  }

  // 变换两个更新操作
  private transformUpdateUpdate(
    op1: CollabOperation,
    op2: CollabOperation
  ): CollabOperation {
    const result = { ...op1 };

    // 合并属性更新
    if (op1.data && op2.data) {
      // 检查冲突的属性
      const conflicts = this.findPropertyConflicts(op1.data, op2.data);

      if (conflicts.length > 0) {
        // 解决冲突：使用时间戳较晚的操作
        const winningOp = op1.timestamp > op2.timestamp ? op1 : op2;

        for (const conflict of conflicts) {
          if (winningOp === op2) {
            // 移除冲突属性，让服务器操作获胜
            delete result.data[conflict];
          }
        }
      }

      // 合并非冲突属性
      result.data = { ...op1.data };
    }

    return result;
  }

  // 变换更新-删除操作
  private transformUpdateDelete(
    updateOp: CollabOperation,
    deleteOp: CollabOperation
  ): CollabOperation {
    // 如果元素被删除，更新操作变为无效
    return {
      ...updateOp,
      type: 'noop' as any,
    };
  }

  // 变换删除-更新操作
  private transformDeleteUpdate(
    deleteOp: CollabOperation,
    updateOp: CollabOperation
  ): CollabOperation {
    // 删除操作优先级高于更新操作
    return deleteOp;
  }

  // 变换两个删除操作
  private transformDeleteDelete(
    op1: CollabOperation,
    op2: CollabOperation
  ): CollabOperation {
    // 两个删除操作，结果相同
    return op1;
  }

  // 变换两个添加操作
  private transformAddAdd(
    op1: CollabOperation,
    op2: CollabOperation
  ): CollabOperation {
    // ID冲突处理
    if (op1.elementId === op2.elementId) {
      // 使用时间戳和作者ID决定优先级
      const priority1 = `${op1.timestamp}-${op1.authorId}`;
      const priority2 = `${op2.timestamp}-${op2.authorId}`;

      if (priority1 < priority2) {
        // op2优先，op1需要生成新ID
        return {
          ...op1,
          elementId: this.generateNewElementId(op1.elementId!),
        };
      }
    }

    return op1;
  }

  // 查找属性冲突
  private findPropertyConflicts(data1: any, data2: any): string[] {
    const conflicts: string[] = [];

    for (const key in data1) {
      if (key in data2 && data1[key] !== data2[key]) {
        conflicts.push(key);
      }
    }

    return conflicts;
  }

  // 生成新的元素ID
  private generateNewElementId(originalId: string): string {
    return `${originalId}-resolved-${Math.random().toString(36).substr(2, 9)}`;
  }

  // 基于向量时钟的冲突检测
  detectConflict(op1: CollabOperation, op2: CollabOperation): boolean {
    if (!op1.vectorClock || !op2.vectorClock) {
      return false;
    }

    // 检查向量时钟是否并发
    return this.areVectorClocksConcurrent(op1.vectorClock, op2.vectorClock);
  }

  // 检查向量时钟是否并发
  private areVectorClocksConcurrent(vc1: VectorClock, vc2: VectorClock): boolean {
    let vc1Greater = false;
    let vc2Greater = false;

    const allUsers = new Set([...Object.keys(vc1), ...Object.keys(vc2)]);

    for (const user of allUsers) {
      const val1 = vc1[user] || 0;
      const val2 = vc2[user] || 0;

      if (val1 > val2) {
        vc1Greater = true;
      } else if (val2 > val1) {
        vc2Greater = true;
      }
    }

    // 如果两个时钟都不完全小于对方，则并发
    return vc1Greater && vc2Greater;
  }
}
```

### 操作缓冲区

```typescript
// packages/excalidraw/collab/OperationBuffer.ts
export class OperationBuffer {
  private pendingOperations: Map<string, CollabOperation> = new Map();
  private confirmedOperations: Set<string> = new Set();
  private maxBufferSize = 1000;
  private cleanupInterval = 60000; // 1分钟

  constructor() {
    this.startCleanupTimer();
  }

  // 添加操作到缓冲区
  add(operation: CollabOperation) {
    this.pendingOperations.set(operation.id, operation);
    this.limitBufferSize();
  }

  // 确认操作
  confirm(operationId: string) {
    this.pendingOperations.delete(operationId);
    this.confirmedOperations.add(operationId);
  }

  // 获取待确认的操作
  getPendingOperations(): CollabOperation[] {
    return Array.from(this.pendingOperations.values()).sort(
      (a, b) => a.timestamp - b.timestamp
    );
  }

  // 检查操作是否已确认
  isConfirmed(operationId: string): boolean {
    return this.confirmedOperations.has(operationId);
  }

  // 检查操作是否待确认
  isPending(operationId: string): boolean {
    return this.pendingOperations.has(operationId);
  }

  // 清理过期操作
  cleanup() {
    const now = Date.now();
    const maxAge = 300000; // 5分钟

    // 清理过期的待确认操作
    for (const [id, operation] of this.pendingOperations) {
      if (now - operation.timestamp > maxAge) {
        this.pendingOperations.delete(id);
        console.warn(`Operation ${id} expired without confirmation`);
      }
    }

    // 清理过期的已确认操作记录
    const confirmedArray = Array.from(this.confirmedOperations);
    if (confirmedArray.length > this.maxBufferSize) {
      const toRemove = confirmedArray.slice(0, confirmedArray.length - this.maxBufferSize);
      toRemove.forEach(id => this.confirmedOperations.delete(id));
    }
  }

  // 限制缓冲区大小
  private limitBufferSize() {
    if (this.pendingOperations.size > this.maxBufferSize) {
      const operations = Array.from(this.pendingOperations.values())
        .sort((a, b) => a.timestamp - b.timestamp);

      const toRemove = operations.slice(0, operations.length - this.maxBufferSize);
      toRemove.forEach(op => {
        this.pendingOperations.delete(op.id);
        console.warn(`Operation ${op.id} removed from buffer due to size limit`);
      });
    }
  }

  // 启动清理定时器
  private startCleanupTimer() {
    setInterval(() => {
      this.cleanup();
    }, this.cleanupInterval);
  }

  // 获取缓冲区统计
  getStats() {
    return {
      pendingCount: this.pendingOperations.size,
      confirmedCount: this.confirmedOperations.size,
      oldestPending: this.getOldestPendingTimestamp(),
    };
  }

  private getOldestPendingTimestamp(): number | null {
    const operations = Array.from(this.pendingOperations.values());
    if (operations.length === 0) return null;

    return Math.min(...operations.map(op => op.timestamp));
  }
}
```

### 在线状态追踪

```typescript
// packages/excalidraw/collab/PresenceTracker.ts
export class PresenceTracker {
  private vectorClock: VectorClock = {};
  private userId: string;
  private presenceUpdateInterval = 1000; // 1秒
  private lastPresenceUpdate = 0;

  constructor(userId: string) {
    this.userId = userId;
    this.vectorClock[userId] = 0;
  }

  // 更新向量时钟
  updateVectorClock(remoteVectorClock: VectorClock) {
    for (const [userId, timestamp] of Object.entries(remoteVectorClock)) {
      this.vectorClock[userId] = Math.max(
        this.vectorClock[userId] || 0,
        timestamp
      );
    }
  }

  // 递增本地时钟
  incrementClock() {
    this.vectorClock[this.userId] = (this.vectorClock[this.userId] || 0) + 1;
  }

  // 获取当前向量时钟
  getVectorClock(): VectorClock {
    return { ...this.vectorClock };
  }

  // 检查是否应该发送在线状态更新
  shouldSendPresenceUpdate(): boolean {
    const now = Date.now();
    return now - this.lastPresenceUpdate >= this.presenceUpdateInterval;
  }

  // 标记已发送在线状态
  markPresenceSent() {
    this.lastPresenceUpdate = Date.now();
  }

  // 创建在线状态消息
  createPresenceMessage(
    pointer?: CollaboratorPointer,
    selectedElementIds?: AppState["selectedElementIds"]
  ) {
    return {
      type: 'presence-update',
      userId: this.userId,
      timestamp: Date.now(),
      vectorClock: this.getVectorClock(),
      data: {
        pointer,
        selectedElementIds,
        userState: this.getUserState(),
      },
    };
  }

  // 获取用户状态
  private getUserState() {
    return {
      isActive: document.visibilityState === 'visible',
      lastActivity: Date.now(),
    };
  }
}
```

## 实战示例：完整协作系统

```typescript
// 协作系统使用示例
class CollaborativeExcalidraw {
  private collabManager: CollabManager;
  private elements: readonly ExcalidrawElement[] = [];
  private appState: AppState;

  constructor(roomId: string, userId: string) {
    this.collabManager = new CollabManager(roomId, userId);
    this.setupCollaborationCallbacks();
  }

  // 设置协作回调
  private setupCollaborationCallbacks() {
    // 监听元素变化
    this.collabManager.setOnElementsChange((elements) => {
      this.elements = elements;
      this.rerenderCanvas();
    });

    // 监听协作者变化
    this.collabManager.setOnCollaboratorsChange((collaborators) => {
      this.updateCollaboratorsList(collaborators);
      this.renderCollaboratorPointers(collaborators);
    });

    // 监听连接状态变化
    this.collabManager.setOnConnectionChange((connected) => {
      this.updateConnectionStatus(connected);
    });
  }

  // 连接到协作服务器
  async connect(serverUrl: string) {
    const connected = await this.collabManager.connect(serverUrl);

    if (!connected) {
      this.showErrorMessage('Failed to connect to collaboration server');
    }
  }

  // 处理本地元素变化
  onElementsChange(newElements: readonly ExcalidrawElement[]) {
    const changes = this.calculateElementChanges(this.elements, newElements);

    // 发送变化到服务器
    changes.forEach(change => {
      this.collabManager.sendElementOperation(
        change.type,
        change.elementId,
        change.data
      );
    });

    // 更新本地状态
    this.elements = newElements;
  }

  // 处理应用状态变化
  onAppStateChange(newAppState: AppState) {
    // 只同步必要的状态
    const syncableState = this.extractSyncableState(newAppState);

    if (this.hasStateChanges(this.appState, syncableState)) {
      this.collabManager.sendStateUpdate(syncableState);
    }

    this.appState = newAppState;
  }

  // 处理指针移动
  onPointerMove(pointer: CollaboratorPointer) {
    this.collabManager.sendPointerUpdate(pointer);
  }

  // 计算元素变化
  private calculateElementChanges(
    oldElements: readonly ExcalidrawElement[],
    newElements: readonly ExcalidrawElement[]
  ): ElementChange[] {
    const changes: ElementChange[] = [];
    const oldMap = new Map(oldElements.map(el => [el.id, el]));
    const newMap = new Map(newElements.map(el => [el.id, el]));

    // 检测新增元素
    for (const [id, element] of newMap) {
      if (!oldMap.has(id)) {
        changes.push({
          type: 'element-add',
          elementId: id,
          data: element,
        });
      }
    }

    // 检测删除元素
    for (const [id] of oldMap) {
      if (!newMap.has(id)) {
        changes.push({
          type: 'element-delete',
          elementId: id,
          data: null,
        });
      }
    }

    // 检测修改元素
    for (const [id, newElement] of newMap) {
      const oldElement = oldMap.get(id);
      if (oldElement && !this.elementsEqual(oldElement, newElement)) {
        changes.push({
          type: 'element-update',
          elementId: id,
          data: this.calculateElementDiff(oldElement, newElement),
        });
      }
    }

    return changes;
  }

  // 提取可同步的状态
  private extractSyncableState(appState: AppState): Partial<AppState> {
    return {
      selectedElementIds: appState.selectedElementIds,
      currentItemStrokeColor: appState.currentItemStrokeColor,
      currentItemBackgroundColor: appState.currentItemBackgroundColor,
      currentItemFillStyle: appState.currentItemFillStyle,
      currentItemStrokeWidth: appState.currentItemStrokeWidth,
      currentItemRoughness: appState.currentItemRoughness,
      currentItemOpacity: appState.currentItemOpacity,
    };
  }

  // 检查状态变化
  private hasStateChanges(oldState: AppState, newState: Partial<AppState>): boolean {
    for (const [key, value] of Object.entries(newState)) {
      if (!this.deepEqual((oldState as any)[key], value)) {
        return true;
      }
    }
    return false;
  }

  // 更新协作者列表UI
  private updateCollaboratorsList(collaborators: Map<string, Collaborator>) {
    const collabList = document.getElementById('collaborators-list');
    if (!collabList) return;

    collabList.innerHTML = '';

    for (const [socketId, collaborator] of collaborators) {
      if (collaborator.isCurrentUser) continue;

      const collabElement = document.createElement('div');
      collabElement.className = 'collaborator-item';
      collabElement.innerHTML = `
        <div class="collaborator-avatar" style="background-color: ${collaborator.color?.background}">
          ${collaborator.username?.charAt(0) || '?'}
        </div>
        <span class="collaborator-name">${collaborator.username || 'Anonymous'}</span>
        ${collaborator.userState?.isActive ? '<span class="status-active">●</span>' : '<span class="status-idle">○</span>'}
      `;

      collabList.appendChild(collabElement);
    }
  }

  // 渲染协作者指针
  private renderCollaboratorPointers(collaborators: Map<string, Collaborator>) {
    const canvas = document.getElementById('collaborative-canvas') as HTMLCanvasElement;
    if (!canvas) return;

    const overlay = canvas.nextElementSibling as HTMLElement;
    if (!overlay) return;

    // 清除现有指针
    overlay.querySelectorAll('.collaborator-pointer').forEach(el => el.remove());

    // 绘制协作者指针
    for (const [socketId, collaborator] of collaborators) {
      if (collaborator.isCurrentUser || !collaborator.pointer) continue;

      const pointer = document.createElement('div');
      pointer.className = 'collaborator-pointer';
      pointer.style.cssText = `
        position: absolute;
        left: ${collaborator.pointer.x}px;
        top: ${collaborator.pointer.y}px;
        background-color: ${collaborator.color?.background};
        border: 2px solid ${collaborator.color?.stroke};
        width: 12px;
        height: 12px;
        border-radius: 50%;
        pointer-events: none;
        z-index: 1000;
        transform: translate(-50%, -50%);
      `;

      // 添加用户名标签
      const label = document.createElement('span');
      label.className = 'collaborator-label';
      label.textContent = collaborator.username || 'Anonymous';
      label.style.cssText = `
        position: absolute;
        top: -30px;
        left: 50%;
        transform: translateX(-50%);
        background: ${collaborator.color?.background};
        color: white;
        padding: 4px 8px;
        border-radius: 4px;
        font-size: 12px;
        white-space: nowrap;
      `;

      pointer.appendChild(label);
      overlay.appendChild(pointer);
    }
  }

  // 更新连接状态
  private updateConnectionStatus(connected: boolean) {
    const statusElement = document.getElementById('connection-status');
    if (!statusElement) return;

    statusElement.className = connected ? 'connected' : 'disconnected';
    statusElement.textContent = connected ? 'Connected' : 'Disconnected';

    if (!connected) {
      this.showErrorMessage('Connection lost. Trying to reconnect...');
    }
  }

  // 显示错误消息
  private showErrorMessage(message: string) {
    console.error(message);
    // 这里可以添加UI错误提示
  }

  // 工具方法
  private elementsEqual(a: ExcalidrawElement, b: ExcalidrawElement): boolean {
    return JSON.stringify(a) === JSON.stringify(b);
  }

  private calculateElementDiff(oldElement: ExcalidrawElement, newElement: ExcalidrawElement): Partial<ExcalidrawElement> {
    const diff: Partial<ExcalidrawElement> = {};

    for (const key in newElement) {
      if ((newElement as any)[key] !== (oldElement as any)[key]) {
        (diff as any)[key] = (newElement as any)[key];
      }
    }

    return diff;
  }

  private deepEqual(a: any, b: any): boolean {
    if (a === b) return true;
    if (typeof a !== typeof b) return false;
    if (typeof a === 'object' && a !== null && b !== null) {
      const keysA = Object.keys(a);
      const keysB = Object.keys(b);
      if (keysA.length !== keysB.length) return false;
      return keysA.every(key => this.deepEqual(a[key], b[key]));
    }
    return false;
  }

  private rerenderCanvas() {
    // 触发canvas重绘
    window.dispatchEvent(new CustomEvent('excalidraw-rerender'));
  }
}

// 接口定义
interface ElementChange {
  type: 'element-add' | 'element-update' | 'element-delete';
  elementId: string;
  data: any;
}

// 使用示例
const roomId = 'drawing-room-123';
const userId = 'user-456';

const collaborativeApp = new CollaborativeExcalidraw(roomId, userId);

// 连接到服务器
collaborativeApp.connect('wss://collab.excalidraw.com');

// 集成到Excalidraw组件
const ExcalidrawWithCollab = () => {
  const [elements, setElements] = useState<readonly ExcalidrawElement[]>([]);
  const [appState, setAppState] = useState<AppState>(getDefaultAppState());

  useEffect(() => {
    // 监听元素变化
    collaborativeApp.onElementsChange(elements);
  }, [elements]);

  useEffect(() => {
    // 监听状态变化
    collaborativeApp.onAppStateChange(appState);
  }, [appState]);

  return (
    <div className="collaborative-excalidraw">
      <div id="connection-status" className="connection-status"></div>
      <div id="collaborators-list" className="collaborators-list"></div>

      <Excalidraw
        initialData={{
          elements,
          appState,
        }}
        onChange={(elements, appState) => {
          setElements(elements);
          setAppState(appState);
        }}
        onPointerUpdate={(payload) => {
          if (payload.pointer) {
            collaborativeApp.onPointerMove(payload.pointer);
          }
        }}
      />

      <div className="collaborator-overlay"></div>
    </div>
  );
};
```

## 思考题

1. **大规模协作如何优化？** 支持100+用户同时协作的技术挑战？

2. **网络分区如何处理？** 客户端离线后重连的数据同步策略？

3. **协作安全如何保证？** 防止恶意用户破坏共享画板的机制？

4. **性能瓶颈在哪里？** 实时协作的主要性能制约因素？

5. **历史版本如何协作？** 协作环境下的版本控制和分支管理？

## 总结

Excalidraw的协作系统展现了现代实时协作技术的精髓：

1. **操作变换算法**：解决并发编辑冲突的核心技术
2. **分层架构设计**：清晰的模块划分和职责分离
3. **状态同步机制**：高效的增量同步和冲突解决
4. **用户体验优化**：流畅的实时反馈和直观的协作提示
5. **容错性设计**：网络异常和服务中断的优雅处理

这套协作系统为构建企业级协作工具提供了完整的技术参考。

## 下一步

下一章将探讨Excalidraw的插件系统和扩展机制，了解如何构建可扩展的插件架构。