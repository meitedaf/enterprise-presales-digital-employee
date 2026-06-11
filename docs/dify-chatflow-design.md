# Dify Chatflow 设计说明

## 1. 当前实现目标

本 Chatflow 用于实现“企业售前数字员工”的 MVP 流程。

当前产品定位为“售前内部 Copilot”。直接使用者是销售、售前顾问或解决方案团队成员，不是终端客户本人。Start 节点接收的是售前人员输入的客户需求描述、客户沟通记录、会议纪要摘要或补充客户信息。

本 Chatflow 的目标是：

1. 将售前人员输入的客户需求描述、客户沟通记录或会议纪要摘要转化为结构化需求。
2. 支持多轮追问和需求补齐。
3. 在信息不足时暂停后续流程，输出售前人员需向客户进一步确认的问题。
4. 将完整需求拆分为逐项可检索问题。
5. 对每个 search_question 单独进行知识库检索。
6. 基于逐项检索结果生成 evidence_results 和 retrieval_summary。
7. 基于证据结果评估产品能力支持等级、技术依赖、交付边界和人工确认项。
8. 基于结构化需求、证据和能力匹配结果生成售前可审核的初版方案。
9. 在最终输出前进行风险校验，检查过度承诺、无证据能力、系统依赖遗漏、人工确认项遗漏和支持等级表达错误。
10. 最终输出面向售前人员的安全方案文本，不暴露中间 JSON。

当前阶段已完成：

- 需求梳理 Agent
- 多轮会话状态保存
- 条件分支
- 逐项知识检索
- 证据检索 Agent
- 能力匹配评估 Agent
- 方案生成 Agent
- 风险校验 Agent
- 最终安全方案输出
- 分模块测试
- 端到端 E2E 测试

---

## 2. 当前 Chatflow 主流程

```text
Start
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
判断需求是否完整 If/Else
├── need_follow_up
│   |
│   v
│ 追问缺失信息 Answer
│ 输出：clarification.questions
│ 暂停本轮流程，等待售前人员下一轮补充客户信息
│
└── technical_matching
    |
    v
  逐项检索问题循环 Iteration
  输入：search_question_items
    |
    v
  单需求项知识检索 Knowledge Retrieval
  每个 search_question 单独检索知识库
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
  输出：capability_assessments + assessment_summary
    |
    v
  方案生成 Agent
  输出：proposal_json + proposal_text_draft
    |
    v
  风险校验 Agent
  输出：risk_check_result + final_safe_response
    |
    v
  最终安全方案输出 Answer
  输出：final_safe_response
```

---

## 3. 节点命名规范

| 节点类型            | 推荐中文名           | 作用                                                                       |
| ------------------- | -------------------- | -------------------------------------------------------------------------- |
| Start               | 售前输入             | 接收售前人员本轮输入的客户需求描述、沟通记录、会议纪要摘要或补充信息       |
| LLM                 | 需求梳理 Agent       | 提取结构化需求、缺失字段、待客户确认问题、检索问题和路由决策               |
| Code                | 格式化需求与检索问题 | 将 structured_requirement 转文本，将 search_questions 转为文本和可迭代数组 |
| Variable Assigner   | 更新需求会话状态     | 保存多轮状态变量                                                           |
| If/Else             | 判断需求是否完整     | 根据 route_decision 进入追问或证据检索                                     |
| Answer              | 追问缺失信息         | 输出 clarification.questions，暂停本轮流程，等待售前人员补充客户信息       |
| Iteration           | 逐项检索问题循环     | 遍历 search_question_items                                                 |
| Knowledge Retrieval | 单需求项知识检索     | 对当前 search_question 单独检索知识库                                      |
| Code                | 格式化单项检索结果   | 将单次检索结果转成 single_retrieval_text                                   |
| Code                | 合并全部检索结果     | 合并 Iteration 输出，生成 evidence_context_text                            |
| LLM                 | 证据检索 Agent       | 基于 evidence_context_text 输出 evidence_results 和 retrieval_summary      |
| LLM                 | 能力匹配评估 Agent   | 基于 evidence_results 生成能力支持等级、交付边界、技术依赖和待确认项       |
| LLM                 | 方案生成 Agent       | 基于结构化需求、证据和能力匹配结果生成售前可审核的初版方案                 |
| LLM                 | 风险校验 Agent       | 检查方案中的过度承诺、无证据能力、支持等级错误、依赖遗漏和人工确认项遗漏   |
| Answer              | 最终安全方案输出     | 只输出风险校验后的 final_safe_response                                     |

---

## 4. 会话变量

当前使用以下会话变量支持多轮追问：

| 变量名                 | 类型          | 用途                                                             |
| ---------------------- | ------------- | ---------------------------------------------------------------- |
| requirement_state      | Object        | 保存最新 structured_requirement，供流程节点使用                  |
| requirement_state_text | String        | 保存 structured_requirement 的文本版本，供下一轮 LLM Prompt 读取 |
| missing_fields_state   | Array[String] | 保存当前仍缺失字段                                               |
| last_route_decision    | String        | 保存上一轮 route_decision                                        |

### 多轮逻辑

Chatflow 不在同一轮流程中等待售前人员补充客户信息。

当 route_decision = need_follow_up 时：

1. Answer 节点输出售前需向客户确认的问题。
2. 当前轮流程结束。
3. 售前人员下一轮补充客户信息。
4. Chatflow 重新从 Start 执行。
5. 需求梳理 Agent 读取 requirement_state_text、missing_fields_state 和 last_route_decision。
6. Agent 将历史已确认信息与本轮输入合并后，重新判断 required_fields 是否完整。

---

## 5. 需求梳理 Agent

### 输入

- 本轮售前输入：`sys.query`
- 已收集结构化需求：`conversation.requirement_state_text`
- 当前仍缺失字段：`conversation.missing_fields_state`
- 上一轮路由结果：`conversation.last_route_decision`

### 输出

```json
{
  "structured_requirement": {
    "business_goal": "",
    "scenario": "",
    "target_users": [],
    "functional_requirements": [],
    "deployment_requirement": "",
    "integration_requirements": []
  },
  "search_questions": [
    {
      "requirement_item": "",
      "query": ""
    }
  ],
  "clarification": {
    "missing_fields": [],
    "questions": []
  },
  "route_decision": "need_follow_up | technical_matching",
  "route_reason": ""
}
```

### 当前 Prompt 版本

```text
prompts/01_requirement_clarifier_prompt/requirement_clarifier_prompt_v0.5.md
```

### 当前 Schema 版本

```text
schemas/requirement_clarifier_output.schema.json
```

### 关键规则

- 用户输入应理解为售前人员输入的客户需求描述或补充信息。
- 不应将“请帮我梳理需求”“帮我看看”等售前操作意图写入客户需求字段。
- 信息不足时：
  - route_decision = need_follow_up
  - search_questions = []
  - 输出面向售前人员的追问，例如“请向客户确认 / 请补充客户信息”
- 信息完整时：
  - route_decision = technical_matching
  - search_questions 必须非空
  - 检索问题应覆盖功能需求、部署要求和系统对接需求
- 如果客户明确不需要系统对接：
  - integration_requirements = ["no_need"]
  - 不应为 no_need 生成系统对接类 search_question

说明：Dify 对复杂 JSON Schema 条件约束支持有限，因此 route_decision 与 search_questions 的跨字段一致性主要通过 Prompt 和后续流程 Guardrail 控制。

---

## 6. 格式化需求与检索问题 Code 节点

### 作用

该节点位于需求梳理 Agent 之后、更新会话状态之前。

它负责：

1. 将 structured_requirement 转为 structured_requirement_text，用于写入会话变量。
2. 将 search_questions 转为 search_questions_text，供后续 LLM 理解全部检索问题。
3. 将 search_questions 转为 search_question_items，供 Iteration 节点逐项检索。

### 输入

```text
llm_output = 需求梳理 Agent.structured_output
```

### 输出

| 输出变量                    | 类型          | 用途                              |
| --------------------------- | ------------- | --------------------------------- |
| structured_requirement_text | String        | 写入 requirement_state_text       |
| search_questions_text       | String        | 给证据检索 Agent 作为检索问题总览 |
| search_question_items       | Array[String] | 给 Iteration 逐项检索             |

### Python 代码

```python
import json


def main(llm_output: dict) -> dict:
    output = llm_output or {}

    structured_requirement = output.get("structured_requirement", {})
    search_questions = output.get("search_questions", [])

    search_question_lines = []
    search_question_items = []

    if isinstance(search_questions, list):
        for item in search_questions:
            if not isinstance(item, dict):
                continue

            requirement_item = item.get("requirement_item", "").strip()
            query = item.get("query", "").strip()

            if requirement_item and query:
                line = f"{requirement_item}：{query}"
            elif query:
                line = query
            elif requirement_item:
                line = requirement_item
            else:
                continue

            search_question_lines.append(line)
            search_question_items.append(line)

    return {
        "structured_requirement_text": json.dumps(
            structured_requirement,
            ensure_ascii=False,
        ),
        "search_questions_text": "\n".join(search_question_lines),
        "search_question_items": search_question_items,
    }
```

---

## 7. 更新需求会话状态节点

### 作用

将本轮需求梳理结果保存到会话变量，供下一轮用户补充时继续使用。

### 变量赋值

| 会话变量               | 赋值来源                                                      |
| ---------------------- | ------------------------------------------------------------- |
| requirement_state      | 需求梳理 Agent.structured_output.structured_requirement       |
| requirement_state_text | 格式化需求与检索问题.structured_requirement_text              |
| missing_fields_state   | 需求梳理 Agent.structured_output.clarification.missing_fields |
| last_route_decision    | 需求梳理 Agent.structured_output.route_decision               |

说明：

- requirement_state 是 Object，用于流程节点。
- requirement_state_text 是 String，用于 LLM Prompt 读取。

---

## 8. 条件分支节点

### 分支 1：need_follow_up

条件：

```text
route_decision == need_follow_up
```

动作：

```text
追问缺失信息 Answer
```

Answer 内容建议：

```text
当前客户需求信息还不完整，暂时无法进入证据检索流程。

请向客户进一步确认，或补充以下客户信息：
{{需求梳理 Agent.structured_output.clarification.questions}}
```

注意：正式产品中不要输出过多内部字段，例如 missing_fields 可以在调试时输出；面向售前人员时建议只输出待确认问题和少量必要说明。

### 分支 2：technical_matching

条件：

```text
route_decision == technical_matching
```

动作：

```text
进入逐项检索问题循环
```

---

## 9. 逐项检索问题循环 Iteration

### 作用

遍历 search_question_items，让每个 search_question 单独调用一次 Knowledge Retrieval。

这样可以避免把多个问题合并成一个 query 时，被 Top K 挤掉部分需求项。

### 输入

```text
search_question_items = 格式化需求与检索问题.search_question_items
```

### Iteration 内部流程

```text
当前 search_question item
        |
        v
单需求项知识检索
        |
        v
格式化单项检索结果
        |
        v
输出 single_retrieval_text
```

---

## 10. 单需求项知识检索节点

### 作用

在 Iteration 内部，对当前 search_question 单独检索知识库。

### Query

```text
Iteration 当前 item
```

### 当前知识库

```text
AI客服售前方案生成_MVP知识库
```

### 推荐检索配置

| 配置项          | 当前建议           |
| --------------- | ------------------ |
| Top K           | 3                  |
| Score Threshold | 暂不启用或保持默认 |
| Rerank          | 可后续开启         |
| 元数据过滤      | MVP 阶段关闭       |

说明：逐项检索后，每个需求项都有独立 Top K，因此 Top K = 3 已经可以先用于 MVP。

---

## 11. 格式化单项检索结果 Code 节点

### 作用

将单个 search_question 的 Knowledge Retrieval 结果转为 LLM 可读文本。

### 输入

| 输入变量          | 来源                    |
| ----------------- | ----------------------- |
| query             | Iteration 当前 item     |
| retrieval_results | 单需求项知识检索.result |

### 输出

| 输出变量              | 类型   |
| --------------------- | ------ |
| single_retrieval_text | String |

### Python 代码

```python
def main(query: str, retrieval_results: list) -> dict:
    results = retrieval_results or []

    blocks = [f"### 检索问题\n{query}"]

    if not results:
        blocks.append("### 检索结果\n未召回相关知识库片段。")
        return {
            "single_retrieval_text": "\n".join(blocks),
        }

    blocks.append("### 检索结果")

    for idx, item in enumerate(results, start=1):
        if not isinstance(item, dict):
            continue

        title = item.get("title") or ""
        content = item.get("content") or ""
        metadata = item.get("metadata") or {}

        document_name = metadata.get("document_name") or title
        score = metadata.get("score", "")
        segment_position = metadata.get("segment_position", "")
        segment_id = metadata.get("segment_id", "")

        block = f"""[证据 {idx}]
source_doc: {document_name}
title: {title}
score: {score}
segment_position: {segment_position}
segment_id: {segment_id}
content: {content}
"""
        blocks.append(block)

    return {
        "single_retrieval_text": "\n\n".join(blocks),
    }
```

---

## 12. 合并全部检索结果 Code 节点

### 作用

将 Iteration 输出的多个 single_retrieval_text 合并为一个 evidence_context_text，供证据检索 Agent 读取。

### 输入

```text
retrieval_texts = Iteration 输出的 single_retrieval_text 数组
```

### 输出

| 输出变量              | 类型   | 用途            |
| --------------------- | ------ | --------------- |
| evidence_context_text | String | LLM2 证据上下文 |
| retrieval_group_count | Number | 检索问题组数量  |

### Python 代码

```python
def main(retrieval_texts) -> dict:
    if retrieval_texts is None:
        texts = []
    elif isinstance(retrieval_texts, list):
        texts = retrieval_texts
    else:
        texts = [str(retrieval_texts)]

    cleaned = []
    for text in texts:
        if isinstance(text, str) and text.strip():
            cleaned.append(text.strip())

    evidence_context_text = "\n\n---\n\n".join(cleaned)

    return {
        "evidence_context_text": evidence_context_text,
        "retrieval_group_count": len(cleaned),
    }
```

---

## 13. 证据检索 Agent

### 作用

基于逐项检索后的 evidence_context_text，整理每个需求项对应的证据结果。

该 Agent 不负责最终判断支持等级，不生成方案，不承诺报价或交期。

### 输入

- structured_requirement_text
- search_questions_text
- evidence_context_text
- retrieval_group_count

### 当前 Prompt 版本

```text
prompts/02_evidence_retrieval_prompt/evidence_retrieval_prompt_v0.1.md
```

### 当前 Schema 版本

```text
schemas/evidence_retrieval_output.schema.json
```

### 输出

```json
{
  "evidence_results": [
    {
      "requirement_item": "",
      "query": "",
      "evidence_found": true,
      "evidence_quality": "high | medium | low | none | conflicting",
      "support_summary": "",
      "evidence": [
        {
          "source_doc": "",
          "quote": "",
          "evidence_type": "product_capability | technical_solution | delivery_boundary | case_reference | faq | unknown",
          "limitations_or_conditions": []
        }
      ],
      "technical_dependencies": [],
      "retrieval_gaps": [],
      "possible_conflicts": [],
      "recommended_next_step": ""
    }
  ],
  "retrieval_summary": {
    "strong_evidence_items": [],
    "weak_evidence_items": [],
    "no_evidence_items": [],
    "conflicting_items": [],
    "need_human_confirmation": [],
    "recommended_next_step": ""
  }
}
```

### 证据归因规则

1. evidence 中只能放入直接回答当前 requirement_item 的证据。
2. 如果证据只与同一业务场景相关，但不直接证明当前需求项，不得作为该 requirement_item 的 evidence。
3. 不得把其他需求项的证据挪用到当前需求项。
4. “退换货政策问答”不能作为“商品咨询”的直接证据。
5. “物流进度查询”不能作为“订单状态查询”的直接证据，除非客户需求明确包含物流查询。
6. “客服系统 / 工单系统对接”不能作为“订单系统对接”的直接证据。
7. 弱相关证据不能标记为 high quality evidence。

### evidence_quality 判断规则

| 等级        | 含义                                                                |
| ----------- | ------------------------------------------------------------------- |
| high        | 有直接、明确、同需求项的正式产品能力、技术方案、交付边界或 FAQ 证据 |
| medium      | 有直接相关证据，但包含“可评估”“需确认”“依赖客户系统”“需对接”等限制  |
| low         | 只有弱相关或间接证据                                                |
| none        | 没有直接证据                                                        |
| conflicting | 存在明显冲突证据                                                    |

---

## 14. 能力匹配评估 Agent

### 作用

基于证据检索 Agent 输出，将每个需求项转化为售前可使用的能力支持等级、交付边界、技术依赖、人工确认项和风险标记。

该 Agent 不生成自然语言方案，只做能力评估。

### 输入

- structured_requirement_text
- 证据检索 Agent.structured_output

### 当前 Prompt 版本

```text
prompts/03_capability_matching_prompt/capability_matching_prompt_v0.1.md
```

### 当前 Schema 版本

```text
schemas/capability_matching_output.schema.json
```

### 输出核心字段

```json
{
  "capability_assessments": [
    {
      "requirement_item": "",
      "support_level": "standard_supported | configurable_supported | integration_required | needs_human_confirmation | not_supported_or_no_evidence",
      "human_confirmation_required": false,
      "technical_dependencies": [],
      "delivery_boundaries": [],
      "human_confirmation_items": [],
      "risk_flags": [],
      "plan_expression": "",
      "plan_eligible": true
    }
  ],
  "assessment_summary": {}
}
```

### 支持等级

| 支持等级 | 含义 |
| --- | --- |
| `standard_supported` | 有直接证据证明标准支持 |
| `configurable_supported` | 可通过配置、知识库、规则或流程支持 |
| `integration_required` | 可支持，但依赖客户系统、API、数据源或第三方接口对接 |
| `needs_human_confirmation` | 可评估，但需人工确认交付条件、周期、费用、实施范围或商务条款 |
| `not_supported_or_no_evidence` | 无直接支持证据，或证据表明暂不支持 |

### 关键规则

- 系统/API/数据源对接类能力应优先标记为 integration_required，同时可设置 human_confirmation_required = true。
- 不能因为系统对接需要人工确认，就把主要支持等级改成 needs_human_confirmation。
- 私有化部署、报价、交付周期、定制开发范围等事项应标记为 needs_human_confirmation。
- not_supported_or_no_evidence 不应进入推荐方案主体。

---

## 15. 方案生成 Agent

### 作用

基于结构化需求、证据检索结果和能力匹配评估结果，生成售前可审核的初版方案。

该 Agent 输出方案草稿，但不作为最终 Answer 直接展示。方案生成结果必须经过风险校验 Agent。

### 输入

- structured_requirement_text
- 证据检索 Agent.structured_output
- 能力匹配评估 Agent.structured_output

### 当前 Prompt 版本

```text
prompts/04_solution_generation_prompt/solution_generation_prompt_v0.2.md
```

### 当前 Schema 版本

```text
schemas/proposal_generation_output.schema.json
```

### 输出

```json
{
  "proposal_json": {},
  "proposal_text_draft": ""
}
```

### 关键规则

- 只能使用能力匹配结果中 plan_eligible = true 的能力生成方案主体。
- standard_supported 可以表达为“支持该能力”。
- configurable_supported 必须表达为“可通过配置支持”。
- integration_required 必须表达为“完成系统/API/数据源对接后支持”。
- needs_human_confirmation 必须表达为“可评估，需进一步确认”。
- not_supported_or_no_evidence 不得写入推荐方案主体，只能写入未确认或暂不纳入方案的说明。
- 不得承诺报价、交期、上线时间、私有化部署一定可交付。
- 需要对 human_confirmation_items、risk_notes、delivery_boundaries、deployment_notes、excluded_or_unconfirmed_items 进行语义去重。

---

## 16. 风险校验 Agent

### 作用

风险校验 Agent 位于方案生成 Agent 之后、最终 Answer 之前。

它负责检查方案初稿是否存在：

- 过度承诺
- 无证据能力写入方案主体
- 支持等级表达错误
- 系统/API 对接依赖遗漏
- 私有化部署边界遗漏
- 报价、交期、上线等人工确认项遗漏
- 风险提示遗漏

### 输入

- structured_requirement_text
- 证据检索 Agent.structured_output
- 能力匹配评估 Agent.structured_output
- 方案生成 Agent.proposal_json
- 方案生成 Agent.proposal_text_draft

### 当前 Prompt 版本

```text
prompts/05_risk_review_prompt/risk_review_prompt_v0.1.md
```

### 当前 Schema 版本

```text
schemas/risk_review_output.schema.json
```

### 输出

```json
{
  "risk_check_result": {
    "passed": true,
    "risk_level": "low | medium | high | blocked",
    "must_block_output": false,
    "issues": [],
    "summary": ""
  },
  "final_safe_response": ""
}
```

### 关键规则

- integration_required 必须保留系统/API 对接前提。
- needs_human_confirmation 必须保留人工确认边界。
- not_supported_or_no_evidence 不得作为确定能力进入推荐方案主体。
- 不得输出确定报价、确定交期、保证上线、私有化部署可直接交付等表达。
- 最终输出必须是可直接展示给售前人员的自然语言方案。
- 最终输出不应暴露中间 JSON。

---

## 17. 最终安全方案输出 Answer

### 作用

正式输出给售前人员的最终 Answer。

### 输出模板

```text
{{风险校验 Agent.final_safe_response}}
```

正式版本只输出 final_safe_response。

调试阶段可临时输出：

```text
## 风险校验结果

{{风险校验 Agent.risk_check_result}}

---

## 最终安全方案

{{风险校验 Agent.final_safe_response}}
```

调试完成后必须改回只输出 final_safe_response。

---

## 18. 知识库

MVP 阶段使用一个 Dify 知识库：

```text
AI客服售前方案生成_MVP知识库
```

导入来源：

```text
knowledge_docs/knowledge_docs_v0.1/
```

当前建议文档结构：

```text
knowledge_docs_v0.1/
├── 01_产品能力说明.md
├── 02_技术方案与系统对接说明.md
├── 03_交付边界与人工确认规则.md
└── 04_FAQ_售前常见问题.md
```

当前元数据过滤暂不启用。

后续当知识库文档增多或多产品线共用知识库时，可考虑增加 metadata，例如：

```json
{
  "product": "AI客服",
  "doc_type": "product_capability | technical_integration | delivery_boundary | faq | case_study",
  "status": "active",
  "version": "v1.0"
}
```

---

## 19. 当前已验证测试

当前项目已完成分模块测试与端到端测试。

### 需求梳理 Agent

覆盖：

- 信息缺失追问
- 完整需求路由
- 结构化需求抽取
- 字段语义边界
- 多轮状态合并
- 售前内部输入口径

### 证据检索 Agent

覆盖：

- 逐项检索
- 证据归因
- 引用来源整理
- 限制条件提取
- 证据缺口识别

### 能力匹配评估 Agent

覆盖：

- 支持等级判断
- 系统/API 对接能力识别
- 私有化部署人工确认
- 风险标记
- 方案可进入性判断

### 方案生成 Agent

覆盖：

- 生成 proposal_json
- 生成 proposal_text_draft
- 保留能力边界
- 保留系统依赖
- 保留人工确认项
- 语义去重

### 风险校验 Agent

覆盖：

- 过度承诺检查
- 系统依赖遗漏检查
- 私有化部署边界检查
- 报价、交期、上线等人工确认项检查
- 最终安全方案输出

### 端到端 E2E 测试

已通过以下 5 条端到端测试：

| Case ID | 结果 |
| --- | --- |
| E2E-001 | Pass |
| E2E-002 | Pass |
| E2E-003 | Pass |
| E2E-004 | Pass |
| E2E-005 | Pass |

覆盖场景：

1. 售前内部输入主链路。
2. 信息不足时必须阻断，不得假设需求。
3. 明确不需要系统对接时不得强行生成 API 依赖。
4. 外部数据查询必须保留 API 对接边界。
5. 不支持能力与报价交期高风险场景。

---

## 20. 当前状态

当前 Chatflow 已完成 MVP 阶段核心能力验证。

已验证完整闭环：

```text
售前人员转述客户需求
  -> 系统判断信息完整性
  -> 信息不足时生成售前口吻追问
  -> 信息完整后进入证据检索
  -> 基于知识库整理证据
  -> 完成能力匹配评估
  -> 生成售前初版方案
  -> 进行风险校验
  -> 输出最终安全方案
```

当前版本可以用于作品集 / 面试展示，展示一个从需求梳理到安全方案输出的多 Agent 售前 Copilot 工作流。

---

## 21. 后续优化方向

当前 MVP 主链路已跑通，后续可继续优化：

1. 扩展知识库覆盖范围，例如公有云部署、SLA、报价流程、行业案例等。
2. 增加更多行业场景，例如制造、金融、政企、教育等。
3. 增加复杂会议纪要输入测试。
4. 增加批量需求项、多客户角色、多系统对接场景。
5. 引入自动化 Eval 脚本，对结构化输出进行批量断言。
6. 开启 Rerank、metadata 过滤和多知识库路由。
7. 优化最终方案的销售表达质量和可复用性。
8. 增加 Dify DSL 导出文件和部署复现说明。
