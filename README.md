<div align="center">

# Ledger

**Agent 写完了。三周后你还找得到吗？**

一个给 AI Agent 用的**账本式工作流**：三本账 + 五道闸门，让「做完了」变成机器能验证的事实。

[![Type](https://img.shields.io/badge/type-agent%20skill-4B32C3?style=flat-square)](./skills/ledger/SKILL.md)
[![Deps](https://img.shields.io/badge/dependencies-0-2EA043?style=flat-square)](./tools/ledger.mjs)
[![Node](https://img.shields.io/badge/node-%E2%89%A518-339933?style=flat-square&logo=node.js&logoColor=white)](./tools/ledger.mjs)
[![License](https://img.shields.io/badge/license-MIT-0366D6?style=flat-square)](./LICENSE)
[![Docs](https://img.shields.io/badge/docs-%E4%B8%AD%E6%96%87%20%7C%20English-EA4AAA?style=flat-square)](#english)

**🌐 语言 / Language**　**中文**（当前） · [English ↓](#english)

</div>

---

## 目录

| | | |
|---|---|---|
| [**问题**](#问题) | [**实测**](#实测不是断言) | [**它做什么**](#三本账) |
| [**五道闸门**](#五道闸门) | [**什么时候用**](#什么时候用-什么时候别用) | [**为什么不用一份长 CLAUDE.md**](#为什么不用一份很长的-claudemd) |
| [**横向对比**](#和主流方案横向对比) | [**安装**](#安装60-秒) | [**已知局限**](#已知局限) |

---

## 问题

长会话的失败方式只有四种，而且都**无法靠"让 Agent 认真点"解决**。

<table>
<tr><th width="22%">症状</th><th width="48%">具体表现</th><th width="30%">代价</th></tr>
<tr><td>🔍 <b>结论丢失</b></td><td>你知道 Agent 想通了，但答案在 400 轮之前的对话里，某个地方</td><td>重新推导，而且推得更差</td></tr>
<tr><td>📁 <b>文件名腐烂</b></td><td><code>report.md</code> → <code>report_v2.md</code> → <code>report_final.md</code> → <code>report_final_2.md</code></td><td>没人知道哪份是真的</td></tr>
<tr><td>💥 <b>上下文爆炸</b></td><td>为了回答「当时怎么定缓存的」，Agent 去 grep 11.5 万个文件</td><td>时间、token，以及比读书还慢的答案</td></tr>
<tr><td>✅ <b>「完成」不可证伪</b></td><td>Agent 说做完了。没有任何东西检查这件事</td><td>交付物里有静默的空洞</td></tr>
</table>

通常的做法是加长提示词：*"把工作整理好。"* 那是**建议**。建议的执行是不均匀的，事后你也看不出到底有没有执行。

> **Ledger 的做法：把纪律变成「有结构的文件」+「违规就非零退出的命令」。**

---

## 实测，不是断言

以下数字来自参考实现对 **87 个真实任务目录**的统计。

<div align="center">

| 指标 | 改之前 | 改之后 |
|:--|:--|:--|
| 回答一个问题要读的文件 | 约 **115,000** 个文件 / 扫 **46 GB** | 所有 `BOOK.md` 索引 **≈14 KB**，再读 **1 个**章节文件 |
| 检索成本 | 全项目 grep | **小三个数量级** |
| 有「孤儿产出」闸门的任务 | 9 / 83（**11%**） | **100%** |
| 小任务上的台账开销 | 2.1 KB 固定模板，最高占**总产出 100%** | 1.2 KB，门槛以下**根本不建** |
| 工作流自己造出的空壳任务 | 26 / 83（**31%** 台账占比 >20%） | 结构上为 **0** |

</div>

最后两行是这套东西最该看的部分。

> [!IMPORTANT]
> **Ledger 的第一个版本是错的，而且是我们自己量出来的。**
> 当时的规则是「任何任务，哪怕只改一行，都必须先建台账」。
> 这条规则制造了 **26 个「除了台账什么都没产出」的任务**，其中 5 个台账占 100%。
>
> [`SKILL.md` 第 6 节](./skills/ledger/SKILL.md) 的门槛是**修复**，不是装饰。

---

## 三本账

每本账对应一个**能被机器判定**的问题。

<div align="center">

| 账 | 文件 | 回答什么 | 谁来判定 |
|:--|:--|:--|:--|
| 📒 **文件账** | `MANIFEST.md` | 这文件谁建的、干什么、谁读它、怎么验证？ | `close` — 未登记即 exit 1 |
| 📖 **章节账** | `BOOK.md` | 我要的答案在**哪一章**？（≤200 行，摘要 + 关键词） | `book check` — 孤儿 / 章号重复 / 索引腐烂 |
| 🗄️ **产物账** | `ledger artifact` | 留着、只记路径、还是删掉？ | `clean` 默认 dry-run + `guard` 体积预算 |

</div>

<details>
<summary><b>磁盘布局（点击展开）</b></summary>

```
_work/<slug>/              ← 默认根；--root 或 $LEDGER_ROOT 可改
├── MANIFEST.md            文件账
├── BOOK.md                章节账 —— 唯一检索入口
├── 00-brief/ch00.md       ← 章节目录由章号推导
├── 10-plan/ch10.md
├── 20-exec/ch20.md
├── 30-evidence/ch30.md
├── 40-deliver/ch40.md     唯一的权威结论
├── out/                   真产物
├── backup/                改动前原件（回滚用，永不删）
└── tmp/                   探针；收尾必须清空
```

章节目录由编号推导：`ch0x`→`00-brief`，`ch1x`→`10-plan`，`ch2x`→`20-exec`，`ch3x`→`30-evidence`，`ch4x`→`40-deliver`。

**永远不要用「第 17 轮」给文档编号。** 轮次不是结构，阶段才是。

</details>

### 检索：找一次，只读一章

```console
$ ledger book find cache
[ledger] 2 chapter(s) across 1 book(s)

## search-cache
  | ch30 | Evidence   | benchmarks and failure modes | bench    | 30-evidence/ch30.md | done |
  | ch40 | Conclusion | the one decision             | decision | 40-deliver/ch40.md  | done |

>> Read only the chapter bodies you hit. Do not scan directories.
```

**两个文件。不是一次全库扫描。**

---

## 五道闸门

不是"建议"，是**会 exit 1 的检查**。

<div align="center">

| # | 命令 | 管什么 | 失败 |
|:-:|:--|:--|:--|
| 1 | `book check <slug>` | 任务内：章号重复、状态/文件不一致、孤儿产出、BOOK 超 200 行 | **exit 1** |
| 2 | `book orphans [slug]` | 任务内：**不需要建书**也能查未登记文件 | **exit 1** |
| 3 | `book root` | 任务之间：根目录只允许任务目录 | **exit 1** |
| 4 | `artifact clean` | 产物留/删/只记路径，**每条打印理由** | dry-run 默认 |
| 5 | `close <slug>` | 未登记文件 + `tmp/` 残留 | **exit 1** |

</div>

**产物三分法**：失败构建物 → 删（只留 `.md` 报告）· 成功 ≤50MB → 留 · 成功 >50MB → 删文件记路径 · **任何 `.md` 报告 → 永远留**。

**图片四分类**：`keyframe/关键帧/参考/联系表` → 交付资产**保留** · `screen/shot/capture` 或 `tmp/` → 分析截图**删** · 包内资源 → 删 · 认不出 → **保守保留**。

> 实测反例：某项目的 **76 张 AI 关键帧（133 MB）** 差点被全删——因为旧规则按目录判定。改成按文件名判定后一张不删。
> 分类是启发式的，所以**每条都打印理由**，且 `clean` 默认 dry-run。

---

## 什么时候用 — 什么时候别用

多数工作流工具在这里搞错：它们坚持你**永远**都该用它。

```
新任务
  ├─ 产出 ≤3 且一次性            → 不建账，直接干
  ├─ 产出 ≤3 但要留痕            → 只 init（不建书）
  ├─ 产出 >3 或步骤 >3           → init + 建书
  └─ 跨会话需要被检索            → init + 建书，关键词写全
```

<div align="center">

| ✅ 该用（ROI 为正） | ❌ 不该用（会亏） |
|:--|:--|
| 可能中断、要续上的多步工作 | 改一行配置（开销可达 **100%**） |
| 别人会反复追问的结论 | 一次性查询 |
| 要交给另一个人 / 另一个会话 | 纯构建交付、无中间态 |
| 改共享配置、要回滚路径 | 结果大概率被推翻的探索 |
| 会产生多副本和大文件 | 明确的机械替换 |

</div>

**分阶段退出**（工作流不是全生命周期都该在）：

```
探索期（结果不确定）→ 不建书，允许 tmp/ 乱放，结论稳了再一次性写
收敛期（方向已定）  → init + 建书，逐章补
交付期              → check + orphans + guard + close
```

**该停用的信号**：连续三次 `check` 失败都是「忘了 touch」→ 降级为只留文件账 · 同一章被重写超过两次 → 书建早了 · 台账开销 > 任务总产出 30% → 下次别 init。

---

## 为什么不用一份很长的 CLAUDE.md

**因为文字不会失败。**

<div align="center">

| | 规则文件<br>`CLAUDE.md` / `AGENTS.md` | **Ledger** |
|:--|:--|:--|
| 强制方式 | 模型愿意遵守 | **5 条命令非零退出** |
| 事后能看出有没有遵守吗 | ❌ 不能 | ✅ 未登记文件 / 孤儿产出 / 缺失章节 |
| 检索成本 | 随项目增长 | 近似常数：索引层 → 一章 |
| 新人上手 | 读完整份规则 | 读一个 `BOOK.md` |
| 上下文压缩后还在吗 | 经常丢 | ✅ 在磁盘上 |

</div>

代价是明确的：每个非微小任务付约 **1.2 KB** 台账，换来一个**可机器验证的「完成」**。

---

## 和主流方案横向对比

多数 skill 仓库是**能力集合**——增加 Agent 能做的事。
Ledger 是**底层规范**——改变其他工作的记录方式，所以它**可以叠加**，不是竞争。

<div align="center">

|  | [anthropics/skills](https://github.com/anthropics/skills)<br>`199k ★` | [obra/superpowers](https://github.com/obra/superpowers) | [alirezarezvani/claude-skills](https://github.com/alirezarezvani/claude-skills)<br>`434 skills` | **Ledger** |
|:--|:--|:--|:--|:--|
| **是什么** | 官方 skill 参考实现 | 一套软件开发方法论 | 海量能力集合 | **可追溯工作流** |
| **覆盖范围** | 按任务，多域 | 单项目开发周期 | 多域、多角色 | **跨域、跨会话** |
| **强制力** | 引导 | 强制工作流（提示词层） | 引导 + 加载测试 | **5 条命令 exit 1** |
| **持久化** | 会话内 | Git 分支 + 计划文件 | 会话内 | **磁盘上的账，按任务** |
| **检索方案** | — | 每分支计划文件 | skill 描述 | **先索引，再一章** |
| **公开自己的失败** | 不适用 | 部分 | 公开加载测试 | **公开，且带数字 26/83** |
| **最佳搭配** | — | 项目型开发 | 领域专长 | **上面任意一个** |

</div>

### 也要说清楚：Ledger 严格更差的地方

<details>
<summary><b>5 条硬伤（点击展开）</b></summary>

| # | 更差在哪 | 后果 |
|:-:|:--|:--|
| 1 | **不给 Agent 任何新能力** | 如果问题是「Agent 不会做 X」，Ledger 是错的工具 |
| 2 | **小任务上更慢** | 1.2 KB 台账对一行改动是净亏（这就是门槛存在的原因） |
| 3 | **限制探索期** | 不知道会产出 3 个还是 30 个文件时，建书要反复重写 |
| 4 | **章号永久，很烦** | 计划错了只能追加，不能重排 |
| 5 | **无可视化** | 纯 markdown + CLI，没有 dashboard |

还有 5 条实现层面的局限，见 [已知局限](#已知局限)。

</details>

> 对比表里唯一一列别的仓库没有的：**公开自己的失败，且带数字**。
> 这不是谦虚——它同时证明两件相反的事：这套东西被真实使用过，而且被真实测量过。

---

## 安装：60 秒

零依赖 · Node ≥18 · 无构建步骤。

**1. 装 skill**（Claude Code / 任何读 `~/.claude/skills` 的 harness）

```bash
git clone https://github.com/yinxia2000y-hash/Ledger.git
cp -r Ledger/skills/ledger ~/.claude/skills/ledger
```

**2. 装 CLI**

```bash
cp Ledger/tools/ledger.mjs ~/bin/ledger && chmod +x ~/bin/ledger
ledger --help
```

**3. 验证装好了**（逐字执行，我们就是这么测的）

```bash
mkdir -p /tmp/ledger-demo && cd /tmp/ledger-demo
ledger init demo --goal "verify the install" --expect 6 --trace
ledger book init demo --title "Install check" --goal "verify" \
  --ch "ch00|Brief|goal|goal" --ch "ch40|Conclusion|the answer|answer"
ledger book touch demo ch40 --answer "does it work"
ledger book check demo     # -> PASS
ledger book root           # -> PASS
```

<details>
<summary><b>其他 harness（Codex / Cursor / Gemini CLI / OpenCode / Kimi）</b></summary>

读的都是同一份 `SKILL.md` frontmatter，放进各自的 skills 目录即可：

```bash
# 项目内（入库、团队共享）
mkdir -p .claude/skills && cp -r Ledger/skills/ledger .claude/skills/ledger
```

CLI 是纯 Node 脚本，哪都能跑。**没有 marketplace 打包**——单体 skill 用 `cp -r` 就够。

</details>

---

## 用法

<details>
<summary><b>完整五步（点击展开）</b></summary>

```bash
# 1. 建任务目录（小任务会直接劝退）
ledger init search-cache --goal "决定缓存失效策略" --expect 8 --trace

# 2. 一次写全章节表，正文可以一个字没有
ledger book init search-cache --title "缓存失效" --goal "选一个策略" \
  --ch "ch00|Brief|约束与验收|goal" \
  --ch "ch10|Options|三个候选|candidates" \
  --ch "ch30|Evidence|压测与失败模式|bench" \
  --ch "ch40|Conclusion|唯一的决定|decision"

# 3. 逐章：先建文件，再写正文
ledger book touch search-cache ch30 --answer "压测到底显示了什么"

# 4. 每个产出登记两次（文件账 + 该章的产出表）
ledger add search-cache out/bench.json --does "p99 数据" --for "ch30"

# 5. 过闸
ledger book check search-cache
ledger book orphans search-cache
ledger book root
ledger artifact guard search-cache --max-mb 200
ledger close search-cache
```

**状态机**：`pending` --touch--> `writing` --结论写完--> `done`；废弃标 `dropped`，**章号永不复用**。

忘了加章？`ledger book add <slug> --ch "ch11|..."` —— 可以追加，不能改号。

</details>

<details>
<summary><b>产物判定细则</b></summary>

| 产物 | 处置 |
|:--|:--|
| 失败的 apk/ipa/dex/img/dump | 删，只留 `.md` 报告 |
| 成功且 ≤50MB | 留 |
| 成功但 >50MB | 删文件，报告里记路径+大小 |
| `dex/so/dll/class`（可从包重解） | 删 |
| dump / 反编译 / 日志 / 一次性脚本 | 删 |
| **任何 `.md` 报告 / 分析数据** | **永远留** |

`backup/` 下的回滚原件**永不进入删除候选**（已于本版本修复：三个闸门口径统一）。

</details>

---

## 已知局限

不遮掩。**一个隐瞒失败模式的工作流不值得信任。**

| # | 局限 | 状态 |
|:-:|:--|:--|
| 1 | 小任务开销可能超过产出（实测 26/83，5 个 100%） | ✅ **已修** — 门槛 |
| 2 | `MANIFEST.md` 固定模板税 | ✅ **已瘦身** ~1.2 KB；`sync` 可刷新旧账 |
| 3 | 书只在 11% 任务被使用 | ⚠️ **部分无解** — 靠判断，不强制 |
| 4 | 两本账会漂移 | ⚠️ **已检测**（告警，不阻断） |
| 5 | 索引腐烂：正文改了摘要没改 | ⚠️ **已检测**（告警，不阻断） |
| 6 | 章号永久，重构成本高 | 🔒 **接受** — 引用稳定 > 重构便利 |
| 7 | 三道闸门曾对 89% 未建书任务失效 | ✅ **已修** — `book orphans` |
| 8 | 产物分类是启发式，会静默判错 | 🔒 **接受** — 每条打印理由 + 默认 dry-run |
| 9 | **全靠提示词层，无 hook 运行时强制** | ❌ **无解** — Agent 可以完全无视 |

第 9 条是最严重的。**Agent 可以忽略整个工作流，没有任何东西阻止它。** 闸门买到的是「事后检测」，不是「合规保证」。

<details>
<summary><b>为什么不做 hook 层强制</b></summary>

因为要为每个 harness 的 hook API 做一次集成，而每次集成都是一份永久维护成本，收益只在该集成还活着的时候存在。与其做一个半吊子的拦截器，不如把检测做实。

</details>

---

<div align="center">

**如果这套东西帮你找回过一个结论，给个 ⭐ 就是最好的反馈。**

[⬆ 回到顶部](#ledger) · [English ↓](#english)

</div>

---
---

<div align="center">

# English

**🌐 Language / 语言**　[中文 ↑](#ledger) · **English**（current）

</div>

# Ledger

**Your agent already wrote the answer. Can you find it three weeks later?**

A **ledger-style workflow** for AI agents: three ledgers, five gates, and a "done" that a machine can verify.

[![Type](https://img.shields.io/badge/type-agent%20skill-4B32C3?style=flat-square)](./skills/ledger/SKILL.md)
[![Deps](https://img.shields.io/badge/dependencies-0-2EA043?style=flat-square)](./tools/ledger.mjs)
[![Node](https://img.shields.io/badge/node-%E2%89%A518-339933?style=flat-square&logo=node.js&logoColor=white)](./tools/ledger.mjs)
[![License](https://img.shields.io/badge/license-MIT-0366D6?style=flat-square)](./LICENSE)

---

## Table of Contents

| | | |
|---|---|---|
| [**The problem**](#the-problem) | [**Measured**](#measured-not-asserted) | [**What it does**](#three-ledgers) |
| [**Gates**](#five-gates) | [**When to use**](#when-to-use-when-not-to) | [**Why not a rules file**](#why-not-just-a-long-claudemd) |
| [**Comparison**](#compared-to-other-skill-repositories) | [**Install**](#install-in-60-seconds) | [**Limitations**](#known-limitations) |

---

## The problem

Long agent sessions fail in four predictable ways. None of them are fixed by telling the agent to be more careful.

<table>
<tr><th width="22%">Symptom</th><th width="48%">What it looks like</th><th width="30%">What it costs</th></tr>
<tr><td>🔍 <b>Lost conclusions</b></td><td>You know the agent figured it out. It is in the transcript, 400 turns back, somewhere</td><td>Re-deriving it, badly</td></tr>
<tr><td>📁 <b>Filename rot</b></td><td><code>report.md</code> → <code>report_v2.md</code> → <code>report_final.md</code> → <code>report_final_2.md</code></td><td>Nobody knows which one is true</td></tr>
<tr><td>💥 <b>Context blowup</b></td><td>To answer "what did we decide about caching", the agent greps 115,000 files</td><td>Time, tokens, a slower answer than reading a book</td></tr>
<tr><td>✅ <b>Unfalsifiable "done"</b></td><td>The agent says it is complete. Nothing checks that</td><td>Silent gaps in the deliverable</td></tr>
</table>

The usual fix is a longer prompt: *"keep your work organised."* That is a **suggestion**. Suggestions are followed inconsistently, and you cannot tell afterwards whether they were followed at all.

> **Ledger's fix: turn the discipline into structured files plus commands that exit non-zero when the schema is violated.**

---

## Measured, not asserted

Numbers from the reference implementation across **87 real task directories**.

<div align="center">

| Metric | Before | After |
|:--|:--|:--|
| Files examined to answer one question | ~**115,000** files / **46 GB** scanned | all `BOOK.md` indexes **≈14 KB**, then **one** chapter file |
| Retrieval cost | Full-project grep | **Three orders of magnitude smaller** |
| Tasks with an orphan-output gate | 9 of 83 (**11%**) | **100%** |
| Ledger overhead on small tasks | 2.1 KB of boilerplate, up to **100% of output** | 1.2 KB, and **not created at all** below the threshold |
| Empty-shell tasks caused by the workflow itself | 26 of 83 (**31%** were >20%) | **0** by construction |

</div>

> [!IMPORTANT]
> **The first version of this workflow was wrong, and we measured it.**
> Its rule was "every task, even a one-line change, must create a ledger."
> That rule manufactured **26 tasks whose only output was the ledger**, 5 of them at 100%.
>
> The threshold in [`SKILL.md` §6](./skills/ledger/SKILL.md) is the **repair**, not a nicety.

---

## Three ledgers

Each answers one machine-checkable question.

<div align="center">

| Ledger | File | Question it answers | Who checks it |
|:--|:--|:--|:--|
| 📒 **File ledger** | `MANIFEST.md` | Who made this, what for, who reads it, how to verify? | `close` — unregistered = exit 1 |
| 📖 **Chapter book** | `BOOK.md` | Which chapter answers my question? (≤200 lines, summary + keywords) | `book check` — orphans / duplicate numbers / index rot |
| 🗄️ **Artifact ledger** | `ledger artifact` | Keep it, register the path, or delete it? | `clean` dry-run by default + `guard` size budget |

</div>

<details>
<summary><b>Disk layout (click to expand)</b></summary>

```
_work/<slug>/              ← default root; override with --root or $LEDGER_ROOT
├── MANIFEST.md            file ledger
├── BOOK.md                chapter book — the only retrieval entry point
├── 00-brief/ch00.md       ← directories are derived from the chapter number
├── 10-plan/ch10.md
├── 20-exec/ch20.md
├── 30-evidence/ch30.md
├── 40-deliver/ch40.md     the single authoritative conclusion
├── out/                   real deliverables
├── backup/                originals for rollback — never deleted
└── tmp/                   probes; must be empty at close
```

`ch0x`→`00-brief`, `ch1x`→`10-plan`, `ch2x`→`20-exec`, `ch3x`→`30-evidence`, `ch4x`→`40-deliver`.

**Never number documents by round.** Rounds are not structure; stages are.

</details>

### Retrieval: locate once, read one chapter

```console
$ ledger book find cache
[ledger] 2 chapter(s) across 1 book(s)

## search-cache
  | ch30 | Evidence   | benchmarks and failure modes | bench    | 30-evidence/ch30.md | done |
  | ch40 | Conclusion | the one decision             | decision | 40-deliver/ch40.md  | done |

>> Read only the chapter bodies you hit. Do not scan directories.
```

**Two files. Not a repository scan.**

---

## Five gates

Not advice — **checks that exit 1**.

<div align="center">

| # | Command | Covers | On failure |
|:-:|:--|:--|:--|
| 1 | `book check <slug>` | Inside a task: duplicate numbers, status/file mismatch, orphans, `BOOK.md` >200 lines | **exit 1** |
| 2 | `book orphans [slug]` | Inside a task: unregistered files **without needing a book** | **exit 1** |
| 3 | `book root` | Between tasks: the root holds only task directories | **exit 1** |
| 4 | `artifact clean` | Keep / delete / register, **each decision prints its reason** | dry-run by default |
| 5 | `close <slug>` | Unregistered files + `tmp/` leftovers | **exit 1** |

</div>

**Artifact retention**: failed build output → delete (keep only the `.md` report) · success ≤50MB → keep · success >50MB → delete the file, register the path · **any `.md` report → always keep**.

**Image classification**: `keyframe/reference/contact-sheet` → deliverable, **keep** · `screen/shot/capture` or under `tmp/` → analysis screenshot, **delete** · bundled package resources → delete · unrecognised → **keep conservatively**.

> A measured counter-example: **76 AI keyframes (133 MB)** were almost deleted because the old rule classified by directory. Switching to filename-based classification saved all 76.
> Classification is heuristic, so **every decision prints its reason** and `clean` is dry-run by default.

---

## When to use — when not to

Most workflow tools get this wrong by insisting you always use them.

```
New task
  ├─ ≤3 outputs, one-shot            → no ledger. Just work.
  ├─ ≤3 outputs, needs a paper trail → init only (no book)
  ├─ >3 outputs or >3 steps          → init + book
  └─ must be retrievable later       → init + book, keywords filled in
```

<div align="center">

| ✅ Good fit (positive ROI) | ❌ Bad fit (negative ROI) |
|:--|:--|
| Multi-step work that may be interrupted | Changing one line of config (overhead up to **100%**) |
| A conclusion people will re-ask about | A one-shot lookup |
| Work handed to a person or a later session | Build-and-ship with no intermediates |
| A shared config change needing rollback | Exploratory thrash that will be rewritten |
| Outputs that multiply copies and big files | A mechanical find-and-replace |

</div>

**Staged exit** (the workflow is not for the whole lifecycle):

```
exploration (outcome unknown) → no book; write it once the answer settles
convergence (direction set)   → init + book, fill chapters
delivery                      → check + orphans + guard + close
```

**Signals to stop**: three consecutive `check` failures that are all "forgot to touch" → manifest-only · same chapter rewritten >2 times → the book was built too early · overhead above 30% of total output → don't init next time.

---

## Why not just a long CLAUDE.md

**Because prose cannot fail.**

<div align="center">

| | A rules file<br>`CLAUDE.md` / `AGENTS.md` | **Ledger** |
|:--|:--|:--|
| Enforcement | The model may comply | **5 commands exit non-zero** |
| Can you tell afterwards whether it was followed? | ❌ No | ✅ Unregistered file / orphan / missing chapter |
| Retrieval cost | Grows with the project | Near-constant: index layer → one chapter |
| Newcomer onboarding | Read the whole rules file | Read one `BOOK.md` |
| Survives context compaction | Often lost | ✅ On disk |

</div>

The trade is explicit: ~**1.2 KB** of ledger per non-trivial task, for a machine-verifiable **"done"**.

---

## Compared to other skill repositories

Most skill repositories are **collections** — they add capabilities.
Ledger is a **substrate** — it changes how the work is recorded, so it **composes** instead of competing.

<div align="center">

|  | [anthropics/skills](https://github.com/anthropics/skills)<br>`199k ★` | [obra/superpowers](https://github.com/obra/superpowers) | [alirezarezvani/claude-skills](https://github.com/alirezarezvani/claude-skills)<br>`434 skills` | **Ledger** |
|:--|:--|:--|:--|:--|
| **What it is** | Reference skill implementations | A software-dev methodology | A large capability collection | **A traceability workflow** |
| **Scope** | Per task, many domains | One project's dev lifecycle | Many domains, many roles | **Cross-domain, cross-session** |
| **Enforcement** | Guidance | Mandatory workflows (prompt-level) | Guidance + load tests | **5 commands exit 1** |
| **Persistence** | Session | Git branch + plan files | Session | **On-disk ledgers, per task** |
| **Retrieval** | — | Plan files per branch | Skill descriptions | **Index first, then one chapter** |
| **Admits its own failures** | n/a | Partly | Publishes load tests | **Yes, with the numbers — 26/83** |
| **Best paired with** | — | Project work | Domain expertise | **Any of the above** |

</div>

### And where Ledger is strictly worse

<details>
<summary><b>5 hard truths (click to expand)</b></summary>

| # | Where it is worse | Consequence |
|:-:|:--|:--|
| 1 | **It gives your agent no new ability** | If the problem is "the agent cannot do X", Ledger is the wrong tool |
| 2 | **Slower on small work** | A 1.2 KB ledger on a one-line change is a loss (hence the threshold) |
| 3 | **Imposes structure on exploration** | Building a book before you know the shape means rewriting chapters |
| 4 | **Chapter numbers are permanent** | Get the plan wrong and you must append, not reorder |
| 5 | **No visualisation** | Markdown and a CLI. No dashboard |

Five more implementation limits are in [Known limitations](#known-limitations).

</details>

> The one row no other repository has: **admits its own failures, with numbers.**
> That is not modesty — it proves two opposite things at once: this has been used for real, and measured for real.

---

## Install in 60 seconds

Zero dependencies · Node ≥18 · no build step.

**1. Install the skill** (Claude Code / any harness reading `~/.claude/skills`)

```bash
git clone https://github.com/yinxia2000y-hash/Ledger.git
cp -r Ledger/skills/ledger ~/.claude/skills/ledger
```

**2. Install the CLI**

```bash
cp Ledger/tools/ledger.mjs ~/bin/ledger && chmod +x ~/bin/ledger
ledger --help
```

**3. Verify the install** (run verbatim — this is how we tested it)

```bash
mkdir -p /tmp/ledger-demo && cd /tmp/ledger-demo
ledger init demo --goal "verify the install" --expect 6 --trace
ledger book init demo --title "Install check" --goal "verify" \
  --ch "ch00|Brief|goal|goal" --ch "ch40|Conclusion|the answer|answer"
ledger book touch demo ch40 --answer "does it work"
ledger book check demo     # -> PASS
ledger book root           # -> PASS
```

<details>
<summary><b>Other harnesses (Codex / Cursor / Gemini CLI / OpenCode / Kimi)</b></summary>

They all read the same `SKILL.md` frontmatter. Drop the folder into that harness's skills directory:

```bash
# project-local (checked into the repo, shared with the team)
mkdir -p .claude/skills && cp -r Ledger/skills/ledger .claude/skills/ledger
```

The CLI is a plain Node script that runs anywhere. **No marketplace packaging** — a single skill needs only `cp -r`.

</details>

---

## Use

<details>
<summary><b>The full five steps (click to expand)</b></summary>

```bash
# 1. create the task directory (small tasks get refused)
ledger init search-cache --goal "decide the cache invalidation strategy" --expect 8 --trace

# 2. write the whole chapter table up front. Bodies may be empty.
ledger book init search-cache --title "Cache invalidation" --goal "pick a strategy" \
  --ch "ch00|Brief|constraints and acceptance|goal" \
  --ch "ch10|Options|the three candidates|candidates" \
  --ch "ch30|Evidence|benchmarks and failure modes|bench" \
  --ch "ch40|Conclusion|the one decision|decision"

# 3. work chapter by chapter: create the file, then write the body
ledger book touch search-cache ch30 --answer "what did the benchmark actually show"

# 4. register every output twice (file ledger + the chapter's output table)
ledger add search-cache out/bench.json --does "p99 numbers" --for "ch30"

# 5. gates
ledger book check search-cache
ledger book orphans search-cache
ledger book root
ledger artifact guard search-cache --max-mb 200
ledger close search-cache
```

**State machine**: `pending` --touch--> `writing` --conclusion written--> `done`; abandoned chapters are marked `dropped` and **numbers are never reused**.

Forgot a chapter? `ledger book add <slug> --ch "ch11|..."` — append yes, renumber no.

</details>

<details>
<summary><b>Artifact retention rules</b></summary>

| Artifact | Ruling |
|:--|:--|
| Failed apk/ipa/dex/img/dump | delete; keep only the `.md` report |
| Successful, ≤50MB | keep |
| Successful, >50MB | delete the file, register path + size |
| `dex/so/dll/class` (regenerable) | delete |
| dump / decompile / log / one-off script | delete |
| **Any `.md` report or analysis data** | **always keep** |

Rollback originals under `backup/` **never enter the deletion candidate set** (fixed in this version — all three gates now agree).

</details>

---

## Known limitations

Nothing hidden. **A workflow that conceals its failure modes cannot be trusted.**

| # | Limitation | Status |
|:-:|:--|:--|
| 1 | Small-task overhead can exceed the output (26/83 measured, 5 at 100%) | ✅ **Fixed** — threshold |
| 2 | Fixed boilerplate in `MANIFEST.md` | ✅ **Slimmed** to ~1.2 KB; `sync` refreshes old ledgers |
| 3 | Books used in only 11% of tasks | ⚠️ **Partly unsolvable** — a judgement call, not a mandate |
| 4 | The two ledgers can drift | ⚠️ **Detected** (warn, non-blocking) |
| 5 | Index rot: body changed, summary not updated | ⚠️ **Detected** (warn, non-blocking) |
| 6 | Chapter numbers are permanent | 🔒 **Accepted** — stable references > refactor convenience |
| 7 | Three gates were dead for 89% of tasks | ✅ **Fixed** — `book orphans` |
| 8 | Artifact classification is heuristic and can be silently wrong | 🔒 **Accepted** — every decision prints its reason, dry-run by default |
| 9 | **Everything is prompt-level; no hook-level enforcement** | ❌ **Unresolved** — an agent can ignore all of it |

Number 9 is the serious one. **An agent can ignore this entire workflow and nothing stops it.** The gates buy *detection after the fact*, not compliance.

<details>
<summary><b>Why there is no hook layer</b></summary>

Because it requires one integration per harness's hook API, and every integration is a permanent maintenance cost whose benefit exists only while that integration works. A half-working interceptor is worse than solid detection.

</details>

---

<div align="center">

**If this ever helped you find an old conclusion, a ⭐ is the best feedback.**

[⬆ Back to top](#english) · [中文 ↑](#ledger)

</div>
