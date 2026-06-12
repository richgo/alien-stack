# How Alien Stack becomes enterprise-ready

Research date: 2026-06-12  
Repository: `superfield-ai/alien-stack`  
Scope: Enterprise maturity plan for Alien Stack, with emphasis on running across serverless container platforms, containerd/Wasm shims, Kubernetes, and cloud Wasm runtimes.

## Executive take

Alien Stack is promising as an agent-native research stack, but it is not yet an enterprise platform. The core idea is strong: keep behavior in a low-level canonical representation, attach machine-checkable contracts, declare effects, and fail builds when proofs or gates do not hold. The repository now demonstrates that thesis at small scale: docs describe LLVM IR plus PCF metadata and effect atoms (`README.md:30-50`), the storage demo is explicitly the L2 verification anchor (`demo/storage/spec.md:3-13`), and the build path now has real evidence gates for native demos and UI acceptance (`README.md:106-117`, `demo/ui-kit/spec.md:53-58`).

The biggest strategic change is that "enterprise Alien Stack" should stop treating native Linux binaries as the primary deployment artifact. It should treat **WebAssembly components plus WIT-defined capability surfaces** as the portable enterprise artifact, with LLVM IR retained as the agent/proof source representation. Native binaries can remain a performance target, but the product should converge on:

1. **LLVM IR source -> verified Wasm component -> OCI artifact -> runtime adapter.**
2. **WIT interfaces as the stable ABI**, not ad hoc libc/POSIX calls.
3. **A certification matrix** across Wasmtime, runwasi/containerd, SpinKube, wasmCloud, Cloudflare Workers where feasible, and fallback "Wasm-in-container" adapters for AWS Lambda, Google Cloud Run, and Azure Container Apps.
4. **A release evidence bundle** containing proof reports, effect reports, SBOM/provenance, SLSA/in-toto attestations, runtime conformance results, performance results, and signed artifact digests.

The enterprise version is therefore less "write raw IR web servers" and more "generate small, formally constrained Wasm components that run behind any enterprise host capability provider."

## Current repo evidence

### What is good enough to build on

- **Clear architecture and honest maturity model.** The README says the demos are proofs-of-concept, not production software (`README.md:12-15`), while the whitepaper defines staged conformance from structural metadata through linked/sealed and durable levels (`docs/alien-stack-whitepaper.md:421-433`).
- **Good enterprise instincts already exist.** The whitepaper calls for fail-closed verification, structural lint, contract extraction, proof check/discharge, link gate, and artifact seal (`docs/alien-stack-whitepaper.md:314-325`). That is exactly the right shape for enterprise governance.
- **Effect boundaries are first-class.** The README defines effect atoms as a declared, mechanically checked surface (`README.md:44-50`), and the shared effect lint now maps actual IR calls to accepted atoms and fails on undeclared observed effects (`tools/effect-lint.sh:1-28`, `tools/effect-lint.sh:211-236`).
- **The storage demo has real proof evidence.** It runs behavioral IPS evidence, Z3 proof discharge, and structural effect lint in the build (`demo/storage/build.sh:49-72`). The verifier fails if Z3 is absent unless `ALLOW_MISSING_Z3=1` is explicitly set (`demo/storage/verify-pcf.sh:33-56`), and the storage spec states all `check-sat` calls must return `unsat` (`demo/storage/spec.md:9-13`).
- **Artifact sealing has started.** The webserver build emits an `alienstack.artifact.v1` manifest with source/artifact digests and toolchain records (`demo/webserver/seal-artifacts.sh:1-7`, `demo/webserver/seal-artifacts.sh:106-149`).
- **The client direction already matches the Wasm future.** The isomorphic client whitepaper defines a narrow browser ABI, WASM module requirements, import allowlists, and artifact seals (`docs/isomorphic-web-whitepaper.md:122-156`, `docs/isomorphic-web-whitepaper.md:192-216`).

### What is not enterprise-ready

- **Deployment target is still mostly native Linux.** README states native demos target x86_64 Linux and use POSIX syscalls such as `pread`, `pwrite`, and `fork` (`README.md:117`). `demo/README.md` repeats that native demos target x86_64 Linux (`demo/README.md:8`). That is incompatible with "runs anywhere serverless" unless wrapped in a Linux container.
- **Wasm is client/demo-oriented, not the main server runtime.** The UI kit compiles to `wasm32-unknown-unknown` (`demo/ui-kit/build.sh:35-41`), but there is no WASI/component server build that exposes HTTP through standard WIT interfaces.
- **No component model or WIT package exists.** The repo defines an `alien-stack.client.abi.v1` in prose (`docs/isomorphic-web-whitepaper.md:122-144`), but there are no `.wit` packages, no component adaptation, no `wasm-tools component new`, and no adapter tests.
- **No runtime conformance matrix.** CI validates demos, but not "same artifact runs on Wasmtime, runwasi, SpinKube, wasmCloud, Cloud Run container, Lambda custom runtime, Azure Container Apps." Enterprise buyers will ask for exactly that matrix.
- **L3/L4 are still future work.** The conformance table says L3 "Linked and Sealed" and L4 "Durable" are future work (`docs/alien-stack-whitepaper.md:425-433`). Enterprise readiness needs L3 as table stakes for stateless services and L4 for stateful/durable workflows.
- **Artifact sealing is partial.** One demo has a manifest, but it is not a repo-wide, signed, schema-validated, CI-enforced release bundle. There is no SBOM, provenance attestation, vulnerability scan, policy bundle, or reproducible rebuild job.
- **Operational surface is missing.** There is no logging/metrics/tracing ABI, no health/readiness model, no config/secrets model, no IAM/capability policy story, no deployment manifests, no upgrade/rollback procedure, and no incident response hooks.

## External ecosystem evidence

The enterprise path should align with the standards and projects that are already becoming the common Wasm/cloud substrate:

- **WASI** is the standards-track system interface for Wasm applications that may run from browsers to clouds to embedded devices; it uses capability-based sandboxing where modules start with no ambient authority and only get host-granted capabilities. The WASI site lists P1/P2/P3 milestones and current runtimes such as Wasmtime, WAMR, WasmEdge, wazero, Wasmer, jco, and others. Source: <https://wasi.dev/>.
- **WASI 0.2 / Preview 2 and 0.3 / Preview 3** are defined in WIT and the Component Model direction. The WebAssembly/WASI repo says WASI 0.2 is modular APIs defined with WIT, and WASI 0.3 builds on it with component-model async via `future` and `stream`. Source: <https://github.com/WebAssembly/WASI>.
- **The WebAssembly Component Model** is the standardization path for portable, cross-language composition. The official repo states it is where the component model is standardized, with WIT, binary/text formats, canonical ABI, and future formal spec/test suite. Source: <https://github.com/WebAssembly/component-model>. Wasmtime describes the Component Model as a binary format for portable, cross-language composition and notes WASI is defined in terms of component model interfaces. Source: <https://docs.wasmtime.dev/introduction.html>.
- **Wasmtime** is a strong default enterprise runtime for this stack: it supports WebAssembly, WASI, and the Component Model; it is configurable for CPU/memory control; and it emphasizes correctness/security and standards compliance. Source: <https://docs.wasmtime.dev/introduction.html>.
- **runwasi/containerd** is the bridge to Kubernetes/containerd. The runwasi project says it facilitates running Wasm workloads managed by containerd directly or via kubelet/CRI, and includes Wasmtime runtime usage examples with `--runtime=io.containerd.wasmtime.v1`. Source: <https://github.com/containerd/runwasi>.
- **SpinKube** is the Kubernetes path for Spin-based serverless Wasm. Its site says it combines Spin Operator, containerd shim Spin, runtime class manager, and a Spin CLI plugin. Spin Operator introduces SpinApp/SpinAppExecutor CRDs, scaling, and runtime class configuration; the containerd shim lets Spin workloads run similarly to Pods. Source: <https://www.spinkube.dev/>.
- **Spin** is a Wasm component framework for event-driven microservices, with local, self-hosted, Kubernetes, and cloud-hosted implementations. Source: <https://spinframework.dev/v3/index>.
- **wasmCloud** is a broader distributed platform for Wasm components across Kubernetes, cloud, datacenter, or edge. Its docs state components target WASI, are pushed to OCI registries, can run on Kubernetes or standalone runtime hosts, and are deny-by-default sandboxes. Source: <https://wasmcloud.com/docs/>.
- **Cloudflare Workers supports Wasm but WASI is experimental.** Workers can run WebAssembly modules and entire Workers in Rust; threading is not supported, binary size impacts startup, and WASI support is experimental with only some syscalls implemented. Source: <https://developers.cloudflare.com/workers/runtime-apis/webassembly/>.
- **Cloud Run, Lambda, and Azure Container Apps are serverless-container targets, not native Wasm-component targets.** Cloud Run requires Linux x86_64 ABI executables inside a container image and HTTP listening on `0.0.0.0:$PORT`; it accepts Docker/OCI images. Source: <https://docs.cloud.google.com/run/docs/container-contract>. AWS Lambda supports container images and custom runtimes, but images must implement the Lambda runtime API, run on Linux, be single-architecture, and use `/tmp` for writable storage. Source: <https://docs.aws.amazon.com/lambda/latest/dg/images-create.html>. Azure Container Apps is serverless containers with ingress, Dapr, jobs, KEDA scaling, secrets, revisions, traffic splitting, and registry support. Source: <https://learn.microsoft.com/en-us/azure/container-apps/overview>.
- **Docker Desktop Wasm is not a production anchor.** Docker's Wasm workload docs explicitly say the feature is deprecated and will be removed in a future Docker Desktop release, even though the doc is still useful for understanding `platform: wasi/wasm` and runtime-class-style packaging. Source: <https://docs.docker.com/desktop/features/wasm/>.

## Target enterprise architecture

### 1. Make the deployable unit a verified component

Current: `.ll` -> native x86_64 binary for server demos; `.ll` -> core Wasm for browser UI.  
Target: `.ll` -> normalized LLVM module -> `.wasm` core module -> **component** -> signed OCI artifact.

The source can remain LLVM IR because that is the thesis and proof surface. But enterprise deployment should be a Wasm component because that is where standards, portability, and cloud runtimes are converging. The component gives Alien Stack a natural place to express:

- imported capabilities (`wasi:http`, `wasi:filesystem`, `wasi:clocks`, `wasi:random`, `wasi:keyvalue`, `wasi:blobstore`, `wasi:logging`, `wasi:config`, plus `alien:*` extensions);
- exported service interfaces (`handle`, `init`, `migrate`, `health`, `metrics`, `admin`);
- canonical ABI constraints;
- generated host adapters.

### 2. Replace libc/POSIX-first effects with capability-first effects

The current effect model maps symbols like `read`, `write`, `pread`, `pwrite`, `socket`, `accept`, and `fork` to `libc.*`/`sys.*` atoms (`tools/effect-lint.sh:45-78`). That works for native Linux but does not compose cleanly with WASI or serverless.

Enterprise Alien Stack should define effect layers:

| Layer | Example atoms | Purpose |
|---|---|---|
| Core deterministic | `pure`, `global.read`, `global.write`, `alloc.linear` | Host-independent reasoning. |
| WASI capabilities | `wasi:http.incoming`, `wasi:http.outgoing`, `wasi:filesystem.read`, `wasi:clocks.wall`, `wasi:random.get`, `wasi:config.get`, `wasi:keyvalue.*` | Portable component permissions. |
| Host bindings | `aws.lambda.invoke`, `cloudrun.http`, `azure.containerapp.ingress`, `cloudflare.fetch`, `spin.redis`, `wasmcloud.messaging` | Adapter-specific capabilities. |
| Native fallback | `libc.read`, `sys.accept`, `libc.pwrite` | Allowed only for native profile or test harnesses. |

Release policy should ban raw `libc.*`/`sys.*` in the portable profile unless they are behind a verified adapter. Native syscalls become an implementation detail of hosts, not an application effect surface.

### 3. Define WIT packages and generated adapters

Minimum WIT package set:

- `alien:runtime/http` - inbound HTTP request/response and streaming body.
- `alien:runtime/config` - configuration and secrets reads with explicit key allowlists.
- `alien:runtime/logging` and `alien:runtime/telemetry` - structured logs, metrics, traces.
- `alien:runtime/storage` - key/value, blob, and durable journal abstractions.
- `alien:runtime/time` and `alien:runtime/random` - non-deterministic sources.
- `alien:pcf/proof` - manifest interface for proof/effect evidence associated with a component.
- `alien:admin/health` - readiness, liveness, version, schema, migration status.

Then generate runtime adapters:

- **Wasmtime adapter:** local and CI runner for conformance.
- **runwasi adapter:** OCI image with `platform: wasi/wasm`; Kubernetes `RuntimeClass` examples for Wasmtime and WasmEdge.
- **Spin adapter:** Spin manifest + SpinKube `SpinApp`/`SpinAppExecutor` templates.
- **wasmCloud adapter:** component packaged to OCI, with capability provider declarations.
- **Cloudflare adapter:** compile only if imports are compatible with Workers' Wasm constraints and no unsupported WASI syscalls are required.
- **Serverless-container adapter:** Linux container embedding Wasmtime or Spin for AWS Lambda, Cloud Run, and Azure Container Apps.

### 4. Treat cloud serverless containers as compatibility wrappers

Cloud Run, Lambda, and Azure Container Apps are critical enterprise targets, but their documented contracts are container contracts. Therefore the stable path is:

```
Alien verified component
  -> OCI component artifact
  -> tiny host container
      - embeds wasmtime/spin runtime
      - maps HTTP/event source to WIT interface
      - maps cloud config/secrets/logging to capability imports
      - enforces allowed capabilities
```

For Cloud Run, the wrapper listens on `0.0.0.0:$PORT` as required by the Cloud Run container contract. For Lambda, the wrapper implements the Lambda runtime API/custom runtime and maps invocation events into component calls. For Azure Container Apps, the wrapper uses ingress/jobs/KEDA/Dapr as host infrastructure but keeps all application policy inside the component.

This is less pure than native Wasm hosting, but it is operationally enterprise-friendly and lets the same verified component run before every cloud has a first-class component runtime.

## Enterprise maturity gaps and required changes

### Product and architecture

1. **Define product profiles.**
   - `research`: current demos and native experiments.
   - `portable`: Wasm component, WASI-only, no raw libc/sys effects.
   - `enterprise`: portable profile plus signed provenance, SBOM, conformance matrix, support lifecycle.
   - `native-perf`: native Linux build, explicit non-portable profile.

2. **Split source representation from deployment representation.**
   LLVM IR remains canonical authoring/proof input. Wasm component becomes canonical deployment. Do not ask enterprises to deploy raw LLVM IR or bespoke native binaries as the main path.

3. **Replace demo-specific scripts with a real CLI.**
   Build a `alien` CLI:
   - `alien verify`
   - `alien build --target component`
   - `alien package --profile runwasi|spin|wasmcloud|cloudrun|lambda|aca`
   - `alien conformance`
   - `alien seal`
   - `alien attest`

4. **Create a schema registry.**
   Version and validate:
   - `alienstack.pcf.v1`
   - `alienstack.effects.v1`
   - `alienstack.artifact.v1`
   - `alienstack.runtime-matrix.v1`
   - `alienstack.evidence.v1`

### Verification and safety

1. **Move every demo to L2 minimum.**
   Storage is L2; plaintext, webserver, and ui-kit are L1 (`docs/alien-stack-whitepaper.md:425-433`). Enterprise baseline should require L2 for every exported function in the portable profile.

2. **Implement L3 linked-and-sealed release.**
   The whitepaper already specifies link-gate steps (`docs/alien-stack-whitepaper.md:327-337`) and artifact manifests (`docs/alien-stack-whitepaper.md:351-358`). Productize them:
   - schema-validated JSON reports;
   - fail if manifest references missing artifacts;
   - fail if tool hashes are missing/unavailable;
   - fail if any proof is stale relative to normalized obligation hash;
   - sign the final bundle.

3. **Upgrade effect lint from regex shell to parser-backed verifier.**
   The shell lint is a good prototype, but enterprises need a Rust/Go verifier using LLVM parsing and wasm-tools for components. Keep the shell lint as a smoke test, not as the root trust anchor.

4. **Introduce runtime contract assertions.**
   Verified mode trusts proof results, but audit mode should sample boundary assertions at runtime. This catches host adapter drift and spec mistakes.

5. **Add fuzzing and fault injection.**
   L4 needs fault injection against storage, host adapters, malformed inputs, capability denial, clock/random nondeterminism, and partial writes. Every runtime adapter should have a denial test: if a capability is not granted, the component must fail safely.

### Supply chain and compliance

Enterprise-ready release artifacts should include:

- SBOM for host wrapper and toolchain.
- SLSA/in-toto provenance for the build.
- Signed OCI artifacts (for both component and wrapper images).
- Reproducible rebuild job from manifest.
- CVE scanning for wrapper images and included runtimes.
- Policy-as-code checks (OPA/Conftest or equivalent) for Kubernetes manifests.
- License inventory.
- Retention policy for proof/evidence artifacts.

Artifact sealing in `demo/webserver/seal-artifacts.sh` is a good seed, but it must become repo-wide and cryptographically enforced (`demo/webserver/seal-artifacts.sh:106-149`).

### Operations

Alien Stack needs an operational contract as much as a code contract:

- **Health/readiness:** exported health interface plus platform-specific probes.
- **Metrics:** p50/p95/p99 latency, cold start, memory, instruction fuel, trap count, capability-denied count, verifier version.
- **Tracing:** OpenTelemetry spans from host adapter to component call boundary.
- **Logs:** structured JSON logs with request IDs, component digest, proof bundle digest, and adapter version.
- **Config/secrets:** no ambient env reads in portable profile; all config/secrets through explicit capabilities.
- **Resource budgets:** max memory, fuel/epoch interruption, request timeout, max body size.
- **Upgrade and rollback:** component digest pinning, canary/traffic splitting where platform supports it.
- **Runbooks:** traps, proof mismatch, capability denial, failed migration, cold-start regression.

### Performance and scale

The plaintext benchmark is useful but too narrow. Enterprise performance evidence should include:

- cold start across Wasmtime, Spin, wasmCloud, runwasi, Cloud Run wrapper, Lambda wrapper, ACA wrapper;
- steady-state RPS/latency under HTTP;
- max concurrency and backpressure behavior;
- memory floor/ceiling;
- CPU/fuel budget behavior;
- component instantiation cache behavior;
- artifact size and pull time;
- comparison against equivalent Rust/Go container baselines.

Benchmarks should be schema-stored and gated, not just uploaded as logs. The whitepaper already frames scientific hypotheses and machine-readable CI outputs (`docs/alien-stack-whitepaper.md:496-515`); enterprise maturity means using those as release gates.

## Runtime support strategy

### Tier A: first-class support

These should be automated in CI and documented as supported:

1. **Wasmtime direct.** Primary local runner and trust-base runtime.
2. **runwasi/containerd.** Package as OCI `wasi/wasm`; provide `RuntimeClass` manifests.
3. **Spin + SpinKube.** Provide Spin app packaging and Kubernetes CRDs/templates.
4. **wasmCloud.** Publish component to OCI and provide wasmCloud deployment descriptors.

### Tier B: serverless-container compatibility

These should be "supported via wrapper":

1. **Google Cloud Run.** Wasmtime/Spin wrapper container, HTTP on `$PORT`, no persistent filesystem assumptions.
2. **AWS Lambda.** Custom runtime or container image wrapper implementing Lambda runtime API.
3. **Azure Container Apps.** Container wrapper with ingress, jobs, KEDA scaling, and optional Dapr bridge.

### Tier C: constrained edge

1. **Cloudflare Workers.** Use only for compatible modules; Workers supports Wasm, but WASI is experimental and only some syscalls are implemented. Avoid claiming full WASI portability here.
2. **Browser.** UI kit path remains valid, with the caveat that browser ABI must be versioned and imported syscalls allowlisted.

### Tier D: non-anchor

1. **Docker Desktop Wasm.** Useful for developer education, but not a production target because Docker documents Wasm workloads as deprecated.

## Proposed roadmap

### Phase 0: clarify claims (1-2 weeks)

- Rename current state to "research profile".
- Add a `PORTABILITY.md` explaining native Linux, browser Wasm, and future component targets.
- Add `docs/enterprise-readiness.md` with explicit non-goals and target runtime matrix.
- Define support matrix terms: experimental, preview, supported, certified.

### Phase 1: component baseline (2-4 weeks)

- Add `wit/` package definitions for HTTP, config, storage, telemetry, and admin.
- Add a minimal stateless HTTP component compiled from LLVM IR.
- Build with Wasmtime component tooling.
- Run the same component under Wasmtime direct and Spin.
- Add import allowlist verification using wasm-tools.

Exit criterion: one HTTP component has L1/L2-style metadata, effect lint, and runs under two Wasm runtimes.

### Phase 2: OCI and Kubernetes (4-6 weeks)

- Package component as OCI artifact.
- Add runwasi `RuntimeClass` examples.
- Add SpinKube `SpinApp`/`SpinAppExecutor` examples.
- Add wasmCloud deployment descriptors.
- Add CI jobs for kind/k3d if feasible; otherwise nightly integration tests with recorded outputs.

Exit criterion: same digest deploys to Wasmtime, runwasi, SpinKube, and wasmCloud without code changes.

### Phase 3: serverless container wrappers (4-8 weeks)

- Build a tiny host wrapper around Wasmtime or Spin.
- Implement adapters for Cloud Run, Lambda, and Azure Container Apps.
- Add platform contract tests:
  - Cloud Run: listens on `$PORT`, startup deadline, no persistent filesystem assumption.
  - Lambda: runtime API events and `/tmp` storage only.
  - ACA: ingress/jobs/KEDA-compatible behavior.

Exit criterion: same component digest, different wrapper config, deployed to three serverless-container platforms.

### Phase 4: enterprise evidence bundle (6-10 weeks)

- Add signed artifact seal.
- Add SBOM and provenance.
- Add schema-validated proof/effect/runtime reports.
- Add reproducible rebuild job.
- Add security scans and policy checks.
- Add runtime conformance report generation.

Exit criterion: a release can be audited from source commit to deployed component digest with machine-readable evidence.

### Phase 5: stateful/durable profile (ongoing)

- Lift the storage IPS model into WASI-compatible capabilities.
- Add durable journal abstraction.
- Test fault injection across adapters.
- Define migration/versioning contracts.
- Prove invariants over storage transitions, not just single-file native POSIX behavior.

Exit criterion: L4-style durability works in at least one portable host and one serverless-container wrapper.

## What should change in the repo first

1. Add `wit/alien-runtime/*.wit`.
2. Add `tools/alien-verify` as a real verifier prototype in Rust or Go.
3. Add `demo/component-http/` as the first server-side Wasm component.
4. Add `packages/runwasi/`, `packages/spin/`, `packages/wasmcloud/`, `packages/cloudrun/`, `packages/lambda/`, `packages/azure-container-apps/`.
5. Replace ad hoc JSON with checked schemas under `schemas/`.
6. Add `conformance/` test harness that records runtime, adapter, component digest, imports, startup time, request result, memory, and logs.
7. Move artifact sealing from demo-specific script to repo-level `tools/seal`.
8. Add a `SECURITY.md`, `SUPPORT.md`, `ADRs/`, and threat model.

## Main risks

- **LLVM IR as source may remain too hard to maintain.** Mitigation: keep IR as canonical proof/deployment IR but allow higher-level agent-facing generators as long as generated IR is checked and committed/sealed.
- **WASI/component standards are still moving.** Mitigation: version WIT packages and adapters; certify against specific runtime versions.
- **Cloud platforms are not uniformly Wasm-native.** Mitigation: use host-wrapper containers where first-class Wasm is absent.
- **Proof obligations may not scale.** Mitigation: start with capability/effect and boundary proofs; use bounded verification; reserve full SMT for high-value invariants.
- **Enterprise buyers distrust bespoke verification.** Mitigation: keep TCB small, publish schemas, use standard tools (Z3/CVC5/wasm-tools/Wasmtime), and produce reproducible evidence.

## Bottom line

Alien Stack becomes enterprise-ready by becoming **component-first, capability-first, evidence-first**.

The repo's research thesis should stay intact: agents author low-level explicit representations with contracts, effects, and proofs. But the enterprise deployment unit should be a verified Wasm component, not a native Linux binary. Runtime support should be proven through a matrix: Wasmtime, runwasi/containerd, SpinKube, wasmCloud, Cloudflare where compatible, and host-wrapper containers for Cloud Run, AWS Lambda, and Azure Container Apps. The maturity jump is not another demo; it is productizing the verifier, packaging, runtime adapters, operational telemetry, and signed evidence bundle.

If the team does that, Alien Stack could become a credible enterprise platform for high-assurance, portable agent-authored services. If it does not, it will remain an interesting LLVM/Wasm research repo with good demos but no deployable enterprise story.
