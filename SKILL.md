---
name: skill-manager
description: 'Search, install, bind, sync, and remove skill-manager-managed units: skills, plugins, doc-repos, and harnesses. CLI tools resolve transitively into $SKILL_MANAGER_HOME/bin/cli/ (use env.sh for absolute paths); MCP tools register with the virtual-mcp-gateway; plugins register with Claude/Codex through the local marketplace; doc-repos bind markdown sources into project CLAUDE.md / AGENTS.md via tracked copies; harnesses compose skills, plugins, doc-repos, and MCP tool selections into reusable agent/project instances. Use whenever the user asks to find, add, remove, inspect, bind, unbind, instantiate, sync, or upgrade one of those units, or to manage / deploy / invoke an MCP tool that came from one.'
---

# skill-manager

You can **discover, install, bind, and instantiate** agent capability units on demand using the `skill-manager` CLI. Treat this as your package manager for skills, plugins, doc-repos, harnesses, CLI tools, and MCP servers. Everywhere below "unit" means any of the four installable unit kinds unless a section narrows it.

## When to use this skill

- The user asks what skills/plugins/doc-repos/harnesses are available ("what skills do I have?" / "find a plugin for X" / "bind these docs" / "instantiate that harness").
- The user asks to install, remove, publish, upgrade, sync, or inspect a skill, plugin, doc-repo, or harness.
- The user asks to bind or unbind a unit into a project, especially doc-repo markdown into `CLAUDE.md` / `AGENTS.md`.
- The user asks to instantiate or tear down a harness profile for Claude/Codex/project-local docs.
- The user asks to add, describe, deploy, or invoke an MCP server / tool through the gateway.
- The user asks "what MCP tools can I use" or "what's available right now" — for units surfaced by `skill-manager list`, route through the gateway (see next section).
- You've identified a capability gap — e.g. a task needs a CLI tool or MCP server you don't yet have — and you should propose finding one.

Always narrate the plan before running commands that modify state. Install/publish/register/upgrade are side-effecting; confirm the scope before acting.

## How skill-manager-managed MCP and CLI tools are reached

When you install a unit via `skill-manager install`, declared tools are resolved transitively. For a plugin, the install pipeline walks **both** the plugin-level `skill-manager-plugin.toml` and every contained skill's `skill-manager.toml`, then unions the deps. For a harness, the resolver walks every `units = [...]` and `docs = [...]` reference before install completes:

- **CLI tools** (`[[cli_dependencies]]` in any reachable manifest) land under `$SKILL_MANAGER_HOME/bin/cli/`. Use the `env.sh` / `env.py` helper (described in **Locating CLIs by absolute path** below) to get their absolute paths so you bypass anything conflicting on the user's `PATH`.
- **MCP tools** (`[[mcp_dependencies]]`) are registered with the **virtual-mcp-gateway** — the single MCP endpoint every agent's MCP config points at. The gateway is then how you manage, deploy, and invoke them. There is no CLI for any of those operations.

To know which units (and therefore which MCP servers and CLI tools) are skill-manager-managed in the current environment, run:

```
skill-manager list
```

The MCP servers behind those units are discoverable, deployable, and callable only through the gateway's virtual tools below — not through any tool-catalog or search primitive your harness might also expose. When the user asks "what MCP tools do I have", "deploy server X", "the env var is set now, try again", or "call tool Y", route the call through the gateway's virtual tools.

For the full architectural reference — what the gateway does, how scopes work, the disclosure gate, and how to debug a server that won't deploy — see [`references/virtual-mcp-gateway.md`](references/virtual-mcp-gateway.md).

### Gateway virtual tool reference

Call all of these on the `virtual-mcp-gateway` MCP server.

**Discovery**

- `browse_mcp_servers()` — list every registered downstream server with `default_scope`, deployment state, tool counts, last error. Start here whenever the user asks what's available.
- `describe_mcp_server(server_id)` — full record for one server: `init_schema` (the env vars / secrets it needs), `default_scope`, `deployment` (`initialized_at`, `expires_at`, `init_values` with secrets redacted), `last_error`. Read this **before** deploying so you know what `initialization_params` to ask the user for.
- `browse_active_tools(server_id?)` — list tools currently exposed by deployed servers. Optional `server_id` narrows to one. Returns `tool_path`, `tool_name`, `description`.
- `search_tools(query)` — semantic search across active tool names + descriptions. Use this when the user describes a *capability* rather than naming a tool.
- `describe_tool(tool_path)` — full schema (JSON-Schema for `arguments`, server init schema). **Calling this also satisfies the gateway's per-session disclosure gate**, so you must call it at least once per session+tool before `invoke_tool` will accept the call.

**Deployment** (only available through the gateway — no CLI equivalent)

- `deploy_mcp_server(server_id, scope?, initialization_params?, reuse_last_initialization?)` — deploy or re-deploy a registered server. Pass `initialization_params` as `{ "FIELD_NAME": "value", ... }` for any required `init_schema` fields the gateway is missing (typically API keys, endpoints). `scope` defaults to the server's `default_scope`; pass `"session"` to deploy only for the current agent session, isolated from other agents. `reuse_last_initialization=true` re-uses the values from the last successful deploy when nothing has changed but the server idle-timed-out.
- `refresh_registry()` — force a registry refresh. Rarely needed; `skill-manager install` and `sync` already trigger refreshes.

**Invocation**

- `invoke_tool(tool_path, arguments)` — call a downstream tool. `tool_path` is `<server_id>/<tool_name>` (read it off `browse_active_tools` or `search_tools`). `arguments` is a JSON object matching the schema returned by `describe_tool`.

### Common flows

**"What MCP tools do I have right now?"**

1. `browse_mcp_servers()` to see registered servers and their deployment state.
2. `browse_active_tools()` (no filter) to see what's currently callable.
3. If something the user wants is registered but not deployed, follow the deployment flow.

**"I need a tool that does X" (capability search)**

1. `search_tools(query="X")` → pick the most relevant result.
2. `describe_tool(tool_path=…)` to confirm the argument shape.
3. `invoke_tool(tool_path=…, arguments=…)`.

**"Deploy / re-deploy server X" (especially after the user just set an env var or rotated a secret)**

1. `describe_mcp_server(server_id="X")` to read its `init_schema` and current `last_error`.
2. Ask the user for any required+missing values (don't fabricate API keys).
3. `deploy_mcp_server(server_id="X", initialization_params={…})`.
4. Alternative: if the user just exported the env var into their shell and re-launched their agent, run `skill-manager sync` from the CLI — it re-registers every installed unit's MCP deps and picks up env-var values for required init fields, so all eligible servers get auto-deployed in one shot.

**"Invoke tool Y"**

1. `describe_tool(tool_path="server/tool")` once per session for the disclosure gate.
2. `invoke_tool(tool_path="server/tool", arguments={…})`.

## Unit Kinds

skill-manager installs four kinds of unit:

- **Skills** — single `SKILL.md` + `skill-manager.toml`, one capability per skill. The traditional shape.
- **Plugins** — a bundle that contains one or more skills plus shared metadata. Layout: `.claude-plugin/plugin.json` + `skill-manager-plugin.toml` + `skills/<contained>/`. Use plugins when you want to ship a coherent capability set together.
- **Doc-repos** — a manifested collection of markdown sources. Layout: `skill-manager.toml` with `[doc-repo]` and `[[sources]]` rows. They install into the store, then bind into a project root with tracked copies under `docs/agents/` and managed `@docs/agents/...` imports in `CLAUDE.md` / `AGENTS.md`.
- **Harnesses** — reusable templates in `harness.toml` that compose skills, plugins, doc-repos, and MCP tool selections. They install as templates, then `skill-manager harness instantiate` materializes an instance into Claude/Codex config dirs plus a project root.

When the user describes a capability you'd like to install, default to looking for a **skill** unless they explicitly ask for a plugin, doc-repo, or harness, or the registry's matching unit is shaped that way. `skill-manager search` returns the kind in the `KIND` column. `skill-manager show <name>` renders kind-specific metadata and dependency attribution.

When referencing other units in a `skill-manager.toml` or `skill-manager-plugin.toml`, prefix with the kind to disambiguate:

- `skill:hello-skill` — bare skill named `hello-skill`
- `plugin:repo-intelligence` — plugin named `repo-intelligence`
- `doc:team-prompts` or `doc:team-prompts/review-stance` — a doc-repo or one doc source
- `harness:code-reviewer` — a harness template
- `hello-skill` (no prefix) — either kind; registry warns on ambiguity
- `github:user/repo` / `file:./path` — kind detected from the source

Contained skills inside a plugin are **not separately addressable** from the registry. `skill-manager install <contained-skill-name>` fails after the parent plugin is installed; install the parent plugin instead.

## Bindings and projections

Install puts unit bytes in the store. Binding is the projection step that makes those bytes visible to an agent or project. Each binding is recorded in `~/.skill-manager/installed/<unit>.projections.json`, so `unbind`, `sync`, `uninstall`, and harness teardown can reverse exactly what was materialized.

Common binding flows:

| Flow | Command | Result |
| --- | --- | --- |
| Bind a skill or plugin into a target root | `skill-manager bind skill:my-skill --to <root>` / `bind plugin:my-plugin --to <root>` | Symlink at `<root>/<name>`. |
| Bind one doc source | `skill-manager bind doc:team-prompts/review-stance --to <project>` | Copy to `<project>/docs/agents/<file>.md`; add imports to `CLAUDE.md` / `AGENTS.md` based on the source's `agents` list. |
| Bind all doc sources | `skill-manager bind doc:team-prompts --to <project>` | One binding per `[[sources]]` row. |
| Inspect bindings | `skill-manager bindings list` / `bindings show <bindingId>` | Shows target root, sub-element, policy, owner, and projections. |
| Move a whole-unit binding | `skill-manager rebind <bindingId> --to <newRoot>` | Reverses old projections and recreates the symlink under the new root. |
| Remove a binding | `skill-manager unbind <bindingId>` | Reverses projections and removes the ledger row. |

Doc-repo bindings default to `--policy rename` so existing `CLAUDE.md` / `AGENTS.md` files are preserved before managed imports are inserted. Whole-unit skill/plugin binds default to `--policy error`.

## Lockfile (`units.lock.toml`)

Every install / sync / upgrade / uninstall flips `~/.skill-manager/units.lock.toml` atomically at commit. The lock records `(name, kind, version, install_source, origin, ref, resolved_sha)` for every installed unit so a vendored lock can reproduce the install set byte-for-byte.

| Step | Command | When to use |
| --- | --- | --- |
| Show drift | `skill-manager lock status` | Diagnose why install state disagrees with the lock. |
| Reconcile to a vendored lock | `skill-manager sync --lock <path>` | Reproduce a known-good install set; idempotent. |
| Re-write lock from live state | `skill-manager sync --refresh` | After out-of-band edits to `~/.skill-manager/skills/` or `~/.skill-manager/plugins/`. |
| Advance lock to latest | `skill-manager upgrade <name>` / `--all` | Upgrade and bump the lock atomically. |

Suggest `sync --lock <path>` when the user wants to reproduce a vendored install set. Suggest `upgrade` when they want to advance to the registry's latest. Suggest `lock status` whenever drift is suspected (e.g. install commands behave unexpectedly after a manual edit).

## The CLI at a glance

All subcommands are run as `skill-manager <command>`. Most modifying commands take `--dry-run` (show the plan) and `--yes` (skip interactive confirmation). Policy-gated actions will refuse to proceed without a plan review.

### Discovering and installing units (skills + plugins)

| Step | Command |
| --- | --- |
| Search by keyword (returns kind in the `KIND` column) | `skill-manager search "<query>"` |
| Describe a hit | `skill-manager registry describe <name>` |
| Install by name (kind auto-detected at parse time) | `skill-manager install <name>[@<version>]` |
| Install kind-pinned | `skill-manager install skill:<name>` / `plugin:<name>` / `doc:<name>` / `harness:<name>` |
| Install from local path | `skill-manager install ./path/to/unit` |
| Install from a git repo | `skill-manager install github:user/repo` |
| Install from any git URL | `skill-manager install git+https://example.com/repo.git` |
| One-shot bootstrap of the bundled skills (skill-manager + skill-publisher; clones them from github by default) | `skill-manager onboard` |
| List installed (shows kind + sha + source columns) | `skill-manager list` |
| Show an installed unit | `skill-manager show <name>` |
| Show transitive deps | `skill-manager deps <name>` |
| Show drift between lock and live state | `skill-manager lock status` |
| Reconcile to a vendored lock | `skill-manager sync --lock <path>` |
| Re-write lock from live install set | `skill-manager sync --refresh` |
| Re-run install side effects and binding reconciliation without re-fetching | `skill-manager sync [<name>]` |
| Upgrade to the latest registry version (rolls back on failure) | `skill-manager upgrade <name>` / `--all` / `--self` |
| Uninstall (clears store + agent projections + orphan MCP servers; plugin uninstall re-walks contained skills) | `skill-manager uninstall <name>` |
| Lower-level remove (store entry only; doesn't unlink agents by default) | `skill-manager remove <name> [--from claude,codex]` |
| Scaffold a new skill | `skill-manager create <name>` |
| Scaffold a new plugin | `skill-manager create <name> --kind plugin` |
| Bind an installed unit | `skill-manager bind <coord> --to <root> [--policy error|rename|skip|overwrite]` |
| Inspect binding ledgers | `skill-manager bindings list` / `bindings show <bindingId>` |
| Move or remove a binding | `skill-manager rebind <bindingId> --to <root>` / `unbind <bindingId>` |
| Harness lifecycle | `skill-manager harness list` / `show <name>` / `instantiate <name>` / `rm <instanceId>` |

`install` always builds a plan first — fetches the unit + every transitive reference into staging, then prints what will happen (fetches, CLI installs, MCP registrations, plugin marketplace registrations). Nothing is committed to the store until consent is given.

#### Doc-repo install + bind flow

A source is installed as a **doc-repo** when its root has `skill-manager.toml` with `[doc-repo]` and one or more `[[sources]]` rows:

```toml
[doc-repo]
name = "team-prompts"
version = "0.1.0"
description = "Project prompt snippets."

[[sources]]
file = "claude-md/review-stance.md"

[[sources]]
file = "claude-md/build-instructions.md"
id = "build-instructions"
agents = ["claude"]
```

Install only stores the repo under `~/.skill-manager/docs/<name>/`. To use it in a project, bind either the whole repo or one source:

```
skill-manager install github:org/team-prompts
skill-manager bind doc:team-prompts/review-stance --to /path/to/project
skill-manager bind doc:team-prompts --to /path/to/project
```

Binding writes managed copies under `<project>/docs/agents/` and inserts import lines into `CLAUDE.md` and/or `AGENTS.md`. `skill-manager sync <doc-name>` refreshes managed copies when upstream bytes change; local edits are preserved unless `--force` is used.

#### Harness install + instantiate flow

A source is installed as a **harness** when its root has `harness.toml` with `[harness]`:

```toml
[harness]
name = "code-reviewer"
version = "0.1.0"
description = "Review harness for repo work."

units = ["skill:reviewer", "plugin:repo-tools"]
docs = ["doc:team-prompts/review-stance"]

[[mcp_tools]]
server = "repo-mcp"
tools = ["search", "open"]
```

`install` stores the template and installs every transitive skill, plugin, and doc-repo reference. `instantiate` materializes an instance:

```
skill-manager harness instantiate code-reviewer \
  --id repo-review \
  --claude-config-dir "$CLAUDE_CONFIG_DIR" \
  --codex-home "$CODEX_HOME" \
  --project-dir /path/to/project
```

Resolution order is CLI flag, then `CLAUDE_CONFIG_DIR` / `CODEX_HOME` where applicable, then a sandbox under `~/.skill-manager/harnesses/instances/<id>/`. Skills are symlinked into both Claude and Codex skill dirs. Plugins are symlinked into Claude's plugin dir only. Docs bind into the project root. The instance lock at `harnesses/instances/<id>/.harness-instance.json` preserves resolved paths so `skill-manager sync harness:<name>` can re-plan the same layout. Tear an instance down with `skill-manager harness rm <id>`.

#### Plugin install: detection + flow

A unit is installed as a **plugin** when its root contains `.claude-plugin/plugin.json` (Claude Code's runtime manifest). Plugin layout — minimum viable:

```
my-plugin/
├── .claude-plugin/plugin.json          # required — marker that this is a plugin
├── skill-manager-plugin.toml           # optional sidecar (CLI deps, MCP deps, references)
└── skills/<contained>/SKILL.md         # zero or more contained skills
```

`skill-manager-plugin.toml` is **optional**. A plugin without it still installs cleanly — the only side effect on top of bytes-on-disk is the marketplace + harness registration. When the sidecar is present, plugin-level `[[cli_dependencies]]` / `[[mcp_dependencies]]` / `references` get unioned with every contained skill's deps at parse time, so the install pipeline registers them all in one pass.

Walk-through of `skill-manager install plugin:my-plugin`:

1. Resolver fetches the source, detects `.claude-plugin/plugin.json` → kind=PLUGIN. Bytes go to `~/.skill-manager/plugins/my-plugin/`.
2. Plan-build unions the plugin-level toml's deps with every contained skill's deps. Policy gates show `! HOOKS / ! MCP / ! CLI` lines — `--yes` is blocked when a flagged category still requires confirmation in `policy.toml`.
3. CLI deps install into `~/.skill-manager/bin/cli/`; MCP servers register with the gateway.
4. The skill-manager marketplace at `~/.skill-manager/plugin-marketplace/` regenerates and the plugin gets `claude plugin install my-plugin@skill-manager --scope user` (and `codex plugin marketplace add` if codex is on PATH). Hooks/commands/agents bundled in the plugin load on the agent's next session.
5. `units.lock.toml` flips atomically with `kind = "plugin"`.

### Where units land on disk

Every installed unit gets a directory at `$SKILL_MANAGER_HOME/<skills|plugins|docs|harnesses>/<name>/` (defaults to `~/.skill-manager/`). Skills go to `skills/<name>/`; plugins to `plugins/<name>/`; doc-repos to `docs/<name>/`; harness templates to `harnesses/<name>/`. On a successful install, the CLI prints one line per newly-installed unit in a stable, parseable shape:

```
INSTALLED: hello-skill@0.1.0 -> /Users/you/.skill-manager/skills/hello-skill
INSTALLED: my-plugin@0.4.2  -> /Users/you/.skill-manager/plugins/my-plugin
INSTALLED: team-prompts@0.1.0 -> /Users/you/.skill-manager/docs/team-prompts
INSTALLED: code-reviewer@0.1.0 -> /Users/you/.skill-manager/harnesses/code-reviewer
```

Read those lines to find the unit you just acquired — no agent restart needed. A skill's directory contains `SKILL.md`, referenced assets, and `skill-manager.toml`. A plugin's directory contains `.claude-plugin/plugin.json` (required), an optional `skill-manager-plugin.toml` sidecar, and a `skills/<contained>/` tree of contained skills. A doc-repo contains `skill-manager.toml` plus markdown files declared by `[[sources]]`. A harness contains `harness.toml`.

#### How skills are exposed to agents

Skills get a per-agent symlink pointing back at the store path:

```
~/.claude/skills/<name> -> ~/.skill-manager/skills/<name>
~/.codex/skills/<name>  -> ~/.skill-manager/skills/<name>
```

#### How plugins are exposed to agents

Plugins flow through a different mechanism — the harness CLIs (`claude plugin`, `codex plugin marketplace`) only install plugins from a configured marketplace, not from arbitrary symlinks. skill-manager owns a single local marketplace that catalogs every installed plugin:

```
~/.skill-manager/plugin-marketplace/
├── .claude-plugin/marketplace.json     # auto-generated catalog ("name": "skill-manager")
└── plugins/
    └── <plugin-name> -> ~/.skill-manager/plugins/<plugin-name>
```

On every install / sync / upgrade / uninstall, skill-manager:

1. Regenerates `marketplace.json` from the current installed-plugin set.
2. If `claude` is on PATH: `claude plugin marketplace add <root>` (idempotent), `marketplace update skill-manager`, then `claude plugin uninstall <name>@skill-manager` followed by `claude plugin install <name>@skill-manager --scope user` for each plugin (the uninstall+reinstall cycle forces hooks/commands/agents to reload from the new bytes).
3. If `codex` is on PATH: `codex plugin marketplace add <root>` (also idempotent — codex re-reads the local marketplace.json each time). Final plugin install in Codex requires the user's interactive `/plugins` UI; skill-manager registers the marketplace so the user can complete it.
4. If either CLI is missing on PATH: skill-manager records `HARNESS_CLI_UNAVAILABLE` on each plugin's installed-record with a `brew install <bin>` hint. The error self-clears on the next sync once the binary is reachable — install of the plugin's bytes still completes regardless.

Use `env.sh --for claude` (or `--for codex`) to ask for the agent-visible path of a skill; default output reports the original store path.

### Locating CLIs by absolute path (avoiding PATH conflicts)

Installed CLI tools land in `$SKILL_MANAGER_HOME/bin/cli/`, but skill-manager does **not** mutate your PATH. The supported `[[cli_dependencies]]` backends are `pip`, `npm`, `brew`, `tar`, and `skill-script` (the last runs an install script bundled inside the skill itself — the escape hatch for private CLIs that can't be published to a public package manager). All five land their final binary in the same `bin/cli/` dir; the difference is purely how the bytes get there.

To invoke a skill's CLI dependency without colliding with whatever the user already has on PATH (different `npm`, different `uv`, etc.), call `env.sh` to get absolute paths:

```
<skill-manager-skill>/scripts/env.sh --skills hello-skill pip-cli-skill
# or omit --skills to dump every installed skill
<skill-manager-skill>/scripts/env.sh --pretty
```

`env.sh` is a thin wrapper that locates `uv` (skill-manager's bundled copy under `$SKILL_MANAGER_HOME/pm/uv/current/bin/uv` first, then system PATH) and runs `env.py` via `uv run --script`, so the right Python is guaranteed without requiring the user's interpreter to be 3.11+. If `uv` cannot be found in either location, `env.sh` exits with code 3 and a clear install hint.

It returns JSON with these keys you'll typically use:

- `skills` — per-skill paths, keyed by skill name. Each entry has `path` (the path you should use), `original` (always the store path under `~/.skill-manager/skills/<name>`), and `agents` (a dict of every agent symlink that exists on disk, e.g. `claude` and `codex`). Pass `--for claude` or `--for codex` to set `path` to that agent's symlink (with original-path fallback if no symlink exists). Default `--for` is unset, so `path` equals `original`.
- `package_managers` — absolute paths to bundled `uv`, `node`, `npm`, `npx` (from `~/.skill-manager/pm/<id>/current/bin/<tool>`), with system-PATH fallback. `brew` is system-only.
- `clis` — absolute path to each declared CLI dependency that is actually installed under `bin/cli/`, keyed by binary name.
- `missing` — declared CLI deps that are not on disk; each entry includes the `candidate_names` checked, so the agent can decide whether to re-install or fail loudly.

The script never mutates PATH or any shell state — just reports. Invoke the returned `path` directly (`/abs/path/to/cowsay --moo`) to bypass any conflicting tool on the user's PATH.

### Authentication

Most reads (`search`, `show`, `list`, fetching a public skill) work without logging in. Mutating operations (`publish`, creating campaigns) require a bearer token that's cached at `$SKILL_MANAGER_HOME/auth.token` after the user runs `skill-manager login`.

The CLI refreshes its access token silently from the saved refresh token — in practice the user logs in once a week (7-day refresh TTL) and never sees an auth prompt during normal work.

When the refresh token is also expired or rejected, the CLI exits with code `7` and emits a stable banner on stderr:

```
ACTION_REQUIRED: skill-manager login
Reason: <specifics>
Ask the user to run the following in their terminal, then retry the task:

    skill-manager login
```

**When you see `ACTION_REQUIRED: skill-manager login`, relay it to the user verbatim** (including the `skill-manager login` line so they can copy/paste), pause the task, and retry only after they confirm they've signed in. Never try to auth on their behalf — the browser flow needs their input.

### Working with the MCP gateway

The gateway fronts every MCP server registered by a skill-manager-managed unit; agents only ever see one MCP endpoint. The CLI owns only the gateway process lifecycle — everything else happens over MCP via the virtual tools documented in **How skill-manager-managed MCP and CLI tools are reached** above.

| Step | Command |
| --- | --- |
| Start / stop | `skill-manager gateway up` / `gateway down` |
| See URL + health | `skill-manager gateway status` |
| Re-register every installed unit's MCP deps and retry deploy with current env | `skill-manager sync` |

**How MCP servers get into the gateway**: by declaring them as `[[mcp_dependencies]]` in a skill's `skill-manager.toml`, a plugin sidecar, or a contained skill manifest, then installing that unit. Registration is a side effect of install — there is no CLI to register an MCP server directly.

After a unit's MCP deps are registered, they persist across gateway restarts. Prefer asking the user for init params (secrets, tokens) at deploy time via `deploy_mcp_server` rather than hardcoding them in the manifest.

### Publishing your own units

A skill, plugin, doc-repo, or harness is installable as soon as its bytes live anywhere `skill-manager install <source>` can reach — a github repo, an arbitrary git URL, a local path, or a tarball. The `publish` command is **only** for getting registry-supported units into a skill-manager-server registry so they show up in `skill-manager search`; it is not a prerequisite for distribution. In this CLI, registry publish/package support is still skill/plugin-shaped, so validate doc-repos and harnesses through direct `install file://...` plus bind/instantiate round-trips. For authoring guidance (skills, plugins, doc-repos, harnesses, CLI deps including `skill-script`, MCP deps, validation), route the user to **skill-publisher** — `skill-manager install github:haydenrear/skill-publisher-skill` if they don't have it.

| Step | Command |
| --- | --- |
| Inspect registry config | `skill-manager registry status` |
| Package + upload to registry for a skill/plugin (only needed for search discoverability) | `skill-manager publish [<skill-or-plugin-dir>]` |
| Package only for a skill/plugin (no upload) | `skill-manager publish <skill-or-plugin-dir> --dry-run` |

Authoring shape quick reference:

- Skill: `SKILL.md` plus `skill-manager.toml` with `[skill]`.
- Plugin: `.claude-plugin/plugin.json`, optional `skill-manager-plugin.toml`, optional contained skills under `skills/<name>/`.
- Doc-repo: `skill-manager.toml` with `[doc-repo]` plus declared `[[sources]]` markdown files.
- Harness: `harness.toml` with `[harness]`, `units`, `docs`, and optional `[[mcp_tools]]`.

### Safety and policy

Never bypass policy with `--yes` blindly. The plan output is the security surface — read it. Blocked items (`BLOCKED` / `CONFLICT`) won't run even with `--yes`; the user must explicitly loosen policy at `~/.skill-manager/policy.toml` to unblock.

If a command produces a `CONFLICT [pip] <tool>` line, two installed units want different versions of the same CLI tool. Resolve by aligning versions in one of the manifests, or removing the conflicting row from `~/.skill-manager/cli-lock.toml` if the pinned version is wrong.

## Recipes

**"Find a unit that does X and install it"**

```
skill-manager search "X"
# pick a hit, then:
skill-manager install <name> --dry-run     # review the plan
skill-manager install <name> --yes
```

**"What MCP tools can I use right now?"**

Run `skill-manager list` first to confirm which units are
skill-manager-managed in this environment, then use the gateway's
virtual tools — `browse_mcp_servers` followed by
`browse_active_tools` (or `search_tools` for capability-based search).
See **How skill-manager-managed MCP and CLI tools are reached** above
for the full flow. There is no CLI equivalent for the MCP side.

**"Add a new MCP server"**

Make a skill or plugin (even a one-off) that declares the server as an
`[[mcp_dependencies]]` entry in its manifest, then install that unit.
Registration with the gateway happens transitively. See "MCP dependencies"
in the authoring spec for the supported `load` types (docker, npm, uv,
binary, shell) and `default_scope` options.

**"Publish the unit I just edited"**

```
cd /path/to/my-unit
skill-manager publish --dry-run      # sanity check
skill-manager publish                # actual upload
```

## Model notes

- If a command fails because the gateway or registry is down, say so — don't retry silently.
- Plans show sizes + sha256 for fetched bundles; quote them when summarizing "this will download X".
- `skill-manager.toml` keys for skills are `skill_references`, `cli_dependencies`, `mcp_dependencies`, `skill`. Top-level arrays must come **before** the `[skill]` table or they get scoped under it. Doc-repos use `[doc-repo]` plus `[[sources]]`; harnesses use `harness.toml`.
