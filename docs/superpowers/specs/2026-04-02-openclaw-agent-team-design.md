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

All inter-agent `sessions_send` calls are intercepted by a **PostToolUse hook** in the plugin and written to the TID event log (`~/.openclaw/shared/events/<TID>.jsonl`). They are **not** injected inline into Q仔's context (avoids context bloat from multi-round technical discussions). Q仔 reads the event log only when processing a callback (task complete/fail), pulling only the summary it needs.

### 2.4 State Store

- **TASKS state:** JSON file with file locking (not markdown) — concurrent-safe
  - Lock timeout: 5s; exponential backoff retry (max 3 attempts before error)
  - Lock file contains PID + timestamp; on next acquire, check if PID is alive — if dead, treat as stale lock and forcibly clear
  - Gateway startup: scan and clear all stale lock files before accepting requests
- **Human views:** `STATUS.md` and `TASKS.md` are read-only derived views, regenerated after each state change
- **Handoff artifacts:** `~/.openclaw/shared/handoffs/<TID>.md` — each pipeline stage appends a section to the same TID file. Retained until task completion + 24h.
- **Event log:** `~/.openclaw/shared/events/<TID>.jsonl` — P2P inter-agent messages appended here (not injected into Q仔 context). Q仔 queries on demand when processing a callback.
- **GC:** Plugin registers a daily cron (03:00) to delete expired handoff artifacts and event logs. Q仔 HEARTBEAT.md triggers GC check.

### 2.5 TID Format

```
YYYYMMDD-HHMMss-<tag>-<4randchars>
```
Example: `20260402-143012-feat-auth-x7k2`

Second-precision + 4-char random suffix eliminates same-minute collision for parallel spawns. TID is the idempotency key for all dispatch, callback, and handoff operations. Max 3 orchestration rounds per TID (generation counter tracked in state store) to prevent infinite loops.

### 2.6 Git Worktree per Feature

Every new feature or infra change gets its own **git worktree**. This isolates work-in-progress across parallel tasks and prevents branch context pollution in the main checkout.

**Convention (冷燕):**
```bash
# On task receive
git worktree add ~/.openclaw/worktrees/<TID> -b feat/<TID>

# Work entirely inside the worktree (tmux session cd's here)
cd ~/.openclaw/worktrees/<TID>

# When done: commit, push, open PR
git push origin feat/<TID>
gh pr create --title "..." --body "..."

# Worktree removed after PR merge (傅红雪 or Q仔 triggers cleanup)
git worktree remove ~/.openclaw/worktrees/<TID>
```

**Convention (傅红雪):**
```bash
git worktree add ~/.openclaw/worktrees/infra-<TID> -b infra/<TID>
```

**Rules:**
- One worktree per TID. Worktree path is `~/.openclaw/worktrees/<TID>`.
- The tmux ACP session always operates inside the task's worktree, not the main checkout.
- Worktree is created at task start, removed only after PR is merged (not before).
- If a task is abandoned (L3 escalation, cancelled), worktree is archived to `~/.openclaw/worktrees/archived/<TID>` for post-mortem, not deleted.

### 2.7 GitHub as Single Source of Truth

**GitHub is the canonical truth for all durable artifacts.** No artifact is considered "done" until it exists in a GitHub repository.

| Artifact | Owner | GitHub location |
|----------|-------|----------------|
| Business code | 冷燕 | Feature branch → PR → merge |
| Tests | 冷燕 | Same PR as code |
| Infra configs (K8s/TF/CI) | 傅红雪 | Infra repo, dedicated branch → PR |
| PRD documents | 花满楼 | Docs repo (e.g. `docs/prd/<TID>.md`) |
| Architecture decisions | 李寻欢 | ADR in docs repo |

**Rules:**
- 冷燕 must commit + push + open PR before notifying Q仔 of completion. The handoff artifact must include the PR URL and head commit SHA.
- 傅红雪 must commit + push infra configs before requesting 李寻欢 infra review. Review happens on the PR diff, not on local files.
- 花满楼 must commit PRD to the docs repo before dispatching to 李寻欢 for architecture review.
- No "it works locally" handoffs. If it's not in GitHub, it doesn't exist.
- Q仔 treats the PR URL as the authoritative reference for any task's output.

### 2.8 tmux Persistent ACP Sessions

`sessions_spawn` with `runtime: "acp"` spawns a one-shot process. For agents that need multi-step edit-run-verify loops (冷燕, 傅红雪), a **persistent tmux session** is required to maintain file system state, git context, and ACP process across turns.

**Convention:**
- 冷燕: tmux session named `builder-acp`, window `claude-code` or `gemini`
- 傅红雪: tmux session named `devops-acp`, window `claude-code` or `gemini`

**Protocol:**
1. On task receive: check if named tmux session exists (`tmux has-session -t builder-acp`). Create if missing.
2. Send task context to the running ACP process via tmux (`tmux send-keys`).
3. ACP process executes edits, runs tests, commits, pushes.
4. When done: agent reads ACP output from tmux pane, extracts PR URL + commit SHA, writes to handoff artifact, then `sessions_send` callback to Q仔.
5. tmux session stays alive between tasks (do not kill after each task).

**openclaw integration:** `sessions_spawn` with `runtime: "acp"` and `thread: true` + `mode: "session"` maps naturally to this — the session persists and subsequent dispatches route to the same session. The tmux layer lives inside the ACP harness.

### 2.9 Per-Worktree Agent Log

Each git worktree gets a single append-only log file:
```
~/.openclaw/worktrees/<TID>/.agent-log.jsonl
```
Each action appended as one JSON line: `{ ts, agent, action, result, error? }`.

- 冷燕 and 傅红雪 write to this log during ACP execution.
- Q仔 reads `.agent-log.jsonl` when processing escalations — replaces need for human to dig through tmux history.
- No Vector, no Prometheus, no metrics endpoint at this stage. Revisit when simple logs prove insufficient for Q仔 triage.

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
李寻欢 (adversarial QA gate, independent subagent) [max 3 rounds]
    ↓ [if QA pass]
傅红雪 (produce infra configs: K8s YAML / Terraform / CI scripts)
    ↓
李寻欢 (infra config review gate)
    ↓ [if infra review pass]
傅红雪 (execute deployment via ACP)
    ↓
Q仔 (dispatch to 花满楼 for acceptance)
    ↓
花满楼 (acceptance sign-off)
    ↓
Q仔 (mark delivery complete, notify user)
```

**Research phase trigger:** 花满楼 or 李寻欢 may request Q仔 dispatch to 陆小凤 for competitive analysis or technical feasibility before finalizing PRD or architecture.

**PRD review loop limit:** max 2 rounds between 花满楼 and 李寻欢. If unresolved after 2 rounds → L3 escalation (block for human decision).

**QA gate failure:** 李寻欢 blocks merge, notifies Q仔 → Q仔 re-dispatches to 冷燕. **Max 3 QA rounds** per TID (tracked in state store); if 3rd round still fails → L3 escalation (block for human decision).

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

## Harness Review（模型升级时触发）
模型升级后，执行轻量 harness review：
1. 用新模型跑现有 QA 测试集，先关闭 SOUL.md 约束，记录通过率 A。
2. 再开启 SOUL.md 约束，记录通过率 B。
3. 若 A ≈ B（差距 < 2%）：该约束为**候选删除**，标记供人工确认。
4. 若 B > A：该约束**仍有价值**，保留。
5. 连续 2 次 harness review 中未触发的规则 → 候选删除，需明确签字方可移除。
6. 将 review 结果写入 `~/.openclaw/shared/harness-review-<date>.md`，通知用户。
目的：防止 SOUL.md 积累针对旧模型弱点的"cargo cult"规则。

## Harness 改进飞轮（L3 触发）
每次 L3 升级解决后，Q仔 必须执行：
1. 诊断根因：什么约束缺失导致此次 L3？
2. 在责任 Agent 的 SOUL.md 中补充对应规则（1-3 行）。
3. 规则使用 YAML frontmatter 格式（见下）确保可追踪。
目的：每次故障都让系统变强一次。

## SOUL.md 文件结构原则
- 核心 SOUL.md 控制在约 60 行以内（目录，非百科全书）。
- 详细规则移入 `RULES/` 子目录（如 `RULES/qa-gate.md`、`RULES/acp-constraint.md`）。
- Q仔 在 dispatch 时根据任务类型注入相关 RULES/ 文件到子 agent 上下文，不依赖 agent 自行判断加载。
- 每条规则使用 YAML frontmatter 追踪生命周期：
```yaml
---
rules:
  - id: R-001
    text: "凡写代码/改代码：必须在 tmux builder-acp 中通过 ACP 执行"
    added: 2026-04-02
    reason: "设计决策：确保所有实现落地在持久 ACP session，防止上下文丢失"
    status: active
---
```
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
- QA subagent 严苛程度：以"这个 bug 会在凌晨三点叫醒人"的标准审查。
- QA subagent 必须覆盖以下类别，每类明确报告"发现问题"或"已测，未发现"：
  边界条件 / 错误处理 / 输入校验 / 并发安全 / 安全漏洞
- **禁止"整体感觉不错"式通过**；必须有逐类验证记录，方可出具 pass。
- QA gate 通过 → 通知 Q仔 继续流程。
- QA gate 失败 → 阻断合并，sessions_send Q仔，Q仔 重新 dispatch 冷燕。

## Pipeline 规则
- 架构决策分级：L1（单服务内）/L2（跨服务接口）自主决定；L3（多仓库/基础设施）需 L3 升级。

## Application Legibility（代码对 Agent 可读）
- 强制分层架构：Types → Config → Repo → Service → Runtime → UI，只允许向下依赖。
- 层边界违规由 linter 机械执行，CI 直接挂，无需 LLM 判断。
- Linter 错误信息必须嵌入修复指引（不只报"违规"，还说"怎么改"）。
- 架构决策以 ADR 格式写入仓库（`docs/adr/`），不用外部 wiki。层级规则本身也有对应 ADR。

## 基础设施配置评审（强制）
- 傅红雪产出的所有 K8s YAML / Terraform / CI 脚本，必须经 李寻欢 审查后方可执行。
- 审查点：权限配置、资源限额、密钥引用、幂等性、回滚可行性。
- 通过 → sessions_send Q仔 "INFRA_REVIEW_OK: <TID>"
- 拒绝 → sessions_send Q仔 "INFRA_REVIEW_FAIL: <TID> <原因>"，Q仔 重新 dispatch 傅红雪修改。
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
- 每个新功能/修复：先创建 git worktree（`~/.openclaw/worktrees/<TID>`，分支 `feat/<TID>`）。
- 所有编码在 worktree 内进行，不在 main checkout 里工作。
- 必须在 **tmux session `builder-acp`** 中通过 Claude Code 或 Gemini CLI 执行，tmux session cd 到对应 worktree。
- tmux session 常驻，不在任务间销毁；每次新任务 cd 到新 worktree。
- 禁止在非 ACP 的普通对话/子代理模式里直接产出大段实现代码。
- ACP 执行完成后，必须 commit + push + 开 PR，将 PR URL + commit SHA 写入 handoff artifact。
- PR merge 后由 Q仔/傅红雪 触发 worktree 清理。
- **没有 PR = 没有完成。**
```

**Plugin enforcement:** PreToolUse hook in `openclaw-team-plugin` intercepts `exec`/`write`/`edit` tool calls from builder in non-ACP sessions and rejects them.

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
- Q仔 发送功能请求 → 花满楼 写 PRD → commit + push 到 docs repo（`docs/prd/<TID>.md`）→ sessions_send Q仔 附 GitHub URL → Q仔 dispatch 李寻欢 架构评审
- 如需竞品/可行性分析：向 Q仔 请求 dispatch 陆小凤，等待结果后再写 PRD
- **没有提交到 GitHub 的 PRD 不算完成。**

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
- 每个 infra 变更：先创建 git worktree（`~/.openclaw/worktrees/infra-<TID>`，分支 `infra/<TID>`）。
- 所有配置修改在 worktree 内进行。
- 必须在 **tmux session `devops-acp`** 中通过 Claude Code 或 Gemini CLI 执行，tmux session cd 到对应 worktree。
- tmux session 常驻，不在任务间销毁。
- 例外：只读操作（查看 dashboard、读取日志）无需 ACP 和 worktree。
- 配置产出后必须 commit + push 到 infra repo，开 PR，然后 sessions_send Q仔 "INFRA_READY: <TID> <PR_URL>"。
- 李寻欢 对 PR diff 做 infra review（不是本地文件）。
- 收到 INFRA_REVIEW_OK 后，在 tmux session 中执行部署，并将 rollback 命令写入 handoff artifact。

## Dispatch 触发
- 李寻欢 QA gate 通过 → Q仔 dispatch 傅红雪 **先产出**基础设施配置并推送 PR（不执行）
- PR 推送后 → sessions_send Q仔 "INFRA_READY: <TID> <PR_URL>"，等待 李寻欢 infra review
- 收到 INFRA_REVIEW_OK 后 → 执行部署
- 收到部署任务时，必须确认 QA gate TID 存在且状态为 pass

## 部署完成回调
- 部署成功 → sessions_send Q仔 "DEPLOY_OK: <TID> <env> <version>"
- 部署失败 → sessions_send Q仔 "DEPLOY_FAIL: <TID> <原因>" → Q仔 决定回滚或升级

## CI 重试限制（强制）
- CI 失败 → 傅红雪自动修复一次（`max_ci_retries: 2`，可按 repo 覆盖）。
- 第 2 次仍失败 → 直接 L3 升级，不再重试。
- 例外：若修复 diff 超过原始 diff 的 50% 行数，无论剩余重试次数，自动 L3 升级（大改可能引入新问题）。

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
  api.registerHook('agent:bootstrap', injectSoulHook);         // Inject SOUL.md + IDENTITY.md into subagents
  api.registerHook('tools:post', p2pEventLogHook);             // Append P2P messages to event log (not Q仔 context)
  api.registerHook('tools:pre', enforceAcpHook);               // Block exec/write/edit tools for builder/devops in non-ACP
  api.registerCron('0 3 * * *', gcHandoffsAndEvents);          // Daily GC: delete expired handoffs + event logs
}
```

### 6.2 team_dispatch Tool

**Parameters:**
- `agentId` (required): target agent ID
- `task` (required): task description
- `pattern`: `"baton"` | `"parallel"` (default: `"baton"`)
- `tid?`: existing TID to continue (omit to generate new)
- `priority?`: `"l1"` | `"l2"` | `"l3"` (default: `"l1"`)
- `nodeType`: `"agentic"` | `"deterministic"` (default: `"agentic"`)
  - `"deterministic"`: scripted execution, zero LLM calls (e.g., run linter, git push, CI status check)
  - `"agentic"`: full LLM session (e.g., write code, write PRD, architecture review)
- `escalation_trigger?`: condition that promotes a `"deterministic"` node to `"agentic"` (e.g., `{ exit_code: "!= 0" }`, `{ output_lines: "> 50" }`)
- `max_cost_usd?`: budget cap for this dispatch; auto-escalate to L2 when exceeded (prevents runaway agentic loops)

**Behavior:**
1. Generate TID if not provided
2. Create/append handoff artifact at `~/.openclaw/shared/handoffs/<TID>.md`
3. Update JSON state store (file-locked write)
4. If `nodeType: "deterministic"`: run scripted command, check escalation_trigger, promote to agentic if triggered
5. Execute `sessions_send` (baton) or `sessions_spawn` (parallel) for agentic nodes
6. Register callback expectation in state store; enforce `max_cost_usd` budget

### 6.3 agent:bootstrap Hook

Fires on every `sessions_spawn`. Reads:
- `~/.openclaw/workspace-<agentId>/SOUL.md`
- `~/.openclaw/workspace-<agentId>/IDENTITY.md`

Prepends both files to subagent system prompt. Hard-fails spawn if files not found.

### 6.4 P2P Visibility Hook (PostToolUse on sessions_send)

When any agent calls `sessions_send` to a target other than Q仔:
- Append event to `~/.openclaw/shared/events/<TID>.jsonl` (NOT injected into Q仔 context)
- Event format: `{ ts, from, to, summary }` where summary is first 200 chars of message
- Q仔 reads event log only when processing a callback (on-demand, not inline)

### 6.5 ACP Enforcement Hook (PreToolUse on write/exec tools)

When `builder` or `devops` agent calls `exec`, `write`, or `edit` tools in a non-ACP session:
- Intercept and reject with: `"[BLOCKED] File write/exec requires ACP session. Use sessions_spawn with runtime: acp."`
- **Explanatory code blocks in message text are not blocked** — only actual tool invocations that write or execute.

### 6.6 Per-Agent Tool Whitelist

Principle of least privilege — each agent receives only the tools it needs. More tools ≠ better performance (Stripe finding).

| Agent | Allowed Tools | Scope Notes |
|-------|--------------|-------------|
| main (Q仔) | all tools | Orchestrator — full access |
| research (陆小凤) | `read`, `write`, `web_search`, `memory_search` | `write` scoped to research output dir only |
| cto (李寻欢) | `read`, `memory_search`, `sessions_spawn` | `sessions_spawn` for QA subagent; subagent inherits exec from ACP |
| builder (冷燕) | `read`, `sessions_spawn` (runtime:acp only) | `read` for existing code; ACP enforcement handles write/exec gate |
| pm (花满楼) | `read`, `write`, `memory_search` | `write` path-scoped to `docs/**` and `*.md` only |
| biz (高老大) | `read`, `write`, `web_search` | `write` path-scoped to `docs/**` only |
| devops (傅红雪) | `read`, `sessions_spawn` (runtime:acp), `bash` (deterministic nodes only) | `bash` allowed for deterministic CI/status checks without full ACP overhead |
| cio (阿飞) | `read`, `web_search`, `memory_search` | Read-only analysis; no write |

Plugin enforces whitelists via PreToolUse hook. Unlisted tool calls return: `"[BLOCKED] Tool not in agent <id> allowlist."`

### 6.7 Linter as Hard Gate

Every code PR must pass linter before QA gate can run:
- Linter config lives at **repo root** (not per-worktree copy — worktree creation verifies config exists, does not duplicate it)
- Linter runs as a `nodeType: "deterministic"` dispatch node in the pipeline
- `escalation_trigger: { exit_code: "!= 0" }` — linter failure promotes to agentic (冷燕 fixes then re-runs)
- Linter pass is a required PR check in GitHub Actions; merge blocked if linter fails

### 6.8 GC Cron Job + Entropy Checks

Plugin registers a daily cron at 03:00:
- Delete `~/.openclaw/shared/handoffs/<TID>.md` where `completed_at + 24h < now`
- Delete `~/.openclaw/shared/events/<TID>.jsonl` by same rule
- **Entropy checks** (folded into same cron, no separate agent):
  - Scan for stale TODOs in last week's merged PRs — log count to gateway
  - Run architecture violation check (same linter rules as CI) across full codebase — log violations found
  - Scan for SOUL.md files exceeding 80 lines (proxy for "directory becoming encyclopedia") — flag for Q仔
- Log all counts to gateway log; Q仔 receives summary on next session start via HEARTBEAT.md
- Build a dedicated entropy subagent only when daily checks consistently surface issues that the linter + CI pipeline cannot auto-fix

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
