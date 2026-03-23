# Andrej Karpathy's Autoresearch — As a Universal AI Skill

A self-improving prompt optimization skill for Cursor and Claude Code, derived from [Andrej Karpathy's autoresearch](https://github.com/karpathy/autoresearch) pattern. Drop it into any repository, and it scans your codebase, suggests what to optimize, defines evaluation metrics, and runs an autonomous loop that makes your prompts better over time — without you lifting a finger.

---

## How Andrej Karpathy's Autoresearch Works

In early 2026, Andrej Karpathy (former founding member of OpenAI, former head of AI at Tesla) released a small GitHub repo called **autoresearch**. The idea was deceptively simple:

> Let an AI agent autonomously run experiments, measure results, keep what works, throw away what doesn't, and repeat — forever.

In Karpathy's case, the "experiment" was training a small language model (nanoGPT). The agent would modify the training code (`train.py`), run it for 5 minutes, measure the result (a number called `val_bpb` — lower is better), and decide: did the change help? If yes, keep it. If no, revert and try something else.

The repo had only three files that mattered:

- **`program.md`** — instructions for the AI agent (what to do, how to measure, how to log)
- **`train.py`** — the thing being optimized (the agent modifies this)
- **`prepare.py`** — the measurement tool (read-only, the agent cannot touch it)

The loop looked like this:

```
1. Read the current code
2. Make a change (an "experiment")
3. Run the experiment
4. Measure the result (a hard number, not vibes)
5. If better → keep the change
   If worse  → throw it away, revert
6. Go to step 1. Never stop.
```

That's it. Evolution applied to code. Generate variations, test them against reality, keep the fittest, kill the rest.

### Why It Works

Three ingredients make autoresearch work:

1. **An objective metric** — a number you can measure. Not "feels faster" or "looks better." An actual number. In Karpathy's case: `val_bpb` (bits per byte on a validation set).

2. **An automated measurement tool** — something that produces that number without a human in the loop. In Karpathy's case: the `evaluate_bpb()` function in `prepare.py`.

3. **Something to change** — the thing being optimized. In Karpathy's case: `train.py`. In our case: a prompt.

If you have all three, you can run the loop forever and the thing will get better over time. The agent does the work while you sleep.

---

## From Autoresearch to a Universal Skill

A YouTuber took Karpathy's pattern and asked: what if, instead of optimizing ML training code, we optimize **prompts**?

He built a Claude Code skill that optimized a diagram-generation prompt. The skill would:

1. Generate 10 diagrams using the current prompt (via Gemini image gen)
2. Evaluate each diagram against 4 yes/no criteria using Claude's vision (Is text legible? Are colors pastel? Is layout linear? Are there numbers?)
3. Score the batch (10 diagrams x 4 criteria = max 40 points)
4. Keep the prompt if the score improved, discard if it didn't
5. Mutate the winning prompt to try to fix remaining failures
6. Repeat every 2 minutes

He started at 32/40 and hit 40/40 (perfect) by run 6 — about 12 minutes of unattended operation.

**This skill takes that same idea and makes it universal.** Instead of being hardcoded to diagrams, it adapts to any repository. Drop it into a Python API, a React frontend, a docs site, a SQL project, a CLI tool — it scans the repo, figures out what can be optimized, defines the right metrics, and runs the loop.

---

## What's in This Repo

```
Andrej-Karpathy-s-Autoresearch-As-a-Universal-Skill/
├── README.md                        ← You are here
├── ARTICLE.md                       ← Full story of how this skill was built
└── autoresearch-universal/
    └── SKILL.md                     ← The skill file (works with Cursor and Claude Code)
```

---

## Installation

### For Cursor

```bash
mkdir -p ~/.cursor/skills/autoresearch-universal
cp autoresearch-universal/SKILL.md ~/.cursor/skills/autoresearch-universal/SKILL.md
```

### For Claude Code

```bash
mkdir -p ~/.claude/skills/autoresearch-universal
cp autoresearch-universal/SKILL.md ~/.claude/skills/autoresearch-universal/SKILL.md
```

Once installed, the skill is available globally across all your projects.

---

## How to Use It

### Step 1: Switch to Plan Mode

**Start in Plan mode.** The skill will ask you to switch if you aren't already. This ensures the repo scan and planning happen read-only — nothing is created or modified until you approve.

### Step 2: Invoke the Skill

Open any repository and say something like:

- "Run autoresearch"
- "Optimize my test cases using autoresearch"
- "What can autoresearch improve in this repo?"

### Step 3: Review the Plan

The agent will:

1. **Scan your repo** — identify languages, frameworks, purpose, existing quality tools
2. **Present an optimization template** — asking what you want to optimize (target, scope, context). If you don't have a goal in mind, say "suggest" and it will generate 5-8 tailored suggestions.
3. **Define binary eval criteria** — 4-6 yes/no questions based on industry best practices

You review everything. Adjust targets, tweak metrics, or provide your own goal entirely.

### Step 4: Switch to Agent Mode and Go

Once you're happy with the plan, switch to Agent mode. The agent will:

1. Create an `.autoresearch/` directory in your repo
2. Write the initial prompt
3. Run a baseline cycle to establish a starting score
4. Enter the autonomous loop — generate, evaluate, score, keep/discard, mutate, repeat

You can walk away. Come back later to see how far it's progressed.

### Step 5: Check Results

Results are logged to `.autoresearch/results.jsonl`. The agent prints a summary after each cycle. When you're done, check `.autoresearch/best_prompt.txt` for the optimized prompt.

---

## Files the Skill Creates (and Why)

When the loop starts, the skill creates an `.autoresearch/` directory in your repo root:

```
.autoresearch/
├── prompt.txt          Current prompt being tested this cycle
├── best_prompt.txt     Best-scoring prompt found so far (the winner)
├── state.json          Loop state — scores, run count, validation set, sample tracking
└── results.jsonl       Append-only log of every cycle's results
```

### `prompt.txt`

The prompt currently being optimized. This changes every cycle — the agent mutates it after each run. If a mutation makes things worse, this file gets reverted to `best_prompt.txt`.

### `best_prompt.txt`

The highest-scoring prompt found so far. This only gets overwritten when a new prompt beats the current best by a confidence margin. This is your final output — the optimized prompt.

### `state.json`

Tracks everything the loop needs across cycles:

```json
{
  "best_score": 28,
  "best_validation_score": 16,
  "run_number": 7,
  "target": "docstring completeness",
  "scope": "public API functions",
  "context": "Google-style docstrings, NumPy project",
  "max_score": 40,
  "criteria_count": 4,
  "batch_size": 10,
  "validation_items": ["src/auth.py", "src/db.py", "src/api.py"],
  "sampled_items": ["src/utils.py", "src/models.py", "src/routes.py", "..."],
  "item_failures": {"src/legacy.py:has_docstring": 6},
  "plateau_counter": 0
}
```

Key fields:

- `validation_items` — 3-5 fixed files that appear in every cycle for fair comparison
- `sampled_items` — tracks which files have been tested, ensuring full repo coverage before repeats
- `item_failures` — detects items that always fail (broken items, not bad prompts)
- `plateau_counter` — triggers a "plateau break" (complete prompt rewrite) after 5 stale runs

### `results.jsonl`

One JSON line per cycle. Contains scores, per-criterion breakdowns, the full prompt text used, failure reasons, which mutation operator was applied, and any flags. This is your experiment history — every run is recorded.

Example entry:

```json
{
  "run": 3,
  "timestamp": "2026-03-21T14:30:00Z",
  "score": 34,
  "validation_score": 18,
  "max": 40,
  "criteria": {"has_docstring": 9, "describes_params": 8, "includes_return_type": 7, "has_example": 10},
  "status": "keep",
  "mutation_operator": "tighten_language",
  "prompt_len": 342,
  "prompt_text": "Generate a complete docstring for the following function...",
  "failures": ["src/utils.py: missing return type description", "src/auth.py: params not documented"],
  "items_flagged": [],
  "flaky_commands": []
}
```

---

## Developer Use Case: Front-End Developer

You're a front-end developer working on a React component library. Your components work fine, but accessibility is inconsistent — some have ARIA labels, some don't; some handle keyboard navigation, some don't.

### Invoke

```
Run autoresearch to optimize accessibility in my React components
```

### The agent scans and proposes

```
Repo: my-component-library
Stack: TypeScript, React, Storybook, Jest
Purpose: Shared UI component library
Quality tools: ESLint, Prettier, Jest, axe-core

Eval criteria for accessibility compliance (8 items x 5 criteria = max 40):

1. Does the component have an aria-label or aria-labelledby? — yes/no — command: grep -q "aria-label" [file]
2. Is the component keyboard-navigable (handles onKeyDown)? — yes/no — llm-judge
3. Does the component use semantic HTML (button, nav, main) instead of generic divs? — yes/no — llm-judge
4. Does the component have a role attribute where semantic HTML isn't possible? — yes/no — command: grep -q "role=" [file]
5. Does the component pass axe-core automated checks? — yes/no — command: npx jest --testPathPattern=a11y [component]
```

### The loop runs

- **Run 1 (baseline):** 22/40 — most components lack keyboard handling and semantic HTML
- **Run 3:** 28/40 — mutation added explicit "MUST use `<button>` instead of `<div onClick>`" rule
- **Run 5:** 33/40 — counterexample mutation showed a before/after of div-to-semantic conversion
- **Run 8:** 38/40 — restructured prompt moved keyboard navigation rules to the top
- **Run 12:** 40/40 — perfect score, 3 consecutive runs

You come back to find a battle-tested prompt in `.autoresearch/best_prompt.txt` that reliably produces accessible component patterns.

---

## Skill Best Practices

### 1. Start in Plan Mode

Always begin in Plan mode. Let the agent scan your repo and present its plan before anything is modified. Review the suggested targets and metrics. Adjust if needed. Only switch to Agent mode when you're satisfied.

### 2. Keep Evals Binary

The skill enforces yes/no criteria for a reason. Scales (rate 1-10, score out of 7) introduce noise — the model gives inconsistent scores across runs, making it impossible to tell if a prompt actually improved. Binary questions are stable and reproducible.

### 3. Trust the Loop, Don't Micromanage

Once the loop starts, let it run. Don't interrupt every cycle to adjust things. The skill is designed to be autonomous — it handles plateaus, detects broken items, rotates mutation strategies, and re-evaluates its own judgments. Check in after 10-15 runs to see the trajectory.

### 4. Start With 4 Criteria, Not 10

More criteria means more noise and more chances for the prompt to game individual checks without improving overall quality. Start with 4 well-chosen binary questions. You can add more after the first pass if needed.

### 5. Prefer Command Evals Over LLM Judgment

Wherever a shell command can check something (linting, compilation, grep for a pattern, test execution), use it. Command evals are deterministic and honest. LLM self-judgment has inherent bias — the skill mitigates this with eval isolation and adversarial re-eval, but a real command is always more trustworthy.

### 6. Check the Criteria Health Flags

At run 10, the skill flags criteria that are too easy (always 100%) or too hard (never above 20%). Pay attention to these. A criterion that's always passing isn't helping. A criterion that never passes might need rewording or might require actual code changes rather than prompt optimization.

### 7. Review the Mutation Operator Log

The final summary ranks mutation operators by effectiveness (KEEP rate). If "add counterexample" produced 3 out of 4 KEEPs while "remove bloat" produced 0, that tells you something about what your specific prompt responds to. Use this data.

### 8. The `.autoresearch/` Directory Is Your Lab Notebook

Don't delete it after a run. The `results.jsonl` file is a complete experiment history — every prompt version, every score, every failure. If you want to pick up where you left off, or try a different target, the history is there. Add `.autoresearch/` to `.gitignore` (the skill does this automatically) so it doesn't pollute your repo.

---

## License

This skill is provided as-is. Derived from the concepts in [Andrej Karpathy's autoresearch](https://github.com/karpathy/autoresearch). Use it, modify it, share it.
