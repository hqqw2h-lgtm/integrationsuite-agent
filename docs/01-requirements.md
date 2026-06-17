# Integration Suite Agent 需求文档

## 1. 背景与目标

SAP Integration Suite / Cloud Integration 的 iFlow 本质上是一个 BPMN 流程图，加上 SAP `ifl` 扩展属性、资源文件、外部化参数和画布布局。人工在图形化建模器里拖拽节点、配置 adapter、查 metadata、部署、测试和读 MPL 日志会消耗大量研发时间。

本项目目标是构建一个 agentic iFlow 开发平台：

- 大模型不直接生成 BPMN XML。
- 大模型不直接读取或写入后端内部模型。
- 大模型只看到强类型 tools、tool 参数 schema、compact state summary 和 AI-friendly error。
- 后端 tools 把业务语义命令转换为可验证、可回滚的 iFlow 内部模型变更。
- 后端 compiler 把内部模型确定性投影为 BPMN XML、SAP `ifl` 扩展属性、资源文件和可部署 iFlow ZIP。

## 2. 核心原则

### 2.1 LLM 只调用 tools

LLM-facing 层只暴露具体工具，例如：

- `createIFlow`
- `addParticipant`
- `addSenderChannel`
- `addReceiverChannel`
- `addScriptStep`
- `addContentModifierStep`
- `addJsonToXmlConverter`
- `addXmlToJsonConverter`
- `addProcessCallStep`
- `addRequestReplyStep`
- `addEdge`
- `updateEdge`
- `deleteEdge`
- `deleteStep`
- `setAdapterPolicy`
- `getIFlowStateMarkdown`
- `rollbackIFlow`
- `validateIFlow`
- `compileIFlow`

LLM-facing tools 不暴露通用 `addStep(kind, config)`，也不接受任意模型补丁。每类节点、边、adapter、step 都必须通过明确 tool schema 和枚举类型约束。

### 2.2 后端维护类型化 iFlow 内部模型

内部模型是 BPMN-backed typed graph data structure。它不是一门给 LLM 写的文本语言，也不是原始 BPMN XML。

内部模型必须表达：

- participant
- channel
- process
- step
- flow / edge
- resource
- parameter
- adapter policy
- exception policy
- layout hint
- origin ref
- semantic fingerprint

### 2.3 Maintainer 与 Compiler 分离

不同维护方式可以并存：

- `BpmnIFlowMaintainer`: 直接维护 BPMN model，适合 round-trip 已有 iFlow。
- `TypedModelIFlowMaintainer`: 维护类型化流程图，适合新建和抽象编辑。
- `JsonIFlowMaintainer`: 维护 JSON graph，适合未来兼容或轻量场景。

Maintainer 负责状态维护：

- `apply(command)`
- `snapshot()`
- `rollbackTo(revision)`
- `renderMarkdown()`
- `renderMermaid()`

Compiler 负责编译 snapshot：

- `BpmnIFlowCompiler`
- `TypedModelToBpmnCompiler`
- `JsonToBpmnCompiler`

Compiler 不编译 Maintainer 本身，只编译 `IFlowDocument` snapshot。

## 3. 核心用户场景

### 3.1 新建查询类 iFlow

用户输入：

```text
帮我做一个查询采购订单的 iFlow，传 PO 号，返回 header 和 item。
```

系统流程：

1. 检索相关 knowledge / archetype / rules。
2. 查询 S/4HANA OData metadata。
3. 调用 tools 创建 iFlow、sender channel、request-reply、receiver channel、mapping / converter。
4. Maintainer 创建 model revisions。
5. Validator 校验当前 snapshot。
6. Compiler 编译 iFlow ZIP。
7. 部署、smoke test、读取 MPL。
8. 如果失败，LLM 根据 AI-friendly error 继续调用 tools，或触发 rollback / human handoff。

### 3.2 导入已有 iFlow 并改造

系统必须支持导入已有 artifact ZIP / BPMN XML：

- 解析 BPMN + SAP `ifl` properties。
- 解码 `headerTable`、`propertyTable`、`xmlJsonPathTable` 等 SAP table XML。
- 构建 canonical model。
- 计算 semantic fingerprint。
- 识别重复导入、相似 archetype 或新模型。
- 保留 origin refs 供 round-trip、diff、debug、review 使用。

LLM 仍然只通过 tools 修改参与方、通道、步骤、边、mapping 和策略。

### 3.3 REST 入站到 XI/SOAP 接收方

给定 InboundDelivery / IncomingInvoice 这类 iFlow 样例，系统必须抽象出同一个 archetype：

```text
HTTPS sender
  -> set headers/properties
  -> log request
  -> local process read URL params
  -> JSON to XML
  -> set S/4 properties
  -> request-reply XI receiver
  -> XML to JSON
  -> set receiver
  -> log response
  -> message end
```

需要识别真实语义差异：

- business object
- namespace
- endpoint path
- receiver service interface
- adapter externalization policy
- certificate / QoS / retry / proxy settings

需要忽略非语义差异：

- BPMN element numeric ID
- sequence flow ID
- BPMN DI coordinate
- SAP editor generated shape ID
- XML attribute order

## 4. Tool 工程化要求

### 4.1 强类型命令

所有 LLM-facing mutation tools 必须使用 schema 和枚举约束：

```text
StepType =
  MESSAGE_START
  MESSAGE_END
  SCRIPT
  CONTENT_MODIFIER
  JSON_TO_XML_CONVERTER
  XML_TO_JSON_CONVERTER
  PROCESS_CALL
  REQUEST_REPLY
  ROUTER
  MESSAGE_MAPPING

EdgeType =
  SEQUENCE_FLOW
  MESSAGE_FLOW
  EXCEPTION_FLOW
  ROUTER_BRANCH

AdapterType =
  HTTPS
  XI
  ODATA
  SOAP
  SFTP
```

自由字符串类型必须被 schema 拒绝，防止 hallucinated node / edge / adapter type 进入内部模型。

### 4.2 原子 mutation

增删改节点和边必须是原子命令：

- `add*` 必须一次性携带该对象的完整必填信息。
- 不允许创建空节点、空边、半配置 adapter。
- `update*` 必须采用 schema 化 replace / patch，并返回变更前后摘要。
- `delete*` 必须返回影响分析，例如断开的上下游 step、receiver channel、需要重连的流程片段。
- 每个成功 mutation 必须创建 revision。
- 失败 mutation 不能留下半更新状态。

### 4.3 当前状态可见性

LLM 需要看到当前流程上下文，但不能读取完整内部模型。Maintainer 必须支持：

- `getIFlowStateMarkdown`
- `getIFlowStateMermaid`

返回内容应包括：

- iFlow name / revision / validation status
- 主流程 Mermaid 图
- local process / exception subprocess 摘要
- channels 摘要
- unresolved parameters
- open validation issues
- recommended next actions

### 4.4 回退

系统必须支持：

- rollback 到任意已保存 revision。
- rollback 到最后一个 valid revision。
- 自动修复失败后回退。
- 用户要求撤销后回退。
- rollback 本身生成新的 revision，并记录 reason。

## 5. 防止大模型死循环

### 5.1 执行预算

每个 requirement session 必须有执行预算：

- max tool calls per turn
- max mutation calls per turn
- max compile attempts
- max deploy attempts
- max smoke test attempts
- max auto-fix iterations
- max repeated same-error attempts

超过预算后必须停止自动循环，返回 AI-friendly escalation。

### 5.2 Progress detector

系统必须检测是否有实质进展：

有进展的信号：

- model revision changed
- validation issue count decreased
- blocking error changed category
- compile progressed to later phase
- deployment reached new state
- smoke test failed at a later assertion

无进展的信号：

- 连续多次相同 tool call + 相同参数。
- 连续多次相同 error code + 相同 affected object。
- revision 未变化。
- validation issue set 未变化。
- compile / deploy / test failure fingerprint 未变化。

无进展达到阈值后必须触发 loop guard。

### 5.3 Loop guard 行为

Loop guard 触发后必须选择以下动作之一：

- rollback 到最后一个 valid revision。
- 请求用户补充信息。
- 切换到更保守的 repair strategy。
- 暂停自动修复并输出诊断摘要。
- 转人工 review。

Loop guard 返回的错误必须包含：

- loop reason
- repeated tool calls
- repeated errors
- last valid revision
- current invalid revision
- suggested user decision

### 5.4 Idempotency 与 no-op 检测

所有 mutation commands 必须有 idempotency key 或 command fingerprint。

系统必须识别：

- 重复提交同一 mutation。
- mutation 后模型 canonical fingerprint 未变化。
- tool 建议修复但实际 no-op。

No-op mutation 不应无限重试；应返回 `NO_EFFECT_MUTATION`。

### 5.5 Circuit breaker

对外部系统调用必须有 circuit breaker：

- SAP design-time API 连续失败。
- deployment status 长时间不变化。
- MPL/trace 读取持续超时。
- metadata service 不可用。

Circuit breaker 打开后，LLM 不应继续调用同类外部 tool；系统应返回 retry-after 或人工介入建议。

## 6. AI-friendly Error

所有 tool、validator、compiler、deploy/test wrapper 必须返回结构化错误对象：

```json
{
  "status": "failed",
  "errorCode": "REPEATED_NO_PROGRESS",
  "category": "loop_guard",
  "severity": "error",
  "message": "Auto-fix stopped after 3 attempts with the same validation error.",
  "affectedObject": {
    "kind": "channel",
    "semanticKey": "channel.receiver/SOAP_Receiver_MM"
  },
  "reason": "The last three setAdapterPolicy calls did not change the missing QualityOfService issue.",
  "suggestedFixes": [
    {
      "action": "askUser",
      "arguments": {
        "question": "Which XI QualityOfService should be used: BestEffort, ExactlyOnce, or ExactlyOnceInOrder?"
      }
    },
    {
      "action": "rollbackIFlow",
      "arguments": {
        "targetRevision": 12
      }
    }
  ],
  "retryable": false
}
```

不得直接把 Java stack trace、完整 BPMN XML、完整内部模型或低层 SAP property dump 返回给 LLM。

## 7. 编译、部署与测试

Compiler 输入 `IFlowDocument` snapshot，输出：

- BPMN XML
- SAP `ifl` properties
- resources
- manifest
- externalized parameters
- iFlow ZIP

部署和测试必须记录：

- artifact checksum
- deployed artifact id
- deployment id
- endpoint
- smoke test request / response
- MPL id
- trace summary

## 8. 安全与审计

- Secret 不落内部模型。
- LLM tool 参数不得包含 password、token、client secret。
- Credential 只能以 credential alias / security material reference 表达。
- ToolCallTrace 必须脱敏。
- 生产部署必须支持人工确认门禁。
- 所有 mutation、compile、deploy、rollback 都必须审计。

## 9. MVP 范围

MVP 支持：

- Java 21 + Spring Boot + LangChain4j。
- 内存版 session / model repository。
- LLM-facing concrete tools。
- `TypedModelIFlowMaintainer` 初版。
- model revision / rollback 初版。
- Markdown / Mermaid state summary 初版。
- AI-friendly error contract。
- OData discovery stub。
- Communication discovery stub。
- Lifecycle stub。
- REST API + Swagger。
- 单元测试。

暂不包含：

- 完整 SAP Integration Suite design-time API client。
- 完整 `.iflw` round-trip compiler。
- 生产级权限模型。
- 复杂 message mapping 图形化编辑器。
- 真实向量知识库。
