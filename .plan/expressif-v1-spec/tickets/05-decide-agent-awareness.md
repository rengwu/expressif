---
type: grilling
blocked_by: [01, 02, 03]
---

# Decide the agent-awareness mechanism

## Question

How does the agent know it is talking to an expressif surface and what it can render? Candidates, not mutually exclusive: MCP tool descriptions (awareness rides on the tools themselves), system-prompt injection via each harness's injection point, and skill files that teach usage patterns per scenario.

To resolve: which channel carries what (capability discovery vs usage guidance vs scenario packs), how per-plugin enablement changes what the agent sees, prompt-weight budget, and what happens when the same session is later opened in a plain terminal. Produce the decision plus the injection recipe per supported harness.

Constraint from [Map Claude Code's programmatic surface](02-map-claude-code-surface.md): Claude Code's MCP tool search withholds tool definitions from context by default — awareness cannot rely on tool descriptions alone on that harness; decide whether to opt expressif's server out of tool search or carry discovery in the system prompt.
