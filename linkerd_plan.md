# Linkerd install and GitOps plan (k3s + Argo CD)

This document is the detailed companion to the Servarr stack: workloads under [`apps/servarr/`](apps/servarr/) assume Linkerd CRDs, the identity webhook, and (for policy YAML) a running control plane. **Concrete Argo CD manifests** live in [`ct27stf/argocd`](https://github.com/ct27stf/argocd) (local clone: [`argocd/argocd/`](argocd/argocd/) in this workspace; overview [`argocd/README.md`](argocd/README.md)). The [appendices](#appendix-argocd-manifests) in this file duplicate key YAML and a helper script for copy-paste. **cert-manager / trust-manager operations** are summarized in [ADR-0003 — cert-manager and trust-manager](ADR-0003-cert-manager-trust-manager.md).

## Goals

- Install **Linkerd edge** pinned to **edge-26.6.3** via Helm charts from [`https://helm.linkerd.io/edge`](https://helm.linkerd.io/edge): both `linkerd-crds` and `linkerd-control-plane` use chart version **2026.6.3** (published `appVersion` for the control-plane chart is **edge-26.6.3**; CRDs chart shares the same chart semver). **Stable mapping:** Linkerd stable **2.20** corresponds to **edge-26.6.3** ([releases](https://linkerd.io/releases/)); Homebrew formula `linkerd` **2.20** reports Client **edge-26.6.3**. When bumping edge, run `helm repo update` and align both charts from the same edge train. Both Applications use Argo CD **AppProject `platform`** (see appendix).
- Order: **cert-manager + trust-manager + Linkerd identity PKI** (Argo waves **-6 → -4**) → **Linkerd CRDs** (wave **-3**) → **control plane** (wave **-2**, external CA) → **`servarr` namespace bundle** → **Servarr workloads** (see [README.md — Commit phasing](README.md) for optional Git slices).
- **Traefik → Servarr LAN ingress** uses mesh identity `kube-system/traefik` ([`jellyfin/meshtlsauthentication.yaml`](apps/servarr/jellyfin/meshtlsauthentication.yaml), [`qbittorrent/meshtlsauthentication.yaml`](apps/servarr/qbittorrent/meshtlsauthentication.yaml)). **Implemented (2026-05-15):** meshed k3s Traefik in `kube-system`, injector allowlist via `linkerd.io/proxy-admission=enabled`, skip ports on container `8000`/`8443`/`8080` — see [SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md](SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md) and `ct27stf/argocd` `platform/traefik/helmchartconfig.yaml`.

## Prerequisites

- **Kubernetes 1.31+** (edge **26.6.x** Helm charts declare `kubeVersion: ">=1.31.0-0"`).
- [Gateway API CRDs](https://linkerd.io/2-edge/features/gateway-api/) installed if missing (edge expects them for policy / routing features).
- `kubectl` configured for the target cluster.
- Linkerd CLI matching the control plane (**edge-26.6.3**): Homebrew `linkerd` **2.20** (pin with `brew pin linkerd`), or [install-edge](https://linkerd.io/2-edge/getting-started/#step-1-install-the-cli) with `LINKERD2_VERSION=edge-26.6.3`.
- [step](https://smallstep.com/docs/step-cli/installation) CLI **or** OpenSSL, for trust anchor + issuer material **only if you use Option 1** (Helm PEM values); Option 3 uses cert-manager instead (see [Phase 0](#phase-0--cert-manager-trust-manager-and-linkerd-identity-pki-argo)).
- Argo CD **2.6+**, able to pull Helm from `https://helm.linkerd.io/edge`, **`https://charts.jetstack.io`** (cert-manager / trust-manager), and your **argocd** Git repo ([multi-source Applications](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/) for `linkerd-control-plane` `valueFiles` when using Git-hosted values).

## Phase 0 — cert-manager, trust-manager, and Linkerd identity PKI (Argo)

Applies **before** [`linkerd-crds`](#phase-a--linkerd-crds-argo-application) when using **Option 3** (cert-manager external CA; see below). All three use Argo **AppProject `platform`** (see appendix `platform.project.yaml`: Jetstack repo + destinations `cert-manager` and `linkerd`).

1. Sync **`cert-manager`** (wave **-6**): Helm `cert-manager` chart **`1.20.2`** from `https://charts.jetstack.io`, namespace `cert-manager`, `crds.enabled=true`.
2. Sync **`trust-manager`** (wave **-5**): Helm `trust-manager` chart **`0.22.1`**, namespace `cert-manager`, `app.trust.namespace=cert-manager`.
3. Sync **`linkerd-identity-pki`** (wave **-4**): Git path `argocd/platform/linkerd/identity-pki` (see [identity-pki README](argocd/argocd/platform/linkerd/identity-pki/README.md) in this repo’s `argocd/` clone). Wait for the bootstrap `Job` in `cert-manager` if the trust `Bundle` shows transient errors until `linkerd-previous-anchor` exists.

**Verify:** `kubectl -n cert-manager get deploy,pods`; `kubectl -n cert-manager get certificate`; `kubectl -n linkerd get certificate,secret`; `kubectl -n linkerd get configmap linkerd-identity-trust-roots` (after trust-manager reconciles).

**Homelab status:** Phase 0 Applications (waves **-6 … -4**) and **`linkerd-crds`** (**-3**) were **synced and healthy** as of **2026-05-13** ([SESSION-2026-05-13-linkerd-identity-pki-rollout.md](SESSION-2026-05-13-linkerd-identity-pki-rollout.md)). **`linkerd-control-plane`** (wave **-2**) is **synced and healthy** as of **2026-05-15** ([SESSION-2026-05-15-linkerd-control-plane-complete.md](SESSION-2026-05-15-linkerd-control-plane-complete.md)). Proceed to [Phase D](#phase-d--namespace-and-servarr) (`servarr-namespace`, wave **-1**, then Servarr apps).

## Phase A — `linkerd-crds` (Argo Application)

Ensure [Gateway API CRDs](https://linkerd.io/2-edge/features/gateway-api/) are present (edge relies on them for policy and routing).

Sync the `linkerd-crds` Application first (sync wave **-3** in the templates). It installs cluster-scoped CRDs (`policy.linkerd.io`, etc.) required by [`apps/servarr/_namespace/networkauthentication.yaml`](apps/servarr/_namespace/networkauthentication.yaml) and per-app policy manifests.

**Verify:** `kubectl get crd | grep linkerd` shows CRDs; Argo app Healthy.

**Homelab status:** **`linkerd-crds`** is **synced and healthy** alongside Phase 0 (2026-05-13). **`linkerd-control-plane`** is **synced and healthy** (2026-05-15); see [Phase C](#phase-c--linkerd-control-plane-argo-application) verify commands and [SESSION-2026-05-15-linkerd-control-plane-complete.md](SESSION-2026-05-15-linkerd-control-plane-complete.md).

## Phase B — Identity material (before control plane)

**Skip Phase B** if you followed **Phase 0 / Option 3** (cert-manager + trust-manager + `linkerd-identity-pki`): identity material is already in-cluster. Phase B below is for **Option 1** (Helm-inlined PEM files) and similar PEM-based flows.

The Helm chart **requires** three PEM blocks when **not** using `identity.externalCA` (no safe defaults for production):

| Helm value | Role |
|------------|------|
| `identityTrustAnchorsPEM` | Trust anchor (root CA cert) |
| `identity.issuer.tls.crtPEM` | Issuer cert |
| `identity.issuer.tls.keyPEM` | Issuer private key |

### Option 1 — Script (homelab)

From the **servarr** repo root, use [`scripts/generate-linkerd-identity-values.sh`](scripts/generate-linkerd-identity-values.sh) (or copy from the appendix). `chmod +x` if needed. Write the output **into your `ct27stf/argocd` checkout** (paths below are relative to that repo’s `argocd/` tree):

```bash
/path/to/servarr/scripts/generate-linkerd-identity-values.sh > argocd/platform/linkerd/control-plane.identity.values.yaml
```

**Do not commit private keys** to a public repository. Prefer a private fork, Sealed Secrets, SOPS, or CI you control. This repo’s [`.gitignore`](.gitignore) ignores `**/control-plane.identity.values.yaml` so a local file next to a clone is less likely to be committed by mistake.

### Option 2 — Official GitOps path (Sealed Secrets + cert-manager)

Follow [Using GitOps with Linkerd with Argo CD](https://linkerd.io/2-edge/tasks/gitops/) (trust anchor as `SealedSecret`, then set `identityTrustAnchorsPEM` on the control-plane Application). Heavier setup; best long-term for multi-node or shared repos.

### Option 3 — cert-manager as CA (GitOps default in this repo)

Use **cert-manager** and **trust-manager** so the identity issuer and trust bundle are Kubernetes-native secrets and a ConfigMap, per [Automatically rotating control plane TLS credentials](https://linkerd.io/2-edge/tasks/automatically-rotating-control-plane-tls-credentials/). In this workspace, Argo CD installs:

- **`cert-manager`** (Jetstack chart **`1.20.2`**, `targetRevision` **without** a `v` prefix) at sync wave **-6**, namespace `cert-manager`, `crds.enabled=true`.
- **`trust-manager`** (chart **`0.22.1`**) at wave **-5**, same namespace, `app.trust.namespace=cert-manager`.
- **`linkerd-identity-pki`** (Kustomize under `argocd/platform/linkerd/identity-pki/`) at wave **-4**: `linkerd` namespace (with `linkerd.io/is-control-plane=true`), trust anchor + identity `Certificate` resources, `ClusterIssuer`, trust-manager `Bundle`, and a bootstrap `Job` that seeds `linkerd-previous-anchor` for rotation.

The **`linkerd-control-plane`** Application then uses **`identity.externalCA: true`** and **`identity.issuer.scheme: kubernetes.io/tls`** in `control-plane.values.yaml` only (no `control-plane.identity.values.yaml`). Those values and Applications live in [`ct27stf/argocd`](https://github.com/ct27stf/argocd) under `argocd/platform/linkerd/` and `argocd/applications/platform/` (local clone under this workspace: `argocd/argocd/...`).

## Phase C — `linkerd-control-plane` (Argo Application)

- Sync wave **-2** (after CRDs and, when using Option 3, after Phase 0 PKI).
- Application uses **two sources**: Helm chart + same Git repo `ref: values` for `control-plane.values.yaml` (see template). When using **Option 3**, only that values file is listed; **Option 1** adds `control-plane.identity.values.yaml` as a second `valueFile`.
- Namespace `linkerd`: `CreateNamespace=true` on the Application (redundant if Phase 0 already created the namespace; harmless).

**Values split:**

- `argocd/platform/linkerd/control-plane.values.yaml` — non-secret tuning **and**, for **Option 3**, `identity.externalCA` + `identity.issuer.scheme` (see **Appendix — Helm values (non-secret)**).
- `argocd/platform/linkerd/control-plane.identity.values.yaml` — **Option 1 only:** trust anchor + issuer PEMs (**generated / secret-managed**, not committed to public repos).

**Verify:**

```bash
linkerd check
linkerd version
```

Resolve any webhook or CNI warnings before rolling injected workloads.

**Homelab status (2026-05-15):** **`linkerd-control-plane`** Application is **synced and healthy**; platform train **-6 … -2** complete. Session log: [SESSION-2026-05-15-linkerd-control-plane-complete.md](SESSION-2026-05-15-linkerd-control-plane-complete.md).

## Phase D — Namespace and Servarr

1. Sync **`servarr-namespace`** Application (path `apps/servarr/_namespace/`, sync wave **-1**) so `servarr` and `kubelet-probes` `NetworkAuthentication` exist.
2. Sync per-app Servarr Applications (wave **0** or default).

**Verify:** `kubectl -n servarr get pods`; `linkerd viz stat deploy -n servarr` (optional, if you install linkerd-viz later).

## Optional extensions

- **linkerd-viz** — dashboards / `stat` (separate Helm chart from the edge repo; add Application after control plane is healthy).
- **Proxy resource overrides** — tune under `proxy.resources` in `control-plane.values.yaml` if nodes are small.
- **Jackett → Prowlarr policy** — if you enable indexer traffic, add an `AuthorizationPolicy` (see [README.md](README.md)).

## Upgrade notes

- Bump **both** charts on the **same edge release** (same chart semver on `helm.linkerd.io/edge`). Cross-check [Artifact Hub — linkerd2-edge](https://artifacthub.io/packages/search?ts_query_web=linkerd2-edge) and [Linkerd releases](https://linkerd.io/releases/).
- Upgrade order: `linkerd-crds` first, then `linkerd-control-plane`; read edge release notes for breaking changes (Gateway API, policy CRD versions, values key renames).
- **cert-manager / trust-manager:** pin Jetstack chart versions in Argo (`targetRevision` **without** a `v` prefix, e.g. `1.20.2` / `0.22.1`); upgrade `cert-manager` before `trust-manager` if release notes require it.

## Troubleshooting

| Symptom | Check |
|---------|--------|
| Argo sync fails on policy CRDs | `linkerd-crds` synced and CRDs present |
| Injector not injecting | `linkerd check --pre` / webhook `linkerd-proxy-injector` endpoints |
| Readiness probes fail after deny | [`networkauthentication.yaml`](apps/servarr/_namespace/networkauthentication.yaml) CIDR matches **kubelet** source IP on your node |
| Traefik → backend 403 | Traefik not meshed (1/1 pod) or identity ≠ `kube-system/traefik`; see session notes |
| Injector ignores `kube-system` | Chart excludes namespace by name; label `linkerd.io/proxy-admission=enabled` + `control-plane.values.yaml` |
| `objectSelector` on `linkerd.io/inject` | Fails silently — inject is an **annotation**; objectSelector matches **labels** only |
| Traefik dashboard 502/timeout after mesh | Wrong `skip-inbound-ports` (`80,443` vs container `8000,8443`); add `skip-outbound-ports: "8080"` for `api@internal` |
| Traefik → Jellyfin 403 / TLS | MeshTLSAuthentication identity vs Traefik’s actual mesh identity / injection |

## References

- [Installing Linkerd with Helm (edge)](https://linkerd.io/2-edge/tasks/install-helm/)
- [GitOps with Argo CD (edge)](https://linkerd.io/2-edge/tasks/gitops/)
- [ADR-0002 — Servarr + Linkerd posture](ADR-0002-servarr-stack-architecture.md)

---

## Appendix: Argo CD manifests

Paths are relative to the **argocd** Git repo root (same layout as `argocd/` in [`ct27stf/argocd`](https://github.com/ct27stf/argocd): `argocd/applications/...`, `argocd/platform/...`, `argocd/projects/...`). Linkerd Applications use the **`platform`** AppProject (not `default`).

### Appendix — `platform.project.yaml`

Save as `argocd/projects/platform.project.yaml` and list it in `kustomization.yaml` **before** the Linkerd `Application` manifests:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
spec:
  description: Cluster platform components (Linkerd and similar).
  sourceRepos:
    - https://helm.linkerd.io/edge
    - https://charts.jetstack.io
    - https://github.com/ct27stf/argocd.git
  destinations:
    - namespace: cert-manager
      server: https://kubernetes.default.svc
    - namespace: linkerd
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
```

`clusterResourceWhitelist` is intentionally broad so Helm can install Linkerd CRDs, webhooks, and RBAC. Tighten later if your org requires a minimal list.

### Appendix — `cert-manager.application.yaml`

Save as `argocd/applications/platform/cert-manager.application.yaml` (sync wave **-6**). Jetstack chart `targetRevision` uses the published chart version string **without** a `v` prefix (here **`1.20.2`**).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-6"
spec:
  project: platform
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: "1.20.2"
    helm:
      values: |
        crds:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Appendix — `trust-manager.application.yaml`

Save as `argocd/applications/platform/trust-manager.application.yaml` (sync wave **-5**). Chart **`0.22.1`** (no `v` prefix).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trust-manager
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-5"
spec:
  project: platform
  source:
    repoURL: https://charts.jetstack.io
    chart: trust-manager
    targetRevision: "0.22.1"
    helm:
      values: |
        app:
          trust:
            namespace: cert-manager
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

### Appendix — `linkerd-identity-pki.application.yaml`

Save as `argocd/applications/platform/linkerd-identity-pki.application.yaml` (sync wave **-4**). Manifests live under `argocd/platform/linkerd/identity-pki/` (Kustomize).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: linkerd-identity-pki
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-4"
spec:
  project: platform
  source:
    repoURL: https://github.com/ct27stf/argocd.git
    targetRevision: main
    path: argocd/platform/linkerd/identity-pki
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
```

### Appendix — `linkerd-crds.application.yaml`

Save as `argocd/applications/platform/linkerd-crds.application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: linkerd-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
spec:
  project: platform
  source:
    repoURL: https://helm.linkerd.io/edge
    chart: linkerd-crds
    targetRevision: 2026.6.3
  destination:
    server: https://kubernetes.default.svc
    namespace: linkerd
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Appendix — `linkerd-control-plane.application.yaml`

Save as `argocd/applications/platform/linkerd-control-plane.application.yaml`. This uses [multi-source](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/) `ref: values` so Helm can load value files from the same repo. Adjust `repoURL` / `targetRevision` if your remote or branch differ.

**Option 3 (cert-manager external CA, default here):** only `control-plane.values.yaml` is listed. **Option 1 (PEM values):** add a second line `- $values/argocd/platform/linkerd/control-plane.identity.values.yaml` under `valueFiles`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: linkerd-control-plane
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  project: platform
  sources:
    - repoURL: https://helm.linkerd.io/edge
      chart: linkerd-control-plane
      targetRevision: 2026.6.3
      helm:
        valueFiles:
          - $values/argocd/platform/linkerd/control-plane.values.yaml
    - repoURL: https://github.com/ct27stf/argocd.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: linkerd
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Appendix — `servarr-namespace.application.yaml`

Save as `argocd/applications/servarr-stack/servarr-namespace.application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: servarr-namespace
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  project: servarr-stack
  source:
    repoURL: https://github.com/ct27stf/servarr.git
    targetRevision: main
    path: apps/servarr/_namespace
  destination:
    server: https://kubernetes.default.svc
    namespace: servarr
  syncPolicy:
    automated:
      prune: false
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Appendix — kustomization `resources` snippet

Append under `resources:` in `argocd/kustomization.yaml` (include **`projects/platform.project.yaml`** before Linkerd if it is not already listed):

```yaml
  - projects/platform.project.yaml
  - applications/platform/cert-manager.application.yaml
  - applications/platform/trust-manager.application.yaml
  - applications/platform/linkerd-identity-pki.application.yaml
  - applications/platform/linkerd-crds.application.yaml
  - applications/platform/linkerd-control-plane.application.yaml
  - applications/servarr-stack/servarr-namespace.application.yaml
```

### Appendix — Helm values (non-secret)

Save as `argocd/platform/linkerd/control-plane.values.yaml`:

```yaml
# edge-26.6.3 (chart 2026.6.3; stable 2.20): non-secret overrides; Option 3 adds identity.externalCA (cert-manager).
controllerReplicas: 1
identity:
  externalCA: true
  issuer:
    scheme: kubernetes.io/tls
```

### Appendix — identity values shell script

Save as `scripts/generate-linkerd-identity-values.sh` (repo root or argocd repo—run from a machine with `step` installed), `chmod +x`:

```bash
#!/usr/bin/env bash
# Emit Helm values YAML for linkerd-control-plane identity.* and identityTrustAnchorsPEM.
# Requires: https://smallstep.com/docs/step-cli/installation
# Usage: ./scripts/generate-linkerd-identity-values.sh > argocd/platform/linkerd/control-plane.identity.values.yaml

set -euo pipefail

if ! command -v step >/dev/null 2>&1; then
  echo "error: step CLI not found (install: https://smallstep.com/docs/step-cli/installation)" >&2
  exit 1
fi

tmp="$(mktemp -d)"
trap 'rm -rf "$tmp"' EXIT

step certificate create root.linkerd.cluster.local "$tmp/ca.crt" "$tmp/ca.key" \
  --profile root-ca --no-password --not-after 43800h --insecure >/dev/null
step certificate create identity.linkerd.cluster.local "$tmp/issuer.crt" "$tmp/issuer.key" \
  --profile intermediate-ca --no-password --not-after 8760h --insecure \
  --ca "$tmp/ca.crt" --ca-key "$tmp/ca.key" >/dev/null

indent_block() {
  local prefix="$1"
  local file="$2"
  sed "s/^/${prefix}/" "$file"
}

echo "identityTrustAnchorsPEM: |"
indent_block "  " "$tmp/ca.crt"
echo "identity:"
echo "  issuer:"
echo "    tls:"
echo "      crtPEM: |"
indent_block "        " "$tmp/issuer.crt"
echo "      keyPEM: |"
indent_block "        " "$tmp/issuer.key"
```
