# Agent 4 · proofmesh-checker

**Job:** turn the results into a pull request description with the proof attached. It uses a template, not AI, so the text is always the same shape and never overstates the proof.

```
START → write_pr (Write a message) → END
```

**Status of your export:** ❌ **fix needed.** The template currently has instructions ("Write a short…") that would be printed word for word. Replace it with the template in Step 3.

---

## Step 1 · Create and name it

| Field | Value |
|---|---|
| Agent Name | `proofmesh-checker` |
| Vertical | `horizontal` |
| Description | `Writes the pull request description with the proof attached. Never merges.` |
| Governance Level | **Strict** |
| Framework | LangGraph |

---

## Step 2 · State fields

| Field | Type | Req | Description |
|---|---|---|---|
| `file_name` | string | ✔ | File that changed. |
| `what_changed` | string | ✔ | From sql-check. |
| `proof_tier` | string | ✔ | GREEN, AMBER, RED or OUT. |
| `proof_summary` | string | ✔ | From the proof engine. |
| `decision` | string | ✔ | From sql-check. |
| `personal_fields` | list (string) | | From the spotter. |
| `pr_text` | string | | The pull request description. |

That's 7 fields: 5 required. **Delete** the extra `render_result` field.

---

## Step 3 · Template (Prompts)

1. Open **Prompts** and select `pr_prompts`. If you're starting fresh, use the key `pr_template`.
2. **Delete everything in it** and paste this. Re-insert each placeholder from **Insert**:

```
## What changed
{{ state.what_changed }} (file: {{ state.file_name }})

## Proof
Tier: **{{ state.proof_tier }}**
{{ state.proof_summary }}

## Personal data
{{ state.personal_fields }}

## Status
{{ state.decision }}

_Needs reviewer approval. Not auto-merged._
```

There are no instructions in it: this is the exact text of the PR.

---

## Step 4 · `write_pr` (Write a message)

| Setting | Value |
|---|---|
| Step name | `write_pr` |
| What this step does | `Fill the PR description template.` |
| Template | `pr_prompts` (or `pr_template`) |
| Format | Markdown |
| Save the result as | `pr_text` |

Press **Ctrl+S**.

---

## Step 5 · Guard (optional, recommended)

| Guard | Settings | on_fail |
|---|---|---|
| Enum Check | `proof_tier` in `GREEN, AMBER, RED, OUT` | reject |

---

## Step 6 · Test

**Test A.** Expect `pr_text` to start with "## What changed", show "Tier: **GREEN**", and end with "Not auto-merged."
```json
{"file_name": "CustomerDao.java",
 "what_changed": "Reordered the WHERE conditions and added a table alias.",
 "proof_tier": "GREEN",
 "proof_summary": "Old and new matched on 12 rows. Planted mistakes caught: 3 of 3.",
 "personal_fields": ["customer.first_name"],
 "decision": "ready_for_review"}
```

**Test B: no personal data.** Expect the Personal data section to show `[]` or be empty, with no error.
```json
{"file_name": "report.sql", "what_changed": "Added an index hint.", "proof_tier": "AMBER",
 "proof_summary": "Both queries returned no rows.", "personal_fields": [], "decision": "needs_person_check"}
```

---

## Done checklist
- [ ] The template has no instructions, only the PR text.
- [ ] `render_result` deleted.
- [ ] `pr_text` is filled after the run.
- [ ] Tests A and B pass.

## If something goes wrong
| Problem | Fix |
|---|---|
| `pr_text` contains "Write a short pull request…" | The old template is still there; replace it (Step 3) |
| `pr_text` is empty but `render_result` is filled | Set "Save the result as" to `pr_text` |
| A section is blank | Re-insert that placeholder from the Insert list |
