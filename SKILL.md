---
name: security-analyst
description: "Use when the user wants a security audit, penetration test, threat model, vulnerability hunt, security fix plan, SBOM, compliance mapping, privacy assessment, or security posture comparison between runs."
---

# Security Analyst

Offensive security analysis suite that coordinates specialized agents through a multi-group penetration testing pipeline. Finds real vulnerabilities with concrete exploits, not checkbox compliance.

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

## Available Commands

| Command | Purpose | Agents | Output |
|---------|---------|--------|--------|
| `/security-analyst` | Interactive — choose scope and output mode | Varies | Varies |
| `/security-analyst:full` | All 9 phases, all attack surfaces | 23+ | Full run directory |
| `/security-analyst:focused [component]` | Target a specific area (e.g., authentication, data pipeline) | 3-6 | Inline findings |
| `/security-analyst:recon` | Phase 0 only — codebase security map | 14 (parallel) | Recon index + step files |
| `/security-analyst:threat-model` | Recon + STRIDE analysis + attack trees | 1+ | Threat model doc |
| `/security-analyst:variant-hunt [vuln]` | Find all unpatched variants of a known vulnerability | 2-6 | Variant findings |
| `/security-analyst:fix-plan [run-dir]` | Generate implementation plan from existing report | 1 | Fix plan |
| `/security-analyst:sbom` | Software Bill of Materials — recon + dependency audit only | 15 | SBOM report |
| `/security-analyst:diff [run-a] [run-b]` | Compare two runs — new, resolved, persistent findings | 1 | Delta report |
| `/security-analyst:compliance [framework]` | Map findings to SOC 2 / ISO 27001 / PCI DSS / HIPAA / GDPR | 1 | Compliance report |
| `/security-analyst:privacy` | Privacy assessment — PII flows, consent, data subject rights | 15+ | Privacy report |

## How It Works

### Group-Based Pipeline

Analysis runs in dependency-ordered execution groups. Each group's findings feed the next. Groups are execution-order groupings of phases. Finding IDs use category prefixes (HTTP, INJ, DEP, etc.).

```
Recon: Reconnaissance (14 agents, all parallel — 2 waves)
  └─ Wave A (12 agents, batch-spawned): metadata, docs, HTTP, boundaries, crown jewels, auth, integrations, secrets, security work, config, frontend, deps
  └─ Wave B (2 agents, batch-spawned): data flows, scope notes (depend on Wave A)
  └─ Assembly: orchestrator builds recon/index.md from LOD-0+1 returns

Surface: Attack Surface + Git History + Dependencies + Config (up to 16 agents, batch-spawned in parallel)
  ├─ HTTP entry points, authz rules, integrations, frontend
  ├─ LLM/AI security (OWASP Top 10 for LLM Applications)
  ├─ API schema validation (OpenAPI, GraphQL, gRPC)
  ├─ WebSocket / SSE real-time security
  ├─ File upload security
  ├─ Injection/auth/SSRF/data-exposure variant hunting via git history
  ├─ Dependency audit (npm audit, supply chain)
  ├─ Infrastructure config, secrets, KMS, IAM
  ├─ CI/CD pipeline security (GitHub Actions, GitLab CI)
  └─ Container security (Docker, Kubernetes)

┌─ SBOM Assembly (orchestrator, no agent — runs in parallel with logic stage)
│
Logic: Business Logic (4 agents batch-spawned, needs surface stage)
  ├─ Race conditions and TOCTOU
  ├─ Authorization escalation and IDOR
  ├─ Pipeline exploitation (input → AI → decision → action)
  └─ DoS and resource exhaustion

Tracing: Data Flow Tracing (up to 4 agents batch-spawned, needs surface + logic)
  └─ End-to-end trace of critical data flows with sanitization gap analysis

Exploits: Exploit Development (1 agent, needs all findings)
  └─ Develops complete exploits with PoCs, CVSS scores, CWE/ATT&CK IDs, chains

Validation: Finding Validation (1 critic agent)
  └─ Adversarial review — catches false positives, validates fixes, adjusts severity

Reporting: Final Report (1 agent)
  └─ Executive summary, risk dashboard, remediation roadmap

Remediation: Fix Plan (1 agent)
  └─ Actionable tasks with fix code, regression tests, effort estimates
```

### Finding Format

Every Medium+ finding includes: CVSS 3.1 score, CWE ID, MITRE ATT&CK technique, exploit scoring (exploitability/blast radius/detectability/remediation ROI), attack steps, proof-of-concept, concrete fix code, and regression test.

### Output Directory

Full runs write to `docs/security/runs/{YYYY-MM-DD-HHMMSS}/`:

| File | Content |
|------|---------|
| `recon/index.md` | Recon LOD-0 + LOD-1 index |
| `recon/step-*.md` | 14 atomic LOD-2 recon section files |
| `findings/` | Atomic LOD-2 finding files + index |
| `reports/surface.md` | Attack surface, git history, deps, config |
| `reports/sbom.md` | Software Bill of Materials (deps, licenses, supply chain) |
| `reports/logic.md` | Business logic exploitation |
| `reports/tracing.md` | Data flow tracing |
| `reports/exploits.md` | Exploit catalog with PoCs |
| `reports/validation.md` | Validated findings |
| `reports/final.md` | Complete security report |
| `reports/fix-plan.md` | Implementation plan |
| `reports/compliance.md` | Compliance mapping (when requested) |
| `reports/delta.md` | Delta between two runs (when requested) |
| `reports/privacy.md` | Privacy & data protection assessment (when requested) |

## Command Details

### Full Analysis

Runs all 9 phases with no prompts. Most thorough — spawns up to 28+ agents across 6 execution groups. Expect significant processing time.

```
/security-analyst:full
```

### Focused Analysis

Analyzes a specific component with only relevant agents. Faster (3-6 agents) but still thorough within scope. Findings presented inline — no separate report.

```
/security-analyst:focused authentication
/security-analyst:focused data processing pipeline
/security-analyst:focused frontend
/security-analyst:focused database rules
/security-analyst:focused AI integration
/security-analyst:focused CI/CD
/security-analyst:focused containers
/security-analyst:focused API
/security-analyst:focused real-time
/security-analyst:focused file uploads
```

### Reconnaissance Only

Quick security map without vulnerability analysis. Good for orientation before a full audit or for manual review planning.

```
/security-analyst:recon
```

### Threat Model

Recon plus STRIDE analysis on each trust boundary, attack trees for each crown jewel, and a risk matrix. Useful for compliance (SOC 2, ISO 27001) and security planning.

```
/security-analyst:threat-model
```

### Variant Hunt

Given a known vulnerability (commit hash, CVE, or description), systematically finds all unpatched variants using git history mining and pattern analysis.

```
/security-analyst:variant-hunt 6dc9bb7
/security-analyst:variant-hunt OData injection
/security-analyst:variant-hunt CVE-2024-XXXX
```

### Software Bill of Materials

Generates a compliance-ready SBOM covering all languages, frameworks, direct and transitive dependencies, licenses, and supply chain health indicators. Runs recon plus the dependency audit agent only — no exploitation or attack surface analysis.

```
/security-analyst:sbom
```

### Delta Report

Compares two security analysis runs to highlight new, resolved, and persistent findings. Tracks dependency changes, attack surface evolution, and fix plan progress. Auto-detects the two most recent runs or accepts specific run directories.

```
/security-analyst:diff
/security-analyst:diff docs/security/runs/2025-06-01-100000 docs/security/runs/2025-06-15-143022
```

### Compliance Mapping

Maps findings from an existing security run to a compliance framework. Assesses each control as Pass/Partial/Fail/N/A and produces a compliance score with gap analysis.

```
/security-analyst:compliance soc2
/security-analyst:compliance iso27001
/security-analyst:compliance pci-dss
/security-analyst:compliance hipaa
/security-analyst:compliance gdpr
```

### Privacy Assessment

Traces all PII data flows, evaluates consent mechanisms, checks data subject rights implementation (access, erasure, portability), reviews data retention and deletion, and assesses cross-border transfers. Can run standalone or enrich an existing security run.

```
/security-analyst:privacy
```

### Fix Plan

Generates an implementation plan from an existing security report. Auto-detects the latest run or accepts a specific run directory. Tasks are prioritized: Quick Wins, Immediate, Short-term, Medium-term, Backlog.

```
/security-analyst:fix-plan
/security-analyst:fix-plan docs/security/runs/2025-06-15-143022
```

## Architecture

For detailed architecture reference (directory structure, agent design, LOD architecture, customization), see [references/architecture.md](references/architecture.md). For all constants (paths, filenames, agent registry, placeholders), see [references/constants.md](references/constants.md).

**Key concepts:**
- **LOD system**: Three-tier Level of Detail (LOD-0/1/2) saves 80-96% of prompt tokens for downstream agents while preserving full detail on disk
- **Group discipline**: Later groups depend on earlier findings; never start a group early
- **Dynamic scoping**: Recon determines which agents to skip
- **Framework plugins**: 16 auto-detected plugins inject framework-specific checks into agent prompts. See [references/plugins/README.md](references/plugins/README.md)
- **Critic validation**: Adversarial review catches false positives before the final report

## Installation

Copy or symlink this skill folder to your skills directory:

```bash
# Claude Code
cp -r . ~/.claude/skills/security-analyst

# Cursor
cp -r . ~/.cursor/skills/security-analyst
```

## Troubleshooting

**Skill doesn't trigger on security-related requests**
- Ensure the description includes your trigger phrase. Try: "run a security audit", "penetration test", "threat model", "vulnerability scan", "SBOM", "compliance mapping"

**MCP or subagent connection fails during analysis**
- Verify the environment supports mcp_task/subagent spawning (Claude Code, Cursor with appropriate tools)
- Check that the project being analyzed is the current workspace so recon agents can Read/Glob the codebase

**Findings seem generic or lack concrete exploits**
- The skill expects "offensive framing" — findings should include attack steps and PoC. If outputs are shallow, re-run with a focused scope (e.g., `/security-analyst:focused authentication`) for deeper analysis
