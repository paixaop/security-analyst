# Security Analyst

[![GitHub](https://img.shields.io/github/stars/paixaop/security-analyst?style=social)](https://github.com/paixaop/security-analyst)

Comprehensive security analysis skill for Claude Code, Cursor, and Codex. Identifies vulnerabilities, assesses exploitability, and generates actionable remediation plans. Coordinates specialized agents through a multi-phase pipeline — built for developers and security researchers who want depth, not checkbox compliance.

**Repository:** https://github.com/paixaop/security-analyst

> **Note:** A full analysis spawns 20+ specialized agents across multiple phases. On large codebases this can take significant time and consume substantial context. For a quicker first look, start with `/security-analyst:recon` (codebase map only) or `/security-analyst:focused [component]` (targets a single area with 3-6 agents).

## When to Use

- Before deploying to production or after major feature work
- When a security audit or penetration test is requested
- After a vulnerability is reported, to find unpatched variants
- To generate a threat model for compliance or planning
- To create a fix plan from an existing security report
- To generate a Software Bill of Materials (SBOM) for compliance
- To map findings against SOC 2, ISO 27001, PCI DSS, HIPAA, or GDPR controls
- To compare security posture between two analysis runs (delta/regression tracking)
- To conduct a privacy and data protection assessment (PII flows, consent, data subject rights)

## Quick Start

| Command | Purpose |
|---------|---------|
| `/security-analyst` | Interactive — choose scope and output mode |
| `/security-analyst:full` | All phases, all attack surfaces |
| `/security-analyst:focused [component]` | Target one area (auth, API, frontend, etc.) |
| `/security-analyst:recon` | Reconnaissance only — codebase security map |
| `/security-analyst:threat-model` | STRIDE, attack trees, risk matrix |
| `/security-analyst:variant-hunt [vuln]` | Find unpatched variants of a known vuln |
| `/security-analyst:fix-plan [run-dir]` | Implementation plan from existing report |
| `/security-analyst:sbom` | Software Bill of Materials |
| `/security-analyst:diff [run-a] [run-b]` | Compare two runs — new, resolved, persistent |
| `/security-analyst:compliance [framework]` | Map findings to compliance controls |
| `/security-analyst:privacy` | PII flows, consent, data subject rights |

## Before You Install

This skill performs comprehensive security analysis. It instructs agents to read large portions of your repository (including environment and secrets files), run dependency auditors, mine git history, and write proof-of-concept reproductions and reports to disk. Before installing or running it:

1. **Only run on codebases you control** — You must have explicit permission to perform security testing on the target project and environment.
2. **Use an isolated workspace** — Run inside a disposable environment, not a production system or a workspace containing real credentials. If you must analyze a production codebase, clone it into a sandboxed directory first.
3. **Expect persistent artifacts** — Generated PoCs, exploit chains, and fix code are written to `docs/security/runs/` by default. Choose a safe output directory (configurable in `references/constants.md`), review artifacts promptly, and delete or secure them after triage.
4. **Verify tooling availability** — The skill requires `git` and benefits from `npm audit`, `pip-audit`, `govulncheck`, and other ecosystem tools. See [Requirements](#requirements) for the full list and fallback behavior.
5. **Run supervised first** — If you are unsure about scope or safety, do not enable autonomous invocation. Run a manual, supervised test on a throwaway repository before trusting it on a real codebase. The `/security-analyst:recon` command is a good low-risk starting point — it maps the codebase without producing exploits.

## Installation

Copy or symlink this skill to your skills directory:

```bash
# Claude Code
cp -r . ~/.claude/skills/security-analyst

# Cursor
cp -r . ~/.cursor/skills/security-analyst
```

## Requirements

### Workspace Access

The skill requires **read access** to the project codebase and **write access** to the project directory. Agents use Read, Grep, and Glob to analyze source files, and full runs write output to `docs/security/runs/{YYYY-MM-DD-HHMMSS}/` inside the project workspace. The project being analyzed must be the current working directory.

### Platform Tools

All agents are spawned as `generalPurpose` subagents via the **Task** tool and require:

| Tool | Purpose |
|------|---------|
| **Task** | Spawn and coordinate subagents (Claude Code, Cursor, Codex) |
| **Bash / Shell** | Run git commands, dependency auditors, and inline scripts |
| **Read** | Read source files, configs, lockfiles, and prior-stage outputs |
| **Grep** | Search for patterns across the codebase |
| **Glob** | Find files by name/extension patterns |
| **Write** | Write findings, reports, and recon step files to the run directory |

### Required: `git`

Git is a **hard dependency**. Four agents mine git history (`git log`, `git diff`, `git show`, `git blame`) to find unpatched vulnerability variants. The dependency audit agent uses `git ls-files` to verify lockfile integrity. The variant-hunt command relies entirely on git history analysis. If the project is not a git repository, history-based agents will produce no findings and the variant-hunt command will not function.

### Language-Specific Auditors (used when available)

The dependency audit agent attempts to run vulnerability scanners for detected package ecosystems. These tools are **not required** — if unavailable, the agent falls back to static analysis of manifest and lockfiles, but automated CVE detection will be less thorough.

| Ecosystem | Tool | Invocation | Fallback |
|-----------|------|------------|----------|
| Node.js | `npm` | `npm audit --json` | Parse `package.json` + `package-lock.json` directly |
| Python | `pip-audit` | `pip-audit --json` | Parse `requirements.txt` / `pyproject.toml` directly |
| Go | `govulncheck` | `govulncheck ./...` | Parse `go.mod` / `go.sum` directly |

### SBOM Export Tools (optional)

The SBOM report template references machine-readable export tools. These are **documentation-only recommendations** — the skill does not attempt to run them automatically. The SBOM report is always generated from the dependency audit agent's static analysis regardless of whether these tools are installed.

| Format | Tool |
|--------|------|
| CycloneDX | `npx @cyclonedx/cyclonedx-npm`, `cyclonedx-py`, `cyclonedx-gomod` |
| SPDX | `spdx-sbom-generator`, `syft` |

### Credentials

No credentials, API keys, or authentication tokens are required. All analysis is performed locally against the project source code and git history. The compliance and privacy commands assess implementation by reading code — they do not connect to external compliance platforms.

## Output

Full runs write to `docs/security/runs/{YYYY-MM-DD-HHMMSS}/`. To change the output location, edit the `RUN_DIR` path in [references/constants.md](references/constants.md).

| Path | Content |
|------|---------|
| `recon/` | Codebase security map — project metadata, HTTP entry points, trust boundaries, crown jewels, auth mechanisms, external integrations, secrets & encryption, data flows, existing security work, configuration, frontend surface, dependencies, and scope notes |
| `findings/` | Atomic finding files + index |
| `reports/surface.md` | Attack surface, git history, deps, config |
| `reports/sbom.md` | Software Bill of Materials |
| `reports/logic.md` | Business logic exploitation |
| `reports/tracing.md` | Data flow tracing |
| `reports/exploits.md` | Exploit catalog with PoCs |
| `reports/validation.md` | Validated findings |
| `reports/final.md` | Complete security report |
| `reports/fix-plan.md` | Implementation plan |
| `reports/compliance.md` | Compliance mapping (when requested) |
| `reports/delta.md` | Delta between runs (when requested) |
| `reports/privacy.md` | Privacy assessment (when requested) |

### Recon (`recon/`)

The reconnaissance phase is the foundation — every downstream agent reads it. 14 parallel agents map the codebase across: project metadata and tech stack, documentation index, HTTP entry points (authenticated and unauthenticated), trust boundaries, crown jewels and attacker profiles, auth mechanisms and session management, external integrations and SSRF surface, secrets and encryption posture, critical data flows (registration, auth, payment, exports), existing security work and prior fixes, configuration and security headers, frontend surface (rendering, client storage, service workers), dependencies with security relevance, and scope notes with limitations. The index aggregates summaries from all 14 areas into a single document, with every entry linked to exact file:line references so downstream agents can jump directly to relevant code.

Recon produces no findings and no exploits — it is purely a map. This makes `/security-analyst:recon` safe to run as a first step.

### Final Report (`reports/final.md`)

The main deliverable for stakeholders. Contains an executive summary (overall posture, critical findings, top 3 actions), a risk dashboard (severity x confidence matrix with exploit chain count), all Critical/High/Medium findings with full exploit scenarios ordered by CVSS, Low/Informational findings in brief table format, exploit chains showing how multiple findings combine into multi-step attacks, root cause analysis grouping findings by underlying cause with systemic fixes, MITRE ATT&CK tactic coverage with gap analysis, a remediation roadmap (quick wins, immediate, short-term, medium-term, backlog), comparison with previously identified security work, positive security observations, methodology notes, and critic validation statistics (confirmed/downgraded/removed/upgraded findings).

### Fix Plan (`reports/fix-plan.md`)

Converts findings into actionable implementation tasks organized by priority: Quick Wins (high ROI + high exploitability), Immediate (critical findings, before next deploy), Short-term (remaining highs, within 1 week), Medium-term (mediums + systematic improvements, within 1 month), and Backlog (lows + defense-in-depth). Each task includes the exact files to modify, concrete fix code (not descriptions), a regression test to prevent reintroduction, effort estimate, and a suggested commit message. A root cause fixes section groups systemic changes that resolve multiple findings at once.

## Pipeline Overview

```
Recon (14 agents, parallel)
    ↓
Surface (≤16 agents) — HTTP, auth, integrations, frontend, LLM, API, deps, config, CI/CD, containers
    ↓
Logic (4 agents) — race conditions, authz escalation, pipeline exploitation, DoS
    ↓
Tracing (≤4 agents) — data flow + sanitization gaps
    ↓
Exploits (1) → Validation (1 critic) → Reporting (1) → Remediation (1 fix plan)
```

- **LOD system**: Three tiers reduce prompt tokens while keeping full detail on disk
- **Framework plugins**: 16 auto-detected plugins (Firebase, React, Django, etc.) inject framework-specific checks
- **Critic validation**: Adversarial review before final report

## Project Structure

```
security-analyst/
├── SKILL.md
├── README.md (this file)
├── references/
│   ├── architecture.md    # Architecture reference
│   ├── constants.md       # Paths, agent registry, templates
│   ├── commands/          # Entry points
│   ├── prompts/           # Agent prompt templates
│   └── plugins/           # Framework-specific checks (16 plugins)
└── assets/
    └── templates/         # Output format templates
```

## Troubleshooting

**Skill doesn't trigger**
- Use phrases like "run a security audit", "penetration test", "threat model", "SBOM", "compliance mapping"

**Analysis hangs / model waits forever**
- On Cursor: the orchestrator may be calling TaskList/TaskCreate which don't exist. See `references/platform-tools.md` for the correct flow.

**Subagent fails**
- Verify the environment supports the Task tool for subagent spawning
- Ensure the project being analyzed is the current workspace

**Findings seem generic**
- Re-run with a focused scope: `/security-analyst:focused authentication`

## Contributing a Plugin

Plugins add framework-specific security checks that are automatically injected into agent prompts when the framework is detected. There are currently 16 plugins (Firebase, React, Django, Next.js, Spring Boot, etc.). To add a new one:

### 1. Create the plugin file

Add a markdown file to `references/plugins/` using lowercase with hyphens (e.g., `my-framework.md`).

### 2. Add frontmatter with detection criteria

```yaml
---
name: "My Framework"
detect:
  files: ["my-framework.config.js", "my-framework.json"]
  dependencies: ["my-framework", "@my-framework/core"]
  keywords: ["My Framework"]
---
```

A plugin matches if **any single** criterion hits:

| Field | How it matches |
|-------|---------------|
| `detect.files` | File exists in the project root (glob supported) |
| `detect.dependencies` | Package found in `step-13-dependencies.md` recon output |
| `detect.keywords` | String match in `step-01-metadata.md` (Framework/Infrastructure/Runtime fields) |

### 3. Add agent sections

Each `## {agent-name}` section contains framework-specific checks for that agent. Only include sections relevant to your framework — omitted sections are silently skipped.

Available agent sections:

| Section | Phase | Focus |
|---------|-------|-------|
| `recon-agent` | Recon | Additional discovery patterns |
| `attack-surface-http` | Surface | HTTP endpoint patterns |
| `attack-surface-authz` | Surface | Authorization analysis |
| `attack-surface-frontend` | Surface | Client-side security |
| `attack-surface-llm` | Surface | AI/LLM integrations |
| `attack-surface-integrations` | Surface | External API patterns |
| `git-history-injections` | Surface | Injection attack vectors |
| `git-history-auth-bypass` | Surface | Auth bypass patterns |
| `git-history-ssrf` | Surface | SSRF patterns |
| `git-history-data-exposure` | Surface | Data leakage patterns |
| `logic-race-conditions` | Logic | Concurrency issues |
| `logic-authz-escalation` | Logic | Privilege escalation |
| `logic-pipeline-exploit` | Logic | Pipeline logic flaws |
| `logic-dos` | Logic | Denial of service |
| `data-flow-tracer` | Tracing | Data flow analysis |
| `dependency-audit` | Surface | Dependency vulnerabilities |
| `config-infrastructure` | Surface | Infrastructure config |

### 4. Write actionable checks

Each section should contain numbered checks with specific patterns to search for, concrete exploit scenarios, and framework-specific context. Example:

```markdown
## attack-surface-authz

1. Are Firestore security rules using `request.auth != null` without role checks? Grep for `allow read, write: if request.auth != null` in `firestore.rules`.
2. Can a user escalate by writing directly to a role field? Check if `users/{userId}` rules restrict which fields are writable.
```

Checks should be **complementary** to the generic agent checks, not repetitive. Focus on framework-specific gotchas, defaults, and attack vectors.

### 5. Test your plugin

Run `/security-analyst:recon` against a project that uses your framework. The recon output should trigger your plugin's detection criteria. Then run a focused analysis to verify your checks produce meaningful findings.

See [references/plugins/README.md](references/plugins/README.md) for the full plugin specification and existing plugins for reference.

## Disclaimer

This skill is provided for **authorized security testing only**. By using this skill you agree to the following:

- You may only run it against projects and environments you own or have explicit written authorization to test.
- You are solely responsible for ensuring that your use complies with all applicable laws, regulations, and policies.
- This skill must not be used to attack, exploit, or compromise systems you do not own or have permission to test.
- The authors are not responsible for any misuse, damage, or legal consequences resulting from the use of this skill.
- Output artifacts (exploit PoCs, attack chains, vulnerability reports) are sensitive — handle, store, and share them responsibly.

**If you do not have authorization to perform security testing on a target, do not use this skill against it.**

## License

Copyright (c) 2025 Pedro Paixao

This project is licensed under the [GNU Affero General Public License v3.0 (AGPL-3.0)](https://www.gnu.org/licenses/agpl-3.0.html).

You are free to use, modify, and distribute this software under the terms of the AGPL-3.0. If you modify this skill and make it available to others — including over a network — you must release your modifications under the same license and make the source code available.

See [LICENSE](LICENSE) for the full license text.

## References

- [references/architecture.md](references/architecture.md) — Architecture, LOD system, agent design
- [references/constants.md](references/constants.md) — Paths, filenames, agent registry
- [references/plugins/README.md](references/plugins/README.md) — Plugin format and detection
