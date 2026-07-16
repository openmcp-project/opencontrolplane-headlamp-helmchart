# headlamp-deployment

Helm chart for deploying [Headlamp™](https://headlamp.dev) as the Kubernetes® UI for OpenControlPlane (OCP) control planes.

This is a thin wrapper chart around the upstream [headlamp](https://kubernetes-sigs.github.io/headlamp/) chart. It pins the configuration needed for the OCP BFF proxy pattern and bundles three Headlamp plugins. Chart and Headlamp versions are tracked in `Chart.yaml`.

## What the chart deploys

Installing this chart creates the following Kubernetes resources:

| Resource | Name | Notes |
|----------|------|-------|
| Deployment | `headlamp` | Headlamp container + `pluginsManager` sidecar |
| Service | `headlamp` | ClusterIP only — not exposed externally |
| ServiceAccount | `headlamp` | Used by the Headlamp pod |
| ClusterRoleBinding | `headlamp` | Binds the ServiceAccount to `cluster-admin` |
| ConfigMap | `headlamp-plugin-config` | Plugin list consumed by the `pluginsManager` sidecar |

No Ingress or HTTPRoute is created. Headlamp is only reachable from within the cluster via the BFF.

### Bundled plugins

| Plugin | Purpose |
|--------|---------|
| [headlamp-flux](https://artifacthub.io/packages/headlamp/headlamp-plugins/headlamp_flux) | Flux GitOps views (default landing view for an MCP) |
| [headlamp-ocp](https://artifacthub.io/packages/headlamp/opencontrolplane-headlamp-plugin/opencontrolplane) | OCP-specific UI extensions |
| [headlamp-crossplane](https://artifacthub.io/packages/headlamp/crossplane-headlamp-plugin/headlamp_crossplane) | Crossplane resource views |

To add or update a plugin, edit `headlamp.pluginsManager.configContent` in `values.yaml`.

## Key Headlamp flags set by this chart

| Flag | Value | Purpose |
|------|-------|---------|
| `-base-url` | `/api/headlamp` | Matches the BFF proxy prefix so assets and routes resolve correctly |
| `-enable-dynamic-clusters` | — | Accept cluster context from the `KUBECONFIG` header per-request |
| `-user-plugins-dir` | `/headlamp/user-plugins` | Secondary plugin directory |
| `-watch-plugins-changes` | `true` | Hot-reload plugins without restarting the pod |
| `-in-cluster` | — | Headlamp reads its own service account for in-cluster API access |

## Architecture

```
Browser (iframe src="/api/headlamp/c/<alias>/flux/overview")
  │
  ▼
BFF (ui-frontend, /api/headlamp/*)
  ├─ injects Authorization: Bearer <mcp_accessToken>  (from encrypted session)
  └─ forwards KUBECONFIG header (base64 kubeconfig)   (from encrypted session)
  │
  ▼
Headlamp pod (ClusterIP, not externally reachable)
  ├─ -base-url=/api/headlamp
  └─ -enable-dynamic-clusters  →  resolves cluster per-request from KUBECONFIG header
```

- No token or kubeconfig ever reaches the browser.
- One Headlamp instance serves all control planes (multi-tenant via per-request kubeconfig).
- Headlamp's own OIDC is not configured — the BFF handles all authentication.

### Multi-tenancy

Each MCP gets a unique cluster alias (`project--workspace--name`). The frontend calls Headlamp's `parseKubeConfig` endpoint to register the cluster in the browser's IndexedDB, then navigates to `/c/<alias>`. Stale aliases from previous MCPs are deleted via `DELETE /cluster/<name>` before registering a new one. IndexedDB is per-browser, so different users are naturally isolated.

## Plugin sidecar

The `pluginsManager` sidecar (image: `node:lts-alpine`) runs at pod startup alongside the main Headlamp container. It executes:

```sh
npx @headlamp-k8s/pluginctl install --config /config/plugin.yml \
  --folderName /headlamp/plugins --watch
```

Plugins are written to a shared `emptyDir` volume (`/headlamp/plugins`) that both the sidecar and the Headlamp container mount. The `--watch` flag keeps the sidecar running so plugins are hot-reloaded without restarting the pod. The plugin list is stored in the `headlamp-plugin-config` ConfigMap and mounted at `/config/plugin.yml`.

## Prerequisites

- `helm` >= 3.10
- BFF (`ui-frontend`) deployed with `HEADLAMP_UPSTREAM_URL` pointing at the Headlamp service:

```
HEADLAMP_UPSTREAM_URL=http://headlamp.<namespace>.svc.cluster.local
```

## Install / upgrade

Install directly from the OCI registry (recommended):

```bash
helm upgrade --install headlamp \
  oci://ghcr.io/openmcp-project/helm-charts/headlamp-deployment \
  --version <version> \
  --namespace headlamp --create-namespace
```

Or from a local checkout:

```bash
helm dependency update
helm upgrade --install headlamp . \
  --namespace headlamp --create-namespace \
  -f values.yaml
```

## Releases

Every push to `main` triggers a GitHub Actions workflow that:

1. Packages the chart and pushes a versioned OCI image to `ghcr.io/openmcp-project/helm-charts/headlamp-deployment`.
2. Creates a GitHub Release tagged `v<version>` with the packaged `.tgz` attached.

The chart version is taken from `Chart.yaml`. Bump `version` (and `appVersion` if upgrading Headlamp) there to produce a new release.

## Feature toggle

The Headlamp tab in the MCP detail page is hidden by default. Enable it in `frontend-config.json`:

```json
{
  "featureToggles": {
    "enableHeadlamp": true
  }
}
```

## Support, Feedback, Contributing

This project is open to feature requests/suggestions, bug reports etc. via [GitHub issues](https://github.com/openmcp-project/opencontrolplane-headlamp-helmchart/issues). Contribution and feedback are encouraged and always welcome. For more information about how to contribute, the project structure, as well as additional contribution information, see our [Contribution Guidelines](https://github.com/openmcp-project/.github/blob/main/CONTRIBUTING.md).

## Code of Conduct

We as members, contributors, and leaders pledge to make participation in our community a harassment-free experience for everyone. By participating in this project, you agree to abide by its [Code of Conduct](https://github.com/openmcp-project/.github/blob/main/CODE_OF_CONDUCT.md) at all times.

## Licensing

Copyright © Linux Foundation Europe. OpenControlPlane is a project of NeoNephos Foundation. For applicable policies including privacy policy, terms of use and trademark usage guidelines, please see https://linuxfoundation.eu. Linux is a registered trademark of Linus Torvalds.
Please see our [LICENSE](LICENSE) for copyright and license information. Detailed information including third-party components and their licensing/copyright information is available [via the REUSE tool](https://api.reuse.software/info/github.com/openmcp-project/opencontrolplane-headlamp-helmchart).

<p align="center"><img alt="NeoNephos foundation logo" src="https://raw.githubusercontent.com/neonephos/.github/refs/heads/main/assets/logo.svg" width="400"/></p>
