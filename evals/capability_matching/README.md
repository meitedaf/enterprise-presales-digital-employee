# capability_matching 测试说明

## 目录目的

本目录用于记录“能力匹配评估 Agent”的测试用例、运行记录和问题修复过程。

能力匹配评估 Agent 位于证据检索 Agent 之后，负责将 evidence_results 转换为售前可用的能力判断，包括：

- 支持等级
- 交付边界
- 技术依赖
- 报价注意事项
- 人工确认项
- 风险标记
- 可用于方案生成的安全表达

该 Agent 不负责重新检索知识库，也不负责生成完整售前方案。

---

## 输入来源

能力匹配评估 Agent 的主要输入为：

text 证据检索 Agent.structured_output 

其中包括：

text evidence_results retrieval_summary 

必要时也可以输入：

text structured_requirement_text 

用于辅助理解客户上下文。

---

## 输出目标

能力匹配评估 Agent 应输出：

json {   "capability_assessments": [],   "assessment_summary": {} } 

每个 capability_assessment 对应一个需求项。

---

## 支持等级定义

### standard_supported

表示有明确、直接证据证明该能力标准支持，且没有明显配置、系统对接、交付审批或人工确认限制。

示例：

text 商品咨询：AI 客服标准支持基于商品知识库回答商品咨询问题。 

### configurable_supported

表示该能力可以通过配置支持，例如知识库配置、规则配置、话术配置、流程配置等。

示例：

text 退换货政策问答：支持，但属于可配置支持能力。 

### integration_required

表示该能力可以支持，但依赖客户系统、API、数据源或第三方接口。

示例：

text 订单状态查询：可以支持，但需要客户订单系统 API 对接。 

### needs_human_confirmation

表示该能力或交付项可评估，但需要人工确认部署条件、交付周期、费用、实施范围、商务条款或复杂技术可行性。

示例：

text 私有化部署：企业版可评估私有化部署，但必须由交付负责人确认。 

### not_supported_or_no_evidence

表示当前没有明确支持证据，或证据明确说明不支持。

示例：

text 当前知识库未找到该能力支持证据。 

---

## 判断优先级

如果同一个需求项同时命中多个条件，按以下优先级判断主支持等级：

text not_supported_or_no_evidence needs_human_confirmation integration_required configurable_supported standard_supported 

注意：

- 主支持等级不等于是否需要人工确认。
- 例如“订单系统对接”主支持等级应为 integration_required，但仍可设置 human_confirmation_required = true。
- 私有化部署主支持等级通常是 needs_human_confirmation。
- “需 API 对接后支持”不能被判为 standard_supported。

---

## 核心检查项

### 1. 支持等级是否正确

检查 Agent 是否能根据证据中的限制条件判断支持等级。

例如：

text 需要 API 对接 -> integration_required 可配置支持 -> configurable_supported 需交付负责人确认 -> needs_human_confirmation 无证据 -> not_supported_or_no_evidence 

### 2. 是否避免过度承诺

Agent 不得承诺：

- 确定报价
- 确定交期
- 确定私有化部署可交付
- 确定客户系统一定可对接
- 无证据能力可支持

### 3. 人工确认项是否完整

以下情况应进入人工确认：

- 私有化部署
- 交付周期
- 费用
- 订单系统 API 是否开放
- API 鉴权方式
- 字段映射
- 调用频率
- 实时性要求
- 定制开发
- 中低质量证据
- 冲突证据

### 4. plan_expression 是否安全

plan_expression 是后续方案生成 Agent 可以使用的安全表达。

示例：

text standard_supported -> 支持该能力 configurable_supported -> 可通过配置支持该能力 integration_required -> 可在完成系统/API 对接后支持 needs_human_confirmation -> 可评估，需进一步确认交付条件、周期和费用 not_supported_or_no_evidence -> 当前知识库未找到明确支持证据，需人工确认后再写入方案 

### 5. plan_eligible 是否合理

一般规则：

- standard_supported 可以进入方案
- configurable_supported 可以进入方案，但应说明配置条件
- integration_required 可以进入方案，但必须说明对接依赖
- needs_human_confirmation 可以进入方案，但必须标记待确认
- not_supported_or_no_evidence 默认不进入方案，除非人工确认后补充

---

