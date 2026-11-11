<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="512">
</div>

# 🔒 OMEGA-WAF

**Firewall de aplicação mod_magnet (script Lua autônomo)**

> Desenvolvido por [kraynux](https://github.com/kraynux) para [Omega-Server](https://kraynux.snake-mackarel.ts.net)   

Página oficial: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/pt-BR/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#compatibilidade)

🌐 Documentação disponível em **17 idiomas**.  
**Idiomas:**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](../it/README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](../de/README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](../nl/README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](../pl/README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](../vi/README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](../id/README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](../ja/README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](../ko/README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](../hi/README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

O **Omega-Waf** é um único script Lua (`omega-waf.lua`), sem dependência externa, feito para ser executado a cada requisição HTTP por um servidor web compatível com `mod_magnet` (o lighttpd nativamente, ou o omega-serv por meio de sua ponte de compatibilidade `magnet`). Ele inspeciona o caminho, a query e, em POST, o corpo da requisição em busca de assinaturas de ataque conhecidas (injeção de SQL, XSS, injeção de comandos, travessia de diretório), aplica limitação de taxa, mantém uma lista de bloqueio persistente com escalonamento progressivo e registra cada decisão em log.

Nenhuma instalação no sentido clássico: um único arquivo a copiar, referenciar na configuração do servidor, e pronto.

## Sumário

- [Apresentação](#apresentação)
- [Instalação](#instalação)
- [Configuração](#configuração)
- [Pipeline de decisão](#pipeline-de-decisão)
- [Assinaturas detectadas](#assinaturas-detectadas)
- [Caminhos sensíveis monitorados](#caminhos-sensíveis-monitorados)
- [Limitação de taxa e escalonamento](#limitação-de-taxa-e-escalonamento)
- [Persistência](#persistência)
- [Códigos de resposta HTTP emitidos](#códigos-de-resposta-http-emitidos)
- [Compatibilidade](#compatibilidade)
- [Personalização](#personalização)
- [Limitações conhecidas e fora do escopo](#limitações-conhecidas-e-fora-do-escopo)
- [Solução de problemas](#solução-de-problemas)
- [Licença](#licença)

---

## Apresentação

### Objetivos

- Filtrar as requisições maliciosas o mais perto possível do servidor, antes que cheguem à aplicação servida.
- Continuar sendo um arquivo único, legível, sem dependência externa nem etapa de compilação.
- Nunca derrubar o servidor web: todo erro interno ao script é capturado e registrado em log em vez de ser propagado.
- Limitar o custo de CPU/memória por requisição (caches temporais, leitura do corpo em blocos, profundidade de decodificação limitada).
- Oferecer um escalonamento progressivo (aviso e depois banimento) em vez de um banimento seco no primeiro incidente menor.

### O que o Omega-Waf faz

- Bloqueia os endereços IP já banidos (lista de bloqueio persistente em disco), com uma exceção para os caminhos explicitamente autorizados (CSS da página de erro etc.).
- Monitora um conjunto de caminhos sensíveis (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store` etc.) e responde com uma página de aviso HTML ou com um banimento direto em caso de reincidência.
- Detecta os User-Agents conhecidos como ferramentas de varredura ou bots indesejados (sqlmap, nmap, nikto, dirbuster, crawlers genéricos...).
- Aplica uma limitação de taxa global por IP, considerando todas as rotas.
- Aplica uma política dedicada, mais permissiva, às rotas de um mecanismo de busca SearXNG (evita falsos positivos em buscas legítimas que contêm palavras-chave parecidas com um ataque).
- Inspeciona o caminho e a query string (SQLi, XSS, travessia de diretório, byte NUL).
- Inspeciona o corpo das requisições POST (SQLi, XSS, injeção de comandos, travessia de diretório) com um sistema de pontuação cumulativa.
- Gerencia um escalonamento progressivo: um comportamento suspeito isolado dispara um alerta ou uma recusa pontual; a reincidência dentro de uma janela de tempo dispara um banimento definitivo.
- Registra em log cada decisão (alertas, inclusões na lista de bloqueio) em formato texto ou JSON.
- Funciona tanto no lighttpd nativo (caminhos fixos no código) quanto injetado no omega-serv (caminhos recebidos dinamicamente da configuração do servidor).

### O que o script não faz

- Ele não substitui um WAF dedicado de nível empresarial (ModSecurity, Coraza, um WAF em nuvem): as assinaturas são propositalmente simples (correspondência de padrões Lua, não um verdadeiro motor de regras PCRE).
- Ele não analisa os cabeçalhos HTTP um a um além do User-Agent.
- Ele não decodifica corpos multipart/form-data nem corpos compactados (gzip) — a inspeção se aplica ao texto bruto recebido.
- Ele nunca inspeciona o corpo de uma requisição que não seja POST.
- Ele não faz nenhuma detecção de injeção de comandos no caminho da URL (escolha deliberada, veja [Limitações conhecidas](#limitações-conhecidas-e-fora-do-escopo)).
- Ele não oferece nenhuma interface de administração: a configuração é feita editando as variáveis no início do arquivo, e a lista de bloqueio e a whitelist por meio de simples arquivos de texto.
- Ele não cuida sozinho da rotação dos próprios logs.

### Aviso de uso

As assinaturas fornecidas visam famílias de ataques comuns, mas não constituem uma proteção exaustiva. Uma pontuação de detecção baixa demais em um site com tráfego incomum (busca técnica, documentação contendo comandos de shell etc.) pode gerar falsos positivos — ajuste `waf_score_threshold` e a lista de caminhos permitidos de acordo. Sempre teste uma mudança de assinatura ou de limiar em um ambiente não crítico antes de implantá-la em produção.

---

## Instalação

### Pré-requisitos

- Um servidor web que exponha uma interface compatível com `mod_magnet`:
  - **lighttpd** com o módulo `mod_magnet` ativado (`server.modules += ("mod_magnet")`), ou
  - **omega-serv** com o extra `magnet` instalado (dependência `lupa` — veja abaixo e o guia do omega-serv, seção WAF Magnet).
- Um interpretador Lua embarcado fornecido pelo host (o lighttpd embarca seu próprio Lua; o omega-serv embarca o `lupa`, nenhum interpretador `lua` do sistema é necessário em nenhum dos dois casos — o `lupa` é distribuído como wheel pré-compilada para todas as arquiteturas/versões comuns do Python, sem compilação local nem cabeçalhos Lua do sistema).

#### Instalar o extra `magnet` do omega-serv

**Sempre a partir do `.venv` já criado pelo `omega-serv/install.sh`, nunca com um `pip` do sistema** — em algumas distribuições (em especial o Arch Linux), o comando `pip` puro não existe (só o `python-pip` o instala, e nem sempre por padrão) ou se recusa a instalar qualquer coisa fora de um ambiente virtual (`error: externally-managed-environment`, PEP 668):

```bash
cd /caminho/para/omega-serv

# Opcao A - venv ja ativado no shell atual
source .venv/bin/activate
pip install -e ".[magnet]"

# Opcao B - sem ativar o venv (o mais portavel: nunca depende
# de um 'pip' presente ou nao no PATH do sistema, ja que o python do
# venv sempre traz o seu proprio)
.venv/bin/python -m pip install -e ".[magnet]"
```

Se o `install.sh` ainda não foi executado (nenhum `.venv/` presente), execute-o primeiro — ele cria o ambiente virtual e já instala nele o próprio omega-serv; resta então apenas adicionar o extra `magnet` (opcional) com um dos dois comandos acima.

### Implantação

1. Copie `omega-waf.lua` para um local legível pelo servidor (ex.: `secure/waf/scripts/omega-waf.lua` para uma árvore no estilo omega-serv, ou qualquer caminho para o lighttpd nativo).
2. Crie os diretórios/arquivos esperados:
   - **omega-serv**: nada a fazer, o próprio servidor cria as pastas na inicialização (incluindo `var/lib/waf/`, compartilhada com a conta do serviço).
   - **lighttpd nativo**: primeiro substitua os 5 caminhos por caminhos absolutos (veja [Caminhos no lighttpd nativo](#caminhos-no-lighttpd-nativo)), depois crie as pastas e torne-as graváveis pela conta do lighttpd (`www-data` no Debian/Ubuntu, `http` no Arch, `lighttpd` no Fedora/Alpine):
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. Referencie o script na configuração do servidor:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/caminho/para/omega-waf.lua" )
     ```
   - **omega-serv**: ative o WAF Magnet na tela TUI dedicada (Segurança ativa → WAF → Magnet), que aponta para o script e injeta automaticamente a tabela `omega_magnet_paths` (veja [Compatibilidade](#compatibilidade)).
4. Reinicie (não recarregue — uma recarga nunca recarrega um script Lua já em memória persistente) o servidor em questão.

---

## Configuração

Todas as opções são ajustadas modificando diretamente as variáveis locais no início do arquivo (seção `CONFIGURATION`, linhas ~24-70). Nenhum arquivo de configuração separado.

| Variável | Função | Valor padrão |
|---|---|---|
| `blocklist_file` | Arquivo de texto dos IPs banidos definitivamente (um IP por linha, `#` para comentário) — **escrito** pelo script | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Pasta das regras externas (contém `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Log dos alertas (detecções, recusas, excessos) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Log dedicado às inclusões na lista de bloqueio | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | Arquivo de persistência dos contadores (limite de taxa / reincidência), formato tabela Lua | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Ativa a inspeção do corpo das requisições POST | `true` |
| `enable_rate_limit` | Ativa a limitação de taxa (global + SearXNG) | `true` |
| `enable_auto_block` | Permite a inclusão automática na lista de bloqueio | `true` |
| `never_auto_block` | Redes **nunca** banidas automaticamente (a requisição continua recusada e registrada em log, apenas o banimento é evitado); no omega-serv, a lista injetada pelo servidor (`omega_never_auto_block`) sempre prevalece | host local, Tailscale, redes locais |
| `use_json_logs` | Muda os logs de alerta para JSON (caso contrário, texto legível) | `false` |
| `waf_score_threshold` | Pontuação mínima acumulada para que uma requisição POST seja recusada | `5` |
| `body_max_inspect` | Tamanho máximo inspecionado do corpo (bytes) | `65536` (64 KB) |
| `chunk_size` | Tamanho dos blocos de leitura do corpo | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | Cota e janela (s) da limitação de taxa global por IP | `60` requisições / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | Cota e janela (s) da limitação de taxa dedicada às rotas SearXNG | `40` requisições / `60` s |
| `searx_max_query_len` | Tamanho máximo aceito da query string nas rotas SearXNG | `4096` caracteres |
| `suspicious_threshold` | Número de hits suspeitos antes do banimento definitivo | `3` |
| `suspicious_window` | Janela deslizante (s) na qual os hits suspeitos são contados | `300` (5 min) |
| `blocklist_refresh_interval` | Intervalo (s) de releitura do arquivo da lista de bloqueio | `5` |
| `whitelist_refresh_interval` | Intervalo (s) de releitura da whitelist de caminhos | `15` |

No omega-serv, os 5 primeiros caminhos (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) são substituídos automaticamente pelos configurados na tela WAF Magnet (tabela global `omega_magnet_paths` injetada antes da execução) — os valores fixos acima servem então apenas de fallback se essa tabela estiver ausente.

> ℹ️ **Mudança de 2026-10-18**: `blocklist_file` passa de `secure/waf/magnet-blocklist.txt` para `var/lib/waf/magnet-blocklist.txt`. O script **escreve** nesse arquivo a cada banimento, e o serviço systemd do omega-serv só pode escrever em `var/`: com o local antigo, nenhum banimento automático jamais era registrado, sem nenhuma mensagem. O omega-serv recupera o arquivo antigo uma única vez na inicialização (o antigo permanece no lugar como backup); um caminho personalizado na tela WAF Magnet nunca é modificado.

### Caminhos no lighttpd nativo

> ⚠️ **Esses 5 valores padrão são calcados na convenção de pastas do omega-serv** (`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, caminhos relativos resolvidos em relação a um diretório de trabalho do tipo projeto) — eles não são neutros. No **lighttpd nativo**, onde não existe nenhuma convenção de pastas equivalente, esses caminhos relativos são resolvidos em relação ao diretório de trabalho do processo lighttpd em tempo de execução (muitas vezes `/`), raramente previsível. **Substitua essas 5 linhas por caminhos absolutos** adequados à sua instalação, por exemplo:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> Antes de reiniciar, crie essas pastas (comandos na etapa 2 da [Implantação](#implantação)) respeitando estas permissões:
>
> | Arquivo | Acesso necessário à conta do lighttpd |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **escrita** (o arquivo e a pasta que o contém) |
> | `waf_rules_dir` (e `whitelist_paths.txt`) | somente leitura |
>
> Uma falha de escrita é **silenciosa** — o script nunca quebra a requisição —: um banimento que não aparece em `blocklist_file` é o único sintoma. Verifique com `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` (adapte a conta). Se a unidade systemd do lighttpd restringir as escritas (`ProtectSystem=strict`, `ReadWritePaths=`), adicione essas pastas a ela.

---

## Pipeline de decisão

O script é executado inteiramente dentro de um `pcall` global: todo erro Lua imprevisto é registrado em log e responde `500`, sem nunca derrubar o processo do servidor. As etapas são executadas nesta ordem, e cada uma pode interromper o processamento com uma resposta final:

1. **Lista de bloqueio** — se o IP estiver banido, a requisição é recusada (`451`), exceto se o caminho solicitado estiver explicitamente na whitelist (serve, por exemplo, a própria página de erro).
2. **Caminhos sensíveis** — se o caminho corresponder a um padrão monitorado (`.git`, `.env`, `wp-admin`...), uma página de aviso HTML é devolvida (`403`) na primeira vez; a reincidência dentro da janela de suspeita dispara um banimento (`451`).
3. **User-Agent de scanner** — se o User-Agent corresponder a uma ferramenta de varredura/bot conhecida, `403` na primeira vez, `451` em caso de reincidência.
4. **Limite de taxa global** — se a cota global por IP for excedida, `429`.
5. **Política SearXNG** — para as rotas de busca reconhecidas: limite de taxa dedicado (`429`), proteção contra o tamanho da query string (`414`), rejeição de bytes NUL (`451`), e depois saída imediata (`0`, continua) para nunca submeter uma busca legítima às verificações agressivas seguintes.
6. **Proteção do caminho** — busca de SQLi e XSS no caminho decodificado (`451` se detectado); busca de travessia de diretório (`451`). Propositalmente **sem** detecção de injeção de comandos nesta etapa (veja [Limitações conhecidas](#limitações-conhecidas-e-fora-do-escopo)).
7. **Proteção leve da query string** (fora do SearXNG) — travessia de diretório e byte NUL (`451`).
8. **Inspeção do corpo POST** — somente se `enable_body_inspection` estiver ativo e o método for `POST`: corpo grande demais recusado diretamente (`413`); caso contrário, pontuação acumulada sobre SQLi/XSS/CMDi/travessia, recusa (`451`) se a pontuação atingir `waf_score_threshold`, com banimento imediato se a pontuação atingir 8, senão apenas um hit suspeito.
9. **Nenhum bloqueio** — a requisição continua normalmente (`0`).

---

## Assinaturas detectadas

Quatro famílias de padrões (`patterns`), sendo cada entrada uma string (sensível a maiúsculas e minúsculas) ou uma tabela `{ pat = "motif", ci = true }` (insensível a maiúsculas e minúsculas):

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, injeções de comandos SQL via ponto e vírgula (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: tags `<script>`, `<iframe>`, atributos `on*=` (manipuladores de eventos), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: encadeamento de comandos shell (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, variantes com `|`), substituição de comando (`` `...` ``, `$(...)`), tag PHP (`<?php`) e funções de execução (`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

E uma lista dedicada de **User-Agents de scanners/bots** (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, crawlers de SEO (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`) e padrões genéricos (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

A decodificação de URL é aplicada recursivamente (até 5 níveis no caminho, 2 níveis na query string) antes da inspeção, para pegar payloads com dupla codificação.

---

## Caminhos sensíveis monitorados

Padrões verificados no caminho decodificado: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

Para cada correspondência, o script tenta determinar se um recurso físico existe de fato nesse caminho (via `lighty.stat`, com fallback para `lighty.c.stat` por compatibilidade) e o indica na página de aviso e no log — útil para distinguir uma varredura às cegas de um caminho que existe de verdade no servidor.

---

## Limitação de taxa e escalonamento

Dois limitadores de taxa independentes, baseados em contadores em memória persistidos em disco (janela deslizante simples, não uma janela deslizante precisa ao segundo):

- **Global**: chave `g:<ip>`, `global_rate_max_req` requisições a cada `global_rate_window` segundos, considerando todas as rotas.
- **SearXNG**: chave `sx:<ip>`, `searx_rate_max_req` requisições a cada `searx_rate_window` segundos, apenas nas rotas de busca reconhecidas.

O escalonamento progressivo (`record_suspicious_hit`) é um mecanismo separado da limitação de taxa: cada detecção não crítica (caminho sensível sem reincidência, User-Agent de scanner, hit menor no corpo POST) incrementa um contador por IP (chave `susp:<ip>`) em uma janela de `suspicious_window` segundos. Quando `suspicious_threshold` hits são atingidos nessa janela, o IP é adicionado à lista de bloqueio definitiva — evita um banimento seco por um único falso positivo, sem deixar de banir a reincidência.

**Redes nunca banidas**: um endereço de `never_auto_block` (host local, Tailscale `100.64.0.0/10`, redes locais IPv4/IPv6, incluindo a forma IPv4 mapeada em IPv6) nunca dispara escalonamento nem inclusão na lista de bloqueio; um banimento evitado é registrado por uma linha `BAN EVITE` em `blocklist_log` (nunca `AJOUT BLOCKLIST`, relida pelo omega-jail). Comparação CIDR em Lua portável, do 5.1 ao 5.5 e LuaJIT.

**Modo log-only do omega-serv**: quando o WAF do omega-serv está em `log-only`, o script não aplica nenhum banimento (nem lista de bloqueio, nem linha `AJOUT BLOCKLIST`, portanto nenhum banimento de rede pelo fail2ban); uma linha `BAN EVITE` o registra. Se a lista de bloqueio não puder ser gravada, uma linha `ECHEC BLOCKLIST` substitui `AJOUT BLOCKLIST`.

---

## Persistência

| Arquivo | Conteúdo | Formato |
|---|---|---|
| `blocklist_file` | Um IP banido por linha, precedido de um comentário com data e hora (`# date - raison`) | Texto puro |
| `waf_rules_dir/whitelist_paths.txt` | Um padrão Lua (`string.match`) por linha, caminhos sempre autorizados mesmo para um IP banido | Texto puro |
| `rate_store_file` | Contadores de limitação de taxa e de reincidência (`{ count = N, first = timestamp }` por chave) | Tabela Lua serializada (`return { ... }`, relida via `load()`) |
| `alerts_log` | Uma linha por detecção/recusa | Texto legível, ou JSON se `use_json_logs = true` |
| `blocklist_log` | Uma linha por inclusão na lista de bloqueio | Texto legível |

O arquivo de limitação de taxa nunca é reescrito a cada requisição (no máximo a cada 300 segundos, durante a limpeza periódica) para evitar um custo de I/O desproporcional. A lista de bloqueio e a whitelist são mantidas em cache na memória e relidas no máximo a cada `blocklist_refresh_interval`/`whitelist_refresh_interval` segundos — uma inclusão disparada pelo próprio script atualiza o cache imediatamente, sem esperar esse intervalo.

---

## Códigos de resposta HTTP emitidos

| Código | Significado | Gatilho |
|---|---|---|
| `0` | Continua o processamento normal (convenção do `mod_magnet`) | Nenhuma detecção |
| `403` | Acesso negado, primeiro aviso | Caminho sensível detectado (primeira vez), User-Agent de scanner (primeira vez) |
| `413` | Entidade grande demais | Corpo POST acima de `body_max_inspect` |
| `414` | URI longa demais | Query string SearXNG acima de `searx_max_query_len` |
| `429` | Requisições demais | Limite de taxa global ou SearXNG excedido |
| `451` | Indisponível por motivos legais (usado aqui como código de banimento) | IP já banido, reincidência após escalonamento, SQLi/XSS/travessia detectada no caminho ou na query string, byte NUL, pontuação POST ≥ limiar |
| `500` | Erro interno | Exceção Lua imprevista (capturada pelo `pcall` global) |

---

## Compatibilidade

O script funciona em **dois contextos**, sem nenhuma modificação:

- **lighttpd nativo**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` são as APIs nativas do módulo `mod_magnet`. Os 5 caminhos de configuração usam diretamente seus valores padrão fixos no código.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): uma ponte Python-Lua (`lupa`) reproduz fielmente o mesmo contrato `lighty.*`, com um runtime Lua **persistente** entre as requisições (os caches temporais e a limitação de taxa sobrevivem, portanto, corretamente de uma requisição para outra, como no lighttpd). A tabela global `omega_magnet_paths` é injetada antes da execução a partir da configuração da tela WAF Magnet, para que os 5 caminhos apontem para os locais escolhidos na interface em vez dos valores fixos.

Nenhuma divergência de lógica de negócio entre os dois contextos — só muda a origem dos 5 caminhos de configuração.

### O que NÃO é compatível como está

O script depende inteiramente do contrato `lighty.*` (API nativa do `mod_magnet`) — ele **não** roda como está em outro servidor, mesmo que este ofereça alguma forma de scripting Lua:

- **Apache** (`mod_lua`): API diferente, baseada em um objeto de requisição `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — nenhuma chamada `lighty.*` funcionaria.
- **nginx** (`lua-nginx-module` / OpenResty): API diferente (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), com um modelo de fases de execução (`access_by_lua`, `content_by_lua`...) distinto do simples retorno de código do `mod_magnet`.
- **Traefik**: nenhum scripting Lua nativo (escrito em Go) — só um plugin Go (Yaegi) permitiria executar lógica personalizada, nunca Lua diretamente.
- **Caddy**: nenhum módulo Lua oficial — exigiria um plugin de terceiros compilado via `xcaddy`, nada equivalente disponível nativamente.

Tornar este script compatível com um desses servidores exigiria escrever uma ponte de compatibilidade dedicada que traduzisse sua API nativa para `lighty.*` (exatamente o que o `lua_waf_adapter.py` faz para o omega-serv) — um trabalho de engenharia completo, não uma simples adaptação de configuração.

---

## Personalização

- **Adaptar a outra aplicação que não o SearXNG**: modifique `is_searxng_path()` (lista de caminhos) e, se necessário, remova inteiramente a seção 3 do pipeline (política dedicada) se a aplicação protegida não precisar de um tratamento diferenciado das buscas.
- **Adicionar caminhos sempre autorizados**: edite `waf_rules_dir/whitelist_paths.txt` (um padrão Lua por linha), ou adicione uma exceção fixa no código em `is_whitelisted_path()`.
- **Ajustar a agressividade**: `waf_score_threshold` (limiar de recusa POST), `suspicious_threshold`/`suspicious_window` (velocidade do escalonamento até um banimento definitivo), `global_rate_max_req`/`global_rate_window` (limite de taxa geral).
- **Mudar os logs para JSON** (ingestão por SIEM/log shipper): `use_json_logs = true`.
- **Desativar um subsistema sem mexer no resto**: `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block` são independentes.

---

## Limitações conhecidas e fora do escopo

- **Sem detecção de CMDi no caminho da URL**: escolha deliberada para evitar falsos positivos em caminhos que contêm `.php` (confundido com `<?php` se a detecção de CMDi incluísse a tag PHP nessa superfície). A detecção de CMDi continua ativa no corpo POST.
- **Assinaturas por correspondência de padrões Lua, não um verdadeiro motor PCRE**: variantes de ataque não cobertas pelos padrões listados não serão detectadas — isto não substitui um WAF baseado em regras exaustivas (tipo CRS do ModSecurity).
- **Sem parsing de multipart/form-data**: um payload malicioso encapsulado em um campo de formulário multipart pode escapar da inspeção se os padrões não corresponderem ao fluxo bruto não decodificado.
- **Sem descompactação gzip do corpo**: um corpo compactado não é inspecionado em texto claro.
- **Limitação de taxa aproximada**: janela deslizante simplificada (reinicialização completa ao ultrapassar a janela), não um algoritmo de janela deslizante precisa (sliding log/token bucket).
- **Nenhuma interface de supervisão**: toda observação passa pela leitura direta dos arquivos de log e da lista de bloqueio.

---

## Solução de problemas

- **O script parece nunca ser executado**: verifique se o `mod_magnet` está realmente carregado (lighttpd) ou se o extra `magnet` está instalado e o WAF Magnet ativado (omega-serv); uma recarga da configuração nunca basta para carregar um script Lua modificado, uma reinicialização completa é necessária nos dois casos.
- **Os contadores de limite de taxa parecem ser zerados a cada requisição**: sintoma de um runtime Lua não persistente entre as requisições (o script parte de um `rate_store` vazio a cada execução) — no omega-serv, verifique se a tela WAF Magnet indica um runtime persistente ativo.
- **Um IP legítimo acaba banido**: consulte `blocklist_log` e `alerts_log` para identificar a regra disparada, ajuste o padrão ou o limiar correspondente e depois remova a linha do IP em `blocklist_file` (o cache em memória é atualizado na próxima releitura, no máximo `blocklist_refresh_interval` segundos após a modificação do arquivo).
- **As buscas legítimas no SearXNG são bloqueadas**: verifique `is_searxng_path()` (caminho não reconhecido) antes de ajustar os limiares de limite de taxa dedicados.
- **`pip install -e ".[magnet]"` falha com `pip: comando não encontrado` ou `externally-managed-environment`**: o comando foi digitado fora do `.venv` do omega-serv (algumas distribuições, em especial o Arch Linux, não expõem nenhum `pip` do sistema utilizável diretamente). Use `.venv/bin/python -m pip install -e ".[magnet]"` a partir do diretório do omega-serv — funciona sem depender de um `pip` presente no `PATH` do sistema (veja [Instalar o extra `magnet` do omega-serv](#instalar-o-extra-magnet-do-omega-serv)).

---

## Licença

O Omega-Waf é distribuído sob a licença **MIT**. Consulte o arquivo [`LICENSE`](../../../LICENSE) para o texto completo.

---

> **Omega-Waf — Um script, uma linha de defesa.**
>
> Um firewall de aplicação mod_magnet em um único arquivo Lua, sem dependências, compatível com lighttpd e omega-serv.
