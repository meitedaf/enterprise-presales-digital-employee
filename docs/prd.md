# 售前方案生成 Agent PRD

## 1. 项目背景

B2B 售前工作需要同时理解客户需求、企业产品能力、技术实现边界、历史案例和交付约束。现实中，客户需求往往表达模糊，产品资料分散在规格书、FAQ、历史方案、交付说明和报价规则中，售前人员需要花费大量时间查资料、确认能力边界并撰写初版方案。

本项目设计一个基于 Dify Chatflow 的售前方案生成 Agent，用于模拟企业售前数字员工的核心工作流：将销售、售前顾问或解决方案团队输入的客户需求描述、客户沟通记录或会议纪要摘要结构化，基于企业知识库检索证据，评估产品能力匹配关系，生成可供售前人员审核的初版方案，并标记风险和人工确认项。

该项目定位为“售前内部 Copilot”，直接使用者是销售、售前顾问或解决方案团队成员，不是终端客户本人。系统输出用于内部需求梳理、方案准备和人工审核，不直接作为面向客户的正式承诺。

## 2. 目标用户

直接目标用户包括：

- 售前顾问：需要快速理解客户需求并生成初版解决方案。
- AE / 技术支持：需要查询产品规格、技术能力和限制条件。
- 销售人员：需要在早期沟通中获得可参考的方案方向和待客户确认问题。
- 产品 / 交付团队：需要识别客户需求中涉及的定制、系统对接和交付风险。

终端客户不是本 MVP 的直接使用者。客户需求通过销售转述、客户沟通记录、会议纪要摘要、已上传资料或售前人员补充信息进入系统。

## 3. 核心问题

当前售前方案生成过程中存在以下问题：

1. 客户需求在销售沟通记录或售前转述中经常表达不完整，售前需要反复向客户确认应用场景、目标用户、部署方式、系统对接和交付约束。
2. 产品资料、技术文档、历史案例和交付边界分散，人工检索成本高。
3. 单次 RAG 检索容易遗漏复合需求中的子需求，例如订单查询、转人工、私有化部署等能力需要逐项验证。
4. 历史案例、营销材料和正式产品能力容易被混用，导致方案中出现过度承诺。
5. 报价、交期、定制开发、私有化部署和客户系统对接等高风险事项需要人工确认，但容易在初版方案中被弱化或遗漏。

## 4. 产品目标

MVP 阶段目标：

1. 将售前人员输入的客户需求描述、沟通记录或会议纪要摘要转化为结构化需求，并在信息不足时生成“售前需向客户确认的问题”。
2. 将结构化需求拆分为可检索子问题，并通过 Iteration 对每个 search_question 单独检索企业知识库，避免复合需求被一次性 Top K 检索遗漏。
3. 为每个需求项整理产品能力证据、引用来源、相关案例、技术依赖和检索缺口。
4. 基于证据评估需求项支持等级，包括标准支持、需配置、需系统对接、需定制、不支持、证据不足和证据冲突。
5. 生成可供售前人员审核的初版方案，而不是面向客户的最终承诺。
6. 对方案进行风险校验，标记过度承诺、无引用结论、冲突证据和人工确认项。

非目标：

- 不自动生成最终报价。
- 不自动承诺交期。
- 不自动确认定制开发范围。
- 不替代售前、产品或交付负责人做最终决策。

## 5. 核心流程

```text
售前人员输入客户需求描述 / 客户沟通记录 / 会议纪要摘要
        |
        v
需求梳理 Agent
输出：structured_requirement + search_questions + clarification + route_decision + route_reason
        |
        v
格式化需求与检索问题 Code 节点
输出：structured_requirement_text + search_questions_text + search_question_items
        |
        v
更新需求会话状态
保存：requirement_state + requirement_state_text + missing_fields_state + last_route_decision
        |
        v
条件判断
- need_follow_up -> 输出售前需向客户确认的问题，暂停本轮流程，等待售前人员下一轮补充
- technical_matching -> 进入逐项证据检索
        |
        v
逐项检索问题循环 Iteration
输入：search_question_items
        |
        v
单需求项知识检索
每个 search_question 单独调用 Knowledge Retrieval
        |
        v
格式化单项检索结果 Code 节点
输出：single_retrieval_text
        |
        v
合并全部检索结果 Code 节点
输出：evidence_context_text + retrieval_group_count
        |
        v
证据检索 Agent
输出：evidence_results + retrieval_summary
        |
        v
能力匹配评估 Agent
输出：支持等级 + 交付边界 + 报价注意事项 + 待确认项
        |
        v
方案生成 Agent
输出：方案大纲 + 初版方案
        |
        v
风险校验 Agent
输出：过度承诺检查 + 无引用结论检查 + 冲突证据检查 + 人工确认项
        |
        v
最终汇总输出
输出：最终方案 + 支持等级表 + 风险与待确认项 + 引用来源
```

## 6. Agent 模块设计

### 6.1 需求梳理 Agent

职责：

- 解析本轮售前人员输入，并读取上一轮会话状态。
- 合并已收集结构化需求和本轮用户补充信息，支持多轮追问补齐需求。
- 提取业务目标、应用场景、目标用户、功能需求、部署要求、系统对接和交付约束。
- 判断 required_fields 是否完整。
- 信息不足时输出缺失字段和“售前需向客户确认的问题”，并暂停本轮流程。
- 信息充分时生成 search_questions，供后续 Iteration 逐项检索。
- 避免字段语义混淆，例如不能将“订单查询”写入 integration_requirements，不能将“AI客服”写入 functional_requirements，不能将客户行业写入 target_users。
- 不得将“请帮我梳理需求”“帮我生成方案”等售前人员操作意图写入客户需求字段。

关键输出：

- structured_requirement
- search_questions
- clarification
- route_decision
- route_reason
- structured_requirement_text：由格式化需求与检索问题 Code 节点生成，用于写入 requirement_state_text。
- search_questions_text：由格式化需求与检索问题 Code 节点生成，供后续 Agent 理解全部检索问题。
- search_question_items：由格式化需求与检索问题 Code 节点生成，作为 Iteration 输入数组。

### 6.1.1 多轮状态与检索问题格式化节点

职责：

- 将需求梳理 Agent 输出的 structured_requirement 转换为 structured_requirement_text，写入会话变量 requirement_state_text，供下一轮 LLM Prompt 读取。
- 将 search_questions 转换为 search_questions_text，供后续 Agent 理解全部检索问题。
- 将 search_questions 转换为 search_question_items 数组，供 Iteration 节点逐项检索。
- 更新会话变量 requirement_state、requirement_state_text、missing_fields_state 和 last_route_decision。

Dify 实现方式：

- 使用 Code 节点格式化需求与检索问题。
- 使用会话变量保存多轮状态。
- 使用条件分支判断 route_decision，决定进入追问分支或 technical_matching 分支。

关键输出：

- structured_requirement_text
- search_questions_text
- search_question_items
- requirement_state
- requirement_state_text
- missing_fields_state
- last_route_decision

### 6.2 证据检索 Agent

职责：

- 读取 structured_requirement、search_questions_text 和逐项检索后合并得到的 evidence_context_text。
- 按“检索问题 -> 检索结果”分组整理证据。
- 对每个需求项提取 evidence_result，包括证据来源、引用片段、限制条件、技术依赖、检索缺口和可能冲突。
- 汇总生成 evidence_results 和 retrieval_summary。
- 标记强证据、弱证据、无证据项、可能冲突和需要人工确认项。
- 严格控制证据归因，不能将其他需求项的证据挪用到当前需求项。

Dify 实现方式：

- 使用 Iteration 节点遍历 search_question_items。
- 在 Iteration 内部使用 Knowledge Retrieval 节点对每个 search_question 单独检索产品文档、技术方案、FAQ、历史案例和交付边界说明。
- 使用 Code 节点将每个 search_question 的检索结果格式化为 single_retrieval_text。
- 使用 Code 节点合并所有 single_retrieval_text，生成 evidence_context_text 和 retrieval_group_count。
- 使用 Rerank / Weighted Score 提升与当前 search_question 更相关的片段排序。

关键输出：

- evidence_results
- retrieval_summary
- evidence_context_text：由合并全部检索结果 Code 节点生成，作为 LLM2 的证据上下文输入。
- retrieval_group_count：逐项检索问题组数量。

### 6.2.1 证据归因规则

证据检索 Agent 必须遵守以下证据归因规则：

- evidence 中只能放入直接回答当前 requirement_item 的证据。
- 如果证据只与同一业务场景相关，但不直接证明当前需求项，不得作为该 requirement_item 的 evidence。
- 不得把其他需求项的证据挪用到当前需求项。
- “退换货政策问答”不能作为“商品咨询”的直接证据。
- “物流进度查询”不能作为“订单状态查询”的直接证据，除非客户需求明确包含物流查询。
- “客服系统 / 工单系统对接”不能作为“订单系统对接”的直接证据。
- 弱相关证据不能标记为 high quality evidence。

证据质量判断规则：

- high：有直接、明确、同需求项的正式产品能力、技术方案、交付边界或 FAQ 证据。
- medium：有直接相关证据，但包含“可评估”“需确认”“依赖客户系统”“需对接”等限制。
- low：只有弱相关或间接证据。
- none：没有直接证据。
- conflicting：存在明显冲突证据。

### 6.3 能力匹配评估 Agent

职责：

- 基于 evidence_results 和 retrieval_summary 判断每个需求项的支持等级。
- 区分标准支持、可配置支持、需系统对接、需定制、不支持、证据不足和证据冲突。
- 输出交付边界、报价注意事项和人工确认项。
- 决定哪些需求项可以进入方案生成，哪些只能作为待确认或风险项。

关键输出：

- capability_match_results
- delivery_boundary_summary
- unsupported_or_uncertain_items
- plan_eligible_items

### 6.4 方案生成 Agent

职责：

- 基于 structured_requirement、capability_match_results 和 plan_eligible_items 生成初版售前方案。
- 只将已支持、可配置支持、需系统对接且边界明确的能力写入方案主体。
- 对不支持、证据不足、证据冲突或需定制的需求项进行风险提示或待确认处理。

关键输出：

- solution_draft
- capability_mapping_table
- pending_confirmation_items
- excluded_or_risk_items
- suggested_next_steps

### 6.5 风险校验 Agent

职责：

- 检查方案是否存在过度承诺。
- 检查关键结论是否有引用依据。
- 检查是否遗漏交付边界、报价/交期/定制开发确认项。
- 检查历史案例是否被误用为标准能力承诺。
- 输出风险项和修订建议。

关键输出：

- risk_check_results
- revision_suggestions
- final_confirmation_items

## 7. 输入与输出

### 7.1 用户输入

用户输入包括：

- 售前人员输入的客户需求描述。
- 客户沟通记录或会议纪要摘要。
- 售前人员基于客户沟通补充的信息。
- 客户基础信息，例如行业、公司规模、客户角色、销售阶段。
- 已有项目上下文，例如会议纪要、历史沟通记录、已上传资料、已确认约束。

示例：

```text
客户反馈他们想做 AI 客服，目标是降低客服压力。补充信息：客户主要希望给消费者和客服坐席使用，需要支持商品咨询和订单查询，希望私有化部署，并需要对接客户的订单系统。
```

### 7.2 知识库输入

MVP 知识库包括：

- 产品能力文档。
- 技术方案说明。
- FAQ。
- 历史案例。
- 交付边界说明。
- 报价/商务规则摘要。

### 7.3 最终输出

最终输出包括：

- 客户需求摘要。
- 推荐方案。
- 产品能力匹配表。
- 引用来源。
- 不支持 / 待确认项。
- 交付边界和报价注意事项。
- 风险提示。
- 建议下一步。

最终输出定位为“售前可审核的内部初版方案”，不作为面向客户的正式承诺。售前人员需要基于该输出进行审核、删改和确认后，才能转化为客户沟通版本。当前 Dify Chatflow 中的直接回复节点仅用于调试中间结果，例如 evidence_results 的 JSON 输出；正式产品中应在能力匹配评估、方案生成和风险校验后再输出面向售前人员的汇总结果。

## 8. 处理边界与人工确认

### 8.1 需求信息不足

当应用场景、目标用户、核心功能需求、部署方式或系统对接信息缺失时，流程应暂停在需求梳理阶段，输出售前人员需要向客户进一步确认的问题，不进入证据检索和方案生成。售前人员下一轮补充客户信息后，系统应读取已保存的 requirement_state_text、missing_fields_state 和 last_route_decision，将历史已确认信息与本轮输入合并后重新判断。
### 8.2.1 证据归因边界

证据检索 Agent 只能将直接回答当前需求项的知识片段写入 evidence。弱相关、同场景但不同能力、其他系统对接或其他需求项的证据不得被挪用。例如：

- “退换货政策问答”不能作为“商品咨询”的直接证据。
- “物流进度查询”不能作为“订单状态查询”的直接证据，除非客户需求明确包含物流查询。
- “客服系统 / 工单系统对接”不能作为“订单系统对接”的直接证据。

如果检索结果弱相关但不能直接证明当前需求项，应标记为 low evidence 或写入 retrieval_gaps / recommended_next_step，而不是作为 high quality evidence。

### 8.2 证据不足或证据冲突

当知识库未检索到有效证据，或不同文档之间存在支持/不支持、标准版/企业版、旧版本/新版本冲突时，不能自行选择有利结论，必须标记为证据不足或证据冲突，并进入人工确认。

### 8.3 历史案例使用边界

历史案例只能作为参考，不能直接作为当前客户可交付能力的证明。涉及历史定制能力时，必须标记为需定制评估或人工确认。

### 8.4 报价、交期和定制开发

Agent 不得自动承诺报价、交期和定制开发范围。涉及这些内容时，只能输出报价注意事项、交付边界和需人工确认项。

### 8.5 系统对接和私有化部署

涉及客户系统 API、数据源、权限、接口联调、私有化部署环境和安全要求时，方案必须说明客户侧依赖和待确认条件，不能写成开箱即用。

## 9. 评估指标

MVP 使用测试集评估 Agent 效果。测试集应覆盖清晰需求、模糊需求、不支持能力、版本冲突、系统对接和报价/交期高风险场景。

核心评估指标：

1. 需求结构化准确率  
   structured_requirement 是否正确提取客户业务目标、场景、目标用户、功能需求和约束条件。

2. 追问合理性  
   信息不足时，Agent 是否能识别缺失字段，并生成面向售前人员的客户确认问题，例如“请向客户确认 / 请补充客户信息”，而不是把终端客户当作直接使用者。

3. 子问题覆盖率  
   search_questions 是否覆盖主要功能需求、部署要求、系统对接和交付约束，并能被转换为 search_question_items 逐项检索。

4. 引用准确率  
   evidence_results 中的引用是否真实、直接支持对应需求项判断，是否避免将其他需求项或弱相关证据错误归因到当前 requirement_item。

5. 逐项检索覆盖率  
   Iteration 是否对每个 search_question 单独执行知识检索，避免一次性合并检索导致部分需求项被 Top K 挤掉。

6. 不支持能力识别率  
   对知识库明确不支持或无证据支持的需求，Agent 是否能避免写成已支持。

7. 冲突证据识别率  
   当不同文档出现版本、产品线或支持结论冲突时，Agent 是否能标记 conflict_pending。

8. 过度承诺率  
   最终方案中是否出现无依据能力、确定报价、确定交期或未经确认的定制开发承诺。该指标越低越好。

9. 人工确认项召回率  
   报价、交期、私有化部署、系统对接、定制开发、证据不足和证据冲突是否被正确纳入人工确认项。

10. 方案可用性  
    售前人员是否可以基于输出快速修改成客户沟通版本，减少从零起草方案的时间。
