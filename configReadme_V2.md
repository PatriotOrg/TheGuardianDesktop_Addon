# README_CONFIG

## Overview

`config.json` controls the behavior of `tg_addon.py`.

The configuration is grouped by functional area:

- global master switches
- debug/logging
- rulesets
- injection
- site blocking
- whitelist / bypass
- streaming
- USERNAV / telemetry
- auto-reload

Where possible, the addon uses:
- a global master flag
- plus feature-specific enable flags

This lets the GUI expose both broad and granular controls.

## Global master switches

### `globally_enable_rulesets`
**Type:** boolean  
**Default:** `true`

Master switch for the browser-like filtering layer.

When `false`, the addon disables:
- DNR-like rulesets
- CSS injection
- JS injection
- redirect-resource support tied to rulesets

Use this when you want the addon running but want to suspend browser-style filtering logic.

### `globally_enable_blockers`
**Type:** boolean  
**Default:** `true`

Master switch for `.blk`-based domain blocking.

When `false`, the addon disables:
- `.blk` domain checks
- site block decisions
- block page handling tied to site blockers

Use this when you want to keep rulesets active but temporarily disable domain blocklists.

## Debug and logging

### `debug_log_streaming`
**Type:** boolean  
Enables logs such as:
- `STREAM_ON(SSE)`
- `STREAM_ON(DL)`
- `STREAM_ON(UL)`

Useful for ChatGPT streaming and upload/download debugging.

### `debug_log_usernav`
**Type:** boolean (if present in your config/version)  
Controls extra USERNAV diagnostics if implemented.

### lease / bypass debug flags
Depending on your current addon version, config may include lease/bypass debug flags used to emit:
- `LEASE_HIT`
- `LEASE_OPEN`
- `LEASE_RESET`
- `BYPASS`

These are useful for whitelist debugging.

## Rulesets

### `dnr_rulesets`
**Type:** array of objects

Each object typically includes:
- `id`
- `path`
- `enabled`

Only enabled entries are loaded when `globally_enable_rulesets = true`.

Purpose:
- define the DNR-like rule sources the addon uses during request evaluation

## CSS injection

### `css_injection.enabled`
**Type:** boolean

Enables CSS injection, but only if `globally_enable_rulesets = true`.

### `css_injection.enabled_css_files`
**Type:** array of file paths

List of CSS files to concatenate and inject.

Purpose:
- cosmetic filtering
- hiding placeholders
- visual page cleanup

## JS injection

### `js_injection.enabled`
**Type:** boolean

Enables JS injection, but only if `globally_enable_rulesets = true`.

### `js_injection.base_dir`
**Type:** string

Base directory containing JS helper files.

### `js_injection.auto_from_rulesets`
**Type:** boolean

If enabled, derive JS file selection automatically from enabled rulesets.

Purpose:
- support browser-like behavioral fixes
- inject helper scripts related to enabled filtering capabilities

## Redirect resources

### `redirect_resources.enabled`
**Type:** boolean

Enables redirect-resource support for ruleset actions such as redirect, but only if `globally_enable_rulesets = true`.

Used when rules need to replace blocked resources with local/safe placeholders.

## Site blocking / `.blk`

### `site_block_enabled`
**Type:** boolean

Feature-specific switch for `.blk` domain blocking.

Effective only when `globally_enable_blockers = true`.

### `site_blocklists`
**Type:** array of objects

Each object typically includes:
- `path`
- `enabled`
- metadata/label fields depending on your config version

Purpose:
- load blocklists of domains that should trigger site-block logic

### `site_block_respects_bypass`
**Type:** boolean

Controls whether whitelist may override domain blocklists.

- `true`: whitelist can win
- `false`: a blocked domain remains blocked even if whitelisted

This is an important policy setting.

## Whitelist / bypass

### `bypass_hosts`
**Type:** array of host/domain patterns

Static whitelist entries.

Requests matching these entries can be bypassed directly.

### `page_bypass_ttl_sec`
**Type:** number

Lifetime of the page bypass lease.

A short lease allows related subrequests to inherit bypass briefly after a whitelisted page is detected.

This is a key parameter for:
- completeness of whitelist behavior
- contamination risk between unrelated requests

Longer values help difficult ad-tech chains, but increase contamination risk.

## USERNAV

### `usernav.dedupe_sec`
**Type:** number

Controls how quickly repeated navigations for the same browser key/domain are suppressed.

Purpose:
- keep GUI visited-sites list clean
- reduce repeated USERNAV spam for reloads or near-duplicate document requests

## Streaming

### `stream_downloads.enabled`
**Type:** boolean

Enables response streaming for:
- SSE
- large/binary/chunked downloads
- other response cases supported by addon logic

### `stream_downloads.min_bytes`
**Type:** integer

Minimum content length for considering a response “large”.

### `stream_uploads.enabled`
**Type:** boolean

Enables request streaming for large/chunked upload-like requests.

### `stream_uploads.min_bytes`
**Type:** integer

Minimum upload size threshold.

Purpose of streaming section:
- preserve ChatGPT progressive output
- avoid proxy buffering of large uploads/downloads
- keep sensitive flows working

## Auto reload

### `auto_reload`
Object controlling the GUI-coordinated reload mechanism.

Common fields may include:

### `auto_reload.auto_reload_pages`
Enable/disable the reload system.

### `auto_reload.reload_poll_ms`
Polling interval for the reload control endpoint.

### `auto_reload.reload_only_when_visible`
Restrict reload behavior to visible tabs when supported by the injected agent.

### `auto_reload.reload_mode`
Common values:
- `all`
- `hosts`

Controls whether reload instructions apply broadly or only to selected targets.

### `auto_reload.reload_targets`
List of hosts/targets involved in host-targeted reload mode.

Purpose:
- allow GUI-driven refresh when bypass/filtering state changes

## Reload agent on bypass

Depending on your current config/addon version, related fields may include:

### `inject_reload_agent_on_bypass`
Controls whether the reload agent may be injected even on bypassed pages.

### `reload_agent_bypass_exclude_hosts`
Hosts where reload-agent injection should be suppressed even if bypassed.

Typical examples include sensitive sites such as ChatGPT or payment/auth sites.

Purpose:
- avoid breaking sensitive pages while still supporting controlled reload on other bypassed sites

## Telemetry / GUI support

Some config sections control:
- log verbosity
- counters/statistics
- extra debug fields used by the GUI

These settings are not always about filtering directly; many exist to make the GUI informative and diagnosable.

## Practical profiles

### Profile A — Full protection
- `globally_enable_rulesets = true`
- `globally_enable_blockers = true`
- rulesets enabled
- blocklists enabled

Use when you want full addon behavior.

### Profile B — Rulesets only
- `globally_enable_rulesets = true`
- `globally_enable_blockers = false`

Use when you want browser-style filtering without hard blocklist domain blocking.

### Profile C — Blockers only
- `globally_enable_rulesets = false`
- `globally_enable_blockers = true`

Use when you want only `.blk`-based blocking and no DNR/CSS/JS behavior.

### Profile D — Passive / diagnostics
- `globally_enable_rulesets = false`
- `globally_enable_blockers = false`

Use when you want the addon minimally active for diagnostics, logging, GUI integration, or controlled testing.

## Notes

1. Global master switches are stronger than feature-local flags.
2. Streaming should generally remain enabled unless debugging a specific issue.
3. Lease TTL should be tuned carefully: too short weakens bypass completeness, too long increases contamination risk.
4. USERNAV dedupe should stay conservative to avoid noisy visited-site lists.
