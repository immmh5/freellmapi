# FreeLLMAPI on Render — deployment notes

Deployed: 2026-10-01
Upstream: https://github.com/tashfeenahmed/freellmapi (latest: v0.13.3)
Fork:     https://github.com/immmh5/freellmapi
Service:  https://dashboard.render.com/web/srv-dav8ea142hec73838dm0
URL:      https://freellmapi-4khz.onrender.com

## CRITICAL — keep this safe
ENCRYPTION_KEY (64-char hex, encrypts your stored provider API keys at rest):
  88f7503220705201cf5658b79e892265f61e6b489cc034f8e8a30c42a841d6f5

If this key is lost, every provider key already stored in the dashboard becomes
undecryptable. If it is changed, the same applies. Back this up somewhere safe.

## How the automatic updates work (two layers)

1. GitHub Action "Sync upstream" (in the fork at
   `.github/workflows/sync-upstream.yml`):
     - Runs on schedule automatically, twice a day (00:00 and 12:00 UTC).
     - Merges new commits from tashfeenahmed/freellmapi into the fork's `main`.
     - On a render.yaml conflict it keeps the fork's copy.
     - Also runnable manually: Actions tab → "Sync upstream" → Run workflow.
       The "force" input pushes an empty commit to trigger a Render rebuild
       even when nothing new came from upstream.

2. Render auto-deploy (commit trigger, enabled on the service):
     - Any push to the fork's `main` branch makes Render rebuild and redeploy
       automatically. No manual step.

So: upstream update → twice-a-day sync lands it in the fork → Render rebuilds.

## Manual control (if you want to decide yourself)

- Pause automatic syncs: Actions tab → "Sync upstream" → click the "..." menu
  → Disable workflow. Re-enable the same way.
- Pause Render auto-deploy: Dashboard → the service → Settings → toggle
  "Auto-Deploy" off. Then you deploy manually with the "Manual Deploy" button
  (or the Dashboard "Clear build cache & deploy" on failure).
- Update right now, on demand: Actions tab → "Sync upstream" → Run workflow.
- Force a rebuild with no upstream changes: same run, set force=true.

## First login (important)

The dashboard is public on Render, so the first account creation requires a
one-time setup code that the server prints in its startup logs while no account
exists yet. Find it in the Render Dashboard → the service → Logs, near the top
of a fresh boot, then use it at https://freellmapi-4khz.onrender.com to create
the admin account. After the first account exists the code is no longer needed.

## Free-plan limitation to know about

The service is on the free plan, which has no persistent disk. SQLite (which
holds your provider keys, routing settings, and usage data) lives in the
container filesystem, so it is wiped on every redeploy and when the service
sleeps/wakes. In practice this means re-adding provider keys after each update.

To make data durable, upgrade the service to Starter ($7/mo) and attach a
persistent disk at /app/server/data (Dashboard → Settings → change plan, then
Disks → add disk, mount path /app/server/data). The Dockerfile and its
entrypoint were written with Render in mind and handle the volume ownership for
you. Keep the ENCRYPTION_KEY unchanged across the upgrade.

## Custom domain

Dashboard → Settings → Custom Domains. A freellmapi-4khz.onrender.com URL is
already active and TLS-terminated by Render.
