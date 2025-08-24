
# Workflow Instances & Tool Cards — iPhone App UI Plan (SwiftUI)

> Scope: iPhone-first. iPad/Mac adapt with `NavigationSplitView`. Thumbnails are a future upgrade—design below leaves clear hooks to add them later without breaking UX.

---

## 1) Top-Level Navigation & Screens

**Home (“Workflows”)**  
- `NavigationStack` → `List` of *workflow instance* rows (cards).  
- Controls: `.searchable(text:)`, segmented `Picker` filter (All/Active/Done/Failed), `Menu` for sort (Recent/Name).  
- Rows include: title, status badge, progress (e.g., `2/5`), “updated _2m_ ago”. *(Thumbnail placeholder reserved; hidden by default.)*  
- Swipe actions: Duplicate, Archive, Delete.  
- Toolbar primary: “+ New from Template.”  
- Row tap: **push** → `WorkflowDetailView(id:)` (the vertical list of **ToolCards**).

**Workflow Detail**  
- Scrollable vertical stack (or `List`) of **ToolCards** in the order defined by the template.  
- Sticky toolbar actions: *Run all remaining*, *Pause/Resume polling*, *Duplicate workflow*.
- Each ToolCard handles: parameters, run, progress, logs, outputs, history, errors, and dependency gating.

---

## 2) Data Model (short recap — compatible with SwiftData/Core Data)

```swift
@Model final class WorkflowInstance { /* ... */ var tools: [ToolInstance] }
@Model final class ToolInstance { var runs: [ToolRun]; var state: ToolState; /* ... */ }
@Model final class ToolRun { var progress: Double; var jobID: String?; var producedAssets: [AssetRef]; /* ... */ }
enum ToolState: String, Codable { case blocked, ready, runningLocal, queuedRemote, runningRemote, downloading, succeeded, failed, cancelled }
struct AssetRef: Codable { var id: String; var type: String; var url: URL; var metaJSON: Data?; var createdAt: Date }
```

**Keys**  
- Template defines `requiredKeys` and `producedKeys` (e.g., `"image"`, `"mask"`, `"depth"`).  
- Instance stores `selectedOutputs: [String: String]` mapping key → `AssetRef.id` chosen by the user.  
- **Enable rule:** step is enabled when `requiredKeys ⊆ availableKeys`.

---

## 3) ToolCard UX Spec (what each card shows)

**Collapsed state (default when idle/complete):**
- Title + icon (tool kind).  
- State badge: *Blocked*, *Ready*, *Running 42%*, *Downloading*, *Done*, *Failed*.  
- Mini summary: last run’s result type (image/video/json) + created time.  
- **Primary actions:** `Run` (enabled when dependencies satisfied), `More (…)` menu.

**Expanded state (tap to expand / `DisclosureGroup`):**
1. **Parameters Panel**  
   - Editable form rows (`TextField`, `Picker`, `Toggle`, `Stepper`) bound to `paramsJSON` via view model.  
   - “Reset to defaults” action.  
   - Optional presets menu.  
2. **Inputs Panel** *(read-only)*  
   - Shows which upstream outputs are fulfilling each `requiredKey`.  
   - If multiple candidate outputs exist upstream, a **Select Input** button opens a *sheet* picker (grid or list).  
3. **Run & Status**  
   - `Run` / `Cancel` / `Retry` buttons.  
   - `ProgressView(value:)` + “updated _Xs_ ago”.  
   - Short **log tail** (monospaced caption, max ~6 lines) with an expand-to-full-logs sheet.  
4. **Outputs Panel**  
   - Placeholder tiles for each `producedKey` (e.g., Image, Video, JSON).  
   - When a result exists: show tiny preview, **Use for next step** action chip (sets `selectedOutputs`).  
   - “Open” → full-screen viewer for media; JSON opens a pretty-viewer sheet.  
5. **History**  
   - Inline last 3 runs with status/time; *View All* pushes `RunHistoryView`.

**Error & Empty states**  
- Friendly error line with code + short hint, `Retry` preserves last params.  
- If blocked → surface **why**: “Waiting for *Mask* from *Step 2*”.

---

## 4) Card Layout & Components (SwiftUI)

```swift
struct ToolCardView: View {
    @Bindable var tool: ToolInstanceVM
    @State private var showParams = false
    @State private var showSelectInput: String? // key being selected
    @State private var showLogs = false

    var body: some View {
        Section {
            header

            if showParams { ParametersEditor(tool: tool) }
            InputsPanel(tool: tool, showSelectInput: $showSelectInput)
            RunBar(tool: tool)

            if tool.isRunning {
                ProgressView(value: tool.progress)
                LogTailView(text: tool.logTail, onExpand: { showLogs = true })
            }

            OutputsPanel(tool: tool)
            HistoryStrip(tool: tool)
        } header: {
            ToolHeader(tool: tool, onToggleParams: { showParams.toggle() })
        }
        .sheet(item: $showSelectInput) { key in
            InputPickerSheet(tool: tool, key: key)
        }
        .sheet(isPresented: $showLogs) {
            LogsSheet(run: tool.currentRun)
        }
    }
}
```

**Header**  
- `Label(tool.displayName, systemImage: tool.icon)` + `StatusBadge`.  
- Right-aligned buttons: *Run* (primary), *… menu* with *Reset Params*, *Duplicate Step*, *Delete Step*.

**RunBar**  
- `Run` (enabled), `Cancel` (only while running), `Retry` (after failure).  
- Lightweight “Est. remaining” text if server provides ETA.

**OutputsPanel**  
- For each produced key: `OutputTile`.  
- Tile shows preview if available; otherwise a placeholder with key name and a “waiting” spinner when upstream is running.  
- Tap tile → *sheet* (image/video viewer) with actions: *Use for Key X in next step*, *Share/Save*, *Open in…*.

---

## 5) Dependency Gating (dataflow → enabling)

```swift
extension ToolInstanceVM {
    var missingKeys: [String] { /* compare requiredKeys vs availableKeys */ }
    var isEnabled: Bool { missingKeys.isEmpty && state.isUserInvokable }
}
```

- Show disabled `Run` with sub-caption: “Waiting for **\(missingKeys.joined)**.”  
- Auto-scroll to the card that just became `ready` using `ScrollViewReader`.

---

## 6) Job Lifecycle & Polling UI

**States & transitions:**  
`ready → runningLocal | queuedRemote → runningRemote → downloading → succeeded | failed | cancelled`

**UI rules:**  
- While `queuedRemote/runningRemote` → show progress percent if provided; else indeterminate.  
- On connection loss, badge switches to “Reconnecting…”; keep polling with backoff.  
- When results arrive, “New result” haptic + briefly highlight the `OutputTile`.

---

## 7) History & Re-runs

- Each press of **Run** creates a **ToolRun**.  
- History strip shows last N with colored dots, duration, and quick **Re-run**.  
- *View All* pushes a table (date, status, duration, params diff).  
- **Compare** action shows params diff vs current.

---

## 8) Parameters Editing Patterns

- Group related controls in `Section`s; provide inline help (`info` button → sheet).  
- Persist edits instantly; maintain **dirty** flag to warn before navigating away.  
- “Reset to defaults” sets `paramsJSON` = `defaultParamsJSON`.  
- Optional **presets** via `Menu` → apply deltas to params JSON.

---

## 9) Media Handling

- Image/video previews are lightweight thumbnails.  
- Full-screen viewer via `.fullScreenCover` for immersive editors.  
- JSON outputs: syntax-highlighted viewer with copy/download.  
- “Use for next step” writes `selectedOutputs[key] = asset.id` and updates gating.

---

## 10) Error UX

- Short, plain-language summary; show raw error code below (monospaced).  
- Helpful suggestions: “Check API key”, “Retry with smaller resolution”, etc.  
- One-tap **Retry** keeps the last successful inputs/params.

---

## 11) Accessibility

- All buttons labelled with `accessibilityLabel`.  
- `StatusBadge` uses SF Symbols + text (don’t color-only encode).  
- Large Content Viewer: ensure long titles wrap; use `.minimumScaleFactor(0.8)` as needed.  
- Dynamic Type: prefer `List`/`Form` for automatic spacing.

---

## 12) Performance & Offline

- Large lists: use `List` with `ForEach` (stable IDs) and lazy thumbnails (future).  
- Cache recent `AssetRef` previews; avoid logging raw media (respect privacy).  
- Offline: keep **Run** disabled for remote-only tools; allow local tools. Resume when online.

---

## 13) View Models & Services (sketch)

```swift
@MainActor final class WorkflowsVM: ObservableObject {
    @Published var items: [WorkflowInstanceVM] = []
    @Published var searchText = ""
    @Published var filter: Filter = .all
    @Published var sort: Sort = .recent
    var filtered: [WorkflowInstanceVM] { /* … */ }
    func newFromTemplate() { /* … */ }
    func duplicate(_ wf: WorkflowInstanceVM) { /* … */ }
    func archive(_ wf: WorkflowInstanceVM) { /* … */ }
    func delete(_ wf: WorkflowInstanceVM) { /* … */ }
}

@MainActor final class ToolInstanceVM: ObservableObject, Identifiable {
    // Wraps ToolInstance model + derived UI state
    @Published var state: ToolState
    @Published var progress: Double = 0
    @Published var logTail: String? = nil
    var requiredKeys: [String]; var producedKeys: [String]
    var currentRun: ToolRunVM?
    var isRunning: Bool { state == .runningLocal || state == .runningRemote || state == .downloading }
    var isEnabled: Bool { /* gating */ }
    func run() async { /* submit to JobManager */ }
    func cancel() async { /* … */ }
    func selectOutput(_ asset: AssetRef, for key: String) { /* … */ }
}
```

`JobManager` actor handles idempotent submission and polling (10s cadence), persisting after every state change for app relaunch recovery.

---

## 14) Styling Tokens (keep it consistent)

- Corner radius: `12` for cards, `20` for tiles.  
- Spacing: `8 / 12 / 16`.  
- Color roles:  
  - Primary text, Secondary text  
  - Status: info (blue), success (green), warning (yellow), danger (red), running (indigo).  
- Badge icons: `clock` (queued), `arrow.triangle.2.circlepath` (running), `square.and.arrow.down` (downloading), `checkmark.circle` (done), `xmark.circle` (failed).

---

## 15) Testing Checklist

- Gating: next step disabled until required keys satisfied.  
- Resume: kill app mid-run; relaunch restores progress and continues polling.  
- Network flap: simulate offline; ensure graceful reconnect.  
- Large outputs: many assets; performance stays smooth.  
- Multiple instances: run two workflows simultaneously; UI stays separated.  
- Accessibility: VoiceOver reads statuses and buttons clearly.

---

## 16) Future Upgrade: Thumbnails

- Add `thumbnailURL` to `AssetRef.metaJSON`.  
- `WorkflowRow` shows first available image/video thumbnail; fallback to type icon.  
- Lazy load via `AsyncImage`; cache by `AssetRef.id`.  
- Grid mode: add a `LazyVGrid` toggle when thumbnails become primary.

---

## 17) Example: WorkflowRow (thumbnail-ready, thumbnail hidden by default)

```swift
struct WorkflowRow: View {
    let wf: WorkflowInstanceVM
    var body: some View {
        HStack(spacing: 12) {
            if let thumb = wf.thumbnail, wf.showThumbnails {
                ThumbnailView(asset: thumb)
                    .frame(width: 52, height: 52)
                    .clipShape(RoundedRectangle(cornerRadius: 10))
                    .transition(.opacity.combined(with: .scale))
            }
            VStack(alignment: .leading, spacing: 4) {
                HStack {
                    Text(wf.title).font(.headline).lineLimit(1)
                    Spacer()
                    StatusBadge(status: wf.status)
                }
                HStack(spacing: 8) {
                    ProgressView(value: wf.progress).frame(width: 120)
                    Text("\\(wf.completedSteps)/\\(wf.totalSteps) steps").font(.caption2).foregroundStyle(.secondary)
                    Text("· updated \\(wf.relativeUpdated)").font(.caption2).foregroundStyle(.secondary)
                }
            }
        }
        .contentShape(Rectangle())
    }
}
```

---

## 18) Example: ToolCard Header & RunBar

```swift
struct ToolHeader: View {
    let tool: ToolInstanceVM
    var onToggleParams: () -> Void
    var body: some View {
        HStack(alignment: .firstTextBaseline) {
            Label(tool.displayName, systemImage: tool.icon)
            Spacer()
            StatusBadge(status: tool.state)
            Button("Params") { onToggleParams() }
        }
    }
}

struct RunBar: View {
    @Bindable var tool: ToolInstanceVM
    var body: some View {
        HStack {
            Button("Run") { Task { await tool.run() } }
                .buttonStyle(.borderedProminent)
                .disabled(!tool.isEnabled)
            if tool.isRunning {
                Button("Cancel") { Task { await tool.cancel() } }
            } else if tool.state == .failed {
                Button("Retry") { Task { await tool.run() } }
            }
            Spacer()
            Text(tool.updatedRelative).font(.caption).foregroundStyle(.secondary)
        }
    }
}
```

---

## 19) Conclusion

- Use **List + Navigation push** for workflow instances; **ToolCards** live in the detail page.  
- Gate steps with required/produced keys, surface why a step is blocked, and make selecting inputs explicit.  
- Keep logs/results close to the action, with full-screen editors reserved for heavy media tasks.  
- The design is thumbnail-ready but avoids visual clutter until you enable it.
