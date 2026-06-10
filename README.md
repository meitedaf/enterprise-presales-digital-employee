# 企业售前数字员工

基于 Dify Chatflow 构建的企业售前内部 Copilot MVP。

本项目模拟企业售前团队在客户需求沟通后的核心工作流：将销售、售前顾问或解决方案团队输入的客户需求描述、沟通记录或会议纪要摘要，转化为结构化需求；基于企业知识库逐项检索证据；评估产品能力匹配关系；生成可供售前人员审核的初版方案；并在最终输出前进行风险校验，避免过度承诺和无证据结论。

> 定位说明：本系统面向售前团队内部使用，用于辅助需求梳理、能力判断和方案准备，不直接替代售前、产品、交付或商务负责人作出最终客户承诺。

---

## 项目背景

在企业售前场景中，客户需求往往以口语化、碎片化或会议纪要的形式出现。售前人员需要反复完成以下工作：

- 梳理客户真实业务目标和核心场景。
- 判断客户信息是否足够进入方案设计。
- 查找产品能力、技术方案、FAQ 和交付边界。
- 判断哪些能力标准支持、哪些需要配置、哪些依赖系统对接。
- 生成初版方案，并避免在报价、交期、私有化部署、系统对接等方面作出不当承诺。

传统流程依赖人工经验，容易出现信息遗漏、证据查找不完整、方案表达不一致或边界承诺过度等问题。

本项目尝试用多 Agent Chatflow 将该流程拆解为可控、可测试、可追溯的自动化工作流。

---

## 项目目标

本项目目标是构建一个售前内部 Copilot MVP，使其能够：

- 将模糊客户需求转为结构化需求。
- 在信息不足时生成面向售前人员的客户追问清单，并暂停进入技术匹配。
- 将完整需求拆分为逐项可检索问题，降低复合需求一次性 RAG 检索遗漏的风险。
- 基于知识库证据整理产品能力、限制条件、技术依赖和证据缺口。
- 评估每个需求项的支持等级、交付边界和人工确认项。
- 生成售前可审核的初版方案。
- 对方案进行风险校验，避免过度承诺、无证据结论、支持等级错误和人工确认项遗漏。
- 沉淀可复用的 Prompt、Schema、测试用例和运行记录。

---

## 核心流程

text 售前人员输入客户需求 / 沟通记录 / 会议纪要摘要
         |
         v
需求梳理 Agent 输出：structured_requirement + search_questions + clarification + route_decision         
         |         
         v 
格式化需求与检索问题 Code 节点 输出：structured_requirement_text + search_questions_text + search_question_items         
         |         
         v
更新需求会话状态 保存：requirement_state + requirement_state_text + missing_fields_state + last_route_decision         
         |
         v 
条件判断 
├── need_follow_up 
│   └── 输出待客户确认问题，等待下一轮补充 
│
└── technical_matching
     └── 进入证据检索与方案生成流程
         |
         v 
逐项知识检索         
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
最终安全方案输出 

---

## Agent 模块设计

| 模块 | 主要职责 | 关键输出 |
| --- | --- | --- |
| 需求梳理 Agent | 提取结构化需求，判断信息是否完整，生成追问或检索问题 | `structured_requirement`<br>`search_questions`<br>`clarification`<br>`route_decision` |
| 证据检索 Agent | 基于知识库检索结果整理每个需求项的证据、引用、限制条件和证据缺口 | `evidence_results`<br>`retrieval_summary` |
| 能力匹配评估 Agent | 将证据转成售前可用的支持等级、交付边界、技术依赖和人工确认项 | `capability_assessments`<br>`assessment_summary` |
| 方案生成 Agent | 基于结构化需求、证据和能力匹配结果生成售前初版方案 | `proposal_json`<br>`proposal_text_draft` |
| 风险校验 Agent | 检查过度承诺、支持等级错误、系统依赖遗漏和人工确认项遗漏 | `risk_check_result`<br>`final_safe_response` |

---

## 支持等级

能力匹配评估使用以下支持等级：

| 支持等级 | 含义 | 典型表达边界 |
| --- | --- | --- |
| `standard_supported` | 有直接证据证明产品标准支持。 | 可表达为“标准支持”。 |
| `configurable_supported` | 可通过知识库、规则、话术或流程配置支持。 | 必须说明“需配置后支持”。 |
| `integration_required` | 可支持，但依赖客户系统、API、数据源或第三方接口对接。 | 必须说明“完成系统/API 对接后支持”。 |
| `needs_human_confirmation` | 可评估，但需人工确认部署条件、交付周期、费用、实施范围或商务条款。 | 必须说明“可评估，需进一步确认”。 |
| `not_supported_or_no_evidence` | 当前没有明确支持证据，或证据明确说明不支持。 | 不得写入推荐方案主体。 |

涉及以下事项时，系统只能生成待确认项，不能输出最终承诺：

- 报价
- 交期
- 定制开发范围
- 私有化部署
- 客户系统对接
- 第三方系统依赖
- 实时数据查询效果
- 合同或商务条款

---

## 项目目录结构

```text
企业售前数字员工/
├── README.md
├── docs/
│   ├── prd.md                    # 产品需求与业务边界
│   ├── sop.md                    # 售前流程 SOP
│   └── dify-chatflow-design.md   # Dify Chatflow 节点设计
├── knowledge_docs/
│   └── knowledge_docs_v0.1/      # 导入 Dify 知识库的产品、技术、FAQ、案例文档
├── prompts/
│   ├── 01_requirement_clarifier_prompt/
│   ├── 02_evidence_retrieval_prompt/
│   ├── 03_capability_matching_prompt/
│   ├── 04_solution_generation_prompt/
│   └── 05_risk_check_prompt/
├── schemas/                      # 各 Agent 结构化输出 JSON Schema
├── evals/
│   ├── requirement_clarifier/
│   ├── evidence_retrieval/
│   ├── capability_matching/
│   ├── solution_generation/
│   ├── risk_check/
│   └── e2e_chatflow/             # 端到端 Chatflow 测试用例与记录
└── dify/                         # Dify 导出与配置笔记，默认不提交
```

---

## 知识库说明

knowledge_docs/knowledge_docs_v0.1/ 中的 Markdown 文档用于导入 Dify 知识库。

MVP 阶段建议放入同一个知识库，便于证据检索 Agent 对以下资料统一检索：

- 产品能力说明
- 技术方案与系统对接说明
- 交付边界与人工确认规则
- 售前常见 FAQ
- 历史案例
- 版本与冲突测试文档

证据检索 Agent 只基于知识库召回内容输出引用来源、限制条件、技术依赖和检索缺口；不得在无证据时自行假设产品支持。

---

## 评估体系

每个 `evals/` 子目录通常包含：

| 文件 | 说明 |
| --- | --- |
| `README.md` | 测试目标、输入输出口径和通过标准。 |
| `cases.jsonl` | 测试用例。 |
| `run_records.md` | 实际运行记录、问题和修复过程。 |

当前评估覆盖：

| 评估目录 | 覆盖内容 |
| --- | --- |
| `requirement_clarifier/` | 信息缺失追问、完整需求路由、结构化输出、多轮状态合并。 |
| `evidence_retrieval/` | 逐项证据整理、引用来源、限制条件、证据缺口。 |
| `capability_matching/` | 支持等级、交付边界、人工确认项和风险标记。 |
| `solution_generation/` | 基于能力匹配结果生成售前初版方案，避免新增未验证能力。 |
| `risk_check/` | 检查并修正过度承诺、系统依赖遗漏、私有化部署边界和人工确认项。 |
| `e2e_chatflow/` | 记录完整多轮 Chatflow 的端到端用例与运行结果。 |

---

## 示例测试链路

典型端到端输入：

```text 
客户反馈他们想做 AI 客服，目标是降低客服压力。请帮我先梳理需求。
``` 

系统在信息不足时输出追问：

```text 
当前客户需求信息还不完整，暂时无法进入证据检索流程。  
请向客户进一步确认，或补充以下客户信息： 
- 该需求对应的 AI 系统主要服务哪些使用对象？ 
- 客户希望该 AI 系统具体支持哪些业务功能？ 
- 客户期望的部署方式是什么？ 
- 是否需要对接现有系统？如果需要，请确认系统名称。 
```

售前人员补充信息：

```text 补充信息：客户主要希望给消费者和客服坐席使用，需要支持商品咨询和订单查询，希望私有化部署，并需要对接客户的订单系统。 
```

系统进入证据检索、能力匹配、方案生成和风险校验，最终输出面向售前人员的安全初版方案。

---

## 使用方式

1. 将 knowledge_docs/knowledge_docs_v0.1/ 导入 Dify 知识库。
2. 按 docs/dify-chatflow-design.md 搭建或更新 Chatflow 节点。
3. 将 prompts/ 中对应版本的 Prompt 配置到各 LLM 节点。
4. 将 schemas/ 中的 JSON Schema 配置为对应 Agent 的结构化输出约束。
5. 使用 evals/ 中的用例进行逐节点测试。
6. 使用 evals/e2e_chatflow/ 记录完整 Chatflow 端到端运行结果。
7. 根据 run_records.md 中的问题记录持续优化 Prompt、Schema、知识库和流程节点。

---

## 当前状态

当前 MVP 已完成以下能力验证：

- 多轮需求梳理与缺失字段追问。
- 售前内部输入口径适配。
- 逐项知识库检索。
- 证据检索与引用整理。
- 产品能力支持等级评估。
- 售前初版方案生成。
- 风险校验与最终安全方案输出。
- 分模块评估与端到端评估。

---

## 项目价值

本项目重点展示了 AI 产品从 Prompt Demo 到可评估工作流的完整设计过程，包括：

- AI PM 对业务场景和用户角色的定义。
- 多 Agent 分工与流程编排。
- RAG 检索与结构化证据整理。
- 能力匹配和交付边界判断。
- 风险校验与安全输出控制。
- Prompt、Schema、Eval 和运行记录的闭环管理。

该项目不是单一问答机器人，而是一个面向真实售前流程的多 Agent Copilot MVP。

---

## 注意事项

- 本项目仅为 MVP 和学习/作品集用途。
- 示例知识库、Prompt 和测试用例均为模拟数据，不代表真实企业产品承诺。
- 最终方案输出仍需售前、产品、交付或商务负责人审核。
- 不建议将系统输出直接作为正式客户方案、报价或合同附件使用。
