---
type: grilling
blocked_by: [01, 06, 07]
---

# Spec the three primitives

## Question

Pin each primitive's v1 contract:

- **Markdown+**: exact feature set (code highlighting, Mermaid, math, images — what else? tables, callouts?), and how it renders while streaming.
- **Sandboxed HTML**: the contract an agent-authored HTML widget gets (viewport, size negotiation, theming variables passed in, lifecycle, what APIs are exposed over the bridge).
- **Structured inputs**: the v1 set (choices, slider, datepicker, form — anything else?), their schemas, and their canonical text forms per the transcript decision.

Each primitive's spec includes its degradation story and its agent-facing description (the prompt weight it costs).
