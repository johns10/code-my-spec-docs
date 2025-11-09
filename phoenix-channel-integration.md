# Phoenix Channel Integration

## Overview

This document describes the Phoenix Channel integration for real-time session updates, replacing HTTP polling with WebSocket-based real-time communication.

## Architecture

### Components

1. **Phoenix Channel Client** (`src/phoenixChannel.ts`)
   - Manages WebSocket connection to Phoenix server
   - Authenticates using JWT token
   - Joins user-specific channel (`vscode:user:{user_id}`)
   - Handles real-time session events

2. **Session State Manager** (`src/sessionState.ts`)
   - Centralized state management for all sessions
   - Provides pub/sub pattern for session updates
   - Single source of truth for session data
   - Supports both session-specific and global subscriptions

3. **Session Manager** (`src/sessionManager.ts`)
   - Coordinates HTTP client, Phoenix channel, and state manager
   - Loads initial sessions via HTTP
   - Connects to Phoenix channel for real-time updates
   - Provides unified interface for session operations

4. **Updated Session List Provider** (`src/views/sessionList.ts`)
   - Now uses SessionManager instead of direct HTTP calls
   - Subscribes to session state changes
   - Automatically refreshes UI when sessions update via Phoenix

## Data Flow

### Initial Load
```
Extension Activation
  → SessionManager.initialize()
    → HTTP: Load initial sessions
    → Populate SessionStateManager
    → Connect Phoenix Channel
      → Subscribe to vscode:user:{user_id}
```

### Real-time Updates
```
Backend Session Update
  → Phoenix Channel broadcast
    → PhoenixChannelClient receives event
      → SessionStateManager.updateSession()
        → Notify all subscribed listeners
          → Session List Provider refreshes UI
```

## Event Types

The Phoenix channel handles three event types:

1. **session_created** - New session created
2. **session_updated** - Session state changed
3. **session_deleted** - Session removed

All events trigger updates to the SessionStateManager, which notifies subscribed components.

## Benefits

### Before (HTTP Polling)
- Manual refresh required
- Stale data between refreshes
- Higher server load from polling
- Delayed updates

### After (Phoenix Channels)
- Automatic real-time updates
- Always current data
- Lower server load (single WebSocket connection)
- Instant updates across all views

## Implementation Details

### Authentication

The Phoenix Socket authenticates using the JWT token from the authentication provider:

```typescript
this.socket = new Socket(`${this.serverUrl}/socket`, {
  params: { token: authToken }
});
```

### Channel Topic

Each user connects to their own channel topic:
```
vscode:user:{user_id}
```

This ensures users only receive updates for their own sessions.

### Reconnection

The Phoenix JavaScript client handles reconnection automatically using exponential backoff. If the connection drops, it will:

1. Attempt to reconnect
2. Rejoin the channel
3. Continue receiving updates

### Fallback Behavior

If Phoenix channel connection fails:
- System falls back to HTTP polling
- SessionManager provides `refreshSessions()` method
- UI can manually refresh when needed
- No loss of functionality, just loss of real-time updates

## Type Definitions

### Phoenix Library Types

Custom type definitions added in `src/types/phoenix.d.ts` for the Phoenix JavaScript client.

### Session Interface

Updated to include all fields from backend schema:
- `user_id` - Required for channel subscription
- `agent`, `environment`, `execution_mode` - Session configuration
- `external_conversation_id` - Claude conversation tracking
- Full interaction history with commands and results

## Usage

### For View Providers

```typescript
// Subscribe to session updates
const unsubscribe = sessionManager.onSessionsUpdate((sessions) => {
  // Update UI
  this.refresh();
});

// Get current sessions (no HTTP call)
const sessions = sessionManager.getAllSessions();

// Clean up subscription
unsubscribe();
```

### For Individual Session Updates

```typescript
// Subscribe to specific session
const unsubscribe = sessionManager.onSessionUpdate(sessionId, (session) => {
  // Handle session update
});
```

## Testing

To test the Phoenix Channel integration:

1. **Start a session** - Should appear immediately in all views
2. **Update a session** - Changes should propagate instantly
3. **Complete a session** - Status updates should be immediate
4. **Disconnect network** - Should reconnect automatically when restored
5. **Check logs** - `[PhoenixChannel]` and `[SessionState]` log entries show activity

## Future Enhancements

1. **Presence tracking** - Show which sessions are actively being viewed
2. **Collaborative features** - Multiple users working on same project
3. **Progress updates** - Stream command output in real-time
4. **Channel push** - Send commands from extension to backend via channel
5. **Interaction-level subscriptions** - Subscribe to individual interaction updates

## Configuration

Phoenix channel connection uses the same `codeMySpec.serverUrl` configuration as HTTP client:
- Automatically converts `http://` to `ws://` for WebSocket
- Automatically converts `https://` to `wss://` for secure WebSocket

## Logging

All Phoenix channel activity is logged with `[PhoenixChannel]` prefix:
- Connection events
- Channel join/leave
- Message received
- Errors and reconnections

SessionStateManager logs with `[SessionState]` prefix:
- Session updates
- Listener subscriptions
- State changes

## Error Handling

Errors are logged but don't crash the extension:
- Connection failures → Fall back to HTTP polling
- Message parsing errors → Logged, ignored
- State update errors → Logged in listener error handlers
