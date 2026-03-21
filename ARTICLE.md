# I Turned Andrej Karpathy's Autoresearch Into a Universal AI Skill — Here's How

*By Balasubramanyam*

---

I'm a technical writer. I spend my days in documentation repos — Markdown files, API references, style guides, SEO audits. I don't train language models. I don't write CUDA kernels. But when Andrej Karpathy dropped his autoresearch repo in early 2026, I couldn't stop thinking about it.

The idea was so simple it felt obvious in hindsight: let an AI agent run experiments on its own, measure the results, keep what works, throw away what doesn't, and repeat until it's good. Karpathy was using it to train a small GPT model. The agent would modify the training code, run it for 5 minutes, check if the model got better, and decide — keep or revert. Then try something else. Over and over, autonomously, while he slept.

I remember reading the repo and thinking: this isn't just for machine learning. This pattern — generate, evaluate, keep the winner, mutate, repeat — is universal. It's evolution. It works on anything you can measure.

But I didn't know how to apply it to my world. Not yet.

---

## The Video That Connected the Dots

A few days after Karpathy's repo went public, I stumbled on a YouTube video that changed how I thought about it. The creator (who builds Claude Code skills for a living) had taken Karpathy's exact pattern and applied it to something completely different: optimizing a text-to-image prompt for generating whiteboard-style diagrams.

His setup was elegant:

- **The thing being optimized**: a prompt sent to Gemini's image generation model
- **The measurement**: Claude's vision API evaluating each generated diagram against 4 binary criteria
- **The loop**: generate 10 diagrams, grade them, keep the prompt if it scores higher, mutate it to fix failures, repeat every 2 minutes

His 4 criteria were dead simple:

1. Is all the text legible and grammatically correct? Yes/No
2. Are the colors soft pastels? Yes/No
3. Is the layout linear (left-to-right or top-to-bottom)? Yes/No
4. Are there any numbers or ordinals? Yes/No

10 diagrams, 4 questions each, max score of 40. He started at 32/40. By run 6 — about 12 minutes later — he hit 40/40. Perfect score.

What struck me wasn't the diagrams. It was the realization that the three ingredients Karpathy needed — an objective metric, an automated measurement tool, and something to change — mapped perfectly to prompts:

| Karpathy's Autoresearch | Prompt Optimization |
|---|---|
| `train.py` (code being modified) | `prompt.txt` (prompt being modified) |
| `val_bpb` (objective number) | Eval score out of 40 (objective number) |
| `evaluate_bpb()` (automated test) | Claude vision grading against yes/no criteria |
| Git keep/revert | Keep best prompt / revert to best |

The mapping was perfect. And if it worked for diagrams, it could work for anything with a measurable output. Including documentation.

---

## Vibe-Coding It Into a Skill

I decided to build it. Not just for diagrams — for everything. A universal autoresearch skill that could adapt to any repository, any tech stack, any optimization goal.

I opened Cursor, pulled up the Karpathy repo, read through the three files that mattered (`program.md`, `train.py`, `prepare.py`), rewatched the video, and started prompting.

The first thing I got the AI to do was understand the pattern deeply. Not just "what files are in the repo" but the actual mechanics — the loop, the keep/discard logic, the mutation step, why binary evals matter, why you mutate from the best prompt and not the latest failed one.

Then I said: build me a skill that does this for any repo.

### The First Version

The v1 skill had 5 phases:

1. **Repo Discovery** — scan the codebase, identify languages, frameworks, purpose
2. **Target Suggestions** — based on the scan, suggest what could be optimized (test quality, doc completeness, accessibility, SQL patterns — whatever fit the repo)
3. **Metric Definition** — auto-generate 4-6 binary yes/no eval criteria from industry best practices
4. **Baseline** — run the prompt once to establish a starting score
5. **Autoresearch Loop** — generate, evaluate, score, keep/discard, mutate, repeat autonomously

It worked. But it had gaps.

### The Critique That Made It Better

I ran the v1 skill through a rigorous analysis. The feedback was surgical:

**"The mutation strategy is underspecified."** Telling the agent to "analyze failures and rewrite the prompt" is too vague. The fix: define 6 explicit mutation operators — add a constraint, add a negative example, restructure the prompt, tighten vague language, remove bloat, add a counterexample. Rotate through them so each gets tried. Log which one was used so you can see which mutation types are most productive.

**"No validation set."** If you pick different items each cycle, score differences might reflect item difficulty rather than prompt quality. The fix: designate 3-5 fixed items that appear in every single cycle. Compare scores on this fixed set for apples-to-apples evaluation. Rotate the rest for coverage.

**"The eval step conflates generator and judge."** Claude writes the prompt, generates outputs from it, and then judges those outputs — all in the same conversation. By the time it evaluates, it knows what the output was *trying* to do and grades charitably. The fix: evaluate in isolation. Present only the raw output and the criterion text. No prompt context. Judge as if seeing it for the first time.

**"No handling of context window limits."** Over 20+ cycles, the conversation grows enormous. Early-cycle details fall off. The mutation step — which depends on understanding failure patterns — degrades silently. The fix: re-read all state from disk at the start of every cycle. Files are the source of truth, not conversational memory.

**"Sample variance isn't tracked."** "Vary the sample" with no memory of what was already sampled means you could run 20 cycles and accidentally never hit a particular file. The fix: track sampled items in `state.json` and do coverage-first selection — prefer untested items, only repeat after full coverage.

Each critique led to a specific, targeted fix. The skill got tighter with each round.

### What the Final Version Includes

The skill that emerged from this process has features the original diagram skill never had:

- **Eval isolation** — outputs are judged without knowledge of the prompt that produced them
- **Validation set** — fixed items for consistent comparison, rotating items for coverage
- **Structured mutation operators** — 6 named strategies, rotated and logged
- **Sample tracking** — coverage-first selection, no accidental gaps
- **Context window management** — re-reads state from disk every cycle
- **Command evals** — shell commands for deterministic checks (linting, compilation, grep) alongside LLM judgment
- **Item-level failure detection** — flags items that always fail (broken items, not bad prompts)
- **Adversarial re-eval** — every cycle, re-checks passing outputs with skeptical prompting
- **Criteria health check** — at run 10, flags criteria that are too easy or too hard
- **Plateau breaker** — after 5 stale runs, throws away the prompt structure and writes a fresh one from scratch using only the criteria and failure history
- **Confidence margin** — prevents noise from being mistaken for progress on small batches
- **Plan mode start** — scans and plans read-only before touching anything

---

## A Docs Example: SEO for Technical Documentation

Since I'm a technical writer, let me walk through exactly how this skill would work on a documentation repo. Say I have a docs site built with Docusaurus, and I want to improve SEO compliance across my pages.

### I open the repo and say:

```
Run autoresearch to optimize SEO for my documentation
```

### Phase 1 — The agent scans:

```
Repo: my-product-docs
Stack: Markdown, MDX, Docusaurus, Node.js
Purpose: Technical documentation site for a SaaS API product
Quality tools found: markdownlint, broken-link-checker, Docusaurus build
```

### Phase 2 — Skipped (I already stated my goal)

### Phase 3 — The agent defines eval criteria:

```
Eval criteria for docs SEO compliance (8 pages x 5 criteria = max 40):

1. Does the page have a unique, descriptive <title> under 60 characters?
   — yes/no — command: grep -q "title:" [file]

2. Does the page have a meta description between 120-160 characters?
   — yes/no — command: grep -q "description:" [file]

3. Does the page use exactly one H1 heading?
   — yes/no — command: grep -c "^# " [file] | grep -q "^1$"

4. Do all images have descriptive alt text (not empty, not "image", not "screenshot")?
   — yes/no — llm-judge

5. Does the page have internal links to at least 2 other documentation pages?
   — yes/no — llm-judge
```

I review. Looks good. I switch to Agent mode.

### Phase 4 — Baseline:

The agent creates `.autoresearch/` and writes an initial prompt:

```
Generate SEO-optimized frontmatter and structural improvements for the following
documentation page. Ensure the title is descriptive and under 60 characters.
Include a meta description of 120-160 characters that summarizes the page content.
The page should have exactly one H1 heading. All images must have descriptive alt
text. Include internal links to related documentation pages.
```

Baseline score: **24/40**

Breakdown:
- Title under 60 chars: 7/8
- Meta description: 4/8 (many pages missing or too short)
- Single H1: 8/8
- Alt text: 3/8 (generic "screenshot" on most images)
- Internal links: 2/8 (most pages are islands)

### The loop runs:

**Run 2** — Mutation: *add constraint*. Added "Meta descriptions MUST be between 120-160 characters. If the page discusses an API endpoint, the description must include the HTTP method and path." Score: 28/40. **KEEP.**

**Run 4** — Mutation: *add negative example*. Added "DO NOT use generic alt text like 'image', 'screenshot', 'diagram', or 'figure'. Instead describe what the image shows: 'Authentication flow showing JWT token exchange between client and server'." Score: 31/40. **KEEP.**

**Run 6** — Mutation: *tighten language*. Changed "Include internal links to related pages" to "MUST contain at least 2 internal links to other documentation pages. Link text must be descriptive (not 'click here' or 'see more')." Score: 35/40. **KEEP.**

**Run 9** — Mutation: *add counterexample*. Added a before/after example:
```
BAD:  description: "This page covers authentication"  (too short, 42 chars)
GOOD: description: "Configure OAuth 2.0 authentication for your API integration with step-by-step setup, token management, and troubleshooting"  (138 chars)
```
Score: 38/40. **KEEP.**

**Run 12** — Score: 40/40. **KEEP.** Perfect score.
**Run 13** — 40/40. **Run 14** — 40/40. Three consecutive perfect scores. Loop stops.

### Final result:

```
AUTORESEARCH COMPLETE
  Runs: 14
  Starting score: 24/40
  Final best score: 40/40
  Improvement: 66.7%
  Runs kept: 6
  Most effective mutation operators:
    1. add_counterexample (2/2 KEEPs — 100%)
    2. tighten_language (2/3 KEEPs — 67%)
    3. add_constraint (1/2 KEEPs — 50%)
    4. add_negative_example (1/2 KEEPs — 50%)
    5. restructure (0/2 KEEPs — 0%)
    6. remove_bloat (0/1 KEEPs — 0%)

Best prompt saved to: .autoresearch/best_prompt.txt
Full history: .autoresearch/results.jsonl
```

The best prompt in `.autoresearch/best_prompt.txt` is now a battle-tested, optimized set of instructions for generating SEO-compliant documentation. I can use it as a template, feed it to other tools, or apply it across my entire docs site.

And I got here by doing nothing after the initial setup. The agent ran 14 cycles autonomously, figured out what worked, threw away what didn't, and handed me the winner.

---

## The Evolution: From Karpathy's Repo to a Universal Skill

Here's the lineage:

**Stage 1: Karpathy's autoresearch** — An agent modifies `train.py`, runs a 5-minute experiment, measures `val_bpb`, keeps or reverts. Loop forever. The insight: autonomous experimentation with objective metrics.

**Stage 2: The diagram skill** — A YouTuber applies the same pattern to prompt optimization. Instead of training code, he optimizes a text-to-image prompt. Instead of `val_bpb`, he uses binary eval scores (40 max). Instead of `evaluate_bpb()`, he uses Claude vision as the judge. The insight: the pattern works on prompts, not just code.

**Stage 3: This universal skill** — I take the diagram skill's approach and make it domain-agnostic. Instead of hardcoding to diagrams, the skill scans any repo and adapts. Instead of 4 fixed criteria, it generates criteria from its own knowledge of best practices. Instead of one mutation strategy, it rotates through 6 named operators. And I added the engineering rigor the diagram skill lacked: eval isolation, context window management, sample tracking, item-level failure detection, plateau breaking, and adversarial self-evaluation.

The pattern is the same at every stage. Only the thing being optimized changes:

```
Karpathy:  train.py        → val_bpb        → keep/revert code
Diagrams:  prompt.txt      → eval score     → keep/revert prompt
Universal: prompt.txt      → eval score     → keep/revert prompt
            (for anything)   (any criteria)   (any domain)
```

---

## Am I in the Right Direction?

I'm a technical writer. I didn't build a startup. I didn't train a model. I took an idea from one of the most respected AI researchers alive, watched a YouTuber translate it from ML training to prompt optimization, and vibe-coded it into something I can use in my own work — documentation SEO, style guide compliance, content quality, API reference completeness.

The skill isn't perfect. It will evolve. The eval isolation could be stronger. The mutation operators could be smarter. Future models will make the loop faster and the judgments more accurate. But the architecture is sound: scan, suggest, define metrics, run the loop, keep winners, throw away losers.

If you're reading this and thinking "I could use this for [my thing]" — you're right. That's the point. The skill is universal because the pattern is universal. Generate, evaluate, keep the best, try again.

This is the part that still amazes me. I didn't write a single line of the skill by hand. I described what I wanted in natural language, let the AI build it, ran it through two rounds of critique, and incorporated the fixes — all in one conversation. The AI wrote the skill. The AI critiqued the skill. The AI fixed the skill. And now the skill itself uses AI to optimize AI-generated outputs.

We've reached the point where a technical writer can take a concept from a world-class ML researcher, translate it through a YouTube video, and build production-grade tooling by having a conversation. No Python expertise required. No ML background. Just an idea and the ability to describe what you want.

This is how AI works today. Not next year. Today.

---

## Try It Yourself

The skill files are in this folder:

- **Cursor**: copy `cursor/SKILL.md` to `~/.cursor/skills/autoresearch-universal/SKILL.md`
- **Claude Code**: copy `claude-code/SKILL.md` to `~/.claude/skills/autoresearch-universal/SKILL.md`

Open any repo. Switch to Plan mode. Say "run autoresearch." See what happens.

If you want to go deeper, read the `SKILL.md` itself — it's 400 lines of plain English that tell the agent exactly what to do, step by step. Every design decision is intentional. Every operational rule exists because something went wrong without it.

If you're a technical writer like me, start with docs SEO. If you're a front-end dev, start with accessibility. If you're a backend engineer, start with test case quality. If you're a DBA, start with query patterns.

The skill doesn't care what you optimize. It only cares that you can measure it with yes or no.
