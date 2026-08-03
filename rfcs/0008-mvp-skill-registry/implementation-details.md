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
| `name` | `String(256)` | PK |
| `display_name` | `String(256)` | mutable human-readable label; UI falls back to `name` when null |
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
| `version` | `Integer` | PK, server-assigned monotonic integer |
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

**Version ordering**: versions are monotonic integers assigned by
the server. Each new version for a given `(workspace, name)` receives
the next integer. Ordering is a simple integer comparison.

**Index**: `ix_skill_versions_latest_lookup` on `(workspace, name,
status, version)` supports latest-resolution lookups.

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
| `version` | `Integer` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `alias` | `String(256)` | PK |
| `version` | `Integer` | target version |

### `skill_bundles`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, default `'default'` |
| `name` | `String(256)` | PK |
| `display_name` | `String(256)` | mutable human-readable label; UI falls back to `name` when null |
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
| `version` | `Integer` | PK, server-assigned monotonic integer |
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
Version ordering and index follow the same pattern as
`skill_versions`.

### `skill_bundle_version_members`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK |
| `bundle_name` | `String(256)` | PK, FK to `skill_bundle_versions` |
| `bundle_version` | `Integer` | PK, FK to `skill_bundle_versions` |
| `member_name` | `String(256)` | PK |
| `member_version` | `Integer` | PK |
| `member_subpath` | `String(2048)` | nullable; parsed from `#subpath` fragment of member URI |

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
| `version` | `Integer` | PK, FK |
| `key` | `String(256)` | PK |
| `value` | `Text` | |

### `skill_bundle_aliases`

| Column | Type | Notes |
|--------|------|-------|
| `workspace` | `String(63)` | PK, FK |
| `name` | `String(256)` | PK, FK |
| `alias` | `String(256)` | PK |
| `version` | `Integer` | target bundle version |

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
  may retain enough metadata to explain historical bundle
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
    aliases: dict[str, int] = field(default_factory=dict)  # read-only; populated from skill_aliases table, e.g. {"production": 2}
    latest_version: int | None = None  # read-only, shared latest-resolution rule
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

| Field | Type | Description |
|---|---|---|
| `name` | `str` | Stable logical asset name, unique within a workspace |
| `display_name` | `str` | Mutable human-readable label for UI display. When null, the UI falls back to `name` |
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
| `version` | `int` | Server-assigned monotonic integer. Each new version receives the next integer |
| `display_name` | `str` | Mutable human-readable label for UI display. When null, the UI falls back to `name` |
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
skills/code-review/1/
  SKILL.md
  scripts/analyze.sh
  scripts/lint-config.json
  reference/style-guide.md
```

The `source` field contains the artifact URI resolved by MLflow's
artifact storage (for example,
`mlflow-artifacts:/skills/code-review/1/sha256-abc123/` when using
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
3. The client checks that the target skill name can accept a new
   version.
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
    aliases: dict[str, int] = field(default_factory=dict)  # read-only; populated from skill_bundle_aliases table
    latest_version: int | None = None  # read-only, shared latest-resolution rule
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

`SkillBundle.status` is read-only and uses the same parent-resolved
version rule as `Skill`: highest active version number if present,
otherwise highest non-deleted non-active version number. Latest
version resolution follows the same fallback.

### SkillBundleVersion entity

A versioned snapshot of a skill bundle's membership. In this RFC, all
members are skills.

Bundle members are referenced by URI string rather than a separate
data class. The URI format follows MLflow's `models:/name/version`
convention:

- `skills:/name/version` pins a specific version
- `skills:/name@alias` resolves through an alias
- `skills:/name/version#subpath` identifies an embedded skill inside
  a monolithic bundle artifact (subpath relative to the bundle root)

```python
@dataclass
class SkillBundleVersion:
    name: str
    version: int
    display_name: str | None = None
    source_type: SkillSourceType | None = None
    source: str | None = None
    subpath: str | None = None
    content_digest: str | None = None
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
  authoritative source. Every source-less member must include a
  `#subpath` fragment in its URI identifying where it lives inside
  the bundle artifact (e.g., `skills:/name/1#skills/name`).
- **Assembled:** has individual member references. Each skill member
  has its own source. `pull` fetches members individually. If a skill
  member has no source, `pull` fails rather than producing a partial
  local bundle. For assembled bundles, member URIs must not include
  a `#subpath` fragment because the member's own `source` and
  `subpath` identify its content.

The API rejects a member URI with a `#subpath` fragment when the
member version has its own source. It also rejects a source-less member
of a monolithic bundle when the URI lacks a `#subpath` fragment.

**Immutability contract.** The member list and source fields of a
bundle version are immutable after creation. To change the set of
members or source pointer, register a new bundle version. Mutable
fields (`display_name`, `status`, `tags`) can be updated independently.

Correctness of the artifact layout is the publisher's responsibility;
the registry does not validate artifact contents at registration time.

A member can appear in multiple bundles and multiple bundle versions.
Membership is at the version level, so a bundle version is a
reproducible snapshot of "these specific skill versions work together."

### Skill URI format

Skill URIs are used for CLI target identification and bundle
member lists, following the `models:/name/version` convention
established by MLflow's Model Registry. The Python SDK and REST
API continue to use separate `name`, `version`, and `alias`
parameters for primary resource identification.

| Pattern | Meaning | Example |
|---------|---------|---------|
| `skills:/name` | Identify a skill (name only) | `skills:/code-review` |
| `skills:/name/version` | Pin a specific version | `skills:/code-review/1` |
| `skills:/name@alias` | Resolve through an alias | `skills:/code-review@production` |
| `skills:/name/version#subpath` | Embedded skill inside a monolithic bundle | `skills:/review/1#skills/review` |

In the CLI, every command that targets a skill or bundle accepts a
`--skill-uri` flag (parallel to `--model-uri` in `mlflow models`).
In bundle member lists, URIs appear as plain strings in `list[str]`.
The server parses the URI into its constituent fields (`member_name`,
`member_version`, `member_subpath`) for storage and validation. Alias
URIs are resolved to a concrete version at the time of the API call.

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
        display_name: str | None = None,
        source_type: str | None = None,
        source: str | None = None,
        subpath: str | None = None,
        content_digest: str | None = None,
        status: str = "draft",
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_version(
        self, name: str, version: int,
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
        version: int,
        display_name: str | None = NOT_SET,
        status: SkillStatus | None = NOT_SET,
    ) -> SkillVersion:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_version(
        self, name: str, version: int,
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
        self, name: str, version: int,
        key: str, value: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_version_tag(
        self, name: str, version: int, key: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- Skill alias operations ---

    def set_skill_alias(
        self, name: str, alias: str, version: int,
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
        display_name: str | None = None,
        skills: list[str] | None = None,
        source_type: str | None = None,
        source: str | None = None,
        subpath: str | None = None,
        content_digest: str | None = None,
    ) -> SkillBundleVersion:
        raise NotImplementedError(self.__class__.__name__)

    def get_skill_bundle_version(
        self, name: str, version: int,
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
        version: int,
        display_name: str | None = NOT_SET,
        status: SkillStatus | None = NOT_SET,
    ) -> SkillBundleVersion:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_bundle_version(
        self, name: str, version: int,
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
        self, name: str, version: int,
        key: str, value: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    def delete_skill_bundle_version_tag(
        self, name: str, version: int, key: str,
    ) -> None:
        raise NotImplementedError(self.__class__.__name__)

    # --- SkillBundle alias operations ---

    def set_skill_bundle_alias(
        self, name: str, alias: str, version: int,
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
    name: str | None = None,
    display_name: str | None = None,
    source_type: str | None = None,
    source: str | None = None,
    subpath: str | None = None,
    status: str = "draft",
    content_path: str | None = None,
    content_digest: str | None = None,
) -> SkillVersion:
    """Register a skill version. The server assigns the next
    monotonic integer version. Auto-creates the parent Skill if
    it does not exist (with null display_name and description) and
    otherwise reuses the existing parent. To set parent-level
    metadata, use create_skill() before registering versions or
    update_skill() afterward. This matches the MCP Server Registry
    behavior (register_mcp_server). If name is omitted and source
    is provided, the server fetches the source and extracts the
    name from the skill's SKILL.md entry point. If name is omitted
    and content_path is provided, the client extracts the name from
    the local SKILL.md before uploading. If source_type is omitted
    but source is provided, the server infers the type from the URL
    (.git suffix or git:// = git, oci:// = oci, .zip = zip); if
    inference fails, an error asks the caller to specify source_type
    explicitly. If content_path is provided, the client uploads the
    files through existing MLflow artifact APIs and sets source_type,
    source, and content_digest. content_path is mutually exclusive
    with source_type, source, subpath, and content_digest."""


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
    display_name: str | None = None,
    source_type: str | None = None,
    source: str | None = None,
    subpath: str | None = None,
    content_digest: str | None = None,
) -> SkillVersion: ...


def get_skill_version(*, name: str, version: int) -> SkillVersion: ...


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
    version: int,
    display_name: str | None = NOT_SET,
    status: str | None = NOT_SET,
) -> SkillVersion: ...


def delete_skill_version(*, name: str, version: int) -> None: ...


def create_skill_bundle(
    *,
    name: str,
    display_name: str | None = None,
    description: str | None = None,
) -> SkillBundle: ...


def create_skill_bundle_version(
    *,
    name: str,
    display_name: str | None = None,
    skills: list[str] | None = None,
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
    *, name: str, version: int,
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
    version: int,
    display_name: str | None = NOT_SET,
    status: str | None = NOT_SET,
) -> SkillBundleVersion: ...


def delete_skill_bundle_version(*, name: str, version: int) -> None: ...


def set_skill_tag(*, name: str, key: str, value: str) -> None: ...

def delete_skill_tag(*, name: str, key: str) -> None: ...

def set_skill_version_tag(*, name: str, version: int, key: str, value: str) -> None: ...

def delete_skill_version_tag(*, name: str, version: int, key: str) -> None: ...

def set_skill_alias(*, name: str, alias: str, version: int) -> None: ...

def delete_skill_alias(*, name: str, alias: str) -> None: ...

def set_skill_bundle_tag(*, name: str, key: str, value: str) -> None: ...

def delete_skill_bundle_tag(*, name: str, key: str) -> None: ...

def set_skill_bundle_version_tag(*, name: str, version: int, key: str, value: str) -> None: ...

def delete_skill_bundle_version_tag(*, name: str, version: int, key: str) -> None: ...

def set_skill_bundle_alias(*, name: str, alias: str, version: int) -> None: ...

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
    skills: list[IntrospectedSkill]
    warnings: list[PluginImportWarning]


@dataclass
class PluginImportResult:
    bundle_version: SkillBundleVersion
    skill_versions: list[SkillVersion]
    warnings: list[PluginImportWarning]


def introspect_bundle(
    *,
    source: str,
    source_type: str | None = None,
    subpath: str | None = None,
) -> PluginIntrospectionResult:
    """Inspect a local or remote plugin without modifying the registry."""


def import_bundle(
    *,
    source: str,
    bundle_name: str | None = None,
    source_type: str | None = None,
    subpath: str | None = None,
) -> PluginImportResult:
    """Import a plugin as a monolithic bundle.

    Discovers skill directories (subdirectories containing a SKILL.md
    entry point), registers them, preserves the plugin source on the
    bundle version, and returns warnings for non-skill content that is
    included in the bundle but does not receive individual registry
    entries. If bundle_name is omitted, it is inferred from the source
    directory name. If source_type is omitted, it is inferred from the
    URL scheme.
    """


def pull(
    *,
    name: str | None = None,
    bundle: str | None = None,
    version: int | None = None,
    alias: str | None = None,
    destination: str = ".",
) -> str:
    """Pull skill or bundle content from registered sources to a
    local directory. Specify name for a single skill or bundle
    for a skill bundle."""


# Example usage:
version = mlflow.genai.register_skill(name="code-review", source_type="git", source="...")
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

**Field inference.** The server infers optional fields from source
content when possible, so the simplest call requires only what cannot
be derived. When `name` is omitted from a registration request, the
server fetches the source and extracts the name from the skill's
SKILL.md entry point. When `source_type` is omitted but `source` is
provided, the server infers the type from the URL scheme. This keeps
SDKs as thin REST wrappers, avoids reimplementing inference in every
language, and prepares for future server-side content inspection
(e.g., signature verification).

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
| `POST` | `/register` | Register a skill version (name optional; server infers from source when omitted, auto-creates parent) |
| `GET` | `/{name}` | Get skill by name |
| `PATCH` | `/{name}` | Update skill fields |
| `DELETE` | `/{name}` | Hard-delete skill (cascades, subject to references) |
| `POST` | `/{name}/versions` | Create a skill version (name from path, no inference) |
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

**Skills:** `name LIKE '%review%'`, `status = 'active'`,
`tags.team = 'platform'`

**Bundles:** Same as skills, plus `member_name = 'code-review'` to
find bundles that include a given skill (reverse membership lookup)

**Versions (all entity types):** `status = 'active'`,
`source_type = 'git'`, `tags.approved = 'true'`

### Request and response models

Request models contain only the mutable fields; resource identifiers
come from path parameters, with one exception: `POST /register`
accepts `name` in the request body (optional, inferred from source
content when omitted) rather than the path. This parallels RFC-0004's
top-level `register_mcp_server()` pattern:

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
    name: str | None = None  # optional for POST /register (inferred from source when omitted); ignored for POST /{name}/versions (name from path)
    display_name: str | None = None
    source_type: str | None = None
    source: str | None = None
    subpath: str | None = None
    content_digest: str | None = None
    status: str = "draft"


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


class CreateSkillBundleVersionRequest(BaseModel):
    display_name: str | None = None
    skills: list[str] | None = None
    source_type: str | None = None
    source: str | None = None
    subpath: str | None = None
    content_digest: str | None = None


class UpdateSkillBundleVersionRequest(BaseModel):
    display_name: str | None = None
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
    latest_version: int | None = None
    aliases: list[AliasResponse] = Field(default_factory=list)
    tags: dict[str, str] = Field(default_factory=dict)
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None


class SkillBundleVersionResponse(BaseModel):
    name: str
    version: int
    display_name: str | None = None
    source_type: str | None = None
    source: str | None = None
    subpath: str | None = None
    content_digest: str | None = None
    status: str = "draft"
    skills: list[str] = Field(default_factory=list)
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

## Python SDK and CLI

The `mlflow.genai` module exposes the public registry functions,
delegating to `MlflowClient`, plus client-side import and pull
operations that compose those registry functions. Skill-specific entity and request types are also re-exported
from `mlflow.genai`. The `mlflow skills` CLI command group
provides the same operations from the command line:

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
| `mlflow skills bundles create` | `create_skill_bundle()` | Create a skill bundle |
| `mlflow skills bundles create-version` | `create_skill_bundle_version()` | Create a bundle version with members |
| `mlflow skills bundles get` | `get_skill_bundle()` | Get bundle metadata |
| `mlflow skills bundles search` | `search_skill_bundles()` | Search bundles |
| `mlflow skills bundles search-versions` | `search_skill_bundle_versions()` | Search bundle versions |
| `mlflow skills bundles set-alias` | `set_skill_bundle_alias()` | Set a bundle alias |
| `mlflow skills bundles set-tag` | `set_skill_bundle_tag()` | Set a bundle-level tag |
| `mlflow skills bundles set-version-tag` | `set_skill_bundle_version_tag()` | Set a bundle version tag |
| `mlflow skills bundles update-version` | `update_skill_bundle_version()` | Update bundle version status |
| `mlflow skills bundles introspect` | `introspect_bundle()` | Preview a local or remote plugin without registry writes |
| `mlflow skills bundles import` | `import_bundle()` | Import a plugin as a monolithic bundle |

**Relationship to existing `mlflow skills` subcommands.** MLflow already
has `mlflow skills list` and `mlflow skills view` subcommands
(`mlflow/cli/skills.py`) that inspect the bundled Assistant skills
shipping with the MLflow installation (under `mlflow/assistant/skills/`).
The registry's subcommands use different names (e.g., `register`,
`search`, `pull`), so there is no direct conflict.

## Plugin import

Plugin import is implemented in the SDK and CLI layer. There is no
dedicated REST import endpoint: the client fetches and inspects the
source locally, then calls the existing skill and bundle creation APIs.
The registry server does not fetch user-supplied plugin URLs.

### Read-only preview

`introspect_bundle()` and `mlflow skills bundles introspect` run the same plugin
discovery used by import but do not create or modify registry records.
They accept either a local path or a remote Git, OCI, ZIP, or MLflow
artifact source and return the discovered skill names and paths,
available plugin name metadata, and warnings for unregistered
non-skill content. A local path must not specify `source_type`;
remote sources use an explicit source type or the same unambiguous
syntax inference as import.

### Supported input format

Import expects a standard layout: a directory tree where each skill is
a subdirectory containing a SKILL.md entry point. This layout is used
by Claude Code and other harnesses. The importer:

1. Fetches the Git, OCI, ZIP, or MLflow artifact source using the same
   source-type-aware logic as `pull`.
2. Applies `subpath`, when provided, to select the plugin root.
3. Reads `.claude-plugin/plugin.json` when present to obtain supported
   plugin metadata such as name. An explicit `bundle_name` argument
   takes precedence. Version is server-assigned at import time.
4. Discovers skill directories that contain a SKILL.md entry point.
   The SKILL.md name is used when present; otherwise the directory
   name is used.
5. Detects non-skill content for warning purposes only.

The resulting bundle name must be available after explicit
arguments and plugin metadata are considered. The version is
server-assigned. When `source_type` is omitted, the client infers
it from the source syntax and fails if the source type is ambiguous.

### Registration behavior

For each discovered skill, the importer creates a `SkillVersion` whose
version is server-assigned and whose `source_type`, `source`,
and `subpath` are null. The importer records the skill's plugin-relative
directory as the `#subpath` fragment in the member URI (e.g.,
`skills:/embedded-review/1#skills/embedded-review`).

After registering the embedded skills, the importer creates one
monolithic `SkillBundleVersion` with the original `source_type`,
`source`, and `subpath`, plus member references for all discovered
skills. This preserves a pullable link to the complete original plugin
while keeping registry entries limited to skills.

The import fails if no skills are discovered. Since versions are
server-assigned, import always creates new versions and does not
conflict with existing version numbers. A name conflict (an existing
skill with the same name) is resolved by the server creating the
next version for that skill.

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
translate an MLflow bundle into a downstream bundle format.

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

## SDK and CLI code examples

### Register skills from an OCI artifact with subpath

```python
import mlflow

mlflow.genai.register_skill(
    name="code-review",
    source_type="oci",
    source="ghcr.io/acme/agent-plugin:v1.0.0",
    subpath="skills/code-review",
)

mlflow.genai.register_skill(
    name="test-coverage",
    source_type="oci",
    source="ghcr.io/acme/agent-plugin:v1.0.0",
    subpath="skills/test-coverage",
)

# Assembled bundle: each member has its own source
bundle_version = mlflow.genai.create_skill_bundle_version(
    name="pr-workflow",
    skills=[
        "skills:/code-review/1",
        "skills:/test-coverage/1",
    ],
)

# Monolithic bundle from a single OCI image. Embedded member
# versions are registered without their own sources.
mlflow.genai.register_skill(
    name="embedded-review",
)

bundle_version = mlflow.genai.create_skill_bundle_version(
    name="pr-workflow-mono",
    source_type="oci",
    source="ghcr.io/acme/agent-plugin:v1.0.0",
    skills=[
        "skills:/embedded-review/1#skills/embedded-review",
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
    version=1,
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
    version=1,
)
# bundle_version.skills == ["skills:/code-review/1", ...]

# Resolve a bundle alias
bundle_version = mlflow.genai.get_skill_bundle_version_by_alias(
    name="pr-workflow",
    alias="production",
)
```
