# 8.1 开发工作流与最佳实践

本章节将系统性地介绍 Excalidraw 的开发工作流程、代码规范、测试策略和团队协作最佳实践。

## 开发环境搭建

### 1. 项目结构理解

```bash
excalidraw/
├── packages/
│   ├── excalidraw/           # 核心库
│   │   ├── src/
│   │   │   ├── components/   # UI 组件
│   │   │   ├── element/      # 元素系统
│   │   │   ├── actions/      # 动作系统
│   │   │   ├── renderer/     # 渲染引擎
│   │   │   └── utils/        # 工具函数
│   │   ├── types/            # TypeScript 类型定义
│   │   └── index.ts          # 导出接口
│   ├── utils/                # 通用工具包
│   └── math/                 # 数学计算包
├── excalidraw-app/           # Web 应用
│   ├── src/
│   │   ├── components/       # 应用级组件
│   │   ├── data/            # 数据持久化
│   │   └── collab/          # 协作功能
│   └── public/              # 静态资源
├── examples/                 # 集成示例
└── dev-docs/                # 开发文档
```

### 2. 开发环境配置

```bash
# 克隆项目
git clone https://github.com/excalidraw/excalidraw.git
cd excalidraw

# 安装依赖（使用 yarn）
yarn install

# 启动开发服务器
yarn start

# 运行测试
yarn test

# 类型检查
yarn test:typecheck

# 构建项目
yarn build
```

### 3. IDE 配置（VS Code）

```json
// .vscode/settings.json
{
  "typescript.preferences.includePackageJsonAutoImports": "on",
  "typescript.suggest.autoImports": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact"
  ],
  "files.exclude": {
    "node_modules": true,
    "dist": true,
    "build": true
  }
}

// .vscode/extensions.json
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "ms-vscode.vscode-typescript-next",
    "bradlc.vscode-tailwindcss",
    "ms-vscode.test-adapter-converter"
  ]
}
```

## 代码规范与架构

### 1. TypeScript 编程规范

```typescript
// ✅ 良好的接口设计
interface ElementOperations {
  createElement(type: ElementType, props: ElementProps): ExcalidrawElement;
  updateElement(
    element: ExcalidrawElement,
    updates: Partial<ExcalidrawElement>
  ): ExcalidrawElement;
  deleteElement(element: ExcalidrawElement): void;
}

// ✅ 使用严格的类型约束
type StrictElementType =
  | 'rectangle'
  | 'ellipse'
  | 'diamond'
  | 'line'
  | 'arrow'
  | 'text'
  | 'image'
  | 'freedraw';

// ✅ 函数式编程风格
const createElement = (
  type: StrictElementType,
  props: Partial<ExcalidrawElement>
): ExcalidrawElement => {
  return {
    id: generateId(),
    type,
    x: 0,
    y: 0,
    width: 100,
    height: 100,
    strokeColor: '#000000',
    backgroundColor: 'transparent',
    ...props, // 覆盖默认值
    // 确保关键属性不被覆盖
    versionNonce: getRandomInteger(),
    isDeleted: false,
    seed: Math.floor(Math.random() * 2 ** 31)
  };
};

// ✅ 不可变状态更新
const updateAppState = (
  currentState: AppState,
  updates: Partial<AppState>
): AppState => {
  return {
    ...currentState,
    ...updates,
    // 确保嵌套对象的不可变性
    selectedElementIds: updates.selectedElementIds
      ? { ...updates.selectedElementIds }
      : currentState.selectedElementIds
  };
};

// ❌ 避免的反模式
// 不要直接修改对象
function badUpdate(element: ExcalidrawElement): void {
  element.x = 100; // ❌ 直接修改
  element.y = 200; // ❌ 直接修改
}

// ❌ 避免使用 any 类型
function badFunction(data: any): any { // ❌
  return data.someProperty;
}
```

### 2. 组件设计原则

```typescript
// ✅ 组件职责单一
interface ToolbarProps {
  activeTool: Tool;
  onToolChange: (tool: Tool) => void;
  isReadOnly?: boolean;
}

const Toolbar: React.FC<ToolbarProps> = ({
  activeTool,
  onToolChange,
  isReadOnly = false
}) => {
  const tools = useTools();

  return (
    <div className="toolbar">
      {tools.map(tool => (
        <ToolButton
          key={tool.id}
          tool={tool}
          isActive={activeTool.id === tool.id}
          onClick={() => onToolChange(tool)}
          disabled={isReadOnly}
        />
      ))}
    </div>
  );
};

// ✅ 自定义 Hook 封装逻辑
const useElementSelection = (elements: ExcalidrawElement[]) => {
  const [selectedIds, setSelectedIds] = useState<Record<string, true>>({});

  const selectElement = useCallback((elementId: string) => {
    setSelectedIds(prev => ({
      ...prev,
      [elementId]: true
    }));
  }, []);

  const deselectElement = useCallback((elementId: string) => {
    setSelectedIds(prev => {
      const { [elementId]: removed, ...rest } = prev;
      return rest;
    });
  }, []);

  const selectedElements = useMemo(() =>
    elements.filter(el => selectedIds[el.id]),
    [elements, selectedIds]
  );

  return {
    selectedIds,
    selectedElements,
    selectElement,
    deselectElement,
    clearSelection: () => setSelectedIds({})
  };
};

// ✅ 错误边界处理
class ExcalidrawErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean; error?: Error }
> {
  constructor(props: { children: React.ReactNode }) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    // 发送错误报告
    console.error('Excalidraw Error:', error, errorInfo);

    // 可以集成错误监控服务
    if (process.env.NODE_ENV === 'production') {
      this.reportError(error, errorInfo);
    }
  }

  render() {
    if (this.state.hasError) {
      return <ErrorFallback error={this.state.error} />;
    }

    return this.props.children;
  }

  private reportError(error: Error, errorInfo: React.ErrorInfo): void {
    // 错误上报逻辑
  }
}
```

## 测试策略

### 1. 单元测试

```typescript
// test/element.test.ts
import { createElement, updateElement } from '../src/element';
import { ExcalidrawElement } from '../src/types';

describe('Element Operations', () => {
  describe('createElement', () => {
    it('should create element with default values', () => {
      const element = createElement('rectangle', {});

      expect(element).toMatchObject({
        type: 'rectangle',
        x: 0,
        y: 0,
        width: 100,
        height: 100,
        strokeColor: '#000000',
        backgroundColor: 'transparent'
      });

      expect(element.id).toBeDefined();
      expect(element.versionNonce).toBeDefined();
      expect(element.seed).toBeDefined();
    });

    it('should override default values with props', () => {
      const element = createElement('rectangle', {
        x: 50,
        y: 100,
        strokeColor: '#ff0000'
      });

      expect(element.x).toBe(50);
      expect(element.y).toBe(100);
      expect(element.strokeColor).toBe('#ff0000');
    });
  });

  describe('updateElement', () => {
    it('should create new element with updated properties', () => {
      const original = createElement('rectangle', { x: 0, y: 0 });
      const updated = updateElement(original, { x: 100, y: 200 });

      expect(updated).not.toBe(original); // 不可变性
      expect(updated.x).toBe(100);
      expect(updated.y).toBe(200);
      expect(updated.id).toBe(original.id); // ID 保持不变
    });

    it('should increment version nonce on update', () => {
      const original = createElement('rectangle', {});
      const updated = updateElement(original, { x: 100 });

      expect(updated.versionNonce).not.toBe(original.versionNonce);
    });
  });
});
```

### 2. 集成测试

```typescript
// test/excalidraw.integration.test.tsx
import { render, fireEvent, waitFor } from '@testing-library/react';
import { Excalidraw } from '../src/index';

describe('Excalidraw Integration', () => {
  it('should create rectangle on canvas click and drag', async () => {
    const onChange = jest.fn();
    const { container } = render(
      <Excalidraw onChange={onChange} />
    );

    const canvas = container.querySelector('canvas')!;

    // 选择矩形工具
    const rectangleTool = container.querySelector('[data-testid="rectangle-tool"]')!;
    fireEvent.click(rectangleTool);

    // 在画布上绘制矩形
    fireEvent.pointerDown(canvas, { clientX: 100, clientY: 100 });
    fireEvent.pointerMove(canvas, { clientX: 200, clientY: 200 });
    fireEvent.pointerUp(canvas, { clientX: 200, clientY: 200 });

    await waitFor(() => {
      expect(onChange).toHaveBeenCalled();
      const lastCall = onChange.mock.calls[onChange.mock.calls.length - 1];
      const [elements] = lastCall;

      expect(elements).toHaveLength(1);
      expect(elements[0].type).toBe('rectangle');
      expect(elements[0].x).toBe(100);
      expect(elements[0].y).toBe(100);
    });
  });

  it('should handle keyboard shortcuts', async () => {
    const { container } = render(<Excalidraw />);

    // 测试撤销快捷键
    fireEvent.keyDown(document, { key: 'z', ctrlKey: true });

    // 测试工具切换快捷键
    fireEvent.keyDown(document, { key: 'r' }); // 矩形工具

    await waitFor(() => {
      const activeTool = container.querySelector('[data-testid="active-tool"]');
      expect(activeTool).toHaveAttribute('data-tool', 'rectangle');
    });
  });
});
```

### 3. 性能测试

```typescript
// test/performance.test.ts
describe('Performance Tests', () => {
  it('should render 1000 elements within acceptable time', async () => {
    const elements: ExcalidrawElement[] = Array.from({ length: 1000 }, (_, i) =>
      createElement('rectangle', {
        x: Math.random() * 1000,
        y: Math.random() * 1000,
        width: 50,
        height: 50
      })
    );

    const startTime = performance.now();

    const { container } = render(
      <Excalidraw initialData={{ elements }} />
    );

    await waitFor(() => {
      expect(container.querySelector('canvas')).toBeInTheDocument();
    });

    const endTime = performance.now();
    const renderTime = endTime - startTime;

    // 渲染时间应该在合理范围内（比如 500ms）
    expect(renderTime).toBeLessThan(500);
  });

  it('should handle rapid pointer events without lag', async () => {
    const { container } = render(<Excalidraw />);
    const canvas = container.querySelector('canvas')!;

    const startTime = performance.now();

    // 模拟快速移动鼠标
    for (let i = 0; i < 100; i++) {
      fireEvent.pointerMove(canvas, {
        clientX: i * 2,
        clientY: i * 2
      });
    }

    const endTime = performance.now();
    const processingTime = endTime - startTime;

    // 事件处理时间应该很短
    expect(processingTime).toBeLessThan(100);
  });
});
```

## Git 工作流

### 1. 分支策略

```bash
# 主要分支
main              # 生产版本
develop           # 开发集成分支

# 功能开发分支
feature/tool-system
feature/collaboration
feature/export-pdf

# 修复分支
hotfix/security-fix
bugfix/selection-issue

# 发布分支
release/v1.2.0
```

### 2. 提交规范

```bash
# 提交格式
<type>(<scope>): <description>

[optional body]

[optional footer]

# 示例
feat(tools): add custom shape tool support

- Implement ShapeCreator class
- Add shape tool registry
- Update toolbar with new shape options

Closes #123

# 类型说明
feat     # 新功能
fix      # 修复bug
docs     # 文档更新
style    # 代码格式（不影响功能）
refactor # 代码重构
perf     # 性能优化
test     # 测试相关
chore    # 构建过程或辅助工具的变动

# 示例提交
git commit -m "feat(renderer): implement viewport culling optimization

- Add ViewportCuller class for efficient element filtering
- Implement bounding box intersection detection
- Reduce render calls by 60% for large canvases

Performance improvement for canvases with 1000+ elements"
```

### 3. 代码审查清单

```markdown
## Code Review Checklist

### 功能性 ✅
- [ ] 功能按预期工作
- [ ] 边界情况处理正确
- [ ] 错误处理完善
- [ ] 与现有功能集成良好

### 代码质量 ✅
- [ ] 代码可读性良好
- [ ] 遵循项目编码规范
- [ ] 适当的注释和文档
- [ ] 没有重复代码

### 性能 ✅
- [ ] 没有明显的性能问题
- [ ] 内存使用合理
- [ ] 算法复杂度可接受
- [ ] 异步操作处理正确

### 测试 ✅
- [ ] 有适当的单元测试
- [ ] 测试覆盖率合理
- [ ] 集成测试通过
- [ ] 手动测试验证

### 安全性 ✅
- [ ] 输入验证完善
- [ ] 没有安全漏洞
- [ ] 数据处理安全
- [ ] 权限控制正确

### 兼容性 ✅
- [ ] 浏览器兼容性测试
- [ ] 移动设备兼容性
- [ ] 向后兼容性保持
- [ ] API 接口稳定
```

## CI/CD 流程

### 1. GitHub Actions 配置

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [16.x, 18.x, 20.x]

    steps:
    - uses: actions/checkout@v3

    - name: Setup Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v3
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'yarn'

    - name: Install dependencies
      run: yarn install --frozen-lockfile

    - name: Run linting
      run: yarn lint

    - name: Run type checking
      run: yarn test:typecheck

    - name: Run tests
      run: yarn test --coverage

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/coverage-final.json

  build:
    runs-on: ubuntu-latest
    needs: test

    steps:
    - uses: actions/checkout@v3

    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18.x'
        cache: 'yarn'

    - name: Install dependencies
      run: yarn install --frozen-lockfile

    - name: Build packages
      run: yarn build

    - name: Build app
      run: yarn build:app

    - name: Run E2E tests
      run: yarn test:e2e

  deploy:
    runs-on: ubuntu-latest
    needs: [test, build]
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v3

    - name: Deploy to production
      run: |
        # 部署脚本
        echo "Deploying to production..."
```

### 2. 质量门禁

```yaml
# .github/workflows/quality-gate.yml
name: Quality Gate

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  quality-check:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0  # 获取完整历史用于分析

    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18.x'
        cache: 'yarn'

    - name: Install dependencies
      run: yarn install --frozen-lockfile

    - name: Run tests with coverage
      run: yarn test --coverage

    - name: Check coverage threshold
      run: |
        # 检查覆盖率是否达到 80%
        COVERAGE=$(node -p "require('./coverage/coverage-summary.json').total.lines.pct")
        if (( $(echo "$COVERAGE < 80" | bc -l) )); then
          echo "Coverage $COVERAGE% is below threshold 80%"
          exit 1
        fi

    - name: Check bundle size
      run: |
        yarn build
        # 检查包大小是否超过限制
        SIZE=$(stat -c%s "dist/excalidraw.min.js")
        if [ $SIZE -gt 1048576 ]; then  # 1MB
          echo "Bundle size $SIZE bytes exceeds 1MB limit"
          exit 1
        fi

    - name: Security audit
      run: yarn audit --level high
```

这套完整的开发工作流程确保了 Excalidraw 项目的代码质量、性能和可维护性。通过规范的代码风格、全面的测试覆盖和自动化的 CI/CD 流程，团队可以高效地协作开发和维护这个复杂的绘图应用。