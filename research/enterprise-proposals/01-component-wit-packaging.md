# Proposal 01: WIT and WebAssembly Component Packaging

Status: Draft enterprise proposal  
Date: 2026-06-12  
Finding addressed: Alien Stack has no WIT/component packaging path, so its strongest portability story is still aspirational.

## Problem

Alien Stack currently treats LLVM IR as the canonical executable source and demonstrates native Linux binaries plus browser Wasm. That is a strong research baseline, but not an enterprise portability story. The repo itself documents that native demos target x86_64 Linux and depend on POSIX calls such as `pread`, `pwrite`, and `fork` (`README.md:117`; `demo/README.md:8`). The UI kit compiles LLVM IR to `wasm32-unknown-unknown` (`demo/ui-kit/build.sh:35-41`) and the client whitepaper defines an `alien-stack.client.abi.v1` syscall surface in prose (`docs/isomorphic-web-whitepaper.md:122-156`), but there are no `.wit` packages, no WebAssembly component build, and no component-level import allowlist.

This blocks enterprise readiness because the deployment artifact cannot yet express portable host capabilities in a standard way. WASI is the standards-track system interface for Wasm apps, and the WebAssembly Component Model provides portable cross-language composition via WIT-defined interfaces. Official sources: <https://wasi.dev/>, <https://github.com/WebAssembly/WASI>, <https://github.com/WebAssembly/component-model>, <https://docs.wasmtime.dev/introduction.html>.

## Target architecture

The enterprise deployment unit should be:

```text
LLVM IR source
  -> normalized IR
  -> PCF/effect/proof verification
  -> core wasm32 module
  -> WebAssembly component lifted against WIT world
  -> signed OCI artifact
  -> runtime adapter
```

LLVM IR remains the source-of-truth for agent authoring and proof extraction. The **Wasm component** becomes the source-of-truth for deployment, capability binding, and runtime certification.

Key invariant:

> A component is releasable only when its WIT imports are a subset of its declared PCF effects, its exported functions have complete PCF metadata, and its evidence bundle binds the component digest to the verified IR/proof/toolchain digests.

## Proposed repository layout

```text
wit/
  alien-stack.wit
  interfaces/
    http.wit
    config.wit
    secrets.wit
    logging.wit
    metrics.wit
    tracing.wit
    storage.wit
    health.wit
    admin.wit
    host-dom.wit
  worlds/
    server.wit
    storage.wit
    client.wit
    bare.wit

tools/
  alien-component-build
  alien-wit-check
  effect-atom-map.yaml

demo/
  component-http/
    ir/
    wit/
    build.sh
    verify.sh
```

## Proposed WIT worlds

### `alien:runtime/server`

For stateless HTTP handlers and serverless entrypoints.

```wit
package alien:runtime;

interface logging {
  enum level { debug, info, warn, error }
  log: func(level: level, message: string);
}

interface metrics {
  increment: func(name: string, value: u64);
  observe: func(name: string, value: f64);
}

interface config {
  get: func(key: string) -> option<string>;
}

interface health {
  readiness: func() -> bool;
  liveness: func() -> bool;
}

world server {
  import wasi:http/incoming-handler@0.2.0;
  import logging;
  import metrics;
  import config;
  export health;
}
```

### `alien:runtime/storage`

For IPS-backed state. The current storage demo proves invariants over a native file (`demo/storage/spec.md:9-13`; `demo/storage/build.sh:49-72`). The enterprise version should abstract durable I/O through explicit capabilities:

```wit
interface journal {
  append: func(domain: string, frame: list<u8>) -> result<u64, string>;
  read: func(domain: string, epoch: u64) -> result<list<u8>, string>;
  fsync: func(domain: string) -> result<_, string>;
}

world storage {
  import journal;
  import logging;
  import metrics;
  export health;
}
```

### `alien:runtime/client`

This should formalize the prose browser ABI already specified in the client whitepaper (`docs/isomorphic-web-whitepaper.md:122-144`) and implemented in the UI kit shim (`demo/ui-kit/spec.md:11-14`).

```wit
interface host-dom {
  dom-create: func(tag: string) -> u32;
  dom-append: func(parent: u32, child: u32);
  dom-set-text: func(node: u32, value: string);
  dom-set-attr: func(node: u32, key: string, value: string);
  dom-set-style: func(node: u32, key: string, value: string);
  dom-listen: func(node: u32, event-id: u32);
  dom-remove: func(node: u32);
}

world client {
  import host-dom;
  export init: func();
  export on-event: func(node: u32, event-id: u32);
}
```

## Verification changes

1. **WIT import allowlist gate.** Parse the component and fail if imported interfaces are not in the selected world.
2. **Effect-to-WIT cross-check.** Each WIT import maps to one or more canonical effect atoms. Example:
   - `wasi:http/incoming-handler` -> `wasi.http.incoming`
   - `alien:runtime/logging.log` -> `ops.log.write`
   - `alien:runtime/storage.journal.append` -> `storage.journal.append`
3. **PCF export coverage.** Every component export must bind to a PCF with `pcf.schema`, `pcf.pre`, `pcf.post`, `pcf.effects`, `pcf.proof`, `pcf.bind`, and `pcf.toolchain` metadata. This follows the whitepaper metadata requirement (`docs/alien-stack-whitepaper.md:238-250`).
4. **Component digest binding.** The artifact/evidence bundle must bind component digest, core Wasm digest, IR digest, WIT world version, verifier reports, and toolchain hashes.

The current shell `effect-lint.sh` should remain as a smoke gate, but enterprise trust should move to a parser-backed verifier using LLVM and `wasm-tools`. The shell lint is explicitly regex/awk-based (`tools/effect-lint.sh:1-28`) and is not enough as a root enterprise trust anchor.

## Implementation phases

1. **WIT foundation.** Add `wit/` packages and an `alien-wit-check` script that validates syntax and world names.
2. **UI kit component.** Convert `demo/ui-kit` to lift `public/app.wasm` into a component using `client.wit`; verify import allowlist and PCF export coverage.
3. **Minimal server component.** Add `demo/component-http` with an LLVM IR handler compiled to a WASI HTTP component.
4. **Storage component experiment.** Rework IPS storage behind `journal.wit`, keeping native storage as a compatibility profile.
5. **Component link gate.** Add inter-component compatibility checks using WIT worlds plus PCF pre/post/effect compatibility.
6. **OCI packaging.** Push signed components to OCI as release artifacts.

## Acceptance criteria

- `wit/` contains versioned package definitions and docs.
- At least one server component builds from LLVM IR and runs in Wasmtime.
- UI kit imports match `alien:runtime/client` exactly.
- Component import set is proven to be a subset of declared PCF effects.
- Evidence bundle includes component digest, WIT package digest, IR digest, proof report, effect report, and toolchain hashes.
- CI blocks portable-profile releases when unknown imports or undeclared effects appear.

## Risks and mitigations

- **Canonical ABI mismatch with existing pointer/length contracts.** Keep PCF bindings at both core-module and component-lifted levels until the mapping is proven.
- **Toolchain immaturity.** Pin Wasmtime/wasm-tools versions and certify exact versions.
- **Metadata stripping by LLVM optimization.** Preserve PCF metadata before optimization or store a normalized sidecar bound by digest.
- **Native demos cannot immediately become components.** Define native as a separate `native-perf` profile and build new component demos rather than forcing all existing code through WASI at once.

## Open questions

- Should WIT definitions live as first-class source or generated from PCF metadata?
- Should standard WASI interfaces be wrapped by `alien:*` interfaces, or imported directly?
- How much of the PCF link gate can reuse `wasm-tools compose`, and where does Alien-specific contract checking need a custom verifier?
