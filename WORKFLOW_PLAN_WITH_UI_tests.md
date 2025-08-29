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



---

# Appendix B — UI Specification

# Workflow App — UI Specification (Deterministic)

This document defines the **end‑to‑end UI** for the deterministic workflow runtime. It is implementation‑ready and aligned with the reviewed plan.

---

## 0. Design Principles
- **Deterministic**: Tool input bindings are **read‑only** and defined by the template. Interaction happens only in **explicit interactive nodes**.
- **Non‑intrusive**: The app never auto‑opens editors. All human‑in‑the‑loop actions are surfaced through **ToolCards** and the **Action Center**.
- **Legible state**: Every card shows **exact state** with color + text and why it’s blocked.
- **Accessible**: Dynamic Type, VoiceOver labels, sufficient contrast, and non‑color cues (chips/badges).

---

## 1. Tab Bar
Bottom **TabView** with three tabs (icons are suggestions):
- **Workflows** (`list.bullet.rectangle`)
- **Action Center** (`bell.badge`)
- **Settings** (`gearshape`)

Badges may appear on tabs (e.g., Action Center shows pending await count).

---

## 2. Workflows Tab

### 2.1 WorkflowsView (List of Instances)
- **Toolbar**: “+ New from Template”
- **Searchable** + **Segmented control**: `All / Active / Done / Failed`
- **Sort menu**: `Recent / Name`

**Row content:**
- Title (primary), Status badge (right), “Updated {N} ago”
- Subline: progress `2 of 5 steps` with subtle bar
- Optional thumbnail (latest output)
- **Row badges**: red dot if workflow has **blocking awaits**

**Swipe actions**: `Duplicate · Archive · Delete`

**Empty state**: “No workflows yet — create one from a template.”

### 2.2 Navigation
Tap row → **WorkflowDetailView**

---

## 3. Workflow Detail

### 3.1 Layout
A vertical list of **ToolCards** in the execution order.

### 3.2 ToolCard — States & Visuals
**Collapsed** (default):
- Icon + Title (left), Status badge (right)
- Param digest (e.g., `res=1024×1024 · steps=30 · cfg=4.0`)
- Mini result summary (type/time) when available

**Expanded** (tap to expand):
1) **Parameters** (editable form; reset; presets)
2) **Inputs** (*read‑only* bindings per deterministic rule)
3) **Run & Status** (Run / Cancel / Retry / Re‑run with latest, progress, log tail)
4) **Outputs** (tiles; tap → media viewer sheet with Save/Share/Open in…)
5) **History** (last N runs + View All)

**State Highlighting (color + cue):**
- **Running (Local/Remote)**: **Blue** border + **blue fill 15%**; progress bar
- **Awaiting user**: **Orange** border + **orange fill 15%**; **chip** “Needs your input”; *optional gentle pulse*
- **Succeeded**: **Green** chip “Done”
- **Failed**: **Red** border + chip with error code
- **Blocked**: Grey chip with reason (“Upstream failed” or “Input changed”)

> **Tap behavior:** If state is **awaiting user**, tap opens the **editor** for that node (mask drawer, asset picker, param confirm). Otherwise, tap toggles expand/collapse.

### 3.3 Toolbar (Detail)
- **Run all ready**
- **Pause polling / Resume polling** (tooltip: *“Pauses status checks only; remote job continues running.”*)
- **Duplicate workflow**

### 3.4 Focus on a Node
When navigated with a `nodeId` focus (e.g., from Action Center), the view **scrolls smoothly** to that ToolCard and **pulses** it for ~6s. The editor still opens **only after user tap**.

---

## 4. Action Center

### 4.1 Content
A **queue of Await Requests** grouped into sections:
- **Pending**
- **Snoozed**
- **Done** (auto‑cleared after retention; e.g., 7 days)

**Await Cell** (compact card):
- Workflow title + Tool/node title
- Tool icon; small preview (if available)
- Subline: “added 5m ago · expires in 1h”
- Actions: **Open** (primary), **Snooze** (1h default), **Skip** (if allowed)

> **This is not a ToolCard list.** It is an inbox of interactive tasks.

### 4.2 Navigation on Tap
- Tapping **Open** navigates to **WorkflowDetailView** with `focusNodeId`
- The detail view **scrolls** and **pulses** the awaiting ToolCard
- The user then **taps the ToolCard** to open its editor (claim occurs on tap)

---

## 5. Settings
- Provider/model configs
- Cache usage view + “Free up space”
- Pinned (“Keep offline”) assets
- Developer/debug tools (show JSON, enable logs, test harness)
- Accessibility preferences

---

## 6. Color & Status Tokens
| State | Border | Fill | Badge/Chip |
|---|---|---|---|
| Awaiting user | Orange | Orange 15% | Chip: “Needs your input” |
| Running (local/remote) | Blue | Blue 15% | Progress + label “Running” |
| Succeeded | Clear | Neutral | Green “Done” |
| Failed | Red | Neutral | Red chip w/ code |
| Blocked | Gray | Neutral | Gray “Blocked: reason” |

> Respect iOS **high‑contrast** modes and **Dynamic Type**. Pulse uses **scale/opacity** only, no color flashing.

---

## 7. Editors (High Level)
- **UserAssetPicker** (interactive): Photos/Files/Camera → emits one `AssetRef`
- **Mask Editor** (interactive): draw/erase/feather; emits `MaskRef`
- **Param Confirm** (interactive): review set; emits `params`
- **Other local tools** (viewers, crop, compress) as needed

Editors are **sheets** or **pages** invoked from ToolCards; they **claim** the await on open, and **complete** with `AwaitResult`.

---

## 8. Empty/Loading/Error States
- **Skeletons** on first load
- **Retry** CTA on transient errors
- **No awaits** → “You’re all set!” illustration
- **No workflows** → “Create one from a template” CTA

---

## 9. Acceptance Criteria (UI)
- ToolCard states render exactly as table above
- From Action Center → focus/pulse behavior verified
- Pause polling changes only status checks (does not stop jobs)
- Accessibility: all tappables labeled; VoiceOver reads state and actions
- Metrics hooks: card taps, editor claims/completions, retries, cancels


---

# Appendix C — Navigation & Router Guide

# Navigation & Router Guide (SwiftUI)

This document specifies a **unified navigation approach** so both the **Workflows tab** and the **Action Center** navigate consistently into `WorkflowDetailView` with optional node focus.

---

## 1. Router

```swift
struct WorkflowFocus: Hashable {
    let workflowId: UUID
    let nodeId: String? // nil = just open workflow; non-nil = focus a ToolCard
}

@MainActor
final class Router: ObservableObject {
    @Published var path: [WorkflowFocus] = []
    func openWorkflow(_ workflowId: UUID, focusNodeId: String? = nil) {
        path = [WorkflowFocus(workflowId: workflowId, nodeId: focusNodeId)]
    }
}
```
Inject `Router` as an `@EnvironmentObject` and bind its `path` to every `NavigationStack` that needs to reach `WorkflowDetailView`.

---

## 2. App Shell

```swift
@main
struct MyApp: App {
    @StateObject private var router = Router()
    var body: some Scene {
        WindowGroup {
            TabView {
                NavigationStack(path: $router.path) {
                    WorkflowsView()
                        .navigationDestination(for: WorkflowFocus.self) { f in
                            WorkflowDetailView(workflowId: f.workflowId, focusNodeId: f.nodeId)
                        }
                }.tabItem { Label("Workflows", systemImage: "list.bullet.rectangle") }

                NavigationStack(path: $router.path) {
                    ActionCenterView()
                        .navigationDestination(for: WorkflowFocus.self) { f in
                            WorkflowDetailView(workflowId: f.workflowId, focusNodeId: f.nodeId)
                        }
                }.tabItem { Label("Action Center", systemImage: "bell.badge") }

                SettingsView().tabItem { Label("Settings", systemImage: "gearshape") }
            }
            .environmentObject(router)
        }
    }
}
```

---

## 3. Workflows → Detail

```swift
struct WorkflowsView: View {
    @EnvironmentObject private var router: Router
    var body: some View {
        List(workflows) { wf in
            Button { router.openWorkflow(wf.id) } label { WorkflowRow(workflow: wf) }
        }
        .navigationTitle("Workflows")
    }
}
```

---

## 4. Action Center → Detail (with focus)

```swift
struct ActionCenterView: View {
    @EnvironmentObject private var router: Router
    var body: some View {
        List(pendingAwaits) { a in
            Button { router.openWorkflow(a.workflowId, focusNodeId: a.nodeId) } label { AwaitCell(await: a) }
        }
        .navigationTitle("Action Center")
    }
}
```

---

## 5. Smooth focus in `WorkflowDetailView`

**One‑liner extension:**

```swift
extension ScrollViewProxy {
    func smoothScrollTo<ID: Hashable>(_ id: ID, anchor: UnitPoint = .center) {
        withAnimation(.easeInOut) { scrollTo(id, anchor: anchor) }
    }
}
```

**Usage:**

```swift
struct WorkflowDetailView: View {
    let workflowId: UUID
    let focusNodeId: String?
    @StateObject private var vm = ToolGraphVM()
    @State private var pulseNodeId: String?

    var body: some View {
        ScrollViewReader { proxy in
            List(vm.nodes) { node in
                ToolCard(node: node) { tapped in
                    if tapped.state == .awaitingUser { Task { try? await vm.claimAndOpenEditor(for: tapped) } }
                }
                .id(node.id)
                .modifier(PulseIfFocused(pulse: pulseNodeId == node.id && node.state == .awaitingUser))
            }
            .onAppear {
                vm.load(workflowId: workflowId)
                if let id = focusNodeId {
                    DispatchQueue.main.asyncAfter(deadline: .now() + 0.15) {
                        proxy.smoothScrollTo(id)
                        pulseNodeId = id
                        DispatchQueue.main.asyncAfter(deadline: .now() + 6) { pulseNodeId = nil }
                    }
                }
            }
        }
    }
}
```

This guarantees **consistent routing** and **smooth focus** from both entry points.


---

# Appendix D — Await Editors Contract

# Await Editors — Contract & UX

This document defines how **interactive nodes** (awaits) integrate with the UI and runtime.

---

## 1. AwaitRequest → Editor contract

**When a ToolCard is in `awaitingUser`:**
- The card shows chip **“Needs your input.”**
- On **tap**, the app **claims** the await and presents the editor (sheet or page).

**Editor responsibilities:**
- Receive context (workflowId, nodeId, awaitId, expected output type).
- Present task‑specific UI (picker, mask editor, confirm form).
- Validate results **before** completion (type & size constraints).
- On **Complete** → return `AwaitResult` (e.g., `.asset(AssetRef)`, `.params(Data)`, `.choice(String)`).  
- On **Cancel** → either restore await to **Pending** or mark **cancelled** per `resumePolicy`.

---

## 2. UX Rules

- **No auto‑present**: Editors are opened only after the user taps the **awaiting** ToolCard.
- **Non‑blocking**: If the user dismisses the editor, the await returns to **Pending** unless the editor committed a result.
- **Snooze**: Available from Action Center (default = 1h); does not claim the await.

---

## 3. Mask Editor (example spec)

- Tools: Brush, Eraser, Feather, Undo/Redo, Reset, Zoom/Pan
- Overlay: image with mask alpha preview
- Output: `MaskRef` (PNG or RLE + metadata); record provenance (nodeId, runId, port)
- Validation: max resolution, empty mask detection (warn → allow continue)

---

## 4. Asset Picker (UserAssetPicker)

- Sources: Photos, Files, Camera
- Output: single `AssetRef`
- Constraints: media type enforced by the tool’s input port (e.g., `image`)
- Privacy: no raw media in logs; thumb cache allowed

---

## 5. Param Confirm

- Summary of key parameters; toggles/sliders for safe edits
- Output: serialized params (`Data`) matching tool schema
- Defaults surfaced; “Reset to defaults” available

---

## 6. Completion semantics

- On success: write asset(s) with **Provenance**, update ToolCard to **ready/queued**, and schedule downstream
- On cancel: follow `resumePolicy` (cancel | autoDefault | skip)
- On timeout/lease expiry: await returns to **Pending** (unclaimed)


---

# Appendix E — TEST PLAN

# TEST PLAN — Workflow App (Deterministic)

This plan defines **acceptance tests** for each build task. Developers will implement unit/UI tests (XCTest/XCUITest) and contract tests using the provider harness.

**Notation:** G/W/T = Given / When / Then.  
**Environments:** iOS 17+, SwiftUI.  
**Data:** Use fixture JSON templates and stubbed provider responses (Harness Milestone 2.5).

---

## 0) Project Setup & Navigation

**Goal:** Tabs render; Router navigates consistently; smooth scroll extension works.

**Scenarios**
- **G**: App shell with Router injected. **W**: Open app. **T**: Tabs visible (Workflows, Action Center, Settings).
- **G**: Existing workflow `wfA`. **W**: Tap workflow row. **T**: `WorkflowDetailView` appears.
- **G**: Await for node `N` in `wfA`. **W**: Tap await in Action Center. **T**: Navigates to `wfA`, smooth-scrolls to `N` (center anchor).

**Pass/Fail:** Navigation path equals `[WorkflowFocus(wfA,N)]`; scroll animation occurs; no crash on relaunch.

---

## 1) Data & Models

**Goal:** Deterministic graph persistence; provider meta JSON versioned.

**Scenarios**
- **G**: Model with required port having 2 producers. **W**: Persist graph. **T**: Save rejected with `multiProducerOnRequiredPort`.
- **G**: Optional port with 2 producers. **W**: Persist graph. **T**: Save rejected with `multiProducerOnOptionalPort`.
- **G**: Asset with `metaJSON` lacking `providerSchema`. **W**: Save. **T**: Warning logged; record stored with default `"providerSchema":"v1"`.

**Pass/Fail:** Errors are exact; no partial commit on failure.

---

## 2) Graph Loader & Validation

**Goal:** Reject non-DAG/typemismatch; accept valid graphs.

**Scenarios**
- **G**: Cycle A→B→A. **W**: Load template. **T**: `cycleDetected` with cycle path in message.
- **G**: Edge type mismatch (`mask` → `image`). **W**: Load. **T**: `portTypeMismatch` with node/port ids.
- **G**: Valid template. **W**: Load. **T**: Nodes marked correctly (`ready`/`blocked`).

**Pass/Fail:** Loader returns structured errors; no crashes.

---

## 3) Runtime Engine

**Goal:** Ready-Set resolver; interactive nodes create awaits; polling 10s; persistence on state change.

**Scenarios**
- **G**: Node becomes ready. **W**: Engine ticks. **T**: Node transitions to `queuedRemote` (or `runningLocal`) per schedule.
- **G**: Interactive node. **W**: Becomes ready. **T**: State `awaitingUser` + `AwaitRequest` created; no editor auto-open.
- **G**: Running nodes + app kill. **W**: Relaunch. **T**: State restored; polling resumes if enabled.

**Pass/Fail:** State transitions recorded; no duplicate awaits after relaunch.

---

## 4) Provenance & Invalidation

**Goal:** New outputs invalidate downstream correctly; recovery actions surfaced.

**Scenarios**
- **G**: Downstream C consumes A’s output. **W**: Re-run A, produce new asset. **T**: C becomes `blocked: Input from 'A' has changed.`
- **G**: Blocked C. **W**: Tap **Re-run with latest**. **T**: C schedules with latest asset id.
- **G**: Queued retry for B pinned to old assets. **W**: Old asset deleted. **T**: Retry aborted pre-start.

**Pass/Fail:** Correct messages; single-click recovery works.

---

## 5) Backoff & Retry Policy

**Goal:** Backoff on transient failures; cap retries.

**Scenarios**
- **G**: Provider returns 503 → 503 → 200. **W**: Submit. **T**: Waits ~30s → ~2m, then succeeds.
- **G**: HTTP 429 with `Retry-After: 45`. **W**: Submit. **T**: First wait ≈45s; then standard backoff if needed.
- **G**: Four 504s. **W**: Submit. **T**: Stops after 3 auto-retries; shows cooldown reason.

**Pass/Fail:** Delays within ±25%; jitter present; logs show reasons.

---

## 6) Action Center

**Goal:** Inbox of awaits; snooze behavior; retention; navigation.

**Scenarios**
- **G**: 2 awaits pending, 1 snoozed. **W**: Open tab. **T**: Lists Pending(2), Snoozed(1), Done(0).
- **G**: Pending await. **W**: Tap **Snooze**. **T**: Moves to Snoozed with `snoozeUntil = now+1h`.
- **G**: Tap **Open**. **W**: Navigate. **T**: Detail scrolls to node; card pulses; editor does **not** auto-open.

**Pass/Fail:** After 1h, snoozed returns to Pending; Done auto-clears after 7 days; snooze attempts >5 are blocked.

---

## 7) Workflow Detail UI (ToolCards)

**Goal:** Visual tokens; accessibility; tap-to-edit for awaits.

**Scenarios**
- **G**: Node states (Awaiting, Running, Succeeded, Failed, Blocked). **W**: Render. **T**: Colors + chips per spec; VoiceOver reads state and actions.
- **G**: Awaiting card. **W**: Tap. **T**: Editor opens; await is claimed.
- **G**: Running. **W**: Tap **Pause polling**. **T**: Polling stops; label toggles to **Resume polling**; job continues remotely.

**Pass/Fail:** Snapshot tests pass; tooltip visible; inputs panel is read-only.

---

## 8) Editors (Await flows)

**Goal:** Editors claim, validate, complete/cancel correctly.

**Scenarios**
- **G**: UserAssetPicker. **W**: Open, choose photo. **T**: Emits single `AssetRef`; downstream node becomes ready.
- **G**: Mask Editor with empty mask. **W**: Complete. **T**: Warn user; allow continue; output saved with provenance.
- **G**: Param Confirm. **W**: Cancel. **T**: Await returns to Pending; no partial outputs.

**Pass/Fail:** Type constraints enforced; no claims left dangling after cancel.

---

## 9) Settings & Storage

**Goal:** Cache sizes; purge; pins persist.

**Scenarios**
- **G**: Cache 200MB, 3 pinned assets. **W**: Tap **Free up space**. **T**: Purges unpinned previews; size drops; pins remain.
- **G**: Toggle provider model config. **W**: Save. **T**: Jobs use updated defaults.

**Pass/Fail:** Sizes reported match filesystem usage; pins survive relaunch.

---

## 10) Notifications & Deep Links

**Goal:** Correct routing; no premature claims; cold-start works.

**Scenarios**
- **G**: Local notification `app://workflow/W/node/N?await=A`. **W**: Tap. **T**: Detail opens, focuses N; await unclaimed until card tapped.
- **G**: App killed. **W**: Tap push. **T**: Store rehydrates state; same behavior as above.

**Pass/Fail:** No duplicate editors; no background claim; bad link handled gracefully.

---

## 11) Testing & Accessibility

**Goal:** CI, harness, a11y.

**Scenarios**
- **G**: Harness endpoints (success, slow, malformed, 429/503/504, -1009). **W**: Run suite. **T**: All branches covered; backoff seen; contract mismatch flagged.
- **G**: VoiceOver. **W**: Navigate ToolCards. **T**: Labels and traits are correct; hit targets ≥44pt.

**Pass/Fail:** CI green; Lighthouse-like a11y audit passes internal thresholds.

---

## 12) Release Readiness

**Goal:** Privacy, metrics, stability.

**Scenarios**
- **G**: Logs enabled. **W**: Run flow with media. **T**: No raw media logged; PII scrubbed.
- **G**: Metrics enabled. **W**: Use app. **T**: Events for card taps, claims, completions, retries, cancels appear.
- **G**: Crash loop test (kill at various states). **W**: Relaunch. **T**: No crash; state recovered.

**Pass/Fail:** App Store privacy answers match; crash-free launch rate ≥ target.
