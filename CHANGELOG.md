<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

# Changelog

Toutes les modifications notables de ce projet sont documentées dans ce fichier.

Le format est basé sur [Keep a Changelog](https://keepachangelog.com/fr/1.0.0/),
et ce projet adhère au [Semantic Versioning](https://semver.org/lang/fr/).

Pour un script de sécurité comme celui-ci, une attention particulière est portée
aux changements de **signatures** (ajout/retrait/modification d'un motif de
détection) et de **comportement par défaut** (seuils, codes de réponse HTTP) —
ce sont les changements les plus susceptibles d'affecter une installation déjà
en production, donc toujours signalés explicitement, même mineurs.

## [Non publié]

### Ajouté
- `never_auto_block` : réseaux jamais bannis automatiquement (hôte local, Tailscale
  `100.64.0.0/10` et `fd7a:115c:a1e0::/48`, réseaux locaux IPv4/IPv6). Sous omega-serv,
  la liste est injectée par le serveur (table globale `omega_never_auto_block`, même
  réglage que son WAF natif) et prime sur la valeur du script.
- Comparaison CIDR IPv4/IPv6 en Lua portable (5.1 à 5.5, LuaJIT) : `::`, zone
  d'interface, forme IPv4-mappée-IPv6 comparée comme l'IPv4 qu'elle représente.
- Ligne `BAN EVITE` dans `blocklist_log` quand un ban est évité (jamais une ligne
  `AJOUT BLOCKLIST`, relue par omega-jail pour bannir au niveau réseau).

### Modifié
- **Chemin par défaut** : `blocklist_file` passe de `secure/waf/magnet-blocklist.txt` à
  `var/lib/waf/magnet-blocklist.txt` (le script écrit dans ce fichier ; sous omega-serv,
  seul `var/` est inscriptible par le service systemd, aucun ban n'était donc enregistré).
  omega-serv reprend l'ancien fichier une fois. En lighttpd natif, rien ne change si les
  chemins ont déjà été remplacés par des chemins absolus.
- README : section « Chemins en lighttpd natif » — chemins absolus, comptes
  `www-data`/`http`/`lighttpd`, tableau des droits (écriture pour la blocklist, les deux
  journaux et les compteurs ; lecture seule pour `waf_rules_dir`), échec d'écriture
  silencieux et commande de vérification, unité systemd restreinte.
- **Comportement par défaut** : une adresse de `never_auto_block` n'est plus jamais
  ajoutée à la liste noire ni escaladée par la récidive. La requête elle-même reste
  refusée et journalisée comme avant.

### Corrigé
- **Signatures de chemins sensibles** : `^/%phpmyadmin%..+`, `^/%adminer%..+`,
  `^/%wp%-admin%..+` et `^/%wp%-login%.php%..+` utilisaient par erreur les classes Lua
  `%p` (ponctuation), `%a` (lettre) et `%w` (alphanumérique) : `/phpmyadmin.bak` n'était
  jamais détecté, tandis que `/xdminer.old` ou `/1p-admin.zip` l'étaient. Motifs corrigés
  en `^/phpmyadmin%..+`, `^/adminer%..+`, `^/wp%-admin%..+`, `^/wp%-login%.php%..+`.
- Pied de page HTML de la page d'avertissement : balise `<div class="footer"` non fermée.
- Commentaire de la section CMDI abîmé par un collage accidentel, restauré.

### Limite connue (non modifiée)
- Le script utilise `goto continue` : il exige Lua 5.2 ou plus récent, ou LuaJIT
  (refusé par `luac5.1`). Les ajouts de cette version restent compatibles Lua 5.1.

---

## [1.0.0] - 2026-10-13

Première version packagée et distribuable (script + documentation + licence).
Correspond en interne à l'itération "v2, patterns nettoyés" mentionnée dans
l'en-tête du script (détection insensible à la casse gérée côté Lua plutôt que
par pseudo-flag PCRE).

### Ajouté
- Script `omega-waf.lua` autonome, compatible `mod_magnet` (lighttpd natif et
  omega-serv via `lua_waf_adapter.py`).
- Quatre familles de signatures : injection SQL (`sqli_body`), XSS (`xss_body`),
  injection de commandes (`cmdi_body`), traversée de répertoire
  (`path_traversal`).
- Liste dédiée de signatures User-Agent (scanners/bots connus : sqlmap, nmap,
  nikto, dirbuster, crawlers SEO, etc.).
- Surveillance d'une liste de chemins sensibles (`.git`, `.env`, `wp-admin`,
  `phpmyadmin`, etc.) avec détection de présence physique réelle.
- Politique dédiée, plus permissive, pour les routes d'un moteur de recherche
  SearXNG (évite les faux positifs sur des recherches légitimes).
- Rate limiting à deux niveaux (global et SearXNG), avec persistance des
  compteurs sur disque.
- Escalade progressive par IP (hits suspects → bannissement définitif après
  un seuil de récidive).
- Liste noire persistante avec exception pour les chemins explicitement
  whitelistés.
- Score cumulatif pour l'inspection du corps des requêtes POST.
- Journalisation au format texte ou JSON (`alerts_log`, `blocklist_log`).
- Documentation complète (`README.md`) en français, anglais, espagnol, russe
  et chinois simplifié.
- Licence MIT (`LICENSE`).

### Limites connues dès cette version
- Pas de détection d'injection de commandes sur le chemin de l'URL (choix
  délibéré, voir README § Limites connues).
- Pas de parsing multipart/form-data ni de décompression gzip du corps.
- Signatures par correspondance de motifs Lua, pas un moteur PCRE complet.

### Intégrité de l'archive distribuée
```
sha256sum omega-waf.tar.gz
771ba66b2995523cb0eccb5c8f38f44269a0d175d78f7e02e1dd54e1287a3bc5  omega-waf.tar.gz
```

[Non publié]: https://github.com/kraynux/omega-waf/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/kraynux/omega-waf/releases/tag/v1.0.0
