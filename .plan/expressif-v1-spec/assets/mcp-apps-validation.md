# MCP Apps validation against expressif's requirements

Research asset for ticket [Validate MCP Apps against expressif's requirements](../tickets/01-validate-mcp-apps.md). Investigated 2026-07-11 against primary sources: the MCP Apps spec (2026-01-26, SEP-1865), the MCP elicitation spec (2025-06-18), the MCP blog, Claude Agent SDK docs, and opencode docs.

## Verdict: GO — adopt MCP Apps, with the adapter acting as the Apps host

MCP Apps satisfies every hard requirement, but neither v1 harness implements the host side of it. expressif's adapter layer must *be* the MCP Apps host (and elicitation client) itself — the "proxy-host" pattern described below. This is compensable by design and does not require forking or extending the protocol.

## Findings by requirement

### 1. Offline — satisfied outright

- UI resources are ordinary MCP resources fetched via `resources/read` over any transport, including stdio; content is inline HTML (`text` field) or base64 `blob`, MIME `text/html;profile=mcp-app`. Nothing in delivery implies a network. ([spec §resources](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx))
- Default CSP when `ui.csp` is omitted is `default-src 'none'; script-src 'self' 'unsafe-inline'…` — no external connections at all. External access is opt-in via `connectDomains` / `resourceDomains` / `frameDomains` metadata, which the **host constructs and enforces**. expressif can therefore enforce its offline constraint mechanically: the host refuses or strips any plugin CSP that requests external domains. ([spec §csp](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx))
- External-URL embedding (`text/uri-list`) is explicitly deferred from the MVP spec — the current standard is local-content-only by construction.

### 2. Blocking structured inputs — satisfied via elicitation; Apps has no native pause

- The Apps spec itself has **no protocol-level mechanism to pause a tool call awaiting user input**; a widget can reach back via `tools/call` / `ui/message`, so a server *can* hold its original tool result open until the view reports interaction, but that is a server-side pattern, not a spec feature.
- **MCP elicitation is the purpose-built fit**: a server sends `elicitation/create` mid-request, the client renders input UI, and the server blocks until `accept`/`decline`/`cancel` returns. Its restricted schema maps almost one-to-one onto our structured-input primitive: `enum` + `enumNames` → choice buttons; `string` with `format: date`/`date-time` → datepicker; `number` with `minimum`/`maximum` → slider; flat objects → forms. ([elicitation spec](https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation))
- Elicitation is a **client capability** that must be advertised at initialization — see finding 4 for who the client is.
- **New constraint surfaced**: a blocked tool call must outlast the harness's tool-execution timeout (Claude Code: `MCP_TIMEOUT`/tool timeout env vars). Return-channel design must treat harness timeouts as a first-class failure mode.

### 3. Host obligations — enumerable and bounded

A conformant host must: advertise the `io.modelcontextprotocol/ui` extension at `initialize`; fetch `ui://` resources via `resources/read`; construct and enforce CSP from `_meta.ui.csp`; render views in sandboxed iframes; proxy JSON-RPC between view and server over `postMessage` (views act as MCP clients); handle view-initiated requests (`ui/message`, `ui/update-model-context`, `ui/open-link`, `ui/request-display-mode`); emit lifecycle notifications (`ui/notifications/tool-input`, `tool-result`, `ui/resource-teardown`); and, for web hosts, implement the intermediate sandbox-proxy iframe. This is the checklist ticket 13's conformance section inherits.

### 4. Harness reality — the gap, and the compensating pattern

- **Claude Code (CLI/Agent SDK)**: MCP servers attach per session (`mcpServers` option or `.mcp.json`); the SDK stream exposes `tool_use` blocks and results, and MCP tools need `allowedTools` grants. But nothing in the SDK surface advertises the `ui` extension or exposes elicitation to the consumer — the terminal client is not an Apps host, and MCP Apps' supported-clients list (Claude web/desktop, ChatGPT, Goose, VS Code) does not include Claude Code. ([Agent SDK MCP docs](https://code.claude.com/docs/en/agent-sdk/mcp), [MCP Apps blog](https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/))
- **opencode**: supports local (stdio) and remote MCP servers; docs are silent on resources, elicitation, and `ui://` handling — ticket 03 must check the source. ([opencode MCP docs](https://opencode.ai/docs/mcp-servers/))
- **Compensating pattern — the proxy-host**: expressif's adapter registers *one* MCP server with the harness: a proxy. The proxy forwards plain `tools/*` traffic between harness and plugin servers, while itself connecting to plugins as a fully capable client — advertising the `ui` extension *and* elicitation, fetching `ui://` resources, and routing view/elicitation UI to the expressif chat surface. The harness never needs to know Apps exists; it just sees tools (some of which are slow because a human is clicking). Consequences:
  - Works with harnesses that support nothing beyond baseline MCP tools — the lowest bar both harnesses clear.
  - Unmodified third-party MCP Apps servers can work through the proxy in the future, even though v1 plugins are first-party.
  - The proxy is where the Apps host obligations (finding 3) physically live. Tickets **Define the harness adapter contract** and **Decide the core/transport architecture** should treat the proxy-host as the leading candidate architecture.

### 5. Model-facing representation — defined, with our policy to fill in

The model sees: tool input, the tool result `content` array, content a view sends via `ui/message`, and context posted via `ui/update-model-context` (applied to future turns at host discretion). The model does **not** see `structuredContent` (UI-only) or raw in-widget interactions. So the transcript question is answerable within the protocol: every plugin tool must put the canonical text form in result `content`, and `ui/update-model-context` covers state that changes after the result. Ticket 07 decides the conventions.

### 6. Spec churn — Final status, bounded risk

SEP-1865 is Final; MCP Apps is co-maintained by Anthropic and OpenAI core maintainers and shipped in four major clients — abandonment risk is low. Explicitly deferred (may land later, none blocking us): external URL content, state persistence/restore, custom sandbox policies, view-to-view communication, screenshot/preview APIs, non-HTML content types. The absence of state persistence means expressif must define its own widget-state story for transcript replay (feeds ticket 07).

## Gaps expressif must fill itself

1. The proxy-host adapter (finding 4) — the biggest piece of expressif-specific engineering.
2. Blocking-input UX policy: supersede/cancel semantics when the user types instead of clicking, and harness-timeout survival (ticket 06).
3. Canonical text forms and replay/degradation conventions (ticket 07) — the spec gives channels, not policy.
4. Offline enforcement policy: reject or strip plugin CSPs requesting external domains.

## Sources

- MCP Apps spec 2026-01-26: https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx
- MCP Apps announcement: https://blog.modelcontextprotocol.io/posts/2026-01-26-mcp-apps/
- MCP elicitation spec 2025-06-18: https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation
- Claude Agent SDK — MCP: https://code.claude.com/docs/en/agent-sdk/mcp
- Claude MCP Apps getting started: https://claude.com/docs/connectors/building/mcp-apps/getting-started
- opencode MCP docs: https://opencode.ai/docs/mcp-servers/
