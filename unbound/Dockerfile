# syntax=docker/dockerfile:1
#
# Unbound - recursive, DNSSEC-validating resolver for Home Assistant OS.
#
# Pinned Home Assistant base image (Alpine 3.24, s6-overlay, bashio 0.19.0).
# The digest pins the multi-arch index; update tag AND digest together.
ARG BASE_IMAGE=ghcr.io/home-assistant/base:3.24-2026.10.3@sha256:880016115f356afc814398e17ba68a7ee66dfc1b4eaef09755ea2c9fa18ea481

###############################################################################
# Stage 1: build Unbound from source
###############################################################################
FROM ${BASE_IMAGE} AS builder

SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

# Source: official NLnet Labs repository, pinned to the commit of the
# release-1.26.2 tag. The commit hash pins the exact source tree; the build
# aborts if the fetched commit or the version in ./configure differs.
# NOTE: the release tarball's SHA256/PGP signature is not verified here
# (decision of the repository owner, see DOCS.md "Supply chain").
ARG UNBOUND_VERSION=1.26.2
ARG UNBOUND_COMMIT=f0cd7067d950921427c91ade4ecf060e944f88eb
ARG UNBOUND_REPO=https://github.com/NLnetLabs/unbound.git
# Keep RAM usage low when the Supervisor builds on a Raspberry Pi.
ARG MAKE_JOBS=2

# Root trust anchors (DS of KSK-2017 and KSK-2024) as shipped in the pinned
# source (smallapp/unbound-anchor.c). Cross-checked below.
ARG ROOT_DS_20326=E06D44B80B8F1D39A95C0B0D7C65D08458E880409BBC683457104237C7F8EC8D
ARG ROOT_DS_38696=683D2D0ACB8C9B712A1948B27F741219298D0A450D612C483AF444A4C0FB2B16

RUN apk add --no-cache \
        bison \
        build-base \
        expat-dev \
        flex \
        git \
        libevent-dev \
        openssl-dev

WORKDIR /src/unbound

RUN git init -q . \
    && git remote add origin "${UNBOUND_REPO}" \
    && git fetch -q --depth 1 origin "${UNBOUND_COMMIT}" \
    && git checkout -q FETCH_HEAD \
    && test "$(git rev-parse HEAD)" = "${UNBOUND_COMMIT}" \
    && grep -qx "PACKAGE_VERSION='${UNBOUND_VERSION}'" configure

# Only what a validating recursive resolver needs. Everything else
# (python, dnscrypt, dnstap, cachedb, subnet, systemd) stays disabled.
RUN ./configure \
        --prefix=/usr \
        --sysconfdir=/etc \
        --localstatedir=/var \
        --with-username=unbound \
        --with-ssl=/usr \
        --with-libevent=/usr \
        --with-libexpat=/usr \
        --without-pythonmodule \
        --without-pyunbound \
        --disable-static \
        --disable-rpath \
        --enable-pie \
        --enable-relro-now \
    && make -j"${MAKE_JOBS}" \
    && make install DESTDIR=/out \
    && rm -f /out/usr/sbin/unbound-control-setup \
    && cd /out/usr/sbin \
    && strip unbound unbound-anchor unbound-checkconf unbound-control unbound-host \
    && find /out/usr/lib -name 'libunbound.so.*' -type f -exec strip {} + \
    && rm -rf /out/usr/include /out/usr/lib/pkgconfig /out/usr/lib/*.la \
        /out/usr/share/man /out/etc/unbound

# Seed trust anchor file (DS format, tracked afterwards via RFC 5011).
RUN mkdir -p /out/usr/share/unbound \
    && grep -oE '\. IN DS [0-9]+ 8 2 [0-9A-F]{64}' smallapp/unbound-anchor.c \
        > /out/usr/share/unbound/root.key \
    && grep -qx ". IN DS 20326 8 2 ${ROOT_DS_20326}" /out/usr/share/unbound/root.key \
    && grep -qx ". IN DS 38696 8 2 ${ROOT_DS_38696}" /out/usr/share/unbound/root.key \
    && test "$(wc -l < /out/usr/share/unbound/root.key)" -eq 2 \
    && cp LICENSE /out/usr/share/unbound/LICENSE

###############################################################################
# Stage 2: runtime
###############################################################################
FROM ${BASE_IMAGE}

SHELL ["/bin/ash", "-eo", "pipefail", "-c"]

# Runtime libraries only (+ dig for diagnostics). curl is part of the base.
RUN apk add --no-cache \
        bind-tools \
        ca-certificates \
        expat \
        libevent \
        libssl3 \
        libcrypto3 \
    && addgroup -S unbound \
    && adduser -S -D -H -h /var/empty -s /sbin/nologin -G unbound unbound

COPY --from=builder /out/ /
COPY run.sh /run.sh
RUN chmod 0755 /run.sh \
    && /usr/sbin/unbound -V

# Plain TCP connect to the local port; no name resolution involved.
HEALTHCHECK --interval=60s --timeout=5s --start-period=30s --retries=3 \
    CMD bash -c '</dev/tcp/127.0.0.1/5335' || exit 1

CMD ["/run.sh"]

ARG BUILD_VERSION=dev
ARG BUILD_ARCH
LABEL \
    io.hass.name="Unbound (recursive)" \
    io.hass.description="Recursive, DNSSEC-validating DNS resolver" \
    io.hass.version="${BUILD_VERSION}" \
    io.hass.type="app" \
    io.hass.arch="${BUILD_ARCH}" \
    org.opencontainers.image.title="Unbound (recursive)" \
    org.opencontainers.image.licenses="BSD-3-Clause"
