# 需求梳理 Agent 运行记录

## 2026-06-07

### 运行环境

- 平台：Dify Chatflow
- 模型：qwen3.6-flash
- Temperature：0.1
- 是否开启结构化输出：是
- Prompt 版本：
  - prompts/01_requirement_clarifier_prompt/requirement_clarifier_prompt_v0.1.md
  - prompts/01_requirement_clarifier_prompt/requirement_clarifier_prompt_v0.2.md
  - prompts/01_requirement_clarifier_prompt/requirement_clarifier_prompt_v0.3.md
  - prompts/01_requirement_clarifier_prompt/requirement_clarifier_prompt_v0.4.md
- Schema 版本：schemas/requirement_clarifier_output.schema.json

---

## 本轮测试摘要

本轮测试用于验证需求梳理 Agent 是否能够稳定完成结构化需求抽取、信息完整性判断、追问生成、search_questions 生成和 route_decision 路由判断。

截至当前记录：

- RC-001：Pass
- RC-002：Pass
- RC-003：Pass
- RC-004：Pass
- RC-005：v0.1 Fail, v0.2 Retest Pass
- RC-006：v0.2 Fail, v0.3 Retest Pass
- RC-006：v0.3 Partial Pass, v0.4 Retest Pass

测试发现并修复了三个关键问题：

1. RC-005：输出契约违反  
   Prompt 已声明 `route_decision = technical_matching` 时必须生成 `search_questions`，但模型仍输出了 `technical_matching` 且 `search_questions = []`。该问题被归类为 Agent 输出契约违反。后续通过增强 Prompt 强约束和输出前自检完成修复。复测结果：Pass。

2. RC-006：字段语义混淆  
   Prompt 已声明不能自行假设目标用户，但模型仍将“客户是一家零售企业”错误补全为 `target_users = ["零售企业客户"]`，并错误进入 `technical_matching`。该问题被归类为字段语义混淆。后续通过增加 target_users 字段级提取规则完成修复：target_users 只能来自用户明确描述的使用对象，客户行业、客户公司、客户类型不能作为 target_users；如果用户未明确说明目标用户，则 target_users 必须为空数组，并且 route_decision 必须为 need_follow_up。复测结果：Pass。

3. RC-007: 字段语义混淆
   scenario 与 functional_requirements 字段语义混淆问题：模型将“商品咨询和订单查询”误填为 scenario。后续通过增加 scenario 字段级提取规则和输出前自检进行修复。复测结果：Pass。
---

## RC-001 信息不足-泛化提效需求

### 输入

我们想做一个 AI 系统，帮助业务提效。

### 预期

- route_decision = need_follow_up
- search_questions = []
- missing_fields 包含 scenario、target_users、functional_requirements、deployment_requirement、integration_requirements
- 生成追问问题

### 实际输出

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
  "search_questions": [],
  "clarification": {
    "missing_fields": [
      "business_goal",
      "scenario",
      "target_users",
      "functional_requirements",
      "deployment_requirement",
      "integration_requirements"
    ],
    "questions": [
      "请描述您的核心业务目标是什么？",
      "您希望这个系统应用于什么具体场景？",
      "谁将是该系统的主要使用者（目标用户）？",
      "您需要系统具备哪些具体功能？",
      "您对系统的部署方式有什么要求（如公有云、私有化等）？",
      "系统是否需要与现有的其他系统进行对接？如果有，请说明对接需求。"
    ]
  },
  "route_decision": "need_follow_up",
  "route_reason": "用户输入中未包含任何结构化需求字段，所有必填字段均缺失，无法进行技术匹配，需进一步追问以获取完整信息。"
}
```

### 结果

Pass

### Usage

```json
{
  "prompt_tokens": 1198,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0002396",
  "completion_tokens": 265,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.00053",
  "total_tokens": 1463,
  "total_price": "0.0007696",
  "currency": "RMB",
  "latency": 7.923,
  "time_to_first_token": 6.09,
  "time_to_generate": 1.833
}
```

### 问题记录

无

### 调整动作

无

---

## RC-002 信息充分-零售AI客服需求

### 输入

客户是一家连锁零售企业，想做 AI 客服，目标是降低人工客服重复咨询压力。使用对象是消费者和客服坐席。需要支持商品咨询、订单状态查询、退换货政策问答和投诉转人工。希望私有化部署，并需要对接订单系统和现有客服系统。

### 预期

- route_decision = technical_matching
- missing_fields = []
- questions = []
- search_questions 覆盖商品咨询、订单查询、退换货政策、转人工、私有化部署、系统对接

### 实际输出

```json
{
  "structured_requirement": {
    "business_goal": "降低人工客服重复咨询压力",
    "scenario": "连锁零售企业AI客服场景",
    "target_users": [
      "消费者",
      "客服坐席"
    ],
    "functional_requirements": [
      "商品咨询",
      "订单状态查询",
      "退换货政策问答",
      "投诉转人工"
    ],
    "deployment_requirement": "私有化部署",
    "integration_requirements": [
      "对接订单系统",
      "对接现有客服系统"
    ]
  },
  "search_questions": [
    {
      "requirement_item": "业务目标",
      "query": "AI客服降低人工重复咨询压力的解决方案"
    },
    {
      "requirement_item": "应用场景",
      "query": "连锁零售企业AI客服应用场景"
    },
    {
      "requirement_item": "功能需求-商品咨询",
      "query": "AI客服商品咨询能力"
    },
    {
      "requirement_item": "功能需求-订单状态查询",
      "query": "AI客服订单状态查询能力"
    },
    {
      "requirement_item": "功能需求-退换货政策问答",
      "query": "AI客服退换货政策问答能力"
    },
    {
      "requirement_item": "功能需求-投诉转人工",
      "query": "AI客服投诉转人工能力"
    },
    {
      "requirement_item": "部署要求",
      "query": "AI客服私有化部署方案"
    },
    {
      "requirement_item": "集成需求-订单系统",
      "query": "AI客服对接订单系统集成方案"
    },
    {
      "requirement_item": "集成需求-客服系统",
      "query": "AI客服对接现有客服系统集成方案"
    }
  ],
  "clarification": {
    "missing_fields": [],
    "questions": []
  },
  "route_decision": "technical_matching",
  "route_reason": "所有必填字段均已明确，可进入技术匹配流程。"
}
```

### 结果

Pass

### Usage

```json
{
  "prompt_tokens": 1252,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0002504",
  "completion_tokens": 491,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.000982",
  "total_tokens": 1743,
  "total_price": "0.0012324",
  "currency": "RMB",
  "latency": 4.365,
  "time_to_first_token": 0.945,
  "time_to_generate": 3.421
}
```

### 问题记录

无

### 调整动作

无

---

## RC-003 缺部署方式

### 输入

客户是一家零售企业，想做 AI 客服，目标是减少客服重复咨询。使用对象是消费者和客服坐席。需要支持商品咨询、订单查询和退换货政策问答，并对接订单系统。

### 预期

- route_decision = need_follow_up
- search_questions = []
- missing_fields 包含 deployment_requirement
- clarification.questions 需要追问部署方式
- 不应进入 technical_matching

### 实际输出

```json
{
  "structured_requirement": {
    "business_goal": "减少客服重复咨询",
    "scenario": "零售企业AI客服场景",
    "target_users": [
      "消费者",
      "客服坐席"
    ],
    "functional_requirements": [
      "商品咨询",
      "订单查询",
      "退换货政策问答"
    ],
    "deployment_requirement": "",
    "integration_requirements": [
      "对接订单系统"
    ]
  },
  "search_questions": [],
  "clarification": {
    "missing_fields": [
      "deployment_requirement"
    ],
    "questions": [
      "请问您希望AI客服系统采用何种部署方式（例如：公有云、私有化部署或混合部署）？"
    ]
  },
  "route_decision": "need_follow_up",
  "route_reason": "缺少部署要求信息，无法进入后续技术匹配流程。"
}
```

### 结果

Pass

### Usage

```json
{
  "prompt_tokens": 1235,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.000247",
  "completion_tokens": 215,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.00043",
  "total_tokens": 1450,
  "total_price": "0.000677",
  "currency": "RMB",
  "latency": 2.547,
  "time_to_first_token": 1.104,
  "time_to_generate": 1.442
}
```

### 问题记录

无

### 调整动作

无

---

## RC-004 缺系统对接需求

### 输入

客户是一家零售企业，想做 AI 客服，目标是减少客服重复咨询。使用对象是消费者和客服坐席。需要支持商品咨询、订单查询和退换货政策问答，希望私有化部署。

### 预期

- route_decision = need_follow_up
- search_questions = []
- missing_fields 包含 integration_requirements
- clarification.questions 需要追问是否需要对接订单系统、客服系统或其他业务系统
- 不应自行假设客户需要对接订单系统
- 不应进入 technical_matching

### 实际输出

```json
{
  "structured_requirement": {
    "business_goal": "减少客服重复咨询",
    "scenario": "零售企业AI客服场景",
    "target_users": [
      "消费者",
      "客服坐席"
    ],
    "functional_requirements": [
      "商品咨询",
      "订单查询",
      "退换货政策问答"
    ],
    "deployment_requirement": "私有化部署",
    "integration_requirements": []
  },
  "search_questions": [],
  "clarification": {
    "missing_fields": [
      "integration_requirements"
    ],
    "questions": [
      "请明确AI客服系统需要对接哪些现有系统（如ERP、CRM、订单系统等）？"
    ]
  },
  "route_decision": "need_follow_up",
  "route_reason": "缺少系统对接需求信息，无法进入技术匹配流程。"
}
```

### 结果

Pass

### Usage

```json
{
  "prompt_tokens": 1235,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.000247",
  "completion_tokens": 205,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.00041",
  "total_tokens": 1440,
  "total_price": "0.000657",
  "currency": "RMB",
  "latency": 2.639,
  "time_to_first_token": 1.205,
  "time_to_generate": 1.435
}
```

### 问题记录

无

### 调整动作

无

---

## RC-005 口语化表达标准化

### 输入

客户是连锁零售企业，想上一个客服机器人，主要给买东西的用户用，目标是减少人工重复回复。功能上要能问商品、查订单进度、问退货规则，遇到投诉能接人工。希望私有化，还要连他们自己的订单系统和客服系统。

### 预期

- route_decision = technical_matching
- clarification.missing_fields = []
- clarification.questions = []
- functional_requirements 应包含：
  - 商品咨询
  - 订单状态查询
  - 退换货政策问答
  - 转人工
- integration_requirements 应包含：
  - 订单系统
  - 客服系统
- search_questions 不为空
- search_questions 应按需求项逐项生成，而不是合并成一个宽泛问题
- 口语化表达应被标准化：
  - “问商品” → 商品咨询
  - “查订单进度” → 订单状态查询
  - “问退货规则” → 退换货政策问答
  - “接人工” → 转人工

### 实际输出

```json
{
  "structured_requirement": {
    "business_goal": "减少人工重复回复",
    "scenario": "连锁零售企业客服场景",
    "target_users": [
      "购买商品的客户"
    ],
    "functional_requirements": [
      "商品咨询",
      "订单进度查询",
      "退货规则咨询",
      "投诉转人工"
    ],
    "deployment_requirement": "私有化部署",
    "integration_requirements": [
      "订单系统对接",
      "客服系统对接"
    ]
  },
  "search_questions": [],
  "clarification": {
    "missing_fields": [],
    "questions": []
  },
  "route_decision": "technical_matching",
  "route_reason": "所有必填字段均已明确，可进入后续知识库检索流程。"
}
```

### 结果

Fail

### Usage

```json
{
  "prompt_tokens": 1249,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0002498",
  "completion_tokens": 189,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.000378",
  "total_tokens": 1438,
  "total_price": "0.0006278",
  "currency": "RMB",
  "latency": 4.937,
  "time_to_first_token": 3.52,
  "time_to_generate": 1.417
}
```

### 问题记录

"route_decision"="technical_matching", 但是"search_questions":[]，为空。

### 根因分析

该问题不是 Prompt 完全缺失规则。Prompt 中已经声明 route_decision = technical_matching 时必须生成 search_questions，但模型仍然输出了 technical_matching 且 search_questions = []。

因此问题属于 Agent 输出契约未被可靠保证，具体包括：

1. Prompt 约束是软约束，模型在复杂结构化输出中可能漏执行部分跨字段规则。
2. 结构化输出 Schema 只约束字段类型，没有表达 `route_decision = technical_matching` 时 `search_questions` 必须非空的条件关系。
3. 缺少输出前自检，模型未检查 route_decision 与 search_questions 的一致性。
4. Chatflow 流程层缺少 Guardrail，未在 technical_matching 但 search_questions 为空时阻断下游流程。

### 调整动作

1. Prompt 增加强约束：
   - route_decision = technical_matching 时，search_questions 必须为非空数组。
   - 如果无法生成 search_questions，则不得输出 technical_matching。
2. Prompt 增加输出前自检：
   - technical_matching 时检查 search_questions 是否非空。
   - 检查每个 functional_requirement、deployment_requirement、integration_requirement 是否有对应 search_question。
3. Schema / 后处理增强：
   - 如果平台支持条件 Schema，则增加 route_decision 与 search_questions 的条件约束。
   - 如果平台不支持，则通过后处理节点校验。
4. Dify 流程增加 Guardrail：
   - route_decision = technical_matching 且 search_questions 为空时，不进入证据检索。
   - 触发重新生成、返回错误提示或转为 need_follow_up。

### 复测记录：v0.2 Prompt 优化后

#### 复测时间

2026-06-07

#### Prompt 版本

prompts/01_requirement_clarifier_prompt/requirement_clarifier_prompt_v0.2.md

#### 优化内容

1. 增加 `route_decision = technical_matching` 时 `search_questions` 必须为非空数组的强约束。
2. 增加如果无法生成 `search_questions`，不得输出 `technical_matching` 的约束。
3. 增加输出前自检：
   - technical_matching 时检查 search_questions 是否非空。
   - 检查每个 functional_requirement 是否有对应 search_question。
   - 检查 deployment_requirement 是否有对应 search_question。
   - 检查每个 integration_requirement 是否有对应 search_question。

#### 复测实际输出

```json
{
  "structured_requirement": {
    "business_goal": "减少人工重复回复",
    "scenario": "连锁零售企业客服场景",
    "target_users": [
      "购买商品的客户"
    ],
    "functional_requirements": [
      "商品咨询能力",
      "订单状态查询能力",
      "退货规则查询能力",
      "投诉转人工能力"
    ],
    "deployment_requirement": "私有化部署",
    "integration_requirements": [
      "对接订单系统",
      "对接客服系统"
    ]
  },
  "search_questions": [
    {
      "requirement_item": "业务目标：减少人工重复回复",
      "query": "客服机器人如何降低人工重复回复率"
    },
    {
      "requirement_item": "功能需求：商品咨询",
      "query": "电商客服机器人商品咨询功能实现方案"
    },
    {
      "requirement_item": "功能需求：订单状态查询",
      "query": "客服机器人订单状态查询接口集成指南"
    },
    {
      "requirement_item": "功能需求：退货规则查询",
      "query": "客服机器人退货政策自动问答配置方法"
    },
    {
      "requirement_item": "功能需求：投诉转人工",
      "query": "智能客服机器人投诉场景转人工策略"
    },
    {
      "requirement_item": "部署要求：私有化部署",
      "query": "客服机器人私有化部署架构与实施步骤"
    },
    {
      "requirement_item": "对接需求：订单系统",
      "query": "客服机器人对接企业订单系统API规范"
    },
    {
      "requirement_item": "对接需求：客服系统",
      "query": "客服机器人对接现有客服工单系统方案"
    }
  ],
  "clarification": {
    "missing_fields": [],
    "questions": []
  },
  "route_decision": "technical_matching",
  "route_reason": "所有必填字段均已明确，且已生成对应的检索问题，可进入后续流程。"
}
```

### 复测结果

Pass

### 复测Usage

```json
{
  "prompt_tokens": 1422,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0002844",
  "completion_tokens": 488,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.000976",
  "total_tokens": 1910,
  "total_price": "0.0012604",
  "currency": "RMB",
  "latency": 4.436,
  "time_to_first_token": 1.228,
  "time_to_generate": 3.208
}
```

### 复测结论

v0.2 Prompt 修复了 RC-005 中 technical_matching 但 search_questions 为空的问题。复测结果中 route_decision = technical_matching，且 search_questions 已按功能需求、部署要求和系统对接需求生成，满足下游证据检索 Agent 的输入要求。

### Schema v0.2 条件约束验证

#### 测试目的

验证 `schemas/requirement_clarifier_output.schema.json v0.2` 是否能够拦截以下非法输出：

- `route_decision = technical_matching`
- `search_questions = []`

按照 v0.2 Schema 设计，`technical_matching` 时 `search_questions` 应满足 `minItems = 1`。

v0.2 Schema:

```json
{
  "type": "object",
  "properties": {
    "structured_requirement": {
      "type": "object",
      "properties": {
        "business_goal": {
          "type": "string"
        },
        "scenario": {
          "type": "string"
        },
        "target_users": {
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "functional_requirements": {
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "deployment_requirement": {
          "type": "string"
        },
        "integration_requirements": {
          "type": "array",
          "items": {
            "type": "string"
          }
        }
      },
      "required": [
        "business_goal",
        "scenario",
        "target_users",
        "functional_requirements",
        "deployment_requirement",
        "integration_requirements"
      ],
      "additionalProperties": false
    },
    "search_questions": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "requirement_item": {
            "type": "string"
          },
          "query": {
            "type": "string"
          }
        },
        "required": [
          "requirement_item",
          "query"
        ],
        "additionalProperties": false
      }
    },
    "clarification": {
      "type": "object",
      "properties": {
        "missing_fields": {
          "type": "array",
          "items": {
            "type": "string"
          }
        },
        "questions": {
          "type": "array",
          "items": {
            "type": "string"
          }
        }
      },
      "required": [
        "missing_fields",
        "questions"
      ],
      "additionalProperties": false
    },
    "route_decision": {
      "type": "string",
      "enum": [
        "need_follow_up",
        "technical_matching"
      ]
    },
    "route_reason": {
      "type": "string"
    }
  },
  "required": [
    "structured_requirement",
    "search_questions",
    "clarification",
    "route_decision",
    "route_reason"
  ],
  "additionalProperties": false,
  "allOf": [
    {
      "if": {
        "properties": {
          "route_decision": {
            "const": "technical_matching"
          }
        },
        "required": [
          "route_decision"
        ]
      },
      "then": {
        "properties": {
          "search_questions": {
            "minItems": 1
          },
          "clarification": {
            "properties": {
              "missing_fields": {
                "maxItems": 0
              },
              "questions": {
                "maxItems": 0
              }
            }
          }
        }
      }
    },
    {
      "if": {
        "properties": {
          "route_decision": {
            "const": "need_follow_up"
          }
        },
        "required": [
          "route_decision"
        ]
      },
      "then": {
        "properties": {
          "search_questions": {
            "maxItems": 0
          },
          "clarification": {
            "properties": {
              "missing_fields": {
                "minItems": 1
              },
              "questions": {
                "minItems": 1
              }
            }
          }
        }
      }
    }
  ]
}
```

#### 测试配置

- Prompt 版本：v0.1
- Schema 版本：v0.2
- 模型：qwen3.5-flash
- 测试用例：RC-005

#### 实际输出

```json
{
  "structured_requirement": {
    "business_goal": "减少人工重复回复",
    "scenario": "连锁零售企业客服场景",
    "target_users": [
      "购买商品的客户"
    ],
    "functional_requirements": [
      "商品咨询",
      "订单进度查询",
      "退货规则咨询",
      "投诉转人工"
    ],
    "deployment_requirement": "私有化部署",
    "integration_requirements": [
      "订单系统对接",
      "客服系统对接"
    ]
  },
  "search_questions": [],
  "clarification": {
    "missing_fields": [],
    "questions": []
  },
  "route_decision": "technical_matching",
  "route_reason": "所有必填字段均已明确，可进入后续技术匹配流程。"
}
```

#### 测试结果

Fail

#### 结论

Dify 当前结构化输出未能强制执行 v0.2 Schema 中的跨字段条件约束。虽然 Schema 中定义了 route_decision = technical_matching 时 search_questions.minItems = 1，但实际运行中仍返回了 search_questions = []。

因此，Schema v0.2 不能作为该业务规则的唯一防线。后续需要在流程层增加 Guardrail，例如通过条件分支或 Code 节点校验：

- technical_matching 且 search_questions 为空时，不允许进入证据检索；
- need_follow_up 且 search_questions 非空时，不允许继续执行；
- 必要时触发重新生成或返回错误提示。

---

## RC-006 不得自行假设目标用户

### 输入

客户是一家零售企业，想做 AI 客服，目标是降低人工客服压力。需要支持商品咨询和订单查询。希望私有化部署，并对接订单系统。

### 预期

- route_decision = need_follow_up
- search_questions = []
- missing_fields 包含 target_users
- clarification.questions 需要追问主要使用对象是谁
- 不能自行将 target_users 补全为“消费者”或“客服坐席”
- 不应进入 technical_matching

### 实际输出

```json
{
  "structured_requirement": {
    "business_goal": "降低人工客服压力",
    "scenario": "零售企业AI客服场景",
    "target_users": [
      "零售企业客户"
    ],
    "functional_requirements": [
      "商品咨询",
      "订单查询"
    ],
    "deployment_requirement": "私有化部署",
    "integration_requirements": [
      "对接订单系统"
    ]
  },
  "search_questions": [
    {
      "requirement_item": "业务目标：降低人工客服压力",
      "query": "AI客服如何降低人工客服压力"
    },
    {
      "requirement_item": "功能需求：商品咨询",
      "query": "AI客服商品咨询能力"
    },
    {
      "requirement_item": "功能需求：订单查询",
      "query": "AI客服订单状态查询能力"
    },
    {
      "requirement_item": "部署要求：私有化部署",
      "query": "AI客服私有化部署方案"
    },
    {
      "requirement_item": "集成需求：对接订单系统",
      "query": "AI客服对接订单系统集成方案"
    }
  ],
  "clarification": {
    "missing_fields": [],
    "questions": []
  },
  "route_decision": "technical_matching",
  "route_reason": "所有必填字段均已明确，可进入技术匹配流程。"
}
```

### 结果

Fail

### Usage

```json
{
  "prompt_tokens": 1400,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.00028",
  "completion_tokens": 337,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.000674",
  "total_tokens": 1737,
  "total_price": "0.000954",
  "currency": "RMB",
  "latency": 3.61,
  "time_to_first_token": 1.173,
  "time_to_generate": 2.436
}
```

### 问题记录

模型将“客户是一家零售企业”错误理解为目标用户，并输出：

```json
"target_users": ["零售企业客户"]
```
但用户原文只说明了客户行业，没有明确 AI 客服的实际使用对象。因此该输出违反了“不能自行假设目标用户”的规则。

但该规则属于抽象约束，未明确区分“客户行业 / 客户公司”与“目标用户”，因此模型仍将客户行业背景错误映射为 target_users。

同时，由于模型错误补全了 target_users，导致 route_decision 被错误设置为 technical_matching，并生成了 search_questions，错误推进到后续流程。

### 根因分析

1. Prompt 中虽然要求不能自行假设目标用户，但没有明确区分“客户行业 / 客户公司”和“目标用户”。
2. 模型将“零售企业”这种客户基础信息误映射为 target_users，发生字段语义混淆。
3. Schema 只能保证 target_users 是数组，无法判断其中的内容是否来自用户明确表达。
4. 缺少输出前自检，没有检查 target_users 是否由用户原文明确给出，而非从行业背景推断。

### 调整动作

1. Prompt 增加目标用户提取规则：
    - target_users 只能来自用户明确描述的使用对象。
    - 客户行业、客户公司、客户类型不能作为 target_users。
    - 如果用户未明确说明目标用户，target_users 必须为空数组。
    - target_users 缺失时，route_decision 必须为 need_follow_up。
2. Prompt 增加输出前自检：
    - 检查 target_users 是否由用户明确表达。
    - 如果 target_users 来自客户行业或模型推断，必须清空 target_users，并将 target_users 加入 missing_fields。
3. 后续 Guardrail 增强：
    - 如果 target_users 的值包含“企业客户”“行业客户”等泛化表达，标记为疑似推断，需要人工或规则校验。

### 复测记录：v0.2 Prompt 优化后

#### 复测时间

2026-06-07

#### Prompt 版本

prompts/01_requirement_clarifier_prompt/requirement_clarifier_prompt_v0.3.md

#### 优化内容

Prompt 增加目标用户提取规则：
  - target_users 只能来自用户明确描述的使用对象。
  - 客户行业、客户公司、客户类型不能作为 target_users。
  - 如果用户未明确说明目标用户，target_users 必须为空数组。
  - target_users 缺失时，route_decision 必须为 need_follow_up。

#### 复测实际输出

```json
{
  "structured_requirement": {
    "business_goal": "降低人工客服压力",
    "scenario": "AI 客服场景，支持商品咨询和订单查询",
    "target_users": [],
    "functional_requirements": [
      "商品咨询能力",
      "订单查询能力"
    ],
    "deployment_requirement": "私有化部署",
    "integration_requirements": [
      "对接订单系统"
    ]
  },
  "search_questions": [],
  "clarification": {
    "missing_fields": [
      "target_users"
    ],
    "questions": [
      "请问该 AI 客服系统的实际使用对象是谁？例如是消费者、客服坐席还是其他人员？"
    ]
  },
  "route_decision": "need_follow_up",
  "route_reason": "目标用户（target_users）字段缺失，无法确定系统的实际使用对象，需进一步追问。"
}
```

### 复测结果

Pass

### 复测Usage

```json
{
  "prompt_tokens": 1583,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0003166",
  "completion_tokens": 211,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.000422",
  "total_tokens": 1794,
  "total_price": "0.0007386",
  "currency": "RMB",
  "latency": 4.7,
  "time_to_first_token": 2.991,
  "time_to_generate": 1.708
}
```

### 复测结论

v0.3 Prompt 修复了 RC-006 中 模型将客户行业背景错误映射为 target_users的问题。复测结果 "missing_fields":["target_users"],"questions":["请问该 AI 客服系统的实际使用对象是谁？例如是消费者、客服坐席还是其他人员？"]已修复。



## RC-007 多轮追问补齐需求

### 输入
#### 第一轮输入
我们想做 AI 客服，降低客服压力。

#### 第二轮输入
主要给消费者和客服坐席使用，需要商品咨询和订单查询，私有化部署，并对接订单系统。

### 预期

#### 第一轮预期
- route_decision = need_follow_up
- search_questions = []
- structured_requirement 应保留已明确字段：
  - business_goal = 降低客服压力
  - scenario 为空
- missing_fields 应包含：
  - scenario
  - target_users
  - functional_requirements
  - deployment_requirement
  - integration_requirements
- clarification.questions 应生成面向用户的追问问题
- 系统应将第一轮已提取的 structured_requirement 保存到 requirement_state
- 系统应将缺失字段保存到 missing_fields_state
- 系统应将 last_route_decision 保存为 need_follow_up

#### 第二轮预期
- LLM1 应读取上一轮 requirement_state，并与本轮用户补充信息合并
- business_goal 应保留为“降低客服压力”
- scenario 为空
- target_users 应包含：
  - 消费者
  - 客服坐席
- functional_requirements 应包含：
  - 商品咨询
  - 订单状态查询
- deployment_requirement = 私有化部署
- integration_requirements 应包含：
  - 订单系统对接
- route_decision = technical_matching
- clarification.missing_fields = []
- clarification.questions = []
- search_questions 不为空
- search_questions 应覆盖商品咨询、订单状态查询、私有化部署、订单系统对接等需求项
- technical_matching 分支不应在正式产品中直接向用户输出 structured_requirement 和 search_questions，而应进入知识检索节点

### 实际输出
#### 第一轮实际输出
```json
{
  "structured_requirement": {
    "business_goal": "降低客服压力",
    "scenario": "",
    "target_users": [],
    "functional_requirements": [],
    "deployment_requirement": "",
    "integration_requirements": []
  },
  "search_questions": [],
  "clarification": {
    "missing_fields": [
      "scenario",
      "target_users",
      "deployment_requirement",
      "integration_requirements"
    ],
    "questions": [
      "请描述具体的应用场景（如：售前咨询、售后支持、订单查询等）？",
      "该AI客服系统的实际使用对象是谁（如：消费者、客服坐席、门店员工等）？",
      "对部署方式有什么要求（如：公有云、私有化部署、混合云等）？",
      "需要与哪些现有系统进行对接（如：CRM系统、工单系统、知识库等）？"
    ]
  },
  "route_decision": "need_follow_up",
  "route_reason": "用户输入仅包含业务目标，缺失应用场景、目标用户、部署要求和集成需求等关键字段，无法进入证据检索流程。"
}
```

#### 第一轮 Bot 输出

当前需求信息还不完整，暂时无法进入证据检索流程。
缺失字段：
scenario
target_users
deployment_requirement
integration_requirements
请补充以下问题：
请描述具体的应用场景（如：售前咨询、售后支持、订单查询等）？
该AI客服系统的实际使用对象是谁（如：消费者、客服坐席、门店员工等）？
对部署方式有什么要求（如：公有云、私有化部署、混合云等）？
需要与哪些现有系统进行对接（如：CRM系统、工单系统、知识库等）？

####  第一轮Usage
```json
{
  "prompt_tokens": 1629,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0003258",
  "completion_tokens": 294,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.000588",
  "total_tokens": 1923,
  "total_price": "0.0009138",
  "currency": "RMB",
  "latency": 3.43,
  "time_to_first_token": 1.281,
  "time_to_generate": 2.149
}
```

#### 第二轮实际输出

```json
{
  "structured_requirement": {
    "business_goal": "降低客服压力",
    "scenario": "商品咨询和订单查询",
    "target_users": [
      "消费者",
      "客服坐席"
    ],
    "functional_requirements": [
      "AI客服",
      "商品咨询",
      "订单查询"
    ],
    "deployment_requirement": "私有化部署",
    "integration_requirements": [
      "对接订单系统"
    ]
  },
  "search_questions": [
    {
      "requirement_item": "业务目标：降低客服压力",
      "query": "AI客服降低客服压力的解决方案"
    },
    {
      "requirement_item": "功能需求：商品咨询",
      "query": "AI客服商品咨询能力"
    },
    {
      "requirement_item": "功能需求：订单查询",
      "query": "AI客服订单状态查询能力"
    },
    {
      "requirement_item": "部署要求：私有化部署",
      "query": "AI客服私有化部署方案"
    },
    {
      "requirement_item": "集成需求：对接订单系统",
      "query": "AI客服对接订单系统接口能力"
    }
  ],
  "clarification": {
    "missing_fields": [],
    "questions": []
  },
  "route_decision": "technical_matching",
  "route_reason": "所有必填字段均已明确，可以进入证据检索流程。"
}
```

#### 第二轮 Bot 输出

需求信息已完整，可以进入证据检索。
结构化需求：
{
  "business_goal": "降低客服压力",
  "scenario": "商品咨询和订单查询",
  "target_users": ["消费者", "客服坐席"],
  "functional_requirements": ["AI客服", "商品咨询", "订单查询"],
  "deployment_requirement": "私有化部署",
  "integration_requirements": ["对接订单系统"]
}
检索问题：
业务目标：降低客服压力：AI客服降低客服压力的解决方案
功能需求：商品咨询：AI客服商品咨询能力
功能需求：订单查询：AI客服订单状态查询能力
部署要求：私有化部署：AI客服私有化部署方案
集成需求：对接订单系统：AI客服对接订单系统接口能力

####  第二轮Usage
```json
{
  "prompt_tokens": 1652,
  "prompt_unit_price": "0.0002",
  "prompt_price_unit": "0.001",
  "prompt_price": "0.0003304",
  "completion_tokens": 319,
  "completion_unit_price": "0.002",
  "completion_price_unit": "0.001",
  "completion_price": "0.000638",
  "total_tokens": 1971,
  "total_price": "0.0009684",
  "currency": "RMB",
  "latency": 3.728,
  "time_to_first_token": 1.526,
  "time_to_generate": 2.201
}
```

### 结果

Partial Pass

### 通过点

* 第一轮正确识别信息不足，并输出 route_decision = need_follow_up。
* 第一轮正确生成 clarification.questions，引导用户补充缺失信息。
* 会话变量成功保存第一轮已提取的 requirement_state。
* 第二轮能够读取上一轮 requirement_state，并合并本轮补充信息。
* 第二轮成功从 need_follow_up 转为 technical_matching。
* 第二轮生成了 search_questions，满足进入证据检索的基础条件。
* If/Else 分支流转正确，能够根据 route_decision 进入不同分支。

### 问题记录

第二轮中 scenario 字段错误。

模型输出：

"scenario": "商品咨询和订单查询"

但“商品咨询”和“订单查询”属于具体功能需求，应进入 functional_requirements，不应作为 scenario。

更合理的结果应为：

"scenario": ""

该问题属于 scenario 与 functional_requirements 字段语义混淆，实际上从子问题生成时语义混淆问题就已经出现，“请问这个AI客服主要应用于哪些具体的业务场景（例如：售前咨询、售后支持、订单查询等）？”，应该在子问题生成时，约束该产品具体的业务场景。


### 根因分析

1. Prompt 中虽然定义了 required_fields，但 scenario 字段缺少更明确的字段级定义。
2. 第一轮追问问题本身存在语义混淆：针对 scenario 的追问中举例包含了“订单查询”，但“订单查询”在当前字段体系中更适合归类为 functional_requirements，而不是应用场景。这导致用户第二轮补充“商品咨询和订单查询”时，模型更容易将其误填入 scenario。
3. 模型将“商品咨询和订单查询”这类功能需求误填为应用场景，说明 scenario 和 functional_requirements 的语义边界不够清晰。
4. 多轮合并场景下，模型会尝试用本轮补充内容填满上一轮缺失字段。如果上一轮追问问题的示例边界不清，模型可能会把用户对功能需求的回答错误合并到 scenario 字段。
5. technical_matching 分支当前仍使用测试 Answer 节点，因此暴露了内部 structured_requirement 和 search_questions。正式流程中该分支应进入 Knowledge Retrieval 节点，而不是直接向用户输出内部中间态。

### 调整动作

required_fields 中存在多个语义相近、容易被模型混填的字段，如果只修单点字段，后面还会在其他 case 里复现。
### 调整动作

1. Prompt 增加 required_fields 字段级语义边界规则，重点约束容易混淆的字段：

   - scenario：
     - 表示 AI 系统所在的业务场景、服务环节或客户旅程阶段。
     - 合法示例：AI客服、售前咨询、售后服务、在线客服、门店导购、内部客服支持、工单处理。
     - 不能填写具体功能项。
     - “商品咨询”“订单查询”“退换货政策问答”“投诉转人工”等属于 functional_requirements，不能作为 scenario。

   - target_users：
     - 表示 AI 系统的实际使用对象。
     - 合法示例：消费者、客服坐席、门店员工、售后人员、内部员工、客户经理。
     - 客户行业、客户公司、客户类型不能作为 target_users。
     - “零售企业”“制造业公司”“金融机构”“企业客户”等不能直接作为 target_users。

   - functional_requirements：
     - 表示 AI 系统需要完成的具体任务或功能能力。
     - 合法示例：商品咨询、订单状态查询、退换货政策问答、投诉转人工、知识库问答、工单创建。
     - 不能填写业务目标、部署方式、系统对接对象或客户行业。
     - “降低客服压力”属于 business_goal，不属于 functional_requirements。
     - “私有化部署”属于 deployment_requirement，不属于 functional_requirements。
     - “对接订单系统”属于 integration_requirements，不属于 functional_requirements。

   - deployment_requirement：
     - 表示系统部署形态或部署约束。
     - 合法示例：公有云、私有化部署、混合云、本地化部署。
     - 不能填写功能需求、系统对接对象或业务场景。
     - “订单系统”“客服系统”“CRM系统”属于 integration_requirements，不属于 deployment_requirement。

   - integration_requirements：
     - 表示需要对接的外部系统、数据源、平台或接口。
     - 合法示例：订单系统、客服系统、CRM系统、工单系统、商品知识库、用户系统。
     - 不能填写具体功能本身。
     - “订单查询”属于 functional_requirements；“对接订单系统”才属于 integration_requirements。

2. Prompt 增加 clarification.questions 生成规则：
   - 追问问题中的示例必须与被追问字段的语义一致。
   - 追问 scenario 时，只能举业务场景、服务环节、客户旅程阶段类示例。
   - 追问 target_users 时，只能举实际使用对象类示例。
   - 追问 functional_requirements 时，才能举具体功能能力类示例。
   - 追问 deployment_requirement 时，只能举部署形态类示例。
   - 追问 integration_requirements 时，只能举系统、平台、数据源或接口类示例。
   - 不得在 scenario 的追问示例中使用“订单查询”“商品咨询”等具体功能项。
   - 不得在 target_users 的追问示例中使用“零售企业”“制造业公司”等客户行业信息。

3. Prompt 增加字段归类优先级规则：
   - “降低成本、提升效率、减少人工压力、提升转化率”等表达优先归入 business_goal。
   - “AI客服、售前咨询、售后服务、在线客服、门店导购”等表达优先归入 scenario。
   - “消费者、客服坐席、门店员工、内部员工”等表达优先归入 target_users。
   - “商品咨询、订单状态查询、退换货政策问答、转人工”等表达优先归入 functional_requirements。
   - “私有化部署、公有云、混合云、本地化部署”等表达优先归入 deployment_requirement。
   - “订单系统、客服系统、CRM、工单系统、知识库、用户系统”等表达优先归入 integration_requirements。

4. Prompt 增加输出前自检：
   - 检查 scenario 是否被具体功能项污染。
   - 检查 target_users 是否由客户行业、客户公司或客户类型推断而来。
   - 检查 functional_requirements 是否混入部署方式、系统对接对象或业务目标。
   - 检查 deployment_requirement 是否混入系统名称或功能需求。
   - 检查 integration_requirements 是否混入功能项，而不是系统或数据源。
   - 如果发现字段混淆，必须先重新归类，再判断 required_fields 是否完整。
   - 如果重新归类后存在字段缺失，route_decision 必须为 need_follow_up。

### 复测

#### 第一轮输入

我们想做 AI 客服，降低客服压力。

#### 第二轮输入

主要给消费者和客服坐席使用，需要商品咨询和订单查询，私有化部署，并对接订单系统。

#### 复测结果

* 第一轮仍进入 need_follow_up。
* 第二轮 route_decision = technical_matching。
* scenario = AI客服。
* functional_requirements 包含商品咨询、订单状态查询。
* search_questions 非空。
* technical_matching 分支不再输出调试信息，而是进入 Knowledge Retrieval 节点。

#### 第一轮实际输出

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
      "请问该 AI 客服系统的目标使用对象是谁？（例如：终端消费者、内部员工等）",
      "请问 AI 客服具体需要处理哪些业务场景或功能？（例如：商品咨询、订单查询、退换货政策问答等）",
      "请问系统对部署方式有什么要求？（例如：公有云、私有化部署等）",
      "请问 AI 客服是否需要对接现有的业务系统？（例如：订单系统、CRM、工单系统等）"
    ]
  },
  "route_decision": "need_follow_up",
  "route_reason": "缺少 target_users, functional_requirements, deployment_requirement, integration_requirements 字段，信息不足，需要进一步追问。"
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
- 请问该 AI 客服系统的目标使用对象是谁？（例如：终端消费者、内部员工等）
- 请问 AI 客服具体需要处理哪些业务场景或功能？（例如：商品咨询、订单查询、退换货政策问答等）
- 请问系统对部署方式有什么要求？（例如：公有云、私有化部署等）
- 请问 AI 客服是否需要对接现有的业务系统？（例如：订单系统、CRM、工单系统等）

####  第一轮Usage
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
  "latency": 3.921,
  "time_to_first_token": 2.035,
  "time_to_generate": 1.886
}
```

#### 第二轮实际输出

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

#### 第二轮 Bot 输出

需求信息已完整，可以进入证据检索。

结构化需求：
{{"business_goal": "降低客服压力", "scenario": "AI客服", "target_users": ["消费者", "客服坐席"], "functional_requirements": ["商品咨询", "订单查询"], "deployment_requirement": "私有化部署", "integration_requirements": ["订单系统"]}

检索问题：
商品咨询：AI 客服是否支持基于商品知识库的商品咨询能力，支持条件是什么？
订单查询：AI 客服是否支持订单状态查询能力，是否需要对接客户订单系统？
私有化部署：AI 客服是否支持私有化部署，交付条件、限制和人工确认项是什么？
订单系统：AI 客服对接订单系统需要哪些技术条件和客户侧依赖？

####  第二轮Usage
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
  "latency": 3.887,
  "time_to_first_token": 1.616,
  "time_to_generate": 2.271
}
```

### 结果

Pass

### 结论

RC-007 验证了多轮追问补齐需求的状态流转。测试结果显示，Chatflow 已能通过会话变量保存上一轮结构化需求，并在第二轮用户补充后从 `need_follow_up` 进入 `technical_matching`，说明多轮状态机制基本跑通。但测试也暴露出 scenario 与 functional_requirements 字段语义混淆问题：模型将“商品咨询和订单查询”误填为 scenario。后续通过增加 scenario 字段级提取规则和输出前自检进行修复。
