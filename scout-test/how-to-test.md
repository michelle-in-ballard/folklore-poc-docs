---
name: FolkloreContentService
dependencies:
  FolkloreContentServiceTests:
    classification: strong
    connects_to: [how-to-test]
    usage: "Integration test package with test runner definitions and TestNG suites. Required for personal stack integration testing. Contains test templates, test configurations, and end-to-end test cases."
  AssertJ:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "Assertion library listed in dependencies. Note: tests in this package primarily use Hamcrest matchers (org.hamcrest.MatcherAssert.assertThat) rather than AssertJ for test assertions."
  ExperimentAllocationProvider:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "ExperimentAllocationProvider is part of the internal A/B testing framework, providing experiment allocation capabilities. Integrated into FolkloreContentService through ExperimentModule to create ExperimentAccessor, which is essential for the service's extensive feature gating and experimentation strategy mentioned throughout the architectural analysis."
  ExperimentRuntimeSupport:
    classification: weak
    connects_to: [how-to-use-me]
    usage: "ExperimentRuntimeSupport provides runtime support for the experimentation framework in cloud environments. Enables A/B testing infrastructure mentioned in implementation patterns but operates as a supporting runtime component."
---
# Testing Guide - FolkloreContentService

## Testing Strategy
FolkloreContentService employs a comprehensive testing strategy with multiple layers:
- **Unit Testing**: JUnit 5 with comprehensive mocking using Mockito/MockK
- **Integration Testing**: Test platform with end-to-end scenario validation
- **E2E Testing**: Portal integration with real content generation scenarios
- **Load Testing**: Dedicated test accounts for performance validation
- **Smoke Testing**: Container-based validation with base image requirements

### Testing Frameworks
- **Java Testing**: JUnit 5 with Mockito for mocking
- **Kotlin Testing**: JUnit 5 with MockK for Kotlin-specific mocking
- **Assertions**: Hamcrest matchers (org.hamcrest.MatcherAssert.assertThat) for test assertions
- **A/B Testing**: ExperimentAllocationProvider for experiment allocation testing

## Running Tests

### Local Testing
```bash
# Run all tests
build test

# Run specific test class
build test -Dtest.single=ContentGenerationHandlerTest

# Run tests with coverage
build test jacocoTestReport
```

### Container Testing
```bash
# Smoke tests (requires base image)
rde workflow run validate

# Manual health check
curl 127.0.0.1:8080/ping

# Container inspection
docker exec -it FolkloreContentService bash
```

### Build Output Debugging
When debugging test failures, save the build output once and search it repeatedly instead of rebuilding for each query:
```bash
# Save build output to a file
build test 2>&1 | tee build-output.log

# Search for specific failures
grep -A 10 "FAILED" build-output.log

# Find assertion messages
grep -B 2 "AssertionError" build-output.log
```

## Unit Testing

### Test Structure
```
src/test/
├── java/
│   └── com/internal/folklore/
│       ├── handlers/
│       │   ├── ContentGenerationHandlerTest.java
│       │   ├── SynthesisHandlerTest.java
│       │   └── ValidationHandlerTest.java
│       ├── providers/
│       │   ├── ClaudeProviderTest.java
│       │   └── SonnetProviderTest.java
│       └── validators/
│           ├── SchemaValidatorTest.java
│           └── FreshnessValidatorTest.java
└── kotlin/
    └── com/internal/folklore/
        ├── ContentEditingHandlerTest.kt
        └── SynthesisEngineIntegrationTest.kt
```

### Mocking Patterns

#### Java (Mockito)
```java
@ExtendWith(MockitoExtension.class)
class ContentGenerationHandlerTest {
    @Mock private ContentPipeline contentPipeline;
    @Mock private FeatureGateCheck featureGateCheck;
    @InjectMocks private ContentGenerationHandler handler;

    @Test
    void generateContent_validRequest_returnsTopicFile() {
        when(featureGateCheck.hasAccess("content.generate", userId))
            .thenReturn(true);
        when(contentPipeline.generate(any()))
            .thenReturn(GenerationResult.success(topicFile));

        GenerateContentResponse response = handler.generateContent(request);

        assertThat(response.getStatus(), is("SUCCESS"));
        verify(contentPipeline).generate(argThat(req ->
            req.getPackageName().equals("VegaScriptCore")));
    }
}
```

#### Kotlin (MockK)
```kotlin
class ContentEditingHandlerTest {
    private val synthesisEngine = mockk<SynthesisEngine>()
    private val handler = ContentEditingHandler(synthesisEngine)

    @Test
    fun `edit content with valid request returns updated document`() {
        every { synthesisEngine.edit(any()) } returns EditResult.success(updatedDoc)

        val result = handler.editContent(editRequest)

        result.status shouldBe "SUCCESS"
        verify { synthesisEngine.edit(match { it.documentId == "doc-123" }) }
    }
}
```

## Integration Testing

### Personal Stack Testing
```bash
# Provision personal stack
rde stack provision

# Run integration tests against personal stack
build integration-test --stack personal

# Verify content generation end-to-end
curl -X POST http://localhost:8080/generate \
  -H "Content-Type: application/json" \
  -d '{"packageName": "TestPackage", "contentType": "TOPIC_FILE"}'
```

### Test Platform Integration
```yaml
# Test definition
testSuite: FolkloreContentServiceE2E
environment: beta
testCases:
  - name: GenerateTopicFile
    endpoint: /generate
    method: POST
    body:
      packageName: "TestPackage"
      contentType: "TOPIC_FILE"
      modelTier: "SONNET"
    assertions:
      - status: 200
      - body.status: "SUCCESS"
      - body.content: contains("---\nname: TestPackage")
```

## Performance Testing

### Load Test Configuration
```yaml
loadTest:
  name: FolkloreContentService-LoadTest
  scenarios:
    - name: SteadyState
      rps: 50
      duration: 30m
      endpoint: /generate
    - name: PeakLoad
      rps: 200
      duration: 5m
      endpoint: /generate
    - name: SynthesisHeavy
      rps: 10
      duration: 15m
      endpoint: /synthesize
  thresholds:
    p95_latency: 5000ms
    error_rate: 0.1%
    timeout_rate: 0.5%
```

### Performance Baselines
| Operation | P50 | P95 | P99 |
|-----------|-----|-----|-----|
| Topic File Generation | 3.2s | 4.8s | 7.1s |
| Content Edit | 1.5s | 2.8s | 4.2s |
| Synthesis (3 packages) | 8.1s | 12.4s | 18.7s |
| Schema Validation | 45ms | 120ms | 250ms |

## Test Data Management

### Fixtures
```java
public class TestFixtures {
    public static GenerateContentRequest validGenerationRequest() {
        return GenerateContentRequest.builder()
            .packageName("TestPackage")
            .contentType(ContentType.TOPIC_FILE)
            .topicName("what-is-it")
            .modelTier(ModelTier.SONNET)
            .userId("test-user-001")
            .build();
    }

    public static String validTopicFileContent() {
        return """
            ---
            name: TestPackage
            dependencies:
              DepA:
                classification: strong
                connects_to: [what-is-it]
                usage: "Test dependency A"
            ---
            # TestPackage
            ## Overview
            Test package documentation.
            """;
    }
}
```

### Test Isolation
- Each integration test uses a dedicated test namespace
- Content generated during tests is tagged with `test:true` metadata
- Cleanup runs automatically after test suite completion
- No test data persists in shared environments

## Coverage Requirements

| Module | Minimum Coverage | Current |
|--------|:---:|:---:|
| Handlers | 85% | 91% |
| Providers | 80% | 87% |
| Validators | 90% | 94% |
| Models | 70% | 78% |
| Overall | 80% | 88% |

## Common Test Failures

### Flaky Tests
- **Timeout-sensitive tests**: Increase timeout for AI model calls in test config
- **Order-dependent tests**: Ensure each test creates its own fixtures
- **Environment-dependent tests**: Use test profiles with mocked external dependencies

### Debugging Tips
```bash
# Run single test with debug logging
build test -Dtest.single=ContentHandlerTest -Dlog.level=DEBUG

# Generate coverage report
build test jacocoTestReport
open build/reports/jacoco/test/html/index.html

# Run tests in isolation (no parallel)
build test -Dtest.parallel=false
```
