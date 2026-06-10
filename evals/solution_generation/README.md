# Solution Generation Evals

## 目录目的

本目录用于记录“方案生成 Agent”的测试用例、运行记录和问题修复过程。

方案生成 Agent 位于能力匹配评估 Agent 之后，负责基于结构化需求、证据检索结果和能力匹配评估结果，生成售前可审核的初版方案。

该 Agent 的输出同时包含：

- proposal_json：结构化方案，用于后续风险校验 Agent 和系统处理。
- proposal_text_draft：自然语言方案初稿，用于风险校验后作为最终对话输出候选内容。

方案生成 Agent 不负责重新检索知识库，不重新判断能力支持等级，也不能承诺报价、交期、私有化部署一定可交付或定制开发范围。

---

## 输入来源

方案生成 Agent 的主要输入包括：

text 结构化需求： 格式化需求与检索问题.structured_requirement_text  证据检索结果： 证据检索 Agent.structured_output  能力匹配评估结果： 能力匹配评估 Agent.structured_output 

如果 Dify 不能稳定将 Object 插入 Prompt，可以通过 Code 节点将证据检索结果和能力匹配评估结果转换为 JSON 字符串后再输入方案生成 Agent。

---

## 输出目标

方案生成 Agent 应输出严格 JSON：

json {   "proposal_json": {     "title": "",     "customer_requirement_summary": "",     "solution_overview": "",     "recommended_capabilities": [],     "integration_plan": [],     "deployment_notes": [],     "delivery_boundaries": [],     "human_confirmation_items": [],     "risk_notes": [],     "excluded_or_unconfirmed_items": []   },   "proposal_text_draft": "" } 

其中：

- proposal_json 用于后续风险校验和结构化处理。
- proposal_text_draft 是自然语言初版方案，但仍然只是售前可审核版本，不是最终客户承诺版本。

---

## 核心生成规则

### 1. 基于能力匹配结果生成

方案生成 Agent 必须以 capability_assessments 为准，不得自行新增能力判断。

不同支持等级的表达规则如下：

text standard_supported -> 可写为支持该能力 configurable_supported -> 必须写明可通过配置支持 integration_required -> 必须写明完成系统/API/数据源对接后支持 needs_human_confirmation -> 必须写明可评估，需进一步确认 not_supported_or_no_evidence -> 不得写入推荐方案主体，只能写入未确认项 

### 2. 保留风险和待确认项

方案中必须保留：

- 系统/API 对接依赖
- 客户侧配合事项
- 私有化部署边界
- 报价、费用、交付周期待确认项
- 技术细节缺口
- 能力匹配评估中的 risk_flags
- 能力匹配评估中的 human_confirmation_items

### 3. 禁止过度承诺

方案中不得出现以下表达：

text 保证上线 一定可以 确定交付周期 确定费用 无需客户系统改造即可支持 私有化部署可直接交付 开箱即用支持订单查询 

涉及订单状态查询、物流查询、CRM 查询、工单查询等系统数据能力时，必须写明需要完成客户系统/API对接。

涉及私有化部署、定制开发、交付周期、费用时，必须写明需人工确认。

### 4. 去重规则

方案生成 Agent 必须对以下字段进行语义去重：

text human_confirmation_items risk_notes delivery_boundaries deployment_notes excluded_or_unconfirmed_items 

如果多个条目表达同一件事，只保留一个更完整、更具体的版本。

示例：

text “获取 API 文档” “获取客户订单系统的 API 文档” 

应合并为：

text 获取客户订单系统 API 文档 

示例：

text “评估对接可行性” “安排技术团队评估对接可行性” 

应合并为：

text 安排技术团队评估订单系统 API 对接可行性 

去重后仍需保留所有关键确认点，不得因为合并而遗漏 API 文档、鉴权方式、字段映射、测试环境、私有化部署条件、交付周期或费用确认。

---

## 当前测试集

### SG-001 零售 AI 客服初版方案生成

来源：CA-001 能力匹配评估结果。

覆盖能力项：

- 商品咨询
- 订单查询
- 私有化部署
- 订单系统

预期：

text 商品咨询 -> 作为标准支持能力写入方案 订单查询 -> 写成完成订单系统 API 对接后支持 订单系统 -> 写清客户需提供 API 文档、鉴权方式、字段映射、测试环境等 私有化部署 -> 写成可评估，需交付负责人确认部署条件、周期和费用 

不得承诺：

text 确定报价 确定交期 私有化部署一定可交付 订单查询无需 API 对接即可支持 

---

## 结果标记

使用以下结果：

```text
Pass, Pass with minor issues, Partial Pass, Fail 
 ```

### Pass

方案同时输出 JSON 和自然语言初稿，关键能力表达、风险、依赖和待确认项均符合预期，无明显重复或过度承诺。

### Pass with minor issues

方案整体可用，但存在轻微措辞、标题、去重或非关键表达问题，不影响进入风险校验 Agent。

### Partial Pass

方案主体可用，但存在需要 Prompt 修复的问题，例如待确认项重复较多、部分依赖表达不完整、轻微边界混淆等。

### Fail

出现严重过度承诺、无证据能力写入方案主体、把 integration_required 写成 standard_supported，或把 needs_human_confirmation 写成确定交付。