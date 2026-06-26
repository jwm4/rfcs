---
start_date: 2026-04-27
mlflow_issue: https://github.com/mlflow/mlflow/issues/22833
rfc_pr: https://github.com/mlflow/rfcs/pull/10
---

# RFC: Skill Registry Harness Integration

| Author(s)              | Bill Murdock (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-06-12 |
| **AI Assistant(s)**    | Claude Code (Opus 4.6) |

# Summary

Add harness-specific installation to the MLflow Skill Registry
(RFC-0008). Where RFC-0008 provides `mlflow skills pull` to fetch
registered content to a local directory, this RFC adds
`mlflow skills install` and `mlflow skills install-bundle` to generate
harness-specific manifests, place files in the correct directories,
and configure the agent harness to use the installed capabilities.

This bridges the gap between "I found a skill bundle in the registry"
and "my agent harness can use it."

# Basic example

## Install a skill bundle for Claude Code

```bash
mlflow skills install-bundle --name pr-workflow --alias production \
    --harness claude-code
```

This resolves the `pr-workflow` skill bundle, pulls the bundle content
according to its registered mode, and generates:

```
.claude/plugins/pr-workflow/
  .claude-plugin/plugin.json      # Generated manifest
  skills/
    code-review/SKILL.md          # Installed skill content
  agents/
    security-auditor.md           # Installed subagent content
  .mcp.json                       # Generated from mcp-server members
```

## Install for other harnesses

```bash
# Same command, different --harness: codex-cli, cursor, antigravity
mlflow skills install-bundle --name pr-workflow --alias production \
    --harness cursor

# Install globally (user scope) instead of project scope
mlflow skills install-bundle --name pr-workflow --alias production \
    --harness claude-code --scope user
```

## Import an existing plugin as a skill bundle

```bash
# Register an existing Claude Code plugin as a monolithic skill bundle
mlflow skills import --source https://github.com/acme/plugins.git@v1.0.0 \
    --subpath pr-workflow \
    --harness claude-code --bundle-name my-plugin --version 1.0.0
```

This fetches the artifact, inspects it to identify skills, subagents,
hooks, and MCP servers, and registers a monolithic skill bundle version
with discovered skill, subagent, and hook members. The source must be
remotely accessible (git, OCI, ZIP, or MLflow artifact URI) so that the
registered bundle has a pullable source pointer.

## Python SDK

```python
import mlflow

mlflow.genai.skills.install_bundle(
    name="pr-workflow",
    alias="production",
    harness="claude-code",
    scope="project",  # or "user" for global install
)

# Import an existing harness-specific plugin into the registry
mlflow.genai.skills.import_bundle(
    source="https://github.com/acme/plugins.git@v1.0.0",
    subpath="pr-workflow",
    harness="claude-code",
    bundle_name="my-plugin",
    version="1.0.0",
)
```

## Motivation

### The problem

RFC-0008 provides `pull` for fetching content to a local directory,
but each harness has its own directory layout, manifest format, and
discovery mechanism (see table below). Without harness-specific
installation, users must manually create manifests, place files in
the right directories, and configure discovery. This is error-prone
and discourages adoption.

### The cross-harness landscape

The following table summarizes the capability types and installation
conventions across major agent harnesses:

| Harness | Skills | Agents | MCP | Hooks | Manifest | Install dir |
|---|---|---|---|---|---|---|
| Claude Code | SKILL.md | agent .md | .mcp.json | settings.json | plugin.json | `.claude/plugins/` |
| Codex CLI | SKILL.md | agent .md | .mcp.json | hooks | plugin.json | `.codex/plugins/` |
| Cursor | SKILL.md | agent .md | mcp.json | -- | -- | `.cursor/skills/`, `.cursor/agents/` |
| GitHub Copilot | skills/ | agents/ | .mcp.json | hooks/*.json | plugin.json | project |
| Lola | SKILL.md | agents/*.md | mcps.json | lola.yaml | (auto-discovered) | per-harness |
| OpenClaw | SKILL.md | -- | -- | plugin hooks | openclaw.plugin.json | `skills/` |
| Kilo Code | SKILL.md | custom modes | mcp.json | -- | -- | project |
| Antigravity | SKILL.md | -- | -- | -- | -- | `.agent/skills/` |
| OpenCode | .md/.ts | agent configs | config | JS events | -- | `.opencode/` |
| Continue | -- | config.yaml | mcpServers/ | -- | -- | `.continue/` |
| Windsurf | -- | -- | mcp_config.json | -- | -- | project |
| Amazon Q | -- | -- | mcp.json | -- | -- | `.amazonq/` |
| Goose | -- | -- | MCP only | -- | -- | config |
| Zed | -- | profiles | settings.json | -- | -- | config |

Key insight: the SKILL.md file format is portable across harnesses.
Only the directory placement and manifest format differ.

### Out of scope

- Registry operations (covered in RFC-0008).
- Extending harness functionality (e.g., adding hook support).
- Automatic harness detection (follow-up).

## Detailed design

### Harness adapters

Each supported harness has an adapter that knows how to:

1. **Map member types to harness paths.** Given the bundle's member
   types (skill, subagent, hook, mcp_server) and the install scope,
   determine where each member's content should be placed.
2. **Declare install paths per scope.** Each adapter knows both the
   project-level path (e.g., `.claude/plugins/`) and the user-level
   global path (e.g., `~/.claude/plugins/`). The `scope` parameter
   selects which one to use.
3. **Generate manifests.** Create harness-specific manifest files
   (e.g., `plugin.json`, `.mcp.json`) from registry metadata.
4. **Handle unsupported types.** Skip member types the harness does
   not support, with a warning by default. If `strict` mode is enabled,
   fail the install instead of producing a partial harness artifact.
5. **Introspect existing bundles.** Given a harness-specific artifact
   (e.g., a Claude Code plugin directory), identify the individual
   capabilities it contains and their types.

The adapter does not download content from sources. The MLflow client
handles all source fetching (Git clone, OCI pull, ZIP download, etc.)
via the same pull logic as `mlflow skills pull` (RFC-0008), then
passes pre-fetched local paths to the adapter. This keeps adapters
simple: they only need to know about directory layout and manifest
generation, not about source types.

```python
from abc import abstractmethod
from typing import Literal


@dataclass
class PulledMember:
    name: str
    kind: str  # "skill", "subagent", "hook", "mcp_server"
    local_path: str  # pre-fetched content on local filesystem
    version: str
    metadata: dict[str, str] | None = None


@dataclass
class PulledBundle:
    name: str
    version: str
    mode: Literal["assembled", "monolithic"]
    # Assembled: pulled member content. Monolithic: registered embedded members.
    members: list[PulledMember] = field(default_factory=list)
    mcp_servers: list[tuple[str, dict]] = field(default_factory=list)
    bundle_path: str | None = None  # pulled monolithic bundle artifact
    metadata: dict[str, str] | None = None


@dataclass
class IntrospectedMember:
    name: str
    kind: str  # "skill", "subagent", "hook", "mcp_server"
    source_path: str
    description: str | None = None
    metadata: dict[str, str] | None = None


@dataclass
class IntrospectedBundle:
    name: str
    description: str | None = None
    members: list[IntrospectedMember] = field(default_factory=list)


class HarnessAdapter:
    @abstractmethod
    def install_skill(
        self,
        member: PulledMember,
        scope: str = "project",  # "project" or "user"
    ) -> str: ...

    @abstractmethod
    def install_skill_bundle(
        self,
        bundle: PulledBundle,
        scope: str = "project",  # "project" or "user"
    ) -> str: ...

    @abstractmethod
    def introspect_bundle(
        self, source: str,
    ) -> IntrospectedBundle: ...

    @abstractmethod
    def supported_member_types(self) -> set[str]: ...
```

### Adapter summaries

Each builtin adapter maps member types to harness-specific paths,
generates manifests, and skips unsupported types with warnings. See
[implementation-details.md: Adapter
summaries](implementation-details.md#adapter-summaries) for
per-adapter behavior (Claude Code / Codex CLI, Cursor, Antigravity,
and harness-agnostic bundle formats).

Detailed directory layouts, MCP config generation rules, and
hook handling behavior are in
[implementation-details.md](implementation-details.md).

### Other harness adapters

Additional adapters (OpenClaw, GitHub Copilot, Kilo Code, OpenCode,
Continue, etc.) follow the same pattern: map member types to paths,
generate manifests, skip unsupported types with warnings.

New adapters can be contributed without changes to the registry or
the adapter interface. Adapters are registered via Python entrypoints
(group `mlflow.skill_harness_adapters`), so third-party adapters can
be installed via `pip install` without modifying MLflow core. MLflow
ships builtin adapters for Claude Code, Codex CLI, and Cursor;
additional harnesses are community-contributed.

### Bundle import

Installation takes registry metadata and produces a harness-specific
artifact. Bundle import is the reverse: it takes an existing artifact
in any supported format (e.g., a Claude Code plugin or a cross-harness
module), introspects it to discover individual capabilities, and
registers the artifact as a monolithic skill bundle version with member
references.

#### Contract

The import operation takes four inputs:

- **source**: a reference to the artifact: git URL, OCI reference,
  ZIP URL, or MLflow artifact URI. The import operation fetches the
  artifact from the source before introspection. The source must be
  a remotely accessible location so that the registered bundle version
  has a source pointer other users can pull from. To import from a local
  directory, first upload it to MLflow artifact storage
  (`source_type="mlflow"`) or push it to a Git/OCI/ZIP source, then
  import using that remote reference.
- **harness**: the harness format to interpret the artifact as (e.g.,
  `claude-code`, `cursor`). Required in the initial release.
  Automatic detection is a follow-up feature.
- **bundle_name**: the name for the resulting skill bundle. If
  omitted, the adapter derives a name from the artifact (e.g., from
  `plugin.json` or the directory name).
- **version**: the semantic version for the resulting skill bundle
  version. If omitted, the adapter may derive it from artifact metadata
  when available. Import fails if neither the caller nor the artifact
  provides a valid semantic version.

The import operation:

1. Calls the adapter's `introspect_bundle` method, which parses the
   artifact and returns an `IntrospectedBundle` listing each
   discovered member with its type, source path, and any metadata the
   adapter can extract.
2. Creates or updates the parent skill bundle.
3. Creates a monolithic skill bundle version with the import source as
   the bundle-level source pointer.
4. Registers discovered skills, subagents, and hooks as member versions
   in the skill registry. These member versions may omit `source`
   because their content lives inside the bundle-level artifact.
5. Adds the discovered member references to the bundle version and
   records each member's `source_path` as the membership `member_subpath` inside
   the bundle artifact. Returns the created bundle version plus an
   introspection summary.

Embedded MCP configs remain in the source artifact unless the import
can match them to existing MCP Registry entries. If matching MCP server
versions exist, the bundle may reference them through `mcp_servers`;
otherwise the embedded config remains bundle artifact content.

#### Conflict handling

When a bundle version with the same `(bundle_name, version)` already
exists, import reports the conflict and does not overwrite it. The
caller can resolve the conflict by choosing a different bundle name or
version.

Import also never overwrites existing skill, subagent, or hook versions.
If a discovered member's `(type, name, version)` already exists, import
may reuse that version only when it is compatible with the discovered
member. For example, it can reuse a source-less embedded member version
that already belongs to the same bundle artifact. If the existing
version points to different content, has an incompatible source model,
or cannot be proven compatible, import reports a conflict and fails
rather than binding the new bundle to the wrong governed member. The
caller can resolve the conflict by changing the imported bundle version,
renaming the member, or registering the embedded member under a new
version.

#### SDK

```python
# Preview what an artifact contains (read-only, no registry writes)
# Introspect works on local paths or remote sources
preview = mlflow.genai.skills.introspect_bundle(
    source="./my-claude-plugin",
    harness="claude-code",
)
# preview.members lists discovered skills, subagents, hooks, MCP configs

# Import the artifact as a monolithic bundle (source must be remotely accessible)
mlflow.genai.skills.import_bundle(
    source="https://github.com/acme/plugins.git@v1.0.0",
    subpath="pr-workflow",
    harness="claude-code",
    bundle_name="my-plugin",
    version="1.0.0",
)
```

#### CLI

```bash
# Preview what an artifact contains (read-only, works on local paths)
mlflow skills introspect --source ./my-claude-plugin \
    --harness claude-code

# Import from git, OCI, ZIP, or MLflow artifact sources
mlflow skills import --source https://github.com/acme/plugins.git@v1.0.0 \
    --subpath pr-workflow \
    --harness claude-code --bundle-name my-plugin --version 1.0.0
```

Import is a CLI and SDK operation only. There is no UI for import.
Import requires fetching artifacts from user-supplied URLs, which the
server should not do on behalf of clients.

### Future marketplace integration

Some harnesses (Claude Code, Codex CLI) support marketplace catalogs:
a JSON endpoint that lists available plugins so users can browse and
install them natively from within the harness. Marketplace catalog
generation is useful, but it is follow-up work outside the initial
release of this RFC. The initial installation path is the adapter-based
CLI/SDK flow (`mlflow skills install` / `install-bundle`).

A future marketplace integration could expose published skill bundles
through a harness-specific catalog endpoint such as:

```
GET /ajax-api/3.0/mlflow/skill-bundles/marketplace.json?harness=claude-code
```

That endpoint would need to define the harness-specific response schema,
authentication behavior, packaging or redirect strategy, and how entries
map to monolithic versus assembled bundle versions. Those details are
deferred to the follow-up marketplace work in the adoption strategy.

Until marketplace integration exists, the MLflow Skills page
(RFC-0008) serves as the browsing interface. Users search and filter
registered bundles in the MLflow UI, then copy the install command from
the bundle detail page:

```
mlflow skills install-bundle --name pr-workflow --alias production \
    --harness cursor
```

The bundle detail page in the MLflow UI displays a ready-to-copy
install command for each supported harness, reducing the manual steps
required.

### Implementation details

SDK function signatures (`install_skill`, `install_bundle`,
`import_bundle`) and CLI commands are in
[implementation-details.md](implementation-details.md).

### Lock file

A project can check in an `mlflow-skills.lock` file that records the
harness, install scope, exact resolved versions, source URIs, and
content digests so that `mlflow skills install` with no arguments
reproduces the same local setup (analogous to `package-lock.json` or
`poetry.lock`).

```bash
# First install: resolves from registry and writes lock file
mlflow skills install-bundle --name pr-workflow --alias production \
    --harness claude-code --lock

# Subsequent installs: reads lock file, no alias or version resolution needed
mlflow skills install

# Update: re-resolves the explicit selector and updates lock file
mlflow skills install-bundle --name pr-workflow --alias production \
    --harness claude-code --lock --update
```

The lock file records resolved versions, not aliases, version ranges,
or other selectors. This ensures reproducible installs and avoids
stale, non-authoritative selector metadata. The `--update` flag uses
the explicit selector supplied to that command, such as `--alias
production`, to resolve a new target and write the new resolved version
to the lock file.

Lock file replay still contacts the registry to verify version
status, so governance actions (deprecation, deletion) take effect
even for existing lock files. The lock file can optionally record the
registry URI and workspace used at lock time. If present, replay uses
them; if omitted, replay falls back to the MLflow client configuration.
This follows the pip lock pattern, letting users choose between full
determinism and cross-environment portability.

Lock file format and SDK functions are in
[implementation-details.md](implementation-details.md).

### Trace integration

RFC-0008 defines `mlflow.skill_context()`, a context manager that
creates SKILL spans in MLflow traces (see RFC-0008, Trace
integration). The install commands can automate this: they write a
manifest mapping installed skill names to registry coordinates.
The recommended instrumentation approach is enhancing the MLflow
autologger to recognize skill invocations using this manifest and
emit SKILL spans with registry coordinates. Because the autologger
runs in the same process that owns the active trace, it has natural
access to thread-local trace context and produces correctly nested
parent-child span relationships.

For monolithic bundle installs, the manifest records the installed
bundle version and any registered embedded skill versions discovered
during import. Automatic per-skill SKILL spans require the local skill
name to resolve to a registered skill version in the manifest. For
harnesses where in-process autologger integration is not available,
other approaches (explicit trace correlation for hook commands,
invocation event annotation, harness-native extensions) may be
feasible. Users can always call `mlflow.skill_context()` manually
in SDK-based agent code.

Manifest format, hook configuration examples, and per-harness
instrumentation details are in
[implementation-details.md](implementation-details.md).

## Drawbacks

- **Adapter maintenance.** Each harness adapter must be maintained as
  harness plugin formats evolve. This is ongoing work.
- **Incomplete coverage.** Not all harnesses support all capability
  types. By default, installs skip unsupported types with warnings.
  Users who need fail-fast behavior can use strict mode. Even with
  warnings, users need to understand that the installed harness artifact
  can be a subset of the governed bundle.
- **Manifest format drift.** Generated manifests may not cover all
  features of a harness's native plugin format (e.g., Codex CLI's
  `interface` block with branding, or OpenClaw's `requires` field).

# Alternatives

## Let users write their own install scripts

Provide only `pull` (RFC-0008) and let users or third parties build
harness-specific tooling.

Rejected because the gap between "pull" and "working in my harness"
is the main adoption barrier. A first-party install experience is
critical for driving adoption.

## Delegate installation to an existing skill package manager

Several open-source projects already handle skill installation:

- **skills.sh** ([vercel-labs/skills](https://github.com/vercel-labs/skills)):
  CLI for installing individual SKILL.md files. Supports 70+ harnesses.
- **Lola** ([LobsterTrap/lola](https://github.com/LobsterTrap/lola)):
  Cross-harness package manager. Its "AI Context Module" format bundles
  skills, subagents, commands, hooks, and MCP servers.
- **SkillHub** ([iflytek/skillhub](https://github.com/iflytek/skillhub)):
  Self-hosted skill registry with CLI installation. Individual skills
  only, 14 harnesses.

We considered delegating installation to one of these tools rather
than implementing our own adapters. skills.sh and SkillHub operate on
individual skills in isolation and have no bundle concept, so they
cannot handle the general case of installing a skill bundle with
skills, subagents, hooks, and MCP server configurations together.
Lola is closer: its AI Context Module format supports all the member
types we need. However, delegating installation to the Lola CLI
would introduce a third-party runtime dependency for a relatively
narrow special case (Lola-format bundles targeting Lola-supported
harnesses) while still requiring our own implementation for the
general problem (any bundle format, any harness, with registry
governance and trace integration). Instead, we implement installation
ourselves via the adapter interface. The adapter interface is
extensible to harness-agnostic bundle formats (see "Harness-agnostic
bundle formats" above), so support for formats like Lola's can be
added as demand warrants without architectural changes.

# Adoption strategy

**Initial release:** Claude Code, Codex CLI, and Cursor adapters.
Bundle import. Install-time trace manifest and autologger
instrumentation for skill tracing.

**Follow-up:** Marketplace catalog generation for Claude Code /
Codex CLI. Additional adapters based on demand (including
harness-agnostic bundle formats), automatic harness detection,
bi-directional sync (detect local plugins and register them).
