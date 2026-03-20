# TheGuardian Addon v.2

## Overview

TheGuardian Addon is a mitmproxy addon that behaves like a browser-side content blocker, but runs at the proxy layer. It emulates a browser-level blocker without relying on browser extensions.

Its responsibilities include:

- applying DNR-like rulesets
- injecting CSS and JS into HTML pages
- enforcing domain blocklists from `.blk` files
- supporting a full-page whitelist / bypass model
- exposing logs consumed by the WPF GUI
- supporting streaming-safe behavior for SSE, uploads, and downloads
- supporting auto-reload flows coordinated by the GUI

The addon is intentionally conservative in a few critical areas:

- HTML rewriting is isolated to the `response()` hook
- SSE and large/binary transfers are handled in `responseheaders()` / `requestheaders()`
- page whitelist logic uses a lease model to avoid reintroducing the old unstable top-remembered architecture
- USERNAV tries to represent only true user page navigations, not browser internal traffic

## Main goals of the current architecture

The current addon (version 2) is a rewrite intended to improve maintainability and stability compared to the old implementation (version 1). It's also significantly smaller and faster than the previous version.

Primary goals:

1. keep rulesets / injection / blockers / reload logic clearly separated
2. avoid cross-site contamination in whitelist logic
3. preserve compatibility with sensitive sites such as ChatGPT and YouTube
4. expose logs that are easy for the GUI to consume
5. keep configuration centralized in `config.json`

## High-level request lifecycle

The addon logic is distributed across mitmproxy hooks:

### `requestheaders(flow)`

Used before request bodies are fully processed.

Purpose:

- enable streaming uploads for large or chunked upload-like requests

This avoids buffering the full upload in the proxy.

### `request(flow)`

Main request-time logic.

Purpose:

- compute client/browser key
- evaluate whitelist / bypass
- evaluate site blocklists
- evaluate DNR-like rulesets
- log MATCH / BYPASS / USERNAV / SITE_BLOCK
- serve virtual control endpoints such as `/__tg/ctl`

This is the main decision point for filtering behavior.

### `responseheaders(flow)`

Used after response headers arrive, before the body is fully buffered.

Purpose:

- preserve SSE streaming
- preserve large / binary / chunked downloads
- avoid streaming HTML, because HTML may need later rewriting/injection in `response()`

This hook is essential for ChatGPT/Github like streaming and for upload/download correctness.

### `response(flow)`

Main response-time logic for HTML.

Purpose:

- detect HTML documents
- decide whether injection is allowed
- inject CSS / JS / reload agent where appropriate
- keep bypassed pages as untouched as possible

This hook must remain careful: streaming HTML here would break rewriting.

## Whitelist / bypass model

Whitelist support is one of the most important and delicate parts of the addon.

### Goal

If a page belongs to a whitelisted site, all resources genuinely belonging to that page should be allowed through, without breaking unrelated tabs or unrelated sites.

### Why this is hard

Modern sites load many resources through chains such as:

- ad-tech redirects
- vendor domains
- nested iframes
- background fetches
- requests where initiator/referer/origin do not clearly point back to the top page

A purely static whitelist check on the request host is not enough.

### Current approach

The addon uses a **page bypass lease** model.

A lease is opened for a browser/client key when strong evidence exists that a whitelisted top page is active. That lease can then allow related resource chains for a short time.

This is much more conservative than the old remembered-top architecture, but much more stable.

### Main concepts

#### Static whitelist

A request can be bypassed immediately when its host matches a configured whitelist entry.

#### Link-based whitelist

A request can also be bypassed when strong linkage exists via:

- initiator
- referer
- origin

#### Lease

When a whitelisted page is detected, the addon opens a short-lived lease:

- `site`
- `top_host`
- `top_dom`
- timestamp / expiry

Later requests on the same browser key may inherit bypass when they look related to that whitelisted page.

### Guardrails

The lease logic contains guardrails to reduce contamination:

- browser key now includes browser family, not only client IP
- explicit other-first-party signals are treated carefully
- ad-tech chains are recognized separately from unrelated first-party tabs

This is what prevents, for example, cnn.com in one browser tab from freely leaking into unrelated sites.

### Logs

Relevant logs include:

- `BYPASS`
- `LEASE_HIT`
- `LEASE_OPEN`
- `LEASE_RESET`

These are especially useful for debugging placeholder-only ads and cross-tab contamination.

## DNR-like rulesets

The addon loads DNR-like rulesets from JSON files configured in `config.json`.

Supported behaviors include:

- block
- redirect
- allow
- modifyheaders (where applicable in the implementation)

The rulesets are evaluated during `request()`.

### Global master switch

`globally_enable_rulesets` disables:

- rulesets
- CSS injection
- JS injection
- redirect resources

This is intended as a GUI master switch.

### Logs

Each ruleset hit can produce `MATCH` lines, which the GUI can use for telemetry and counters.

## CSS / JS injection

The addon supports injection into HTML documents when rulesets are globally enabled and the specific injection sections are enabled.

### CSS injection

Used for:

- cosmetic filtering
- hiding placeholders
- page cleanup

### JS injection

Used for:

- browser-like behavioral fixes
- anti-annoyance flows
- helper scripts derived from enabled rulesets

### Important constraint

Injection should not be forced onto bypassed pages unless explicitly intended. Sensitive sites can break if modified unnecessarily.

## Site blockers (`.blk` files)

The addon can block navigation to domains listed in `.blk` files.

This is conceptually separate from rulesets.

### Purpose

Rulesets emulate adblock/browser filtering.  
Blocklists represent policy-level domain blocking.

### Global master switch

`globally_enable_blockers` disables the entire `.blk`-based blocking system.

### Behavior

Depending on configuration, whitelist may or may not override site blocklists.

### Logs

When active, the addon can emit `SITE_BLOCK` logs and serve a block page.

## USERNAV

USERNAV is used by the GUI to build the user’s visited-sites list.

### Purpose

Represent true top-level pages intentionally visited by the user.

### Challenge

Browsers also make internal/document-like requests that can look like navigations.

### Current approach

USERNAV uses a stricter heuristic:

- requires real document-navigation-like signals
- prefers strong browser hints such as `sec-fetch-user`
- rejects technical/API/download-like pseudo-navigation

This avoids false positives like browser model downloads or optimization service fetches.

### Logs

Example:

- `USERNAV key=... domain=... url=...`

## Streaming logic

Streaming support is intentionally separated from HTML rewriting.

### Response streaming

Handled in `responseheaders()`.

Purpose:

- preserve SSE
- preserve chunked/large/binary downloads

This is what restored normal ChatGPT progressive streaming.

### Request streaming

Handled in `requestheaders()`.

Purpose:

- avoid buffering large upload bodies
- preserve upload behavior for large/chunked upload-like requests

### Important rule

HTML is not streamed through this mechanism because HTML may need rewriting later.

## Auto-reload / reload agent

The addon supports a browser reload agent and a virtual endpoint used by the GUI.

### Purpose

When bypass or filtering state changes, the GUI may trigger page reload behavior so tabs reflect the new policy.

### Components

- injected reload agent script
- virtual `/__tg/ctl` endpoint
- config-driven generation / targets / mode

### Important caution

Reload and agent injection must stay conservative on sensitive pages.

## GUI integration

The WPF GUI depends on addon logs and config-driven behavior.

The addon is designed to expose logs such as:

- `USERNAV`
- `MATCH`
- `BYPASS`
- `SITE_BLOCK`
- telemetry-like streaming/debug lines

The GUI uses these for:

- visited sites
- counters
- ruleset statistics
- current state/debugging
- config reload behavior

## Recommended debugging workflow

For whitelist/bypass issues:

1. inspect `BYPASS`, `LEASE_HIT`, `LEASE_OPEN`
2. check whether requests still produce `MATCH`
3. compare top page host, initiator, and lease fields

For streaming issues:

1. inspect `STREAM_ON(SSE)`, `STREAM_ON(DL)`, `STREAM_ON(UL)`
2. verify that HTML is not being streamed
3. verify behavior on ChatGPT / uploads / downloads

For USERNAV issues:

1. inspect `USERNAV`
2. verify request shape
3. check whether pseudo-navigation requests are being filtered out

## Design philosophy

The addon should prefer:

- conservative correctness over overly clever attribution
- localized fixes over global fragile heuristics
- minimal interference with sensitive sites
- logs that explain why something happened

The current implementation is the result of multiple iterations focused on:

- stable whitelist behavior
- preserving streaming
- avoiding old remembered-top chaos
- keeping the addon understandable and maintainable
