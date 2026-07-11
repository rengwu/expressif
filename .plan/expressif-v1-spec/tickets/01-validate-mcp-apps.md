---
type: research
blocked_by: []
claimed_by: claude-fable-5-session-847e1e98
claimed_at: 2026-07-11T16:10:00+08:00
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
