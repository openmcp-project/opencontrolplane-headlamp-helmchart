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

Plugins are installed automatically at pod startup by the `pluginsManager` init container. Configured in `values.yaml` under `headlamp.pluginsManager.configContent`.

## Support, Feedback, Contributing

This project is open to feature requests/suggestions, bug reports etc. via [GitHub issues](https://github.com/openmcp-project/opencontrolplane-headlamp-helmchart/issues). Contribution and feedback are encouraged and always welcome. For more information about how to contribute, the project structure, as well as additional contribution information, see our [Contribution Guidelines](https://github.com/openmcp-project/.github/blob/main/CONTRIBUTING.md).

## Code of Conduct

We as members, contributors, and leaders pledge to make participation in our community a harassment-free experience for everyone. By participating in this project, you agree to abide by its [Code of Conduct](https://github.com/openmcp-project/.github/blob/main/CODE_OF_CONDUCT.md) at all times.

## Licensing

Copyright © Linux Foundation Europe. OpenControlPlane is a project of NeoNephos Foundation. For applicable policies including privacy policy, terms of use and trademark usage guidelines, please see https://linuxfoundation.eu. Linux is a registered trademark of Linus Torvalds.
Please see our [LICENSE](LICENSE) for copyright and license information. Detailed information including third-party components and their licensing/copyright information is available [via the REUSE tool](https://api.reuse.software/info/github.com/openmcp-project/opencontrolplane-headlamp-helmchart).

<p align="center"><img alt="NeoNephos foundation logo" src="https://raw.githubusercontent.com/neonephos/.github/refs/heads/main/assets/logo.svg" width="400"/></p>
