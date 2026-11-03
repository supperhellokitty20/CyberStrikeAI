---
name: proxy-tool-bootstrap
description: >-
  Find your own proxies + tool bootstrap: SOCKS5/HTTP/Tor rotation sequence, Python tool bootstrap, wordlist self-generation, OOB infrastructure. Use when blocked by 403/429/WAF/timeout, missing tools, or needing OOB confirmation.
metadata:
  tags: [penetration-testing, penetration-testing, red-team]
---

## Find Your Own Proxies + Tool Bootstrap (switch routes when blocked, write your own tools when missing)

```
🔴Proxy (on rejection/rate limiting/timeout → the first reaction is not to retry, but to switch routes. Not switching routes = giving up = violating blackboard trigger 2 (see `pentest-blackboard`)):
  Execution sequence (in order; proceed to the next step only if the previous one fails):
  ①Probe the target's region: terminal: curl -s "http://ip-api.com/json/{TARGET_IP}" | python3 -c "import sys,json;d=json.load(sys.stdin);print(d['country'],d['countryCode'])"
  ②Obtain a SOCKS5 proxy for that region: terminal: curl -s "https://api.proxyscrape.com/v2/?request=displayproxies&protocol=socks5&country={countryCode}&timeout=5000" | head -20
  ③Verify the proxy can reach the target: terminal: curl --socks5 {PROXY_IP:PORT} --connect-timeout 8 -s -o /dev/null -w "%{http_code}" {TARGET_URL}
  ④200=usable, immediately retry the blocked request through this proxy; non-200=switch to the next proxy and repeat ③
  ⑤All SOCKS5 proxies fail→obtain an HTTP proxy: terminal: curl -s "https://api.proxyscrape.com/v2/?request=displayproxies&protocol=http&country={countryCode}&timeout=5000" | head -20
  ⑥HTTP proxy verification: terminal: curl --proxy http://{PROXY_IP:PORT} --connect-timeout 8 -s -o /dev/null -w "%{http_code}" {TARGET_URL}
  ⑦All proxies fail→Tor: terminal: curl --socks5 127.0.0.1:9050 --connect-timeout 15 {TARGET_URL}
  Add proxy parameters to all tools uniformly: curl --socks5 / sqlmap --proxy=socks5://{P} / nmap --proxies socks5://{P} / nuclei -proxy socks5://{P} / ffuf -x socks5://{P}
  Rotation strategy: 429/403→immediately switch to the next proxy, proactively rotate every 20 requests (to avoid the new IP also being banned) | Cloudflare→proxy pool + random request interval of 2-5s
  🚨HTTP proxy vs SOCKS5: HTTP proxies insert their own error pages (502/cannot display this page), making it impossible to distinguish the target's actual response→SOCKS5 (--socks5) must be used when probing
    ProxyScrape SOCKS5: https://api.proxyscrape.com/v2/?request=displayproxies&protocol=socks5&country=CN,JP&timeout=5000
    Verify SOCKS5: curl --socks5 IP:PORT --connect-timeout 5 target | HTTP proxies are only suitable for anonymity/rotation against targets already confirmed reachable
Tool bootstrap (which X || implement in Python):
  No nmap→socket port scan | no ffuf→requests directory brute force | no sqlmap→manual payload testing | no hydra→requests brute force | no nuclei→requests send known payloads
  Simulate complex tools in Python: crawler requests+bs4 / encoding base64/hex / hashing hashlib / encryption pycryptodome / sniffing scapy
Wordlist self-generation: build variants from the target domain/company name | extract keywords from web pages | username+year+special character combinations | service default credentials
OOB infrastructure (all blind vulnerabilities rely on it): interactsh-client to get an oast.fun domain | or VPS python3 -m http.server/nc to watch callbacks | ngrok/cloudflared tunnel
  → Only when an OOB callback (DNS query/HTTP request) is seen is it confirmed → write a Fact
```

