# homelab-proxy — Internal TLS & Reverse Proxy (Caddy)

## Purpose

This repository defines the **internal reverse proxy and PKI termination layer** for the home lab using **Caddy**.

It provides:

* Automatic internal TLS
* A single ingress IP for services
* Consistent HTTPS for all internal apps
* Git-driven, reproducible configuration

---

## Architecture

| Component     | Value                 |
| ------------- | --------------------- |
| Reverse Proxy | Caddy                 |
| Proxmox LXC   | `caddy`               |
| CTID          | **105**               |
| IP Address    | **192.168.1.3**       |
| TLS CA        | Caddy Local Authority |
| Domain        | **home.arpa**         |

All HTTPS traffic terminates at Caddy.

---

## Trust Model

* Caddy runs an internal **local Certificate Authority**
* Root CA is installed **once per client** (System keychain on macOS)
* All proxied services automatically inherit trust

Certificate chain:

```
service.home.arpa
  ↳ Caddy Local Authority – ECC Intermediate
    ↳ Caddy Local Authority (Root, trusted)
```

---

## Repository Layout

```
Caddyfile
sites.d/
  proxy.caddy
  ha.caddy
  adguard.caddy
  proxmox.caddy
  pve.caddy
  pve02.caddy
mount/        (bind-mount mirror, not tracked)
```

---

## Configuration Files

### `Caddyfile`

Global configuration only:

* `local_certs`
* Import of site definitions

```caddy
{
  local_certs
}

import sites.d/*.caddy
```

### `sites.d/*.caddy`

One file per service. Examples:

#### Proxy self-test

```caddy
proxy.home.arpa {
  respond "Caddy proxy is alive"
}
```

#### Home Assistant

```caddy
ha.home.arpa {
  reverse_proxy 192.168.1.4:8123 {
    header_up Host {host}
    header_up X-Real-IP {remote_host}
    header_up X-Forwarded-For {remote_host}
    header_up X-Forwarded-Proto {scheme}
  }
}
```

---

## Deployment Model

```
Git repo
  ↓
bind-mount (/opt/homelab-proxy)
  ↓
apply-git-caddy.service
  ↓
/etc/caddy
  ↓
caddy reload
```

The `mount/` directory is intentionally excluded from Git.

---

## BAU — Adding a New Service

1. Create a site file:

   ```bash
   nano sites.d/myservice.caddy
   ```
2. Example:

   ```caddy
   myservice.home.arpa {
     reverse_proxy 192.168.1.50:8080
   }
   ```
3. Commit and push:

   ```bash
   git commit -am "Add myservice proxy"
   git push
   ```
4. Deploy:

   ```bash
   rsync -a --delete . mount --exclude .git --exclude mount
   systemctl start apply-git-caddy.service
   ```
5. Add DNS override if required

---

## Home Assistant Notes

Home Assistant requires trusted proxy configuration:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 192.168.1.3
```

---

## Safety Patterns

* Every proxied service has a `*-direct.home.arpa` DNS alias
* Proxy IP hosts **no backend services**
* TLS is always terminated at Caddy (no passthrough)

---

## Recovery

In the event of failure:

1. Rebuild the Caddy LXC
2. Restore Git repo
3. Recreate bind-mount
4. Enable `apply-git-caddy.service`

Certificates are regenerated automatically.

---

## Future Enhancements (Optional)

* Basic Auth on admin services
* mTLS for sensitive endpoints
* External ACME if ever exposed
* Per-VLAN routing or auth
