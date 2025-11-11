# Subprocess Executor

Executes shell commands in background subprocesses for agentic mode. Maps to `src/sessions/executors/subprocess.ts`.

## Purpose

The Subprocess Executor runs shell commands in background Node.js child processes, invisible to the user. It captures stdout/stderr and waits for process completion. This executor is used exclusively for agentic mode when executing non-Claude shell commands.

## Core Responsibilities

1. Spawn shell commands in background child processes
2. Capture stdout and stderr streams
3. Wait for process completion and exit code
4. Return structured command results
5. Handle process errors gracefully

## What This Module Does NOT Do

- Does NOT run commands in VSCode terminal (TerminalExecutor handles that)
- Does NOT use Claude Agent SDK (ClaudeExecutor handles that)
- Does NOT handle Claude CLI commands (ClaudeExecutor handles that)
- Does NOT use CommandProcessor for Claude logic (simplified to only handle shell commands)
- Does NOT interact with callback servers (no hooks for shell commands)

## Architecture

### Execution Flow

```
SubprocessExecutor.execute()
        ↓
Validate command is shell command (not Claude)
        ↓
Set up environment variables
        ↓
spawn(command, { shell: true, cwd, env })
        ↓
Attach stdout/stderr data handlers
        ↓
Attach close/error event handlers
        ↓
Wait for process completion
        ↓
Return { stdout, stderr, exitCode, duration_ms }
```

## Public Interface

```typescript
export class SubprocessExecutor {
  constructor(sessionUtils?: SessionUtilities)

  // Execute a command in subprocess
  async execute(
    sessionId: string,
    interaction: Interaction,
    session: Session
  ): Promise<CommandResult>

  // Clean up subprocess executor resources
  cleanupSession(sessionId: string): void
}

interface SessionUtilities {
  getApiUrl(): string;
  getAuthToken(): Promise<string | null>;
}
```

## Command Validation

SubprocessExecutor should only handle shell commands, not Claude commands:

```typescript
private validateShellCommand(command: Command): void {
  const isClaudeCommand = command.command === 'claude' || command.metadata?.prompt;

  if (isClaudeCommand) {
    throw new Error(
      'SubprocessExecutor cannot execute Claude commands. Use ClaudeExecutor for agentic Claude execution.'
    );
  }
}
```

## Environment Variables

The executor sets minimal environment variables for shell commands:

```typescript
const env = {
  ...process.env, // Inherit parent process environment

  // API credentials (in case hooks are configured)
  CODE_MY_SPEC_API_URL: apiUrl,
  CODE_MY_SPEC_API_TOKEN: apiToken || '',
  CODE_MY_SPEC_SESSION_ID: String(session.id),

  // User-defined env vars from command metadata
  ...command.metadata?.env_vars
};
```

Note: Claude-specific env vars (like `CODE_MY_SPEC_CALLBACK_URL`) are NOT set since this executor doesn't handle Claude commands.

## Working Directory

The subprocess respects the working directory from command metadata:

```typescript
const workingDirectory =
  command.metadata?.working_directory ||
  process.env.CLAUDE_WORKSPACE_PATH ||
  process.cwd();
```

## Complete Execute Method

```typescript
async execute(
  sessionId: string,
  interaction: Interaction,
  session: Session
): Promise<CommandResult> {
  if (!interaction.id || !interaction.command) {
    throw new Error('No interaction ID or command available');
  }

  // Normalize command
  const command: Command = typeof interaction.command === 'string'
    ? { module: 'unknown', command: interaction.command, timestamp: new Date().toISOString() }
    : interaction.command;

  const startTime = Date.now();

  return new Promise(async (resolve) => {
    try {
      // Handle empty commands
      if (!command.command || command.command.trim() === '') {
        resolve({ stdout: '', stderr: '', exitCode: 0, duration_ms: 0 });
        return;
      }

      // Validate this is a shell command
      this.validateShellCommand(command);

      // Get API credentials
      const apiUrl = this.sessionUtils?.getApiUrl();
      const apiToken = this.sessionUtils ? await this.sessionUtils.getAuthToken() : null;

      // Build environment
      const env = {
        ...process.env,
        CODE_MY_SPEC_API_URL: apiUrl || '',
        CODE_MY_SPEC_API_TOKEN: apiToken || '',
        CODE_MY_SPEC_SESSION_ID: String(session.id),
        ...command.metadata?.env_vars
      };

      // Determine working directory
      const workingDirectory =
        command.metadata?.working_directory ||
        process.env.CLAUDE_WORKSPACE_PATH ||
        process.cwd();

      let stdout = '';
      let stderr = '';

      // Spawn subprocess
      const childProcess: ChildProcess = spawn(command.command, {
        shell: true,
        cwd: workingDirectory,
        env
      });

      // Collect stdout
      if (childProcess.stdout) {
        childProcess.stdout.on('data', (data: Buffer) => {
          const text = data.toString();
          stdout += text;
          getLogger().appendLine(`[subprocess stdout]: ${text}`);
        });
      }

      // Collect stderr
      if (childProcess.stderr) {
        childProcess.stderr.on('data', (data: Buffer) => {
          const text = data.toString();
          stderr += text;
          getLogger().appendLine(`[subprocess stderr]: ${text}`);
        });
      }

      // Handle process completion
      childProcess.on('close', (code: number | null) => {
        const duration_ms = Date.now() - startTime;
        const exitCode = code ?? 0;

        getLogger().appendLine(
          `[Subprocess] Completed with exit code: ${exitCode} for session ${sessionId}`
        );

        resolve({
          stdout: stdout.trim(),
          stderr: stderr.trim(),
          exitCode,
          duration_ms
        });
      });

      // Handle process errors
      childProcess.on('error', (error: Error) => {
        const duration_ms = Date.now() - startTime;

        getLogger().appendLine(`[Subprocess] Process error: ${error.message}`);

        resolve({
          stdout: stdout.trim(),
          stderr: error.message,
          exitCode: 1,
          duration_ms
        });
      });

    } catch (error) {
      const duration_ms = Date.now() - startTime;
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';

      getLogger().appendLine(`[Subprocess] Failed to spawn: ${errorMessage}`);

      resolve({
        stdout: '',
        stderr: errorMessage,
        exitCode: 1,
        duration_ms
      });
    }
  });
}
```

## Stream Handling

The executor attaches data handlers to stdout and stderr streams:

```typescript
childProcess.stdout.on('data', (data: Buffer) => {
  const text = data.toString();
  stdout += text;
  // Log in real-time for debugging
  getLogger().appendLine(`[subprocess stdout]: ${text}`);
});

childProcess.stderr.on('data', (data: Buffer) => {
  const text = data.toString();
  stderr += text;
  getLogger().appendLine(`[subprocess stderr]: ${text}`);
});
```

Output is accumulated throughout execution and returned when process closes.

## Process Completion

Two events indicate process completion:

### Event: `close`

Process has exited and all streams are closed:

```typescript
childProcess.on('close', (code: number | null) => {
  const exitCode = code ?? 0; // null treated as success
  resolve({
    stdout: stdout.trim(),
    stderr: stderr.trim(),
    exitCode,
    duration_ms: Date.now() - startTime
  });
});
```

### Event: `error`

Process failed to spawn or encountered error:

```typescript
childProcess.on('error', (error: Error) => {
  resolve({
    stdout: stdout.trim(),
    stderr: error.message,
    exitCode: 1,
    duration_ms: Date.now() - startTime
  });
});
```

## Empty Command Handling

Some interactions may have empty commands:

```typescript
if (!command.command || command.command.trim() === '') {
  resolve({
    stdout: '',
    stderr: '',
    exitCode: 0,
    duration_ms: 0
  });
  return;
}
```

## Error Handling

The executor handles errors at multiple levels:

1. **Validation errors**: Command is Claude command (should use ClaudeExecutor)
2. **Spawn errors**: Failed to spawn process (bad command, permissions, etc.)
3. **Process errors**: Process encountered error during execution
4. **Stream errors**: stdout/stderr stream errors (rare)

All errors are caught and converted to CommandResult with exitCode 1.

## Cleanup

```typescript
cleanupSession(sessionId: string): void {
  // Note: No persistent resources to clean up
  // Each execute() creates and manages its own child process
  getLogger().appendLine(`[Subprocess] Cleaned up session ${sessionId}`);
}
```

Unlike TerminalExecutor, SubprocessExecutor doesn't maintain long-lived resources per session. Each command spawns a new child process that is managed within the execute() method.

## Logging

The executor logs to the extension output channel:

```typescript
getLogger().appendLine(`[Subprocess] Executing command for session ${sessionId}`);
getLogger().appendLine(`[Subprocess] Command details: ${JSON.stringify(command, null, 2)}`);
getLogger().appendLine(`[Subprocess] Executing: ${command.command}`);
getLogger().appendLine(`[Subprocess] Working directory: ${workingDirectory}`);
getLogger().appendLine(`[subprocess stdout]: ${text}`);
getLogger().appendLine(`[subprocess stderr]: ${text}`);
getLogger().appendLine(`[Subprocess] Completed with exit code: ${exitCode}`);
```

## Process Management

### Spawning

```typescript
const childProcess = spawn(command, {
  shell: true,      // Run in shell (allows pipes, redirects, etc.)
  cwd: workingDirectory,
  env: env
});
```

The `shell: true` option is important - it allows commands like:
- `ls -la | grep foo`
- `echo "test" > output.txt`
- `cd /tmp && pwd`

### Signals and Termination

Currently, the executor does NOT handle process termination explicitly. If needed, could add:

```typescript
// Store child process reference
private processes = new Map<string, ChildProcess>();

// In execute():
this.processes.set(interactionId, childProcess);

// In cleanup:
cleanupSession(sessionId: string): void {
  // Kill any running processes for this session
  for (const [interactionId, process] of this.processes) {
    if (interactionId.startsWith(sessionId)) {
      process.kill('SIGTERM');
      this.processes.delete(interactionId);
    }
  }
}
```

This is not currently implemented but could be added if needed.

## Comparison with TerminalExecutor

| Feature | SubprocessExecutor | TerminalExecutor |
|---|---|---|
| Execution Mode | Agentic (background) | Manual (visible) |
| Visibility | No terminal | User sees terminal |
| Command Types | Shell commands only | Shell + Claude CLI |
| Output Capture | Direct stream capture | Via terminal manager |
| Working Directory | Command metadata | Terminal's working directory |
| User Interaction | None (fully automated) | User can interact with terminal |

## Comparison with ClaudeExecutor

| Feature | SubprocessExecutor | ClaudeExecutor |
|---|---|---|
| Command Type | Shell commands | Claude commands |
| Execution | Node.js child_process | Claude Agent SDK |
| Output | stdout/stderr | SDK event stream |
| Event Streaming | None | Real-time to server |
| Conversation Tracking | N/A | Conversation IDs |

## Dependencies

- `child_process` (Node.js built-in) - Process spawning
- `getLogger()` - Extension output channel logging
- `SessionUtilities` - API URL and auth token (optional)
