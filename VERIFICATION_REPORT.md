# Deep Source Code Verification Report

## Summary of Findings

This report documents the verification of three dev-guide chapters against actual Excalidraw source code.

---

## Chapter 17: Action System (`17-action-system.md`)

### ✅ VERIFIED - Accurate Sections:
1. **Action Interface** - Correctly documented with all fields
2. **ActionResult type** - Accurate definition
3. **CaptureUpdateActionType** - Correct enum values

### ❌ ISSUES FOUND:

#### 1. ActionName Enum - INCOMPLETE
**Current**: Lists ~65 actions
**Actual**: 147 total actions in `packages/excalidraw/actions/types.ts`

**Missing Actions**:
- `gridMode`, `zenMode`, `objectsSnapMode`, `stats`
- `changeArrowProperties`, `changeRoundness`
- `increaseFontSize`, `decreaseFontSize`
- `unbindText`, `bindText`, `createContainerFromText`, `autoResize`
- `toggleFullScreen`, `toggleShortcuts`
- `exportWithDarkMode`, `unlockAllElements`
- `zoomToFitSelectionInViewport`
- `alignVerticallyCentered`, `alignHorizontallyCentered`
- `copyElementLink`, `linkToElement`
- `goToCollaborator`, `addToLibrary`
- `selectAllElementsInFrame`, `removeAllElementsFromFrame`
- `updateFrameRendering`, `setFrameAsActiveTool`, `wrapSelectionInFrame`
- `setEmbeddableAsActiveTool`
- `elementStats`, `searchMenu`, `cropEditor`
- `toggleShapeSwitch`

**Status**: ✅ FIXED - Updated with complete list

#### 2. ActionManager Class - INCORRECT IMPLEMENTATION
**Issue**: Documentation shows a fictional implementation with:
- `Map<ActionName, ExcalidrawAction>` storage (WRONG - uses `Record`)
- `HistoryManager` and `BroadcastManager` (DON'T EXIST in ActionManager)
- `executeAction` method with history management (NOT IN SOURCE)
- `registerDefaultActions` method (DOESN'T EXIST)

**Actual Implementation** (`packages/excalidraw/actions/manager.tsx`):
```typescript
export class ActionManager {
  actions = {} as Record<ActionName, Action>;  // NOT Map!
  updater: (actionResult: ActionResult | Promise<ActionResult>) => void;
  getAppState: () => Readonly<AppState>;
  getElementsIncludingDeleted: () => readonly OrderedExcalidrawElement[];
  app: AppClassProperties;

  constructor(
    updater: UpdaterFn,
    getAppState: () => AppState,
    getElementsIncludingDeleted: () => readonly OrderedExcalidrawElement[],
    app: AppClassProperties,
  )

  registerAction(action: Action)
  registerAll(actions: readonly Action[])
  handleKeyDown(event: React.KeyboardEvent | KeyboardEvent)
  executeAction<T extends Action>(action: T, source?: ActionSource, value?: any)
  renderAction(name: ActionName, data?: PanelComponentProps["data"])
  isActionEnabled(action: Action)
}
```

**Key Differences**:
1. No history management in ActionManager
2. No broadcast manager
3. Uses `updater` callback pattern instead of direct execution
4. `handleKeyDown` method for keyboard shortcuts
5. `renderAction` for rendering action panels
6. Actions registered via `registerAll` from external source

**Status**: ⚠️ NEEDS MAJOR REWRITE

#### 3. Action Registration - INCORRECT
**Documentation shows**: Individual registration in constructor
**Actual**: Uses `register.ts` module pattern:
```typescript
// packages/excalidraw/actions/register.ts
export let actions: readonly Action[] = [];

export const register = <T extends Action>(action: T) => {
  actions = actions.concat(action);
  return action;
};
```

Actions are registered in individual action files, then imported and registered via `registerAll()`.

**Status**: ⚠️ NEEDS CORRECTION

---

## Chapter 18: History Management (`18-history-management.md`)

### ✅ VERIFIED - Mostly Accurate:
1. **HistoryDelta class** - Correctly documented
2. **History class structure** - Accurate
3. **Stack-based architecture** - Correct

### ❌ ISSUES FOUND:

#### 1. History Class - SIMPLIFIED vs DOCUMENTED
**Issue**: Documentation shows an overly complex `HistoryManager` with:
- Snapshot compression
- Delta calculation
- Persistence management
- Auto-save functionality

**Actual Implementation** (`packages/excalidraw/history.ts`):
```typescript
export class History {
  public readonly onHistoryChangedEmitter = new Emitter<[HistoryChangedEvent]>();
  public readonly undoStack: HistoryDelta[] = [];
  public readonly redoStack: HistoryDelta[] = [];

  constructor(private readonly store: Store) {}

  public record(delta: StoreDelta)
  public undo(elements: SceneElementsMap, appState: AppState)
  public redo(elements: SceneElementsMap, appState: AppState)
  public clear()
  private perform(...)
  private static pop(stack: HistoryDelta[]): HistoryDelta | null
  private static push(stack: HistoryDelta[], entry: HistoryDelta)
}
```

**Key Points**:
- Much simpler than documented
- NO compression built-in
- NO persistence in core History class
- NO auto-save mechanism
- Uses `Store` reference for snapshots
- Emits `HistoryChangedEvent` for UI updates

**Status**: ⚠️ DOCUMENTATION IS OVERENGINEERED - Shows "advanced" features not in core

#### 2. Missing Core Details:
- `Store` integration not explained
- `StoreSnapshot` usage not covered
- How `HistoryDelta.inverse()` works
- Version/versionNonce exclusion in collaboration context

**Status**: ⚠️ NEEDS ADDITIONAL DETAILS

---

## Chapter 19: Collaboration System (`19-collaboration-system.md`)

### ✅ VERIFIED - Partially Accurate:
1. **Collaborator type** - Mostly correct
2. **CollaboratorPointer** - Correct structure
3. **WebSocket communication** - Generally accurate concept

### ❌ ISSUES FOUND:

#### 1. Collaborator Type - MISSING FIELDS
**Documentation shows**:
```typescript
export interface Collaborator {
  socketId: string;
  id?: string;
  username?: string;
  avatarUrl?: string;
  color?: { background: string; stroke: string };
  pointer?: CollaboratorPointer;
  button?: "down" | "up";
  selectedElementIds?: AppState["selectedElementIds"];
  userState?: UserIdleState;
  isCurrentUser?: boolean;
  isInCall?: boolean;
}
```

**Actual** (`packages/excalidraw/types.ts`):
```typescript
export type Collaborator = Readonly<{
  pointer?: CollaboratorPointer;
  button?: "up" | "down";
  selectedElementIds?: AppState["selectedElementIds"];
  username?: string | null;  // Note: can be null!
  userState?: UserIdleState;
  color?: { background: string; stroke: string };
  avatarUrl?: string;
  id?: string;
  socketId?: SocketId;  // SocketId is a branded type!
  isCurrentUser?: boolean;
  isInCall?: boolean;
  isSpeaking?: boolean;  // MISSING!
  isMuted?: boolean;     // MISSING!
}>;
```

**Missing Fields**:
- `isSpeaking` - For voice collaboration
- `isMuted` - For voice collaboration
- Type is `Readonly<>` not mutable interface
- `socketId` is `SocketId` branded type, not `string`

**Status**: ⚠️ NEEDS UPDATE

#### 2. CollaboratorPointer - MISSING FIELDS
**Documentation shows**:
```typescript
export interface CollaboratorPointer {
  x: number;
  y: number;
  tool: "pointer" | ToolType;
}
```

**Actual**:
```typescript
export type CollaboratorPointer = {
  x: number;
  y: number;
  tool: "pointer" | "laser";  // Only 2 options, not full ToolType!
  renderCursor?: boolean;     // MISSING!
  laserColor?: string;        // MISSING!
};
```

**Status**: ⚠️ NEEDS UPDATE

#### 3. CollabManager - FICTIONAL IMPLEMENTATION
**Issue**: The documented `CollabManager` class doesn't exist!

**Actual Implementation**: Located in `excalidraw-app/collab/Collab.tsx`:
- Named `Collab` (not `CollabManager`)
- React PureComponent (not plain class)
- Uses `Portal` class for WebSocket communication
- No `OperationBuffer`, `ConflictResolver`, or `PresenceTracker` classes
- Uses Firebase for persistence
- Uses Jotai atoms for state management

**Real Architecture**:
```
excalidraw-app/collab/
├── Collab.tsx          # Main collaboration component
├── Portal.tsx          # WebSocket communication
└── CollabError.tsx     # Error handling
```

**Status**: ❌ MAJOR ISSUE - Documented fictional architecture

#### 4. Operational Transform (OT) - NOT IMPLEMENTED
**Issue**: Documentation shows detailed OT algorithm implementation.

**Reality**: Excalidraw uses:
- `reconcileElements()` for conflict resolution
- Version numbers and timestamps
- Last-write-wins strategy
- NO formal OT implementation
- NO vector clocks in core

**Status**: ❌ FICTIONAL - Shows theoretical implementation not in codebase

#### 5. Actual Collaboration Flow:
```typescript
// excalidraw-app/collab/Portal.tsx
class Portal {
  socket: Socket | null
  roomId: string | null
  roomKey: string | null
  broadcastedElementVersions: Map<string, number>

  broadcastScene(updateType, elements, syncAll)
  broadcastIdleChange(userState)
  queueFileUpload()  // Throttled file upload
}

// excalidraw-app/collab/Collab.tsx
class Collab extends PureComponent {
  portal: Portal
  fileManager: FileManager
  collaborators: Map<SocketId, Collaborator>

  startCollaboration(roomData)
  stopCollaboration()
  syncElements(elements)
  onPointerUpdate(payload)
}
```

**Status**: ⚠️ NEEDS COMPLETE REWRITE based on actual implementation

---

## Recommendations

### Priority 1 - Critical Fixes:
1. ✅ **Chapter 17**: Expand ActionName enum (DONE)
2. ⚠️ **Chapter 17**: Rewrite ActionManager implementation to match actual source
3. ⚠️ **Chapter 17**: Document actual action registration pattern
4. ⚠️ **Chapter 19**: Update Collaborator and CollaboratorPointer types
5. ❌ **Chapter 19**: Completely rewrite collaboration system based on actual Collab/Portal implementation

### Priority 2 - Enhancements:
1. **Chapter 18**: Clarify which features are "advanced concepts" vs actual implementation
2. **Chapter 18**: Add Store/StoreSnapshot integration details
3. **Chapter 19**: Remove fictional OT implementation, document actual reconciliation
4. **Chapter 19**: Add Firebase integration details
5. **Chapter 19**: Document Jotai atoms for collab state

### Priority 3 - Documentation Structure:
1. Separate "Core Implementation" from "Advanced Patterns"
2. Add source file references for all code samples
3. Mark theoretical implementations clearly
4. Add version compatibility notes

---

## Files Verified

### Source Code Files:
- ✅ `/packages/excalidraw/actions/types.ts`
- ✅ `/packages/excalidraw/actions/manager.tsx`
- ✅ `/packages/excalidraw/actions/register.ts`
- ✅ `/packages/excalidraw/history.ts`
- ✅ `/packages/excalidraw/types.ts` (Collaborator types)
- ✅ `/excalidraw-app/collab/Collab.tsx`
- ✅ `/excalidraw-app/collab/Portal.tsx`

### Documentation Files:
- ⚠️ `/dev-guide/17-action-system.md` - Partially accurate
- ⚠️ `/dev-guide/18-history-management.md` - Mostly accurate
- ❌ `/dev-guide/19-collaboration-system.md` - Majorly inaccurate

---

## Next Steps

1. ✅ Fix ActionName enum completeness
2. Rewrite ActionManager section with actual implementation
3. Update Collaborator types with missing fields
4. Rewrite collaboration system chapter based on Collab/Portal
5. Add clear labels for "theoretical" vs "actual" implementations
6. Add source code references throughout

---

Generated: 2025-10-01
Verified by: Claude (Sonnet 4.5)
Codebase: Excalidraw (commit: 369dfb92)
