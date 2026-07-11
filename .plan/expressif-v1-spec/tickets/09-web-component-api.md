---
type: grilling
blocked_by: [10]
---

# Design the `<agent-chat>` web component API

## Question

The public embedding surface: attributes/properties the host app sets (session handle, plugin set, theme), events it receives (message, widget interaction, session state, errors), slots/parts it can style or replace, and how the host supplies the transport decided in [Decide the core/transport architecture](10-core-transport-architecture.md). Decide what is public-stable vs internal in v1, and what the minimal "hello world" embedding looks like for a consumer that is not iudex.
