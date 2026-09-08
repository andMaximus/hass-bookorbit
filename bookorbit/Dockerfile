ARG BUILD_FROM
# hadolint ignore=DL3006
FROM ${BUILD_FROM}

# The upstream image already carries node, su-exec, tini, poppler-utils, ffmpeg and the
# kobo-cloudscraper python venv. It is node:26-alpine3.23 underneath, and Alpine 3.23 community
# ships postgresql18 plus a pgvector built against it — the same major upstream bundles via
# pgvector/pgvector:pg18. So the database goes in here rather than needing a second container.
ARG S6_OVERLAY_VERSION
ARG BASHIO_VERSION
ARG UPSTREAM_VERSION
ARG BUILD_ARCH

ENV \
    S6_BEHAVIOUR_IF_STAGE2_FAILS=2 \
    S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 \
    S6_SERVICES_GRACETIME=30000 \
    PGDATA=/data/postgres \
    PGHOST=/run/postgresql \
    ADDON_UPSTREAM_VERSION=${UPSTREAM_VERSION}

# hadolint ignore=DL3018
RUN set -eux; \
    apk add --no-cache \
        bash jq curl xz tzdata \
        postgresql18 postgresql18-client postgresql18-contrib postgresql-pgvector; \
    \
    # s6-overlay releases are named by uname -m, not by the HA arch slug.
    case "${BUILD_ARCH}" in \
        amd64)   S6_ARCH="x86_64"  ;; \
        aarch64) S6_ARCH="aarch64" ;; \
        *) echo "unsupported BUILD_ARCH: ${BUILD_ARCH}" >&2; exit 1 ;; \
    esac; \
    curl -fsSL -o /tmp/s6-noarch.tar.xz \
        "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz"; \
    curl -fsSL -o /tmp/s6-arch.tar.xz \
        "https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_ARCH}.tar.xz"; \
    tar -C / -Jxpf /tmp/s6-noarch.tar.xz; \
    tar -C / -Jxpf /tmp/s6-arch.tar.xz; \
    rm -f /tmp/s6-noarch.tar.xz /tmp/s6-arch.tar.xz; \
    \
    curl -fsSL -o /tmp/bashio.tar.gz \
        "https://github.com/hassio-addons/bashio/archive/v${BASHIO_VERSION}.tar.gz"; \
    mkdir -p /tmp/bashio; \
    tar -C /tmp/bashio -xzf /tmp/bashio.tar.gz --strip-components=1; \
    mv /tmp/bashio/lib /usr/lib/bashio; \
    ln -s /usr/lib/bashio/bashio /usr/bin/bashio; \
    rm -rf /tmp/bashio /tmp/bashio.tar.gz; \
    \
    # postgres needs a home for its socket dir and an initdb-capable user. The upstream image
    # already has a `node` user; `postgres` comes from the postgresql18 package.
    # s6-overlay v3 installs into /command. Home Assistant's own base images add this
    # symlink, and add-on scripts conventionally use the /usr/bin shebang, so keep it.
    ln -sf /command/with-contenv /usr/bin/with-contenv; \
    \
    mkdir -p /run/postgresql; \
    chown postgres:postgres /run/postgresql; \
    \
    # Fail the build rather than the first boot if the extension set is not actually installed.
    test -f /usr/lib/postgresql18/vector.so || \
        test -f "$(pg_config --pkglibdir)/vector.so"; \
    postgres --version

COPY rootfs /

RUN find /etc/s6-overlay/s6-rc.d -type f -name 'run' -exec chmod +x {} + \
    && find /etc/s6-overlay/s6-rc.d -type f -name 'up' -exec chmod +x {} + \
    && find /etc/s6-overlay/s6-rc.d -type f -name 'finish' -exec chmod +x {} +

# s6 is PID 1; upstream's tini entrypoint is replaced, not wrapped.
ENTRYPOINT ["/init"]

# Redeclared rather than inherited: this container also has to create a PostgreSQL
# cluster and run the schema migrations on a first start, which upstream's 20s
# start-period does not allow for. The Supervisor watches this instead of `watchdog:`.
HEALTHCHECK --interval=30s --timeout=5s --start-period=180s --retries=3 \
    CMD wget -q -T 4 -O /dev/null "http://127.0.0.1:${PORT:-3000}/api/v1/health" || exit 1

# hadolint ignore=DL3048
LABEL \
    io.hass.name="BookOrbit" \
    io.hass.description="Self-hosted library and reading platform with a bundled PostgreSQL 18 + pgvector database" \
    io.hass.type="addon" \
    org.opencontainers.image.source="https://github.com/andMaximus/hass-bookorbit"
