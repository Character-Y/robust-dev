# Robust-Dev Architecture Evolution — Discussion Record

Date: 2026-03-08

---

## 1. Problem Statement

robust-dev 方法论写好了（SKILL.md + reference.md），但 Claude 在实际开发中**始终不遵循**。两次实验均失败：

- **第一次**：CLAUDE.md 写 `Always apply the robust-dev skill for all development work.` → 太抽象，被忽略
- **第二次**：CLAUDE.md 改为命令式指令（"收到开发任务后、写第一行代码之前调用 Skill tool"）→ 仍然被忽略

Claude 自己的事后分析：当它处于"解决问题"心态中时，CLAUDE.md 的流程性要求容易被跳过。即使事实上做了增量测试，也没有走 robust-dev 的完整流程（decision log、资产归档等）。

---

## 2. Root Cause Analysis

### 2.1 根本原因：LLM 注意力机制的限制

Claude 没有"后台进程"。它只有一个注意力流，集中在当前最相关的 token 上。当 Claude 在写代码时，代码上下文、错误信息、功能需求是最相关的 token。方法论指令不管放在 CLAUDE.md 还是 skill 里，从注意力权重角度看都是"不相关的背景噪音"。

这不是指令质量的问题，是 transformer attention 的根本机制。**Prompt 可以改变 Claude 在某个时刻的行为，但无法让它在持续开发过程中"记住"要跑测试、写 log、归档资产。**

### 2.2 为什么各种方案都失败

| 方案 | 失败原因 |
|------|---------|
| 抽象指令（"always apply"）| 没有触发锚点，被上下文竞争淹没 |
| 命令式指令（"写代码前调用 Skill"）| Claude 进入开发模式后注意力转移，跳过流程步骤 |
| 频繁加载 skill | skill 内容比任务指令还多，Claude 注意力反而被方法论拉走，"为了 debug 而 debug" |
| 启发式深度思考 | "深度思考"可能是浅层的——输出一段"我会认真测试"然后照旧 |
| 在 plan mode 穿插 skill | 聚焦在启动时而非过程中，无法解决持续性行为问题 |

### 2.3 核心矛盾

robust-dev 的三条规则是**持续性行为要求**（每次改动都测试、每次 debug 都归档、行为变更都记录），但 Skill 系统是为**任务型调用**设计的（加载一次，用完就完）。用任务型机制承载持续性行为要求是机制与意图的错配。

---

## 3. Solution Evolution

### 3.1 Phase 1: Enriched Rules in CLAUDE.md

提出将三条规则写入 CLAUDE.md，不是一行话而是带足够细节的版本（~20行）。让 main agent 不需要加载 skill 就能正确执行。

**被否决的原因**：规则不是自足的。"Run the e2e test" 看起来清晰，但正确执行需要知道：加了新功能要先扩展测试、测试要验证正确性而非只跑通、资产存哪里用什么格式。脱离了方法论上下文的规则变成空壳，Claude 会"执行"但执行得很浅（机械跑测试、不知道怎么写 decision log、不知道资产结构）。

### 3.2 Phase 2: Persistent Subagent (Manual Agent Tool)

提出用 Agent tool 创建一个持久化的 robust-dev subagent，通过 resume 保持 session 内的连续性。Main agent 只需要 commit 前 resume 一下。

**优点**：
- 解决了注意力竞争（main agent 不需要理解方法论）
- 解决了 decision log 悖论（main agent 传意图，subagent 有方法论知识，两者汇合）
- 资产管理自动化（subagent 每次审查时自然维护）

**局限**：
- 需要 main agent 手动管理 agent ID
- Agent tool 创建的 subagent 跨 session 不持久

### 3.3 Phase 3: Official Subagent System (Current Proposal)

发现 Claude Code 官方的 subagent 系统（`.claude/agents/`）直接支持我们需要的所有机制：

| 需求 | 官方支持 |
|------|---------|
| 独立 context window | subagent 天然隔离 |
| 方法论知识注入 | `skills: [robust-dev]` 自动预加载 |
| 跨 session 持久记忆 | `memory: project` → `.claude/agent-memory/dev-auditor/MEMORY.md` |
| 记忆隔离 | main agent 压缩不影响 subagent |
| 自动委派 | Claude 根据 description 自动判断何时委派 |
| 权限控制 | `permissionMode` 字段 |
| 用户交互 | foreground 模式下 AskUserQuestion 透传给用户 |

---

## 4. Proposed Architecture (Consensus)

### 4.1 File Structure

```
robust-dev/                          ← GitHub repo (standalone)
├── SKILL.md                         ← 方法论内容（被 subagent 预加载）
├── reference.md                     ← 完整设计原理（002 文档）
├── agent.md                         ← subagent 定义文件（dev-auditor）
└── README.md                        ← 安装说明

安装到项目后：
project/
├── CLAUDE.md                        ← main agent 指令（简化的委派说明）
└── .claude/
    ├── agents/
    │   └── dev-auditor.md           ← 从 agent.md 安装
    ├── skills/
    │   └── robust-dev/
    │       ├── SKILL.md             ← 方法论（subagent 通过 skills 字段预加载）
    │       └── reference.md         ← 设计原理
    └── agent-memory/
        └── dev-auditor/             ← 自动创建，subagent 持久记忆
            └── MEMORY.md
```

### 4.2 Core Flow

```
Main agent 正常开发
         ↓
    准备 commit
         ↓
Main agent 委派给 dev-auditor（显式或自动）
  传入：git diff 摘要 + 变更意图 + 实现方案文档路径（如有）
         ↓
dev-auditor 第一轮（独立 context，预加载方法论，有持久记忆）:
  1. 读 agent memory（上次积累的项目知识）
  2. 读 conventions doc + index
  3. 读 diff，理解改了什么
  4. 判断测试覆盖 → 不够则设计新测试 / 扩展 e2e
  5. 跑所有相关测试
  6. 测试失败 → 完整 debug workflow
  7. 判断是否需要 decision log
     → 技术/设计问题信息不足 → 返回具体问题给 main agent（第一轮结束）
     → 验收标准/产品级问题不确定 → AskUserQuestion 问用户
  8. 如果有问题返回给 main agent → 进入第二轮
         ↓
Main agent 回答技术问题（此时空闲，无注意力竞争）
         ↓
dev-auditor 第二轮（resume，带着 main agent 的回答继续）:
  1. 基于补充信息写 decision log
  2. 更新 assets（baseline, index 等）
  3. 写 .robust-dev-approved 文件
  4. 返回审查报告给 main agent
         ↓
Main agent commit
```

注：大多数情况下第一轮就能完成（diff + 意图足够）。两轮委派是信息不足时的回退机制，不是每次都走。

### 4.3 Agent Definition (agent.md) Key Fields

```yaml
name: dev-auditor
description: "...Use proactively before every commit..."
skills: [robust-dev]           # 预加载方法论
memory: project                # 跨 session 持久记忆
tools: Bash, Read, Write, Edit, Grep, Glob
permissionMode: default        # 测试阶段，用户可见每个操作
model: sonnet                  # 保证 debug 质量，避免 haiku 能力不足
```

### 4.4 SKILL.md Changes

```yaml
disable-model-invocation: true  # main agent 不自动加载（避免注意力污染）
user-invocable: true            # 用户仍可 /robust-dev 手动查看
```

正文内容不变——它是 subagent 的方法论知识，通过 `skills` 字段注入。

### 4.5 CLAUDE.md (Project Level)

简化为 3 段：
- commit 前委派给 dev-auditor（提供 diff + 意图 + 方案文档）
- 首次使用的注意事项（初始化、测试命令传递）
- session 内尽量 resume 同一个实例
- 如果 dev-auditor 返回技术问题，回答后再次委派（两轮委派）

### 4.6 Pre-commit Hook (Optional Safety Net)

检查 `.robust-dev-approved` 文件是否存在。如果 main agent 忘记委派，hook 拦住 commit。

---

## 5. Key Design Decisions

### 5.1 为什么用 subagent 而不是 skill

| 维度 | Skill (old) | Subagent (new) |
|------|-------------|----------------|
| Context | 注入 main agent 的 context，与开发任务竞争注意力 | 独立 context window，完全隔离 |
| 记忆 | 无持久化，每个 session 从零开始 | `memory: project` 跨 session 持久 |
| 执行 | 依赖 main agent "记住"遵循规则 | 独立 agent 专注执行方法论 |
| Main agent 负担 | 理解并执行整套方法论 | 一句话："commit 前委派给 dev-auditor" |

### 5.2 为什么 permissionMode 选 default

测试阶段，用户需要看到 subagent 的每个操作（跑了什么命令、改了什么文件）来验证它是否正确理解和执行方法论。生产阶段可以考虑 `acceptEdits` 或 `bypassPermissions`。

### 5.3 为什么 model 选 sonnet

- haiku 可能 debug 能力不足，用户会把 debug 失败归因于工具问题
- opus 太贵太慢，不适合每次 commit 都调用
- sonnet 是质量与成本的平衡点
- 未来可以在 agent.md 中按需调整

### 5.4 Decision log 的写作流程

Main agent 不理解方法论 → 不适合写 decision log。
Subagent 不理解变更意图 → 不适合独立写 decision log。

**解决方案**：信息汇合 + 分层提问。

1. Main agent 委派时提供：diff + 意图 + 方案文档路径
2. dev-auditor 读 diff + 意图，用方法论知识判断是否需要 log
3. 如果信息不够，按问题类型分流：
   - **技术/设计问题**（为什么选方案 A、是否临时方案、哪些模块受影响的设计考量）→ 返回给 main agent，由 main agent 回答后再次委派（两轮委派）
   - **产品/验收问题**（行为变更是否符合预期、验收标准、业务优先级）→ AskUserQuestion 问用户（foreground 透传）
4. dev-auditor 基于所有信息写出有深度的 decision log

**为什么分层**：main agent 掌握实现细节和设计取舍，用户掌握产品意图和验收标准。问错人会得到低质量的回答。dev-auditor 最终的审查结果由用户审查（approve/reject），所以验收相关的不确定性问用户是合理的。

**为什么两轮委派可行**：main agent 委派后处于空闲等待状态，回答具体问题没有注意力竞争。这和"持续遵循流程"是完全不同性质的认知负担。即使极端情况（compaction）导致中断，pre-commit hook 提供系统级兜底。

### 5.5 命名：skill 与 agent 分离

Skill（方法论）保持原名 `robust-dev`。Agent 改名为 `dev-auditor`。

**原因**：agent 不负责开发，它的职责是审查、测试、归档、决策记录——是审计角色。参考官方 subagent 命名模式（`code-reviewer`、`debugger`、`data-scientist`、`db-reader`），命名应反映 agent 做什么或是谁，而不是它使用的方法论。

`dev-auditor` = 开发审计员，简洁准确。

### 5.6 深度思考优先于机械执行

dev-auditor 拥有独立的 context window、充足的注意力、sonnet 级别的推理能力。不应当把它当作一个机械执行 checklist 的工具——它应该**用方法论作为框架，自主思考和判断**。

具体体现：
- **测试设计**：不是"跑已有的测试"，而是先思考这个变更可能在哪里出问题（跨模块副作用、隐式依赖、配置变化），然后设计**针对那些风险**的测试
- **Decision log**：不是"填模板"，而是分析变更的深层影响——未来哪个开发者（或 AI），如果不知道这个变更，会做出错误假设？
- **资产归档**：不是"保存所有东西"，而是判断哪些信息对未来 debug 最有价值，写出有洞察力的 debug summary
- **Memory 积累**：不只是记录事实（"module X 容易出 bug"），而是记录**理解**（"module X 出 bug 是因为它和 Y 通过共享配置有隐式耦合"）

**Self-evolving**：每次调用都让 dev-auditor 对项目的理解更深。通过 persistent memory 积累的不只是数据，是**项目认知**——哪些区域脆弱以及为什么脆弱、什么类型的变更容易出问题、什么测试策略对这个项目最有效。这让 dev-auditor 从一个通用审计员逐渐变成这个项目的专家。

### 5.7 跨 session 的连续性

**Session 内**：通过 resume 同一个 agent 实例保持 context。Auto-compaction 不改变 agent ID，压缩后仍可 resume。

**跨 session**：新的 agent 实例，但 `memory: project` 提供持久记忆。Agent 的 MEMORY.md（前 200 行）自动注入 system prompt。文件系统的资产（conventions doc, index, tests, logs）提供完整的长期记忆。

**类比**：session 内的 resume = 同一个人继续工作。跨 session = 交接班，新人读交接文档。

---

## 6. Addressed Concerns

### 6.1 Subagent 看不到项目 MEMORY.md

Subagent 不继承 main agent 的 MEMORY.md（含 conda 路径、测试命令等）。

**解决方式**：
- 首次委派时，main agent 告知测试命令和环境信息
- Subagent 记到自己的 agent memory 中
- 后续不需要再传
- 对于新项目，subagent 在初始化时自行发现（或问用户）

### 6.2 Subagent 不能创建 subagent

官方限制："Subagents cannot spawn other subagents."

对我们没影响——robust-dev agent 自己跑测试、写 log、管资产，不需要再委派。它有 Read/Grep/Glob 工具可以搜索代码库。

### 6.3 Main agent 用多个 subagents 并行开发

Robust-dev agent 不需要实时监控。Commit 是天然同步点——所有并行 subagent 的变更最终汇聚到 git diff 中。Robust-dev agent 在 commit 前看到的是完整的变更合集。

### 6.4 用户如何监控 subagent

- Foreground 模式：每个操作（Bash/Write/Edit）都会弹权限确认
- Agent memory：用户可以直接读 `.claude/agent-memory/dev-auditor/MEMORY.md`
- 资产文件：decision logs, debug summaries, index 都是可读文件
- Transcript：存储在 `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl`

### 6.5 Claude 可能用错工具启动新 agent 而非 resume

确实有风险。缓解措施：
- CLAUDE.md 明确写 "session 内尽量 resume 同一个 dev-auditor 实例"
- 即使创建了新实例，`memory: project` 保证持久知识不丢
- Pre-commit hook 作为兜底

### 6.6 资产管理是否会随项目增长越来越混乱

Subagent 架构下，资产管理是 subagent 的日常工作（每次审查时自然维护），不是额外的维护任务。Subagent 的持久记忆会记住哪些区域高风险、哪些测试过时。Conventions doc 定义了何时拆分 index。

比让 main agent 或用户手动维护可靠得多——subagent 的注意力完全集中在这件事上。

---

## 7. Open Questions / TODO

- [ ] 实际测试 subagent 的 `skills` 预加载是否正确工作（skill 全文是否真的注入 system prompt）
- [ ] 测试 `memory: project` 的跨 session 持久化效果
- [ ] 测试 foreground 模式下 AskUserQuestion 是否正确透传
- [ ] 确认 auto-compaction 不影响 agent ID（需要实际验证）
- [ ] 评估是否需要 pre-commit hook，还是仅靠 Claude 的自动委派就够了
- [ ] 确认 subagent 是否可以访问 MCP 工具（preview 等浏览器测试工具）
- [ ] 发布为 GitHub repo 时需要 install script 简化安装流程
- [ ] 考虑是否需要一个 uninstall/update 机制

---

## 8. Next Steps

1. 创建 `agent.md`（subagent 定义 + system prompt）
2. 修改 `SKILL.md` frontmatter（`disable-model-invocation: true`）
3. 重写 AITA 项目的 `CLAUDE.md`（简化为委派指令）
4. 安装到 AITA 项目（`.claude/agents/` + `.claude/skills/`）
5. 在 AITA 项目中实际测试一轮开发 → 验证 subagent 是否正确触发、执行、归档
6. 根据测试结果迭代调整
7. 推送到 GitHub
