# CONTEXT

Glossary of the expressif domain. Terms only — no implementation detail.

## expressif

The framework itself: an embeddable agent-chat surface that turns a text-only coding-harness session into a rich GUI chat, letting agents express themselves beyond text.

## Harness

A CLI/TUI coding agent runtime that expressif wraps — e.g. Claude Code, opencode, codex, pi. The harness owns the agent loop and model access; expressif never talks to a model directly.

## Harness adapter

The layer that normalizes one harness's programmatic surface (sessions, events, input, permissions) into expressif's common contract.

## Chat surface

The visible chat panel expressif provides — messages, widgets, and the fallback input. The thing a consuming app embeds.

## Fallback input

The plain text box that is always present in the chat surface, regardless of what widgets are pending. The floor the experience can never drop below.

## Primitive

One of the three rendering capabilities the framework itself defines: **Markdown+** (rich markdown: code, Mermaid, math, images), **sandboxed HTML** (agent-authored HTML in an isolated frame), and **structured inputs** (widgets that collect a user response and return it to the agent).

## Widget

A single rendered instance of a primitive inside the chat — one choice prompt, one HTML preview, one diagram.

## Return channel

The path by which a user's interaction with a structured input travels back to the agent as data (rather than the user echoing it as text).

## Plugin

A unit of extension that exposes rendering capability to the agent (working assumption: an MCP Apps server plus a host-side manifest). Plugins are enabled per deployment and must work fully offline. First-party only in v1.

## Scenario pack

A skill-plus-plugin bundle tuned to a use case (ideation grilling, D&D, finance). Scenario packs compose primitives; they are not new framework widgets.

## Agent awareness

Whatever makes the agent know an expressif surface is attached and how to use its capabilities — tool descriptions, prompt injection, and/or skills. Mechanism to be decided.

## Host (embedding app)

The application that embeds the chat surface — iudex first. Distinct from the MCP sense of "host"; in expressif docs, "host" means the embedding app unless qualified.
