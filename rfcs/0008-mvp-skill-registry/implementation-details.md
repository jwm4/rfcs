# RFC-0008: Skill Registry Implementation Details

This document contains implementation-level specifications for
RFC-0008 (Skill Registry). It covers database schema, entity
dataclasses, store interface method signatures, REST API endpoints,
pagination/filtering, SDK convenience functions, and CLI mapping. These details
support implementers; the main RFC covers the design rationale.

## Database schema

Tables are created via a single Alembic migration. All tables are
workspace-scoped.

### `skills`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `organization` | `String(256)` | PK, default `''` (empty string) |
| `name` | `String(256)` | PK |
| `description` | `String(5000)` | |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

PrimaryKey: `(workspace, organization, name)`.

### `skill_versions`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `skills` |
| `organization` | `String(256)` | PK, FK to `skills` |
| `name` | `String(256)` | PK, FK to `skills` |
| `version` | `Integer` | PK, server-assigned monotonic integer |
| `source_type` | `String(20)` | nullable; `git`, `oci`, `zip`, `mlflow` |
| `source` | `String(2048)` | nullable pointer to skill content |
| `subpath` | `String(2048)` | nullable; path within the artifact |
| `status` | `String(20)` | default `'draft'` |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, organization, name)` references `skills`, CASCADE
delete. This supports administrative hard deletion of the parent
`Skill`; normal version deletion is a status transition to `deleted`
and does not physically remove the version row.

**Version ordering**: versions are monotonic integers assigned by
the server. Each new version for a given skill receives
the next integer. Ordering is a simple integer comparison.

**Index**: `ix_skill_versions_latest_lookup` on `(workspace,
organization, name, status, version)` supports latest-resolution
lookups.

### `skill_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `skills` |
| `organization` | `String(256)` | PK, FK to `skills` |
| `name` | `String(256)` | PK, FK to `skills` |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_version_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `skill_versions` |
| `organization` | `String(256)` | PK, FK to `skill_versions` |
| `name` | `String(256)` | PK, FK to `skill_versions` |
| `version` | `Integer` | PK, FK to `skill_versions` |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `skills` |
| `organization` | `String(256)` | PK, FK to `skills` |
| `name` | `String(256)` | PK, FK to `skills` |
| `alias` | `String(256)` | PK |
| `version` | `Integer` | target version |

### `agent_plugins`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `organization` | `String(256)` | PK, default `''` (empty string) |
| `name` | `String(256)` | PK |
| `description` | `String(5000)` | |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

PrimaryKey: `(workspace, organization, name)`.

### `agent_plugin_versions`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `agent_plugins` |
| `organization` | `String(256)` | PK, FK to `agent_plugins` |
| `name` | `String(256)` | PK, FK to `agent_plugins` |
| `version` | `Integer` | PK, server-assigned monotonic integer |
| `source_type` | `String(20)` | optional; `git`, `oci`, `zip`, `mlflow` |
| `source` | `String(2048)` | optional pointer to agent plugin |
| `subpath` | `String(2048)` | nullable; path within the artifact |
| `status` | `String(20)` | default `'draft'` |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, organization, name)` references `agent_plugins`,
CASCADE delete. Version ordering and index follow the same pattern
as `skill_versions`.

### `agent_plugin_version_members`

| Column | Type | Notes |
|--------|------|-------|
| `plugin_workspace` | `String(63)` | PK, FK to `agent_plugin_versions` |
| `plugin_organization` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `plugin_name` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `plugin_version` | `Integer` | PK, FK to `agent_plugin_versions` |
| `member_organization` | `String(256)` | PK, FK to `skill_versions` |
| `member_name` | `String(256)` | PK, FK to `skill_versions` |
| `member_version` | `Integer` | PK, FK to `skill_versions` |
| `member_subpath` | `String(2048)` | nullable; parsed from `#subpath` fragment of member URI |

FK: `(plugin_workspace, plugin_organization, plugin_name,
plugin_version)` references `agent_plugin_versions`, CASCADE
delete. A FK to `skill_versions` via `(plugin_workspace,
member_organization, member_name, member_version)` enforces
referential integrity with RESTRICT delete. Skills and agent
plugins share the same workspace; `plugin_workspace` is reused for
the skill FK.

### `agent_plugin_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `agent_plugins` |
| `organization` | `String(256)` | PK, FK to `agent_plugins` |
| `name` | `String(256)` | PK, FK to `agent_plugins` |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `agent_plugin_version_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `agent_plugin_versions` |
| `organization` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `name` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `version` | `Integer` | PK, FK to `agent_plugin_versions` |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `agent_plugin_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `agent_plugins` |
| `organization` | `String(256)` | PK, FK to `agent_plugins` |
| `name` | `String(256)` | PK, FK to `agent_plugins` |
| `alias` | `String(256)` | PK |
| `version` | `Integer` | target agent plugin version |

**Workspace handling.** All tables carry a `workspace` column as part
of the composite key. Single-tenant deployments use `'default'`.
Child tables reference the parent by `(workspace, organization,
name)`, so workspace is part of every table's primary key.

**Timestamps.** Set at the application layer via
`get_current_time_millis()`, not via DDL defaults.

**Deletion semantics.** The registry follows the mixed deletion pattern
used by the Model Registry and RFC-0004:

- Top-level entity delete operations (`delete_skill` and
  `delete_agent_plugin`) are administrative hard deletes. They
  physically remove the parent row and cascade to child rows, subject
  to referential-integrity checks.
- Version delete operations (`delete_skill_version` and
  `delete_agent_plugin_version`) are soft deletes. They set
  `status='deleted'` when allowed by the lifecycle transition rules,
  update `last_updated_timestamp`, remove aliases that point to the
  deleted version, and exclude the version from normal
  get/search/list/latest resolution. Active versions must first be
  unpublished or deprecated before they can be deleted.
- The `deleted` status is terminal. Internal audit or provenance paths
  may retain enough metadata to explain historical agent plugin
  snapshots, but deleted versions are not surfaced to consumers.

## Entity dataclasses

### Skill entity

```python
from dataclasses import dataclass, field
from enum import StrEnum


class SkillStatus(StrEnum):
    DRAFT = "draft"
    ACTIVE = "active"
    DEPRECATED = "deprecated"
    DELETED = "deleted"


@dataclass
class Skill:
    name: str
    organization: str = ""
    description: str | None = None
    workspace: str | None = None
    status: SkillStatus | None = None  # read-only, derived from parent-resolved version
    tags: dict[str, str] = field(default_factory=dict)
    aliases: dict[str, int] = field(default_factory=dict)  # read-only; populated from skill_aliases table, e.g. {"production": 2}
    latest_version: int | None = None  # read-only, shared latest-resolution rule
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Human-readable name, unique within `(workspace, organization)` |
| `organization` | `str` | Scopes ownership (e.g., team or publisher); defaults to `""` (empty string) |
| `description` | `str` | Optional human-readable description of the skill |
| `status` | `SkillStatus` | Read-only; derived from the parent-resolved version: highest active version number if present, otherwise highest non-deleted non-active version number |
| `aliases` | `dict[str, int]` | Stable version pointers (e.g., `{"production": 2}`); read-only, populated from `skill_aliases` table |
| `latest_version` | `int` | Read-only; highest version number among `active` versions if one exists, otherwise highest non-`deleted` non-`active` version |
| `workspace` | `str` | Visibility boundary |

### SkillVersion entity

```python
class SkillSourceType(StrEnum):
    GIT = "git"
    OCI = "oci"
    ZIP = "zip"
    MLFLOW = "mlflow"


@dataclass
class SkillVersion:
    name: str
    version: int
    organization: str = ""
    source_type: SkillSourceType | None = None
    source: str | None = None
    subpath: str | None = None
    status: SkillStatus = SkillStatus.DRAFT

    tags: dict[str, str] = field(default_factory=dict)
    aliases: list[str] = field(default_factory=list)
    workspace: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Skill name (part of composite key with workspace and organization) |
| `version` | `int` | Server-assigned monotonic integer. Each new version receives the next integer |
| `organization` | `str` | Organization scope, from parent Skill |
| `source_type` | `SkillSourceType` | Server-inferred distribution mechanism: `git`, `oci`, `zip`, `mlflow`. Inferred from the `source` URL scheme |
| `source` | `str` | Pointer to the content in the source system. Required for standalone pull. May be omitted only when the version's content lives within an agent plugin-level artifact, in which case the containing agent plugin membership identifies the embedded content path |
| `subpath` | `str` | Optional path within the artifact where this skill's content lives. See subpath usage table below |
| `status` | `SkillStatus` | Per-version lifecycle: `draft`, `active`, `deprecated`, `deleted` |
| `aliases` | `list[str]` | Alias names currently pointing at this version (read-only, projected from alias table) |

### SkillVersion field details

**Source type extensibility.** The `source_type` enum is intentionally
small for the initial implementation. New source types (e.g., `s3`,
`azure-blob`, `opensharing`) can be added without schema changes
since the column stores a string value. In particular, the
[OpenSharing](https://github.com/OpenSharing-IO/OpenSharing) protocol
(Linux Foundation) defines AgentSkill as a first-class asset type
using the same SKILL.md directory structure. An `opensharing` source
type would let the registry govern and track skills whose content is
shared via OpenSharing's credential-vending protocol.

**Subpath usage by source type.** The `subpath` field separates "what
to download" from "where inside the downloaded content the relevant
asset lives." Its applicability varies by source type:

| Source type | `subpath` usage |
|---|---|
| `oci` | Path within the OCI image (e.g., `plugins/code-review`). Used when multiple skills share a single image. |
| `zip` | Path within the archive (e.g., `plugins/code-review`). Used when multiple skills share a single archive. |
| `git` | Path within the repository (e.g., `code-review`). Used when the skill content is not at the repository root. The `source` field contains the clone URL with `@<ref>` suffix; `subpath` locates the content within the repo. |
| `mlflow` | Not used. The artifact path is derived from the skill ID and version. |

**Git source format.** For `source_type="git"`, `source` is a Git
clone URL with an `@<ref>` suffix to identify the branch, tag, or
commit (e.g., `https://github.com/acme/agent-skills.git@v1.0.0`).
The `subpath` field identifies the path within the repository where
the skill content lives (e.g., `code-review`). This separates the
clone target, the ref, and the content path into distinct fields
rather than relying on hosting-provider-specific tree URL conventions.
Mutable refs (branches, tags) are allowed.

**MLflow artifact storage (`source_type="mlflow"`).** In addition to
external source pointers, the registry supports storing skill content
directly in MLflow's artifact storage. This serves users who do not
have external Git/OCI infrastructure, who want agent capabilities
stored alongside their models, or who operate in airgapped
environments where external sources are not reachable.

Content is stored as a directory tree of individual files under a
controlled artifact path derived from the skill's identity and
version. Workspace scoping is handled at the artifact store level
(each workspace has its own artifact root), so the path within the
store is `skills/<organization>/<name>/<version>/`. When
organization is empty, the path uses `_` as a placeholder (e.g.,
`skills/_/<name>/<version>/`). This is consistent with how MLflow
stores model artifacts. The `source` field is null for
MLflow-stored content; the system knows where to find it by
convention. Pull downloads the directory tree from the artifact
store. The MLflow UI can browse individual files within a stored
skill version when artifact proxying is enabled.

The same artifact path convention applies to agent plugins stored
with `source_type="mlflow"`: `agent-plugins/<organization>/<name>/<version>/`,
with `_` for empty organization.

**Client-side upload flow.** When `source` is a local path (detected
by the absence of a `://` scheme), the SDK uploads the content to
MLflow artifact storage rather than treating it as a remote pointer:

1. The client validates the local directory and preflights that the
   path can be registered.
2. The client creates the `SkillVersion` with `source` set to null.
   The server assigns the version number and infers
   `source_type="mlflow"` from the null source.
3. Using the returned version number, the client uploads each file
   through MLflow's existing artifact APIs to the controlled artifact
   prefix (`skills/<organization>/<name>/<version>/`, with `_` for
   empty organization).

Version creation and upload are not atomic. If upload fails after the
version record is created, the version exists with no content. The
client makes a best-effort attempt to delete the version record and
any partially uploaded files. A backend without deletion support can
retain unreferenced uploaded files until garbage collection.

**Version uniqueness.** The combination of
`(workspace, organization, name, version)` is unique. A skill
version represents a single logical version of a capability;
`source` describes where to find it but is not part of its
identity.

**Immutability contract.** `source_type`, `source`, `subpath`,
and `version` are immutable after creation. To point
to different content, register a new version. Mutable fields
(`status`, `tags`) can be updated independently.

### AgentPlugin entity

An agent plugin groups related skills into a governed unit that maps
to the "plugin" concept in agent harnesses. Follows the same
top-level pattern as Skill: versions, tags, and aliases.

```python
@dataclass
class AgentPlugin:
    name: str
    organization: str = ""
    description: str | None = None
    workspace: str | None = None
    status: SkillStatus | None = None  # read-only, derived from parent-resolved version
    tags: dict[str, str] = field(default_factory=dict)
    aliases: dict[str, int] = field(default_factory=dict)  # read-only; populated from agent_plugin_aliases table
    latest_version: int | None = None  # read-only, shared latest-resolution rule
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

`AgentPlugin.status` is read-only and uses the same parent-resolved
version rule as `Skill`: highest active version number if present,
otherwise highest non-deleted non-active version number. Latest
version resolution follows the same fallback.

### AgentPluginVersion entity

A versioned snapshot of an agent plugin's membership. In this RFC, all
members are skills.

Agent plugin members are referenced by URI string rather than a separate
data class. The URI format follows MLflow's `models:/name/version`
convention:

- `skills:/name/version` pins a specific version
- `skills:/name@alias` resolves through an alias
- `skills:/name/version#subpath` identifies an embedded skill inside
  a monolithic agent plugin artifact (subpath relative to the agent plugin root)

```python
@dataclass
class AgentPluginVersion:
    name: str
    version: int
    organization: str = ""
    source_type: SkillSourceType | None = None
    source: str | None = None
    subpath: str | None = None

    status: SkillStatus = SkillStatus.DRAFT
    tags: dict[str, str] = field(default_factory=dict)
    skills: list[str] = field(default_factory=list)
    aliases: list[str] = field(default_factory=list)
    workspace: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

### AgentPluginVersion field details

**Version uniqueness.** The combination of
`(workspace, organization, name, version)` is unique.

**Agent plugin-level source.** An agent plugin version is either monolithic or
assembled, never both. All versions of a given agent plugin must be the
same kind; the server rejects a version whose kind differs from
existing versions of the same agent plugin.

- **Monolithic:** has its own `source_type`, `source`, and `subpath`,
  pointing to a single artifact (e.g., an OCI
  image or Git repo) that contains the complete agent plugin. `pull`
  fetches the agent plugin as a unit. Member skill versions may
  omit their own `source` because the agent plugin is the
  authoritative source. Every source-less member must include a
  `#subpath` fragment in its URI identifying where it lives inside
  the agent plugin (e.g., `skills:/name/1#skills/name`).
- **Assembled:** has individual member references. Each skill member
  has its own source. `pull` fetches members individually. If a skill
  member has no source, `pull` fails rather than producing a partial
  local agent plugin. For assembled agent plugins, member URIs must not include
  a `#subpath` fragment because the member's own `source` and
  `subpath` identify its content.

The API rejects a member URI with a `#subpath` fragment when the
member version has its own source. It also rejects a source-less member
of a monolithic agent plugin when the URI lacks a `#subpath` fragment.

**Immutability contract.** The member list and source fields of an
agent plugin version are immutable after creation. To change the set of
members or source pointer, register a new agent plugin version. Mutable
fields (`status`, `tags`) can be updated independently.

Correctness of the artifact layout is the publisher's responsibility;
the registry does not validate artifact contents at registration time.

A member can appear in multiple agent plugins and multiple agent plugin versions.
Membership is at the version level, so an agent plugin version is a
reproducible snapshot of "these specific skill versions work together."

### Skill URI format

Skill URIs are used for CLI target identification and agent plugin
member lists, following the `models:/name/version` convention
established by MLflow's Model Registry. The Python SDK and REST
API continue to use separate `name`, `organization`, `version`,
and `alias` parameters for primary resource identification.

| Pattern | Meaning | Example |
|---------|---------|---------|
| `skills:/name` | Skill with no organization | `skills:/code-review` |
| `skills:/name/version` | Pin a specific version | `skills:/code-review/1` |
| `skills:/name@alias` | Resolve through an alias | `skills:/code-review@production` |
| `skills:/org/name` | Skill with organization | `skills:/acme/code-review` |
| `skills:/org/name/version` | Pin version with organization | `skills:/acme/code-review/1` |
| `skills:/org/name@alias` | Alias with organization | `skills:/acme/code-review@production` |
| `skills:/name/version#subpath` | Embedded skill in monolithic agent plugin | `skills:/review/1#skills/review` |

**URI disambiguation.** When three path segments are present (e.g.,
`skills:/a/b/1`), the first is the organization, the second is the
name, and the third (an integer) is the version. When two segments
are present, the second is either a version (if it parses as an
integer) or the URI is `organization/name` (if it does not). When
one segment is present, it is the name with no organization.

In the CLI, every command that targets a skill or agent plugin accepts a
`--skill-uri` flag (parallel to `--model-uri` in `mlflow models`).
In agent plugin member lists, URIs appear as plain strings in `list[str]`.
The server parses the URI into its constituent fields
(`member_organization`, `member_name`, `member_version`,
`member_subpath`) for storage and validation. Alias URIs are
resolved to a concrete version at the time of the API call.

## Store interface

The store interface follows the mixin pattern established by the MCP
Server Registry (RFC-0004). Methods raise `NotImplementedError` rather
than using `@abstractmethod`, allowing stores that do not support
skills (e.g., `FileStore`) to work without stubbing every method.

In the store interface, `delete_*` methods on top-level entities are
hard deletes, while `delete_*_version` methods are soft deletes that
transition the version to `deleted`.

```python
from mlflow.store.tracking import SEARCH_MAX_RESULTS_DEFAULT


NOT_SET = object()


class SkillRegistryMixin:
    # Methods raise NotImplementedError rather than using @abstractmethod,
    # following the GatewayStoreMixin pattern. This allows stores that don't
    # support skills (e.g., FileStore) to work without stubbing every method.

    # --- Skill operations ---

    def create_skill(
        self, name: str,
        organization: str = "",
        description: str | None = None,
    ) -> Skill:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill(
        self, name: str, organization: str = "",
    ) -> Skill:
        raise NotImplementedError(self.__class__.__name__)

    def search_skills(
        self,
        filter_string: str | None = None,
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[Skill]:
        raise NotImplementedError(self.__class__.__name__)

    def update_skill(
        self,
        name: str,
        organization: str = "",
        description: str | None = NOT_SET,
    ) -> Skill:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill(
        self, name: str, organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- SkillVersion operations ---

    def create_skill_version(
        self,
        name: str,
        organization: str = "",
        source_type: str | None = None,
        source: str | None = None,
        subpath: str | None = None,
        status: str = "draft",
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_version(
        self, name: str, version: int,
        organization: str = "",
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_version_by_alias(
        self, name: str, alias: str,
        organization: str = "",
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_latest_skill_version(
        self, name: str, organization: str = "",
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def search_skill_versions(
        self,
        name: str,
        organization: str = "",
        filter_string: str | None = None,
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillVersion]:
        raise NotImplementedError(self.__class__.__name__)

    def update_skill_version(
        self,
        name: str,
        version: int,
        organization: str = "",
        status: SkillStatus | None = NOT_SET,
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_version(
        self, name: str, version: int,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- Skill tag operations ---

    def set_skill_tag(
        self, name: str, key: str, value: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_tag(
        self, name: str, key: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def set_skill_version_tag(
        self, name: str, version: int,
        key: str, value: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_version_tag(
        self, name: str, version: int, key: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- Skill alias operations ---

    def set_skill_alias(
        self, name: str, alias: str, version: int,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_alias(
        self, name: str, alias: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- AgentPlugin operations ---

    def create_agent_plugin(
        self, name: str,
        organization: str = "",
        description: str | None = None,
    ) -> AgentPlugin:
        raise NotImplementedError(self.__class__.__name__)

    def get_agent_plugin(
        self, name: str, organization: str = "",
    ) -> AgentPlugin:
        raise NotImplementedError(self.__class__.__name__)

    def search_agent_plugins(
        self,
        filter_string: str | None = None,
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[AgentPlugin]:
        raise NotImplementedError(self.__class__.__name__)

    def update_agent_plugin(
        self,
        name: str,
        organization: str = "",
        description: str | None = NOT_SET,
    ) -> AgentPlugin:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin(
        self, name: str, organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- AgentPluginVersion operations ---

    def create_agent_plugin_version(
        self,
        name: str,
        organization: str = "",
        skills: list[str] | None = None,
        source_type: str | None = None,
        source: str | None = None,
        subpath: str | None = None,
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_agent_plugin_version(
        self, name: str, version: int,
        organization: str = "",
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_agent_plugin_version_by_alias(
        self, name: str, alias: str,
        organization: str = "",
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_latest_agent_plugin_version(
        self, name: str, organization: str = "",
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def search_agent_plugin_versions(
        self,
        name: str,
        organization: str = "",
        filter_string: str | None = None,
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[AgentPluginVersion]:
        raise NotImplementedError(self.__class__.__name__)

    def update_agent_plugin_version(
        self,
        name: str,
        version: int,
        organization: str = "",
        status: SkillStatus | None = NOT_SET,
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_version(
        self, name: str, version: int,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- AgentPlugin tag operations ---

    def set_agent_plugin_tag(
        self, name: str, key: str, value: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_tag(
        self, name: str, key: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def set_agent_plugin_version_tag(
        self, name: str, version: int,
        key: str, value: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_version_tag(
        self, name: str, version: int, key: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- AgentPlugin alias operations ---

    def set_agent_plugin_alias(
        self, name: str, alias: str, version: int,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_alias(
        self, name: str, alias: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)
```

The alias name `latest` is reserved for both skills and agent plugins.
The corresponding `set_*_alias()` methods reject it. Alias lookup with
`latest` delegates to `get_latest_skill_version()` or
`get_latest_agent_plugin_version()` rather than reading a stored alias
row.

For update fields, omitting a parameter leaves the stored value
unchanged, while passing `None` to a nullable field explicitly sets
the field to `null`.

All store methods that identify a specific entity use `name` and
`organization` (with `organization` defaulting to `""` when
omitted). Workspace scoping is handled implicitly by the store
implementation based on the caller's context.

## SDK convenience functions

The `mlflow.genai` namespace provides convenience functions that
combine store operations, matching the top-level public SDK pattern
established by `mlflow.genai.register_mcp_server()` in RFC-0004.

```python
from dataclasses import dataclass

import mlflow


def register_skill(
    *,
    name: str | None = None,
    organization: str = "",
    source: str | None = None,
    subpath: str | None = None,
    status: str = "draft",
) -> SkillVersion:
    """Register a skill version. The server assigns the next
    monotonic integer version. Auto-creates the parent Skill if
    it does not exist (with null description) and otherwise reuses
    the existing parent. To set parent-level metadata, use
    create_skill() before registering versions or update_skill()
    afterward. This matches the MCP Server Registry behavior
    (register_mcp_server). If name is omitted, the name is
    extracted from the skill's SKILL.md entry point (server-side
    for remote sources, client-side for local paths). The server
    infers source_type from the source URL (.git suffix or git://
    = git, oci:// = oci, .zip = zip). If source is a local path
    (no :// scheme), the client uploads the files through existing
    MLflow artifact APIs."""


def create_skill(
    *,
    name: str,
    organization: str = "",
    description: str | None = None,
) -> Skill: ...


def get_skill(*, name: str, organization: str = "") -> Skill: ...


def search_skills(
    *,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[Skill]: ...


def update_skill(
    *,
    name: str,
    organization: str = "",
    description: str | None = NOT_SET,
) -> Skill: ...


def delete_skill(*, name: str, organization: str = "") -> None: ...


def create_skill_version(
    *,
    name: str,
    organization: str = "",
    source: str | None = None,
    subpath: str | None = None,
) -> SkillVersion: ...


def get_skill_version(*, name: str, version: int, organization: str = "") -> SkillVersion: ...


def get_skill_version_by_alias(*, name: str, alias: str, organization: str = "") -> SkillVersion: ...


def get_latest_skill_version(*, name: str, organization: str = "") -> SkillVersion: ...


def search_skill_versions(
    *,
    name: str,
    organization: str = "",
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[SkillVersion]: ...


def update_skill_version(
    *,
    name: str,
    version: int,
    organization: str = "",
    status: str | None = NOT_SET,
) -> SkillVersion: ...


def delete_skill_version(*, name: str, version: int, organization: str = "") -> None: ...


def create_agent_plugin(
    *,
    name: str,
    organization: str = "",
    description: str | None = None,
) -> AgentPlugin: ...


def create_agent_plugin_version(
    *,
    name: str,
    organization: str = "",
    skills: list[str] | None = None,
    source: str | None = None,
    subpath: str | None = None,
) -> AgentPluginVersion: ...


def get_agent_plugin(*, name: str, organization: str = "") -> AgentPlugin: ...


def search_agent_plugins(
    *,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[AgentPlugin]: ...


def update_agent_plugin(
    *,
    name: str,
    organization: str = "",
    description: str | None = NOT_SET,
) -> AgentPlugin: ...


def delete_agent_plugin(*, name: str, organization: str = "") -> None: ...


def get_agent_plugin_version(
    *, name: str, version: int, organization: str = "",
) -> AgentPluginVersion: ...


def get_agent_plugin_version_by_alias(
    *, name: str, alias: str, organization: str = "",
) -> AgentPluginVersion: ...


def get_latest_agent_plugin_version(*, name: str, organization: str = "") -> AgentPluginVersion: ...


def search_agent_plugin_versions(
    *,
    name: str,
    organization: str = "",
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[AgentPluginVersion]: ...


def update_agent_plugin_version(
    *,
    name: str,
    version: int,
    organization: str = "",
    status: str | None = NOT_SET,
) -> AgentPluginVersion: ...


def delete_agent_plugin_version(*, name: str, version: int, organization: str = "") -> None: ...


def set_skill_tag(*, name: str, key: str, value: str, organization: str = "") -> None: ...

def delete_skill_tag(*, name: str, key: str, organization: str = "") -> None: ...

def set_skill_version_tag(*, name: str, version: int, key: str, value: str, organization: str = "") -> None: ...

def delete_skill_version_tag(*, name: str, version: int, key: str, organization: str = "") -> None: ...

def set_skill_alias(*, name: str, alias: str, version: int, organization: str = "") -> None: ...

def delete_skill_alias(*, name: str, alias: str, organization: str = "") -> None: ...

def set_agent_plugin_tag(*, name: str, key: str, value: str, organization: str = "") -> None: ...

def delete_agent_plugin_tag(*, name: str, key: str, organization: str = "") -> None: ...

def set_agent_plugin_version_tag(*, name: str, version: int, key: str, value: str, organization: str = "") -> None: ...

def delete_agent_plugin_version_tag(*, name: str, version: int, key: str, organization: str = "") -> None: ...

def set_agent_plugin_alias(*, name: str, alias: str, version: int, organization: str = "") -> None: ...

def delete_agent_plugin_alias(*, name: str, alias: str, organization: str = "") -> None: ...


@dataclass(frozen=True)
class PluginImportWarning:
    category: str
    path: str
    message: str


@dataclass(frozen=True)
class IntrospectedSkill:
    name: str
    path: str


@dataclass
class PluginIntrospectionResult:
    plugin_name: str | None
    skills: list[IntrospectedSkill]
    warnings: list[PluginImportWarning]


@dataclass
class PluginImportResult:
    plugin_version: AgentPluginVersion
    skill_versions: list[SkillVersion]
    warnings: list[PluginImportWarning]


def introspect_plugin(
    *,
    source: str,
    subpath: str | None = None,
) -> PluginIntrospectionResult:
    """Inspect a local or remote plugin without modifying the registry."""


def import_plugin(
    *,
    source: str,
    plugin_name: str | None = None,
    organization: str = "",
    subpath: str | None = None,
) -> PluginImportResult:
    """Import a plugin as a monolithic agent plugin.

    Discovers skill directories (subdirectories containing a SKILL.md
    entry point), registers them, preserves the plugin source on the
    agent plugin version, and returns warnings for non-skill content that is
    included in the agent plugin but does not receive individual registry
    entries. If plugin_name is omitted, it is inferred from the source
    directory name. The server infers source_type from the URL scheme.
    """


def pull(
    *,
    name: str | None = None,
    organization: str = "",
    entity_type: str = "skill",
    version: int | None = None,
    alias: str | None = None,
    destination: str = ".",
) -> str:
    """Pull skill or agent plugin content from registered sources to a
    local directory. Set entity_type to 'skill' or 'agent_plugin'."""


# Example usage:
version = mlflow.genai.register_skill(name="code-review", source="https://github.com/acme/skills.git@v1.0.0")
# version.name and version.organization identify the skill for subsequent operations
skills = mlflow.genai.search_skills(filter_string="status = 'active'")
```

For SDK update methods, `NOT_SET` means "leave unchanged" while `None`
means "clear this nullable field". This mirrors the store-layer update
contract so callers can distinguish partial updates from explicit
nulling.

`pull` is implemented in the SDK/CLI layer, not the store mixin. The
client calls `get_skill_version` (or resolves an alias) to obtain the
source pointer, then fetches content locally using source-type-specific
logic (git clone, OCI pull, ZIP download, or MLflow artifact download).
This keeps the store as a pure data-access layer.

## REST API

The REST API is implemented as a FastAPI router using RESTful nested
resource paths. It is exposed under both `/api/3.0/mlflow/skills` and
`/ajax-api/3.0/mlflow/skills`, plus the corresponding static-prefix
variants, following the MCP Server Registry (RFC-0004) pattern.

**Field inference.** The server infers optional fields from source
content when possible, so the simplest call requires only what cannot
be derived. When `name` is omitted from a registration request, the
server fetches the source and extracts the name from the skill's
SKILL.md entry point. If the server cannot access the source (e.g.,
private repositories requiring client-side credentials), registration
fails with an error indicating that `name` must be provided explicitly.
The server always infers `source_type` from the
`source` value: `.git` before the `@ref` portion or `git://` scheme
= git, `oci://` = oci, `.zip` = zip, and null `source` = mlflow
(content stored in MLflow artifact storage). The one exception is
embedded skills created during agent plugin import, where the
importer sets `source_type`, `source`, and `subpath` all to null
because the content lives inside the agent plugin artifact rather
than in standalone storage. `source_type` is not a user-facing
parameter. This keeps SDKs as thin REST wrappers, avoids
reimplementing inference in every language, and prepares for future
server-side content inspection (e.g., signature verification).

There is no skill-registry content-upload endpoint. When `source` is
a local path, the client creates a version record with null source to
obtain the server-assigned version number, then uploads through
existing MLflow artifact APIs to the controlled path.

### Skill endpoints

All paths relative to the logical skills router prefix.

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Create a skill |
| `GET` | `/` | Search skills |
| `POST` | `/register` | Register a skill version (name optional; server infers from source when omitted, auto-creates parent) |
| `GET` | `/{organization}/{name}` | Get skill by organization and name |
| `PATCH` | `/{organization}/{name}` | Update skill fields |
| `DELETE` | `/{organization}/{name}` | Hard-delete skill (cascades, subject to references) |
| `POST` | `/{organization}/{name}/versions` | Create a skill version |
| `GET` | `/{organization}/{name}/versions` | Search versions |
| `GET` | `/{organization}/{name}/versions/{version}` | Get a specific version |
| `PATCH` | `/{organization}/{name}/versions/{version}` | Update version |
| `DELETE` | `/{organization}/{name}/versions/{version}` | Soft-delete a version (`status='deleted'`) |
| `POST` | `/{organization}/{name}/tags` | Set a skill-level tag |
| `DELETE` | `/{organization}/{name}/tags/{key}` | Delete a skill-level tag |
| `POST` | `/{organization}/{name}/versions/{version}/tags` | Set a version-level tag |
| `DELETE` | `/{organization}/{name}/versions/{version}/tags/{key}` | Delete a version tag |
| `POST` | `/{organization}/{name}/aliases` | Set an alias |
| `GET` | `/{organization}/{name}/aliases/{alias}` | Resolve alias to `SkillVersion` |
| `DELETE` | `/{organization}/{name}/aliases/{alias}` | Delete an alias |

When `organization` is empty, the path segment uses the literal
value `_` as a placeholder (e.g., `/_/code-review/versions/1`).

Similarly, agent plugin endpoints are exposed under both
`/api/3.0/mlflow/agent-plugins` and
`/ajax-api/3.0/mlflow/agent-plugins`.

### Agent plugin endpoints

All paths relative to the logical agent-plugins router prefix.

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Create an agent plugin |
| `GET` | `/` | Search agent plugins |
| `GET` | `/{organization}/{name}` | Get agent plugin by organization and name |
| `PATCH` | `/{organization}/{name}` | Update agent plugin fields |
| `DELETE` | `/{organization}/{name}` | Hard-delete agent plugin (cascades versions and memberships) |
| `POST` | `/{organization}/{name}/versions` | Create an agent plugin version with members |
| `GET` | `/{organization}/{name}/versions` | Search agent plugin versions |
| `GET` | `/{organization}/{name}/versions/{version}` | Get a specific agent plugin version |
| `PATCH` | `/{organization}/{name}/versions/{version}` | Update agent plugin version status |
| `DELETE` | `/{organization}/{name}/versions/{version}` | Soft-delete an agent plugin version (`status='deleted'`) |
| `POST` | `/{organization}/{name}/tags` | Set an agent plugin-level tag |
| `DELETE` | `/{organization}/{name}/tags/{key}` | Delete an agent plugin-level tag |
| `POST` | `/{organization}/{name}/versions/{version}/tags` | Set an agent plugin version tag |
| `DELETE` | `/{organization}/{name}/versions/{version}/tags/{key}` | Delete an agent plugin version tag |
| `POST` | `/{organization}/{name}/aliases` | Set an agent plugin alias |
| `GET` | `/{organization}/{name}/aliases/{alias}` | Resolve agent plugin alias to version |
| `DELETE` | `/{organization}/{name}/aliases/{alias}` | Delete an agent plugin alias |

Same `_` placeholder convention for empty organization as skill
endpoints.

### Pagination and filtering

Search endpoints use page-token-based pagination and `filter_string`
expressions following existing MLflow conventions.

**Skills:** `name LIKE '%review%'`, `status = 'active'`,
`tags.team = 'platform'`

**Agent plugins:** Same as skills, plus `member_name = 'code-review'` to
find agent plugins that include a given skill (reverse membership lookup)

**Versions (all entity types):** `status = 'active'`,
`source_type = 'git'`, `tags.approved = 'true'`

### Request and response models

Request models contain only the mutable fields; resource identifiers
come from path parameters (`organization` and `name`), with one
exception: `POST /register` accepts `name` and `organization` in
the request body (`name` is optional, inferred from source content
when omitted; `organization` defaults to `""`) and returns the
created skill identity. This parallels RFC-0004's top-level
`register_mcp_server()` pattern:

```python
from pydantic import BaseModel, Field


class CreateSkillRequest(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None


class UpdateSkillRequest(BaseModel):
    description: str | None = None


class CreateSkillVersionRequest(BaseModel):
    name: str | None = None  # optional for POST /register (inferred from source when omitted); ignored for POST /{organization}/{name}/versions (name from parent)
    organization: str = ""  # for POST /register only; ignored for versioned paths
    source: str | None = None
    subpath: str | None = None
    status: str = "draft"


class UpdateSkillVersionRequest(BaseModel):
    status: str | None = None


class CreateAgentPluginRequest(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None


class UpdateAgentPluginRequest(BaseModel):
    description: str | None = None


class CreateAgentPluginVersionRequest(BaseModel):
    skills: list[str] | None = None
    source: str | None = None
    subpath: str | None = None


class UpdateAgentPluginVersionRequest(BaseModel):
    status: str | None = None


class AliasResponse(BaseModel):
    alias: str
    version: int


class SetAliasRequest(BaseModel):
    alias: str
    version: int


class SetTagRequest(BaseModel):
    key: str
    value: str


class SkillVersionResponse(BaseModel):
    name: str
    version: int
    organization: str = ""
    source_type: str | None = None
    source: str | None = None
    subpath: str | None = None
    status: str = "draft"
    aliases: list[str] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class SkillResponse(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None
    status: str | None = None
    latest_version: int | None = None
    aliases: list[AliasResponse] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class AgentPluginVersionResponse(BaseModel):
    name: str
    version: int
    organization: str = ""
    source_type: str | None = None
    source: str | None = None
    subpath: str | None = None
    status: str = "draft"
    skills: list[str] = Field(default_factory=list)
    aliases: list[str] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class AgentPluginResponse(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None
    status: str | None = None
    latest_version: int | None = None
    aliases: list[AliasResponse] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

`Skill.aliases` is modeled as a `dict[str, int]` in the entity layer
for convenience, while REST responses expose aliases as
`list[AliasResponse]` to keep the payload shape explicit and
consistent with the MCP Server Registry (RFC-0004).

**Composite primary keys.** Skills and agent plugins use a
`(workspace, organization, name)` composite primary key. The
`organization` field scopes ownership (e.g., a team or publisher
name) and defaults to `""` (empty string) when not specified.
This allows the same skill name to exist under different
organizations without collision (e.g., `acme/code-review` and
`beta-corp/code-review` are distinct skills). When `organization`
is empty, the skill is identified by name alone within its
workspace, consistent with the MCP Server Registry (RFC-0004).
The Prompt Registry is considering adding `organization` for
similar reasons; this design aligns with that direction.

**Name and organization validation.** The server rejects
`organization` and `name` values that would create ambiguity in
URIs, REST paths, or artifact storage paths:

- `organization` cannot be `_` (reserved as the empty-organization
  placeholder in REST paths and artifact paths).
- `name` cannot be purely numeric (e.g., `123`), because URI
  disambiguation relies on integer parsing to distinguish
  `name/version` from `organization/name`.
- Neither field may contain `/`, `@`, `#`, or `?` (URI-significant
  characters).

## Python SDK and CLI

The `mlflow.genai` module exposes the public registry functions,
delegating to `MlflowClient`, plus client-side import and pull
operations that compose those registry functions. Skill-specific
entity and request types are also re-exported from `mlflow.genai`.
The `mlflow skills` CLI command group provides the same operations
from the command line. CLI commands accept `--name` and optional
`--organization` flags for entity identification, and
`--skill-uri` as a convenience that parses name, organization,
and version from a URI:

| CLI subcommand | SDK function | Description |
|---|---|---|
| `mlflow skills register` | `register_skill()` | Register a skill version (auto-creates parent) |
| `mlflow skills get` | `get_skill()` | Get skill metadata |
| `mlflow skills search` | `search_skills()` | Search skills |
| `mlflow skills get-version` | `get_skill_version()` | Get a specific version |
| `mlflow skills update-version` | `update_skill_version()` | Update version status |
| `mlflow skills set-alias` | `set_skill_alias()` | Set a version alias |
| `mlflow skills set-tag` | `set_skill_tag()` | Set a tag |
| `mlflow skills pull` | `pull()` | Pull content to local filesystem |
| `mlflow agent-plugins create` | `create_agent_plugin()` | Create an agent plugin |
| `mlflow agent-plugins create-version` | `create_agent_plugin_version()` | Create an agent plugin version with members |
| `mlflow agent-plugins get` | `get_agent_plugin()` | Get agent plugin metadata |
| `mlflow agent-plugins search` | `search_agent_plugins()` | Search agent plugins |
| `mlflow agent-plugins search-versions` | `search_agent_plugin_versions()` | Search agent plugin versions |
| `mlflow agent-plugins set-alias` | `set_agent_plugin_alias()` | Set an agent plugin alias |
| `mlflow agent-plugins set-tag` | `set_agent_plugin_tag()` | Set an agent plugin-level tag |
| `mlflow agent-plugins set-version-tag` | `set_agent_plugin_version_tag()` | Set an agent plugin version tag |
| `mlflow agent-plugins update-version` | `update_agent_plugin_version()` | Update agent plugin version status |
| `mlflow agent-plugins introspect` | `introspect_plugin()` | Preview a local or remote plugin without registry writes |
| `mlflow agent-plugins import` | `import_plugin()` | Import a plugin as a monolithic agent plugin |

**Relationship to existing `mlflow skills` subcommands.** MLflow already
has `mlflow skills list` and `mlflow skills view` subcommands
(`mlflow/cli/skills.py`) that inspect the bundled Assistant skills
shipping with the MLflow installation (under `mlflow/assistant/skills/`).
The registry's subcommands use different names (e.g., `register`,
`search`, `pull`), so there is no direct conflict.

## Plugin import

Plugin import is implemented in the SDK and CLI layer. There is no
dedicated REST import endpoint: the client fetches and inspects the
source locally, then calls the existing skill and agent plugin creation APIs.
The registry server does not fetch user-supplied plugin URLs.

### Read-only preview

`introspect_plugin()` and `mlflow agent-plugins introspect` run the same plugin
discovery used by import but do not create or modify registry records.
They accept either a local path or a remote Git, OCI, ZIP, or MLflow
artifact source and return the discovered skill names and paths,
available plugin name metadata, and warnings for unregistered
non-skill content. The server infers `source_type` from the source
value using the same rules as registration.

### Supported input format

Import expects a standard layout: a directory tree where each skill is
a subdirectory containing a SKILL.md entry point. This layout is used
by Claude Code and other harnesses. The importer:

1. Fetches the Git, OCI, ZIP, or MLflow artifact source using the same
   source-type-aware logic as `pull`.
2. Applies `subpath`, when provided, to select the plugin root.
3. Reads `.claude-plugin/plugin.json` when present to obtain supported
   plugin metadata such as name. An explicit `plugin_name` argument
   takes precedence. Version is server-assigned at import time.
4. Discovers skill directories that contain a SKILL.md entry point.
   The SKILL.md name is used when present; otherwise the directory
   name is used.
5. Detects non-skill content for warning purposes only.

The resulting agent plugin name must be available after explicit
arguments and plugin metadata are considered. The version is
server-assigned. The server infers `source_type` from the source
value using the same rules as registration.

### Registration behavior

For each discovered skill, the importer creates a `SkillVersion` whose
version is server-assigned and whose `source_type`, `source`,
and `subpath` are null. The importer records the skill's plugin-relative
directory as the `#subpath` fragment in the member URI (e.g.,
`skills:/embedded-review/1#skills/embedded-review`).

After registering the embedded skills, the importer creates one
monolithic `AgentPluginVersion` with the original `source_type`,
`source`, and `subpath`, plus member references for all discovered
skills. This preserves a pullable link to the complete original plugin
while keeping registry entries limited to skills.

The import fails if no skills are discovered.

#### Re-import behavior

When the target agent plugin already has at least one version, import
matches discovered skills to existing members using the `#subpath`
fragment from the most recent agent plugin version's member list:

1. **Matching subpath:** The discovered skill's plugin-relative
   directory matches a `#subpath` in the previous member list. Import
   creates a new version of that existing skill (identified by the
   member reference in the previous agent plugin version, not by name
   lookup).
2. **New subpath:** The directory does not match any previous member.
   Import creates a new skill. The embedded skill's version number
   matches the agent plugin version, consistent with the rule that
   embedded skills use the agent plugin version.
3. **Removed subpath:** A previous member's subpath is not found in the
   new source. The member is omitted from the new agent plugin version.
   The skill and its existing versions remain in the registry.

After processing all discovered skills, import creates a new
`AgentPluginVersion` with updated member references. Previous agent
plugin versions are immutable and unchanged.

Since versions are server-assigned, import does not conflict with
existing version numbers.

### Warnings and result

Subagents, hooks, MCP configurations, and unrecognized content remain
in the plugin artifact but are not registered. Each discovered skipped
category produces a `PluginImportWarning` containing its category,
path, and an explanation that this RFC does not create registry entries
for non-skill content (though the content remains in the agent plugin). The CLI
prints these warnings after registration. The SDK returns them together
with the created agent plugin and skill versions in `PluginImportResult`.

Import translates an existing plugin into MLflow's registry
representation, creating registry entries for discovered skills while
preserving the complete plugin source. It does not install the plugin,
translate an MLflow agent plugin into a downstream agent plugin format.

## Pull semantics details

**Source availability.** The registry stores source pointers but does
not cache or proxy content. If a source is unreachable or the content
has been deleted, pull fails with an error that surfaces the
underlying failure from the source system (e.g., Git clone failure,
OCI pull 404, HTTP download error, MLflow artifact download error).
Source availability is the publisher's responsibility. For assembled
agent plugin pulls, if one member's source is unavailable, the entire pull
fails rather than producing a partial result.

**Source authentication.** The registry server stores source pointers
but does not validate source accessibility at registration time and is
not involved in content transfer at pull time. Authentication to
external sources is handled entirely by the client environment:

| Source type | Authentication mechanism |
|---|---|
| `git` | Standard Git credential resolution: SSH keys (`~/.ssh/`), Git credential helpers (`git-credential-manager`, `git-credential-store`), `.netrc`, and `GIT_SSH_COMMAND`. Private repos work if the caller's Git is configured to access them. |
| `oci` | OCI registry credential resolution: Docker config (`~/.docker/config.json`), registry-specific credential helpers, and container runtime auth. Private registries work if the caller has a valid login session. |
| `zip` | No authentication support. ZIP sources must be publicly accessible URLs. For private content, use `git` or `oci` source types instead. |
| `mlflow` | MLflow artifact storage authentication, using the same credentials as other MLflow API calls. |

The registry does not store, proxy, or manage source credentials.
Pull failures due to authentication errors are surfaced to the caller
with the underlying error from the source system.

## SDK and CLI code examples

### Register skills from an OCI artifact with subpath

```python
import mlflow

mlflow.genai.register_skill(
    name="code-review",
    source="oci://ghcr.io/acme/agent-plugin:v1.0.0",
    subpath="skills/code-review",
)

mlflow.genai.register_skill(
    name="test-coverage",
    source="oci://ghcr.io/acme/agent-plugin:v1.0.0",
    subpath="skills/test-coverage",
)

# Assembled agent plugin: each member has its own source
plugin = mlflow.genai.create_agent_plugin(name="pr-workflow")
plugin_version = mlflow.genai.create_agent_plugin_version(
    name="pr-workflow",
    skills=[
        "skills:/code-review/1",
        "skills:/test-coverage/1",
    ],
)
```

Monolithic agent plugins are created through `import_plugin()`, which
handles embedded skill creation internally. See the
[Plugin import](#plugin-import) section for details.

### Discover and consume skills

```python
# Find a skill by name
skills = mlflow.genai.search_skills(filter_string="name = 'code-review'")
skill = skills[0]

# Search for active versions of that skill
versions = mlflow.genai.search_skill_versions(
    name=skill.name,
    organization=skill.organization,
    filter_string="status = 'active'",
)

# Search for active agent plugins
plugins = mlflow.genai.search_agent_plugins(
    filter_string="status = 'active'",
)

# Get a specific version
version = mlflow.genai.get_skill_version(
    name=skill.name,
    organization=skill.organization,
    version=1,
)
# version.source_type == "git"
# version.source == "https://github.com/acme/agent-skills.git@v1.0.0"
# version.subpath == "code-review"

# Resolve by alias
version = mlflow.genai.get_skill_version_by_alias(
    name=skill.name,
    organization=skill.organization,
    alias="production",
)

# Get an agent plugin version and its pinned members
plugin = mlflow.genai.search_agent_plugins(
    filter_string="name = 'pr-workflow'",
)[0]
plugin_version = mlflow.genai.get_agent_plugin_version(
    name=plugin.name,
    organization=plugin.organization,
    version=1,
)
# plugin_version.skills == ["skills:/code-review/1", ...]

# Resolve an agent plugin alias
plugin_version = mlflow.genai.get_agent_plugin_version_by_alias(
    name=plugin.name,
    organization=plugin.organization,
    alias="production",
)
```
