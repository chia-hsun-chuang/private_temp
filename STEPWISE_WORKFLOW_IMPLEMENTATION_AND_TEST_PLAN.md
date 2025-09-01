# Stepwise Workflow System Implementation & Test Plan (iOS/macOS, Swift/SwiftUI)

> Goal: Build the workflow system incrementally. Each step includes **Definition of Done (DoD)**, **self-validating tests (XCTest)**, and **secondary manual procedures**. All tests run in CI and must pass before moving on.

---

## 0) Foundation & CI scaffolding
**Build:** Xcode project with 3 targets: `App`, `Engine` (pure Swift), `EngineTests` (XCTest). Add SwiftLint, swift-format, Danger (optional), and unit-test code coverage gate (e.g., ≥80% on `Engine`).

**DoD**
- Repo boots with `swift test` or Xcode test scheme.
- CI (GitHub Actions) runs `xcodebuild test` on push/PR and enforces coverage gate.
- Logging is structured; **no media payloads in logs** (unit test asserts).

**Self-validating tests**
- `LoggingPolicyTests`: verify no “.png”, “.jpg”, or raw `Data` is logged.
- `CoverageCheck` (script): fail build if coverage < threshold.

**Manual**
- Open workspace, run tests. Confirm CI badge green.

---

## 1) Core domain model (pure Swift types)
**Build:** Define the domain types as `Codable`, `Sendable`, `Equatable` where possible.

- `WorkflowTemplate` (id, name, nodes, edges, inputPorts, outputPorts, schemaVersion)
- `NodeTemplate` (id, toolKind, inputBindings, outputDecls, awaitSpec?)
- `WorkflowInstance` (id, templateRef, bindings, createdAt, updatedAt, schemaVersion)
- `ToolInstance` (id, nodeId, state, runs: [ToolRun], lastError?, awaitTicket?)
- `ToolRun` (id, startedAt, finishedAt?, jobID?, progress, outputs[], metrics)
- `AssetRef` (id, kind: image/video/text/file, uri, hash, bytes?, metadata)
- `ToolState` enum: `.ready`, `.queuedRemote`, `.runningRemote`, `.runningLocal`, `.awaitingUser`, `.succeeded`, `.failed(error)`, `.cancelled`
- `AwaitRequest` (workflowId, nodeId, runId, formSpec, createdAt, status: pending/snoozed/completed)

**DoD**
- Models compile; JSON encode/decode survives a round-trip.
- Backward-compat shim for enum cases via custom `init(from:)` (future-proof).

**Self-validating tests**
- `ModelRoundTripTests`: random instances -> encode -> decode → equal (seeded RNG).
- `EnumBackcompatTests`: decode legacy JSON fixtures (renamed/added cases) succeed.

**Manual**
- Generate a sample JSON for a simple 2-node workflow, inspect readability.

**Example: ToolState Codable shim**
```swift
enum ToolState: Equatable, Codable, Sendable {
    case ready, queuedRemote, runningRemote, runningLocal, awaitingUser
    case succeeded, failed(String), cancelled

    private enum K: String, Codable { case ready, queuedRemote, runningRemote, runningLocal, awaitingUser, succeeded, failed, cancelled }

    init(from decoder: Decoder) throws {
        let c = try decoder.singleValueContainer()
        if let raw = try? c.decode(String.self), raw == "blocked" { // legacy rename example
            self = .awaitingUser
            return
        }
        let keyed = try decoder.container(keyedBy: K.self)
        if keyed.contains(.failed) {
            let reason = try keyed.decode(String.self, forKey: .failed)
            self = .failed(reason)
        } else if let k = try? c.decode(K.self) {
            switch k {
            case .ready: self = .ready
            case .queuedRemote: self = .queuedRemote
            case .runningRemote: self = .runningRemote
            case .runningLocal: self = .runningLocal
            case .awaitingUser: self = .awaitingUser
            case .succeeded: self = .succeeded
            case .cancelled: self = .cancelled
            case .failed: self = .failed("")
            }
        } else {
            self = .ready
        }
    }

    func encode(to encoder: Encoder) throws {
        var c = encoder.singleValueContainer()
        switch self {
        case .failed(let msg):
            var keyed = encoder.container(keyedBy: K.self)
            try keyed.encode(msg, forKey: .failed)
        case .ready: try c.encode(K.ready)
        case .queuedRemote: try c.encode(K.queuedRemote)
        case .runningRemote: try c.encode(K.runningRemote)
        case .runningLocal: try c.encode(K.runningLocal)
        case .awaitingUser: try c.encode(K.awaitingUser)
        case .succeeded: try c.encode(K.succeeded)
        case .cancelled: try c.encode(K.cancelled)
        }
    }
}
```

---

## 2) In-memory Runtime (deterministic, no I/O)
**Build:** A tiny deterministic scheduler executing a DAG of nodes whose “tools” are pure functions (local-only).

- `Runtime` with `register(graph:)`, `start(instance:)`, `tick()`
- Topological sort; detect cycles with Kahn’s algorithm.
- State transitions: `.ready → runningLocal → succeeded/failed`

**DoD**
- Given a valid DAG and pure tools, `Runtime` processes to completion.
- Cycles are rejected with a meaningful error.

**Self-validating tests**
- `GraphOrderTests`: ensure topological order; cycle detection works.
- `StateMachineTests`: verify legal/illegal transitions.
- Property-based: With random small DAGs (acyclic), runtime terminates; with random single back-edge, detect cycle.

**Manual**
- Run a 3-node sample (Text→Uppercase→Append) and verify outputs printed.

**Example: Transition guard**
```swift
func canTransition(from: ToolState, to: ToolState) -> Bool {
    switch (from, to) {
    case (.ready, .runningLocal), (.runningLocal, .succeeded), (.runningLocal, .failed):
        return true
    default: return false
    }
}
```

---

## 3) Persistence Layer (SwiftData/Core Data) + Rehydration
**Build:** Map domain to persistence models with `schemaVersion` and migrations. Write-through on each mutation. Rehydration entrypoint to reconstruct runtime state.

**DoD**
- Mutations persist immediately; kill app mid-run and re-launch → instances render from disk.
- Migration harness exists with a sample v1→v2 migration.
- Rehydrate resumes polling-ready nodes later (next step).

**Self-validating tests**
- `PersistenceRoundTripTests`: create → persist → fetch → equal at the semantic level.
- `RehydrationRenderTests`: after “quit”, detail screen shows the same persisted states.
- `MigrationTests`: v1 fixture loads into v2 schema correctly.

**Manual**
- Start instance, kill app, relaunch: list shows same states (no UI editors auto-present).

**Example: Rehydrate signature**
```swift
@MainActor
protocol Rehydrator {
    func rehydrateAll() async throws -> [WorkflowInstance]
}
```

---

## 4) JobManager & Remote Adapters (10s polling, idempotent)
**Build:** Protocol-based remote adapter; submission idempotency; resume polling when app relaunches.

- `RemoteProviderAdapter` with `submit(job:)`, `status(jobID:)`, `cancel(jobID:)`
- `JobManager` with exponential backoff, 10s base poll, jitter, cancellation, dedupe.

**DoD**
- Submitting when a `jobID` exists is a no-op (idempotent).
- On relaunch, jobs in `.queuedRemote` or `.runningRemote` resume polling (not resubmitted).

**Self-validating tests**
- `IdempotencyTests`: duplicate submit doesn’t create a new remote job.
- `ResumePollingTests`: persist, “quit”, rehydrate → polling resumes and reconciles to success/failure.
- `BackoffTests`: transient 5xx errors back off; 4xx mark failed.

**Manual**
- Run with a Fake provider that transitions Queued→Running→Succeeded; watch state reconcile in UI.

**Example: Adapter protocol**
```swift
protocol RemoteProviderAdapter: Sendable {
    func submit(_ payload: Data) async throws -> String // jobID
    func status(jobID: String) async throws -> RemoteStatus
    func cancel(jobID: String) async throws
}
```

---

## 5) Await/Interactive Nodes
**Build:** Await Center that persists `AwaitRequest`s and guarantees uniqueness per `(workflowId,nodeId,runId)`; supports snooze/complete.

**DoD**
- Await requests survive app restart and appear in an Action Center inbox.
- Completing an await advances the node and unblocks dependents.

**Self-validating tests**
- `AwaitUniquenessTests`: duplicates prevented.
- `AwaitLifecycleTests`: create→snooze→complete transitions and persistence.

**Manual**
- Trigger an await node; restart app; confirm the same item remains pending.

---

## 6) Action Center UI (minimal)
**Build:** A screen listing pending/snoozed awaits; tapping opens appropriate editor/sheet (not auto-presented).

**DoD**
- UI renders persisted awaits; completing one updates the underlying instance and list.

**Self-validating tests**
- UI test: launch with seed DB; verify count and cell content; complete one and count decrements.

**Manual**
- Use sample await workflow; complete inputs; downstream nodes proceed.

---

## 7) Reconciliation & Interrupted Local Work
**Build:** On rehydrate, mark `.runningLocal` as `.failed("interrupted")` and surface Retry; remote jobs reconcile via polling.

**DoD**
- UI accurately reflects last persisted state instantly, then updates as reconciliation arrives.
- Interrupted locals never silently “succeed”.

**Self-validating tests**
- `InterruptedLocalTests`: local-running → kill → rehydrate → show failed(interrupted); user taps Retry works.

**Manual**
- Force-quit during a long local node; relaunch shows Retry badge.

---

## 8) Tool SDK (extensibility)
**Build:** Public `Tool` protocol, registration, validation of IO contracts, and deterministic preview for pure tools.

**DoD**
- Third-party tool can register; IO validation prevents runtime mismatch.
- Preview function returns a small, cheap preview for UI thumbnails.

**Self-validating tests**
- `ToolContractTests`: wrong ports rejected with descriptive errors.
- `PreviewDeterminismTests`: same input → same preview bytes hash.

**Manual**
- Create a sample “UppercaseTextTool” plugin; verify preview in UI card.

---

## 9) Template Import/Export (JSON)
**Build:** Load/store templates from JSON with strict schema + versioning; friendly errors; optional comments via `x-` fields.

**DoD**
- Import validates DAG structure and port bindings; export is stable and diff-friendly.

**Self-validating tests**
- `TemplateValidationTests`: missing node/port is caught; cycle is caught; types match.
- Fuzz JSON with random field order; decoder robust.

**Manual**
- Import a provided template; instantiate; run end-to-end with fake adapters.

---

## 10) Asset Storage & Garbage Collection
**Build:** Asset registry; reference counting from outputs; background GC leaves N days of orphans; never logs media bytes.

**DoD**
- Assets are persisted and referenced by `AssetRef`; GC removes unreachable safely.

**Self-validating tests**
- `AssetRefcountTests`: delete upstream → orphaned child cleaned after threshold.
- `NoMediaInLogsTests`: ensure logging policy still holds.

**Manual**
- Run workflows that create images/videos; verify disk usage stabilizes after GC.

---

## 11) Scheduling & Fairness (multi-workflow)
**Build:** Cooperative scheduler across instances; ensures fair progress and caps concurrent remote jobs.

**DoD**
- No starvation; user-facing progress is monotonic and truthful.

**Self-validating tests**
- `FairnessTests`: simulate 10 workflows; assert each advances.
- `ConcurrencyCapTests`: remote submissions respect cap; queue length reported.

**Manual**
- Start many instances; observe smooth progress.

---

## 12) UI Polishing (List, Detail, Cards)
**Build:** Workflow List with search/filter/sort; Detail with node cards; badges for states, progress, last updated; thumbnails via preview API.

**DoD**
- Screens render entirely from persisted state; no hidden in-memory truth.

**Self-validating tests**
- Snapshot tests of key screens for several seeded DB states.
- Accessibility tests (labels, traits, dynamic type).

**Manual**
- Exercise filters/sorts; confirm determinism across relaunches.

---

## 13) Performance, Telemetry, Privacy
**Build:** Metric timers around steps; memory audit for large assets; privacy review: no media in logs, no PII in crash reports.

**DoD**
- P95 launch ≤ X ms with 100 instances; polling loop <2% CPU idle.
- Telemetry redaction verified.

**Self-validating tests**
- `MetricBudgetTests`: run seeded workload, assert budget not exceeded (using XCTest metrics APIs).
- `RedactionTests`: crash sample shows no sensitive fields.

**Manual**
- Instruments profile for leaks and allocations during a batch run.

---

## 14) End-to-End Sample Workflows (5 templates)
**Build:** Implement the five sample templates you provided (text→display, select image→display, text→image→display, etc.) using Fakes for remote tools first, then real adapters.

**DoD**
- Each template runs from start to finish with reproducible outputs under Fakes. Real adapters optional but gated behind keys.

**Self-validating tests**
- `E2E_Template1_to_5_Tests`: each scenario executes; assert final states and outputs’ metadata (not raw bytes).

**Manual**
- Run each template in the app; confirm UX and Action Center behavior.

---

## 15) Documentation & Dev Ergonomics
**Build:** Developer docs for adapter contracts, polling rules (10s), JSON template schema, persistence, and how to add a new tool.

**DoD**
- New engineer can add an adapter and a template in <1 day using docs.

**Self-validating tests**
- Lint docs (markdown links, code blocks compile in doctests where applicable).

**Manual**
- Follow the doc to add a trivial “ReverseTextTool” end-to-end.

---

# Example XCTest Snippets (copy into `EngineTests`)

## A. Model round-trip
```swift
import XCTest
@testable import Engine

final class ModelRoundTripTests: XCTestCase {
    func testToolStateRoundTrip() throws {
        let states: [ToolState] = [.ready, .queuedRemote, .runningRemote, .awaitingUser, .succeeded, .failed("oops"), .cancelled]
        for s in states {
            let data = try JSONEncoder().encode(s)
            let decoded = try JSONDecoder().decode(ToolState.self, from: data)
            XCTAssertEqual(s, decoded)
        }
    }
}
```

## B. Idempotent remote submission
```swift
final class IdempotencyTests: XCTestCase {
    func testDuplicateSubmitIsNoop() async throws {
        let fake = ScriptedAdapter(script: .init([.queued("job-123")]))
        let jm = JobManager(adapter: fake, pollInterval: 10)
        var node = ToolInstance(id: "T1", nodeId: "N1", state: .queuedRemote, runs: [ToolRun(id:"R1", startedAt: .now, jobID: "job-123", progress: 0, outputs: [], metrics: [:])])
        try await jm.ensureSubmitted(&node) // should detect jobID and not resubmit
        XCTAssertEqual(fake.submits, 0)
    }
}
```

## C. Resume polling on rehydrate
```swift
final class ResumePollingTests: XCTestCase {
    func testResumePolling() async throws {
        let fake = ScriptedAdapter(script: .init([.queued("job-1"), .running("job-1"), .succeeded("job-1")]))
        let jm = JobManager(adapter: fake, pollInterval: 10)

        // Simulate persisted node
        let node = ToolInstance(id: "T", nodeId: "N", state: .runningRemote,
                                runs: [ToolRun(id:"R", startedAt: .now, jobID: "job-1", progress: 0, outputs: [], metrics: [:])])

        let runtime = Runtime(jobManager: jm)
        try await runtime.rehydrate(nodes: [node])

        // wait a few poll ticks
        try await Task.sleep(nanoseconds: 35 * NSEC_PER_SEC)
        let final = try await runtime.node(withId: "T")
        guard case .succeeded = final.state else { XCTFail("Expected succeeded"); return }
    }
}
```

## D. Await uniqueness
```swift
final class AwaitUniquenessTests: XCTestCase {
    func testNoDuplicateAwait() throws {
        let center = AwaitCenter(store: InMemoryStore())
        let ticket1 = AwaitRequest(workflowId:"W1", nodeId:"N1", runId:"R1", formSpec:.text("desc"), createdAt:.now, status:.pending)
        try center.enqueue(ticket1)
        XCTAssertThrowsError(try center.enqueue(ticket1)) // same triple key
    }
}
```

## E. Interrupted local on rehydrate
```swift
final class InterruptedLocalTests: XCTestCase {
    func testInterruptedLocalMarksFailed() async throws {
        // node was running locally when app died
        var t = ToolInstance(id: "T1", nodeId: "N1", state: .runningLocal, runs: [ToolRun(id:"R1", startedAt:.now, jobID:nil, progress:40, outputs:[], metrics:[:])])
        let r = Runtime(jobManager: .noop)
        try await r.rehydrate(nodes: [t])
        let updated = try await r.node(withId: "T1")
        if case .failed(let msg) = updated.state {
            XCTAssertTrue(msg.contains("interrupted"))
        } else {
            XCTFail("Expected failed(interrupted)")
        }
    }
}
```

# Secondary Manual Test Procedures (per step)
For each step above, we include a brief manual check. Here’s the **golden path** E2E manual test after Steps 1–7:
1. Import the “Text → Uppercase → Display” template.
2. Create an instance; verify nodes progress `.ready → runningLocal → succeeded` and the UI shows outputs.
3. Create a template with an **await** node (e.g., “Enter caption”). When blocked, force-quit the app, relaunch, and open **Action Center** — the same await appears. Complete it; downstream nodes continue.
4. Run a remote job template with the **fake adapter**; kill the app during Running; relaunch → the card initially shows persisted state, then reconciles to **succeeded** once polling catches up. Confirm **no resubmission** (jobID unchanged).
5. Verify that any node that was `.runningLocal` when killed is now **failed (interrupted)** and offers **Retry**.

# Notes
- Polling interval is **10 seconds** (with small jitter). Don’t block the main thread; use `Task` and cooperative cancellation.
- Respect privacy: never log raw media or PII. Tests enforce this policy.
- All domain logic lives in `Engine` so it’s testable without UI. UI renders **persisted** truth only.
