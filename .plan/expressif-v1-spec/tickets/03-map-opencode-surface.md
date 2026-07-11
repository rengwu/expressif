---
type: research
blocked_by: []
claimed_by: claude-fable-5-session-c94c4171
claimed_at: 2026-07-11T16:40:00+08:00
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
