---
type: research
blocked_by: []
assets: [.plan/expressif-v1-spec/assets/mcp-apps-validation.md]
---

# Validate MCP Apps against expressif's requirements

## Question

The map's working assumption is that MCP Apps (the official MCP UI extension, spec 2026-01-26) is expressif's widget protocol. Pressure-test that against our hard requirements before the spec locks it:

- **Offline**: are `ui://` resources fully deliverable over local transports (stdio) with no network fetches implied anywhere in the spec? What does the sandbox CSP model require or forbid?
- **Blocking structured inputs**: does the tool-call + JSON-RPC bridge support a widget that blocks awaiting user input and returns the selection as the tool result (AskUserQuestion-style)? Is MCP elicitation relevant here?
- **Host obligations**: enumerate exactly what a conformant host must implement (resource fetching, iframe lifecycle, postMessage JSON-RPC, theming hooks, state persistence status).
- **Transcript/model-facing representation**: what does the spec say the model sees after a widget renders and the user interacts?
- **Harness reality**: do Claude Code and opencode already pass `ui://` resources and `_meta` through to a consuming client, or does the adapter need to compensate?
- **Spec churn risk**: what is still in flux (SEP-1865 open PRs — external URLs, state persistence, view-to-view)?

Primary sources: the ext-apps spec repo, MCP blog, working-group discussions. Deliver a cited summary asset with a go/no-go recommendation and, if go, the list of spec gaps expressif must fill itself.

## Answer

**GO — adopt MCP Apps, with expressif's adapter acting as the Apps host.** Full findings in [the validation asset](../assets/mcp-apps-validation.md).

- **Offline holds outright**: `ui://` resources travel inline over stdio; the default CSP is `default-src 'none'`, external domains are opt-in metadata the host enforces — expressif enforces offline by refusing them.
- **Blocking inputs**: the Apps spec has no native pause, but **MCP elicitation** is purpose-built for it — blocking by design, and its restricted schema (enum, date/date-time, bounded number, flat object) maps one-to-one onto the structured-input primitive. New constraint: blocked calls must survive harness tool timeouts.
- **The gap**: neither Claude Code (CLI/SDK) nor opencode advertises the `ui` extension or elicitation — neither harness is an Apps host. **Compensation: the proxy-host pattern** — the adapter registers a single proxy MCP server with the harness, forwards plain tool traffic to plugins, and is itself the fully-capable Apps host + elicitation client, routing UI to the chat surface. The harness only needs baseline MCP tools, which both clear.
- **Model-facing channels exist** (result `content`, `ui/message`, `ui/update-model-context`); the conventions are ours to set in the transcript ticket.
- **Churn risk low** (SEP-1865 Final, Anthropic+OpenAI co-maintained, four shipped clients). Deferred spec items (state persistence, external URLs, view-to-view) don't block v1; missing state persistence means our own replay story.

Gaps expressif fills: the proxy-host adapter, supersede/timeout UX policy, canonical text forms, offline CSP enforcement.
