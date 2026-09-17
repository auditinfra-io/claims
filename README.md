# Audit Engine CLI — Public Architecture

Status: Public technical overview  
Audience: Security engineers, protocol teams, auditors, and platform integrators  
Implementation language: Python 3.10+ (with a small amount of Rust/toolchain integration)  
Product class: Local-first static analysis and formal-verification tooling for smart contracts and zero-knowledge circuits

This document describes the externally observable architecture and engineering principles of Audit Engine CLI. It is intentionally implementation-neutral where disclosure would expose proprietary detection heuristics, corpus composition, or operational procedures. The goal is to make the system’s contracts, assumptions, and evidence model reviewable without publishing intellectual property.

## 1. What the system does

Audit Engine CLI analyzes source trees and, where available, compiler artifacts to identify security-relevant defects in:

- Solidity and EVM-oriented projects;
- Rust, Anchor, and Solana programs;
- Circom and selected ZK/zkVM ecosystems;
- Move and other supported adapters; and
- mixed-language repositories.

It combines source-level rules, structural analysis, information-flow analysis, bounded symbolic reasoning, and optional external formal-verification tools. The output is a machine-readable audit bundle and operator-friendly reports in JSON, SARIF, HTML, and text forms.

The engine is an analysis aid, not a proof that a project is secure. A clean result can mean either that no supported defect was found or that a requested analysis tier was unavailable, skipped, timed out, or incomplete. Reports therefore distinguish findings, confidence, and analysis completeness.

## 2. Design goals

### 2.1 Evidence over inference

Every finding should be explainable through source locations, matched conditions, data-flow steps, solver status, or a concrete witness. A rule is not considered stronger merely because it has a high severity label. Severity describes potential impact; confidence describes evidence quality.

### 2.2 Graceful degradation without silent success

The scanner has useful compiler-free modes for source-level analysis. Compiler, solver, or external-tool failures are isolated and recorded as capability gaps or incomplete analysis. They do not get converted into “safe.” Operators can select strict policies that fail a run when a requested tier cannot execute.

### 2.3 Independent safety nets

High-value source detectors run before expensive proving work. This keeps their results available when a solver process reaches a timeout or memory limit. Findings are merged using stable identity fields and retain their origin and evidence.

### 2.4 Reproducibility

A scan records its input identity, configuration, tool capabilities, analysis modes, and output schema. Optional replay and evidence-manifest workflows allow a later operator to verify what was run and whether an artifact was altered.

### 2.5 Bounded resource use

Deep analysis is deliberately bounded by file-size, function-count, solver-time, process-memory, depth, and worker limits. The engine prefers an explicit incomplete result to an unbounded analysis that can exhaust a CI runner or workstation.

## 3. High-level pipeline

```text
Command-line configuration
        │
        ▼
Profile and capability resolution
        │
        ▼
File discovery, filtering, and language routing
        │
        ▼
Compiler-free source fast pass
        │
        ▼
Language-specific parsing and structural analysis
        │
        ├── pattern and semantic detectors
        ├── taint / information-flow analysis
        ├── interprocedural and cross-file analysis
        └── optional IR and compiler-artifact analysis
        │
        ▼
Invariant extraction and solver-backed checking
        │
        ▼
Bounded multi-step model checking (optional)
        │
        ▼
Cross-tier correlation and protocol-aware enrichment
        │
        ▼
Finding gate, suppression ledger, confidence, and completeness
        │
        ▼
Canonical result assembly and reporting
```

Adversarial objective search and exploit-chain synthesis are explicit opt-in operations. They reuse bounded transition models but are not silently enabled by an ordinary scan.

## 4. Lexical and structural parsing

The parser stack is intentionally layered. A source file may be useful even when a compiler is unavailable, while compiler-backed tiers can add type, inheritance, and artifact context when it exists.

### 4.1 Source normalization

Before source rules inspect text, the engine normalizes line endings and maintains source offsets. Comments and string literals are handled by shared preprocessing rather than ad-hoc substring removal. This prevents comment text, quoted examples, and documentation from being mistaken for executable behavior.

The normalizer preserves enough positional information to report findings against the original file. When imports are expanded for analysis, findings retain both the analysis-space position and the original-file provenance.

### 4.2 Balanced lexical regions

Source-level extraction uses delimiter-aware scanning for braces, parentheses, and brackets. Function, modifier, handler, and contract-like regions are accepted only when their boundaries are balanced. Nested blocks, multiline declarations, attributes, and ordinary formatting changes must not change the region's meaning.

The lexer treats the following as distinct lexical concerns:

- declarations versus executable statements;
- state storage versus local variables and parameters;
- modifiers or attributes versus function bodies;
- external calls versus internal dispatch;
- comments and strings versus code; and
- test-only or generated regions versus deployable code where the adapter can establish that distinction.

These are semantic categories, not a promise that every language construct can be parsed in compiler-equivalent detail in direct mode.

### 4.3 Compiler-aware enrichment

When a compatible compiler or build artifact is available, the engine may add AST, type, inheritance, source-map, and build-context information. Compiler-backed analysis is additive: losing AST access must not erase findings already produced by the source fast pass.

The system records whether a file was analyzed in direct, AST-assisted, artifact, or degraded mode. Consumers should use this capability metadata when interpreting “no finding.”

### 4.4 Import and provenance handling

Import expansion is cycle-aware and deduplicates shared dependencies. Findings are mapped back to origin files and lines. Dependencies outside the requested project scope are treated differently from project-owned files so that a library finding is not duplicated into every importer.

## 5. State and execution modeling

### 5.1 Explicit pipeline state

Scan state is carried through explicit context objects and typed result structures rather than hidden process-global variables. The context includes selected files, language capabilities, project roots, profile choices, resource budgets, and tier outcomes.

Each finding is an evidence-bearing record. In addition to a rule and location, it can contain matched premises, trace steps, source provenance, reachability information, solver status, counterexample data, and suppression history.

### 5.2 Symbolic state

For supported languages, the modeler represents relevant state variables with pre-state and post-state symbols. Handler transitions describe reads, writes, calls, guards, and selected arithmetic relationships. Unsupported or unresolved behavior is represented as unknown or conservatively bounded input rather than silently assumed safe.

Protocol invariant packs use canonical concepts such as total assets, supply, exchange rate, collateral, or message source. Adapter-specific aliases bind those concepts to identifiers discovered in the target project. A binding is evidence-backed; an unbound concept is reported as unavailable rather than treated as proven.

### 5.3 Multi-step state

Bounded model checking composes a finite sequence of callable transitions. State variables persist across steps; ephemeral call inputs, authorization predicates, and other per-call facts are refreshed at the correct boundary. The checker can then ask whether a safety property is violated within a configured depth.

Adversarial mode changes the question from “can a safety property be violated?” to “what is the greatest bounded change to a selected asset under this attacker profile?” It emits an ordered witness only when the model has meaningful constraints and the result is classified as a violating witness.

### 5.4 Unknown and incomplete results

The result vocabulary distinguishes at least:

- proven safe within the modeled scope;
- violated with a counterexample or witness;
- unknown because the solver could not decide;
- skipped because prerequisites were absent;
- incomplete because a budget or quality floor was reached; and
- error because an analysis component failed.

“unknown”, “skipped”, and “incomplete” are never silently collapsed into “safe”.

## 6. Detection coverage

The public rule families include, among others:

- missing or inconsistent access control and privileged parameters;
- reentrancy and unsafe balance-delta accounting;
- arithmetic, division, rounding, and zero-substitution hazards;
- replay protection and incorrect replay-key binding;
- unsafe signed routing fields and payload/source binding gaps;
- cross-chain message authentication and confirmation binding;
- oracle staleness and fallback weaknesses;
- pause asymmetry and accounting/NAV inflation;
- bridge metadata and assumed token-transfer semantics;
- vault donation and share-accounting exposure;
- locked value and liveness/gas griefing conditions;
- Rust/Anchor signer, owner, CPI, stale-read, and account-writer risks;
- ZK witness consistency, range/modulus, non-canonical representation, and constraint-soundness risks; and
- protocol invariant violations discovered by bounded symbolic analysis.

These are bug classes the engine is designed to investigate, not an exhaustive list of all possible exploits. Business logic, economic assumptions, deployment configuration, off-chain components, and bugs outside modeled language semantics remain in scope for human review.

## 7. Finding quality model

The reporting contract separates three questions:

1. Impact: If the suspected condition is exploitable, how serious could it be?
2. Evidence: How directly does the analysis establish the condition?
3. Completeness: Did the requested analysis actually execute with sufficient coverage?

Evidence can progress from a lexical match to parsed context, cross-function data flow, a solver counterexample, a bounded witness, or an executed replay. A generated proof-of-concept or an attempted solver run is not itself proof. Suppression and demotion decisions are recorded with a reason and remain auditable.

## 8. Scaling and performance

There is no honest single “contracts per second” number: runtime depends on source size, import closure, compiler availability, selected profile, solver depth, protocol packs, hardware, and the number of findings requiring enrichment. The project should publish measured benchmark artifacts for any release-specific throughput claim rather than use a synthetic guarantee here.

The scaling strategy is:

- cheap source passes before expensive proving;
- bounded parallelism across independent work;
- process isolation for solver and BMC work;
- per-file and per-query timeouts;
- memory limits for solver workers;
- caching of build and reusable analysis context where safe;
- complexity gates for exceptionally large files;
- optional contract-level sharding for large Solidity inputs; and
- pre-merging and deduplication of invariant packs to avoid repeated handler discovery.

For a production evaluation, measure cold and warm runs separately and record CPU count, memory, compiler/tool versions, profile, source-tree hash, file counts, lines of code, solver depth, timeout policy, and degraded-tier reasons. Report median and tail latency, peak memory, findings by tier, and completeness—not only wall-clock time.

## 9. Isolation, failure handling, and security

Untrusted source is treated as input. Expensive or crash-prone components run in bounded child processes where practical. A child failure is converted into structured telemetry and does not erase parent-process findings. Output bundles can include hashes, manifests, proof metadata, and detached signatures for organizations that need evidence-chain verification.

The engine does not execute target code as part of ordinary lexical analysis. Optional compilation, replay, fuzzing, or external formal tools should be run in an appropriately isolated environment with dependency and network policies chosen by the operator.

## 10. Integration surface

The stable integration points are:

- CLI profiles for Solidity/EVM, Rust/Solana, ZK, Move, and mixed projects;
- JSON output for automated triage and regression comparison;
- SARIF for code-scanning systems;
- HTML and text for human review;
- replayable scan bundles for repeatability; and
- optional Foundry, fuzzing, Kontrol/KEVM, and solver integrations.

Exact flags, defaults, exit codes, and optional dependencies belong to the user-facing usage reference and may change independently of this architecture overview.

## 11. What this document deliberately does not disclose

This document does not publish proprietary detector heuristics, private benchmark targets, customer material, internal rule-tuning history, model prompts, unpublished invariant packs, or operational secrets. It describes interfaces and guarantees at the level needed to evaluate suitability and integration. Public behavior can be validated through the repository’s documented fixtures, tests, schemas, and reproducible benchmark commands.

## 12. Limitations and responsible use

No static analyzer can establish the absence of every exploit. In particular, results may be limited by unsupported language features, unresolved dynamic dispatch, incomplete build metadata, unknown external contracts, economic assumptions, solver incompleteness, and bounded depth. Reviewers should inspect the completeness and capability sections of every report, validate important findings against the deployed configuration, and combine automated analysis with manual threat modeling and testing.

## 13. Summary

Audit Engine CLI is a Python-based, local-first analyzer that layers lexical detection, structural and data-flow analysis, bounded symbolic reasoning, and optional formal verification. Its architecture is designed to preserve useful findings under degraded conditions, make uncertainty visible, maintain source traceability, and scale expensive reasoning through explicit resource budgets. The public contract is therefore not “a clean scan means secure”; it is “the engine reports what it established, what it could not establish, and the evidence supporting each conclusion.”
