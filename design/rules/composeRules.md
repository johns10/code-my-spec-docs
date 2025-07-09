# Rules Composition Engine

## Overview
Core business logic for the rules system. Matches code file paths against glob patterns defined in rule files and composes applicable rules into a single prompt for Claude Code.

## Design Principles
- **Pure Business Logic**: No VSCode dependencies, easily testable
- **Simple Matching**: Uses glob patterns for file path matching
- **Composable Rules**: Multiple rules can apply to a single file
- **Fail Fast**: Throws errors for malformed rules or missing files

## Core Function

```typescript
composeRules(codeFilePath: string): Promise<string>
```

Composes all applicable rules for a given code file path into a single prompt string.

## Rule Processing Flow

1. **Discovery**: List all `.md` files in `docs/rules/` directory
2. **Parsing**: Extract frontmatter metadata from each rule file
3. **Matching**: Test code file path against each rule's glob pattern
4. **Composition**: Concatenate all matching rule contents with newlines
5. **Return**: Single composed prompt string

## Rule File Format

Rule files are markdown with YAML frontmatter:

```markdown
---
glob: "lib/**/repositories/*.ex"
---

# Repository Pattern Rules
[Rule content here]
```

### Frontmatter Fields
- `glob` (required): Glob pattern to match against file paths

## Glob Matching
- Uses standard glob patterns (e.g., `**`, `*`, `?`)
- Matches against the provided `codeFilePath` parameter
- Case-sensitive matching
- Examples:
  - `*.ex` - All Elixir files
  - `lib/**/*.ex` - All Elixir files in lib directory tree
  - `lib/**/repositories/*.ex` - Repository files in lib

## Composition Rules
- Multiple rules can match a single file
- Matching rules are concatenated with newlines
- No specific ordering (LLM doesn't care about rule order)
- Empty rules or malformed files cause errors

## Error Handling
- **Missing rules directory**: Throws file system error
- **Malformed frontmatter**: Throws YAML parsing error
- **Missing glob field**: Throws validation error
- **File read errors**: Bubbles up file system errors

## Dependencies
- File system operations via `FileSystemOperations` interface
- Glob matching library (minimatch or similar)
- YAML frontmatter parsing (gray-matter or similar)

## Usage Example

```typescript
// For a repository file
const prompt = await composeRules('lib/my_app/repositories/user_repository.ex');

// Result might include:
// - Base Elixir rules (glob: "*.ex")
// - Repository-specific rules (glob: "lib/**/repositories/*.ex")
// - Any other matching patterns
```

## Testing Strategy
- Mock file system operations for isolated testing
- Test glob pattern matching with various file paths
- Verify rule composition order and formatting
- Test error conditions (missing files, malformed frontmatter)

## Implementation Notes
- Keeps business logic separate from file system concerns
- Designed to be framework-agnostic (no VSCode dependencies)
- Uses dependency injection for file system operations
- Frontmatter parsing handles both YAML and simple key-value formats
