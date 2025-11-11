# Event and Result Ordering

Critical timing and ordering considerations for the event-driven architecture.

## The Race Condition Problem

When ClaudeTerminalExecutor completes, multiple things happen asynchronously:

```
execute() returns
        ↓
    [RACE CONDITION]
        ↓
   ┌────┴────┐
   ↓         ↓
Callback    Client calls
forwarding  submitResult()
to server   to server
   ↓         ↓
   └────┬────┘
        ↓
    Which arrives first?
```

## Possible Orderings

### Scenario 1: Result Arrives First (Common)
```
1. execute() returns (terminal + callback received)
2. orchestrator.submitResult() called
3. Server receives result → marks complete → emits interaction:completed
4. Callback server finishes forwarding SessionEnd event
5. Server receives SessionEnd event → already marked complete → idempotent
```

### Scenario 2: Event Arrives First (Rare)
```
1. execute() returns (terminal + callback received)
2. Callback server forwards SessionEnd event immediately
3. Server receives SessionEnd → marks complete → emits interaction:completed
4. orchestrator.submitResult() called
5. Server receives result → already marked complete → updates details
```

## Server Must Handle Both Orders

**Idempotent Processing Required:**

```typescript
// Server event handler
function handleInteractionEvent(sessionId, interactionId, event) {
  const interaction = db.getInteraction(sessionId, interactionId);

  if (event.type === 'session_end') {
    // Mark as complete if not already
    if (interaction.status !== 'complete') {
      interaction.status = 'complete';
      interaction.completed_at = now();

      // Emit Phoenix event
      broadcast('interaction:completed', {
        session_id: sessionId,
        interaction_id: interactionId,
        status: 'ok'
      });
    }

    // Always update event data (even if already complete)
    interaction.events.push(event);
  }
}

// Server result handler
function handleInteractionResult(sessionId, interactionId, result) {
  const interaction = db.getInteraction(sessionId, interactionId);

  // Always update result data
  interaction.result = result;

  // Mark as complete if not already
  if (interaction.status !== 'complete') {
    interaction.status = 'complete';
    interaction.completed_at = now();

    // Emit Phoenix event
    broadcast('interaction:completed', {
      session_id: sessionId,
      interaction_id: interactionId,
      status: result.status
    });
  }
}
```

## Why This Design Works

1. **Idempotent Completion**: Can mark complete multiple times safely
2. **Event Accumulation**: All events stored regardless of order
3. **Single Phoenix Event**: Only emit once (first to mark complete)
4. **Data Merge**: Both result and events end up in final state

## Client Doesn't Care About Order

The client:
1. Calls `submitResult()` (fire and forget)
2. Waits for `interaction:completed` Phoenix event
3. Proceeds when event received

The client doesn't know or care whether the result or event arrived first - it just waits for the server to signal completion via Phoenix.

## Edge Case: Callback Never Arrives

If SessionEnd hook fails to fire or callback server is down:

```
1. execute() waits indefinitely for callback promise
2. Timeout needed!
```

**Solution: Add Timeout**

```typescript
// In ClaudeTerminalExecutor
const CALLBACK_TIMEOUT = 30000; // 30 seconds

const timeoutPromise = new Promise((_, reject) => {
  setTimeout(() => reject(new Error('Callback timeout')), CALLBACK_TIMEOUT);
});

try {
  await Promise.race([
    Promise.all([executePromise, callback.promise]),
    timeoutPromise
  ]);
} catch (error) {
  if (error.message === 'Callback timeout') {
    // Hook didn't fire, but command might have succeeded
    // Log warning and continue with terminal result
    getLogger().appendLine('[ClaudeTerminal] Callback timeout - using terminal result');
  } else {
    throw error;
  }
}
```

## Edge Case: Multiple SessionEnd Hooks

If Claude CLI fires SessionEnd multiple times (shouldn't happen but could):

```
1. First hook → forwarded → marks complete → emits event
2. Second hook → forwarded → already complete → no-op
```

Server's idempotent handling prevents duplicate Phoenix events.

## Event Deduplication

Server should deduplicate events based on:
- session_id + interaction_id + event_type + timestamp

If exact same event arrives twice:
```typescript
const eventKey = `${sessionId}:${interactionId}:${event.type}:${timestamp}`;
if (processedEvents.has(eventKey)) {
  return; // Already processed
}
processedEvents.add(eventKey);
```

## Ordering Guarantees

### What IS Guaranteed:
- execute() returns after both terminal AND callback received locally
- submitResult() called after execute() returns
- Server processes events idempotently

### What is NOT Guaranteed:
- Network delivery order (result vs event)
- Callback forwarding speed
- Phoenix event delivery timing

### What Client Can Rely On:
- `interaction:completed` Phoenix event signals "server says done"
- After this event, calling getNextCommand() will work

## Summary

**Problem**: Result and event can arrive at server in any order
**Solution**: Server handles both orders idempotently
**Client Impact**: None - client waits for Phoenix event regardless
**Edge Cases**: Timeout needed for missing callbacks
