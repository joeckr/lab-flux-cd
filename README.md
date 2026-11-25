# lab-flux-cd

A Helm wrapper chart for deploying **Flux CD** via the official [Flux Operator](https://fluxoperator.dev) on a hardened Kubernetes cluster.

---

## Architecture & Integration

- **Flux Operator (`oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator`, pinned `0.63.1`)**: Manages the `FluxInstance` lifecycle and serves the Flux Status Web UI on port `9080`.
- **Kubernetes Gateway API (`HTTPRoute`)**: Attaches to a shared cluster `Gateway` (`traefik-gateway` in namespace `gateway`, `sectionName: web`).
- **Reverse Proxy / Tunnel**: Expects TLS termination at the edge (e.g., Cloudflare Tunnel) forwarding traffic to the cluster Gateway.
- **OIDC Authentication**: Configured via `chart/templates/secrets.yaml` (`flux-web-config`, key `config.yaml`) with credentials supplied at install time via an untracked `values.local.yaml`, `--set`, or `oidc.existingSecret`.
- **Persistent Storage (`longhorn` StorageClass)**: Backs `source-controller` artifact storage with a 5Gi `ReadWriteOnce` PVC (`source-controller-data`), `Recreate` deployment strategy, and `fsGroup: 65534` (`OnRootMismatch`).
- **Hardened Security (Kubernetes PSS `restricted`)**: Runs under `restricted` Pod Security Admission in `flux-system`, non-root execution, `RuntimeDefault` seccomp, dropped `ALL` capabilities, and control-plane/master tolerations for compact clusters.

---

## Directory Structure

```text
.
├── .github/
│   └── workflows/
│       ├── lint.yml             # Lints workflows, commits, and shell scripts
│       ├── security.yml         # Scans for secrets and audits workflow security
│       ├── release.yml          # Semantic release and Helm chart publishing to GHCR
│       └── test_release.yml     # PR dry-run test for releases and chart packaging
├── chart/
│   ├── Chart.yaml               # Wrapper chart depending on flux-operator OCI chart
│   ├── Chart.lock               # Locked dependency versions
│   ├── values.yaml              # Default configuration (OIDC, FluxInstance, PVC, HTTPRoute, RBAC)
│   └── templates/
│       ├── _helpers.tpl         # Standard Helm template helpers
│       ├── flux-instance.yaml   # FluxInstance CR with tolerations + source-controller PVC patch
│       ├── pvc.yaml             # PersistentVolumeClaim for source-controller artifacts
│       ├── rbac.yaml            # ClusterRoleBindings for Flux Web UI OIDC users/groups
│       ├── secrets.yaml         # Flux Web UI OIDC config Secret and optional Git pull Secret
│       └── NOTES.txt            # Post-install verification instructions
├── hk.pkl                       # hk git hooks and checks configuration
├── mise.toml                    # Mise tools and tasks
└── README.md
```

---

## Prerequisites

1. **Kubernetes cluster** with `kubectl` context configured.
2. **StorageClass** (defaults to `longhorn`) for `source-controller` artifact persistence.
3. **Gateway API controller** providing a shared `Gateway` (defaults to `traefik-gateway` in namespace `gateway`).
4. **Edge ingress / tunnel** routing your Flux hostname (e.g., `flux.example.com`) to the cluster Gateway service.
5. **OIDC identity provider** (e.g., Zitadel) with a **Web (Authorization Code)** application configured:
   - **Redirect URI**: `https://flux.example.com/oauth2/callback`
   - **Auth Method**: Authorization Code with Client ID and Client Secret

---

## Installation

### 1. Create the `flux-system` Namespace

Create the namespace and apply `restricted` Pod Security Standard labels:

```sh
kubectl create namespace flux-system
kubectl label namespace flux-system \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest
```

### 2. Configure Local Values & Secrets (`values.local.yaml`)

Never commit secrets to git. Create an untracked `values.local.yaml` (ignored by `.gitignore`) with your domain, OIDC credentials, and optional Git sync credentials:

```yaml
oidc:
  baseURL: "https://flux.example.com"
  issuerURL: "https://auth.example.com"
  clientID: "<oidc-client-id>"
  clientSecret: "<oidc-client-secret>"

flux-operator:
  web:
    httpRoute:
      hostnames:
        - "flux.example.com"

# Optional: enable once your GitOps repository is ready
# fluxInstance:
#   sync:
#     enabled: true
#     url: "https://git.example.com/<org>/<gitops-repo>.git"
#     ref: "refs/heads/main"
#     path: "clusters/homelab"
#     username: "<git-username>"
#     password: "<git-token>"
```

*(Alternatively, set `oidc.existingSecret` / `fluxInstance.sync.existingSecret` if you manage Secrets out-of-band.)*

### 3. Lint & Install Flux CD

```sh
mise run helm-l
mise run helm-i -- -f values.local.yaml
```

### 4. Verify Deployment

```sh
# Verify namespace PSS labels
kubectl get ns flux-system --show-labels

# Check pods, FluxInstance, PVC, and HTTPRoute
kubectl -n flux-system get pods,fluxinstance,pvc,httproute

# Optional local port-forward to the Flux Web UI (http://localhost:9080)
mise run ui
```

---

## Available Tasks

Run tasks with `mise run <task>`:

| Task | Description | Command |
|---|---|---|
| `install` | Install tools and set up git hooks | `hk install --mise` |
| `hk` (or `check`) | Run all linters and hook checks | `hk check --all` |
| `helm-d` | Build Helm chart dependencies | `helm dependency build chart/` |
| `helm-up` | Update Helm chart dependencies and `Chart.lock` | `helm dependency update chart/` |
| `helm-l` | Lint Helm chart | `helm lint chart/` |
| `helm-t` | Render Helm chart templates to `rendered.yaml` | `helm template flux chart/ --namespace flux-system > rendered.yaml` |
| `helm-i` | Install or upgrade Flux CD in `flux-system` | `helm upgrade --install flux chart/ --namespace flux-system` |
| `helm-u` | Uninstall Flux CD from `flux-system` | `helm uninstall flux --namespace flux-system` |
| `ui` | Port-forward Flux Web UI to `http://localhost:9080` | `kubectl -n flux-system port-forward svc/flux-operator 9080:9080` |
| `trivy-fs` | Scan repository filesystem for security vulnerabilities | `trivy fs .` |

---

## Support

If you find this project useful, consider supporting my work on [Ko-fi](https://ko-fi.com/joeckr):

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/joeckr)

## License

Please refer to the `LICENSE` file for details.
