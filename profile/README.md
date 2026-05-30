# yahav2305-sailpoint — Linker

This organization contains everything needed to run, deploy, and operate **Linker**, an internal URL shortener service. The infrastructure follows a GitOps model: merging to `main` in any repo automatically propagates the change through to the running cluster.

## Repositories

| Repo | Purpose |
| --- | --- |
| [app](https://github.com/yahav2305-sailpoint/app) | Python application source code, Dockerfile, and CI pipeline |
| [app-helm](https://github.com/yahav2305-sailpoint/app-helm) | Helm chart for deploying the app, published to GitHub Pages as an OCI-compatible Helm repository |
| [gitops](https://github.com/yahav2305-sailpoint/gitops) | Production Helm values consumed by Argo CD |

## Architecture

```text
┌──────────────────────────────────────────────────────┐
│                     Developer                        │
│         merge to main in `app`                       │
└─────────────────────┬────────────────────────────────┘
                      │ GitHub Actions (prod.yaml)
                      ▼
         Build & push multi-arch Docker image
         to ghcr.io/yahav2305-sailpoint/app
                      │
                      │ on git tag  → also bumps appVersion
                      ▼             in app-helm + tags new chart
         ┌────────────────────────┐
         │        app-helm        │
         │  Helm chart published  │
         │  to GitHub Pages OCI   │
         └────────────┬───────────┘
                      │
         ┌────────────▼───────────┐
         │          gitops        │  ← production values live here
         │  values.yaml (overlay) │
         └────────────┬───────────┘
                      │ Argo CD watches both
                      ▼
         ┌────────────────────────┐
         │     Kubernetes         │
         │  (kind / minikube /    │
         │   any cluster)         │
         └────────────────────────┘
```

## End-to-end flow

1. A developer opens a pull request against `app`. CI builds the Docker image (tagged `sha-<pr-head-sha>`), runs a Dockerfile lint, and scans for vulnerabilities with Trivy.
1. When the PR is merged to `main`, the same image is re-tagged `sha-<merge-sha>` and `latest`.
1. When a semver tag (e.g. `v1.2.0`) is pushed to `app`, CI promotes the image to that tag and opens an automated commit in `app-helm` that bumps `appVersion` and creates a new chart version.
1. Argo CD detects the new chart version, pulls the production values from `gitops`, and rolls out the new deployment automatically.

## Bootstrapping a local cluster

See the full bootstrap guide in [app-helm's README](https://github.com/yahav2305-sailpoint/app-helm#deploying-the-chart-in-a-production-kind-cluster).

## Trade-offs and known limitations

**App access via port-forward only** — the current setup exposes the service through `kubectl port-forward`, which is fine for local development but not suitable for real use. Ideally the chart would be deployed behind an Ingress controller (e.g. ingress-nginx) or at minimum a `LoadBalancer` service. The Helm chart already has `ingress` and `httpRoute` values defined; they just need a compatible controller installed in the cluster to be enabled.

**No network policies** — Kind's default CNI (kindnet) does not support `NetworkPolicy` resources, so none are defined. In a real environment I would replace kindnet with [Cilium](https://cilium.io/), which supports fine-grained L3/L4 network policies, and add policies to restrict ingress to the app pods to only the ingress controller and egress to only what the app actually needs.

**Argo CD polling instead of webhooks** — Argo CD currently detects changes by polling the Git repositories every ~3 minutes. Configuring a [GitHub webhook](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/) to notify Argo CD on push would make syncs near-instant and reduce unnecessary API calls.

## Security posture

- **Distroless, rootless image** — the runtime image contains no shell and runs as UID 65532.
- **Read-only filesystem** — enforced in the production values overlay (`gitops/values.yaml`).
- **Dropped capabilities** — `ALL` capabilities are dropped; privilege escalation is disabled.
- **Pinned Action SHA refs** — every GitHub Actions step references an immutable commit SHA, not a mutable tag.
- **Trivy scanning** — every pull request scans the built image and uploads results to GitHub Security as SARIF.
