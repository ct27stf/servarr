# ADR-0002: Servarr stack architecture, exposure model, and security posture

## Status

Accepted. Servarr workload manifests now exist under `apps/servarr/` in the `ct27stf/servarr` working tree. The manifests are authored in the **Phase 3 target shape**: all Servarr workloads carry Linkerd injection, the `servarr` namespace declares default-deny, internal allow edges are modeled with Linkerd policy resources, and `jellyfin` is prepared for Traefik identity-based ingress.

**Homelab operational (2026-05-15):** Linkerd control plane healthy; **k3s Traefik meshed**; LAN HTTPS ingress verified for `jellyfin.lan` and `qbittorrent.lan`. See [SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md](SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md).

Platform notes:

- **Linkerd** (edge **edge-26.6.3**, chart **2026.6.3**; stable **2.20** ↔ this edge) via Argo wave **-2**; PKI: **cert-manager** + **trust-manager** ([ADR-0003](ADR-0003-cert-manager-trust-manager.md), [SESSION-2026-05-15-linkerd-control-plane-complete.md](SESSION-2026-05-15-linkerd-control-plane-complete.md)).
- **Traefik** meshing and injector namespace allowlist live in `ct27stf/argocd` (`platform/traefik/`, `platform/linkerd/control-plane.values.yaml`).

**LAN DNS is running** for the control plane per ADR-0001 (`*.lan`, Argo `Application` for `ct27stf/lan-dns`). IPv6 for that stack is **not** in place yet; see ADR-0001 follow-ups (near term, non-blocking).

## Context

- This repo contains the Servarr application manifests under `apps/servarr/`.
- Platform dependencies (LAN DNS for **IPv4** `*.lan` naming, ingress) are in place for the control plane; IPv6 for DNS is still outstanding (ADR-0001).
- Argo CD configuration is intended to live in the dedicated `ct27stf/argocd` repo. The local `argocd/` directory in this workspace is ignored from the `ct27stf/servarr` repo and should be moved out later.
- LAN DNS is implemented as a separate platform component in `ct27stf/lan-dns` and is **deployed by Argo CD**.
- The stack is primarily HTTP-based today, but we expect additional protocols over time.

## Decision

### 1) Deployment model

- Use Kubernetes manifests (GitOps-managed) to deploy:

  `jellyfin`, `qbittorrent`, `prowlarr`, `sonarr`, `radarr`, `bazarr`, `jackett`, and `seerr` (Helm chart).
- Organize manifests so they are independent of where Argo CD configuration lives (only Argo `Application` wiring should move between repos).
- Keep a shared `apps/servarr/_namespace/` bundle for namespace-level concerns (`Namespace`, default-deny annotation, kubelet probe `NetworkAuthentication`).

### 2) Exposure model

- Use Traefik as the cluster ingress.
- Phase 1 exposure:

  - `jellyfin` and `qbittorrent` are exposed to the LAN via Traefik `IngressRoute` (`*.lan`, `lan-allowlist`, TLS on `websecure`).
  - `seerr` is exposed at `https://seerr.lan` (`websecure`, `homelab-lan-ca`) for media requests.
  - Other apps are internal-only by default (no LAN ingress) unless explicitly required later.
- Hostnames use `.lan` (never `.local`) and rely on LAN DNS as a dependency (see ADR-0001).

### 3) Security posture (networking)

- Adopt Linkerd service mesh with a **default-deny** policy posture for the Servarr namespace.
- Apply default-deny with the namespace annotation `config.linkerd.io/default-inbound-policy: deny` on `servarr` only.
- Permit kubelet probes through a shared Linkerd `NetworkAuthentication` for the current node IP (`192.168.68.62/32`).
- Model east-west app access with Linkerd `Server`, `MeshTLSAuthentication`, and `AuthorizationPolicy` resources selected by ServiceAccount identity:

  - `sonarr` -> `qbittorrent`
  - `radarr` -> `qbittorrent`
  - `bazarr` -> `sonarr`
  - `bazarr` -> `radarr`
  - `prowlarr` -> `sonarr`
  - `prowlarr` -> `radarr`
  - `sonarr` -> `prowlarr` (Newznab indexer API after Prowlarr sync)
  - `radarr` -> `prowlarr` (Newznab indexer API after Prowlarr sync)
  - `seerr` -> `sonarr`
  - `seerr` -> `radarr`
  - `seerr` -> `jellyfin`

- Keep `jackett` deployed with no inbound allow rule. Add `prowlarr` -> `jackett` later only if a tracker requires Jackett.
- Meshed workloads allow Traefik via a cross-namespace `MeshTLSAuthentication` for the `kube-system/traefik` ServiceAccount (`jellyfin`, `qbittorrent`, and any future LAN ingress using the same pattern). Traefik itself is meshed in `kube-system` with explicit namespace/pod opt-in (see session notes above).

## Rationale

- Internal-only by default reduces blast radius and supports a “zero-trust” mindset.
- Default-deny with explicit allow edges forces correct modeling of actual dependencies between apps (macro-world practice).
- `.lan` naming avoids mDNS `.local` ambiguity and is short enough for daily use.

## Consequences

- LAN DNS is a hard dependency for stable `*.lan` naming; it is **operational for IPv4** today. Until IPv6 is implemented (ADR-0001), assume **IPv4-only** DNS and record behavior for planning tests and client expectations.
- Linkerd introduces resource overhead and requires explicit policy authoring for inter-service traffic.
- Meshed ingress requires Traefik **container-port** skip annotations (`8000`, `8443`, …) and `skip-outbound-ports` for `api@internal` (`8080`); Service ports `80`/`443` are not sufficient.

## Follow-ups

- When prioritized: **IPv6** for LAN DNS and ingress targets (ADR-0001); not required for current IPv4 homelab use.
- qBittorrent Web UI **reverse proxy support** if HTTPS login/CSRF issues appear (application setting, not mesh).

