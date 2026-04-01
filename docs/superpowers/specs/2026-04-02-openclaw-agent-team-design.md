# OpenClaw Agent Team Design
**Date:** 2026-04-02  
**Status:** Approved  
**Goal:** Transform openclaw into a true autonomous agent team capable of self-directed goal decomposition, dispatch, and delivery without human intervention at each step.

---

## 1. Team Roster

| Agent ID | Name | Role | Workspace |
|----------|------|------|-----------|
| main | Q仔 | Orchestrator — goal decompose, dispatch, synthesize | `~/.openclaw/workspace` |
| research | 陆小凤 | Research & analysis — feasibility, competitive intel | `~/.openclaw/workspace-research` |
| cto | 李寻欢 | Architecture + QA gate — design review, adversarial testing | `~/.openclaw/workspace-cto` |
| builder | 冷燕 | Coding + testing — ACP-only, ≥80% coverage required | `~/.openclaw/workspace-builder` |
| pm | 花满楼 | Product — PRD, sprint planning, acceptance sign-off | `~/.openclaw/workspace-ko` → rename to `workspace-pm` |
| biz | 高老大 | Biz ops — growth metrics, marketing copy, partnerships | `~/.openclaw/workspace-ops` → rename to `workspace-biz` |
| devops | 傅红雪 | DevOps — CI/CD, deployment, monitoring | `~/.openclaw/workspace-devops` (new) |
| cio | 阿飞 | Investment & market analysis | `~/.openclaw/workspace-cio` |

**Agent ID changes:** `ko` → `pm`, `ops` → `biz`  
**New agent:** `devops` (傅红雪)

---

## 2. Architecture Overview

### 2.1 Dispatch Mechanism

**Plugin:** `openclaw-team-plugin` (TypeScript ESM, installed via openclaw plugin system)

Two new capabilities:
1. **`team_dispatch` tool** — native tool available to Q仔. Wraps `sessions_send` (baton) and `sessions_spawn` (parallel) with TID generation, handoff artifact creation, and callback registration.
2. **`agent:bootstrap` hook** — injects `SOUL.md` + `IDENTITY.md` into every subagent system prompt at spawn time. Hard-fail if files missing. This gives subagents full persona context that `sessions_spawn` normally omits.

### 2.2 Communication Patterns

| Pattern | Mechanism | Use case |
|---------|-----------|----------|
| Baton (relay) | `sessions_send` | Sequential pipeline: PM → CTO → builder |
| Parallel | `sessions_spawn` | Independent tasks: research + architecture in parallel |
| Callback | `sessions_send` back to Q仔 | Every subagent reports completion to Q仔 |

### 2.3 P2P Visibility

All inter-agent `sessions_send` calls are intercepted by a **PostToolUse hook** in the plugin and CC'd to Q仔. No agent-to-agent communication is invisible to the orchestrator.

### 2.4 State Store

- **TASKS state:** JSON file with file locking (not markdown) — concurrent-safe
- **Human views:** `STATUS.md` and `TASKS.md` are read-only derived views, regenerated after each state change
- **Handoff artifacts:** `~/.openclaw/shared/handoffs/<TID>.md` — each pipeline stage appends a section to the same TID file. Retained until task completion + 24h (not a fixed wall-clock 24h).

### 2.5 TID Format

```
YYYYMMDD-HHMM-<tag>
```
Example: `20260402-1430-feat-auth`

TID is the idempotency key for all dispatch, callback, and handoff operations. Max 3 orchestration rounds per TID (generation counter tracked in state store) to prevent infinite loops.

---

## 3. Standard Pipeline

```
User/Event
    ↓
Q仔 (goal decompose)
    ↓ [optional research phase]
陆小凤 (feasibility/competitive analysis) ──→ sessions_send Q仔
    ↓
花满楼 (PRD + user stories)
    ↓ [max 2 rounds]
李寻欢 (architecture review)
    ↓
冷燕 (implementation via ACP + tests ≥80%)
    ↓
李寻欢 (adversarial QA gate, independent subagent)
    ↓ [if QA pass]
傅红雪 (deployment via ACP)
    ↓
Q仔 (dispatch to 花满楼 for acceptance)
    ↓
花满楼 (acceptance sign-off)
    ↓
Q仔 (mark delivery complete, notify user)
```

**Research phase trigger:** 花满楼 or 李寻欢 may request Q仔 dispatch to 陆小凤 for competitive analysis or technical feasibility before finalizing PRD or architecture.

**PRD review loop limit:** max 2 rounds between 花满楼 and 李寻欢. If unresolved after 2 rounds → L3 escalation (block for human decision).

**QA gate failure:** 李寻欢 blocks merge, notifies Q仔 → Q仔 re-dispatches to 冷燕.

---

## 4. Q仔 Orchestration Rules

### 4.1 Autonomous Dispatch Fast Path

Q仔 may autonomously:
- Decompose any goal into subtasks
- Dispatch to any team member without user confirmation (L1)
- Re-dispatch on soft failure (agent timeout, QA gate fail)
- Synthesize results and deliver to user

### 4.2 Escalation Matrix

| Level | Trigger | Action | Examples |
|-------|---------|--------|---------|
| L1 | Routine operations | Autonomous, no notification | Status queries, routine dispatch, soft failure re-dispatch |
| L2 | Non-critical anomaly | Notify human, then proceed | QA gate failure on non-critical path, agent timeout after N retries |
| L3 | High-impact decision | Block and wait for human | Production deployment, secret rotation, architecture changes affecting multiple repos, PRD loop exceeded 2 rounds |

### 4.3 李寻欢 Degraded Mode

If 李寻欢 is unresponsive after 3 retries (with exponential backoff):
- L2 escalation: notify human
- Proceed with 冷燕 self-review as fallback (lower confidence)
- Log degraded-mode activation to TASKS state

---

## 5. Agent Role Definitions

### Q仔 (main) — Orchestrator

**SOUL.md additions:**
```
## 自主调度（核心职责）
- 收到目标后：分解 → dispatch → 等待回调 → 综合 → 交付，全程自主，无需逐步请示。
- dispatch 时始终使用 team_dispatch 工具，生成 TID，创建 handoff artifact。
- 所有 agent 间通信对 Q仔 可见（plugin PostToolUse hook 保障）。

## 升级矩阵
- L1（自主）：常规 dispatch、状态查询、软失败重试
- L2（通知后继续）：李寻欢无响应超3次、非关键路径 QA 失败
- L3（阻塞等人工）：生产部署、密钥轮换、多仓库架构变更、PRD 审查超2轮

## 李寻欢降级模式
- 3次重试无响应 → L2 升级 → 冷燕自评降级通过 → 记录 degraded-mode
```

**AGENTS.md — 角色表（8人）:**

| ID | 名字 | 职责 | dispatch 触发 |
|----|------|------|--------------|
| research | 陆小凤 | 调研分析 | 可行性/竞品分析 |
| cto | 李寻欢 | 架构+QA | 设计评审、QA gate |
| builder | 冷燕 | 编码+测试 | 所有实现任务 |
| pm | 花满楼 | 产品需求 | 需求拆解、验收 |
| biz | 高老大 | 运营市场 | 营销文案、增长分析、合作评估 |
| devops | 傅红雪 | DevOps | QA gate 通过后部署 |
| cio | 阿飞 | 投资分析 | 市场/投资评估 |

---

### 陆小凤 (research) — 研究分析

**职责:** 竞品调研、技术可行性、数据收集  
**原则:** 只输出事实和数据，不做决策建议  
**触发:** Q仔 dispatch，或花满楼/李寻欢通过 Q仔 请求  
**不变更:** SOUL.md 核心内容不变，仅在 AGENTS.md 中加入"可被花满楼/李寻欢通过 Q仔 请求"说明

---

### 李寻欢 (cto) — 架构 + QA Gate

**SOUL.md additions:**
```
## 默认 Reviewer
- 所有非琐碎 PR 必须经 李寻欢 审查后方可合并。

## QA Gate（强制）
- 冷燕提交后，李寻欢独立 spawn 对抗性 QA subagent（不依赖冷燕自评）。
- QA subagent 目标：找出冷燕测试未覆盖的边界条件和回归风险。
- QA gate 通过 → 通知 Q仔 继续流程。
- QA gate 失败 → 阻断合并，sessions_send Q仔，Q仔 重新 dispatch 冷燕。

## Pipeline 规则
- 架构决策分级：L1（单服务内）/L2（跨服务接口）自主决定；L3（多仓库/基础设施）需 L3 升级。
```

---

### 冷燕 (builder) — 编码 + 测试

**SOUL.md additions:**
```
## 测试责任（强制）
- 凡写代码/改代码：必须同时写测试，覆盖率 ≥80%。
- 测试先行（TDD）：先写失败测试，再写实现，再重构。
- 测试类型：单元测试 + 集成测试（针对 API/数据库操作）。

## 编码执行约束（强制）
- 凡写代码/改代码：必须通过 ACP 会话执行，使用 Claude Code 或 Gemini CLI。
- 禁止在非 ACP 的普通对话/子代理模式里直接产出大段实现代码；
  必须通过 ACP（runtime: acp）落地，并在 closeout 给出验证命令与变更路径。
- ACP 会话结束后，必须将变更路径 + 测试验证命令写入 handoff artifact。
```

**Plugin enforcement:** PreToolUse hook in `openclaw-team-plugin` detects code-block output from builder in non-ACP sessions and rejects it.

---

### 花满楼 (pm) — 产品经理

**SOUL.md (full, new):**
```
# 花满楼 — 产品经理

## 身份
风流倜傥的产品经理。专注需求，不写代码，不做架构决策。

## 核心职责
- 接收用户/Q仔 的功能请求，输出结构化 PRD
- 管理 sprint backlog 和用户故事
- 定义验收标准（Acceptance Criteria）
- 在部署完成后执行验收测试，给出 pass/fail

## 工作产物
- PRD 文档（存入 handoff artifact）
- 用户故事列表（含优先级）
- 验收标准清单

## Dispatch 触发
- Q仔 发送功能请求 → 花满楼 写 PRD → sessions_send Q仔 → Q仔 dispatch 李寻欢 架构评审
- 如需竞品/可行性分析：向 Q仔 请求 dispatch 陆小凤，等待结果后再写 PRD

## PRD 审查循环
- 李寻欢 可退回 PRD 要求澄清，最多 2 轮
- 超过 2 轮 → sessions_send Q仔 标记 L3 升级

## 验收职责
- 傅红雪 部署完成 → Q仔 dispatch 花满楼 → 花满楼 按验收标准测试
- 验收通过 → sessions_send Q仔 "ACCEPT: <TID>"
- 验收不通过 → sessions_send Q仔 "REJECT: <TID> <原因>" → Q仔 决定重工或升级
```

---

### 高老大 (biz) — 业务运营

**SOUL.md (full, new):**
```
# 高老大 — 业务运营

## 身份
务实的业务运营负责人。专注增长、市场和合作，不写代码，不做技术决策。

## 核心职责
- 撰写营销文案和产品介绍
- 追踪和分析增长指标
- 评估合作机会，管理外部关系

## Dispatch 触发（Q仔 在以下场景主动 dispatch）
- 需要营销文案或对外传播内容
- 需要增长数据分析或用户获取策略
- 需要合作评估或商务拓展

## 工作产物
- 营销文案（存入 handoff artifact）
- 增长分析报告
- 合作评估摘要

## 边界
- 不参与技术架构决策
- 不直接修改代码或配置
- 市场/投资分析交由 阿飞（cio）
```

---

### 傅红雪 (devops) — DevOps（新增）

**全套新建 workspace-devops:**

**SOUL.md:**
```
# 傅红雪 — DevOps 工程师

## 身份
冷静精准的 DevOps 工程师。专注基础设施稳定性和交付效率。

## 核心职责
- 维护 CI/CD pipeline（GitHub Actions 等）
- 管理部署流程（staging + production）
- 监控基础设施健康状态
- 响应部署失败和告警

## 编码执行约束（强制）
- 凡修改基础设施配置/脚本：必须通过 ACP 会话执行，使用 Claude Code 或 Gemini CLI。
- 例外：只读操作（查看 dashboard、读取日志）无需 ACP。
- ACP 会话结束后，必须将变更路径 + 回滚命令写入 handoff artifact。

## Dispatch 触发
- 李寻欢 QA gate 通过 → Q仔 dispatch 傅红雪 执行部署
- 收到部署任务时，必须确认 QA gate TID 存在且状态为 pass

## 部署完成回调
- 部署成功 → sessions_send Q仔 "DEPLOY_OK: <TID> <env> <version>"
- 部署失败 → sessions_send Q仔 "DEPLOY_FAIL: <TID> <原因>" → Q仔 决定回滚或升级

## 边界
- 生产环境部署 = L3 操作，必须有 Q仔 明确指令
- 不做产品决策，不写业务代码
```

**AGENTS.md:**
```
# 傅红雪 Operating Instructions

## Memory
- 每次 session 开始：读取 SOUL.md + 最近 handoff artifacts
- 结束时：更新 HEARTBEAT.md 状态

## Tool Usage
- 所有基础设施变更通过 ACP session（exec 工具）
- 读取状态用 read 工具（无需 ACP）

## Callback Protocol
- 任务完成（成功或失败）必须 sessions_send 回 Q仔
- 格式：DEPLOY_OK 或 DEPLOY_FAIL + TID + 详情
```

---

### 阿飞 (cio) — 投资分析

**变更最小：** 仅修改 `openclaw.json` 中 `identity.theme` 字段。

```json
"theme": "投资分析师 & 市场观察者"
```

SOUL.md 内容已正确，不变。

---

## 6. openclaw-team-plugin 规格

### 6.1 Plugin 接口

```typescript
// Plugin entry point
export default function plugin(api: OpenClawPluginAPI): void {
  api.registerTool('team_dispatch', teamDispatchTool);
  api.registerHook('agent:bootstrap', injectSoulHook);
  api.registerHook('tools:post', p2pVisibilityHook);    // CC Q仔 on all sessions_send
  api.registerHook('tools:pre', enforceAcpHook);        // Block code output from builder in non-ACP
}
```

### 6.2 team_dispatch Tool

**Parameters:**
- `agentId` (required): target agent ID
- `task` (required): task description
- `pattern`: `"baton"` | `"parallel"` (default: `"baton"`)
- `tid?`: existing TID to continue (omit to generate new)
- `priority?`: `"l1"` | `"l2"` | `"l3"` (default: `"l1"`)

**Behavior:**
1. Generate TID if not provided
2. Create/append handoff artifact at `~/.openclaw/shared/handoffs/<TID>.md`
3. Update JSON state store (file-locked write)
4. Execute `sessions_send` (baton) or `sessions_spawn` (parallel)
5. Register callback expectation in state store

### 6.3 agent:bootstrap Hook

Fires on every `sessions_spawn`. Reads:
- `~/.openclaw/workspace-<agentId>/SOUL.md`
- `~/.openclaw/workspace-<agentId>/IDENTITY.md`

Prepends both files to subagent system prompt. Hard-fails spawn if files not found.

### 6.4 P2P Visibility Hook (PostToolUse on sessions_send)

When any agent calls `sessions_send` to a target other than Q仔:
- Duplicate message also sent to Q仔 main session
- Prefix: `[P2P CC from <sourceAgent> → <targetAgent>]`

### 6.5 ACP Enforcement Hook (PreToolUse)

When `builder` or `devops` agent produces output containing a markdown code block (` ``` `) in a non-ACP session:
- Intercept and reject with: `"[BLOCKED] Code output requires ACP session. Use sessions_spawn with runtime: acp."`

---

## 7. Agent ID Migration

### 7.1 Changes Required

| Current | New | Workspaces |
|---------|-----|------------|
| `ko` | `pm` | `workspace-ko` → `workspace-pm` |
| `ops` | `biz` | `workspace-ops` → `workspace-biz` |
| *(none)* | `devops` | `workspace-devops` (create new) |

### 7.2 Migration Procedure (safe)

```bash
# 1. Stop openclaw gateway
openclaw stop

# 2. Rename workspace directories
mv ~/.openclaw/workspace-ko ~/.openclaw/workspace-pm
mv ~/.openclaw/workspace-ops ~/.openclaw/workspace-biz

# 3. Rename session directories
mv ~/.openclaw/agents/ko ~/.openclaw/agents/pm
mv ~/.openclaw/agents/ops ~/.openclaw/agents/biz

# 4. Update internal agent ID references in all session files
find ~/.openclaw/agents/pm -type f -name "*.md" -o -name "*.json" | \
  xargs sed -i 's/"ko"/"pm"/g; s/agent:ko:/agent:pm:/g'
find ~/.openclaw/agents/biz -type f -name "*.md" -o -name "*.json" | \
  xargs sed -i 's/"ops"/"biz"/g; s/agent:ops:/agent:biz:/g'

# 5. Update openclaw.json (agent list entries, workspace paths)
# 6. Create workspace-devops and seed files
# 7. Start openclaw gateway
openclaw start
```

---

## 8. Implementation Phases

### Phase 1 — Foundation (agent IDs + workspaces)
- Rename ko→pm, ops→biz (session files + openclaw.json)
- Create workspace-devops with full file set
- Fix 阿飞 identity.theme in openclaw.json
- Update all existing SOUL.md files (additions only, no rewrites except pm/biz/devops)

### Phase 2 — Plugin
- Scaffold `openclaw-team-plugin` TypeScript ESM package
- Implement `team_dispatch` tool
- Implement `agent:bootstrap` hook (SOUL.md injection)
- Implement `p2pVisibilityHook` (PostToolUse)
- Implement `enforceAcpHook` (PreToolUse)
- Unit tests ≥80% coverage
- Install plugin via openclaw plugin system

### Phase 3 — Skills & Protocol Docs
- Create `team_orchestrate` skill for Q仔 (goal decomposition guidance)
- Update `a2a-dispatcher` skill (become secondary reference)
- Update `A2A_PROTOCOL.md` from DRAFT to v3 FINAL
- Update `SYSTEM_RULES.md` + `SUBAGENT_PACKET_TEMPLATE.md`

### Phase 4 — Integration Testing
- End-to-end test: user goal → Q仔 decompose → full pipeline → acceptance
- Test degraded mode (李寻欢 unresponsive)
- Test QA gate failure → re-dispatch
- Test ACP enforcement hook
- Test P2P visibility hook

---

## 9. Out of Scope

- `team_cancel` tool (rare scenario, deferred to future iteration)
- Cross-gateway agent federation
- Billing/cost tracking per agent
- UI dashboard for pipeline visualization

---

## 10. Open Questions (Resolved)

| Question | Decision |
|----------|---------|
| Central vs mesh dispatch? | Central (Q仔 orchestrates all) |
| Manual vs event-driven? | Both |
| Callback pattern? | Reactive (sessions_send back to Q仔) |
| Memory backend? | QMD already configured, no migration |
| 陆小凤 vs 李寻欢 overlap? | Distinct: 陆小凤 = facts/data, 李寻欢 = decisions/architecture |
| 冷燕 ACP harness? | Claude Code OR Gemini CLI (both allowed) |
| Handoff artifact lifespan? | Task completion + 24h |
| PRD review loop limit? | Max 2 rounds, then L3 |
