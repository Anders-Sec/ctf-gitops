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

## Node properties worth knowing before you debug

Two characteristics of `k8s-lab` that have each cost a diagnosis already. Neither is
visible from inside Kubernetes, and both produce symptoms that point elsewhere.

### Guest CPU features depend on the Proxmox CPU *type*

**Current state (fixed 2026-10-19):** the VM passes the host CPU through and reports
*Intel Xeon E5-2680 v3*, with `avx`, `avx2`, `fma` and `bmi2` present. Note
`avx512f` is **absent and always will be** on this hardware — the E5-2680 v3 is
Haswell-era and predates AVX-512. A dependency needing AVX-512 cannot be satisfied
by changing the Proxmox setting.

**What it was, and why it matters:** the VM previously ran Proxmox's default `kvm64`
model, reported as *QEMU Virtual CPU version 2.5+*, which kept `sse4_2`, `popcnt`
and `aes` but masked the whole AVX family. That is not old hardware, it is the model
hiding features the physical CPU has.

The symptom is worth remembering because it is actively misleading. The Kusto
emulator (spec 085) failed `libKusto.NativeInfra.so` init with error `2161770698`
(`0x80DA00CA`), after which Kusto's **own** `FailSlow` crash handler SIGKILLed the
process — surfacing as **exit code 137 and "Killed"**, which reads exactly like an
OOM kill. It was not: there was no cgroup `OOMKilled` reason, no kernel OOM message,
and 87GB free against a 6Gi limit. Check all three before believing exit 137.

Changing the CPU type needs a **VM power cycle**; a guest reboot cannot alter
emulated CPU features.

### Guest disk size is not backed space

The disk was thin-provisioned on a pool shared with another VM. On 2026-10-19 an
image pull of ~1.2GB grew this guest into space promised to that other VM and
**killed both**. The guest reported ~108GB free throughout; nothing in the cluster
could see the real pool. The other VM has been removed, so the pool now backs this
guest, but the property is permanent: `df` inside the guest is an upper bound, not a
guarantee.

That outage also left a **silently corrupt image**: containerd had written a
truncated layer, and because layer content is digest-verified only at download time
and never after extraction, it served a 0-byte `/start-kusto.sh` from a
"successfully pulled" image indefinitely. A disk-full during a pull can leave an
image that looks cached and healthy and is not. Clearing it needs the layer removed
from the content store, not just `crictl rmi`:

```sh
sudo k3s ctr -n k8s.io content rm <layer-digest>
sudo k3s crictl rmi <image-ref>
```

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

### `key-exchange-secret` / `key-exchange-key-hex` — set once, NEVER rotate

Set 2026-10-18 in `ctf-secrets` and `ctf-bld-secrets`. Unlike every other
credential here these are **identical in both environments** on purpose: the
Lantern stage-3 sample is a single build, so a code computed against it must
verify in either environment (app spec 079).

Taken byte-exact from the build tree rather than retyped —
`src/common/buildlib.py` (`FLAG1_VALUE`) and `.work/stage2/stage2.json`
(`stage3_key_hex`). The secret is a UTF-8 string with **no trailing newline and
no surrounding quotes** (it is used as an HMAC key) and the key is 64 lowercase
hex characters verbatim. A stray newline anywhere in the chain breaks every
player's code silently, so verify the byte length the *pod* sees (31 and 64
respectively), not just that the variable is non-empty.

**Never rotate these, and the reason is worse than for `canary-secret`.** The key
is compiled into a shipped sample. Rotating does not merely break attribution —
it makes Chandelier flags 2 and 3 unreachable for every player, silently, with
the sample simply doing nothing. Recovery requires rebuilding and redistributing
the sample, not just resetting a Secret. Exclude from any blanket rotation sweep.

Read only at pod start, so the same delayed-blast-radius property applies: a
mistaken rotation stays invisible until an unrelated roll.

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
