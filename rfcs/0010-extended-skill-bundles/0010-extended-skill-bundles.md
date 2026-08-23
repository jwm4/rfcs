# RFC 0010: Extended Skill Bundles

| start_date   | 2026-07-20 |
| :----------- | :--------- |
| mlflow_issue | https://github.com/mlflow/mlflow/issues/22833 |
| rfc_pr       | |

| Author(s)              | [Bill Murdock](https://github.com/jwm4) (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-07-20 |
| **AI Assistant(s)**    | Claude Code |

**Table of contents**

- [Summary](#summary)
- [Motivation](#motivation)
  - [User journeys](#user-journeys)
  - [Out of scope](#out-of-scope)
- [Open questions](#open-questions)

# Summary

[RFC-0008 (MVP Skill Registry)](https://github.com/mlflow/rfcs/pull/26) established a governed registry for AI
agent skills and skill bundles. RFC-0008 bundles already pull and
install non-skill content (subagents, hooks, MCP configurations, and
other harness-specific components) alongside skills, but only skills
receive individual registry entries. This RFC extends bundle membership
to register non-skill content, making it discoverable, trackable, and
queryable.

The design is generic rather than type-specific. RFC-0008's
`skill_bundle_version_members` table has no `member_type` column. This
RFC adds `member_type VARCHAR` with a default of `'skill'`, so existing
RFC-0008 rows are backfilled automatically with no data loss. The field
is a free-form string: `skill`, `agent`, `hook`, `mcp-server`,
`command`, `lsp-server`, `monitor`, `theme`, or any value a package
manager or harness defines. MLflow treats the type as an opaque label.
New component types do not require schema changes, new entity tables,
or new API endpoints. The plugin component landscape across agent
harnesses is large (15+ component types across Claude Code, Cursor,
Copilot, and others) and actively growing. A type-per-entity approach
would not scale.

The bundle remains the governance unit. Non-skill members are governed
through the bundle's lifecycle: versioning, status transitions,
aliases, and tags. They do not have independent lifecycles. This keeps
the registry simple while giving organizations visibility into what
their bundles contain.

# Basic example

```python
from mlflow.genai import SkillBundleMemberRef

mlflow.genai.create_skill_bundle_version(
    name="pr-workflow",
    version="2.0.0",
    members=[
        SkillBundleMemberRef(
            name="code-review", version="1.0.0", member_type="skill",
        ),
        SkillBundleMemberRef(
            name="style-check", version="2.0.0", member_type="skill",
        ),
        SkillBundleMemberRef(
            name="security-auditor", version="1.0.0", member_type="agent",
        ),
        SkillBundleMemberRef(
            name="pre-commit-scan", version="1.0.0", member_type="hook",
        ),
    ],
)
```

## Motivation

### The problem

AI agent plugins are not just skills. A Claude Code plugin can contain
agents, hooks, MCP server configurations, LSP servers, monitors,
themes, commands, and executables. A Copilot plugin can contain
instructions, prompts, canvas extensions, and hooks in different
formats. A Cursor plugin uses `.mdc` rules alongside skills.

RFC-0008's Skill Registry registers only skills within a bundle.
When a team imports a Claude Code plugin that contains 3 skills, 2
agents, a hook, and an MCP server configuration, only the 3 skills
appear in the registry. The agents, hook, and MCP config are installed
alongside the skills but are invisible to governance: they cannot be
discovered through search, tracked in bundle composition views, or
queried for impact analysis.

This creates a governance gap. An administrator asking "which bundles
include an MCP server?" or "show me all bundles with hooks" cannot
answer that question from the registry. A developer browsing bundles
sees only the skill members and must inspect the raw artifact to
understand what else is included.

There is also an observability gap. RFC-0008 introduces SKILL spans
in traces so that teams can answer "which skill version was active
during this agent run?" Non-skill members get no such treatment. When
a bundled hook fires or a bundled agent is invoked, there is no trace
span linking that execution back to the bundle and registry. Extending
trace integration to non-skill members closes this gap: the same
mechanisms from RFC-0008 (the `mlflow.skill_context()` context manager
for manual instrumentation and the install-time trace manifest for
automatic instrumentation via harness autologgers) generalize to
non-skill member types, producing spans annotated with registry
coordinates and enabling the same "which version was active?" queries
across all bundle content.

The component type landscape is large and actively growing:

- **Claude Code** defines 10+ component types including skills,
  commands, agents, hooks, MCP servers, LSP servers, monitors, themes,
  output styles, and executables.
- **APM** (Agent Package Manager) defines 7 primitive types plus
  MCP and LSP integrators, deployed across 14 harness targets.
- **Lola** supports 5 component types deployed to 7 targets.
- **Copilot** has canvas extensions and prompts that no other harness
  supports.
- **Cursor** uses `.mdc` rules that are unique to its format.

Creating a separate registry entity type (with its own DB tables,
store methods, REST endpoints, and CLI commands) for each component
type would not scale. It would also require MLflow to track and
understand the evolving component taxonomies of every harness.

### User journeys

These journeys illustrate the workflows that extended bundle membership
enables. Each builds on the infrastructure established in RFC-0008.

#### Register a bundle with mixed component types

A platform team packages a code review workflow as a Claude Code
plugin containing skills, an agent, and a hook.

1. Register the individual skills (unchanged from RFC-0008):
   ```bash
   mlflow skills-registry register --name code-review --version 1.0.0 \
       --source-type git \
       --source https://github.com/acme/plugins.git@v2.0.0 \
       --subpath skills/code-review

   mlflow skills-registry register --name style-check --version 2.0.0 \
       --source-type git \
       --source https://github.com/acme/plugins.git@v2.0.0 \
       --subpath skills/style-check
   ```

2. Create a bundle version with mixed member types:
   ```bash
   mlflow skills-registry bundles create-version \
       --name pr-workflow --version 2.0.0 \
       --source-type git \
       --source https://github.com/acme/plugins.git@v2.0.0 \
       --skill code-review:1.0.0 \
       --skill style-check:2.0.0 \
       --member agent:security-auditor:1.0.0:agents/security-auditor \
       --member hook:pre-commit-scan:1.0.0:hooks/pre-commit-scan
   ```
   Skills use the `--skill name:version` format established in
   RFC-0008. Non-skill members use `--member type:name:version[:subpath]`
   and include a `member_subpath` pointing to their location within
   the bundle artifact.

3. View the bundle composition:
   ```bash
   mlflow skills-registry bundles get-version --name pr-workflow --version 2.0.0
   ```
   Output shows all members grouped by type:
   ```
   Bundle: pr-workflow v2.0.0 (status: draft)
   Source: git https://github.com/acme/plugins.git@v2.0.0

   Members:
     Skills:
       code-review 1.0.0
       style-check 2.0.0
     Agents:
       security-auditor 1.0.0  (subpath: agents/security-auditor)
     Hooks:
       pre-commit-scan 1.0.0   (subpath: hooks/pre-commit-scan)
   ```

   **UI path:** The bundle detail page shows a tabbed or grouped view
   of members by type. Each member type has a badge (e.g., "2 skills",
   "1 agent", "1 hook"). Non-skill members link to their subpath in
   the source rather than to a registry detail page.

#### Discover non-skill content in a bundle

A developer wants to find bundles that include an MCP server
configuration, to understand which bundles require MCP infrastructure.

1. Search bundles by member type using the standard `--filter`
   expression (consistent with `search_runs`, `search_experiments`,
   and other MLflow search APIs):
   ```bash
   mlflow skills-registry bundles search-versions \
       --filter "member_type = 'mcp-server'"
   ```

2. Results show bundles containing MCP server members:
   ```
   pr-workflow      2.0.0  active   [skill(2), agent(1), hook(1)]
   data-pipeline    1.0.0  active   [skill(3), mcp-server(1)]
   full-stack-dev   3.0.0  draft    [skill(5), mcp-server(2), agent(2)]
   ```

   **UI path:** The bundle gallery view shows member type badges on
   each card. A filter panel lets users select member types to narrow
   the list. Clicking "mcp-server" shows only bundles that include
   MCP server members.

#### Cross-registry MCP server reference

A bundle includes an MCP server that is registered in the MCP Server
Registry (RFC-0004). The bundle member references the MCP registry
entry rather than embedding configuration directly.

1. Register a bundle that references an MCP server from RFC-0004:
   ```bash
   mlflow skills-registry bundles create-version \
       --name data-pipeline --version 1.0.0 \
       --skill sql-query:1.0.0 \
       --skill data-transform:1.0.0 \
       --skill visualization:1.0.0 \
       --member mcp-server:database-connector:2.0.0
   ```
   The `mcp-server` member references the name and version of an MCP
   server registered through `mlflow.genai.register_mcp_server()`
   (RFC-0004). MLflow does not embed the MCP server configuration in
   the bundle; it stores a cross-registry reference.

2. When the bundle is installed, the package manager plugin resolves
   the MCP server configuration from the MCP Registry and includes it
   in the harness-specific installation.

3. Impact analysis queries can now span both registries:
   "Which bundles reference the deprecated `database-connector` v1.0.0
   MCP server?"

   **UI path:** MCP server members in the bundle detail view link to
   the MCP server's registry detail page (RFC-0004). A deprecation
   banner on the MCP server page shows which bundles reference it.

#### Import a plugin with non-skill content

RFC-0008's `mlflow skills-registry bundles import` discovers skills
using the standard SKILL.md directory layout and warns about non-skill
content. This RFC adds an optional `--plugin-format` flag that enables
format-specific discovery rules for non-skill member types. When
omitted, import uses the RFC-0008 default (SKILL.md layout, skills
only). When provided, import uses the specified format's discovery
rules to detect and register all member types.

1. Import a Claude Code plugin:
   ```bash
   mlflow skills-registry bundles import \
       --source https://github.com/acme/plugins.git@v2.0.0 \
       --subpath pr-workflow \
       --plugin-format claude-code \
       --bundle-name pr-workflow \
       --version 2.0.0
   ```
   The `--plugin-format claude-code` flag tells import to use Claude
   Code's directory conventions to discover non-skill members alongside
   skills.

2. With this RFC, output shows all members (no more warnings for
   recognized component types):
   ```
   Discovered 4 members:
     skill: code-review (skills/code-review)
     skill: style-check (skills/style-check)
     agent: security-auditor (agents/security-auditor)
     hook:  pre-commit-scan (hooks/pre-commit-scan)

   Created bundle: pr-workflow v2.0.0 (monolithic, 4 members)
   ```
   All discovered members are registered with their detected
   `member_type`. The import command uses the plugin format's
   directory conventions to assign types (e.g., files under `agents/`
   get `member_type="agent"`).

3. If the plugin contains component types the import command does not
   recognize, it still registers them with a generic type (e.g.,
   `member_type="other"`) and includes a note in the output.

Format parsers are pluggable, using the same entrypoint mechanism as
RFC-0008's package manager plugins. This lets third parties add
discovery rules for new harness formats without changes to MLflow
itself. The plugin interface details are left to implementation.

#### Register a Cursor plugin with instructions

A team maintains a Cursor plugin that includes skills and `.mdc`
instruction rules. This journey shows the generic member type working
with a non-Claude-Code harness.

1. Import the Cursor plugin with `--plugin-format cursor` to use
   Cursor's `.mdc`-based discovery rules:
   ```bash
   mlflow skills-registry bundles import \
       --source https://github.com/acme/cursor-plugin.git@v1.0.0 \
       --plugin-format cursor \
       --bundle-name cursor-coding-standards \
       --version 1.0.0
   ```

2. Output:
   ```
   Discovered 3 members:
     skill: api-design (skills/api-design)
     skill: error-handling (skills/error-handling)
     instruction: coding-standards (rules/coding-standards.mdc)

   Created bundle: cursor-coding-standards v1.0.0 (monolithic, 3 members)
   ```
   The `instruction` member type is assigned by the Cursor format
   parser. MLflow does not need to know what a `.mdc` file is; it
   registers the member with the type the format parser provides.

3. Install for Cursor:
   ```bash
   mlflow skills-registry bundles install \
       --name cursor-coding-standards --alias production \
       --harness cursor
   ```
   The Cursor package manager plugin handles placing `.mdc` files in
   `.cursor/rules/` and skills in `.cursor/skills/`. MLflow passes
   all members to the plugin regardless of type.

#### Discover bundles with a specific component type across harnesses

An administrator wants to understand the governance surface across
all bundles, regardless of which harness they target.

1. List all distinct member types in use:
   ```bash
   mlflow skills-registry bundles search-versions --summarize member_types
   ```
   Output:
   ```
   Member types in use:
     skill         42 bundles, 156 members
     agent         12 bundles, 28 members
     hook           8 bundles, 15 members
     mcp-server     5 bundles, 7 members
     command        4 bundles, 11 members
     instruction    3 bundles, 6 members
     lsp-server     1 bundle, 1 member
   ```

2. This works without MLflow knowing anything about what these types
   mean. They are strings that package managers and import commands
   assigned. New types (e.g., `monitor`, `theme`, `canvas`) appear
   automatically as plugins using those types are imported.

#### Trace non-skill member execution

A team has deployed the `pr-workflow` bundle containing skills, an
agent, and a hook. They want to understand execution patterns across
all bundle members, not just skills.

1. An agent run triggers several bundle members: the `code-review`
   skill, the `security-auditor` agent, and the `pre-commit-scan`
   hook. Each produces a trace span annotated with registry
   coordinates (workspace, bundle name, bundle version, member name,
   member type):
   ```
   Trace: agent-run-2026-07-21T14:30:00
   ├── SKILL  code-review 1.0.0      (pr-workflow v2.0.0)
   ├── AGENT  security-auditor 1.0.0  (pr-workflow v2.0.0)
   └── HOOK   pre-commit-scan 1.0.0   (pr-workflow v2.0.0)
   ```

2. Query traces by non-skill member type:
   ```python
   traces = mlflow.search_traces(
       filter_string="span.span_type = 'HOOK'"
   )
   ```
   This returns all traces where a hook fired, with the registry
   coordinates on each span identifying which bundle and version
   the hook came from.

3. Compare execution across bundle versions. After upgrading
   `pr-workflow` from v2.0.0 to v3.0.0, the team can filter traces
   by bundle version to see whether the new `security-auditor` agent
   version changed behavior (latency, token usage, tool calls).

4. The MLflow UI Traces page extends the existing "Skills" tab to
   show all member types. Filtering by span type (SKILL, AGENT,
   HOOK, etc.) lets users isolate specific component behavior.

### Out of scope

- **Independent lifecycle for non-skill members.** Non-skill members
  do not have their own versioning, status transitions, aliases, or
  tags independent of the bundle. The bundle is the governance unit.
  Member-level tags on the membership table are not included; teams
  that need to label individual members can use bundle-level tags
  with a naming convention (e.g., `hook:pre-commit-scan:security`).
  If a future need arises for standalone agent or hook entities with
  full lifecycle, that would be a separate proposal.

- **Harness-specific format validation.** MLflow registers metadata
  about non-skill members (type, name, subpath) but does not validate
  the content format. It does not parse hook JSON schemas, agent
  frontmatter, or instruction rule syntax. Validation is the
  responsibility of the package manager plugin and the harness.

- **Security scan metadata.** As noted in RFC-0008's open questions,
  structured scan results should be addressed as a cross-registry
  capability that covers skills, MCP servers, and all other registered
  assets uniformly.

# Open questions

- Should cross-registry MCP server references be validated at bundle
  creation time (fail if the referenced MCP server version doesn't
  exist) or deferred to pull/install time?
