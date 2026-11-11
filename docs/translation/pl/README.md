<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="512">
</div>

# 🔒 OMEGA-WAF

**Zapora aplikacyjna mod_magnet (samodzielny skrypt Lua)**

> Stworzony przez [kraynux](https://github.com/kraynux) dla [Omega-Server](https://kraynux.snake-mackarel.ts.net)   

Strona oficjalna: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/pl/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#zgodność)

🌐 Dokumentacja dostępna w **17 językach**.  
**Języki:**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](../pt-BR/README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](../it/README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](../de/README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](../nl/README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](../vi/README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](../id/README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](../ja/README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](../ko/README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](../hi/README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

**Omega-Waf** to pojedynczy skrypt Lua (`omega-waf.lua`), bez zewnętrznych zależności, przeznaczony do uruchamiania przy każdym żądaniu HTTP przez serwer WWW zgodny z `mod_magnet` (natywnie lighttpd albo omega-serv przez swój most zgodności `magnet`). Sprawdza ścieżkę, zapytanie oraz, przy POST, treść żądania pod kątem znanych sygnatur ataków (wstrzyknięcie SQL, XSS, wstrzyknięcie poleceń, przechodzenie katalogów), stosuje ograniczanie szybkości (rate limiting), zarządza trwałą listą blokad ze stopniową eskalacją i loguje każdą decyzję.

Żadnej instalacji w klasycznym sensie: jeden plik do umieszczenia, odwołanie do niego w konfiguracji serwera — i to wszystko.

## Spis treści

- [Prezentacja](#prezentacja)
- [Instalacja](#instalacja)
- [Konfiguracja](#konfiguracja)
- [Potok decyzyjny](#potok-decyzyjny)
- [Wykrywane sygnatury](#wykrywane-sygnatury)
- [Monitorowane wrażliwe ścieżki](#monitorowane-wrażliwe-ścieżki)
- [Ograniczanie szybkości i eskalacja](#ograniczanie-szybkości-i-eskalacja)
- [Trwałość danych](#trwałość-danych)
- [Zwracane kody odpowiedzi HTTP](#zwracane-kody-odpowiedzi-http)
- [Zgodność](#zgodność)
- [Dostosowanie](#dostosowanie)
- [Znane ograniczenia i elementy poza zakresem](#znane-ograniczenia-i-elementy-poza-zakresem)
- [Rozwiązywanie problemów](#rozwiązywanie-problemów)
- [Licencja](#licencja)

---

## Prezentacja

### Cele

- Filtrować złośliwe żądania jak najbliżej serwera, zanim dotrą do serwowanej aplikacji.
- Pozostać jednym, czytelnym plikiem, bez zewnętrznych zależności i bez etapu kompilacji.
- Nigdy nie położyć serwera WWW: każdy wewnętrzny błąd skryptu jest przechwytywany i logowany, a nie propagowany.
- Ograniczyć koszt CPU/pamięci na żądanie (pamięci podręczne z czasem ważności, odczyt treści porcjami, ograniczona głębokość dekodowania).
- Zapewnić stopniową eskalację (ostrzeżenie, a potem ban) zamiast natychmiastowego bana przy pierwszym drobnym incydencie.

### Co robi Omega-Waf

- Blokuje już zbanowane adresy IP (trwała lista blokad na dysku), z wyjątkiem dla jawnie dozwolonych ścieżek (CSS strony błędu itp.).
- Monitoruje zestaw wrażliwych ścieżek (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store` itp.) i odpowiada albo ostrzegawczą stroną HTML, albo bezpośrednim banem w przypadku recydywy.
- Wykrywa User-Agenty znane jako narzędzia skanujące lub niepożądane boty (sqlmap, nmap, nikto, dirbuster, ogólne crawlery...).
- Stosuje globalne ograniczanie szybkości na IP, niezależnie od trasy.
- Stosuje dedykowaną, łagodniejszą politykę na trasach wyszukiwarki SearXNG (pozwala uniknąć fałszywych alarmów przy uzasadnionych zapytaniach wyszukiwania zawierających słowa kluczowe, które mogłyby przypominać atak).
- Sprawdza ścieżkę i query-string (SQLi, XSS, przechodzenie katalogów, bajt NUL).
- Sprawdza treść żądań POST (SQLi, XSS, wstrzyknięcie poleceń, przechodzenie katalogów) z kumulatywnym systemem punktacji.
- Obsługuje stopniową eskalację: pojedyncze podejrzane zachowanie wywołuje alert lub jednorazową odmowę; recydywa w danym oknie czasowym wywołuje ban stały.
- Loguje każdą decyzję (alerty, dodania do listy blokad) w formacie tekstowym lub JSON.
- Działa zarówno w natywnym lighttpd (ścieżki zapisane na stałe), jak i wstrzyknięty do omega-serv (ścieżki otrzymywane dynamicznie z konfiguracji serwera).

### Czego skrypt nie robi

- Nie zastępuje dedykowanego WAF klasy korporacyjnej (ModSecurity, Coraza, WAF w chmurze): sygnatury są celowo proste (dopasowywanie wzorców Lua, a nie prawdziwy silnik reguł PCRE).
- Nie analizuje nagłówków HTTP jeden po drugim poza User-Agentem.
- Nie dekoduje treści multipart/form-data ani treści skompresowanych (gzip) — inspekcja dotyczy surowego otrzymanego tekstu.
- Nigdy nie sprawdza treści żądania innego niż POST.
- Nie wykrywa wstrzyknięcia poleceń w ścieżce URL (świadomy wybór, zobacz [Znane ograniczenia](#znane-ograniczenia-i-elementy-poza-zakresem)).
- Nie zapewnia żadnego interfejsu administracyjnego: konfiguracja polega na edycji zmiennych na początku pliku, a lista blokad i biała lista to zwykłe pliki tekstowe.
- Sam nie obsługuje rotacji swoich logów.

### Ostrzeżenie dotyczące użycia

Dostarczone sygnatury obejmują popularne rodziny ataków, ale nie stanowią wyczerpującej ochrony. Zbyt niski próg wykrywania na stronie o nietypowym ruchu (wyszukiwanie techniczne, dokumentacja zawierająca polecenia powłoki itp.) może generować fałszywe alarmy — dostosuj odpowiednio `waf_score_threshold` i białą listę ścieżek. Każdą zmianę sygnatury lub progu testuj zawsze w środowisku niekrytycznym, zanim wdrożysz ją produkcyjnie.

---

## Instalacja

### Wymagania wstępne

- Serwer WWW udostępniający interfejs zgodny z `mod_magnet`:
  - **lighttpd** z włączonym modułem `mod_magnet` (`server.modules += ("mod_magnet")`), lub
  - **omega-serv** z zainstalowanym dodatkiem `magnet` (zależność `lupa` — zobacz poniżej oraz przewodnik omega-serv, sekcja WAF Magnet).
- Wbudowany interpreter Lua dostarczany przez hosta (lighttpd ma własny wbudowany Lua; omega-serv ma wbudowaną `lupa` — w obu przypadkach systemowy interpreter `lua` nie jest potrzebny; `lupa` jest dystrybuowana jako prekompilowany pakiet wheel dla wszystkich popularnych architektur/wersji Pythona, bez lokalnej kompilacji i bez systemowych nagłówków Lua).

#### Instalacja dodatku `magnet` omega-serv

**Zawsze z poziomu `.venv` utworzonego już przez `omega-serv/install.sh`, nigdy systemowym `pip`** — w niektórych dystrybucjach (zwłaszcza Arch Linux) samo polecenie `pip` nie istnieje (instaluje je tylko `python-pip`, i nie zawsze domyślnie) albo odmawia instalacji czegokolwiek poza środowiskiem wirtualnym (`error: externally-managed-environment`, PEP 668):

```bash
cd /path/to/omega-serv

# Opcja A - venv już aktywowany w bieżącej powłoce
source .venv/bin/activate
pip install -e ".[magnet]"

# Opcja B - bez aktywowania venv (najbardziej przenośna: nigdy nie zależy
# od obecności 'pip' w systemowym PATH, ponieważ python z venv
# zawsze ma własny)
.venv/bin/python -m pip install -e ".[magnet]"
```

Jeśli `install.sh` nie został jeszcze uruchomiony (brak katalogu `.venv/`), uruchom go najpierw — tworzy środowisko wirtualne i od razu instaluje w nim sam omega-serv; pozostaje wtedy tylko dodać (opcjonalny) dodatek `magnet` jednym z dwóch powyższych poleceń.

### Wdrożenie

1. Skopiuj `omega-waf.lua` w miejsce czytelne dla serwera (np. `secure/waf/scripts/omega-waf.lua` przy strukturze katalogów w stylu omega-serv lub dowolna ścieżka w natywnym lighttpd).
2. Utwórz oczekiwane katalogi/pliki:
   - **omega-serv**: nic do zrobienia, serwer sam tworzy katalogi przy uruchomieniu (w tym `var/lib/waf/`, współdzielony z kontem usługi).
   - **natywny lighttpd**: najpierw zastąp 5 ścieżek ścieżkami bezwzględnymi (zobacz [Ścieżki w natywnym lighttpd](#ścieżki-w-natywnym-lighttpd)), następnie utwórz katalogi i nadaj kontu lighttpd prawo zapisu do nich (`www-data` w Debian/Ubuntu, `http` w Arch, `lighttpd` w Fedora/Alpine):
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. Odwołaj się do skryptu w konfiguracji serwera:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/path/to/omega-waf.lua" )
     ```
   - **omega-serv**: włącz WAF Magnet na dedykowanym ekranie TUI (Aktywne zabezpieczenia → WAF → Magnet), który wskazuje na skrypt i automatycznie wstrzykuje tabelę `omega_magnet_paths` (zobacz [Zgodność](#zgodność)).
4. Uruchom ponownie (nie przeładowuj — przeładowanie nigdy nie wczytuje ponownie skryptu Lua, który już jest w pamięci trwałej) dany serwer.

---

## Konfiguracja

Wszystkie opcje ustawia się, modyfikując bezpośrednio zmienne lokalne na początku pliku (sekcja `CONFIGURATION`, wiersze ~24-70). Nie ma osobnego pliku konfiguracyjnego.

| Zmienna | Rola | Wartość domyślna |
|---|---|---|
| `blocklist_file` | Plik tekstowy z adresami IP zbanowanymi na stałe (jeden IP na wiersz, `#` dla komentarza) — **zapisywany** przez skrypt | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Katalog reguł zewnętrznych (zawiera `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Log alertów (wykrycia, odmowy, przekroczenia) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Log poświęcony dodaniom do listy blokad | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | Plik utrwalający liczniki (limit szybkości / recydywa), format tabeli Lua | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Włącza inspekcję treści żądań POST | `true` |
| `enable_rate_limit` | Włącza ograniczanie szybkości (globalne + SearXNG) | `true` |
| `enable_auto_block` | Zezwala na automatyczne dodawanie do listy blokad | `true` |
| `never_auto_block` | Sieci **nigdy** nie banowane automatycznie (żądanie nadal jest odrzucane i logowane, unika się tylko bana); w omega-serv lista wstrzyknięta przez serwer (`omega_never_auto_block`) ma zawsze pierwszeństwo | lokalny host, Tailscale, sieci lokalne |
| `use_json_logs` | Przełącza logi alertów na JSON (w przeciwnym razie czytelny tekst) | `false` |
| `waf_score_threshold` | Minimalny skumulowany wynik, przy którym żądanie POST jest odrzucane | `5` |
| `body_max_inspect` | Maksymalny sprawdzany rozmiar treści (bajty) | `65536` (64 KB) |
| `chunk_size` | Rozmiar bloków odczytu treści | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | Limit i okno (s) globalnego ograniczania szybkości na IP | `60` żądań / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | Limit i okno (s) ograniczania szybkości dla tras SearXNG | `40` żądań / `60` s |
| `searx_max_query_len` | Maksymalny akceptowany rozmiar query-string na trasach SearXNG | `4096` znaków |
| `suspicious_threshold` | Liczba podejrzanych trafień przed banem stałym | `3` |
| `suspicious_window` | Okno przesuwne (s), w którym liczone są podejrzane trafienia | `300` (5 min) |
| `blocklist_refresh_interval` | Odstęp (s) ponownego odczytu pliku listy blokad | `5` |
| `whitelist_refresh_interval` | Odstęp (s) ponownego odczytu białej listy ścieżek | `15` |

W omega-serv pierwsze 5 ścieżek (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) jest automatycznie zastępowanych tymi skonfigurowanymi na ekranie WAF Magnet (globalna tabela `omega_magnet_paths` wstrzykiwana przed wykonaniem) — powyższe wartości zapisane na stałe służą wtedy tylko jako rezerwa, gdy tej tabeli brak.

> ℹ️ **Zmiana z 2026-10-18**: `blocklist_file` zmienia się z `secure/waf/magnet-blocklist.txt` na `var/lib/waf/magnet-blocklist.txt`. Skrypt **zapisuje** do tego pliku przy każdym banie, a usługa systemd omega-serv może zapisywać tylko pod `var/`: przy starej lokalizacji żaden automatyczny ban nigdy nie był rejestrowany, bez żadnego komunikatu. omega-serv przejmuje stary plik jednorazowo przy uruchomieniu (stary pozostaje na miejscu jako kopia zapasowa); własna ścieżka ustawiona na ekranie WAF Magnet nigdy nie jest zmieniana.

### Ścieżki w natywnym lighttpd

> ⚠️ **Te 5 wartości domyślnych jest wzorowanych na konwencji katalogów omega-serv** (`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, ścieżki względne rozwiązywane względem katalogu roboczego w stylu projektu) — nie są neutralne. W **natywnym lighttpd**, gdzie nie istnieje żadna równoważna konwencja katalogów, te ścieżki względne są rozwiązywane względem katalogu roboczego procesu lighttpd w czasie działania (często `/`), co rzadko jest przewidywalne. **Zastąp te 5 wierszy ścieżkami bezwzględnymi** dopasowanymi do twojej instalacji, na przykład:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> Przed ponownym uruchomieniem utwórz te katalogi (polecenia w kroku 2 sekcji [Wdrożenie](#wdrożenie)), przestrzegając tych uprawnień:
>
> | Plik | Dostęp wymagany dla konta lighttpd |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **zapis** (plik i katalog, który go zawiera) |
> | `waf_rules_dir` (i `whitelist_paths.txt`) | tylko odczyt |
>
> Niepowodzenie zapisu jest **ciche** — skrypt nigdy nie przerywa żądania —: ban, który nie pojawia się w `blocklist_file`, jest jedynym objawem. Sprawdź poleceniem `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` (dostosuj konto). Jeśli jednostka systemd lighttpd ogranicza zapisy (`ProtectSystem=strict`, `ReadWritePaths=`), dodaj do niej te katalogi.

---

## Potok decyzyjny

Skrypt wykonuje się w całości wewnątrz globalnego `pcall`: każdy nieprzewidziany błąd Lua jest logowany, a odpowiedzią jest `500`, bez wywracania procesu serwera. Etapy wykonują się w tej kolejności, a każdy z nich może zakończyć przetwarzanie odpowiedzią końcową:

1. **Lista blokad** — jeśli IP jest zbanowany, żądanie jest odrzucane (`451`), chyba że żądana ścieżka jest jawnie na białej liście (np. serwuje samą stronę błędu).
2. **Wrażliwe ścieżki** — jeśli ścieżka pasuje do monitorowanego wzorca (`.git`, `.env`, `wp-admin`...), za pierwszym razem zwracana jest ostrzegawcza strona HTML (`403`); recydywa w oknie podejrzeń wywołuje ban (`451`).
3. **User-Agent skanera** — jeśli User-Agent pasuje do znanego narzędzia skanującego/bota, `403` za pierwszym razem, `451` w przypadku recydywy.
4. **Globalny limit szybkości** — jeśli globalny limit na IP zostanie przekroczony, `429`.
5. **Polityka SearXNG** — dla rozpoznanych tras wyszukiwania: dedykowany limit szybkości (`429`), zabezpieczenie rozmiaru query-string (`414`), odrzucanie bajtów NUL (`451`), a następnie natychmiastowe wyjście (`0`, kontynuuj), aby uzasadnione wyszukiwanie nigdy nie trafiło do kolejnych, agresywnych kontroli.
6. **Ochrona ścieżki** — wyszukiwanie SQLi i XSS w zdekodowanej ścieżce (`451` w razie wykrycia); wyszukiwanie przechodzenia katalogów (`451`). Celowo **brak** wykrywania wstrzyknięcia poleceń na tym etapie (zobacz [Znane ograniczenia](#znane-ograniczenia-i-elementy-poza-zakresem)).
7. **Lekka ochrona query-string** (poza SearXNG) — przechodzenie katalogów i bajt NUL (`451`).
8. **Inspekcja treści POST** — tylko gdy `enable_body_inspection` jest aktywne, a metodą jest `POST`: zbyt duża treść jest od razu odrzucana (`413`); w przeciwnym razie skumulowany wynik dla SQLi/XSS/CMDi/przechodzenia katalogów, odmowa (`451`), jeśli wynik osiągnie `waf_score_threshold`, z natychmiastowym banem, jeśli wynik osiągnie 8, w przeciwnym razie zwykłe podejrzane trafienie.
9. **Brak blokady** — żądanie jest przetwarzane normalnie (`0`).

---

## Wykrywane sygnatury

Cztery rodziny wzorców (`patterns`), a każdy wpis to albo ciąg znaków (z rozróżnianiem wielkości liter), albo tabela `{ pat = "wzorzec", ci = true }` (bez rozróżniania wielkości liter):

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, wstrzyknięcia poleceń SQL przez średnik (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: znaczniki `<script>`, `<iframe>`, atrybuty `on*=` (procedury obsługi zdarzeń), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: łączenie poleceń powłoki (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, warianty z `|`), podstawianie poleceń (`` `...` ``, `$(...)`), znacznik PHP (`<?php`) i funkcje wykonujące (`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

Oraz dedykowana lista **User-Agentów skanerów/botów** (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, crawlery SEO (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`) oraz wzorce ogólne (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

Dekodowanie URL jest stosowane rekurencyjnie (do 5 poziomów dla ścieżki, 2 poziomy dla query-string) przed inspekcją, aby wychwycić podwójnie zakodowane payloady.

---

## Monitorowane wrażliwe ścieżki

Wzorce sprawdzane w zdekodowanej ścieżce: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

Przy każdym dopasowaniu skrypt próbuje ustalić, czy pod tą ścieżką naprawdę istnieje zasób fizyczny (przez `lighty.stat`, z rezerwą `lighty.c.stat` dla zgodności), i podaje to na stronie ostrzegawczej oraz w logu — przydatne do odróżnienia skanowania na ślepo od ścieżki, która rzeczywiście istnieje na serwerze.

---

## Ograniczanie szybkości i eskalacja

Dwa niezależne ograniczniki szybkości, oparte na licznikach w pamięci utrwalanych na dysku (proste okno przesuwne, a nie okno przesuwne precyzyjne co do sekundy):

- **Globalny**: klucz `g:<ip>`, `global_rate_max_req` żądań na `global_rate_window` sekund, niezależnie od trasy.
- **SearXNG**: klucz `sx:<ip>`, `searx_rate_max_req` żądań na `searx_rate_window` sekund, tylko na rozpoznanych trasach wyszukiwania.

Stopniowa eskalacja (`record_suspicious_hit`) to mechanizm oddzielny od ograniczania szybkości: każde niekrytyczne wykrycie (wrażliwa ścieżka bez recydywy, User-Agent skanera, drobne trafienie w treści POST) zwiększa licznik na IP (klucz `susp:<ip>`) w oknie `suspicious_window` sekund. Po osiągnięciu `suspicious_threshold` trafień w tym oknie IP zostaje dodany do stałej listy blokad — unika to natychmiastowego bana z powodu jednego fałszywego alarmu, a jednocześnie banuje recydywę.

**Sieci nigdy nie banowane**: adres z `never_auto_block` (lokalny host, Tailscale `100.64.0.0/10`, sieci lokalne IPv4/IPv6, łącznie z formą IPv4 mapowaną na IPv6) nigdy nie wywołuje eskalacji ani dodania do listy blokad; uniknięty ban jest odnotowywany wierszem `BAN EVITE` w `blocklist_log` (nigdy `AJOUT BLOCKLIST`, odczytywanym przez omega-jail). Porównanie CIDR w przenośnym Lua, od 5.1 do 5.5 i LuaJIT.

**Tryb log-only omega-serv**: gdy WAF omega-serv działa w trybie `log-only`, skrypt nie nakłada żadnej blokady (ani listy blokad, ani wiersza `AJOUT BLOCKLIST`, więc fail2ban nie blokuje w sieci); odnotowuje to wiersz `BAN EVITE`. Jeśli listy blokad nie da się zapisać, wiersz `ECHEC BLOCKLIST` zastępuje `AJOUT BLOCKLIST`.

---

## Trwałość danych

| Plik | Zawartość | Format |
|---|---|---|
| `blocklist_file` | Jeden zbanowany IP na wiersz, poprzedzony komentarzem ze znacznikiem czasu (`# data - powód`) | Zwykły tekst |
| `waf_rules_dir/whitelist_paths.txt` | Jeden wzorzec Lua (`string.match`) na wiersz, ścieżki zawsze dozwolone, nawet dla zbanowanego IP | Zwykły tekst |
| `rate_store_file` | Liczniki ograniczania szybkości i recydywy (`{ count = N, first = timestamp }` dla każdego klucza) | Zserializowana tabela Lua (`return { ... }`, odczytywana przez `load()`) |
| `alerts_log` | Jeden wiersz na wykrycie/odmowę | Czytelny tekst lub JSON, jeśli `use_json_logs = true` |
| `blocklist_log` | Jeden wiersz na dodanie do listy blokad | Czytelny tekst |

Plik ograniczania szybkości nigdy nie jest zapisywany na nowo przy każdym żądaniu (najwyżej co 300 sekund, podczas okresowego czyszczenia), aby uniknąć nieproporcjonalnego kosztu I/O. Lista blokad i biała lista są przechowywane w pamięci podręcznej i odczytywane ponownie najwyżej co `blocklist_refresh_interval`/`whitelist_refresh_interval` sekund — dodanie wywołane przez sam skrypt aktualizuje pamięć podręczną natychmiast, bez czekania na ten odstęp.

---

## Zwracane kody odpowiedzi HTTP

| Kod | Znaczenie | Wyzwalacz |
|---|---|---|
| `0` | Kontynuuje normalne przetwarzanie (konwencja `mod_magnet`) | Brak wykrycia |
| `403` | Odmowa dostępu, pierwsze ostrzeżenie | Wykryta wrażliwa ścieżka (pierwszy raz), User-Agent skanera (pierwszy raz) |
| `413` | Zbyt duża encja | Treść POST przekraczająca `body_max_inspect` |
| `414` | Zbyt długi URI | Query-string SearXNG przekraczający `searx_max_query_len` |
| `429` | Zbyt wiele żądań | Przekroczony globalny limit szybkości lub limit SearXNG |
| `451` | Niedostępne z powodów prawnych (używane tu jako kod bana) | IP już zbanowany, recydywa po eskalacji, SQLi/XSS/przechodzenie katalogów wykryte w ścieżce lub query-string, bajt NUL, wynik POST ≥ próg |
| `500` | Błąd wewnętrzny | Nieprzewidziany wyjątek Lua (przechwycony przez globalny `pcall`) |

---

## Zgodność

Skrypt działa w **dwóch kontekstach**, bez żadnych zmian:

- **natywny lighttpd**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` to natywne API modułu `mod_magnet`. 5 ścieżek konfiguracji używa bezpośrednio swoich wartości domyślnych zapisanych na stałe.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): most Python-Lua (`lupa`) wiernie odtwarza ten sam kontrakt `lighty.*`, z **trwałym** środowiskiem uruchomieniowym Lua między żądaniami (pamięci podręczne z czasem ważności i ograniczanie szybkości poprawnie przetrwają więc od jednego żądania do następnego, jak w lighttpd). Globalna tabela `omega_magnet_paths` jest wstrzykiwana przed wykonaniem z konfiguracji ekranu WAF Magnet, tak aby 5 ścieżek wskazywało na lokalizacje wybrane w interfejsie, a nie na wartości zapisane na stałe.

Żadnej rozbieżności logiki biznesowej między tymi dwoma kontekstami — zmienia się tylko pochodzenie 5 ścieżek konfiguracji.

### Co NIE jest zgodne bez zmian

Skrypt zależy całkowicie od kontraktu `lighty.*` (natywnego API `mod_magnet`) — **nie** działa bez zmian na innym serwerze, nawet jeśli ten oferuje jakąś formę skryptów Lua:

- **Apache** (`mod_lua`): inne API, oparte na obiekcie żądania `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — żadne wywołanie `lighty.*` by nie zadziałało.
- **nginx** (`lua-nginx-module` / OpenResty): inne API (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), z modelem faz wykonania (`access_by_lua`, `content_by_lua`...) odmiennym od prostego zwracania kodu w `mod_magnet`.
- **Traefik**: brak natywnych skryptów Lua (napisany w Go) — tylko wtyczka Go (Yaegi) pozwoliłaby wykonać własną logikę, nigdy bezpośrednio Lua.
- **Caddy**: brak oficjalnego modułu Lua — wymagałby zewnętrznej wtyczki skompilowanej przez `xcaddy`, nic równoważnego nie jest dostępne natywnie.

Uzyskanie zgodności tego skryptu z którymkolwiek z tych serwerów wymagałoby napisania dedykowanego mostu zgodności, tłumaczącego jego natywne API na `lighty.*` (dokładnie to, co robi `lua_waf_adapter.py` dla omega-serv) — to pełnoprawna praca inżynierska, a nie zwykła zmiana konfiguracji.

---

## Dostosowanie

- **Dostosowanie do aplikacji innej niż SearXNG**: zmodyfikuj `is_searxng_path()` (lista ścieżek) i w razie potrzeby całkowicie usuń sekcję 3 potoku (dedykowana polityka), jeśli chroniona aplikacja nie potrzebuje odrębnego traktowania wyszukiwań.
- **Dodanie ścieżek zawsze dozwolonych**: edytuj `waf_rules_dir/whitelist_paths.txt` (jeden wzorzec Lua na wiersz) albo dodaj wyjątek zapisany na stałe w `is_whitelisted_path()`.
- **Regulacja agresywności**: `waf_score_threshold` (próg odmowy POST), `suspicious_threshold`/`suspicious_window` (szybkość eskalacji do bana stałego), `global_rate_max_req`/`global_rate_window` (ogólny limit szybkości).
- **Przełączenie logów na JSON** (pobieranie przez SIEM/log shipper): `use_json_logs = true`.
- **Wyłączenie podsystemu bez naruszania reszty**: `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block` są niezależne.

---

## Znane ograniczenia i elementy poza zakresem

- **Brak wykrywania CMDi w ścieżce URL**: świadomy wybór, aby uniknąć fałszywych alarmów na ścieżkach zawierających `.php` (mylonych z `<?php`, gdyby wykrywanie CMDi obejmowało znacznik PHP na tej powierzchni). Wykrywanie CMDi pozostaje aktywne dla treści POST.
- **Sygnatury oparte na dopasowywaniu wzorców Lua, a nie prawdziwy silnik PCRE**: warianty ataków nieobjęte wymienionymi wzorcami nie zostaną wykryte — to nie jest zamiennik WAF opartego na wyczerpujących zestawach reguł (typu ModSecurity CRS).
- **Brak parsowania multipart/form-data**: złośliwy payload zamknięty w polu formularza multipart może uniknąć inspekcji, jeśli wzorce nie pasują do surowego, niezdekodowanego strumienia.
- **Brak dekompresji gzip treści**: skompresowana treść nie jest sprawdzana jako tekst jawny.
- **Przybliżone ograniczanie szybkości**: uproszczone okno przesuwne (pełny reset po przekroczeniu okna), a nie algorytm precyzyjnego okna przesuwnego (sliding log/token bucket).
- **Brak interfejsu nadzoru**: każda obserwacja odbywa się przez bezpośredni odczyt plików logów i listy blokad.

---

## Rozwiązywanie problemów

- **Skrypt wydaje się nigdy nie uruchamiać**: sprawdź, czy `mod_magnet` jest poprawnie załadowany (lighttpd) albo czy dodatek `magnet` jest zainstalowany, a WAF Magnet włączony (omega-serv); przeładowanie konfiguracji nigdy nie wystarcza do wczytania zmodyfikowanego skryptu Lua, w obu przypadkach potrzebne jest pełne ponowne uruchomienie.
- **Liczniki limitu szybkości wydają się zerować przy każdym żądaniu**: objaw środowiska uruchomieniowego Lua, które nie jest trwałe między żądaniami (skrypt zaczyna od pustego `rate_store` przy każdym wykonaniu) — w omega-serv sprawdź, czy ekran WAF Magnet zgłasza aktywne trwałe środowisko uruchomieniowe.
- **Uzasadniony IP zostaje zbanowany**: przejrzyj `blocklist_log` i `alerts_log`, aby ustalić, która reguła zadziałała, dostosuj odpowiedni wzorzec lub próg, a następnie usuń wiersz tego IP z `blocklist_file` (pamięć podręczna aktualizuje się przy następnym odczycie, najwyżej `blocklist_refresh_interval` sekund po modyfikacji pliku).
- **Uzasadnione wyszukiwania SearXNG są blokowane**: sprawdź `is_searxng_path()` (nierozpoznana ścieżka), zanim dostosujesz progi dedykowanego limitu szybkości.
- **`pip install -e ".[magnet]"` kończy się błędem `pip: nie znaleziono polecenia` lub `externally-managed-environment`**: polecenie zostało wpisane poza `.venv` omega-serv (niektóre dystrybucje, zwłaszcza Arch Linux, nie udostępniają żadnego systemowego `pip` nadającego się do bezpośredniego użycia). Użyj `.venv/bin/python -m pip install -e ".[magnet]"` z katalogu omega-serv — działa bez względu na to, czy `pip` jest obecny w systemowym `PATH` (zobacz [Instalacja dodatku `magnet` omega-serv](#instalacja-dodatku-magnet-omega-serv)).

---

## Licencja

Omega-Waf jest dystrybuowany na licencji **MIT**. Pełny tekst znajdziesz w pliku [`LICENSE`](../../../LICENSE).

---

> **Omega-Waf — Jeden skrypt, jedna linia obrony.**
>
> Zapora aplikacyjna mod_magnet w jednym pliku Lua, bez zależności, zgodna z lighttpd i omega-serv.
