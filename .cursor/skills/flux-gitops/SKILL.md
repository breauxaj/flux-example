---
name: flux-gitops
description: >-
  Guides Flux Operator GitOps in this repository: FluxInstance entrypoints,
  layered Kustomizations with dependsOn, base/env overlays, SOPS/age secrets,
  and CI validation. Use when adding or changing clusters/, infrastructure/,
  platform/, apps/, HelmReleases, Flux Kustomization CRs, .sops.yaml, or
  GitOps layout.
---

# Flux GitOps

Operator-managed clusters. The generated GitRepository and root Kustomization
are named `flux-system` (the FluxInstance **namespace**). Child Kustomizations
must `sourceRef.name: flux-system`. Never name a child `flux-system`.

## Current vs target

Today each cluster entrypoint is only the FluxInstance:

```
clusters/{dev,test,prod}/flux-system/
  flux-instance.yaml
  kustomization.yaml          # lists flux-instance.yaml
```

`spec.sync.path` is `clusters/<env>/flux-system` — not the whole repo and not
`clusters/<env>`. Workloads are pulled in later by **child** Flux Kustomization
CRs listed in that same `kustomization.yaml`.

Do not invent `infrastructure/`, `platform/`, `compute/`, or `apps/` until the
user asks for a real component. When they do, follow the layout below.

## Sync URL

`spec.sync.url` must be protocol-form SSH: `ssh://git@host/org/repo.git`
(slash after the host). Optional port is fine: `ssh://git@host:22/org/repo.git`.
Reject scp-style `ssh://git@host:org/repo.git`.

## Adding a layer

When introducing the first workload layer, add a Flux Kustomization CR next to
the FluxInstance and list it in `clusters/<env>/flux-system/kustomization.yaml`.

Recommended DAG (omit layers you do not need):

1. `cluster-secrets` — SOPS Secret, `prune: false`
2. `infrastructure` — controllers/CRDs, `dependsOn: cluster-secrets`, `wait: true`
3. `platform` — cluster-scoped resources that need those CRDs, `dependsOn: infrastructure`
4. `compute` — node pools / autoscaling, optional, `dependsOn: infrastructure`
5. `apps` — workloads, `dependsOn` infrastructure and platform if they exist

Split apps into core vs custom only when bootstrap order requires it (custom
`dependsOn` core).

Template (replace `NAME`, `ENV`, `PATH`):

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: NAME
  namespace: flux-system
spec:
  interval: 10m0s
  retryInterval: 2m0s
  timeout: 20m0s
  path: ./PATH/ENV
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  dependsOn:
    - name: infrastructure
  decryption:
    provider: sops
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
      - kind: Secret
        name: cluster-secrets
```

Use `suspend: true` until the layer is ready to reconcile. Secrets layer uses
`prune: false` and no `wait` requirement on apps. Never set `prune: true` with
`targetNamespace: kube-system`.

`cluster-vars` is a ConfigMap of non-secret substitutions (`cluster_env`,
`cluster_domain`, `replica_count`, …). `cluster-secrets` is a SOPS-encrypted
Secret. Both live under `clusters/<env>/` and are listed from the entrypoint
`kustomization.yaml` (vars) or a `cluster-secrets` Kustomization (secrets path).

SOPS default in this repo is **age** (`.sops.yaml`). Cloud KMS is optional.
Controller cloud-IAM annotations (IRSA, Workload Identity, …) are optional
patches on the FluxInstance, not required.

## Adding a component

Per component:

```
<layer>/base/<name>/
  kustomization.yaml
  namespace.yaml          # if needed
  helmrepository.yaml     # if Helm
  helmrelease.yaml
<layer>/<env>/<name>/
  kustomization.yaml      # resources: ../../base/<name>
  helmrelease-patch.yaml  # env-specific values only
<layer>/<env>/kustomization.yaml   # list <name>/
```

Then list `<name>/` in `<layer>/<env>/kustomization.yaml`. Do not duplicate
base resources in the overlay except via `resources:`.

HelmRelease: pin chart version (`"1.20.x"`), set install/upgrade retries, put
`${cluster_domain}` / `${replica_count}` in values for `postBuild` substitution.

## Hard rules

- Child Flux Kustomizations: `kustomize.toolkit.fluxcd.io/v1`, not native kustomize
- Overlay patches are `*-patch.yaml`; keep them small
- Encrypt `*secrets.yaml` / `cluster-secrets.yaml` with SOPS before commit
- Match existing env YAML (dev/test/prod) — do not copy one env’s path onto another

Read [reference.md](reference.md) for CI checks, substitution, and examples.
