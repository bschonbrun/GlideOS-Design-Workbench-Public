# GlideOS Design Workbench — Public (delivery mirror)

> Response style governed globally by `~/.claude/CLAUDE.md`.

## What this repo is

**Not a dev repo.** It is the public delivery surface for the GlideOS Design Workbench: `manifest.json`, `workbench-install.md`, `LICENSE`, `README.md`. Installed GlideOS agents fetch these via `raw.githubusercontent.com` (no PAT). Source lives in the private sibling `bschonbrun/GlideOS-Design-Workbench`.

## Rules

- **One-way sync.** Never edit `manifest.json` / `workbench-install.md` here by hand. Change them in the private repo, then run its `./sync-public.sh` (private → public), review the diff, push.
- **Pushes are manual and land on `main`.** The agent bridge does not reach this repo; a human/session pushes from this folder.
- **`manifest.json` `app_files` must never be null/empty.** Commit `a92b1c1` (v2.5.6) nulled it and dropped `server.ts` / `LayoutCanvas.tsx` / `source-bundle*.ts` from delivery — installs broke. Last known-good with a full 25-file list: **v2.5.3** (`b0cf0a0`). `sync-public.sh` guards against re-shipping a null list.
- **Nothing secret** ever belongs here — it is public.
