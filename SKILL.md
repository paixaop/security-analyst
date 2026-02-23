---
name: security-analyst
description: "Deep offensive security analysis of any codebase — penetration tester mindset with team-based agent orchestration across 9 phases. Use when the user wants a security audit, penetration test, threat model, vulnerability hunt, or security fix plan."
---

# Security Analyst

Offensive security analysis suite that coordinates specialized agents through a multi-group penetration testing pipeline. Finds real vulnerabilities with concrete exploits, not checkbox compliance.

## When to Use

- Before deploying to production or after major feature work
- When a security audit or penetration test is requested
- After a vulnerability is reported, to find unpatched variants
- To generate a threat model for compliance or planning
- To create a fix plan from an existing security report

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

## How It Works

### Group-Based Pipeline

Analysis runs in dependency-ordered execution groups. Each group's findings feed the next. Groups are execution-order groupings of phases. Finding IDs use Phase numbers (P1-P9).

```
Group 0: Reconnaissance (14 agents, parallel — 2 waves)
  └─ Wave A (12 agents): metadata, docs, HTTP, boundaries, crown jewels, auth, integrations, secrets, security work, config, frontend, deps
  └─ Wave B (2 agents): data flows, scope notes (depend on Wave A)
  └─ Assembly: orchestrator builds recon/index.md from LOD-0+1 returns

Group 1: Attack Surface + Git History + Dependencies + Config (up to 11 agents, parallel)
  ├─ HTTP entry points, authz rules, integrations, frontend
  ├─ LLM/AI security (OWASP Top 10 for LLM Applications)
  ├─ Injection/auth/SSRF/data-exposure variant hunting via git history
  ├─ Dependency audit (npm audit, supply chain)
  └─ Infrastructure config, secrets, KMS, IAM

Group 2: Business Logic (4 agents, needs Group 1)
  ├─ Race conditions and TOCTOU
  ├─ Authorization escalation and IDOR
  ├─ Pipeline exploitation (input → AI → decision → action)
  └─ DoS and resource exhaustion

Group 3: Data Flow Tracing (up to 4 agents, needs Groups 1+2)
  └─ End-to-end trace of critical data flows with sanitization gap analysis

Group 4: Exploit Development (1 agent, needs all findings)
  └─ Develops complete exploits with PoCs, CVSS scores, CWE/ATT&CK IDs, chains

Group 4.5: Finding Validation (1 critic agent)
  └─ Adversarial review — catches false positives, validates fixes, adjusts severity

Group 5: Final Report (1 agent)
  └─ Executive summary, risk dashboard, remediation roadmap

Post: Fix Plan (1 agent)
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
| `group-1-attack-surface.md` | Attack surface, git history, deps, config |
| `group-2-business-logic.md` | Business logic exploitation |
| `group-3-data-flow.md` | Data flow tracing |
| `group-4-exploits.md` | Exploit catalog with PoCs |
| `group-4.5-critic-review.md` | Validated findings |
| `security-analyst-final-report.md ` | Complete security report |
| `fix-plan.md` | Implementation plan |

## Command Details

### Full Analysis

Runs all 9 phases with no prompts. Most thorough — spawns 23+ agents across 6 execution groups. Expect significant processing time.

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

### Fix Plan

Generates an implementation plan from an existing security report. Auto-detects the latest run or accepts a specific run directory. Tasks are prioritized: Quick Wins, Immediate, Short-term, Medium-term, Backlog.

```
/security-analyst:fix-plan
/security-analyst:fix-plan docs/security/runs/2025-06-15-143022
```

## Architecture

### Directory Structure

```
.claude/commands/security-analyst/
├── SKILL.md                              # This file
├── security-analyst.md                   # Main orchestrator (interactive)
├── full.md                               # Full analysis entry point
├── focused.md                            # Focused analysis entry point
├── recon.md                              # Recon-only entry point
├── threat-model.md                       # Threat model entry point
├── variant-hunt.md                       # Variant hunt entry point
├── fix-plan.md                           # Fix plan entry point
├── prompts/                              # Agent prompt templates (20 files)
│   ├── recon-agent.md                    # Phase 0: reconnaissance
│   ├── attack-surface-http.md            # Phase 1: HTTP entry points
│   ├── attack-surface-authz.md           # Phase 1: database/authz rules
│   ├── attack-surface-integrations.md    # Phase 1: external integrations
│   ├── attack-surface-frontend.md        # Phase 1: frontend security
│   ├── attack-surface-llm.md            # Phase 1: OWASP Top 10 for LLM Applications
│   ├── git-history-injections.md         # Phase 2: injection variants
│   ├── git-history-auth-bypass.md        # Phase 2: auth bypass variants
│   ├── git-history-ssrf.md              # Phase 2: SSRF variants
│   ├── git-history-data-exposure.md      # Phase 2: data exposure variants
│   ├── logic-race-conditions.md          # Phase 3: TOCTOU / race conditions
│   ├── logic-authz-escalation.md         # Phase 3: privilege escalation
│   ├── logic-pipeline-exploit.md         # Phase 3: pipeline logic flaws
│   ├── logic-dos.md                      # Phase 3: DoS / resource exhaustion
│   ├── data-flow-tracer.md              # Phase 4: end-to-end flow tracing
│   ├── dependency-audit.md              # Phase 5: dependency vulnerabilities
│   ├── config-infrastructure.md          # Phase 6: infra config audit
│   ├── exploit-developer.md             # Phase 7: exploit development
│   ├── finding-critic.md               # Phase 7.5: finding validation
│   ├── report-writer.md                # Phase 8: final report
│   ├── fix-planner.md                  # Phase 9: fix plan generation
│   └── threat-model-agent.md           # Threat model: STRIDE + attack trees
├── plugins/                              # Framework-specific security checks (auto-detected)
│   ├── README.md                        # Plugin format spec and instructions
│   ├── firebase.md                      # Firebase / Firestore / Cloud Functions
│   ├── react.md                         # React / React Native
│   ├── nextjs.md                        # Next.js
│   ├── nodejs-express.md                # Node.js + Express
│   ├── python-fastapi.md                # Python + FastAPI
│   └── python-django.md                 # Python + Django
└── templates/                            # Output format templates (6 files)
    ├── recon-report.md                   # Recon LOD format definitions (LOD-0/1/2 per section)
    ├── finding.md                        # Individual finding format (LOD-0/1/2 definitions)
    ├── agent-common.md                   # Shared LOD output + incidental findings instructions
    ├── wave-report.md                    # Intermediate group report
    ├── final-report.md                   # Final report structure
    └── fix-plan.md                       # Fix plan structure
```

### Agent Design

All agents are `generalPurpose` subagents (need Bash, Read, Grep, Glob for code analysis and git history mining). Each agent:

1. Reads the recon index for LOD-0+1 overview, selectively reads LOD-2 step files for relevant sections
2. Reads prior group findings (if applicable)
3. Performs its specialized analysis
4. Reports findings using the standard finding template
5. Reports incidental findings (IX-prefixed) for issues outside its focus area

### Key Principles

- **Offensive framing**: Every analysis starts with "If I were attacking this system..."
- **Concrete exploits**: Every Medium+ finding includes reproducible attack steps and PoC
- **Chain analysis**: Low findings that enable Medium findings that enable Critical exploits = Critical chain
- **Git history mining**: Past fixes often have incomplete variants
- **OWASP LLM Top 10**: Dedicated agent covers all 10 categories (prompt injection, sensitive data disclosure, supply chain, data poisoning, improper output handling, excessive agency, system prompt leakage, embedding weaknesses, hallucinations, unbounded consumption) plus a mitigation verification pass that checks for OWASP-recommended defenses
- **Framework plugins**: After recon, framework-specific plugins (Firebase, React, Next.js, Express, FastAPI, Django) are auto-detected and their checks injected into relevant agent prompts
- **Dynamic scoping**: Recon determines which agents to skip (no frontend agent for backend-only projects, no LLM agent for projects without AI)
- **Group discipline**: Later groups depend on earlier findings; never start a group early
- **Critic validation**: Adversarial review catches false positives before the final report

### Level of Detail (LOD) Architecture

Agent reports use a three-tier Level of Detail system to minimize context consumption across independent agent context windows:

| Level | Content | ~Tokens | Where it lives |
|---|---|---|---|
| LOD-0 | `ID \| Severity \| One sentence` | 30/finding | Agent return message, orchestrator context, `{PRIOR_FINDINGS_SUMMARY}` |
| LOD-1 | ID + title + severity + CVSS + CWE + file:line + paragraph | 100/finding | `{FINDINGS_DIR}/index.md`, terminal agent prompts |
| LOD-2 | Full finding with PoC, remediation, regression test | 800/finding | `{FINDINGS_DIR}/{FINDING-ID}.md`, read on demand |

**How it flows:**
1. Analysis agents write atomic LOD-2 files to `{FINDINGS_DIR}/` and return LOD-0 summaries
2. The orchestrator builds `{FINDINGS_DIR}/index.md` with LOD-0 + LOD-1 views
3. Next-group agents receive LOD-0 in their prompt; selectively `Read` LOD-2 files for relevant findings
4. Terminal agents (exploit dev, critic, report writer) receive LOD-1 index; `Read` all LOD-2 files

This saves 80-96% of prompt tokens for downstream agents while preserving full detail on disk.

### Recon LOD Architecture

The recon report uses the same LOD architecture as findings:

| Level | Content | Where it lives |
|---|---|---|
| LOD-0 | `Step # \| Section Name \| Key facts` | `{RECON_DIR}/index.md` (top table) |
| LOD-1 | Key metric + key files + notable observations + details link | `{RECON_DIR}/index.md` (section briefs) |
| LOD-2 | Full tables with every file:line reference | `{RECON_DIR}/step-{NN}-{name}.md` |

**How it flows:**
1. 14 parallel recon agents each write one LOD-2 step file and return LOD-0+1 summaries
2. The orchestrator assembles `recon/index.md` with LOD-0 table + LOD-1 briefs
3. Downstream agents read the index, then selectively read relevant step files
4. Each downstream agent's prompt specifies which step files are most relevant

### Task-Driven Progress Tracking

All execution groups use TaskCreate for user-visible progress. Every agent gets a task with naming convention `"Group N: <description>"`. Agents claim tasks from TaskList, execute per the task description, and mark tasks completed. Users see real-time progress via the task checklist.

### Context Budget Considerations

The LOD architecture mitigates most context pressure, but be aware:
- The LLM agent prompt (`attack-surface-llm.md`) is ~340 lines before plugin injection — the largest single prompt
- Terminal agents (exploit dev, critic, report writer) still read all LOD-2 files via tool calls; for runs with 40+ findings, consider batching reads
- Plugin sections add to every injected agent's context — keep plugins concise

## Customization

### Adding New Agents

1. Create a prompt file in `prompts/` following the pattern of existing prompts
2. Use placeholders: `{RECON_INDEX_PATH}`, `{RECON_DIR}`, `{FINDING_TEMPLATE_PATH}`, `{PRIOR_FINDINGS_SUMMARY}`
3. Define a finding ID prefix (e.g., `P3-NEW-XXX`)
4. Add the agent to the appropriate execution group in `security-analyst.md`

### Adding New Plugins

1. Create a `.md` file in `plugins/` (e.g., `spring-boot.md`)
2. Add YAML frontmatter with `name` and `detect` criteria (files, dependencies, keywords)
3. Add `## {agent-name}` sections with framework-specific security checks
4. The orchestrator auto-detects plugins after recon and injects matching sections into agent prompts
5. See `plugins/README.md` for the full format spec

Available plugins: Firebase, React, Next.js, Node.js + Express, Python + FastAPI, Python + Django

### Adding New Templates

1. Create a template file in `templates/`
2. Use placeholder variables (`{RUN_DIR}`, `{ISO_TIMESTAMP}`, etc.)
3. Reference it from the orchestrator or relevant agent prompts
