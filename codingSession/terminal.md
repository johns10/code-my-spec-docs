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
Three simple functions that other modules can call:
- createTerminal(name, workingDirectory)
- sendCommand(terminal, command)
- showTerminal(terminal)
