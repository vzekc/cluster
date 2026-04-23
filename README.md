# vzekc/cluster

GitOps target for the `vzekc-prod` k3s cluster. Flux reconciles this
repo every 10 minutes (or on demand via `flux reconcile ...`). Cluster
provisioning and operator runbook live in the companion
[`vzekc/cluster-infra`](https://github.com/vzekc/cluster-infra) repo.

## Layout

```
clusters/
  production/
    flux-system/            populated by `flux bootstrap` (do not hand-edit)
    infrastructure.yaml     Flux Kustomization → ./infrastructure
    platform.yaml           Flux Kustomization → ./platform   (dependsOn: infrastructure)
    apps.yaml               Flux Kustomization → ./apps       (dependsOn: platform)

    infrastructure/         cluster-wide controllers
      sources.yaml          all HelmRepository resources (traefik, cnpg, jetstack, nats, headlamp, grafana, …)
      namespaces.yaml       prod / dev / traefik / cert-manager / external-dns / cnpg-system / observability / platform
      sealed-secrets.yaml   SealedSecret controller
      hcloud-ccm.yaml       Hetzner cloud-controller-manager
      hcloud-csi.yaml       Hetzner CSI driver + default StorageClass
      cert-manager.yaml     + cluster-issuer.yaml (Let's Encrypt prod + staging, HTTP-01 via traefik)
      external-dns.yaml     Hetzner-webhook provider
      cloudnative-pg.yaml   CNPG operator
      traefik.yaml          DaemonSet on agent nodes, NodePort 30080/30443
      k8s-monitoring.yaml   Grafana Alloy bundle → Grafana Cloud

    platform/               shared services
      nats/                 NATS + JetStream + ingress for websocket at nats-ws.k8s.classic-computing.de
      headlamp/             cluster admin UI at headlamp.k8s.classic-computing.de

    apps/                   tenant workloads
      veron/                TN3270 at veron.k8s.classic-computing.de:3270 (dev namespace)
      hasso/                photo catalogue at hasso.k8s.classic-computing.de (prod)
      exhibitron/           exhibition management at exhibitron.k8s.classic-computing.de (prod)
```

## How Flux reconciles

```
GitRepository flux-system
   │
   ▼
Kustomization flux-system  (applies clusters/production/*.yaml)
   │
   ├── Kustomization infrastructure  ← applies clusters/production/infrastructure/
   │       │
   │       ▼
   ├── Kustomization platform        ← dependsOn infrastructure
   │       │                            applies clusters/production/platform/
   │       ▼
   └── Kustomization apps            ← dependsOn platform
                                        applies clusters/production/apps/
```

Each layer adds its own resources. HelmReleases get reconciled by the
helm-controller alongside Kustomizations. `prune: true` is on, so
deleting a file (and committing) removes the resource from the
cluster on the next reconcile.

## Adding a new app

Copy `clusters/production/apps/exhibitron/` as a template (single-
container apps) or `hasso/` (two-container pods). Required files:

```
<app>/
  kustomization.yaml     list every file below
  namespace.yaml         reuse `prod` / `dev` or declare a new one (also register in infrastructure/namespaces.yaml)
  postgres.yaml          CNPG Cluster if the app needs a DB
  sealed-secrets.yaml    SESSION_SECRET and any app-specific creds (see "Sealing" below)
  certificate.yaml       cert-manager Certificate for the hostname
  deployment.yaml        pod spec — `enableServiceLinks: false`, numeric USER, resource.opentelemetry.io/service.name annotation
  service.yaml           ClusterIP
  ingress.yaml           traefik + external-dns + cert-manager annotations
  deployer-rbac.yaml     ServiceAccount + Role + long-lived token for GHA rollout-restart
```

Then add the directory to `apps/kustomization.yaml`:
```yaml
resources:
  - veron/
  - hasso/
  - exhibitron/
  - <newapp>/
```

Push. Flux reconciles within 10 minutes, or trigger immediately with
`flux reconcile kustomization apps --with-source`.

## Sealing a secret

From `vzekc/cluster-infra`:
```bash
scripts/seal.sh <namespace> <secret-name> KEY1=value1 KEY2=value2 \
  > ../cluster/clusters/production/apps/<app>/sealed-<secret-name>.yaml
```

The generated `SealedSecret` is safe to commit; only the in-cluster
controller (with its own key pair) can decrypt into a `Secret`. If the
sealed-secrets controller's key is ever lost, every SealedSecret
becomes unrecoverable — see the offline-backup step in the
[operator runbook](https://github.com/vzekc/cluster-infra/blob/main/README.md#offline-backup-of-the-sealed-secrets-master-key).

## Wiring a GHA deploy job for a new app

1. Create a `ServiceAccount` + `Role` + `RoleBinding` + long-lived token
   `Secret` in the app's namespace (see any `*/deployer-rbac.yaml`).
2. From your laptop, after Flux applies it:

   ```bash
   build_kubeconfig() { … }   # same helper as cluster-infra README
   build_kubeconfig <app>-deployer <namespace> | gh secret set KUBECONFIG_<APP>_DEPLOY --repo vzekc/<app>
   ```
3. Add a `deploy:` job to the app repo's `.github/workflows/image.yml`
   that runs `kubectl rollout restart deployment/<app>` with
   `KUBECONFIG` set from the GH secret. Exhibitron's workflow is the
   canonical small example; veron's is more elaborate because it does
   a Swank hot-reload first.

## Force a reconcile

```bash
flux reconcile source git flux-system
flux reconcile kustomization infrastructure --with-source
flux reconcile kustomization platform --with-source
flux reconcile kustomization apps --with-source
```

Or everything: `flux reconcile kustomization flux-system --with-source` (the parent).

## Debugging Flux

```bash
flux get all -A                        # overall status
flux get kustomization <name>          # specific kustomization
flux logs --all-namespaces --follow    # reconciliation activity
kubectl -n flux-system describe kustomization <name>
kubectl -n <ns> describe helmrelease <name>
```

If a Kustomization is stuck in `False` ready state with a yaml error,
the message usually pinpoints the offending file + line. Common cause:
duplicated keys in `securityContext`, or a broken HelmRepository URL.
