# Dify Config Notes

本文件记录“企业售前数字员工”MVP 在 Dify Chatflow 中的关键配置，便于后续复现、调试和项目展示。

本项目定位为“售前内部 Copilot”，面向销售、售前顾问和解决方案团队成员，用于将售前人员输入的客户需求描述、沟通记录或补充信息转化为结构化需求，并完成证据检索、能力匹配、方案生成和风险校验。

---

## 1. Knowledge Base

### 知识库名称

```text
AI客服售前方案生成_MVP知识库
```

### 导入目录

```text
knowledge_docs/knowledge_docs_v0.1/
```

### 知识库文档

建议导入以下 Markdown 文档：

```text
01_产品能力说明.md
02_技术方案与系统对接说明.md
03_交付边界与人工确认规则.md
04_FAQ_售前常见问题.md
```

### 切片配置

MVP 阶段建议使用：

| 配置项 | 建议值 |
| --- | --- |
| 分段方式 | 通用 |
| 分段标识符 | `\n\n` |
| 分段最大长度 | `1000 chars` |
| 分段重叠 | `80 chars` |
| 索引方式 | 高质量 |

### 预处理

建议开启：

```text
替换连续空格、换行符和制表符
```

不建议开启：

```text
删除 URL 和 email
```

### 检索配置

| 配置项 | 建议值 |
| --- | --- |
| Top K | 3 |
| 检索方式 | 语义检索 / 混合检索均可，MVP 阶段优先保持简单稳定 |
| Metadata Filter | 暂不启用 |

---

## 2. Chatflow 总体结构

最终 Chatflow 主链路如下：

```text
Start
  |
  v
需求梳理 Agent
  |
  v
格式化需求与检索问题 Code
  |
  v
更新需求会话状态
  |
  v
判断需求是否完整 If/Else
├── need_follow_up
│   └── 追问缺失信息 Answer
│
└── technical_matching
    |
    v
  逐项检索问题 Iteration
    |
    v
  单需求项知识检索 Knowledge Retrieval
    |
    v
  格式化单项检索结果 Code
    |
    v
  合并全部检索结果 Code
    |
    v
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
  最终安全方案输出 Answer
```

---

## 3. Start 节点

| 配置项 | 值 |
| --- | --- |
| 节点类型 | `Start` |
| 输入变量 | `sys.query` |

### 输入口径

用户输入不是终端客户本人直接提交的正式需求，而是售前人员输入的客户需求描述、客户沟通记录、会议纪要摘要或补充客户信息。

示例：

```text
客户反馈他们想做 AI 客服，目标是降低客服压力。请帮我先梳理需求。
```

---

## 4. Conversation Variables

为了支持多轮需求补齐，需要配置以下会话变量。

| 变量名 | 类型 | 用途 |
| --- | --- | --- |
| `requirement_state` | `Object` | 保存当前累计的 `structured_requirement` |
| `requirement_state_text` | `String` | 将 `requirement_state` 序列化为 JSON 字符串，供下一轮需求梳理 Agent 读取 |
| `missing_fields_state` | `Array[String]` | 保存当前仍缺失的 `required_fields` |
| `last_route_decision` | `String` | 保存上一轮路由结果，例如 `need_follow_up` 或 `technical_matching` |

---

## 5. LLM1 需求梳理 Agent

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 需求梳理 Agent |
| 节点类型 | `LLM` |
| 模型 | `qwen3.6-flash` |
| Temperature | `0.2` |
| 结构化输出 | 开启 |
| Prompt 文件 | `prompts/01_requirement_clarifier_prompt/requirement_clarifier_prompt_v0.5.md` |
| Output Schema | `schemas/requirement_clarifier_output.schema.json` |

### 输入变量

Prompt 中应读取：

```text
已收集结构化需求：{{#conversation.requirement_state_text#}}
当前仍缺失字段：{{#conversation.missing_fields_state#}}
上一轮路由结果：{{#conversation.last_route_decision#}}
本轮售前输入：{{#sys.query#}}
```

### 关键职责

- 提取结构化客户需求。
- 判断 `required_fields` 是否完整。
- 信息不足时输出 `route_decision = need_follow_up`。
- 信息完整时输出 `route_decision = technical_matching`。
- 信息不足时 `search_questions = []`。
- 信息完整时生成逐项 `search_questions`。
- 追问话术应面向售前人员，例如“请向客户确认 / 请补充客户信息”。

---

## 6. 格式化需求与检索问题 Code 节点

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 格式化需求与检索问题 |
| 节点类型 | `Code` |
| 输入变量 | `llm_output = 需求梳理 Agent.structured_output` |

### 输出变量

| 变量名 | 类型 |
| --- | --- |
| `structured_requirement_text` | `String` |
| `search_questions_text` | `String` |
| `search_question_items` | `Array[String]` |

### Code

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

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 更新需求会话状态 |
| 节点类型 | `Assign / Variable Assigner` |

### 写入变量

```text
conversation.requirement_state = 需求梳理 Agent.structured_output.structured_requirement
conversation.requirement_state_text = 格式化需求与检索问题.structured_requirement_text
conversation.missing_fields_state = 需求梳理 Agent.structured_output.clarification.missing_fields
conversation.last_route_decision = 需求梳理 Agent.structured_output.route_decision
```

---

## 8. 判断需求是否完整 If/Else 节点

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 判断需求是否完整 |
| 节点类型 | `If/Else` |

### 判断条件

| 条件 | 分支 |
| --- | --- |
| `需求梳理 Agent.structured_output.route_decision == "need_follow_up"` | 进入追问分支 |
| `需求梳理 Agent.structured_output.route_decision == "technical_matching"` | 进入证据检索与方案生成分支 |

---

## 9. 追问缺失信息 Answer 节点

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 追问缺失信息 |
| 节点类型 | `Answer` |

### 输出模板

```text
当前客户需求信息还不完整，暂时无法进入证据检索流程。

请向客户进一步确认，或补充以下客户信息：
{{需求梳理 Agent.structured_output.clarification.questions}}
```

### 注意

该节点只用于 `need_follow_up` 分支。

不应输出：

```text
推荐方案
证据检索结果
能力匹配结果
方案初稿
风险校验结果
```

---

## 10. 逐项检索问题 Iteration 节点

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 逐项检索问题 |
| 节点类型 | `Iteration` |
| 输入数组 | `格式化需求与检索问题.search_question_items` |
| 当前 item | 每个 item 是一条检索问题文本 |

---

## 11. 单需求项知识检索节点

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 单需求项知识检索 |
| 节点类型 | `Knowledge Retrieval` |
| 查询输入 | `逐项检索问题.item` |
| 知识库 | `AI客服售前方案生成_MVP知识库` |

### 检索配置

| 配置项 | 值 |
| --- | --- |
| Top K | 3 |

---

## 12. 格式化单项检索结果 Code 节点

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 格式化单项检索结果 |
| 节点类型 | `Code` |

### 输入变量

| 变量名 | 来源 |
| --- | --- |
| `query` | `逐项检索问题.item` |
| `retrieval_results` | `单需求项知识检索.result` |

### 输出变量

| 变量名 | 类型 |
| --- | --- |
| `single_retrieval_text` | `String` |

### Code

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

## 13. 合并全部检索结果 Code 节点

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 合并全部检索结果 |
| 节点类型 | `Code` |
| 输入变量 | `retrieval_texts = 逐项检索问题.output.single_retrieval_text` |

### 输出变量

| 变量名 | 类型 |
| --- | --- |
| `evidence_context_text` | `String` |
| `retrieval_group_count` | `Number` |

### Code

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

## 14. LLM2 证据检索 Agent

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 证据检索 Agent |
| 节点类型 | `LLM` |
| 模型 | `qwen3.6-flash` |
| Temperature | `0.2` |
| 结构化输出 | 开启 |
| Prompt 文件 | `prompts/02_evidence_retrieval_prompt/evidence_retrieval_prompt_v0.1.md` |
| Output Schema | `schemas/evidence_retrieval_output.schema.json` |

### 输入变量

```text
结构化需求：{{格式化需求与检索问题.structured_requirement_text}}
检索问题：{{格式化需求与检索问题.search_questions_text}}
知识库检索结果：{{合并全部检索结果.evidence_context_text}}
检索组数量：{{合并全部检索结果.retrieval_group_count}}
```

### 关键职责

- 整理每个需求项的证据。
- 提取引用来源和限制条件。
- 标记证据质量。
- 标记检索缺口。
- 不直接生成方案。
- 不做最终能力支持等级判断。

---

## 15. LLM3 能力匹配评估 Agent

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 能力匹配评估 Agent |
| 节点类型 | `LLM` |
| 模型 | `qwen3.6-flash` |
| Temperature | `0.2` |
| 结构化输出 | 开启 |
| Prompt 文件 | `prompts/03_capability_matching_prompt/capability_matching_prompt_v0.1.md` |
| Output Schema | `schemas/capability_matching_output.schema.json` |

### 输入变量

```text
结构化需求：{{格式化需求与检索问题.structured_requirement_text}}
证据检索结果：{{证据检索 Agent.structured_output}}
```

### 关键职责

- 将证据转化为支持等级。
- 输出技术依赖、交付边界和人工确认项。
- 标记风险。
- 判断能力是否可进入方案主体。
- 不生成自然语言方案。

---

## 16. LLM4 方案生成 Agent

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 方案生成 Agent |
| 节点类型 | `LLM` |
| 模型 | `qwen3.6-flash` |
| Temperature | `0.2` |
| 结构化输出 | 开启 |
| Prompt 文件 | `prompts/04_solution_generation_prompt/solution_generation_prompt_v0.2.md` |
| Output Schema | `schemas/proposal_generation_output.schema.json` |

### 输入变量

```text
结构化需求：{{格式化需求与检索问题.structured_requirement_text}}
证据检索结果：{{证据检索 Agent.structured_output}}
能力匹配评估结果：{{能力匹配评估 Agent.structured_output}}
```

如 Dify Object 变量插入不稳定，可在前面增加 Code 节点，将输入统一序列化为字符串。

### 关键职责

- 基于结构化需求、证据和能力匹配结果生成售前初版方案。
- 输出 `proposal_json`。
- 输出 `proposal_text_draft`。
- 不新增无证据能力。
- 不把需要人工确认的能力写成确定支持。
- 不输出最终客户承诺。

---

## 17. LLM5 风险校验 Agent

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 风险校验 Agent |
| 节点类型 | `LLM` |
| 模型 | `qwen3.6-flash` |
| Temperature | `0.1` |
| 结构化输出 | 开启 |
| Prompt 文件 | `prompts/05_risk_review_prompt/risk_review_prompt_v0.1.md` |
| Output Schema | `schemas/risk_review_output.schema.json` |

### 输入变量

```text
结构化需求：{{格式化需求与检索问题.structured_requirement_text}}
证据检索结果：{{证据检索 Agent.structured_output}}
能力匹配评估结果：{{能力匹配评估 Agent.structured_output}}
方案结构化结果：{{方案生成 Agent.proposal_json}}
方案文字初稿：{{方案生成 Agent.proposal_text_draft}}
```

如 Dify Object 变量插入不稳定，可增加“格式化风险校验输入”Code 节点，将上游 Object 序列化为字符串。

### 关键职责

- 检查过度承诺。
- 检查支持等级表达错误。
- 检查系统/API 对接依赖遗漏。
- 检查私有化部署、报价、交期等人工确认边界。
- 检查无证据能力是否被写入推荐方案主体。
- 输出 `risk_check_result`。
- 输出最终可展示的 `final_safe_response`。

---

## 18. 最终安全方案输出 Answer 节点

| 配置项 | 值 |
| --- | --- |
| 节点名称 | 最终安全方案输出 |
| 节点类型 | `Answer` |

### 正式输出模板

```text
{{风险校验 Agent.final_safe_response}}
```

### 调试输出模板

调试阶段可临时使用：

```text
## 风险校验结果

{{风险校验 Agent.risk_check_result}}

---

## 最终安全方案

{{风险校验 Agent.final_safe_response}}
```

正式版本应只输出 `final_safe_response`，不暴露中间 JSON。

---

## 19. 已验证 E2E 用例

当前完整 Chatflow 已通过以下端到端测试：

| Case ID | 结果 |
| --- | --- |
| E2E-001 | Pass |
| E2E-002 | Pass |
| E2E-003 | Pass |
| E2E-004 | Pass |
| E2E-005 | Pass |

覆盖场景：

- 售前内部输入主链路。
- 信息不足时必须阻断。
- 明确不需要系统对接时不得强行生成 API 依赖。
- 外部数据查询必须保留 API 对接边界。
- 不支持能力与报价交期高风险场景。

---

## 20. 注意事项

1. 需求梳理 Agent 是唯一负责决定 `need_follow_up` 和 `technical_matching` 的节点。
2. 当 `route_decision = need_follow_up` 时，后续 RAG、证据检索、能力匹配、方案生成和风险校验都不应执行。
3. 当 `route_decision = technical_matching` 时，必须基于 `search_question_items` 逐项检索，不建议使用整段需求一次性检索。
4. `integration_requirements = ["no_need"]` 时，不应为 `no_need` 生成系统对接类检索问题。
5. 方案生成 Agent 只能基于能力匹配评估结果生成方案，不得新增无证据能力。
6. 风险校验 Agent 是最终输出前的安全兜底，但不应替代前面 Agent 的能力判断。
7. 正式 Answer 只输出 `final_safe_response`。
8. 本项目输出仅用于售前内部准备，不构成正式客户承诺。
