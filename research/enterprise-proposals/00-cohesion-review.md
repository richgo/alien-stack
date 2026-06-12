# Cohesion Review: Enterprise Proposal Set

Date: 2026-06-12  
Purpose: Reconcile the four parallel proposal drafts into a consistent enterprise maturity plan.

## Documents reviewed

1. `01-component-wit-packaging.md`
2. `02-runtime-conformance-matrix.md`
3. `03-supply-chain-evidence-bundle.md`
4. `04-operations-contract.md`

## Shared decisions

### 1. LLVM IR remains source; Wasm component becomes deployment unit

All documents use the same split:

- **Source/proof representation:** LLVM IR plus PCF metadata.
- **Portable deployment representation:** WebAssembly component with WIT-defined capabilities.
- **Compatibility representation:** host-wrapper container for platforms that are serverless-container systems rather than first-class Wasm component systems.

This resolves the native-vs-Wasm conflict by defining profiles:

- `research`: current demos.
- `portable`: Wasm component and WIT imports only.
- `enterprise`: portable profile plus signed evidence, runtime matrix, and operations contract.
- `native-perf`: native Linux binary for explicit performance experiments.

### 2. WIT is the capability source of truth

All documents assume WIT packages define the stable host ABI. PCF effects must be checked against WIT imports, and operations capabilities are just another part of the declared effect surface.

### 3. Evidence bundle ties everything together

The supply-chain document owns signing, SBOM, provenance, and release policy. The other proposals feed it:

- component/WIT proposal provides component digest and WIT digest;
- runtime matrix proposal provides conformance report digest;
- operations proposal provides budgets/runbooks/observability evidence.

### 4. Runtime support is tiered, not absolute

Runtime matrix and operations documents agree on:

- Tier 1: Wasmtime direct as the initial blocking target.
- Tier 2: runwasi/containerd, SpinKube, wasmCloud, Cloud Run wrapper, Lambda wrapper, Azure Container Apps wrapper.
- Tier 3: Cloudflare Workers and Docker Desktop Wasm as caveated/preview targets.

Cloudflare is intentionally not treated as full WASI-certified because its docs describe WASI support as experimental with only some syscalls implemented.

### 5. Cloud Run/Lambda/Azure are wrapper targets

The proposals consistently treat these as serverless-container targets. Alien Stack should run there through a small host wrapper that embeds Wasmtime/Spin and maps platform APIs to WIT capabilities.

## Conflicts found and resolved

### Conflict: direct standard WASI imports vs `alien:*` wrapper interfaces

Resolution: allow both, but require a mapping file. Standard WASI interfaces may be imported directly where stable. `alien:*` interfaces wrap policy-sensitive areas such as storage, secrets, admin, and telemetry.

### Conflict: component digest vs evidence bundle digest as rollout unit

Resolution: rollout identity is a tuple:

- component digest;
- WIT world digest;
- wrapper image digest, if applicable;
- evidence bundle digest;
- runtime adapter version.

### Conflict: all runtimes as release blockers vs practical CI cost

Resolution:

- Tier 1 blocks PRs and releases.
- Tier 2 blocks release candidates, not every PR.
- Tier 3 is report-only.

### Conflict: storage portability

Resolution: do not force the native POSIX storage demo directly into portable profile. Instead, define a `journal.wit` abstraction and build a new portable storage component while preserving native storage as research/native-perf evidence.

## Recommended implementation order

1. Add WIT packages and one minimal component HTTP demo.
2. Add Wasmtime direct conformance as the first blocking runtime gate.
3. Add evidence bundle schema and repo-level seal tool.
4. Add operations WIT interfaces and structured logging/health in the Wasmtime adapter.
5. Add runwasi/Spin/wasmCloud packaging.
6. Add serverless-container wrappers.
7. Add signing/SBOM/provenance and release policy.
8. Add stateful storage portability and fault injection.

## Final assessment

The proposal set is cohesive. The central architecture is:

```text
LLVM IR + PCF metadata
  -> verifier/proof/effect gates
  -> WebAssembly component + WIT world
  -> signed evidence bundle
  -> certified runtime adapter
  -> observable, budgeted, rollback-safe production service
```

No remaining conflicts require revisiting the drafts. The largest unresolved design decision is tactical: whether to import stable WASI interfaces directly or always wrap them under `alien:*`. The reconciled recommendation is direct WASI for stable generic capabilities and `alien:*` for policy-bearing capabilities.
