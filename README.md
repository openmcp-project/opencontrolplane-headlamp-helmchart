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
