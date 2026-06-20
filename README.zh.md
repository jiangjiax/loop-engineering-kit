# loop-engineering-kit

> 一套用来在 Claude Code 里搭建**多阶段 agent loop（智能体循环）** 的脚手架。
> 自带 verifier 隔离、品味投票、零打断执行三件套。

[English README](./README.md) · 中文版（本文）

把这套脚手架丢进 `~/.claude/skills/` 或者你项目的 `.claude/skills/` 目录，填上你自己的 `[MAKER]` 和 `[CHECKER]`，你就得到一条流水线，它可以：

- **生产 → 验证 → 迭代 → 归档**，全程不来打扰你
- **把 verifier 隔离在 fresh subagent 里**，这样生产者没法自己给自己作业打分
- **让 checker 的发现先走 gate file（你自己的规则文件）**，再决定要不要升级到人
- **状态跨 phase 持久化**，写在三个只追加（append-only）的文件里（`STATE.md` / `loop-budget.md` / `loop-run-log.md`）

这不是一个 framework，是**一套可以复制改造的模式和模板**。你照着自己的领域改。

这套模式**和具体领域无关**。同样的 9 phase 结构可以套到：

- **文本生产** — 写作、编辑、文案、翻译、摘要
- **判断类工作** — 代码审查、设计评审、研究评估、决策辅助
- **信息综合** — 研究综述、市场分析、多源报告
- **结构化产物** — 计划书、提案、schema、配置文件

具体可跑的例子，在 [`examples/`](./examples/) 目录里。

---

## 为啥 agent loop 需要一个工头

绝大多数多阶段 LLM 工作流死在这三个地方：

1. **生产者自己给自己作业打分** → 质量悄悄掉，等用户反馈过来才发现已经晚了
2. **orchestrator 隔三差五问一句"行不行？"** → 人成了瓶颈，loop 失去意义
3. **状态只活在对话上下文里** → 一旦上下文重置或 loop 崩了，根本不能续跑

这套脚手架里的**工头模式（foreman pattern）** 把这三件事一起解决：

- **verifier 隔离**（每个 checker 都是 fresh subagent，带 hardcoded 的必读清单）
- **先自己决，最后才升级**（5 类阻塞问题允许问人，其他必须自己解决）
- **三文件状态脊柱**（STATE / budget / run-log 跨 phase 跨 run 都活着）

这些都不是我发明的，是 agent 工程界已经反复验证过的模式（详见 [INSPIRATION.md](./INSPIRATION.md)）。这套脚手架做的事，是把它们打包成一个 Claude Code skill，可以直接用。

---

## 里面有啥

```
loop-engineering-kit/
├── README.md / README.zh.md   ← 你正在看的（中英文双版本）
├── PHILOSOPHY.md              ← 5 条铁律 + 设计原理
├── INSPIRATION.md             ← 灵感来源 & 信用名单
│
├── skills/
│   └── loop-foreman/          ← 脚手架本体（领域无关）
│       ├── SKILL.md
│       └── references/
│           ├── phase-templates.md      ← 9 个 phase 模板，带 [MAKER] / [CHECKER] 占位符
│           ├── iron-laws.md            ← 5 条铁律完整原文
│           ├── taste-subagent.md       ← verifier 隔离模式
│           ├── bash-no-go-zone.md      ← 哪些 Bash 命令会触发权限弹窗
│           └── critic-design-pattern.md ← 怎么设计你自己的 critic-rules.yaml
│
├── examples/                  ← 跨领域的可跑示例
│   ├── README.md              ← 挑跟你领域最像的 example 开始
│   ├── email-loop/            ← 最简单：文本生产，跑 9 phase 减 3 + 3.5
│   └── code-review-loop/      ← 生产级：并行多视角 + 交叉审 + polish
│
├── templates/                 ← 直接 cp 到你的项目根目录就能用
│   ├── STATE.md.template
│   ├── loop-budget.md.template
│   └── loop-run-log.md.template
│
└── docs/
    └── how-to-build-your-own-loop.md
```

---

## 5 分钟上手

```bash
# 1. clone 到 Claude Code 的 skills 目录
cd ~/.claude/skills/
git clone https://github.com/jiangjiax/loop-engineering-kit.git

# 2. 把三个状态文件 cp 到你项目根目录
cd /path/to/your/project
cp ~/.claude/skills/loop-engineering-kit/templates/*.template ./
mv STATE.md.template STATE.md
mv loop-budget.md.template loop-budget.md
mv loop-run-log.md.template loop-run-log.md

# 3. 挑一个跟你领域最像的 example
cat ~/.claude/skills/loop-engineering-kit/examples/README.md

# 4. 搭你自己的
# - skills/loop-foreman/ → 复制成 skills/my-loop-foreman/
# - 把 [MAKER_1] / [MAKER_2] / [CHECKER] 占位符换成你自己的 skill
# - 写你自己的 critic-rules.yaml（看 docs/how-to-build-your-own-loop.md）
```

然后在 Claude Code 里这么调：

```
> 跑 my-loop 处理 [你的输入]
```

工头接管，依次跑每个 phase，写状态，最后给你一份报告。

---

## 你要自己准备啥

这是**脚手架**，不是开箱即用的产品。下面这些得你自己写：

1. **你的 producer** — 真正生产产物的 skill（写、做、设计、分析、判断什么都行）。跑在主线程里。
2. **你的 checker** — 给输出打分的 skill。跑在 fresh subagent 里。
3. **你的 critic rules** — 一个 YAML 文件，告诉 checker 什么算 fatal/serious/general 违规。没这个，你的 loop 就没有验收门。
4. **你的领域上下文** — producer 和 checker 需要读的任何参考文件。

脚手架管**接线**（什么时候 spawn subagent，怎么投票，状态写哪，什么时候升级）。你管**内容**（你的 loop 到底要干啥）。

---

## 什么时候**不要用**这套

这套脚手架默认你：

- 有**多个串行阶段**（≥ 3 个），每个阶段都生成草稿、下一个阶段往前推进
- 需要**验证**（有些产出验证比生产更难）
- 想要**不被打扰地迭代**（不是聊天机器人——是个后台生产工人）

如果你的任务一次就完（"总结这个 PDF"），不需要 loop，直接调 skill 就行。

如果你的阶段之间**没依赖**，用并发 agent，别用串行 loop。

如果验证很简单（一个正则就搞定），不需要 fresh subagent，inline 验就行。

---

## License

MIT。随便用，搭什么都行。搭出来什么好玩的，欢迎开 issue 给我看。

---

## 致谢

工头模式是过去一年我搭生产级 agent loop 的产物。这些模式**和具体领域无关**——文本生产、判断类工作、信息综合、任何"多阶段 + 验证 + 迭代"的工作都能套用。如果不是 [INSPIRATION.md](./INSPIRATION.md) 里列的那些人做的工作，这里啥都打包不出来。
