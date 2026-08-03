# Deferred Tracing Content for RFC-0008

This file preserves all tracing-related content removed from
RFC-0008 PR #26. This content will be re-added in a follow-on PR
on the same RFC. Content is organized by source file and location.

---

## From: 0008-mvp-skill-registry.md

### Summary paragraph (was after the plugin import paragraph)

Trace integration supports both manual and automatic instrumentation.
`mlflow.skill_context()` lets SDK applications create SKILL spans
explicitly. For installed skills, the Claude Code autologger
uses the install-time trace manifest to create SKILL spans automatically.
Both paths annotate spans with registry coordinates, enabling adoption
tracking, deprecation impact analysis, per-skill cost attribution, and
regression detection.

### Motivation > The problem > Item 3: "No trace-to-skill linkage"

3. **No trace-to-skill linkage.** MLflow already traces agent
   conversations (Claude Code via `mlflow autolog claude`, SDK
   applications via framework autologgers such as
   `mlflow.langchain.autolog()` and `mlflow.anthropic.autolog()`).
   These traces capture LLM calls, tool use, and token consumption,
   but there is no way to know which governed, versioned skill was
   active during any part of a trace. This RFC introduces
   `mlflow.skill_context()` for manual instrumentation and an
   install-time trace manifest for automatic instrumentation via
   harness autologgers (see [Trace integration](#trace-integration)).
   Without a registry, organizations cannot answer questions like
   "which skill versions are most used?" or "show me all traces where
   the deprecated code-review version 1 was loaded."

### User journey: Install a skill bundle, run the agent, browse traces

#### Install a skill bundle, run the agent, browse traces

1. Install the bundle using a package manager plugin:
   ```bash
   mlflow skills bundles install --skill-uri skills:/pr-workflow@production \
       --harness claude-code
   ```
   This resolves the bundle from the registry, pulls the bundle
   content, delegates to the configured package manager (e.g., APM
   or Lola) for harness-specific installation, and writes a trace
   manifest (`mlflow-skills-manifest.json`) with installed registry
   coordinates. A single skill uses the same package-manager layer:
   ```bash
   mlflow skills install --skill-uri skills:/code-review@production \
       --harness claude-code
   ```
2. Ensure MLflow tracing is enabled for the target harness
   (e.g., `mlflow autolog claude` for Claude Code).
3. Run the agent. The harness loads the installed skills during a
   conversation.
4. Open the MLflow UI and navigate to the Traces page. Click the
   "Skills" tab to filter for traces with SKILL spans.
5. Find the trace for the agent run. Skill invocations appear as
   SKILL spans in the trace tree, annotated with registry coordinates
   (skill name, version, workspace).
6. Click a SKILL span to see which registered skill version was used
   and how long it took. Click the skill name link to navigate to the
   skill's registry detail page.

### User journey: Trace skill lineage to evaluation results

#### Trace skill lineage to evaluation results

After running evaluations, a user wants to know which registered
skill version was active during a traced agent run. The lineage
path flows through traces: evaluation results link to traces, and
traces contain SKILL spans annotated with registry coordinates.

1. Run an agent with installed skills. Skill invocations produce
   SKILL spans in the recorded traces (see
   [Trace integration](#trace-integration)).
2. Run evaluation against the collected traces:
   ```python
   results = mlflow.genai.evaluate(
       data=traces_df,
       scorers=[correctness_scorer, helpfulness_scorer],
   )
   ```
   Each row in `results.result_df` includes a `trace_id` linking
   the evaluation result back to its source trace.
3. Find which skill versions were used in a specific evaluation
   result:
   ```python
   trace = mlflow.get_trace(trace_id)
   skill_spans = trace.search_spans(span_type="SKILL")
   for span in skill_spans:
       print(span.attributes["mlflow.skill.name"],
             span.attributes["mlflow.skill.version"])
   ```
4. Find all traces that used a specific skill version:
   ```python
   traces = mlflow.search_skill_traces(
       experiment_ids=[experiment_id],
       skill_name="code-review",
       skill_version=1,
   )
   ```
   Evaluation results for these traces can then be retrieved via
   their `trace_id` values.

This two-step approach (query traces by skill attributes, then
retrieve associated evaluation results) works with the skill
trace query API and existing MLflow evaluation APIs. Filtering
evaluation results directly by skill version (without the
intermediate trace lookup) remains future work.

### UI > Detail view: skills > "Related traces" bullet

- **Related traces**: link to the GenAI Traces page filtered by this
  skill's name, showing recent SKILL spans that reference this skill

### UI > Trace integration display subsection

#### Trace integration display

The GenAI Traces page includes a "Skills" tab alongside the existing
"Prompts" tab, showing SKILL spans for each trace. The trace detail
view displays SKILL spans with registry coordinates (skill name,
version, workspace) and links to the skill's registry detail page.
Skill version detail pages surface related traces using the same
association data.

### Trace integration section (entire)

### Trace integration

MLflow already traces agent conversations across multiple frameworks:
Claude Code (via `mlflow autolog claude`), SDK applications (via
framework autologgers such as `mlflow.langchain.autolog()` and
`mlflow.anthropic.autolog()`), and others. These traces capture LLM
calls, tool use, and timing as a tree of spans. The skill registry
closes the observability loop by letting agent developers indicate
which registered skill is active during each part of a trace.

#### `mlflow.skill_context()` context manager

The primary instrumentation API is a context manager that creates a
span of type `SKILL` and attaches registry coordinates as span
attributes:

```python
with mlflow.skill_context(
    name="code-review", version=1
) as span:
    # All spans created inside this block (including those from
    # autologgers) become children of this SKILL span.
    result = llm.chat([{"role": "user", "content": "Review this code..."}])
```

The context manager creates a span with `mlflow.skill.name`,
`mlflow.skill.version`, and `mlflow.skill.workspace`
attributes that link the span back to a specific skill version in
the registry. See [implementation-details.md: skill_context() span
attributes](implementation-details.md#skill_context-span-attributes)
for the full attribute table.

#### Skill stacks via nesting

Skills can invoke other skills. Nesting `skill_context()` calls
produces a skill stack in the trace tree:

```
+-- Span: "code-review" (type: SKILL, version: 1)
|   +-- Span: ChatCompletion (type: LLM)
|   +-- Span: "style-check" (type: SKILL, version: 1)
|   |   +-- Span: ChatCompletion (type: LLM)
|   +-- Span: ChatCompletion (type: LLM)
```

Walking up the ancestor chain and collecting SKILL spans reconstructs
the skill stack for any span.

#### Framework autologger compatibility

Because `skill_context()` creates a standard MLflow span, it works
with existing framework autologgers without modification. When a
framework autologger (LangChain, OpenAI, Anthropic, etc.) creates a span
inside a `skill_context()` block, that span automatically becomes a
child of the SKILL span.

#### Automatic harness instrumentation

This RFC extends the Claude Code autologger to recognize skill
invocations and create SKILL spans automatically. The autologger reads
the `mlflow-skills-manifest.json` written during installation, maps the
harness-local skill name to its registered `{workspace, name, version}`
coordinates, and creates a SKILL span around the invocation. LLM and
tool spans produced while the skill is active become children of that
span.

Automatic instrumentation runs in the process that owns the active
trace, so it can preserve correct parent-child relationships without
cross-process trace correlation. It does not perform a registry lookup
during invocation. A missing, malformed, or unmatched manifest entry
does not interrupt the agent run or other autologging; it only prevents
creation of a registry-linked SKILL span for that invocation.

The manifest and instrumentation contract are harness-neutral so other
harness autologgers can adopt them later, but Claude Code integration is
the automatic tracing implementation delivered in this RFC. See
[implementation-details.md: Automatic trace
instrumentation](implementation-details.md#automatic-trace-instrumentation).

#### Registry validation

`skill_context()` does not validate that the named skill exists in
the registry at call time. Validating on every invocation would add
latency and create a hard dependency on registry availability. The
trace records the `{workspace, name, version}` coordinates
regardless; the MLflow UI performs a best-effort lookup when
displaying traces and shows a "not found in registry" indicator if
the coordinates do not resolve.

#### Workspace resolution

When `skill_context()` is called, the workspace is resolved from
the `mlflow-skills-manifest.json` written by the install commands.
The manifest contains the workspace for each installed skill.
For SDK users calling `skill_context()` directly without a manifest,
the workspace defaults to the current MLflow tracking URI's workspace
context, consistent with other MLflow operations.

#### Relationship to MCP trace linking

The MCP Registry (RFC-0004) uses after-the-fact, trace-level
association (`link_mcp_server_versions_to_trace()`). Skills use
span-level, inline annotation because skills are ambient (active
during inference) and can nest. Both approaches produce trace
metadata that the MLflow UI can display together.

#### Trace queries

`mlflow.search_skill_traces()` provides first-class keyword
arguments for common skill trace lookups:

```python
traces = mlflow.search_skill_traces(
    experiment_ids=[experiment_id],
    skill_name="code-review",
    skill_version=1,
)
```

The function filters to traces with `SKILL` spans matching the
given `mlflow.skill.*` attributes. The `filter_string` parameter
can be combined with the keyword filters for additional
constraints such as trace status or tags.

### Trace manifest subsection (under Package manager integration)

#### Trace manifest

Both installation commands write an `mlflow-skills-manifest.json` file
that records installed registry coordinates. The Claude
Code autologger consumes this manifest for automatic SKILL span
creation:

```json
{
  "manifest_version": "1.0",
  "skills": {
    "code-review": {
      "name": "code-review",
      "version": 1,
      "workspace": "default"
    },
    "style-check": {
      "name": "style-check",
      "version": 1,
      "workspace": "default"
    }
  }
}
```

Project-scoped installs write the manifest at the project root.
User-scoped installs write it in the MLflow user configuration
directory. Project entries take precedence over user entries with the
same harness-local skill name. See [implementation-details.md:
Automatic trace
instrumentation](implementation-details.md#automatic-trace-instrumentation)
for discovery, matching, span lifecycle, and failure behavior.

### Drawbacks > "Automatic tracing coverage" bullet

- **Automatic tracing coverage.** Automatic instrumentation is
  implemented for Claude Code. Other harnesses can use manual
  `skill_context()` instrumentation until their autologgers adopt the
  manifest contract.

### Alternatives > "Use APM or Lola directly" > "Trace integration" bullet

- **Trace integration.** No connection between installed skills and
  runtime execution traces. No way to answer "which skill version
  was active during this agent run?"

### Interleaved references (not standalone sections)

These phrases/sentences were edited or removed when deferring tracing.
Listed here so the follow-on PR can restore them.

**Main RFC (0008-mvp-skill-registry.md):**

- **Summary governance list**: was "lifecycle management, usage
  analytics via traces, and federated discovery across sources."
  Changed to "lifecycle management and federated discovery across
  sources." Restore "usage analytics via traces" when tracing returns.
- **User journey intro**: added a qualifier saying evaluation
  journeys use existing MLflow tracing and that registry-specific
  trace linkage comes in a follow-on PR. Remove this qualifier when
  tracing is restored.
- **Evaluate two bundle versions intro**: removed "Because skill
  invocations produce traced SKILL spans, LLM judges can analyze how
  skills were used during an agent run." Restore.
- **Compare agent performance, step 3**: removed "Skill invocations
  appear as SKILL spans in the traces." Restore.
- **Compare agent performance, step 5**: removed "The SKILL spans in
  experiment B's traces confirm which skill version was active during
  each run, enabling attribution of any quality differences." Restore.
- **Status table, active row**: was "Surfaced to discovery, traces,
  consumers". Changed to "Surfaced to discovery and consumers."
  Restore "traces".
- **Import paragraph**: removed "generate a downstream manifest, or"
  from the negative statement. Restore.
- **Package manager intro**: was "MLflow owns registry and source
  resolution plus the trace manifest." Changed to "MLflow owns
  registry and source resolution." Restore trace manifest reference.
- **Uninstall description**: removed "and clean up manifest entries"
  from the sentence about uninstall commands. Restore.
- **Bundle install flow, step 4**: removed the manifest-write step
  ("MLflow writes mlflow-skills-manifest.json...") and renumbered.
  Re-add as a step when tracing returns.
- **Single-skill install flow**: removed "and writes the trace
  manifest using the harness-local name returned by the plugin after
  installation succeeds." Restore.
- **Alternatives "Use Git alone"**: was "status lifecycle,
  trace-to-skill linkage, and federated discovery." Changed to
  "status lifecycle and federated discovery." Restore.
- **Alternatives "Use APM or Lola directly" closing**: was
  "governance, discovery, and observability layer." Changed to
  "governance and discovery layer." Restore "observability".
- **Alternatives "Use APM or Lola directly" intro**: was "governance
  and observability features." Changed to "governance features."
  Restore "observability".
- **Adoption strategy deliverables**: removed
  "`mlflow.skill_context()` for manual trace integration, the
  install-time trace manifest, automatic SKILL spans in the Claude
  Code autologger, and `mlflow.search_skill_traces()` for querying
  traces by skill attributes." Restore these deliverables.
- **Future improvements**: added a "Trace integration" bullet saying
  it is part of MVP scope and will be added as a separate PR. Remove
  this bullet when tracing is restored to the RFC body.

---

## From: implementation-details.md

### Interleaved references (implementation-details.md)

- **Intro line**: was "...package manager plugin interface, and
  trace integration details." Changed to "...package manager plugin
  interface." Restore.
- **Deletion semantics**: was "...explain historical traces and
  bundle snapshots..." Changed to "...explain historical bundle
  snapshots..." Restore "traces and".
- **CLI table, uninstall rows**: removed "and remove its manifest
  entry" from both `mlflow skills uninstall` and `mlflow skills
  bundles uninstall` descriptions. Restore.
- **Import paragraph**: removed "generate a downstream manifest, or"
  from the negative statement. Restore.
- **Plugin interface description**: removed "so MLflow can write an
  accurate trace manifest even when the package manager renames or
  prefixes skills" and "before MLflow writes its trace manifest or
  resolution lock" (changed to just "resolution lock"). Restore.
- **Bundle install flow, step 4**: removed manifest-write step and
  renumbered. Re-add when tracing returns.
- **Single-skill install flow, step 4**: removed manifest-write step
  and renumbered. Re-add when tracing returns.

### search_skill_traces() function definition

```python
def search_skill_traces(
    *,
    experiment_ids: list[str] | None = None,
    skill_name: str | None = None,
    skill_version: int | None = None,
    workspace: str | None = None,
    filter_string: str | None = None,
    max_results: int = 100,
    order_by: list[str] | None = None,
    page_token: str | None = None,
) -> PagedList[Trace]:
    """Search for traces containing SKILL spans matching the given
    skill attributes. The skill_name, skill_version, and workspace
    parameters provide exact-match filtering on the corresponding
    mlflow.skill.* span attributes. Additional filters can be
    combined via filter_string, which follows the standard MLflow
    filter syntax and is AND-ed with the keyword filters."""
```

Example usage line:
```python
traces = mlflow.search_skill_traces(skill_name="code-review", skill_version=1)
```

### search_skill_traces() explanation paragraph

`search_skill_traces()` filters to traces containing `SKILL` spans
with matching `mlflow.skill.name`, `mlflow.skill.version`, and
`mlflow.skill.workspace` attributes, so callers do not need to know
the span attribute naming convention. The `filter_string` parameter
composes with the keyword filters for additional constraints (trace
status, tags, etc.). Because skill identity is recorded as span attributes (not in a
separate association table like MCP trace linking), implementing
exact-match queries requires the store layer to support structured
matching on span attribute values. The existing
`span.attributes.*` filter path uses text search against
serialized JSON content, which does not provide exact-match
semantics. The implementation will need to enhance span attribute
querying to support this.

### Package manager plugin interface trace manifest references

Original lines about trace manifest in plugin interface:
- "accurate trace manifest even when the package manager renames or prefixes skills"
- "before MLflow writes its trace manifest or resolution lock"

### skill_context() span attributes section (entire)

## skill_context() span attributes

The `skill_context()` context manager creates a span with the
following attributes:

| Attribute | Value | Description |
|---|---|---|
| `mlflow.skill.name` | Skill name | Registry name of the active skill |
| `mlflow.skill.version` | Version number | Registered version (integer) |
| `mlflow.skill.workspace` | Workspace name | Resolved from the install manifest, falling back to the current tracking URI's workspace context |

These three attributes form the `{workspace, name, version}`
coordinates that link the span back to a specific skill version in
the registry.

### Automatic trace instrumentation section (entire)

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
registered embedded skill resolved through its member URI `#subpath`
fragment. For an
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
