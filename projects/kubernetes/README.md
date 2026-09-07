# Kubernetes Homelab

Talos Linux Kubernetes cluster used to host internal homelab infrastructure and applications.

## Cluster

- Cluster: `infra-services`
- Control Plane: `10.10.10.81`
- Worker 01: `10.10.10.82`
- Worker 02: `10.10.10.83`
- Talos Linux
- Kubernetes configuration managed through Flux
- Git hosted on Forgejo

## Infrastructure

- MetalLB - LoadBalancer IP allocation
- Traefik - Ingress
- cert-manager - Certificate management
- NFS CSI - Persistent storage
- Metrics Server - Kubernetes resource metrics
- Rancher - Cluster management

## Design Principles

- Git is the source of truth.
- Kubernetes nodes are treated as disposable compute.
- Persistent data lives outside the cluster on TrueNAS.
- Application installation and configuration are separated in Git.
- Infrastructure changes should normally be made through Flux rather than manually.

## Documentation

- [Architecture](docs/architecture.md)
- [GitOps](docs/gitops.md)
- [Networking and TLS](docs/tls-pki.md)
- [Storage](docs/storage.md)
- [Operations](docs/operations.md)