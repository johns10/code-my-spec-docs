# File System Abstraction

## Overview
Pure file system operations module that provides clean abstractions for reading files and listing directories. Contains no business logic and is designed to be easily mockable for testing.

## Design Principles
- **Pure Operations**: No business logic, only file system interactions
- **Testability**: Interface-based design for easy mocking
- **Error Transparency**: Let file system errors bubble up naturally
- **Simple Scope**: Focus only on the operations needed by the rules system

## Interface

```typescript
interface FileSystemOperations {
  readFile(filePath: string): Promise<string>;
  listFiles(directory: string): Promise<string[]>;
}
```

## Operations

### `readFile(filePath: string): Promise<string>`
Reads the contents of a file and returns it as a UTF-8 string.

**Parameters:**
- `filePath` - Absolute or relative path to the file

**Returns:**
- Promise resolving to file contents as string

**Throws:**
- File system errors (ENOENT, EACCES, etc.) are allowed to bubble up

### `listFiles(directory: string): Promise<string[]>`
Lists all files in a directory (non-recursive, top-level only).

**Parameters:**
- `directory` - Path to directory to list

**Returns:**
- Promise resolving to array of filenames (not full paths)

**Throws:**
- File system errors if directory doesn't exist or can't be read

## Implementation Notes
- Uses Node.js `fs/promises` for async file operations
- Returns only filenames from `listFiles`, not full paths
- No filtering or business logic - pure file system operations
- Designed to be swapped out easily for different environments or testing

## Usage Example

```typescript
const fs = new FileSystem();

// Read a rule file
const content = await fs.readFile('docs/rules/repository_patterns.md');

// List all rule files
const files = await fs.listFiles('docs/rules/');
```

## Testing Strategy
The interface design allows for easy mocking:

```typescript
const mockFileSystem: FileSystemOperations = {
  readFile: jest.fn(),
  listFiles: jest.fn()
};
```
