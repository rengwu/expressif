# opencode's programmatic surface

Research asset for ticket [Map opencode's programmatic surface](../tickets/03-map-opencode-surface.md). Investigated 2026-07-11 against opencode's docs (opencode.ai), the source on the `dev` branch of github.com/anomalyco/opencode (the repo moved there from sst/opencode), and npm metadata. The surface is `opencode serve` — a local HTTP server (default `127.0.0.1:4096`) publishing an OpenAPI 3.1 spec at `/doc` — plus the TS SDK `@opencode-ai/sdk` (v1.17.18, **MIT**), whose types are generated from that spec. `createOpencode()` spawns server+client together; `createOpencodeClient()` attaches to a running server. Optional HTTP basic auth via `OPENCODE_SERVER_PASSWORD`/`USERNAME`.

## Session lifecycle

- Sessions are server objects: `POST /session` creates, plus list/get/children/update/delete, **fork** (`Session.parentID`, forking at a message), **abort** (`POST /session/:id/abort` — the interrupt), summarize (compaction), share/unshare, and revert/unrevert (checkpoint rollback with snapshot diffs). ([server docs](https://opencode.ai/docs/server/), `server/routes/instance/httpapi/groups/session.ts`)
- Persistence is server-owned: messages and parts are stored per session and read back via `GET /session/:id/message`; sessions carry `projectID`, `directory`, `title`, `version`, `time`. Transcript storage/replay lives in the harness, as with Claude Code.
- The whole TUI runs on this same API (the TUI is a client of the server), so the surface is first-party, not a bolt-on.

## Sending input mid-session

- `POST /session/:id/message` (`session.prompt()`) is synchronous — resolves when the turn completes; `POST /session/:id/prompt_async` returns immediately. Body: `parts` (`text` | `file` | `agent` | `subtask`), `model: {providerID, modelID}`, `agent`, **`system`**, `tools: {name: bool}`, `noReply`, optional structured-output JSON schema.
- **Prompting a busy session queues**: the message is persisted and the in-flight run loop picks it up; prompt endpoints do not raise `SessionBusyError` (only `shell`, `revert`, `unrevert`, `deleteMessage` do — `effect/runner.ts` `ensureRunning` awaits the current run rather than rejecting).
- `noReply: true` appends a user message without triggering a response — a sanctioned context-injection channel.

## Event stream

- `GET /event` (SSE): first event `server.connected`, then every bus event; `GET /global/event` for cross-instance. SDK: `event.subscribe()`. ([server docs](https://opencode.ai/docs/server/))
- **Token-level deltas exist**: `message.part.updated` carries optional `delta?: string` for incremental text (`EventMessagePartUpdated` in `packages/sdk/js/src/gen/types.gen.ts`); reasoning parts stream the same way (`updatePartDelta` in `session/processor.ts`).
- Event vocabulary is rich: `session.created/updated/deleted/idle/status/error/compacted/diff`, `message.updated/removed`, `message.part.updated/removed`, `permission.updated/replied`, `question.asked/replied/rejected`, `file.edited`, `todo.updated`, `command.executed`, etc.
- Part types: `text`, `reasoning`, `tool`, `file`, `step-start`/`step-finish` (with token usage/cost), `patch`, `agent`, `snapshot`. `ToolPart.state` is a four-state machine `pending → running → completed | error`; `ToolStateCompleted` = `{input, output: string, title, metadata: record, attachments?: FilePart[], time}`.
- Consequence: progressive rendering gets **both** tiers natively (full part updates + text deltas), and tool lifecycle is more granular than Claude Code's (running-state updates with `title`/`metadata` mid-flight).

## Permissions — the passthrough surface

- Config: `permission` rules per type (`edit`, `bash`, `webfetch`, `question`, `read`, `external_directory`, `doom_loop`, …) with `allow` / `ask` / `deny` and pattern matching, last match wins; overridable per agent. ([permissions docs](https://opencode.ai/docs/permissions/))
- Runtime flow for an API consumer: a gated call emits `permission.updated` (payload: `id`, `type`, `pattern`, `sessionID`, `messageID`, `callID`, `title`, `metadata`, `time`); the client replies `POST /session/:id/permissions/:permissionID` with `{response: "once" | "always" | "reject"}`. No timeout — it blocks until answered.
- Unlike Claude Code there is no input-rewriting on approval (`canUseTool`'s `updatedInput` has no counterpart): the reply is approve/reject only.

## The question tool — native structured input

opencode has a first-party `question` tool, near-isomorphic to Claude Code's `AskUserQuestion` (`packages/schema/src/v1/question.ts`):

- Schema per question: `question`, `header` (≤30 chars), `options: [{label, description}]`, `multiple?: bool`, and `custom?: bool` (free-text answer, default **true**).
- Surfaced to clients as the `question.asked` bus event plus `GET /question` (pending requests); answered via `POST /question/:requestID/reply` with `{answers: string[][]}` (selected labels per question, order-matched) or `POST /question/:requestID/reject`. No timeout — pending indefinitely; rejected on server shutdown.
- Caveats: a known headless-mode hang when no client answers ([issue #10012](https://github.com/anomalyco/opencode/issues/10012)) — expressif always has a client attached, but the adapter must answer or reject every request it sees. The gap is ACP-only ([issue #30343](https://github.com/anomalyco/opencode/issues/30343)); the HTTP API expressif would use exposes it fully.

So this harness, too, has **two** sanctioned structured-input channels — the question tool and elicitation via the proxy MCP server — to unify into one widget model.

## MCP wiring

- Config `mcp` key: `"type": "local"` (`command`, `environment`, `cwd`, `timeout`) or `"type": "remote"` (`url`, `headers`, automatic OAuth). Tools enable/disable via the `tools` map globally, per agent, or **per prompt** (the `tools` field on the prompt body). ([MCP docs](https://opencode.ai/docs/mcp-servers/))
- **Config is global/project-level, not per-session** — all sessions on a server share the MCP set. Per-session variation requires separate server instances or per-prompt tool toggles.
- Result passthrough: `mcp/catalog.ts` `convertTool` preserves the full `CallToolResult` (content array, `structuredContent`); `session/processor.ts` `toolResultOutput` then JSON-stringifies any non-opencode-shaped result into `ToolPart.state.output`, putting the record into `state.metadata`. So structured MCP results **do reach the API consumer**, JSON-encoded in output/metadata — but nothing interprets them: opencode is not an MCP Apps host, has no `ui://` handling, and image content in tool results becomes `FilePart` attachments. Consistent with the proxy-host plan: expressif's proxy talks to the core out-of-band and needs nothing from the harness beyond tool execution.
- MCP resources and prompts are read (`resources()`, `readResource()`, `getPrompt()` in `mcp/index.ts`); users can attach MCP resources to messages as file parts (binary size-capped).
- An experimental "code mode" (MCP tools behind one `execute` catalog tool) exists but is flag-gated (`flags.experimentalCodeMode` in `tool/registry.ts`) — by default MCP tools are individual tools, descriptions in context. **No tool-search caveat here**, unlike Claude Code.

## Awareness injection points

Ranked for expressif's purposes:

1. **`system` field on every prompt** — per-message system-prompt injection with no file-system footprint; the cleanest hook of any harness surface mapped so far.
2. **Agent definitions** — markdown in `.opencode/agents/` / `~/.config/opencode/agents/` or the `agent` config object: `description`, `mode` (`primary`/`subagent`/`all`), `model`, `prompt` (or `{file:./path}`), `temperature`, `permission`, `tools`. The prompt body's `agent` field selects one per message. ([agents docs](https://opencode.ai/docs/agents/))
3. **Rules**: project `AGENTS.md` (walks up-tree), global `~/.config/opencode/AGENTS.md`, plus the `instructions` config array (globs and URLs); CLAUDE.md is honored as a compat fallback. ([rules docs](https://opencode.ai/docs/rules/))
4. MCP tool descriptions (always in context by default) and `noReply` context messages.

## Local models / offline

- Ollama, LM Studio, and llama.cpp are documented first-class via `@ai-sdk/openai-compatible` + `baseURL`; no API key needed. ([providers docs](https://opencode.ai/docs/providers/))
- **`@ai-sdk/openai-compatible` is bundled into the binary** (`BUNDLED_PROVIDERS` in `provider/provider.ts`), so the Ollama path triggers no runtime npm install. Only providers outside that list are fetched from npm on first use (`Npm.add`); `file://` package paths are supported for air-gapped custom providers.
- The models.dev catalog resolves disk cache → **compiled-in snapshot** → network, and `OPENCODE_DISABLE_MODELS_FETCH=1`, `OPENCODE_MODELS_PATH` (local file), `OPENCODE_MODELS_URL` (local mirror) give full offline control (`packages/core/src/models-dev.ts`).
- Residual first-run network: version/update checks and any non-bundled provider. With a bundled provider, models-fetch disabled, and auto-update off, the loop is offline — the docs' blanket "requires provider credentials" claim doesn't apply to local providers.
- Tool-calling quality on local models is the real risk the docs flag ("pick a model with strong tool-calling support") — expressif's widget flow leans entirely on tool calls, so the offline story is capability-bound, not plumbing-bound.

## Stability / versioning

- Rapid release cadence (v1.17.18 as of 2026-07-11; multiple releases weekly). No documented API stability guarantee; the OpenAPI spec is the de-facto contract and the SDK regenerates from it, so **pinning opencode server and SDK versions together** is the safe posture.
- The server is mid-migration internally (legacy routes alongside a new Effect `HttpApi` annotated "version 0.0.1"; an `event-v2-bridge` translating event schemas). Endpoint names have churned before (e.g. `prompt_async`, question routes are recent). The adapter should treat this surface as fast-moving and pin + smoke-test per release.
- Everything is MIT-licensed and vendorable — the asymmetry with the proprietary Claude Agent SDK noted in [Claude Code's surface](./claude-code-surface.md) holds, and reinforces shipping harness adapters as separate packages.

## Adapter-contract implications (feeds tickets 04, 05, 06, 07, 10)

- Transport is a **local HTTP server + SSE**, vs Claude Code's in-process SDK over a child process — the adapter contract must abstract "attached client over a wire" and "embedded SDK" equally (core/transport ticket).
- Normalized events: opencode natively provides the two-tier model (part updates + deltas) the Claude Code mapping proposed; granularity is compatible.
- Permission passthrough maps to `permission.updated` + respond endpoint, but **without input rewriting** — the adapter contract's permission surface must be the intersection (approve/reject + persist-rule) with input-rewriting as a capability flag.
- The question tool and `AskUserQuestion` are near-isomorphic (label/description options, multi-select, free-text) — one normalized "agent question" widget covers both harnesses.
- One opencode server hosts many sessions with shared config — relevant to the multi-agent/multi-session fog patch: the natural shape here is one server, N sessions, N `<agent-chat>` surfaces.
- Interrupt, fork, revert, compaction, and transcript read-back are all server-side — the core need not own transcript storage on this harness either.

## Sources

- Server mode: https://opencode.ai/docs/server/
- SDK: https://opencode.ai/docs/sdk/
- MCP: https://opencode.ai/docs/mcp-servers/
- Permissions: https://opencode.ai/docs/permissions/
- Agents: https://opencode.ai/docs/agents/
- Rules: https://opencode.ai/docs/rules/
- Providers/local models: https://opencode.ai/docs/providers/
- Source (github.com/anomalyco/opencode, `dev` branch, inspected 2026-07-11): `packages/sdk/js/src/gen/types.gen.ts`, `packages/opencode/src/{mcp/catalog.ts, session/processor.ts, session/prompt.ts, session/message-v2.ts, effect/runner.ts, tool/registry.ts, tool/question.ts, question/index.ts, provider/provider.ts, server/routes/instance/httpapi/groups/{session,question}.ts}`, `packages/schema/src/v1/question.ts`, `packages/core/src/models-dev.ts`
- Question-tool caveats: https://github.com/anomalyco/opencode/issues/10012, https://github.com/anomalyco/opencode/issues/30343
- npm metadata: `npm view @opencode-ai/sdk` / `opencode-ai` (2026-07-11)
