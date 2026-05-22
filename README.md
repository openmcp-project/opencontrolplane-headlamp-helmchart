# headlamp-deployment

Helm chart for deploying [Headlamp™](https://headlamp.dev) as the Kubernetes® UI for OpenMCP control planes.

Headlamp is **not exposed directly**. All browser traffic is proxied through the BFF (`ui-frontend`) at `/api/headlamp/*`, which injects authentication server-side. Headlamp runs as a ClusterIP service and is unreachable from outside the cluster.

## Architecture

```
Browser (iframe src="/api/headlamp/c/<alias>/flux/overview")
  │
  ▼
BFF (ui-frontend, /api/headlamp/*)
  ├─ injects Authorization: Bearer <mcp_accessToken>  (from encrypted session)
  └─ injects KUBECONFIG header (base64 kubeconfig)    (from encrypted session)
  │
  ▼
Headlamp pod (ClusterIP, not externally exposed)
  ├─ -base-url=/api/headlamp
  └─ -enable-dynamic-clusters  →  resolves cluster per-request from KUBECONFIG header
```

- No token or kubeconfig ever reaches the browser
- One Headlamp instance serves all control planes (multi-tenant via per-request kubeconfig)
- Headlamp's own OIDC is not used — the BFF handles authentication

## Prerequisites

- `helm` >= 3.10
- BFF (`ui-frontend`) deployed with `HEADLAMP_UPSTREAM_URL` pointing at the Headlamp service

## Setup

```bash
# Add the upstream headlamp Helm repo and fetch dependencies
helm repo add headlamp https://headlamp-k8s.github.io/headlamp/
helm repo update
helm dependency update
```

## Configuration

The only required configuration is setting `HEADLAMP_UPSTREAM_URL` in the BFF deployment:

```
HEADLAMP_UPSTREAM_URL=http://headlamp.<namespace>.svc.cluster.local
```

All chart values are in `values.yaml`. The key flags passed to Headlamp are:

| Flag | Purpose |
|------|---------|
| `-base-url=/api/headlamp` | Matches the BFF proxy prefix so assets and routes resolve correctly |
| `-enable-dynamic-clusters` | Accept cluster context from the `KUBECONFIG` header per-request |

## Install / upgrade

```bash
helm upgrade --install headlamp . \
  --namespace headlamp --create-namespace \
  -f values.yaml
```

## Feature toggle

The Headlamp tab in the MCP detail page is hidden by default. Enable it in `frontend-config.json`:

```json
{
  "featureToggles": {
    "enableHeadlamp": true
  }
}
```

## Plugins

Plugins are loaded from two sources:

### Remote plugins (pluginsManager)

Installed automatically at pod startup by the `pluginsManager` init container. Configured in `values.yaml` under `headlamp.pluginsManager.configContent`. The Flux plugin is enabled by default.

### Local plugins (ConfigMap)

The kiosk and crossplane plugins are not yet published to ArtifactHub or npm, so they are built locally and mounted into the pod as ConfigMaps. A build script is provided for each.

**Build and apply the kiosk plugin:**
```bash
./scripts/build-kiosk-plugin.sh
kubectl apply -f configmap-kiosk-plugin.yaml -n headlamp
```

**Build and apply the crossplane plugin:**
```bash
./scripts/build-crossplane-plugin.sh
kubectl apply -f configmap-crossplane-plugin.yaml -n headlamp
```

Each script clones the respective plugin repository, runs `npm ci && npm run build`, and writes a ready-to-apply ConfigMap manifest to the repo root. The generated `configmap-*-plugin.yaml` files are gitignored.

After applying new ConfigMaps, upgrade the Helm release so the volume mounts pick them up:
```bash
helm upgrade --install headlamp . --namespace headlamp -f values.yaml
```

> **TODO:** Once `headlamp-plugin-kiosk` and `headlamp-plugin-crossplane` are published to ArtifactHub or npm, replace the ConfigMap approach with `pluginsManager` entries in `values.yaml` (same pattern as `headlamp-flux`) and delete the `scripts/build-*-plugin.sh` scripts and their ConfigMap manifests.
