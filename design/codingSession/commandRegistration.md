# Command Registration Module Design

## Purpose
Pure VSCode command registration and entry point. Handles command palette integration only.

## Single Happy Path Operation

### registerStartCodingSessionCommand(callback)
1. Register "Start Coding Session" command with VSCode
2. Set up keyboard shortcut (if any)
3. Wire command to provided callback function
4. Done

## What This Module Does NOT Do
- Business logic execution
- File operations
- Error handling beyond VSCode requirements
- Session management
- User workflow orchestration

## Implementation
- Pure VSCode command API wrapper
- Uses vscode.commands.registerCommand API
- Passes execution to callback function provided by caller
- No side effects beyond command registration
- Easily mockable for testing

## Interface
One simple function that extension activation calls:
- registerStartCodingSessionCommand(callback)

## Usage
```
// Extension calls this during activation
registerStartCodingSessionCommand(() => {
  // Callback executes the actual business logic
  startCodingSession();
});
```
