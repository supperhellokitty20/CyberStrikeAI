---
name: initial-access-phishing
description: >-
  Initial access/phishing/social engineering: credential spraying, AiTM, device code, OAuth consent phishing, payloads, vishing. Use when needing initial access, phishing, AiTM, device code, or social engineering.
metadata:
  tags: [penetration-testing, penetration-testing, red-team]
---

## Initial Access / Phishing / Social Engineering

```
=== Initial access/phishing/social engineering (first hop from outside to inside, recon before the hit) ===
Credential spraying: weak passwords (Season+Year!/companyname123) sprayed across all accounts (1-2 attempts per account to avoid lockout) | Sources: breach databases/OSINT email formats (f.last@)/default credentials
AiTM phishing (bypasses MFA): evilginx3/Modlishka reverse-proxy the real site→man-in-the-middle steals session cookie+token (bypasses MFA) | phishlet configures domain+certificate
Device code phishing (Azure/M365): device code flow→lure the victim into entering the code→obtain access/refresh token (no password/MFA needed) | TokenTactics/AADInternals
Phishing payloads: gophish sending+landing page | payloads lnk/iso/macro/HTA/OneNote | gateway bypass: password-protected zip/cloud drive links/HTML smuggling
OAuth consent phishing (illicit consent): malicious App requests excessive scopes (Mail.Read/offline_access)→victim consents→long-lived access token
Social engineering pretext: OSINT org structure/vendors/password habits (LinkedIn/job postings/GitHub) | pretext (IT support/vendor/HR) | vishing (deepfake voice)
→ Once initial access lands, switch to `redteam-opsec` (don't get caught by EDR the moment you walk in); feed obtained capabilities into `capability-primitive-search` to build the chain
```
