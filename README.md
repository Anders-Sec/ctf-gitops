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

## Credential runbook

Both credentials are plain out-of-band Secrets, created by hand and deliberately
untracked here (same model as `ctf-secrets`), so ArgoCD sync/prune leaves them
alone. Values live only in the cluster; record your own copies somewhere you
control.

| Credential | Where | Type / scope | Created | Expires |
|---|---|---|---|---|
| `ghcr-pull` | Secrets in ns `ctf`, `ctf-bld`, `ctf-instances` | GitHub **classic** PAT, `read:packages` only | 2026-10-16 | **2027-01-14** |
| `repo-anders-sec-ctf` | Secret in ns `argocd`, label `argocd.argoproj.io/secret-type: repository` | GitHub **fine-grained** PAT, `Contents: Read-only`, `Anders-Sec/ctf` only | 2026-10-16 | **2027-01-14** |

Both were issued with a 90-day lifetime. The build request requires expiry to
clear event-end by at least 30 days, so **the event must end on or before 2026-12-15**
for these to hold. If it runs later, reissue both with a longer lifetime before
the event rather than during it.

### `canary-secret` — set once, NEVER rotate during an event

`canary-secret` is the HMAC key behind every canary token (spec 075). Set
2026-10-16 in `ctf-secrets` and `ctf-bld-secrets`, independently generated per
environment (64 bytes each, `openssl rand -base64 48`) so a bld-issued token
cannot resolve against prod or vice versa.

**Do not rotate it while an event is running, and do not rotate it while any
issued sample is still outstanding.** Every already-downloaded malware sample
carries a token HMAC'd with this key. Rotating makes all outstanding tokens
unresolvable, so a callback can no longer be attributed to the player it came
from — which is the entire point of the feature. There is no migration path:
the old tokens simply stop verifying.

Consequences for routine work:
- Never include it in a "rotate all secrets" sweep.
- `kubectl patch` on these Secrets must not blind-overwrite this key; check it is
  absent before setting it (it was absent when first set, so nothing was lost).
- The backend reads it via `secretKeyRef` at pod start, so a change only takes
  effect on a pod roll — which means a mistaken rotation can sit unnoticed until
  the next restart, then break attribution for every sample issued before it.

**Rotating `ghcr-pull`: all THREE namespaces must be updated together** — `ctf`,
`ctf-bld` and `ctf-instances`. A stale copy in any one of them is invisible while
the GHCR packages are public, because anonymous pulls succeed and mask it. It only
surfaces the moment the packages go private, as:

```
failed to authorize: failed to fetch oauth token: 403 Forbidden
```

This happened on 2026-10-16: `ctf-instances` still held a credential from
2026-09-07 while `ctf` and `ctf-bld` had the new token, so the instant the packages
went private, challenge images stopped pulling in `ctf-instances` — meaning
challenge instances could not launch at all, while the app itself looked perfectly
healthy. Fixed by copying the working Secret across. After any rotation, verify a
real pull in **each** namespace rather than assuming.

Note the challenge images are tagged `sha-<gitsha>`; there is no `v1` tag despite
what older reply documents in this repo suggest.

Verified at creation: the `read:packages` token authenticates to GHCR and pulls
`ctf-backend` and `ctf-frontend` in both `ctf` and `ctf-bld`; the fine-grained
PAT clones `Anders-Sec/ctf` over HTTPS and is confirmed **read-only** — a
dry-run push is rejected with `403 Permission to Anders-Sec/ctf.git denied`.

## Notes
- **On-prem/hybrid:** on-prem services and the AI assistant are reached by direct
  cluster networking (in-cluster Service DNS or a plain egress route) — no VPN/
  ExpressRoute tunneling.
- If Vault is later moved to a persistent/HA deployment, these secrets can migrate
  to Vault + External Secrets.
