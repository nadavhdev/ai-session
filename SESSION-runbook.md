# Live Session Runbook — SDLC with Claude Code
### Building `csvdiff` on csvkit, planning- and human-in-the-loop-centered

**Duration:** 90 minutes · **Win condition:** depth on ONE PR, end-to-end ·
**Base:** csvkit (MIT) · **Feature:** `csvdiff` (full PRD; session ships PR #1) ·
**Integrations scripted:** Trello + GitHub + git

> Facilitator note: every command/shortcut below was verified against current
> Claude Code docs. If a shortcut misfires on the room's machine, fallbacks are
> noted inline. Times are wall-clock targets; the **Cut list** (end) is what to
> drop if you fall behind.

---

## 0. Pre-flight (BEFORE the session — do not do this live)

- [ ] csvkit cloned, `pip install .[test]` done, `pytest --cov csvkit` green.
- [ ] `PRD-csvdiff.md` in repo root (or `docs/`). **Committed.**
- [ ] `CLAUDE.md` authored and present — but kept in a **separate branch / stashed**,
      so you can run the "without guardrails" beat first, then drop it in.
- [ ] `.claude/skills/scaffold-csvkit-tool/SKILL.md` authored (see Appendix A).
- [ ] `.claude/hooks` configured: a PostToolUse hook running `flake8`/`pytest`
      (see Appendix B). Kept disabled until the guardrails beat.
- [ ] Trello board "csvdiff" created with one list (e.g. "csvdiff tasks");
      GitHub repo fork ready; `gh` CLI authenticated; Trello MCP connected.
- [ ] Claude Code updated (`npm update -g @anthropic-ai/claude-code`); confirm
      `Shift+Tab` cycles to plan mode on the demo machine (else use `/plan`).
- [ ] Terminal font large; one pre-made "differences" pair of CSVs staged in
      `examples/` as a live demo input.
- [ ] **Dry run done once.** Non-negotiable. Live agentic work has dead air.

---

## 1. Frame the session (0:00–0:06, 6 min)

Say, don't show:
- The thesis: *the model is the same; outcomes diverge based on the planning and
  guardrails around it.* Today we go PRD → research → design → tasks → one PR →
  review, with a human gate at every phase.
- Name the artifacts as a chain: **PRD (intent) → CLAUDE.md (enforcement) →
  research (grounding) → design (decisions) → tasks → PR → review.**
- Set the rule you'll model all session: *Claude proposes, the human disposes.*
  Nothing advances a phase without a human approving the artifact.

Show the repo tree + the PRD open. 30 seconds. Don't read it aloud.

---

## 2. The PRD (0:06–0:12, 6 min)

- Scroll the PRD; land on three things only: §8 (UX vision), §10 (open decisions
  table, all `TBD`), §12 (rollout — *note it says nothing about PRs; that's a
  later artifact*).
- **Teaching beat:** the PRD is repo-blind on purpose. It states the *what/why*,
  not the *how*. The open-decisions table is the seed of the research phase.
- Do NOT generate the PRD live — it's pre-baked. Say so; it keeps the clock.

---

## 3. CLAUDE.md — what, why, multi-repo (0:12–0:22, 10 min)

This is a *concept* segment with a live contrast — your second-most-important
teaching block after planning.

**3a. The "without" shot (3 min).** On the branch with **no CLAUDE.md**, in a
fresh Claude Code session, prompt:
> "Add a new `csvdiff` command to this project that diffs two CSV files."

Let it run ~60–90s. Narrate what it gets wrong against ground truth (Appendix C):
likely hallucinates the base class or registration, may invent `make test`,
may re-add a common flag, guesses test location. **Don't fix it. Stop it.**
This is the pain you're about to remove.

**3b. The "with" reveal (4 min).** Drop `CLAUDE.md` into the repo root. Open it.
Walk the sections fast: architecture (the `CSVKitUtility` pattern), the
inherited-flags rule, the agate idiom, the exact commands, the DoD checklist,
guardrails. Emphasize: **feature-agnostic, repo-level, committed to git so the
whole team + Claude inherit it.** Contrast with `/init` (mention: Claude Code can
generate a starter, but a *grounded* CLAUDE.md beats a generated guess).

**3c. Multi-repo question (3 min).** Pre-empt it (you raised it): standards live
*with the code*. Each repo has its own root `CLAUDE.md`; `~/.claude/CLAUDE.md`
holds only personal cross-repo prefs; monorepo subdirs can override. The rule:
*if it must be true every turn, CLAUDE.md; if it's a sometimes-procedure, a
skill; if it's deterministic automation, a hook.* (You'll show all three today.)

---

## 4. Research phase — plan mode + grounding (0:22–0:40, 18 min)

The heart of the session, part 1.

**4a. Enter plan mode (1 min).** `Shift+Tab` to plan mode (fallback `/plan`).
Say what it means: read-only, Claude explores and proposes, **modifies nothing**
until you approve. Note plan mode has read/research tools only (Read, Grep, Glob,
Task/subagents, WebSearch).

**4b. Research prompt (2 min to type/explain).**
> "We're going to add a `csvdiff` command (see PRD-csvdiff.md). Before any design,
> research this codebase and produce a research artifact at
> `docs/research/csvdiff-research.md` covering: (1) the exact pattern a new tool
> must follow — base class, required methods, registration; (2) the closest
> existing analog and why; (3) how two-file input + STDIN is handled; (4) how
> values are compared — typed vs string; (5) what csvkit does today for exit
> codes and for duplicate keys (look at csvjoin); (6) the test + lint + docs
> obligations. Cite specific files and line ranges. Flag anything the PRD's open
> decisions (§10) depend on. Do not propose a design yet."

**4c. Let it research (~6 min).** Optionally: show a **subagent** (Explore) doing
this in an isolated context so the main thread stays clean — mention `Task`
tool / `.claude/agents`. Keep narrating: *this is grounding, the antidote to 3a.*

**4d. HUMAN-IN-THE-LOOP GATE #1 (~7 min). The money moment.**
Open the research artifact. Grade it live against **Appendix C answer key**.
Drive the audience through the critical questions:
- "It claims the base class is `X` — is that right? Open `cli.py` and check."
- "Does it correctly say csvkit has **no** non-zero result exit code today? That's
  the key finding — FR7 is a *new* pattern, not a copy."
- "Typed vs string: did it catch that agate infers types by default, so `1` ==
  `1.0`? That answers OD4."
- "Did it find that `-c` is taken by csvjoin, so `--key` won't collide?"
Mark OD2 and OD4 in the PRD table as **research-resolved**. If Claude got
something wrong, **that is the lesson** — correct it, show why grounding matters.
Only when the artifact is *trustworthy* do you proceed. Say: "approved."

---

## 5. Technical design phase (0:40–0:55, 15 min)

The heart of the session, part 2.

**5a. Design prompt, still in plan mode (2 min).**
> "Using the research artifact and the PRD, produce a technical design at
> `docs/design/csvdiff-design.md` for **PR #1 only** (the walking skeleton:
> key-matched add/removed/changed/unchanged, human-readable output, exit codes,
> standard input handling — NOT schema drift, NOT machine output). Include: the
> module/class, the new arguments, the in-memory model and its memory tradeoff,
> the exit-code scheme (new pattern — define the numbers), resolution of open
> decisions OD1–OD4 with rationale, the test plan with fixture names, and the
> docs/changelog obligations. Show the `main()` flow as steps."

**5b. Let it design (~5 min).**

**5c. HUMAN-IN-THE-LOOP GATE #2 (~8 min).** Grade against **Appendix D**.
The critique targets — make the audience do the thinking:
- Memory model: "It says hold file B in a dict keyed by `--key`. What breaks at
  5M rows? Is that acceptable for PR #1? Does it match csvjoin's 'reads all into
  memory' precedent and epilog honesty?" (Acceptable for #1 — but it must be
  *stated*, per CLAUDE.md.)
- Exit codes: "0 = identical, 1 = differences, 2 = error — does 2 collide with
  argparse's own exit 2? Is that a problem?" (It does; decide deliberately.)
- OD1 (no-key default): "Did it pick positional or require-a-key? Is the
  rationale a *product* rationale or did it hand-wave?"
- Scope discipline: "Did it stay inside PR #1, or did it sneak schema drift in?"
  (Catch scope creep live — recurrent failure mode.)
Edit the design with Claude until it's right. **Approve before any code.**

---

## 6. Task breakdown → Trello (0:55–1:05, 10 min)

**6a. Breakdown prompt (1 min).**
> "Break the approved PR #1 design into ordered, independently-reviewable tasks.
> For each: title, description, the files it touches, and its definition of done
> from CLAUDE.md. Keep tasks small enough to review in isolation."

**6b. Push to Trello (scripted MCP, ~4 min).** This is where task-breakdown
becomes a tracked artifact.
> "Create a card on the 'csvdiff tasks' Trello list for each task, with the
> description and DoD as the card body."
Show the cards appearing in Trello. **Teaching beat:** the plan is now external,
shareable, the team's source of truth — not trapped in a terminal. (If Trello MCP
fails live: fall back to `gh issue create` per task — see Cut list.)

**6c. Quick human check.** Are the tasks correctly ordered and atomic? Reorder a
card live to show you, the human, own the plan.

---

## 7. Implement PR #1 — ONE task live (1:05–1:20, 15 min)

Do **not** try to implement every task. Implement the **first** task fully; the
rest are "same pattern, fast-forwarded."

**7a. Branch + skill (2 min).** `git checkout -b feature/csvdiff`. Invoke the
**skill**: `/scaffold-csvkit-tool csvdiff` (Appendix A) — show that the skill
encodes the CLAUDE.md pattern as a *reusable procedure*, so the scaffold is
correct by construction (subclass, `add_arguments`, `main`, `launch_new_instance`,
registration line, test stub). **This is the skills payoff:** CLAUDE.md is the
rule; the skill is the procedure that applies it.

**7b. Exit plan mode, implement (8 min).** `Shift+Tab` to normal/acceptEdits.
Prompt Claude to implement task #1 against the design. As it edits, the
**PostToolUse hook** (Appendix B) auto-runs `flake8`/`pytest` after each change.
**Teaching beat — the three layers now visible together:** CLAUDE.md (rules) +
skill (procedure) + hook (deterministic enforcement, can't be hallucinated,
blocks on exit code 2). Let the hook catch a lint error live if it happens —
that's gold.

**7c. HUMAN-IN-THE-LOOP GATE #3 (5 min).** Review the diff *as a human reviewer*,
not a spectator: does it match the design? Tests present and meaningful? Did it
honor the exit-code decision? Run `pytest --cov csvkit` once on screen.

---

## 8. Deliver + review the PR (1:20–1:28, 8 min)

**8a. Commit + push + open PR (scripted git/GitHub, 4 min).**
`git add -p` (show selective staging), commit with a message referencing the
Trello card, `git push`, then:
> "Open a PR with `gh`, summarizing the change, linking the design doc and the
> Trello card, and listing the CLAUDE.md DoD items with checkboxes."
Show the PR live on GitHub. CI (flake8 + pytest matrix) kicks off — point at it.

**8b. Review with the built-in skill (4 min).** Run `/code-review` (bundled
skill) on the PR, OR a custom review subagent. **Teaching beat:** the *same*
CLAUDE.md that guided authoring now grounds the review — the DoD checklist is the
review rubric. Human makes the final call: request a change live, or approve.
Move the Trello card to Done.

---

## 9. Close: the contrast + the infra map (1:28–1:30, 2 min)

One slide / spoken recap:
- Replay the divergence: 3a (no guardrails: guessing, scope creep, wrong base
  class) vs. everything after (grounded, gated, tracked).
- The infra map, one line each: **CLAUDE.md** = always-true rules · **skills** =
  reusable procedures · **hooks** = deterministic automation · **subagents** =
  isolated research/review · **plugins** = bundle all of the above to share ·
  **MCP** = external tools (Trello/GitHub) · **plan mode** = the human gate.
- The point: planning + human gates + grounding turn a confident guesser into a
  dependable engineer. Same model. Different system.

---

## Cut list (drop in this order if behind)

1. Subagent demo in §4c → just research in the main thread.
2. §8b custom review → use bundled `/code-review` only, or skip to human approve.
3. §6b Trello → `gh issue create` one-liner, or verbally note "this would sync
   to Trello."
4. §7a skill scaffold → let Claude write the file directly (lose the skills beat
   but keep the PR).
5. Shrink §5 design critique to the two sharpest targets (memory model + exit
   codes).
**Never cut:** Gate #1 (research grading) and Gate #2 (design critique). They ARE
the session.

---

## What to demo only if AHEAD of time

- `claude cowork` for a non-dev view of the same flow (you flagged interest).
- Agent SDK as the programmatic version of today's flow (mention; don't build).
- A plugin bundling the skill + hook + Trello MCP as the team-shareable unit.

---

## Appendix A — Skill: `.claude/skills/scaffold-csvkit-tool/SKILL.md`

```markdown
---
name: scaffold-csvkit-tool
description: Scaffold a new csvkit command following the CSVKitUtility pattern.
  Use when adding a new tool to csvkit/utilities/.
---
Scaffold a new csvkit tool named `$0`. Create:
1. `csvkit/utilities/$0.py` — a `CSVKitUtility` subclass with `description`,
   `add_arguments()`, `main()` (validate→read→transform→write), and
   `launch_new_instance()`. Multi-file tools set `override_flags = ['f']` and
   take positional `input_paths` default `['-']`, guarding the tty/no-input case.
2. The `[project.scripts]` registration line in `pyproject.toml`.
3. `tests/test_utilities/test_$0.py` subclassing `CSVKitTestCase, EmptyFileTests`
   with `Utility` + `default_args` + a `test_launch_new_instance`.
Do not re-add inherited common flags. Read csvjoin.py first and mirror it.
```

## Appendix B — Hook: PostToolUse lint/test gate

In `.claude/settings.json` (conceptual; verify exact schema on the demo machine):
- Event: `PostToolUse`, matching file-edit tools.
- Command: run `flake8 <changed file>` and the relevant `pytest -k`. Exit code
  **2** blocks and feeds the error back to Claude to fix. (The JSON
  decision/reason return is deprecated; use exit-code semantics.)
- Keep DISABLED until §7 so the §3a "without" shot is honestly ungoverned.

## Appendix C — Research-phase ANSWER KEY (ground truth, verified vs. repo)

| Item | Ground truth |
|------|--------------|
| Base class | `CSVKitUtility` in `csvkit/cli.py`; subclass implements only `add_arguments()` + `main()`; `run()` (base) opens files + calls `main()`. |
| Closest analog | `csvjoin.py` — multi-file, reads all tables into memory, uses `match_column_identifier`. |
| Registration | `[project.scripts]` line in `pyproject.toml`: `csvdiff = "csvkit.utilities.csvdiff:launch_new_instance"`. No Makefile. |
| Multi-file + STDIN | Positional `input_paths` default `['-']`, `override_flags=['f']`, guard `isatty(sys.stdin) and input_paths==['-']`. CI tests every tool via `<` and `|`. |
| Typed vs string | agate infers types by default → `1` and `1.0` compare equal as typed values unless `-I/--no-inference`. (Resolves OD4.) |
| Exit codes today | None for data conditions. Errors go via `argparser.error()` → exit **2**. A non-zero "differences found" code is a NEW pattern (resolves the FR7 design point). |
| Duplicate keys | csvjoin relies on agate join semantics; no first-wins/error convention for a *diff*. Must be decided (OD2). |
| Tests | `tests/test_utilities/test_<tool>.py`, `unittest`, `CSVKitTestCase`+`EmptyFileTests`, helpers `assertRows`/`assertError`/`get_output_as_io`, fixtures in `examples/`. |
| Lint/test | `pytest --cov csvkit`; `flake8 .` (max-line 119, setup.cfg); `isort . --check-only`; `check-manifest`. |
| Docs/changelog | `docs/scripts/csvdiff.rst`; top-of-`CHANGELOG.rst` `feat:` bullet; add to `AUTHORS.rst`; man page optional + data-files entry. |

## Appendix D — Design-phase ANSWER KEY (what a good PR #1 design contains)

- **Module/class:** `csvkit/utilities/csvdiff.py`, `class CSVDiff(CSVKitUtility)`,
  `override_flags=['f']`, two positional inputs.
- **Args:** `--key/-k` (column name(s); multi via comma or repeat — engineer's
  call), reuses inherited input flags. Must NOT collide with `-c`.
- **Memory model:** index the second file by key into a dict; stream/iterate the
  first and classify. O(rows) memory on file B — **must be stated** (epilog),
  acceptable for PR #1 (mirrors csvjoin's "all into memory" honesty).
- **Exit codes:** define explicitly, e.g. 0 identical / 1 differences / 2 usage —
  and acknowledge argparse already uses 2; pick deliberately and document.
- **OD1 default:** a defensible product choice (e.g. require `--key`, or
  positional fallback) WITH rationale.
- **OD2/OD4:** resolved per research (OD4 = typed by default; OD2 = explicit
  duplicate-key policy chosen).
- **Test plan:** fixtures for added / removed / changed / unchanged / no-diff /
  bad-key; both file and STDIN invocation; exit-code assertions.
- **Scope guard:** NO schema drift, NO JSON output — those are PR #2/#3.
- **Red flags to catch:** scope creep into schema drift; silent full-load of both
  files with no statement; reusing `-c`; tests only on the happy path; inventing
  a base class or a `make` target.
```
