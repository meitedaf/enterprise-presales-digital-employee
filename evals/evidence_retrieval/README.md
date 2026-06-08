# Evidence Retrieval Eval

证据检索 Agent 位于需求梳理 Agent 之后、能力匹配评估 Agent 之前。它不负责判断最终支持等级，也不生成售前方案，而是基于知识库检索结果整理每个需求项对应的证据、引用来源、技术依赖、限制条件、证据缺口和可能冲突。


## Agent 目标

证据检索 Agent 的目标是：

1. 读取需求梳理 Agent 输出的 structured_requirement。
2. 读取需求梳理 Agent 输出的 search_questions 或 search_questions_text。
3. 读取 Dify Knowledge Retrieval 节点返回的 retrieval_results。
4. 按需求项整理 evidence_results。
5. 输出 retrieval_summary，供后续能力匹配评估 Agent 使用。

## 当前实现链路

text 需求梳理 Agent         ↓ Code 节点：生成 structured_requirement_text / search_questions_text         ↓ 条件分支         ↓ technical_matching         ↓ Knowledge Retrieval         ↓ Code 节点：format_retrieval_results         ↓ 证据检索 Agent 

当前阶段使用 search_questions_text 进行一次性合并检索。后续可升级为 Iteration 逐个 search_question 检索。

## 测试目标

本目录中的测试主要验证：

1. 证据检索 Agent 是否能基于召回结果逐项整理证据。
2. 是否能识别直接证据、弱证据、无证据和冲突证据。
3. 是否能提取引用来源、原文片段、限制条件和技术依赖。
4. 是否能在没有证据时输出 retrieval_gaps，而不是自行假设支持。
5. 是否能避免输出最终支持等级，将判断留给能力匹配评估 Agent。
6. 是否能识别“需系统对接”“需人工确认”“需配置”“暂不支持”等边界条件。

## 结果判定标准

### Pass

满足以下条件：

- 每个核心需求项都有对应 evidence_result。
- 能够正确识别直接证据和引用来源。
- 能够提取限制条件和技术依赖。
- 无证据项被标记为 evidence_found = false。
- 不输出最终支持等级。
- 不在无证据时自行假设产品支持。

### Partial Pass

满足部分核心能力，但存在以下问题之一：

- 知识库覆盖不足导致部分需求项 no evidence。
- 检索召回不完整。
- 部分证据质量判断不稳定。
- Answer 节点仍输出调试 JSON。
- 当前链路仍为一次性合并检索，尚未逐项检索。

### Fail

出现以下任一情况：

- 无证据时仍输出支持结论。
- 将“需系统对接后支持”写成“标准支持”。
- 忽略限制条件或技术依赖。
- 直接生成售前方案。
- 直接输出最终支持等级。
- 伪造引用来源或证据片段。
- evidence_results 与 retrieval_results 明显不一致。

