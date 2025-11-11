# Command Processor

Manages temporary files and environment variables for command execution. Maps to `src/sessions/commands.ts`.

## Purpose

The Command Processor is a simplified utility that handles temporary file creation for Claude prompts and environment variable management. It no longer contains Claude-specific CLI building logic, which has been moved to TerminalExecutor. This module focuses purely on resource management.

## Core Responsibilities

1. Create temporary files for Claude prompts
2. Track temporary files per session
3. Clean up temporary files when session ends
4. Build environment variables for commands

## What This Module Does NOT Do

- Does NOT build Claude CLI command strings (TerminalExecutor handles that)
- Does NOT execute commands (executors handle that)
- Does NOT determine which executor to use (SessionOrchestrator handles that)
- Does NOT handle resume flags or CLI options (TerminalExecutor handles that)
- Does NOT manage callback servers (CallbackServer module handles that)

## Architecture

### Simplified Design

The CommandProcessor was previously responsible for Claude-specific logic. In the refactored design, it is a pure utility for:
- Temp file lifecycle management
- Environment variable composition

This makes it simpler and more focused.

## Public Interface

```typescript
export class CommandProcessor {
  constructor()

  // Create a temporary file for a prompt
  async createTempFile(sessionId: string, content: string): Promise<string>

  // Build environment variables for a command
  buildEnvironment(
    session: Session,
    command: Command,
    apiUrl?: string,
    apiToken?: string
  ): Record<string, string>

  // Clean up temporary files for a session
  cleanupSession(sessionId: string): void

  // Clean up all temporary files
  cleanupAll(): void
}
```

## Temporary File Management

### Temp Directory

```typescript
private tempDir: string;
private tempFiles: Map<string, string[]> = new Map();

constructor() {
  // Create a temporary directory for prompt files
  this.tempDir = path.join(os.tmpdir(), 'code-my-spec-pipes');
  this.ensureTempDir();
}

private ensureTempDir(): void {
  if (!fs.existsSync(this.tempDir)) {
    fs.mkdirSync(this.tempDir, { recursive: true });
  }
}
```

Example path: `/tmp/code-my-spec-pipes/`

### Creating Temp Files

```typescript
async createTempFile(sessionId: string, content: string): Promise<string> {
  // Ensure temp directory exists
  this.ensureTempDir();

  // Create unique filename
  const tempFileName = `prompt_${sessionId}_${Date.now()}.tmp`;
  const tempFilePath = path.join(this.tempDir, tempFileName);

  // Write content to file
  fs.writeFileSync(tempFilePath, content, 'utf8');

  // Track for cleanup
  if (!this.tempFiles.has(sessionId)) {
    this.tempFiles.set(sessionId, []);
  }
  this.tempFiles.get(sessionId)!.push(tempFilePath);

  return tempFilePath;
}
```

Example output: `/tmp/code-my-spec-pipes/prompt_session123_1704067200000.tmp`

### File Tracking

Temp files are tracked per session:

```typescript
Map<string, string[]>
// sessionId -> [filePath1, filePath2, ...]
```

This allows cleanup when a session ends.

## Environment Variable Management

```typescript
buildEnvironment(
  session: Session,
  command: Command,
  apiUrl?: string,
  apiToken?: string
): Record<string, string> {
  const env: Record<string, string> = {};

  // API credentials (for hooks)
  if (apiUrl) {
    env.CODE_MY_SPEC_API_URL = apiUrl;
  }
  if (apiToken) {
    env.CODE_MY_SPEC_API_TOKEN = apiToken;
  }

  // Session ID (always set)
  env.CODE_MY_SPEC_SESSION_ID = String(session.id);

  // User-defined env vars from command metadata
  if (command.metadata?.env_vars) {
    Object.assign(env, command.metadata.env_vars);
  }

  return env;
}
```

### Environment Variables Set

| Variable | Purpose | When Set |
|---|---|---|
| `CODE_MY_SPEC_API_URL` | Backend API endpoint | Always (if apiUrl provided) |
| `CODE_MY_SPEC_API_TOKEN` | Auth token for API | Always (if apiToken provided) |
| `CODE_MY_SPEC_SESSION_ID` | Current session ID | Always |
| `CODE_MY_SPEC_CALLBACK_URL` | Callback server URL | Claude commands only (set by executor) |
| Custom vars | User-defined | If present in command.metadata.env_vars |

Note: `CODE_MY_SPEC_CALLBACK_URL` is NOT set by CommandProcessor - executors handle that.

## Cleanup

### Per-Session Cleanup

```typescript
cleanupSession(sessionId: string): void {
  const sessionFiles = this.tempFiles.get(sessionId);
  if (sessionFiles) {
    for (const filePath of sessionFiles) {
      try {
        if (fs.existsSync(filePath)) {
          fs.unlinkSync(filePath);
          getLogger().appendLine(`Cleaned up temp file: ${filePath}`);
        }
      } catch (error) {
        getLogger().appendLine(`Error cleaning up temp file ${filePath}: ${error}`);
      }
    }
    this.tempFiles.delete(sessionId);
  }
}
```

Called by executors when a session ends.

### Global Cleanup

```typescript
cleanupAll(): void {
  try {
    if (fs.existsSync(this.tempDir)) {
      const files = fs.readdirSync(this.tempDir);
      for (const file of files) {
        const filePath = path.join(this.tempDir, file);
        fs.unlinkSync(filePath);
      }
      fs.rmdirSync(this.tempDir);
      getLogger().appendLine(`Cleaned up temp directory: ${this.tempDir}`);
    }
  } catch (error) {
    getLogger().appendLine(`Error cleaning up temp directory: ${error}`);
  }
  this.tempFiles.clear();
}
```

Called when extension deactivates.

## Usage Example (ClaudeTerminalExecutor)

```typescript
// In ClaudeTerminalExecutor
const commandProcessor = new CommandProcessor();

async buildClaudeCommand(session: Session, command: Command): Promise<string> {
  // Extract prompt
  const prompt = command.metadata?.prompt || command.command;

  // Create temp file for prompt
  const tempFilePath = await commandProcessor.createTempFile(
    session.id,
    prompt
  );

  // Build environment
  const env = commandProcessor.buildEnvironment(
    session,
    command,
    this.apiUrl,
    this.apiToken
  );

  // Build Claude CLI command
  const claudeCommand = ClaudeContext.buildCommandString('', options);

  // Pipe prompt into claude
  return {
    command: `cat "${tempFilePath}" | ${claudeCommand}`,
    env
  };
}

// On cleanup
async cleanupSession(sessionId: string): Promise<void> {
  commandProcessor.cleanupSession(sessionId);
  // ... other cleanup
}
```

## Migration from Old Design

### Removed

- `processCommand()` method - Too complex, split into smaller methods
- Claude CLI building logic - Moved to ClaudeTerminalExecutor
- Resume flag handling - Moved to ClaudeTerminalExecutor
- Prompt extraction - Moved to ClaudeTerminalExecutor
- Options extraction - Moved to ClaudeTerminalExecutor

### Added

- `createTempFile()` - Explicit temp file creation
- `buildEnvironment()` - Explicit env var building
- Clearer separation of concerns

### Why This Change?

The old `processCommand()` method did too many things:
1. Built Claude CLI commands
2. Created temp files
3. Set environment variables
4. Made decisions about when to pipe prompts

This was problematic because:
- Only ClaudeTerminalExecutor needed Claude CLI building
- SubprocessExecutor called it but didn't need Claude logic
- ClaudeExecutor didn't call it at all
- TerminalExecutor didn't need it for shell commands

The new design is simpler:
- CommandProcessor: Pure utility for files and env vars
- ClaudeTerminalExecutor: Owns Claude CLI building logic
- TerminalExecutor: Simple shell execution, no CLI building
- Each executor gets what it needs, nothing more

## Error Handling

### File Creation Errors

If `createTempFile()` fails (disk full, permissions, etc.):
- Error is thrown to caller
- Caller (executor) decides how to handle

### Cleanup Errors

If cleanup fails:
- Error is logged but not thrown
- Other files continue to be cleaned up
- Prevents one bad file from blocking all cleanup

## File Naming Convention

```
prompt_<sessionId>_<timestamp>.tmp
```

Examples:
- `prompt_session-abc123_1704067200000.tmp`
- `prompt_42_1704067201234.tmp`

The timestamp ensures uniqueness even if multiple commands execute rapidly in the same session.

## Temp File Lifecycle

```
1. Command received
        ↓
2. createTempFile() called
        ↓
3. File written to disk
        ↓
4. File path returned
        ↓
5. Executor uses path in command
        ↓
6. Command executes (reads file)
        ↓
7. Command completes
        ↓
8. Session ends
        ↓
9. cleanupSession() called
        ↓
10. File deleted from disk
```

Note: Files persist until explicit cleanup. This is safe because:
- Files are in OS temp directory
- OS may clean up temp directories on reboot
- Extension cleanup ensures removal on session end

## Dependencies

- `fs` (Node.js built-in) - File system operations
- `path` (Node.js built-in) - Path manipulation
- `os` (Node.js built-in) - Temp directory location
- `getLogger()` - Extension output channel logging
