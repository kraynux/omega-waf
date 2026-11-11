<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="512">
</div>

# 🔒 OMEGA-WAF

**Tường lửa ứng dụng mod_magnet (script Lua độc lập)**

> Được xây dựng bởi [kraynux](https://github.com/kraynux) cho [Omega-Server](https://kraynux.snake-mackarel.ts.net)   

Trang chính thức: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/vi/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#khả-năng-tương-thích)

🌐 Tài liệu có sẵn bằng **17 ngôn ngữ**.  
**Ngôn ngữ:**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](../pt-BR/README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](../it/README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](../de/README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](../nl/README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](../pl/README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](../id/README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](../ja/README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](../ko/README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](../hi/README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

**Omega-Waf** là một script Lua duy nhất (`omega-waf.lua`), không có phụ thuộc bên ngoài, được thiết kế để chạy ở mỗi yêu cầu HTTP bởi một máy chủ web tương thích `mod_magnet` (lighttpd một cách tự nhiên, hoặc omega-serv qua cầu tương thích `magnet` của nó). Script kiểm tra đường dẫn, truy vấn và, với POST, phần thân của yêu cầu để tìm các chữ ký tấn công đã biết (SQL injection, XSS, chèn lệnh, duyệt thư mục), áp dụng giới hạn tốc độ (rate limiting), quản lý một danh sách chặn lưu bền với cơ chế leo thang dần, và ghi log mọi quyết định.

Không cần cài đặt theo nghĩa thông thường: chỉ một tệp cần đặt vào, tham chiếu từ cấu hình máy chủ, và thế là xong.

## Mục lục

- [Giới thiệu](#giới-thiệu)
- [Cài đặt](#cài-đặt)
- [Cấu hình](#cấu-hình)
- [Quy trình ra quyết định](#quy-trình-ra-quyết-định)
- [Các chữ ký được phát hiện](#các-chữ-ký-được-phát-hiện)
- [Các đường dẫn nhạy cảm được giám sát](#các-đường-dẫn-nhạy-cảm-được-giám-sát)
- [Giới hạn tốc độ và leo thang](#giới-hạn-tốc-độ-và-leo-thang)
- [Lưu trữ bền vững](#lưu-trữ-bền-vững)
- [Các mã phản hồi HTTP được trả về](#các-mã-phản-hồi-http-được-trả-về)
- [Khả năng tương thích](#khả-năng-tương-thích)
- [Tùy biến](#tùy-biến)
- [Giới hạn đã biết và ngoài phạm vi](#giới-hạn-đã-biết-và-ngoài-phạm-vi)
- [Khắc phục sự cố](#khắc-phục-sự-cố)
- [Giấy phép](#giấy-phép)

---

## Giới thiệu

### Mục tiêu

- Lọc các yêu cầu độc hại càng gần máy chủ càng tốt, trước khi chúng đến được ứng dụng được phục vụ.
- Luôn là một tệp duy nhất, dễ đọc, không có phụ thuộc bên ngoài và không cần bước biên dịch.
- Không bao giờ làm sập máy chủ web: mọi lỗi nội bộ của script đều được bắt lại và ghi log thay vì lan truyền ra ngoài.
- Hạn chế chi phí CPU/bộ nhớ cho mỗi yêu cầu (bộ nhớ đệm theo thời gian, đọc phần thân theo từng khối, độ sâu giải mã có giới hạn).
- Cung cấp cơ chế leo thang dần (cảnh báo rồi mới cấm) thay vì cấm thẳng ngay ở sự cố nhỏ đầu tiên.

### Omega-Waf làm gì

- Chặn các địa chỉ IP đã bị cấm (danh sách chặn lưu bền trên đĩa), với ngoại lệ cho các đường dẫn được cho phép tường minh (CSS của trang lỗi, v.v.).
- Giám sát một tập hợp đường dẫn nhạy cảm (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store`, v.v.) và phản hồi bằng một trang cảnh báo HTML, hoặc bằng lệnh cấm trực tiếp khi tái phạm.
- Phát hiện các User-Agent được biết đến là công cụ quét hoặc bot không mong muốn (sqlmap, nmap, nikto, dirbuster, crawler chung chung...).
- Áp dụng giới hạn tốc độ toàn cục theo IP, trên mọi tuyến.
- Áp dụng một chính sách riêng, nới lỏng hơn, trên các tuyến của công cụ tìm kiếm SearXNG (tránh dương tính giả với các truy vấn tìm kiếm hợp lệ chứa từ khóa trông giống một cuộc tấn công).
- Kiểm tra đường dẫn và query-string (SQLi, XSS, duyệt thư mục, byte NUL).
- Kiểm tra phần thân của các yêu cầu POST (SQLi, XSS, chèn lệnh, duyệt thư mục) với hệ thống tính điểm cộng dồn.
- Quản lý leo thang dần: một hành vi đáng ngờ đơn lẻ kích hoạt cảnh báo hoặc một lần từ chối; tái phạm trong một khoảng thời gian kích hoạt lệnh cấm vĩnh viễn.
- Ghi log mọi quyết định (cảnh báo, thêm vào danh sách chặn) ở định dạng văn bản hoặc JSON.
- Hoạt động được cả với lighttpd gốc (đường dẫn viết cứng) lẫn khi được nạp vào omega-serv (đường dẫn nhận động từ cấu hình máy chủ).

### Những gì script không làm

- Nó không thay thế một WAF chuyên dụng cấp doanh nghiệp (ModSecurity, Coraza, một WAF đám mây): các chữ ký được cố ý giữ đơn giản (so khớp mẫu Lua, không phải một bộ máy quy tắc PCRE thực sự).
- Nó không phân tích từng tiêu đề HTTP một ngoài User-Agent.
- Nó không giải mã phần thân multipart/form-data hay phần thân nén (gzip) — việc kiểm tra áp dụng trên văn bản thô nhận được.
- Nó không bao giờ kiểm tra phần thân của một yêu cầu không phải POST.
- Nó không phát hiện chèn lệnh trên đường dẫn URL (lựa chọn có chủ ý, xem [Giới hạn đã biết](#giới-hạn-đã-biết-và-ngoài-phạm-vi)).
- Nó không cung cấp giao diện quản trị nào: cấu hình bằng cách sửa các biến ở đầu tệp, danh sách chặn và danh sách trắng qua các tệp văn bản đơn giản.
- Nó không tự quản lý việc xoay vòng log của mình.

### Cảnh báo khi sử dụng

Các chữ ký được cung cấp nhắm đến các họ tấn công phổ biến nhưng không tạo thành một lớp bảo vệ đầy đủ. Một ngưỡng điểm phát hiện quá thấp trên một trang có lưu lượng bất thường (tìm kiếm kỹ thuật, tài liệu chứa lệnh shell, v.v.) có thể gây dương tính giả — hãy điều chỉnh `waf_score_threshold` và danh sách trắng đường dẫn cho phù hợp. Luôn thử nghiệm mọi thay đổi chữ ký hoặc ngưỡng trong môi trường không quan trọng trước khi triển khai lên production.

---

## Cài đặt

### Yêu cầu tiên quyết

- Một máy chủ web cung cấp giao diện tương thích `mod_magnet`:
  - **lighttpd** với mô-đun `mod_magnet` được bật (`server.modules += ("mod_magnet")`), hoặc
  - **omega-serv** với phần bổ sung `magnet` đã được cài đặt (phụ thuộc `lupa` — xem bên dưới và hướng dẫn omega-serv, phần WAF Magnet).
- Một trình thông dịch Lua nhúng do máy chủ chủ cung cấp (lighttpd nhúng Lua riêng của nó; omega-serv nhúng `lupa`, trong cả hai trường hợp đều không cần trình thông dịch `lua` của hệ thống — `lupa` được phân phối dưới dạng gói wheel biên dịch sẵn cho mọi kiến trúc/phiên bản Python phổ biến, không cần biên dịch cục bộ hay tệp tiêu đề Lua của hệ thống).

#### Cài đặt phần bổ sung `magnet` của omega-serv

**Luôn thực hiện từ `.venv` đã được `omega-serv/install.sh` tạo sẵn, không bao giờ dùng `pip` của hệ thống** — trên một số bản phân phối (đặc biệt là Arch Linux), lệnh `pip` trần không tồn tại (chỉ gói `python-pip` cài nó, và không phải lúc nào cũng có sẵn mặc định) hoặc từ chối cài đặt bất cứ thứ gì bên ngoài môi trường ảo (`error: externally-managed-environment`, PEP 668):

```bash
cd /path/to/omega-serv

# Cách A - venv đã được kích hoạt trong shell hiện tại
source .venv/bin/activate
pip install -e ".[magnet]"

# Cách B - không kích hoạt venv (cách di động nhất: không bao giờ phụ thuộc
# vào việc có hay không 'pip' trên PATH hệ thống, vì python của
# venv luôn kèm theo pip riêng)
.venv/bin/python -m pip install -e ".[magnet]"
```

Nếu `install.sh` chưa được chạy (chưa có `.venv/`), hãy chạy nó trước — nó tạo môi trường ảo và cài sẵn chính omega-serv vào đó; khi đó chỉ còn phải thêm phần bổ sung `magnet` (tùy chọn) bằng một trong hai lệnh ở trên.

### Triển khai

1. Sao chép `omega-waf.lua` vào một vị trí mà máy chủ đọc được (ví dụ `secure/waf/scripts/omega-waf.lua` với cấu trúc thư mục kiểu omega-serv, hoặc bất kỳ đường dẫn nào với lighttpd gốc).
2. Tạo các thư mục/tệp cần thiết:
   - **omega-serv**: không cần làm gì, máy chủ tự tạo các thư mục khi khởi động (bao gồm `var/lib/waf/`, dùng chung với tài khoản của dịch vụ).
   - **lighttpd gốc**: trước tiên thay 5 đường dẫn bằng đường dẫn tuyệt đối (xem [Đường dẫn khi dùng lighttpd gốc](#đường-dẫn-khi-dùng-lighttpd-gốc)), sau đó tạo các thư mục và cấp quyền ghi cho tài khoản của lighttpd (`www-data` trên Debian/Ubuntu, `http` trên Arch, `lighttpd` trên Fedora/Alpine):
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. Tham chiếu script từ cấu hình máy chủ:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/path/to/omega-waf.lua" )
     ```
   - **omega-serv**: bật WAF Magnet từ màn hình TUI chuyên dụng (Bảo mật chủ động → WAF → Magnet), màn hình này trỏ tới script và tự động nạp bảng `omega_magnet_paths` (xem [Khả năng tương thích](#khả-năng-tương-thích)).
4. Khởi động lại (không phải nạp lại — nạp lại không bao giờ tải lại một script Lua đã nằm trong bộ nhớ lưu bền) máy chủ liên quan.

---

## Cấu hình

Mọi tùy chọn được thiết lập bằng cách sửa trực tiếp các biến cục bộ ở đầu tệp (phần `CONFIGURATION`, dòng ~24-70). Không có tệp cấu hình riêng.

| Biến | Vai trò | Giá trị mặc định |
|---|---|---|
| `blocklist_file` | Tệp văn bản chứa các IP bị cấm vĩnh viễn (mỗi dòng một IP, `#` cho chú thích) — được script **ghi** vào | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | Thư mục quy tắc bên ngoài (chứa `whitelist_paths.txt`) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | Log cảnh báo (phát hiện, từ chối, vượt ngưỡng) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | Log riêng cho các lần thêm vào danh sách chặn | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | Tệp lưu bền các bộ đếm (giới hạn tốc độ / tái phạm), định dạng bảng Lua | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | Bật kiểm tra phần thân của yêu cầu POST | `true` |
| `enable_rate_limit` | Bật giới hạn tốc độ (toàn cục + SearXNG) | `true` |
| `enable_auto_block` | Cho phép tự động thêm vào danh sách chặn | `true` |
| `never_auto_block` | Các mạng **không bao giờ** bị cấm tự động (yêu cầu vẫn bị từ chối và ghi log, chỉ lệnh cấm được bỏ qua); với omega-serv, danh sách do máy chủ nạp vào (`omega_never_auto_block`) luôn được ưu tiên | máy cục bộ, Tailscale, mạng cục bộ |
| `use_json_logs` | Chuyển log cảnh báo sang JSON (nếu không thì là văn bản dễ đọc) | `false` |
| `waf_score_threshold` | Điểm cộng dồn tối thiểu để một yêu cầu POST bị từ chối | `5` |
| `body_max_inspect` | Kích thước tối đa của phần thân được kiểm tra (byte) | `65536` (64 KB) |
| `chunk_size` | Kích thước các khối đọc phần thân | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | Hạn mức và khoảng thời gian (s) của giới hạn tốc độ toàn cục theo IP | `60` yêu cầu / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | Hạn mức và khoảng thời gian (s) của giới hạn tốc độ riêng cho các tuyến SearXNG | `40` yêu cầu / `60` s |
| `searx_max_query_len` | Kích thước tối đa được chấp nhận của query-string trên các tuyến SearXNG | `4096` ký tự |
| `suspicious_threshold` | Số lần vi phạm đáng ngờ trước khi bị cấm vĩnh viễn | `3` |
| `suspicious_window` | Cửa sổ trượt (s) dùng để đếm các lần vi phạm đáng ngờ | `300` (5 phút) |
| `blocklist_refresh_interval` | Khoảng thời gian (s) giữa các lần đọc lại tệp danh sách chặn | `5` |
| `whitelist_refresh_interval` | Khoảng thời gian (s) giữa các lần đọc lại danh sách trắng đường dẫn | `15` |

Với omega-serv, 5 đường dẫn đầu tiên (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) được tự động thay bằng các đường dẫn cấu hình trong màn hình WAF Magnet (bảng toàn cục `omega_magnet_paths` được nạp vào trước khi thực thi) — các giá trị viết cứng ở trên khi đó chỉ dùng làm phương án dự phòng nếu bảng này không có.

> ℹ️ **Thay đổi ngày 2026-10-18**: `blocklist_file` chuyển từ `secure/waf/magnet-blocklist.txt` sang `var/lib/waf/magnet-blocklist.txt`. Script **ghi** vào tệp này mỗi lần cấm, và dịch vụ systemd của omega-serv chỉ có thể ghi bên dưới `var/`: với vị trí cũ, không lệnh cấm tự động nào từng được ghi lại, mà không có bất kỳ thông báo nào. omega-serv tiếp nhận tệp cũ một lần duy nhất khi khởi động (tệp cũ vẫn được giữ nguyên làm bản sao lưu); một đường dẫn tùy chỉnh trong màn hình WAF Magnet không bao giờ bị thay đổi.

### Đường dẫn khi dùng lighttpd gốc

> ⚠️ **5 giá trị mặc định này được mô phỏng theo quy ước thư mục của omega-serv** (`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, các đường dẫn tương đối được phân giải theo một thư mục làm việc kiểu dự án) — chúng không trung lập. Với **lighttpd gốc**, nơi không có quy ước thư mục tương đương nào, các đường dẫn tương đối này được phân giải theo thư mục làm việc của tiến trình lighttpd lúc chạy (thường là `/`), hiếm khi đoán trước được. **Hãy thay 5 dòng này bằng các đường dẫn tuyệt đối** phù hợp với bản cài đặt của bạn, ví dụ:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> Trước khi khởi động lại, hãy tạo các thư mục này (các lệnh ở bước 2 của phần [Triển khai](#triển-khai)) với các quyền sau:
>
> | Tệp | Quyền truy cập cần cho tài khoản của lighttpd |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **ghi** (tệp và thư mục chứa nó) |
> | `waf_rules_dir` (và `whitelist_paths.txt`) | chỉ đọc |
>
> Một lần ghi thất bại là **im lặng** — script không bao giờ làm hỏng yêu cầu —: một lệnh cấm không xuất hiện trong `blocklist_file` là triệu chứng duy nhất. Kiểm tra bằng `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` (điều chỉnh tài khoản cho phù hợp). Nếu unit systemd của lighttpd giới hạn việc ghi (`ProtectSystem=strict`, `ReadWritePaths=`), hãy thêm các thư mục này vào đó.

---

## Quy trình ra quyết định

Script chạy toàn bộ bên trong một `pcall` toàn cục: mọi lỗi Lua ngoài dự kiến đều được ghi log rồi phản hồi `500`, không bao giờ làm sập tiến trình máy chủ. Các bước chạy theo thứ tự sau, và mỗi bước đều có thể dừng việc xử lý bằng một phản hồi cuối cùng:

1. **Danh sách chặn** — nếu IP đã bị cấm, yêu cầu bị từ chối (`451`), trừ khi đường dẫn được yêu cầu nằm tường minh trong danh sách trắng (ví dụ để phục vụ chính trang lỗi).
2. **Đường dẫn nhạy cảm** — nếu đường dẫn khớp với một mẫu được giám sát (`.git`, `.env`, `wp-admin`...), lần đầu sẽ trả về một trang cảnh báo HTML (`403`); tái phạm trong cửa sổ nghi ngờ sẽ kích hoạt lệnh cấm (`451`).
3. **User-Agent của trình quét** — nếu User-Agent khớp với một công cụ quét/bot đã biết, `403` lần đầu, `451` khi tái phạm.
4. **Giới hạn tốc độ toàn cục** — nếu vượt hạn mức toàn cục theo IP, `429`.
5. **Chính sách SearXNG** — với các tuyến tìm kiếm được nhận diện: giới hạn tốc độ riêng (`429`), chốt chặn kích thước query-string (`414`), từ chối byte NUL (`451`), rồi thoát ngay (`0`, tiếp tục) để một tìm kiếm hợp lệ không bao giờ phải qua các bước kiểm tra gắt gao tiếp theo.
6. **Bảo vệ đường dẫn** — tìm SQLi và XSS trong đường dẫn đã giải mã (`451` nếu phát hiện); tìm duyệt thư mục (`451`). Cố ý **không** phát hiện chèn lệnh ở bước này (xem [Giới hạn đã biết](#giới-hạn-đã-biết-và-ngoài-phạm-vi)).
7. **Bảo vệ nhẹ cho query-string** (ngoài SearXNG) — duyệt thư mục và byte NUL (`451`).
8. **Kiểm tra phần thân POST** — chỉ khi `enable_body_inspection` được bật và phương thức là `POST`: phần thân quá lớn bị từ chối ngay (`413`); nếu không thì tính điểm cộng dồn cho SQLi/XSS/CMDi/duyệt thư mục, từ chối (`451`) nếu điểm đạt `waf_score_threshold`, cấm ngay lập tức nếu điểm đạt 8, nếu không thì chỉ tính là một lần vi phạm đáng ngờ.
9. **Không chặn** — yêu cầu tiếp tục bình thường (`0`).

---

## Các chữ ký được phát hiện

Bốn họ mẫu (`patterns`), mỗi mục là một chuỗi (phân biệt chữ hoa/thường) hoặc một bảng `{ pat = "mẫu", ci = true }` (không phân biệt chữ hoa/thường):

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, chèn lệnh SQL qua dấu chấm phẩy (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`).
- **`xss_body`**: thẻ `<script>`, `<iframe>`, thuộc tính `on*=` (trình xử lý sự kiện), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`.
- **`cmdi_body`**: nối chuỗi lệnh shell (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, các biến thể với `|`), thay thế lệnh (`` `...` ``, `$(...)`), thẻ PHP (`<?php`) và các hàm thực thi (`system(`, `exec(`, `passthru(`).
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`.

Và một danh sách riêng các **User-Agent của trình quét/bot** (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, các crawler SEO (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`), và các mẫu chung (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`).

Giải mã URL được áp dụng đệ quy (tối đa 5 cấp trên đường dẫn, 2 cấp trên query-string) trước khi kiểm tra, để bắt được các payload bị mã hóa hai lần.

---

## Các đường dẫn nhạy cảm được giám sát

Các mẫu được kiểm tra trên đường dẫn đã giải mã: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`.

Với mỗi lần khớp, script cố xác định xem có một tài nguyên vật lý thực sự tồn tại tại đường dẫn đó hay không (qua `lighty.stat`, dự phòng bằng `lighty.c.stat` để tương thích) và ghi rõ điều đó trong trang cảnh báo và trong log — hữu ích để phân biệt một lần quét mù với một đường dẫn thực sự tồn tại trên máy chủ.

---

## Giới hạn tốc độ và leo thang

Hai bộ giới hạn tốc độ độc lập, dựa trên các bộ đếm trong bộ nhớ được lưu bền xuống đĩa (cửa sổ trượt đơn giản, không phải cửa sổ trượt chính xác đến từng giây):

- **Toàn cục**: khóa `g:<ip>`, `global_rate_max_req` yêu cầu mỗi `global_rate_window` giây, trên mọi tuyến.
- **SearXNG**: khóa `sx:<ip>`, `searx_rate_max_req` yêu cầu mỗi `searx_rate_window` giây, chỉ trên các tuyến tìm kiếm được nhận diện.

Leo thang dần (`record_suspicious_hit`) là một cơ chế tách biệt với giới hạn tốc độ: mỗi phát hiện không nghiêm trọng (đường dẫn nhạy cảm chưa tái phạm, User-Agent của trình quét, vi phạm nhỏ trong phần thân POST) làm tăng một bộ đếm theo IP (khóa `susp:<ip>`) trong một cửa sổ `suspicious_window` giây. Khi đạt `suspicious_threshold` lần vi phạm trong cửa sổ này, IP được thêm vào danh sách chặn vĩnh viễn — tránh cấm thẳng chỉ vì một lần dương tính giả duy nhất, trong khi vẫn cấm được kẻ tái phạm.

**Mạng không bao giờ bị cấm**: một địa chỉ thuộc `never_auto_block` (máy cục bộ, Tailscale `100.64.0.0/10`, mạng cục bộ IPv4/IPv6, kể cả dạng IPv4 ánh xạ sang IPv6) không bao giờ kích hoạt leo thang hay bị thêm vào danh sách chặn; một lệnh cấm được bỏ qua sẽ được ghi lại bằng một dòng `BAN EVITE` trong `blocklist_log` (không bao giờ là `AJOUT BLOCKLIST`, dòng mà omega-jail đọc). So sánh CIDR bằng Lua di động, từ 5.1 đến 5.5 và LuaJIT.

**Chế độ log-only của omega-serv**: khi WAF của omega-serv ở chế độ `log-only`, script không đặt lệnh cấm nào (không ghi danh sách chặn, không có dòng `AJOUT BLOCKLIST`, nên fail2ban không cấm ở mức mạng); một dòng `BAN EVITE` ghi lại việc này. Nếu không ghi được danh sách chặn, dòng `ECHEC BLOCKLIST` thay cho `AJOUT BLOCKLIST`.

---

## Lưu trữ bền vững

| Tệp | Nội dung | Định dạng |
|---|---|---|
| `blocklist_file` | Mỗi dòng một IP bị cấm, đứng trước là một chú thích có dấu thời gian (`# ngày - lý do`) | Văn bản thô |
| `waf_rules_dir/whitelist_paths.txt` | Mỗi dòng một mẫu Lua (`string.match`), các đường dẫn luôn được cho phép kể cả với IP bị cấm | Văn bản thô |
| `rate_store_file` | Bộ đếm giới hạn tốc độ và tái phạm (`{ count = N, first = timestamp }` cho mỗi khóa) | Bảng Lua tuần tự hóa (`return { ... }`, đọc lại qua `load()`) |
| `alerts_log` | Mỗi dòng một lần phát hiện/từ chối | Văn bản dễ đọc, hoặc JSON nếu `use_json_logs = true` |
| `blocklist_log` | Mỗi dòng một lần thêm vào danh sách chặn | Văn bản dễ đọc |

Tệp giới hạn tốc độ không bao giờ được ghi lại ở mỗi yêu cầu (tối đa mỗi 300 giây một lần, trong lần dọn dẹp định kỳ) để tránh chi phí I/O quá lớn. Danh sách chặn và danh sách trắng được lưu đệm trong bộ nhớ và đọc lại tối đa mỗi `blocklist_refresh_interval`/`whitelist_refresh_interval` giây — một lần thêm do chính script kích hoạt sẽ cập nhật bộ nhớ đệm ngay lập tức, không phải chờ hết khoảng thời gian này.

---

## Các mã phản hồi HTTP được trả về

| Mã | Ý nghĩa | Tác nhân kích hoạt |
|---|---|---|
| `0` | Tiếp tục xử lý bình thường (quy ước của `mod_magnet`) | Không phát hiện gì |
| `403` | Truy cập bị từ chối, cảnh báo lần đầu | Phát hiện đường dẫn nhạy cảm (lần đầu), User-Agent của trình quét (lần đầu) |
| `413` | Thực thể quá lớn | Phần thân POST vượt quá `body_max_inspect` |
| `414` | URI quá dài | Query-string SearXNG vượt quá `searx_max_query_len` |
| `429` | Quá nhiều yêu cầu | Vượt giới hạn tốc độ toàn cục hoặc SearXNG |
| `451` | Không khả dụng vì lý do pháp lý (dùng ở đây làm mã cấm) | IP đã bị cấm, tái phạm sau leo thang, phát hiện SQLi/XSS/duyệt thư mục trong đường dẫn hoặc query-string, byte NUL, điểm POST ≥ ngưỡng |
| `500` | Lỗi nội bộ | Ngoại lệ Lua ngoài dự kiến (được `pcall` toàn cục bắt lại) |

---

## Khả năng tương thích

Script hoạt động trong **hai bối cảnh**, không cần sửa đổi gì:

- **lighttpd gốc**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` là các API gốc của mô-đun `mod_magnet`. 5 đường dẫn cấu hình dùng trực tiếp các giá trị mặc định viết cứng của chúng.
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): một cầu nối Python-Lua (`lupa`) tái tạo trung thực cùng hợp đồng `lighty.*`, với một runtime Lua **lưu bền** giữa các yêu cầu (vì vậy bộ nhớ đệm theo thời gian và giới hạn tốc độ được duy trì đúng từ yêu cầu này sang yêu cầu khác, như với lighttpd). Bảng toàn cục `omega_magnet_paths` được nạp vào trước khi thực thi từ cấu hình của màn hình WAF Magnet, để 5 đường dẫn trỏ tới các vị trí được chọn trong giao diện thay vì các giá trị viết cứng.

Không có khác biệt nào về logic nghiệp vụ giữa hai bối cảnh — chỉ nguồn gốc của 5 đường dẫn cấu hình là thay đổi.

### Những gì KHÔNG tương thích nguyên trạng

Script phụ thuộc hoàn toàn vào hợp đồng `lighty.*` (API gốc của `mod_magnet`) — nó **không** chạy nguyên trạng trên một máy chủ khác, kể cả khi máy chủ đó có một dạng script Lua nào đó:

- **Apache** (`mod_lua`): API khác, dựa trên một đối tượng yêu cầu `r` (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — không lệnh gọi `lighty.*` nào hoạt động được.
- **nginx** (`lua-nginx-module` / OpenResty): API khác (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), với mô hình các pha thực thi (`access_by_lua`, `content_by_lua`...) khác với việc chỉ trả về một mã như `mod_magnet`.
- **Traefik**: không có script Lua gốc (viết bằng Go) — chỉ một plugin Go (Yaegi) mới cho phép chạy logic tùy chỉnh, không bao giờ chạy Lua trực tiếp.
- **Caddy**: không có mô-đun Lua chính thức — sẽ cần một plugin bên thứ ba được biên dịch qua `xcaddy`, không có gì tương đương sẵn có.

Muốn script này tương thích với một trong các máy chủ trên sẽ cần viết một cầu nối tương thích riêng, chuyển đổi API gốc của máy chủ đó sang `lighty.*` (đúng như những gì `lua_waf_adapter.py` làm cho omega-serv) — một công việc kỹ thuật thực thụ, không phải một điều chỉnh cấu hình đơn giản.

---

## Tùy biến

- **Điều chỉnh cho một ứng dụng khác ngoài SearXNG**: sửa `is_searxng_path()` (danh sách đường dẫn) và, nếu cần, gỡ bỏ hoàn toàn mục 3 của quy trình (chính sách riêng) nếu ứng dụng được bảo vệ không cần xử lý tìm kiếm theo cách riêng.
- **Thêm các đường dẫn luôn được cho phép**: sửa `waf_rules_dir/whitelist_paths.txt` (mỗi dòng một mẫu Lua), hoặc thêm một ngoại lệ viết cứng trong `is_whitelisted_path()`.
- **Điều chỉnh mức độ gắt gao**: `waf_score_threshold` (ngưỡng từ chối POST), `suspicious_threshold`/`suspicious_window` (tốc độ leo thang đến lệnh cấm vĩnh viễn), `global_rate_max_req`/`global_rate_window` (giới hạn tốc độ chung).
- **Chuyển log sang JSON** (để SIEM/log shipper thu thập): `use_json_logs = true`.
- **Tắt một hệ thống con mà không ảnh hưởng phần còn lại**: `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block` là độc lập với nhau.

---

## Giới hạn đã biết và ngoài phạm vi

- **Không phát hiện CMDi trên đường dẫn URL**: lựa chọn có chủ ý để tránh dương tính giả với các đường dẫn chứa `.php` (bị nhầm với `<?php` nếu việc phát hiện CMDi bao gồm thẻ PHP trên bề mặt này). Việc phát hiện CMDi vẫn hoạt động trên phần thân POST.
- **Chữ ký dựa trên so khớp mẫu Lua, không phải một bộ máy PCRE thực sự**: các biến thể tấn công không nằm trong các mẫu đã liệt kê sẽ không bị phát hiện — đây không phải là sự thay thế cho một WAF dựa trên bộ quy tắc đầy đủ (kiểu ModSecurity CRS).
- **Không phân tích multipart/form-data**: một payload độc hại được bọc trong một trường biểu mẫu multipart có thể lọt qua việc kiểm tra nếu các mẫu không khớp với luồng thô chưa giải mã.
- **Không giải nén gzip phần thân**: một phần thân bị nén không được kiểm tra ở dạng văn bản rõ.
- **Giới hạn tốc độ gần đúng**: cửa sổ trượt giản lược (đặt lại hoàn toàn khi vượt quá cửa sổ), không phải một thuật toán cửa sổ trượt chính xác (sliding log/token bucket).
- **Không có giao diện giám sát**: mọi việc quan sát đều thông qua đọc trực tiếp các tệp log và danh sách chặn.

---

## Khắc phục sự cố

- **Script dường như không bao giờ chạy**: kiểm tra xem `mod_magnet` đã được nạp đúng chưa (lighttpd) hoặc phần bổ sung `magnet` đã được cài và WAF Magnet đã được bật chưa (omega-serv); nạp lại cấu hình không bao giờ đủ để tải một script Lua đã sửa, cần khởi động lại hoàn toàn trong cả hai trường hợp.
- **Các bộ đếm giới hạn tốc độ dường như bị đặt lại ở mỗi yêu cầu**: triệu chứng của một runtime Lua không lưu bền giữa các yêu cầu (script bắt đầu lại từ một `rate_store` trống ở mỗi lần thực thi) — với omega-serv, hãy kiểm tra xem màn hình WAF Magnet có báo một runtime lưu bền đang hoạt động hay không.
- **Một IP hợp lệ bị cấm**: xem `blocklist_log` và `alerts_log` để xác định quy tắc đã kích hoạt, điều chỉnh mẫu hoặc ngưỡng tương ứng, rồi xóa dòng của IP đó trong `blocklist_file` (bộ nhớ đệm được cập nhật ở lần đọc lại tiếp theo, tối đa `blocklist_refresh_interval` giây sau khi tệp được sửa).
- **Các tìm kiếm SearXNG hợp lệ bị chặn**: kiểm tra `is_searxng_path()` (đường dẫn không được nhận diện) trước khi điều chỉnh các ngưỡng giới hạn tốc độ riêng.
- **`pip install -e ".[magnet]"` thất bại với `pip: command not found` hoặc `externally-managed-environment`**: lệnh đã được gõ bên ngoài `.venv` của omega-serv (một số bản phân phối, đặc biệt là Arch Linux, không cung cấp `pip` hệ thống nào dùng trực tiếp được). Hãy dùng `.venv/bin/python -m pip install -e ".[magnet]"` từ thư mục của omega-serv — cách này hoạt động mà không phụ thuộc vào việc có `pip` trên `PATH` hệ thống hay không (xem [Cài đặt phần bổ sung `magnet` của omega-serv](#cài-đặt-phần-bổ-sung-magnet-của-omega-serv)).

---

## Giấy phép

Omega-Waf được phân phối theo giấy phép **MIT**. Xem tệp [`LICENSE`](../../../LICENSE) để biết toàn văn.

---

> **Omega-Waf — Một script, một tuyến phòng thủ.**
>
> Một tường lửa ứng dụng mod_magnet trong một tệp Lua duy nhất, không phụ thuộc, tương thích với lighttpd và omega-serv.
