---
name: trace-audit-record
description: Trace an audit RecordId around an approximate time through a configured pipeline. Choose consumer 1 (default, NRTFE plus Audit DR plus Spark), 2 (also Audit CDPDR), or 3 (also NRTDI). Explain missing record events with correlated diagnostics and source clues, distinguishing receipt from forwarding.
---

# Trace an audit record

Use this skill to investigate whether an audit record passed through a pipeline
and produce a timestamped, evidence-backed account of its progress.

This public version contains no organization-specific service endpoints,
namespace mappings, deployment names, or internal log examples. Obtain those
settings from approved configuration available in the investigation environment.
Do not upload private configuration or investigation results to this repository.

## Inputs

Required:

- RecordId, normally a GUID.
- Approximate time, including a date and timezone.

Optional:

- Consumer option: `consumer=1|2|3`; defaults to `1`.
- Expected organization or tenant ID.
- Environment, region, deployment, or workload.
- Known request IDs, correlation IDs, batch IDs, or destinations.

Example:

> Trace RecordId <record-guid> around 2026-01-15 18:35 UTC using the configured
> audit pipeline.

Append `consumer=2` or `consumer=3` to select an additional downstream consumer.

Ask for a missing date or ambiguous timezone before searching. Convert times to
UTC. Do not silently inherit an environment from unrelated conversation history.

An expected tenant ID is not required to start. Discover observed tenant IDs
from record-bearing evidence, keeping them separate from any supplied value.

## Consumer options

| Parameter | Components to trace |
| --- | --- |
| `consumer=1` (default) | NRTFE + Audit DR + Spark |
| `consumer=2` | NRTFE + Audit DR + Spark + Audit CDPDR |
| `consumer=3` | NRTFE + Audit DR + Spark + NRTDI |

Resolve the option before discovery or searching. If omitted, use **1 for this
invocation**; do not inherit an earlier selection. A clearly named path may be
mapped to its corresponding option. Ask for clarification for values outside
1-3, multiple selections, or a conflict between a numeric option and named path.

Option 1 stops at Spark. Option 2 adds only Audit CDPDR. Option 3 adds only
NRTDI. Query only the selected components; other stages are out of scope, not
missing or failed. Do not query optional branches "just in case" or require
their configuration for a different option.

Exception: when explaining a missing selected stage, a narrowly scoped diagnostic
pivot through another branch is allowed if needed to identify the shared
message. Explain why it is relevant; label it supporting evidence, not an
additional selected stage or proof of the missing stage's processing.

NRTFE denotes the ingestion frontend, Audit DR the input router, and Spark the
stream processor. Audit CDPDR and NRTDI are separate downstream consumers.
Resolve each from approved configuration; similar names are not interchangeable.

For an explicitly requested comparison, reuse common-prefix evidence only for
the same record, environment, scope, and time window. Report each option's
selected stages and conclusion separately.

## Environment configuration

Use an approved local configuration or explicit context to establish the stage
mapping for the selected option:

| Logical stage | Required configuration |
| --- | --- |
| NRTFE / ingestion frontend | Endpoint, namespace/table, record event, identity filters |
| Audit DR / input router | Endpoint, namespace/table, diagnostic or record event, identity filters |
| Spark / stream processor | Endpoint, namespace/table, structured record event, identity filters |
| Audit CDPDR (option 2 only) | Its own endpoint, namespace/table, record event, identity filters |
| NRTDI (option 3 only) | Its own endpoint, namespace/table, record event, identity filters |

The options select investigation scope, not proof of actual record routing.
Honor the configured topology and observed branches. Never substitute one
downstream consumer's mapping or evidence for the other.

For each stage, identify the available fields for:

- Record identifiers and tenant identifiers.
- Event time, precise processing time, and record-creation time.
- Deployment identity, application identity, processing state, and destination.
- Request, correlation, batch, or message identifiers.

Use only authorized tools and approved configuration. Do not guess endpoints,
namespace names, deployment aliases, or relationships between services. If the
mapping is unavailable, ask for it rather than scanning arbitrary services.
Once a mapping is available, routine requests need only a RecordId and time.

## Tool prerequisites

1. Discover the available log-query tools and read their actual schemas.
   Provider prefixes and parameter names may vary.
2. Read any provider-supplied search, discovery, and investigation guidance.
   Follow current limits rather than assuming this document overrides them.
3. Use existing authentication and consent. Report an authentication blocker
   accurately and direct the user to the supported sign-in flow. Never expose
   credentials or launch parallel login attempts.
4. Inspect the live columns, supported query language, identity names, and
   relevant identity candidates before querying each event or table.

Metadata can be stale or incomplete. Candidate membership is not proof of
traffic in the incident window. A missing candidate does not invalidate an
explicit incident-backed value for a verified identity column.

## Investigation procedure

### 1. Bound the time range

Start with the five-minute UTC block containing the approximate time, for
example `[18:35:00, 18:40:00)` for a time around 18:35.

After a completed, successful empty search, expand to 15 minutes, then 30
minutes, then at most one hour per query, subject to provider limits. Expanded
windows should include time both before and after the rough time.

Do not widen because of a timeout, query error, failed data partitions, or
missing identity coverage. Address that limitation first. Record exact query
boundaries and use nonoverlapping windows when splitting a search.

### 2. Establish discriminating scope

Prefer the deployment, region, or tenant scope supported by incident evidence.
Distinguish a deployment-scoping identity from a customer tenant field; the
same label can have different meanings in different systems.

When no deployment is known, discover relevant candidates and query bounded
batches within provider limits. Record which batches were searched and which
remain unsearched. Never describe a partial candidate batch as global coverage.

Avoid wildcard or unscoped scans. If no practical scope can be established,
ask for a region or deployment before launching an expensive global search.

### 3. Find record-bearing evidence

Start at the earliest configured stage with a structured record event, unless
existing evidence offers a more discriminating starting point.

Search the exact RecordId first, without a tenant predicate, so an unexpected
tenant remains visible. Use the actual field and language supported by the
event version. Account for GUID casing and null values where necessary.

Schema metadata can combine columns from multiple versions. Do not reference
every possible alias in one expression unless the query engine handles absent
columns safely. If a query fails, simplify to a verified field/version.
An unsupported query is not evidence that the record is absent.

For every match, extract:

- Exact matched RecordId and its position if the row contains a list.
- Associated tenant, workload, operation, and record category when available.
- Processing/event time and record-creation time as separate values.
- Deployment, role, application, and processing state.
- Observed routing destinations and correlation identifiers.

For parallel arrays, associate values by their documented positions. A tenant
elsewhere in a multi-tenant batch is not evidence of the target record's tenant.

If the supplied tenant differs, report the mismatch immediately. Further work
must be labeled a RecordId-only or observed-tenant trace, not confirmation of
the requested tenant/RecordId pair.

### 4. Follow observed routing

Follow only the components selected by the resolved consumer option.

Use each matching row to narrow downstream scope. A record can move to another
deployment or region. Discover matching downstream identities rather than
assuming every stage uses the same deployment.

A destination name is a routing hint, not proof that its consumer processed
the record. Request IDs are not automatically downstream correlation IDs.
Follow only relationships supported by logs, configuration, or applicable
source code.

For routers, distinguish parsing, attempted sending, and successful sending.
For processors, seek a structured processing/completion event rather than
inferring completion solely from upstream or downstream observations.

Fetch full text when an inline sample clips the matching identifier or tenant
association. Preserve only relevant excerpts, not entire unrelated batches.

For option 2, inspect Audit CDPDR's own record and diagnostic events. For option
3, do the same for NRTDI using its independently configured mapping. Collect the
selected consumer's record/tenant identity, precise time, processing action,
destination, and relevant request or correlation identifiers. A match in one
consumer does not confirm the other. Do not infer either consumer's passage
solely from Spark completion.

### 5. Interpret timestamps and success conservatively

Use the field documented as the precise event or processing time. A generic
timestamp can be an ingestion time or coarse time bucket. When fields differ,
show both values and explain which supports the timeline.

Never mix record-creation time with processing time when calculating stage
latency. State uncertainty about timestamp precision or clock alignment.

To claim send success, obtain an explicit success event or verify that the
applicable logging call occurs after an awaited send succeeds. Distinguish
current source behavior from a verified deployed revision.

Check where a processor emits its completion event relative to serialization
and the output writer. A "Processed" event can precede the actual send.
Do not treat payload-storage fields, such as inline versus blob location, as
consumer-routing selectors without explicit source/configuration evidence.

Do not equate an outbound send with downstream durability, indexing, or
customer searchability without separate evidence.

### 6. Explain missing record events

Before concluding that a selected stage is not confirmed, inspect its
diagnostic events as well as its record-completeness event. An empty RecordId
search is an observation to explain, not a cause or proof of non-receipt.

1. Establish whether the stage was expected to receive this record using the
   observed destination and applicable topology, record type, operation, and
   routing configuration. Selection of a consumer alone does not establish this.
2. Pivot from an exact-record match to its local processing correlation, then
   capture the message ID, payload correlation, tenant, and source position.
   Search the missing stage for those observed message identifiers. Processing
   correlation IDs can change between consumers or retries. Partition/sequence/
   offset tuples are not globally unique; verify the source and message link.
   Tenant/time proximity alone is not a record-to-message association.
3. If an initial successful search is empty, expand according to the time-range
   rules. Downstream receipt may occur outside the initial five-minute block.
   When a message is found, use its actual time and local correlation to retrieve
   receive, parse, filter, send, retry, and error diagnostics in a thin slice.
4. Compare input membership with post-filter and sender counts. A received
   message with zero records passed to the sender differs from non-receipt or
   send failure. Aggregate counts alone do not identify which record was removed
   from a multi-record batch. Zero post-filter counts do not prove a specific
   policy blocked it. An informational blob/token message is not a failure
   without a corresponding error.
5. Where needed, trace the applicable code from consumer initialization through
   parsing, filters, sender, and record-event emission. Check the correct
   deployment flavor, allow/block list, feature flags, and logging/sampling
   behavior. Cite repository, path, line range, and commit. Current configuration
   or code is a clue, not proof of the deployed version or runtime flag value.
6. Classify the result as confirmed non-routing, confirmed filtering,
   processing/send failure, received but forwarding unconfirmed, coverage/
   logging limitation, or unresolved. Call a cause confirmed only when correlated
   runtime evidence establishes it. Otherwise separate the observed outcome
   from the code-backed hypothesis and name the missing verification.

For each missing or partially confirmed stage, report the exact diagnostic
excerpt and UTC time, correlation chain, searched scope/window and query
coverage, classification, source clues, and remaining uncertainty. Preserve
unsuccessful checks too. If no reason can be established, say so explicitly.
Do not invent a filtering, routing, or loss explanation to fill the gap.

## Query lifecycle and coverage

- Prefer bounded searches of 20-50 matching rows, subject to provider limits.
  Run independent discovery or stage searches in parallel when supported.
- Preserve query IDs, exact endpoints, predicates, identity filters, and time
  bounds. Query IDs alone are not durable evidence.
- A timeout may leave a query running. Resume its status and result retrieval
  rather than blindly submitting the same expensive search again.
- Fetch useful partial evidence before cancelling a running query. Do not
  poll a disposed query or an early-cancelled search whose results are inline.
- Respect pagination limits and inclusive/exclusive index semantics.
- Inspect completion, failed bytes/partitions, result limits, clipped cells,
  and scope before interpreting an empty result.
- Worker counts and scan-order samples are not global traffic totals or
  statistically representative samples.
- Preserve partial findings if later queries fail. A query failure is an
  investigation limitation, not a pipeline processing failure.
- Say "not confirmed in the searched scope/window," not "did not pass."

## Deliverable

Lead with the conclusion: which stages are confirmed and whether the observed
tenant matches any expected tenant. State the environment, RecordId, and
selected consumer (including whether it defaulted to 1), and searched UTC time
range. Limit a full-path conclusion to the selected components.

Use a compact evidence table for the three common stages:

| Stage | Status | Precise UTC timestamp | Observed action and evidence |
| --- | --- | --- | --- |
| NRTFE | Confirmed / not confirmed / blocked | Time or unavailable | Event, matched identifiers, routing |
| Audit DR | Confirmed / not confirmed / blocked | Parse/send times | Record membership and send result |
| Spark | Confirmed / not confirmed / blocked | Processing time | Processing state, application, stage |

Append an Audit CDPDR row only for option 2, or an NRTDI row only for option 3.
Option 1 has three rows; options 2 and 3 each have four. Never count unselected
components as missing or failed.

Use a more precise status where supported, such as "received; forwarding not
confirmed" or "filtered (confirmed)"; do not reduce these to a binary result.
Put any alternate-branch diagnostic pivot below the selected-stage table.

Explicitly label tenant mismatches. Explain significant timestamp differences
and distinguish directly observed actions from inferred relationships.
Do not manufacture timestamps, tenant mappings, service URLs, or end-to-end
success.

When supported, preserve a JSON artifact in approved session storage containing:

- Requested inputs and observed tenant identity.
- Resolved `consumer`, `consumerDefaulted`, and selected components.
- Per-stage timestamps with their source field names.
- Minimal relevant evidence excerpts.
- Query IDs, endpoints, predicates, scope, and time bounds.
- Completion state, coverage limitations, and unresolved blockers.
- For a missing record event: diagnostic evidence and correlation chain,
  observed outcome, classification, confirmed versus hypothesized cause,
  source references/revision, and unverified deployment or runtime settings.

Attach or link the artifact through supported session tools. Keep credentials,
unrelated customer data, and private configuration out of public repositories.
