---
description: "Open a pull request to contribute a security plugin to the security-analyst project on GitHub."
---

# Security Analyst — Contribute Plugin

Open a pull request to contribute a plugin to the security-analyst project repository.

**Arguments:**
- `[plugin-name]` — kebab-case plugin name without `.md` extension (required)

If no plugin name is provided, list available plugins in `{PLUGINS_DIR}/` and ask the user which one to contribute.

**Read `references/constants.md` for all directory paths, filenames, and placeholder variables.**

**Requires:** `gh` CLI (GitHub CLI) authenticated with a GitHub account that can fork public repositories.

## Execution

**MANDATORY — print this banner before ANY other output, tool call, or action:**

> **Security Analyst** v1.0.0 · contribute-plugin · Pedro Paixao · AGPL-3.0
> Authorized use only — see SKILL.md Disclaimer

### Step 1: Validate the Plugin

1. Check that `{PLUGINS_DIR}/{plugin-name}.md` exists. If not, tell the user:
   > "Plugin file `{plugin-name}.md` not found in `{PLUGINS_DIR}/`. Run `/security-analyst:create-plugin {framework}` first."
2. Read the plugin file and validate:
   - **Frontmatter**: has `name`, `detect` with at least one non-empty list (`files`, `dependencies`, or `keywords`)
   - **Agent sections**: every `## ` heading matches an agent prompt filename (without `.md`) from `{PROMPTS_DIR}/`
   - **Content**: at least one agent section with non-empty content
3. If validation fails, report the issues and stop. Do not proceed with an invalid plugin.

### Step 2: Determine Context

Check whether the current workspace is the security-analyst repository itself or a project where the skill is installed:

1. Check if `plugin/skills/security-analyst/references/plugins/` exists relative to the project root AND `.git` exists
2. Run: `git remote -v` and check if any remote URL contains `paixaop/security-analyst`

**If in the security-analyst repo:**
- Set `REPO_DIR` to the current workspace root
- Set `PLUGIN_SOURCE` to `{REPO_DIR}/plugin/skills/security-analyst/references/plugins/{plugin-name}.md`
- Skip cloning — work directly in the repo

**If in an installed skill (or other project):**
- The plugin file is at `{PLUGINS_DIR}/{plugin-name}.md` (inside the installed skill)
- Fork and clone the source repo:
  ```
  gh repo fork paixaop/security-analyst --clone --remote
  ```
- Set `REPO_DIR` to the cloned directory
- Copy the plugin file:
  ```
  cp {PLUGINS_DIR}/{plugin-name}.md {REPO_DIR}/plugin/skills/security-analyst/references/plugins/{plugin-name}.md
  ```

### Step 3: Branch and Commit

1. Change to `REPO_DIR`
2. Create and switch to a feature branch:
   ```
   git checkout -b plugin/{plugin-name}
   ```
3. Read `{REPO_DIR}/plugin/skills/security-analyst/references/plugins/README.md`
4. Add the new plugin to the `## Available Plugins` table in README.md (if not already present):
   - Read the plugin frontmatter for `name`
   - List the `## ` headings to determine which sections are covered
   - Map section names to display names: `attack-surface-http` → `HTTP`, `attack-surface-authz` → `AuthZ`, `attack-surface-frontend` → `Frontend`, `config-infrastructure` → `Config`, `dependency-audit` → `Dependency Audit`, `logic-race-conditions` → `Race Conditions`, `logic-dos` → `DoS`, `data-flow-tracer` → `Data Flow`, `recon-agent` → `Recon`, `git-history-*` → `{Type} History`
   - Insert the row alphabetically by plugin filename
5. Stage and commit:
   ```
   git add plugin/skills/security-analyst/references/plugins/{plugin-name}.md
   git add plugin/skills/security-analyst/references/plugins/README.md
   git commit -m "Add {plugin-display-name} plugin

   Adds framework-specific security checks for {plugin-display-name}.
   Detection: {summary of detect criteria}
   Sections: {comma-separated section list}"
   ```

### Step 4: Push and Open PR

1. Push the branch:
   ```
   git push -u origin plugin/{plugin-name}
   ```
2. Open a pull request:
   ```
   gh pr create \
     --repo paixaop/security-analyst \
     --title "Add {plugin-display-name} plugin" \
     --body "$(cat <<'EOF'
   ## New Plugin: {plugin-display-name}

   ### Detection Criteria
   - **Files:** {detect.files list or "none"}
   - **Dependencies:** {detect.dependencies list or "none"}
   - **Keywords:** {detect.keywords list or "none"}

   ### Agent Sections
   {for each section: "- **{section-name}**: {number of checks} checks — {brief description}"}

   ### Testing
   - [ ] Ran `/security-analyst:recon` against a project using {framework}
   - [ ] Plugin was auto-detected after recon
   - [ ] Ran focused analysis and confirmed meaningful findings
   - [ ] Checked for overlap with existing plugins

   ### Checklist
   - [x] Frontmatter has valid `name` and `detect` criteria
   - [x] All `## ` headings match agent prompt filenames
   - [x] Checks use offensive framing with specific code patterns
   - [x] No duplication of generic checks from base agent prompts
   - [x] README.md table updated
   EOF
   )"
   ```

### Step 5: Present Results

Show the user:
- The PR URL
- Branch name
- Summary of what was included

> "Pull request opened: {PR_URL}
>
> Branch: `plugin/{plugin-name}`
> Plugin: `{plugin-display-name}` with {N} agent sections
>
> The PR includes the plugin file and an updated README.md table. Review the testing checklist in the PR description and complete any items before requesting review."

**STOP. The contribution is complete. Do not take any further actions or continue reasoning. Wait for the user to ask a follow-up question.**

## Fallback: No `gh` CLI

If `gh` is not available or not authenticated:

1. Tell the user the `gh` CLI is required
2. Provide manual instructions:
   > "To contribute manually:
   > 1. Fork `https://github.com/paixaop/security-analyst` on GitHub
   > 2. Clone your fork and create a branch: `git checkout -b plugin/{plugin-name}`
   > 3. Copy `{PLUGINS_DIR}/{plugin-name}.md` to `plugin/skills/security-analyst/references/plugins/`
   > 4. Update the README.md Available Plugins table
   > 5. Commit, push, and open a PR against `paixaop/security-analyst`"
