<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="512">
</div>

# 🔒 OMEGA-WAF

**mod_magnet 애플리케이션 방화벽(독립 실행형 Lua 스크립트)**

> [kraynux](https://github.com/kraynux)가 [Omega-Server](https://kraynux.snake-mackarel.ts.net)를 위해 제작   

공식 페이지: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/ko/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#호환성)

🌐 문서를 **17개 언어**로 제공합니다.  
**언어:**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](../pt-BR/README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](../it/README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](../de/README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](../nl/README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](../pl/README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](../vi/README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](../id/README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](../ja/README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](../hi/README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

**Omega-Waf**는 외부 의존성이 없는 단일 Lua 스크립트(`omega-waf.lua`)로, `mod_magnet` 호환 웹 서버(기본으로 지원하는 lighttpd, 또는 `magnet` 호환 브리지를 통한 omega-serv)가 HTTP 요청마다 실행하도록 만들어졌습니다. 경로와 요청, 그리고 POST의 경우 요청 본문을 검사해 알려진 공격 시그니처(SQL 인젝션, XSS, 명령 인젝션, 디렉터리 순회)를 찾고, 요청 제한을 적용하며, 단계적 에스컬레이션을 갖춘 영구 차단 목록을 관리하고, 모든 판정을 로그에 기록합니다.

일반적인 의미의 설치는 없습니다. 파일 하나를 두고 서버 설정에서 참조하면 끝입니다.

## 목차

- [개요](#개요)
- [설치](#설치)
- [설정](#설정)
- [판정 파이프라인](#판정-파이프라인)
- [탐지하는 시그니처](#탐지하는-시그니처)
- [감시하는 민감한 경로](#감시하는-민감한-경로)
- [요청 제한과 에스컬레이션](#요청-제한과-에스컬레이션)
- [영속성](#영속성)
- [반환하는 HTTP 응답 코드](#반환하는-http-응답-코드)
- [호환성](#호환성)
- [사용자 정의](#사용자-정의)
- [알려진 제한 및 범위 밖의 사항](#알려진-제한-및-범위-밖의-사항)
- [문제 해결](#문제-해결)
- [라이선스](#라이선스)

---

## 개요

### 목표

- 악성 요청을 제공 중인 애플리케이션에 닿기 전에, 서버에 최대한 가까운 곳에서 걸러냅니다.
- 외부 의존성도 빌드 단계도 없는, 읽기 쉬운 단일 파일로 유지합니다.
- 웹 서버를 절대 멈추게 하지 않습니다. 스크립트 내부 오류는 모두 잡아서 로그에 기록하며 밖으로 전파하지 않습니다.
- 요청당 CPU/메모리 비용을 제한합니다(시간 기반 캐시, 본문 분할 읽기, 제한된 디코딩 깊이).
- 첫 번째 사소한 사고에서 곧바로 차단하는 대신 단계적 에스컬레이션(경고 후 차단)을 제공합니다.

### Omega-Waf가 하는 일

- 이미 차단된 IP 주소를 막습니다(디스크에 저장되는 영구 차단 목록). 단, 명시적으로 허용된 경로(오류 페이지의 CSS 등)는 예외입니다.
- 일련의 민감한 경로(`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store` 등)를 감시하고, HTML 경고 페이지로 응답하거나 재범 시에는 곧바로 차단합니다.
- 스캔 도구나 원치 않는 봇으로 알려진 User-Agent를 탐지합니다(sqlmap, nmap, nikto, dirbuster, 일반 크롤러...).
- 모든 경로를 대상으로 IP별 전역 요청 제한을 적용합니다.
- SearXNG 검색 엔진의 경로에는 더 관대한 전용 정책을 적용합니다(공격처럼 보일 수 있는 키워드가 들어간 정당한 검색 요청에서 오탐을 피하기 위해).
- 경로와 쿼리 문자열을 검사합니다(SQLi, XSS, 디렉터리 순회, NUL 바이트).
- POST 요청의 본문을 누적 점수 방식으로 검사합니다(SQLi, XSS, 명령 인젝션, 디렉터리 순회).
- 단계적 에스컬레이션을 관리합니다. 한 번의 의심스러운 행동은 경고나 일회성 거부로 끝나고, 일정 시간 안의 재범은 영구 차단으로 이어집니다.
- 모든 판정(경고, 차단 목록 추가)을 텍스트 또는 JSON 형식으로 로그에 기록합니다.
- 네이티브 lighttpd(하드코딩된 경로)에서도, omega-serv에 주입된 상태(서버 설정에서 동적으로 받은 경로)에서도 똑같이 동작합니다.

### 스크립트가 하지 않는 일

- 엔터프라이즈급 전용 WAF(ModSecurity, Coraza, 클라우드 WAF)를 대신하지 않습니다. 시그니처는 의도적으로 단순합니다(Lua 패턴 매칭이며, 진짜 PCRE 규칙 엔진이 아닙니다).
- User-Agent 외의 HTTP 헤더를 하나하나 분석하지 않습니다.
- multipart/form-data 본문이나 압축된(gzip) 본문을 디코딩하지 않습니다 — 검사는 받은 그대로의 텍스트를 대상으로 합니다.
- POST가 아닌 요청의 본문은 절대 검사하지 않습니다.
- URL 경로에서는 명령 인젝션 탐지를 전혀 하지 않습니다(의도적인 선택, [알려진 제한](#알려진-제한-및-범위-밖의-사항) 참조).
- 관리 인터페이스를 제공하지 않습니다. 설정은 파일 앞부분의 변수를 편집해서 하고, 차단 목록과 허용 목록은 단순한 텍스트 파일로 관리합니다.
- 자체 로그의 순환(rotation)을 직접 처리하지 않습니다.

### 사용상 주의

제공되는 시그니처는 흔한 공격 유형을 겨냥하지만 완전한 보호를 이루지는 않습니다. 트래픽이 특이한 사이트(기술 검색, 셸 명령이 들어간 문서 등)에서 탐지 점수를 너무 낮게 잡으면 오탐이 생길 수 있습니다 — 그에 맞게 `waf_score_threshold`와 경로 허용 목록을 조정하십시오. 시그니처나 임계값을 바꿀 때는 운영 환경에 배포하기 전에 항상 중요하지 않은 환경에서 먼저 시험하십시오.

---

## 설치

### 사전 요구 사항

- `mod_magnet` 호환 인터페이스를 제공하는 웹 서버:
  - `mod_magnet` 모듈을 활성화한 **lighttpd**(`server.modules += ("mod_magnet")`), 또는
  - `magnet` 추가 항목을 설치한 **omega-serv**(의존성 `lupa` — 아래 내용과 omega-serv 가이드의 WAF Magnet 절 참조).
- 호스트가 제공하는 내장 Lua 인터프리터(lighttpd는 자체 Lua를 내장하고, omega-serv는 `lupa`를 내장합니다. 어느 경우에도 시스템 `lua` 인터프리터는 필요 없습니다 — `lupa`는 일반적인 모든 아키텍처/Python 버전용으로 미리 빌드된 wheel로 배포되므로, 로컬 컴파일도 시스템 Lua 헤더도 필요 없습니다).

#### omega-serv의 `magnet` 추가 항목 설치

**항상 `omega-serv/install.sh`가 이미 만든 `.venv`에서 설치하고, 시스템 `pip`는 절대 사용하지 마십시오** — 일부 배포판(특히 Arch Linux)에서는 그냥 `pip` 명령이 없거나(`python-pip`만 이를 제공하며, 기본 설치가 아닐 수도 있습니다) 가상 환경 밖에는 아무것도 설치하지 않으려 합니다(`error: externally-managed-environment`, PEP 668):

```bash
cd /path/to/omega-serv

# 방법 A - 현재 셸에서 venv가 이미 활성화된 경우
source .venv/bin/activate
pip install -e ".[magnet]"

# 방법 B - venv를 활성화하지 않는 경우(가장 이식성이 높음: 시스템 PATH에
# 'pip'가 있는지 여부에 결코 의존하지 않음. venv의 python은
# 항상 자체 pip를 갖추고 있기 때문)
.venv/bin/python -m pip install -e ".[magnet]"
```

`install.sh`를 아직 실행하지 않았다면(`.venv/`가 없다면) 먼저 실행하십시오 — 가상 환경을 만들고 그 안에 omega-serv 자체를 이미 설치합니다. 그러면 (선택 사항인) `magnet` 추가 항목만 위의 두 명령 중 하나로 더하면 됩니다.

### 배치

1. `omega-waf.lua`를 서버가 읽을 수 있는 위치에 복사합니다(예: omega-serv 방식의 디렉터리 구조라면 `secure/waf/scripts/omega-waf.lua`, 네이티브 lighttpd라면 아무 경로).
2. 필요한 디렉터리/파일을 만듭니다:
   - **omega-serv**: 할 일이 없습니다. 서버가 시작할 때 디렉터리를 직접 만듭니다(서비스 계정과 공유하는 `var/lib/waf/` 포함).
   - **네이티브 lighttpd**: 먼저 5개의 경로를 절대 경로로 바꾸고([네이티브 lighttpd에서의 경로](#네이티브-lighttpd에서의-경로) 참조), 디렉터리를 만든 다음 lighttpd 계정(Debian/Ubuntu는 `www-data`, Arch는 `http`, Fedora/Alpine은 `lighttpd`)이 쓸 수 있게 합니다:
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. 서버 설정에서 스크립트를 참조합니다:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/path/to/omega-waf.lua" )
     ```
   - **omega-serv**: 전용 TUI 화면(능동 보안 → WAF → Magnet)에서 WAF Magnet을 활성화합니다. 이 화면이 스크립트를 지정하고 `omega_magnet_paths` 테이블을 자동으로 주입합니다([호환성](#호환성) 참조).
4. 해당 서버를 다시 시작합니다(다시 불러오기가 아닙니다 — 다시 불러오기는 이미 메모리에 상주하는 Lua 스크립트를 절대 다시 읽지 않습니다).

---

## 설정

모든 옵션은 파일 앞부분의 지역 변수(`CONFIGURATION` 섹션, ~24-70행)를 직접 수정해 설정합니다. 별도의 설정 파일은 없습니다.

| 변수 | 역할 | 기본값 |
|---|---|---|
| `blocklist_file` | 영구 차단된 IP의 텍스트 파일(한 줄에 IP 하나, `#`는 주석) — 스크립트가 **씁니다** | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | 외부 규칙 디렉터리(`whitelist_paths.txt` 포함) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | 경고 로그(탐지, 거부, 초과) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | 차단 목록 추가 전용 로그 | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | 카운터(요청 제한 / 재범)의 영속화 파일, Lua 테이블 형식 | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | POST 요청 본문 검사를 활성화 | `true` |
| `enable_rate_limit` | 요청 제한(전역 + SearXNG)을 활성화 | `true` |
| `enable_auto_block` | 차단 목록 자동 추가를 허용 | `true` |
| `never_auto_block` | **절대** 자동 차단되지 않는 네트워크(요청 자체는 여전히 거부되고 로그에 기록되며, 차단만 건너뜁니다). omega-serv에서는 서버가 주입하는 목록(`omega_never_auto_block`)이 항상 우선합니다 | 로컬 호스트, Tailscale, 로컬 네트워크 |
| `use_json_logs` | 경고 로그를 JSON으로 전환(아니면 읽기 쉬운 텍스트) | `false` |
| `waf_score_threshold` | POST 요청이 거부되는 최소 누적 점수 | `5` |
| `body_max_inspect` | 검사하는 본문의 최대 크기(바이트) | `65536`(64 KB) |
| `chunk_size` | 본문 읽기 블록 크기 | `8192`(8 KB) |
| `global_rate_max_req` / `global_rate_window` | IP별 전역 요청 제한의 한도와 시간 창(초) | `60` 요청 / `60` 초 |
| `searx_rate_max_req` / `searx_rate_window` | SearXNG 경로 전용 요청 제한의 한도와 시간 창(초) | `40` 요청 / `60` 초 |
| `searx_max_query_len` | SearXNG 경로에서 받아들이는 쿼리 문자열의 최대 크기 | `4096` 자 |
| `suspicious_threshold` | 영구 차단 전까지의 의심 적중 횟수 | `3` |
| `suspicious_window` | 의심 적중을 세는 슬라이딩 시간 창(초) | `300`(5분) |
| `blocklist_refresh_interval` | 차단 목록 파일을 다시 읽는 간격(초) | `5` |
| `whitelist_refresh_interval` | 경로 허용 목록을 다시 읽는 간격(초) | `15` |

omega-serv에서는 처음 5개의 경로(`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`)가 WAF Magnet 화면에서 설정한 경로로 자동으로 대체됩니다(실행 전에 주입되는 전역 테이블 `omega_magnet_paths`) — 위의 하드코딩된 값은 이 테이블이 없을 때의 대체값으로만 쓰입니다.

> ℹ️ **2026-10-18 변경 사항**: `blocklist_file`이 `secure/waf/magnet-blocklist.txt`에서 `var/lib/waf/magnet-blocklist.txt`로 옮겨졌습니다. 스크립트는 차단할 때마다 이 파일에 **쓰고**, omega-serv의 systemd 서비스는 `var/` 아래에만 쓸 수 있습니다. 예전 위치에서는 자동 차단이 아무 메시지 없이 한 번도 기록되지 않았습니다. omega-serv는 시작할 때 예전 파일을 한 번만 가져옵니다(예전 파일은 백업으로 그대로 남습니다). WAF Magnet 화면에서 지정한 사용자 정의 경로는 절대 바뀌지 않습니다.

### 네이티브 lighttpd에서의 경로

> ⚠️ **이 5개의 기본값은 omega-serv의 디렉터리 규칙을 본뜬 것입니다**(`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, 프로젝트형 작업 디렉터리를 기준으로 해석되는 상대 경로) — 중립적이지 않습니다. 이에 상응하는 디렉터리 규칙이 없는 **네이티브 lighttpd**에서는 이 상대 경로가 실행 시점의 lighttpd 프로세스 작업 디렉터리(흔히 `/`)를 기준으로 해석되며, 이는 거의 예측할 수 없습니다. **이 5줄을 사용 중인 설치 환경에 맞는 절대 경로로 바꾸십시오**. 예:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> 다시 시작하기 전에 다음 권한을 지켜 이 디렉터리들을 만드십시오([배치](#배치)의 2단계 명령):
>
> | 파일 | lighttpd 계정에 필요한 접근 |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **쓰기**(파일과 그 파일이 있는 디렉터리) |
> | `waf_rules_dir`(및 `whitelist_paths.txt`) | 읽기 전용 |
>
> 쓰기 실패는 **조용히** 일어납니다 — 스크립트는 요청을 절대 깨뜨리지 않습니다 —. `blocklist_file`에 나타나지 않는 차단이 유일한 증상입니다. `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt`로 확인하십시오(계정은 환경에 맞게 변경). lighttpd의 systemd 유닛이 쓰기를 제한한다면(`ProtectSystem=strict`, `ReadWritePaths=`) 이 디렉터리들을 추가하십시오.

---

## 판정 파이프라인

스크립트 전체는 전역 `pcall` 안에서 실행됩니다. 예상치 못한 Lua 오류는 모두 로그에 기록된 뒤 `500`으로 응답하며, 서버 프로세스를 절대 중단시키지 않습니다. 각 단계는 다음 순서로 실행되고, 어느 단계든 최종 응답으로 처리를 멈출 수 있습니다:

1. **차단 목록** — IP가 차단되어 있으면 요청을 거부합니다(`451`). 단, 요청한 경로가 명시적으로 허용 목록에 있는 경우(예를 들어 오류 페이지 자체를 제공하는 경우)는 예외입니다.
2. **민감한 경로** — 경로가 감시 대상 패턴(`.git`, `.env`, `wp-admin`...)과 일치하면 처음에는 HTML 경고 페이지를 반환하고(`403`), 의심 시간 창 안의 재범은 차단으로 이어집니다(`451`).
3. **스캐너 User-Agent** — User-Agent가 알려진 스캔 도구/봇과 일치하면 처음에는 `403`, 재범 시에는 `451`.
4. **전역 요청 제한** — IP별 전역 한도를 넘으면 `429`.
5. **SearXNG 정책** — 인식된 검색 경로에 대해: 전용 요청 제한(`429`), 쿼리 문자열 크기 안전장치(`414`), NUL 바이트 거부(`451`), 그다음 즉시 빠져나옵니다(`0`, 계속). 정당한 검색이 이후의 공격적인 검사를 거치는 일은 절대 없습니다.
6. **경로 보호** — 디코딩한 경로에서 SQLi와 XSS를 찾고(탐지 시 `451`), 디렉터리 순회도 찾습니다(`451`). 이 단계에서는 의도적으로 명령 인젝션 탐지를 **하지 않습니다**([알려진 제한](#알려진-제한-및-범위-밖의-사항) 참조).
7. **쿼리 문자열의 가벼운 보호**(SearXNG 제외) — 디렉터리 순회와 NUL 바이트(`451`).
8. **POST 본문 검사** — `enable_body_inspection`이 켜져 있고 메서드가 `POST`일 때만: 너무 큰 본문은 바로 거부합니다(`413`). 그렇지 않으면 SQLi/XSS/CMDi/순회에 대한 누적 점수를 계산해, 점수가 `waf_score_threshold`에 이르면 거부(`451`), 8에 이르면 즉시 차단, 그 미만이면 단순한 의심 적중으로 처리합니다.
9. **차단 없음** — 요청이 정상적으로 계속됩니다(`0`).

---

## 탐지하는 시그니처

네 가지 패턴 계열(`patterns`)이 있으며, 각 항목은 문자열(대소문자 구분) 또는 테이블 `{ pat = "패턴", ci = true }`(대소문자 무시) 중 하나입니다:

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, 세미콜론을 이용한 SQL 명령 인젝션(`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: `<script>`, `<iframe>` 태그, `on*=` 속성(이벤트 핸들러), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: 셸 명령 연결(`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, `|`를 쓴 변형), 명령 치환(`` `...` ``, `$(...)`), PHP 태그(`<?php`)와 실행 함수(`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

그리고 **스캐너/봇 User-Agent** 전용 목록(`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, SEO 크롤러(`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`), 일반 패턴(`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

이중으로 인코딩된 페이로드를 잡기 위해, 검사 전에 URL 디코딩을 재귀적으로 적용합니다(경로는 최대 5단계, 쿼리 문자열은 2단계).

---

## 감시하는 민감한 경로

디코딩한 경로에서 확인하는 패턴: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

일치할 때마다 스크립트는 그 경로에 물리적 리소스가 실제로 있는지 확인하려 하고(`lighty.stat`를 사용하며, 호환성을 위해 `lighty.c.stat`로 대체), 그 결과를 경고 페이지와 로그에 표시합니다 — 무작위 스캔과 서버에 실제로 있는 경로를 구별하는 데 유용합니다.

---

## 요청 제한과 에스컬레이션

디스크에 영속화되는 메모리 카운터에 기반한 두 개의 독립적인 요청 제한기(단순한 슬라이딩 시간 창이며, 초 단위로 정밀한 슬라이딩 시간 창은 아닙니다):

- **전역**: 키 `g:<ip>`, `global_rate_window`초마다 `global_rate_max_req`개의 요청, 모든 경로 대상.
- **SearXNG**: 키 `sx:<ip>`, `searx_rate_window`초마다 `searx_rate_max_req`개의 요청, 인식된 검색 경로만 대상.

단계적 에스컬레이션(`record_suspicious_hit`)은 요청 제한과는 별개의 장치입니다. 치명적이지 않은 탐지(재범이 아닌 민감한 경로, 스캐너 User-Agent, POST 본문의 사소한 적중)가 있을 때마다 `suspicious_window`초의 시간 창 동안 IP별 카운터(키 `susp:<ip>`)가 늘어납니다. 이 시간 창 안에서 `suspicious_threshold`번의 적중에 이르면 해당 IP가 영구 차단 목록에 추가됩니다 — 오탐 한 번으로 곧바로 차단하는 일은 피하면서도 재범은 차단합니다.

**절대 차단되지 않는 네트워크**: `never_auto_block`에 속한 주소(로컬 호스트, Tailscale `100.64.0.0/10`, IPv4/IPv6 로컬 네트워크, IPv4 매핑 IPv6 형식 포함)는 에스컬레이션도 차단 목록 추가도 절대 일으키지 않습니다. 건너뛴 차단은 `blocklist_log`에 `BAN EVITE` 줄로 기록됩니다(omega-jail이 읽는 `AJOUT BLOCKLIST`로는 절대 기록되지 않습니다). CIDR 비교는 이식 가능한 Lua로 하며, 5.1부터 5.5까지와 LuaJIT를 지원합니다.

**omega-serv의 log-only 모드**: omega-serv의 WAF가 `log-only`일 때 스크립트는 어떤 차단도 설정하지 않습니다(차단 목록도, `AJOUT BLOCKLIST` 줄도 없으므로 fail2ban의 네트워크 차단도 없음). `BAN EVITE` 줄이 이를 기록합니다. 차단 목록을 쓸 수 없으면 `AJOUT BLOCKLIST` 대신 `ECHEC BLOCKLIST` 줄이 남습니다.

---

## 영속성

| 파일 | 내용 | 형식 |
|---|---|---|
| `blocklist_file` | 한 줄에 차단된 IP 하나, 앞에 타임스탬프가 붙은 주석(`# date - 사유`) | 일반 텍스트 |
| `waf_rules_dir/whitelist_paths.txt` | 한 줄에 Lua 패턴(`string.match`) 하나, 차단된 IP라도 항상 허용되는 경로 | 일반 텍스트 |
| `rate_store_file` | 요청 제한 및 재범 카운터(키마다 `{ count = N, first = timestamp }`) | 직렬화된 Lua 테이블(`return { ... }`, `load()`로 다시 읽음) |
| `alerts_log` | 탐지/거부마다 한 줄 | 읽기 쉬운 텍스트, 또는 `use_json_logs = true`이면 JSON |
| `blocklist_log` | 차단 목록 추가마다 한 줄 | 읽기 쉬운 텍스트 |

과도한 I/O 비용을 피하기 위해 요청 제한 파일은 요청마다 다시 쓰지 않습니다(주기적인 정리 때, 최대 300초에 한 번만). 차단 목록과 허용 목록은 메모리에 캐시되며, 최대 `blocklist_refresh_interval`/`whitelist_refresh_interval`초에 한 번만 다시 읽습니다 — 스크립트 자신이 일으킨 추가는 이 지연을 기다리지 않고 즉시 캐시에 반영됩니다.

---

## 반환하는 HTTP 응답 코드

| 코드 | 의미 | 발생 조건 |
|---|---|---|
| `0` | 정상 처리 계속(`mod_magnet` 규칙) | 탐지 없음 |
| `403` | 접근 거부, 첫 번째 경고 | 민감한 경로 탐지(처음), 스캐너 User-Agent(처음) |
| `413` | 페이로드가 너무 큼 | `body_max_inspect`를 넘는 POST 본문 |
| `414` | URI가 너무 김 | `searx_max_query_len`을 넘는 SearXNG 쿼리 문자열 |
| `429` | 요청이 너무 많음 | 전역 또는 SearXNG 요청 제한 초과 |
| `451` | 법적 사유로 이용 불가(여기서는 차단 코드로 사용) | 이미 차단된 IP, 에스컬레이션 후 재범, 경로나 쿼리 문자열에서 SQLi/XSS/순회 탐지, NUL 바이트, POST 점수 ≥ 임계값 |
| `500` | 내부 오류 | 예상치 못한 Lua 예외(전역 `pcall`이 잡음) |

---

## 호환성

이 스크립트는 아무것도 수정하지 않고 **두 가지 환경**에서 동작합니다:

- **네이티브 lighttpd**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat`는 `mod_magnet` 모듈의 네이티브 API입니다. 5개의 설정 경로는 하드코딩된 기본값을 그대로 사용합니다.
- **omega-serv**(`infrastructure/waf/lua_waf_adapter.py`): Python-Lua 브리지(`lupa`)가 같은 `lighty.*` 계약을 충실히 재현하며, 요청 사이에 **영속적인** Lua 런타임을 유지합니다(따라서 시간 기반 캐시와 요청 제한이 lighttpd에서처럼 요청에서 요청으로 올바르게 이어집니다). 전역 테이블 `omega_magnet_paths`는 실행 전에 WAF Magnet 화면의 설정에서 주입되므로, 5개의 경로는 하드코딩된 값이 아니라 인터페이스에서 선택한 위치를 가리킵니다.

두 환경 사이에 비즈니스 로직의 차이는 없습니다 — 달라지는 것은 5개 설정 경로의 출처뿐입니다.

### 그대로는 호환되지 않는 것

이 스크립트는 `lighty.*` 계약(`mod_magnet`의 네이티브 API)에 전적으로 의존합니다 — 어떤 형태로든 Lua 스크립팅을 제공하는 서버라도, 다른 서버에서는 그대로 **실행되지 않습니다**:

- **Apache**(`mod_lua`): 요청 객체 `r`(`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`)에 기반한 다른 API — `lighty.*` 호출은 하나도 동작하지 않습니다.
- **nginx**(`lua-nginx-module` / OpenResty): 다른 API(`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`)이며, 실행 단계 모델(`access_by_lua`, `content_by_lua`...)이 `mod_magnet`의 단순한 코드 반환과 다릅니다.
- **Traefik**: 네이티브 Lua 스크립팅이 전혀 없습니다(Go로 작성됨) — 사용자 정의 로직은 Go 플러그인(Yaegi)으로만 실행할 수 있으며, Lua를 직접 실행할 수는 결코 없습니다.
- **Caddy**: 공식 Lua 모듈이 없습니다 — `xcaddy`로 컴파일한 서드파티 플러그인이 필요하며, 기본으로 제공되는 동등한 기능은 없습니다.

이 서버들 중 하나와 호환되게 하려면 그 서버의 네이티브 API를 `lighty.*`로 옮겨 주는 전용 호환 브리지를 작성해야 합니다(바로 `lua_waf_adapter.py`가 omega-serv를 위해 하는 일) — 단순한 설정 조정이 아니라 그 자체로 온전한 엔지니어링 작업입니다.

---

## 사용자 정의

- **SearXNG가 아닌 다른 애플리케이션에 맞추기**: `is_searxng_path()`(경로 목록)를 수정하고, 보호하는 애플리케이션이 검색을 따로 처리할 필요가 없다면 파이프라인의 3번 섹션(전용 정책)을 통째로 제거합니다.
- **항상 허용할 경로 추가**: `waf_rules_dir/whitelist_paths.txt`(한 줄에 Lua 패턴 하나)를 편집하거나, `is_whitelisted_path()`에 하드코딩된 예외를 추가합니다.
- **엄격함 조정**: `waf_score_threshold`(POST 거부 임계값), `suspicious_threshold`/`suspicious_window`(영구 차단으로 가는 에스컬레이션 속도), `global_rate_max_req`/`global_rate_window`(전체 요청 제한).
- **로그를 JSON으로 전환**(SIEM/로그 전송기 수집용): `use_json_logs = true`.
- **나머지는 건드리지 않고 하위 시스템 끄기**: `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block`은 서로 독립적입니다.

---

## 알려진 제한 및 범위 밖의 사항

- **URL 경로에 대한 CMDi 탐지 없음**: `.php`가 들어간 경로에서 오탐을 피하기 위한 의도적인 선택입니다(이 부분에서 CMDi 탐지가 PHP 태그를 포함하면 `<?php`와 혼동됩니다). POST 본문에 대한 CMDi 탐지는 계속 켜져 있습니다.
- **시그니처는 Lua 패턴 매칭이며, 진짜 PCRE 엔진이 아닙니다**: 나열된 패턴이 다루지 않는 공격 변형은 탐지되지 않습니다 — 포괄적인 규칙 집합에 기반한 WAF(ModSecurity의 CRS 등)를 대신하지 않습니다.
- **multipart/form-data 파싱 없음**: multipart 폼 필드에 담긴 악성 페이로드는, 패턴이 디코딩되지 않은 원시 스트림과 일치하지 않으면 검사를 빠져나갈 수 있습니다.
- **본문 gzip 압축 해제 없음**: 압축된 본문은 평문으로 검사되지 않습니다.
- **근사적인 요청 제한**: 단순화한 슬라이딩 시간 창(시간 창을 넘으면 완전히 초기화)이며, 정밀한 슬라이딩 시간 창 알고리즘(sliding log/token bucket)이 아닙니다.
- **모니터링 인터페이스 없음**: 모든 관찰은 로그 파일과 차단 목록을 직접 읽어서 합니다.

---

## 문제 해결

- **스크립트가 전혀 실행되지 않는 것 같습니다**: `mod_magnet`이 제대로 로드되었는지(lighttpd), 또는 `magnet` 추가 항목이 설치되고 WAF Magnet이 활성화되었는지(omega-serv) 확인하십시오. 수정한 Lua 스크립트를 불러오는 데 설정 다시 불러오기로는 절대 충분하지 않으며, 두 경우 모두 완전히 다시 시작해야 합니다.
- **요청 제한 카운터가 요청마다 초기화되는 것 같습니다**: 요청 사이에 Lua 런타임이 영속되지 않는다는 증상입니다(스크립트가 실행될 때마다 빈 `rate_store`에서 다시 시작합니다) — omega-serv에서는 WAF Magnet 화면이 영속 런타임을 활성으로 표시하는지 확인하십시오.
- **정당한 IP가 차단되었습니다**: `blocklist_log`와 `alerts_log`를 보고 발동된 규칙을 확인한 뒤 해당 패턴이나 임계값을 조정하고, `blocklist_file`에서 그 IP의 줄을 지우십시오(메모리 캐시는 다음 다시 읽기 때 갱신되며, 파일 수정 후 최대 `blocklist_refresh_interval`초가 걸립니다).
- **정당한 SearXNG 검색이 막힙니다**: 전용 요청 제한 임계값을 조정하기 전에 `is_searxng_path()`를 확인하십시오(경로가 인식되지 않았을 수 있습니다).
- **`pip install -e ".[magnet]"`가 `pip: command not found` 또는 `externally-managed-environment`로 실패합니다**: 명령을 omega-serv의 `.venv` 밖에서 입력했습니다(일부 배포판, 특히 Arch Linux에는 바로 쓸 수 있는 시스템 `pip`가 없습니다). 대신 omega-serv 디렉터리에서 `.venv/bin/python -m pip install -e ".[magnet]"`를 사용하십시오 — 시스템 `PATH`에 `pip`가 있든 없든 동작합니다([omega-serv의 `magnet` 추가 항목 설치](#omega-serv의-magnet-추가-항목-설치) 참조).

---

## 라이선스

Omega-Waf는 **MIT** 라이선스로 배포됩니다. 전체 내용은 [`LICENSE`](../../../LICENSE) 파일을 참조하십시오.

---

> **Omega-Waf — 스크립트 하나, 방어선 하나.**
>
> 의존성 없이 단일 Lua 파일로 된 mod_magnet 애플리케이션 방화벽으로, lighttpd 및 omega-serv와 호환됩니다.
