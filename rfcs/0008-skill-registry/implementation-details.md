# RFC-0008: Skill Registry Implementation Details

This document contains implementation-level specifications for
RFC-0008 (Skill Registry). It covers database schema, store interface
method signatures, SDK convenience functions, REST API endpoints,
pagination/filtering, and the Python SDK/CLI mapping. These details
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
| `source_type` | `String(20)` | nullable; `git`, `oci`, `zip`, etc. |
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

### Subagent tables

The `subagents`, `subagent_versions`, `subagent_tags`,
`subagent_version_tags`, and `subagent_aliases` tables follow the
same structure as the corresponding skill tables above, including
`version_major/minor/patch` and `version_prerelease_sort_key`
columns and the latest-lookup index. FK
relationships mirror the skill tables: `subagent_versions` references
`subagents` with CASCADE delete, etc.

### Hook tables

The `hooks`, `hook_versions`, `hook_tags`, `hook_version_tags`,
and `hook_aliases` tables follow the same structure as the
corresponding skill tables, including `version_major/minor/patch`
and `version_prerelease_sort_key` columns and the latest-lookup
index. FK relationships mirror the skill tables.

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
| `source_type` | `String(20)` | optional; `git`, `oci`, `zip`, etc. |
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
| `member_type` | `String(20)` | PK; `skill`, `subagent`, `hook`, or `mcp_server` |
| `member_name` | `String(256)` | PK |
| `member_version` | `String(256)` | PK |
| `member_subpath` | `String(2048)` | nullable; member path inside bundle artifact |

FK: `(workspace, bundle_name, bundle_version)` references `skill_bundle_versions`, CASCADE delete.

The `member_type` column distinguishes member categories. When
`member_type` is `skill`, a FK to `skill_versions` enforces
referential integrity with RESTRICT delete. Similarly for `subagent`
(FK to `subagent_versions`) and `hook` (FK to `hook_versions`).

**Cross-registry references (`member_type='mcp_server'`).** There is no
database-level FK for MCP registry references. Referential integrity
is enforced at the application layer: the store validates that the
referenced `MCPServerVersion` exists when creating a bundle version
and returns `RESOURCE_DOES_NOT_EXIST` if it does not. This avoids
deployment-ordering dependencies between RFC-0004 and RFC-0008
migrations and allows either registry to be deployed independently.

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

- Top-level entity delete operations (`delete_skill`,
  `delete_subagent`, `delete_hook`, and `delete_skill_bundle`) are
  administrative hard deletes. They physically remove the parent row and
  cascade to child rows, subject to referential-integrity checks.
- Version delete operations (`delete_skill_version`,
  `delete_subagent_version`, `delete_hook_version`, and
  `delete_skill_bundle_version`) are soft deletes. They set
  `status='deleted'` when allowed by the lifecycle transition rules,
  update `last_updated_timestamp`, remove aliases that point to the
  deleted version, and exclude the version from normal
  get/search/list/latest resolution. Active versions must first be
  unpublished or deprecated before they can be deleted.
- The `deleted` status is terminal. Internal audit or provenance paths
  may retain enough metadata to explain historical traces and bundle
  snapshots, but deleted versions are not surfaced to consumers.

## Store interface

The store interface follows the mixin pattern established by the MCP
Server Registry (RFC-0004). Methods raise `NotImplementedError` rather
than using `@abstractmethod`, allowing stores that don't support skills
(e.g., `FileStore`) to work without stubbing every method.

In the store interface, `delete_*` methods on top-level entities are
hard deletes, while `delete_*_version` methods are soft deletes that
transition the version to `deleted`.

```python
class SkillRegistryMixin:
    # --- Skill operations ---

    def create_skill(
        self, name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> Skill:
        raise NotImplementedError

    def get_skill(self, name: str) -> Skill:
        raise NotImplementedError

    def search_skills(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[Skill]:
        raise NotImplementedError

    def update_skill(
        self,
        name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> Skill:
        raise NotImplementedError

    def delete_skill(self, name: str) -> None:
        raise NotImplementedError

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
        raise NotImplementedError

    def get_skill_version(
        self, name: str, version: str,
    ) -> SkillVersion:
        raise NotImplementedError

    def get_skill_version_by_alias(
        self, name: str, alias: str,
    ) -> SkillVersion:
        raise NotImplementedError

    def get_latest_skill_version(self, name: str) -> SkillVersion:
        raise NotImplementedError

    def search_skill_versions(
        self,
        name: str,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillVersion]:
        raise NotImplementedError

    def update_skill_version(
        self,
        name: str,
        version: str,
        status: SkillStatus | None = None,
    ) -> SkillVersion:
        raise NotImplementedError

    def delete_skill_version(
        self, name: str, version: str,
    ) -> None:
        raise NotImplementedError

    # --- Skill tag operations ---

    def set_skill_tag(
        self, name: str, key: str, value: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_tag(self, name: str, key: str) -> None:
        raise NotImplementedError

    def set_skill_version_tag(
        self, name: str, version: str,
        key: str, value: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_version_tag(
        self, name: str, version: str, key: str,
    ) -> None:
        raise NotImplementedError

    # --- Skill alias operations ---

    def set_skill_alias(
        self, name: str, alias: str, version: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_alias(
        self, name: str, alias: str,
    ) -> None:
        raise NotImplementedError

    # --- Subagent operations ---
    # Same shape as Skill: create, get, search, update, delete,
    # plus version, tag, and alias operations.

    def create_subagent(
        self, name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> Subagent:
        raise NotImplementedError

    def get_subagent(self, name: str) -> Subagent:
        raise NotImplementedError

    def search_subagents(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[Subagent]:
        raise NotImplementedError

    def update_subagent(
        self, name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> Subagent:
        raise NotImplementedError

    def delete_subagent(self, name: str) -> None:
        raise NotImplementedError

    def create_subagent_version(
        self, name: str, version: str,
        display_name: str | None = None,
        source_type: str | None = None,
        source: str | None = None,
        subpath: str | None = None,
        content_digest: str | None = None,
    ) -> SubagentVersion:
        raise NotImplementedError

    # Remaining subagent version, tag, and alias operations
    # follow the same pattern as skill operations above.

    # --- Hook operations ---
    # Same shape as Skill: create, get, search, update, delete,
    # plus version, tag, and alias operations.

    def create_hook(
        self, name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> Hook:
        raise NotImplementedError

    def get_hook(self, name: str) -> Hook:
        raise NotImplementedError

    def search_hooks(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[Hook]:
        raise NotImplementedError

    def update_hook(
        self, name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> Hook:
        raise NotImplementedError

    def delete_hook(self, name: str) -> None:
        raise NotImplementedError

    def create_hook_version(
        self, name: str, version: str,
        display_name: str | None = None,
        source_type: str | None = None,
        source: str | None = None,
        subpath: str | None = None,
        content_digest: str | None = None,
    ) -> HookVersion:
        raise NotImplementedError

    # Remaining hook version, tag, and alias operations
    # follow the same pattern as skill operations above.

    # --- SkillBundle operations ---

    def create_skill_bundle(
        self, name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> SkillBundle:
        raise NotImplementedError

    def get_skill_bundle(self, name: str) -> SkillBundle:
        raise NotImplementedError

    def search_skill_bundles(
        self,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillBundle]:
        raise NotImplementedError

    def update_skill_bundle(
        self,
        name: str,
        display_name: str | None = None,
        description: str | None = None,
    ) -> SkillBundle:
        raise NotImplementedError

    def delete_skill_bundle(self, name: str) -> None:
        raise NotImplementedError

    # --- SkillBundleVersion operations ---

    def create_skill_bundle_version(
        self,
        name: str,
        version: str,
        display_name: str | None = None,
        skills: list[SkillMemberRef] | None = None,
        subagents: list[SubagentMemberRef] | None = None,
        hooks: list[HookMemberRef] | None = None,
        mcp_servers: list[McpServerMemberRef] | None = None,
        source_type: str | None = None,
        source: str | None = None,
        subpath: str | None = None,
        content_digest: str | None = None,
    ) -> SkillBundleVersion:
        raise NotImplementedError

    def get_skill_bundle_version(
        self, name: str, version: str,
    ) -> SkillBundleVersion:
        raise NotImplementedError

    def get_skill_bundle_version_by_alias(
        self, name: str, alias: str,
    ) -> SkillBundleVersion:
        raise NotImplementedError

    def get_latest_skill_bundle_version(
        self, name: str,
    ) -> SkillBundleVersion:
        raise NotImplementedError

    def search_skill_bundle_versions(
        self,
        name: str,
        filter_string: str | None = None,
        max_results: int = 100,
        order_by: list[str] | None = None,
        page_token: str | None = None,
    ) -> PagedList[SkillBundleVersion]:
        raise NotImplementedError

    def update_skill_bundle_version(
        self,
        name: str,
        version: str,
        status: SkillStatus | None = None,
    ) -> SkillBundleVersion:
        raise NotImplementedError

    def delete_skill_bundle_version(
        self, name: str, version: str,
    ) -> None:
        raise NotImplementedError

    # --- SkillBundle tag operations ---

    def set_skill_bundle_tag(
        self, name: str, key: str, value: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_bundle_tag(
        self, name: str, key: str,
    ) -> None:
        raise NotImplementedError

    def set_skill_bundle_version_tag(
        self, name: str, version: str,
        key: str, value: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_bundle_version_tag(
        self, name: str, version: str, key: str,
    ) -> None:
        raise NotImplementedError

    # --- SkillBundle alias operations ---

    def set_skill_bundle_alias(
        self, name: str, alias: str, version: str,
    ) -> None:
        raise NotImplementedError

    def delete_skill_bundle_alias(
        self, name: str, alias: str,
    ) -> None:
        raise NotImplementedError

```

## SDK convenience functions

The `mlflow.genai.skills` namespace provides convenience functions that
combine store operations, matching the pattern established by
`mlflow.genai.register_mcp_server()` in RFC-0004.

```python
def register_skill(
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


def register_subagent(
    name: str,
    version: str,
    display_name: str | None = None,
    description: str | None = None,
    source_type: str | None = None,
    source: str | None = None,
    subpath: str | None = None,
    content_path: str | None = None,
    content_digest: str | None = None,
) -> SubagentVersion:
    """Register a subagent version. Auto-creates the parent
    Subagent if it does not exist."""


def register_hook(
    name: str,
    version: str,
    display_name: str | None = None,
    description: str | None = None,
    source_type: str | None = None,
    source: str | None = None,
    subpath: str | None = None,
    content_path: str | None = None,
    content_digest: str | None = None,
) -> HookVersion:
    """Register a hook version. Auto-creates the parent Hook if
    it does not exist."""


def pull(
    name: str | None = None,
    bundle: str | None = None,
    version: str | None = None,
    alias: str | None = None,
    destination: str = ".",
) -> str:
    """Pull skill, subagent, hook, or bundle content from registered
    sources to a local directory. Specify name for a single
    capability or bundle for a skill bundle."""
```

## REST API

The REST API uses RESTful nested resource paths, following the pattern
from the MCP Server Registry proposal.

### Skill endpoints

All paths relative to `/ajax-api/3.0/mlflow/skills`.

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

### Subagent endpoints

All paths relative to `/ajax-api/3.0/mlflow/subagents`. Same
structure as skill endpoints: CRUD on subagents and subagent versions,
plus tags and aliases. Parent delete is a hard delete; version delete
sets `status='deleted'`.

### Hook endpoints

All paths relative to `/ajax-api/3.0/mlflow/hooks`. Same structure as
skill endpoints: CRUD on hooks and hook versions, plus tags and
aliases. Parent delete is a hard delete; version delete sets
`status='deleted'`.

### Skill bundle endpoints

All paths relative to `/ajax-api/3.0/mlflow/skill-bundles`.

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

**Skills, subagents, hooks, and bundles:** `name LIKE '%review%'`,
`status = 'active'`, `tags.team = 'platform'`

**Versions (all entity types):** `status = 'active'`,
`source_type = 'git'`

**Skill bundle versions:** `status = 'active'`,
`tags.approved = 'true'`

## Python SDK and CLI

The `mlflow.genai.skills` module exposes top-level functions delegating to
`MlflowClient`, with a 1:1 mapping to the store mixin methods above.
CLI command groups (`mlflow skills`, `mlflow subagents`,
`mlflow hooks`, and `mlflow skill-bundles`) provide the same
operations from the command line. See the basic examples in the main
RFC for usage.

`pull` is implemented in the SDK/CLI layer, not the store mixin. The
client calls `get_skill_version` (or the corresponding subagent/hook
method, or resolves an alias) to obtain the source pointer, then
fetches content locally using source-type-specific logic (git clone,
OCI pull, ZIP download, or MLflow artifact download). This keeps the
store as a pure data-access layer.

## Skill entity

A skill is a directory containing a SKILL.md entry point plus
supporting files (scripts, templates, reference material). The
`Skill` entity is the logical governed asset, scoped to a workspace.

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
    status: SkillStatus | None = None
    tags: dict[str, str] = field(default_factory=dict)
    aliases: list[SkillAlias] = field(default_factory=list)
    latest_version: str | None = None
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
| `aliases` | `list[SkillAlias]` | Stable version pointers (e.g., `production` -> `1.2.0`) |
| `latest_version` | `str` | Read-only; highest semantic version among `active` versions if one exists, otherwise highest non-`deleted` non-`active` version |
| `workspace` | `str` | Visibility boundary |

## SkillVersion entity

A versioned record containing a typed source pointer, status, and
tags.

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
| `subpath` | `str` | Optional path within the artifact where this skill's content lives. Used with Git (path within the repo), OCI (path within the image), and ZIP (path within the archive) when the skill content is not at the artifact root. Not used for MLflow artifacts (path is scoped at upload) |
| `content_digest` | `str` | Optional digest for integrity verification (e.g., `sha256:abc123...`). Aligns with OCI digest terminology |
| `status` | `SkillStatus` | Per-version lifecycle: `draft`, `active`, `deprecated`, `deleted` |
| `aliases` | `list[str]` | Alias names currently pointing at this version (read-only, projected from alias table) |

## SkillVersion field details

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
to different content, register a new version. Mutable fields (`display_name`,
`status`, `tags`) can be updated independently.

## SkillBundle entity

A skill bundle groups related capabilities (skills, subagents, hooks,
and MCP servers) into a governed unit that maps to the "plugin"
concept in agent harnesses. Follows the same top-level pattern as
Skill: versions, tags, and aliases.

```python
@dataclass
class SkillBundle:
    name: str
    display_name: str | None = None
    description: str | None = None
    workspace: str | None = None
    status: SkillStatus | None = None
    tags: dict[str, str] = field(default_factory=dict)
    aliases: list["SkillBundleAlias"] = field(default_factory=list)
    latest_version: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

`SkillBundle.status` is read-only and uses the same parent-resolved
version rule as `Skill`: highest active semantic version if present,
otherwise highest non-deleted non-active semantic version. Latest
version resolution follows the same fallback: highest active semantic
version if one exists, otherwise highest non-deleted non-active
version.

## SkillBundleVersion entity

A versioned snapshot of a skill bundle's membership. Each version
captures a specific set of capabilities that work together, organized
by type.

```python
@dataclass
class SkillMemberRef:
    name: str
    version: str
    member_subpath: str | None = None

@dataclass
class SubagentMemberRef:
    name: str
    version: str
    member_subpath: str | None = None

@dataclass
class HookMemberRef:
    name: str
    version: str
    member_subpath: str | None = None

@dataclass
class McpServerMemberRef:
    name: str
    version: str

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
    subagents: list[SubagentMemberRef] = field(default_factory=list)
    hooks: list[HookMemberRef] = field(default_factory=list)
    mcp_servers: list[McpServerMemberRef] = field(default_factory=list)
    aliases: list[str] = field(default_factory=list)
    workspace: str | None = None
    created_by: str | None = None
    last_updated_by: str | None = None
    creation_timestamp: int | None = None
    last_updated_timestamp: int | None = None
```

Each member list contains typed references. `SkillMemberRef`,
`SubagentMemberRef`, and `HookMemberRef` carry an optional
`member_subpath` for locating the member inside a monolithic bundle
artifact (must be null for assembled bundles). The `skills`,
`subagents`, and `hooks` lists reference entities in this registry.
The `mcp_servers` list references entries in the MCP Server Registry
(RFC-0004).

## SkillBundleVersion field details

**Version uniqueness.** The combination of `(name, version)` is unique
within a workspace.

**Bundle-level source.** A bundle version is either monolithic or
assembled, never both:

- **Monolithic:** has its own `source_type`, `source`, `subpath`,
  and `content_digest`, pointing to a single artifact (e.g., an OCI
  image or Git repo) that contains the complete plugin. `pull`
  fetches the bundle artifact as a unit. The bundle version generally
  has member references so embedded skills, subagents, and hooks remain
  governed and traceable, but those member versions may omit their own
  `source` because the bundle artifact is the authoritative source.
- **Assembled:** has individual member references. Each skill,
  subagent, and hook member has its own
  source. `pull` fetches members individually. If a skill, subagent, or
  hook member has no source, `pull` fails rather than producing a
  partial local bundle.

A monolithic bundle artifact is a generic package of content (skill
files, agent definitions, hook scripts). It may or may not be
harness-ready; the adapter does not assume either way. Harness
adapters (RFC-0009) generate harness-specific manifests from
registry metadata at install time, since the registry is the
governed source of truth. Correctness of the artifact layout is
the publisher's responsibility; the registry does not validate
artifact contents at registration time.

**Immutability contract.** The member lists and source fields of a
bundle version are immutable after creation. To change the set of
members or source pointer, register a new bundle version. Mutable
fields (`display_name`, `status`, `tags`) can be updated independently.

Members with `member_type` of `skill`, `subagent`, or `hook`
reference entities in this registry. For monolithic bundles, those
member versions may omit `source` because their content is embedded in
the bundle artifact. The optional membership `member_subpath` identifies
where the member lives inside the bundle artifact. For assembled bundles,
`member_subpath` must be null because the member's own `source` and
`subpath` identify its content. The API rejects attempts to set
`member_subpath` on a membership whose member version has its own source. Members with
`member_type='mcp_server'` reference an `MCPServerVersion` in the
MCP server registry (RFC-0004). This cross-registry reference
enables:

- **Deduplication.** Two bundles that both need `github-mcp`
  reference the same MCP registry entry. No duplicate configs.
- **Runtime status.** The MCP registry tracks deployment state via
  hosted bindings (`is_deployed`, `endpoint_url`). Install-time
  tooling can check whether a referenced MCP server is already
  running rather than starting a duplicate.
- **Single source of truth.** MCP server definitions are governed in
  the MCP registry; skill bundles reference them rather than carrying
  standalone copies.

A member can appear in multiple bundles and multiple bundle versions.
Membership is at the version level, so a bundle version is a
reproducible snapshot of "these specific asset versions work together."

**Bundle-level source and embedded MCP configs.** When a bundle
version is monolithic (a single OCI image or Git repo containing a
complete plugin), the artifact may include MCP configs alongside
skills and subagents. Embedded skills, subagents, and hooks should
generally be registered as member versions so they remain governed and
traceable. MCP servers are different because they belong in the MCP
Server Registry (RFC-0004): MCP configs within a monolithic artifact do
not need separate MCP registry entries unless the publisher wants them
independently governed and reusable. Cross-registry MCP references are
for bundles where MCP servers are independently registered and managed.

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

`pull` is harness-agnostic. It downloads content but does not generate
harness-specific manifests or place files in harness-specific
directories. Harness-specific installation is covered in RFC-0009.

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

### Register other capability types

```python
# Register a subagent
mlflow.genai.skills.register_subagent(
    name="security-auditor",
    version="1.0.0",
    description="Security specialist for auth and payment code",
    source_type="git",
    source="https://github.com/acme/agent-skills.git@v1.0.0",
    subpath="security-auditor",
)

# Register a hook
mlflow.genai.skills.register_hook(
    name="pre-commit-scan",
    version="1.0.0",
    description="Runs security scan before tool commits",
    source_type="git",
    source="https://github.com/acme/agent-skills.git@v1.0.0",
    subpath="pre-commit-scan",
)
```

### Create a skill bundle with cross-registry references

```python
# Assembled bundle: members reference individually registered versions.
# Each member has its own source. No bundle-level source.
bundle_version = mlflow.genai.skills.create_skill_bundle_version(
    name="pr-workflow",
    version="1.0.0",
    skills=[
        SkillMemberRef(name="code-review", version="1.0.0"),
    ],
    subagents=[
        SubagentMemberRef(name="security-auditor", version="1.0.0"),
    ],
    # Reference MCP servers from the MCP registry (RFC-0004)
    mcp_servers=[
        McpServerMemberRef(name="github-mcp", version="2.0.0"),
    ],
)

# Monolithic bundle from a Git repository. The browsable URL
# https://github.com/acme/plugins/tree/v1.0.0/pr-workflow becomes
# source (clone URL @ ref) + subpath. Embedded member versions are
# located by member_subpath within the bundle artifact.
bundle_version = mlflow.genai.skills.create_skill_bundle_version(
    name="pr-workflow-mono",
    version="1.0.0",
    source_type="git",
    source="https://github.com/acme/plugins.git@v1.0.0",
    subpath="pr-workflow",
    skills=[
        SkillMemberRef(name="embedded-review", version="1.0.0",
                       member_subpath="skills/embedded-review"),
    ],
    hooks=[
        HookMemberRef(name="pre-commit-scan", version="1.0.0",
                      member_subpath="hooks/pre-commit-scan"),
    ],
)

# Monolithic bundle from an OCI artifact: same pattern, different source.
bundle_version = mlflow.genai.skills.create_skill_bundle_version(
    name="pr-workflow-oci",
    version="1.0.0",
    source_type="oci",
    source="ghcr.io/acme/agent-plugin:v1.0.0",
    skills=[
        SkillMemberRef(name="embedded-review", version="1.0.0",
                       member_subpath="skills/embedded-review"),
    ],
)
```

### Register skills from an OCI artifact with subpath

```python
# Register individual skills that live inside a shared OCI image.
# The subpath identifies each skill's location within the image.
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

# Assembled bundle: members reference individually registered skills.
# Each member has its own source. No bundle-level source.
bundle_version = mlflow.genai.skills.create_skill_bundle_version(
    name="pr-workflow",
    version="1.0.0",
    skills=[
        SkillMemberRef(name="code-review", version="1.0.0"),
        SkillMemberRef(name="test-coverage", version="2.1.0"),
    ],
)

# Monolithic bundle: a single OCI image contains the complete plugin.
# Embedded member versions are registered without their own sources.
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
# bundle_version.subagents == [SubagentMemberRef(name="security-auditor", version="1.0.0"), ...]
# bundle_version.mcp_servers == [McpServerMemberRef(name="github-mcp", version="2.0.0"), ...]

# Resolve a bundle alias
bundle_version = mlflow.genai.skills.get_skill_bundle_version_by_alias(
    name="pr-workflow",
    alias="production",
)
```

CLI equivalents for these operations use `mlflow skills`, `mlflow
subagents`, `mlflow hooks`, and `mlflow skill-bundles` command groups.
