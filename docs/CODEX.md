# AI Context — Homelab Infrastructure (Codex / ChatGPT Guardrails)

This document defines **non‑negotiable rules, assumptions, and invariants** for this repository. It exists to keep AI assistants (Codex / ChatGPT) aligned with the actual operating model of the homelab.

If you are an AI assistant reading this file:

* Treat it as **authoritative**
* Prefer **minimal, explicit diffs**
* Do **not** suggest shortcuts that bypass these rules

---

## Repository Scope

This repository is part of a **Git‑managed homelab** running on **Proxmox**.

It participates in a **two‑layer control plane**:

1. **DNS layer** — AdGuard Home (`homelab-dns` repo)
2. **Ingress / TLS layer** — Caddy reverse proxy (`homelab-proxy` repo)

DNS decides **where** traffic goes.
Caddy decides **how** traffic arrives.

---

## Authoritative Sources (Do Not Bypass)

### DNS (homelab-dns)

**Source of truth**:

* `network_objects.csv` — imported from Check Point gateway
* `overrides.csv` — explicit routing and safety overrides

**Generated output**:

* `hosts.home.arpa`

Rules:

* `hosts.home.arpa` is **generated** — never edit by hand
* Manual edits to `/etc/hosts` are forbidden
* Overrides always win over imported objects

---

### Proxy / TLS (homelab-proxy)

**Source of truth**:

* `Caddyfile`
* `sites.d/*.caddy`

Rules:

* Never edit `/etc/caddy` directly
* All changes must occur in Git‑managed files
* One service per `*.caddy` file

---

## Deployment Model (GitOps)

All runtime configuration is derived from Git:

```
Git repo (authoritative)
  ↓
bind‑mount mirror (mount/)
  ↓
apply‑git‑*.service (systemd)
  ↓
Live service configuration
```

Rules:

* `mount/` directories are **deployment artefacts**
* `mount/` must never be committed to Git
* LXCs are disposable and rebuildable

---

## Networking & Identity Assumptions

* Domain: `home.arpa`
* DNS server: AdGuard Home (`192.168.1.20`)
* Reverse proxy: Caddy (`192.168.1.3`)
* Platform: Proxmox LXC

There is **no public exposure** and **no external ACME**.

---

## TLS & Trust Model

* Caddy runs an **internal Certificate Authority**
* TLS always terminates at Caddy
* Clients trust the **Caddy root CA** (System trust)
* Backends may use HTTP or HTTPS

Rules:

* Do not suggest TLS passthrough
* Do not introduce external PKI unless explicitly requested

---

## Safety Invariants (Must Hold)

* Every proxied service retains a `*-direct.home.arpa` DNS alias
* Proxy IP hosts **no backend services**
* DNS and proxy layers are independently recoverable
* No manual config drift inside running LXCs

If a suggestion violates any invariant above, it is invalid.

---

## Home Assistant Specifics

When interacting with Home Assistant:

* Proxy IP (`192.168.1.3`) must be listed in `trusted_proxies`
* `X‑Forwarded‑*` headers are required
* 400 responses usually indicate missing proxy trust

---

## Change Philosophy

When proposing changes:

* Prefer **small, reversible steps**
* Explain *why* a change is required
* Preserve naming, sectioning, and conventions
* Avoid unnecessary abstraction or consolidation

This homelab values **clarity and recoverability over cleverness**.

---

## Summary for AI Assistants

✔ Modify Git‑managed files only
✔ Respect generated vs source files
✔ Assume rebuildability
✔ Preserve safety aliases
✔ Keep DNS authoritative
✔ Keep TLS centralized

If uncertain, **ask before changing structure**.
