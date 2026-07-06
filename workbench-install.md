# Design Workbench v2.7.0 — Bootstrap Install

The Design Workbench is a visual tool for building and iterating on GlideOS apps without writing code. It provides a canvas interface to inspect and style elements, an AI-powered instruction queue, workflow automation, appearance controls, and full build history. This install procedure sets it up in your project and connects it to one of your deployed apps.

You are a GlideOS agent. Follow these instructions exactly. Do not skip steps. Do not proceed past a step until it succeeds.

---

## Prerequisites

Before starting, you need:

1. **A GlideOS project** — you are currently inside it
2. **Your org ID, project ID, and email** — call `auth_whoami` right now and store the results:
   - `installer_org_id` = `orgId` field
   - `installer_project_id` = `projectId` field
   - `installer_email` = the email from the delegation chain (`onBehalfOfUserId` maps to an email — use `x-glide-user-email` header value if available, otherwise ask the user for their email)
3. **A target app** — ask the user: "Which app in this project should the Workbench control? (Give me the app name and its production URL.)" If they don't know the URL, call `app_list` to show them all deployed apps with their URLs. Store `target_app_name` and `target_app_url`

You do **not** need a GitHub Personal Access Token. The Workbench installer serves all scaffold files directly — no GitHub access required.

Do **not** ask about GitHub tokens or invite codes. Access is licensed by project ID, not by email or invite code.

---

## Step 0 — Choose install mode

Ask the user exactly this (word for word):

> **How would you like to install the Workbench?**
>
> **Default** — fastest path. I'll make sensible choices automatically and get you running in minutes. Recommended for most users.
>
> **Expert** — I'll ask you about each configuration decision and explain the tradeoffs. Recommended if you want full control over data sources, proxy paths, fixture seeding, and integrations.

Wait for their answer. Store it as `install_mode` — either `"default"` or `"expert"`.

**If they say anything resembling "default", "fast", "quick", "recommended"** → `install_mode = "default"`.
**If they say "expert", "advanced", "all options", "full control"** → `install_mode = "expert"`.
**If unclear** → default to `"default"` and tell them: "I'll use Default mode — you can always change settings later in the Workbench."

**Expert mode** asks clarifying questions at each step and surfaces all optional integrations up front, including:
- GitHub PAT for `/update` and `/git-push` commands (ask now, before Step 1)
- Whether to use live data vs. fixture data for the preview (ask in Step 7h)
- Proxy path customisation (ask in Step 5)

**Default mode** skips all of those questions and:
- Defers GitHub PAT — prompts only with a brief one-liner at the end
- Auto-detects the data source (Step 7h) and announces its choice without asking for confirmation
- Auto-sets proxy paths based on detected routes (Step 5 and 7d)

Write the install mode to the DB **after Step 2 creates the schema** (see Step 2 for the `install_mode` column DDL). You cannot write it yet — continue to Step 1.

---

## Step 1 — Fetch the manifest

Fetch the installation manifest from the public installer endpoint (no auth required):

```
GET https://workbench-install.glideapps.dev/app-api/manifest
```

**Important:** Use the `javascript` tool with a native `fetch()` call, NOT the `web_fetch` tool or any browser automation. The `web_fetch` tool will hit Glide's auth gateway and return login HTML instead of the manifest.

Parse the JSON response. Note the `version` field.

**If the response is any error:** Tell the user: "❌ Could not reach the Workbench installer — please try again shortly."

---

## Step 2 — Create the database schema

Execute all SQL blocks below using `db_execute`. Each uses `CREATE TABLE IF NOT EXISTS`, so re-runs are safe. Execute each block as a separate call.

### Orgs & org metadata

```sql
CREATE TABLE IF NOT EXISTS design_orgs (
  id TEXT PRIMARY KEY DEFAULT 'default',
  name TEXT NOT NULL DEFAULT 'My Workbench',
  slug TEXT NOT NULL DEFAULT 'my-workbench',
  auto_publish BOOLEAN NOT NULL DEFAULT true,
  scaffold_complete BOOLEAN DEFAULT false,
  scaffold_version TEXT,
  connected_app_url TEXT,
  connected_app_name TEXT,
  connected_app_slug TEXT,
  proxy_paths TEXT[] NOT NULL DEFAULT '{}',
  trigger_mode TEXT NOT NULL DEFAULT 'on_write',
  listener_url TEXT,
  install_mode TEXT NOT NULL DEFAULT 'default',
  created_at TIMESTAMPTZ DEFAULT now(),
  created_by TEXT
);
ALTER TABLE design_orgs ADD COLUMN IF NOT EXISTS install_mode TEXT NOT NULL DEFAULT 'default';
INSERT INTO design_orgs (id, name, slug) VALUES ('default', 'My Workbench', 'my-workbench')
ON CONFLICT (id) DO NOTHING;
```

**Now write the install mode from Step 0:**

```sql
UPDATE design_orgs SET install_mode = '{install_mode}' WHERE id = 'default';
```

### Preferences & session state

```sql
CREATE TABLE IF NOT EXISTS design_session_turns (
  session_id TEXT NOT NULL,
  org_id TEXT NOT NULL DEFAULT 'default',
  turn_count INTEGER NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (session_id, org_id)
);

CREATE TABLE IF NOT EXISTS design_preferences (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default' UNIQUE,
  communication_style TEXT DEFAULT 'balanced',
  model_effort TEXT DEFAULT 'balanced',
  explanation_verbosity TEXT DEFAULT 'normal',
  user_name TEXT,
  user_role TEXT,
  user_tech_level TEXT DEFAULT 'non-technical',
  auto_summarize_turns INTEGER DEFAULT 40,
  summarize_mode TEXT DEFAULT 'prompt',
  current_session_id TEXT,
  agent_key TEXT,
  updated_at TIMESTAMPTZ DEFAULT now()
);
INSERT INTO design_preferences (org_id) VALUES ('default') ON CONFLICT (org_id) DO NOTHING;
```

### Design tokens & themes

```sql
CREATE TABLE IF NOT EXISTS design_tokens (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  accent_color TEXT,
  secondary_color TEXT,
  border_radius INTEGER,
  font_family TEXT,
  spacing_scale TEXT,
  shadow_intensity TEXT,
  font_size_scale TEXT,
  button_primary_bg TEXT,
  button_primary_color TEXT,
  button_primary_radius INTEGER,
  button_secondary_border TEXT,
  button_secondary_color TEXT,
  card_bg TEXT,
  card_border TEXT,
  card_radius INTEGER,
  input_bg TEXT,
  input_border TEXT,
  input_radius INTEGER,
  nav_bg TEXT,
  nav_active_bg TEXT,
  nav_active_color TEXT,
  status_success_bg TEXT,
  status_success_color TEXT,
  status_error_bg TEXT,
  status_error_color TEXT,
  status_warning_bg TEXT,
  status_warning_color TEXT,
  heading_color TEXT,
  body_color TEXT,
  updated_at TIMESTAMPTZ DEFAULT now(),
  updated_by TEXT
);
INSERT INTO design_tokens (org_id) VALUES ('default') ON CONFLICT DO NOTHING;

CREATE TABLE IF NOT EXISTS design_themes (
  id BIGSERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  accent_color TEXT,
  font_family TEXT,
  border_radius INTEGER,
  spacing_scale TEXT,
  org_id TEXT NOT NULL DEFAULT 'default',
  created_by TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  applied_at TIMESTAMPTZ,
  applied_by TEXT
);

CREATE TABLE IF NOT EXISTS design_theme_applications (
  id BIGSERIAL PRIMARY KEY,
  theme_id BIGINT REFERENCES design_themes(id) ON DELETE SET NULL,
  theme_name TEXT,
  accent_color TEXT,
  org_id TEXT NOT NULL DEFAULT 'default',
  applied_by TEXT,
  applied_at TIMESTAMPTZ DEFAULT now(),
  note TEXT
);
```

### Element styling & overrides

```sql
CREATE TABLE IF NOT EXISTS design_element_overrides (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  page_path TEXT NOT NULL,
  wb_id TEXT NOT NULL,
  styles JSONB,
  component_type TEXT,
  element_text TEXT,
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, wb_id)
);

CREATE TABLE IF NOT EXISTS design_element_style_history (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  wb_id TEXT NOT NULL,
  page_path TEXT,
  component_type TEXT,
  styles JSONB,
  changed_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS design_element_chats (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  wb_id TEXT,
  topic TEXT,
  messages JSONB DEFAULT '[]',
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

### Instruction queue & build trigger

```sql
CREATE TABLE IF NOT EXISTS design_instructions (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  status TEXT NOT NULL DEFAULT 'pending',
  type TEXT NOT NULL DEFAULT 'other',
  title TEXT,
  brief TEXT,
  context JSONB DEFAULT '{}',
  diff JSONB,
  result TEXT,
  user_input TEXT,
  ai_raw_response TEXT,
  model_used TEXT,
  input_tokens INTEGER,
  output_tokens INTEGER,
  created_at TIMESTAMPTZ DEFAULT now(),
  executed_at TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS design_build_requests (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  requested_at TIMESTAMPTZ DEFAULT now(),
  processed_at TIMESTAMPTZ,
  status TEXT NOT NULL DEFAULT 'pending',
  triggered_by TEXT DEFAULT 'user'
);
```

### Commands & AI configuration

```sql
CREATE TABLE IF NOT EXISTS design_commands (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  name TEXT NOT NULL,
  aliases TEXT[] DEFAULT '{}',
  description TEXT,
  steps JSONB,
  is_system BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, name)
);

CREATE TABLE IF NOT EXISTS design_ai_config (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  use_glide_credits BOOLEAN DEFAULT true,
  model_style TEXT DEFAULT 'anthropic/claude-haiku-4-5',
  model_conditional TEXT DEFAULT 'anthropic/claude-haiku-4-5',
  model_layout TEXT DEFAULT 'anthropic/claude-sonnet-4-6',
  model_block TEXT DEFAULT 'anthropic/claude-sonnet-4-6',
  model_page TEXT DEFAULT 'anthropic/claude-sonnet-4-6',
  model_complex TEXT DEFAULT 'anthropic/claude-sonnet-4-6',
  workflow_authoring_model TEXT DEFAULT 'anthropic/claude-sonnet-4-6',
  workflow_execution_model TEXT DEFAULT 'anthropic/claude-haiku-4-5',
  workflow_suggestion_model TEXT DEFAULT 'anthropic/claude-haiku-4-5',
  auto_queue BOOLEAN DEFAULT true,
  updated_at TIMESTAMPTZ DEFAULT now()
);
INSERT INTO design_ai_config (org_id) VALUES ('default') ON CONFLICT DO NOTHING;
```

### Blocks & Pages

```sql
CREATE TABLE IF NOT EXISTS design_blocks (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  category_id TEXT NOT NULL,
  category_label TEXT NOT NULL,
  block_id TEXT NOT NULL,
  label TEXT NOT NULL,
  description TEXT,
  brief_template TEXT NOT NULL DEFAULT '',
  thumbnail_type TEXT NOT NULL DEFAULT 'generic',
  thumbnail_meta JSONB DEFAULT '{}',
  model_override TEXT,
  sort_order INTEGER DEFAULT 0,
  is_built_in BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, block_id)
);

CREATE TABLE IF NOT EXISTS design_pages (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  label TEXT NOT NULL,
  emoji TEXT DEFAULT '📄',
  path TEXT NOT NULL,
  file TEXT,
  preview_file TEXT,
  is_built_in BOOLEAN DEFAULT false,
  sort_order INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, path)
);
```

### Workflows

```sql
CREATE TABLE IF NOT EXISTS design_workflows (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  name TEXT NOT NULL,
  description TEXT,
  trigger_type TEXT NOT NULL DEFAULT 'user',
  trigger_config JSONB NOT NULL DEFAULT '{}',
  model TEXT NOT NULL DEFAULT 'anthropic/claude-haiku-4-5',
  webhook_token TEXT,
  enabled BOOLEAN NOT NULL DEFAULT false,
  created_by TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX IF NOT EXISTS design_workflows_webhook_token_idx ON design_workflows(webhook_token) WHERE webhook_token IS NOT NULL;

CREATE TABLE IF NOT EXISTS design_workflow_nodes (
  id BIGSERIAL PRIMARY KEY,
  workflow_id BIGINT REFERENCES design_workflows(id) ON DELETE CASCADE,
  org_id TEXT NOT NULL DEFAULT 'default',
  parent_id BIGINT REFERENCES design_workflow_nodes(id) ON DELETE CASCADE,
  branch TEXT,
  node_type TEXT NOT NULL,
  node_subtype TEXT NOT NULL,
  label TEXT,
  config JSONB NOT NULL DEFAULT '{}',
  position INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS design_workflow_runs (
  id BIGSERIAL PRIMARY KEY,
  workflow_id BIGINT REFERENCES design_workflows(id) ON DELETE CASCADE,
  org_id TEXT NOT NULL DEFAULT 'default',
  trigger TEXT NOT NULL,
  trigger_context JSONB,
  status TEXT NOT NULL DEFAULT 'running',
  started_at TIMESTAMPTZ DEFAULT now(),
  completed_at TIMESTAMPTZ,
  log JSONB NOT NULL DEFAULT '[]'
);

CREATE TABLE IF NOT EXISTS design_workflow_templates (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  name TEXT NOT NULL,
  description TEXT,
  category TEXT,
  trigger_type TEXT NOT NULL DEFAULT 'user',
  nodes JSONB NOT NULL DEFAULT '[]',
  tags TEXT[] DEFAULT '{}',
  is_built_in BOOLEAN DEFAULT false,
  created_by TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS design_workflow_suggestions (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  title TEXT NOT NULL,
  description TEXT,
  category TEXT NOT NULL,
  trigger_type TEXT NOT NULL,
  relevance_score REAL NOT NULL DEFAULT 0.5,
  nl_description TEXT NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  accepted_workflow_id BIGINT REFERENCES design_workflows(id) ON DELETE SET NULL,
  scan_id TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS design_app_context (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  tables JSONB NOT NULL DEFAULT '[]',
  routes JSONB NOT NULL DEFAULT '[]',
  pages JSONB NOT NULL DEFAULT '[]',
  raw_context TEXT,
  scanned_at TIMESTAMPTZ DEFAULT now()
);
```

### History & audit

```sql
CREATE TABLE IF NOT EXISTS design_workbench_history (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  source TEXT,
  action TEXT,
  title TEXT,
  before JSONB,
  after JSONB,
  changed_by TEXT,
  changed_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS design_agent_actions (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  action TEXT NOT NULL,
  params JSONB DEFAULT '{}',
  status TEXT NOT NULL DEFAULT 'pending',
  result TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  executed_at TIMESTAMPTZ
);

CREATE TABLE IF NOT EXISTS design_app_versions (
  id BIGSERIAL PRIMARY KEY,
  app_slug TEXT NOT NULL,
  version TEXT NOT NULL,
  label TEXT,
  deployed_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE IF NOT EXISTS design_runtime_errors (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  app_slug TEXT,
  version TEXT,
  error TEXT,
  route TEXT,
  kind TEXT,
  count INTEGER DEFAULT 1,
  first_seen TIMESTAMPTZ DEFAULT now(),
  last_seen TIMESTAMPTZ DEFAULT now(),
  recorded_at TIMESTAMPTZ DEFAULT now()
);
```

### Agent bridge tokens (required for /update command)

```sql
CREATE TABLE IF NOT EXISTS design_agent_bridge_tokens (
  id BIGSERIAL PRIMARY KEY,
  token TEXT UNIQUE NOT NULL,
  label TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  last_used_at TIMESTAMPTZ,
  revoked BOOLEAN DEFAULT false
);
```

### Console verify shadow row (Gap #2, workbench-side)

This is the WORKBENCH'S OWN copy of `design_console_actions` — a local UI
shadow row so `/app-api/console/action/:id` and
`/app-api/console/action/by-key/:actionKey` have something to poll. It is NOT
the canonical verify row; the installer (separate Neon DB, see below) owns
`expected_hashes` and the verdict.

```sql
CREATE TABLE IF NOT EXISTS design_console_actions (
  id BIGSERIAL PRIMARY KEY,
  action_key TEXT UNIQUE NOT NULL,
  org_id TEXT NOT NULL DEFAULT 'default',
  project_id TEXT,
  kind TEXT NOT NULL DEFAULT 'update',
  command TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  base_hashes JSONB DEFAULT '{}'::jsonb,
  expected_hashes JSONB DEFAULT '{}'::jsonb,
  observed_hashes JSONB DEFAULT '{}'::jsonb,
  token_hash TEXT,
  token_status TEXT DEFAULT 'active',
  token_expires_at TIMESTAMPTZ,
  instruction_ids BIGINT[],
  error TEXT,
  created_by TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

### Installer-side console verify table (MANUAL, out-of-band — Gap #2 REV7)

The public installer app (`workbench-install.glideapps.dev`) runs on a
**SEPARATE Neon database** from every installed workbench. The `/build`
agent cannot reach it with `db_execute` — this table must be created once, by
hand, against the INSTALLER's own DB (not the workbench's). It is the
CANONICAL verify row: the installer self-sources `expected_hashes` from bytes
it serves via its scaffold proxy (Gap #2 REV7 — no key, no push; see
`docs/plans/2026-07-04-rev7-console-verify.md`).

```sql
CREATE TABLE IF NOT EXISTS design_console_actions (
  id BIGSERIAL PRIMARY KEY,
  action_key TEXT UNIQUE NOT NULL,
  org_id TEXT NOT NULL DEFAULT 'default',
  project_id TEXT,
  kind TEXT NOT NULL DEFAULT 'update',
  command TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  base_hashes JSONB DEFAULT '{}'::jsonb,
  expected_hashes JSONB DEFAULT '{}'::jsonb,
  observed_hashes JSONB DEFAULT '{}'::jsonb,
  token_hash TEXT,
  token_status TEXT DEFAULT 'active',
  token_expires_at TIMESTAMPTZ,
  blob_keys JSONB DEFAULT '[]'::jsonb,
  error TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

ALTER TABLE design_agent_bridge_tokens ADD COLUMN IF NOT EXISTS org_id TEXT NOT NULL DEFAULT 'default';
```

### C3 auto-gate tables (Gap #2 REV7, §10.2 — INSTALLER Neon, MANUAL out-of-band, run ONCE)

These live on the **INSTALLER's** Neon DB alongside the installer-side
`design_console_actions` above — NOT the workbench's. The installer (not the
`/build` agent) triggers the off-platform GitHub Action, owns the verdict row,
and receives the GHA callback. `target_hash` + `prod_url` are release-recorded
from the public manifest, NEVER agent-supplied; `callback_nonce` is
installer-minted and echoed by the GHA on callback (two-factor with the
`C3_CALLBACK_SECRET` env). The agent may only READ the verdict via
`/console/c3/status` to relay it — it never writes verdict, so it cannot
fabricate a pass.

```sql
CREATE TABLE IF NOT EXISTS design_c3_verdicts (
  id BIGSERIAL PRIMARY KEY,
  action_key TEXT NOT NULL,
  org_id TEXT NOT NULL DEFAULT 'default',
  target_hash TEXT NOT NULL,
  prod_url TEXT NOT NULL,
  gh_run_id BIGINT,
  verdict TEXT NOT NULL DEFAULT 'pending',  -- pending|pass|fail|error|timeout
  observed_hash TEXT,
  attempts INT NOT NULL DEFAULT 0,
  callback_nonce TEXT NOT NULL,
  error TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (action_key, org_id)
);
```

```sql
-- design_gha_rl — atomic fixed-window rate limiter for the two GHA-triggering
-- endpoints. One guarded upsert per call (ON CONFLICT DO UPDATE ... WHERE hits
-- < limit) increments only if under cap, so concurrent callers cannot overshoot
-- (no count-then-insert race). Buckets:
--   • update/begin — per-(org_id, project_id), hourly (headers present here).
--   • c3/trigger   — GLOBAL (org_id/project_id collapsed to '*'), daily; the
--     true spend backstop, since c3/trigger carries only actionKey + token.
-- The worker self-heals this schema on undefined_table (so a code deploy before
-- this migration cannot wedge updates at 503) and opportunistically prunes
-- buckets older than 2 days. The PK makes every check a point-write, not a scan.
CREATE TABLE IF NOT EXISTS design_gha_rl (
  endpoint   TEXT NOT NULL CHECK (endpoint IN ('update/begin','c3/trigger')),
  org_id     TEXT NOT NULL DEFAULT '*',
  project_id TEXT NOT NULL DEFAULT '*',
  bucket     TIMESTAMPTZ NOT NULL,
  hits       INT NOT NULL DEFAULT 0,
  PRIMARY KEY (endpoint, org_id, project_id, bucket)
);
-- Supports the worker's opportunistic prune (WHERE bucket < …); the PK does not.
CREATE INDEX IF NOT EXISTS design_gha_rl_bucket_idx ON design_gha_rl (bucket);
```

### Installer secrets for the C3 auto-gate (§10.5 — set once on the INSTALLER app)

The installer app needs two secrets for the C3 auto-gate. Set them on the
`design-workbench-installer` app (NOT this project), and set the two GHA repo
secrets on the PRIVATE repo:

- `C3_GH_PAT` (installer env) — a **fine-grained** GitHub PAT: single repo
  `bschonbrun/GlideOS-Design-Workbench`, **Actions: Read and write** only, no
  contents scope. Narrower than `GLIDEOS_ACCESS_TOKEN`. Used only to
  `workflow_dispatch` the deploy-fingerprint-check workflow.
- `C3_CALLBACK_SECRET` (installer env AND GitHub repo secret) — a random shared
  secret. The GHA presents it as `x-c3-callback-token`; the installer
  constant-time-compares it (with the per-run `callback_nonce` as second factor)
  before accepting a verdict callback.
- `C3_ALLOWED_HOST` (GitHub repo secret) — the exact production host of the
  workbench (host only, no scheme/path). The workflow rejects any `prod_url`
  whose host differs (fail-closed positive allowlist, B4).

### Admin + install gating tables

```sql
CREATE TABLE IF NOT EXISTS design_admins (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  email TEXT NOT NULL,
  added_by TEXT NOT NULL DEFAULT 'system',
  added_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, email)
);

CREATE TABLE IF NOT EXISTS design_install_allowlist (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  email TEXT NOT NULL,
  approved_by TEXT,
  approved_at TIMESTAMPTZ DEFAULT now(),
  note TEXT,
  UNIQUE (org_id, email)
);

CREATE TABLE IF NOT EXISTS design_counter_health (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  kind TEXT NOT NULL,
  severity TEXT NOT NULL DEFAULT 'warning',
  message TEXT NOT NULL,
  context JSONB,
  created_at TIMESTAMPTZ DEFAULT now(),
  acknowledged_at TIMESTAMPTZ
);
```

### v2.2.1 orientation & log mirror tables

```sql
CREATE TABLE IF NOT EXISTS design_log_mirror (
  log_id BIGINT PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  trace_id TEXT,
  actor TEXT,
  source TEXT,
  message TEXT,
  detail JSONB,
  created_at TIMESTAMPTZ NOT NULL,
  mirrored_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX IF NOT EXISTS design_log_mirror_trace_idx ON design_log_mirror(trace_id);
CREATE INDEX IF NOT EXISTS design_log_mirror_created_idx ON design_log_mirror(created_at DESC);
COMMENT ON TABLE design_log_mirror IS 'v2.2.1: agent-mirrored snapshot of GlideOS tool-call audit log. Counter derives from COUNT(DISTINCT trace_id).';

CREATE TABLE IF NOT EXISTS design_orientation_signals (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  session_id TEXT,
  kind TEXT NOT NULL,
  source_endpoint TEXT,
  payload JSONB,
  created_at TIMESTAMPTZ DEFAULT now(),
  consumed_at TIMESTAMPTZ
);
CREATE INDEX IF NOT EXISTS design_orientation_signals_unread_idx ON design_orientation_signals(org_id, created_at) WHERE consumed_at IS NULL;
COMMENT ON TABLE design_orientation_signals IS 'v2.2.1: breadcrumbs left by Workbench POST handlers. Agent drains at orientation time.';
```

### v2.4.0 Setup system tables

```sql
CREATE TABLE IF NOT EXISTS design_setup_items (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  item_key TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  category TEXT NOT NULL DEFAULT 'general',
  status TEXT NOT NULL DEFAULT 'not_started',
  status_note TEXT,
  action_label TEXT,
  action_type TEXT,
  action_target TEXT,
  wizard_key TEXT,
  is_required BOOLEAN DEFAULT true,
  sort_order INTEGER DEFAULT 0,
  auto_detect BOOLEAN DEFAULT false,
  last_checked TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, item_key)
);
COMMENT ON TABLE design_setup_items IS 'v2.4.0: one row per setup checklist item. New features INSERT ON CONFLICT DO NOTHING. Status: not_started | in_progress | done | warning | skipped.';

CREATE TABLE IF NOT EXISTS design_wizards (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  wizard_key TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  category TEXT NOT NULL DEFAULT 'general',
  steps JSONB NOT NULL DEFAULT '[]',
  is_active BOOLEAN DEFAULT true,
  sort_order INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, wizard_key)
);
COMMENT ON TABLE design_wizards IS 'v2.4.0: registry of topic wizards (AI models, GitHub, etc). Steps are JSONB array of wizard step definitions.';

CREATE TABLE IF NOT EXISTS design_wizard_state (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  wizard_key TEXT NOT NULL,
  answers JSONB DEFAULT '{}',
  current_step INTEGER DEFAULT 0,
  status TEXT NOT NULL DEFAULT 'not_started',
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, wizard_key)
);
COMMENT ON TABLE design_wizard_state IS 'v2.4.0: last-known answers per wizard per org. Pre-fills re-runs and resumes from where user left off.';
```

### v2.3 heal block — idempotent migration for existing installs

Existing installs may be missing columns. Run this block on every install (fresh or upgrade) — it's a no-op when already current:

```sql
ALTER TABLE design_orgs              ADD COLUMN IF NOT EXISTS proxy_paths      TEXT[]      NOT NULL DEFAULT '{}';
ALTER TABLE design_install_allowlist ADD COLUMN IF NOT EXISTS approved_at      TIMESTAMPTZ NOT NULL DEFAULT now();
ALTER TABLE design_counter_health    ADD COLUMN IF NOT EXISTS acknowledged_at  TIMESTAMPTZ;
```

### Seed installer as admin + allowlist

The installing user (the `x-glide-user-email` header value at install time — let's call it `:installer_email`) must be in both `design_admins` and `design_install_allowlist` or every admin tab returns "Forbidden". Run after the heal block:

```sql
INSERT INTO design_admins (org_id, email, added_by)
VALUES ('default', :installer_email, 'install')
ON CONFLICT (org_id, email) DO NOTHING;

INSERT INTO design_install_allowlist (org_id, email, approved_by, approved_at)
VALUES ('default', :installer_email, 'install', now())
ON CONFLICT (org_id, email) DO NOTHING;
```

---

## Step 2c — Create remaining tables (missing from Step 2)

These tables are referenced by `server.ts` but were not in the Step 2 DDL. Run each block — all use `IF NOT EXISTS` so re-runs are safe.

### AI presets, task models, providers, model catalogs

```sql
CREATE TABLE IF NOT EXISTS design_ai_presets (
  id           BIGSERIAL PRIMARY KEY,
  org_id       TEXT NOT NULL DEFAULT 'default',
  preset_id    TEXT NOT NULL,
  label        TEXT NOT NULL,
  description  TEXT,
  icon         TEXT,
  task_defaults JSONB NOT NULL DEFAULT '{}',
  sort_order   INTEGER DEFAULT 0,
  is_built_in  BOOLEAN DEFAULT true,
  provider_priority TEXT[] NOT NULL DEFAULT '{anthropic,deepseek,openai,google-ai-studio}',
  UNIQUE (org_id, preset_id)
);
COMMENT ON TABLE design_ai_presets IS 'Named AI configuration presets (Fast, Balanced, etc). task_defaults is a map of task_id → {our_model, glide_model, glide_modifier}.';

CREATE TABLE IF NOT EXISTS design_task_models (
  org_id        TEXT NOT NULL DEFAULT 'default',
  task_id       TEXT NOT NULL,
  label         TEXT NOT NULL,
  description   TEXT,
  our_model     TEXT NOT NULL DEFAULT 'anthropic/claude-haiku-4-5',
  our_source    TEXT NOT NULL DEFAULT 'glide_credits',
  glide_model   TEXT NOT NULL DEFAULT 'anthropic/claude-haiku-4-5',
  glide_modifier TEXT NOT NULL DEFAULT 'balanced',
  PRIMARY KEY (org_id, task_id)
);
COMMENT ON TABLE design_task_models IS 'Per-task AI model configuration. our_model = Workbench pre-processing model; glide_model = suggested GlideOS handoff model.';

CREATE TABLE IF NOT EXISTS design_providers (
  id           TEXT PRIMARY KEY,
  label        TEXT NOT NULL,
  description  TEXT,
  key_page_url TEXT,
  secret_name  TEXT NOT NULL,
  logo_domain  TEXT,
  is_local     BOOLEAN DEFAULT false,
  enabled      BOOLEAN DEFAULT true,
  sort_order   INTEGER DEFAULT 99,
  created_at   TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_providers IS 'AI provider catalog. secret_name is the GlideOS project secret that holds the API key.';

CREATE TABLE IF NOT EXISTS design_our_models (
  id           TEXT PRIMARY KEY,
  provider     TEXT NOT NULL,
  tier         TEXT NOT NULL,
  label        TEXT NOT NULL,
  description  TEXT,
  is_default   BOOLEAN DEFAULT false,
  enabled      BOOLEAN DEFAULT true,
  sort_order   INTEGER DEFAULT 99,
  updated_at   TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_our_models IS 'Models the Workbench itself can use for pre-processing (own-API-key mode).';

CREATE TABLE IF NOT EXISTS design_glide_models (
  id           TEXT PRIMARY KEY,
  provider     TEXT NOT NULL,
  tier         TEXT NOT NULL,
  label        TEXT NOT NULL,
  description  TEXT,
  is_default   BOOLEAN DEFAULT false,
  enabled      BOOLEAN DEFAULT true,
  sort_order   INTEGER DEFAULT 99,
  updated_at   TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_glide_models IS 'GlideOS platform model catalog — the models users can reference for the GlideOS handoff column.';
```

### Install management, file cache, scaffold templates

```sql
CREATE TABLE IF NOT EXISTS design_install_log (
  id             BIGSERIAL PRIMARY KEY,
  org_id         TEXT NOT NULL DEFAULT 'default',
  installer_email TEXT NOT NULL,
  project_id     TEXT,
  project_name   TEXT,
  invite_code_used TEXT,
  installed_at   TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_install_log IS 'One row per completed Workbench install.';

CREATE TABLE IF NOT EXISTS design_install_requests (
  id           BIGSERIAL PRIMARY KEY,
  org_id       TEXT NOT NULL DEFAULT 'default',
  email        TEXT NOT NULL,
  status       TEXT NOT NULL DEFAULT 'pending',
  requested_at TIMESTAMPTZ DEFAULT now(),
  reviewed_at  TIMESTAMPTZ,
  reviewed_by  TEXT,
  note         TEXT
);
COMMENT ON TABLE design_install_requests IS 'Legacy email-based install access requests. New installs use license_requests table instead.';

CREATE TABLE IF NOT EXISTS design_invite_codes (
  id         BIGSERIAL PRIMARY KEY,
  org_id     TEXT NOT NULL DEFAULT 'default',
  code       TEXT UNIQUE NOT NULL,
  label      TEXT,
  max_uses   INTEGER DEFAULT 1,
  uses       INTEGER DEFAULT 0,
  revoked    BOOLEAN DEFAULT false,
  expires_at TIMESTAMPTZ,
  created_by TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_invite_codes IS 'Single-use or limited-use invite codes for bypassing the access request flow.';

CREATE TABLE IF NOT EXISTS design_file_cache (
  id         BIGSERIAL PRIMARY KEY,
  org_id     TEXT NOT NULL DEFAULT 'default',
  path       TEXT NOT NULL,
  content    TEXT NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, path)
);
COMMENT ON TABLE design_file_cache IS 'Cached workspace file content used by the GitHub push bridge.';

CREATE TABLE IF NOT EXISTS design_scaffold_templates (
  id        BIGSERIAL PRIMARY KEY,
  file_name TEXT NOT NULL UNIQUE,
  content   TEXT NOT NULL,
  version   TEXT,
  updated_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_scaffold_templates IS 'Canonical versions of scaffold files (AGENTS.md, PROJECT_NOTES.md, PREFERENCES.md). Keeps new installs in sync.';

CREATE TABLE IF NOT EXISTS design_push_log (
  id          BIGSERIAL PRIMARY KEY,
  org_id      TEXT NOT NULL DEFAULT 'default',
  pushed_by   TEXT,
  commit_sha  TEXT,
  files_pushed TEXT[] DEFAULT '{}',
  message     TEXT,
  pushed_at   TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_push_log IS 'GitHub push history — one row per successful commit.';
```

### Org infrastructure tables

```sql
CREATE TABLE IF NOT EXISTS design_org_members (
  id       BIGSERIAL PRIMARY KEY,
  org_id   TEXT NOT NULL DEFAULT 'default',
  email    TEXT NOT NULL,
  role     TEXT NOT NULL DEFAULT 'member',
  added_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, email)
);
COMMENT ON TABLE design_org_members IS 'Workbench org members and their roles.';

CREATE TABLE IF NOT EXISTS design_org_connections (
  id           BIGSERIAL PRIMARY KEY,
  org_id       TEXT NOT NULL DEFAULT 'default',
  provider     TEXT NOT NULL,
  config       JSONB DEFAULT '{}',
  created_at   TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_org_connections IS 'External service connections (Slack, GitHub, webhooks, etc).';

CREATE TABLE IF NOT EXISTS design_connected_apps (
  id           BIGSERIAL PRIMARY KEY,
  org_id       TEXT NOT NULL DEFAULT 'default',
  app_slug     TEXT NOT NULL,
  app_name     TEXT,
  app_url      TEXT,
  proxy_paths  TEXT[] DEFAULT '{}',
  data_source  TEXT DEFAULT 'neon',
  is_primary   BOOLEAN DEFAULT false,
  created_at   TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, app_slug)
);
COMMENT ON TABLE design_connected_apps IS 'Apps connected to this Workbench install for preview.';

CREATE TABLE IF NOT EXISTS design_app_domains (
  id           BIGSERIAL PRIMARY KEY,
  org_id       TEXT NOT NULL,
  app_slug     TEXT NOT NULL,
  app_name     TEXT,
  hostname     TEXT NOT NULL,
  cname_target TEXT,
  status       TEXT DEFAULT 'pending',
  created_at   TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, hostname)
);
COMMENT ON TABLE design_app_domains IS 'Custom domains bound to deployed apps.';
```

### Analytics & telemetry

```sql
CREATE TABLE IF NOT EXISTS design_credit_snapshots (
  id           BIGSERIAL PRIMARY KEY,
  org_id       TEXT NOT NULL DEFAULT 'default',
  credits_used DOUBLE PRECISION,
  credits_remaining DOUBLE PRECISION,
  quota        DOUBLE PRECISION,
  snapped_at   TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_credit_snapshots IS 'Periodic GlideOS credit balance snapshots.';

CREATE TABLE IF NOT EXISTS design_token_snapshots (
  id           BIGSERIAL PRIMARY KEY,
  org_id       TEXT NOT NULL DEFAULT 'default',
  model        TEXT,
  input_tokens BIGINT DEFAULT 0,
  output_tokens BIGINT DEFAULT 0,
  snapped_at   TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_token_snapshots IS 'Periodic own-key token usage snapshots by model.';

CREATE TABLE IF NOT EXISTS design_workflow_scan_log (
  id          BIGSERIAL PRIMARY KEY,
  org_id      TEXT NOT NULL DEFAULT 'default',
  scan_id     TEXT,
  tables_found INTEGER DEFAULT 0,
  routes_found INTEGER DEFAULT 0,
  suggestions_created INTEGER DEFAULT 0,
  scanned_at  TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_workflow_scan_log IS 'History of workflow suggestion scans.';

CREATE TABLE IF NOT EXISTS design_workflow_drafts (
  id          BIGSERIAL PRIMARY KEY,
  org_id      TEXT NOT NULL DEFAULT 'default',
  name        TEXT NOT NULL,
  nodes       JSONB NOT NULL DEFAULT '[]',
  created_at  TIMESTAMPTZ DEFAULT now(),
  updated_at  TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_workflow_drafts IS 'Unsaved / in-progress workflow drafts.';
```

### Heal block — missing columns on existing tables

Run this on every install (fresh or upgrade) — safe to re-run.

**Important:** this block must run BEFORE the seed INSERTs below. `CREATE TABLE IF NOT EXISTS` silently no-ops if a table already exists from a prior partial install, leaving it with a stale schema. These `ALTER TABLE … ADD COLUMN IF NOT EXISTS` statements bring any such table up to the current spec before seeds run.

```sql
-- design_ai_config: missing columns
ALTER TABLE design_ai_config ADD COLUMN IF NOT EXISTS anthropic_key_hint TEXT;
ALTER TABLE design_ai_config ADD COLUMN IF NOT EXISTS openai_key_hint TEXT;
ALTER TABLE design_ai_config ADD COLUMN IF NOT EXISTS google_key_hint TEXT;
ALTER TABLE design_ai_config ADD COLUMN IF NOT EXISTS deepseek_key_hint TEXT;
ALTER TABLE design_ai_config ADD COLUMN IF NOT EXISTS confirm_flagship BOOLEAN DEFAULT false;
ALTER TABLE design_ai_config ADD COLUMN IF NOT EXISTS glide_allowed_models JSONB DEFAULT '{}';

-- design_instructions: missing column
ALTER TABLE design_instructions ADD COLUMN IF NOT EXISTS used_glide_credits BOOLEAN;

-- design_orgs: missing columns
ALTER TABLE design_orgs ADD COLUMN IF NOT EXISTS custom_domain TEXT;
ALTER TABLE design_orgs ADD COLUMN IF NOT EXISTS domain_status TEXT;
ALTER TABLE design_orgs ADD COLUMN IF NOT EXISTS domain_cname_target TEXT;
ALTER TABLE design_orgs ADD COLUMN IF NOT EXISTS credit_poll_minutes INTEGER DEFAULT 60;

-- design_target_routes: ensure full v2 schema columns exist
ALTER TABLE design_target_routes ADD COLUMN IF NOT EXISTS description TEXT;
ALTER TABLE design_target_routes ADD COLUMN IF NOT EXISTS updated_at TIMESTAMPTZ DEFAULT now();

-- design_glide_models: v1 installs missing label/enabled/updated_at columns
ALTER TABLE design_glide_models ADD COLUMN IF NOT EXISTS label TEXT;
ALTER TABLE design_glide_models ADD COLUMN IF NOT EXISTS enabled BOOLEAN DEFAULT true;
ALTER TABLE design_glide_models ADD COLUMN IF NOT EXISTS updated_at TIMESTAMPTZ DEFAULT now();
UPDATE design_glide_models SET label = id WHERE label IS NULL;

-- design_task_models: v1 installs had single `model` column instead of four-column split
-- Neutralise the legacy NOT NULL so the new-column seeds don't conflict
ALTER TABLE design_task_models ALTER COLUMN model SET DEFAULT 'anthropic/claude-haiku-4-5';
ALTER TABLE design_task_models ADD COLUMN IF NOT EXISTS our_model TEXT NOT NULL DEFAULT 'anthropic/claude-haiku-4-5';
ALTER TABLE design_task_models ADD COLUMN IF NOT EXISTS our_source TEXT NOT NULL DEFAULT 'glide_credits';
ALTER TABLE design_task_models ADD COLUMN IF NOT EXISTS glide_model TEXT NOT NULL DEFAULT 'anthropic/claude-haiku-4-5';
ALTER TABLE design_task_models ADD COLUMN IF NOT EXISTS glide_modifier TEXT NOT NULL DEFAULT 'balanced';
```

### Seed providers and model catalogs

```sql
INSERT INTO design_providers (id, label, description, key_page_url, secret_name, logo_domain, sort_order) VALUES
('anthropic',      'Anthropic',       'Claude models — Haiku, Sonnet, Opus.',            'https://console.anthropic.com/account/keys',  'ANTHROPIC_API_KEY', 'anthropic.com',    1),
('openai',         'OpenAI',          'GPT models — GPT-4o, GPT-5 and above.',           'https://platform.openai.com/api-keys',        'OPENAI_API_KEY',    'openai.com',       2),
('google-ai-studio','Google AI Studio','Gemini models — Flash, Pro. Google AI Studio.', 'https://aistudio.google.com/app/apikey',      'GOOGLE_API_KEY',    'google.com',       3),
('deepseek',       'DeepSeek',        'DeepSeek V4 Flash + Pro. Very cost-effective.',   'https://platform.deepseek.com/api_keys',      'DEEPSEEK_API_KEY',  'deepseek.com',     4)
ON CONFLICT (id) DO NOTHING;

INSERT INTO design_our_models (id, provider, tier, label, description, is_default, sort_order) VALUES
('anthropic/claude-haiku-4-5',          'anthropic',       'fast',     'Claude Haiku 4.5',    'Fastest Anthropic model.',                      false, 1),
('anthropic/claude-sonnet-4-6',         'anthropic',       'balanced', 'Claude Sonnet 4.6',   'Best speed/intelligence balance from Anthropic.',true,  2),
('anthropic/claude-opus-4-7',           'anthropic',       'flagship', 'Claude Opus 4.7',     'Most capable Anthropic model.',                 false, 3),
('openai/gpt-5.4-mini',                 'openai',          'fast',     'GPT-5.4 Mini',        'OpenAI high-volume mini.',                      false, 4),
('openai/gpt-5.5',                      'openai',          'flagship', 'GPT-5.5',             'OpenAI flagship model.',                        false, 5),
('google-ai-studio/gemini-3.1-flash-lite','google-ai-studio','flash',  'Gemini 3.1 Flash Lite','Cheapest Gemini 3.1.',                         false, 6),
('google-ai-studio/gemini-3-flash-preview','google-ai-studio','flash', 'Gemini 3 Flash',      'Low-latency Flash model.',                      false, 7),
('google-ai-studio/gemini-3.1-pro-preview','google-ai-studio','pro',  'Gemini 3.1 Pro',      'Most advanced Gemini.',                         false, 8),
('deepseek/deepseek-v4-flash',          'deepseek',        'fast',     'DeepSeek V4 Flash',   'Fast, 1M context. $0.28/M output.',             false, 9),
('deepseek/deepseek-v4-pro',            'deepseek',        'flagship', 'DeepSeek V4 Pro',     'Most capable DeepSeek, 1M context.',            false, 10)
ON CONFLICT (id) DO NOTHING;

INSERT INTO design_glide_models (id, provider, tier, label, description, is_default, sort_order) VALUES
('anthropic/claude-haiku-4-5',           'anthropic',       'fast',     'Claude Haiku 4.5',    'Fastest Anthropic model.',                      false, 1),
('anthropic/claude-sonnet-4-6',          'anthropic',       'balanced', 'Claude Sonnet 4.6',   'Best speed/intelligence balance.',              true,  2),
('anthropic/claude-opus-4-7',            'anthropic',       'flagship', 'Claude Opus 4.7',     'Most capable Anthropic model.',                 false, 3),
('openai/gpt-5.4-mini',                  'openai',          'fast',     'GPT-5.4 Mini',        'OpenAI high-volume mini.',                      false, 4),
('openai/gpt-5.5',                       'openai',          'flagship', 'GPT-5.5',             'OpenAI flagship model.',                        false, 5),
('google-ai-studio/gemini-3.1-flash-lite','google-ai-studio','flash',  'Gemini 3.1 Flash Lite','Cheapest Gemini 3.1.',                         false, 6),
('google-ai-studio/gemini-3-flash-preview','google-ai-studio','flash', 'Gemini 3 Flash',      'Low-latency Flash model.',                      false, 7),
('google-ai-studio/gemini-3.1-pro-preview','google-ai-studio','pro',  'Gemini 3.1 Pro',      'Most advanced Gemini.',                         false, 8),
('deepseek/deepseek-v4-flash',           'deepseek',        'fast',     'DeepSeek V4 Flash',   'Fast, 1M context.',                             false, 9),
('deepseek/deepseek-v4-pro',             'deepseek',        'flagship', 'DeepSeek V4 Pro',     'Most capable DeepSeek.',                        false, 10)
ON CONFLICT (id) DO UPDATE SET
  provider = EXCLUDED.provider, tier = EXCLUDED.tier, label = EXCLUDED.label,
  description = EXCLUDED.description, is_default = EXCLUDED.is_default, sort_order = EXCLUDED.sort_order;

INSERT INTO design_ai_presets (org_id, preset_id, label, icon, task_defaults, sort_order) VALUES
('default', 'fast',     'Fast & cheap',    '⚡', '{"style":{"our_model":"anthropic/claude-haiku-4-5","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"conditional_logic":{"our_model":"anthropic/claude-haiku-4-5","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"layout":{"our_model":"anthropic/claude-haiku-4-5","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"workflows":{"our_model":"anthropic/claude-haiku-4-5","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"data_binding":{"our_model":"anthropic/claude-haiku-4-5","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"ai_suggestions":{"our_model":"google-ai-studio/gemini-3-flash-preview","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"instruction_formatting":{"our_model":"anthropic/claude-haiku-4-5","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"}}', 1),
('default', 'balanced', 'Balanced',        '⚖️', '{"style":{"our_model":"anthropic/claude-sonnet-4-6","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"balanced"},"conditional_logic":{"our_model":"anthropic/claude-sonnet-4-6","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"balanced"},"layout":{"our_model":"anthropic/claude-sonnet-4-6","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"balanced"},"workflows":{"our_model":"anthropic/claude-sonnet-4-6","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"balanced"},"data_binding":{"our_model":"anthropic/claude-sonnet-4-6","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"balanced"},"ai_suggestions":{"our_model":"google-ai-studio/gemini-3-flash-preview","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"balanced"},"instruction_formatting":{"our_model":"anthropic/claude-sonnet-4-6","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"balanced"}}', 2),
('default', 'cheap',    'Light & cheap',   '💰', '{"style":{"our_model":"google-ai-studio/gemini-3.1-flash-lite","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"conditional_logic":{"our_model":"google-ai-studio/gemini-3.1-flash-lite","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"layout":{"our_model":"google-ai-studio/gemini-3.1-flash-lite","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"workflows":{"our_model":"google-ai-studio/gemini-3.1-flash-lite","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"data_binding":{"our_model":"google-ai-studio/gemini-3.1-flash-lite","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"ai_suggestions":{"our_model":"google-ai-studio/gemini-3.1-flash-lite","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"},"instruction_formatting":{"our_model":"google-ai-studio/gemini-3.1-flash-lite","glide_model":"anthropic/claude-haiku-4-5","glide_modifier":"fast"}}', 3),
('default', 'custom',   'Custom',          '🎯', '{}', 4)
ON CONFLICT (org_id, preset_id) DO NOTHING;

INSERT INTO design_task_models (org_id, task_id, label, description, our_model, our_source, glide_model, glide_modifier) VALUES
('default', 'style',               'Style & appearance',  'Color, font, size, spacing changes',             'anthropic/claude-sonnet-4-6',             'glide_credits', 'anthropic/claude-haiku-4-5', 'balanced'),
('default', 'conditional_logic',   'Conditional logic',   'Show/hide rules, if/then based on data',         'anthropic/claude-sonnet-4-6',             'glide_credits', 'anthropic/claude-haiku-4-5', 'balanced'),
('default', 'layout',              'Layout & structure',  'Moving, reordering, adding/removing sections',   'anthropic/claude-sonnet-4-6',             'glide_credits', 'anthropic/claude-haiku-4-5', 'balanced'),
('default', 'workflows',           'Workflow automation', 'Triggers, conditions, actions, notifications',   'anthropic/claude-sonnet-4-6',             'glide_credits', 'anthropic/claude-haiku-4-5', 'balanced'),
('default', 'data_binding',        'Data binding',        'Connecting fields to tables, computed values',   'anthropic/claude-sonnet-4-6',             'glide_credits', 'anthropic/claude-haiku-4-5', 'balanced'),
('default', 'ai_suggestions',      'AI suggestions',      'Background schema scanning, idea generation',    'google-ai-studio/gemini-3-flash-preview', 'glide_credits', 'anthropic/claude-haiku-4-5', 'balanced'),
('default', 'instruction_formatting','Instruction formatting','Format and clarify build instructions',      'anthropic/claude-sonnet-4-6',             'glide_credits', 'anthropic/claude-haiku-4-5', 'balanced')
ON CONFLICT (org_id, task_id) DO UPDATE SET
  label = EXCLUDED.label, description = EXCLUDED.description,
  our_model = EXCLUDED.our_model, our_source = EXCLUDED.our_source,
  glide_model = EXCLUDED.glide_model, glide_modifier = EXCLUDED.glide_modifier;
```

---

## Step 2b — Initialize the setup checklist

Seed the setup items that will track install progress in real-time. These start as `not_started` and get updated as each install step completes. This is what populates the Setup rail panel in the Workbench.

```sql
INSERT INTO design_setup_items (org_id, item_key, title, description, category, status, is_required, sort_order) VALUES
('default', 'project_created',           'GlideOS project created',         'The GlideOS project this Workbench runs in.',                                            'core',         'done',        true, 0),
('default', 'db_schema',                 'Database schema installed',        'All Workbench tables created and heal migrations applied.',                               'core',         'in_progress', true, 1),
('default', 'workbench_deployed',        'Design Workbench deployed',        'The Workbench worker is deployed and published to production.',                           'core',         'not_started', true, 2),
('default', 'workbench_licensed',        'Workbench licensed',               'A valid license is active for this project (required to queue builds).',                  'core',         'done',        true, 3),
('default', 'target_app_connected',      'Target app connected',             'A target app URL is stored in design_orgs.connected_app_url.',                           'core',         'not_started', true, 4),
('default', 'preview_pages_configured', 'Preview pages configured',         'Preview pages exist under preview/pages/ and are registered in design_pages.',            'core',         'not_started', true, 5),
('default', 'ai_providers_configured',    'AI providers configured',          'At least one AI provider API key is set in GlideOS Secrets (Anthropic, OpenAI, Google, DeepSeek).', 'ai', 'not_started', true,  100),
('default', 'ai_preferences_set',        'AI model preferences set',         'Task models configured. GlideOS credits or own-key mode chosen.',                        'ai',           'not_started', false, 110),
('default', 'github_connected',          'GitHub sync enabled',              'Optional. Add GLIDEOS_ACCESS_TOKEN (GitHub classic PAT, repo scope) as a GlideOS project secret to enable /git-push (push project files back to GitHub). Not required for /update — the Workbench installer handles that automatically.', 'integrations', 'not_started', false, 200),
('default', 'resend_configured',         'Resend API key configured',        'RESEND_API_KEY secret set. Required for install access-request notification emails.',    'integrations', 'not_started', false, 210)
ON CONFLICT (org_id, item_key) DO NOTHING;
```

Now mark the schema step as done (it just completed):

```sql
UPDATE design_setup_items
SET status = 'done', completed_at = now(), status_note = 'Schema created by install agent', updated_at = now()
WHERE org_id = 'default' AND item_key = 'db_schema';
```

---

## Step 3 — Write the agent docs (inline — NOT via the proxy)

The scaffold proxy only serves **pinned** paths (the keys in `manifest-hashes.json`, i.e. `manifest.app_files`). The agent docs are per-install and NOT pinned, so the proxy returns **400** for them — do not fetch them. Write each one directly with `file_write`, using the exact content below.

**3a. `AGENTS.md`** — the thin STEP 0 pointer. `file_write` this verbatim:

~~~md
# Design Workbench — Agent Rules

## STEP 0 — before anything else, every session

1. **`file_read` `agents-core.md` and obey ALL of it.** It is the source of truth for how this Workbench behaves — session start, the per-turn turn counter, the pending-build check, the update check, and the slash-command loader all live there. Do this before responding, even to a slash command.
2. **Load your slash commands** — even if step 1 fails: `SELECT name, aliases, description, steps FROM design_commands WHERE org_id = 'default' ORDER BY created_at ASC`. Each row is a live command. This guarantees `/repair`, `/update`, `/build` work even if `agents-core.md` is missing.
3. Read `PROJECT_NOTES.md` for project memory.

If `agents-core.md` is missing: still do step 2, then tell the user "⚠️ Workbench rules missing — type `/repair` to restore them."

Do not copy the core rules here — keep this thin so `/update` never rewrites AGENTS.md. Add PROJECT-SPECIFIC rules below the marker.

---

<!-- ▼▼▼ PROJECT-SPECIFIC RULES BELOW THIS LINE — never touched by /update. Append freely. ▼▼▼ -->
~~~

**3b. `PREFERENCES.md`** — default preferences. `file_write` this verbatim (the DB `design_preferences` row is authoritative; this is the fallback):

~~~md
# Design Workbench — Agent Preferences

Read at session start. The DB `design_preferences` row overrides this file.

## Communication style
**Reply length:** balanced (~120 words)
**Model effort:** balanced
**Explanation verbosity:** normal

## User profile
**Name:** (not set)
**Role:** (not set)
**Technical level:** non-technical
~~~

**3c. `PROJECT_NOTES.md`** — starter notes. `file_write` this verbatim; you maintain this file going forward (a fresh install starts clean — it does NOT inherit any other project's notes):

~~~md
# Project Notes

_Project memory, architecture decisions, and build history. Update this file freely as the project grows._

## Bootstrap
Installed via the Design Workbench installer. Agent rules live in `agents-core.md` (loaded via the STEP 0 pointer in AGENTS.md). Slash commands live in the `design_commands` table.

## Build history
_(empty — first install)_
~~~

**3d. `github/manifest.json`** — `file_write` the manifest JSON you already fetched in Step 1 to `github/manifest.json` (the GitHub-push panel reads it). Do not fetch it from the proxy.

`agents-core.md` (the managed rulebook) is an `app_file` — it is delivered with the rest of the app source in Step 4 via the pinned proxy, not here.

---

## Step 4 — Fetch app source files

For each file listed in `manifest.app_files`, fetch it the same way:

```
GET https://workbench-install.glideapps.dev/app-api/scaffold/{path}
x-wb-org-id:     {installer_org_id}
x-wb-project-id: {installer_project_id}
x-wb-email:      {installer_email}
```

Deliver **every** entry in `manifest.app_files` using the **"Delivering files"** procedure below (client-side chunked fetch + per-file sha256 verify against `manifest-hashes.json`). `agents-core.md` and the large `LayoutCanvas.tsx` are committed, pinned, and served like any other file — deliver them for real. (Source is no longer baked into `source-bundle*.ts`; those bundles are gone. The running app self-populates its file cache from the pin on first boot — see the app's own seed flow — so a fresh install does not need to hand-cache source.)

> **Do NOT single-fetch** a large file. Any file > 15000 bytes must use the chunk loop below — `LayoutCanvas.tsx` is the notable one — because a single fetch-and-`file_write` silently truncates it.

After all app source files are written and verified, mark the workbench_deployed item as in-progress:

```sql
UPDATE design_setup_items
SET status = 'in_progress', status_note = 'Scaffold files written — deploy pending', updated_at = now()
WHERE org_id = 'default' AND item_key = 'workbench_deployed';
```

---

## Delivering files (client-side chunked assembly + hash verification)

Applies to **every** file you write from the scaffold proxy — all of `manifest.app_files`, and the docs in Step 3. The large `LayoutCanvas.tsx` is far bigger than one fetch result can carry, so deliver it in **client-side** slices. This procedure moves any file size byte-exact.

> **The scaffold proxy returns the WHOLE file.** It does **not** honour any `?offset`/`?length` query — you must slice on the client with `Uint8Array`. (An earlier version of this doc claimed ranged fetches work; they do not.)

1. **Fetch the hash manifest.** Alongside `manifest.json`, fetch `manifest-hashes.json` from the same public raw URL. Shape: `{ "version": "...", "commit": "...", "files": { "<path>": "<sha256 hex>", ... } }` — one entry per `app_files` path.

2. **Deliver each file `dest`** (headers on every fetch: `x-wb-org-id`, `x-wb-project-id`, `x-wb-email`, `Cache-Control: no-cache`):
   - First, get the total size with the `javascript` tool:
     ```js
     const r = await fetch("https://workbench-install.glideapps.dev/app-api/scaffold/{path}", { headers });
     const buf = new Uint8Array(await r.arrayBuffer());
     return { total: buf.length };
     ```
   - **Small file (total ≤ 15000 bytes):** fetch once more, `return new TextDecoder().decode(buf)`, and `file_write` it to `dest`.
   - **Large file (total > 15000):** loop with `START` (from 0):
     1. Compute `END`: tentatively `Math.min(START + 15000, total)`. If `END < total`, walk `END` back to just after the **last `\n` at or before END** — every cut lands on a line boundary, so no multibyte character is ever split. **If there is no `\n` in `[START, END]`** (a single line longer than 15000 bytes), instead set `END` to the **last valid UTF-8 codepoint boundary ≤ START+15000** (never split a multibyte char).
     2. `javascript`: `const buf = new Uint8Array(await (await fetch(url,{headers})).arrayBuffer()); return { text: new TextDecoder().decode(buf.slice(START, END)), end: END };`
     3. If `START === 0`: `file_write` the returned `text` to `dest` (creates/overwrites). Else: `file_edit`-append `text` to the end of the file (anchor on the last ~40 characters currently in the file).
     4. `START = end`; repeat until `END === total`.

3. **Verify after every file.** Re-read the whole file, compute sha256, compare to `manifest-hashes.files[path]`. **`file_read` caps at ~200KB**, so for any file larger than that (`LayoutCanvas.tsx`), read it in successive line-range chunks (lines 1–2000, 2001–4000, …) until a read returns fewer lines than requested, concatenate the chunks in order, then sha256 the concatenation (the same read-in-chunks method `/cache-files` uses).
   - **Match:** next file.
   - **Mismatch (1st):** re-deliver the whole file from `START = 0`. The `file_write` overwrites, discarding any corrupt/mis-anchored tail — do **not** append onto the bad file.
   - **Mismatch (2nd, same file):** STOP. Report the path + both hashes (expected from `manifest-hashes.json`, actual) verbatim. Do not proceed past a file that failed twice.

**Notes:**
- Slice client-side, always on a line (or UTF-8 codepoint) boundary — the proxy won't range-fetch for you.
- `manifest-hashes.json` hashes are computed on the delivered (LF) bytes. If a file fails verification on every retry, it may contain CR bytes the workspace strips — the 2-strike STOP surfaces it rather than looping forever.
- **Sandbox:** shell/bash cannot write into `/project`. Only `file_write` and `file_edit` reach the workspace — never script delivery with bash.

---

## Step 5 — Connect the target app

Store the target app URL and name in the database:

```sql
UPDATE design_orgs
SET connected_app_url = '{target_app_url}',
    connected_app_name = '{target_app_name}',
    connected_app_slug = '{target_app_slug_if_known}',
    proxy_paths = '{}'::text[]
WHERE id = 'default';
```

**`proxy_paths`** will be populated automatically in Step 7d/7g once routes are extracted from the target app. Leave it empty here.

> **EXPERT MODE ONLY** — If `install_mode = "expert"`: You may optionally pre-fill `proxy_paths` now if you already know the target app's main API path prefixes. Example: `ARRAY['/app-api/orders', '/app-api/customers', '/app-api/stats']::text[]`. If you're not sure, leave it empty — Step 7d sets it correctly. Cross-origin fetches to the target app are NOT supported and architecturally unreachable through Glide's edge auth gate.

If you don't know the slug yet, leave `connected_app_slug` null. The user can set it later in Settings → Connections → Connected App.

After updating:

```sql
UPDATE design_setup_items
SET status = 'done', completed_at = now(), status_note = 'Target app URL: {target_app_url}', updated_at = now()
WHERE org_id = 'default' AND item_key = 'target_app_connected';
```

---

## Step 6 — Seed system commands

These are the built-in slash commands the agent will use. Execute this SQL:

```sql
INSERT INTO design_commands (org_id, name, aliases, description, steps, is_system) VALUES
('default', '/build',
 '{}',
 'Execute pending instructions, deploy changes, report results',
 '["SELECT all rows from design_instructions WHERE org_id=''default'' AND status=''pending'' ORDER BY created_at ASC. If none found, reply: No pending instructions.", "For each row: UPDATE status to ''in_progress''. Apply the edit described in brief to context.file. If context.previewFile differs from context.file, apply the SAME edit to both.", "For type=''platform'': execute the GlideOS tool described in brief instead.", "After each edit: UPDATE status=''done'', result=summary, executed_at=now().", "After all: call update_preview for design-workbench (DATABASE_URL db, AI ai binding).", "Check design_orgs.auto_publish. If true: call app_publish design-workbench.", "INSERT into design_app_versions (app_slug, version, label, deployed_at).", "Execute any pending design_agent_actions rows. Mark each done.", "Reply with summary table."]'::jsonb,
 true)
ON CONFLICT (org_id, name) DO NOTHING;

INSERT INTO design_commands (org_id, name, aliases, description, steps, is_system) VALUES
('default', '/publish',
 '{}',
 'Publish the design-workbench to production',
 '["Call app_publish for design-workbench.", "Reply: ✅ Published to production."]'::jsonb,
 true)
ON CONFLICT (org_id, name) DO NOTHING;

INSERT INTO design_commands (org_id, name, aliases, description, steps, is_system) VALUES
('default', '/pull-scaffold',
 '{"/update", "/sync"}',
 'Sync local workspace with the latest scaffold from the installer proxy, with installer-owned completeness + live-version verification before reporting success. NETWORK CALL REQUIRED — do not short-circuit.',
 '["Call auth_whoami to get orgId and projectId. Store as installer_org_id and installer_project_id.", "Then use the javascript tool to fetch the manifest: fetch(\"https://workbench-install.glideapps.dev/app-api/manifest?t=\" + Date.now(), { headers: { \"x-wb-org-id\": \"{installer_org_id}\", \"x-wb-project-id\": \"{installer_project_id}\", \"Cache-Control\": \"no-cache\" } }). Parse JSON. Store manifest.version and manifest.updated_at. If you cannot produce both from a LIVE fetch, stop and report which step failed.", "SELECT scaffold_version FROM design_orgs WHERE id = ''default''. Compare to manifest.version from your live fetch.", "If versions match: ping license via javascript fetch: GET https://workbench-install.glideapps.dev/app-api/license/check?version={scaffold_version} with headers x-wb-org-id, x-wb-project-id. Reply: Already on v{version} — nothing to update. (manifest.updated_at = {manifest.updated_at}) and stop.", "If newer: show changelog entries. Ask user to confirm before proceeding.", "On confirmation, BEGIN the verified update: generate a fresh actionKey = a UUID (crypto.randomUUID via javascript). POST { actionKey } to https://workbench-install.glideapps.dev/app-api/console/update/begin with headers x-wb-org-id: installer_org_id, x-wb-project-id: installer_project_id. The installer license-checks the project, records the FULL pinned expected-hash set for this version server-side, and returns { token, expectedCount }. Store token as verifyToken and actionKey. Do NOT proceed if this returns non-200 — the installer owns the expected set; there is no local fallback.", "Fetch manifest-hashes.json from https://raw.githubusercontent.com/bschonbrun/GlideOS-Design-Workbench-Public/main/manifest-hashes.json (cache-bust with ?t= a timestamp). Then for each file in manifest.app_files ONLY (do NOT fetch manifest.files — those docs are not in the pinned set and will 400), deliver it CLIENT-SIDE CHUNKED: javascript fetch the whole file as a Uint8Array from https://workbench-install.glideapps.dev/app-api/scaffold/{path} with headers x-wb-org-id, x-wb-project-id, x-wb-email, x-wb-action-key: {actionKey}; if total bytes <= 15000, file_write the decoded text once; else slice into <=15000-byte pieces each cut on the last newline at or before the boundary (if a single line exceeds 15000 bytes, cut on the last valid UTF-8 codepoint boundary so no multibyte char is split), file_write piece 0 then file_edit-append each remaining piece; then re-read the whole file and sha256 it (file_read caps at ~200KB, so for a file larger than that read it in successive line-range chunks — lines 1-2000, 2001-4000, ... — until a read returns fewer lines than requested, concatenate in order, then sha256), and compare to manifest-hashes.files[{path}]. On mismatch, re-deliver the whole file from the start (the file_write overwrites any corrupt tail — never append onto a bad file); after a SECOND mismatch on the same file, STOP and report the path plus both hashes verbatim. This delivery includes agents-core.md (the managed rulebook lives in app_files and is overwritten wholesale). Never fetch or overwrite AGENTS.md, PROJECT_NOTES.md, or PREFERENCES.md — they are per-install and not in app_files. After the app_files are delivered, also file_write the manifest.json you fetched at the start of this run to github/manifest.json so the GitHub-push panel's copy stays current (it is not pinned, so write it directly — do not fetch it from the proxy).", "Do NOT write scaffold_version yet — the version bump happens ONLY after BOTH verification gates pass (final step). Proceed to deploy + verification.", "Ping license with new version via javascript fetch: GET https://workbench-install.glideapps.dev/app-api/license/check?version={manifest.version} with headers x-wb-org-id, x-wb-project-id.", "Call update_preview for design-workbench. If auto_publish is on, app_publish design-workbench so C3 can read the LIVE production version.", "Run Part C validation. (C2 sweep) Re-read every file just written and POST { actionKey, observed: {path: sha256} } to https://workbench-install.glideapps.dev/app-api/console/sweep with header Authorization: Bearer {verifyToken}. Compute each sha256 the SAME way as the delivery verify: for a file over ~200KB read it in successive line-range chunks and concatenate in order before hashing, so per-file verify and the sweep never disagree. If any single line exceeds the ~200KB read cap so the file cannot be range-read, STOP and report — do not guess a hash. The installer compares your reported hashes to the FULL pinned manifest set and drives the action to done (complete) or needs_attention (incomplete/mismatch). Record the verdict; do NOT assume success.", "(C3 live-version gate) POST { actionKey } to https://workbench-install.glideapps.dev/app-api/console/c3/trigger with header Authorization: Bearer {verifyToken}. This tells the INSTALLER to dispatch the off-platform GitHub Action that reads the edge-stamped x-g3-app-version off the live prod 302. You cannot trigger, read, or write this verdict yourself — you only relay it. Then poll GET https://workbench-install.glideapps.dev/app-api/console/c3/status?actionKey={actionKey} with header Authorization: Bearer {verifyToken} every 10 seconds for up to 120 seconds (12 tries). Stop polling once verdict is pass/fail/error.", "Combined terminal gate: ONLY IF the sweep verdict was complete AND the c3/status verdict is pass — FIRST run UPDATE design_orgs SET scaffold_version = ''{manifest.version}'' WHERE id = ''default'', THEN reply ''✅ Updated to v{manifest.version}. (manifest.updated_at = {manifest.updated_at})''. If the sweep was incomplete/mismatch, report exactly which files were missing/bad, do NOT write scaffold_version, and do NOT claim done. If c3 is fail/error/timeout, report ''Update NOT verified — deploy-fingerprint check returned {verdict}; the live version does not match the target'', do NOT write scaffold_version, and do NOT claim done. Never report success on your own judgement — the installer owns both verdicts; you only relay them. If you cannot produce manifest.updated_at from your live fetch, do NOT fabricate it — report which step failed."]'::jsonb,
 true)
ON CONFLICT (org_id, name) DO UPDATE SET name = EXCLUDED.name, aliases = EXCLUDED.aliases, steps = EXCLUDED.steps, description = EXCLUDED.description, updated_at = now();

INSERT INTO design_commands (org_id, name, aliases, description, steps, is_system) VALUES
('default', '/repair',
 '{}',
 'Restore the Workbench agent rules — re-deliver agents-core.md and ensure AGENTS.md points to it. Use when the session-start canary reports the rules are missing.',
 '["Call auth_whoami for orgId, projectId, and email. Store them.", "Re-deliver agents-core.md. Fetch manifest-hashes.json from https://raw.githubusercontent.com/bschonbrun/GlideOS-Design-Workbench-Public/main/manifest-hashes.json (cache-bust with ?t= a timestamp) and read the expected sha256 at files[''agents-core.md'']. Then javascript-fetch the whole file as a Uint8Array from https://workbench-install.glideapps.dev/app-api/scaffold/agents-core.md with headers x-wb-org-id, x-wb-project-id, x-wb-email. If total bytes <= 15000, file_write the decoded text; else deliver it in <=15000-byte pieces each cut on the last newline at or before the boundary (file_write piece 0, then file_edit-append each remaining piece). Re-read the file, sha256 it, and compare to the expected hash. On mismatch, re-deliver once from the start; if it still mismatches, STOP and report both hashes.", "Ensure AGENTS.md carries the CURRENT pointer — this also UPGRADES a stale one. Do NOT fetch AGENTS.md from the proxy (the proxy 400s unpinned paths). file_read AGENTS.md. If its top already loads slash commands (the STEP 0 text contains ''design_commands''), it is current — leave it unchanged. Otherwise install/upgrade the pointer WITHOUT losing project rules: if AGENTS.md contains a line with the text ''PROJECT-SPECIFIC RULES BELOW THIS LINE'', REPLACE everything ABOVE that line with the canonical block below (this keeps that marker line and everything beneath it). If there is no such marker line, PREPEND the canonical block to the very TOP and keep ALL existing content below it. Canonical block (write it exactly, decoding \n as real newlines):\n\n# Design Workbench — Agent Rules\n\n## STEP 0 — before anything else, every session\n1. file_read agents-core.md and obey ALL of it (session start, turn counter, build-check, update-check, command loader).\n2. Load your slash commands even if step 1 fails: SELECT name, aliases, description, steps FROM design_commands WHERE org_id = ''default'' ORDER BY created_at ASC.\n3. Read PROJECT_NOTES.md.\nIf agents-core.md is missing, still do step 2 then tell the user to type /repair.\n\n<!-- PROJECT-SPECIFIC RULES BELOW THIS LINE — never touched by /update. Append freely. -->\n\nNEVER delete anything below the marker line.", "Reply ''✅ Workbench rules restored — agents-core.md re-delivered'' (add '', AGENTS.md pointer re-inserted'' if you had to add it). Then file_read agents-core.md and follow it for the rest of the session."]'::jsonb,
 true)
ON CONFLICT (org_id, name) DO UPDATE SET name = EXCLUDED.name, aliases = EXCLUDED.aliases, steps = EXCLUDED.steps, description = EXCLUDED.description, updated_at = now();

INSERT INTO design_commands (org_id, name, aliases, description, steps, is_system) VALUES
('default', '/credit-check',
 '{}',
 'Check GlideOS credit usage for this session.',
 '["Call billing_lookup silently.", "Calculate credits used this session: credits_used_now minus sessionStartCreditsUsed (stored in memory at session start).", "Reply: ''Credits this session: {used} used / {remaining} remaining of {quota} total.''"]'::jsonb,
 true)
ON CONFLICT (org_id, name) DO UPDATE SET name = EXCLUDED.name, aliases = EXCLUDED.aliases, steps = EXCLUDED.steps, description = EXCLUDED.description, updated_at = now();
```

> **After connecting the target app (Step 5):** if the commands above ran before the target app was set, re-run them by calling `POST /app-api/admin/refresh-commands` with header `x-glide-user-email: <your-email>`. This re-substitutes `{connected_app_slug}` and `{connected_app_name}` so `/publish` etc. target the right app.

### 6b — Seed built-in blocks

The Blocks panel requires 14 built-in blocks to be present. The easiest way is to open the Workbench as an admin — `self-populate-cache` runs automatically and seeds them. Alternatively, call:

```
POST /app-api/admin/seed-blocks
Headers: x-glide-user-email: <your-admin-email>
```

This is idempotent — safe to re-run.

---

## Step 7 — Wire up the canvas preview

**How the preview works — read this before doing anything else.**

The Workbench preview reads from the Workbench's own Neon database. When the target also uses that database, the preview reads its real data. When the target uses Supabase / external storage, the install seeds same-shape **fixture tables** in the Workbench's database with realistic sample data, and the preview reads those. **The preview is for designing UI against the target's data shape — not for viewing live data.** This is intentional: it removes credential sharing, cross-origin fetches, CORS, and Glide's edge auth wall from the equation entirely.

Concretely:

- The canvas is a same-origin iframe served at `/preview/*` by the Workbench worker.
- Preview pages call `apiFetch("/app-api/<table>")` (from `./proxy`). `apiUrl()` rewrites matching paths to `/app-api/route/*` which the Workbench worker handles **same-origin** against its own Neon DB.
- A *cross-origin* fetch to the target app from the browser is architecturally unreachable on any org-member-locked target — Glide's edge auth gate 302s the OPTIONS preflight to the login screen before the target worker ever sees the request. v2.3.0 attempted this; it does not work, there is no platform fix.

The install routine forks on the target's data source:

- **Shared-Neon target** (the target's `server.ts` only references `neon(env.DATABASE_URL)`) → live mode. `design_orgs.proxy_paths` lists the target's GET paths; `/app-api/route/*` queries the target's real tables.
- **Supabase / external target** (target's `server.ts` references `SUPABASE_*`, `mysql`, `mongodb`, or any non-Neon driver) → fixture mode. The install creates same-shape mirror tables in the Workbench's Neon, seeds them with realistic sample data, and `proxy_paths` lists the same paths.

You copy the target app's pages into `apps/design-workbench/preview/pages/`, copy the canonical `preview/App.tsx` template, and patch it to reference the target's routes.

### 7a — Resolve the target app slug

From the user's answer in Step 5, you have `target_app_name`. If you don't know the slug (the directory name under `apps/`), call `app_list` and find the app whose name matches. Store it as `target_slug`.

### 7b — Parse the target app's routes

Read `apps/{target_slug}/client/App.tsx`. Extract every `<Route path="{path}" element={<ComponentName />} />` to build a `route_list`:

```
[
  { path: "/", component: "Home", label: "Home", emoji: "🏠" },
  { path: "/list", component: "List", label: "Equipment List", emoji: "📋" },
  { path: "/detail/:id", component: "Detail", label: "Equipment Detail", emoji: "🔍" },
  ...
]
```

For each route, assign a sensible label and emoji based on the component name:
- Dashboard/Home → "Home" + 🏠
- *List → label from name + 📋
- *Detail → label from name + 🔍
- New/Create/Add* → "New {name}" + ➕
- Settings → "Settings" + ⚙️
- Otherwise → component name + 📄

### 7c — Copy page files into preview

For each `component` file referenced in routes, read from `apps/{target_slug}/client/pages/{Component}.tsx` and write to `apps/design-workbench/preview/pages/{Component}.tsx`.

After copying each page, you MUST apply **three sweep passes** in order. Skipping any one produces a blank or broken canvas with no visible error message.

**Sweep 1 — Rewrite `apiUrl` imports and raw `fetch` calls to use `apiFetch`:**

The target's pages import `apiUrl` from `"../App"` and call `fetch(apiUrl(path))` directly. In the preview, `apiUrl` lives in `./proxy` and all direct `fetch("/app-api/…")` calls must be replaced. Without this, fetches hit the Workbench worker which has no matching handler, fall through to the SPA shell HTML, and `.json()` silently throws inside `useQuery` — canvas shows empty data with no error.

```
import { apiUrl } from "../App";              →  import { apiFetch } from "../proxy";
import { apiUrl, apiWriteUrl } from "../App"; →  import { apiFetch } from "../proxy";
fetch(apiUrl(x), init)                        →  apiFetch(x, init)
fetch("/app-api/foo", init)                   →  apiFetch("/app-api/foo", init)
```

**Watch out for multi-line imports** — naive "insert after last import line" breaks when the last import wraps across lines (`import {\n  A, B,\n} from "…"`). Track brace balance to find where the import block actually ends before inserting.

**Sweep 2 — Prefix all navigation paths with `/preview`:**

Target pages use bare paths like `/isos/:id`. When the preview iframe navigates to `/isos/3`, there's no match in the Workbench router — it falls through to the catch-all which renders `<LayoutCanvas />` (the whole Workbench shell) inside itself. The symptom is: clicking any row causes the iframe to briefly render a second Workbench, then snap back to the dashboard.

Apply to EVERY `<Link to=`, `navigate(`, and `window.location` in every copied page and component:

```
<Link to={`/isos/${id}`}>          →  <Link to={`/preview/isos/${id}`}>
onClick={() => navigate("/")}      →  onClick={() => navigate("/preview/")}
navigate(`/isos/${id}`)            →  navigate(`/preview/isos/${id}`)
<Route path="/isos/:id" …>        →  <Route path="/preview/isos/:id" …>
```

Also align param names: if the target route uses `:isoId` and the detail page reads `useParams().isoId`, your preview Route must also use `:isoId` — not `:id`.

**Sweep 3 — Lint check (mandatory before deploying):**

Scan every file under `apps/design-workbench/preview/**/*.tsx` and reject any of these patterns:
- `from "../App"` — missed import rewrite (Sweep 1)
- `fetch("/app-api/` — raw fetch not wrapped in `apiFetch` (Sweep 1)
- `<Link to="/"` or `navigate("/")` where path doesn't start with `/preview` (Sweep 2)
- `<Route path="/"` where path doesn't start with `/preview` (Sweep 2)

Fix every hit before calling `update_preview`. These bugs all present identically as "empty page, no error."

**Import depth rewrite rule:** If a page imports local shared components like `import { ComponentName } from '../components/...'`, those imports stay as-is. The paths are relative to the pages directory, so `../components/` is correct from inside `preview/pages/`. 

HOWEVER, if you encounter tab components that import from `../components/tabs/` and they will be copied to `preview/components/tabs/`, the import depth changes. If a tab component at `apps/{target_slug}/client/components/tabs/MyTab.tsx` has `import { Something } from '../'`, it needs to stay as `../` when copied to `apps/design-workbench/preview/components/tabs/MyTab.tsx`. Be explicit about this depth: count the directory levels and verify the path resolves.

### 7d — Patch the canonical preview/App.tsx template

You already fetched `github/scaffold/preview-App.template.tsx` in Step 3 and wrote it to `apps/design-workbench/preview/App.tsx`. **Do not rewrite this file from scratch and do not invent your own shell.** The template is the source of truth — it has the working element interceptor, the WORKBENCH_READY/ELEMENT_HOVER/ELEMENT_SELECTED/LAYOUT_SNAPSHOT postMessage protocol the Workbench listens for, the NAVIGATE handler, the `applyTokens` CSS injection, and the `initProxyConfig` call. Any deviation breaks the canvas.

There is **NO BrowserRouter** in this file. The Workbench's own `client/App.tsx` mounts the parent router; this file's `export default function PreviewApp() { return <Shell />; }` runs inside that router. Do not add `<BrowserRouter>` here.

`apiUrl`, `apiWriteUrl`, `apiFetch`, and `initProxyConfig` live in `./proxy` (you fetched `github/scaffold/proxy.ts` in Step 3 and wrote it to `apps/design-workbench/preview/proxy.ts`). The template imports them from there; the page files copied in 7c also import from there. **Do not move any of those helpers back onto App.tsx** — it creates a circular import (App imports pages; pages would import App) and the preview goes blank with no console error.

Configure proxy paths via the database, not code. After Step 7g (which extracts the target's GET routes into `design_target_routes`), populate `design_orgs.proxy_paths` with the same path prefixes so `apiUrl()` rewrites them:

```sql
UPDATE design_orgs
SET proxy_paths = ARRAY['/app-api/orders', '/app-api/customers', '/app-api/stats']::text[]
WHERE id = 'default';
```

Patch only these three things, in place, against the file you just wrote:

**1. Page imports** (near the top, after the `initProxyConfig` import). Replace:

```tsx
import Dashboard from "./pages/Dashboard";
import EquipmentList from "./pages/EquipmentList";
import EquipmentDetail from "./pages/EquipmentDetail";
import EquipmentNew from "./pages/EquipmentNew";
```

with one `import` per page file you copied in 7c.

**2. The `ROUTES` array inside `useElementInterceptor`** (search for `const ROUTES = [` — there is exactly one). This is what gets sent to the parent in `WORKBENCH_READY` so the Workbench page-picker populates correctly. Replace with the `route_list` you built in 7b, prefixing each `path` with `/preview`:

```ts
const ROUTES = [
  { path: "/preview/",          label: "Dashboard", emoji: "🏠" },
  { path: "/preview/orders",    label: "Orders",    emoji: "📋" },
  { path: "/preview/customers", label: "Customers", emoji: "👥" },
];
```

**3. The `Nav` items and `<Routes>` block inside `Shell()`** (bottom of file). Update:

```tsx
<Nav items={[
  { href: "/preview/",          label: "Dashboard", icon: <LayoutDashboard /> },
  { href: "/preview/orders",    label: "Orders",    icon: <List /> },
  { href: "/preview/settings",  label: "Settings",  icon: <Settings /> },
]} />
<main className="max-w-5xl mx-auto px-4 sm:px-6 py-6">
  <Routes>
    <Route path="/preview"           element={<Dashboard />} />
    <Route path="/preview/"          element={<Dashboard />} />
    <Route path="/preview/orders"    element={<OrderList />} />
    <Route path="/preview/orders/:id" element={<OrderDetail />} />
    <Route path="/preview/settings"  element={<SettingsPage />} />
  </Routes>
</main>
```

Pick lucide-react icons that fit the route names. Also update the `import { ... } from "lucide-react"` line near the top to match the icons you actually use.

**That's it.** Do not touch `applyTokens`, `useElementInterceptor` (other than the `ROUTES` array), `useElementOverrides`, `detectComponent`, `stableWbId`, `onClick`/`onMouseOver`/`onMessage`, the `Toaster` import, or the default export. Those carry the canvas's working protocol — diverging from them breaks element inspection, the NAVIGATE handler, or tokens.

### 7e — Copy target-app-specific shared components

The preview shell at `apps/design-workbench/preview/components/` already contains the Glide UI primitives the Workbench preview ships with: `Nav.tsx` and `Toast.tsx` from Step 3, plus whatever else the canonical template imports.

The target app's pages probably import additional primitives (Button, Dialog, Input, Label, Select, Tabs, Form, Textarea, Checkbox, Progress, WorkflowButton, …) AND target-specific shared components (custom widgets, tab components, etc.). Walk the page files you copied in 7c and resolve every `from "../components/..."` import:

1. **Glide primitives** the target imports (Button, Dialog, Input, Label, Select, Tabs, etc.) — copy from `apps/{target_slug}/client/components/{Name}.tsx` to `apps/design-workbench/preview/components/{Name}.tsx`. The primitives already in `apps/design-workbench/client/components/` from Step 3 are for the Workbench's own UI, not the preview iframe — they live in different directories on purpose.
2. **Target-app-specific shared components** (anything that isn't a Glide primitive) — copy from `apps/{target_slug}/client/components/...` to the matching path under `apps/design-workbench/preview/components/...`, preserving subdirectory structure.

**Import depth check:** A component copied from `apps/{target_slug}/client/components/tabs/Foo.tsx` to `apps/design-workbench/preview/components/tabs/Foo.tsx` keeps the same depth — `../Button` still resolves to `preview/components/Button.tsx`. A page copied from `apps/{target_slug}/client/pages/X.tsx` to `apps/design-workbench/preview/pages/X.tsx` also keeps the same depth — `../components/Button` still resolves correctly. The depth rule only breaks if you flatten or deepen the directory tree, which you shouldn't.

If a target page imports from `../../lib/...`, copy those `lib/` files too — the workspace `lib/` directory is shared across apps so usually no copy is needed, but verify the import resolves.

### 7f — Register pages in the database

Insert one row per discoverable page (skip param-only routes like `/detail/:id`):

```sql
INSERT INTO design_pages (org_id, label, emoji, path, file, preview_file, sort_order)
VALUES
  ('default', 'Home', '🏠', '/', 'apps/{slug}/client/pages/Home.tsx', 'apps/design-workbench/preview/pages/Home.tsx', 0),
  ('default', 'Equipment List', '📋', '/list', 'apps/{slug}/client/pages/EquipmentList.tsx', 'apps/design-workbench/preview/pages/EquipmentList.tsx', 1),
  ('default', 'Equipment Detail', '🔍', '/detail/:id', 'apps/{slug}/client/pages/EquipmentDetail.tsx', 'apps/design-workbench/preview/pages/EquipmentDetail.tsx', 2)
ON CONFLICT (org_id, path) DO UPDATE
  SET label = EXCLUDED.label, emoji = EXCLUDED.emoji, file = EXCLUDED.file, preview_file = EXCLUDED.preview_file;
```

After pages are registered:

```sql
UPDATE design_setup_items
SET status = 'done', completed_at = now(), status_note = 'Preview pages copied and registered in design_pages', updated_at = now()
WHERE org_id = 'default' AND item_key = 'preview_pages_configured';
```

### 7g — Extract and register target app routes

NEW in v2.2.0: Read the target app's `server.ts` and extract every `GET /app-api/*` route to populate the data-driven route table.

Read `apps/{target_slug}/server.ts` and find patterns like:

```ts
app.get('/app-api/equipment', async c => {
  const rows = await query(c.env, 'SELECT * FROM ca_equipment');
  return c.json(rows);
});
```

For each GET route, extract:
- `path_pattern`: the exact path from `app.get()` (e.g. `/app-api/equipment`)
- `method`: `GET`
- `sql_query`: if there's a `query()` or `sql\`\`` call, extract the SQL statement and normalize it to use positional params (`$1`, `$2`, etc.)
- `param_bindings`: ordered list of where params come from. Example: if the SQL has `WHERE id = $1` and it comes from `c.req.param('id')`, the binding is `"path:id"`
- `response_shape`: what the handler returns:
  - `"rows"` if it's `c.json(array)` from a SELECT
  - `"single"` if it's `c.json(one_row)`
  - `"count"` if it returns `c.json({ count: n })`
  - `"raw"` for anything else
- `source`: `"target"`
- `source_slug`: `"{target_slug}"`

Then INSERT each into the `design_target_routes` table:

```sql
CREATE TABLE IF NOT EXISTS design_target_routes (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  method TEXT NOT NULL,
  path_pattern TEXT NOT NULL,
  sql_query TEXT,
  param_bindings TEXT[] DEFAULT '{}',
  response_shape TEXT,
  fallback_json TEXT,
  source TEXT NOT NULL,
  source_slug TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, method, path_pattern)
);

INSERT INTO design_target_routes
  (org_id, method, path_pattern, sql_query, param_bindings, response_shape, source, source_slug)
VALUES
  ('default', 'GET', '/app-api/equipment', 'SELECT * FROM ca_equipment', '{}', 'rows', 'target', '{slug}'),
  ('default', 'GET', '/app-api/equipment/:id', 'SELECT * FROM ca_equipment WHERE id = $1', '{"path:id"}', 'single', 'target', '{slug}'),
  ('default', 'GET', '/app-api/stats', 'SELECT COUNT(*), SUM(health) as total_health FROM ca_equipment', '{}', 'rows', 'target', '{slug}')
ON CONFLICT (org_id, method, path_pattern) DO UPDATE
  SET sql_query = EXCLUDED.sql_query, param_bindings = EXCLUDED.param_bindings;
```

**CRITICAL — path_pattern must NOT include `/app-api` prefix:**
The `/app-api/route/*` handler strips `/app-api/route` before matching `path_pattern`. So if your target route is `GET /app-api/isos`, the `path_pattern` to store is `/isos`, NOT `/app-api/isos`. Storing the full `/app-api/isos` means the route never matches and every canvas fetch returns `{ error: "No route matched..." }` — silently, because `useQuery` swallows non-JSON responses. Double-check every row you insert.

**CRITICAL — Neon `.query()` returns `{rows, fields}`, NOT a bare array:**
The `/app-api/route/*` handler uses `sql.query(text, params[])` to run parameterized SQL. The `@neondatabase/serverless` driver's `.query()` form returns `{ rows: [...], fields: [...] }` — **not** a bare array. The handler must unwrap via `result.rows` before indexing or measuring length. The correct pattern in server.ts is:

```ts
const result = await (sql as any).query(row.sql_query, args);
const rows: any[] = result.rows ?? result; // fallback for bare-array drivers
```

If this unwrap is missing, `shape === "single"` always returns 404 ("Not found") even when the row exists, because `rows[0]` is `undefined` (it's indexing an object, not an array). **This is the single most common reason detail pages return "Not found" after a correct install.**

**Notes:**
- This step is manual + LLM-assisted. The agent can scan the target server.ts, but extracting SQL accurately requires reading what each handler actually queries and returns. Show 2-3 worked examples from the target's server code, ask the user to verify, and insert the rows.
- The `ca_equipment` table name in the example above is CarboNet-specific. Substitute the actual table names from the target app's server.ts.
- Only GET routes need to land in `design_target_routes` for v2.2.0. Writes (POST/PUT/DELETE) still flow through the legacy `/app-api/proxy/*` handler — `apiWriteUrl()` in Step 7d routes them there. Generic writes are a v2.3.0 feature.

### 7h — Detect the target's data source

The Workbench preview always reads same-origin from the Workbench's own Neon DB. Step 7g's SQL assumes the target's tables live in that shared DB. When the target stores its data elsewhere (Supabase, MySQL, MongoDB, external REST), the preview reads a **fixture mirror** of those tables instead — created in the Workbench's Neon, seeded with realistic sample data. The preview is for designing UI; fixture data is the right trade-off.

Inspect the connected app's server file:

```
file_read("apps/{target_slug}/server.ts")
```

Decide the data source mode:

- The file references `SUPABASE_*` env vars, `mysql`, `mongodb`, or any non-Neon driver → **fixture mode** — proceed to Step 7i.
- The file only references `neon(env.DATABASE_URL)` (or no DB at all) → **live mode** — populate `design_orgs.proxy_paths` with the target's GET paths (per the SQL block at the end of Step 7d). `/app-api/route/*` queries the target's real tables. Skip to Step 8.

**Default mode:** Announce your decision concisely and proceed — do not wait for confirmation. Example: "Your target uses shared Neon — I'll wire the preview to read live data directly." Or: "Your target uses Supabase — I'll seed fixture tables in the Workbench's database so the preview has real-looking data to render."

**Expert mode:** Present the detected data source and ask for confirmation before proceeding — "I see your target uses Supabase, so I'll seed fixture data in the Workbench's Neon. Sound right? Or does your target share the Workbench's Postgres database?"

### 7i — Seed fixture tables (fixture mode only)

**Default mode:** Create and seed the fixture tables without step-by-step narration. Just announce when it's done: "Fixture tables seeded with [N] rows each."

**Expert mode:** Explain each table as you create it — what it mirrors, how many rows you're seeding, and what the data represents. Show the user the row count and a sample row for each table before moving on.

For each GET route registered in Step 7g, derive the table name and column shape from the target handler body and any field-name remapping it does. Create matching tables in the Workbench's Neon with these conventions:

- **Numeric columns: `DOUBLE PRECISION`, never `NUMERIC`.** `@neondatabase/serverless` returns `NUMERIC` as a JS string, breaking every `.toFixed()` / arithmetic call in the UI. This is non-negotiable.
- **Array columns: `TEXT[] NOT NULL DEFAULT '{}'`** so the UI's `Array.isArray(x) ? x : []` guards work without null checks.
- **FKs as `BIGINT` with explicit `REFERENCES`.**
- One `COMMENT ON TABLE` per table noting "Fixture mirror of {target_slug} {table_name}".

Seed each table with 15–25 rows of **realistic-looking** sample data. The data is for designing the UI, not for live use — make it look like a real fleet / inventory / CRM the user would recognize, not Lorem Ipsum. Names of real-sounding sites, plausible status mixes, dates spread across the last 90 days, a realistic ratio of statuses.

Then populate `design_orgs.proxy_paths` with the target's GET paths so `apiUrl()` rewrites them to the fixture handlers you'll add in Step 7j.

### 7j — Add /app-api/route/<table> handlers (fixture mode only)

For each fixture table, ensure the Step 7g `design_target_routes` row points its `sql_query` at the fixture table instead of the (non-existent in this Neon) target table. Same shape, same columns, same response_shape — just sourcing from the local fixture you seeded in 7i.

The data-driven `/app-api/route/*` handler doesn't care whether the source is a "real shared table" or a "fixture mirror" — it just runs the SQL against the Workbench's Neon. The distinction lives only in the install routine.

---

## Step 8 — Mark install complete and set versions

```sql
UPDATE design_orgs
SET scaffold_complete = true,
    scaffold_version = '2.4.2'
WHERE id = 'default';
```

---

## Step 9 — Deploy the Workbench

Call `update_preview` for the `design-workbench` app with bindings:

```json
[
  { "name": "DATABASE_URL", "type": "db" },
  { "name": "AI", "type": "ai" }
]
```

Then call `app_publish` to push it to production.

After publish succeeds, mark the workbench_deployed item done:

```sql
UPDATE design_setup_items
SET status = 'done', completed_at = now(), status_note = 'Deployed and published to production', updated_at = now()
WHERE org_id = 'default' AND item_key = 'workbench_deployed';
```

Mark `github_connected` as skipped — `/update` works via the installer proxy without a GitHub PAT:

```sql
UPDATE design_setup_items
SET status = 'skipped', status_note = '/update uses the installer proxy — no GitHub PAT required', updated_at = now()
WHERE org_id = 'default' AND item_key = 'github_connected';
```

---

## Step 10 — Verify the install

1. Open the Workbench at its production URL (check `app_list` for the exact URL)
2. The canvas preview should load the target app's Home page
3. Click any element on the preview — it should highlight and send a message to the Workbench
4. Open the Blocks panel (⊞ icon on the rail) — you should see block categories
5. Open the Setup rail (clipboard icon) — you should see the setup checklist with progress from this install
6. If the preview shows a blank screen, verify that `target_app_url` is publicly accessible and the app is published

Log the completed install:

```
POST https://workbench-install.glideapps.dev/app-api/install/log
Content-Type: application/json
x-wb-email: {user_email}

{ "projectId": "{project_id}", "orgId": "{org_id}", "email": "{user_email}", "appSlug": "design-workbench", "version": "2.4.2" }
```

---

## Step 11 — Done

```
✅ Design Workbench installed successfully ({install_mode} mode).

The Workbench is now deployed and connected to {target_app_name}. Your license has been activated automatically.

You can now:
- Click any element in the preview to inspect and style it
- Use the Blocks panel to queue build instructions
- View build history in the Edits panel
- Create workflows with the Workflow canvas (⚡ icon)
- Check the Setup panel (clipboard icon) to see what's configured and what's next

Check the Setup panel for remaining items — typically:
- AI providers configured — add at least one API key in GlideOS Secrets (Anthropic, OpenAI, Google, or DeepSeek), then open the AI tab to configure model preferences

Type `/update` at any time to get the latest version — no GitHub token required.

Type /build to execute any pending instructions.
```
