# Agent Supply Chain Sentinel

A planned static security scanner for AI-agent skills, MCP configurations, plugins, tool definitions, and supporting scripts.

**Agent Supply Chain Sentinel** is intended to help developers inspect agent-related components *before* they are trusted, installed, or executed. It will identify risky patterns, explain the evidence behind each finding, and produce reports that can be reviewed by a human or consumed by CI.

> **Project status:** Documentation-first starting point. The repository currently contains this README only; the scanner, detection rules, CLI, and CI integration have not yet been implemented. Planned behavior below is a design target, not an existing security guarantee.

## The problem

AI-agent systems increasingly depend on files and packages that define what an agent can do: skills, tool descriptions, MCP server configurations, plugin manifests, shell scripts, dependency declarations, and setup instructions.

These components can be risky for reasons that are easy to miss in a quick review:

- a setup script downloads and executes remote content;
- configuration grants broader permissions than the task requires;
- a tool endpoint points to an unexpected host;
- a script accesses credential stores or sensitive directories;
- a dependency is pinned weakly or its source cannot be established;
- instructions ask an agent to disclose environment variables or bypass safeguards.

A static scanner cannot determine intent perfectly, but it can surface concrete indicators and make reviews more consistent.

## What the project aims to do

- Scan supported files without executing them.
- Identify suspicious commands, risky permission patterns, credential access, unexpected network destinations, and unsafe configuration combinations.
- Attach evidence to every finding: rule ID, file path, line or location, severity, explanation, and suggested next step.
- Generate human-readable terminal output and machine-readable JSON.
- Run in local development and CI before a component is accepted.
- Keep rules understandable, testable, and reviewable.
- Support explicit exceptions without silently suppressing unrelated findings.

## What it will not claim

- It will not prove a skill or plugin is malicious just because a pattern matches.
- It will not prove a component is safe because no finding was produced.
- It will not replace sandboxing, runtime policy enforcement, dependency verification, code review, or permission minimization.
- It will not execute scanned scripts, install scanned packages, or contact discovered endpoints as part of a basic scan.
- It will not rely on an LLM as the sole authority for a security verdict.

The scanner should say **“suspicious pattern detected”** or **“requires review,”** not “malware confirmed,” unless a future rule has evidence sufficient to support that stronger statement.

## Intended scope

The initial formats to evaluate are:

| Input type | Example checks |
| --- | --- |
| Agent skill folders | Dangerous shell patterns, credential requests, suspicious setup instructions |
| MCP server configuration | Command and argument risks, broad environment forwarding, unexpected server endpoints |
| Plugin or tool manifests | Overly broad capabilities, unsafe defaults, unclear executable provenance |
| Shell and Python scripts | Remote-download-and-execute patterns, destructive operations, sensitive-file access |
| Dependency declarations | Unpinned versions, unusual sources, missing integrity metadata where applicable |
| Repository metadata | Installation hooks, hidden execution paths, mismatched declared and observed files |

Each format should be added only with fixtures and clearly stated coverage. A file extension alone should not be treated as proof that a file is safe or unsafe.

## Intended workflow

```text
Local folder / archive / repository checkout
                    |
                    v
            Safe file discovery
                    |
                    v
       Format-aware parsing and limits
                    |
                    v
          Deterministic rule engine
                    |
                    v
      Findings + file/line evidence
                    |
          +---------+---------+
          |                   |
          v                   v
   Human-readable CLI     JSON / CI output
          |
          v
      Human review
```

The basic scanner is designed to inspect local content. It should not automatically execute discovered commands, follow arbitrary links, fetch remote files, or unpack untrusted archives without explicit, bounded handling.

## Example CLI design

The commands below illustrate the intended interface; they are not available yet.

```bash
# Scan a local project directory
agent-supply-chain-sentinel scan ./candidate-skill

# Emit a machine-readable report
agent-supply-chain-sentinel scan ./candidate-skill --format json

# Fail CI when findings meet a configured threshold
agent-supply-chain-sentinel scan ./candidate-skill --fail-on high
```

The final CLI syntax may change as the implementation and usability tests evolve.

## Example finding

Illustrative report structure:

```json
{
  "schema_version": "0.1",
  "scanner": "agent-supply-chain-sentinel",
  "findings": [
    {
      "rule_id": "EXEC-REMOTE-001",
      "severity": "high",
      "title": "Remote content is piped to a shell",
      "path": "scripts/setup.sh",
      "line": 18,
      "evidence": "curl ... | sh",
      "why_it_matters": "Downloaded content may execute without an inspection step.",
      "recommendation": "Pin the source, verify integrity, and review the script before execution.",
      "confidence": "pattern-match"
    }
  ]
}
```

Evidence should be minimally sufficient, escaped for terminal output, and redacted when it could contain credentials. Rules need stable IDs so reports and suppressions remain traceable across releases.

## Design principles

### Deterministic rules first

Security-critical checks should be reproducible. The same content and rule-set version should produce the same findings, apart from explicitly documented metadata. A future optional LLM explanation layer may assist review, but it must not hide findings or override rule results.

### Evidence before severity

A useful finding identifies what matched and where. Severity should reflect the potential impact and the conditions required for exploitation—not merely the presence of a scary word. Where context is insufficient, use a review-oriented severity or confidence label.

### Bounded parsing

Treat candidate files as untrusted input. Use file-size limits, archive expansion limits, path normalization, timeouts where relevant, and graceful handling of invalid encodings and malformed syntax.

### Explicit suppressions

Suppressions should be narrow, reviewable, and optionally expiring. A suppression should identify a rule and a specific target with a rationale; blanket “ignore all” controls should be avoided in examples and defaults.

### Clear coverage

A clean scan means no configured rule matched the content examined. It does not mean the component is safe, its provenance is verified, or every execution path was analyzed.

## Intended MVP

1. Document supported file types and the initial threat model.
2. Implement safe local file discovery with deterministic traversal and limits.
3. Define a finding schema and stable rule identifiers.
4. Add a small set of high-signal, explainable rules.
5. Support line-level evidence for text formats where possible.
6. Provide terminal output and JSON export.
7. Add configurable severity thresholds and narrow suppressions.
8. Build a test corpus containing safe examples, unsafe examples, false-positive cases, malformed inputs, and boundary cases.
9. Add CI, packaging, reproducible installation, and documented examples.

Potential later extensions include repository scanning in CI, baseline comparison, signed release metadata checks, richer dependency analysis, and integrations with agent registries. Each extension should be scoped and validated independently.

## Proposed technology choices

These are proposals to assess during implementation, not dependencies already present in the repository.

- **Language:** Python 3.12+
- **Parsing:** standard library plus focused YAML/TOML parsers where required
- **CLI:** Typer or argparse
- **Validation:** typed data models and schema validation
- **Testing:** pytest with fixture-driven and adversarial tests
- **Quality:** Ruff, type checking, and GitHub Actions
- **Packaging:** Python package and command-line entry point after the CLI stabilizes

Prefer deterministic local analysis and a small dependency footprint for the first release.

## Security considerations

The scanner itself processes potentially hostile content and must be designed accordingly.

- Never import or execute scanned Python files.
- Never run shell commands extracted from a manifest.
- Treat file names, paths, terminal control characters, and evidence strings as untrusted.
- Avoid following symlinks outside the approved scan root unless a future opt-in mode makes the boundary explicit.
- Prevent archive path traversal and decompression/resource-exhaustion attacks if archive scanning is added.
- Do not include raw secrets in findings, console output, test snapshots, or uploaded CI artifacts.
- Make network access unnecessary for the local MVP. If online reputation or remote metadata checks are added later, isolate them behind explicit configuration and document what leaves the machine.
- Pin and review the scanner's own dependencies; a supply-chain scanner also has a supply chain.

## Testing strategy

The test corpus should include:

- known-safe scripts that resemble risky patterns but are harmless in context;
- remote execution and destructive command examples;
- broad environment forwarding and sensitive-file access;
- malformed YAML/JSON/TOML and invalid encodings;
- path traversal, symlinks, hidden files, large files, and deeply nested directories;
- terminal escape sequences in filenames or evidence;
- rules with multiple locations and overlapping matches;
- suppression expiry, invalid suppression scopes, and threshold behavior;
- deterministic output and JSON schema compatibility.

Measure false positives and false negatives on a documented fixture corpus. Do not advertise detection rates without a repeatable evaluation method.

## Roadmap

- [ ] **Milestone 0 — Threat model:** supported formats, severity semantics, and safe parsing boundaries.
- [ ] **Milestone 1 — Scanner core:** file discovery, normalization, rule registry, and finding schema.
- [ ] **Milestone 2 — First rules:** high-signal rules with positive, negative, and edge-case fixtures.
- [ ] **Milestone 3 — CLI and reports:** readable output, JSON output, and exit-code contract.
- [ ] **Milestone 4 — CI usability:** thresholds, scoped suppressions, baseline handling, and sample workflow.
- [ ] **Milestone 5 — Package and release:** installable CLI, versioning, documentation, and reproducible checks.
- [ ] **Milestone 6 — Coverage review:** false-positive assessment, adversarial inputs, and published limitations.

## Contributing

Before adding a detection rule, open or update an issue describing the threat, evidence pattern, expected severity, known false positives, and safe/unsafe fixtures. A rule should be explainable to a reviewer and testable without network access.

Never include real credentials, proprietary skills, or private incident material in public tests. Use synthetic fixtures or appropriately redacted samples.

## License

A license has not yet been selected for this repository. Until a license file is added, do not assume the project is available for unrestricted reuse.

## Disclaimer

This project is a work in progress. Static analysis provides useful signals, not a proof of safety or maliciousness. Do not use a clean report as the sole reason to trust an agent component.
