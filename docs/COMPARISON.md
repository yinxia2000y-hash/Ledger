# Ledger compared

A longer version of the README's comparison table. Written to be useful when you are choosing between approaches, not to win an argument.

---

## The four things being compared

**A rules file** — `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, or a long system prompt. The default. Costs nothing to adopt.

**A methodology skill pack** — e.g. [`obra/superpowers`](https://github.com/obra/superpowers). Takes a position on *how software work should be done* (spec → plan → TDD → review) and ships skills that enforce it through mandatory workflows.

**A skills collection** — e.g. [`alirezarezvani/claude-skills`](https://github.com/alirezarezvani/claude-skills) (434 skills, 21 domains) or [`anthropics/skills`](https://github.com/anthropics/skills) (reference implementations). Adds capabilities across many domains; organisation is the user's problem.

**Ledger** — a traceability substrate. Says nothing about *what* you should build, only about how the work and its conclusions are recorded so they can be found and audited.

These are not mutually exclusive. Ledger is designed to sit underneath a collection or a methodology, not to replace one.

---

## Dimension by dimension

### 1. What enforces it

| | Rules file | Methodology pack | Collection | Ledger |
|---|---|---|---|---|
| Enforcement mechanism | Model compliance | Mandatory workflows, prompt-level | Model compliance + load tests | **Commands that exit non-zero** |
| Can enforcement fail silently? | Yes | Yes | Yes | **Some of it** — see §8 |
| What blocks "done"? | Nothing | A workflow step the model may skip | Nothing | `check` / `orphans` / `root` / `guard` / `close` |

The honest framing: Ledger moves *some* of the enforcement from "the model remembers" to "the shell disagrees with you". It does not move all of it. A model that refuses to run the commands is still unconstrained, and there is no hook layer here.

### 2. What it costs

| | Rules file | Methodology pack | Collection | Ledger |
|---|---|---|---|---|
| Adoption cost | Zero | Install + a behavioural change | Install | Install + copy a 27 KB script |
| Per-task cost | Tokens in the prompt | A spec/plan artifact per task | Skill descriptions in context | ~1.2 KB ledger, only above the threshold |
| Cost scaling | Grows with the rules file | Per task | Per skill loaded | Flat per task; retrieval is index-first |

### 3. What it optimises for

| | Optimised for |
|---|---|
| Rules file | Nothing in particular; lowest friction |
| Methodology pack | Quality of a software change (TDD, review, spec) |
| Collection | Breadth of capability |
| **Ledger** | **Finding an old conclusion, and proving work was finished** |

### 4. Scope

| | Scope |
|---|---|
| Rules file | Whatever you wrote |
| Methodology pack | One software project's development lifecycle |
| Collection | Many domains, per task |
| **Ledger** | **Any task, spanning sessions** |

---

## Where Ledger is strictly worse

Say this out loud or the comparison is marketing.

1. **It gives your agent no new ability.** A collection makes an agent able to do something it could not do. Ledger makes it write down what it did. If your problem is "the agent cannot do X", Ledger is the wrong tool.
2. **It is slower on small work.** A 1.2 KB ledger on a one-line change is a loss. That is exactly why the threshold exists, and why "do not use Ledger" is a legitimate outcome of reading §6.
3. **It imposes structure on exploration.** If you do not yet know whether the work will be three files or thirty, building a book up front means rewriting chapters. The guidance is: do not build the book until the answer starts to settle.
4. **Chapter numbers being permanent is genuinely annoying.** You will get the plan wrong sometimes and want to reorder. You cannot. You append.
5. **No visualisation.** It is markdown and a CLI. There is no dashboard.

---

## Where Ledger is strictly better

1. **Answers "why did we decide that" without a transcript.** Index first, then one chapter. Measured at three orders of magnitude less reading than a project-wide grep.
2. **Makes completion falsifiable.** `book check` exiting 1 means a file exists that no chapter claims. That is a class of bug you cannot see by reading a diff.
3. **Survives compaction.** The ledgers are files. When context is dropped, the next session reads `BOOK.md` and continues; it does not re-derive.
4. **Onboards a newcomer in one read.** Point them at `BOOK.md`, not at 400 turns of history.
5. **Reports its own failures with numbers.** The 26/83 empty-shell finding is in this repository because the workflow was measured against itself and lost. A workflow that cannot be measured against itself cannot be improved.

---

## Combinations that work

| Combination | Why it works |
|---|---|
| Ledger + a collection | The collection adds capability; Ledger makes the *outputs* of that capability findable. |
| Ledger + a methodology pack | The methodology governs how a change is made; Ledger records what the change concluded. They overlap at "plan" — use the methodology's plan as the chapter body, not a second document. |
| Ledger + nothing | A solo agent doing multi-session work where you are the only reader. |

## Combinations that do not

| Combination | Why it fails |
|---|---|
| Ledger over a one-line config change | Overhead approaches 100% of the output |
| Ledger during a spike you intend to throw away | Chapters rewritten more than twice |
| Two books for one subject | Chapter numbers cannot be merged; you get two half-books. Measured: this happened once here, and the accepted cost was to keep both. |

---

## If you only read one row

> Use the collection to give your agent more to do. Use a methodology pack to make it do that work well. Use Ledger so that six weeks later you can find out what it decided and prove it finished.
