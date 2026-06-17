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
      +execute(command) ToolResult
      +stateMarkdown(id) ToolResult
      +rollback(id, revision) ToolResult
      +validate(id) ToolResult
      +compile(id) ToolResult
    }

    class OrchestrationGuard {
      +authorize(toolCall, session) GuardDecision
      +recordResult(toolCall, result)
      +detectLoop(session) LoopAssessment
      +remainingBudget(session) ExecutionBudget
    }

    class ExecutionBudget {
      +maxToolCalls
      +maxMutations
      +maxCompileAttempts
      +maxDeployAttempts
      +maxAutoFixIterations
    }

    class ProgressTracker {
      +assess(before, after) ProgressAssessment
      +hasModelProgress()
      +hasValidationProgress()
      +hasRuntimeProgress()
    }

    class LoopDetector {
      +detectRepeatedCommand(history)
      +detectRepeatedError(history)
      +detectNoOpMutation(history)
    }

    class CircuitBreaker {
      +allow(system, operation) boolean
      +recordSuccess(system, operation)
      +recordFailure(system, operation)
    }

    class HumanHandoffPolicy {
      +shouldEscalate(loopAssessment) boolean
      +buildEscalation(error) AiFriendlyError
    }

    class IFlowBackendRegistry {
      +resolve(workspace) IFlowBackend
    }

    class IFlowBackend {
      <<interface>>
      +maintainer(workspace) IFlowMaintainer
      +validator() IFlowValidator
      +compiler() IFlowCompiler
    }

    class IFlowMaintainer {
      <<interface>>
      +apply(command) ToolResult
      +snapshot() IFlowDocument
      +rollbackTo(revision) ToolResult
      +renderMarkdown(options) String
      +renderMermaid(options) String
    }

    class IFlowDocument {
      <<interface>>
      +id
      +kind
      +revision
      +semanticFingerprint
    }

    class IFlowCompiler {
      <<interface>>
      +compile(document) IFlowArtifact
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

    LlmToolFacade --> OrchestrationGuard
    OrchestrationGuard --> ExecutionBudget
    OrchestrationGuard --> ProgressTracker
    OrchestrationGuard --> LoopDetector
    OrchestrationGuard --> CircuitBreaker
    OrchestrationGuard --> HumanHandoffPolicy
    OrchestrationGuard --> IFlowApplicationService
    IFlowApplicationService --> IFlowBackendRegistry
    IFlowBackendRegistry --> IFlowBackend
    IFlowBackend --> IFlowMaintainer
    IFlowBackend --> IFlowCompiler
    IFlowMaintainer --> IFlowDocument
    IFlowCompiler --> IFlowDocument
    LlmToolFacade --> ToolResult
    ToolResult --> AiFriendlyError
```

## 1.1 Backend 实现组合

```mermaid
classDiagram
    class IFlowBackend {
      <<interface>>
    }
    class IFlowMaintainer {
      <<interface>>
    }
    class IFlowCompiler {
      <<interface>>
    }
    class IFlowDocument {
      <<interface>>
    }

    class BpmnIFlowBackend
    class BpmnIFlowMaintainer
    class BpmnIFlowCompiler
    class BpmnIFlowDocument

    class TypedModelIFlowBackend
    class TypedModelIFlowMaintainer
    class TypedModelToBpmnCompiler
    class TypedIFlowDocument

    class JsonIFlowBackend
    class JsonIFlowMaintainer
    class JsonToBpmnCompiler
    class JsonIFlowDocument

    IFlowBackend <|.. BpmnIFlowBackend
    IFlowMaintainer <|.. BpmnIFlowMaintainer
    IFlowCompiler <|.. BpmnIFlowCompiler
    IFlowDocument <|.. BpmnIFlowDocument

    IFlowBackend <|.. TypedModelIFlowBackend
    IFlowMaintainer <|.. TypedModelIFlowMaintainer
    IFlowCompiler <|.. TypedModelToBpmnCompiler
    IFlowDocument <|.. TypedIFlowDocument

    IFlowBackend <|.. JsonIFlowBackend
    IFlowMaintainer <|.. JsonIFlowMaintainer
    IFlowCompiler <|.. JsonToBpmnCompiler
    IFlowDocument <|.. JsonIFlowDocument
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
      +type: EdgeType
      +condition
      +originRef
    }

    class Step {
      <<abstract>>
      +id
      +name
      +kind: StepType
      +semanticKey
      +originRef
    }

    class StepType {
      <<enumeration>>
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
    }

    class EdgeType {
      <<enumeration>>
      SEQUENCE_FLOW
      MESSAGE_FLOW
      EXCEPTION_FLOW
      ROUTER_BRANCH
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
    Step --> StepType
    Flow --> EdgeType

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
      +addEdge(model, command)
      +updateEdge(model, command)
      +deleteEdge(model, command)
      +deleteStep(model, command)
      +updateAdapterPolicy(model, command)
    }

    class RevisionManager {
      +createRevision(model, reason)
      +rollbackTo(model, revision)
      +lastValidRevision(modelId)
    }

    class StateRenderer {
      +renderMarkdown(document, options) String
      +renderMermaid(document, options) String
      +renderCompactSummary(document) StateSummary
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

    class TypedModelToBpmnCompiler {
      +compile(model) IFlowArtifact
    }

    class IFlowCompiler {
      <<interface>>
      +compile(document) IFlowArtifact
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
    IFlowApplicationService --> RevisionManager
    IFlowApplicationService --> StateRenderer
    IFlowApplicationService --> ChannelFactory
    IFlowApplicationService --> StepFactory
    IFlowApplicationService --> IFlowValidator
    IFlowApplicationService --> IFlowCompiler

    StepFactory <|.. ScriptStepFactory
    StepFactory <|.. ContentModifierStepFactory
    StepFactory <|.. JsonToXmlConverterFactory
    StepFactory <|.. XmlToJsonConverterFactory
    StepFactory <|.. ProcessCallStepFactory
    StepFactory <|.. RequestReplyStepFactory

    IFlowValidator --> ValidationReport
    ValidationReport --> ModelIssue
    ModelIssue --> AiFriendlyError

    IFlowCompiler <|.. TypedModelToBpmnCompiler
    TypedModelToBpmnCompiler --> BpmnElementWriter
    TypedModelToBpmnCompiler --> IFlowArtifact
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

## 5. 全局 SPI 扩展点

所有组件都必须依赖接口，不依赖具体实现。新增维护方式、编译方式、知识库、外部客户端、持久化存储或 guard policy 时，不应修改 LLM-facing tool contract。

```mermaid
classDiagram
    class ToolProvider {
      <<interface>>
      +tools() List~ToolDefinition~
    }

    class ToolExecutor {
      <<interface>>
      +execute(toolCall, context) ToolResult
    }

    class IFlowBackend {
      <<interface>>
      +supports(workspace) boolean
      +maintainer(workspace) IFlowMaintainer
      +validator() IFlowValidator
      +compiler() IFlowCompiler
    }

    class IFlowMaintainer {
      <<interface>>
      +apply(command) ToolResult
      +snapshot() IFlowDocument
      +rollbackTo(revision) ToolResult
      +renderMarkdown(options) String
      +renderMermaid(options) String
    }

    class IFlowCompiler {
      <<interface>>
      +compile(document) IFlowArtifact
    }

    class IFlowValidator {
      <<interface>>
      +validate(document) ValidationReport
    }

    class StateRenderer {
      <<interface>>
      +renderMarkdown(document, options) String
      +renderMermaid(document, options) String
      +renderSummary(document, options) StateSummary
    }

    class GuardPolicy {
      <<interface>>
      +evaluate(toolCall, context) GuardDecision
    }

    class KnowledgeRetriever {
      <<interface>>
      +retrieve(query) KnowledgeResult
    }

    class ArchetypeRepository {
      <<interface>>
      +findCandidates(fingerprint) List~IFlowArchetype~
      +save(archetype)
    }

    class IFlowRepository {
      <<interface>>
      +findById(id) IFlowDocument
      +saveRevision(document)
      +findByFingerprint(fingerprint) IFlowDocument
    }

    class ArtifactStore {
      <<interface>>
      +put(artifact) ArtifactRef
      +get(ref) IFlowArtifact
    }

    class SapDesignTimeClient {
      <<interface>>
      +upload(artifact) DeploymentRef
      +deploy(ref) DeploymentStatus
    }

    class SapRuntimeClient {
      <<interface>>
      +runSmokeTest(endpoint, request) SmokeTestResult
    }

    class MplTraceClient {
      <<interface>>
      +readMpl(id) MplSummary
      +readTrace(id) TraceSummary
    }

    class ODataMetadataClient {
      <<interface>>
      +getMetadata(system, service) ODataMetadata
    }

    ToolProvider --> ToolExecutor
    ToolExecutor --> GuardPolicy
    ToolExecutor --> IFlowBackend
    IFlowBackend --> IFlowMaintainer
    IFlowBackend --> IFlowValidator
    IFlowBackend --> IFlowCompiler
    IFlowMaintainer --> StateRenderer
    IFlowMaintainer --> IFlowRepository
    IFlowCompiler --> ArtifactStore
    ToolExecutor --> KnowledgeRetriever
    ToolExecutor --> ArchetypeRepository
    ToolExecutor --> SapDesignTimeClient
    ToolExecutor --> SapRuntimeClient
    ToolExecutor --> MplTraceClient
    ToolExecutor --> ODataMetadataClient
```

## 6. Guardrail 策略类图

```mermaid
classDiagram
    class GuardPolicy {
      <<interface>>
      +evaluate(toolCall, context) GuardDecision
    }

    class BudgetGuardPolicy {
      +evaluate(toolCall, context) GuardDecision
    }

    class RepeatedToolCallPolicy {
      +evaluate(toolCall, context) GuardDecision
    }

    class RepeatedErrorPolicy {
      +evaluate(toolCall, context) GuardDecision
    }

    class NoOpMutationPolicy {
      +evaluate(toolCall, context) GuardDecision
    }

    class CircuitBreakerPolicy {
      +evaluate(toolCall, context) GuardDecision
    }

    class HumanHandoffPolicy {
      +evaluate(toolCall, context) GuardDecision
    }

    class GuardDecision {
      +allowed
      +reason
      +error
      +requiredAction
    }

    class GuardContext {
      +sessionId
      +toolHistory
      +revisionHistory
      +errorHistory
      +budget
      +lastValidRevision
    }

    GuardPolicy <|.. BudgetGuardPolicy
    GuardPolicy <|.. RepeatedToolCallPolicy
    GuardPolicy <|.. RepeatedErrorPolicy
    GuardPolicy <|.. NoOpMutationPolicy
    GuardPolicy <|.. CircuitBreakerPolicy
    GuardPolicy <|.. HumanHandoffPolicy
    GuardPolicy --> GuardDecision
    GuardPolicy --> GuardContext
    GuardDecision --> AiFriendlyError
```
