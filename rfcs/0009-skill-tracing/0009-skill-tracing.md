# RFC 0009: Skill Tracing

| start_date   | 2026-08-23 |
| :----------- | :--------- |
| mlflow_issue | |
| rfc_pr       | https://github.com/mlflow/rfcs/pull/37 |

| Author(s)              | [Bill Murdock](https://github.com/jwm4) (Red Hat) |
| :--------------------- | :-- |
| **Date Last Modified** | 2026-08-23 |
| **AI Assistant(s)**    | Claude Code |

**Table of contents**

- [Summary](#summary)
- [Basic example](#basic-example)
- [Motivation](#motivation)
  - [The problem](#the-problem)
  - [User journeys](#user-journeys)
  - [Out of scope](#out-of-scope)
- [Detailed design](#detailed-design)
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

This RFC adds a `SKILL` span that carries registry coordinates
(workspace, organization, name, and version) for the skill that was
active while its child spans were produced, plus the version's content
digest when the instrumentation knows it. Skills are ambient rather
than discrete: they are loaded into an agent, influence many
inferences, and can invoke other skills. A span, which has a beginning,
an end, and a parent, models that better than an after-the-fact
trace-level association does. Nested skills produce a skill stack that
can be reconstructed by walking a span's ancestors.

Two terms recur below. A **harness** is a packaged agent application
that skills are installed into and that runs them without code written
by the user: Claude Code, Codex, Gemini CLI, OpenCode, Goose, and
OpenHands are examples. An **agent framework** is a library that
application code builds an agent with, where the
developer's own process loads the skill and drives the run: LangGraph,
Google ADK, the OpenAI Agents SDK, CrewAI, Pydantic AI, and Semantic
Kernel are examples.

Spans are produced along three paths:

- **Explicit instrumentation.** Application code that composes its own
  agent marks the region where a skill is active. This is the path for
  custom agents built on the MLflow SDK, and it is also expressible
  through plain OpenTelemetry for callers that are not using the MLflow
  tracing API directly.
- **Automatic instrumentation in agent frameworks.** When
  application code resolves a skill from the registry and hands it to a
  framework such as LangGraph or the OpenAI Agents SDK, MLflow holds
  the mapping from that skill to its registry coordinates in process,
  and the framework autologger annotates spans from it without the
  developer writing tracing code. Frameworks that MLflow traces by
  receiving their native OpenTelemetry output rather than by running
  instrumentation inside them are handled like the equivalent
  harnesses (see the harness journey).
- **Automatic instrumentation in harnesses.** When MLflow installs
  a skill into a harness such as Claude Code, it records which registry
  coordinates the harness-local skill came from. The harness autologger
  reads that record at run time to annotate spans.

Because the spans carry registry coordinates, traces become queryable by
skill. That query surface is what turns tracing into governance
evidence: adoption tracking, deprecation impact analysis, per-skill cost
attribution, and regression detection across skill versions.

**Relationship to other RFCs.** Skill tracing builds on
[RFC-0008: Skill Registry](https://github.com/mlflow/rfcs/blob/main/rfcs/0008-mvp-skill-registry/0008-mvp-skill-registry.md),
which defines the `Skill` and `SkillVersion` entities, the
`{workspace, organization, name, version}` coordinates, and the
client-asserted content `digest` that this RFC records on spans.
Because the harness path depends on the install-time record described
above, harness installation commands and the package manager
integration behind them are part of this RFC. The
[MCP Server Registry (RFC-0004)](https://github.com/mlflow/rfcs/blob/main/rfcs/0004-mcp-registry/0004-mcp-registry.md)
links MCP servers to traces after the fact and at trace level, which
suits discrete server invocations; skills use span-level inline
annotation for the reasons above, and both produce trace metadata the
UI can display together.

# Basic example

TBD.

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
   still on it. Promoting a version means asserting it is better, but
   there is no way to attribute a quality or cost difference to the
   skill rather than to everything else that changed. Retiring an
   unused skill means knowing it is unused.

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
workflows the resulting spans support.

#### Instrument a custom agent application

A developer building an agent directly on the MLflow SDK wants the
region of the run where a registered skill is active to be identifiable
in the trace.

1. Mark the region where the skill is active:
   ```python
   import mlflow

   with mlflow.genai.skill_context(name="code-review", version=1):
       # Spans created inside this block, including spans created by
       # framework autologgers, become children of the SKILL span.
       result = llm.chat([{"role": "user", "content": "Review this..."}])
   ```
   Because this creates an ordinary MLflow span, existing framework
   autologgers (LangChain, OpenAI, Anthropic, and others) need no
   modification: any span they create inside the block is parented to
   the `SKILL` span automatically.

   **OpenTelemetry equivalent:** a caller instrumenting with plain OTel
   sets the same attributes on its own span, and the resulting trace is
   indistinguishable to consumers:
   ```python
   from opentelemetry import trace

   tracer = trace.get_tracer("my-agent")
   with tracer.start_as_current_span("code-review") as span:
       span.set_attribute("mlflow.spanType", "SKILL")
       span.set_attribute("mlflow.skill.name", "code-review")
       span.set_attribute("mlflow.skill.version", 1)
       span.set_attribute("mlflow.skill.workspace", "default")
       span.set_attribute("mlflow.skill.organization", "")
   ```

2. Resolve the version from an alias rather than pinning it, so the
   trace records the version that actually ran:
   ```python
   from mlflow import MlflowClient

   version = MlflowClient().get_skill_version_by_alias(
       name="code-review", alias="production",
   )
   with mlflow.genai.skill_context(name=version.name, version=version.version):
       ...
   ```
   The recorded coordinates are always a concrete version. Aliases move,
   so a trace that recorded `@production` would become ambiguous the
   moment the alias was repointed.

3. Nest skills that invoke other skills. The trace tree shows the skill
   stack:
   ```
   +-- Span: "code-review" (type: SKILL, version: 1)
   |   +-- Span: ChatCompletion (type: LLM)
   |   +-- Span: "style-check" (type: SKILL, version: 1)
   |   |   +-- Span: ChatCompletion (type: LLM)
   |   +-- Span: ChatCompletion (type: LLM)
   ```
   Walking up the ancestor chain from any span and collecting the
   `SKILL` spans reconstructs the stack that was active for it.

4. **UI path:** open the trace in the MLflow UI. `SKILL` spans appear in
   the trace tree annotated with their registry coordinates, and the
   skill name links to the skill's registry detail page. When the
   recorded coordinates do not resolve (the version was deleted, or the
   trace came from a different workspace), the UI shows a "not found in
   registry" indicator rather than failing to render the span.

#### Trace skills loaded by an agent framework

A developer using an agent framework (LangGraph, the OpenAI Agents
SDK, and similar) resolves a skill from the registry and hands it to
the framework. They
should not have to also wrap their agent code in a tracing context to
get the linkage.

1. Resolve and pull the skill through MLflow, then hand it to the
   framework in whatever form the framework expects:
   ```python
   path = mlflow.genai.pull(
       name="code-review", alias="production", destination="./skills",
   )
   agent = build_agent(skills=[path])
   ```
2. Enable the framework autologger as usual. No manual context is
   needed: because the skill was resolved through MLflow, the
   autologger knows its registry coordinates and emits the `SKILL` span
   when the framework activates the skill.
3. Run the agent and open the trace. The result is the same trace shape
   as the explicit path: a `SKILL` span carrying registry coordinates,
   with the LLM and tool spans produced while the skill was active as
   its children.

This follows a pattern MLflow already uses for other registry
entities: resolving an entity records its identity, and tracing picks
that identity up automatically when related spans are created. Skill
activation is observable in the frameworks surveyed so far, most
directly where loading a skill is itself a tool invocation. The
per-framework mechanisms, and the convention for where the skill's
span begins and ends around a point-in-time activation, belong in the
detailed design.

#### Trace skills installed into a harness

A platform owner installs skills into a harness such as Claude Code,
where there is no application code to instrument and no in-process
resolution step.

1. Install the skill into the harness through MLflow:
   ```bash
   mlflow skills install --skill-uri skills:/code-review@production \
       --harness claude-code
   ```
   MLflow resolves the alias to a concrete version, delegates the
   harness-specific install to the configured package manager plugin,
   and records the resulting harness-local skill name against the
   registry coordinates that were installed. Recording it at install
   time is what makes the linkage survive a package manager that renames
   or prefixes the skill.
2. Enable tracing for the harness:
   ```bash
   mlflow autolog claude
   ```
3. Run the agent. When the harness loads an installed skill during a
   conversation, the autologger matches it against what was recorded at
   install time and opens a `SKILL` span around the invocation. No
   registry call happens during the run, so there is no added latency
   and no runtime dependency on registry availability.
4. Open the MLflow UI and navigate to the Traces page. The `SKILL` spans
   appear in the trace tree with their registry coordinates and link
   back to the skill detail pages.

A skill that was copied into the harness by hand, without an MLflow
install, has nothing recorded for it and produces no annotated span.
Nothing about the run fails in that case: the agent runs normally,
other autologging is unaffected, and the developer can still instrument
explicitly. The same is true of a missing or unreadable install record.

This journey applies in full to harnesses whose tracing integration is
provided by MLflow. Some harnesses instead trace themselves through
native OpenTelemetry export, with MLflow as the receiver; for those,
MLflow annotates the spans the harness already emits with skill
coordinates when the skill invocation is identifiable in them, and
does not create additional spans on the harness's behalf. The
receiving-side mechanics belong in the detailed design.

#### Measure adoption of a registered skill

A platform owner wants to know whether a skill is being used, and which
versions are in play.

1. Query traces by skill coordinates:
   ```python
   traces = mlflow.search_traces(
       locations=[experiment_id],
       filter_string="span.attributes.`mlflow.skill.name` = 'code-review'",
   )
   ```
2. Narrow to a specific version, or leave the version off to see the
   spread across versions:
   ```python
   traces = mlflow.search_traces(
       locations=[experiment_id],
       filter_string=(
           "span.attributes.`mlflow.skill.name` = 'code-review' "
           "AND span.attributes.`mlflow.skill.version` = 1"
       ),
   )
   ```
3. **UI path:** open the skill's registry detail page. A "Related
   traces" link opens the Traces page filtered to that skill, and the
   version detail page does the same for a single version.

This journey is the reason exact-match semantics on span attributes
matter. Substring matching against serialized span content would match
`code-review` inside `code-review-strict`, and would match version `1`
inside version `10`, which makes an adoption count wrong rather than
approximate. Skill trace queries therefore extend the existing
`search_traces` filter syntax rather than adding a skill-specific
search function, and exact matching on span attribute values becomes a
store-level requirement of this RFC.

#### Assess the impact of deprecating a skill version

A skill owner is about to deprecate a version and needs to know who is
still on it before doing so.

1. Find recent traces that used the version slated for deprecation:
   ```python
   traces = mlflow.search_traces(
       filter_string=(
           "span.attributes.`mlflow.skill.name` = 'code-review' "
           "AND span.attributes.`mlflow.skill.version` = 1"
       ),
       order_by=["timestamp_ms DESC"],
   )
   ```
2. Group the results by experiment to see which teams and applications
   are still producing them.
3. Notify those consumers, then transition the version:
   ```bash
   mlflow skills update-version skills:/code-review/1 --status deprecated
   ```
4. Re-run the query after the migration window to confirm that traffic
   on the deprecated version has stopped before deleting it.

The evidence is retrospective: it shows what has run, not what is
installed and idle. A consumer that has the version installed but has
not exercised it since tracing was enabled does not appear.

#### Attribute token cost to a skill

A platform owner wants to know what a skill costs to run.

1. Retrieve traces that used the skill, using the adoption query above.
2. Roll up token usage over each `SKILL` span's subtree. Because the
   `SKILL` span is the parent of the LLM spans produced while the skill
   was active, the subtree is the unit of attribution.
3. Compare cost per run across two versions of the same skill to see
   whether a change to the skill made runs more expensive.

Nested skills need a stated convention: an inner skill's tokens are
inside the outer skill's subtree, so a naive sum over all `SKILL` spans
double counts. Whether the rollup is exclusive (each span charged only
for spans it directly dominates) or inclusive is a design decision, not
a user choice.

#### Trace skill lineage to evaluation results

After running evaluations, a user wants to know which registered skill
version was active during a traced agent run. The lineage path runs
through traces: evaluation results link to traces, and traces contain
`SKILL` spans annotated with registry coordinates.

1. Run an agent with skills active, by any of the instrumentation
   paths.
   Skill invocations produce `SKILL` spans in the recorded traces.
2. Run evaluation against the collected traces:
   ```python
   results = mlflow.genai.evaluate(
       data=traces_df,
       scorers=[correctness_scorer, helpfulness_scorer],
   )
   ```
   Each row in `results.result_df` includes a `trace_id` linking the
   evaluation result back to its source trace.
3. Find which skill versions were active for a specific evaluation
   result:
   ```python
   trace = mlflow.get_trace(trace_id)
   for span in trace.search_spans(span_type="SKILL"):
       print(span.attributes["mlflow.skill.name"],
             span.attributes["mlflow.skill.version"])
   ```
4. Go the other direction, from a skill version to its evaluation
   results, by querying traces for that version and then retrieving the
   evaluation results for the returned `trace_id` values.

This two-step approach composes trace queries with existing evaluation
APIs. Filtering evaluation results directly by skill version, without
the intermediate trace lookup, is not part of this RFC.

#### Detect a regression after a skill update

A team updates a skill and wants to know whether agent quality changed,
attributing the difference to the skill rather than to everything else
that varies between runs.

1. Register the updated content, producing a new version:
   ```bash
   mlflow skills register git --name code-review \
       --url https://github.com/acme/agent-skills.git \
       --ref v2.0.0 --subpath code-review
   ```
2. Run the benchmark suite before and after the change. Both sets of
   traces carry `SKILL` spans, so which version was active is recorded
   rather than inferred from when the run happened.
3. Score both sets with the same scorers, then compare. The `SKILL`
   spans confirm which version was active in each run, and they support
   the comparison even when the runs were not segregated into separate
   experiments.
4. Compare by content rather than by version number when the same
   content has been re-imported under several versions. Automatic
   instrumentation records the version's content `digest` alongside
   the coordinates (the explicit path records it when the caller
   supplies it), so traces for byte-identical content group together
   even though their version numbers differ:
   ```python
   traces = mlflow.search_traces(
       filter_string="span.attributes.`mlflow.skill.digest` = '<sha256>'",
   )
   ```
   The digest is client-asserted and not server-verified, so this groups
   by asserted content identity rather than by a registry-guaranteed
   byte match. RFC-0008 indexes the digest within a skill name, so
   grouping across different skill names that happen to share content is
   a further question for the detailed design.
5. If the new version is an improvement, promote it:
   ```bash
   mlflow skills set-alias skills:/code-review \
       --alias production --version 2
   ```
   Subsequent traces record version 2, and the adoption query above
   shows the migration progressing.

### Out of scope

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

- **OTel alignment.** The journeys show a plain OpenTelemetry path that
  sets the same span attributes as the MLflow helper. That makes the
  attribute names part of the public contract rather than an
  implementation detail. Is that the right trade, and should the
  attribute names be namespaced differently if they are to be set by
  non-MLflow instrumentation?

- **Span exposure.** The journeys write
  `with mlflow.genai.skill_context(...):` without binding the span, so
  callers cannot mutate it after creation. Is there a use case that requires
  access to the span, and if so does it outweigh the risk of
  after-the-fact manipulation?


- **Install record location.** The harness journey depends on a
  record written at install time that maps harness-local skill names to
  registry coordinates. Is it per project, under a user-level
  configuration directory, or both with a precedence rule?
- **Digest-based linking.** `SKILL` spans are shown recording the
  content digest alongside `name` and `version`. On the automatic
  paths the instrumentation has the digest from resolution or install;
  on the explicit path `skill_context()` does not contact the registry,
  so the digest is absent unless the caller supplies it. Is
  best-effort recording acceptable, and should digest grouping be
  supported across skill names as well as within one, given that
  RFC-0008 indexes the digest within a skill name and the digest is
  client-asserted rather than server-verified?
