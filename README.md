# ctf-gitops

GitOps for the self-hosted, D&D-themed **Capture The Flag** platform on `ctf-nm.org`.

> **Deploying the app?** The contract for the app team (datastores, secrets,
> images, ingress, conventions) is agreed with the app side as
> `BuildEnvRequest.md` in the cross-session share; the last reply recorded in
> this repo is [DEPLOYMENT-REPLY-4.md](DEPLOYMENT-REPLY-4.md).

Stack: React/Vite (frontend) · FastAPI (backend) · PostgreSQL (source of truth) ·
Redis (scoreboard cache / pub-sub / rate limiting) · Entra ID + passwordless
magic-link auth. Runs on the shared cluster (ArgoCD · ingress-nginx · cert-manager).

> **Status: infra foundation.** Namespace, datastores, and TLS. The app
> (frontend/backend) is added once its images exist.

## Secrets — plain Kubernetes Secret (`ctf-secrets`)

The cluster's Vault runs in **dev mode (in-memory)**, so it is not a durable
secret store. CTF secrets therefore live in a native Secret named `ctf-secrets`
in the `ctf` namespace, created **out-of-band** (never committed to git). Postgres,
Redis, and the backend consume it via `secretKeyRef` / `envFrom`.

Create it once (datastore passwords are generated; add the app-auth keys when the
app is deployed):

```sh
kubectl create namespace ctf --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic ctf-secrets -n ctf \
  --from-literal=postgres-password="$(openssl rand -hex 24)" \
  --from-literal=redis-password="$(openssl rand -hex 24)" \
  --from-literal=secret-key="$(openssl rand -base64 48)"

# Later, when wiring auth, add the real values (edit or recreate the Secret):
#   entra-tenant-id, entra-client-id, entra-client-secret
#   smtp-host, smtp-user, smtp-password
```

The Secret is untracked by ArgoCD (not in the repo), so sync/prune leaves it alone.

## Layout
- `infra/namespace.yaml` — `ctf` namespace.
- `infra/postgres.yaml` — dedicated Postgres 16 (password from `ctf-secrets`), 8Gi PVC.
- `infra/redis.yaml` — Redis 7 (password from `ctf-secrets`), **persistent**
  (AOF on a 2Gi PVC). Not ephemeral: the app's rate limiter fails CLOSED, so a
  Redis restart takes flag submission offline until it returns, and persistence
  shrinks that window on this single node.
- `infra/cert.yaml` — `ctf-nm.org` (+ wildcard) cert via `letsencrypt-prod`.
- `infra/minio.yaml` — S3-compatible artifact store. Runs **SeaweedFS** behind a
  Service still named `minio` on :9000, so the app's `S3_ENDPOINT_URL` is
  unchanged. MinIO's own images were withdrawn from Docker Hub (both
  `minio/minio` and `minio/mc` 404), so a pull credential would not fix it.
- `instances/` — one-time `ctf-instances` setup: namespace, quota/limits, RBAC,
  standing default-deny, wildcard cert. Instance objects themselves are created
  by the app at runtime.
- `bld/` — the **bld** rehearsal environment in namespace `ctf-bld`: namespace,
  quota/limits, its own Postgres + Redis, wildcard-only cert, NetworkPolicies.
  Reached only via a local hosts-file entry for `bld.ctf-nm.org` → `10.0.0.234`;
  no public DNS record. No artifact store and instances are off by choice.
- `app.yaml` / `infra-app.yaml` / `instances-app.yaml` / `bld-infra-app.yaml` /
  `bld-app.yaml` — the ArgoCD Applications.
- `infra-app.yaml` — ArgoCD Application syncing `infra/`.

## Bootstrap (once)
```sh
# 1. namespace + ctf-secrets (see "Secrets" above)
# 2. deploy the infra
kubectl apply -f infra-app.yaml   # into the existing ArgoCD
```
ArgoCD brings up Postgres + Redis and issues the cert for `ctf-nm.org` (apex +
wildcard), so subdomains are covered without a per-host certificate.

## Add the app (later)
Build + push the frontend/backend images, add the app chart + ingress
(`ctf-nm.org`, same-origin `/api`, websocket-friendly for the live scoreboard),
consuming `ctf-secrets` (`DATABASE_URL`/`REDIS_URL`/auth) via `envFrom`.

## Notes
- **On-prem/hybrid:** on-prem services and the AI assistant are reached by direct
  cluster networking (in-cluster Service DNS or a plain egress route) — no VPN/
  ExpressRoute tunneling.
- If Vault is later moved to a persistent/HA deployment, these secrets can migrate
  to Vault + External Secrets.
