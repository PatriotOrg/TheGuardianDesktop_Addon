# TheGuardianDesktop (mitmproxy add-on)

## Preface

**TheGuardianDesktop** is a [mitmproxy/mitmdump](https://www.mitmproxy.org/) add-on that emulates key behaviors of a browser ad/tracker blocker (like TheGuardian Extension), but at the **network/proxy layer**.

It does two main things:

1. **Applies static blocking/redirect rules** (DNR-like) to requests (block / allow / redirect-to-local). It uses the whole set of rules used by TheGuardian Extension as they are.
2. **Injects JS/CSS** into HTML documents to emulate extension-side page modifications. It uses the whole set of JS/CSS used by TheGuardian Extension as they are.

It also includes two “hard” controls that operate above DNR rules:

- **Bypass** (per-site): make the add-on behave as if it doesn’t exist for a site.
- **Site Blocklists** (per-file): completely deny navigation to listed domains, including HTTPS, so the browser cannot reach them.

 

### Efficiency and speed

#### Where TheGuardianDesktop is fast

With the current optimizations (rule indexing/caching + byte-level injection), the add-on’s **own overhead** can be very low:

- **Request-side DNR matching** is typically in the **single-digit milliseconds per request** on busy pages, and drops further with caching (repeat URLs, repeat hosts).

- **Response-side injection** can be near-negligible (sub-millisecond) when byte-level injection is used and expensive decode/recompress loops are avoided.

In practice, once warm caches kick in, the add-on can sustain high request rates with modest CPU usage, especially on pages that repeat common tracker/ads endpoints.

#### Where it will never beat an extension

Even when optimized, a mitmproxy add-on is still a **user-space network middlebox**. That means it inherently pays costs that browser-native extensions don’t:

- **Extra hop + user-space processing** on every request (proxy parsing, flow objects, event dispatch).

- **TLS interception overhead** (MITM certificate, handshake handling, and in some modes additional bookkeeping).

- **Less perfect context**: the proxy sees network flows, not the browser’s full request context (frame tree, exact initiator chain, document lifecycle, service worker context, etc.).

A browser adblocker extension (especially one using the browser’s native request filtering APIs) is often “faster by design” because:

- matching may be implemented in optimized native code,

- it runs closer to the network stack,

- it can block earlier with less data movement,

- and it has richer browser context to avoid doing unnecessary work.

So: **an extension generally has the advantage in raw filtering efficiency**, especially in edge cases, and can be more “correct” in context-sensitive decisions.

---

### Advantages of using the add-on (why it can still be the better choice)

#### 1) Works beyond the browser (system-wide coverage)

The biggest advantage is scope:

- **All browsers** (even those without extension support)

- **In-app webviews / embedded browsers**

- **Other HTTP(S) clients** that respect proxy settings (desktop apps, launchers, some updaters, etc.)

If your goal is “my whole machine should behave like it has TheGuardian Extension,” the add-on can get you closer than any single browser extension.

#### 2) “Hard” blocking capabilities the browser may not provide

At the proxy layer you can do things that are difficult/impossible for extensions:

- **Hard deny navigation** via Site Blocklists:
  
  - deny at TLS SNI / CONNECT / request stage,
  
  - the browser cannot reach the domain at all.

- **Replace resources** reliably via local redirects (serve local files) even for contexts where extensions might not hook.

This is useful for:

- parental control / compliance

- enterprise environments

- kiosk setups

- locked-down browsers

#### 3) Centralized policy and simpler governance

With an add-on you can manage:

- one `config.json`

- one set of rulesets

- one set of blocklists

- one logging pipeline

This is easier to distribute and audit in a controlled environment than “ensure every browser profile has the right extension and settings.”

#### 4) Observable and debuggable

A proxy can log:

- which rules matched

- which hosts were blocked/redirected

- what was injected and when

- telemetry about bottlenecks and cache hits

That’s valuable for:

- performance tuning

- troubleshooting “why is this site broken?”

- building UI dashboards (which you’re already doing)

#### 5) Bypass controls that behave like a “site exception switch”

Because bypass can be made “active” and sticky, you can effectively say:

- “for cnn.com, pretend the blocker does not exist”  
  and have that apply broadly even to third-party requests while you’re on that site.

---

### Disadvantages (tradeoffs and why extensions often feel smoother)

#### 1) Less context than the browser

An extension can know:

- exact frame that triggered the request,

- exact initiator chain,

- extension-specific allowlists/per-site rules integrated with page lifecycle.

A proxy can infer some of this, but it will never be perfect. That’s why you sometimes see:

- false positives/negatives in edge cases,

- site breakage if rules are aggressive,

- more reliance on bypass/site-fixes.

#### 2) TLS interception complexity

To filter HTTPS you’re effectively MITM’ing:

- you must install/maintain a root cert,

- some apps pin certificates (hard fail),

- some environments/policies disallow MITM.

Extensions don’t require any of that.

#### 3) Operational overhead

A proxy add-on is another moving part:

- start/stop lifecycle

- crash handling

- OS proxy mode selection

- potential conflicts with VPNs/security software

- performance tuning may depend on system load and mode

Extensions are “click install and go.”

#### 4) Potential privacy/security perception

Even if the add-on is safe, users may be wary of:

- traffic passing through a local MITM proxy,

- logs that might contain URLs/hosts.

Extensions stay inside the browser sandbox and often feel “less invasive” to users.

---

### When a user should choose the add-on over a browser extension

#### Use the add-on if you want:

- **system-wide filtering** (multiple browsers and apps)

- **hard domain blocking** that the browser cannot bypass

- a **central configuration** for a family/office/kiosk setup

- **consistent behavior** across browsers/profiles

- **logging/telemetry** and operational control (start/stop/restart)

- extension-like behavior for browsers that can’t run extensions well

#### Prefer a browser extension if you want:

- the **lowest friction** install and daily use

- the **best compatibility** with complex sites

- minimal system/network complexity (no certificates, no proxy modes)

- filtering that is most aligned with browser internals and page lifecycle

---

### Bottom line comparison

- **Extensions** win on “native efficiency + correctness in browser context + simplicity.”

- **TheGuardianDesktop add-on** wins on “coverage + enforceability + centralized control + hard blocking + observability.”

A user should use the add-on instead of an extension when their goal isn’t just “block ads in one browser,” but rather:

> “Make my whole desktop environment behave as though TheGuardian is built into the network stack—controllable, enforceable, and centrally configured.”

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
  - **redirect**: serve a local resource from disk (or redirect internally)

### JS/CSS injection

- Injects assets from:
  - `assets/js/`
  - `assets/css/`
- Injection is applied only to HTML document responses (main-frame/sub-frame documents).

### Bypass (“act as if the add-on doesn’t exist”)

- If a site’s domain is listed in `bypass_hosts`, the add-on can bypass:
  - all DNR filtering (block/redirect/allow)
  - all injections (JS/CSS)
  - all other modifications
- Supports “active bypass” triggered by **top-level navigation** to a bypassed site, so that third-party subrequests are also bypassed even if their initiator is not the site domain.

### Site Blocklists (hard block)

- Reads one or more domain-list files (*.blk text files) and blocks navigation entirely.
- Can block at:
  - HTTP request stage
  - TLS SNI stage (for modes where CONNECT doesn’t exist)
  - CONNECT stage (in regular/upstream proxy modes)
- Shows a customizable **external HTML block page**.
- Supports **per-blocklist block pages** with CSS/images served by the add-on using **same-host assets**.

---

## High-level architecture

The add-on is made of two main components:

- `guardian_addon/guardian_addon.py`
  The mitmproxy add-on entry point.

- `guardian_addon/guardian_core.py`  
  Loads config, manages state, hooks request/response, performs bypass/blocklist checks, applies DNR, performs injection, logging/telemetry.

- `guardian_addon/dnr_engine.py`  
  DNR-like rule engine: loads rulesets and matches requests efficiently (indexes, caching, fast paths for common urlFilter patterns).

In production, `dnr_engine.py` and `guardian_core.py`  modules are compiled using **Cython**, to protect the engineering work from prying eyes and abuse of any kind.

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
   - initiator domain when available (Referer / Sec-Fetch-Site / headers, depending on mode)

2. **Site Blocklists (hard block)**
   
   - If the request host matches any enabled site-blocklist:
     - navigation/document requests are answered immediately with a block page (HTTP 451)
     - other subresource requests are denied (403)
   - This happens *before* DNR and injection logic.

3. **Bypass logic**
   
   - If bypass is active for the current client (or this request matches bypass_hosts),
     the add-on returns early and does **nothing** (no rules, no injection, no redirect).

4. **DNR matching**
   
   - Normalize URL for DNR matching (where needed)
   - Find the first matching rule (respecting rule priority / order within ruleset)
   - Apply the action:
     - **allow**: explicitly allow
     - **block**: deny request
     - **redirect**: serve local resource (cached) or redirect internally

5. **Pass-through**
   
   - If no rule matches, request continues unmodified.

---

## Response pipeline

For each response, the add-on typically does:

1. **Early exits**
   
   - If request is bypassed: do nothing
   - If not HTML/document: do nothing
   - If the response is from an internal block page or internal asset: do nothing

2. **Injection**
   
   - Inject CSS and JS into HTML documents
   - Designed to be fast:
     - byte-level injection is used where possible
     - avoids expensive decode/encode/recompress loops
   - JS may be inserted in different phases (e.g., “start”, “idle”) depending on configuration/strategy.

---

## How DNR rules are applied

### Rulesets

Rulesets are defined in `config.json` under `dnr_rulesets`. Each entry is typically an object like:

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

- optionally **requestDomains / initiatorDomains** checks if present in rules

### urlFilter handling (common forms)

The engine supports the most common DNR urlFilter patterns efficiently:

- `||example.com^`  
  “Domain-anchored”: matches example.com and its subdomains.  
  These rules are indexed by host suffix for speed.

- literal substring patterns  
  Simple `urlFilter` values without wildcards are treated as fast substring checks.

- wildcard patterns  
  Some filters may require slower matching (e.g., wildcard-like semantics).

### Action execution

When a rule matches:

- **block**
  
  - The request is denied immediately.

- **allow**
  
  - The request is explicitly allowed (useful when rules would otherwise block).

- **redirect**
  
  - The add-on serves a local file (or a preconfigured local response) instead of the network request.
  
  - Local redirect content is cached in memory for speed.

### Precedence and ordering

The add-on follows a predictable precedence chain:

1. Site Blocklists (hard deny)

2. Bypass (do nothing at all)

3. DNR matching (allow/block/redirect)

4. Injection (response stage; only if not bypassed and response is HTML)

Within DNR:

- rulesets are processed in configured order

- rules within a ruleset are evaluated in order unless optimized/indexed (the result remains consistent with the same rule priority/ordering policy implemented by the engine)

---

## Bypass behavior

### What “bypass” means

If a site/domain is bypassed, TheGuardianDesktop should behave as if it isn’t running:

- no DNR rules applied

- no redirects

- no injections

- no rewriting

### `bypass_hosts`

Configured in `config.json`:

`"bypass_hosts": ["cnn.com", "example.org"]`

Matching is suffix-based: `cnn.com` bypasses `edition.cnn.com`, `www.cnn.com`, etc.

### “Active bypass”

Modern sites frequently load ads/tracker requests from iframes/scripts whose initiator is *not* the original site.  
To ensure bypass works for the whole browsing session on that site, the add-on uses **active bypass**:

- When a **top-level navigation** targets a bypass host, active bypass is enabled.

- While active bypass is enabled, all subsequent requests from the same client are bypassed,  regardless of initiator, until the user navigates top-level to a non-bypassed host.

### Client key mode + TTL (robustness)

Active bypass is stored per “client key”, configurable:

- `"bypass_key_mode": "ip"` (legacy)

- `"bypass_key_mode": "ip_ua"` (recommended default)

- `"bypass_key_mode": "ip_port"`

- `"bypass_key_mode": "ip_ua_port"`

Active bypass also has a dedicated TTL:

`"bypass_active_ttl_seconds": 600`

The TTL is refreshed (“touched”) while traffic continues, so bypass remains stable during navigation.

### Bypass logs (on-change)

The add-on can log bypass state changes (no spam):

- `BYPASS_ON ...`

- `BYPASS_OFF ...`

---

## Site Blocklists: hard navigation blocking

This feature is designed to prevent the browser from reaching listed domains at all.

### Blocklist files

Text files where each line contains a domain/host, e.g.:

`xvideos.com
example.org
0.xxx-cdn.com`

Comments and empty lines are ignored (`#`, `;`, `//`).

### Configuration

Enable one or more blocklists:

`"site_block_enabled": true,
"site_blocklists": [
{"id": "adult",
"path": "assets/blocklists/adult.blk",
"enabled": true,
"page_path": "assets/blocklists/blocked_adult.html",
"assets_dir": "assets/blocklists"
}, ... ]`

### When and how blocking occurs

Depending on mode and protocol, blocking can occur at different layers:

- **TLS ClientHello (SNI)**
  
  - In modes where CONNECT is not used (e.g., local modes), 
    TLS SNI provides the target host early.
  
  - The add-on can close the connection immediately for blocked SNI hosts.

- **HTTP request stage**
  
  - When an HTTP request is seen, blocked hosts are answered immediately with a block page or 403.

- **CONNECT stage (regular/upstream proxy modes)**
  
  - When a CONNECT is seen, blocked hosts are denied before the tunnel is created.

### Block pages: external HTML + per-list pages

Instead of hard-coding HTML, block pages are loaded from disk:

- global default block page config (fallback)

- per-blocklist `page_path` overrides the default

### CSS/images with relative paths (same-host assets)

Relative assets inside the block page (e.g. `<img src="logo.svg">`) are supported.

To avoid timeouts and ensure assets load even when the domain is blocked, the add-on uses **same-host asset serving**:

- HTML relative paths are rewritten to a special internal path under the *same* host, e.g.  
  `https://blocked-site.com/_tg_block_assets/adult/logo.svg`

- the add-on intercepts these requests and serves the file from `assets_dir`

This prevents DNS issues, bypass issues, and render-blocking timeouts.

---

## Configuration

The add-on reads configuration from a `config.json`. The path can be passed via environment variable:

- `TG_CONFIG=/path/to/config.json`

Typical config keys:

- `dnr_rulesets`: list of ruleset objects `{id, path, enabled}`

- `bypass_hosts`: list of domains to bypass completely

- `bypass_key_mode`, `bypass_active_ttl_seconds`: active bypass behavior

- injection flags/paths (depending on your repository layout)

- site blocklists:
  
  - `site_block_enabled`
  
  - `site_blocklists[]` objects
  
  - `site_block_page` default/fallback settings

---

## Run (mitmdump / mitmproxy)

Example with mitmdump:

`set TG_CONFIG=C:\ProgramData\TheGuardianDesktop\config.json 
mitmdump -s guardian_addon/guardian_addon.py --set confdir="C:\ProgramData\TheGuardianDesktop\mitm_conf --set termlog_verbosity=info --mode local:chrome`

Common modes you may use:

- `regular` (explicit proxy; browser points to proxy)

- `local` / `local:chrome` (OS-level interception modes)

> Note: Some mitmproxy hooks (like CONNECT handling) are only triggered in certain modes. The add-on is designed to still enforce blocking via TLS SNI / request-stage checks in “local” modes.

---

## Logging and telemetry

### Log prefixes

The add-on logs key events with `[TG] ...` prefixes (and telemetry with `[TG][TEL] ...` in builds that include telemetry).

You may see:

- bypass state transitions (`BYPASS_ON`, `BYPASS_OFF`)

- site block events (`SITE_BLOCK ...`)

- DNR match lines (`MATCH ...`)

- periodic telemetry summaries (`[TG][TEL] ...`)

### Log level

The UI or your mitmdump invocation can control log verbosity; additionally the add-on can filter logs based on configured log level.

---

## Performance notes

The add-on includes several performance optimizations:

- Efficient DNR matching:
  
  - host-suffix indexing for `||domain^` rules
  
  - fast substring matching for simple literals
  
  - precompiled regex where needed

- Caching:
  
  - match result cache (hits and misses)
  
  - redirect resource cache (local files cached in memory)
  
  - optional host decision cache to reduce repeated matches

- Injection speed:
  
  - byte-level injection where possible
  
  - avoids full HTML lowercasing and multi-pass string rebuilds
  
  - avoids costly decode/encode/recompress loops

If you enable telemetry, it can show whether time is spent in:

- rule matching

- injection

- decode/encode operations

---

## Troubleshooting

### “Bypass host is in bypass_hosts but rules still trigger”

This usually happens when bypass only checks `initiator` but requests are generated by 3rd-party iframes/scripts.  
The add-on’s **active bypass** is meant to solve this by sticking bypass to a client after top-level navigation.

Ensure:

- your `bypass_hosts` includes the base domain (`cnn.com`, not only `edition.cnn.com`)

- active bypass is enabled (default)

- key mode (`ip_ua`) fits your environment

### “Block page is slow / logo doesn’t load”

If block page assets are not served correctly, the browser may wait on render-blocking resources.  
Use **same-host assets** mode so CSS/images are served from the blocked host via internal paths.

### “No CONNECT logs”

CONNECT only exists in `regular`/explicit proxy modes.  
In local interception modes, TLS SNI/request-stage blocking are used instead.

---

## License / Disclaimer

This project is a network-level filtering tool. Use responsibly and in accordance with local laws and policies.
