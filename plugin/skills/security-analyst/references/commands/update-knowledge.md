---
description: "Refresh the vulnerability knowledge base — OWASP cheat sheets, Top 10 lists, and CWE Top 25."
model: fast
---

**MANDATORY — print this banner before ANY other output, tool call, or action:**

> **Security Analyst** v1.0.0 · update-knowledge · paixaop/security-analyst · AGPL-3.0
> Authorized use only — see SKILL.md Disclaimer

# Security Analyst — Update Knowledge Base

You are executing the `update-knowledge` command. Your job is to refresh the local vulnerability knowledge files that ground all security analysis.

## Paths

```
SKILL_ROOT    = directory containing SKILL.md
KNOWLEDGE_DIR = {SKILL_ROOT}/references/knowledge
```

## Steps

### 1. Update OWASP Cheat Sheets

Check if the OWASP CheatSheetSeries git submodule exists:

```bash
ls {KNOWLEDGE_DIR}/owasp-cheatsheets/.git 2>/dev/null
```

- **If submodule exists:** Update it:
  ```bash
  git submodule update --remote --merge {KNOWLEDGE_DIR}/owasp-cheatsheets
  ```

- **If submodule does not exist:** Add it:
  ```bash
  git submodule add https://github.com/OWASP/CheatSheetSeries.git {KNOWLEDGE_DIR}/owasp-cheatsheets
  ```

### 2. Refresh OWASP Top 10 Resources

Using your web-page reading capability, read each URL and save the content as markdown:

| URL | Output File |
|-----|-------------|
| https://owasp.org/www-project-top-ten/ | `{KNOWLEDGE_DIR}/owasp-top-10/web.md` |
| https://owasp.org/www-project-api-security/ | `{KNOWLEDGE_DIR}/owasp-top-10/api.md` |
| https://genai.owasp.org/ | `{KNOWLEDGE_DIR}/owasp-top-10/llm.md` |

### 3. Refresh CWE Top 25

Read https://cwe.mitre.org/top25/ and save as `{KNOWLEDGE_DIR}/cwe-top-25/top-25.md`.

### 4. Fill Missing Cheat Sheets

Check for these critical cheat sheets. For any that are missing from `{KNOWLEDGE_DIR}/owasp-cheatsheets/`, read the corresponding URL and save as `.md`:

| Cheat Sheet | URL |
|-------------|-----|
| XSS Prevention | https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Scripting_Prevention_Cheat_Sheet.html |
| SQL Injection Prevention | https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html |
| Authentication | https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html |
| Authorization | https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html |
| Input Validation | https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html |
| Output Encoding | https://cheatsheetseries.owasp.org/cheatsheets/Output_Encoding_Cheat_Sheet.html |
| Cryptographic Storage | https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html |
| REST Security | https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html |
| Session Management | https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html |
| Logging | https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html |
| Deserialization | https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html |
| File Upload | https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html |
| CSRF Prevention | https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html |
| Content Security Policy | https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html |
| Password Storage | https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html |
| Key Management | https://cheatsheetseries.owasp.org/cheatsheets/Key_Management_Cheat_Sheet.html |

### 5. Write Timestamp

Write the current date (YYYY-MM-DD) to `{KNOWLEDGE_DIR}/last-updated.txt`.

### 6. Report

Output ONLY:

```
Knowledge updated on YYYY-MM-DD. Files changed: [list of files written or updated]. Re-run full analysis to verify grounding.
```
