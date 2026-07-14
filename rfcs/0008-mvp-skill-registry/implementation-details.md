# RFC-0008: Skill Registry Implementation Details

This document contains implementation-level specifications for
RFC-0008 (Skill Registry). It covers database schema, entity
dataclasses, store interface method signatures, REST API endpoints,
pagination/filtering, SDK convenience functions, CLI mapping, package
manager plugin interface, and trace integration details. These details
support implementers; the main RFC covers the design rationale.

## Database schema

Tables are created via a single Alembic migration. All tables are
workspace-scoped.

### `skills`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `name` | `String(256)` | PK |
| `display_name` | `String(256)` | mutable human-readable label |
| `description` | `String(5000)` | |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

### `skill_versions`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `version` | `String(256)` | PK, valid semantic version |
| `version_major` | `Integer` | extracted from validated semantic version |
| `version_minor` | `Integer` | extracted from validated semantic version |
| `version_patch` | `Integer` | extracted from validated semantic version |
| `version_prerelease_sort_key` | `String(512)` | lexicographically sortable encoding of prerelease identifiers |
| `display_name` | `String(256)` | mutable human-readable label |
| `source_type` | `String(20)` | nullable; `git`, `oci`, `zip`, `mlflow` |
| `source` | `String(2048)` | nullable pointer to skill content |
| `subpath` | `String(2048)` | nullable; path within the artifact |
| `content_digest` | `String(512)` | optional integrity digest |
| `status` | `String(20)` | default `'draft'` |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, name)` references `skills`, CASCADE delete. This
supports administrative hard deletion of the parent `Skill`; normal
version deletion is a status transition to `deleted` and does not
physically remove the version row.

**Semantic version ordering**: `version_major`, `version_minor`,
`version_patch`, and `version_prerelease_sort_key` are materialized
from the validated semantic version string at write time. The
prerelease sort key is a lexicographically sortable encoding of the
prerelease identifiers, following the approach in the MCP Server
Registry implementation (mlflow/mlflow#23952). Release versions
encode to a sentinel that sorts above all prerelease encodings, so
full semver precedence is resolved in SQL without application-level
tie-breaking. Build metadata is ignored for precedence.

**Index**: `ix_skill_versions_latest_lookup` on `(workspace, name,
status, version_major, version_minor, version_patch)` supports
latest-resolution lookups. The prerelease sort key is not indexed
because the major/minor/patch prefix provides sufficient pruning.

### `skill_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_version_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `version` | `String(256)` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `alias` | `String(256)` | PK |
| `version` | `String(256)` | target version string |

### `skill_bundles`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `name` | `String(256)` | PK |
| `display_name` | `String(256)` | mutable human-readable label |
| `description` | `String(5000)` | |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

### `skill_bundle_versions`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `version` | `String(256)` | PK, valid semantic version |
| `version_major` | `Integer` | extracted from validated semantic version |
| `version_minor` | `Integer` | extracted from validated semantic version |
| `version_patch` | `Integer` | extracted from validated semantic version |
| `version_prerelease_sort_key` | `String(512)` | lexicographically sortable encoding of prerelease identifiers |
| `display_name` | `String(256)` | mutable human-readable label |
| `source_type` | `String(20)` | optional; `git`, `oci`, `zip`, `mlflow` |
| `source` | `String(2048)` | optional pointer to bundle artifact |
| `subpath` | `String(2048)` | nullable; path within the artifact |
| `content_digest` | `String(512)` | optional integrity digest |
| `status` | `String(20)` | default `'draft'` |
| `created_by` | `String(256)` | |
| `last_updated_by` | `String(256)` | |
| `creation_timestamp` | `BigInteger` | millis since epoch |
| `last_updated_timestamp` | `BigInteger` | millis since epoch |

FK: `(workspace, name)` references `skill_bundles`, CASCADE delete.
Semantic version ordering and index follow the same pattern as
`skill_versions`.

### `skill_bundle_version_members`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK |
| `bundle_name` | `String(256)` | PK, FK to `skill_bundle_versions` |
| `bundle_version` | `String(256)` | PK, FK to `skill_bundle_versions` |
| `member_type` | `String(20)` | PK; `skill` in Phase 1 (reserved for future extension) |
| `member_name` | `String(256)` | PK |
| `member_version` | `String(256)` | PK |
| `member_subpath` | `String(2048)` | nullable; member path inside bundle artifact |

FK: `(workspace, bundle_name, bundle_version)` references
`skill_bundle_versions`, CASCADE delete. When `member_type` is
`skill`, a FK to `skill_versions` enforces referential integrity
with RESTRICT delete.

The `member_type` column is included for forward compatibility with
RFC-0009, which will extend bundles to include non-skill members
(subagent definitions, hooks, MCP server references). In this RFC,
all members have `member_type='skill'`.

### `skill_bundle_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_bundle_version_tags`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `version` | `String(256)` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_bundle_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `alias` | `String(256)` | PK |
| `version` | `String(256)` | target bundle version string |

**Workspace handling.** All tables use `(workspace, ...)` as the leading
primary key components. Single-tenant deployments use `'default'`.

**Timestamps.** Set at the application layer via
`get_current_time_millis()`, not via DDL defaults.

**Deletion semantics.** The registry follows the mixed deletion pattern
used by the Model Registry and RFC-0004:

- Top-level entity delete operations (`delete_skill` and
  `delete_skill_bundle`) are administrative hard deletes. They
  physically remove the parent row and cascade to child rows, subject
  to referential-integrity checks.
- Version delete operations (`delete_skill_version` and
  `delete_skill_bundle_version`) are soft deletes. They set
  `status='deleted'` when allowed by the lifecycle transition rules,
  update `last_updated_timestamp`, remove aliases that point to the
  deleted version, and exclude the version from normal
  get/search/list/latest resolution. Active versions must first be
  unpublished or deprecated before they can be deleted.
- The `deleted` status is terminal. Internal audit or provenance paths
  may retain enough metadata to explain historical traces and bundle
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
    display_name: str | None = None
    description: str | None = None
    workspace: str | None = None
    status: SkillStatus | None = None  # read-only, derived from parent-resolved version
    tags: dict[str, str] = field(default_factory=dict)
    aliases: dict[str, str] = field(default_factory=dict)  # read-only; populated from skill_aliases table, e.g. {"production": "1.2.0"}
    latest_version: str | None = None  # read-only, highest active semver
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Stable logical asset name, unique within a workspace |
| `display_name` | `str` | Mutable human-readable label for UI display |
| `status` | `SkillStatus` | Read-only; derived from the parent-resolved version: highest active semantic version if present, otherwise highest non-deleted non-active semantic version |
| `aliases` | `dict[str, str]` | Stable version pointers (e.g., `{"production": "1.2.0"}`); read-only, populated from `skill_aliases` table |
| `latest_version` | `str` | Read-only; highest semantic version among `active` versions if one exists, otherwise highest non-`deleted` non-`active` version |
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
    version: str
    display_name: str | None = None
    source_type: SkillSourceType | None = None
    source: str | None = None
    subpath: str | None = None
    status: SkillStatus = SkillStatus.DRAFT
    content_digest: str | None = None
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
| `version` | `str` | Publisher-supplied version string. Semantic versioning is required (e.g., `1.0.0`, `2.1.0-beta.1`) |
| `display_name` | `str` | Mutable human-readable label for UI display |
| `source_type` | `SkillSourceType` | Optional distribution mechanism: `git`, `oci`, `zip`, `mlflow` |
| `source` | `str` | Pointer to the content in the source system. Required for standalone pull. May be omitted only when the version's content lives within a bundle-level artifact, in which case the containing bundle membership identifies the embedded content path |
| `subpath` | `str` | Optional path within the artifact where this skill's content lives. See subpath usage table below |
| `content_digest` | `str` | Optional digest for integrity verification (e.g., `sha256:abc123...`). Aligns with OCI digest terminology |
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
| `mlflow` | Not used. The artifact path is scoped to the specific skill version at upload time. |

**Git source format.** For `source_type="git"`, `source` is a Git
clone URL with an `@<ref>` suffix to identify the branch, tag, or
commit (e.g., `https://github.com/acme/agent-skills.git@v1.0.0`).
The `subpath` field identifies the path within the repository where
the skill content lives (e.g., `code-review`). This separates the
clone target, the ref, and the content path into distinct fields
rather than relying on hosting-provider-specific tree URL conventions.
Mutable refs (branches, tags) are allowed; `content_digest` can be
used to detect content drift when the ref changes.

**MLflow artifact storage (`source_type="mlflow"`).** In addition to
external source pointers, the registry supports storing skill content
directly in MLflow's artifact storage. This serves users who do not
have external Git/OCI infrastructure, who want agent capabilities
stored alongside their models, or who operate in airgapped
environments where external sources are not reachable.

Content is stored as a directory tree of individual files under an
artifact path, consistent with how MLflow stores model artifacts. For
example, a skill with a SKILL.md, scripts, and reference material is
stored as separate artifacts under a version-specific prefix:

```
skills/code-review/1.0.0/
  SKILL.md
  scripts/analyze.sh
  scripts/lint-config.json
  reference/style-guide.md
```

The `source` field contains the artifact URI as resolved by MLflow's
artifact storage (e.g., `mlflow-artifacts:/skills/code-review/1.0.0/`
when using the artifact proxy, or a direct artifact-store URI
otherwise). `source_type="mlflow"` means "stored in MLflow-managed
artifact storage," not a specific URI scheme. Pull downloads the
directory tree from the artifact store. The MLflow UI can browse
individual files within a stored skill version when artifact proxying
is enabled.

The upload API accepts a local directory path and stores each file as
a separate artifact. The `content_digest` is computed over the full
directory contents at upload time.

**Version uniqueness.** The combination of `(name, version)` is unique
within a workspace. A skill version represents a single logical
version of a capability; `source_type` and `source` describe where to
find it but are not part of its identity.

**Content integrity.** The optional `content_digest` field stores a
digest of the skill content at registration time (e.g.,
`sha256:abc123...`). For `source_type="mlflow"`, the server computes
the digest at upload time and stores it on the version; on pull, the
client recomputes the digest over the downloaded content and rejects
the result if it does not match, detecting out-of-band modification
of the underlying artifact store. For external source types (git, oci,
zip), `content_digest` is client-supplied: for OCI sources, this is
the native image digest; for Git sources, a digest of the file
contents at the pinned commit; for ZIP sources, a digest of the
archive. The registry stores the digest but does not verify it on
read; verification is the consumer's responsibility.

**Immutability contract.** `source_type`, `source`, `subpath`,
`content_digest`, and `version` are immutable after creation. To point
to different content, register a new version. Mutable fields
(`display_name`, `status`, `tags`) can be updated independently.

### SkillBundle entity

A skill bundle groups related skills into a governed unit that maps
to the "plugin" concept in agent harnesses. Follows the same
top-level pattern as Skill: versions, tags, and aliases.

```python
@dataclass
class SkillBundle:
    name: str
    display_name: str | None = None
    description: str | None = None
    workspace: str | None = None
    status: SkillStatus | None = None  # read-only, derived from parent-resolved version
    tags: dict[str, str] = field(default_factory=dict)
    aliases: dict[str, str] = field(default_factory=dict)  # read-only; populated from skill_bundle_aliases table
    latest_version: str | None = None  # read-only, highest active semver
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

`SkillBundle.status` is read-only and uses the same parent-resolved
version rule as `Skill`: highest active semantic version if present,
otherwise highest non-deleted non-active semantic version. Latest
version resolution follows the same fallback.

### SkillBundleVersion entity

A versioned snapshot of a skill bundle's membership. In Phase 1, all
members are skills.

```python
@dataclass
class SkillMemberRef:
    name: str
    version: str
    member_subpath: str | None = None


@dataclass
class SkillBundleVersion:
    name: str
    version: str
    display_name: str | None = None
    source_type: SkillSourceType | None = None
    source: str | None = None
    subpath: str | None = None
    content_digest: str | None = None
    status: SkillStatus = SkillStatus.DRAFT
    tags: dict[str, str] = field(default_factory=dict)
    skills: list[SkillMemberRef] = field(default_factory=list)
    aliases: list[str] = field(default_factory=list)
    workspace: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

### SkillBundleVersion field details

**Version uniqueness.** The combination of `(name, version)` is unique
within a workspace.

**Bundle-level source.** A bundle version is either monolithic or
assembled, never both:

- **Monolithic:** has its own `source_type`, `source`, `subpath`,
  and `content_digest`, pointing to a single artifact (e.g., an OCI
  image or Git repo) that contains the complete bundle. `pull`
  fetches the bundle artifact as a unit. Member skill versions may
  omit their own `source` because the bundle artifact is the
  authoritative source. The optional membership `member_subpath`
  identifies where each member lives inside the bundle artifact.
- **Assembled:** has individual member references. Each skill member
  has its own source. `pull` fetches members individually. If a skill
  member has no source, `pull` fails rather than producing a partial
  local bundle. For assembled bundles, `member_subpath` must be null
  because the member's own `source` and `subpath` identify its
  content.

The API rejects attempts to set `member_subpath` on a membership whose
member version has its own source.

**Immutability contract.** The member list and source fields of a
bundle version are immutable after creation. To change the set of
members or source pointer, register a new bundle version. Mutable
fields (`display_name`, `status`, `tags`) can be updated independently.

Correctness of the artifact layout is the publisher's responsibility;
the registry does not validate artifact contents at registration time.

A member can appear in multiple bundles and multiple bundle versions.
Membership is at the version level, so a bundle version is a
reproducible snapshot of "these specific skill versions work together."

## Store interface

The store interface follows the mixin pattern established by the MCP
Server Registry (RFC-0004). Methods raise `NotImplementedError` rather
than using `@abstractmethod`, allowing stores that do not support
skills (e.g., `FileStore`) to work without stubbing every method.

In the store interface, `delete_*` methods on top-level entities are
hard deletes, while `delete_*_version` methods are soft deletes that
transition the version to `deleted`.

```python
NOT_SET = object()


class SkillRegistryMixin:
    # Methods raise NotImplementedError rather than using @abstractmethod,
    # following the GatewayStoreMixin pattern. This allows stores that don't
    # support skills (e.g., FileStore) to work without stubbing every method.

    # --- Skill operations ---

    def create_skill(
        self, name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> Skill:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill(self, name: str) -> Skill:
        raise NotImplementedError(self.__class__.__name__)

    def search_skills(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[Skill]:
        raise NotImplementedError(self.__class__.__name__)

    def update_skill(
        self,
        name: str,
        display_name: str | None = NOT_SET,
        description: str | None = NOT_SET,
    ) -> Skill:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill(self, name: str) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- SkillVersion operations ---

    def create_skill_version(
        self,
        name: str,
        version: str,
        display_name: str | None = None,
        source_type: str | None = None,
        source: str | None = None,
        subpath: str | None = None,
        content_digest: str | None = None,
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_version(
        self, name: str, version: str,
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_version_by_alias(
        self, name: str, alias: str,
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_latest_skill_version(self, name: str) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def search_skill_versions(
        self,
        name: str,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillVersion]:
        raise NotImplementedError(self.__class__.__name__)

    def update_skill_version(
        self,
        name: str,
        version: str,
        display_name: str | None = NOT_SET,
        status: SkillStatus | None = NOT_SET,
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_version(
        self, name: str, version: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- Skill tag operations ---

    def set_skill_tag(
        self, name: str, key: str, value: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_tag(self, name: str, key: str) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def set_skill_version_tag(
        self, name: str, version: str,
        key: str, value: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_version_tag(
        self, name: str, version: str, key: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- Skill alias operations ---

    def set_skill_alias(
        self, name: str, alias: str, version: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_alias(
        self, name: str, alias: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- SkillBundle operations ---

    def create_skill_bundle(
        self, name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> SkillBundle:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_bundle(self, name: str) -> SkillBundle:
        raise NotImplementedError(self.__class__.__name__)

    def search_skill_bundles(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillBundle]:
        raise NotImplementedError(self.__class__.__name__)

    def update_skill_bundle(
        self,
        name: str,
        display_name: str | None = NOT_SET,
        description: str | None = NOT_SET,
    ) -> SkillBundle:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_bundle(self, name: str) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- SkillBundleVersion operations ---

    def create_skill_bundle_version(
        self,
        name: str,
        version: str,
        display_name: str | None = None,
        skills: list[SkillMemberRef] | None = None,
        source_type: str | None = None,
        source: str | None = None,
        subpath: str | None = None,
        content_digest: str | None = None,
    ) -> SkillBundleVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_bundle_version(
        self, name: str, version: str,
    ) -> SkillBundleVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_bundle_version_by_alias(
        self, name: str, alias: str,
    ) -> SkillBundleVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_latest_skill_bundle_version(
        self, name: str,
    ) -> SkillBundleVersion:
        raise NotImplementedError(self.__class__.__name__)

    def search_skill_bundle_versions(
        self,
        name: str,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillBundleVersion]:
        raise NotImplementedError(self.__class__.__name__)

    def update_skill_bundle_version(
        self,
        name: str,
        version: str,
        display_name: str | None = NOT_SET,
        status: SkillStatus | None = NOT_SET,
    ) -> SkillBundleVersion:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_bundle_version(
        self, name: str, version: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- SkillBundle tag operations ---

    def set_skill_bundle_tag(
        self, name: str, key: str, value: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_bundle_tag(
        self, name: str, key: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def set_skill_bundle_version_tag(
        self, name: str, version: str,
        key: str, value: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_bundle_version_tag(
        self, name: str, version: str, key: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- SkillBundle alias operations ---

    def set_skill_bundle_alias(
        self, name: str, alias: str, version: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_bundle_alias(
        self, name: str, alias: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)
```

For update fields, omitting a parameter leaves the stored value unchanged,
while passing `None` to a nullable field explicitly sets the field to
`null`.

## SDK convenience functions

The `mlflow.genai.skills` namespace provides convenience functions that
combine store operations, matching the pattern established by
`mlflow.genai.register_mcp_server()` in RFC-0004.

```python
import mlflow


def register_skill(
    *,
    name: str,
    version: str,
    display_name: str | None = None,
    description: str | None = None,
    source_type: str | None = None,
    source: str | None = None,
    subpath: str | None = None,
    content_path: str | None = None,
    content_digest: str | None = None,
) -> SkillVersion:
    """Register a skill version. Auto-creates the parent Skill if
    it does not exist. If content_path is provided, uploads the
    local directory to MLflow artifact storage and sets source_type
    and source automatically."""


def create_skill(
    *,
    name: str,
    display_name: str | None = None,
    description: str | None = None,
) -> Skill: ...


def get_skill(*, name: str) -> Skill: ...


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
    display_name: str | None = NOT_SET,
    description: str | None = NOT_SET,
) -> Skill: ...


def delete_skill(*, name: str) -> None: ...


def create_skill_version(
    *,
    name: str,
    version: str,
    display_name: str | None = None,
    source_type: str | None = None,
    source: str | None = None,
    subpath: str | None = None,
    content_digest: str | None = None,
) -> SkillVersion: ...


def get_skill_version(*, name: str, version: str) -> SkillVersion: ...


def get_skill_version_by_alias(*, name: str, alias: str) -> SkillVersion: ...


def get_latest_skill_version(*, name: str) -> SkillVersion: ...


def search_skill_versions(
    *,
    name: str,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[SkillVersion]: ...


def update_skill_version(
    *,
    name: str,
    version: str,
    display_name: str | None = NOT_SET,
    status: str | None = NOT_SET,
) -> SkillVersion: ...


def delete_skill_version(*, name: str, version: str) -> None: ...


def create_skill_bundle(
    *,
    name: str,
    display_name: str | None = None,
    description: str | None = None,
) -> SkillBundle: ...


def create_skill_bundle_version(
    *,
    name: str,
    version: str,
    display_name: str | None = None,
    skills: list[SkillMemberRef] | None = None,
    source_type: str | None = None,
    source: str | None = None,
    subpath: str | None = None,
    content_digest: str | None = None,
) -> SkillBundleVersion: ...


def get_skill_bundle(*, name: str) -> SkillBundle: ...


def search_skill_bundles(
    *,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[SkillBundle]: ...


def update_skill_bundle(
    *,
    name: str,
    display_name: str | None = NOT_SET,
    description: str | None = NOT_SET,
) -> SkillBundle: ...


def delete_skill_bundle(*, name: str) -> None: ...


def get_skill_bundle_version(
    *, name: str, version: str,
) -> SkillBundleVersion: ...


def get_skill_bundle_version_by_alias(
    *, name: str, alias: str,
) -> SkillBundleVersion: ...


def get_latest_skill_bundle_version(*, name: str) -> SkillBundleVersion: ...


def search_skill_bundle_versions(
    *,
    name: str,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[SkillBundleVersion]: ...


def update_skill_bundle_version(
    *,
    name: str,
    version: str,
    display_name: str | None = NOT_SET,
    status: str | None = NOT_SET,
) -> SkillBundleVersion: ...


def delete_skill_bundle_version(*, name: str, version: str) -> None: ...


def set_skill_tag(*, name: str, key: str, value: str) -> None: ...

def delete_skill_tag(*, name: str, key: str) -> None: ...

def set_skill_version_tag(*, name: str, version: str, key: str, value: str) -> None: ...

def delete_skill_version_tag(*, name: str, version: str, key: str) -> None: ...

def set_skill_alias(*, name: str, alias: str, version: str) -> None: ...

def delete_skill_alias(*, name: str, alias: str) -> None: ...

def set_skill_bundle_tag(*, name: str, key: str, value: str) -> None: ...

def delete_skill_bundle_tag(*, name: str, key: str) -> None: ...

def set_skill_bundle_version_tag(*, name: str, version: str, key: str, value: str) -> None: ...

def delete_skill_bundle_version_tag(*, name: str, version: str, key: str) -> None: ...

def set_skill_bundle_alias(*, name: str, alias: str, version: str) -> None: ...

def delete_skill_bundle_alias(*, name: str, alias: str) -> None: ...


def pull(
    *,
    name: str | None = None,
    bundle: str | None = None,
    version: str | None = None,
    alias: str | None = None,
    destination: str = ".",
) -> str:
    """Pull skill or bundle content from registered sources to a
    local directory. Specify name for a single skill or bundle
    for a skill bundle."""


# Example usage:
version = mlflow.genai.skills.register_skill(name="code-review", version="1.0.0", source_type="git", source="...")
servers = mlflow.genai.skills.search_skills(filter_string="status = 'active'")
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

### Skill endpoints

All paths relative to the logical skills router prefix.

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Create a skill |
| `GET` | `/` | Search skills |
| `GET` | `/{name}` | Get skill by name |
| `PATCH` | `/{name}` | Update skill fields |
| `DELETE` | `/{name}` | Hard-delete skill (cascades, subject to references) |
| `POST` | `/{name}/versions` | Create a skill version |
| `GET` | `/{name}/versions` | Search versions |
| `GET` | `/{name}/versions/{version}` | Get a specific version |
| `PATCH` | `/{name}/versions/{version}` | Update version |
| `DELETE` | `/{name}/versions/{version}` | Soft-delete a version (`status='deleted'`) |
| `POST` | `/{name}/tags` | Set a skill-level tag |
| `DELETE` | `/{name}/tags/{key}` | Delete a skill-level tag |
| `POST` | `/{name}/versions/{version}/tags` | Set a version-level tag |
| `DELETE` | `/{name}/versions/{version}/tags/{key}` | Delete a version tag |
| `POST` | `/{name}/aliases` | Set an alias |
| `GET` | `/{name}/aliases/{alias}` | Resolve alias to `SkillVersion` |
| `DELETE` | `/{name}/aliases/{alias}` | Delete an alias |

Similarly, skill bundle endpoints are exposed under both
`/api/3.0/mlflow/skill-bundles` and
`/ajax-api/3.0/mlflow/skill-bundles`.

### Skill bundle endpoints

All paths relative to the logical skill-bundles router prefix.

| Method | Path | Description |
|---|---|---|
| `POST` | `/` | Create a skill bundle |
| `GET` | `/` | Search skill bundles |
| `GET` | `/{name}` | Get bundle by name |
| `PATCH` | `/{name}` | Update bundle fields |
| `DELETE` | `/{name}` | Hard-delete bundle (cascades versions and memberships) |
| `POST` | `/{name}/versions` | Create a bundle version with members |
| `GET` | `/{name}/versions` | Search bundle versions |
| `GET` | `/{name}/versions/{version}` | Get a specific bundle version |
| `PATCH` | `/{name}/versions/{version}` | Update bundle version status |
| `DELETE` | `/{name}/versions/{version}` | Soft-delete a bundle version (`status='deleted'`) |
| `POST` | `/{name}/tags` | Set a bundle-level tag |
| `DELETE` | `/{name}/tags/{key}` | Delete a bundle-level tag |
| `POST` | `/{name}/versions/{version}/tags` | Set a bundle version tag |
| `DELETE` | `/{name}/versions/{version}/tags/{key}` | Delete a bundle version tag |
| `POST` | `/{name}/aliases` | Set a bundle alias |
| `GET` | `/{name}/aliases/{alias}` | Resolve bundle alias to version |
| `DELETE` | `/{name}/aliases/{alias}` | Delete a bundle alias |

### Pagination and filtering

Search endpoints use page-token-based pagination and `filter_string`
expressions following existing MLflow conventions.

**Skills and bundles:** `name LIKE '%review%'`, `status = 'active'`,
`tags.team = 'platform'`

**Versions (all entity types):** `status = 'active'`,
`source_type = 'git'`, `tags.approved = 'true'`

### Request and response models

Request models contain only the mutable fields; resource identifiers
come from path parameters:

```python
from pydantic import BaseModel, Field


class CreateSkillRequest(BaseModel):
    name: str
    display_name: str | None = None
    description: str | None = None


class UpdateSkillRequest(BaseModel):
    display_name: str | None = None
    description: str | None = None


class CreateSkillVersionRequest(BaseModel):
    version: str
    display_name: str | None = None
    source_type: str | None = None
    source: str | None = None
    subpath: str | None = None
    content_digest: str | None = None


class UpdateSkillVersionRequest(BaseModel):
    display_name: str | None = None
    status: str | None = None


class CreateSkillBundleRequest(BaseModel):
    name: str
    display_name: str | None = None
    description: str | None = None


class UpdateSkillBundleRequest(BaseModel):
    display_name: str | None = None
    description: str | None = None


class SkillMemberRefPayload(BaseModel):
    name: str
    version: str
    member_subpath: str | None = None


class CreateSkillBundleVersionRequest(BaseModel):
    version: str
    display_name: str | None = None
    skills: list[SkillMemberRefPayload] | None = None
    source_type: str | None = None
    source: str | None = None
    subpath: str | None = None
    content_digest: str | None = None


class UpdateSkillBundleVersionRequest(BaseModel):
    display_name: str | None = None
    status: str | None = None


class AliasResponse(BaseModel):
    alias: str
    version: str


class SetAliasRequest(BaseModel):
    alias: str
    version: str


class SetTagRequest(BaseModel):
    key: str
    value: str


class SkillVersionResponse(BaseModel):
    name: str
    version: str
    display_name: str | None = None
    source_type: str | None = None
    source: str | None = None
    subpath: str | None = None
    content_digest: str | None = None
    status: str = "draft"
    aliases: list[str] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class SkillResponse(BaseModel):
    name: str
    display_name: str | None = None
    description: str | None = None
    status: str | None = None
    latest_version: str | None = None
    aliases: list[AliasResponse] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class SkillBundleVersionResponse(BaseModel):
    name: str
    version: str
    display_name: str | None = None
    source_type: str | None = None
    source: str | None = None
    subpath: str | None = None
    content_digest: str | None = None
    status: str = "draft"
    skills: list[SkillMemberRefPayload] = Field(default_factory=list)
    aliases: list[str] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class SkillBundleResponse(BaseModel):
    name: str
    display_name: str | None = None
    description: str | None = None
    status: str | None = None
    latest_version: str | None = None
    aliases: list[AliasResponse] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

`Skill.aliases` is modeled as a `dict[str, str]` in the entity layer
for convenience, while REST responses expose aliases as
`list[AliasResponse]` to keep the payload shape explicit and
consistent with the MCP Server Registry (RFC-0004).

## Python SDK and CLI

The `mlflow.genai.skills` module exposes top-level functions delegating
to `MlflowClient`, with a 1:1 mapping to the store mixin methods
above. The `mlflow skills` CLI command group provides the same
operations from the command line:

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
| `mlflow skills install` | (direct install) | Download and place in harness directory |
| `mlflow skills install-bundle` | (package manager install) | Install bundle via package manager plugin |
| `mlflow skills create-bundle` | `create_skill_bundle()` | Create a skill bundle |
| `mlflow skills create-bundle-version` | `create_skill_bundle_version()` | Create a bundle version with members |
| `mlflow skills get-bundle` | `get_skill_bundle()` | Get bundle metadata |
| `mlflow skills search-bundles` | `search_skill_bundles()` | Search bundles |
| `mlflow skills search-bundle-versions` | `search_skill_bundle_versions()` | Search bundle versions |
| `mlflow skills set-bundle-alias` | `set_skill_bundle_alias()` | Set a bundle alias |
| `mlflow skills set-bundle-tag` | `set_skill_bundle_tag()` | Set a bundle-level tag |
| `mlflow skills set-bundle-version-tag` | `set_skill_bundle_version_tag()` | Set a bundle version tag |
| `mlflow skills update-bundle-version` | `update_skill_bundle_version()` | Update bundle version status |

**Existing `mlflow skills` CLI group.** MLflow already has an
`mlflow skills` CLI group (`mlflow/cli/skills.py`) with two
subcommands: `list` (list bundled Assistant skills) and `view`
(view details of a bundled skill). These inspect the skills that
ship with the MLflow installation (under `mlflow/assistant/skills/`),
not registry-managed skills. The registry subcommands above extend
this existing group. None of the registry subcommand names conflict
with `list` or `view`, so both sets coexist: `list`/`view` operate
on locally bundled skills, while `search`/`get`/`register`/`pull`
and the other registry subcommands operate on the server-side
registry.

## Package manager plugin interface

Package manager plugins are registered via Python entrypoints (group
`mlflow.skill_package_managers`), so third-party plugins can be
installed via `pip install` without modifying MLflow core.

### Plugin protocol

```python
class PackageManagerPlugin:
    def install_skill(
        self,
        name: str,
        local_path: str,
        harness: str | None = None,
        scope: str = "project",
    ) -> str:
        """Install a single skill from a local path.
        Returns the installed path.

        Args:
            name: skill name for manifest generation
            local_path: local directory with skill content
            harness: target harness (e.g., "claude-code", "cursor").
                If None, auto-detect from environment.
            scope: "project" (cwd) or "user" (home directory)
        """
        ...

    def install_bundle(
        self,
        bundle_name: str,
        member_paths: dict[str, str],
        harness: str | None = None,
        scope: str = "project",
    ) -> str:
        """Install a bundle of skills from local paths.
        member_paths maps skill names to local paths.
        Returns the installed path.

        Args:
            bundle_name: bundle name for manifest generation
            member_paths: {skill_name: local_path} for each member
            harness: target harness. If None, auto-detect.
            scope: "project" or "user"
        """
        ...

    def supported_harnesses(self) -> list[str]:
        """Return list of harness identifiers this plugin supports.
        E.g., ["claude-code", "cursor", "codex-cli", "copilot"]."""
        ...
```

### Entrypoint registration

```toml
# In the package manager plugin's pyproject.toml:
[project.entry-points."mlflow.skill_package_managers"]
apm = "apm_mlflow:ApmPlugin"
lola = "lola_mlflow:LolaPlugin"
```

### Source resolution flow

When `mlflow skills install-bundle` is invoked:

1. **Resolve:** MLflow calls `get_skill_bundle_version()` (or alias
   resolution) to obtain the bundle version and its member list.
2. **Pull:** For each member skill, MLflow pulls content to a local
   temporary directory using source-type-aware logic (git clone, OCI
   pull, ZIP download, MLflow artifact download). For Git-backed
   skills, the package manager can fetch directly from Git if it
   supports Git sources natively, avoiding a redundant local pull.
3. **Delegate:** MLflow passes the local paths to the configured
   package manager plugin via `install_bundle()`. The plugin handles
   harness-specific directory placement and manifest generation.
4. **Manifest:** MLflow writes `mlflow-skills-manifest.json` with
   installed registry coordinates for trace integration.

### Direct install flow

When `mlflow skills install` is invoked (no package manager needed):

1. **Resolve:** MLflow calls `get_skill_version()` (or alias
   resolution) to obtain the source pointer.
2. **Pull:** MLflow pulls content to a local temporary directory.
3. **Place:** MLflow copies the content to the appropriate
   harness-specific directory based on the `--harness` flag:
   - `claude-code`: `.claude/skills/{name}/`
   - `cursor`: `.cursor/skills/{name}/`
   - Other harnesses: configurable via settings
4. **Manifest:** MLflow writes `mlflow-skills-manifest.json`.

## Pull semantics details

**Source availability.** The registry stores source pointers but does
not cache or proxy content. If a source is unreachable or the content
has been deleted, pull fails with an error that surfaces the
underlying failure from the source system (e.g., Git clone failure,
OCI pull 404, HTTP download error, MLflow artifact download error).
Source availability is the publisher's responsibility. For assembled
bundle pulls, if one member's source is unavailable, the entire pull
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

## skill_context() span attributes

The `skill_context()` context manager creates a span with the
following attributes:

| Attribute | Value | Description |
|---|---|---|
| `mlflow.skill.name` | Skill name | Registry name of the active skill |
| `mlflow.skill.version` | Version string | Registered version |
| `mlflow.skill.workspace` | Workspace name | MLflow workspace (defaults to `"default"`) |

These three attributes form the `{workspace, name, version}`
coordinates that link the span back to a specific skill version in
the registry.

## SDK and CLI code examples

### Register skills from an OCI artifact with subpath

```python
import mlflow
from mlflow.genai.skills import SkillMemberRef

mlflow.genai.skills.register_skill(
    name="code-review",
    version="1.0.0",
    source_type="oci",
    source="ghcr.io/acme/agent-plugin:v1.0.0",
    subpath="skills/code-review",
)

mlflow.genai.skills.register_skill(
    name="test-coverage",
    version="2.1.0",
    source_type="oci",
    source="ghcr.io/acme/agent-plugin:v1.0.0",
    subpath="skills/test-coverage",
)

# Assembled bundle: each member has its own source
bundle_version = mlflow.genai.skills.create_skill_bundle_version(
    name="pr-workflow",
    version="1.0.0",
    skills=[
        SkillMemberRef(name="code-review", version="1.0.0"),
        SkillMemberRef(name="test-coverage", version="2.1.0"),
    ],
)

# Monolithic bundle from a single OCI image. Embedded member
# versions are registered without their own sources.
mlflow.genai.skills.register_skill(
    name="embedded-review",
    version="1.0.0",
)

bundle_version = mlflow.genai.skills.create_skill_bundle_version(
    name="pr-workflow-mono",
    version="1.0.0",
    source_type="oci",
    source="ghcr.io/acme/agent-plugin:v1.0.0",
    skills=[
        SkillMemberRef(name="embedded-review", version="1.0.0",
                       member_subpath="skills/embedded-review"),
    ],
)
```

### Discover and consume skills

```python
# Search for active skill versions
versions = mlflow.genai.skills.search_skill_versions(
    name="code-review",
    filter_string="status = 'active'",
)

# Search for active skill bundles
bundles = mlflow.genai.skills.search_skill_bundles(
    filter_string="status = 'active'",
)

# Get a specific version
version = mlflow.genai.skills.get_skill_version(
    name="code-review",
    version="1.0.0",
)
# version.source_type == "git"
# version.source == "https://github.com/acme/agent-skills.git@v1.0.0"
# version.subpath == "code-review"

# Resolve by alias
version = mlflow.genai.skills.get_skill_version_by_alias(
    name="code-review",
    alias="production",
)

# Get a bundle version and its pinned members
bundle_version = mlflow.genai.skills.get_skill_bundle_version(
    name="pr-workflow",
    version="1.0.0",
)
# bundle_version.skills == [SkillMemberRef(name="code-review", version="1.0.0"), ...]

# Resolve a bundle alias
bundle_version = mlflow.genai.skills.get_skill_bundle_version_by_alias(
    name="pr-workflow",
    alias="production",
)
```
