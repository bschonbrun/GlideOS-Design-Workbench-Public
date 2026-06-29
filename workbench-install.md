# Design Workbench — Bootstrap Install

You are a GlideOS agent. Follow these instructions exactly to install the Design Workbench into a fresh GlideOS project. Do not skip steps. Do not proceed past a step until it succeeds.

The Design Workbench is a visual design tool that runs alongside your existing GlideOS app. It gives you a canvas, an AI instruction queue, workflow automation, appearance controls, and a build history — all connected to an app you already have deployed. It does not include or replace your app; it connects to it.

---

## Prerequisites

Before starting:
1. A GlideOS project must be open (you are working inside it).
2. Store the installer's email. Read it from the Glide session header `x-glide-user-email`, or ask the user: "What email address is associated with your Glide account?" Store as `installer_email`.
3. If the user provided an invite code, store it as `invite_code`. Otherwise `invite_code` is empty.
4. Ask the user: "Which app in this project should the Workbench control? Please give me its name and production URL." Wait for their answer. Store: `target_app_name` and `target_app_url`. If they don't know the URL, call `app_list` to show them the deployed apps and their URLs.

---

## Step 1 — Fetch manifest

Fetch the manifest through the Workbench scaffold proxy:

```
GET https://p8056f420-design-workbench--bill.glideos.app/api/scaffold/manifest.json
x-wb-email: {installer_email}
x-wb-invite: {invite_code}   ← omit header if no invite code
```

Parse the JSON. Note the `version` field.

**If the response is 403:** Tell the user exactly what the response `message` field says (it will explain that a request has been sent to the owner). Stop here — do not continue until the user confirms they have been approved and asks you to try again.

**If the response is any other error:** Stop and tell the user: "❌ Could not reach the Workbench scaffold server — please try again shortly."

---

## Step 2 — Run full database schema

Execute the following SQL blocks using `db_execute`. All use `CREATE TABLE IF NOT EXISTS` so re-runs are safe. Execute each block separately.

```sql
-- Orgs
CREATE TABLE IF NOT EXISTS design_orgs (
  id TEXT PRIMARY KEY DEFAULT 'default',
  name TEXT NOT NULL DEFAULT 'My Workbench',
  slug TEXT NOT NULL DEFAULT 'my-workbench',
  custom_domain TEXT,
  domain_status TEXT DEFAULT 'none',
  domain_cname_target TEXT,
  auto_publish BOOLEAN NOT NULL DEFAULT true,
  scaffold_complete BOOLEAN DEFAULT false,
  scaffold_version TEXT,
  credit_poll_minutes INTEGER DEFAULT 5,
  connected_app_url TEXT,
  connected_app_name TEXT,
  connected_app_slug TEXT,
  trigger_mode TEXT NOT NULL DEFAULT 'on_write',
  listener_url TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  created_by TEXT
);
COMMENT ON TABLE design_orgs IS 'Single-row org config for the Design Workbench installation';

INSERT INTO design_orgs (id, name, slug) VALUES ('default', 'My Workbench', 'my-workbench')
ON CONFLICT (id) DO NOTHING;
```

```sql
-- Org members
CREATE TABLE IF NOT EXISTS design_org_members (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  email TEXT NOT NULL,
  name TEXT,
  role TEXT NOT NULL DEFAULT 'member',
  invited_by TEXT,
  joined_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, email)
);
COMMENT ON TABLE design_org_members IS 'Members of the Workbench org';
```

```sql
-- Themes
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
COMMENT ON TABLE design_themes IS 'Saved visual themes';

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
COMMENT ON TABLE design_theme_applications IS 'Log of when themes were applied';
```

```sql
-- Connections + domains
CREATE TABLE IF NOT EXISTS design_org_connections (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  label TEXT NOT NULL,
  provider TEXT,
  conn_url TEXT,
  api_url TEXT,
  api_key_hint TEXT,
  status TEXT DEFAULT 'untested',
  status_message TEXT,
  last_tested_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  created_by TEXT
);
COMMENT ON TABLE design_org_connections IS 'External data connections (databases, APIs)';

CREATE TABLE IF NOT EXISTS design_app_domains (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  app_slug TEXT NOT NULL,
  app_name TEXT,
  hostname TEXT NOT NULL,
  cname_target TEXT,
  status TEXT DEFAULT 'pending',
  bound_at TIMESTAMPTZ,
  bound_by TEXT
);
COMMENT ON TABLE design_app_domains IS 'Custom domain bindings for deployed apps';
```

```sql
-- Design tokens
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
COMMENT ON TABLE design_tokens IS 'Design token values applied to the target app';

INSERT INTO design_tokens (org_id) VALUES ('default') ON CONFLICT DO NOTHING;
```

```sql
-- Token snapshots
CREATE TABLE IF NOT EXISTS design_token_snapshots (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  label TEXT,
  tokens JSONB,
  saved_by TEXT,
  saved_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_token_snapshots IS 'Saved snapshots of design token state';
```

```sql
-- Element overrides + history + chats
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
COMMENT ON TABLE design_element_overrides IS 'Per-element style overrides keyed by wbId';

CREATE TABLE IF NOT EXISTS design_element_style_history (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  wb_id TEXT NOT NULL,
  page_path TEXT,
  component_type TEXT,
  styles JSONB,
  changed_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_element_style_history IS 'History of element style changes';

CREATE TABLE IF NOT EXISTS design_element_chats (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  wb_id TEXT,
  topic TEXT,
  messages JSONB DEFAULT '[]',
  updated_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_element_chats IS 'Mini-chat threads associated with canvas elements';
```

```sql
-- Instruction queue + build trigger
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
  used_glide_credits BOOLEAN DEFAULT false,
  input_tokens INTEGER,
  output_tokens INTEGER,
  created_at TIMESTAMPTZ DEFAULT now(),
  executed_at TIMESTAMPTZ
);
COMMENT ON TABLE design_instructions IS 'Build instruction queue — pending items are executed by the agent on /build';

CREATE TABLE IF NOT EXISTS design_build_requests (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  requested_at TIMESTAMPTZ DEFAULT now(),
  processed_at TIMESTAMPTZ,
  status TEXT NOT NULL DEFAULT 'pending',
  triggered_by TEXT DEFAULT 'user'
);
COMMENT ON TABLE design_build_requests IS 'Trigger table — inserting a row causes the agent to run /build on its next turn';
```

```sql
-- Commands + aliases
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
COMMENT ON TABLE design_commands IS 'Dynamic slash commands loaded by the agent at session start';
```

```sql
-- AI config
CREATE TABLE IF NOT EXISTS design_ai_config (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  use_glide_credits BOOLEAN DEFAULT true,
  anthropic_key_hint TEXT,
  openai_key_hint TEXT,
  google_key_hint TEXT,
  anthropic_key_enc TEXT,
  openai_key_enc TEXT,
  google_key_enc TEXT,
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
  confirm_flagship BOOLEAN DEFAULT true,
  updated_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_ai_config IS 'AI model and API key configuration for the Workbench';

INSERT INTO design_ai_config (org_id) VALUES ('default') ON CONFLICT DO NOTHING;
```

```sql
-- Scaffold templates
CREATE TABLE IF NOT EXISTS design_scaffold_templates (
  id BIGSERIAL PRIMARY KEY,
  file_name TEXT UNIQUE NOT NULL,
  version TEXT NOT NULL DEFAULT '1.0.0',
  content TEXT NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_scaffold_templates IS 'Canonical scaffold file content — AGENTS.md, PROJECT_NOTES.md, PREFERENCES.md';
```

```sql
-- History + agent actions + versions
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
COMMENT ON TABLE design_workbench_history IS 'Audit log of all Workbench changes';

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
COMMENT ON TABLE design_agent_actions IS 'Queue of GlideOS tool calls to be executed by the agent';

CREATE TABLE IF NOT EXISTS design_app_versions (
  id BIGSERIAL PRIMARY KEY,
  app_slug TEXT NOT NULL,
  version TEXT NOT NULL,
  label TEXT,
  deployed_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_app_versions IS 'Version history of deployed app builds';
```

```sql
-- Runtime errors
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
COMMENT ON TABLE design_runtime_errors IS 'Runtime errors captured post-deploy';
```

```sql
-- Session turns + preferences
CREATE TABLE IF NOT EXISTS design_session_turns (
  session_id TEXT NOT NULL,
  org_id TEXT NOT NULL DEFAULT 'default',
  turn_count INTEGER NOT NULL DEFAULT 0,
  updated_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (session_id, org_id)
);
COMMENT ON TABLE design_session_turns IS 'Per-session turn counter for auto-summarize';

CREATE TABLE IF NOT EXISTS design_preferences (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default' UNIQUE,
  communication_style TEXT DEFAULT 'balanced',
  model_effort TEXT DEFAULT 'balanced',
  explanation_verbosity TEXT DEFAULT 'normal',
  user_name TEXT,
  user_role TEXT,
  user_tech_level TEXT DEFAULT 'non-technical',
  user_domain_context TEXT,
  show_session_credits BOOLEAN DEFAULT true,
  credits_display_loc TEXT DEFAULT 'footer',
  show_api_tokens BOOLEAN DEFAULT true,
  tokens_display_loc TEXT DEFAULT 'footer',
  auto_summarize_turns INTEGER DEFAULT 40,
  summarize_mode TEXT DEFAULT 'prompt',
  current_session_id TEXT,
  agent_key TEXT,
  updated_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_preferences IS 'Agent communication preferences';

INSERT INTO design_preferences (org_id) VALUES ('default') ON CONFLICT (org_id) DO NOTHING;
```

```sql
-- File cache (for GitHub push diffs)
CREATE TABLE IF NOT EXISTS design_file_cache (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  file_path TEXT NOT NULL,
  content TEXT NOT NULL,
  content_hash TEXT,
  cached_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, file_path)
);
COMMENT ON TABLE design_file_cache IS 'Cached workspace file contents for GitHub push diffs';

CREATE TABLE IF NOT EXISTS design_push_log (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  commit_sha TEXT,
  commit_message TEXT,
  files_changed TEXT[],
  pushed_by TEXT,
  pushed_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_push_log IS 'Log of GitHub push operations';
```

```sql
-- Workflow engine tables
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
COMMENT ON TABLE design_workflows IS 'Workflow definitions';
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
COMMENT ON TABLE design_workflow_nodes IS 'Nodes within a workflow (flat tree, parent_id reconstructs graph)';

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
COMMENT ON TABLE design_workflow_runs IS 'Execution run log for workflows';

CREATE TABLE IF NOT EXISTS design_workflow_drafts (
  id BIGSERIAL PRIMARY KEY,
  workflow_id BIGINT REFERENCES design_workflows(id) ON DELETE CASCADE,
  org_id TEXT NOT NULL DEFAULT 'default',
  nl_text TEXT,
  structured JSONB,
  synced BOOLEAN DEFAULT false,
  updated_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_workflow_drafts IS 'NL drafts for bi-directional canvas sync';

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
COMMENT ON TABLE design_workflow_suggestions IS 'AI-generated workflow suggestions';

CREATE TABLE IF NOT EXISTS design_workflow_scan_log (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  trigger TEXT NOT NULL,
  ideas_found INTEGER NOT NULL DEFAULT 0,
  ideas_new INTEGER NOT NULL DEFAULT 0,
  model TEXT,
  duration_ms INTEGER,
  created_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_workflow_scan_log IS 'Log of workflow suggestion scan runs';

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
COMMENT ON TABLE design_workflow_templates IS 'Reusable workflow templates';

CREATE TABLE IF NOT EXISTS design_app_context (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  tables JSONB NOT NULL DEFAULT '[]',
  routes JSONB NOT NULL DEFAULT '[]',
  pages JSONB NOT NULL DEFAULT '[]',
  raw_context TEXT,
  scanned_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_app_context IS 'Cached app schema + routes context for workflow LLM calls';
```

```sql
-- Blocks catalog
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
COMMENT ON TABLE design_blocks IS 'DB-backed block catalog for the Blocks panel';
```

```sql
-- Pages registry
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
COMMENT ON TABLE design_pages IS 'Registry of known pages in the target app for canvas navigation';
```

```sql
-- Admin users
CREATE TABLE IF NOT EXISTS design_admins (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  email TEXT NOT NULL,
  added_by TEXT,
  added_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (org_id, email)
);
COMMENT ON TABLE design_admins IS 'Admin users with elevated Workbench access';

-- Credit snapshots (legacy)
CREATE TABLE IF NOT EXISTS design_credit_snapshots (
  id BIGSERIAL PRIMARY KEY,
  org_id TEXT NOT NULL DEFAULT 'default',
  credits_used INTEGER,
  credits_quota INTEGER,
  credits_remaining INTEGER,
  recorded_by TEXT,
  recorded_at TIMESTAMPTZ DEFAULT now()
);
COMMENT ON TABLE design_credit_snapshots IS 'Legacy credit tracking';

-- API token usage view
CREATE OR REPLACE VIEW design_api_token_usage AS
SELECT
  model_used,
  type,
  COUNT(*) as call_count,
  SUM(input_tokens) as total_input_tokens,
  SUM(output_tokens) as total_output_tokens,
  SUM(COALESCE(input_tokens, 0) + COALESCE(output_tokens, 0)) as total_tokens
FROM design_instructions
WHERE model_used IS NOT NULL
GROUP BY model_used, type
ORDER BY total_tokens DESC;
COMMENT ON VIEW design_api_token_usage IS 'Aggregated token usage by model and instruction type';
```

---

## Step 3 — Seed system commands

```sql
INSERT INTO design_commands (org_id, name, aliases, description, steps, is_system) VALUES
('default', '/build',
 '{}',
 'Execute all pending instructions from the design_instructions queue in order, deploy the changes, and report results.',
 '[
   "SELECT all rows from design_instructions WHERE org_id = ''default'' AND status = ''pending'' ORDER BY created_at ASC. If none found, reply: No pending instructions in the queue.",
   "For each pending row in order: UPDATE its status to ''in_progress''. Read context.file from the row and apply the edit described in brief. If context.previewFile exists AND differs from context.file, apply the IDENTICAL edit to context.previewFile as well.",
   "For type = ''platform'' instructions: execute the GlideOS tool call described in brief instead of a file edit.",
   "After each edit succeeds: UPDATE the row — set status = ''done'', result = a one-line summary of what changed, executed_at = now().",
   "After all rows are done: call update_preview for the design-workbench app (bindings: DATABASE_URL db, AI ai).",
   "Check auto_publish: SELECT auto_publish FROM design_orgs WHERE id = ''default''. If true: call app_publish for design-workbench. If false: tell the user Preview updated.",
   "Write a version row: INSERT INTO design_app_versions (app_slug, version, label, deployed_at) VALUES (''design-workbench'', ''{version}'', ''{summary}'', now()).",
   "Check design_agent_actions: SELECT * FROM design_agent_actions WHERE org_id = ''default'' AND status = ''pending'' ORDER BY created_at ASC. Execute any pending rows. Mark each done.",
   "Reply with a summary table: | # | Title | Type | Result |"
 ]'::jsonb,
 true)
ON CONFLICT (org_id, name) DO NOTHING;

INSERT INTO design_commands (org_id, name, aliases, description, steps, is_system) VALUES
('default', '/publish',
 '{}',
 'Publish the design-workbench to production.',
 '["Call app_publish for design-workbench.", "Reply: ✅ Published to production."]'::jsonb,
 true)
ON CONFLICT (org_id, name) DO NOTHING;

INSERT INTO design_commands (org_id, name, aliases, description, steps, is_system) VALUES
('default', '/credit-check',
 '{}',
 'Check GlideOS credit usage for this session.',
 '[
   "Call billing_lookup silently.",
   "Calculate credits used this session: credits_used_now minus sessionStartCreditsUsed (stored in memory).",
   "Reply: ''Credits this session: −{used} used / {remaining} remaining of {quota} total.''"
 ]'::jsonb,
 false)
ON CONFLICT (org_id, name) DO NOTHING;

INSERT INTO design_commands (org_id, name, aliases, description, steps, is_system) VALUES
('default', '/update',
 '{}',
 'Fetch latest scaffold files from GitHub and apply any changes.',
 '[
   "Fetch manifest.json from https://p8056f420-design-workbench--bill.glideos.app/api/scaffold/manifest.json with header x-wb-email set to the current user's Glide email (x-glide-user-email). If 403, tell the user their access has been revoked and stop.",
   "SELECT scaffold_version FROM design_orgs WHERE id = ''default''. Compare to manifest.version.",
   "If versions match: reply ''Already on v{version} — nothing to update.'' and stop.",
   "If manifest is newer: show the changelog entries since current version. Ask user to confirm.",
   "On confirmation: for each file in manifest.files and manifest.app_files, fetch via GET https://p8056f420-design-workbench--bill.glideos.app/api/scaffold/{path} with header x-wb-email set to the current user's Glide email and overwrite in workspace.",
   "UPDATE design_orgs SET scaffold_version = ''{manifest.version}'' WHERE id = ''default''.",
   "Call update_preview for design-workbench.",
   "Reply: ✅ Updated to v{manifest.version}."
 ]'::jsonb,
 true)
ON CONFLICT (org_id, name) DO NOTHING;
```

---

## Step 4 — Write scaffold files from GitHub

For each file listed in `manifest.files`, fetch through the scaffold proxy:
```
GET https://p8056f420-design-workbench--bill.glideos.app/api/scaffold/{path}
x-wb-email: {installer_email}
x-wb-invite: {invite_code}   ← omit if no invite code
```
Write the response body to `{dest}` in the workspace. If any fetch returns 403, stop and show the user the error message — approval may have lapsed.

---

## Step 5 — Write app source files from GitHub

For each file listed in `manifest.app_files`, fetch through the same scaffold proxy:
```
GET https://p8056f420-design-workbench--bill.glideos.app/api/scaffold/{path}
x-wb-email: {installer_email}
x-wb-invite: {invite_code}   ← omit if no invite code
```
Write the response body to `{dest}`. These are the Workbench app source files only.

---

## Step 6 — Connect the target app

Store the target app that the user provided in the Prerequisites:

```sql
UPDATE design_orgs
SET connected_app_url = '{target_app_url}',
    connected_app_name = '{target_app_name}',
    connected_app_slug = '{target_app_slug_if_known}'
WHERE id = 'default';
```

If you don't know the slug yet, leave `connected_app_slug` null — the user can set it later in Settings → Connections.

Then tell the user: "The Workbench canvas will show {target_app_name} as its preview. The element inspector and design tokens will apply to whatever the app renders."

---

## Step 7 — Copy target app pages into the Workbench preview

The Workbench canvas preview is a same-origin iframe served at `/preview/*` by the Workbench worker itself. It cannot directly iframe the target app (Glide's auth gate blocks cross-origin iframes). Instead, you copy the target app's pages into `apps/design-workbench/preview/pages/` and write a matching `preview/App.tsx` that wires them under `/preview/` routes.

### 7a — Determine the target app slug

From the user's answer in Prerequisites, you know `target_app_name`. If you don't already know the slug, call `app_list` and find the app whose name matches `target_app_name`. The slug is the `name` field (the directory under `apps/`). Store it as `target_slug`.

### 7b — Discover routes from the target app's App.tsx

Read `apps/{target_slug}/client/App.tsx`. Parse the `<Route>` elements to extract every `path` and its `element` component. Build a list:

```
route_list = [
  { path: "/", component: "Home", label: "Home", emoji: "🏠" },
  ... (one entry per <Route> found)
]
```

Assign a sensible `label` and `emoji` for each route based on the component name:
- If it contains "Dashboard" or "Home" → label "Dashboard", emoji "🏠"
- If it contains "List" → label "{Entity} List", emoji "📋"
- If it contains "Detail" → label "{Entity} Detail", emoji "🔍"
- If it contains "New" or "Create" or "Add" → label "New {Entity}", emoji "➕"
- If it contains "Settings" → label "Settings", emoji "⚙️"
- Otherwise → label = component name, emoji "📄"

Also note which component files are imported (the import paths) — you'll need them in 7c.

### 7c — Copy page files

For each unique page file imported in `apps/{target_slug}/client/App.tsx`:

1. Read `apps/{target_slug}/client/pages/{PageFile}.tsx`
2. Write it verbatim to `apps/design-workbench/preview/pages/{PageFile}.tsx`

Do not transform the content. The pages already use the `apiUrl()` helper (from `../App` or `./App`) — that import path will resolve correctly from inside `preview/pages/` once `preview/App.tsx` exports `apiUrl`.

If a page file imports local shared components from `../components/` (relative to the pages dir), make a note — handle those in step 7e.

### 7d — Write preview/App.tsx

Read the current `apps/design-workbench/preview/App.tsx` in full. You must preserve ALL of the following infrastructure exactly as-is — do not remove, shorten, or rewrite:

- The `apiUrl()` function and `PROXY_ROUTES` array
- The `applyTokens()` function and all its helpers (`hexToHsl`, `contrastColor`, `SPACING`, `SHADOWS`, `FONT_SIZES`, `r()`, `componentOverrides()`)
- The `DesignTokens` interface
- The `useElementInterceptor()` hook in its entirety (all handlers, the `stableWbId` helper, the `detectComponent` and `detectSentiment` helpers)
- The `useElementOverrides()` hook
- The `applyElementOverrides()` function
- The `Shell` component structure (Nav, main, Routes, Toaster)

Replace ONLY the following three things:

**1. Page imports** — replace the existing import block (Dashboard, EquipmentList, etc.) with imports for the copied pages:
```tsx
import PageA from "./pages/PageA";
import PageB from "./pages/PageB";
// ... one import per copied page file
```

**2. PROXY_ROUTES** — update to include all `/app-api/` routes the target app's server.ts exposes. Read `apps/{target_slug}/server.ts` to discover every `app.get("/app-api/...")` and `app.post("/app-api/...")` route. Build the array with those paths. Keep any routes that were already in the array if they still apply.

**3. The ROUTES constant and Nav items inside Shell** — replace with entries derived from `route_list`:

```tsx
const ROUTES = [
  { path: "/preview/",         label: "Home",   emoji: "🏠" },
  { path: "/preview/foo",      label: "Foo",    emoji: "📋" },
  // ... one entry per route from route_list (skip detail/param routes like /foo/:id)
];
```

Then replace the Nav `items` array to match, and replace the `<Routes>` block with:
```tsx
<Routes>
  <Route path="/preview"          element={<PageA />} />
  <Route path="/preview/"         element={<PageA />} />
  <Route path="/preview/foo"      element={<PageB />} />
  <Route path="/preview/foo/:id"  element={<PageC />} />
  {/* ... all routes prefixed with /preview */}
</Routes>
```

The `window.parent.postMessage({ type: "WORKBENCH_READY", ... })` call inside `useElementInterceptor` must send the updated `ROUTES` array — replace the hardcoded array there with `ROUTES` (declare `ROUTES` at module scope, above the `useElementInterceptor` function, so it's accessible).

**4. Also add a SettingsPage stub** if the target app does not have a Settings page but the Nav references one. If the target app has a real Settings page, use it.

Write the completed file to `apps/design-workbench/preview/App.tsx`, overwriting the existing one.

### 7e — Copy shared components

Read the imports in each copied page file. For any component imported from `../components/{ComponentName}` (i.e. relative imports from the app's `client/components/` directory):

1. Check if `apps/design-workbench/preview/components/{ComponentName}.tsx` already exists.
2. If not, read `apps/{target_slug}/client/components/{ComponentName}.tsx` and write it to `apps/design-workbench/preview/components/{ComponentName}.tsx`.

Skip Glide UI primitives (Button, Input, Select, etc.) — those are already installed in the preview components directory. Only copy app-specific shared components.

### 7f — Register pages in the database

Insert one row per discoverable page (exclude param-only routes like `/foo/:id`):

```sql
INSERT INTO design_pages (org_id, label, emoji, path, file, preview_file, sort_order)
VALUES
  ('default', 'Home',    '🏠', '/',    'apps/{slug}/client/pages/Home.tsx',    'apps/design-workbench/preview/pages/Home.tsx',    0),
  ('default', 'Foo',     '📋', '/foo', 'apps/{slug}/client/pages/Foo.tsx',     'apps/design-workbench/preview/pages/Foo.tsx',     1)
  -- ... one row per page
ON CONFLICT (org_id, path) DO UPDATE
  SET label = EXCLUDED.label, emoji = EXCLUDED.emoji,
      file = EXCLUDED.file, preview_file = EXCLUDED.preview_file;
```

---

## Step 8 — Mark install complete

```sql
UPDATE design_orgs
SET scaffold_complete = true,
    scaffold_version = '{manifest.version}'
WHERE id = 'default';
```

---

## Step 9 — Deploy the Workbench

Call `update_preview` for `design-workbench` with bindings:
```json
[{ "name": "DATABASE_URL", "type": "db" }, { "name": "AI", "type": "ai" }]
```

Then call `app_publish` for `design-workbench`.

---

## Step 10 — Done

Reply to the user:

```
✅ Design Workbench v{manifest.version} installed successfully.

The Workbench is now deployed and connected to {target_app_name}.

Open the Workbench to start designing. The canvas preview will load your app — click any element to inspect and style it, or use the Blocks panel to queue build instructions.

If the preview shows a blank screen, check that {target_app_url} is publicly accessible and the app is published.
```
