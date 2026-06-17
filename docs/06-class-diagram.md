# 核心类图

## 1. 分层关系

LLM-facing 层只暴露具体 tools。后端内部可以用 `StepFactory`、`ChannelFactory` 等工厂统一创建类型化对象，但这些工厂不是 LLM tool name。

```mermaid
classDiagram
    class LlmToolFacade {
      +createIFlow(cmd) ToolResult
      +addParticipant(cmd) ToolResult
      +addSenderChannel(cmd) ToolResult
      +addReceiverChannel(cmd) ToolResult
      +addScriptStep(cmd) ToolResult
      +addContentModifierStep(cmd) ToolResult
      +addJsonToXmlConverter(cmd) ToolResult
      +addXmlToJsonConverter(cmd) ToolResult
      +addProcessCallStep(cmd) ToolResult
      +addRequestReplyStep(cmd) ToolResult
      +connectSteps(cmd) ToolResult
      +setAdapterPolicy(cmd) ToolResult
      +validateIFlow(id) ToolResult
      +compileIFlow(id) ToolResult
    }

    class IFlowApplicationService {
      +createIFlow(cmd) ToolResult
      +addChannel(cmd) ToolResult
      +addTypedStep(cmd) ToolResult
      +connectSteps(cmd) ToolResult
      +validate(id) ValidationReport
      +compile(id) IFlowArtifact
    }

    class ToolResult {
      +status
      +revision
      +stateSummary
      +validationStatus
      +nextActions
      +error
    }

    class AiFriendlyError {
      +errorCode
      +category
      +severity
      +message
      +affectedObject
      +reason
      +suggestedFixes
      +needsUserInput
      +retryable
    }

    LlmToolFacade --> IFlowApplicationService
    LlmToolFacade --> ToolResult
    ToolResult --> AiFriendlyError
```

## 2. 类型化 iFlow 内部模型

```mermaid
classDiagram
    class IFlowModel {
      +id
      +name
      +packageId
      +version
      +metadata
      +participants
      +channels
      +processes
      +resources
      +parameters
      +layoutHints
      +originRefs
      +semanticFingerprint
    }

    class Participant {
      +id
      +name
      +role
      +systemRef
      +originRef
    }

    class Channel {
      +id
      +name
      +direction
      +fromRef
      +toRef
      +adapterConfig
      +originRef
    }

    class ProcessModel {
      +id
      +name
      +kind
      +transactionHandling
      +steps
      +flows
      +exceptionSubprocess
      +originRef
    }

    class Flow {
      +id
      +sourceStepRef
      +targetStepRef
      +type
      +condition
      +originRef
    }

    class Step {
      <<abstract>>
      +id
      +name
      +kind
      +semanticKey
      +originRef
    }

    class ScriptStep {
      +language
      +scriptFile
      +scriptCollection
      +functionName
    }

    class ContentModifierStep {
      +headers
      +properties
      +body
    }

    class JsonToXmlConverterStep {
      +rootElement
      +rootNamespace
      +useNamespaces
    }

    class XmlToJsonConverterStep {
      +suppressRootElement
      +arrayPaths
      +encoding
    }

    class ProcessCallStep {
      +targetProcessRef
      +callMode
    }

    class RequestReplyStep {
      +receiverChannelRef
    }

    class AdapterConfig {
      <<abstract>>
      +type
      +direction
      +externalizationPolicy
      +vendorExtensions
    }

    class HttpsSenderAdapterConfig {
      +path
      +authType
      +userRole
      +maxBodySizeMb
      +clientCertificate
    }

    class XiReceiverAdapterConfig {
      +address
      +credentialAlias
      +serviceInterface
      +qualityOfService
      +retryPolicy
      +proxy
      +timeoutMs
    }

    class ParameterDefinition {
      +semanticName
      +sapExternalizedName
      +type
      +scope
      +required
      +secretRef
      +defaultValue
    }

    class ResourceFile {
      +name
      +type
      +path
      +checksum
    }

    IFlowModel "1" --> "*" Participant
    IFlowModel "1" --> "*" Channel
    IFlowModel "1" --> "*" ProcessModel
    IFlowModel "1" --> "*" ParameterDefinition
    IFlowModel "1" --> "*" ResourceFile
    ProcessModel "1" --> "*" Step
    ProcessModel "1" --> "*" Flow
    Channel --> AdapterConfig

    Step <|-- ScriptStep
    Step <|-- ContentModifierStep
    Step <|-- JsonToXmlConverterStep
    Step <|-- XmlToJsonConverterStep
    Step <|-- ProcessCallStep
    Step <|-- RequestReplyStep

    AdapterConfig <|-- HttpsSenderAdapterConfig
    AdapterConfig <|-- XiReceiverAdapterConfig
```

## 3. Factories、Repository、Validator、Compiler

```mermaid
classDiagram
    class IFlowRepository {
      +findById(id) IFlowModel
      +save(model) IFlowModel
      +saveRevision(model) IFlowModel
      +findBySemanticFingerprint(fingerprint) IFlowModel
    }

    class ModelMutationService {
      +insertChannel(model, channel)
      +insertStep(model, processRef, step, placement)
      +connectSteps(model, command)
      +updateAdapterPolicy(model, command)
    }

    class ChannelFactory {
      +createSender(cmd, context) Channel
      +createReceiver(cmd, context) Channel
    }

    class StepFactory {
      <<interface>>
      +create(cmd, context) Step
    }

    class ScriptStepFactory
    class ContentModifierStepFactory
    class JsonToXmlConverterFactory
    class XmlToJsonConverterFactory
    class ProcessCallStepFactory
    class RequestReplyStepFactory

    class IFlowValidator {
      +validate(model) ValidationReport
      +validateAffected(model, semanticKey) ValidationReport
    }

    class ValidationReport {
      +status
      +issues
    }

    class ModelIssue {
      +code
      +severity
      +affectedObject
      +message
      +suggestedFixes
    }

    class BpmnProjectionCompiler {
      +compile(model) IFlowArtifact
    }

    class BpmnElementWriter {
      +writeParticipant(participant)
      +writeChannel(channel)
      +writeProcess(process)
      +writeStep(step)
      +writeFlow(flow)
    }

    class IFlowArtifact {
      +zipPath
      +checksum
      +manifest
    }

    IFlowApplicationService --> IFlowRepository
    IFlowApplicationService --> ModelMutationService
    IFlowApplicationService --> ChannelFactory
    IFlowApplicationService --> StepFactory
    IFlowApplicationService --> IFlowValidator
    IFlowApplicationService --> BpmnProjectionCompiler

    StepFactory <|.. ScriptStepFactory
    StepFactory <|.. ContentModifierStepFactory
    StepFactory <|.. JsonToXmlConverterFactory
    StepFactory <|.. XmlToJsonConverterFactory
    StepFactory <|.. ProcessCallStepFactory
    StepFactory <|.. RequestReplyStepFactory

    IFlowValidator --> ValidationReport
    ValidationReport --> ModelIssue
    ModelIssue --> AiFriendlyError

    BpmnProjectionCompiler --> BpmnElementWriter
    BpmnProjectionCompiler --> IFlowArtifact
```

## 4. Import 与 Archetype 支撑类

```mermaid
classDiagram
    class IFlowImporter {
      +importArtifact(input) ImportResult
    }

    class BpmnParser {
      +parse(xml) RawBpmnModel
    }

    class SapTableDecoder {
      +decodeHeaderTable(value)
      +decodePropertyTable(value)
      +decodePathTable(value)
    }

    class CanonicalModelBuilder {
      +build(rawBpmn) IFlowModel
    }

    class SemanticFingerprintService {
      +compute(model) String
    }

    class ArchetypeService {
      +derive(models) IFlowArchetype
      +instantiate(archetype, variables) IFlowModel
    }

    class ImportResult {
      +rawChecksum
      +semanticFingerprint
      +matchedModelId
      +matchedArchetypeId
      +ignoredEditorDiffs
      +modelRevision
    }

    IFlowImporter --> BpmnParser
    IFlowImporter --> SapTableDecoder
    IFlowImporter --> CanonicalModelBuilder
    CanonicalModelBuilder --> SemanticFingerprintService
    IFlowImporter --> ArchetypeService
    IFlowImporter --> ImportResult
```
