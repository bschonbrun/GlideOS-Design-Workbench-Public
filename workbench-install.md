# Design Workbench v2.5.2 — Bootstrap Install

The Design Workbench is a visual tool for building and iterating on GlideOS apps without writing code. It provides a canvas interface to inspect and style elements, an AI-powered instruction queue, workflow automation, appearance controls, and full build history. This install procedure sets it up in your project and connects it to one of your deployed apps.

You are a GlideOS agent. Follow these instructions exactly. Do not skip steps. Do not proceed past a step until it succeeds.

---

## Prerequisites

Before starting, you need:

1. **A GlideOS project** — you are currently inside it
2. **Your email address** — read from the Glide session header `x-glide-user-email`, or ask the user directly
3. **An invite code** (optional) — if the user provides one, store it. Otherwise leave blank
4. **A target app** — ask the user: "Which app in this project should the Workbench control? (Give me the app name and its production URL.)" If they don't know the URL, call `app_list` to show them all deployed apps with their URLs. Store `target_app_name` and `target_app_url`

You do **not** need a GitHub Personal Access Token to install. The Workbench installer serves all scaffold files — no direct GitHub access required. A PAT is only needed later if you want `/update` and `/git-push` commands; that is handled in Step 9.

Do **not** ask about GitHub tokens or other optional integrations yet — those are handled in Step 0 based on install mode.

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

> **EXPERT MODE ONLY** — If `install_mode = "expert"`: Ask now:
> "Do you have a GitHub Personal Access Token with `repo` scope? If yes, paste it — I'll add it as `GLIDEOS_ACCESS_TOKEN`. If not, we can skip this and add it later."
> If they provide a token, use `request_secret` with name `GLIDEOS_ACCESS_TOKEN`. Store `github_pat_provided = true`. Otherwise, `github_pat_provided = false`.

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

## Step 2b — Initialize the setup checklist

Seed the setup items that will track install progress in real-time. These start as `not_started` and get updated as each install step completes. This is what populates the Setup rail panel in the Workbench.

```sql
INSERT INTO design_setup_items (org_id, item_key, title, description, category, status, is_required, sort_order) VALUES
('default', 'project_created',           'GlideOS project created',         'The GlideOS project this Workbench runs in.',                                            'core',         'done',        true, 0),
('default', 'db_schema',                 'Database schema installed',        'All Workbench tables created and heal migrations applied.',                               'core',         'in_progress', true, 1),
('default', 'workbench_deployed',        'Design Workbench deployed',        'The Workbench worker is deployed and published to production.',                           'core',         'not_started', true, 2),
('default', 'workbench_licensed',        'Workbench licensed',               'A valid license is active for this project (required to queue builds).',                  'core',         'not_started', true, 3),
('default', 'target_app_connected',      'Target app connected',             'A target app URL is stored in design_orgs.connected_app_url.',                           'core',         'not_started', true, 4),
('default', 'preview_pages_configured', 'Preview pages configured',         'Preview pages exist under preview/pages/ and are registered in design_pages.',            'core',         'not_started', true, 5),
('default', 'github_connected',          'GitHub repository connected',      'GLIDEOS_ACCESS_TOKEN secret is set. Enables /update and /git-push commands.',            'integrations', 'not_started', false, 10),
('default', 'two_repo_github_split',     'Two-repo GitHub split',            'Public outer repo for install doc; private inner repo for source. Required for external installs.', 'integrations', 'not_started', false, 11),
('default', 'github_access_token',       'GitHub access token configured',   'GLIDEOS_ACCESS_TOKEN is set as a GlideOS project secret.',                               'integrations', 'not_started', false, 12),
('default', 'ai_models_configured',      'AI models configured',             'AI model preferences set. Default: GlideOS credits with Sonnet/Haiku split.',            'ai',           'not_started', false, 20),
('default', 'install_access_configured', 'Install access configured',        'Email gate and invite codes are set up so others can install the Workbench.',             'access',       'not_started', false, 30)
ON CONFLICT (org_id, item_key) DO NOTHING;
```

Now mark the schema step as done (it just completed):

```sql
UPDATE design_setup_items
SET status = 'done', completed_at = now(), status_note = 'Schema created by install agent', updated_at = now()
WHERE org_id = 'default' AND item_key = 'db_schema';
```

---

## Step 3 — Fetch scaffold files

Iterate over **every** entry in `manifest.files` (do not assume the list — read it from the manifest you fetched in Step 1). For each entry, fetch it through the Workbench installer and write the response body to its `dest`:

```
GET https://workbench-install.glideapps.dev/app-api/scaffold/{entry.path}
x-wb-email: {user_email}
x-wb-invite: {invite_code}    ← omit if empty
```

The current v2.4.2 manifest's `files` array includes:
- `AGENTS.md`, `PROJECT_NOTES.md`, `PREFERENCES.md` — agent docs at workspace root
- `github/manifest.json`, `github/workbench-install.md` — the manifest + this doc, mirrored into the workspace
- `github/scaffold/auth.d.ts` → `apps/design-workbench/auth.d.ts` — auth type declarations
- `github/scaffold/{Button,Dialog,Input,Label,Select,Tabs}.tsx` → `apps/design-workbench/client/components/*.tsx` — Glide UI primitives for the Workbench UI. `Dialog.tsx` exports `DialogHeader`, `DialogTitle`, `DialogFooter` (LayoutCanvas + SetupModal import these — verify on disk after fetch)
- `github/scaffold/Nav.tsx`, `github/scaffold/Toast.tsx` → `apps/design-workbench/preview/components/*.tsx` — primitives used by the preview iframe shell. `Toast.tsx` exports `Toaster`
- `github/scaffold/proxy.ts` → `apps/design-workbench/preview/proxy.ts` — preview proxy module (`apiUrl`, `apiFetch`, `initProxyConfig`). Lives in its own module so preview pages can import it without creating a circular import with App.tsx
- `github/scaffold/preview-App.template.tsx` → `apps/design-workbench/preview/App.tsx` — canonical preview shell (Step 7d patches target-specific bits only)

If any fetch returns 403, stop — approval may have lapsed.

---

## Step 4 — Fetch app source files

For each file listed in `manifest.app_files`, fetch it the same way:

```
GET https://workbench-install.glideapps.dev/app-api/scaffold/{path}
x-wb-email: {user_email}
x-wb-invite: {invite_code}    ← omit if empty
```

Write the response body to `{dest}`.

**Special case — source bundles:** Files named `source-bundle*.ts` will return 404 — do not fetch them. Instead, create three empty files:

```ts
// apps/design-workbench/source-bundle.ts
export const BUNDLED_SOURCE_FILES = {};
```

```ts
// apps/design-workbench/source-bundle-2.ts
export const BUNDLED_SOURCE_FILES_2 = {};
```

```ts
// apps/design-workbench/source-bundle-3.ts
export const BUNDLED_SOURCE_FILES_3 = {};
```

The `/build` command will regenerate them when the Workbench is deployed.

After all scaffold files are written, mark the workbench_deployed item as in-progress:

```sql
UPDATE design_setup_items
SET status = 'in_progress', status_note = 'Scaffold files written — deploy pending', updated_at = now()
WHERE org_id = 'default' AND item_key = 'workbench_deployed';
```

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
('default', '/update',
 '{}',
 'Fetch latest scaffold files from GitHub',
 '["Fetch manifest.json from https://workbench-install.glideapps.dev/app-api/scaffold/manifest.json with x-wb-email header. If 403, tell user access revoked and stop.", "Compare manifest.version to design_orgs.scaffold_version. If same, reply Already up to date and stop.", "If newer: show changelog and ask confirmation.", "On confirmation: for each file in manifest.files and manifest.app_files, fetch and overwrite in workspace.", "UPDATE design_orgs SET scaffold_version = ''{manifest.version}''.", "Call update_preview for design-workbench.", "Reply: ✅ Updated to v{version}."]'::jsonb,
 true)
ON CONFLICT (org_id, name) DO NOTHING;
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

After copying each page, **rewrite its `apiUrl` imports**: the target's pages import `apiUrl` from `"../App"`, but in the Workbench preview that helper lives in `./proxy` (not `App.tsx`) to break the App↔pages circular import. Run this rewrite on every copied page:

```
import { apiUrl } from "../App";      →  import { apiFetch } from "../proxy";
import { apiUrl, apiWriteUrl } from "../App";  →  import { apiFetch } from "../proxy";
fetch(apiUrl(x), init)                →  apiFetch(x, init)
```

`apiFetch` is the convenience helper from `proxy.ts` — `fetch(apiUrl(path), init)` in one call. Pages that exclusively use `apiUrl` directly should be rewritten to `apiFetch`; the helper exists specifically so the rewrite contract lives in one place.

**Lint check after rewrites:** scan `apps/design-workbench/preview/**/*.tsx` for any remaining `from "../App"` import or any `<Link to="/...">` / `navigate("/...")` whose path does NOT start with `/preview/`. Bare-`/` paths bubble up to the GlideOS shell which interprets them as "open a new app tab" — the visible symptom is the iframe flipping back to dashboard while a duplicate Workbench tab opens. Concrete pattern rewrites:

```
<Link to={`/iso/${id}`}>           →  <Link to={`/preview/iso/${id}`}>
onClick={() => navigate("/")}       →  onClick={() => navigate("/preview/")}
navigate(`/iso/${id}`)              →  navigate(`/preview/iso/${id}`)
```

Apply to every Link / navigate / Route-relative path in the copied preview pages, components, and Nav brand.

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

Check if `GLIDEOS_ACCESS_TOKEN` was provided:

- **Expert mode:** `github_pat_provided` was set in Step 0. If `true`, mark the GitHub items done below.
- **Default mode:** Prompt briefly now — "Want to enable GitHub sync? Drop your GitHub PAT (classic, `repo` scope) here, or type `skip` to set it up later." If they provide one, use `request_secret` with name `GLIDEOS_ACCESS_TOKEN`; set `github_pat_provided = true`. If they skip, set `github_pat_provided = false`.

```sql
-- Only run if github_pat_provided = true
UPDATE design_setup_items
SET status = 'done', completed_at = now(), status_note = 'GLIDEOS_ACCESS_TOKEN secret configured', updated_at = now()
WHERE org_id = 'default' AND item_key IN ('github_access_token', 'github_connected');
```

If `github_pat_provided = false`, mark those items as `pending` instead:

```sql
-- Only run if github_pat_provided = false
UPDATE design_setup_items
SET status = 'pending', status_note = 'Add GLIDEOS_ACCESS_TOKEN secret to enable /update and /git-push', updated_at = now()
WHERE org_id = 'default' AND item_key IN ('github_access_token', 'github_connected');
```

And mark the license item as pending (user needs to request a license after install):

```sql
UPDATE design_setup_items
SET status = 'pending', status_note = 'License required to enable builds — request at workbench-install.glideapps.dev', updated_at = now()
WHERE org_id = 'default' AND item_key = 'workbench_licensed';
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

Reply to the user — tailor the pending items based on `github_pat_provided`:

**If `github_pat_provided = true`:**

```
✅ Design Workbench v2.4.2 installed successfully ({install_mode} mode).

The Workbench is now deployed and connected to {target_app_name}.

You can now:
- Click any element in the preview to inspect and style it
- Use the Blocks panel to queue build instructions
- View build history in the Edits panel
- Create workflows with the Workflow canvas (⚡ icon)
- Check the Setup panel (clipboard icon) to see what's configured and what's next

The Setup panel shows 2 pending items:
- Workbench licensed — request a license at workbench-install.glideapps.dev
- AI models configured — default is GlideOS credits with Sonnet/Haiku (no action needed unless you want to change it)

Type /build to execute pending instructions. Type /update to fetch the latest Workbench version.
```

**If `github_pat_provided = false`:**

```
✅ Design Workbench v2.4.2 installed successfully ({install_mode} mode).

The Workbench is now deployed and connected to {target_app_name}.

You can now:
- Click any element in the preview to inspect and style it
- Use the Blocks panel to queue build instructions
- View build history in the Edits panel
- Create workflows with the Workflow canvas (⚡ icon)
- Check the Setup panel (clipboard icon) to see what's configured and what's next

The Setup panel shows 3 pending items:
- Workbench licensed — request a license at workbench-install.glideapps.dev
- GitHub access token — add a GLIDEOS_ACCESS_TOKEN secret to enable /update and /git-push
- AI models configured — default is GlideOS credits with Sonnet/Haiku (no action needed unless you want to change it)

Type /build to execute pending instructions. Type /update to fetch the latest Workbench version (requires GitHub token).
```
