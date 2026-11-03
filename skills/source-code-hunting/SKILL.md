---
name: source-code-hunting
description: >-
  Source-code hunting: .git leaks, dangerous-function grep, JS RC4 deobfuscation, semgrep/CodeQL, trufflehog, patch diff, supply chain/CI. Use when hunting source leaks, secrets, JS deobfuscation, or supply-chain issues.
metadata:
  tags: [pentesting, penetration-testing, red-team]
---

## Source-Code Hunting

```
=== Source-Code Hunting ===
.git leak: git-dumper → git log -p --all (deleted sensitive files) | .svn/.DS_Store/composer.lock
Dangerous-function grep: exec/system/eval/unserialize/pickle.loads/render/curl_exec + hardcoded secrets (sk-/ghp_/BEGIN RSA)
JS obfuscation cracking (RC4 + base64 string-array pattern): 1) extract the string array (var a0G=[...]) 2) find the decoder function (a0m(idx,key)→RC4 decrypt + base64) 3) find the rotation IIFE (target offset) 4) rebuild the decoder in Node.js to batch-decode all strings → recover plaintext variable names/API paths/config
  UniApp characteristics: app-service.js (business logic, often 800KB+ of obfuscation) + zlsioh.dat (encrypted config, decrypted by a native .so) + dcloud_uniplugins.json (plugin manifest)
Static scanning: semgrep --config=auto for a quick scan / build a CodeQL database and write queries (see `zero-day-discovery` for taint-solving + variant-analysis methodology)
Deep secrets hunting: trufflehog/gitleaks scan the full git history + Docker image layers + npm/PyPI tarballs + frontend bundles (--only-verified distinguishes live from dead keys)
Framework patch diff: clone the before/after versions and diff → what was fixed = where the vulnerability is | dependency chain: composer.json/npm audit/pip-audit
Supply chain/CI: dependency confusion (claim internal package names on public registries) | GHA command injection ${{github.event.issue.title}} | self-hosted runner takeover | .npmrc/.pypirc credentials
```
