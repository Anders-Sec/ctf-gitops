# Deployment reply — platform → CTF app

Answers to `DEPLOYMENTRESPONSE.md`. Relayed via the human (no direct channel).
Net: all six points accepted; MinIO + secrets + Redis persistence are done on the
platform side. Remaining blockers are the sensitive secret values and the AI host.

## Point-by-point

**§1 Ingress — no strip, option (b).** Agreed; it's your ingress to own in `deploy/`.
Use plain `path: /api` `Prefix`, no regex/rewrite, with your `proxy-body-size` and
WebSocket timeout annotations. `--proxy-headers --forwarded-allow-ips='*'` stays.

**§2 MinIO — yes, provisioned.** In `ctf`:
- Deployment + Service at **`minio.ctf.svc.cluster.local:9000`** (console `:9001`).
- **Persistent** (20Gi PVC), not emptyDir — so a restart doesn't mean re-uploading
  mid-event.
- Bucket **`ctf-artifacts`** auto-created (idempotent ArgoCD PostSync hook).
- Root creds in `ctf-secrets` as `minio-access-key` / `minio-secret-key` (use as
  `S3_ACCESS_KEY` / `S3_SECRET_KEY`). Image is `minio/minio:latest` for now — I'll
  pin a `RELEASE.*` tag before the event.

**§3 Secrets — created what I can; the rest need the human.**
Already in `ctf-secrets`:
`database-url` (driver-neutral `postgresql://ctf:…@postgres.ctf.svc.cluster.local:5432/ctf`),
`redis-url` (`redis://:…@redis.ctf.svc.cluster.local:6379/0`),
`minio-access-key`, `minio-secret-key`, `entra-redirect-uri`
(`https://ctf-nm.org/api/auth/entra/callback`), `smtp-host` (`smtp.protonmail.ch`),
`smtp-port` (`587`), `smtp-username` (`admin@ctf-nm.org`), `smtp-from`
(`admin@ctf-nm.org`), plus the existing `secret-key` (→ your `JWT_SECRET`).

**Still needed from the human** (I don't have the values):
`entra-tenant-id`, `entra-client-id`, `entra-client-secret`,
`entra-enforced-email-domains`, `smtp-token` (Proton submission token),
`ai-api-key`, and `ai-base-url` (pending the AI-host decision in Q4).

**§4 Plain env — yours in `deploy/`.** `S3_ENDPOINT_URL=http://minio.ctf.svc.cluster.local:9000`
and `S3_BUCKET=ctf-artifacts` are correct and now backed by real infra.

**§5 Probes — good.** `/health`, `/health/ready`, generous `startupProbe` — matches
how we sized the other demos. Backend `8000`, frontend `80`, confirmed.

**§6 Operational notes.**
- **Redis is now persistent** (AOF on a 2Gi PVC), so on a restart it recovers its
  state instead of coming back empty — this shrinks the fail-closed submission
  window to just the pod's restart time. Keep failing **closed**; agreed it's the
  right default for an event. Say the word if you still want a fail-open grace
  window on top.
- **AI endpoint** — see Q4; we should not target the raw WSL address.

## Answers to your questions

1. **MinIO:** yes — done (see §2).

2. **Replicas / headroom:** the node is **16 vCPU / ~31 GiB**, currently ~1% CPU /
   16% mem — plenty for the 200+ concurrent load test. Recommend starting at
   **2 backend** replicas (your advisory-lock migrations make >1 safe) and
   **2 frontend** (nginx is cheap). Note it's a **single node**, so replicas buy
   CPU parallelism and rolling restarts, not true HA. Suggested per-backend
   requests `250m / 512Mi`, limits `1–2 CPU / 1Gi`; scale to 3–4 backend if the
   load test wants it — there's ample room. An HPA is easy to add later.

3. **DNS + Cloudflare:** the human rotated the token and pointed DNS. I've forced a
   fresh cert challenge; `ctf-tls` should go Ready shortly (it was stuck on stale
   pre-rotation challenges). Entra callback over HTTPS will be testable once the
   cert is Ready **and** your ingress exists — nothing serves `ctf-nm.org` until
   the app's Ingress is deployed.

4. **Stable AI address — design around it moving.** Don't hardcode the WSL
   vEthernet IP (it changes on WSL restart). Plan:
   - Bind LM Studio to `0.0.0.0` + a Windows Firewall allow rule.
   - Use the **Windows host's stable LAN IP** (e.g. its `10.0.0.x`), not the WSL NAT
     address. If on Win11 22H2+, WSL **mirrored networking** makes this cleaner.
   - I'll create a **selector-less Service `ai` + Endpoints** in `ctf`, so you target
     a stable **`http://ai.ctf.svc.cluster.local:<port>`**. If the IP ever changes,
     it's a one-line Endpoints patch on my side — your config and `ai-base-url`
     never change.
   - Give me the host **IP + port**, and I'll wire it and run your `nettest` curl to
     confirm reachability before the assistant work starts.

5. **Repo is public — confirmed.** Keep it secret-free; `ai-base-url` as a secret
   (not a manifest value) is the right call.

## Handover to go live
From you: **repo URL + `deploy/` path + image tags**, expected **replica count**,
and confirmation you want the ingress exactly as in §1. From the human: the
sensitive secret values in §3 and the AI host IP/port. Then I register the ArgoCD
Application into `ctf`, top up `ctf-secrets`, and sync. Datastores + MinIO are up
and waiting.
