# Dify Chatflow 设计说明

## MVP 流程

```text
Start
-> LLM1：需求梳理 Agent
-> 条件分支：route_decision
   -> need_follow_up：Answer 输出追问问题
   -> technical_matching：进入证据检索
-> Iteration：遍历 search_questions
   -> Knowledge Retrieval：按 query 检索知识库
   -> LLM2：证据检索 Agent，输出单项 evidence_result
-> LLM / Code：汇总 evidence_results 与 retrieval_summary
-> LLM3：能力匹配评估 Agent
-> LLM4：方案生成 Agent
-> LLM5：风险校验 Agent
-> Answer：输出方案、支持等级表、风险与待确认项
```

## 知识库

MVP 阶段使用一个 Dify 知识库：

```text
AI客服售前方案生成_MVP知识库
```

导入来源：

```text
knowledge_docs/
```

## 节点配置记录

后续在此记录：

1. 每个 LLM 节点使用的模型。
2. Knowledge Retrieval 的 TopK、Score Threshold、Rerank 配置。
3. 条件分支字段映射。
4. Iteration 输入输出字段映射。
5. Dify 调试中发现的问题。


# Dify Chatflow 设计说明

## 1. 当前实现目标

本 Chatflow 用于实现“企业售前数字员工”的 MVP 流程：

1. 将客户原始需求转化为结构化需求。
2. 支持多轮追问和需求补齐。
3. 将完整需求拆分为可检索问题。
4. 对每个 search_question 单独进行知识库检索。
5. 基于逐项检索结果生成 evidence_results 和 retrieval_summary。
6. 为后续能力匹配评估、方案生成和风险校验提供证据基础。

当前阶段主要完成：

- 需求梳理 Agent
- 多轮会话状态保存
- 条件分支
- 逐项知识检索
- 证据检索 Agent
- 调试输出 evidence_results

能力匹配评估 Agent、方案生成 Agent、风险校验 Agent 后续继续接入。

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
  │       |
  │       v
  │     追问缺失信息 Answer
  │     输出：clarification.questions
  │     暂停本轮流程，等待用户下一轮补充
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
        调试输出证据结果 Answer
        当前仅用于调试，后续替换为能力匹配评估 Agent
```

---

## 3. 节点命名规范

| 节点类型            | 推荐中文名           | 作用                                                                       |
| ------------------- | -------------------- | -------------------------------------------------------------------------- |
| Start               | 用户输入             | 接收用户本轮输入                                                           |
| LLM                 | 需求梳理 Agent       | 提取结构化需求、缺失字段、追问问题、检索问题和路由决策                     |
| Code                | 格式化需求与检索问题 | 将 structured_requirement 转文本，将 search_questions 转为文本和可迭代数组 |
| Variable Assigner   | 更新需求会话状态     | 保存多轮状态变量                                                           |
| If/Else             | 判断需求是否完整     | 根据 route_decision 进入追问或证据检索                                     |
| Answer              | 追问缺失信息         | 输出 clarification.questions，暂停本轮流程                                 |
| Iteration           | 逐项检索问题循环     | 遍历 search_question_items                                                 |
| Knowledge Retrieval | 单需求项知识检索     | 对当前 search_question 单独检索知识库                                      |
| Code                | 格式化单项检索结果   | 将单次检索结果转成 single_retrieval_text                                   |
| Code                | 合并全部检索结果     | 合并 Iteration 输出，生成 evidence_context_text                            |
| LLM                 | 证据检索 Agent       | 基于 evidence_context_text 输出 evidence_results 和 retrieval_summary      |
| Answer              | 调试输出证据结果     | 临时输出 LLM2 structured_output，正式流程中会替换                          |

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

Chatflow 不在同一轮流程中等待用户补充信息。

当 route_decision = need_follow_up 时：

1. Answer 节点输出追问问题。
2. 当前轮流程结束。
3. 用户下一轮补充信息。
4. Chatflow 重新从 Start 执行。
5. 需求梳理 Agent 读取 requirement_state_text、missing_fields_state 和 last_route_decision。
6. Agent 将历史已确认信息与本轮输入合并后重新判断 required_fields 是否完整。

---

## 5. 需求梳理 Agent

### 输入

- 本轮用户输入：`sys.query`
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
prompts/01_requirement_clarifier_prompt.md v0.4
```

### 当前 Schema 版本

```text
schemas/requirement_clarifier_output.schema.json v0.1 / v0.2
```

说明：Dify 对复杂 JSON Schema 条件约束支持有限，因此 route_decision 与 search_questions 的跨字段一致性主要通过 Prompt 和后续流程 Guardrail 控制。

---

## 6. 格式化需求与检索问题 Code 节点

### 作用

该节点位于需求梳理 Agent 之后、更新会话状态之前。

它负责：

1. 将 `structured_requirement` 转为 `structured_requirement_text`，用于写入会话变量。
2. 将 `search_questions` 转为 `search_questions_text`，供后续 LLM 理解全部检索问题。
3. 将 `search_questions` 转为 `search_question_items`，供 Iteration 节点逐项检索。

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
            ensure_ascii=False
        ),
        "search_questions_text": "\n".join(search_question_lines),
        "search_question_items": search_question_items
    }
```

---

## 7. 更新需求会话状态节点

### 作用

将本轮需求梳理结果保存到会话变量，供下一轮用户补充时继续使用。

### 变量赋值

| 会话变量               | 赋值来源                                                     |
| ---------------------- | ------------------------------------------------------------ |
| requirement_state      | 需求梳理 Agent.structured_requirement                        |
| requirement_state_text | 格式化需求与检索问题.structured_requirement_text             |
| missing_fields_state   | 需求梳理 Agent.clarification.missing_fields / missing_fields |
| last_route_decision    | 需求梳理 Agent.route_decision                                |

说明：`requirement_state` 是 Object，用于流程节点；`requirement_state_text` 是 String，用于 LLM Prompt 读取。

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
为了更准确地生成后续方案，我还需要确认以下信息：

{{clarification.questions}}
```

注意：正式产品中不要输出过多内部字段，例如 missing_fields 可以在调试时输出，但面向用户时建议只输出追问问题。

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

遍历 `search_question_items`，让每个 search_question 单独调用一次 Knowledge Retrieval。

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
AI客服售前方案生成知识库
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
            "single_retrieval_text": "\n".join(blocks)
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
content:
{content}
"""
        blocks.append(block)

    return {
        "single_retrieval_text": "\n\n".join(blocks)
    }
```

---

## 12. 合并全部检索结果 Code 节点

### 作用

将 Iteration 输出的多个 `single_retrieval_text` 合并为一个 `evidence_context_text`，供证据检索 Agent 读取。

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
        "retrieval_group_count": len(cleaned)
    }
```

---

## 13. 证据检索 Agent

### 作用

基于逐项检索后的 `evidence_context_text`，整理每个需求项对应的证据结果。

该 Agent 不负责最终判断支持等级，不生成方案，不承诺报价或交期。

### 输入

- structured_requirement_text
- search_questions_text
- evidence_context_text
- retrieval_group_count

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

## 14. 调试输出节点

当前 `调试输出证据结果 Answer` 仅用于开发调试。

它会直接输出：

```text
证据检索 Agent.structured_output
```

正式产品中不应直接将 evidence_results JSON 原样展示给客户或售前人员。

后续正式流程应改为：

```text
证据检索 Agent
        |
        v
能力匹配评估 Agent
        |
        v
方案生成 Agent
        |
        v
风险校验 Agent
        |
        v
最终汇总输出
```

---

## 15. 知识库

MVP 阶段使用一个 Dify 知识库：

```text
AI客服售前方案生成知识库
```

导入来源：

```text
knowledge_docs/
```

当前建议文档结构：

```text
01_产品能力说明.md
02_技术方案与系统对接说明.md
03_交付边界与人工确认规则.md
04_FAQ_售前常见问题.md
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

## 16. 当前已验证测试

### RC-001 ~ RC-006

用于验证需求梳理 Agent 的单轮字段提取、追问、口语标准化和字段边界。

### RC-007 多轮追问补齐需求

验证点：

- 第一轮进入 need_follow_up。
- 会话变量保存 requirement_state。
- 第二轮读取历史状态并合并本轮补充。
- route_decision 从 need_follow_up 转为 technical_matching。
- 修复 scenario 与 functional_requirements 语义混淆问题。

### ER-001 多轮补齐后的零售 AI 客服证据检索

验证点：

- Iteration 对每个 search_question 单独检索。
- 商品咨询、订单查询、私有化部署、订单系统均有独立召回机会。
- 证据检索 Agent 能识别订单系统 API 依赖。
- 证据检索 Agent 能识别私有化部署需人工确认。
- 证据检索 Agent 能输出 strong / weak / no evidence / need human confirmation 分类。
- 当前需继续优化证据归因，避免弱相关证据被错误归入当前 requirement_item。

---

## 17. 后续待办

1. 优化证据检索 Agent Prompt，强化证据归因边界。
2. 将调试 Answer 替换为能力匹配评估 Agent。
3. 增加能力匹配评估 Agent 的 Schema 和测试集。
4. 增加方案生成 Agent。
5. 增加风险校验 Agent。
6. 增加最终汇总输出节点。
7. 补充更多知识库文档，包括订单系统 API 对接细节、私有化部署环境要求、客服系统对接边界等。
8. 后续考虑开启 Rerank、metadata 过滤和多知识库路由。