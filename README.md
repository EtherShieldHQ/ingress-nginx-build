# ingress-nginx-build

Automated build pipeline for [chainguard-forks/ingress-nginx](https://github.com/chainguard-forks/ingress-nginx).

The official Kubernetes ingress-nginx controller reached EOL on April 3, 2026. Chainguard maintains a security-focused fork but does not publish container images or helm charts - only source code. This repo builds and publishes them.

## Architecture

```
chainguard-forks/ingress-nginx (source)
        | daily cron detects new controller-* tag
EtherShieldHQ/ingress-nginx-build (this repo, GitHub Actions)
        | 1. build patched nginx BASE image (make -C images/nginx push)
        | 2. build controller FROM that base (make release BASE_IMAGE=...)
ghcr.io/ethershieldhq/ingress-nginx/{nginx,controller,controller-chroot} (public images)
```

## Images

```
ghcr.io/ethershieldhq/ingress-nginx/nginx:<base-tag>           # patched nginx base
ghcr.io/ethershieldhq/ingress-nginx/controller:<tag>
ghcr.io/ethershieldhq/ingress-nginx/controller-chroot:<tag>
```

`linux/amd64` only. Building arm/arm64/s390x would compile the nginx base under
QEMU emulation (hours, risk of job timeout); the controller image arch must match
the base arch.

Current version: see `UPSTREAM_TAG`.

## How builds work

### Automated (daily)

- `check-upstream.yml` runs daily at 06:00 UTC
- Compares latest `controller-*` tag on `chainguard-forks/ingress-nginx` with `UPSTREAM_TAG` file
- If new tag found, triggers `build-push.yml`
- `helm-chart-*` tags are ignored, only `controller-*` tags are tracked

### Build process

- `build-push.yml` checks out the Chainguard fork at the specified tag
- **Builds the patched nginx base image** from `images/nginx` (`make -C images/nginx push`),
  tagged from `images/nginx/TAG`. Skipped if that tag already exists in GHCR (the compile
  is expensive); `force_base_rebuild` input overrides. See "Why we build the nginx base".
- Syncs `TAG` file with git tag version to ensure consistency (e.g., `controller-v1.15.2` -> `v1.15.2`)
- Runs `make release` with `REGISTRY=ghcr.io/ethershieldhq/ingress-nginx` and
  `BASE_IMAGE=ghcr.io/ethershieldhq/ingress-nginx/nginx:<base-tag>` so the controller is
  built FROM the patched base instead of the frozen upstream base pinned in `NGINX_BASE`
- Builds `controller` + `controller-chroot` (amd64) and pushes to GHCR
- Updates `UPSTREAM_TAG` file in this repo
- Adds OCI annotations (description, source, vendor, base image ref)

### Why we build the nginx base

The fork's `NGINX_BASE` file pins the controller to the upstream base image
`registry.k8s.io/ingress-nginx/nginx`, which is frozen (the project reached EOL 2026-04-03)
and does **not** contain the fork's own nginx CVE backports (e.g. CVE-2026-9256 rewrite heap
overflow). Those patches live in `images/nginx/rootfs/patches/` and are only compiled in when
the nginx base image is built from source. A plain `make release` links the controller against
the unpatched upstream base, so the controller image ships vulnerable nginx despite tracking
the security fork. To actually get the patches we build the base ourselves, publish it, and
point the controller build at it via `BASE_IMAGE`. The base only rebuilds when the fork bumps
`images/nginx/TAG`.

### Tag format

- Upstream git tags: `controller-v1.15.2`
- Image tags: version extracted from git tag (`:v1.15.2`)
- Check [Chainguard fork tags](https://github.com/chainguard-forks/ingress-nginx/tags) for available `controller-*` releases

## Deploy with helm

The official [ingress-nginx helm chart](https://kubernetes.github.io/ingress-nginx) is used as-is. Only the image is overridden.

### Fresh install

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.image.registry=ghcr.io \
  --set controller.image.image=ethershieldhq/ingress-nginx/controller \
  --set controller.image.tag=v1.15.2 \
  --set controller.image.digest=""
```

### Migrate existing installation

```bash
helm upgrade ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --reuse-values \
  --set controller.image.registry=ghcr.io \
  --set controller.image.image=ethershieldhq/ingress-nginx/controller \
  --set controller.image.tag=v1.15.2 \
  --set controller.image.digest=""
```

- `--reuse-values` preserves all existing config (nodeport, replicas, etc.)
- `--set controller.image.digest=""` is required to override the hardcoded digest in the chart

## Manual build

```bash
# Trigger a build for a specific tag
gh workflow run build-push.yml -f tag=controller-v1.15.2 --repo EtherShieldHQ/ingress-nginx-build

# Monitor the build
gh run watch --repo EtherShieldHQ/ingress-nginx-build
```

## Key decisions

- **GHCR over Docker Hub:** Docker Hub doesn't support nested repo paths (`org/repo/image`). GHCR does, allowing `ethershieldhq/ingress-nginx/controller`.
- **Build the nginx base ourselves:** the only way to actually ship the fork's nginx CVE patches (see "Why we build the nginx base"). `BASE_IMAGE` override repoints the controller build off the frozen upstream base. Otherwise the build follows the exact upstream process.
- **amd64-only:** multi-arch would compile the nginx base under QEMU for hours; the controller arch must match the base arch.
- **Public images:** Org-level GitHub setting changed to allow public packages. No imagePullSecret needed on clusters.
- **No helm chart fork:** We use the official `kubernetes.github.io/ingress-nginx` helm chart as-is, only overriding the image.

## Files

| File | Purpose |
|---|---|
| `UPSTREAM_TAG` | Tracks current upstream tag, updated by CI |
| `.github/workflows/check-upstream.yml` | Daily cron, detects new releases |
| `.github/workflows/build-push.yml` | Builds the patched nginx base + controller (amd64) and pushes to GHCR |

## CI authentication

GHCR uses `GITHUB_TOKEN` automatically - no additional secrets needed for pushing images.
