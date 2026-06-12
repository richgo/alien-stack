# Proposal 04: Enterprise Operations Contract

Status: Draft enterprise proposal  
Date: 2026-06-12  
Finding addressed: Alien Stack lacks an operational contract for telemetry, config, secrets, health, rollout/rollback, resource budgets, and incident response.

## Problem

Alien Stack currently focuses on build-time correctness and demo evidence. That is necessary but not sufficient for enterprise production. The repo has verification and evidence gates (`demo/storage/build.sh:49-72`; `demo/ui-kit/spec.md:53-58`) and benchmark claims (`README.md:195-217`), but it does not define operational capabilities: logs, metrics, tracing, health/readiness, config/secrets, admin controls, resource budgets, rollout/rollback, or incident runbooks.

Enterprise runtime adapters must not invent these behaviors independently. They should be narrow host capability interfaces, declared in WIT and reflected in PCF effects.

## Design principles

1. **Operations effects are declared effects.** Logging, metrics, tracing, config, secrets, health, and admin actions must appear in `pcf.effects`.
2. **Host adapters are narrow ABIs.** Cloud Run, Lambda, Azure Container Apps, Spin, wasmCloud, and browser shims adapt platform APIs into WIT capabilities; they do not contain application policy.
3. **Artifact digest is the rollout unit.** Rollback means reverting to a previous signed evidence bundle/component digest.
4. **Runtime budgets are contracts.** Memory, CPU/fuel, timeouts, file descriptors, body size, and startup latency are specified and tested.
5. **SLOs derive from evidence.** Benchmark and conformance reports feed operational targets.

## Host capability interfaces

### Logging

```wit
interface logging {
  enum level { debug, info, warn, error }
  log: func(level: level, message: string, fields: list<tuple<string, string>>);
}
```

Effects:

- `ops.log.write`

Requirements:

- JSON structured output.
- Include component digest, evidence bundle digest, runtime adapter version, request ID, trace ID, and severity.
- Never log secret values.

### Metrics

```wit
interface metrics {
  increment: func(name: string, value: u64, attrs: list<tuple<string, string>>);
  observe: func(name: string, value: f64, attrs: list<tuple<string, string>>);
}
```

Effects:

- `ops.metric.emit`

Required metrics:

- cold start time;
- request latency;
- memory high-water mark;
- trap count;
- capability-denied count;
- proof/evidence version labels;
- host adapter error count.

### Tracing

```wit
interface tracing {
  current-context: func() -> option<string>;
  span-start: func(name: string, attrs: list<tuple<string, string>>) -> u64;
  span-end: func(span: u64, status: string);
}
```

Effects:

- `ops.trace.emit`

Use OpenTelemetry as the common model. Host adapters should map platform trace context to component calls and export OTLP where available.

### Config and secrets

```wit
interface config {
  get: func(key: string) -> option<string>;
}

interface secrets {
  get-secret: func(key: string) -> result<list<u8>, string>;
}
```

Effects:

- `ops.config.read`
- `ops.secret.read`

Rules:

- No ambient environment reads in portable profile.
- Keys must be allowlisted in deployment policy.
- Secret reads are auditable and never logged.

### Health and admin

```wit
interface health {
  liveness: func() -> bool;
  readiness: func() -> bool;
  version: func() -> string;
}

interface admin {
  drain: func() -> result<_, string>;
  reload-config: func() -> result<_, string>;
}
```

Effects:

- `ops.health.read`
- `ops.admin.invoke`

## Cloud adapter responsibilities

Each adapter must:

- verify component digest at startup;
- load only declared capabilities;
- enforce resource budgets;
- map platform request/event to WIT world;
- map logs/metrics/traces to platform collector;
- expose readiness/liveness;
- propagate request IDs and trace context;
- surface capability-denied events;
- include evidence bundle digest in all operational metadata.

Platform specifics:

- **Cloud Run:** Listen on `0.0.0.0:$PORT`, respect startup/request timeout, avoid persistent filesystem assumptions (<https://docs.cloud.google.com/run/docs/container-contract>).
- **AWS Lambda:** Implement runtime API, use `/tmp` only for explicit temporary storage, report cold starts (<https://docs.aws.amazon.com/lambda/latest/dg/images-create.html>).
- **Azure Container Apps:** Support revisions, traffic splitting, secrets, KEDA scaling, and jobs where appropriate (<https://learn.microsoft.com/en-us/azure/container-apps/overview>).
- **wasmCloud:** Map capabilities to wasmCloud provider/interface model and preserve deny-by-default semantics (<https://wasmcloud.com/docs/>).
- **SpinKube:** Use SpinApp/SpinAppExecutor and runtime class conventions (<https://www.spinkube.dev/>).

## Runtime resource budgets

Define budgets in deployment policy:

```json
{
  "schema": "alienstack.runtime-budget.v1",
  "memory_max_mb": 64,
  "cold_start_p95_ms": 50,
  "request_timeout_ms": 1000,
  "body_max_bytes": 1048576,
  "fuel_max": 10000000,
  "capabilities": [
    "wasi:http.incoming",
    "ops.log.write",
    "ops.metric.emit"
  ]
}
```

The conformance matrix must verify these budgets for each supported runtime.

## Rollout and rollback

Rollout unit:

- component digest;
- wrapper image digest where used;
- WIT world digest;
- evidence bundle digest;
- runtime adapter version.

Deployment gate:

1. Verify signatures.
2. Verify evidence bundle policy.
3. Verify runtime conformance row for target platform.
4. Deploy to canary.
5. Monitor SLO and trap/capability-denial metrics.
6. Promote or rollback.

Rollback:

- switch traffic to previous digest/evidence bundle;
- do not roll back storage schema without a migration contract;
- require IPS migration proofs for stateful rollback.

## Runbooks

Minimum runbooks:

- verification gate failure;
- effect lint failure;
- Z3 proof discharge failure;
- unknown WIT import;
- component instantiation failure;
- capability denied in production;
- health/readiness timeout;
- OTel exporter backpressure;
- cold-start regression;
- IPS recovery failure;
- wrapper image CVE.

Each runbook should include symptoms, detection query, immediate mitigation, root-cause workflow, rollback criteria, and evidence to collect.

## SLOs

Initial SLOs should be conservative and evidence-driven:

- Availability: 99.9% for Tier 1 runtime services.
- Error rate: <0.1% non-2xx/non-declared failures.
- Latency: p95 target per component and runtime matrix row.
- Cold start: p95 target per runtime.
- Safety: zero undeclared capability successes; all denied capabilities are logged.
- Release integrity: 100% production deployments must reference signed evidence bundles.

## Implementation phases

1. **Operational WIT interfaces.** Add logging, metrics, tracing, config, secrets, health, and admin WIT files.
2. **Effect vocabulary.** Add `ops.*` atoms and require import/effect cross-checks.
3. **Adapter instrumentation.** Implement Wasmtime wrapper first, then cloud wrappers.
4. **OTel integration.** Emit structured logs and OTLP traces/metrics.
5. **Runbooks and dashboards.** Add docs and example dashboards.
6. **SLO gates.** Tie benchmark/conformance evidence to release gates.

## Acceptance criteria

- All operational host calls are WIT-defined and effect-declared.
- Every supported adapter emits structured logs, metrics, and traces with component/evidence digests.
- Readiness/liveness/version interfaces work across Tier 1/Tier 2 targets.
- Config/secrets are allowlisted capabilities, not ambient reads.
- Runtime budgets are enforced and reported.
- Rollout/rollback procedure is documented and tested.
- Required runbooks exist and are linked from release evidence.
