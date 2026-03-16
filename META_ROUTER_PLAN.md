# Meta-Router 整合实施方案

> 将 Symphony、Agentic Engineer、OpenClaw 三个系统通过上层路由器串联为统一的 AI 编码工作流平台。

## 1. 目标架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Linear (单一事实来源)                         │
│  工单标签: route:auto / route:review / route:design             │
└──────────────────────────┬──────────────────────────────────────┘
                           │ Webhook (issue.create / issue.update)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Meta-Router Service                         │
│  ┌───────────┐  ┌──────────────┐  ┌──────────────────────┐     │
│  │ Webhook   │  │ Route Engine │  │ State Sync           │     │
│  │ Listener  │→ │ (决策+分发)   │→ │ (跨系统状态同步)      │     │
│  └───────────┘  └──────────────┘  └──────────────────────┘     │
│  ┌───────────────────────────────────────────────────────┐     │
│  │ Dashboard API (统一查询各子系统状态)                     │     │
│  └───────────────────────────────────────────────────────┘     │
└──────────┬──────────────────┬──────────────────┬───────────────┘
           │                  │                  │
           ▼                  ▼                  ▼
    ┌────────────┐     ┌───────────┐     ┌────────────┐
    │  Agentic   │     │ Symphony  │     │  OpenClaw  │
    │  Engineer  │     │ (全自动)   │     │ (人工审批)  │
    │  (规划)     │     │           │     │            │
    └────────────┘     └───────────┘     └────────────┘
```

## 2. 分阶段实施计划

### Phase 0: 准备工作（第 1 周）

#### 0.1 统一 Linear 项目设置

在 Linear 中创建统一的工单标签体系：

```
路由标签 (互斥，必选一个):
  route:auto        → Symphony 全自动处理
  route:review      → OpenClaw 人工审批处理
  route:design      → Agentic Engineer 先规划再执行

风险标签 (可选):
  risk:high         → 强制走 OpenClaw
  risk:low          → 允许 Symphony

复杂度标签 (可选):
  complexity:arch   → 涉及架构变更，强制走 design
  complexity:simple → 简单修改

Agent 偏好标签 (可选，仅 OpenClaw 通道):
  agent:codex
  agent:claude
  agent:gemini
```

#### 0.2 为 Symphony 添加标签过滤

Symphony 目前采集标签但**不做过滤**。需要添加标签过滤能力，让它只处理 `route:auto` 标签的工单。

**改动文件：**

`elixir/lib/symphony_elixir/config/schema.ex` — 添加 `labels_include` 配置字段：

```elixir
# 在 tracker schema 中新增
field :labels_include, {:array, :string}, default: []
```

`elixir/lib/symphony_elixir/orchestrator.ex` — 修改 `candidate_issue?/3`：

```elixir
# 在现有过滤逻辑后追加（约 line 591-607）
defp candidate_issue?(issue, active_states, terminal_states) do
  # ... 现有检查 ...
  and labels_match?(issue)
end

defp labels_match?(issue) do
  required = Config.settings!().tracker[:labels_include] || []
  case required do
    [] -> true  # 未配置则不过滤
    labels -> Enum.any?(labels, &(&1 in issue.labels))
  end
end
```

`elixir/WORKFLOW.md` — 添加标签过滤配置：

```yaml
tracker:
  kind: linear
  project_slug: "your-project"
  labels_include:
    - "route:auto"
  # ... 其余配置不变
```

#### 0.3 为 OpenClaw 添加 REST API

OpenClaw 目前只有 Web UI 创建任务，**没有 REST API**。需要添加任务创建接口。

**新增文件：** `src/app/api/tasks/route.ts`

```typescript
import { db } from "@/lib/db";
import { tasks } from "@/lib/schema";
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  const body = await req.json();

  const task = await db.insert(tasks).values({
    title: body.title,
    description: body.description,
    prompt: body.prompt,
    status: "queued",
    metadata: JSON.stringify({
      linear_id: body.linear_id,
      linear_identifier: body.linear_identifier,
      linear_url: body.linear_url,
      agent_adapter: body.agent_adapter || process.env.AGENT_ADAPTER,
      routed_by: "meta-router",
    }),
  }).returning();

  return NextResponse.json(task[0], { status: 201 });
}

export async function GET() {
  const allTasks = await db.select().from(tasks);
  return NextResponse.json(allTasks);
}
```

**新增文件：** `src/app/api/tasks/[id]/route.ts`

```typescript
import { db } from "@/lib/db";
import { tasks } from "@/lib/schema";
import { eq } from "drizzle-orm";
import { NextRequest, NextResponse } from "next/server";

export async function GET(_req: NextRequest, { params }: { params: { id: string } }) {
  const task = await db.select().from(tasks).where(eq(tasks.id, params.id));
  if (!task.length) return NextResponse.json({ error: "not_found" }, { status: 404 });
  return NextResponse.json(task[0]);
}
```

---

### Phase 1: Meta-Router 核心服务（第 2-3 周）

#### 1.1 技术选型

```
语言:       TypeScript (与 OpenClaw 统一生态)
运行时:     Node.js 20+ / Bun
框架:       Hono (轻量 HTTP 框架)
数据库:     SQLite (Drizzle ORM，与 OpenClaw 共享技术栈)
部署:       单进程，与 OpenClaw 可共用机器
```

#### 1.2 项目结构

```
meta-router/
├── src/
│   ├── index.ts                 # 入口：HTTP server
│   ├── webhook/
│   │   └── linear.ts            # Linear webhook handler
│   ├── router/
│   │   ├── engine.ts            # 路由决策引擎
│   │   ├── rules.ts             # 路由规则定义
│   │   └── types.ts             # 统一任务模型
│   ├── dispatchers/
│   │   ├── symphony.ts          # Symphony 分发器
│   │   ├── openclaw.ts          # OpenClaw 分发器
│   │   └── agentic.ts           # Agentic Engineer 分发器
│   ├── sync/
│   │   ├── state-sync.ts        # 跨系统状态同步
│   │   └── linear-updater.ts    # 回写 Linear 状态
│   ├── dashboard/
│   │   └── api.ts               # 统一查询 API
│   └── db/
│       ├── schema.ts            # 路由记录表
│       └── client.ts            # 数据库连接
├── package.json
├── drizzle.config.ts
├── .env.example
└── tsconfig.json
```

#### 1.3 统一任务模型

```typescript
// src/router/types.ts

export interface UnifiedTask {
  // 来源
  id: string;                          // meta-router 内部 ID
  linear_id: string;                   // Linear issue ID
  linear_identifier: string;           // e.g. "SYM-123"
  linear_url: string;

  // 工单内容
  title: string;
  description: string | null;
  priority: number | null;             // 1=urgent, 4=low
  labels: string[];
  state: string;

  // 路由决策
  route_decision: RouteDecision;
  routed_at: string;                   // ISO8601
  route_reason: string;                // 人类可读的决策原因

  // 执行状态
  execution_status: ExecutionStatus;
  target_system: "symphony" | "openclaw" | "agentic-engineer";
  agent_adapter?: "codex" | "claude" | "gemini";

  // 跟踪
  created_at: string;
  updated_at: string;
  completed_at?: string;
}

export type RouteDecision = "auto" | "review" | "design";

export type ExecutionStatus =
  | "routed"              // 已决策，待分发
  | "dispatched"          // 已分发到目标系统
  | "planning"            // Agentic Engineer 规划中
  | "plan_ready"          // 规划完成，待拆解
  | "in_progress"         // 执行中
  | "awaiting_review"     // 等待人工审批 (OpenClaw)
  | "human_review"        // 等待人工审批 (Symphony)
  | "merging"             // 合并中
  | "completed"           // 完成
  | "failed"              // 失败
```

#### 1.4 路由决策引擎

```typescript
// src/router/engine.ts

import { LinearIssue, RouteDecision } from "./types";

interface RouteResult {
  decision: RouteDecision;
  reason: string;
  agent_adapter?: string;
}

export function routeIssue(issue: LinearIssue): RouteResult {
  const labels = new Set(issue.labels.map(l => l.toLowerCase()));

  // ── 规则 1: 显式标签路由（最高优先级）──
  if (labels.has("route:design")) {
    return { decision: "design", reason: "显式标签 route:design" };
  }
  if (labels.has("route:review")) {
    return {
      decision: "review",
      reason: "显式标签 route:review",
      agent_adapter: extractAgentPreference(labels),
    };
  }
  if (labels.has("route:auto")) {
    return { decision: "auto", reason: "显式标签 route:auto" };
  }

  // ── 规则 2: 风险标签推断 ──
  if (labels.has("risk:high")) {
    return { decision: "review", reason: "高风险标签，需人工审批" };
  }

  // ── 规则 3: 复杂度推断 ──
  if (labels.has("complexity:arch")) {
    return { decision: "design", reason: "架构级复杂度，需先规划" };
  }

  // ── 规则 4: 优先级推断 ──
  if (issue.priority !== null && issue.priority <= 1) {
    // urgent / high priority → 人工审批更安全
    return { decision: "review", reason: `高优先级 (P${issue.priority})，走人工审批` };
  }

  // ── 规则 5: 默认路由 ──
  return { decision: "auto", reason: "无特殊标签，默认全自动" };
}

function extractAgentPreference(labels: Set<string>): string | undefined {
  if (labels.has("agent:claude")) return "claude";
  if (labels.has("agent:codex")) return "codex";
  if (labels.has("agent:gemini")) return "gemini";
  return undefined;
}
```

#### 1.5 Linear Webhook Handler

```typescript
// src/webhook/linear.ts

import { Hono } from "hono";
import { routeIssue } from "../router/engine";
import { dispatchToSymphony } from "../dispatchers/symphony";
import { dispatchToOpenClaw } from "../dispatchers/openclaw";
import { dispatchToAgenticEngineer } from "../dispatchers/agentic";
import { db } from "../db/client";
import { routingRecords } from "../db/schema";

const app = new Hono();

app.post("/webhook/linear", async (c) => {
  const payload = await c.req.json();
  const { action, data, type } = payload;

  // 只处理 Issue 事件
  if (type !== "Issue") return c.json({ ok: true });

  // 只处理创建和状态变更
  if (action !== "create" && action !== "update") {
    return c.json({ ok: true });
  }

  // 对于 update，只关心状态变更
  if (action === "update" && !payload.updatedFrom?.stateId) {
    return c.json({ ok: true });
  }

  const issue = normalizeLinearWebhookPayload(data);

  // 跳过终止状态的工单
  const terminalStates = new Set(["done", "closed", "cancelled", "canceled", "duplicate"]);
  if (terminalStates.has(issue.state.toLowerCase())) {
    return c.json({ ok: true, skipped: "terminal_state" });
  }

  // 检查是否已路由过
  const existing = await db.select()
    .from(routingRecords)
    .where(eq(routingRecords.linear_id, issue.id))
    .limit(1);

  if (existing.length > 0 && existing[0].execution_status !== "failed") {
    return c.json({ ok: true, skipped: "already_routed" });
  }

  // 路由决策
  const route = routeIssue(issue);

  // 记录路由决策
  const record = await db.insert(routingRecords).values({
    linear_id: issue.id,
    linear_identifier: issue.identifier,
    title: issue.title,
    route_decision: route.decision,
    route_reason: route.reason,
    target_system: decisionToSystem(route.decision),
    agent_adapter: route.agent_adapter,
    execution_status: "routed",
  }).returning();

  // 分发到目标系统
  try {
    switch (route.decision) {
      case "auto":
        await dispatchToSymphony(issue);
        break;
      case "review":
        await dispatchToOpenClaw(issue, route.agent_adapter);
        break;
      case "design":
        await dispatchToAgenticEngineer(issue);
        break;
    }

    await db.update(routingRecords)
      .set({ execution_status: "dispatched", updated_at: new Date().toISOString() })
      .where(eq(routingRecords.id, record[0].id));

  } catch (err) {
    await db.update(routingRecords)
      .set({ execution_status: "failed", route_reason: `${route.reason} | dispatch error: ${err}` })
      .where(eq(routingRecords.id, record[0].id));
  }

  return c.json({
    ok: true,
    routed_to: route.decision,
    reason: route.reason,
  });
});

function decisionToSystem(decision: RouteDecision): string {
  switch (decision) {
    case "auto": return "symphony";
    case "review": return "openclaw";
    case "design": return "agentic-engineer";
  }
}
```

#### 1.6 三个分发器实现

```typescript
// src/dispatchers/symphony.ts
// Symphony 通过 Linear 标签 + 状态来拾取工单，不需要直接 API 调用。
// 分发器只需确保工单有正确标签，然后触发 Symphony 立即轮询。

export async function dispatchToSymphony(issue: LinearIssue) {
  // 1. 确保工单有 route:auto 标签（Symphony 的 labels_include 过滤依赖此标签）
  await ensureLinearLabel(issue.id, "route:auto");

  // 2. 确保工单处于 Symphony 的 active_states（如 Todo）
  // 通常工单创建时就是 Todo，无需额外操作

  // 3. 触发 Symphony 立即轮询（如果 Symphony 已启动且开启了 HTTP 端口）
  const symphonyUrl = process.env.SYMPHONY_API_URL; // e.g. http://localhost:4000
  if (symphonyUrl) {
    await fetch(`${symphonyUrl}/api/v1/refresh`, { method: "POST" });
  }
  // 如果 Symphony 未开启 HTTP，它会在下一个轮询周期（默认 5s）自动拾取
}
```

```typescript
// src/dispatchers/openclaw.ts
// 通过 REST API 在 OpenClaw 中创建任务。

export async function dispatchToOpenClaw(
  issue: LinearIssue,
  agentAdapter?: string
) {
  const openclawUrl = process.env.OPENCLAW_API_URL; // e.g. http://localhost:3000

  const response = await fetch(`${openclawUrl}/api/tasks`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      title: `[${issue.identifier}] ${issue.title}`,
      description: issue.description,
      prompt: buildOpenClawPrompt(issue),
      linear_id: issue.id,
      linear_identifier: issue.identifier,
      linear_url: issue.url,
      agent_adapter: agentAdapter,
    }),
  });

  if (!response.ok) {
    throw new Error(`OpenClaw API error: ${response.status} ${await response.text()}`);
  }

  return response.json();
}

function buildOpenClawPrompt(issue: LinearIssue): string {
  return `You are working on Linear issue ${issue.identifier}.

Title: ${issue.title}

Description:
${issue.description || "No description provided."}

Requirements:
1. Work only in the provided repository copy.
2. Run tests before committing.
3. Create a PR when done.
4. Reference ${issue.identifier} in the PR title.`;
}
```

```typescript
// src/dispatchers/agentic.ts
// Agentic Engineer 是方法论框架，无运行时 API。
// 分发器创建一个规划任务目录，然后通过 Claude Code 触发规划流程。

import { execSync } from "child_process";
import { mkdirSync, writeFileSync } from "fs";
import { join } from "path";

const PLANNING_ROOT = process.env.PLANNING_ROOT || "~/planning-tasks";

export async function dispatchToAgenticEngineer(issue: LinearIssue) {
  const taskDir = join(PLANNING_ROOT, issue.identifier.toLowerCase());
  mkdirSync(taskDir, { recursive: true });

  // 1. 写入灵感文档（Step 1 of Agentic Engineer）
  writeFileSync(join(taskDir, "inspiration.md"), `# ${issue.title}

## Source
- Linear: ${issue.identifier}
- URL: ${issue.url}
- Priority: ${issue.priority}

## Raw Requirements
${issue.description || "No description provided."}

## Labels
${issue.labels.join(", ")}
`);

  // 2. 写入自动化脚本，用 Claude Code 驱动规划流程
  writeFileSync(join(taskDir, "run_planning.sh"), `#!/bin/bash
set -e
cd "${taskDir}"

# Step 1: 灵感已捕获 (inspiration.md)

# Step 2: 生成 SDD
claude -p "You are using the spec-driven-dev methodology. \\
  Read inspiration.md and generate a structured design document. \\
  Output to sdd_v1.md following the SDD template." < inspiration.md

# Step 3: 压力测试
claude -p "Stress test the spec in sdd_v1.md. \\
  Output a scorecard to scorecard_v1.json." < sdd_v1.md

# Step 4: 检查收敛
python3 ${process.env.AGENTIC_TOOLS_PATH}/scorecard_parser.py scorecard_v1.json --format json > convergence.json

# Step 5: 如果收敛，生成 spec_final.md 并回调 Meta-Router
if python3 -c "import json; c=json.load(open('convergence.json')); exit(0 if c.get('converged') else 1)"; then
  cp sdd_v1.md spec_final.md
  # 回调 Meta-Router：规划完成，可以拆解为子工单
  curl -X POST ${process.env.META_ROUTER_URL}/api/planning-complete \\
    -H "Content-Type: application/json" \\
    -d '{"linear_identifier": "${issue.identifier}", "spec_path": "${taskDir}/spec_final.md"}'
else
  echo "未收敛，需要人工介入迭代"
  # 更新 Linear 工单状态，通知人工
  curl -X POST ${process.env.META_ROUTER_URL}/api/planning-needs-iteration \\
    -d '{"linear_identifier": "${issue.identifier}"}'
fi
`);

  // 3. 后台启动规划流程
  execSync(`chmod +x "${join(taskDir, "run_planning.sh")}"`);
  execSync(`nohup bash "${join(taskDir, "run_planning.sh")}" > "${join(taskDir, "planning.log")}" 2>&1 &`);
}
```

---

### Phase 2: 状态同步（第 3-4 周）

#### 2.1 状态同步服务

三个子系统的状态需要**双向同步回 Linear**，让工程师在 Linear 看板上看到统一视图。

```typescript
// src/sync/state-sync.ts

// 每 30 秒执行一次状态同步

export async function syncAllSystems() {
  const records = await db.select()
    .from(routingRecords)
    .where(notIn(routingRecords.execution_status, ["completed", "failed"]));

  for (const record of records) {
    switch (record.target_system) {
      case "symphony":
        await syncSymphonyState(record);
        break;
      case "openclaw":
        await syncOpenClawState(record);
        break;
      case "agentic-engineer":
        await syncAgenticState(record);
        break;
    }
  }
}
```

#### 2.2 Symphony 状态同步

```typescript
// Symphony 有 JSON API，直接查询

async function syncSymphonyState(record: RoutingRecord) {
  const symphonyUrl = process.env.SYMPHONY_API_URL;
  if (!symphonyUrl) return;

  // 查询 Symphony 中该 issue 的状态
  const res = await fetch(`${symphonyUrl}/api/v1/${record.linear_identifier}`);

  if (res.status === 404) {
    // Symphony 还未拾取，可能在队列中
    return;
  }

  const data = await res.json();

  // 映射 Symphony 状态到统一状态
  let newStatus: ExecutionStatus;
  if (data.status === "running") {
    newStatus = data.running?.state === "human_review" ? "human_review" : "in_progress";
  } else if (data.status === "retrying") {
    newStatus = "in_progress";
  } else {
    return;
  }

  // 更新路由记录
  if (record.execution_status !== newStatus) {
    await db.update(routingRecords)
      .set({
        execution_status: newStatus,
        updated_at: new Date().toISOString(),
        metadata: JSON.stringify({
          ...JSON.parse(record.metadata || "{}"),
          symphony_turn_count: data.running?.turn_count,
          symphony_tokens: data.running?.tokens,
        }),
      })
      .where(eq(routingRecords.id, record.id));
  }
}
```

#### 2.3 OpenClaw 状态同步

```typescript
async function syncOpenClawState(record: RoutingRecord) {
  const openclawUrl = process.env.OPENCLAW_API_URL;
  const meta = JSON.parse(record.metadata || "{}");
  if (!meta.openclaw_task_id) return;

  const res = await fetch(`${openclawUrl}/api/tasks/${meta.openclaw_task_id}`);
  if (!res.ok) return;

  const task = await res.json();

  // 映射 OpenClaw 状态
  const statusMap: Record<string, ExecutionStatus> = {
    "queued": "dispatched",
    "in_progress": "in_progress",
    "awaiting_review": "awaiting_review",
    "merged": "completed",
    "rejected": "failed",
    "orphaned": "failed",
  };

  const newStatus = statusMap[task.status] || record.execution_status;

  if (record.execution_status !== newStatus) {
    await db.update(routingRecords)
      .set({ execution_status: newStatus, updated_at: new Date().toISOString() })
      .where(eq(routingRecords.id, record.id));

    // 如果 OpenClaw 完成了，同步回 Linear
    if (newStatus === "completed") {
      await updateLinearIssueState(record.linear_id, "Done");
    }
  }
}
```

#### 2.4 回写 Linear

```typescript
// src/sync/linear-updater.ts

import { linearClient } from "./linear-client";

// 在 Linear 工单上添加路由信息评论
export async function postRoutingComment(
  issueId: string,
  decision: RouteDecision,
  reason: string,
  targetSystem: string
) {
  const body = `## Meta-Router 路由决策

| 项目 | 值 |
|------|------|
| 决策 | \`${decision}\` |
| 目标系统 | \`${targetSystem}\` |
| 原因 | ${reason} |
| 时间 | ${new Date().toISOString()} |`;

  await linearClient.commentCreate({ issueId, body });
}

// 更新 Linear 工单状态
export async function updateLinearIssueState(issueId: string, stateName: string) {
  // 先查找 state ID
  const states = await linearClient.workflowStates();
  const target = states.find(s => s.name === stateName);
  if (target) {
    await linearClient.issueUpdate(issueId, { stateId: target.id });
  }
}
```

---

### Phase 3: 规划完成回调 + 工单拆解（第 4-5 周）

当 Agentic Engineer 完成规划（`spec_final.md` 就绪），Meta-Router 自动拆解为子工单。

#### 3.1 规划完成回调端点

```typescript
// src/webhook/planning-callback.ts

app.post("/api/planning-complete", async (c) => {
  const { linear_identifier, spec_path } = await c.req.json();

  // 1. 读取 spec_final.md
  const spec = readFileSync(spec_path, "utf-8");

  // 2. 用 Claude 拆解为子任务
  const subtasks = await decomposeSpecToTasks(spec);

  // 3. 在 Linear 中创建子工单
  const parentIssue = await getLinearIssueByIdentifier(linear_identifier);

  for (const task of subtasks) {
    const childIssue = await linearClient.issueCreate({
      title: task.title,
      description: task.description + `\n\n---\n_Parent: ${linear_identifier}_\n_Source: spec_final.md_`,
      projectId: parentIssue.projectId,
      // 根据子任务特征决定路由标签
      labelIds: [task.isSimple ? routeAutoLabelId : routeReviewLabelId],
      priority: parentIssue.priority,
    });

    // 建立父子关系
    await linearClient.issueRelationCreate({
      issueId: childIssue.id,
      relatedIssueId: parentIssue.id,
      type: "blocks",  // 子任务被父任务阻塞
    });
  }

  // 4. 更新父工单状态
  await linearClient.commentCreate({
    issueId: parentIssue.id,
    body: `## 规划完成\n\n已拆解为 ${subtasks.length} 个子工单。\n\nSpec: \`${spec_path}\``,
  });

  // 5. 更新路由记录
  await db.update(routingRecords)
    .set({ execution_status: "plan_ready" })
    .where(eq(routingRecords.linear_identifier, linear_identifier));

  return c.json({ ok: true, subtasks_created: subtasks.length });
});

async function decomposeSpecToTasks(spec: string): Promise<SubTask[]> {
  // 调用 Claude API 拆解规范文档为可执行的子任务
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 4096,
    messages: [{
      role: "user",
      content: `将以下设计规范拆解为独立的、可执行的开发任务。

每个任务应包含:
- title: 简洁的任务标题
- description: 包含验收标准的任务描述
- isSimple: 是否为简单任务（true/false）

以 JSON 数组格式输出。

规范文档:
${spec}`
    }],
  });

  return JSON.parse(response.content[0].text);
}
```

---

### Phase 4: 统一监控面板（第 5-6 周）

#### 4.1 统一查询 API

```typescript
// src/dashboard/api.ts

app.get("/api/dashboard", async (c) => {
  // 1. 查询所有路由记录
  const records = await db.select().from(routingRecords).orderBy(desc(routingRecords.updated_at));

  // 2. 聚合统计
  const stats = {
    total: records.length,
    by_system: {
      symphony: records.filter(r => r.target_system === "symphony").length,
      openclaw: records.filter(r => r.target_system === "openclaw").length,
      agentic: records.filter(r => r.target_system === "agentic-engineer").length,
    },
    by_status: groupBy(records, "execution_status"),
    active: records.filter(r => !["completed", "failed"].includes(r.execution_status)),
  };

  // 3. 实时查询各子系统状态
  let symphonyState = null;
  if (process.env.SYMPHONY_API_URL) {
    const res = await fetch(`${process.env.SYMPHONY_API_URL}/api/v1/state`);
    if (res.ok) symphonyState = await res.json();
  }

  let openclawTasks = null;
  if (process.env.OPENCLAW_API_URL) {
    const res = await fetch(`${process.env.OPENCLAW_API_URL}/api/tasks`);
    if (res.ok) openclawTasks = await res.json();
  }

  return c.json({
    routing: { stats, records },
    symphony: symphonyState,
    openclaw: openclawTasks,
  });
});
```

#### 4.2 可选：嵌入 OpenClaw UI

在 OpenClaw 的 Next.js 前端中添加一个 Meta-Router 视图页面，嵌入统一监控数据。

```typescript
// OpenClaw: src/app/router/page.tsx

export default async function RouterDashboard() {
  const res = await fetch(`${process.env.META_ROUTER_URL}/api/dashboard`);
  const data = await res.json();

  return (
    <div>
      <h1>Meta-Router Dashboard</h1>

      {/* 统计概览 */}
      <div className="grid grid-cols-3 gap-4">
        <StatCard title="Symphony (全自动)" count={data.routing.stats.by_system.symphony} />
        <StatCard title="OpenClaw (人工审批)" count={data.routing.stats.by_system.openclaw} />
        <StatCard title="Agentic Engineer (规划)" count={data.routing.stats.by_system.agentic} />
      </div>

      {/* Symphony 实时状态 */}
      {data.symphony && (
        <SymphonyPanel
          running={data.symphony.running}
          retrying={data.symphony.retrying}
          tokens={data.symphony.codex_totals}
        />
      )}

      {/* 路由记录表 */}
      <RoutingTable records={data.routing.records} />
    </div>
  );
}
```

---

## 3. 部署拓扑

```
┌─────────────────── 单台服务器 (推荐起步) ───────────────────┐
│                                                              │
│  ┌──────────────────┐  Port 4000                            │
│  │ Symphony         │  (Elixir escript)                     │
│  │ WORKFLOW.md:     │                                       │
│  │   labels_include:│                                       │
│  │     - route:auto │                                       │
│  └──────────────────┘                                       │
│                                                              │
│  ┌──────────────────┐  Port 3000                            │
│  │ OpenClaw         │  (Next.js + Worker)                   │
│  │ + REST API 扩展   │                                       │
│  └──────────────────┘                                       │
│                                                              │
│  ┌──────────────────┐  Port 3100                            │
│  │ Meta-Router      │  (Hono + SQLite)                      │
│  │ ← Linear Webhook │                                       │
│  └──────────────────┘                                       │
│                                                              │
│  ┌──────────────────┐  ~/planning-tasks/                    │
│  │ Agentic Engineer │  (Python tools + Claude Code)         │
│  │ (无常驻进程)       │                                       │
│  └──────────────────┘                                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘

外部依赖:
  - Linear API (Webhook + GraphQL)
  - Codex CLI (Symphony + OpenClaw)
  - Claude CLI (OpenClaw + Agentic Engineer)
  - ngrok / Cloudflare Tunnel (Linear Webhook 回调)
```

## 4. 环境变量清单

```bash
# .env — Meta-Router

# Linear
LINEAR_API_KEY=lin_api_xxxxx
LINEAR_WEBHOOK_SECRET=whsec_xxxxx           # 验签用
LINEAR_PROJECT_ID=xxxxxxxx                   # 目标项目 ID

# 路由标签 ID（从 Linear API 获取）
LINEAR_LABEL_ROUTE_AUTO=label_id_1
LINEAR_LABEL_ROUTE_REVIEW=label_id_2
LINEAR_LABEL_ROUTE_DESIGN=label_id_3

# 子系统连接
SYMPHONY_API_URL=http://localhost:4000       # Symphony JSON API
OPENCLAW_API_URL=http://localhost:3000       # OpenClaw REST API
META_ROUTER_URL=http://localhost:3100        # 自身（回调用）

# Agentic Engineer
PLANNING_ROOT=~/planning-tasks
AGENTIC_TOOLS_PATH=~/agentic-engineer/tools  # scorecard_parser.py 等

# Meta-Router 自身
META_ROUTER_PORT=3100
META_ROUTER_DB_PATH=.data/meta-router.db
SYNC_INTERVAL_MS=30000                       # 状态同步间隔
```

## 5. 实施时间线

```
第 1 周    Phase 0: 准备工作
           ├─ Linear 标签体系搭建
           ├─ Symphony 添加 labels_include 过滤 (改 3 个文件)
           └─ OpenClaw 添加 REST API (新增 2 个文件)

第 2-3 周  Phase 1: Meta-Router 核心
           ├─ 项目脚手架 (Hono + Drizzle + SQLite)
           ├─ Linear Webhook handler
           ├─ 路由决策引擎
           ├─ 三个分发器
           └─ 基本冒烟测试

第 3-4 周  Phase 2: 状态同步
           ├─ Symphony 状态同步 (JSON API 查询)
           ├─ OpenClaw 状态同步 (REST API 查询)
           ├─ Linear 回写 (评论 + 状态更新)
           └─ 同步定时器

第 4-5 周  Phase 3: 规划回调
           ├─ Agentic Engineer 规划完成回调
           ├─ Claude API 工单拆解
           └─ 子工单自动创建 + 路由

第 5-6 周  Phase 4: 统一监控
           ├─ Dashboard API
           ├─ OpenClaw UI 嵌入
           └─ 端到端集成测试

第 7 周    试运行 + 调优
           ├─ 小批量工单验证各通道
           ├─ 路由规则调优
           └─ 异常处理完善
```

## 6. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| Linear Webhook 不可达 | 工单无法路由 | 备用：Meta-Router 主动轮询 Linear（类似 Symphony） |
| OpenClaw 无 REST API | 无法程序化创建任务 | Phase 0 中实现；备选：直接写 SQLite |
| Agentic Engineer 规划不收敛 | 工单卡在规划阶段 | 设置最大迭代次数（3次），超时降级为 route:review |
| Symphony 未开启 HTTP 端口 | 无法触发即时轮询 | 依赖 Symphony 自身轮询周期（5s），可接受 |
| 状态同步延迟 | Linear 看板与实际不一致 | 同步间隔设为 30s；关键状态变更立即同步 |
| 跨系统工单重复处理 | 同一工单被多个系统执行 | 路由记录做唯一约束；Symphony labels_include 隔离 |

## 7. 最小可行产品（MVP）

如果要在 **3 天内** 跑通一个最小原型，只需要：

1. **Day 1**: 写一个 100 行的 Python/TS 脚本，轮询 Linear，按标签分流
2. **Day 2**: Symphony 配置 `labels_include: ["route:auto"]`；OpenClaw 手动创建 `route:review` 的任务
3. **Day 3**: 端到端验证：创建 3 个工单（分别打 3 种标签），观察它们流向不同系统

MVP 不需要 Webhook、不需要状态同步、不需要统一面板 — 只要证明**路由分流可行**。
