<!-- Copyright (c) 2026 kraynux - kraynux@proton.me - Licence MIT (voir fichier LICENSE) -->

<div align="center">
  <img src="https://raw.githubusercontent.com/kraynux/kraynux/refs/heads/main/docs/assets/omega-waf.png" alt="Omega-Waf" width="384">
</div>

# 🔒 OMEGA-WAF

**mod_magnet एप्लिकेशन फ़ायरवॉल (स्वतंत्र Lua स्क्रिप्ट)**

> [kraynux](https://github.com/kraynux) द्वारा [Omega-Server](https://kraynux.snake-mackarel.ts.net) के लिए विकसित   

आधिकारिक पेज: [OMEGA-WAF](https://kraynux.snake-mackarel.ts.net/omega-waf/translation/hi/)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../../LICENSE)
[![Platform](https://img.shields.io/badge/Platform-Linux-informational.svg)](https://www.linux.org/)
[![Lua](https://img.shields.io/badge/Lua-5.1%20%2F%20LuaJIT-blue.svg)](https://www.lua.org/)
[![Compatible](https://img.shields.io/badge/Compatible-lighttpd%20mod__magnet%20%7C%20omega--serv-cyan.svg)](#संगतता)

🌐 दस्तावेज़ **17 भाषाओं** में उपलब्ध।  
**भाषाएँ:**  
<img src="https://flagcdn.com/w20/fr.png" alt="Français" style="height:14px;"> [Français](../../../README.md) · <img src="https://flagcdn.com/w20/gb.png" alt="English" style="height:14px;"> [English](../en/README.md) · <img src="https://flagcdn.com/w20/es.png" alt="Español" style="height:14px;"> [Español](../es/README.md) · <img src="https://flagcdn.com/w20/br.png" alt="Português (Brasil)" style="height:14px;"> [Português (Brasil)](../pt-BR/README.md) · <img src="https://flagcdn.com/w20/it.png" alt="Italiano" style="height:14px;"> [Italiano](../it/README.md) · <img src="https://flagcdn.com/w20/de.png" alt="Deutsch" style="height:14px;"> [Deutsch](../de/README.md) · <img src="https://flagcdn.com/w20/nl.png" alt="Nederlands" style="height:14px;"> [Nederlands](../nl/README.md) · <img src="https://flagcdn.com/w20/pl.png" alt="Polski" style="height:14px;"> [Polski](../pl/README.md) · <img src="https://flagcdn.com/w20/ru.png" alt="Русский" style="height:14px;"> [Русский](../ru/README.md) · <img src="https://flagcdn.com/w20/cn.png" alt="中文" style="height:14px;"> [中文](../zh-CN/README.md) · <img src="https://flagcdn.com/w20/vn.png" alt="Tiếng Việt" style="height:14px;"> [Tiếng Việt](../vi/README.md) · <img src="https://flagcdn.com/w20/th.png" alt="ไทย" style="height:14px;"> [ไทย](../th/README.md) · <img src="https://flagcdn.com/w20/id.png" alt="Bahasa Indonesia" style="height:14px;"> [Bahasa Indonesia](../id/README.md) · <img src="https://flagcdn.com/w20/jp.png" alt="日本語" style="height:14px;"> [日本語](../ja/README.md) · <img src="https://flagcdn.com/w20/kr.png" alt="한국어" style="height:14px;"> [한국어](../ko/README.md) · <img src="https://flagcdn.com/w20/in.png" alt="हिन्दी" style="height:14px;"> [हिन्दी](README.md) · <img src="https://flagcdn.com/w20/ae.png" alt="العربية" style="height:14px;"> [العربية](../ar/README.md)

---

**Omega-Waf** एक अकेली Lua स्क्रिप्ट (`omega-waf.lua`) है, बिना किसी बाहरी निर्भरता के, जिसे `mod_magnet`-संगत वेब सर्वर (मूल रूप से lighttpd, या omega-serv अपने `magnet` संगतता ब्रिज के ज़रिए) द्वारा हर HTTP अनुरोध पर चलाने के लिए बनाया गया है। यह पाथ, अनुरोध और, POST पर, अनुरोध के body में ज्ञात हमला सिग्नेचर (SQL इंजेक्शन, XSS, कमांड इंजेक्शन, डायरेक्टरी ट्रैवर्सल) खोजती है, दर सीमा लागू करती है, क्रमिक escalation के साथ एक स्थायी ब्लॉक सूची प्रबंधित करती है, और हर निर्णय को लॉग करती है।

पारंपरिक अर्थ में कोई इंस्टॉलेशन नहीं: बस एक फ़ाइल रखनी है, उसे सर्वर कॉन्फ़िगरेशन से संदर्भित करना है, और बस इतना ही।

## विषय-सूची

- [अवलोकन](#अवलोकन)
- [इंस्टॉलेशन](#इंस्टॉलेशन)
- [कॉन्फ़िगरेशन](#कॉन्फ़िगरेशन)
- [निर्णय पाइपलाइन](#निर्णय-पाइपलाइन)
- [पहचाने जाने वाले सिग्नेचर](#पहचाने-जाने-वाले-सिग्नेचर)
- [निगरानी में संवेदनशील पाथ](#निगरानी-में-संवेदनशील-पाथ)
- [दर सीमा और escalation](#दर-सीमा-और-escalation)
- [स्थायी भंडारण](#स्थायी-भंडारण)
- [लौटाए जाने वाले HTTP प्रतिक्रिया कोड](#लौटाए-जाने-वाले-http-प्रतिक्रिया-कोड)
- [संगतता](#संगतता)
- [अनुकूलन](#अनुकूलन)
- [ज्ञात सीमाएँ और दायरे से बाहर](#ज्ञात-सीमाएँ-और-दायरे-से-बाहर)
- [समस्या निवारण](#समस्या-निवारण)
- [लाइसेंस](#लाइसेंस)

---

## अवलोकन

### उद्देश्य

- दुर्भावनापूर्ण अनुरोधों को सर्वर के जितना हो सके उतने पास फ़िल्टर करना, इससे पहले कि वे परोसे जा रहे एप्लिकेशन तक पहुँचें।
- एक अकेली, पठनीय फ़ाइल बने रहना, बिना बाहरी निर्भरता और बिना कंपाइलेशन चरण के।
- वेब सर्वर को कभी न गिराना: स्क्रिप्ट की कोई भी आंतरिक त्रुटि आगे फैलने के बजाय पकड़ी और लॉग की जाती है।
- प्रति अनुरोध CPU/मेमोरी लागत सीमित रखना (समय-आधारित कैश, body को टुकड़ों में पढ़ना, सीमित डिकोडिंग गहराई)।
- पहली छोटी घटना पर सीधे प्रतिबंध के बजाय क्रमिक escalation (चेतावनी, फिर प्रतिबंध) देना।

### Omega-Waf क्या करता है

- पहले से प्रतिबंधित IP पतों को ब्लॉक करता है (डिस्क पर स्थायी ब्लॉक सूची), स्पष्ट रूप से अनुमत पाथ के लिए अपवाद के साथ (त्रुटि पेज की CSS, आदि)।
- संवेदनशील पाथ के एक समूह (`.git`, `.env`, `wp-admin`, `phpmyadmin`, `.DS_Store`, आदि) की निगरानी करता है और या तो HTML चेतावनी पेज से, या पुनरावृत्ति होने पर सीधे प्रतिबंध से जवाब देता है।
- स्कैनिंग टूल या अवांछित बॉट के रूप में ज्ञात User-Agent पहचानता है (sqlmap, nmap, nikto, dirbuster, सामान्य crawler...)।
- सभी रूट पर, प्रति IP वैश्विक दर सीमा लागू करता है।
- SearXNG सर्च इंजन के रूट पर एक अलग, अधिक उदार नीति लागू करता है (ऐसे कीवर्ड वाले वैध खोज अनुरोधों पर false positive से बचाता है जो अन्यथा हमले जैसे दिखते)।
- पाथ और query-string का निरीक्षण करता है (SQLi, XSS, डायरेक्टरी ट्रैवर्सल, NUL बाइट)।
- POST अनुरोधों के body का निरीक्षण करता है (SQLi, XSS, कमांड इंजेक्शन, डायरेक्टरी ट्रैवर्सल), संचयी स्कोर प्रणाली के साथ।
- क्रमिक escalation संभालता है: एक अकेला संदिग्ध व्यवहार एक अलर्ट या एक बार का इनकार ट्रिगर करता है; किसी समय-विंडो के भीतर पुनरावृत्ति स्थायी प्रतिबंध ट्रिगर करती है।
- हर निर्णय (अलर्ट, ब्लॉक सूची में जोड़) को टेक्स्ट या JSON फ़ॉर्मेट में लॉग करता है।
- नेटिव lighttpd (हार्डकोड किए गए पाथ) और omega-serv में इंजेक्ट होकर (सर्वर कॉन्फ़िगरेशन से गतिशील रूप से मिले पाथ), दोनों में समान रूप से काम करता है।

### स्क्रिप्ट क्या नहीं करती

- यह किसी एंटरप्राइज़-स्तर के समर्पित WAF (ModSecurity, Coraza, क्लाउड WAF) का विकल्प नहीं है: सिग्नेचर जानबूझकर सरल हैं (Lua पैटर्न मिलान, कोई असली PCRE नियम इंजन नहीं)।
- यह User-Agent के अलावा HTTP हेडर का एक-एक करके विश्लेषण नहीं करती।
- यह न तो multipart/form-data body को डिकोड करती है और न ही संपीड़ित (gzip) body को — निरीक्षण प्राप्त कच्चे टेक्स्ट पर होता है।
- यह POST के अलावा किसी अन्य अनुरोध का body कभी निरीक्षित नहीं करती।
- यह URL पाथ पर कमांड इंजेक्शन की कोई पहचान नहीं करती (जानबूझकर किया गया चुनाव, देखें [ज्ञात सीमाएँ](#ज्ञात-सीमाएँ-और-दायरे-से-बाहर))।
- यह कोई प्रशासन इंटरफ़ेस नहीं देती: कॉन्फ़िगरेशन फ़ाइल की शुरुआत के वेरिएबल संपादित करके होता है, ब्लॉक सूची और व्हाइटलिस्ट साधारण टेक्स्ट फ़ाइलों से।
- यह अपने लॉग का रोटेशन स्वयं नहीं संभालती।

### उपयोग संबंधी चेतावनी

दिए गए सिग्नेचर आम हमला-परिवारों को लक्षित करते हैं, लेकिन संपूर्ण सुरक्षा नहीं देते। असामान्य ट्रैफ़िक वाली साइट (तकनीकी खोज, shell कमांड वाले दस्तावेज़, आदि) पर बहुत कम पहचान स्कोर false positive पैदा कर सकता है — उसी के अनुसार `waf_score_threshold` और पाथ की व्हाइटलिस्ट समायोजित करें। सिग्नेचर या सीमा-मान में कोई भी बदलाव प्रोडक्शन में लगाने से पहले हमेशा किसी गैर-महत्वपूर्ण वातावरण में परखें।

---

## इंस्टॉलेशन

### पूर्वापेक्षाएँ

- `mod_magnet`-संगत इंटरफ़ेस देने वाला एक वेब सर्वर:
  - **lighttpd**, `mod_magnet` मॉड्यूल सक्रिय के साथ (`server.modules += ("mod_magnet")`), या
  - **omega-serv**, `magnet` extra इंस्टॉल के साथ (`lupa` निर्भरता — नीचे और omega-serv गाइड का WAF Magnet खंड देखें)।
- होस्ट द्वारा दिया गया एक एम्बेडेड Lua इंटरप्रेटर (lighttpd अपना Lua एम्बेड करता है; omega-serv `lupa` एम्बेड करता है, दोनों में से किसी भी स्थिति में सिस्टम का `lua` इंटरप्रेटर आवश्यक नहीं — `lupa` सभी आम आर्किटेक्चर/Python संस्करणों के लिए precompiled wheel के रूप में वितरित होता है, न कोई स्थानीय कंपाइलेशन, न सिस्टम के Lua हेडर आवश्यक)।

#### omega-serv का `magnet` extra इंस्टॉल करना

**हमेशा `omega-serv/install.sh` द्वारा पहले से बनाए गए `.venv` से, सिस्टम `pip` से कभी नहीं** — कुछ डिस्ट्रीब्यूशन पर (ख़ासकर Arch Linux), अकेली `pip` कमांड मौजूद ही नहीं होती (केवल `python-pip` पैकेज उसे इंस्टॉल करता है, और ज़रूरी नहीं कि डिफ़ॉल्ट रूप से) या वर्चुअल एनवायरनमेंट के बाहर कुछ भी इंस्टॉल करने से मना कर देती है (`error: externally-managed-environment`, PEP 668):

```bash
cd /path/to/omega-serv

# विकल्प A - venv वर्तमान shell में पहले से सक्रिय
source .venv/bin/activate
pip install -e ".[magnet]"

# विकल्प B - venv सक्रिय किए बिना (सबसे पोर्टेबल तरीका: सिस्टम PATH पर
# 'pip' के होने या न होने पर कभी निर्भर नहीं करता, क्योंकि venv का
# python हमेशा अपना pip साथ रखता है)
.venv/bin/python -m pip install -e ".[magnet]"
```

अगर `install.sh` अभी तक नहीं चलाया गया है (कोई `.venv/` मौजूद नहीं), तो पहले उसे चलाएँ — वह वर्चुअल एनवायरनमेंट बनाता है और उसमें omega-serv को पहले ही इंस्टॉल कर देता है; तब केवल (वैकल्पिक) `magnet` extra ऊपर की दो कमांड में से किसी एक से जोड़ना बाकी रहता है।

### सेटअप

1. `omega-waf.lua` को सर्वर द्वारा पढ़े जा सकने वाले स्थान पर कॉपी करें (उदा. omega-serv जैसी संरचना के लिए `secure/waf/scripts/omega-waf.lua`, या नेटिव lighttpd के लिए कोई भी पाथ)।
2. अपेक्षित डायरेक्टरी/फ़ाइलें बनाएँ:
   - **omega-serv**: कुछ करने की ज़रूरत नहीं, सर्वर शुरू होते समय फ़ोल्डर स्वयं बनाता है (जिनमें `var/lib/waf/` भी है, जो सेवा खाते के साथ साझा है)।
   - **नेटिव lighttpd**: पहले 5 पाथ को absolute पाथ से बदलें (देखें [नेटिव lighttpd में पाथ](#नेटिव-lighttpd-में-पाथ)), फिर फ़ोल्डर बनाएँ और उन्हें lighttpd खाते द्वारा लिखने योग्य बनाएँ (Debian/Ubuntu पर `www-data`, Arch पर `http`, Fedora/Alpine पर `lighttpd`):
     ```bash
     sudo install -d -o www-data -g www-data -m 750 /var/lib/lighttpd/waf /var/log/lighttpd /var/cache/lighttpd
     sudo install -d -m 755 /var/lib/lighttpd/waf/magnet-rules.d
     sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt
     ```
3. सर्वर कॉन्फ़िगरेशन से स्क्रिप्ट को संदर्भित करें:
   - **lighttpd**:
     ```
     magnet.attract-physical-path-to = ( "/path/to/omega-waf.lua" )
     ```
   - **omega-serv**: समर्पित TUI स्क्रीन (एक्टिव सुरक्षा → WAF → Magnet) से WAF Magnet सक्रिय करें, जो स्क्रिप्ट की ओर इशारा करती है और `omega_magnet_paths` तालिका अपने-आप इंजेक्ट करती है (देखें [संगतता](#संगतता))।
4. संबंधित सर्वर को रीस्टार्ट करें (रीलोड नहीं — रीलोड स्थायी मेमोरी में पहले से मौजूद Lua स्क्रिप्ट को कभी दोबारा लोड नहीं करता)।

---

## कॉन्फ़िगरेशन

सभी विकल्प फ़ाइल की शुरुआत में स्थानीय वेरिएबल को सीधे संपादित करके सेट किए जाते हैं (`CONFIGURATION` खंड, पंक्तियाँ ~24-70)। कोई अलग कॉन्फ़िगरेशन फ़ाइल नहीं।

| वेरिएबल | भूमिका | डिफ़ॉल्ट मान |
|---|---|---|
| `blocklist_file` | स्थायी रूप से प्रतिबंधित IP की टेक्स्ट फ़ाइल (हर पंक्ति में एक IP, टिप्पणी के लिए `#`) — स्क्रिप्ट द्वारा **लिखी जाती** है | `var/lib/waf/magnet-blocklist.txt` |
| `waf_rules_dir` | बाहरी नियमों का फ़ोल्डर (इसमें `whitelist_paths.txt` है) | `secure/waf/magnet-rules.d/` |
| `alerts_log` | अलर्ट लॉग (पहचान, इनकार, सीमा-उल्लंघन) | `var/log/waf-magnet-alerts.log` |
| `blocklist_log` | ब्लॉक सूची में जोड़ के लिए समर्पित लॉग | `var/log/waf-magnet-blocklist.log` |
| `rate_store_file` | काउंटर (दर सीमा / पुनरावृत्ति) की स्थायी भंडारण फ़ाइल, Lua तालिका फ़ॉर्मेट | `var/cache/waf-magnet-rate_store.lua` |
| `enable_body_inspection` | POST अनुरोधों के body का निरीक्षण सक्रिय करता है | `true` |
| `enable_rate_limit` | दर सीमा सक्रिय करता है (वैश्विक + SearXNG) | `true` |
| `enable_auto_block` | ब्लॉक सूची में स्वचालित जोड़ की अनुमति देता है | `true` |
| `never_auto_block` | नेटवर्क जो **कभी** स्वचालित रूप से प्रतिबंधित नहीं होते (अनुरोध फिर भी अस्वीकृत और लॉग होता है, केवल प्रतिबंध टाला जाता है); omega-serv में, सर्वर द्वारा इंजेक्ट की गई सूची (`omega_never_auto_block`) को हमेशा प्राथमिकता मिलती है | स्थानीय host, Tailscale, स्थानीय नेटवर्क |
| `use_json_logs` | अलर्ट लॉग को JSON में बदलता है (अन्यथा पठनीय टेक्स्ट) | `false` |
| `waf_score_threshold` | किसी POST अनुरोध को अस्वीकार करने के लिए न्यूनतम संचयी स्कोर | `5` |
| `body_max_inspect` | body का अधिकतम निरीक्षित आकार (बाइट) | `65536` (64 KB) |
| `chunk_size` | body पढ़ने के खंडों का आकार | `8192` (8 KB) |
| `global_rate_max_req` / `global_rate_window` | प्रति IP वैश्विक दर सीमा का कोटा और विंडो (s) | `60` अनुरोध / `60` s |
| `searx_rate_max_req` / `searx_rate_window` | SearXNG रूट के लिए समर्पित दर सीमा का कोटा और विंडो (s) | `40` अनुरोध / `60` s |
| `searx_max_query_len` | SearXNG रूट पर query-string का अधिकतम स्वीकार्य आकार | `4096` अक्षर |
| `suspicious_threshold` | स्थायी प्रतिबंध से पहले संदिग्ध hit की संख्या | `3` |
| `suspicious_window` | sliding विंडो (s) जिसमें संदिग्ध hit गिने जाते हैं | `300` (5 min) |
| `blocklist_refresh_interval` | ब्लॉक सूची फ़ाइल को दोबारा पढ़ने का अंतराल (s) | `5` |
| `whitelist_refresh_interval` | पाथ व्हाइटलिस्ट को दोबारा पढ़ने का अंतराल (s) | `15` |

omega-serv में, पहले 5 पाथ (`blocklist_file`, `waf_rules_dir`, `alerts_log`, `blocklist_log`, `rate_store_file`) WAF Magnet स्क्रीन में कॉन्फ़िगर किए गए पाथ से अपने-आप बदल दिए जाते हैं (वैश्विक तालिका `omega_magnet_paths`, निष्पादन से पहले इंजेक्ट की गई) — तब ऊपर के हार्डकोड मान केवल fallback का काम करते हैं, अगर यह तालिका मौजूद न हो।

> ℹ️ **2026-10-18 का बदलाव**: `blocklist_file` `secure/waf/magnet-blocklist.txt` से `var/lib/waf/magnet-blocklist.txt` पर चला गया है। स्क्रिप्ट हर प्रतिबंध पर इस फ़ाइल में **लिखती** है, और omega-serv की systemd सेवा केवल `var/` के अंदर ही लिख सकती है: पुराने स्थान के साथ कोई भी स्वचालित प्रतिबंध कभी दर्ज नहीं होता था, बिना किसी संदेश के। omega-serv शुरू होते समय पुरानी फ़ाइल को एक बार अपना लेता है (पुरानी फ़ाइल बैकअप के रूप में अपनी जगह रहती है); WAF Magnet स्क्रीन में सेट किया गया कस्टम पाथ कभी नहीं बदला जाता।

### नेटिव lighttpd में पाथ

> ⚠️ **ये 5 डिफ़ॉल्ट मान omega-serv के अपने फ़ोल्डर कन्वेंशन पर आधारित हैं** (`var/lib/waf/...`, `secure/waf/...`, `var/log/...`, `var/cache/...`, प्रोजेक्ट-जैसी कार्य डायरेक्टरी के सापेक्ष हल होने वाले relative पाथ) — ये तटस्थ नहीं हैं। **नेटिव lighttpd** में, जहाँ ऐसा कोई समतुल्य फ़ोल्डर कन्वेंशन नहीं है, ये relative पाथ रनटाइम पर lighttpd प्रोसेस की कार्य डायरेक्टरी (अक्सर `/`) के सापेक्ष हल होते हैं, जिसका अनुमान शायद ही लगाया जा सके। **इन 5 पंक्तियों को absolute पाथ से बदलें**, अपने इंस्टॉलेशन के अनुरूप, उदाहरण के लिए:
> ```lua
> local blocklist_file         = omega_paths.blocklist_file or "/var/lib/lighttpd/waf/magnet-blocklist.txt"
> local waf_rules_dir          = omega_paths.rules_dir or "/var/lib/lighttpd/waf/magnet-rules.d/"
> local alerts_log             = omega_paths.alerts_log or "/var/log/lighttpd/waf-magnet-alerts.log"
> local blocklist_log          = omega_paths.blocklist_log or "/var/log/lighttpd/waf-magnet-blocklist.log"
> local rate_store_file        = omega_paths.rate_store_file or "/var/cache/lighttpd/waf-magnet-rate_store.lua"
> ```
> रीस्टार्ट करने से पहले, ये फ़ोल्डर बनाएँ ([सेटअप](#सेटअप) के चरण 2 की कमांड) और ये अधिकार रखें:
>
> | फ़ाइल | lighttpd खाते को आवश्यक पहुँच |
> |---|---|
> | `blocklist_file`, `alerts_log`, `blocklist_log`, `rate_store_file` | **लिखना** (फ़ाइल और उसे रखने वाला फ़ोल्डर) |
> | `waf_rules_dir` (और `whitelist_paths.txt`) | केवल पढ़ना |
>
> लिखने में विफलता **मौन** होती है — स्क्रिप्ट कभी अनुरोध नहीं तोड़ती —: `blocklist_file` में न दिखने वाला प्रतिबंध ही एकमात्र लक्षण है। `sudo -u www-data touch /var/lib/lighttpd/waf/magnet-blocklist.txt` से जाँचें (खाता अपने अनुसार बदलें)। अगर lighttpd की systemd यूनिट लिखने पर रोक लगाती है (`ProtectSystem=strict`, `ReadWritePaths=`), तो ये फ़ोल्डर उसमें जोड़ें।

---

## निर्णय पाइपलाइन

स्क्रिप्ट पूरी तरह एक वैश्विक `pcall` के भीतर चलती है: कोई भी अप्रत्याशित Lua त्रुटि लॉग होती है और फिर `500` लौटाती है, सर्वर प्रोसेस को कभी क्रैश किए बिना। चरण इसी क्रम में चलते हैं, और हर चरण अंतिम प्रतिक्रिया के साथ प्रोसेसिंग रोक सकता है:

1. **ब्लॉक सूची** — अगर IP प्रतिबंधित है, अनुरोध अस्वीकृत (`451`), सिवाय इसके कि माँगा गया पाथ स्पष्ट रूप से व्हाइटलिस्ट में हो (उदाहरण के लिए वह त्रुटि पेज स्वयं परोसता हो)।
2. **संवेदनशील पाथ** — अगर पाथ किसी निगरानी वाले पैटर्न (`.git`, `.env`, `wp-admin`...) से मेल खाता है, तो पहली बार एक HTML चेतावनी पेज लौटाया जाता है (`403`); संदेह विंडो के भीतर पुनरावृत्ति प्रतिबंध ट्रिगर करती है (`451`)।
3. **स्कैनर User-Agent** — अगर User-Agent किसी ज्ञात स्कैनिंग टूल/बॉट से मेल खाता है, पहली बार `403`, पुनरावृत्ति पर `451`।
4. **वैश्विक दर सीमा** — अगर प्रति IP वैश्विक कोटा पार हो गया है, `429`।
5. **SearXNG नीति** — पहचाने गए खोज रूट के लिए: समर्पित दर सीमा (`429`), query-string के आकार पर सुरक्षा-सीमा (`414`), NUL बाइट की अस्वीकृति (`451`), फिर तत्काल निकास (`0`, जारी रखें) ताकि कोई वैध खोज कभी नीचे की आक्रामक जाँचों से न गुज़रे।
6. **पाथ सुरक्षा** — डिकोड किए गए पाथ में SQLi और XSS की खोज (पता चलने पर `451`); डायरेक्टरी ट्रैवर्सल की खोज (`451`)। इस चरण में जानबूझकर कमांड इंजेक्शन की **कोई** पहचान नहीं (देखें [ज्ञात सीमाएँ](#ज्ञात-सीमाएँ-और-दायरे-से-बाहर))।
7. **query-string की हल्की सुरक्षा** (SearXNG के बाहर) — डायरेक्टरी ट्रैवर्सल और NUL बाइट (`451`)।
8. **POST body निरीक्षण** — केवल तब जब `enable_body_inspection` सक्रिय हो और मेथड `POST` हो: बहुत बड़ा body सीधे अस्वीकृत (`413`); अन्यथा SQLi/XSS/CMDi/ट्रैवर्सल पर संचयी स्कोर, स्कोर `waf_score_threshold` तक पहुँचने पर अस्वीकृति (`451`), स्कोर 8 तक पहुँचने पर तत्काल प्रतिबंध, अन्यथा केवल एक संदिग्ध hit।
9. **कोई ब्लॉक नहीं** — अनुरोध सामान्य रूप से जारी रहता है (`0`)।

---

## पहचाने जाने वाले सिग्नेचर

चार पैटर्न परिवार (`patterns`), जिनकी हर प्रविष्टि या तो एक स्ट्रिंग (case-sensitive) होती है, या एक तालिका `{ pat = "pattern", ci = true }` (case-insensitive):

- **`sqli_body`**: `UNION SELECT`, `SELECT ... FROM`, `INSERT INTO`, `DROP TABLE`, `OR 1=1`, `' OR '1'='1`, `SLEEP(...)`, `BENCHMARK(...)`, अर्धविराम के ज़रिए SQL कमांड इंजेक्शन (`; DROP`, `; ALTER`, `; EXEC`, `; SHUTDOWN`)।
- **`xss_body`**: `<script>`, `<iframe>` टैग, `on*=` एट्रिब्यूट (इवेंट हैंडलर), `javascript:`, `eval(...)`, `document.cookie`, `svg/onload`।
- **`cmdi_body`**: shell कमांड की शृंखला (`; ls`, `; id`, `; whoami`, `; cat`, `; wget`, `; curl`, `; nc`, `; bash`, `; sh`, `; python`, `|` वाले रूप), कमांड प्रतिस्थापन (`` `...` ``, `$(...)`), PHP टैग (`<?php`) और निष्पादन फ़ंक्शन (`system(`, `exec(`, `passthru(`)।
- **`path_traversal`**: `../`, `..\`, `/proc/self/`, `etc/passwd`, `etc/shadow`, `file://`, `php://filter`।

साथ ही **स्कैनर/बॉट User-Agent** की एक समर्पित सूची (`patterns_scanner_ua`): `masscan`, `sqlmap`, `nmap`, `nikto`, `dirbuster`, `wfuzz`, `zgrab`, `python-requests`, `go-http-client`, SEO crawler (`ahrefsbot`, `mj12bot`, `semrushbot`, `blexbot`, `dotbot`, `spbot`, `yandexbot`, `sogoubot`), और सामान्य पैटर्न (`spider`, `crawler`, `bot`, `scraper`, `facebookexternalhit`, `lwp::simple`)।

निरीक्षण से पहले URL डिकोडिंग पुनरावर्ती रूप से लागू की जाती है (पाथ पर 5 स्तर तक, query-string पर 2 स्तर तक), ताकि दोहरे एन्कोड किए गए payload पकड़े जा सकें।

---

## निगरानी में संवेदनशील पाथ

डिकोड किए गए पाथ पर जाँचे जाने वाले पैटर्न: `.env`, `.git`, `.svn`, `.hg`, `.bzr`, `composer.json`/`composer.lock`, `vendor/`, `phpunit/`, `phpmyadmin`/`pma`/`adminer.php`, `wp-admin`/`wp-login.php`, `/server-status`, `.DS_Store`।

हर मेल के लिए, स्क्रिप्ट यह पता लगाने की कोशिश करती है कि उस पाथ पर वास्तव में कोई भौतिक संसाधन मौजूद है या नहीं (`lighty.stat` के ज़रिए, संगतता के लिए `lighty.c.stat` पर fallback के साथ) और इसे चेतावनी पेज और लॉग में दर्शाती है — अंधाधुंध स्कैन को सर्वर पर वास्तव में मौजूद पाथ से अलग पहचानने में उपयोगी।

---

## दर सीमा और escalation

दो स्वतंत्र rate limiter, डिस्क पर सहेजे जाने वाले इन-मेमोरी काउंटर पर आधारित (सरल sliding विंडो, सेकंड-दर-सेकंड सटीक sliding विंडो नहीं):

- **वैश्विक**: कुंजी `g:<ip>`, हर `global_rate_window` सेकंड में `global_rate_max_req` अनुरोध, सभी रूट पर।
- **SearXNG**: कुंजी `sx:<ip>`, हर `searx_rate_window` सेकंड में `searx_rate_max_req` अनुरोध, केवल पहचाने गए खोज रूट पर।

क्रमिक escalation (`record_suspicious_hit`) दर सीमा से अलग एक तंत्र है: हर गैर-गंभीर पहचान (बिना पुनरावृत्ति वाला संवेदनशील पाथ, स्कैनर User-Agent, POST body पर मामूली hit) `suspicious_window` सेकंड की विंडो पर प्रति IP एक काउंटर (कुंजी `susp:<ip>`) बढ़ाती है। इस विंडो में `suspicious_threshold` hit पूरे होते ही IP स्थायी ब्लॉक सूची में जोड़ दिया जाता है — इससे एक अकेले false positive पर सीधा प्रतिबंध टलता है, फिर भी पुनरावृत्ति करने वाले प्रतिबंधित होते हैं।

**कभी प्रतिबंधित न होने वाले नेटवर्क**: `never_auto_block` का कोई पता (स्थानीय host, Tailscale `100.64.0.0/10`, स्थानीय IPv4/IPv6 नेटवर्क, IPv4-mapped-IPv6 रूप सहित) कभी escalation या ब्लॉक सूची में जोड़ ट्रिगर नहीं करता; टाला गया प्रतिबंध `blocklist_log` में `BAN EVITE` पंक्ति से दर्ज होता है (कभी `AJOUT BLOCKLIST` नहीं, जिसे omega-jail पढ़ता है)। पोर्टेबल Lua में CIDR तुलना, 5.1 से 5.5 और LuaJIT तक।

**omega-serv का log-only मोड**: जब omega-serv का WAF `log-only` में होता है, तो स्क्रिप्ट कोई प्रतिबंध नहीं लगाती (न ब्लॉक सूची, न `AJOUT BLOCKLIST` पंक्ति, इसलिए fail2ban द्वारा कोई नेटवर्क प्रतिबंध नहीं); `BAN EVITE` पंक्ति इसे दर्ज करती है। यदि ब्लॉक सूची लिखी न जा सके, तो `AJOUT BLOCKLIST` की जगह `ECHEC BLOCKLIST` पंक्ति आती है।

---

## स्थायी भंडारण

| फ़ाइल | सामग्री | फ़ॉर्मेट |
|---|---|---|
| `blocklist_file` | हर पंक्ति में एक प्रतिबंधित IP, उससे पहले समय-मुहर वाली टिप्पणी (`# date - reason`) | सादा टेक्स्ट |
| `waf_rules_dir/whitelist_paths.txt` | हर पंक्ति में एक Lua पैटर्न (`string.match`), ऐसे पाथ जो प्रतिबंधित IP के लिए भी हमेशा अनुमत हैं | सादा टेक्स्ट |
| `rate_store_file` | दर सीमा और पुनरावृत्ति के काउंटर (हर कुंजी के लिए `{ count = N, first = timestamp }`) | serialized Lua तालिका (`return { ... }`, `load()` से दोबारा पढ़ी जाती है) |
| `alerts_log` | हर पहचान/अस्वीकृति के लिए एक पंक्ति | पठनीय टेक्स्ट, या `use_json_logs = true` होने पर JSON |
| `blocklist_log` | ब्लॉक सूची में हर जोड़ के लिए एक पंक्ति | पठनीय टेक्स्ट |

दर सीमा फ़ाइल हर अनुरोध पर कभी दोबारा नहीं लिखी जाती (आवधिक सफ़ाई के दौरान, अधिकतम हर 300 सेकंड में एक बार) ताकि असंगत I/O लागत से बचा जा सके। ब्लॉक सूची और व्हाइटलिस्ट मेमोरी में कैश होती हैं और अधिकतम हर `blocklist_refresh_interval`/`whitelist_refresh_interval` सेकंड में दोबारा पढ़ी जाती हैं — स्क्रिप्ट द्वारा स्वयं ट्रिगर किया गया जोड़ कैश को तुरंत अपडेट करता है, इस देरी की प्रतीक्षा किए बिना।

---

## लौटाए जाने वाले HTTP प्रतिक्रिया कोड

| कोड | अर्थ | ट्रिगर |
|---|---|---|
| `0` | सामान्य प्रोसेसिंग जारी (`mod_magnet` परिपाटी) | कोई पहचान नहीं |
| `403` | पहुँच अस्वीकृत, पहली चेतावनी | संवेदनशील पाथ मिला (पहली बार), स्कैनर User-Agent (पहली बार) |
| `413` | payload बहुत बड़ा | `body_max_inspect` से बड़ा POST body |
| `414` | URI बहुत लंबा | `searx_max_query_len` से लंबी SearXNG query-string |
| `429` | बहुत अधिक अनुरोध | वैश्विक या SearXNG दर सीमा पार |
| `451` | कानूनी कारणों से अनुपलब्ध (यहाँ प्रतिबंध कोड के रूप में प्रयुक्त) | IP पहले से प्रतिबंधित, escalation के बाद पुनरावृत्ति, पाथ या query-string में SQLi/XSS/ट्रैवर्सल मिला, NUL बाइट, POST स्कोर ≥ सीमा-मान |
| `500` | आंतरिक त्रुटि | अप्रत्याशित Lua अपवाद (वैश्विक `pcall` द्वारा पकड़ा गया) |

---

## संगतता

स्क्रिप्ट **दो संदर्भों** में काम करती है, बिना किसी भी बदलाव के:

- **नेटिव lighttpd**: `lighty.env`/`lighty.header`/`lighty.content`/`lighty.stat` `mod_magnet` मॉड्यूल के नेटिव API हैं। 5 कॉन्फ़िगरेशन पाथ सीधे अपने हार्डकोड डिफ़ॉल्ट मान उपयोग करते हैं।
- **omega-serv** (`infrastructure/waf/lua_waf_adapter.py`): एक Python-Lua ब्रिज (`lupa`) उसी `lighty.*` अनुबंध को हूबहू दोहराता है, अनुरोधों के बीच **स्थायी** Lua रनटाइम के साथ (इसलिए समय-आधारित कैश और दर सीमा एक अनुरोध से अगले तक सही ढंग से बने रहते हैं, ठीक lighttpd की तरह)। वैश्विक तालिका `omega_magnet_paths` निष्पादन से पहले WAF Magnet स्क्रीन के कॉन्फ़िगरेशन से इंजेक्ट की जाती है, ताकि 5 पाथ हार्डकोड मानों के बजाय इंटरफ़ेस में चुने गए स्थानों की ओर इशारा करें।

दोनों संदर्भों के बीच व्यावसायिक तर्क में कोई अंतर नहीं — केवल 5 कॉन्फ़िगरेशन पाथ का स्रोत बदलता है।

### क्या जैसा-का-तैसा संगत नहीं है

स्क्रिप्ट पूरी तरह `lighty.*` अनुबंध (`mod_magnet` का नेटिव API) पर निर्भर है — यह किसी दूसरे सर्वर पर जैसी-की-तैसी **नहीं** चलती, भले ही वह किसी रूप में Lua स्क्रिप्टिंग देता हो:

- **Apache** (`mod_lua`): अलग API, अनुरोध ऑब्जेक्ट `r` पर आधारित (`r.method`, `r.uri`, `r.headers_in`, `r:puts(...)`) — कोई भी `lighty.*` कॉल काम नहीं करेगी।
- **nginx** (`lua-nginx-module` / OpenResty): अलग API (`ngx.var`, `ngx.req`, `ngx.header`, `ngx.exit()`), निष्पादन चरणों के मॉडल के साथ (`access_by_lua`, `content_by_lua`...) जो `mod_magnet` की साधारण कोड-वापसी से भिन्न है।
- **Traefik**: कोई नेटिव Lua स्क्रिप्टिंग नहीं (Go में लिखा गया) — केवल एक Go प्लगइन (Yaegi) कस्टम तर्क चला सकता है, सीधे Lua कभी नहीं।
- **Caddy**: कोई आधिकारिक Lua मॉड्यूल नहीं — `xcaddy` से कंपाइल किया गया तृतीय-पक्ष प्लगइन चाहिए होगा, नेटिव रूप से कुछ भी समतुल्य उपलब्ध नहीं।

इस स्क्रिप्ट को इनमें से किसी सर्वर के साथ संगत बनाने के लिए एक समर्पित संगतता ब्रिज लिखना होगा जो उसके नेटिव API को `lighty.*` में अनुवादित करे (ठीक वही जो `lua_waf_adapter.py` omega-serv के लिए करता है) — यह अपने आप में पूरा इंजीनियरिंग कार्य है, कॉन्फ़िगरेशन का साधारण समायोजन नहीं।

---

## अनुकूलन

- **SearXNG के अलावा किसी अन्य एप्लिकेशन के लिए ढालना**: `is_searxng_path()` (पाथ की सूची) बदलें और, ज़रूरत हो तो, पाइपलाइन का खंड 3 (समर्पित नीति) पूरी तरह हटा दें, अगर संरक्षित एप्लिकेशन को खोजों के अलग व्यवहार की ज़रूरत नहीं है।
- **हमेशा अनुमत पाथ जोड़ना**: `waf_rules_dir/whitelist_paths.txt` संपादित करें (हर पंक्ति में एक Lua पैटर्न), या `is_whitelisted_path()` में एक हार्डकोड अपवाद जोड़ें।
- **आक्रामकता समायोजित करना**: `waf_score_threshold` (POST अस्वीकृति का सीमा-मान), `suspicious_threshold`/`suspicious_window` (स्थायी प्रतिबंध की ओर escalation की गति), `global_rate_max_req`/`global_rate_window` (सामान्य दर सीमा)।
- **लॉग को JSON में बदलना** (SIEM/log shipper में इनजेस्ट के लिए): `use_json_logs = true`।
- **बाकी को छुए बिना किसी उप-प्रणाली को अक्षम करना**: `enable_body_inspection`, `enable_rate_limit`, `enable_auto_block` स्वतंत्र हैं।

---

## ज्ञात सीमाएँ और दायरे से बाहर

- **URL पाथ पर कोई CMDi पहचान नहीं**: `.php` वाले पाथ पर false positive से बचने के लिए जानबूझकर किया गया चुनाव (अगर इस सतह पर CMDi पहचान में PHP टैग शामिल होता तो `.php` को `<?php` समझ लिया जाता)। POST body पर CMDi पहचान सक्रिय रहती है।
- **Lua पैटर्न मिलान से सिग्नेचर, असली PCRE इंजन नहीं**: सूचीबद्ध पैटर्न से न ढके गए हमले के रूप पकड़े नहीं जाएँगे — यह संपूर्ण नियम-समूहों पर आधारित WAF (जैसे ModSecurity CRS) का विकल्प नहीं है।
- **multipart/form-data पार्सिंग नहीं**: multipart फ़ॉर्म फ़ील्ड में लिपटा कोई दुर्भावनापूर्ण payload निरीक्षण से बच सकता है, अगर पैटर्न कच्ची, अनडिकोडेड स्ट्रीम से मेल नहीं खाते।
- **body का gzip डीकम्प्रेशन नहीं**: संपीड़ित body सादे रूप में निरीक्षित नहीं होता।
- **अनुमानित दर सीमा**: सरलीकृत sliding विंडो (विंडो पार होने पर पूर्ण रीसेट), सटीक sliding-विंडो एल्गोरिद्म (sliding log/token bucket) नहीं।
- **कोई निगरानी इंटरफ़ेस नहीं**: सारा अवलोकन लॉग फ़ाइलों और ब्लॉक सूची को सीधे पढ़कर होता है।

---

## समस्या निवारण

- **स्क्रिप्ट कभी चलती हुई नहीं लगती**: जाँचें कि `mod_magnet` ठीक से लोड है (lighttpd) या कि `magnet` extra इंस्टॉल है और WAF Magnet सक्रिय है (omega-serv); बदली गई Lua स्क्रिप्ट लोड करने के लिए कॉन्फ़िगरेशन रीलोड कभी पर्याप्त नहीं, दोनों स्थितियों में पूरा रीस्टार्ट ज़रूरी है।
- **दर सीमा के काउंटर हर अनुरोध पर रीसेट होते लगते हैं**: अनुरोधों के बीच गैर-स्थायी Lua रनटाइम का लक्षण (स्क्रिप्ट हर निष्पादन पर खाली `rate_store` से शुरू करती है) — omega-serv में, जाँचें कि WAF Magnet स्क्रीन एक सक्रिय स्थायी रनटाइम दिखा रही है।
- **कोई वैध IP प्रतिबंधित हो जाता है**: ट्रिगर हुए नियम की पहचान के लिए `blocklist_log` और `alerts_log` देखें, संबंधित पैटर्न या सीमा-मान समायोजित करें, फिर `blocklist_file` में उस IP की पंक्ति हटा दें (मेमोरी कैश अगली बार दोबारा पढ़ने पर अपडेट होता है, फ़ाइल बदलने के अधिकतम `blocklist_refresh_interval` सेकंड बाद)।
- **वैध SearXNG खोजें ब्लॉक हो रही हैं**: समर्पित दर सीमा के सीमा-मान बदलने से पहले `is_searxng_path()` जाँचें (पहचाना न गया पाथ)।
- **`pip install -e ".[magnet]"` `pip: command not found` या `externally-managed-environment` के साथ विफल होता है**: कमांड omega-serv के `.venv` के बाहर टाइप की गई थी (कुछ डिस्ट्रीब्यूशन, ख़ासकर Arch Linux, सीधे उपयोग योग्य कोई सिस्टम `pip` नहीं देते)। इसके बजाय omega-serv की डायरेक्टरी से `.venv/bin/python -m pip install -e ".[magnet]"` उपयोग करें — यह सिस्टम `PATH` पर `pip` के होने पर निर्भर हुए बिना काम करता है (देखें [omega-serv का `magnet` extra इंस्टॉल करना](#omega-serv-का-magnet-extra-इंस्टॉल-करना))।

---

## लाइसेंस

Omega-Waf **MIT** लाइसेंस के तहत वितरित है। पूरे पाठ के लिए [`LICENSE`](../../../LICENSE) फ़ाइल देखें।

---

> **Omega-Waf — एक स्क्रिप्ट, रक्षा की एक पंक्ति।**
>
> एक ही Lua फ़ाइल में mod_magnet एप्लिकेशन फ़ायरवॉल, बिना निर्भरता के, lighttpd और omega-serv के साथ संगत।
