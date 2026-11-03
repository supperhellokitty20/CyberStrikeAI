---
name: active-directory-attack
description: >-
  Internal domain attacks: BloodHound, Kerberoast, ADCS ESC1/ESC8, NTLM Relay, Coerce, DACL, DCSync, Zerologon/NoPac/PrintNightmare, mitm6, LLMNR, Linux internal network. Use when attacking Active Directory, ADCS, NTLM relay, or internal domain.
metadata:
  tags: [penetration-testing, penetration-testing, red-team]
---

## Internal Domain Attacks

```
=== Internal Domain (the real main battlefield in 2023+) ===
Recon: BloodHound (SharpHound collect → attack paths) | Kerberos: GetNPUsers (AS-REP)/GetUserSPNs (Kerberoast)
🚨ADCS (Certipy one-shot): certipy find -vulnerable | ESC1 specify SAN to request a domain admin certificate | ESC8 relay to CA to get a DC certificate
🚨NTLM Relay (more important than PtH, which is often blocked by EDR): ntlmrelayx -t ldap--escalate-user / -t http CA --adcs (ESC8) / RBCD
🚨Coerce authentication: PetitPotam (MS-EFSRPC)/coercer spray all protocols/printerbug → feed to relay
🚨DACL abuse: WriteDACL → add DCSync for yourself | Shadow Credentials certipy shadow (GenericWrite is enough; no password change, no trace)
DCSync: secretsdump -just-dc → krbtgt hash → Golden Ticket
🚨One-shot domain CVEs (test first; a hit gives domain admin directly): Zerologon (CVE-2020-1472, blank the DC machine account password → DCSync) | NoPac (CVE-2021-42278/42287, rename a machine account and request a DC TGT) | PrintNightmare (CVE-2021-34527, print spooler RCE/load a malicious driver) | EternalBlue (MS17-010, direct RCE on old SMBv1)
🚨IPv6/mitm6 (must attack on default dual-stack internal networks): mitm6 hijacks DHCPv6+DNS → WPAD → ntlmrelayx to LDAP/ADCS (stealthier than LLMNR; first choice on modern internal networks)
LLMNR/NBT-NS poisoning: responder captures NetNTLMv2 → hashcat crack/relay
Linux internal network: Redis unauthorized access (CONFIG SET dir writes an SSH key) | NFS showmount | Docker 2375
```
