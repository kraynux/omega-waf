<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="384">
</div>

# 🔒 OMEGA-WAF

**Firewall applicatif mod_magnet (script Lua autonome)**

> Élaboré par **kraynux** pour **Omega-server**
> [kraynux.snake-mackarel.ts.net](https://kraynux.snake-mackarel.ts.net)

Page officielle : [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#compatibilité)

**Langues:**
[Français](README.md) · [English](README.en.md) · [Español](README.es.md) · [Русский](README.ru.md) · [中文](README.zh-CN.md)

---

**Omega-Waf** est un unique script Lua (`omega-waf.lua`), sans dépendance externe, destiné à être exécuté à chaque requête HTTP par un serveur web compatible `mod_magnet` (lighttpd nativement, ou omega-serv via son pont de compatibilité `magnet`). Il inspecte le chemin, la requête et, sur POST, le corps de la requête à la recherche de signatures d'attaque connues (injection SQL, XSS, injection de commandes, traversée de répertoire), applique du rate limiting, gère une liste noire persistante avec escalade progressive, et journalise chaque décision.

Aucune installation au sens classique : un seul fichier à déposer, référencer depuis la configuration du serveur, et c'est tout.

## Sommaire

- [Présentation](#présentation)
- [Installation](#installation)
- [Configuration](#configuration)
- [Pipeline de décision](#pipeline-de-décision)
- [Signatures détectées](#signatures-détectées)
- [Chemins sensibles surveillés](#chemins-sensibles-surveillés)
- [Rate limiting et escalade](#rate-limiting-et-escalade)
- [Persistance](#persistance)
- [Codes de réponse HTTP émis](#codes-de-réponse-http-émis)
- [Compatibilité](#compatibilité)
- [Personnalisation](#personnalisation)
- [Limites connues et hors périmètre](#limites-connues-et-hors-périmètre)
- [Dépannage](#dépannage)
- [Licence](#licence)

---

## Présentation

### Objectifs

- Filtrer les requêtes malveillantes au plus près du serveur, avant qu'elles n'atteignent l'application servie.
- Rester un fichier unique, lisible, sans dépendance externe ni étape de compilation.
- Ne jamais faire tomber le serveur web : toute erreur interne au script est rattrapée et journalisée plutôt que propagée.
- Limiter le coût CPU/mémoire par requête (caches temporels, lecture du corps par morceaux, profondeur de décodage bornée).
- Offrir une escalade progressive (avertissement puis bannissement) plutôt qu'un bannissement sec au premier incident mineur.

### Ce que fait Omega-Waf

- Bloque les adresses IP déjà bannies (liste noire persistante sur disque), avec une exception pour les chemins explicitement autorisés (CSS de la page d'erreur, etc.).
- Surveille un ensemble de chemins sensibles (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store`, etc.) et répond soit par une page d'avertissement HTML, soit par un bannissement direct en cas de récidive.
- Détecte les User-Agent connus comme outils de scan ou bots indésirables (sqlmap, nmap, nikto, dirbuster, crawlers génériques...).
- Applique un rate limiting global par IP, toutes routes confondues.
- Applique une politique dédiée, plus permissive, sur les routes d'un moteur de recherche SearXNG (évite les faux positifs sur des requêtes de recherche légitimes contenant des mots-clés qui ressembleraient à une attaque).
- Inspecte le chemin et la query-string (SQLi, XSS, traversée de répertoire, octet NUL).
- Inspecte le corps des requêtes POST (SQLi, XSS, injection de commandes, traversée de répertoire) avec un système de score cumulatif.
- Gère une escalade progressive : un comportement suspect isolé déclenche une alerte ou un refus ponctuel ; la récidive dans une fenêtre de temps déclenche un bannissement définitif.
- Journalise chaque décision (alertes, ajouts à la liste noire) au format texte ou JSON.
- Fonctionne aussi bien en lighttpd natif (chemins codés en dur) qu'injecté dans omega-serv (chemins reçus dynamiquement depuis la configuration du serveur).

### Ce que le script ne fait pas

- Il ne remplace pas un WAF dédié de niveau entreprise (ModSecurity, Coraza, un WAF cloud) : les signatures sont volontairement simples (correspondance de motifs Lua, pas un vrai moteur de règles PCRE).
- Il n'analyse pas les en-têtes HTTP un par un au-delà du User-Agent.
- Il ne décode pas les corps multipart/form-data ni les corps compressés (gzip) — l'inspection porte sur le texte brut reçu.
- Il n'inspecte jamais le corps d'une requête autre que POST.
- Il ne fait aucune détection d'injection de commandes sur le chemin de l'URL (choix délibéré, voir [Limites connues](#limites-connues-et-hors-périmètre)).
- Il ne fournit aucune interface d'administration : la configuration se fait en éditant les variables en tête de fichier, la liste noire et la whitelist via de simples fichiers texte.
- Il ne gère pas lui-même la rotation de ses journaux.

### Avertissement d'usage

Les signatures fournies ciblent des familles d'attaques courantes mais ne constituent pas une protection exhaustive. Un score de détection trop bas sur un site au trafic inhabituel (recherche technique, documentation contenant des commandes shell, etc.) peut générer des faux positifs — ajustez `waf_score_threshold` et la liste blanche de chemins en conséquence. Testez toujours un changement de signature ou de seuil en environnement non critique avant de le déployer en production.

---

## Installation

### Prérequis

- Un serveur web exposant une interface compatible `mod_magnet` :
  - **lighttpd** avec le module `mod_magnet` activé (`server.modules += ("mod_magnet")`), ou
  - **omega-serv** avec l'extra `magnet` installé (`pip install -e ".[magnet]"`, dépendance `lupa` — voir le guide omega-serv, section WAF Magnet).
- Un interpréteur Lua embarqué fourni par l'hôte (lighttpd embarque son propre Lua ; omega-serv embarque `lupa`, aucun interpréteur `lua` système requis dans les deux cas).

### Mise en place

1. Copier `omega-waf.lua` dans un emplacement lisible par le serveur (ex. `secure/waf/scripts/omega-waf.lua` pour une arborescence façon omega-serv, ou n'importe quel chemin pour lighttpd natif).
2. Créer les répertoires/fichiers attendus par la configuration par défaut (voir [Configuration](#configuration)) :
   ```bash
   mkdir -p secure/waf/magnet-rules.d var/log var/cache
   touch secure/waf/magnet-blocklist.txt
   ```
3. Référencer le script depuis la configuration du serveur :
   - **lighttpd** :
     ```
     magnet.attract-physical-path-to = ( "/chemin/vers/omega-waf.lua" )
     ```
   - **omega-serv** : activer le WAF Magnet depuis l'écran TUI dédié (Sécurité active → WAF → Magnet), qui pointe vers le script et injecte automatiquement la table `omega_magnet_paths` (voir [Compatibilité](#compatibilité)).
4. Redémarrer (pas recharger — un rechargement ne recharge jamais un script Lua déjà en mémoire persistante) le serveur concerné.

---

## Configuration

Toutes les options se règlent en modifiant directement les variables locales en tête du fichier (section `CONFIGURATION`, lignes ~24-70). Aucun fichier de configuration séparé.

| Variable | Rôle | Valeur par défaut |
|---|---|---|
| `blocklist_file` | Fichier texte des IP bannies définitivement (une IP par ligne, `#` pour commentaire) | `secure/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Dossier des règles externes (contient `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Journal des alertes (détections, refus, dépassements) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Journal dédié aux ajouts en liste noire | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | Fichier de persistance des compteurs (rate limit / récidive), format table Lua | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Active l'inspection du corps des requêtes POST | `true` |
| `enable_rate_limit` | Active le rate limiting (global + SearXNG) | `true` |
| `enable_auto_block` | Autorise l'ajout automatique à la liste noire | `true` |
| `use_json_logs` | Bascule les journaux d'alerte en JSON (sinon texte lisible) | `false` |
| `waf_score_threshold` | Score minimum cumulé pour qu'une requête POST soit refusée | `5` |
| `body_max_inspect` | Taille maximale inspectée du corps (octets) | `65536` (64 Ko) |
| `chunk_size` | Taille des blocs de lecture du corps | `8192` (8 Ko) |
| `global_rate_max_req` / `global_rate_window` | Quota et fenêtre (s) du rate limiting global par IP | `60` requêtes / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | Quota et fenêtre (s) du rate limiting dédié aux routes SearXNG | `40` requêtes / `60` s |
| `searx_max_query_len` | Taille maximale acceptée de la query-string sur les routes SearXNG | `4096` caractères |
| `suspicious_threshold` | Nombre de hits suspects avant bannissement définitif | `3` |
| `suspicious_window` | Fenêtre glissante (s) sur laquelle les hits suspects sont comptés | `300` (5 min) |
| `blocklist_refresh_interval` | Intervalle (s) de relecture du fichier de liste noire | `5` |
| `whitelist_refresh_interval` | Intervalle (s) de relecture de la whitelist de chemins | `15` |

Sous omega-serv, les 5 premiers chemins (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) sont automatiquement remplacés par ceux configurés dans l'écran WAF Magnet (table globale `omega_magnet_paths` injectée avant exécution) — les valeurs en dur ci-dessus ne servent alors que de repli si cette table est absente.

> ⚠️ **Ces 5 valeurs par défaut sont calquées sur la convention de dossiers d'omega-serv** (`secure/waf/...`, `var/log/...`, `var/cache/...`, des chemins relatifs résolus par rapport à un répertoire de travail de type projet) — elles ne sont pas neutres. En **lighttpd natif**, où aucune convention de dossier équivalente n'existe, ces chemins relatifs se résolvent par rapport au répertoire de travail du processus lighttpd au runtime, rarement prévisible. Il est recommandé de remplacer ces 5 lignes par des chemins absolus adaptés à votre installation, par exemple :
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> N'oubliez pas de créer ces répertoires (et de donner les droits d'écriture nécessaires à l'utilisateur sous lequel lighttpd s'exécute) avant de redémarrer.

---

## Pipeline de décision

Le script s'exécute entièrement dans un `pcall` global : toute erreur Lua imprévue est journalisée puis répond `500`, sans jamais faire planter le processus serveur. Les étapes s'exécutent dans cet ordre, chacune pouvant arrêter le traitement avec une réponse finale :

1. **Liste noire** — si l'IP est bannie, requête refusée (`451`), sauf si le chemin demandé est explicitement whitelisté (sert par exemple la page d'erreur elle-même).
2. **Chemins sensibles** — si le chemin correspond à un motif surveillé (`.git`, `.env`, `wp-admin`...), une page d'avertissement HTML est renvoyée (`403`) la première fois ; la récidive dans la fenêtre de suspicion déclenche un bannissement (`451`).
3. **User-Agent scanner** — si le User-Agent correspond à un outil de scan/bot connu, `403` la première fois, `451` en cas de récidive.
4. **Rate limit global** — si le quota global par IP est dépassé, `429`.
5. **Politique SearXNG** — pour les routes de recherche reconnues : rate limit dédié (`429`), garde-fou sur la taille de la query-string (`414`), rejet des octets NUL (`451`), puis sortie immédiate (`0`, continue) pour ne jamais soumettre une recherche légitime aux vérifications agressives suivantes.
6. **Protection du chemin** — recherche de SQLi et XSS dans le chemin décodé (`451` si détecté) ; recherche de traversée de répertoire (`451`). Volontairement **pas** de détection d'injection de commandes à cette étape (voir [Limites connues](#limites-connues-et-hors-périmètre)).
7. **Protection légère de la query-string** (hors SearXNG) — traversée de répertoire et octet NUL (`451`).
8. **Inspection du corps POST** — uniquement si `enable_body_inspection` est actif et que la méthode est `POST` : corps trop volumineux refusé directement (`413`) ; sinon score cumulé sur SQLi/XSS/CMDi/traversée, refus (`451`) si le score atteint `waf_score_threshold`, avec bannissement immédiat si le score atteint 8, sinon simple hit suspect.
9. **Aucun blocage** — la requête continue normalement (`0`).

---

## Signatures détectées

Quatre familles de motifs (`patterns`), chaque entrée étant soit une chaîne (sensible à la casse), soit une table `{ pat = "motif", ci = true }` (insensible à la casse) :

- **`sqli_body`** : `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, injections de commandes SQL via point-virgule (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`** : balises `<script>`, `<iframe>`, attributs `on*=` (gestionnaires d'événements), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`** : chaînage de commandes shell (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, variantes avec `|`), substitution de commande (`` `...` ``, `$(...)`), balise PHP (`<?php`) et fonctions d'exécution (`system(`, `exec(`, `passthru(`).
- **`path_traversal`** : `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

Et une liste dédiée de **User-Agent de scanners/bots** (`patterns_scanner_ua`) : `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, crawlers de référencement (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`), et motifs génériques (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

Le décodage URL est appliqué récursivement (jusqu'à 5 niveaux sur le chemin, 2 niveaux sur la query-string) avant inspection, pour attraper les payloads doublement encodés.

---

## Chemins sensibles surveillés

Motifs vérifiés sur le chemin décodé : `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

Pour chaque correspondance, le script tente de déterminer si une ressource physique existe réellement à ce chemin (via `lighty.stat`, avec repli sur `lighty.c.stat` pour compatibilité) et l'indique dans la page d'avertissement et le journal — utile pour distinguer un scan à l'aveugle d'un chemin qui existe réellement sur le serveur.

---

## Rate limiting et escalade

Deux rate limiters indépendants, basés sur des compteurs en mémoire persistés sur disque (fenêtre glissante simple, pas de fenêtre glissante précise à la seconde) :

- **Global** : clé `g:<ip>`, `global_rate_max_req` requêtes par `global_rate_window` secondes, toutes routes confondues.
- **SearXNG** : clé `sx:<ip>`, `searx_rate_max_req` requêtes par `searx_rate_window` secondes, uniquement sur les routes de recherche reconnues.

L'escalade progressive (`record_suspicious_hit`) est un mécanisme séparé du rate limiting : chaque détection non critique (chemin sensible non récidivant, User-Agent scanner, hit mineur sur le corps POST) incrémente un compteur par IP (clé `susp:<ip>`) sur une fenêtre de `suspicious_window` secondes. Une fois `suspicious_threshold` hits atteints dans cette fenêtre, l'IP est ajoutée à la liste noire définitive — évite un bannissement sec sur un unique faux positif, tout en bannissant la récidive.

---

## Persistance

| Fichier | Contenu | Format |
|---|---|---|
| `blocklist_file` | Une IP bannie par ligne, précédée d'un commentaire horodaté (`# date - raison`) | Texte brut |
| `waf_rules_dir/whitelist_paths.txt` | Un motif Lua (`string.match`) par ligne, chemins toujours autorisés même pour une IP bannie | Texte brut |
| `rate_store_file` | Compteurs de rate limiting et de récidive (`{ count = N, first = timestamp }` par clé) | Table Lua sérialisée (`return { ... }`, relue via `load()`) |
| `alerts_log` | Une ligne par détection/refus | Texte lisible, ou JSON si `use_json_logs = true` |
| `blocklist_log` | Une ligne par ajout à la liste noire | Texte lisible |

Le fichier de rate limiting n'est jamais réécrit à chaque requête (uniquement au plus toutes les 300 secondes, lors du nettoyage périodique) pour éviter un coût I/O disproportionné. La liste noire et la whitelist sont mises en cache mémoire et relues au plus toutes les `blocklist_refresh_interval`/`whitelist_refresh_interval` secondes — un ajout déclenché par le script lui-même met à jour le cache immédiatement, sans attendre ce délai.

---

## Codes de réponse HTTP émis

| Code | Signification | Déclencheur |
|---|---|---|
| `0` | Continue le traitement normal (convention `mod_magnet`) | Aucune détection |
| `403` | Accès refusé, premier avertissement | Chemin sensible détecté (première fois), User-Agent scanner (première fois) |
| `413` | Entité trop volumineuse | Corps POST dépassant `body_max_inspect` |
| `414` | URI trop longue | Query-string SearXNG dépassant `searx_max_query_len` |
| `429` | Trop de requêtes | Rate limit global ou SearXNG dépassé |
| `451` | Indisponible pour raisons légales (utilisé ici comme code de bannissement) | IP déjà bannie, récidive après escalade, SQLi/XSS/traversée détectée dans le chemin ou la query-string, octet NUL, score POST ≥ seuil |
| `500` | Erreur interne | Exception Lua non prévue (rattrapée par le `pcall` global) |

---

## Compatibilité

Le script fonctionne dans **deux contextes**, sans aucune modification :

- **lighttpd natif** : `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` sont les API natives du module `mod_magnet`. Les 5 chemins de configuration utilisent directement leurs valeurs par défaut codées en dur.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`) : un pont Python-Lua (`lupa`) reproduit fidèlement le même contrat `lighty.*`, avec un runtime Lua **persistant** entre les requêtes (les caches temporels et le rate limiting survivent donc correctement d'une requête à l'autre, comme sous lighttpd). La table globale `omega_magnet_paths` est injectée avant exécution depuis la configuration de l'écran WAF Magnet, pour que les 5 chemins pointent vers les emplacements choisis dans l'interface plutôt que vers les valeurs en dur.

Aucune divergence de logique métier entre les deux contextes — seule l'origine des 5 chemins de configuration change.

### Ce qui n'est PAS compatible tel quel

Le script dépend entièrement du contrat `lighty.*` (API native de `mod_magnet`) — il ne tourne **pas** tel quel sur un autre serveur, même si celui-ci propose une forme de scripting Lua :

- **Apache** (`mod_lua`) : API différente, basée sur un objet requête `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — aucun appel `lighty.*` ne fonctionnerait.
- **nginx** (`lua-nginx-module` / OpenResty) : API différente (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), avec un modèle de phases d'exécution (`access_by_lua`, `content_by_lua`...) distinct du simple retour de code de `mod_magnet`.
- **Traefik** : aucun scripting Lua natif (écrit en Go) — seul un plugin Go (Yaegi) permettrait d'exécuter de la logique personnalisée, jamais du Lua directement.
- **Caddy** : aucun module Lua officiel — nécessiterait un plugin tiers compilé via `xcaddy`, rien d'équivalent disponible nativement.

Rendre ce script compatible avec l'un de ces serveurs demanderait d'écrire un pont de compatibilité dédié traduisant son API native vers `lighty.*` (exactement ce que fait `lua_waf_adapter.py` pour omega-serv) — un travail d'ingénierie à part entière, pas une simple adaptation de configuration.

---

## Personnalisation

- **Adapter à une autre application que SearXNG** : modifier `is_searxng_path()` (liste de chemins) et, si besoin, retirer entièrement la section 3 du pipeline (politique dédiée) si l'application protégée n'a pas besoin d'un traitement différencié des recherches.
- **Ajouter des chemins toujours autorisés** : éditer `waf_rules_dir/whitelist_paths.txt` (un motif Lua par ligne), ou ajouter une exception codée en dur dans `is_whitelisted_path()`.
- **Ajuster l'agressivité** : `waf_score_threshold` (seuil de refus POST), `suspicious_threshold`/`suspicious_window` (vitesse d'escalade vers un bannissement définitif), `global_rate_max_req`/`global_rate_window` (rate limit général).
- **Basculer les journaux en JSON** (ingestion SIEM/log shipper) : `use_json_logs = true`.
- **Désactiver un sous-système sans toucher au reste** : `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block` sont indépendants.

---

## Limites connues et hors périmètre

- **Pas de détection CMDi sur le chemin de l'URL** : choix délibéré pour éviter les faux positifs sur des chemins contenant `.php` (confondu avec `<?php` si la détection CMDi incluait la balise PHP sur cette surface). La détection CMDi reste active sur le corps POST.
- **Signatures par correspondance de motifs Lua, pas un vrai moteur PCRE** : des variantes d'attaque non couvertes par les motifs listés ne seront pas détectées — ceci n'est pas un remplacement d'un WAF à base de règles exhaustives (type CRS ModSecurity).
- **Pas de parsing multipart/form-data** : un payload malveillant encapsulé dans un champ de formulaire multipart peut échapper à l'inspection si les motifs ne correspondent pas au flux brut non décodé.
- **Pas de décompression gzip du corps** : un corps compressé n'est pas inspecté en clair.
- **Rate limiting approximatif** : fenêtre glissante simplifiée (réinitialisation complète au dépassement de la fenêtre), pas un algorithme de fenêtre glissante précise (sliding log/token bucket).
- **Aucune interface de supervision** : toute observation passe par la lecture directe des fichiers de log et de la liste noire.

---

## Dépannage

- **Le script ne semble jamais s'exécuter** : vérifier que `mod_magnet` est bien chargé (lighttpd) ou que l'extra `magnet` est installé et le WAF Magnet activé (omega-serv) ; un rechargement de configuration ne suffit jamais à charger un script Lua modifié, un redémarrage complet est nécessaire dans les deux cas.
- **Les compteurs de rate limit semblent se réinitialiser à chaque requête** : symptôme d'un runtime Lua non persistant entre les requêtes (le script repart d'un `rate_store` vide à chaque exécution) — sous omega-serv, vérifier que l'écran WAF Magnet signale bien un runtime persistant actif.
- **Une IP légitime se retrouve bannie** : consulter `blocklist_log` et `alerts_log` pour identifier la règle déclenchée, ajuster le motif ou le seuil correspondant, puis retirer la ligne de l'IP dans `blocklist_file` (le cache mémoire se met à jour au prochain rechargement, au plus `blocklist_refresh_interval` secondes après modification du fichier).
- **Les recherches SearXNG légitimes sont bloquées** : vérifier `is_searxng_path()` (chemin non reconnu) avant d'ajuster les seuils de rate limit dédiés.

---

## Licence

Omega-Waf est distribué sous licence **MIT**. Consultez le fichier [`LICENSE`](LICENSE) pour le texte complet.

---

> **Omega-Waf — Un script, une ligne de défense.**
>
> Un firewall applicatif mod_magnet en un seul fichier Lua, sans dépendance, compatible lighttpd et omega-serv.
