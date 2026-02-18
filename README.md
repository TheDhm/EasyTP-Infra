# EasyTP Infrastructure

GitOps repository for EasyTP Kubernetes infrastructure managed by Flux CD.

## Structure

```
├── clusters/production/     # Cluster-level Kustomizations
├── infrastructure/          # Namespaces, cert-manager, monitoring
├── apps/                    # Application deployments
│   ├── django-app/          # Django backend + PostgreSQL + Redis + Nginx
│   └── registry/            # Self-hosted Docker registry
└── image-automation/        # Flux image update automation
```

## Related Repositories

- **Frontend**: [EasyTP-Frontend](https://github.com/TheDhm/EasyTP-Frontend)
- **Backend**: [EasyTP-Backend](https://github.com/TheDhm/EasyTP-Backend)

## Secrets (Manual)

These secrets must be created manually (not stored in git):

```bash
# Registry credentials for Flux image scanning
kubectl create secret docker-registry registry-credentials \
  --namespace=flux-system \
  --docker-server=registry.melekabderrahmane.com \
  --docker-username=<username> \
  --docker-password=<password>

# Registry pull secret for django-app
kubectl create secret docker-registry registry-pull-secret \
  --namespace=django-app \
  --docker-server=registry.melekabderrahmane.com \
  --docker-username=<username> \
  --docker-password=<password>

# Django secrets (DB_PASSWORD, SECRET_KEY, WEBHOOK_SECRET, TURNSTILE_SECRET_KEY)
kubectl create secret generic django-secrets \
  --namespace=django-app \
  --from-literal=DB_PASSWORD=<password> \
  --from-literal=SECRET_KEY=<secret-key> \
  --from-literal=WEBHOOK_SECRET=<webhook-secret> \
  --from-literal=TURNSTILE_SECRET_KEY=<turnstile-secret-key>

# Grafana admin credentials (monitoring)
kubectl create secret generic grafana-admin-credentials \
  --namespace=monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=<password>
```

## Bootstrap

```bash
export GITHUB_TOKEN=<your-pat>

flux bootstrap github \
  --owner=<your-github-username> \
  --repository=EasyTP-Infra \
  --branch=main \
  --path=./clusters/production \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller
```