# RFC 0009: Skill Tracing

| start_date   | 2026-08-23 |
| :----------- | :--------- |
| mlflow_issue | |
| rfc_pr       | |

| Author(s)              | [Bill Murdock](https://github.com/jwm4) (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-08-23 |
| **AI Assistant(s)**    | Claude Code |

> **Status: stub.** This RFC is a placeholder. Content is in progress and
> will be filled in before this PR is opened for review.

# Summary

Skill tracing connects MLflow traces to the skills that produced them, so
that agent developers can answer questions like "which traces used this
skill?" and "how did behavior change when this skill was updated?"

Tracing was originally drafted as part of
[RFC-0008 (MVP Skill Registry)](https://github.com/mlflow/rfcs/pull/26)
and deferred out of that RFC to keep the MVP focused on the registry
itself. This RFC picks that work back up as a standalone proposal.

# Motivation

TBD.

## Out of scope

TBD.

# Detailed design

TBD.

# Drawbacks

TBD.

# Alternatives

TBD.

# Adoption strategy

TBD.

# Open questions

The following questions were raised during review of RFC-0008 and are
carried forward into this RFC:

- **OTel alignment.** Should skill-to-trace linkage be expressible
  through standard OpenTelemetry APIs, or is an MLflow-specific wrapper
  the only supported path?
- **Query API.** Should skill trace queries extend the existing
  `search_traces` filter syntax rather than adding a skill-specific
  search function?
- **Span exposure.** Should a skill tracing context manager expose the
  span object to the caller, or manage it entirely internally?
- **Manifest location.** Where does the manifest that maps installed
  skills to registry coordinates live, for project-scoped versus
  user-scoped installs?
- **Automatic context capture.** Can the skill-to-trace mapping be
  captured automatically when a skill is resolved and handed to a
  harness (LangGraph, ADK, and others), instead of requiring the user to
  set context manually?
- **Digest-based linking.** Should traces link to skill content digests
  in addition to specific `name/version` pairs, so that traces correlate
  across re-imports of identical content?
