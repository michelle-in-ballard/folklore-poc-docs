---
name: FolkloreContentService
dependencies: {}
---

# API Guide - FolkloreContentService

## Getting Started

FolkloreContentService provides AI-generated documentation for the Folklore developer portal through two main API approaches:

1. **Direct RPC API** - Traditional service-to-service communication
2. **Agent Action APIs** - Integration with AI coding agent systems

The service supports content generation from source code analysis and editing of previously generated documentation with comprehensive quality validation and multi-model AI provider support.

## Installation

### Service Integration
FolkloreContentService is deployed as an RPC service and can be integrated through:

- **RPC Framework**: Direct service-to-service calls via GenerateContentActivity
- **Agent Action APIs**: Integration through ContentGeneration action APIs
- **Federated Architecture**: Automatic content-type-based routing

### Client Requirements
- **Authorization**: AuthGate user-package association validation
- **Feature Access**: FeatureGateCheck for API and model access control
- **Content Quality**: Quality validation integration for input/output checks

## Public API

### Interfaces

#### GenerateContent API (Direct Service)
**Interface**: `IGenerateContentActivity.generateContent()`

**Request Model**: `GenerateContentRequest`
```java
- packageName: String (target package for generation)
- contentType: ContentType (TopicFile, Synthesis, ReleaseNote, Index)
- topicName: String (optional, specific topic file)
- requestMetadata: RequestMetadata
  - userId: String
  - sessionId: String
  - workspace: Workspace
  - modelTier: ModelTier
```

**Response Model**: `GenerateContentResponse`
```java
- contentItems: List<ContentItem>
- responseMetadata: ResponseMetadata
  - resultStatus: String
  - failureReason: String (if failed)
  - modelUsed: String
  - tokensConsumed: int
```

#### ContentGeneration Action APIs (Agent Integration)

**GenerateTopicFile Action API**
```java
@Name("packageName") String packageName
@Name("topicName") String topicName
@Name("modelTier") String modelTier (optional, default SONNET)
@Name("includeDepGraph") boolean includeDepGraph (optional, default true)
```

**EditContent Action API**
```java
@Name("editRequest") String editRequest
@Name("packageName") String packageName
@Name("topicName") String topicName
@Name("editType") EditType editType (optional)
@Name("preserveFrontmatter") boolean preserveFrontmatter (optional, default true)
```

**SynthesizeWorkflow Action API**
```java
@Name("workflowName") String workflowName
@Name("packageNames") List<String> packageNames
@Name("depthLimit") int depthLimit (optional, default 2)
@Name("modelTier") String modelTier (optional, default OPUS)
```

### Methods

#### Core Content Types
```java
public enum ContentType {
    TOPIC_FILE,    // Per-package Scout documentation
    SYNTHESIS,     // Cross-package workflow guides
    RELEASE_NOTE,  // Auto-generated release notes
    INDEX          // Registry-driven package listing
}
```

#### Edit Types
```java
public enum EditType {
    FULL_REWRITE,    // Complete content regeneration
    INCREMENTAL,     // Diff-based update from code changes
    SECTION_UPDATE,  // Update specific section only
    FRONTMATTER_ONLY // Update dependencies/metadata only
}
```

#### Response Models
**ContentGenerationResult** (returned by all generation APIs)
```java
- resultStatus: String
- failureReason: String          // @Optional
- contentType: ContentType
- modelUsed: String              // @Optional
- tokensConsumed: int            // @Optional
- generatedContent: String       // @Optional — the markdown content
- validationStatus: String       // PASSED, FAILED, SKIPPED
- qualityScore: double           // @Optional, 0.0-1.0
- packageName: String
- topicName: String              // @Optional
```

## Usage Examples

### Direct RPC API Usage
```java
// Create request
GenerateContentRequest request = GenerateContentRequest.builder()
    .packageName("VegaScriptCore")
    .contentType(ContentType.TOPIC_FILE)
    .topicName("what-is-it")
    .requestMetadata(RequestMetadata.builder()
        .userId("engineer-001")
        .sessionId("session-456")
        .workspace(workspace)
        .modelTier(ModelTier.SONNET)
        .build())
    .build();

// Call service
GenerateContentResponse response = generateContentActivity.generateContent(request);

// Process response
if ("SUCCESS".equals(response.getResponseMetadata().getResultStatus())) {
    List<ContentItem> items = response.getContentItems();
    // Handle generated topic files
}
```

### Agent Action API Usage
```java
// Generate topic file for a package
ContentGenerationResult result = contentGeneration.generateTopicFile(
    "VegaScriptCore",              // packageName
    "what-is-it",                  // topicName
    "SONNET",                      // modelTier
    true                           // includeDepGraph
);

// Edit existing content
ContentGenerationResult editResult = contentGeneration.editContent(
    "Update the dependencies section",    // editRequest
    "VegaScriptCore",                     // packageName
    "what-is-it",                         // topicName
    EditType.SECTION_UPDATE,              // editType
    true                                  // preserveFrontmatter
);
```

**Cross-Package Synthesis**
```java
// Generate workflow guide spanning multiple packages
ContentGenerationResult synthesisResult = contentGeneration.synthesizeWorkflow(
    "build-test-ship",                           // workflowName
    List.of("VegaScriptCore", "VegaCI",         // packageNames
             "VegaUIAutomator", "VegaPipeline"),
    2,                                           // depthLimit
    "OPUS"                                       // modelTier
);
```

## Best Practices

### Input Validation
- **Package Name**: Must match an existing package in the registry
- **Topic Name**: Must be one of the 6 standard topics or a custom nested path
- **Model Tier**: Use SONNET for standard generation, OPUS only for synthesis
- **Content Guidelines**: Follow Scout schema to pass validation

### Error Handling
- **400 BadRequestException**: Invalid input, unknown package or topic name
- **403 ForbiddenException**: User lacks access to target package
- **429 ThrottlingException**: Rate limiting - implement retry with backoff
- **503 ServiceUnavailableException**: AI model unavailable, capacity issues

### Performance Optimization
- **Batch Generation**: Use batch endpoint for generating all 6 topic files at once
- **Incremental Updates**: Prefer INCREMENTAL edit type over FULL_REWRITE
- **Caching**: Service caches dependency graphs for 5 minutes
- **Timeout Handling**: Expect 5-second timeouts for AI model calls
- **Fallback Strategy**: Service automatically falls back to simpler models when needed

## Real-World Examples

### Integration Points
- **Kiro IDE**: Agent action APIs for in-editor content generation
- **Pipeline Hooks**: Pre-CR hook triggers incremental updates
- **Coverage Dashboard**: Batch validation of all packages
- **Portal Rendering**: Content aggregation for Mintlify deployment

### Content Type-Specific Usage
- **Topic Files**: Standard 6-file generation per package (what-is-it, how-to-build, etc.)
- **Synthesis**: Cross-package workflow guides spanning 3-20 packages
- **Release Notes**: Consume version set diffs and produce categorized markdown
- **Index**: Generate searchable package registry from ownership data

### Quality Validation Integration
- **Schema Validation**: All generated content validated against Scout frontmatter schema
- **Quality Scoring**: AI-scored quality threshold (minimum 0.7 for publication)
- **Freshness Check**: Content compared against latest code commits
- **Dependency Validation**: All referenced packages must exist in registry

## Appendix
[Any other relevant information]
