# Coding Session API Design

## Purpose
Public API for the coding session context. Provides clean interface for starting coding sessions and registering commands.

## Public Interface

### startDesignSession()
Main function to initiate a coding session from the currently active file.

**Returns**: Promise<void>
**Throws**: Error if not in a valid doc file or if session setup fails

### registerCommands()
Registers all coding session commands with VSCode during extension activation.

**Returns**: void
**Usage**: Called once during extension startup

## What This Module Does
- Exports the public API for the coding session context
- Hides internal implementation details
- Provides clean interface for other parts of the extension
- Acts as the single entry point for one-off code generation

## What This Module Does NOT Do
- Business logic implementation (delegates to startDesignSession.ts)
- VSCode API calls (delegates to utility modules)
- Command registration logic (delegates to commandRegistration.ts)

## Usage Examples

### From Extension Activation
```typescript
import { registerCommands } from './designSession';

export function activate(context: vscode.ExtensionContext) {
  registerCommands();
}
```

### From Other Modules
```typescript
import { startDesignSession } from './designSession';

// Start a coding session programmatically
await startDesignSession();
```

## Internal Module Coordination
- Imports startDesignSession from startDesignSession.ts
- Imports registerCommands from commandRegistration.ts
- Re-exports both as clean public API
- Internal modules remain hidden from external consumers

## Interface Design
Simple, focused API with just two functions:
- startDesignSession() - the main workflow
- registerCommands() - extension setup
