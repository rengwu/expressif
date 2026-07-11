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

## Not yet specified

- **Streaming and progressive rendering.** How text and widgets render mid-stream, given each harness's event granularity — can't sharpen until the harness surfaces are mapped. <clears-with: 04>
- **Multi-agent and multi-session shape.** iudex runs several agents at once — one `<agent-chat>` per session, or a multiplexed surface? Hangs on who owns the harness process. <clears-with: 10>
- **Conversation persistence.** Whether transcript storage and replay belong to the harness, the core, or the host app. <clears-with: 04>
- **Theming depth.** Which CSS custom properties, slots, and parts the component commits to publicly. <clears-with: 09>

## Out of scope

- **Third-party plugin authoring, distribution, and registry** — v1 plugins are first-party; opening the API is a later effort.
- **codex and pi adapters** — v1 commits to Claude Code + opencode; further harnesses are post-v1 efforts.
- **Native/mobile SDK packaging beyond the web component** — webview embedding is assumed sufficient for v1 consumers.
- **Implementing the framework** — the destination is the spec; implementation is the next effort.
