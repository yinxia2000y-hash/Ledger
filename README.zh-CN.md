# Ledger 工作流

**你的 agent 早就写出过答案。三周后你还找得到吗？**

Ledger 是一套给 AI coding agent 用的工作流 skill：它让 agent 维护三本账 —— 文件账、章节账、产物账 —— 使每一份产出可追溯、每个结论**读一次就能翻到**、而"做完了"变成**机器可判定的结论**，不再是一句自称。

<p>
<img alt="skill" src="https://img.shields.io/badge/type-agent%20skill-blue">
<img alt="deps" src="https://img.shields.io/badge/dependencies-zero-brightgreen">
<img alt="node" src="https://img.shields.io/badge/node-%3E%3D18-informational">
<img alt="license" src="https://img.shields.io/badge/license-MIT-lightgrey">
</p>

> 英文为权威版本（[README.md](README.md)）。本文件服务中文读者，内容一致，不是逐句翻译。

---

## 它解决什么问题

长会话的失败方式只有四种，而且都很难靠"记得更认真"避免。

| 症状 | 具体表现 | 代价 |
|---|---|---|
| **结论丢失** | 你知道 agent 想通了。答案在 400 轮之前的对话里，某个地方。 | 重新推一遍，而且推得更差 |
| **文件名腐烂** | `report.md`、`report_v2.md`、`report_final.md`、`report_final_2.md` —— 没有一份权威 | 没人知道哪份是真的 |
| **上下文爆炸** | 为了回答"我们当时怎么定缓存的"，agent 去 grep 11.5 万个文件 | 时间、token，以及一个比读书还慢的答案 |
| **"完成"不可证伪** | agent 说做完了。没有任何东西检查这件事。 | 交付物里有静默的空洞 |

常见做法是加长提示词：*"把工作整理好。"* 那是建议。建议的执行是不均匀的，而且事后你也看不出到底有没有执行。

**Ledger 的做法是把纪律变成"有结构的文件 + 违规就非零退出的命令"。**

---

## 实测，不是断言

数据来自参考实现对 87 个真实任务目录的统计。

| 指标 | 改之前 | 改之后 |
|---|---|---|
| 回答一个问题要看的文件 | 约 115,000 个文件 / 扫 46 GB | 所有 `BOOK.md` 索引，**合计约 14 KB**，然后**读一个章节文件** |
| 检索成本 | 全项目 grep | **小三个数量级** |
| 有"孤儿产出"闸门的任务占比 | 83 个里 9 个（**11%**） | **100%**（`book orphans` 不需要建书） |
| 小任务上的台账开销 | 2.1 KB 固定模板，最高占到**总产出的 100%** | 1.2 KB，并且在门槛以下**根本不建** |
| 工作流自己制造的空壳任务 | 83 个里 26 个（**31%** 的台账占比 >20%） | 结构上为 0 |

最后两行是关键：**这套工作流的第一个版本是错的，而且是我们自己量出来的。** 当时的规则是"任何任务，哪怕只改一行，都必须先建台账"。这条规则制造了 26 个"除了台账什么都没产出"的任务。`skills/ledger/SKILL.md` 第 6 节的门槛是**修复**，不是装饰。

---

## 它具体做什么

三本账，各对应一个可机器判定的问题。

| 账 | 文件 | 回答什么 | 闸门 |
|---|---|---|---|
| **文件账** | `MANIFEST.md` | 这文件谁建的、干什么、谁读它、怎么验证？ | `close` 报未登记文件，exit 1 |
| **章节账** | `BOOK.md` | 我要的答案在哪一章？（≤200 行，每章一句话摘要 + 关键词） | `book check` —— 孤儿、章号重复、索引腐烂、两账漂移 |
| **产物账** | `ledger.mjs artifact` | 留着、只记路径、还是删掉？ | `clean` 默认 dry-run；`guard` 卡体积预算 |

```
_work/<slug>/             （默认根；用 --root 或 $LEDGER_ROOT 改）
├── MANIFEST.md           文件账
├── BOOK.md               章节账 —— 唯一检索入口
├── 00-brief/ch00.md
├── 10-plan/ch10.md
├── 20-exec/ch20.md
├── 30-evidence/ch30.md
├── 40-deliver/ch40.md    唯一的权威结论
├── out/                  真产物
├── backup/               你改过的文件的原件（回滚用）
└── tmp/                  探针；收尾必须清空
```

章节目录由章号推导（`ch0x`→`00-brief`，`ch1x`→`10-plan`……）。**永远不要用"第 17 轮"给文档编号。** 轮次不是结构，阶段才是。

---

## 安装

skill 就是一个带 `SKILL.md` 的文件夹。没有编译，没有依赖。

**Claude Code / 任何读 `~/.claude/skills` 的 harness**

```bash
git clone https://github.com/<you>/ledger.git
cp -r ledger/skills/ledger ~/.claude/skills/ledger
```

**项目内（入库、团队共享）**

```bash
mkdir -p .claude/skills
cp -r ledger/skills/ledger .claude/skills/ledger
```

**Codex / Cursor / Gemini CLI / OpenCode / Kimi** —— 读的都是同一份 `SKILL.md` frontmatter，放进各自的 skills 目录即可；CLI 是纯 Node 脚本，哪都能跑。

```bash
cp ledger/tools/ledger.mjs ~/bin/ledger
chmod +x ~/bin/ledger
ledger --help
```

验证装好了：

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

## 用法

```bash
# 1. 建任务目录（小任务会直接劝退，见下）
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
ledger add search-cache out/bench.json --does "p99 数据" --for "ch30" --verify "cat out/bench.json"

# 5. 过闸
ledger book check search-cache
ledger book orphans search-cache
ledger book root
ledger artifact guard search-cache --max-mb 200
ledger close search-cache
```

日常最常用的检索：

```bash
ledger book find cache
```

```
[ledger] 2 chapter(s) across 2 book(s)

## search-cache
  | ch30 | Evidence | benchmarks and failure modes | bench | `30-evidence/ch30.md` | done |
  | ch40 | Conclusion | the one decision | decision | `40-deliver/ch40.md` | done |

>> Read only the chapter bodies you hit. Do not scan directories.
```

两个文件。不是一次全库扫描。

---

## 什么时候用，什么时候别用

这是多数工作流工具会搞错的地方 —— 它们坚持你永远都该用它。

```
新任务
  ├─ 产出 ≤3 且一次性            → 不建账，直接干
  ├─ 产出 ≤3 但要留痕            → 只 init（不建书）
  ├─ 产出 >3 或步骤 >3           → init + 建书
  └─ 跨会话需要被检索            → init + 建书，关键词写全
```

| 该用 | 不该用 |
|---|---|
| 可能中断、需要续上的多步工作 | 改一行配置 |
| 别人会反复追问的结论 | 一次性查询 |
| 要交给另一个人 / 另一个会话 | 纯构建交付、没有中间态 |
| 改共享配置、需要回滚路径 | 结果大概率被推翻的探索 |
| 会产生多副本和大文件的产出 | 明确的机械替换 |

**分阶段退出**（工作流不是全生命周期都该在）：

```
探索期（结果不确定）→ 不建书，允许 tmp/ 乱放，结论稳了再一次性写
收敛期（方向已定）  → init + 建书，逐章补
交付期              → check + orphans + guard + close
```

**该停用的信号**：连续三次 `check` 失败都是"忘了 touch"（降级为只留文件账）；同一章被重写超过两次（书建早了）；台账开销持续超过任务总产出的 30%（任务太小，下次别 init）。

---

## 为什么不直接用一份很长的 `CLAUDE.md` / `AGENTS.md`？

因为文字不会失败。

| | 规则文件 | Ledger |
|---|---|---|
| 强制方式 | 模型愿意遵守 | `check`、`orphans`、`root`、`guard`、`close` 非零退出 |
| 事后能看出有没有遵守吗 | 不能 | 能 —— 未登记文件、孤儿产出、缺失章节 |
| 检索成本 | 随项目增长 | 近似常数：先索引层，再一章 |
| 新人上手 | 读完整个规则文件 | 读一个 `BOOK.md` |
| 上下文压缩后还在吗 | 经常丢 | 在磁盘上 |

代价是明确的：每个非微小任务付约 1.2 KB 台账，换来一个可机器验证的"完成"。

---

## 与其他 skill 仓库的对比

多数 skill 仓库是**能力集合**——增加 agent 能做的事。Ledger 是**底层规范**——改变其他工作的记录方式，所以它是叠加而不是竞争。

| | [`anthropics/skills`](https://github.com/anthropics/skills) | [`obra/superpowers`](https://github.com/obra/superpowers) | [`alirezarezvani/claude-skills`](https://github.com/alirezarezvani/claude-skills) | **Ledger** |
|---|---|---|---|---|
| 是什么 | 官方 skill 参考实现 | 一套软件开发方法论 | 434 个 skill / 21 个域 | 一套可追溯工作流 |
| 覆盖范围 | 按任务，多域 | 单个项目的开发周期（spec → plan → TDD → review） | 多域、多角色 | 跨域、跨会话 |
| 强制力 | 引导 | 强制工作流，但仍属提示词层 | 引导 + 加载测试 | **5 条命令非零退出** |
| 持久化 | 会话内 | Git 分支 + 计划文件 | 会话内 | **磁盘上的账，按任务** |
| 检索方案 | — | 每分支计划文件 | skill 描述 | **先索引，再一章** |
| 是否公开自己的失败 | 不适用 | 部分（有诊断 skill） | 公开加载测试结果 | **公开，且带数字 —— 26/83 空壳** |
| 最佳搭配 | 所有 | 项目型工作 | 领域专长 | 上面任意一个 |

> Ledger 不给你的 agent 新能力。它让 agent 已有的能力**可被问责**。

---

## 仓库结构

```
skills/ledger/SKILL.md     skill 本体（这个才是你安装的东西）
tools/ledger.mjs           CLI：init/add/close/sync、book *、artifact *（零依赖）
docs/COMPARISON.md         与其他工作流方案的较详细对比
docs/DESIGN-NOTES.md       门槛为什么存在，以及背后的测量
README.md / README.zh-CN.md
LICENSE                    MIT
```

---

## 已知局限

写在最前面，因为**隐瞒失败模式的工作流不值得信任**。

1. **没有运行时强制。** 一切都是提示词层纪律 + agent 必须主动去跑的命令。agent 可以完全无视，没人拦它。这里没有 hook 层。
2. **小任务代价过高。** 门槛以下，台账是纯开销。门槛是修复手段，但不是彻底修复。
3. **索引腐烂只能被检测，不能预防。** 章文件比 `BOOK.md` 新时 `check` 会告警，但更新摘要仍靠人。
4. **章号永久。** 不支持重排；可以用 `book add` 追加，永远不能改号。这是用重构便利换引用稳定的自觉取舍。
5. **产物分类是启发式。** `artifact clean` 按扩展名和文件名判留删，可能静默判错。所以它默认 dry-run，且每条决定都打印理由。（`backup/` 已豁免 —— 回滚原件永不进入删除候选。）
6. **两本账会漂移。** `MANIFEST.md` 与 `BOOK.md` 由不同命令写；`check` 告警但不阻断。

---

## 许可

MIT，见 [LICENSE](LICENSE)。
