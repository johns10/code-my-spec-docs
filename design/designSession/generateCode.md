# Generate Code Command

## Purpose
Generates code from a design document by calling Claude with the appropriate rules and file paths. This allows users to quickly generate code from their design specifications to validate the design quality.

## Command Flow
1. Get the currently active file in VSCode
2. Verify it's a markdown file in the `docs/design/` directory
3. Use `getCodeFilePathFromDocPath()` to determine the corresponding code file path
4. Create the code file (or clear it if it exists) using `createCodeFile()`
5. Compose rules for the code file using `composeRules(codeFilePath, rulesDirectory)`
6. Execute Claude command: `claude --verbose --prompt "{rules} create @{codeFilePath} from @{docFilePath}"` in a subprocess

## Implementation Structure
- **File**: `src/designSession/generateCode.ts`
- **Function**: `generateCode(): Promise<void>`
- **Command Registration**: Add to `commandRegistration.ts`

## Key Differences from startDesignSession
- No two-column layout setup
- No file opening in columns
- Single terminal for Claude command execution
- Focus purely on code generation, not interactive session

## Dependencies
- Reuses existing functions from `startDesignSession.ts`:
  - `getCodeFilePathFromDocPath()`
  - `createCodeFile()`
- Uses `composeRules()` from rules system
- Uses terminal utilities for command execution