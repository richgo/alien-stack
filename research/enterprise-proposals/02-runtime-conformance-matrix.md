# Proposal 02: Runtime Conformance and Certification Matrix

Status: Draft enterprise proposal  
Date: 2026-06-12  
Finding addressed: Alien Stack lacks a runtime conformance matrix proving that the same verified artifact runs across serverless container shims and cloud Wasm targets.

## Problem

Current CI validates demo behavior, proof discharge, and effect lint, but not runtime portability. The README says native demos target x86_64 Linux (`README.md:117`), while the UI kit targets browser Wasm (`demo/ui-kit/spec.md:53-58`). There is no matrix proving a component runs under Wasmtime, runwasi/containerd, SpinKube, wasmCloud, Cloudflare Workers where feasible, or serverless-container wrappers.

Enterprise buyers need clear answers:

- Which runtimes are certified?
- Which are experimental?
- What exact artifact digest was tested?
- Which capabilities were granted?
- Which tests passed or failed?
- Which runtime versions are supported?

## Runtime support tiers

### Tier 1: Certified

Blocking release gates. The runtime is tested on every main/release build.

- Wasmtime direct
- One OCI/component packaging path
- One local host adapter

### Tier 2: Supported adapter

Nightly or release-candidate gates. Failures block release tags but not every PR.

- runwasi/containerd
- Spin and SpinKube
- wasmCloud
- Google Cloud Run wrapper
- AWS Lambda wrapper
- Azure Container Apps wrapper

### Tier 3: Compatibility preview

Best-effort, non-blocking. Clear caveats required.

- Cloudflare Workers, because Workers supports Wasm but documents WASI as experimental with only some syscalls implemented: <https://developers.cloudflare.com/workers/runtime-apis/webassembly/>.
- Docker Desktop Wasm, documentation-only/dev only, because Docker marks Wasm workloads deprecated: <https://docs.docker.com/desktop/features/wasm/>.

## External runtime evidence

- **Wasmtime** supports WebAssembly, WASI, and the Component Model, and emphasizes configurability, correctness/security, and standards compliance: <https://docs.wasmtime.dev/introduction.html>.
- **runwasi** facilitates Wasm workloads managed by containerd directly or via kubelet/CRI and shows use of `io.containerd.wasmtime.v1`: <https://github.com/containerd/runwasi>.
- **SpinKube** combines Spin Operator, containerd shim Spin, runtime class manager, and CLI tooling for Kubernetes Wasm workloads: <https://www.spinkube.dev/>.
- **Spin** is an event-driven microservice framework for Wasm components with local, self-hosted, Kubernetes, and cloud-hosted implementations: <https://spinframework.dev/v3/index>.
- **wasmCloud** runs Wasm components across Kubernetes, cloud, datacenter, or edge, with OCI registry distribution and deny-by-default capability sandboxes: <https://wasmcloud.com/docs/>.
- **Cloud Run** requires Linux x86_64 container images, HTTP listening on `0.0.0.0:$PORT`, and Docker/OCI image format support: <https://docs.cloud.google.com/run/docs/container-contract>.
- **AWS Lambda** supports container images/custom runtimes that implement the Lambda runtime API and run on Linux with `/tmp` as writable storage: <https://docs.aws.amazon.com/lambda/latest/dg/images-create.html>.
- **Azure Container Apps** provides serverless containers, ingress, KEDA scaling, jobs, Dapr, secrets, revisions, and traffic splitting: <https://learn.microsoft.com/en-us/azure/container-apps/overview>.

## Proposed conformance harness

Directory layout:

```text
conformance/
  matrix.yaml
  schemas/runtime-conformance.v1.json
  probes/
    http-probe.ll
    capability-probe.ll
    storage-probe.ll
  adapters/
    wasmtime/
    runwasi/
    spin/
    spinkube/
    wasmcloud/
    cloudrun/
    lambda/
    azure-container-apps/
    cloudflare-workers/
  reports/
```

Execution flow:

1. Build verified component from LLVM IR.
2. Seal component and record digest.
3. For each runtime:
   - install/pin runtime version;
   - deploy component or wrapper;
   - run smoke tests;
   - run capability-denial tests;
   - run HTTP request tests;
   - run telemetry/log capture tests;
   - collect startup, memory, latency, and trap metrics;
   - emit a JSON report.
4. Aggregate into `runtime-conformance-matrix.json`.

## Matrix schema

```json
{
  "schema": "alienstack.runtime-conformance.v1",
  "component_digest": "sha256:...",
  "wit_world": "alien:runtime/server@0.1.0",
  "generated_at": "2026-06-12T00:00:00Z",
  "runtimes": [
    {
      "name": "wasmtime",
      "tier": "certified",
      "version": "pinned",
      "platform": "linux/amd64",
      "adapter": "direct",
      "status": "pass",
      "capabilities": ["wasi:http", "alien:logging", "alien:metrics"],
      "tests": {
        "instantiate": "pass",
        "http_request": "pass",
        "capability_denial": "pass",
        "telemetry": "pass",
        "resource_budget": "pass"
      },
      "metrics": {
        "cold_start_ms": 0,
        "rss_mb": 0,
        "p95_ms": 0
      },
      "evidence": {
        "log": "reports/wasmtime/log.txt",
        "json": "reports/wasmtime/report.json"
      }
    }
  ]
}
```

## Runtime-specific acceptance tests

### Wasmtime direct

- Component instantiates under pinned Wasmtime.
- Imports match selected WIT world.
- HTTP handler returns expected response.
- Denied capability traps or returns a declared error.
- Fuel/timeout/memory limits are enforced.

### runwasi/containerd

- OCI artifact uses `wasi/wasm` platform where applicable.
- RuntimeClass maps to the intended shim.
- Pod starts and exits/serves as expected.
- No Linux container fallback is used for the Wasm profile.

### Spin/SpinKube

- Spin manifest references the component digest.
- Spin app serves HTTP through Spin.
- SpinKube `SpinApp` deploys with expected executor/runtime class.
- Scaling and cold-start evidence are captured.

### wasmCloud

- Component is pushed to OCI.
- wasmCloud host starts it with explicit capabilities.
- Deny-by-default behavior is verified.
- Logs/metrics use the operations interfaces.

### Cloud Run wrapper

- Wrapper listens on `0.0.0.0:$PORT`.
- Component digest is pinned in image metadata.
- No persistent filesystem dependency exists.
- Health and request timeout behavior matches Cloud Run contract.

### AWS Lambda wrapper

- Wrapper implements Lambda runtime API.
- Invocation event maps to WIT handler.
- Writable state uses `/tmp` only when explicitly granted.
- Cold start and timeout are reported.

### Azure Container Apps wrapper

- Container app revision pins component digest.
- Ingress/jobs/KEDA mode is declared.
- Secrets/config are capability-bound.
- Traffic splitting and rollback test pass.

### Cloudflare Workers preview

- Only compatible modules are tested.
- No unsupported WASI requirements are present.
- Binary size/startup caveats are recorded.

## CI strategy

- **PR gate:** Tier 1 Wasmtime direct only.
- **Main branch gate:** Tier 1 plus packaging validation.
- **Nightly:** Tier 2 local/cluster/cloud adapters.
- **Release candidate:** Full matrix; any Tier 1/Tier 2 failure blocks release.
- **Tier 3:** Report-only.

## Release policy

A release may claim:

- `portable-l1`: component builds and WIT imports validate.
- `portable-l2`: component builds, imports validate, and PCF/effect/proof gates pass.
- `runtime-certified`: Tier 1 passes.
- `runtime-supported`: Tier 1 and selected Tier 2 targets pass.

No runtime should be listed as "supported" without a current matrix row bound to the release component digest.

## Risks

- **Cloud CI cost/flakiness.** Keep full cloud tests nightly/release-candidate, not every PR.
- **WASI instability.** Pin runtime versions and WIT package versions.
- **Native POSIX semantic gap.** Do not certify native demos as portable; add component probes.
- **Cloudflare limitations.** Keep Workers in preview until required imports are known compatible.

## Acceptance criteria

- `conformance/` harness exists and emits schema-validated reports.
- Wasmtime direct is a blocking PR/release gate.
- At least two Tier 2 runtimes have repeatable nightly reports.
- Release notes include matrix summary and links to raw reports.
- Evidence bundle references the runtime conformance report by digest.
