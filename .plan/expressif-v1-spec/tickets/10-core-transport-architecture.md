---
type: grilling
blocked_by: [04]
---

# Decide the core/transport architecture

## Question

A webview cannot spawn processes, but the harness and the MCP plugins are local processes. Who owns them, and how does the TS core reach them?

To resolve: where the adapter runs (host app's backend? a bundled sidecar the framework ships? consumer's choice?), the transport between the `<agent-chat>` core and that process layer (websocket on localhost, Tauri IPC, pluggable bridge interface?), and whether the framework ships a reference sidecar or only the bridge contract. Must work for iudex (Tauri + Go) without precluding a plain browser tab talking to a local daemon.
