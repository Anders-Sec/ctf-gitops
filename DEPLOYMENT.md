# CTF platform — deployment contract

For the session/team building the CTF app. It tells you **how to ship onto the
shared cluster without needing host or cluster-admin access**. Deployment is via
GitOps (ArgoCD) + GHCR images. You never run privileged `kubectl` on the host.

## Ownership split
- **Platform session (owns):** cluster, ArgoCD, ingress-nginx, cert-manager, the
  `ctf` namespace, Postgres, Redis, `ctf-secrets`, the TLS cert, the
  `ghcr-pull` image-pull secret, and **registering the ArgoCD Application**.
- **You (the app, own):** app source, Dockerfiles, images pushed to GHCR, and the
  app's k8s manifests / Helm chart (Deployments, Services, Ingress) in **your app
  repo** under `deploy/`. You do **not** create the namespace, datastores, or secrets.

To go live you hand the platform session: your **repo URL + deploy path**, the
**image tags**, the **ingress host**, and the **list of secret keys** your app needs
(so they're added to `ctf-secrets`). The platform session registers the ArgoCD app.

## Cluster facts
- Single-node **k3s v1.35** (node `k8s-lab`, internal IP `10.0.0.234`).
- **ingress-nginx**, `ingressClassName: nginx`, NodePorts `30080` (http) / `30443` (https).
- **cert-manager** ClusterIssuers: `letsencrypt-prod` (use this), `letsencrypt-staging`.
- **ArgoCD** deploys from git; auto-sync + self-heal + prune.
- Image registry: **GHCR** `ghcr.io/anders-sec/<image>` (private).

## Namespace & datastores (already running in `ctf`)
| Thing | Value |
|---|---|
| Namespace | `ctf` |
| Postgres host | `postgres.ctf.svc.cluster.local:5432` (Service `postgres`, headless) |
| Postgres db / user | `ctf` / `ctf` |
| Postgres password | Secret `ctf-secrets`, key `postgres-password` |
| Redis host | `redis.ctf.svc.cluster.local:6379` (Service `redis`) |
| Redis password | Secret `ctf-secrets`, key `redis-password` |

- **Postgres is the source of truth.** Run your Alembic migrations + any seed from
  the backend's entrypoint on startup (see conventions below).
- **Redis is ephemeral** (emptyDir) — scoreboard cache / pub-sub / rate limiting only.
  A restart clears it; never store anything you can't rebuild from Postgres.

Compose your connection URLs in app config from the parts above, e.g.
`postgresql+asyncpg://ctf:<postgres-password>@postgres.ctf.svc.cluster.local:5432/ctf`
and `redis://:<redis-password>@redis.ctf.svc.cluster.local:6379/0`.
**Tell the platform session the exact env var names / URL format your app expects**
and it can bake ready-made `DATABASE_URL` / `REDIS_URL` keys into `ctf-secrets`.

## Secrets — `ctf-secrets` (plain k8s Secret in `ctf`)
Managed by the platform session **out-of-band** (never in git — the repo is public).
Consume with explicit `secretKeyRef` (keys are hyphenated, so `envFrom` would skip
them — map them to your own env var names).

Present now: `postgres-password`, `redis-password`, `secret-key`.
Ask the platform session to add what your auth needs, e.g.:
`entra-tenant-id`, `entra-client-id`, `entra-client-secret`,
`smtp-host`, `smtp-user`, `smtp-password`, plus the app callback/base URLs.

```yaml
env:
  - name: SECRET_KEY
    valueFrom: { secretKeyRef: { name: ctf-secrets, key: secret-key } }
  - name: POSTGRES_PASSWORD
    valueFrom: { secretKeyRef: { name: ctf-secrets, key: postgres-password } }
  - name: ENTRA_CLIENT_SECRET
    valueFrom: { secretKeyRef: { name: ctf-secrets, key: entra-client-secret } }
```

## Images (GHCR, private)
- Build + push `ghcr.io/anders-sec/ctf-backend:<tag>` and `…/ctf-frontend:<tag>`
  (pin explicit tags like `v1`, not `latest`).
- The namespace already has an image-pull secret **`ghcr-pull`**. Reference it:
  ```yaml
  imagePullSecrets:
    - name: ghcr-pull
  ```

## Ingress (same-origin) + TLS
Serve the SPA at `/` and the API under `/api` on **one host** so the browser makes
same-origin calls. Host: **`ctf-nm.org`**. TLS secret: **`ctf-tls`** (provisioned by
the platform — see status below). Two ingress objects on the same host:

```yaml
# API: strip the /api prefix before forwarding to the backend
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    # WebSockets for the live scoreboard: keep long-lived upgrades open
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["ctf-nm.org"]
      secretName: ctf-tls
  rules:
    - host: ctf-nm.org
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend: { service: { name: ctf-backend, port: { number: 8000 } } }
---
# Frontend: SPA at root (no rewrite)
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["ctf-nm.org"]
      secretName: ctf-tls
  rules:
    - host: ctf-nm.org
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: ctf-frontend, port: { number: 80 } } }
```

## Backend / frontend conventions (learned the hard way on the other demos)
- **FastAPI + the `/api` strip:** the ingress above forwards `/api/foo` to the
  backend as `/foo`. So either (a) **mount your routers at root** and set
  `--root-path /api` (via a `ROOT_PATH` env your entrypoint passes to uvicorn) so
  docs/redirects stay correct — **recommended**; or (b) keep an in-app `/api`
  prefix and **do not strip** (drop the `rewrite-target`/regex, use `path: /api`
  `pathType: Prefix`). Pick one; don't do both (double prefix → 404s).
- Run uvicorn with `--proxy-headers --forwarded-allow-ips='*'` behind the ingress.
- **Health endpoint** at `/health` (root, not under `/api`) for probes — probes hit
  the pod directly. Give the backend a `startupProbe` generous enough for
  migrations+seed on first boot.
- **Migrations/seed** run from the backend entrypoint (`alembic upgrade head`) so a
  fresh DB self-populates.
- **Frontend must be a production build served by nginx** (multi-stage → `nginx:alpine`
  on port 80). Do **not** ship the Vite dev server — Vite 5/6 rejects unknown Hosts
  behind the proxy. Vite bakes `VITE_*` at build time, so build with the API base set
  to the **same-origin** path (`/api`), e.g. `--build-arg VITE_API_BASE_URL=/api`.
- SPA nginx needs `try_files $uri /index.html;` so client-side routes work.

## Deploy flow
1. Build + push both images to GHCR; pin tags.
2. In **your app repo**, add `deploy/` with the Deployments/Services/Ingress above
   (a small Helm chart or plain manifests), referencing `ghcr-pull`, `ctf-secrets`,
   and the datastore Service DNS names. Make the repo readable by ArgoCD (public, or
   provide a deploy key to the platform session).
3. Hand the platform session: repo URL + `deploy/` path + image tags + ingress host
   + the secret keys you need. It creates an ArgoCD `Application` pointing at your
   `deploy/` path into namespace `ctf`, adds your secret keys to `ctf-secrets`, and
   syncs. Updates later = push a new image tag + bump it in `deploy/`.

## TLS / DNS status (blockers to sort with the platform session)
- The `ctf-tls` cert for `ctf-nm.org` + `*.ctf-nm.org` is defined but **won't issue
  until the Cloudflare API token is refreshed** (the old one expired). The new token
  needs **Zone:DNS:Edit on the ctf-nm.org zone**; the platform session updates the
  `cloudflare-api-token` secret in `cert-manager`.
- **DNS:** create a record for `ctf-nm.org` pointing at whatever fronts the node's
  ingress (NodePort 30080/30443), same as the other sites. cert-manager uses DNS-01
  (a TXT record) so the cert can issue before the A/CNAME exists.

## On-prem / networking
On-prem services and any AI assistant are reached by **direct cluster networking**
(in-cluster Service DNS, or a plain egress route to the on-prem endpoint) — there is
no VPN/ExpressRoute tunnel. If your backend calls an on-prem AI endpoint, reach it
directly by its cluster-routable address; don't assume tunneled connectivity.
