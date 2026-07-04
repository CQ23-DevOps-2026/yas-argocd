# YAS ArgoCD GitOps

This repository stores the desired deployment state for YAS applications.

## Environments

- `yas-dev`: continuously updated from the `main` branch image tags.
- `yas-staging`: updated only when a release tag such as `v1.2.3` is promoted.

## Bootstrap

Apply the root ArgoCD applications:

```sh
kubectl apply -f infrastructure/argocd/yas-dev.yaml
kubectl apply -f infrastructure/argocd/yas-staging.yaml
```

Before applying, replace `REPLACE_WITH_YAS_ARGOCD_REPO_URL` in the ArgoCD manifests with this repository URL.

## Layout

```text
charts/                  Helm charts copied from the source repository
environments/dev/        Dev namespace and generated ArgoCD applications
environments/staging/    Staging namespace and generated ArgoCD applications
infrastructure/argocd/   Root app-of-apps entry points
```

Runtime infrastructure such as PostgreSQL, Kafka, Redis, Keycloak, and observability is intentionally not managed here yet because it is already provisioned separately.
