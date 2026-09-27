# Gemma 4 Developer Agent Competition — Working Plan

Nine and a half weeks, 5 to 10 hours each, solo, on Kaggle free GPUs. Roughly 50 to 95 hours total. This plan is built around that budget, not around what you'd do with a lab.

## Fixed dates

| Date | What |
|---|---|
| **12 Nov** | Paper track deadline |
| **25 Nov** | Entry and team merger deadline — rules must be accepted by now |
| **2 Dec, 23:59 UTC** | Final submission |

---

## Five principles

**1. Baseline before cleverness.** Get the dumbest possible agent scored first. Every later change is judged against that number, not against your intuition.

**2. One variable at a time, with receipts.** Every experiment gets a row in the log: what changed, score before, score after, cost. If you can't attribute a gain, you haven't learned anything and you can't write it up.

**3. The model is fixed, so the scaffold is the lever.** Everyone has the same weights. Prompts, tool use, control flow and budget management are where the differences live. LoRA is a late refinement, not the strategy.

**4. Submit early, submit often.** A mediocre submission on the board beats a brilliant one that misses a packaging deadline. Package and submit in week 1 with something trivial, so you find the format problems while they're cheap.

**5. Fail fast on compute.** The single biggest risk is that a 31B model doesn't fit your quota. Find that out in week 1, not week six.

---

## Week 1 (27 Sep – 4 Oct) — Feasibility and a score on the board

This week is a go/no-go, not a build week.

- [ ] **Accept the competition rules on Kaggle.** Costs nothing, preserves the option, and it's a hard gate you cannot fix later.
- [ ] Read the competition page and `HARNESS_README` properly. Write down every tool signature and every budget limit: `read_file`, `edit_file`, `write_file`, `run_command`, `submit_patch`, `get_status`, `get_code_neighbors`, `search_similar_code`, `get_code_subgraph`, `run_skill_script`, `load_skill_resource`.
- [ ] Download the competition dataset: graphs, embeddings, snapshots, harness libraries.
- [ ] **The compute spike.** Load `gemma-4-31b-it-qat-w4a16-ct` on your Kaggle allocation and run inference. Record: does it load at all, VRAM used, tokens per second, maximum workable context, and how much of your weekly GPU quota one evaluation run consumes.
- [ ] Ship a trivial `agent.yaml` with a minimal system prompt, package it, verify `agent.yaml` sits at the zip root, and submit.

**Gate at end of week 1.**

- Inference works within quota → continue as planned.
- Inference works but is painfully slow → continue, but cut your local evaluation subset hard and assume LoRA is off the table.
- Inference doesn't fit → switch to the fallback in the last section. Do not spend week 2 fighting it.

---

## Week 2 (5 – 11 Oct) — Instrumentation, which is the real project

You cannot improve what you cannot see. This week produces the thing that drives every subsequent decision.

- [ ] Build a **local evaluation loop** over a fixed subset of tasks — 20 to 30, chosen for variety, kept constant all competition so scores stay comparable.
- [ ] **Trace every run.** For each task record: tools called in order, number of steps, tokens spent, where it stopped, and why.
- [ ] Build the **failure taxonomy**. Categorise every failed task:
  - Never found the relevant file
  - Found the file, edited the wrong place
  - Edit was syntactically broken
  - Broke out of the loop, stopped calling tools, started chatting
  - Exhausted the step or token budget
  - Patch didn't apply
  - Patch applied, tests failed
  - Patch applied, tests passed, but something else regressed
- [ ] Record the baseline distribution across those categories.

**This taxonomy is the most valuable artifact you will produce.** It tells you what to fix, in what order, and it is the backbone of the paper.

---

## Weeks 3 – 4 (12 – 25 Oct) — Scaffold, the highest-return work

Attack the largest failure category first. Do not attack what's interesting.

**If the dominant failure is localisation** (never found the right file), work on retrieval: use `get_code_subgraph` and `search_similar_code` to localise *before* reading files, rather than wandering the tree. The harness hands you a code graph; most entrants will underuse it.

**If the dominant failure is loop breakout**, work on the system prompt and tool descriptions. Tighten the output contract, add explicit "you must call a tool" instructions, shorten the reasoning budget per step, consider a forced-tool-call first step.

**If the dominant failure is budget exhaustion**, work on control flow: a cheap localisation pass before an expensive editing pass, and a rule for when to stop exploring and submit what you have.

**If the dominant failure is broken edits**, work on verification: read back after every edit, run a syntax check before proceeding.

Discipline for the whole fortnight: one change, one evaluation run, one log row. If a change doesn't move the number, revert it and record that it didn't. Negative results are paper content.

---

## Week 5 (26 Oct – 1 Nov) — Subagents and verification

- [ ] **Split the agent.** A read-only analyser exposed as an `AgentTool` for localisation, and a coder agent that edits. The config format supports different adapters per agent, so this structure also sets up week 8.
- [ ] **Add a verification step** before `submit_patch`: run the tests yourself, and if they fail, iterate rather than submitting. This is usually one of the largest single gains available, because it converts blind submission into a closed loop.
- [ ] **Custom skills** via `run_skill_script` for anything mechanical and repeated — applying a diff safely, running a targeted test subset, checking syntax.
- [ ] Full evaluation run. Compare against the week 2 baseline and write down the delta by failure category.

---

## Weeks 6 – 7 (2 – 15 Nov) — The paper, due 12 November

Start drafting in week 6, submit in week 7 with days to spare.

The paper writes itself from work you've already done:

- **Method**: your scaffold architecture and why each piece exists.
- **The failure taxonomy**: the categorisation scheme and the baseline distribution. This is a contribution in its own right and fits the "new resource" category.
- **Ablations**: your experiment log, turned into a table. Each scaffold change, its effect, and your interpretation — including the ones that didn't work.
- **Constraints**: what it's like to do this on free-tier compute. Genuinely useful to the community and almost nobody writes it down.

If you release the evaluation subset and taxonomy publicly alongside it, you're squarely in the "new resource" track rather than competing on raw leaderboard position.

**Do not skip this because the leaderboard feels more urgent.** The paper is the highest expected-value part of this project for you: it's marginal effort on top of work already done, the field is small, and a public technical write-up with your name on it is a durable CV artifact whatever the placing.

---

## Week 8 (16 – 22 Nov) — LoRA, if and only if compute allows

Now, and not before. If the week 1 spike said training isn't feasible, skip this entirely and spend the week on scaffold refinement instead — that's a legitimate outcome, not a failure.

If it is feasible, the highest-value target is not "make it better at coding". It's **format and loop compliance**: train on trajectories where the agent stayed in the loop and emitted valid tool calls. That's the cheapest measurable win, because it attacks the breakout failure category directly.

- [ ] Collect trajectories from your successful runs as training data.
- [ ] Train a small LoRA adapter. Keep rank low; you're not teaching it to code, you're teaching it to behave.
- [ ] Place it under `adapters/<name>/` and reference it from the relevant agent in the config.
- [ ] Evaluate against your best non-LoRA configuration. **If it doesn't beat it, don't ship it.** An adapter that doesn't help is a liability.

---

## Week 9 (23 – 29 Nov) — Freeze and package

**Entry deadline is 25 November. Confirm rules acceptance on day one of this week.**

- [ ] Freeze the configuration. No new ideas after Wednesday.
- [ ] Full packaging run: `submission.zip` with `agent.yaml` at the archive root, not nested in a directory. Adapters in the right place with both `adapter_config.json` and `adapter_model.safetensors`.
- [ ] Submit the known-good version now, so a broken final week cannot cost you an entry.
- [ ] Final full evaluation run, results recorded.

---

## Week 10 (30 Nov – 2 Dec) — Submit

- [ ] Final submission before 2 December, 23:59 UTC. Do it on the 1st.
- [ ] Publish the repository: README explaining the architecture, the failure taxonomy, the experiment log, and honest results.
- [ ] Write the short public post about what you learned. This is the piece that reaches recruiters.

---

## The experiment log

One file, `experiments.md`, appended to after every run. Never skipped.

```
| # | Date | Change | Subset score | Δ | Tokens/task | Notes |
|---|------|--------|--------------|---|-------------|-------|
| 1 | 05/10 | Baseline, minimal prompt | 4/25 | — | 18k | Loop breakout on 11 |
| 2 | 08/10 | Explicit tool-call contract | 7/25 | +3 | 19k | Breakout 11 → 4 |
| 3 | 11/10 | Graph localisation before read | 7/25 | 0 | 12k | No gain, 33% cheaper. Keep |
```

Row 3 is the kind that matters. No score change, a third of the cost, and it's a paper result.

---

## If you fall behind

Cut in this order, and don't feel bad about it:

1. **LoRA goes first.** It's the smallest expected gain per hour.
2. **Subagents go second.** A single well-prompted agent with verification beats a badly coordinated crew.
3. **Leaderboard optimisation goes third.**

What survives to the end no matter what: a working submission on the board, the failure taxonomy, the experiment log, and the paper. That set is the whole value of this project for you, and none of it depends on placing.

---

## What you get out of it regardless

- Hands-on agent scaffolding under a fixed, unforgiving evaluation. This transfers directly to the agent layer of your CodeReview project — the harness even hands you code-graph tools that mirror what you planned to build with Neo4j.
- A measured, documented engineering process with attributable results, which is exactly the evidence AI engineering interviews probe for.
- A public repository and a paper, both of which outlive the competition.
- An honest answer to "tell me about a time you worked under a hard deadline with constraints you couldn't change."

The placing is the least valuable thing here. Optimise for the artifacts.