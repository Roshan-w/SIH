# Agent 5 · proofmesh-onboarder

**Job:** before a check starts, ask the developer for the repo link, database type, data owner and reviewer, then mark the project ready or not.

```
START → ask_details (Ask a person) → unpack (Calculate) → check_ready (If/then) → END
```

**Status of your export:** ❌ **2 fixes needed.**
1. Rule 2 in `check_ready` has a **blank condition**, so `ready` can always come out as `no`.
2. `repo_url` is optional, so `unpack` crashes when it's left blank.

Steps 4 and 6 below include both fixes.

---

## Step 1 · Create and name it

| Field | Value |
|---|---|
| Agent Name | `proofmesh-onboarder` |
| Vertical | `horizontal` |
| Description | `Asks the developer for the repo, the database type and the data owner before a check starts.` |
| Governance Level | **Standard** |
| Framework | LangGraph |

---

## Step 2 · State fields

| Field | Type | Req | Description |
|---|---|---|---|
| `requested_by` | string | ✔ | Email of the developer. |
| `details` | dict | | The answers from the details form. |
| `repo_url` | string | | Azure DevOps or Git repo link. |
| `db_type` | string | | Database type, such as postgres, oracle, sqlserver, db2. |
| `data_owner` | string | | Email of the data owner. |
| `reviewer` | string | | Email of the code reviewer. |
| `ready` | string | | yes or no. |

That's 7 fields: 1 required. **Delete** the extras: `field_2`, `field_3`, `field_4`, `result` and `ask_user_result`.

No prompt is needed for this agent.

---

## Step 3 · (no prompt)

Skip the Prompts panel.

---

## Step 4 · `ask_details` (Ask a person)

Click **+** between START and END and choose **Ask a person**.

| Setting | Value |
|---|---|
| Step name | `ask_details` |
| What this step does | `Ask the developer for repo, database type, data owner and reviewer.` |
| Question | `Before we check your SQL, please give: repo link, database type, data owner email and reviewer email.` |

**Form fields:** delete the example rows (`confirmed`, `comment`), then **+ Add a field** four times:

| Name | Type | required | Description |
|---|---|---|---|
| `repo_url` | Text | ✔ (**fix**) | Azure DevOps or Git repo link |
| `db_type` | Text | ✔ | Database type, e.g. postgres, oracle, sqlserver, db2 |
| `data_owner` | Text | ✔ | Email of the person who confirms personal data |
| `reviewer` | Text | ✔ | Email of the code reviewer |

**Below the preview:**

| Setting | Value |
|---|---|
| Save the answer as | `details` |
| Who may answer | `roshan.warade@zensar.com` (later: the developer, or `{{ state.requested_by }}`) |
| Answer within (hours) | `24` |
| The person who started the run may answer | ✔ for testing |
| Advanced → A platform admin may answer instead | ☐ |
| Advanced → Run this step only when | Always runs |
| Advanced → If this step fails | Stop the run |
| Advanced → Note for the AI's working memory | empty |

---

## Step 5 · `unpack` (Calculate values)

This splits the saved form answer into separate fields.

| Setting | Value |
|---|---|
| Step name | `unpack` |
| What this step does | `Copy the form answers into their own fields.` |

| Field | Formula (type exactly) |
|---|---|
| `repo_url` | `state.details['repo_url']` |
| `db_type` | `state.details['db_type']` |
| `data_owner` | `state.details['data_owner']` |
| `reviewer` | `state.details['reviewer']` |

---

## Step 6 · `check_ready` (If / then values)

| Setting | Value |
|---|---|
| Step name | `check_ready` |
| What this step does | `Mark the project ready when an owner and reviewer are given.` |

Rules:

| Rule | Condition | Then set `ready` = value |
|---|---|---|
| Rule 1 | `data_owner` · **is empty** | `no` |
| Rule 2 (**fix**: it's blank now) | `reviewer` · **is empty** | `no` |
| Otherwise | (none) | `yes` |

If there's no "is empty" operator, use `is` · value · and leave the box empty. Never leave the whole condition blank; if you can't set it, **delete rule 2** (the form already makes reviewer required).

Press **Ctrl+S**.

---

## Step 7 · Test

**Test A: full answer.** Input:
```json
{"requested_by": "roshan.warade@zensar.com"}
```
The form appears in the test panel. Fill in:
- repo_url: `https://dev.azure.com/demo/proofmesh`
- db_type: `postgres`
- data_owner: `roshan.warade@zensar.com`
- reviewer: `roshan.warade@zensar.com`

Submit. Expect the 4 fields to be filled, and `ready` = `yes`.

**Test B: decline.** Run again and **decline** the form. Expect the run to end as rejected.

**If `unpack` errors:** click **Copy snapshot** on `ask_details` and look at how `details` is stored. If the answers sit inside another key (for example `details.answer.db_type`), change each formula to match, such as `state.details['answer']['db_type']`.

---

## Done checklist
- [ ] 7 state fields; the 5 extras deleted.
- [ ] Form has 4 fields, all required.
- [ ] Answer saved as `details`.
- [ ] `unpack` has 4 formulas.
- [ ] `check_ready` has no blank condition.
- [ ] Test A gives `ready` = `yes`.

## If something goes wrong
| Problem | Fix |
|---|---|
| `ready` is always `no` | Rule 2 is blank; fix or delete it |
| KeyError 'repo_url' | Tick required on repo_url in the form |
| You can't fill the form | Tick "The person who started the run may answer" |
