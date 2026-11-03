---
name: redteam-opsec
description: >-
  OPSEC covert operations discipline: IP blacklist bypass, rate/timing, traffic obfuscation, minimal footprint, anti-forensics, gradual exposure. Use when maintaining stealth, bypassing IP bans, or planning covert red-team ops.
metadata:
  tags: [penetration-testing, penetration-testing, red-team]
---

## OPSEC / Covert Operations Discipline (AV evasion / stability / stealth)

```
Core: Getting in ≠ staying stable. Discovery means the operation is over. Before every action, ask "what does this step look like to the defender?"
🚨IP blacklist bypass (try first after being banned): X-Forwarded-For: random IP (when the CDN trusts this header, bypass the application-layer blacklist directly; when it returns code:10010/blacklist, try XFF/X-Real-IP/CF-Connecting-IP/True-Client-IP one by one)
  Verification: a normal request returns "blacklist"+your IP → after adding the XFF header it returns a normal 200 → all subsequent requests carry XFF
  Advanced: change the XFF value every N requests (to avoid the new IP also being banned) | some CDNs trust only the first XFF value, some trust the last
Rate and timing: rate-limit scans (nuclei -rl / nmap -T2 --max-rate / ffuf -p delay) to avoid WAF bans + IDS threshold alerts | high-risk actions at low frequency + random jitter | avoid business peak hours and late night (matching the target's routine is least conspicuous)
Traffic obfuscation: blend with normal business traffic (common UA/Referer/legitimate paths) | probe defenses before heavy weapons (process list/known EDR/SIEM agent signatures) → if present, prefer silent techniques; only use automated mass exploitation when there is no monitoring
Minimal footprint: prioritize in-memory execution without touching disk (DDexec/memfd/reflective loading) | webshell strong password + uncommon path + functional disguise | tunnels over 443/DNS blending with common outbound traffic | delete tools after use (/dev/shm memory disk, leave no remnants)
Anti-forensics: command history unset HISTFILE / set +o history | selectively delete your own log entries (truncating everything actually triggers alerts) | timestamps touch -r against neighboring files to preserve mtime | do not touch monitoring/audit services (stopping them is an alert)
Gradual exposure: passive reconnaissance (certificate transparency/passive DNS/search engines/asset engines)→confirm there is no strong monitoring→then active scanning→only finally exploitation and access. Never actively touch the target for anything obtainable from public intelligence. At each escalation, ask "is it worth the exposure?"
> Stealth is not fastidiousness; it is a red team's survival capability. One reckless full-port full-speed scan can end the entire operation.
```

