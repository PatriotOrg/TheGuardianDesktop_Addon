# TheGuardianDesktop (mitmproxy add-on)

## Preface

**TheGuardianDesktop** is a [mitmproxy/mitmdump](https://www.mitmproxy.org/) add-on that emulates key behaviors of a browser ad/tracker blocker (like TheGuardian Extension), but at the **network/proxy layer**.

It does two main things:

1. **Applies static blocking/redirect rules** (DNR-like) to requests (block / allow / redirect-to-local). It uses the same rule sets used by TheGuardian Extension.
2. **Injects JS/CSS** into HTML documents to emulate extension-side page modifications. It uses the same JS/CSS assets used by TheGuardian Extension.

It also includes two “hard” controls that operate above DNR rules:

- **Bypass** (per-site): make the add-on behave as if it doesn’t exist for a site.
- **Site Blocklists** (per-file): completely deny navigation to listed domains, including HTTPS, so the browser cannot reach them.

## Performance & Efficiency

TheGuardianDesktop runs as a **user-space network middlebox** (mitmproxy add-on).  
This provides **system-wide coverage** (not limited to a single browser profile), but it also means some overhead is unavoidable compared to in-browser extensions.

### What can feel slower (normal)

Some websites may feel slightly slower to open compared to the same setup using a native browser extension.  
This can be caused by:

- proxy-level processing on every request (matching, bypass checks, policy checks)
- TLS interception / connection handling (depending on proxy mode)
- high request volume websites (news portals, large SPAs)

The goal of the add-on is to keep the overhead small, stable, and predictable.

### Main optimizations implemented

**1) Fast-path bypass**  
If bypass is active for a site, the add-on exits early and skips:

- DNR matching
- injection planning
- rewrite/redirect logic

This is critical for modern SPAs and streaming apps (ChatGPT-like websites).

**2) Efficient DNR matching**

- Host-suffix indexing for `||domain^` rules
- Fast substring matching for simple `urlFilter` values
- Precompiled patterns where needed

**3) Request metadata caching**
Per-request (per-flow) metadata is cached and reused:

- resource type (`document`, `xhr`, `script`, …)
- initiator domain (when available)
- normalized URL used by the matcher

This reduces duplicated parsing work on noisy websites.

**4) Redirect resource caching**
When rules redirect to local files, those resources are cached in memory to avoid repeated disk reads.

**5) Streaming compatibility**
If the response is `text/event-stream` (SSE), streaming is enabled to preserve real-time rendering.
This prevents “buffered” responses on modern SPAs.

### Practical note

Performance depends heavily on:

- the number of enabled rulesets
- how “noisy” a site is (requests/second)
- proxy mode (regular vs local interception)
- the machine CPU and I/O

For best responsiveness:

- keep debug logging disabled during normal usage
- enable only the rulesets you actually need
- use bypass for sites that are known to break with any filtering

### Browser Extension vs Proxy Add-on (quick comparison)

| Feature / Behavior                                  | Browser Extension (MV3 / DNR) | TheGuardianDesktop (mitmproxy add-on) |
| --------------------------------------------------- |:-----------------------------:|:-------------------------------------:|
| Works only inside the browser                       | ✅                             | ❌                                     |
| System-wide coverage (multiple apps/browsers)       | ❌                             | ✅                                     |
| Lowest latency / best “native” speed                | ✅                             | ❌ *(slightly more overhead)*          |
| Enforceable hard blocking (before page loads)       | ✅                             | ✅                                     |
| Hard domain navigation blocking (SNI/CONNECT level) | ❌ *(limited)*                 | ✅                                     |
| CSS/JS injection capabilities                       | ✅                             | ✅ *(HTML response stage)*             |
| Works with browsers without installing extensions   | ❌                             | ✅                                     |
| Easier compatibility with “special” apps/clients    | ❌                             | ✅                                     |
| Requires proxy setup / certificate (HTTPS MITM)     | ❌                             | ✅                                     |

---

## Contents

- [Key features](#key-features)
- [High-level architecture](#high-level-architecture)
- [Request pipeline](#request-pipeline)
- [Response pipeline](#response-pipeline)
- [How DNR rules are applied](#how-dnr-rules-are-applied)
- [Bypass behavior](#bypass-behavior)
- [Site Blocklists: hard navigation blocking](#site-blocklists-hard-navigation-blocking)
- [Configuration](#configuration)
- [Run (mitmdump / mitmproxy)](#run-mitmdump--mitmproxy)
- [Logging and telemetry](#logging-and-telemetry)
- [Performance notes](#performance-notes)
- [Troubleshooting](#troubleshooting)

---

## Key features

### DNR-like static rules

- Loads one or more **rulesets** from `assets/dnr_rules/*.json`.
- Matches requests using common DNR fields (subset):
  - `urlFilter` (including `||domain^` forms)
  - `resourceTypes` (script/image/xhr/document/…)
  - `initiatorDomains`, `requestDomains` (when present)
- Executes the rule’s `action`:
  - **block**: deny request
  - **allow**: allow request
  - **redirect**: serve a local resource from disk (or rewrite internally)

### JS/CSS injection

- Injects assets from:
  - `assets/js/`
  - `assets/css/`
- Injection is applied only to HTML document responses (main-frame/sub-frame documents).
- Injection supports phased execution strategies (e.g. “start” / “idle”) depending on configuration.

### Bypass (“act as if the add-on doesn’t exist”)

- If a site’s domain is listed in `bypass_hosts`, the add-on can bypass:
  - all DNR filtering (block/redirect/allow)
  - all injections (JS/CSS)
  - all other modifications
- Supports “active bypass” triggered by **top-level navigation** to a bypassed site.
- Supports third-party bypass chaining (optional): bypass cascades to requests “caused by” that site.

### SPA/streaming compatibility (ChatGPT and similar)

Modern SPAs may stream responses using `text/event-stream` (SSE).  
The add-on enables **streaming pass-through** for SSE so that token-by-token rendering is preserved even while the proxy is running.

### Site Blocklists (hard block)

- Reads one or more domain-list files (*.blk text files) and blocks navigation entirely.
- Can block at:
  - HTTP request stage
  - TLS SNI stage (for interception modes)
  - CONNECT stage (in regular/upstream proxy modes)
- Shows a customizable **external HTML block page**.
- Supports **per-blocklist block pages** with CSS/images served by the add-on using **same-host assets**.

---

## High-level architecture

The add-on is made of three main components:

- `guardian_addon/guardian_addon.py`  
  The mitmproxy add-on entry point.

- `guardian_addon/guardian_core.py`  
  Loads config, manages state, hooks request/response, performs bypass/blocklist checks, applies DNR, performs injection, logging/telemetry.

- `guardian_addon/dnr_engine.py`  
  DNR-like rule engine: loads rulesets and matches requests efficiently (indexes, caching, fast paths for common urlFilter patterns).

In production, `dnr_engine.py` and `guardian_core.py` modules may be compiled using **Cython**.

### Assets

- `assets/dnr_rules/*.json`: static DNR-like rulesets
- `assets/js/*.js`: scripts to inject
- `assets/css/*.css`: styles to inject
- `assets/blocklists/*.txt|*.blk`: optional site blocklist files
- `assets/blocklists/*.html|*.css|*.svg|...`: optional block pages + their assets

---

## Request pipeline

For each request, the add-on processes it in this general order:

1. **Determine request metadata**
   
   - URL / host / scheme
   - resource type (document, script, image, xhr, …)
   - initiator domain (when available: Referer/Origin/Sec-Fetch-*)

2. **Site Blocklists (hard block)**
   
   - If the request host matches any enabled site-blocklist:
     - navigation/document requests are answered immediately with a block page (HTTP 451)
     - other subresource requests are denied (403)
   - This happens *before* DNR and injection logic.

3. **Bypass logic**
   
   - If bypass is active for the current client (or this request matches `bypass_hosts`),
     the add-on returns early and does **nothing** (no rules, no injection, no redirect).

4. **DNR matching**
   
   - Normalize URL for DNR matching (where needed)
   - Find the first matching rule (respecting rule priority / order policy)
   - Apply the action:
     - **allow**: explicitly allow
     - **block**: deny request
     - **redirect**: serve local resource or rewrite internally

5. **Pass-through**
   
   - If no rule matches, request continues unmodified.

---

## Response pipeline

For each response, the add-on typically does:

1. **Early exits**
   
   - If request is bypassed: do nothing
   - If not HTML/document: do nothing
   - If the response is from an internal block page or internal asset: do nothing

2. **Streaming handling (SSE)**
   
   - If `Content-Type: text/event-stream` is detected, the add-on enables streaming
     pass-through so the browser can render incremental chunks in real-time.

3. **Injection**
   
   - Inject CSS and JS into HTML documents
   - Designed to be fast:
     - byte-level injection where possible
     - avoids expensive decode/encode/recompress loops

---

## How DNR rules are applied

### Rulesets

Rulesets are defined in `config.json` under `dnr_rulesets`:

```json
{
  "id": "ads",
  "path": "assets/dnr_rules/ads.json",
  "enabled": true
}
```

Only enabled rulesets are loaded and used for matching.

### Matching inputs

When matching a request, the engine uses:

- **URL** (normalized)

- **request host**

- **initiator domain** (when available)

- **resource type** (document/script/image/xhr/…)

- optionally domain constraints (`requestDomains`, `initiatorDomains`) when present

### urlFilter handling (common forms)

- `||example.com^`  
  Domain-anchored rule: matches example.com and its subdomains.  
  These rules are indexed by host suffix for speed.

- literal substring patterns  
  Simple `urlFilter` values without wildcards are treated as fast substring checks.

- wildcard patterns  
  Some patterns require slower matching (e.g. wildcard-like semantics).

### Action execution

When a rule matches:

- **block**
  
  - The request is denied immediately.

- **allow**
  
  - The request is explicitly allowed.

- **redirect**
  
  - The add-on serves a local file (or a preconfigured local response).
  
  - Local redirect content is cached in memory for speed.

### Precedence and ordering

1. Site Blocklists (hard deny)

2. Bypass (do nothing at all)

3. DNR matching (allow/block/redirect)

4. Injection (response stage; only if not bypassed and response is HTML)

---

## Bypass behavior

### What “bypass” means

If a site/domain is bypassed, TheGuardianDesktop behaves as if it isn’t running:

- no DNR rules applied

- no redirects

- no injections

- no rewriting

### `bypass_hosts`

Configured in `config.json`:

`"bypass_hosts": ["chatgpt.com", "example.org"]`

Matching is suffix-based: `cnn.com` bypasses `edition.cnn.com`, `www.cnn.com`, etc.

### “Active bypass”

Some sites break if even *background/system requests* are modified while the user is actively browsing them.  
To provide stable exceptions, TheGuardianDesktop supports **active bypass**:

- When a **top-level navigation** targets a bypass host, active bypass is enabled.

- While active bypass is enabled, requests are bypassed for that browsing context.

### Third-party bypass chaining (recommended for modern SPAs)

Some sites load resources from separate hosts/CDNs; downloads (including ChatGPT attachments) often use different domains.

If enabled, bypass chaining allows bypass to propagate to third-party requests triggered by a bypassed site:

`"bypass_chain_third_party": true`

### Client key mode + TTL

Active bypass is tracked per “client key”:

- `"bypass_key_mode": "ip"`

- `"bypass_key_mode": "ip_ua"` (recommended)

- `"bypass_key_mode": "ip_port"`

- `"bypass_key_mode": "ip_ua_port"`

TTL settings:

`"bypass_active_ttl_seconds": 600, "bypass_context_ttl_seconds": 60`

Notes:

- `bypass_active_ttl_seconds` controls how long bypass stays ON.

- `bypass_context_ttl_seconds` controls how long bypass context is kept for correlation/chaining.

- TTL may be refreshed (“touched”) while traffic continues to keep bypass stable during navigation.

### Bypass logs (on-change)

State changes are logged without spamming:

- `BYPASS_CTX_ON ...`

- `BYPASS_ON ...`

- `BYPASS_OFF ...`

---

## Site Blocklists: hard navigation blocking

This feature prevents the browser from reaching listed domains at all.

### Blocklist files

Text files where each line contains a domain/host, e.g.:

`xvideos.com example.org 0.xxx-cdn.com`

Comments and empty lines are ignored (`#`, `;`, `//`).

### Configuration

`"site_block_enabled": true, "site_blocklists": [   {     "id": "adult",     "path": "assets/blocklists/adult.blk",     "enabled": true,     "page_path": "assets/blocklists/blocked_adult.html",     "assets_dir": "assets/blocklists"   } ]`

### When and how blocking occurs

Depending on mode and protocol, blocking can occur at different layers:

- **TLS ClientHello (SNI)**  
  In interception modes where CONNECT is not used, TLS SNI provides the target host early.

- **HTTP request stage**  
  When an HTTP request is seen, blocked hosts are answered immediately with a block page or 403.

- **CONNECT stage (regular/upstream proxy modes)**  
  When CONNECT is used, blocked hosts are denied before the tunnel is created.

### Block pages: external HTML + per-list pages

Block pages are loaded from disk:

- global default block page config (fallback)

- per-blocklist `page_path` overrides the default

### CSS/images with relative paths (same-host assets)

Relative assets inside the block page (e.g. `<img src="logo.svg">`) are supported.

To avoid timeouts and ensure assets load even when the domain is blocked, the add-on uses same-host asset serving:

- relative paths are rewritten to an internal path under the same host

- the add-on intercepts and serves the assets from `assets_dir`

---

## Configuration

The add-on reads configuration from `config.json`. The path can be passed via environment variable:

- `TG_CONFIG=/path/to/config.json`

Typical config keys:

- `dnr_rulesets`

- `bypass_hosts`

- `bypass_chain_third_party`

- `bypass_key_mode`

- `bypass_active_ttl_seconds`

- `bypass_context_ttl_seconds`

- injection plan / asset paths (repo dependent)

- `site_block_enabled`

- `site_blocklists[]`

---

## Logging and telemetry

### Log prefixes

- `[TG] ...` main add-on logs

Common messages:

- bypass transitions (`BYPASS_*`)

- site blocks (`SITE_BLOCK ...`)

- rule matches (`MATCH ...`)

- injections (`INJECT_PLAN ...`, `INJECT ...`)

### Match logging

If enabled, matched network rules are logged:

`"debug_log_matches": true`

Optional file output:

`"debug_log_matches_to_file": true, "debug_log_matches_file": "matches.log"`

> Note: the file may be created on first write (i.e. after the first match).

---

## Performance notes

The add-on includes several optimizations:

- Efficient DNR matching:
  
  - host-suffix indexing for `||domain^` rules
  
  - fast substring matching for simple literals
  
  - precompiled regex where needed

- Caching:
  
  - match result cache
  
  - redirect resource cache (local files cached in memory)
  
  - request metadata caching per-flow (resource type, initiator, normalized URL)

- Injection speed:
  
  - byte-level injection where possible
  
  - avoids costly decode/encode/recompress loops

Even when optimized, a mitmproxy add-on is still a user-space network middlebox:  
extensions remain “closer to the network stack” and often feel smoother,  
but the proxy-layer approach enables system-wide coverage and enforceable policy.

---

## Troubleshooting

### “Site is in bypass_hosts but downloads fail”

Some sites (including modern SPAs) use external file hosts/CDNs.  
Enable third-party bypass chaining:

`"bypass_chain_third_party": true`

### “Bypass host is in bypass_hosts but rules still trigger”

This can happen if bypass only checks initiator but requests come from iframes/scripts. Use active bypass and ensure `bypass_key_mode` fits your environment (usually `ip_ua`).

### “No CONNECT logs”

CONNECT exists in explicit proxy modes (`regular`, not implemented yet).  
In local interception modes, TLS SNI/request-stage blocking are used instead.

### In short

Extensions are usually faster because they live inside the browser, while TheGuardianDesktop is more flexible and system-wide because it works at proxy level.

---

## License / Disclaimer

This project is a network-level filtering tool. Use responsibly and in accordance with local laws and policies.
