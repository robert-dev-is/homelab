# GitOps and Flux

## Overview

The Kubernetes cluster is managed using Flux with Forgejo as the Git source.

The repository is structured as:

```text
kubernetes/
├── clusters/
├── services/
├── talos/
└── cheat_sheet.md
```

## Service Organization

Services are grouped by function:

```text
services/
├── management/
├── monitoring/
├── networking/
├── security/
├── storage/
└── testing/
```

## Service Convention

Helm-managed services use:

```text
service/
├── release/
│   ├── helmrepository.yaml
│   ├── helmrelease.yaml
│   ├── values.yaml
│   └── kustomization.yaml
├── config/
│   └── kustomization.yaml
└── kustomization.yaml
```

### Responsibilities

`release/`

Installs the software.

`config/`

Contains Kubernetes resources that configure the installed software.

This separation is used consistently even when a service currently has no additional configuration.

An intentionally empty config Kustomization uses:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources: []
```

## Flux Dependencies

Networking is split so MetalLB is available before services that require LoadBalancer functionality.

Current dependency model:

```text
networking-lb
      ↓
networking
      ↓
management
```

Storage does not depend on the networking Kustomization because NFS only requires normal IP connectivity to TrueNAS.

## Useful Commands

```bash
flux get kustomizations -A
flux get helmreleases -A
flux reconcile kustomization <name> --with-source
```

For a HelmRelease that has reached a terminal failed state after fixing the underlying problem:

```bash
flux reconcile helmrelease <name> -n flux-system --reset
```

## Lessons Learned

Kustomize paths are case-sensitive.

Always check:

- Filename spelling
- Filename capitalization
- Directory spelling
- References inside `kustomization.yaml`

before assuming a deeper Flux problem.
