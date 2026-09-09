# Node.js/Next.js Flag API: Cheap Percentage Gates for US/EU Rollback Audits

Short answer: choose the smallest feature-flag control plane that records an immutable configuration revision, applies deterministic percentage gates, and lets a nightly customer-support pipeline return to the previous revision without changing application code. A cheap, simple API is useful only after those rollback properties are proven.

| Pick this control plane | Pick it when | Rollback unit | Main trade-off |
|---|---|---|---|
| Versioned file in the deployment | One team owns both the flag and the nightly job | Revert the deployment or file commit | Clear history, but an emergency change follows the deployment path |
| Small internal flag API | Several jobs need the same US/EU targeting rules | Move an alias to the prior immutable revision | More operational ownership: availability, authorization, audit retention, and caching are now yours |
| Managed flag service | Many teams need delegated changes, policy, and a mature operator workflow | Restore a prior configuration through the service | Greater capability and less infrastructure work, with a larger external dependency and contract to evaluate |

The decisive question isn't how many toggles a system can store. It is whether an operator can answer, from logs alone, which rule and revision admitted each batch. That turns rollback from a guess into a bounded operation.

## Make rollback deterministic before comparing options

Treat evaluation as a pure function of five inputs: flag key, immutable revision, stable subject key, normalized region, and rollout percentage. The output should include the boolean decision plus a reason code. Keep request time out of the function. Keep random numbers out too. Given the same inputs, a retry must land in the same cohort.

For this customer-support pipeline, the subject key can be a tenant ID rather than a ticket ID. Tenant-level bucketing prevents one tenant's tickets from being split across old and new transformations during the same nightly run. The region rule should execute before the percentage rule: an allowlist of `US` and `EU` first establishes eligibility, then the percentage gate selects a stable subset of eligible tenants.

Pin it.

A flag response should carry a revision such as `cfg_0187`, not only `enabled: true`. The pipeline pins that revision at the start of a run and uses it for every batch. If the control plane changes halfway through, the active run stays internally consistent; the next run may adopt the new revision. A rollback then means repointing the active alias to an earlier immutable revision and starting the next run with that known configuration. It does not mean editing the old object in place.

The API surface can stay small: read the active revision, read one immutable configuration, and promote a tested revision. The exact route names matter less than the behavior because this is a generic contract, not a vendor integration. Reads should be cacheable for a short, explicit period, while promotion needs authentication, authorization, and an audit record. Don't let an evaluator silently invent a default after its pinned configuration is unavailable; use the last validated copy for that revision or stop before processing a new batch.

## How should Node.js and Next.js teams pick a simple flag API for US/EU rollout?

A versioned file is the clean pick when flag changes can move at deployment speed. It has few moving pieces, reviews fit the existing code workflow, and the application can log the commit identifier as its configuration revision. Stick with this option when a separate operator console would add a second source of truth without giving the team a faster recovery path. The catch is that an urgent targeting change is now coupled to whatever approval and deployment path owns the file.

An internal API fits when several Node.js or Next.js workloads must consume the same rule and the team can own a small control service. The useful boundary is not `GET` versus `POST`; it is immutable configuration versus mutable activation. Store each configuration once, validate it before activation, and make the active pointer the only mutable state. This option is not suitable when the team cannot staff authorization reviews, backups, audit retention, and an availability objective for the control plane. In that case, the apparent simplicity moves operational work into the wrong team.

A managed service fits when change governance is the larger problem: many operators, delegated environments, approval policy, and a user interface for recovery. Evaluate it with a proof, not a feature matrix. Can it expose the evaluated revision and reason to your logs? Can a run pin a revision? Can an operator restore an earlier state without reconstructing rules by hand? I'm not sure a static vendor comparison can settle those questions because the answer depends on the selected plan, SDK mode, and integration design. A test in your actual request path will.

Cost belongs in the decision, but count the whole control loop: service charges, engineering ownership, on-call load, log volume, and the time needed to prove a rollback. A low request price cannot compensate for decisions that are impossible to reconstruct.

## Implement a reversible evaluator and log contract

The implementation below is intentionally narrow. It validates one immutable rule, normalizes a trusted region attribute, assigns a stable bucket with SHA-256, and emits one structured decision record per batch. It does not fetch configuration inside the evaluator. Fetch and validate the pinned revision once before the batch loop, then pass that object in.

```ts
import { createHash } from "node:crypto";

type Region = "US" | "EU";
type Reason = "REGION_EXCLUDED" | "PERCENTAGE_MATCH" | "PERCENTAGE_MISS";

type FlagConfig = Readonly<{
  key: string;
  revision: string;
  enabledRegions: readonly Region[];
  rolloutBasisPoints: number;
}>;

type Decision = Readonly<{
  enabled: boolean;
  reason: Reason;
  bucket: number | null;
}>;

function stableBucket(flagKey: string, tenantId: string): number {
  const digest = createHash("sha256")
    .update(`${flagKey}:${tenantId}`)
    .digest();
  return digest.readUInt32BE(0) % 10_000;
}

function evaluateFlag(
  config: FlagConfig,
  tenantId: string,
  region: Region,
): Decision {
  if (!config.enabledRegions.includes(region)) {
    return { enabled: false, reason: "REGION_EXCLUDED", bucket: null };
  }

  const bucket = stableBucket(config.key, tenantId);
  const enabled = bucket < config.rolloutBasisPoints;
  return {
    enabled,
    reason: enabled ? "PERCENTAGE_MATCH" : "PERCENTAGE_MISS",
    bucket,
  };
}

type BatchContext = Readonly<{
  runId: string;
  batchId: string;
  tenantId: string;
  region: Region;
}>;

function recordDecision(
  context: BatchContext,
  config: FlagConfig,
  decision: Decision,
): void {
  const record = {
    timestamp: new Date().toISOString(),
    severityText: "INFO",
    eventName: "feature_flag.evaluated",
    runId: context.runId,
    batchId: context.batchId,
    tenantId: context.tenantId,
    region: context.region,
    flagKey: config.key,
    flagRevision: config.revision,
    rolloutBasisPoints: config.rolloutBasisPoints,
    enabled: decision.enabled,
    reason: decision.reason,
    bucket: decision.bucket,
  };

  process.stdout.write(`${JSON.stringify(record)}\n`);
}

const pinnedConfig: FlagConfig = {
  key: "support-search-v2",
  revision: "cfg_0187",
  enabledRegions: ["US", "EU"],
  rolloutBasisPoints: 1_000,
};

const batch: BatchContext = {
  runId: "nightly-2026-08-12",
  batchId: "batch-0042",
  tenantId: "tenant-314",
  region: "EU",
};

const decision = evaluateFlag(pinnedConfig, batch.tenantId, batch.region);
recordDecision(batch, pinnedConfig, decision);
```

Basis points make boundaries explicit: `1_000` means 10%, and `10_000` means every eligible tenant. Validate the configuration before use so the value is an integer from 0 through 10,000, region entries belong to the accepted set, and the revision cannot be overwritten. The code logs the bucket for debugging, but dashboards should aggregate on low-cardinality fields such as flag key, revision, region, decision, and reason. Keep tenant and batch identifiers available for targeted search rather than turning each value into a metric label.

This record follows the useful shape described by the OpenTelemetry logs model: a timestamp, observed event data, severity, and attributes that provide context. OpenTelemetry also explains how logs can carry trace context, so add trace and span identifiers when the nightly pipeline already creates spans. Do not manufacture them just for appearance. The event name remains stable while the revision and outcome change, which gives a log query a durable anchor.

Severity needs restraint. A normal match or miss is informational, not a warning. RFC 5424 defines ordered severity levels, and using warning for every disabled decision makes the signal noisy. Reserve an error-level event for a batch that cannot proceed under its pinned and validated configuration. An ordinary `REGION_EXCLUDED` result is policy working as designed.

Before promotion, test boundaries rather than a handful of happy paths: 0 and 10,000 basis points, both accepted regions, a rejected region at input validation, repeated evaluation of the same tenant, and two flag keys for the same tenant. Then run a shadow evaluation against a production-shaped sample without changing processing. Compare decision counts by revision and region. Promote only after the distribution matches the intended rule.

The rollback drill is short. Pin revision A, promote revision B, prove that new runs report B, restore the active pointer to A, and prove that the following run reports A. Do this before the flag carries risk. A rollback mechanism first exercised during a bad nightly run is only a hypothesis.

## Search the logs before changing the flag

Start with the run ID and failed batch, then group `feature_flag.evaluated` records by `flagRevision`, `region`, `enabled`, and `reason`. If one run contains multiple revisions, the pipeline did not honor its pinning boundary. If only percentage matches show the changed behavior, the blast radius is bounded to that cohort. If both enabled and disabled cohorts changed together, investigate the shared pipeline path before touching the flag.

This is the crisp before and after to preserve: revision A establishes the baseline, revision B changes one rule, and the logs retain both decisions with the same event contract. Roll back the active pointer when B is implicated. Do not delete B; its immutable record is part of the explanation. After recovery, replay only when the pipeline's data contract makes replay safe. Feature flags control code paths, not the idempotency of downstream writes.

One sentence matters most: **a flag without its evaluated revision in the log is not rollback evidence.**

## Limits that should change the decision

This design handles basic regional targeting and percentage rollout. It deliberately does not solve legal residency classification, identity resolution, multivariate experiments, statistical analysis, or approval policy. Derive `US` or `EU` from an authoritative tenant attribute; don't infer a compliance region inside the evaluator from an unreviewed request field.

A small internal API is the wrong pick when browser-side evaluation would expose sensitive rules, when sub-millisecond local evaluation is mandatory without a validated cache, or when many non-engineers need governed changes. A deployment file is the wrong pick when rollback must be independent of deployment. A managed control plane is the wrong pick when its evaluation metadata cannot be exported into the log contract your operators search.

Choose the boundary you can test. Then keep the rule small, the revision immutable, and the decision visible.

## Sources

- https://opentelemetry.io/docs/concepts/signals/logs/
- https://datatracker.ietf.org/doc/html/rfc5424
