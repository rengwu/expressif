---
type: grilling
blocked_by: [02, 03]
---

# Define the harness adapter contract

## Question

Given the mapped surfaces of Claude Code and opencode, what is the minimal adapter interface expressif's core defines — the contract any harness must be wrapped into?

To resolve: the normalized session model (spawn/attach/resume/interrupt), the normalized event stream (what the chat surface needs vs what each harness emits), sending user input, permission-prompt passthrough, MCP plugin registration, and error/exit semantics. Decide what is required vs optional capability (feature-detection for harnesses that lack e.g. thinking events). The contract must be honest about the two harnesses' variance, not a lowest common denominator that cripples both.

## Answer

Grilled 2026-07-12 against the surface assets of [Claude Code](../assets/claude-code-surface.md) and [opencode](../assets/opencode-surface.md) and the proxy-host pattern from [MCP Apps validation](../assets/mcp-apps-validation.md). The contract is a **required floor plus a static capability descriptor** — the floor is what a chat surface cannot function without; everything either harness has beyond it is a named capability flag, never silently dropped and never emulated with lying semantics.

### Shape

A **two-level contract**. `HarnessAdapter` (per harness installation) owns capability discovery, `listSessions`, and create/open. `HarnessSession` (per conversation) is the live handle: event stream, input, permissions, questions, interrupt, close. This maps opencode's one-server-N-sessions and Claude Code's one-process-per-query equally, and gives the multi-session question its seam without deciding process ownership (ticket 10's business).

### Persistence

**The harness owns transcript storage; the contract requires read-back** (`listSessions` + per-session history). The core keeps no durable message store — the chat surface rehydrates from the harness transcript, so what we replay is what the harness will actually resume with. Whether widget state needs a core-side sidecar for replay is ticket 07's decision; this contract only guarantees transcript read-back. *(Clears the map's conversation-persistence fog patch.)*

### Session lifecycle

Required is the chat minimum: `capabilities()`, `listSessions()`, `createSession()`, `openSession(id)` (resume; also the recovery path after disconnection) on the adapter; events, send, `interrupt()`, `close()`, `history()` on the session. `fork`, `delete`, `revert`, `compact`, `liveAttach` (join a session another client is driving), and `rename` are capability flags — both v1 harnesses have fork, but the required tier is "what a chat needs", not "what these two happen to share".

### Event stream

**Part-based with in-place updates, two granularity tiers.** Required: part lifecycle (`created`/`updated`/`completed`) within messages, message/turn boundaries, and session status (`idle`/`busy`/`error`); a conforming adapter may emit a part only on completion. Capability `textDeltas` adds token-level deltas per part (opencode natively; Claude Code via `includePartialMessages` mapping `content_block_*` onto part lifecycle). Tool parts carry the normalized four-state machine `pending → running → completed | error`. `reasoningParts` and `toolProgress` (mid-flight tool title/metadata updates — opencode only) are capability flags. *(Clears the map's streaming/progressive-rendering fog patch: progressive rendering is per-part everywhere, per-token where `textDeltas` holds — which is both v1 harnesses.)*

### Sending input

Required: `send(text)` with a **queue-while-busy guarantee** — sending into a busy session never errors (opencode queues server-side; Claude Code's mandated streaming-input mode queues too). This is the mechanical floor ticket 06's supersede policy builds on. Capabilities: `attachments`, `contextInjection` (append context without triggering a reply — opencode `noReply`, Claude Code streaming injection; ticket 05's likely lever), `perMessageOverrides` (model/agent/system per message, exposed as an opaque harness-specific bag — not normalized fields, so the contract doesn't pretend Claude Code can switch agents per message).

### Permissions

Required: `permission.requested` event (id, tool, input, title/metadata) + `respond(approveOnce | approveAlways | deny)` — the honest intersection. Three capability flags carry the variance: `permissionInputRewrite` (approve with edited input — Claude Code `updatedInput`; opencode lacks it, per the surface asset's recommendation, confirmed here), `denyMessage` (deny with a model-visible reason — Claude Code only), `deferredApproval` (approval survives process exit — Claude Code `defer`). The union-with-emulation alternative was rejected: a deny-plus-resend is not a rewritten call, and emulation that lies breaks trust in the contract.

### Agent questions

The native question tools (`AskUserQuestion`, opencode `question`) are near-isomorphic, so the contract **normalizes one question schema** — `question.asked` carrying `questions[{text, header, options[{label, description}], multiSelect, allowFreeText}]`, answered via `respond(answers)` or `reject()` — but gates availability behind capability `nativeQuestions` (+ `questionHtmlPreviews`, Claude Code only), since a future harness without a native tool still gets question widgets via proxy elicitation (the widget-level unification is tickets 06/11). **Hard invariant: an adapter resolves every question request it surfaces; `close()` rejects all pending questions** — this turns opencode's known headless-hang failure mode into a can't-happen.

### MCP plugin registration

**The contract's plugin surface is the proxy, and only the proxy.** Required: the expressif proxy is attached and reachable from every session the adapter opens; proxy connection status is reported; and widget traffic arriving at the proxy is **attributable to the originating session** (how — per-session in-process server on Claude Code, shared-config correlation on opencode — is adapter-internal, mechanics with ticket 10). Capability `perSessionPluginEnablement` (Claude Code native; opencode genuinely emulable via per-prompt tool toggles). No generic register-any-MCP-server passthrough: plugin registration lives behind the proxy in the core (ticket 12), and exposing raw wiring would leak per-session-vs-global harness variance for a feature expressif doesn't need.

### Errors and exit

**Three required layers**, so "agent failed" is never conflated with "lost the wire": (1) every turn ends `completed | interrupted | error`, turn errors leave the session usable; (2) session status `error` means the harness reports the session wedged, history still readable; (3) the handle emits `disconnected` on transport/process loss, and recovery is `openSession(id)` — guaranteed to rehydrate by the persistence decision. Invariants: `close()` is graceful (rejects pending questions/permissions, then releases); no error path loses the transcript. `usageMetrics` (cost/tokens) is a capability flag.

### Capability model

A **static descriptor**: `capabilities()` returns the flag set once, fixed per adapter instance (adapter package + harness version); the chat surface reads it at mount and shapes affordances. No per-session renegotiation. The eighteen flags: `fork`, `delete`, `revert`, `compact`, `liveAttach`, `rename`, `textDeltas`, `reasoningParts`, `toolProgress`, `attachments`, `contextInjection`, `perMessageOverrides`, `permissionInputRewrite`, `denyMessage`, `deferredApproval`, `nativeQuestions`, `questionHtmlPreviews`, `perSessionPluginEnablement`, plus `usageMetrics` — nineteen with the last, all boolean.

### Packaging and versioning

**Harness-free core; one package per adapter.** The core defines contract types and depends on no harness SDK. The Claude Code adapter declares the proprietary `@anthropic-ai/claude-agent-sdk` as a peer dependency (licensing stays between consumer and Anthropic); the opencode adapter pins `@opencode-ai/sdk` exactly and declares the opencode server range it was smoke-tested against, revalidated per release. The contract requires **version self-reporting**: adapters expose harness identity/version and whether the attached harness is outside the tested range — outside-range warns but runs (refusing would brick users on opencode's weekly cadence).
