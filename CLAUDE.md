# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the Terraform CLI/Core repository. Terraform is written in Go and uses a plugin-based architecture where providers are separate from core. This repository contains only the core engine, CLI, and state management - providers live in separate repositories and are distributed via the Terraform Registry.

## Core Architecture

### Key Components

**Graph Engine** (`/internal/terraform/`)
- Builds dependency graphs from configuration
- Manages parallel execution of operations
- Core orchestration logic for plan/apply workflow

**Configuration System** (`/internal/configs/`)
- Parses HCL configuration files
- Validates configuration structure
- Handles module loading and composition

**State Management** (`/internal/states/`, `/internal/backend/`)
- Tracks infrastructure state
- Supports multiple storage backends (S3, Azure, GCS, local, etc.)
- State locking and versioning

**Provider Framework** (`/internal/providers/`, `/internal/plugin/`)
- gRPC-based plugin system for providers
- Schema validation and resource lifecycle management
- Protocol Buffers for provider communication

**Plan/Apply System** (`/internal/plans/`)
- Creates execution plans showing intended changes
- Compares desired state (config) vs. current state (state file)
- Applies changes with proper ordering

**CLI Layer** (`/internal/command/`)
- Implements all Terraform commands (plan, apply, init, destroy, etc.)
- User interaction and output formatting

**Addressing System** (`/internal/addrs/`)
- Uniform addressing for resources, modules, providers
- Resource references like `aws_instance.web[0]`

**Language/Expression Evaluation** (`/internal/lang/`)
- HCL expression evaluation
- Function implementation
- Variable interpolation

**Stacks** (`/internal/stacks/`)
- Newer stack-based orchestration feature
- Allows managing multiple Terraform configurations together

## Development Workflow

### Building

```bash
# Build and install to GOPATH/bin
go install .

# Build to specific location
go build -o bin/terraform .
```

The `.go-version` file specifies the production Go version. The `go.mod` specifies the minimum compatible version.

### Running Tests

```bash
# Run all unit tests
go test ./...

# Run tests for specific package
go test ./internal/terraform
go test ./internal/command/...

# Run specific test function
go test ./internal/terraform -run TestContext2Apply_basic

# Run with verbose output
go test -v ./internal/terraform -run TestContext2Apply

# Run acceptance tests (interact with external services)
TF_ACC=1 go test ./internal/initwd
```

**Important test patterns:**
- Tests use standard Go `testing` package
- Test fixtures in `testdata/` directories within packages
- Helper functions: `testModule()`, `testProvider()`, `testContext2()`
- Mock providers via `testing_provider.MockProvider`
- Acceptance tests require `TF_ACC=1` environment variable

### Code Quality Checks

```bash
# Run all checks before committing
make fmtcheck      # Check Go formatting
make importscheck  # Check Go imports
make vetcheck      # Run go vet
make staticcheck   # Run static analysis
make exhaustive    # Check exhaustive switches
make copyright     # Check copyright headers
```

### Code Generation

```bash
# Generate Go code (except protobuf)
go generate ./...

# Generate protobuf stubs (requires protoc installed)
make protobuf
```

## Testing Philosophy

- **Unit tests should be self-contained**: Use mocks, avoid external dependencies
- **Acceptance tests** (with `TF_ACC=1`) test external service interactions
- **End-to-end tests** (`/internal/e2e/`) test the full compiled binary
- Tests should run offline whenever possible
- Use `testdata/` directories for test fixtures

## Contribution Guidelines

### Before Making Changes

1. Discuss proposed changes in a GitHub issue first
2. Get feedback from maintainers on approach
3. Run existing tests to ensure they pass: `go test ./...`
4. Make small, focused changes rather than large refactors

### Areas of Special Concern

**State Storage Backends** - Not accepting new backends. Existing backends have assigned maintainers (see CODEOWNERS).

**Provisioners** - Tool-specific provisioners are deprecated. Generic provisioners (file, local-exec, remote-exec) are low priority.

**Core Language/Graph Changes** - Require extensive discussion and design review before implementation.

### Changelog Requirements

User-facing changes require changelog entries using `changie`:

```bash
# Create a new changelog entry interactively
npx changie new
```

Place change files in `.changes/v1.XX/` matching the version in `version/VERSION`.

Change types:
- **NEW FEATURES** - New functionality (e.g., ephemeral resources)
- **ENHANCEMENTS** - Improvements to existing features
- **BUG FIXES** - User-facing bug fixes
- **NOTES** - Internal changes with potential edge cases
- **UPGRADE NOTES** - Changes requiring user action when upgrading
- **BREAKING CHANGES** - Changes that could break valid configurations

For non-user-facing changes, add the `no-changelog-needed` label.

### Commit Guidelines

- Keep commits small and focused
- Run full test suite before committing
- All tests must pass
- Separate structural changes from behavioral changes
- Dependency changes should be in separate commits

## Module System

Terraform uses a hierarchical module system:
- **Root module** - The main configuration directory
- **Child modules** - Called from root or other modules
- Module loading handled by `/internal/initwd/` and `/internal/configs/`

## State File Format

State files are JSON with schema versioned separately from Terraform versions. The state includes:
- Resource instances and their attributes
- Resource dependencies
- Provider configurations
- Output values

## Provider Protocol

Terraform uses gRPC with Protocol Buffers for provider communication:
- Protocol version in `/internal/plugin/`
- Schema negotiation at plugin initialization
- CRUD operations: Read, Plan, Apply, Delete
- Provider configuration and validation

## Common Gotcaths

1. **Tests must be self-contained** - Don't rely on external state or services (except in acceptance tests)
2. **Provider plugins are separate** - This repo only contains the plugin system, not actual providers
3. **HCL parsing is complex** - Use existing helpers in `/internal/configs/` rather than parsing HCL directly
4. **Graph cycles must be prevented** - The graph engine will error on cycles
5. **State locking is critical** - Backend operations must respect locks
6. **Windows compatibility** - Note: test suite has Unix-specific assumptions

## Useful Commands for Development

```bash
# Quick validation while developing
go test ./internal/terraform -run <TestName>

# Check for race conditions
go test -race ./...

# Generate test coverage
go test -cover ./...

# Profile tests
go test -cpuprofile cpu.prof ./internal/terraform
```

## Go Module Dependencies

Dependencies managed via Go modules (`go.mod`, `go.sum`):
```bash
# Add/update dependency
go get github.com/hashicorp/hcl/v2@2.0.0

# Clean up dependencies
go mod tidy

# Verify dependencies
go mod verify
```

License restrictions: No proprietary or copyleft licenses. MPL v2, MIT, and BSD are acceptable.
