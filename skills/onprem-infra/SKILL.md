---
name: onprem-infra
description: "Use when touching the dylabs on-prem k3s cluster — the `dylabs-onprem` kube context, the KoreaIDC bare-metal node, `*.int.sbalyd.com`, the `onprem` Ansible repo, `meloming-qa` on-prem, or migrating a QA/internal service off EKS. Especially when about to `kubectl apply`/`rollout restart`/`helm` against onprem, edit `roles/k3s_platform`, add LimitRanges/resources, or ask 'how do I apply this to onprem'. Triggers: onprem, dylabs-onprem, k3s, int.sbalyd.com, KoreaIDC, 온프렘, 온박스, ansible-playbook."
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

## Hard constraints

- **Bandwidth: the node has 30 Mbps contracted** (not the 1G NIC). Everything shares it — serving, on-prem<->AWS cross-links, telemetry, image pulls. Migration is bandwidth-budgeted; heavy telemetry (monitoring) stays on EKS. Don't schedule chatty/high-egress workloads here.
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
