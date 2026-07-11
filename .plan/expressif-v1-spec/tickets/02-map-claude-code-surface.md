---
type: research
blocked_by: []
claimed_by: claude-fable-5-session-847e1e98
claimed_at: 2026-07-11T16:45:00+08:00
---

# Map Claude Code's programmatic surface

## Question

For the harness adapter contract, map how a host application drives Claude Code programmatically (Claude Agent SDK and/or headless CLI modes):

- Session lifecycle: spawn, resume, interrupt, kill.
- Event stream: message/token streaming granularity, tool-use events, thinking, errors.
- Sending user input mid-session; answering permission prompts programmatically.
- Wiring MCP servers into a session (config, scopes, stdio) — can expressif's renderer plugins be attached per-session?
- System prompt / context injection points (system prompt append, CLAUDE.md, skills) — candidate channels for agent awareness.
- How structured tools like AskUserQuestion surface to an SDK consumer.
- Licensing/redistribution constraints on embedding it as a dependency of a framework.

Deliver a cited summary asset. This feeds [Define the harness adapter contract](04-define-adapter-contract.md).
