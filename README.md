# uclab-dev-prod

A GitOps-driven Kubernetes cluster on **Talos Linux**, provisioned via **Sidero Omni** (self-hosted) using the **vSphere infrastructure provider**, networked with **Cilium**, and managed via **Flux**.

## Stack

| Component | Notes |
|---|---|
| Talos Linux | v1.13.8, no default CNI (`cluster.network.cni.name: none`) |
| Kubernetes | v1.36.3 |
| Omni | Self-hosted at `omni.uclab.dev`, provisions/manages the cluster |
| Infra provider | [omni-infra-provider-vsphere](https://github.com/siderolabs/omni-infra-provider-vsphere) — clones VMs from a vSphere content library template |
| CNI | Cilium (kube-proxy replacement, Gateway API, Hubble) |
| GitOps | Flux, bootstrapped after Cilium brings nodes to `Ready` |
