# Session Orchestrator

Event-driven coordinator for session execution. Maps to `src/sessions.ts`.

## Purpose

The Session Orchestrator coordinates command execution across multiple executors using an event-driven architecture. It subscribes to Phoenix channel events, manages auto-play state, selects appropriate executors, and handles parallel session coordination. This module replaces the polling/loop-based architecture with reactive event handling.

## Core Responsibilities

1. Subscribe to Phoenix channel events and react to session state changes
2. Manage auto-play state per session
3. Select appropriate executor based on execution mode and command type
4. Coordinate parallel child session execution
5. Submit command results to server
6. Track external conversation IDs for Claude sessions

## What This Module Does NOT Do

- Does NOT poll for next commands (reacts to events instead)
- Does NOT run auto-execution loops (removed `autoLoop()`)
- Does NOT manage terminal/subprocess lifecycle (executors handle this)
- Does NOT build commands (executors and CommandProcessor handle this)
- Does NOT handle Phoenix channel connection (SessionManager handles this)

## Architecture

### Event-Driven Execution Flow

```
Phoenix Event → SessionManager → SessionOrchestrator.handleEvent()
                                          ↓
                                  pickExecutor()
                                          ↓
                                  executor.execute()
                                          ↓
                                  submitResult()
                                          ↓
                                  Server (HTTP POST)
                                          ↓
                                  Phoenix broadcasts interaction:completed
                                          ↓
                                  (loop if autoPlay enabled)
```

## Public Interface

```typescript
export class SessionOrchestrator {
  constructor(authProvider: AuthenticationProvider)

  // Initialize and set up event subscriptions
  async initialize(): Promise<void>

  // Register callback for session completion
  onSessionComplete(callback: (session: Session) => void): void

  // Start a session (marks as active, doesn't loop)
  async start(session: Session): Promise<Session>

  // Execute next command for a session (called by event handler or manually)
  async executeNext(sessionId: string): Promise<{ session: Session, result?: CommandResult }>

  // Enable/disable auto-play for a session
  setAutoPlay(sessionId: string, enabled: boolean): void

  // Get session state
  async getSession(sessionId: string): Promise<Session>

  // Submit command result to server
  async submitResult(sessionId: string, interactionId: string, result: CommandResult): Promise<void>

  // Clean up session resources
  async cleanupSession(sessionId: string): Promise<void>
}

interface Executors {
  terminal: TerminalExecutor;           // Manual mode, shell commands
  claudeTerminal: ClaudeTerminalExecutor; // Manual mode, Claude CLI
  subprocess: SubprocessExecutor;       // Agentic mode, shell commands
  claude: ClaudeExecutor;               // Agentic mode, Claude SDK
}
```

## State Management

```typescript
interface SessionState {
  session: Session;
  autoPlayEnabled: boolean;
  isExecuting: boolean; // Prevents concurrent execution
}

private sessionStates = new Map<string, SessionState>();
```

## Executor Selection Logic

The orchestrator picks the appropriate executor based on execution mode and command type:

```typescript
private pickExecutor(session: Session, command: Command): Executor {
  const isClaudeCommand = command.command === 'claude' || command.metadata?.prompt;

  if (session.execution_mode === 'agentic') {
    // Agentic mode: run in background
    if (isClaudeCommand) {
      return this.executors.claude;        // Claude Agent SDK
    } else {
      return this.executors.subprocess;    // Shell in subprocess
    }
  } else {
    // Manual mode: run in visible terminal
    if (isClaudeCommand) {
      return this.executors.claudeTerminal; // Claude CLI in terminal
    } else {
      return this.executors.terminal;       // Shell in terminal
    }
  }
}
```

### Executor Selection Matrix

| Execution Mode | Command Type | Executor | Behavior |
|---|---|---|---|
| `manual` | Shell | TerminalExecutor | Visible terminal, wait for completion |
| `manual` | Claude | ClaudeTerminalExecutor | Visible terminal, Claude CLI, hook callbacks |
| `agentic` | Shell | SubprocessExecutor | Background subprocess, capture output |
| `agentic` | Claude | ClaudeExecutor | Background SDK execution, stream events |

## Event Handling

### Phoenix Events Subscribed

```typescript
// In initialize()
this.sessionManager.subscribeToInteractionEvents((event) => {
  this.handleInteractionEvent(event);
});
```

### Event: `interaction:completed`

Fired when the server receives and processes a command result.

**Payload:**
```typescript
{
  session_id: string;
  interaction_id: string;
  status: 'ok' | 'error';
}
```

**Handler:**
```typescript
private async handleInteractionCompleted(event: InteractionCompletedEvent): Promise<void> {
  const state = this.sessionStates.get(event.session_id);
  if (!state) return;

  // Mark not executing
  state.isExecuting = false;

  // If auto-play enabled, execute next command
  if (state.autoPlayEnabled) {
    await this.executeNext(event.session_id);
  }
}
```

### Event: `session:completed`

Fired when session reaches terminal state (complete/failed).

**Handler:**
```typescript
private handleSessionCompleted(event: SessionCompletedEvent): void {
  const state = this.sessionStates.get(event.session_id);
  if (!state) return;

  // Disable auto-play
  state.autoPlayEnabled = false;

  // Trigger callback
  if (this.onSessionCompleteCallback) {
    this.onSessionCompleteCallback(state.session);
  }

  // Clean up
  this.cleanupSession(event.session_id);
}
```

## Auto-Play Implementation

Auto-play is now a **client-side toggle**, not a server-side execution mode.

```typescript
setAutoPlay(sessionId: string, enabled: boolean): void {
  const state = this.sessionStates.get(sessionId);
  if (state) {
    state.autoPlayEnabled = enabled;

    // If enabling and not currently executing, kick off execution
    if (enabled && !state.isExecuting) {
      this.executeNext(sessionId);
    }
  }
}
```

### User Experience

- User opens session (manual mode)
- User clicks "Play" button → `setAutoPlay(sessionId, true)`
- Extension executes current command
- Server processes, emits `interaction:completed`
- Extension receives event, executes next command automatically
- User clicks "Pause" button → `setAutoPlay(sessionId, false)`
- Current command completes but next one doesn't start

## Parallel Session Execution

When a command spawns child sessions, the orchestrator coordinates their execution.

```typescript
private async handleParallelSessions(
  interaction: Interaction,
  parentSession: Session
): Promise<void> {
  const childSessionIds = interaction.command.metadata?.child_session_ids || [];

  // Spawn all children in parallel
  const childPromises = childSessionIds.map(childId =>
    this.spawnChildSession(childId, parentSession.execution_mode)
  );

  // Wait for all to complete
  const results = await Promise.allSettled(childPromises);

  // Aggregate results and submit to parent
  await this.submitParallelResults(interaction, results, parentSession);
}
```

### Child Session Behavior

- Child sessions inherit execution mode from parent
- Each child runs independently with its own orchestration
- Parent waits for all children before continuing
- Results are aggregated and submitted as single result to parent interaction

## Command Execution Flow

```typescript
async executeNext(sessionId: string): Promise<{ session: Session, result?: CommandResult }> {
  const state = this.sessionStates.get(sessionId);
  if (!state || state.isExecuting) {
    return { session: state.session };
  }

  // Mark as executing
  state.isExecuting = true;

  // Get next command from server
  const session = await this.getNextCommand(sessionId);
  state.session = session;

  // Check if session complete
  if (session.status === 'complete' || session.status === 'failed') {
    this.handleSessionCompleted({ session_id: sessionId, status: session.status });
    return { session };
  }

  // Get current interaction
  const interaction = this.getCurrentInteraction(session);
  if (!interaction) {
    state.isExecuting = false;
    return { session };
  }

  // Check for parallel sessions
  if (this.isParallelSessionsCommand(interaction.command)) {
    await this.handleParallelSessions(interaction, session);
    return { session };
  }

  // Execute single command
  const executor = this.pickExecutor(session, interaction.command);
  const result = await executor.execute(sessionId, interaction, session);

  // Submit result
  await this.submitResult(sessionId, interaction.id, result);

  // Note: isExecuting will be set to false when interaction:completed event received
  return { session, result };
}
```

## Result Submission

```typescript
async submitResult(sessionId: string, interactionId: string, result: CommandResult): Promise<void> {
  const resultParams: ResultParams = {
    status: result.exitCode === 0 ? 'ok' : 'error',
    stdout: result.stdout,
    stderr: result.stderr || undefined,
    code: result.exitCode.toString(),
    duration_ms: result.duration_ms
  };

  if (result.exitCode !== 0) {
    resultParams.error_message = `Command exited with code ${result.exitCode}`;
  }

  const client = await this.ensureClient();
  await client.sessions.submitResult(sessionId, interactionId, resultParams);

  // Server will process and emit interaction:completed event
  // Event handler will trigger next execution if autoPlay enabled
}
```

## External Conversation ID Management

For Claude sessions, conversation IDs are tracked by the **server**, not the client.

**How It Works:**
1. ClaudeExecutor or ClaudeTerminalExecutor posts events to server
2. Events contain conversation_id in event_data
3. Server extracts conversation_id and updates session.external_conversation_id
4. Server emits `session_updated` Phoenix event
5. Client receives event and updates local state

**Client Does NOT Call API:**
The client does not call `updateExternalConversationId()` directly. The server extracts conversation IDs from events automatically. This follows the "server is smart, client is dumb" philosophy.

**Local State Update:**
```typescript
// In SessionManager, on receiving session_updated Phoenix event
phoenixClient.on('session_updated', (payload) => {
  const state = this.sessionStates.get(payload.session.id);
  if (state) {
    state.session = payload.session;  // Includes updated external_conversation_id
  }
});
```

## Cleanup

```typescript
async cleanupSession(sessionId: string): Promise<void> {
  // Clean up all executors
  await this.executors.terminal.cleanupSession(sessionId);
  await this.executors.claudeTerminal.cleanupSession(sessionId);
  this.executors.subprocess.cleanupSession(sessionId);
  this.executors.claude.cleanupSession(sessionId);

  // Remove from state
  this.sessionStates.delete(sessionId);
}
```

## Migration from Old Architecture

### Removed

- `autoLoop()` method - Replaced by event-driven execution
- `isSessionRunning()` / `stopSession()` - Replaced by `isExecuting` flag
- `activeSessions` map with `isRunning` boolean - Replaced by `sessionStates` with richer state
- Polling behavior - Replaced by Phoenix event subscriptions
- `execution_mode: 'auto'` - Replaced by `autoPlayEnabled` client-side flag

### Added

- `initialize()` method for event subscription setup
- `setAutoPlay()` for user-controlled auto-execution
- `handleInteractionCompleted()` event handler
- `handleSessionCompleted()` event handler
- `isExecuting` flag to prevent concurrent execution

## Error Handling

- If `executeNext()` throws, `isExecuting` is set to false and auto-play continues
- If parallel child sessions fail, parent receives aggregated error result
- If Phoenix connection drops, SessionManager handles reconnection
- If executor throws, error is caught, submitted as failed result, and execution continues

## Dependencies

- `SessionManager` - Phoenix event subscription and session state
- `CodeMySpecClient` - HTTP API calls for getNextCommand, submitResult
- `AuthenticationProvider` - Token management
- `TerminalExecutor` - Manual mode, shell commands
- `ClaudeTerminalExecutor` - Manual mode, Claude CLI commands
- `SubprocessExecutor` - Agentic mode, shell commands
- `ClaudeExecutor` - Agentic mode, Claude SDK commands
