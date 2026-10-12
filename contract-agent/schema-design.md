# 知识库 Schema 设计说明

> 配套 PRD v2.0（对话形态）。本文档说明 `schemas/` 下 7 个 JSON Schema（Draft 2020-12）的设计决策、枚举字典、版本/快照机制与检索落地建议，供工程实现直接使用。

## 1. 总览与关系

```
SourceMaterial (M-)  投喂素材及解析结果
   │ 提炼（经用户确认）
   ├──> PrincipleItem (P-)  原则层：立场/红线/裁决优先级
   ├──> RuleItem      (R-)  规则层：结构化判定条件 + 建议文本（审核主力）
   └──> CaseItem      (C-)  案例层：正反条款 + 点评（语义比对/佐证）
                 ▲                │
                 │ related_rule_ids / related_case_ids
Feedback (FB-) ──┘                │ 审核时引用（必须带版本）
   ▲ 处置回流                     ▼
ReviewTask (T-) ── 1:N ── Finding (F-)
   └── kb_snapshot (SNAP-) 出审时条目版本只读清单
```

- 一次审核 `ReviewTask` 含 N 条 `Finding`；每条 Finding 的 `evidence_refs` 引用快照内的知识条目（`item_id + version`）或系统检查项 `check_code`。
- `Feedback` 挂在 Finding 上；反馈经沉淀确认后产生新的候选 Item（状态 `candidate`），回到学习确认流程，**不直接修改**任何 active 条目。
- 所有 Item 共四态：`candidate → active →（suspended | superseded）`。

## 2. ID 编号规则

| 对象 | 规则 | 示例 |
|---|---|---|
| 素材 | `M-` + 3 位以上 | M-1001 |
| 原则 | `P-` + 3 位 | P-002 |
| 规则 | `R-` + 条款类型代码 + `-` + 3 位 | R-PAY-014、R-FORCE-003 |
| 案例 | `C-` + 3 位 | C-007 |
| 审核任务 | `T-` + 6 位以上 | T-20261012001 |
| 发现 | `F-` + 6 位以上 | F-20261012001 |
| 反馈 | `FB-` + 4 位以上 | FB-0001 |
| 快照 | `SNAP-` + 3 位 | SNAP-217 |

- ID 全生命周期不变；版本演进不改 ID。
- 规则 ID 中的条款类型代码是 ID 的一部分，不可随条目改类而变更；如需改条款类型，新建条目并将旧条目标 `superseded`。

## 3. 枚举字典（全系统唯一口径）

| 维度 | 取值 | 含义 |
|---|---|---|
| stance 立场 | `party_a` / `party_b` / `both` | 甲方 / 乙方 / 双方通用 |
| risk_level | `high` / `medium` / `low` / `info` | 高 / 中 / 低 / 提示（非风险定性） |
| confidence | `high` / `medium` / `low` | 命中规则条件 / 语义相似或仅案例 / 推断关联 |
| strictness | `loose` / `standard` / `strict` | 只影响展示范围，不改变等级 |
| item status | `candidate` / `active` / `suspended` / `superseded` | 候选 / 生效 / 停用 / 被取代 |
| task status | `reviewing` / `pending_review` / `completed` / `interrupted` / `failed` | |
| finding status | `pending` / `accepted` / `rejected` / `risk_accepted` / `applied_suggestion` | |
| feedback type | `false_positive` / `missing_risk` / `suggestion_edited` / `level_disagreement` | 误报 / 漏报 / 建议修正 / 等级异议 |

**条款类型代码（clause_type）**：`payment`（付款）、`delivery_acceptance`（交付验收）、`penalty`（违约）、`confidentiality`（保密）、`ip`（知识产权）、`warranty`（质保）、`force_majeure`（不可抗力）、`dispute_resolution`（争议解决）、`termination`（终止解除）、`liability_limit`（责任限制）、`indemnity`（赔偿）、`assignment`（转让）、`auto_renewal`（自动续约）、`tax_invoice`（税费发票）、`data_compliance`（数据合规）、`non_compete`（竞业限制）、`other`。

**合同类型代码（contract_type）**：`sales` / `procurement` / `service` / `lease` / `loan` / `employment` / `nda` / `general` / `other`。

> 扩展新代码 = 升级 Schema（枚举是强约束，刻意为之，避免标签失控导致检索漏召回）。

## 4. 版本化与审计

- Item 每次写入：`version + 1`，旧记录整体归档（建议同集合保留历史版本行或独立 `*_history` 表），字段 `supersedes_version` 指向上一版。
- 冲突裁决：被取代条目不物理删除，置 `superseded` 并在审计日志记录裁决人、理由、胜出条目。
- 撤销窗口内的「撤销」= 用归档版本回写一个新版本，不做物理回退，保证审计链连续。
- **审计日志（audit_log）建议结构**（未单列 Schema，实现时建集合）：
  `{ at, actor, action: activate|update|suspend|resume|rollback|resolve_conflict|delete_material|export, object_type, object_id, from_version, to_version, summary, conversation_ref }`。

## 5. 规则判定条件（conditions）求值约定

规则可以只有语义描述（无 `conditions`），但审核命中此类规则时置信度封顶 `medium`。结构化条件结构：

```json
{
  "logic": "any",
  "conditions": [
    {
      "kind": "numeric",
      "field": "derived.daily_rate_annualized",
      "operator": "gt",
      "value": 0.138,
      "unit": "ratio",
      "annualize": true,
      "note": "日费率×365 年化后高于一年期 LPR 4 倍"
    },
    {
      "kind": "numeric",
      "field": "derived.penalty_cap_ratio",
      "operator": "gt",
      "value": 0.10,
      "unit": "ratio",
      "annualize": false,
      "note": "违约金总额超合同金额 10%"
    }
  ]
}
```

- `field = clause_text` 时按关键词/正则对条款原文求值；`field = derived.*` 时由模型先从条款抽取数值（日费率、总额比例、天数等），**由服务端规则引擎做权威比较**，模型不自行判定阈值是否命中，只负责在 `confidence_reason` 中复述计算过程。
- `annualize: true` 表示对 `derived.daily_rate` 先 ×365 再比较；单位换算口径写入 `note`，并随审核意见展示给用户。
- 阈值全部来自用户投喂的规则内容（Schema 不内置任何数值）；动态阈值（如 LPR 倍数）P0 存静态值 + 由投喂人维护，条目上可借 `tags`/`change_note` 标注时效。
- 嵌套逻辑用 `groups` 递归表达（AND/OR 混合）。

## 6. 知识库快照（kb_snapshot）

- `create_review_task` 时服务端调用 `snapshot_kb()`：按本次 profile（stance 命中规则 + 双方通用 + scope 限定）冻结全部适用 active 条目的 `(类型, ID, version)` 清单。
- Finding 的依据只允许引用快照内版本；审核中途即使条目被修订，本次结论不变。
- 用户日后提起旧任务：从快照版本渲染依据；若该条目已有更新版本，在对话中提示并可做版本对比。

## 7. 存储与检索落地建议

| 数据 | 建议存储 | 关键索引 |
|---|---|---|
| items（三类合一或分集合均可） | 文档库（Mongo/PostgreSQL JSONB） | `(id, version)` 唯一、`status`、`stance`、`clause_types`、`contract_types` |
| case 语义检索 | 向量库（pgvector/Milvus/ES kNN） | 对 `clause_text + expert_comment` 建向量；元数据过滤 `status=active, stance, clause_types, contract_types` |
| materials / tasks / feedback / audit | 文档库/关系库 | 会话引用、时间、关联 ID |

**审核时的检索编排（hybrid）**：
1. 规则：按 `clause_type + stance（含 both）+ contract_type（含 general）+ active` 过滤后**全量加载**（单类型规则量可控），逐条跑条件引擎；无命中的规则不参与。
2. 案例：同过滤条件下向量 top_k=5 语义召回，相似度低于阈值的不引用；仅案例佐证的发现置信度 ≤ medium。
3. 原则：可全量驻留系统上下文（量小），用于立场校验与红线兜底；红线原则命中直接 high 并进 verdict 摘要。
4. 系统检查（缺失/一致性）：纯解析层计算，不经知识库，输出为 `category=missing_clause|cross_clause` 的 Finding，依据用 `check_code`。

## 8. 与对话层的契约要点

- 模型不直连数据库：所有读写经 PRD 所列工具（function call）；Schema 即工具入参/出参的校验层，建议在工具网关对写入数据做 Schema 校验（`candidate` 写入除外——候选允许字段不全，见下）。
- **候选宽松、生效严格**：`propose_knowledge` 入库的 candidate 建议用同一 Schema 但放宽必填（`status=candidate` 时仅要求 id/type/version/status/source/created_at）；`activate_item` 时按完整 Schema 强校验，不通过则要求模型/用户补齐。
- 时间字段统一 ISO 8601 UTC；逻辑删除用 status，不做物理删除（素材删除除外，且需用户确认并留痕）。

## 9. 文件清单

```
schemas/
├── principle-item.schema.json   原则 P-
├── rule-item.schema.json        规则 R-（含 conditions 条件组 $def）
├── case-item.schema.json        案例 C-
├── source-material.schema.json  素材 M-（含解析切分）
├── review-task.schema.json      审核任务 T-（内含快照，$ref finding）
├── finding.schema.json          发现 F-（条款风险强制要求知识依据；驳回强制原因）
└── feedback.schema.json         反馈 FB-（误报强制原因；与知识库解耦）
examples/
└── seed-knowledge.json          示例夹具，已通过全部 Schema 校验
```
