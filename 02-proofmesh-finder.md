# Agent 2 · proofmesh-finder

**Job:** read pasted code (SQL, Java, Python or COBOL) and list every SQL statement with its line number, the tables it uses, a count and a short summary. It doesn't change or judge the SQL.

```
START → find_sql (Ask AI) → END
```

**Status of your export:** ✅ correct. Nothing to change.

---

## Step 1 · Create and name it

1. Click **New Agent → Blank agent**.
2. Fill in **Meta**:

| Field | Value |
|---|---|
| Agent Name | `proofmesh-finder` |
| Vertical | `horizontal` |
| Description | `Lists every SQL statement in pasted code, with its line number and a one-line meaning.` |
| Governance Level | **Strict** |
| Framework | LangGraph |

3. Press **Ctrl+S**.

---

## Step 2 · State fields

| Field | Type | Req | Description |
|---|---|---|---|
| `code_text` | string | ✔ | Source code pasted by the developer (SQL, Java, Python or COBOL). |
| `file_name` | string | ✔ | Name of the file the code came from. |
| `statements` | list (string) | | Each SQL statement found, as "line N: SQL". Filled by the AI. |
| `statement_count` | int | | How many SQL statements were found. Filled by the AI. |
| `tables_used` | list (string) | | Every table name used. Filled by the AI. |
| `summary` | string | | One paragraph on what this code does with the database. Filled by the AI. |

That's 6 fields: 2 required.

---

## Step 3 · Prompt

**Prompts** → key `finder_prompt` → **+**. Insert the placeholders from **Insert**.

```
You are a code reader. Find every SQL statement in the code below, including SQL
inside strings in Java, Python or COBOL.

For each one, give "line N: <the SQL>". Then list every table used, give the total
count, and write one short paragraph on what this code does with the database.
Do not change or judge the SQL. If you find none, return an empty list and count 0.

FILE: {{ state.file_name }}

CODE:
{{ state.code_text }}
```

---

## Step 4 · `find_sql` (Ask AI, fields)

1. Click **+** between START and END and choose **Ask AI (fields)**.
2. Set:

| Setting | Value |
|---|---|
| Step name | `find_sql` |
| What this step does | `AI finds every SQL statement in the code.` |
| Instructions for the AI | `finder_prompt` |
| AI model | Auto |

3. **What the AI must answer with:**

| Name | Type | Description |
|---|---|---|
| `statements` | List (text) | Each SQL statement as "line N: SQL". |
| `statement_count` | Number | Total SQL statements found. |
| `tables_used` | List (text) | Every table name used. |
| `summary` | Text | What the code does with the database. |

4. **Save parts of the answer:** map each one to the state field with the same name (4 mappings).
5. Press **Ctrl+S**.

No guard is needed; the two required fields are checked automatically.

---

## Step 5 · Test

**Test A: Java with 2 queries.** Expect `statement_count` = 2, `tables_used` to contain `customer`, and 2 lines in `statements`.
```json
{"file_name": "CustomerDao.java",
 "code_text": "public List<Customer> active() {\n  String q = \"SELECT first_name, city FROM customer WHERE status = 'A'\";\n  return jdbc.query(q);\n}\npublic void close(int id) {\n  jdbc.update(\"UPDATE customer SET status = 'C' WHERE id = ?\", id);\n}"}
```

**Test B: no SQL.** Expect `statement_count` = 0 and `statements` = [].
```json
{"file_name": "MathUtil.java", "code_text": "public int add(int a, int b) { return a + b; }"}
```

**Test C: plain SQL file with a join.** Expect a count of 2, and `tables_used` to contain `orders` and `customer`.
```json
{"file_name": "report.sql",
 "code_text": "SELECT o.id, c.first_name FROM orders o JOIN customer c ON c.id = o.customer_id;\nDELETE FROM orders WHERE created_at < '2020-01-01';"}
```

Click **Snapshot** on each passing run.

---

## Step 6 · Golden Set

Set the evaluator to exact match on `statement_count`, and the threshold to `0.9`.
```json
[
 {"input": {"file_name": "a.sql", "code_text": "SELECT 1 FROM t;\nSELECT 2 FROM u;"}, "expected": {"statement_count": 2}},
 {"input": {"file_name": "b.java", "code_text": "int x = 1;"}, "expected": {"statement_count": 0}}
]
```

---

## Done checklist
- [ ] Name set and saved.
- [ ] 6 state fields, 2 required.
- [ ] 1 step with 4 answers mapped.
- [ ] Tests A–C pass.

## If something goes wrong
| Problem | Fix |
|---|---|
| `statement_count` is 2.0 rather than 2 | Fine. The AI answers with a number and the state field stores it as int |
| `statements` is empty for Java | Make sure the code was pasted with `\n` line breaks, as in Test A |
| The AI rewrites the SQL | Check that the prompt line "Do not change or judge the SQL" is there |
