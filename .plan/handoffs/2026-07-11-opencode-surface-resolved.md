# Handoff — 2026-07-11: opencode surface mapped

Session record for the **expressif v1 spec** wayfinder effort. The map is `.plan/expressif-v1-spec/map.md`; orient there first, and read `.pocock-skills/wayfinder/SKILL.md` + `TRACKER-MARKDOWN.md` for how to claim and resolve tickets.

## What this session did

- Claimed and resolved **[Map opencode's programmatic surface](../expressif-v1-spec/tickets/03-map-opencode-surface.md)** (commits `88fbe82`, `da60d4f`). The full findings are in the ticket's Answer and the cited asset [opencode-surface.md](../expressif-v1-spec/assets/opencode-surface.md) — don't re-derive them from here.
- Rewrote history to strip `Co-Authored-By: Claude` trailers from this session's two commits and force-pushed `main` (old `cb905a3` → `da60d4f`). **Repo convention going forward: no Claude co-author trailers on commits.**

## State of the map

Both harness surfaces (Claude Code, opencode) are now mapped, which was the long pole. The frontier at the time of writing — derive the current one from the tickets, not from this record:

- **Define the harness adapter contract** (04) — unblocked by this session; the natural next claim. Both surface assets end with an "adapter-contract implications" section written to feed it.
- **Decide the agent-awareness mechanism** (05) — also newly unblocked.
- **Return-channel semantics** (06), **Transcript & degradation** (07), **Sandboxing in Tauri** (08) — were already takeable.

## Judgment calls the next session should know

- The two structured-input channels (Claude Code's `AskUserQuestion`, opencode's `question` tool) are near-isomorphic; both assets recommend one normalized "agent question" widget. Nobody has decided this yet — it belongs to tickets 04/06/11, not to the research.
- opencode's permission reply cannot rewrite tool input (Claude Code's `canUseTool` can). The asset recommends capability-flagging input-rewriting in the adapter contract rather than intersecting it away silently — treat that as a recommendation to grill, not a decision made.
- opencode's API is fast-moving with no stability guarantee; the asset recommends pinning server+SDK together. If the adapter contract wants version-pinning language, that's ticket 04's call.

## Suggested skills

(Only if available in the next agent's environment.)

- **wayfinder** (`.pocock-skills/wayfinder/`) — the effort's operating method; claim before working, one ticket per session.
- **grill-me** / **domain-modeling** (`.pocock-skills/`) — tickets 04–07 are grilling type; the map's Notes require domain-modeling so terms land in `/CONTEXT.md`.
- **research** (`.pocock-skills/research/`) — only ticket 08 (Tauri sandboxing) remains research-type on the current frontier.
