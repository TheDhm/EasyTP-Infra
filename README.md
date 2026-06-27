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

# Django secrets (DB_PASSWORD, SECRET_KEY, WEBHOOK_SECRET, TURNSTILE_SECRET_KEY,
# CLOUDFLARE_TURN_KEY_ID, CLOUDFLARE_TURN_API_TOKEN). The two CLOUDFLARE_TURN_*
# values come from a TURN key created in the Cloudflare Realtime dashboard.
kubectl create secret generic django-secrets \
  --namespace=django-app \
  --from-literal=DB_PASSWORD=<password> \
  --from-literal=SECRET_KEY=<secret-key> \
  --from-literal=WEBHOOK_SECRET=<webhook-secret> \
  --from-literal=TURNSTILE_SECRET_KEY=<turnstile-secret-key> \
  --from-literal=CLOUDFLARE_TURN_KEY_ID=<cf-turn-key-id> \
  --from-literal=CLOUDFLARE_TURN_API_TOKEN=<cf-turn-api-token>

# Grafana admin credentials (monitoring)
kubectl create secret generic grafana-admin-credentials \
  --namespace=monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=<password>
```

## TURN relay for WebRTC apps (Cloudflare Realtime TURN)

WebRTC (Selkies) apps stream media peer-to-peer over UDP, which cannot traverse the
k3s pod NAT — without a TURN relay the video never starts. We use **Cloudflare Realtime
TURN** (managed), not a self-hosted relay, so media rides Cloudflare's network and the
node's public IP is never exposed.

- Create a **TURN key** in the Cloudflare Realtime dashboard and store its
  `KEY_ID` + `API_TOKEN` in `django-secrets` as `CLOUDFLARE_TURN_KEY_ID` /
  `CLOUDFLARE_TURN_API_TOKEN` (see above). Keep the API token server-side only.
- The Django backend (`deploy_app`) mints short-lived credentials per WebRTC app
  deploy and injects `SELKIES_TURN_*` env into the pod; the pod connects **outbound**
  to `turn.cloudflare.com`.
- **No firewall changes and no DNS record** are needed (no inbound relay ports,
  no origin IP exposure). TURN-over-TLS on 443 is available for restrictive client
  networks via `SELKIES_TURN_TLS`/port 443 (set in the backend's deploy env).

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