# Gemma 4 Developer Agent — Competition Entry

A software engineering agent built on Gemma 4 for the Kaggle × Google DeepMind **Gemma 4 Developer Agent Competition** (23 Sep – 2 Dec 2026). The agent navigates a real codebase, localises the code relevant to an issue, drafts a patch, verifies it against the test suite, and submits.

This repository is also an engineering log. Every scaffold change is measured against a fixed evaluation subset and recorded with its effect, including the changes that made nothing better.

> **Status:** in development. Sections marked _(planned)_ aren't built yet. Results tables are placeholders until the runs exist.

---

## The constraints

These are set by the competition and they shape everything below.

- **One base model, fixed:** `gemma-4-31b-it-qat-w4a16-ct`, for every agent and subagent. No model selection, no larger model, no ensemble of different bases.
- **Declarative submission:** an ADK Agent Config tree packaged as `submission.zip` with `agent.yaml` at the archive root.
- **Adapters are per-agent:** PEFT LoRA directories under `adapters/<name>/`, referenced from any `LlmAgent` in `agent.yaml` or `sub_agents/*.yaml`.
- **Sandboxed tools only:** `read_file`, `edit_file`, `write_file`, `run_command`, `submit_patch`, `get_status`, `get_code_neighbors`, `search_similar_code`, `get_code_subgraph`, `run_skill_script`, `load_skill_resource`.
- **Evaluation:** SWE-style PASS/FAIL over real software issues.
- **Compute here:** Kaggle free-tier GPUs only. This is a real constraint and it is documented rather than hidden.

Because the model is fixed, this is not a fine-tuning contest. It is a **scaffold, retrieval, control-flow and budget-management** problem, with LoRA available as a late refinement. The whole approach follows from that.

---

## Approach

### Baseline first

The first submission was the smallest agent that runs: one prompt, no structure, no retrieval strategy. Everything since is measured against it. Nothing is adopted on intuition.

### Localise before reading

The harness provides a code graph. Naive agents wander the file tree, burning budget before they reach the relevant code. This agent queries `get_code_subgraph` and `search_similar_code` to narrow the candidate set first, and only then reads files.

### Two agents, one job each

| Agent | Role | Tools |
|---|---|---|
| `analyzer` | Read-only. Localises the issue, returns candidate locations with reasoning. Exposed as an `AgentTool`. | graph and read tools only |
| `coder` | Edits, verifies, submits. | full tool set |

The separation exists so the localisation step cannot accidentally mutate the repository, and so each half can carry its own adapter.

### Verify before submitting

The agent runs the tests itself before calling `submit_patch`. A failing patch is iterated on, not submitted. This closes the loop rather than submitting blind, and it is the largest single improvement recorded so far _(planned — see results)_.

### Budget discipline

Step and token ceilings per task, a cheap localisation pass before an expensive editing pass, and an explicit rule for when to stop exploring and submit the best patch available.

---

## Repository layout

```
submission/
├── agent.yaml               root coder agent — packaged at the zip root
├── prompts/
│   ├── system.md            root system prompt (versioned)
│   └── analyzer.md
├── sub_agents/
│   └── analyzer.yaml        read-only localisation agent
├── skills/                  helper scripts invoked via run_skill_script
└── adapters/                PEFT LoRA directories (if any survive evaluation)

eval/
├── tasks.json               the fixed evaluation subset — never changed mid-competition
├── runner.py                runs the agent over the subset, writes traces
├── taxonomy.py              classifies each failure
└── traces/                  per-run tool sequences, step counts, token spend

docs/
├── failure-taxonomy.md      the categorisation scheme and baseline distribution
└── paper/                   paper track submission

experiments.md               the log. One row per change. Never skipped.
scripts/pack_submission.sh
Makefile
```

---

## Running it

```bash
make eval            # run the agent over the fixed subset, write traces
make taxonomy        # classify failures, print the distribution
make pack            # build submission.zip
make verify          # assert agent.yaml sits at the archive root
```

Model weights and competition data are **not** in this repository. Download them from the competition page under its terms.

---

## Evaluation method

**A fixed subset**, chosen for variety and held constant for the whole competition, so every score is comparable to every other score. Changing the subset mid-way would make the log worthless.

**Every run is traced**: tools called in order, step count, token spend, stop reason.

**Every failure is categorised:**

| Category | Meaning |
|---|---|
| `NOT_LOCALISED` | Never found the relevant file |
| `WRONG_EDIT_SITE` | Found the file, changed the wrong place |
| `BROKEN_EDIT` | Edit was syntactically invalid |
| `LOOP_BREAKOUT` | Stopped calling tools, started producing prose |
| `BUDGET_EXHAUSTED` | Ran out of steps or tokens |
| `PATCH_REJECTED` | Patch did not apply |
| `TESTS_FAILED` | Applied, tests failed |
| `REGRESSION` | Target test passed, another broke |

The distribution across these categories drives what gets worked on next. Effort goes to the largest category, not the most interesting one.

---

## Results _(placeholder)_

| # | Change | Subset score | Δ | Tokens/task | Notes |
|---|---|---|---|---|---|
| 1 | Baseline, minimal prompt | — | — | — | |
| 2 | Explicit tool-call contract | — | — | — | |
| 3 | Graph localisation before read | — | — | — | |
| 4 | Test verification before submit | — | — | — | |
| 5 | Analyzer/coder split | — | — | — | |

Full log in [`experiments.md`](experiments.md), including changes that produced no gain. Those are recorded deliberately: a change that leaves the score flat while cutting cost by a third is a result, and so is a good idea that turned out not to work.

---

## Design decisions

1. **Scaffold before post-training.** With a fixed base model, prompt, retrieval and control-flow changes have a far better return per hour than LoRA. Adapters are only introduced once scaffold gains have been measured and exhausted.
2. **A read-only analyser.** Localisation should be incapable of mutating the repository. Enforced structurally, not by instruction.
3. **Self-verification before submission.** Running the tests converts a guess into a checked answer.
4. **Graph retrieval before file reading.** The harness provides a code graph; using it is cheaper than exploring.
5. **A fixed evaluation subset.** Comparability across the whole competition matters more than subset size.
6. **Adapters must earn their place.** A LoRA that does not beat the best non-adapter configuration is not shipped.

---

## Compute notes

Everything here runs on free-tier Kaggle GPUs. Measured load time, throughput and workable context for `gemma-4-31b-it-qat-w4a16-ct` under that allocation are documented in [`docs/compute.md`](docs/compute.md) _(planned)_, along with what it implies for the feasibility of local post-training.

This is written down because it's genuinely useful to anyone attempting the competition without lab resources, and because published results rarely mention the hardware floor.

---

## Paper track

An accompanying write-up covering the method, the failure taxonomy, the ablation results and the free-tier compute constraints is prepared for the competition's paper track (deadline 12 November 2026). Draft in [`docs/paper/`](docs/paper/) _(planned)_.

---

## Rules and licensing

Entry is subject to the competition rules, which must be accepted on Kaggle before the entry deadline of 25 November 2026. Model weights, competition datasets and harness libraries are distributed under their own terms and are not redistributed here.

Code in this repository: MIT.