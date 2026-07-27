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
| `member_name` | `String(256)` | PK |
| `member_version` | `String(256)` | PK |
| `member_subpath` | `String(2048)` | nullable; member path inside bundle artifact |

FK: `(workspace, bundle_name, bundle_version)` references
`skill_bundle_versions`, CASCADE delete. A FK to `skill_versions`
enforces referential integrity with RESTRICT delete.

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
    latest_version: str | None = None  # read-only, shared latest-resolution rule
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

The `source` field contains the artifact URI resolved by MLflow's
artifact storage (for example,
`mlflow-artifacts:/skills/code-review/1.0.0/sha256-abc123/` when using
the artifact proxy, or a direct artifact-store URI otherwise).
`source_type="mlflow"` means "stored in MLflow-managed artifact
storage," not a specific URI scheme. Pull downloads the directory tree
from the artifact store. The MLflow UI can browse individual files
within a stored skill version when artifact proxying is enabled.

**Client-side upload flow.** Direct artifact storage is implemented by
the `register_skill(content_path=...)` SDK/CLI convenience path rather
than a new skill-registry upload endpoint:

1. The client validates the local directory and preflights that the
   path can be registered.
2. The client computes `content_digest` over a canonical representation
   of the directory. Regular files are ordered by normalized relative
   path; both paths and file contents participate in the digest.
   Symlinks are rejected, and empty directories are not preserved.
3. The client checks the target `(name, version)`. If the version
   already exists, registration fails with an error. This matches
   the MCP Server Registry behavior (`register_mcp_server()`).
4. The client uploads each file through MLflow's existing artifact APIs
   to a digest-qualified, version-specific artifact prefix.
5. After upload succeeds, the client creates the `SkillVersion` with
   `source_type="mlflow"`, the resolved artifact URI, and the computed
   digest.

The upload and registry write are not atomic. If version creation fails,
the client makes a best-effort attempt to delete the uploaded prefix when
the artifact backend supports deletion. Any remaining files are
unreferenced orphaned artifacts and may be removed by artifact-store
garbage collection. A concurrent writer can still win after preflight;
the losing client follows the same cleanup behavior.

**Version uniqueness.** The combination of `(name, version)` is unique
within a workspace. A skill version represents a single logical
version of a capability; `source_type` and `source` describe where to
find it but are not part of its identity.

**Content integrity.** The optional `content_digest` field stores a
digest of the skill content at registration time (e.g.,
`sha256:abc123...`). For `source_type="mlflow"`, the client computes
the digest before upload and stores it on the version; on pull, the
client recomputes the digest over the downloaded content and rejects
the result if it does not match, detecting out-of-band modification
of the underlying artifact store. For external source types (git, oci,
zip), `content_digest` is also client-supplied: for OCI sources, this is
the native image digest; for Git sources, a digest of the file contents
at the pinned commit; for ZIP sources, a digest of the archive. The
registry stores the digest but does not verify it on read; verification
is the consumer's responsibility.

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
    latest_version: str | None = None  # read-only, shared latest-resolution rule
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

A versioned snapshot of a skill bundle's membership. In this RFC, all
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
  authoritative source. Every source-less member must provide a
  membership `member_subpath` identifying where it lives inside the
  bundle artifact.
- **Assembled:** has individual member references. Each skill member
  has its own source. `pull` fetches members individually. If a skill
  member has no source, `pull` fails rather than producing a partial
  local bundle. For assembled bundles, `member_subpath` must be null
  because the member's own `source` and `subpath` identify its
  content.

The API rejects attempts to set `member_subpath` on a membership whose
member version has its own source. It also rejects a source-less member
of a monolithic bundle when `member_subpath` is missing or empty.

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
from mlflow.store.tracking import SEARCH_MAX_RESULTS_DEFAULT


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
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
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
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
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
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
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
        max_results: int = SEARCH_MAX_RESULTS_DEFAULT,
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

The alias name `latest` is reserved for both skills and skill bundles.
The corresponding `set_*_alias()` methods reject it. Alias lookup with
`latest` delegates to `get_latest_skill_version()` or
`get_latest_skill_bundle_version()` rather than reading a stored alias
row.

For update fields, omitting a parameter leaves the stored value unchanged,
while passing `None` to a nullable field explicitly sets the field to
`null`.

## SDK convenience functions

The `mlflow.genai` namespace provides convenience functions that
combine store operations, matching the top-level public SDK pattern
established by `mlflow.genai.register_mcp_server()` in RFC-0004.

```python
from dataclasses import dataclass

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
    it does not exist and otherwise reuses the existing parent. If the
    version already exists, an MlflowException is raised. This matches
    the MCP Server Registry behavior (register_mcp_server). If
    content_path is provided, the client uploads the files through
    existing MLflow artifact APIs and sets source_type, source, and
    content_digest. content_path is mutually exclusive with source_type,
    source, subpath, and content_digest."""


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
    bundle_name: str | None
    version: str | None
    skills: list[IntrospectedSkill]
    warnings: list[PluginImportWarning]


@dataclass
class PluginImportResult:
    bundle_version: SkillBundleVersion
    skill_versions: list[SkillVersion]
    warnings: list[PluginImportWarning]


@dataclass(frozen=True)
class InstalledSkill:
    registry_name: str
    harness_local_name: str
    installed_path: str


@dataclass
class PackageManagerInstallResult:
    installed_path: str
    skills: list[InstalledSkill]


@dataclass(frozen=True)
class MlflowSkillLockEntry:
    entity_type: str
    name: str
    version: str
    workspace: str
    package_manager: str
    harness: str
    scope: str


def introspect_bundle(
    *,
    source: str,
    plugin_format: str,
    source_type: str | None = None,
    subpath: str | None = None,
) -> PluginIntrospectionResult:
    """Inspect a local or remote plugin without modifying the registry."""


def import_bundle(
    *,
    source: str,
    plugin_format: str,
    bundle_name: str | None = None,
    version: str | None = None,
    source_type: str | None = None,
    subpath: str | None = None,
) -> PluginImportResult:
    """Import a plugin as a monolithic bundle.

    Fetches and inspects the plugin in the client environment, registers
    discovered skills, preserves the plugin source on the bundle version,
    and returns warnings for non-skill content that is included in the
    bundle but does not receive individual registry entries.
    """


def install_skill(
    *,
    name: str,
    harness: str,
    version: str | None = None,
    alias: str | None = None,
    package_manager: str | None = None,
    scope: str = "project",
    lock_file: str | None = None,
) -> PackageManagerInstallResult:
    """Resolve a skill and install it through a package manager plugin.
    The harness argument is required to keep behavior predictable
    across plugins. If lock_file is provided, record the exact resolved
    version and installation inputs for replay."""


def install_bundle(
    *,
    name: str,
    harness: str,
    version: str | None = None,
    alias: str | None = None,
    package_manager: str | None = None,
    scope: str = "project",
    lock_file: str | None = None,
) -> PackageManagerInstallResult:
    """Resolve a bundle and install it through a package manager plugin.
    For monolithic bundles, non-skill content is included in the
    installed artifact. The harness argument is required to keep
    behavior predictable across plugins. If lock_file is provided,
    record the exact resolved version and installation inputs for
    replay."""


def install_from_lock(
    *, lock_file: str = "mlflow-skills.lock",
) -> list[PackageManagerInstallResult]:
    """Replay exact skill and bundle versions from an MLflow resolution
    lock using the recorded package manager, harness, and scope."""


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
version = mlflow.genai.register_skill(name="code-review", version="1.0.0", source_type="git", source="...")
servers = mlflow.genai.search_skills(filter_string="status = 'active'")
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

There is no skill-registry content-upload endpoint. The client-side
`register_skill(content_path=...)` helper uploads through existing
MLflow artifact APIs, then uses the version-creation endpoint below to
store the resulting artifact URI and digest.

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

The `mlflow.genai` module exposes the public registry functions,
delegating to `MlflowClient`, plus client-side import, pull, and
package-manager installation operations that compose those registry
functions. Skill-specific entity and request types are also re-exported
from `mlflow.genai`. The `mlflow skills-registry` CLI command group
provides the same operations from the command line:

| CLI subcommand | SDK function | Description |
|---|---|---|
| `mlflow skills-registry register` | `register_skill()` | Register a skill version (auto-creates parent) |
| `mlflow skills-registry get` | `get_skill()` | Get skill metadata |
| `mlflow skills-registry search` | `search_skills()` | Search skills |
| `mlflow skills-registry get-version` | `get_skill_version()` | Get a specific version |
| `mlflow skills-registry update-version` | `update_skill_version()` | Update version status |
| `mlflow skills-registry set-alias` | `set_skill_alias()` | Set a version alias |
| `mlflow skills-registry set-tag` | `set_skill_tag()` | Set a tag |
| `mlflow skills-registry pull` | `pull()` | Pull content to local filesystem |
| `mlflow skills-registry install` | `install_skill()` | Install one skill through a package manager plugin |
| `mlflow skills-registry bundles install` | `install_bundle()` | Install a bundle through a package manager plugin |
| `mlflow skills-registry install --from-lock` | `install_from_lock()` | Replay exact registry versions from an MLflow resolution lock |
| `mlflow skills-registry bundles create` | `create_skill_bundle()` | Create a skill bundle |
| `mlflow skills-registry bundles create-version` | `create_skill_bundle_version()` | Create a bundle version with members |
| `mlflow skills-registry bundles get` | `get_skill_bundle()` | Get bundle metadata |
| `mlflow skills-registry bundles search` | `search_skill_bundles()` | Search bundles |
| `mlflow skills-registry bundles search-versions` | `search_skill_bundle_versions()` | Search bundle versions |
| `mlflow skills-registry bundles set-alias` | `set_skill_bundle_alias()` | Set a bundle alias |
| `mlflow skills-registry bundles set-tag` | `set_skill_bundle_tag()` | Set a bundle-level tag |
| `mlflow skills-registry bundles set-version-tag` | `set_skill_bundle_version_tag()` | Set a bundle version tag |
| `mlflow skills-registry bundles update-version` | `update_skill_bundle_version()` | Update bundle version status |
| `mlflow skills-registry bundles introspect` | `introspect_bundle()` | Preview a local or remote plugin without registry writes |
| `mlflow skills-registry bundles import` | `import_bundle()` | Import a plugin as a monolithic bundle |
| `mlflow skills-registry list-package-managers` | `list_package_managers()` | List installed package manager plugins and their supported harnesses |

**Relationship to existing `mlflow skills` CLI group.** MLflow already
has an `mlflow skills` CLI group (`mlflow/cli/skills.py`) with two
subcommands: `list` (list bundled Assistant skills) and `view`
(view details of a bundled skill). These inspect the skills that
ship with the MLflow installation (under `mlflow/assistant/skills/`),
not registry-managed skills. The registry uses a separate
`mlflow skills-registry` group to avoid confusion between the two.

## Plugin import

Plugin import is implemented in the SDK and CLI layer. There is no
dedicated REST import endpoint: the client fetches and inspects the
source locally, then calls the existing skill and bundle creation APIs.
The registry server does not fetch user-supplied plugin URLs.

### Read-only preview

`introspect_bundle()` and `mlflow skills-registry bundles introspect` run the same plugin
discovery used by import but do not create or modify registry records.
They accept either a local path or a remote Git, OCI, ZIP, or MLflow
artifact source and return the discovered skill names and paths,
available plugin name and version metadata, and warnings for
unregistered non-skill content. A local path must not specify
`source_type`; remote
sources use an explicit source type or the same unambiguous syntax
inference as import. Introspection does not require the plugin to provide
a name or version because those values are only required when importing.

### Supported input format

This RFC supports the Claude Code plugin layout. The caller passes
`plugin_format="claude-code"`; automatic format detection and additional
input formats are follow-up work. The importer:

1. Fetches the Git, OCI, ZIP, or MLflow artifact source using the same
   source-type-aware logic as `pull`.
2. Applies `subpath`, when provided, to select the plugin root.
3. Reads `.claude-plugin/plugin.json` when present to obtain supported
   plugin metadata such as name and version. Explicit `bundle_name` and
   `version` arguments take precedence.
4. Discovers skill directories under `skills/` that contain a SKILL.md
   entry point. The SKILL.md name is used when present; otherwise the
   directory name is used.
5. Detects non-skill plugin content, including subagents, hooks, and MCP
   configuration, for warning purposes only.

The resulting bundle name and version must be available after explicit
arguments and plugin metadata are considered. The version must be a
valid semantic version. When `source_type` is omitted, the client infers
it from the source syntax and fails if the source type is ambiguous.

### Registration behavior

For each discovered skill, the importer creates a `SkillVersion` whose
version defaults to the bundle version and whose `source_type`, `source`,
and `subpath` are null. The importer records the skill's plugin-relative
directory as `SkillMemberRef.member_subpath`.

After registering the embedded skills, the importer creates one
monolithic `SkillBundleVersion` with the original `source_type`,
`source`, and `subpath`, plus member references for all discovered
skills. This preserves a pullable link to the complete original plugin
while keeping registry entries limited to skills.

The import fails if no skills are discovered. It also preflights all
target `(name, version)` pairs and fails if any skill or bundle version
already exists. It never overwrites or reuses an existing version. A
caller resolves a conflict by choosing a different bundle version or
renaming the conflicting skill before import.

### Warnings and result

Subagents, hooks, MCP configurations, and unrecognized content remain
in the plugin artifact but are not registered. Each discovered skipped
category produces a `PluginImportWarning` containing its category,
path, and an explanation that this RFC does not create registry entries
for non-skill content (though the content remains in the bundle). The CLI
prints these warnings after registration. The SDK returns them together
with the created bundle and skill versions in `PluginImportResult`.

Import translates an existing plugin into MLflow's registry
representation, creating registry entries for discovered skills while
preserving the complete plugin source. It does not install the plugin,
generate a downstream manifest, or translate an MLflow bundle into a
downstream bundle format.

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

## Package manager plugin interface

Package manager plugins are registered via Python entrypoints (group
`mlflow.skill_package_managers`), so third-party plugins can be
installed via `pip install` without modifying MLflow core.

These plugins receive resolved skills or bundle content
(which may include non-skill content in monolithic bundles). They
install the content using an existing package manager. The package
manager handles placement of all content, including non-skill files
that do not have individual registry entries. It returns the actual
harness-local name of every installed skill so MLflow can write an
accurate trace manifest even when the package manager renames or prefixes
skills. The result must contain exactly one `InstalledSkill` for every
requested registry skill; missing or duplicate mappings fail the install
before MLflow writes its trace manifest or resolution lock.

Both `mlflow skills-registry install` and `mlflow skills-registry
bundles install` require a package manager plugin. The caller can select
a plugin explicitly, or MLflow uses the configured default. If no plugin
is selected or available, installation fails with guidance to install or
configure one; `mlflow skills-registry pull` remains available without a
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

When `mlflow skills-registry bundles install` is invoked:

1. **Resolve:** MLflow calls `get_skill_bundle_version()` (or alias
   resolution) to obtain the bundle version and its member list.
2. **Materialize member paths:**
   - For an assembled bundle, MLflow pulls each member skill to its own
     subdirectory under `.mlflow-skills/` using source-type-aware logic (Git clone,
     OCI pull, ZIP download, or MLflow artifact download).
   - For a monolithic bundle, MLflow pulls the bundle-level source once
     using the same source-type-aware logic and retains the complete root
     as `bundle_path`, including opaque non-skill content. For each
     member, it resolves a local path by joining the pulled bundle root
     with `member_subpath`. Every monolithic member must provide a
     non-empty `member_subpath`; installation fails if the path is missing,
     escapes the pulled bundle root after normalization, or does not
     contain the embedded skill.
3. **Delegate:** MLflow passes `member_paths` and, for a monolithic
   bundle, `bundle_path` to the configured package manager plugin via
   `install_bundle()`. The plugin installs the complete monolithic bundle
   or the assembled skills and returns each skill's harness-local name.
4. **Manifest:** MLflow writes `mlflow-skills-manifest.json`, keyed by
   the returned harness-local names and populated with the corresponding
   registry coordinates.
5. **Resolution lock:** If `lock_file` was supplied, MLflow atomically
   updates it with the exact resolved bundle version and installation
   inputs after the install and manifest write succeed.

### Single-skill installation flow

When `mlflow skills-registry install` is invoked:

1. **Resolve:** MLflow calls `get_skill_version()`, alias resolution, or
   latest resolution to obtain the registered source pointer. `version`
   and `alias` are mutually exclusive; omitting both selects the
   system-defined latest version.
2. **Pull:** MLflow pulls the skill content to a local temporary
   directory using the same source-type-aware logic as `pull`.
3. **Delegate:** MLflow passes the skill name and local path to the
   configured package manager plugin via `install_skill()`. The plugin
   owns harness-specific behavior, scope handling, directory placement,
   naming, and any generated package-manager or harness metadata. An
   explicit harness selection from the caller is passed through to the
   plugin, which returns the actual harness-local skill name.
4. **Manifest:** After the plugin reports success, MLflow writes or
   updates `mlflow-skills-manifest.json` under the returned harness-local
   name with the resolved registry coordinates.
5. **Resolution lock:** If `lock_file` was supplied, MLflow atomically
   updates it with the exact resolved skill version and installation
   inputs after the install and manifest write succeed.

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
| `mlflow.skill.workspace` | Workspace name | Resolved from the install manifest, falling back to the current tracking URI's workspace context |

These three attributes form the `{workspace, name, version}`
coordinates that link the span back to a specific skill version in
the registry.

## Automatic trace instrumentation

Automatic instrumentation uses the install-time
`mlflow-skills-manifest.json` to map harness-local skill invocations to
registered skill coordinates. This RFC implements this behavior in the
Claude Code autologger. The manifest format is harness-neutral so other
harness integrations can adopt the same contract later.

### Manifest writing and discovery

Installation commands write or update the manifest after all requested
skills have been installed successfully. Each entry is keyed by the
harness-local skill name returned by the package manager plugin and
contains the registered `workspace`, `name`, and resolved `version`.
Aliases are resolved before the manifest is written and are not stored
in place of versions.

Project-scoped installation writes the manifest at the project root.
User-scoped installation writes it in the MLflow user configuration
directory. Project entries take precedence over user entries with the
same harness-local skill name.

For a monolithic bundle, installation writes an entry for every
registered embedded skill resolved through its `member_subpath`. For an
assembled bundle, it writes an entry for every installed member skill.
The bundle itself does not produce a SKILL span because tracing is at
the invoked-skill level.

### Claude Code invocation matching

The Claude Code autologger matches harness skill invocations
against manifest entries by skill name. When a match is found, it
creates a span with:

- span type `SKILL`
- span name equal to the harness-local skill name
- `mlflow.skill.name`, `mlflow.skill.version`, and
  `mlflow.skill.workspace` attributes from the manifest

LLM and tool spans produced while the skill is active become children
of the SKILL span.

If a matching SKILL span with the same registry coordinates is already
active because application code used `mlflow.skill_context()`, the
autologger reuses that active context and does not create a duplicate
SKILL span.

### Failure behavior

Automatic instrumentation does not contact the registry during skill
invocation and does not add runtime latency or create a dependency on
registry availability.

A missing manifest, malformed manifest, or unmatched skill name never
interrupts the agent run or other autologging; it only prevents
creation of a registry-linked SKILL span for the affected invocation.
Skills copied into a harness without an MLflow installation command
have no manifest entry and are not linked automatically; callers can
still use `mlflow.skill_context()` manually.

## SDK and CLI code examples

### Register skills from an OCI artifact with subpath

```python
import mlflow
from mlflow.genai import SkillMemberRef

mlflow.genai.register_skill(
    name="code-review",
    version="1.0.0",
    source_type="oci",
    source="ghcr.io/acme/agent-plugin:v1.0.0",
    subpath="skills/code-review",
)

mlflow.genai.register_skill(
    name="test-coverage",
    version="2.1.0",
    source_type="oci",
    source="ghcr.io/acme/agent-plugin:v1.0.0",
    subpath="skills/test-coverage",
)

# Assembled bundle: each member has its own source
bundle_version = mlflow.genai.create_skill_bundle_version(
    name="pr-workflow",
    version="1.0.0",
    skills=[
        SkillMemberRef(name="code-review", version="1.0.0"),
        SkillMemberRef(name="test-coverage", version="2.1.0"),
    ],
)

# Monolithic bundle from a single OCI image. Embedded member
# versions are registered without their own sources.
mlflow.genai.register_skill(
    name="embedded-review",
    version="1.0.0",
)

bundle_version = mlflow.genai.create_skill_bundle_version(
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
versions = mlflow.genai.search_skill_versions(
    name="code-review",
    filter_string="status = 'active'",
)

# Search for active skill bundles
bundles = mlflow.genai.search_skill_bundles(
    filter_string="status = 'active'",
)

# Get a specific version
version = mlflow.genai.get_skill_version(
    name="code-review",
    version="1.0.0",
)
# version.source_type == "git"
# version.source == "https://github.com/acme/agent-skills.git@v1.0.0"
# version.subpath == "code-review"

# Resolve by alias
version = mlflow.genai.get_skill_version_by_alias(
    name="code-review",
    alias="production",
)

# Get a bundle version and its pinned members
bundle_version = mlflow.genai.get_skill_bundle_version(
    name="pr-workflow",
    version="1.0.0",
)
# bundle_version.skills == [SkillMemberRef(name="code-review", version="1.0.0"), ...]

# Resolve a bundle alias
bundle_version = mlflow.genai.get_skill_bundle_version_by_alias(
    name="pr-workflow",
    alias="production",
)
```
