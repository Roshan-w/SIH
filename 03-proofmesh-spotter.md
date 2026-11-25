# Agent 3 · proofmesh-spotter

**Job:** look at SQL statements and flag columns that hold personal data. If any are found, the data owner must confirm before the run continues.

```
START → spot (Ask AI) → owner_check (Approval, only if personal data) → record (Calculate) → END
```

**Status of your export:** ✅ correct after your `'done'` fix. One thing to check while testing (Step 7).

---

## Step 1 · Create and name it

| Field | Value |
|---|---|
| Agent Name | `proofmesh-spotter` |
| Vertical | `horizontal` |
| Description | `Flags columns that hold personal data. A data owner confirms the list.` |
| Governance Level | **Strict** |
| Framework | LangGraph |

Press **Ctrl+S**.

---

## Step 2 · State fields

| Field | Type | Req | Description |
|---|---|---|---|
| `statements` | list (string) | ✔ | SQL statements from the finder. |
| `data_owner` | string | ✔ | Email of the person who confirms personal data. |
| `personal_fields` | list (string) | | Columns holding personal data, as "table.column". Filled by the AI. |
| `has_personal_data` | bool | | True if any personal column is used. Filled by the AI. |
| `reason` | string | | Why each column was flagged. Filled by the AI. |
| `owner_confirmed` | string | | Set to done after the data owner approves. |

That's 6 fields: 2 required. **Delete** the extra `result` field the Builder added.

---

## Step 3 · Prompt

**Prompts** → key `spotter_prompt` → **+**.

```
You are a privacy reviewer. Look at the SQL statements below and list every column
that holds personal data about a person: name, date of birth, address, phone, email,
government ID, account or card number, salary.

Do NOT flag technical columns such as job_id, merchant_name, email_template,
created_at or status. Give each flagged column as table.column with a short reason.
If you are not sure, flag it; a person will check.

SQL:
{{ state.statements }}
```

---

## Step 4 · `spot` (Ask AI, fields)

| Setting | Value |
|---|---|
| Step name | `spot` |
| What this step does | `AI flags personal-data columns.` |
| Instructions for the AI | `spotter_prompt` |
| AI model | Auto |

**What the AI must answer with:**

| Name | Type | Description |
|---|---|---|
| `personal_fields` | List (text) | Personal-data columns as table.column. |
| `has_personal_data` | Yes/No | True if any column was flagged. |
| `reason` | Text | Why each column was flagged. |

**Save parts of the answer:** map all 3 to the state fields with the same names.

---

## Step 5 · `owner_check` (Approval)

Click **+** under `spot` and choose **Approval**.

| Setting | Value |
|---|---|
| Step name | `owner_check` |
| What this step does | `Data owner confirms the personal-data list.` |
| Message | (below) |
| Who may approve | insert `{{ state.data_owner }}`, and also add your own email for testing |
| Decide within (hours) | `24` |
| The person who started the run may approve | ✔ for testing |

Message (insert the placeholders from Insert):
```
Please confirm these personal-data columns: {{ state.personal_fields }}.
Reason: {{ state.reason }}
```

**Advanced settings:**

| Setting | Value |
|---|---|
| Run this step only when | `has_personal_data` · **is** · value · `true` |
| If this step fails | Stop the run |

---

## Step 6 · `record` (Calculate values)

Required after a conditional Approval.

| Setting | Value |
|---|---|
| Step name | `record` |
| What this step does | `Record that the owner check is done.` |
| Field → formula | `owner_confirmed` → `'done'` (**with** single quotes) |

Press **Ctrl+S**.

---

## Step 7 · Test

**Test A: personal data.** Put **your own email** in `data_owner`. Expect `has_personal_data` = true, `personal_fields` to include `customer.first_name` and `customer.dob` but not `batch_job.job_id`, a pause for approval, then after you approve, `owner_confirmed` = `done`.
```json
{"data_owner": "roshan.warade@zensar.com",
 "statements": ["line 2: SELECT first_name, city, dob FROM customer WHERE status = 'A'",
                "line 6: SELECT job_id, job_name FROM batch_job"]}
```

**Test B: no personal data.** Expect `has_personal_data` = false, no pause, and `owner_confirmed` = `done`.
```json
{"data_owner": "roshan.warade@zensar.com",
 "statements": ["line 1: SELECT job_id, status FROM batch_job", "line 3: SELECT COUNT(*) FROM merchant_category"]}
```

**Test C: reject.** Run Test A and click **Reject**. Expect the run to end as rejected, with `owner_confirmed` empty.

**If Test A doesn't pause:**
1. Click **Copy snapshot** next to `spot` and look at `has_personal_data`.
2. If it's `true`, the condition doesn't match. In owner_check → Run this step only when, try the Yes/No choice instead of the typed `true`, or use the value `True`.
3. If it's `false`, check that `has_personal_data` is mapped under Save parts of the answer.

---

## Done checklist
- [ ] 6 state fields; the `result` field deleted.
- [ ] 3 steps: spot → owner_check → record.
- [ ] Approval only runs when `has_personal_data` is true.
- [ ] `record` uses `'done'` with quotes.
- [ ] Tests A, B and C behave as expected.

## If something goes wrong
| Problem | Fix |
|---|---|
| "unknown name 'done'" | Add quotes: `'done'` |
| You can't see the approval | `data_owner` isn't your email; change the test input |
| Technical columns are flagged | Fine. A person checks the list; you can add the column name to the "Do NOT flag" line |
