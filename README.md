# Security Analyst

Offensive security analysis skill for Claude Code / Cursor that coordinates specialized agents through a multi-group penetration testing pipeline. Finds real vulnerabilities with concrete exploits, not checkbox compliance.

## Overview

- **23+ agents** across 9 phases in 6 execution groups
- **Framework plugins** for Firebase, React, Next.js, Express, FastAPI, Django
- **OWASP LLM Top 10** dedicated coverage
- **LOD architecture** minimizes context consumption across agent windows
- Produces CVSS 3.1 scores, CWE IDs, MITRE ATT&CK mappings, PoC exploits, and fix code

## Commands

| Command | Purpose |
|---------|---------|
| `/security-analyst` | Interactive — choose scope and output mode |
| `/security-analyst:full` | All 9 phases, all attack surfaces |
| `/security-analyst:focused [component]` | Target a specific area |
| `/security-analyst:recon` | Phase 0 only — codebase security map |
| `/security-analyst:threat-model` | Recon + STRIDE analysis + attack trees |
| `/security-analyst:variant-hunt [vuln]` | Find unpatched variants of a known vulnerability |
| `/security-analyst:fix-plan [run-dir]` | Generate implementation plan from existing report |

## Installation

Copy or symlink this directory into your skills location:

```bash
# Claude Code
cp -r . ~/.claude/skills/security-analyst

# Cursor
cp -r . ~/.cursor/skills/security-analyst
```

## Structure

```
├── SKILL.md                    # Skill manifest and documentation
├── security-analyst.md         # Main orchestrator (interactive)
├── full.md                     # Full analysis entry point
├── focused.md                  # Focused analysis entry point
├── recon.md                    # Recon-only entry point
├── threat-model.md             # Threat model entry point
├── variant-hunt.md             # Variant hunt entry point
├── fix-plan.md                 # Fix plan entry point
├── prompts/                    # 20 agent prompt templates
├── plugins/                    # Framework-specific security checks
└── templates/                  # Output format templates
```

See [SKILL.md](SKILL.md) for full documentation.

## License

MIT
