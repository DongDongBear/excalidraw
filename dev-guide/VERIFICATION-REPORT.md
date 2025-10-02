# Excalidraw Dev-Guide Source Code Verification Report

**Date**: 2025-10-01
**Scope**: Chapters 20-27 (Extensibility & Workflow)
**Status**: CRITICAL ISSUES FOUND - REQUIRES MAJOR REVISIONS

---

## Executive Summary

After deep source code verification against Excalidraw's actual codebase (packages/excalidraw), **significant inaccuracies** were found in chapters 20-27. These chapters describe theoretical systems that **do not exist** in the current Excalidraw codebase.

### Critical Findings:

1. **Chapter 20 (Plugin Architecture)**: Describes a complete plugin system with PluginManager, sandboxing, and permission systems that **DOES NOT EXIST**
2. **Chapters 22-24 (Core Systems)**: Present theoretical implementations not found in actual code
3. **Chapters 25-27**: Mix accurate commands with fictional implementation details

---

## Detailed Verification Results

### Chapter 20: Plugin Architecture ❌ CRITICAL ISSUES

**Status**: ~90% inaccurate - describes non-existent systems

#### What the Document Claims:
- Complete plugin system with PluginManager class
- Plugin sandboxing with PluginSandbox class
- Permission system with PluginPermission enum
- ElementAPI, UIAPI, ActionAPI, ToolAPI classes
- Plugin lifecycle (onLoad, onEnable, onDisable, onUnload)

#### Actual Reality (Verified Sources):
```
Source: packages/excalidraw/types.ts
Source: packages/excalidraw/index.tsx
```

Excalidraw **DOES NOT** have a traditional plugin system. Extension is achieved through:

1. **React Props API** (VERIFIED):
   - `renderTopRightUI?: (isMobile: boolean, appState: UIAppState) => JSX.Element | null` (Line 574-577, types.ts)
   - `renderCustomStats?: (elements, appState) => JSX.Element` (Line 587-590, types.ts)
   - `UIOptions?: Partial<UIOptions>` (Line 591, types.ts)
   - `onChange?: (elements, appState, files) => void` (Line 538-542, types.ts)
   - `onLibraryChange?: (libraryItems) => void | Promise<any>` (Line 594, types.ts)

2. **UIOptions Type** (VERIFIED):
```typescript
// Source: packages/excalidraw/types.ts, Line 666-674
export type UIOptions = Partial<{
  dockedSidebarBreakpoint: number;
  canvasActions: CanvasActions;
  tools: {
    image: boolean;
  };
}>;
```

3. **ExcalidrawAPI** (via ref):
   - getSceneElements()
   - updateScene()
   - getAppState()
   - scrollToContent()
   - etc.

#### Required Actions:
- **REWRITE ENTIRE CHAPTER** to focus on actual extensibility mechanisms
- Remove all references to non-existent plugin classes
- Document actual Props API with correct type signatures
- Add real-world examples using actual APIs

---

### Chapter 21: Import/Export System ✅ MOSTLY ACCURATE

**Status**: ~70% accurate - APIs exist but some details incorrect

#### Verified Functions (Source: packages/excalidraw/scene/export.ts, data/index.ts, data/blob.ts):

**Export Functions** ✅ VERIFIED:
```typescript
// Line 163-275, scene/export.ts
exportToCanvas(
  elements: readonly NonDeletedExcalidrawElement[],
  appState: AppState,
  files: BinaryFiles,
  options: {
    exportBackground: boolean;
    viewBackgroundColor: string;
    exportPadding?: number;
    exportingFrame?: ExcalidrawFrameLikeElement | null;
  }
): HTMLCanvasElement

// Line 284-397, scene/export.ts
exportToSvg(
  elements: readonly NonDeletedExcalidrawElement[],
  appState: {
    exportBackground: boolean;
    exportPadding?: number;
    exportScale?: number;
    viewBackgroundColor: string;
    exportWithDarkMode?: boolean;
    exportEmbedScene?: boolean;
    frameRendering?: AppState["frameRendering"];
  },
  files: BinaryFiles | null,
  opts?: { exportingFrame?: ExcalidrawFrameLikeElement | null }
): Promise<SVGSVGElement>
```

**Import Functions** ✅ VERIFIED:
```typescript
// Line 197-215, data/blob.ts
loadFromBlob(
  blob: Blob,
  localAppState: AppState | null,
  localElements: readonly ExcalidrawElement[] | null,
  fileHandle?: FileSystemHandle | null
): Promise<RestoredData>

// Line 229-234, data/blob.ts
loadLibraryFromBlob(
  blob: Blob,
  defaultStatus: LibraryItem["status"] = "unpublished"
): Promise<LibraryItem[]>
```

**MIME_TYPES** ✅ VERIFIED:
```typescript
// Source: @excalidraw/common package
MIME_TYPES = {
  excalidraw: "application/vnd.excalidraw+json",
  excalidrawlib: "application/vnd.excalidrawlib+json",
  json: "application/json",
  svg: "image/svg+xml",
  png: "image/png",
  jpg: "image/jpeg",
  gif: "image/gif",
  webp: "image/webp",
  binary: "application/octet-stream",
  pdf: "application/pdf"
}
```

#### Issues Found:
1. Some export option names differ (e.g., `exportingFrame` not `exportFrame`)
2. Return types need clarification (Canvas vs Promise)
3. Missing mention of `exportCanvas()` wrapper function in data/index.ts

#### Required Actions:
- Update function signatures to match exact source
- Add source file references for each API
- Clarify sync vs async behaviors
- Update code examples with correct imports

---

### Chapter 22: Minimal Core ⚠️ PARTIALLY ACCURATE

**Status**: ~40% accurate - concepts correct but implementations fictional

#### What's Correct:
- General architecture concepts (state, render, events)
- Data flow principles (unidirectional)
- Core responsibilities separation

#### What's Wrong:
- Specific class names don't exist (MinimalExcalidrawCore, MinimalCanvas, etc.)
- Actual implementation is significantly more complex
- React-based, not vanilla JS classes

#### Actual Core Files (VERIFIED):
```
packages/excalidraw/
├── components/App.tsx          # Main component (2000+ lines)
├── scene/                      # Rendering system
│   ├── export.ts              # Export functionality
│   └── Scene.tsx              # Scene management
├── renderer/
│   ├── renderScene.ts         # Main render loop
│   └── staticScene.ts         # Static rendering
├── element/                    # Element system
└── actions/                    # Action system
```

#### Required Actions:
- Clarify this is conceptual, not actual code
- Reference actual source files
- Update examples to reflect React-based architecture
- Add note about complexity (2000+ line components)

---

### Chapter 23: Performance Core ⚠️ PARTIALLY ACCURATE

**Status**: ~50% accurate - concepts exist but specific implementations vary

#### What's Correct:
- General optimization strategies (viewport culling, caching, batching)
- Performance monitoring concepts
- Memory management principles

#### What's Wrong:
- Specific class names fictional (OptimizedRenderer, ViewportCuller, etc.)
- Actual implementation details differ significantly
- Missing references to actual performance code

#### Actual Performance Code (PARTIALLY VERIFIED):
```
packages/excalidraw/
├── scene/Shape.ts              # Shape caching
├── renderer/renderElement.ts   # Element rendering optimizations
└── scene/scroll.ts             # Viewport management
```

#### Required Actions:
- Mark as design patterns, not actual code
- Reference real optimization code locations
- Update with actual performance metrics
- Add links to GitHub performance issues

---

### Chapter 24: Extensibility Core ⚠️ PARTIALLY ACCURATE

**Status**: ~40% accurate - describes non-existent plugin APIs

Same issues as Chapter 20 - describes theoretical plugin systems not present in codebase.

#### Required Actions:
- Redirect to Chapter 20's rewrite
- Focus on actual extension mechanisms (Props, callbacks)
- Remove fictional plugin APIs
- Document actual integration patterns

---

### Chapter 25: Development Workflow ✅ MOSTLY ACCURATE

**Status**: ~80% accurate - commands verified but some details need updates

#### Verified Commands:
```bash
✅ yarn install          # Verified in package.json
✅ yarn start           # Verified - starts dev server
✅ yarn test            # Verified - runs tests
✅ yarn test:typecheck  # Verified - TypeScript checking
✅ yarn build           # Verified - builds project
✅ yarn fix             # Verified - runs linting/formatting
```

#### Issues Found:
1. `yarn test:update` command not found in package.json
2. Some file paths in project structure need verification
3. Git workflow section is generic, not Excalidraw-specific

#### Required Actions:
- Verify all commands against package.json scripts
- Update project structure to match current repo
- Add Excalidraw-specific workflow details from CONTRIBUTING.md

---

### Chapter 26: Debugging & Optimization ⚠️ PARTIALLY ACCURATE

**Status**: ~60% accurate - tools exist but custom implementations are fictional

#### What's Correct:
- Browser DevTools usage
- React DevTools integration
- Performance profiling concepts
- Error boundary patterns

#### What's Wrong:
- ExcalidrawDebugger class doesn't exist
- Custom performance monitoring tools are fictional
- Specific debugging APIs not verified

#### Actual Debugging Support (VERIFIED):
```
packages/excalidraw/
├── components/App.tsx          # Error boundaries present
└── constants.ts                # Debug flags (IS_DEVELOPMENT, etc.)
```

#### Required Actions:
- Mark custom classes as example implementations
- Document actual debug flags and environment variables
- Reference browser DevTools instead of custom tools
- Add real error tracking integration examples

---

### Chapter 27: Ecosystem Integration ⚠️ PARTIALLY ACCURATE

**Status**: ~50% accurate - integration patterns correct but examples need verification

#### What's Correct:
- General integration approaches (Next.js, Electron, React Native)
- Cloud service concepts
- WebSocket collaboration pattern

#### What's Wrong:
- Specific API usage may have changed
- Import paths need verification
- Some examples use deprecated patterns

#### Required Actions:
- Verify all imports against current @excalidraw/excalidraw package
- Update examples with current API signatures
- Test integration examples
- Add links to official examples/ directory

---

## Recommended Action Plan

### Priority 1: Critical Rewrites
1. **Chapter 20**: Complete rewrite focusing on actual Props API
2. **Chapter 24**: Align with Chapter 20 rewrite

### Priority 2: Major Updates
3. **Chapter 22**: Add disclaimers, reference actual code locations
4. **Chapter 23**: Mark as patterns, add real code references
5. **Chapter 21**: Update signatures, add source references

### Priority 3: Minor Corrections
6. **Chapter 25**: Verify commands, update structure
7. **Chapter 26**: Mark examples as theoretical, add real debug info
8. **Chapter 27**: Update APIs, verify examples

---

## Verification Methodology

### Sources Examined:
1. **Primary Source**: `/Users/dongdong/Desktop/excalidraw/packages/excalidraw/`
   - types.ts (ExcalidrawProps, UIOptions, AppState)
   - index.tsx (ExcalidrawBase component)
   - scene/export.ts (export functions)
   - data/*.ts (import/blob/json functions)

2. **Verification Tools**:
   - grep searches for API names
   - Direct file reading
   - TypeScript type definitions
   - Package.json scripts verification

3. **Search Patterns**:
   ```bash
   # APIs verified
   grep -r "UIOptions\|renderTopRightUI\|renderFooter" packages/excalidraw/types.ts
   grep -r "exportToCanvas\|exportToBlob\|exportToSvg" packages/excalidraw/scene/
   grep -r "loadFromBlob\|loadLibraryFromBlob" packages/excalidraw/data/
   grep -r "MIME_TYPES" packages/excalidraw/
   ```

---

## Key Learnings

1. **Documentation Drift**: These chapters appear to describe an idealized/planned architecture rather than actual implementation
2. **Plugin System Myth**: Major confusion around "plugin system" - Excalidraw uses React props-based extension
3. **Code vs. Concepts**: Mix of actual code references and theoretical implementations
4. **Source Attribution**: Lack of source file references makes verification difficult

---

## Recommendations for Future Documentation

1. **Always include source file references** in format:
   ```
   Source: packages/excalidraw/types.ts (Line 574-577)
   ```

2. **Distinguish clearly** between:
   - ✅ Actual APIs (with verified signatures)
   - 💡 Design patterns (theoretical)
   - 📋 Example implementations (not production code)

3. **Link to actual code**:
   ```
   See: https://github.com/excalidraw/excalidraw/blob/master/packages/excalidraw/types.ts#L574
   ```

4. **Include verification dates**: "Verified against v0.17.0 (2025-01-01)"

5. **Test all code examples** in actual environment before documenting

---

## Conclusion

While these chapters provide valuable conceptual frameworks, they **require significant revision** to accurately reflect Excalidraw's actual implementation. The most critical issue is the complete plugin architecture described in Chapter 20, which does not exist in the codebase.

**Recommendation**: Prioritize rewriting chapters 20 and 24, then systematically update others with source references and accurate API signatures.

**Next Steps**:
1. Get feedback on this verification report
2. Create rewrite plan for each chapter
3. Establish source code referencing standards
4. Set up automated verification where possible

---

**Verified by**: Claude (Sonnet 4.5)
**Repository**: https://github.com/excalidraw/excalidraw
**Branch**: master
**Commit**: 369dfb92 (latest at verification time)
