# Start Coding Session Module Design

## Purpose
Core business logic for starting a coding session. Orchestrates file creation, window setup, and terminal initialization.

## Single Happy Path Operation

### startDesignSession()
1. Get currently active file in VSCode
2. Verify it's a doc file in docs/design/ directory
3. Determine corresponding code file path (docs/design/my_app/user_auth.md → lib/my_app/user_auth.ex)
4. Create code file if it doesn't exist (basic Elixir module template)
5. Set up two-column layout
6. Open doc file in left column
7. Open code file in right column
8. Create "Claude Code (Docs)" terminal in docs directory
9. Create "Claude Code (Code)" terminal in project root
10. Compose prompt rules using Rules.composeRules(codeFilePath)
11. Send Claude Code initialization commands to both terminals with composed rules
12. Done

## File Path Logic
- Always starts from doc file (user must be in docs/my_app/something.md)
- Maps doc file to code file: docs/my_app/user_auth.md → lib/my_app/user_auth.ex
- Creates code file from basic template if missing

## Template Creation
- Creates basic Elixir module
- Uses file name to generate module name
- No complex template resolution - one simple template

## Claude Code Setup
- Composes prompt rules using Rules.composeRules(codeFilePath)
- Sends initialization commands to both terminals with composed rules
- Sets appropriate working directory for each terminal

## Dependencies
Uses utility modules:
- window.setupTwoColumnLayout()
- window.openFileInLeftColumn(docPath)
- window.openFileInRightColumn(codePath)
- terminal.createTerminal(name, workingDir)
- terminal.sendCommand(terminal, command)
- terminal.showTerminal(terminal)
- Rules.composeRules(codeFilePath) for prompt rule composition

## What This Module Does NOT Do
- VSCode API calls (delegates to utility modules)
- Complex template resolution
- Multiple workflow types
- Error recovery logic

## Implementation
- Pure business logic with no VSCode API dependencies
- Uses Node.js fs for file operations
- Uses path utilities for file path manipulation
- Delegates all VSCode operations to utility modules
- Easily testable with mocked utilities
