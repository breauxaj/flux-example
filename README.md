# Flux Example

## Operator Install

Prepare the target cluster(s) by installing the Flux Operator:

```bash
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace
```

This is preferred over the flux bootstrap method. Much cleaner and only
requires readonly access to the git respository.

## SSH Keys

Each cluster will need SSH keys to access the flux repository. Generate a set
of SSH keys (be careful not to overwrite your own keys):

```bash
ssh-keygen -t ed25519
```

Add the SSH keys as a secret to the cluster:

```bash
kubectl create secret generic flux-system \
  --namespace flux-system \
  --from-file=identity=$HOME/.ssh/flux_id_ed25519 \
  --from-file=identity.pub=$HOME/.ssh/flux_id_ed25519.pub \
  --from-literal=known_hosts="$(ssh-keyscan bitbucket.org 2>/dev/null)"
```

## FluxInstance

To connect the cluster to the Git repository, apply the flux-instance:

```bash
kubctl apply -f clusters/dev/flux-system/flux-instance.yaml
```

This will start the process of reconciliation.

## Secrets

We're going to use SOPS/Age to manage secrets, it requires no external
resources and can safeky exist in the repository. Flux has built in support
for SOPS and age encrypted secrets.

```bash
brew install age
```

Generate an Age key

```bash
mkdir -p $HOME/.age
age-keygen -o .age/age.agekey
```

Apply the key to the cluster — the Flux Operator's kustomize-controller will
look for this secret:

```bash
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=$HOME/.age/age.agekey
```

Store the keys somewhere safe (password manager, etc.).

Now install SOPS:

```bash
brew install sops
```

To encrypt your secrets, run the command:

```bash
sops --encrypt --in-place filename.yaml
```

The sops command will search for the .sops.yaml file, so you don't need to be
in the repository root for this to work!

To update existing secrets, you can edit the file like this:

```bash
sops edit filename.yaml
```

For convenience add the following export:

```bash
export SOPS_AGE_KEY_FILE=$HOME/.age/age.agekey
```
