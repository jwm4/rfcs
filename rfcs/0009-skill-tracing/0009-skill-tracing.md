# RFC 0009: Skill Tracing

| start_date   | 2026-08-23 |
| :----------- | :--------- |
| mlflow_issue | |
| rfc_pr       | https://github.com/mlflow/rfcs/pull/37 |

| Author(s)              | [Bill Murdock](https://github.com/jwm4) (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-09-11 |
| **AI Assistant(s)**    | Claude Code |

**Table of contents**

- [Summary](#summary)
- [Basic example](#basic-example)
- [Motivation](#motivation)
  - [The problem](#the-problem)
  - [User journeys](#user-journeys)
  - [Out of scope](#out-of-scope)
- [Detailed design](#detailed-design)
  - [Link model](#link-model)
  - [SDK method and attribute contract](#sdk-method-and-attribute-contract)
  - [Queries](#queries)
  - [Install record](#install-record)
  - [Autologger behavior](#autologger-behavior)
  - [UI](#ui)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Adoption strategy](#adoption-strategy)
- [Open questions](#open-questions)

# Summary

Skill tracing connects MLflow traces to the registered skills that
produced them, so that agent developers and platform owners can answer
questions like "which traces used this skill?", "who is still on the
deprecated version?", and "how did behavior change when this skill was
updated?"

MLflow already traces agent conversations across harnesses and agent
frameworks: Claude Code via `mlflow autolog claude`, SDK applications
via framework autologgers such as `mlflow.langchain.autolog()` and
`mlflow.anthropic.autolog()`, and others. Those traces capture LLM
calls, tool use, timing, and token consumption as a tree of spans. What
they do not capture is which governed, versioned skill was active during
any part of the run.

This RFC links traces to skills. A skill activation produces a
**trace-level link** from the trace to the skill version, following
the pattern MLflow already uses to link prompts to traces. When the
activation is identifiable as a specific span, that span is also
annotated with the skill's registry coordinates: workspace,
organization, name, and version, plus the version's content digest
when the instrumentation has it. Tool calls that use a skill's
bundled files are annotated the same way. Each link records how it
was produced (explicit instrumentation, in-process resolution,
install-record matching, or content-marker inference), so consumers
of the linkage can weigh the evidence behind it.

Two terms recur below. A **harness** is a packaged agent application
that skills are installed into and that runs them without code written
by the user: Claude Code, Codex, Gemini CLI, OpenCode, Goose, and
OpenHands are examples. An **agent framework** is a library that
application code uses to build an agent; the developer's own process
loads the skill and drives the run: LangGraph,
Google ADK, the OpenAI Agents SDK, CrewAI, Pydantic AI, and Semantic
Kernel are examples.

Links are recorded along three paths:

- **Explicit instrumentation.** Application code that composes its own
  agent records the link where it activates a skill. This is the path
  for custom agents built on the MLflow SDK, and it is also
  expressible through plain OpenTelemetry for callers that are not
  using the MLflow tracing API directly.
- **Automatic instrumentation in agent frameworks.** When
  application code resolves a skill from the registry and hands it to
  a framework such as LangGraph or the OpenAI Agents SDK, MLflow holds
  the mapping from that skill to its registry coordinates in process,
  and the framework autologger records the link and annotates the
  activation span without the developer writing tracing code.
  Frameworks that MLflow traces by receiving their native
  OpenTelemetry output rather than by running instrumentation inside
  them are handled like the equivalent harnesses (see the harness
  journey).
- **Automatic instrumentation in harnesses.** When a skill installed
  into a harness such as Claude Code is tracked with MLflow, MLflow
  records which registry coordinates the harness-local skill came
  from. The harness autologger reads that record at run time to link
  and annotate.

Because traces carry skill links, they become queryable by skill. That
query surface is what turns tracing into governance evidence: adoption
tracking, impact analysis for deprecated or vulnerable versions, and
regression detection across skill versions.

**Relationship to other RFCs.** Skill tracing builds on
[RFC-0008: Skill Registry](https://github.com/mlflow/rfcs/blob/main/rfcs/0008-mvp-skill-registry/0008-mvp-skill-registry.md),
which defines the `Skill` and `SkillVersion` entities, the
`{workspace, organization, name, version}` coordinates, and the
client-asserted content `digest` that this RFC records on links.
The harness path depends on the install record described above, so
this RFC specifies that record and the command that writes it for a
skill the user has installed by whatever means they choose. It does
not specify an MLflow installer or package manager integration; an
installer could follow later and would have to produce the same
record. Skill
linking follows the same trace-level association pattern as prompt
linking and the MCP Server Registry's trace linking
([RFC-0004](https://github.com/mlflow/rfcs/blob/main/rfcs/0004-mcp-registry/0004-mcp-registry.md)).
It adds activation-span annotation because skill activation, unlike
an MCP server association, is an observable event inside the trace.

# Basic example

```python
import mlflow

with mlflow.start_span("review") as span:
    span.link_skill(name="code-review", version=1)
    result = llm.chat([{"role": "user", "content": "Review this..."}])

traces = mlflow.search_traces(
    locations=[experiment_id],
    filter_string="skill = 'code-review/1'",
)
```

The first call links the span's trace to version 1 of the registered
skill and annotates the span as the activation point. The query
returns every trace in the experiment linked to that version.

## Motivation

### The problem

Enterprises adopting the skill registry gain governance over skill
content: versions, status lifecycle, aliases, and discovery. What they
do not gain is any evidence about what happens when those skills run.
The registry knows what was published; the traces know what the agent
did; nothing connects the two.

1. **Traces do not say which skill was active.** A trace shows LLM
   calls, tool use, and token counts, but there is no way to tell
   whether the agent was operating under `code-review` version 1,
   `code-review` version 2, or no registered skill at all. Every
   question that starts with "for runs that used this skill" is
   unanswerable today.

2. **Harness-local names are not registry identity.** A skill installed
   into a harness may be renamed, prefixed, or namespaced by the package
   manager that installed it. Even when a harness emits the skill name
   it loaded, that string does not identify a registered version, and it
   carries no workspace or version. Matching on it heuristically after
   the fact is guesswork.

3. **Governance decisions have no evidence base.** Deprecating a version
   means telling consumers to move, but there is no way to see who is
   still on it. A security finding against a skill means asking which
   runs were exposed, and nothing can answer that. Promoting a version
   means asserting it is better, but there is no way to attribute a
   quality difference to the skill rather than to everything else that
   changed. Retiring an unused skill means knowing it is unused.

4. **Content identity and version identity diverge.** RFC-0008 re-mints
   a version on every import, so the same unchanged skill content can
   exist under many version numbers. Linking traces only to a specific
   `name/version` pair fragments the history of what is, by content, one
   skill. RFC-0008 already records a content `digest` on each version
   for exactly this grouping purpose, and traces should be able to use
   it.

### User journeys

These journeys illustrate the end-to-end workflows that skill tracing
enables. They cover the three instrumentation paths and the analysis
workflows the resulting links support.

#### Instrument a custom agent application

A developer building an agent directly on the MLflow SDK wants the
traces it produces to record which registered skill was active.

1. Record the link on the span where the skill is activated:
   ```python
   import mlflow

   with mlflow.start_span("review") as span:
       span.link_skill(name="code-review", version=1)
       ...
   ```
   The method is on the span object, so the target is explicit: it
   annotates that span as the activation point and links the span's
   trace to the skill version. Code without a span in hand uses
   `mlflow.get_current_active_span()`. Like the rest of the registry
   surface, the method also accepts an alias:
   ```python
   span.link_skill(name="code-review", alias="production")
   ```
   The alias is resolved when the link is recorded and the trace
   stores the concrete version, consistent with RFC-0008's rule that
   aliases are accepted as input but never stored in place of
   versions. Aliases move; a trace that stored `@production` would
   become ambiguous the moment the alias was repointed. Alias
   resolutions are cached per process with a short expiry, following
   the prompt registry's alias cache, so repeated links in a hot path
   do not query the registry on every call; a caller can shorten or
   disable the cache per call. Pinned versions need no resolution.
   The method also accepts an `organization`; the examples here omit
   it and use the empty default organization, as RFC-0008's examples
   do.

   **Without the MLflow SDK:** a caller instrumenting with plain
   OpenTelemetry has no `link_skill` method and instead sets these
   attributes on the activation span. MLflow recognizes the
   `mlflow.skill.*` attributes as a skill activation when it receives
   the span and records the link from them:
   ```python
   span.set_attribute("mlflow.skill.name", "code-review")
   span.set_attribute("mlflow.skill.version", 1)
   span.set_attribute("mlflow.skill.workspace", "default")
   span.set_attribute("mlflow.skill.organization", "")
   ```

2. **UI path:** open the trace in the MLflow UI. The trace view lists
   the linked skill versions among the trace's linked entities, each
   linking to the skill's registry detail page; how linked entities
   are presented (for example, one consolidated lineage view across
   prompts, skills, and other assets) is aligned across assets outside
   this RFC. Annotated activation spans show the skill coordinates
   in the span detail view. When recorded coordinates do not resolve
   (the version was deleted, or the trace came from a different
   workspace), the UI shows a "not found in registry" indicator.

#### Trace skills loaded by an agent framework

A developer using an agent framework (LangGraph, the OpenAI Agents
SDK, and similar) resolves a skill from the registry and hands it to
the framework. The linkage should not require explicit calls in
their code.

1. Resolve and pull the skill through MLflow, then hand it to the
   framework in whatever form the framework expects:
   ```python
   path = mlflow.genai.pull(
       "skills:/code-review@production", destination="./skills",
   )
   agent = build_agent(skills=[path])
   ```
2. Enable the framework autologger as usual. No explicit call is
   needed: because the skill was resolved through MLflow, the
   autologger knows its registry coordinates, links the trace, and
   annotates the span where the framework activates the skill.
3. Run the agent and open the trace. The result is the same as the
   explicit path: the trace is linked to the skill version, and the
   activation is annotated where it happened.

This follows a pattern MLflow already uses for other registry
entities: resolving an entity records its identity, and tracing picks
that identity up automatically. Skill activation is observable in
current frameworks, most directly where loading a skill is itself a
tool invocation. The per-framework mechanisms belong in the
detailed design.

#### Trace skills installed into a harness

A platform owner installs skills into a harness such as Claude Code,
where there is no application code to instrument and no in-process
resolution step.

1. Install the skill into the harness by any means (the harness's
   own installer, a package manager, or a plain copy of the pulled
   content), then tell MLflow where it landed:
   ```bash
   mlflow skills track skills:/code-review@production \
       --path ~/.claude/skills/code-review --harness claude-code
   ```
   MLflow resolves the alias to a concrete version, verifies that the
   content at the path matches the version's registered digest, and
   writes an install record mapping the harness-local skill name and
   location to the skill's workspace, organization, name, version,
   and content digest. Recording this is what makes the linkage
   survive an installer that renames or prefixes the skill.
   Project-scoped tracking writes the record in the project, and
   user-scoped tracking writes it in the user's MLflow configuration.
   When both define the same harness-local skill name, the project
   entry wins.
2. Enable tracing for the harness:
   ```bash
   mlflow autolog claude
   ```
3. Run the agent. The autologger identifies skill activations in the
   recorded conversation and, using the install record, links the
   trace to the skill version and annotates the activation span. It
   recognizes activations by the harness's own signals where they
   exist (a dedicated skill tool call, a slash-command invocation)
   and otherwise by observing a tool call that reads a skill's
   `SKILL.md` from its installed location. Tool calls that use a skill's bundled
   files, such as a script under the skill's directory, are annotated
   as skill usage by the same location matching. No registry call
   happens during the run, so there is no added latency and no
   runtime dependency on registry availability.
4. Open the MLflow UI and navigate to the Traces page. Linked skills
   appear on each trace, and annotated spans carry the coordinates.

A skill present in the harness but not tracked has nothing recorded
for it and produces no link.
Nothing about the run fails in that case: the agent runs normally,
other autologging is unaffected, and the developer can still link
explicitly. The same is true of a missing or unreadable install
record. Before linking, the autologger also validates the installed
content against the digest recorded at install time; on a mismatch
it logs an error and records no link, and the validation result is
cached locally so content is not re-hashed on every run.

This journey applies in full to harnesses whose tracing integration
is provided by MLflow. Some harnesses instead trace themselves
through native OpenTelemetry export, with MLflow as the receiver. The
receiving server cannot read an install record on the harness's
machine, so linkage on this path requires the coordinates to reach
the server inside the spans themselves. Two routes do that. The
first is the attribute contract shown in the first journey, set by
instrumentation on the host that can read the record. The second is
a best-effort fallback: tracking appends a machine-readable marker
carrying the coordinates to the skill body, and the server
recognizes the marker inside captured LLM input at ingestion,
creating the link only when the marker's coordinates resolve in the
registry and its digest matches the version's. Marker-derived links
carry the content-marker provenance, and the route works only when
the harness captures LLM content in its spans, which several
harnesses leave off by default. Digest validation excludes the
marker line from hashing. MLflow does not create additional spans on
the harness's behalf.

#### Measure adoption of a registered skill

A platform owner wants to know whether a skill is being used, and which
versions are in play.

1. Query traces linked to the skill:
   ```python
   traces = mlflow.search_traces(
       locations=[experiment_id],
       filter_string="skill = 'code-review/1'",
   )
   ```
   Leaving the version off lists traces across all versions of the
   skill and shows the version spread.
2. **UI path:** open the skill's registry detail page. A "Related
   traces" link opens the Traces page filtered to that skill, and the
   version detail page does the same for a single version. The Traces
   page shows linked skills on each row.

The filter follows the precedent of the existing `prompt` filter for
prompt-to-trace links, and so does the storage behind it: a skill link
is a lineage record associating the trace with the skill version,
the same mechanism prompt links use, and the filter is an exact-match
query over those records. Span attributes mark where activation
happened but are not the query path. Links created at ingestion from
attributes or content markers produce the same lineage record, so
every provenance is reached by the one query. A skill in a named
organization is qualified as in the registry's URI form,
`skill = '@acme/code-review/1'`. The name-only and
organization-qualified forms are extensions this RFC proposes;
the prompt filter accepts only the `name/version` form. Exact
matching matters here; substring matching over trace
content would match `code-review` inside `code-review-strict` and
version `1` inside version `10`, making an adoption count wrong
rather than approximate.

#### Assess the impact of a deprecated or vulnerable skill version

A skill owner is about to deprecate a version and needs to know who is
still on it. A security engineer has learned a skill version is
susceptible to prompt injection and needs to know which runs were
exposed. Both are the same query: find the traces linked to an
affected version.

1. Find recent traces linked to the affected version:
   ```python
   traces = mlflow.search_traces(
       locations=experiment_ids,
       filter_string="skill = 'code-review/1'",
       order_by=["timestamp_ms DESC"],
   )
   ```
   The query names the locations to search; the result is only as
   complete as the locations the organization traces into.
2. Group the results by experiment to see which teams and applications
   produced them. For the security case, this is the exposure set:
   the runs in which the vulnerable version was active.
3. Notify those consumers, then transition the version:
   ```bash
   mlflow skills update-version skills:/code-review/1 --status deprecated
   ```
4. Re-run the query after the migration window to confirm that traffic
   on the affected version has stopped.

When the same content has been re-imported under several version
numbers, the affected traces span all of those versions. RFC-0008
indexes a skill's versions by content digest, so a registry lookup by
digest yields every version that shares the content, and the trace
query then covers those versions (see
[Open questions](#open-questions) on digest-based linking).

The evidence is retrospective: it shows what has run, not what is
installed and idle. A consumer that has the version installed but has
not exercised it since tracing was enabled does not appear.

#### Evaluate and compare skill versions on a benchmark

A team updates a skill and wants to know whether agent quality or cost
changed, connecting evaluation results to the skill versions that were
active.

1. Register the updated content, producing a new version:
   ```bash
   mlflow skills register git --name code-review \
       --url https://github.com/acme/agent-skills.git \
       --ref v2.0.0 --subpath code-review
   ```
2. Run the benchmark suite before and after the change. Both sets of
   traces carry skill links, so which version was active in each run
   is recorded rather than inferred from when the run happened.
3. Run evaluation against the collected traces:
   ```python
   results = mlflow.genai.evaluate(
       data=traces_df,
       scorers=[correctness_scorer, helpfulness_scorer],
   )
   ```
   Each row in `results.result_df` includes a `trace_id`. Reading a
   trace's linked skills connects any evaluation result back to the
   skill versions that were active in its run, and querying traces by
   skill (as in the adoption journey) walks the same lineage in the
   other direction.
4. Compare the two versions' runs on the same scorers, including cost
   per run: the linked traces carry their usual token and cost
   metrics, and because the benchmark holds the workload constant,
   the skill version is the variable.
5. If the new version is an improvement, promote it:
   ```bash
   mlflow skills set-alias skills:/code-review \
       --alias production --version 2
   ```
6. **UI path:** the experiment page lists the skill versions linked
   from the experiment's traces alongside its other linked entities,
   such as prompts. When comparing evaluation runs, the
   comparison view shows a diff of the linked skill versions, so a
   quality change can be read against exactly what changed in the
   skill configuration.

#### Compare skill versions on production traffic

A team promotes an updated skill version to production and wants to
know whether success metrics changed, using the traffic the agent
already serves rather than a benchmark run.

1. Promote the new version:
   ```bash
   mlflow skills set-alias skills:/code-review \
       --alias production --version 2
   ```
   Traces recorded after the promotion link to version 2; earlier
   traces link to version 1. The links partition traffic by the
   version that actually ran within the application's single
   production location, which stays exact even when a rollout is
   gradual or both versions serve concurrently, where a
   split-by-deploy-time would misattribute runs.
2. After enough traffic accumulates, retrieve each version's traces
   from the production location with the adoption journey's query,
   one query per version.
3. Score each version's traces with the same scorers, producing one
   evaluation run per version in the same experiment, or compare
   assessments already collected on the production traces.
4. Compare the two evaluation runs side by side. The comparison view
   shows the diff of linked skill versions alongside the metric
   deltas, so the change in outcomes is read against the change in
   skill configuration.

Production comparison trades rigor for reach. The samples are
unpaired, so statistical significance requires more data than a
paired benchmark comparison; production traffic is what supplies that
volume. The input mix can also shift between the two periods, and
that risk is accepted as a cost of evaluating on production data.
The benchmark journey above is the controlled complement.

### Out of scope

- **Installing skills into harnesses.** This RFC records where a skill
  was installed; it does not perform the installation. An MLflow
  installer with package manager integration could follow later and
  would have to write the same install record.
- **Skill-level cost attribution.** Without a delimited region for a
  skill's influence, tokens cannot be attributed to a skill. Cost is
  compared per run across skill versions in the evaluation journeys.
- **Tracing of non-skill plugin members.** Linking traces to agents,
  hooks, and other agent plugin members is deferred to
  [RFC-0010: Extended Agent Plugins](https://github.com/mlflow/rfcs/pull/27),
  which reuses the mechanism defined here.
- **Filtering evaluation results by skill directly.** Evaluation
  results reach skill versions through their traces' links; a direct
  filter on evaluation results is not added.
- **Server-side digest verification.** Digests remain client-asserted
  as in RFC-0008; a verification job with a verified-digest flag is
  registry-side follow-up work.

# Detailed design

This section is deliberately high level. It fixes the shapes that
other work depends on (the link, the SDK method, the attribute
contract, the install record, the query forms) and leaves internals
to the implementation.

## Link model

A skill link is a lineage record associating a trace with a skill
version, stored and queried through the same entity-association
mechanism as prompt links. A record carries:

- the trace id;
- the skill version's identity: `workspace`, `organization`, `name`,
  and `version`, with the association id in the URI form
  `[@organization/]name/version`;
- a `provenance` value: `explicit` (set by application or
  OpenTelemetry instrumentation), `resolution` (recorded by a
  framework autologger from the in-process mapping), `install_record`
  (recorded by a harness autologger from the install record), or
  `content_marker` (inferred by the server from a marker in captured
  LLM input).

There is at most one record per trace and skill version. When more
than one route produces the same link, the record keeps the more
authoritative provenance; `content_marker` is the least
authoritative, and the other three are treated as equal.

Independently of the record, the span on which activation was
observed is annotated with the attributes below, and tool spans that
use a skill's bundled files are annotated with the same attributes
plus `mlflow.skill.role = "usage"`. Annotation is for inspection in
the trace view; it is not a query path.

## SDK method and attribute contract

```python
class Span:
    def link_skill(
        self,
        *,
        name: str,
        version: int | None = None,
        alias: str | None = None,
        organization: str = "",
        cache_ttl_seconds: float | None = None,
    ) -> None: ...
```

Exactly one of `version` and `alias` is given. An alias is resolved
through the registry and the concrete version is what the link
records. Alias resolutions are cached per process with a short
expiry, following the prompt registry's alias cache: the default TTL
comes from `MLFLOW_ALIAS_SKILL_CACHE_TTL_SECONDS` (60 seconds), a
per-call `cache_ttl_seconds` overrides it, and `0` bypasses the
cache. The workspace is the caller's current workspace context. The
method writes the annotation attributes on the span and the lineage
record on the span's trace with provenance `explicit`.

Instrumentation that does not use the MLflow SDK sets the same
attributes directly, and MLflow creates the lineage record from them
when it receives the span:

| Attribute | Value |
|---|---|
| `mlflow.skill.name` | registered skill name |
| `mlflow.skill.version` | registered version (integer) |
| `mlflow.skill.workspace` | workspace name |
| `mlflow.skill.organization` | organization, empty string when none |
| `mlflow.skill.digest` | content digest, when known |
| `mlflow.skill.role` | `activation` (default) or `usage` |

These names are a public contract. Whether they should live in a
vendor-neutral namespace is an open question.

## Queries

The `skill` filter on `search_traces` matches lineage records by
exact match, following the `prompt` filter:

- `skill = 'code-review/1'`: one version;
- `skill = '@acme/code-review/1'`: one version in a named
  organization;
- `skill = 'code-review'`: any version of the skill;
- `skill.provenance = 'install_record'`: qualifies by provenance, for
  example to exclude `content_marker` links from an exposure count.

Digest queries resolve through the registry: RFC-0008's digest index
yields the versions sharing a digest, and the trace query covers
those versions.

## Install record

The install record maps skills installed in a harness to registry
coordinates. It is written by `mlflow skills track` and read by the
harness autologger. The project-scoped record is `mlflow-skills.json`
at the project root; the user-scoped record has the same name in the
MLflow user configuration directory. A project entry takes precedence
over a user entry with the same harness-local name.

```json
{
  "record_version": 1,
  "tracking_uri": "https://mlflow.example.com",
  "skills": {
    "code-review": {
      "harness": "claude-code",
      "path": "/home/dev/.claude/skills/code-review",
      "workspace": "default",
      "organization": "",
      "name": "code-review",
      "version": 3,
      "digest": "sha256:..."
    }
  }
}
```

`mlflow skills track <skill-uri> --path <dir> --harness <name>
[--user]` resolves the URI (aliases are resolved and never stored),
verifies the content at the path against the version's registered
digest and refuses on mismatch, appends the content marker to the
skill body, and writes the entry. `mlflow skills untrack` removes an
entry. The record contains no credentials; the tracking URI is
recorded so that a reader knows which registry the coordinates refer
to.

## Autologger behavior

**Harnesses with MLflow-provided tracing** (Claude Code, Codex, Qwen
Code, OpenCode). The autologger loads the install records, then for
each skill activation it observes in the recorded conversation it
validates the installed content against the recorded digest (the
result is cached locally), and on success annotates the activation
span and writes the lineage record with provenance `install_record`.
Activations are recognized by the harness's own signals where they
exist (a dedicated skill tool call, a slash-command invocation) and
otherwise by a tool call that reads a tracked skill's `SKILL.md`.
Tool calls whose inputs reference paths under a tracked skill's
directory are annotated as usage. On a digest mismatch the autologger
logs an error and records nothing for that skill. A missing or
unreadable record disables linking and nothing else.

**Agent frameworks with MLflow autologgers.** `mlflow.genai.pull`
accepts a skill URI (`skills:/name@alias` or `skills:/name/version`)
as its first argument, so the call's shape names the entity, in
addition to the keyword form RFC-0008 specifies, which is retained
because it is already being implemented. Either form records, in
process, the mapping from each pulled skill's location to its
coordinates and digest. The framework autologger consults that
mapping when it observes an activation (a skill-loading tool call, or
a read of a pulled `SKILL.md`), annotates the span, and writes the
record with provenance `resolution`. Frameworks that MLflow traces
only by receiving their OpenTelemetry output are handled as
receivers below.

**OpenTelemetry receivers** (Gemini CLI, Goose, OpenHands, Google
ADK). At ingestion, a received span carrying the attribute contract
produces a lineage record with provenance `explicit`. As a fallback,
the server scans captured LLM input for the content marker
(`<!-- mlflow-skill: {...} -->` as the last line of a tracked skill's
body) and produces a record with provenance `content_marker` only if
the marker's coordinates resolve in the registry and its digest
matches the version's. Markers are matched only in LLM input, never
in output, and duplicate matches within a trace produce one record.
This route works only when the harness captures LLM content in its
spans. Digest verification everywhere excludes the marker line from
hashing.

## UI

These are the surfaces the journeys need. How linked entities are
presented (for example, one consolidated lineage view across
prompts, skills, and other assets rather than a tab per asset type)
is aligned across assets outside this RFC.

- Trace view: linked skill versions listed among the trace's linked
  entities, each linking to its registry detail page; annotated spans
  show the coordinates in the span detail; unresolvable coordinates
  show a "not found in registry" indicator.
- Skill and version detail pages: a "Related traces" link opening
  the Traces page filtered by that skill or version.
- Experiment page: skill versions linked from the experiment's traces
  listed alongside its other linked entities.
- Run comparison: a diff of linked skill versions alongside the
  metric deltas.

# Drawbacks

- **A manual tracking step for harness users.** Without an MLflow
  installer, each installed skill must be tracked once by hand. Digest
  verification at tracking time keeps this honest, but it is a step
  users can forget, and untracked skills are silently unlinked.
- **The marker mutates installed content.** Appending a marker to
  `SKILL.md` changes the file on disk and requires every digest check
  to exclude that line. It is also visible to the model, and the
  fallback depends on harness telemetry settings that are commonly
  off by default.
- **Alias links can lag a repoint** for up to the cache TTL, the same
  trade-off prompt aliases make.
- **A public attribute contract** commits MLflow to the
  `mlflow.skill.*` names for as long as non-MLflow instrumentation
  writes them.

# Alternatives

## A SKILL span that parents the work

Modeling skill activation as a `SKILL` span whose children are the
LLM and tool spans produced while the skill is active was considered
and rejected. A span asserts an operation with a meaningful start and
end. Skill activation is broadly observable, but the end of a
skill's influence frequently is not: the Agent Skills specification
defines activation as a one-way load with no counterpart, and
harnesses commonly keep the loaded content in context for the rest
of the session. Parenting spans under a SKILL span therefore asserts
a containment the instrumentation cannot accurately record, and it
cannot represent concurrent activations.

## Attribute-based search instead of lineage records

Querying traces by span attributes was considered and rejected. Span
attributes are stored inside serialized span content, and the
existing attribute filter is substring matching over that content,
which cannot give the exact-match semantics an adoption or exposure
count needs; structured attribute search would be new store work for
a worse result than the lineage mechanism prompt links already use.

## A module-level linking call

A module-level `mlflow.genai.link_skill()` acting on the current
trace and span was considered and rejected in favor of the span
method, because a module-level call in the `genai` namespace does not
say whether it links a trace, a span, or an active evaluation run.

## An MLflow installer in this RFC

Specifying harness installation and package manager integration here
was considered and rejected. It would re-open the installation design
that RFC-0008 deferred, inside a tracing RFC, and the install record
plus a tracking command give the autologger what it needs without it.

# Adoption strategy

New feature, not a breaking change. Existing traces are unaffected;
existing framework autologgers gain skill linking without user
changes once a skill is resolved through `mlflow.genai.pull`. This
RFC delivers `Span.link_skill`, the attribute contract, lineage
records and the `skill` filter, the install record and `mlflow
skills track`, skill recognition in the Claude Code, Codex, Qwen
Code, and OpenCode tracing integrations and in framework
autologgers, attribute and marker recognition at OpenTelemetry
ingestion, and the UI surfaces above. RFC-0010 reuses the link model
for non-skill plugin members. An MLflow installer that writes the
install record is possible follow-on work but not committed.

# Open questions

- **OTel alignment.** The explicit journey shows a plain
  OpenTelemetry path that sets `mlflow.skill.*` attributes, from
  which MLflow records the link. That makes the attribute names part
  of the public contract rather than an implementation detail. Is
  that the right trade, and should the attribute names be namespaced
  differently if they are to be set by non-MLflow instrumentation?
- **Digest-based linking.** Digest queries resolve through the
  registry: RFC-0008 indexes a skill's versions by content digest, so
  grouping traces by content is a registry lookup followed by a trace
  query over the resulting versions. That index is scoped within a
  skill name, and the digest is client-asserted rather than
  server-verified. Should digest grouping also be supported across
  skill names, which would require a broader index?
