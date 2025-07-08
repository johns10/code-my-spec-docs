# Rules Module API

## Overview
Public API for the rules system that provides a clean interface for composing documentation rules based on code file paths. This module serves as the entry point for VSCode extension integration.

## Design Principles
- **Simple Public Interface**: Single static method for easy consumption
- **Delegation Pattern**: Delegates to core business logic
- **Framework Integration**: Designed for VSCode extension integration
- **Error Transparency**: Allows business logic errors to bubble up

## Public API

### `Rules.composeRules(codeFilePath: string): Promise<string>`

Composes all applicable documentation rules for a given code file into a single prompt string suitable for Claude Code.

**Parameters:**
- `codeFilePath` (string): Path to the code file being processed. Can be absolute or relative path.

**Returns:**
- `Promise<string>`: Resolves to composed prompt text containing all applicable rules

**Throws:**
- File system errors if rules directory or files are inaccessible
- Parsing errors if rule files have malformed frontmatter
- Validation errors if required rule metadata is missing

## Usage Examples

### Basic Usage
```typescript
import { Rules } from './rules';

// Compose rules for an Elixir repository file
const prompt = await Rules.composeRules('lib/my_app/repositories/user_repository.ex');

// Use with Claude Code
console.log(prompt);
```

### VSCode Extension Integration
```typescript
import * as vscode from 'vscode';
import { Rules } from './rules';

export async function generateDocumentation() {
  const activeEditor = vscode.window.activeTextEditor;
  if (!activeEditor) return;

  const filePath = activeEditor.document.uri.fsPath;
  try {
    const prompt = await Rules.composeRules(filePath);
    // Send prompt to Claude Code...
  } catch (error) {
    vscode.window.showErrorMessage(`Rules composition failed: ${error.message}`);
  }
}
```

### Error Handling
```typescript
try {
  const prompt = await Rules.composeRules('lib/controllers/user_controller.ex');
  // Process prompt...
} catch (error) {
  if (error.code === 'ENOENT') {
    console.error('Rules directory not found');
  } else {
    console.error('Rule processing failed:', error.message);
  }
}
```

## Integration Points

### With VSCode Commands
- Called from command palette actions
- Integrated with file context menus
- Triggered by keyboard shortcuts

### With Claude Code
- Output serves as prompt input for Claude Code
- Composed rules provide context for code generation
- Rules guide documentation and code style

## Architecture
- **Thin API Layer**: Minimal logic, focuses on delegation
- **Core Business Logic**: Handled by `composeRules` module
- **File System Abstraction**: Uses `fileSystem` module for operations
- **No VSCode Dependencies**: Core logic remains testable and portable

## Configuration
No direct configuration required. Rules are automatically discovered from:
- `docs/rules/` directory
- All `.md` files with valid frontmatter
- Glob patterns defined in rule frontmatter

## Performance Considerations
- Rules are read fresh on each call (no caching)
- File system operations are async
- Rule composition is done in memory
- Performance scales with number of rule files
