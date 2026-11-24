# ProofMesh — AgentMesh build pack

Everything here is copy-paste ready. Field names, step types and buttons are taken from the AgentMesh Low Code manual (PPE, 6 Oct 2026).

Before you start:
- Zensar laptop on the Cisco VPN, Chrome or Edge.
- **Do not lose the VPN during a test.** A paused approval is lost if you do (manual 14.6).

## What we build first (the core agent, about 1 hour)

Build one agent that works end to end, then grow it.

```
START → explain_change (Ask AI) → decide (If/then) → reviewer_approval (Approval) → finalize (Calculate) → END
         + guards: required fields, tier must be GREEN/AMBER/RED/OUT
```

**The bridge (until the proof engine is a tool):** the developer runs our engine on their laptop and pastes its result into the agent.

```bash
diamond compare old.sql new.sql
```

It prints a tier (GREEN, AMBER or RED) and a summary. The agent reads those as inputs. The agent never decides "safe" itself.

---

## Step 1 · Create and name it (Meta)

1. **New Agent → Blank agent.**
2. **Meta panel:** set these fields first. The Builder auto-saves under whatever name it has.

| Field | Value |
|---|---|
| Agent Name | `proofmesh-sql-check` |
| Vertical | `horizontal` |
| Description | `Explains a SQL change, reads its proof from the Diamond engine, and sends it to a person for approval. Never marks a change safe on its own.` |
| Governance Level | **Strict** (masks personal data; stops if the checker is down) |
| Framework | LangGraph (leave it) |

---

## Step 2 · State fields

Open **State**. Type each name, press Enter, then set the Type, tick Req for inputs, and paste the Description.

| Field | Type | Req | Description |
|---|---|---|---|
| `old_sql` | string | ✔ | The SQL query before the change. |
| `new_sql` | string | ✔ | The SQL query after the change. |
| `proof_tier` | string | ✔ | Tier from the Diamond proof engine: GREEN, AMBER, RED or OUT. |
| `proof_summary` | string | ✔ | Plain-English proof summary printed by the Diamond engine. |
| `explanation` | string | | What the old and new query do, in plain English. Filled by the AI. |
| `what_changed` | string | | The difference between old and new. Filled by the AI. |
| `touches_personal_data` | bool | | True if the query reads or writes personal data. Filled by the AI. |
| `personal_fields` | list (string) | | Personal-data columns used, such as name, date of birth, address. Filled by the AI. |
| `risk_notes` | string | | Anything a reviewer should look at. Filled by the AI. |
| `decision` | string | | ready_for_review, needs_person_check or blocked. Set by the rule step. |
| `result` | string | | Final outcome. |

---

## Step 3 · Prompt

Open **Prompts**, set the key to `explain_prompt`, and click **+**.

Insert the `{{ state.… }}` chips from the **Insert** list. Don't type them: a misspelled placeholder becomes blank without any error.

```
You are a senior database reviewer. Compare the OLD and NEW SQL below.

1. Explain in plain English what the old query does.
2. Say exactly what changed in the new query.
3. Say whether either query reads or writes personal data (names, dates of birth,
   addresses, phone, email, ID numbers) and list those columns.
4. Note anything a reviewer should check. Do NOT say whether the change is safe;
   a separate proof engine decides that.

OLD SQL:
{{ state.old_sql }}

NEW SQL:
{{ state.new_sql }}
```

Test it in **Play** with a sample query before going further.

---

## Step 4 · AI step (Ask AI, fields)

1. Click **+** between START and END and choose **Ask AI (fields)**.
2. Click the new step and set these:
   - **Name:** `explain_change`
   - **Description:** `AI explains the change and spots personal data.`
   - **Instructions for the AI:** `explain_prompt`
   - **AI model:** Auto (the LLM Router picks it)
3. Under **What the AI must answer with**, add these rows (or use **Edit as JSON**):

| Name | Type | Description |
|---|---|---|
| explanation | Text | What the old and new query do. |
| what_changed | Text | The exact difference. |
| touches_personal_data | Yes/No | Reads or writes personal data. |
| personal_fields | List | Personal-data columns used. |
| risk_notes | Text | What a reviewer should check. |

4. Under **Save parts of the answer**, map each field to the state field of the same name.

---

## Step 5 · Rule step (If / then values)

1. Click **+** under `explain_change` and choose **If / then values**.
2. Name it `decide`, then add these rules in this order (the first match wins):

| # | Condition | Then set |
|---|---|---|
| 1 | `proof_tier` **is** `GREEN` | `decision` = `ready_for_review` |
| 2 | `proof_tier` **is** `AMBER` | `decision` = `needs_person_check` |
| 3 | `proof_tier` **is** `OUT` | `decision` = `ready_for_review` |
| Otherwise | | `decision` = `blocked` |

"Otherwise → blocked" is our rule that unsure means RED. Use the no-code condition builder, not a formula (manual known issue 3).

---

## Step 6 · Human approval (Approval)

1. Click **+** under `decide` and choose **Approval**.
2. Name it `reviewer_approval`.
3. Set the **Message**, inserting the chips from the list:

```
SQL change ready for review. Proof: {{ state.proof_tier }} – {{ state.proof_summary }}
What changed: {{ state.what_changed }}
Personal data: {{ state.touches_personal_data }} {{ state.personal_fields }}
Check: {{ state.risk_notes }}
```

4. **Who may approve:** your team's reviewer emails, or **Reviewers**.
5. **Decide within:** 24 hours.
6. **Advanced settings:**
   - **Run this step only when:** `decision` **is not** `blocked`.
   - For testing only, tick **The person who started the run may approve it**. Untick it later.

---

## Step 7 · Finish step (Calculate values). Don't skip this.

An Approval with a condition cannot be followed directly by END: the test fails even though validation passes (manual known issue 6).

1. Click **+** under `reviewer_approval` and choose **Calculate values**.
2. Name it `finalize`, with description `Record the final outcome.`
3. Set one value: `result` = `state.decision`.
4. Click **Save** (Ctrl+S) and check that the toast says "Saved proofmesh-sql-check".

---

## Step 8 · Guards

Open **Guards** and add these two. Set **on_fail** on purpose.

| Guard | Settings | on_fail |
|---|---|---|
| Required Fields | `old_sql`, `new_sql`, `proof_tier`, `proof_summary` | reject |
| Enum Check | field `proof_tier`, allowed `GREEN, AMBER, RED, OUT` | reject |

---

## Step 9 · Test (Test panel, Ctrl+Shift+T)

**Test A: proven change.** It should pause at approval; approve it and `result` = `ready_for_review`.

```json
{"old_sql": "SELECT first_name, city FROM customer WHERE status = 'A' AND city = 'Pune'",
 "new_sql": "SELECT c.first_name, c.city FROM customer c WHERE c.city = 'Pune' AND c.status = 'A'",
 "proof_tier": "GREEN",
 "proof_summary": "Old and new matched on 12 rows across 2 parameter set(s). Planted mistakes caught: 3 of 3."}
```

**Test B: wrong change.** It should skip approval, and `result` = `blocked`.

```json
{"old_sql": "SELECT first_name FROM customer WHERE status = 'A'",
 "new_sql": "SELECT first_name FROM customer",
 "proof_tier": "RED",
 "proof_summary": "Old and new differed on 2 of 2 trials: the new query returns inactive customers."}
```

**Test C: bad input.** The Enum guard should reject the run.

```json
{"old_sql": "SELECT 1", "new_sql": "SELECT 1", "proof_tier": "SAFE", "proof_summary": "n/a"}
```

After each passing run, click **Snapshot** to save it as a test case.

---

## Step 10 · Golden Set

Open **Test → Golden Set**.
- **Evaluator:** exact match on `decision`.
- **Pass threshold:** 0.9.

Paste these samples:

```json
[
 {"input": {"old_sql": "SELECT a FROM t WHERE x = 1", "new_sql": "SELECT a FROM t WHERE x = 1", "proof_tier": "GREEN", "proof_summary": "matched"}, "expected": {"decision": "ready_for_review"}},
 {"input": {"old_sql": "SELECT a FROM t WHERE x = 1", "new_sql": "SELECT a FROM t", "proof_tier": "RED", "proof_summary": "differed"}, "expected": {"decision": "blocked"}},
 {"input": {"old_sql": "SELECT a FROM t WHERE x = 1", "new_sql": "SELECT a FROM t WHERE x = 1", "proof_tier": "AMBER", "proof_summary": "empty result"}, "expected": {"decision": "needs_person_check"}},
 {"input": {"old_sql": "SELECT name FROM branch", "new_sql": "SELECT name FROM branch", "proof_tier": "OUT", "proof_summary": "not affected"}, "expected": {"decision": "ready_for_review"}}
]
```

The exact JSON shape the Golden Set expects isn't shown in the manual. If it rejects this, click **Snapshot** on Tests A and B instead, and use **Load saved**.

---

## Step 11 · Show it

- **Serve:** Custom WebApp. This is the only generally available option.
- **Deploy → Preview Generated Code → Push to Azure DevOps.** This needs the Deployer role, so ask the platform team.
- **Review:** submit for peer review.

---

## Phase 2 · the full five-agent mesh (after the core works)

Build each agent below as its own small agent, then call them from a `diamond-orchestrator` agent with **Use another agent** steps.

| Agent | Main step | Job |
|---|---|---|
| proofmesh-finder | Ask AI (fields) | List the SQL statements in pasted code, with line numbers |
| proofmesh-spotter | Ask AI (fields) | Flag personal-data columns |
| proofmesh-sql-check | (this pack) | Explain the change, apply the proof rule, get approval |
| proofmesh-checker | Write a message | Write the PR text with the proof |
| proofmesh-onboarder | Ask a person | Collect repo and schema details |

Two known issues with "Use another agent" steps:
- Set the sub-agent's required inputs by hand.
- Remove any input it doesn't accept.

**Replacing the bridge:** ask the platform team: *"How do we add our own MCP server, or a Custom code (custom_python) step, so an agent can run our proof engine?"* Once they answer, add a **Use a tool** step before `decide` that fills `proof_tier` and `proof_summary` automatically.

---

## Final checklist (from manual 17.1)
- [ ] Real name set, and the header no longer says *unsaved*.
- [ ] The banner shows no errors.
- [ ] The prompt comes from the Prompts panel, and its placeholders came from the Insert list.
- [ ] The decision rule is in `decide`, not in the prompt.
- [ ] The Approval has approvers and a deadline, and "starter may approve" is unticked after testing.
- [ ] Governance is Strict.
- [ ] Both guards have an on_fail you chose on purpose.
- [ ] Tests A, B and C behave as expected.
- [ ] The Golden Set passes at 90% or more.
