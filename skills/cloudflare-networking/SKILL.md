---
name: cloudflare-networking
description: Set up Cloudflare DNS, SSL, and direct port-443 serving for multiplayer-fabric. Use when adding a new machine, exposing a new service, or onboarding a zone node.
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: Cloudflare DNS and networking setup

## Concepts

| Component | Purpose |
|---|---|
| Cloudflare DNS (orange cloud) | Proxied — Cloudflare terminates TLS at the edge, forwards to origin |
| Cloudflare DNS (grey cloud) | DNS only — resolves directly to your IP, no proxying |
| Origin Certificate | Cloudflare-issued cert for your origin; only trusted by Cloudflare edge |
| Full (strict) SSL mode | Cloudflare validates origin cert before forwarding traffic |

---

## Hub server (HTTP/1.1 + HTTP/2 via Caddy on TCP 443)

### 1. Issue a Cloudflare Origin Certificate

Cloudflare dashboard → **SSL/TLS → Origin Server → Create Certificate**

- Key type: RSA
- Hostnames: `*.chibifire.com`, `chibifire.com`
- Validity: 15 years

Download **Origin Certificate** (`.crt`) and **Private Key** (`.key`).

### 2. Store cert files

```sh
mkdir -p multiplayer-fabric-hosting/certs
# paste cert → certs/origin.crt
# paste key  → certs/origin.key
chmod 600 certs/origin.key
```

Add to `.gitignore`:
```
certs/origin.key
```

Commit `certs/origin.crt` (public); never commit `origin.key`.

### 3. Configure Caddy (Caddyfile)

```
hub-<mac4>.chibifire.com {
    tls /etc/caddy/certs/origin.crt /etc/caddy/certs/origin.key

    handle_path /api/v1/* {
        reverse_proxy uro:4000
    }
    handle /uploads/* {
        reverse_proxy uro:4000
    }
    handle {
        reverse_proxy frontend:3000
    }
}
```

### 4. Expose ports in docker-compose

```yaml
zone-backend:
  image: caddy:2.9.1-alpine
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./Caddyfile:/etc/caddy/Caddyfile
    - ./certs:/etc/caddy/certs:ro
    - caddydata:/data
    - caddyconfig:/config
```

### 5. Set Cloudflare SSL mode to Full (strict)

Cloudflare dashboard → **SSL/TLS → Overview → Full (strict)**

### 6. Add DNS A/AAAA records

Cloudflare dashboard → **DNS → Records**:

| Type | Name | Content | Proxy |
|---|---|---|---|
| A | `hub-<mac4>` | `<public-ipv4>` | Orange cloud (Proxied) |
| AAAA | `hub-<mac4>` | `<public-ipv6>` | Orange cloud (Proxied) |

Get current public IPs:
```sh
curl -4 ifconfig.me   # IPv4
curl -6 ifconfig.me   # IPv6
```

### 7. Router port forwarding

Forward **TCP 443 → local IP:443** on the router.

Find local IP:
```sh
ipconfig getifaddr en0
```

Check if Tailscale is blocking port 443 before starting Caddy:
```sh
sudo lsof -iTCP:443 -sTCP:LISTEN
# If tailscaled is listed:
sudo tailscale set --webclient=false
```

### 8. Verify

```sh
# Local TLS (bypasses Cloudflare)
curl -sk --resolve "hub-<mac4>.chibifire.com:443:127.0.0.1" \
  -o /dev/null -w "%{http_code}" "https://hub-<mac4>.chibifire.com/"
# → 200

# External port reachability
curl -s "https://portchecker.io/api/v1/query" \
  -H "Content-Type: application/json" \
  -d '{"host":"<public-ipv4>","ports":["443"]}'
# → {"check":[{"port":443,"status":true}],...}

# Through Cloudflare
curl -so /dev/null -w "%{http_code}" "https://hub-<mac4>.chibifire.com/"
# → 200
```

---

## Zone server (WebTransport / QUIC on UDP 7443)

Zone servers use HTTP/3 WebTransport over **UDP**. Cloudflare cannot proxy raw QUIC,
so these records must be DNS-only (grey cloud) pointing directly to the public IP.

### DNS records

| Type | Name | Content | Proxy |
|---|---|---|---|
| A | `zone-<mac4>` | `<public-ipv4>` | Grey cloud (DNS only) |
| AAAA | `zone-<mac4>` | `<public-ipv6>` | Grey cloud (DNS only) |

### Router port forwarding

Forward **UDP 7443 → local IP:7443** on the router.

### Verify

```sh
dig +short zone-<mac4>.chibifire.com
nc -zvu zone-<mac4>.chibifire.com 7443
```

---

## Naming convention

Use the last 4 hex digits of the primary NIC MAC address as the node identifier:

```sh
ifconfig en0 | awk '/ether/{print $2}' | tr -d ':' | tail -c 5
# e.g. 700a → hub-700a, zone-700a
```

## DNS record comment template

```
Origin: <full-mac> en0. TCP 443 proxied via CF Full-strict + Origin Cert.
```
