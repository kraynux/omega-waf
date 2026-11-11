<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="512">
</div>

# 🔒 OMEGA-WAF

**mod_magnet アプリケーションファイアウォール（単体の Lua スクリプト）**

> [kraynux](https://github.com/kraynux) が [Omega-Server](https://kraynux.snake-mackarel.ts.net) のために作成   

公式ページ：[OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/ja/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#互換性)

🌐 ドキュメントは **17 言語** で提供しています。  
**言語：**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](../pt-BR/README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](../it/README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](../de/README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](../nl/README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](../pl/README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](../vi/README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](../id/README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](../ko/README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](../hi/README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

**Omega-Waf** は外部依存のない単一の Lua スクリプト（`omega-waf.lua`）で、`mod_magnet` 互換の Web サーバー（ネイティブの lighttpd、または `magnet` 互換ブリッジを介した omega-serv）によって HTTP リクエストごとに実行されることを想定しています。パス、リクエスト、そして POST の場合はリクエストボディを検査して既知の攻撃シグネチャ（SQL インジェクション、XSS、コマンドインジェクション、ディレクトリトラバーサル）を探し、レート制限を適用し、段階的なエスカレーションを備えた永続的なブロックリストを管理し、すべての判定をログに記録します。

一般的な意味でのインストールはありません：ファイルを 1 つ置き、サーバーの設定から参照するだけです。

## 目次

- [概要](#概要)
- [インストール](#インストール)
- [設定](#設定)
- [判定パイプライン](#判定パイプライン)
- [検出するシグネチャ](#検出するシグネチャ)
- [監視する機微なパス](#監視する機微なパス)
- [レート制限とエスカレーション](#レート制限とエスカレーション)
- [永続化](#永続化)
- [返す HTTP 応答コード](#返す-http-応答コード)
- [互換性](#互換性)
- [カスタマイズ](#カスタマイズ)
- [既知の制限と対象範囲外](#既知の制限と対象範囲外)
- [トラブルシューティング](#トラブルシューティング)
- [ライセンス](#ライセンス)

---

## 概要

### 目的

- 悪意のあるリクエストを、配信するアプリケーションに届く前に、できるだけサーバーの近くでフィルタリングすること。
- 外部依存もビルド手順もない、読みやすい単一ファイルであり続けること。
- Web サーバーを決して落とさないこと：スクリプト内部のエラーはすべて捕捉してログに記録し、外へ伝播させません。
- リクエストあたりの CPU/メモリコストを抑えること（時間ベースのキャッシュ、ボディの分割読み込み、デコードの深さの上限）。
- 最初の軽微なインシデントでいきなり BAN するのではなく、段階的なエスカレーション（警告、その後 BAN）を提供すること。

### Omega-Waf が行うこと

- すでに BAN された IP アドレスをブロックします（ディスク上の永続的なブロックリスト）。ただし明示的に許可されたパス（エラーページの CSS など）は例外です。
- 一連の機微なパス（`.git`、`.env`、`wp-admin`、`phpmyadmin`、`.DS_Store` など）を監視し、HTML の警告ページを返すか、再犯の場合は直接 BAN します。
- スキャンツールや望ましくないボットとして知られる User-Agent を検出します（sqlmap、nmap、nikto、dirbuster、汎用クローラーなど）。
- すべてのルートを対象に、IP ごとのグローバルなレート制限を適用します。
- SearXNG 検索エンジンのルートには、より寛容な専用ポリシーを適用します（攻撃に見えかねないキーワードを含む正当な検索クエリでの誤検知を避けるため）。
- パスとクエリ文字列を検査します（SQLi、XSS、ディレクトリトラバーサル、NUL バイト）。
- POST リクエストのボディを累積スコア方式で検査します（SQLi、XSS、コマンドインジェクション、ディレクトリトラバーサル）。
- 段階的なエスカレーションを管理します：単発の不審な振る舞いはアラートまたは一回限りの拒否にとどまり、一定の時間枠内での再犯は恒久的な BAN につながります。
- すべての判定（アラート、ブロックリストへの追加）をテキストまたは JSON 形式でログに記録します。
- ネイティブの lighttpd（パスはハードコード）でも、omega-serv への組み込み（パスはサーバーの設定から動的に受け取る）でも同じように動作します。

### このスクリプトが行わないこと

- エンタープライズ級の専用 WAF（ModSecurity、Coraza、クラウド WAF）の代わりにはなりません：シグネチャは意図的に単純です（Lua のパターンマッチングであり、本物の PCRE ルールエンジンではありません）。
- User-Agent 以外の HTTP ヘッダーを 1 つずつ解析することはしません。
- multipart/form-data のボディや圧縮（gzip）されたボディはデコードしません — 検査は受信した生のテキストに対して行います。
- POST 以外のリクエストのボディは決して検査しません。
- URL のパスに対するコマンドインジェクションの検出は一切行いません（意図的な選択です。[既知の制限](#既知の制限と対象範囲外) を参照）。
- 管理インターフェースはありません：設定はファイル冒頭の変数を編集して行い、ブロックリストとホワイトリストは単純なテキストファイルで管理します。
- 自身のログのローテーションは行いません。

### 使用上の注意

付属のシグネチャは一般的な攻撃のファミリーを対象としていますが、網羅的な防御ではありません。通常とは異なるトラフィックのサイト（技術的な検索、シェルコマンドを含むドキュメントなど）で検出スコアを低くしすぎると、誤検知が発生することがあります — それに応じて `waf_score_threshold` とパスのホワイトリストを調整してください。シグネチャやしきい値の変更は、本番環境に展開する前に必ず重要でない環境でテストしてください。

---

## インストール

### 前提条件

- `mod_magnet` 互換のインターフェースを提供する Web サーバー：
  - `mod_magnet` モジュールを有効にした **lighttpd**（`server.modules += ("mod_magnet")`）、または
  - `magnet` エクストラをインストールした **omega-serv**（依存関係 `lupa` — 下記と omega-serv のガイドの WAF Magnet の節を参照）。
- ホストが提供する組み込みの Lua インタープリター（lighttpd は独自の Lua を組み込み、omega-serv は `lupa` を組み込みます。どちらの場合もシステムの `lua` インタープリターは不要です — `lupa` は一般的なすべてのアーキテクチャ/Python バージョン向けにビルド済みの wheel で配布されており、ローカルでのコンパイルもシステムの Lua ヘッダーも不要です）。

#### omega-serv の `magnet` エクストラをインストールする

**必ず `omega-serv/install.sh` が作成済みの `.venv` から行い、システムの `pip` は決して使わないでください** — 一部のディストリビューション（特に Arch Linux）では、素の `pip` コマンドが存在しない（`python-pip` だけがそれを提供し、デフォルトで入っているとは限りません）か、仮想環境の外には何もインストールしようとしません（`error: externally-managed-environment`、PEP 668）：

```bash
cd /path/to/omega-serv

# 方法 A - 現在のシェルで venv をすでに有効化している場合
source .venv/bin/activate
pip install -e ".[magnet]"

# 方法 B - venv を有効化しない場合（最も移植性が高い：システムの PATH に
# 'pip' があるかどうかに決して左右されない。venv の python は
# 常に自前の pip を備えているため）
.venv/bin/python -m pip install -e ".[magnet]"
```

`install.sh` をまだ実行していない場合（`.venv/` がない場合）は、先に実行してください — 仮想環境を作成し、そこに omega-serv 自体をすでにインストールします。あとは（任意の）`magnet` エクストラを上の 2 つのコマンドのどちらかで追加するだけです。

### セットアップ

1. `omega-waf.lua` をサーバーが読み取れる場所にコピーします（例：omega-serv 風のディレクトリ構成なら `secure/waf/scripts/omega-waf.lua`、ネイティブの lighttpd なら任意のパス）。
2. 必要なディレクトリ/ファイルを作成します：
   - **omega-serv**：何もする必要はありません。サーバーが起動時にディレクトリを自ら作成します（サービスアカウントと共有する `var/lib/waf/` を含む）。
   - **ネイティブの lighttpd**：まず 5 つのパスを絶対パスに置き換え（[ネイティブ lighttpd でのパス](#ネイティブ-lighttpd-でのパス) を参照）、次にディレクトリを作成して lighttpd のアカウント（Debian/Ubuntu では `www-data`、Arch では `http`、Fedora/Alpine では `lighttpd`）が書き込めるようにします：
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. サーバーの設定からスクリプトを参照します：
   - **lighttpd**：
     ```
     magnet.attract-physical-path-to = ( "/path/to/omega-waf.lua" )
     ```
   - **omega-serv**：専用の TUI 画面（アクティブセキュリティ → WAF → Magnet）から WAF Magnet を有効にします。この画面がスクリプトを指定し、`omega_magnet_paths` テーブルを自動で注入します（[互換性](#互換性) を参照）。
4. 対象のサーバーを再起動します（リロードではありません — リロードでは、すでにメモリ上に常駐している Lua スクリプトは決して読み直されません）。

---

## 設定

すべてのオプションは、ファイル冒頭のローカル変数（`CONFIGURATION` セクション、~24-70 行目）を直接編集して設定します。別の設定ファイルはありません。

| 変数 | 役割 | デフォルト値 |
|---|---|---|
| `blocklist_file` | 恒久的に BAN された IP のテキストファイル（1 行に 1 つの IP、`#` はコメント）— スクリプトが **書き込みます** | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | 外部ルールのディレクトリ（`whitelist_paths.txt` を含む） | `secure/waf/magnet-rules.d/` |
| `alerts_log` | アラートのログ（検出、拒否、超過） | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | ブロックリストへの追加専用のログ | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | カウンター（レート制限 / 再犯）の永続化ファイル、Lua テーブル形式 | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | POST リクエストのボディ検査を有効にする | `true` |
| `enable_rate_limit` | レート制限（グローバル + SearXNG）を有効にする | `true` |
| `enable_auto_block` | ブロックリストへの自動追加を許可する | `true` |
| `never_auto_block` | **決して** 自動 BAN されないネットワーク（リクエスト自体は拒否されログに記録され、BAN だけが回避されます）。omega-serv では、サーバーが注入するリスト（`omega_never_auto_block`）が常に優先されます | ローカルホスト、Tailscale、ローカルネットワーク |
| `use_json_logs` | アラートのログを JSON に切り替える（それ以外は読みやすいテキスト） | `false` |
| `waf_score_threshold` | POST リクエストが拒否される累積スコアの最小値 | `5` |
| `body_max_inspect` | 検査するボディの最大サイズ（バイト） | `65536`（64 KB） |
| `chunk_size` | ボディを読み込むブロックのサイズ | `8192`（8 KB） |
| `global_rate_max_req` / `global_rate_window` | IP ごとのグローバルなレート制限の上限と時間枠（秒） | `60` リクエスト / `60` 秒 |
| `searx_rate_max_req` / `searx_rate_window` | SearXNG のルート専用のレート制限の上限と時間枠（秒） | `40` リクエスト / `60` 秒 |
| `searx_max_query_len` | SearXNG のルートで受け付けるクエリ文字列の最大サイズ | `4096` 文字 |
| `suspicious_threshold` | 恒久的な BAN に至るまでの不審なヒットの回数 | `3` |
| `suspicious_window` | 不審なヒットを数えるスライディングウィンドウ（秒） | `300`（5 分） |
| `blocklist_refresh_interval` | ブロックリストファイルを読み直す間隔（秒） | `5` |
| `whitelist_refresh_interval` | パスのホワイトリストを読み直す間隔（秒） | `15` |

omega-serv では、最初の 5 つのパス（`blocklist_file`、`waf_rules_dir`、`alerts_log`、`blocklist_log`、`rate_store_file`）は WAF Magnet 画面で設定したパスに自動で置き換えられます（実行前に注入されるグローバルテーブル `omega_magnet_paths`）— 上記のハードコードされた値は、このテーブルがない場合の代替としてのみ使われます。

> ℹ️ **2026-10-18 の変更**：`blocklist_file` は `secure/waf/magnet-blocklist.txt` から `var/lib/waf/magnet-blocklist.txt` に移りました。スクリプトは BAN のたびにこのファイルに **書き込み**、omega-serv の systemd サービスは `var/` の下にしか書き込めません：以前の場所では、自動 BAN が一度も記録されず、何のメッセージも出ませんでした。omega-serv は起動時に一度だけ古いファイルを引き継ぎます（古いファイルはバックアップとしてそのまま残ります）。WAF Magnet 画面で設定したカスタムパスは決して変更されません。

### ネイティブ lighttpd でのパス

> ⚠️ **この 5 つのデフォルト値は omega-serv のディレクトリ規約に合わせたものです**（`var/lib/waf/...`、`secure/waf/...`、`var/log/...`、`var/cache/...`、プロジェクト型の作業ディレクトリを基準に解決される相対パス）— 中立ではありません。同等のディレクトリ規約が存在しない **ネイティブの lighttpd** では、これらの相対パスは実行時の lighttpd プロセスの作業ディレクトリ（多くの場合 `/`）を基準に解決され、ほとんど予測できません。**この 5 行を、お使いの環境に合った絶対パスに置き換えてください**。例：
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> 再起動の前に、次の権限を守ってこれらのディレクトリを作成してください（コマンドは [セットアップ](#セットアップ) の手順 2）：
>
> | ファイル | lighttpd のアカウントに必要なアクセス |
> |---|---|
> | `blocklist_file`、`alerts_log`、`blocklist_log`、`rate_store_file` | **書き込み**（ファイルとそれを含むディレクトリ） |
> | `waf_rules_dir`（と `whitelist_paths.txt`） | 読み取りのみ |
>
> 書き込みの失敗は **表に出ません** — スクリプトはリクエストを決して壊しません —：`blocklist_file` に現れない BAN が唯一の症状です。`sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` で確認してください（アカウントは適宜変更）。lighttpd の systemd ユニットが書き込みを制限している場合（`ProtectSystem=strict`、`ReadWritePaths=`）は、これらのディレクトリを追加してください。

---

## 判定パイプライン

スクリプト全体はグローバルな `pcall` の中で実行されます：予期しない Lua エラーはすべてログに記録されたうえで `500` を返し、サーバープロセスを落とすことは決してありません。各ステップは次の順序で実行され、どのステップも最終的な応答で処理を止めることができます：

1. **ブロックリスト** — IP が BAN されていればリクエストを拒否します（`451`）。ただし要求されたパスが明示的にホワイトリストに登録されている場合（たとえばエラーページ自体を配信する場合）は除きます。
2. **機微なパス** — パスが監視対象のパターン（`.git`、`.env`、`wp-admin`...）に一致すると、初回は HTML の警告ページを返します（`403`）。疑いの時間枠内での再犯は BAN につながります（`451`）。
3. **スキャナーの User-Agent** — User-Agent が既知のスキャンツール/ボットに一致すると、初回は `403`、再犯の場合は `451`。
4. **グローバルなレート制限** — IP ごとのグローバルな上限を超えると `429`。
5. **SearXNG ポリシー** — 認識された検索ルートに対して：専用のレート制限（`429`）、クエリ文字列のサイズの安全装置（`414`）、NUL バイトの拒否（`451`）、その後すぐに抜けます（`0`、続行）。正当な検索が以降の積極的なチェックを受けることは決してありません。
6. **パスの保護** — デコードしたパスで SQLi と XSS を探します（検出時は `451`）。ディレクトリトラバーサルも探します（`451`）。この段階では意図的にコマンドインジェクションの検出を **行いません**（[既知の制限](#既知の制限と対象範囲外) を参照）。
7. **クエリ文字列の軽い保護**（SearXNG 以外）— ディレクトリトラバーサルと NUL バイト（`451`）。
8. **POST ボディの検査** — `enable_body_inspection` が有効でメソッドが `POST` の場合のみ：大きすぎるボディは直接拒否します（`413`）。それ以外は SQLi/XSS/CMDi/トラバーサルの累積スコアを計算し、スコアが `waf_score_threshold` に達すると拒否（`451`）、スコアが 8 に達すると即時 BAN、それ未満なら単なる不審なヒットとなります。
9. **ブロックなし** — リクエストは通常どおり続行されます（`0`）。

---

## 検出するシグネチャ

4 つのパターンのファミリー（`patterns`）があり、各エントリは文字列（大文字と小文字を区別）か、テーブル `{ pat = "パターン", ci = true }`（大文字と小文字を区別しない）のいずれかです：

- **`sqli_body`**：`UNION SELECT`、`SELECT ... FROM`、`INSERT INTO`、`DROP TABLE`、`OR 1=1`、`' OR '1'='1`、`SLEEP(...)`、`BENCHMARK(...)`、セミコロンを使った SQL コマンドのインジェクション（`; DROP`、`; ALTER`、`; EXEC`、`; SHUTDOWN`）。
- **`xss_body`**：`<script>`、`<iframe>` タグ、`on*=` 属性（イベントハンドラー）、`javascript:`、`eval(...)`、`document.cookie`、`svg/onload`。
- **`cmdi_body`**：シェルコマンドの連結（`; ls`、`; id`、`; whoami`、`; cat`、`; wget`、`; curl`、`; nc`、`; bash`、`; sh`、`; python`、`|` を使った変種）、コマンド置換（`` `...` ``、`$(...)`）、PHP タグ（`<?php`）と実行関数（`system(`、`exec(`、`passthru(`）。
- **`path_traversal`**：`../`、`..\`、`/proc/self/`、`etc/passwd`、`etc/shadow`、`file://`、`php://filter`。

さらに、**スキャナー/ボットの User-Agent** 専用のリスト（`patterns_scanner_ua`）：`masscan`、`sqlmap`、`nmap`、`nikto`、`dirbuster`、`wfuzz`、`zgrab`、`python-requests`、`go-http-client`、SEO クローラー（`ahrefsbot`、`mj12bot`、`semrushbot`、`blexbot`、`dotbot`、`spbot`、`yandexbot`、`sogoubot`）、汎用パターン（`spider`、`crawler`、`bot`、`scraper`、`facebookexternalhit`、`lwp::simple`）。

二重にエンコードされたペイロードを捕まえるため、検査の前に URL デコードを再帰的に適用します（パスでは最大 5 段階、クエリ文字列では 2 段階）。

---

## 監視する機微なパス

デコードしたパスに対して確認するパターン：`.env`、`.git`、`.svn`、`.hg`、`.bzr`、`composer.json`/`composer.lock`、`vendor/`、`phpunit/`、`phpmyadmin`/`pma`/`adminer.php`、`wp-admin`/`wp-login.php`、`/server-status`、`.DS_Store`。

一致するたびに、スクリプトはそのパスに物理的なリソースが実際に存在するかどうかを判定しようとし（`lighty.stat` を使い、互換性のために `lighty.c.stat` へフォールバック）、警告ページとログにそれを示します — やみくもなスキャンと、サーバー上に実在するパスとを区別するのに役立ちます。

---

## レート制限とエスカレーション

ディスクに永続化されるメモリ上のカウンターに基づく、独立した 2 つのレート制限（単純なスライディングウィンドウで、秒単位の正確なスライディングウィンドウではありません）：

- **グローバル**：キー `g:<ip>`、`global_rate_window` 秒あたり `global_rate_max_req` リクエスト、すべてのルートが対象。
- **SearXNG**：キー `sx:<ip>`、`searx_rate_window` 秒あたり `searx_rate_max_req` リクエスト、認識された検索ルートのみが対象。

段階的なエスカレーション（`record_suspicious_hit`）はレート制限とは別の仕組みです：重大でない検出（再犯でない機微なパス、スキャナーの User-Agent、POST ボディでの軽微なヒット）のたびに、`suspicious_window` 秒の時間枠で IP ごとのカウンター（キー `susp:<ip>`）が増えます。この時間枠内で `suspicious_threshold` 回のヒットに達すると、その IP は恒久的なブロックリストに追加されます — 1 回の誤検知でいきなり BAN することを避けつつ、再犯は BAN します。

**決して BAN されないネットワーク**：`never_auto_block` に含まれるアドレス（ローカルホスト、Tailscale `100.64.0.0/10`、IPv4/IPv6 のローカルネットワーク、IPv4 射影 IPv6 形式を含む）は、エスカレーションもブロックリストへの追加も決して引き起こしません。回避された BAN は `blocklist_log` に `BAN EVITE` という行で記録されます（omega-jail が読み取る `AJOUT BLOCKLIST` には決してなりません）。CIDR の比較は移植性のある Lua で行い、5.1 から 5.5 と LuaJIT に対応します。

**omega-serv の log-only モード**：omega-serv の WAF が `log-only` のとき、スクリプトは BAN を一切設定しません（ブロックリストにも書かず、`AJOUT BLOCKLIST` 行も出さないため、fail2ban によるネットワーク BAN もありません）。`BAN EVITE` 行がこれを記録します。ブロックリストに書き込めない場合は、`AJOUT BLOCKLIST` の代わりに `ECHEC BLOCKLIST` 行が出ます。

---

## 永続化

| ファイル | 内容 | 形式 |
|---|---|---|
| `blocklist_file` | 1 行に 1 つの BAN された IP、その前にタイムスタンプ付きのコメント（`# date - 理由`） | プレーンテキスト |
| `waf_rules_dir/whitelist_paths.txt` | 1 行に 1 つの Lua パターン（`string.match`）、BAN された IP であっても常に許可されるパス | プレーンテキスト |
| `rate_store_file` | レート制限と再犯のカウンター（キーごとに `{ count = N, first = timestamp }`） | シリアライズされた Lua テーブル（`return { ... }`、`load()` で読み直し） |
| `alerts_log` | 検出/拒否ごとに 1 行 | 読みやすいテキスト、または `use_json_logs = true` なら JSON |
| `blocklist_log` | ブロックリストへの追加ごとに 1 行 | 読みやすいテキスト |

過大な I/O コストを避けるため、レート制限のファイルはリクエストのたびに書き直されることはありません（定期的なクリーンアップのときに、最大でも 300 秒に 1 回だけ）。ブロックリストとホワイトリストはメモリにキャッシュされ、読み直しは最大でも `blocklist_refresh_interval`/`whitelist_refresh_interval` 秒に 1 回です — スクリプト自身が行った追加は、この遅延を待たずにすぐキャッシュに反映されます。

---

## 返す HTTP 応答コード

| コード | 意味 | きっかけ |
|---|---|---|
| `0` | 通常の処理を続行（`mod_magnet` の規約） | 検出なし |
| `403` | アクセス拒否、最初の警告 | 機微なパスを検出（初回）、スキャナーの User-Agent（初回） |
| `413` | ペイロードが大きすぎる | `body_max_inspect` を超える POST ボディ |
| `414` | URI が長すぎる | `searx_max_query_len` を超える SearXNG のクエリ文字列 |
| `429` | リクエストが多すぎる | グローバルまたは SearXNG のレート制限を超過 |
| `451` | 法的理由により利用不可（ここでは BAN のコードとして使用） | すでに BAN された IP、エスカレーション後の再犯、パスまたはクエリ文字列で SQLi/XSS/トラバーサルを検出、NUL バイト、POST スコア ≥ しきい値 |
| `500` | 内部エラー | 予期しない Lua 例外（グローバルな `pcall` で捕捉） |

---

## 互換性

このスクリプトは、一切変更せずに **2 つの環境** で動作します：

- **ネイティブの lighttpd**：`lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` は `mod_magnet` モジュールのネイティブ API です。5 つの設定パスには、ハードコードされたデフォルト値がそのまま使われます。
- **omega-serv**（`infrastructure/waf/lua_waf_adapter.py`）：Python-Lua ブリッジ（`lupa`）が同じ `lighty.*` の契約を忠実に再現し、リクエスト間で **永続的な** Lua ランタイムを持ちます（そのため時間ベースのキャッシュとレート制限は、lighttpd と同じように、リクエストからリクエストへ正しく引き継がれます）。グローバルテーブル `omega_magnet_paths` は実行前に WAF Magnet 画面の設定から注入され、5 つのパスはハードコードされた値ではなく、インターフェースで選んだ場所を指します。

2 つの環境の間で業務ロジックに違いはありません — 変わるのは 5 つの設定パスの出どころだけです。

### そのままでは互換性がないもの

このスクリプトは `lighty.*` の契約（`mod_magnet` のネイティブ API）に完全に依存しています — 何らかの形で Lua スクリプトを扱えるサーバーであっても、他のサーバーではそのままでは **動きません**：

- **Apache**（`mod_lua`）：リクエストオブジェクト `r`（`r.method`、`r.uri`、`r.headers_in`、`r:puts(...)`）に基づく別の API — `lighty.*` の呼び出しは 1 つも動きません。
- **nginx**（`lua-nginx-module` / OpenResty）：別の API（`ngx.var`、`ngx.req`、`ngx.header`、`ngx.exit()`）で、実行フェーズのモデル（`access_by_lua`、`content_by_lua`...）は `mod_magnet` の単純なコードの返却とは異なります。
- **Traefik**：ネイティブの Lua スクリプトはまったくありません（Go で書かれています）— カスタムのロジックを実行できるのは Go プラグイン（Yaegi）だけで、Lua を直接実行することは決してできません。
- **Caddy**：公式の Lua モジュールはありません — `xcaddy` でコンパイルするサードパーティのプラグインが必要になり、ネイティブで利用できる同等のものはありません。

このスクリプトをこれらのサーバーのいずれかに対応させるには、そのネイティブ API を `lighty.*` に変換する専用の互換ブリッジを書く必要があります（まさに `lua_waf_adapter.py` が omega-serv のために行っていること）— 単なる設定の調整ではなく、それ自体が本格的なエンジニアリング作業です。

---

## カスタマイズ

- **SearXNG 以外のアプリケーションに合わせる**：`is_searxng_path()`（パスのリスト）を変更し、保護するアプリケーションが検索を区別して扱う必要がなければ、パイプラインのセクション 3（専用ポリシー）を丸ごと削除します。
- **常に許可するパスを追加する**：`waf_rules_dir/whitelist_paths.txt`（1 行に 1 つの Lua パターン）を編集するか、`is_whitelisted_path()` にハードコードの例外を追加します。
- **厳しさを調整する**：`waf_score_threshold`（POST の拒否しきい値）、`suspicious_threshold`/`suspicious_window`（恒久的な BAN へのエスカレーションの速さ）、`global_rate_max_req`/`global_rate_window`（全体のレート制限）。
- **ログを JSON に切り替える**（SIEM/ログシッパーへの取り込み）：`use_json_logs = true`。
- **他に触れずにサブシステムを無効にする**：`enable_body_inspection`、`enable_rate_limit`、`enable_auto_block` はそれぞれ独立しています。

---

## 既知の制限と対象範囲外

- **URL のパスに対する CMDi 検出はありません**：`.php` を含むパスでの誤検知を避けるための意図的な選択です（この部分で CMDi 検出が PHP タグを含むと、`<?php` と混同されます）。POST ボディでの CMDi 検出は有効なままです。
- **シグネチャは Lua のパターンマッチングで、本物の PCRE エンジンではありません**：列挙したパターンでカバーされない攻撃の変種は検出されません — 網羅的なルールセットに基づく WAF（ModSecurity の CRS など）の代わりにはなりません。
- **multipart/form-data の解析はありません**：multipart フォームのフィールドに包まれた悪意のあるペイロードは、パターンがデコードされていない生のストリームに一致しなければ、検査をすり抜けることがあります。
- **ボディの gzip 展開はありません**：圧縮されたボディは平文として検査されません。
- **おおよそのレート制限**：簡略化したスライディングウィンドウ（時間枠を超えると完全にリセット）であり、正確なスライディングウィンドウのアルゴリズム（sliding log/token bucket）ではありません。
- **監視インターフェースはありません**：観察はすべて、ログファイルとブロックリストを直接読むことで行います。

---

## トラブルシューティング

- **スクリプトが一度も実行されていないように見える**：`mod_magnet` が正しく読み込まれているか（lighttpd）、または `magnet` エクストラがインストールされ WAF Magnet が有効になっているか（omega-serv）を確認してください。どちらの場合も、変更した Lua スクリプトを読み込むには設定のリロードでは決して足りず、完全な再起動が必要です。
- **レート制限のカウンターがリクエストのたびにリセットされるように見える**：リクエスト間で Lua ランタイムが永続していないことの症状です（スクリプトは実行のたびに空の `rate_store` から始まります）— omega-serv では、WAF Magnet 画面が永続的なランタイムを有効と表示しているか確認してください。
- **正当な IP が BAN されてしまった**：`blocklist_log` と `alerts_log` を見て発動したルールを特定し、該当するパターンまたはしきい値を調整したうえで、`blocklist_file` からその IP の行を削除してください（メモリ上のキャッシュは次の読み直しで更新されます。ファイルの変更から最大でも `blocklist_refresh_interval` 秒後です）。
- **正当な SearXNG の検索がブロックされる**：専用のレート制限のしきい値を調整する前に、`is_searxng_path()` を確認してください（パスが認識されていない可能性があります）。
- **`pip install -e ".[magnet]"` が `pip: command not found` または `externally-managed-environment` で失敗する**：コマンドが omega-serv の `.venv` の外で入力されています（一部のディストリビューション、特に Arch Linux では、直接使えるシステムの `pip` がありません）。代わりに omega-serv のディレクトリから `.venv/bin/python -m pip install -e ".[magnet]"` を使ってください — システムの `PATH` に `pip` があるかどうかに関係なく動作します（[omega-serv の `magnet` エクストラをインストールする](#omega-serv-の-magnet-エクストラをインストールする) を参照）。

---

## ライセンス

Omega-Waf は **MIT** ライセンスで配布されています。全文は [`LICENSE`](../../../LICENSE) ファイルを参照してください。

---

> **Omega-Waf — 1 つのスクリプト、1 本の防衛線。**
>
> 依存関係のない単一の Lua ファイルで書かれた mod_magnet アプリケーションファイアウォール。lighttpd と omega-serv に対応しています。
