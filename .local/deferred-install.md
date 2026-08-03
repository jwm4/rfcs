# Deferred Install/Lockfile/install_count Content for RFC-0008

This file preserves all install, lockfile, install_count, and
package manager content removed from RFC-0008 PR #26. This content
will be covered in a separate RFC. Content is organized by source
file and location.

---

## From: 0008-mvp-skill-registry.md

### Basic example > "Install and use" section

## Install and use

```bash
# Install a skill bundle for Claude Code via a package manager
mlflow skills bundles install --skill-uri skills:/pr-workflow@production \
    --harness claude-code

# Or install a single skill through the same package-manager layer
mlflow skills install --skill-uri skills:/code-review@production \
    --harness claude-code
```

### Summary paragraph: harness-specific installation sentence

Original: "Harness-specific installation delegates to package managers
(APM, Lola, or others via a plugin interface) that already support
cross-harness skill installation."

### Out of scope > "Custom harness adapters" bullet

- **Custom harness adapters.** This RFC does not build per-harness
  installation adapters. Instead, it delegates harness-specific
  installation to existing package managers (APM, Lola) via a plugin
  interface.

### SkillVersion narrative: install_count sentence

Each version tracks an `install_count` that the server increments on
each `install` call, enabling popularity-based discovery.

### UI fallback behavior: install_count reference

Original line: "For version-level fields shown on parent cards (e.g.,
`source_type`, `install_count`), the UI derives values from the
latest-resolved version."

### User journey: Compare agent performance > install step

2. Install the skill:
   ```bash
   mlflow skills install --skill-uri skills:/code-review@production \
       --harness claude-code
   ```

### User journey: CI pipeline > install step

3. The job installs the new bundle version and runs it against a
   benchmark dataset, collecting traces in a dedicated MLflow
   experiment.

### Package manager integration section (entire)

### Package manager integration

Rather than building custom harness adapters for each agent harness,
the skill registry delegates harness-specific installation to existing
package managers that already support cross-harness skill
installation. This avoids duplicating work that projects like
[APM](https://github.com/microsoft/apm) and
[Lola](https://github.com/LobsterTrap/lola) already handle well, and
lets the MLflow community benefit from their evolving harness support.

The registry boundary is intentionally narrow. MLflow creates
registry entries only for skills within a bundle; non-skill content
(e.g., subagents, MCP configurations) remains in the bundle source and
is included when the bundle is pulled or installed, but does not receive
individual registry entries.
[RFC-0009: Extended Skill Bundles](https://github.com/mlflow/rfcs/pull/27)
will add registry entries for non-skill member types. MLflow resolves
bundles into concrete skill
sources or local paths, then passes those to an existing package manager
for installation.

#### Installation commands

Both harness-aware installation commands delegate to a configured
package manager plugin:

1. **Single-skill install** (`mlflow skills install`): MLflow resolves
   one registered skill and materializes its content locally, then calls
   the plugin's `install_skill()` operation.

2. **Bundle install** (`mlflow skills bundles install`): MLflow resolves
   a bundle and materializes its content locally (including any non-skill
   content in monolithic bundles), then calls the plugin's
   `install_bundle()` operation.

MLflow owns registry and source resolution. The package manager owns
all harness-specific behavior, including directory placement and any
package-manager or harness manifest generation. Both
installation commands increment the server-side install count for the
installed skill or bundle version, and both accept a `--harness`
argument (auto-detected from the local environment when omitted).
Both also accept an optional `--local-path`
argument pointing to previously pulled content, which skips the fetch
from source and passes the local path directly to the package manager.
Users who only want to download content without installing it into a
harness use the package-manager-free `mlflow skills pull`
command.

`mlflow skills update` re-resolves installed references and reinstalls
skills with newer versions available. `mlflow skills uninstall` and
`mlflow skills bundles uninstall` remove installed skills or bundles.

**Why MLflow materializes locally.** MLflow resolves registry
coordinates (versions, aliases, workspaces) and fetches content from the
source before passing local paths to the package manager. This design
choice has two benefits. First, registry resolution stays in MLflow, so
every package manager plugin resolves aliases and versions the same way
without needing to interact with the registry API. Second, the registry
supports multiple source types (Git repos, OCI registries, ZIP archives,
MLflow artifact storage), and materializing locally means plugins only
need to handle local paths, not implement fetching logic for every source
type. The tradeoff is that package managers with their own remote source
capabilities (e.g., Git-aware caching or shallow clones) cannot apply
those optimizations when MLflow has already fetched the content.

**Harness selection.** When `--harness` is provided, installation
targets that specific harness. When omitted, the client detects the
local harness from the environment (e.g., presence of `.claude/`
suggests Claude Code). If detection finds exactly one harness, it is
used automatically; if multiple or none are detected, MLflow raises
an error asking the user to specify `--harness` explicitly.

**Reproducible installation.** MLflow defines a small resolution lock,
`mlflow-skills.lock`, that records the tracking server URL and the exact
registry coordinates and installation inputs selected by an install
command: entity type, name, resolved version, workspace, package manager,
harness, and scope. Aliases are resolved before they are written. Passing
`--lock-file` to either installation command writes or updates this file;
`mlflow skills install --from-lock` replays it by connecting to
the recorded tracking server, resolving the pinned registry versions, and
delegating them to the recorded package manager.

**Cached content.** When MLflow materializes skill content during
installation, it stores the pulled content in `.mlflow-skills/` at the
project root. This fixed path convention allows teams to commit pulled
skills to version control so that collaborators can install from the
local copy without needing registry access. The package manager plugin
receives paths within this directory and installs content into the
harness-specific location.

Package-manager lockfiles complement the MLflow resolution lock rather
than replace it. APM provides `apm.lock.yaml` with resolved commits and
content hashes. Lola provides version-constraint files (`.lola-req`) and
ref pinning but no lockfile. The MLflow lock makes registry resolution
reproducible across both plugins; a package-manager lock can additionally
capture package-manager-specific layout and integrity information.
Package manager plugins may write their native lockfile as part of the
install operation. Combined with the cached content in `.mlflow-skills/`,
this allows collaborators to use the package manager directly without
needing MLflow or registry access.

#### Package manager plugin interface

Package manager plugins are registered via Python entrypoints (group
`mlflow.skill_package_managers`), so third-party plugins can be
installed via `pip install` without modifying MLflow core.

```python
class PackageManagerPlugin:
    def install_skill(
        self,
        name: str,
        local_path: str,
        harness: str,
        scope: str = "project",
    ) -> PackageManagerInstallResult:
        """Install a single skill from a local path.
        Returns the installed path and harness-local skill name."""
        ...

    def install_bundle(
        self,
        bundle_name: str,
        member_paths: dict[str, str],
        harness: str,
        bundle_path: str | None = None,
        scope: str = "project",
    ) -> PackageManagerInstallResult:
        """Install a bundle from local paths. For monolithic bundles,
        bundle_path is the complete artifact root, including opaque
        non-skill content. Returns the installed path and mapping from
        registry skill names to harness-local names."""
        ...

    def supported_harnesses(self) -> list[str]:
        """Return list of harness identifiers this plugin supports."""
        ...

    def check_requirements(self) -> PackageManagerCheckResult:
        """Verify the package manager is installed and meets minimum
        version requirements. Called before install operations."""
        ...
```

**Discovering installed plugins.** `mlflow skills
list-package-managers` enumerates the package manager plugins
registered via the `mlflow.skill_package_managers` entrypoint group
and calls `supported_harnesses()` on each one, displaying both the
available package managers and the harnesses they support.

#### Source resolution flow

When `mlflow skills bundles install` is invoked:

1. MLflow resolves the bundle version from the registry (by name +
   version or alias).
2. MLflow materializes a local path for each member skill according to
   the bundle kind:
   - For an assembled bundle, MLflow pulls each member from its own
     source to `.mlflow-skills/`.
   - For a monolithic bundle, MLflow pulls the bundle source once and
     retains the complete pulled root as `bundle_path`, including opaque
     non-skill content, while resolving each skill path from that root
     and the member URI's `#subpath` fragment.
3. MLflow passes the skill-name-to-local-path mapping and, for a
   monolithic bundle, the complete `bundle_path` to the configured
   package manager plugin. The plugin handles harness-specific placement
   of the entire bundle and returns the actual harness-local name for
   each installed skill.
4. If requested, MLflow updates the resolution lock with the exact
   bundle version and installation inputs.

When `mlflow skills install` is invoked for a single skill, MLflow
resolves the version, pulls its content to `.mlflow-skills/`,
passes that path to the configured plugin's `install_skill()` operation.
If requested, it then updates
the resolution lock with the exact skill version and installation inputs.

### Drawback: "Package manager dependency"

- **Package manager dependency.** Full harness-specific installation
  requires a package manager plugin (APM, Lola, or similar). Users
  who do not install a package manager can still use `mlflow
  skills pull` for harness-agnostic content download.

### Alternative: "Build custom harness adapters in MLflow"

## Build custom harness adapters in MLflow

Build per-harness installation adapters within MLflow (as proposed in
the earlier version of this RFC).

Rejected in favor of delegating to existing package managers. APM and
Lola already support 8+ and 6+ harnesses respectively, and their
harness support evolves independently of MLflow releases. Building
custom adapters would duplicate this work and create an ongoing
maintenance burden as harness plugin formats evolve. The plugin
interface allows MLflow to integrate with any package manager without
coupling to a specific one.

### Alternative: "Use APM or Lola directly without a registry"

## Use APM or Lola directly without a registry

Use a client-side package manager (APM, Lola, or `gh skill install`)
as the sole mechanism for skill management.

These tools solve the client-side "make it portable and reproducible"
problem well. However, they are not server-side registries and do not
provide the governance features that enterprises need:

- **Lifecycle management.** No concept of draft, active, deprecated,
  or deleted status. No way to signal consumers that a skill version
  is deprecated or approved for production.
- **Rich discovery.** Limited search and metadata capabilities. No
  centralized catalog with tags, descriptions, and compatibility
  information.
- **RBAC and workspace scoping.** No per-user or per-team access
  controls. No visibility boundaries between teams or projects.

The skill registry and package managers are complementary: the
registry provides the server-side governance and discovery layer,
while package managers handle client-side
installation and harness-specific adaptation.

### Adoption strategy: install/lockfile/plugin deliverables

Original: "This RFC delivers Skill and SkillBundle entities, store,
REST API, SDK, CLI, UI, `mlflow skills pull`, plugin import,
package-manager-backed single-skill and bundle installation, the
package manager plugin interface, and the `mlflow-skills.lock`
resolution lock."

---

## From: implementation-details.md

### install_count in DB schema

skill_versions table, line 40:
| `install_count` | `BigInteger` | default `0`; incremented server-side on each install |

skill_bundle_versions table, line 112:
| `install_count` | `BigInteger` | default `0`; incremented server-side on each install |

### install_count in entity dataclasses

SkillVersion, line 254:
    install_count: int = 0

SkillBundleVersion, line 439:
    install_count: int = 0

### install_count in response models

SkillVersionResponse, line 1356:
    install_count: int = 0

SkillBundleVersionResponse, line 1389:
    install_count: int = 0

### Supporting dataclasses

```python
@dataclass(frozen=True)
class InstalledSkill:
    registry_name: str
    harness_local_name: str
    installed_path: str


@dataclass
class PackageManagerInstallResult:
    installed_path: str
    skills: list[InstalledSkill]


@dataclass
class PackageManagerInfo:
    name: str
    harnesses: list[str]


@dataclass
class PackageManagerCheckResult:
    satisfied: bool
    message: str | None = None


@dataclass(frozen=True)
class MlflowSkillLockEntry:
    entity_type: str
    name: str
    version: int
    workspace: str
    package_manager: str
    harness: str
    scope: str
```

### SDK functions

```python
def install_skill(
    *,
    name: str,
    harness: str | None = None,
    version: int | None = None,
    alias: str | None = None,
    package_manager: str | None = None,
    scope: str = "project",
    lock_file: str | None = None,
    local_path: str | None = None,
) -> PackageManagerInstallResult:
    """Resolve a skill and install it through a package manager plugin.
    If harness is omitted, the client detects the local harness from
    the environment (e.g., presence of .claude/ suggests Claude Code).
    If lock_file is provided, record the exact resolved version and
    installation inputs for replay. If local_path is provided, skip
    fetching from source and pass the local path directly to the
    package manager."""


def install_bundle(
    *,
    name: str,
    harness: str | None = None,
    version: int | None = None,
    alias: str | None = None,
    package_manager: str | None = None,
    scope: str = "project",
    lock_file: str | None = None,
    local_path: str | None = None,
) -> PackageManagerInstallResult:
    """Resolve a bundle and install it through a package manager plugin.
    For monolithic bundles, non-skill content is included in the
    installed artifact. If harness is omitted, the client detects the
    local harness from the environment. If lock_file is provided,
    record the exact resolved version and installation inputs for
    replay. If local_path is provided, skip fetching from source
    and pass the local path directly to the package manager."""


def update_installed_skills() -> list[InstalledSkill]: ...


def uninstall_skill(*, name: str) -> None: ...


def uninstall_bundle(*, name: str) -> None: ...


def install_from_lock(
    *, lock_file: str = "mlflow-skills.lock",
) -> list[PackageManagerInstallResult]:
    """Replay exact skill and bundle versions from an MLflow resolution
    lock using the recorded package manager, harness, and scope."""


def list_package_managers() -> list[PackageManagerInfo]:
    """List installed package manager plugins and their supported
    harnesses. Discovers plugins via the entrypoint mechanism."""
```

### CLI mapping table rows

| `mlflow skills install` | `install_skill()` | Install one skill through a package manager plugin; `--local-path` skips fetch |
| `mlflow skills bundles install` | `install_bundle()` | Install a bundle through a package manager plugin; `--local-path` skips fetch |
| `mlflow skills install --from-lock` | `install_from_lock()` | Replay exact registry versions from an MLflow resolution lock |
| `mlflow skills update` | `update_installed_skills()` | Re-resolve installed references and reinstall skills with newer versions |
| `mlflow skills uninstall` | `uninstall_skill()` | Uninstall a skill |
| `mlflow skills bundles uninstall` | `uninstall_bundle()` | Uninstall a bundle |
| `mlflow skills list-package-managers` | `list_package_managers()` | List installed package manager plugins and their supported harnesses |

### MLflow resolution lock section (entire)

## MLflow resolution lock

Package managers receive materialized local paths, so their own
lockfiles cannot by themselves reconstruct which MLflow registry
versions produced those paths. When `lock_file` is supplied to
`install_skill()` or `install_bundle()`, MLflow writes or updates an
`mlflow-skills.lock` resolution lock after installation succeeds.

The lock file records the tracking server URL used to resolve the
entries, so `install --from-lock` can connect to the correct server
without requiring the user to configure it separately. Each entry
records the entity type (`skill` or `bundle`), name, exact resolved
version, workspace, selected package manager, harness, and scope.
Aliases and `latest` are resolved before writing the entry and are
never stored in place of a version. A bundle entry does not repeat
its members because bundle membership is immutable and can be recovered
from the exact bundle version.

A resolution lock is scoped to one workspace and one tracking server.
Appending an entry from a different workspace fails, and replay
connects to the recorded tracking server.

`install_from_lock()` reads the entries, resolves the exact versions
through the currently configured MLflow client, materializes their
content, and delegates to the recorded package manager. Normal registry
visibility and lifecycle rules apply during replay, so an unavailable or
deleted version causes the replay to fail rather than silently installing
different content. Package-manager lockfiles may additionally capture
package-manager-specific layout, cached sources, and integrity metadata.
The CLI `--from-lock` mode uses the recorded installation inputs and is
mutually exclusive with name, version, alias, package manager, harness,
scope, and lock-writing options.

### Package manager plugin interface section (entire)

## Package manager plugin interface

Package manager plugins are registered via Python entrypoints (group
`mlflow.skill_package_managers`), so third-party plugins can be
installed via `pip install` without modifying MLflow core.

These plugins receive resolved skills or bundle content
(which may include non-skill content in monolithic bundles). They
install the content using an existing package manager. The package
manager handles placement of all content, including non-skill files
that do not have individual registry entries. It returns the actual
harness-local name of every installed skill. The result must contain
exactly one `InstalledSkill` for every requested registry skill;
missing or duplicate mappings fail the install before MLflow writes
its resolution lock.

Both `mlflow skills install` and `mlflow skills
bundles install` require a package manager plugin. The caller can select
a plugin explicitly, or MLflow uses the configured default. If no plugin
is selected or available, installation fails with guidance to install or
configure one; `mlflow skills pull` remains available without a
package manager.

### Plugin protocol

```python
class PackageManagerPlugin:
    def install_skill(
        self,
        name: str,
        local_path: str,
        harness: str,
        scope: str = "project",
    ) -> PackageManagerInstallResult:
        """Install a single skill from a local path.
        Returns the installed path and harness-local skill name.

        Args:
            name: registry skill name
            local_path: local directory with skill content
            harness: target harness (e.g., "claude-code", "cursor")
            scope: "project" (cwd) or "user" (home directory)
        """
        ...

    def install_bundle(
        self,
        bundle_name: str,
        member_paths: dict[str, str],
        harness: str,
        bundle_path: str | None = None,
        scope: str = "project",
    ) -> PackageManagerInstallResult:
        """Install a bundle from local paths. member_paths maps registry
        skill names to local paths. For a monolithic bundle, bundle_path
        is the complete artifact root and must be installed as a unit.
        Returns the installed path and harness-local skill names.

        Args:
            bundle_name: registry bundle name
            member_paths: {skill_name: local_path} for each member
            harness: target harness (e.g., "claude-code", "cursor")
            bundle_path: complete monolithic bundle root, or None for an
                assembled bundle
            scope: "project" or "user"
        """
        ...

    def supported_harnesses(self) -> list[str]:
        """Return list of harness identifiers this plugin supports.
        E.g., ["claude-code", "cursor", "codex-cli", "copilot"]."""
        ...

    def check_requirements(self) -> PackageManagerCheckResult:
        """Verify the package manager is installed and meets minimum
        version requirements. Called before install operations."""
        ...
```

### Entrypoint registration

```toml
# In the package manager plugin's pyproject.toml:
[project.entry-points."mlflow.skill_package_managers"]
apm = "apm_mlflow:ApmPlugin"
lola = "lola_mlflow:LolaPlugin"
```

### Bundle installation flow

When `mlflow skills bundles install` is invoked:

1. **Resolve:** MLflow calls `get_skill_bundle_version()` (or alias
   resolution) to obtain the bundle version and its member list.
2. **Materialize member paths:**
   - For an assembled bundle, MLflow pulls each member skill to its own
     subdirectory under `.mlflow-skills/` using source-type-aware logic
     (Git clone, OCI pull, ZIP download, or MLflow artifact download).
   - For a monolithic bundle, MLflow pulls the bundle-level source once
     using the same source-type-aware logic and retains the complete root
     as `bundle_path`, including opaque non-skill content. For each
     member, it resolves a local path by joining the pulled bundle root
     with the `#subpath` fragment from the member URI. Every monolithic
     member URI must include a `#subpath` fragment; installation fails
     if the fragment is missing, if the resolved path escapes the
     pulled bundle root after normalization, or if the path does not
     contain the embedded skill.
3. **Delegate:** MLflow passes `member_paths` and, for a monolithic
   bundle, `bundle_path` to the configured package manager plugin via
   `install_bundle()`. The plugin installs the complete monolithic bundle
   or the assembled skills and returns each skill's harness-local name.
4. **Resolution lock:** If `lock_file` was supplied, MLflow atomically
   updates it with the exact resolved bundle version and installation
   inputs after the install succeeds.

### Single-skill installation flow

When `mlflow skills install` is invoked:

1. **Resolve:** MLflow calls `get_skill_version()`, alias resolution, or
   latest resolution to obtain the registered source pointer. `version`
   and `alias` are mutually exclusive; omitting both selects the
   system-defined latest version.
2. **Pull:** MLflow pulls the skill content to `.mlflow-skills/` at the
   project root using the same source-type-aware logic as `pull`.
3. **Delegate:** MLflow passes the skill name and local path to the
   configured package manager plugin via `install_skill()`. The plugin
   owns harness-specific behavior, scope handling, directory placement,
   naming, and any generated package-manager or harness metadata. An
   explicit harness selection from the caller is passed through to the
   plugin, which returns the actual harness-local skill name.
4. **Resolution lock:** If `lock_file` was supplied, MLflow atomically
   updates it with the exact resolved skill version and installation
   inputs after the install succeeds.

### Interleaved references (not standalone sections)

These phrases/sentences were edited or removed when deferring
install content. Listed here so the new RFC can reference them.

**Main RFC (0008-mvp-skill-registry.md):**

- **Summary paragraph**: removed "Harness-specific installation
  delegates to package managers (APM, Lola, or others via a plugin
  interface) that already support cross-harness skill installation."
- **Out of scope**: removed "Custom harness adapters" bullet about
  delegating to package managers.
- **UI fallback behavior**: removed `install_count` from the
  example list of version-level fields on parent cards.
- **SkillVersion narrative**: removed "Each version tracks an
  `install_count` that the server increments on each `install` call,
  enabling popularity-based discovery."
- **Installation commands paragraph**: removed reference to
  incrementing install count.
- **User journey "Compare agent performance"**: changed install
  step to use `mlflow skills pull` + manual harness installation.
- **User journey "CI pipeline"**: changed install step to use
  `mlflow skills pull` + manual harness installation.
- **Pull semantics**: removed reference to package manager plugins
  for harness-specific installation.
- **Adoption strategy**: removed "package-manager-backed single-skill
  and bundle installation, the package manager plugin interface, and
  the `mlflow-skills.lock` resolution lock" from deliverables list.
- **TOC**: removed "Package manager integration" entry.

**Implementation details (implementation-details.md):**

- **Intro line**: removed "package manager plugin interface".
- **install_count in DB schema**: removed from both skill_versions
  and skill_bundle_versions tables.
- **install_count in entity dataclasses**: removed from SkillVersion
  and SkillBundleVersion.
- **install_count in response models**: removed from
  SkillVersionResponse and SkillBundleVersionResponse.
- **Plugin interface narrative**: removed reference about failing
  install before writing resolution lock.
- **SDK intro paragraph**: removed "package-manager installation
  operations" from the list of operations in `mlflow.genai`.
- **Existing subcommand conflict note**: changed `install` to `pull`
  in the list of example subcommand names that avoid conflicts.
- **User journey "Compare agent performance" (bundle steps)**: changed
  "Install version 1/2" to "Pull version 1/2, install it into the
  harness manually" for consistency with the single-skill journey edits.
- **UI skill card fields**: removed "Install count" from the bullet list
  of fields shown on skill cards.
