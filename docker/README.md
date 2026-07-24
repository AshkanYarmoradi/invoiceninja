# Huisscan patched Invoice Ninja image

This fork exists for **one** reason and should carry **nothing else**: a single
one-line fix to the client-portal post-payment redirect, delivered as a Docker image
that Coolify pulls. If a second unrelated patch is ever wanted here, that is a signal
to reconsider self-hosting Invoice Ninja — not to grow the fork.

## What it fixes

Invoice Ninja lets you set a per-invoice post-payment redirect via a `?redirect=`
query parameter on invoice creation. After an anonymous payer pays, the client portal
is supposed to send them to that URL. Instead it returns **HTTP 500**:
`ClientPortal\PaymentController::show()` consumes the redirect with
`unset($backup->redirect)`, which leaves the **typed** property
`InvoiceBackup::$redirect` *uninitialized*; the immediately-following
`$invoice->saveQuietly()` re-reads it through the model cast and throws a fatal
`Error`. Because `$redirect` is declared `public ?string $redirect = null`, assigning
`null` instead of unsetting is type-safe and lets the save (and the redirect) succeed.

Huisscan uses this redirect to return a buyer to the specific property report they
just paid for — including agent-channel buyers who have no Huisscan tab open.

## How it's built

`docker/Dockerfile` starts `FROM invoiceninja/invoiceninja:${IN_VERSION}` — the
official image, unchanged — and applies `docker/post-payment-redirect.patch` to the
app at `/var/www/html`. Nothing is compiled or reinstalled; exactly one PHP file
changes. The build then greps for the fixed line so a fuzzy patch apply cannot pass
silently, and fails otherwise.

`.github/workflows/build-image.yml` builds this on every PR (without pushing, so the
PR check proves the patch still applies) and, on merge to the default branch, pushes
`ghcr.io/<owner>/invoiceninja:<IN_VERSION>` and `:latest` to GHCR.

## Deploying in Coolify

Point the Invoice Ninja **app** service at `ghcr.io/<owner>/invoiceninja:<IN_VERSION>`
instead of `invoiceninja/invoiceninja:<IN_VERSION>`. Everything else (nginx, database,
redis, environment) is unchanged — this image is the stock app image plus one file.
GHCR packages default to private; either make the package public or give Coolify a
pull token.

## Upgrading Invoice Ninja

1. Bump `ARG IN_VERSION` in `docker/Dockerfile` to the new version.
2. Open a PR. The PR's build check applies the patch against the new base image.
   - **Green** → the fix still applies; merge, then repoint Coolify at the new tag.
   - **Red (patch failed)** → upstream changed this code. Inspect
     `PaymentController::show()` at the new version and either regenerate the patch or
     retire the fork (see below).

## Retiring this fork

When upstream Invoice Ninja ships an equivalent fix, the patch will stop applying
(the `unset` line will be gone) and the build will go red. That red build is the
signal to **remove** `docker/` and this workflow and repoint Coolify back at the stock
`invoiceninja/invoiceninja` image — not to force the patch through.

Track the upstream fix via the issue linked at the top of
`docker/post-payment-redirect.patch`.
