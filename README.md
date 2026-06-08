# 企业售前数字员工

基于 Dify Chatflow 的 AI 客服售前方案生成 Agent MVP。

项目目标：将客户模糊需求结构化，基于知识库逐项检索证据，评估产品能力匹配关系，生成售前可审核的初版方案，并进行风险校验。

## 目录结构

```text
企业售前数字员工/
├── README.md
├── docs/              # PRD、SOP、Chatflow 设计说明
├── knowledge_docs/    # Dify 知识库文档
├── prompts/           # 各 LLM 节点 Prompt
├── schemas/           # 各 Agent 输出 JSON Schema
├── evals/             # 分节点评估用例与记录
└── dify/              # Dify 导出文件与配置说明
```

## 当前 Chatflow 节点

1. 需求梳理 Agent
2. 证据检索 Agent
3. 能力匹配评估 Agent
4. 方案生成 Agent
5. 风险校验 Agent

## 知识库说明

`knowledge_docs/` 中的 Markdown 文档用于导入 Dify 知识库。MVP 阶段建议放入同一个知识库，便于证据检索 Agent 对产品能力、技术对接、交付边界、FAQ、历史案例和冲突测试材料统一检索。

