# Feature: OAuth Integration

## Requirements
- Support Google and GitHub OAuth
- Maintain session after page refresh
- Handle token refresh automatically

## Files to Modify
- src/auth/oauth.ts — OAuth client setup
- src/auth/session.ts — Session persistence
- server/routes/auth.ts — API endpoints

## Implementation Plan
1. Set up OAuth provider configuration
2. Add login flow UI components
3. Implement token refresh logic
4. Add integration tests

## Done When
- `npm test` passes
- Can log in with Google and GitHub
- Session survives page refresh