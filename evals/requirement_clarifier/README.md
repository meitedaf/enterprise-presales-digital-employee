# 需求梳理 Agent 测试说明

## 测试对象

需求梳理 Agent

## 测试目标

验证需求梳理 Agent 是否能够根据用户输入完成以下任务：

1. 从客户原始需求中提取结构化需求。
2. 判断 required_fields 是否完整。
3. 信息不足时输出追问问题，并将 route_decision 设置为 need_follow_up。
4. 信息充分时输出 search_questions，并将 route_decision 设置为 technical_matching。
5. 不自行假设客户未明确提供的信息。
6. 输出符合指定 JSON Schema。

## required_fields

- business_goal
- scenario
- target_users
- functional_requirements
- deployment_requirement
- integration_requirements

## 路由规则

- 任一 required_field 缺失：route_decision = need_follow_up
- required_fields 全部明确：route_decision = technical_matching

## 通过标准

### 信息不足用例

- route_decision 必须为 need_follow_up
- search_questions 必须为空数组
- clarification.missing_fields 必须包含缺失字段
- clarification.questions 必须包含面向客户的追问问题

### 信息充分用例

- route_decision 必须为 technical_matching
- clarification.missing_fields 必须为空数组
- clarification.questions 必须为空数组
- search_questions 必须覆盖主要功能需求、部署要求和系统对接需求

## 失败标准

- 在信息缺失时进入 technical_matching
- 自行补全用户未提供的信息
- 在 need_follow_up 时生成 search_questions
- 输出非 JSON 或 JSON 字段缺失