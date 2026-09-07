# Kubernetes Operations

## Normal Workflow

Infrastructure changes should normally follow:

1. Edit repository
2. Commit
3. Push to Forgejo
4. Flux detects or is manually reconciled
5. Verify Kubernetes state

## Cluster Status

```bash
flux get kustomizations -A
flux get helmreleases -A
kubectl get nodes
kubectl get pods -A
```

## Force Flux Reconciliation

```bash
flux reconcile kustomization <name> --with-source
```

## HelmRelease Recovery

When a Flux-managed HelmRelease reaches `RetriesExceeded` or becomes stalled, fixing the original problem may not automatically restart it.

Reset it with:

```bash
flux reconcile helmrelease <name> -n flux-system --reset
```

## PVC Troubleshooting

Check:

```bash
kubectl describe pvc <name>
kubectl get events --sort-by=.lastTimestamp
kubectl get pv
```

For NFS CSI provisioning:

```bash
kubectl logs -n kube-system deploy/csi-nfs-controller -c nfs
```

A successful NFS mount followed by:

```text
failed to make subdirectory: permission denied
```

indicates NFS filesystem/export permissions rather than connectivity or CSI failure.

## Git / Kustomize Troubleshooting

A missing file error should first be treated literally.

Check:

- Filename
- Capitalization
- Directory
- `resources:` entry

Linux paths are case-sensitive.

Example:

```text
storageClass.yaml
```

is not:

```text
storageclass.yaml
```
