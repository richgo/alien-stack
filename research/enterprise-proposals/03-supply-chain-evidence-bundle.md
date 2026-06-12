# Proposal 03: Supply Chain, Artifact Sealing, and Evidence Bundles

Status: Draft enterprise proposal  
Date: 2026-06-12  
Finding addressed: Artifact sealing is partial and Alien Stack lacks signed evidence, SBOM, provenance, policy, and reproducible release bundles.

## Problem

Alien Stack already specifies deterministic artifacts and TCB capture as design principles (`README.md:69-76`; `docs/alien-stack-whitepaper.md:351-358`). The webserver demo has a promising artifact seal script that records source/artifact digests and tool records (`demo/webserver/seal-artifacts.sh:1-7`, `demo/webserver/seal-artifacts.sh:106-149`). However, this is demo-local and unsigned. There is no repo-wide release evidence bundle, no SBOM, no SLSA/in-toto provenance, no policy gate, no reproducible rebuild job, and no cryptographic link from deployed artifact back to proof/effect/runtime evidence.

Enterprise release must be auditable from source commit to deployed component/container digest.

## Target evidence bundle

Every release should emit:

```text
dist/
  alienstack.evidence-bundle.json
  alienstack.evidence-bundle.json.sig
  alienstack.sbom.cdx.json
  alienstack.provenance.intoto.jsonl
  artifact-manifest.json
  pcf-proof-report.json
  effect-lint-report.json
  runtime-conformance-matrix.json
  policy-report.json
  reproducibility-report.json
```

## Evidence bundle schema

```json
{
  "schema": "alienstack.evidence-bundle.v1",
  "release": {
    "version": "vX.Y.Z",
    "git_commit": "...",
    "source_repo": "superfield-ai/alien-stack",
    "profile": "portable"
  },
  "subjects": [
    {"name": "component", "digest": "sha256:..."},
    {"name": "wrapper-image", "digest": "sha256:..."}
  ],
  "verification": {
    "pcf": {"status": "pass", "digest": "sha256:..."},
    "effects": {"status": "pass", "digest": "sha256:..."},
    "link_gate": {"status": "pass", "digest": "sha256:..."},
    "runtime_matrix": {"status": "pass", "digest": "sha256:..."}
  },
  "supply_chain": {
    "sbom": {"format": "cyclonedx-json", "digest": "sha256:..."},
    "provenance": {"format": "slsa-v1/intoto", "digest": "sha256:..."},
    "signature": {"provider": "sigstore", "rekor_index": "..."}
  },
  "tcb": [
    {"tool": "clang", "version": "...", "sha256": "..."},
    {"tool": "wasmtime", "version": "...", "sha256": "..."},
    {"tool": "z3", "version": "...", "sha256": "..."}
  ],
  "policy": {
    "status": "pass",
    "ruleset": "alienstack.release-policy.v1",
    "digest": "sha256:..."
  }
}
```

## Signing and attestation flow

1. Build verified component and optional wrapper images.
2. Generate all reports.
3. Generate SBOM for:
   - component artifact;
   - wrapper image;
   - verifier/toolchain container;
   - vendored scripts/tools.
4. Generate SLSA provenance with builder identity, materials, commands, artifact subjects, and environment metadata.
5. Generate evidence bundle and policy report.
6. Sign subjects and attestations using Sigstore keyless signing.
7. Publish OCI artifacts and attach attestations.
8. Release workflow verifies signatures before publishing release notes.

## SBOM strategy

Use CycloneDX JSON as the primary SBOM format because it can represent components, services, hashes, licenses, and external references. Enrich SBOM entries with Alien-specific properties:

```json
{
  "type": "file",
  "name": "demo/storage/ips.ll",
  "hashes": [{"alg": "SHA-256", "content": "..."}],
  "properties": [
    {"name": "alienstack.pcf.coverage", "value": "complete"},
    {"name": "alienstack.effects.status", "value": "pass"},
    {"name": "alienstack.conformance", "value": "L2"}
  ]
}
```

## Provenance strategy

Target SLSA Level 3-style controls:

- version-controlled build workflow;
- isolated builder;
- pinned toolchain image;
- non-forgeable provenance;
- signed subjects;
- complete materials list;
- reproducible rebuild attempt.

## Policy gate

Release policy should fail if:

- any required report is missing;
- any report status is not `pass`;
- Z3 proof is skipped in release profile;
- tool hashes are missing/unavailable;
- component digest is not bound to WIT world digest;
- SBOM or provenance is missing;
- signature is missing;
- runtime matrix for required tier is missing or failing;
- any unknown effect exists.

## Reproducible rebuild

Introduce a hermetic build image and a rebuild workflow:

1. Checkout release commit.
2. Verify toolchain hashes.
3. Rebuild component/wrapper.
4. Recompute proofs/effect reports.
5. Compare artifact digests with release evidence.
6. Emit `reproducibility-report.json`.

The current webserver seal records tool hashes opportunistically (`demo/webserver/seal-artifacts.sh:109-145`); enterprise release must fail when required hashes are absent.

## Release process

1. Tag release candidate.
2. Run verification gates.
3. Run runtime conformance matrix.
4. Generate SBOM/provenance/evidence.
5. Run policy gate.
6. Sign and publish artifacts.
7. Run reproducible rebuild.
8. Publish release with evidence links.

## Trust model

Trusted:

- pinned verifier implementation;
- pinned SMT solver/proof checker;
- pinned Wasm/component tooling;
- CI identity and signing identity;
- runtime versions listed in conformance matrix.

Untrusted:

- agent-authored IR until verified;
- host adapters until tested and signed;
- runtime capabilities not declared in WIT/effects;
- manual release notes not backed by evidence.

## Implementation roadmap

1. **Foundation:** Move artifact sealing to `tools/seal`; define JSON schemas.
2. **SBOM/provenance:** Add CycloneDX and SLSA/in-toto generation.
3. **Signing:** Add Sigstore signatures for evidence and artifacts.
4. **Policy:** Add OPA/Rego or equivalent release policy.
5. **Reproducibility:** Add hermetic rebuild job.
6. **L3 declaration:** Claim L3 only when link gate, seal, evidence, and signatures are enforced.

## Acceptance criteria

- Release evidence bundle is schema-validated.
- All subjects are signed and attestations are verifiable.
- SBOM lists source, verifier, toolchain, component, and wrapper artifacts.
- Provenance identifies builder, materials, commands, and output digests.
- Policy gate is blocking for release.
- Reproducible rebuild report is generated.
- Evidence bundle includes runtime matrix digest and WIT/component digest.
