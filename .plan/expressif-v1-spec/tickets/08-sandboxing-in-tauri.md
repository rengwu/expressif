---
type: research
blocked_by: [01]
assets: [.plan/expressif-v1-spec/assets/tauri-sandboxing.md]
---

# Validate sandboxing of agent HTML in a Tauri webview

## Question

Agent-authored HTML in the chat is XSS by design; MCP Apps prescribes sandboxed iframes + CSP. Validate that this model actually holds inside expressif's first embedding target — a Tauri webview (WKWebView on macOS, WebView2 on Windows, WebKitGTK on Linux):

- Do nested sandboxed iframes and strict CSP behave correctly and consistently in all three webviews?
- Can the sandbox be made airtight against Tauri's IPC bridge (a widget must never reach `window.__TAURI__`)?
- Does the offline constraint (no external fetches) conflict with anything in how these webviews resolve iframe content (blob/data/custom-scheme URLs)?
- Known platform quirks that should become spec requirements for hosts.

Deliver a cited summary asset; escalate to a cheap prototype only if documentation genuinely can't answer the airtightness question.

## Answer

Researched 2026-07-12; full findings with citations in [tauri-sandboxing.md](../assets/tauri-sandboxing.md). **The MCP Apps sandbox model holds in Tauri webviews**, and no prototype was needed — airtightness is answerable from documentation because Tauri shipped and patched exactly this vulnerability (CVE-2024-35222: iframes reaching the IPC bridge).

- **Airtight against IPC, by construction**: since Tauri 2.0.0-beta.20 the IPC init script is never injected into subframes and an invoke key blocks messages from uninitialized frames. The two documented residual paths — Windows same-origin iframes retain IPC, and Linux/Android can't distinguish iframe IPC from window IPC — are both closed by four hard host requirements: widget iframes never get `allow-same-origin` (opaque origin always), widget content is never served from the app's origin, no `remote` capability ever covers a widget origin, and Tauri ≥ 2.0 stable. The MCP Apps sandbox-proxy iframe (different origin, expressif's own trusted code) layers cleanly on top; only the inner, opaque iframe holds agent HTML.
- **Consistency across engines**: sandbox + CSP semantics are baseline in all three engines (Tauri's own Isolation Pattern is built on sandboxed iframes cross-platform); what varies is engine *currency* — WebView2 is evergreen, WKWebView rides the macOS version, WebKitGTK rides the distro. The spec must declare minimum OS/engine versions with Linux as the floor.
- **Offline and delivery reinforce each other**: the recipe is fully inlined widget HTML delivered via `srcdoc`/`blob:` with CSP as a `<meta>` tag (no HTTP response exists there). Custom-protocol delivery is rejected as the default: custom-scheme origins have different shapes per platform (breaking `'self'`-based CSP reasoning), and WebView2 has a documented quirk where external files fail to load inside sandboxed iframes.
- The asset's quirk table (seven entries) is written to become host spec MUSTs in tickets 09/11/13, plus one implementation-phase obligation: a per-platform CI conformance probe that asserts `window.__TAURI__`, IPC postMessage, parent access, and external fetches all fail from inside a widget frame.
