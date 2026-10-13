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
-

### Modifié
-

### Corrigé
-

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
