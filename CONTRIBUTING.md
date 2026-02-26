# Contributing

## Contributing a Plugin

Plugins add framework-specific security checks that are automatically injected into agent prompts when the framework is detected. There are currently 16 plugins (Firebase, React, Django, Next.js, Spring Boot, etc.). To add a new one:

### 1. Create the plugin file

Add a markdown file to `plugin/skills/security-analyst/references/plugins/` using lowercase with hyphens (e.g., `my-framework.md`).

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

See [plugin/skills/security-analyst/references/plugins/README.md](plugin/skills/security-analyst/references/plugins/README.md) for the full plugin specification and existing plugins for reference.
