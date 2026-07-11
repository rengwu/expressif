# Sandboxing agent HTML in a Tauri webview

Research asset for ticket [Validate sandboxing of agent HTML in a Tauri webview](../tickets/08-sandboxing-in-tauri.md). Investigated 2026-07-12 against the Tauri v2 docs, the wry API docs, Tauri's security advisory GHSA-57fm-592m-34r7 (CVE-2024-35222), and the MCP Apps spec (2026-01-26). Question: does the MCP Apps model — sandboxed iframes + host-constructed CSP — actually hold inside a Tauri webview (WKWebView / WebView2 / WebKitGTK)?

## Verdict: holds, with four hard host requirements

The sandbox is airtight against Tauri's IPC bridge **provided the host follows the requirements below** — all derived from documented behavior, none requiring a Tauri fork. The airtightness question is answerable from documentation (chiefly the CVE-2024-35222 patch mechanics), so per the ticket's escalation rule, **no spec-phase prototype is needed**; a runtime conformance probe belongs in implementation tests instead.

## 1. Do sandboxed iframes + strict CSP behave consistently across the three engines?

The mechanisms are baseline web-platform features in all three engines (WebKit on macOS, Chromium via WebView2 on Windows, WebKit via WebKitGTK on Linux), and Tauri's own **Isolation Pattern is built on sandboxed iframes as a core cross-platform feature** — first-party evidence the machinery works everywhere Tauri runs ([isolation pattern](https://v2.tauri.app/concept/inter-process-communication/isolation/)).

What actually varies is **engine currency**, not feature semantics ([webview versions](https://v2.tauri.app/reference/webview-versions/)):

- **Windows**: WebView2 is evergreen Chromium — "comes preinstalled on Windows 11", auto-updated. Best-case currency.
- **macOS**: WKWebView is "updated with the regular OS updates", and "unsupported macOS versions do not receive WebKit updates" — the engine is as old as the OS.
- **Linux**: "very hard to compile accurate information about WebKitGTK on the various distros" — the engine is whatever the distro ships.

**Spec consequence**: expressif must declare minimum engine/OS versions and treat Linux (distro WebKitGTK) as the compatibility floor; sandbox/CSP behavior should be assumed consistent only on engines new enough to have baseline `sandbox` + CSP Level 3.

## 2. Can the sandbox be made airtight against Tauri's IPC bridge?

**Yes — this is the best-documented part, because Tauri shipped and patched exactly this vulnerability.** [CVE-2024-35222 / GHSA-57fm-592m-34r7](https://github.com/tauri-apps/tauri/security/advisories/GHSA-57fm-592m-34r7): before Tauri 1.6.7 / 2.0.0-beta.20, the IPC initialization script was unintentionally injected into iframes (worst on macOS), letting script-enabled iframes invoke the same APIs as the app. The patch introduced the two mechanisms expressif's sandbox rests on:

1. **No IPC init script in subframes**, on any platform.
2. **An invoke key (`__TAURI_INVOKE_KEY__`)** "used to prevent frames that have not been initialized by the Tauri core from sending messages to the Tauri IPC" — messages without it are ignored.

Two documented residual paths remain, and both are closed by sandbox construction:

- **Windows**: iframes *same-origin with the app window* retain IPC access — and wry notes that on Windows initialization "scripts are always added to subframes regardless of the `for_main_frame_only` option" ([wry `WebViewBuilder`](https://docs.rs/wry/latest/wry/struct.WebViewBuilder.html)). A same-origin frame could also simply read the parent's invoke key. **Closed by**: never granting `allow-same-origin`, and never serving widget content from the app's own origin.
- **Linux/Android**: "Tauri is unable to distinguish between requests from an embedded `<iframe>` and the window itself" ([capabilities](https://v2.tauri.app/security/capabilities/)) — Tauri's own origin filtering cannot be the barrier there. **Closed by** the same construction: an opaque-origin frame that never learned the invoke key has nothing valid to send.

**The four hard requirements** (these become spec MUSTs for any Tauri host, feeding tickets 09/11/13):

1. Widget iframes are sandboxed **without `allow-same-origin`** — opaque origin, always. (`allow-scripts` alone is the baseline grant; MCP Apps' inner view iframe needs nothing more.)
2. Widget content is **never served from the app's origin** (including the app's own custom protocol) — a distinct scheme or inline delivery only.
3. **No `remote` capability** is ever configured for any origin widget content could occupy ([capabilities](https://v2.tauri.app/security/capabilities/) — remote IPC access is opt-in; expressif never opts widgets in).
4. **Tauri ≥ 2.0 stable** (the invoke-key protection landed in 2.0.0-beta.20; v1 is out of scope for expressif anyway).

Defense-in-depth, optional: Tauri's [Isolation Pattern](https://v2.tauri.app/concept/inter-process-communication/isolation/) additionally routes every IPC message through a sandboxed isolation iframe with runtime AES-GCM keys — worth recommending to consuming apps (iudex), not required by expressif's model. Note the layering: MCP Apps' **sandbox-proxy** iframe (which the spec requires to have `allow-scripts, allow-same-origin` on **a different origin from the host**) is expressif's own trusted code — only the *inner* view iframe holds agent HTML, and that one is opaque. The proxy's different-origin requirement maps cleanly onto a dedicated custom protocol.

## 3. Does the offline constraint conflict with iframe content delivery?

No — they reinforce each other. MCP Apps delivers view HTML **inline** (`resources/read` returns `text` or base64 `blob`; delivery into the iframe is "left to hosts"), and the spec's default CSP is `default-src 'none'; … connect-src 'none'` with hosts forbidden from allowing undeclared domains ([apps spec](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx)). Nothing needs the network. The delivery mechanics inside Tauri:

- **Inline (`srcdoc`/`blob:`) with CSP as a `<meta http-equiv>` tag** — no HTTP response exists, so the meta tag is the only CSP carrier. Simplest and fully offline.
- **Custom protocol per widget** — wry's `with_custom_protocol` handlers return full HTTP responses, so real CSP *headers* are possible. But two documented quirks argue against it as the default:
  - Custom-scheme URLs have **different shapes and therefore different origins per platform**: `scheme://path` on macOS/Linux vs `http://scheme.localhost/path` on Windows/Android ([wry](https://docs.rs/wry/latest/wry/struct.WebViewBuilder.html)) — CSP `'self'` and origin-based reasoning silently mean different things per OS.
  - Tauri's isolation docs record **"external files not loading correctly inside sandboxed `<iframe>`s on Windows"**, worked around by inlining scripts ([isolation pattern](https://v2.tauri.app/concept/inter-process-communication/isolation/)) — subresource loading into sandboxed frames on WebView2 is exactly the pattern to avoid.

**Recommendation**: the spec's Tauri-host recipe is **fully inlined widget HTML** (all scripts/styles embedded — consistent with the MCP Apps default CSP of `'self' 'unsafe-inline'`), delivered via `srcdoc`/`blob:` with a meta CSP, and assets as `data:` URIs. This sidesteps both Windows quirks and makes the offline property structural rather than policed.

## 4. Platform quirks that should become host spec requirements

| # | Quirk (source) | Spec requirement |
|---|---|---|
| 1 | Same-origin iframes keep IPC on Windows; init scripts always reach subframes there ([advisory](https://github.com/tauri-apps/tauri/security/advisories/GHSA-57fm-592m-34r7), [wry](https://docs.rs/wry/latest/wry/struct.WebViewBuilder.html)) | Never `allow-same-origin`; never app-origin widget content |
| 2 | Linux/Android can't distinguish iframe IPC from window IPC ([capabilities](https://v2.tauri.app/security/capabilities/)) | Tauri origin filtering is never the primary barrier; opaque origin + invoke key are |
| 3 | Invoke-key gate exists only in Tauri ≥ 1.6.7 / 2.0.0-beta.20 ([advisory](https://github.com/tauri-apps/tauri/security/advisories/GHSA-57fm-592m-34r7)) | Require Tauri ≥ 2.0 stable |
| 4 | Custom-scheme origin shape differs per platform ([wry](https://docs.rs/wry/latest/wry/struct.WebViewBuilder.html)) | No CSP/origin logic that depends on `'self'` under a custom scheme |
| 5 | External files fail in sandboxed iframes on WebView2 ([isolation docs](https://v2.tauri.app/concept/inter-process-communication/isolation/)) | Inline all widget subresources; no cross-frame subresource fetches |
| 6 | WKWebView/WebKitGTK ride the OS/distro; no updates on unsupported macOS ([webview versions](https://v2.tauri.app/reference/webview-versions/)) | Declare minimum OS/engine versions; Linux WebKitGTK is the floor |
| 7 | `srcdoc`/`blob:` frames have no HTTP response ([apps spec](https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx) leaves delivery to hosts) | CSP travels as a `<meta http-equiv>` tag in host-assembled HTML |

Plus one implementation-phase obligation (not spec prose): a **runtime conformance probe** — a test widget that attempts `window.__TAURI__`, `window.__TAURI_INTERNALS__`, `window.ipc.postMessage`, `parent` access, and an external fetch, asserting all fail — run per platform in CI. That is the cheap verification the ticket contemplated as a prototype, landed where it stays useful.

## Sources

- Tauri security overview: https://v2.tauri.app/security/
- Capabilities & remote-URL/iframe caveats: https://v2.tauri.app/security/capabilities/
- CSP in Tauri: https://v2.tauri.app/security/csp/
- Isolation pattern (sandboxed-iframe precedent; Windows inline-script quirk): https://v2.tauri.app/concept/inter-process-communication/isolation/
- iFrame IPC bypass advisory (CVE-2024-35222): https://github.com/tauri-apps/tauri/security/advisories/GHSA-57fm-592m-34r7
- wry WebViewBuilder (init-script frame targeting, custom protocols, IPC handler): https://docs.rs/wry/latest/wry/struct.WebViewBuilder.html
- Webview versioning per platform: https://v2.tauri.app/reference/webview-versions/
- MCP Apps spec 2026-01-26 (sandbox/CSP/host obligations): https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx
