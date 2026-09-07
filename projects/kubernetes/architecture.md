# Cluster Architecture

## Overview

The `infra-services` Kubernetes cluster runs on Talos Linux virtual machines.

Current nodes:

| Node | Role | IP |
|---|---|---|
| `k8s-control-01` | Control Plane | `10.10.10.81` |
| `k8s-worker-01` | Worker | `10.10.10.82` |
| `k8s-worker-02` | Worker | `10.10.10.83` |

Each VM currently has:

- 4 vCPU
- 6 GB RAM

## Physical Failure Domains

`k8s-control-01` and `k8s-worker-01` run on the Ryzen 9 server.

`k8s-worker-02` runs on `usff-node-01`.

This provides two physical compute failure domains.

The cluster currently has a single control-plane node and is therefore not control-plane highly available.

## Administrative Separation

`k8s-admin-01` is the administrative system used for:

- `kubectl`
- `flux`
- Talos administration

code-server is used for editing the Git repository but intentionally does not have Kubernetes administrative access.

## Infrastructure Responsibilities

| Component | Responsibility |
|---|---|
| Talos / Kubernetes | Compute and orchestration |
| Forgejo | Git source of truth |
| Flux | GitOps reconciliation |
| MetalLB | LoadBalancer addressing |
| Traefik | Ingress and TLS termination |
| cert-manager | Certificate integration |
| TrueNAS | Persistent application storage |
| NFS CSI | Dynamic Kubernetes storage provisioning |
| PBS | VM / cluster recovery |

## Design Philosophy

Kubernetes nodes are considered replaceable compute resources.

Persistent application data should not depend on the lifecycle of an individual Kubernetes node.

The intended architecture is:

Kubernetes = compute and orchestration  
TrueNAS = persistent data  
Forgejo = desired configuration  
PBS = disaster recovery