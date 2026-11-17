# syntax=docker/dockerfile:1
# ---------------------------------------------------------------------------
# NPM + ModSecurity overlay image (Option A: thin overlay on the official image)
#
# Build:  docker build -f build/Dockerfile -t npm-modsec:2.17.0-modsec3.0.17-crs4.30.0 .
#         (build context = repo root; needs modsec/ tree, see below)
#
# Validated by spike 001-npm-modsec:
#   * official NPM 2.17.0 ships openresty/1.31.8.1 built with --with-compat
#   * configure args are extracted FROM THE BINARY, not from docs
#   * libmodsecurity -j must be capped: seclang parser TUs take ~2 GB each
#   * geoip2 must be cloned to /tmp/openresty/... because that absolute path
#     appears verbatim in the base binary's configure arguments
# ---------------------------------------------------------------------------
ARG NPM_VERSION=2.17.0
ARG OPENRESTY_VERSION=1.31.8.1
ARG MODSEC_VERSION=3.0.17
ARG MODSEC_NGINX_VERSION=1.0.4
ARG CRS_VERSION=v4.30.0
ARG VTS_VERSION=v0.2.7
ARG CACHE_PURGE_VERSION=3.0.4
ARG BUILD_JOBS=6

# --- stage 1: capture exact configure args from the official binary --------
# This is the ABI guardrail: if upstream bumps openresty, the pin check fails
# the build loudly instead of shipping an incompatible module.
FROM jc21/nginx-proxy-manager:${NPM_VERSION} AS args
ARG OPENRESTY_VERSION
RUN set -eu; \
    nginx -V 2>&1 | sed -n 's/^nginx version: openresty\///p' > /tmp/version; \
    grep -qx "${OPENRESTY_VERSION}" /tmp/version \
      || { echo "FATAL: base image has openresty $(cat /tmp/version), pinned ${OPENRESTY_VERSION} — update pins"; exit 1; }; \
    nginx -V 2>&1 | sed -n 's/^configure arguments: //p' > /tmp/args

# --- stage 2: build libmodsecurity + connector + fetch CRS ------------------
FROM debian:trixie-slim AS builder
ARG MODSEC_VERSION MODSEC_NGINX_VERSION OPENRESTY_VERSION CRS_VERSION VTS_VERSION CACHE_PURGE_VERSION BUILD_JOBS
RUN set -eu; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      build-essential g++ gcc make git wget ca-certificates xz-utils \
      flex bison pkg-config autoconf automake libtool perl rsync \
      libpcre2-dev libyajl-dev libxml2-dev libcurl4-openssl-dev liblmdb-dev \
      libssl-dev zlib1g-dev libmaxminddb-dev; \
    rm -rf /var/lib/apt/lists/*

RUN set -eu; \
    git clone --depth 1 --branch v${MODSEC_VERSION} --recurse-submodules --shallow-submodules \
      https://github.com/owasp-modsecurity/ModSecurity /tmp/ModSecurity; \
    cd /tmp/ModSecurity; \
    ./build.sh; \
    ./configure --prefix=/usr/local/modsecurity; \
    make -j${BUILD_JOBS}; \
    make install; \
    mkdir -p /usr/local/modsecurity/etc; \
    cp modsecurity.conf-recommended /usr/local/modsecurity/etc/modsecurity.conf; \
    cp unicode.mapping /usr/local/modsecurity/etc/unicode.mapping

RUN set -eu; \
    git clone --depth 1 --branch v${MODSEC_NGINX_VERSION} \
      https://github.com/owasp-modsecurity/ModSecurity-nginx /tmp/ModSecurity-nginx; \
    git clone --depth 1 --branch ${CRS_VERSION} \
      https://github.com/coreruleset/coreruleset /tmp/crs; \
    cp /tmp/crs/crs-setup.conf.example /tmp/crs/crs-setup.conf; \
    git clone --depth 1 --branch ${VTS_VERSION} \
      https://github.com/vozlt/nginx-module-vts /tmp/nginx-module-vts; \
    git clone --depth 1 --branch ${CACHE_PURGE_VERSION} \
      https://github.com/nginx-modules/ngx_cache_purge /tmp/ngx_cache_purge

# openresty source at the tag the base binary was built from; geoip2 at the
# literal path baked into the base's configure args
RUN set -eu; \
    cd /tmp; \
    wget -q https://openresty.org/download/openresty-${OPENRESTY_VERSION}.tar.gz; \
    tar xzf openresty-${OPENRESTY_VERSION}.tar.gz; \
    mkdir -p /tmp/openresty; \
    git clone --depth 1 https://github.com/leev/ngx_http_geoip2_module \
      /tmp/openresty/ngx_http_geoip2_module

COPY --from=args /tmp/args /tmp/nginx-args
# NOTE (spike finding): the -V args are NGINX-level args. Never feed them to the
# openresty bundle wrapper (it re-mangles --prefix and resolves the relative
# --add-module=../... paths wrong). Two-step: 1) wrapper configure as plain prep
# (extracts bundle into build/, builds LuaJIT), 2) nginx's own configure inside
# build/nginx-*/ with the verbatim args + our connector, then `make modules`.
RUN set -eu; \
    cd /tmp/openresty-${OPENRESTY_VERSION}; \
    ./configure > /tmp/or-prep.log 2>&1; \
    NGX="$(ls -d build/nginx-*/)"; \
    LJLIB="$(dirname "$(find "$PWD/build/luajit-root" -name 'libluajit-5.1.a' -print -quit)")"; \
    LJINC="$(dirname "$(find "$PWD/build/luajit-root" -name 'lauxlib.h' -path '*include*' -print -quit)")"; \
    export LUAJIT_LIB="$LJLIB" LUAJIT_INC="$LJINC" \
           MODSECURITY_INC=/usr/local/modsecurity/include \
           MODSECURITY_LIB=/usr/local/modsecurity/lib; \
    cd "$NGX"; \
    ./configure $(cat /tmp/nginx-args) \
      --add-dynamic-module=/tmp/ModSecurity-nginx \
      --add-dynamic-module=/tmp/nginx-module-vts \
      --add-dynamic-module=/tmp/ngx_cache_purge; \
    make modules; \
    mkdir -p /tmp/out; \
    cp objs/ngx_http_modsecurity_module.so \
       objs/ngx_http_vhost_traffic_status_module.so \
       objs/ngx_http_cache_purge_module.so /tmp/out/

# --- stage 3: overlay onto the pristine official image ---------------------
FROM jc21/nginx-proxy-manager:${NPM_VERSION}
ARG CRS_VERSION

# runtime deps of libmodsecurity (verify with ldd after build; trixie names)
RUN set -eu; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      libyajl2 libxml2 libcurl4t64 liblmdb0 libpcre2-8-0; \
    rm -rf /var/lib/apt/lists/*

# slim copy: runtime objects only (skip 222 MB static lib, headers, pkgconfig)
# NOTE: the .so.3 SONAME symlink is created in the final RUN — BuildKit COPY of a
# symlink to an explicit path dereferences it (duplicate 58 MB regular file).
COPY --from=builder /usr/local/modsecurity/lib/libmodsecurity.so.3.0.17 /usr/local/modsecurity/lib/libmodsecurity.so.3.0.17
COPY --from=builder /usr/local/modsecurity/etc /usr/local/modsecurity/etc
COPY --from=builder /tmp/out/ngx_http_modsecurity_module.so /usr/lib/nginx/modules/
COPY --from=builder /tmp/out/ngx_http_vhost_traffic_status_module.so /usr/lib/nginx/modules/
COPY --from=builder /tmp/out/ngx_http_cache_purge_module.so /usr/lib/nginx/modules/
COPY --from=builder /tmp/crs /usr/local/modsecurity/crs

# engine config: paths here match the /data/modsec layout seeded on first run
COPY modsec/ /etc/nginx/modsec/

# s6 oneshot that seeds /data/modsec tunables on first run.
# Boot-log finding: s6-rc v3 oneshot `up` files are EXECLINE, not shell —
# a shell script there exits 127. Shell logic is in scripts/modsec-seed.sh;
# `up` is the single execline line that invokes it.
COPY rootfs-s6/ /
RUN set -eu; \
    printf 'oneshot\n' > /etc/s6-overlay/s6-rc.d/modsec-seed/type; \
    chmod +x /etc/s6-overlay/scripts/modsec-seed.sh; \
    touch /etc/s6-overlay/s6-rc.d/user/contents.d/modsec-seed

# Build-time nginx -t needs the runtime dirs (see spike findings) and an `npm`
# user for the stock `user npm;` directive.
# Boot-log finding: NEVER ship the npm user — the stock boot's usermod branch
# has no -o, and with the default PUID=0 any pre-existing npm user dies with
# "UID '0' already exists", killing the prepare bundle and the whole boot.
# So the user+group exist only WITHIN this RUN: the stock nginx.conf user
# directive references BOTH getpwnam("npm") and getgrnam("npm"), so useradd
# must create the group too (no -N!); both are removed again after -t.
RUN set -eu; \
    ln -sf libmodsecurity.so.3.0.17 /usr/local/modsecurity/lib/libmodsecurity.so.3; \
    echo '/usr/local/modsecurity/lib' > /etc/ld.so.conf.d/modsecurity.conf; \
    ldconfig; \
    mkdir -p /etc/nginx/modules; \
    printf 'load_module /usr/lib/nginx/modules/ngx_http_modsecurity_module.so;\n' \
      > /etc/nginx/modules/modsecurity.conf; \
    printf 'load_module /usr/lib/nginx/modules/ngx_http_vhost_traffic_status_module.so;\n' \
      > /etc/nginx/modules/vts.conf; \
    printf 'load_module /usr/lib/nginx/modules/ngx_http_cache_purge_module.so;\n' \
      > /etc/nginx/modules/cache-purge.conf; \
    useradd -r -M npm; \
    mkdir -p /tmp/nginx /run/nginx /var/log/nginx /var/cache/nginx \
             /var/lib/nginx/cache/public /var/lib/nginx/cache/private \
             /data/logs; \
    nginx -t; \
    userdel npm; \
    groupdel npm 2>/dev/null || true

# NOTE: /etc/nginx/modules/*.conf is already included by NPM's stock nginx.conf
# (main context) — no other stock file is modified. WAF is OFF by default:
# enable per host in the UI Advanced tab:
#   modsecurity on;
#   modsecurity_rules_file /etc/nginx/modsec/main.conf;
# or globally via /data/nginx/custom/http_top.conf.
