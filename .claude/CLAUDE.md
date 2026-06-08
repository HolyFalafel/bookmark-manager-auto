# Bookmark Manager

A single-page HTML bookmark manager application with vanilla JavaScript and localStorage persistence.

## Project Overview

This is a simple, self-contained bookmark manager that runs entirely in the browser. Users can add, edit, delete, and search bookmarks organized by categories.

## Architecture

### Technology Stack
- **Frontend**: Vanilla HTML, CSS, JavaScript (no frameworks)
- **Storage**: Browser localStorage
- **Deployment**: Static HTML file, no backend required

### Core Features
1. **Add Bookmarks**: Form with URL, title, and category fields
2. **Edit Bookmarks**: Click edit button to populate form with existing data
3. **Delete Bookmarks**: Remove bookmarks with confirmation
4. **Search/Filter**: Real-time search across title, URL, and category
5. **Category Grouping**: Automatic grouping and alphabetical sorting by category
6. **Persistence**: All data stored in browser localStorage

### Data Model
```javascript
{
  id: Number,        // Timestamp-based ID (Date.now())
  url: String,       // Full URL with protocol
  title: String,     // Bookmark display name
  category: String   // Category for grouping
}
```

## File Structure

```
.
├── index.html              # Main application file (HTML, CSS, JS)
├── .claude/
│   ├── CLAUDE.md          # This file
│   └── skills/
│       ├── code-reviewer/
│       │   └── SKILL.md   # Code review skill definition
│       └── code-reviewer-manual/
└── .mcp.json              # MCP server configuration (project-scoped)
```

## Known Issues

### Critical (Fixed)
1. ~~**XSS Vulnerability**: Inline onclick handlers in edit/delete buttons~~ ✓ Fixed
   - Replaced inline onclick with event delegation using data attributes
   - Added `handleBookmarkAction()` method for secure event handling

2. ~~**localStorage Corruption**: No error handling for `JSON.parse()`~~ ✓ Fixed
   - Added try-catch wrapper in `loadBookmarks()`
   - App now returns empty array and logs error instead of crashing

### Important
3. **ID Collision**: Using `Date.now()` for IDs can create duplicates with rapid additions
   - Fix: Add randomness: `Date.now() + Math.random()`

4. **Stale Edit State**: Deleting a bookmark being edited leaves form in edit mode
   - Fix: Clear edit state in `deleteBookmark()` if editing that bookmark

5. **localStorage Quota**: No handling for QuotaExceededError
   - Fix: Wrap `setItem()` in try-catch, notify user

### Minor
6. **Whitespace Categories**: Empty or whitespace-only category names create invisible headers
7. **URL Sanitization**: href attributes should validate protocols to prevent `javascript:` URLs

## Development Guidelines

### Code Review Process
Use the custom code review skill located at `.claude/skills/code-reviewer/SKILL.md` when reviewing changes:
- Focus on correctness, security, and code quality
- Check for XSS, injection vulnerabilities, and error handling
- Verify tests cover edge cases
- Follow the project's existing patterns

### Security Considerations
- Always escape user input before rendering (use `escapeHtml()` method)
- Validate URLs have http/https protocols
- Never use inline event handlers (onclick, onerror, etc.)
- Handle localStorage errors gracefully

### Testing Approach
Since this is a simple client-side app:
1. Manual testing in browser for UI/UX verification
2. Test edge cases: empty fields, long URLs, special characters
3. Test localStorage: corruption, quota limits, missing data
4. Cross-browser testing for localStorage compatibility

### Commit Guidelines
- Use descriptive commit messages
- Include "Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>" in commits
- Don't commit sensitive files (.env, credentials)
- Stage specific files rather than using `git add -A`

## MCP Servers

Configured MCP servers for this project:
- **GitHub**: Connected (user scope, authenticated with GITHUB_PAT)

## Future Enhancements

Potential features to consider:
- Export/import bookmarks as JSON
- Browser extension integration
- Tag support (in addition to categories)
- Bookmark favicons
- Bulk operations (delete, move categories)
- Keyboard shortcuts
- Accessibility improvements (ARIA labels, keyboard navigation)
- Confirmation dialogs for destructive actions
