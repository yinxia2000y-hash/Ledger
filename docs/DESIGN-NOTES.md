# Design notes

Why the threshold exists, and what the numbers behind it were.

---

## The mistake this workflow made first

The first version of Ledger had this rule:

> **Every task, even a one-line change, must create a ledger before any work begins.**

It sounds disciplined. It is defensible in a code review. It was wrong, and the way we found out was by counting.

### The measurement

Across 87 task directories, we compared each task's `MANIFEST.md` size against the task's total output:

| Task | Ledger | Total output | Ledger share |
|---|---:|---:|---:|
| `xxl-knowledge-book` | 3008 B | 3008 B | **100%** |
| `xxl-hv-bomb-release` | 2953 B | 2953 B | **100%** |
| `susfs-kernel` | 2083 B | 2083 B | **100%** |
| `clear-38-47-226-42-logs` | 2048 B | 2048 B | **100%** |
| `ace-detector` | 2031 B | 2031 B | **100%** |
| `task-manifest-20261111` | 2300 B | 2378 B | 97% |
| `cb-cli-checkin-all-20261118` | 2431 B | 4379 B | 56% |
| `env-triage-fix-20261111` | 2108 B | 3853 B | 55% |
| `apkgg-cleanup-20261123` | 2463 B | 5132 B | 48% |

**26 of 83 ledgers (31%) were more than 20% of the task's total output. Five were 100%.**

For five tasks, the ledger *was* the deliverable. The workflow had produced paperwork about work that either did not exist or did not need recording.

### Why the rule survived as long as it did

Because the cost is invisible in review. A `MANIFEST.md` with three filled rows looks like care. You only see the problem when you compare it against what else the task produced — and nobody does that unless they set out to.

This is the general lesson, and it outlives the specific rule:

> **A process rule must be measured against the output it produces, not against how it looks while it is being followed.**

---

## The repair

The rule became:

```
<=3 output files AND no need for later retrieval  ->  do not create a ledger
```

And `init` now refuses by default in that case, printing the reason and the measurement:

```
[ledger] skip: 2 output file(s), no need for traceability
  Work directly and state the artifact path in your reply.
  Measured: 26 of 83 ledgers were >20% of their task's total output.
  Force it with --force if you disagree.
```

The refusal is a suggestion with a paper trail, not a wall. `--force` exists because the estimate can be wrong, and a tool that cannot be overridden gets worked around instead of corrected.

### Why the threshold is a refusal rather than an error

An error would imply the user is wrong. The user may know something the estimate does not: that the output is small but will be cited for months. A refusal that prints the trade-off lets the user disagree with full information. Exit code 0, because nothing failed.

---

## Why three ledgers and not one

One file would be simpler. The split exists because the three questions have different lifetimes.

| Ledger | Answers | Changes | Fails independently |
|---|---|---|---|
| `MANIFEST.md` | Who made this, why, who reads it | Every time a file is created | Files pile up unregistered |
| `BOOK.md` | Which chapter has my answer | On chapter creation and (ideally) on every conclusion change | Index goes stale while bodies move on |
| artifact rules | Keep, register, or delete | Rarely | A heuristic silently deletes the wrong thing |

Collapsing them means one of these failure modes gets hidden by the others. They are also written by different commands at different moments, which is why `check` has to warn about drift between them rather than assert equality.

The drift is real: `MANIFEST.md` can register `out/x.log` while no chapter claims it. `check` prints a warning; `close` passes, because `close` only looks at `MANIFEST.md`. Two ledgers, two opinions, one warning. That is a **known limitation**, not a design goal — the fix was scoped out because making `close` read `BOOK.md` changes what `close` means.

---

## Why chapters cannot be renumbered

A chapter number is a citation. Conclusions point at evidence with `ch30`, `ch40`, and those pointers live in prose that nothing validates.

If numbers were reusable, a renumber would silently repoint every citation. The failure is not "the tool errors", it is "a conclusion now cites the wrong evidence and nobody notices for months". Renumbering is therefore treated the same way as rewriting git history: possible in principle, forbidden by convention, and unsafe without a global rewrite of every reference.

The cost is real: get the plan wrong and you must append a chapter out of order, or start a second book. Both happened here. The second is worse — two half-books on one subject — and it was accepted rather than fixed.

---

## Why artifact classification still prints a reason

`artifact clean` decides keep/delete from an extension list and a filename regex. Heuristics of this kind fail silently, and a silent delete is unrecoverable.

Two examples from the reference implementation:

1. **`dex/so/dll/class` were classified as build output and kept.** They had been listed as retained deliverables. Reclassifying them as regenerable raised one command's reclaim from 439 KB to 80.1 MB. The wrong rule never errored; it just quietly kept files nobody needed.
2. **76 AI keyframes totalling 133 MB were almost deleted.** The original rule classified by directory. The frames lived in a working directory, so all 76 were "process material". The rule that saved them was checking the filename for `keyframe` first.

Neither fix is obviously correct in general. A filename containing `keyframe` is not proof of value; a directory named `tmp` is not proof of worthlessness. The classification is a heuristic and is documented as one, which is why every decision prints a reason and why `clean` is dry-run unless `--apply` is passed.

**Fixed in the open-source CLI**: `artifact clean` classified by file extension alone, so a
`backup/old.log` was "process material" and a `.orig` only survived by falling through to
"unclassified". The classifier now returns early for anything under `backup/`, which makes it
agree with `book check`, `book orphans` and `book root` — all three already exempted the
directory. The gates disagreeing about what is safe to delete is the actual bug; the deleted
`.orig` was the symptom.

---

## Why no runtime enforcement

Every rule here lives in a prompt and a CLI that the agent must choose to invoke. There is no hook.

This is the most serious limitation and it is worth being blunt about: **an agent can ignore the entire workflow and nothing will stop it.** What the gates buy is not compliance; it is *detection after the fact*. A `check` failure is evidence that discipline was skipped, which is more than a rules file gives you, and less than a hook would.

The alternative — a hook that blocks file writes outside a task directory — was not built because it requires integrating with each harness's hook API, and every such integration is a permanent maintenance cost for a benefit that only exists while the integration works.

---

## What would be built next, in order

1. **~~`artifact clean` must exempt `backup/`~~.** Done in this release. The gates now agree.
2. **A disambiguation chapter.** When the goal or acceptance criteria are ambiguous, write the ambiguity down with the assumption you chose and what breaks if it was wrong. Cost: one table per ambiguous task. Benefit: in three months, "why did we do it this way" has an answer that is not the transcript.
3. **Evidence edges.** Each conclusion line names the chapters it rests on. Cost: three chapter numbers. Benefit: from a conclusion you can jump to the evidence without searching.
4. **A `close` that reads `BOOK.md`.** Removes the two-ledger drift warning by making it a hard check. Small change, changes what `close` means, needs a migration note.

Everything else — visualisation, phase-based context switching, an evidence graph — is speculative. The measured problems are the four above.
