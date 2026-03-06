# Architecture

[1-3 paragraphs: Bird's eye view of what this project does and how it approaches the problem]

## Code Map

### `directory-or-module-name`

[Brief description of purpose and responsibilities]

Key types: TypeName, AnotherType, ImportantInterface

**Architecture Invariant:** [Important constraint, boundary, or guarantee]

### `another/module`

[Purpose and how it relates to other modules]

Key files: important_file.ext, config_file.ext

**Architecture Invariant:** [e.g., "This module does NOT depend on X", "All operations are idempotent", "Provides thread-safe API"]

### `third/module`

[Describe the module]

**API Boundary:** This module is intended for external use. [If applicable]

[Continue for all major modules/directories...]

## Cross-Cutting Concerns

### Testing

[Testing philosophy, strategy, where tests live, how to run them]

### Error Handling

[How errors are handled throughout the system]

### Code Generation

[If applicable: what's generated, how, when to regenerate]

### Logging/Observability

[How to observe system behavior, logging approach]

### [Other concerns as relevant]

[Build system, dependency management, performance monitoring, etc.]
