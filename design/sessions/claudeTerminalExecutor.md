# Claude Terminal Executor

Executes Claude CLI commands in visible VSCode terminal for manual mode. Maps to `src/sessions/executors/claudeTerminal.ts`.

## Purpose

The Claude Terminal Executor runs Claude CLI commands in a user-visible VSCode terminal. It builds the complete CLI invocation, sets up hook callbacks, and forwards results to the server. This executor is used exclusively for manual execution mode with Claude commands where users can see Claude's output and interact with the terminal.

## Core Responsibilities

1. Execute Claude CLI commands in VSCode terminal (visible to user)
2. Build Claude CLI commands with flags, resume options, and prompt piping
3. Set up local callback server for Claude hook events
4. Forward hook events to backend server
5. Wait for hook callback (not terminal completion)
6. Return structured command results

## What This Module Does NOT Do

- Does NOT handle shell commands (TerminalExecutor handles that)
- Does NOT run commands in background (ClaudeExecutor handles that)
- Does NOT use Claude Agent SDK (ClaudeExecutor handles that)
- Does NOT manage Phoenix channel events (SessionManager handles that)
- Does NOT select which executor to use (SessionOrchestrator handles that)
- Does NOT poll for next commands (event-driven via SessionOrchestrator)

## Architecture

### Execution Flow

```
ClaudeTerminalExecutor.execute()
        ↓
Extract prompt from command.metadata
        ↓
Build Claude CLI command string
  - Add --resume flag if conversation exists
  - Build flags from options
  - Create temp file for prompt
  - Pipe prompt: cat prompt.tmp | claude
        ↓
getCallbackServer(sessionId)
        ↓
Register callback handler
  - Callback server forwards to server API
  - POST /api/sessions/:id/events
        ↓
Set CODE_MY_SPEC_CALLBACK_URL env var
        ↓
sessionTerminalManager.executeCommand()
        ↓
VSCode Terminal executes: cat prompt.tmp | claude --resume xyz
        ↓
Claude CLI runs, calls hooks
        ↓
SessionStart hook → POST to server (update conversation ID)
SessionEnd hook → POST to callback server
        ↓
Callback server forwards to server API
        ↓
Promise resolves, return result
```

## Public Interface

```typescript
export class ClaudeTerminalExecutor {
  constructor(sessionUtils?: SessionUtilities)

  // Execute a Claude CLI command in VSCode terminal
  async execute(
    sessionId: string,
    interaction: Interaction,
    session: Session
  ): Promise<CommandResult>

  // Clean up Claude terminal executor resources
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

ClaudeTerminalExecutor should only handle Claude commands:

```typescript
private validateClaudeCommand(command: Command): void {
  const isClaudeCommand = command.command === 'claude' || command.metadata?.prompt;

  if (!isClaudeCommand) {
    throw new Error(
      'ClaudeTerminalExecutor can only execute Claude commands. Use TerminalExecutor for shell commands.'
    );
  }
}
```

## Claude CLI Command Building

```typescript
private async buildClaudeCommand(
  session: Session,
  command: Command,
  apiUrl: string,
  apiToken?: string
): Promise<{ command: string, env: Record<string, string> }> {
  // Extract prompt and options from metadata
  const prompt = ClaudeContext.extractPrompt(command.metadata);
  const options = ClaudeContext.extractOptions(command.metadata);

  // Add resume flag if conversation exists
  if (session.external_conversation_id && !options.resume) {
    options.resume = session.external_conversation_id;
  }

  // Build Claude CLI command (without prompt, will be piped)
  const claudeCommand = ClaudeContext.buildCommandString('', options);

  // Create temp file for prompt
  const tempFilePath = await this.commandProcessor.createTempFile(
    session.id,
    prompt
  );

  // Build environment variables
  const env = {
    CODE_MY_SPEC_API_URL: apiUrl,
    CODE_MY_SPEC_API_TOKEN: apiToken || '',
    CODE_MY_SPEC_SESSION_ID: String(session.id)
  };

  // Pipe prompt into claude command
  return {
    command: `cat "${tempFilePath}" | ${claudeCommand}`,
    env
  };
}
```

### Resume Functionality

If `session.external_conversation_id` exists, the `--resume` flag is added automatically:

```bash
cat prompt.tmp | claude --resume conv_abc123xyz
```

This allows continuing previous conversations across multiple interactions.

## Callback Server Integration

A local callback server receives hook events and forwards them to the backend.

```typescript
private async setupClaudeCallback(
  sessionId: string,
  interactionId: string
): Promise<{ callbackUrl: string, promise: Promise<any> }> {
  // Get or create callback server for this session
  const callbackServer = await getCallbackServer(sessionId);

  // Get callback URL for this interaction
  const callbackUrl = callbackServer.getCallbackUrl(interactionId);

  // Set up promise that resolves when callback received
  // Note: Callback server handles forwarding to backend automatically
  const callbackPromise = new Promise((resolve) => {
    callbackServer.onCommandComplete(interactionId, resolve);
  });

  return { callbackUrl, callbackPromise };
}
```

**Separation of Concerns:**
- **ClaudeTerminalExecutor**: Sets up callback, waits for completion signal
- **CallbackServer**: Receives hooks, wraps with context, forwards to server
- **Server**: Processes events, makes decisions, broadcasts Phoenix events

The executor does NOT forward events - it just waits for the callback server to signal that the hook was received.

## Environment Variables

The executor sets these environment variables for Claude CLI:

```typescript
const env = {
  // API credentials (for hooks to call back)
  CODE_MY_SPEC_API_URL: apiUrl,
  CODE_MY_SPEC_API_TOKEN: apiToken,
  CODE_MY_SPEC_SESSION_ID: String(session.id),

  // Callback URL (for SessionEnd hook)
  CODE_MY_SPEC_CALLBACK_URL: callbackUrl,

  // User-defined env vars from command metadata
  ...command.metadata?.env_vars
};
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

  // Normalize command format
  const command: Command = typeof interaction.command === 'string'
    ? { module: 'unknown', command: interaction.command, timestamp: new Date().toISOString() }
    : interaction.command;

  // Handle empty commands
  if (!command.command || command.command.trim() === '') {
    return { stdout: '', stderr: '', exitCode: 0, duration_ms: 0 };
  }

  // Validate this is a Claude command
  this.validateClaudeCommand(command);

  // Get API credentials
  const apiUrl = this.sessionUtils?.getApiUrl();
  const apiToken = this.sessionUtils ? await this.sessionUtils.getAuthToken() : null;

  // Build Claude CLI command
  const processedCommand = await this.buildClaudeCommand(
    session,
    command,
    apiUrl,
    apiToken || undefined
  );

  // Set up callback server
  const callback = await this.setupClaudeCallback(sessionId, interaction.id);
  processedCommand.env.CODE_MY_SPEC_CALLBACK_URL = callback.callbackUrl;

  // Pass API credentials to callback server (via env, it will use them when forwarding)
  processedCommand.env.CODE_MY_SPEC_API_URL = apiUrl;
  processedCommand.env.CODE_MY_SPEC_API_TOKEN = apiToken || '';
  processedCommand.env.CODE_MY_SPEC_SESSION_ID = String(sessionId);

  // Execute in terminal
  const executePromise = sessionTerminalManager.executeCommand(
    sessionId,
    interaction.id,
    { command: processedCommand.command },
    processedCommand.env
  );

  // Wait for both terminal AND callback
  // Callback server will forward hooks to server with context
  await Promise.all([executePromise, callback.promise]);

  // Get result (terminal may not have meaningful output)
  const result = await executePromise;
  return result;
}
```

## Why Wait for Both Terminal and Callback?

The executor waits for both `executePromise` and `callback.promise`:

- **Terminal completion**: Claude CLI process exits
- **Callback arrival**: SessionEnd hook POSTs to callback server

We need both because:
1. Terminal may complete before hook fires (race condition)
2. Hook may fire before terminal fully exits
3. We want to ensure both cleanup and result capture happen

```typescript
await Promise.all([executePromise, callback.promise]);
// Returns when BOTH terminal exits AND hook received
```

**Important Timing:**
```
execute() returns when:
  1. ✓ Terminal process exits
  2. ✓ Hook received by callback server

Later (asynchronously):
  3. Callback server forwards to backend
  4. Server processes event
  5. Server emits interaction:completed Phoenix event
  6. Client receives event → triggers next command (if autoPlay)
```

The executor returns "locally done" (terminal + hook received). The server processes the event asynchronously and notifies via Phoenix when it's "globally done" (processed, state updated, ready for next command).

## Empty Command Handling

Some interactions may have empty commands (initialization steps). These are treated as no-ops:

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
  // Clean up CommandProcessor temp files
  this.commandProcessor.cleanupSession(sessionId);

  // Clean up terminal manager
  sessionTerminalManager.cleanupSession(sessionId);

  // Clean up callback server
  await cleanupCallbackServer(sessionId);

  getLogger().appendLine(`[ClaudeTerminal] Cleaned up session ${sessionId}`);
}
```

All three resources must be cleaned up:
- Temp files (prompts)
- Terminals (VSCode)
- Callback servers (HTTP)

## Error Handling

- If command fails, return non-zero exit code in result
- If validation fails (non-Claude command passed in), throw error
- If callback server setup fails, throw error (cannot proceed without hooks)
- If hook forwarding fails, log but don't fail the command
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
- Capturing output (Claude's stdout)
- Waiting for completion

## Command Processor Dependency

ClaudeTerminalExecutor uses CommandProcessor for:
- Creating temp files for Claude prompts
- Managing temp file lifecycle
- Building environment variables

```typescript
this.commandProcessor = new CommandProcessor();
```

## Callback Server Dependency

ClaudeTerminalExecutor uses CallbackServer for:
- Receiving SessionEnd hook POSTs
- Forwarding hooks to backend
- Promise-based waiting mechanism

```typescript
const callbackServer = await getCallbackServer(sessionId);
```

## Logging

```typescript
getLogger().appendLine(`[ClaudeTerminal] Executing Claude CLI command for session ${sessionId}`);
getLogger().appendLine(`[ClaudeTerminal] Prompt length: ${prompt.length} characters`);
getLogger().appendLine(`[ClaudeTerminal] Resume conversation: ${session.external_conversation_id || 'none'}`);
getLogger().appendLine(`[ClaudeTerminal] Callback URL: ${callbackUrl}`);
getLogger().appendLine(`[ClaudeTerminal] Waiting for hook callback...`);
getLogger().appendLine(`[ClaudeTerminal] Hook callback received`);
getLogger().appendLine(`[ClaudeTerminal] Command completed`);
```

## Comparison with TerminalExecutor

| Feature | ClaudeTerminalExecutor | TerminalExecutor |
|---|---|---|
| Command Type | Claude CLI commands only | Shell commands only |
| CLI Building | Full Claude CLI with flags | Not needed |
| Callback Server | Required for hooks | Not used |
| Temp Files | Prompt piped from temp file | Not used |
| Wait For | Hook callback | Terminal completion |
| Resume Support | --resume flag support | N/A |
| CommandProcessor | Used (temp files) | Not used |

## Comparison with ClaudeExecutor

| Feature | ClaudeTerminalExecutor | ClaudeExecutor |
|---|---|---|
| Execution Mode | Manual (visible) | Agentic (background) |
| Claude Interface | Claude CLI | Claude Agent SDK |
| Visibility | User sees terminal | No terminal |
| Prompt Handling | Temp file + pipe | Direct to SDK |
| Event Streaming | Hooks only | Real-time to server |
| Resume Support | --resume flag | SDK handles internally |
| Callback/Hooks | Callback server | SDK events |

## Hook Configuration

For this executor to work, Claude CLI hooks must be configured in `~/.config/claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": "/path/to/hooks/update-session.sh",
    "SessionEnd": "/path/to/hooks/submit-result.sh"
  }
}
```

The hooks use environment variables set by this executor:
- `CODE_MY_SPEC_CALLBACK_URL` - Where to POST completion
- `CODE_MY_SPEC_API_URL` - Backend API for SessionStart hook
- `CODE_MY_SPEC_API_TOKEN` - Auth for backend API
- `CODE_MY_SPEC_SESSION_ID` - Current session ID

## Dependencies

- `sessionTerminalManager` - VSCode terminal operations
- `CommandProcessor` - Temp file and env var management
- `getCallbackServer()` / `cleanupCallbackServer()` - Callback server lifecycle
- `ClaudeContext` - Claude CLI command building utilities
- `SessionUtilities` - API URL and auth token access
- `getLogger()` - Extension output channel logging
