# Security Analyst — Architecture Reference

**See `references/constants.md` for the authoritative registry** of all directory paths, output filenames (including reports under `reports/`), template files, agent registry, stages, recon step files, and placeholder variables. **See `references/platform-tools.md` for environment-specific tool mapping (Cursor, Claude Code, Codex) — required to prevent hangs on Cursor.**

## Directory Structure

Per the skill layout guide, the skill uses `references/` for documentation and `assets/` for templates:

```
security-analyst/
├── SKILL.md                              # Skill entry point (required)
├── references/
│   ├── architecture.md                  # This file
│   ├── constants.md                     # Paths, agent registry, templates
│   ├── skills-complete-guide.md        # Compliance reference (maintainers)
│   ├── plan-lod.md                      # Internal planning
│   ├── commands/                        # Entry points and orchestrator
│   │   ├── security-analyst.md          # Main orchestrator (interactive)
│   │   ├── full.md                      # Full analysis entry point
│   │   ├── focused.md                   # Focused analysis entry point
│   │   ├── recon.md                     # Recon-only entry point
│   │   ├── threat-model.md              # Threat model entry point
│   │   ├── variant-hunt.md              # Variant hunt entry point
│   │   ├── fix-plan.md                  # Fix plan entry point
│   │   ├── sbom.md                      # SBOM entry point
│   │   ├── diff.md                      # Delta report entry point
│   │   ├── compliance.md                # Compliance mapping entry point
│   │   ├── privacy.md                   # Privacy assessment entry point
│   │   └── security-analysis.md         # Plan-level security analysis
│   ├── prompts/                         # Agent prompt templates (25 files)
│   │   ├── recon-agent.md               # Phase 0: reconnaissance
│   │   ├── attack-surface-http.md        # Phase 1: HTTP entry points
│   │   ├── attack-surface-authz.md      # Phase 1: database/authz rules
│   │   └── ... (see constants.md)       # All agent prompts
│   └── plugins/                         # Framework-specific security checks
│       ├── README.md                    # Plugin format spec and instructions
│       └── *.md                         # 16 plugins (Firebase, React, etc.)
└── assets/
    └── templates/                       # Output format templates (11 files)
        ├── recon-report.md              # Recon LOD format definitions
        ├── finding.md                   # Individual finding format
        ├── agent-common.md              # Shared LOD output + incidental findings
        ├── group-report.md              # Per-stage report
        ├── executive-report.md           # Executive report structure
        ├── fix-plan.md                  # Fix plan structure
        ├── sbom-report.md               # Software Bill of Materials
        ├── compliance-report.md         # Compliance framework mapping
        ├── delta-report.md              # Delta report between runs
        ├── privacy-report.md            # Privacy & data protection assessment
        └── checkpoint.md               # Stage tracking + resume checkpoint
```

## Agent Design

All agents are `generalPurpose` subagents (need Bash, Read, Grep, Glob for code analysis and git history mining). Each agent:

1. Reads the recon index for LOD-0+1 overview, selectively reads LOD-2 step files for relevant sections
2. Reads prior stage findings (if applicable)
3. Performs its specialized analysis
4. Reports findings using the standard finding template
5. Reports incidental findings (IX-prefixed) for issues outside its focus area

### Key Principles

- **Offensive framing**: Every analysis starts with "If I were attacking this system..."
- **Concrete exploits**: Every Medium+ finding includes reproducible attack steps and PoC
- **Chain analysis**: Low findings that enable Medium findings that enable Critical exploits = Critical chain
- **Git history mining**: Past fixes often have incomplete variants
- **OWASP LLM Top 10**: Dedicated agent covers all 10 categories plus a mitigation verification pass
- **Framework plugins**: After recon, framework-specific plugins are auto-detected and injected into relevant agent prompts. 16 plugins available
- **Dynamic scoping**: Recon determines which agents to skip (no frontend agent for backend-only projects, no LLM agent for projects without AI)
- **Stage discipline**: Later stages depend on earlier findings; never start a stage early
- **Trust model integration**: Recon Step 4 extracts the project's documented trust model (from SECURITY.md) and feeds it to all downstream agents, reducing false positives from findings that contradict documented trust assumptions
- **External audit integration**: Recon detects openclaw and other external audit tool results; downstream agents cross-reference to avoid duplicating confirmed findings
- **Critic validation**: Adversarial review catches false positives before the final report

## Level of Detail (LOD) Architecture

Agent reports use a three-tier Level of Detail system to minimize context consumption across independent agent context windows:

| Level | Content | ~Tokens | Where it lives |
|---|---|---|---|
| LOD-0 | `[ID](findings/{ID}.md) \| Severity \| One sentence` | 30/finding | Agent return message, orchestrator context, `{PRIOR_FINDINGS_SUMMARY}` |
| LOD-1 | [ID](findings/{ID}.md) + title + severity + CVSS + CWE + file:line + paragraph | 100/finding | `{FINDINGS_DIR}/index.md`, terminal agent prompts |
| LOD-2 | Full finding with PoC, remediation, regression test | 800/finding | `{FINDINGS_DIR}/{FINDING-ID}.md`, read on demand |

**How it flows:**
1. Analysis agents write atomic LOD-2 files to `{FINDINGS_DIR}/` and return LOD-0 summaries
2. The orchestrator builds `{FINDINGS_DIR}/index.md` with LOD-0 + LOD-1 views
3. Next-stage agents receive LOD-0 in their prompt; selectively `Read` LOD-2 files for relevant findings
4. Terminal agents (exploit dev, critic, report writer) receive LOD-1 index; `Read` all LOD-2 files

This saves 80-96% of prompt tokens for downstream agents while preserving full detail on disk.

### Recon LOD Architecture

The recon report uses the same LOD architecture as findings:

| Level | Content | Where it lives |
|---|---|---|
| LOD-0 | `[Step #](recon/step-{NN}-{name}.md) \| Section Name \| Key facts` | `{RECON_DIR}/index.md` (top table) |
| LOD-1 | Key metric + key files + notable observations + [details link](recon/step-{NN}-{name}.md) | `{RECON_DIR}/index.md` (section briefs) |
| LOD-2 | Full tables with every file:line reference | `{RECON_DIR}/step-{NN}-{name}.md` |

**How it flows:**
1. 14 parallel recon agents each write one LOD-2 step file and return LOD-0+1 summaries
2. The orchestrator assembles `recon/index.md` with LOD-0 table + LOD-1 briefs
3. Downstream agents read the index, then selectively read relevant step files
4. Each downstream agent's prompt specifies which step files are most relevant

### Task-Driven Progress Tracking

All platforms use agents-only orchestration (no Teams). Agents are spawned via the Task tool with self-contained prompts and return results via the Task return value. On **Claude Code / Codex**, TaskCreate/TaskUpdate provide user-visible progress tracking — the orchestrator creates tasks before spawning and marks them completed after each agent returns. On **Cursor**, task tracking is not available (no TaskCreate/TaskList). See `references/platform-tools.md`.

### Orchestrator Architecture

The top-level orchestrator is a **thin dispatcher** — it creates the run directory, analyzes the recon output, and dispatches each stage as a stage-orchestrator Task. Each stage-orchestrator is a `generalPurpose` subagent that reads prompt templates from disk, spawns analysis agents (sub-subagents), consolidates findings, and returns a compact summary. This gives each stage a fresh context window and keeps the top-level orchestrator under ~20K tokens. A checkpoint file on disk enables resuming interrupted runs.

## Large Codebase Handling

The LOD architecture mitigates context pressure *between* agents (80-96% token savings), but individual agents can still exhaust their context window when a codebase has many targets (endpoints, integrations, rule files, findings). Three mechanisms address this:

### 1. Orchestrator-Level Work Partitioning (primary)

After recon, the orchestrator counts targets per agent from the recon step files (Step 3.2 in the orchestrator). When a count exceeds the agent's threshold (see `constants.md` → "Chunking Thresholds"), the work is split into chunks and multiple instances of the same agent are spawned in parallel.

Each chunk gets the same prompt template with an appended `TARGET SUBSET` section containing only its assigned target rows. Finding ID offsets prevent collisions: batch 1 starts at `{PREFIX}-001`, batch 2 at `{PREFIX}-100`, etc.

Agents that support chunking (have enumerable targets):
- `surface-http` (endpoints from step-03)
- `surface-authz` (rule files from step-06)
- `surface-integrations` (integrations from step-07)
- `surface-frontend` (rendering points from step-12)
- `history-*` (commits from git log)
- `exploit-dev` (findings from index)
- `finding-critic` (findings from index)

Agents that do NOT chunk (holistic analysis required):
- `surface-llm` (cross-cutting OWASP LLM categories)
- `logic-*` (cross-cutting business logic)
- `report-writer` (global dedup + root cause)
- Recon agents (grep-based enumeration, not deep analysis per item)

### 2. PARTIAL Return Protocol (safety valve)

Any agent can return with `PARTIAL` status when approaching its context budget. The agent writes all findings discovered so far to disk and returns a structured message listing remaining targets. The orchestrator then spawns a continuation agent for the remaining work.

This catches cases where pre-chunking underestimates context usage (e.g., individual source files are unusually large). See `agent-common.md` for the protocol format and heuristics.

### 3. Targeted Reading

All agents are instructed to read ~60 lines centered on the `file:line` reference from recon step files instead of reading entire source files. This prevents large files (1000+ lines) from consuming 10-20x more context than the ~100-line function body the agent actually needs. Git history agents use `git show --stat` to assess diff size before reading, and read per-file diffs for large commits.

### 4. Stage-Based Dispatching (orchestrator context management)

The orchestrator coordinates 28+ agents across 8 stages. Without mitigation, its own context accumulates ~100K+ tokens from reading prompt templates, Task call prompts/returns, and inter-stage reasoning.

**Solution:** Each stage runs inside its own **stage-orchestrator Task** — a `generalPurpose` subagent with a fresh context window. The stage-orchestrator reads templates from disk, spawns its agents (sub-subagents), consolidates findings, writes the stage report, and returns a compact summary (~200 tokens).

**Top-level orchestrator context budget:**
- Orchestrator prompt: ~3K tokens
- 8 stage Task calls × ~300 tokens each (compact prompt-in + summary-out): ~2.5K tokens
- Step 3 recon analysis (reads recon index + step files): ~5K tokens
- Own reasoning: ~5K tokens
- **Total: ~15-20K tokens** (vs ~100K+ without dispatching)

**Stage-orchestrator context budget (per stage):**
- Stage prompt: ~1K tokens
- Template reads: ~3-5K tokens
- Agent Task calls (prompt + LOD-0 return): ~2K per agent
- Consolidation reasoning: ~2K tokens
- **Total: ~15-30K tokens** depending on agent count

### 5. Checkpoint & Resume

A checkpoint file (`{RUN_DIR}/checkpoint.md`) tracks stage completion, configuration (skipped agents, plugins, partitions), and paths. Updated after each stage. If the orchestrator is interrupted, a new session reads the checkpoint and resumes from the first incomplete stage without re-running completed work.

### 6. File Exclusions

All agents inherit file exclusion rules from `agent-common.md`. The primary mechanism is **respecting `.gitignore`**: agents use `git ls-files` to enumerate tracked files and the Grep tool (which uses ripgrep, which respects `.gitignore` by default) for content searches. This automatically excludes `node_modules/`, `vendor/`, `dist/`, `build/`, `.next/`, `__pycache__/`, and any other project-specific ignores. Agents also restrict searches to source directories or file types rather than searching the project root broadly. The `dependency-audit` agent is the only exception — it intentionally reads package metadata from `node_modules/`.

### Context Budget Notes

- The LLM agent prompt (`attack-surface-llm.md`) is ~340 lines before plugin injection — the largest single prompt
- Plugin sections add to every injected agent's context — keep plugins concise
- For small/medium codebases (under chunking thresholds), the system runs exactly as before with no overhead

## Trust Model & External Audit Integration

### Trust Model Flow

The project's `SECURITY.md` (or equivalent security policy / threat model document) is consumed at the earliest stage of analysis and propagated through the entire pipeline:

```
Recon Step 4 (Wave A)
  ├─ Glob for SECURITY.md, SECURITY*, threat-model*
  ├─ Extract: trust levels, accepted risks, threat assumptions, security invariants
  ├─ Write to step-04-boundaries.md → "Documented Trust Model" section
  └─ Annotate code-inferred boundaries: [DOCUMENTED] / [MISMATCH] / [UNDOCUMENTED]
       │
       ▼
Orchestrator Step 3 (Analysis)
  ├─ Read trust model from step-04-boundaries.md
  ├─ Log: trust_model status, source document, counts
  └─ Record in checkpoint.md for resume
       │
       ▼
All Downstream Agents (via agent-common.md)
  ├─ Read step-04-boundaries.md "Documented Trust Model" section
  ├─ Filter: findings within documented trust levels → require stronger evidence or downgrade
  ├─ Annotate: [ACCEPTED-RISK] for documented risk acceptances
  └─ Escalate: [INVARIANT-VIOLATION] for findings that break documented security invariants
       │
       ▼
Finding Critic (Validation Stage)
  ├─ Cross-check all findings against trust model
  ├─ Downgrade findings targeting explicitly-trusted components (if trust assumption is reasonable)
  └─ Upgrade findings that violate documented invariants
```

This early integration reduces false positive rates by letting agents filter against the project's own security contract rather than discovering and re-assessing trust assumptions independently.

### External Audit Tool Integration (OpenClaw)

External audit tool results (openclaw, or similar) are detected during recon and used to avoid duplicate work:

```
Recon Step 4 (Wave A)
  ├─ Glob for openclaw*, .openclaw*, openclaw-report*, openclaw-results*
  ├─ Extract: confirmed finding IDs, severities, affected files
  └─ Write to step-04-boundaries.md → "External Audit Findings" section

Recon Step 10 (Wave A)
  ├─ Glob for openclaw config/results (broader pattern set)
  └─ Write to step-10-security-work.md → "External Audit Tool Results" section
       │
       ▼
All Downstream Agents (via agent-common.md)
  ├─ Read external audit findings from step-04/step-10
  ├─ Cross-reference: annotate matching findings with [CORROBORATES: {tool} {id}]
  ├─ Avoid duplicates: same root cause + same file → reference external finding instead
  └─ Deepen: if additional attack surface beyond external tool's scope → new finding with reference

Finding Critic
  ├─ Fast-track findings corroborated by external tools
  └─ Note coverage gaps where pipeline found issues the external tool missed
```

## Adding New Agents

1. Create a prompt file in `references/prompts/` following the pattern of existing prompts
2. Use placeholders: `{RECON_INDEX_PATH}`, `{RECON_DIR}`, `{FINDING_TEMPLATE_PATH}`, `{PRIOR_FINDINGS_SUMMARY}`
3. Define a finding ID prefix (e.g., `NEW-XXX`)
4. Add the agent to the appropriate stage in `references/commands/security-analyst.md`

## Adding New Plugins

To create a new plugin interactively, use `/security-analyst:create-plugin [framework]`. This validates against existing plugins, checks base agent coverage, researches framework security characteristics, and generates the plugin file. To contribute it back, use `/security-analyst:contribute-plugin [plugin-name]`.

For manual creation:

1. Create a `.md` file in `references/plugins/` (e.g., `spring-boot.md`)
2. Add YAML frontmatter with `name` and `detect` criteria (files, dependencies, keywords)
3. Add `## {agent-name}` sections with framework-specific security checks
4. The orchestrator auto-detects plugins after recon and injects matching sections into agent prompts
5. See `references/plugins/README.md` for the full format spec
