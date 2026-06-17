# 类型化 iFlow 内部模型与 Tool 设计

## 1. 设计目标

iFlow artifact 的底层包含 BPMN XML、SAP `ifl` 扩展属性、BPMN DI 坐标、adapter 配置、脚本、mapping、外部化参数和资源文件。直接让大模型生成这些文件不稳定；把它们原样塞进一份 JSON 也不够抽象。

本项目不定义给 LLM 编写的一门文本语言，而是定义后端使用的类型化 iFlow 数据结构。iFlow 本质上是 BPMN 流程图加 SAP 扩展配置；内部模型用更稳定的对象结构表达这张图，再由 compiler 投影为 `.iflw` BPMN XML 和 ZIP：

- LLM 面对的是 tools；tools 面对的是 integration flow、endpoint、channel、process、step、resource、parameter 等后端语义对象。
- JSON/YAML/数据库记录只是内部模型的序列化形式，不是抽象边界。
- SAP BPMN XML、`cmdVariantUri`、`ifl:property` 和图形坐标是 compiler projection 或 import trace，不应主导建模。
- 未建模的 SAP 专有属性可以被保留为 vendor extension，但常见 adapter / flow step 必须有类型化配置。

## 2. 设计原则

### 2.1 LLM / Tool / 内部模型边界

LLM 不直接读取、生成或修改内部模型。系统边界必须是：

```text
LLM
  -> tool name + tool argument schema
  -> tool call
  -> tool result / AI-friendly error
  -> next tool call

Tool layer
  -> validate command
  -> normalize semantic action
  -> mutate typed internal model
  -> run validators
  -> return compact state summary

Typed internal model
  -> versioned backend state
  -> compiler input
  -> import / diff / trace representation
```

因此，文档里的对象结构示意和 JSON 示例是给后端、tool 设计、validator、compiler 和 reviewer 使用的，不是要求 LLM 在 prompt 中读写完整内部模型。

### 2.2 语义优先

内部模型首先表达“这个 iFlow 做什么”，其次才表达“SAP XML 里如何落地”。

例如样例 iFlow 中的 `CallActivity_1000` 不应该在内部模型中只是一段属性表，而应该是一个 `set` 步骤：

```text
set "Set Header and Message Type" {
  headers {
    SAP_Sender = parameter("SAP_Sender")
    SAP_SenderInterface = parameter("SAP_SenderInterface")
    SAP_MessageType = header("SAP_SenderInterface")
  }
  properties {
    LogRequest = parameter("LogRequest")
    LogResponse = parameter("LogResponse")
  }
}
```

### 2.3 类型化配置优先

SAP iFlow 里大量配置以 `ifl:property` 保存。内部模型不能简单把它们变成 `Map<String, String>`。常见构件应升级为类型化字段：

- HTTPS sender: `path`、`auth`、`userRole`、`maxBodySize`、`csrfProtection`。
- XI receiver: `address`、`credentialAlias`、`locationId`、`qos`、`timeout`、`retry`、`proxy`。
- Content modifier / enricher: `headers`、`properties`、`body`、`sourceKind`。
- JSON/XML converter: root element、namespace、streaming、array paths、encoding。
- Groovy script: file、function、script collection、required resources。

### 2.4 导入可追踪，编辑要抽象

从已有 iFlow 导入时，内部模型元素应记录来源：

```text
origin {
  bpmnElement = "CallActivity_1004"
  sapVariant = "JsonToXmlConverter"
  properties = ["additionalRootElementName", "additionalRootElementNamespace", ...]
}
```

这些信息用于 round-trip、diff、debug 和 compiler；LLM 不直接编辑 BPMN ID，也不直接编辑 origin refs。

### 2.5 Layout 是 hint

样例 XML 里的 `bpmndi:BPMNShape` 和 `bpmndi:BPMNEdge` 只表达画布位置。内部模型可以保留 layout hint 方便 UI 展示，但 layout 不应影响流程语义和编译校验。

## 3. 顶层内部数据模型

概念上，类型化 iFlow 内部模型由以下元素组成：

```text
IFlow
  metadata
  namespaces
  parameters
  participants
  channels
  processes
  resources
  policies
  layoutHints
  vendorExtensions
```

### 3.1 元素职责

| 元素 | 职责 | 样例中的对应内容 |
| --- | --- | --- |
| `metadata` | iFlow 名称、package、描述、版本、全局运行配置 | collaboration extension properties |
| `namespace` | XML namespace alias 和 URI | `ns1`, `ns0` namespace mapping |
| `parameter` | 外部化参数、credential alias、tenant property 引用 | `{{MessageFlow_1_urlPath}}`, `${property.SAPHost}` |
| `participant` | 外部系统或 Integration Process 参与者 | `CSI_NSRM`, `CSI_P_S4`, `Integration Process`, `Get URL Param` |
| `channel` | sender/receiver adapter 连接 | HTTPS sender message flow, XI receiver message flow |
| `process` | 主流程、本地流程、异常子流程 | `Process_1`, `Process_2`, exception subprocesses |
| `step` | 流程中的处理动作 | script、enricher、converter、request-reply、process call |
| `flow` | process 内部控制流或 channel 绑定 | sequence flow、message flow |
| `resource` | 脚本、schema、mapping、collection | `LogPayload.groovy`, `ReadUrlPath.groovy` |
| `policy` | 异常、事务、审计、部署门禁策略 | transaction handling、error subprocess |
| `layoutHint` | UI 画布位置 | BPMN DI bounds / waypoints |
| `vendorExtension` | 暂未类型化的 SAP 属性 | 罕见 adapter property |

## 4. 后端 / Tool 设计用对象结构示例

以下不是给 LLM 使用的语法，而是便于讨论的对象结构示意。后端可以把同一模型序列化成 JSON；LLM 只调用 tools，由 tool 层把业务语义命令映射为内部模型变更。

```text
iflow "REST InboundDelivery to S4 XI" {
  namespace ns1 = "urn:csisolar.com:NSRM:S4:InboundDelivery"
  namespace ns0 = "urn:csisolar.com:basic"

  parameter MessageFlow_1_urlPath : endpointPath externalized
  parameter MessageFlow_1_userRole : role externalized
  parameter MessageFlow_1_senderAuthType : authType externalized
  parameter SAPHost : tenantProperty
  parameter SAPClient : tenantProperty
  parameter SAPCredentials : credentialAlias
  parameter LocationID : tenantProperty

  participant sender "CSI_NSRM" {
    kind externalSystem
  }

  participant receiver "CSI_P_S4" {
    kind externalSystem
  }

  channel "REST_InboundDelivery_Sender" {
    direction sender
    from participant("CSI_NSRM")
    to process("Integration Process").start
    adapter https {
      path parameter("MessageFlow_1_urlPath")
      auth parameter("MessageFlow_1_senderAuthType")
      userRole parameter("MessageFlow_1_userRole")
      maxBodySizeMb 40
      csrfProtection false
    }
  }

  channel "SOAP_Receiver_MM" {
    direction receiver
    from step("Request Reply")
    to participant("CSI_P_S4")
    adapter xi over http {
      address "http://${property.SAPHost}/sap/xi/engine?type=entry&sap-client=${property.SAPClient}"
      credentialAlias property("SAPCredentials")
      locationId property("LocationID")
      serviceInterface {
        namespace "urn:csisolar.com:NSRM:S4:InboundDelivery"
        name parameter("MessageFlow_3_ServiceInterfaceReceiver")
      }
      timeoutMs 60000
      qualityOfService parameter("MessageFlow_3_QualityOfService")
      retry {
        interval parameter("MessageFlow_3_RetryInterval")
        maxInterval parameter("MessageFlow_3_MaxRetryInterval")
        exponentialBackoff parameter("MessageFlow_3_ExponentialBackoff")
      }
    }
  }

  process "Integration Process" {
    kind integration
    transactionHandling notRequired

    start message

    set "Set Header and Message Type" {
      properties {
        LogResponse = parameter("LogResponse")
        LogRequest = parameter("LogRequest")
      }
      headers {
        SAP_Sender = parameter("SAP_Sender")
        SAP_SenderInterface = parameter("SAP_SenderInterface")
        SAP_MessageType = header("SAP_SenderInterface")
      }
    }

    script "Log Request" {
      language groovy
      file "LogPayload.groovy"
      collection "Common_Script_Collection"
      function "LogRequestMessage"
    }

    call process "Get URL Param"

    convert "JSON to XML Converter" {
      from json
      to xml
      rootElement "MT_REQ"
      rootNamespace namespace("ns1")
      useNamespaces false
    }

    set "Set Props" {
      properties {
        SAPHost = globalPersistedVariable("SAPHost")
        SAPClient = globalPersistedVariable("SAPClient")
        SAPCredentials = globalPersistedVariable("SAPCredentials")
        LocationID = globalPersistedVariable("LocationID")
      }
    }

    requestReply "Request Reply" {
      receiver channel("SOAP_Receiver_MM")
    }

    convert "XML to JSON Converter" {
      from xml
      to json
      streaming true
      suppressRootElement true
      convertElements specific
      arrayPaths [
        "/ns0:MT_reponse/datas",
        "/ns0:MT_reponse/datas/head/message_detail",
        "/ns0:MT_reponse/datas/head/message_detail/additionInfo",
        "/ns0:MT_reponse/datas/head/message_detail/additionInfo/parm"
      ]
      encoding "UTF-8"
    }

    set "SetReceiver" {
      headers {
        SAP_Receiver = constant("CSI_P_S4")
      }
    }

    script "LogResponse" {
      language groovy
      file "LogPayload.groovy"
      collection "Common_Script_Collection"
      function "LogResponseMessage"
    }

    end message

    onError {
      start error
      end error
    }
  }

  process "Get URL Param" {
    kind localIntegrationProcess
    callMode direct
    transactionHandling fromCallingProcess

    start
    script "Read URL path" {
      language groovy
      file "ReadUrlPath.groovy"
    }
    script "Read GET parameters" {
      language groovy
      file "ReadUrlGetParameters.groovy"
    }
    end

    onError {
      start error
      end error
    }
  }
}
```

## 5. iFlow Archetype 与实例化

连续分析 `InboundDelivery` 和 `IncomingInvoice` 两个样例后，可以看到它们不是两个完全独立的流程，而是同一个 REST-to-XI request/reply 模式的两个实例：

```text
archetype "REST JSON to S4 XI request-reply" {
  sender https
  mainProcess [
    messageStart,
    set("Set Header and Message Type"),
    script("Log Request"),
    processCall("Get URL Param"),
    jsonToXml("JSON to XML Converter"),
    set("Set Props"),
    requestReply("Request Reply"),
    xmlToJson("XML to JSON Converter"),
    set("SetReceiver"),
    script("LogResponse"),
    messageEnd
  ]
  localProcesses [
    "Get URL Param"
  ]
  receiver xi over http
  exceptionPolicy errorSubprocess
}
```

每个业务 iFlow 应该是 archetype 的一个实例：

```text
instance "IncomingInvoice" of "REST JSON to S4 XI request-reply" {
  namespace ns1 = "urn:csisolar.com:NSRM:S4:IncomingInvoice"
  senderChannel.name = "REST_IncomingInvoice_Sender"
  receiverChannel.serviceInterface.namespace = namespace("ns1")
  jsonToXml.rootNamespace = namespace("ns1")
}
```

这样 archetype tools 可以复用同一套模式，只调整业务对象、namespace、adapter 参数和资源；LLM 只需要选择或确认工具参数，而不是学习一组随机 BPMN ID。

### 5.1 语义身份

导入已有 iFlow 时，SAP 编辑器可能改变以下内容：

- `CallActivity_10017` 变成 `CallActivity_31233324`。
- `SequenceFlow_10018` 变成 `SequenceFlow_31233325`。
- `LogResponse` 的 BPMN ID 从 `CallActivity_10015` 变成 `CallActivity_10018`。
- BPMN DI 坐标整体平移或局部调整。

这些变化不应改变内部模型语义。内部模型需要为元素建立 stable semantic key：

```text
semanticKey = processPath + stepKind + normalizedName + role

examples:
  process.main/set/Set Props
  process.main/script/LogResponse
  process.main/requestReply/Request Reply
  channel.receiver/SOAP_Receiver_MM
```

`origin.bpmnElement` 只记录导入来源，不作为长期身份。

### 5.2 语义 diff

两个样例之间真正有意义的变化包括：

| 维度 | InboundDelivery | IncomingInvoice | model diff |
| --- | --- | --- | --- |
| 业务对象 | `InboundDelivery` | `IncomingInvoice` | instance variable changed |
| `ns1` namespace | `urn:csisolar.com:NSRM:S4:InboundDelivery` | `urn:csisolar.com:NSRM:S4:IncomingInvoice` | namespace changed |
| Sender channel | `REST_InboundDelivery_Sender` | `REST_IncomingInvoice_Sender` | channel name changed |
| Receiver interface namespace | InboundDelivery namespace | IncomingInvoice namespace | receiver service interface changed |
| JSON to XML namespace | InboundDelivery namespace | IncomingInvoice namespace | converter root namespace changed |
| Sender certificate issuer DN | empty | externalized parameter | adapter auth parameter policy changed |
| Receiver cleanup/compress/proxy properties | some literals / empty values | mostly externalized | externalization policy changed |

无意义变化包括：

- BPMN element ID number changes。
- sequence flow ID changes。
- layout coordinate shifts。
- SAP editor generated shape IDs。

### 5.3 参数目录

内部模型应显式维护参数目录，而不是只在字符串里保留 `{{...}}`：

```text
parameters {
  sender.urlPath : endpointPath externalized("MessageFlow_1_urlPath") required
  sender.authType : authType externalized("MessageFlow_1_senderAuthType") required
  sender.clientCertificate.subjectDn : certificateSubject externalized("MessageFlow_1_clientCertificate.subjectDN") optional
  sender.clientCertificate.issuerDn : certificateIssuer externalized("MessageFlow_1_clientCertificate.issuerDN") optional

  receiver.authType : authType externalized("MessageFlow_3_AuthenticationType") required
  receiver.qualityOfService : xiQualityOfService externalized("MessageFlow_3_QualityOfService") required
  receiver.cleanupHeaders : boolean externalized("MessageFlow_3_cleanupHeaders") default(true)
  receiver.proxy.host : host externalized("MessageFlow_3_proxyHost") optional

  SAPHost : tenantProperty required
  SAPClient : tenantProperty required
  SAPCredentials : credentialAlias required secretRef
  LocationID : tenantProperty optional
}
```

参数目录让 compiler、UI 和 validation 能回答：

- 哪些参数部署前必须填。
- 哪些参数是 credential alias，不能写明文。
- 哪些参数属于 sender channel、receiver channel 或 process step。
- 哪些参数从 global persisted variable 写入 exchange property。

### 5.4 Externalization Policy

第二个样例把更多 XI receiver 属性 externalized，例如 `cleanupHeaders`、`CompressMessage`、`sendHttpResponseCode`、`proxyHost`。内部模型不应只保存“当前值”，还应保存策略：

```text
adapter xi over http {
  cleanupHeaders externalized("MessageFlow_3_cleanupHeaders")
  compression {
    compressMessage externalized("MessageFlow_3_CompressMessage")
    useMessageCompression externalized("MessageFlow_3_UseMessageCompression")
  }
  http {
    sendHttpResponseCode externalized("MessageFlow_3_sendHttpResponseCode")
    allowChunking externalized("MessageFlow_3_AllowChunking")
    keepAlive externalized("MessageFlow_3_KeepConnectionAlive")
  }
  proxy {
    type externalized("MessageFlow_3_proxyType")
    host externalized("MessageFlow_3_proxyHost")
  }
}
```

这可以支持同一个 archetype 在不同环境或业务对象上采用不同 externalization 策略。

### 5.5 Canonical Form 与幂等导入

同一个 IncomingInvoice iFlow 可能被多次提交给系统，或者从 SAP 下载时只改变 XML 格式、属性顺序、BPMN ID、坐标。导入器必须先生成 canonical form，再决定是创建新模型、更新 revision，还是识别为重复样例。

Canonical import pipeline：

```text
raw BPMN XML
  -> parse BPMN / ifl properties
  -> decode escaped SAP table XML
  -> classify participant / channel / process / step
  -> resolve sequence flow into process order
  -> assign semantic keys
  -> normalize parameters and expressions
  -> drop or isolate layout hints
  -> produce canonical内部模型
  -> compute semantic fingerprint
```

Canonical form 必须排除：

- BPMN element numeric suffix。
- shape / edge ID。
- waypoint 和 Bounds 坐标。
- XML attribute order、空 extensionElements、空 description。
- SAP 编辑器重排但语义不变的 sequence flow ID。

Canonical form 必须保留：

- participant role 和 system name。
- channel adapter type、direction、protocol、message protocol。
- typed adapter policy。
- process kind、step kind、normalized step name、step order。
- header/property/body assignments。
- namespace URI、service interface namespace、credential alias reference。
- externalization policy。

`semanticFingerprint` 应由 canonical内部模型计算，例如：

```text
semanticFingerprint = sha256(canonicalDslWithoutOriginAndLayout)
```

同一个 fingerprint 再次导入时，系统应记录新的 raw artifact checksum 和 origin refs，但不应创建重复 archetype。

### 5.6 参数 semantic name 与 SAP name

SAP externalized parameter 常带有 MessageFlow ID，例如 `MessageFlow_1_clientCertificate.issuerDN`。这个名字适合编译回 SAP artifact，但不适合作为内部模型的长期语义名，因为 MessageFlow ID 可能变化。

内部模型参数需要同时保存：

```text
parameter sender.clientCertificate.issuerDn {
  type certificateIssuer
  required false
  externalized {
    sapName "MessageFlow_1_clientCertificate.issuerDN"
    owner channel("REST_IncomingInvoice_Sender")
  }
}

parameter receiver.proxy.host {
  type host
  required false
  externalized {
    sapName "MessageFlow_3_proxyHost"
    owner channel("SOAP_Receiver_MM")
  }
}
```

Tools、validation、semantic diff 使用 semantic name；compiler projection 使用 `sapName`。如果 SAP 重新生成 MessageFlow ID，只需要更新 projection alias，不应影响业务内部模型。

## 6. 结构化序列化示意

API 和持久化层仍可使用 JSON，但 JSON 应表达同一个结构化模型，而不是一堆无约束属性。

```json
{
  "modelVersion": "0.2",
  "kind": "IFlow",
  "metadata": {
    "name": "REST InboundDelivery to S4 XI"
  },
  "participants": [
    { "id": "sender.csi_nsrm", "role": "sender", "name": "CSI_NSRM", "kind": "externalSystem" },
    { "id": "receiver.csi_p_s4", "role": "receiver", "name": "CSI_P_S4", "kind": "externalSystem" }
  ],
  "channels": [
    {
      "id": "channel.rest_inbound",
      "direction": "sender",
      "adapter": {
        "type": "https",
        "mode": "sender",
        "path": { "parameterRef": "MessageFlow_1_urlPath" },
        "auth": { "parameterRef": "MessageFlow_1_senderAuthType" }
      }
    }
  ],
  "processes": [
    {
      "id": "process.main",
      "kind": "integration",
      "steps": [
        {
          "id": "step.set_headers",
          "kind": "set",
          "headers": [
            { "name": "SAP_MessageType", "source": { "headerRef": "SAP_SenderInterface" } }
          ]
        }
      ]
    }
  ]
}
```

## 7. Process 与 Step 模型

### 7.1 ProcessKind

MVP 需要支持：

- `integration`
- `localIntegrationProcess`
- `exceptionSubprocess`

后续扩展：

- `eventSubprocess`
- `reusableProcess`
- `prePostExit`

### 7.2 StepKind

MVP 需要支持：

- `start`
- `end`
- `messageStart`
- `messageEnd`
- `errorStart`
- `errorEnd`
- `set`
- `script`
- `processCall`
- `jsonToXml`
- `xmlToJson`
- `requestReply`
- `router`
- `messageMapping`

后续扩展：

- SOAP sender / receiver
- SFTP sender / receiver
- JMS sender / receiver
- Splitter
- Gather
- Multicast
- Value mapping
- Content-based routing
- Exception handling strategies beyond simple error subprocess

### 7.3 Flow

`flow` 表达 process 内部控制流：

```text
flow {
  from step("JSON to XML Converter")
  to step("Set Props")
}
```

MVP 中可以由 step 顺序推导简单 sequence flow；显式 flow 用于 router、parallel split、join、exception path 和导入已有 iFlow 的 round-trip。

### 7.4 Channel Binding

`channel` 表达跨 participant 或跨 process 边界的消息流：

```text
channel "SOAP_Receiver_MM" {
  direction receiver
  from step("Request Reply")
  to participant("CSI_P_S4")
  adapter xi over http { ... }
}
```

这比把 receiver adapter 做成普通节点更清晰：request-reply 是流程步骤，XI receiver 是外部 channel，两者通过 binding 相连。

## 8. 参数、表达式与作用域

样例 iFlow 同时出现多类占位：

- `{{MessageFlow_1_urlPath}}`: Integration Suite externalized parameter。
- `${property.SAPHost}`: runtime exchange property。
- credential alias / tenant property: 不能落明文 secret。
- `global persisted variables`: 通过 content modifier 写入 exchange property。
- SAP table XML: `clientCertificates`、`headerTable`、`propertyTable`、`xmlJsonPathTable` 需要解析为结构化列表。

内部模型需要显式区分：

```text
parameter SAPHost : tenantProperty
parameter SAPCredentials : credentialAlias

address "http://${property.SAPHost}/sap/xi/engine?type=entry&sap-client=${property.SAPClient}"
```

表达式 source 必须类型化：

```text
constant("CSI_P_S4")
parameter("SAP_Sender")
property("SAPHost")
header("SAP_SenderInterface")
globalPersistedVariable("SAPClient")
```

## 9. 样例 iFlow 的抽象分析

### 9.1 全局配置

样例 collaboration 层包含：

- namespace mapping: `ns1`, `ns0`。
- HTTP session handling、CORS、logging、trace 等运行配置。
- component version 和 SAP variant URI。

内部模型应把 namespace 和运行策略类型化；component version / variant URI 作为 compiler projection 默认值或 import trace。

### 9.2 参与方与通道

样例有两个外部系统：

- `CSI_NSRM`: HTTPS sender。
- `CSI_P_S4`: XI over HTTP receiver。

以及两个 Integration Process：

- `Integration Process`: 主流程。
- `Get URL Param`: local integration process。

内部模型应把外部 participant、process participant 和 adapter channel 分开建模。

### 9.3 主流程

主流程语义顺序：

1. Message start。
2. 设置 header / property，建立 SAP sender、message type、日志开关。
3. 执行请求日志脚本。
4. 调用 local process 读取 URL path 和 query 参数。
5. JSON 转 XML，并添加 `MT_REQ` root 和 namespace。
6. 从 global persisted variables 读取 S/4 连接参数。
7. request-reply 调用 S/4 XI receiver。
8. XML 转 JSON，指定数组路径。
9. 设置 `SAP_Receiver` header。
10. 执行响应日志脚本。
11. Message end。
12. 旁路 exception subprocess：error start 到 error end。

这些都应是内部模型一等语义，不应靠 `activityType` 字符串和 `propertyTable` HTML 片段推断。

### 9.4 本地流程

`Get URL Param` 是 direct call local integration process：

1. Start。
2. Groovy `ReadUrlPath.groovy`。
3. Groovy `ReadUrlGetParameters.groovy`。
4. End。
5. Exception subprocess。

内部模型应支持 `processCall` 引用 local process，并校验目标 process 存在。

### 9.5 表格型属性

SAP XML 中 `headerTable`、`propertyTable`、`xmlJsonPathTable` 以 escaped XML table 存储。内部模型必须解析为结构化列表：

```text
headers [
  {
    action create
    name "SAP_MessageType"
    source header("SAP_SenderInterface")
  }
]

arrayPaths [
  "/ns0:MT_reponse/datas",
  "/ns0:MT_reponse/datas/head/message_detail"
]
```

编译器再负责把结构化列表投影回 SAP 期望的 table XML。

### 9.6 多样例归纳

导入多个相似 iFlow 时，importer 应能归纳出：

- stable archetype: 流程骨架、步骤顺序、通道类型、异常策略。
- instance variables: 业务对象、namespace、channel name、service interface、endpoint path。
- externalization policy: 哪些 adapter 属性固定，哪些使用 `{{...}}` 外部化，哪些来自 `${property.*}`。
- unstable editor artifacts: BPMN ID、shape ID、waypoint、随机 sequence ID。

这些归纳结果应进入知识库 / few-shot，使后续创建同类 iFlow 时可以从 archetype 实例化，而不是从空白图开始。

## 10. Tool 设计

工具可以继续走 JSON RPC 风格，但 LLM 只看到 tool schema。Tool 参数应面向业务语义 action，后端再映射为内部模型变更；不能要求 LLM 提交任意模型补丁。

每个 state mutation tool 的成功返回必须包含：

- `revision`: 新 model revision。
- `stateSummary`: compact iFlow summary，例如参与方、通道、流程步骤、未解决参数。
- `validationStatus`: `valid`、`warning`、`invalid`。
- `nextActions`: 可选的下一步建议 tool action。

每个失败返回必须是 AI-friendly error：

```json
{
  "status": "failed",
  "errorCode": "UNKNOWN_PROCESS_REFERENCE",
  "category": "validation",
  "severity": "error",
  "message": "Step Get URL Param references a local process that does not exist.",
  "affectedObject": {
    "kind": "step",
    "semanticKey": "process.main/processCall/Get URL Param"
  },
  "reason": "processCall.targetProcessName is Get URL Param, but no localIntegrationProcess with that name exists.",
  "suggestedFixes": [
    {
      "action": "addProcess",
      "arguments": {
        "name": "Get URL Param",
        "kind": "localIntegrationProcess",
        "callMode": "direct"
      }
    }
  ],
  "needsUserInput": false,
  "retryable": true
}
```

Tool 不应把 Java exception、SAP stack trace、完整 BPMN XML 或完整内部模型dump 直接返回给 LLM。低层信息应进入 diagnostics / trace，并用 `errorCode`、`affectedObject`、`suggestedFixes` 摘要给 LLM。

### 10.1 createIFlow

创建 iFlow 根对象。

```json
{
  "sessionId": "req_001",
  "name": "REST InboundDelivery to S4 XI",
  "packageId": "PKG_SALES"
}
```

### 10.2 addParticipant

```json
{
  "iflowId": "iflow_001",
  "role": "sender",
  "name": "CSI_NSRM",
  "kind": "externalSystem"
}
```

### 10.3 addSenderChannel / addReceiverChannel

```json
{
  "iflowId": "iflow_001",
  "name": "REST_InboundDelivery_Sender",
  "direction": "sender",
  "from": { "participantName": "CSI_NSRM" },
  "to": { "processName": "Integration Process", "port": "start" },
  "adapter": {
    "type": "https",
    "mode": "sender",
    "path": { "parameterRef": "MessageFlow_1_urlPath" }
  }
}
```

### 10.4 addProcess

```json
{
  "iflowId": "iflow_001",
  "name": "Get URL Param",
  "kind": "localIntegrationProcess",
  "callMode": "direct"
}
```

### 10.5 Typed step tools

LLM-facing tools 不暴露通用 `addStep(kind, config)`。每类 step 使用明确的 tool schema，后端内部再通过 `StepFactory` 创建类型化 step 对象。

#### addScriptStep

```json
{
  "iflowId": "iflow_001",
  "processName": "Integration Process",
  "name": "Log Request",
  "language": "groovy",
  "scriptFile": "LogPayload.groovy",
  "scriptCollection": "Common_Script_Collection",
  "function": "LogRequestMessage",
  "afterStep": "Set Header and Message Type"
}
```

#### addJsonToXmlConverter

```json
{
  "iflowId": "iflow_001",
  "processName": "Integration Process",
  "name": "JSON to XML Converter",
  "afterStep": "Get URL Param",
  "rootElement": "MT_REQ",
  "rootNamespace": "urn:csisolar.com:NSRM:S4:IncomingInvoice",
  "useNamespaces": false
}
```

#### addContentModifierStep

```json
{
  "iflowId": "iflow_001",
  "processName": "Integration Process",
  "name": "Set Header and Message Type",
  "headers": [
    {
      "name": "SAP_MessageType",
      "source": { "headerRef": "SAP_SenderInterface" }
    }
  ],
  "properties": [
    {
      "name": "LogRequest",
      "source": { "parameterRef": "LogRequest" }
    }
  ]
}
```

Backend-only factory abstraction：

```text
addScriptStep tool
  -> ScriptStepFactory
  -> ScriptStep
  -> ModelMutationService inserts into ProcessModel

addJsonToXmlConverter tool
  -> JsonToXmlConverterFactory
  -> JsonToXmlConverterStep
  -> ModelMutationService inserts into ProcessModel
```

`StepFactory`、`ChannelFactory`、`ProcessFactory` 是后端抽象，不是 LLM tool name。

### 10.6 connectSteps

简单线性流程可由 step order 推导；复杂流程应显式连接。

```json
{
  "iflowId": "iflow_001",
  "processName": "Integration Process",
  "fromStep": "Request Reply",
  "toStep": "XML to JSON Converter",
  "type": "sequence"
}
```

### 10.7 importIFlow

从已有 artifact / BPMN XML 解析为内部模型：

```json
{
  "artifactId": "REST_InboundDelivery_To_S4",
  "preserveOriginRefs": true,
  "unknownPropertyStrategy": "vendorExtensions",
  "idempotencyMode": "semantic-fingerprint"
}
```

导入结果必须返回 raw artifact checksum、semantic fingerprint、是否匹配已有内部模型/ archetype，以及被忽略的 editor-only diff。

### 10.8 deriveArchetype

从多个导入样例归纳模板：

```json
{
  "sourceIFlowIds": ["iflow_inbound_delivery", "iflow_incoming_invoice"],
  "semanticDiffMode": "ignore-editor-artifacts",
  "candidateName": "REST JSON to S4 XI request-reply"
}
```

### 10.9 instantiateArchetype

从模板生成新的业务 iFlow：

```json
{
  "archetypeId": "rest-json-to-s4-xi-request-reply",
  "instanceName": "IncomingInvoice",
  "variables": {
    "businessObject": "IncomingInvoice",
    "namespace.ns1": "urn:csisolar.com:NSRM:S4:IncomingInvoice",
    "senderChannel.name": "REST_IncomingInvoice_Sender",
    "receiverChannel.name": "SOAP_Receiver_MM"
  }
}
```

### 10.10 compareIFlows

比较两个导入结果或两个 model revision：

```json
{
  "leftIFlowId": "iflow_incoming_invoice_a",
  "rightIFlowId": "iflow_incoming_invoice_b",
  "mode": "semantic",
  "ignore": ["originRefs", "layoutHints", "editorGeneratedIds"]
}
```

重复导入同一 IncomingInvoice XML 时，期望 diff 为空或只包含 raw artifact checksum / origin refs。

## 11. Validation Rules

MVP 必须包含：

- channel 的 `from` / `to` 必须引用存在的 participant、process 或 step。
- LLM-facing state mutation tool 不允许接受 arbitrary model patch。
- Tool result 不允许默认返回完整内部模型；必须返回 compact state summary。
- Tool failure 必须返回 AI-friendly error object，并包含 `errorCode`、`affectedObject`、`reason`、`suggestedFixes`。
- `processCall` 必须引用存在的 local process。
- `requestReply` 必须绑定一个 receiver channel。
- HTTPS sender 必须有 path；认证类型不能保存明文 secret。
- XI receiver 必须有 address、timeout、credential alias 或明确的 anonymous 策略。
- 所有 `${property.*}` 引用必须来自已声明 property 写入或 tenant property 参数。
- 所有 `{{...}}` externalized parameter 必须声明类型。
- `set` step 中 header/property/body 的 source 必须类型化。
- 所有 externalized parameter 引用必须在参数目录中有类型、作用域和 required / optional 标记。
- archetype instance 必须填充所有 required variables。
- semantic diff 默认忽略 BPMN ID、shape ID、waypoint 和 SAP 编辑器生成的 sequence ID。
- 重复导入相同 semantic fingerprint 时必须幂等，不能创建重复 archetype。
- 每个 externalized parameter 必须同时有 semantic name 和 SAP projection name。
- 表格型属性导入后必须结构化，不能只保存 escaped XML 字符串。
- 生产 iFlow 必须有 exception subprocess 或显式声明无需异常处理。
- 内部模型禁止保存 password、secret、token、client secret。

## 12. Compiler Projection

编译器负责从内部模型生成 SAP artifact：

1. 为 process、step、flow、channel 分配稳定 BPMN ID。
2. 根据 adapter / step kind 选择 SAP variant 和 `cmdVariantUri`。
3. 把类型化配置投影为 `ifl:property`。
4. 把 table 配置序列化成 SAP 期望的 escaped XML table。
5. 生成 BPMN DI layout；没有 layout hint 时自动布局。
6. 打包脚本、schema、mapping 和参数文件。
7. 输出可上传的 iFlow ZIP。

导入已有 iFlow 后重新编译时，compiler 应尽量复用 origin refs，减少无意义 diff。

## 13. 与当前 Graph 实现的关系

当前代码中的 Graph node / edge 模型可以作为 MVP 存储骨架，但需要逐步升级：

- `NodeType` 应演进为 process-aware 的 `StepKind`，并区分 `participant`、`channel`、`step`。
- `properties: Map<String, Object>` 应演进为 typed config + `vendorExtensions`。
- `edges` 应区分 process 内部 `flow` 和跨 participant 的 `channel binding`。
- `resources` 应包含脚本、schema、mapping、script collection、message mapping artifact。
- `externalizedParameters` 应包含参数类型、作用域、默认值、是否敏感、是否必填。

这样既保留当前 Graph 工具的可迭代实现，又避免内部模型长期停留在“节点 + 边 + 任意 JSON 属性”的层次。
