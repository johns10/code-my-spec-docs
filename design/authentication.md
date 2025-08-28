# Authentication Context Design

## Overview

This document outlines the authentication system for the Code My Spec VS Code extension, which needs to authenticate with a Phoenix LiveView application using OAuth2 for accessing MCP (Model Context Protocol) servers and potentially REST APIs.

## Phoenix App OAuth2 Endpoints

Based on the router configuration, the Phoenix app exposes:

### Discovery Endpoints
- `GET /.well-known/oauth-authorization-server` - Authorization server metadata (RFC 8414)
- `GET /.well-known/oauth-protected-resource` - Protected resource metadata (RFC 9728)

### OAuth2 Flow Endpoints
- `GET /oauth/authorize` - Authorization page
- `POST /oauth/authorize` - Authorization decision
- `DELETE /oauth/authorize` - Authorization denial
- `POST /oauth/token` - Token exchange and refresh
- `POST /oauth/revoke` - Token revocation
- `POST /oauth/register` - Dynamic client registration

## VS Code Extension Authentication Context

### Core Components

#### 1. AuthenticationProvider
```typescript
interface AuthenticationProvider {
  // OAuth2 PKCE flow
  startAuthFlow(): Promise<AuthResult>
  refreshToken(): Promise<string>
  revokeToken(): Promise<void>
  
  // Token management
  getAccessToken(): Promise<string | null>
  isAuthenticated(): Promise<boolean>
  
  // Events
  onDidChangeAuthenticationStatus: Event<AuthStatusChange>
}
```

#### 2. TokenManager
```typescript
interface TokenManager {
  storeTokens(tokens: TokenSet): Promise<void>
  getTokens(): Promise<TokenSet | null>
  clearTokens(): Promise<void>
  
  // Automatic refresh handling
  getValidAccessToken(): Promise<string | null>
}
```

#### 3. HttpClient
```typescript
interface HttpClient {
  // Authenticated requests to Phoenix app
  get<T>(path: string): Promise<T>
  post<T>(path: string, data?: any): Promise<T>
  put<T>(path: string, data?: any): Promise<T>
  delete<T>(path: string): Promise<T>
  
  // WebSocket/Phoenix Channel support
  createChannel(topic: string): PhoenixChannel
}
```

### Authentication Flow

#### 1. Dynamic Client Registration
1. Extension starts and checks for stored client credentials
2. If none exist, register with `/oauth/register`:
   ```json
   {
     "client_name": "Code My Spec VS Code Extension",
     "redirect_uris": ["vscode://publisher.extension-name/oauth-callback"],
     "grant_types": ["authorization_code"],
     "response_types": ["code"],
     "scope": "read write"
   }
   ```
3. Store `client_id` and `client_secret` in VS Code's SecretStorage

#### 2. OAuth2 PKCE Authorization Flow
1. Generate PKCE code verifier and challenge
2. Open system browser to `/oauth/authorize` with:
   - `client_id`
   - `redirect_uri`
   - `scope=read write`
   - `response_type=code`
   - `code_challenge` (PKCE)
   - `code_challenge_method=S256`
   - `state` (CSRF protection)

3. Handle redirect via VS Code URI handler
4. Exchange authorization code for tokens at `/oauth/token`
5. Store tokens securely using VS Code's SecretStorage

#### 3. Token Management
- Store access token, refresh token, and expiry
- Automatically refresh tokens before expiry
- Handle token revocation and re-authentication
- Clear tokens on logout or revocation

### Integration Points

#### VS Code Extension APIs
- `vscode.authentication` - For auth provider registration
- `vscode.env.openExternal()` - For opening browser
- `vscode.window.registerUriHandler()` - For OAuth callback
- `context.secrets` - For secure token storage
- `context.globalState` - For client credentials

#### Phoenix App Integration
- Standard OAuth2 flows with PKCE
- MCP server access via `/mcp/*` endpoints
- Potential REST API access for other resources
- WebSocket authentication for Phoenix Channels

### Security Considerations

1. **PKCE Flow**: Use code challenge/verifier to prevent authorization code interception
2. **State Parameter**: Prevent CSRF attacks during OAuth flow
3. **Secure Storage**: Use VS Code's SecretStorage for tokens
4. **Token Rotation**: Implement automatic token refresh
5. **Scope Limitation**: Request minimal required scopes
6. **URL Validation**: Validate redirect URIs to prevent hijacking

### Configuration

```typescript
interface AuthConfig {
  serverUrl: string          // Phoenix app base URL
  clientName: string         // For dynamic registration
  redirectUri: string        // VS Code extension URI
  scopes: string[]          // OAuth scopes to request
  
  // Optional overrides
  clientId?: string         // If pre-registered
  clientSecret?: string     // If pre-registered
}
```

### Error Handling

- Network connectivity issues
- Token expiry and refresh failures
- OAuth flow interruption or denial
- Server-side authentication errors
- Invalid or revoked tokens

### Usage Example

```typescript
// In extension activation
const authProvider = new PhoenixAuthProvider(config)
const httpClient = new AuthenticatedHttpClient(authProvider)

// Authenticate user
await authProvider.startAuthFlow()

// Make authenticated requests
const stories = await httpClient.get('/mcp/stories')
const channel = httpClient.createChannel('project:123')
```

This design provides a secure, standard-compliant OAuth2 integration that supports both MCP server access and potential future REST API or Phoenix Channel usage.