# Nightly restic backup (HA + Z-Wave) → Cloudflare R2

A Flux-managed CronJob that runs nightly (04:00 UTC) and ships a restic
repository of the home-automation state to a Cloudflare R2 bucket.

**Threat model: single-node failure.** The repo is off-node and off-prem
(R2 side-effect: also survives whole-premises loss, at $0). On-site targets
(Mac/desktop/Pi SD card) were rejected: backups depending on another device
being awake silently go stale, and SD cards are the least durable storage we
own.

## What is captured

| Source | How | Why |
|---|---|---|
| HA `/config` (`.storage`, yaml, `custom_components` incl. HACS, `zigbee.db`) | hostPath read-only; sqlite online `.backup` for the live DBs | full HA config + ZHA device catalog |
| zwave-js-ui store | REST API zip (`GET /api/store/backup`, port 8091, `Accept: application/json`) | controller config, node cache, provisioning keys |
| Z-Wave NVM | zwave-js-ui's built-in scheduler writes `store/backups/nvm/NVM_*.bin` at 03:00 UTC; newest file snapshotted | stick NVRAM: topology, routing, S2/S0 keys (800-series stick confirmed) |
| ZHA network backup (network key) | manual, quarterly (see below) | stick-death insurance — not exportable headlessly |

`home-assistant_v2.db` (recorder history) is included in the tar/snapshot but
HA's default 10-day purge keeps it at a plateau (~100 MB), so it stays small.
If you configure long history retention, reconsider.

The **zwave NVM export only exists on 700/800-series sticks** — this node has
an 800-series stick, so the scheduler produces real NVM files nightly.

One-time: enable the built-in backup schedulers in zwave-js-ui (Settings →
save, then restart the gateway) so it writes backups at 03:00 UTC, before the
04:00 restic run. Done on the live cluster already (settings key `backup`,
`nvmCron`/`storeCron` = `0 3 * * *`, keep 14):

```sh
# easiest via the UI, or:
# fetch settings, set settings.backup = {nvmBackup:true, nvmCron:"0 3 * * *",
#   nvmBackupOnEvent:false, nvmKeep:14, storeBackup:true, storeCron:"0 3 * * *", storeKeep:14},
# then POST /api/settings and POST /api/restart (config is re-read on restart)
# -- both with header "Accept: application/json" (the API is content-negotiated)
```

## One-time setup (outside the repo, matching existing secret convention)

The repo only references secrets by name — nothing sensitive is committed
(consistent with `cloudflare-api-token`).

```sh
# 1. Cloudflare: create bucket "ha-backup" + an R2 API token scoped to
#    Object Read & Write on that bucket only. Note your account ID.

# 2. Generate a repo password (the ONLY key to the backups):
openssl rand -base64 48

# 3. Create the credentials secret:
kubectl create secret generic restic-backup-credentials -n flux-system \
  --from-literal=RESTIC_REPOSITORY='s3:https://<ACCOUNT_ID>.r2.cloudflarestorage.com/ha-backup/restic' \
  --from-literal=AWS_ACCESS_KEY_ID='<R2_ACCESS_KEY>' \
  --from-literal=AWS_SECRET_ACCESS_KEY='<R2_SECRET>' \
  --from-literal=RESTIC_PASSWORD='<GENERATED>'

# optional: Discord webhook for failure alerts
kubectl patch secret restic-backup-credentials -n flux-system \
  --patch '{"stringData":{"NOTIFY_URL":"https://discord.com/api/webhooks/..."}}'
```

Flux applies the CronJob via the `infrastructure` Kustomization. First run
initialises the repository automatically (`restic init`).

## Why these choices

- **restic, pinned** (`restic/restic:0.19.1`): dedup keeps the repo ~0.5–1 GiB
  forever, client-side encryption for the network keys, point-in-time
  recovery, and the same tool/runbook shape as the Minecraft repo on OCI.
  Pinned deliberately — a backup tool must be the most boring thing in the
  stack, and newer restic always reads older repos, so restores never break.
- **R2, standard storage class**: free tier covers the size; IA's cheaper
  per-GiB only matters past the free tier, and its min-duration billing
  penalizes pruned (young) packs + costs more on reads — the wrong shape for
  nightly deltas read all-at-once during a restore.
- **restic `s3` backend against R2 directly** — no rclone layer
  (rclone stays for the OCI/Minecraft repo where the endpoint is quirky).
- **zwave API on port 8091 with `Accept: application/json`** — the API is
  content-negotiated: without the header it serves the SPA (discovered during
  bring-up; the store backup is `GET /api/store/backup`, not `/api/backup`,
  and v11.x has no REST NVM route at all — the app's own scheduler + hostPath
  capture is the correct mechanism).

## ZHA network key — the one manual step (quarterly / before stick changes)

Home Assistant UI → Settings → Devices & Services → ZHA → device page →
**network backup** (download). Save the file **into `/config/backups/` on the
HA hostPath** (i.e. `/root/container-data/homeassistant/backups/`) — the next
nightly restic run picks it up automatically. It is the only record of the
coordinator's NVRAM → needed to restore a network onto a *replacement* stick.

## Restore (fresh machine)

```sh
export RESTIC_REPOSITORY='s3:https://<ACCOUNT_ID>.r2.cloudflarestorage.com/ha-backup/restic'
export AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=... RESTIC_PASSWORD=...
restic -o s3.region=auto snapshots          # find the latest
restic -o s3.region=auto restore latest --target /tmp/restore
```

Map back onto the node:

- `/tmp/restore/homeassistant/*` → `/root/container-data/homeassistant/`
- extract `zwave/store-backup.zip` → `/root/container-data/zwave-ui/`
  (or copy `zwave/store/` raw if the API was down that night)
- `zwave/nvm.bin` → `/root/container-data/zwave-ui/backups/nvm/` (restore
  from anywhere via UI: Settings → Backup NVM → upload), or restore directly
  over a replacement 800-series stick via the UI "Restore NVM" flow

Do this with the HA / zwave pods scaled to 0, then scale back up. The sqlite
snapshots are consistent by construction — restore as-is.

Occasional hygiene: `restic -o s3.region=auto check` (full integrity) and a
test restore — cheap at this size, and it's the only way to know the net
works before you need it.

## Retention

`--keep-daily 7 --keep-weekly 4 --keep-monthly 3` (with `--prune`), applied
in the script each run. R2 lifecycle rules can also be added later as a
second layer of cleanup.