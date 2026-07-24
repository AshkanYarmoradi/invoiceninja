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
ARG IN_VERSION=5.13.26
FROM invoiceninja/invoiceninja:${IN_VERSION}

COPY post-payment-redirect.patch /tmp/post-payment-redirect.patch

# Apply the patch, then assert the fixed line is actually present — a fuzzy apply
# could otherwise land at the wrong offset and still exit 0. Any failure aborts the
# build (set -e).
RUN set -eux; \
    patch -p1 -d /var/www/html < /tmp/post-payment-redirect.patch; \
    grep -q 'backup->redirect = null;' \
        /var/www/html/app/Http/Controllers/ClientPortal/PaymentController.php; \
    rm /tmp/post-payment-redirect.patch
