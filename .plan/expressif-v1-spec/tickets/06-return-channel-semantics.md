---
type: grilling
blocked_by: [01]
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

## Answer

Grilled 2026-07-12, on the mechanical floor laid by [Define the harness adapter contract](04-define-adapter-contract.md) (queue guarantee, normalized question schema, answer-or-reject invariant) and the text conventions of [Pin transcript representation and text degradation](07-transcript-and-degradation.md).

### Blocking

**Structured inputs block the agent turn; display primitives don't.** An input render suspends the turn and its resolution *is* the tool result / elicitation response, carrying the record-of-consequence text — the shape every underlying channel (elicitation, `AskUserQuestion`, opencode `question`) natively has. Markdown+ and HTML views return immediately; later HTML interaction reaches the model via 07's coalesced delta path, never by holding a turn open. Fire-and-forget input collection is representable as an HTML view + deltas, not a structured input.

### Typed input supersedes

The user typing in the fallback input while an input widget pends **supersedes the active widget**: its tool result returns immediately with "user replied in free text instead — their reply follows", the typed message is delivered right behind it, and the widget freezes as *answered in chat*. Coexist/queue was rejected (the agent stays deaf to what the user just said — the maddening case); cancel was rejected (the model sees a bare rejection and re-asks what was just answered). **Permission prompts are excluded**: a typed message never approves or denies a tool call — permission requests stay pending and typed input queues behind them per 04. A multi-question card is superseded whole.

### Multiple pending widgets

**Allowed to exist, presented serially.** Parallel tool calls and concurrent elicitations make simultaneous pending inputs possible at the protocol level, and forbidding them would mean auto-rejecting questions the user never saw. The chat surface shows one active card at a time; the rest queue FIFO, visibly. Supersede-by-typing hits only the active card — typing answers what's in front of you.

### Lifecycle: no timeout, dismissal, escape, downgrade

- **No expressif-imposed timeout** — widgets pend until answered, superseded, dismissed, or detached, matching both harnesses' native questions (which never time out).
- **Explicit dismissal** (✕) on every widget = decline with the canonical trace "user dismissed without answering". Dismissal says *no answer*; supersede says *answered differently*.
- **Free-text escape**: `allowFreeText` in the normalized schema renders an "Other…" field on the card; even where a plugin disables it, the fallback-input supersede path remains the universal escape — no widget can trap the user, by construction.
- **Harness tool timeouts**: adapters configure the harness's MCP/tool timeout for the proxy to maximum/disabled (Claude Code `MCP_TIMEOUT`, opencode per-server `timeout`). If one fires anyway, the tool call dies but **the widget survives in *downgraded* mode**: the user's eventual answer travels the fallback path as a normal chat message carrying the canonical outcome text. The return channel degrades from tool-result to chat-message; user input is never lost to an infrastructure event. This is the recovery UX [Validate MCP Apps](01-validate-mcp-apps.md) demanded.

### One input-widget model

Native question tools, plugin elicitation (asset 01's schema mapping: enum → choices, date-formatted string → datepicker, bounded number → slider, flat object → form), and Claude Code's `requiresUserInteraction`-marked tools all map at the adapter/proxy boundary into **one normalized input card with one state machine: `pending → answered | superseded | dismissed | downgraded | detached`**. Every rule above applies identically regardless of source; the user can never tell which protocol channel a card rode in on. Post-answer, the card **freezes inline with its resolution shown** (selection highlighted, reply quoted, or terminal state labeled), and that frozen rendering is the same one 07's rehydration replays from tool input + result — live history and reopened history share one renderer.
