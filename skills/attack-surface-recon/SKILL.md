---
name: attack-surface-recon
description: >-
  Recon/attack-surface mapping: passive whois/amass/crt.sh/FOFA/Shodan; active subfinder/httpx/naabu/katana/nuclei; DNS geolocation/CDN/Nginx catch-all/BT Panel/UniApp fingerprinting. The first action at kickoff; write findings to the project blackboard. Use when starting recon, asset mapping, fingerprinting, or CDN/DNS bypass discovery.
metadata:
  tags: [pentesting, penetration-testing, red-team]
---

## Recon / Attack-Surface Mapping

```
=== Recon/Attack-Surface Mapping (first action at kickoff; 60% of engagement time is here — don't rush into exploitation naked; immediately upsert ports/fingerprints etc. with upsert_project_fact) ===
Passive first (do not touch the target; run all of these):
  terminal: whois {domain} | grep -iE 'org|name|email|registrant'
  terminal: amass intel -asn {ASN} -d {domain}
  browser_navigate: https://crt.sh/?q=%.{domain} (certificate transparency subdomain mining)
  terminal: curl -s "https://crt.sh/?q=%.{domain}&output=json" | python3 -c "import sys,json;[print(x['name_value']) for x in json.load(sys.stdin)]" | sort -u
  terminal: dig +short {domain} @114.114.114.114; dig +short {domain} @8.8.8.8 (difference = CDN)
  terminal: curl -s "https://web.archive.org/cdx/search/cdx?url=*.{domain}/*&output=text&fl=original&collapse=urlkey" | head -100
  browser_navigate: https://fofa.info/result?qbase64=$(echo -n 'domain="{domain}"' | base64)
  browser_navigate: https://www.shodan.io/search?query=hostname:{domain}
Active pipeline (execute step by step):
  terminal: subfinder -d {domain} -silent | tee subs.txt; amass enum -d {domain} -passive >> subs.txt; sort -u subs.txt -o subs.txt
  terminal: cat subs.txt | dnsx -silent | tee alive_subs.txt
  terminal: cat alive_subs.txt | httpx -silent -title -status-code -tech-detect -cdn | tee httpx_out.txt
  terminal: naabu -l alive_subs.txt -top-ports 1000 -silent | tee ports.txt; nmap -sCV -iL <(head -20 alive_subs.txt) -oN nmap_out.txt
  terminal: katana -list alive_subs.txt -silent -d 3 | tee urls.txt; gau {domain} >> urls.txt
  terminal: cat urls.txt | grep -iE '\.(js|json)$' | httpx -silent -mc 200 | while read u; do curl -s "$u" | grep -oiE '(api|secret|key|token|password|aws|endpoint)[^"]*'; done
  terminal: ffuf -u https://{target}/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -mc 200,301,302,403 -fs {catch-all-size}
  terminal: nuclei -l alive_subs.txt -t /root/nuclei-templates/ -severity medium,high,critical -o nuclei_results.txt
  → If the httpx output identifies a framework/version → immediately trigger the `component-vuln-intel` search sequence
🚨 DNS geolocation restriction bypass: Chinese CDNs (987dns/dnspod/Alibaba Cloud) often return 0.0.0.0 to overseas DNS; only resolution via 114.114.114.114 yields the real IP. Compare multiple DNS servers: dig @1.1.1.1 vs @114.114.114.114 vs @8.8.8.8; any difference is CDN geolocation policy. The real IP is hidden in the Chinese DNS results.
🚨 CDN response filtering bypass (GCCDN/PWS, etc.): the CDN returns 502 for specific URL patterns (admin paths/actuator) instead of forwarding the backend response:
  Fingerprint: Server: PWS/x.x.x.x | Via: 1.1 PS-XXX-XXXX:N (W) | X-Px: ms CS-XXX-XXXXnone(origin) | wildcard *.gccdn.net CNAME
  502 ≠ not found: 502 = backend response filtered by CDN; 404 = path truly does not exist; 403 = CDN route denied. Use the 502/404/403 differences to enumerate live endpoints
  Path traversal confirmation: /api/v1/v/..;/..;/actuator/env → 502 (exists but filtered) vs /api/v1/v/..;/..;/xxxx → 404 (natively absent in Tomcat)
  CDN node multi-port: scan ports 3000-9200 on the CDN IP (8000/8080/9090 often proxy the same backend under different rules; 9200 may be an independent nginx)
  Bypass attempt matrix (all tested and failed this session; record to avoid repetition): URL encoding/double encoding/case variation/semicolon suffix/TE-space smuggling/HTTP pipelining/Accept-Encoding/Range
  Useful information extraction: even with 502, POST actuator/env may still be executed by the backend (blind property write) → needs a refresh to trigger (if refresh is 404, the chain breaks)
🚨 Nginx catch-all trap: a PHP CMS with rewrite on every route → all paths return 200 with the same size = catch-all (not proof a file exists!). Identify: compare 3 random paths; identical size = catch-all. Test truly live paths with a .php suffix (nginx `location ~ \.php$` forwards directly to FastCGI; nonexistent paths return a real 404).
🚨 Pure-FTPd puredb corruption ≠ auth bypass: "Unable to read the indexed puredb file" = the virtual-user DB is broken; every username (including anonymous) triggers a 421 disconnect. PAM system-user auth also goes through puredb first → all fail. Not an attack surface; skip.
🚨 Same-IP multi-site lateral movement: reverse-lookup the IP (SecurityTrails/ThreatBook/VirusTotal) to find other domains on the same server → breach the weaker site (an independent nginx config returning a real 404 = enumerable) → a shared BT Panel means lateral movement to every site.
Fingerprint priority: once httpx/wappalyzer identifies a framework version → immediately switch to `component-vuln-intel` online intelligence. Map the attack surface first, then reason out the fattest edge; don't dive headfirst into the first hole you see.
🚨 DNS wildcard CDN detection: every subdomain (including nonexistent random subdomains) resolves to the CDN = wildcard DNS (*.domain → CDN CNAME). Subdomain enumeration is pointless here; every IP is a CDN node.
  Verify: dig random123456.target.com → if it also has an A record pointing to a known CDN IP = wildcard. Obtain the real origin IP through other channels (historical DNS/certificate transparency/other domains on the same server/mail headers/SSRF)
🚨 BT Panel detection: a cookie named as a 32-char hex hash (e.g. 789d6d2de16419a1e8bbfea926c8996e) + 302 redirect to /login + nginx reverse proxy + an "entry verification failed" page = BT Panel random security entry
  Windows edition: IIS + nginx coexist; panel on port 8888; phpMyAdmin on port 888 (possibly); /index.html may leak site configuration (reverse-proxy target URL/site name)
  Attack: brute-force the entry path (8 random alphanumeric characters) | historical CVEs (unauthenticated API key/phar deserialization) | reuse default panel FTP/MySQL accounts
🚨 Nginx 404 differential fingerprinting: when an nginx catch-all rewrite masks real paths, compare response sizes — a real nginx 404 (146B) = the file exists on disk but the location block denies it; catch-all (large page) = route did not match. Batch-probe CMS signature paths (/caches/configs/, /phpcms/modules/, /statics/, etc.) and classify by response size to confirm the CMS type. See references/nginx-404-differential-fingerprinting.md for details.
🚨 UniApp/DCloud APK reversing fast path: assets/apps/__UNI__*/www/app-service.js = all business logic (obfuscated JS); assets/zlsioh.dat = encrypted config (API domain); lib/arm64-v8a/libirzlemnr.so = decryption library. Deobfuscate the JS with RC4 + base64 (extract the a0G array + a0m decoder); in the .so, strings-search for a 16-character random string = possible key, and search for the `x<digit>.<digit>.<digit>.<digit>x` format = hardcoded IP (usually DCloud HTTPDNS infrastructure, not the target API). See references/uniapp-dcloud-apk-reversing.md for details.
```
