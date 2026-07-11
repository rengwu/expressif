---
type: grilling
blocked_by: [01, 02, 03]
---

# Decide the agent-awareness mechanism

## Question

How does the agent know it is talking to an expressif surface and what it can render? Candidates, not mutually exclusive: MCP tool descriptions (awareness rides on the tools themselves), system-prompt injection via each harness's injection point, and skill files that teach usage patterns per scenario.

To resolve: which channel carries what (capability discovery vs usage guidance vs scenario packs), how per-plugin enablement changes what the agent sees, prompt-weight budget, and what happens when the same session is later opened in a plain terminal. Produce the decision plus the injection recipe per supported harness.

Constraint from [Map Claude Code's programmatic surface](02-map-claude-code-surface.md): Claude Code's MCP tool search withholds tool definitions from context by default — awareness cannot rely on tool descriptions alone on that harness; decide whether to opt expressif's server out of tool search or carry discovery in the system prompt.

## Answer

Grilled 2026-07-12. Awareness is **three layers, split by what the content is**, each on the only channel that can carry it well — and all of it strictly session-scoped.

### The three layers

1. **Existence + invariants → system-prompt injection.** A static preamble (≤150 tokens): an expressif surface is attached; the three primitives exist; every widget carries its canonical text form; the expressif tool prefix. Prompt-injected because it must be present before any tool is relevant — it is what makes the agent reach for a widget at all. It never enumerates plugins and never varies per session, so it is cache-stable by construction.
2. **Capability discovery → the proxy's MCP tool descriptions.** Which renderer tools exist right now, their schemas, and when to prefer them over plain text (≤~100 tokens per tool; the default three-primitive surface under ~500 total). Tools are the one channel that self-updates with plugin enablement.
3. **Usage guidance → scenario-pack skills.** Deep composition guidance (≤~2,000 tokens per pack) loads only when its pack is enabled — weight is paid only when relevant.

The rejected extremes: everything-in-prompt duplicates tool schemas and regenerates on every toggle; everything-on-tools is defeated by Claude Code's tool search and gives no existence signal.

### Tool search on Claude Code

**Opt expressif's proxy out of tool search** — its tools are always in context, matching opencode's default, so discovery behaves identically on both harnesses. Tool search exists to hide hundreds of tools; expressif exposes a half-dozen always-relevant ones by design (scenario packs add skills and HTML, never framework tools). Belt-and-suspenders: the preamble names the tool prefix, so a harness that hides tools anyway still leaves the model knowing what to search for.

### Enablement propagation

**The enabled-tool list is the single source of truth.** Toggling a plugin changes exactly one thing — which tools the proxy exposes — so prompt-vs-reality drift (the prompt promising a datepicker the deployment disabled) is structurally impossible. Mid-session toggles (contract capability `perSessionPluginEnablement` from [Define the harness adapter contract](04-define-adapter-contract.md)) get a one-line nudge via the contract's `contextInjection` capability, since the model won't spontaneously re-read a tool list; propagation mechanics are proxy/adapter plumbing (tickets 10/12).

### Budget posture

Ceilings are a **first-party review gate, not a runtime mechanism** — nothing truncates at runtime; a plugin that cannot fit its description in budget fails review. Numbers above are spec guidance to validate in implementation, sized for the offline/local-model story where context is scarce.

### Plain-terminal behavior

**Awareness is session-scoped and non-durable: expressif never writes to files a terminal session would read** (no CLAUDE.md, no AGENTS.md, no project skill dirs). A session reopened in a plain terminal is then clean by construction — no preamble, no proxy tools, no pack skills; past widget turns read as their canonical text forms (ticket 07's obligation). The inverse holds too: awareness is freshly injected each expressif session, never assumed from history. Accepted edge: a user who attaches their own opencode TUI to *expressif's* server instance sees expressif's config — attaching to expressif's server is opting into its world. The durable-install alternative (with guarded text, or cleanup-on-detach) was rejected: it pollutes the user's repo, and the error model can't guarantee cleanup.

### Injection recipe per harness

**Claude Code**: preamble via `systemPrompt: {type: "preset", preset: "claude_code", append: …}` with `excludeDynamicSections: true`; discovery via the in-process proxy MCP server in `mcpServers`, opted out of tool search, renderer tools auto-allowed by server-scoped `allowedTools` wildcard where deployment policy agrees; packs via SDK-scoped skills through controlled `settingSources` — nothing written to the project.

**opencode**: preamble on the **per-message `system` field** of every prompt expressif sends (no filesystem footprint, perfectly session-scoped; incidentally keeps the preamble recent in context for weaker local models); discovery via the proxy registered in the config of the server instance expressif owns (tools always in context — no tool-search issue); mid-session toggle nudges via `noReply` context messages. **Agent definitions deliberately stay free** as a scenario-pack vehicle — carrying the preamble there would couple awareness to agent selection (model, temperature, permissions), which belongs to the user and to packs.
