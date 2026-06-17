# 实施路线图

## Phase 0: 当前 MVP 骨架

已完成：

- Spring Boot 项目结构。
- LangChain4j tool annotation 集成。
- 内存版 session repository。
- 内存版 graph repository。
- Graph-based MVP domain model，作为类型化 iFlow 内部模型的第一版存储骨架。
- 原子 iFlow editing tool。
- OData discovery stub。
- Communication discovery stub。
- Knowledge tool stub。
- Compile/deploy/smoke test lifecycle stub。
- REST API。
- Maven Wrapper。
- 单元测试。

## Phase 1: 持久化与真实 Trace

目标：让每个需求可恢复、可回放。

任务：

- 引入 PostgreSQL。
- 建立 Flyway/Liquibase migration。
- 持久化 RequirementSession。
- 持久化 ConversationMessage。
- 持久化 ToolCallTrace。
- 持久化 GraphSnapshot。
- 持久化 Artifact/TestRun。
- 增加 trace 查询和导出接口。

## Phase 2: OData Discovery 实现

目标：让 Agent 能真实查询 SAP metadata，而不是猜。

任务：

- 实现 S/4HANA system registry。
- 支持 Basic/OAuth credential alias 配置。
- 实现 `$metadata` 拉取。
- 解析 OData V2 metadata。
- 解析 OData V4 metadata。
- 支持 entity set、entity type、property、navigation 查询。
- 支持 sample query。
- 缓存 metadata 并支持刷新。

## Phase 3: Knowledge / Skills / Rules

目标：让 Agent 有企业上下文和可复用经验。

任务：

- 引入 pgvector 或 Milvus。
- 定义 KnowledgeDocument。
- 定义 SkillDocument。
- 定义 RuleDocument。
- 定义 FewShotCase。
- 实现 hybrid retrieval。
- 将 PO query case 作为第一批 few-shot。
- 将 REST-to-XI 同构 iFlow 样例沉淀为 archetype / few-shot。
- 将安全规则接入 model validator。

## Phase 4: LangChain4j Tool Calling Loop

目标：让模型真正多轮调用工具，而不是只返回建议。

任务：

- 配置 OpenAI/Ollama/企业模型 provider。
- 加载系统提示词。
- 注入工具集合。
- 加载 session history。
- 加载 retrieved context。
- 执行工具调用循环。
- 实现 AI-friendly tool error contract，避免把 stack trace / model dump 直接返回给模型。
- 对工具异常做可恢复处理，并把 suggestedFixes 转成下一步候选 tool call。
- 限制最大自动修复轮数。

## Phase 5: Template-based iFlow Compiler

目标：把类型化 iFlow 内部模型编译为可上传 iFlow ZIP。

任务：

- 收集基础 iFlow template ZIP。
- 分析 `.iflw` BPMN XML 结构。
- 实现 import canonicalization：解析 BPMN / `ifl` 属性、解码 SAP table XML、分离 layout hint。
- 计算 semantic fingerprint，支持重复导入幂等识别。
- 建立 participant / channel / process / step 到 SAP component XML 的映射。
- 建立 process flow 和 channel binding 到 sequence/message flow 的映射。
- 把 typed adapter / step config 投影为 `ifl:property`。
- 把 headerTable、propertyTable、xmlJsonPathTable 等表格型属性从结构化列表投影为 SAP table XML。
- 支持 semantic diff，忽略 BPMN ID、shape ID、waypoint 和 SAP 编辑器生成的 sequence ID。
- 支持从多个导入 iFlow 归纳 archetype，并从 archetype 实例化新 iFlow。
- 生成 BPMN DI 坐标。
- 注入 adapter properties 和 externalized parameters。
- 注入 scripts/mappings/schemas。
- 生成 externalized parameters。
- 生成 manifest。
- 打包 ZIP。
- 加入 compiler golden tests。

## Phase 6: Integration Suite 部署与监控

目标：完成生成、部署、测试闭环。

任务：

- 实现 package 查询/创建。
- 实现 artifact 上传/更新。
- 实现部署/取消部署。
- 实现部署状态轮询。
- 实现 endpoint discovery。
- 实现 smoke test 调用。
- 实现 MPL 查询。
- 实现 trace step 读取。
- 实现错误摘要。

## Phase 7: 自动修复闭环

目标：部署或测试失败时自动定位并修改。

任务：

- 定义 error classifier。
- 将 MPL error 映射到修复 skill。
- 支持模型根据 AI-friendly error 调用 tools 修改 iFlow 状态，后端更新内部模型后重新编译。
- 支持有限次数 redeploy/retest。
- 将失败和修复过程写入 trace。
- 将成功修复沉淀为 few-shot。

## Phase 8: UI Workbench

目标：给开发者一个可视化工作台。

任务：

- Requirement session list。
- Chat + tool trace timeline。
- 类型化 iFlow 内部模型可视化。
- Participant / channel / process / step editor。
- Knowledge hit viewer。
- Compile/deploy/test 状态面板。
- MPL log viewer。
- Human approval gate。

## Phase 9: 企业化能力

目标：支撑团队和生产环境。

任务：

- 多 tenant。
- RBAC。
- 审计日志。
- secret manager 集成。
- 生产部署审批。
- template marketplace。
- eval dashboard。
- CI/CD 集成。
