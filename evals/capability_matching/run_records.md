# 能力匹配评估 Agent 运行记录

## 2026-06-08

### 运行环境

- 平台：Dify Chatflow
- 模型：qwen3.5-flash
- Temperature：0.2
- 是否开启结构化输出：是
- Prompt 版本：prompts/03_capability_assessment_prompt.md v0.1
- Schema 版本：schemas/capability_assessment_output.schema.json
- 上游输入：ER-001 证据检索 Agent structured_output

---

## CA-001 零售 AI 客服能力匹配评估

### 输入来源

ER-001 多轮补齐后的零售 AI 客服证据检索结果。

### 输入摘要

本次能力匹配评估基于以下 evidence_results：

text 商品咨询： - evidence_found = true - evidence_quality = high - 证据显示 AI 客服标准支持基于商品知识库回答商品咨询问题。  订单查询： - evidence_found = true - evidence_quality = high - 证据显示可以支持订单状态查询，但需要客户订单系统 API 对接。 - 如果客户订单系统不开放 API，则不能承诺支持实时订单状态查询。  私有化部署： - evidence_found = true - evidence_quality = medium - 证据显示 AI 客服企业版可评估私有化部署。 - 部署条件、交付周期和费用需由交付负责人确认。  订单系统： - evidence_found = true - evidence_quality = medium - 证据显示订单状态查询依赖客户订单系统 API。 - 缺少 API 字段映射、鉴权机制、实时性等详细说明。 

### 预期

- 商品咨询：
  - support_level = standard_supported
  - human_confirmation_required = false
  - plan_eligible = true

- 订单查询：
  - support_level = integration_required
  - human_confirmation_required = true
  - technical_dependencies 包含客户订单系统 API
  - plan_expression 应表达“完成系统/API 对接后支持”

- 私有化部署：
  - support_level = needs_human_confirmation
  - human_confirmation_required = true
  - human_confirmation_items 包含交付负责人、部署条件、周期、费用

- 订单系统：
  - support_level = integration_required
  - human_confirmation_required = true
  - technical_dependencies 包含订单系统 API
  - human_confirmation_items 包含 API 文档、字段映射、鉴权方式等
  - plan_expression 应表达“完成订单系统 API 对接和技术评估后支持”

### 实际输出

```json 
{   "capability_assessments": [     {       "requirement_item": "商品咨询",       "support_level": "standard_supported",       "assessment_summary": "AI 客服标准支持基于商品知识库回答商品咨询问题，包括规格、使用方式等。退换货政策问答属于可配置支持能力，但核心商品咨询为标准支持。",       "basis": [         "01_产品能力说明.md: AI 客服标准支持基于商品知识库回答商品咨询问题",         "04_FAQ_售前常见问题.md: 支持基于商品知识库回答商品规格、使用方式等问题"       ],       "delivery_boundaries": [         "退换货政策问答属于可配置支持能力，需确认具体逻辑和规则"       ],       "technical_dependencies": [],       "pricing_notes": [],       "human_confirmation_required": false,       "human_confirmation_items": [],       "risk_flags": [],       "plan_expression": "支持该能力",       "plan_eligible": true     },     {       "requirement_item": "订单查询",       "support_level": "integration_required",       "assessment_summary": "AI 客服支持订单状态查询，但必须依赖客户订单系统的 API 对接。如果客户订单系统不开放 API，则无法承诺支持实时查询。",       "basis": [         "04_FAQ_售前常见问题.md: 可以支持，但需要客户订单系统 API 对接",         "02_技术方案与系统对接说明.md: 如果客户订单系统不开放 API，则不能承诺支持实时订单状态查询"       ],       "delivery_boundaries": [         "需客户订单系统开放 API 接口"       ],       "technical_dependencies": [         "客户订单系统 API 对接"       ],       "pricing_notes": [],       "human_confirmation_required": true,       "human_confirmation_items": [         "确认客户订单系统是否提供标准 API 接口及鉴权方式"       ],       "risk_flags": [         "依赖客户侧系统开放 API，若未开放则无法实现"       ],       "plan_expression": "可在完成系统/API 对接后支持",       "plan_eligible": true     },     {       "requirement_item": "私有化部署",       "support_level": "needs_human_confirmation",       "assessment_summary": "AI 客服企业版可评估私有化部署方案，但具体部署条件、交付周期、费用必须由交付负责人进一步确认，AI 客服团队不能自动承诺。",       "basis": [         "01_产品能力说明.md: AI 客服企业版可评估私有化部署方案，具体部署条件、交付周期和费用需由交付负责人进一步确认",         "03_交付边界与人工确认规则.md: AI 客服企业版可评估私有化部署，但私有化部署必须由交付负责人确认",         "02_技术方案与系统对接说明.md: 私有化部署交付周期和费用不能由 AI 客服自动承诺，需要交付负责人确认"       ],       "delivery_boundaries": [         "需交付负责人确认部署条件、交付周期和费用"       ],       "technical_dependencies": [],       "pricing_notes": [         "费用需交付负责人确认"       ],       "human_confirmation_required": true,       "human_confirmation_items": [         "联系交付负责人评估私有化部署的具体技术环境要求和商务条款"       ],       "risk_flags": [         "交付周期和费用不确定，需人工介入评估"       ],       "plan_expression": "可评估，需进一步确认交付条件、周期和费用",       "plan_eligible": true     },     {       "requirement_item": "订单系统",       "support_level": "needs_human_confirmation",       "assessment_summary": "对接订单系统主要依赖客户订单系统开放 API 接口以支持订单状态查询。文档中未详细列出所有技术对接细节（如字段映射、鉴权协议），仅强调了 API 开放的必要性。证据质量中等，需人工确认技术可行性。",       "basis": [         "04_FAQ_售前常见问题.md: 可以支持，但需要客户订单系统 API 对接",         "02_技术方案与系统对接说明.md: 如果客户订单系统不开放 API，则不能承诺支持实时订单状态查询"       ],       "delivery_boundaries": [         "需客户订单系统开放 API 接口"       ],       "technical_dependencies": [         "客户订单系统 API 接口"       ],       "pricing_notes": [],       "human_confirmation_required": true,       "human_confirmation_items": [         "获取客户订单系统的 API 文档，并安排技术人员评估对接可行性"       ],       "risk_flags": [         "缺少关于订单系统对接的具体技术协议、字段映射要求、鉴权方式的详细说明"       ],       "plan_expression": "可评估，需进一步确认交付条件、周期和费用",       "plan_eligible": true     }   ],   "assessment_summary": {     "standard_supported_items": [       "商品咨询"     ],     "configurable_supported_items": [],     "integration_required_items": [       "订单查询"     ],     "needs_human_confirmation_items": [       "私有化部署",       "订单系统"     ],     "not_supported_or_no_evidence_items": [],     "overall_risk_flags": [       "依赖客户侧系统开放 API，若未开放则无法实现",       "交付周期和费用不确定，需人工介入评估",       "缺少关于订单系统对接的具体技术协议、字段映射要求、鉴权方式的详细说明"     ],     "plan_eligible_items": [       "商品咨询",       "订单查询",       "私有化部署",       "订单系统"     ],     "blocked_items": [],     "recommended_next_step": "针对私有化部署联系交付负责人；针对订单系统对接，需客户提供 API 文档并进行技术评估。"   } } 
```

### 结果

Partial Pass

### 通过点

- 商品咨询正确识别为 standard_supported。
- 订单查询正确识别为 integration_required。
- 私有化部署正确识别为 needs_human_confirmation。
- Agent 正确识别订单查询依赖客户订单系统 API。
- Agent 正确识别私有化部署的部署条件、交付周期和费用需要人工确认。
- Agent 没有承诺私有化部署的确定交期和费用。
- plan_eligible_items 包含可进入方案但需带边界表达的需求项。

### 问题记录

1. 订单系统主支持等级判断不准确。

当前输出：

json "requirement_item": "订单系统", "support_level": "needs_human_confirmation" 

但订单系统对接本质属于系统/API 对接类需求，主支持等级应为：

json "support_level": "integration_required" 

同时保留：

json "human_confirmation_required": true 

该问题说明 Agent 将“需要人工确认 API 细节”误作为主分类，而没有保留“系统对接需求”的主支持等级。

2. 订单系统 plan_expression 不准确。

当前输出：

text 可评估，需进一步确认交付条件、周期和费用 

该表达更适合私有化部署，不适合订单系统对接。

更合理的表达应为：

text 可在客户订单系统开放 API 并完成接口评估后支持对接。 

或：

text 可在完成订单系统 API 对接后支持相关订单查询能力，需客户提供 API 文档、鉴权方式和字段说明。 

3. 商品咨询中混入了退换货政策问答的边界说明。

当前输出：

text 退换货政策问答属于可配置支持能力，需确认具体逻辑和规则 

该内容并非商品咨询本身的交付边界。当前阶段暂不处理，后续可通过强化证据归因规则修复。

### 根因分析

1. Prompt 中对 support_level 与 human_confirmation_required 的关系约束不够明确。

订单系统对接同时具备两个属性：

text 主分类：integration_required 人工确认：需要 

但模型将“需要人工确认”提升成了主 support_level。

2. Prompt 中的判断优先级可能导致 needs_human_confirmation 覆盖了 integration_required。

当前规则中如果 evidence_quality = medium 或需要人工确认，模型倾向将 support_level 输出为 needs_human_confirmation。对于 API 对接类需求，应明确“系统对接是主支持等级，人工确认是附加属性”。

3. plan_expression 缺少按 support_level 和 requirement_type 区分的表达约束。

订单系统对接应使用 API 对接类表达，而不是部署/交付类表达。

### 调整动作

1. Prompt 增加主支持等级与人工确认拆分规则：

text support_level 表示需求项的主要支持类型。 human_confirmation_required 表示是否需要人工确认。 二者不是互斥关系。  如果需求项本质是系统/API/数据源对接，即使需要人工确认 API 文档、鉴权方式、字段映射或联调条件，support_level 仍应优先为 integration_required，同时设置 human_confirmation_required = true。 

2. Prompt 调整 support_level 判断规则：

text 如果 evidence 或 technical_dependencies 显示该需求依赖客户系统、API、数据源或第三方接口，则 support_level = integration_required。  如果该需求同时需要人工确认 API 细节、联调条件、字段映射、鉴权方式或实时性要求，不要改成 needs_human_confirmation，而应保留 integration_required，并设置 human_confirmation_required = true。 

3. Prompt 增加 plan_expression 分类规则：

text 对 integration_required： plan_expression 应表达为“可在完成系统/API 对接后支持”，并说明客户侧依赖。  对订单系统： plan_expression 应表达为“可在客户订单系统开放 API 并完成接口评估后支持对接”。  对 needs_human_confirmation： plan_expression 才使用“可评估，需进一步确认交付条件、周期和费用”。 

4. 后续复测 CA-001，重点验证：

text 订单系统 -> integration_required human_confirmation_required -> true integration_required_items 包含订单系统 needs_human_confirmation_items 可以包含订单系统作为人工确认项，但不能替代主支持等级 plan_expression 使用订单系统 API 对接类表达 

