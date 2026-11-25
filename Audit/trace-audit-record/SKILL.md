---
name: trace-audit-record
description: Trace an audit RecordId around an approximate time through a configured ingestion pipeline. Collect precise timestamps, routing evidence, tenant identity, query references, and search-coverage limitations.
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

- Expected organization or tenant ID.
- Environment, region, deployment, or workload.
- Known request IDs, correlation IDs, batch IDs, or destinations.

Example:

> Trace RecordId <record-guid> around 2026-01-15 18:35 UTC using the configured
> audit pipeline.

Ask for a missing date or ambiguous timezone before searching. Convert times to
UTC. Do not silently inherit an environment from unrelated conversation history.

An expected tenant ID is not required to start. Discover observed tenant IDs
from record-bearing evidence, keeping them separate from any supplied value.

## Environment configuration

Use an approved local configuration or explicit context to establish the stage
mapping. A typical pipeline contains:

| Logical stage | Required configuration |
| --- | --- |
| Ingestion frontend | Endpoint, namespace/table, record event, identity filters |
| Input router | Endpoint, namespace/table, diagnostic or record event, identity filters |
| Stream processor | Endpoint, namespace/table, structured record event, identity filters |
| Output router | Endpoint, namespace/table, record event, identity filters |

These are logical roles, not a claim that every pipeline uses four stages or
processes them in this order. Honor the configured topology, including branches.

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

### 5. Interpret timestamps and success conservatively

Use the field documented as the precise event or processing time. A generic
timestamp can be an ingestion time or coarse time bucket. When fields differ,
show both values and explain which supports the timeline.

Never mix record-creation time with processing time when calculating stage
latency. State uncertainty about timestamp precision or clock alignment.

To claim send success, obtain an explicit success event or verify that the
applicable logging call occurs after an awaited send succeeds. Distinguish
current source behavior from a verified deployed revision.

Do not equate an outbound send with downstream durability, indexing, or
customer searchability without separate evidence.

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
searched UTC time range.

Use a compact evidence table adapted to the configured stages:

| Stage | Status | Precise UTC timestamp | Observed action and evidence |
| --- | --- | --- | --- |
| Ingestion frontend | Confirmed / not confirmed / blocked | Time or unavailable | Event, matched identifiers, routing |
| Input router | Confirmed / not confirmed / blocked | Parse/send times | Record membership and send result |
| Stream processor | Confirmed / not confirmed / blocked | Processing time | Processing state, application, stage |
| Output router | Confirmed / not confirmed / blocked | Event time | Destination and send evidence |

Explicitly label tenant mismatches. Explain significant timestamp differences
and distinguish directly observed actions from inferred relationships.
Do not manufacture timestamps, tenant mappings, service URLs, or end-to-end
success.

When supported, preserve a JSON artifact in approved session storage containing:

- Requested inputs and observed tenant identity.
- Per-stage timestamps with their source field names.
- Minimal relevant evidence excerpts.
- Query IDs, endpoints, predicates, scope, and time bounds.
- Completion state, coverage limitations, and unresolved blockers.

Attach or link the artifact through supported session tools. Keep credentials,
unrelated customer data, and private configuration out of public repositories.
