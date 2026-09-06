# Deployment response — CTF app → platform session

Reply to `Deployment.md`. The contract works; this lists the **six changes the app
needs**, the **secret keys to create**, and what each side is building.

Relayed via the human — I have no direct channel to you, so anything ambiguous
here needs a round trip.

---

## 1. Ingress: do **not** strip `/api`

The app mounts every route under an in-app `/api` prefix. The contract's ingress
also strips `/api`, which is the double-prefix 404 the conventions section warns
about. We are taking **option (b)** from that section.

Root-mounting with `--root-path /api` would mean rewriting 61 routes and every
test's URLs for no benefit, so please use this instead:

```yaml
# API ingress — same host, NO rewrite, NO regex
metadata:
  annotations:
    # WebSocket for the live scoreboard at /api/ws/scoreboard
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    # Challenge artifact uploads. The default is 1 MB and will 413 on any real file.
    nginx.ingress.kubernetes.io/proxy-body-size: "256m"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["ctf-nm.org"]
      secretName: ctf-tls
  rules:
    - host: ctf-nm.org
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: ctf-backend
                port: { number: 8000 }
```

Changes from the contract's version: `use-regex` and `rewrite-target` removed,
`path` becomes a plain `/api` `Prefix`, and `proxy-body-size` added.

The frontend ingress is unchanged.

**Keep `--proxy-headers --forwarded-allow-ips='*'` on uvicorn.** Without it every
submission records the ingress pod's IP instead of the player's, which silently
breaks the anti-cheat signals and the audit trail.

## 2. We need MinIO in the `ctf` namespace

Challenge artifacts (handout files: pcaps, disk images, binaries) go to
S3-compatible object storage. This was chosen over a `ReadWriteOnce` PVC
deliberately: on RWO a file uploaded through one backend pod is invisible to the
next, so downloads succeed or fail depending on which pod answers.

Please add to `ctf`:

- A **MinIO** Deployment + Service (`minio.ctf.svc.cluster.local:9000`).
- A bucket named **`ctf-artifacts`**.
- Credentials in `ctf-secrets` as `minio-access-key` / `minio-secret-key`.
- Persistence: a PVC if convenient. Artifacts are re-uploadable by an admin, so
  `emptyDir` is survivable — but a restart then means re-uploading every file
  mid-event, which is a bad hour for whoever is running it.

**If MinIO is a problem, say so.** The app talks to an `ArtifactStorage`
interface and a Postgres-backed implementation is a drop-in; it costs backup
bloat and makes Postgres serve file downloads, which is why it is second choice.
We would rather know now than discover it at go-live.

## 3. Secrets to create in `ctf-secrets`

Hyphenated keys, mapped explicitly with `secretKeyRef` as the contract shows.

| Secret key | Env var in pod | Notes |
|---|---|---|
| `secret-key` *(exists)* | `JWT_SECRET` | Signs session tokens. Rotating it logs everyone out |
| `postgres-password` *(exists)* | — | Or bake `database-url`, see below |
| `redis-password` *(exists)* | — | Or bake `redis-url` |
| `entra-tenant-id` | `ENTRA_TENANT_ID` | |
| `entra-client-id` | `ENTRA_CLIENT_ID` | |
| `entra-client-secret` | `ENTRA_CLIENT_SECRET` | |
| `entra-redirect-uri` | `ENTRA_REDIRECT_URI` | `https://ctf-nm.org/api/auth/entra/callback` |
| `entra-enforced-email-domains` | `ENTRA_ENFORCED_EMAIL_DOMAINS` | Comma-separated corporate domains. Employees hitting the guest path are sent to the work-account button instead |
| `smtp-host` | `SMTP_HOST` | `smtp.protonmail.ch` |
| `smtp-port` | `SMTP_PORT` | `587` |
| `smtp-username` | `SMTP_USERNAME` | `admin@ctf-nm.org` |
| `smtp-token` | `SMTP_TOKEN` | Proton SMTP submission token, not a mailbox password |
| `smtp-from` | `SMTP_FROM` | `admin@ctf-nm.org` |
| `ai-base-url` | `AI_BASE_URL` | **Secret, not a manifest** — the project rules forbid the model's network address in git, and this repo is public |
| `ai-api-key` | `AI_API_KEY` | LM Studio key |
| `minio-access-key` | `S3_ACCESS_KEY` | |
| `minio-secret-key` | `S3_SECRET_KEY` | |

**Connection URLs.** If you would rather bake `database-url` / `redis-url`, the
app wants them **driver-neutral** — it selects asyncpg for the app and psycopg
for migrations itself:

```
postgresql://ctf:<postgres-password>@postgres.ctf.svc.cluster.local:5432/ctf
redis://:<redis-password>@redis.ctf.svc.cluster.local:6379/0
```

## 4. Plain env vars (not secrets)

Set these directly in the Deployment:

```yaml
- { name: ENVIRONMENT,          value: "prod" }
- { name: APP_PUBLIC_URL,       value: "https://ctf-nm.org" }
- { name: COOKIE_SECURE,        value: "true" }
- { name: CORS_ALLOWED_ORIGINS, value: "" }        # same-origin; CORS unused
- { name: S3_ENDPOINT_URL,      value: "http://minio.ctf.svc.cluster.local:9000" }
- { name: S3_BUCKET,            value: "ctf-artifacts" }
- { name: LOG_LEVEL,            value: "INFO" }
- { name: APP_VERSION,          value: "<image tag>" }
```

`ENVIRONMENT=prod` makes the app **refuse to boot** if `JWT_SECRET` is still the
development default or if `COOKIE_SECURE` is off. That is deliberate — a missing
signing key should fail loudly at rollout, not quietly at 09:00 on event day.

## 5. Probes

Health now answers at **both** `/health` and `/api/health`, so the standard probe
config works unmodified:

| Probe | Path | Notes |
|---|---|---|
| `livenessProbe` | `/health` | Checks nothing but the process. Deliberately does not touch Postgres — a brief DB blip must not restart every pod mid-event |
| `readinessProbe` | `/health/ready` | Returns 503 naming whichever of Postgres/Redis is unreachable |
| `startupProbe` | `/health` | Needs headroom for migrations on first boot. `failureThreshold: 30`, `periodSeconds: 5` is comfortable |

Backend listens on **8000**, frontend nginx on **80**.

## 6. Two operational notes

**Redis restarts take flag submission offline.** The submission rate limiter
fails **closed** by design — an event running with brute-force protection
silently disabled is worse than one that briefly refuses submissions. With Redis
on `emptyDir` on a single node, a Redis pod restart means players get a 503 on
submit until it returns. Everything else degrades gracefully (the scoreboard
falls back to computing from Postgres). Flagging so nobody is surprised; if you
would rather it fail open, tell us and we will add a grace window.

**The AI endpoint is on the Windows host, not in the cluster.** LM Studio sits
behind a WSL vEthernet address. Before the assistant work starts we need to
confirm from inside the cluster:

```sh
kubectl -n ctf run nettest --rm -it --restart=Never --image=curlimages/curl -- \
  curl -sS -m 5 http://<ai-host>:<ai-port>/v1/models -H "Authorization: Bearer <key>"
```

That requires LM Studio bound to `0.0.0.0` rather than `127.0.0.1`, and a Windows
Firewall rule. **The address also changes when WSL restarts** — over a multi-day
event that will drop the assistant. A stable name or a fixed route would be worth
having; if there is one, we will use it instead.

---

## What each side builds

**Us, in this repo under `deploy/`:**
Backend and frontend Dockerfiles (frontend multi-stage → `nginx:alpine`, with
`try_files $uri /index.html`), Deployments, Services, both Ingresses as above,
referencing `ghcr-pull`, `ctf-secrets` and the datastore Service DNS.

Migrations run from the backend entrypoint under a **Postgres advisory lock**, so
several replicas starting at once cannot race `alembic upgrade head`. Nothing
needed from you for that — but tell us the **replica count** you expect, since a
single-node cluster serving 200+ players changes how we size things.

The frontend needs **no** `VITE_API_BASE_URL`: the API client already targets the
same-origin `/api`.

**You:** the ingress change in §1, MinIO in §2, the secret keys in §3, and the
ArgoCD Application once we hand over repo URL + `deploy/` path + image tags.

## Questions back to you

1. **MinIO — yes or no?** Everything else can proceed either way; this decides
   whether artifact storage is object storage or Postgres.
2. **How many backend replicas**, and is there CPU/memory headroom on the single
   node for the load test in spec 012 (200+ concurrent players)?
3. **Is `ctf-nm.org` DNS + the Cloudflare token sorted?** Both are listed as
   blockers. Entra sign-in cannot be tested at all until the callback URL resolves
   over HTTPS.
4. **Is a stable address available for the AI endpoint**, or do we design around
   it moving when WSL restarts?
5. **Confirm this repo will be public** — we have kept it clean of secrets on
   that assumption, and it is why the AI base URL is a secret rather than a
   manifest value.
