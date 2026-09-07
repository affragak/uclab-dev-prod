# uclab-dev-prod

A GitOps-driven Kubernetes cluster on **Talos Linux**, provisioned via **Sidero Omni** (self-hosted) using the **vSphere infrastructure provider**, networked with **Cilium**, and managed via **Flux**.

## Platform

| Component | Notes |
|---|---|
| Talos Linux | v1.13.8, no default CNI (`cluster.network.cni.name: none`) |
| Kubernetes | v1.36.3 |
| Omni | Self-hosted at `omni.uclab.dev`, provisions/manages the cluster |
| Infra provider | [omni-infra-provider-vsphere](https://github.com/siderolabs/omni-infra-provider-vsphere) — clones VMs from a vSphere content library template |
| CNI | Cilium (kube-proxy replacement, Gateway API, Hubble) |
| Ingress | Gateway API (Cilium) behind a single shared **Cloudflare Tunnel**; `*.uclab.dev` **and** the apex `uclab.dev` are routed to the Gateway |
| GitOps | Flux, bootstrapped after Cilium brings nodes to `Ready` |
| Storage | Longhorn (`RWX`/`RWO` volumes); Synology CSI |
| Databases | CloudNativePG — versioned clusters, backed up to MinIO via the Barman Cloud plugin |
| Secrets | External Secrets Operator + Vault (`ClusterSecretStore` `vault-backend-global`, KV mount `apps`) |
| TLS | cert-manager (Let's Encrypt, DNS-01 via Cloudflare); one cert covers `*.uclab.dev` + `uclab.dev` |

## Observability

Deployed under `monitoring/`, discovered by the Grafana sidecar (`grafana_dashboard` ConfigMaps).

| Component | Notes |
|---|---|
| Metrics | kube-prometheus-stack (Prometheus, Alertmanager, Grafana); discovers all ServiceMonitors/PodMonitors cluster-wide |
| Logs | Loki (single-binary, filesystem) + **Grafana Alloy** DaemonSet shipping pod logs with OTEL labels (`k8s_namespace_name`, `k8s_pod_name`, `service_name`, …) |
| Landing page | Grafana opens on **"START HERE — Cluster Performance"** → links into **"Why Is This App Slow?"** |
| Dashboards | performance cockpit, app-performance investigator, Loki logs, CloudNativePG, Cilium agent, Hubble, etcd |
| Datasources | Prometheus, Loki, Alertmanager |
| Network telemetry | Cilium + Hubble metrics via ServiceMonitors |
| DB telemetry | CNPG PodMonitors (`:9187`) + default CNPG PrometheusRule alerts |

Grafana is at `grafana.uclab.dev`.

## Applications

Under `apps/` (each: `base/<app>` + `uclab-dev-prod/<app>` overlay).

| App | Purpose | Notes |
|---|---|---|
| hugoblog | Static site at **`uclab.dev`** (apex) | nginx serving a Hugo build; 2 replicas + topology spread + PDB; image pulled from the Forgejo registry |
| forgejo | Git forge at `forgejo.uclab.dev` (SSH `git.uclab.dev`) | Postgres via CNPG |
| forgejo-runner | Forgejo Actions runner (`jotunheim-runner`) | Docker-in-Docker (RWO volume so overlayfs works); labels `docker`, `ubuntu-latest`, `hugo` |
| n8n | Workflow automation at `n8n.uclab.dev` | Postgres via CNPG |
| linkding | Bookmark manager | Postgres via CNPG |

### CloudNativePG databases

Each app DB is a versioned cluster (`<app>-db-v1`) that **bootstraps by recovery** from the object store and exposes a **stable `rw` service** (`<app>-db`) via `managed.services`, so app config is independent of the cluster version. WAL/base backups go to MinIO (`minio-objectstore`) through the Barman Cloud plugin; `enablePodMonitor` feeds metrics to Prometheus.

## CI/CD & supply chain (hugoblog)

The blog source lives in Forgejo (`antonis/uclab`) and deploys itself into this repo:

1. **Forgejo Actions** builds the image and pushes it to the Forgejo registry `forgejo.uclab.dev/antonis/uclab`.
2. **Trivy** scans; **Cosign** signs the image (key from CI secrets, stored base64).
3. The pipeline clones this repo and bumps the digest in `apps/base/hugoblog/deployment.yaml`.
4. **Flux** reconciles and rolls the Deployment; pods pull from the Forgejo registry using the `forgejo-registry` pull secret (synced from Vault).

**Signature enforcement:** the **sigstore policy-controller** (`cosign-system`) enforces a `ClusterImagePolicy` requiring `forgejo.uclab.dev/antonis/uclab**` images to be Cosign-signed with the project public key. Enforcement is opt-in per namespace via the `policy.sigstore.dev/include: "true"` label (currently: `hugoblog`). Because signing runs *before* the deploy step, an unsigned image never reaches the cluster.

## Repository layout

```
clusters/uclab-dev-prod/   # Flux entrypoints (Kustomizations: infra, apps, monitoring)
infra/
  controllers/             # platform: cilium, cert-manager, eso, cloudnativepg,
                           #   longhorn, gateway, cloudflare, minio-objectstore,
                           #   policy-controller, synology-csi
  configs/                 # their configs (image policy, issuers, etc.)
apps/                      # forgejo, forgejo-runner, hugoblog, linkding, n8n
monitoring/
  controllers/             # kube-prometheus-stack, loki, alloy
  configs/                 # Grafana dashboards, ServiceMonitors, alerts
omni/                      # Omni cluster template / machine config
```

Each area follows a `base/` + per-cluster `uclab-dev-prod/` overlay pattern. HelmReleases live in their target namespace and reference an in-namespace `HelmRepository`.

## GitOps reconcile order

Flux `Kustomization` dependencies:

```
flux-system
  └─ infra-controllers
       └─ infra-configs
            ├─ apps
            └─ monitoring-controllers
                 └─ monitoring-configs
```

CRD-providing controllers (CloudNativePG, Prometheus Operator, sigstore policy-controller) are installed by `*-controllers`; the resources that use those CRDs (Cluster, ServiceMonitor, ClusterImagePolicy, …) are applied by the dependent `*-configs`.

## Bootstrap

Order matters: the CNI is `none`, so nodes stay `NotReady` until Cilium is installed, and most secrets come from Vault.

1. **Provision (Omni).** Apply the cluster template `omni/cluster-template/cluster.yaml` — Talos v1.13.8 / Kubernetes v1.36.3, 3 control-plane + 3 workers from the machine classes in `omni/machineclass/`. The `omni/patches/disable-cni.yaml` patch disables kube-proxy and the built-in CNI and adds bootstrap `extraManifests` (kubelet-serving-cert-approver, metrics-server).
2. **CNI (Cilium).** Install Cilium via Helm using `omni/cni/cilium-values.yaml` (release name `cilium`, kube-proxy replacement). Once the agents are up the nodes go `Ready`. Flux later adopts this same release via `infra/controllers/base/cilium` (chart 1.20.1) — keep the release name `cilium` so it reconciles instead of duplicating.
3. **Flux.** Bootstrap Flux against this repo (`https://github.com/affragak/uclab-dev-prod.git`, branch `main`, path `clusters/uclab-dev-prod`). This installs the GitOps controllers and the root sync in `clusters/uclab-dev-prod/flux-system/`.
4. **Seed the Vault token (root of trust).** The `vault-backend-global` `ClusterSecretStore` authenticates to `https://vault.uclab8.net` with a token from a Kubernetes secret **`vault-token`** in the ESO namespace. This secret is intentionally **not in Git** — create it manually. Every other secret (DB creds, registry pull secrets, the Cosign key, Grafana creds, the Cloudflare tunnel token) then materializes from Vault via ExternalSecrets.
5. **Converge.** Flux reconciles in the dependency order above: `infra-controllers` → `infra-configs` → `apps` + `monitoring-*`.

## Disaster recovery

| Layer | Recovery |
|---|---|
| Cluster / platform | Re-run bootstrap (Omni → Cilium → Flux → seed `vault-token`). Flux rebuilds all controllers, configs, apps, and monitoring from Git. |
| Secrets | Reappear automatically from Vault via ESO once `vault-token` is seeded. |
| Databases | CNPG `<app>-db-v1` clusters **bootstrap by recovery** from MinIO (Barman base backup + WAL) with no manual steps, provided valid backups exist in `minio-objectstore`. Confirm a base backup / `firstRecoverabilityPoint` exists before relying on it. |
| Ingress / DNS | Cloudflare tunnel token comes from Vault; the tunnel ingress config and DNS records live in Cloudflare. |
| Persistent volumes | Longhorn volumes via Longhorn's own backup/restore; DinD/build caches are ephemeral and rebuild themselves. |

External dependencies that must survive a cluster loss (back them up independently — they are the roots of trust/state): **Omni**, **Vault** (`vault.uclab8.net`), the **MinIO** object store (DB backups), the **Forgejo registry** (`forgejo.uclab.dev`), and **Cloudflare** (tunnel + DNS).
