# lab-traefik-ingress

A Helm wrapper chart repository tailored for deploying a hardened, rootless Traefik Ingress Controller (`traefik`) to Kubernetes clusters (such as Talos Linux) enforcing Pod Security Standards (`restricted` profile).

> **Note**: This repository is an umbrella/wrapper Helm chart wrapping the official upstream [`traefik`](https://traefik.github.io/charts) chart. It provides pre-configured, production-hardened defaults for clusters that enforce rootless execution across all namespaces.

---

## Features

- **Helm Wrapper Chart (`chart/`)**:
  - Overrides and configures the official upstream `traefik` chart (version `42.0.0`, Traefik `v3.7.14`).
  - **Rootless & Hardened by Default**: Strictly compliant with the Kubernetes **Pod Security Admission (PSA) `restricted` profile**.
  - **Zero Elevated Capabilities**: Runs completely unprivileged by dropping `ALL` capabilities (`capabilities.drop: ["ALL"]`) with *zero* added capabilities. Binds to unprivileged container ports `8000` (`web`) and `8443` (`websecure`).
  - **Read-Only Root Filesystem**: Enforces `readOnlyRootFilesystem: true` with ephemeral `emptyDir` volumes mounted for `/tmp` and `/data`.
  - **No Host Networking**: Sets `hostNetwork: false`, preventing host namespace access and ensuring strict network isolation.
  - **Unprivileged User Execution**: Runs under non-root UID `65532` (`nonroot`), GID `65532`, and `fsGroup: 65532`.
  - **Service-Based Traffic Ingress**: Configured with a `LoadBalancer` service mapping external ports `80 -> 8000` and `443 -> 8443` without requiring host port mapping.
  - **DaemonSet Architecture**: Deploys as a `DaemonSet` with control-plane and master node tolerations for compact lab and bare-metal environments.
  - **Dual Provider Support**: Out-of-the-box support for both standard Kubernetes Ingress (`networking.k8s.io/v1`) and Traefik Custom Resource Definitions (`traefik.io` CRDs like `IngressRoute`, `Middleware`, and `TLSOption`).
  - **Default IngressClass**: Provisions and designates the `traefik` IngressClass as default (`isDefaultClass: true`).
- **Compose (`compose.yml`)**:
  - Spin up and test Traefik locally with Podman Compose without needing a cluster.
  - Pre-configured entrypoints, dashboard access, and healthcheck.
- **Environment & Tooling (`mise` & `hk`)**:
  - `mise.toml`: Tool version management (`helm`, `betterleaks`, `addlicense`, `trivy`, `actionlint`, `shellcheck`, `zizmor`, `hk`, `pkl`, `tombi`, `yamllint`) and task aliases.
  - `hk.pkl`: Fast git hooks and project checks enforcing Conventional Commits, branch protection, secret scanning (`betterleaks`), Helm linting, workflow linting (`actionlint`), security audits (`zizmor`), YAML linting (`yamllint`), TOML linting (`tombi`), shell checking (`shellcheck`), and license checks (`addlicense`).
- **GitHub Actions CI (`.github/workflows/`)**:
  - Reusable workflows powered by [`joeckr/ci-templates`](https://github.com/joeckr/ci-templates):
    - `lint.yml`: Lints GitHub Actions workflow syntax (`actionlint`), Conventional Commits (`commitlint`), and shell scripts (`shellcheck`).
    - `security.yml`: Scans commits and PRs for secret leaks (`betterleaks`) and workflow security audits (`zizmor`).
    - `release.yml`: Automated Semantic Versioning, git tagging, release notes, `Chart.yaml` version syncing (`semantic`), and packaging/publishing Helm charts to GitHub Container Registry (GHCR) as OCI artifacts.
    - `test_release.yml`: PR dry-run preview for Semantic Versioning and Helm chart packaging.

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
│   ├── Chart.yaml               # Wrapper chart definition with traefik dependency
│   ├── Chart.lock               # Pinned dependency lockfile
│   ├── values.yaml              # Hardened rootless configuration overrides
│   ├── .helmignore              # Ignore rules for chart packaging
│   └── templates/
│       └── .gitkeep             # Wrapper chart templates placeholder
├── compose.yml                  # Local lab service definition
├── hk.pkl                       # hk git hooks and checks configuration
├── mise.toml                    # Mise tools and tasks
└── README.md
```

---

## Security & Hardened Architecture

Modern hardened Kubernetes environments (such as Talos Linux or clusters with Pod Security Admission enabled) enforce least privilege and namespace isolation. This wrapper chart is specifically tailored to meet the Kubernetes **Pod Security Standards (PSS) `restricted` profile**.

### Hardened Container Standards (Kubernetes PSS `restricted`)

Under the Kubernetes **Pod Security Admission (PSA)** `restricted` profile:

- **Must Run As Non-Root**: Pod specification sets `runAsNonRoot: true` with non-root UID `65532` and GID `65532`. Containers cannot execute as UID 0.
- **No Host Namespaces (`hostNetwork: false`)**: Traefik runs inside its own isolated network namespace, preventing unauthorized access to the host network stack.
- **Drop All Capabilities**: Traefik runs on unprivileged container ports (`8000`, `8443`, `8080`, `9100`) and explicitly drops all capabilities (`capabilities.drop: ["ALL"]`). Unlike ingress controllers binding directly to ports `< 1024`, Traefik does not require `NET_BIND_SERVICE` or file capabilities, preventing `EPERM` kernel execution rejections under `allowPrivilegeEscalation: false`.
- **Read-Only Root Filesystem**: The container runs with `readOnlyRootFilesystem: true`, mounting ephemeral `emptyDir` volumes at `/tmp` and `/data` to prevent runtime tampering with container binaries or configuration files.
- **Disallow Privilege Escalation**: Pods enforce `allowPrivilegeEscalation: false` to prevent child processes from acquiring additional privileges.
- **RuntimeDefault Seccomp**: Workloads enforce `seccompProfile: { type: RuntimeDefault }` to restrict syscalls to safe defaults.
- **Group Isolation**: Configured with `fsGroup: 65532` matching the controller's runtime group.

### Compliance Matrix

| Security Dimension | Restricted Standard Requirement | Implementation in This Repo |
|---|---|---|
| **Workload Execution** | Non-root UID (`runAsNonRoot: true`) | UID `65532` (`nonroot`), GID `65532`, `runAsNonRoot: true` |
| **Group Permissions** | Non-root GID / `fsGroup` | `fsGroup: 65532` |
| **Host Namespaces** | Forbidden (`hostNetwork: false`) | `hostNetwork: false` |
| **Capabilities** | Must drop `ALL` capabilities | `capabilities.drop: ["ALL"]` (no elevated capabilities required) |
| **Privilege Escalation** | Prohibited (`allowPrivilegeEscalation: false`) | `allowPrivilegeEscalation: false` |
| **Root Filesystem** | Read-only root filesystem recommended | `readOnlyRootFilesystem: true` with ephemeral `emptyDir` for `/tmp` and `/data` |
| **Port Binding** | Unprivileged (> 1024) | Listens on container ports `8000` (web), `8443` (websecure), `8080` (internal/ping), `9100` (metrics) |
| **Seccomp Profile** | `RuntimeDefault` or `Localhost` | `seccompProfile: { type: RuntimeDefault }` |
| **Service Account Token** | Scoped API access | Managed via upstream chart RBAC & ServiceAccount |
| **Traffic Exposure** | Standard Kubernetes Service | `type: LoadBalancer` exposing external ports `80` and `443` |

---

## Testing & Deployment Process

### 1. Build Chart Dependencies

Before linting or rendering, download the upstream `traefik` dependency tarball:

```bash
mise run helm-d
# or: helm dependency build chart/
```

### 2. Linting & Manifest Validation

Validate chart syntax and render Kubernetes manifests:

```bash
# Lint the chart
mise run helm-l
# or: helm lint chart/

# Render templates to rendered.yaml
mise run helm-t
# or: helm template test chart/ > rendered.yaml
```

### 3. Deploying to the Kubernetes Cluster

Deploy the controller to your active cluster:

```bash
mise run helm-i
# or: helm install test chart/
```

### 4. Verifying Health & Controller Status

Verify that all controller pods run cleanly without restarts:

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/name=traefik

# Inspect pod events for any security policy rejections
kubectl describe pod -l app.kubernetes.io/name=traefik

# Stream controller logs
kubectl logs -l app.kubernetes.io/name=traefik -f

# Verify the LoadBalancer service
kubectl get svc -l app.kubernetes.io/name=traefik
```

### 5. Uninstalling from the Cluster

Clean up the deployed release:

```bash
mise run helm-u
# or: helm uninstall test
```

---

## Local Development & Tasks

### Setup

```bash
# Install tools and git hooks
mise run install
```

### Available Tasks

Run tasks with `mise run <task>`:

| Task | Description | Command |
|---|---|---|
| `install` | Install tools and set up git hooks | `hk install --mise` |
| `hk` (or `check`) | Run all linters and hook checks | `hk check --all` |
| `compose` | Start local Traefik container with Podman Compose | `podman compose up -d` |
| `down` | Stop local Podman Compose stack | `podman compose down` |
| `logs` | View Podman Compose logs | `podman compose logs -f` |
| `play` | Test Helm chart manifests locally with Podman Play Kube | `podman play kube rendered.yaml --publish-all` |
| `play-d` | Stop and remove Podman Play Kube pods | `podman play kube rendered.yaml --down` |
| `helm-d` | Build Helm chart dependencies | `helm dependency build chart/` |
| `helm-l` | Lint Helm chart | `helm lint chart/` |
| `helm-t` | Render Helm chart templates to `rendered.yaml` | `helm template test chart/ > rendered.yaml` |
| `helm-i` | Install Helm chart to current Kubernetes cluster | `helm install test chart/` |
| `helm-u` | Uninstall Helm chart release from cluster | `helm uninstall test` |
| `trivy-fs` | Scan repository filesystem for security vulnerabilities | `trivy fs .` |

---

## Conventional Commits & Releases

Commits must follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:
- `feat: add new feature` -> Triggers a **minor** release (e.g., `v0.1.0` -> `v0.2.0`).
- `fix: resolve issue` -> Triggers a **patch** release (e.g., `v0.1.0` -> `v0.1.1`).
- `feat!: breaking change` -> Triggers a **major** release.
- `chore:`, `docs:`, `ci:`, `test:`, `refactor:` -> Maintenance changes (no release bump).

Upon merging to `main`, the `release.yml` workflow automatically computes the next version, updates `version` and `appVersion` in `chart/Chart.yaml`, creates a Git tag, publishes a GitHub Release, and packages and pushes the Helm chart to GHCR.

## Support

If you find this project useful, consider supporting my work on [Ko-fi](https://ko-fi.com/joeckr):

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/joeckr)

## License

Please refer to the `LICENSE` file for details.
