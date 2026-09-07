# Deployment reply 3 — platform → CTF app

Backend is live. Two things on your side: a frontend crash and an image-tag
convention.

## Backend: up
`ctf-backend` 2/2 Running, migrations applied, `/health` + `/health/ready` → 200.
Datastores, MinIO, secrets, cert, ArgoCD all green. Once the two items below are
fixed the site is fully up.

## 1. Frontend crashes — nginx can't chown under `drop: ["ALL"]`
The frontend container sets `securityContext.capabilities.drop: ["ALL"]`, but the
stock `nginx` image starts as root and chowns its cache dirs at boot:
```
nginx: [emerg] chown("/var/cache/nginx/client_temp", 101) failed (Operation not permitted)
```
`CAP_CHOWN` is dropped, so it exits → CrashLoopBackOff.

**Recommended fix (you offered it): switch to `nginxinc/nginx-unprivileged`.**
It runs as uid 101, listens on **8080**, and does no startup chown — so it works
with `drop: ["ALL"]` and needs no added caps.
- Dockerfile: base `nginxinc/nginx-unprivileged:alpine`; put the SPA config under
  `/etc/nginx/conf.d/` and `listen 8080;`.
- `deploy/frontend.yaml`: `containerPort: 8080`; Service `targetPort: 8080`
  (Service `port` can stay 80); point the frontend Ingress at the Service port.

**Alternative (manifest-only, less clean):** keep stock nginx but add back the caps
it needs — `capabilities: { drop: ["ALL"], add: ["CHOWN","SETUID","SETGID","NET_BIND_SERVICE"] }`.
Weakens the hardening you intended; the unprivileged image is the better path.

## 2. Image tags — build pushes `sha-<gitsha>`, deploy references `:v1`
Your build pushed `ghcr.io/anders-sec/ctf-backend:sha-c1a6893` (and frontend), but
`deploy/` references `:v1` — that mismatch was the 404/ImagePullBackOff. To unblock
I **server-side retagged `v1` → `sha-c1a6893`** for both images, so `:v1` resolves
now. But that's a manual alias — your next build pushes a new sha and `:v1` will
still point at the old one.

Please standardize one of:
- have the build also push/move a `:v1` (or `:latest`-style) tag, **or**
- reference the immutable `sha-<gitsha>` in `deploy/` and bump it per release
  (cleaner GitOps — no moving tags).

Either way, drop the mismatch so deploys aren't chasing a tag that doesn't exist.

## Still open (unchanged)
Sensitive Entra keys (tenant/client id + secret) are blank — fine, Entra login
stays disabled until set. AI host IP:port still needed to wire the `ai` Service.
