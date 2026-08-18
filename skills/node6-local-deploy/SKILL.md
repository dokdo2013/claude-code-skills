---
name: node6-local-deploy
description: "Use when deploying a retained production app to Node6 (dylabs-onprem-6) by hand because GitHub Actions cannot build — Vault (vault.pri.sbalyd.com) is permanently gone with node3/4/5, so deploy-prod.yml fails at 'Verify Vault over tailnet' and no new GHCR image can be produced. Covers meloming-front, meloming-commission-front, meloming-id-front, dylabs-admin (Next standalone under PM2) and meloming-back, meloming-commission-back, meloming-partners-back, meloming-image-worker. Triggers: node6, dylabs-onprem-6, 로컬 업로드, CI 없이 배포, 직접 올려, PM2 릴리스, /srv/apps, standalone 배포, Vault 없이 빌드, commission.meloming.com 배포."
---

# Node6 local deploy (no CI, no Vault)

Node6 (`dylabs-onprem-6`, Tailscale `100.91.255.63`) is the **only retained production runtime** after the AWS/OnPrem shutdown. It is a plain Ubuntu box — **not Kubernetes, not containers**. Nginx + PM2 + local MySQL/Valkey.

Authoritative context lives in `docs/aws-onprem-shutdown-master-plan-2026-08-16.md` (terraform-main) and the `onprem` repo (`roles/node6_front_staging`, `roles/node6_front_activation`). **Read those before acting; never assert Node6 state from memory.**

## IRON RULES

1. **Never print real secrets.** Master plan: "값이 있는 secret은 어떤 source, plan, log, receipt 또는 chat에도 노출하지 않는다." That means server-side credentials — DB passwords, `SHORTY_API_KEY`, Vault/credstore contents. Move those as files (`scp`, `install -m 0600`), never `cat`/`echo` them.
   **`NEXT_PUBLIC_*` is not in that class.** Next inlines it into the browser bundle, so every visitor already has it — PortOne `storeId`, Toss/Danal/Payple client keys included. Reading and printing them is fine, and since Vault is gone the deployed bundle is the *only* surviving copy. Recover them there (see below) instead of treating them as secrets.
2. **Check the lease first.** Node6 is worked by parallel sessions under an exact-lease contract. Read-only inspection is always fine; mutating an app you do not own is a fail-stop. Touch only your target service.
3. **Only the target-eight allowlist may run.** `meloming-back`, `meloming-commission-back`, `meloming-partners-back`, `meloming-front`, `meloming-commission-front`, `meloming-image-worker`, `meloming-id-front`, `dylabs-admin`. Never add a ninth process.
4. **Production source is `main` only.** The staging contract rejects `qa` refs and QA images. Merge to `main` first; deploy that exact SHA.
5. **Release dirs are immutable and additive.** Never edit a live release in place. New SHA → new dir → flip symlink. The old release stays for instant rollback.
6. **Production DB is SELECT-only.**

## Layout

```
/srv/apps/<service>/
├── current -> releases/<full-git-sha>     # root-owned symlink
├── releases/<full-git-sha>/               # Next standalone, hoisted to root
│   ├── server.js  .next/  node_modules/  public/  package.json
│   └── NODE6-RELEASE.json                 # provenance receipt
├── shared/ecosystem.config.cjs            # PM2 definition
└── logs/
```

Each app runs as its own system user (`<service>`, nologin, home = `/srv/apps/<service>`).
Runtime is pinned: `/opt/dylabs/node/v20.20.2/bin/node`, entrypoint `server.js`.
Supervisor: `systemd` unit `pm2-app@<service>.service`.

| service | loopback port | health |
|---|---|---|
| `meloming-front` | 3103 | `/api/health` |
| `meloming-id-front` | 3104 | `/api/health` |
| `meloming-commission-front` | 3105 | `/api/health` |
| `dylabs-admin` | 3106 | `/api/health` |

Public edge is Cloudflare → tunnel → these loopback ports. Do not change routes/DNS from this skill.

## Why CI is dead

`deploy-prod.yml` does `Verify Vault over tailnet` then pulls ~30 values from `vault.pri.sbalyd.com` and passes them as `--build-arg`. Vault lived on node3/4/5, which are **permanently shut down**. The step fails with `curl: (28)`, so no new GHCR image exists for any new commit.

Consequence: the frontends' `NEXT_PUBLIC_*` values were **only** in Vault. Node6 env custody (`/etc/credstore.encrypted`, `/var/lib/dylabs-node6-foundation/app-env-custody`) covers **backends only** — frontends bake their values at build time.

### Recovering the frontend build env

The deployed release is now the authoritative copy. Next replaced each `process.env.NEXT_PUBLIC_X` with a literal, so read them back out of the bundle:

```bash
R=/srv/apps/<service>/releases/<sha>
# API_CONFIG block gives AUTH_API_URL / COMMISSION_API_URL / APP_URL / MAIN_SERVICE_URL
sudo -n grep -rhoE 'AUTH_API_URL:"[^"]*"[^}]{0,300}' $R/.next/server | head -1
# payment + analytics literals, with enough context to map each to its provider
sudo -n grep -rhoE '.{60}(storeId|channelKey|clientKey|merchantId):"[^"]{4,}"' $R/.next/server | sort -u
```

Key *names* mostly vanish (only a few survive, e.g. `NEXT_PUBLIC_ID_URL` inside `resolveIdServiceUrl`), so map by call-site context, not by name.

Enumerate what you must recover from source, not from guesswork:

```bash
grep -rhoE 'process\.env\.NEXT_PUBLIC_[A-Z0-9_]+' src/ | sed 's/process.env.//' | sort -u
```

**Prove the set before shipping.** Rebuild the *currently deployed* SHA with your recovered env and diff the bundle literals against the live release. Match ⇒ the env is right. Mismatch ⇒ stop; a mis-mapped payment key breaks production checkout.

## Frontend deploy (Next standalone)

### 1. Confirm ownership and current state (read-only)

```bash
ssh dylabs@100.91.255.63 'sudo -n readlink /srv/apps/<service>/current; \
  systemctl is-active pm2-app@<service>; \
  sudo -n cat /srv/apps/<service>/releases/*/NODE6-RELEASE.json | head -20'
```

Record the current release SHA — that is your rollback target.

### 2. Build locally at the exact `main` SHA

`next.config.ts` sets `output: "standalone"`. Write the recovered env to `.env.production.local` (it outranks the repo's `.env.local`, which points at prod already):

```bash
cd <repo> && git fetch origin main && git checkout <full-sha>
install -m 0600 /path/to/prod.env .env.production.local
pnpm install --frozen-lockfile && pnpm build
```

Package exactly what the release dir expects — `.next/standalone` hoisted to root, plus `static` and `public`:

```bash
OUT=$(mktemp -d)
cp -a .next/standalone/. "$OUT"/
mkdir -p "$OUT/.next" && cp -a .next/static "$OUT/.next/static"
cp -a public "$OUT/public"
rm -f .env.production.local                                # do not ship build env
tar -C "$OUT" -I 'zstd -19' -cf <service>-<sha12>-rootfs.tar.zst .
sha256sum <service>-<sha12>-rootfs.tar.zst
```

### 3. Stage as a new release (does not touch live traffic)

```bash
scp <artifact> dylabs@100.91.255.63:/var/cache/dylabs-node/
ssh dylabs@100.91.255.63 'sudo -n sha256sum /var/cache/dylabs-node/<artifact>'   # must match
ssh dylabs@100.91.255.63 'sudo -n install -d -o <service> -g <service> -m 0750 \
    /srv/apps/<service>/releases/<full-sha> && \
  sudo -n tar -C /srv/apps/<service>/releases/<full-sha> -I zstd -xf /var/cache/dylabs-node/<artifact> && \
  sudo -n chown -R <service>:<service> /srv/apps/<service>/releases/<full-sha>'
```

Arm64 image extracts need the x86_64 `sharp` overlay (`sharp-linux-x64`, `sharp-libvips-linux-x64`, pinned versions) linked into pnpm resolution; a locally built x86_64 artifact does not. Verify either way:

```bash
ssh dylabs@100.91.255.63 'cd /srv/apps/<service>/releases/<full-sha> && \
  sudo -n -u <service> /opt/dylabs/node/v20.20.2/bin/node -e \
  "const s=require(\"sharp\"); if(process.arch!==\"x64\")process.exit(2)"'
```

Write `NODE6-RELEASE.json` recording `sourceRevision`, `artifactSha256`, `nodeVersion`, `port`, `healthPath`.

### 4. Smoke the new release before it takes traffic

Run it on a scratch port and check health, then stop it. Never bind the live port for this.

### 5. Flip and restart

```bash
ssh dylabs@100.91.255.63 'sudo -n ln -sfn /srv/apps/<service>/releases/<full-sha> \
    /srv/apps/<service>/current.new && \
  sudo -n mv -Tf /srv/apps/<service>/current.new /srv/apps/<service>/current && \
  sudo -n systemctl restart pm2-app@<service>'
```

`ln` + `mv -T` makes the swap atomic; a bare `ln -sfn` onto an existing symlink can nest.

### 6. Verify before claiming anything

All five must pass:

1. `systemctl is-active pm2-app@<service>` → `active`, restart counter not climbing
2. loopback `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:<port>/api/health` → `200`
3. `sudo -n readlink /srv/apps/<service>/current` ends with your SHA
4. the **running process** points at the new release, not just the symlink:
   `sudo -n ps -eo args | grep /srv/apps/<service>` must show your SHA in the path
5. public URL returns the expected status **and** the changed behavior is observable

Check 4 is not redundant with check 3. PM2 takes its working directory from the
release's own `ecosystem.config.cjs`, so a release whose config still names the
previous directory serves old code while the symlink advertises the new SHA.
Grep the built artifact for something only the new code contains when the change
is not visible from outside.

A public URL that answers 403 to your `curl` is not necessarily down. The
meloming.com zone runs bot protection that rejects non-browser clients on some
hosts, so verify with a real browser before calling a deploy broken, and give CI
verification steps the release credential held in the GitHub environment. The
project memory reference on that zone has the exact rule set and credential
names.

Do not say "배포 완료" until 1–5 all pass. Then report the release SHA and the rollback SHA.

### Rollback

Point `current` back at the previous release dir and restart. It is one symlink flip because the old release was never mutated.

## Backend deploy

Backends differ: they are **built on Node6** from a full `main` SHA using Node6's own git identity and `corepack pnpm install --frozen-lockfile`, resolving exactly one of `dist/main.js` or `dist/src/main.js`. Their env comes from `/etc/credstore.encrypted/node6-app-env-active-<service>.json` custody, not from a build arg — so backends do **not** need Vault. Never mix host Node 24 native modules with the Node 20 app runtime.

### The release must carry its own `ecosystem.config.cjs`

The unit declares `ConditionPathExists=/srv/apps/<service>/current/ecosystem.config.cjs`.
A release unpacked from a plain source archive does not contain that file, and the
next restart is **skipped, not failed** — systemd reports the condition and the
service stays down. This took `meloming-back` down on 2026-08-18.

Copy it from the previous release, then rewrite `cwd` to the new release path:

```bash
ssh dylabs@100.91.255.63 'sudo -n cp -p /srv/apps/<service>/releases/<prev-sha>/ecosystem.config.cjs \
    /srv/apps/<service>/releases/<new-sha>/ecosystem.config.cjs && \
  sudo -n sed -i "s|<prev-sha>|<new-sha>|g" /srv/apps/<service>/releases/<new-sha>/ecosystem.config.cjs && \
  sudo -n grep -n cwd /srv/apps/<service>/releases/<new-sha>/ecosystem.config.cjs'
```

Skipping the `sed` is worse than forgetting the file: the service starts, health
passes, and PM2 quietly keeps executing the previous release.

### Building a Nest backend on Node6

```bash
sudo -n -u <service> env PATH=/opt/dylabs/node/v20.20.2/bin:/usr/bin:/bin \
  COREPACK_ENABLE_DOWNLOAD_PROMPT=0 HOME=/srv/apps/<service> \
  NODE_OPTIONS=--max-old-space-size=6144 \
  bash -c 'cd /srv/apps/<service>/releases/<sha> && corepack pnpm install --frozen-lockfile && corepack pnpm exec prisma generate && corepack pnpm build'
```

Two failures look like broken source but are not:

- Without `prisma generate` first, `nest build` reports thousands of type errors
  because the generated client types are absent.
- Without the raised heap, V8 aborts mid-build (`exit 134`) and leaves `dist/`
  incomplete. The build script's own exit code can still read as success, so
  check that `dist/main.js` exists and is newer than the build.

### Changing backend environment

The unit loads env through `LoadCredentialEncrypted` at start, so an env change is
decrypt, merge, reseal, restart, health check, and restore the previous file if the
service does not come back. Do it with a reconciler that writes a receipt under
`/var/lib/dylabs-node6-foundation/`, never by hand: a parallel session needs to see
what changed and why. `onprem` carries both directions for the export client
(`node6_dlp_export_client_retirement` and `node6_dlp_export_client_activation`) as
the working example.

## Preferred path: Ansible, not hand commands

The `onprem` repo already encodes this contract:

- `playbooks/activate-node6-production-frontends-local.yml` → `roles/node6_front_activation`
- staging via `roles/node6_front_staging` (`node6_front_staging_apps` pins service/revision/image/digest/archive sha256)

Update `roles/node6_front_staging/defaults/main.yml` with the new revision + artifact checksum and run the playbook. Hand commands are the fallback when the controller artifact path is unavailable; the layout, checksums, and verification above must still hold.

## Do not

- deploy a `qa` ref or QA image to Node6
- rebuild or patch `.next` inside an existing release
- add env to the PM2 ecosystem beyond `NODE_ENV`/`HOSTNAME`/`PORT`
- restart or touch apps you are not deploying
- flip `current` before the new release has its own `ecosystem.config.cjs` with its own `cwd`
- treat a green health check as proof the new code is live — check the process path
- claim completion from CI success or a gitops tag — that pipeline no longer reaches Node6

## Parallel sessions

Another session may deploy the same service while you work. Before flipping, read
`current` again and confirm your SHA is a descendant of what is live
(`git merge-base --is-ancestor <live-sha> <your-sha>`). If it is not, rebuild on
top of the live revision instead of overwriting it. Announce the service you hold,
the SHA you flipped, and the rollback SHA so the next session can plan around you.
