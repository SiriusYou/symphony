# AI 编码工作流统一流水线方案

> Agentic Engineer (规划) → OpenClaw/Zoe (编排) → Symphony/Claude/Codex (执行)

---

## 1. 概述

### 1.1 问题

当前 AI 辅助编码领域有三类工具各自为战：

- **规划工具**（如 Agentic Engineer）擅长把模糊需求变成严谨的设计规范，但不管执行
- **编排工具**（如 OpenClaw/Zoe）擅长任务分发和人工审批，但缺乏深度 Agent 执行能力
- **执行引擎**（如 Symphony）擅长自主多轮编码和 PR 管理，但缺乏上游规划和灵活路由

### 1.2 方案

将三者串联为**三阶段流水线**，每个系统只做自己最擅长的事：

```
阶段 1: 想清楚          阶段 2: 分好活           阶段 3: 做出来
┌──────────────┐     ┌──────────────┐     ┌──────────────────┐
│   Agentic    │     │   OpenClaw   │     │ Claude / Codex   │
│   Engineer   │────→│   (Zoe)      │────→│ 或 Symphony      │
│              │     │              │     │                  │
│ 模糊需求      │     │ 路由 + 审批   │     │ 编码 + PR + 合并  │
│ → 设计规范    │     │              │     │                  │
└──────────────┘     └──────────────┘     └──────────────────┘
  spec_final.md        Linear 工单           代码交付
```

### 1.3 核心原则

1. **Agentic Engineer 是前置步骤**，不是可路由的"通道"——所有非平凡任务都应先经过规划
2. **OpenClaw (Zoe) 是唯一的编排层**——任务入口、路由决策、人工审批全在这里
3. **Symphony 是 OpenClaw 的执行后端之一**——与 Claude CLI、Codex CLI 并列，通过 adapter 模式接入
4. **Linear 是共享状态层**——OpenClaw 和 Symphony 通过 Linear 工单状态通信，不需要自建同步机制

---

## 2. 系统定位与职责

### 2.1 Agentic Engineer：规划引擎

| 项目 | 说明 |
|------|------|
| 角色 | 流水线的第一阶段，所有复杂任务的前置步骤 |
| 输入 | 模糊的产品需求、功能想法、问题描述 |
| 输出 | `spec_final.md`（经过对抗性压力测试的设计规范） |
| 技术栈 | Python + Claude Code Skill |
| 运行模式 | 按需触发，人工驱动（未来可自动化） |

**核心流程：**

```
灵感捕获 → SDD 生成 → 对抗性压力测试 → 迭代修正 → spec_final.md
                              ↑                │
                              └── 未收敛 ───────┘
```

**关键工具：**

- `scorecard_parser.py` — 解析压力测试评分卡，判断是否收敛
- `spec_lint.py` — 验证设计文档结构完整性
- `check_workflow_consistency.py` — 跨文档一致性校验

**产出标准：**

规划完成的标志是 scorecard 收敛——所有关键问题已解决或显式标记为范围外。

### 2.2 OpenClaw (Zoe)：编排层

| 项目 | 说明 |
|------|------|
| 角色 | 统一控制中心，负责任务路由和人工审批 |
| 输入 | Linear 工单（由 spec 拆解而来，或直接创建的简单任务） |
| 输出 | 路由决策 + 审批结果 |
| 技术栈 | TypeScript / Next.js / SQLite / Drizzle ORM |
| 运行模式 | 常驻 Web 服务 + Worker 进程 |

**Agent Adapter 体系：**

| Adapter | 适用场景 | 执行方式 |
|---------|---------|---------|
| `claude` | 简单任务、文档生成、代码分析 | Claude Code CLI 直接执行 |
| `codex` | 简单编码任务 | Codex CLI 直接执行 |
| `symphony` | 复杂多步任务、需要多轮对话和自动 PR 管理 | 委托给 Symphony 执行 |

**核心价值：**

- 统一任务入口
- 灵活路由（按任务复杂度、风险级别选择 adapter）
- 人工审批把关（所有任务完成后必须人工确认）
- 拒绝后可触发返工

### 2.3 Symphony：深度执行引擎

| 项目 | 说明 |
|------|------|
| 角色 | OpenClaw 的高级执行后端，处理复杂任务 |
| 输入 | Linear 工单（由 OpenClaw 创建并打 `route:symphony` 标签） |
| 输出 | 代码提交、PR、Linear 状态更新 |
| 技术栈 | Elixir / OTP / Phoenix LiveView |
| 运行模式 | 独立守护进程，轮询 Linear |

**核心能力（OpenClaw 直接调用 Claude/Codex 不具备的）：**

1. **多轮 Codex 对话**（max_turns: 20）—— 一个任务可以跨多轮持续执行
2. **WORKFLOW.md prompt 工程** —— Liquid 模板 + 完整状态机定义
3. **自动 PR 管理** —— 创建 PR、扫描反馈、合并
4. **Linear 深度集成** —— Workpad 评论、状态流转、阻塞检测
5. **工作空间隔离** —— 每个 issue 独立目录
6. **失败自动恢复** —— 指数退避重试、状态协调

---

## 3. 端到端工作流

### 3.1 全流程图

```
工程师
  │
  │  ① 提出需求（模糊想法）
  ▼
Agentic Engineer
  │  - 灵感捕获 → SDD 生成
  │  - 对抗性压力测试
  │  - 迭代直到收敛
  │
  │  ② 产出 spec_final.md
  ▼
工单拆解（人工或自动）
  │  - 将 spec 按模块/功能拆为独立工单
  │  - 每个工单写入 Linear
  │  - 标注复杂度和风险
  │
  │  ③ 工单进入 Linear
  ▼
OpenClaw (Zoe)
  │  - 工程师在 Web UI 查看工单
  │  - 为每个工单选择 adapter
  │
  ├── 简单任务 (adapter=claude/codex)
  │     │  - Worker 认领
  │     │  - 在 git worktree 中执行
  │     │  - 产出代码变更
  │     ▼
  │   awaiting_review
  │     │  - 工程师审批
  │     ▼
  │   merged 或 rejected
  │
  └── 复杂任务 (adapter=symphony)
        │  - Worker 认领
        │  - 在 Linear 创建工单 + 打标签
        │  - 触发 Symphony 拾取
        ▼
      Symphony
        │  - 创建隔离工作空间
        │  - 多轮 Codex 执行
        │  - 自动创建 PR
        │  - 推进到 Human Review
        ▼
      OpenClaw 收到完成通知
        │  - 工程师审批
        │
        ├── 批准 → Linear 状态改为 Merging
        │           → Symphony 自动执行 land skill 合并
        │           → 状态变为 Done
        │
        └── 拒绝 → Linear 状态改为 Rework
                    → Symphony 自动关闭旧 PR
                    → 新建分支从头执行
```

### 3.2 各阶段详细说明

#### 阶段 1：Agentic Engineer 规划

**触发条件：** 新需求涉及架构变更、多模块协同、或工程师判断需要先想清楚再动手。

**执行步骤：**

1. **灵感捕获** — 将原始需求写入 `inspiration.md`
2. **SDD 生成** — 使用 Claude Code 的 spec-driven-dev skill，根据模板生成结构化设计文档
3. **压力测试** — 对设计进行对抗性评估，输出评分卡（scorecard JSON）
4. **收敛判断** — 运行 `scorecard_parser.py` 检查是否所有关键问题已解决
5. **迭代修正** — 未收敛则修改设计，重复步骤 3-4（建议最多 3 轮）
6. **锁定输出** — 收敛后产出 `spec_final.md`

**质量门禁：**

```bash
# 结构验证
python3 tools/spec_lint.py spec/spec_final.md

# 收敛验证
python3 tools/scorecard_parser.py spec/scorecard_v1.json --format json
# 输出中 converged: true 才算通过

# 一致性验证
python3 tools/check_workflow_consistency.py
```

**产出物：**

| 文件 | 说明 |
|------|------|
| `spec_final.md` | 锁定的设计规范，包含模块划分、接口定义、验收标准 |
| `scorecard_v1.json` | 压力测试评分卡，证明设计经过审查 |

#### 阶段 1→2 过渡：工单拆解

将 `spec_final.md` 拆解为可独立执行的 Linear 工单。

**拆解原则：**

1. 每个工单对应 spec 中的一个独立模块或功能点
2. 工单之间的依赖关系用 Linear 的 `blockedBy` 表达
3. 每个工单必须有明确的验收标准（来自 spec）
4. 标注复杂度和风险级别

**工单模板：**

```markdown
Title: [模块名] 功能简述

## 来源
- Spec: spec_final.md § 章节号
- 父需求: [原始需求链接]

## 描述
[从 spec 中提取的具体要求]

## 验收标准
- [ ] 标准 1（来自 spec）
- [ ] 标准 2（来自 spec）
- [ ] 测试通过

## 建议 Adapter
- complexity:simple → claude/codex
- complexity:complex → symphony
```

**拆解方式：**

- **人工拆解**（推荐初期）：工程师阅读 spec，手动创建工单
- **半自动拆解**（未来）：用 Claude API 将 spec 拆为工单草稿，人工审核后导入 Linear

#### 阶段 2：OpenClaw (Zoe) 路由与审批

**任务入口：**

工程师在 OpenClaw Web UI 中创建任务，或从 Linear 工单自动导入。

**路由决策矩阵：**

| 条件 | 选择 Adapter | 理由 |
|------|-------------|------|
| 单文件修改、文档更新、简单 bug fix | `claude` | Claude Code CLI 足够 |
| 简单编码任务、单模块功能 | `codex` | Codex CLI 直接执行即可 |
| 多文件变更、需要多轮对话、需要 PR 管理 | `symphony` | 需要 Symphony 的深度执行能力 |
| 涉及 Linear 状态流转和 Workpad 管理 | `symphony` | Symphony 原生支持 |

**审批流程：**

1. Agent 执行完成后，任务进入 `awaiting_review`
2. 工程师在 OpenClaw UI 中查看代码变更
3. 批准：触发合并（直接合并或通知 Symphony 执行 land）
4. 拒绝：触发返工（直接重试或通知 Symphony 进入 Rework 流程）

#### 阶段 3：执行

**直接执行（adapter=claude/codex）：**

OpenClaw Worker 在 git worktree 中启动 Agent CLI，执行完成后等待审批。

**委托执行（adapter=symphony）：**

详见下一章"Symphony Adapter 技术实现"。

---

## 4. 技术实现

### 4.1 Symphony 侧改动：标签过滤

让 Symphony 只处理被 OpenClaw 路由过来的工单（约 20 行改动）。

**文件：`elixir/lib/symphony_elixir/config/schema.ex`**

新增配置字段：

```elixir
field :labels_include, {:array, :string}, default: []
```

**文件：`elixir/lib/symphony_elixir/orchestrator.ex`**

在 `candidate_issue?/3` 函数中追加标签检查：

```elixir
defp candidate_issue?(issue, active_states, terminal_states) do
  # ... 现有检查逻辑保持不变 ...
  and labels_match?(issue)
end

defp labels_match?(%Issue{labels: labels}) when is_list(labels) do
  required = Config.settings!().tracker[:labels_include] || []
  case required do
    [] -> true  # 未配置则不过滤（向后兼容）
    filter_labels ->
      normalized = Enum.map(filter_labels, &String.downcase/1)
      Enum.any?(normalized, &(&1 in labels))
  end
end

defp labels_match?(_issue), do: true
```

**文件：`elixir/WORKFLOW.md`**

```yaml
tracker:
  kind: linear
  project_slug: "your-project"
  labels_include:
    - "route:symphony"
  # ... 其余配置不变
```

### 4.2 OpenClaw 侧改动：Symphony Adapter

新增一个 adapter，让 OpenClaw Worker 可以将任务委托给 Symphony（约 250 行）。

**新增文件：`src/lib/adapters/symphony.ts`**

```typescript
import { LinearClient } from "@linear/sdk";

// ─── 配置 ───

interface SymphonyAdapterConfig {
  symphonyApiUrl: string;       // Symphony JSON API 地址
  linearApiKey: string;         // Linear API Key
  linearTeamId: string;         // Linear Team ID
  linearProjectId: string;      // Linear Project ID
  routeLabel: string;           // 路由标签名，默认 "route:symphony"
  pollIntervalMs: number;       // 状态轮询间隔（建议 10000ms）
  timeoutMs: number;            // 最大等待时间（建议 3600000ms = 1h）
}

// ─── 类型 ───

interface TaskInput {
  title: string;
  description: string;
}

interface AdapterResult {
  success: boolean;
  linearIdentifier?: string;
  prUrl?: string;
  error?: string;
  finalState?: string;
}

// ─── Adapter 实现 ───

export class SymphonyAdapter {
  private linear: LinearClient;
  private config: SymphonyAdapterConfig;

  constructor(config: SymphonyAdapterConfig) {
    this.config = config;
    this.linear = new LinearClient({ apiKey: config.linearApiKey });
  }

  /**
   * 执行流程：
   * 1. 在 Linear 创建带 route:symphony 标签的工单
   * 2. 触发 Symphony 立即轮询拾取
   * 3. 轮询监控直到 Symphony 完成（到达 Human Review）
   */
  async execute(task: TaskInput): Promise<AdapterResult> {
    // Step 1: 创建 Linear 工单
    const issue = await this.createLinearIssue(task);

    // Step 2: 触发 Symphony 轮询
    await this.triggerSymphonyRefresh();

    // Step 3: 等待 Symphony 执行完成
    return await this.pollUntilComplete(issue.identifier);
  }

  private async createLinearIssue(task: TaskInput) {
    // 查找 route:symphony 标签 ID
    const labels = await this.linear.issueLabels({
      filter: { name: { eq: this.config.routeLabel } },
    });
    const labelId = labels.nodes[0]?.id;
    if (!labelId) {
      throw new Error(
        `Label "${this.config.routeLabel}" not found in Linear`
      );
    }

    // 查找 Todo 状态 ID
    const team = await this.linear.team(this.config.linearTeamId);
    const states = await team.states();
    const todoState = states.nodes.find(
      (s) => s.name.toLowerCase() === "todo"
    );

    // 创建工单
    const result = await this.linear.createIssue({
      teamId: this.config.linearTeamId,
      projectId: this.config.linearProjectId,
      title: task.title,
      description: task.description,
      stateId: todoState?.id,
      labelIds: [labelId],
    });

    const created = await result.issue;
    if (!created) throw new Error("Failed to create Linear issue");
    return created;
  }

  private async triggerSymphonyRefresh(): Promise<void> {
    try {
      await fetch(`${this.config.symphonyApiUrl}/api/v1/refresh`, {
        method: "POST",
      });
    } catch {
      // Symphony 可能未启用 HTTP 端口，依赖下一个轮询周期（默认 5s）
    }
  }

  private async pollUntilComplete(
    identifier: string
  ): Promise<AdapterResult> {
    const startTime = Date.now();

    while (Date.now() - startTime < this.config.timeoutMs) {
      await new Promise((r) => setTimeout(r, this.config.pollIntervalMs));

      const linearState = await this.queryLinearState(identifier);
      const symphonyInfo = await this.querySymphonyStatus(identifier);

      // 到达 Human Review — Symphony 完成执行，等待 OpenClaw 审批
      if (linearState === "human review") {
        return {
          success: true,
          linearIdentifier: identifier,
          prUrl: symphonyInfo?.prUrl,
          finalState: "human_review",
        };
      }

      // 已完成
      if (linearState === "done") {
        return {
          success: true,
          linearIdentifier: identifier,
          finalState: "done",
        };
      }

      // 终止状态
      if (["closed", "cancelled", "canceled", "duplicate"].includes(linearState)) {
        return {
          success: false,
          linearIdentifier: identifier,
          error: `Reached terminal state: ${linearState}`,
          finalState: linearState,
        };
      }
    }

    return {
      success: false,
      linearIdentifier: identifier,
      error: `Timeout after ${this.config.timeoutMs}ms`,
    };
  }

  private async queryLinearState(identifier: string): Promise<string> {
    const issues = await this.linear.issues({
      filter: { identifier: { eq: identifier } },
    });
    const issue = issues.nodes[0];
    if (!issue) return "unknown";
    const state = await issue.state;
    return state?.name?.toLowerCase() ?? "unknown";
  }

  private async querySymphonyStatus(identifier: string) {
    try {
      const res = await fetch(
        `${this.config.symphonyApiUrl}/api/v1/${identifier}`
      );
      if (!res.ok) return null;
      return await res.json();
    } catch {
      return null;
    }
  }
}
```

**修改：OpenClaw Worker 的 adapter 选择逻辑**

在现有 adapter switch/map 中添加 symphony 选项：

```typescript
case "symphony":
  const symphonyAdapter = new SymphonyAdapter({
    symphonyApiUrl: process.env.SYMPHONY_API_URL!,
    linearApiKey: process.env.LINEAR_API_KEY!,
    linearTeamId: process.env.LINEAR_TEAM_ID!,
    linearProjectId: process.env.LINEAR_PROJECT_ID!,
    routeLabel: "route:symphony",
    pollIntervalMs: 10_000,
    timeoutMs: 3_600_000,
  });
  return symphonyAdapter.execute(task);
```

### 4.3 审批回调：OpenClaw → Symphony

当工程师在 OpenClaw 中做出审批决策时，需要同步到 Linear 触发 Symphony 响应。

```typescript
// 批准 → 通知 Symphony 合并
async function onSymphonyTaskApproved(linearIdentifier: string) {
  await transitionLinearState(linearIdentifier, "Merging");
  await triggerSymphonyRefresh();
  // Symphony 检测到 Merging 状态后自动执行 land skill 合并 PR
}

// 拒绝 → 通知 Symphony 返工
async function onSymphonyTaskRejected(linearIdentifier: string) {
  await transitionLinearState(linearIdentifier, "Rework");
  await triggerSymphonyRefresh();
  // Symphony 检测到 Rework 状态后自动关闭旧 PR、新建分支、重新执行
}

// 工具函数
async function transitionLinearState(identifier: string, targetStateName: string) {
  const linear = new LinearClient({ apiKey: process.env.LINEAR_API_KEY });
  const issues = await linear.issues({
    filter: { identifier: { eq: identifier } },
  });
  const issue = issues.nodes[0];
  if (!issue) throw new Error(`Issue ${identifier} not found`);

  const team = await issue.team;
  if (!team) throw new Error(`Team not found for ${identifier}`);
  const states = await team.states();
  const target = states.nodes.find(
    (s) => s.name.toLowerCase() === targetStateName.toLowerCase()
  );
  if (!target) throw new Error(`State "${targetStateName}" not found`);

  await issue.update({ stateId: target.id });
}

async function triggerSymphonyRefresh() {
  try {
    await fetch(`${process.env.SYMPHONY_API_URL}/api/v1/refresh`, {
      method: "POST",
    });
  } catch {
    // 非致命：Symphony 会在下一个轮询周期检测到状态变更
  }
}
```

### 4.4 工单拆解自动化（可选）

将 Agentic Engineer 的 `spec_final.md` 自动拆解为 Linear 工单。

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { LinearClient } from "@linear/sdk";

interface SubTask {
  title: string;
  description: string;
  acceptanceCriteria: string[];
  complexity: "simple" | "complex";
  dependencies: string[];       // 依赖的其他子任务 title
}

async function decomposeSpecToLinearIssues(
  specContent: string,
  projectId: string,
  teamId: string
) {
  const anthropic = new Anthropic();
  const linear = new LinearClient({ apiKey: process.env.LINEAR_API_KEY });

  // Step 1: 用 Claude 拆解 spec
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 4096,
    messages: [
      {
        role: "user",
        content: `将以下设计规范拆解为独立的、可执行的开发任务。

要求:
- 每个任务可以独立执行和验证
- 标注任务之间的依赖关系
- 标注复杂度: simple（单文件/单模块）或 complex（多文件/多轮对话）
- 每个任务必须有明确的验收标准

以 JSON 数组格式输出，每个元素包含:
  title, description, acceptanceCriteria (string[]),
  complexity ("simple"|"complex"), dependencies (string[])

设计规范:
${specContent}`,
      },
    ],
  });

  const subtasks: SubTask[] = JSON.parse(
    response.content[0].type === "text" ? response.content[0].text : ""
  );

  // Step 2: 在 Linear 中创建工单
  const createdIssues: Map<string, string> = new Map(); // title → issueId

  // 查找标签
  const labels = await linear.issueLabels();
  const simpleLabelId = labels.nodes.find(
    (l) => l.name === "complexity:simple"
  )?.id;
  const complexLabelId = labels.nodes.find(
    (l) => l.name === "complexity:complex"
  )?.id;
  const symphonyLabelId = labels.nodes.find(
    (l) => l.name === "route:symphony"
  )?.id;

  for (const task of subtasks) {
    const labelIds: string[] = [];
    if (task.complexity === "simple" && simpleLabelId) {
      labelIds.push(simpleLabelId);
    }
    if (task.complexity === "complex") {
      if (complexLabelId) labelIds.push(complexLabelId);
      if (symphonyLabelId) labelIds.push(symphonyLabelId);
    }

    const description = `${task.description}

## 验收标准
${task.acceptanceCriteria.map((c) => `- [ ] ${c}`).join("\n")}

## 来源
自动拆解自 spec_final.md`;

    const result = await linear.createIssue({
      teamId,
      projectId,
      title: task.title,
      description,
      labelIds,
    });

    const created = await result.issue;
    if (created) {
      createdIssues.set(task.title, created.id);
    }
  }

  // Step 3: 建立依赖关系
  for (const task of subtasks) {
    const issueId = createdIssues.get(task.title);
    if (!issueId) continue;

    for (const depTitle of task.dependencies) {
      const depId = createdIssues.get(depTitle);
      if (depId) {
        await linear.createIssueRelation({
          issueId,
          relatedIssueId: depId,
          type: "blocks",
        });
      }
    }
  }

  return {
    totalCreated: createdIssues.size,
    issues: Array.from(createdIssues.entries()).map(([title, id]) => ({
      title,
      id,
    })),
  };
}
```

---

## 5. 状态映射

### 5.1 跨系统状态对照表

```
Agentic Engineer    OpenClaw 状态          Linear 状态        Symphony 行为
────────────────────────────────────────────────────────────────────────────
规划中               —                     —                  —
spec_final 就绪      —                     —                  —
                    queued                 (工单刚创建)         —
                    in_progress            Todo               Symphony 即将拾取
                    in_progress            In Progress        Codex 多轮执行中
                    in_progress            Rework             返工执行中
                    awaiting_review        Human Review       执行完毕，等待审批
                    (approved)             Merging            Symphony 执行 land
                    merged                 Done               合并完成，清理工作空间
                    (rejected)             Rework             关闭旧 PR，重新执行
```

### 5.2 状态流转图

```
OpenClaw 创建任务
       │
       ▼
    queued ──→ Worker 认领 ──→ in_progress
                                   │
                                   │ (adapter=symphony)
                                   │ 创建 Linear 工单
                                   ▼
                          Symphony 拾取 (Todo)
                                   │
                                   ▼
                          Codex 执行 (In Progress)
                                   │
                                   ▼
                          Human Review ──→ OpenClaw: awaiting_review
                                                    │
                                          ┌─────────┴─────────┐
                                          ▼                   ▼
                                     批准                    拒绝
                                          │                   │
                                          ▼                   ▼
                                     Merging              Rework
                                          │                   │
                                          ▼                   ▼
                                     Symphony land       Symphony 重新执行
                                          │                   │
                                          ▼                   ▼
                                     Done → merged       回到 In Progress
```

---

## 6. 部署配置

### 6.1 部署拓扑

```
┌─────────────────── 单台服务器 ───────────────────┐
│                                                   │
│  ┌──────────────────┐  Port 4000                 │
│  │ Symphony         │  (Elixir escript)          │
│  │ WORKFLOW.md:     │                            │
│  │   labels_include:│                            │
│  │   - route:symphony                            │
│  └──────────────────┘                            │
│                                                   │
│  ┌──────────────────┐  Port 3000                 │
│  │ OpenClaw (Zoe)   │  (Next.js + Worker)        │
│  │ + symphony adapter                            │
│  └──────────────────┘                            │
│                                                   │
│  ┌──────────────────┐  按需运行                   │
│  │ Agentic Engineer │  (Python CLI tools)        │
│  └──────────────────┘                            │
│                                                   │
└───────────────────────────────────────────────────┘

外部依赖:
  - Linear (API + 工单管理)
  - Codex CLI (Symphony 调用)
  - Claude CLI (OpenClaw 直接调用)
```

### 6.2 环境变量

```bash
# ─── OpenClaw .env.local ───

# Agent adapter (per-task 可覆盖)
AGENT_ADAPTER=symphony

# Symphony 连接
SYMPHONY_API_URL=http://localhost:4000

# Linear 连接
LINEAR_API_KEY=lin_api_xxxxx
LINEAR_TEAM_ID=team_xxxxx
LINEAR_PROJECT_ID=project_xxxxx

# ─── Symphony ───
# 只需确保 WORKFLOW.md 中配置了 labels_include
# 环境变量沿用现有配置，无需新增

LINEAR_API_KEY=lin_api_xxxxx
```

### 6.3 Linear 设置

1. 创建以下标签：
   - `route:symphony` — 标记由 Symphony 执行的工单
   - `complexity:simple` — 简单任务（可选，用于路由参考）
   - `complexity:complex` — 复杂任务（可选）

2. 确保工单状态包含以下全部：
   - Todo, In Progress, Human Review, Rework, Merging, Done
   - （Closed, Cancelled, Duplicate 作为终止状态）

---

## 7. 实施计划

### 7.1 时间线

```
第 1 周    Agentic Engineer 规划流程建立
           ├─ 在目标仓库中配置 Claude Code spec-driven-dev skill
           ├─ 跑通一个端到端规划流程（灵感→spec_final.md）
           └─ 建立规划质量门禁（spec_lint + scorecard 收敛）

第 2 周    Symphony 标签过滤 + OpenClaw Symphony Adapter
           ├─ Symphony: 添加 labels_include 配置（3 个文件，~20 行）
           ├─ OpenClaw: 实现 SymphonyAdapter（1 个新文件，~200 行）
           ├─ OpenClaw: 添加审批回调（~50 行）
           └─ 端到端冒烟测试

第 3 周    工单拆解流程 + 集成测试
           ├─ 手动拆解 spec → Linear 工单（验证流程）
           ├─ 可选：实现自动拆解脚本
           └─ 全流程集成测试（规划→拆解→路由→执行→审批→合并）

第 4 周    试运行 + 调优
           ├─ 选择 3-5 个真实任务走完整流水线
           ├─ 调优路由决策（哪些任务走 symphony vs claude/codex）
           ├─ 调优 Symphony WORKFLOW.md prompt
           └─ 文档化运维手册
```

### 7.2 验收标准

| 里程碑 | 验收条件 |
|--------|---------|
| Agentic Engineer 就绪 | 能产出通过 spec_lint + scorecard 收敛的 spec_final.md |
| Symphony Adapter 就绪 | OpenClaw 创建 symphony 任务 → Symphony 自动拾取执行 → OpenClaw 收到完成通知 |
| 审批流程就绪 | OpenClaw 批准 → Symphony 自动合并；OpenClaw 拒绝 → Symphony 自动返工 |
| 全流程打通 | 从模糊需求到代码合并，端到端跑通一个真实任务 |

### 7.3 改动量汇总

| 系统 | 改动 | 代码量 |
|------|------|--------|
| Symphony | `config/schema.ex` + `orchestrator.ex` + `WORKFLOW.md` | ~20 行 |
| OpenClaw | `adapters/symphony.ts`（新）+ adapter 选择 + 审批回调 | ~250 行 |
| Agentic Engineer | 无代码改动，只需在目标仓库配置 skill | 0 行 |
| Linear | 创建标签 + 确认状态 | 配置项 |
| **合计** | | **~270 行** |

---

## 8. 风险与缓解

| 风险 | 影响 | 概率 | 缓解 |
|------|------|------|------|
| Symphony 轮询延迟 | 工单拾取有 ≤5s 延迟 | 低 | `POST /api/v1/refresh` 触发即时轮询 |
| Linear API 限流 | 高频轮询被限制 | 中 | OpenClaw 监控轮询间隔设为 10s+ |
| Symphony 进程崩溃 | 正在执行的任务中断 | 低 | OTP Supervisor 自动重启 + 状态协调恢复 |
| 状态不一致 | OpenClaw 和 Linear 状态不匹配 | 中 | OpenClaw 始终以 Linear 状态为准（查询而非缓存） |
| Agentic Engineer 规划不收敛 | 任务卡在规划阶段 | 中 | 设最大迭代次数（3 轮），超时由人工接管 |
| Spec 拆解粒度不当 | 工单太大或太碎 | 中 | 初期人工拆解积累经验，再逐步自动化 |

---

## 9. 未来演进

### 短期（1-3 月）

- 完善路由决策：基于历史执行数据优化 adapter 选择
- 自动化工单拆解：Claude API + 人工审核
- 统一监控面板：在 OpenClaw UI 中嵌入 Symphony 状态

### 中期（3-6 月）

- Symphony 添加 `POST /api/v1/issues` 端点，支持直接推送工单（绕过 Linear）
- OpenClaw 添加 Agentic Engineer 集成，规划流程自动触发
- 基于任务执行结果的路由规则自学习

### 长期（6-12 月）

- 将 Symphony 的 AgentRunner 执行能力抽象为独立库
- OpenClaw 原生集成 Symphony 执行引擎（不再需要 Linear 作为中间层）
- 支持更多 Agent 后端（Gemini、Cursor 等）

---

## 附录 A: 参考仓库

| 项目 | 仓库 | 定位 |
|------|------|------|
| Symphony | github.com/openai/symphony | 深度执行引擎（Elixir） |
| Agentic Engineer | github.com/SiriusYou/agentic-engineer | 规划方法论框架（Python） |
| OpenClaw (Zoe) | github.com/SiriusYou/openclaw-healthcare | 编排控制台（TypeScript） |

## 附录 B: 关键 API 参考

### Symphony JSON API

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/v1/state` | GET | 当前所有运行中/重试中的工单概览 |
| `/api/v1/{identifier}` | GET | 特定工单的详细执行状态 |
| `/api/v1/refresh` | POST | 触发 Symphony 立即轮询 Linear |

### Linear GraphQL（OpenClaw 使用）

| 操作 | 用途 |
|------|------|
| `createIssue` | 创建带标签的工单 |
| `issueUpdate` | 修改工单状态（审批回调） |
| `issues(filter: {identifier})` | 查询工单当前状态 |
| `createIssueRelation` | 建立工单依赖关系 |
