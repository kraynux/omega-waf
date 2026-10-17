<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - MIT License (see LICENSE file) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="384">
</div>

# 🔒 OMEGA-WAF

**mod_magnet application firewall (standalone Lua script)**

> Built by **kraynux** for **Omega-server**
> [kraynux.snake-mackarel.ts.net](https://kraynux.snake-mackarel.ts.net)

Official page: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#compatibility)

**Languages:**
[Français](README.md) · [English](README.en.md) · [Español](README.es.md) · [Русский](README.ru.md) · [中文](README.zh-CN.md)

---

**Omega-Waf** is a single Lua script (`omega-waf.lua`), with no external dependency, meant to run on every HTTP request through a web server compatible with `mod_magnet` (lighttpd natively, or omega-serv via its `magnet` compatibility bridge). It inspects the path, the request and, on POST, the request body for known attack signatures (SQL injection, XSS, command injection, path traversal), applies rate limiting, manages a persistent blocklist with progressive escalation, and logs every decision.

No installation in the classic sense: a single file to drop in, reference from the server configuration, and that's it.

## Table of contents

- [Overview](#overview)
- [Installation](#installation)
- [Configuration](#configuration)
- [Decision pipeline](#decision-pipeline)
- [Detected signatures](#detected-signatures)
- [Monitored sensitive paths](#monitored-sensitive-paths)
- [Rate limiting and escalation](#rate-limiting-and-escalation)
- [Persistence](#persistence)
- [HTTP response codes emitted](#http-response-codes-emitted)
- [Compatibility](#compatibility)
- [Customization](#customization)
- [Known limitations and out of scope](#known-limitations-and-out-of-scope)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Overview

### Goals

- Filter malicious requests as close to the server as possible, before they reach the served application.
- Remain a single, readable file, with no external dependency and no build step.
- Never bring the web server down: any internal script error is caught and logged rather than propagated.
- Limit CPU/memory cost per request (time-based caches, chunked body reads, bounded decoding depth).
- Offer progressive escalation (warning then ban) rather than an immediate hard ban on a first minor incident.

### What Omega-Waf does

- Blocks already-banned IP addresses (persistent blocklist on disk), with an exception for explicitly allowed paths (error page CSS, etc.).
- Monitors a set of sensitive paths (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store`, etc.) and responds either with an HTML warning page or a direct ban on repeat offense.
- Detects User-Agents known as scanning tools or unwanted bots (sqlmap, nmap, nikto, dirbuster, generic crawlers...).
- Applies global per-IP rate limiting across all routes.
- Applies a dedicated, more permissive policy on the routes of a SearXNG search engine (avoids false positives on legitimate search queries containing keywords that would otherwise look like an attack).
- Inspects the path and query string (SQLi, XSS, path traversal, NUL byte).
- Inspects POST request bodies (SQLi, XSS, command injection, path traversal) with a cumulative scoring system.
- Handles progressive escalation: an isolated suspicious behavior triggers an alert or a one-off refusal; repeat offense within a time window triggers a permanent ban.
- Logs every decision (alerts, blocklist additions) in text or JSON format.
- Works both under native lighttpd (hardcoded paths) and injected into omega-serv (paths received dynamically from the server configuration).

### What the script does not do

- It does not replace an enterprise-grade dedicated WAF (ModSecurity, Coraza, a cloud WAF): the signatures are deliberately simple (Lua pattern matching, not a real PCRE rules engine).
- It does not analyze HTTP headers one by one beyond the User-Agent.
- It does not decode multipart/form-data bodies nor compressed (gzip) bodies — inspection operates on the raw text received.
- It never inspects the body of a request other than POST.
- It performs no command-injection detection on the URL path (deliberate choice, see [Known limitations](#known-limitations-and-out-of-scope)).
- It provides no administration interface: configuration is done by editing the variables at the top of the file, the blocklist and whitelist via plain text files.
- It does not handle its own log rotation.

### Usage warning

The provided signatures target common attack families but do not constitute exhaustive protection. A detection score set too low on a site with unusual traffic (technical search, documentation containing shell commands, etc.) can generate false positives — adjust `waf_score_threshold` and the path whitelist accordingly. Always test a signature or threshold change in a non-critical environment before deploying it to production.

---

## Installation

### Prerequisites

- A web server exposing a `mod_magnet`-compatible interface:
  - **lighttpd** with the `mod_magnet` module enabled (`server.modules += ("mod_magnet")`), or
  - **omega-serv** with the `magnet` extra installed (`lupa` dependency — see below and the omega-serv guide, WAF Magnet section).
- An embedded Lua interpreter provided by the host (lighttpd embeds its own Lua; omega-serv embeds `lupa`, no system `lua` interpreter required in either case — `lupa` ships as a precompiled wheel for all common architectures/Python versions, no local compilation or system Lua headers needed).

#### Installing omega-serv's `magnet` extra

**Always from the `.venv` already created by `omega-serv/install.sh`, never with a system `pip`** — on some distributions (notably Arch Linux), the bare `pip` command either doesn't exist (only the `python-pip` package provides it, and not always by default) or refuses to install anything outside a virtual environment (`error: externally-managed-environment`, PEP 668):

```bash
cd /path/to/omega-serv

# Option A - venv already activated in the current shell
source .venv/bin/activate
pip install -e ".[magnet]"

# Option B - without activating the venv (the most portable form: never
# depends on a 'pip' being present on the system PATH, since the venv's
# own python always ships its own)
.venv/bin/python -m pip install -e ".[magnet]"
```

If `install.sh` hasn't been run yet (no `.venv/` present), run it first — it creates the virtual environment and already installs omega-serv itself there; only the (optional) `magnet` extra then remains to be added with one of the two commands above.

### Setup

1. Copy `omega-waf.lua` to a location readable by the server (e.g. `secure/waf/scripts/omega-waf.lua` for an omega-serv-style layout, or any path for native lighttpd).
2. Create the directories/files expected by the default configuration (see [Configuration](#configuration)):
   ```bash
   mkdir -p secure/waf/magnet-rules.d var/log var/cache
   touch secure/waf/magnet-blocklist.txt
   ```
3. Reference the script from the server configuration:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/path/to/omega-waf.lua" )
     ```
   - **omega-serv**: enable WAF Magnet from the dedicated TUI screen (Active Security → WAF → Magnet), which points to the script and automatically injects the `omega_magnet_paths` table (see [Compatibility](#compatibility)).
4. Restart (not reload — a reload never reloads an already-persistent Lua script in memory) the server concerned.

---

## Configuration

All options are set by directly editing the local variables at the top of the file (`CONFIGURATION` section, lines ~24-70). No separate configuration file.

| Variable | Role | Default value |
|---|---|---|
| `blocklist_file` | Text file of permanently banned IPs (one IP per line, `#` for comments) | `secure/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Directory for external rules (contains `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Alert log (detections, refusals, overruns) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Dedicated log for blocklist additions | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | Persistence file for counters (rate limit / repeat offense), Lua table format | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Enables POST request body inspection | `true` |
| `enable_rate_limit` | Enables rate limiting (global + SearXNG) | `true` |
| `enable_auto_block` | Allows automatic blocklist additions | `true` |
| `use_json_logs` | Switches alert logs to JSON (otherwise readable text) | `false` |
| `waf_score_threshold` | Minimum cumulative score for a POST request to be refused | `5` |
| `body_max_inspect` | Maximum inspected body size (bytes) | `65536` (64 KB) |
| `chunk_size` | Body read chunk size | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | Quota and window (s) of global per-IP rate limiting | `60` requests / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | Quota and window (s) of rate limiting dedicated to SearXNG routes | `40` requests / `60` s |
| `searx_max_query_len` | Maximum accepted query-string size on SearXNG routes | `4096` characters |
| `suspicious_threshold` | Number of suspicious hits before a permanent ban | `3` |
| `suspicious_window` | Sliding window (s) over which suspicious hits are counted | `300` (5 min) |
| `blocklist_refresh_interval` | Interval (s) for re-reading the blocklist file | `5` |
| `whitelist_refresh_interval` | Interval (s) for re-reading the path whitelist | `15` |

Under omega-serv, the first 5 paths (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) are automatically replaced by those configured in the WAF Magnet screen (global `omega_magnet_paths` table injected beforehand) — the hardcoded values above only serve as a fallback if that table is absent.

> ⚠️ **These 5 default values are modeled on omega-serv's own directory convention** (`secure/waf/...`, `var/log/...`, `var/cache/...`, relative paths resolved against a project-style working directory) — they are not neutral. Under **native lighttpd**, where no equivalent directory convention exists, these relative paths resolve against the lighttpd process's working directory at runtime, which is rarely predictable. It is recommended to replace these 5 lines with absolute paths suited to your installation, for example:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> Don't forget to create these directories (and grant the write permissions needed by the user lighttpd runs as) before restarting.

---

## Decision pipeline

The script runs entirely inside a global `pcall`: any unexpected Lua error is logged and then answers `500`, never crashing the server process. The steps execute in this order, each able to stop processing with a final response:

1. **Blocklist** — if the IP is banned, the request is refused (`451`), unless the requested path is explicitly whitelisted (e.g. serves the error page itself).
2. **Sensitive paths** — if the path matches a monitored pattern (`.git`, `.env`, `wp-admin`...), an HTML warning page is returned (`403`) the first time; repeat offense within the suspicion window triggers a ban (`451`).
3. **Scanner User-Agent** — if the User-Agent matches a known scanning tool/bot, `403` the first time, `451` on repeat offense.
4. **Global rate limit** — if the global per-IP quota is exceeded, `429`.
5. **SearXNG policy** — for recognized search routes: dedicated rate limit (`429`), query-string size guard (`414`), NUL byte rejection (`451`), then immediate exit (`0`, continue) so legitimate searches never go through the aggressive checks below.
6. **Path protection** — SQLi and XSS lookup on the decoded path (`451` if detected); path traversal lookup (`451`). Deliberately **no** command-injection detection at this stage (see [Known limitations](#known-limitations-and-out-of-scope)).
7. **Lightweight query-string protection** (outside SearXNG) — path traversal and NUL byte (`451`).
8. **POST body inspection** — only if `enable_body_inspection` is active and the method is `POST`: an oversized body is refused directly (`413`); otherwise a cumulative score over SQLi/XSS/CMDi/traversal, refused (`451`) if the score reaches `waf_score_threshold`, with an immediate ban if the score reaches 8, otherwise a simple suspicious hit.
9. **No block** — the request continues normally (`0`).

---

## Detected signatures

Four pattern families (`patterns`), each entry being either a string (case-sensitive) or a table `{ pat = "pattern", ci = true }` (case-insensitive):

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, SQL command injections via semicolon (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: `<script>` and `<iframe>` tags, `on*=` attributes (event handlers), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: shell command chaining (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, variants with `|`), command substitution (`` `...` ``, `$(...)`), PHP tag (`<?php`) and execution functions (`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

Plus a dedicated list of **scanner/bot User-Agents** (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, SEO crawlers (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`), and generic patterns (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

URL decoding is applied recursively (up to 5 levels on the path, 2 levels on the query string) before inspection, to catch double-encoded payloads.

---

## Monitored sensitive paths

Patterns checked against the decoded path: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

For each match, the script attempts to determine whether a physical resource actually exists at that path (via `lighty.stat`, falling back to `lighty.c.stat` for compatibility) and reports it in the warning page and the log — useful to distinguish a blind scan from a path that genuinely exists on the server.

---

## Rate limiting and escalation

Two independent rate limiters, based on in-memory counters persisted to disk (simple sliding window, not a precise second-by-second sliding window):

- **Global**: key `g:<ip>`, `global_rate_max_req` requests per `global_rate_window` seconds, across all routes.
- **SearXNG**: key `sx:<ip>`, `searx_rate_max_req` requests per `searx_rate_window` seconds, only on recognized search routes.

Progressive escalation (`record_suspicious_hit`) is a mechanism separate from rate limiting: each non-critical detection (non-repeating sensitive path, scanner User-Agent, minor POST body hit) increments a per-IP counter (key `susp:<ip>`) over a `suspicious_window`-second window. Once `suspicious_threshold` hits are reached within that window, the IP is added to the permanent blocklist — avoids a hard ban on a single false positive while still banning repeat offenders.

---

## Persistence

| File | Content | Format |
|---|---|---|
| `blocklist_file` | One banned IP per line, preceded by a timestamped comment (`# date - reason`) | Plain text |
| `waf_rules_dir/whitelist_paths.txt` | One Lua pattern (`string.match`) per line, paths always allowed even for a banned IP | Plain text |
| `rate_store_file` | Rate limiting and repeat-offense counters (`{ count = N, first = timestamp }` per key) | Serialized Lua table (`return { ... }`, read back via `load()`) |
| `alerts_log` | One line per detection/refusal | Readable text, or JSON if `use_json_logs = true` |
| `blocklist_log` | One line per blocklist addition | Readable text |

The rate limiting file is never rewritten on every request (only at most every 300 seconds, during periodic cleanup) to avoid a disproportionate I/O cost. The blocklist and whitelist are cached in memory and re-read at most every `blocklist_refresh_interval`/`whitelist_refresh_interval` seconds — an addition triggered by the script itself updates the cache immediately, without waiting for this delay.

---

## HTTP response codes emitted

| Code | Meaning | Trigger |
|---|---|---|
| `0` | Continue normal processing (`mod_magnet` convention) | No detection |
| `403` | Access denied, first warning | Sensitive path detected (first time), scanner User-Agent (first time) |
| `413` | Payload too large | POST body exceeding `body_max_inspect` |
| `414` | URI too long | SearXNG query string exceeding `searx_max_query_len` |
| `429` | Too many requests | Global or SearXNG rate limit exceeded |
| `451` | Unavailable for legal reasons (used here as the ban code) | IP already banned, repeat offense after escalation, SQLi/XSS/traversal detected in path or query string, NUL byte, POST score ≥ threshold |
| `500` | Internal error | Unexpected Lua exception (caught by the global `pcall`) |

---

## Compatibility

The script works in **two contexts**, with no modification whatsoever:

- **Native lighttpd**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` are the native APIs of the `mod_magnet` module. The 5 configuration paths use their hardcoded default values directly.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): a Python-Lua bridge (`lupa`) faithfully reproduces the same `lighty.*` contract, with a **persistent** Lua runtime across requests (time-based caches and rate limiting therefore correctly survive from one request to the next, just like under lighttpd). The global `omega_magnet_paths` table is injected before execution from the WAF Magnet screen configuration, so the 5 paths point to the locations chosen in the interface rather than the hardcoded values.

No business-logic divergence between the two contexts — only the origin of the 5 configuration paths changes.

### What is NOT compatible as-is

The script depends entirely on the `lighty.*` contract (native `mod_magnet` API) — it does **not** run as-is on another server, even one offering some form of Lua scripting:

- **Apache** (`mod_lua`): different API, based on a request object `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — no `lighty.*` call would work.
- **nginx** (`lua-nginx-module` / OpenResty): different API (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), with an execution-phase model (`access_by_lua`, `content_by_lua`...) distinct from `mod_magnet`'s simple status-code return.
- **Traefik**: no native Lua scripting at all (written in Go) — only a Go plugin (Yaegi) could run custom logic, never Lua directly.
- **Caddy**: no official Lua module — would require a third-party plugin compiled via `xcaddy`, nothing equivalent available natively.

Making this script compatible with one of these servers would require writing a dedicated compatibility bridge translating its native API to `lighty.*` (exactly what `lua_waf_adapter.py` does for omega-serv) — a full engineering effort of its own, not a simple configuration tweak.

---

## Customization

- **Adapting to an application other than SearXNG**: modify `is_searxng_path()` (list of paths) and, if needed, remove pipeline section 3 entirely (dedicated policy) if the protected application doesn't need differentiated handling of searches.
- **Adding always-allowed paths**: edit `waf_rules_dir/whitelist_paths.txt` (one Lua pattern per line), or add a hardcoded exception in `is_whitelisted_path()`.
- **Adjusting aggressiveness**: `waf_score_threshold` (POST refusal threshold), `suspicious_threshold`/`suspicious_window` (speed of escalation to a permanent ban), `global_rate_max_req`/`global_rate_window` (general rate limit).
- **Switching logs to JSON** (SIEM/log shipper ingestion): `use_json_logs = true`.
- **Disabling a subsystem without touching the rest**: `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block` are independent.

---

## Known limitations and out of scope

- **No CMDi detection on the URL path**: deliberate choice to avoid false positives on paths containing `.php` (which would be confused with `<?php` if CMDi detection included the PHP tag on this surface). CMDi detection remains active on the POST body.
- **Signatures via Lua pattern matching, not a real PCRE engine**: attack variants not covered by the listed patterns will not be detected — this is not a replacement for a WAF based on exhaustive rule sets (e.g. ModSecurity CRS).
- **No multipart/form-data parsing**: a malicious payload encapsulated in a multipart form field can escape inspection if the patterns don't match the raw, undecoded stream.
- **No gzip body decompression**: a compressed body is not inspected in clear text.
- **Approximate rate limiting**: simplified sliding window (full reset when the window is exceeded), not a precise sliding-window algorithm (sliding log/token bucket).
- **No supervision interface**: all observation goes through directly reading the log files and the blocklist.

---

## Troubleshooting

- **The script never seems to run**: check that `mod_magnet` is properly loaded (lighttpd) or that the `magnet` extra is installed and WAF Magnet enabled (omega-serv); a configuration reload never suffices to load a modified Lua script, a full restart is required in both cases.
- **Rate limit counters seem to reset on every request**: a symptom of a non-persistent Lua runtime across requests (the script starts over with an empty `rate_store` each execution) — under omega-serv, check that the WAF Magnet screen reports an active persistent runtime.
- **A legitimate IP ends up banned**: check `blocklist_log` and `alerts_log` to identify the triggered rule, adjust the pattern or the corresponding threshold, then remove the IP's line in `blocklist_file` (the in-memory cache updates on the next refresh, at most `blocklist_refresh_interval` seconds after the file is modified).
- **Legitimate SearXNG searches are blocked**: check `is_searxng_path()` (unrecognized path) before adjusting the dedicated rate limit thresholds.
- **`pip install -e ".[magnet]"` fails with `pip: command not found` or `externally-managed-environment`**: the command was typed outside omega-serv's `.venv` (some distributions, notably Arch Linux, expose no directly usable system `pip`). Use `.venv/bin/python -m pip install -e ".[magnet]"` from omega-serv's directory instead — works regardless of whether `pip` is on the system `PATH` (see [Installing omega-serv's `magnet` extra](#installing-omega-servs-magnet-extra)).

---

## License

Omega-Waf is distributed under the **MIT** license. See the [`LICENSE`](LICENSE) file for the full text.

---

> **Omega-Waf — One script, one line of defense.**
>
> A mod_magnet application firewall in a single Lua file, with no dependency, compatible with lighttpd and omega-serv.
