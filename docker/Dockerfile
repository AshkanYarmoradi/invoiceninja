# syntax=docker/dockerfile:1
#
# Huisscan patched Invoice Ninja image.
#
# Overlays a single one-line fix onto the official image so that a buyer who pays an
# invoice created with a ?redirect= query param is returned to their report instead
# of hitting an HTTP 500. See docker/README.md for the full rationale.
#
# Bump IN_VERSION to match the Invoice Ninja version Coolify runs. The build FAILS if
# the patch no longer applies to that version — that failure is the signal to review
# the change against upstream (and, once upstream fixes it, to retire this fork).
#
# The official image is Alpine-based, the app lives at /var/www/app, and app files are
# owned by uid:gid 1500:1500 (user "invoiceninja"). Verify these against the image
# before bumping IN_VERSION.
ARG IN_VERSION=5.13.26

# Stage 1 — patch the file in a throwaway Debian stage that has `patch` (the base image
# ships no patch utility). Applying a diff, rather than committing a whole vendored copy
# of the file, preserves drift detection: if the pinned upstream ever changes this code
# the patch fails, and so does the build.
FROM invoiceninja/invoiceninja:${IN_VERSION} AS source

FROM debian:bookworm-slim AS patcher
RUN apt-get update \
    && apt-get install -y --no-install-recommends patch \
    && rm -rf /var/lib/apt/lists/*
COPY --from=source \
    /var/www/app/app/Http/Controllers/ClientPortal/PaymentController.php \
    /work/PaymentController.php
COPY post-payment-redirect.patch /work/post-payment-redirect.patch
RUN set -eux; \
    cd /work; \
    patch PaymentController.php < post-payment-redirect.patch; \
    grep -q 'backup->redirect = null;' PaymentController.php

# Stage 2 — the official image, unchanged, with only the patched file overlaid.
FROM invoiceninja/invoiceninja:${IN_VERSION}
COPY --from=patcher --chown=1500:1500 \
    /work/PaymentController.php \
    /var/www/app/app/Http/Controllers/ClientPortal/PaymentController.php
