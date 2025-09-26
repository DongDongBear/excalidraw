# Chapter 6.3: 导入导出系统实现

## 概述

导入导出功能是现代编辑器的重要特性，用户需要能够保存作品、分享内容、以及与其他工具协作。Excalidraw支持多种格式的导入导出，包括PNG、SVG、JSON、PDF等。本章将深入解析这套完整的数据交换系统。

## 导入导出架构

### 支持的格式

```
导入导出格式支持
├── 矢量格式
│   ├── SVG          # 可缩放矢量图形
│   ├── PDF          # 便携式文档格式
│   └── EPS          # 封装PostScript
├── 位图格式
│   ├── PNG          # 便携式网络图形
│   ├── JPG          # JPEG图像
│   └── WebP         # 现代图像格式
├── 数据格式
│   ├── JSON         # Excalidraw原生格式
│   ├── .excalidraw  # 包含场景数据
│   └── .excalidrawlib # 图库格式
└── 文档格式
    ├── Markdown     # 嵌入图表
    ├── HTML         # 网页嵌入
    └── Office       # Word/PowerPoint集成
```

### 核心架构

```typescript
// 导出系统核心接口
export interface ExportOptions {
  // 基础选项
  format: ExportFormat;
  quality?: number;           // 图像质量 (0-1)
  scale?: number;            // 缩放比例

  // 范围选项
  exportSelection?: boolean;  // 只导出选中内容
  exportBackground?: boolean; // 包含背景
  exportPadding?: number;     // 边距

  // 高级选项
  exportEmbedScene?: boolean; // 嵌入场景数据
  exportWithDarkMode?: boolean; // 暗色模式
  exportWithTheme?: Theme;    // 指定主题

  // 格式特定选项
  svgOptions?: SVGExportOptions;
  pdfOptions?: PDFExportOptions;
  pngOptions?: PNGExportOptions;
}

export interface ImportOptions {
  // 导入行为
  mergeWithCurrent?: boolean; // 合并到当前场景
  replaceScene?: boolean;     // 替换整个场景

  // 位置选项
  insertAtCenter?: boolean;   // 插入到中心
  insertAtPosition?: Point;   // 插入到指定位置

  // 数据处理
  preserveIds?: boolean;      // 保留元素ID
  generateNewIds?: boolean;   // 生成新ID
}

export type ExportFormat =
  | 'png' | 'jpg' | 'webp'    // 位图
  | 'svg' | 'pdf'             // 矢量
  | 'json' | 'excalidraw'     // 数据
  | 'clipboard'               // 剪贴板
  | 'blob';                   // 二进制
```

## 导出系统实现

### ExportManager核心类

```typescript
// packages/excalidraw/export/ExportManager.ts
export class ExportManager {
  private canvas: HTMLCanvasElement;
  private tempCanvas: HTMLCanvasElement;
  private renderConfig: RenderConfig;

  constructor(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.tempCanvas = document.createElement('canvas');
    this.renderConfig = getDefaultRenderConfig();
  }

  // 主导出方法
  async exportScene(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles,
    options: ExportOptions
  ): Promise<ExportResult> {
    try {
      // 预处理元素
      const processedElements = this.preprocessElements(elements, options);

      // 根据格式分发
      switch (options.format) {
        case 'png':
        case 'jpg':
        case 'webp':
          return await this.exportRaster(processedElements, appState, files, options);

        case 'svg':
          return await this.exportSVG(processedElements, appState, files, options);

        case 'pdf':
          return await this.exportPDF(processedElements, appState, files, options);

        case 'json':
        case 'excalidraw':
          return await this.exportJSON(processedElements, appState, files, options);

        case 'clipboard':
          return await this.exportToClipboard(processedElements, appState, files, options);

        default:
          throw new Error(`Unsupported export format: ${options.format}`);
      }
    } catch (error) {
      console.error('Export failed:', error);
      throw error;
    }
  }

  // 位图导出
  private async exportRaster(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles,
    options: ExportOptions
  ): Promise<ExportResult> {
    // 计算导出区域
    const bounds = this.calculateExportBounds(elements, options);

    // 设置画布尺寸
    const scale = options.scale || window.devicePixelRatio || 1;
    const exportWidth = Math.ceil(bounds.width * scale);
    const exportHeight = Math.ceil(bounds.height * scale);

    // 检查尺寸限制
    this.validateCanvasSize(exportWidth, exportHeight);

    // 设置临时画布
    this.tempCanvas.width = exportWidth;
    this.tempCanvas.height = exportHeight;

    const context = this.tempCanvas.getContext('2d')!;

    // 设置高质量渲染
    context.imageSmoothingEnabled = true;
    context.imageSmoothingQuality = 'high';

    // 缩放和平移
    context.scale(scale, scale);
    context.translate(-bounds.x, -bounds.y);

    // 渲染背景
    if (options.exportBackground !== false) {
      this.renderBackground(context, bounds, appState);
    }

    // 渲染元素
    await this.renderElements(context, elements, files, appState);

    // 转换为指定格式
    const mimeType = this.getMimeType(options.format);
    const quality = options.quality || 0.92;

    const blob = await this.canvasToBlob(this.tempCanvas, mimeType, quality);

    return {
      data: blob,
      width: exportWidth,
      height: exportHeight,
      mimeType,
    };
  }

  // SVG导出
  private async exportSVG(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles,
    options: ExportOptions
  ): Promise<ExportResult> {
    const bounds = this.calculateExportBounds(elements, options);

    // 创建SVG文档
    const svgGenerator = new SVGGenerator(bounds, options.svgOptions);

    // 添加背景
    if (options.exportBackground !== false) {
      svgGenerator.addBackground(appState.viewBackgroundColor);
    }

    // 渲染元素到SVG
    for (const element of elements) {
      await svgGenerator.renderElement(element, files);
    }

    // 嵌入场景数据（可选）
    if (options.exportEmbedScene) {
      const sceneData = this.createSceneData(elements, appState, files);
      svgGenerator.embedSceneData(sceneData);
    }

    const svgString = svgGenerator.toString();
    const blob = new Blob([svgString], { type: 'image/svg+xml' });

    return {
      data: blob,
      width: bounds.width,
      height: bounds.height,
      mimeType: 'image/svg+xml',
    };
  }

  // PDF导出
  private async exportPDF(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles,
    options: ExportOptions
  ): Promise<ExportResult> {
    const bounds = this.calculateExportBounds(elements, options);

    // 使用PDF库（如jsPDF）
    const pdfGenerator = new PDFGenerator(bounds, options.pdfOptions);

    // 先导出为SVG，然后转换为PDF
    const svgResult = await this.exportSVG(elements, appState, files, {
      ...options,
      format: 'svg',
    });

    const svgString = await this.blobToString(svgResult.data);
    await pdfGenerator.addSVG(svgString);

    const pdfBlob = await pdfGenerator.generateBlob();

    return {
      data: pdfBlob,
      width: bounds.width,
      height: bounds.height,
      mimeType: 'application/pdf',
    };
  }

  // JSON导出
  private async exportJSON(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles,
    options: ExportOptions
  ): Promise<ExportResult> {
    const sceneData = this.createSceneData(elements, appState, files);
    const jsonString = JSON.stringify(sceneData, null, 2);
    const blob = new Blob([jsonString], { type: 'application/json' });

    return {
      data: blob,
      mimeType: 'application/json',
    };
  }

  // 剪贴板导出
  private async exportToClipboard(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles,
    options: ExportOptions
  ): Promise<ExportResult> {
    const clipboardItems: ClipboardItem[] = [];

    // PNG格式
    const pngResult = await this.exportRaster(elements, appState, files, {
      ...options,
      format: 'png',
    });
    clipboardItems.push(new ClipboardItem({
      'image/png': pngResult.data,
    }));

    // SVG格式
    const svgResult = await this.exportSVG(elements, appState, files, options);
    clipboardItems.push(new ClipboardItem({
      'image/svg+xml': svgResult.data,
    }));

    // 写入剪贴板
    await navigator.clipboard.write(clipboardItems);

    return {
      data: pngResult.data,
      mimeType: 'image/png',
    };
  }

  // 预处理元素
  private preprocessElements(
    elements: readonly ExcalidrawElement[],
    options: ExportOptions
  ): readonly ExcalidrawElement[] {
    let processedElements = [...elements];

    // 过滤删除的元素
    processedElements = processedElements.filter(el => !el.isDeleted);

    // 如果只导出选中内容
    if (options.exportSelection) {
      const selectedIds = new Set(Object.keys(/* selectedElementIds */));
      processedElements = processedElements.filter(el => selectedIds.has(el.id));
    }

    // 按z-index排序
    processedElements.sort((a, b) => a.index.localeCompare(b.index));

    return processedElements;
  }

  // 计算导出边界
  private calculateExportBounds(
    elements: readonly ExcalidrawElement[],
    options: ExportOptions
  ): Rectangle {
    if (elements.length === 0) {
      return { x: 0, y: 0, width: 100, height: 100 };
    }

    let minX = Infinity;
    let minY = Infinity;
    let maxX = -Infinity;
    let maxY = -Infinity;

    for (const element of elements) {
      const bounds = getElementBounds(element);
      minX = Math.min(minX, bounds.x);
      minY = Math.min(minY, bounds.y);
      maxX = Math.max(maxX, bounds.x + bounds.width);
      maxY = Math.max(maxY, bounds.y + bounds.height);
    }

    // 添加边距
    const padding = options.exportPadding || 10;

    return {
      x: minX - padding,
      y: minY - padding,
      width: maxX - minX + padding * 2,
      height: maxY - minY + padding * 2,
    };
  }

  // 验证画布尺寸
  private validateCanvasSize(width: number, height: number): void {
    const maxSize = 16384; // 大多数浏览器的限制
    const maxPixels = 268435456; // 256MB

    if (width > maxSize || height > maxSize) {
      throw new Error(`Canvas size too large: ${width}x${height}. Maximum size is ${maxSize}x${maxSize}.`);
    }

    if (width * height > maxPixels) {
      throw new Error(`Canvas area too large: ${width * height} pixels. Maximum area is ${maxPixels} pixels.`);
    }
  }

  // 辅助方法
  private getMimeType(format: ExportFormat): string {
    const mimeTypes = {
      png: 'image/png',
      jpg: 'image/jpeg',
      webp: 'image/webp',
      svg: 'image/svg+xml',
      pdf: 'application/pdf',
      json: 'application/json',
    };

    return mimeTypes[format] || 'application/octet-stream';
  }

  private async canvasToBlob(
    canvas: HTMLCanvasElement,
    mimeType: string,
    quality: number
  ): Promise<Blob> {
    return new Promise((resolve, reject) => {
      canvas.toBlob(
        (blob) => {
          if (blob) {
            resolve(blob);
          } else {
            reject(new Error('Failed to create blob from canvas'));
          }
        },
        mimeType,
        quality
      );
    });
  }

  private async blobToString(blob: Blob): Promise<string> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => resolve(reader.result as string);
      reader.onerror = reject;
      reader.readAsText(blob);
    });
  }

  private createSceneData(
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles
  ): SceneData {
    return {
      type: 'excalidraw',
      version: 2,
      source: 'https://excalidraw.com',
      elements,
      appState: {
        theme: appState.theme,
        viewBackgroundColor: appState.viewBackgroundColor,
        currentItemStrokeColor: appState.currentItemStrokeColor,
        currentItemBackgroundColor: appState.currentItemBackgroundColor,
        currentItemFillStyle: appState.currentItemFillStyle,
        currentItemStrokeWidth: appState.currentItemStrokeWidth,
        currentItemStrokeStyle: appState.currentItemStrokeStyle,
        currentItemRoughness: appState.currentItemRoughness,
        currentItemOpacity: appState.currentItemOpacity,
        gridSize: appState.gridSize,
      },
      files,
    };
  }
}

// 导出结果接口
interface ExportResult {
  data: Blob;
  width?: number;
  height?: number;
  mimeType: string;
}

interface SceneData {
  type: 'excalidraw';
  version: number;
  source: string;
  elements: readonly ExcalidrawElement[];
  appState: Partial<AppState>;
  files?: BinaryFiles;
}
```

### SVG生成器

```typescript
// packages/excalidraw/export/SVGGenerator.ts
export class SVGGenerator {
  private svgElement: SVGElement;
  private defs: SVGDefsElement;
  private bounds: Rectangle;
  private options: SVGExportOptions;

  constructor(bounds: Rectangle, options: SVGExportOptions = {}) {
    this.bounds = bounds;
    this.options = options;
    this.createSVGDocument();
  }

  // 创建SVG文档
  private createSVGDocument(): void {
    // 创建SVG根元素
    this.svgElement = document.createElementNS('http://www.w3.org/2000/svg', 'svg');
    this.svgElement.setAttribute('width', this.bounds.width.toString());
    this.svgElement.setAttribute('height', this.bounds.height.toString());
    this.svgElement.setAttribute('viewBox',
      `${this.bounds.x} ${this.bounds.y} ${this.bounds.width} ${this.bounds.height}`);
    this.svgElement.setAttribute('version', '1.1');
    this.svgElement.setAttribute('xmlns', 'http://www.w3.org/2000/svg');
    this.svgElement.setAttribute('xmlns:xlink', 'http://www.w3.org/1999/xlink');

    // 创建defs元素用于定义可重用的元素
    this.defs = document.createElementNS('http://www.w3.org/2000/svg', 'defs');
    this.svgElement.appendChild(this.defs);

    // 添加手绘风格的滤镜
    this.addRoughFilters();
  }

  // 添加背景
  addBackground(backgroundColor: string): void {
    const rect = document.createElementNS('http://www.w3.org/2000/svg', 'rect');
    rect.setAttribute('x', this.bounds.x.toString());
    rect.setAttribute('y', this.bounds.y.toString());
    rect.setAttribute('width', this.bounds.width.toString());
    rect.setAttribute('height', this.bounds.height.toString());
    rect.setAttribute('fill', backgroundColor);

    this.svgElement.appendChild(rect);
  }

  // 渲染元素到SVG
  async renderElement(element: ExcalidrawElement, files: BinaryFiles): Promise<void> {
    const group = document.createElementNS('http://www.w3.org/2000/svg', 'g');

    // 应用变换
    if (element.angle !== 0) {
      const centerX = element.x + element.width / 2;
      const centerY = element.y + element.height / 2;
      group.setAttribute('transform',
        `rotate(${element.angle * 180 / Math.PI} ${centerX} ${centerY})`);
    }

    // 根据元素类型渲染
    switch (element.type) {
      case 'rectangle':
        this.renderRectangle(group, element);
        break;
      case 'ellipse':
        this.renderEllipse(group, element);
        break;
      case 'diamond':
        this.renderDiamond(group, element);
        break;
      case 'arrow':
      case 'line':
        this.renderLine(group, element);
        break;
      case 'freedraw':
        this.renderFreedraw(group, element);
        break;
      case 'text':
        this.renderText(group, element);
        break;
      case 'image':
        await this.renderImage(group, element, files);
        break;
    }

    this.svgElement.appendChild(group);
  }

  // 渲染矩形
  private renderRectangle(parent: SVGElement, element: ExcalidrawRectangleElement): void {
    const rect = document.createElementNS('http://www.w3.org/2000/svg', 'rect');

    rect.setAttribute('x', element.x.toString());
    rect.setAttribute('y', element.y.toString());
    rect.setAttribute('width', element.width.toString());
    rect.setAttribute('height', element.height.toString());

    // 应用样式
    this.applyElementStyles(rect, element);

    // 圆角处理
    if (element.roundness) {
      const radius = Math.min(element.width, element.height) * 0.1;
      rect.setAttribute('rx', radius.toString());
      rect.setAttribute('ry', radius.toString());
    }

    parent.appendChild(rect);
  }

  // 渲染椭圆
  private renderEllipse(parent: SVGElement, element: ExcalidrawEllipseElement): void {
    const ellipse = document.createElementNS('http://www.w3.org/2000/svg', 'ellipse');

    const cx = element.x + element.width / 2;
    const cy = element.y + element.height / 2;
    const rx = element.width / 2;
    const ry = element.height / 2;

    ellipse.setAttribute('cx', cx.toString());
    ellipse.setAttribute('cy', cy.toString());
    ellipse.setAttribute('rx', rx.toString());
    ellipse.setAttribute('ry', ry.toString());

    this.applyElementStyles(ellipse, element);
    parent.appendChild(ellipse);
  }

  // 渲染菱形
  private renderDiamond(parent: SVGElement, element: ExcalidrawDiamondElement): void {
    const polygon = document.createElementNS('http://www.w3.org/2000/svg', 'polygon');

    const centerX = element.x + element.width / 2;
    const centerY = element.y + element.height / 2;
    const halfWidth = element.width / 2;
    const halfHeight = element.height / 2;

    const points = [
      `${centerX},${element.y}`,                    // 顶部
      `${element.x + element.width},${centerY}`,    // 右侧
      `${centerX},${element.y + element.height}`,   // 底部
      `${element.x},${centerY}`,                    // 左侧
    ].join(' ');

    polygon.setAttribute('points', points);
    this.applyElementStyles(polygon, element);
    parent.appendChild(polygon);
  }

  // 渲染线条
  private renderLine(parent: SVGElement, element: ExcalidrawLinearElement): void {
    const path = document.createElementNS('http://www.w3.org/2000/svg', 'path');

    // 构建路径数据
    let pathData = '';

    element.points.forEach((point, index) => {
      const x = element.x + point[0];
      const y = element.y + point[1];

      if (index === 0) {
        pathData += `M ${x} ${y}`;
      } else {
        pathData += ` L ${x} ${y}`;
      }
    });

    path.setAttribute('d', pathData);
    path.setAttribute('fill', 'none');
    path.setAttribute('stroke', element.strokeColor);
    path.setAttribute('stroke-width', element.strokeWidth.toString());

    // 线条样式
    if (element.strokeStyle === 'dashed') {
      path.setAttribute('stroke-dasharray', '5,5');
    } else if (element.strokeStyle === 'dotted') {
      path.setAttribute('stroke-dasharray', '1,3');
    }

    // 箭头处理
    if (element.type === 'arrow') {
      this.addArrowheads(path, element);
    }

    parent.appendChild(path);
  }

  // 渲染自由绘制
  private renderFreedraw(parent: SVGElement, element: ExcalidrawFreeDrawElement): void {
    const path = document.createElementNS('http://www.w3.org/2000/svg', 'path');

    // 使用平滑曲线连接点
    let pathData = '';

    for (let i = 0; i < element.points.length; i++) {
      const point = element.points[i];
      const x = element.x + point[0];
      const y = element.y + point[1];

      if (i === 0) {
        pathData += `M ${x} ${y}`;
      } else if (i === 1) {
        pathData += ` L ${x} ${y}`;
      } else {
        // 使用二次贝塞尔曲线进行平滑
        const prevPoint = element.points[i - 1];
        const prevX = element.x + prevPoint[0];
        const prevY = element.y + prevPoint[1];

        const cpX = (prevX + x) / 2;
        const cpY = (prevY + y) / 2;

        pathData += ` Q ${prevX} ${prevY} ${cpX} ${cpY}`;
      }
    }

    path.setAttribute('d', pathData);
    path.setAttribute('fill', 'none');
    path.setAttribute('stroke', element.strokeColor);
    path.setAttribute('stroke-width', element.strokeWidth.toString());
    path.setAttribute('stroke-linecap', 'round');
    path.setAttribute('stroke-linejoin', 'round');

    parent.appendChild(path);
  }

  // 渲染文本
  private renderText(parent: SVGElement, element: ExcalidrawTextElement): void {
    const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');

    text.setAttribute('x', element.x.toString());
    text.setAttribute('y', (element.y + element.fontSize).toString());
    text.setAttribute('font-family', element.fontFamily);
    text.setAttribute('font-size', element.fontSize.toString());
    text.setAttribute('fill', element.strokeColor);

    // 文本对齐
    if (element.textAlign === 'center') {
      text.setAttribute('text-anchor', 'middle');
      text.setAttribute('x', (element.x + element.width / 2).toString());
    } else if (element.textAlign === 'right') {
      text.setAttribute('text-anchor', 'end');
      text.setAttribute('x', (element.x + element.width).toString());
    }

    // 分行处理
    const lines = element.text.split('\n');
    lines.forEach((line, index) => {
      const tspan = document.createElementNS('http://www.w3.org/2000/svg', 'tspan');
      tspan.setAttribute('x', text.getAttribute('x')!);
      tspan.setAttribute('dy', index === 0 ? '0' : '1.2em');
      tspan.textContent = line;
      text.appendChild(tspan);
    });

    parent.appendChild(text);
  }

  // 渲染图片
  private async renderImage(
    parent: SVGElement,
    element: ExcalidrawImageElement,
    files: BinaryFiles
  ): Promise<void> {
    const fileData = files[element.fileId];
    if (!fileData) {
      console.warn(`Image file ${element.fileId} not found`);
      return;
    }

    const image = document.createElementNS('http://www.w3.org/2000/svg', 'image');

    // 转换为data URL
    const dataURL = await this.fileToDataURL(fileData);

    image.setAttribute('x', element.x.toString());
    image.setAttribute('y', element.y.toString());
    image.setAttribute('width', element.width.toString());
    image.setAttribute('height', element.height.toString());
    image.setAttribute('href', dataURL);

    parent.appendChild(image);
  }

  // 应用元素样式
  private applyElementStyles(svgElement: SVGElement, element: ExcalidrawElement): void {
    // 描边
    svgElement.setAttribute('stroke', element.strokeColor);
    svgElement.setAttribute('stroke-width', element.strokeWidth.toString());

    // 填充
    if (element.backgroundColor === 'transparent') {
      svgElement.setAttribute('fill', 'none');
    } else {
      svgElement.setAttribute('fill', element.backgroundColor);

      // 填充样式
      if (element.fillStyle === 'hachure') {
        const patternId = this.createHachurePattern(element);
        svgElement.setAttribute('fill', `url(#${patternId})`);
      } else if (element.fillStyle === 'cross-hatch') {
        const patternId = this.createCrossHatchPattern(element);
        svgElement.setAttribute('fill', `url(#${patternId})`);
      }
    }

    // 透明度
    if (element.opacity < 100) {
      svgElement.setAttribute('opacity', (element.opacity / 100).toString());
    }

    // 线条样式
    if ('strokeStyle' in element) {
      if (element.strokeStyle === 'dashed') {
        svgElement.setAttribute('stroke-dasharray', `${element.strokeWidth * 3},${element.strokeWidth * 2}`);
      } else if (element.strokeStyle === 'dotted') {
        svgElement.setAttribute('stroke-dasharray', `${element.strokeWidth},${element.strokeWidth * 2}`);
      }
    }
  }

  // 创建阴影线图案
  private createHachurePattern(element: ExcalidrawElement): string {
    const patternId = `hachure-${Math.random().toString(36).substr(2, 9)}`;
    const pattern = document.createElementNS('http://www.w3.org/2000/svg', 'pattern');

    pattern.setAttribute('id', patternId);
    pattern.setAttribute('patternUnits', 'userSpaceOnUse');
    pattern.setAttribute('width', '8');
    pattern.setAttribute('height', '8');
    pattern.setAttribute('patternTransform', 'rotate(45)');

    const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
    line.setAttribute('x1', '0');
    line.setAttribute('y1', '0');
    line.setAttribute('x2', '0');
    line.setAttribute('y2', '8');
    line.setAttribute('stroke', element.backgroundColor);
    line.setAttribute('stroke-width', '1');

    pattern.appendChild(line);
    this.defs.appendChild(pattern);

    return patternId;
  }

  // 添加手绘风格滤镜
  private addRoughFilters(): void {
    // 这里可以添加SVG滤镜来模拟手绘效果
    const filter = document.createElementNS('http://www.w3.org/2000/svg', 'filter');
    filter.setAttribute('id', 'rough');

    const turbulence = document.createElementNS('http://www.w3.org/2000/svg', 'feTurbulence');
    turbulence.setAttribute('baseFrequency', '0.02');
    turbulence.setAttribute('numOctaves', '3');
    turbulence.setAttribute('result', 'noise');

    const displacementMap = document.createElementNS('http://www.w3.org/2000/svg', 'feDisplacementMap');
    displacementMap.setAttribute('in', 'SourceGraphic');
    displacementMap.setAttribute('in2', 'noise');
    displacementMap.setAttribute('scale', '1');

    filter.appendChild(turbulence);
    filter.appendChild(displacementMap);
    this.defs.appendChild(filter);
  }

  // 嵌入场景数据
  embedSceneData(sceneData: SceneData): void {
    const metadata = document.createElementNS('http://www.w3.org/2000/svg', 'metadata');
    metadata.textContent = JSON.stringify(sceneData);
    this.svgElement.appendChild(metadata);
  }

  // 转换为字符串
  toString(): string {
    return new XMLSerializer().serializeToString(this.svgElement);
  }

  // 辅助方法
  private async fileToDataURL(fileData: BinaryFileData): Promise<string> {
    const blob = new Blob([fileData.dataURL], { type: fileData.mimeType });
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => resolve(reader.result as string);
      reader.onerror = reject;
      reader.readAsDataURL(blob);
    });
  }

  private createCrossHatchPattern(element: ExcalidrawElement): string {
    // 实现交叉阴影线图案
    const patternId = `crosshatch-${Math.random().toString(36).substr(2, 9)}`;
    // ... 实现细节
    return patternId;
  }

  private addArrowheads(path: SVGPathElement, element: ExcalidrawArrowElement): void {
    // 实现箭头头部
    // ... 实现细节
  }
}

interface SVGExportOptions {
  embedFonts?: boolean;
  convertTextToPath?: boolean;
  includeMetadata?: boolean;
}
```

## 导入系统实现

### ImportManager核心类

```typescript
// packages/excalidraw/import/ImportManager.ts
export class ImportManager {
  private fileHandlers: Map<string, FileHandler> = new Map();
  private maxFileSize = 50 * 1024 * 1024; // 50MB

  constructor() {
    this.registerDefaultHandlers();
  }

  // 注册默认处理器
  private registerDefaultHandlers(): void {
    this.fileHandlers.set('.json', new JSONHandler());
    this.fileHandlers.set('.excalidraw', new ExcalidrawHandler());
    this.fileHandlers.set('.svg', new SVGHandler());
    this.fileHandlers.set('.png', new ImageHandler());
    this.fileHandlers.set('.jpg', new ImageHandler());
    this.fileHandlers.set('.jpeg', new ImageHandler());
    this.fileHandlers.set('.gif', new ImageHandler());
    this.fileHandlers.set('.webp', new ImageHandler());
  }

  // 导入文件
  async importFile(
    file: File,
    options: ImportOptions = {}
  ): Promise<ImportResult> {
    try {
      // 检查文件大小
      if (file.size > this.maxFileSize) {
        throw new Error(`File too large: ${file.size} bytes. Maximum size is ${this.maxFileSize} bytes.`);
      }

      // 获取文件扩展名
      const extension = this.getFileExtension(file.name);
      const handler = this.fileHandlers.get(extension);

      if (!handler) {
        throw new Error(`Unsupported file type: ${extension}`);
      }

      // 读取文件
      const fileContent = await this.readFile(file);

      // 使用相应处理器处理文件
      const result = await handler.handle(fileContent, file, options);

      return {
        elements: result.elements || [],
        appState: result.appState || {},
        files: result.files || {},
        success: true,
      };
    } catch (error) {
      console.error('Import failed:', error);
      return {
        elements: [],
        appState: {},
        files: {},
        success: false,
        error: error instanceof Error ? error.message : String(error),
      };
    }
  }

  // 导入多个文件
  async importFiles(
    files: FileList | File[],
    options: ImportOptions = {}
  ): Promise<ImportResult[]> {
    const results: ImportResult[] = [];

    for (const file of Array.from(files)) {
      const result = await this.importFile(file, options);
      results.push(result);
    }

    return results;
  }

  // 从URL导入
  async importFromURL(
    url: string,
    options: ImportOptions = {}
  ): Promise<ImportResult> {
    try {
      const response = await fetch(url);

      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const blob = await response.blob();
      const filename = this.extractFilenameFromURL(url) || 'import';
      const file = new File([blob], filename, { type: blob.type });

      return await this.importFile(file, options);
    } catch (error) {
      console.error('Import from URL failed:', error);
      return {
        elements: [],
        appState: {},
        files: {},
        success: false,
        error: error instanceof Error ? error.message : String(error),
      };
    }
  }

  // 从剪贴板导入
  async importFromClipboard(): Promise<ImportResult> {
    try {
      const clipboardItems = await navigator.clipboard.read();

      for (const item of clipboardItems) {
        // 尝试JSON格式
        if (item.types.includes('text/plain')) {
          const text = await (await item.getType('text/plain')).text();
          try {
            const jsonData = JSON.parse(text);
            if (this.isExcalidrawData(jsonData)) {
              return await this.importExcalidrawData(jsonData);
            }
          } catch {
            // 不是JSON，继续尝试其他格式
          }
        }

        // 尝试图片格式
        for (const type of item.types) {
          if (type.startsWith('image/')) {
            const blob = await item.getType(type);
            const file = new File([blob], `clipboard.${type.split('/')[1]}`, { type });
            return await this.importFile(file);
          }
        }
      }

      throw new Error('No supported content found in clipboard');
    } catch (error) {
      console.error('Import from clipboard failed:', error);
      return {
        elements: [],
        appState: {},
        files: {},
        success: false,
        error: error instanceof Error ? error.message : String(error),
      };
    }
  }

  // 辅助方法
  private async readFile(file: File): Promise<string | ArrayBuffer> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();

      reader.onload = () => resolve(reader.result!);
      reader.onerror = reject;

      // 根据文件类型选择读取方式
      if (this.isTextFile(file)) {
        reader.readAsText(file);
      } else {
        reader.readAsArrayBuffer(file);
      }
    });
  }

  private getFileExtension(filename: string): string {
    const lastDotIndex = filename.lastIndexOf('.');
    return lastDotIndex !== -1 ? filename.slice(lastDotIndex).toLowerCase() : '';
  }

  private isTextFile(file: File): boolean {
    const textTypes = ['application/json', 'text/plain', 'image/svg+xml'];
    const textExtensions = ['.json', '.excalidraw', '.svg', '.txt'];

    return textTypes.includes(file.type) ||
           textExtensions.some(ext => file.name.toLowerCase().endsWith(ext));
  }

  private extractFilenameFromURL(url: string): string | null {
    try {
      const urlObj = new URL(url);
      const pathname = urlObj.pathname;
      return pathname.split('/').pop() || null;
    } catch {
      return null;
    }
  }

  private isExcalidrawData(data: any): boolean {
    return data &&
           typeof data === 'object' &&
           (data.type === 'excalidraw' || Array.isArray(data.elements));
  }

  private async importExcalidrawData(data: any): Promise<ImportResult> {
    const handler = this.fileHandlers.get('.excalidraw')!;
    return await handler.handle(JSON.stringify(data), null as any, {});
  }
}

// 文件处理器接口
interface FileHandler {
  handle(
    content: string | ArrayBuffer,
    file: File,
    options: ImportOptions
  ): Promise<Partial<ImportResult>>;
}

// JSON处理器
class JSONHandler implements FileHandler {
  async handle(
    content: string | ArrayBuffer,
    file: File,
    options: ImportOptions
  ): Promise<Partial<ImportResult>> {
    const jsonString = content as string;
    const data = JSON.parse(jsonString);

    // 验证是否为Excalidraw格式
    if (data.type === 'excalidraw' || Array.isArray(data.elements)) {
      return {
        elements: data.elements || [],
        appState: data.appState || {},
        files: data.files || {},
      };
    }

    throw new Error('Invalid Excalidraw JSON format');
  }
}

// SVG处理器
class SVGHandler implements FileHandler {
  async handle(
    content: string | ArrayBuffer,
    file: File,
    options: ImportOptions
  ): Promise<Partial<ImportResult>> {
    const svgString = content as string;

    // 解析SVG
    const parser = new DOMParser();
    const svgDoc = parser.parseFromString(svgString, 'image/svg+xml');

    // 检查是否包含嵌入的Excalidraw数据
    const metadata = svgDoc.querySelector('metadata');
    if (metadata) {
      try {
        const sceneData = JSON.parse(metadata.textContent || '');
        if (sceneData.type === 'excalidraw') {
          return {
            elements: sceneData.elements || [],
            appState: sceneData.appState || {},
            files: sceneData.files || {},
          };
        }
      } catch {
        // 忽略解析错误，继续处理SVG
      }
    }

    // 将SVG转换为图片元素
    const svgElement = svgDoc.documentElement;
    const width = parseFloat(svgElement.getAttribute('width') || '200');
    const height = parseFloat(svgElement.getAttribute('height') || '200');

    // 创建图片文件
    const fileId = this.generateFileId();
    const files = {
      [fileId]: {
        mimeType: 'image/svg+xml',
        id: fileId,
        dataURL: `data:image/svg+xml;base64,${btoa(svgString)}`,
        created: Date.now(),
        lastRetrieved: Date.now(),
      }
    };

    // 创建图片元素
    const imageElement: ExcalidrawImageElement = {
      type: 'image',
      id: this.generateElementId(),
      x: 0,
      y: 0,
      width,
      height,
      angle: 0,
      strokeColor: '#000000',
      backgroundColor: 'transparent',
      fillStyle: 'hachure',
      strokeWidth: 1,
      strokeStyle: 'solid',
      roughness: 1,
      opacity: 100,
      versionNonce: Math.floor(Math.random() * 2 ** 31),
      isDeleted: false,
      groupIds: [],
      frameId: null,
      index: 'a0',
      roundness: null,
      seed: Math.floor(Math.random() * 2 ** 31),
      fileId,
      status: 'saved',
      scale: [1, 1],
    };

    return {
      elements: [imageElement],
      files,
    };
  }

  private generateElementId(): string {
    return Math.random().toString(36).substr(2, 9);
  }

  private generateFileId(): string {
    return Math.random().toString(36).substr(2, 9);
  }
}

// 图片处理器
class ImageHandler implements FileHandler {
  async handle(
    content: string | ArrayBuffer,
    file: File,
    options: ImportOptions
  ): Promise<Partial<ImportResult>> {
    const arrayBuffer = content as ArrayBuffer;

    // 转换为data URL
    const blob = new Blob([arrayBuffer], { type: file.type });
    const dataURL = await this.blobToDataURL(blob);

    // 获取图片尺寸
    const dimensions = await this.getImageDimensions(dataURL);

    // 创建图片文件
    const fileId = this.generateFileId();
    const files = {
      [fileId]: {
        mimeType: file.type,
        id: fileId,
        dataURL,
        created: Date.now(),
        lastRetrieved: Date.now(),
      }
    };

    // 创建图片元素
    const imageElement: ExcalidrawImageElement = {
      type: 'image',
      id: this.generateElementId(),
      x: options.insertAtPosition?.x || 0,
      y: options.insertAtPosition?.y || 0,
      width: dimensions.width,
      height: dimensions.height,
      angle: 0,
      strokeColor: '#000000',
      backgroundColor: 'transparent',
      fillStyle: 'hachure',
      strokeWidth: 1,
      strokeStyle: 'solid',
      roughness: 1,
      opacity: 100,
      versionNonce: Math.floor(Math.random() * 2 ** 31),
      isDeleted: false,
      groupIds: [],
      frameId: null,
      index: 'a0',
      roundness: null,
      seed: Math.floor(Math.random() * 2 ** 31),
      fileId,
      status: 'saved',
      scale: [1, 1],
    };

    return {
      elements: [imageElement],
      files,
    };
  }

  private async blobToDataURL(blob: Blob): Promise<string> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => resolve(reader.result as string);
      reader.onerror = reject;
      reader.readAsDataURL(blob);
    });
  }

  private async getImageDimensions(dataURL: string): Promise<{ width: number; height: number }> {
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => resolve({ width: img.width, height: img.height });
      img.onerror = reject;
      img.src = dataURL;
    });
  }

  private generateElementId(): string {
    return Math.random().toString(36).substr(2, 9);
  }

  private generateFileId(): string {
    return Math.random().toString(36).substr(2, 9);
  }
}

// 导入结果接口
interface ImportResult {
  elements: readonly ExcalidrawElement[];
  appState: Partial<AppState>;
  files: BinaryFiles;
  success: boolean;
  error?: string;
}
```

## 思考题

1. **大文件导出如何优化？** 处理包含大量元素的场景时的性能策略？

2. **格式兼容性如何保证？** 确保导出的文件在不同环境下正确显示？

3. **数据压缩如何实现？** 减少导出文件大小的压缩算法？

4. **批量操作如何设计？** 同时导入导出多个文件的用户体验？

5. **版本兼容如何处理？** 不同版本Excalidraw文件的向前向后兼容？

## 总结

Excalidraw的导入导出系统体现了现代应用的数据交换能力：

1. **多格式支持**：满足不同场景的数据交换需求
2. **高质量输出**：保证导出内容的视觉质量
3. **智能处理**：自动识别文件类型和优化处理流程
4. **错误处理**：完善的异常处理和用户反馈机制
5. **扩展性设计**：支持新格式的轻松集成

这套系统为用户提供了完整的数据管理解决方案。

## 下一步

下一章将探讨Excalidraw的最小核心实现，了解如何构建一个精简但功能完整的绘图编辑器。