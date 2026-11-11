<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="384">
</div>

# 🔒 OMEGA-WAF

**Applicatiefirewall voor mod_magnet (zelfstandig Lua-script)**

> Ontwikkeld door [kraynux](https://github.com/kraynux) voor [Omega-Server](https://kraynux.snake-mackarel.ts.net)   

Officiële pagina: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/nl/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#compatibiliteit)

🌐 Documentatie beschikbaar in **17 talen**.  
**Talen:**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](../pt-BR/README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](../it/README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](../de/README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](../pl/README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](../vi/README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](../id/README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](../ja/README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](../ko/README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](../hi/README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

**Omega-Waf** is één enkel Lua-script (`omega-waf.lua`), zonder externe afhankelijkheid, dat bij elk HTTP-verzoek wordt uitgevoerd door een webserver die compatibel is met `mod_magnet` (lighttpd native, of omega-serv via zijn compatibiliteitsbrug `magnet`). Het controleert het pad, het verzoek en, bij POST, de body van het verzoek op bekende aanvalssignaturen (SQL-injectie, XSS, commando-injectie, directory traversal), past snelheidsbeperking toe, beheert een persistente blokkeerlijst met geleidelijke escalatie en logt elke beslissing.

Geen installatie in de klassieke zin: één bestand neerzetten, ernaar verwijzen vanuit de serverconfiguratie, en klaar.

## Inhoudsopgave

- [Overzicht](#overzicht)
- [Installatie](#installatie)
- [Configuratie](#configuratie)
- [Beslissingspipeline](#beslissingspipeline)
- [Gedetecteerde signaturen](#gedetecteerde-signaturen)
- [Bewaakte gevoelige paden](#bewaakte-gevoelige-paden)
- [Snelheidsbeperking en escalatie](#snelheidsbeperking-en-escalatie)
- [Persistentie](#persistentie)
- [Verzonden HTTP-responscodes](#verzonden-http-responscodes)
- [Compatibiliteit](#compatibiliteit)
- [Aanpassing](#aanpassing)
- [Bekende beperkingen en buiten scope](#bekende-beperkingen-en-buiten-scope)
- [Probleemoplossing](#probleemoplossing)
- [Licentie](#licentie)

---

## Overzicht

### Doelen

- Kwaadaardige verzoeken zo dicht mogelijk bij de server filteren, voordat ze de geserveerde applicatie bereiken.
- Eén enkel, leesbaar bestand blijven, zonder externe afhankelijkheid of compilatiestap.
- De webserver nooit laten vallen: elke interne fout van het script wordt opgevangen en gelogd in plaats van doorgegeven.
- De CPU-/geheugenkosten per verzoek beperken (tijdgebonden caches, de body in blokken lezen, begrensde decodeerdiepte).
- Een geleidelijke escalatie bieden (waarschuwing, daarna ban) in plaats van een harde ban bij het eerste kleine incident.

### Wat Omega-Waf doet

- Blokkeert al gebande IP-adressen (persistente blokkeerlijst op schijf), met een uitzondering voor uitdrukkelijk toegestane paden (CSS van de foutpagina, enz.).
- Bewaakt een reeks gevoelige paden (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store`, enz.) en antwoordt met een HTML-waarschuwingspagina of, bij herhaling, met een directe ban.
- Detecteert User-Agents die bekendstaan als scantools of ongewenste bots (sqlmap, nmap, nikto, dirbuster, generieke crawlers...).
- Past een globale snelheidsbeperking per IP toe, over alle routes heen.
- Past op de routes van een SearXNG-zoekmachine een eigen, ruimer beleid toe (voorkomt valse positieven bij legitieme zoekopdrachten met trefwoorden die op een aanval zouden lijken).
- Controleert het pad en de query-string (SQLi, XSS, directory traversal, NUL-byte).
- Controleert de body van POST-verzoeken (SQLi, XSS, commando-injectie, directory traversal) met een cumulatief scoresysteem.
- Beheert een geleidelijke escalatie: een geïsoleerd verdacht gedrag leidt tot een waarschuwing of een eenmalige weigering; herhaling binnen een tijdvenster leidt tot een definitieve ban.
- Logt elke beslissing (waarschuwingen, toevoegingen aan de blokkeerlijst) in tekst- of JSON-formaat.
- Werkt zowel onder native lighttpd (hardgecodeerde paden) als ingebed in omega-serv (paden dynamisch ontvangen uit de serverconfiguratie).

### Wat het script niet doet

- Het vervangt geen specifieke WAF van ondernemingsniveau (ModSecurity, Coraza, een cloud-WAF): de signaturen zijn bewust eenvoudig (Lua-patroonherkenning, geen echte PCRE-regelengine).
- Het analyseert de HTTP-headers niet één voor één, behalve de User-Agent.
- Het decodeert geen multipart/form-data-bodies en geen gecomprimeerde bodies (gzip) — de controle gebeurt op de ontvangen ruwe tekst.
- Het controleert nooit de body van een ander verzoek dan POST.
- Het detecteert geen commando-injectie in het URL-pad (bewuste keuze, zie [Bekende beperkingen](#bekende-beperkingen-en-buiten-scope)).
- Het biedt geen beheerinterface: de configuratie gebeurt door de variabelen bovenaan het bestand te bewerken, de blokkeerlijst en de whitelist via eenvoudige tekstbestanden.
- Het regelt de rotatie van zijn logs niet zelf.

### Waarschuwing bij gebruik

De meegeleverde signaturen richten zich op gangbare aanvalsfamilies, maar bieden geen volledige bescherming. Een te lage detectiescore op een site met ongebruikelijk verkeer (technisch zoeken, documentatie met shell-commando's, enz.) kan valse positieven opleveren — pas `waf_score_threshold` en de whitelist van paden daarop aan. Test een wijziging van signatuur of drempel altijd eerst in een niet-kritieke omgeving voordat u die in productie uitrolt.

---

## Installatie

### Vereisten

- Een webserver met een interface die compatibel is met `mod_magnet`:
  - **lighttpd** met de module `mod_magnet` ingeschakeld (`server.modules += ("mod_magnet")`), of
  - **omega-serv** met de extra `magnet` geïnstalleerd (afhankelijkheid `lupa` — zie hieronder en de handleiding van omega-serv, sectie WAF Magnet).
- Een ingebedde Lua-interpreter die door de host wordt geleverd (lighttpd bevat zijn eigen Lua; omega-serv bevat `lupa`, in beide gevallen is geen systeeminterpreter `lua` nodig — `lupa` wordt verspreid als voorgecompileerde wheel voor alle gangbare architecturen/Python-versies, zonder lokale compilatie of Lua-systeemheaders).

#### De `magnet`-extra van omega-serv installeren

**Altijd vanuit de `.venv` die `omega-serv/install.sh` al heeft aangemaakt, nooit met een systeem-`pip`** — op sommige distributies (met name Arch Linux) bestaat het kale commando `pip` niet (alleen `python-pip` installeert het, en niet noodzakelijk standaard) of weigert het iets te installeren buiten een virtuele omgeving (`error: externally-managed-environment`, PEP 668):

```bash
cd /pad/naar/omega-serv

# Optie A - venv al geactiveerd in de huidige shell
source .venv/bin/activate
pip install -e ".[magnet]"

# Optie B - zonder de venv te activeren (het meest draagbaar: hangt nooit
# af van een 'pip' die wel of niet op het systeem-PATH staat, want de
# python van de venv brengt altijd zijn eigen mee)
.venv/bin/python -m pip install -e ".[magnet]"
```

Als `install.sh` nog niet is uitgevoerd (geen `.venv/` aanwezig), voer het dan eerst uit — het maakt de virtuele omgeving aan en installeert omega-serv zelf daar al in; daarna hoeft alleen nog de (optionele) extra `magnet` te worden toegevoegd met een van de twee bovenstaande commando's.

### Inrichting

1. Kopieer `omega-waf.lua` naar een locatie die de server kan lezen (bijv. `secure/waf/scripts/omega-waf.lua` voor een mappenstructuur in de stijl van omega-serv, of een willekeurig pad voor native lighttpd).
2. Maak de verwachte mappen/bestanden aan:
   - **omega-serv**: niets te doen, de server maakt de mappen zelf aan bij het starten (waaronder `var/lib/waf/`, gedeeld met het dienstaccount).
   - **native lighttpd**: vervang eerst de 5 paden door absolute paden (zie [Paden bij native lighttpd](#paden-bij-native-lighttpd)), maak daarna de mappen aan en maak ze schrijfbaar voor het lighttpd-account (`www-data` onder Debian/Ubuntu, `http` onder Arch, `lighttpd` onder Fedora/Alpine):
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. Verwijs naar het script vanuit de serverconfiguratie:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/pad/naar/omega-waf.lua" )
     ```
   - **omega-serv**: schakel WAF Magnet in vanuit het bijbehorende TUI-scherm (Actieve beveiliging → WAF → Magnet), dat naar het script verwijst en automatisch de tabel `omega_magnet_paths` injecteert (zie [Compatibiliteit](#compatibiliteit)).
4. Herstart de betreffende server (niet herladen — een herlading laadt een Lua-script dat al persistent in het geheugen staat nooit opnieuw).

---

## Configuratie

Alle opties worden ingesteld door de lokale variabelen bovenaan het bestand rechtstreeks te wijzigen (sectie `CONFIGURATION`, regels ~24-70). Geen apart configuratiebestand.

| Variabele | Rol | Standaardwaarde |
|---|---|---|
| `blocklist_file` | Tekstbestand van de definitief gebande IP's (één IP per regel, `#` voor commentaar) — wordt door het script **geschreven** | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Map met de externe regels (bevat `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Log van de waarschuwingen (detecties, weigeringen, overschrijdingen) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Eigen log voor toevoegingen aan de blokkeerlijst | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | Persistentiebestand van de tellers (snelheidsbeperking / herhaling), formaat Lua-tabel | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Schakelt de controle van de body van POST-verzoeken in | `true` |
| `enable_rate_limit` | Schakelt de snelheidsbeperking in (globaal + SearXNG) | `true` |
| `enable_auto_block` | Staat het automatisch toevoegen aan de blokkeerlijst toe | `true` |
| `never_auto_block` | Netwerken die **nooit** automatisch worden gebanned (het verzoek blijft geweigerd en gelogd, alleen de ban wordt vermeden); onder omega-serv heeft de door de server geïnjecteerde lijst (`omega_never_auto_block`) altijd voorrang | lokale host, Tailscale, lokale netwerken |
| `use_json_logs` | Zet de waarschuwingslogs om naar JSON (anders leesbare tekst) | `false` |
| `waf_score_threshold` | Minimale cumulatieve score waarbij een POST-verzoek wordt geweigerd | `5` |
| `body_max_inspect` | Maximale gecontroleerde grootte van de body (bytes) | `65536` (64 KB) |
| `chunk_size` | Grootte van de leesblokken van de body | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | Quotum en venster (s) van de globale snelheidsbeperking per IP | `60` verzoeken / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | Quotum en venster (s) van de eigen snelheidsbeperking voor SearXNG-routes | `40` verzoeken / `60` s |
| `searx_max_query_len` | Maximaal geaccepteerde lengte van de query-string op SearXNG-routes | `4096` tekens |
| `suspicious_threshold` | Aantal verdachte hits vóór een definitieve ban | `3` |
| `suspicious_window` | Glijdend venster (s) waarin verdachte hits worden geteld | `300` (5 min) |
| `blocklist_refresh_interval` | Interval (s) voor het opnieuw inlezen van het blokkeerlijstbestand | `5` |
| `whitelist_refresh_interval` | Interval (s) voor het opnieuw inlezen van de whitelist van paden | `15` |

Onder omega-serv worden de eerste 5 paden (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) automatisch vervangen door die welke in het scherm WAF Magnet zijn ingesteld (globale tabel `omega_magnet_paths`, vóór de uitvoering geïnjecteerd) — de hierboven hardgecodeerde waarden dienen dan alleen als terugval als die tabel ontbreekt.

> ℹ️ **Wijziging van 2026-10-18**: `blocklist_file` gaat van `secure/waf/magnet-blocklist.txt` naar `var/lib/waf/magnet-blocklist.txt`. Het script **schrijft** bij elke ban naar dit bestand, en de systemd-dienst van omega-serv mag alleen onder `var/` schrijven: op de oude locatie werd nooit een automatische ban vastgelegd, zonder enige melding. omega-serv neemt het oude bestand eenmalig over bij het starten (het oude blijft staan als back-up); een eigen pad in het scherm WAF Magnet wordt nooit gewijzigd.

### Paden bij native lighttpd

> ⚠️ **Deze 5 standaardwaarden volgen de mapconventie van omega-serv** (`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, relatieve paden die worden opgelost ten opzichte van een werkmap in projectstijl) — ze zijn niet neutraal. Onder **native lighttpd**, waar geen vergelijkbare mapconventie bestaat, worden deze relatieve paden tijdens runtime opgelost ten opzichte van de werkmap van het lighttpd-proces (vaak `/`), wat zelden voorspelbaar is. **Vervang deze 5 regels door absolute paden** die passen bij uw installatie, bijvoorbeeld:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> Maak deze mappen aan vóór het herstarten (commando's in stap 2 van de [Inrichting](#inrichting)) en houd daarbij rekening met deze rechten:
>
> | Bestand | Toegang die het lighttpd-account nodig heeft |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **schrijven** (het bestand en de map die het bevat) |
> | `waf_rules_dir` (en `whitelist_paths.txt`) | alleen lezen |
>
> Een schrijffout is **stil** — het script laat het verzoek nooit mislukken —: een ban die niet in `blocklist_file` verschijnt, is het enige symptoom. Controleer met `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` (pas het account aan). Als de systemd-unit van lighttpd schrijfacties beperkt (`ProtectSystem=strict`, `ReadWritePaths=`), voeg deze mappen daar dan aan toe.

---

## Beslissingspipeline

Het script draait volledig binnen een globale `pcall`: elke onvoorziene Lua-fout wordt gelogd en beantwoord met `500`, zonder ooit het serverproces te laten crashen. De stappen worden in deze volgorde uitgevoerd, en elke stap kan de verwerking beëindigen met een definitief antwoord:

1. **Blokkeerlijst** — als het IP gebanned is, wordt het verzoek geweigerd (`451`), tenzij het gevraagde pad uitdrukkelijk op de whitelist staat (serveert bijvoorbeeld de foutpagina zelf).
2. **Gevoelige paden** — als het pad overeenkomt met een bewaakt patroon (`.git`, `.env`, `wp-admin`...), wordt de eerste keer een HTML-waarschuwingspagina teruggestuurd (`403`); herhaling binnen het verdachtheidsvenster leidt tot een ban (`451`).
3. **Scanner-User-Agent** — als de User-Agent overeenkomt met een bekend scantool/bot, de eerste keer `403`, bij herhaling `451`.
4. **Globale snelheidsbeperking** — als het globale quotum per IP is overschreden, `429`.
5. **SearXNG-beleid** — voor herkende zoekroutes: eigen snelheidsbeperking (`429`), bescherming tegen te lange query-strings (`414`), weigering van NUL-bytes (`451`), daarna onmiddellijke uitgang (`0`, doorgaan) zodat een legitieme zoekopdracht nooit aan de volgende agressieve controles wordt onderworpen.
6. **Bescherming van het pad** — zoeken naar SQLi en XSS in het gedecodeerde pad (`451` bij detectie); zoeken naar directory traversal (`451`). In deze stap bewust **geen** detectie van commando-injectie (zie [Bekende beperkingen](#bekende-beperkingen-en-buiten-scope)).
7. **Lichte bescherming van de query-string** (behalve SearXNG) — directory traversal en NUL-byte (`451`).
8. **Controle van de POST-body** — alleen als `enable_body_inspection` actief is en de methode `POST` is: te grote body direct geweigerd (`413`); anders cumulatieve score over SQLi/XSS/CMDi/traversal, weigering (`451`) als de score `waf_score_threshold` bereikt, met onmiddellijke ban als de score 8 bereikt, anders alleen een verdachte hit.
9. **Geen blokkering** — het verzoek gaat normaal verder (`0`).

---

## Gedetecteerde signaturen

Vier patroonfamilies (`patterns`), waarbij elke invoer ofwel een string is (hoofdlettergevoelig), ofwel een tabel `{ pat = "patroon", ci = true }` (niet hoofdlettergevoelig):

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, SQL-commando-injecties via puntkomma (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: tags `<script>`, `<iframe>`, attributen `on*=` (event handlers), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: koppeling van shell-commando's (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, varianten met `|`), commandosubstitutie (`` `...` ``, `$(...)`), PHP-tag (`<?php`) en uitvoeringsfuncties (`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

En een eigen lijst van **User-Agents van scanners/bots** (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, SEO-crawlers (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`), en generieke patronen (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

URL-decodering wordt vóór de controle recursief toegepast (tot 5 niveaus op het pad, 2 niveaus op de query-string), om dubbel gecodeerde payloads te onderscheppen.

---

## Bewaakte gevoelige paden

Patronen die op het gedecodeerde pad worden gecontroleerd: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

Bij elke overeenkomst probeert het script vast te stellen of er op dat pad echt een fysieke bron bestaat (via `lighty.stat`, met terugval op `lighty.c.stat` voor compatibiliteit) en vermeldt dit op de waarschuwingspagina en in de log — handig om een blinde scan te onderscheiden van een pad dat echt op de server bestaat.

---

## Snelheidsbeperking en escalatie

Twee onafhankelijke snelheidsbegrenzers, gebaseerd op tellers in het geheugen die op schijf worden bewaard (eenvoudig glijdend venster, geen op de seconde nauwkeurig glijdend venster):

- **Globaal**: sleutel `g:<ip>`, `global_rate_max_req` verzoeken per `global_rate_window` seconden, over alle routes heen.
- **SearXNG**: sleutel `sx:<ip>`, `searx_rate_max_req` verzoeken per `searx_rate_window` seconden, alleen op herkende zoekroutes.

De geleidelijke escalatie (`record_suspicious_hit`) is een mechanisme dat los staat van de snelheidsbeperking: elke niet-kritieke detectie (niet-herhaald gevoelig pad, scanner-User-Agent, kleine hit in de POST-body) verhoogt een teller per IP (sleutel `susp:<ip>`) over een venster van `suspicious_window` seconden. Zodra `suspicious_threshold` hits binnen dat venster zijn bereikt, wordt het IP aan de definitieve blokkeerlijst toegevoegd — voorkomt een harde ban door één enkele valse positief, terwijl herhaling toch wordt gebanned.

**Nooit gebande netwerken**: een adres uit `never_auto_block` (lokale host, Tailscale `100.64.0.0/10`, lokale IPv4/IPv6-netwerken, inclusief de IPv4-gemapte IPv6-vorm) leidt nooit tot escalatie of toevoeging aan de blokkeerlijst; een vermeden ban wordt vastgelegd met een regel `BAN EVITE` in `blocklist_log` (nooit `AJOUT BLOCKLIST`, die door omega-jail wordt gelezen). CIDR-vergelijking in draagbaar Lua, van 5.1 tot 5.5 en LuaJIT.

**Log-only-modus van omega-serv**: als de WAF van omega-serv op `log-only` staat, plaatst het script geen blokkering (geen blokkeerlijst, geen regel `AJOUT BLOCKLIST`, dus geen netwerkblokkering door fail2ban); een regel `BAN EVITE` legt dat vast. Kan de blokkeerlijst niet worden geschreven, dan vervangt een regel `ECHEC BLOCKLIST` de regel `AJOUT BLOCKLIST`.

---

## Persistentie

| Bestand | Inhoud | Formaat |
|---|---|---|
| `blocklist_file` | Eén gebande IP per regel, voorafgegaan door een commentaar met tijdstempel (`# datum - reden`) | Platte tekst |
| `waf_rules_dir/whitelist_paths.txt` | Eén Lua-patroon (`string.match`) per regel, paden die altijd zijn toegestaan, zelfs voor een gebande IP | Platte tekst |
| `rate_store_file` | Tellers voor snelheidsbeperking en herhaling (`{ count = N, first = timestamp }` per sleutel) | Geserialiseerde Lua-tabel (`return { ... }`, ingelezen via `load()`) |
| `alerts_log` | Eén regel per detectie/weigering | Leesbare tekst, of JSON als `use_json_logs = true` |
| `blocklist_log` | Eén regel per toevoeging aan de blokkeerlijst | Leesbare tekst |

Het bestand van de snelheidsbeperking wordt nooit bij elk verzoek herschreven (hoogstens om de 300 seconden, bij de periodieke opschoning) om onevenredige I/O-kosten te vermijden. De blokkeerlijst en de whitelist worden in het geheugen gecachet en hoogstens om de `blocklist_refresh_interval`/`whitelist_refresh_interval` seconden opnieuw ingelezen — een toevoeging die door het script zelf wordt veroorzaakt, werkt de cache meteen bij, zonder die termijn af te wachten.

---

## Verzonden HTTP-responscodes

| Code | Betekenis | Aanleiding |
|---|---|---|
| `0` | Normale verwerking voortzetten (conventie van `mod_magnet`) | Geen detectie |
| `403` | Toegang geweigerd, eerste waarschuwing | Gevoelig pad gedetecteerd (eerste keer), scanner-User-Agent (eerste keer) |
| `413` | Entiteit te groot | POST-body groter dan `body_max_inspect` |
| `414` | URI te lang | SearXNG-query-string langer dan `searx_max_query_len` |
| `429` | Te veel verzoeken | Globale of SearXNG-snelheidsbeperking overschreden |
| `451` | Niet beschikbaar om juridische redenen (hier gebruikt als bancode) | IP al gebanned, herhaling na escalatie, SQLi/XSS/traversal gedetecteerd in het pad of de query-string, NUL-byte, POST-score ≥ drempel |
| `500` | Interne fout | Onvoorziene Lua-uitzondering (opgevangen door de globale `pcall`) |

---

## Compatibiliteit

Het script werkt in **twee contexten**, zonder enige wijziging:

- **native lighttpd**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` zijn de native API's van de module `mod_magnet`. De 5 configuratiepaden gebruiken rechtstreeks hun hardgecodeerde standaardwaarden.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): een Python-Lua-brug (`lupa`) bootst hetzelfde `lighty.*`-contract getrouw na, met een Lua-runtime die **persistent** is tussen verzoeken (tijdgebonden caches en snelheidsbeperking blijven dus correct behouden van het ene verzoek naar het andere, net als onder lighttpd). De globale tabel `omega_magnet_paths` wordt vóór de uitvoering geïnjecteerd vanuit de configuratie van het scherm WAF Magnet, zodat de 5 paden verwijzen naar de in de interface gekozen locaties in plaats van naar de hardgecodeerde waarden.

Geen enkel verschil in bedrijfslogica tussen de twee contexten — alleen de herkomst van de 5 configuratiepaden verandert.

### Wat NIET zonder meer compatibel is

Het script hangt volledig af van het `lighty.*`-contract (native API van `mod_magnet`) — het draait **niet** zomaar op een andere server, ook niet als die een vorm van Lua-scripting aanbiedt:

- **Apache** (`mod_lua`): andere API, gebaseerd op een verzoekobject `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — geen enkele `lighty.*`-aanroep zou werken.
- **nginx** (`lua-nginx-module` / OpenResty): andere API (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), met een model van uitvoeringsfasen (`access_by_lua`, `content_by_lua`...) dat verschilt van de eenvoudige teruggave van een code bij `mod_magnet`.
- **Traefik**: geen native Lua-scripting (geschreven in Go) — alleen een Go-plug-in (Yaegi) zou eigen logica kunnen uitvoeren, nooit rechtstreeks Lua.
- **Caddy**: geen officiële Lua-module — zou een plug-in van derden vereisen die via `xcaddy` wordt gecompileerd, native is er niets vergelijkbaars beschikbaar.

Dit script compatibel maken met een van deze servers zou een eigen compatibiliteitsbrug vereisen die hun native API naar `lighty.*` vertaalt (precies wat `lua_waf_adapter.py` voor omega-serv doet) — een volwaardig stuk ontwikkelwerk, geen eenvoudige configuratieaanpassing.

---

## Aanpassing

- **Aanpassen aan een andere applicatie dan SearXNG**: wijzig `is_searxng_path()` (lijst van paden) en verwijder zo nodig sectie 3 van de pipeline (eigen beleid) volledig als de beschermde applicatie geen aparte behandeling van zoekopdrachten nodig heeft.
- **Altijd toegestane paden toevoegen**: bewerk `waf_rules_dir/whitelist_paths.txt` (één Lua-patroon per regel), of voeg een hardgecodeerde uitzondering toe in `is_whitelisted_path()`.
- **De strengheid bijstellen**: `waf_score_threshold` (weigeringsdrempel voor POST), `suspicious_threshold`/`suspicious_window` (snelheid van de escalatie naar een definitieve ban), `global_rate_max_req`/`global_rate_window` (algemene snelheidsbeperking).
- **De logs omzetten naar JSON** (verwerking door SIEM/log shipper): `use_json_logs = true`.
- **Een subsysteem uitschakelen zonder de rest aan te raken**: `enable_body_inspection`, `enable_rate_limit` en `enable_auto_block` zijn onafhankelijk.

---

## Bekende beperkingen en buiten scope

- **Geen CMDi-detectie op het URL-pad**: bewuste keuze om valse positieven te vermijden bij paden met `.php` (verward met `<?php` als de CMDi-detectie de PHP-tag op dit vlak zou meenemen). De CMDi-detectie blijft actief op de POST-body.
- **Signaturen via Lua-patroonherkenning, geen echte PCRE-engine**: aanvalsvarianten die niet door de vermelde patronen worden gedekt, worden niet gedetecteerd — dit is geen vervanging voor een WAF op basis van uitgebreide regelsets (zoals ModSecurity CRS).
- **Geen parsing van multipart/form-data**: een kwaadaardige payload in een veld van een multipart-formulier kan aan de controle ontsnappen als de patronen niet overeenkomen met de ruwe, niet-gedecodeerde stroom.
- **Geen gzip-decompressie van de body**: een gecomprimeerde body wordt niet in leesbare vorm gecontroleerd.
- **Benaderende snelheidsbeperking**: vereenvoudigd glijdend venster (volledige reset na afloop van het venster), geen nauwkeurig algoritme met glijdend venster (sliding log/token bucket).
- **Geen bewakingsinterface**: elke observatie gebeurt door de logbestanden en de blokkeerlijst rechtstreeks te lezen.

---

## Probleemoplossing

- **Het script lijkt nooit te worden uitgevoerd**: controleer of `mod_magnet` echt geladen is (lighttpd) of dat de extra `magnet` geïnstalleerd is en WAF Magnet ingeschakeld (omega-serv); een herlading van de configuratie volstaat nooit om een gewijzigd Lua-script te laden, in beide gevallen is een volledige herstart nodig.
- **De tellers van de snelheidsbeperking lijken bij elk verzoek te worden gereset**: symptoom van een Lua-runtime die niet persistent is tussen verzoeken (het script begint bij elke uitvoering met een lege `rate_store`) — controleer onder omega-serv of het scherm WAF Magnet wel een actieve persistente runtime meldt.
- **Een legitiem IP is gebanned**: raadpleeg `blocklist_log` en `alerts_log` om de geactiveerde regel te vinden, pas het betreffende patroon of de drempel aan, en verwijder daarna de regel van het IP uit `blocklist_file` (de geheugencache wordt bijgewerkt bij de volgende herlading, hoogstens `blocklist_refresh_interval` seconden na de wijziging van het bestand).
- **Legitieme SearXNG-zoekopdrachten worden geblokkeerd**: controleer `is_searxng_path()` (pad niet herkend) voordat u de eigen drempels van de snelheidsbeperking aanpast.
- **`pip install -e ".[magnet]"` mislukt met `pip: opdracht niet gevonden` of `externally-managed-environment`**: het commando is buiten de `.venv` van omega-serv ingevoerd (sommige distributies, met name Arch Linux, bieden geen direct bruikbare systeem-`pip`). Gebruik `.venv/bin/python -m pip install -e ".[magnet]"` vanuit de map van omega-serv — werkt zonder af te hangen van een `pip` op het systeem-`PATH` (zie [De `magnet`-extra van omega-serv installeren](#de-magnet-extra-van-omega-serv-installeren)).

---

## Licentie

Omega-Waf wordt verspreid onder de **MIT**-licentie. Zie het bestand [`LICENSE`](../../../LICENSE) voor de volledige tekst.

---

> **Omega-Waf — Eén script, één verdedigingslinie.**
>
> Een applicatiefirewall voor mod_magnet in één enkel Lua-bestand, zonder afhankelijkheid, compatibel met lighttpd en omega-serv.
