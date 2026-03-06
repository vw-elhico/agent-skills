---
name: architecture
description: Create ARCHITECTURE.md documentation for codebases (10k-200k LOC). Use when asked to document project architecture, create ARCHITECTURE.md, explain codebase structure to contributors, or map out high-level system design. Based on matklad's ARCHITECTURE.md pattern for bridging the gap between new and core contributors.
metadata:
  internal: true
---

# Architecture Documentation

Create ARCHITECTURE.md files that help contributors understand where code lives and how modules relate.

## Overview

ARCHITECTURE.md bridges the gap between new contributors (who read code sequentially) and core developers (who have a mental map). It answers:
- "Where's the thing that does X?" (codemap)
- "What does this code I'm looking at do?" (module purpose)

**Target**: 10k-200k line codebases with recurring contributors.

## Workflow

### 1. Analyze Codebase Structure

Use the Task tool (explore agent, "very thorough") to understand:

- **Directory structure**: Major directories/modules/crates
- **Key files**: Entry points, main configuration files
- **Module boundaries**: How code is organized
- **Dependencies**: How modules relate to each other
- **Important types/interfaces**: Core abstractions

Example Task prompt:
```
Analyze this codebase architecture. Find:
1. All top-level directories and their purpose
2. Key entry points (main.rs, app.py, index.ts, etc.)
3. Major modules and how they relate
4. Important types/classes/interfaces
5. Build system and configuration approach
```

### 2. Identify Architecture Invariants

Look for and call out explicitly:

**Absences** (deliberately missing dependencies):
- "Module X does NOT depend on Y"
- "The syntax crate knows nothing about LSP"
- "base_db doesn't know about file system paths"

**Boundaries**:
- Which modules are API boundaries (public-facing)
- Which are internal implementation details
- Layering: "IDE features → semantic model → syntax tree"

**Constraints**:
- "Parsing never fails, returns `(T, Vec<Error>)`"
- "Syntax tree is a value type (no global context)"
- "All operations are idempotent"
- "Typing inside function bodies never invalidates global data"

**Look for what's NOT there** — absences are critical invariants but hard to discover from code alone.

### 3. Draft Bird's Eye Overview

Start ARCHITECTURE.md with 1-3 paragraphs answering:
- What problem does this project solve?
- What's the high-level approach?
- What are the main input/output?

Example (rust-analyzer):
> On the highest level, rust-analyzer accepts input source code from the client and produces a structured semantic model of the code.

### 4. Create Codemap

For each major directory/module, write a section:

```markdown
### `directory-name` or `crate-name`

[1-2 sentences: purpose and responsibilities]

Key types: TypeName, InterfaceName, ClassName
Key files: important.rs, config.toml

**Architecture Invariant:** [Important constraint or boundary]
**API Boundary:** [If this module is public-facing]
```

**DO:**
- Name specific types, files, modules (enables symbol search)
- Explain relationships between modules
- Group related modules together
- Reflect on directory structure vs conceptual structure

**DON'T:**
- Link to files directly (goes stale)
- Explain internal implementation (use inline docs)
- Include fine-grained details
- Document how each module works internally

**Use symbol search approach**: Name entities without linking. Readers use editor's "go to symbol" to find them. This:
- Doesn't require maintenance
- Works as code evolves
- Discovers similarly-named related things

### 5. Document Cross-Cutting Concerns

Separate section for system-wide concerns:

**Common concerns:**
- Testing (philosophy, location, how to run)
- Error handling (approach throughout system)
- Logging/Observability (how to see what's happening)
- Code generation (what's generated, when to regenerate)
- Configuration management
- Build system
- Cancellation/interruption handling

### 6. Keep It Short

**Target length**: 200-500 lines
**Maximum**: 1000 lines before splitting

**Why short?**
- Every recurring contributor reads it
- Less likely to go stale
- Easier to maintain

**What to exclude:**
- Implementation details (use inline docs)
- How-to guides (use CONTRIBUTING.md)
- Frequently changing info
- Complete API docs (use language-specific tooling)

### 7. Create the File

Use the template from `references/template.md` as starting structure. Follow guidelines from `references/guidelines.md`.

**Maintenance strategy:**
- Only include things unlikely to change frequently
- Don't try to keep synchronized with code
- Revisit 2-4 times per year
- Update on major architectural refactoring

## Tips

**Be specific with names**: "The `Parser` type in `syntax/parser.rs`" not "the parser"

**Explain the 'why'**: Why this boundary? Why this invariant?

**Use present tense**: Describe architecture as it currently is

**Think in layers**: Show the dependency flow (UI → features → core → syntax)

**Mark API boundaries**: Clarify which modules are for external use

**Use concrete examples**: Show actual module relationships, not abstractions

## References

See `references/guidelines.md` for detailed ARCHITECTURE.md writing guidelines.

See `references/template.md` for a structure template.

**Original article**: https://matklad.github.io/2021/02/06/ARCHITECTURE.md.html

**Example**: rust-analyzer architecture.md (referenced in guidelines)
