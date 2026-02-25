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

Full runs write to `docs/security/runs/{YYYY-MM-DD-HHMMSS}/`:

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

## References

- [references/architecture.md](references/architecture.md) — Architecture, LOD system, agent design
- [references/constants.md](references/constants.md) — Paths, filenames, agent registry
- [references/plugins/README.md](references/plugins/README.md) — Plugin format and detection
