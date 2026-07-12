---
type: grilling
blocked_by: [01]
claimed_by: claude-fable-5-session-163b94bf
claimed_at: 2026-07-12T13:35:00+08:00
---

# Pin return-channel semantics for structured inputs

## Question

Structured inputs (choices, slider, datepicker, form) block the agent awaiting a user response. Pin the interaction model:

- Does a render call block the agent turn until interaction, and how is that expressed in the protocol (tool result? elicitation?).
- The user ignores the widget and types in the fallback text box instead — does the typed message cancel, supersede, or coexist with the pending widget? (Get this wrong and the UX is maddening.)
- Multiple pending widgets at once: allowed or forbidden?
- Timeouts, dismissal, and "Other/free-text" escape on choice widgets.
- What the widget looks like after answering (frozen with the selection shown?).
- Surviving harness tool-execution timeouts (surfaced by [Validate MCP Apps against expressif's requirements](01-validate-mcp-apps.md)): a blocked elicitation/tool call is a "slow tool" to the harness — decide timeout configuration and the recovery UX when the harness gives up before the user answers.
- Unifying channels (surfaced by [Map Claude Code's programmatic surface](02-map-claude-code-surface.md)): Claude Code offers native `AskUserQuestion` (via `canUseTool`, with optional HTML option previews) and `requiresUserInteraction`-marked MCP tools alongside plugin elicitation — the chat surface must present all of these as one widget model with one set of supersede/timeout rules.
