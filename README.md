# argocd-ops

This repository contains documentation and configs for setting up ArgoCD

## Install ArgoCD

### Prepare sops age key (informative)

Note: For simplicity, `code-monkey-local-keys.txt` is pre-generated. Following steps are just
informative and guidelines for generating keys for other environments.

WARNING: Code-monkey-local-keys.txt is publicly known, for the purpose of demonstration.
Please do not use it for protected environments.

In admin workstation, install age and create key pair:

```bash
mkdir -p secret
# Generate key pair
age-keygen -o secret/code-monkey-local-keys.txt

# Output similar to:
#  # created: 2025-03-24T10:17:48+07:00
#  # public key: age13xcxxcns...
#  AGE-SECRET-KEY-1FMVW0ZNFJ9ZJ...

cp ~/.config/sops/age/keys.txt ~/.config/sops/age/keys.txt.old
cat secret/code-monkey-local-keys.txt >> ~/.config/sops/age/keys.txt

AGE_PUBLIC_KEY=$(grep "public key" secret/code-monkey-local-keys.txt | cut -d ":" -f2 | tr -d " ")
echo "creation_rules:" > local.sops.yaml
echo "  - age: >-" >> local.sops.yaml
echo "     $AGE_PUBLIC_KEY" >> local.sops.yaml

# Create local-secret.values.dec.yaml
AGE_SECRET_KEY=$(cat secret/code-monkey-local-keys.txt | grep AGE-SECRET-KEY)
echo "secrets:" > helm-chart/local-secret.values.dec.yaml
echo "  data:" >> helm-chart/local-secret.values.dec.yaml
echo "    - name: helm-secrets-private-keys" >> helm-chart/local-secret.values.dec.yaml
echo "      namespace: argocd" >> helm-chart/local-secret.values.dec.yaml
echo "      type: Opaque" >> helm-chart/local-secret.values.dec.yaml
echo "      data:" >> helm-chart/local-secret.values.dec.yaml
echo "        keys.txt: |" >> helm-chart/local-secret.values.dec.yaml
AGE_SECRET_KEY_BASE64=$(echo $AGE_SECRET_KEY | base64 -w0)
echo "          $AGE_SECRET_KEY_BASE64" >> helm-chart/local-secret.values.dec.yaml

# Encrypt local-secret.values.dec.yaml
sops --config local.sops.yaml -e helm-chart/local-secret.values.dec.yaml > helm-chart/local-secret.values.yaml
```

In the end, following files are created in admin workstation:
- `secret/keys.txt`: Can be deleted afterward, or should be kept in secured place.
- `~/.config/sops/age/keys.txt`: For editing encrypted files later.
- `local.sops.yaml`: Environment specific sops config file, specifying which public key (recipient)
  should be used for encryption.
- Following files SHOULD NOT be tracked to git because they are specific for a local environment,
  and multiple developers SHOULD have different local environments.
  + `local.sops.yaml`
  + `local-secret.values.yaml`
  + `local-secret.values.dec.yaml`
- However, encrypted files for integration environments SHOULD be tracked, for example:
  + `qa.sops.yaml`
  + `qa-secret.values.yaml`

### Test helm secrets template

In admin workstation:

```yaml
## Prepared keys in: ~/.config/sops/age/keys.txt

cd sample-app

## *.sops.yaml specify which recipients (public keys) to use
sops --config ../local.sops.yaml -e local-secret.values.dec.yaml > local-secret.values.yaml
sops -d local-secret.values.yaml

helm secrets template . -f local-secret.values.yaml
```

Note: Do not commit actual plain secret files to Git

### Install ArgoCD helm chart

Disable ingress certificates for first installation, as cert-manager CRDs are not installed yet.

```yaml
# values.yaml
argo-cd:
  server:
    certificate:
      enabled: false
```

After cert-manager is installed, we can come back and upgrade argocd with certificate enabled.

```bash
kubectl create namespace argocd

cd helm-chart

## Test the template
helm dependency build
helm secrets template -n argocd argocd . -f values.yaml -f {env}.values.yaml -f {env}-secret.values.yaml

## If helm does not recognize kubectl config
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

## With installation of plugins, argocd-repo-server could take several minutes to initialize
## and probably restarted several times; be patient!
helm secrets upgrade -i -n argocd argocd . -f values.yaml -f {env}.values.yaml -f {env}-secret.values.yaml
## Watch initialization
## kubectl get pod -n argocd -w
```

Helm will call helm-secrets because it is registered as downloader plugin.

You can find the password by running:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Add new ArgoCD users (informative)

- Update `argocd-rbac-cm.yml` for required roles and groups
- Update `argocd-cm.yml` (argo-cd chart) to add `accounts.<username>` entries
- Apply the changes
- Set user password:

  ```bash
  argocd login <server_ip>:<port>
  # Example
  # argocd login argocd.localhost:443

  argocd account list
  # if you are managing users as the admin user,
  # <current-user-password> should be the current admin password.
  argocd account update-password \
    --account <name> \
    --current-password <current-user-password> \
    --new-password <new-user-password>
  ```

## References

- https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd
- https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/
- https://github.com/getsops/sops
- https://github.com/jkroepke/helm-secrets
- https://github.com/jkroepke/helm-secrets/wiki/ArgoCD-Integration
- https://github.com/argoproj/argo-helm/tree/main/charts/argocd-apps
