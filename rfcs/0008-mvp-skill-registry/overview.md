# MLflow Skills Registry: Overview

## What is the Skills Registry?

AI agents rely on skills to do useful work: code review, data
transformation, security scanning, report generation, and hundreds of
others. As organizations adopt agentic workflows, the number of skills
in use grows quickly, but there is no central place to manage them.

The proposed MLflow Skills Registry is a governed, central registry for
discovering, versioning, and installing AI agent skills. It integrates
with MLflow's tracing infrastructure so that every skill execution is
automatically linked back to the registry, giving teams full visibility
into which skill version ran during any agent execution.

## The problem today

Today, skills live in scattered Git repositories, team wikis, and chat
threads. Nobody can answer "what skills do we have?" or "who is using
this skill?" Teams duplicate effort because they cannot find what
already exists. When someone updates a skill, nothing tracks the
version, manages the lifecycle, or provides a way to roll back.

Installation is manual and inconsistent. The same skill ends up
configured differently across Claude Code, Cursor, and Copilot,
leading to subtle behavioral differences that are hard to diagnose.

When something goes wrong in an agent run, traces show what the agent
did, but not which skill was responsible or which version was active.
Debugging means guessing which skill changed, then manually
correlating timestamps and deployments.

## What the registry provides

### Visibility

Every skill in the registry is visible to the entire
organization. Teams can see what skills exist, who published them, and
what status they are in. Skills no longer hide in private repos that
only one team knows about.

### Discoverability

Teams can search and browse skills by name, tags, status, and
description, and filter to find skills relevant to a specific domain
or workflow. The
MLflow UI provides a gallery view for browsing and a detail page for
each skill with version history, source information, and usage
metadata.

### Versioning and lifecycle

Skills have explicit versions and move through a managed lifecycle:
draft, active, deprecated, and archived. Teams can promote a skill
from draft to active when it is ready for production, deprecate it when
a replacement is available, and archive it when it is no longer needed.
Aliases like "production" or "latest" let consumers reference a
logical version that the skill owner can update without breaking
downstream users.

### Bundles

Skills rarely work alone. A code review workflow might combine a
review skill, a style-check skill, and a security scan. Bundles let
platform teams package multiple skills into a curated workflow that can
be versioned, governed, and installed as a unit. Bundle members are
pinned to specific skill versions, so the combination is reproducible.

### Harness-agnostic installation

The registry works across AI agent harnesses (Claude Code, Cursor,
Copilot, and others) through package manager plugins. A single registry
entry can be installed into any supported harness, with the package
manager handling the harness-specific file layout and configuration.
Teams manage skills in one place regardless of which harness their
developers prefer.

### Reproducibility

A lock file records the exact registry coordinates (skill names,
versions, sources, and tracking server) used in a workspace. Replaying
from the lock file reproduces the same environment on another machine
or in CI, eliminating "works on my machine" problems for agent
configurations.

### Tracing and observability

MLflow already traces agent execution. The registry builds on this by
annotating every skill execution span with the skill's
registry coordinates: name, version, workspace, and bundle membership.

Teams can query traces by skill name and version to answer questions
that were previously impossible: "Which version of the code-review
skill was active when this agent run failed?" "Did latency increase
after we upgraded from v1.2 to v1.3?" "How does token usage compare
across skill versions?" These queries work across the entire trace
history, not just the most recent run.

Because the registry and the tracing infrastructure live in the same
platform, the connection is automatic, not an integration that teams
have to build and maintain.

## Possible extension: extended bundles

Real-world agent extensions contain more than skills. A Claude Code
extension might include skills alongside agents, hooks, and MCP server
configurations. A Cursor extension might include instruction rules in
formats specific to that harness.

In the MVP registry design, only skills receive individual registry
entries. The other components ship alongside skills but remain
invisible to governance: teams cannot search, track, or query them for
impact analysis.

Extended bundles would register all components in an extension, not
just the skills. An administrator could then answer questions like "which
bundles include a hook?" or "show me every bundle with an MCP server
dependency." The same tracing integration that links skill executions
to the registry would extend to non-skill components, closing the
observability gap for hooks, agents, and other extension content.

Extended bundles also enable cross-registry connections. When a bundle
includes an MCP server, it can reference the server's entry in the
MLflow MCP Server Registry rather than embedding configuration
directly. This lets teams answer questions that span both registries,
such as "which bundles depend on a deprecated MCP server?" and see the
full dependency picture in one place.

The design is generic: MLflow treats the component type as a label, not
a schema. New component types appear automatically when an extension
that uses them is imported, with no changes to MLflow itself.

## Further reading

- [PR #26: RFC-0008, Skill Registry (MVP)](https://github.com/mlflow/rfcs/pull/26):
  the core registry design, including registration, versioning, bundles,
  installation, tracing, and the package manager plugin interface.
- [PR #27: RFC-0009, Extended Skill Bundles](https://github.com/mlflow/rfcs/pull/27):
  the extended bundle design for registering non-skill components.
