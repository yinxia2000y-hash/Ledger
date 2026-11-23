# Ledger

**Your agent already wrote the answer. Can you find it three weeks later?**

Ledger is a workflow skill that makes an AI coding agent keep three ledgers — a file ledger, a chapter book, and an artifact ledger — so that every piece of work is traceable, every conclusion is retrievable in one read, and "done" is a machine-checkable claim instead of an opinion.

<p>
<img alt="skill" src="https://img.shields.io/badge/type-agent%20skill-blue">
<img alt="deps" src="https://img.shields.io/badge/dependencies-zero-brightgreen">
<img alt="node" src="https://img.shields.io/badge/node-%3E%3D18-informational">
<img alt="license" src="https://img.shields.io/badge/license-MIT-lightgrey">
</p>

---

## The problem

Long agent sessions fail in four predictable ways.

| Symptom | What it looks like | What it costs |
|---|---|---|
| **Lost conclusions** | You know the agent figured out the answer. It is in the transcript, somewhere, 400 turns back. | Re-deriving it, badly |
| **Filename rot** | `report.md`, `report_v2.md`, `report_final.md`, `report_final_2.md` — none authoritative | Nobody knows which one is true |
| **Context blowup** | To answer "what did we decide about caching", the agent greps 115,000 files | Time, tokens, and a slower answer than reading a book |
| **Unfalsifiable "done"** | The agent says the task is complete. Nothing checks that. | Silent gaps in the deliverable |

The usual fix is a longer prompt: *"keep your work organised."* That is a suggestion. Suggestions are followed inconsistently, and you cannot tell afterwards whether they were followed at all.

**Ledger's fix is to convert the discipline into files with a schema, plus commands that exit non-zero when the schema is violated.**

---

## Measured, not asserted

These numbers come from the reference implementation running over 87 real task directories.

| Metric | Before | After |
|---|---|---|
| Files examined to answer one question | ~115,000 files / 46 GB scanned | All `BOOK.md` indexes, **~14 KB total**, then **one chapter file** |
| Retrieval cost | Full-project grep | **Three orders of magnitude smaller** |
| Tasks with an orphan-output gate | 9 of 83 (**11%**) | **100%** (`book orphans` needs no book) |
| Ledger overhead on a small task | 2.1 KB of boilerplate, up to **100% of total output** | 1.2 KB, and **not created at all** below the threshold |
| Empty-shell tasks caused by the workflow itself | 26 of 83 (**31%** were >20% boilerplate) | 0 by construction |

The last two rows matter: **the first version of this workflow was wrong, and we measured it.** Its rule was "every task, even a one-line change, must create a ledger." That rule manufactured 26 tasks whose only output was the ledger. The threshold in `skills/ledger/SKILL.md` §6 is the repair, not a nicety.

---

## What it actually does

Three ledgers, each with a single machine-checked question.

| Ledger | File | Question it answers | Gate |
|---|---|---|---|
| **File ledger** | `MANIFEST.md` | Who made this file, what is it for, who reads it, how do I verify it? | `close` reports unregistered files, exit 1 |
| **Chapter book** | `BOOK.md` | Which chapter answers my question? (≤200 lines, summary + keywords per chapter) | `book check` — orphans, duplicate chapter numbers, index rot, cross-ledger drift |
| **Artifact ledger** | `ledger.mjs artifact` | Keep it, register the path, or delete it? | `clean` is dry-run by default; `guard` enforces a size budget |

```
win/_work/<slug>/         (default root; override with --root or $LEDGER_ROOT)
├── MANIFEST.md           file ledger
├── BOOK.md               chapter book — the ONLY retrieval entry point
├── 00-brief/ch00.md
├── 10-plan/ch10.md
├── 20-exec/ch20.md
├── 30-evidence/ch30.md
├── 40-deliver/ch40.md    the single authoritative conclusion
├── out/                  real deliverables
├── backup/               originals of anything you changed
└── tmp/                  probes; must be empty at close
```

Chapter directories are derived from the chapter number (`ch0x`→`00-brief`, `ch1x`→`10-plan`, …). **You never number documents "round 17".** Rounds are not structure; stages are.

---

## Install

The skill is a folder with a `SKILL.md`. There is nothing to compile and nothing to depend on.

**Claude Code / any harness reading `~/.claude/skills`**

```bash
git clone https://github.com/<you>/ledger.git
cp -r ledger/skills/ledger ~/.claude/skills/ledger
```

**Project-local (checked into the repo, shared with the team)**

```bash
mkdir -p .claude/skills
cp -r ledger/skills/ledger .claude/skills/ledger
```

**Codex / Cursor / Gemini CLI / OpenCode / Kimi** — every one of them reads the same `SKILL.md` frontmatter. Drop the folder into that harness's skills directory; the tool is a plain Node script that runs anywhere.

```bash
# the CLI has zero dependencies; put it wherever your PATH can see it
cp ledger/tools/ledger.mjs ~/bin/ledger
chmod +x ~/bin/ledger
ledger --help
```

Verify the install actually works:

```bash
mkdir -p /tmp/ledger-demo && cd /tmp/ledger-demo
ledger init demo --goal "verify the install" --expect 6 --trace
ledger book init demo --title "Install check" --goal "verify" \
  --ch "ch00|Brief|goal|goal" --ch "ch40|Conclusion|the answer|answer"
ledger book touch demo ch40 --answer "does it work"
ledger book check demo     # -> PASS
ledger book root           # -> PASS
```

---

## Use

```bash
# 1. create the task directory (skipped entirely for small tasks — see below)
ledger init search-cache --goal "decide the cache invalidation strategy" --expect 8 --trace

# 2. write the whole chapter table up front. Bodies may be empty.
ledger book init search-cache --title "Cache invalidation" --goal "pick a strategy" \
  --ch "ch00|Brief|constraints and acceptance|goal" \
  --ch "ch10|Options|the three candidates|candidates" \
  --ch "ch30|Evidence|benchmarks and failure modes|bench" \
  --ch "ch40|Conclusion|the one decision|decision"

# 3. work chapter by chapter: create the file, then write the body
ledger book touch search-cache ch30 --answer "what did the benchmark actually show"

# 4. register every output, twice (file ledger + the chapter's output table)
ledger add search-cache out/bench.json --does "p99 numbers" --for "ch30" --verify "cat out/bench.json"

# 5. gates
ledger book check search-cache
ledger book orphans search-cache
ledger book root
ledger artifact guard search-cache --max-mb 200
ledger close search-cache
```

Retrieval — the part you will use every day:

```bash
ledger book find cache        # reads only BOOK.md indexes, never chapter bodies
```

```
[ledger] 2 chapter(s) across 2 book(s)

## search-cache
  | ch30 | Evidence | benchmarks and failure modes | bench | `30-evidence/ch30.md` | done |
  | ch40 | Conclusion | the one decision | decision | `40-deliver/ch40.md` | done |

>> Read only the chapter bodies you hit. Do not scan directories.
```

Two files. Not a repository scan.

---

## When to use it — and when not to

This is the part most workflow tools get wrong by insisting you always use them.

```
New task
  ├─ ≤3 output files, one-shot           -> do NOT create a ledger. Just work.
  ├─ ≤3 outputs but needs a paper trail  -> init only (no book)
  ├─ >3 outputs or >3 steps              -> init + book
  └─ must be retrievable across sessions -> init + book, keywords filled in
```

| Good fit | Bad fit |
|---|---|
| Multi-step work that may be interrupted and resumed | Changing one line of config |
| A conclusion people will re-ask about | A one-shot lookup |
| Work handed to another person or another session | Pure build-and-ship with no intermediates |
| A shared config change that needs a rollback path | Exploratory thrash that will be rewritten |
| Outputs that multiply copies and big files | A mechanical find-and-replace |

**Staged exit.** The workflow is not for the whole lifecycle:

```
exploration (outcome unknown) -> no book; write it once the answer settles
convergence (direction set)   -> init + book, fill chapters
delivery                      -> check + orphans + guard + close
```

**Signals to stop using it**: three consecutive `check` failures that are all "forgot to touch" (demote to manifest-only); the same chapter rewritten more than twice (you built the book too early); ledger overhead above 30% of the task's total output (the task was too small — don't init next time).

---

## Why not just a long `CLAUDE.md` / `AGENTS.md`?

Because prose cannot fail.

| | A rules file | Ledger |
|---|---|---|
| Enforcement | The model may comply | `check`, `orphans`, `root`, `guard`, `close` exit non-zero |
| Can you tell afterwards whether it was followed? | No | Yes — unregistered file, orphan output, missing chapter |
| Retrieval cost | Grows with the project | Constant-ish: the index layer, then one chapter |
| Newcomer onboarding | Read the whole rules file | Read one `BOOK.md` |
| Survives context compaction | Often lost | On disk |

The trade is explicit: Ledger costs a 1.2 KB ledger per non-trivial task, and it buys a machine-verifiable "done".

---

## Compared to other skills repositories

Most skill repositories are **collections** — they add capabilities. Ledger is a **substrate** — it changes how the other work is recorded, so it composes with them instead of competing.

| | [`anthropics/skills`](https://github.com/anthropics/skills) | [`obra/superpowers`](https://github.com/obra/superpowers) | [`alirezarezvani/claude-skills`](https://github.com/alirezarezvani/claude-skills) | **Ledger** |
|---|---|---|---|---|
| What it is | Reference implementations of skills | A software-development methodology | 434 skills across 21 domains | A traceability workflow |
| Scope | Per-task, many domains | One project's dev lifecycle (spec → plan → TDD → review) | Many domains, many roles | Cross-domain, cross-session |
| Enforcement | Guidance | Mandatory workflows, but prompt-level | Guidance + test harness | **5 commands that exit non-zero** |
| Persistence | Session | Git branch + plan files | Session | **On-disk ledgers, per task** |
| Retrieval story | — | Plan files per branch | Skill descriptions | **Index first, then one chapter** |
| Admits its own failures | n/a | Partly (diagnosing skill) | Reports load-test results | **Yes, with the numbers — 26/83 empty shells** |
| Best paired with | Everything | Project work | Domain expertise | Any of the above |

> Ledger does not give your agent new abilities. It makes the abilities it already has **accountable**.

---

## Repository layout

```
skills/ledger/SKILL.md     the skill (this is what you install)
tools/ledger.mjs           the CLI: init/add/close/sync, book *, artifact *  (zero deps)
docs/COMPARISON.md         longer comparison against other workflow approaches
docs/DESIGN-NOTES.md       why the threshold exists, with the measurements
README.zh-CN.md            Chinese edition (this English file is authoritative)
LICENSE                    MIT
```

---

## Known limitations

Stated up front, because a workflow that hides its failure modes is a workflow you cannot trust.

1. **No runtime enforcement.** Everything is prompt-level discipline plus commands the agent must choose to run. An agent can ignore all of it and nobody stops it. There is no hook layer here.
2. **Small tasks pay too much.** Below the threshold a ledger is pure overhead. The threshold is the fix; it is not a complete fix.
3. **Index rot is only detected, not prevented.** `check` warns when a chapter is newer than `BOOK.md`. Updating the summary is still manual.
4. **Chapter numbers are permanent.** Reordering is not supported; chapters can be appended via `book add`, never renumbered. This is a deliberate trade of refactor convenience for stable references.
5. **Heuristic artifact classification.** `artifact clean` decides keep/delete by extension and filename. It can be wrong silently. That is why it is dry-run by default and why every decision prints a reason. (`backup/` is exempt — rollback originals are never candidate for deletion.)
6. **Two ledgers can drift.** `MANIFEST.md` and `BOOK.md` are written by different commands; `check` warns about drift but does not block.

---

## License

MIT. See [LICENSE](LICENSE).
