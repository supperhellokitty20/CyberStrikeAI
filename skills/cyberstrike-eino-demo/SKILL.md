---
name: cyberstrike-eino-demo
description: >-
  Full-featured demo skill package: SKILL.md plus optional scripts/, references/, and assets/ directories;
  verifies the Eino skill loader and HTTP package-relative paths (authorized security testing and training only).
---

# CyberStrike × Eino Full-Featured Skill Demo

This package follows the [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) layout: **`SKILL.md` is the manifest plus the main instructions** (there is no separate `SKILL.yaml`). The same directory may contain arbitrary subdirectories such as **`scripts/`**, **`references/`**, and **`assets/`** (as long as paths are safe and stay within the package depth/file-count limits); they are read by **`ListPackageFiles` / `resource_path`** and by the Eino host file tools. Additional notes are in `FORMS.md` and `REFERENCE.md`.

## Overview

Used for one-time verification of:

- The HTTP `GET /api/skills` listing (`script_count`, `file_count`, `progressive`, etc. are derived/scan results)
- `GET /api/skills/cyberstrike-eino-demo?depth=summary|full`
- `section=` mapping to a **`##` heading** in `SKILL.md` or a short ASCII-heading id (for example, `## Payload Samples` typically maps to `section=payload`)
- The ADK **`skill`** tool inside multi-agent sessions (plus optional host file tools) reading package-relative resources
- Eino `FilesystemSkillsRetriever` retrieval over the package summary, `##` sections, and script entries

**Hard requirement:** any testing must have written authorization and must stay within the agreed scope and time window.

## Authorized Testing Workflow

1. **Scope confirmation**: domains / IPs, endpoint list, prohibited actions (DoS, bulk data exfiltration, etc.).
2. **Baseline recording**: perform read-only probing of the agreed assets and save timestamps plus raw request/response summaries.
3. **Categorized testing**: split tasks by vulnerability class; re-confirm the authorization boundary before any high-risk action.
4. **Evidence and reporting**: attach reproduction steps, impact, and remediation to every finding; redact sensitive data.
5. **Cleanup**: delete temporary accounts, remove test data, and hand over the report.

## Payload Samples

The following are **training placeholders**; replace them with target-specific context during real testing and never use them against unauthorized systems:

- SQLi probe (error-based): `"'` (observe whether a database error message is disclosed)
- Reflected XSS (harmless): `<script>alert(1)</script>` → a lab target should encode it or block it with CSP
- Path traversal (read-only verification): `....//....//etc/passwd` (only in authorized file-read scenarios)

The full list is in `scripts/payloads.txt`.

## references/ and assets/

Used to verify that subdirectories other than `scripts/` are treated the same way:

| Path | Purpose |
|------|---------|
| `references/citations.md` | Citation and HTTP `resource_path` test notes |
| `assets/README.txt` | Placeholder resource (can be replaced with a real binary to test the file-read size limit) |

## Recommended Toolchain

| Phase | Example tools |
|-------|---------------|
| Proxy and replay | Burp Suite, mitmproxy |
| Scanning and directories | ffuf, nuclei (lower concurrency to stay within authorization) |
| Vulnerability validation | Self-written PoCs, official CLIs (sqlmap, etc.) only within the authorized scope |
| Record keeping | Markdown + JSON snippet templates (see `scripts/report-snippet.json`) |

## Checklist and Verification

- [ ] Written authorization and testing window saved
- [ ] Files under `scripts/` match the references in the body
- [ ] The index can be checked via the Web UI or `GET /api/skills?...`; inside a multi-agent session, load packages with the **`skill`** tool to save tokens
- [ ] Pull the full text via **`skill`**, or via HTTP `depth=full` / `section=<heading or short id>` when details are needed
- [ ] Read script source via the host file tool or HTTP `resource_path=scripts/check-env.sh`
- [ ] `resource_path=references/citations.md` and `resource_path=assets/README.txt` are readable
