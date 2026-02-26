# Security Analysis Checkpoint

## Run
- directory: {RUN_DIR}
- skill_root: {SKILL_ROOT}
- timestamp: {TIMESTAMP}
- scope: {SCOPE}

## Stage Completion

| Stage | Status | Finding Count | Report |
|-------|--------|---------------|--------|
| recon | {RECON_STATUS} | — | recon/index.md |
| surface | {SURFACE_STATUS} | {SURFACE_FINDINGS} | reports/surface.md |
| sbom | {SBOM_STATUS} | — | reports/sbom.md |
| logic | {LOGIC_STATUS} | {LOGIC_FINDINGS} | reports/logic.md |
| tracing | {TRACING_STATUS} | {TRACING_FINDINGS} | reports/tracing.md |
| exploits | {EXPLOITS_STATUS} | {EXPLOITS_FINDINGS} | reports/exploits.md |
| validation | {VALIDATION_STATUS} | {VALIDATION_FINDINGS} | reports/validation.md |
| reporting | {REPORTING_STATUS} | — | reports/final.md |
| remediation | {REMEDIATION_STATUS} | — | reports/fix-plan.md |

## Configuration

### Skipped Agents
{SKIPPED_AGENTS}

### Matched Plugins
{MATCHED_PLUGINS}

### Partition Data
{PARTITION_DATA}

### Critical Data Flows
{CRITICAL_FLOWS}

## Paths
- findings_index: {FINDINGS_DIR}/index.md
- recon_index: {RECON_DIR}/index.md

## Resume

To resume from this checkpoint, read this file, skip completed stages, and continue from the first stage with status `pending` or `in-progress`. All completed stage data is on disk at the paths listed above.
