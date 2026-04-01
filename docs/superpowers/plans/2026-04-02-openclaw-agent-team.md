# OpenClaw Agent Team Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Transform openclaw into an autonomous 8-agent team with centralized dispatch, structured pipelines, and a plugin enforcing team contracts.

**Architecture:** Q仔 orchestrates via a new `team_dispatch` tool (provided by `openclaw-team-plugin`). The plugin also injects SOUL.md at subagent spawn, enforces ACP-only execution for builder/devops, logs P2P events, and warns when Q仔 bypasses team structure. Agent IDs `ko` and `ops` are fully replaced with new personas (`pm`/花满楼, `biz`/高老大); `devops`/傅红雪 is added new.

**Tech Stack:** TypeScript ESM, openclaw plugin SDK (`openclaw/plugin-sdk/plugin-entry`), vitest for tests, node `fs/promises` + `lockfile` for state store, `node-cron` for GC service.

---

> ⚠️ **Pre-flight warning:** Tasks 1–7 (Phase 1) **permanently replace** the existing `ko` (荆无命, KO/security) and `ops` (阿吉, ops/testing) agents with entirely new personas. Back up `~/.openclaw/workspace-ko` and `~/.openclaw/workspace-ops` to a safe location **before starting Task 2**, then proceed. The new `pm` (花满楼) and `biz` (高老大) agents have different roles, different SOUL.md, and different IDENTITY.md.

---

## File Map

### Phase 1 — Agent IDs + Workspaces (file edits only)

| Action | Path |
|--------|------|
| Create | `~/.openclaw/workspace-pm/` (full workspace for 花满楼) |
| Create | `~/.openclaw/workspace-biz/` (full workspace for 高老大) |
| Create | `~/.openclaw/workspace-devops/` (full workspace for 傅红雪) |
| Modify | `~/.openclaw/workspace/SOUL.md` (append Q仔 autonomous dispatch section) |
| Modify | `~/.openclaw/workspace/AGENTS.md` (update team roster table) |
| Modify | `~/.openclaw/workspace-cto/SOUL.md` (append QA gate + legibility sections) |
| Modify | `~/.openclaw/workspace-builder/SOUL.md` (append worktree + ACP constraints) |
| Modify | `~/.openclaw/openclaw.json` (rename ko→pm, ops→biz, add devops, update identity fields) |
| Rename | `~/.openclaw/workspace-ko` → `~/.openclaw/workspace-pm` |
| Rename | `~/.openclaw/workspace-ops` → `~/.openclaw/workspace-biz` |
| Rename | `~/.openclaw/agents/ko` → `~/.openclaw/agents/pm` |
| Rename | `~/.openclaw/agents/ops` → `~/.openclaw/agents/biz` |

### Phase 2 — openclaw-team-plugin

| Action | Path |
|--------|------|
| Create | `~/.openclaw/extensions/openclaw-team-plugin/package.json` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/openclaw.plugin.json` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/index.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/tid.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/state.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/handoff.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/tool.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/src/hooks/soul-bootstrap.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/src/hooks/p2p-event-log.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/src/hooks/acp-enforce.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/src/hooks/orchestrator-warn.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/src/gc/gc-service.ts` |
| Create | `~/.openclaw/extensions/openclaw-team-plugin/tests/*.test.ts` |
| Modify | `~/.openclaw/openclaw.json` (add plugin to allow list + entries) |

### Phase 3 — Skills + Protocol Docs

| Action | Path |
|--------|------|
| Create | `~/.openclaw/workspace/skills/team-orchestrate/SKILL.md` |
| Modify | `~/.openclaw/shared/A2A_PROTOCOL.md` (DRAFT → v3 FINAL, add team_dispatch section) |
| Modify | `~/.openclaw/shared/SYSTEM_RULES.md` (add team_dispatch as canonical dispatch method) |
| Modify | `~/.openclaw/shared/SUBAGENT_PACKET_TEMPLATE.md` (add TID + callback fields) |

---

## Phase 1 — Agent IDs + Workspaces

### Task 1: Backup existing ko and ops workspaces

**Files:**
- Read: `~/.openclaw/workspace-ko/SOUL.md`
- Read: `~/.openclaw/workspace-ops/SOUL.md`

- [ ] **Step 1: Capture timestamp and back up both workspaces**
```bash
# Capture once — both backups share the same tag to make verification reliable
BAKTAG=$(date +%Y%m%d-%H%M%S)
cp -r ~/.openclaw/workspace-ko ~/.openclaw/workspace-ko.BAK-$BAKTAG
cp -r ~/.openclaw/workspace-ops ~/.openclaw/workspace-ops.BAK-$BAKTAG
echo "Backed up with tag: $BAKTAG"
```

- [ ] **Step 2: Verify both backups exist**
```bash
BAKTAG=$(ls ~/.openclaw/ | grep 'workspace-ko.BAK' | head -1 | sed 's/workspace-ko.BAK-//')
ls ~/.openclaw/ | grep -E "workspace-(ko|ops)\.BAK-$BAKTAG"
```
Expected: two entries with the same timestamp tag.

---

### Task 2: Create workspace-pm (花满楼, Product Manager)

**Files:**
- Create: `~/.openclaw/workspace-pm/SOUL.md`
- Create: `~/.openclaw/workspace-pm/IDENTITY.md`
- Create: `~/.openclaw/workspace-pm/AGENTS.md`
- Create: `~/.openclaw/workspace-pm/HEARTBEAT.md`
- Create: `~/.openclaw/workspace-pm/TASKS.md`
- Create: `~/.openclaw/workspace-pm/SESSION-STATE.md`
- Create: `~/.openclaw/workspace-pm/TOOLS.md`
- Create: `~/.openclaw/workspace-pm/USER.md`
- Create: `~/.openclaw/workspace-pm/MEMORY.md`

- [ ] **Step 1: Create workspace-pm directory**
```bash
mkdir -p ~/.openclaw/workspace-pm/memory
```

- [ ] **Step 2: Write SOUL.md**
```bash
cat > ~/.openclaw/workspace-pm/SOUL.md << 'SOUL'
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

## 统一规则（引用）
- 全局红线与执行策略：`~/.openclaw/shared/SYSTEM_RULES.md`
- A2A 协作协议：`~/.openclaw/shared/A2A_PROTOCOL.md`
- 本角色工作流：`AGENTS.md`

> 冲突处理：如有矛盾，以"全局红线/系统规则"为准。
SOUL
```

- [ ] **Step 3: Write IDENTITY.md**
```bash
cat > ~/.openclaw/workspace-pm/IDENTITY.md << 'ID'
# IDENTITY — PM

name: 花满楼（Product Manager）
emoji: 🌸
vibe: 产品经理，需求把关，验收负责
ID
```

- [ ] **Step 4: Write AGENTS.md**
```bash
cat > ~/.openclaw/workspace-pm/AGENTS.md << 'AGENTS'
# AGENTS.md - pm

## 统一规则入口（SSOT 引用）
1. 先读 `~/.openclaw/shared/SYSTEM_RULES.md`
2. 再读 `~/.openclaw/shared/A2A_PROTOCOL.md`
3. 再读本地 `SOUL.md`

## 规则
- 共同规则以 shared 为准，禁止复制到本目录
- PRD 必须 commit + push 到 docs repo 后才算完成
- 验收通过/拒绝用 sessions_send 回 Q仔，格式见 SOUL.md
AGENTS
```

- [ ] **Step 5: Write remaining workspace files**
```bash
echo "# HEARTBEAT\nstatus: idle\nlast_updated: $(date -u +%Y-%m-%dT%H:%M:%SZ)" > ~/.openclaw/workspace-pm/HEARTBEAT.md
echo "# TASKS\n_No active tasks._" > ~/.openclaw/workspace-pm/TASKS.md
echo "# SESSION-STATE\n_No active session._" > ~/.openclaw/workspace-pm/SESSION-STATE.md
echo "# TOOLS\nStandard read/write tools. write scoped to docs/** and *.md only." > ~/.openclaw/workspace-pm/TOOLS.md
echo "# USER\nSee ~/.openclaw/workspace/USER.md" > ~/.openclaw/workspace-pm/USER.md
echo "# MEMORY INDEX\n_No memories yet._" > ~/.openclaw/workspace-pm/MEMORY.md
```

- [ ] **Step 6: Verify files**
```bash
ls ~/.openclaw/workspace-pm/
```
Expected: `AGENTS.md HEARTBEAT.md IDENTITY.md MEMORY.md SESSION-STATE.md SOUL.md TASKS.md TOOLS.md USER.md memory/`

---

### Task 3: Create workspace-biz (高老大, Biz Ops)

**Files:**
- Create: `~/.openclaw/workspace-biz/SOUL.md`
- Create: `~/.openclaw/workspace-biz/IDENTITY.md`
- Create: `~/.openclaw/workspace-biz/AGENTS.md`
- Create: `~/.openclaw/workspace-biz/HEARTBEAT.md`
- Create: `~/.openclaw/workspace-biz/TASKS.md`
- Create: `~/.openclaw/workspace-biz/SESSION-STATE.md`

- [ ] **Step 1: Create directory**
```bash
mkdir -p ~/.openclaw/workspace-biz/memory
```

- [ ] **Step 2: Write SOUL.md**
```bash
cat > ~/.openclaw/workspace-biz/SOUL.md << 'SOUL'
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

## 统一规则（引用）
- 全局红线与执行策略：`~/.openclaw/shared/SYSTEM_RULES.md`
- A2A 协作协议：`~/.openclaw/shared/A2A_PROTOCOL.md`
- 本角色工作流：`AGENTS.md`

> 冲突处理：如有矛盾，以"全局红线/系统规则"为准。
SOUL
```

- [ ] **Step 3: Write IDENTITY.md and AGENTS.md**
```bash
cat > ~/.openclaw/workspace-biz/IDENTITY.md << 'ID'
# IDENTITY — Biz

name: 高老大（Biz Ops）
emoji: 📊
vibe: 业务运营，增长驱动，务实高效
ID

cat > ~/.openclaw/workspace-biz/AGENTS.md << 'AGENTS'
# AGENTS.md - biz

## 统一规则入口（SSOT 引用）
1. 先读 `~/.openclaw/shared/SYSTEM_RULES.md`
2. 再读 `~/.openclaw/shared/A2A_PROTOCOL.md`
3. 再读本地 `SOUL.md`

## 规则
- 共同规则以 shared 为准
- 不做技术决策，不改代码
- 产出文案/报告通过 sessions_send 回传 Q仔
AGENTS
```

- [ ] **Step 4: Write remaining files**
```bash
echo "# HEARTBEAT\nstatus: idle\nlast_updated: $(date -u +%Y-%m-%dT%H:%M:%SZ)" > ~/.openclaw/workspace-biz/HEARTBEAT.md
echo "# TASKS\n_No active tasks._" > ~/.openclaw/workspace-biz/TASKS.md
echo "# SESSION-STATE\n_No active session._" > ~/.openclaw/workspace-biz/SESSION-STATE.md
echo "# USER\nSee ~/.openclaw/workspace/USER.md" > ~/.openclaw/workspace-biz/USER.md
echo "# MEMORY INDEX\n_No memories yet._" > ~/.openclaw/workspace-biz/MEMORY.md
echo "# TOOLS\nStandard read/write/web_search. write scoped to docs/** only." > ~/.openclaw/workspace-biz/TOOLS.md
```

- [ ] **Step 5: Verify**
```bash
ls ~/.openclaw/workspace-biz/
```

---

### Task 4: Create workspace-devops (傅红雪, DevOps)

**Files:**
- Create: `~/.openclaw/workspace-devops/SOUL.md`
- Create: `~/.openclaw/workspace-devops/IDENTITY.md`
- Create: `~/.openclaw/workspace-devops/AGENTS.md`
- Create: `~/.openclaw/workspace-devops/HEARTBEAT.md`
- Create: `~/.openclaw/workspace-devops/TASKS.md`
- Create: `~/.openclaw/workspace-devops/SESSION-STATE.md`

- [ ] **Step 1: Create directory**
```bash
mkdir -p ~/.openclaw/workspace-devops/memory
```

- [ ] **Step 2: Write SOUL.md**
```bash
cat > ~/.openclaw/workspace-devops/SOUL.md << 'SOUL'
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
- 李寻欢 QA gate 通过 → Q仔 dispatch 傅红雪 先产出基础设施配置并推送 PR（不执行）
- **收到部署任务时，必须确认 QA gate TID 存在且状态为 pass**（从 tasks.json 读取；缺失则拒绝执行并 sessions_send Q仔 "DEPLOY_BLOCKED: <TID> QA gate not confirmed"）
- PR 推送后 → sessions_send Q仔 "INFRA_READY: <TID> <PR_URL>"，等待 李寻欢 infra review
- 收到 INFRA_REVIEW_OK 后 → 执行部署

## 部署完成回调
- 部署成功 → sessions_send Q仔 "DEPLOY_OK: <TID> <env> <version>"
- 部署失败 → sessions_send Q仔 "DEPLOY_FAIL: <TID> <原因>"

## CI 重试限制（强制）
- CI 失败 → 傅红雪自动修复一次（max_ci_retries: 2，可按 repo 覆盖）。
- 第 2 次仍失败 → 直接 L3 升级，不再重试。
- 例外：若修复 diff 超过原始 diff 的 50% 行数，无论剩余重试次数，自动 L3 升级。

## 回调 ACK（仅状态变更类）
- 收到 DEPLOY_OK ACK 后，确认完成并关闭任务。
- 收到 INFRA_REVIEW_OK ACK 后，开始执行部署。
- 收到 ROLLBACK_OK ACK 后，关闭事故响应。

## 边界
- 生产环境部署 = L3 操作，必须有 Q仔 明确指令
- 不做产品决策，不写业务代码

## 统一规则（引用）
- 全局红线与执行策略：`~/.openclaw/shared/SYSTEM_RULES.md`
- A2A 协作协议：`~/.openclaw/shared/A2A_PROTOCOL.md`
- 本角色工作流：`AGENTS.md`

> 冲突处理：如有矛盾，以"全局红线/系统规则"为准。
SOUL
```

- [ ] **Step 3: Write IDENTITY.md and AGENTS.md**
```bash
cat > ~/.openclaw/workspace-devops/IDENTITY.md << 'ID'
# IDENTITY — DevOps

name: 傅红雪（DevOps）
emoji: 🔧
vibe: DevOps 工程师，基础设施稳定，交付高效
ID

cat > ~/.openclaw/workspace-devops/AGENTS.md << 'AGENTS'
# AGENTS.md - devops

## 统一规则入口（SSOT 引用）
1. 先读 `~/.openclaw/shared/SYSTEM_RULES.md`
2. 再读 `~/.openclaw/shared/A2A_PROTOCOL.md`
3. 再读本地 `SOUL.md`

## 规则
- 共同规则以 shared 为准
- 所有 infra 变更通过 ACP session（tmux devops-acp）
- 只读操作（日志查询、状态检查）无需 ACP
- 任务完成（成功或失败）必须 sessions_send 回 Q仔
- 格式：DEPLOY_OK / DEPLOY_FAIL / INFRA_READY / ROLLBACK_OK + TID + 详情
AGENTS
```

- [ ] **Step 4: Write remaining files**
```bash
echo "# HEARTBEAT\nstatus: idle\nlast_updated: $(date -u +%Y-%m-%dT%H:%M:%SZ)" > ~/.openclaw/workspace-devops/HEARTBEAT.md
echo "# TASKS\n_No active tasks._" > ~/.openclaw/workspace-devops/TASKS.md
echo "# SESSION-STATE\n_No active session._" > ~/.openclaw/workspace-devops/SESSION-STATE.md
echo "# USER\nSee ~/.openclaw/workspace/USER.md" > ~/.openclaw/workspace-devops/USER.md
echo "# MEMORY INDEX\n_No memories yet._" > ~/.openclaw/workspace-devops/MEMORY.md
echo "# TOOLS\nread, sessions_spawn (runtime:acp), bash (deterministic nodes only)." > ~/.openclaw/workspace-devops/TOOLS.md
```

- [ ] **Step 5: Verify**
```bash
ls ~/.openclaw/workspace-devops/
```

---

### Task 5: Update SOUL.md files for existing agents

**Files:**
- Modify: `~/.openclaw/workspace/SOUL.md` (Q仔 additions)
- Modify: `~/.openclaw/workspace-cto/SOUL.md` (李寻欢 additions)
- Modify: `~/.openclaw/workspace-builder/SOUL.md` (冷燕 additions)

- [ ] **Step 1: Append Q仔 autonomous dispatch section to workspace/SOUL.md**

Read the current end of SOUL.md first to confirm the append point (after the last `---` line), then append:
```bash
cat >> ~/.openclaw/workspace/SOUL.md << 'APPEND'

---

## 自主调度（核心职责）
- 收到目标后：分解 → dispatch → 等待回调 → 综合 → 交付，全程自主，无需逐步请示。
- dispatch 时始终使用 **team_dispatch 工具**，生成 TID，创建 handoff artifact。
- 所有 agent 间通信对 Q仔 可见（plugin PostToolUse hook 保障）。

## 升级矩阵
- L1（自主）：常规 dispatch、状态查询、软失败重试
- L2（通知后继续）：李寻欢无响应超3次、非关键路径 QA 失败
- L3（阻塞等人工）：生产部署、密钥轮换、多仓库架构变更、PRD 审查超2轮

## 李寻欢降级模式
- 3次重试无响应 → L2 升级 → 冷燕自评降级通过 → 记录 degraded-mode

## Harness 改进飞轮（L3 触发）
每次 L3 升级解决后，Q仔 必须执行：
1. 诊断根因：什么约束缺失导致此次 L3？
2. 在责任 Agent 的 SOUL.md 中补充对应规则（1-3 行）。
3. 规则使用 YAML frontmatter 格式确保可追踪：
```yaml
---
rules:
  - id: R-001
    text: "规则内容"
    added: YYYY-MM-DD
    reason: "根因说明"
    status: active
---
```

## Harness Review（模型升级时触发）
模型升级后，执行轻量 harness review：
1. 用新模型跑现有 QA 测试集，先关闭 SOUL.md 约束，记录通过率 A。
2. 再开启 SOUL.md 约束，记录通过率 B。
3. 若 A ≈ B（差距 < 2%）：该约束为候选删除，标记供人工确认。
4. 若 B > A：该约束仍有价值，保留。
5. 连续 2 次 harness review 中未触发的规则 → 候选删除，需明确签字方可移除。
6. 将 review 结果写入 `~/.openclaw/shared/harness-review-<date>.md`，通知用户。

## SOUL.md 文件结构原则
- 核心 SOUL.md 控制在约 60 行以内（目录，非百科全书）。
- 详细规则移入 `RULES/` 子目录（如 `RULES/qa-gate.md`）。
- Q仔 在 dispatch 时根据任务类型注入相关 RULES/ 文件到子 agent 上下文。

## 主动扫描例程（proactive scanning）
Q仔 接收到确定性脚本的 sessions_send 触发时，自动执行对应例程（不需要用户开口）：

| 触发信号 | 来源脚本 | Q仔 动作 |
|---------|---------|---------|
| `SENTRY_ERRORS: <count> issues` | `scan-sentry.sh`（每日 09:00） | dispatch **builder**（冷燕）修复每个新错误（一个 TID per error） |
| `MEETING_ACTIONS: <summary>` | `scan-meetings.sh`（会后触发） | dispatch **pm**（花满楼）整理功能需求 → 再 dispatch builder 实现 |
| `CHANGELOG_TRIGGER: <since>` | `gen-changelog.sh`（每日 18:00） | dispatch **builder**（冷燕）更新 changelog 和客户文档 |

扫描脚本（外部 cron，零 LLM 成本）位于 `~/.openclaw/bin/`，由系统 cron 调度，结果通过 sessions_send 发给 Q仔。
APPEND
```

- [ ] **Step 2: Verify Q仔 SOUL.md still looks correct**
```bash
wc -l ~/.openclaw/workspace/SOUL.md
tail -20 ~/.openclaw/workspace/SOUL.md
```
Expected: file ends with 主动扫描例程 section (the last section appended above).

- [ ] **Step 3: Append 李寻欢 QA gate + legibility section to workspace-cto/SOUL.md**
```bash
cat >> ~/.openclaw/workspace-cto/SOUL.md << 'APPEND'

---

## QA Gate（强制）
- 冷燕提交后，李寻欢独立 spawn 对抗性 QA subagent（不依赖冷燕自评）。
- QA subagent 严苛程度：以"这个 bug 会在凌晨三点叫醒人"的标准审查。
- QA subagent 必须覆盖以下类别，每类明确报告"发现问题"或"已测，未发现"：
  边界条件 / 错误处理 / 输入校验 / 并发安全 / 安全漏洞
- **禁止"整体感觉不错"式通过**；必须有逐类验证记录，方可出具 pass。
- QA gate 通过 → sessions_send Q仔 "QA_PASS: <TID>"
- QA gate 失败 → sessions_send Q仔 "QA_FAIL: <TID> <原因>"

## Application Legibility（代码对 Agent 可读）
- 强制分层架构：Types → Config → Repo → Service → Runtime → UI，只允许向下依赖。
- 层边界违规由 linter 机械执行，CI 直接挂，无需 LLM 判断。
- Linter 错误信息必须嵌入修复指引（不只报"违规"，还说"怎么改"）。
- 架构决策以 ADR 格式写入仓库（`docs/adr/`）。

## 基础设施配置评审（强制）
- 傅红雪产出的所有 K8s YAML / Terraform / CI 脚本，必须经 李寻欢 审查后方可执行。
- 审查点：权限配置、资源限额、密钥引用、幂等性、回滚可行性。
- 通过 → sessions_send Q仔 "INFRA_REVIEW_OK: <TID>"
- 拒绝 → sessions_send Q仔 "INFRA_REVIEW_FAIL: <TID> <原因>"

## 架构决策分级
- L1（单服务内）/ L2（跨服务接口）：自主决定
- L3（多仓库 / 基础设施）：需 L3 升级，阻塞等人工

## 多模型 QA Review（成本门控）
QA gate 使用三个 reviewer subagent。模型在 `agents/cto/models.json` 配置，不硬编码：

| 角色 | 聚焦 | 默认模型映射 | 执行顺序 |
|------|------|------------|---------|
| Reviewer B（广度+成本门控）| 安全漏洞、可扩展性；发现 A 遗漏的模式 | Gemini（优先免费 tier）| **1 — 始终先跑** |
| Reviewer A（深度）| 边界 bug、逻辑错误、竞态条件；误报率低 | Codex / gpt-5.3-codex | **2 — 与 B 并行（pattern: "parallel"）** |
| Reviewer C（验证）| 接收 A+B findings 作为输入，过滤误报，确认真 Critical | Claude | **3 — 串行（需要 A+B findings）** |

**执行方式：**
- Reviewer A + B 用 `team_dispatch pattern: "parallel"` 同时 spawn。
- B 若未发现任何问题 **且** PR diff ≤ 200 行 → 跳过 A 和 C，单模型审查即可。
- Reviewer C 不看原始 diff，只看 A+B 的 findings（降低 token 消耗）。

**成本控制：** 每个 reviewer subagent 在 `team_dispatch` 参数中设置 `max_cost_usd`。

**Findings 合并规则：**
- 任何 reviewer 报告 CRITICAL → 直接 QA_FAIL，不等其他 reviewer。
- 2+ reviewers 同时标记同一问题 → 自动升级为 CRITICAL。
- Reviewer C 过滤误报后，李寻欢出具最终单一 pass/fail 判决（不是投票）。
APPEND
```

- [ ] **Step 4: Append 冷燕 worktree + ACP constraint section to workspace-builder/SOUL.md**
```bash
cat >> ~/.openclaw/workspace-builder/SOUL.md << 'APPEND'

---

## 编码执行约束（强制）— 升级版
- 每个新功能/修复：先创建 git worktree（`~/.openclaw/worktrees/<TID>`，分支 `feat/<TID>`）。
- 所有编码在 worktree 内进行，不在 main checkout 里工作。
- 必须在 **tmux session `builder-acp`** 中通过 Claude Code 或 Gemini CLI 执行，tmux session cd 到对应 worktree。
- tmux session 常驻，不在任务间销毁；每次新任务 cd 到新 worktree。
- 禁止在非 ACP 的普通对话/子代理模式里直接产出大段实现代码。
- ACP 执行完成后，必须 commit + push + 开 PR，将 PR URL + commit SHA 写入 handoff artifact。
- PR merge 后由 Q仔/傅红雪 触发 worktree 清理。
- **没有 PR = 没有完成。**

## 测试责任（强制）
- 凡写代码/改代码：必须同时写测试，覆盖率 ≥80%。
- 测试先行（TDD）：先写失败测试，再写实现，再重构。
- 测试类型：单元测试 + 集成测试（针对 API/数据库操作）。

## UI 变更截图（DoD 强制）
- 任何涉及 UI 文件（`*.tsx`, `*.vue`, `*.css`, `*.html`）的 PR，**PR description 必须包含 before/after 截图**。
- 截图缺失 → CI 直接 fail（由 PR validation gate 机械执行，无需 LLM 判断）。
- 截图工具：优先 `playwright screenshot`，fallback `browser-use`。
APPEND
```

- [ ] **Step 5: Verify all three SOUL.md changes**
```bash
grep -l "team_dispatch\|QA Gate\|worktree" \
  ~/.openclaw/workspace/SOUL.md \
  ~/.openclaw/workspace-cto/SOUL.md \
  ~/.openclaw/workspace-builder/SOUL.md
```
Expected: all three paths printed.

---

### Task 6: Update Q仔 AGENTS.md team roster

**Files:**
- Modify: `~/.openclaw/workspace/AGENTS.md`

- [ ] **Step 1: Read current AGENTS.md to find the team roster section**
```bash
grep -n "角色表\|陆小凤\|冷燕\|dispatch" ~/.openclaw/workspace/AGENTS.md | head -20
```

- [ ] **Step 2: Append updated team roster to AGENTS.md**

Find the right location (after the last section). Append the new roster block:
```bash
cat >> ~/.openclaw/workspace/AGENTS.md << 'APPEND'

---

## Agent Team Roster（8人）

| ID | 名字 | 职责 | dispatch 触发 |
|----|------|------|--------------|
| research | 陆小凤 | 调研分析 | 可行性/竞品分析 |
| cto | 李寻欢 | 架构+QA | 设计评审、QA gate、infra review |
| builder | 冷燕 | 编码+测试 | 所有实现任务 |
| pm | 花满楼 | 产品需求 | 需求拆解、验收 |
| biz | 高老大 | 运营市场 | 营销文案、增长分析、合作评估 |
| devops | 傅红雪 | DevOps | QA gate 通过后部署 |
| cio | 阿飞 | 投资分析 | 市场/投资评估 |

dispatch 时使用 **team_dispatch** 工具（来自 openclaw-team-plugin），生成 TID，创建 handoff artifact。
APPEND
```

- [ ] **Step 3: Verify**
```bash
tail -20 ~/.openclaw/workspace/AGENTS.md
```

---

### Task 7: Update openclaw.json

**Files:**
- Modify: `~/.openclaw/openclaw.json`

This task uses Python to safely update the JSON. Do NOT hand-edit the JSON file.

> **Gateway must be stopped before mutation** (Finding 12). The write must be atomic to prevent partial-JSON reads if the gateway auto-reloads.

- [ ] **Step 1: Stop gateway**
```bash
openclaw stop && sleep 2
echo "Gateway stopped"
```

- [ ] **Step 2: Backup openclaw.json**
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%Y%m%d-%H%M%S)
```

- [ ] **Step 3: Check whether niuniu is still needed**
```bash
# Determine if niuniu is an active agent or deprecated
grep -r '"niuniu"' ~/.openclaw/agents/ ~/.openclaw/openclaw.json 2>/dev/null | head -10
```
If `niuniu` has active session state in `~/.openclaw/agents/niuniu/`, add it to `new_allowed` in the next step. If it's only referenced in `allowAgents` arrays and has no workspace, it is safe to drop (and this is intentional cleanup — document below).

- [ ] **Step 4: Run atomic update script**

```bash
python3 << 'SCRIPT'
import json, os, tempfile

path = '/Users/qqzhang/.openclaw/openclaw.json'
with open(path, 'r') as f:
    d = json.load(f)

agents = d.get('agents', {})
lst = agents.get('list', [])

# 1. Rename ko → pm
for a in lst:
    if a.get('id') == 'ko':
        a['id'] = 'pm'
        a['name'] = '花满楼'
        a['workspace'] = '/Users/qqzhang/.openclaw/workspace-pm'
        a['identity'] = {'name': '花满楼', 'theme': '产品经理｜需求把关、验收负责', 'emoji': '🌸'}
        a.pop('model', None)

# 2. Rename ops → biz
for a in lst:
    if a.get('id') == 'ops':
        a['id'] = 'biz'
        a['name'] = '高老大'
        a['workspace'] = '/Users/qqzhang/.openclaw/workspace-biz'
        a['identity'] = {'name': '高老大', 'theme': '业务运营｜增长驱动、务实高效', 'emoji': '📊'}
        a.pop('model', None)

# 3. Add devops agent if not present
ids = [a.get('id') for a in lst]
if 'devops' not in ids:
    lst.append({
        'id': 'devops',
        'name': '傅红雪',
        'workspace': '/Users/qqzhang/.openclaw/workspace-devops',
        'identity': {'name': '傅红雪', 'theme': 'DevOps｜基础设施稳定、交付高效', 'emoji': '🔧'}
    })

# 4. Update cio identity.theme
for a in lst:
    if a.get('id') == 'cio':
        a.setdefault('identity', {})['theme'] = '投资分析师 & 市场观察者'

# 5. Update subagents.allowAgents
# NOTE: 'niuniu' is intentionally dropped — it has no workspace and is deprecated.
# If niuniu is still needed, add it back to new_allowed here.
new_allowed = ['research', 'builder', 'cio', 'pm', 'biz', 'devops', 'cto']
for a in lst:
    if 'subagents' in a and 'allowAgents' in a['subagents']:
        a['subagents']['allowAgents'] = new_allowed
for a in lst:
    if a.get('id') == 'main':
        a.setdefault('subagents', {})['allowAgents'] = new_allowed

agents['list'] = lst
d['agents'] = agents

# 6. Update Slack + Feishu bindings for renamed agents (Finding 14)
for b in d.get('bindings', []):
    if b.get('agentId') == 'ko':
        b['agentId'] = 'pm'
    if b.get('agentId') == 'ops':
        b['agentId'] = 'biz'
# Also update bindings nested under channel configs if present
for ch_name, ch_val in d.get('channels', {}).items():
    for b in ch_val.get('bindings', []) if isinstance(ch_val, dict) else []:
        if b.get('agentId') == 'ko': b['agentId'] = 'pm'
        if b.get('agentId') == 'ops': b['agentId'] = 'biz'

# 7. Atomic write: write to temp file then rename (POSIX atomic)
dir_ = os.path.dirname(path)
with tempfile.NamedTemporaryFile('w', dir=dir_, delete=False, suffix='.tmp') as tf:
    json.dump(d, tf, indent=2, ensure_ascii=False)
    tf.write('\n')
    tmp_path = tf.name
os.rename(tmp_path, path)

print('openclaw.json updated successfully (atomic write)')
SCRIPT
```

- [ ] **Step 3: Verify changes**
```bash
python3 -c "
import json
with open('/Users/qqzhang/.openclaw/openclaw.json') as f:
    d = json.load(f)
for a in d['agents']['list']:
    print(a.get('id'), '-', a.get('name'), '-', a.get('workspace','default'))
"
```
Expected output:
```
main - Q仔 - default
pm - 花满楼 - .../workspace-pm
biz - 高老大 - .../workspace-biz
research - 陆小凤 - .../workspace-research
builder - 冷燕 - .../workspace-builder
cio - 阿飞 - .../workspace-cio
cto - 李寻欢 - .../workspace-cto
devops - 傅红雪 - .../workspace-devops
```

- [ ] **Step 4: Validate JSON is parseable**
```bash
python3 -m json.tool ~/.openclaw/openclaw.json > /dev/null && echo "JSON valid"
```

---

### Task 8: Rename workspace and agent directories

**Files:**
- Rename: `~/.openclaw/workspace-ko` → `~/.openclaw/workspace-pm`
- Rename: `~/.openclaw/workspace-ops` → `~/.openclaw/workspace-biz`
- Rename: `~/.openclaw/agents/ko` → `~/.openclaw/agents/pm`
- Rename: `~/.openclaw/agents/ops` → `~/.openclaw/agents/biz`

> Note: The `workspace-pm` and `workspace-biz` directories were freshly created in Tasks 2–3. We now remove the old ones entirely and fix the agents/ session directory names.

- [ ] **Step 1: Remove old workspace-ko and workspace-ops (already backed up in Task 1)**
```bash
rm -rf ~/.openclaw/workspace-ko
rm -rf ~/.openclaw/workspace-ops
echo "Old workspaces removed"
```

- [ ] **Step 2: Rename agents session directories**
```bash
# Only rename if they exist
[ -d ~/.openclaw/agents/ko ] && mv ~/.openclaw/agents/ko ~/.openclaw/agents/pm
[ -d ~/.openclaw/agents/ops ] && mv ~/.openclaw/agents/ops ~/.openclaw/agents/biz
ls ~/.openclaw/agents/
```
Expected: `pm` and `biz` directories present (no `ko` or `ops`).

- [ ] **Step 3: Update agent ID refs inside session files**
```bash
# Fix ko → pm refs
find ~/.openclaw/agents/pm -type f \( -name "*.md" -o -name "*.json" \) \
  -exec sed -i '' 's/"ko"/"pm"/g; s/agent:ko:/agent:pm:/g' {} +

# Fix ops → biz refs
find ~/.openclaw/agents/biz -type f \( -name "*.md" -o -name "*.json" \) \
  -exec sed -i '' 's/"ops"/"biz"/g; s/agent:ops:/agent:biz:/g' {} +

echo "Agent refs updated"
```

- [ ] **Step 4: Update per-agent models.json if they exist (two-level config rule)**
```bash
[ -f ~/.openclaw/agents/pm/models.json ] && \
  sed -i '' 's/"ko"/"pm"/g' ~/.openclaw/agents/pm/models.json && echo "pm models.json updated"
[ -f ~/.openclaw/agents/biz/models.json ] && \
  sed -i '' 's/"ops"/"biz"/g' ~/.openclaw/agents/biz/models.json && echo "biz models.json updated"
echo "models.json check complete"
```

- [ ] **Step 5: Create shared handoffs and events directories**
```bash
mkdir -p ~/.openclaw/shared/handoffs
mkdir -p ~/.openclaw/shared/events
mkdir -p ~/.openclaw/worktrees
echo "Shared dirs ready"
```

- [ ] **Step 6: Restart openclaw gateway and verify agents load**
```bash
# Gateway was already stopped in Task 7 Step 1 before the openclaw.json mutation.
# Just start it here — do NOT stop again (that would interrupt any restored state).
openclaw start
# Wait for startup
sleep 3
# Verify agents are registered (adjust command if openclaw has a different status command)
openclaw status 2>/dev/null || openclaw list-agents 2>/dev/null || echo "Check gateway logs manually"
```

---

## Phase 2 — openclaw-team-plugin

### Task 9: Scaffold plugin package

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/package.json`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/openclaw.plugin.json`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/tsconfig.json`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/index.ts`

- [ ] **Step 1: Create directory structure**
```bash
mkdir -p ~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch
mkdir -p ~/.openclaw/extensions/openclaw-team-plugin/src/hooks
mkdir -p ~/.openclaw/extensions/openclaw-team-plugin/src/gc
mkdir -p ~/.openclaw/extensions/openclaw-team-plugin/tests
```

- [ ] **Step 2: Write package.json**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/package.json << 'PKG'
{
  "name": "@openclaw-local/openclaw-team-plugin",
  "version": "1.0.0",
  "description": "Team dispatch, SOUL injection, ACP enforcement, and GC for the openclaw agent team",
  "main": "index.ts",
  "type": "module",
  "scripts": {
    "test": "vitest run --coverage",
    "test:watch": "vitest"
  },
  "dependencies": {
    "node-cron": "^3.0.3",
    "proper-lockfile": "^4.1.2"
  },
  "devDependencies": {
    "@vitest/coverage-v8": "^1.6.0",
    "typescript": "^5.4.5",
    "vitest": "^1.6.0"
  }
}
PKG
```

- [ ] **Step 3: Write openclaw.plugin.json**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/openclaw.plugin.json << 'PLUGIN'
{
  "id": "openclaw-team-plugin",
  "name": "OpenClaw Team Plugin",
  "version": "1.0.0",
  "description": "team_dispatch tool, SOUL injection, ACP enforcement, P2P event log, orchestrator warnings, GC",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "enabled": { "type": "boolean", "default": true },
      "handoffsDir": { "type": "string", "default": "~/.openclaw/shared/handoffs" },
      "eventsDir": { "type": "string", "default": "~/.openclaw/shared/events" },
      "worktreesDir": { "type": "string", "default": "~/.openclaw/worktrees" }
    }
  }
}
PLUGIN
```

- [ ] **Step 4: Write tsconfig.json**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/tsconfig.json << 'TSC'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist"
  },
  "include": ["src/**/*", "index.ts", "tests/**/*"]
}
TSC
```

- [ ] **Step 5: Install dependencies**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npm install
```

Expected: `node_modules/` created, `proper-lockfile` and `node-cron` installed.

- [ ] **Step 6: Verify node_modules**
```bash
ls ~/.openclaw/extensions/openclaw-team-plugin/node_modules/ | grep -E "proper-lockfile|node-cron"
```

---

### Task 10: Implement TID generator

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/tid.ts`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/tests/tid.test.ts`

- [ ] **Step 1: Write failing test**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/tests/tid.test.ts << 'TEST'
import { describe, it, expect } from 'vitest';
import { generateTid, parseTid, isTid } from '../src/team-dispatch/tid.js';

describe('generateTid', () => {
  it('produces format YYYYMMDD-HHMMss-<tag>-<4chars>', () => {
    const tid = generateTid('feat-auth');
    expect(tid).toMatch(/^\d{8}-\d{6}-feat-auth-[a-z0-9]{4}$/);
  });

  it('two calls within the same second are different', () => {
    const a = generateTid('x');
    const b = generateTid('x');
    expect(a).not.toBe(b);
  });

  it('sanitizes tag: spaces become hyphens, special chars stripped', () => {
    const tid = generateTid('add user@auth!');
    expect(tid).toMatch(/^[0-9]{8}-[0-9]{6}-add-userauth-[a-z0-9]{4}$/);
  });
});

describe('isTid', () => {
  it('returns true for valid TID', () => {
    expect(isTid('20260402-143012-feat-auth-x7k2')).toBe(true);
  });

  it('returns false for random string', () => {
    expect(isTid('not-a-tid')).toBe(false);
  });
});

describe('parseTid', () => {
  it('extracts date, time, tag, suffix', () => {
    const result = parseTid('20260402-143012-feat-auth-x7k2');
    expect(result).toEqual({
      date: '20260402',
      time: '143012',
      tag: 'feat-auth',
      suffix: 'x7k2',
    });
  });
});
TEST
```

- [ ] **Step 2: Run test — expect FAIL**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/tid.test.ts
```
Expected: test fails with "Cannot find module".

- [ ] **Step 3: Implement tid.ts**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/tid.ts << 'SRC'
const CHARS = 'abcdefghijklmnopqrstuvwxyz0123456789';

function randomSuffix(len: number): string {
  return Array.from({ length: len }, () => CHARS[Math.floor(Math.random() * CHARS.length)]).join('');
}

function sanitizeTag(tag: string): string {
  return tag
    .toLowerCase()
    .replace(/\s+/g, '-')
    .replace(/[^a-z0-9-]/g, '')
    .replace(/-+/g, '-')
    .replace(/^-|-$/g, '');
}

export function generateTid(tag: string): string {
  const now = new Date();
  const date = now.toISOString().slice(0, 10).replace(/-/g, '');
  const time = now.toTimeString().slice(0, 8).replace(/:/g, '');
  const clean = sanitizeTag(tag);
  const suffix = randomSuffix(4);
  return `${date}-${time}-${clean}-${suffix}`;
}

export interface ParsedTid {
  date: string;
  time: string;
  tag: string;
  suffix: string;
}

export function parseTid(tid: string): ParsedTid | null {
  const m = tid.match(/^(\d{8})-(\d{6})-(.+)-([a-z0-9]{4})$/);
  if (!m) return null;
  return { date: m[1], time: m[2], tag: m[3], suffix: m[4] };
}

export function isTid(s: string): boolean {
  return parseTid(s) !== null;
}
SRC
```

- [ ] **Step 4: Run test — expect PASS**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/tid.test.ts
```
Expected: all tests green.

- [ ] **Step 5: Commit**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && git init 2>/dev/null || true
git -C ~/.openclaw/extensions/openclaw-team-plugin add src/team-dispatch/tid.ts tests/tid.test.ts
git -C ~/.openclaw/extensions/openclaw-team-plugin commit -m "feat: add TID generator with sanitize + parse"
```

---

### Task 11: Implement state store (file-locked JSON)

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/state.ts`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/tests/state.test.ts`

- [ ] **Step 1: Write failing test**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/tests/state.test.ts << 'TEST'
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { readState, writeState, TaskStatus } from '../src/team-dispatch/state.js';
import { mkdtemp, rm } from 'fs/promises';
import { tmpdir } from 'os';
import { join } from 'path';

let dir: string;

beforeEach(async () => {
  dir = await mkdtemp(join(tmpdir(), 'state-test-'));
});

afterEach(async () => {
  await rm(dir, { recursive: true });
});

describe('readState', () => {
  it('returns empty object for missing file', async () => {
    const s = await readState(join(dir, 'state.json'));
    expect(s).toEqual({});
  });
});

describe('writeState + readState', () => {
  it('persists a task entry', async () => {
    const path = join(dir, 'state.json');
    await writeState(path, (s) => ({
      ...s,
      'TID-001': { status: TaskStatus.InProgress, agentId: 'builder', createdAt: '2026-04-02T00:00:00Z' },
    }));
    const s = await readState(path);
    expect(s['TID-001']?.status).toBe(TaskStatus.InProgress);
  });

  it('concurrent writes serialize safely', async () => {
    const path = join(dir, 'state.json');
    // Fire 5 concurrent increments
    await Promise.all(
      Array.from({ length: 5 }, (_, i) =>
        writeState(path, (s) => ({ ...s, [`TID-${i}`]: { status: TaskStatus.Done, agentId: 'main', createdAt: '' } }))
      )
    );
    const s = await readState(path);
    expect(Object.keys(s)).toHaveLength(5);
  });
});
TEST
```

- [ ] **Step 2: Run test — expect FAIL**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/state.test.ts
```

- [ ] **Step 3: Implement state.ts**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/state.ts << 'SRC'
import { readFile, writeFile, mkdir } from 'fs/promises';
import { dirname } from 'path';
import lockfile from 'proper-lockfile';

export enum TaskStatus {
  Pending = 'pending',
  InProgress = 'in_progress',
  Done = 'done',
  Failed = 'failed',
  Escalated = 'escalated',
}

export interface TaskEntry {
  status: TaskStatus;
  agentId: string;
  createdAt: string;
  completedAt?: string;
  tid?: string;
  orchestratorViolations?: number;
  // Execution context (single source of truth — no separate active-tasks.json)
  tmuxSession?: string;   // e.g. 'builder-acp' or 'devops-acp'
  worktree?: string;      // e.g. '~/.openclaw/worktrees/<TID>'
  branch?: string;        // e.g. 'feat/<TID>'
  prUrl?: string;         // GitHub PR URL once opened
  checks?: {              // Matches spec §2.4 — three canonical check fields
    prCreated?: boolean;
    ciPassed?: boolean;
    reviewPassed?: boolean;
  };
}

export type StateStore = Record<string, TaskEntry>;

export async function readState(path: string): Promise<StateStore> {
  try {
    const raw = await readFile(path, 'utf8');
    return JSON.parse(raw) as StateStore;
  } catch (err: unknown) {
    if ((err as NodeJS.ErrnoException).code === 'ENOENT') return {};
    throw err;
  }
}

export async function writeState(
  path: string,
  updater: (current: StateStore) => StateStore
): Promise<void> {
  await mkdir(dirname(path), { recursive: true });

  // Ensure file exists for lockfile
  try {
    await readFile(path);
  } catch {
    await writeFile(path, '{}', 'utf8');
  }

  let release: (() => Promise<void>) | undefined;
  try {
    release = await lockfile.lock(path, { retries: { retries: 5, minTimeout: 100, maxTimeout: 500 } });
    const current = await readState(path);
    const updated = updater(current);
    await writeFile(path, JSON.stringify(updated, null, 2), 'utf8');
  } finally {
    await release?.();
  }
}
SRC
```

- [ ] **Step 4: Run test — expect PASS**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/state.test.ts
```

- [ ] **Step 5: Commit**
```bash
git -C ~/.openclaw/extensions/openclaw-team-plugin add src/team-dispatch/state.ts tests/state.test.ts
git -C ~/.openclaw/extensions/openclaw-team-plugin commit -m "feat: file-locked JSON state store"
```

---

### Task 12: Implement handoff artifact manager

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/handoff.ts`

- [ ] **Step 1: Write handoff.ts**

No complex logic — pure file I/O. No unit test needed beyond integration (covered in Task 18).
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/handoff.ts << 'SRC'
import { readFile, writeFile, mkdir, readdir, rm, stat } from 'fs/promises';
import { join, dirname } from 'path';
import { expandHome } from './paths.js';

export interface HandoffSection {
  agent: string;
  ts: string;
  content: string;
}

export async function appendHandoff(handoffsDir: string, tid: string, section: HandoffSection): Promise<void> {
  const dir = expandHome(handoffsDir);
  await mkdir(dir, { recursive: true });
  const path = join(dir, `${tid}.md`);

  let existing = '';
  try {
    existing = await readFile(path, 'utf8');
  } catch {}

  const newSection = `\n## ${section.agent} — ${section.ts}\n\n${section.content}\n`;
  await writeFile(path, existing + newSection, 'utf8');
}

export async function readHandoff(handoffsDir: string, tid: string): Promise<string> {
  const path = join(expandHome(handoffsDir), `${tid}.md`);
  return readFile(path, 'utf8');
}

/**
 * GC handoff + event files that are older than TTL.
 *
 * Age is measured from `completedAt` in the state store (canonical source of truth),
 * NOT from file mtime. mtime resets on every write, causing GC to miss completed tasks.
 * Falls back to file mtime only for files with no matching TID in the state store.
 */
export async function gcHandoffs(
  handoffsDir: string,
  eventsDir: string,
  statePath: string
): Promise<number> {
  const now = Date.now();
  const ttlMs = 24 * 60 * 60 * 1000; // 24h after task completion
  let deleted = 0;

  // Load state store for completedAt lookups
  let stateStore: Record<string, { completedAt?: string }> = {};
  try {
    const raw = await readFile(statePath, 'utf8');
    stateStore = JSON.parse(raw);
  } catch {
    // State file missing — use mtime as fallback for all files
  }

  const dirs = [expandHome(handoffsDir), expandHome(eventsDir)];
  for (const dir of dirs) {
    let files: string[];
    try {
      files = await readdir(dir);
    } catch {
      continue;
    }
    for (const file of files) {
      // Extract TID from filename (e.g. "20260402-143012-feat-auth-x7k2.md")
      const tid = file.replace(/\.[^.]+$/, '');
      const entry = stateStore[tid];

      let ageMs: number;
      if (entry?.completedAt) {
        ageMs = now - new Date(entry.completedAt).getTime();
      } else {
        // Fallback to mtime if TID not in state (e.g. orphaned files)
        const filePath = join(dir, file);
        const s = await stat(filePath).catch(() => null);
        if (!s) continue;
        ageMs = now - s.mtimeMs;
      }

      if (ageMs > ttlMs) {
        const filePath = join(dir, file);
        await rm(filePath).catch(() => {});
        deleted++;
      }
    }
  }
  return deleted;
}
SRC
```

- [ ] **Step 2: Create paths helper**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/paths.ts << 'SRC'
import { homedir } from 'os';

export function expandHome(p: string): string {
  if (p.startsWith('~/')) return homedir() + p.slice(1);
  return p;
}
SRC
```

- [ ] **Step 3: Commit**
```bash
git -C ~/.openclaw/extensions/openclaw-team-plugin add src/team-dispatch/handoff.ts src/team-dispatch/paths.ts
git -C ~/.openclaw/extensions/openclaw-team-plugin commit -m "feat: handoff artifact manager + paths helper"
```

---

### Task 13: Implement team_dispatch tool

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/tool.ts`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/tests/tool.test.ts`

- [ ] **Step 1: Write failing test**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/tests/tool.test.ts << 'TEST'
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { buildTeamDispatchTool } from '../src/team-dispatch/tool.js';

const mockStateDir = '/tmp/test-state';
const mockHandoffsDir = '/tmp/test-handoffs';
const mockEventsDir = '/tmp/test-events';

describe('buildTeamDispatchTool', () => {
  it('returns a tool with name team_dispatch', () => {
    const tool = buildTeamDispatchTool({ stateDir: mockStateDir, handoffsDir: mockHandoffsDir, eventsDir: mockEventsDir });
    expect(tool.name).toBe('team_dispatch');
  });

  it('tool has required parameters: agentId, task', () => {
    const tool = buildTeamDispatchTool({ stateDir: mockStateDir, handoffsDir: mockHandoffsDir, eventsDir: mockEventsDir });
    const props = tool.parameters.properties;
    expect(props).toHaveProperty('agentId');
    expect(props).toHaveProperty('task');
  });

  it('tool parameters include nodeType with enum', () => {
    const tool = buildTeamDispatchTool({ stateDir: mockStateDir, handoffsDir: mockHandoffsDir, eventsDir: mockEventsDir });
    const nodeType = tool.parameters.properties.nodeType;
    expect(nodeType.enum).toContain('agentic');
    expect(nodeType.enum).toContain('deterministic');
  });

  it('tool parameters include model override', () => {
    const tool = buildTeamDispatchTool({ stateDir: mockStateDir, handoffsDir: mockHandoffsDir, eventsDir: mockEventsDir });
    expect(tool.parameters.properties).toHaveProperty('model');
  });
});
TEST
```

- [ ] **Step 2: Run test — expect FAIL**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/tool.test.ts
```

- [ ] **Step 3: Implement tool.ts**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/team-dispatch/tool.ts << 'SRC'
import { generateTid } from './tid.js';
import { writeState, TaskStatus } from './state.js';
import { appendHandoff } from './handoff.js';
import { join } from 'path';

interface ToolConfig {
  stateDir: string;
  handoffsDir: string;
  eventsDir: string;
}

interface RetryContext {
  failureType: 'context_overflow' | 'wrong_direction' | 'needs_clarification' | 'ci_failure';
  scopeNarrow?: string[];        // File paths to focus on (for context_overflow)
  originalRequirement?: string;  // From handoff artifact (for wrong_direction)
  additionalContext?: string;    // PRD excerpt, customer email, etc. (for needs_clarification)
  previousAttemptTid: string;   // Required — links retry to the failed attempt
}

interface DispatchParams {
  agentId: string;
  task: string;
  pattern?: 'baton' | 'parallel';
  tid?: string;
  priority?: 'l1' | 'l2' | 'l3';
  nodeType?: 'agentic' | 'deterministic';
  escalation_trigger?: Record<string, string>;
  max_cost_usd?: number;
  model?: string;
  retryContext?: RetryContext;   // Populate on retry dispatches only
}

export function buildTeamDispatchTool(config: ToolConfig) {
  return {
    name: 'team_dispatch',
    description: [
      'Dispatch a task to a team member. Generates a TID, creates a handoff artifact, updates state, and triggers the target agent.',
      'Use pattern=baton for sequential relay (sessions_send), pattern=parallel for concurrent tasks (sessions_spawn).',
      'Set nodeType=deterministic for scripted no-LLM steps; use escalation_trigger to promote to agentic on failure.',
      'model= allows rule-based model override (specify only for triggers enumerated in SOUL.md, e.g. architecture review).',
    ].join(' '),
    parameters: {
      type: 'object',
      required: ['agentId', 'task'],
      properties: {
        agentId: { type: 'string', description: 'Target agent ID (research, cto, builder, pm, biz, devops, cio)' },
        task: { type: 'string', description: 'Task description for the target agent' },
        pattern: { type: 'string', enum: ['baton', 'parallel'], default: 'baton' },
        tid: { type: 'string', description: 'Existing TID to continue. Omit to generate new.' },
        priority: { type: 'string', enum: ['l1', 'l2', 'l3'], default: 'l1' },
        nodeType: { type: 'string', enum: ['agentic', 'deterministic'], default: 'agentic' },
        escalation_trigger: { type: 'object', description: 'Condition to promote deterministic → agentic' },
        max_cost_usd: { type: 'number', description: 'Budget cap; auto-escalate L2 when exceeded' },
        model: { type: 'string', description: 'Explicit model override for this dispatch. Rule-based only.' },
        retryContext: {
          type: 'object',
          description: 'Structured context for retry dispatches. Omit on first attempt.',
          properties: {
            failureType: { type: 'string', enum: ['context_overflow', 'wrong_direction', 'needs_clarification', 'ci_failure'] },
            scopeNarrow: { type: 'array', items: { type: 'string' }, description: 'File paths to focus on (for context_overflow)' },
            originalRequirement: { type: 'string' },
            additionalContext: { type: 'string' },
            previousAttemptTid: { type: 'string' },
          },
          required: ['failureType', 'previousAttemptTid'],
        },
      },
    },
    handler: async (params: DispatchParams): Promise<string> => {
      const p = params;
      const tid = p.tid ?? generateTid(p.agentId);
      const now = new Date().toISOString();
      const statePath = join(config.stateDir, 'tasks.json');

      // Update state store
      await writeState(statePath, (s) => ({
        ...s,
        [tid]: {
          status: TaskStatus.InProgress,
          agentId: p.agentId,
          createdAt: now,
          tid,
        },
      }));

      // Append to handoff artifact
      await appendHandoff(config.handoffsDir, tid, {
        agent: 'Q仔→' + p.agentId,
        ts: now,
        content: [
          `**Task:** ${p.task}`,
          `**Pattern:** ${p.pattern ?? 'baton'}`,
          `**Node type:** ${p.nodeType ?? 'agentic'}`,
          p.model ? `**Model override:** ${p.model}` : '',
          p.max_cost_usd ? `**Budget cap:** $${p.max_cost_usd}` : '',
          p.retryContext ? `**Retry:** failureType=${p.retryContext.failureType}` +
            (p.retryContext.previousAttemptTid ? `, prevTid=${p.retryContext.previousAttemptTid}` : '') +
            (p.retryContext.scopeNarrow ? `\n**Scope narrow:** ${p.retryContext.scopeNarrow}` : '') : '',
        ].filter(Boolean).join('\n'),
      });

      return JSON.stringify({
        ok: true,
        tid,
        agentId: p.agentId,
        pattern: p.pattern ?? 'baton',
        message: `Dispatched to ${p.agentId} with TID ${tid}. The plugin will route via sessions_send/spawn.`,
      });
    },
  };
}
SRC
```

- [ ] **Step 4: Run test — expect PASS**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/tool.test.ts
```

- [ ] **Step 5: Commit**
```bash
git -C ~/.openclaw/extensions/openclaw-team-plugin add src/team-dispatch/tool.ts tests/tool.test.ts
git -C ~/.openclaw/extensions/openclaw-team-plugin commit -m "feat: team_dispatch tool implementation"
```

---

### Task 14: Implement soul-bootstrap hook

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/src/hooks/soul-bootstrap.ts`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/tests/soul-bootstrap.test.ts`

This hook fires on `before_agent_start` and injects SOUL.md + IDENTITY.md into the agent's context via `prependContext`.

- [ ] **Step 1: Write failing test**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/tests/soul-bootstrap.test.ts << 'TEST'
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { buildSoulBootstrapHook } from '../src/hooks/soul-bootstrap.js';
import { mkdtemp, writeFile, mkdir, rm } from 'fs/promises';
import { join, tmpdir as osTmpdir } from 'path';
import { tmpdir } from 'os';

let workspacesRoot: string;

beforeEach(async () => {
  workspacesRoot = await mkdtemp(join(tmpdir(), 'soul-test-'));
  // Create workspace-builder with SOUL.md + IDENTITY.md
  await mkdir(join(workspacesRoot, 'workspace-builder'), { recursive: true });
  await writeFile(join(workspacesRoot, 'workspace-builder', 'SOUL.md'), '# 冷燕 SOUL');
  await writeFile(join(workspacesRoot, 'workspace-builder', 'IDENTITY.md'), '# 冷燕 IDENTITY');
});

afterEach(async () => {
  await rm(workspacesRoot, { recursive: true });
});

describe('buildSoulBootstrapHook', () => {
  it('returns prependContext containing SOUL and IDENTITY content for known agent', async () => {
    const hook = buildSoulBootstrapHook({ workspacesRoot });
    const result = await hook(
      { prompt: 'do something', messages: [] },
      { agentId: 'builder' }
    );
    expect(result?.prependContext).toContain('冷燕 SOUL');
    expect(result?.prependContext).toContain('冷燕 IDENTITY');
  });

  it('returns undefined for main agent (no injection needed)', async () => {
    const hook = buildSoulBootstrapHook({ workspacesRoot });
    const result = await hook({ prompt: 'hello' }, { agentId: 'main' });
    expect(result).toBeUndefined();
  });

  it('returns undefined if SOUL.md not found (hard-fail log but no crash)', async () => {
    const hook = buildSoulBootstrapHook({ workspacesRoot });
    const result = await hook({ prompt: '' }, { agentId: 'devops' });
    expect(result).toBeUndefined();
  });
});
TEST
```

- [ ] **Step 2: Run test — expect FAIL**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/soul-bootstrap.test.ts
```

- [ ] **Step 3: Implement soul-bootstrap.ts**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/hooks/soul-bootstrap.ts << 'SRC'
import { readFile } from 'fs/promises';
import { join } from 'path';

interface Config {
  workspacesRoot: string;
}

// Agents that get SOUL injection (all except main which already has its own workspace)
const INJECTABLE_AGENTS = new Set(['research', 'cto', 'builder', 'pm', 'biz', 'devops', 'cio']);

export function buildSoulBootstrapHook(config: Config) {
  return async (
    event: { prompt: string; messages?: unknown[] },
    ctx: { agentId?: string }
  ): Promise<{ prependContext: string } | undefined> => {
    const agentId = ctx.agentId;
    if (!agentId || !INJECTABLE_AGENTS.has(agentId)) return undefined;

    const workspaceDir = join(config.workspacesRoot, `workspace-${agentId}`);

    let soul = '';
    let identity = '';

    try {
      soul = await readFile(join(workspaceDir, 'SOUL.md'), 'utf8');
    } catch {
      console.error(`[openclaw-team-plugin] soul-bootstrap: SOUL.md not found for agent ${agentId} at ${workspaceDir}`);
      return undefined;
    }

    try {
      identity = await readFile(join(workspaceDir, 'IDENTITY.md'), 'utf8');
    } catch {
      // IDENTITY.md optional — log and continue with just SOUL
      console.warn(`[openclaw-team-plugin] soul-bootstrap: IDENTITY.md not found for agent ${agentId}`);
    }

    const prependContext = [
      '<!-- soul-bootstrap: injected by openclaw-team-plugin -->',
      soul,
      identity ? `\n---\n${identity}` : '',
    ].join('\n').trim();

    return { prependContext };
  };
}
SRC
```

- [ ] **Step 4: Run test — expect PASS**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/soul-bootstrap.test.ts
```

- [ ] **Step 5: Commit**
```bash
git -C ~/.openclaw/extensions/openclaw-team-plugin add src/hooks/soul-bootstrap.ts tests/soul-bootstrap.test.ts
git -C ~/.openclaw/extensions/openclaw-team-plugin commit -m "feat: soul-bootstrap hook injects SOUL.md + IDENTITY.md"
```

---

### Task 15: Implement P2P event log hook

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/src/hooks/p2p-event-log.ts`

- [ ] **Step 1: Write failing test for TID extraction**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/tests/p2p-event-log.test.ts << 'TEST'
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { buildP2pEventLogHook } from '../src/hooks/p2p-event-log.js';
import { mkdtemp, rm, readdir, readFile } from 'fs/promises';
import { join } from 'path';
import { tmpdir } from 'os';

let eventsDir: string;

beforeEach(async () => {
  eventsDir = await mkdtemp(join(tmpdir(), 'p2p-test-'));
});

afterEach(async () => {
  await rm(eventsDir, { recursive: true });
});

describe('buildP2pEventLogHook', () => {
  it('ignores non-sessions_send tool calls', async () => {
    const hook = buildP2pEventLogHook({ eventsDir });
    await hook({ toolName: 'read', params: {}, result: 'ok' }, { agentId: 'main' });
    const files = await readdir(eventsDir);
    expect(files).toHaveLength(0);
  });

  it('extracts TID from "TID: <tid>" pattern and writes JSONL', async () => {
    const hook = buildP2pEventLogHook({ eventsDir });
    await hook(
      { toolName: 'sessions_send', params: { target: 'builder', message: 'TID: 20260402-143012-feat-auth-x7k2 please implement' } },
      { agentId: 'main' }
    );
    const files = await readdir(eventsDir);
    expect(files).toEqual(['20260402-143012-feat-auth-x7k2.jsonl']);
    const content = await readFile(join(eventsDir, files[0]), 'utf8');
    const record = JSON.parse(content.trim());
    expect(record.from).toBe('main');
    expect(record.to).toBe('builder');
  });

  it('extracts TID from bare TID format in message', async () => {
    const hook = buildP2pEventLogHook({ eventsDir });
    await hook(
      { toolName: 'sessions_send', params: { target: 'cto', message: 'QA_PASS: 20260402-150000-fix-bug-ab12 done' } },
      { agentId: 'builder' }
    );
    const files = await readdir(eventsDir);
    expect(files).toEqual(['20260402-150000-fix-bug-ab12.jsonl']);
  });

  it('falls back to unknown TID when no TID in message', async () => {
    const hook = buildP2pEventLogHook({ eventsDir });
    await hook(
      { toolName: 'sessions_send', params: { target: 'main', message: 'hello no tid here' } },
      { agentId: 'research' }
    );
    const files = await readdir(eventsDir);
    expect(files).toEqual(['unknown.jsonl']);
  });
});
TEST
```

- [ ] **Step 2: Run test — expect FAIL**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/p2p-event-log.test.ts
```
Expected: fails with "Cannot find module".

- [ ] **Step 3: Implement p2p-event-log.ts**

This is a simple `after_tool_call` hook. No blocking. Append to JSONL.
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/hooks/p2p-event-log.ts << 'SRC'
import { appendFile, mkdir } from 'fs/promises';
import { join, dirname } from 'path';
import { expandHome } from '../team-dispatch/paths.js';

interface Config {
  eventsDir: string;
}

const SUMMARY_MAX = 200;

export function buildP2pEventLogHook(config: Config) {
  return async (
    event: { toolName: string; params: Record<string, unknown>; result?: unknown; error?: string },
    ctx: { agentId?: string }
  ): Promise<void> => {
    if (event.toolName !== 'sessions_send') return;

    const params = event.params;
    const to = String(params.target ?? params.sessionKey ?? 'unknown');
    const from = ctx.agentId ?? 'unknown';

    // Only log P2P (non-Q仔-directed) messages
    // Q仔-directed callbacks are handled separately; we log all sends for visibility
    const message = String(params.message ?? params.content ?? '');
    const summary = message.slice(0, SUMMARY_MAX);

    // Extract TID from message if present
    const tidMatch = message.match(/\bTID[:\s]?([0-9]{8}-[0-9]{6}-[^\s]+)/i) ??
                     message.match(/\b([0-9]{8}-[0-9]{6}-\w+-[a-z0-9]{4})\b/);
    const tid = tidMatch?.[1] ?? 'unknown';

    const eventsDir = expandHome(config.eventsDir);
    await mkdir(eventsDir, { recursive: true });
    const eventPath = join(eventsDir, `${tid}.jsonl`);

    const record = JSON.stringify({ ts: new Date().toISOString(), from, to, summary }) + '\n';
    await appendFile(eventPath, record, 'utf8');
  };
}
SRC
```

- [ ] **Step 4: Run test — expect PASS**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/p2p-event-log.test.ts
```
Expected: all tests green.

- [ ] **Step 5: Commit**
```bash
git -C ~/.openclaw/extensions/openclaw-team-plugin add src/hooks/p2p-event-log.ts tests/p2p-event-log.test.ts
git -C ~/.openclaw/extensions/openclaw-team-plugin commit -m "feat: P2P event log hook + TID extraction tests"
```

---

### Task 16: Implement ACP enforcement hook

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/src/hooks/acp-enforce.ts`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/tests/acp-enforce.test.ts`

- [ ] **Step 1: Write failing test**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/tests/acp-enforce.test.ts << 'TEST'
import { describe, it, expect } from 'vitest';
import { buildAcpEnforceHook } from '../src/hooks/acp-enforce.js';

const hook = buildAcpEnforceHook();

describe('buildAcpEnforceHook', () => {
  it('blocks write tool from builder in non-ACP session', async () => {
    const result = await hook(
      { toolName: 'write', params: { path: 'src/foo.ts' } },
      { agentId: 'builder', sessionKey: 'agent:builder:slack:channel:C123:thread:111' }
    );
    expect(result?.block).toBe(true);
    expect(result?.blockReason).toContain('ACP session');
  });

  it('blocks exec tool from devops in non-ACP session', async () => {
    const result = await hook(
      { toolName: 'exec', params: { command: 'npm run build' } },
      { agentId: 'devops', sessionKey: 'agent:devops:slack:channel:C456:thread:222' }
    );
    expect(result?.block).toBe(true);
  });

  it('allows write tool from builder in ACP session', async () => {
    const result = await hook(
      { toolName: 'write', params: {} },
      { agentId: 'builder', sessionKey: 'agent:builder:acp:session:abc123' }
    );
    expect(result?.block).toBeFalsy();
  });

  it('does not block non-write tools from builder', async () => {
    const result = await hook(
      { toolName: 'read', params: {} },
      { agentId: 'builder', sessionKey: 'agent:builder:slack:channel:C123:thread:111' }
    );
    expect(result?.block).toBeFalsy();
  });

  it('does not block write tools from main agent', async () => {
    const result = await hook(
      { toolName: 'write', params: {} },
      { agentId: 'main', sessionKey: 'agent:main:slack:channel:C123:thread:111' }
    );
    expect(result?.block).toBeFalsy();
  });
});
TEST
```

- [ ] **Step 2: Run test — expect FAIL**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/acp-enforce.test.ts
```

- [ ] **Step 3: Implement acp-enforce.ts**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/hooks/acp-enforce.ts << 'SRC'
const ACP_ENFORCED_AGENTS = new Set(['builder', 'devops']);
const BLOCKED_TOOLS = new Set(['write', 'edit', 'exec', 'bash']);

function isAcpSession(sessionKey?: string): boolean {
  if (!sessionKey) return false;
  // ACP sessions contain ':acp:' in the session key
  return sessionKey.includes(':acp:');
}

export function buildAcpEnforceHook() {
  return async (
    event: { toolName: string; params: Record<string, unknown> },
    ctx: { agentId?: string; sessionKey?: string }
  ): Promise<{ block: boolean; blockReason: string } | undefined> => {
    const agentId = ctx.agentId;
    if (!agentId || !ACP_ENFORCED_AGENTS.has(agentId)) return undefined;
    if (!BLOCKED_TOOLS.has(event.toolName)) return undefined;
    if (isAcpSession(ctx.sessionKey)) return undefined;

    return {
      block: true,
      blockReason: `[BLOCKED] Tool "${event.toolName}" requires ACP session for agent "${agentId}". ` +
        `Use sessions_spawn with runtime: acp, or ensure you are running inside tmux builder-acp / devops-acp.`,
    };
  };
}
SRC
```

- [ ] **Step 4: Run test — expect PASS**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/acp-enforce.test.ts
```

- [ ] **Step 5: Commit**
```bash
git -C ~/.openclaw/extensions/openclaw-team-plugin add src/hooks/acp-enforce.ts tests/acp-enforce.test.ts
git -C ~/.openclaw/extensions/openclaw-team-plugin commit -m "feat: ACP enforcement hook for builder + devops"
```

---

### Task 17: Implement orchestrator soft-warning hook

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/src/hooks/orchestrator-warn.ts`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/tests/orchestrator-warn.test.ts`

Strategy: `after_tool_call` increments a counter in a shared in-memory map (keyed by TID or runId). The `before_agent_start` hook reads the counter and prepends a warning if > 0.

- [ ] **Step 1: Write failing test**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/tests/orchestrator-warn.test.ts << 'TEST'
import { describe, it, expect, beforeEach } from 'vitest';
import { buildOrchestratorWarnHooks } from '../src/hooks/orchestrator-warn.js';

describe('orchestrator violation tracking', () => {
  let hooks: ReturnType<typeof buildOrchestratorWarnHooks>;

  beforeEach(() => {
    hooks = buildOrchestratorWarnHooks();
  });

  it('after_tool_call does not count violations from non-main agents', async () => {
    await hooks.afterToolCall(
      { toolName: 'write', params: {}, runId: 'run-1' },
      { agentId: 'builder' }
    );
    const result = await hooks.beforeAgentStart({ prompt: 'hi' }, { agentId: 'main', runId: 'run-1' });
    expect(result?.prependContext).toBeUndefined();
  });

  it('after_tool_call counts write from main agent', async () => {
    await hooks.afterToolCall(
      { toolName: 'write', params: {}, runId: 'run-2' },
      { agentId: 'main' }
    );
    const result = await hooks.beforeAgentStart({ prompt: 'hi' }, { agentId: 'main', runId: 'run-2' });
    expect(result?.prependContext).toContain('HARNESS WARN');
    expect(result?.prependContext).toContain('Violation count: 1');
  });

  it('warning escalates to L2 notice at N=3', async () => {
    for (let i = 0; i < 3; i++) {
      await hooks.afterToolCall(
        { toolName: 'edit', params: {}, runId: 'run-3' },
        { agentId: 'main' }
      );
    }
    const result = await hooks.beforeAgentStart({ prompt: 'hi' }, { agentId: 'main', runId: 'run-3' });
    expect(result?.prependContext).toContain('L2');
  });
});
TEST
```

- [ ] **Step 2: Run test — expect FAIL**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/orchestrator-warn.test.ts
```

- [ ] **Step 3: Implement orchestrator-warn.ts**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/hooks/orchestrator-warn.ts << 'SRC'
const IMPL_TOOLS = new Set(['write', 'edit', 'exec', 'bash']);
// In-memory counter: runId → violation count
const violationCounters = new Map<string, number>();

export function buildOrchestratorWarnHooks() {
  const afterToolCall = async (
    event: { toolName: string; params: Record<string, unknown>; runId?: string },
    ctx: { agentId?: string }
  ): Promise<void> => {
    if (ctx.agentId !== 'main') return;
    if (!IMPL_TOOLS.has(event.toolName)) return;
    const key = event.runId ?? 'unknown';
    violationCounters.set(key, (violationCounters.get(key) ?? 0) + 1);
  };

  const beforeAgentStart = async (
    event: { prompt: string },
    ctx: { agentId?: string; runId?: string }
  ): Promise<{ prependContext: string } | undefined> => {
    if (ctx.agentId !== 'main') return undefined;
    const key = ctx.runId ?? 'unknown';
    const count = violationCounters.get(key) ?? 0;
    if (count === 0) return undefined;

    const escalation = count >= 3 ? ' ⚠️ L2 ESCALATION: notify human — Q仔 is bypassing team structure.' : '';
    const warn = [
      `[HARNESS WARN] Q仔 called implementation tools directly ${count} time(s) this run.`,
      `Route implementation work through team_dispatch → builder/devops.`,
      `Violation count: ${count}.${escalation}`,
    ].join(' ');

    return { prependContext: warn };
  };

  return { afterToolCall, beforeAgentStart };
}
SRC
```

- [ ] **Step 4: Run test — expect PASS**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/orchestrator-warn.test.ts
```

- [ ] **Step 5: Commit**
```bash
git -C ~/.openclaw/extensions/openclaw-team-plugin add src/hooks/orchestrator-warn.ts tests/orchestrator-warn.test.ts
git -C ~/.openclaw/extensions/openclaw-team-plugin commit -m "feat: orchestrator soft-warning hook (N=3 → L2)"
```

---

### Task 18: Implement GC service + wire all pieces in index.ts

**Files:**
- Create: `~/.openclaw/extensions/openclaw-team-plugin/src/gc/gc-service.ts`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/tests/gc-service.test.ts`
- Create: `~/.openclaw/extensions/openclaw-team-plugin/index.ts`

- [ ] **Step 1: Write failing GC test**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/tests/gc-service.test.ts << 'TEST'
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import { gcHandoffs } from '../src/team-dispatch/handoff.js';
import { mkdtemp, rm, writeFile, mkdir } from 'fs/promises';
import { join } from 'path';
import { tmpdir } from 'os';

let handoffsDir: string;
let eventsDir: string;
let stateDir: string;

beforeEach(async () => {
  const base = await mkdtemp(join(tmpdir(), 'gc-test-'));
  handoffsDir = join(base, 'handoffs');
  eventsDir = join(base, 'events');
  stateDir = base;
  await mkdir(handoffsDir, { recursive: true });
  await mkdir(eventsDir, { recursive: true });
});

afterEach(async () => {
  await rm(stateDir, { recursive: true });
});

describe('gcHandoffs', () => {
  it('deletes files whose completedAt is older than 24h', async () => {
    const tid = '20260401-000000-old-task-aaaa';
    const statePath = join(stateDir, 'tasks.json');
    // completedAt > 24h ago
    const old = new Date(Date.now() - 25 * 60 * 60 * 1000).toISOString();
    await writeFile(statePath, JSON.stringify({ [tid]: { status: 'done', agentId: 'builder', createdAt: old, completedAt: old } }));
    await writeFile(join(handoffsDir, `${tid}.md`), '# test');

    const deleted = await gcHandoffs(handoffsDir, eventsDir, statePath);
    expect(deleted).toBe(1);
  });

  it('keeps files whose completedAt is less than 24h ago', async () => {
    const tid = '20260402-120000-new-task-bbbb';
    const statePath = join(stateDir, 'tasks.json');
    const recent = new Date(Date.now() - 1 * 60 * 60 * 1000).toISOString(); // 1h ago
    await writeFile(statePath, JSON.stringify({ [tid]: { status: 'done', agentId: 'cto', createdAt: recent, completedAt: recent } }));
    await writeFile(join(handoffsDir, `${tid}.md`), '# test');

    const deleted = await gcHandoffs(handoffsDir, eventsDir, statePath);
    expect(deleted).toBe(0);
  });

  it('uses mtime fallback for orphaned files not in state store', async () => {
    const statePath = join(stateDir, 'tasks.json');
    await writeFile(statePath, JSON.stringify({})); // empty state
    // Write a file and manually set mtime via atime trick is not reliable in tests,
    // so just verify it does not crash and returns 0 for a fresh file.
    const orphanFile = join(handoffsDir, 'unknown-orphan.md');
    await writeFile(orphanFile, '# orphan');

    const deleted = await gcHandoffs(handoffsDir, eventsDir, statePath);
    expect(deleted).toBe(0); // just created, mtime is recent
  });
});
TEST
```

- [ ] **Step 2: Run GC test — expect FAIL**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/gc-service.test.ts
```
Expected: fails — gcHandoffs signature does not yet accept statePath.

- [ ] **Step 3: Implement gc-service.ts**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/src/gc/gc-service.ts << 'SRC'
import cron from 'node-cron';
import { gcHandoffs } from '../team-dispatch/handoff.js';
import { join } from 'path';

interface GcConfig {
  handoffsDir: string;
  eventsDir: string;
  stateDir: string;
  // workspacesRoot is derived as stateDir/.. (= ~/.openclaw) — no separate field needed
}

// Entropy checks: deterministic scans folded into the same daily cron (§6.10)
async function runEntropyChecks(
  workspacesRoot: string,
  logger: { info: (m: string) => void }
): Promise<void> {
  const { readdir, readFile, stat } = await import('fs/promises');

  // 1. Scan for SOUL.md files exceeding 80 lines
  let soulmdsOver80 = 0;
  try {
    const entries = await readdir(workspacesRoot);
    for (const entry of entries) {
      if (!entry.startsWith('workspace')) continue;
      const soulPath = join(workspacesRoot, entry, 'SOUL.md');
      try {
        const content = await readFile(soulPath, 'utf8');
        const lines = content.split('\n').length;
        if (lines > 80) {
          soulmdsOver80++;
          logger.info(`[openclaw-team-plugin] GC entropy: ${entry}/SOUL.md has ${lines} lines (>80 — candidate for RULES/ split)`);
        }
      } catch { /* SOUL.md missing for this workspace, skip */ }
    }
  } catch { /* workspacesRoot missing, skip */ }

  // 2. Count stale worktrees (directories >7 days old with no recent git activity)
  let staleWorktrees = 0;
  const worktreesDir = join(workspacesRoot, '..', 'worktrees');
  try {
    const trees = await readdir(worktreesDir);
    const now = Date.now();
    for (const tree of trees) {
      const treePath = join(worktreesDir, tree);
      const s = await stat(treePath).catch(() => null);
      if (s && (now - s.mtimeMs) > 7 * 24 * 60 * 60 * 1000) {
        staleWorktrees++;
        logger.info(`[openclaw-team-plugin] GC entropy: stale worktree ${tree} (>7 days, no activity)`);
      }
    }
  } catch { /* worktrees dir missing, skip */ }

  logger.info(`[openclaw-team-plugin] GC entropy summary: soulmds_over_80=${soulmdsOver80}, stale_worktrees=${staleWorktrees}`);
}

export function buildGcService(config: GcConfig) {
  return {
    id: 'openclaw-team-gc',
    start: async (ctx: { logger: { info: (m: string) => void } }) => {
      const statePath = join(config.stateDir, 'tasks.json');
      const workspacesRoot = join(config.stateDir, '..'); // ~/.openclaw

      // Run daily at 03:00 local time
      cron.schedule('0 3 * * *', async () => {
        ctx.logger.info('[openclaw-team-plugin] GC: starting daily cleanup');
        const deleted = await gcHandoffs(config.handoffsDir, config.eventsDir, statePath);
        ctx.logger.info(`[openclaw-team-plugin] GC: deleted ${deleted} expired artifacts`);
        // Entropy checks folded into same cron (§6.10 — no separate agent needed)
        await runEntropyChecks(workspacesRoot, ctx.logger);
      });
      ctx.logger.info('[openclaw-team-plugin] GC service started (daily 03:00)');
    },
  };
}
SRC
```

- [ ] **Step 4: Run GC test — expect PASS**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run tests/gc-service.test.ts
```

- [ ] **Step 5: Write index.ts**
```bash
cat > ~/.openclaw/extensions/openclaw-team-plugin/index.ts << 'SRC'
import type { OpenClawPluginApi } from 'openclaw/plugin-sdk/plugin-entry';
import { homedir } from 'os';
import { join } from 'path';

import { buildTeamDispatchTool } from './src/team-dispatch/tool.js';
import { buildSoulBootstrapHook } from './src/hooks/soul-bootstrap.js';
import { buildP2pEventLogHook } from './src/hooks/p2p-event-log.js';
import { buildAcpEnforceHook } from './src/hooks/acp-enforce.js';
import { buildOrchestratorWarnHooks } from './src/hooks/orchestrator-warn.js';
import { buildGcService } from './src/gc/gc-service.js';

const HOME = homedir();
const DEFAULTS = {
  handoffsDir: join(HOME, '.openclaw', 'shared', 'handoffs'),
  eventsDir: join(HOME, '.openclaw', 'shared', 'events'),
  worktreesDir: join(HOME, '.openclaw', 'worktrees'),
  workspacesRoot: join(HOME, '.openclaw'),
  stateDir: join(HOME, '.openclaw', 'shared'),
};

export default {
  id: 'openclaw-team-plugin',
  name: 'OpenClaw Team Plugin',
  description: 'team_dispatch, SOUL injection, ACP enforcement, P2P event log, orchestrator warnings, GC',

  register(api: OpenClawPluginApi) {
    const cfg = (api as any).config?.plugins?.entries?.['openclaw-team-plugin']?.config ?? {};
    const handoffsDir = cfg.handoffsDir ?? DEFAULTS.handoffsDir;
    const eventsDir = cfg.eventsDir ?? DEFAULTS.eventsDir;
    const stateDir = cfg.stateDir ?? DEFAULTS.stateDir;
    const workspacesRoot = DEFAULTS.workspacesRoot;

    // 1. team_dispatch tool
    const dispatchTool = buildTeamDispatchTool({ stateDir, handoffsDir, eventsDir });
    api.registerTool(dispatchTool as any);

    // 2+5. Combine SOUL injection + orchestrator warning into ONE before_agent_start handler.
    // Two separate hooks both returning prependContext may cause openclaw to use only the last one.
    // Merging them ensures both prependContext values are always concatenated.
    const soulHook = buildSoulBootstrapHook({ workspacesRoot });
    const warnHooks = buildOrchestratorWarnHooks();
    api.registerHook('before_agent_start', async (event: any, ctx: any) => {
      const [soulResult, warnResult] = await Promise.all([
        soulHook(event, ctx),
        warnHooks.beforeAgentStart(event, ctx),
      ]);
      const parts = [soulResult?.prependContext, warnResult?.prependContext].filter(Boolean);
      if (parts.length === 0) return undefined;
      return { prependContext: parts.join('\n\n---\n\n') };
    });

    // 3. P2P event log (after sessions_send calls)
    const p2pHook = buildP2pEventLogHook({ eventsDir });
    api.registerHook('after_tool_call', p2pHook as any);

    // 4. ACP enforcement (block write/edit/exec from builder/devops outside ACP)
    const acpHook = buildAcpEnforceHook();
    api.registerHook('before_tool_call', acpHook as any);

    // 5b. Orchestrator violation counter (after_tool_call side — separate from before_agent_start)
    api.registerHook('after_tool_call', warnHooks.afterToolCall as any);

    // 6. GC service (daily 03:00)
    const gcService = buildGcService({ handoffsDir, eventsDir, stateDir });
    api.registerService(gcService as any);

    (api as any).logger?.info?.('[openclaw-team-plugin] Registered: team_dispatch + 4 hooks + GC service');
  },
};
SRC
```

- [ ] **Step 6: Run all tests**
```bash
cd ~/.openclaw/extensions/openclaw-team-plugin && npx vitest run --coverage
```
Expected: all test files pass, coverage ≥80%.

- [ ] **Step 7: Commit**
```bash
git -C ~/.openclaw/extensions/openclaw-team-plugin add src/gc/gc-service.ts tests/gc-service.test.ts index.ts
git -C ~/.openclaw/extensions/openclaw-team-plugin commit -m "feat: GC service (uses completedAt from state) + wire all hooks in index.ts"
```

---

### Task 19: Install plugin in openclaw

**Files:**
- Modify: `~/.openclaw/openclaw.json` (add plugin to allow list + entries)

> **Gateway must be stopped before mutation** (same rule as Task 7). Use atomic write.

- [ ] **Step 1: Stop gateway**
```bash
openclaw stop && sleep 2
echo "Gateway stopped"
```

- [ ] **Step 2: Add plugin to openclaw.json (atomic write)**
```bash
python3 << 'SCRIPT'
import json, os, tempfile

path = '/Users/qqzhang/.openclaw/openclaw.json'
with open(path, 'r') as f:
    d = json.load(f)

plugins = d.setdefault('plugins', {})

# Add to allow list
allow = plugins.setdefault('allow', [])
if 'openclaw-team-plugin' not in allow:
    allow.append('openclaw-team-plugin')

# Add to entries
entries = plugins.setdefault('entries', {})
entries['openclaw-team-plugin'] = {
    'enabled': True,
    'path': '/Users/qqzhang/.openclaw/extensions/openclaw-team-plugin',
    'config': {
        'handoffsDir': '/Users/qqzhang/.openclaw/shared/handoffs',
        'eventsDir': '/Users/qqzhang/.openclaw/shared/events'
    }
}

# Atomic write: write to temp then rename (POSIX atomic)
dir_ = os.path.dirname(path)
with tempfile.NamedTemporaryFile('w', dir=dir_, delete=False, suffix='.tmp') as tf:
    json.dump(d, tf, indent=2, ensure_ascii=False)
    tf.write('\n')
    tmp_path = tf.name
os.rename(tmp_path, path)

print('Plugin registered in openclaw.json (atomic write)')
SCRIPT
```

- [ ] **Step 3: Validate JSON**
```bash
python3 -m json.tool ~/.openclaw/openclaw.json > /dev/null && echo "JSON valid"
```

- [ ] **Step 4: Restart gateway and verify plugin loads**
```bash
openclaw start
sleep 3
openclaw status 2>/dev/null | grep -i "team-plugin\|team_dispatch" || \
  grep -i "team-plugin\|registered" ~/.openclaw/logs/*.log 2>/dev/null | tail -5 || \
  echo "Check ~/.openclaw/logs/ for plugin registration confirmation"
```

Expected: log line containing `[openclaw-team-plugin] Registered` or similar.

- [ ] **Step 5: Verify team_dispatch tool is available in Q仔 session**

Open Q仔 session and run:
```
/tools
```
or ask Q仔: "list your available tools" — `team_dispatch` should appear.

---

## Phase 3 — Skills + Protocol Docs

### Task 20: Create team_orchestrate skill for Q仔

**Files:**
- Create: `~/.openclaw/workspace/skills/team-orchestrate/SKILL.md`

- [ ] **Step 1: Write SKILL.md**
```bash
mkdir -p ~/.openclaw/workspace/skills/team-orchestrate
cat > ~/.openclaw/workspace/skills/team-orchestrate/SKILL.md << 'SKILL'
---
name: team-orchestrate
version: 1.0.0
description: "Q仔 goal decomposition and autonomous team dispatch. Activates the full pipeline: Research → PM → CTO → Builder → DevOps → Acceptance."
---

# Team Orchestrate

Activate when Q仔 receives a multi-step product or engineering goal.

## When to Use

Invoke this skill when:
- User requests a new feature, product, or engineering task
- The task requires more than one agent (e.g., requires both PM and Builder)
- Any task that will produce a GitHub PR as output

**Do NOT invoke for:** Single-agent tasks (pure research, pure biz copy, pure analysis).

## Dispatch Sequence

```
1. [Optional] Dispatch research (陆小凤) for feasibility / competitive analysis
2. Dispatch pm (花满楼) → PRD + GitHub commit
3. Dispatch cto (李寻欢) → architecture review [max 2 rounds with pm]
4. Dispatch builder (冷燕) → implementation + PR [ACP only, ≥80% coverage]
5. Dispatch cto (李寻欢) → QA gate [max 3 rounds with builder]
6. Dispatch devops (傅红雪) → produce infra configs + PR
7. Dispatch cto (李寻欢) → infra review
8. Dispatch devops (傅红雪) → execute deployment [after INFRA_REVIEW_OK]
9. Dispatch pm (花满楼) → acceptance sign-off
10. Mark delivery complete, notify user
```

## Rules

1. **Always use team_dispatch** — never call sessions_send/spawn directly for team operations.
2. **One TID per goal** — reuse the same TID across all pipeline stages for the same feature.
3. **Wait for callbacks** — do not proceed to the next stage until the current stage's callback arrives.
4. **Escalation matrix:**
   - L1 (autonomous): routine dispatch, soft failure re-dispatch
   - L2 (notify, then continue): 李寻欢 unresponsive 3x, non-critical QA failure
   - L3 (block for human): production deploy, secret rotation, PRD loop > 2 rounds, architecture changes affecting multiple repos
5. **No direct implementation** — Q仔 routes all code/infra work through builder/devops. If you feel the urge to write code directly, use team_dispatch instead.
6. **GitHub = done** — no artifact is complete until it exists as a GitHub PR or committed doc.

## Goal Decomposition Template

When receiving a goal, write to SESSION-STATE.md:
```
Objective: <one sentence>
Non-goals: <what this does NOT include>
Done-when: <verifiable completion criteria>
Pipeline: <list of dispatch steps planned>
TID: <generated TID>
```
SKILL
```

- [ ] **Step 2: Verify**
```bash
cat ~/.openclaw/workspace/skills/team-orchestrate/SKILL.md | head -10
```

- [ ] **Step 3: Commit to openclaw workspace git**
```bash
cd ~/.openclaw/workspace && git add skills/team-orchestrate/SKILL.md && \
  git commit -m "feat: add team-orchestrate skill for Q仔 autonomous dispatch" 2>/dev/null || \
  echo "No git in workspace — file saved successfully"
```

---

### Task 21: Update A2A_PROTOCOL.md, SYSTEM_RULES.md, and SUBAGENT_PACKET_TEMPLATE.md

**Files:**
- Modify: `~/.openclaw/shared/A2A_PROTOCOL.md`
- Modify: `~/.openclaw/shared/SYSTEM_RULES.md`
- Modify: `~/.openclaw/shared/SUBAGENT_PACKET_TEMPLATE.md`

- [ ] **Step 1: Update A2A_PROTOCOL.md status from DRAFT to v3 FINAL**
```bash
sed -i '' 's/状态：DRAFT（待 Master 审核）/状态：v3 FINAL（已审核，2026-04-02）/' \
  ~/.openclaw/shared/A2A_PROTOCOL.md
grep "状态" ~/.openclaw/shared/A2A_PROTOCOL.md | head -3
```

- [ ] **Step 2: Append team_dispatch section to A2A_PROTOCOL.md**
```bash
cat >> ~/.openclaw/shared/A2A_PROTOCOL.md << 'APPEND'

---

## 5) team_dispatch（推荐的 A2A 调用方式）

从 2026-04-02 起，Q仔 的所有 A2A dispatch 应优先使用 `team_dispatch` 工具（来自 openclaw-team-plugin），而非直接调用 `sessions_send` 或 `sessions_spawn`。

`team_dispatch` 封装了：
- TID 生成与幂等性保障
- Handoff artifact 创建
- 状态存储（file-locked JSON）
- Callback 注册
- Budget cap（max_cost_usd）

直接使用 `sessions_send` 仍允许用于：
- 回调（agent → Q仔）
- P2P 协作中非 dispatch 性质的消息

`team_dispatch` 取代的只是"Q仔 → 子 agent 的派单动作"，不取代 callback 机制。
APPEND
```

- [ ] **Step 3: Update SYSTEM_RULES.md — replace a2a-dispatcher reference with team_dispatch**
```bash
sed -i '' 's/a2a-dispatcher.*skill/team_dispatch tool (from openclaw-team-plugin)/g' \
  ~/.openclaw/shared/SYSTEM_RULES.md
grep -n "team_dispatch\|a2a-dispatcher" ~/.openclaw/shared/SYSTEM_RULES.md | head -5
```

- [ ] **Step 4: Update SUBAGENT_PACKET_TEMPLATE.md — add TID + callback fields**

Read current template first to find the append point:
```bash
tail -20 ~/.openclaw/shared/SUBAGENT_PACKET_TEMPLATE.md
```
Then append:
```bash
cat >> ~/.openclaw/shared/SUBAGENT_PACKET_TEMPLATE.md << 'APPEND'

---

## TID + Callback Fields（openclaw-team-plugin v1+）

Every packet dispatched via `team_dispatch` MUST include:

```yaml
tid: <YYYYMMDD-HHMMss-<tag>-<4chars>>   # Idempotency key; reuse across pipeline stages for the same feature
callback_to: main                        # Always report completion back to Q仔
callback_format: "<STATUS>: <TID> [details]"
  # STATUS options:
  #   QA_PASS / QA_FAIL         (李寻欢 → Q仔)
  #   INFRA_READY / INFRA_REVIEW_OK / INFRA_REVIEW_FAIL  (傅红雪 ↔ Q仔)
  #   DEPLOY_OK / DEPLOY_FAIL   (傅红雪 → Q仔)
  #   ACCEPT / REJECT           (花满楼 → Q仔)
  #   ROLLBACK_OK               (傅红雪 → Q仔)
ack_required: false  # Only set true for DEPLOY_OK / INFRA_REVIEW_OK / ROLLBACK_OK (state-changing)
```
APPEND
```

- [ ] **Step 5: Verify all three files are valid**
```bash
wc -l ~/.openclaw/shared/A2A_PROTOCOL.md ~/.openclaw/shared/SYSTEM_RULES.md ~/.openclaw/shared/SUBAGENT_PACKET_TEMPLATE.md
```

---

## Phase 4 — Integration Testing

### Task 22: End-to-end smoke test

This task verifies the full plugin stack is running. Do this with Q仔's session open.

- [ ] **Step 1: Verify team_dispatch appears in Q仔's tool list**

In Q仔 session:
```
/tools
```
Expected: `team_dispatch` listed.

- [ ] **Step 2: Test TID generation via team_dispatch**

Ask Q仔:
```
Use team_dispatch to dispatch a test task to research with task="Hello from integration test" and return the TID.
```
Expected: Q仔 responds with a TID like `20260402-XXXXXX-research-XXXX`.

- [ ] **Step 3: Verify handoff artifact was created**
```bash
ls ~/.openclaw/shared/handoffs/
cat ~/.openclaw/shared/handoffs/*.md | head -20
```
Expected: a `.md` file exists with content from the dispatch.

- [ ] **Step 4: Verify state store was updated**
```bash
cat ~/.openclaw/shared/tasks.json | python3 -m json.tool | head -30
```
Expected: a JSON object with the TID key, status `in_progress`.

---

### Task 23: Test ACP enforcement

- [ ] **Step 1: Open builder (冷燕) session directly**

```bash
openclaw session --agent builder
```

- [ ] **Step 2: Attempt a write tool call outside ACP**

In the builder session, try:
```
Write a file test.txt with content "hello"
```
Expected: blocked with `[BLOCKED] Tool "write" requires ACP session for agent "builder"`.

- [ ] **Step 3: Verify read tool is not blocked**

In the builder session:
```
Read the file ~/.openclaw/workspace-builder/SOUL.md
```
Expected: file content returned (not blocked).

---

### Task 24: Test soul-bootstrap injection

- [ ] **Step 1: Spawn a fresh cto session**
```bash
openclaw session --agent cto --new
```

- [ ] **Step 2: Ask the agent who it is**

```
Who are you? What are your responsibilities?
```
Expected: responds as 李寻欢, mentions QA gate, architecture review — content from the injected SOUL.md.

- [ ] **Step 3: Verify the QA gate section is visible**

Ask:
```
What categories must your QA subagent cover?
```
Expected: mentions 边界条件, 错误处理, 输入校验, 并发安全, 安全漏洞.

---

### Task 25: Test orchestrator soft-warning

- [ ] **Step 1: In Q仔 session, attempt a direct write**

```
Write a file /tmp/test-orch-warn.txt with content "testing orchestrator warn"
```
Expected: write executes but Q仔 receives a prepended warning on the next turn: `[HARNESS WARN] Q仔 called write directly...`

- [ ] **Step 2: Repeat twice more (3 total) and verify L2 notice**

Do two more direct write/edit calls. On the 4th turn, the warning should include `L2 ESCALATION`.

- [ ] **Step 3: Clean up test file**
```bash
rm -f /tmp/test-orch-warn.txt
```

---

### Task 26: Create monitoring heartbeat + system cron

This implements Section 2.6 of the spec: a zero-LLM-cost shell script that runs every 10 minutes and writes `monitor-status.json`. It does NOT spawn any agent — it just checks process/session liveness.

**Files:**
- Create: `~/.openclaw/bin/check-agents.sh`
- Modify: user's system crontab (once, during setup)

- [ ] **Step 1: Create ~/.openclaw/bin directory**
```bash
mkdir -p ~/.openclaw/bin
```

- [ ] **Step 2: Write check-agents.sh**
```bash
cat > ~/.openclaw/bin/check-agents.sh << 'SCRIPT'
#!/usr/bin/env bash
# check-agents.sh — Zero-LLM-cost monitoring heartbeat (spec §2.6)
# Runs every 10 minutes via system cron. Writes monitor-status.json.
# Does NOT spawn any LLM agent. Pure shell/process/GitHub API checks.
#
# Alert routing:
#   - Gateway alive: alert via "openclaw sessions send main <msg>" CLI
#   - Gateway dead:  alert via macOS osascript notification
#   - ntfy fallback: if OPENCLAW_NTFY_TOPIC env is set, also push to ntfy

set -euo pipefail

OUTPUT="$HOME/.openclaw/shared/monitor-status.json"
TASKS_FILE="$HOME/.openclaw/shared/tasks.json"
mkdir -p "$(dirname "$OUTPUT")" "$HOME/.openclaw/logs"

ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)
alerts=()   # Collect alert messages; sent at end

# ── 1. Gateway liveness ──────────────────────────────────────────────────────
gateway_pid=$(pgrep -f "openclaw" 2>/dev/null | head -1 || true)
gateway_ok=$([ -n "$gateway_pid" ] && echo "true" || echo "false")
[ "$gateway_ok" = "false" ] && alerts+=("ALERT: openclaw gateway is DOWN at $ts")

# ── 2. ACP tmux sessions ─────────────────────────────────────────────────────
builder_acp=$(tmux has-session -t builder-acp 2>/dev/null && echo "true" || echo "false")
devops_acp=$(tmux has-session -t devops-acp 2>/dev/null && echo "true" || echo "false")

# ── 3. Open PR status (gh CLI) ───────────────────────────────────────────────
# Check each open TID's PR for CI failures
pr_failures=()
if [ -f "$TASKS_FILE" ] && command -v gh &>/dev/null; then
  open_tids=$(python3 -c "
import json
with open('$TASKS_FILE') as f:
  d = json.load(f)
for tid, v in d.items():
  if v.get('status') in ('pending', 'in_progress') and v.get('branch'):
    print(v['branch'])
" 2>/dev/null || true)

  for branch in $open_tids; do
    # Check for failed CI runs on this branch
    ci_fail=$(gh run list --branch "$branch" --status failure --limit 1 --json databaseId \
              2>/dev/null | python3 -c "import json,sys; d=json.load(sys.stdin); print('true' if d else 'false')" 2>/dev/null || echo "false")
    if [ "$ci_fail" = "true" ]; then
      pr_failures+=("$branch")
      alerts+=("ALERT: CI failing on branch $branch at $ts")
    fi
  done
fi

# ── 4. Open task count ───────────────────────────────────────────────────────
open_tasks=0
if [ -f "$TASKS_FILE" ]; then
  open_tasks=$(python3 -c "
import json
with open('$TASKS_FILE') as f:
  d = json.load(f)
print(sum(1 for v in d.values() if v.get('status') in ('pending', 'in_progress')))
" 2>/dev/null || echo "0")
fi

# ── 5. Shared dirs ───────────────────────────────────────────────────────────
handoffs_dir=$([ -d "$HOME/.openclaw/shared/handoffs" ] && echo "true" || echo "false")
events_dir=$([ -d "$HOME/.openclaw/shared/events" ] && echo "true" || echo "false")
worktrees_dir=$([ -d "$HOME/.openclaw/worktrees" ] && echo "true" || echo "false")

# ── 6. Write atomic JSON output ──────────────────────────────────────────────
pr_failures_json=$(python3 -c "import json; print(json.dumps($( [ ${#pr_failures[@]} -gt 0 ] && printf '%s\n' "${pr_failures[@]}" | python3 -c 'import sys,json; print(json.dumps(sys.stdin.read().splitlines()))' || echo '[]')))" 2>/dev/null || echo "[]")

tmp=$(mktemp "$HOME/.openclaw/shared/.monitor-status.XXXXXX.tmp")
cat > "$tmp" << JSON
{
  "ts": "$ts",
  "gateway": { "running": $gateway_ok, "pid": "${gateway_pid:-null}" },
  "acp_sessions": { "builder_acp": $builder_acp, "devops_acp": $devops_acp },
  "shared_dirs": { "handoffs": $handoffs_dir, "events": $events_dir, "worktrees": $worktrees_dir },
  "open_tasks": $open_tasks,
  "ci_failures": $pr_failures_json,
  "alerts_sent": ${#alerts[@]}
}
JSON
mv "$tmp" "$OUTPUT"

# ── 7. Send alerts ───────────────────────────────────────────────────────────
for msg in "${alerts[@]}"; do
  # Primary: openclaw CLI (only if gateway alive)
  if [ "$gateway_ok" = "true" ]; then
    openclaw sessions send main "$msg" 2>/dev/null || true
  fi
  # Fallback 1: macOS notification
  if command -v osascript &>/dev/null; then
    osascript -e "display notification \"$msg\" with title \"openclaw monitor\"" 2>/dev/null || true
  fi
  # Fallback 2: ntfy push (if topic configured)
  if [ -n "${OPENCLAW_NTFY_TOPIC:-}" ]; then
    curl -s -d "$msg" "https://ntfy.sh/$OPENCLAW_NTFY_TOPIC" &>/dev/null || true
  fi
  echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] $msg" >> "$HOME/.openclaw/logs/monitor.log"
done
SCRIPT
chmod +x ~/.openclaw/bin/check-agents.sh
```

- [ ] **Step 3: Run once manually to verify output**
```bash
~/.openclaw/bin/check-agents.sh
cat ~/.openclaw/shared/monitor-status.json | python3 -m json.tool
```
Expected: valid JSON with `gateway`, `acp_sessions`, `shared_dirs`, `open_tasks`, `ci_failures`, `alerts_sent` fields.

- [ ] **Step 4: Install system cron (every 10 minutes)**
```bash
# Add to crontab if not already present
(crontab -l 2>/dev/null; echo "*/10 * * * * $HOME/.openclaw/bin/check-agents.sh >> $HOME/.openclaw/logs/monitor.log 2>&1") | \
  sort -u | crontab -
crontab -l | grep check-agents
```
Expected: cron entry printed — `*/10 * * * * .../check-agents.sh`.

- [ ] **Step 5: Add log directory**
```bash
mkdir -p ~/.openclaw/logs
echo "Log directory ready: ~/.openclaw/logs"
```

---

### Task 28: Commit plan and spec to claude_code_src repo

**Files:**
- This plan is already at `docs/superpowers/plans/2026-04-02-openclaw-agent-team.md`
- Spec is at `docs/superpowers/specs/2026-04-02-openclaw-agent-team-design.md`

- [ ] **Step 1: Commit plan file**
```bash
cd /Users/qqzhang/workspace/claude_code_src
git add docs/superpowers/plans/2026-04-02-openclaw-agent-team.md
git commit -m "docs: add openclaw agent team implementation plan"
```

---

## Self-Review

**Spec coverage check:**

| Spec section | Covered by task(s) |
|---|---|
| Team Roster (8 agents) | Tasks 2–4, 7 |
| 2.1 Dispatch Mechanism | Tasks 13, 18–19 |
| 2.2 Communication Patterns | Task 13 (baton/parallel params) |
| 2.3 P2P Visibility | Task 15 |
| 2.4 State Store (file-locked) | Task 11 |
| 2.5 Config Override Hierarchy | Task 8 (models.json step) |
| 2.6 Monitoring Heartbeat | Task 26 (check-agents.sh: process + tmux + PR/CI + alerts) |
| 2.7 TID Format | Task 10 |
| 2.8 Git Worktree per Feature | Task 5 (冷燕 SOUL.md) |
| 2.9 GitHub as Single Source of Truth | Tasks 2–5 (SOUL.md rules) |
| 2.10 tmux Persistent ACP Sessions | Tasks 4–5 (SOUL.md) |
| 2.11 Per-Worktree Agent Log | Noted in SOUL.md; full impl deferred (spec says defer) |
| Q仔 Orchestration Rules (incl. proactive scanning) | Tasks 5–6, 20 |
| Agent Role Definitions (incl. 傅红雪 QA gate TID check) | Tasks 2–5 |
| 6.1 Plugin Interface | Task 18 (index.ts) |
| 6.2 team_dispatch tool (incl. retryContext spec enum) | Task 13 |
| 6.3 agent:bootstrap hook | Task 14 |
| 6.4 P2P Visibility Hook | Task 15 |
| 6.5 ACP Enforcement Hook | Task 16 |
| 6.5.1 Callback ACK Protocol | Tasks 3–4 (傅红雪 SOUL.md) |
| 6.6 Orchestrator Soft-Warning | Task 17 |
| 6.7 Per-Agent Tool Whitelist | Noted in SOUL.md; plugin enforcement via 6.5 is the gate |
| 6.8 Linter as Hard Gate | Not in this plan (requires per-repo linter config — deferred) |
| 6.9 Skill Security Review Gate | Task 20 references security-reviewer; full impl is a separate skill |
| 6.10 GC Cron + entropy checks | Task 18 (gc-service.ts incl. runEntropyChecks) |
| 6.11 PR Screenshot Validation Gate | Task 5 Step 4 (冷燕 SOUL.md rule) + deferred GitHub Actions config |
| 7. Agent ID Migration | Tasks 7–8 |
| Phase 3 Protocol Docs (incl. SUBAGENT_PACKET_TEMPLATE.md) | Tasks 20–21 |
| 8. Implementation Phases 1–4 | Tasks 1–28 |

**Deferred (out of scope for this plan, per spec Section 9):**
- Per-worktree `.agent-log.jsonl` (spec says "revisit when simple logs prove insufficient")
- Per-agent tool whitelist enforcement beyond ACP (requires plugin per-agent config)
- Linter as hard gate (requires target repo linter setup)
- Skill security review gate skill (separate deliverable)
