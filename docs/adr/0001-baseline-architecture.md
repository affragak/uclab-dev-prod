# 0001. Baseline architecture of uclab-dev-prod

## Status

Accepted (retrospective — this ADR records the architecture as it exists today; it does not propose changes).

## Context

`uclab-dev-prod` has grown organically without a baseline architecture document. This is the first ADR for the cluster, established to give future design decisions something concrete to diff against. This is ADR 0001; `docs/adr/` did not exist before this document and its numbering convention (`NNNN-title.md`) starts here.

Primary source for this document is the `affragak/uclab-dev-prod` Git repository (`main` branch, commit `86e6777`), which is the GitOps source of truth. The live cluster was used only to spot-check reconciliation state, not as a primary source, because the Architect role holds a read-only kubeconfig that is explicitly denied cluster-scoped resources (Nodes, StorageClasses) and does not appear to carry Gateway API read access either. Where the repo doesn't say something and the cluster couldn't confirm it, this document says so explicitly rather than guessing.

Live-cluster spot check performed: all eight Flux `Kustomization` objects (`flux-system`, `infra-controllers`, `infra-configs`, `apps`, `hugoblog`, `monitoring-controllers`, `monitoring-configs`, `external-secrets-crds`) report `Ready: True` at `main@sha1:86e6777a`, and all expected namespaces are `Active`. The repo and the live cluster agree as of this writing.

## Decision

The following architectural choices are already embodied in the cluster.

### Provisioning and OS

- Talos Linux v1.13.8 is the node OS, with the built-in CNI disabled (`cluster.network.cni.name: none`) and kube-proxy disabled — Cilium fully replaces both.
- Kubernetes v1.36.3.
- Sidero Omni (self-hosted at `omni.uclab.dev`) provisions and manages the cluster, using the `omni-infra-provider-vsphere` infra provider, which clones VMs from a vSphere content-library template (`worker-talos-omni-1-13-8`, EFI firmware, 4 vCPU / 32 GB / 10 GB disk template spec in `omni/machineclass/worker-machine-class.yaml`).
- **3 control-plane nodes**, defined as a `ControlPlane` in `omni/cluster-template/cluster.yaml` using the `worker` machine class with `allowSchedulingOnControlPlanes: true` (no separate worker pool — all three nodes are both control plane and schedulable). Unverified: exact current node names/IPs and live health — the kubeconfig cannot list `Node` objects (cluster-scoped, outside the `view` ClusterRole this profile is bound to) and node topology wasn't otherwise available to this ADR.
- Bootstrap ordering is deliberate and encoded in `omni/patches/cluster-config.yaml` and the top-level README: Omni provisions nodes → Cilium is installed manually via Helm (release name `cilium`, values from `omni/cni/cilium-values.yaml`) so nodes reach `Ready` → Flux is bootstrapped against this repo, path `clusters/uclab-dev-prod` → Flux later adopts the same `cilium` Helm release (keeping the release name is required so it reconciles rather than duplicating) → the `vault-token` Secret is seeded manually into the `external-secrets` namespace (root of trust, intentionally not in Git) → Flux converges everything else.
- The Talos patch also: mounts `/var/lib/longhorn` and a dedicated `/dev/sdb` partition at `/var/mnt/longhorn` for Longhorn; binds etcd/scheduler/controller-manager metrics to `0.0.0.0` so Prometheus can scrape them; and pre-applies three `extraManifests` at bootstrap time (kubelet-serving-cert-approver, metrics-server, Gateway API CRDs v1.6.1) — the Gateway API CRDs are applied here specifically because Cilium's `gatewayAPI.enabled` requires them but doesn't ship them, and applying them at bootstrap avoids a manual `kubectl apply` gap that previously stalled a DR rebuild.

### GitOps (Flux)

- Flux watches `ssh://git@github.com/affragak/uclab-dev-prod`, branch `main`. The root `Kustomization` (`flux-system`, in `clusters/uclab-dev-prod/flux-system/`) syncs the GitRepository every 1 minute and reconciles every 10 minutes, path `./clusters/uclab-dev-prod`.
- Dependency graph (`dependsOn`), all in namespace `flux-system`:

  ```
  flux-system (root, path ./clusters/uclab-dev-prod)
    └─ infra-controllers        (path ./infra/controllers/uclab-dev-prod)          interval 1h
         └─ infra-configs       (path ./infra/configs/uclab-dev-prod)              interval 1h
              ├─ apps            (path ./apps/uclab-dev-prod)                       interval 10m
              │    └─ hugoblog   (path ./apps/uclab-dev-prod/hugoblog)              interval 10m
              └─ monitoring-controllers (path ./monitoring/controllers/uclab-dev-prod) interval 1h
                   └─ monitoring-configs (path ./monitoring/configs/uclab-dev-prod)     interval 1h

  external-secrets-crds (independent, its own GitRepository "external-secrets" @ tag v0.19.2, path ./deploy/crds) interval 10m
  ```

  `hugoblog` is deliberately split out of `apps` into its own Kustomization so that a not-yet-ready Forgejo registry or a signature-policy denial on the blog image only blocks `hugoblog`, never forgejo/n8n/linkding recovery — this is called out explicitly in the repo as a DR consideration. All Kustomizations set `prune: true`, `retryInterval: 1m`, `timeout: 5m` (except the Flux-managed `flux-system` and `external-secrets-crds` roots).

  `external-secrets-crds` installs CRDs from the upstream `external-secrets/external-secrets` repo directly (not from this repo), separately from the Helm release (`installCRDs: false` in the ESO `HelmRelease`) — this is intentional so CRDs and controller version stay pinned together via the GitRepository tag.

- Layout convention: each area (`infra/controllers`, `infra/configs`, `apps`, `monitoring/controllers`, `monitoring/configs`) has a `base/<component>` directory plus a `uclab-dev-prod/<component>` overlay that references it via `../../base/<component>/` — a base+overlay pattern, currently with exactly one overlay (this cluster is the only consumer). `*-controllers` paths install CRD-providing controllers (Cilium, cert-manager, ESO, CloudNativePG, Longhorn, cloudflared, MinIO object-store operator, sigstore policy-controller, Synology CSI, Gateway API `GatewayClass`/namespace); the dependent `*-configs` paths apply the resources that use those CRDs (ClusterIssuers, ClusterSecretStore, storage classes, Longhorn backup target/recurring jobs, the image policy, the Gateway itself).

### Networking

- Cilium (chart v1.20.1, values in `infra/controllers/base/cilium/values.yaml`) is the sole CNI, in full kube-proxy replacement mode (`kubeProxyReplacement: true`, `k8sServiceHost: localhost:7445`), IPAM mode `kubernetes`.
- Gateway API is enabled on Cilium (`gatewayAPI.enabled: true`, ALPN + appProtocol on), and the Gateway API CRDs are pre-seeded at Talos bootstrap (see above) rather than installed by Flux.
- L2 announcements are enabled (`l2announcements.enabled: true`) with a `CiliumL2AnnouncementPolicy` covering external IPs and LoadBalancer IPs, backed by a `CiliumLoadBalancerIPPool` of `10.10.10.50`–`10.10.10.65`.
- Hubble is enabled with Relay and a UI, plus a defined metrics set (dns, drop, tcp, flow, port-distribution, icmp, httpV2 with workload/pod/identity context). Its built-in `serviceMonitor.enabled` is deliberately left `false` — because the Prometheus Operator CRDs (from kube-prometheus-stack) don't exist yet when Cilium installs during a cold bootstrap — and Hubble/Cilium metrics are instead scraped via hand-written custom `ServiceMonitor`s under `monitoring/configs`. Grafana ships a Cilium-agent dashboard and a Hubble dashboard.
- Single shared **Cloudflare Tunnel** (controller in `infra/controllers/base/cloudflare`, config/token in `infra/configs/uclab-dev-prod/cloudflare`) is the ingress path from the internet; both the wildcard `*.uclab.dev` and the apex `uclab.dev` are routed through it to the in-cluster Gateway.
- A single Cilium `Gateway` (`uclab-gateway`, namespace `gateway-system`, `gatewayClassName: cilium`) terminates TLS for four listeners: HTTP/HTTPS on `*.uclab.dev` and dedicated HTTP/HTTPS listeners on the bare apex `uclab.dev` (the wildcard listener doesn't match the apex, so hugoblog — served at the apex — needed its own listener pair). All HTTPRoutes attach to this one Gateway; per-app `HTTPRoute` objects live under each app's `uclab-dev-prod` overlay (forgejo, hugoblog, linkding, n8n).
- Unverified: this ADR could not enumerate live `Gateway`/`HTTPRoute` status or `CiliumLoadBalancerIPPool` allocation from the cluster — the read-only kubeconfig does not appear to cover Gateway API CRDs, consistent with the RBAC note in the task brief.

### Storage

- **Longhorn** (chart v1.12.1) provides the default RWX/RWO block storage backing most app PVCs (e.g. forgejo, n8n, linkding data volumes) and is configured with `replicaSoftAntiAffinity: false` (hard anti-affinity — replicas must land on distinct nodes) and `replicaAutoBalance: best-effort`. It backs up volume snapshots over NFS to a Synology NAS (`nfs://10.10.10.9:/volume1/longhorn`) via a `RecurringJob`, explicitly to make a full worker wipe recoverable — the repo notes this addressed a gap surfaced by a prior DR incident. Longhorn also gets its own metrics `NetworkPolicy` (to allow Prometheus scraping) and Grafana dashboard.
- **Synology CSI** provisions two additional `StorageClass`es against the same Synology NAS (`10.10.10.9`) directly (not via Longhorn): `synology-iscsi-storage` (`csi.san.synology.com`, ext4, block) and `synology-nfs-storage` (NFS 4.1). Both are explicitly marked non-default (`is-default-class: "false"`). This ADR could not confirm from the repo which workloads currently consume these two classes — no app manifest under `apps/` sets `storageClassName` to either `synology-*` class (`botkube`'s PVC is the only explicit `storageClassName`, and it uses `longhorn`); the Synology CSI controllers/StorageClasses appear present in the repo but their consumers, if any beyond Longhorn's own NFS backup target, are unverified. Flag this as a documentation gap rather than an assumption either way.
- No `StorageClass` in the repo is marked `is-default-class: "true"`, so the cluster's actual default storage class (if any) could not be confirmed from Git, and live confirmation was blocked by RBAC (`storageclasses.storage.k8s.io` is cluster-scoped and outside this profile's `view` binding).

### Data

- **CloudNativePG** (chart `cloudnative-pg` v0.26.1, namespace `cnpg-system`) plus the `plugin-barman-cloud` CNPG plugin (v0.3.1) provide Postgres for forgejo, n8n, and linkding.
- Convention: each app's DB is a versioned `Cluster` object (`<app>-db-v1`) that **bootstraps by recovery** from object storage rather than from scratch, and exposes a stable `<app>-db` `rw` Service via `managed.services` — so application configuration references the stable service name and is decoupled from the underlying cluster version/generation. WAL and base backups go to the in-cluster `minio-objectstore` (its own controller under `infra/controllers/base/minio-objectstore`) via the Barman Cloud plugin. `enablePodMonitor` feeds CNPG metrics to Prometheus (`:9187`), and there's a bundled CNPG Grafana dashboard plus a default `PrometheusRule` alert set (`cnpg-alerts.yaml`).
- Unverified: this ADR did not inspect live `Cluster`/`Backup` objects, so current replica counts, backup recency, and `firstRecoverabilityPoint` per app were not confirmed — the README calls out confirming a valid base backup exists before relying on CNPG's bootstrap-by-recovery for DR, which stands as a live operational check, not something Git alone can answer.

### Secrets

- External Secrets Operator (chart v0.19.2, CRDs pinned to the same tag via a dedicated `external-secrets-crds` Flux Kustomization, `installCRDs: false` on the HelmRelease) backed by a single `ClusterSecretStore` named `vault-backend-global`, pointing at Vault (`https://vault.uclab8.net`, KV v2, mount path `apps`), authenticating via a token held in the `vault-token` Kubernetes Secret in the `external-secrets` namespace. That token Secret is intentionally excluded from Git and seeded manually — it is the cluster's root of trust; every other secret (DB creds, registry pull secrets, the Cosign key, Grafana creds, the Cloudflare tunnel token, Alertmanager's Slack/Hermes webhook credentials) materializes from Vault via `ExternalSecret` objects once that token exists.
- Observed KV-path convention from ExternalSecret manifests is **`<app>-secrets/<secret-name>`** (e.g. `forgejo-secrets/forgejo-db-credentials`, `n8n-secrets/n8n-db-env`), not uniformly `apps/<service>/<name>` as the task brief assumed — the monitoring-related secrets deviate further still, using bare paths (`hermes/alertmanager-webhook`, `alertmanager-slack`) under the same `apps` KV mount rather than an `<app>-secrets/` prefix. This ADR records what the manifests actually show rather than the assumed convention; if a single canonical convention is wanted going forward, that's a normalization decision for a future ADR, not something to retroactively assert here.
- `ExternalSecret.spec.refreshInterval` varies by consumer: 15s for DB credentials (fast rotation-sensitive path), 1h for the Alertmanager webhook secrets.

### Certificates

- cert-manager issues from a single `ClusterIssuer` (`letsencrypt-production`, ACME v2 production endpoint, DNS-01 via Cloudflare, scoped to the `uclab.dev` DNS zone only). A `staging` ClusterIssuer also exists in the repo (`clusterissuer-staging.yaml`) for testing without hitting Let's Encrypt rate limits.
- One `Certificate` (`uclab-wildcard-tls`, namespace `gateway-system`) covers both `*.uclab.dev` and the apex `uclab.dev` as SANs, and is referenced by all four Gateway listeners' `certificateRefs`.

### Observability

- **Metrics**: kube-prometheus-stack (Prometheus, Alertmanager, Grafana), discovering ServiceMonitors/PodMonitors cluster-wide. `kube-state-metrics` is extended with `customResourceState` config to emit Flux `gotk_*` metrics (Kustomization/HelmRelease readiness, suspend status, revision) for a dedicated Flux dashboard, while standard `kube_*` collectors are deliberately left enabled alongside it.
- **Logs**: Loki (single-binary, filesystem storage) plus a Grafana Alloy DaemonSet shipping pod logs with OTEL-style labels (`k8s_namespace_name`, `k8s_pod_name`, `service_name`, …).
- **Grafana**: opens on a curated "START HERE — Cluster Performance" landing dashboard (`default_home_dashboard_path` set via `grafana.ini`) linking into a "Why Is This App Slow?" investigation dashboard; served at `monitoring.uclab.dev`. Bundled dashboards: performance cockpit, app-performance, Loki logs, CloudNativePG, Cilium agent, Hubble, etcd, Longhorn, Node Exporter Full, plus several Flux dashboards (cluster, control-plane, logs).
- **Alerting (Alertmanager)**: single route tree, grouped by `[alertname, namespace]`, `group_wait: 30s`, `group_interval: 5m`, `repeat_interval: 4h`. `Watchdog` (the always-firing heartbeat alert) is routed to a `null` receiver so it reaches neither Slack nor Hermes. Every other alert fans out to **both**:
  - `slack` receiver → **`#talos-uclab-dev-prod`** (the pre-existing receiver; Slack API URL delivered via `slack_api_url_file` from a mounted secret, never plaintext in Git).
  - `hermes-webhook` receiver → Hermes's webhook endpoint at `http://10.10.10.166:8644/webhooks/alertmanager`, for enrichment; delivered to **`#hermes-alerts`** downstream of Hermes's own routing. This is a recent addition (repo commit `b821622`, "route alerts to Hermes webhook alongside Slack"), implemented as a fan-out (`continue: true` on the `slack` route so evaluation falls through to `hermes-webhook` too) rather than a replacement, specifically so the existing Slack alerting path can't be silently lost if the Hermes path misbehaves. The repo comments note how to make Hermes the sole destination later (drop `continue: true` / repoint or remove the slack route) if that's ever desired — no such change has been made.
  - Auth to the Hermes webhook is a plain shared token in the `X-Gitlab-Token` header (Alertmanager's `http_config` doesn't support HMAC, only basic/bearer/token-via-header), sourced from a mounted secret file, never plaintext in Git.
  - Both secrets (`alertmanager-slack-webhook`, `alertmanager-hermes-webhook`) are synced from Vault via ExternalSecrets and mounted into the Alertmanager pod via `alertmanagerSpec.secrets`.
- **Botkube** is also deployed, separately from kube-prometheus-stack/Alertmanager, posting to the same `#talos-uclab-dev-prod` Slack channel (`settings.clusterName: uclab-dev-prod`) — this appears to be a general cluster-events-to-Slack integration distinct from the Prometheus alert pipeline; this ADR did not fully characterize which cluster events it forwards.
- Unverified: current Alertmanager silence/inhibition state, actual alert volume/history, and whether `#hermes-alerts` is receiving traffic as expected were not checked live (would require Alertmanager/Hermes runtime inspection, out of scope for a Git-sourced baseline).

### Supply chain (hugoblog specifically)

- hugoblog's source lives in a separate Forgejo repo (`antonis/uclab`); Forgejo Actions builds and pushes the image to the in-cluster Forgejo registry, Trivy scans it, Cosign signs it, then the same pipeline clones `uclab-dev-prod` and bumps the image digest in `apps/base/hugoblog/deployment.yaml` for Flux to reconcile.
- The sigstore `policy-controller` (namespace `cosign-system`) enforces a `ClusterImagePolicy` requiring any image matching `forgejo.uclab.dev/antonis/uclab**` to be Cosign-signed with a pinned public key, in `enforce` mode. Enforcement is opt-in per namespace via the `policy.sigstore.dev/include: "true"` label — currently only applied to `hugoblog`, so this is not a cluster-wide supply-chain gate.

### Workloads deployed

Under `apps/` (`base/<app>` + `uclab-dev-prod/<app>` overlay pattern), all reconciled via the `apps` Kustomization except hugoblog (its own Kustomization, see GitOps section):

| App | Purpose | Notable details |
|---|---|---|
| hugoblog | Static blog at apex `uclab.dev` | nginx serving a prebuilt Hugo site; 2 replicas, topology-spread constraints, a PodDisruptionBudget; image pulled from the in-cluster Forgejo registry and Cosign-signature-gated (see above) |
| forgejo | Git forge, `forgejo.uclab.dev` (HTTP) / `git.uclab.dev` (SSH) | Postgres via CNPG; RWX PVCs for data/config |
| forgejo-runner | Forgejo Actions CI runner ("jotunheim-runner") | Docker-in-Docker; RWO Longhorn volume specifically because DinD's overlay snapshotter can't mount overlayfs on the NFS-backed RWX class; labels `docker`, `ubuntu-latest`, `hugo`; single replica, `Recreate` strategy |
| n8n | Workflow automation, `n8n.uclab.dev` | Postgres via CNPG |
| linkding | Bookmark manager | Postgres via CNPG; RWX data volume |
| botkube | Slack-facing cluster events bot | Posts to `#talos-uclab-dev-prod`; separate from the Alertmanager pipeline described above |

Unverified: current replica health, resource utilization, and whether all Pods are actually `Running` were not checked live — the repo describes desired state, not runtime state, and this ADR is scoped to the former per the task's "primary source is the repo" instruction. The one live check performed (Flux `Kustomization` readiness + namespace list, both green) supports but does not fully substitute for a live workload health check.

## Consequences

- This document is the reference baseline: future ADRs proposing changes to topology, networking, storage, secrets conventions, or the alerting pipeline should describe a diff against this one, not restate the whole architecture.
- Several points are marked unverified rather than asserted, specifically: live node inventory/health, live Gateway/HTTPRoute/LoadBalancerIPPool status, the actual default StorageClass and which workloads (if any) use the two Synology CSI classes, live CNPG backup recency, and live Pod/workload health. Any of these becoming load-bearing for a future decision should be independently confirmed at that time, not assumed from this document.
- The secrets KV-path convention in practice (`<app>-secrets/<name>`, with monitoring secrets on bare paths) differs from the `apps/<service>/<name>` convention referenced in this task's brief. If a single convention is wanted going forward, that requires its own ADR and a migration — this document only records current reality.
- The Hermes alert fan-out (`#hermes-alerts` alongside `#talos-uclab-dev-prod`) is additive and reversible by design (repo comments document the reversal path); this baseline does not take a position on whether/when to consolidate onto one destination.
- Any future work touching `infra/` — network policy, storage class defaults, the Vault/ESO trust chain, the Gateway/tunnel path — needs explicit human sign-off per standing process; this ADR does not grant or imply approval for changes, it only documents what's already there.
