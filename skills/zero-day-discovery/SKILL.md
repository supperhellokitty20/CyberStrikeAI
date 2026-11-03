---
name: zero-day-discovery
description: >-
  0day autonomous discovery engine: variant analysis/patch gaps/differential/Fuzzing/taint reasoning/N-day
  weaponization/hunter mindset. Use when public vulns not found and need to discover 0day or weaponize N-day.
metadata:
  tags: [penetration-testing, red-team]
---

## 0day Autonomous Discovery Engine (dig your own when no vulnerability can be found online)

```
Core shift: from "match known vulnerability databases" → "understand how code/protocols work and infer where they break". 0day is not luck; it is method.
Five main paths:
  1. Variant analysis (highest yield): take a CVE patch → extract the vulnerability pattern → grep the entire codebase for the same pattern elsewhere → places the patch did not cover = 0day
  2. Patch gaps: read the patch's filtering logic; blocklists can almost always be bypassed (missed encoding/equivalent function/alias) → new CVE
  3. Differential testing: two components understand the same input differently (WAF vs backend/validator vs executor) → smuggling/SSRF bypass
  4. Fuzzing: write a harness (wrap the function handling untrusted input) + build corpus/dictionary + crash triage (reproducible/controllable/exploitable)
     AFL++/libFuzzer (coverage-guided memory corruption) boofuzz (protocols) radamsa (black box) restler (REST API)
  5. Taint reasoning (strongest with source): source (parameter/Header/deserialized field) reaches a sink (exec/SQL/template) without an effective sanitizer = 0day
     Write CodeQL queries to automatically compute data-flow reachability / Semgrep / Joern
N-day weaponization (advisory out but no PoC anywhere): reverse-reconstruct an exploit from the patch diff (bindiff/diaphora binary diff;
  vendor regression tests are often a PoC prototype) → debug it in a local lab → hit the target. Maximize the vulnerability window; one of the most valuable red-team skills.
Hunter mindset (interrogate any code/endpoint/protocol layer by layer; every "yes" is a 0day candidate):
  Trust boundary: is the input assumed trustworthy? When does that fail? | State/timing: can state change between two steps (TOCTOU)? Can the order be scrambled?
  Parsing/normalization: how many times is it parsed? Is normalize before or after validation? | Boundary extremes: negative/0/oversized/type confusion/encoding/null byte?
  Implicit capability: what does this feature "incidentally" give me? | Uniqueness: are IDs/tokens predictable? Is the "secret" really secret?
Turn the component into a "list of assumptions" and break them one by one: file upload implicitly assumes "only images are uploaded/extensions are trustworthy/filenames contain no paths/content is just data" → break each = vulnerability, combine = chain.
0day verification (higher bar than CVE): reproducible (minimal PoC) + clear root cause (which line/which assumption) + provable impact (actual read/write/execute) + false-positive elimination.
```
