# ctf-gitops

GitOps for the self-hosted, D&D-themed **Capture The Flag** platform on `ctf.itpluto.dev`.

Stack: React/Vite (frontend) · FastAPI (backend) · PostgreSQL (source of truth) ·
Redis (scoreboard cache / pub-sub / rate limiting) · Entra ID + passwordless
magic-link auth. Runs on the shared cluster (ArgoCD · ingress-nginx · cert-manager
· Vault + External Secrets).

> **Status: infra foundation.** Namespace, datastores, TLS, and Vault→ESO secret
> plumbing. The app (frontend/backend) is added once its images exist.

## Layout
- `infra/namespace.yaml` — `ctf` namespace + `external-secrets-sa` (ESO→Vault auth).
- `infra/secretstore.yaml` — ESO `SecretStore` (Vault k8s auth, role `ctf`).
- `infra/externalsecret.yaml` — syncs `secret/ctf/app` → k8s Secret `ctf-secrets`.
- `infra/postgres.yaml` — dedicated Postgres 16 (password from `ctf-secrets`), 8Gi PVC.
- `infra/redis.yaml` — Redis 7 (password from `ctf-secrets`), ephemeral.
- `infra/cert.yaml` — `ctf.itpluto.dev` cert via `letsencrypt-prod`.
- `infra-app.yaml` — ArgoCD Application syncing `infra/`.

## Bootstrap (once)

### 1. Vault — policy, role, and secret values
Needs a Vault token with admin rights (run from a shell with `VAULT_ADDR`/`VAULT_TOKEN`,
or `kubectl exec -n vault vault-0 -- vault ...`). The KV mount (`secret/`, v2) and the
`kubernetes` auth backend already exist.

```sh
# Policy: read the CTF secrets
vault policy write ctf-read - <<'EOF'
path "secret/data/ctf/*"     { capabilities = ["read"] }
path "secret/metadata/ctf/*" { capabilities = ["read", "list"] }
EOF

# k8s auth role: bind the ctf namespace's external-secrets-sa to that policy
vault write auth/kubernetes/role/ctf \
  bound_service_account_names=external-secrets-sa \
  bound_service_account_namespaces=ctf \
  token_policies=ctf-read \
  token_ttl=1h

# Secret values (replace the placeholders with real creds; generate strong
# passwords for postgres/redis). Keys become entries in the ctf-secrets Secret.
vault kv put secret/ctf/app \
  postgres-password="$(openssl rand -base64 24)" \
  redis-password="$(openssl rand -base64 24)" \
  secret-key="$(openssl rand -base64 48)" \
  entra-tenant-id="<tenant-id>" \
  entra-client-id="<client-id>" \
  entra-client-secret="<client-secret>" \
  smtp-host="<smtp host>" \
  smtp-user="<smtp user>" \
  smtp-password="<smtp password>"
```

### 2. Deploy the infra
```sh
kubectl apply -f infra-app.yaml   # into the existing ArgoCD
```
ESO creates `ctf-secrets`, then Postgres + Redis start and the cert issues.

### 3. DNS
`ctf.itpluto.dev` resolves via the existing `*.itpluto.dev` record. Confirm once the
app's ingress exists.

## Add the app (later)
Build + push the frontend/backend images, add the app chart + ingress
(`ctf.itpluto.dev`, same-origin `/api`, websocket-friendly for live scoreboard),
consuming `ctf-secrets` (`DATABASE_URL`/`REDIS_URL`/auth) via `envFrom`.

## Notes
- **On-prem/hybrid:** on-prem services and the AI assistant are reached by direct
  cluster networking (in-cluster Service DNS or a plain egress route) — no VPN/
  ExpressRoute tunneling.
- The Vault policy/role above are currently set out-of-band; codify them in the
  platform Terraform when convenient so they're reproducible.
