# OpenClaw (Zoe) + Symphony 整合方案：Symphony 作为 OpenClaw 的执行后端

> 核心问题：Symphony 能否置于 OpenClaw 编排层之下，由 OpenClaw 做上层路由，Symphony 做二次执行？

## 1. 技术可行性分析

### 1.1 Symphony 的可控制性（源码级分析）

| 能力 | 现状 | 结论 |
|------|------|------|
| 关闭独立轮询 | Orchestrator 启动即轮询，无 `--no-poll` 开关 | **不可直接关闭** |
| 外部注入工单 | 无 `POST /issues` 接口，无 `inject_issue` API | **不可直接推送** |
| 替换 Tracker 数据源 | `tracker.ex` 有 Behaviour 抽象，`kind: memory` 已实现 | **可以替换** |
| 直接调用 AgentRunner | `AgentRunner.run/3` 接口干净，不依赖 Orchestrator 状态 | **可以直接调用** |
| 通过 Linear 标签间接控制 | Symphony 采集标签但不过滤（需少量代码新增过滤逻辑） | **可行，改动小** |
| 通过 API 触发轮询 | `POST /api/v1/refresh` 触发立即轮询 | **已支持** |
| 通过 API 查询状态 | `GET /api/v1/{identifier}` 返回执行详情 | **已支持** |

### 1.2 关键发现

**有利因素：**

1. **Tracker 是抽象层**（`tracker.ex:40-45`）：
   ```elixir
   def adapter do
     case Config.settings!().tracker.kind do
       "memory" -> SymphonyElixir.Tracker.Memory
       _ -> SymphonyElixir.Linear.Adapter
     end
   end
   ```
   只要实现 5 个 callback 就能替换整个数据源。

2. **AgentRunner 高度独立**（`agent_runner.ex:13`）：
   ```elixir
   def run(issue, codex_update_recipient \\ nil, opts \\ [])
   ```
   只需要一个 Issue struct + 可选回调 PID，不耦合 Orchestrator。

3. **JSON API 已就绪**：`/api/v1/state`、`/api/v1/{identifier}`、`/api/v1/refresh` 提供完整的外部监控能力。

**阻碍因素：**

1. **Symphony 是守护进程，不是库** — 启动即拥有自己的轮询循环和生命周期管理
2. **无法直接推送工单** — 只能通过修改 Linear 工单状态/标签让 Symphony 间接拾取
3. **双重生命周期冲突** — Symphony 的状态机（Todo→In Progress→Human Review→Done）和 OpenClaw 的状态机（queued→in_progress→awaiting_review→merged）会冲突

---

## 2. 推荐方案：Symphony 作为 OpenClaw 的 Agent Adapter

### 2.1 架构

```
OpenClaw (Zoe) ── 顶层编排，人工控制
│
├─ adapter=claude    → Claude CLI 直接执行（简单任务）
├─ adapter=codex     → Codex CLI 直接执行（简单任务）
│
└─ adapter=symphony  → 交给 Symphony 执行（复杂任务）
     │
     │  ① OpenClaw worker 在 Linear 上打标签
     │  ② POST /api/v1/refresh 触发 Symphony 拾取
     │  ③ 轮询 GET /api/v1/{id} 监控进度
     │  ④ Symphony 完成后 OpenClaw 进入 awaiting_review
     │
     Symphony (独立守护进程)
     ├─ 仅处理 labels_include: ["route:symphony"] 的工单
     ├─ 多轮 Codex 对话 (max_turns: 20)
     ├─ 自动 PR 创建 + Linear 状态流转
     └─ 完整的 prompt 工程 (WORKFLOW.md)
```

### 2.2 工作流时序

```
工程师                  OpenClaw (Zoe)              Linear                Symphony
  │                        │                          │                      │
  ├─ 创建任务 ────────────→│                          │                      │
  │  (选择 adapter=symphony)                          │                      │
  │                        │                          │                      │
  │                        ├─ worker 认领任务          │                      │
  │                        ├─ 创建 Linear 工单 ──────→│                      │
  │                        │  label: route:symphony   │                      │
  │                        │  state: Todo              │                      │
  │                        │                          │                      │
  │                        ├─ POST /api/v1/refresh ──────────────────────────→│
  │                        │                          │                      ├─ 轮询拾取
  │                        │                          │                      ├─ 创建工作空间
  │                        │                          │                      ├─ 启动 Codex
  │                        │                          │←── 状态→In Progress ──┤
  │                        │                          │                      ├─ 多轮执行...
  │                        │                          │                      ├─ 创建 PR
  │                        │                          │←── 状态→Human Review ─┤
  │                        │                          │                      │
  │                        ├─ GET /api/v1/SYM-123 ──────────────────────────→│
  │                        │  ← status: human_review  │                      │
  │                        │                          │                      │
  │                        ├─ 标记 awaiting_review     │                      │
  │                        │                          │                      │
  ├─ 审批/拒绝 ──────────→│                          │                      │
  │                        ├─ 如果批准：               │                      │
  │                        │  更新 Linear 状态 ──────→│←── 状态→Merging ─────→│
  │                        │                          │                      ├─ 执行 land skill
  │                        │                          │←── 状态→Done ────────┤
  │                        ├─ 标记 merged              │                      │
  │                        │                          │                      │
  │                        ├─ 如果拒绝：               │                      │
  │                        │  更新 Linear 状态 ──────→│←── 状态→Rework ──────→│
  │                        │                          │                      ├─ 重新执行
```

### 2.3 谁拥有什么

| 职责 | 归属 | 原因 |
|------|------|------|
| 任务入口 | **OpenClaw** | 统一入口，人工创建或批量导入 |
| 路由决策 | **OpenClaw** | 决定用哪个 adapter |
| Agent 执行 | **Symphony** | 多轮对话 + 工作空间隔离 + prompt 工程 |
| PR 创建 | **Symphony** | 由 Codex 在 WORKFLOW.md 指导下完成 |
| Linear 状态管理 | **Symphony** | 保持其核心价值（状态机 + 评论管理） |
| 人工审批 | **OpenClaw** | 统一审批界面 |
| 最终合并决策 | **OpenClaw → Symphony** | OpenClaw 批准后通知 Symphony 执行 land |

---

## 3. 具体实现

### 3.1 Symphony 侧：添加标签过滤（3 个文件，~30 行）

让 Symphony 只处理被 OpenClaw 路由过来的工单。

**文件 1：`elixir/lib/symphony_elixir/config/schema.ex`**

在 tracker schema 中新增 `labels_include` 字段：

```elixir
field :labels_include, {:array, :string}, default: []
```

**文件 2：`elixir/lib/symphony_elixir/orchestrator.ex`**

在 `candidate_issue?/3`（约 line 591）后追加标签检查：

```elixir
defp candidate_issue?(issue, active_states, terminal_states) do
  # ... 现有逻辑 ...
  and labels_match?(issue)
end

defp labels_match?(%Issue{labels: labels}) when is_list(labels) do
  required = Config.settings!().tracker[:labels_include] || []
  case required do
    [] -> true
    filter_labels ->
      normalized = Enum.map(filter_labels, &String.downcase/1)
      Enum.any?(normalized, &(&1 in labels))
  end
end

defp labels_match?(_issue), do: true
```

**文件 3：`elixir/WORKFLOW.md`**

```yaml
tracker:
  kind: linear
  project_slug: "your-project"
  labels_include:
    - "route:symphony"
  # ... 其余不变
```

### 3.2 OpenClaw 侧：新增 Symphony Adapter（~200 行 TypeScript）

**新增文件：`src/lib/adapters/symphony.ts`**

```typescript
import { LinearClient } from "@linear/sdk";

interface SymphonyAdapterConfig {
  symphonyApiUrl: string;      // e.g. http://localhost:4000
  linearApiKey: string;
  linearTeamId: string;
  linearProjectId: string;
  pollIntervalMs: number;      // 监控轮询间隔，建议 10000
  timeoutMs: number;           // 最大等待时间，建议 3600000 (1h)
}

interface TaskInput {
  title: string;
  description: string;
  prompt: string;
  metadata?: Record<string, unknown>;
}

interface AdapterResult {
  success: boolean;
  linearIdentifier?: string;
  prUrl?: string;
  error?: string;
}

export class SymphonyAdapter {
  private linear: LinearClient;
  private config: SymphonyAdapterConfig;

  constructor(config: SymphonyAdapterConfig) {
    this.config = config;
    this.linear = new LinearClient({ apiKey: config.linearApiKey });
  }

  /**
   * 主执行流程：
   * 1. 在 Linear 创建工单（带 route:symphony 标签）
   * 2. 触发 Symphony 拾取
   * 3. 轮询监控直到完成
   */
  async execute(task: TaskInput): Promise<AdapterResult> {
    // Step 1: 在 Linear 创建工单
    const issue = await this.createLinearIssue(task);
    console.log(`[symphony-adapter] Created Linear issue: ${issue.identifier}`);

    // Step 2: 触发 Symphony 立即轮询
    await this.triggerSymphonyRefresh();

    // Step 3: 轮询监控 Symphony 执行状态
    const result = await this.pollUntilComplete(issue.identifier);

    return result;
  }

  private async createLinearIssue(task: TaskInput) {
    // 查找 route:symphony 标签
    const labels = await this.linear.issueLabels({
      filter: { name: { eq: "route:symphony" } },
    });
    const labelId = labels.nodes[0]?.id;

    // 查找 Todo 状态
    const team = await this.linear.team(this.config.linearTeamId);
    const states = await team.states();
    const todoState = states.nodes.find(
      (s) => s.name.toLowerCase() === "todo"
    );

    const issue = await this.linear.createIssue({
      teamId: this.config.linearTeamId,
      projectId: this.config.linearProjectId,
      title: task.title,
      description: task.description,
      stateId: todoState?.id,
      labelIds: labelId ? [labelId] : [],
    });

    const created = await issue.issue;
    if (!created) throw new Error("Failed to create Linear issue");
    return created;
  }

  private async triggerSymphonyRefresh(): Promise<void> {
    try {
      const res = await fetch(`${this.config.symphonyApiUrl}/api/v1/refresh`, {
        method: "POST",
      });
      if (!res.ok) {
        console.warn(`[symphony-adapter] Refresh returned ${res.status}`);
      }
    } catch (err) {
      // Symphony 可能未启用 HTTP；下一个轮询周期会自动拾取
      console.warn(`[symphony-adapter] Could not trigger refresh: ${err}`);
    }
  }

  private async pollUntilComplete(
    identifier: string
  ): Promise<AdapterResult> {
    const startTime = Date.now();

    while (Date.now() - startTime < this.config.timeoutMs) {
      await sleep(this.config.pollIntervalMs);

      // 查询 Symphony API
      const symphonyStatus = await this.querySymphony(identifier);

      // 查询 Linear 工单状态（Symphony 可能已经推进到 Human Review / Done）
      const linearStatus = await this.queryLinearState(identifier);

      console.log(
        `[symphony-adapter] ${identifier}: symphony=${symphonyStatus?.status ?? "not_found"} linear=${linearStatus}`
      );

      // 完成条件：Linear 状态到达 Human Review 或 Done
      if (linearStatus === "human review") {
        // Symphony 完成了执行，等待 OpenClaw 人工审批
        return {
          success: true,
          linearIdentifier: identifier,
          prUrl: symphonyStatus?.pr_url,
        };
      }

      if (linearStatus === "done") {
        return {
          success: true,
          linearIdentifier: identifier,
        };
      }

      // 终止条件
      const terminalStates = ["closed", "cancelled", "canceled", "duplicate"];
      if (terminalStates.includes(linearStatus)) {
        return {
          success: false,
          linearIdentifier: identifier,
          error: `Issue reached terminal state: ${linearStatus}`,
        };
      }
    }

    return {
      success: false,
      linearIdentifier: identifier,
      error: `Timeout after ${this.config.timeoutMs}ms`,
    };
  }

  private async querySymphony(
    identifier: string
  ): Promise<{ status: string; pr_url?: string } | null> {
    try {
      const res = await fetch(
        `${this.config.symphonyApiUrl}/api/v1/${identifier}`
      );
      if (res.status === 404) return null;
      if (!res.ok) return null;
      return await res.json();
    } catch {
      return null;
    }
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
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
```

**修改文件：OpenClaw worker 的 adapter 选择逻辑**

在现有 adapter switch 中添加：

```typescript
case "symphony":
  const adapter = new SymphonyAdapter({
    symphonyApiUrl: process.env.SYMPHONY_API_URL!,
    linearApiKey: process.env.LINEAR_API_KEY!,
    linearTeamId: process.env.LINEAR_TEAM_ID!,
    linearProjectId: process.env.LINEAR_PROJECT_ID!,
    pollIntervalMs: 10_000,
    timeoutMs: 3_600_000,
  });
  return adapter.execute(task);
```

### 3.3 OpenClaw 审批后的合并触发

当人工在 OpenClaw 中批准任务时，需要通知 Symphony 执行合并：

```typescript
// OpenClaw: 审批通过回调
async function onTaskApproved(task: Task) {
  const meta = JSON.parse(task.metadata || "{}");

  if (meta.adapter === "symphony" && meta.linearIdentifier) {
    // 将 Linear 工单状态从 Human Review 改为 Merging
    // Symphony 会自动执行 land skill 合并 PR
    const linear = new LinearClient({ apiKey: process.env.LINEAR_API_KEY });
    const issues = await linear.issues({
      filter: { identifier: { eq: meta.linearIdentifier } },
    });
    const issue = issues.nodes[0];
    if (issue) {
      const team = await issue.team;
      const states = await team?.states();
      const mergingState = states?.nodes.find(
        (s) => s.name.toLowerCase() === "merging"
      );
      if (mergingState) {
        await issue.update({ stateId: mergingState.id });
      }
    }

    // 触发 Symphony 轮询，让它看到状态变更并执行 land
    await fetch(`${process.env.SYMPHONY_API_URL}/api/v1/refresh`, {
      method: "POST",
    });
  }
}

// OpenClaw: 审批拒绝回调
async function onTaskRejected(task: Task) {
  const meta = JSON.parse(task.metadata || "{}");

  if (meta.adapter === "symphony" && meta.linearIdentifier) {
    // 将 Linear 工单状态改为 Rework
    // Symphony 会自动关闭旧 PR、新建分支、重新执行
    const linear = new LinearClient({ apiKey: process.env.LINEAR_API_KEY });
    const issues = await linear.issues({
      filter: { identifier: { eq: meta.linearIdentifier } },
    });
    const issue = issues.nodes[0];
    if (issue) {
      const team = await issue.team;
      const states = await team?.states();
      const reworkState = states?.nodes.find(
        (s) => s.name.toLowerCase() === "rework"
      );
      if (reworkState) {
        await issue.update({ stateId: reworkState.id });
      }
    }

    await fetch(`${process.env.SYMPHONY_API_URL}/api/v1/refresh`, {
      method: "POST",
    });
  }
}
```

---

## 4. 配置清单

### 4.1 环境变量

```bash
# OpenClaw .env.local 新增
AGENT_ADAPTER=symphony              # 或保持原值，per-task 选择
SYMPHONY_API_URL=http://localhost:4000
LINEAR_API_KEY=lin_api_xxxxx
LINEAR_TEAM_ID=team_xxxxx
LINEAR_PROJECT_ID=project_xxxxx

# Symphony 无需新增环境变量，只改 WORKFLOW.md
```

### 4.2 Linear 设置

1. 创建标签 `route:symphony`
2. 确保工单状态包含：Todo, In Progress, Human Review, Merging, Rework, Done
   （Symphony 的 WORKFLOW.md 已依赖这些状态）

### 4.3 Symphony WORKFLOW.md

唯一改动 — 添加标签过滤：

```yaml
tracker:
  labels_include:
    - "route:symphony"
```

---

## 5. 状态映射

两个系统的状态需要对齐：

```
OpenClaw 状态           ←→    Linear 状态        ←→    Symphony 含义
─────────────────────────────────────────────────────────────────────
queued                  →     (工单未创建)              (尚未分发)
in_progress             ←     Todo                      Symphony 即将拾取
in_progress             ←     In Progress               Codex 执行中
in_progress             ←     Rework                    返工执行中
awaiting_review         ←     Human Review              Symphony 执行完毕，等待审批
(approved callback)     →     Merging                   OpenClaw 批准，触发合并
merged                  ←     Done                      合并完成
rejected                →     Rework                    OpenClaw 拒绝，触发返工
```

---

## 6. 改动量总结

| 系统 | 文件 | 改动量 | 说明 |
|------|------|--------|------|
| **Symphony** | `config/schema.ex` | +3 行 | 新增 `labels_include` 字段 |
| **Symphony** | `orchestrator.ex` | +15 行 | 新增 `labels_match?/1` 过滤函数 |
| **Symphony** | `WORKFLOW.md` | +2 行 | 添加 `labels_include` 配置 |
| **OpenClaw** | `adapters/symphony.ts` | ~200 行（新文件） | Symphony adapter 实现 |
| **OpenClaw** | worker adapter 选择 | +5 行 | 添加 `case "symphony"` |
| **OpenClaw** | 审批回调 | +40 行 | 批准→Merging / 拒绝→Rework |
| **Linear** | 配置 | — | 创建 `route:symphony` 标签 |

**总改动：Symphony ~20 行，OpenClaw ~250 行，无需新建独立服务。**

---

## 7. 此方案的优势与局限

### 优势

1. **OpenClaw 保持完全控制** — 任务入口、路由决策、人工审批全在 OpenClaw
2. **Symphony 保持独立运行** — 不需要改架构，只加了一个标签过滤
3. **利用 Symphony 的核心价值** — 多轮 Codex 对话、WORKFLOW.md prompt 工程、Linear 状态管理、自动 PR
4. **改动量极小** — 两边加起来不到 300 行
5. **通过 Linear 解耦** — 两个系统通过 Linear 工单状态通信，不需要直接 RPC

### 局限

1. **间接控制** — OpenClaw 对 Symphony 的控制是通过 Linear 状态间接实现的，不是直接 API 调用
2. **延迟** — 状态变更依赖 Symphony 的轮询周期（默认 5s），不是实时的
3. **两套 UI** — Symphony 有自己的 LiveView 仪表盘，OpenClaw 也有 Web UI，运维需要看两个地方
4. **Linear 依赖** — 此方案强依赖 Linear 作为中间层；如果不用 Linear，需要走方案 B（Memory Tracker）
5. **Symphony 仍是独立进程** — 需要单独部署和运维

### 未来演进

如果 OpenClaw 需要更深度的控制，可以逐步推进：
- **Phase 2**: 给 Symphony 添加 `POST /api/v1/issues` 端点，直接推送工单（绕过 Linear）
- **Phase 3**: 把 Symphony 的 AgentRunner 抽成独立库，OpenClaw 直接调用（绕过 Orchestrator）
- **最终态**: Symphony 的执行引擎用 TypeScript 重写，原生集成到 OpenClaw 中
