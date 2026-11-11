<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="512">
</div>

# 🔒 OMEGA-WAF

**Firewall applicativo mod_magnet (script Lua autonomo)**

> Sviluppato da [kraynux](https://github.com/kraynux) per [Omega-Server](https://kraynux.snake-mackarel.ts.net)   

Pagina ufficiale: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/it/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#compatibilità)

🌐 Documentazione disponibile in **17 lingue**.  
**Lingue:**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](../pt-BR/README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](../de/README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](../nl/README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](../pl/README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](../vi/README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](../id/README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](../ja/README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](../ko/README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](../hi/README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

**Omega-Waf** è un unico script Lua (`omega-waf.lua`), senza dipendenze esterne, pensato per essere eseguito a ogni richiesta HTTP da un server web compatibile con `mod_magnet` (lighttpd in modo nativo, oppure omega-serv tramite il suo ponte di compatibilità `magnet`). Ispeziona il percorso, la query e, in POST, il corpo della richiesta alla ricerca di firme di attacco note (SQL injection, XSS, command injection, directory traversal), applica la limitazione della frequenza, gestisce una lista di blocco persistente con escalation progressiva e registra ogni decisione nei log.

Nessuna installazione in senso classico: un solo file da copiare, da richiamare nella configurazione del server, e basta.

## Sommario

- [Presentazione](#presentazione)
- [Installazione](#installazione)
- [Configurazione](#configurazione)
- [Pipeline di decisione](#pipeline-di-decisione)
- [Firme rilevate](#firme-rilevate)
- [Percorsi sensibili sorvegliati](#percorsi-sensibili-sorvegliati)
- [Limitazione della frequenza ed escalation](#limitazione-della-frequenza-ed-escalation)
- [Persistenza](#persistenza)
- [Codici di risposta HTTP emessi](#codici-di-risposta-http-emessi)
- [Compatibilità](#compatibilità)
- [Personalizzazione](#personalizzazione)
- [Limiti noti e fuori ambito](#limiti-noti-e-fuori-ambito)
- [Risoluzione dei problemi](#risoluzione-dei-problemi)
- [Licenza](#licenza)

---

## Presentazione

### Obiettivi

- Filtrare le richieste malevole il più vicino possibile al server, prima che raggiungano l'applicazione servita.
- Restare un file unico, leggibile, senza dipendenze esterne né fase di compilazione.
- Non far mai cadere il server web: ogni errore interno allo script viene intercettato e registrato nei log invece di essere propagato.
- Limitare il costo CPU/memoria per richiesta (cache temporali, lettura del corpo a blocchi, profondità di decodifica limitata).
- Offrire un'escalation progressiva (avviso e poi ban) invece di un ban secco al primo incidente minore.

### Cosa fa Omega-Waf

- Blocca gli indirizzi IP già bannati (lista di blocco persistente su disco), con un'eccezione per i percorsi esplicitamente autorizzati (CSS della pagina di errore, ecc.).
- Sorveglia un insieme di percorsi sensibili (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store`, ecc.) e risponde con una pagina di avviso HTML oppure con un ban diretto in caso di recidiva.
- Rileva gli User-Agent noti come strumenti di scansione o bot indesiderati (sqlmap, nmap, nikto, dirbuster, crawler generici...).
- Applica una limitazione della frequenza globale per IP, su tutte le rotte.
- Applica una politica dedicata, più permissiva, alle rotte di un motore di ricerca SearXNG (evita falsi positivi su ricerche legittime contenenti parole chiave che somiglierebbero a un attacco).
- Ispeziona il percorso e la query string (SQLi, XSS, directory traversal, byte NUL).
- Ispeziona il corpo delle richieste POST (SQLi, XSS, command injection, directory traversal) con un sistema di punteggio cumulativo.
- Gestisce un'escalation progressiva: un comportamento sospetto isolato fa scattare un avviso o un rifiuto puntuale; la recidiva entro una finestra di tempo fa scattare un ban definitivo.
- Registra nei log ogni decisione (avvisi, aggiunte alla lista di blocco) in formato testo o JSON.
- Funziona sia in lighttpd nativo (percorsi scritti nel codice) sia iniettato in omega-serv (percorsi ricevuti dinamicamente dalla configurazione del server).

### Cosa lo script non fa

- Non sostituisce un WAF dedicato di livello enterprise (ModSecurity, Coraza, un WAF cloud): le firme sono volutamente semplici (corrispondenza di pattern Lua, non un vero motore di regole PCRE).
- Non analizza le intestazioni HTTP una per una oltre allo User-Agent.
- Non decodifica i corpi multipart/form-data né i corpi compressi (gzip) — l'ispezione riguarda il testo grezzo ricevuto.
- Non ispeziona mai il corpo di una richiesta diversa da POST.
- Non esegue alcun rilevamento di command injection sul percorso dell'URL (scelta deliberata, vedi [Limiti noti](#limiti-noti-e-fuori-ambito)).
- Non fornisce alcuna interfaccia di amministrazione: la configurazione si fa modificando le variabili in testa al file, la lista di blocco e la whitelist tramite semplici file di testo.
- Non gestisce da sé la rotazione dei propri log.

### Avvertenza d'uso

Le firme fornite mirano a famiglie di attacchi comuni ma non costituiscono una protezione esaustiva. Una soglia di rilevamento troppo bassa su un sito con traffico insolito (ricerca tecnica, documentazione contenente comandi shell, ecc.) può generare falsi positivi — regola di conseguenza `waf_score_threshold` e la lista bianca dei percorsi. Testa sempre una modifica di firma o di soglia in un ambiente non critico prima di distribuirla in produzione.

---

## Installazione

### Prerequisiti

- Un server web che esponga un'interfaccia compatibile con `mod_magnet`:
  - **lighttpd** con il modulo `mod_magnet` attivato (`server.modules += ("mod_magnet")`), oppure
  - **omega-serv** con l'extra `magnet` installato (dipendenza `lupa` — vedi sotto e la guida di omega-serv, sezione WAF Magnet).
- Un interprete Lua incorporato fornito dall'host (lighttpd incorpora il proprio Lua; omega-serv incorpora `lupa`, in nessuno dei due casi serve un interprete `lua` di sistema — `lupa` è distribuito come wheel precompilata per tutte le architetture/versioni di Python comuni, senza alcuna compilazione locale né intestazioni Lua di sistema).

#### Installare l'extra `magnet` di omega-serv

**Sempre dal `.venv` già creato da `omega-serv/install.sh`, mai con un `pip` di sistema** — su alcune distribuzioni (in particolare Arch Linux), il comando `pip` nudo non esiste (lo installa solo `python-pip`, e non sempre di default) oppure si rifiuta di installare qualsiasi cosa al di fuori di un ambiente virtuale (`error: externally-managed-environment`, PEP 668):

```bash
cd /percorso/di/omega-serv

# Opzione A - venv gia attivato nella shell corrente
source .venv/bin/activate
pip install -e ".[magnet]"

# Opzione B - senza attivare il venv (la piu portabile: non dipende mai
# dalla presenza o meno di un 'pip' nel PATH di sistema, perche il python
# del venv include sempre il proprio)
.venv/bin/python -m pip install -e ".[magnet]"
```

Se `install.sh` non è ancora stato eseguito (nessuna `.venv/` presente), eseguilo prima — crea l'ambiente virtuale e vi installa già omega-serv stesso; resta quindi da aggiungere solo l'extra `magnet` (facoltativo) con uno dei due comandi qui sopra.

### Messa in opera

1. Copia `omega-waf.lua` in una posizione leggibile dal server (es. `secure/waf/scripts/omega-waf.lua` per una struttura in stile omega-serv, o qualsiasi percorso per lighttpd nativo).
2. Crea le cartelle/i file attesi:
   - **omega-serv**: niente da fare, il server crea da sé le cartelle all'avvio (tra cui `var/lib/waf/`, condivisa con l'account del servizio).
   - **lighttpd nativo**: sostituisci prima i 5 percorsi con percorsi assoluti (vedi [Percorsi in lighttpd nativo](#percorsi-in-lighttpd-nativo)), poi crea le cartelle e rendile scrivibili dall'account di lighttpd (`www-data` su Debian/Ubuntu, `http` su Arch, `lighttpd` su Fedora/Alpine):
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. Richiama lo script nella configurazione del server:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/percorso/di/omega-waf.lua" )
     ```
   - **omega-serv**: attiva il WAF Magnet dalla schermata TUI dedicata (Sicurezza attiva → WAF → Magnet), che punta allo script e inietta automaticamente la tabella `omega_magnet_paths` (vedi [Compatibilità](#compatibilità)).
4. Riavvia (non ricaricare — un ricaricamento non ricarica mai uno script Lua già in memoria persistente) il server interessato.

---

## Configurazione

Tutte le opzioni si regolano modificando direttamente le variabili locali in testa al file (sezione `CONFIGURATION`, righe ~24-70). Nessun file di configurazione separato.

| Variabile | Ruolo | Valore predefinito |
|---|---|---|
| `blocklist_file` | File di testo degli IP bannati definitivamente (un IP per riga, `#` per i commenti) — **scritto** dallo script | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Cartella delle regole esterne (contiene `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Log degli avvisi (rilevamenti, rifiuti, superamenti) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Log dedicato alle aggiunte alla lista di blocco | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | File di persistenza dei contatori (limite di frequenza / recidiva), formato tabella Lua | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Attiva l'ispezione del corpo delle richieste POST | `true` |
| `enable_rate_limit` | Attiva la limitazione della frequenza (globale + SearXNG) | `true` |
| `enable_auto_block` | Consente l'aggiunta automatica alla lista di blocco | `true` |
| `never_auto_block` | Reti **mai** bannate automaticamente (la richiesta resta rifiutata e registrata nei log, si evita solo il ban); sotto omega-serv, la lista iniettata dal server (`omega_never_auto_block`) prevale sempre | host locale, Tailscale, reti locali |
| `use_json_logs` | Passa i log degli avvisi in JSON (altrimenti testo leggibile) | `false` |
| `waf_score_threshold` | Punteggio minimo cumulato perché una richiesta POST venga rifiutata | `5` |
| `body_max_inspect` | Dimensione massima ispezionata del corpo (byte) | `65536` (64 KB) |
| `chunk_size` | Dimensione dei blocchi di lettura del corpo | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | Quota e finestra (s) della limitazione della frequenza globale per IP | `60` richieste / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | Quota e finestra (s) della limitazione della frequenza dedicata alle rotte SearXNG | `40` richieste / `60` s |
| `searx_max_query_len` | Dimensione massima accettata della query string sulle rotte SearXNG | `4096` caratteri |
| `suspicious_threshold` | Numero di hit sospetti prima del ban definitivo | `3` |
| `suspicious_window` | Finestra mobile (s) su cui vengono contati gli hit sospetti | `300` (5 min) |
| `blocklist_refresh_interval` | Intervallo (s) di rilettura del file della lista di blocco | `5` |
| `whitelist_refresh_interval` | Intervallo (s) di rilettura della whitelist dei percorsi | `15` |

Sotto omega-serv, i primi 5 percorsi (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) vengono sostituiti automaticamente da quelli configurati nella schermata WAF Magnet (tabella globale `omega_magnet_paths` iniettata prima dell'esecuzione) — i valori scritti nel codice qui sopra servono allora solo da ripiego se questa tabella è assente.

> ℹ️ **Modifica del 2026-10-18**: `blocklist_file` passa da `secure/waf/magnet-blocklist.txt` a `var/lib/waf/magnet-blocklist.txt`. Lo script **scrive** in questo file a ogni ban, e il servizio systemd di omega-serv può scrivere solo sotto `var/`: con la vecchia posizione, nessun ban automatico veniva mai registrato, senza alcun messaggio. omega-serv riprende il vecchio file una sola volta all'avvio (il vecchio resta al suo posto come backup); un percorso personalizzato nella schermata WAF Magnet non viene mai modificato.

### Percorsi in lighttpd nativo

> ⚠️ **Questi 5 valori predefiniti ricalcano la convenzione delle cartelle di omega-serv** (`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, percorsi relativi risolti rispetto a una directory di lavoro di tipo progetto) — non sono neutri. In **lighttpd nativo**, dove non esiste alcuna convenzione di cartelle equivalente, questi percorsi relativi vengono risolti rispetto alla directory di lavoro del processo lighttpd a runtime (spesso `/`), raramente prevedibile. **Sostituisci queste 5 righe con percorsi assoluti** adatti alla tua installazione, per esempio:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> Prima di riavviare, crea queste cartelle (comandi al passo 2 della [Messa in opera](#messa-in-opera)) rispettando questi permessi:
>
> | File | Accesso necessario all'account di lighttpd |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **scrittura** (il file e la cartella che lo contiene) |
> | `waf_rules_dir` (e `whitelist_paths.txt`) | sola lettura |
>
> Un errore di scrittura è **silenzioso** — lo script non interrompe mai la richiesta —: un ban che non compare in `blocklist_file` è l'unico sintomo. Verifica con `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` (adatta l'account). Se l'unità systemd di lighttpd limita le scritture (`ProtectSystem=strict`, `ReadWritePaths=`), aggiungi queste cartelle.

---

## Pipeline di decisione

Lo script viene eseguito interamente dentro un `pcall` globale: ogni errore Lua imprevisto viene registrato nei log e poi risponde `500`, senza mai far crollare il processo del server. I passi vengono eseguiti in quest'ordine, e ognuno può interrompere l'elaborazione con una risposta finale:

1. **Lista di blocco** — se l'IP è bannato, richiesta rifiutata (`451`), salvo se il percorso richiesto è esplicitamente nella whitelist (serve per esempio la pagina di errore stessa).
2. **Percorsi sensibili** — se il percorso corrisponde a un pattern sorvegliato (`.git`, `.env`, `wp-admin`...), viene restituita una pagina di avviso HTML (`403`) la prima volta; la recidiva entro la finestra di sospetto fa scattare un ban (`451`).
3. **User-Agent di scanner** — se lo User-Agent corrisponde a uno strumento di scansione/bot noto, `403` la prima volta, `451` in caso di recidiva.
4. **Limite di frequenza globale** — se la quota globale per IP è superata, `429`.
5. **Politica SearXNG** — per le rotte di ricerca riconosciute: limite di frequenza dedicato (`429`), protezione sulla dimensione della query string (`414`), rifiuto dei byte NUL (`451`), poi uscita immediata (`0`, continua) per non sottoporre mai una ricerca legittima ai controlli aggressivi successivi.
6. **Protezione del percorso** — ricerca di SQLi e XSS nel percorso decodificato (`451` se rilevato); ricerca di directory traversal (`451`). Volutamente **nessun** rilevamento di command injection in questo passo (vedi [Limiti noti](#limiti-noti-e-fuori-ambito)).
7. **Protezione leggera della query string** (esclusa SearXNG) — directory traversal e byte NUL (`451`).
8. **Ispezione del corpo POST** — solo se `enable_body_inspection` è attivo e il metodo è `POST`: corpo troppo voluminoso rifiutato direttamente (`413`); altrimenti punteggio cumulato su SQLi/XSS/CMDi/traversal, rifiuto (`451`) se il punteggio raggiunge `waf_score_threshold`, con ban immediato se il punteggio raggiunge 8, altrimenti semplice hit sospetto.
9. **Nessun blocco** — la richiesta prosegue normalmente (`0`).

---

## Firme rilevate

Quattro famiglie di pattern (`patterns`), dove ogni voce è una stringa (sensibile alle maiuscole) oppure una tabella `{ pat = "motif", ci = true }` (insensibile alle maiuscole):

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, iniezioni di comandi SQL tramite punto e virgola (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: tag `<script>`, `<iframe>`, attributi `on*=` (gestori di eventi), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: concatenazione di comandi shell (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, varianti con `|`), sostituzione di comando (`` `...` ``, `$(...)`), tag PHP (`<?php`) e funzioni di esecuzione (`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

E una lista dedicata di **User-Agent di scanner/bot** (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, crawler SEO (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`) e pattern generici (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

La decodifica URL viene applicata ricorsivamente (fino a 5 livelli sul percorso, 2 livelli sulla query string) prima dell'ispezione, per intercettare i payload codificati due volte.

---

## Percorsi sensibili sorvegliati

Pattern verificati sul percorso decodificato: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

Per ogni corrispondenza, lo script cerca di stabilire se una risorsa fisica esiste davvero in quel percorso (tramite `lighty.stat`, con ripiego su `lighty.c.stat` per compatibilità) e lo indica nella pagina di avviso e nel log — utile per distinguere una scansione alla cieca da un percorso che esiste davvero sul server.

---

## Limitazione della frequenza ed escalation

Due limitatori di frequenza indipendenti, basati su contatori in memoria resi persistenti su disco (finestra mobile semplice, non una finestra mobile precisa al secondo):

- **Globale**: chiave `g:<ip>`, `global_rate_max_req` richieste ogni `global_rate_window` secondi, su tutte le rotte.
- **SearXNG**: chiave `sx:<ip>`, `searx_rate_max_req` richieste ogni `searx_rate_window` secondi, solo sulle rotte di ricerca riconosciute.

L'escalation progressiva (`record_suspicious_hit`) è un meccanismo separato dalla limitazione della frequenza: ogni rilevamento non critico (percorso sensibile senza recidiva, User-Agent di scanner, hit minore sul corpo POST) incrementa un contatore per IP (chiave `susp:<ip>`) su una finestra di `suspicious_window` secondi. Una volta raggiunti `suspicious_threshold` hit in questa finestra, l'IP viene aggiunto alla lista di blocco definitiva — evita un ban secco per un singolo falso positivo, pur bannando la recidiva.

**Reti mai bannate**: un indirizzo di `never_auto_block` (host locale, Tailscale `100.64.0.0/10`, reti locali IPv4/IPv6, compresa la forma IPv4 mappata in IPv6) non fa mai scattare un'escalation né un'aggiunta alla lista di blocco; un ban evitato viene tracciato da una riga `BAN EVITE` in `blocklist_log` (mai `AJOUT BLOCKLIST`, riletta da omega-jail). Confronto CIDR in Lua portabile, dalla 5.1 alla 5.5 e LuaJIT.

**Modalità log-only di omega-serv**: quando il WAF di omega-serv è in `log-only`, lo script non applica alcun ban (né lista di blocco, né riga `AJOUT BLOCKLIST`, quindi nessun ban di rete da parte di fail2ban); una riga `BAN EVITE` lo registra. Se la lista di blocco non può essere scritta, una riga `ECHEC BLOCKLIST` sostituisce `AJOUT BLOCKLIST`.

---

## Persistenza

| File | Contenuto | Formato |
|---|---|---|
| `blocklist_file` | Un IP bannato per riga, preceduto da un commento con data e ora (`# date - raison`) | Testo semplice |
| `waf_rules_dir/whitelist_paths.txt` | Un pattern Lua (`string.match`) per riga, percorsi sempre autorizzati anche per un IP bannato | Testo semplice |
| `rate_store_file` | Contatori di limitazione della frequenza e di recidiva (`{ count = N, first = timestamp }` per chiave) | Tabella Lua serializzata (`return { ... }`, riletta tramite `load()`) |
| `alerts_log` | Una riga per rilevamento/rifiuto | Testo leggibile, o JSON se `use_json_logs = true` |
| `blocklist_log` | Una riga per aggiunta alla lista di blocco | Testo leggibile |

Il file della limitazione della frequenza non viene mai riscritto a ogni richiesta (al massimo ogni 300 secondi, durante la pulizia periodica) per evitare un costo di I/O sproporzionato. La lista di blocco e la whitelist sono tenute in cache in memoria e rilette al massimo ogni `blocklist_refresh_interval`/`whitelist_refresh_interval` secondi — un'aggiunta fatta scattare dallo script stesso aggiorna subito la cache, senza attendere questo intervallo.

---

## Codici di risposta HTTP emessi

| Codice | Significato | Causa |
|---|---|---|
| `0` | Prosegue l'elaborazione normale (convenzione di `mod_magnet`) | Nessun rilevamento |
| `403` | Accesso negato, primo avviso | Percorso sensibile rilevato (prima volta), User-Agent di scanner (prima volta) |
| `413` | Entità troppo grande | Corpo POST oltre `body_max_inspect` |
| `414` | URI troppo lungo | Query string SearXNG oltre `searx_max_query_len` |
| `429` | Troppe richieste | Limite di frequenza globale o SearXNG superato |
| `451` | Non disponibile per motivi legali (usato qui come codice di ban) | IP già bannato, recidiva dopo escalation, SQLi/XSS/traversal rilevati nel percorso o nella query string, byte NUL, punteggio POST ≥ soglia |
| `500` | Errore interno | Eccezione Lua imprevista (intercettata dal `pcall` globale) |

---

## Compatibilità

Lo script funziona in **due contesti**, senza alcuna modifica:

- **lighttpd nativo**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` sono le API native del modulo `mod_magnet`. I 5 percorsi di configurazione usano direttamente i loro valori predefiniti scritti nel codice.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): un ponte Python-Lua (`lupa`) riproduce fedelmente lo stesso contratto `lighty.*`, con un runtime Lua **persistente** tra le richieste (le cache temporali e la limitazione della frequenza sopravvivono quindi correttamente da una richiesta all'altra, come sotto lighttpd). La tabella globale `omega_magnet_paths` viene iniettata prima dell'esecuzione a partire dalla configurazione della schermata WAF Magnet, affinché i 5 percorsi puntino alle posizioni scelte nell'interfaccia invece che ai valori scritti nel codice.

Nessuna divergenza di logica applicativa tra i due contesti — cambia solo l'origine dei 5 percorsi di configurazione.

### Cosa NON è compatibile così com'è

Lo script dipende interamente dal contratto `lighty.*` (API nativa di `mod_magnet`) — **non** gira così com'è su un altro server, anche se questo offre una forma di scripting Lua:

- **Apache** (`mod_lua`): API diversa, basata su un oggetto richiesta `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — nessuna chiamata `lighty.*` funzionerebbe.
- **nginx** (`lua-nginx-module` / OpenResty): API diversa (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), con un modello di fasi di esecuzione (`access_by_lua`, `content_by_lua`...) distinto dal semplice codice di ritorno di `mod_magnet`.
- **Traefik**: nessuno scripting Lua nativo (scritto in Go) — solo un plugin Go (Yaegi) permetterebbe di eseguire logica personalizzata, mai direttamente Lua.
- **Caddy**: nessun modulo Lua ufficiale — richiederebbe un plugin di terze parti compilato tramite `xcaddy`, niente di equivalente disponibile in modo nativo.

Rendere questo script compatibile con uno di questi server richiederebbe di scrivere un ponte di compatibilità dedicato che traduca la sua API nativa verso `lighty.*` (esattamente ciò che fa `lua_waf_adapter.py` per omega-serv) — un lavoro di ingegneria a tutti gli effetti, non un semplice adattamento della configurazione.

---

## Personalizzazione

- **Adattare a un'applicazione diversa da SearXNG**: modifica `is_searxng_path()` (lista dei percorsi) e, se serve, rimuovi interamente la sezione 3 della pipeline (politica dedicata) se l'applicazione protetta non ha bisogno di un trattamento differenziato delle ricerche.
- **Aggiungere percorsi sempre autorizzati**: modifica `waf_rules_dir/whitelist_paths.txt` (un pattern Lua per riga), oppure aggiungi un'eccezione scritta nel codice in `is_whitelisted_path()`.
- **Regolare l'aggressività**: `waf_score_threshold` (soglia di rifiuto POST), `suspicious_threshold`/`suspicious_window` (velocità dell'escalation verso un ban definitivo), `global_rate_max_req`/`global_rate_window` (limite di frequenza generale).
- **Passare i log in JSON** (acquisizione da SIEM/log shipper): `use_json_logs = true`.
- **Disattivare un sottosistema senza toccare il resto**: `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block` sono indipendenti.

---

## Limiti noti e fuori ambito

- **Nessun rilevamento CMDi sul percorso dell'URL**: scelta deliberata per evitare falsi positivi su percorsi contenenti `.php` (confuso con `<?php` se il rilevamento CMDi includesse il tag PHP su questa superficie). Il rilevamento CMDi resta attivo sul corpo POST.
- **Firme basate sulla corrispondenza di pattern Lua, non un vero motore PCRE**: le varianti di attacco non coperte dai pattern elencati non verranno rilevate — questo non sostituisce un WAF basato su regole esaustive (tipo CRS di ModSecurity).
- **Nessun parsing di multipart/form-data**: un payload malevolo incapsulato in un campo di un modulo multipart può sfuggire all'ispezione se i pattern non corrispondono al flusso grezzo non decodificato.
- **Nessuna decompressione gzip del corpo**: un corpo compresso non viene ispezionato in chiaro.
- **Limitazione della frequenza approssimativa**: finestra mobile semplificata (azzeramento completo al superamento della finestra), non un algoritmo di finestra mobile precisa (sliding log/token bucket).
- **Nessuna interfaccia di supervisione**: ogni osservazione passa per la lettura diretta dei file di log e della lista di blocco.

---

## Risoluzione dei problemi

- **Lo script sembra non essere mai eseguito**: verifica che `mod_magnet` sia davvero caricato (lighttpd) oppure che l'extra `magnet` sia installato e il WAF Magnet attivato (omega-serv); un ricaricamento della configurazione non basta mai a caricare uno script Lua modificato, serve un riavvio completo in entrambi i casi.
- **I contatori del limite di frequenza sembrano azzerarsi a ogni richiesta**: sintomo di un runtime Lua non persistente tra le richieste (lo script riparte da un `rate_store` vuoto a ogni esecuzione) — sotto omega-serv, verifica che la schermata WAF Magnet indichi un runtime persistente attivo.
- **Un IP legittimo finisce bannato**: consulta `blocklist_log` e `alerts_log` per individuare la regola scattata, regola il pattern o la soglia corrispondente, poi rimuovi la riga dell'IP in `blocklist_file` (la cache in memoria si aggiorna alla rilettura successiva, al massimo `blocklist_refresh_interval` secondi dopo la modifica del file).
- **Le ricerche SearXNG legittime vengono bloccate**: verifica `is_searxng_path()` (percorso non riconosciuto) prima di regolare le soglie dedicate del limite di frequenza.
- **`pip install -e ".[magnet]"` fallisce con `pip: comando non trovato` o `externally-managed-environment`**: il comando è stato digitato al di fuori del `.venv` di omega-serv (alcune distribuzioni, in particolare Arch Linux, non espongono alcun `pip` di sistema utilizzabile direttamente). Usa `.venv/bin/python -m pip install -e ".[magnet]"` dalla directory di omega-serv — funziona senza dipendere da un `pip` presente nel `PATH` di sistema (vedi [Installare l'extra `magnet` di omega-serv](#installare-lextra-magnet-di-omega-serv)).

---

## Licenza

Omega-Waf è distribuito con licenza **MIT**. Consulta il file [`LICENSE`](../../../LICENSE) per il testo completo.

---

> **Omega-Waf — Uno script, una linea di difesa.**
>
> Un firewall applicativo mod_magnet in un unico file Lua, senza dipendenze, compatibile con lighttpd e omega-serv.
