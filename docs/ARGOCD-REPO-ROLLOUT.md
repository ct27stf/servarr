# Argo CD platform repo (`ct27stf/argocd`): layout and staged rollout

**Authoritative YAML** (Applications, AppProjects, `platform/linkerd/…`) lives in [`ct27stf/argocd`](https://github.com/ct27stf/argocd). This file lives in **servarr** so it is tracked even when a local `argocd/` clone is gitignored (see [README](../README.md#local-sibling-repos)).

Related: [ADR-0003 — cert-manager and trust-manager](../ADR-0003-cert-manager-trust-manager.md), [linkerd_plan.md](../linkerd_plan.md).

## Platform layout (paths under `argocd/` in `ct27stf/argocd`)

**Linkerd train:** edge **edge-26.6.3**, Helm `https://helm.linkerd.io/edge`, chart `targetRevision` **2026.6.3** for `linkerd-crds` and `linkerd-control-plane`. Stable **2.20** corresponds to **edge-26.6.3** ([releases](https://linkerd.io/releases/)). **Jetstack:** `https://charts.jetstack.io`, **`targetRevision` without `v` prefix** — **cert-manager `1.20.2`**, **trust-manager `0.22.1`**. Platform apps use **`project: platform`**.

| Path under `argocd/` | Purpose |
|----------------------|---------|
| `projects/platform.project.yaml` | AppProject **`platform`** (Linkerd + Jetstack + this repo; **`cert-manager`**, **`linkerd`**) |
| `projects/servarr-stack.project.yaml` | AppProject **`servarr-stack`** (`ct27stf/servarr`; namespace **`servarr`**) |
| `applications/platform/cert-manager.application.yaml` | cert-manager Helm (wave **-6**) |
| `applications/platform/trust-manager.application.yaml` | trust-manager Helm (wave **-5**) |
| `applications/platform/linkerd-identity-pki.application.yaml` | PKI Kustomize (wave **-4**) |
| `platform/linkerd/identity-pki/**` | Issuer / Certificates / Bundle / bootstrap Job |
| `applications/platform/linkerd-crds.application.yaml` | Linkerd CRDs chart (wave **-3**) |
| `applications/platform/linkerd-control-plane.application.yaml` | Control plane + values (wave **-2**); Option 3: only `control-plane.values.yaml` |
| `applications/servarr-stack/servarr-namespace.application.yaml` | Namespace bundle (wave **-1**) |
| `platform/linkerd/control-plane.values.yaml` | Non-secret values (`identity.externalCA`, `proxyInjector.namespaceSelector` allowlist) |
| `platform/linkerd/control-plane.identity.values.yaml` | **Option 1 only** — do not commit PEMs to public repos |
| `platform/traefik/**` | k3s Traefik `HelmChartConfig`, `lan-allowlist`, dashboard `IngressRoute`, `kube-system` proxy-admission label |

List **`projects/*.project.yaml`** before platform `Application` entries in `argocd/kustomization.yaml`.

**Application sync waves**

| Wave | Application |
|------|-------------|
| -6 | `cert-manager` |
| -5 | `trust-manager` |
| -4 | `linkerd-identity-pki` |
| -3 | `linkerd-crds` |
| -2 | `linkerd-control-plane` |
| -1 | `servarr-namespace` |
| 0 | `servarr-*` apps |

The **root** app-of-apps lists children in **`argocd/kustomization.yaml`**. Nothing syncs until a resource appears there (unless you apply Applications manually).

### Rollout status (homelab log)

| Date | Milestone |
|------|-----------|
| 2026-05-13 | Argo **`platform`** waves **-6 … -3** healthy (`cert-manager`, `trust-manager`, `linkerd-identity-pki`, `linkerd-crds`). Session notes: [SESSION-2026-05-13-linkerd-identity-pki-rollout.md](../SESSION-2026-05-13-linkerd-identity-pki-rollout.md). |
| 2026-05-15 | Wave **-2** **`linkerd-control-plane`** healthy; platform train **-6 … -2** complete. Session notes: [SESSION-2026-05-15-linkerd-control-plane-complete.md](../SESSION-2026-05-15-linkerd-control-plane-complete.md). **Next:** wave **-1** (`servarr-namespace`), then **`servarr-*`**. |
| 2026-05-15 | **Meshed Traefik** + `qbittorrent.lan` ingress verified. **`platform/traefik/`** (HelmChartConfig inject/skip ports, `kube-system-proxy-admission.yaml`); **`control-plane.values.yaml`** proxy injector allowlist (`linkerd.io/proxy-admission=enabled`). Session: [SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md](../SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md). |

---

## Staged commits and checks

This section is for turning on the **platform** stack in Argo CD **safely**. Paths are inside **`ct27stf/argocd`**, under its `argocd/` directory.

### Two repos, one order

- **`ct27stf/argocd`**: Application CRs, AppProjects, values read from this repo.
- **`ct27stf/servarr`**: `apps/servarr/…` for `servarr-*` Applications.

**Rule:** For `source.repoURL: …/servarr.git`, the `path` on `targetRevision` must **already** exist before you sync that Application.

### Preconditions

- K8s version supported by your Linkerd edge (see [linkerd_plan.md](../linkerd_plan.md)).
- [Gateway API CRDs](https://linkerd.io/2-edge/features/gateway-api/) if the cluster does not ship them.
- Argo CD can reach Jetstack, Linkerd Helm, and both Git repos.

### Golden rule for chunked Git work

**Never append an Application to `argocd/kustomization.yaml` until every path it needs exists on the branch Argo syncs** (`main` unless otherwise set).

| Application | Must exist on branch before sync |
|-------------|----------------------------------|
| `linkerd-identity-pki` | `argocd/platform/linkerd/identity-pki/` |
| `linkerd-control-plane` | `argocd/platform/linkerd/control-plane.values.yaml` |
| `servarr-*` | Matching `apps/servarr/…` in **servarr** |

### Strategy A — One merge

1. Ensure `projects/`, `applications/`, `platform/` are complete (including `applications/servarr-stack/` when ready).
2. Update **`argocd/kustomization.yaml`** with all `Application` and `AppProject` lines.
3. Merge; sync; run [end-to-end checks](#end-to-end-checks).

### Strategy B — Staged commits

**Chunking:** You may commit YAML not yet listed in `kustomization.yaml`; add **`resources:`** lines only when ready to sync.

#### Stage 0 — AppProjects only

- Commit: `argocd/projects/platform.project.yaml`, `argocd/projects/servarr-stack.project.yaml`.
- `kustomization.yaml`: only those project lines; no Applications yet.
- Checks: `kubectl get appproject -n argocd` lists `platform` and `servarr-stack`.

#### Stage 1 — Git-backed paths (no new Applications)

- Commit: `argocd/platform/linkerd/identity-pki/**`, `argocd/platform/linkerd/control-plane.values.yaml`.
- `kustomization.yaml`: unchanged from Stage 0.
- Checks: `kustomize build argocd/platform/linkerd/identity-pki` from argocd repo root; no secrets in diff.

#### Stage 2 — cert-manager + trust-manager

- Commit both Application YAML files; append their lines to `kustomization.yaml`.
- Checks: `kubectl -n cert-manager get deploy,pods`; CRDs for cert-manager present.

#### Stage 3 — linkerd-identity-pki

- Commit Application; append to `kustomization.yaml`.
- Checks: `kubectl -n cert-manager get certificate,job`; `kubectl -n linkerd get configmap linkerd-identity-trust-roots` after trust-manager reconciles.

#### Stage 4 — linkerd-crds

- Commit Application; append to `kustomization.yaml`.
- Checks: `kubectl get crd | grep linkerd`.

#### Stage 5 — linkerd-control-plane

- Commit Application; append to `kustomization.yaml`.
- Checks: `linkerd check`; `kubectl -n linkerd get pods`.

#### Stage 6 — servarr-namespace

- Commit `servarr-namespace.application.yaml`; append to `kustomization.yaml`.
- **servarr:** `apps/servarr/_namespace/**` on `targetRevision`.
- Checks: `kubectl get ns servarr`.

#### Stage 7 — servarr workload apps

- Commit per-app `applications/servarr-stack/*.yaml`; append as ready.
- **servarr:** follow [README — Commit phasing](../README.md#commit-phasing) for mesh ordering.
- **Jellyfin (`servarr-jellyfin`):** on the node, create `/mnt/servarr/jellyfin/{config,transcode}` and `chown 1000:1000` before sync ([README — Jellyfin](../README.md#jellyfin)). If upgrading from the old `jellyfin-config` PVC, rsync into `config/` first. After sync, set library paths to `/data/media/tv` and `/data/media/movies` if needed; enable VAAPI transcoding in the Jellyfin UI.
- Checks: Argo Healthy; pods Ready.

### End-to-end checks

ADR-0003 §5 and linkerd_plan Phase 0–C. At minimum:

```bash
kubectl -n cert-manager get deploy,pods,certificate
kubectl -n linkerd get certificate,secret,configmap linkerd-identity-trust-roots
linkerd check
```

### Rollback mindset

- Helm: revert `targetRevision` in Git.
- PKI: avoid reverting live `Bundle` / identity without Linkerd rotation procedure.
- Removing a `kustomization.yaml` line stops Argo managing that app; cluster objects may remain.

### Waves vs Git stages

**Sync waves** order Applications when the parent syncs many children. **Git stages** limit how much config lands per merge; they do not replace waves.
