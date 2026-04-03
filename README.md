# ingress-nginx-build

Automated build pipeline for [chainguard-forks/ingress-nginx](https://github.com/chainguard-forks/ingress-nginx).

The official Kubernetes ingress-nginx controller reached EOL on April 3, 2026. Chainguard maintains a security-focused fork but does not publish container images or helm charts - only source code. This repo builds and publishes them.

## Architecture

```
chainguard-forks/ingress-nginx (source)
        | daily cron detects new controller-* tag
EtherShieldHQ/ingress-nginx-build (this repo, GitHub Actions)
        | make release (identical to upstream build process)
ghcr.io/ethershieldhq/ingress-nginx/controller (public image)
```

## Images

```
ghcr.io/ethershieldhq/ingress-nginx/controller:<tag>
ghcr.io/ethershieldhq/ingress-nginx/controller-chroot:<tag>
```

Multi-arch: `linux/amd64`, `linux/arm64`, `linux/arm/v7`, `linux/s390x`

Current version: **v1.15.2**

## How builds work

### Automated (daily)

- `check-upstream.yml` runs daily at 06:00 UTC
- Compares latest `controller-*` tag on `chainguard-forks/ingress-nginx` with `UPSTREAM_TAG` file
- If new tag found, triggers `build-push.yml`
- `helm-chart-*` tags are ignored, only `controller-*` tags are tracked

### Build process

- `build-push.yml` checks out the Chainguard fork at the specified tag
- Syncs `TAG` file with git tag version to ensure consistency (e.g., `controller-v1.15.2` -> `v1.15.2`)
- Runs `make release` with `REGISTRY=ghcr.io/ethershieldhq/ingress-nginx`
- This is the **exact same build process** as upstream - no custom Dockerfile or build args
- `make release` reads `NGINX_BASE` file for base image, `TAG` file for version
- Builds multi-arch and pushes `controller` + `controller-chroot` images to GHCR
- Updates `UPSTREAM_TAG` file in this repo
- Adds OCI annotations (description, source, vendor)

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
- **`make release` over custom build:** We use the exact upstream build process to guarantee identical output. Only `REGISTRY` env var is changed.
- **Public images:** Org-level GitHub setting changed to allow public packages. No imagePullSecret needed on clusters.
- **No helm chart fork:** We use the official `kubernetes.github.io/ingress-nginx` helm chart as-is, only overriding the image.

## Files

| File | Purpose |
|---|---|
| `UPSTREAM_TAG` | Tracks current upstream tag, updated by CI |
| `.github/workflows/check-upstream.yml` | Daily cron, detects new releases |
| `.github/workflows/build-push.yml` | Builds and pushes multi-arch images |

## CI authentication

GHCR uses `GITHUB_TOKEN` automatically - no additional secrets needed for pushing images.
