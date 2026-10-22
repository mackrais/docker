# MackRais PHP security images

This fork builds the existing Pimcore 5.x Dockerfile with the exact PHP 8.5.12
base for CVE-2026-109618 and CVE-2026-109619. Advisory:
https://www.php.net/.
The PHP fix comes from `php:8.5.12-fpm-trixie`; no PHP source patch is maintained here.
If that base is unavailable, the build fails instead of falling back to an older PHP.

## Build and publish

1. Merge the fork workflow into this fork's default branch (`5.x`) and enable
   GitHub Actions in `mackrais/docker`. A workflow_dispatch workflow must exist
   on the default branch before GitHub exposes its manual trigger.
2. Run **MackRais PHP security images** with `publish=false` to test the build.
3. Run with `publish=true` to build, verify and publish both variants to GHCR.
   The workflow uses `GITHUB_TOKEN` with `packages: write`; no Docker Hub secrets
   or Pimcore credentials are needed. For an existing GHCR package, grant this
   repository Actions write access to that package.
4. Make the `mackrais/pimcore` package public for anonymous VPS pulls, or configure
   read-only GHCR authentication on the VPS for a private package.

Each variant is built and tested natively on AMD64 and ARM64. Checks cover the
exact CLI and FPM PHP version, required extensions, FPM configuration, Composer,
and the supervisord executable. Multi-platform deployment tags are published
only after all four build jobs succeed:

```text
ghcr.io/mackrais/pimcore:php8.5.12-max-5.x
ghcr.io/mackrais/pimcore:php8.5.12-supervisord-5.x
```

These tags can change on a subsequent successful run. Record the manifest digests
from the run summary and pin them for reproducible production deployments.
Intermediate architecture tags include the run ID and attempt to avoid mixing builds.
These images are not Copa-hardened and the checks do not certify absence of all CVEs.

## Sonechko deployment

After publication, change the `php` and `supervisord` image references in the
site's `docker-compose-prod.yaml` to the two tags above (preferably with digests).
Keep the previous image references/digests for rollback. Commit and deliver that
Compose change through the site's normal deployment process; the deployment
script updates its checkout from upstream, so do not rely on uncommitted VPS edits.

Before restarting services, pull and verify both downloaded images from the site
directory on the VPS:

```bash
docker compose --env-file .env.local -f docker-compose-prod.yaml pull php supervisord
docker compose --env-file .env.local -f docker-compose-prod.yaml run --rm --no-deps -T php php-fpm -v
docker compose --env-file .env.local -f docker-compose-prod.yaml run --rm --no-deps -T supervisord php -v
```

Both must report PHP 8.5.12. Use the site's maintenance/deploy process to recreate
both services, then repeat the version checks with `exec -T` and check the site
through its origin before disabling maintenance. Also run the site's Composer
platform requirements check and application smoke checks against these images.

## Upstream contribution

The upstream release workflow is restricted to `pimcore/docker` and publishes
under Pimcore's registry namespace; this fork workflow intentionally stands alone.
Do not propose MackRais registry settings as an upstream security fix. The
upstream Dockerfile already accepts `PHP_VERSION`; upstream may only need to
rebuild/publish. A separate PR can improve fresh-base pulls or version checks
after discussing the release requirements with maintainers.
