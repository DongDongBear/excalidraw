# 8.3 生态系统集成与扩展实践

本节将介绍如何将 Excalidraw 集成到各种应用场景中，以及构建围绕 Excalidraw 的生态系统。

## 框架集成

### 1. Next.js 集成

```typescript
// pages/drawing.tsx
import dynamic from 'next/dynamic';
import { useState, useCallback } from 'react';
import { ExcalidrawElement, AppState } from '@excalidraw/excalidraw/types';

// 动态导入 Excalidraw 避免 SSR 问题
const Excalidraw = dynamic(
  () => import('@excalidraw/excalidraw').then(mod => mod.Excalidraw),
  { ssr: false }
);

interface DrawingPageProps {
  initialData?: {
    elements: ExcalidrawElement[];
    appState: Partial<AppState>;
  };
}

const DrawingPage: React.FC<DrawingPageProps> = ({ initialData }) => {
  const [excalidrawAPI, setExcalidrawAPI] = useState<any>(null);
  const [isLoading, setIsLoading] = useState(true);

  // 处理保存到数据库
  const handleSave = useCallback(async () => {
    if (!excalidrawAPI) return;

    const elements = excalidrawAPI.getSceneElements();
    const appState = excalidrawAPI.getAppState();

    try {
      await fetch('/api/drawings/save', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          elements,
          appState: {
            viewBackgroundColor: appState.viewBackgroundColor,
            gridSize: appState.gridSize
          }
        })
      });

      console.log('Drawing saved successfully');
    } catch (error) {
      console.error('Failed to save drawing:', error);
    }
  }, [excalidrawAPI]);

  // 处理实时协作
  const handleCollaboration = useCallback((elements, appState, files) => {
    // 发送到 WebSocket 服务器
    if (typeof window !== 'undefined' && window.wsConnection) {
      window.wsConnection.send(JSON.stringify({
        type: 'scene-update',
        elements,
        appState
      }));
    }
  }, []);

  return (
    <div className="drawing-page">
      {isLoading && (
        <div className="loading-overlay">
          <div>Loading Excalidraw...</div>
        </div>
      )}

      <div className="excalidraw-container" style={{ height: '100vh' }}>
        <Excalidraw
          ref={(api) => setExcalidrawAPI(api)}
          initialData={initialData}
          onChange={handleCollaboration}
          onPointerUpdate={(payload) => {
            // 处理指针位置更新
            console.log('Pointer update:', payload);
          }}
          UIOptions={{
            canvasActions: {
              saveScene: false, // 禁用默认保存，使用自定义保存
              loadScene: false,
              export: true,
              toggleTheme: true
            }
          }}
          onLibraryChange={(library) => {
            // 同步图库到服务器
            localStorage.setItem('excalidraw-library', JSON.stringify(library));
          }}
          onLoad={() => setIsLoading(false)}
        />
      </div>

      {/* 自定义工具栏 */}
      <div className="custom-toolbar">
        <button onClick={handleSave}>保存绘图</button>
        <button onClick={() => excalidrawAPI?.updateScene({ appState: { theme: 'dark' }})}>
          切换主题
        </button>
      </div>
    </div>
  );
};

export default DrawingPage;

// API 路由：pages/api/drawings/save.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { prisma } from '../../../lib/prisma'; // 假设使用 Prisma

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  try {
    const { elements, appState } = req.body;

    const drawing = await prisma.drawing.create({
      data: {
        elements: JSON.stringify(elements),
        appState: JSON.stringify(appState),
        createdAt: new Date(),
        updatedAt: new Date()
      }
    });

    res.status(200).json({ id: drawing.id, success: true });
  } catch (error) {
    console.error('Save error:', error);
    res.status(500).json({ error: 'Failed to save drawing' });
  }
}
```

### 2. Electron 桌面应用

```typescript
// main.ts - Electron 主进程
import { app, BrowserWindow, Menu, dialog, ipcMain } from 'electron';
import * as path from 'path';
import * as fs from 'fs';

class ExcalidrawDesktopApp {
  private mainWindow: BrowserWindow | null = null;

  constructor() {
    app.whenReady().then(() => {
      this.createWindow();
      this.setupMenu();
      this.setupIPC();
    });

    app.on('window-all-closed', () => {
      if (process.platform !== 'darwin') {
        app.quit();
      }
    });
  }

  private createWindow(): void {
    this.mainWindow = new BrowserWindow({
      width: 1200,
      height: 800,
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        preload: path.join(__dirname, 'preload.js')
      },
      titleBarStyle: 'hiddenInset', // macOS 样式
      show: false
    });

    // 加载 Excalidraw
    this.mainWindow.loadFile('dist/index.html');

    this.mainWindow.once('ready-to-show', () => {
      this.mainWindow?.show();
    });
  }

  private setupMenu(): void {
    const template: any = [
      {
        label: 'File',
        submenu: [
          {
            label: 'New',
            accelerator: 'CmdOrCtrl+N',
            click: () => this.newDrawing()
          },
          {
            label: 'Open',
            accelerator: 'CmdOrCtrl+O',
            click: () => this.openDrawing()
          },
          {
            label: 'Save',
            accelerator: 'CmdOrCtrl+S',
            click: () => this.saveDrawing()
          },
          {
            label: 'Save As',
            accelerator: 'CmdOrCtrl+Shift+S',
            click: () => this.saveAsDrawing()
          },
          { type: 'separator' },
          {
            label: 'Export as PNG',
            click: () => this.exportPNG()
          },
          {
            label: 'Export as SVG',
            click: () => this.exportSVG()
          }
        ]
      },
      {
        label: 'Edit',
        submenu: [
          { role: 'undo' },
          { role: 'redo' },
          { type: 'separator' },
          { role: 'cut' },
          { role: 'copy' },
          { role: 'paste' },
          { role: 'selectall' }
        ]
      },
      {
        label: 'View',
        submenu: [
          { role: 'reload' },
          { role: 'forceReload' },
          { role: 'toggleDevTools' },
          { type: 'separator' },
          { role: 'resetZoom' },
          { role: 'zoomIn' },
          { role: 'zoomOut' },
          { type: 'separator' },
          { role: 'togglefullscreen' }
        ]
      }
    ];

    const menu = Menu.buildFromTemplate(template);
    Menu.setApplicationMenu(menu);
  }

  private setupIPC(): void {
    // 文件操作 IPC
    ipcMain.handle('save-drawing', async (event, data) => {
      const result = await dialog.showSaveDialog(this.mainWindow!, {
        filters: [
          { name: 'Excalidraw Files', extensions: ['excalidraw'] },
          { name: 'JSON Files', extensions: ['json'] }
        ]
      });

      if (!result.canceled && result.filePath) {
        fs.writeFileSync(result.filePath, JSON.stringify(data, null, 2));
        return { success: true, filePath: result.filePath };
      }

      return { success: false };
    });

    ipcMain.handle('open-drawing', async () => {
      const result = await dialog.showOpenDialog(this.mainWindow!, {
        filters: [
          { name: 'Excalidraw Files', extensions: ['excalidraw'] },
          { name: 'JSON Files', extensions: ['json'] }
        ]
      });

      if (!result.canceled && result.filePaths.length > 0) {
        const data = fs.readFileSync(result.filePaths[0], 'utf-8');
        return { success: true, data: JSON.parse(data) };
      }

      return { success: false };
    });

    // 导出功能
    ipcMain.handle('export-image', async (event, { imageData, format }) => {
      const result = await dialog.showSaveDialog(this.mainWindow!, {
        filters: [
          { name: `${format.toUpperCase()} Files`, extensions: [format] }
        ]
      });

      if (!result.canceled && result.filePath) {
        const base64Data = imageData.replace(/^data:image\/[a-z]+;base64,/, '');
        fs.writeFileSync(result.filePath, base64Data, 'base64');
        return { success: true };
      }

      return { success: false };
    });
  }

  private newDrawing(): void {
    this.mainWindow?.webContents.send('menu-action', { type: 'new' });
  }

  private openDrawing(): void {
    this.mainWindow?.webContents.send('menu-action', { type: 'open' });
  }

  private saveDrawing(): void {
    this.mainWindow?.webContents.send('menu-action', { type: 'save' });
  }

  private saveAsDrawing(): void {
    this.mainWindow?.webContents.send('menu-action', { type: 'save-as' });
  }

  private exportPNG(): void {
    this.mainWindow?.webContents.send('menu-action', { type: 'export-png' });
  }

  private exportSVG(): void {
    this.mainWindow?.webContents.send('menu-action', { type: 'export-svg' });
  }
}

new ExcalidrawDesktopApp();

// preload.js - 预加载脚本
import { contextBridge, ipcRenderer } from 'electron';

contextBridge.exposeInMainWorld('electronAPI', {
  saveDrawing: (data: any) => ipcRenderer.invoke('save-drawing', data),
  openDrawing: () => ipcRenderer.invoke('open-drawing'),
  exportImage: (imageData: string, format: string) =>
    ipcRenderer.invoke('export-image', { imageData, format }),

  onMenuAction: (callback: (data: any) => void) => {
    ipcRenderer.on('menu-action', (event, data) => callback(data));
  }
});
```

### 3. React Native 移动端集成

```typescript
// ExcalidrawWebView.tsx
import React, { useRef, useImperativeHandle, forwardRef } from 'react';
import { WebView, WebViewMessageEvent } from 'react-native-webview';
import { Dimensions, StyleSheet } from 'react-native';

interface ExcalidrawWebViewRef {
  saveDrawing: () => void;
  loadDrawing: (data: any) => void;
  exportImage: () => void;
}

interface ExcalidrawWebViewProps {
  onSave?: (data: any) => void;
  onLoad?: () => void;
  initialData?: any;
}

const ExcalidrawWebView = forwardRef<ExcalidrawWebViewRef, ExcalidrawWebViewProps>(
  ({ onSave, onLoad, initialData }, ref) => {
    const webViewRef = useRef<WebView>(null);

    useImperativeHandle(ref, () => ({
      saveDrawing: () => {
        webViewRef.current?.postMessage(JSON.stringify({
          type: 'save-drawing'
        }));
      },
      loadDrawing: (data: any) => {
        webViewRef.current?.postMessage(JSON.stringify({
          type: 'load-drawing',
          data
        }));
      },
      exportImage: () => {
        webViewRef.current?.postMessage(JSON.stringify({
          type: 'export-image'
        }));
      }
    }));

    const handleMessage = (event: WebViewMessageEvent) => {
      try {
        const message = JSON.parse(event.nativeEvent.data);

        switch (message.type) {
          case 'drawing-saved':
            onSave?.(message.data);
            break;
          case 'loaded':
            onLoad?.();
            break;
          case 'image-exported':
            // 处理导出的图像
            console.log('Image exported:', message.imageData);
            break;
        }
      } catch (error) {
        console.error('Failed to parse WebView message:', error);
      }
    };

    // Excalidraw HTML 页面
    const htmlContent = `
      <!DOCTYPE html>
      <html>
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
        <title>Excalidraw</title>
        <style>
          body {
            margin: 0;
            padding: 0;
            overflow: hidden;
          }
          .excalidraw-container {
            height: 100vh;
            width: 100vw;
          }
        </style>
      </head>
      <body>
        <div id="app" class="excalidraw-container"></div>
        <script src="https://unpkg.com/react@17/umd/react.production.min.js"></script>
        <script src="https://unpkg.com/react-dom@17/umd/react-dom.production.min.js"></script>
        <script src="https://unpkg.com/@excalidraw/excalidraw/dist/excalidraw.production.min.js"></script>
        <script>
          let excalidrawAPI;

          const App = () => {
            return React.createElement(ExcalidrawLib.Excalidraw, {
              ref: (api) => { excalidrawAPI = api; },
              initialData: ${JSON.stringify(initialData || {})},
              onChange: (elements, appState) => {
                // 发送变化到 React Native
                window.ReactNativeWebView.postMessage(JSON.stringify({
                  type: 'scene-changed',
                  elements,
                  appState
                }));
              },
              UIOptions: {
                canvasActions: {
                  loadScene: false,
                  saveScene: false
                }
              }
            });
          };

          ReactDOM.render(React.createElement(App), document.getElementById('app'));

          // 监听来自 React Native 的消息
          window.addEventListener('message', (event) => {
            const message = JSON.parse(event.data);

            switch (message.type) {
              case 'save-drawing':
                if (excalidrawAPI) {
                  const elements = excalidrawAPI.getSceneElements();
                  const appState = excalidrawAPI.getAppState();

                  window.ReactNativeWebView.postMessage(JSON.stringify({
                    type: 'drawing-saved',
                    data: { elements, appState }
                  }));
                }
                break;

              case 'load-drawing':
                if (excalidrawAPI && message.data) {
                  excalidrawAPI.updateScene(message.data);
                }
                break;

              case 'export-image':
                if (excalidrawAPI) {
                  ExcalidrawLib.exportToCanvas({
                    elements: excalidrawAPI.getSceneElements(),
                    appState: excalidrawAPI.getAppState()
                  }).then((canvas) => {
                    const imageData = canvas.toDataURL('image/png');

                    window.ReactNativeWebView.postMessage(JSON.stringify({
                      type: 'image-exported',
                      imageData
                    }));
                  });
                }
                break;
            }
          });

          // 通知加载完成
          window.ReactNativeWebView.postMessage(JSON.stringify({
            type: 'loaded'
          }));
        </script>
      </body>
      </html>
    `;

    return (
      <WebView
        ref={webViewRef}
        source={{ html: htmlContent }}
        style={styles.webview}
        onMessage={handleMessage}
        javaScriptEnabled={true}
        domStorageEnabled={true}
        startInLoadingState={true}
        mixedContentMode="compatibility"
      />
    );
  }
);

const styles = StyleSheet.create({
  webview: {
    flex: 1,
    width: Dimensions.get('window').width,
    height: Dimensions.get('window').height
  }
});

export default ExcalidrawWebView;

// 使用示例
const DrawingScreen: React.FC = () => {
  const excalidrawRef = useRef<ExcalidrawWebViewRef>(null);

  const handleSave = (data: any) => {
    // 保存到本地存储或服务器
    console.log('Drawing saved:', data);
  };

  return (
    <View style={{ flex: 1 }}>
      <ExcalidrawWebView
        ref={excalidrawRef}
        onSave={handleSave}
        initialData={{ elements: [] }}
      />

      <View style={styles.toolbar}>
        <TouchableOpacity onPress={() => excalidrawRef.current?.saveDrawing()}>
          <Text>保存</Text>
        </TouchableOpacity>
        <TouchableOpacity onPress={() => excalidrawRef.current?.exportImage()}>
          <Text>导出</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};
```

## 云服务集成

### 1. 实时协作服务

```typescript
// collaboration-server.ts - WebSocket 服务器
import WebSocket from 'ws';
import { createServer } from 'http';
import express from 'express';

interface CollaborationRoom {
  id: string;
  clients: Map<string, WebSocket>;
  elements: any[];
  appState: any;
  lastModified: number;
}

class CollaborationServer {
  private rooms = new Map<string, CollaborationRoom>();
  private wss: WebSocket.Server;
  private app = express();
  private server = createServer(this.app);

  constructor(port: number = 8080) {
    this.wss = new WebSocket.Server({ server: this.server });
    this.setupWebSocketHandlers();
    this.setupHTTPRoutes();

    this.server.listen(port, () => {
      console.log(`Collaboration server started on port ${port}`);
    });
  }

  private setupWebSocketHandlers(): void {
    this.wss.on('connection', (ws: WebSocket, request) => {
      const url = new URL(request.url!, `http://${request.headers.host}`);
      const roomId = url.searchParams.get('roomId');
      const clientId = url.searchParams.get('clientId');

      if (!roomId || !clientId) {
        ws.close(1008, 'Missing roomId or clientId');
        return;
      }

      this.joinRoom(roomId, clientId, ws);

      ws.on('message', (data: WebSocket.Data) => {
        this.handleMessage(roomId, clientId, data.toString());
      });

      ws.on('close', () => {
        this.leaveRoom(roomId, clientId);
      });

      ws.on('error', (error) => {
        console.error('WebSocket error:', error);
        this.leaveRoom(roomId, clientId);
      });
    });
  }

  private joinRoom(roomId: string, clientId: string, ws: WebSocket): void {
    if (!this.rooms.has(roomId)) {
      this.rooms.set(roomId, {
        id: roomId,
        clients: new Map(),
        elements: [],
        appState: {},
        lastModified: Date.now()
      });
    }

    const room = this.rooms.get(roomId)!;
    room.clients.set(clientId, ws);

    // 发送当前场景状态给新用户
    ws.send(JSON.stringify({
      type: 'scene-init',
      elements: room.elements,
      appState: room.appState
    }));

    // 通知其他用户有新用户加入
    this.broadcastToRoom(roomId, {
      type: 'user-joined',
      clientId
    }, clientId);

    console.log(`Client ${clientId} joined room ${roomId}`);
  }

  private leaveRoom(roomId: string, clientId: string): void {
    const room = this.rooms.get(roomId);
    if (room) {
      room.clients.delete(clientId);

      // 通知其他用户有用户离开
      this.broadcastToRoom(roomId, {
        type: 'user-left',
        clientId
      });

      // 如果房间为空，删除房间
      if (room.clients.size === 0) {
        this.rooms.delete(roomId);
        console.log(`Room ${roomId} deleted`);
      }
    }
  }

  private handleMessage(roomId: string, clientId: string, data: string): void {
    try {
      const message = JSON.parse(data);
      const room = this.rooms.get(roomId);

      if (!room) return;

      switch (message.type) {
        case 'scene-update':
          room.elements = message.elements;
          room.appState = message.appState;
          room.lastModified = Date.now();

          // 广播场景更新到其他客户端
          this.broadcastToRoom(roomId, {
            type: 'scene-update',
            elements: message.elements,
            appState: message.appState,
            clientId
          }, clientId);
          break;

        case 'pointer-update':
          // 广播指针位置更新
          this.broadcastToRoom(roomId, {
            type: 'pointer-update',
            pointer: message.pointer,
            clientId
          }, clientId);
          break;

        case 'cursor-update':
          // 广播光标位置
          this.broadcastToRoom(roomId, {
            type: 'cursor-update',
            cursor: message.cursor,
            clientId
          }, clientId);
          break;
      }
    } catch (error) {
      console.error('Failed to parse message:', error);
    }
  }

  private broadcastToRoom(roomId: string, message: any, excludeClientId?: string): void {
    const room = this.rooms.get(roomId);
    if (!room) return;

    const messageStr = JSON.stringify(message);

    room.clients.forEach((ws, clientId) => {
      if (clientId !== excludeClientId && ws.readyState === WebSocket.OPEN) {
        ws.send(messageStr);
      }
    });
  }

  private setupHTTPRoutes(): void {
    this.app.use(express.json());

    // 获取房间信息
    this.app.get('/api/rooms/:roomId', (req, res) => {
      const room = this.rooms.get(req.params.roomId);

      if (room) {
        res.json({
          id: room.id,
          clientCount: room.clients.size,
          lastModified: room.lastModified
        });
      } else {
        res.status(404).json({ error: 'Room not found' });
      }
    });

    // 创建房间
    this.app.post('/api/rooms', (req, res) => {
      const roomId = this.generateRoomId();

      this.rooms.set(roomId, {
        id: roomId,
        clients: new Map(),
        elements: req.body.elements || [],
        appState: req.body.appState || {},
        lastModified: Date.now()
      });

      res.json({ roomId });
    });
  }

  private generateRoomId(): string {
    return Math.random().toString(36).substring(2, 15) +
           Math.random().toString(36).substring(2, 15);
  }
}

// 启动服务器
new CollaborationServer(process.env.PORT ? parseInt(process.env.PORT) : 8080);
```

### 2. 云存储集成

```typescript
// cloud-storage.ts - 云存储适配器
interface CloudStorageProvider {
  save(id: string, data: any): Promise<string>;
  load(id: string): Promise<any>;
  delete(id: string): Promise<void>;
  list(userId?: string): Promise<string[]>;
}

// Google Drive 适配器
class GoogleDriveProvider implements CloudStorageProvider {
  private accessToken: string;

  constructor(accessToken: string) {
    this.accessToken = accessToken;
  }

  async save(id: string, data: any): Promise<string> {
    const metadata = {
      name: `excalidraw-${id}.json`,
      parents: ['appDataFolder'] // 存储在应用数据文件夹
    };

    const form = new FormData();
    form.append('metadata', JSON.stringify(metadata));
    form.append('file', new Blob([JSON.stringify(data)], { type: 'application/json' }));

    const response = await fetch('https://www.googleapis.com/upload/drive/v3/files', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.accessToken}`
      },
      body: form
    });

    const result = await response.json();
    return result.id;
  }

  async load(id: string): Promise<any> {
    const response = await fetch(`https://www.googleapis.com/drive/v3/files/${id}?alt=media`, {
      headers: {
        'Authorization': `Bearer ${this.accessToken}`
      }
    });

    return await response.json();
  }

  async delete(id: string): Promise<void> {
    await fetch(`https://www.googleapis.com/drive/v3/files/${id}`, {
      method: 'DELETE',
      headers: {
        'Authorization': `Bearer ${this.accessToken}`
      }
    });
  }

  async list(userId?: string): Promise<string[]> {
    const response = await fetch(
      `https://www.googleapis.com/drive/v3/files?q=name contains 'excalidraw-' and parents in 'appDataFolder'`,
      {
        headers: {
          'Authorization': `Bearer ${this.accessToken}`
        }
      }
    );

    const result = await response.json();
    return result.files?.map((file: any) => file.id) || [];
  }
}

// AWS S3 适配器
class S3Provider implements CloudStorageProvider {
  private bucketName: string;
  private region: string;
  private credentials: any;

  constructor(bucketName: string, region: string, credentials: any) {
    this.bucketName = bucketName;
    this.region = region;
    this.credentials = credentials;
  }

  async save(id: string, data: any): Promise<string> {
    const key = `drawings/${id}.json`;

    // 使用 AWS SDK 或直接 REST API
    const response = await fetch(
      `https://${this.bucketName}.s3.${this.region}.amazonaws.com/${key}`,
      {
        method: 'PUT',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': this.generateAuthHeader('PUT', key)
        },
        body: JSON.stringify(data)
      }
    );

    if (response.ok) {
      return key;
    }

    throw new Error('Failed to save to S3');
  }

  async load(id: string): Promise<any> {
    const key = `drawings/${id}.json`;

    const response = await fetch(
      `https://${this.bucketName}.s3.${this.region}.amazonaws.com/${key}`,
      {
        headers: {
          'Authorization': this.generateAuthHeader('GET', key)
        }
      }
    );

    if (response.ok) {
      return await response.json();
    }

    throw new Error('Failed to load from S3');
  }

  async delete(id: string): Promise<void> {
    const key = `drawings/${id}.json`;

    await fetch(
      `https://${this.bucketName}.s3.${this.region}.amazonaws.com/${key}`,
      {
        method: 'DELETE',
        headers: {
          'Authorization': this.generateAuthHeader('DELETE', key)
        }
      }
    );
  }

  async list(userId?: string): Promise<string[]> {
    const prefix = userId ? `drawings/${userId}/` : 'drawings/';

    const response = await fetch(
      `https://${this.bucketName}.s3.${this.region}.amazonaws.com/?list-type=2&prefix=${prefix}`,
      {
        headers: {
          'Authorization': this.generateAuthHeader('GET', '/')
        }
      }
    );

    // 解析 XML 响应获取文件列表
    const xmlText = await response.text();
    // ... 解析 XML 逻辑
    return [];
  }

  private generateAuthHeader(method: string, path: string): string {
    // AWS Signature Version 4 签名逻辑
    // 这里简化处理，实际项目中应使用 AWS SDK
    return 'AWS4-HMAC-SHA256 ...';
  }
}

// 云存储管理器
class CloudStorageManager {
  private provider: CloudStorageProvider;

  constructor(provider: CloudStorageProvider) {
    this.provider = provider;
  }

  async saveDrawing(
    elements: any[],
    appState: any,
    metadata?: any
  ): Promise<string> {
    const drawingData = {
      elements,
      appState,
      metadata: {
        ...metadata,
        createdAt: new Date().toISOString(),
        version: '1.0.0'
      }
    };

    const id = this.generateDrawingId();
    const storageId = await this.provider.save(id, drawingData);

    // 缓存到本地存储
    localStorage.setItem(`excalidraw-${id}`, JSON.stringify(drawingData));

    return storageId;
  }

  async loadDrawing(id: string): Promise<any> {
    try {
      // 先尝试从本地缓存加载
      const cached = localStorage.getItem(`excalidraw-${id}`);
      if (cached) {
        const cachedData = JSON.parse(cached);

        // 异步同步云端数据
        this.syncFromCloud(id).catch(console.error);

        return cachedData;
      }

      // 从云端加载
      const data = await this.provider.load(id);

      // 缓存到本地
      localStorage.setItem(`excalidraw-${id}`, JSON.stringify(data));

      return data;
    } catch (error) {
      console.error('Failed to load drawing:', error);
      throw error;
    }
  }

  async deleteDrawing(id: string): Promise<void> {
    await this.provider.delete(id);
    localStorage.removeItem(`excalidraw-${id}`);
  }

  async listDrawings(userId?: string): Promise<string[]> {
    return await this.provider.list(userId);
  }

  private async syncFromCloud(id: string): Promise<void> {
    try {
      const cloudData = await this.provider.load(id);
      const cached = localStorage.getItem(`excalidraw-${id}`);

      if (cached) {
        const cachedData = JSON.parse(cached);

        // 比较时间戳，保留最新版本
        if (new Date(cloudData.metadata.createdAt) > new Date(cachedData.metadata.createdAt)) {
          localStorage.setItem(`excalidraw-${id}`, JSON.stringify(cloudData));
        }
      }
    } catch (error) {
      console.warn('Failed to sync from cloud:', error);
    }
  }

  private generateDrawingId(): string {
    return Date.now().toString(36) + Math.random().toString(36).substring(2);
  }
}
```

这些集成示例展示了如何将 Excalidraw 嵌入到各种平台和服务中，构建完整的生态系统。通过这些集成方案，开发者可以在不同场景下使用 Excalidraw 的强大功能。