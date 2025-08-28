# HTTP Client Design

## Overview
Manual HTTP client implementation for SessionsController endpoints. Avoiding code generation due to small API surface area.

## Architecture

### Base HTTP Client
- Handle authentication (scope-based)
- Base URL configuration
- Common headers and request/response handling
- Standardized error handling

### Sessions Client
Typed methods for SessionsController endpoints:
- `getSession(id: string): Promise<Session>`
- `createSession(params: CreateSessionParams): Promise<Session>`
- `getNextCommand(sessionId: string): Promise<CommandResponse | CompletionResponse>`
- `submitResult(sessionId: string, interactionId: string, result: ResultParams): Promise<Session>`

## Type Definitions

### Request Types
```typescript
interface CreateSessionParams {
  session: {
    // Define based on backend expectations
  }
}

interface ResultParams {
  status: 'ok' | 'error' | 'warning'
  data?: any
  code?: string
  error_message?: string
  stdout?: string
  stderr?: string
  duration_ms?: number
}
```

### Response Types
```typescript
interface Session {
  id: string
  // Other session fields
}

interface CommandResponse {
  interaction_id: string
  command: any
}

interface CompletionResponse {
  status: 'complete' | 'failed'
  message: string
}
```

## Implementation Strategy
1. Use VSCode's native fetch capabilities
2. Manual implementation for full control
3. Strong TypeScript typing
4. Comprehensive error handling for all response scenarios
5. Scope-based authentication handling

## Error Handling
- Network errors
- HTTP status errors
- Validation errors from backend
- Authentication failures
- Timeout handling