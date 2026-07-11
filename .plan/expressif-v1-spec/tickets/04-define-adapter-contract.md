---
type: grilling
blocked_by: [02, 03]
claimed_by: claude-fable-5-session-163b94bf
claimed_at: 2026-07-11T19:05:00+08:00
---

# Define the harness adapter contract

## Question

Given the mapped surfaces of Claude Code and opencode, what is the minimal adapter interface expressif's core defines — the contract any harness must be wrapped into?

To resolve: the normalized session model (spawn/attach/resume/interrupt), the normalized event stream (what the chat surface needs vs what each harness emits), sending user input, permission-prompt passthrough, MCP plugin registration, and error/exit semantics. Decide what is required vs optional capability (feature-detection for harnesses that lack e.g. thinking events). The contract must be honest about the two harnesses' variance, not a lowest common denominator that cripples both.
