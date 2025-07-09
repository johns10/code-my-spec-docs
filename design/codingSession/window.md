# Window Module Design

## Purpose
Pure VSCode window and workspace control operations. Handles layout configuration, file opening, workspace information, and user messaging.

## Operations

### Layout Operations
#### setupTwoColumnLayout()
1. If less than 2 columns, configure VSCode to two-column layout
2. Done

#### openFileInLeftColumn(filePath)
1. Open specified file in left column
2. Done

#### openFileInRightColumn(filePath)
1. Open specified file in right column  
2. Done

### Context Operations
#### getActiveFilePath()
1. Get the file path of the currently active editor
2. Return file path string or null if no active editor
3. Done

#### getWorkspaceRoot()
1. Get the root directory of the current workspace
2. Return workspace root path string or null if no workspace
3. Done

### User Messaging Operations
#### showErrorMessage(message)
1. Display error message to user via VSCode UI
2. Done

#### showInformationMessage(message)
1. Display information message to user via VSCode UI
2. Done

## What This Module Does NOT Do
- File creation or modification
- File path manipulation or transformation
- Template resolution
- Business logic
- Terminal operations

## Implementation
- Pure VSCode API wrapper functions
- Uses VSCode window API for active editor and messaging
- Uses VSCode workspace API for workspace information
- Uses VSCode workbench API for layout
- No side effects beyond VSCode window/workspace operations
- Easily mockable for testing

## Interface
Seven simple functions that other modules can call:
- setupTwoColumnLayout()
- openFileInLeftColumn(filePath)  
- openFileInRightColumn(filePath)
- getActiveFilePath()
- getWorkspaceRoot()
- showErrorMessage(message)
- showInformationMessage(message)

## Design Rationale
This module provides a complete abstraction layer for all VSCode window and workspace operations needed by the coding session workflow. By centralizing these operations here, the business logic in startCodingSession.ts can remain pure and testable without any direct VSCode API dependencies.
