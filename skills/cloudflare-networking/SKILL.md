---
name: cloudflare-networking
description: Set up Cloudflare DNS records and Cloudflare Tunnel for multiplayer-fabric. Use when adding a new machine, exposing a new service, or onboarding a zone node.
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: Cloudflare DNS and Tunnel setup

## Concepts

| Component | Purpose |
|---|---|
| Cloudflare DNS (orange cloud) | Proxied — Cloudflare terminates TLS, serves HTTP/1.1 and HTTP/3 to clients |
| Cloudflare DNS (grey cloud) | DNS only — resolves directly to your IP, no proxying |
| Cloudflare Tunnel (`cloudflared`) | Outbound-only agent that connects your LAN service to Cloudflare's edge without opening inbound firewall ports |

## Service routing table

| Service | DNS record | Proxy | Transport |
|---|---|---|---|
| zone-backend (Phoenix API) | `hub-<mac4>.chibifire.com` → tunnel `multiplayer_fabric` | Orange cloud | HTTP/1.1 + HTTP/3 via Cloudflare edge |
| zone-server (Godot WebTransport) | `zone-<mac4>.chibifire.com` → public IP | Grey cloud (DNS only) | QUIC/UDP direct — Cloudflare cannot proxy raw QUIC |

Cloudflare Tunnel **cannot** proxy WebTransport sessions. Zone servers must be
reachable via direct UDP.

## Naming convention

Use the last 4 hex digits of the NIC MAC address as the node identifier.

```sh
# Get last 4 of en0 MAC
ifconfig en0 | awk '/ether/{print $2}' | tr -d ':' | tail -c 5
# e.g. 700a → hub-700a, zone-700a
```

## Adding a hub record (zone-backend via tunnel)

1. In Cloudflare Zero Trust dashboard → Networks → Tunnels → `multiplayer_fabric` → Public Hostnames → Add:
   - Subdomain: `hub-<mac4>`
   - Domain: `chibifire.com`
   - Service: `http://zone-backend:4000`
2. Cloudflare auto-creates the CNAME with orange cloud. No manual DNS entry needed.
3. Verify:
   ```sh
   dig hub-<mac4>.chibifire.com @1.1.1.1
   curl -sv https://hub-<mac4>.chibifire.com/
   ```

## Adding a zone record (WebTransport direct)

1. Get the machine's public IPv4:
   ```sh
   curl -4 ifconfig.me
   ```
2. Confirm UDP 443 is reachable (port-forward on router: UDP 443 → local IP):
   ```sh
   nc -zvu <public-ip> 443
   ```
3. Add DNS A record in Cloudflare dashboard:
   - Name: `zone-<mac4>`
   - Content: `<public-ip>`
   - Proxy: **DNS only** (grey cloud)
   - Comment: `WebTransport QUIC/UDP 443 zone server. NIC en0 <full-mac>. DNS-only, no proxy.`
4. Verify:
   ```sh
   dig +short zone-<mac4>.chibifire.com
   nc -zvu zone-<mac4>.chibifire.com 443
   ```

## Installing and authenticating cloudflared

```sh
brew install cloudflared
cloudflared tunnel login        # opens browser, writes ~/.cloudflared/cert.pem
cloudflared tunnel list         # confirm multiplayer_fabric tunnel is visible
```

## Getting the tunnel token (for docker-compose)

```sh
cloudflared tunnel token multiplayer_fabric
```

Paste this value into `.env` as `CLOUDFLARE_TUNNEL_TOKEN`. The `.env` file is
gitignored — never commit the token.

## Verifying the tunnel is connected

```sh
cloudflared tunnel info multiplayer_fabric
```

Healthy output shows 4 CONNECTIONS across 2+ edge PoPs (e.g. sea, yvr).

## DNS record comment template (max 100 chars)

```
WebTransport QUIC/UDP 443 zone server. NIC en0 <full-mac>. DNS-only, no proxy.
```
