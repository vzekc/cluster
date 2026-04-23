# vzekc/cluster

GitOps target for the vzekc k3s cluster. Flux reconciles this repo into
the cluster provisioned by
[`vzekc/cluster-infra`](../cluster-infra).

Layout:

```
clusters/production/
  flux-system/         # populated by `flux bootstrap`
  infrastructure.yaml  # Flux Kustomization -> ./infrastructure
  platform.yaml        # Flux Kustomization -> ./platform (depends on infra)
  apps.yaml            # Flux Kustomization -> ./apps (depends on platform)
  infrastructure/      # cluster-wide controllers (cert-manager, external-dns, ...)
  platform/            # shared services (NATS, …)
  apps/                # tenant workloads (veron, …)
```

Workflow:

- `cluster-infra` stands up iron and runs `bootstrap-cluster.sh`, which
  installs sealed-secrets + hcloud CCM/CSI + cert-manager (+ webhook) +
  external-dns + CloudNativePG + grafana-agent and then
  `flux bootstrap`s this repo.
- After that, all changes happen via PRs here. Flux adopts the helm
  releases installed during bootstrap by matching their release names.

Sealing secrets:

```bash
# in cluster-infra/
scripts/seal.sh <namespace> <secret-name> key1=value1 key2=value2 \
  > ../cluster/clusters/production/apps/<app>/sealed-<secret-name>.yaml
```
