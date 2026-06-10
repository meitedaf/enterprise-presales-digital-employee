# Risk Check Evals

## 目录目的

本目录用于记录“风险校验 Agent”的测试用例、运行记录和问题修复过程。

风险校验 Agent 位于方案生成 Agent 之后，负责检查售前初版方案是否存在过度承诺、无证据能力、支持等级表达错误、系统依赖遗漏、人工确认项遗漏或风险提示遗漏等问题，并输出更安全的最终方案文本。

该 Agent 的输出包括：

- risk_check_result：结构化风险检查结果，用于调试和评估。
- final_safe_response：风险修正后的最终安全方案文本，用于最终 Answer 展示给售前人员。

风险校验 Agent 不负责重新检索知识库，不重新判断产品能力，不新增未被证据或能力匹配支持的能力。

---

## 产品定位

本产品定位为“售前内部 Copilot”。

直接使用者是销售、售前顾问或解决方案团队成员，不是终端客户本人。

因此，风险校验 Agent 输出的 final_safe_response 应面向售前人员，用于内部审核、客户沟通准备和方案准备，不应写成正式客户承诺文件。

---

## 输入来源

风险校验 Agent 的直接输入包括：

text 结构化需求： 格式化需求与检索问题.structured_requirement_text  证据检索结果： 证据检索 Agent.structured_output  能力匹配评估结果： 能力匹配评估 Agent.structured_output  方案结构化结果： 方案生成 Agent.proposal_json  方案文字初稿： 方案生成 Agent.proposal_text_draft 

如果 Dify 无法稳定将 Object 变量插入 Prompt，可以通过 Code 节点将证据检索结果、能力匹配评估结果和方案结构化结果序列化为 JSON 字符串，再统一输入风险校验 Agent。

---

## 输出目标

风险校验 Agent 应输出严格 JSON：

json {   "risk_check_result": {     "passed": true,     "risk_level": "low",     "must_block_output": false,     "issues": [],     "summary": ""   },   "final_safe_response": "" } 

其中：

- risk_check_result.passed 表示风险校验是否通过。
- risk_check_result.risk_level 用于标记风险等级。
- risk_check_result.must_block_output 表示是否必须阻断最终输出。
- risk_check_result.issues 记录发现的问题。
- final_safe_response 是最终安全版本方案。

最终 Answer 节点在正式运行时只输出：

text {{风险校验 Agent.final_safe_response}} 

调试阶段可以同时输出：

text {{风险校验 Agent.risk_check_result}}  {{风险校验 Agent.final_safe_response}} 

---

## 核心检查规则

### 1. 支持等级一致性

风险校验 Agent 必须检查方案文字是否与能力匹配评估结果一致。

text standard_supported -> 可以表达为支持该能力 configurable_supported -> 必须表达为可通过配置支持 integration_required -> 必须表达为完成系统/API/数据源对接后支持 needs_human_confirmation -> 必须表达为可评估，需进一步确认 not_supported_or_no_evidence -> 不得写入推荐方案主体 

如果方案初稿把 integration_required 写成标准支持，或把 needs_human_confirmation 写成确定交付，应记录为风险问题，并在 final_safe_response 中修正。

### 2. 过度承诺检查

以下表达属于高风险，应删除或改写：

text 保证上线 一定可以 确定支持 确定交付 确定周期 确定费用 私有化部署可直接交付 无需客户系统改造 开箱即用支持订单查询 实时查询一定可用 不需要客户配合即可完成 自动完成系统对接 已确认报价 已确认交付周期 

推荐安全表达：

text 可评估，需进一步确认 完成系统/API 对接后支持 需客户提供 API 文档、鉴权方式、字段映射和测试环境后进一步评估 具体部署条件、交付周期和费用需由交付负责人确认 当前不构成最终交付承诺 

### 3. 系统对接依赖检查

如果方案涉及订单查询、物流查询、CRM 查询、工单查询、用户信息查询等外部系统数据能力，必须保留：

- 客户系统需开放 API
- API 文档
- 鉴权方式
- 字段映射
- 测试环境
- 联调条件或联调窗口
- 若客户系统不开放 API，则不能承诺实时查询或自动对接

如果方案初稿遗漏上述依赖，风险校验 Agent 应在 final_safe_response 中补充。

### 4. 私有化部署风险检查

如果方案涉及私有化部署，必须保留：

- 私有化部署仅可评估
- 需交付负责人确认
- 需确认部署环境
- 需确认交付周期
- 需确认费用或报价
- 不能自动承诺

如果方案初稿写成确定交付，风险校验 Agent 必须改写。

### 5. 人工确认项检查

风险校验 Agent 必须检查能力匹配评估结果和方案结构化结果中的人工确认项是否已经体现在最终文本中。

常见人工确认项包括：

- 订单系统是否开放 API
- API 文档
- 鉴权方式
- 字段映射
- 测试环境
- 联调窗口
- 私有化部署条件
- 交付周期
- 费用或报价

### 6. 风险项检查

风险校验 Agent 必须检查风险项是否已经体现在最终文本中。

常见风险项包括：

- 客户系统不开放 API 导致无法支持实时查询
- 缺少字段映射、鉴权方式或数据同步机制
- 私有化部署条件、交付周期和费用未确认
- 无证据能力不得写入方案主体

---

## 风险等级

使用以下风险等级：

text low medium high blocked 

### low

方案基本安全，没有明显过度承诺，最多只有轻微措辞优化。

### medium

存在依赖、风险或待确认项表达不完整，但可以通过改写修复。

### high

存在明显过度承诺、支持等级表达错误、无证据能力写入方案主体等严重问题，但仍可通过改写得到安全版本。

### blocked

存在无法修复的严重问题，不应输出方案。

例如：

- 方案主体完全基于无证据能力。
- 方案包含严重虚假承诺且无法根据能力匹配结果修正。
- 方案无法生成安全版本。

---

## 当前测试集

### RR-001 零售 AI 客服方案安全校验

来源：SG-002 售前内部输入口径端到端方案生成结果。

测试目标：

- 验证风险校验 Agent 能否输出 risk_check_result 和 final_safe_response。
- 验证最终安全方案是否保留完整方案结构。
- 验证订单状态查询是否被安全表达为 API 对接后支持。
- 验证私有化部署是否被安全表达为可评估，需人工确认。
- 验证最终方案是否未出现报价、交期、上线等过度承诺。

预期结果：

text risk_level = low 或 medium must_block_output = false passed = true 

### RR-002 过度承诺表达修正

用于验证风险校验 Agent 是否能识别和修正高风险承诺表达。

覆盖风险：

- 保证上线
- 确定交付周期
- 确定费用
- 私有化部署可直接交付
- 无需客户系统改造即可支持

### RR-003 系统对接依赖遗漏补全

用于验证风险校验 Agent 是否能识别并补全外部系统对接依赖。

覆盖风险：

- 把订单查询写成开箱即用
- 遗漏 API 文档
- 遗漏鉴权方式
- 遗漏字段映射
- 遗漏测试环境
- 遗漏联调条件

---

## 结果标记

使用以下结果：

text Pass Pass with minor issues Partial Pass Fail 

### Pass

风险校验 Agent 输出结构完整，final_safe_response 安全、完整、可直接展示，没有明显遗漏或过度承诺。

### Pass with minor issues

最终方案整体安全，但存在轻微措辞、标题、结构或重复问题，不影响进入最终 Answer。

### Partial Pass

识别了部分风险，但仍有关键依赖、人工确认项或风险提示遗漏，需要继续修 Prompt。

### Fail

未识别严重过度承诺、未修正支持等级错误、输出缺少关键字段，或最终方案仍不安全。