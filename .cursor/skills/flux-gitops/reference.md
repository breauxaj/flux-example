# Flux GitOps reference

Read this when adding layers, HelmReleases, substitution, or CI-related
manifests.

## Entrypoint

`clusters/<env>/flux-system/kustomization.yaml` is native Kustomize. It must
include `flux-instance.yaml`. Add child Flux Kustomization files here as they
are created (`infrastructure.yaml`, `apps.yaml`, `kustomizations.yaml`, …).

FluxInstance `spec.sync.path` must equal `clusters/<env>/flux-system`.

Generated source name is the FluxInstance **namespace** (`flux-system`), not
`metadata.name` (`flux-dev` / `flux-test` / `flux-prod`).

## Substitution

kubeconform runs on source YAML **before** Flux `postBuild`. Integer fields
must not be unquoted `${replica_count}` at validate time unless CI substitutes
a dummy. CI replaces:

- `${replica_count}` → `1`
- `${cluster_domain}` → `example.com`

Use those names for portable overlays. Do not invent org-prefixed variable
names.

## HelmRelease skeleton

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: example
  namespace: example
spec:
  interval: 1h
  timeout: 10m
  install:
    remediation:
      retries: 3
  upgrade:
    remediation:
      retries: 3
      remediateLastFailure: true
    cleanupOnFail: true
  chart:
    spec:
      chart: example
      version: "1.0.x"
      interval: 12h
      sourceRef:
        kind: HelmRepository
        name: example
        namespace: flux-system
  values:
    replicaCount: ${replica_count}
```

CRDs from a HelmRelease belong in **infrastructure**. Resources that need those
CRDs belong in **platform** or **apps** with `dependsOn`. Putting both in the
same Flux Kustomization causes dry-run deadlocks.

## CI (`.github/workflows/ci.yml`)

Pinned tools. On `pull_request` and `push` to `develop`/`main`:

1. Reject scp-style FluxInstance SSH URLs (`ssh://git@host:org/...`)
2. `kustomize build` each `clusters/{dev,test,prod}/flux-system`
3. kubeconform YAML with CRDs-catalog and `--ignore-missing-schemas`, skipping
   `kustomization.yaml`, `*-patch.yaml`, `*secrets*.yaml`, `.sops.yaml`
4. Fail if `*secrets*.yaml` / `cluster-secrets.yaml` exists without a `sops:`
   stanza
5. Fail Flux Kustomizations with `prune: true` and `targetNamespace: kube-system`

When layer directories exist, extend the workflow’s `kustomize build` loop to
`<layer>/{dev,test,prod}` the same way. Do not add builds for trees that are
absent.

## Secrets Kustomization

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cluster-secrets
  namespace: flux-system
spec:
  decryption:
    provider: sops
  interval: 10m0s
  retryInterval: 2m0s
  path: ./clusters/ENV/secrets
  prune: false
  sourceRef:
    kind: GitRepository
    name: flux-system
```

Age key in-cluster: Secret `sops-age` in `flux-system` (see README). Optional
cloud KMS goes in `.sops.yaml` `creation_rules` per path; do not require it.
