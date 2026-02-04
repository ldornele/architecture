# HyperFleet Documentation Standard

**Status**: Active
**Owner**: HyperFleet Team
**Last Updated**: 2026-01-08

---

## Purpose

This document establishes the standard approach for structuring documentation across all HyperFleet repositories. The goal is to create a homogeneous documentation structure that engineers can navigate consistently regardless of which repo they are working in.

---

## Standard Documentation Structure

All HyperFleet repositories MUST follow this directory structure:

```text
repository-name/
├── README.md                    # Project overview and getting started (REQUIRED)
├── CONTRIBUTING.md              # Development and contribution guidelines (REQUIRED)
├── CHANGELOG.md                 # Release history in Keep a Changelog format (REQUIRED)
├── docs/                        # Detailed documentation directory (OPTIONAL)
│   ├── configs/                 # Configuration guides and references
│   ├── development/             # Development setup and workflows
│   ├── deployment/              # Deployment guides and procedures
│   ├── troubleshooting/         # Debugging and troubleshooting guides
│   └── examples/                # Usage examples and tutorials
└── templates/                   # Template files (if applicable to the project)
```

### Directory Descriptions

#### `/docs/configs/`
- Configuration file references and examples
- Environment-specific configuration guides
- Configuration validation and troubleshooting

#### `/docs/development/`
- Local development setup instructions
- Development workflows and best practices
- Coding standards and contribution guidelines
- Build and test procedures

#### `/docs/deployment/`
- Production deployment procedures
- Environment setup and configuration
- Infrastructure requirements
- Monitoring and operational guides

#### `/docs/troubleshooting/`
- Common issues and solutions
- Debugging guides and techniques
- Performance troubleshooting
- Log analysis and error resolution

#### `/docs/examples/`
- Usage examples and tutorials
- Integration examples
- Configuration examples
- Common use case walkthroughs

### Notes on Structure
- **docs/ directory is optional** - only create it if you have substantial documentation beyond README and CONTRIBUTING
- **Keep it simple** - don't create directories unless you have multiple files to organize

---

## README.md Required Sections

Every repository README.md MUST include these sections in order:

### 1. Title & Description
```markdown
# Repository Name

[2-3 sentence description of what this component does and its role in HyperFleet]
```

### 2. Quick Start
```markdown
## Quick Start

[Essential setup steps that get someone running the project/service]
```

### 3. Prerequisites
```markdown
## Prerequisites

[Required tools, versions, access, etc.]
```

### 4. Installation
```markdown
## Installation

[Detailed setup instructions with commands]
```

### 5. Usage/Core Features
```markdown
## Usage
# OR
## API Resources
# OR
## Core Features

[How to use the component, main endpoints, key functionality]
```

### 6. Architecture Repository Link
```markdown
## Documentation

- [Architecture](https://github.com/openshift-hyperfleet/architecture) - System architecture and API documentation
```

### 7. Additional Documentation (if docs/ directory exists)
```markdown
- [Configuration](docs/configs/README.md) - Configuration guides and reference
- [Development](docs/development/README.md) - Development setup and workflows
- [Deployment](docs/deployment/README.md) - Deployment guides and procedures
- [Examples](docs/examples/README.md) - Usage examples and tutorials
- [Troubleshooting](docs/troubleshooting/README.md) - Debugging and troubleshooting guides
```

### Repository-Specific Sections

After the required sections, repositories MAY add additional sections specific to their domain:

- **API repositories**: API Resources, Authentication, Rate Limiting
- **Infrastructure repositories**: Architecture, Terraform modules, Environment setup
- **CLI tools**: Commands, Configuration, Examples
- **Libraries**: Usage examples, API reference, Migration guides

---

## CONTRIBUTING.md Required Sections

Every repository MUST include a CONTRIBUTING.md file with these sections:

### 1. Development Setup
```markdown
## Development Setup

[Complete local environment setup including prerequisites, installation, and first run]
```

### 2. Repository Structure
```markdown
## Repository Structure

[Overview of key directories and files - what's where and why]
```

### 3. Testing
```markdown
## Testing

[How to run unit tests, integration tests, linting, and validation]
```

### 4. Common Development Tasks
```markdown
## Common Development Tasks

[Build commands, running locally, generating code, etc.]
```

### 5. Commit Standards
```markdown
## Commit Standards

This project follows [HyperFleet commit standards](../standards/commit-standard.md).
```

### 6. Release Process (if applicable)
```markdown
## Release Process

[How releases are created and published, or note if not applicable]
```

---

## Changelog Format

All repositories MUST maintain a `CHANGELOG.md` file following the [Keep a Changelog](https://keepachangelog.com/) format.

### Template Structure
```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- New features

### Changed
- Changes in existing functionality

### Deprecated
- Soon-to-be removed features

### Removed
- Now removed features

### Fixed
- Bug fixes

### Security
- Vulnerability fixes

## [1.0.0](https://github.com/openshift-hyperfleet/architecture/compare/v0.0.0...v1.0.0) - YYYY-MM-DD
### Added
- Initial release
```

---

## API Documentation Approach

### REST APIs
- **MUST** use OpenAPI 3.0 specifications
- **MUST** embed specifications using `//go:embed` for Go services (following hyperfleet-api pattern)
- **SHOULD** provide Swagger UI endpoints for interactive exploration
- **MUST** include API examples in documentation

### Go Packages
- **MUST** use standard `godoc` conventions
- **SHOULD** include package-level documentation
- **MUST** document public functions, types, and methods

### API Documentation Location
- API and architecture documentation are captured in the [architecture repository](https://github.com/openshift-hyperfleet/architecture)
- Service-specific examples: `docs/examples/` or main README.md if simple

---

## Architecture Decision Records (ADRs)

ADRs are **optional** but recommended for complex components that require architectural decisions to be documented.

### ADR Location (if used)
- **Location**: [architecture repository](https://github.com/openshift-hyperfleet/architecture) `docs/architecture/adr/` directory
- **Naming**: `NNNN-title-of-decision.md` (e.g., `0001-use-openapi-for-api-specs.md`)
- **Format**: Follow standard ADR template with Status, Context, Decision, Consequences

### When to Use ADRs
- Significant architectural decisions that affect multiple components
- Technology choices with trade-offs that should be documented
- Design patterns that need rationale for future reference
- Integration approaches between HyperFleet components

---

## Template Files

This standard includes template files to bootstrap new repositories:

- `templates/README.md.template` - README template with all required sections
- `templates/CONTRIBUTING.md.template` - Contributing guidelines template
- `templates/CHANGELOG.md.template` - Changelog template in Keep a Changelog format

### Using Templates

When creating a new repository:

1. Copy the appropriate template files
2. Replace placeholder text with project-specific information
3. Customize repository-specific sections as needed
4. Ensure all required sections are completed

---

## Related Standards

This documentation standard builds upon and references other HyperFleet standards:
- [Commit Standard](../standards/commit-standard.md) - Commit message formatting
- [Generated Code Policy](../standards/generated-code-policy.md) - Handling generated documentation
- [Linting Standard](../standards/linting.md) - Code quality that affects inline documentation

---

## Changelog

### 2026-01-08 - Initial Version
- Created standard for HyperFleet documentation structure
- Defined required README.md and CONTRIBUTING.md sections
- Created template structure for new repositories
- Specified Keep a Changelog format for all repositories

---