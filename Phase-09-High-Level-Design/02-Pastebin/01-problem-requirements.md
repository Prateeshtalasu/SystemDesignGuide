# Pastebin - Problem & Requirements

## What is Pastebin?

Pastebin is a web service that allows users to store and share plain text or code snippets. Users paste their content, receive a unique URL, and can share that URL with others. The service is commonly used for sharing code, logs, configuration files, and any text that's too long for chat or email.

**Example:**
- User pastes 500 lines of Python code
- Receives URL: `https://pastebin.com/abc123`
- Anyone with the URL can view the code with syntax highlighting

### Why Does This Exist?

1. **Character limits**: Chat apps and emails have size limits. Pastebin handles large text.
2. **Code sharing**: Syntax highlighting makes code readable. Better than plain text in emails.
3. **Temporary storage**: Share something quickly without creating accounts or files.
4. **Collaboration**: Multiple people can view the same paste.
5. **Privacy options**: Control who can see your content (public, unlisted, private).

### What Breaks Without It?

- Developers can't easily share code snippets
- Log files get truncated in chat messages
- Configuration sharing becomes error-prone
- No centralized way to share formatted text

---

## Clarifying Questions (Ask the Interviewer)

| Question | Why It Matters | Assumed Answer |
|----------|----------------|----------------|
| What's the maximum paste size? | Affects storage and bandwidth | 10 MB per paste |
| How long should pastes be retained? | Storage planning | Default 30 days, configurable up to 1 year |
| Do we need syntax highlighting? | Frontend complexity | Yes, support 100+ languages |
| Do we need user accounts? | Authentication complexity | Yes, optional accounts for management |
| What access levels are needed? | Authorization logic | Public, Unlisted, Private |
| Do we need paste editing? | Versioning complexity | No, pastes are immutable |
| What's the expected traffic? | Infrastructure scale | 10M pastes/month, 100M views/month |

---

## Functional Requirements

### Core Features (Must Have)

1. **Create Paste**
   - Upload text content (up to 10 MB)
   - Specify syntax highlighting language (optional)
   - Set expiration time (optional)
   - Choose visibility (public, unlisted, private)
   - Generate unique short URL

2. **View Paste**
   - Retrieve paste content by URL
   - Display with syntax highlighting
   - Show metadata (creation time, expiration, language)
   - Handle expired/deleted pastes gracefully

3. **Delete Paste**
   - Owner can delete their paste
   - Automatic deletion after expiration
   - Soft delete for audit trail

4. **Expiration Policies**
   - Never (permanent)
   - 10 minutes, 1 hour, 1 day, 1 week, 1 month, 1 year
   - Default: 30 days

### Secondary Features (Nice to Have)

5. **User Accounts**
   - Register/login
   - View paste history
   - Manage (delete) own pastes
   - API key for programmatic access

6. **Access Controls**
   - Public: Listed in recent pastes, searchable
   - Unlisted: Only accessible via URL (not listed)
   - Private: Requires authentication

7. **Syntax Highlighting**
   - Support 100+ programming languages
   - Auto-detect language (optional)
   - Line numbers
   - Copy to clipboard

8. **Rate Limiting**
   - Prevent spam and abuse
   - Different limits for anonymous vs authenticated users

---

## Non-Functional Requirements

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Paste creation latency | < 500ms | Acceptable for upload operation |
| Paste view latency | < 100ms | Users expect fast page loads |
| Availability | 99.9% (8.7 hours downtime/year) | Not mission-critical |

### Scale

| Metric | Value | Calculation |
|--------|-------|-------------|
| New pastes/month | 10 million | Given assumption |
| Paste views/month | 100 million | 10:1 read-to-write ratio |
| Average paste size | 10 KB | Most pastes are small code snippets |
| Total storage (1 year) | ~1.2 TB | 10M × 10KB × 12 months |

### Reliability

- **No data loss**: Pastes must be retrievable until expiration
- **Graceful degradation**: If syntax highlighting fails, show plain text
- **Idempotent operations**: Retry-safe API

### Security

- No execution of paste content (XSS prevention)
- Rate limiting to prevent abuse
- Content scanning for malware/spam (optional)

---

## What's Out of Scope

1. **Real-time collaboration**: No Google Docs-style editing
2. **Version history**: Pastes are immutable
3. **Comments/discussions**: No social features
4. **File uploads**: Text only, no binary files
5. **Encryption**: No end-to-end encryption (can be added later)
6. **Search**: No full-text search across pastes

---

## System Constraints

### Technical Constraints

1. **Paste size**: Maximum 10 MB per paste
   - Larger files should use file storage services

2. **Short URL format**: 8 alphanumeric characters
   - 62^8 = 218 trillion combinations

3. **Content type**: Plain text only
   - No images, no binary files

4. **Retention**: Maximum 1 year
   - Permanent pastes for premium users only

### Business Constraints

1. **Global service**: Users worldwide
2. **Cost-effective**: Storage is the main cost driver
3. **Compliance**: May need to remove illegal content

---

## Comparison: Pastebin vs URL Shortener

| Aspect | URL Shortener | Pastebin |
|--------|---------------|----------|
| Data stored | URL (< 2KB) | Text (up to 10MB) |
| Read:Write ratio | 100:1 | 10:1 |
| Storage concern | Minimal | Significant |
| Primary operation | Redirect (fast) | Content retrieval |
| Latency sensitivity | Very high | Moderate |
| Content type | URLs only | Any text |

Key difference: **Pastebin is storage-heavy**, URL Shortener is throughput-heavy.

---

## User Stories

### Story 1: Developer Shares Code Snippet
```
As a developer,
I want to paste my code and get a shareable link,
So that I can share it with my team on Slack.

Acceptance Criteria:
- I can paste up to 10MB of text
- I can select the programming language for highlighting
- I receive a short URL immediately
- My team can view the code with proper formatting
```

### Story 2: User Views Shared Paste
```
As a user,
I want to view a paste shared with me,
So that I can read the content.

Acceptance Criteria:
- I can open the paste URL in my browser
- I see the content with syntax highlighting
- I can copy the content to clipboard
- I see when the paste expires
```

### Story 3: User Manages Their Pastes
```
As a registered user,
I want to see all my pastes,
So that I can manage or delete them.

Acceptance Criteria:
- I can log in to my account
- I see a list of all my pastes
- I can delete pastes I no longer need
- I can see view counts for each paste
```

---

## Access Control Matrix

| Visibility | Anonymous Create | View without Auth | Listed in Public | Searchable |
|------------|------------------|-------------------|------------------|------------|
| Public | Yes | Yes | Yes | Yes |
| Unlisted | Yes | Yes | No | No |
| Private | No (requires account) | No | No | No |

---

## Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Paste creation success rate | > 99.9% | Successful creates / total attempts |
| P50 view latency | < 50ms | Percentile from monitoring |
| P99 view latency | < 200ms | Percentile from monitoring |
| Storage efficiency | < $0.01/paste/month | Total storage cost / active pastes |
| System availability | 99.9% | Uptime monitoring |

---

## Interview Tips

### What Interviewers Look For

1. **Understanding the difference from URL Shortener**: Storage is the key concern
2. **Handling large content**: Chunking, streaming, compression
3. **Access control design**: How to implement private pastes
4. **Expiration handling**: Efficient cleanup at scale

### Common Mistakes

1. **Treating it like URL Shortener**: Ignoring storage implications
2. **Forgetting about large pastes**: 10MB needs special handling
3. **Ignoring access controls**: Security is important for private pastes
4. **Not considering expiration**: How to efficiently delete expired content

---

## Summary

| Aspect | Decision |
|--------|----------|
| Primary use case | Store and share text/code snippets |
| Scale | 10M creates/month, 100M views/month |
| Read:Write ratio | 10:1 |
| Max paste size | 10 MB |
| Latency target | < 100ms view, < 500ms create |
| Availability target | 99.9% |
| Short URL format | 8 alphanumeric characters |
| Default expiration | 30 days |
| Access levels | Public, Unlisted, Private |

