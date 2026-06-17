# Integration Suite Agent 需求文档

## 1. 背景

SAP Integration Suite / Cloud Integration 的 iFlow 开发通常依赖图形化建模器。对于大量相似集成场景，人工拖拽节点、配置 adapter、查 OData metadata、部署、测试、读 MPL 日志会消耗大量研发时间。

本项目目标是构建一个面向 SAP Integration Suite 的 agentic 开发平台：大模型不直接生成 SAP BPMN XML，也不直接读写结构化 iFlow DSL。大模型只知道可调用 tools、tool 参数 schema、tool 返回结果和 AI 友好的异常信息；后端 tool 层负责把工具调用转换为结构化 iFlow DSL 的受控变更，再由确定性 Java 编译器生成 iFlow ZIP，并自动上传、部署、测试和追踪结果。

## 2. 产品目标

- 用自然语言描述集成需求，自动完成 iFlow 设计草案。
- 通过工具查询 SAP OData metadata、Communication Arrangement、凭证别名和已有模板。
- 用 `createIFlow`、`addParticipant`、`addChannel`、`addStep`、`connectSteps`、`setAdapterPolicy`、`addDataMappings` 等 tools 逐步更新后端内部结构化 iFlow DSL。
- 编译结构化 iFlow DSL 为可部署 iFlow ZIP；JSON 仅作为一种持久化和 API 传输格式，不是 DSL 的抽象边界。
- 自动上传、部署、运行 smoke test、读取 MPL/trace，并支持失败后的自动修正循环。
- 记录每个需求的完整对话、工具调用、DSL 版本、编译产物、部署结果、测试结果和人工反馈。
- 当 tool 调用、DSL validation、编译、部署或测试失败时，系统必须返回 AI 友好的异常信息：错误分类、受影响的语义对象、原因、可执行修复建议和可重试 tool call 示例。
- 沉淀样本库、few-shot、skills、rules 和知识库，使系统持续变聪明。

## 3. 用户角色

### 3.1 集成开发者

- 输入业务需求。
- 审核 Agent 通过 tools 生成的 iFlow 草案。
- 介入复杂 mapping、异常处理、认证和部署决策。

### 3.2 集成架构师

- 维护 rules、skills、模板库和技术规范。
- 审核结构化 iFlow DSL 设计和编译器输出。
- 定义上线前质量门禁。

### 3.3 运维 / 平台管理员

- 配置 SAP Integration Suite tenant、S/4HANA system、credential alias、Communication Arrangement。
- 管理部署权限、测试环境、日志访问和审计。

## 4. 核心场景

### 4.1 查询 S/4HANA Purchase Order

用户输入：

```text
帮我做一个查询采购订单的 iFlow，传 PO 号，返回 header 和 item。
```

Agent 应执行：

1. 检索 PO 查询相关 skill/few-shot/rules。
2. 查询 `API_PURCHASEORDER_2` metadata。
3. 确认 `SAP_COM_0053` inbound service URL。
4. 调用 `createIFlow` 创建 iFlow 草案。
5. 调用 tools 添加 HTTP sender、content modifier、request reply、OData receiver、end event。
6. 配置 `$expand=_PurchaseOrderItem`。
7. 调用 validation tool 校验 DSL。
8. 编译 iFlow ZIP。
9. 上传部署到 Integration Suite。
10. 发起测试请求并读取 MPL。
11. 如果失败，基于 AI 友好的错误信息继续调用 tools 修改 iFlow 或 mapping 后重试。

### 4.2 SOAP 到 OData 集成

- 查询源 WSDL / XSD。
- 查询目标 OData metadata。
- 生成 mapping 规则。
- 添加异常子流程、value mapping、content modifier 和 receiver adapter。

### 4.3 基于已有 iFlow 模板改造

- 导入已有 artifact ZIP。
- 后端 importer 解析出结构化 iFlow DSL，并保留原始 SAP BPMN / `ifl` 元数据的可追溯引用。
- 大模型只通过工具修改参与方、通道、步骤、mapping 和策略。
- 重新编译并部署。

### 4.4 REST 入站到 XI/SOAP 接收方

给定一个已有 iFlow：HTTPS sender 接收 JSON 请求，设置 SAP header / property，执行日志 Groovy 脚本，调用本地流程读取 URL 参数，JSON 转 XML，请求-响应调用 S/4 XI receiver，再 XML 转 JSON、设置接收方 header、记录响应并返回。

工具层 / DSL 必须能抽象表达：

- 外部参与方和 Integration Process，而不是直接暴露 BPMN participant / shape ID。
- Sender / receiver channel 的 adapter 语义，例如 HTTPS sender、XI over HTTP receiver、认证、QoS、timeout、地址和 externalized 参数。
- 主流程、local integration process、exception subprocess 之间的调用关系。
- 处理步骤类型：content modifier / enricher、Groovy script、process call、JSON/XML converter、request-reply、message start/end、error start/end。
- Header、property、body、global persisted variable、credential alias 和 namespace mapping 的不同作用域。
- Adapter 和 flow step 的 SAP 专有属性需要通过类型化配置承载；未知属性可作为 vendor extension 保留，但不能成为主要建模方式。
- 图形坐标属于 layout hint，不能影响流程语义。

### 4.5 同构 REST 到 XI/SOAP iFlow 家族

当系统连续看到 `InboundDelivery`、`IncomingInvoice` 这类结构几乎相同的 iFlow 时，后端 import / archetype tools 必须识别它们属于同一个集成模式家族，而不是生成两份无关联的低层图：

- 共享的 archetype：HTTPS sender -> set headers/properties -> log request -> local process 读取 URL 参数 -> JSON to XML -> set S/4 properties -> request-reply XI receiver -> XML to JSON -> set receiver -> log response -> message end。
- 可变业务参数：业务对象名、sender endpoint 名称、namespace URI、receiver service interface namespace、externalized parameter 集合、证书 DN、adapter retry/QoS/proxy/header cleanup 策略。
- 不稳定实现细节：`CallActivity_*`、`SequenceFlow_*`、BPMN DI 坐标、SAP 编辑器重新生成的数字 ID，不能作为语义身份。
- DSL 必须支持从多个样例抽取可复用模板，并能对模板实例做 semantic diff：只报告 namespace、channel 参数、adapter policy 等真实变化。
- DSL 必须支持参数目录和 externalization policy：哪些 adapter 属性固定、哪些必须外部化、哪些从 tenant property / global persisted variable / credential alias 读取。
- 重复导入同一个或等价 iFlow XML 必须幂等：系统应复用已有语义模型或创建新 revision，不能因为 BPMN ID、坐标或 XML 属性顺序产生重复模板。

## 5. 功能需求

### 5.1 需求会话

- 创建需求 session。
- 保存用户消息、助手消息、系统消息和工具消息。
- 支持恢复历史上下文继续构建。
- 每次工具调用必须记录输入、输出、耗时、状态和错误。

### 5.2 结构化 iFlow DSL

- iFlow 内部模型必须采用结构化 DSL 表达；JSON/YAML/数据库记录只是序列化形式。
- DSL 必须有一等概念：integration flow、participant、endpoint channel、process、step、flow、resource、mapping、parameter、exception policy、layout hint。
- DSL 必须表达业务语义和集成模式，不要求作者理解 SAP BPMN XML ID、`cmdVariantUri` 或画布坐标。
- 节点、边、资源、mapping、外部化参数、adapter 配置和异常策略必须可版本化。
- DSL 必须支持导入已有 iFlow 后的 round-trip trace：每个语义元素可记录来源 BPMN element ID / `ifl` property key，但这些 trace 默认只供 importer、validator、compiler 和 reviewer 使用，不直接暴露给大模型。
- DSL 必须支持稳定语义身份：同一业务步骤在不同 iFlow 导入中即使 BPMN ID 不同，也应能通过 kind、name、process path、adapter binding 和业务角色匹配。
- DSL 必须支持 iFlow archetype / template instance：把重复集成模式抽象成模板，把业务对象、namespace、endpoint path、receiver interface、adapter policy 和参数清单作为实例化变量。
- DSL 必须支持 canonical form 和 semantic fingerprint，用于重复导入识别、模板归并、semantic diff 和 review。
- DSL 参数必须区分 semantic name 与 SAP externalized parameter name，例如 `sender.clientCertificate.issuerDn` 可投影为 `MessageFlow_1_clientCertificate.issuerDN`。
- SAP 专有属性必须优先映射到类型化 adapter / step 配置；暂未建模的属性才放入 `vendorExtensions`。
- DSL 不允许保存明文密码、token、client secret。
- 所有 node type、edge type、properties 必须 schema 校验。

### 5.3 Tool Calling

必须提供至少以下工具族：

- iFlow editing tools。
- OData discovery tools。
- Communication discovery tools。
- Knowledge / rules / skills retrieval tools。
- Compile / deploy / test / monitoring tools。
- Mapping suggestion / validation tools。

Tool calling 硬约束：

- 大模型只能通过 tools 观察和改变 iFlow 状态，不能直接读取或提交完整 DSL 文档。
- state mutation tool 是唯一可以修改内部 DSL 的入口；大模型只能请求业务语义 action，每次修改必须产生 DSL revision 和 trace。
- tool 参数必须是业务语义 action，例如添加 sender channel、设置 receiver retry policy、添加 script step，而不是要求模型提交任意 DSL JSON patch。
- tool 返回值必须包含 compact state summary，例如当前流程步骤、受影响对象、revision、validation status。
- tool 失败必须返回 AI-friendly error，禁止只返回 Java stack trace、BPMN XML 片段或低层 SAP property dump。
- 如果 error 可自动修复，返回建议的下一步 tool call；如果需要用户输入，明确列出缺失信息。

### 5.4 编译器

- 输入结构化 iFlow DSL。
- 输出 SAP Integration Suite 可上传的 iFlow ZIP。
- 第一阶段采用 template-based compiler，不从零手写所有 BPMN XML。
- 编译前必须执行 DSL validation。

### 5.5 自动部署与测试

- 支持上传 iFlow artifact。
- 支持部署 / undeploy。
- 支持调用 smoke test endpoint。
- 支持读取 MPL 和 trace。
- 支持有限次数的自动修复循环。

### 5.6 知识增强

- 支持样本库、few-shot、skills、rules 和技术知识库。
- 支持根据 task type、API、系统、组件类型检索。
- 成功案例可沉淀为 few-shot。
- 失败案例可沉淀为 rules 或 debug skill。

## 6. 非功能需求

- 安全：secret 不落结构化 DSL；敏感值只以 credential alias 引用。
- 可追踪：每个需求全链路可回放。
- 可验证：核心 tool action、DSL mutation 和 validation 必须单元测试覆盖。
- 可扩展：SAP client、vector store、compiler、LLM provider 都必须可替换。
- 稳定性：大模型只能通过工具请求修改 iFlow 状态；后端负责更新结构化 DSL，大模型不能直接写 DSL 或最终 BPMN XML。
- 审计：生产部署应支持人工确认门禁。

## 7. MVP 范围

MVP 只支持：

- Java 21 + Spring Boot + LangChain4j。
- 内存版 session / graph repository。
- 结构化 iFlow DSL 原子操作，以及 JSON 序列化持久化。
- OData discovery stub。
- Communication discovery stub。
- Lifecycle stub。
- PO 查询场景相关 metadata 示例。
- REST API + Swagger。
- 单元测试。

## 8. 暂不包含

- 完整 SAP Integration Suite design-time API 客户端。
- 完整 iFlow `.iflw` XML round-trip 编译器。
- 真实 pgvector 知识库。
- 复杂 message mapping 图形化编辑器。
- 生产级权限模型。
