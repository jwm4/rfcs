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
| `search_text` | `Text` | derived discovery projection of name and description |
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
| `source_type` | `String(20)` | server-set; `git`, `oci`, `zip`, `mlflow`, `embedded` |
| `source` | `String(2048)` | nullable pointer to skill content |
| `ref` | `String(2048)` | nullable; git branch, tag, or commit |
| `subpath` | `String(2048)` | nullable; path within the artifact |
| `status` | `String(20)` | default `'active'` |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, organization, name)` references `skills`, CASCADE
delete. This supports administrative hard deletion of the parent
`Skill`; normal version deletion is a status transition to `deleted`
and does not physically remove the version row.

**Version ordering**: versions are monotonic integers assigned by
the server. The next version number is one greater than the maximum
`version` across all existing rows for that skill, including soft-`deleted`
ones, and is assigned atomically so concurrent registrations cannot
collide or reuse a number. Because soft-deleted versions keep their rows,
a number is never reused even after the version it identified is deleted.
Ordering is a simple integer comparison.

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
| `version` | `Integer` | target `skill_versions.version` under the same parent; see alias integrity below |

**Alias integrity.** An alias `version` targets a version row under the
same `(workspace, organization, name)` parent. Integrity is enforced by
the application rather than by a separate database FK: `set_skill_alias`
(and `set_agent_plugin_alias`) rejects a missing or `deleted` target, and
because a soft-deleted row persists a plain FK could not enforce the
not-`deleted` rule. Soft-deleting a version removes any aliases that point
to it. Agent plugin aliases (`agent_plugin_aliases`) follow the same
pattern against `agent_plugin_versions.version`, and additionally reject a
withdrawn target (a plugin version that contains a `deleted` member), so
an alias cannot bypass the kill switch (see Deletion semantics).

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
| `version` | `String(256)` | PK; equal to canonical `plugin_json["version"]` |
| `version_major` | `Integer` | extracted SemVer major component |
| `version_minor` | `Integer` | extracted SemVer minor component |
| `version_patch` | `Integer` | extracted SemVer patch component |
| `plugin_json` | `JSON` | canonical Agent Plugins manifest; immutable after creation, with the version field canonicalized (normalized) on ingest |
| `search_text` | `Text` | derived discovery projection of name, mutable parent description, organization, and this version's manifest description, keywords, and author name; parent search matches the latest-resolved version's value |
| `source_type` | `String(20)` | server-set; `git`, `oci`, `zip`, `mlflow`, `assembled` |
| `source` | `String(2048)` | optional pointer to agent plugin |
| `ref` | `String(2048)` | nullable; git branch, tag, or commit |
| `subpath` | `String(2048)` | nullable; path within the artifact |
| `status` | `String(20)` | default `'active'` |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, organization, name)` references `agent_plugins`,
CASCADE delete.

**Version ordering.** All versions are valid SemVer (semverish inputs are
normalized on creation). Latest resolution first selects active candidates, or
non-deleted non-active candidates when none are active. Semantic precedence
determines the result. The numeric components narrow candidates efficiently;
full prerelease precedence is applied in application code. Creation time breaks
equal semantic precedence, including versions that differ only in build
metadata.

**Index:** `ix_agent_plugin_versions_latest_lookup` on `(workspace,
organization, name, status, version_major, version_minor, version_patch,
creation_timestamp)` supports both resolution paths.

### `agent_plugin_version_members`

| Column | Type | Notes |
|--------|------|-------|
| `plugin_workspace` | `String(63)` | PK, FK to `agent_plugin_versions` |
| `plugin_organization` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `plugin_name` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `plugin_version` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `member_organization` | `String(256)` | PK, FK to `skill_versions` |
| `member_name` | `String(256)` | PK, FK to `skill_versions` |
| `member_version` | `Integer` | PK, FK to `skill_versions` |

FK: `(plugin_workspace, plugin_organization, plugin_name,
plugin_version)` references `agent_plugin_versions`, CASCADE
delete. A FK to `skill_versions` via `(plugin_workspace,
member_organization, member_name, member_version)` enforces
referential integrity with RESTRICT delete. The RESTRICT applies to
physical (hard) deletion of a `skill_versions` row; it does not block a
member's soft delete (status transition to `deleted`), which is allowed
and handled as a derived withdrawal of the containing plugin versions
across resolution, discovery, and pull (see Deletion semantics). Skills
and agent plugins share the same workspace;
`plugin_workspace` is reused for the skill FK.

**Member-name uniqueness.** A `UNIQUE` constraint on `(plugin_workspace,
plugin_organization, plugin_name, plugin_version, member_name)` enforces that
member names are distinct within an agent plugin version. The primary key alone
does not guarantee this, because it also includes `member_organization` and
`member_version`; those two columns are retained for the `skill_versions` FK and
as stored data, not to distinguish rows for uniqueness. For an assembled plugin
the pull layout writes each member to `skills/<member-name>/`, keyed on the name
alone, so a name collision would be ambiguous on disk; for a monolithic plugin
the embedded skills are discovered from distinct `skills/*/SKILL.md`
directories, which are already name-unique. Because the name is the on-disk key,
a plugin version cannot contain two skills of the same name from different
organizations. Create requests with duplicate member names are rejected before
insert.

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
| `version` | `String(256)` | PK, FK to `agent_plugin_versions` |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `agent_plugin_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK to `agent_plugins` |
| `organization` | `String(256)` | PK, FK to `agent_plugins` |
| `name` | `String(256)` | PK, FK to `agent_plugins` |
| `alias` | `String(256)` | PK |
| `version` | `String(256)` | target agent plugin manifest version |

**Canonical manifest storage.** `plugin_json` uses SQLAlchemy's `JSON` type,
following RFC-0004's `server_json` precedent. It maps to native JSON where the
database supports it and to the platform's text-backed JSON representation for
SQLite and SQL Server. The full payload is preserved, while identity, ordering,
and search projections are materialized separately.

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
  unpublished or deprecated before they can be deleted. A soft-deleted
  skill version also withdraws every agent plugin version that contains
  it, whether as a pinned member (assembled) or an embedded member
  (monolithic): such a plugin version is treated as `deleted` for
  resolution, discovery, and pull, so soft delete acts as a kill switch
  across both plugin kinds. The withdrawal is derived from member status,
  so the containing plugin version's membership rows and stored `status`
  are unchanged; an operator can publish a replacement plugin version
  referencing a fixed member. A `deprecated` member, by contrast, does not
  trigger withdrawal and still resolves and pulls.
- The `deleted` status is terminal. Internal audit or provenance paths
  may retain enough metadata to explain historical agent plugin
  snapshots, but deleted versions are not surfaced to consumers.

## Entity dataclasses

### Skill entity

```python
from dataclasses import dataclass, field
from enum import StrEnum
from typing import Any


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
| `status` | `SkillStatus \| None` | Read-only; derived from the parent-resolved version: highest active version number if present, otherwise highest non-deleted non-active version number. `None` when the skill has no non-`deleted` version |
| `aliases` | `dict[str, int]` | Stable version pointers (e.g., `{"production": 2}`); read-only, populated from `skill_aliases` table |
| `latest_version` | `int \| None` | Read-only; highest version number among `active` versions if one exists, otherwise highest non-`deleted` non-`active` version. `None` when the skill has no non-`deleted` version |
| `workspace` | `str` | Visibility boundary |

### SkillVersion entity

```python
class SkillSourceType(StrEnum):
    GIT = "git"
    OCI = "oci"
    ZIP = "zip"
    MLFLOW = "mlflow"
    EMBEDDED = "embedded"  # skill member stored inside a monolithic agent plugin
    ASSEMBLED = "assembled"  # agent plugin version composed from member references


@dataclass
class GitSource:
    url: str
    ref: str | None = None
    subpath: str | None = None


@dataclass
class OCISource:
    image: str
    subpath: str | None = None


@dataclass
class ZipSource:
    url: str
    subpath: str | None = None


@dataclass
class SkillVersion:
    name: str
    version: int
    organization: str = ""
    source: GitSource | OCISource | ZipSource | None = None
    source_type: SkillSourceType | None = None
    status: SkillStatus = SkillStatus.ACTIVE

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
| `source` | `GitSource \| OCISource \| ZipSource \| None` | Typed source descriptor for external content (git, OCI, zip). `None` for content the registry resolves by convention rather than an external pointer: MLflow artifact storage (`source_type="mlflow"`, path derived from identity) and skills embedded in a monolithic agent plugin (`source_type="embedded"`). The persisted `source_type` distinguishes these two null-source cases. The REST API represents this as flat `source_type`, `source`, `ref`, `subpath` fields; the SDK wraps and unwraps the typed classes |
| `source_type` | `SkillSourceType \| None` | Server-set discriminator (`git`, `oci`, `zip`, `mlflow`, `embedded`), populated on responses. Clients never supply it on create; the server infers it (see the field-inference rules below). Together with `source` it determines how content is stored and how `pull` routes, and it is what distinguishes the two null-`source` cases (`mlflow` vs `embedded`) |
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

**Typed source classes.** Each source type has a dedicated class with
only the fields relevant to that type:

| Class | Fields | Description |
|---|---|---|
| `GitSource` | `url`, `ref`, `subpath` | Git repository. `url` is the clone URL. `ref` is the branch, tag, or commit (optional; defaults to the repository's default branch). `subpath` is the path within the repo (optional). |
| `OCISource` | `image`, `subpath` | OCI image. `image` is the image reference, supplied with the `oci://` scheme (e.g., `oci://ghcr.io/acme/plugin:v1`) so the server can infer `source_type`. The scheme is an inference hint only: the persisted `source` is the bare reference (`ghcr.io/acme/plugin:v1`). `subpath` is the path within the image (optional). |
| `ZipSource` | `url`, `subpath` | ZIP archive. `url` is the archive URL. `subpath` is the path within the archive (optional). |

MLflow artifact storage does not use a source class. When `source`
is `None`, the artifact path is derived from the skill's identity
and version.

The REST API represents these as flat fields (`source_type`,
`source`, `ref`, `subpath`); the SDK converts between typed classes
and flat fields. The server determines `source_type` from the source
value for external pointers and from the creation flow for content it
stores or resolves by convention (see the field-inference rules below),
and returns it in responses. The SDK surfaces `source_type` as a field on
the version and uses it to reconstruct the typed class for external
sources; for null-`source` content it is the only discriminator between
the `mlflow` and `embedded` cases.

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
store is `skills/@<organization>/<name>/<version>/` when the skill has
an organization and `skills/<name>/<version>/` when it does not; the
`@<organization>` segment is omitted entirely, along with its slash, when
the organization is empty. The `source` field is null for
MLflow-stored content; the system knows where to find it by
convention. Pull downloads the directory tree from the artifact
store. The MLflow UI can browse individual files within a stored
skill version when artifact proxying is enabled.

The same artifact path convention applies to agent plugins stored
with `source_type="mlflow"`: `agent-plugins/@<organization>/<name>/<version>/`,
with the `@<organization>` segment omitted when there is no organization.

**Client-side upload flow.** When `source` is a local path (detected
by the absence of a `://` scheme), the SDK uploads the content to
MLflow artifact storage rather than treating it as a remote pointer:

1. The client validates the local directory and preflights that the
   path can be registered.
2. The client creates the `SkillVersion` with `source` set to null and
   `status="draft"`, regardless of the caller's requested final status.
   The server assigns the version number and infers
   `source_type="mlflow"` from the null source. Because a draft is not
   preferred by latest resolution over an existing `active` or
   `deprecated` version, an in-flight upload never displaces the currently
   recommended version. (When the entity has no other version, latest
   resolution may surface the in-flight draft; it carries `status="draft"`
   and is discarded if the upload fails, per step 4.)
3. Using the returned version number, the client uploads each file
   through MLflow's existing artifact APIs to the controlled artifact
   prefix (`skills/@<organization>/<name>/<version>/`, with the
   `@<organization>` segment omitted when there is no organization).
4. On success the client transitions the version to the caller's
   requested final status (`active` by default, or `draft` when the
   caller explicitly registered a draft). On failure the client discards
   the version with the normal `draft` -> `deleted` transition and
   removes any partially uploaded files.

This keeps the published view atomic: a version becomes `active` only
after its content is fully uploaded, and a failed upload leaves a
discarded draft rather than a content-less `active` version. Creating as `draft` first means the flow needs no special
single-version hard delete; it reuses the existing `draft` -> `deleted`
transition. A backend without deletion support can retain the discarded
draft's unreferenced files until garbage collection.

**Version uniqueness.** The combination of
`(workspace, organization, name, version)` is unique. A skill
version represents a single logical version of a capability;
`source` describes where to find it but is not part of its
identity.

**Immutability contract.** `source` and `version` are immutable
after creation. To point to different content, register a new
version. Mutable fields (`status`, `tags`) can be updated
independently.

### AgentPlugin entity

An agent plugin is the stable governed identity for an open Agent Plugins
package. It follows the same top-level MLflow pattern as Skill while deriving
its canonical name and version metadata from immutable manifests.

```python
@dataclass
class AgentPlugin:
    name: str
    organization: str = ""
    description: str | None = None
    workspace: str | None = None
    status: SkillStatus | None = None  # read-only, derived from parent-resolved version
    tags: dict[str, str] = field(default_factory=dict)
    aliases: dict[str, str] = field(default_factory=dict)  # read-only; manifest version targets
    latest_version: str | None = None  # read-only, Agent Plugin resolution rule
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

`AgentPlugin.status` is read-only and comes from the same latest-resolved
version as `latest_version`. Both are `None` when the plugin has no
non-`deleted` version. Parent `description` is mutable MLflow presentation
metadata. When unset, the UI may fall back to the resolved
`plugin_json["description"]`; the API returns the parent value as stored.

### AgentPluginVersion entity

A versioned snapshot of the canonical package manifest, source authority,
lifecycle state, and optional registered-skill membership. In this RFC, all
members are skills.

Agent plugin members are referenced by URI string rather than a separate
data class. The URI format follows MLflow's `models:/name/version`
convention:

- `skills:/name` resolves to the skill's current latest version at creation
- `skills:/name/version` pins a specific version
- `skills:/name@alias` resolves through an alias

All three forms are resolved to a concrete `member_version` at create time and
frozen into the member row (see the member-list URI resolution note below).

```python
@dataclass
class AgentPluginVersion:
    name: str
    version: str
    organization: str = ""
    plugin_json: dict[str, Any] = field(default_factory=dict)
    source: GitSource | OCISource | ZipSource | None = None
    source_type: SkillSourceType | None = None

    status: SkillStatus = SkillStatus.ACTIVE
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
`(workspace, organization, name, version)` is unique. `name` and `version` are
extracted from the canonical payload, and an explicitly supplied path name must
match `plugin_json["name"]`.

**Canonical manifest.** `plugin_json` is validated against the fields MLflow
requires, using `extra="allow"` to accept and preserve unknown fields. The
`$schema` identifier is not gated on a specific version. A non-object
`extensions` field generates a nonfatal warning, remains preserved in the
stored payload, and receives no MLflow semantics. The payload is immutable
after creation.

**Version validation.** When `plugin_json["version"]` is present, MLflow
validates it as SemVer. Semverish inputs (e.g., `1.0`) are normalized to full
SemVer (e.g., `1.0.0`). Non-SemVer version strings are rejected. When the
manifest does not include a version, the user must supply one explicitly.
MLflow inserts the supplied version into the stored payload. In every stored
entity, `version == plugin_json["version"]`.

**Manifest synthesis.** For an assembled plugin, both registration and
low-level version creation may receive `plugin_json=None`. The user must
supply a `version` in this case. The server-side registry layer synthesizes
a minimal valid manifest using the parent name and supplied version before
storing the entity. Monolithic import always supplies
a complete manifest after validation or adapter translation. Persistence never
stores a null or incomplete canonical payload.

**Latest resolution.** Active candidates take precedence. Semantic precedence
selects latest. Creation time breaks ties (e.g., versions differing only in
build metadata). The same rule applies to non-deleted non-active candidates
when no active version exists.

**Agent plugin-level source and kind.** An agent plugin version is either
monolithic or assembled, never both. Kind is derived from the persisted
`source_type`: a version whose `source_type` is `git`, `oci`, `zip`, or
`mlflow` is **monolithic**; a version whose `source_type` is `assembled`
is **assembled**. All versions of a given agent plugin must be the same
kind; the server rejects a version whose derived kind differs from
existing versions of the same agent plugin.

- **Monolithic:** has its own package that contains the complete agent
  plugin, either an external typed source (`source_type` of `git`, `oci`,
  or `zip`) or a package tree uploaded to MLflow artifact storage
  (`source_type="mlflow"`). `pull` fetches the agent plugin as a unit.
  Member skill versions omit their own `source` (they are
  `source_type="embedded"`) because the agent plugin package is the
  authoritative source. Embedded members are referenced by name and are
  not individually addressable within the package.
- **Assembled:** has no plugin-level package; its content is defined
  entirely by individual member references (`source_type="assembled"`).
  Each skill member has its own source. `pull` fetches members
  individually. If a skill member has no source, `pull` fails rather than
  producing a partial local agent plugin.

The server sets `source_type` from the creation flow, not from a
user-supplied value: a package source (external pointer or MLflow upload)
yields a monolithic version, and a version created from member references
with no plugin-level package yields `source_type="assembled"`. A
source-less (`embedded`) member is valid only in a monolithic agent
plugin, whose package is authoritative for the member's content. In an
assembled agent plugin, every member must have its own source. The API
rejects a source-less member of an assembled agent plugin and a sourced
member of a monolithic one.

**Immutability contract.** `plugin_json`, the member list, and source fields of
an agent plugin version are immutable after creation. To change the canonical
manifest, members, or source pointer, register a new agent plugin version.
Mutable fields (`status`, `tags`) can be updated independently.

The low-level registry API validates the manifest but does not fetch a remote
artifact to validate its layout. Client-side import performs format-specific
filesystem discovery and containment validation before registration.

A member can appear in multiple agent plugins and multiple agent plugin versions.
Membership is at the version level, so an agent plugin version is a
reproducible record of "these specific skill versions work together." The
membership record is preserved even if a member is later withdrawn; a
soft-deleted member withdraws the containing plugin version from
resolution, discovery, and pull (see Deletion semantics), while a
`deprecated` member still resolves and pulls.

### Skill and Agent Plugin URI formats

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
| `skills:/@org/name` | Skill with organization | `skills:/@acme/code-review` |
| `skills:/@org/name/version` | Pin version with organization | `skills:/@acme/code-review/1` |
| `skills:/@org/name@alias` | Alias with organization | `skills:/@acme/code-review@production` |

**URI disambiguation.** An organization is always marked by a leading
`@` on the first segment (e.g., `skills:/@acme/...`), and names cannot
begin with `@`, so the organization is unambiguous whether or not it is
present. After the optional `@organization` segment, the segments are the
name followed by an optional version, and a trailing `@alias` selects an
alias instead of a version. For example, `skills:/code-review/1` is name
`code-review` at version `1`, and `skills:/@acme/code-review/1` is that
version within organization `acme`.

Agent plugins use a separate URI scheme because their versions are
SemVer strings rather than integers. Organization uses the same leading
`@` marker and is omitted when empty, so the parser identifies the
organization by the `@` marker rather than by inspecting the version:

| Pattern | Meaning | Example |
|---------|---------|---------|
| `agent-plugins:/name` | Agent plugin parent (no org) | `agent-plugins:/pr-workflow` |
| `agent-plugins:/@org/name` | Agent plugin parent (with org) | `agent-plugins:/@acme/pr-workflow` |
| `agent-plugins:/name/version` | Exact version (no org) | `agent-plugins:/pr-workflow/1.0.0-beta.11` |
| `agent-plugins:/@org/name/version` | Exact version (with org) | `agent-plugins:/@acme/pr-workflow/1.0.0` |
| `agent-plugins:/name@alias` | Resolve through an alias | `agent-plugins:/pr-workflow@production` |
| `agent-plugins:/@org/name@alias` | Resolve through an alias (with org) | `agent-plugins:/@acme/pr-workflow@production` |

Agent plugin versions in URIs must be nonempty, valid SemVer, fit within the
database field, and cannot contain `/`, `@`, `#`, or `?`.

In the CLI, commands that operate on an already-registered skill or agent
plugin accept its URI as an optional positional argument, with the equivalent
`--skill-uri` (skills) or `--plugin-uri` (agent plugins) flag also accepted.
For example, `mlflow skills get skills:/code-review` and
`mlflow skills get --skill-uri skills:/code-review` are equivalent.
Identity-establishing commands (`register`, and `agent-plugins create`) use
`--name` (with an optional `--organization`) rather than a URI, since the entity
does not exist yet.
In agent plugin member lists, URIs appear as plain strings in `list[str]`.
The server parses the URI into its constituent fields
(`member_organization`, `member_name`, `member_version`)
for storage and validation. Every reference is resolved to a concrete
`member_version` at create time and that integer is frozen into the member row:
a pinned `skills:/name/version` reference stores the given version, a name-only
`skills:/name` reference resolves to the skill's current latest version, and a
`skills:/name@alias` reference resolves to the version the alias points to at
that moment. Name-only resolution uses the standard latest-resolution rule, so
it may select a `draft` when the skill has no `active` or `deprecated` version;
if no version resolves, the create request fails with
`RESOURCE_DOES_NOT_EXIST`. Because the concrete version is captured on creation,
`member_version` is `NOT NULL` and a later change to the skill's latest version
or alias target does not alter existing member rows.

## Store interface

The store interface follows the mixin pattern established by the MCP
Server Registry (RFC-0004). Methods raise `NotImplementedError` rather
than using `@abstractmethod`, allowing stores that do not support
skills (e.g., `FileStore`) to work without stubbing every method.

In the store interface, `delete_*` methods on top-level entities are
hard deletes, while `delete_*_version` methods are soft deletes that
transition the version to `deleted`.

Version-creation methods (`create_skill_version`,
`create_agent_plugin_version`) receive the server-resolved `source_type`
as a flat field. The server infers it per the field-inference rules
before calling the store, and the store persists it without re-inferring.
At this layer `source_type` is server-internal, not a user-facing input;
the client-facing `MlflowClient` and `mlflow.genai` methods take only
`source`.

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
        ref: str | None = None,
        subpath: str | None = None,
        status: str = "active",
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
        """Raises RESOURCE_DOES_NOT_EXIST when the skill has no
        non-deleted version (nothing resolves as latest)."""
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
        """Raises RESOURCE_DOES_NOT_EXIST when the target version does
        not exist or is deleted, so an alias can never dangle. Any other
        status (draft, active, deprecated) is a valid target."""
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
        version: str | None = None,
        plugin_json: dict[str, Any] | None = None,
        skills: list[str] | None = None,
        source_type: str | None = None,
        source: str | None = None,
        ref: str | None = None,
        subpath: str | None = None,
        status: str = "active",
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_agent_plugin_version(
        self, name: str, version: str,
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
        """Raises RESOURCE_DOES_NOT_EXIST when the plugin has no
        non-deleted version (nothing resolves as latest)."""
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
        version: str,
        organization: str = "",
        status: SkillStatus | None = NOT_SET,
    ) -> AgentPluginVersion:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_version(
        self, name: str, version: str,
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
        self, name: str, version: str,
        key: str, value: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_agent_plugin_version_tag(
        self, name: str, version: str, key: str,
        organization: str = "",
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- AgentPlugin alias operations ---

    def set_agent_plugin_alias(
        self, name: str, alias: str, version: str,
        organization: str = "",
    ) -> None:
        """Raises RESOURCE_DOES_NOT_EXIST when the target version does
        not exist, is deleted, or is withdrawn because it contains a
        deleted member, so an alias can never dangle or bypass the kill
        switch. Any other status (draft, active, deprecated) is a valid
        target."""
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

The SDK is split into two layers, following the MLflow model and
prompt registries. The `mlflow.genai` namespace exposes a small set
of high-level, workflow-oriented functions for the common cases
(register, import, introspect, pull, search). The full low-level
create/get/update/delete surface, including version, tag, and alias
operations, lives on `MlflowClient`; the `mlflow.genai` helpers call
these methods internally. This keeps the top-level namespace concise
while still making every operation reachable. The two search
functions appear in both layers, mirroring
`mlflow.search_registered_models` and
`MlflowClient.search_registered_models` in the model registry.

### High-level workflow functions (`mlflow.genai`)

```python
from dataclasses import dataclass

import mlflow


def register_skill(
    *,
    name: str | None = None,
    organization: str = "",
    source: GitSource | OCISource | ZipSource | str | None = None,
    status: str = "active",
) -> SkillVersion:
    """Register a skill version. The server assigns the next
    monotonic integer version. Auto-creates the parent Skill if
    it does not exist (with null description) and otherwise reuses
    the existing parent. To set parent-level metadata, use
    MlflowClient.create_skill() before registering versions or
    MlflowClient.update_skill() afterward. This matches the MCP
    Server Registry behavior
    (register_mcp_server). If name is omitted, the name is
    extracted from the skill's SKILL.md entry point (server-side
    for remote sources, client-side for local paths). The server
    sets source_type from the source value (git, oci, or zip for
    external pointers) or from the creation flow (mlflow when the
    client uploads a local path). A typed source class
    (GitSource, OCISource, ZipSource) is converted to flat REST
    fields; a plain string is also accepted for convenience and
    passed as the source field with type inferred by the server.
    If source is a local path (no :// scheme), the client uploads
    the files through existing MLflow artifact APIs."""


def search_skills(
    *,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[Skill]: ...


def register_agent_plugin(
    *,
    plugin_json: dict[str, Any] | None = None,
    name: str | None = None,
    version: str | None = None,
    organization: str = "",
    skills: list[str] | None = None,
    source: GitSource | OCISource | ZipSource | str | None = None,
    status: str = "active",
) -> AgentPluginVersion:
    """Validate or synthesize a canonical manifest, create or reuse the
    parent AgentPlugin, and register one immutable version. version is
    required when plugin_json is None or does not contain a version."""


def search_agent_plugins(
    *,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[AgentPlugin]: ...


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
    detected_format: str
    plugin_name: str | None
    plugin_json: dict[str, Any]
    skills: list[IntrospectedSkill]
    recognized_unregistered_content: list[str]
    warnings: list[PluginImportWarning]


@dataclass
class PluginImportResult:
    detected_format: str
    plugin_version: AgentPluginVersion
    skill_versions: list[SkillVersion]
    recognized_unregistered_content: list[str]
    warnings: list[PluginImportWarning]


def introspect_plugin(
    *,
    source: GitSource | OCISource | ZipSource | str,
) -> PluginIntrospectionResult:
    """Inspect a local or remote plugin without modifying the registry."""


def import_plugin(
    *,
    source: GitSource | OCISource | ZipSource | str,
    plugin_name: str | None = None,
    version: str | None = None,
    organization: str = "",
) -> PluginImportResult:
    """Import a plugin as a monolithic agent plugin.

    Auto-detects Agent Plugins, Claude Code, or generic skill-directory input;
    validates or constructs canonical plugin_json; discovers and registers
    skills; and preserves the package source. Recognized mcp.json content is
    reported but not registered. plugin_name is used only when the selected
    adapter cannot derive a name. version is required when the detected
    manifest does not contain a version field. A typed source class
    (GitSource, OCISource, ZipSource) is converted to flat REST fields; a
    plain string is also accepted for convenience.
    """


def pull(
    *,
    name: str | None = None,
    organization: str = "",
    entity_type: str = "skill",
    version: int | str | None = None,
    alias: str | None = None,
    destination: str = ".",
) -> str:
    """Pull skill or agent plugin content from registered sources to a
    local directory. Set entity_type to 'skill' or 'agent_plugin'."""


# Example usage:
version = mlflow.genai.register_skill(
    name="code-review",
    source=GitSource(url="https://github.com/acme/skills.git", ref="v1.0.0"),
)
# version.name and version.organization identify the skill for subsequent operations
skills = mlflow.genai.search_skills(filter_string="status = 'active'")
```

### Low-level CRUD (`MlflowClient` methods)

The full create/get/update/delete surface, including version, tag,
and alias operations, is exposed as `MlflowClient` methods. The
`mlflow.genai` helpers above call these internally. `search_skills`
and `search_agent_plugins` appear in both layers, mirroring
`search_registered_models` in the model registry.

```python
class MlflowClient:
    # --- Skills ---
    def create_skill(
        self, *, name: str, organization: str = "", description: str | None = None
    ) -> Skill: ...

    def get_skill(self, *, name: str, organization: str = "") -> Skill: ...

    def search_skills(
        self,
        *,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[Skill]: ...

    def update_skill(
        self, *, name: str, organization: str = "", description: str | None = NOT_SET
    ) -> Skill: ...

    def delete_skill(self, *, name: str, organization: str = "") -> None: ...

    def create_skill_version(
        self,
        *,
        name: str,
        organization: str = "",
        source: GitSource | OCISource | ZipSource | str | None = None,
    ) -> SkillVersion: ...

    def get_skill_version(
        self, *, name: str, version: int, organization: str = ""
    ) -> SkillVersion: ...

    def get_skill_version_by_alias(
        self, *, name: str, alias: str, organization: str = ""
    ) -> SkillVersion: ...

    def get_latest_skill_version(
        self, *, name: str, organization: str = ""
    ) -> SkillVersion: ...

    def search_skill_versions(
        self,
        *,
        name: str,
        organization: str = "",
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillVersion]: ...

    def update_skill_version(
        self, *, name: str, version: int, organization: str = "", status: str | None = NOT_SET
    ) -> SkillVersion: ...

    def delete_skill_version(
        self, *, name: str, version: int, organization: str = ""
    ) -> None: ...

    # --- Agent plugins ---
    def create_agent_plugin(
        self, *, name: str, organization: str = "", description: str | None = None
    ) -> AgentPlugin: ...

    def create_agent_plugin_version(
        self,
        *,
        name: str,
        organization: str = "",
        version: str | None = None,
        plugin_json: dict[str, Any] | None = None,
        skills: list[str] | None = None,
        source: GitSource | OCISource | ZipSource | str | None = None,
        status: str = "active",
    ) -> AgentPluginVersion:
        """version is required when plugin_json is None or when
        plugin_json does not contain a version field."""

    def get_agent_plugin(self, *, name: str, organization: str = "") -> AgentPlugin: ...

    def search_agent_plugins(
        self,
        *,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[AgentPlugin]: ...

    def update_agent_plugin(
        self, *, name: str, organization: str = "", description: str | None = NOT_SET
    ) -> AgentPlugin: ...

    def delete_agent_plugin(self, *, name: str, organization: str = "") -> None: ...

    def get_agent_plugin_version(
        self, *, name: str, version: str, organization: str = ""
    ) -> AgentPluginVersion: ...

    def get_agent_plugin_version_by_alias(
        self, *, name: str, alias: str, organization: str = ""
    ) -> AgentPluginVersion: ...

    def get_latest_agent_plugin_version(
        self, *, name: str, organization: str = ""
    ) -> AgentPluginVersion: ...

    def search_agent_plugin_versions(
        self,
        *,
        name: str,
        organization: str = "",
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[AgentPluginVersion]: ...

    def update_agent_plugin_version(
        self, *, name: str, version: str, organization: str = "", status: str | None = NOT_SET
    ) -> AgentPluginVersion: ...

    def delete_agent_plugin_version(
        self, *, name: str, version: str, organization: str = ""
    ) -> None: ...

    # --- Tags and aliases ---
    def set_skill_tag(self, *, name: str, key: str, value: str, organization: str = "") -> None: ...

    def delete_skill_tag(self, *, name: str, key: str, organization: str = "") -> None: ...

    def set_skill_version_tag(self, *, name: str, version: int, key: str, value: str, organization: str = "") -> None: ...

    def delete_skill_version_tag(self, *, name: str, version: int, key: str, organization: str = "") -> None: ...

    def set_skill_alias(self, *, name: str, alias: str, version: int, organization: str = "") -> None: ...

    def delete_skill_alias(self, *, name: str, alias: str, organization: str = "") -> None: ...

    def set_agent_plugin_tag(self, *, name: str, key: str, value: str, organization: str = "") -> None: ...

    def delete_agent_plugin_tag(self, *, name: str, key: str, organization: str = "") -> None: ...

    def set_agent_plugin_version_tag(self, *, name: str, version: str, key: str, value: str, organization: str = "") -> None: ...

    def delete_agent_plugin_version_tag(self, *, name: str, version: str, key: str, organization: str = "") -> None: ...

    def set_agent_plugin_alias(self, *, name: str, alias: str, version: str, organization: str = "") -> None: ...

    def delete_agent_plugin_alias(self, *, name: str, alias: str, organization: str = "") -> None: ...
```

For SDK update methods, `NOT_SET` means "leave unchanged" while `None`
means "clear this nullable field". This mirrors the store-layer update
contract so callers can distinguish partial updates from explicit
nulling.

`pull` is implemented in the SDK/CLI layer, not the store mixin. The
client calls `get_skill_version` (or resolves an alias) to obtain the
version's `source_type` and source pointer, then routes on `source_type`:
`git` clone, `oci` pull, `zip` download, or `mlflow` artifact-tree
download. A skill version with `source_type="embedded"` is not
standalone-pullable and returns an error directing the caller to pull the
containing monolithic agent plugin.
For agent plugin pulls, the stored `plugin.json` manifest is always
written to the destination root. Assembled plugin members are placed
under `skills/<member-name>/` to match the Agent Plugins `skills/*/SKILL.md`
discovery layout, making the result a conformant Agent Plugins package.
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
The server sets `source_type`; it is not a user-facing parameter. For
external pointers it is inferred from the `source` value: `.git` suffix
or `git://` scheme = git, `oci://` scheme = oci, `.zip` = zip. The
`oci://` scheme is an inference hint only: the server records
`source_type="oci"` and persists the bare image reference without the
scheme, so `pull` and `introspect` operate on the native OCI reference.
For content
without an external pointer the server sets it from the creation flow:
a standalone skill or a monolithic plugin package uploaded with a null
`source` becomes `mlflow` (content stored in MLflow artifact storage);
a skill created as a member during monolithic agent plugin import becomes
`embedded` (the content lives inside the agent plugin package rather than
in standalone storage); and an agent plugin version created from member
references with no plugin-level package becomes `assembled`. Keeping
`source_type` server-set keeps SDKs as thin REST wrappers, avoids
reimplementing inference in every language, and prepares for future
server-side content inspection (e.g., signature verification).

For agent plugins, `POST /register` and version creation validate the submitted
canonical `plugin_json`. The server extracts and checks `name` and validates
the version as SemVer (normalizing semverish values). When `plugin_json` is
absent or does not contain a version, the request must include a `version`
field. When both `plugin_json["version"]` and the request-level `version` are
present, they must agree; a mismatch is rejected. The server does not fetch remote package content. Client-side
`import_plugin()` performs source fetching, format detection, filesystem
validation, and adapter translation before submitting the canonical payload and
member references.

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
| `GET` | `/@{organization}/{name}` | Get skill by organization and name |
| `PATCH` | `/@{organization}/{name}` | Update skill fields |
| `DELETE` | `/@{organization}/{name}` | Hard-delete skill (cascades, subject to references) |
| `POST` | `/@{organization}/{name}/versions` | Create a skill version |
| `GET` | `/@{organization}/{name}/versions` | Search versions |
| `GET` | `/@{organization}/{name}/versions/{version}` | Get a specific version |
| `PATCH` | `/@{organization}/{name}/versions/{version}` | Update version |
| `DELETE` | `/@{organization}/{name}/versions/{version}` | Soft-delete a version (`status='deleted'`) |
| `POST` | `/@{organization}/{name}/tags` | Set a skill-level tag |
| `DELETE` | `/@{organization}/{name}/tags/{key}` | Delete a skill-level tag |
| `POST` | `/@{organization}/{name}/versions/{version}/tags` | Set a version-level tag |
| `DELETE` | `/@{organization}/{name}/versions/{version}/tags/{key}` | Delete a version tag |
| `POST` | `/@{organization}/{name}/aliases` | Set an alias |
| `GET` | `/@{organization}/{name}/aliases/{alias}` | Resolve alias to `SkillVersion` |
| `DELETE` | `/@{organization}/{name}/aliases/{alias}` | Delete an alias |

Each entity-specific row above (those whose path begins
`/@{organization}/{name}`) shows the with-organization route. Every such
operation is exposed as two concrete route templates: the with-organization
form listed and a no-organization form that is the same path with the leading
`@{organization}/` segment removed. For
example, "Get a specific version" is both
`GET /@{organization}/{name}/versions/{version}` (organization present) and
`GET /{name}/versions/{version}` (no organization); "Get skill by
organization and name" is both `GET /@{organization}/{name}` and
`GET /{name}`. There is no placeholder for the empty organization: the
`@{organization}` segment and its slash are omitted entirely. So
`GET /code-review/versions/1` addresses a skill with no organization, and
`GET /@acme/code-review/versions/1` addresses the same-named skill in
organization `acme`.

This mirrors the URI format, which uses the same `@organization` marker and
likewise omits it when empty. The `@` marker is what keeps the paths
unambiguous even though fixed keyword subresource segments (`versions`,
`tags`, `aliases`) follow the name: an organization segment always begins
with `@` and names cannot begin with `@`. So `/acme/versions` can only mean
`(no organization, name=acme, the versions collection)`, never
`(organization=acme, name=versions)`; and conversely `/@acme/versions` can
only mean the skill literally named `versions` in organization `acme`, never
a versions collection.

Because both the with-organization template `/@{organization}/{name}` and the
no-organization template `/{name}/versions` structurally match a two-segment
path, the routing layer must not rely on match order alone: the `{name}`
segment (and the leading no-organization segment) is constrained to reject a
leading `@`, so `/@acme/versions` binds only the organization template and
`/acme/versions` binds only the no-organization template. Splitting each
operation into a with-organization and a no-organization route template is
the cost of omitting the empty-organization segment, but it needs no
placeholder value and no reserved subresource keywords. The artifact storage
paths use the same `@organization` convention.

Similarly, agent plugin endpoints are exposed under both
`/api/3.0/mlflow/agent-plugins` and
`/ajax-api/3.0/mlflow/agent-plugins`.

### Agent plugin endpoints

All paths relative to the logical agent-plugins router prefix.

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Create an agent plugin |
| `GET` | `/` | Search agent plugins |
| `POST` | `/register` | Validate or synthesize `plugin_json`, create or reuse the parent, and create a version |
| `GET` | `/@{organization}/{name}` | Get agent plugin by organization and name |
| `PATCH` | `/@{organization}/{name}` | Update agent plugin fields |
| `DELETE` | `/@{organization}/{name}` | Hard-delete agent plugin (cascades versions and memberships) |
| `POST` | `/@{organization}/{name}/versions` | Create an agent plugin version with members |
| `GET` | `/@{organization}/{name}/versions` | Search agent plugin versions |
| `GET` | `/@{organization}/{name}/versions/{version}` | Get a specific agent plugin version |
| `PATCH` | `/@{organization}/{name}/versions/{version}` | Update agent plugin version status |
| `DELETE` | `/@{organization}/{name}/versions/{version}` | Soft-delete an agent plugin version (`status='deleted'`) |
| `POST` | `/@{organization}/{name}/tags` | Set an agent plugin-level tag |
| `DELETE` | `/@{organization}/{name}/tags/{key}` | Delete an agent plugin-level tag |
| `POST` | `/@{organization}/{name}/versions/{version}/tags` | Set an agent plugin version tag |
| `DELETE` | `/@{organization}/{name}/versions/{version}/tags/{key}` | Delete an agent plugin version tag |
| `POST` | `/@{organization}/{name}/aliases` | Set an agent plugin alias |
| `GET` | `/@{organization}/{name}/aliases/{alias}` | Resolve agent plugin alias to version |
| `DELETE` | `/@{organization}/{name}/aliases/{alias}` | Delete an agent plugin alias |

Same `@organization` segment convention as skill endpoints: each
entity-specific row (those beginning `/@{organization}/{name}`) shows the
with-organization route, and every such operation also has a no-organization
route with the leading `@{organization}/` segment removed. Agent plugin
version path values follow the same length and reserved character validation
as Agent Plugin URIs.

### Pagination and filtering

Search endpoints use page-token-based pagination and `filter_string`
expressions following existing MLflow conventions. Free-text discovery is also
expressed through `filter_string`: parent search endpoints expose a derived
`search_text` field that is matched with `LIKE`/`ILIKE` (e.g.,
`search_text LIKE '%review%'`). This field concatenates several
discovery fields into one column so that a single `LIKE` matches across
all of them without requiring `OR`, which MLflow `filter_string` does not
support. For skills the projection is derived from name and description; for
agent plugins it is derived from the fields listed below.

The two entity kinds store `search_text` in different places. A skill's
discovery fields (name, description) all live on the parent row, so `search_text`
is a column on the `skills` parent table. An agent plugin's discovery fields
come mostly from the manifest, which is version-scoped, so there is no
`search_text` column on the `agent_plugins` parent; the column lives on
`agent_plugin_versions`, and parent search joins to the latest-resolved version
and matches that version's `search_text`. This is why a lifecycle transition
that changes which version resolves as latest can change the parent's free-text
matches. Because a version's `search_text` also folds in the mutable parent
description, updating that description recomputes `search_text` across the
plugin's version rows, so parent search stays consistent regardless of which
version later resolves as latest.

**Skills:** the `search_text` field covers name and description. Structured
examples include `name LIKE '%review%'`, `description LIKE '%security%'`,
`organization = 'acme'`, `status = 'active'`, and `tags.team = 'platform'`.

**Agent plugins:** the `search_text` field covers name, mutable parent
description, organization, and the latest-resolved manifest's description,
keywords, and author name. Structured filters are the same as Skills, plus
`member_name = 'code-review'` to find agent plugins that include a given skill.
Unlike the other parent filters, which resolve against the latest version,
`member_name` matches a plugin when any of its versions lists that member, not
only the latest-resolved one. This makes it the discovery path for "which
plugins depend on this skill," including plugins pinned to an older version, now
that memberships are not a stored field on the skill.

On parent search, a `status` filter matches the parent's derived status,
resolved from the same latest-resolved version that drives `latest_version`
and the entity's read-only `status`, since parent tables have no status
column of their own. A parent with no non-`deleted` version has a `None`
derived status and is excluded by any `status` equality filter. To filter on
the status of a specific version, use version search.

**Versions (all entity types):** `status = 'active'`,
`organization = 'acme'`, `source_type = 'git'`, and
`tags.approved = 'true'`.

Manifest keywords are not copied into MLflow tags. A derived version-level
search projection supports portable free-text matching while `plugin_json`
remains canonical.

### Request and response models

Version-creation requests include immutable creation payloads and mutable
initial status; later update requests contain only mutable fields. Resource
identifiers normally come from path parameters. The Skill and Agent Plugin
`POST /register` endpoints accept identity inputs in the body so they can create
or reuse the parent and create a version in one operation. Agent Plugin identity
is extracted from or checked against `plugin_json`.

```python
from typing import Any

from pydantic import BaseModel, ConfigDict, Field


class CreateSkillRequest(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None


class UpdateSkillRequest(BaseModel):
    description: str | None = None


class CreateSkillVersionRequest(BaseModel):
    name: str | None = None  # optional for POST /register (inferred from source when omitted); ignored for POST /@{organization}/{name}/versions (name from parent)
    organization: str = ""  # for POST /register only; ignored for versioned paths
    source: str | None = None
    ref: str | None = None
    subpath: str | None = None
    status: str = "active"


class UpdateSkillVersionRequest(BaseModel):
    status: str | None = None


class CreateAgentPluginRequest(BaseModel):
    name: str
    organization: str = ""
    description: str | None = None


class UpdateAgentPluginRequest(BaseModel):
    description: str | None = None


class PluginJSONPayload(BaseModel):
    model_config = ConfigDict(extra="allow", populate_by_name=True)

    schema_: str = Field(alias="$schema")
    name: str
    version: str | None = None
    description: str | None = None
    author: dict[str, str] | None = None
    homepage: str | None = None
    repository: str | None = None
    license: str | None = None
    keywords: list[str] | None = None
    extensions: Any = None


class CreateAgentPluginVersionRequest(BaseModel):
    version: str | None = None
    plugin_json: PluginJSONPayload | None = None
    skills: list[str] | None = None
    source: str | None = None
    ref: str | None = None
    subpath: str | None = None
    status: str = "active"


class RegisterAgentPluginRequest(CreateAgentPluginVersionRequest):
    name: str | None = None
    organization: str = ""


class UpdateAgentPluginVersionRequest(BaseModel):
    status: str | None = None


class SkillAliasResponse(BaseModel):
    alias: str
    version: int


class SetSkillAliasRequest(BaseModel):
    alias: str
    version: int


class AgentPluginAliasResponse(BaseModel):
    alias: str
    version: str


class SetAgentPluginAliasRequest(BaseModel):
    alias: str
    version: str


class SetTagRequest(BaseModel):
    key: str
    value: str


class SkillVersionResponse(BaseModel):
    name: str
    version: int
    organization: str = ""
    source_type: str | None = None
    source: str | None = None
    ref: str | None = None
    subpath: str | None = None
    status: str = "active"
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
    aliases: list[SkillAliasResponse] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class AgentPluginVersionResponse(BaseModel):
    name: str
    version: str
    organization: str = ""
    plugin_json: dict[str, Any]
    source_type: str | None = None
    source: str | None = None
    ref: str | None = None
    subpath: str | None = None
    status: str = "active"
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
    latest_version: str | None = None
    aliases: list[AgentPluginAliasResponse] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

Aliases are modeled as dictionaries on parent entities for convenience, while
REST responses expose explicit lists. Skill alias targets are integers and
agent plugin alias targets are strings.

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

**Canonical manifest validation.** `PluginJSONPayload` provides typed access to
known fields using `extra="allow"` so that unknown fields at any level are
accepted and preserved. The `$schema` identifier is not gated on a specific
version; manifests with newer schema identifiers are accepted as long as the
required fields are present. This matches RFC-0004's forward-compatibility
approach for MCP `server_json`. A non-object `extensions` field is preserved
and reported as a nonfatal warning but ignored semantically. Missing or
invalid required fields are fatal. `plugin_json["name"]` must match an explicit path or request name. When the
manifest does not include a version, the request must supply one. When both
are present, they must agree.

**Name and organization validation.** Skill and agent plugin identifiers
become segments of URIs, REST paths, and artifact storage paths, so the
server validates them on create and rejects any value that could introduce
path ambiguity or escape its controlled storage prefix:

- **Skill `name`:** 1 to 64 characters; lowercase ASCII letters (`a-z`),
  digits (`0-9`), and hyphens only; must not begin or end with a hyphen;
  and no consecutive hyphens. This matches the
  [Agent Skills naming rules](https://agentskills.io/specification), so no
  otherwise-valid skill name is rejected.
- **Agent plugin `name`:** the canonical Agent Plugins constraints: 1 to 64
  characters; lowercase ASCII letters, digits, hyphens, and periods;
  alphanumeric first and last characters; and no consecutive hyphens or
  periods. Agent plugin names that parse as valid SemVer are permitted,
  since the `@` marker distinguishes an organization from a `name/version`
  sequence without relying on SemVer recognition.
- **`organization`** (both entity types): empty (the default), or the same
  rule as an agent plugin name, which also allows periods so domain-style
  publisher names such as `acme.io` are valid: 1 to 64 characters; lowercase
  ASCII letters, digits, hyphens, and periods; alphanumeric first and last
  characters; and no consecutive hyphens or periods.

These rules exclude path-significant values by construction: `/`, `\`, `@`,
`#`, `?`, whitespace, control characters, percent-encoded separators, and
dot-segments such as `.` and `..` cannot appear in any identifier, and the
leading `@` that marks an organization segment is never part of a value.
Because an organization is always marked by a leading `@` in URIs and REST
paths, the parser never has to guess whether a segment is an organization or
a name; numeric skill names (e.g., `123`) are therefore permitted.

As defense in depth, artifact operations build storage paths only from
validated segments (organization, name, and version) and verify that the
resolved path stays within the controlled prefix (`skills/...` or
`agent-plugins/...`), rejecting any operation that would resolve outside it.

## Python SDK and CLI

The `mlflow.genai` module exposes the high-level workflow functions
(register, search, import, introspect, pull); the full create/get/
update/delete surface, including version, tag, and alias operations,
is available as `MlflowClient` methods, which the `mlflow.genai`
helpers call internally. Skill-specific entity and request types are
also re-exported from `mlflow.genai`. The `mlflow skills` and
`mlflow agent-plugins` CLI command groups provide the same operations
from the command line, mapping to whichever layer defines each
function (the SDK function column below). Identity-establishing commands
(`register`, and `agent-plugins create`) accept `--name` with an optional
`--organization`. Commands that operate on
an already-registered entity accept its URI as an optional positional argument,
with the equivalent `--skill-uri` (skills) or `--plugin-uri` (agent plugins)
flag also accepted; for example, `mlflow skills get skills:/code-review` and
`mlflow skills get --skill-uri skills:/code-review` are equivalent:

| CLI subcommand | SDK function | Description |
|---|---|---|
| `mlflow skills register git` | `register_skill(source=GitSource(...))` | Register from a Git repository |
| `mlflow skills register oci` | `register_skill(source=OCISource(...))` | Register from an OCI image |
| `mlflow skills register zip` | `register_skill(source=ZipSource(...))` | Register from a ZIP archive |
| `mlflow skills register` | `register_skill(source="./local-path")` | Register from a local directory (uploaded to MLflow artifact storage) |
| `mlflow skills get` | `get_skill()` | Get skill metadata |
| `mlflow skills search` | `search_skills()` | Search skills |
| `mlflow skills get-version` | `get_skill_version()` | Get a specific version |
| `mlflow skills update-version` | `update_skill_version()` | Update version status |
| `mlflow skills set-alias` | `set_skill_alias()` | Set a version alias |
| `mlflow skills set-tag` | `set_skill_tag()` | Set a tag |
| `mlflow skills pull` | `pull()` | Pull content to local filesystem |
| `mlflow agent-plugins create` | `create_agent_plugin()` | Create an agent plugin |
| `mlflow agent-plugins create-version` | `create_agent_plugin_version()` | Create a version on an existing parent |
| `mlflow agent-plugins register` | `register_agent_plugin()` | Create or reuse the parent and register a canonical version |
| `mlflow agent-plugins get` | `get_agent_plugin()` | Get agent plugin metadata |
| `mlflow agent-plugins search` | `search_agent_plugins()` | Search agent plugins |
| `mlflow agent-plugins search-versions` | `search_agent_plugin_versions()` | Search agent plugin versions |
| `mlflow agent-plugins set-alias` | `set_agent_plugin_alias()` | Set an agent plugin alias |
| `mlflow agent-plugins set-tag` | `set_agent_plugin_tag()` | Set an agent plugin-level tag |
| `mlflow agent-plugins set-version-tag` | `set_agent_plugin_version_tag()` | Set an agent plugin version tag |
| `mlflow agent-plugins update-version` | `update_agent_plugin_version()` | Update agent plugin version status |
| `mlflow agent-plugins introspect` | `introspect_plugin()` | Preview a local or remote plugin without registry writes |
| `mlflow agent-plugins import` | `import_plugin()` | Import a plugin as a monolithic agent plugin |

`create-version` and `register` accept `--plugin-json PATH` for a full standard
manifest. When omitted for an assembled plugin, `--version` is required and
the command synthesizes a minimal manifest from the target identity (the
positional URI or `--plugin-uri` for `create-version`, which targets an
existing parent; `--name` for `register`) and the supplied version. Search
commands accept `--filter-string`, which
covers both structured filters and free-text matching against the derived
`search_text` field (e.g., `--filter-string "search_text LIKE '%review%'"`).

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
They accept either a local path or a remote Git, OCI, ZIP, or MLflow artifact
source and return the detected format, canonical manifest preview, discovered
skill names and paths, recognized unregistered content such as `mcp.json`, and
warnings. The server infers `source_type` from the source value using the same
rules as registration.

### Supported input formats

The importer:

1. Fetches the Git, OCI, ZIP, or MLflow artifact source using the same
   source-type-aware logic as `pull`.
2. Applies `subpath`, when provided, to select the plugin root.
3. Checks for a root `plugin.json` with an Agent Plugins `$schema` identifier.
   When found, it validates the required fields and discovers immediate children
   at `skills/*/SKILL.md`. A manifest missing required fields fails import.
   Unknown fields and newer schema versions are accepted and preserved.
4. Otherwise checks for `.claude-plugin/plugin.json`, translates available
   metadata into canonical `plugin_json`, and applies Claude Code discovery.
5. Otherwise applies the existing generic skill-directory discovery and
   synthesizes a minimal canonical manifest. `plugin_name` is used only when
   the adapter cannot infer a name.
6. Reports root `mcp.json` as recognized but unregistered standard content and
   reports other non-skill content without assigning membership semantics.

When multiple markers exist, the standard root manifest takes precedence. The
canonical name follows the Agent Plugins naming constraints. A supplied
manifest version is used as the registry version (normalized to full
SemVer); when the manifest does not include a version, the user must
supply one via `--version` or the `version` parameter.

### Registration behavior

For each discovered skill, the importer creates a `SkillVersion` whose
version is server-assigned, whose `source` is `None` (no typed source
class, no `ref`), and whose `source_type` the server sets to `embedded`.
The importer references each embedded skill by name in
the member list (e.g., `skills:/embedded-review/1`).

After registering the embedded skills, the importer creates one
monolithic `AgentPluginVersion` whose `source_type` reflects where the
package lives: the original typed source (preserving `ref` for Git
sources) for a `git`, `oci`, or `zip` package, or `mlflow` with a null
`source` for a package stored in MLflow artifact storage. It carries the
immutable canonical `plugin_json` and member references for all
discovered skills. This preserves a pullable link to the
complete original package while keeping registry membership limited to skills.
A valid package with no skills creates an agent plugin version with an empty
member list.

#### Re-import behavior

When the target agent plugin already has at least one version, import first
rejects an incoming canonical version that already exists. It then
matches discovered skills to existing members by name, using the
member skill names from the most recently created agent plugin version's member list:

1. **Matching name:** The discovered skill's name matches a member name
   in the previous member list. Import
   creates a new version of that existing skill.
2. **New name:** The name does not match any previous member.
   Import creates a new skill with its own next server-assigned integer version.
3. **Removed name:** A previous member's name is not found in the
   new source. The member is omitted from the new agent plugin version.
   The skill and its existing versions remain in the registry.

A skill that is renamed between versions is treated as a removed skill
and a new one.

After processing all discovered skills, import creates a new
`AgentPluginVersion` with updated member references. Previous agent
plugin versions are immutable and unchanged.

Agent plugin and embedded skill version sequences are independent: a plugin
version such as `"1.2.0"` may reference skill version `7`.

### Warnings and result

Root `mcp.json` and other non-skill content remain in the package but are not
registered. Recognized standard content is returned separately from warnings;
adapter-specific or unknown skipped categories produce a `PluginImportWarning`
containing category, path, and explanation. The CLI prints the detected format,
canonical identity, recognized unregistered content, and warnings. The SDK
returns them with the created versions in `PluginImportResult`.

Import normalizes supported inputs into the canonical Agent Plugins registry
representation while preserving the complete package source. It does not
install or export an MLflow agent plugin.

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
    source=OCISource(
        image="oci://ghcr.io/acme/agent-plugin:v1.0.0",
        subpath="skills/code-review",
    ),
)

mlflow.genai.register_skill(
    name="test-coverage",
    source=OCISource(
        image="oci://ghcr.io/acme/agent-plugin:v1.0.0",
        subpath="skills/test-coverage",
    ),
)

# Assembled agent plugin: each member has its own source. With no explicit
# plugin_json, version is required and a minimal manifest is synthesized.
plugin_version = mlflow.genai.register_agent_plugin(
    name="pr-workflow",
    version="0.1.0",
    skills=[
        "skills:/code-review/1",
        "skills:/test-coverage/1",
    ],
)
```

Monolithic agent plugins are typically created through `import_plugin()`, which
handles package inspection and embedded skill creation internally. See the
[Plugin import](#plugin-import) section for details.

### Discover and consume skills

```python
from mlflow import MlflowClient

client = MlflowClient()

# Free-text discovery across skill names and descriptions (high-level API)
skills = mlflow.genai.search_skills(filter_string="search_text LIKE '%code review%'")
skill = skills[0]

# Search for active versions of that skill (low-level CRUD on MlflowClient)
versions = client.search_skill_versions(
    name=skill.name,
    organization=skill.organization,
    filter_string="status = 'active'",
)

# Search for active agent plugins (high-level API)
plugins = mlflow.genai.search_agent_plugins(
    filter_string="search_text LIKE '%pull request review%' AND status = 'active'",
)

# Get a specific version
version = client.get_skill_version(
    name=skill.name,
    organization=skill.organization,
    version=1,
)
# isinstance(version.source, GitSource)
# version.source.url == "https://github.com/acme/agent-skills.git"
# version.source.ref == "v1.0.0"
# version.source.subpath == "code-review"

# Resolve by alias
version = client.get_skill_version_by_alias(
    name=skill.name,
    organization=skill.organization,
    alias="production",
)

# Get an agent plugin version and its pinned members
plugin = mlflow.genai.search_agent_plugins(
    filter_string="name = 'pr-workflow'",
)[0]
plugin_version = client.get_agent_plugin_version(
    name=plugin.name,
    organization=plugin.organization,
    version="0.1.0",
)
# plugin_version.skills == ["skills:/code-review/1", ...]

# Resolve an agent plugin alias
plugin_version = client.get_agent_plugin_version_by_alias(
    name=plugin.name,
    organization=plugin.organization,
    alias="production",
)
```
