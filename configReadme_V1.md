# TheGuardianDesktop – `config.json` Reference

This document describes **all configuration settings** available in `config.json`, what they do, and how they affect the mitmproxy add-on behavior.

**Add-on entry point:** `guardian_addon/guardian_addon.py`  
**Core logic:** `guardian_addon/guardian_core.py`  
**Static rules engine:** `guardian_addon/dnr_engine.py`

> Path note: paths like `assets/...` are resolved **relative to the folder that contains `config.json`**.

---

## 1) Logging & Debug

### `debug_log_matches` *(bool)*

If `true`, prints a log line for each **network DNR match** (block/allow/redirect), e.g.:
`[TG] MATCH seq=... ruleset=... action=... host=... url=...`

✅ Useful for rules debugging  
⚠️ Can slow down UI logging on noisy websites (news, social, etc.)

---

### `debug_log_matches_to_file` *(bool)*

If `true`, writes MATCH lines to a log file as well.

---

### `debug_log_matches_file` *(string)*

File name/path used to store MATCH entries.
Example: `matches.log`

> Note: the file may be created lazily (on first MATCH write).

---

### `debug_log_injection` *(bool)*

If `true`, logs injection planning and execution details, such as:

- `[TG] INJECT_PLAN start=...`
- `[TG] INJECT_PLAN idle=...`

---

### `debug_log_bypass_hit` *(bool)*

If `true`, logs every request that is actually bypassed (useful to detect over-bypass or under-bypass).

---

### `debug_log_streaming` *(bool)*

If `true`, logs when streaming pass-through is enabled for `text/event-stream` responses (SSE).

---

### `log_bypass_state` *(bool)*

If `true`, logs bypass state transitions (not per-request spam):

- `BYPASS_CTX_ON ...`
- `BYPASS_ON ...`
- `BYPASS_OFF ...`

Recommended:

- `true` during tests
- `false` if you want a cleaner production log

---

### `debug_telemetry_exceptions` *(bool)*

If `true`, logs internal telemetry exceptions (mostly for development diagnostics).

---

## 2) Bypass System (static + active)

Bypass means: **“act as if the add-on does not exist for this site”**, therefore:

- no DNR matching (block/allow/redirect)
- no JS/CSS injection
- no rewriting/modification of requests/responses

---

### `bypass_hosts` *(array of strings)*

List of domains that must be bypassed.

Example:

```json
"bypass_hosts": ["chatgpt.com", "paypal.com", "stripe.com"]
```

Matching is suffix-based:

- `cnn.com` matches `edition.cnn.com`

- `example.com` matches `a.b.example.com`

---

### `bypass_key_mode` *(string)*

Defines how the “client key” is computed for bypass contexts.

Supported values:

- `ip` → single identity per source IP

- `ip_ua` → identity per IP + User-Agent (recommended)

Practical guidance:

- Use `ip_ua` if multiple browsers/profiles may share the same proxy/IP

- Use `ip` if you want a very stable bypass and you have a single client

---

### `bypass_include_initiator` *(bool)*

If `true`, bypass decisions may also consider the **initiator** (Referer/Origin/Sec-Fetch-*).  
This improves bypass stability on modern SPAs and third-party resource chains.

Recommended: `true`

---

### `bypass_active_ttl_seconds` *(int)*

TTL (seconds) for **active bypass**.  
Active bypass indicates “the user is currently browsing a bypassed site”.

Typical values:

- `600` (10 minutes)

- `1800` (30 minutes)

- `3600` (60 minutes)

Recommended:

- 600 for regular usage

- 1800+ for fragile SPAs (streaming apps, complex sites)

---

### `bypass_context_ttl_seconds` *(int)*

TTL (seconds) for short-lived **bypass context** used to correlate requests and bootstrap chaining.

Recommended: `60` to `180`

---

### `bypass_active_touch_interval_sec` *(float)*

Throttle for refreshing active bypass TTL while traffic continues.  
This avoids touching TTL on every single request.

Example: `2.0` means “refresh at most once every 2 seconds”.

Recommended: `1.0` to `2.0`

---

### `bypass_promote_static_to_active` *(bool)*

If `true`, a static bypass can be promoted to an active bypass when the add-on detects a real top-level navigation to a bypassed host.

Useful when:

- the proxy starts while a browser tab is already open

- the first visible request is not a clean “navigation” request

Recommended: `true`

---

### `bypass_chain_third_party` *(bool)*

If `true`, during active bypass, the add-on will bypass also the **third-party requests caused by that site** (CDNs / file hosts / external downloads).

This is very important for modern SPAs (e.g. ChatGPT):  
downloads and attachments often come from different domains.

Recommended:

- `true` for SPA compatibility and reliable downloads

- `false` for a stricter/smaller bypass scope

---

### `bypass_ublock_allowlist` *(bool)*

If `true`, enables the **uBlock-style allowlist bypass** behavior for hosts in `bypass_hosts`.

Why it exists: in mitmproxy **local interception** modes (e.g. `local:chrome`) it is not always possible to map every third‑party request back to the correct browser tab/top‑frame.  
Some ad-tech requests may therefore still match ADS/TRK rules even when the site is bypassed.

When enabled, if the user navigates to a bypass host (active bypass), the add-on can apply a *client-scoped* bypass to emulate how uBlock Origin behaves when you add a site to its allowlist.

Default: `false`

---

### `bypass_ublock_scope` *(string)*

Defines how aggressive the uBlock-style allowlist bypass should be.

Supported values:

- `"client"` *(most reliable)*  
  Bypasses **all TG processing for the entire client** while the bypassed site is active.  
  ✅ fixes placeholders and restores all ads  
  ⚠️ other tabs of the same browser client may be temporarily unfiltered

- `"client_ads_only"` *(less intrusive)*  
  Bypasses only:
  - the bypassed site itself
  - + a list of common **ad-tech domains** (see `bypass_ublock_ads_domains`)  
  ✅ reduces side effects on other tabs  
  ⚠️ if an ad endpoint is missing from the list, some placeholders may still appear

Default: `"client"`

---

### `bypass_ublock_client_timeout_sec` *(int)*

Optional auto-timeout for the uBlock-style allowlist bypass.

When `> 0`, the add-on automatically disables the client-scoped bypass if it **does not observe traffic related to the active bypassed site** for the specified number of seconds.

This helps avoid leaving the “client bypass” enabled after you have navigated away.

Examples:

- `0` → disabled (no timeout)
- `20` → disable after 20 seconds of inactivity
- `30` → disable after 30 seconds of inactivity

Default: `0`

---

### `bypass_ublock_ads_domains` *(array of strings)*

Optional list of **ad-tech domains** used by `"client_ads_only"` scope.

You can use this to override or extend the built-in list.

Example:

```json
"bypass_ublock_ads_domains": [
  "doubleclick.net",
  "googlesyndication.com",
  "googleadservices.com",
  "amazon-adsystem.com"
]
```

Note: the add-on already ships with a reasonable default list (Criteo, Rubicon, Index, AppNexus, etc.).  
Leaving this empty is fine.

---


### `active_bypass_system_hosts` *(array of strings)*

Optional allowlist of “system” hosts to bypass while active bypass is enabled.  
This is useful for browser background services that may break a bypassed SPA if modified.

Example (only if needed):

`"active_bypass_system_hosts": ["push.services.mozilla.com"]`

---

### `bypass_static_cache_size` *(int)*

LRU cache size used to speed up static bypass host checks.

Typical: `4096`

---

### `ua_tag_cache_size` *(int)*

LRU cache size used for User-Agent hashing/tagging (CPU optimization).

Typical: `1024`–`4096`

---

### `ctx_cleanup_interval_sec` *(float)*

Minimum interval between cache/context cleanup operations (throttling).

Typical: `2.0`

---

### `bypass_decision_microcache` *(object)*

Short-lived micro-cache for repeated bypass decisions during request bursts.

Example:

`"bypass_decision_microcache": { "ttl_sec": 1.0, "size": 8192 }`

Recommended: keep enabled as-is.

---

### `bypass_propagate_from_host` *(bool)*

Legacy/compatibility flag (typically leave `false`).

---

## 3) DNR Rulesets (ads / trackers / cookies / site_fixes)

### `dnr_rulesets` *(array of objects)*

List of DNR-like ruleset entries.

Each entry:

`{   "id": "ads",   "path": "assets/dnr_rules/ads.json",   "enabled": true }`

- `id`: logical name (also used by injection logic when auto-deriving scripts)

- `path`: JSON ruleset path

- `enabled`: enable/disable the ruleset

> `site_fixes` is often important: it contains site-specific fixes required for layout and functionality.

---

## 4) CSS Injection

### `css_injection.enabled` *(bool)*

Enables CSS injection.

---

### `css_injection.enabled_css_files` *(array of strings)*

List of CSS files to inject (relative paths).

---

### `css_injection.style_id_head` / `css_injection.style_id_end` *(string)*

`id` values used for injected `<style>` blocks.

---

### `css_injection.locations` *(array of strings)*

Where to inject:

- `"head"` → inside `<head>`

- `"end"` → near the end of `<body>`

---

## 5) JS Injection

### `js_injection.enabled` *(bool)*

Enables JS injection.

---

### `js_injection.auto_from_rulesets` *(bool)*

If `true`, the add-on auto-selects which JS bundles to inject based on enabled rulesets.

---

### `js_injection.base_dir` *(string)*

Base directory containing injection scripts.

Example: `assets/js`

---

## 6) Browser Heuristics

### `browser_heuristics.user_agent_contains_any` *(array of strings)*

If not empty, injection is only performed if User-Agent contains at least one specified substring.  
This is used to avoid injecting into non-browser clients.

---

### `browser_heuristics.require_sec_fetch_headers` *(bool)*

If `true`, requires `Sec-Fetch-*` headers to treat a request as a browser navigation/request.  
This is stricter and may exclude some clients.

---

## 7) Redirect Resources (DNR redirect-to-local)

### `redirect_resources.enabled` *(bool)*

Enables redirect-to-local resources (noop scripts, 1x1 pixels, etc.).

---

### `redirect_resources.base_dir` *(string)*

Directory containing local redirect files.

---

### `redirect_resources.known` *(object map)*

Mapping of redirect identifiers to local filenames.

---

## 8) Telemetry

### `telemetry.enabled` *(bool)*

Enables internal counters/statistics.

---

### `telemetry.log_interval_sec` *(number)*

How often to log telemetry summaries.

---

### `telemetry.log_to_mitmproxy` *(bool)*

Logs telemetry to mitmproxy UI log.

---

### `telemetry.log_to_file` *(bool)*

Writes telemetry to a file.

---

### `telemetry.file_path` *(string)*

Path of telemetry file.

---

### `telemetry.top_n` *(int)*

Top N hosts/domains for telemetry summaries.

---

## 9) HTML Rewrite / Injection Engine

### `html_rewrite.decode_before_inject` *(bool)*

If `true`, decode the response body into text before injection.

---

### `html_rewrite.bytes_inject` *(bool)*

If `true`, try byte-level injection when possible (usually faster).

---

### `html_rewrite.bytes_inject_utf8_only` *(bool)*

If `true`, only allow byte-level injection when charset is UTF-8.

---

### `html_rewrite.prefix_scan_limit` *(int)*

Maximum number of bytes to scan for fast injection markers (e.g. `</head>`).

---

## 10) Match Cache

### `match_cache_enabled` *(bool)*

Enables caching of recent DNR match decisions.

---

### `match_cache_size` *(int)*

Maximum entries in match cache.

---

## 11) Host Decision Cache

### `host_decision_cache.enabled` *(bool)*

Enables a host decision cache to reduce repeated expensive checks.

---

### `host_decision_cache.size` *(int)*

Max size.

---

### `host_decision_cache.ttl_sec` *(float)*

TTL for cached entries.

---

### `host_decision_cache.min_hits` *(int)*

Minimum hits before caching a host decision.

---

### `host_decision_cache.count_window_sec` *(float)*

Time window used to count hits.

---

## 12) Site Blocker (Hard Block)

### `site_block_enabled` *(bool)*

Enables hard domain blocking.

---

### `site_blocklists` *(array of objects)*

List of `.blk` lists.

Each entry commonly includes:

- `id`

- `path`

- `enabled`

- `page_path` (custom block page HTML)

- `assets_dir` (assets used by the block page)

---

### `log_site_block` *(bool)*

If `true`, logs when a host is blocked by site blocker.

---

### `site_block_respects_bypass` *(bool)*

If `true`, a bypassed host will not be blocked by site blocker.

If `false`, site blocker has precedence over bypass.

---

### `site_block_page` *(object)*

Global fallback block page configuration.

Common fields:

- `enabled`

- `path`

- `content_type`

- `assets_host`

- `assets_prefix`

- `assets_dir`

- `assets_mode`
  
  - `same_host` *(recommended)*: assets are served under the blocked host
  
  - `assets_host`: use a dedicated “fake” host for assets

---

## 13) CSP Handling (Injection Compatibility)

### `csp_strip` *(object)*

If enabled, removes Content-Security-Policy headers (maximum compatibility, more invasive).

Fields:

- `enabled`

- `remove_report_only`

Recommended: enable only if needed.

---

### `csp_nonce_patch` *(object)*

Less invasive CSP support: attempts to reuse existing nonces and patch directives.

Fields:

- `enabled`

- `patch_report_only`

- `patch_elem_directives`

- `create_missing_directives` *(usually keep false)*

---

## 14) Reserved / Not Used

### `site_allow_hosts` *(object/map)*

Reserved placeholder (currently unused).  
May become a future allowlist feature.

---

# Recommended config presets

## FAST (low overhead + standard filtering)

Goal: best perceived responsiveness, minimal logging overhead, stable SPA compatibility.

`{   "debug_log_matches": false,   "debug_log_injection": false,   "debug_log_bypass_hit": false,   "debug_log_streaming": false,    "telemetry": {     "enabled": true,     "log_interval_sec": 15,     "log_to_mitmproxy": false,     "log_to_file": true,     "file_path": "telemetry.log",     "top_n": 8   },    "match_cache_enabled": true,   "match_cache_size": 30000,    "host_decision_cache": {     "enabled": true,     "size": 8192,     "ttl_sec": 120,     "min_hits": 2,     "count_window_sec": 60   },    "bypass_include_initiator": true,   "bypass_key_mode": "ip_ua",   "bypass_active_ttl_seconds": 1800,   "bypass_context_ttl_seconds": 120,   "bypass_active_touch_interval_sec": 2.0,   "bypass_promote_static_to_active": true,   "bypass_chain_third_party": true,    "csp_strip": { "enabled": false, "remove_report_only": true },   "csp_nonce_patch": {     "enabled": true,     "patch_report_only": true,     "patch_elem_directives": true,     "create_missing_directives": false   } }`

---

## STRICT (maximum blocking + robust injection compatibility)

Goal: aggressive blocking and best injection behavior even on difficult CSP-heavy sites.

`{   "debug_log_matches": false,   "debug_log_injection": false,    "bypass_include_initiator": true,   "bypass_key_mode": "ip_ua",   "bypass_active_ttl_seconds": 3600,   "bypass_context_ttl_seconds": 180,   "bypass_active_touch_interval_sec": 2.0,   "bypass_promote_static_to_active": true,   "bypass_chain_third_party": true,    "match_cache_enabled": true,   "match_cache_size": 50000,    "csp_strip": { "enabled": true, "remove_report_only": true },   "csp_nonce_patch": {     "enabled": true,     "patch_report_only": true,     "patch_elem_directives": true,     "create_missing_directives": false   } }`

> Note: enabling `csp_strip.enabled=true` is more invasive.  
> It increases injection compatibility but may affect how some sites enforce security policies.
