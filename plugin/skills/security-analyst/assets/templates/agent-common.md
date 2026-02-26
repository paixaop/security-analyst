# Shared Agent Output Instructions

## Grounding Mandate

The following vulnerability patterns are from the OWASP Cheat Sheet Series and CWE Top 25 knowledge base, selected for your specific focus area by the KnowledgeRouter. Every finding you report MUST cite the exact knowledge `Source` (format: `file:section`) that identifies the matched vulnerability pattern. If no pattern below matches what you observe, state "no known pattern" in the finding's Known Patterns Matched section.

{KNOWLEDGE_CHECKS}

## Finding Output (LOD Architecture)

For each finding you discover:

1. **Write the atomic finding file** to `{FINDINGS_DIR}/{FINDING-ID}.md` using the LOD-2 format from `{FINDING_TEMPLATE_PATH}`
2. **Build your LOD-0 summary table** — one row per finding. The ID column MUST be a markdown link to the atomic finding file: `| [{ID}](findings/{FINDING-ID}.md) | [Severity] | [One-sentence summary] |`

When you are done with all analysis, return ONLY the following to the orchestrator:

```
## Agent: {AGENT_NAME}

**Endpoints/items analyzed:** [count]
**Findings:** [count] ([severity breakdown])

### LOD-0 Summary

| ID          | Severity | Summary        |
| ----------- | -------- | -------------- |
| [HTTP-001](findings/HTTP-001.md) | High     | [one sentence] |
| [HTTP-002](findings/HTTP-002.md) | Medium   | [one sentence] |

### IDOR-Suspect Endpoints (if applicable)

[list of endpoints flagged for logic stage deep dive]

### Incidental Findings

[IX-prefixed findings in abbreviated format]
```

Do NOT send full finding details in your return message. They are on disk.

## Target Subsetting (Pre-Chunked Agents)

The orchestrator may assign you a **subset** of targets when the full target list exceeds chunking thresholds. If your prompt includes a `TARGET SUBSET` section, analyze ONLY those items — other chunks are handled by parallel instances of the same agent.

When pre-chunked, your finding IDs start at the offset specified in your prompt (e.g., `HTTP-101` for batch 2).

## PARTIAL Return Protocol

If you are working through a large list of targets and risk exhausting your context window, **stop early and return with PARTIAL status** rather than producing incomplete or degraded analysis. Write all findings discovered so far to disk, then return:

```
## Agent: {AGENT_NAME} (PARTIAL)

**Status:** PARTIAL — context budget reached
**Processed:** [count processed] / [total count]
**Remaining targets:**
- [target identifier] — file:line
- [target identifier] — file:line
- ...

### LOD-0 Summary

| ID          | Severity | Summary        |
| ----------- | -------- | -------------- |
| [{ID}](findings/{ID}.md) | ...      | ...            |

### Incidental Findings

[if any]
```

The orchestrator will spawn a **continuation agent** for the remaining targets, with findings from your partial run included as prior context. The continuation inherits your finding ID sequence (e.g., if you stopped at HTTP-012, the continuation starts at HTTP-013).

**When to return PARTIAL:**
- You have processed ~25 targets (endpoints, integrations, components, or findings) and a comparable or larger number remains
- You notice tool call responses are being truncated or you are running low on output capacity
- You are reading very large source files (500+ lines each) and have processed ~15 targets

Err toward returning PARTIAL early rather than risking incomplete analysis at the end. A clean PARTIAL with good findings is far better than a complete run with degraded analysis on the last 30% of targets.

## Reading Prior Findings

Your `{PRIOR_FINDINGS_SUMMARY}` contains LOD-0 summaries from earlier execution groups. If you need full detail on a specific finding:

1. Note the finding ID from the LOD-0 table
2. `Read` the file at `{FINDINGS_DIR}/{FINDING-ID}.md` for the full LOD-2 content
3. Only read findings relevant to your analysis — do not read all of them

## Incidental Findings

While performing your primary analysis, you may discover security issues OUTSIDE your focus area:

- Use finding ID prefix `IX-{AGENT_PREFIX}-XXX` (IX = incidental)
- Use abbreviated format (severity, file:line, one-paragraph description, ATT&CK technique if obvious)
- Do NOT write atomic files for incidentals — include them in your return message only
- Do NOT let incidental findings distract from your primary analysis

## File Exclusions

When searching the codebase, **only search project source files** — never dependency, generated, or build output. A single unscoped Grep across `node_modules/` can waste thousands of tokens on irrelevant code.

### Primary mechanism: respect .gitignore

Use `git ls-files` to enumerate tracked source files. Everything in `.gitignore` (dependencies, build output, caches, IDE files, logs) is excluded automatically.

```bash
# List all tracked source files
git ls-files

# List tracked files matching a pattern
git ls-files '*.ts' '*.js'

# List tracked files in a specific directory
git ls-files 'src/'
```

When you need to search content, the **Grep tool respects `.gitignore` by default** (it uses ripgrep). Searches via the Grep tool automatically skip `node_modules/`, `dist/`, `.git/`, and anything else the project gitignores. Prefer Grep over `git grep` or shell grep for this reason.

### Additional exclusions (may not be in .gitignore)

Some directories are tracked but still wasteful to search broadly:

- `*.min.js`, `*.min.css` — minified files (unreadable, no security value)
- `*.map` — source maps
- `*.snap` — test snapshots
- `**/fixtures/`, `**/testdata/` — test data (read only when tracing a specific code path)
- `*.generated.*`, `*.pb.go`, `*_pb2.py` — generated code (read only if referenced by source)
- `*.lock`, `package-lock.json`, `yarn.lock` — lockfiles (read directly for dependency analysis, not via broad search)

### Scoping Grep and Glob searches

**Grep:** Use the `path` parameter to restrict to source directories (e.g., `path: "src/"`) or the `glob` parameter for file types (e.g., `glob: "*.{ts,js}"`). Both are more efficient than searching the project root.

**Glob:** Use specific directory prefixes from recon step files (e.g., `src/**/*.ts`) rather than broad root patterns (e.g., `**/*.ts`).

**Shell grep:** If you must use shell-based grep, scope it: `git ls-files '*.ts' | xargs grep -l 'pattern'` or `rg 'pattern' src/`.

### Exception

The `dependency-audit` and `config-infrastructure` agents may intentionally read `node_modules/` package metadata, lockfiles, or config files in build output. These are targeted reads of specific known files, not broad searches.

## Targeted Reading

When reading source files referenced in recon step files, use **targeted line ranges** instead of reading entire files:

1. Use the `file:line` references from recon step files as your anchor point
2. Read ~60 lines centered on the target: 30 lines before and 30 lines after the referenced line number (use the Read tool's `offset` and `limit` parameters)
3. If you need more context (e.g., to find the end of a function or trace an import), expand incrementally in 30-line steps — do not read the entire file
4. **Exception:** files under ~100 lines — read the whole file

This prevents large source files (1000+ lines) from consuming excessive context. A 2000-line file read in full uses ~20x more context than the ~100-line function body you actually need.

## Trust Model Awareness

If `{RECON_DIR}/step-04-boundaries.md` contains a "Documented Trust Model" section (extracted from the project's SECURITY.md or security policy), use it to calibrate your analysis:

1. **Check trust levels before reporting.** If the documented model explicitly declares a component as trusted (e.g., "internal services are trusted"), a finding that assumes that component is hostile requires stronger evidence. Annotate such findings with `[TRUST-MODEL: contradicts documented trust level — {reason it still applies}]` or downgrade/omit if the trust assumption is reasonable.
2. **Respect accepted risks.** If the documented model lists an accepted risk that matches your finding, annotate the finding with `[ACCEPTED-RISK: documented in {source document}]` and set severity to Info unless you can demonstrate the acceptance is invalid (e.g., the risk has changed since acceptance).
3. **Flag mismatches.** If you discover a code path that violates a documented security invariant, this is *higher* severity than a generic finding — the project explicitly depends on that invariant holding. Annotate with `[INVARIANT-VIOLATION]`.

This reduces false positives from findings that the project maintainers have already assessed while surfacing violations of the project's own security contract.

## External Audit Cross-Reference

If `{RECON_DIR}/step-04-boundaries.md` contains an "External Audit Findings" section (from openclaw or other tools), or `{RECON_DIR}/step-10-security-work.md` contains "External Audit Tool Results":

1. **Avoid re-reporting confirmed findings.** If an external tool has already confirmed a finding with the same root cause and affected file, reference the external finding ID instead of creating a duplicate. Use: `[CORROBORATES: {tool} {finding-id}]` in your finding.
2. **Deepen external findings.** If your analysis reveals additional attack surface, exploit chains, or severity factors beyond what the external tool reported, create a new finding that references the external one and explains the additional impact.
3. **Identify gaps.** If external audit results exist but miss patterns you find, note this — it helps calibrate the external tool's coverage.

## Reading Recon Data

Your recon context comes in three levels:
- **LOD-0 + LOD-1**: In `{RECON_INDEX_PATH}` — summary table + paragraph per section
- **LOD-2**: Atomic step files in `{RECON_DIR}/step-{NN}-{name}.md` — full tables with file:line references

Your "Inputs" section specifies which step files are most relevant to your analysis. Read the index first, then selectively read the step files you need. Do not read all 14 step files unless your analysis requires broad context.
