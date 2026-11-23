---
name: ledger
description: A traceability workflow for AI agents. Three on-disk ledgers — a file ledger (MANIFEST.md), a chapter book (BOOK.md, index-first retrieval), and an artifact ledger — plus five gate commands that exit non-zero when work is untraceable. Use when a task will produce more than a few files or run across more than a few steps, when outputs are landing in scattered or duplicate files, when the context is bloating from re-reading history, or when you need to decide whether a task deserves a ledger at all.
whenToUse: More than 3 output files or more than 3 steps; work that must be retrieved in a later session; files appearing as v2/final/copy; needing to decide whether to create a ledger; a gate command failed and you need the fix.
---

# Ledger

> **Everything is ledgered; everything is paged on demand.**

A conclusion that cannot be found again did not happen.
A "done" that nothing checks is an opinion.

---

## 0. The one-line loop

```
init -> book -> chapter -> record -> gate -> close
```

Do not skip to `chapter`. The plan is itself a verifiable artifact.

**Ledger is not "always better". It is better in the right band.** Applied to a one-line change it produces empty shells; that failure has been measured and the threshold in §6 exists because of it.

---

## 1. Three iron rules

| # | Rule | Machine check |
|---|---|---|
| 1 | **Pass a threshold before you ledger.** >3 outputs, >3 steps, or must be retrievable later. Otherwise work directly and state the path. | `init --expect N [--trace]` refuses and suggests skipping |
| 2 | **One chapter, one file; the index is the only entry point.** Prose lives in chapter files. `BOOK.md` holds title, one-line summary, keywords, status — nothing else. | `book check` (duplicate numbers, status/file mismatch) |
| 3 | **Every output is attached.** A file belongs to a chapter, or the plan had a gap. | `book check` / `book orphans` (exit 1) |

Rule 1 is a correction, not a preference. Its earlier form — *every task, even a one-line change, must init* — produced 26 of 83 ledgers that were more than 20% of the task's total output, 5 of them 100%. A rule that manufactures empty shells is a bad rule, however tidy it sounds.

---

## 2. Structure

```
Ledger
├─ 3 iron rules
├─ 3 ledgers
│   ├─ file ledger      MANIFEST.md   who made it / what for / who reads it / how to verify
│   ├─ chapter book     BOOK.md       which chapter answers my question
│   └─ artifact ledger  artifact *    keep / register path only / delete, each with a reason
└─ 1 CLI               tools/ledger.mjs   zero dependencies, Node >= 18
```

A ledger is not a transcript. A transcript is read top to bottom; a ledger is **paged**.

---

## 3. Layout

```
<root>/<slug>/
├── MANIFEST.md          file ledger
├── BOOK.md              chapter book (<=200 lines, the only retrieval entry)
├── 00-brief/ch00.md
├── 10-plan/ch10.md
├── 20-exec/ch20.md
├── 30-evidence/ch30.md
├── 40-deliver/ch40.md   the single authoritative conclusion
├── out/                 real deliverables
├── backup/              originals, for rollback
└── tmp/                 probes; must be empty at close
```

`<root>` resolves as `--root <dir>` → `$LEDGER_ROOT` → `./_work`.

Chapter directories are **derived from the number**: `ch0x`→`00-brief`, `ch1x`→`10-plan`, `ch2x`→`20-exec`, `ch3x`→`30-evidence`, `ch4x`→`40-deliver`.

**Never number documents by round.** `round-17-notes.md` is not structure. Stages are structure. Permanent assets (skills, promoted tools) do not live in a task directory — move them out and register the move.

---

## 4. The loop

| # | Action | Command | Verify |
|---|---|---|---|
| 1 | Create the task dir | `ledger init <slug> --goal "..." --expect N [--trace]` | `MANIFEST.md` exists |
| 2 | **Write the whole chapter table at once** (empty bodies are fine) | `ledger book init <slug> --title T --goal G --ch "ch10\|Title\|Summary\|kw1,kw2" xN` | `book check` chapter count |
| 3 | Create the file **before** writing the body | `ledger book touch <slug> chNN --answer "what must this answer"` | first section non-empty after edit |
| 4 | Register every output twice | `ledger add` + the chapter's "Outputs from this chapter" table | `book check` reports 0 orphans |
| 5 | One conclusion, one place | only `40-deliver/ch40.md`; other chapters point at it | no second conclusion |
| 6 | Gates | `book check` → `book orphans` → `book root` → `artifact guard` → `close` | all pass |

```bash
ledger init cache --goal "pick an invalidation strategy" --expect 8 --trace
ledger book init cache --title "Invalidation" --goal "pick one" \
  --ch "ch10|Options|the candidates|candidates" \
  --ch "ch30|Evidence|bench results|bench" \
  --ch "ch40|Conclusion|the decision|decision"
ledger book touch cache ch30 --answer "what the benchmark showed"
ledger add cache out/bench.json --does "p99 numbers" --for "ch30" --verify "cat out/bench.json"
ledger book check cache
ledger close cache
```

**State machine**: `pending` --touch--> `writing` --conclusion written--> `done`; abandoned chapters are marked `dropped` and **the number is never reused**.

`check` rules: `status != pending` but the file is missing → fail. File exists but status is still `pending` → warn.

Append a chapter you forgot with `ledger book add <slug> --ch "ch11|..."`. Renumbering is not supported.

---

## 5. Gates

| # | Gate | Covers | On failure |
|---|---|---|---|
| 1 | `book check <slug>` | **Inside one task**: duplicate chapter numbers, status/file mismatch, orphan outputs, `BOOK.md` >200 lines; warns on index rot and ledger drift | exit 1 |
| 2 | `book orphans [slug]` | **Inside one task, no book needed**: unregistered files against `MANIFEST.md` | exit 1 |
| 3 | `book root` | **Between tasks**: the root holds only task directories | exit 1 |
| 4 | `artifact clean <slug> [--apply]` | Keep / register / delete. **Dry-run by default** | exit 1 only on refusal cases |
| 5 | `artifact guard <slug> --max-mb N` | Size budget for one task | exit 1 |
| 6 | `close <slug>` | Unregistered files + `tmp/` leftovers; flips status to `done` | exit 1 |

`check` guards inside a task, `root` guards between tasks. Together they answer "are the files scattered".

### Artifact retention

| Artifact | Ruling |
|---|---|
| Failed build output (apk/ipa/dex/img/dump) | delete; keep only the `.md` report |
| Successful, <=50MB | keep |
| Successful, >50MB | delete the file, register path + size in the report |
| `dex/so/dll/class` (regenerable from a package) | delete |
| dump / decompile / log / one-off script | delete (promote a general one to a permanent tool first) |
| Any `.md` report or analysis data | **always keep** |

Images: a filename carrying `keyframe / contact-sheet / storyboard / reference` is a deliverable asset and is kept; `screen / shot / capture / screenshot` or anything under `tmp/` is an analysis screenshot — keep the data, drop the image; anything unrecognised is kept conservatively. Every ruling prints its reason, which is the only reason a heuristic is acceptable here.

---

## 6. The threshold (the important section)

```
New task
  |- <=3 outputs, one-shot            -> no ledger. Work. State the path in your reply.   [A]
  |- <=3 outputs, needs a paper trail -> init only, no book.                              [B]
  |- >3 outputs or >3 steps           -> init + book.                                     [C]
  '- must be retrievable later        -> init + book, keywords filled in.                 [C]
```

| Use it (positive return) | Do not (negative return) |
|---|---|
| More than 3 output files | Changing one line of config (overhead can hit 100%) |
| More than 3 steps, interruptible | A single lookup |
| A conclusion people will re-ask about | Build-and-ship with no intermediates |
| Handing work to a person or a later session | A mechanical replace |
| A shared config change needing rollback | Exploratory thrash that will be rewritten |
| Outputs that multiply copies and big files | — |

**Staged exit**:

```
exploration (outcome unknown) -> no book; allow tmp/ to be messy; write the book once the answer settles
convergence (direction set)   -> init + book, fill chapters
delivery                      -> check + orphans + guard + close
```

A measured counter-example: a task with 528 files and 964 MB kept a `MANIFEST.md` and no book. **That was the correct call** — a book would have been rewritten every time the exploration turned.

**Signals to stop**: three consecutive `check` failures that are all "forgot to touch" → demote to manifest-only. Same chapter rewritten >2 times → the book was built too early. Overhead above 30% of the task's total output → do not init next time.

---

## 7. Retrieval

1. Read `BOOK.md` (<=200 lines) → pick 1–2 chapters from the summary and keyword columns.
2. Read **only those chapter files**.
3. Only if the index misses may you `grep`, and only inside `20-exec/` and `30-evidence/`. **Never recurse the project.**
4. Cross-task history → `ledger book find <keyword>` (reads indexes only).
5. **Write the summary and keywords back** after you learn something, so step 1 hits next time.

| Method | Read volume |
|---|---|
| `grep -r` the whole workspace | ~115,000 files / 46 GB |
| `book find` | the `BOOK.md` indexes, ~14 KB |
| The chapter you hit | one file, ~5 KB |

Three orders of magnitude, measured.

---

## 8. Known limitations

Stated plainly. A workflow that hides its failure modes cannot be trusted.

| # | Limitation | Status |
|---|---|---|
| 1 | Small-task overhead can exceed the output (26/83 measured, 5 at 100%) | **Fixed** by the §6 threshold |
| 2 | Fixed boilerplate in `MANIFEST.md` | Slimmed to ~1.2 KB; `sync` refreshes old ledgers |
| 3 | Books were used in only 11% of tasks; 45 tasks with >3 files had none | **Partly unsolvable** — a judgement call, not a mandate |
| 4 | `MANIFEST.md` and `BOOK.md` can disagree | **Detected** by `check` (warn, non-blocking) |
| 5 | Index rot: body changed, summary not updated | **Detected** by `check` (warn, non-blocking) |
| 6 | Chapter numbers are permanent | **Accepted** — stable references beat refactor convenience. Chapters can be appended, never renumbered |
| 7 | Three gates used to be dead for tasks without a book (89% of them) | **Fixed** by `book orphans` |
| 8 | Artifact classification is heuristic; it can be silently wrong | **Accepted** — every decision prints its reason, and clean is dry-run by default |
| 9 | Everything is prompt-level; there is no hook-level enforcement | **Unresolved** — an agent can ignore all of it |
| 10 | `artifact clean` treating `backup/` as process material would delete rollback originals | **Fixed** — `backup/` is exempt in every gate and in the classifier |

---

## 9. Long-term memory does not replace a book

Memory tools have a hard size cap and are billed every turn. They are for cross-session *facts*: environment quirks, user preferences, lessons that do not belong to one task.

Task bodies belong on disk. Write the fact into `BOOK.md`'s keyword column so step 1 of §7 hits, and leave the prose in the chapter.

---

## 10. Self-check

Four lines. If you cannot fill one, the task is not finished.

```
Book:      <root>/<slug>/BOOK.md   (N chapters, all done)
Retrieval: read BOOK.md -> hit chXX -> no full scan was needed
Recording: MANIFEST.md and BOOK.md output tables agree (N items each)
Cleanup:   tmp cleared N, archived N, deleted N; check / orphans / root / guard all pass
```

And the closing three lines go in your reply:

```
Authoritative artifact: <single path>
Verification: <command> -> exit 0
Cleanup: tmp cleared N, archived N, deleted N
```
