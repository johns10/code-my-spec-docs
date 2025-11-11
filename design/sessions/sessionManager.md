# Session Manager

Manages session state and Phoenix channel event subscriptions. Maps to `src/sessionManager.ts`.

## Purpose

The Session Manager coordinates session state across HTTP API and Phoenix channels. It subscribes to real-time events from the backend, maintains local session state, and provides event handlers for the SessionOrchestrator to react to session changes. This module bridges Phoenix events with the extension's execution logic.

## Core Responsibilities

1. Initialize Phoenix channel connection
2. Subscribe to session-related Phoenix events
3. Maintain local session state cache
4. Provide event subscription interface for orchestrator
5. Notify subscribers of session state changes
6. Handle Phoenix connection lifecycle

## What This Module Does NOT Do

- Does NOT execute commands (SessionOrchestrator handles that)
- Does NOT manage terminal/subprocess lifecycle (executors handle that)
- Does NOT make HTTP API calls (CodeMySpecClient handles that)
- Does NOT build commands (CommandProcessor and executors handle that)
- Does NOT decide when to execute next command (SessionOrchestrator handles that)

## Architecture

### Event Flow

```
Backend Phoenix Channel
        ↓
PhoenixChannelClient.on('interaction:completed', ...)
        ↓
SessionManager.handleInteractionCompleted()
        ↓
Update sessionStateManager
        ↓
Notify subscribers (SessionOrchestrator)
        ↓
SessionOrchestrator.handleInteractionCompleted()
        ↓
Execute next command (if autoPlay enabled)
```

## Public Interface

```typescript
export class SessionManager {
  constructor(authProvider: AuthenticationProvider)

  // Initialize Phoenix connection and subscriptions
  async initialize(): Promise<void>

  // Subscribe to interaction completion events
  subscribeToInteractionEvents(callback: (event: InteractionEvent) => void): void

  // Subscribe to session completion events
  onSessionComplete(callback: (session: Session) => void): void

  // Refresh sessions from HTTP API (fallback)
  async refreshSessions(): Promise<void>

  // Get current sessions
  getSessions(): Session[]

  // Clean up resources
  async cleanup(): Promise<void>
}
```

## Phoenix Events Subscribed

### Event: `session_created`

Fired when a new session is created.

**Payload:**
```typescript
{
  session: Session
}
```

**Handler:**
```typescript
private handleSessionCreated(payload: any): void {
  sessionStateManager.updateSession(payload.session);
  // Notify subscribers if needed
}
```

### Event: `session_updated`

Fired when session metadata changes (status, name, etc.).

**Payload:**
```typescript
{
  session: Session
}
```

**Handler:**
```typescript
private handleSessionUpdated(payload: any): void {
  sessionStateManager.updateSession(payload.session);

  // Check if session completed
  if (payload.session.status === 'complete' || payload.session.status === 'failed') {
    this.handleSessionCompleted(payload.session);
  }
}
```

### Event: `session_deleted`

Fired when a session is deleted.

**Payload:**
```typescript
{
  session_id: string
}
```

**Handler:**
```typescript
private handleSessionDeleted(payload: any): void {
  sessionStateManager.removeSession(payload.session_id);
}
```

### Event: `interaction:completed` (NEW)

Fired when an interaction completes (result submitted and processed).

**Payload:**
```typescript
{
  session_id: string;
  interaction_id: string;
  interaction: Interaction;
  status: 'ok' | 'error';
}
```

**Handler:**
```typescript
private handleInteractionCompleted(payload: InteractionCompletedEvent): void {
  // Update local state
  const session = sessionStateManager.getSession(payload.session_id);
  if (session) {
    // Update interaction with result
    const interaction = session.interactions?.find(i => i.id === payload.interaction_id);
    if (interaction) {
      interaction.result = payload.interaction.result;
    }
    sessionStateManager.updateSession(session);
  }

  // Notify orchestrator
  this.notifyInteractionSubscribers(payload);
}
```

### Event: `interaction:started` (NEW)

Fired when a new interaction is ready to execute.

**Payload:**
```typescript
{
  session_id: string;
  interaction_id: string;
  interaction: Interaction;
}
```

**Handler:**
```typescript
private handleInteractionStarted(payload: InteractionStartedEvent): void {
  // Update local state
  const session = sessionStateManager.getSession(payload.session_id);
  if (session) {
    // Add or update interaction
    if (!session.interactions) {
      session.interactions = [];
    }
    const existingIndex = session.interactions.findIndex(i => i.id === payload.interaction_id);
    if (existingIndex >= 0) {
      session.interactions[existingIndex] = payload.interaction;
    } else {
      session.interactions.push(payload.interaction);
    }
    sessionStateManager.updateSession(session);
  }

  // Notify orchestrator
  this.notifyInteractionSubscribers(payload);
}
```

## Event Subscription Interface

### Interaction Events

```typescript
private interactionEventSubscribers: Array<(event: InteractionEvent) => void> = [];

subscribeToInteractionEvents(callback: (event: InteractionEvent) => void): void {
  this.interactionEventSubscribers.push(callback);
}

private notifyInteractionSubscribers(event: InteractionEvent): void {
  for (const callback of this.interactionEventSubscribers) {
    try {
      callback(event);
    } catch (error) {
      getLogger().appendLine(`[SessionManager] Error in interaction event subscriber: ${error}`);
    }
  }
}
```

### Session Completion

```typescript
private sessionCompleteCallback?: (session: Session) => void;

onSessionComplete(callback: (session: Session) => void): void {
  this.sessionCompleteCallback = callback;
}

private handleSessionCompleted(session: Session): void {
  // Remove from active sessions
  sessionStateManager.removeSession(session.id);

  // Notify callback
  if (this.sessionCompleteCallback) {
    this.sessionCompleteCallback(session);
  }
}
```

## Initialization

```typescript
async initialize(): Promise<void> {
  try {
    // Get user ID and auth token
    const userId = await this.authProvider.getUserId();
    const authToken = await this.authProvider.getAccessToken();
    const serverUrl = getServerUrl();

    if (!userId || !authToken) {
      throw new Error('Not authenticated');
    }

    // Initialize Phoenix channel
    await initializePhoenixChannel(serverUrl, userId, authToken);

    // Subscribe to events
    this.subscribeToPhoenixEvents();

    // Do initial HTTP fetch
    await this.refreshSessions();

  } catch (error) {
    getLogger().appendLine(`[SessionManager] Failed to initialize: ${error}`);
    // Don't throw - Phoenix is optional, fall back to HTTP polling
  }
}
```

## Phoenix Event Subscription Setup

```typescript
private subscribeToPhoenixEvents(): void {
  const phoenixClient = getPhoenixClient();
  if (!phoenixClient) {
    return;
  }

  // Existing events
  phoenixClient.on('session_created', (payload) => this.handleSessionCreated(payload));
  phoenixClient.on('session_updated', (payload) => this.handleSessionUpdated(payload));
  phoenixClient.on('session_deleted', (payload) => this.handleSessionDeleted(payload));

  // New events for command execution
  phoenixClient.on('interaction:completed', (payload) => this.handleInteractionCompleted(payload));
  phoenixClient.on('interaction:started', (payload) => this.handleInteractionStarted(payload));
}
```

## Session State Management

The SessionManager uses `sessionStateManager` (a global singleton) for local state:

```typescript
import { sessionStateManager } from './sessionState';

// Update session
sessionStateManager.updateSession(session);

// Get session
const session = sessionStateManager.getSession(sessionId);

// Remove session
sessionStateManager.removeSession(sessionId);

// Get all sessions
const sessions = sessionStateManager.getAllSessions();
```

The state manager handles:
- Deduplication (same session updated multiple times)
- Subscriber notifications
- Persistence (if needed)

## HTTP Fallback

If Phoenix connection is unavailable, SessionManager falls back to HTTP polling:

```typescript
async refreshSessions(): Promise<void> {
  try {
    const client = await CodeMySpecClient.createAuthenticated(this.authProvider);
    const sessions = await client.sessions.listSessions();

    // Update local state
    for (const session of sessions) {
      sessionStateManager.updateSession(session);
    }
  } catch (error) {
    getLogger().appendLine(`[SessionManager] Failed to refresh sessions: ${error}`);
  }
}
```

This can be called:
- On initialization
- Periodically (if Phoenix disconnected)
- Manually by user

## Orchestrator Integration

The SessionOrchestrator registers event handlers during initialization:

```typescript
// In SessionOrchestrator.initialize()
this.sessionManager.subscribeToInteractionEvents((event) => {
  if (event.type === 'interaction:completed') {
    this.handleInteractionCompleted(event);
  } else if (event.type === 'interaction:started') {
    this.handleInteractionStarted(event);
  }
});

this.sessionManager.onSessionComplete((session) => {
  this.handleSessionCompleted(session);
});
```

## Event Types

```typescript
type InteractionEventType = 'interaction:started' | 'interaction:completed';

interface InteractionEvent {
  type: InteractionEventType;
  session_id: string;
  interaction_id: string;
  interaction?: Interaction;
  status?: 'ok' | 'error';
}

interface InteractionCompletedEvent extends InteractionEvent {
  type: 'interaction:completed';
  interaction: Interaction;
  status: 'ok' | 'error';
}

interface InteractionStartedEvent extends InteractionEvent {
  type: 'interaction:started';
  interaction: Interaction;
}
```

## Phoenix Connection Lifecycle

### Connection

```typescript
// Phoenix client handles connection automatically
await initializePhoenixChannel(serverUrl, userId, authToken);
```

### Reconnection

The Phoenix client handles reconnection automatically:
- On disconnect, attempts reconnect after 5 seconds
- Exponential backoff for repeated failures
- Automatic channel rejoin after reconnect

### Disconnection

On disconnect, SessionManager:
- Continues to work (local state cached)
- Falls back to HTTP for new data
- Automatically reconnects via Phoenix client

## Cleanup

```typescript
async cleanup(): Promise<void> {
  // Clear subscribers
  this.interactionEventSubscribers = [];
  this.sessionCompleteCallback = undefined;

  // Phoenix client cleanup happens automatically
  // (handled by phoenix module)
}
```

Called when extension deactivates.

## Error Handling

### Phoenix Event Errors

If a Phoenix event handler throws:
- Error is logged
- Other subscribers still get notified
- Connection remains active

### Initialization Errors

If initialization fails:
- Error is logged
- Extension continues to work via HTTP only
- User may see delayed updates (no real-time events)

### Subscription Errors

If subscriber callback throws:
- Error is logged
- Other subscribers still get notified
- Event processing continues

## Logging

```typescript
getLogger().appendLine(`[SessionManager] Initializing with user ${userId}`);
getLogger().appendLine(`[SessionManager] Phoenix connected`);
getLogger().appendLine(`[SessionManager] Received session_created event`);
getLogger().appendLine(`[SessionManager] Received interaction:completed event`);
getLogger().appendLine(`[SessionManager] Session ${sessionId} completed`);
getLogger().appendLine(`[SessionManager] Notifying ${count} interaction subscribers`);
```

## Migration from Old Design

### Removed

- Polling logic for session updates
- Manual refresh timers

### Added

- `subscribeToInteractionEvents()` - New event subscription interface
- `interaction:completed` event handler
- `interaction:started` event handler
- Auto-play coordination via events

### Changed

- Session completion now triggers via Phoenix event, not polling
- Interaction updates are real-time, not on-demand

## Event-Driven Execution Example

```
1. User starts session (manual mode, autoPlay enabled)
        ↓
2. SessionOrchestrator.executeNext()
        ↓
3. Executor runs command
        ↓
4. Executor.submitResult()
        ↓
5. Backend processes result
        ↓
6. Backend emits 'interaction:completed' via Phoenix
        ↓
7. SessionManager receives event
        ↓
8. SessionManager notifies SessionOrchestrator
        ↓
9. SessionOrchestrator checks autoPlay = true
        ↓
10. SessionOrchestrator.executeNext() (loop continues)
```

## Dependencies

- `PhoenixChannelClient` - Phoenix channel connection and subscriptions
- `sessionStateManager` - Local session state cache
- `CodeMySpecClient` - HTTP API fallback
- `AuthenticationProvider` - User ID and auth token
- `getLogger()` - Extension output channel logging
