---
type: grilling
blocked_by: [01, 11]
---

# Define the plugin interface

## Question

What *is* an expressif plugin, concretely? Working assumption: an MCP (Apps) server plus a host-side manifest. To resolve: the manifest shape (name, version, tools, awareness text, enablement), how plugin sets are configured by the embedding app (minimal / scenario / full), how a plugin declares which primitive(s) it uses, packaging/loading (npm package? directory convention?), and how the interface is versioned and marked unstable while first-party. Everything must satisfy the offline constraint — no network at install-load-run time beyond what the consumer explicitly does.
