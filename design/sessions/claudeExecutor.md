# Claude Executor

Executes Claude commands using the Agent SDK with real-time event streaming. Maps to `src/sessions/executors/claude.ts`.

## Purpose

The Claude Executor runs Claude commands in agentic mode using the Claude Agent SDK. It streams SDK events to the backend server in real-time, enabling live progress updates and conversation tracking. This executor runs entirely in the background without user-visible terminals.

## Core Responsibilities

1. Execute Claude commands using `@anthropic-ai/claude-agent-sdk`
2. Stream SDK events to backend server in real-time
3. Capture conversation IDs for resume functionality
4. Aggregate final result from event stream
5. Handle SDK errors and convert to CommandResult format

## What This Module Does NOT Do

- Does NOT run commands in terminal (TerminalExecutor handles that)
- Does NOT build Claude CLI commands (uses SDK directly)
- Does NOT manage callback servers (SDK events go to server, not local callback)
- Does NOT handle shell commands (SubprocessExecutor handles that)
- Does NOT create temp files for prompts (passes prompt directly to SDK)

## Architecture

### Execution Flow

```
ClaudeExecutor.execute()
        ↓
Extract prompt from command.metadata
        ↓
Import Claude Agent SDK dynamically
        ↓
Call SDK query() with prompt
        ↓
Stream events:
  - For each event:
    → POST to server API /api/sessions/:id/interactions/:id/events
    → Capture conversation_id if present
    → Accumulate assistant messages
        ↓
Stream completes
        ↓
Return aggregated CommandResult
```

## Public Interface

```typescript
export class ClaudeExecutor {
  constructor(
    updateExternalConversationId?: (sessionId: string, conversationId: string) => Promise<void>
  )

  // Execute a command using Claude Agent SDK
  async execute(
    sessionId: string,
    interaction: Interaction,
    session: Session
  ): Promise<CommandResult>

  // Clean up Claude executor resources
  cleanupSession(sessionId: string): void
}
```

## SDK Integration

### Dynamic Import

The SDK is imported dynamically to avoid loading it at startup:

```typescript
private static async getSDK() {
  const sdk = await import('@anthropic-ai/claude-agent-sdk');
  return sdk;
}
```

### SDK Configuration

```typescript
const options = {
  settingSources: [],  // Disable loading settings from filesystem
  cwd: process.env.CLAUDE_WORKSPACE_PATH || process.cwd(),
  permissionMode: 'bypassPermissions' as const  // Auto-approve all tool uses
};

const stream = query({ prompt, options });
```

## Event Streaming to Backend

For each SDK event, the executor POSTs directly to the backend (no callback server needed):

```typescript
private async streamEventToServer(
  sessionId: string,
  interactionId: string,
  event: any
): Promise<void> {
  const apiUrl = this.sessionUtils?.getApiUrl();
  const apiToken = await this.sessionUtils?.getAuthToken();

  if (!apiUrl || !apiToken) {
    return;
  }

  await fetch(`${apiUrl}/api/sessions/${sessionId}/events`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${apiToken}`
    },
    body: JSON.stringify({
      session_id: sessionId,
      interaction_id: interactionId,
      event_type: event.type,
      event_data: event,
      timestamp: new Date().toISOString()
    })
  });

  // Backend processes event, makes decisions:
  // - Captures conversation_id from events
  // - Logs events for audit trail
  // - On final result event → emits interaction:completed Phoenix event
  // - Updates session state, emits session_updated Phoenix event
}
```

**Why Direct Posting?**
- SDK runs in extension process (not terminal)
- Can reach remote server directly (no localhost limitation)
- No need for callback server bridge

## SDK Event Types

The Claude Agent SDK emits these event types:

### Event: `assistant`

Assistant's response message.

```typescript
{
  type: 'assistant',
  message: {
    role: 'assistant',
    content: [
      { type: 'text', text: 'Here is my response...' }
    ]
  }
}
```

**Handling:**
- Accumulate text blocks for final stdout
- Stream to server for live UI updates
- Log to extension output channel

### Event: `tool_use`

Claude is using a tool (e.g., bash, edit, read).

```typescript
{
  type: 'tool_use',
  tool_name: 'bash',
  tool_input: { command: 'ls -la' }
}
```

**Handling:**
- Stream to server for progress tracking
- Log to extension output channel

### Event: `tool_result`

Result from a tool execution.

```typescript
{
  type: 'tool_result',
  tool_use_id: 'toolu_123',
  content: 'file1.txt\nfile2.txt'
}
```

**Handling:**
- Stream to server
- Log to extension output channel

### Event: `result`

Final result indicating completion or error.

```typescript
{
  type: 'result',
  is_error: false,
  subtype: 'success' | 'error_max_turns' | 'error_during_execution'
}
```

**Handling:**
- Determine exit code based on `is_error`
- Capture error message if present
- End event streaming

### Event: Conversation ID

Conversation IDs may appear in various events:

```typescript
{
  conversation_id: 'conv_abc123xyz',
  // ... other event data
}
```

**Handling:**
- Capture the first conversation ID encountered
- Call `updateExternalConversationId(sessionId, conversationId)`
- Enables resume functionality for future interactions

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

  try {
    // Handle empty commands
    if (!command.command || command.command.trim() === '') {
      return { stdout: '', stderr: '', exitCode: 0, duration_ms: 0 };
    }

    // Extract prompt from metadata
    const prompt = command.metadata?.prompt || command.command;

    // Import SDK
    const { query } = await ClaudeExecutor.getSDK();

    // Configure SDK
    const options = {
      settingSources: [],
      cwd: process.env.CLAUDE_WORKSPACE_PATH || process.cwd(),
      permissionMode: 'bypassPermissions' as const
    };

    // Start streaming
    const stream = query({ prompt, options });

    let stdout = '';
    let stderr = '';
    let hasError = false;
    let conversationIdUpdated = false;

    // Process event stream
    for await (const event of stream) {
      // Stream event to backend
      await this.streamEventToServer(sessionId, interaction.id, event);

      // Capture conversation ID (first occurrence only)
      // Note: Server will extract and update from events, but we track locally too
      if (!conversationIdUpdated) {
        const conversationId = event.conversation_id || event.conversationId;
        if (conversationId) {
          conversationIdUpdated = true;
          // Server extracts this from event_data, no need to call updateExternalConversationId
        }
      }

      // Handle event types
      if (event.type === 'assistant') {
        // Accumulate assistant messages
        for (const block of event.message.content) {
          if (block.type === 'text') {
            stdout += block.text + '\n';
          }
        }
      } else if (event.type === 'result') {
        // Check for errors
        if (event.is_error) {
          hasError = true;
          if (event.subtype === 'error_max_turns') {
            stderr = 'Maximum turns reached';
          } else if (event.subtype === 'error_during_execution') {
            stderr = 'Error during execution';
          }
        }
      }
    }

    const duration_ms = Date.now() - startTime;

    return {
      stdout: stdout.trim(),
      stderr: stderr.trim(),
      exitCode: hasError ? 1 : 0,
      duration_ms
    };

  } catch (error) {
    const duration_ms = Date.now() - startTime;
    const errorMessage = error instanceof Error ? error.message : 'Unknown error';

    return {
      stdout: '',
      stderr: errorMessage,
      exitCode: 1,
      duration_ms
    };
  }
}
```

## Resume Functionality

ClaudeExecutor does NOT handle resume flags - the SDK manages conversation continuity internally.

Conversation IDs are captured in events and sent to server:

```typescript
// SDK event contains conversation_id
{
  type: 'assistant',
  conversation_id: 'conv_abc123',
  message: { ... }
}

// Posted to server in event_data
await streamEventToServer(sessionId, interactionId, event);

// Server extracts conversation_id from event_data
// Server updates session.external_conversation_id
// Server emits session_updated Phoenix event
```

The client doesn't call `updateExternalConversationId()` - the server extracts it from events.

## Empty Command Handling

Some interactions may have empty commands (initialization steps):

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

## Error Handling

### SDK Errors

If the SDK throws an error:

```typescript
catch (error) {
  const errorMessage = error instanceof Error ? error.message : 'Unknown error';
  return {
    stdout: '',
    stderr: errorMessage,
    exitCode: 1,
    duration_ms
  };
}
```

### Event Streaming Errors

If streaming an event to the server fails:
- Log the error
- Continue processing remaining events
- Don't fail the entire execution

```typescript
try {
  await this.streamEventToServer(sessionId, interaction.id, event);
} catch (streamError) {
  getLogger().appendLine(`[ClaudeExecutor] Failed to stream event: ${streamError}`);
  // Continue processing
}
```

## Cleanup

```typescript
cleanupSession(sessionId: string): void {
  // Note: No cleanup needed for Claude Agent SDK
  // SDK manages its own resources
  getLogger().appendLine(`[Claude] Cleaned up session ${sessionId}`);
}
```

The SDK handles its own resource cleanup, so this is a no-op.

## Logging

The executor logs extensively to the extension output channel:

```typescript
getLogger().appendLine(`[Claude] Executing command for session ${sessionId}`);
getLogger().appendLine(`[Claude] Command details: ${JSON.stringify(command, null, 2)}`);
getLogger().appendLine(`[Claude] Executing Claude agent query: ${prompt.substring(0, 100)}...`);
getLogger().appendLine(`[Claude] Calling query with prompt length: ${prompt.length}`);
getLogger().appendLine(`[Claude] Stream created successfully`);
getLogger().appendLine(`[Claude] Captured conversation ID: ${conversationId}`);
getLogger().appendLine(`Claude: ${block.text}`);
getLogger().appendLine(`[Claude] Execution completed for session ${sessionId}`);
```

## Backend Event Processing

When the backend receives streamed events, it should:

1. Update session state (e.g., "Claude is thinking...", "Running bash command...")
2. Store events for audit trail
3. Broadcast live updates via Phoenix channel to connected clients
4. Update interaction state on completion

This enables real-time progress visibility in the UI.

## Comparison with TerminalExecutor

| Feature | ClaudeExecutor | TerminalExecutor |
|---|---|---|
| Execution Mode | Agentic (background) | Manual (visible) |
| SDK vs CLI | Claude Agent SDK | Claude CLI |
| Prompt Handling | Direct to SDK | Temp file + pipe |
| Event Streaming | POST to server API | Hooks → callback server → server |
| Terminal Visibility | No terminal | User sees terminal |
| Resume Support | SDK handles internally | `--resume` flag in CLI |
| Tool Approval | `bypassPermissions` | Prompts user in terminal |

## Dependencies

- `@anthropic-ai/claude-agent-sdk` - Claude Agent SDK (dynamically imported)
- `getLogger()` - Extension output channel logging
- `updateExternalConversationId()` callback - Conversation ID tracking
- Backend API - Event streaming endpoint
