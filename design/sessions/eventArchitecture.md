# Event Architecture

Simplified event-driven architecture for session execution. This document describes the hybrid REST + Phoenix approach.

## Philosophy

**Server is Smart, Client is Dumb**

The client executes commands and reports events. The server processes events, makes decisions, mutates state, and broadcasts updates. The client reacts to server decisions.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                      CLIENT (Extension)                  │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  Executors   │  │   Callback   │  │   Session    │ │
│  │              │  │    Server    │  │   Manager    │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│         │                 │                  │         │
└─────────┼─────────────────┼──────────────────┼─────────┘
          │                 │                  │
          │ REST            │ REST             │ Phoenix
          │ (results)       │ (events)         │ (updates)
          ▼                 ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│                       SERVER                             │
│                                                          │
│  Process Events → Make Decisions → Mutate State         │
│                          ↓                               │
│                  Broadcast Updates                       │
└─────────────────────────────────────────────────────────┘
```

## Communication Protocols

### REST API (Command/Result Cycle)

Used for:
- Getting next command: `GET /api/sessions/:id/next-command`
- Submitting results: `POST /api/sessions/:id/submit-result/:interactionId`

**Flow:**
1. Client calls `getNextCommand()` → Server returns command
2. Client executes command
3. Client calls `submitResult()` → Server processes, broadcasts `interaction:completed`
4. Client receives `interaction:completed` Phoenix event
5. If autoPlay enabled → goto step 1

### Phoenix Channels (State Updates)

Used for:
- State synchronization (`session_updated`)
- AutoPlay triggers (`interaction:completed`)

**Why Phoenix + REST?**
- REST: Explicit request/response for commands (client controls when)
- Phoenix: Reactive state updates (server controls when)
- Best of both: Client pulls commands, server pushes updates

## Event Flow by Executor

### TerminalExecutor (Shell Commands)

```
User Terminal
      ↓
Execute shell command
      ↓
Wait for completion
      ↓
POST /api/sessions/:id/submit-result/:interactionId
      ↓
Server processes
      ↓
Phoenix: interaction:completed
      ↓
Client: if autoPlay → getNextCommand()
```

No events to server during execution. Just result at end.

### ClaudeTerminalExecutor (Claude CLI)

```
Claude CLI in Terminal
      ↓
Hooks fire (SessionStart, SessionEnd, etc.)
      ↓
POST to Callback Server (localhost)
      ↓
Callback Server wraps with context:
  { session_id, interaction_id, event_type, event_data, timestamp }
      ↓
POST /api/sessions/:id/events
      ↓
Server processes event:
  - SessionStart → update conversation_id
  - SessionEnd → mark interaction complete
  - Others → log/broadcast
      ↓
Phoenix: session_updated (for conversation_id)
Phoenix: interaction:completed (for SessionEnd)
      ↓
Client: if autoPlay → getNextCommand()
```

**Why Callback Server?**
- Claude CLI runs as terminal command
- Can't reach remote server directly (localhost only)
- Callback server bridges: localhost → remote

### ClaudeExecutor (Claude Agent SDK)

```
Claude Agent SDK
      ↓
Stream events (assistant, tool_use, tool_result, result)
      ↓
For each event:
  POST /api/sessions/:id/events
  Body: { session_id, interaction_id, event_type, event_data, timestamp }
      ↓
Server processes event:
  - Capture conversation_id
  - Log events for audit
  - On final result → mark complete
      ↓
Phoenix: session_updated (for conversation_id)
Phoenix: interaction:completed (on completion)
      ↓
Client: if autoPlay → getNextCommand()
```

**No Callback Server**
- SDK runs in extension process
- Can reach remote server directly
- Posts events immediately

### SubprocessExecutor (Shell in Background)

```
Subprocess
      ↓
Execute shell command
      ↓
Wait for completion
      ↓
POST /api/sessions/:id/submit-result/:interactionId
      ↓
Server processes
      ↓
Phoenix: interaction:completed
      ↓
Client: if autoPlay → getNextCommand()
```

Same as TerminalExecutor. No events during execution.

## Server Event Processing

Single endpoint: `POST /api/sessions/:sessionId/events`

**Request Body:**
```typescript
{
  session_id: string;
  interaction_id: string;
  event_type: string;
  event_data: any;
  timestamp: string;
}
```

**Server Logic:**
```
1. Validate session/interaction exists
2. Determine event type:
   - SessionStart → update external_conversation_id
   - SessionEnd → mark interaction complete, emit interaction:completed
   - assistant/tool_use/etc → log, optionally broadcast
3. Mutate database state
4. Broadcast Phoenix events:
   - session_updated (for state changes)
   - interaction:completed (for completions)
```

**Why Single Endpoint?**
- Simpler client (one place to POST)
- Server decides what each event means
- Easy to add new event types

## Phoenix Events

### Event: `session_updated`

**When Emitted:**
- Conversation ID updated
- Session status changes
- Session metadata changes

**Payload:**
```typescript
{
  session: Session  // Full updated session
}
```

**Client Action:**
- Update local state
- Refresh UI

### Event: `interaction:completed`

**When Emitted:**
- Result submitted via REST API
- SessionEnd hook received
- Claude SDK result event received

**Payload:**
```typescript
{
  session_id: string;
  interaction_id: string;
  status: 'ok' | 'error';
}
```

**Client Action:**
- Mark interaction done
- If autoPlay enabled → call `getNextCommand()`

## AutoPlay Implementation

**Client Side:**
```typescript
// User clicks "Play"
orchestrator.setAutoPlay(sessionId, true);
orchestrator.executeNext(sessionId); // Kick off first command

// On Phoenix event
phoenixClient.on('interaction:completed', (event) => {
  const state = sessionStates.get(event.session_id);
  if (state && state.autoPlayEnabled && !state.isExecuting) {
    // Call REST API for next command
    orchestrator.executeNext(event.session_id);
  }
});
```

**Flow:**
1. User enables autoPlay
2. Client calls `getNextCommand()` (REST)
3. Client executes command
4. Client submits result (REST)
5. Server emits `interaction:completed` (Phoenix)
6. Client receives event
7. If autoPlay → goto step 2

## Client Business Logic Removed

### Before (Client Decides)
```typescript
// Client decided when to update conversation ID
if (conversationId) {
  await client.updateExternalConversationId(sessionId, conversationId);
}

// Client decided when to execute next
if (autoPlay) {
  await executeNext();
}
```

### After (Server Decides)
```typescript
// Client just forwards events
await postEventToServer({ event_type: 'SessionStart', event_data });

// Server processes, updates conversation ID, broadcasts session_updated
// Client reacts to broadcast by updating local state

// Client just reacts to completion
phoenixClient.on('interaction:completed', () => {
  if (autoPlay) await executeNext();
});
```

## Callback Server Role

**Purpose:** Bridge between localhost (Claude CLI) and remote server

**Responsibilities:**
1. Receive POST from Claude CLI hooks
2. Extract hook data
3. Wrap with context:
   ```typescript
   {
     session_id: sessionId,        // From environment
     interaction_id: interactionId, // From URL
     event_type: hookType,          // SessionStart, SessionEnd, etc.
     event_data: hookPayload,       // Whatever hook sent
     timestamp: new Date().toISOString()
   }
   ```
4. POST to server: `/api/sessions/:id/events`
5. Resolve promise (signal to executor)

**No Business Logic:**
- Doesn't decide what events mean
- Doesn't update conversation IDs
- Doesn't call other APIs
- Just wraps and forwards

## Error Handling

### Event Posting Fails

If POST to `/api/sessions/:id/events` fails:
- Log error
- Continue execution (don't fail command)
- Result will still be submitted via REST

### Phoenix Disconnects

If Phoenix connection drops:
- Client continues to work via REST
- AutoPlay pauses (no events received)
- On reconnect, Phoenix resubscribes
- Client may need manual refresh to sync state

### Server Event Processing Fails

If server fails to process event:
- Returns 500 error
- Client logs but doesn't fail
- Result submission still works (separate endpoint)

## Migration Path

### Phase 1: Keep Current Architecture
- No events endpoint
- No Phoenix events
- Polling/manual execution only

### Phase 2: Add Events (No AutoPlay)
- Server accepts `/events` endpoint
- Client posts events
- Server processes but doesn't broadcast
- Used for logging/audit only

### Phase 3: Add Phoenix Events
- Server broadcasts `session_updated`, `interaction:completed`
- Client subscribes but doesn't react
- Used for real-time UI updates only

### Phase 4: Event-Driven AutoPlay
- Client reacts to `interaction:completed`
- AutoPlay triggered by events
- Full event-driven architecture

## Benefits

1. **Server is smart** - Business logic centralized
2. **Client is simple** - Execute and report
3. **Extensible** - Easy to add new event types
4. **Auditable** - All events logged server-side
5. **Real-time** - Phoenix for instant updates
6. **Reliable** - REST fallback if Phoenix fails
7. **Testable** - Clear boundaries between client/server

## Trade-offs

**Pros:**
- Centralized logic (easier to maintain)
- Server controls behavior (easier to change)
- Event history (debugging, audit)

**Cons:**
- More network calls (events posted during execution)
- Phoenix dependency (though REST fallback exists)
- Slightly more complex client setup (Phoenix + REST)

## Decision: Hybrid Architecture

**Why not pure REST?** No real-time updates, manual polling
**Why not pure Phoenix?** Request/response better for command retrieval
**Why hybrid?** Best of both - explicit pulls, reactive pushes
