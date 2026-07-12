# Map: expressif v1 spec

## Destination

A buildable v1 spec for **expressif** — an embeddable agent-chat framework that turns a CLI/TUI coding-harness session into a rich GUI chat. The spec locks architecture (widget protocol, harness adapters, plugin model, embedding API), cuts scope, and is ready to hand to implementation sessions. Decisions, not code.

## Notes

Scope fixed during charting (2026-07-11):

- **Form**: a framework-agnostic web component (`<agent-chat>`) backed by a TypeScript core library. First consumer is iudex (Tauri, TS/SCSS frontend, Go/Rust backend).
- **Harnesses in v1**: Claude Code and opencode — a deliberately dissimilar pair to prove the adapter abstraction. Others come later.
- **Widget protocol**: working assumption is MCP Apps (`ui://` resources, sandboxed iframes, JSON-RPC bridge), to be validated before the spec locks it — see [Validate MCP Apps against expressif's requirements](./tickets/01-validate-mcp-apps.md).
- **Widget surface**: three primitives only — Markdown+ (code, Mermaid, math, images), sandboxed agent-authored HTML, and structured inputs with a return channel. Scenario packs (casino, D&D, finance) are skills + HTML on top, never framework widgets.
- **Plugins**: first-party only in v1; the plugin interface is specced docs-quality but marked unstable, designed to open to third parties later.
- **Hard constraints**: the framework and all plugins run fully offline — zero internet calls; only harness↔model inference leaves the machine. A plain text input is always present as fallback. Every widget must have a canonical text form (transcript, compaction, terminal degradation).
- **Standing preferences**: challenge loftiness; the user rejected a proof-of-concept destination, so prototype tickets only where a decision genuinely can't be made on paper, and cheap ones.
- **Skills**: tickets use `research`, `grill-me`, and `domain-modeling` from `.pocock-skills/` per their type. Glossary lives in `/CONTEXT.md` — keep it sharp as terms crystallise.

## Decisions so far

<!-- one line per resolved ticket: gist + link -->

- [Validate MCP Apps against expressif's requirements](./tickets/01-validate-mcp-apps.md) — GO: MCP Apps + MCP elicitation cover all hard requirements fully offline; neither harness is an Apps host, so the adapter compensates via the proxy-host pattern (one proxy MCP server facing the harness, expressif as the real Apps host behind it).
- [Map Claude Code's programmatic surface](./tickets/02-map-claude-code-surface.md) — Agent SDK is the surface: streaming-input mode mandatory, token deltas available, `canUseTool` is the permission/input passthrough, native `AskUserQuestion` (with HTML option previews) and `requiresUserInteraction` tools give two sanctioned input channels to unify with elicitation; tool search hides tool descriptions by default (awareness caveat); SDK is proprietary — peer-dependency only.
- [Map opencode's programmatic surface](./tickets/03-map-opencode-surface.md) — `opencode serve` HTTP+SSE is the surface (MIT SDK generated from its OpenAPI spec): token deltas and busy-session queueing native, permission respond has no input-rewriting (capability-flag it in the contract), the native `question` tool is near-isomorphic to `AskUserQuestion`, MCP results pass through JSON-encoded (proxy-host stands), per-message `system` field is the cleanest awareness hook yet, offline via bundled openai-compatible provider + models.dev snapshot; fast-moving API — pin server+SDK together.
- [Define the harness adapter contract](./tickets/04-define-adapter-contract.md) — two-level contract (`HarnessAdapter` + `HarnessSession`): required floor is the chat minimum (part-based two-tier event stream, send with queue guarantee, permission intersection, three-layer errors, proxy-only plugin surface with session attribution, harness-owned persistence with required read-back); all variance rides on a static descriptor of ~19 capability flags; adapters ship as separate packages (Claude Code SDK as peer dep, opencode SDK pinned exactly) that warn, not refuse, on version drift.
- [Decide the agent-awareness mechanism](./tickets/05-decide-agent-awareness.md) — three layers by content: a static ≤150-token existence preamble via system-prompt injection (CC preset-append; opencode per-message `system` field), capability discovery on the proxy's tool descriptions (opted out of CC tool search; the tool list is the single source of truth for enablement), usage guidance in scenario-pack skills paid per enabled pack; budgets are a first-party review gate; awareness is session-scoped and never written to project files, so plain-terminal reopens are clean by construction.
- [Pin return-channel semantics for structured inputs](./tickets/06-return-channel-semantics.md) — structured inputs block the agent turn (the resolution is the tool result), display primitives don't; typed input supersedes the active widget with the reply delivered as its answer (permission prompts excluded); multiple pending inputs queue FIFO behind one active card; no expressif timeout, explicit dismissal, `allowFreeText` + the fallback input as universal escape; harness tool timeouts are configured away and survived by downgrading the answer to a chat message; all sources (native questions, elicitation, `requiresUserInteraction` tools) unify into one input card with one lifecycle, frozen after resolution with the same rendering 07 replays.
- [Pin transcript representation and text degradation](./tickets/07-transcript-and-degradation.md) — canonical text is per-primitive (Markdown+ identity, structured inputs core-generated from schema+answer, HTML requires an agent-supplied summary as a mandatory tool argument); the record-of-consequence convention makes result text terse and self-contained so compaction survival is free; no widget-state sidecar — reopening replays frozen widgets from tool input+result in the harness transcript; on detach, pending widgets are rejected with a model-visible trace so nothing vanishes unexplained.
- [Validate sandboxing of agent HTML in a Tauri webview](./tickets/08-sandboxing-in-tauri.md) — HOLDS: the MCP Apps sandbox is airtight against Tauri's IPC by construction (post-CVE-2024-35222 invoke key + no init script in subframes), given four host MUSTs — never `allow-same-origin`, never app-origin widget content, no `remote` capability for widget origins, Tauri ≥ 2.0; delivery recipe is fully inlined HTML via `srcdoc`/`blob:` with meta-tag CSP (custom protocols rejected: per-platform origin shapes, WebView2 sandboxed-iframe quirk); Linux WebKitGTK is the engine-version floor; no prototype needed.

## Not yet specified

- **Multi-agent and multi-session shape.** iudex runs several agents at once — one `<agent-chat>` per session, or a multiplexed surface? Hangs on who owns the harness process. <clears-with: 10>
- **Theming depth.** Which CSS custom properties, slots, and parts the component commits to publicly. <clears-with: 09>

## Out of scope

- **Third-party plugin authoring, distribution, and registry** — v1 plugins are first-party; opening the API is a later effort.
- **codex and pi adapters** — v1 commits to Claude Code + opencode; further harnesses are post-v1 efforts.
- **Native/mobile SDK packaging beyond the web component** — webview embedding is assumed sufficient for v1 consumers.
- **Implementing the framework** — the destination is the spec; implementation is the next effort.
