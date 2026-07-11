---
type: research
blocked_by: [01]
claimed_by: claude-fable-5-session-163b94bf
claimed_at: 2026-07-12T11:20:00+08:00
---

# Validate sandboxing of agent HTML in a Tauri webview

## Question

Agent-authored HTML in the chat is XSS by design; MCP Apps prescribes sandboxed iframes + CSP. Validate that this model actually holds inside expressif's first embedding target — a Tauri webview (WKWebView on macOS, WebView2 on Windows, WebKitGTK on Linux):

- Do nested sandboxed iframes and strict CSP behave correctly and consistently in all three webviews?
- Can the sandbox be made airtight against Tauri's IPC bridge (a widget must never reach `window.__TAURI__`)?
- Does the offline constraint (no external fetches) conflict with anything in how these webviews resolve iframe content (blob/data/custom-scheme URLs)?
- Known platform quirks that should become spec requirements for hosts.

Deliver a cited summary asset; escalate to a cheap prototype only if documentation genuinely can't answer the airtightness question.
