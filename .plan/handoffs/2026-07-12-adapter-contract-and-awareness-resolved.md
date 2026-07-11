# Handoff — 2026-07-12: adapter contract and agent awareness resolved

Session record for the **expressif v1 spec** wayfinder effort. The map is `.plan/expressif-v1-spec/map.md`; orient there first, and read `.pocock-skills/wayfinder/SKILL.md` + `TRACKER-MARKDOWN.md` for how to claim and resolve tickets.

## What this session did

- Claimed and resolved **[Define the harness adapter contract](../expressif-v1-spec/tickets/04-define-adapter-contract.md)** (commits `8728b8d`, `902c28f`) — eleven grilled decisions; the full contract is in the ticket's Answer.
- On the user's explicit instruction, went past wayfinder's one-ticket-per-session rule and also claimed and resolved **[Decide the agent-awareness mechanism](../expressif-v1-spec/tickets/05-decide-agent-awareness.md)** (commits `4d68939`, `4bd9f92`). That override was a one-off, not a precedent.
- Struck the two fog patches anchored to 04 (streaming/progressive rendering, conversation persistence) — both were answered by 04's resolution directly; no new tickets were needed.
- Added glossary terms to `/CONTEXT.md`: **Harness session**, **Part**, **Capability flag**, **Existence preamble**, and sharpened **Agent awareness** (its "mechanism to be decided" clause is now false and gone).

## State of the map

The frontier at the time of writing — derive the current one from the tickets, not from this record:

- **Pin return-channel semantics for structured inputs** (06) — takeable; now the natural next claim. 04 gave it its mechanical floor (send-with-queue guarantee, the normalized question schema, the answer-or-reject invariant); 06 owns the *policy*: blocking, supersede-vs-coexist when the user types instead, timeouts, and unifying native questions + elicitation into one widget model.
- **Decide transcript and degradation conventions** (07) — takeable; both 04 (harness-owned persistence, widget-state sidecar question) and 05 (plain-terminal degradation leans on canonical text forms) deferred obligations to it.
- **Investigate sandboxing in Tauri** (08) — takeable; the one research-type ticket on the frontier.
- Tickets 09–13 remain blocked; check their `blocked_by` against what is now resolved.

## Judgment calls the next session should know

- **04 and 05 deliberately drew a policy/mechanics line for 06**: the contract guarantees a send into a busy session queues and never errors, and that every surfaced question is answered or rejected (`close()` rejects pending). Ticket 06 decides what the *UX does* with those guarantees — nothing in 04 prejudges supersede-vs-coexist.
- **The capability descriptor is static per adapter instance** (~19 boolean flags, listed in 04's Answer). If 06 or 09 finds a capability that genuinely varies per session, that is a contract revision to raise explicitly, not something to bolt on quietly.
- **Awareness never touches project files** (05). If ticket 07's degradation story or a scenario-pack design ever wants durable file installs, it collides with a decided invariant — mark 05 undermined rather than quietly re-deciding.
- **Budget numbers in 05 are review-gate guidance, not protocol constants** — implementation may recalibrate them without reopening the ticket, but the posture (review gate, no runtime truncation) is the decision.
- Repo conventions: commit messages are `Claim:` / `Resolve:` / `Handoff:` + the ticket name; **no Claude co-author trailers** (history was force-pushed once to strip them).

## Suggested skills

(Only if available in the next agent's environment.)

- **wayfinder** (`.pocock-skills/wayfinder/`) — the effort's operating method; claim before working, one ticket per session.
- **grill-me** + **domain-modeling** (`.pocock-skills/`) — 06 and 07 are grilling type; keep `/CONTEXT.md` sharp as terms crystallise (Return channel is already defined there; 06 will stress it).
- **research** (`.pocock-skills/research/`) — for 08 (Tauri sandboxing), the remaining research ticket on the frontier.
