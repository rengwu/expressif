# CONTEXT

Glossary of the expressif domain. Terms only — no implementation detail.

## expressif

The framework itself: an embeddable agent-chat surface that turns a text-only coding-harness session into a rich GUI chat, letting agents express themselves beyond text.

## Harness

A CLI/TUI coding agent runtime that expressif wraps — e.g. Claude Code, opencode, codex, pi. The harness owns the agent loop and model access; expressif never talks to a model directly.

## Harness adapter

The layer that normalizes one harness's programmatic surface (sessions, events, input, permissions) into expressif's common contract.

## Harness session

One conversation with an agent, as normalized by a harness adapter — the live handle the chat surface drives: its event stream, input, interrupt, and history. The harness owns the durable transcript behind it.

## Part

The unit of content within a message on the normalized event stream — a run of text, a reasoning block, a tool call, a file. Parts appear and update in place as the agent works; a tool part carries its lifecycle state (pending, running, completed, error).

## Capability flag

A named feature a harness adapter declares it supports beyond the required contract floor (e.g. text deltas, input rewriting on approval, fork). The chat surface reads the flags once and shapes its affordances; absent capabilities degrade visibly, never silently.

## Chat surface

The visible chat panel expressif provides — messages, widgets, and the fallback input. The thing a consuming app embeds.

## Fallback input

The plain text box that is always present in the chat surface, regardless of what widgets are pending. The floor the experience can never drop below.

## Primitive

One of the three rendering capabilities the framework itself defines: **Markdown+** (rich markdown: code, Mermaid, math, images), **sandboxed HTML** (agent-authored HTML in an isolated frame), and **structured inputs** (widgets that collect a user response and return it to the agent).

## Widget

A single rendered instance of a primitive inside the chat — one choice prompt, one HTML preview, one diagram.

## Canonical text form

The plain-text record a widget leaves in the conversation — the outcome, not the UI ("record of consequence"): what was asked, what was chosen. Self-contained by rule, so it survives compaction and reads coherently in a plain terminal. Generated per primitive: markdown is its own form, structured-input forms are derived from schema and answer, agent-authored HTML must bring its own summary.

## Return channel

The path by which a user's interaction with a structured input travels back to the agent as data (rather than the user echoing it as text). Degradable: if the harness abandons the blocked call, the answer falls back to a plain chat message carrying the canonical text.

## Input card

The unified widget every structured-input source (native question tools, plugin elicitation, interaction-marked tools) renders as. One lifecycle regardless of source: pending, then answered, superseded (user typed instead), dismissed, downgraded (harness gave up waiting), or detached. One card is active at a time; the rest queue. Frozen with its resolution shown once resolved.

## Plugin

A unit of extension that exposes rendering capability to the agent (working assumption: an MCP Apps server plus a host-side manifest). Plugins are enabled per deployment and must work fully offline. First-party only in v1.

## Scenario pack

A skill-plus-plugin bundle tuned to a use case (ideation grilling, D&D, finance). Scenario packs compose primitives; they are not new framework widgets.

## Agent awareness

What makes the agent know an expressif surface is attached and how to use its capabilities. Three layers: the existence preamble (prompt-injected invariants), capability discovery (the enabled tools themselves), and usage guidance (scenario-pack skills). Session-scoped — never written into the user's project.

## Existence preamble

The short, static system-prompt block that tells the agent an expressif surface is attached, names the three primitives and the canonical-text obligation, and points at the expressif tools. It never enumerates plugins — which tools exist is the tool list's truth, not the preamble's.

## Host (embedding app)

The application that embeds the chat surface — iudex first. Distinct from the MCP sense of "host"; in expressif docs, "host" means the embedding app unless qualified.
