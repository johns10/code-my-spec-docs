# Test Strategy for Session Execution

Comprehensive testing strategy to avoid manual testing hell.

## The Problem

Manual testing of session execution is painful:
- Start session, wait for command, execute, check result, repeat
- Claude commands take forever
- Terminal output is messy
- Hard to test edge cases (failures, timeouts, race conditions)
- No way to test parallel sessions easily

## The Solution: Multi-Layer Testing

### Layer 1: Unit Tests (Fast, Isolated)
Test individual components in isolation with mocks.

### Layer 2: Integration Tests (Medium, Real Components)
Test multiple components together with real HTTP/Phoenix but mocked executors.

### Layer 3: End-to-End Tests (Slow, Full Stack)
Test complete flows with real server, but controllable Claude commands.

### Layer 4: Manual Testing (Only for Final Verification)
Human verification of UX, only after automated tests pass.

---

## Layer 1: Unit Tests

### Test Each Executor Independently

**TerminalExecutor**
```typescript
describe('TerminalExecutor', () => {
  let mockTerminalManager: jest.Mocked<SessionTerminalManager>;
  let executor: TerminalExecutor;

  beforeEach(() => {
    mockTerminalManager = {
      executeCommand: jest.fn().mockResolvedValue({
        stdout: 'test output',
        stderr: '',
        exitCode: 0,
        duration_ms: 100
      })
    };
    executor = new TerminalExecutor();
    // Inject mock
  });

  it('should execute shell command in terminal', async () => {
    const result = await executor.execute('session-1', interaction, session);

    expect(mockTerminalManager.executeCommand).toHaveBeenCalledWith(
      'session-1',
      'interaction-1',
      { command: 'ls -la' },
      expect.objectContaining({ CODE_MY_SPEC_SESSION_ID: 'session-1' })
    );
    expect(result.exitCode).toBe(0);
  });

  it('should reject Claude commands', async () => {
    const claudeCommand = { command: 'claude', metadata: { prompt: 'test' } };
    await expect(executor.execute('session-1', { command: claudeCommand }, session))
      .rejects.toThrow('cannot execute Claude commands');
  });

  it('should handle empty commands', async () => {
    const result = await executor.execute('session-1', { command: '' }, session);
    expect(result.exitCode).toBe(0);
    expect(mockTerminalManager.executeCommand).not.toHaveBeenCalled();
  });
});
```

**ClaudeTerminalExecutor**
```typescript
describe('ClaudeTerminalExecutor', () => {
  let mockCallbackServer: jest.Mocked<CallbackServer>;
  let mockCommandProcessor: jest.Mocked<CommandProcessor>;
  let executor: ClaudeTerminalExecutor;

  beforeEach(() => {
    mockCallbackServer = {
      getCallbackUrl: jest.fn().mockReturnValue('http://localhost:3000/callback/int-1'),
      onCommandComplete: jest.fn()
    };
    mockCommandProcessor = {
      createTempFile: jest.fn().mockResolvedValue('/tmp/prompt_123.tmp')
    };
    // Setup executor with mocks
  });

  it('should build Claude CLI command with resume flag', async () => {
    session.external_conversation_id = 'conv_abc123';

    await executor.execute('session-1', claudeInteraction, session);

    expect(mockTerminalManager.executeCommand).toHaveBeenCalledWith(
      expect.anything(),
      expect.anything(),
      { command: expect.stringContaining('--resume conv_abc123') },
      expect.anything()
    );
  });

  it('should wait for both terminal and callback', async () => {
    const terminalPromise = Promise.resolve({ exitCode: 0 });
    const callbackPromise = new Promise(resolve => setTimeout(resolve, 100));

    // Mock both promises
    // Assert execute() doesn't return until both resolve
  });

  it('should timeout if callback never arrives', async () => {
    mockCallbackServer.onCommandComplete.mockImplementation(() => {
      // Never calls callback
    });

    await expect(executor.execute('session-1', claudeInteraction, session))
      .rejects.toThrow('Callback timeout');
  });
});
```

**CallbackServer**
```typescript
describe('CallbackServer', () => {
  let server: CallbackServer;
  let mockFetch: jest.Mock;

  beforeEach(() => {
    mockFetch = jest.fn().mockResolvedValue({ ok: true });
    global.fetch = mockFetch;
    server = new CallbackServer('session-1', 'http://api.test', 'token-123');
  });

  it('should wrap hook data with context', async () => {
    await server.start();
    const url = server.getCallbackUrl('int-1');

    // Simulate hook POST
    await fetch(url, {
      method: 'POST',
      body: JSON.stringify({ status: 'complete' })
    });

    expect(mockFetch).toHaveBeenCalledWith(
      'http://api.test/api/sessions/session-1/events',
      expect.objectContaining({
        body: expect.stringContaining('"session_id":"session-1"'),
        body: expect.stringContaining('"interaction_id":"int-1"')
      })
    );
  });

  it('should resolve promise when callback received', async () => {
    await server.start();

    let resolved = false;
    server.onCommandComplete('int-1', () => { resolved = true; });

    const url = server.getCallbackUrl('int-1');
    await fetch(url, { method: 'POST', body: '{}' });

    expect(resolved).toBe(true);
  });

  it('should handle missing credentials', async () => {
    const badServer = new CallbackServer('session-1', '', '');
    await expect(badServer.forwardToBackend('int-1', {}))
      .rejects.toThrow('Missing required credentials');
  });
});
```

**SessionOrchestrator**
```typescript
describe('SessionOrchestrator', () => {
  let orchestrator: SessionOrchestrator;
  let mockExecutors: MockExecutors;
  let mockClient: jest.Mocked<CodeMySpecClient>;

  beforeEach(() => {
    // Mock all 4 executors
    mockExecutors = {
      terminal: createMockExecutor(),
      claudeTerminal: createMockExecutor(),
      subprocess: createMockExecutor(),
      claude: createMockExecutor()
    };
    // Inject mocks
  });

  it('should pick correct executor for manual shell', () => {
    const session = { execution_mode: 'manual' };
    const command = { command: 'ls' };

    const executor = orchestrator.pickExecutor(session, command);
    expect(executor).toBe(mockExecutors.terminal);
  });

  it('should pick correct executor for manual Claude', () => {
    const session = { execution_mode: 'manual' };
    const command = { command: 'claude', metadata: { prompt: 'test' } };

    const executor = orchestrator.pickExecutor(session, command);
    expect(executor).toBe(mockExecutors.claudeTerminal);
  });

  it('should pick correct executor for agentic shell', () => {
    const session = { execution_mode: 'agentic' };
    const command = { command: 'npm test' };

    const executor = orchestrator.pickExecutor(session, command);
    expect(executor).toBe(mockExecutors.subprocess);
  });

  it('should pick correct executor for agentic Claude', () => {
    const session = { execution_mode: 'agentic' };
    const command = { command: 'claude', metadata: { prompt: 'test' } };

    const executor = orchestrator.pickExecutor(session, command);
    expect(executor).toBe(mockExecutors.claude);
  });

  it('should execute next command when interaction:completed received and autoPlay enabled', async () => {
    orchestrator.setAutoPlay('session-1', true);

    // Simulate Phoenix event
    await orchestrator.handleInteractionCompleted({
      session_id: 'session-1',
      interaction_id: 'int-1',
      status: 'ok'
    });

    expect(mockClient.sessions.getNextCommand).toHaveBeenCalledWith('session-1');
  });

  it('should NOT execute next command when autoPlay disabled', async () => {
    orchestrator.setAutoPlay('session-1', false);

    await orchestrator.handleInteractionCompleted({
      session_id: 'session-1',
      interaction_id: 'int-1',
      status: 'ok'
    });

    expect(mockClient.sessions.getNextCommand).not.toHaveBeenCalled();
  });

  it('should handle parallel sessions', async () => {
    const parentSession = { id: 'parent-1', execution_mode: 'agentic' };
    const interaction = {
      command: {
        command: 'spawn_sessions',
        metadata: { child_session_ids: ['child-1', 'child-2', 'child-3'] }
      }
    };

    mockClient.sessions.getSession
      .mockResolvedValueOnce({ id: 'child-1', status: 'active' })
      .mockResolvedValueOnce({ id: 'child-2', status: 'active' })
      .mockResolvedValueOnce({ id: 'child-3', status: 'active' });

    await orchestrator.handleParallelSessions(interaction, parentSession);

    expect(mockClient.sessions.getSession).toHaveBeenCalledTimes(3);
    expect(mockClient.sessions.submitResult).toHaveBeenCalledWith(
      'parent-1',
      expect.anything(),
      expect.objectContaining({
        stdout: expect.stringContaining('"total_children":3')
      })
    );
  });
});
```

---

## Layer 2: Integration Tests

### Test Component Interactions with Real HTTP/Phoenix

**Phoenix Event Flow**
```typescript
describe('Phoenix Event Integration', () => {
  let testServer: TestPhoenixServer;
  let orchestrator: SessionOrchestrator;
  let sessionManager: SessionManager;

  beforeAll(async () => {
    testServer = await TestPhoenixServer.start();
  });

  it('should trigger next command on interaction:completed event', async (done) => {
    // Real Phoenix connection
    await sessionManager.initialize();
    orchestrator.setAutoPlay('session-1', true);

    // Mock HTTP client for getNextCommand
    mockClient.sessions.getNextCommand.mockResolvedValue({
      id: 'session-1',
      interactions: [{ id: 'int-2', command: { command: 'ls' } }]
    });

    // Emit event from test server
    testServer.broadcast('interaction:completed', {
      session_id: 'session-1',
      interaction_id: 'int-1',
      status: 'ok'
    });

    // Assert getNextCommand called
    setTimeout(() => {
      expect(mockClient.sessions.getNextCommand).toHaveBeenCalled();
      done();
    }, 100);
  });

  it('should update local state on session_updated event', async () => {
    await sessionManager.initialize();

    testServer.broadcast('session_updated', {
      session: {
        id: 'session-1',
        external_conversation_id: 'conv_new_123'
      }
    });

    await wait(100);

    const state = sessionStateManager.getSession('session-1');
    expect(state.external_conversation_id).toBe('conv_new_123');
  });
});
```

**HTTP + Executor Integration**
```typescript
describe('Execute and Submit Flow', () => {
  let mockServer: MockServer;
  let orchestrator: SessionOrchestrator;

  beforeAll(() => {
    mockServer = createMockServer();
  });

  it('should execute command and submit result', async () => {
    // Mock executor returns success
    const mockExecutor = {
      execute: jest.fn().mockResolvedValue({
        stdout: 'test output',
        stderr: '',
        exitCode: 0,
        duration_ms: 100
      })
    };

    mockServer.onPost('/api/sessions/session-1/submit-result/int-1', (req, res) => {
      expect(req.body).toMatchObject({
        result: {
          status: 'ok',
          stdout: 'test output'
        }
      });
      res.json({ data: { id: 'session-1' } });
    });

    const session = { id: 'session-1', execution_mode: 'manual' };
    const interaction = { id: 'int-1', command: { command: 'ls' } };

    await orchestrator.executeCommand(session, interaction);

    expect(mockExecutor.execute).toHaveBeenCalled();
    // Assert HTTP POST was made
  });
});
```

---

## Layer 3: End-to-End Tests

### Use Test Doubles for Slow/Unreliable Parts

**Fake Claude Executor**
```typescript
class FakeClaudeExecutor extends ClaudeExecutor {
  async execute(sessionId, interaction, session) {
    // Don't call real Claude API
    // Return canned responses based on prompt
    const prompt = interaction.command.metadata?.prompt || '';

    if (prompt.includes('error')) {
      return {
        stdout: '',
        stderr: 'Simulated error',
        exitCode: 1,
        duration_ms: 10
      };
    }

    // Simulate streaming events
    await this.streamEventToServer(sessionId, interaction.id, {
      type: 'assistant',
      conversation_id: 'test_conv_123',
      message: { content: [{ type: 'text', text: 'Fake response' }] }
    });

    await this.streamEventToServer(sessionId, interaction.id, {
      type: 'result',
      is_error: false
    });

    return {
      stdout: 'Fake response',
      stderr: '',
      exitCode: 0,
      duration_ms: 10
    };
  }
}
```

**Test Complete Flows**
```typescript
describe('End-to-End Session Execution', () => {
  let backend: TestBackend;
  let orchestrator: SessionOrchestrator;

  beforeAll(async () => {
    backend = await TestBackend.start(); // Real Phoenix + HTTP
    orchestrator = new SessionOrchestrator(authProvider);
    orchestrator.executors.claude = new FakeClaudeExecutor(); // Fake Claude
  });

  it('should execute complete session with autoPlay', async () => {
    // Create session on backend
    const session = await backend.createSession({
      type: 'test',
      execution_mode: 'agentic'
    });

    // Add 3 commands
    await backend.addCommand(session.id, { command: 'echo 1' });
    await backend.addCommand(session.id, { command: 'echo 2' });
    await backend.addCommand(session.id, { command: 'echo 3' });

    // Enable autoPlay and start
    orchestrator.setAutoPlay(session.id, true);
    await orchestrator.executeNext(session.id);

    // Wait for all commands to complete
    await waitForSessionComplete(session.id, 10000);

    const finalSession = await backend.getSession(session.id);
    expect(finalSession.status).toBe('complete');
    expect(finalSession.interactions).toHaveLength(3);
    expect(finalSession.interactions.every(i => i.result)).toBe(true);
  });

  it('should handle parallel sessions', async () => {
    const parentSession = await backend.createSession({
      type: 'test',
      execution_mode: 'agentic'
    });

    // Create 5 child sessions
    const childIds = await Promise.all(
      [1, 2, 3, 4, 5].map(i => backend.createSession({ type: 'test' }))
    );

    // Add parallel command to parent
    await backend.addCommand(parentSession.id, {
      command: 'spawn_sessions',
      metadata: { child_session_ids: childIds.map(s => s.id) }
    });

    // Execute
    await orchestrator.executeNext(parentSession.id);

    // Wait for all children to complete
    await waitForSessionComplete(parentSession.id, 30000);

    const result = await backend.getSession(parentSession.id);
    expect(result.status).toBe('complete');

    const children = await Promise.all(childIds.map(s => backend.getSession(s.id)));
    expect(children.every(c => c.status === 'complete')).toBe(true);
  });

  it('should handle failures gracefully', async () => {
    const session = await backend.createSession({
      type: 'test',
      execution_mode: 'agentic'
    });

    // Add command that will fail
    await backend.addCommand(session.id, {
      command: 'claude',
      metadata: { prompt: 'error please' } // FakeClaudeExecutor returns error
    });

    orchestrator.setAutoPlay(session.id, true);
    await orchestrator.executeNext(session.id);

    await wait(1000);

    const finalSession = await backend.getSession(session.id);
    expect(finalSession.interactions[0].result.status).toBe('error');
  });
});
```

---

## Layer 4: Manual Testing Checklist

Only run these AFTER automated tests pass:

### Manual Test Cases

**[ ] Manual Mode - Shell Command**
1. Start session (manual mode)
2. Execute `ls -la`
3. Verify terminal opens and shows output
4. Verify result submitted to server
5. Check session updated in UI

**[ ] Manual Mode - Claude Command**
1. Start session (manual mode)
2. Execute Claude command with prompt
3. Verify terminal opens with Claude CLI
4. Verify hooks fire (check logs)
5. Verify conversation ID captured
6. Execute another Claude command with --resume
7. Verify conversation continues

**[ ] Agentic Mode - Shell Command**
1. Start session (agentic mode)
2. Execute `npm test`
3. Verify NO terminal opens
4. Verify output captured in background
5. Verify result submitted

**[ ] Agentic Mode - Claude Command**
1. Start session (agentic mode)
2. Execute Claude command
3. Verify NO terminal opens
4. Verify events streamed to server (check backend logs)
5. Verify conversation ID captured

**[ ] AutoPlay**
1. Start session with multiple commands
2. Enable autoPlay
3. Verify commands execute automatically
4. Disable autoPlay mid-execution
5. Verify execution stops after current command

**[ ] Parallel Sessions**
1. Create session with 3 child sessions
2. Execute spawn_sessions command
3. Verify all 3 children execute in parallel
4. Verify parent receives aggregated result

**[ ] Error Handling**
1. Execute command that fails (exit code 1)
2. Verify error captured and submitted
3. Execute Claude command with bad prompt
4. Verify error handled gracefully

**[ ] Phoenix Disconnection**
1. Start session with autoPlay
2. Disconnect network
3. Verify execution pauses
4. Reconnect network
5. Verify execution resumes

---

## Test Utilities

### Mock Factories

```typescript
// createMockExecutor.ts
export function createMockExecutor(): jest.Mocked<Executor> {
  return {
    execute: jest.fn().mockResolvedValue({
      stdout: '',
      stderr: '',
      exitCode: 0,
      duration_ms: 0
    }),
    cleanupSession: jest.fn()
  };
}

// createTestSession.ts
export function createTestSession(overrides?: Partial<Session>): Session {
  return {
    id: 'test-session-1',
    user_id: 'test-user',
    type: 'test',
    status: 'active',
    execution_mode: 'manual',
    interactions: [],
    created_at: new Date().toISOString(),
    updated_at: new Date().toISOString(),
    ...overrides
  };
}

// waitForCondition.ts
export async function waitForCondition(
  condition: () => boolean | Promise<boolean>,
  timeout: number = 5000
): Promise<void> {
  const start = Date.now();
  while (Date.now() - start < timeout) {
    if (await condition()) return;
    await new Promise(resolve => setTimeout(resolve, 100));
  }
  throw new Error('Timeout waiting for condition');
}
```

### Test Backend Server

```typescript
// testBackend.ts
export class TestBackend {
  private app: Express;
  private phoenixServer: PhoenixTestServer;

  async start() {
    this.app = express();
    this.phoenixServer = new PhoenixTestServer();

    // Set up routes that match real backend
    this.setupRoutes();

    await this.phoenixServer.start();
    return this;
  }

  async createSession(params: CreateSessionParams): Promise<Session> {
    // Create session in memory
    // Return session object
  }

  async addCommand(sessionId: string, command: Command): Promise<void> {
    // Add interaction to session
    // Emit interaction:started Phoenix event
  }

  async getSession(sessionId: string): Promise<Session> {
    // Return session with all interactions and results
  }

  broadcast(event: string, payload: any): void {
    this.phoenixServer.broadcast(`vscode:user:test-user`, event, payload);
  }
}
```

---

## Running Tests

### Fast Feedback Loop

```bash
# Unit tests only (< 5 seconds)
npm run test:unit

# Integration tests (< 30 seconds)
npm run test:integration

# E2E tests (< 2 minutes)
npm run test:e2e

# All automated tests
npm run test

# Watch mode for development
npm run test:watch
```

### CI Pipeline

```yaml
# .github/workflows/test.yml
jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - run: npm run test:unit

  integration:
    runs-on: ubuntu-latest
    needs: unit
    steps:
      - run: npm run test:integration

  e2e:
    runs-on: ubuntu-latest
    needs: integration
    steps:
      - run: npm run test:e2e
```

---

## Coverage Goals

- **Unit Tests**: 90%+ coverage of all executors, orchestrator, callback server
- **Integration Tests**: All Phoenix event flows, all HTTP endpoints
- **E2E Tests**: All execution modes, autoPlay, parallel sessions, error cases

---

## Benefits

1. **Fast Feedback**: Unit tests run in seconds
2. **No Manual Clicking**: Automated test doubles for slow operations
3. **Regression Prevention**: Tests catch breaks immediately
4. **Documentation**: Tests show how components should work
5. **Confidence**: Can refactor without fear
6. **Parallel Execution**: Tests run concurrently
7. **CI Integration**: Automatic verification on every commit

---

## Summary

**Stop manually testing.** Build test doubles for Claude, use real backend for integration tests, mock everything else. Run fast unit tests constantly, integration tests before committing, E2E tests before releasing. Manual testing only for final UX verification.

**Time Investment:**
- Building test infrastructure: 2-3 days
- Writing tests as you implement: +30% development time
- **Payoff**: 10x faster iteration, zero manual testing, catch bugs immediately
