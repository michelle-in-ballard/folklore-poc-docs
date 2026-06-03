---
name: FolkloreContentService
dependencies:
  FolkloreContentServiceModel:
    classification: strong
    connects_to: [how-to-build]
    usage: "Team-owned model definition package for FolkloreContentService. Used as build-time dependency for generating data models and service interfaces. Referenced in Config file but not directly imported in source code, indicating it's used for code generation or build-time model validation."
  CopyConfigurationPlugin:
    classification: weak
    connects_to: [how-to-build]
    usage: "Internal Gradle plugin for copying and managing configuration files during build process. Essential for proper deployment configuration setup across different environments (Beta, Gamma, Prod) as mentioned in service architecture."
  JsonProcessingLib:
    classification: weak
    connects_to: [how-to-build]
    usage: "Internal JSON processing library (org.json package). Extensively used for MetricsLake trace publishing, content type trace building, and JSON data manipulation. Critical for analytics and monitoring infrastructure, particularly in ContentTraceBuilder and MetricsLake integration handlers."
  InfraConfigBuilder:
    classification: weak
    connects_to: [how-to-build]
    usage: "Internal infrastructure configuration builder for generating runtime configurations. Essential for FolkloreContentService's container-based deployment and orchestration integration as mentioned in service architecture."
  InfraConfigGradleBuildLogic:
    classification: weak
    connects_to: [how-to-build]
    usage: "Internal Gradle build logic for integrating infrastructure configuration builder into the build process. Works with InfraConfigBuilder to ensure proper configuration generation for FolkloreContentService deployment."
---

# Build Guide - FolkloreContentService

## Build System

FolkloreContentService uses **Internal Gradle** as its primary build system with multi-language support:

```gradle
plugins {
    java
    kotlin("jvm")
    id("internal-gradle")
    id("internal-gradle-java-presets")
    id("internal-gradle-kotlin-presets")
    id("java-format")
    id("com.github.spotbugs")
    id("io.gitlab.arturbosch.detekt")
    id("org.jlleitschuh.gradle.ktlint")
}
```

### Key Build Components
- **Internal Gradle 8.x** - Internal build system
- **Multi-language presets** - Java 17 + Kotlin 2.x support
- **Code quality tools** - Checkstyle, SpotBugs, Detekt, KtLint
- **Configuration management** - CopyConfigurationPlugin for environment-specific configs
- **Infrastructure generation** - InfraConfigBuilder for deployment configurations

## Prerequisites

### Software Requirements
- **Java Runtime**: OpenJDK 17 or higher
- **Kotlin**: Kotlin 2.x (managed by Gradle)
- **Build CLI**: Latest version for workspace management
- **Docker**: For containerized development and testing
- **Git**: For source control operations

### Environment Setup
```bash
# Create workspace
ws create --root ~/workplace/FolkloreContentService
cd ~/workplace/FolkloreContentService

# Sync package
ws use -p FolkloreContentService

# Verify workspace
ws show
```

## Build Commands

### Standard Build
```bash
# Full build (compile + test + quality checks)
build

# Compile only (skip tests)
build compile

# Build with specific Gradle tasks
build gradle :compileJava :compileKotlin
```

### Incremental Build
```bash
# Fast rebuild (cached artifacts)
build --incremental

# Clean build (full recompile)
build clean build
```

### Docker Build
```bash
# Build container image
build docker

# Build and run locally
build docker-run

# Build with custom tag
build docker --tag folklore-content:dev
```

## Build Configuration

### Config File Structure
```
FolkloreContentService/
├── Config                    # Package metadata
├── build.gradle.kts         # Build logic
├── settings.gradle.kts      # Project settings
├── src/
│   ├── main/
│   │   ├── java/           # Java source
│   │   ├── kotlin/         # Kotlin source
│   │   └── resources/      # Application configs
│   └── test/
│       ├── java/           # Java tests
│       └── kotlin/         # Kotlin tests
├── configuration/
│   ├── beta/               # Beta environment configs
│   ├── gamma/              # Gamma environment configs
│   └── prod/               # Production configs
└── docker/
    └── Dockerfile.template  # Container definition
```

### Version Management
```properties
# Config file
build-system = internal-gradle
build-tools = InternalGradle-8.x, JDK17, DaggerBuildTool, CopyConfigurationPlugin

# Dependencies section
dependencies = ContentPipelineCommon, FolkloreContentServiceModel, ...
test-dependencies = JUnit5, Mockito-inline, AssertJ
runtime-dependencies = JDK17, Log4j2Core, ServiceMeshAgent
```

## Build Artifacts

### Primary Artifacts
| Artifact | Type | Purpose |
|----------|------|---------|
| `FolkloreContentService-1.0.jar` | JAR | Main application |
| `FolkloreContentService-1.0-config.tar.gz` | TAR | Environment configs |
| `FolkloreContentService-1.0-docker.tar.gz` | TAR | Container image |

### Build Outputs
```
build/
├── classes/            # Compiled bytecode
├── libs/              # JAR artifacts
├── reports/           # Test and quality reports
│   ├── tests/         # HTML test results
│   ├── spotbugs/      # Static analysis
│   └── detekt/        # Kotlin analysis
└── docker/            # Container artifacts
```

## Common Build Issues

### Dependency Resolution Failures
```bash
# Error: Could not resolve dependency ContentPipelineCommon
# Fix: Ensure version set is current
ws sync --version-set FolkloreContentService/development

# Error: Version conflict between transitive dependencies
# Fix: Add explicit resolution
build gradle dependencies --scan
```

### Kotlin Compilation Errors
```bash
# Error: Unresolved reference in Kotlin source
# Fix: Ensure Kotlin plugin matches Java version
# Check build.gradle.kts:
kotlin {
    jvmToolchain(17)
}
```

### Docker Build Failures
```bash
# Error: Base image not found
# Fix: Authenticate with container registry
docker-login

# Error: Template substitution failed
# Fix: Verify InfraConfigBuilder version matches template
build gradle :verifyDockerTemplate
```

## CI/CD Integration

### Pipeline Configuration
```yaml
stages:
  - name: Build
    actions:
      - build
      - build test
      - build docker
  - name: Quality
    actions:
      - build gradle spotbugsMain
      - build gradle detekt
      - build gradle ktlintCheck
  - name: Package
    actions:
      - build release
```

### Quality Gates
| Gate | Threshold | Action on Failure |
|------|-----------|-------------------|
| Unit Test Coverage | ≥80% | Block promotion |
| SpotBugs | 0 high/critical | Block promotion |
| Detekt | 0 errors | Block promotion |
| KtLint | 0 violations | Block promotion |

## Performance Optimization

### Build Speed Tips
1. **Use incremental builds** — `build --incremental` skips unchanged modules
2. **Parallel compilation** — Already enabled via `org.gradle.parallel=true`
3. **Daemon mode** — Gradle daemon reuses JVM across builds (default on)
4. **Focused testing** — Run specific test classes during development: `build test -Dtest.single=ContentHandlerTest`

### Resource Requirements
| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 4 GB | 8 GB |
| CPU | 2 cores | 4 cores |
| Disk | 5 GB | 10 GB |
| Network | Required for dependency resolution | — |
