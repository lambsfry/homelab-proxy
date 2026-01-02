# Homelab DNS & Internal TLS — Architecture Overview

This document provides a **high-level overview** of the home lab’s DNS, internal PKI, and reverse-proxy architecture. It is intended as a **mental model**, onboarding reference, and recovery guide.

For implementation details, see:

* `homelab-dns/README.md` — Authoritative DNS (AdGuard)
* `homelab-proxy/README.md` — Internal TLS & Reverse Proxy (Caddy)

---

## Goals & Design Principles

* **Single source of truth** (Git) for DNS and proxy configuration
* **Deterministic internal DNS** using `home.arpa`
* **One ingress IP** for all HTTPS services
* **Automatic internal TLS** with zero manual certificate management
* **Safe operations** with direct-access fallbacks (`*-direct.home.arpa`)
* **Rebuildable infrastructure** (no hidden state)

---

## High-Level Architecture

```mermaid
graph TD
    subgraph Clients
        Mac[Mac / iOS / Browsers]
        IOT[IoT Devices]
    end

    subgraph Network
        GW[Check Point Gateway\n192.168.1.1]
    end

    subgraph DNS
        AG[AdGuard Home\nCTID 102\n192.168.1.20]
    end

    subgraph Proxy
        Caddy[Caddy Reverse Proxy\nCTID 105\n192.168.1.3]
    end

    subgraph Services
        HA[Home Assistant\n192.168.1.4]
        PVE[Proxmox UI\n192.168.1.5 / 192.168.1.6]
        PBS[Proxmox Backup Server\n192.168.1.12]
        Other[Other Internal Services]
    end

    Clients -->|DNS queries| AG
    AG -->|A records for *.home.arpa| Clients

    Clients -->|HTTPS (443)| Caddy
    Caddy -->|HTTP / HTTPS| HA
    Caddy -->|HTTPS (8006)| PVE
    Caddy -->|HTTPS| PBS
    Caddy -->|HTTP / HTTPS| Other

    AG -->|Upstream DNS (DoT)| GW
```

---

## DNS Flow (Authoritative)

1. Clients use **AdGuard Home (192.168.1.20)** as their DNS server
2. AdGuard resolves `*.home.arpa` using a **generated hosts file**
3. Hostnames for user-facing services resolve to **192.168.1.3** (Caddy)
4. Direct aliases (`*-direct.home.arpa`) bypass the proxy if required

**DNS source of truth:**

* `network_objects.csv` (imported from Check Point)
* `overrides.csv` (explicit routing decisions)
* `generate_hosts_sectioned.py`

---

## HTTPS & TLS Flow

1. Client connects to `https://service.home.arpa`
2. DNS resolves to **192.168.1.3** (Caddy)
3. Caddy:

   * Terminates TLS using its **local CA**
   * Presents a trusted certificate
4. Caddy proxies traffic to the backend service IP

### Trust Model

* Caddy runs an **internal Certificate Authority**
* The **root CA** is installed once per client (System trust)
* All internal services automatically inherit trust

---

## GitOps Deployment Model

Both DNS and proxy layers follow the same pattern:

```text
Git repo (source of truth)
  ↓
bind-mount mirror (mount/)
  ↓
apply-* systemd service
  ↓
Live service configuration
```

### Key Properties

* No manual edits on running systems
* Reproducible from Git alone
* Safe rollbacks via Git history

---

## Day-2 Operations (BAU Summary)

### Add a new hostname

* Update `network_objects.csv` or `overrides.csv`
* Regenerate `hosts.home.arpa`
* Commit, push, deploy

### Add HTTPS for a service

* Create a new `sites.d/*.caddy` file
* Commit, push, deploy
* Add DNS override if needed

### Recover from failure

* Rebuild LXC
* Restore Git repo
* Recreate bind-mount
* Enable apply service

---

## Safety & Resilience

* All critical services retain **direct-access DNS names**
* Proxy is stateless and disposable
* DNS and TLS layers are independent
* No reliance on external PKI or internet reachability

---

## Scope Boundaries

This architecture intentionally does **not**:

* Expose services to the public internet
* Provide DHCP
* Perform authentication beyond TLS

These can be layered later if required.

---

## Status

This document reflects the **current stable baseline** of the home lab:

* AdGuard Home (CTID 102)
* Caddy Reverse Proxy (CTID 105)
* Internal TLS fully operational
* DNS → Proxy → Services flow validated
