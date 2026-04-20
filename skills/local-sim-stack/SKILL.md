---
name: local-sim-stack
description: Start, verify, and tear down the local multiplayer-fabric simulation stack (CockroachDB, versitygw, Cloudflare Tunnel, zone-backend, zone-server). Use when developing or testing the full stack on a single machine.
license: MIT
metadata:
  author: V-Sekai-fire
---

# SOP: Local simulation stack

Runs the full multiplayer-fabric backend on one machine and exposes it globally
via Cloudflare Tunnel (HTTP/HTTPS/HTTP3) and direct UDP (WebTransport).

## Prerequisites

| Requirement | Check |
|---|---|
| Docker with Compose V2 | `docker compose version` |
| cloudflared authenticated | `cloudflared tunnel list` shows `multiplayer_fabric` |
| UDP 443 port-forwarded on router | `nc -zvu <public-ip> 443` succeeds |
| DNS `zone-700a.chibifire.com → <public-ip>` (grey cloud) | `dig +short zone-700a.chibifire.com` |
| `.env` file present | `cp .env.example .env && fill in CLOUDFLARE_TUNNEL_TOKEN` |

## Start

```sh
cd multiplayer-fabric-hosting
docker compose up -d
```

Bring-up order is enforced by `depends_on`:
1. `crdb` (CockroachDB) — waits for SQL healthcheck
2. `versitygw` (S3 gateway) — waits for HTTP healthcheck
3. `versitygw-init` (creates `uro-uploads` bucket) — runs once
4. `zone-backend` (Phoenix/Bandit) — waits for crdb + versitygw-init
5. `cloudflared` (Cloudflare Tunnel) — proxies zone-backend to the internet
6. `zone-server` (Godot WebTransport) — direct UDP 443

## Verify

```sh
# CockroachDB admin UI
open http://localhost:8080

# zone-backend API (local)
curl http://localhost:4000/api/v1/

# zone-backend API (via Cloudflare Tunnel — confirm HTTP/3 at edge)
curl -v https://hub.vsekai.cloud/api/v1/

# WebTransport reachability (UDP 443)
nc -zvu zone-700a.chibifire.com 443

# Cloudflare Tunnel status
cloudflared tunnel info multiplayer_fabric
```

## Logs

```sh
docker compose logs -f zone-backend
docker compose logs -f cloudflared
docker compose logs -f zone-server
```

## Stop

```sh
docker compose down
```

Volumes (`crdbdata`, `versitygwdata`) persist across restarts. To wipe all
state:

```sh
docker compose down -v
```

## Adding a second zone machine

1. Port-forward UDP 443 on the new machine's router.
2. Get its public IP: `curl -4 ifconfig.me`
3. Get its NIC MAC last 4: `ifconfig en0 | awk '/ether/{print $2}' | tr -d ':' | tail -c 5`
4. Add DNS A record `zone-<last4>.chibifire.com → <public-ip>` (grey cloud, DNS-only).
5. Copy `.env` from the first machine; set `ZONE_HOST=zone-<last4>.chibifire.com`.
6. Run `docker compose up -d` — each machine is an independent zone node.

## Cloudflare Tunnel token

The tunnel token is bound to the `multiplayer_fabric` tunnel in Cloudflare Zero Trust.
Retrieve it any time:

```sh
cloudflared tunnel token multiplayer_fabric
```

Routing (public hostname → `http://zone-backend:4000`) is configured in the
Cloudflare dashboard under Networks → Tunnels → `multiplayer_fabric` → Public Hostnames.

## S3 credentials

versitygw uses static credentials (`minioadmin` / `minioadmin`) for local dev only.
The `uro-uploads` bucket is created automatically by `versitygw-init` on first start.
zone-backend reads credentials from `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in `.env`.

## WebTransport cert hash

The zone server prints its self-signed cert hash at startup:

```sh
docker compose logs zone-server | grep cert_hash
```

Set `ZONE_CERT_HASH_B64` in `.env` and distribute to clients for pinning.
