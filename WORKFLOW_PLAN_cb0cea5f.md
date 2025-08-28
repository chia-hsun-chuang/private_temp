# Workflow Runtime & UI Plan (iOS) — Developer Hand‑Off

> **Concept:** An iOS app that runs AI workflows defined as **directed acyclic graphs (DAGs)** of **tools** (“nodes”). Tools connect using **typed input/output ports** (e.g., `image`, `mask`, `depth`). Heavy jobs run **remotely** on GPU through API calls; quick transforms can run **locally** on the device. Some tools are **interactive** and pause for user input; these are handled **out‑of‑band** via an **Action Center**, notifications, and deep links — **never auto‑present UI**.

---

## 1) Critical Requirements (Non‑Negotiables)

1. **Deterministic dataflow**
   - The workflow template (JSON) defines all connections between typed ports.
   - A tool becomes **ready** only when all **required** input ports are satisfied.

2. **Interactive nodes are non‑intrusive & queueable**
   - Tools may declare `await` work (mask drawing, asset selection, param confirmation).
   - The runtime **does not auto‑open** UI. It creates an **AwaitRequest** and surfaces it via:
     - An **Action Center** screen/tab (queue of pending interactions).
     - **Local/Push notifications** with deep links.
     - **Badges** on the app tab and per workflow row/card.
   - Multiple awaits can accumulate; users resolve them when convenient.

3. **Parallel branches + mixed scheduling**
   - Independent branches run in parallel.
   - Two queues with limits: **Remote** (GPU/API jobs) and **Local** (device‑light tasks).

4. **Cancellable‑only long jobs**
   - Long jobs can be **cancelled**; there is **no pause**. “Pause/Resume” toggles **status‑update polling** only.

5. **Resumable state & migrations**
   - Persist on every state change. The app must recover after relaunch and across app updates using schema versioning/migrations.

6. **Transparent status & errors**
   - Every card shows **why it is blocked**, progress, and actionable recovery: **Retry (same inputs)** and **Re‑run with latest**.

7. **Performance, accessibility, and safe storage**
   - Smooth lists, Dynamic Type & VoiceOver support.
   - Thumbnail/preview cache with eviction; never log raw media.
   - Default polling cadence for remote jobs: **10 seconds**.

---

## 2) System Architecture

### 2.1 UI Layer (SwiftUI)

- **WorkflowsView** (list):
  - Searchable; segmented filter (All/Active/Done/Failed); sort menu (Recent/Name).
  - Each row: title, status badge, step progress (e.g., `2/5`), “updated 2m ago”, optional thumbnail.
  - Swipe actions: Duplicate, Archive, Delete. Toolbar: **+ New from Template**.
  - Tap a row → **WorkflowDetailView**.

- **WorkflowDetailView** (ToolCards rendered in graph order):
  - **Collapsed:** title + icon, status badge, mini result summary (type/time), compact **param digest** (selected “summary” params).
  - **Expanded:** Parameters; Inputs; Run/Status (Run/Cancel/Retry, progress, “updated Xs ago”, short log tail with sheet); Outputs tiles (viewer with Share/Save/Open in…); History strip (last N, View All).

- **Action Center** (tab/screen for interactive nodes):
  - Sections: **Pending**, **Snoozed**, **Done**.
  - Cells show: title, node/tool icon, small preview, **Open**, **Snooze**, and optional **Skip** (if policy allows).
  - Global badge on the tab; per‑workflow badge on list rows; per‑card badge “Needs your input”.

### 2.2 View Models

```swift
@MainActor final class WorkflowsVM: ObservableObject {
    // list, search/filter/sort, CRUD on workflows
}

@MainActor final class ToolInstanceVM: ObservableObject, Identifiable {
    // derives state/progress; run(), cancel(), retry(), input selection
}

@MainActor final class AwaitVM: ObservableObject {
    @Published var pending: [AwaitRequest] = []
    func claim(_ id: String) async throws
    func complete(_ id: String, with result: AwaitResult) async throws
    func snooze(_ id: String, until date: Date) async throws
}
```

### 2.3 Runtime / Engine

- **Graph Loader:** parses JSON → builds in‑memory DAG with typed ports.
- **Ready‑Set Resolver:** a node is **ready** when all required input ports are bound to valid assets.
- **JobManager:** submission, state machine, **10s polling**, backoff, persistence on every state change.
- **Schedulers:** two queues — **Remote** (GPU/API) with higher concurrency; **Local** (device) with lower concurrency.
- **Fairness:** prioritize the active workflow view, then round‑robin between workflows.

### 2.4 Storage

- SwiftData/Core Data models (see §4). Provider‑specific payloads inside typed, opaque `metaJSON` blobs.
- Asset store + caches: LRU for thumbnails; time‑boxed purge for full previews; per‑asset “Keep offline” pin.

---

## 3) Workflow Template Format (JSON)

```jsonc
{
  "schema": "app.workflow/v2",
  "name": "Stylize with user mask",
  "nodes": [
    {
      "id": "load1",
      "kind": "ImageLoader",
      "params": { "source": "photos" },
      "outputs": { "image": "ImageRef" }
    },
    {
      "id": "mask1",
      "kind": "UserMaskEditor",
      "inputs": { "image": "ImageRef" },
      "outputs": { "mask": "MaskRef" },
      "await": {
        "type": "draw_mask",
        "blocking": true,
        "urgency": "normal",
        "expiresIn": "PT24H",
        "notification": { "title": "Draw a mask", "body": "Mask the subject for the next step." },
        "resumePolicy": { "onTimeout": "cancel" },
        "ui": "mask-sheet"
      }
    },
    {
      "id": "stylize1",
      "kind": "StylizeRemote",
      "params": { "steps": 30, "cfg": 4.0 },
      "inputs": { "image": "ImageRef", "mask": "MaskRef" },
      "outputs": { "image": "ImageRef" },
      "resources": { "class": "remote" }
    },
    {
      "id": "compress1",
      "kind": "CompressLocal",
      "params": { "targetMB": 2 },
      "inputs": { "image": "ImageRef" },
      "outputs": { "file": "FileRef" },
      "resources": { "class": "local" }
    }
  ],
  "edges": [
    { "from": ["load1","image"],    "to": ["mask1","image"] },
    { "from": ["load1","image"],    "to": ["stylize1","image"] },
    { "from": ["mask1","mask"],     "to": ["stylize1","mask"] },
    { "from": ["stylize1","image"], "to": ["compress1","image"] }
  ],
  "options": {
    "concurrency": { "remote": 8, "local": 2 },
    "errorPolicy": "fail-fast" // or "isolate-node"
  }
}
```

**Rules**
- Typed ports; edges must match types; the graph must be acyclic.
- Unknown `kind` → node is **blocked** with “Tool not installed.”
- `await` marks interactive nodes (see §5).

---

## 4) Data Model (SwiftData/Core Data Friendly)

```swift
@Model final class WorkflowInstance {
    var id: UUID
    var title: String
    var tools: [ToolInstance]
    var createdAt: Date
    var updatedAt: Date
    var schemaVersion: Int
}

@Model final class ToolInstance {
    var id: UUID
    var kind: String
    var state: ToolState
    var runs: [ToolRun]
    var paramsJSON: Data
    var inputBindings: [PortBinding] // bound AssetRefs by port name
    var outputPorts: [PortDecl]
}

@Model final class ToolRun {
    var id: UUID
    var jobID: String?
    var progress: Double
    var startedAt: Date?
    var finishedAt: Date?
    var producedAssets: [AssetRef]
    var logsTail: String?
}

enum ToolState: String, Codable {
    case blocked, ready, awaitingUser, runningLocal, queuedRemote, runningRemote, downloading, succeeded, failed, cancelled
}

struct PortDecl: Codable { var name: String; var type: String; var optional: Bool }
struct PortBinding: Codable { var port: String; var assetId: String }
struct Provenance: Codable { var nodeId: String; var runId: String; var port: String }
struct AssetRef: Codable { var id: String; var type: String; var url: URL; var metaJSON: Data?; var createdAt: Date; var provenance: Provenance }
```

**Gating rule:** a tool is **enabled** when all required input ports are bound to valid assets.  
**Provenance:** stamp every produced asset so downstream invalidation is automatic when upstream changes.

---

## 5) Interactive Nodes (iOS‑Safe Design)

### 5.1 AwaitRequest model (runtime)

```swift
struct AwaitRequest: Codable, Identifiable {
    var id: String
    var workflowId: UUID
    var nodeId: String
    var type: AwaitType          // .drawMask, .selectAsset, .confirmParams, etc.
    var title: String
    var detail: String?
    var urgency: Urgency         // .blocking, .normal, .low
    var blocking: Bool
    var createdAt: Date
    var expiresAt: Date?
    var deepLink: URL            // app://workflow/{wf}/node/{node}?await={id}
    var thumbnailURLs: [URL]?
    var schemaRef: String?
    var resumePolicy: ResumePolicy // onTimeout: .cancel | .autoDefault | .skip
}
```

**State machine:** for interactive nodes use `ready → awaitingUser` (**do not** enqueue a job).

### 5.2 Surfacing & UX

- **Action Center** lists awaits with **Open**, **Snooze**, optional **Skip**.
- **Notifications**: `UNUserNotificationCenter` with categories and actions (Approve/Reject/Snooze/Open). All notifications deep‑link to the exact editor.
- **Foreground**: show a non‑blocking banner/toast and update badges. Do not auto‑present a sheet.

### 5.3 Concurrency & Claiming

```swift
struct AwaitClaim { let awaitId: String; let claimedAt: Date; let lease: TimeInterval = 300 }
```
- When the user opens an await, **claim** it to avoid double handling. Lease expires if the app/background is killed.

### 5.4 Completion & Resume

```swift
enum AwaitResult { case asset(AssetRef), choice(String), params(Data), cancel }
```
- On completion: validate types, write provenance, transition `awaitingUser → ready` (enqueueable), or `cancelled/failed` per result. Invalidate downstream if prior inputs were replaced.

### 5.5 Expiration & Fallback

- If `expiresAt` passes:
  - `.cancel`: mark failed with explanation.
  - `.autoDefault`: submit with default params or the single best candidate.
  - `.skip`: treat outputs as absent; allow graph to continue if consuming ports are optional.

---

## 6) Execution & Scheduling

1. **Load & Validate** the JSON; type check ports; ensure acyclic graph.
2. **Resolve Ready Set** using the gating rule; interactive nodes move to `awaitingUser` (create AwaitRequest).
3. **Enqueue** ready nodes by `resources.class`:
   - `remote` → Remote Queue (high concurrency)
   - `local` → Local Queue (low concurrency)
4. **Parallelism & Fairness**
   - Fill queues up to concurrency caps; prioritize active workflow; round‑robin across workflows.
5. **Lifecycle & Polling**
   ```
   blocked → ready → awaitingUser? → (runningLocal | queuedRemote → runningRemote) → downloading → succeeded | failed | cancelled
   ```
   - Show percent if available; otherwise indeterminate; “Reconnecting…” on network flap; exponential backoff.
6. **Retry & Re‑run**
   - **Retry (same inputs)**: re‑run with frozen input asset IDs + last params.
   - **Re‑run with latest**: resolve current edges and use newest upstream assets.
7. **Cancellation**
   - Only **Cancel** stops the job. “Pause/Resume status updates” toggles polling only (does not pause compute).

---

## 7) UI Specifications (Key Components)

### 7.1 Workflows List

- Searchable `List` with segmented filter (All/Active/Done/Failed) and sort menu (Recent/Name).
- Row: title, status badge, `ProgressView` for overall steps, “updated N min ago,” optional thumbnail.
- Swipe: Duplicate / Archive / Delete. Toolbar: **+ New from Template**.

### 7.2 ToolCard

- **Collapsed:** title+icon, status badge, last result mini‑summary, **param digest** (e.g., `res=1024×1024 · steps=30 · cfg=4.0`).  
- **Expanded Panels:**
  1) **Parameters** (form bound to `paramsJSON`, reset to defaults, optional presets).
  2) **Inputs** (which upstream outputs fill each port; if multiple, **Select Input** opens an asset picker).
  3) **Run & Status** (Run/Cancel/Retry, `ProgressView`, updated‑ago text, short log tail with “Open logs” sheet).
  4) **Outputs** (tiles per output port; tap → viewer/sheet with Share/Save/Open in…).
  5) **History** (last N runs; View All → table with status/time/duration/params diff).

### 7.3 Await in UI

- **Card badge:** “Needs your input” with actions **Open editor**, **Snooze**, optional **Skip**.
- **Workflow row badge:** count of awaits in that workflow.
- **Action Center:** centralized queue; groups by workflow; sorts by urgency/age.

### 7.4 Toolbar (Detail)

- **Run all ready steps**, **Pause/Resume status updates**, **Duplicate workflow**.

---

## 8) Error Handling & Recovery

- **Actionable errors:** short summary + raw code + hints (e.g., “Try smaller resolution”).  
- **Blocked by upstream failure:** show which input/step is missing; offer **Open upstream**, **Retry upstream**, **Replace input…**.  
- **Provenance & invalidation:** when upstream changes, downstream consumers flip to `blocked` with “Input changed”; add **Re‑run with latest**.  
- **Network flap:** display “Reconnecting…” and keep polling with exponential backoff.

---

## 9) Persistence, Resilience, and Migrations

- Persist on every state change; replay on relaunch.  
- Add `schemaVersion` to stored roots; migrate on load (default fields, enum renames).  
- Keep provider‑specific blobs in typed `metaJSON` (opaque to core decoder).

---

## 10) Performance, Caching & Offline

- `List`/`ForEach` with stable IDs; lazy thumbnail loading; avoid logging raw media.  
- **Cache policy:**
  - Thumbnails: LRU up to N MB (e.g., 200 MB).
  - Full previews: auto‑purge after 7 days.
  - Per‑asset **Keep offline** pin.
- Offline: disable remote tools with a clear reason; allow local tools; auto‑resume when online.

---

## 11) Testing Checklist

- **Gating:** steps remain disabled until required ports are bound; missing ports are listed on the Run button.  
- **Resume:** kill mid‑run; relaunch restores and continues polling.  
- **Network:** simulate offline/online; check graceful recovery and backoff.  
- **Parallel workflows:** two or more workflows running do not interfere; progress isolated.  
- **Interactive nodes:** multiple awaits queued; Action Center flows; deep links open the exact editor; claim/lease prevents double handling.  
- **Accessibility:** VoiceOver labels for all controls; Dynamic Type friendly.

---

## 12) Developer Handoff Order (Milestones)

1. **Graph Loader + Validation** (typed ports, DAG check, unknown‑kind blocking).  
2. **Data Model + Persistence** (SwiftData models + migrations skeleton).  
3. **Ready‑Set Gating & JobManager** (state machine + **10s polling** + backoff).  
4. **Schedulers** (Remote and Local queues; concurrency caps; fairness).  
5. **ToolCard UI** (collapsed digest, expanded panels, history; Retry modes).  
6. **Interactive System** (AwaitRequest model, Action Center UI, notifications + deep links, claim/complete/snooze).  
7. **Provenance & Invalidation** (auto‑block downstream on upstream changes).  
8. **Caching & Storage UI** (usage display, “Free up space,” per‑asset pin).

---

###  Appendix A — Deep Link Format & Notification Categories

- **Deep link:** `app://workflow/{workflowId}/node/{nodeId}?await={awaitId}`  
- **Notification categories (examples):**
  - `AWAIT_DRAW_MASK` — actions: Open, Snooze
  - `AWAIT_SELECT_ASSET` — actions: Choose A, Choose B, Open
  - `AWAIT_CONFIRM_PARAMS` — actions: Approve, Edit, Snooze

