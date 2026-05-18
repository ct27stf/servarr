# ct27stf/servarr

Kubernetes workload manifests for the Servarr media stack. Argo CD watches each subdirectory under `apps/servarr/` and syncs it into the `servarr` namespace.

## Current status

- **Homelab (2026-05-15):** Meshed Servarr workloads, **meshed k3s Traefik**, and LAN HTTPS ingress verified for **`traefik.lan`**, **`jellyfin.lan`**, and **`qbittorrent.lan`**. Session log: [SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md](SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md).
- **Platform Linkerd (Argo waves -6 … -2)** — **`cert-manager`**, **`trust-manager`**, **`linkerd-identity-pki`**, **`linkerd-crds`**, **`linkerd-control-plane`** — synced and healthy ([SESSION-2026-05-13-linkerd-identity-pki-rollout.md](SESSION-2026-05-13-linkerd-identity-pki-rollout.md), [SESSION-2026-05-15-linkerd-control-plane-complete.md](SESSION-2026-05-15-linkerd-control-plane-complete.md)). Traefik mesh + injector allowlist live in **`ct27stf/argocd`** (`platform/traefik/`, `platform/linkerd/control-plane.values.yaml`).
- **LAN TLS:** Per-host HTTPS certs via cert-manager (`ClusterIssuer` `homelab-lan-ca`); install root CA from **`http://ca.lan/`** ([homelab-lan-pki README](argocd/argocd/platform/homelab-lan-pki/README.md)) — [ADR-0004](ADR-0004-homelab-lan-tls.md). **Pod CA trust:** trust-manager `Bundle` `homelab-lan-trust-roots` in `homelab-lan-pki` publishes `homelan.crt` to a ConfigMap in `servarr` (used by Seerr `NODE_EXTRA_CA_CERTS`).
- GitOps: keep **`servarr`** and **`argocd`** repos in sync with cluster; [linkerd_plan.md](linkerd_plan.md).

## Repo layout

```
apps/servarr/
  _namespace/                  # Namespace + shared Linkerd NetworkAuthentication
    namespace.yaml             # `servarr` Namespace, default-inbound-policy: deny
    networkauthentication.yaml # `kubelet-probes` for 192.168.68.62/32
  jellyfin/                    # Media server, LAN ingress
  qbittorrent/                 # Download client, LAN ingress (Web UI)
  prowlarr/                    # Indexer manager (modern)
  sonarr/                      # TV automation
  radarr/                      # Movie automation
  bazarr/                      # Subtitle automation
  jackett/                     # Legacy indexer proxy (deployed; no inbound allow rule)
  flaresolverr/                # Cloudflare bypass for Prowlarr (internal; prowlarr only)
  seerr/                       # Media requests (Helm chart + LAN ingress / Linkerd policy)
```

Each per-app directory contains:

- `serviceaccount.yaml` - app identity for Linkerd mTLS
- `deployment.yaml` - workload with explicit resource requests/limits and `linkerd.io/inject` annotation
- `service.yaml` - in-cluster ClusterIP service
- `pvc.yaml` - config volume on `local-path` (all apps except Jellyfin; see [Jellyfin](#jellyfin) below)
- `server.yaml` - Linkerd `Server` selecting this workload's pods on its HTTP port
- `authorizationpolicy.yaml` - one or more Linkerd `AuthorizationPolicy` resources targeting the `Server`
- `meshtlsauthentication.yaml` (only where there are inbound allowed identities)
- `certificate.yaml` + `ingressroute.yaml` on **`websecure`** (qbittorrent, prowlarr, sonarr, radarr, bazarr, seerr); `jellyfin` uses HTTP-only `ingressroute.yaml` on **`web`** (no cert)
- `seerr/` deploys the workload via the **official Seerr Helm chart** (Argo multi-source); plain YAML in that directory covers Traefik, cert-manager, and Linkerd only

## How Argo CD wires this up

Argo `Application` objects live in `ct27stf/argocd` at `argocd/applications/servarr-stack/` and already point at this repo:

```
servarr-jellyfin     -> apps/servarr/jellyfin
servarr-qbittorrent  -> apps/servarr/qbittorrent
servarr-prowlarr     -> apps/servarr/prowlarr
servarr-sonarr       -> apps/servarr/sonarr
servarr-radarr       -> apps/servarr/radarr
servarr-bazarr       -> apps/servarr/bazarr
servarr-jackett      -> apps/servarr/jackett
servarr-flaresolverr -> apps/servarr/flaresolverr
servarr-seerr        -> apps/servarr/seerr (Helm + Git; chart oci://ghcr.io/seerr-team/seerr/seerr-chart v3.6.0, app v3.2.0)
```

Additional Argo `Application` objects belong in `ct27stf/argocd` (copy from [linkerd_plan.md — Appendix](linkerd_plan.md#appendix-argocd-manifests)):

- `servarr-namespace` -> `apps/servarr/_namespace/` so the namespace and `kubelet-probes` `NetworkAuthentication` are GitOps-managed. Should sync first among Servarr apps (sync wave -1 or equivalent).
- `linkerd` -> split into **`linkerd-crds`** and **`linkerd-control-plane`** Helm Applications (see plan). **Default identity path:** **`cert-manager`**, **`trust-manager`**, and **`linkerd-identity-pki`** (sync waves -6 .. -4) supply external CA TLS; PEM `control-plane.identity.values.yaml` is **Option 1** only.

## Architecture context

- LAN DNS naming and infrastructure: [ADR-0001-lan-dns-bootstrap.md](ADR-0001-lan-dns-bootstrap.md).
- Servarr exposure model and Linkerd posture: [ADR-0002-servarr-stack-architecture.md](ADR-0002-servarr-stack-architecture.md).
- cert-manager + trust-manager (Linkerd identity PKI, Argo `platform` train): [ADR-0003-cert-manager-trust-manager.md](ADR-0003-cert-manager-trust-manager.md).
- **Linkerd install + Argo CD** (edge **edge-26.5.1**, Helm charts **2026.5.1** from `helm.linkerd.io/edge`): [linkerd_plan.md](linkerd_plan.md) — YAML and script live in that document’s appendices; Argo manifests live in [`ct27stf/argocd`](https://github.com/ct27stf/argocd) (local clone under [`argocd/argocd/`](argocd/argocd/), overview in [`argocd/README.md`](argocd/README.md)).
- Session notes: [SESSION-2026-05-08-dns-and-mesh-planning.md](SESSION-2026-05-08-dns-and-mesh-planning.md), [SESSION-2026-05-12-cert-manager-linkerd-identity.md](SESSION-2026-05-12-cert-manager-linkerd-identity.md), [SESSION-2026-05-13-linkerd-identity-pki-rollout.md](SESSION-2026-05-13-linkerd-identity-pki-rollout.md), [SESSION-2026-05-15-linkerd-control-plane-complete.md](SESSION-2026-05-15-linkerd-control-plane-complete.md), [SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md](SESSION-2026-05-15-meshed-traefik-and-qbittorrent-ingress.md).

## Linkerd policy posture (summary)

- Namespace `servarr` is annotated `config.linkerd.io/default-inbound-policy: deny`.
- All meshed pods accept inbound only when an `AuthorizationPolicy` permits it.
- Kubelet probes pass via a shared `NetworkAuthentication` (`kubelet-probes`) scoped to the k3s node IP `192.168.68.62/32`.
- Allowed Servarr east-west edges, expressed as source `ServiceAccount` -> target `Server`:

  | Source        | Target        |
  |---------------|---------------|
  | `sonarr`      | `qbittorrent` |
  | `radarr`      | `qbittorrent` |
  | `bazarr`      | `sonarr`      |
  | `bazarr`      | `radarr`      |
  | `prowlarr`    | `sonarr`      |
  | `prowlarr`    | `radarr`      |
  | `sonarr`      | `prowlarr`    |
  | `radarr`      | `prowlarr`    |
  | `prowlarr`    | `flaresolverr`|
  | `seerr`       | `sonarr`      |
  | `seerr`       | `radarr`      |
  | `seerr`       | `jellyfin`    |

- **Traefik** (k3s, `kube-system`) is meshed and presents identity `kube-system/traefik` to backends. Namespace must carry `linkerd.io/proxy-admission: enabled`; pod uses `linkerd.io/inject: enabled` and skip-port annotations (see `ct27stf/argocd` `platform/traefik/helmchartconfig.yaml`).
- `jellyfin` and `qbittorrent` allow Traefik via cross-namespace `MeshTLSAuthentication` + `AuthorizationPolicy` on their `Server` resources.
- `jackett` is deployed with no inbound allow rule; default-deny silently blocks it. Add one `AuthorizationPolicy` for `prowlarr -> jackett` if needed later.
- `flaresolverr` allows inbound only from `prowlarr` (Cloudflare CAPTCHA bypass). Configure in Prowlarr: **Settings → Indexers → FlareSolverr URL** `http://flaresolverr:8191`.

## Commit phasing

The working tree contains the full target state. When committing to `main`, you can split into phases for safer rollout:

- Phase 0: per-app `serviceaccount.yaml`, `deployment.yaml` (no `linkerd.io/inject` annotation), `service.yaml`, `pvc.yaml` (except Jellyfin); `jellyfin/ingressroute.yaml`.
- Phase 1: `apps/servarr/_namespace/*`; add `linkerd.io/inject: enabled` to the six internal apps; add `server.yaml`, `authorizationpolicy.yaml`, `meshtlsauthentication.yaml` to the six internal apps.
- Phase 2: add `linkerd.io/inject: enabled` to `jellyfin/deployment.yaml`; add `server.yaml`, `authorizationpolicy.yaml`, `networkauthentication-traefik.yaml` to `jellyfin/`.
- Phase 3: replace `jellyfin/networkauthentication-traefik.yaml` with `jellyfin/meshtlsauthentication.yaml` referencing Traefik's `ServiceAccount`.

## Conventions

- Images: `lscr.io/linuxserver/<app>` family, pinned by tag (never `latest`). **Seerr** uses `ghcr.io/seerr-team/seerr:v3.2.0` via the official Helm chart. Pin by digest when promoting to long-term.
- Resource requests and limits are mandatory on every container.
- App config storage: per-app PVC on the `local-path` storage class (Jellyfin uses hostPath; see below).
- Media storage: hostPath mounts from `/mnt/servarr/media/{tv,movies}` and `/mnt/servarr/downloads`. On the node, create `downloads/torrents/` for qBittorrent “copy .torrent files” (container path `/downloads/torrents`; set in the qBittorrent Web UI).
- Linkerd identity is the `ServiceAccount`, never pod labels.
- Hostnames use `.lan` only.

### Jellyfin

Jellyfin uses **three hostPath mounts** (no `pvc.yaml`). Prepare on the node before the first sync after this layout:

```bash
sudo mkdir -p /mnt/servarr/jellyfin/{config,transcode}
sudo chown -R 1000:1000 /mnt/servarr/jellyfin
```

| Host path | Container | Access | Purpose |
|-----------|-----------|--------|---------|
| `/mnt/servarr/jellyfin/config` | `/config` | read-write | DB, plugins, artwork cache, watched state |
| `/mnt/servarr/jellyfin/transcode` | `/transcode` | read-write | Transcode scratch (plan **32–64Gi** free space) |
| `/mnt/servarr/media` | `/data/media` | read-only | Libraries at `/data/media/tv` and `/data/media/movies` |

**Metadata:** keep **Save artwork into media folders** and **Save metadata as NFO** off in Jellyfin (Dashboard → Libraries → Metadata). Sonarr/Radarr own writes on `/mnt/servarr/media`; Jellyfin metadata stays in `/config`.

**GPU (AMD VAAPI):** the deployment mounts host `/dev/dri`. On the homelab node this is **Radeon 780M** (`renderD128`). After deploy, set **Dashboard → Playback → Transcoding**: hardware acceleration **VAAPI**, device `/dev/dri/renderD128`, transcode path `/transcode`.

**LAN access** (single-node k3s; Jellyfin is **HTTP-only** on the LAN):

| Port | Protocol | Path | Purpose |
|------|----------|------|---------|
| 80 | TCP | Traefik `web` | UI and streaming at `http://jellyfin.lan` |
| 8096 | TCP | `hostPort` on Jellyfin pod | Direct HTTP (`http://<node-ip>:8096`); Linkerd allows `lan-clients` CIDR |
| 7359 | UDP | `hostPort` on Jellyfin pod | Client auto-discovery (Linkerd skips inbound proxy on this port) |

If the node uses **ufw**, allow `80/tcp`, `8096/tcp`, and `7359/udp` from the LAN. Traefik ingress uses `lan-allowlist` middleware; direct `8096` is restricted by Linkerd `NetworkAuthentication` (`192.168.68.0/22`, `127.0.0.1/32`).

Traefik must not redirect `web` → `websecure` for Jellyfin (`ports.web.redirectTo: null` in [`platform/traefik/helmchartconfig.yaml`](argocd/argocd/platform/traefik/helmchartconfig.yaml)). Verify: `curl -v http://jellyfin.lan/` returns **200**, not **301**.

**Clients (Wholphin, Jellyfin Android TV, browsers):** use `http://jellyfin.lan` (port **80**, SSL off) or `http://<node-ip>:8096`. `JELLYFIN_PublishedServerUrl` in Git is `http://jellyfin.lan` so API image URLs match.

**After sync (one-time / if settings were migrated from HTTPS):**

1. Open **Dashboard → Networking** at `http://jellyfin.lan`.
2. **Published Server URL:** `http://jellyfin.lan` (or empty to inherit the env var).
3. **Require HTTPS:** off.
4. **LAN networks:** `192.168.68.0/22` (and guest VLAN CIDR if clients are on another subnet).
5. **Known proxies:** Traefik pod/service CIDR so reverse-proxy clients are detected correctly.

```bash
curl -v http://jellyfin.lan/                    # expect 200
curl -vk https://jellyfin.lan/                  # expect no route / fail (OK)
```

Wholphin: clear app cache once after Thumb metadata is refreshed on the server.

**Upgrading from `jellyfin-config` PVC:** before sync, copy the old PVC data to `/mnt/servarr/jellyfin/config`, then `chown -R 1000:1000`. Delete the stale PVC after the pod is healthy. Update library paths if they still point at `/data/tvshows` or `/data/movies` (use `/data/media/tv` and `/data/media/movies`).

**Future:** repoint only the transcode hostPath in Git (e.g. to a faster disk) without changing `/mnt/servarr/media`.

## Local sibling repos

`/argocd/` and `/lan-dns/` directories in this workspace are local clones of `ct27stf/argocd` and `ct27stf/lan-dns`. They are git-ignored from this repo (`.gitignore`) and will be moved out of this directory in a later cleanup pass.
