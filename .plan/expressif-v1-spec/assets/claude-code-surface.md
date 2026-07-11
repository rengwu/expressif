# Claude Code's programmatic surface

Research asset for ticket [Map Claude Code's programmatic surface](../tickets/02-map-claude-code-surface.md). Investigated 2026-07-11 against the Claude Agent SDK docs (code.claude.com) and npm registry metadata. The Agent SDK (TypeScript `@anthropic-ai/claude-agent-sdk` v0.3.207, Python `claude-agent-sdk`) is the intended embedding surface — it bundles the CLI binary and runs it as a child process. The TS SDK matches expressif's TS core directly.

## Session lifecycle

- `query()` is the primitive; the experimental V2 session API (`createSession`) was **removed** in TS SDK 0.3.142 — do not design against it. ([sessions](https://code.claude.com/docs/en/agent-sdk/sessions))
- **Create**: each `query()` starts a session. **Continue** (`continue: true`) resumes the most recent session in the cwd; **resume** takes an explicit session ID; **fork** (`resume` + `forkSession: true`) branches history into a new ID. Session IDs arrive on the init `SystemMessage` and every `ResultMessage`.
- **Persistence**: transcripts are JSONL under `~/.claude/projects/<encoded-cwd>/<session-id>.jsonl` (cwd-keyed — a resume from a different cwd silently misses). `listSessions()` / `getSessionMessages()` / `getSessionInfo()` / `renameSession()` / `tagSession()` exist for building session pickers and transcript viewers — directly relevant to expressif's conversation-persistence question. TS-only `persistSession: false` for in-memory sessions.
- **Interrupt**: real-time interruption requires streaming input mode; single-message mode cannot interrupt. Kill = terminate the child process; the JSONL transcript survives for resume.

## Event stream

- Default granularity: complete messages — init `SystemMessage` (session ID, MCP server connection status, available tools), `AssistantMessage` (content blocks incl. `tool_use`), tool results, `ResultMessage` (result text, cost, usage, session ID), and a compact-boundary message when history is compacted.
- **Token-level streaming exists**: `includePartialMessages: true` adds `stream_event` messages carrying raw Claude API stream events (`content_block_start/delta/stop`, `text_delta`, `input_json_delta`, `ttft_ms`). Subagent deltas are not forwarded — only complete messages carry `parent_tool_use_id`. ([streaming output](https://code.claude.com/docs/en/agent-sdk/streaming-output))
- Consequence for expressif: progressive markdown rendering and live tool-call display are fully feasible; the adapter's normalized event model can offer both "complete message" and "delta" tiers.

## Sending input mid-session

Streaming input mode (AsyncGenerator prompt / Python `ClaudeSDKClient`) is the recommended mode and the only one supporting queued messages, images, interruption, and (in Python) `canUseTool` at all. Python caveat: a *finite* message generator closes the input stream before permission callbacks can fire unless a registered hook or in-process MCP server holds it open. ([streaming vs single](https://code.claude.com/docs/en/agent-sdk/streaming-vs-single-mode), [user input](https://code.claude.com/docs/en/agent-sdk/user-input))

## Permissions — the passthrough surface

Six-step evaluation per tool call: hooks → deny rules → ask rules → permission mode → allow rules → `canUseTool`. ([permissions](https://code.claude.com/docs/en/agent-sdk/permissions))

- **`canUseTool` is expressif's permission-prompt hook**: async callback receiving `(toolName, input, {suggestions, signal})`, returning allow (optionally with `updatedInput` and `updatedPermissions` to persist a rule) or deny with a message the model sees. It can stay pending indefinitely — and the **`defer` hook decision** lets the process exit and resume later from the persisted session, which matters for approvals a user answers hours later.
- Auto-approved tools never reach the callback; `PreToolUse` hooks are the only gate that sees every call. `PermissionRequest` hook exists for external notifications.
- Modes: `default`, `dontAsk`, `acceptEdits`, `bypassPermissions`, `plan`, `auto`; switchable mid-session (`setPermissionMode`).

## AskUserQuestion — a native structured-input channel

`AskUserQuestion` always routes to `canUseTool` even when allow rules match. The consumer receives 1–4 questions (2–4 options each, `header`, `multiSelect`), renders them however it likes, and returns selections via `updatedInput: {questions, answers}`, with free-text answers and a whole-card free-text `response` supported. Not available in subagents. ([user input](https://code.claude.com/docs/en/agent-sdk/user-input))

**Notable**: `toolConfig.askUserQuestion.previewFormat: "html"` makes Claude attach an HTML preview fragment to options (SDK strips `<script>`/`<style>` before the callback) — a built-in, sanctioned "rich choice card" channel that expressif's chat surface can render natively.

Implication: expressif has **two** structured-input paths on this harness — native `AskUserQuestion` (agent-initiated clarification) and plugin elicitation via the proxy-host (skill/plugin-driven widgets). The adapter contract must present both to the chat surface as one widget model.

## MCP wiring

- Per-session via the `mcpServers` option (stdio command, `http`/`sse`, or **in-process SDK servers** — custom tools running inside the consumer's own process, an attractive home for expressif's proxy) or `.mcp.json` with `settingSources: ["project"]`. ([SDK MCP](https://code.claude.com/docs/en/agent-sdk/mcp))
- Tools are named `mcp__<server>__<tool>`; auto-approve with `allowedTools` (server-scoped wildcards). Init message reports per-server connection status.
- **`_meta["anthropic/requiresUserInteraction"]`** (≥ v2.1.199): an MCP server can mark a tool so it always falls through to `canUseTool` even when allowed — a second sanctioned interception point for expressif's input widgets.
- **Caution for awareness**: MCP **tool search is enabled by default** — tool definitions are withheld from context and loaded on demand. Expressif cannot assume its renderer tools' descriptions are always in the model's context; the awareness ticket must account for this (disable tool search for expressif's server, or carry awareness in the system prompt).

## Awareness injection points

Ranked for expressif's purposes ([modifying system prompts](https://code.claude.com/docs/en/agent-sdk/modifying-system-prompts)):

1. `systemPrompt: {type: "preset", preset: "claude_code", append: "…"}` — adds instructions without losing Claude Code's tool guidance; `excludeDynamicSections: true` keeps the appended prompt cache-stable across sessions/machines.
2. Tool descriptions on the proxy's MCP tools (subject to the tool-search caveat above).
3. CLAUDE.md / skills / output styles via `settingSources` — file-based, good for scenario packs, but touches the user's project.
4. Per-message context injection through streaming input.

## Licensing / redistribution

Both npm packages (`@anthropic-ai/claude-agent-sdk` 0.3.207, `@anthropic-ai/claude-code` 2.1.207) are **proprietary** — `license: "SEE LICENSE IN README.md"`, governed by Anthropic's Commercial Terms (verified via `npm view`). expressif can declare the SDK as a dependency/peer dependency that the consumer installs from npm, but cannot vendor or redistribute it, and every end user needs their own Anthropic authentication (subscription or API key). This asymmetry with open-source opencode is an argument for keeping harness adapters as separately installable packages.

## Adapter-contract implications (feeds tickets 04, 05, 06, 07, 10)

- The adapter should require **streaming input mode** on this harness; single-message mode lacks interruption and (in Python) permission callbacks.
- Normalized events can be two-tier: complete messages always, deltas capability-gated.
- Permission passthrough maps cleanly onto `canUseTool` + hooks; long-pending approvals should consider `defer`.
- AskUserQuestion and elicitation must unify into one widget model at the chat surface.
- Session persistence/resume/fork is rich enough that expressif's core need not own transcript storage on this harness — read it back via `getSessionMessages()`.
- The in-process MCP server option means the expressif proxy can live inside the sidecar process with no extra child process.

## Sources

- Sessions: https://code.claude.com/docs/en/agent-sdk/sessions
- Streaming input modes: https://code.claude.com/docs/en/agent-sdk/streaming-vs-single-mode
- Streaming output: https://code.claude.com/docs/en/agent-sdk/streaming-output
- Permissions: https://code.claude.com/docs/en/agent-sdk/permissions
- Approvals & AskUserQuestion: https://code.claude.com/docs/en/agent-sdk/user-input
- MCP in the SDK: https://code.claude.com/docs/en/agent-sdk/mcp
- System prompts: https://code.claude.com/docs/en/agent-sdk/modifying-system-prompts
- npm registry metadata: `npm view @anthropic-ai/claude-agent-sdk` / `@anthropic-ai/claude-code` (2026-07-11)
