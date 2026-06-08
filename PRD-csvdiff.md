# PRD: `csvdiff` — a CSV-aware comparison tool for csvkit

---

## 1. Problem

People who work with CSVs — data journalists, analysts, data engineers, QA
engineers — constantly need to answer one question: **"What changed between
these two versions of this file?"** Today the available tools fail them:

- **`diff` / `git diff`** operate on lines, not records. A reordered column, a
  re-sorted file, or a quoting change produces a diff that is pure noise even
  when the *data* is identical. Conversely, a one-cell change inside a long row
  is reported as a whole-line replacement, hiding *which field* changed.
- **Spreadsheet "compare" features** need a GUI, are manual, don't script, and
  choke past a few hundred thousand rows.
- **Bespoke pandas scripts** are the common fallback. Everyone rewrites the same
  brittle merge-and-compare logic, with no standard output and no CLI.

csvkit already owns the CLI workflow for converting, filtering, joining,
sorting, and summarizing CSVs. It has every adjacent verb — `csvjoin`,
`csvsort`, `csvstat`, `csvgrep` — **except the ability to compare two files.**
This is a conspicuous, user-visible gap in an otherwise complete toolkit.

## 2. Proposed solution

Add a new command, **`csvdiff`**, that compares two CSV files *semantically*:

- It matches rows by a user-specified **key** (one or more columns) so
  comparison survives re-sorting and row insertion — and falls back to a
  sensible default when no key is given.
- It classifies rows as **added**, **removed**, **changed**, or **unchanged**,
  and for changed rows it identifies **which fields** differ.
- It surfaces **schema drift** (columns added, removed, or reordered) distinctly
  from row changes.
- It returns a meaningful **exit code** and offers **machine-readable output**,
  so it works as a gate inside scripts and data pipelines.

`csvdiff` behaves like a native member of the suite: same input options, same
help style, same conventions as `csvjoin` and friends. It is the missing "diff"
verb that completes csvkit's vocabulary.

## 3. Why now / why us

- csvkit is the de-facto standard CLI for CSV manipulation; users already reach
  for it and are surprised "diff" isn't there.
- The capability composes with the existing suite (e.g. `csvsort` then
  `csvdiff`, or `csvgrep` then `csvdiff`) within one mental model and one install.
- No new heavyweight dependency is required — csvkit already provides typed
  columns and row iteration that a record-level diff can build on.

## 4. Users & primary scenarios

**Persona A — Data journalist ("did the dataset change?")**
Receives a monthly data dump from an agency. Needs to know which records were
added, removed, or amended versus last month, matched by a record ID, ignoring
the fact that the agency re-sorts the file each time.

**Persona B — Data engineer ("did my pipeline change the output?")**
Has a transformation step and wants a CI check that fails the build if the
output CSV differs from a golden/expected file — with a non-zero exit code and a
precise report of what diverged.

**Persona C — Analyst ("did the schema change on me?")**
A vendor silently added and reordered columns. Needs that surfaced loudly,
separately from row-level changes, before it corrupts a downstream join.

## 5. User stories

- **US1** — As a user, I run one command against two CSV files and get a
  human-readable summary of what differs.
- **US2** — As a user, I match rows by a key (one or more columns) so I see
  added, removed, and changed rows even when the file was re-sorted.
- **US3** — As a user who provides no key, I still get a useful default
  comparison so the tool works with zero configuration.
- **US4** — As a user, when the two files have different columns (added /
  removed / reordered), I am told so clearly and separately from row differences.
- **US5** — As a user in a script or CI, I get an exit code I can branch on
  (identical vs. differences vs. error).
- **US6** — As a user, I can get machine-readable output describing the changes,
  to feed into another tool.
- **US7** — As a user, the command's options, help, and file handling match the
  rest of csvkit, so it feels like part of the suite.

## 6. Goals (outcomes we want)

| # | Goal |
|---|------|
| G1 | Users can see differences at the **record and field level**, not the line level |
| G2 | Comparison is **robust to re-sorting and row insertion** (identity-based, not positional) |
| G3 | **Schema drift** is detected and reported clearly and separately |
| G4 | The tool is **usable as an automated gate** (clear pass/fail signal) |
| G5 | The tool feels like a **native, first-class** member of csvkit |

## 7. Requirements (what the product must do)

*Capability-level. The intended **experience** is described in §8 (Proposed user
experience). The literal **interface implementation** — exact option spellings,
specific exit-code numbers, output column layout — is illustrative there and is
finalized in technical design against csvkit's actual conventions.*

**Priority scale:** **P0** = highest (the feature is not shippable without it) ·
**P1** = important, expected for general availability · **P2** = lowest, can
follow later.

| ID | Requirement | Priority |
|----|-------------|----------|
| FR1 | Compare exactly two input CSV files | P0 |
| FR2 | Match rows by a user-specified key of one or more columns | P0 |
| FR3 | Classify rows as added, removed, changed, or unchanged | P0 |
| FR4 | For changed rows, indicate which fields differ | P0 |
| FR5 | Provide a useful default comparison when no key is specified | P1 |
| FR6 | Detect and report column-set differences (added / removed / reordered) distinctly from row differences | P1 |
| FR7 | Signal the result via process exit status: equivalent vs. differences vs. error | P0 |
| FR8 | Provide a default human-readable summary | P0 |
| FR9 | Provide a machine-readable output option | P2 |
| FR10 | Inherit csvkit's standard input handling (encoding, delimiter, quoting, header options, etc.) | P0 |
| FR11 | Provide help consistent with the rest of the suite | P1 |

## 8. Proposed user experience

*This is the PM's vision for how the command should feel to use. Flag names and
exit numbers below are **illustrative, shown for clarity** — the final spelling
and values are the engineer's call in technical design, constrained by csvkit's
existing conventions. The PM owns the **experience**; the engineer owns the
**interface implementation** and may challenge any specific below.*

**Basic comparison, matched by a key.** The user names the two files and the
column(s) that identify a row:

```
$ csvdiff --key id  old.csv  new.csv

3 changed, 1 added, 2 removed (of 1,204 rows compared)

~ id=4471   price: 19.99 -> 24.99   in_stock: yes -> no
~ id=4490   name: "ACME widget" -> "ACME Widget"
~ id=5012   price: 5.00 -> 5.50
+ id=9001   name: "New SKU"  price: 12.00  in_stock: yes
- id=3300   name: "Discontinued"  price: 8.00  in_stock: no
```

The experience goals embodied here:

- **Summary first.** The first line is a scannable headline a human can read in
  one glance, and a script can parse.
- **Marked, aligned rows.** `~` changed, `+` added, `-` removed — borrowed from
  the visual grammar of `diff` so it's instantly familiar.
- **Field-level changes are explicit.** A changed row shows *which* field moved
  and its before/after, not the whole row, so the eye goes straight to the
  delta.
- **Identity is shown.** Each line leads with the key value, so the user can
  locate the affected record in the source.

**Schema drift is called out separately, and loudly**, before row diffs — so a
column change is never silently mistaken for a wave of row changes:

```
$ csvdiff --key id  jan.csv  feb.csv

! schema changed:
    + column added:   region
    - column removed:  legacy_code
    ~ columns reordered

42 changed, 5 added, 0 removed (of 980 rows compared)
...
```

**Scripting / CI use.** The user relies on exit status, and can silence output:

```
$ csvdiff --key id expected.csv actual.csv  --quiet  &&  echo "match"  ||  echo "differs"
```

— exit `0` when equivalent, non-zero when differences are found, and a distinct
non-zero for misuse (e.g. a key column that doesn't exist). *(Exact numbers per
design.)*

**Machine-readable output** for piping into another tool (P2):

```
$ csvdiff --key id old.csv new.csv  --format json
```

— emitting a structured record per change (status, key, per-field before/after)
whose schema is defined in design.

**No-key default (P1).** Running without `--key` still does something useful
rather than erroring — the exact default behavior (positional comparison vs.
requiring a key) is an open product decision (see §10, OD1).

## 9. Non-functional requirements

- **NFR1 — Consistency:** behaves like a native csvkit tool; a user who knows
  `csvjoin` should feel at home.
- **NFR2 — Scale:** handles files in the hundreds of thousands of rows without
  pathological memory or time cost; any memory tradeoff is stated to the user.
- **NFR3 — Determinism:** identical inputs always produce identical output and
  exit status.
- **NFR4 — Tested:** covered per the project's existing test conventions, with
  fixtures for added / removed / changed / schema-drift cases.
- **NFR5 — Documented:** ships with per-tool documentation matching the suite's
  existing format.

## 10. Open product decisions

*These are decisions that must be made before the feature is built. They shape
behavior the user sees. They are listed here so they are resolved deliberately —
the **Resolution** column is filled in during planning, informed by research,
not left to chance during implementation.*

**"Depends on" key:**
- **Product judgment** — a values call with no single right answer in the code;
  the PM/team decides.
- **Codebase-constrained** — the answer is largely dictated by how csvkit /
  agate already work; research must confirm it before the PM can "decide," and
  diverging from the existing behavior would break consistency (NFR1). These are
  the items to verify against the actual code during the research phase.

| # | Decision | Options | Depends on | Resolution |
|---|----------|---------|-----------|-----------|
| OD1 | **No-key behavior** — what happens when the user supplies no key | (a) positional row-by-row compare; (b) refuse and require a key | Product judgment (convenience vs. avoiding misleading results on re-sorted files) | *TBD in planning* |
| OD2 | **Duplicate keys** — behavior when the key is not unique within a file | (a) error; (b) first-wins; (c) report rows as ambiguous | Codebase-constrained — confirm what `csvjoin` already does with duplicate join keys; match it for consistency | *TBD — research first* |
| OD3 | **Changed-row detail** — default verbosity for a changed row | (a) full before/after rows; (b) only the changed fields; (c) terse, detail behind a flag | Product judgment (ties to the §8 UX vision) | *TBD in planning* |
| OD4 | **What counts as "changed"** — are values differing only in formatting (`1` vs `1.0`, whitespace, case) equal or changed by default | (a) compare as agate-typed values; (b) compare as raw strings | Codebase-constrained — largely determined by whether csvkit loads values typed or as strings; confirm in research | *TBD — research first* |

*Implementation questions — in-memory model, type-inference mechanics,
machine-output schema — are deliberately excluded here and belong to technical
design, not the PRD.*

## 11. Success metrics

- **Adoption:** `csvdiff` appears in the suite's tool list and docs; shows up in
  community references within two release cycles.
- **Correctness:** zero "wrong diff" reports on core add/remove/change
  classification in the first release.
- **CI utility:** at least one documented recipe using `csvdiff` as a pass/fail
  pipeline gate.
- **Maintainer acceptance:** merged without requiring a rearchitecture (conforms
  to existing patterns).

## 12. Rollout plan

The feature is delivered **complete** (all Must requirements), but **released
gradually**, in OSS-appropriate stages:

1. **Experimental release.** Ship `csvdiff` in a minor version, documented as
   *experimental*, with the core record/field diff (G1, G2, G4) and exit-code
   behavior enabled. The doc explicitly invites feedback and notes the interface
   may change.
2. **Stabilization.** Over one or two release cycles, gather real-world issues
   (edge cases in keys, encodings, scale), confirm the open product decisions in
   §9 against actual usage, and lock the interface.
3. **General availability.** Remove the *experimental* label, announce in
   release notes / changelog as a stable, first-class tool, and add it to any
   "getting started" / tutorial material so it's discoverable alongside
   `csvjoin` and `csvsort`.

Schema-drift reporting (G3) and machine-readable output (FR9) may be introduced
in the experimental window or promoted slightly behind the core diff, depending
on feedback — but the GA feature is the full scope described above.

---

*This PRD is intentionally repo-blind: it states problem, users, and
requirements. Mapping these onto csvkit's actual architecture — base classes,
option plumbing, data-library idioms, test layout — and answering the
implementation questions is the job of the technical design phase. Breaking the
work into pull requests is the job of the task-breakdown phase. Neither belongs
in this document.*
