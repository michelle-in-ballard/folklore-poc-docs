---
name: FolkloreContentService
dependencies:
  FolkloreContentServiceEncryption:
    classification: strong
    connects_to: [how-to-use-me]
    usage: "Team-owned encryption utilities package. Currently not directly used as the service uses VaultManager for encryption/decryption needs (VaultManagerUtil.java with VaultManagerEncryptionCodec). Available for team-specific encryption requirements beyond standard VaultManager functionality."
  ContentPipelineCommon:
    classification: strong
    connects_to: [how-to-use-me]
    usage: "Shared library for content pipeline services providing common data models, handlers, and utilities. Extensively used throughout FolkloreContentService for ContentHandler interfaces, data models (Author, Document, Session), event handling, and MetricsLake integration. Critical for action API implementations and cross-service consistency."
  SynthesisEngineModel:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "SynthesisEngine is an AI model service for document generation and editing. It's integrated as one of the core AI providers in FolkloreContentService's multi-model strategy, accessible through SynthesisEngineClientWrapper and used in ContentSynthesisProvider for document editing operations. The service uses feature gates to control access to SynthesisEngine model."
  CoverageAnalyzerAgent:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "CoverageAnalyzer agent for documentation coverage monitoring and gap tracking. Integrated in the main FolkloreContentService bootstrap process and configured through CoverageModule with application-specific settings. Started during service initialization to capture coverage metrics for monitoring and analysis purposes."
  FeatureGateCheck:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "FeatureGateCheck provides the internal feature access control framework, implementing hierarchical feature gating system. Extensively used throughout FolkloreContentService for API access control, AI model access (Claude, Sonnet, Haiku), and team-specific feature overrides. Core to the service's progressive model access strategy and user-based feature control mentioned in the implementation analysis."
  ServiceMeshClientLib:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "Service mesh client library for communicating with mesh agent for service discovery and container metadata. Used in ThrottlingModule to create ServiceMeshClientImpl for distributed throttling peer resolution. Essential for the service's ability to discover other instances in the cluster for coordinated rate limiting."
  ThrottleResolverLib:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "Throttle resolver library for distributed throttling peer discovery. Used by ThrottlingModule to implement LoadBalancingPeerResolver for discovering hosts under the same load balancer to create a mesh for distributed policy enforcement. Critical for the service's traffic shaping and rate limiting capabilities in container deployment."
  PortalContentGen:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "PortalContentGen provides portal-specific content generation capabilities. Integrated into FolkloreContentService through PortalContentModule in the main service module and provides PortalContentHandler for portal rendering routing. Part of the federated architecture where FolkloreContentService handles agent retrieval while PortalRenderService handles browser display."
  AuthGateService:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "AuthGate is the internal authorization service that provides user-team association validation and policy enforcement. FolkloreContentService uses AuthGateService to validate user-package relationships before processing content generation requests, ensuring users can only modify content in their authorized packages."
  AuthPolicyEngine:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "AuthPolicyEngine is the policy evaluation engine used by AuthGate for authorization decisions. It processes authorization policies to determine if users have access to specific resources and packages in FolkloreContentService."
  ContentModelSDK:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "Internal RPC framework SDK for generating service models and RPC interfaces. Critical for FolkloreContentService's RPC-based architecture, generating the service interfaces used by GenerateContentActivity and other RPC endpoints."
  SecurityEventsSupport:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "Internal security events framework for RPC services. Integrated in RPCModule to provide security event reporting capabilities through SecurityEventsHandler. Essential for compliance and security monitoring of the content generation service."
  InferenceServiceClient:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "Inference Service Java client for routing LLM inference through the unified inference endpoint. Used by InferenceModule to provide InferenceServiceClient with authentication and retry configuration. InferenceAccess wraps the client to call the Converse API. ClaudeInferencer dispatches to the inference service when enabled, with quality validation before the call."
  ContentScreeningServiceModel:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "Content Screening Service model for text content moderation. Actively used via ContentScreeningServiceClient in TextQualityHandler.kt and GenerateContentValidationHandler.java for validating content quality and safety. Configured through ValidationModule.java Guice bindings."
  MetricsAgentRPM:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "MetricsAgentRPM is a runtime package manager component that handles metrics collection and reporting infrastructure. Deployed as part of the runtime environment to enable system-level metrics gathering for FolkloreContentService operations."
  ServiceMonitoring:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "ServiceMonitoring provides runtime service health monitoring and alerting capabilities. Deployed as part of the runtime environment to monitor FolkloreContentService health, performance, and availability metrics."
---

# Development Guide - FolkloreContentService

## Architecture Patterns

### Multi-Model Provider Strategy Pattern

FolkloreContentService implements a progressive model selection strategy for AI content generation:

```java
public class ModelSelectionRegistry {
    private final Map<ContentComplexity, List<ModelProvider>> providerChain;

    public ModelProvider selectProvider(GenerateContentRequest request) {
        ContentComplexity complexity = complexityScorer.score(request);
        List<ModelProvider> chain = providerChain.get(complexity);

        for (ModelProvider provider : chain) {
            if (featureGateCheck.hasModelAccess(request.getUserId(), provider.getModelId())) {
                return provider;
            }
        }
        return fallbackProvider; // Haiku (always available)
    }
}
```

Model selection chain:
- **HIGH complexity** (cross-package synthesis, architecture docs): Claude Opus → Sonnet → Haiku
- **MEDIUM complexity** (standard topic files, updates): Sonnet → Haiku
- **LOW complexity** (formatting, metadata extraction): Haiku only

### Parallel Validation Pipeline Pattern

Content validation runs as a parallel pipeline with independent validators:

```java
public class ValidationPipeline {
    private final List<ContentValidator> validators = List.of(
        new SchemaValidator(),          // Frontmatter schema compliance
        new FreshnessValidator(),       // Staleness detection
        new QualityScoreValidator(),    // AI quality threshold
        new DependencyValidator(),      // Dependency graph validity
        new SecurityValidator()         // No secrets/credentials
    );

    public ValidationResult validate(GeneratedContent content) {
        return validators.parallelStream()
            .map(v -> v.validate(content))
            .reduce(ValidationResult::merge)
            .orElse(ValidationResult.pass());
    }
}
```

### Intelligent Fallback Strategy Pattern

When the primary AI model fails or is unavailable, the service cascades through fallback options:

```kotlin
class ContentGenerationFallbackStrategy(
    private val primaryProvider: ModelProvider,
    private val fallbackProviders: List<ModelProvider>,
    private val circuitBreaker: CircuitBreakerExecutor
) {
    suspend fun generate(request: GenerateContentRequest): GenerationResult {
        return try {
            circuitBreaker.execute {
                primaryProvider.generate(request)
            }
        } catch (e: ThrottlingException) {
            // AIMD circuit breaker opened — fall back
            fallbackProviders.firstNotNullOfOrNull { provider ->
                runCatching { provider.generate(request) }.getOrNull()
            } ?: GenerationResult.failure("All providers exhausted")
        }
    }
}
```

## Code Organization

### Source Code Structure
```
src/main/
├── java/com/internal/folklore/service/
│   ├── FolkloreContentServiceApplication.java    # Entry point
│   ├── activities/
│   │   ├── GenerateContentActivity.java          # RPC endpoint
│   │   ├── EditContentActivity.java              # Edit endpoint
│   │   └── ValidateContentActivity.java          # Validation endpoint
│   ├── handlers/
│   │   ├── ContentGenerationHandler.java         # Generation orchestration
│   │   ├── SynthesisHandler.java                 # Cross-package synthesis
│   │   ├── TopicFileHandler.java                 # Per-package topic files
│   │   └── ReleaseNoteHandler.java               # Release note generation
│   ├── providers/
│   │   ├── ModelProvider.java                    # Provider interface
│   │   ├── ClaudeOpusProvider.java               # Opus integration
│   │   ├── ClaudeSonnetProvider.java             # Sonnet integration
│   │   └── ClaudeHaikuProvider.java              # Haiku integration
│   ├── validators/
│   │   ├── SchemaValidator.java                  # Scout schema validation
│   │   ├── FreshnessValidator.java               # Staleness checks
│   │   ├── QualityScoreValidator.java            # Quality thresholds
│   │   └── DependencyValidator.java              # Dep graph validation
│   ├── modules/
│   │   ├── ServiceModule.java                    # Main Guice module
│   │   ├── InferenceModule.java                  # AI model wiring
│   │   ├── ValidationModule.java                 # Validator wiring
│   │   ├── ThrottlingModule.java                 # Rate limiting
│   │   └── AuthModule.java                       # Auth integration
│   └── models/
│       ├── GenerateContentRequest.java
│       ├── GenerateContentResponse.java
│       ├── TopicFile.java
│       └── SynthesisDocument.java
├── kotlin/com/internal/folklore/service/
│   ├── editing/
│   │   ├── ContentEditingHandler.kt              # Edit orchestration
│   │   └── IncrementalUpdateHandler.kt           # Diff-based updates
│   ├── synthesis/
│   │   ├── DependencyGraphTraverser.kt           # Graph walking
│   │   └── CrossPackageSynthesizer.kt            # Multi-package content
│   └── quality/
│       ├── ContentQualityScorer.kt               # AI quality scoring
│       └── TextQualityHandler.kt                 # Content screening
```

### Handler Organization Pattern

Handlers follow a consistent pattern:
1. **Validate input** — Schema compliance, required fields
2. **Authorize** — User has access to target package
3. **Select model** — Based on complexity + feature gates
4. **Generate/edit** — Call AI provider
5. **Validate output** — Schema + quality + safety
6. **Persist** — Store result, notify certifier

### Module-Based Dependency Injection

```java
public class ServiceModule extends AbstractModule {
    @Override
    protected void configure() {
        install(new InferenceModule());
        install(new ValidationModule());
        install(new ThrottlingModule());
        install(new AuthModule());
        install(new PortalContentModule());

        bind(ContentGenerationHandler.class).in(Singleton.class);
        bind(SynthesisHandler.class).in(Singleton.class);

        MapBinder<ContentType, ContentHandler> handlerBinder =
            MapBinder.newMapBinder(binder(), ContentType.class, ContentHandler.class);
        handlerBinder.addBinding(ContentType.TOPIC_FILE).to(TopicFileHandler.class);
        handlerBinder.addBinding(ContentType.SYNTHESIS).to(SynthesisHandler.class);
        handlerBinder.addBinding(ContentType.RELEASE_NOTE).to(ReleaseNoteHandler.class);
    }
}
```

## Key Classes

### Core Service Classes
- `FolkloreContentServiceApplication` — Service bootstrap and lifecycle
- `GenerateContentActivity` — Primary RPC endpoint for content generation
- `EditContentActivity` — Content editing and incremental updates
- `ValidateContentActivity` — On-demand content validation

### Handler Classes
- `ContentGenerationHandler` — Orchestrates full generation pipeline
- `SynthesisHandler` — Cross-package synthesis with dependency traversal
- `TopicFileHandler` — Per-package Scout topic file generation
- `ReleaseNoteHandler` — Release note generation from version diffs

### Provider Abstractions
```java
public interface ModelProvider {
    String getModelId();
    ModelTier getTier();
    GenerationResult generate(GenerationRequest request);
    boolean isAvailable();
    int getMaxTokens();
}
```

### Registry → Provider Contract
```java
public class ModelSelectionRegistry {
    // Complexity → ordered provider list
    // Each provider checked for availability + feature access
    // First accessible provider wins
    // Haiku always terminal fallback
}
```

### Validation Classes
- `SchemaValidator` — Validates Scout topic file frontmatter schema
- `FreshnessValidator` — Checks content age against code changes
- `QualityScoreValidator` — AI-scored quality threshold enforcement
- `DependencyValidator` — Validates dependency graph references exist

## Design Patterns

### Strategy Pattern
Used for model selection — different content types route to different AI model configurations without changing the handler logic.

### Registry Pattern
`ModelSelectionRegistry` maps content complexity → provider chains. `ContentHandlerRegistry` maps content types → handler implementations.

### Chain of Responsibility Pattern
Validation pipeline chains independent validators. Each validator passes or fails independently; results merge at the end.

### Feature Gate Dual-Path Pattern
```java
// Feature-gated model access with fallback
if (featureGateCheck.hasAccess(userId, "model.opus")) {
    return opusProvider.generate(request);
} else if (featureGateCheck.hasAccess(userId, "model.sonnet")) {
    return sonnetProvider.generate(request);
} else {
    return haikuProvider.generate(request); // Always available
}
```

### Circuit Breaker Pattern
```java
// AIMD circuit breaker for AI model calls
CircuitBreakerExecutor breaker = CircuitBreakerExecutor.builder()
    .failureThreshold(5)
    .recoveryTimeout(Duration.ofSeconds(30))
    .halfOpenRequests(3)
    .build();
```

## Code Style Guidelines

### Multi-Language Standards
- **Java**: Google Java Style Guide with internal extensions
- **Kotlin**: Official Kotlin Coding Conventions + Detekt rules
- **Both**: Maximum line length 120 characters, 4-space indent

### Naming Conventions
- Handlers: `*Handler.java/kt`
- Providers: `*Provider.java`
- Validators: `*Validator.java`
- Modules: `*Module.java`
- Models: Descriptive noun (e.g., `TopicFile`, `SynthesisDocument`)

### Error Handling Standards
```java
// Always wrap external service calls
try {
    return inferenceClient.converse(request);
} catch (ThrottlingException e) {
    metrics.increment("inference.throttled");
    throw new ServiceUnavailableException("AI model temporarily unavailable", e);
} catch (ValidationException e) {
    metrics.increment("inference.validation_failed");
    throw new BadRequestException("Content failed quality validation", e);
}
```

## Development Best Practices

### Parallel Processing Best Practices
- Use `CompletableFuture` for Java async operations
- Use Kotlin coroutines for Kotlin async operations
- Always set timeouts on external calls (5s default)
- Use structured concurrency patterns

### Feature Gating Best Practices
- Always check feature access before model calls
- Log feature gate decisions for debugging
- Use fallback paths for every gated feature
- Never hard-code feature access — always go through FeatureGateCheck

### Content Safety Best Practices
- Validate all generated content before persisting
- Never bypass quality checks, even in development
- Log all validation failures with request context
- Implement retry with different prompts on quality failure

### Performance Optimization
- Cache feature gate decisions for 60s
- Use batch inference for multi-file generation
- Pre-compute dependency graphs at startup
- Pool HTTP connections to inference service
