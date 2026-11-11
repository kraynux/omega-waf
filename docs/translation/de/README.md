<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="512">
</div>

# 🔒 OMEGA-WAF

**Anwendungs-Firewall für mod_magnet (eigenständiges Lua-Skript)**

> Entwickelt von [kraynux](https://github.com/kraynux) für [Omega-Server](https://kraynux.snake-mackarel.ts.net)   

Offizielle Seite: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/de/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#kompatibilität)

🌐 Dokumentation in **17 Sprachen** verfügbar.  
**Sprachen:**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](../pt-BR/README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](../it/README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](../nl/README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](../pl/README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](../vi/README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](../id/README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](../ja/README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](../ko/README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](../hi/README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

**Omega-Waf** ist ein einzelnes Lua-Skript (`omega-waf.lua`) ohne externe Abhängigkeit, das bei jeder HTTP-Anfrage von einem mit `mod_magnet` kompatiblen Webserver ausgeführt wird (nativ von lighttpd oder von omega-serv über dessen Kompatibilitätsbrücke `magnet`). Es untersucht den Pfad, die Anfrage und bei POST den Anfrage-Body auf bekannte Angriffssignaturen (SQL-Injection, XSS, Befehlsinjektion, Verzeichnis-Traversal), wendet eine Ratenbegrenzung an, verwaltet eine persistente Sperrliste mit schrittweiser Eskalation und protokolliert jede Entscheidung.

Keine Installation im klassischen Sinn: eine einzige Datei ablegen, in der Serverkonfiguration referenzieren, fertig.

## Inhaltsverzeichnis

- [Überblick](#überblick)
- [Installation](#installation)
- [Konfiguration](#konfiguration)
- [Entscheidungspipeline](#entscheidungspipeline)
- [Erkannte Signaturen](#erkannte-signaturen)
- [Überwachte sensible Pfade](#überwachte-sensible-pfade)
- [Ratenbegrenzung und Eskalation](#ratenbegrenzung-und-eskalation)
- [Persistenz](#persistenz)
- [Ausgegebene HTTP-Antwortcodes](#ausgegebene-http-antwortcodes)
- [Kompatibilität](#kompatibilität)
- [Anpassung](#anpassung)
- [Bekannte Grenzen und nicht abgedeckte Bereiche](#bekannte-grenzen-und-nicht-abgedeckte-bereiche)
- [Fehlerbehebung](#fehlerbehebung)
- [Lizenz](#lizenz)

---

## Überblick

### Ziele

- Bösartige Anfragen so nah wie möglich am Server filtern, bevor sie die ausgelieferte Anwendung erreichen.
- Eine einzige, lesbare Datei bleiben, ohne externe Abhängigkeit und ohne Kompilierschritt.
- Den Webserver niemals zum Absturz bringen: Jeder interne Fehler des Skripts wird abgefangen und protokolliert, statt weitergereicht zu werden.
- Die CPU-/Speicherkosten pro Anfrage begrenzen (zeitbasierte Caches, blockweises Lesen des Bodys, begrenzte Dekodiertiefe).
- Eine schrittweise Eskalation (Warnung, dann Sperre) bieten statt einer harten Sperre beim ersten kleinen Vorfall.

### Was Omega-Waf tut

- Blockiert bereits gesperrte IP-Adressen (persistente Sperrliste auf der Festplatte), mit einer Ausnahme für ausdrücklich erlaubte Pfade (CSS der Fehlerseite usw.).
- Überwacht eine Reihe sensibler Pfade (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store` usw.) und antwortet entweder mit einer HTML-Warnseite oder bei Wiederholung mit einer direkten Sperre.
- Erkennt User-Agents, die als Scan-Werkzeuge oder unerwünschte Bots bekannt sind (sqlmap, nmap, nikto, dirbuster, generische Crawler...).
- Wendet eine globale Ratenbegrenzung pro IP an, über alle Routen hinweg.
- Wendet auf die Routen einer SearXNG-Suchmaschine eine eigene, großzügigere Richtlinie an (vermeidet Fehlalarme bei legitimen Suchanfragen mit Stichwörtern, die wie ein Angriff aussehen könnten).
- Untersucht den Pfad und den Query-String (SQLi, XSS, Verzeichnis-Traversal, NUL-Byte).
- Untersucht den Body von POST-Anfragen (SQLi, XSS, Befehlsinjektion, Verzeichnis-Traversal) mit einem kumulativen Punktesystem.
- Verwaltet eine schrittweise Eskalation: Ein vereinzeltes verdächtiges Verhalten löst eine Warnung oder eine einmalige Ablehnung aus; eine Wiederholung innerhalb eines Zeitfensters führt zu einer dauerhaften Sperre.
- Protokolliert jede Entscheidung (Warnungen, Einträge in die Sperrliste) im Text- oder JSON-Format.
- Funktioniert sowohl unter nativem lighttpd (fest eingetragene Pfade) als auch eingebunden in omega-serv (Pfade werden dynamisch aus der Serverkonfiguration übernommen).

### Was das Skript nicht tut

- Es ersetzt keinen dedizierten WAF auf Unternehmensniveau (ModSecurity, Coraza, ein Cloud-WAF): Die Signaturen sind bewusst einfach gehalten (Lua-Mustervergleich, keine echte PCRE-Regel-Engine).
- Es analysiert die HTTP-Header über den User-Agent hinaus nicht einzeln.
- Es dekodiert weder multipart/form-data-Bodys noch komprimierte Bodys (gzip) — die Untersuchung betrifft den empfangenen Rohtext.
- Es untersucht niemals den Body einer anderen Anfrage als POST.
- Es führt keine Erkennung von Befehlsinjektion im URL-Pfad durch (bewusste Entscheidung, siehe [Bekannte Grenzen](#bekannte-grenzen-und-nicht-abgedeckte-bereiche)).
- Es bietet keine Verwaltungsoberfläche: Die Konfiguration erfolgt durch Bearbeiten der Variablen am Dateianfang, Sperrliste und Whitelist über einfache Textdateien.
- Es kümmert sich nicht selbst um die Rotation seiner Logs.

### Nutzungshinweis

Die mitgelieferten Signaturen zielen auf verbreitete Angriffsfamilien, bieten aber keinen lückenlosen Schutz. Eine zu niedrige Erkennungsschwelle auf einer Website mit ungewöhnlichem Verkehr (technische Suche, Dokumentation mit Shell-Befehlen usw.) kann Fehlalarme erzeugen — passen Sie `waf_score_threshold` und die Pfad-Whitelist entsprechend an. Testen Sie eine Änderung an Signaturen oder Schwellen immer zuerst in einer unkritischen Umgebung, bevor Sie sie in der Produktion einsetzen.

---

## Installation

### Voraussetzungen

- Ein Webserver mit einer zu `mod_magnet` kompatiblen Schnittstelle:
  - **lighttpd** mit aktiviertem Modul `mod_magnet` (`server.modules += ("mod_magnet")`), oder
  - **omega-serv** mit installiertem Extra `magnet` (Abhängigkeit `lupa` — siehe unten und das omega-serv-Handbuch, Abschnitt WAF Magnet).
- Ein vom Host bereitgestellter eingebetteter Lua-Interpreter (lighttpd bringt sein eigenes Lua mit; omega-serv bringt `lupa` mit, in beiden Fällen ist kein System-Interpreter `lua` nötig — `lupa` wird als vorkompiliertes Wheel für alle gängigen Architekturen/Python-Versionen verteilt, weder lokale Kompilierung noch System-Lua-Header erforderlich).

#### Das Extra `magnet` von omega-serv installieren

**Immer aus dem bereits von `omega-serv/install.sh` erstellten `.venv`, niemals mit einem System-`pip`** — auf manchen Distributionen (insbesondere Arch Linux) existiert der bloße Befehl `pip` nicht (nur `python-pip` installiert ihn, und nicht unbedingt standardmäßig) oder er verweigert jede Installation außerhalb einer virtuellen Umgebung (`error: externally-managed-environment`, PEP 668):

```bash
cd /pfad/zu/omega-serv

# Option A - venv bereits in der aktuellen Shell aktiviert
source .venv/bin/activate
pip install -e ".[magnet]"

# Option B - ohne das venv zu aktivieren (am portabelsten: haengt nie
# davon ab, ob ein 'pip' im System-PATH vorhanden ist, denn das python
# des venv bringt immer sein eigenes mit)
.venv/bin/python -m pip install -e ".[magnet]"
```

Wenn `install.sh` noch nicht ausgeführt wurde (kein `.venv/` vorhanden), führen Sie es zuerst aus — es erstellt die virtuelle Umgebung und installiert omega-serv selbst bereits darin; danach muss nur noch das (optionale) Extra `magnet` mit einem der beiden obigen Befehle hinzugefügt werden.

### Einrichtung

1. `omega-waf.lua` an einen für den Server lesbaren Ort kopieren (z. B. `secure/waf/scripts/omega-waf.lua` für eine Verzeichnisstruktur nach Art von omega-serv, oder einen beliebigen Pfad für natives lighttpd).
2. Die erwarteten Verzeichnisse/Dateien anlegen:
   - **omega-serv**: nichts zu tun, der Server legt die Ordner beim Start selbst an (darunter `var/lib/waf/`, gemeinsam mit dem Dienstkonto genutzt).
   - **natives lighttpd**: zuerst die 5 Pfade durch absolute Pfade ersetzen (siehe [Pfade bei nativem lighttpd](#pfade-bei-nativem-lighttpd)), dann die Ordner anlegen und für das lighttpd-Konto beschreibbar machen (`www-data` unter Debian/Ubuntu, `http` unter Arch, `lighttpd` unter Fedora/Alpine):
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. Das Skript in der Serverkonfiguration referenzieren:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/pfad/zu/omega-waf.lua" )
     ```
   - **omega-serv**: WAF Magnet im zugehörigen TUI-Bildschirm aktivieren (Aktive Sicherheit → WAF → Magnet), der auf das Skript verweist und automatisch die Tabelle `omega_magnet_paths` einbindet (siehe [Kompatibilität](#kompatibilität)).
4. Den betreffenden Server neu starten (nicht neu laden — ein Neuladen lädt ein bereits dauerhaft im Speicher befindliches Lua-Skript niemals neu).

---

## Konfiguration

Alle Optionen werden direkt über die lokalen Variablen am Dateianfang eingestellt (Abschnitt `CONFIGURATION`, Zeilen ~24-70). Keine separate Konfigurationsdatei.

| Variable | Rolle | Standardwert |
|---|---|---|
| `blocklist_file` | Textdatei der dauerhaft gesperrten IPs (eine IP pro Zeile, `#` für Kommentare) — wird vom Skript **geschrieben** | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Ordner der externen Regeln (enthält `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Log der Warnungen (Erkennungen, Ablehnungen, Überschreitungen) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Eigenes Log für Einträge in die Sperrliste | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | Persistenzdatei der Zähler (Ratenbegrenzung / Wiederholung), Format Lua-Tabelle | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Aktiviert die Untersuchung des Bodys von POST-Anfragen | `true` |
| `enable_rate_limit` | Aktiviert die Ratenbegrenzung (global + SearXNG) | `true` |
| `enable_auto_block` | Erlaubt das automatische Eintragen in die Sperrliste | `true` |
| `never_auto_block` | Netze, die **niemals** automatisch gesperrt werden (die Anfrage bleibt abgelehnt und protokolliert, nur die Sperre entfällt); unter omega-serv hat die vom Server eingebundene Liste (`omega_never_auto_block`) immer Vorrang | lokaler Host, Tailscale, lokale Netze |
| `use_json_logs` | Stellt die Warn-Logs auf JSON um (sonst lesbarer Text) | `false` |
| `waf_score_threshold` | Kumulierte Mindestpunktzahl, ab der eine POST-Anfrage abgelehnt wird | `5` |
| `body_max_inspect` | Maximal untersuchte Größe des Bodys (Bytes) | `65536` (64 KB) |
| `chunk_size` | Größe der Leseblöcke des Bodys | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | Kontingent und Fenster (s) der globalen Ratenbegrenzung pro IP | `60` Anfragen / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | Kontingent und Fenster (s) der eigenen Ratenbegrenzung für SearXNG-Routen | `40` Anfragen / `60` s |
| `searx_max_query_len` | Maximal akzeptierte Länge des Query-Strings auf SearXNG-Routen | `4096` Zeichen |
| `suspicious_threshold` | Anzahl verdächtiger Treffer bis zur dauerhaften Sperre | `3` |
| `suspicious_window` | Gleitendes Fenster (s), in dem verdächtige Treffer gezählt werden | `300` (5 min) |
| `blocklist_refresh_interval` | Intervall (s) für das erneute Einlesen der Sperrliste | `5` |
| `whitelist_refresh_interval` | Intervall (s) für das erneute Einlesen der Pfad-Whitelist | `15` |

Unter omega-serv werden die ersten 5 Pfade (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) automatisch durch die im Bildschirm WAF Magnet konfigurierten ersetzt (globale Tabelle `omega_magnet_paths`, vor der Ausführung eingebunden) — die oben fest eingetragenen Werte dienen dann nur als Rückfall, falls diese Tabelle fehlt.

> ℹ️ **Änderung vom 2026-10-18**: `blocklist_file` wechselt von `secure/waf/magnet-blocklist.txt` zu `var/lib/waf/magnet-blocklist.txt`. Das Skript **schreibt** bei jeder Sperre in diese Datei, und der systemd-Dienst von omega-serv darf nur unter `var/` schreiben: Am alten Speicherort wurde nie eine automatische Sperre gespeichert, ohne jede Meldung. omega-serv übernimmt die alte Datei einmalig beim Start (die alte bleibt als Sicherung erhalten); ein im Bildschirm WAF Magnet eigens gesetzter Pfad wird nie verändert.

### Pfade bei nativem lighttpd

> ⚠️ **Diese 5 Standardwerte folgen der Ordnerkonvention von omega-serv** (`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, relative Pfade, die gegenüber einem Arbeitsverzeichnis im Projektstil aufgelöst werden) — sie sind nicht neutral. Unter **nativem lighttpd**, wo es keine entsprechende Ordnerkonvention gibt, werden diese relativen Pfade zur Laufzeit gegenüber dem Arbeitsverzeichnis des lighttpd-Prozesses aufgelöst (oft `/`), was selten vorhersehbar ist. **Ersetzen Sie diese 5 Zeilen durch absolute Pfade**, die zu Ihrer Installation passen, zum Beispiel:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> Legen Sie diese Ordner vor dem Neustart an (Befehle in Schritt 2 der [Einrichtung](#einrichtung)) und beachten Sie dabei folgende Rechte:
>
> | Datei | Vom lighttpd-Konto benötigter Zugriff |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **Schreiben** (Datei und der Ordner, der sie enthält) |
> | `waf_rules_dir` (und `whitelist_paths.txt`) | nur Lesen |
>
> Ein Schreibfehler bleibt **stumm** — das Skript lässt die Anfrage nie scheitern —: Eine Sperre, die nicht in `blocklist_file` auftaucht, ist das einzige Symptom. Prüfen Sie mit `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` (Konto anpassen). Falls die systemd-Unit von lighttpd Schreibzugriffe einschränkt (`ProtectSystem=strict`, `ReadWritePaths=`), fügen Sie diese Ordner dort hinzu.

---

## Entscheidungspipeline

Das Skript läuft vollständig in einem globalen `pcall`: Jeder unvorhergesehene Lua-Fehler wird protokolliert und mit `500` beantwortet, ohne jemals den Serverprozess abstürzen zu lassen. Die Schritte laufen in dieser Reihenfolge ab, und jeder kann die Verarbeitung mit einer endgültigen Antwort beenden:

1. **Sperrliste** — ist die IP gesperrt, wird die Anfrage abgelehnt (`451`), außer der angeforderte Pfad steht ausdrücklich auf der Whitelist (liefert zum Beispiel die Fehlerseite selbst aus).
2. **Sensible Pfade** — entspricht der Pfad einem überwachten Muster (`.git`, `.env`, `wp-admin`...), wird beim ersten Mal eine HTML-Warnseite zurückgegeben (`403`); eine Wiederholung innerhalb des Verdachtsfensters löst eine Sperre aus (`451`).
3. **Scanner-User-Agent** — entspricht der User-Agent einem bekannten Scan-Werkzeug/Bot, beim ersten Mal `403`, bei Wiederholung `451`.
4. **Globale Ratenbegrenzung** — ist das globale Kontingent pro IP überschritten, `429`.
5. **SearXNG-Richtlinie** — für erkannte Suchrouten: eigene Ratenbegrenzung (`429`), Schutz gegen zu lange Query-Strings (`414`), Ablehnung von NUL-Bytes (`451`), danach sofortiger Ausstieg (`0`, weiter), damit eine legitime Suche nie den folgenden aggressiven Prüfungen unterzogen wird.
6. **Schutz des Pfads** — Suche nach SQLi und XSS im dekodierten Pfad (`451` bei Erkennung); Suche nach Verzeichnis-Traversal (`451`). In diesem Schritt bewusst **keine** Erkennung von Befehlsinjektion (siehe [Bekannte Grenzen](#bekannte-grenzen-und-nicht-abgedeckte-bereiche)).
7. **Leichter Schutz des Query-Strings** (außer SearXNG) — Verzeichnis-Traversal und NUL-Byte (`451`).
8. **Untersuchung des POST-Bodys** — nur wenn `enable_body_inspection` aktiv ist und die Methode `POST` lautet: zu großer Body wird direkt abgelehnt (`413`); andernfalls kumulierte Punktzahl über SQLi/XSS/CMDi/Traversal, Ablehnung (`451`), wenn die Punktzahl `waf_score_threshold` erreicht, mit sofortiger Sperre, wenn die Punktzahl 8 erreicht, sonst nur ein verdächtiger Treffer.
9. **Keine Blockierung** — die Anfrage läuft normal weiter (`0`).

---

## Erkannte Signaturen

Vier Musterfamilien (`patterns`), wobei jeder Eintrag entweder eine Zeichenkette (Groß-/Kleinschreibung beachtet) oder eine Tabelle `{ pat = "Muster", ci = true }` (Groß-/Kleinschreibung ignoriert) ist:

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, SQL-Befehlsinjektionen per Semikolon (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: Tags `<script>`, `<iframe>`, Attribute `on*=` (Event-Handler), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: Verkettung von Shell-Befehlen (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, Varianten mit `|`), Befehlssubstitution (`` `...` ``, `$(...)`), PHP-Tag (`<?php`) und Ausführungsfunktionen (`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

Dazu eine eigene Liste von **Scanner-/Bot-User-Agents** (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, SEO-Crawler (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`) und generische Muster (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

Die URL-Dekodierung wird vor der Untersuchung rekursiv angewendet (bis zu 5 Ebenen auf dem Pfad, 2 Ebenen auf dem Query-String), um doppelt kodierte Payloads abzufangen.

---

## Überwachte sensible Pfade

Auf dem dekodierten Pfad geprüfte Muster: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

Bei jeder Übereinstimmung versucht das Skript festzustellen, ob unter diesem Pfad tatsächlich eine physische Ressource existiert (über `lighty.stat`, mit Rückfall auf `lighty.c.stat` aus Kompatibilitätsgründen), und vermerkt dies in der Warnseite und im Log — nützlich, um einen blinden Scan von einem tatsächlich auf dem Server existierenden Pfad zu unterscheiden.

---

## Ratenbegrenzung und Eskalation

Zwei unabhängige Ratenbegrenzer auf Basis von Zählern im Speicher, die auf der Festplatte persistiert werden (einfaches gleitendes Fenster, kein sekundengenaues gleitendes Fenster):

- **Global**: Schlüssel `g:<ip>`, `global_rate_max_req` Anfragen pro `global_rate_window` Sekunden, über alle Routen hinweg.
- **SearXNG**: Schlüssel `sx:<ip>`, `searx_rate_max_req` Anfragen pro `searx_rate_window` Sekunden, nur auf erkannten Suchrouten.

Die schrittweise Eskalation (`record_suspicious_hit`) ist ein von der Ratenbegrenzung getrennter Mechanismus: Jede nicht kritische Erkennung (nicht wiederholter sensibler Pfad, Scanner-User-Agent, kleiner Treffer im POST-Body) erhöht einen Zähler pro IP (Schlüssel `susp:<ip>`) über ein Fenster von `suspicious_window` Sekunden. Sobald `suspicious_threshold` Treffer in diesem Fenster erreicht sind, wird die IP in die dauerhafte Sperrliste eingetragen — vermeidet eine harte Sperre wegen eines einzelnen Fehlalarms und sperrt trotzdem Wiederholungstäter.

**Niemals gesperrte Netze**: Eine Adresse aus `never_auto_block` (lokaler Host, Tailscale `100.64.0.0/10`, lokale IPv4-/IPv6-Netze, einschließlich der IPv4-mapped-IPv6-Form) löst niemals eine Eskalation oder einen Eintrag in die Sperrliste aus; eine vermiedene Sperre wird durch eine Zeile `BAN EVITE` in `blocklist_log` vermerkt (niemals `AJOUT BLOCKLIST`, die von omega-jail ausgewertet wird). CIDR-Vergleich in portablem Lua, von 5.1 bis 5.5 und LuaJIT.

**Log-only-Modus von omega-serv**: Steht die WAF von omega-serv auf `log-only`, setzt das Skript keine Sperre (weder Sperrliste noch Zeile `AJOUT BLOCKLIST`, also auch keine Netzwerksperre durch fail2ban); eine Zeile `BAN EVITE` protokolliert es. Kann die Sperrliste nicht geschrieben werden, ersetzt eine Zeile `ECHEC BLOCKLIST` die Zeile `AJOUT BLOCKLIST`.

---

## Persistenz

| Datei | Inhalt | Format |
|---|---|---|
| `blocklist_file` | Eine gesperrte IP pro Zeile, mit vorangestelltem Kommentar samt Zeitstempel (`# Datum - Grund`) | Klartext |
| `waf_rules_dir/whitelist_paths.txt` | Ein Lua-Muster (`string.match`) pro Zeile, Pfade, die selbst für eine gesperrte IP immer erlaubt sind | Klartext |
| `rate_store_file` | Zähler für Ratenbegrenzung und Wiederholung (`{ count = N, first = timestamp }` pro Schlüssel) | Serialisierte Lua-Tabelle (`return { ... }`, eingelesen über `load()`) |
| `alerts_log` | Eine Zeile pro Erkennung/Ablehnung | Lesbarer Text oder JSON bei `use_json_logs = true` |
| `blocklist_log` | Eine Zeile pro Eintrag in die Sperrliste | Lesbarer Text |

Die Datei der Ratenbegrenzung wird nie bei jeder Anfrage neu geschrieben (höchstens alle 300 Sekunden, bei der periodischen Bereinigung), um unverhältnismäßige I/O-Kosten zu vermeiden. Sperrliste und Whitelist werden im Speicher zwischengespeichert und höchstens alle `blocklist_refresh_interval`/`whitelist_refresh_interval` Sekunden neu eingelesen — ein vom Skript selbst ausgelöster Eintrag aktualisiert den Cache sofort, ohne diese Frist abzuwarten.

---

## Ausgegebene HTTP-Antwortcodes

| Code | Bedeutung | Auslöser |
|---|---|---|
| `0` | Normale Verarbeitung fortsetzen (Konvention von `mod_magnet`) | Keine Erkennung |
| `403` | Zugriff verweigert, erste Warnung | Sensibler Pfad erkannt (erstes Mal), Scanner-User-Agent (erstes Mal) |
| `413` | Anfrage-Body zu groß | POST-Body überschreitet `body_max_inspect` |
| `414` | URI zu lang | SearXNG-Query-String überschreitet `searx_max_query_len` |
| `429` | Zu viele Anfragen | Globale oder SearXNG-Ratenbegrenzung überschritten |
| `451` | Aus rechtlichen Gründen nicht verfügbar (hier als Sperrcode verwendet) | IP bereits gesperrt, Wiederholung nach Eskalation, SQLi/XSS/Traversal im Pfad oder Query-String erkannt, NUL-Byte, POST-Punktzahl ≥ Schwelle |
| `500` | Interner Fehler | Unvorhergesehene Lua-Ausnahme (vom globalen `pcall` abgefangen) |

---

## Kompatibilität

Das Skript funktioniert in **zwei Kontexten**, ohne jede Änderung:

- **natives lighttpd**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` sind die nativen APIs des Moduls `mod_magnet`. Die 5 Konfigurationspfade verwenden direkt ihre fest eingetragenen Standardwerte.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): Eine Python-Lua-Brücke (`lupa`) bildet denselben `lighty.*`-Vertrag getreu nach, mit einer zwischen den Anfragen **persistenten** Lua-Laufzeit (zeitbasierte Caches und Ratenbegrenzung bleiben daher wie unter lighttpd von einer Anfrage zur nächsten korrekt erhalten). Die globale Tabelle `omega_magnet_paths` wird vor der Ausführung aus der Konfiguration des Bildschirms WAF Magnet eingebunden, damit die 5 Pfade auf die in der Oberfläche gewählten Orte zeigen statt auf die fest eingetragenen Werte.

Keine Abweichung der Fachlogik zwischen den beiden Kontexten — nur die Herkunft der 5 Konfigurationspfade ändert sich.

### Was so NICHT kompatibel ist

Das Skript hängt vollständig vom `lighty.*`-Vertrag ab (native API von `mod_magnet`) — es läuft **nicht** unverändert auf einem anderen Server, selbst wenn dieser eine Form von Lua-Scripting anbietet:

- **Apache** (`mod_lua`): andere API, basierend auf einem Anfrageobjekt `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — kein einziger `lighty.*`-Aufruf würde funktionieren.
- **nginx** (`lua-nginx-module` / OpenResty): andere API (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), mit einem Modell von Ausführungsphasen (`access_by_lua`, `content_by_lua`...), das sich von der einfachen Code-Rückgabe von `mod_magnet` unterscheidet.
- **Traefik**: kein natives Lua-Scripting (in Go geschrieben) — nur ein Go-Plugin (Yaegi) könnte eigene Logik ausführen, niemals direkt Lua.
- **Caddy**: kein offizielles Lua-Modul — würde ein über `xcaddy` kompiliertes Drittanbieter-Plugin erfordern, nativ ist nichts Vergleichbares verfügbar.

Dieses Skript mit einem dieser Server kompatibel zu machen, würde eine eigene Kompatibilitätsbrücke erfordern, die deren native API in `lighty.*` übersetzt (genau das, was `lua_waf_adapter.py` für omega-serv tut) — eine eigenständige Entwicklungsarbeit, keine bloße Konfigurationsanpassung.

---

## Anpassung

- **Für eine andere Anwendung als SearXNG anpassen**: `is_searxng_path()` (Liste der Pfade) ändern und, falls nötig, Abschnitt 3 der Pipeline (eigene Richtlinie) ganz entfernen, wenn die geschützte Anwendung keine gesonderte Behandlung von Suchanfragen braucht.
- **Immer erlaubte Pfade hinzufügen**: `waf_rules_dir/whitelist_paths.txt` bearbeiten (ein Lua-Muster pro Zeile) oder eine fest eingetragene Ausnahme in `is_whitelisted_path()` ergänzen.
- **Die Strenge anpassen**: `waf_score_threshold` (Ablehnungsschwelle für POST), `suspicious_threshold`/`suspicious_window` (Geschwindigkeit der Eskalation bis zur dauerhaften Sperre), `global_rate_max_req`/`global_rate_window` (allgemeine Ratenbegrenzung).
- **Logs auf JSON umstellen** (Übernahme durch SIEM/Log-Shipper): `use_json_logs = true`.
- **Ein Teilsystem abschalten, ohne den Rest anzutasten**: `enable_body_inspection`, `enable_rate_limit` und `enable_auto_block` sind unabhängig voneinander.

---

## Bekannte Grenzen und nicht abgedeckte Bereiche

- **Keine CMDi-Erkennung im URL-Pfad**: bewusste Entscheidung, um Fehlalarme bei Pfaden mit `.php` zu vermeiden (würde mit `<?php` verwechselt, wenn die CMDi-Erkennung das PHP-Tag auf dieser Fläche einschlösse). Die CMDi-Erkennung bleibt für den POST-Body aktiv.
- **Signaturen per Lua-Mustervergleich, keine echte PCRE-Engine**: Angriffsvarianten, die nicht von den aufgeführten Mustern abgedeckt sind, werden nicht erkannt — dies ist kein Ersatz für einen WAF mit umfassendem Regelwerk (wie ModSecurity CRS).
- **Kein Parsing von multipart/form-data**: Ein bösartiger Payload in einem Feld eines Multipart-Formulars kann der Untersuchung entgehen, wenn die Muster nicht auf den undekodierten Rohdatenstrom passen.
- **Keine gzip-Dekomprimierung des Bodys**: Ein komprimierter Body wird nicht im Klartext untersucht.
- **Ungefähre Ratenbegrenzung**: vereinfachtes gleitendes Fenster (vollständiges Zurücksetzen nach Ablauf des Fensters), kein präziser Algorithmus mit gleitendem Fenster (Sliding Log/Token-Bucket).
- **Keine Überwachungsoberfläche**: Jede Beobachtung erfolgt durch direktes Lesen der Log-Dateien und der Sperrliste.

---

## Fehlerbehebung

- **Das Skript scheint nie ausgeführt zu werden**: Prüfen, ob `mod_magnet` wirklich geladen ist (lighttpd) bzw. ob das Extra `magnet` installiert und WAF Magnet aktiviert ist (omega-serv); ein Neuladen der Konfiguration genügt nie, um ein geändertes Lua-Skript zu laden, in beiden Fällen ist ein vollständiger Neustart nötig.
- **Die Zähler der Ratenbegrenzung scheinen sich bei jeder Anfrage zurückzusetzen**: Symptom einer zwischen den Anfragen nicht persistenten Lua-Laufzeit (das Skript beginnt bei jeder Ausführung mit einem leeren `rate_store`) — unter omega-serv prüfen, ob der Bildschirm WAF Magnet tatsächlich eine aktive persistente Laufzeit meldet.
- **Eine legitime IP wurde gesperrt**: `blocklist_log` und `alerts_log` einsehen, um die ausgelöste Regel zu ermitteln, das betreffende Muster oder die Schwelle anpassen, dann die Zeile der IP aus `blocklist_file` entfernen (der Speicher-Cache wird beim nächsten Einlesen aktualisiert, höchstens `blocklist_refresh_interval` Sekunden nach Änderung der Datei).
- **Legitime SearXNG-Suchen werden blockiert**: `is_searxng_path()` prüfen (Pfad nicht erkannt), bevor die eigenen Schwellen der Ratenbegrenzung angepasst werden.
- **`pip install -e ".[magnet]"` schlägt mit `pip: Befehl nicht gefunden` oder `externally-managed-environment` fehl**: Der Befehl wurde außerhalb des `.venv` von omega-serv eingegeben (manche Distributionen, insbesondere Arch Linux, stellen kein direkt nutzbares System-`pip` bereit). Verwenden Sie `.venv/bin/python -m pip install -e ".[magnet]"` aus dem Verzeichnis von omega-serv — funktioniert, ohne von einem `pip` im System-`PATH` abzuhängen (siehe [Das Extra `magnet` von omega-serv installieren](#das-extra-magnet-von-omega-serv-installieren)).

---

## Lizenz

Omega-Waf wird unter der **MIT**-Lizenz verteilt. Den vollständigen Text finden Sie in der Datei [`LICENSE`](../../../LICENSE).

---

> **Omega-Waf — Ein Skript, eine Verteidigungslinie.**
>
> Eine Anwendungs-Firewall für mod_magnet in einer einzigen Lua-Datei, ohne Abhängigkeit, kompatibel mit lighttpd und omega-serv.
