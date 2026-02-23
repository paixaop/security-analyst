# Execution Group Report Template

This template is used by the orchestrator to write intermediate findings after each execution group. Output to `{RUN_DIR}/{group-filename}`.

---

    # {GROUP_NAME}
    **Run:** {RUN_DIR}
    **Agents:** {AGENT_LIST}
    **Timestamp:** {ISO_TIMESTAMP}
    **Prior Groups:** {PRIOR_GROUP_FILES}

    ---

    ## Severity Summary

    | Severity | Count |
    |----------|-------|
    | Critical | |
    | High     | |
    | Medium   | |
    | Low      | |
    | Info     | |
    | **Total** | |

    ---

    ## Findings

    ### Summary (LOD-0)

    | ID | Severity | Summary |
    |----|----------|---------|
    {LOD0_TABLE}

    ### Finding Details

    Full LOD-2 findings are in `{FINDINGS_DIR}/`. To read any finding: `Read {FINDINGS_DIR}/{FINDING-ID}.md`

    ---

    ## Incidental Findings

    {IX_FINDINGS}

    Incidental findings (IX-prefixed) discovered by this group's agents.

    ---

    ## Notes

    - Agents skipped: {SKIPPED_AGENTS} (and why)
    - Cross-references to prior group findings: {CROSS_REFS}
