---
name: capability-primitive-search
description: >-
  Capability primitives + state-space search: combine read/write/exec/ssrf and other primitives into RCE equation A-F, low-severity mapping, forward/reverse search, cross-domain cash-out. Use when no single RCE, chaining low-severity vulns, or deriving novel attack chains.
metadata:
  tags: [pentesting, penetration-testing, red-team]
---

## Capability Primitives + State-Space Search

```
Core idea: RCE is not a "vulnerability"; it is an emergent property of a complete set of "capabilities." With no single big bug, several info-level/low-severity issues can still assemble code execution.
  Fixed mindset (break it): "No RCE/deserialization/upload found → this site is unpwnable." Your job is not to "find an RCE vulnerability" but to "assemble the primitives needed for execution."
  Abstract every Fact into capability primitives (don't record "found X vulnerability", record "what capability I now have + its limits"):
    read(path) write(path) exec(cmd) ssrf(url) sqli redirect(url) eval_expr idor(id) cred(svc,priv) coerce_auth write_acl

RCE only requires any one of these equations; break it down and assemble the obtainable primitives:
  A. Can write a file + the file is executed as code = RCE
  B. Can control config/env + config points to your code = RCE
  C. Can reach the admin plane + the panel has built-in execution = RCE (not a vulnerability, a feature!)
  D. Have credentials + the service has a legitimate execution entry = RCE (abuse legitimate functionality)
  E. Arbitrary read + read credentials + credentials can log in to an execution point = RCE
  F. Control data + data flows into a dangerous sink (eval/template/SQL) = RCE

Low-severity → primitive mapping (translate "low-value bugs" into puzzle pieces):
  info leak (.git/backup/stack trace) → source/paths/keys → feed B/E/F | LFI/arbitrary read → read config keys → E, or log poisoning → A
  SSRF (even GET-only) → hit internal Redis/Consul/K8s/cloud metadata → C; cloud metadata yields temporary credentials → D
  weak/default/reused credentials → enter an admin panel with tasks/plugins/webhooks/CI features → C | CORS/CSRF/XSS → use the admin's browser to invoke execution-class features → C
  controllable upload (even with an extension restriction) → pair with path traversal/parsing differences/.htaccess → A | config write → modify templates/logs/connection strings → B
  SQLi (even read-only) → read hashes/keys → E, or OUTFILE → A | controllable template → SSTI → F | prototype pollution → pollute downstream properties → F

State-space search (search it yourself when no ready-made chain exists): state = current capability set, action = use a capability to unlock a new capability, goal = Goal.
  Forward: for each capability ask "what can it unlock?"; for each pair of capabilities ask "what can they combine into?"
    (read+write = modify config; ssrf+internal redis = RCE; sqli+FILE = webshell; coerce_auth+relay = domain SYSTEM; idor+massassign = change another user's admin)
  Reverse (primary when stuck): lock Goal = execute commands → choose the equation closest to the current state as a template → set the missing primitive as a subgoal →
    which low-severity bug/feature/info leak on hand can produce it? → if not, recursively decompose/switch equations → forward and reverse meet in the middle = the full chain emerges → verify segment by segment

Breakthrough points (don't walk past them):
  · "Features are primitives": scheduled tasks/plugins/template editing/SQL console/file manager/import-export/webhooks in the admin panel — legitimate features, but once logged in they are ready-made execution/read-write primitives. To a pentester there is no "feature/vulnerability" distinction, only "capability."
  · Cross-protocol jumps: SSRF's gopher/dict/file turns "can only send HTTP" into "hit Redis/send SMTP/read files."
  · Credential reuse is universal glue: spray any password/key obtained anywhere across the whole network by default. Lateral movement is often faster than vertical.
  · Parsing differences: upload validation/routing/reverse proxy interpret things inconsistently → bypasses live in the gaps (double extensions/encoding/Host confusion).
  · Time dimension: TOCTOU/predictable tokens/cache poisoning turn "occasional" into "stable" and unexploitable into exploitable.
  · Cross-domain cash-out: for every capability gained ask "what is it worth in another domain" — capability is universal currency. Web SSRF → enter cloud metadata and take over the account; APK hardcoding → connect directly to internal APIs and bypass frontend auth; supply chain → obtain CI keys and enter production.
  · Creative mode (when known combinations are exhausted): re-examine capability boundaries (can only read /var/log? what about /proc/self/environ) | find sinks equivalent to RCE (writing crontab/.bashrc/CI config/LD_PRELOAD/authorized_keys/systemd unit all = RCE) | use information (errors/timing/response length) as a side channel | invert assumptions: list "I thought this was impossible" and ask "why is it impossible?" item by item
> A derived chain is a hypothesis (can be recorded tentatively as a note/chain); only after actual execution + evidence do you write a confirmed Fact / record_vulnerability. The entire chain holds only if every step has been verified.
```

