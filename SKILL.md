---
name: deep-research
version: 3.3.0
description: |
  深度调研技能 v3.3 — File-based Handoff + Merge Agent + Stage State + IMA Publish。Scout(3路并行)→Merge→Analyze(2路并行)→Draft→Review→Revise→Publish(ima-upload)。
  Use when: 深度调研,技术评估,行业分析,多源交叉,L3任务
  NOT for: 简单搜索, 纪检分析(L2任务请使用对应技能)
metadata:
  openclaw:
    emoji: 🔍
    requires:
      sessions: sessions_spawn 可用
---

# Deep-Research v3.3 🔍

> 8-Agent File-based Handoff Pipeline + IMA Publish。不再通过 task 参数传递数据——Agent 读写 `artifacts/` 目录。
> **v3.3: 嵌入 solo-file-transfer (Phase 6) + 复盘改进**

## ⚡ Solo Status 协议（强制）

所有技能（DR/DI/MSF）在每次管线阶段切换时，**必须更新状态文件** `./solo/pipeline-status.json`，实现统一的进度可视化和`solo status`命令支持。

**更新时机：** 每个阶段 spawn 前 + 每个阶段完成后

**写文件格式：**
```json
{
  "pipeline_id": "DR-20260524-topic-name",
  "skill": "deep-research",
  "topic": "研究主题",
  "started_at": "ISO时间",
  "last_updated": "ISO时间",
  "phases": {
    "Phase 1: Scout":      {"status": "completed", "detail": "3路·41条来源"},
    "Phase 1.5: Merge":    {"status": "running",   "detail": "去重中"},
    "Phase 2: Analyze":    {"status": "pending",   "detail": ""},
    "Phase 3: Draft":      {"status": "pending",   "detail": ""},
    "Phase 4: Review":     {"status": "pending",   "detail": ""},
    "Phase 5: Revise":     {"status": "pending",   "detail": ""}
  },
  "running_subagents": [
    {"name": "当前运行子代理", "running_seconds": 0}
  ]
}
```

**status取值：** `completed`（完成）| `running`（运行中）| `pending`（等待）| `failed`（失败）

`solo status` 响应时读取此文件，格式化为进度面板展示。

---

## 六阶段管线（8 Agent）

```
Phase 1: Scout (3路并行 sessions_spawn)
  ├─ Agent 1a: Web Scout → 写 artifacts/scout_web.json
  ├─ Agent 1b: Academic Scout → 写 artifacts/scout_academic.json
  └─ Agent 1c: Deep Scout → 写 artifacts/scout_deep.json

Phase 1.5: Merge (sessions_spawn) ← NEW
  └─ Agent 1d: Merge → 读 3 个 scout_*.json → 去重+评分→ 写 artifacts/merge.json + handoff.md

Phase 2: Analyze (2路并行 sessions_spawn)
  ├─ Agent 2a: 主视角 → 读 artifacts/merge.json → 写 artifacts/analyze_main.json
  └─ Agent 2b: 对立视角 → 读 artifacts/merge.json → 写 artifacts/analyze_counter.json

Phase 3: Draft (sessions_spawn)
  └─ Agent 3 → 读 artifacts/merge.json + analyze_*.json → 写 artifacts/draft_report.md

Phase 4: Review (sessions_spawn) ⚠️ 独立审计
  └─ Agent 4 → 读 artifacts/merge.json + draft_report.md → 写 artifacts/review_ledger.json

  📊 评分矩阵（8维·基于 Google Code Review 指南 + Conventional Comments + Evidence Grading）

  | # | 维度 | 权重 | GitHub 来源 | 检查内容 |
  |:-:|:-----|:----:|:-----------|:--------|
  | 1 | 研究设计合理性 | 15% | Google CR: Design | 调研框架是否合理？假设是否显式化？搜索策略是否有覆盖度？ |
  | 2 | **事实准确性** | 20% | Google CR: Functionality | 关键事实是否多源交叉验证？数据是否准确？结论是否从证据推导？ |
  | 3 | 信息密度与清晰度 | 10% | Google CR: Complexity+Naming | 报告是否清晰可读？术语是否准确？是否有无关冗余？ |
  | 4 | 可证伪性 | 10% | Google CR: Tests | 关键结论是否可被验证/证伪？反对意见是否处理？局限性是否声明？ |
  | 5 | **来源可追溯** | 20% | Google CR: Comments + ConvCom | 每条关键claim有来源标注？有 [UNSOURCED] 标记？质量分级(S1/S2/S3)合理？ |
  | 6 | 偏误控制 | 10% | Google CR: Style + Evidence | 是否检查了确认偏误/选择偏误/幸存者偏误？多源是否有对立视角？ |
  | 7 | 知识缺口扫描 | 5% | Google CR: Documentation | 是否诚实地标注了"不知道"的部分？缺口是否量化？ |
  | 8 | 创新洞察力 | 10% | ConvCom: praise + DR专属 | 报告是否有超越信息汇总的洞察？有"这对用户意味着什么"的价值判断？ |
  | **合计** | **100%** | | |

  **Conventional Comments 标签映射（审查意见分级）：**
  - issue (blocking) → must_fix（不修复则 REJECT）
  - issue (non-blocking) → P1 fix
  - suggestion → P2 improvement
  - nitpick → P3 nice-to-have
  - praise → strengths
  - question → 需要澄清的疑点

  **权重设计逻辑：**
  - 事实准确性(20%)+来源可追溯(20%)=40% — 深度调研的根基：事实要准、来源要可查
  - 研究设计合理性(15%)+偏误控制(10%)+可证伪性(10%)=35% — 方法论质量：怎么调研的、有没有偏见、能不能推翻
  - 信息密度(10%)+洞察力(10%)+缺口扫描(5%)=25% — 交付质量：好不好读、有没有深度、是否诚实

  **降档规则（根基维度·独立于总分）：**
  - 事实准确性 < 40 → 强制 REVISE（事实不准则报告不可用）
  - 来源可追溯 < 30 → 强制 REVISE（来源不明则无法验证）
  - 研究设计 < 30 → 强制 REVISE（方法不对则结论无效）

  **与 DI/MSF 评分矩阵的核心差异：**
  | 技能 | 维度来源 | 最高权重 | 场景 |
  |:-----|:--------|:--------|:-----|
  | DI ⚔️ | 二十四字方针 6维 | 定性准确 25% | 纪检案件，法规准确性优先 |
  | MSF ⚒️ | 民政部5A + GitHub 9维 | 证据质量 20% | 学会调研，信息可复现性优先 |
  | DR 🔍 | Google CR 8维 | 事实准确性+来源可追溯 各20% | 深度调研，事实可靠性优先 |

Phase 5: Revise (sessions_spawn)
  └─ Agent 5 → 读 artifacts/draft_report.md + review_ledger.json → 写 artifacts/final_report.md + revision_log.json

Phase 6: Publish (主会话直调) ← 🆕 v3.3
  └─ 若用户指定了IMA知识库 → 调用 solo-file-transfer 技能
     node skills/solo/scripts/ima-upload.cjs artifacts/final_report.md <KB_ID>

  ⛔ 报告纯净原则：上传 IMA 的报告必须是纯分析内容。
     禁止：审计声明 JSON / Agent 统计 / SOURCED/UNSOURCED 计数 / handoff 状态 / Review 台账
     允许：标题、日期、执行摘要、分析正文、来源引用（可附条款号+原文）
     上述元数据均在 artifacts/ 目录已有独立文件，无需在报告中重复。
```

## File-based Handoff 协议

**核心原则（来自 fainir）：**
- "Write state to files, not just prompt context"
- "Resume from files rather than trusting long prompt history"
- "Leave an explicit handoff with next actions, blockers, open questions"

**artifacts/ 目录结构：**
```
artifacts/
  ├── handoff.md         ← 当前管线状态（谁完成、谁待执行、数据在哪）
  ├── status.md          ← 全局进度
  ├── scout_web.json     ← Agent 1a 输出
  ├── scout_academic.json← Agent 1b 输出
  ├── scout_deep.json    ← Agent 1c 输出
  ├── merge.json         ← Agent 1d 合并输出
  ├── analyze_main.json  ← Agent 2a 输出
  ├── analyze_counter.json← Agent 2b 输出
  ├── draft_report.md    ← Agent 3 输出
  ├── review_ledger.json ← Agent 4 输出
  ├── final_report.md    ← Agent 5 最终输出
  └── revision_log.json  ← Agent 5 修订日志
```

**Phase 6 KB_ID 映射（公司分析相关）：**
```
公司分析: rOmwQPa9EIuGwZ-5oN93qPsZjJTViyawEUPyd1aYzvE=
```
其他KB_ID见 MEMORY.md → IMA知识库段。

**handoff.md 格式：**
```markdown
# Pipeline Handoff
- Phase: Scout_COMPLETE / Merge_COMPLETE / ...
- Next: merge / analyze / draft / review / revise
- Input file: artifacts/merge.json
- Status: ready / blocked / failed
- Blockers: [如有]
- Last update: ISO timestamp
```

## 编排流程（主会话）

```
主会话创建 artifacts/ + 初始化 handoff.md
  │
  ├─ Phase 1: 并行 spawn 1a+1b+1c (context: isolated)
  │     各 Agent 写 artifacts/scout_*.json
  │
  ├─ Phase 1.5: spawn Agent 1d (merge)
  │     读 3 个 scout_*.json → 去重评分 → 写 artifacts/merge.json
  │
  ├─ Phase 2: 并行 spawn 2a+2b (context: isolated)
  │     读 artifacts/merge.json → 写 artifacts/analyze_*.json
  │
  ├─ Phase 3: spawn Agent 3 (draft)
  │     读 artifacts/merge.json + analyze_*.json → 写 artifacts/draft_report.md
  │
  ├─ Phase 4: spawn Agent 4 (review) ⚠️ 独立审计
  │     读 artifacts/merge.json + draft_report.md → 写 artifacts/review_ledger.json
  │
  ├─ Gate: score ≥ 80 → 交付
  │        score < 80 → Phase 5
  │
  ├─ Phase 5: spawn Agent 5 (revise)
  │     读 draft_report.md + review_ledger.json → 写 artifacts/final_report.md
  │
  ├─ Phase 6: 主会话调用 solo-file-transfer (v3.3 🆕)
  │     若用户指定了IMA知识库 → node skills/solo/scripts/ima-upload.cjs artifacts/final_report.md <KB_ID>
  └─ 完成 → 主会话输出摘要（自动路由到当前通道，无需调 wecom_mcp）
```

**主会话不传递数据——只做 spawn + gate 检查。**

> ⛔ **推送规则：** 隔离 session 只产出，不推送。交付消息由 cron/framework 的 delivery.announce 自动路由。主会话的回复自动路由到用户通道，无需调 wecom_mcp。

## 子代理文件

| Agent | 文件 | v3.2 变化 |
|:-----:|:-----|:---------|
| 1a | agent1a-scout-web.md | 输出改为写 `artifacts/scout_web.json` |
| 1b | agent1b-scout-academic.md | 输出改为写 `artifacts/scout_academic.json` |
| 1c | agent1c-scout-deep.md | 输出改为写 `artifacts/scout_deep.json` |
| **1d** | **agent1d-merge.md** | **🆕 Merge Agent** |
| 2a | agent2a-analyze.md | 输入改为读 `artifacts/merge.json` |
| 2b | agent2b-analyze.md | 输入改为读 `artifacts/merge.json` |
| 3 | agent3-draft.md | 输入改为读 `artifacts/` 多文件 |
| 4 | agent4-review.md | 读 `artifacts/` → 写 `artifacts/review_ledger.json` |
| 5 | agent5-revise.md | 读/写 `artifacts/` |

## 与 v3.1 的关键差异

| 维度 | v3.1 | v3.2 | v3.3 |
|:-----|:-----|:-----|:-----|
| 数据传递 | task 参数手写摘要 | `artifacts/` 文件读写 | 同v3.2 |
| 合并 | 主会话手工 | **Agent 1d Merge** 自动去重评分 | 同v3.2 |
| 管线状态 | 无 | `handoff.md` + `status.md` | 同v3.2 |
| 信息损失 | 严重（三次摘要转述） | 零损失（文件原始 JSON） | 同v3.2 |
| 交付 | 手动复制到聊天窗口 | 手动复制 | **🆕 Phase 6: ima-upload** 自动上传IMA |
| 恢复能力 | 无（失败后从头来） | 读取 handoff.md + 续传 | 同v3.2 |
| Agent 数量 | 7 | **8** (+Merge) | **8** (+IMA技能复用) |
| 阶段数 | 5 | **6** | **7** (+Publish) |

## LEARNED PATTERNS

### v3.3: IMA Upload 融入管线 + 复盘改进 (2026-05-27)
来源：华润三九DR全链路复盘（IMA上传手动调试40min → 零）
改动：
- DR SKILL.md → v3.3。新增 Phase 6: Publish（调用 solo-file-transfer 技能）
- 管线从6阶段升级为7阶段（Scout→Merge→Analyze→Draft→Review→Revise→Publish）
- 新增KB_ID映射表（公司分析 rOmwQPa9…）
- 核心教训：子任务（如"发ima"）必须重新匹配技能，不能只用入口匹配的DR技能

### v4.0: babata-browser v4.0 — Persistent Login + 学术搜索首选 (2026-05-24)
来源：CNKI反爬全链路实战（石冰提供中山大学登录cookie）
改动：
- babata-browser SKILL.md → v4.0。新增Persistent Login Profiles章节（CNKI storage_state）
- agent1b-scout-academic.md → 搜索策略重写：babata-browser加载storage_state优先，搜索引擎兜底
- TOOLS.md → 新增学术搜索规则：涉及知网/万方/百度学术→跳过tavily/web_fetch，直接走babata-browser🔥
- MSF agent1-scout.md → 学术库CNKI优先babata-browser
- PLUGIN-REGISTRY.md → 版本号v3.1→v4.0，触发词新增「学术搜索」
实测：加载中山大学机构登录cookie后，CNKI首页/详情页/搜索页均绕过拼图滑块验证码
登录态有效期至2026-06-23，cron已设提前提醒

### v3.2.2: Academic Scout 反爬降级规则补齐 (2026-05-24)
来源：光迅科技DR实战复盘（石冰反馈）
改动：agent1b-scout-academic.md 新增页面抓取降级规则（反爬关键词检测+babata-browser自动切换）
触发：CNKI/百度学术/万方数据等中国学术站点反爬拦截
背景：此前Academic Scout只有搜索引擎级降级(tavily→web_search)，但没有抓取级降级，导致学术来源零篇

### v3.2.3: CNKI实测反爬详细策略 (2026-05-24)
来源：babata-browser vs CNKI 实战测试（石冰要求直接验证）
实测发现：
- CNKI使用**自研拼图滑块验证码**（「拖动下方拼图完成验证」），非标准reCAPTCHA/Cloudflare
- CloakBrowser能加载CNKI首页并渲染内容（3074字符全部可见），但搜索交互环节被拼图滑块拦截
- 详情页直链反爬最弱，手机版次之，首页搜索反爬最强
改动：
- agent1b-scout-academic.md → 新增CNKI各入口反爬强度排序+抓取优先级策略
- MSF agent1-scout.md → 同步更新CNKI反爬策略
- MSF agent1-scout.md → 同步添加CNKI详情页/手机版/首页搜索的反爬优先级

### v3.2.1: web_fetch 反爬检测规则注入 (2026-05-22)
来源：MSF 尽职调研实战复盘
改动：agent1c-scout-deep.md 新增反爬关键词检测（验证码|captcha|滑块|403|404等），命中→自动切 babata-browser
触发：企查查/广东社会组织平台/BOSS直聘等反爬拦截

### v3.2: File-based Handoff (2026-05-21)
来源: fainir/most-capable-agent-system-prompt + GPT Researcher + Magentic
v3.1 task 参数大小限制导致 30+ 来源摘要丢失 → v3.2 改为 artifacts/ 文件读写。
新增 Merge Agent 消除手工去重瓶颈。

### v3.1: 纸面造假修复 (2026-05-21)
v3.0 无子代理文件 → v3.1 补齐 7 个 Agent prompt。

### 审稿分离
Review Agent 独立会话，不继承上游上下文。
