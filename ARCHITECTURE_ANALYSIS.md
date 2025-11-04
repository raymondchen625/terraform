# Terraform Architecture Analysis

## Project Overview

**Project**: Terraform CLI/Core
**Language**: Go
**Go Version**: 1.25
**License**: Business Source License 1.1
**Repository**: github.com/hashicorp/terraform

Terraform is an infrastructure as code tool that enables users to define and provision datacenter infrastructure using a declarative configuration language. This repository contains the core Terraform engine, CLI, and state management - providers are distributed separately via plugins.

## Core Architecture

### Architectural Pattern
Terraform uses a **plugin-based architecture** with a **directed acyclic graph (DAG)** execution model. The core engine orchestrates infrastructure changes while providers implement resource-specific logic through a gRPC-based plugin system.

### Key Architectural Components

#### 1. Graph Engine (`/internal/terraform/`)
- **Purpose**: Core orchestration logic
- **Responsibilities**:
  - Builds dependency graphs from configuration
  - Manages parallel execution of operations
  - Implements plan/apply workflow
  - Handles resource lifecycle management
- **Size**: Largest component with 234 files

#### 2. Configuration System (`/internal/configs/`)
- **Purpose**: Configuration parsing and validation
- **Responsibilities**:
  - Parses HCL (HashiCorp Configuration Language) files
  - Validates configuration structure
  - Handles module loading and composition
  - Schema validation
- **Size**: 68 files

#### 3. State Management (`/internal/states/`, `/internal/backend/`)
- **Purpose**: Infrastructure state tracking
- **Responsibilities**:
  - Tracks current state of managed infrastructure
  - Supports multiple storage backends (S3, Azure, GCS, Kubernetes, PostgreSQL, etc.)
  - Implements state locking and versioning
  - Handles state serialization/deserialization
- **Backends**: 9 separate remote state backend modules

#### 4. Provider Framework (`/internal/providers/`, `/internal/plugin/`, `/internal/tfplugin5/`, `/internal/tfplugin6/`)
- **Purpose**: Plugin system for providers
- **Responsibilities**:
  - gRPC-based provider communication using Protocol Buffers
  - Schema negotiation and validation
  - Resource CRUD operations (Create, Read, Update, Delete)
  - Provider lifecycle management
- **Protocol Versions**: Supports both plugin protocol v5 and v6

#### 5. Plan/Apply System (`/internal/plans/`)
- **Purpose**: Change execution planning
- **Responsibilities**:
  - Creates execution plans showing intended changes
  - Compares desired state (config) vs. current state
  - Applies changes with proper dependency ordering
  - Implements plan serialization (protobuf format)

#### 6. CLI Layer (`/internal/command/`)
- **Purpose**: User interface
- **Responsibilities**:
  - Implements all Terraform commands (plan, apply, init, destroy, etc.)
  - User interaction and output formatting
  - Command-line argument parsing
- **Size**: 141 files

#### 7. Language/Expression System (`/internal/lang/`)
- **Purpose**: Expression evaluation
- **Responsibilities**:
  - HCL expression evaluation
  - Function implementation (built-in Terraform functions)
  - Variable interpolation
  - Type system integration

#### 8. Addressing System (`/internal/addrs/`)
- **Purpose**: Resource addressing
- **Responsibilities**:
  - Uniform addressing for resources, modules, providers
  - Implements references like `aws_instance.web[0]`
  - Handles count and for_each indexing
- **Size**: 70 files

#### 9. Stacks (`/internal/stacks/`)
- **Purpose**: Stack-based orchestration
- **Responsibilities**:
  - Newer feature for managing multiple Terraform configurations together
  - Orchestrates deployments across multiple configurations
  - Uses protobuf for data serialization

#### 10. RPC API (`/internal/rpcapi/`)
- **Purpose**: Remote procedure call interface
- **Responsibilities**:
  - Provides RPC interface for external integrations
  - Handles dependencies, packages, setup, and stacks operations
  - Protocol Buffers-based communication

## Technology Stack

### Core Language & Runtime
- **Go 1.25**: Primary programming language
- **Go Modules**: Dependency management

### Key Frameworks & Libraries

#### Configuration & Parsing
- **github.com/hashicorp/hcl/v2** (v2.24.0): HashiCorp Configuration Language parser
- **github.com/zclconf/go-cty** (v1.16.3): Dynamic type system for Terraform values
- **github.com/zclconf/go-cty-yaml** (v1.1.0): YAML integration for cty types

#### Plugin System
- **github.com/hashicorp/go-plugin** (v1.7.0): Plugin system framework
- **google.golang.org/grpc** (v1.69.4): gRPC framework for provider communication
- **google.golang.org/protobuf** (v1.36.6): Protocol Buffers implementation

#### Networking & HTTP
- **github.com/hashicorp/go-cleanhttp** (v0.5.2): HTTP client utilities
- **github.com/hashicorp/go-retryablehttp** (v0.7.8): Retryable HTTP client
- **golang.org/x/net** (v0.44.0): Extended network packages
- **golang.org/x/oauth2** (v0.27.0): OAuth2 client

#### Logging & Observability
- **github.com/hashicorp/go-hclog** (v1.6.3): Structured logging
- **go.opentelemetry.io/otel** (v1.38.0): OpenTelemetry SDK
- **go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc** (v0.46.1): gRPC instrumentation
- **go.opentelemetry.io/contrib/exporters/autoexport** (v0.45.0): Telemetry exporters

#### CLI & Terminal
- **github.com/hashicorp/cli** (v1.1.7): CLI framework
- **github.com/mitchellh/colorstring** (v0.0.0-20190213212951): Colored output
- **github.com/posener/complete** (v1.2.3): Command-line completion
- **github.com/mattn/go-isatty** (v0.0.20): Terminal detection
- **github.com/chzyer/readline** (v0.0.0-20180603132655): Line editing

#### State Backend Dependencies

**AWS**
- github.com/aws/aws-sdk-go-v2 (v1.39.2): AWS SDK v2
- S3, DynamoDB, IAM, STS services

**Azure**
- github.com/hashicorp/go-azure-sdk (v0.20250131.1134653): Azure SDK
- github.com/hashicorp/go-azure-helpers (v0.72.0): Azure utilities

**Google Cloud**
- cloud.google.com/go/storage (v1.30.1): GCS client
- google.golang.org/api (v0.155.0): Google APIs

**Kubernetes**
- k8s.io/client-go (v0.33.0): Kubernetes client
- k8s.io/api (v0.33.0): Kubernetes API types

**Other Backends**
- github.com/hashicorp/consul/api (v1.32.1): Consul backend
- github.com/lib/pq (v1.10.3): PostgreSQL driver
- github.com/aliyun/aliyun-oss-go-sdk: Alibaba Cloud OSS
- github.com/tencentyun/cos-go-sdk-v5: Tencent Cloud COS
- github.com/oracle/oci-go-sdk/v65: Oracle Cloud Infrastructure

#### Cryptography & Security
- **golang.org/x/crypto** (v0.42.0): Cryptographic functions
- **github.com/ProtonMail/go-crypto** (v1.1.3): OpenPGP implementation
- **github.com/xanzy/ssh-agent** (v0.3.3): SSH agent client

#### Utilities
- **github.com/google/go-cmp** (v0.7.0): Deep comparison
- **github.com/google/uuid** (v1.6.0): UUID generation
- **github.com/hashicorp/go-version** (v1.7.0): Version parsing
- **github.com/mitchellh/go-wordwrap** (v1.0.1): Text wrapping
- **github.com/spf13/afero** (v1.9.5): Filesystem abstraction

#### Module & Provider Fetching
- **github.com/hashicorp/go-getter** (v1.8.2): URL-based file downloads
- **github.com/hashicorp/terraform-registry-address** (v0.4.0): Registry addressing
- **github.com/hashicorp/terraform-svchost** (v0.1.1): Service host discovery

#### Testing & Development
- **go.uber.org/mock** (v0.6.0): Mock generation
- **github.com/Netflix/go-expect** (v0.0.0-20220104043353): Testing interactive applications
- **github.com/go-test/deep** (v1.0.3): Deep equality testing

## Build Tools & Development Workflow

### Build System

#### Primary Build Tool: Go Toolchain
```bash
# Build and install
go install .

# Build to specific location
go build -o bin/terraform .
```

#### Makefile Targets
The Makefile provides several targets for code quality and generation:

**Code Generation**
- `make generate`: Runs `go generate ./...` to build dynamically generated source files
- `make protobuf`: Generates protobuf stubs using Protocol Buffers compiler

**Code Quality Checks**
- `make fmtcheck`: Validates Go code formatting (gofmt)
- `make importscheck`: Validates Go import organization (goimports)
- `make vetcheck`: Runs `go vet` for static analysis
- `make staticcheck`: Runs staticcheck for advanced static analysis
- `make exhaustive`: Checks exhaustive switch statements
- `make copyright`: Validates copyright headers
- `make copyrightfix`: Automatically fixes copyright headers

**Dependency Management**
- `make syncdeps`: Synchronizes dependencies across internal modules

### Code Quality Tools

#### Static Analysis
- **honnef.co/go/tools** (staticcheck v0.6.0): Advanced static analysis
- **github.com/nishanths/exhaustive** (v0.12.0): Exhaustive switch checking
- Configuration: `staticcheck.conf` - enables all checks except style (ST*) and deprecation warnings (SA1019)

#### Code Generation Tools (via `tool` directive in go.mod)
- **go.uber.org/mock/mockgen**: Mock generation for testing
- **golang.org/x/tools/cmd/stringer**: String method generation for enums
- **golang.org/x/tools/cmd/goimports**: Import organization
- **golang.org/x/tools/cmd/cover**: Test coverage analysis
- **google.golang.org/grpc/cmd/protoc-gen-go-grpc**: gRPC code generation
- **github.com/hashicorp/copywrite**: Copyright header management

### Testing Framework

#### Test Infrastructure
- **Standard Go Testing**: Uses built-in `testing` package
- **Test Organization**:
  - Unit tests: Self-contained, no external dependencies
  - Acceptance tests: Require `TF_ACC=1` environment variable
  - End-to-end tests: Located in `/internal/e2e/`, test compiled binary

#### Running Tests
```bash
# All unit tests
go test ./...

# Specific package
go test ./internal/terraform

# Specific test function
go test ./internal/terraform -run TestContext2Apply_basic

# Verbose output
go test -v ./internal/terraform

# Race condition detection
go test -race ./...

# Test coverage
go test -cover ./...
```

#### Test Utilities
- Test fixtures stored in `testdata/` directories
- Helper functions: `testModule()`, `testProvider()`, `testContext2()`
- Mock providers via testing infrastructure
- `testing/` directory for shared test utilities

### Changelog Management

**Tool**: Changie (changelot.io)
- **Configuration**: `.changie.yaml`
- **Command**: `npx changie new`
- **Storage**: `.changes/v1.XX/` directories

**Change Types**:
1. NEW FEATURES: New functionality
2. ENHANCEMENTS: Improvements to existing features
3. BUG FIXES: User-facing bug fixes
4. NOTES: Internal changes with potential edge cases
5. UPGRADE NOTES: Changes requiring user action
6. BREAKING CHANGES: Breaking configuration changes

### Version Management
- **Version File**: `version/VERSION`
- **Go File**: `version.go`
- **Script**: `scripts/version-bump.sh`

### Continuous Integration Scripts

Located in `/scripts/`:
- `build.sh`: Build automation
- `gofmtcheck.sh`: Format validation
- `goimportscheck.sh`: Import validation
- `staticcheck.sh`: Static analysis
- `exhaustive.sh`: Exhaustive checking
- `copyright.sh`: Copyright validation
- `changelog.sh`: Changelog generation
- `changelog-links.sh`: Changelog link validation
- `syncdeps.sh`: Dependency synchronization

## Module Structure

### Internal Module System
Terraform uses Go modules with several internal sub-modules for different backend implementations:

```
github.com/hashicorp/terraform/internal/backend/remote-state/azure
github.com/hashicorp/terraform/internal/backend/remote-state/consul
github.com/hashicorp/terraform/internal/backend/remote-state/cos
github.com/hashicorp/terraform/internal/backend/remote-state/gcs
github.com/hashicorp/terraform/internal/backend/remote-state/kubernetes
github.com/hashicorp/terraform/internal/backend/remote-state/oci
github.com/hashicorp/terraform/internal/backend/remote-state/oss
github.com/hashicorp/terraform/internal/backend/remote-state/pg
github.com/hashicorp/terraform/internal/backend/remote-state/s3
github.com/hashicorp/terraform/internal/legacy
```

These are managed via `replace` directives in `go.mod` and are never published separately. The `make syncdeps` command ensures all sub-modules stay synchronized.

## Protocol Buffers Architecture

### Protobuf Files
Terraform uses Protocol Buffers for:
1. **Provider Communication**: tfplugin5/tfplugin6 protocols
2. **Plan Serialization**: `internal/plans/planproto/planfile.proto`
3. **Stack Data**: `internal/stacks/tfstackdata1/tfstackdata1.proto`
4. **RPC API**: `internal/rpcapi/terraform1/` (stacks, setup, dependencies, packages)
5. **Cloud Plugin**: `internal/cloudplugin/cloudproto1/cloudproto1.proto`
6. **Stacks Plugin**: `internal/stacksplugin/stacksproto1/stacksproto1.proto`

### Code Generation
```bash
make protobuf  # Generates Go code from .proto files
```

## Security & Compliance

### Security Tools
- **Cryptography**: golang.org/x/crypto for secure operations
- **SSH**: github.com/xanzy/ssh-agent for SSH key management
- **TLS**: Standard Go crypto/tls with extensions

### License Compliance
- **Acceptable Licenses**: MPL v2, MIT, BSD
- **Prohibited**: Proprietary or copyleft licenses
- **Copyright**: Managed via `github.com/hashicorp/copywrite`

## Performance & Observability

### Telemetry
- **OpenTelemetry Integration**: Full OTLP support
- **Trace Instrumentation**: gRPC and HTTP instrumentation
- **Exporters**: Auto-export support for various backends
- **File**: `telemetry.go` contains telemetry setup

### Performance Features
- **Parallel Execution**: DAG-based parallel resource operations
- **Provider Caching**: Local provider cache to avoid repeated downloads
- **Incremental State**: Only compares changed resources
- **Remote State Locking**: Prevents concurrent modifications

## Dependency Graph Visualization

```
┌─────────────────────────┐
│   Terraform CLI         │
│   (internal/command)    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   Core Graph Engine     │
│   (internal/terraform)  │
└───────────┬─────────────┘
            │
    ┌───────┼────────┬──────────┬─────────┐
    ▼       ▼        ▼          ▼         ▼
┌─────┐ ┌──────┐ ┌──────┐  ┌────────┐ ┌─────────┐
│Confi│ │Plans │ │State │  │Provider│ │Language │
│gs   │ │      │ │      │  │Plugin  │ │  (HCL)  │
└─────┘ └──────┘ └──────┘  └────────┘ └─────────┘
                                │
                        ┌───────┴────────┐
                        ▼                ▼
                   ┌────────┐      ┌────────┐
                   │Provider│      │Provider│
                   │   A    │      │   B    │
                   │(gRPC)  │      │(gRPC)  │
                   └────────┘      └────────┘
```

## Notable Design Patterns

1. **Plugin Architecture**: Extensibility through external providers
2. **DAG Execution**: Parallel execution with dependency resolution
3. **State Machine**: Resource lifecycle management (Create, Read, Update, Delete)
4. **Declarative Model**: Configuration defines desired state, not steps
5. **Immutable Infrastructure**: Changes create new resources rather than modifying existing
6. **Provider Protocol Versioning**: Supports multiple protocol versions simultaneously
7. **Backend Abstraction**: Multiple storage backends with consistent interface
8. **Module Composition**: Hierarchical configuration organization

## Key Technical Decisions

1. **Go Language**: Performance, concurrency, cross-platform compilation
2. **HCL Configuration**: Human-readable, machine-friendly configuration
3. **gRPC Plugins**: Language-agnostic provider development
4. **State Locking**: Prevents race conditions in team environments
5. **Plan-Apply Workflow**: Preview changes before execution
6. **Registry-based Provider Distribution**: Decentralized provider ecosystem

## Development Considerations

### Code Organization Principles
- **Internal Packages**: All core code in `/internal/` (not importable externally)
- **Domain-Driven Structure**: Packages organized by domain (configs, states, providers)
- **Protocol Separation**: Clear separation between protocol versions
- **Backend Modularity**: Each backend is a separate Go module

### Testing Philosophy
- Self-contained unit tests
- Acceptance tests with external dependencies gated by environment variable
- Extensive use of mocks and test fixtures
- E2E tests on compiled binary

### Contribution Requirements
- All tests must pass
- Code quality checks must pass (fmt, imports, vet, staticcheck)
- Changelog entry required for user-facing changes
- Separate structural changes from behavioral changes
- Small, focused commits

## Future Directions

### Stacks Feature
The `/internal/stacks/` component represents a newer architectural direction for managing multiple Terraform configurations together, with its own:
- Protobuf-based data format
- RPC API for orchestration
- Plugin system for stack-aware providers

### Ephemeral Resources
Recent additions suggest support for ephemeral resources (temporary resources that don't persist in state).

### Enhanced Observability
Growing OpenTelemetry integration suggests focus on observability and debugging.

## Summary

Terraform's architecture is built on:
- **Modularity**: Plugin-based provider system with clear separation of concerns
- **Scalability**: DAG-based parallel execution
- **Reliability**: State management with locking and versioning
- **Extensibility**: Well-defined protocols and interfaces
- **Maintainability**: Strong testing, code quality tools, and documentation

The codebase demonstrates mature software engineering practices with comprehensive testing, static analysis, clear architectural boundaries, and thoughtful dependency management.
