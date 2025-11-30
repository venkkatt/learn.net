# Secure OAuth 2.0 PKCE Implementation in .NET for SPAs with Azure Entra ID
## Complete End-to-End Guide with Backend-for-Frontend (BFF) Pattern

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Browser (SPA)                              │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ Frontend (React/Angular)                              │   │
│  │ - No token storage in JS                              │   │
│  │ - Makes requests to BFF backend                       │   │
│  │ - Cookies handled automatically                       │   │
│  └─────────────────────┬──────────────────────────────────┘   │
└──────────────────────────┼─────────────────────────────────────┘
                           │ HTTP/HTTPS (Same-origin)
                           │
┌──────────────────────────▼─────────────────────────────────────┐
│           .NET Backend-for-Frontend (BFF)                      │
│  ┌────────────────────────────────────────────────────────┐   │
│  │ - OAuth client (confidential)                          │   │
│  │ - Handles token exchange                              │   │
│  │ - Stores tokens server-side                           │   │
│  │ - Issues HttpOnly/Secure cookies                      │   │
│  │ - Validates all requests                              │   │
│  │ - Forwards requests to backend API                    │   │
│  └──────────┬──────────────────────────────┬──────────────┘   │
└─────────────┼──────────────────────────────┼──────────────────┘
              │                              │
    ┌─────────▼────────────────────────┐    │
    │   Azure Entra ID                 │    │
    │ /oauth2/v2.0/authorize           │    │
    │ /oauth2/v2.0/token               │    │
    │ /oauth2/v2.0/logout              │    │
    └──────────────────────────────────┘    │
                                            │
                                 ┌──────────▼──────────────┐
                                 │  Backend API Service   │
                                 │  (Protected Resources) │
                                 └───────────────────────┘
```

---

## Step 1: Configure Azure Entra ID Application Registration

### 1.1 Register SPA Application

**Azure Portal → App Registrations → New Registration:**

```
Name: MySecureApp-Frontend
Supported Account Types: Single/Multi-tenant (as needed)
Redirect URI: 
  - Type: Single Page Application (SPA)
  - URI: https://localhost:5173/ (dev), https://yourdomain.com/ (prod)
```

### 1.2 Register Backend API Application

**Azure Portal → App Registrations → New Registration:**

```
Name: MySecureApp-Backend-API
Supported Account Types: Single/Multi-tenant
No redirect URIs needed (resource server)
```

**Expose an API:**
- Application ID URI: `api://your-app-id` or `https://yourtenant.onmicrosoft.com/backend-api`
- Add Scope: `api://your-app-id/ReadWrite`

### 1.3 Register BFF Backend Application

**Azure Portal → App Registrations → New Registration:**

```
Name: MySecureApp-BFF-Backend
Supported Account Types: Single/Multi-tenant
Redirect URI: https://localhost:5000/signin-oidc (dev)
             https://yourdomain.com/signin-oidc (prod)
```

**Certificates & Secrets:**
- Create a client secret (store securely in Key Vault)

**API Permissions:**
- Microsoft Graph: User.Read, offline_access
- Backend API: api://your-app-id/ReadWrite

**Token Configuration:**
- Add optional claims: Email, Profile
- Enable ID token for implicit flow (for BFF)
- Enable access token for hybrid flow

### 1.4 Configure App Manifest (All Apps)

```json
{
  "spa": {
    "redirectUris": [
      "https://localhost:5173/",
      "https://yourdomain.com/"
    ]
  }
}
```

---

## Step 2: Frontend (SPA) Implementation - React/Angular

### 2.1 Frontend Environment Setup

```bash
npm create vite@latest my-secure-app -- --template react
cd my-secure-app
npm install axios zustand
```

### 2.2 Frontend OAuth Service (`src/services/authService.ts`)

```typescript
// src/services/authService.ts

const AUTHORITY = 'https://login.microsoftonline.com/your-tenant-id';
const CLIENT_ID = 'your-spa-client-id';
const REDIRECT_URI = `${window.location.origin}/callback`;
const BFF_BASE_URL = 'http://localhost:5000';

/**
 * PKCE Implementation - Generate code verifier and challenge
 */
function generateCodeVerifier(): string {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~';
  let result = '';
  for (let i = 0; i < 128; i++) {
    result += chars.charAt(Math.floor(Math.random() * chars.length));
  }
  return result;
}

async function generateCodeChallenge(verifier: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const hash = await crypto.subtle.digest('SHA-256', data);
  return btoa(String.fromCharCode(...new Uint8Array(hash)))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=/g, '');
}

function generateState(): string {
  return crypto.getRandomValues(new Uint8Array(32)).toString();
}

/**
 * Initiate login - Redirect to Azure Entra ID
 */
export async function initiateLogin(): Promise<void> {
  const codeVerifier = generateCodeVerifier();
  const codeChallenge = await generateCodeChallenge(codeVerifier);
  const state = generateState();

  // Store in session storage (not localStorage for security)
  sessionStorage.setItem('pkce_verifier', codeVerifier);
  sessionStorage.setItem('oauth_state', state);

  const params = new URLSearchParams({
    client_id: CLIENT_ID,
    response_type: 'code',
    redirect_uri: REDIRECT_URI,
    scope: 'openid profile offline_access',
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    state: state,
    prompt: 'login',
  });

  // Redirect to Azure Entra ID
  window.location.href = `${AUTHORITY}/oauth2/v2.0/authorize?${params.toString()}`;
}

/**
 * Handle callback from Azure Entra ID
 * Exchange authorization code for tokens via BFF backend
 */
export async function handleAuthCallback(): Promise<void> {
  const params = new URLSearchParams(window.location.search);
  const code = params.get('code');
  const state = params.get('state');
  const storedState = sessionStorage.getItem('oauth_state');

  // Validate state parameter
  if (state !== storedState) {
    throw new Error('State parameter mismatch - possible CSRF attack');
  }

  if (!code) {
    throw new Error('No authorization code received');
  }

  // Send code to BFF backend (NOT directly to Azure)
  try {
    const response = await fetch(`${BFF_BASE_URL}/auth/callback`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      credentials: 'include', // Include cookies
      body: JSON.stringify({ code }),
    });

    if (!response.ok) {
      throw new Error('Token exchange failed');
    }

    // Clear session storage
    sessionStorage.removeItem('pkce_verifier');
    sessionStorage.removeItem('oauth_state');

    // Redirect to app (BFF sets HttpOnly cookies)
    window.location.href = '/dashboard';
  } catch (error) {
    console.error('Auth callback error:', error);
    window.location.href = '/login?error=callback_failed';
  }
}

/**
 * Logout
 */
export async function logout(): Promise<void> {
  try {
    await fetch(`${BFF_BASE_URL}/auth/logout`, {
      method: 'POST',
      credentials: 'include',
    });
  } catch (error) {
    console.error('Logout error:', error);
  }

  // Redirect to Azure logout
  const params = new URLSearchParams({
    client_id: CLIENT_ID,
    redirect_uri: window.location.origin,
  });
  window.location.href = `${AUTHORITY}/oauth2/v2.0/logout?${params.toString()}`;
}

/**
 * Get current user info via BFF
 */
export async function getCurrentUser(): Promise<any> {
  const response = await fetch(`${BFF_BASE_URL}/auth/user`, {
    credentials: 'include',
  });

  if (!response.ok) {
    return null;
  }

  return response.json();
}

/**
 * Make authenticated API requests via BFF
 */
export async function makeAuthenticatedRequest(
  endpoint: string,
  options: RequestInit = {}
): Promise<Response> {
  return fetch(`${BFF_BASE_URL}${endpoint}`, {
    ...options,
    credentials: 'include', // Include HttpOnly cookies
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });
}
```

### 2.3 Frontend Auth Context (`src/context/AuthContext.tsx`)

```typescript
import { createContext, useContext, useState, useEffect } from 'react';
import {
  initiateLogin,
  handleAuthCallback,
  logout,
  getCurrentUser,
  makeAuthenticatedRequest,
} from '../services/authService';

interface User {
  id: string;
  email: string;
  name: string;
}

interface AuthContextType {
  user: User | null;
  isLoading: boolean;
  login: () => Promise<void>;
  logout: () => Promise<void>;
  makeRequest: (endpoint: string, options?: RequestInit) => Promise<Response>;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // Initialize on mount
  useEffect(() => {
    async function initAuth() {
      try {
        // Check if returning from OAuth callback
        if (window.location.pathname === '/callback') {
          await handleAuthCallback();
        }

        // Get current user
        const currentUser = await getCurrentUser();
        setUser(currentUser);
      } catch (error) {
        console.error('Auth init error:', error);
        setUser(null);
      } finally {
        setIsLoading(false);
      }
    }

    initAuth();
  }, []);

  const value: AuthContextType = {
    user,
    isLoading,
    login: initiateLogin,
    logout: logout,
    makeRequest: makeAuthenticatedRequest,
  };

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

### 2.4 Frontend Components

```typescript
// src/pages/Login.tsx
import { useAuth } from '../context/AuthContext';

export function LoginPage() {
  const { login } = useAuth();

  return (
    <div className="login-container">
      <h1>Secure Login</h1>
      <button onClick={login}>Sign in with Azure AD</button>
    </div>
  );
}

// src/pages/Callback.tsx
import { useEffect, useState } from 'react';

export function CallbackPage() {
  const [status, setStatus] = useState('Processing login...');

  useEffect(() => {
    // The AuthProvider handles the callback
    const timer = setTimeout(() => {
      setStatus('Redirecting...');
    }, 1000);
    return () => clearTimeout(timer);
  }, []);

  return <div>{status}</div>;
}

// src/pages/Dashboard.tsx
import { useAuth } from '../context/AuthContext';
import { useEffect, useState } from 'react';

export function DashboardPage() {
  const { user, logout, makeRequest } = useAuth();
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function fetchData() {
      try {
        const response = await makeRequest('/api/protected-resource');
        if (!response.ok) throw new Error('Failed to fetch');
        const result = await response.json();
        setData(result);
      } catch (err) {
        setError('Failed to load data');
        console.error(err);
      }
    }

    if (user) {
      fetchData();
    }
  }, [user, makeRequest]);

  if (!user) return <div>Not authenticated</div>;

  return (
    <div className="dashboard">
      <h1>Welcome, {user.name}</h1>
      <p>Email: {user.email}</p>
      <div>{data ? JSON.stringify(data) : error ? error : 'Loading...'}</div>
      <button onClick={logout}>Sign Out</button>
    </div>
  );
}

// src/App.tsx
import { AuthProvider } from './context/AuthContext';
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { LoginPage } from './pages/Login';
import { CallbackPage } from './pages/Callback';
import { DashboardPage } from './pages/Dashboard';

function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <Routes>
          <Route path="/login" element={<LoginPage />} />
          <Route path="/callback" element={<CallbackPage />} />
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="/" element={<Navigate to="/dashboard" />} />
        </Routes>
      </AuthProvider>
    </BrowserRouter>
  );
}

export default App;
```

### 2.5 Frontend CORS & Security Headers (`vite.config.ts`)

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    strictPort: true,
    headers: {
      'Content-Security-Policy': "default-src 'self'; script-src 'self'; connect-src 'self' https://login.microsoftonline.com",
      'X-Content-Type-Options': 'nosniff',
      'X-Frame-Options': 'DENY',
      'X-XSS-Protection': '1; mode=block',
      'Referrer-Policy': 'strict-origin-when-cross-origin',
    },
  },
});
```

---

## Step 3: .NET Backend-for-Frontend (BFF) Implementation

### 3.1 Create .NET Project

```bash
dotnet new webapi -n MySecureApp.BFF
cd MySecureApp.BFF
dotnet add package Microsoft.Identity.Web
dotnet add package Microsoft.Identity.Web.DownstreamApi
dotnet add package Azure.Identity
dotnet add package Azure.Security.KeyVault.Secrets
dotnet add package System.Net.Http.Json
```

### 3.2 Configuration (`appsettings.json`)

```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "your-tenant-id",
    "ClientId": "your-bff-client-id",
    "ClientSecret": "${ClientSecret}",
    "Authority": "https://login.microsoftonline.com/your-tenant-id",
    "TokenEndpoint": "https://login.microsoftonline.com/your-tenant-id/oauth2/v2.0/token"
  },
  "Spa": {
    "ClientId": "your-spa-client-id",
    "RedirectUri": "https://localhost:5173/callback"
  },
  "BackendApi": {
    "Scope": "api://your-backend-api-id/.default"
  },
  "TokenRefreshThreshold": 300,
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

### 3.3 Token Management Service (`Services/TokenService.cs`)

```csharp
using System;
using System.Collections.Generic;
using System.Net.Http;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace MySecureApp.BFF.Services
{
    public interface ITokenService
    {
        Task<TokenResponse> ExchangeAuthorizationCodeAsync(
            string code,
            string redirectUri,
            CancellationToken cancellationToken = default);

        Task<TokenResponse> RefreshAccessTokenAsync(
            string refreshToken,
            CancellationToken cancellationToken = default);

        Task RevokeTokenAsync(
            string token,
            CancellationToken cancellationToken = default);

        bool IsTokenExpiringSoon(DateTime expiresAt, int thresholdSeconds = 300);
    }

    public class TokenResponse
    {
        [JsonPropertyName("access_token")]
        public string AccessToken { get; set; }

        [JsonPropertyName("refresh_token")]
        public string RefreshToken { get; set; }

        [JsonPropertyName("id_token")]
        public string IdToken { get; set; }

        [JsonPropertyName("token_type")]
        public string TokenType { get; set; }

        [JsonPropertyName("expires_in")]
        public int ExpiresIn { get; set; }

        [JsonPropertyName("ext_expires_in")]
        public int ExtExpiresIn { get; set; }

        [JsonPropertyName("scope")]
        public string Scope { get; set; }

        public DateTime ExpiresAt => DateTime.UtcNow.AddSeconds(ExpiresIn);
    }

    public class TokenService : ITokenService
    {
        private readonly HttpClient _httpClient;
        private readonly IConfiguration _configuration;
        private readonly ILogger<TokenService> _logger;

        public TokenService(
            HttpClient httpClient,
            IConfiguration configuration,
            ILogger<TokenService> logger)
        {
            _httpClient = httpClient;
            _configuration = configuration;
            _logger = logger;
        }

        /// <summary>
        /// Exchange authorization code for tokens
        /// This happens on the backend, never exposing tokens to frontend
        /// </summary>
        public async Task<TokenResponse> ExchangeAuthorizationCodeAsync(
            string code,
            string redirectUri,
            CancellationToken cancellationToken = default)
        {
            var tokenEndpoint = _configuration["AzureAd:TokenEndpoint"];
            var clientId = _configuration["AzureAd:ClientId"];
            var clientSecret = _configuration["AzureAd:ClientSecret"];
            var scope = _configuration["BackendApi:Scope"];

            var parameters = new Dictionary<string, string>
            {
                { "grant_type", "authorization_code" },
                { "code", code },
                { "client_id", clientId },
                { "client_secret", clientSecret },
                { "redirect_uri", redirectUri },
                { "scope", scope },
            };

            var content = new FormUrlEncodedContent(parameters);

            try
            {
                _logger.LogInformation("Exchanging authorization code for tokens");

                var response = await _httpClient.PostAsync(
                    tokenEndpoint,
                    content,
                    cancellationToken);

                if (!response.IsSuccessStatusCode)
                {
                    var errorContent = await response.Content.ReadAsStringAsync(cancellationToken);
                    _logger.LogError($"Token exchange failed: {errorContent}");
                    throw new HttpRequestException(
                        $"Token endpoint returned {response.StatusCode}: {errorContent}");
                }

                var responseContent = await response.Content.ReadAsStringAsync(cancellationToken);
                var tokenResponse = JsonSerializer.Deserialize<TokenResponse>(responseContent);

                if (tokenResponse == null)
                    throw new InvalidOperationException("Failed to parse token response");

                _logger.LogInformation("Token exchange successful");
                return tokenResponse;
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error during token exchange: {ex.Message}");
                throw;
            }
        }

        /// <summary>
        /// Refresh access token using refresh token
        /// Implements refresh token rotation (new refresh token issued on each use)
        /// </summary>
        public async Task<TokenResponse> RefreshAccessTokenAsync(
            string refreshToken,
            CancellationToken cancellationToken = default)
        {
            var tokenEndpoint = _configuration["AzureAd:TokenEndpoint"];
            var clientId = _configuration["AzureAd:ClientId"];
            var clientSecret = _configuration["AzureAd:ClientSecret"];
            var scope = _configuration["BackendApi:Scope"];

            var parameters = new Dictionary<string, string>
            {
                { "grant_type", "refresh_token" },
                { "refresh_token", refreshToken },
                { "client_id", clientId },
                { "client_secret", clientSecret },
                { "scope", scope },
            };

            var content = new FormUrlEncodedContent(parameters);

            try
            {
                _logger.LogInformation("Refreshing access token");

                var response = await _httpClient.PostAsync(
                    tokenEndpoint,
                    content,
                    cancellationToken);

                if (!response.IsSuccessStatusCode)
                {
                    var errorContent = await response.Content.ReadAsStringAsync(cancellationToken);
                    _logger.LogError($"Token refresh failed: {errorContent}");
                    throw new HttpRequestException($"Token refresh failed: {errorContent}");
                }

                var responseContent = await response.Content.ReadAsStringAsync(cancellationToken);
                var tokenResponse = JsonSerializer.Deserialize<TokenResponse>(responseContent);

                if (tokenResponse == null)
                    throw new InvalidOperationException("Failed to parse token response");

                _logger.LogInformation("Token refresh successful - new refresh token issued");
                return tokenResponse;
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error during token refresh: {ex.Message}");
                throw;
            }
        }

        /// <summary>
        /// Revoke tokens on logout
        /// </summary>
        public async Task RevokeTokenAsync(
            string token,
            CancellationToken cancellationToken = default)
        {
            var revokeEndpoint = _configuration["AzureAd:Instance"] +
                                _configuration["AzureAd:TenantId"] +
                                "/oauth2/v2.0/revoke";
            var clientId = _configuration["AzureAd:ClientId"];
            var clientSecret = _configuration["AzureAd:ClientSecret"];

            var parameters = new Dictionary<string, string>
            {
                { "client_id", clientId },
                { "client_secret", clientSecret },
                { "token", token },
            };

            var content = new FormUrlEncodedContent(parameters);

            try
            {
                _logger.LogInformation("Revoking token");
                await _httpClient.PostAsync(revokeEndpoint, content, cancellationToken);
                _logger.LogInformation("Token revoked successfully");
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error during token revocation: {ex.Message}");
                // Don't throw - logout should succeed even if revocation fails
            }
        }

        /// <summary>
        /// Check if token is expiring soon (default: within 5 minutes)
        /// </summary>
        public bool IsTokenExpiringSoon(DateTime expiresAt, int thresholdSeconds = 300)
        {
            var timeUntilExpiration = expiresAt - DateTime.UtcNow;
            return timeUntilExpiration.TotalSeconds < thresholdSeconds;
        }
    }
}
```

### 3.4 Session/Token Storage Service (`Services/SessionTokenStore.cs`)

```csharp
using System;
using System.Collections.Concurrent;
using System.Text.Json;
using Microsoft.Extensions.Logging;

namespace MySecureApp.BFF.Services
{
    /// <summary>
    /// Server-side token storage associated with user sessions
    /// Tokens NEVER sent to client in response bodies
    /// </summary>
    public interface ISessionTokenStore
    {
        void StoreTokens(string sessionId, TokenResponse tokens);
        TokenResponse? GetTokens(string sessionId);
        void UpdateTokens(string sessionId, TokenResponse tokens);
        void ClearTokens(string sessionId);
        void ClearExpiredSessions();
    }

    public class SessionTokenStore : ISessionTokenStore
    {
        // In production, use distributed cache (Redis) instead
        private readonly ConcurrentDictionary<string, SessionData> _store =
            new();

        private readonly ILogger<SessionTokenStore> _logger;

        private class SessionData
        {
            public TokenResponse Tokens { get; set; }
            public DateTime LastAccessedAt { get; set; }
        }

        private const int SessionTimeoutMinutes = 30;

        public SessionTokenStore(ILogger<SessionTokenStore> logger)
        {
            _logger = logger;
        }

        public void StoreTokens(string sessionId, TokenResponse tokens)
        {
            _store[sessionId] = new SessionData
            {
                Tokens = tokens,
                LastAccessedAt = DateTime.UtcNow,
            };
            _logger.LogInformation($"Tokens stored for session {sessionId}");
        }

        public TokenResponse? GetTokens(string sessionId)
        {
            if (_store.TryGetValue(sessionId, out var data))
            {
                data.LastAccessedAt = DateTime.UtcNow;
                return data.Tokens;
            }
            return null;
        }

        public void UpdateTokens(string sessionId, TokenResponse tokens)
        {
            if (_store.TryGetValue(sessionId, out var data))
            {
                data.Tokens = tokens;
                data.LastAccessedAt = DateTime.UtcNow;
                _logger.LogInformation($"Tokens updated for session {sessionId}");
            }
        }

        public void ClearTokens(string sessionId)
        {
            _store.TryRemove(sessionId, out _);
            _logger.LogInformation($"Tokens cleared for session {sessionId}");
        }

        public void ClearExpiredSessions()
        {
            var timeout = TimeSpan.FromMinutes(SessionTimeoutMinutes);
            var expiredKeys = _store
                .Where(x => DateTime.UtcNow - x.Value.LastAccessedAt > timeout)
                .Select(x => x.Key)
                .ToList();

            foreach (var key in expiredKeys)
            {
                _store.TryRemove(key, out _);
                _logger.LogInformation($"Expired session removed: {key}");
            }
        }
    }
}
```

### 3.5 Authentication Controller (`Controllers/AuthController.cs`)

```csharp
using System;
using System.Collections.Generic;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;
using MySecureApp.BFF.Services;

namespace MySecureApp.BFF.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class AuthController : ControllerBase
    {
        private readonly ITokenService _tokenService;
        private readonly ISessionTokenStore _tokenStore;
        private readonly IConfiguration _configuration;
        private readonly ILogger<AuthController> _logger;

        public AuthController(
            ITokenService tokenService,
            ISessionTokenStore tokenStore,
            IConfiguration configuration,
            ILogger<AuthController> logger)
        {
            _tokenService = tokenService;
            _tokenStore = tokenStore;
            _configuration = configuration;
            _logger = logger;
        }

        /// <summary>
        /// Handle OAuth callback from Azure Entra ID
        /// Exchange authorization code for tokens
        /// Store tokens server-side in session
        /// Issue HttpOnly cookie to client
        /// </summary>
        [HttpPost("callback")]
        public async Task<IActionResult> Callback([FromBody] CallbackRequest request)
        {
            if (string.IsNullOrEmpty(request.Code))
            {
                _logger.LogWarning("Callback received without authorization code");
                return BadRequest("Authorization code is required");
            }

            try
            {
                var redirectUri = _configuration["Spa:RedirectUri"];

                // Exchange authorization code for tokens
                var tokens = await _tokenService.ExchangeAuthorizationCodeAsync(
                    request.Code,
                    redirectUri);

                // Generate session ID
                var sessionId = Guid.NewGuid().ToString();

                // Store tokens server-side (NOT in client)
                _tokenStore.StoreTokens(sessionId, tokens);

                // Parse ID token to get user claims
                var handler = new JwtSecurityTokenHandler();
                var token = handler.ReadJwtToken(tokens.IdToken);

                // Set secure, HttpOnly cookie with session ID
                Response.Cookies.Append(
                    ".AspNetCore.BFF.Session",
                    sessionId,
                    new CookieOptions
                    {
                        HttpOnly = true,
                        Secure = true,
                        SameSite = SameSiteMode.Lax,
                        Expires = DateTimeOffset.UtcNow.AddHours(24),
                        Path = "/",
                        IsEssential = true,
                    });

                _logger.LogInformation($"Authentication successful for user, session {sessionId} created");

                return Ok(new { message = "Authentication successful" });
            }
            catch (HttpRequestException ex)
            {
                _logger.LogError($"Token exchange failed: {ex.Message}");
                return Unauthorized("Token exchange failed");
            }
            catch (Exception ex)
            {
                _logger.LogError($"Unexpected error in callback: {ex.Message}");
                return StatusCode(500, "Internal server error");
            }
        }

        /// <summary>
        /// Get current user info
        /// Validate session and return user claims from stored tokens
        /// </summary>
        [HttpGet("user")]
        public IActionResult GetUser()
        {
            try
            {
                if (!Request.Cookies.TryGetValue(".AspNetCore.BFF.Session", out var sessionId))
                {
                    return Unauthorized("Session not found");
                }

                var tokens = _tokenStore.GetTokens(sessionId);
                if (tokens == null)
                {
                    return Unauthorized("Tokens not found");
                }

                // Parse ID token
                var handler = new JwtSecurityTokenHandler();
                var token = handler.ReadJwtToken(tokens.IdToken);

                var user = new
                {
                    id = token.Claims.FirstOrDefault(c => c.Type == "oid")?.Value,
                    email = token.Claims.FirstOrDefault(c => c.Type == "email")?.Value,
                    name = token.Claims.FirstOrDefault(c => c.Type == "name")?.Value,
                    claims = token.Claims.Select(c => new { c.Type, c.Value }).ToList(),
                };

                return Ok(user);
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error getting user: {ex.Message}");
                return Unauthorized();
            }
        }

        /// <summary>
        /// Refresh access token
        /// Implements refresh token rotation
        /// </summary>
        [HttpPost("refresh")]
        public async Task<IActionResult> RefreshToken()
        {
            try
            {
                if (!Request.Cookies.TryGetValue(".AspNetCore.BFF.Session", out var sessionId))
                {
                    return Unauthorized("Session not found");
                }

                var tokens = _tokenStore.GetTokens(sessionId);
                if (tokens?.RefreshToken == null)
                {
                    return Unauthorized("No refresh token found");
                }

                // Check if access token is expiring soon
                if (!_tokenService.IsTokenExpiringSoon(tokens.ExpiresAt))
                {
                    return Ok(new { message = "Token still valid" });
                }

                // Refresh tokens
                var newTokens = await _tokenService.RefreshAccessTokenAsync(tokens.RefreshToken);

                // Update stored tokens (refresh token rotation)
                _tokenStore.UpdateTokens(sessionId, newTokens);

                _logger.LogInformation($"Tokens refreshed for session {sessionId}");

                return Ok(new { message = "Tokens refreshed" });
            }
            catch (HttpRequestException ex)
            {
                _logger.LogError($"Token refresh failed: {ex.Message}");
                return Unauthorized("Token refresh failed");
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error refreshing token: {ex.Message}");
                return StatusCode(500, "Internal server error");
            }
        }

        /// <summary>
        /// Logout - Revoke tokens and clear session
        /// </summary>
        [HttpPost("logout")]
        public async Task<IActionResult> Logout()
        {
            try
            {
                if (!Request.Cookies.TryGetValue(".AspNetCore.BFF.Session", out var sessionId))
                {
                    return Ok(new { message = "Already logged out" });
                }

                var tokens = _tokenStore.GetTokens(sessionId);
                if (tokens != null)
                {
                    // Revoke refresh token
                    await _tokenService.RevokeTokenAsync(tokens.RefreshToken);
                }

                // Clear tokens from server
                _tokenStore.ClearTokens(sessionId);

                // Clear session cookie
                Response.Cookies.Delete(".AspNetCore.BFF.Session");

                _logger.LogInformation($"User logged out, session {sessionId} cleared");

                return Ok(new { message = "Logged out successfully" });
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error during logout: {ex.Message}");
                return StatusCode(500, "Logout failed");
            }
        }
    }

    public class CallbackRequest
    {
        public string Code { get; set; }
    }
}
```

### 3.6 Protected API Proxy Middleware (`Middleware/TokenInjectingHttpClientHandler.cs`)

```csharp
using System;
using System.Net.Http;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using MySecureApp.BFF.Services;

namespace MySecureApp.BFF.Middleware
{
    /// <summary>
    /// HTTP client handler that injects access tokens into downstream API calls
    /// Implements token refresh if needed before making request
    /// </summary>
    public class TokenInjectingHttpClientHandler : DelegatingHandler
    {
        private readonly IHttpContextAccessor _httpContextAccessor;
        private readonly ISessionTokenStore _tokenStore;
        private readonly ITokenService _tokenService;
        private readonly ILogger<TokenInjectingHttpClientHandler> _logger;

        public TokenInjectingHttpClientHandler(
            IHttpContextAccessor httpContextAccessor,
            ISessionTokenStore tokenStore,
            ITokenService tokenService,
            ILogger<TokenInjectingHttpClientHandler> logger)
        {
            _httpContextAccessor = httpContextAccessor;
            _tokenStore = tokenStore;
            _tokenService = tokenService;
            _logger = logger;
        }

        protected override async Task<HttpResponseMessage> SendAsync(
            HttpRequestMessage request,
            CancellationToken cancellationToken)
        {
            try
            {
                var httpContext = _httpContextAccessor.HttpContext;
                if (httpContext == null)
                {
                    _logger.LogError("HttpContext is null");
                    return await base.SendAsync(request, cancellationToken);
                }

                if (!httpContext.Request.Cookies.TryGetValue(
                        ".AspNetCore.BFF.Session",
                        out var sessionId))
                {
                    _logger.LogWarning("Session cookie not found");
                    return await base.SendAsync(request, cancellationToken);
                }

                var tokens = _tokenStore.GetTokens(sessionId);
                if (tokens == null)
                {
                    _logger.LogWarning("Tokens not found for session");
                    return await base.SendAsync(request, cancellationToken);
                }

                // Check if token needs refresh
                if (_tokenService.IsTokenExpiringSoon(tokens.ExpiresAt))
                {
                    _logger.LogInformation("Access token expiring soon, refreshing...");
                    try
                    {
                        var newTokens = await _tokenService.RefreshAccessTokenAsync(
                            tokens.RefreshToken,
                            cancellationToken);
                        _tokenStore.UpdateTokens(sessionId, newTokens);
                        tokens = newTokens;
                    }
                    catch (Exception ex)
                    {
                        _logger.LogError($"Failed to refresh token: {ex.Message}");
                        // Continue with existing token
                    }
                }

                // Add authorization header
                request.Headers.Authorization = new System.Net.Http.Headers.AuthenticationHeaderValue(
                    "Bearer",
                    tokens.AccessToken);

                _logger.LogInformation("Authorization header injected");

                return await base.SendAsync(request, cancellationToken);
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error in token injection: {ex.Message}");
                return await base.SendAsync(request, cancellationToken);
            }
        }
    }
}
```

### 3.7 API Proxy Controller (`Controllers/ApiController.cs`)

```csharp
using System;
using System.Net.Http;
using System.Text;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Logging;

namespace MySecureApp.BFF.Controllers
{
    /// <summary>
    /// BFF acts as proxy to backend API
    /// Forwards requests with injected access tokens
    /// Clients never see the tokens
    /// </summary>
    [ApiController]
    [Route("api")]
    public class ApiController : ControllerBase
    {
        private readonly HttpClient _httpClient;
        private readonly IConfiguration _configuration;
        private readonly ILogger<ApiController> _logger;

        public ApiController(
            HttpClient httpClient,
            IConfiguration configuration,
            ILogger<ApiController> logger)
        {
            _httpClient = httpClient;
            _configuration = configuration;
            _logger = logger;
        }

        /// <summary>
        /// Proxy GET requests to backend API
        /// </summary>
        [HttpGet("{*path}")]
        public async Task<IActionResult> GetAsync(string path)
        {
            try
            {
                var backendUrl = _configuration["BackendApi:BaseUrl"];
                var fullUrl = $"{backendUrl}/{path}";

                _logger.LogInformation($"Proxying GET request to {fullUrl}");

                var response = await _httpClient.GetAsync(fullUrl);

                // Return response (access token was injected by middleware)
                var content = await response.Content.ReadAsStringAsync();
                return StatusCode((int)response.StatusCode, content);
            }
            catch (HttpRequestException ex)
            {
                _logger.LogError($"Backend API error: {ex.Message}");
                return StatusCode(502, "Backend API unavailable");
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error in proxy request: {ex.Message}");
                return StatusCode(500, "Internal server error");
            }
        }

        /// <summary>
        /// Proxy POST requests to backend API
        /// </summary>
        [HttpPost("{*path}")]
        public async Task<IActionResult> PostAsync(string path, [FromBody] object body)
        {
            try
            {
                var backendUrl = _configuration["BackendApi:BaseUrl"];
                var fullUrl = $"{backendUrl}/{path}";

                _logger.LogInformation($"Proxying POST request to {fullUrl}");

                var json = System.Text.Json.JsonSerializer.Serialize(body);
                var content = new StringContent(json, Encoding.UTF8, "application/json");

                var response = await _httpClient.PostAsync(fullUrl, content);
                var responseContent = await response.Content.ReadAsStringAsync();

                return StatusCode((int)response.StatusCode, responseContent);
            }
            catch (HttpRequestException ex)
            {
                _logger.LogError($"Backend API error: {ex.Message}");
                return StatusCode(502, "Backend API unavailable");
            }
            catch (Exception ex)
            {
                _logger.LogError($"Error in proxy request: {ex.Message}");
                return StatusCode(500, "Internal server error");
            }
        }
    }
}
```

### 3.8 Startup Configuration (`Program.cs`)

```csharp
using MySecureApp.BFF.Services;
using MySecureApp.BFF.Middleware;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Cors.Infrastructure;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = WebApplicationBuilder.CreateBuilder(args);

// Add services
builder.Services.AddHttpClient<ITokenService, TokenService>();
builder.Services.AddScoped<ISessionTokenStore, SessionTokenStore>();
builder.Services.AddScoped<TokenInjectingHttpClientHandler>();

builder.Services
    .AddHttpClient("BackendApi")
    .AddHttpMessageHandler<TokenInjectingHttpClientHandler>();

builder.Services.AddHttpContextAccessor();

// CORS - Allow SPA frontend only
builder.Services.AddCors(options =>
{
    options.AddPolicy("SpaPolicy", policy =>
    {
        policy
            .WithOrigins(
                "http://localhost:5173",
                "https://yourdomain.com",
                "https://www.yourdomain.com")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials() // Required for cookies
            .WithExposedHeaders("Content-Disposition");
    });
});

builder.Services.AddControllers();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// HTTPS redirect in production
if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();
    app.UseHsts(); // HTTP Strict Transport Security
}

// Security headers middleware
app.Use(async (context, next) =>
{
    context.Response.Headers["X-Content-Type-Options"] = "nosniff";
    context.Response.Headers["X-Frame-Options"] = "DENY";
    context.Response.Headers["X-XSS-Protection"] = "1; mode=block";
    context.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
    context.Response.Headers["Permissions-Policy"] = "geolocation=(), microphone=(), camera=()";
    
    // Remove server header
    context.Response.Headers.Remove("Server");
    context.Response.Headers.Remove("X-Powered-By");
    
    await next();
});

app.UseRouting();

app.UseCors("SpaPolicy");

// Token cleanup background job
var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();
lifetime.ApplicationStarted.Register(() =>
{
    var timer = new System.Timers.Timer(60000); // Run every minute
    timer.Elapsed += (s, e) =>
    {
        using var scope = app.Services.CreateScope();
        var tokenStore = scope.ServiceProvider.GetRequiredService<ISessionTokenStore>();
        tokenStore.ClearExpiredSessions();
    };
    timer.Start();
});

app.MapControllers();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.Run();
```

---

## Step 4: Backend API Implementation

### 4.1 Protected API Controller

```csharp
using System.Collections.Generic;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Linq;
using System.Security.Claims;

namespace MySecureApp.API.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    [Authorize] // Requires valid JWT from BFF
    public class ProtectedResourceController : ControllerBase
    {
        [HttpGet]
        public IActionResult GetResource()
        {
            // Get user ID from JWT claims
            var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            var userEmail = User.FindFirst("email")?.Value;

            var resource = new
            {
                message = "This is protected data",
                userId = userId,
                userEmail = userEmail,
                timestamp = System.DateTime.UtcNow,
                data = new[] { "Item 1", "Item 2", "Item 3" },
            };

            return Ok(resource);
        }
    }
}
```

---

## Step 5: Security Best Practices Implementation

### 5.1 CORS Configuration

**Already implemented in Step 3.8**, but key points:

```csharp
✓ AllowCredentials() - Required for cookies
✓ Specific origins - Never use "*" with credentials
✓ Specific methods - Only required methods
✓ ExposedHeaders - Only necessary headers
```

### 5.2 Content Security Policy (CSP)

Add to BFF `Program.cs`:

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers["Content-Security-Policy"] = 
        "default-src 'self'; " +
        "script-src 'self'; " +
        "style-src 'self' 'nonce-{random}'; " +
        "font-src 'self'; " +
        "img-src 'self' data: https:; " +
        "connect-src 'self' https://login.microsoftonline.com; " +
        "frame-ancestors 'none'; " +
        "base-uri 'self'; " +
        "form-action 'self'";

    await next();
});
```

### 5.3 Token Validation

```csharp
// In BFF startup
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = configuration["AzureAd:Authority"];
        options.Audience = configuration["AzureAd:ClientId"];
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromMinutes(5),
        };
    });
```

### 5.4 Rate Limiting

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("auth-limiter", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 10;
        opt.QueueLimit = 0;
    });
});

// Apply to auth endpoints
[Authorize(Policy = "RateLimit")]
[HttpPost("refresh")]
public async Task<IActionResult> RefreshToken() { ... }
```

---

## Step 6: Deployment Checklist

### Security Configuration

- [ ] HTTPS everywhere (self-signed for dev, valid cert for prod)
- [ ] HSTS preloading enabled
- [ ] Secrets in Azure Key Vault (not in code)
- [ ] SameSite=Lax on cookies
- [ ] HttpOnly + Secure flags on session cookies
- [ ] Token expiration: 5-15 minutes (access), 24 hours (refresh)
- [ ] Exact redirect URI matching
- [ ] CORS limited to specific origins
- [ ] CSP headers configured
- [ ] Rate limiting on auth endpoints
- [ ] Token introspection/revocation implemented

### Monitoring

- [ ] Log all authentication events
- [ ] Alert on failed token exchanges
- [ ] Monitor token refresh frequency
- [ ] Track session creation/deletion
- [ ] Audit logs for sensitive operations

---

## Step 7: Testing Flow

### Manual Testing Sequence

```bash
# 1. Start services
dotnet run --project MySecureApp.BFF      # Port 5000
npm run dev                                 # Port 5173 (SPA)

# 2. Open browser
http://localhost:5173

# 3. Click "Sign in with Azure AD"
# - Redirected to Azure login
# - Browser returns to http://localhost:5173/callback?code=...
# - Frontend POST to http://localhost:5000/api/auth/callback
# - BFF exchanges code for tokens (tokens stored server-side)
# - BFF sets HttpOnly cookie
# - Frontend redirected to /dashboard

# 4. Verify in DevTools
# - Network tab: Cookie present, no tokens visible
# - Storage tab: No localStorage/sessionStorage tokens
# - Make API call: Authorization header auto-injected via middleware

# 5. Test token refresh (wait >5 minutes or force refresh)
# - Next API call triggers auto-refresh
# - New tokens issued server-side
# - New refresh token issued (rotation)

# 6. Logout
# - Session cleared
# - Cookie deleted
# - Tokens revoked at Azure
```

---

## Security Comparison

| Attack Vector | Risk | Mitigation |
|---|---|---|
| Authorization Code Interception | Medium | PKCE (code verifier binding) |
| Token Theft from Browser Storage | High | No tokens in JS (server-side storage) |
| XSS Stealing Tokens | High | HttpOnly cookies (immune to JS access) |
| CSRF | Medium | State/CSRF tokens + SameSite cookies |
| Token Replay | Medium | Short expiration + refresh rotation |
| Man-in-Middle | Medium | HTTPS + HSTS preloading |
| Phishing | Medium | Authorization server responsibility + MFA |
| Over-Privileging | Low | Narrow scopes (offline_access + specific API scopes) |
| Redirect URI Bypass | Medium | Exact URI matching + app manifest validation |
| Session Fixation | Low | Secure session ID generation + HttpOnly |

---

## Production Considerations

1. **Token Storage**: Use Redis/distributed cache instead of in-memory ConcurrentDictionary
2. **Load Balancing**: Session affinity OR distributed token store
3. **Key Vault**: Store client secrets in Azure Key Vault
4. **Logging**: Use Application Insights for telemetry
5. **Encryption**: Encrypt tokens in session store
6. **API Gateway**: Consider API Management for rate limiting and versioning
7. **Health Checks**: Implement /health endpoints for load balancers

---

## References

- [Microsoft Identity Platform - OAuth 2.0 Auth Code Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow)
- [OAuth 2.0 for Browser-Based Apps (BFF Pattern)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps)
- [OWASP OAuth 2.0 Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html)
- [Azure AD B2C PKCE and Refresh Tokens](https://condatis.com/news/blog/microsoft-azure-ad-b2c-and-refresh-tokens-for-single-page-applications/)
