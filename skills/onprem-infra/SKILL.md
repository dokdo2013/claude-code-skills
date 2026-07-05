---
name: onprem-infra
description: "Use when touching the dylabs on-prem k3s cluster — the `dylabs-onprem` kube context, the KoreaIDC bare-metal node, `*.int.sbalyd.com`, the `onprem` Ansible repo, `meloming-qa` on-prem, migrating a QA/internal service off EKS, OR serving an external-facing PROD app from on-prem via Cloudflare Tunnel (cloudflared, cfargotunnel). Especially when about to `kubectl apply`/`rollout restart`/`helm` against onprem, edit `roles/k3s_platform`, add LimitRanges/resources, add a public hostname to the cloudflared tunnel, or ask 'how do I apply this to onprem'. Triggers: onprem, dylabs-onprem, k3s, int.sbalyd.com, KoreaIDC, cloudflared, Cloudflare Tunnel, cfargotunnel, 온프렘, 온박스, ansible-playbook."
---

# on-prem k3s infra (dylabs)

The dylabs on-prem cluster is a **single KoreaIDC bare-metal node** running **k3s**, reachable **only over Tailscale**, hosting internal + QA workloads moved off EKS to cut cost. It is an **active migration** owned by a parallel session — read `~/.claude/projects/-Users-hyeonwoo-DEV/memory/project-onprem-k3s-migration-2026-07-02.md` for live state before acting, and **verify claims against the live cluster/code, never assert from memory** (stale-memory assertions have already burned trust on this project).

Node: `dylabs@dylabs-onprem-k3s-1` (Tailscale MagicDNS, IP `100.96.80.113`). Repo: `github.com/dylabs/onprem` (Ansible, **main branch**).

## THE IRON RULE (CRITICAL): git -> reconciler. NEVER mutate the cluster by hand.

**Every change to on-prem cluster state goes through git and is applied by a reconciler. You NEVER run `kubectl apply`, `kubectl rollout restart`, `kubectl edit`, `kubectl delete`, or `helm` against the cluster to make a change.**

**Violating the letter of this rule is violating the spirit of it.** "I'll just apply it quickly and also commit it" is the exact failure this skill exists to stop — it desyncs the reconciler, surprises the session that owns the cluster, and blips live services.

There are exactly two reconcilers (pick by what you're changing):

| Changing... | Plane | Reconciler | Source of truth |
|---|---|---|---|
| Platform (ingress-nginx, cert-manager, argocd, arc, coredns/metrics-server/local-path, LimitRanges, node config) | **Ansible** | `ansible-playbook` **run on the node** | `onprem` repo `roles/k3s_platform`, `inventory/group_vars/all/main.yml` |
| App workload (a service in `meloming-qa`) | **ArgoCD** | ArgoCD auto-sync (selfHeal+prune) | `k8s-manifests` App + `deploy-gitops/apps/<svc>/onprem/values.yaml` + `devops-helm` chart |

Read-only kubectl **from your Mac is fine and expected** for observation: `kubectl --context dylabs-onprem get/describe/logs/top ...`. Reads never mutate. **Writes go through a reconciler.**

### Red flags — STOP, you are about to break the rule
- Typing `kubectl apply`, `kubectl rollout restart`, `kubectl edit/delete/patch`, or `helm upgrade` with `--context dylabs-onprem` (or after switching to it)
- "It's just a LimitRange / one restart / a quick fix, I'll commit it too"
- "The ansible run is slow / needs a password, kubectl is faster"
- "I already committed the git change, so applying by hand is basically the same"
- Editing a live HelmChart CR or Deployment instead of the `roles/k3s_platform` template

**All of these mean: revert the hand-change, put it in git, apply via the reconciler.**

| Rationalization | Reality |
|---|---|
| "kubectl apply is faster than ansible-on-node" | Speed isn't the goal; a reconciler-owned cluster is. Hand-apply = drift another session must discover and undo. |
| "I committed it too, so it's gitops" | Committing is not applied-by-reconciler. A k3s restart reverts your CR edit to the file's version; the reconciler is the only durable applier. |
| "Just a restart to pick up a change" | A single-node ingress/coredns restart blips every live service on the box. Only the reconciler's rollout is sanctioned. |
| "Read-only context, so a small write is ok" | Reads are fine; the moment it is apply/edit/delete/restart, it is a mutation. No exceptions. |

## Apply path A — platform change (Ansible-on-node)

The vault secret (`inventory/group_vars/all/vault.yml`) and `.vault_pass` live **on the box**, not in git. sudo needs the node password. So ansible runs **on the node**, not from your Mac.

```bash
# 1. Commit the change to the onprem repo (main) and push.
cd /Users/hyeonwoo/DEV/onprem
git add -A && git commit -m "..." && git push origin main   # onprem is main-based IaC

# 2. Transfer the working tree onto the box — PRESERVE the box's vault.yml/.vault_pass.
#    Use `tar xzf - -C ~/onprem` (overlay). NEVER `rm -rf ~/onprem` — that nukes the box vault.
tar czf - --no-xattrs --exclude='.git' --exclude='.DS_Store' --exclude='.vault_pass' . | \
  ssh -i ~/.ssh/dylabs_onprem_ed25519 -o BatchMode=yes dylabs@dylabs-onprem-k3s-1 \
  'tar xzf - -C ~/onprem && ls ~/onprem/inventory/group_vars/all/'   # confirm vault.yml still there

# 3. Run the playbook AS ROOT ON THE NODE (k3s.yml = k3s+platform; site.yml = full, incl. hardening).
#    The node sudo/become password is operator-supplied — NEVER commit or print it.
#    (the migration thread feeds it via an expect wrapper: ONPREM_PW env -> `sudo -S bash ...`)
#    export LC_ALL=C.UTF-8 LANG=C.UTF-8; cd ~/onprem
#    ansible-playbook -c local -i inventory/hosts.yml playbooks/k3s.yml
```

Gotchas: box may need `apt install -y ansible` once; if a prior root run left root-owned files, `sudo chown -R dylabs ~/onprem` before re-transfer; `LC_ALL=C.UTF-8` is required; the platform HelmCharts are k3s auto-deploy manifests under `/var/lib/rancher/k3s/server/manifests/` — the playbook renders them and the k3s Helm Controller upgrades the releases (rolling the pods **once**).

**If you don't have the node sudo password:** commit + push the git change (that is the durable artifact) and hand the on-node `ansible-playbook` step to the operator. Do **not** substitute local kubectl.

## Apply path B — app workload (ArgoCD gitops)

Mirror an existing on-prem app (e.g. `k8s-manifests/clusters/dylabs-onprem-k3s/apps/meloming-docs.yaml`). Multi-source: `devops-helm/charts/backend-app` (chart) + `deploy-gitops/apps/<svc>/onprem/values.yaml` (values: `ingressClassName: nginx`, `cert-manager.io/cluster-issuer: letsencrypt-int`, host `<svc>.int.sbalyd.com`, `imagePullSecrets: [ecr-pull]`, rollout off), destination `meloming-qa`. Commit all repos. The Application object is registered by `kubectl apply`-ing it **on the node** (`cat app.yaml | ssh ... 'kubectl apply -f -'`) — after which ArgoCD **auto-syncs from git**; you do not hand-apply the workload.

## Migration cutover — EKS -> on-prem, the COMPLETE checklist

Deploying the workload on-prem is HALF the job. "It serves HTTP 200" != "it works". Skip a
step and you ship login-broken apps AND leave EKS double-running. Do EVERY step, IN ORDER.

**Apps are GitOps-managed by the `platform` root-app** (`clusters/dylabs-onprem-k3s/apps/root-app.yaml`, app-of-apps, prune+selfHeal) — add/remove an app file under that dir and ArgoCD reconciles. NEVER `kubectl apply` an App CR by hand (leaves an untracked orphan). EKS apps dir is **kustomize** (edit `kustomization.yaml` too); on-prem is directory-sync (top-level `*.yaml`, no kustomization).

**URL-serving app (admin/front):**
1. **On-prem up** — backend-app chart + `deploy-gitops/apps/<x>/onprem/values.yaml` (nginx, `letsencrypt-int`, host `<x>.int.sbalyd.com`, `ecr-pull`) + k8s-manifests app file (root-app auto-adopts) + `<x>` CI amd64 (`arc-runner-onprem-amd64`, `:onprem`) + terraform `int_a_records`. Dest `meloming-prod` for prod.
2. **Backend CORS** — the browser origin becomes `https://<x>.int.sbalyd.com`; it MUST be in the CORS allowlist of EVERY backend the app calls: the **main API (api.meloming.com)** AND the **app-specific data API** (commission-api/partners-api). Fixing only the main API = login works but the dashboard DATA stays blank (the data API is a separate backend). Enumerate them from the app's `use-api-environment.ts`/config. Allowlist LOCATION varies: commission-back = `CORS_ORIGINS` env in deploy-gitops values (+ hardcoded default); meloming-back = `CORS_EXTRA_ORIGINS` env (+ hardcoded static list; prod host-check exact-match, NO wildcard) which ALSO gates OAuth returnTo (return-to.util.ts reuses it); partners-back = `CORS_ORIGINS` **in a vault secret** — override by adding an explicit `CORS_ORIGINS` env in the git values (k8s env wins over envFrom). OPTIONS-preflight EACH backend for `access-control-allow-origin` before calling it done.
3. **OAuth redirect URI** — if login is OAuth, register `https://<x>.int.sbalyd.com/<callback>` with the IdP.
4. **App build config** — NEXT_PUBLIC_* are baked at BUILD (CI vault-action; repoint its url to `https://vault.pri.sbalyd.com` for the on-prem runner). Confirm API base URLs.
5. **VERIFY IT WORKS** — login + one real action, not just 200. If you can't drive a browser, the operator confirms BEFORE step 6.
6. **301 redirect (EKS)** — ONLY after 2-5 pass. Change `clusters/dylabs-prod-eks-main/apps/<x>-prod.yaml` source `charts/backend-app` -> `charts/redirect`, inline `valuesObject: {name, fromHost: <x>.dylabs.app, toHost: <x>.int.sbalyd.com, alb: {scheme, groupName, certificateArn}}` (reuse the app's ALB fields from its prod values). KEEP the kustomization entry — change content, don't delete the file (deleting a kustomize-listed file breaks `kustomize build` -> platform ComparisonError freezes ALL prod syncs). Ref: `meloming-business-front-qa.yaml`. Drops EKS pods (compute-free) + 301s the old host.
7. **Verify cutover** — `curl -I <x>.dylabs.app` -> 301 -> `<x>.int`; `<x>.int` -> 200.

**Cronjob/worker (no URL):** steps 1 (+ ESO secret) + 5 (a Job runs Complete). Cutover = `suspend: true` on the EKS cronjob once the on-prem one succeeds. No CORS/OAuth/301.

**NEVER** push the 301 (step 6) before the on-prem app is verified working — it sends staff to a broken app. **NEVER** leave EKS running after a verified cutover — that's the double-running you were migrating away from.

## Cloudflare Tunnel — external-facing PROD served from on-prem

The int.sbalyd.com path above is **tailnet-only**. To serve a **public prod domain** (e.g.
`developers.meloming.com`) from on-prem, use **Cloudflare Tunnel (cloudflared)** — outbound-only, no
inbound port, no public IP on the node. (Deep dive: memory `project-cloudflare-tunnel-onprem-2026-07-05`.)

**ONE tunnel per cluster, MANY apps (canonical).** The `dylabs-onprem` tunnel
(`40adc988-514a-41ee-b691-d8adee9a06ec`) serves ALL external prod apps. A new app does **NOT** get a new
tunnel — add a hostname to the existing cloudflared config ingress:
`k8s-manifests/clusters/dylabs-onprem-k3s/manifests/cloudflared/cloudflared.yaml` (resources named
`cloudflared-*`, ns `cloudflared`, ArgoCD `apps/cloudflared.yaml` = directory-sync). Tunnel creds via ESO
from Vault `onprem/cloudflare/tunnel-*`; the API token (DNS edits) is Vault `onprem/cloudflare/tunnel`
(field `api_token`, all-zone). **A config change needs a POD RESTART** — updating the ConfigMap alone does
NOT reload cloudflared; bump a pod annotation, or let a resource rename recreate the pods.

**App deploy (like the int path, but prod dir + tunnel exposure):**
1. `<app>` repo `deploy-onprem.yml` (workflow_dispatch, `runs-on arc-runner-onprem-amd64`, Vault
   `https://vault.pri.sbalyd.com`, `--platform linux/amd64`, tag `onprem-<sha>`). workflow_dispatch must be
   on the DEFAULT (main) branch — if the repo's qa->main PR carries other sessions' work, land ONLY the
   workflow in an isolated main PR (don't merge the omnibus release PR).
2. `deploy-gitops/apps/<app>/onprem-prod/values.yaml` — prod dir, ns `meloming-prod`,
   `ingress.enabled: false` (tunnel exposes it, NOT ingress-nginx), ClusterIP.
3. `k8s-manifests` ArgoCD app (dest `meloming-prod`) + add the hostname to cloudflared ingress pointing at
   `http://<svc>.meloming-prod.svc.cluster.local:80`.
4. **DNS origin flip** (the cutover, replaces the int-path 301): in the Cloudflare zone, host CNAME ->
   `<tunnel-id>.cfargotunnel.com`, **proxied (orange)**, ttl auto.

**[CRITICAL] Cloudflare free/Pro routes KR traffic to the LAX (US) edge.** Even with the connector in
Seoul, `cdn-cgi/trace` colo=LAX and latency is ~0.6-1s vs ~46ms origin-direct (15-20x). 1.1.1.1 (DNS) hits
ICN but the zone PROXY (web) is LAX; only Enterprise+China-network reliably pins the Seoul PoP. So
tunnel/proxy is a **latency hit for KR-facing apps regardless of connector location** — fine for
low-traffic/non-critical, weigh it otherwise. (Local browser not seeing CF = system resolver
cache/propagation — dig @8.8.8.8 vs the connection IP to tell apart — NOT a server problem.)

**EKS teardown after tunnel cutover (do NOT skip — else double-running):** DNS origin is now cfargotunnel,
so the EKS ALB path is dead — REMOVE the EKS prod app entirely (not a 301):
- `k8s-manifests/clusters/dylabs-prod-eks-main/apps/<app>-prod.yaml` deleted + its `kustomization.yaml`
  line removed (comment why) -> ArgoCD prunes EKS pods/svc/ingress/ALB-rule.
- `deploy-gitops/apps/<app>/prod/values.yaml` (EKS values) removed.
- `<app>` repo `deploy-prod.yml` (EKS arm64 build CI) removed — prod builds via `deploy-onprem.yml` now.
- Keep the QA/int path untouched (separate concern). Verify: EKS app NotFound, meloming-prod pods gone,
  ingress gone, `<host>` still 200 via tunnel.

**Rename/refactor cloudflared with ZERO downtime:** same tunnel ID = connectors coexist. Push the new
resource set, wait for it Running + serving (200), THEN prune the old — both connect to the one tunnel in
between, so no gap. (Vault `kv put` is hook-blocked here -> hand a creds-path rename to the operator; the
Cloudflare tunnel NAME needs an account-scoped token or the dashboard.)

## Hard constraints

- **Bandwidth: 30 Mbps is the committed *average*, NOT a hard cap — bursts pull the full physical 1G NIC.** The only real constraint is a *sustained, continuous high-throughput stream* whose 24/7 average exceeds the committed rate (e.g. monitoring telemetry ingest ~150 Mbps, continuous bulk replication) — keep those on EKS. Bursty traffic — serving, on-demand queries, DB request/response, image pulls, batch, backup — runs freely on the 1G burst and averages low. So don't co-locate a service that holds a continuous high-bandwidth stream; ordinary bursty workloads are fine.
- **EKS stays for critical** (meloming front/back, commission, partners, overlay, gateway, clickhouse, chat-worker, RDS/ElastiCache). On-prem = non-core prod + all QA + heavy-common. "amd64 everywhere" is wrong — EKS prod is arm64 Graviton.
- **Multi-session**: a parallel thread actively owns this migration. Don't bulldoze the shared node/cluster; sequence disruptive rolls (ingress) and preserve its in-flight box state (the `tar` overlay, never `rm -rf`).
- **Domains**: on-prem = `*.int.sbalyd.com`; AWS keeps `pri.sbalyd.com`. `pri` resolves only inside the VPC/box (verify `pri` from the node, `int` from the tailnet). Never guess domains.

## Quick reference

| Task | Do | Never |
|---|---|---|
| Observe cluster | `kubectl --context dylabs-onprem get ...` (from Mac) | — |
| Change platform (ingress/cert-manager/argocd/arc/coredns/LimitRange) | edit `roles/k3s_platform` -> commit -> transfer -> `ansible-playbook ... k3s.yml` on node | local `kubectl apply` / `helm` / `rollout restart` |
| Change/add app | ArgoCD App + deploy-gitops onprem values + devops-helm | hand-`kubectl apply` the workload |
| Give a pod resources/limits | put them in the HelmChart values / a LimitRange in `roles/k3s_platform` | `kubectl apply` a LimitRange, `rollout restart` to pick it up |
| Apply a fix "quickly" | there is no quick path — reconciler only | any hand-mutation |
