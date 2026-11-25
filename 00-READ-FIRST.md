# ProofMesh agents: read this first

There's one file per agent. Build them in this order:

| # | File | Agent | Main job | Time |
|---|---|---|---|---|
| 1 | `01-proofmesh-sql-check.md` | proofmesh-sql-check | Explains the change, applies the proof rule, gets reviewer approval | 60 min |
| 2 | `02-proofmesh-finder.md` | proofmesh-finder | Finds every SQL statement in code | 20 min |
| 3 | `03-proofmesh-spotter.md` | proofmesh-spotter | Flags personal data; the data owner confirms | 30 min |
| 4 | `04-proofmesh-checker.md` | proofmesh-checker | Writes the PR text with the proof | 15 min |
| 5 | `05-proofmesh-onboarder.md` | proofmesh-onboarder | Collects repo, DB type, owner and reviewer | 30 min |
| 6 | `06-diamond-orchestrator.md` | diamond-orchestrator | Runs all five in order | 60 min |

## Rules for every agent

1. **Name the agent first.** Open Meta and set the Agent Name before anything else, because the Builder auto-saves under whatever name it has.
2. **Use the Insert list for placeholders.** Never type `{{ state.x }}` by hand; a misspelled placeholder becomes blank with no error.
3. **Typed text in a Calculate values step needs quotes.** For example `'done'`. Without quotes, the Builder reads the word as a field name ("unknown name 'done'").
4. **In If/then and Then set, choose "value" from the dropdown** and type the text with no quotes. "From a state field" copies another field instead.
5. **Never put END straight after a conditional Approval.** Always add a step in between.
6. **Use the condition builder, not "Formula (advanced)".** Never leave a rule's condition blank.
7. **Stay on the VPN while a run is paused** for approval. If it drops, the run is lost.
8. **For testing**, tick "The person who started the run may approve/answer". Untick it before the demo.
9. **Save with Ctrl+S** and check that the toast says "Saved <agent-name>".
10. **Delete fields the Builder adds by itself** that you don't use, such as `result`, `field_2` or `render_result`. They're harmless, but reviewers notice them.

## Words used in these files

| Here | On screen |
|---|---|
| Ask AI step | **Ask AI (fields)**: the AI fills in fields you define |
| Rule step | **If / then values** |
| Formula step | **Calculate values** |
| Approval | **Approval**: the run pauses until a person approves or rejects |
| Form | **Ask a person**: the run pauses until a person fills in a form |
| Template | **Write a message**: fills in a template, with no AI |
| Sub-agent | **Use another agent** |
