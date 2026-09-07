# Deployment reply 4 — platform → CTF app (spec 009 / instances)

All five setup items are done, GitOps-managed (ArgoCD app `ctf-instances`), and
verified end-to-end. **You're clear to flip `INSTANCES_ENABLED=true` +
`COOKIE_DOMAIN=ctf-nm.org`.**

## Done + verified
| Item | Status |
|---|---|
| 1. `ctf-instances` ns + ResourceQuota (80 pods, 8/20Gi req, 16/24Gi lim) + LimitRange | ✅ quota active |
| 2. Role/RoleBinding for `ctf/ctf-backend` (pods, services, networkpolicies, ingresses) | ✅ verified: SA **can** create pods/netpols, **cannot** create secrets |
| 3. `ghcr-pull` copied into `ctf-instances` | ✅ (+ see improvement 4) |
| 4. Wildcard TLS `*.ctf-nm.org` in `ctf-instances` | ✅ secret **`ctf-tls`**, cert Ready |
| 5. Standing `ctf-default-deny-all` | ✅ platform-owned (see improvement 2) |

End-to-end proof (as `system:serviceaccount:ctf:ctf-backend`): created a per-instance
NetworkPolicy + a gVisor pod → **kernel:gvisor, DNS:ok, DB egress:BLOCKED**. The
deny-all + your per-instance allow union works exactly as designed.

## Your five questions
1. **gVisor RuntimeClass name: `gvisor`** (handler `runsc`). Your default is correct — no env change.
2. **ingress-nginx** is in namespace **`ingress-nginx`**, and the controller has **no `--watch-namespace`** → it watches all namespaces and will pick up Ingress objects in `ctf-instances`. Your `kubernetes.io/metadata.name: ingress-nginx` selector is right.
3. **CoreDNS is in `kube-system`** (Service `kube-dns`, label `k8s-app=kube-dns`, port 53). Your DNS-egress allowance is correct.
4. **`allow-snippet-annotations` is off (false)** and should stay off. `auth-url` is a first-class annotation, not a snippet, so your flow needs nothing enabled. (See cookie note below.)
5. **Per-instance objects: no changes requested.** non-root + drop-ALL + read-only-root + no SA token + gVisor is a great profile. Two image-compat heads-ups (your side, not blockers): with `readOnlyRootFilesystem`, images that write to `/tmp` need an `emptyDir` there; and gVisor adds ~15–50Mi/sandbox, so the 128Mi request / 256Mi limit may be tight for heavier challenge images (LimitRange `max` is 1Gi, so you can raise per-pod).

## Improvements I made (call-outs)
1. **TLS is a `Certificate`, not a copied secret.** A manual copy of `ctf-tls` would silently expire in ~60 days; cert-manager now issues/renews `*.ctf-nm.org` **in `ctf-instances` directly** → secret `ctf-tls`. No maintenance.
2. **I own `ctf-default-deny-all` (GitOps + self-heal),** so isolation exists independent of app startup. **You can stop asserting it.** If you'd rather keep asserting as belt-and-suspenders, that's safe too — it's byte-identical, so ArgoCD won't fight it. Just don't modify its shape or we'll ping-pong.
3. **Hardened the default SA** (`automountServiceAccountToken: false`) so a pod that forgets can't get a token — defense-in-depth under your per-pod setting.
4. **Attached `ghcr-pull` to the default SA's `imagePullSecrets`,** so instance pods pull private images (e.g. `ctf-demo:v1`) with **no per-pod plumbing**. (If your pod specs already set it, harmless.)

## Cookie-header stripping (your defense-in-depth ask)
I'd **not** do it, and here's the tradeoff: ingress-nginx has no first-class "strip request header to upstream" — it needs a `configuration-snippet`, which means enabling `allow-snippet-annotations` cluster-wide. That flag lets any Ingress annotation inject arbitrary nginx config — a materially bigger attack surface than the risk it closes. Since your ownership check already means a container only ever sees **its own owner's** cookie (not an escalation, as you noted), enabling snippets to strip it is net-negative. Recommendation: leave it. If you still want it, better to strip the cookie **in the app** at the instance proxy layer than to open snippets on the shared controller.

## Go-live
Everything platform-side is ready and proven. Flip `INSTANCES_ENABLED=true` +
`COOKIE_DOMAIN=ctf-nm.org` in `deploy/backend.yaml` whenever you like; push
`ctf-demo:v1` from your CI. Ping me if anything doesn't behave and I'll dig in.
