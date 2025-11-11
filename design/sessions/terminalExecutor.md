# Terminal Executor

Executes shell commands in visible VSCode terminal for manual mode. Maps to `src/sessions/executors/terminal.ts`.

## Purpose

The Terminal Executor runs shell commands in a user-visible VSCode terminal. This executor is used exclusively for manual execution mode with shell commands where users can see and interact with the terminal.

## Core Responsibilities

1. Execute shell commands in VSCode terminal (visible to user)
2. Wait for terminal command completion
3. Return structured command results

## What This Module Does NOT Do

- Does NOT handle Claude commands (ClaudeTerminalExecutor handles that)
- Does NOT run commands in background (SubprocessExecutor handles that)
- Does NOT use Claude Agent SDK (ClaudeExecutor handles that)
- Does NOT manage Phoenix channel events (SessionManager handles that)
- Does NOT select which executor to use (SessionOrchestrator handles that)
- Does NOT poll for next commands (event-driven via SessionOrchestrator)

## Architecture

### Execution Flow

```
TerminalExecutor.execute()
        ↓
Validate command is shell command (not Claude)
        ↓
Build environment variables
        ↓
sessionTerminalManager.executeCommand()
        ↓
VSCode Terminal (visible)
        ↓
Wait for terminal completion
        ↓
Return { stdout, stderr, exitCode, duration_ms }
```

## Public Interface

```typescript
export class TerminalExecutor {
  constructor(sessionUtils?: SessionUtilities)

  // Execute a shell command in VSCode terminal
  async execute(
    sessionId: string,
    interaction: Interaction,
    session: Session
  ): Promise<CommandResult>

  // Clean up terminal executor resources
  async cleanupSession(sessionId: string): Promise<void>
}

interface SessionUtilities {
  getApiUrl(): string;
  getAuthToken(): Promise<string | null>;
}
```

## Command Result Format

```typescript
interface CommandResult {
  stdout: string;
  stderr: string;
  exitCode: number;
  duration_ms: number;
}
```

## Command Validation

TerminalExecutor should only handle shell commands, not Claude commands:

```typescript
private validateShellCommand(command: Command): void {
  const isClaudeCommand = command.command === 'claude' || command.metadata?.prompt;

  if (isClaudeCommand) {
    throw new Error(
      'TerminalExecutor cannot execute Claude commands. Use ClaudeTerminalExecutor for manual Claude execution.'
    );
  }
}
```

## Environment Variables

The executor sets minimal environment variables for shell commands:

```typescript
const env = {
  // API credentials (in case hooks are configured)
  CODE_MY_SPEC_API_URL: apiUrl,
  CODE_MY_SPEC_API_TOKEN: apiToken || '',
  CODE_MY_SPEC_SESSION_ID: String(session.id),

  // User-defined env vars from command metadata
  ...command.metadata?.env_vars
};
```

Note: Claude-specific env vars (like `CODE_MY_SPEC_CALLBACK_URL`) are NOT set since this executor doesn't handle Claude commands.

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

  // Normalize command format
  const command: Command = typeof interaction.command === 'string'
    ? { module: 'unknown', command: interaction.command, timestamp: new Date().toISOString() }
    : interaction.command;

  // Handle empty commands
  if (!command.command || command.command.trim() === '') {
    return { stdout: '', stderr: '', exitCode: 0, duration_ms: 0 };
  }

  // Validate this is a shell command
  this.validateShellCommand(command);

  // Get API credentials
  const apiUrl = this.sessionUtils?.getApiUrl();
  const apiToken = this.sessionUtils ? await this.sessionUtils.getAuthToken() : null;

  // Build environment variables
  const env = {
    CODE_MY_SPEC_API_URL: apiUrl || '',
    CODE_MY_SPEC_API_TOKEN: apiToken || '',
    CODE_MY_SPEC_SESSION_ID: String(session.id),
    ...command.metadata?.env_vars
  };

  // Execute in terminal
  const result = await sessionTerminalManager.executeCommand(
    sessionId,
    interaction.id,
    { command: command.command },
    env
  );

  return result;
}
```

## Empty Command Handling

Some interactions may have empty commands (initialization steps, state transitions). These are treated as no-ops:

```typescript
if (!command.command || command.command.trim() === '') {
  return {
    stdout: '',
    stderr: '',
    exitCode: 0,
    duration_ms: 0
  };
}
```

## Cleanup

```typescript
async cleanupSession(sessionId: string): Promise<void> {
  // Clean up terminal manager
  sessionTerminalManager.cleanupSession(sessionId);

  getLogger().appendLine(`[Terminal] Cleaned up session ${sessionId}`);
}
```

Note: No CommandProcessor cleanup needed since shell commands don't use temp files. No callback server cleanup needed since shell commands don't use hooks.

## Error Handling

- If command fails, return non-zero exit code in result
- If validation fails (Claude command passed in), throw error
- If terminal execution throws, let error propagate to orchestrator

## Integration with VSCode Terminal Manager

The executor delegates actual terminal operations to `sessionTerminalManager`:

```typescript
sessionTerminalManager.executeCommand(
  sessionId: string,
  interactionId: string,
  command: { command: string },
  env?: Record<string, string>
): Promise<CommandResult>
```

The terminal manager handles:
- Creating/reusing terminals per session
- Sending commands to terminal
- Capturing output
- Waiting for completion

## Logging

```typescript
getLogger().appendLine(`[Terminal] Executing shell command for session ${sessionId}`);
getLogger().appendLine(`[Terminal] Command: ${command.command}`);
getLogger().appendLine(`[Terminal] Environment variables: ${Object.keys(env).join(', ')}`);
getLogger().appendLine(`[Terminal] Command completed with exit code: ${result.exitCode}`);
```

## Comparison with ClaudeTerminalExecutor

| Feature | TerminalExecutor | ClaudeTerminalExecutor |
|---|---|---|
| Command Type | Shell commands only | Claude CLI commands only |
| CLI Building | Not needed | Full Claude CLI with flags |
| Callback Server | Not used | Required for hooks |
| Temp Files | Not used | Prompt piped from temp file |
| Wait For | Terminal completion | Hook callback |
| Resume Support | N/A | --resume flag support |

## Comparison with SubprocessExecutor

| Feature | TerminalExecutor | SubprocessExecutor |
|---|---|---|
| Execution Mode | Manual (visible) | Agentic (background) |
| Visibility | User sees terminal | No terminal |
| Command Types | Shell commands only | Shell commands only |
| Output Capture | Via terminal manager | Direct stream capture |
| User Interaction | User can interact | None (fully automated) |

## Dependencies

- `sessionTerminalManager` - VSCode terminal operations
- `getLogger()` - Extension output channel logging
- `SessionUtilities` - API URL and auth token (optional)
