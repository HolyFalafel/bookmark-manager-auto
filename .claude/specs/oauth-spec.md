# OAuth Authentication Specification

## Overview

Add OAuth 2.0 authentication to the bookmark manager to enable:
- Multi-device bookmark synchronization
- Secure user authentication via third-party providers
- Cloud-based bookmark storage
- Sharing bookmarks between users (future enhancement)

## Goals

1. **User Authentication**: Support Google, GitHub, and Microsoft OAuth providers
2. **Data Migration**: Preserve existing localStorage bookmarks during first login
3. **Offline Support**: Allow read-only access when offline, sync on reconnection
4. **Zero Lock-in**: Export functionality to prevent vendor lock-in
5. **Privacy**: User data encrypted at rest, minimal data collection

## Non-Goals (v1)

- Self-hosted OAuth server (use established providers)
- Social features (sharing, public bookmarks)
- Mobile native apps
- Browser extension

## Architecture Changes

### Current Architecture
```
Browser (localStorage) ← → index.html (vanilla JS)
```

### Proposed Architecture
```
Browser                    Backend API              Database           OAuth Provider
  ↓                            ↓                        ↓                    ↓
index.html  ←→  API Client  ←→  Express/Node.js  ←→  PostgreSQL  ←→  Google/GitHub/MS
  ↓                            ↓                        ↓
localStorage              JWT tokens              User data
(cache)                   Session mgmt            Bookmarks
```

## OAuth Flow

### 1. Authorization Code Flow (Recommended)

```
User clicks "Login with Google"
  ↓
Frontend redirects to: /api/auth/google
  ↓
Backend redirects to: https://accounts.google.com/o/oauth2/v2/auth
  ↓
User grants permission
  ↓
Google redirects to: /api/auth/google/callback?code=xxx
  ↓
Backend exchanges code for access token
  ↓
Backend creates user session (JWT)
  ↓
Backend redirects to: /?token=jwt_token
  ↓
Frontend stores JWT, loads user bookmarks
```

### 2. Token Storage

**Access Token**: Stored server-side, never exposed to client
**JWT Token**: Stored in httpOnly cookie + localStorage (for SPA state)
**Refresh Token**: Stored server-side, rotated on use

### 3. Session Management

- **JWT Expiry**: 7 days
- **Refresh Mechanism**: Silent token refresh before expiry
- **Logout**: Invalidate JWT, clear localStorage cache, revoke OAuth token

## Backend Requirements

### Technology Stack

- **Runtime**: Node.js 18+
- **Framework**: Express.js
- **Database**: PostgreSQL 14+ (or MongoDB for document store)
- **Auth Library**: Passport.js with OAuth strategies
- **Security**: Helmet.js, rate limiting, CORS

### API Endpoints

#### Authentication
```
POST   /api/auth/login              # Initiate OAuth flow
GET    /api/auth/:provider          # OAuth provider redirect
GET    /api/auth/:provider/callback # OAuth callback handler
POST   /api/auth/logout             # Invalidate session
POST   /api/auth/refresh            # Refresh JWT token
GET    /api/auth/me                 # Get current user info
```

#### Bookmarks
```
GET    /api/bookmarks               # List user's bookmarks
POST   /api/bookmarks               # Create bookmark
GET    /api/bookmarks/:id           # Get single bookmark
PUT    /api/bookmarks/:id           # Update bookmark
DELETE /api/bookmarks/:id           # Delete bookmark
POST   /api/bookmarks/import        # Import from localStorage
GET    /api/bookmarks/export        # Export as JSON
```

#### Sync
```
GET    /api/sync/status             # Last sync timestamp
POST   /api/sync                    # Sync local changes to server
```

### Database Schema

#### users table
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  oauth_provider VARCHAR(50) NOT NULL,        -- 'google', 'github', 'microsoft'
  oauth_id VARCHAR(255) NOT NULL,             -- Provider's user ID
  email VARCHAR(255) NOT NULL,
  name VARCHAR(255),
  avatar_url TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  last_login TIMESTAMP,
  UNIQUE(oauth_provider, oauth_id)
);
```

#### bookmarks table
```sql
CREATE TABLE bookmarks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  url TEXT NOT NULL,
  title VARCHAR(500) NOT NULL,
  category VARCHAR(100) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP,                       -- Soft delete for sync
  INDEX idx_user_bookmarks (user_id, deleted_at),
  INDEX idx_user_category (user_id, category)
);
```

#### sessions table (optional - if not using JWT-only)
```sql
CREATE TABLE sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  jwt_hash VARCHAR(64) NOT NULL,              -- SHA-256 of JWT for revocation
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_session_expiry (expires_at)
);
```

## Frontend Changes

### 1. Authentication UI

Add to index.html (before form):
```html
<div class="auth-container">
  <div id="loginView" class="auth-view">
    <h2>Sign in to sync your bookmarks</h2>
    <button class="oauth-btn google-btn">Sign in with Google</button>
    <button class="oauth-btn github-btn">Sign in with GitHub</button>
    <button class="oauth-btn microsoft-btn">Sign in with Microsoft</button>
    <button class="continue-offline">Continue offline</button>
  </div>
  
  <div id="userView" class="auth-view" style="display: none;">
    <div class="user-info">
      <img src="" id="userAvatar" class="avatar">
      <span id="userName"></span>
    </div>
    <button id="syncBtn">Sync Now</button>
    <button id="logoutBtn">Logout</button>
  </div>
</div>
```

### 2. API Client

Create `api.js` module:
```javascript
class BookmarkAPI {
  constructor() {
    this.baseURL = '/api';
    this.token = localStorage.getItem('jwt');
  }

  async request(endpoint, options = {}) {
    const headers = {
      'Content-Type': 'application/json',
      ...(this.token && { Authorization: `Bearer ${this.token}` })
    };

    const response = await fetch(`${this.baseURL}${endpoint}`, {
      ...options,
      headers: { ...headers, ...options.headers }
    });

    if (response.status === 401) {
      await this.refreshToken();
      return this.request(endpoint, options);
    }

    if (!response.ok) throw new Error(`API error: ${response.statusText}`);
    return response.json();
  }

  async login(provider) {
    window.location.href = `${this.baseURL}/auth/${provider}`;
  }

  async getBookmarks() {
    return this.request('/bookmarks');
  }

  async createBookmark(bookmark) {
    return this.request('/bookmarks', {
      method: 'POST',
      body: JSON.stringify(bookmark)
    });
  }

  async sync(localBookmarks) {
    return this.request('/sync', {
      method: 'POST',
      body: JSON.stringify({ bookmarks: localBookmarks })
    });
  }

  async refreshToken() {
    const response = await fetch(`${this.baseURL}/auth/refresh`, {
      method: 'POST',
      credentials: 'include'
    });
    const { token } = await response.json();
    this.token = token;
    localStorage.setItem('jwt', token);
  }
}
```

### 3. Sync Strategy

**Conflict Resolution**: Last-write-wins (LWW) using `updated_at` timestamp

**Sync Flow**:
1. On login: Fetch server bookmarks, merge with local
2. On create/update/delete: Optimistic UI update + queue API call
3. On offline: Queue changes in localStorage
4. On reconnect: Replay queued changes, fetch latest

**localStorage Schema**:
```javascript
{
  bookmarks: [...],           // Current bookmarks (cache)
  pendingSync: [...],         // Offline changes queue
  lastSyncTime: 1234567890,   // Unix timestamp
  jwt: "eyJ..."               // Auth token
}
```

## Security Considerations

### 1. OAuth Security

- **State Parameter**: CSRF protection during OAuth flow (generate random nonce)
- **PKCE**: Use Proof Key for Code Exchange for mobile/SPA
- **Token Rotation**: Rotate refresh tokens on each use
- **Scope Limitation**: Request minimal OAuth scopes (profile, email only)

### 2. API Security

- **Rate Limiting**: 100 requests/minute per user
- **Input Validation**: Validate URL format, title length (max 500), category (max 100)
- **SQL Injection**: Use parameterized queries (pg-promise or Sequelize ORM)
- **XSS Prevention**: Continue using `escapeHtml()`, CSP headers
- **HTTPS Only**: Enforce HTTPS in production, HSTS headers

### 3. Data Protection

- **Encryption at Rest**: Database-level encryption (PostgreSQL TDE or AWS RDS encryption)
- **Encryption in Transit**: TLS 1.3
- **PII Handling**: Email and name only, no additional tracking
- **GDPR Compliance**: Data export, account deletion endpoints

### 4. Session Security

- **JWT Signing**: HS256 or RS256 with strong secret (256-bit minimum)
- **httpOnly Cookies**: Prevent XSS token theft
- **SameSite Cookies**: CSRF protection
- **Short Expiry**: 7-day max, refresh on activity

## Error Handling

### OAuth Errors
- **User Denies Access**: Show message, allow offline mode
- **Invalid State**: Potential CSRF, show error, clear state
- **Network Failure**: Retry with exponential backoff
- **Token Expired**: Silent refresh, fallback to re-login

### Sync Errors
- **Conflict**: Server version newer → fetch and merge
- **Network Offline**: Queue changes, show "offline" indicator
- **Server Error 500**: Retry up to 3 times, then show error

## Migration Path

### Phase 1: Add Authentication (Week 1)
- Set up backend (Express + PostgreSQL)
- Implement OAuth flows (Google only)
- Add login UI
- Deploy backend

### Phase 2: Cloud Sync (Week 2)
- Migrate localStorage bookmarks to server on first login
- Implement sync API
- Add conflict resolution
- Test offline/online transitions

### Phase 3: Multi-Provider (Week 3)
- Add GitHub OAuth
- Add Microsoft OAuth
- Add provider linking (same email → same account)

### Phase 4: Polish (Week 4)
- Export/import functionality
- Account deletion
- Improve error messages
- Performance optimization (pagination, caching)

## Configuration

### Environment Variables
```bash
# OAuth Credentials
GOOGLE_CLIENT_ID=xxx
GOOGLE_CLIENT_SECRET=xxx
GITHUB_CLIENT_ID=xxx
GITHUB_CLIENT_SECRET=xxx
MICROSOFT_CLIENT_ID=xxx
MICROSOFT_CLIENT_SECRET=xxx

# Session
JWT_SECRET=xxx                    # 256-bit random string
JWT_EXPIRY=7d

# Database
DATABASE_URL=postgresql://...

# Server
PORT=3000
NODE_ENV=production
FRONTEND_URL=https://bookmarks.example.com
ALLOWED_ORIGINS=https://bookmarks.example.com

# Rate Limiting
RATE_LIMIT_MAX=100
RATE_LIMIT_WINDOW=60000          # 1 minute in ms
```

### OAuth Provider Setup

**Google Cloud Console**:
1. Create project at console.cloud.google.com
2. Enable Google+ API
3. Create OAuth 2.0 credentials
4. Add authorized redirect: `https://yourdomain.com/api/auth/google/callback`
5. Scopes: `openid`, `profile`, `email`

**GitHub OAuth App**:
1. Settings → Developer Settings → OAuth Apps
2. Create new OAuth app
3. Callback URL: `https://yourdomain.com/api/auth/github/callback`
4. Scopes: `user:email`

**Microsoft Azure AD**:
1. Azure Portal → App Registrations
2. New registration
3. Redirect URI: `https://yourdomain.com/api/auth/microsoft/callback`
4. API Permissions: `User.Read`

## Testing Strategy

### Unit Tests
- JWT generation/validation
- URL validation
- Conflict resolution logic
- Sync queue management

### Integration Tests
- OAuth callback handling
- Bookmark CRUD operations
- Sync endpoint with conflicts
- Token refresh flow

### E2E Tests
- Complete OAuth login flow (mock provider)
- Create bookmark → logout → login → verify persistence
- Offline mode → create bookmark → go online → verify sync
- Multi-device simulation (two browser sessions)

## Monitoring & Logging

### Metrics
- OAuth login success/failure rate
- API response times (p50, p95, p99)
- Sync conflicts per user
- Active users (DAU, MAU)

### Logs
- OAuth errors (provider, error code)
- API errors (endpoint, status code, user_id)
- Sync conflicts (user_id, bookmark_id, resolution)
- Security events (failed auth, rate limit hits)

### Alerts
- OAuth provider downtime
- Database connection failures
- API error rate > 5%
- Disk space < 20%

## Cost Estimation

### Infrastructure (monthly)
- **Hosting**: $5-15 (Heroku Hobby, Railway, or DigitalOcean)
- **Database**: $7-15 (Managed PostgreSQL)
- **Domain**: $1/month
- **SSL**: $0 (Let's Encrypt)
- **Total**: ~$13-31/month for < 1000 users

### Scaling (10K users)
- **Hosting**: $25-50 (multiple instances)
- **Database**: $50-100 (larger instance, replicas)
- **CDN**: $10 (CloudFlare or BunnyCDN)
- **Monitoring**: $0-20 (New Relic free tier or paid)
- **Total**: ~$85-180/month

## Future Enhancements

### v2 Features
- Browser extension (Chrome, Firefox)
- Bookmark tags (in addition to categories)
- Full-text search
- Bookmark screenshots/previews
- Shared collections
- Teams/organizations

### v3 Features
- Mobile apps (React Native)
- Browser import (Chrome, Firefox bookmarks)
- Web scraping for metadata (title, description, favicon)
- AI-powered categorization
- Dead link detection

## References

- [OAuth 2.0 RFC](https://datatracker.ietf.org/doc/html/rfc6749)
- [JWT Best Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Passport.js Documentation](http://www.passportjs.org/)
