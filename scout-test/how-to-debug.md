---
name: FolkloreContentService
dependencies: {}
---


# FolkloreContentService debugging

## System Overview

FolkloreContentService is a Java/Kotlin-based AI content generation service running on container orchestration with the following key characteristics:

- **Runtime**: OpenJDK 17 with Kotlin 2.x support
- **Framework**: Internal RPC framework with Guice dependency injection
- **Deployment**: Container orchestration with rolling deployment system
- **AI Models**: Progressive access to Claude Opus, Sonnet, Haiku via inference service
- **Architecture**: Federated API design with content-type-based routing
- **Traffic Management**: Distributed throttling policies with mesh-based rate limiting

## Configuration & Deployment

### Multi-Stage Deployment
```
Environments: Alpha → Beta → Gamma → Prod (Canary + Fleet)
Regions: us-west-2, us-east-1, eu-west-1
Accounts:
- Beta: 381491823554
- Gamma: 654654474247
- Prod: 058264312636
```

### Configuration Management
- **Environment-Specific Settings**: Hierarchical configuration with inheritance
- **AI Model Configuration**: Regional model deployment (Claude Sonnet, Haiku)
- **Traffic Shaping**: Throttling policies for request rate limiting
- **Security Configuration**: VaultManager encryption, AuthGate authorization

### Deployment Procedures
- **Normal Operations**: Horizontal/vertical scaling via container orchestration and CDK
- **Emergency Procedures**: Manual task count adjustment, emergency rollback
- **Configuration Validation**: Pre-deployment config validation

## Environment Setup for Debugging

### Local Development
```bash
# Development environment configuration
export DEV_DESKTOP='your-clouddesk.internal.example.com'
rde stack provision
rde workflow run

# Health check
curl 127.0.0.1:8080/ping

# Container debugging
docker exec -it FolkloreContentService bash
```

### Debug Configuration
```yaml
# Personal Stack Configuration
devAccount:
  type: Conduit
  roleArn: <%.MyDevAccountIAMRoleArn%>

applications:
  FolkloreContentService:
    type: container
    environment:
      REGION: "us-west-2"
      STAGE: "alpha"
      IS_LOCAL_DEV: "true"
```

## Monitoring & Health Checks

### Health Endpoints
- **Deep Ping**: `/deep_ping` endpoint for comprehensive service health
- **Basic Health**: `/ping` for basic connectivity checks
- **Metrics Collection**: AutoMetrics with operation-level tracking

### Monitoring Tools
- **CloudWatch Logs**: Application logs per environment (Beta, Gamma, Prod)
- **Log Analysis**: Service-specific log analysis tool for structured queries
- **Trace Analysis**: LLM trace analysis for request lifecycle
- **Dashboard Access**: CloudWatch dashboards per environment

### Key Metrics
- **Operation-Level Metrics**: Success/failure rates per API operation
- **Content-Type Metrics**: Separate tracking for TopicFile vs Synthesis vs ReleaseNote
- **Model-Specific Metrics**: Performance tracking per AI model tier
- **Latency Tracking**: End-to-end request timing with breakdown by phase

## Common Issues & Troubleshooting

### Content Quality Issues
- **400 BadRequestException**: Content failed schema validation
- **Fallback Strategy**: Service attempts generation with simpler prompt on quality failure
- **Double Validation**: Both AI-generated and post-processed content are validated

### Feature Access Issues
- **Progressive Model Access**: Check Opus → Sonnet → Haiku access chain
- **User-Based Gating**: Verify individual user feature access via FeatureGateCheck
- **Team Rules**: Different access rules per team and content type

### Performance Issues
- **Throttling**: 429 ThrottlingException indicates rate limiting
- **Circuit Breaker**: AI model calls are gated by CircuitBreakerExecutor with AIMD feedback; on throttle rejection, automatically falls back to simpler model
- **Timeout Issues**: 5-second timeouts on inference service calls
- **Capacity Issues**: 503 ServiceUnavailableException for capacity problems

### Content Routing Issues
- **Content Type Detection**: Verify content type classification
- **Handler Selection**: Check TopicFile vs Synthesis vs ReleaseNote handler routing
- **Feature Gate Override**: Certain users can force specific model tiers

### Dependency Graph Traversal Issues
The dependency graph traverser walks package dependencies for synthesis generation. Two failure modes:
- **Circular dependency detected**: The traverser maintains a visited set but cycles can still cause excessive depth. Check `dependencyDepth` parameter (max 5). Surfaces as `TimeoutException: Graph traversal exceeded depth limit` in application logs.
- **Missing package in registry**: `DependencyValidator.validate()` expects all referenced packages to exist in `package_registry.json`. When a frontmatter dependency references a package not in registry, it surfaces as `ValidationException: Unknown dependency`. The registry refreshes weekly — newly created packages may not appear until next refresh cycle.

## Logging & Tracing

### Log4j2 Multi-Appender Setup
```xml
<Appenders>
  <Socket name="ApplicationTcp" host="localhost" port="5170"/>
  <Socket name="RequestTcp" host="localhost" port="5171"/>
  <Socket name="MetricsTcp" host="localhost" port="5172"/>
</Appenders>
```

### Structured Logging
- **Application Logs**: JSON format for CloudWatch Log Insights
- **Request Logs**: Separate stream with request/response correlation
- **Metrics Logs**: EMF format for CloudWatch metrics integration
- **Failover**: STDOUT fallback for socket connection failures

### Log Levels
- **Production**: INFO level with WARN for external libraries
- **Development**: DEBUG level with service explorer enabled
- **Wire Logging**: Disabled for performance (WIRE level OFF)

### Distributed Tracing
- **MDC Context Propagation**: Maintains context across async operations
- **MetricsLake Integration**: Comprehensive trace publishing
- **External Service Tracing**: Tracks calls to inference service and dependencies

## Performance Considerations

### Latency Breakdown (P95)
| Phase | Duration |
|-------|----------|
| Auth validation | 45ms |
| Feature gate check | 12ms |
| Model selection | 5ms |
| AI inference (Sonnet) | 3.8s |
| AI inference (Haiku) | 1.2s |
| Content validation | 120ms |
| Persistence | 85ms |
| **Total (Sonnet path)** | **~4.1s** |

### Scaling Limits
- **Max concurrent requests**: 200 per instance (4 instances = 800 total)
- **AI model throughput**: 50 TPS per model endpoint
- **Graph traversal**: Max depth 5, max nodes 100 per traversal
- **Content size**: Max 100KB per generated document

### Performance Tuning
- Increase instance count for higher throughput
- Use Haiku for bulk operations (3x faster than Sonnet)
- Pre-warm dependency graph cache at startup
- Batch validation for multi-file generation

## Error Handling Strategy

### Retry Mechanisms
- **AI model calls**: 3 retries with exponential backoff (1s, 2s, 4s)
- **Registry lookups**: 2 retries with 500ms delay
- **Auth checks**: No retry (fail fast)

### Fallback Behaviors
- **Model unavailable**: Fall back to next tier (Opus → Sonnet → Haiku)
- **Validation failure**: Retry with simplified prompt (max 2 attempts)
- **Registry stale**: Use cached version (up to 1 hour old)
- **Total failure**: Return partial result with error metadata

### Error Codes
| Code | Meaning | Recovery |
|------|---------|----------|
| GENERATION_FAILED | AI model returned invalid content | Retry with simpler prompt |
| VALIDATION_FAILED | Content failed schema check | Fix frontmatter and resubmit |
| MODEL_UNAVAILABLE | All model tiers exhausted | Wait and retry in 30s |
| PACKAGE_NOT_FOUND | Target package not in registry | Verify package name |
| AUTH_DENIED | User lacks package access | Check team permissions |

## Appendix
[Any other relevant information]
