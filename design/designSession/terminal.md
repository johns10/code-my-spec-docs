# Terminal Module Design

## Purpose
Pure VSCode terminal control operations. Handles terminal creation and command injection only.

## Single Happy Path Operations

### createTerminal(name, workingDirectory)
1. Create VSCode terminal with specified name and working directory
2. Return terminal instance
3. Done

### sendCommand(terminal, command)
1. Send text command to specified terminal
2. Done

### showTerminal(terminal)
1. Make specified terminal visible and focused
2. Done

### upsertTerminal(name, workingDirectory)
1. Find existing terminal by name
2. If found, return existing terminal
3. If not found, create new terminal with specified name and working directory
4. Return terminal instance
5. Done

### createSplitTerminal(name, workingDirectory)
1. Create VSCode terminal with specified name and working directory
2. Show terminal in split view
3. Return terminal instance
4. Done

## What This Module Does NOT Do
- Claude Code session management
- Prompt rule file reading
- Command orchestration
- Error handling
- Business logic

## Implementation
- Pure VSCode terminal API wrapper functions
- Uses VSCode window.createTerminal API
- Uses terminal.sendText API for command injection
- No side effects beyond terminal operations
- Easily mockable for testing

## Interface
Five simple functions that other modules can call:
- createTerminal(name, workingDirectory)
- sendCommand(terminal, command)
- showTerminal(terminal)
- upsertTerminal(name, workingDirectory)
- createSplitTerminal(name, workingDirectory)
