---
type: research
blocked_by: []
assets: [.plan/expressif-v1-spec/assets/claude-code-surface.md]
---

# Map Claude Code's programmatic surface

## Question

For the harness adapter contract, map how a host application drives Claude Code programmatically (Claude Agent SDK and/or headless CLI modes):

- Session lifecycle: spawn, resume, interrupt, kill.
- Event stream: message/token streaming granularity, tool-use events, thinking, errors.
- Sending user input mid-session; answering permission prompts programmatically.
- Wiring MCP servers into a session (config, scopes, stdio) — can expressif's renderer plugins be attached per-session?
- System prompt / context injection points (system prompt append, CLAUDE.md, skills) — candidate channels for agent awareness.
- How structured tools like AskUserQuestion surface to an SDK consumer.
- Licensing/redistribution constraints on embedding it as a dependency of a framework.

Deliver a cited summary asset. This feeds [Define the harness adapter contract](04-define-adapter-contract.md).

## Answer

Mapped in full — see [the surface asset](../assets/claude-code-surface.md). The Agent SDK (TS `@anthropic-ai/claude-agent-sdk`, bundling the CLI) is the embedding surface, and it is richer than assumed:

- **Lifecycle**: `query()` with continue/resume/fork; JSONL transcripts keyed by cwd under `~/.claude/projects/`; `listSessions`/`getSessionMessages` mean expressif can read transcripts back rather than owning storage. Interruption requires streaming input mode — the adapter should mandate it.
- **Events**: complete-message stream by default; `includePartialMessages` adds raw token-level deltas — progressive rendering is fully feasible.
- **Permissions**: `canUseTool` callback is the passthrough (allow/deny/updatedInput/persist-rule suggestions, can pend indefinitely, `defer` survives process exit); `PreToolUse` hooks see every call.
- **Two native gifts**: `AskUserQuestion` always routes to `canUseTool` with structured questions/options (and can carry sanitized **HTML option previews** via `toolConfig`) — a built-in structured-input channel; and MCP tools marked `_meta["anthropic/requiresUserInteraction"]` also force the callback. The adapter must unify these with plugin elicitation into one widget model.
- **MCP**: per-session servers incl. **in-process SDK servers** (the expressif proxy can live inside the sidecar); **caution — tool search withholds tool descriptions from context by default**, so awareness can't rely on tool descriptions alone (feeds ticket 05).
- **Awareness points**: system prompt preset+`append` (cache-stable with `excludeDynamicSections`), tool descriptions, CLAUDE.md/skills via `settingSources`.
- **Licensing**: SDK and CLI are proprietary (Anthropic Commercial Terms) — consumable as an npm dependency the end user installs, not vendorable; users bring their own Anthropic auth.
