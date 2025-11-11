# Callback Server

Local HTTP server that bridges Claude CLI hooks to backend server. Maps to `src/callbackServer.ts`.

## Purpose

The Callback Server is a **dumb forwarder** that bridges localhost (Claude CLI) and remote server. Claude CLI can only POST to localhost, so the callback server receives hooks, wraps them with context (session_id, interaction_id), and forwards to the backend `/events` endpoint. The server makes all decisions about what events mean.

## Core Responsibilities

1. Start ephemeral HTTP server per session on localhost
2. Provide callback URLs for Claude CLI hooks
3. Receive POST requests from Claude CLI hooks
4. **Wrap hook data with context** (session_id, interaction_id, timestamp)
5. **Forward to backend** `/api/sessions/:id/events` endpoint
6. Resolve promises to signal command completion
7. Clean up servers when sessions end

## No Business Logic

The callback server is **deliberately dumb**:
- Does NOT decide what events mean
- Does NOT update conversation IDs
- Does NOT call multiple APIs
- Does NOT process hook data
- Just wraps and forwards

## What This Module Does NOT Do

- Does NOT receive events from Claude Agent SDK (ClaudeExecutor streams directly to server)
- Does NOT run for subprocess execution (no hooks in background processes)
- Does NOT run for shell terminal execution (no hooks for shell commands)
- Does NOT manage session state (server handles that)
- Does NOT create Phoenix channel connections (SessionManager handles that)

## ALL Claude Hooks Go Through Callback Server

**Including SessionStart hooks.** All Claude CLI hooks (SessionStart, SessionEnd, etc.) POST to the callback server, which wraps them and forwards to the backend `/events` endpoint. This provides a consistent flow and ensures the server gets proper context with every event.

## Architecture

### Why Local Callback Server?

Claude CLI runs on the local machine and calls hooks. We need a local endpoint for it to POST to. Options:

1. **Direct to backend** - Claude CLI can't reach remote server if behind firewall/VPN
2. **Local callback server** - Always reachable from local machine ✅

The callback server acts as a bridge: Claude CLI → Local callback → Backend API

### Request Flow

```
Claude CLI (terminal)
        ↓
Hook executes (SessionStart, SessionEnd, etc.)
        ↓
POST http://localhost:PORT/command-complete/:interactionId
Body: { ...hook data from Claude... }
        ↓
CallbackServer receives request
        ↓
Wrap with context:
  {
    session_id: from env (CODE_MY_SPEC_SESSION_ID),
    interaction_id: from URL param,
    event_type: from hook data or inferred,
    event_data: raw hook payload,
    timestamp: new Date().toISOString()
  }
        ↓
POST /api/sessions/:sessionId/events
(Single events endpoint for everything)
        ↓
Backend processes event:
  - SessionStart → update conversation_id
  - SessionEnd → mark complete, emit interaction:completed
  - Others → log/broadcast
        ↓
CallbackServer resolves promise
        ↓
ClaudeTerminalExecutor.execute() returns
```

## Public Interface

```typescript
// Get or create callback server for a session
export async function getCallbackServer(sessionId: string): Promise<CallbackServer>

// Clean up callback server for a session
export async function cleanupCallbackServer(sessionId: string): Promise<void>

export class CallbackServer {
  constructor(
    sessionId: string,
    apiUrl: string,
    apiToken: string
  )

  // Start the server
  async start(): Promise<void>

  // Get callback URL for an interaction
  getCallbackUrl(interactionId: string): string

  // Register callback handler for interaction completion
  onCommandComplete(interactionId: string, callback: (data: any) => void): void

  // Stop the server
  async stop(): Promise<void>
}
```

## Server Lifecycle

### Global Server Map

```typescript
// One server per session
const servers = new Map<string, CallbackServer>();

export async function getCallbackServer(sessionId: string): Promise<CallbackServer> {
  if (!servers.has(sessionId)) {
    const server = new CallbackServer(sessionId);
    await server.start();
    servers.set(sessionId, server);
  }
  return servers.get(sessionId)!;
}
```

### Server Creation

```typescript
async start(): Promise<void> {
  return new Promise((resolve, reject) => {
    this.app = express();
    this.app.use(express.json());

    // Endpoint for Claude hooks
    this.app.post('/command-complete/:interactionId', async (req, res) => {
      await this.handleCallback(req, res);
    });

    // Start on random available port (0 = OS assigns)
    this.server = this.app.listen(0, 'localhost', () => {
      const address = this.server!.address() as AddressInfo;
      this.port = address.port;
      resolve();
    });
  });
}
```

### Port Assignment

The server uses port `0`, which tells the OS to assign any available port. This prevents port conflicts when multiple sessions are active.

```typescript
this.server = this.app.listen(0, 'localhost', () => {
  const address = this.server!.address() as AddressInfo;
  this.port = address.port; // OS-assigned port
});
```

## Callback URLs

```typescript
getCallbackUrl(interactionId: string): string {
  if (!this.port) {
    throw new Error('Server not started');
  }
  return `http://localhost:${this.port}/command-complete/${interactionId}`;
}
```

Example: `http://localhost:54321/command-complete/interaction-abc123`

This URL is set as `CODE_MY_SPEC_CALLBACK_URL` environment variable for the Claude CLI command.

## Hook Handling

### Endpoint Handler

```typescript
private async handleCallback(req: express.Request, res: express.Response): Promise<void> {
  const interactionId = req.params.interactionId;
  const hookData = req.body;

  try {
    // Get registered callback handler
    const handler = this.callbacks.get(interactionId);

    if (handler) {
      // Forward to backend server
      await this.forwardToBackend(interactionId, hookData);

      // Notify local handler (resolves promise in executor)
      handler(hookData);

      // Remove handler (one-time use)
      this.callbacks.delete(interactionId);

      res.status(200).json({ success: true });
    } else {
      res.status(404).json({ error: 'No handler registered for this interaction' });
    }
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
}
```

### Callback Registration

```typescript
private callbacks = new Map<string, (data: any) => void>();

onCommandComplete(interactionId: string, callback: (data: any) => void): void {
  this.callbacks.set(interactionId, callback);
}
```

TerminalExecutor registers a callback before executing the command:

```typescript
// In TerminalExecutor
const callbackPromise = new Promise((resolve) => {
  callbackServer.onCommandComplete(interactionId, resolve);
});

// Execute command...

// Wait for callback
await callbackPromise;
```

## Forwarding to Backend

**The Core Responsibility: Dumb Wrapping**

```typescript
private async forwardToBackend(interactionId: string, hookData: any): Promise<void> {
  // Get credentials from constructor (passed by ClaudeTerminalExecutor when creating server)
  // Note: These are stored as instance variables when server is created, not from process.env
  const apiUrl = this.apiUrl;
  const apiToken = this.apiToken;
  const sessionId = this.sessionId;

  if (!apiUrl || !apiToken || !sessionId) {
    throw new Error('Missing required credentials');
  }

  // Infer event type from hook data or default
  const eventType = hookData.event_type || hookData.type || 'claude_hook';

  // Wrap hook data with context (this is ALL we do)
  const wrappedEvent = {
    session_id: sessionId,
    interaction_id: interactionId,
    event_type: eventType,
    event_data: hookData,  // Raw hook payload, unchanged
    timestamp: new Date().toISOString()
  };

  // POST to single events endpoint
  const response = await fetch(
    `${apiUrl}/api/sessions/${sessionId}/events`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiToken}`
      },
      body: JSON.stringify(wrappedEvent)
    }
  );

  if (!response.ok) {
    throw new Error(`Failed to forward event to backend: ${response.statusText}`);
  }

  // Done. Server decides what to do with event.
}
```

### Backend Processing

When the backend receives the event (from callback server or ClaudeExecutor):
1. Validates session/interaction exists
2. Determines event type and extracts relevant data
3. Makes decisions (update conversation ID, mark complete, etc.)
4. Mutates database state
5. Emits Phoenix events (`session_updated`, `interaction:completed`)
6. Extension receives Phoenix event and reacts

## Cleanup

```typescript
async stop(): Promise<void> {
  return new Promise((resolve) => {
    if (this.server) {
      this.server.close(() => {
        this.callbacks.clear();
        resolve();
      });
    } else {
      resolve();
    }
  });
}

export async function cleanupCallbackServer(sessionId: string): Promise<void> {
  const server = servers.get(sessionId);
  if (server) {
    await server.stop();
    servers.delete(sessionId);
  }
}
```

Called by TerminalExecutor when session ends.

## Hook Environment Variables

The TerminalExecutor sets this environment variable for Claude CLI:

```bash
CODE_MY_SPEC_CALLBACK_URL=http://localhost:54321/command-complete/interaction-abc123
```

The SessionEnd hook script uses this:

```bash
#!/bin/bash
# hooks/submit-result.sh

if [ -n "$CODE_MY_SPEC_CALLBACK_URL" ]; then
  curl -X POST "$CODE_MY_SPEC_CALLBACK_URL" \
    -H "Content-Type: application/json" \
    -d "{\"status\":\"complete\",\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"}"
fi
```

## Multiple Interactions Per Session

A single session may have multiple interactions (commands). Each interaction gets its own callback:

```typescript
// Session 1, Interaction A
http://localhost:54321/command-complete/interaction-A

// Session 1, Interaction B
http://localhost:54321/command-complete/interaction-B
```

Same server (same port), different endpoints.

## Error Handling

### Server Start Failure

If the server fails to start (port in use, permissions, etc.):
- Error is thrown to caller
- TerminalExecutor.execute() fails
- Orchestrator receives error and submits failed result

### Hook Forwarding Failure

If forwarding to backend fails:
- Error is logged
- Local callback still resolves (command completes locally)
- Backend won't receive event (may need retry logic)

### Missing Callback Handler

If hook is received but no handler registered:
- Return 404 response
- Log warning
- Hook is lost (may indicate race condition or bug)

## Concurrency

Multiple commands can be executing simultaneously in the same session:

```typescript
// Both register callbacks on same server
interaction1: http://localhost:54321/command-complete/int-1
interaction2: http://localhost:54321/command-complete/int-2

// Callbacks stored in map
callbacks = {
  'int-1': resolve1,
  'int-2': resolve2
}

// Hooks can arrive in any order
POST /command-complete/int-2 → resolve2() → delete('int-2')
POST /command-complete/int-1 → resolve1() → delete('int-1')
```

This works because each interaction has a unique ID.

## Security Considerations

### Localhost Only

The server binds to `localhost`, not `0.0.0.0`:

```typescript
this.server = this.app.listen(0, 'localhost', () => { ... });
```

This ensures only local processes can reach it.

### No Authentication

The callback server does NOT authenticate requests. This is acceptable because:
1. Only accessible from localhost
2. Claude CLI is trusted (runs on same machine)
3. Interaction IDs are effectively nonces (hard to guess)

If needed, could add:
- HMAC signatures
- Time-limited tokens
- Request origin validation

### Port Randomization

Using port `0` provides some security through obscurity - each session gets a random port that's hard to predict.

## Logging

```typescript
getLogger().appendLine(`[CallbackServer] Started for session ${sessionId} on port ${port}`);
getLogger().appendLine(`[CallbackServer] Received callback for interaction ${interactionId}`);
getLogger().appendLine(`[CallbackServer] Forwarding hook to backend...`);
getLogger().appendLine(`[CallbackServer] Stopped for session ${sessionId}`);
```

## Comparison with ClaudeExecutor

| Feature | CallbackServer (ClaudeTerminalExecutor) | ClaudeExecutor |
|---|---|---|
| Execution Mode | Manual (terminal) | Agentic (SDK) |
| Hook Source | Claude CLI | Claude Agent SDK |
| Event Destination | Local callback → Backend | Directly to backend |
| Network | HTTP localhost | HTTP to backend |
| Why Different? | CLI runs locally, can't reach backend directly | SDK runs in extension, can reach backend |

## Dependencies

- `express` - HTTP server framework
- `fetch` - HTTP client for forwarding to backend
- `getLogger()` - Extension output channel logging
