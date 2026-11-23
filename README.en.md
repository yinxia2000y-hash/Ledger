<div align="center">

# Ledger

**Your AI finished the job. Can you still find what it did three weeks later?**

A ledger for AI work. Three books, five checks, and a "done" you can actually prove.

[![Type](https://img.shields.io/badge/type-AI%20skill-4B32C3?style=flat-square)](./skills/ledger/SKILL.md)
[![Deps](https://img.shields.io/badge/deps-zero-2EA043?style=flat-square)](./tools/ledger.mjs)
[![Node](https://img.shields.io/badge/Node-18%2B-339933?style=flat-square&logo=node.js&logoColor=white)](./tools/ledger.mjs)
[![License](https://img.shields.io/badge/license-MIT-0366D6?style=flat-square)](./LICENSE)

**Language**　[中文 ←](./README.md) · **English**

</div>

---

## Plain talk first

You have probably hit at least one of these.

**One: the answer is gone.** The AI figured it out and told you. Two weeks later you want to double-check. You scroll back through the chat, can't find it, and ask it to redo the work — worse than the first time.

**Two: too many files, none of them true.** `report.md`, `report_v2.md`, `report_final.md`, `report_final_2.md`. Four files, four different answers, nobody knows which one counts.

**Three: every question triggers a full-disk search.** You ask "what did we decide about caching?" It greps 115,000 files, takes a while, and comes back with something you remembered better yourself.

**Four: it says "done" and you have no idea.** No checklist, no verification. You take its word. Two days later you find half of it missing.

You cannot fix these by *asking the AI to be more careful*. Will it comply? Maybe. And afterwards you cannot tell whether it did.

> **Ledger does one thing: turns "be careful" into "you have to be careful, or something errors out".**

---

## What it actually saves you

These are not marketing numbers. They come from 87 real task directories.

<div align="center">

| | Before | After |
|:--|:--|:--|
| Files to look at, to answer one question | about **115,000**, 40+ GB | an index of **14 KB**, then **one file** |
| How much faster | full-disk search | **~1,000×** (three orders of magnitude) |
| Tasks that can catch stray unregistered files | 9 of 83 (**11%**) | **100%** |
| Boilerplate on a small job | 2 KB minimum, sometimes **the entire output** | 1.2 KB, and **not created at all** for small jobs |
| Tasks whose only output was the paperwork | **26 of 83** | **0**, structurally |

</div>

> [!IMPORTANT]
> **The last two rows are the point: version one of this thing was wrong, and we measured it ourselves.**
>
> The original rule was "no matter how small the job, create a ledger before you start." Sounds rigorous.
> The result: **26 of 87 tasks** ended with the ledger as their only output. Five of them were 100% paperwork.
>
> So there is now a threshold. Small job, just do it. **That is a bug fix, not a compromise.**

---

## Three books, one job each

<div align="center">

| Book | File | The question it answers | Who checks it |
|:--|:--|:--|:--|
| 📒 **File book** | `MANIFEST.md` | Who made this, what for, who reads it, how to verify? | `close` — unregistered file, hard error |
| 📖 **Chapter book** | `BOOK.md` | Which chapter has my answer? | `book check` — strays, duplicate numbers, stale index |
| 🗄️ **Artifact book** | `ledger artifact` | Keep it, note the path, or delete it? | `clean` is dry-run by default; `guard` watches size |

</div>

<details>
<summary><b>What the folders look like (click)</b></summary>

```
_work/task-name/
├── MANIFEST.md          file book
├── BOOK.md              chapter book — the only thing retrieval reads
├── 00-brief/ch00.md     the ask
├── 10-plan/ch10.md      the plan
├── 20-exec/ch20.md      the work
├── 30-evidence/ch30.md  the proof
├── 40-deliver/ch40.md   the one conclusion
├── out/                 real deliverables
├── backup/              originals before you changed things — never deleted
└── tmp/                 scratch, must be empty when you finish
```

Folder names are computed from the chapter number: `ch0x` → `00-brief`, `ch1x` → `10-plan`, and so on.

**Never name files by round.** "round-17-notes.md" is not structure. Stages are.

</details>

### What finding something feels like

```console
$ ledger book find cache
[ledger] 2 chapter(s) across 1 book(s)

## search-cache
  | ch30 | Evidence   | benchmarks and failure modes | bench    | 30-evidence/ch30.md | done |
  | ch40 | Conclusion | the one decision             | decision | 40-deliver/ch40.md  | done |

>> Read only the chapter bodies you hit. Do not scan directories.
```

**Two files. Not a disk scan.**

---

## Five checks that will stop you

Not suggestions. **They exit non-zero.**

<div align="center">

| # | Command | What it checks | When it fails |
|:-:|:--|:--|:--|
| 1 | `book check` | duplicate chapter numbers, chapters claimed but missing, stray files, index over 200 lines | **error, blocked** |
| 2 | `book orphans` | same stray-file check, but **works without a chapter book** | **error, blocked** |
| 3 | `book root` | loose files dumped at the parent level | **error, blocked** |
| 4 | `artifact clean` | what to delete vs keep, **with a reason printed for every file** | dry-run by default |
| 5 | `close` | unregistered files, leftover scratch | **error, blocked** |

</div>

**How it decides on artifacts:** failed build → delete (keep the report) · success under 50 MB → keep · success over 50 MB → delete but record the path · **any `.md` report → always keep**.

**How it decides on images:** filename contains `keyframe`, `reference`, `contact sheet` → a deliverable, **keep** · contains `screen`, `shot`, `capture`, or lives in `tmp/` → analysis scratch, **delete** · unrecognised → **keep, to be safe**.

> A real case: **76 AI keyframes (133 MB)** were nearly wiped because the old rule looked at the folder, not the filename.
> Switching to filename-based checking saved every one.
>
> Which is why this is a **guess**, why every decision prints its reason, and why `clean` does not delete unless you say so.

---

## When to use it, when to skip it

Most tools tell you to always use them. That is wrong.

```
A new job
  ├─ under 3 outputs, one and done   → no ledger, just do it
  ├─ under 3 outputs, but needs a record → file book only
  ├─ over 3 outputs, or over 3 steps → file book + chapter book
  └─ must be findable weeks later    → book, with keywords filled in
```

<div align="center">

| ✅ Use it (you save time) | ❌ Skip it (pure loss) |
|:--|:--|
| Work that may be interrupted and resumed | Changing one line of config (overhead can hit **100%**) |
| A conclusion people will ask about again | A one-off lookup |
| Handing off to a person, or to next week's session | Build-and-ship with nothing in between |
| Touching shared config, needing rollback | Experiments whose answer you will probably throw away |
| Outputs that multiply copies and big files | A mechanical find-and-replace |

</div>

**Use it in stages, not end to end:**

```
Poking around (no idea yet) → skip the chapter book, let tmp/ be messy, write it once the answer lands
Direction settled            → file book + chapter book, fill chapters
About to deliver            → run all five checks
```

**Signals to stop:** three failures in a row that are all "forgot to add the chapter" means it is not helping you — drop to file-book only · the same chapter rewritten more than twice means you built the book too early · overhead above 30% of total output means the job was too small, do not create one next time.

---

## Why not just write a long CLAUDE.md

**Because prose cannot fail.**

<div align="center">

| | A pile of rules<br>`CLAUDE.md` / `AGENTS.md` | **Ledger** |
|:--|:--|:--|
| How it is enforced | if the model feels like it | **five commands that exit non-zero** |
| Can you tell afterwards whether it complied | no | yes — strays, orphans, missing chapters all surface |
| Cost of looking something up | grows with the project | roughly flat: index, then one chapter |
| Onboarding a newcomer | read the whole rules file | read one `BOOK.md` |
| Survives a long chat getting compacted | rules often lost | the books are on disk |

</div>

The cost, stated plainly: about **1.2 KB** of bookkeeping per real job, in exchange for a **"done" that can be checked**.

---

## How it compares

Most skill repos are **toolboxes** — they give the AI new abilities.
Ledger is **bookkeeping** — it governs how the rest of the work gets recorded. So it **stacks**, it does not compete.

<div align="center">

|  | [anthropics/skills](https://github.com/anthropics/skills)<br>`199k ★` | [obra/superpowers](https://github.com/obra/superpowers) | [alirezarezvani/claude-skills](https://github.com/alirezarezvani/claude-skills)<br>`434 skills` | **Ledger** |
|:--|:--|:--|:--|:--|
| **What it is** | official skill examples | a development process | a big pile of skills | **a bookkeeping process** |
| **Scope** | per task, many domains | one project's lifecycle | many domains, many roles | **cross-domain, cross-session** |
| **Enforcement** | trust | process (still prose) | trust + load tests | **five commands that error** |
| **Where work lives** | gone when the chat ends | git branch + plan files | gone when the chat ends | **on disk, per task** |
| **How you find things** | — | plan files per branch | skill descriptions | **index first, then one chapter** |
| **Admits its weak spots** | n/a | somewhat | publishes load tests | **yes, with numbers: 26/83** |
| **Pairs with** | — | project work | domain expertise | **any of the above** |

</div>

### And where Ledger is worse

<details>
<summary><b>Five hard truths (click)</b></summary>

| # | Where it is worse | What that means |
|:-:|:--|:--|
| 1 | **It adds no new ability** | if your problem is "the AI can't do this", Ledger is the wrong tool |
| 2 | **Slower on small jobs** | 1.2 KB of bookkeeping for a one-line change is a loss (hence the threshold) |
| 3 | **Gets in the way while exploring** | before you know if this is 3 files or 30, a chapter book gets rewritten |
| 4 | **Chapter numbers are permanent, and annoying** | get the plan wrong and you can only append |
| 5 | **No UI** | text and a command line, no dashboard |

Five more implementation-level limits are further down.

</details>

> The one row nobody else has: **admits its weak spots, with numbers.**
> That is not modesty. It proves two opposite things at once — this has been used for real, and measured for real.

---

## Install in a minute

No dependencies. Node 18+. Nothing to compile.

**Step one, the skill:**

```bash
git clone https://github.com/yinxia2000y-hash/Ledger.git
cp -r Ledger/skills/ledger ~/.claude/skills/ledger
```

**Step two, the command line tool:**

```bash
cp Ledger/tools/ledger.mjs ~/bin/ledger && chmod +x ~/bin/ledger
ledger --help
```

**Step three, prove it works** (copy-paste; this is exactly how we tested it):

```bash
mkdir -p /tmp/ledger-demo && cd /tmp/ledger-demo
ledger init demo --goal "verify the install" --expect 6 --trace
ledger book init demo --title "Install check" --goal "verify" \
  --ch "ch00|Brief|goal|goal" --ch "ch40|Conclusion|the answer|answer"
ledger book touch demo ch40 --answer "does it work"
ledger book check demo
ledger book root
```

Two `PASS` lines and you are done.

<details>
<summary><b>Other tools (Codex / Cursor / Gemini CLI / OpenCode / Kimi)</b></summary>

They all read the same `SKILL.md`. Drop the folder in:

```bash
mkdir -p .claude/skills && cp -r Ledger/skills/ledger .claude/skills/ledger
```

The CLI is a plain Node script and runs anywhere.

</details>

---

## How to use it

<details>
<summary><b>The full five steps (click)</b></summary>

```bash
# 1. create the task folder (it will talk you out of it if the job is small)
ledger init search-cache --goal "decide the cache invalidation strategy" --expect 8 --trace

# 2. write the whole chapter list at once. Bodies can be empty.
ledger book init search-cache --title "Cache invalidation" --goal "pick a strategy" \
  --ch "ch00|Brief|constraints and acceptance|goal" \
  --ch "ch10|Options|the three candidates|candidates" \
  --ch "ch30|Evidence|benchmarks and failure modes|bench" \
  --ch "ch40|Conclusion|the one decision|decision"

# 3. one chapter at a time: create the file, then write it
ledger book touch search-cache ch30 --answer "what did the benchmark actually show"

# 4. register every output twice (file book + that chapter's output table)
ledger add search-cache out/bench.json --does "p99 numbers" --for "ch30"

# 5. run the five checks
ledger book check search-cache
ledger book orphans search-cache
ledger book root
ledger artifact guard search-cache --max-mb 200
ledger close search-cache
```

**How status moves:** `pending` → (file created) → `writing` → (conclusion written) → `done`. Chapters you drop are marked `dropped`, and **a number is never reused**.

Forgot a chapter? `ledger book add task-name --ch "ch11|..."` — append yes, renumber no.

</details>

<details>
<summary><b>Keep or delete an artifact</b></summary>

| Artifact | What to do |
|:--|:--|
| failed apk/ipa/dex/img/dump | delete, keep only the `.md` report |
| success, under 50 MB | keep |
| success, over 50 MB | delete the file, record path and size |
| `dex/so/dll/class` (regenerable from a package) | delete |
| dump / decompile / logs / throwaway scripts | delete |
| **any `.md` report or analysis data** | **always keep** |

Rollback originals in `backup/` **never enter the delete list** (fixed in this version — the three checks used to disagree).

</details>

---

## Known problems

Nothing hidden. **A tool that will not admit where it falls short is not worth trusting.**

| # | Problem | Status |
|:-:|:--|:--|
| 1 | bookkeeping can exceed the output on small jobs (26/83 measured, 5 at 100%) | ✅ **fixed** — threshold added |
| 2 | the file book template is one fixed block | ✅ **trimmed** — 1.2 KB, old books refreshable |
| 3 | the chapter book is used on only 11% of tasks | ⚠️ **cannot force it** — it is a judgement call |
| 4 | the two books can disagree | ⚠️ **detectable** (warns, does not block) |
| 5 | body edited, index left stale | ⚠️ **detectable** (warns, does not block) |
| 6 | chapter numbers are permanent, refactoring is painful | 🔒 **accepted** — stable references beat convenience |
| 7 | three checks were dead for 89% of tasks without a chapter book | ✅ **fixed** — `book orphans` |
| 8 | delete decisions are guessed, and a wrong guess is silent | 🔒 **accepted** — every decision prints its reason, dry-run by default |
| 9 | **all of it is prose-level, with no runtime enforcement** | ❌ **cannot fix** — the AI can ignore the whole thing |

Number nine is the serious one. **The AI can ignore this entire workflow and nothing stops it.** The five checks buy you "you can tell afterwards", not "it is guaranteed to comply".

<details>
<summary><b>Why there is no hard interceptor</b></summary>

Because every tool has its own interception API, each one costs forever to maintain, and it only works while that integration survives. A half-working interceptor that still lets things through is worse than solid after-the-fact checking.

</details>

---

<div align="center">

**If this ever helped you find a conclusion you had lost, a ⭐ is the best feedback.**

[↑ Back to top](#ledger) · [中文 ←](./README.md)

</div>
