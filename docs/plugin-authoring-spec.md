# Security Analyst — Plugin Authoring Spec

> **Audience**: Coding agents and developers creating new framework/infrastructure plugins for the security-analyst skill.
>
> **What is a plugin?** A single markdown file that injects framework-specific security checks into the security-analyst agent pipeline. Plugins are auto-detected after reconnaissance and their sections are appended to the relevant agent prompts via the `{PLUGIN_CHECKS}` placeholder.

---

## Table of Contents

1. [Plugin Architecture Overview](#1-plugin-architecture-overview)
2. [File Location and Naming](#2-file-location-and-naming)
3. [File Structure](#3-file-structure)
4. [Frontmatter Schema](#4-frontmatter-schema)
5. [Agent Sections](#5-agent-sections)
6. [Writing Effective Security Checks](#6-writing-effective-security-checks)
7. [Detection Criteria Design](#7-detection-criteria-design)
8. [Plugin Sizing and Scope](#8-plugin-sizing-and-scope)
9. [Complete Examples](#9-complete-examples)
10. [Validation Checklist](#10-validation-checklist)
11. [Reference: Agent Section Catalog](#11-reference-agent-section-catalog)
12. [Reference: Recon Step Files](#12-reference-recon-step-files)

---

## 1. Plugin Architecture Overview

### How plugins flow through the pipeline

```
Recon Stage (14 parallel agents map the codebase)
      │
      ▼
Orchestrator Step 3.5: Plugin Detection
  ├─ Glob {PLUGINS_DIR}/*.md (exclude README.md)
  ├─ For each plugin, read YAML frontmatter
  ├─ Match detect.files against project root (Glob check)
  ├─ Match detect.dependencies against recon step-13-dependencies.md
  ├─ Match detect.keywords against recon step-01-metadata.md
  ├─ A plugin matches if ANY single criterion hits
  ├─ Parse ## {agent-name} sections from matched plugins
  └─ Log: "Matched plugins: Next.js, React" (or "No plugins matched")
      │
      ▼
Agent Spawning
  ├─ For each agent, collect all matching plugin sections keyed by agent name
  ├─ Concatenate sections from multiple plugins (complementary, not exclusive)
  └─ Inject into the agent prompt at the {PLUGIN_CHECKS} placeholder
      │
      ▼
Agent Execution
  └─ Agent treats plugin checks as additional analysis tasks
      alongside the base prompt's generic checks
```

### Key constraints

- **Plugins are passive markdown** — no executable code, no scripts, no imports.
- **Plugins extend, not replace** — they add framework-specific checks on top of generic checks already in each agent's base prompt. Never repeat checks that the base agent prompt already covers.
- **Plugins are injected per-agent** — each agent only receives the `## {agent-name}` section relevant to it, not the entire plugin file.
- **Multiple plugins can inject into the same agent** — when this happens, sections are concatenated. Write checks that are complementary, not conflicting.
- **Plugins add to every injected agent's context budget** — keep sections concise.

---

## 2. File Location and Naming

```
{SKILL_ROOT}/references/plugins/{plugin-name}.md
```

| Rule | Example |
|------|---------|
| Kebab-case filename | `spring-boot.md`, `ruby-rails.md`, `aws-gcp-azure.md` |
| `.md` extension required | Not `.yaml`, not `.json` |
| One file per logical framework/platform | `firebase.md` covers Firestore + Cloud Functions + Auth |
| Related frameworks can share a file | `vuejs-nuxtjs.md` covers both Vue.js and Nuxt.js |
| `README.md` is excluded from detection | It is reserved for the plugin directory documentation |

### When to combine vs. split

**Combine** when frameworks are tightly coupled and share security concerns (Vue.js + Nuxt.js, AWS + GCP + Azure cloud patterns).

**Split** when frameworks are independent and have distinct security surfaces (React and Django should be separate plugins even if a project uses both — each will match independently).

---

## 3. File Structure

Every plugin file has exactly two parts: YAML frontmatter and agent sections.

```markdown
---
name: "Human-readable plugin name"
detect:
  files: ["filename1", "filename2"]
  dependencies: ["package-a", "package-b"]
  keywords: ["Framework", "Platform"]
---

## agent-section-name-1

**Category Title:**
1. **Check name:**
   - Specific instruction for the agent
   - What to grep/glob for
   - What the vulnerability looks like
   - How an attacker exploits it

## agent-section-name-2

**Category Title:**
1. ...
```

There is no content between the frontmatter closing `---` and the first `## agent-name` heading. No introductory text, no table of contents, no metadata outside the frontmatter.

---

## 4. Frontmatter Schema

### Required fields

```yaml
---
name: "Human-readable plugin name"
detect:
  files: ["list", "of", "filenames"]
  dependencies: ["package-a", "package-b"]
  keywords: ["Framework", "Platform"]
---
```

### Field reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Human-readable name shown in orchestrator logs (e.g., "Spring Boot (Java)") |
| `detect.files` | string[] | Yes* | Filenames to Glob for in the project root. Match if ANY exists. |
| `detect.dependencies` | string[] | Yes* | Package names to search for in `step-13-dependencies.md`. Match if ANY appears. |
| `detect.keywords` | string[] | Yes* | Strings to search for in `step-01-metadata.md` (Framework, Infrastructure, Runtime fields). Match if ANY appears. |

*At least one of the three `detect` lists must be non-empty. Empty lists (`[]`) are allowed for fields that don't apply.

### Detection semantics

- **Within a list**: OR logic — any single match triggers.
- **Across lists**: OR logic — a hit in `files`, `dependencies`, OR `keywords` is sufficient.
- A plugin matches if **any single criterion hits across any of the three lists**.

### Writing detection criteria

**Be specific enough to avoid false positives, broad enough to catch real usage.**

| Criterion Type | Good Examples | Bad Examples | Why |
|----------------|--------------|-------------|-----|
| `files` | `["firebase.json", ".firebaserc"]` | `["config.json"]` | `config.json` is too generic — matches non-Firebase projects |
| `files` | `["next.config.js", "next.config.mjs", "next.config.ts"]` | `["next.config.js"]` | Missing `.mjs` and `.ts` variants means missed detection |
| `dependencies` | `["express", "koa", "fastify"]` | `["http"]` | `http` is a Node.js built-in, not a dependency package |
| `dependencies` | `["gin-gonic/gin", "labstack/echo"]` | `["gin"]` | Go dependencies use full module paths |
| `keywords` | `["Firebase", "Cloud Functions"]` | `["Functions"]` | Too generic — matches any project mentioning "Functions" |

### Common file detection patterns by ecosystem

| Ecosystem | Typical Detection Files |
|-----------|------------------------|
| Node.js frameworks | `package.json` (too generic — use dependencies instead), framework config files |
| Python frameworks | `manage.py` (Django), `pyproject.toml` (generic — use dependencies) |
| Go frameworks | `go.mod` (generic — use dependencies for specific frameworks) |
| Java/Kotlin | `pom.xml`, `build.gradle`, `build.gradle.kts` (generic — use dependencies) |
| Ruby | `Gemfile` (generic — use dependencies) |
| Infrastructure | `Dockerfile`, `docker-compose.yml`, `terraform/*.tf`, `k8s/*.yaml` |
| Cloud | `firebase.json`, `serverless.yml`, `app.yaml` (GCP), `sam-template.yaml` (AWS) |

---

## 5. Agent Sections

### Format

Each section heading MUST exactly match an agent prompt filename (without `.md`):

```markdown
## attack-surface-http

Content for the HTTP entry points agent...

## config-infrastructure

Content for the infrastructure config agent...
```

### Available agent section names

| Section Name | Agent Focus | When to Add Checks |
|-------------|-------------|-------------------|
| `recon-agent` | Reconnaissance / discovery | Additional file patterns, config locations, framework-specific project structure |
| `attack-surface-http` | HTTP entry points | Framework-specific endpoint patterns, middleware quirks, routing security |
| `attack-surface-authz` | Authorization rules | Framework-specific access control, permission models, rule evaluation |
| `attack-surface-frontend` | Frontend & client-side | Framework-specific XSS patterns, client-side security, state management |
| `attack-surface-llm` | LLM/AI security | Framework-specific AI integration patterns |
| `attack-surface-integrations` | External integrations | Framework-specific API client patterns |
| `git-history-injections` | Injection variant hunting | Framework-specific injection patterns to search git history for |
| `git-history-auth-bypass` | Auth bypass variant hunting | Framework-specific auth bypass patterns in git history |
| `git-history-ssrf` | SSRF variant hunting | Framework-specific SSRF patterns in git history |
| `git-history-data-exposure` | Data exposure variant hunting | Framework-specific data leak patterns in git history |
| `logic-race-conditions` | Race conditions & TOCTOU | Framework-specific concurrency patterns, transaction semantics |
| `logic-authz-escalation` | Privilege escalation | Framework-specific privilege escalation vectors |
| `logic-pipeline-exploit` | Pipeline exploitation | Framework-specific pipeline logic flaws |
| `logic-dos` | Denial of service | Framework-specific resource limits, amplification vectors |
| `data-flow-tracer` | Data flow tracing | Framework-specific data flow patterns |
| `dependency-audit` | Dependency audit | Framework-specific dependency concerns, known vulnerable packages |
| `config-infrastructure` | Infrastructure config | Framework-specific configuration security, secrets management |

**Only include sections where you have framework-specific checks to add.** Omitted sections are silently skipped — there is no penalty for not covering every agent.

### Section content structure

Each section should follow this pattern:

```markdown
## agent-section-name

**Category Title (describes the class of checks):**
1. **Specific Check Name:**
   - What to search for (Grep patterns, Glob patterns, file locations)
   - What the vulnerability is (root cause, not just symptoms)
   - How an attacker exploits it (offensive framing)
   - What the secure alternative looks like
2. **Another Check:**
   - ...
```

---

## 6. Writing Effective Security Checks

### Principles

1. **Offensive framing** — Write from the attacker's perspective: "Can an attacker...", "What happens if...", "Is there a way to...". This is how the base agent prompts are written, and plugin checks should match.

2. **Concrete search instructions** — Tell the agent exactly what to look for. Don't say "check for insecure configuration." Say "Grep for `cors({ origin: '*' })` in the Express middleware setup."

3. **Specific code patterns** — Reference actual function names, config keys, file locations, and API calls specific to the framework. This is the entire value of a plugin — generic checks already exist in the base prompts.

4. **Exploit scenarios** — Describe what an attacker can do if the check fails. "If CSRF middleware is missing, an attacker can craft a page that submits state-changing requests using the victim's session cookies."

5. **Framework defaults matter** — Many frameworks have insecure defaults. Call these out explicitly: "Express body parser default limit is 100kb", "Go HTTP server accepts unlimited request bodies by default", "Next.js API routes have no built-in auth middleware."

6. **Don't repeat generic checks** — The base agent prompts already cover SQL injection, XSS, CSRF, SSRF, auth bypass, etc. in generic terms. Plugin checks should only cover what is **unique** to the framework — specific API surfaces, framework-specific gotchas, configuration options that don't exist in other frameworks.

### Good vs. bad check examples

**Good — framework-specific, actionable, offensive:**

```markdown
1. **No Built-in CSRF Protection:**
   - Go web frameworks do not include CSRF protection by default
   - Check if a CSRF middleware is installed (`gorilla/csrf`, `nosurf`, framework-specific)
   - If there is no CSRF middleware: all state-changing endpoints accepting cookies are vulnerable
```

**Bad — generic, vague, defensive:**

```markdown
1. **CSRF Protection:**
   - Make sure CSRF protection is enabled
   - Check that tokens are validated
```

**Good — specific search instructions:**

```markdown
2. **SQL Injection via String Formatting:**
   - Grep for `fmt.Sprintf` used to construct SQL queries — this is the #1 Go SQL injection pattern
   - Check for raw query construction: `db.Raw(`, `db.Exec(` with string concatenation
   - Are parameterized queries used consistently? (`db.Where("id = ?", id)` vs `db.Where("id = " + id)`)
```

**Bad — no search guidance:**

```markdown
2. **SQL Injection:**
   - Check for SQL injection vulnerabilities in database queries
```

### Severity guidance

While plugins don't assign CVSS scores directly (that's the agent's job), you should indicate relative severity when the framework makes it clear-cut:

```markdown
- **Critical**: Is AES used in ECB mode? (ECB preserves plaintext patterns — always a finding)
- **High**: Is AES-CBC used without a separate HMAC? (vulnerable to padding oracle attacks)
```

Use severity indicators sparingly — only when the framework-specific context makes the severity unambiguous.

---

## 7. Detection Criteria Design

### Strategy by framework type

**Web frameworks** (Express, Django, Spring Boot, Gin):
- `files`: Framework-specific config files (not `package.json` — too generic)
- `dependencies`: The framework's core package name(s)
- `keywords`: Framework name as it appears in project metadata

**Frontend frameworks** (React, Vue, Angular, SolidJS):
- `files`: Framework-specific entry points (`src/App.tsx`, `next.config.js`)
- `dependencies`: Core framework packages and common companion packages
- `keywords`: Framework name

**Infrastructure** (Docker, Kubernetes, Cloud providers):
- `files`: Infrastructure config files (`Dockerfile`, `k8s/`, `terraform/`)
- `dependencies`: SDK packages (`@aws-sdk/*`, `@google-cloud/*`)
- `keywords`: Platform names

**Cross-cutting concerns** (GraphQL, secrets/encryption, OWASP):
- `files`: Concern-specific config files (`schema.graphql`, `vault.hcl`)
- `dependencies`: All known libraries for that concern across ecosystems
- `keywords`: Concern-specific terminology

### Testing your detection criteria

Before finalizing your detection criteria, verify:

1. **True positive**: Would a project using your target framework definitely have at least one of your detection criteria?
2. **False positive**: Could a project NOT using your target framework accidentally match? (e.g., `["config.json"]` matches almost anything)
3. **Variant coverage**: Have you covered all common filename variants? (`.js`, `.mjs`, `.ts`, `.cjs` for JavaScript configs)

---

## 8. Plugin Sizing and Scope

### Context budget awareness

Plugin sections are injected into agent context windows. Large plugins consume tokens that could otherwise be used for analysis. Guidelines:

| Plugin Scope | Target Size | Example |
|-------------|-------------|---------|
| Single framework, few agents | 30-80 lines total | `react.md` — 1 section, focused |
| Single framework, many agents | 80-180 lines total | `go-gin.md`, `nextjs.md` — 3-5 sections |
| Comprehensive platform | 150-300 lines total | `firebase.md` — 7 sections, deep coverage |
| Cross-cutting concern | 100-200 lines total | `secrets-encryption-storage.md` — 7 sections |

**Individual section sizes**: Aim for 10-40 lines per agent section. If a section exceeds 50 lines, consider whether some checks are generic enough to belong in the base agent prompt instead.

### When NOT to create a plugin

- The framework has no security-relevant quirks beyond what generic checks cover.
- The framework is used by fewer than ~1% of projects (consider whether the context cost is justified).
- The checks would duplicate what another matched plugin already covers (e.g., don't create a separate "Express middleware" plugin if `nodejs-express.md` already covers it).

---

## 9. Complete Examples

### Example A: Minimal plugin (1 section)

A simple frontend framework plugin that only extends the frontend agent:

```markdown
---
name: Svelte
detect:
  files: ["svelte.config.js", "svelte.config.ts"]
  dependencies: ["svelte", "@sveltejs/kit", "@sveltejs/adapter-auto"]
  keywords: ["Svelte", "SvelteKit"]
---

## attack-surface-frontend

**Svelte HTML Injection:**
1. **`{@html}` Tag Audit:**
   - Grep for every instance of `{@html` in `.svelte` files
   - For EACH instance: trace the expression to its source — is it user-controlled or from external data?
   - Is the HTML sanitized before rendering? (e.g., DOMPurify)
   - `{@html}` bypasses Svelte's auto-escaping — this is the primary XSS vector in Svelte apps

2. **Action Directive Injection:**
   - Grep for `use:action` patterns where the action receives user-controlled parameters
   - Can an attacker influence DOM manipulation performed by custom actions?

3. **Store-Based Data Leaks:**
   - Are there writable stores (`writable()`) containing sensitive data (tokens, PII)?
   - Can a component subscribe to stores it shouldn't access?
   - Are store values serialized during SSR and included in the initial HTML payload?

4. **SvelteKit Form Actions:**
   - Grep for `export const actions` in `+page.server.ts` files
   - Are form action inputs validated on the server?
   - Can an attacker call a form action with unexpected data types or extra fields?
   - Is CSRF protection enabled? (SvelteKit has built-in CSRF via origin checking — is it disabled?)
```

### Example B: Multi-section plugin (3-5 sections)

A backend framework with HTTP, config, auth, and dependency checks:

```markdown
---
name: Flask (Python)
detect:
  files: []
  dependencies: ["flask", "Flask", "flask-restful", "flask-restx", "flask-login"]
  keywords: ["Flask"]
---

## attack-surface-http

**Flask Routing Vulnerabilities:**
1. **Debug Mode in Production:**
   - Grep for `app.run(debug=True)`, `FLASK_DEBUG=1`, `DEBUG = True` in config
   - Debug mode enables the Werkzeug interactive debugger — provides arbitrary code execution via the debugger PIN
   - The debugger PIN can be computed from server metadata (`/etc/machine-id`, MAC address, module paths)
2. **Blueprint Auth Gaps:**
   - Flask blueprints do not inherit middleware from the app — auth decorators must be applied per-blueprint or per-route
   - Are there blueprints registered without `@login_required` or equivalent on routes that need auth?
3. **`send_file` / `send_from_directory` Path Traversal:**
   - Grep for `send_file`, `send_from_directory` with user-controlled paths
   - `send_from_directory` is safe if the directory argument is hardcoded, but `send_file` with a user-controlled path is vulnerable
4. **Jinja2 SSTI:**
   - Grep for `render_template_string` with user-controlled input — this is a direct SSTI vector
   - `render_template` with file-based templates is safe unless template files themselves are user-controlled

## attack-surface-authz

**Flask Authorization Patterns:**
1. **Decorator Ordering:**
   - Flask decorators execute bottom-to-top — `@login_required` must be the innermost (last) decorator before the function
   - Grep for routes with multiple decorators and verify ordering
2. **Session-Based Auth:**
   - Is `app.secret_key` hardcoded? (Grep for `secret_key`, `SECRET_KEY`)
   - Is the session cookie `httponly` and `secure`? (check `SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SECURE`)
   - Is `SESSION_COOKIE_SAMESITE` set to `"Lax"` or `"Strict"`?

## config-infrastructure

**Flask Configuration Security:**
1. **Secret Key Management:**
   - Is `SECRET_KEY` loaded from environment or hardcoded?
   - Is the default/development secret key used in production?
2. **CORS Configuration:**
   - If `flask-cors` is used: is `origins` set to `"*"`? (bad for authenticated endpoints)
   - Is CORS applied globally or per-route?
3. **Error Handlers:**
   - Are custom error handlers registered for 404 and 500? (`@app.errorhandler`)
   - Do error responses expose tracebacks in production? (Werkzeug's default error pages include full tracebacks unless debug is off)

## dependency-audit

**Flask-Specific Dependency Concerns:**
1. Check Flask version — older versions have known vulnerabilities (CVE-2023-30861: session cookie parsing)
2. Is `Werkzeug` up to date? (Flask's underlying WSGI library — frequent CVEs)
3. Is `Jinja2` up to date? (template engine — sandbox escape CVEs in older versions)
4. Are `flask-login`, `flask-jwt-extended`, or `flask-security` versions current?
```

### Example C: Infrastructure/cross-cutting plugin

```markdown
---
name: Terraform / OpenTofu
detect:
  files: ["main.tf", "terraform.tfvars", "terraform.tfstate", ".terraform.lock.hcl"]
  dependencies: []
  keywords: ["Terraform", "OpenTofu", "Infrastructure as Code"]
---

## recon-agent

**Terraform Discovery:**
1. Glob for `**/*.tf`, `**/*.tfvars`, `**/.terraform.lock.hcl`
2. Identify cloud providers from `required_providers` blocks
3. Check for remote state configuration (`backend "s3"`, `backend "gcs"`, `backend "azurerm"`)
4. Identify modules referenced from external registries (`source = "registry.terraform.io/..."`)

## config-infrastructure

**Terraform State Security:**
1. **State File Exposure:**
   - Is `terraform.tfstate` committed to the repository? (contains all resource attributes including secrets in plaintext)
   - Check `.gitignore` for `*.tfstate`, `*.tfstate.backup`
   - Is remote state storage encrypted at rest? (S3 bucket encryption, GCS default encryption)
   - Is remote state access restricted by IAM? (who can `terraform state pull`?)
2. **Provider Credentials:**
   - Are provider credentials hardcoded in `.tf` files? (Grep for `access_key`, `secret_key`, `credentials`, `token`)
   - Are credentials passed via environment variables or a credentials file?
3. **Module Supply Chain:**
   - Are external modules pinned to a specific version? (`source = "..." version = "x.y.z"`)
   - Are module sources from trusted registries? (check for arbitrary Git URLs)
   - Is `.terraform.lock.hcl` committed? (ensures reproducible provider versions)
4. **Overly Permissive IAM:**
   - Grep for `"*"` in `actions` or `resources` in IAM policy documents — wildcard permissions
   - Are there IAM roles with `AdministratorAccess` or equivalent broad policies?
   - Are service accounts using least-privilege?

## dependency-audit

**Terraform Provider Audit:**
1. Read `.terraform.lock.hcl` for provider versions and hashes
2. Are providers pinned with `required_providers` version constraints?
3. Are there outdated providers with known security issues?
4. Check for unsigned or community providers (vs. official HashiCorp providers)
```

---

## 10. Validation Checklist

Use this checklist to verify a plugin before considering it complete.

### Structure
- [ ] File is in `{SKILL_ROOT}/references/plugins/` with a kebab-case `.md` filename
- [ ] File starts with `---` YAML frontmatter delimiters
- [ ] Frontmatter has `name` (string) and `detect` (object with `files`, `dependencies`, `keywords` arrays)
- [ ] At least one detection list is non-empty
- [ ] No content between closing `---` and first `## ` heading
- [ ] Every `## ` heading exactly matches an agent prompt filename (without `.md`)

### Detection
- [ ] `detect.files`: filenames are specific to the framework (not generic like `config.json`)
- [ ] `detect.files`: all common filename variants are covered (`.js`, `.mjs`, `.ts`, `.cjs`)
- [ ] `detect.dependencies`: package names match the ecosystem's naming convention (full Go module paths, npm scope prefixes)
- [ ] `detect.keywords`: terms are specific enough to avoid false positives
- [ ] Detection criteria would match a real project using the framework
- [ ] Detection criteria would NOT match a project that doesn't use the framework

### Content quality
- [ ] Checks use offensive framing ("Can an attacker...", "What happens if...")
- [ ] Checks include specific code patterns to search for (function names, config keys, Grep patterns)
- [ ] Checks explain the exploit scenario, not just the vulnerability class
- [ ] Checks are framework-specific — generic security advice is NOT included
- [ ] No duplication of checks already in the base agent prompts
- [ ] Section sizes are reasonable (10-40 lines each, max ~50)
- [ ] Total plugin size is within guidelines (30-300 lines depending on scope)

### Overlap
- [ ] Checks don't conflict with existing plugins that may also match
- [ ] If the framework is related to an existing plugin (e.g., Nuxt.js and Vue.js), checks are complementary

---

## 11. Reference: Agent Section Catalog

Each agent has a specific focus area. Use this catalog to decide which sections your plugin needs.

### Recon stage

| Agent | Section Name | What It Discovers | Plugin Should Add |
|-------|-------------|-------------------|-------------------|
| Reconnaissance | `recon-agent` | Project structure, file inventory, architecture | Framework-specific file patterns, config file locations, project structure conventions |

### Surface stage — attack surface analysis

| Agent | Section Name | What It Analyzes | Plugin Should Add |
|-------|-------------|-----------------|-------------------|
| HTTP Entry Points | `attack-surface-http` | Every HTTP endpoint for input validation, auth, injection | Framework-specific endpoint patterns, middleware behavior, routing quirks, default security gaps |
| Authorization Rules | `attack-surface-authz` | Access control rules, permission models | Framework-specific ACL syntax, rule evaluation semantics, permission escalation paths |
| Frontend & Client-Side | `attack-surface-frontend` | XSS, client-side injection, state management | Framework-specific template injection patterns, component security, state leaks |
| LLM/AI Security | `attack-surface-llm` | OWASP Top 10 for LLM Applications | Framework-specific AI integration patterns |
| External Integrations | `attack-surface-integrations` | Third-party API clients, webhooks | Framework-specific API client patterns, SDK quirks |

### Surface stage — git history mining

| Agent | Section Name | What It Mines | Plugin Should Add |
|-------|-------------|--------------|-------------------|
| Injection Variants | `git-history-injections` | Past injection fixes in git history | Framework-specific injection patterns to search for |
| Auth Bypass Variants | `git-history-auth-bypass` | Past auth fixes in git history | Framework-specific auth bypass patterns |
| SSRF Variants | `git-history-ssrf` | Past SSRF fixes in git history | Framework-specific SSRF patterns |
| Data Exposure Variants | `git-history-data-exposure` | Past data leak fixes in git history | Framework-specific data leak patterns, historical key/secret exposure |

### Surface stage — dependencies & config

| Agent | Section Name | What It Checks | Plugin Should Add |
|-------|-------------|----------------|-------------------|
| Dependency Audit | `dependency-audit` | Package vulnerabilities, supply chain | Framework-specific packages with known CVEs, version-specific concerns, supply chain risks |
| Infrastructure Config | `config-infrastructure` | Security headers, secrets, IAM, TLS | Framework-specific config files, defaults, secret management patterns, deployment settings |

### Logic stage — business logic analysis

| Agent | Section Name | What It Finds | Plugin Should Add |
|-------|-------------|--------------|-------------------|
| Race Conditions | `logic-race-conditions` | TOCTOU, concurrency bugs | Framework-specific transaction semantics, concurrency primitives, atomic operation gaps |
| Privilege Escalation | `logic-authz-escalation` | IDOR, role escalation | Framework-specific privilege escalation vectors |
| Pipeline Exploitation | `logic-pipeline-exploit` | Input → processing → action chain flaws | Framework-specific pipeline patterns |
| Denial of Service | `logic-dos` | Resource exhaustion, amplification | Framework-specific resource limits, default configurations, amplification vectors |

### Tracing stage

| Agent | Section Name | What It Traces | Plugin Should Add |
|-------|-------------|----------------|-------------------|
| Data Flow Tracer | `data-flow-tracer` | End-to-end data flows, sanitization gaps | Framework-specific data flow patterns, ORM behaviors, serialization quirks |

---

## 12. Reference: Recon Step Files

These are the recon output files that the orchestrator checks during plugin detection. Understanding them helps you write accurate `detect.dependencies` and `detect.keywords` criteria.

| Step File | Content | Used for Detection |
|-----------|---------|-------------------|
| `step-01-metadata.md` | Project metadata: languages, frameworks, infrastructure, runtime | `detect.keywords` matches against Framework, Infrastructure, Runtime fields |
| `step-13-dependencies.md` | All dependencies from manifest files (package.json, go.mod, requirements.txt, etc.) | `detect.dependencies` matches against package names listed here |

The `detect.files` criterion does NOT use recon step files — it performs a direct Glob check against the project root at detection time.

---

## Appendix: Existing Plugins

For reference, these plugins already exist. Check them before creating a new plugin to avoid overlap.

| Plugin | Target | Sections Covered |
|--------|--------|-----------------|
| `firebase.md` | Firebase / Firestore / Cloud Functions | HTTP, AuthZ, Frontend, Config, Dependency, Race Conditions, DoS |
| `react.md` | React / React Native | Frontend |
| `nextjs.md` | Next.js | HTTP, Frontend, Config |
| `nodejs-express.md` | Node.js + Express | HTTP, Config |
| `python-fastapi.md` | Python + FastAPI | HTTP, Config |
| `python-django.md` | Python + Django | HTTP, AuthZ, Config |
| `spring-boot.md` | Spring Boot (Java) | Recon, HTTP, AuthZ, Config, Dependency |
| `ruby-rails.md` | Ruby on Rails | Recon, HTTP, AuthZ, Config, Dependency |
| `solidjs.md` | SolidJS / SolidStart | Frontend, HTTP, Config, Dependency |
| `vuejs-nuxtjs.md` | Vue.js / Nuxt.js | Frontend, HTTP, Config, Dependency |
| `angular.md` | Angular | Frontend, Config, Dependency |
| `go-gin.md` | Go (Gin / Echo / Fiber) | HTTP, AuthZ, Config, DoS, Dependency |
| `graphql.md` | GraphQL (Apollo, Yoga, etc.) | Recon, HTTP, AuthZ, Config, Data Flow, DoS |
| `docker-kubernetes.md` | Docker / Kubernetes | Recon, Config, Dependency |
| `aws-gcp-azure.md` | AWS / GCP / Azure Cloud | Recon, HTTP, Config, Data Flow, Dependency |
| `secrets-encryption-storage.md` | Database Secrets Encryption | Recon, AuthZ, Config, Data Flow, Race Conditions, Data Exposure History, Dependency |
| `owasp-infra-top10.md` | OWASP Top 10 Infrastructure Risks | Recon, HTTP, AuthZ, Config, Data Flow, DoS, Race Conditions, Data Exposure History, Dependency |
