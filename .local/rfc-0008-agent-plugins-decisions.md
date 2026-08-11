# RFC-0008 Agent Plugins Decision Log

This document records decisions made while revising RFC-0008 to align MLflow's
`AgentPlugin` entity with the open Agent Plugins specification. It is a working
discussion aid, not RFC text.

## Goal

Make the open Agent Plugins format the canonical package representation for
MLflow `AgentPlugin` versions. MLflow adds governance and composition around
the standard rather than defining a competing package format.

## Settled decisions

### Use the hybrid storage precedent from RFC-0004

Each `AgentPluginVersion` will preserve the complete canonical `plugin.json`
payload in a JSON database column. Important fields will also be extracted into
ordinary columns where needed for identity, search, UI, indexing, or joins.

MLflow-managed information remains outside the canonical payload, including
lifecycle status, aliases, tags, permissions, source information, and member
references.

**Why:** This preserves the complete upstream representation, including nested
extensions and future schema additions, while retaining efficient relational
queries and referential integrity.

### Use string versions based on `plugin_json["version"]`

Agent plugin versions will use strings rather than server-assigned integers.

- New plugin and manifest supplies a version: use it.
- New plugin and manifest omits a version: assign `0.1.0`.
- Existing plugin and new manifest supplies a version: use it.
- Existing plugin and new manifest omits a version: assign a valid SemVer using
  a simple implementation-defined heuristic.

An automatically assigned version will be inserted into the canonical stored
`plugin_json`, maintaining this invariant:

```python
agent_plugin_version.version == agent_plugin_version.plugin_json["version"]
```

The RFC does not need to specify the exact version-generation heuristic. It
should guarantee that generated versions are valid SemVer and unique for the
agent plugin.

**Why:** This follows publisher-supplied standard versions whenever possible
without creating an obscure user-facing version argument for manifests that
omit an optional field.

### Resolve latest semantically when possible

When all eligible versions can be semantically ordered, `latest` uses SemVer
precedence. When semantic ordering is unavailable, including a mixture of valid
SemVer and non-SemVer strings, `latest` uses creation time. Creation time also
breaks equal SemVer precedence, including versions that differ only in build
metadata.

**Why:** The Agent Plugins specification recommends SemVer but permits other
strings. MLflow should preserve supplied standard values without losing a
well-defined `latest` result.

### Retain mutable parent presentation metadata

The parent `AgentPlugin` retains mutable MLflow-managed presentation metadata,
following RFC-0004. Each version also preserves the publisher's canonical
metadata inside its immutable `plugin_json`.

When parent presentation fields are unset, the UI may fall back to metadata
from the latest resolved version. API responses do not silently apply this
fallback.

**Why:** Organizations can curate how a governed plugin is presented without
altering or losing publisher metadata.

### Accept the complete canonical payload in registration APIs

The primary registration path accepts a complete `plugin_json` payload plus
MLflow member references and source information. MLflow validates the payload,
extracts its name and version, generates a version when necessary, and creates
or reuses the parent entity.

Import adapters and assembled-plugin creation converge on this registration
path.

**Why:** This follows RFC-0004's `server_json` API pattern and avoids creating a
second flattened API for every standard manifest field.

### Make Agent Plugins canonical and Claude Code an import adapter

- A standard Agent Plugins package contributes its root `plugin.json` and uses
  the standard `skills/*/SKILL.md` discovery rule.
- A Claude Code plugin is translated from `.claude-plugin/plugin.json` into a
  canonical Agent Plugins `plugin_json` payload and uses Claude-specific skill
  discovery.
- Both paths call the same registration logic and produce the same stored
  representation.

**Why:** MLflow remains neutral at ingestion while having one canonical model
instead of treating every package format as an equal internal representation.

### Allow versions with no skill members

An `AgentPluginVersion` may contain zero skill-member references. This includes
valid MCP-only and component-free Agent Plugins packages.

RFC-0008 will preserve the complete monolithic package and recognize the
presence of `mcp.json`, but it will not register individual MCP entries as
members. Searchable MCP membership remains follow-up work.

**Why:** RFC-0008 can represent every valid Agent Plugins package without
prematurely designing MCP membership or requiring every plugin to contain a
skill.

### Synthesize minimal manifests for assembled plugins

Callers creating assembled plugins are not required to construct a complete
`plugin_json`. When one is not supplied, MLflow constructs a minimal valid
payload containing the canonical schema identifier, plugin name, and generated
version. Callers may supply a richer `plugin_json` when they want to preserve
additional standard metadata such as author, repository, license, keywords, or
extensions.

This synthesis occurs in the server-side registry layer for both registration
and low-level version creation. Monolithic import always submits the complete
manifest produced by validation or format translation, and persistence never
stores a null canonical payload.

**Why:** Every stored version receives the canonical representation without
making the common assembled-plugin workflow more complicated.

### Make `plugin_json` immutable per version

Once an `AgentPluginVersion` is created, its canonical `plugin_json` cannot be
modified. Changing canonical package fields requires creating a new version.
MLflow-managed status, tags, aliases, permissions, and parent presentation
metadata remain mutable outside the payload.

**Why:** A registered version must continue to identify the same package
definition for reproducibility, auditing, and reliable alias resolution.

### Validate `plugin_json` according to Agent Plugins v1

MLflow initially recognizes the Agent Plugins v1.0.0 schema identifier and
applies the standard's validation and failure rules.

- Fatal manifest violations reject registration.
- An unsupported `$schema` rejects registration.
- Unknown top-level fields and a non-object `extensions` field are reported as
  nonfatal warnings and ignored semantically, consistent with the standard.
- Nonfatal unknown content is preserved in the immutable JSON payload but is
  not interpreted by MLflow.
- MLflow assigns the optional version, when necessary, before storing the final
  canonical payload.

**Why:** MLflow should validate a canonical package the same way as a conforming
Agent Plugins client rather than inventing separate manifest rules.

### Retain assembled and monolithic source authority

- A monolithic agent plugin version has one package-level source containing the
  complete package. Its embedded skill versions have no individual sources and
  are located through member subpaths.
- An assembled agent plugin version has no package-level source. Each referenced
  skill version has its own authoritative source until a future export process
  materializes a package.
- A version cannot combine a package-level source with independently sourced
  skill members.
- Both kinds have canonical `plugin_json` and may contain zero skill members.

**Why:** A single authority for content avoids ambiguous pull and future export
behavior.

### Use string database versions for agent plugins only

`agent_plugin_versions.version` changes from an integer to a string and becomes
the canonical version extracted from or inserted into `plugin_json`. The table
also stores the immutable JSON payload and nullable parsed SemVer components to
support semantic latest-version resolution.

Agent plugin alias targets and the plugin side of membership foreign keys also
change to strings. Skill versions and the skill side of membership references
remain server-assigned integers.

The current rule that an imported embedded skill version equals its containing
agent plugin version is removed. Each embedded skill receives its own next
integer version and the membership row records that independent version.

**Why:** Agent Plugins defines an optional publisher version string, while the
Agent Skills format does not provide an equivalent version identity.

### Map the standard name into MLflow's scoped identity

`plugin_json["name"]` is the canonical `AgentPlugin.name`. MLflow retains
`organization` as a separate registry namespace, so uniqueness remains scoped
to `(workspace, organization, name)` rather than becoming global. If an API path
or explicit argument also supplies a name, it must match the canonical payload.

Agent plugin names follow the Agent Plugins naming constraints. Skill naming is
unchanged.

**Why:** This preserves standard package identity while allowing different
organizations to govern packages with the same upstream name.

### Use a dedicated unambiguous Agent Plugin URI

Agent Plugins use `agent-plugins:/<organization-or-_>/<name>` for parent targets
and `agent-plugins:/<organization-or-_>/<name>/<version>` for exact versions.
The `_` segment explicitly represents an empty organization. Alias targets use
the same fixed identity shape with `@alias`.

Agent plugin version strings may be non-SemVer but must be nonempty, no longer
than the database limit, and exclude URI-significant `/`, `@`, `#`, and `?`
characters. Skill member URIs remain unchanged.

**Why:** The prior shared `skills:/name/version` grammar distinguished integer
versions from `organization/name` by parsing the final segment. String plugin
versions make that grammar ambiguous, and a distinct scheme also identifies the
correct resource type.

### Keep RFC-0008 membership skill-specific

RFC-0008 member references continue to point only to registered
`SkillVersion` records. MCP configuration and other standard package content
may be recognized and preserved in a monolithic package, but it does not
receive member references in RFC-0008.

MCP references and other non-skill membership remain follow-up work in the
proposal currently numbered RFC-0009, whose existing draft will need to be
redesigned around the canonical Agent Plugins model.

**Why:** Adding non-skill membership now would require defining identity,
versioning, validation, and referential integrity beyond the Skill Registry MVP.

### Recognize and preserve MCP content without registering it

RFC-0008 recognizes root `mcp.json` as standard Agent Plugins content and
preserves it unchanged in monolithic pulls. Introspection reports MCP content
as recognized but not individually registered. MCP-only packages are valid and
may be registered with zero skill members.

RFC-0008 does not create membership rows for MCP entries or link them to MCP
Registry records. Those capabilities remain follow-up work in RFC-0009.

**Why:** This accurately supports the complete standard package boundary while
keeping first-class MCP governance outside the Skill Registry MVP.

### Preserve chronological re-import behavior

On re-import, MLflow extracts or generates the incoming manifest version. If
that agent plugin version already exists, registration fails rather than
overwriting its immutable `plugin_json`. Otherwise MLflow creates the new
version and matches discovered skills by subpath against the most recently
created prior agent plugin version. Each matched skill receives its own next
integer version.

**Why:** Creation time preserves the existing chronological re-import behavior
when publisher version strings are not reliably ordered.

### Auto-detect import formats in canonical-first order

After fetching the source and applying its optional subpath, import detects the
format without requiring a user-supplied format flag:

1. A root `plugin.json` declaring a recognized Agent Plugins schema.
2. A `.claude-plugin/plugin.json` Claude Code manifest.
3. The generic skill-directory layout already supported by RFC-0008.

The standard root manifest takes precedence when more than one marker exists.
A root manifest that declares the Agent Plugins schema but fails validation
causes import to fail rather than silently falling back to a looser adapter.

**Why:** The canonical open format receives priority while existing Claude Code
and generic imports remain convenient and backward compatible.

### Retain low-level, registration, and import API layers

The public API follows RFC-0004 with three convenience levels over the same
stored entities:

- `create_agent_plugin()` creates the stable parent explicitly.
- `create_agent_plugin_version()` creates a version on an existing parent and
  accepts its canonical `plugin_json`, skill references, and source metadata.
- `register_agent_plugin()` validates and extracts the canonical identity,
  creates or reuses the parent, and creates the version and memberships in one
  call. For assembled plugins it may synthesize the minimal manifest already
  defined above.
- `import_plugin()` adds source fetching, format detection, translation, and
  skill discovery before invoking the same registration workflow.

**Why:** Advanced callers retain explicit lifecycle control while common and
external-package workflows remain concise. All paths converge on one canonical
representation and validation path.

### Provide matching CLI convenience levels

The `mlflow agent-plugins` CLI exposes `create`, `create-version`, `register`,
and `import` commands corresponding to the public API layers. `register` creates
or reuses the parent and creates a version in one operation, while `import`
additionally fetches and translates an external package.

**Why:** CLI users should receive the same one-call registration convenience as
SDK users rather than being required to create the parent separately.

### Separate free-text discovery from structured filters

The general UI search box searches user-visible descriptive metadata rather
than requiring users to know which underlying field contains a term.

- Skill free-text search covers name and description.
- Agent Plugin free-text search covers name, mutable parent description,
  resolved `plugin_json.description`, `plugin_json.keywords`,
  `plugin_json.author.name`, and MLflow organization.

Structured filters remain focused on precise governance and relationship
fields: name, description, organization, status, parent and version tags, skill
member name for Agent Plugins, and source type for versions. Dedicated author
and keyword filter syntax is not required for the MVP because those fields are
covered by free-text discovery.

Operational or opaque fields such as repository URLs, homepage URLs, license,
and extension payloads are visible through `plugin_json` but are not included
in free-text search or structured filters by default.

Agent Plugin manifest `keywords` are not copied into MLflow tags. Keywords are
immutable, publisher-provided, version-specific discovery metadata. Tags are
mutable MLflow governance metadata with independent parent and version
lifecycles. Keyword search operates on the canonical field or a dedicated
search projection rather than presenting copied keywords as editable tags.

**Why:** Descriptive information exposed for discovery should be discoverable
through the ordinary search box, while precise governance filters should retain
clear field semantics.

The SDK search APIs expose an optional `search_text` parameter separately from
`filter_string`. The two may be combined. Skills search existing name and
description columns. Each Agent Plugin version stores or otherwise maintains a
derived search projection for its immutable manifest description, keywords,
and author name; parent name, description, and organization also participate in
free-text matching. The projection is an index for discovery, while
`plugin_json` remains the canonical source of publisher metadata.

Parent search uses the latest-resolved version's projection. A lifecycle change
that selects a different latest version therefore changes the parent plugin's
manifest-derived search matches.

## Current status

The settled design is now reflected in the main RFC and implementation details.
Further review may surface implementation questions, but no design question
from this discussion remains open.
