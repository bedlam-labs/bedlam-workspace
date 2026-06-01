# Refactor Plan: Consolidate Dev Server + Viewer Architecture

## Goal

Eliminate the layered HTTP/SSE/WebSocket-proxy mess by:
1. Merging prismarine-viewer's server into dev-server's HTTP server (one port, one socket.io instance)
2. Replacing SSE with socket.io events on a named namespace
3. Breaking `dev-server.ts` (1211 LOC god object) into focused classes
4. Removing the `TEST_SERVER_PORT` env-var visualizer coupling in `setup.ts`
5. Deduplicating `applyLayout` (currently in both `setup.ts` and `dev-server.ts`)

---

## Current Architecture

```
Browser
  ├── fetch() + EventSource → adminPort        (REST + SSE)
  └── socket.io → adminPort/viewer → PROXY → viewerPort  (3D viewer)

Ports (relative to `port`, e.g. 25600):
  port        = MC server (flying-squid)
  port + 1000 = viewerPort (prismarine-viewer Express + socket.io)
  port + 2000 = adminPort  (dev-server HTTP + WS proxy)
```

The proxy code lives at:
- `packages/mineflayer-test-suite/src/dev-server.ts:1027–1055`
  - HTTP GET `/viewer*` → forward to `viewerPort`
  - `server.on('upgrade')` → pipe WebSocket to `viewerPort`

---

## Target Architecture

```
Browser
  ├── fetch() → adminPort              (REST API, unchanged paths)
  └── socket.io /editor ns → adminPort (replaces SSE)
  └── socket.io /viewer  ns → adminPort (replaces proxy, same connection)

Ports:
  port        = MC server
  adminPort   = unified HTTP + socket.io (viewerPort eliminated)
```

---

## File Map

All changes are in two packages:

| Package | Files Touched |
|---|---|
| `packages/mineflayer-test-suite/src/` | `dev-server.ts` (major), `setup.ts` (minor) |
| `packages/prismarine-viewer/lib/` | `mineflayer.js` (API change only) |

New files to create (all in `packages/mineflayer-test-suite/src/`):
- `WorldEditor.ts`
- `ScenarioManager.ts`
- `MarkerManager.ts`
- `TestRunner.ts`
- `ChunkRelay.ts` (shared applyLayout + resend logic)

---

## Change 1 — `prismarine-viewer/lib/mineflayer.js`

**Current behavior:** Creates its own `http.Server` + `socket.io` instance, listens on `port`.

**New behavior:** Additive — support both the original `port`-based standalone mode and a new `httpServer` + `namespace` integrated mode. Old callers are unaffected but cleaned up.

### API change (backward-compatible overload)

```js
// packages/prismarine-viewer/lib/mineflayer.js:4
// Replace the destructured parameter signature with `options` object, then branch:

module.exports = (bot, options) => {
  const { viewDistance = 6, firstPerson = false, prefix = '' } = options;

  let httpServer, ns;

  if (options.httpServer) {
    // Integrated mode: caller owns the HTTP server (used by dev-server)
    // No Express setup, no http.listen — viewer piggybacks on caller's server
    httpServer = options.httpServer;
    const io = require('socket.io')(httpServer);
    ns = options.namespace ? io.of(options.namespace) : io;
  } else {
    // Standalone mode: original behavior, unchanged
    const express = require('express');
    const app = express();
    const { setupRoutes } = require('./common');
    setupRoutes(app, prefix);
    httpServer = require('http').createServer(app);
    const io = require('socket.io')(httpServer, { path: prefix + '/socket.io' });
    ns = io;
    httpServer.listen(options.port ?? 3000, () => {
      console.log(`Prismarine viewer web server running on *:${options.port ?? 3000}`);
    });
  }

  // rest of function unchanged EXCEPT:
  // replace `io.on('connection', ...)` → `ns.on('connection', ...)`
  // bot.viewer.close(): in integrated mode skip http.close() — caller owns it
}
```

### Specific line changes in `mineflayer.js`

| Lines | What to change |
|---|---|
| 4 | Change destructured params to `options` object; add branch as above |
| 5–13 | Wrap Express/http setup in `else` block (standalone path only) |
| 10 | In integrated path, bind `io` to `options.httpServer`; get `ns` via `io.of(namespace)` |
| 80 | `io.on('connection'` → `ns.on('connection'` |
| 141–143 | Move `http.listen(...)` into standalone branch only |
| 145–150 | In `bot.viewer.close`: guard `http.close()` with `if (!options.httpServer)` |

`setupRoutes` (static file serving from `/public`) stays in the standalone path. In integrated mode, dev-server serves its own UI — no conflict since paths don't overlap.

---

## Change 2 — New file: `ChunkRelay.ts`

**Purpose:** Single source of truth for layout application + chunk resend. Eliminates duplication between `setup.ts:32–54` and `dev-server.ts:21–68`.

```ts
// packages/mineflayer-test-suite/src/ChunkRelay.ts

import { Vec3 } from 'vec3';
import type { WorldLayout } from './types';

export function stateIdOf(mcData: any, PBlock: any, blockName: string): number {
  const info = mcData.blocksByName[blockName];
  return new PBlock(info.id, 0, 0).stateId ?? (info.id << 4);
}

export async function applyLayout(
  server: any,
  world: any,
  layout: WorldLayout,
  mcData: any,
  PBlock: any,
  dirtyPositions: Array<{ x: number; y: number; z: number }>
): Promise<void> {
  // move the full body of the current dev-server.ts:21-68 here
  // dirtyPositions is passed in/mutated by reference (or returned)
}

export function resendChunks(
  server: any,
  world: any,
  positions: Array<{ x: number; z: number }>
): void {
  // move dev-server.ts:243-255 here
}
```

**Then in `setup.ts`:** Delete `applyLayoutStandalone` (lines 32–54) and call `applyLayout` from `ChunkRelay.ts` instead (no dirty tracking needed in standalone mode — pass a throwaway array).

**Then in `dev-server.ts`:** Delete the top-level `applyLayout` and `stateIdOf` functions (lines 16–68) and import from `ChunkRelay.ts`.

---

## Change 3 — New file: `WorldEditor.ts`

**Purpose:** Encapsulate all world mutation, undo/redo, clipboard, dirty tracking. Currently scattered across `dev-server.ts:199–588`.

```ts
// packages/mineflayer-test-suite/src/WorldEditor.ts

import { Vec3 } from 'vec3';
import { stateIdOf, resendChunks } from './ChunkRelay';

export interface UndoEntry {
  undo: Array<{ pos: { x: number; y: number; z: number }; stateId: number }>;
  redo: Array<{ pos: { x: number; y: number; z: number }; stateId: number }>;
}

export class WorldEditor {
  private dirtyPositions: Array<{ x: number; y: number; z: number }> = [];
  private undoStack: UndoEntry[] = [];
  private redoStack: UndoEntry[] = [];
  private clipboard: Array<{ dx: number; dy: number; dz: number; stateId: number }> | null = null;

  constructor(
    private world: any,
    private server: any,
    private mcData: any,
    private PBlock: any,
    private onChanged: () => void   // callback → triggers broadcastEditState
  ) {}

  get undoSize() { return this.undoStack.length; }
  get redoSize() { return this.redoStack.length; }
  get hasClipboard() { return this.clipboard !== null; }
  get clipboardSize() { return this.clipboard?.length ?? 0; }
  getDirtyPositions() { return this.dirtyPositions; }
  clearDirty() { this.dirtyPositions = []; }

  // Move these methods verbatim from dev-server.ts, replacing server/world/mcData/PBlock
  // with `this.*` references and calling `this.onChanged()` in place of broadcastEditState():
  async place(pos: Vec3, block: string, radius?: number): Promise<void> { ... }
  async break(pos: Vec3, radius?: number): Promise<void> { ... }
  async fill(pos1: Vec3, pos2: Vec3): Promise<void> { ... }
  async replace(pos1: Vec3, pos2: Vec3, fromBlock: string, toBlock: string): Promise<void> { ... }
  async copy(pos1: Vec3, pos2: Vec3): Promise<void> { ... }
  async paste(anchor: Vec3): Promise<void> { ... }
  async translateSelection(pos1: Vec3, pos2: Vec3, dx: number, dy: number, dz: number): Promise<{ pos1: Vec3; pos2: Vec3 }> { ... }
  async rotateSelection(pos1: Vec3, pos2: Vec3, axis: string, angle: number): Promise<{ pos1: Vec3; pos2: Vec3 }> { ... }
  scaleSelection(pos1: Vec3, pos2: Vec3, axis: string, amount: number): { pos2: Vec3 } { ... }
  async undo(): Promise<void> { ... }
  async redo(): Promise<void> { ... }
  async captureLayout(): Promise<WorldLayout> { ... }
  clearHistory(): void { this.undoStack.length = 0; this.redoStack.length = 0; }
}
```

**Source material in `dev-server.ts`:**
- `stateIdOf` → moved to `ChunkRelay.ts`
- `getBlockStateId` (lines 318–325) → private method
- `pushUndo` (lines 327–330) → private method
- `doBreak` (lines 334–346) → `break()`
- `doPlace` (lines 348–361) → `place()`
- `radiusPositions` (lines 363–377) → private method
- `selectionPositions` (lines 379–390) → stays in `DevServer` since it needs pos1/pos2 state, OR move pos1/pos2 into WorldEditor
- `doFill` (lines 392–406) → `fill()`
- `doReplace` (lines 408–425) → `replace()`
- `doCopy` (lines 427–439) → `copy()`
- `doPaste` (lines 441–457) → `paste()`
- `doTranslateSelection` (lines 461–497) → `translateSelection()` — returns new pos1/pos2
- `doScaleSelection` (lines 499–508) → `scaleSelection()` — returns new pos2
- `doRotateSelection` (lines 510–568) → `rotateSelection()` — returns new pos1/pos2
- `doUndo` (lines 570–578) → `undo()`
- `doRedo` (lines 580–588) → `redo()`
- `captureLayout` (lines 650–664) → `captureLayout()`

---

## Change 4 — New file: `ScenarioManager.ts`

**Purpose:** Scenario CRUD + favorites. Currently `dev-server.ts:590–664`.

```ts
// packages/mineflayer-test-suite/src/ScenarioManager.ts

export class ScenarioManager {
  constructor(private projectCwd: string | undefined) {}

  list(): ScenarioSummary[] { ... }
  save(layout: WorldLayout, name: string, meta: ScenarioMetadata, markers: Marker[]): void { ... }
  load(name: string): Scenario | null { ... }
  toggleFavorite(name: string): boolean { ... }

  // private
  private scenariosDir(): string | null { ... }
  private favoritesPath(): string | null { ... }
  private loadFavorites(): Set<string> { ... }
  private saveFavorites(favs: Set<string>): void { ... }
}
```

**Source material:** `dev-server.ts:590–664` verbatim.

---

## Change 5 — New file: `MarkerManager.ts`

**Purpose:** Marker state + 3D rendering sync. Currently `dev-server.ts:272–314` (logic) + scattered draw calls.

```ts
// packages/mineflayer-test-suite/src/MarkerManager.ts

export class MarkerManager {
  private markers: Marker[] = [];
  private drawnIds = new Set<string>();

  constructor(
    private getViewer: () => any,   // () => observerBot.viewer
    private onChanged: () => void
  ) {}

  getAll(): Marker[] { return this.markers; }
  add(name: string, position: { x: number; y: number; z: number }): Marker { ... }
  update(id: string, position: { x: number; y: number; z: number }): boolean { ... }
  remove(id: string): boolean { ... }
  replaceAll(markers: Marker[]): void { ... }  // used on scenario load
  syncAll(): void { ... }  // erase + redraw all

  // private
  private draw(m: Marker): void { ... }
  private erase(id: string): void { ... }
  private color(name: string): number { ... }
  private colorStr(name: string): string { ... }
}
```

**Source material:** `dev-server.ts:272–314`.

---

## Change 6 — New file: `TestRunner.ts`

**Purpose:** Child process spawn + output streaming. Currently `dev-server.ts:668–689`.

```ts
// packages/mineflayer-test-suite/src/TestRunner.ts

export class TestRunner {
  private activeTest: ChildProcess | null = null;

  constructor(
    private resolveRunner: (suite: string) => { script: string; cwd: string; env?: Record<string, string> },
    private mcPort: number,
    private onOutput: (line: string) => void,
    private onDone: (code: number) => void
  ) {}

  run(suite: string, testName?: string): boolean { ... }  // returns false if already running
  isRunning(): boolean { return this.activeTest !== null; }
}
```

**Source material:** `dev-server.ts:668–689`.

---

## Change 7 — Rewrite `dev-server.ts`

After the above extractions, `dev-server.ts` becomes a thin coordinator:

```ts
// packages/mineflayer-test-suite/src/dev-server.ts (skeleton after refactor)

import * as http from 'http';
import { Server as SocketIOServer } from 'socket.io';
import { WorldEditor } from './WorldEditor';
import { ScenarioManager } from './ScenarioManager';
import { MarkerManager } from './MarkerManager';
import { TestRunner } from './TestRunner';
import { applyLayout } from './ChunkRelay';
const { mineflayer: attachViewer } = require('@aidentran900/prismarine-viewer');

export function startDevServer(options: DevServerOptions) {
  const { port, adminPort = port + 2000, version = '1.8.8', viewDistance = 4, cwd, testRunner } = options;
  // NOTE: viewerPort is GONE — viewer is on same server

  const mcData = require('minecraft-data')(version);
  const PBlock = require('prismarine-block')(version);

  // ---- Tool/selection state (lightweight) ----
  let currentTool: Tool = 'place';
  let selectedBlock = 'stone';
  let replaceFrom = 'stone';
  let pos1: Vec3 | null = null;
  let pos2: Vec3 | null = null;
  let ghostRadius = 0;
  let lastHoverPlacePos: any = null;
  let scenarioName = 'untitled';
  let scenarioMetadata: ScenarioMetadata = { description: '', tags: [], category: '', difficulty: '' };

  // ---- HTTP + socket.io ----
  const httpServer = http.createServer(handleRequest);
  const io = new SocketIOServer(httpServer);
  const editorNs = io.of('/editor');    // replaces SSE
  // viewer namespace wired after observerBot spawns

  // ---- Domain objects ----
  const scenarios = new ScenarioManager(cwd);

  // initialized after observerBot spawns:
  let editor: WorldEditor;
  let markerMgr: MarkerManager;
  let testRunner_: TestRunner;
  let pathVisualizer: PrismarinePathVisualizer;

  function broadcastEditState() {
    editorNs.emit('edit-state', getEditState());
  }

  function broadcastLog(line: string) {
    editorNs.emit('log', line);
  }

  // ---- HTTP request handler (REST API only — no SSE, no proxy) ----
  async function handleRequest(req, res) {
    // same routes as today EXCEPT:
    // - /events endpoint REMOVED (replaced by socket.io /editor namespace)
    // - /viewer* proxy REMOVED (viewer runs on same httpServer)
    // everything else unchanged
  }

  // ---- MC server ----
  const server = createMCServer({ ... port ... });

  server.once('listening', async () => {
    httpServer.listen(adminPort, () => console.log(`[panel] http://localhost:${adminPort}`));

    const observerBot = mineflayer.createBot({ host: 'localhost', port, username: 'Observer', version, auth: 'offline' });

    observerBot.once('spawn', () => {
      // Attach viewer to SAME httpServer, viewer namespace = '/viewer'
      attachViewer(observerBot, { httpServer, namespace: '/viewer', firstPerson: false, viewDistance });

      editor = new WorldEditor(server.overworld, server, mcData, PBlock, broadcastEditState);
      markerMgr = new MarkerManager(() => observerBot.viewer, broadcastEditState);
      testRunner_ = new TestRunner(testRunner, port, broadcastLog, (code) => {
        broadcastLog(`\n[done] exit code ${code}\n`);
        editorNs.emit('done');
      });
      pathVisualizer = new PrismarinePathVisualizer(observerBot);

      // Wire viewer events (same as today lines 1078–1201, but calling editor.*, markerMgr.*)
      observerBot.viewer.on('blockClicked', async (block, _face, button, placePos, meta) => { ... });
      // etc.
    });

    if (cwd) watchLayoutFiles(cwd, server, mcData, PBlock);
  });

  // ---- socket.io /editor namespace (replaces SSE) ----
  editorNs.on('connection', (socket) => {
    socket.emit('edit-state', getEditState());
  });
}
```

### Routes that change in the HTTP handler

| Route | Change |
|---|---|
| `GET /events` (SSE) | **Remove** — replaced by socket.io `/editor` namespace |
| `GET /viewer*` (proxy) | **Remove** — viewer runs on same server |
| `WS upgrade /viewer*` | **Remove** — socket.io handles its own upgrade |
| All other routes | Unchanged paths, replace `broadcastEditState()` + `broadcast()` calls with `editorNs.emit(...)` |

---

## Change 8 — Browser client (UI) SSE → socket.io

The Svelte admin UI in `packages/mineflayer-test-suite/ui/` currently uses:
- `new EventSource('/events')` for server push
- `fetch('/api/config')` on startup

**Replace `EventSource` with a socket.io-client connection to `/editor`:**

```js
// Find EventSource usage in ui/src/ (grep: EventSource, /events)
// Replace with:
import { io } from 'socket.io-client';
const socket = io('/editor');
socket.on('edit-state', (state) => { /* same handler as SSE edit-state event */ });
socket.on('log', (line) => { /* same handler as SSE data event */ });
socket.on('done', () => { /* same handler as SSE done event */ });
```

Find the SSE setup by searching `ui/src/` for `EventSource` or `"/events"`.

---

## Change 9 — `setup.ts` visualizer coupling

**Current** (`setup.ts:59–105`): Reads `process.env.TEST_SERVER_PORT` to decide which visualizer to create.

**New**: Accept optional `visualizer` in `SetupOptions`. Fall back to `NullPathVisualizer`.

```ts
// packages/mineflayer-test-suite/src/types.ts
// Add to SetupOptions:
visualizer?: PathVisualizer;

// packages/mineflayer-test-suite/src/setup.ts
// Replace lines 101-104:
const visualizer: PathVisualizer = options.visualizer ?? new NullPathVisualizer();

// Remove: externalPort logic for visualizer, RemotePathVisualizer import if unused
```

`RemotePathVisualizer` stays available for callers who want it — `dev-server.ts`'s `TestRunner` passes the visualizer explicitly when spawning the test process instead of relying on `TEST_SERVER_PORT`.

---

## Execution Order

Apply changes in this order to avoid broken intermediate states:

1. **Create `ChunkRelay.ts`** — no dependents yet, pure extraction
2. **Rewrite `prismarine-viewer/lib/mineflayer.js`** — change signature, test with a standalone script
3. **Create `WorldEditor.ts`** (imports `ChunkRelay.ts`)
4. **Create `ScenarioManager.ts`** (no imports from above)
5. **Create `MarkerManager.ts`** (no imports from above)
6. **Create `TestRunner.ts`** (no imports from above)
7. **Rewrite `dev-server.ts`** — imports all new classes, removes proxy/SSE
8. **Update `setup.ts`** — remove env var coupling
9. **Update UI** — EventSource → socket.io-client

Commit each submodule separately (`packages/mineflayer-test-suite` and `packages/prismarine-viewer`), then update the parent workspace pointer.

---

## Change 10 — REST API Restructure

Current routes are a flat verb-noun soup (`/set-replace-from`, `/toggle-favorite`, `/export-layout`, etc.). Replace with resource-grouped paths.

### Full route mapping (old → new)

| Method | Old path | New path | Notes |
|---|---|---|---|
| GET | `/api/config` | `/api/config` | unchanged |
| GET | `/api/edit-state` | `/api/editor/state` | |
| POST | `/select-block` | `/api/editor/block` | body: `{ block }` |
| POST | `/select-tool` | `/api/editor/tool` | body: `{ tool }` |
| POST | `/set-replace-from` | `/api/editor/replace-from` | body: `{ block }` |
| POST | `/undo` | `/api/editor/undo` | |
| POST | `/redo` | `/api/editor/redo` | |
| POST | `/clear-selection` | `/api/selection/clear` | |
| POST | `/fill` | `/api/selection/fill` | |
| POST | `/replace` | `/api/selection/replace` | |
| POST | `/copy` | `/api/selection/copy` | |
| POST | `/paste` | `/api/selection/paste` | body: `{ x, y, z }` |
| POST | `/reset` | `/api/world/reset` | body: `WorldLayout` |
| POST | `/visualize` | `/api/world/visualize` | body: `{ nodes? clear? }` |
| GET | `/api/scenarios` | `/api/scenarios` | unchanged |
| GET | `/api/scenario/:name` | `/api/scenarios/:name` | |
| POST | `/set-scenario-name` | `/api/scenarios/active-name` | body: `{ name }` |
| POST | `/set-metadata` | `/api/scenarios/metadata` | body: `ScenarioMetadata` |
| POST | `/save-scenario` | `/api/scenarios/save` | |
| POST | `/load-scenario` | `/api/scenarios/load` | body: `{ name }` |
| POST | `/export-layout` | `/api/scenarios/export` | body: `WorldLayout` |
| POST | `/toggle-favorite` | `/api/scenarios/:name/favorite` | |
| GET | `/api/tests` | `/api/tests` | unchanged |
| POST | `/run/:suite` | `/api/tests/:suite/run` | query: `?test=` |
| GET | `/api/markers` | `/api/markers` | new — returns marker list |
| POST | `/add-marker` | `/api/markers` | body: `{ name, position }` |
| POST | `/update-marker` | `/api/markers/:id` | body: `{ position }` |
| POST | `/remove-marker` | `/api/markers/:id/delete` | (use POST not DELETE — simpler fetch) |

### Implementation notes

- All route handling in `dev-server.ts` is in one big `if/else` chain (lines 726–1041). After extracting domain classes (Changes 3–6), replace this with a simple router map:

```ts
const routes: Record<string, (req, res, url) => Promise<void>> = {
  'GET /api/config':             handleConfig,
  'GET /api/editor/state':       handleEditorState,
  'POST /api/editor/block':      handleSetBlock,
  'POST /api/editor/tool':       handleSetTool,
  // ...
};

async function handleRequest(req, res) {
  const key = `${req.method} ${req.url?.split('?')[0]}`;
  const handler = routes[key] ?? routeWithParams(req.url);
  if (handler) return handler(req, res, req.url);
  if (req.method === 'GET') return serveStatic(res, req.url ?? '/');
  res.writeHead(404).end();
}
```

- UI fetch calls must be updated to match new paths. Search `ui/src/` for all `fetch('` occurrences and remap.
- `setup.ts:postLayout` (line 23) posts to `/reset` → update to `/api/world/reset`.
- `RemotePathVisualizer.ts` posts to `/visualize` → update to `/api/world/visualize`.

---

## What Does NOT Change

- `viewer/lib/worldView.js`, `viewer/lib/viewer.js`, `viewer/lib/primitives.js`, `viewer/lib/controls.js` — untouched
- `setup.ts` `setup()` function signature (callers unaffected unless passing new `visualizer` field)
- `types.ts` — only `SetupOptions` gets the optional `visualizer` field added
- `cli.ts` — unchanged
- Flying-squid server config — unchanged

---

## Key Invariants to Verify After Refactor

1. Browser loads `http://localhost:adminPort` — admin panel renders
2. Browser loads `http://localhost:adminPort/viewer` (or connects to `/viewer` socket.io ns) — 3D world visible
3. Observer bot movement reflected in viewer
4. Block click → place/break/select works
5. Gizmo transform → marker move works
6. Test run (via `/run/integration`) → output streams to browser log panel
7. `setup()` called from a test process connects to dev server and layout applies
8. `setup()` called standalone (no `TEST_SERVER_PORT`) still works independently
