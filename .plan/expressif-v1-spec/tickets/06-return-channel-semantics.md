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
