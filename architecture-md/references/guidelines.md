# ARCHITECTURE.md Guidelines

Based on matklad's article: https://matklad.github.io/2021/02/06/ARCHITECTURE.md.html

## Core Principles

**Target audience**: Recurring contributors who need to understand where code lives and how modules relate.

**Key value**: Bridge the gap between reading code sequentially (new contributors) and having a mental map (core developers). The ARCHITECTURE.md creates that mental map.

**Main questions to answer:**
- "Where's the thing that does X?" (codemap)
- "What does the thing I'm looking at do?" (module purpose)

## What to Include

### 1. Bird's Eye Overview
Start with the problem domain. What does this project solve? What's the high-level approach?

Example from rust-analyzer:
> On the highest level, rust-analyzer is a thing which accepts input source code from the client and produces a structured semantic model of the code.

### 2. Codemap
Describe coarse-grained modules and their relationships. This is a map of the country, not an atlas of individual states.

**DO:**
- Name important files, modules, and types
- Explain module boundaries and relationships
- Group related directories/crates together
- Reflect on whether directory structure matches conceptual structure

**DON'T:**
- Link directly to files (links go stale)
- Explain *how* each module works internally (use inline docs for that)
- Go into fine-grained details

**Technique:** Encourage readers to use symbol search to find named entities. This:
- Doesn't require maintenance
- Helps discover similarly-named related things
- Works even as code evolves

### 3. Architecture Invariants
Explicitly call out invariants, especially:
- **Absences**: "Module X does NOT depend on Y"
- **Boundaries**: "The `syntax` crate is completely independent"
- **Constraints**: "Parsing never fails, produces `(T, Vec<Error>)`"
- **API boundaries**: Which crates/modules are public-facing

Example invariants from rust-analyzer:
- "syntax tree is a value type" (no global context needed)
- "syntax tree is built for a single file" (enables parallel parsing)
- "base_db knows nothing about cargo" (abstraction boundary)
- "typing inside a function body never invalidates global derived data" (incremental compilation)

### 4. Cross-Cutting Concerns
Separate section for things that span multiple modules:
- Testing strategy
- Error handling approach
- Logging/observability
- Code generation
- Cancellation/interruption handling
- Configuration management

## What to Exclude

**Don't include:**
- Low-level implementation details (use inline documentation)
- How-to guides for specific tasks (use CONTRIBUTING.md or docs/)
- Frequently changing information (will go stale)
- Complete API documentation (use rustdoc/javadoc/etc.)

## Maintenance Strategy

**Keep it short**: Every recurring contributor reads it. Shorter = less likely to become stale.

**Main rule**: Only specify things unlikely to change frequently.

**Don't try to keep synchronized with code.** Instead, revisit a couple times per year.

**When to update:**
- Major architectural refactoring
- New major modules/boundaries
- Significant invariant changes
- Annual/quarterly reviews

## Structure Template

```markdown
# Architecture

High-level overview of the problem domain (1-3 paragraphs)

## Code Map

### `directory/module-name`
Purpose and responsibilities. Key types: TypeName, OtherType.

**Architecture Invariant:** Important constraints or guarantees.

### `another/module`
...

## Cross-Cutting Concerns

### Testing
Testing philosophy and approach

### Error Handling
How errors are handled throughout

### [Other concerns]
...
```

## Length Guidance

- **Ideal**: 200-500 lines
- **Maximum**: 1000 lines before considering splitting into multiple docs
- If longer than 1000 lines, create separate architecture docs per major subsystem

## Tips for Writing

1. **Use present tense**: Describe the architecture as it currently is
2. **Be specific about names**: "The `Parser` type in `syntax/parser.rs`" not "the parser"
3. **Explain the 'why'**: Why this boundary? Why this invariant? 
4. **Use examples**: Show concrete module relationships, not just abstract descriptions
5. **Think in layers**: UI → IDE features → Semantic model → Syntax tree
6. **Highlight API boundaries**: Mark which modules are meant for external consumption
