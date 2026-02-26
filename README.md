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

### Plugin install (recommended)

```
/plugin marketplace add paixaop/security-analyst
/plugin install security-analyst
```

This works in both Claude Code and Cursor.

### Manual install

```bash
# Clone and run the install script
git clone https://github.com/paixaop/security-analyst.git
cd security-analyst
./install.sh

# Or copy directly
cp -r plugin/skills/security-analyst ~/.claude/skills/security-analyst   # Claude Code
cp -r plugin/skills/security-analyst ~/.cursor/skills/security-analyst   # Cursor
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

Full runs write to `docs/security/runs/{YYYY-MM-DD-HHMMSS}/`. To change the output location, edit the `RUN_DIR` path in [references/constants.md](plugin/skills/security-analyst/references/constants.md).

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
| `reports/executive-report.md` | Executive security report |
| `reports/fix-plan.md` | Implementation plan |
| `reports/project-recon.md` | Full LOD-2 consolidated recon report |
| `reports/compliance.md` | Compliance mapping (when requested) |
| `reports/delta.md` | Delta between runs (when requested) |
| `reports/privacy.md` | Privacy assessment (when requested) |
| `checkpoint.md` | Stage completion tracking (enables resume) |

### Recon (`recon/`)

The reconnaissance phase is the foundation — every downstream agent reads it. 14 parallel agents map the codebase across: project metadata and tech stack, documentation index, HTTP entry points (authenticated and unauthenticated), trust boundaries, crown jewels and attacker profiles, auth mechanisms and session management, external integrations and SSRF surface, secrets and encryption posture, critical data flows (registration, auth, payment, exports), existing security work and prior fixes, configuration and security headers, frontend surface (rendering, client storage, service workers), dependencies with security relevance, and scope notes with limitations. The index aggregates summaries from all 14 areas into a single document, with every entry linked to exact file:line references so downstream agents can jump directly to relevant code.

Recon produces no findings and no exploits — it is purely a map. This makes `/security-analyst:recon` safe to run as a first step.

### Executive Report (`reports/executive-report.md`)

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
- **Large codebase handling**: Six mechanisms prevent context exhaustion (see below)
- **Framework plugins**: 16 auto-detected plugins (Firebase, React, Django, etc.) inject framework-specific checks
- **Critic validation**: Adversarial review before final report

## Large Codebase Handling

A full analysis spawns 28+ agents that read source code, grep for patterns, and accumulate findings. On large codebases (hundreds of endpoints, dozens of integrations), individual agents and the orchestrator itself can exhaust their context windows. Six mechanisms prevent this:

### 1. File exclusions (.gitignore + hardcoded list)

All agents respect `.gitignore` automatically — the Grep tool uses ripgrep which skips gitignored paths by default. Agents also use `git ls-files` to enumerate tracked source files instead of broad glob patterns. An additional hardcoded list in `agent-common.md` excludes minified files, source maps, test snapshots, generated code, and lockfiles from broad searches. This prevents a single unscoped grep from pulling `node_modules/` into an agent's context.

### 2. Targeted reads (line ranges, not whole files)

Instead of reading entire source files, agents read ~60 lines centered on the `file:line` reference from recon step files. A 2000-line file read in full uses ~20x more context than the ~100-line function body the agent actually needs. Agents expand incrementally in 30-line steps only if they need more context to trace a specific code path.

### 3. Work partitioning (orchestrator-level chunking)

After recon, the orchestrator counts targets per agent (endpoints, integrations, rule files, findings). When a count exceeds the agent's threshold (e.g., 25 endpoints for the HTTP agent), the work is split into chunks and multiple instances of the same agent run in parallel, each covering a subset. Finding ID offsets prevent collisions. See `references/constants.md` for per-agent thresholds.

### 4. PARTIAL return protocol (agent-level safety valve)

Any agent can return early with `PARTIAL` status when approaching its context budget. The agent writes all findings discovered so far to disk and returns a structured list of remaining targets. The orchestrator spawns a continuation agent for the remaining work and merges the results. This catches cases where pre-chunking underestimates context usage.

### 5. Stage-based dispatching (fresh context per stage)

Each analysis stage (surface, logic, tracing, exploits, validation, reporting, remediation) runs inside its own **stage-orchestrator Task** — a subagent with a fresh context window. The stage orchestrator reads prompt templates from disk, spawns its analysis agents, consolidates findings, writes the stage report, and returns a compact summary. The top-level orchestrator stays under ~20K tokens instead of accumulating ~100K+ from all stages.

### 6. Checkpoint and resume

A checkpoint file (`checkpoint.md`) is written to the run directory after each stage completes, recording stage status, configuration (skipped agents, matched plugins, partition data), and paths. If the orchestrator is interrupted — by context exhaustion, user abort, or error — a new session can resume from the checkpoint without re-running completed stages.

**On small/medium codebases** (under chunking thresholds), these mechanisms add no overhead. The system runs exactly as it would without them — single agent instances, no chunking, no PARTIAL returns.

## Project Structure

```
security-analyst/                           # GitHub repo root
├── .claude-plugin/
│   └── marketplace.json                    # Marketplace manifest
├── README.md                               # This file
├── LICENSE
├── CONTRIBUTING.md                         # Plugin authoring guide
├── install.sh                              # Legacy manual install
├── docs/                                   # Dev-only artifacts
│
└── plugin/                                 # The installable plugin
    ├── .claude-plugin/
    │   └── plugin.json                     # Plugin manifest
    ├── .cursor-plugin/
    │   └── plugin.json                     # Cursor compatibility
    └── skills/
        └── security-analyst/               # The skill folder
            ├── SKILL.md
            ├── references/
            │   ├── architecture.md
            │   ├── constants.md
            │   ├── commands/               # Entry points
            │   ├── prompts/                # Agent prompt templates
            │   └── plugins/                # Framework-specific checks (16 plugins)
            └── assets/
                └── templates/              # Output format templates
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

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add framework-specific plugins and contribute to the project.

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

- [plugin/skills/security-analyst/references/architecture.md](plugin/skills/security-analyst/references/architecture.md) — Architecture, LOD system, agent design
- [plugin/skills/security-analyst/references/constants.md](plugin/skills/security-analyst/references/constants.md) — Paths, filenames, agent registry
- [plugin/skills/security-analyst/references/plugins/README.md](plugin/skills/security-analyst/references/plugins/README.md) — Plugin format and detection
