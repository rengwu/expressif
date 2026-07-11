---
type: research
blocked_by: []
assets: [.plan/expressif-v1-spec/assets/opencode-surface.md]
---

# Map opencode's programmatic surface

## Question

Same mapping as [Map Claude Code's programmatic surface](02-map-claude-code-surface.md), for opencode's server mode / API:

- Session lifecycle, event stream shape and granularity, mid-session input, permission handling.
- MCP server wiring per session; whether `ui://` resources and tool `_meta` pass through to the API consumer.
- System prompt / rules / agent-definition injection points for agent awareness.
- Local-model support path (Ollama etc.) — this is our fully-offline story; note anything that breaks when no cloud provider is configured.
- Stability/versioning of the API surface across recent releases.

Deliver a cited summary asset. This feeds [Define the harness adapter contract](04-define-adapter-contract.md).

## Answer

Mapped in [opencode's programmatic surface](../assets/opencode-surface.md) (2026-07-11, opencode v1.17.18). The surface is `opencode serve` — a local HTTP server with an OpenAPI 3.1 contract — plus the MIT-licensed `@opencode-ai/sdk` generated from it; the TUI itself is a client of this API, so it is first-party. Highlights against the ticket's questions:

- **Lifecycle/events/input**: full session CRUD plus fork, abort, revert, and compaction server-side; SSE event bus with part-level updates *and* token deltas (`message.part.updated.delta`); prompting a busy session queues rather than erroring; `prompt_async` for fire-and-forget. Transcripts are server-owned and readable back.
- **Permissions**: `permission.updated` event → respond endpoint with `once`/`always`/`reject`. No input-rewriting on approval (unlike `canUseTool.updatedInput`) — the adapter contract's permission surface must treat rewriting as a capability flag.
- **Structured input**: a native `question` tool, near-isomorphic to Claude Code's `AskUserQuestion` (options with label+description, multi-select, free-text), exposed via `question.asked` events and reply/reject endpoints — one normalized "agent question" widget covers both harnesses.
- **MCP**: config-level (global/project, not per-session); full `CallToolResult` including `structuredContent` reaches the API consumer JSON-encoded; no `ui://`/Apps hosting (proxy-host pattern stands); tool descriptions stay in context by default — no tool-search caveat here.
- **Awareness**: the prompt body's `system` field is a per-message system-prompt injection with zero filesystem footprint — the cleanest hook mapped so far; agent definitions and AGENTS.md/`instructions` cover the file-based tiers.
- **Offline**: solid — `@ai-sdk/openai-compatible` (Ollama/LM Studio/llama.cpp) is compiled into the binary, the models.dev catalog has a bundled snapshot and kill-switch env vars; the real offline risk is local-model tool-calling quality, not plumbing.
- **Stability**: fast-moving, no compatibility guarantee, internals mid-migration — pin server+SDK versions together and smoke-test per release.
