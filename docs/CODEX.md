# AI Context — Homelab Infrastructure Rules

This repository uses a strict GitOps model.

## Authoritative Sources
- DNS:
  - Source: `network_objects.csv`, `overrides.csv`
  - Generated: `hosts.home.arpa`
  - Never edit generated files manually
- Proxy:
  - Source: `Caddyfile`, `sites.d/*.caddy`
  - Never edit `/etc/caddy` directly

## Deployment Model
- Git is the source of truth
- `mount/` directories are deployment artefacts
- Changes are applied via `apply-git-*.service`

## Invariants (Do Not Violate)
- DNS is authoritative before proxying
- TLS terminates at Caddy only
- `*-direct.home.arpa` aliases must remain available
- No manual config changes inside LXCs

## Assumptions
- Domain: `home.arpa`
- DNS: AdGuard Home (192.168.1.20)
- Proxy: Caddy (192.168.1.3)
- Platform: Proxmox LXC

When suggesting changes:
- Modify Git-managed files only
- Preserve sectioning and naming conventions
- Prefer minimal, explicit diffs
