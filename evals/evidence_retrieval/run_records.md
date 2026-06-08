# 证据检索 Agent 运行记录

## 2026-06-08

### 运行环境

- 平台：Dify Chatflow
- 模型：qwen3.6-flash
- Temperature：0.2
- 是否开启结构化输出：是
- Prompt 版本：
  - prompts/02_evidence_retrieval_prompt/evidence_retrieval_prompt_v0.1.md
- Schema 版本：schemas/evidence_retrieval_output.schema.json
- 知识库版本：
  - knowledge_docs/knowledge_docs_v0.1
  - knowledge_docs/knowledge_docs_v0.2
- 检索方式：
  - 当前阶段：将 LLM1 生成的 search_questions 转换为 search_questions_text 后进行一次性知识检索
  - 后续优化方向：Iteration 逐个 search_question 检索

---

## 本轮测试摘要

本轮测试用于验证证据检索 Agent 是否能够基于需求梳理 Agent 输出的 structured_requirement、search_questions，以及 Dify 知识检索节点返回的 retrieval_results，整理出每个需求项对应的产品能力证据、引用来源、技术依赖、限制条件、证据缺口和整体检索摘要。

截至当前记录：

- RC-001：Partial Pass，知识库 v0.1 下召回不完整；知识库 v0.2 已补充相关能力内容，待复测。

测试发现并修复 / 推进了三个关键问题：

1. 知识库覆盖不完整
    ER-001 初次测试中，知识检索节点主要召回了订单查询和物流查询相关内容，未召回商品咨询和私有化部署的直接证据。LLM2 正确将商品咨询和私有化部署标记为 no evidence，没有自行假设支持，说明证据检索 Agent 的边界控制基本有效。问题主要来自知识库 v0.1 覆盖不足和检索召回不完整。
2. 知识检索结果格式不适合直接传给 LLM2
    Dify 知识检索节点返回的是 Array[Object]，包含 content、title、metadata、files 等字段。LLM2 Prompt 不适合直接读取原始 Object，因此增加 Code 节点，将 retrieval_results 转换为 evidence_context_text，突出 source_doc、score、segment_position 和 content，减少 metadata 噪声，提升 LLM2 的证据整理稳定性。
3. 当前仍是一次性合并检索，不是逐项检索
    当前流程将多个 search_questions 合并成 search_questions_text 后进行一次知识检索，因此 Top K 结果可能被某些高相关需求项占用，导致其他需求项漏召回。后续优化方向是使用 Iteration 节点逐个 search_question 检索，使每个需求项都有独立召回结果。


---

## ER-01 多轮补齐后的零售 AI 客服证据检索

### 输入
#### 第一轮输入
我们想做 AI 客服，降低客服压力。

#### 第二轮输入
主要给消费者和客服坐席使用，需要商品咨询和订单查询，私有化部署，并对接订单系统。

### 预期

#### 第一轮预期

- 需求梳理 Agent 应判断信息不足。
- route_decision = need_follow_up。
- search_questions = []。
- clarification.missing_fields 应包含缺失字段。
- clarification.questions 应生成面向客户的追问问题。
- Chatflow 应进入 need_follow_up 分支，通过 Answer 节点向用户追问。
- 会话变量应保存第一轮已提取的结构化需求：
    - requirement_state
    - requirement_state_text
    - missing_fields_state
    - last_route_decision

#### 第二轮预期

需求梳理 Agent 应读取上一轮 requirement_state_text，并结合本轮用户补充信息，生成完整 structured_requirement：

- business_goal = 降低客服压力
- scenario = AI客服
- target_users 包含：
    - 消费者
    - 客服坐席
- functional_requirements 包含：
    - 商品咨询
    - 订单状态查询
- deployment_requirement = 私有化部署
- integration_requirements 包含：
    - 订单系统

当 required_fields 全部明确后：

- route_decision = technical_matching
- clarification.missing_fields = []
- clarification.questions = []
- search_questions 非空
- search_questions 至少覆盖：
    - 商品咨询
    - 订单状态查询
    - 私有化部署
    - 订单系统对接

进入证据检索链路后：

- 知识检索节点应基于 search_questions_text 检索知识库。
- Code 节点应将知识检索节点返回的 Array[Object] 转换为 evidence_context_text。
- 证据检索 Agent 应逐项输出 evidence_results。
- 对每个需求项应判断：
    - 是否找到证据
    - 证据质量
    - 引用来源
    - 原文片段
    - 限制条件
    - 技术依赖
    - 证据缺口
- LLM2 不得直接输出最终支持等级，例如 standard_supported、integration_required、unsupported。
- LLM2 不得在无证据时自行假设产品支持。


### 实际输出

知识检索节点输出摘要

知识检索节点在 knowledge_docs_v0.1 下召回了 3 条结果，主要来自：

- 04_FAQ_售前常见问题.md
- 01_产品能力说明.md

召回内容主要覆盖：

- 物流进度查询
- 订单状态查询
- 物流进度查询相关订单 / 物流接口依赖

未召回直接覆盖：

- 商品咨询
- 私有化部署

证据检索 Agent 输出摘要

证据检索 Agent 输出了 evidence_results，并正确识别：

- 商品咨询：未找到直接证据，evidence_found = false，evidence_quality = none。
- 订单查询：找到直接证据，evidence_found = true，evidence_quality = high，并识别客户订单系统 API 对接依赖。
- 私有化部署：未找到直接证据，evidence_found = false，evidence_quality = none。
- 订单系统：找到中等质量证据，evidence_found = true，evidence_quality = medium，并识别订单系统 API、接口文档、鉴权、字段映射等后续确认项。
  
#### 第一轮实际输出
##### 需求梳理 Agent
```json
{
  "structured_requirement": {
    "business_goal": "降低客服压力",
    "scenario": "AI客服",
    "target_users": [],
    "functional_requirements": [],
    "deployment_requirement": "",
    "integration_requirements": []
  },
  "search_questions": [],
  "clarification": {
    "missing_fields": [
      "target_users",
      "functional_requirements",
      "deployment_requirement",
      "integration_requirements"
    ],
    "questions": [
      "请问该 AI 客服系统的实际使用对象是谁？（例如：终端消费者、内部员工等）",
      "请问 AI 客服具体需要处理哪些业务场景或功能？（例如：商品咨询、订单查询、退换货政策问答等）",
      "请问系统部署方式有何要求？（例如：公有云、私有化部署等）",
      "请问是否需要对接现有的外部系统？（例如：订单系统、CRM 系统等，若不需要请明确说明）"
    ]
  },
  "route_decision": "need_follow_up",
  "route_reason": "缺少 target_users, functional_requirements, deployment_requirement, integration_requirements 字段信息，无法进入技术匹配流程。"
}
```

#### 第一轮 Bot 输出

当前需求信息还不完整，暂时无法进入证据检索流程。

缺失字段：
- target_users
- functional_requirements
- deployment_requirement
- integration_requirements

请补充以下问题：
- 请问该 AI 客服系统的实际使用对象是谁？（例如：终端消费者、内部员工等）
- 请问 AI 客服具体需要处理哪些业务场景或功能？（例如：商品咨询、订单查询、退换货政策问答等）
- 请问系统部署方式有何要求？（例如：公有云、私有化部署等）
- 请问是否需要对接现有的外部系统？（例如：订单系统、CRM 系统等，若不需要请明确说明）

####  第一轮Usage

##### 需求梳理 Agent
```json
{
  "prompt_tokens": 2992,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0005984",
  "completion_tokens": 271,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.000542",
  "total_tokens": 3263,
  "total_price": "0.0011404",
  "currency": "RMB",
  "latency": 3.633,
  "time_to_first_token": 2.206,
  "time_to_generate": 1.426
}
```

#### 第二轮实际输出
##### 需求梳理 Agent
```json
{
  "structured_requirement": {
    "business_goal": "降低客服压力",
    "scenario": "AI客服",
    "target_users": [
      "消费者",
      "客服坐席"
    ],
    "functional_requirements": [
      "商品咨询",
      "订单查询"
    ],
    "deployment_requirement": "私有化部署",
    "integration_requirements": [
      "订单系统"
    ]
  },
  "search_questions": [
    {
      "requirement_item": "商品咨询",
      "query": "AI 客服是否支持基于商品知识库的商品咨询能力，支持条件是什么？"
    },
    {
      "requirement_item": "订单查询",
      "query": "AI 客服是否支持订单状态查询能力，是否需要对接客户订单系统？"
    },
    {
      "requirement_item": "私有化部署",
      "query": "AI 客服是否支持私有化部署，交付条件、限制和人工确认项是什么？"
    },
    {
      "requirement_item": "订单系统",
      "query": "AI 客服对接订单系统需要哪些技术条件和客户侧依赖？"
    }
  ],
  "clarification": {
    "missing_fields": [],
    "questions": []
  },
  "route_decision": "technical_matching",
  "route_reason": "所有 required_fields (business_goal, scenario, target_users, functional_requirements, deployment_requirement, integration_requirements) 均已明确，可以进入技术匹配流程。"
}
```

##### 知识检索 节点
```json

  {
    "metadata": {
      "_source": "knowledge",
      "dataset_id": "3357db5e-3b89-43c6-8907-b1957716daa0",
      "dataset_name": "AI客服售前方案生成知识库",
      "document_id": "c4c54b34-7877-4b44-ace1-c3bd08f9577a",
      "document_name": "04_FAQ_售前常见问题.md",
      "data_source_type": "upload_file",
      "segment_id": "1d141ce6-9356-49a6-ad40-5a649b25defa",
      "retriever_from": "workflow",
      "score": 0.6460791377070737,
      "child_chunks": [],
      "segment_hit_count": 2,
      "segment_word_count": 44,
      "segment_position": 9,
      "segment_index_node_hash": "3374197e183e8d874a5c7006439bed0a42fd5a8b9082567dd21c7e82f45f0a90",
      "doc_metadata": null,
      "position": 1
    },
    "title": "04_FAQ_售前常见问题.md",
    "files": null,
    "content": "Q4：AI 客服是否支持物流进度查询？\n可以评估支持，但通常依赖订单系统或物流系统接口。",
    "summary": null
  },
  {
    "metadata": {
      "_source": "knowledge",
      "dataset_id": "3357db5e-3b89-43c6-8907-b1957716daa0",
      "dataset_name": "AI客服售前方案生成知识库",
      "document_id": "c4c54b34-7877-4b44-ace1-c3bd08f9577a",
      "document_name": "04_FAQ_售前常见问题.md",
      "data_source_type": "upload_file",
      "segment_id": "92ef2d03-6e55-415e-ae8e-7ce01b346b52",
      "retriever_from": "workflow",
      "score": 0.5827251727519387,
      "child_chunks": [],
      "segment_hit_count": 2,
      "segment_word_count": 42,
      "segment_position": 7,
      "segment_index_node_hash": "f5231e029de66a529a34032ba3c27e00ae2b68ee1bfa82f8f47e65f3bb53263c",
      "doc_metadata": null,
      "position": 2
    },
    "title": "04_FAQ_售前常见问题.md",
    "files": null,
    "content": "Q3：AI 客服是否支持订单状态查询？\n可以支持，但需要客户订单系统 API 对接。",
    "summary": null
  },
  {
    "metadata": {
      "_source": "knowledge",
      "dataset_id": "3357db5e-3b89-43c6-8907-b1957716daa0",
      "dataset_name": "AI客服售前方案生成知识库",
      "document_id": "9c6b1bfd-8f26-465f-87c3-d115ae69ee5d",
      "document_name": "01_产品能力说明.md",
      "data_source_type": "upload_file",
      "segment_id": "eb1e5a27-2a15-4e0d-bda4-909fa555d6cc",
      "retriever_from": "workflow",
      "score": 0.49323906711242255,
      "child_chunks": [],
      "segment_hit_count": 1,
      "segment_word_count": 56,
      "segment_position": 22,
      "segment_index_node_hash": "9ea5d4d981c46118d7309deb9c13746d5c9a0b755b1178680fd7e2d61e453e1d",
      "doc_metadata": null,
      "position": 3
    },
    "title": "01_产品能力说明.md",
    "files": null,
    "content": "3.2 物流进度查询\nAI 客服可以支持物流进度查询，但通常需要对接客户订单系统、物流系统或第三方物流查询接口。",
    "summary": null
  }
]
```

##### 证据检索 Agent
```json
{
  "evidence_results": [
    {
      "requirement_item": "商品咨询",
      "query": "商品咨询：AI 客服是否支持基于商品知识库的商品咨询能力，支持条件是什么？",
      "evidence_found": false,
      "evidence_quality": "none",
      "support_summary": "检索结果中未找到关于“商品咨询”或“商品知识库”能力的直接证据。现有证据主要涉及订单和物流查询。",
      "evidence": [],
      "technical_dependencies": [],
      "retrieval_gaps": [
        "缺少关于 AI 客服是否支持基于商品知识库进行商品咨询的能力说明。",
        "缺少商品咨询场景下的支持条件、配置要求或限制说明。"
      ],
      "possible_conflicts": [],
      "recommended_next_step": "需进一步检索产品能力文档中关于‘商品咨询’或‘知识库问答’的具体章节，或确认该功能是否为标准能力。"
    },
    {
      "requirement_item": "订单查询",
      "query": "订单查询：AI 客服是否支持订单状态查询能力，是否需要对接客户订单系统？",
      "evidence_found": true,
      "evidence_quality": "high",
      "support_summary": "AI 客服支持订单状态查询，但明确需要与客户侧的订单系统 API 进行对接。",
      "evidence": [
        {
          "source_doc": "04_FAQ_售前常见问题.md",
          "quote": "Q3：AI 客服是否支持订单状态查询？可以支持，但需要客户订单系统 API 对接。",
          "evidence_type": "faq",
          "limitations_or_conditions": [
            "需要客户订单系统 API 对接"
          ]
        },
        {
          "source_doc": "01_产品能力说明.md",
          "quote": "3.2 物流进度查询 AI 客服可以支持物流进度查询，但通常需要对接客户订单系统、物流系统或第三方物流查询接口。",
          "evidence_type": "product_capability",
          "limitations_or_conditions": [
            "通常需要对接客户订单系统、物流系统或第三方物流查询接口"
          ]
        }
      ],
      "technical_dependencies": [
        "客户订单系统 API 对接",
        "可能需要物流系统或第三方物流查询接口对接"
      ],
      "retrieval_gaps": [],
      "possible_conflicts": [],
      "recommended_next_step": "确认客户订单系统 API 的可用性及对接细节，评估物流查询是否包含在订单查询范围内。"
    },
    {
      "requirement_item": "私有化部署",
      "query": "私有化部署：AI 客服是否支持私有化部署，交付条件、限制和人工确认项是什么？",
      "evidence_found": false,
      "evidence_quality": "none",
      "support_summary": "检索结果中未找到关于“私有化部署”的支持情况、交付条件或限制信息。",
      "evidence": [],
      "technical_dependencies": [],
      "retrieval_gaps": [
        "缺少关于 AI 客服是否支持私有化部署的证据。",
        "缺少私有化部署的交付条件、环境要求、限制及人工确认项。"
      ],
      "possible_conflicts": [],
      "recommended_next_step": "需检索产品部署架构相关文档，确认是否支持私有化部署及其具体技术要求。"
    },
    {
      "requirement_item": "订单系统",
      "query": "订单系统：AI 客服对接订单系统需要哪些技术条件和客户侧依赖？",
      "evidence_found": true,
      "evidence_quality": "medium",
      "support_summary": "对接订单系统需要客户侧提供订单系统 API 接口。此外，若涉及物流查询，可能还需对接物流系统或第三方接口。",
      "evidence": [
        {
          "source_doc": "04_FAQ_售前常见问题.md",
          "quote": "Q3：AI 客服是否支持订单状态查询？可以支持，但需要客户订单系统 API 对接。",
          "evidence_type": "faq",
          "limitations_or_conditions": [
            "需要客户订单系统 API 对接"
          ]
        },
        {
          "source_doc": "01_产品能力说明.md",
          "quote": "3.2 物流进度查询 AI 客服可以支持物流进度查询，但通常需要对接客户订单系统、物流系统或第三方物流查询接口。",
          "evidence_type": "product_capability",
          "limitations_or_conditions": [
            "通常需要对接客户订单系统、物流系统或第三方物流查询接口"
          ]
        }
      ],
      "technical_dependencies": [
        "客户订单系统 API",
        "物流系统接口（如需物流查询）",
        "第三方物流查询接口（可选）"
      ],
      "retrieval_gaps": [
        "缺少具体的 API 字段映射、鉴权方式等技术细节。",
        "缺少对接过程中的数据同步频率或实时性要求说明。"
      ],
      "possible_conflicts": [],
      "recommended_next_step": "获取客户订单系统 API 文档，确认接口可用性、鉴权方式及数据字段映射关系。"
    }
  ],
  "retrieval_summary": {
    "strong_evidence_items": [
      "订单查询"
    ],
    "weak_evidence_items": [
      "订单系统"
    ],
    "no_evidence_items": [
      "商品咨询",
      "私有化部署"
    ],
    "conflicting_items": [],
    "need_human_confirmation": [
      "商品咨询",
      "私有化部署"
    ],
    "recommended_next_step": "针对‘商品咨询’和‘私有化部署’需求，需补充检索相关产品能力与部署文档；针对‘订单系统’对接，需进一步确认 API 技术细节。"
  }
}
```

#### 第二轮 Bot 输出

证据检索结果：

{{
  "evidence_results": [
    {
      "requirement_item": "商品咨询",
      "query": "商品咨询：AI 客服是否支持基于商品知识库的商品咨询能力，支持条件是什么？",
      "evidence_found": false,
      "evidence_quality": "none",
      "support_summary": "检索结果中未找到关于“商品咨询”或“商品知识库”能力的直接证据。现有证据主要涉及订单和物流查询。",
      "evidence": [],
      "technical_dependencies": [],
      "retrieval_gaps": [
        "缺少关于 AI 客服是否支持基于商品知识库进行商品咨询的能力说明。",
        "缺少商品咨询场景下的支持条件、配置要求或限制说明。"
      ],
      "possible_conflicts": [],
      "recommended_next_step": "需进一步检索产品能力文档中关于‘商品咨询’或‘知识库问答’的具体章节，或确认该功能是否为标准能力。"
    },
    {
      "requirement_item": "订单查询",
      "query": "订单查询：AI 客服是否支持订单状态查询能力，是否需要对接客户订单系统？",
      "evidence_found": true,
      "evidence_quality": "high",
      "support_summary": "AI 客服支持订单状态查询，但明确需要与客户侧的订单系统 API 进行对接。",
      "evidence": [
        {
          "source_doc": "04_FAQ_售前常见问题.md",
          "quote": "Q3：AI 客服是否支持订单状态查询？可以支持，但需要客户订单系统 API 对接。",
          "evidence_type": "faq",
          "limitations_or_conditions": [
            "需要客户订单系统 API 对接"
          ]
        },
        {
          "source_doc": "01_产品能力说明.md",
          "quote": "3.2 物流进度查询 AI 客服可以支持物流进度查询，但通常需要对接客户订单系统、物流系统或第三方物流查询接口。",
          "evidence_type": "product_capability",
          "limitations_or_conditions": [
            "通常需要对接客户订单系统、物流系统或第三方物流查询接口"
          ]
        }
      ],
      "technical_dependencies": [
        "客户订单系统 API 对接",
        "可能需要物流系统或第三方物流查询接口对接"
      ],
      "retrieval_gaps": [],
      "possible_conflicts": [],
      "recommended_next_step": "确认客户订单系统 API 的可用性及对接细节，评估物流查询是否包含在订单查询范围内。"
    },
    {
      "requirement_item": "私有化部署",
      "query": "私有化部署：AI 客服是否支持私有化部署，交付条件、限制和人工确认项是什么？",
      "evidence_found": false,
      "evidence_quality": "none",
      "support_summary": "检索结果中未找到关于“私有化部署”的支持情况、交付条件或限制信息。",
      "evidence": [],
      "technical_dependencies": [],
      "retrieval_gaps": [
        "缺少关于 AI 客服是否支持私有化部署的证据。",
        "缺少私有化部署的交付条件、环境要求、限制及人工确认项。"
      ],
      "possible_conflicts": [],
      "recommended_next_step": "需检索产品部署架构相关文档，确认是否支持私有化部署及其具体技术要求。"
    },
    {
      "requirement_item": "订单系统",
      "query": "订单系统：AI 客服对接订单系统需要哪些技术条件和客户侧依赖？",
      "evidence_found": true,
      "evidence_quality": "medium",
      "support_summary": "对接订单系统需要客户侧提供订单系统 API 接口。此外，若涉及物流查询，可能还需对接物流系统或第三方接口。",
      "evidence": [
        {
          "source_doc": "04_FAQ_售前常见问题.md",
          "quote": "Q3：AI 客服是否支持订单状态查询？可以支持，但需要客户订单系统 API 对接。",
          "evidence_type": "faq",
          "limitations_or_conditions": [
            "需要客户订单系统 API 对接"
          ]
        },
        {
          "source_doc": "01_产品能力说明.md",
          "quote": "3.2 物流进度查询 AI 客服可以支持物流进度查询，但通常需要对接客户订单系统、物流系统或第三方物流查询接口。",
          "evidence_type": "product_capability",
          "limitations_or_conditions": [
            "通常需要对接客户订单系统、物流系统或第三方物流查询接口"
          ]
        }
      ],
      "technical_dependencies": [
        "客户订单系统 API",
        "物流系统接口（如需物流查询）",
        "第三方物流查询接口（可选）"
      ],
      "retrieval_gaps": [
        "缺少具体的 API 字段映射、鉴权方式等技术细节。",
        "缺少对接过程中的数据同步频率或实时性要求说明。"
      ],
      "possible_conflicts": [],
      "recommended_next_step": "获取客户订单系统 API 文档，确认接口可用性、鉴权方式及数据字段映射关系。"
    }
  ],
  "retrieval_summary": {
    "strong_evidence_items": [
      "订单查询"
    ],
    "weak_evidence_items": [
      "订单系统"
    ],
    "no_evidence_items": [
      "商品咨询",
      "私有化部署"
    ],
    "conflicting_items": [],
    "need_human_confirmation": [
      "商品咨询",
      "私有化部署"
    ],
    "recommended_next_step": "针对‘商品咨询’和‘私有化部署’需求，需补充检索相关产品能力与部署文档；针对‘订单系统’对接，需进一步确认 API 技术细节。"
  }
}

####  第二轮Usage
##### 需求梳理 Agent
```json
{
  "prompt_tokens": 3016,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0006032",
  "completion_tokens": 351,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.000702",
  "total_tokens": 3367,
  "total_price": "0.0013052",
  "currency": "RMB",
  "latency": 5.614,
  "time_to_first_token": 1.989,
  "time_to_generate": 3.625

```

##### 证据检索 Agent
```json
{
  "prompt_tokens": 1972,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0003944",
  "completion_tokens": 1348,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.002696",
  "total_tokens": 3320,
  "total_price": "0.0030904",
  "currency": "RMB",
  "latency": 9.295,
  "time_to_first_token": 1.934,
  "time_to_generate": 7.36
}
```

### 结果

Partial Pass



### 通过点

- 多轮需求补齐流程已能进入 technical_matching 分支。
- 知识检索节点能够基于 search_questions_text 召回相关知识片段。
- Code 节点能够将 retrieval_results 转换为 LLM2 可读的 evidence_context_text。
- LLM2 能够基于召回结果输出结构化 evidence_results。
- LLM2 正确识别订单查询需要客户订单系统 API 对接。
- LLM2 没有在商品咨询和私有化部署无证据时自行假设支持。
- LLM2 能够输出 retrieval_gaps 和 need_human_confirmation。
- LLM2 没有直接输出最终支持等级，符合证据检索 Agent 的职责边界。

### 问题记录

问题 1：知识库 v0.1 覆盖不足

知识库 v0.1 未能为商品咨询和私有化部署召回直接证据，导致 LLM2 将其标记为 no evidence。

这不是 LLM2 的幻觉问题，而是知识库内容覆盖不足或召回不完整问题。LLM2 在证据不足时输出 retrieval_gaps 是正确行为。

问题 2：当前检索方式不是逐项检索

当前流程将多个 search_questions 合并成 search_questions_text 后进行一次知识检索。由于 Top K 有限，召回结果容易被“订单查询 / 物流查询 / 订单系统”相关内容占用，导致商品咨询、私有化部署等需求项没有召回到直接证据。

问题 3：知识检索节点返回 Array[Object]，不适合直接输入 LLM2

Dify 知识检索节点返回的 result 是 Array[Object]，其中包含 content、title、metadata、files 等字段。直接传入 LLM2 Prompt 不稳定，也会引入较多 metadata 噪声。因此需要 Code 节点将其格式化为 evidence_context_text。

问题 4：调试阶段 Bot 仍直接输出内部 JSON

当前 Answer 节点仍将 LLM2 structured_output 原样输出给用户。这适合调试，但不适合作为正式产品输出。后续应接入能力匹配评估 Agent、方案生成 Agent 或最终汇总 Answer，而不是直接展示 evidence_results 原始 JSON。

### 调整动作

1. 增加知识库内容（未实现，使用 Iteration 后，召回证据完整）
    将知识库从 knowledge_docs_v0.1 升级到 knowledge_docs_v0.2，补充以下内容：
    - 商品咨询能力说明
    - 订单状态查询及订单系统 API 对接说明
    - 私有化部署支持条件、交付边界和人工确认项
    - 退换货政策问答、转人工等后续测试所需能力说明
2. 增加 retrieval_results 格式化 Code 节点
    在 Knowledge Retrieval 节点后增加 Code 节点，将 Array[Object] 转换为 evidence_context_text，供 LLM2 稳定读取。
3. 引入 Iteration 节点，对每个 search_question 进行逐项检索。
4. 后续移除调试 Answer
    LLM2 structured_output 当前仅用于调试，不作为正式用户输出。后续应接入能力匹配评估 Agent，再由最终汇总节点输出面向用户的方案摘要。

---