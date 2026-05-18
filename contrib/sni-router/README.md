# SNI-routing deployment for mtg

A turnkey `docker compose` setup that puts an SNI-aware TCP router
(HAProxy) in front of mtg **and** a real web server (Caddy with
automatic HTTPS).

## Why

Modern DPI systems actively probe suspected proxies.  If the server
closes the connection or returns something unexpected, the IP gets
flagged.  With this setup:

- **Telegram clients** connect to port 443, HAProxy sees the configured
  SNI and routes them to mtg (FakeTLS).
- **Everything else** (browsers, DPI probes, scanners) is routed to
  Caddy, which responds with a real Let's Encrypt certificate and serves
  genuine web content.

Because your domain's DNS points to this server, the SNI/IP match is
natural and passive DPI has nothing to flag.

## Quick start

```bash
# 1. Point your domain's DNS A/AAAA record to this server's IP.

# 2. Generate an mtg secret:
docker run --rm nineseconds/mtg:2 generate-secret --hex YOUR_DOMAIN

# 3. Configure:
#    - .env (or export)  →  DOMAIN=your.domain   # used by HAProxy + Caddy
#    - render mtg-config.toml from the tracked template
#      (the rendered file is gitignored — secret stays out of git):
export MTG_SECRET=...    # paste the hex secret from step 2
envsubst < mtg-config.toml.example > mtg-config.toml
#      (Or `cp mtg-config.toml.example mtg-config.toml` and edit ${MTG_SECRET}
#      by hand if you don't have envsubst.)

# 4. (Optional) put your site content into www/

# 5. Start:
docker compose up -d

# 6. Verify:
#    - Open https://YOUR_DOMAIN in a browser → you should see the web page
#    - Configure Telegram with the proxy link from:
docker compose exec mtg mtg access /config/config.toml
```

## Real client IPs (PROXY protocol)

HAProxy forwards TCP connections to mtg and Caddy with a PROXY protocol
v2 header so both backends see the real client IP instead of HAProxy's
container address.  Caddy also receives PROXY v2 from mtg on the
fronting path (see "Fronting loop" below), so all four pieces below
must stay in sync:

- `haproxy.cfg` — `send-proxy-v2` on the `mtg` and `web` backend `server` lines
- `mtg-config.toml` — `proxy-protocol-listener = true` (HAProxy → mtg)
- `mtg-config.toml` — `[domain-fronting].proxy-protocol = true` (mtg → Caddy on fronting)
- `Caddyfile` — `listener_wrappers { proxy_protocol { ... } tls }` on `:8443`

If you disable one, disable all four, otherwise the backend will fail
to parse the connection.

### Why HAProxy uses `network_mode: host`

When a container is on a bridge network and a port is published with
`ports: "443:443"`, the source IP of inbound connections is rewritten
to the bridge gateway before HAProxy sees it — Docker's `docker-proxy`
userland forwarder accepts on the host and re-opens the connection
from the gateway; Podman's `slirp4netns` / `pasta` does the same in
rootless mode.  The PROXY v2 header HAProxy then sends downstream
carries that gateway address (e.g. `172.x.x.1`), not the real client.

`network_mode: host` puts HAProxy in the host network namespace, so it
binds `:443` / `:80` directly with no NAT in the path and observes the
true source address of every connection.  mtg and Caddy stay on the
compose bridge and are published only on `127.0.0.1` — HAProxy reaches
them via host loopback, and the PROXY v2 header carries the real
client IP (v4 or v6) end-to-end.

Trade-off: HAProxy occupies the host's `:443` and `:80`.  Don't run
anything else on those ports on the same host.

## Fronting loop (why `[domain-fronting]` is set explicitly)

When mtg sees TLS that isn't valid Telegram (a probe or a browser
hitting the domain on `:443`), it forwards that connection to a real
web server — "domain fronting".  By default mtg uses the secret's
hostname as the fronting target and resolves it via DNS, which in
this setup points back to this server: the fronting dial lands on
HAProxy, SNI matches the secret, HAProxy routes the connection back
to mtg → loop.

The trigger is DNS, not name equality: any time the secret's hostname
resolves to this host, the loop reproduces.  In an SNI-router
deployment the secret's hostname has to point here for clients to
reach mtg in the first place, so the loop is the default state unless
mtg is steered away from HAProxy.

`mtg-config.toml` therefore pins the fronting target to the Caddy
container directly:

```toml
[domain-fronting]
host = "web"
port = 8443
proxy-protocol = true
```

`host = "web"` resolves through compose-network DNS to the `web`
service (Caddy), bypassing HAProxy.  `proxy-protocol = true` matches
Caddy's `:8443` listener wrapper so the real client IP still
propagates to Caddy's logs.

Requires mtg ≥ 2.4 — hostname acceptance for the fronting target was
added in #480.

## ACME (Let's Encrypt) notes

HAProxy passes `/.well-known/acme-challenge/` requests on `:80` to
Caddy so that HTTP-01 validation works out of the box.  Make sure your
domain's DNS A/AAAA record points to this server before starting.

## Architecture

```
              ┌──────────────────┐
 :443  ──────>│    HAProxy       │
              │  (TCP, SNI peek) │
              └──┬───────────┬───┘
    SNI match    │           │  default
                 v           v
           ┌─────────┐  ┌─────────┐
           │   mtg   │  │  Caddy  │
           │ :3128   │  │ :8443   │
           │ FakeTLS │  │ real TLS│
           └─────────┘  └─────────┘
```

## OpenWrt + podman-compose

OpenWrt's firewall zones are bound to interface *names*.  With bare
`podman` you pin the static `podman0` bridge into a zone and you're
done — but `podman-compose up` creates a project-scoped network, and
netavark spawns a *new* bridge for it (`podman1`, `podman2`, …) that
has no firewall rules, so containers lose outbound access.

Reuse the pre-configured `podman0` by adding to this compose file:

```yaml
networks:
  default:
    external: true
    name: podman
```

That tells compose to attach to the router-managed network instead of
spinning up a new one.  Background:
[discussion #513](https://github.com/9seconds/mtg/discussions/513)
and the [OpenWrt forum thread](https://forum.openwrt.org/t/podman-compose-dontt-have-network-access/250230).

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Service definitions |
| `haproxy.cfg` | SNI routing rules (reads `$DOMAIN` from the environment) |
| `mtg-config.toml.example` | mtg proxy config template — render with `envsubst` or copy + edit |
| `mtg-config.toml` | Rendered mtg proxy config (gitignored, contains your secret) |
| `Caddyfile` | Web server config (auto-HTTPS) |
| `www/` | Static site content served by Caddy |
