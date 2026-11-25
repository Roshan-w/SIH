# Agent 6 · diamond-orchestrator

**Job:** run the five agents in order for one SQL change. This is the agent you demo.

```
START
 → onboard    (Use another agent: proofmesh-onboarder)
 → gate_ready (If/then)
 → find       (Use another agent: proofmesh-finder)    only if ready = yes
 → spot       (Use another agent: proofmesh-spotter)   only if ready = yes
 → check      (Use another agent: proofmesh-sql-check) only if ready = yes
 → write_pr   (Use another agent: proofmesh-checker)   only if decision is not blocked
 → finish     (If/then)
 → END
```

**Build it only after agents 1–5 pass their tests.**

---

## Step 1 · Create and name it

| Field | Value |
|---|---|
| Agent Name | `diamond-orchestrator` |
| Vertical | `horizontal` |
| Description | `Runs the five ProofMesh agents in order for one SQL change. AI explains, the proof engine decides, people approve.` |
| Governance Level | **Strict** |
| Framework | LangGraph |

---

## Step 2 · State fields

| Field | Type | Req | Description |
|---|---|---|---|
| `requested_by` | string | ✔ | Developer email. |
| `file_name` | string | ✔ | File being changed. |
| `code_text` | string | ✔ | The code around the change. |
| `old_sql` | string | ✔ | Query before. |
| `new_sql` | string | ✔ | Query after. |
| `proof_tier` | string | ✔ | From the proof engine: GREEN, AMBER, RED or OUT. |
| `proof_summary` | string | ✔ | From the proof engine. |
| `data_owner` | string | | From the onboarder. |
| `reviewer` | string | | From the onboarder. |
| `ready` | string | | From the onboarder. |
| `statements` | list (string) | | From the finder. |
| `personal_fields` | list (string) | | From the spotter. |
| `what_changed` | string | | From sql-check. |
| `decision` | string | | From sql-check. |
| `pr_text` | string | | From the checker. |
| `final_status` | string | | The final answer shown to the user. |

That's 16 fields: 7 required.

---

## Step 3 · Wiring "Use another agent" steps (applies to all 5)

For each sub-agent step:
1. Click **+** and choose **Use another agent**, then pick the agent.
2. **Inputs:** for each of the sub-agent's inputs, pick the orchestrator field it comes from.
3. **Save outputs:** map each output you need back to the orchestrator field with the same name.

Two known issues:
- **Set every required input by hand.** The auto-fill is often wrong.
- **Delete any input the sub-agent doesn't have.** An extra input makes it fail to start.

---

## Step 4 · The steps

### 4.1 `onboard` (Use another agent → proofmesh-onboarder)
| Inputs (sub ← orchestrator) | Save outputs (orchestrator ← sub) |
|---|---|
| `requested_by` ← `requested_by` | `data_owner`, `reviewer`, `ready` |

### 4.2 `gate_ready` (If / then values)
| Rule | Condition | Then set `final_status` = value |
|---|---|---|
| Rule 1 | `ready` · is · value · `no` | `stopped: missing project details` |
| Otherwise | | `in progress` |

### 4.3 `find` (Use another agent → proofmesh-finder)
| Inputs | Save outputs |
|---|---|
| `code_text` ← `code_text`, `file_name` ← `file_name` | `statements` |

Advanced → Run this step only when: `ready` · is · value · `yes`

### 4.4 `spot` (Use another agent → proofmesh-spotter)
| Inputs | Save outputs |
|---|---|
| `statements` ← `statements`, `data_owner` ← `data_owner` | `personal_fields` |

Advanced → Run this step only when: `ready` · is · value · `yes`

### 4.5 `check` (Use another agent → proofmesh-sql-check)
| Inputs | Save outputs |
|---|---|
| `old_sql`, `new_sql`, `proof_tier`, `proof_summary` ← the fields with the same names | `what_changed`, `decision` |

Advanced → Run this step only when: `ready` · is · value · `yes`

### 4.6 `write_pr` (Use another agent → proofmesh-checker)
| Inputs | Save outputs |
|---|---|
| `file_name`, `what_changed`, `proof_tier`, `proof_summary`, `personal_fields`, `decision` ← the fields with the same names | `pr_text` |

Advanced → Run this step only when: `decision` · is not · value · `blocked`

### 4.7 `finish` (If / then values)
This is also the step after the conditional ones, so the flow never ends straight after a conditional step.

| Rule | Condition | Then set `final_status` = value |
|---|---|---|
| Rule 1 | `ready` · is · value · `no` | `stopped: missing project details` |
| Rule 2 | `decision` · is · value · `blocked` | `blocked: proof failed, developer must fix the query` |
| Rule 3 | `decision` · is · value · `needs_person_check` | `approved after person check, PR drafted` |
| Otherwise | | `approved, PR drafted` |

Press **Ctrl+S**.

---

## Step 5 · Guards

| Guard | Settings | on_fail |
|---|---|---|
| Required Fields | `requested_by`, `file_name`, `code_text`, `old_sql`, `new_sql`, `proof_tier`, `proof_summary` | reject |
| Enum Check | `proof_tier` in `GREEN, AMBER, RED, OUT` | reject |

---

## Step 6 · Test

Stay on the VPN for the whole run. Each pause shows in the test panel.

**Test A: GREEN, happy path.**
```json
{"requested_by": "roshan.warade@zensar.com", "file_name": "CustomerDao.java",
 "code_text": "String q = \"SELECT first_name, city FROM customer WHERE status = 'A' AND city = 'Pune'\";",
 "old_sql": "SELECT first_name, city FROM customer WHERE status = 'A' AND city = 'Pune'",
 "new_sql": "SELECT c.first_name, c.city FROM customer c WHERE c.city = 'Pune' AND c.status = 'A'",
 "proof_tier": "GREEN",
 "proof_summary": "Old and new matched on 12 rows across 2 parameter sets. Planted mistakes caught: 3 of 3."}
```
Expect three pauses:
1. The onboarder form: fill it in with your email as data owner and reviewer.
2. The data-owner approval: approve it.
3. The reviewer approval: approve it.

End result: `final_status` = `approved, PR drafted`, and `pr_text` filled.

**Test B: RED, blocked.** Same input, but use the new query and tier below. Expect the form, then the data-owner approval, **no reviewer pause**, an empty `pr_text`, and `final_status` = `blocked: proof failed, developer must fix the query`.
```json
"new_sql": "SELECT first_name, city FROM customer WHERE city = 'Pune'",
"proof_tier": "RED",
"proof_summary": "Old and new differed on 2 of 2 trials: the new query returns inactive customers."
```

**Test C: bad tier.** Set `"proof_tier": "SAFE"`. Expect rejection by the guard before any step runs.

Click **Snapshot** on A and B.

---

## Step 7 · Golden Set
Set the evaluator to exact match on `final_status`, and the threshold to `0.9`. Use **Load saved** with the Snapshots of Tests A and B. Runs with approvals usually can't go in a JSON Golden Set.

---

## Step 8 · Serve, deploy and review
1. **Serve → Custom WebApp.** Copy the link for the demo.
2. **Deploy → Preview Generated Code.** Take a screenshot.
3. **Push to Azure DevOps.** This needs the Deployer role (ask the platform team).
4. **Review:** submit all 6 agents for peer review.
5. Before the demo, untick "starter may approve/answer" in onboarder, spotter and sql-check, and add the real approvers.

---

## Done checklist
- [ ] 16 state fields, 7 required.
- [ ] 7 steps in order, ending with `finish`.
- [ ] Every sub-agent's required inputs set by hand, with no extra inputs.
- [ ] `find`, `spot` and `check` only run when `ready` = yes.
- [ ] `write_pr` only runs when `decision` isn't blocked.
- [ ] Tests A, B and C behave as expected.

## If something goes wrong
| Problem | Fix |
|---|---|
| A sub-agent fails to start | Fix its inputs: set the required ones by hand and remove extras |
| An output is blank in the orchestrator | Map it under Save outputs |
| A paused run vanished | The VPN dropped; run it again |
| `final_status` is "in progress" | `finish` is missing or not last |
| Every step after onboard is skipped | `ready` isn't `yes`; check onboarder rule 2 (see file 05) |
