<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - MIT 许可证（见 LICENSE 文件） -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="384">
</div>

# 🔒 OMEGA-WAF

**mod_magnet 应用层防火墙（独立 Lua 脚本）**

> 由 **kraynux** 为 **Omega-server** 打造
> [kraynux.snake-mackarel.ts.net](https://kraynux.snake-mackarel.ts.net)

官方页面：[OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#兼容性)

**语言：**
[Français](README.md) · [English](README.en.md) · [Español](README.es.md) · [Русский](README.ru.md) · [中文](README.zh-CN.md)

---

**Omega-Waf** 是一个单独的 Lua 脚本（`omega-waf.lua`），无任何外部依赖，设计用于在每次 HTTP 请求时由兼容 `mod_magnet` 的 Web 服务器执行（lighttpd 原生支持，或 omega-serv 通过其 `magnet` 兼容桥接）。它检查路径、请求本身，并在 POST 时检查请求体，以查找已知的攻击特征（SQL 注入、XSS、命令注入、目录穿越），应用速率限制，维护带有渐进式升级机制的持久化黑名单，并记录每一次决策。

没有传统意义上的"安装"：只需放置一个文件，并在服务器配置中引用它即可。

## 目录

- [概述](#概述)
- [安装](#安装)
- [配置](#配置)
- [决策流程](#决策流程)
- [可检测的特征](#可检测的特征)
- [监控的敏感路径](#监控的敏感路径)
- [速率限制与升级机制](#速率限制与升级机制)
- [持久化](#持久化)
- [发出的 HTTP 响应码](#发出的-http-响应码)
- [兼容性](#兼容性)
- [定制](#定制)
- [已知限制与范围之外](#已知限制与范围之外)
- [故障排查](#故障排查)
- [许可证](#许可证)

---

## 概述

### 目标

- 尽可能在靠近服务器的位置过滤恶意请求，在它们到达被保护的应用之前拦截。
- 保持单一、可读的文件，无外部依赖，无需构建步骤。
- 永不导致 Web 服务器崩溃：任何脚本内部错误都会被捕获并记录，而不会向外传播。
- 限制每个请求的 CPU/内存开销（基于时间的缓存、分块读取请求体、有限的解码深度）。
- 提供渐进式升级（先警告，再封禁），而不是在首次轻微事件时就直接封禁。

### Omega-Waf 的功能

- 拦截已被封禁的 IP 地址（持久化在磁盘上的黑名单），但对明确允许的路径有例外（如错误页面的 CSS 等）。
- 监控一组敏感路径（`.git`、`.env`、`wp-admin`、`phpmyadmin`、`.DS_Store` 等），并以 HTML 警告页面或在重复出现时直接封禁作为响应。
- 检测已知为扫描工具或不受欢迎机器人的 User-Agent（sqlmap、nmap、nikto、dirbuster、通用爬虫……）。
- 对所有路由应用基于 IP 的全局速率限制。
- 对 SearXNG 搜索引擎的路由应用专门的、更宽松的策略（避免对包含看似攻击关键词的合法搜索请求产生误报）。
- 检查路径和查询字符串（SQL 注入、XSS、目录穿越、NUL 字节）。
- 检查 POST 请求体（SQL 注入、XSS、命令注入、目录穿越），采用累加评分系统。
- 管理渐进式升级：孤立的可疑行为触发一次警告或单次拒绝；在时间窗口内的重复行为则触发永久封禁。
- 以文本或 JSON 格式记录每一次决策（警报、黑名单新增）。
- 既能在原生 lighttpd 中运行（硬编码路径），也能注入到 omega-serv 中运行（路径从服务器配置中动态获取）。

### 该脚本不做的事

- 它不能替代企业级专用 WAF（ModSecurity、Coraza、云端 WAF）：特征规则刻意保持简单（Lua 模式匹配，而非真正的 PCRE 规则引擎）。
- 除 User-Agent 外，它不会逐一分析 HTTP 头部。
- 它不解码 multipart/form-data 请求体，也不解压 gzip 压缩的请求体——检查仅针对接收到的原始文本。
- 它绝不检查除 POST 之外的请求体。
- 它不对 URL 路径执行任何命令注入检测（有意为之的选择，见[已知限制](#已知限制与范围之外)）。
- 它不提供任何管理界面：配置通过编辑文件头部的变量完成，黑名单和白名单则通过简单的文本文件管理。
- 它不自行管理日志轮转。

### 使用警告

所提供的特征规则针对常见的攻击类型，但并不构成详尽的防护。在流量特殊的站点上（技术性搜索、包含 shell 命令的文档等），检测评分阈值设置过低可能会产生误报——请相应调整 `waf_score_threshold` 和路径白名单。在部署到生产环境之前，务必先在非关键环境中测试特征规则或阈值的更改。

---

## 安装

### 前置条件

- 一个暴露 `mod_magnet` 兼容接口的 Web 服务器：
  - **lighttpd**，并启用 `mod_magnet` 模块（`server.modules += ("mod_magnet")`），或
  - **omega-serv**，并安装 `magnet` 附加组件（依赖 `lupa` —— 见下文及 omega-serv 指南的 WAF Magnet 章节）。
- 由宿主提供的内置 Lua 解释器（lighttpd 内置自己的 Lua；omega-serv 内置 `lupa`，两种情况下都无需系统级 `lua` 解释器 —— `lupa` 以预编译 wheel 的形式分发，覆盖所有常见架构/Python 版本，无需本地编译，也无需系统级 Lua 头文件）。

#### 安装 omega-serv 的 `magnet` 附加组件

**务必在 `omega-serv/install.sh` 已创建的 `.venv` 内执行，切勿使用系统级 `pip`** —— 在某些发行版上（尤其是 Arch Linux），裸 `pip` 命令要么根本不存在（只有安装 `python-pip` 包才会提供，且并非默认安装），要么拒绝在虚拟环境之外安装任何东西（`error: externally-managed-environment`，PEP 668）：

```bash
cd /path/to/omega-serv

# 方案 A —— 当前 shell 中已激活 venv
source .venv/bin/activate
pip install -e ".[magnet]"

# 方案 B —— 不激活 venv（最具可移植性的写法：从不依赖系统 PATH 中
# 是否存在 'pip'，因为 venv 自带的 python 总是包含自己的 pip）
.venv/bin/python -m pip install -e ".[magnet]"
```

如果尚未运行过 `install.sh`（不存在 `.venv/`），请先运行它 —— 它会创建虚拟环境，并已在其中安装好 omega-serv 本身；之后只需用上面两条命令之一补充安装（可选的）`magnet` 附加组件即可。

### 部署步骤

1. 将 `omega-waf.lua` 复制到服务器可读取的位置（例如 omega-serv 风格目录结构下的 `secure/waf/scripts/omega-waf.lua`，或原生 lighttpd 下的任意路径）。
2. 创建默认配置所需的目录/文件（见[配置](#配置)）：
   ```bash
   mkdir -p secure/waf/magnet-rules.d var/log var/cache
   touch secure/waf/magnet-blocklist.txt
   ```
3. 在服务器配置中引用该脚本：
   - **lighttpd**：
     ```
     magnet.attract-physical-path-to = ( "/path/to/omega-waf.lua" )
     ```
   - **omega-serv**：从专用的 TUI 界面（主动安全 → WAF → Magnet）启用 WAF Magnet，它会指向该脚本并自动注入 `omega_magnet_paths` 表（见[兼容性](#兼容性)）。
4. 重启（而非重新加载——重新加载永远不会重新加载已经常驻内存的 Lua 脚本）相应的服务器。

---

## 配置

所有选项都通过直接修改文件头部的局部变量来设置（`CONFIGURATION` 部分，约第 24-70 行）。没有独立的配置文件。

| 变量 | 作用 | 默认值 |
|---|---|---|
| `blocklist_file` | 永久封禁 IP 的文本文件（每行一个 IP，`#` 表示注释） | `secure/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | 外部规则目录（包含 `whitelist_paths.txt`） | `secure/waf/magnet-rules.d/` |
| `alerts_log` | 警报日志（检测、拒绝、超限等） | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | 专门记录黑名单新增的日志 | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | 计数器持久化文件（速率限制/重犯计数），Lua 表格式 | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | 启用 POST 请求体检查 | `true` |
| `enable_rate_limit` | 启用速率限制（全局 + SearXNG） | `true` |
| `enable_auto_block` | 允许自动加入黑名单 | `true` |
| `use_json_logs` | 将警报日志切换为 JSON 格式（否则为可读文本） | `false` |
| `waf_score_threshold` | 拒绝 POST 请求所需的最低累加评分 | `5` |
| `body_max_inspect` | 检查的最大请求体大小（字节） | `65536`（64 KB） |
| `chunk_size` | 请求体读取的分块大小 | `8192`（8 KB） |
| `global_rate_max_req` / `global_rate_window` | 全局按 IP 速率限制的配额与窗口（秒） | `60` 次请求 / `60` 秒 |
| `searx_rate_max_req` / `searx_rate_window` | 专用于 SearXNG 路由的速率限制配额与窗口（秒） | `40` 次请求 / `60` 秒 |
| `searx_max_query_len` | SearXNG 路由上可接受的查询字符串最大长度 | `4096` 字符 |
| `suspicious_threshold` | 触发永久封禁前的可疑命中次数 | `3` |
| `suspicious_window` | 统计可疑命中的滑动窗口（秒） | `300`（5 分钟） |
| `blocklist_refresh_interval` | 重新读取黑名单文件的间隔（秒） | `5` |
| `whitelist_refresh_interval` | 重新读取路径白名单的间隔（秒） | `15` |

在 omega-serv 下，前 5 个路径（`blocklist_file`、`waf_rules_dir`、`alerts_log`、`blocklist_log`、`rate_store_file`）会自动替换为 WAF Magnet 界面中配置的路径（执行前注入的全局表 `omega_magnet_paths`）——上述硬编码值仅在该表不存在时作为后备使用。

> ⚠️ **这 5 个默认值是照搬 omega-serv 自身的目录约定而来的**（`secure/waf/...`、`var/log/...`、`var/cache/...`，相对于项目式工作目录解析的相对路径）——它们并非中立的。在**原生 lighttpd** 下，由于不存在等效的目录约定，这些相对路径会相对于 lighttpd 进程运行时的工作目录解析，而该目录往往难以预测。建议将这 5 行替换为适合你自身部署的绝对路径，例如：
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> 重启之前别忘了创建这些目录（并为运行 lighttpd 的用户授予必要的写入权限）。

---

## 决策流程

脚本整体运行在一个全局 `pcall` 中：任何未预期的 Lua 错误都会被记录，然后返回 `500`，绝不会使服务器进程崩溃。各步骤按以下顺序执行，每一步都可能以最终响应终止处理：

1. **黑名单** —— 若 IP 已被封禁，请求被拒绝（`451`），除非请求的路径明确在白名单中（例如用于提供错误页面本身）。
2. **敏感路径** —— 若路径匹配受监控的模式（`.git`、`.env`、`wp-admin`……），首次返回 HTML 警告页面（`403`）；在可疑窗口期内重复出现则触发封禁（`451`）。
3. **扫描器 User-Agent** —— 若 User-Agent 匹配已知的扫描工具/机器人，首次返回 `403`，重复出现则返回 `451`。
4. **全局速率限制** —— 若超过按 IP 的全局配额，返回 `429`。
5. **SearXNG 策略** —— 对于已识别的搜索路由：专用速率限制（`429`），查询字符串大小限制（`414`），拒绝 NUL 字节（`451`），随后立即退出（`0`，继续处理），确保合法搜索永远不会经过后续的激进检查。
6. **路径防护** —— 在解码后的路径中查找 SQL 注入和 XSS（检测到则返回 `451`）；查找目录穿越（`451`）。此阶段刻意**不**进行命令注入检测（见[已知限制](#已知限制与范围之外)）。
7. **查询字符串轻量防护**（SearXNG 之外）—— 目录穿越和 NUL 字节（`451`）。
8. **POST 请求体检查** —— 仅当 `enable_body_inspection` 启用且方法为 `POST` 时：请求体过大直接拒绝（`413`）；否则对 SQL 注入/XSS/命令注入/穿越进行累加评分，若评分达到 `waf_score_threshold` 则拒绝（`451`），若评分达到 8 则立即封禁，否则仅记为一次可疑命中。
9. **不阻断** —— 请求正常继续处理（`0`）。

---

## 可检测的特征

四类模式（`patterns`），每个条目要么是字符串（区分大小写），要么是表 `{ pat = "模式", ci = true }`（不区分大小写）：

- **`sqli_body`**：`UNION SELECT`、`SELECT ... FROM`、`INSERT INTO`、`DROP TABLE`、`OR 1=1`、`' OR '1'='1`、`SLEEP(...)`、`BENCHMARK(...)`，以及通过分号进行的 SQL 命令注入（`; DROP`、`; ALTER`、`; EXEC`、`; SHUTDOWN`）。
- **`xss_body`**：`<script>`、`<iframe>` 标签，`on*=` 属性（事件处理器），`javascript:`，`eval(...)`，`document.cookie`，`svg/onload`。
- **`cmdi_body`**：Shell 命令链（`; ls`、`; id`、`; whoami`、`; cat`、`; wget`、`; curl`、`; nc`、`; bash`、`; sh`、`; python`，以及使用 `|` 的变体），命令替换（`` `...` ``、`$(...)`），PHP 标签（`<?php`）以及执行函数（`system(`、`exec(`、`passthru(`）。
- **`path_traversal`**：`../`、`..\`、`/proc/self/`、`etc/passwd`、`etc/shadow`、`file://`、`php://filter`。

此外还有一份专门的**扫描器/机器人 User-Agent** 列表（`patterns_scanner_ua`）：`masscan`、`sqlmap`、`nmap`、`nikto`、`dirbuster`、`wfuzz`、`zgrab`、`python-requests`、`go-http-client`，SEO 爬虫（`ahrefsbot`、`mj12bot`、`semrushbot`、`blexbot`、`dotbot`、`spbot`、`yandexbot`、`sogoubot`），以及通用模式（`spider`、`crawler`、`bot`、`scraper`、`facebookexternalhit`、`lwp::simple`）。

在检查之前会递归应用 URL 解码（路径最多 5 层，查询字符串最多 2 层），以捕获二次编码的恶意载荷。

---

## 监控的敏感路径

针对解码后路径检查的模式：`.env`、`.git`、`.svn`、`.hg`、`.bzr`、`composer.json`/`composer.lock`、`vendor/`、`phpunit/`、`phpmyadmin`/`pma`/`adminer.php`、`wp-admin`/`wp-login.php`、`/server-status`、`.DS_Store`。

对于每一次匹配，脚本都会尝试判断该路径下是否真实存在物理资源（通过 `lighty.stat`，并为兼容性回退到 `lighty.c.stat`），并在警告页面和日志中标明——这有助于区分盲目扫描与服务器上真实存在的路径。

---

## 速率限制与升级机制

两个独立的速率限制器，基于持久化到磁盘的内存计数器（简单滑动窗口，而非精确到秒的滑动窗口）：

- **全局**：键 `g:<ip>`，每 `global_rate_window` 秒 `global_rate_max_req` 次请求，涵盖所有路由。
- **SearXNG**：键 `sx:<ip>`，每 `searx_rate_window` 秒 `searx_rate_max_req` 次请求，仅适用于已识别的搜索路由。

渐进式升级（`record_suspicious_hit`）是一个与速率限制分开的机制：每一次非关键检测（非重复的敏感路径、扫描器 User-Agent、POST 请求体中的轻微命中）都会在 `suspicious_window` 秒的窗口内，为该 IP（键 `susp:<ip>`）递增一次计数。一旦在该窗口内达到 `suspicious_threshold` 次命中，该 IP 将被加入永久黑名单——既避免了因单次误报而直接封禁，又能对重犯者进行封禁。

---

## 持久化

| 文件 | 内容 | 格式 |
|---|---|---|
| `blocklist_file` | 每行一个被封禁的 IP，前面带有带时间戳的注释（`# 日期 - 原因`） | 纯文本 |
| `waf_rules_dir/whitelist_paths.txt` | 每行一个 Lua 模式（`string.match`），即使 IP 被封禁也始终允许的路径 | 纯文本 |
| `rate_store_file` | 速率限制与重犯计数器（每个键对应 `{ count = N, first = timestamp }`） | 序列化的 Lua 表（`return { ... }`，通过 `load()` 重新读取） |
| `alerts_log` | 每次检测/拒绝一行 | 可读文本，或在 `use_json_logs = true` 时为 JSON |
| `blocklist_log` | 每次黑名单新增一行 | 可读文本 |

速率限制文件绝不会在每次请求时重写（仅在定期清理时，最多每 300 秒一次），以避免不成比例的 I/O 开销。黑名单和白名单会被缓存在内存中，并最多每 `blocklist_refresh_interval`/`whitelist_refresh_interval` 秒重新读取一次——由脚本自身触发的新增会立即更新缓存，无需等待该延迟。

---

## 发出的 HTTP 响应码

| 代码 | 含义 | 触发条件 |
|---|---|---|
| `0` | 继续正常处理（`mod_magnet` 约定） | 未检测到任何问题 |
| `403` | 访问被拒绝，首次警告 | 检测到敏感路径（首次）、扫描器 User-Agent（首次） |
| `413` | 实体过大 | POST 请求体超过 `body_max_inspect` |
| `414` | URI 过长 | SearXNG 查询字符串超过 `searx_max_query_len` |
| `429` | 请求过多 | 超过全局或 SearXNG 速率限制 |
| `451` | 因法律原因不可用（此处用作封禁代码） | IP 已被封禁、升级后的重犯、在路径或查询字符串中检测到 SQL 注入/XSS/穿越、NUL 字节、POST 评分 ≥ 阈值 |
| `500` | 内部错误 | 未预期的 Lua 异常（被全局 `pcall` 捕获） |

---

## 兼容性

该脚本可在**两种环境**下运行，无需任何修改：

- **原生 lighttpd**：`lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` 是 `mod_magnet` 模块的原生 API。5 个配置路径直接使用其硬编码的默认值。
- **omega-serv**（`infrastructure/waf/lua_waf_adapter.py`）：一个 Python-Lua 桥接层（`lupa`）忠实地重现了相同的 `lighty.*` 契约，并在请求之间保持**持久化**的 Lua 运行时（因此基于时间的缓存和速率限制能像在 lighttpd 下一样，正确地在请求之间延续）。全局表 `omega_magnet_paths` 在执行前从 WAF Magnet 界面配置中注入，使 5 个路径指向界面中选择的位置，而非硬编码值。

两种环境下业务逻辑没有任何差异——仅 5 个配置路径的来源不同。

### 哪些情况下不兼容

该脚本完全依赖于 `lighty.*` 契约（`mod_magnet` 的原生 API）——即使另一个服务器提供某种形式的 Lua 脚本支持，它也**不能**原样运行：

- **Apache**（`mod_lua`）：不同的 API，基于请求对象 `r`（`r.method`、`r.uri`、`r.headers_in`、`r:puts(...)`）——任何 `lighty.*` 调用都无法工作。
- **nginx**（`lua-nginx-module` / OpenResty）：不同的 API（`ngx.var`、`ngx.req`、`ngx.header`、`ngx.exit()`），且执行阶段模型（`access_by_lua`、`content_by_lua`……）与 `mod_magnet` 简单的状态码返回方式不同。
- **Traefik**：完全没有原生 Lua 脚本支持（用 Go 编写）——只能通过 Go 插件（Yaegi）执行自定义逻辑，永远无法直接运行 Lua。
- **Caddy**：没有官方的 Lua 模块——需要通过 `xcaddy` 编译第三方插件，原生没有任何等效方案。

要让此脚本兼容其中任何一个服务器，都需要编写一个专门的兼容性桥接层，将其原生 API 转换为 `lighty.*`（正是 `lua_waf_adapter.py` 为 omega-serv 所做的事）——这是一项独立的工程工作，而非简单的配置调整。

---

## 定制

- **适配 SearXNG 以外的应用**：修改 `is_searxng_path()`（路径列表），如有需要，若被保护的应用不需要对搜索进行差异化处理，可完全移除流程第 3 部分（专用策略）。
- **添加始终允许的路径**：编辑 `waf_rules_dir/whitelist_paths.txt`（每行一个 Lua 模式），或在 `is_whitelisted_path()` 中添加硬编码例外。
- **调整严格程度**：`waf_score_threshold`（POST 拒绝阈值）、`suspicious_threshold`/`suspicious_window`（升级为永久封禁的速度）、`global_rate_max_req`/`global_rate_window`（总体速率限制）。
- **将日志切换为 JSON**（供 SIEM/日志采集工具摄入）：`use_json_logs = true`。
- **在不影响其余部分的情况下禁用某个子系统**：`enable_body_inspection`、`enable_rate_limit`、`enable_auto_block` 彼此独立。

---

## 已知限制与范围之外

- **不对 URL 路径进行命令注入检测**：这是有意的选择，以避免对包含 `.php` 的路径产生误报（如果在此处的命令注入检测中包含了 PHP 标签，就会与 `<?php` 混淆）。命令注入检测在 POST 请求体中仍然有效。
- **基于 Lua 模式匹配的特征规则，而非真正的 PCRE 引擎**：未被列出模式覆盖的攻击变体将不会被检测到——这并非替代基于详尽规则集的 WAF（例如 ModSecurity CRS）。
- **不解析 multipart/form-data**：如果模式与未解码的原始数据流不匹配，封装在 multipart 表单字段中的恶意载荷可能逃过检查。
- **不解压请求体的 gzip 压缩**：压缩后的请求体不会以明文形式被检查。
- **近似的速率限制**：简化的滑动窗口（超过窗口时完全重置），而非精确的滑动窗口算法（滑动日志/令牌桶）。
- **没有监控界面**：所有观察都必须通过直接读取日志文件和黑名单来完成。

---

## 故障排查

- **脚本似乎从未执行**：检查 `mod_magnet` 是否已正确加载（lighttpd），或 `magnet` 附加组件是否已安装且 WAF Magnet 已启用（omega-serv）；仅重新加载配置永远不足以加载已修改的 Lua 脚本，两种情况下都需要完整重启。
- **速率限制计数器似乎每次请求都重置**：这是请求之间 Lua 运行时不持久的症状（脚本每次执行都从空的 `rate_store` 重新开始）——在 omega-serv 下，检查 WAF Magnet 界面是否报告持久化运行时处于激活状态。
- **合法 IP 被封禁**：查看 `blocklist_log` 和 `alerts_log` 以确定触发的规则，调整对应的模式或阈值，然后在 `blocklist_file` 中删除该 IP 所在行（内存缓存会在下次刷新时更新，最多在文件修改后 `blocklist_refresh_interval` 秒内生效）。
- **合法的 SearXNG 搜索被拦截**：在调整专用速率限制阈值之前，先检查 `is_searxng_path()`（路径未被识别）。
- **`pip install -e ".[magnet]"` 失败，提示 `pip: command not found` 或 `externally-managed-environment`**：该命令是在 omega-serv 的 `.venv` 之外执行的（某些发行版，尤其是 Arch Linux，并不直接暴露可用的系统级 `pip`）。请改为在 omega-serv 目录下使用 `.venv/bin/python -m pip install -e ".[magnet]"` —— 无论系统 `PATH` 中是否存在 `pip` 都能正常工作（参见[安装 omega-serv 的 `magnet` 附加组件](#安装-omega-serv-的-magnet-附加组件)）。

---

## 许可证

Omega-Waf 基于 **MIT** 许可证发布。完整文本请参见 [`LICENSE`](LICENSE) 文件。

---

> **Omega-Waf — 一个脚本，一道防线。**
>
> 一个单文件、无依赖的 mod_magnet 应用层防火墙，兼容 lighttpd 与 omega-serv。
