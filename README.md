# npm-modsec — Nginx Proxy Manager + ModSecurity overlay image

> **⛔ SUPERSEDED (Nov 2026, Phase A2b) — do not build from this repo.**
> The WAF layer lives in [funkysys-org/nginx-full](https://github.com/funkysys-org/nginx-full)
> now: one pipeline (parity-gated openresty base + NPM app assembly + the
> WAF layer), one pin set, one drift-check. The last image published here —
> already built FROM our own parity-gated base (Phase A2) — was
> `ghcr.io/funkysys-org/npm-modsec:2.17.0-modsec3.0.17-crs4.30.0-vts0.2.7-purge3.0.4`.
> Successor image:
> `ghcr.io/funkysys-org/nginx-full:2.17.0-or1.31.8.1-modsec3.0.17-crs4.30.0-vts0.2.7-purge3.0.4`.
> The product consuming it: [funkysys-org/proxywarden](https://github.com/funkysys-org/proxywarden).
> The repo stays as the historical reference for the overlay mechanics
> (builder stages, s6 seed, payload-matrix smoke).

Historical note — the original mission: thin overlay on the official
`jc21/nginx-proxy-manager` image: libmodsecurity v3 +
`ngx_http_modsecurity_module.so` + OWASP CRS, loaded via NPM's own
`/etc/nginx/modules/*.conf` include. No stock file was modified; the WAF is
**off by default** and enabled per proxy host from the UI.

Validated by spike `001-npm-modsec` (see ../README.md).

## Build

```bash
cd phase1
docker build -f build/Dockerfile \
  -t npm-modsec:2.17.0-modsec3.0.17-crs4.30.0-vts0.2.7-purge3.0.4 .
```

Pins live in the Dockerfile ARGs. The build **fails loudly** if the official
image's openresty no longer matches the pinned `OPENRESTY_VERSION` — that is
the ABI guardrail: never ship a module built against a different nginx.

## Test before deploy

```bash
NPM_MODSEC_IMAGE=npm-modsec:2.17.0-modsec3.0.17-crs4.30.0-vts0.2.7-purge3.0.4 \
  docker compose -f docker-compose.test.yml up -d
bash scripts/smoke-test.sh                      # self-diagnosing; expects SMOKE OK
docker compose -f docker-compose.test.yml down -v   # -v wipes test data
```

The stack boots the real image (admin UI on :8181, WAF test instance on :8080,
stub backend) and the script verifies: UI up, `nginx -t` with the module
loaded at a real boot, `/data/modsec` seeded by the s6 oneshot, benign request
200, SQLi/RCE/XSS payloads 403, and `949110` (Inbound Anomaly Score) present
in the audit log. On any failure it prints `compose ps` + the last 40
container log lines.

## Layout inside the image

| Path | Content |
|---|---|
| `/usr/lib/nginx/modules/ngx_http_modsecurity_module.so` | connector module |
| `/usr/local/modsecurity/` | libmodsecurity + headers + recommended conf |
| `/usr/local/modsecurity/crs/` | CRS, version-pinned (rules/ + crs-setup.conf) |
| `/etc/nginx/modsec/` | `main.conf` (include chain) + engine `modsecurity.conf` |
| `/etc/nginx/modules/modsecurity.conf` | `load_module` line (NPM already includes this dir) |
| `/usr/lib/nginx/modules/ngx_http_vhost_traffic_status_module.so` | **vts** — per-host traffic stats; loaded, inactive until configured |
| `/usr/lib/nginx/modules/ngx_http_cache_purge_module.so` | **cache purge** — invalidation for `Cache Assets`; loaded, inactive until configured |
| `/etc/nginx/modules/vts.conf` + `cache-purge.conf` | their `load_module` lines |
| `/data/modsec/` | **tunables, seeded on first run** (survives rebuilds): `crs-setup.conf`, `rules-before.conf`, `rules-after.conf` |

## Enable the WAF

Per proxy host (recommended), in the host's **Advanced** tab:

```nginx
modsecurity on;
modsecurity_rules_file /etc/nginx/modsec/main.conf;
```

That host now **logs but does not block** (engine default is DetectionOnly).
To make the host actually block, point it at the blocking chain instead:

```nginx
modsecurity on;
modsecurity_rules_file /etc/nginx/modsec/main-block.conf;
```

`main-block.conf` is the same include chain with `SecRuleEngine On` as its
last directive. Do NOT try `modsecurity_rules 'SecRuleEngine On';` after the
rules_file — verified empirically: it does not override the file's
DetectionOnly (rules fire, requests still pass).

Globally (all HTTP hosts): same two directives in `/data/nginx/custom/http_top.conf`.
Stream (TCP) hosts are NOT covered — ModSecurity v3 is HTTP-only.

## Optional proxy modules (vts, cache purge)

Both are **loaded but inactive** — no directive is set until you configure them.

### vts — per-host traffic visibility (the tuning feedback loop)

`/data/nginx/custom/http_top.conf`:

```nginx
vhost_traffic_status_zone;
```

Then expose the status JSON somewhere protected (e.g. its own proxy host
behind an access list):

```nginx
location /vts_status {
    vhost_traffic_status_display;
    vhost_traffic_status_display_format json;
    # protect with an access list / allow-list!
}
```

Per server zone you get request rates, status-code distribution, upstream
response times, cache hit/miss and bandwidth — the data you tune
`limit_req` / `proxy_cache_valid` with instead of guessing.

### cache purge — invalidation for "Cache Assets"

On hosts with Cache Assets enabled, add to the Advanced tab:

```nginx
proxy_cache_purge PURGE from 127.0.0.1;
```

Then invalidate a cached path:

```bash
curl -X PURGE https://host.example.com/path/to/asset
```

## Update cadence

1. NPM release → bump `NPM_VERSION`; if openresty changed, bump `OPENRESTY_VERSION` too (the args stage re-derives everything from the binary).
2. CRS release → bump `CRS_VERSION`, rebuild, re-run smoke tests.
3. Never edit CRS files in place — fork `crs-setup.conf` / exclusion files under `/data/modsec/`.
4. Module pins (vts / cache purge) bump like any other: workflow env AND Dockerfile ARGs — `pins-sync` enforces the pair — then rebuild. The image tag carries them: `…-vts<vts>-purge<purge>`.

## Rollback

The image changes nothing global — swap the tag back to the official
`jc21/nginx-proxy-manager:<version>` and remove any `modsecurity` directives
from Advanced tabs (or leave them: unknown directives would fail a reload, so
remove them in the same window).
