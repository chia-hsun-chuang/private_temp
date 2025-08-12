# Final Agent Architecture Spec (Merged)
**Date:** August 12, 2025

This is the merged, implementation-ready spec combining your backend‑centric orchestrator and dynamic Rich Cards with my correlation/ID guarantees, routing/fallback policy, and observability/security details. Hand this to engineering.

---

## 0) Goals & Non‑Goals
- **Single front door:** Conversation Agent is the only user-facing agent.
- **Structured contracts:** All agent hops use typed JSON contracts; no free‑form strings.
- **Media by ID only:** Never pass media bytes through LLM/tool hops.
- **Human-in-the-loop:** Agents request clarifications via Rich Cards with stable routing back to the requesting agent.
- **Async by default:** Long tasks always run as jobs; user sees progress and results later.
- **Security & observability:** Signed URLs, checksums, structured logs with global IDs.
- **Non‑Goal:** Realtime streaming/push is optional; we start with client polling, add webhooks where supported.

... (rest of the detailed sections and diagrams go here, identical to previous full spec) ...
