# URL Shortener (TinyURL) - Problem & Requirements

## What is a URL Shortener?

A URL Shortener is a service that takes a long URL and creates a shorter, unique alias that redirects to the original URL. When someone clicks the short URL, the service looks up the original URL and redirects the user there.

**Example:**

- Original URL: `https://www.example.com/products/electronics/smartphones/iphone-15-pro-max-256gb-titanium?color=blue&size=large`
- Short URL: `https://tiny.url/abc123`

### Why Does This Exist?

1. **Character limits**: Twitter (X) and SMS have character limits. Long URLs consume precious space.
2. **Readability**: Short URLs are easier to share verbally, in print, or on social media.
3. **Tracking**: Companies want to track how many people click their links, from where, and when.
4. **Branding**: Custom short domains (like `go.company.com/promo`) look professional.
5. **Link management**: Change the destination URL without changing the short link.

### What Breaks Without It?

- Marketing campaigns cannot track link performance
- Sharing long URLs becomes error-prone (copy-paste mistakes)
- QR codes become denser and harder to scan with longer URLs
- Social media posts look cluttered

---

## Clarifying Questions (Ask the Interviewer)

Before diving into design, a good engineer asks questions to understand scope:

| Question                                            | Why It Matters                       | Assumed Answer                                    |
| --------------------------------------------------- | ------------------------------------ | ------------------------------------------------- |
| What's the expected traffic?                        | Determines infrastructure scale      | 100M URLs created/month, 10B redirects/month      |
| Should short URLs expire?                           | Affects storage and cleanup strategy | Yes, default 5 years, customizable                |
| Do we need custom aliases?                          | Adds complexity to uniqueness checks | Yes, users can pick custom short codes            |
| Do we need analytics?                               | Adds click tracking infrastructure   | Yes, basic analytics (click count, referrer, geo) |
| Is this a multi-tenant SaaS?                        | Affects data isolation, billing      | No, single service for now                        |
| What's the acceptable latency for redirects?        | Critical for user experience         | < 100ms for redirects                             |
| Should we support private/password-protected links? | Adds authentication layer            | Out of scope for MVP                              |

---

## Functional Requirements

### Core Features (Must Have)

1. **URL Shortening**

   - Given a long URL, generate a unique short URL
   - Short code should be 6-8 characters (alphanumeric)
   - Support custom aliases (user-provided short codes)

2. **URL Redirection**

   - Given a short URL, redirect to the original URL
   - HTTP 301 (permanent) or 302 (temporary) redirect
   - Handle expired or deleted URLs gracefully

3. **URL Expiration**

   - Default expiration: 5 years
   - Custom expiration dates
   - Automatic cleanup of expired URLs

4. **Basic Analytics**
   - Total click count per URL
   - Click timestamps
   - Referrer information
   - Geographic location (country/city)

### Secondary Features (Nice to Have)

5. **Custom Short Codes**

   - Users can choose their own short code (e.g., `tiny.url/my-promo`)
   - Validation: no profanity, no reserved words

6. **User Accounts**

   - Registered users can manage their URLs
   - View analytics dashboard
   - Edit/delete their URLs

7. **API Access**
   - RESTful API for programmatic URL creation
   - API key authentication
   - Rate limiting per API key

---

## Non-Functional Requirements

### Performance

| Metric               | Target                        | Rationale                              |
| -------------------- | ----------------------------- | -------------------------------------- |
| Redirect latency     | < 50ms (p99 < 100ms)          | Users expect instant redirects         |
| URL creation latency | < 200ms                       | Acceptable for non-real-time operation |
| Availability         | 99.99% (52 min downtime/year) | Broken redirects damage trust          |

### Scale

| Metric               | Value       | Calculation               |
| -------------------- | ----------- | ------------------------- |
| New URLs/month       | 100 million | Given assumption          |
| Redirects/month      | 10 billion  | 100:1 read-to-write ratio |
| Total URLs (5 years) | 6 billion   | 100M × 12 × 5             |

### Reliability

- **No data loss**: Every created URL must be retrievable
- **Graceful degradation**: If analytics fails, redirects still work
- **Idempotent creation**: Same long URL can return same short URL (optional)

### Security

- Prevent malicious URL creation (phishing, malware)
- Rate limiting to prevent abuse
- No enumeration of short codes (prevent scraping)

---

## What's Out of Scope

To keep the design focused, we explicitly exclude:

1. **Password-protected URLs**: Requires authentication flow during redirect
2. **QR code generation**: Can be added as a separate service
3. **Link previews**: Fetching metadata from destination URLs
4. **A/B testing**: Splitting traffic between multiple destinations
5. **Team collaboration**: Shared workspaces, permissions
6. **Billing/monetization**: Paid tiers, usage limits

---

## System Constraints

### Technical Constraints

1. **Short code length**: 6-8 characters using base62 (a-z, A-Z, 0-9)

   - 62^6 = 56.8 billion combinations (more than enough for 6B URLs)
   - 62^7 = 3.5 trillion combinations

2. **URL validation**: Must be valid HTTP/HTTPS URL

   - Maximum original URL length: 2048 characters (browser limit)

3. **Storage**: URLs must persist for at least 5 years
   - No URL reuse after deletion (prevents security issues)

### Business Constraints

1. **Global service**: Must handle users worldwide
2. **Cost-effective**: Storage and compute costs must scale linearly
3. **Regulatory**: May need to comply with GDPR (right to deletion)

---

## Success Metrics

How do we know if the system is working well?

| Metric                    | Target   | How to Measure                        |
| ------------------------- | -------- | ------------------------------------- |
| Redirect success rate     | > 99.99% | Successful redirects / total requests |
| P50 redirect latency      | < 20ms   | Percentile from monitoring            |
| P99 redirect latency      | < 100ms  | Percentile from monitoring            |
| URL creation success rate | > 99.9%  | Successful creations / total attempts |
| System availability       | 99.99%   | Uptime monitoring                     |

---

## User Stories

### Story 1: Marketing Manager Creates Campaign Link

```
As a marketing manager,
I want to create a short URL for my product page,
So that I can share it on Twitter and track clicks.

Acceptance Criteria:
- I can paste a long URL and get a short one
- I can optionally choose a custom short code
- I can see how many people clicked my link
```

### Story 2: User Clicks Short Link

```
As an end user,
I want to click a short URL and reach the destination,
So that I can access the content I'm interested in.

Acceptance Criteria:
- Clicking the short URL takes me to the original page
- The redirect happens in under 100ms
- If the link is expired, I see a friendly error page
```

### Story 3: Developer Integrates via API

```
As a developer,
I want to create short URLs programmatically,
So that I can integrate URL shortening into my application.

Acceptance Criteria:
- I can call a REST API to create short URLs
- I receive the short URL in the response
- I'm rate-limited to prevent abuse
```

---

## Interview Tips

### What Interviewers Look For

1. **Requirement gathering**: Did you ask clarifying questions?
2. **Prioritization**: Can you identify MVP vs nice-to-have?
3. **Trade-off awareness**: Do you understand the implications of each requirement?

### Common Mistakes

1. **Jumping to solution**: Don't start with "we'll use Redis" before understanding requirements
2. **Ignoring scale**: 10 billion redirects/month is very different from 10 million
3. **Forgetting analytics**: Many candidates focus only on redirect, missing the tracking aspect

### Good Follow-up Questions from Interviewer

- "What if we need to support 100x more traffic?"
- "How would you handle a celebrity tweeting a short URL?" (thundering herd)
- "What if we need to change the short URL after creation?"

---

## Summary

| Aspect              | Decision                                   |
| ------------------- | ------------------------------------------ |
| Primary use case    | Shorten URLs, redirect users, track clicks |
| Scale               | 100M creates/month, 10B redirects/month    |
| Read:Write ratio    | 100:1 (read-heavy)                         |
| Latency target      | < 50ms redirect, < 200ms create            |
| Availability target | 99.99%                                     |
| Short code format   | 6-8 alphanumeric characters (base62)       |
| Expiration          | Default 5 years, customizable              |
