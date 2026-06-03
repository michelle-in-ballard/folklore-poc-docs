---
name: FolkloreContentService
dependencies:
  FolkloreContentServiceEncryption:
    classification: strong
    connects_to: [what-is-it]
    usage: "Team-owned encryption utilities package. Currently not directly used as the service uses VaultManager for encryption/decryption needs (VaultManagerUtil.java with VaultManagerEncryptionCodec). Available for team-specific encryption requirements beyond standard VaultManager functionality."
  ContentPipelineCommon:
    classification: strong
    connects_to: [what-is-it]
    usage: "Shared library for content pipeline services providing common data models, handlers, and utilities. Extensively used throughout FolkloreContentService for ContentHandler interfaces, data models (Author, Document, Session), event handling, and MetricsLake integration. Critical for action API implementations and cross-service consistency."
  SynthesisEngineModel:
    classification: weak
    connects_to: [what-is-it]
    usage: "SynthesisEngine is an AI model service for document generation and editing. It's integrated as one of the core AI providers in FolkloreContentService's multi-model strategy, accessible through SynthesisEngineClientWrapper and used in ContentSynthesisProvider for document editing operations. The service uses feature gates to control access to SynthesisEngine model."
  CoverageAnalyzerAgent:
    classification: weak
    connects_to: [what-is-it]
    usage: "CoverageAnalyzer agent for documentation coverage monitoring and gap tracking. Integrated in the main FolkloreContentService bootstrap process and configured through CoverageModule with application-specific settings. Started during service initialization to capture coverage metrics for monitoring and analysis purposes."
  FeatureGateCheck:
    classification: weak
    connects_to: [what-is-it]
    usage: "FeatureGateCheck provides the internal feature access control framework, implementing hierarchical feature gating system. Extensively used throughout FolkloreContentService for API access control, AI model access (Claude, Sonnet, Haiku), and team-specific feature overrides. Core to the service's progressive model access strategy and user-based feature control mentioned in the implementation analysis."
  ServiceMeshClientLib:
    classification: weak
    connects_to: [what-is-it]
    usage: "Service mesh client library for communicating with mesh agent for service discovery and container metadata. Used in ThrottlingModule to create ServiceMeshClientImpl for distributed throttling peer resolution. Essential for the service's ability to discover other instances in the cluster for coordinated rate limiting."
  ThrottleResolverLib:
    classification: weak
    connects_to: [what-is-it]
    usage: "Throttle resolver library for distributed throttling peer discovery. Used by ThrottlingModule to implement LoadBalancingPeerResolver for discovering hosts under the same load balancer to create a mesh for distributed policy enforcement. Critical for the service's traffic shaping and rate limiting capabilities in container deployment."
  PortalContentGen:
    classification: weak
    connects_to: [what-is-it]
    usage: "PortalContentGen provides portal-specific content generation capabilities. Integrated into FolkloreContentService through PortalContentModule in the main service module and provides PortalContentHandler for portal rendering routing. Part of the federated architecture where FolkloreContentService handles agent retrieval while PortalRenderService handles browser display."
  AuthGateService:
    classification: weak
    connects_to: [what-is-it]
    usage: "AuthGate is the internal authorization service that provides user-team association validation and policy enforcement. FolkloreContentService uses AuthGateService to validate user-package relationships before processing content generation requests, ensuring users can only modify content in their authorized packages."
  AuthPolicyEngine:
    classification: weak
    connects_to: [what-is-it]
    usage: "AuthPolicyEngine is the policy evaluation engine used by AuthGate for authorization decisions. It processes authorization policies to determine if users have access to specific resources and packages in FolkloreContentService."
  ContentModelSDK:
    classification: weak
    connects_to: [what-is-it]
    usage: "Internal RPC framework SDK for generating service models and RPC interfaces. Critical for FolkloreContentService's RPC-based architecture, generating the service interfaces used by GenerateContentActivity and other RPC endpoints."
  SecurityEventsSupport:
    classification: weak
    connects_to: [what-is-it]
    usage: "Internal security events framework for RPC services. Integrated in RPCModule to provide security event reporting capabilities through SecurityEventsHandler. Essential for compliance and security monitoring of the content generation service."
---

# FolkloreContentService

## Overview

FolkloreContentService is a Java/Kotlin backend service that powers AI-driven documentation generation for the Folklore developer portal. It provides multi-model AI content generation, cross-package synthesis, and automated documentation freshness management through a federated API design.

## Architecture

### High-Level Design

```
┌─────────────────────────────────────────────┐
│           FolkloreContentService             │
├─────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Generate │  │  Edit    │  │ Validate │  │
│  │ Content  │  │  Content │  │ Content  │  │
│  │ Activity │  │  Activity│  │ Activity │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  │
│       │              │              │        │
│  ┌────▼──────────────▼──────────────▼────┐  │
│  │         Content Pipeline              │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  Multi-Model Provider Strategy  │  │  │
│  │  │  Claude → Sonnet → Haiku        │  │  │
│  │  └─────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  Validation Pipeline            │  │  │
│  │  │  Schema → Freshness → Quality   │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
├─────────────────────────────────────────────┤
│  External Dependencies:                      │
│  • VaultManager (encryption)                 │
│  • AuthGate (authorization)                  │
│  • MetricsLake (analytics)                   │
│  • PackageRegistry (ownership)               │
│  • SynthesisEngine (AI generation)           │
└─────────────────────────────────────────────┘
```

### Core Capabilities

1. **AI Content Generation** — Generates documentation from source code analysis using multi-model AI strategy with progressive access control
2. **Cross-Package Synthesis** — Produces workflow guides spanning multiple packages via dependency graph traversal
3. **Content Validation** — Validates generated content against Scout topic file schema, freshness requirements, and quality thresholds
4. **Federated Routing** — Routes requests to appropriate handlers based on content type (per-package vs. cross-cutting)
5. **Distributed Throttling** — Manages request rate across service instances using mesh-based peer discovery

### Service Architecture

FolkloreContentService follows a federated architecture pattern:

- **Content Generation Path**: Handles document generation, editing, and synthesis through AI providers
- **Validation Path**: Ensures generated content meets schema requirements and quality thresholds
- **Retrieval Path**: Serves content to agents via get_context mechanism with dependency traversal

### Key Design Decisions

1. **Multi-Model Strategy**: Progressive AI model selection (Claude Opus → Claude Sonnet → Claude Haiku) based on content complexity and user tier
2. **Feature-Gated Access**: All AI model access controlled through FeatureGateCheck with user-based and team-based overrides
3. **Federated Architecture**: Separate handlers for portal content vs. agent retrieval, unified through single service interface
4. **Content Safety**: All generated content validated through quality checks before publishing

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Language | Java 17 + Kotlin 2.x |
| Framework | Internal RPC framework with Guice DI |
| Build System | Gradle with internal plugins |
| Deployment | Container orchestration with rolling updates |
| AI Models | Claude Opus, Sonnet, Haiku via inference service |
| Storage | Object storage for generated content |
| Monitoring | Custom metrics with dashboard integration |

## Key Concepts

### Content Types

| Type | Description | Handler |
|------|-------------|---------|
| TopicFile | Per-package Scout documentation (6 standard files) | TopicFileGenerationHandler |
| Synthesis | Cross-package workflow guides (3+ package spans) | SynthesisGenerationHandler |
| ReleaseNote | Auto-generated release notes from version diffs | ReleaseNoteHandler |
| PackageIndex | Registry-driven package listing | IndexGenerationHandler |

### Content Lifecycle

```
Source Code Change → Pre-CR Hook (nudge) → CR Merge → Post-Merge Pipeline
    → FolkloreContentService.generateContent() → Validation → Certification Queue
    → Published (unverified) → Certifier Review (48h) → Verified
```

### Multi-Model Selection

The service uses a progressive model selection strategy:
- **Tier 1** (Claude Opus): Complex architecture docs, cross-package synthesis
- **Tier 2** (Claude Sonnet): Standard topic file generation, updates
- **Tier 3** (Claude Haiku): Simple formatting, metadata extraction, quick edits

Selection is automatic based on content complexity scoring and user feature access level.

## Dependencies

### Strong Dependencies

| Package | Purpose |
|---------|---------|
| `FolkloreContentServiceEncryption-1.0` | Team-owned encryption utilities |
| `ContentPipelineCommon-2.1` | Shared content models, handlers, utilities |

### Weak Dependencies

| Package | Purpose |
|---------|---------|
| `SynthesisEngineModel-1.0` | AI model service for document generation |
| `CoverageAnalyzerAgent-1.2` | Coverage monitoring and gap tracking |
| `FeatureGateCheck-3.0` | Feature access control framework |
| `ServiceMeshClientLib-2.0` | Service discovery and container metadata |
| `ThrottleResolverLib-1.5` | Distributed throttling peer discovery |
| `PortalContentGen-1.0` | Portal-specific content generation |
| `AuthGateService-2.3` | Authorization and policy enforcement |
| `AuthPolicyEngine-1.1` | Policy evaluation for access control |
| `ContentModelSDK-4.0` | RPC interface generation |
| `SecurityEventsSupport-1.8` | Security event reporting |

## Deployment

### Multi-Stage Deployment

```
Environments: Alpha → Beta → Gamma → Production
Regions: us-west-2, us-east-1, eu-west-1
```

### Deployment Architecture

- **Container Orchestration**: 4 instances per availability zone
- **Auto-Scaling**: CPU-based scaling (target 60%)
- **Health Checks**: Deep ping every 30s with dependency validation
- **Rolling Updates**: Zero-downtime deployments with canary validation

## Related Components

- **PortalRenderService**: Renders content for browser-based portal consumption
- **PackageRegistryPipeline**: Maintains ownership and metadata for all packages
- **FreshnessDaemon**: Scheduled validation of content staleness
- **CertificationService**: Manages human review workflow for generated content
