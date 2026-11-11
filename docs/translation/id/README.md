<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="512">
</div>

# 🔒 OMEGA-WAF

**Firewall aplikasi mod_magnet (skrip Lua mandiri)**

> Dibuat oleh [kraynux](https://github.com/kraynux) untuk [Omega-Server](https://kraynux.snake-mackarel.ts.net)   

Halaman resmi: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/id/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#kompatibilitas)

🌐 Dokumentasi tersedia dalam **17 bahasa**.  
**Bahasa:**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](../pt-BR/README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](../it/README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](../de/README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](../nl/README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](../pl/README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](../vi/README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](../ja/README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](../ko/README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](../hi/README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

**Omega-Waf** adalah satu skrip Lua (`omega-waf.lua`), tanpa dependensi eksternal, yang dimaksudkan untuk dijalankan pada setiap permintaan HTTP oleh server web yang kompatibel dengan `mod_magnet` (lighttpd secara native, atau omega-serv melalui jembatan kompatibilitas `magnet`-nya). Skrip ini memeriksa jalur, permintaan, dan, pada POST, badan permintaan untuk mencari tanda tangan serangan yang dikenal (injeksi SQL, XSS, injeksi perintah, path traversal), menerapkan rate limiting, mengelola daftar blokir persisten dengan eskalasi bertahap, dan mencatat setiap keputusan ke log.

Tidak ada instalasi dalam arti klasik: cukup satu berkas yang diletakkan, dirujuk dari konfigurasi server, dan selesai.

## Daftar isi

- [Ikhtisar](#ikhtisar)
- [Instalasi](#instalasi)
- [Konfigurasi](#konfigurasi)
- [Pipeline keputusan](#pipeline-keputusan)
- [Tanda tangan yang dideteksi](#tanda-tangan-yang-dideteksi)
- [Jalur sensitif yang dipantau](#jalur-sensitif-yang-dipantau)
- [Rate limiting dan eskalasi](#rate-limiting-dan-eskalasi)
- [Persistensi](#persistensi)
- [Kode respons HTTP yang dikirim](#kode-respons-http-yang-dikirim)
- [Kompatibilitas](#kompatibilitas)
- [Kustomisasi](#kustomisasi)
- [Batasan yang diketahui dan di luar cakupan](#batasan-yang-diketahui-dan-di-luar-cakupan)
- [Pemecahan masalah](#pemecahan-masalah)
- [Lisensi](#lisensi)

---

## Ikhtisar

### Tujuan

- Menyaring permintaan berbahaya sedekat mungkin dengan server, sebelum mencapai aplikasi yang disajikan.
- Tetap menjadi satu berkas yang mudah dibaca, tanpa dependensi eksternal maupun tahap kompilasi.
- Tidak pernah menjatuhkan server web: setiap kesalahan internal skrip ditangkap dan dicatat ke log alih-alih diteruskan.
- Membatasi biaya CPU/memori per permintaan (cache berbasis waktu, pembacaan badan per potongan, kedalaman dekode terbatas).
- Menawarkan eskalasi bertahap (peringatan lalu blokir) alih-alih blokir langsung pada insiden kecil pertama.

### Yang dilakukan Omega-Waf

- Memblokir alamat IP yang sudah diblokir (daftar blokir persisten di disk), dengan pengecualian untuk jalur yang diizinkan secara eksplisit (CSS halaman kesalahan, dll.).
- Memantau sekumpulan jalur sensitif (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store`, dll.) dan merespons dengan halaman peringatan HTML, atau dengan blokir langsung jika terjadi residivisme.
- Mendeteksi User-Agent yang dikenal sebagai alat pemindai atau bot yang tidak diinginkan (sqlmap, nmap, nikto, dirbuster, crawler generik...).
- Menerapkan rate limiting global per IP, untuk semua rute.
- Menerapkan kebijakan khusus yang lebih longgar pada rute mesin pencari SearXNG (menghindari positif palsu pada kueri pencarian sah yang berisi kata kunci yang menyerupai serangan).
- Memeriksa jalur dan query string (SQLi, XSS, path traversal, byte NUL).
- Memeriksa badan permintaan POST (SQLi, XSS, injeksi perintah, path traversal) dengan sistem skor kumulatif.
- Mengelola eskalasi bertahap: perilaku mencurigakan yang terisolasi memicu peringatan atau penolakan sekali; residivisme dalam rentang waktu tertentu memicu blokir permanen.
- Mencatat setiap keputusan (peringatan, penambahan ke daftar blokir) dalam format teks atau JSON.
- Berfungsi baik di lighttpd native (jalur ditulis langsung di kode) maupun saat disisipkan ke omega-serv (jalur diterima secara dinamis dari konfigurasi server).

### Yang tidak dilakukan skrip ini

- Skrip ini tidak menggantikan WAF khusus kelas perusahaan (ModSecurity, Coraza, WAF cloud): tanda tangannya sengaja dibuat sederhana (pencocokan pola Lua, bukan mesin aturan PCRE sungguhan).
- Skrip ini tidak menganalisis header HTTP satu per satu di luar User-Agent.
- Skrip ini tidak mendekode badan multipart/form-data maupun badan terkompresi (gzip) — pemeriksaan dilakukan pada teks mentah yang diterima.
- Skrip ini tidak pernah memeriksa badan permintaan selain POST.
- Skrip ini tidak melakukan deteksi injeksi perintah pada jalur URL (pilihan yang disengaja, lihat [Batasan yang diketahui](#batasan-yang-diketahui-dan-di-luar-cakupan)).
- Skrip ini tidak menyediakan antarmuka administrasi: konfigurasi dilakukan dengan mengedit variabel di bagian atas berkas, daftar blokir dan whitelist melalui berkas teks biasa.
- Skrip ini tidak menangani sendiri rotasi log-nya.

### Peringatan penggunaan

Tanda tangan yang disediakan menyasar keluarga serangan umum, tetapi tidak merupakan perlindungan yang menyeluruh. Ambang skor deteksi yang terlalu rendah pada situs dengan lalu lintas tidak biasa (pencarian teknis, dokumentasi yang berisi perintah shell, dll.) dapat menimbulkan positif palsu — sesuaikan `waf_score_threshold` dan whitelist jalur sesuai kebutuhan. Selalu uji perubahan tanda tangan atau ambang di lingkungan yang tidak kritis sebelum menerapkannya di produksi.

---

## Instalasi

### Prasyarat

- Server web yang menyediakan antarmuka yang kompatibel dengan `mod_magnet`:
  - **lighttpd** dengan modul `mod_magnet` aktif (`server.modules += ("mod_magnet")`), atau
  - **omega-serv** dengan extra `magnet` terpasang (dependensi `lupa` — lihat di bawah dan panduan omega-serv, bagian WAF Magnet).
- Interpreter Lua tertanam yang disediakan oleh host (lighttpd menyertakan Lua-nya sendiri; omega-serv menyertakan `lupa`, tidak memerlukan interpreter `lua` sistem dalam kedua kasus — `lupa` didistribusikan sebagai wheel prakompilasi untuk semua arsitektur/versi Python yang umum, tanpa kompilasi lokal maupun header Lua sistem).

#### Memasang extra `magnet` omega-serv

**Selalu dari `.venv` yang sudah dibuat oleh `omega-serv/install.sh`, jangan pernah dengan `pip` sistem** — pada beberapa distribusi (terutama Arch Linux), perintah `pip` polos tidak ada (hanya paket `python-pip` yang memasangnya, dan tidak selalu secara bawaan) atau menolak memasang apa pun di luar lingkungan virtual (`error: externally-managed-environment`, PEP 668):

```bash
cd /path/to/omega-serv

# Opsi A - venv sudah aktif di shell saat ini
source .venv/bin/activate
pip install -e ".[magnet]"

# Opsi B - tanpa mengaktifkan venv (paling portabel: tidak pernah bergantung
# pada ada tidaknya 'pip' di PATH sistem, karena python milik
# venv selalu menyertakan pip-nya sendiri)
.venv/bin/python -m pip install -e ".[magnet]"
```

Jika `install.sh` belum dijalankan (belum ada `.venv/`), jalankan terlebih dahulu — skrip itu membuat lingkungan virtual dan sudah memasang omega-serv sendiri di dalamnya; tinggal extra `magnet` (opsional) yang perlu ditambahkan dengan salah satu dari dua perintah di atas.

### Penyiapan

1. Salin `omega-waf.lua` ke lokasi yang dapat dibaca oleh server (mis. `secure/waf/scripts/omega-waf.lua` untuk struktur direktori ala omega-serv, atau jalur mana pun untuk lighttpd native).
2. Buat direktori/berkas yang diperlukan:
   - **omega-serv**: tidak perlu melakukan apa pun, server membuat sendiri foldernya saat mulai (termasuk `var/lib/waf/`, yang dibagi dengan akun layanan).
   - **lighttpd native**: ganti dulu 5 jalur dengan jalur absolut (lihat [Jalur pada lighttpd native](#jalur-pada-lighttpd-native)), lalu buat foldernya dan jadikan dapat ditulis oleh akun lighttpd (`www-data` di Debian/Ubuntu, `http` di Arch, `lighttpd` di Fedora/Alpine):
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. Rujuk skrip dari konfigurasi server:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/path/to/omega-waf.lua" )
     ```
   - **omega-serv**: aktifkan WAF Magnet dari layar TUI khusus (Keamanan Aktif → WAF → Magnet), yang menunjuk ke skrip dan secara otomatis menyisipkan tabel `omega_magnet_paths` (lihat [Kompatibilitas](#kompatibilitas)).
4. Mulai ulang (bukan muat ulang — muat ulang tidak pernah memuat ulang skrip Lua yang sudah berada di memori persisten) server yang bersangkutan.

---

## Konfigurasi

Semua opsi diatur dengan mengedit langsung variabel lokal di bagian atas berkas (bagian `CONFIGURATION`, baris ~24-70). Tidak ada berkas konfigurasi terpisah.

| Variabel | Peran | Nilai bawaan |
|---|---|---|
| `blocklist_file` | Berkas teks IP yang diblokir permanen (satu IP per baris, `#` untuk komentar) — **ditulis** oleh skrip | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Folder aturan eksternal (berisi `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Log peringatan (deteksi, penolakan, pelampauan) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Log khusus untuk penambahan ke daftar blokir | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | Berkas persistensi penghitung (rate limit / residivisme), format tabel Lua | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Mengaktifkan pemeriksaan badan permintaan POST | `true` |
| `enable_rate_limit` | Mengaktifkan rate limiting (global + SearXNG) | `true` |
| `enable_auto_block` | Mengizinkan penambahan otomatis ke daftar blokir | `true` |
| `never_auto_block` | Jaringan yang **tidak pernah** diblokir otomatis (permintaan tetap ditolak dan dicatat, hanya blokirnya yang dihindari); di omega-serv, daftar yang disisipkan oleh server (`omega_never_auto_block`) selalu diutamakan | host lokal, Tailscale, jaringan lokal |
| `use_json_logs` | Mengalihkan log peringatan ke JSON (jika tidak, teks yang mudah dibaca) | `false` |
| `waf_score_threshold` | Skor kumulatif minimum agar permintaan POST ditolak | `5` |
| `body_max_inspect` | Ukuran maksimum badan yang diperiksa (byte) | `65536` (64 KB) |
| `chunk_size` | Ukuran potongan pembacaan badan | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | Kuota dan jendela (dtk) rate limiting global per IP | `60` permintaan / `60` dtk |
| `searx_rate_max_req` / `searx_rate_window` | Kuota dan jendela (dtk) rate limiting khusus rute SearXNG | `40` permintaan / `60` dtk |
| `searx_max_query_len` | Ukuran maksimum query string yang diterima pada rute SearXNG | `4096` karakter |
| `suspicious_threshold` | Jumlah hit mencurigakan sebelum blokir permanen | `3` |
| `suspicious_window` | Jendela geser (dtk) tempat hit mencurigakan dihitung | `300` (5 mnt) |
| `blocklist_refresh_interval` | Interval (dtk) pembacaan ulang berkas daftar blokir | `5` |
| `whitelist_refresh_interval` | Interval (dtk) pembacaan ulang whitelist jalur | `15` |

Di omega-serv, 5 jalur pertama (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) otomatis digantikan oleh jalur yang dikonfigurasi di layar WAF Magnet (tabel global `omega_magnet_paths` yang disisipkan sebelum eksekusi) — nilai tetap di atas hanya berfungsi sebagai cadangan jika tabel tersebut tidak ada.

> ℹ️ **Perubahan 2026-10-18**: `blocklist_file` berpindah dari `secure/waf/magnet-blocklist.txt` ke `var/lib/waf/magnet-blocklist.txt`. Skrip **menulis** ke berkas ini pada setiap blokir, dan layanan systemd omega-serv hanya dapat menulis di bawah `var/`: dengan lokasi lama, tidak ada blokir otomatis yang pernah tercatat, tanpa pesan apa pun. omega-serv mengambil alih berkas lama satu kali saat mulai (berkas lama tetap ada sebagai cadangan); jalur khusus di layar WAF Magnet tidak pernah diubah.

### Jalur pada lighttpd native

> ⚠️ **Kelima nilai bawaan ini mengikuti konvensi folder omega-serv** (`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, jalur relatif yang diselesaikan terhadap direktori kerja bergaya proyek) — nilai ini tidak netral. Pada **lighttpd native**, yang tidak memiliki konvensi folder serupa, jalur relatif ini diselesaikan terhadap direktori kerja proses lighttpd saat berjalan (sering kali `/`), yang jarang dapat diprediksi. **Ganti kelima baris ini dengan jalur absolut** yang sesuai dengan instalasi Anda, misalnya:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> Sebelum memulai ulang, buat folder-folder ini (perintah di langkah 2 bagian [Penyiapan](#penyiapan)) dengan hak akses berikut:
>
> | Berkas | Akses yang diperlukan akun lighttpd |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **tulis** (berkas dan folder yang memuatnya) |
> | `waf_rules_dir` (dan `whitelist_paths.txt`) | hanya baca |
>
> Kegagalan menulis bersifat **senyap** — skrip tidak pernah merusak permintaan —: blokir yang tidak muncul di `blocklist_file` adalah satu-satunya gejala. Periksa dengan `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` (sesuaikan akunnya). Jika unit systemd lighttpd membatasi penulisan (`ProtectSystem=strict`, `ReadWritePaths=`), tambahkan folder-folder ini ke dalamnya.

---

## Pipeline keputusan

Skrip berjalan sepenuhnya di dalam `pcall` global: setiap kesalahan Lua yang tidak terduga dicatat ke log lalu dijawab dengan `500`, tanpa pernah membuat proses server macet. Langkah-langkah dijalankan dalam urutan ini, dan masing-masing dapat menghentikan pemrosesan dengan respons akhir:

1. **Daftar blokir** — jika IP diblokir, permintaan ditolak (`451`), kecuali jalur yang diminta secara eksplisit ada di whitelist (misalnya untuk menyajikan halaman kesalahan itu sendiri).
2. **Jalur sensitif** — jika jalur cocok dengan pola yang dipantau (`.git`, `.env`, `wp-admin`...), halaman peringatan HTML dikembalikan (`403`) pada kali pertama; residivisme dalam jendela kecurigaan memicu blokir (`451`).
3. **User-Agent pemindai** — jika User-Agent cocok dengan alat pemindai/bot yang dikenal, `403` pada kali pertama, `451` jika terjadi residivisme.
4. **Rate limit global** — jika kuota global per IP terlampaui, `429`.
5. **Kebijakan SearXNG** — untuk rute pencarian yang dikenali: rate limit khusus (`429`), pengaman ukuran query string (`414`), penolakan byte NUL (`451`), lalu keluar segera (`0`, lanjut) agar pencarian yang sah tidak pernah melewati pemeriksaan agresif berikutnya.
6. **Perlindungan jalur** — pencarian SQLi dan XSS di jalur yang sudah didekodekan (`451` jika terdeteksi); pencarian path traversal (`451`). Sengaja **tidak** ada deteksi injeksi perintah pada tahap ini (lihat [Batasan yang diketahui](#batasan-yang-diketahui-dan-di-luar-cakupan)).
7. **Perlindungan ringan query string** (di luar SearXNG) — path traversal dan byte NUL (`451`).
8. **Pemeriksaan badan POST** — hanya jika `enable_body_inspection` aktif dan metodenya `POST`: badan yang terlalu besar langsung ditolak (`413`); jika tidak, skor kumulatif atas SQLi/XSS/CMDi/traversal, ditolak (`451`) jika skor mencapai `waf_score_threshold`, dengan blokir segera jika skor mencapai 8, jika tidak hanya dihitung sebagai hit mencurigakan.
9. **Tidak ada blokir** — permintaan berlanjut seperti biasa (`0`).

---

## Tanda tangan yang dideteksi

Empat keluarga pola (`patterns`), setiap entri berupa string (peka huruf besar/kecil) atau tabel `{ pat = "motif", ci = true }` (tidak peka huruf besar/kecil):

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, injeksi perintah SQL melalui titik koma (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: tag `<script>`, `<iframe>`, atribut `on*=` (penangan peristiwa), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: perangkaian perintah shell (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, varian dengan `|`), substitusi perintah (`` `...` ``, `$(...)`), tag PHP (`<?php`) dan fungsi eksekusi (`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

Ditambah daftar khusus **User-Agent pemindai/bot** (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, crawler SEO (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`), dan pola generik (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

Dekode URL diterapkan secara rekursif (hingga 5 tingkat pada jalur, 2 tingkat pada query string) sebelum pemeriksaan, untuk menangkap payload yang dikodekan ganda.

---

## Jalur sensitif yang dipantau

Pola yang diperiksa pada jalur yang sudah didekodekan: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

Untuk setiap kecocokan, skrip mencoba menentukan apakah sumber daya fisik benar-benar ada di jalur tersebut (melalui `lighty.stat`, dengan cadangan `lighty.c.stat` demi kompatibilitas) dan mencantumkannya di halaman peringatan serta di log — berguna untuk membedakan pemindaian membabi buta dari jalur yang benar-benar ada di server.

---

## Rate limiting dan eskalasi

Dua pembatas laju independen, berbasis penghitung di memori yang disimpan persisten ke disk (jendela geser sederhana, bukan jendela geser yang presisi per detik):

- **Global**: kunci `g:<ip>`, `global_rate_max_req` permintaan per `global_rate_window` detik, untuk semua rute.
- **SearXNG**: kunci `sx:<ip>`, `searx_rate_max_req` permintaan per `searx_rate_window` detik, hanya pada rute pencarian yang dikenali.

Eskalasi bertahap (`record_suspicious_hit`) adalah mekanisme yang terpisah dari rate limiting: setiap deteksi non-kritis (jalur sensitif tanpa residivisme, User-Agent pemindai, hit kecil pada badan POST) menaikkan penghitung per IP (kunci `susp:<ip>`) dalam jendela `suspicious_window` detik. Begitu `suspicious_threshold` hit tercapai dalam jendela ini, IP ditambahkan ke daftar blokir permanen — menghindari blokir langsung akibat satu positif palsu, sambil tetap memblokir residivis.

**Jaringan yang tidak pernah diblokir**: alamat dalam `never_auto_block` (host lokal, Tailscale `100.64.0.0/10`, jaringan lokal IPv4/IPv6, termasuk bentuk IPv4-mapped IPv6) tidak pernah memicu eskalasi maupun penambahan ke daftar blokir; blokir yang dihindari dicatat dengan baris `BAN EVITE` di `blocklist_log` (tidak pernah `AJOUT BLOCKLIST`, yang dibaca oleh omega-jail). Perbandingan CIDR dalam Lua portabel, dari 5.1 hingga 5.5 dan LuaJIT.

**Mode log-only omega-serv**: saat WAF omega-serv dalam mode `log-only`, skrip tidak memasang blokir apa pun (tidak ada daftar blokir, tidak ada baris `AJOUT BLOCKLIST`, jadi tidak ada blokir jaringan oleh fail2ban); baris `BAN EVITE` mencatatnya. Jika daftar blokir tidak dapat ditulis, baris `ECHEC BLOCKLIST` menggantikan `AJOUT BLOCKLIST`.

---

## Persistensi

| Berkas | Isi | Format |
|---|---|---|
| `blocklist_file` | Satu IP yang diblokir per baris, didahului komentar bertanda waktu (`# date - raison`) | Teks biasa |
| `waf_rules_dir/whitelist_paths.txt` | Satu pola Lua (`string.match`) per baris, jalur yang selalu diizinkan bahkan untuk IP yang diblokir | Teks biasa |
| `rate_store_file` | Penghitung rate limiting dan residivisme (`{ count = N, first = timestamp }` per kunci) | Tabel Lua terserialisasi (`return { ... }`, dibaca ulang melalui `load()`) |
| `alerts_log` | Satu baris per deteksi/penolakan | Teks yang mudah dibaca, atau JSON jika `use_json_logs = true` |
| `blocklist_log` | Satu baris per penambahan ke daftar blokir | Teks yang mudah dibaca |

Berkas rate limiting tidak pernah ditulis ulang pada setiap permintaan (paling sering setiap 300 detik, saat pembersihan berkala) untuk menghindari biaya I/O yang berlebihan. Daftar blokir dan whitelist di-cache di memori dan dibaca ulang paling sering setiap `blocklist_refresh_interval`/`whitelist_refresh_interval` detik — penambahan yang dipicu oleh skrip itu sendiri langsung memperbarui cache, tanpa menunggu jeda ini.

---

## Kode respons HTTP yang dikirim

| Kode | Arti | Pemicu |
|---|---|---|
| `0` | Melanjutkan pemrosesan normal (konvensi `mod_magnet`) | Tidak ada deteksi |
| `403` | Akses ditolak, peringatan pertama | Jalur sensitif terdeteksi (kali pertama), User-Agent pemindai (kali pertama) |
| `413` | Entitas terlalu besar | Badan POST melebihi `body_max_inspect` |
| `414` | URI terlalu panjang | Query string SearXNG melebihi `searx_max_query_len` |
| `429` | Terlalu banyak permintaan | Rate limit global atau SearXNG terlampaui |
| `451` | Tidak tersedia karena alasan hukum (di sini digunakan sebagai kode blokir) | IP sudah diblokir, residivisme setelah eskalasi, SQLi/XSS/traversal terdeteksi di jalur atau query string, byte NUL, skor POST ≥ ambang |
| `500` | Kesalahan internal | Pengecualian Lua tak terduga (ditangkap oleh `pcall` global) |

---

## Kompatibilitas

Skrip ini berfungsi dalam **dua konteks**, tanpa modifikasi apa pun:

- **lighttpd native**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` adalah API native modul `mod_magnet`. Kelima jalur konfigurasi langsung menggunakan nilai bawaan yang ditulis di kode.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): jembatan Python-Lua (`lupa`) mereproduksi dengan setia kontrak `lighty.*` yang sama, dengan runtime Lua yang **persisten** antarpermintaan (cache berbasis waktu dan rate limiting karenanya bertahan dengan benar dari satu permintaan ke permintaan berikutnya, seperti di lighttpd). Tabel global `omega_magnet_paths` disisipkan sebelum eksekusi dari konfigurasi layar WAF Magnet, agar kelima jalur menunjuk ke lokasi yang dipilih di antarmuka, bukan ke nilai tetap.

Tidak ada perbedaan logika bisnis antara kedua konteks — hanya asal kelima jalur konfigurasi yang berubah.

### Yang TIDAK kompatibel apa adanya

Skrip ini sepenuhnya bergantung pada kontrak `lighty.*` (API native `mod_magnet`) — skrip ini **tidak** berjalan apa adanya di server lain, meskipun server tersebut menawarkan semacam skrip Lua:

- **Apache** (`mod_lua`): API berbeda, berbasis objek permintaan `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — tidak ada panggilan `lighty.*` yang akan berfungsi.
- **nginx** (`lua-nginx-module` / OpenResty): API berbeda (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), dengan model fase eksekusi (`access_by_lua`, `content_by_lua`...) yang berbeda dari pengembalian kode sederhana `mod_magnet`.
- **Traefik**: tidak ada skrip Lua native (ditulis dalam Go) — hanya plugin Go (Yaegi) yang dapat menjalankan logika khusus, tidak pernah Lua secara langsung.
- **Caddy**: tidak ada modul Lua resmi — memerlukan plugin pihak ketiga yang dikompilasi melalui `xcaddy`, tidak ada padanan yang tersedia secara native.

Membuat skrip ini kompatibel dengan salah satu server tersebut memerlukan penulisan jembatan kompatibilitas khusus yang menerjemahkan API native-nya ke `lighty.*` (persis seperti yang dilakukan `lua_waf_adapter.py` untuk omega-serv) — pekerjaan rekayasa tersendiri, bukan sekadar penyesuaian konfigurasi.

---

## Kustomisasi

- **Menyesuaikan untuk aplikasi selain SearXNG**: ubah `is_searxng_path()` (daftar jalur) dan, bila perlu, hapus seluruh bagian 3 dari pipeline (kebijakan khusus) jika aplikasi yang dilindungi tidak memerlukan penanganan pencarian yang dibedakan.
- **Menambahkan jalur yang selalu diizinkan**: edit `waf_rules_dir/whitelist_paths.txt` (satu pola Lua per baris), atau tambahkan pengecualian tetap di `is_whitelisted_path()`.
- **Menyesuaikan tingkat agresivitas**: `waf_score_threshold` (ambang penolakan POST), `suspicious_threshold`/`suspicious_window` (kecepatan eskalasi menuju blokir permanen), `global_rate_max_req`/`global_rate_window` (rate limit umum).
- **Mengalihkan log ke JSON** (penyerapan SIEM/log shipper): `use_json_logs = true`.
- **Menonaktifkan satu subsistem tanpa menyentuh yang lain**: `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block` saling independen.

---

## Batasan yang diketahui dan di luar cakupan

- **Tidak ada deteksi CMDi pada jalur URL**: pilihan yang disengaja untuk menghindari positif palsu pada jalur yang berisi `.php` (yang akan tertukar dengan `<?php` jika deteksi CMDi menyertakan tag PHP pada permukaan ini). Deteksi CMDi tetap aktif pada badan POST.
- **Tanda tangan berbasis pencocokan pola Lua, bukan mesin PCRE sungguhan**: varian serangan yang tidak tercakup oleh pola yang terdaftar tidak akan terdeteksi — ini bukan pengganti WAF berbasis aturan yang menyeluruh (seperti CRS ModSecurity).
- **Tidak ada parsing multipart/form-data**: payload berbahaya yang dibungkus dalam kolom formulir multipart dapat lolos dari pemeriksaan jika pola tidak cocok dengan aliran mentah yang belum didekodekan.
- **Tidak ada dekompresi gzip pada badan**: badan yang terkompresi tidak diperiksa dalam bentuk teks biasa.
- **Rate limiting perkiraan**: jendela geser yang disederhanakan (reset penuh saat jendela terlampaui), bukan algoritme jendela geser yang presisi (sliding log/token bucket).
- **Tidak ada antarmuka pemantauan**: semua pengamatan dilakukan dengan membaca langsung berkas log dan daftar blokir.

---

## Pemecahan masalah

- **Skrip tampaknya tidak pernah berjalan**: periksa bahwa `mod_magnet` benar-benar dimuat (lighttpd) atau bahwa extra `magnet` terpasang dan WAF Magnet diaktifkan (omega-serv); memuat ulang konfigurasi tidak pernah cukup untuk memuat skrip Lua yang diubah, mulai ulang penuh diperlukan dalam kedua kasus.
- **Penghitung rate limit tampak ter-reset pada setiap permintaan**: gejala runtime Lua yang tidak persisten antarpermintaan (skrip mulai dari `rate_store` kosong pada setiap eksekusi) — di omega-serv, periksa bahwa layar WAF Magnet memang menunjukkan runtime persisten yang aktif.
- **IP yang sah ikut diblokir**: periksa `blocklist_log` dan `alerts_log` untuk mengidentifikasi aturan yang terpicu, sesuaikan pola atau ambang yang terkait, lalu hapus baris IP tersebut dari `blocklist_file` (cache memori diperbarui pada pembacaan ulang berikutnya, paling lambat `blocklist_refresh_interval` detik setelah berkas diubah).
- **Pencarian SearXNG yang sah diblokir**: periksa `is_searxng_path()` (jalur tidak dikenali) sebelum menyesuaikan ambang rate limit khusus.
- **`pip install -e ".[magnet]"` gagal dengan `pip: command not found` atau `externally-managed-environment`**: perintah diketik di luar `.venv` omega-serv (beberapa distribusi, terutama Arch Linux, tidak menyediakan `pip` sistem yang dapat langsung digunakan). Gunakan `.venv/bin/python -m pip install -e ".[magnet]"` dari direktori omega-serv — berfungsi tanpa bergantung pada ada tidaknya `pip` di `PATH` sistem (lihat [Memasang extra `magnet` omega-serv](#memasang-extra-magnet-omega-serv)).

---

## Lisensi

Omega-Waf didistribusikan di bawah lisensi **MIT**. Lihat berkas [`LICENSE`](../../../LICENSE) untuk teks lengkapnya.

---

> **Omega-Waf — Satu skrip, satu garis pertahanan.**
>
> Firewall aplikasi mod_magnet dalam satu berkas Lua, tanpa dependensi, kompatibel dengan lighttpd dan omega-serv.
