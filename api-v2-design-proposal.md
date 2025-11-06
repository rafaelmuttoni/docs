## Design Principles & Decisions

### 1. Standardized Response Envelopes

**Decision**: All endpoints return the same structure.

**V2 Response Format:**

```json
{
  "data": { ... },
  "metadata": { ... },// Optional, might be worth to consider extra debuggging info like request_id
  "pagination": {  // Optional, only for paginated endpoints
    "cursor": "eyJ...",
    "has_more": true,
    "count": 20
  }
}
```

### 2. Consistent snake_case Naming

**Decision**: All JSON properties use `snake_case`.

**Why**:

- Most REST APIs use snake_case (Stripe, Twilio, GitHub)
- Clear convention across entire API

**Example**:

```json
{
  "first_name": "John",
  "last_name": "Doe",
  "linkedin_url": "https://...",
  "image_url": "https://...",
  "position": { ... }
}
```

### 3. POST Instead of GET for Paths

**V1**: `GET /people/paths/{url}` (URL-encoded LinkedIn URL in path)
**V2**: `POST /people/paths` with body

**Why**:

- ✅ Flexible request body
- ✅ No URL encoding issues with LinkedIn URLs
- ✅ Cleaner API design (body for complex input)

**Example**:

```bash
POST /v2/people/paths
{
  "url": "https://linkedin.com/in/johndoe",
  "limit": 5
}
```

### 4. Flexible Identifiers (Single `url` Parameter)

**🚨 STILL NOT 100% ON THIS ONE**: need to consider if we want to support IDs etc, if we want a fully generic endpoint `identifier` might be better

**Decision**: Accept LinkedIn URLs, domains in a single `url` field.

**Why**:

- ✅ No separate resolution endpoint needed
- ✅ Backend auto-detects identifier type
- ✅ Simpler developer experience

**Accepted formats**:

```json
// LinkedIn profile URL
{"url": "https://linkedin.com/in/johndoe"}

// Company domain
{"url": "acme.com"}

// LinkedIn company URL
{"url": "https://linkedin.com/company/acme"}
```

### 5. Cursor-Based Pagination

**Decision**: Use cursor-based pagination instead of offset/limit.

**Why**:

- ✅ Works with our search
- ✅ Industry standard (Stripe, GitHub, Slack)

**Example**:

```json
{
  "success": true,
  "data": [...],
  "pagination": {
    "cursor": "eyJvZmZzZXQiOjIwfQ==",
    "has_next_page": true,
    "count": 20
  }
}
```

### 6. RFC 7807 Error Format

**Decision**: Use RFC 7807 Problem Details standard for errors.

**Why**:

- ✅ Industry standard (IETF spec)
- ✅ Machine-readable error types
- ✅ Human-readable messages
- ✅ Consistent error structure

**Example**:

```json
{
  "error": {
    "type": "validation_error",
    "title": "Invalid request parameters",
    "status": 400,
    "detail": "Request validation failed",
    "trace_id": "trace_abc123",
    "errors": [
      {
        "field": "query",
        "message": "Query is required and must not be empty",
        "code": "REQUIRED_FIELD"
      }
    ]
  },
  "meta": {
    "request_id": "req_xyz789",
    "timestamp": "2024-01-06T12:00:00Z"
  }
}
```

---

## API Endpoints Overview

### V2 Endpoints

| Endpoint                 | Method | Description                      | V1 Equivalent                |
| ------------------------ | ------ | -------------------------------- | ---------------------------- |
| `/auth/token`            | POST   | Generate user access token       | ❌ New                       |
| `/people/paths`          | POST   | Get connection paths to a person | `GET /people/paths/{url}`    |
| `/people/search`         | POST   | Search for people                | `POST /people/search`        |
| `/people/sort`           | POST   | Sort people by warmth            | `POST /people/sort`          |
| `/companies/paths`       | POST   | Get connection paths to company  | `GET /companies/paths/{url}` |
| `/companies/search`      | POST   | Search for companies             | `POST /companies/search`     |
| `/companies/batch/paths` | POST   | Get paths to multiple companies  | ❌ New                       |
| `/companies/sort`        | POST   | Sort companies by warmth         | `POST /companies/sort`       |
| `/health`                | GET    | Health check                     | ❌ New                       |

### Key Differences from V1

1. **All paths/sort endpoints are POST** (not GET)
2. **New `/auth/tokens` endpoint** for partner-generated tokens
3. **New batch endpoints** for bulk operations
4. **Identifiers in request body** (not URL path)
5. **Consistent response format** across all endpoints

---

## Core Schema Definitions

### Person Schema

The **same Person schema** is used everywhere in V2.

```json
{
  "id": "person_abc123",
  "first_name": "John",
  "last_name": "Doe",
  "full_name": "John Doe",
  "linkedin_url": "https://linkedin.com/in/johndoe",
  "headline": "CEO at Acme Corporation",
  "location": {
    "city": "San Francisco",
    "state": "CA",
    "country": "US"
  },
  "current_position": {
    "title": "Chief Executive Officer",
    "company": {
      "id": "company_xyz789",
      "name": "Acme Corporation",
      "linkedin_url": "https://linkedin.com/company/acme",
      "domain": "acme.com"
    },
    "start_date": "2020-01-15"
  },
  "profile_image_url": "https://media.licdn.com/dms/image/...",
  "email": "john.doe@example.com"
}
```

**Field Definitions**:

| Field                         | Type           | Required | Description                            |
| ----------------------------- | -------------- | -------- | -------------------------------------- |
| `id`                          | string         | ✅ Yes   | Village person ID (format: `person_*`) |
| `first_name`                  | string         | ✅ Yes   | First name                             |
| `last_name`                   | string         | ✅ Yes   | Last name                              |
| `full_name`                   | string         | No       | Full name (computed)                   |
| `linkedin_url`                | string (uri)   | No       | LinkedIn profile URL                   |
| `headline`                    | string         | No       | Professional headline                  |
| `location`                    | object         | No       | Geographic location                    |
| `location.city`               | string         | No       | City                                   |
| `location.state`              | string         | No       | State/province                         |
| `location.country`            | string         | No       | Country code                           |
| `current_position`            | object         | No       | Current job position                   |
| `current_position.title`      | string         | No       | Job title                              |
| `current_position.company`    | Company        | No       | Company object (see below)             |
| `current_position.start_date` | string (date)  | No       | Start date (YYYY-MM-DD)                |
| `profile_image_url`           | string (uri)   | No       | Profile image URL                      |
| `email`                       | string (email) | No       | Email address (if available)           |

**Used in**:

- `ConnectionPath.source` and `ConnectionPath.target`
- `PersonSearchResult.person`
- `EmployeeWithPaths.person`
- `Introducer.person`
- `PersonWarmthScore.person`

### Company Schema

The **same Company schema** is used everywhere in V2.

```json
{
  "id": "company_xyz789",
  "name": "Acme Corporation",
  "linkedin_url": "https://linkedin.com/company/acme",
  "domain": "acme.com",
  "industry": "Technology",
  "employee_count": 5000,
  "logo_url": "https://logo.clearbit.com/acme.com",
  "location": {
    "city": "San Francisco",
    "state": "CA",
    "country": "US"
  }
}
```

**Field Definitions**:

| Field              | Type         | Required | Description                              |
| ------------------ | ------------ | -------- | ---------------------------------------- |
| `id`               | string       | ✅ Yes   | Village company ID (format: `company_*`) |
| `name`             | string       | ✅ Yes   | Company name                             |
| `linkedin_url`     | string (uri) | No       | LinkedIn company page URL                |
| `domain`           | string       | No       | Company website domain                   |
| `industry`         | string       | No       | Primary industry                         |
| `employee_count`   | integer      | No       | Number of employees                      |
| `logo_url`         | string (uri) | No       | Company logo URL                         |
| `location`         | object       | No       | Company HQ location                      |
| `location.city`    | string       | No       | City                                     |
| `location.state`   | string       | No       | State/province                           |
| `location.country` | string       | No       | Country code                             |

**Used in**:

- `CompanySearchResult.company`
- `CompanyWarmthScore.company`
- `Person.current_position.company`

### ConnectionPath Schema

Represents a path from user to target person.

```json
{
  "id": "path_xyz789",
  "source": {
    /* Person object */
  },
  "target": {
    /* Person object */
  },
  "degree": 2,
  "warmth_score": 8.5,
  "warmth_label": "Excellent",
  "introducers": [
    {
      "person": {
        /* Person object */
      },
      "relationship_strength": 9.2,
      "mutual_connections_count": 15,
      "context": "Strong professional relationship through TechCorp"
    }
  ]
}
```

**Field Definitions**:

| Field          | Type          | Description                                                               |
| -------------- | ------------- | ------------------------------------------------------------------------- |
| `id`           | string        | Unique path identifier (format: `path_*`)                                 |
| `source`       | Person        | The authenticated user                                                    |
| `target`       | Person        | The target person                                                         |
| `degree`       | integer (1-3) | Degrees of separation (1 = direct)                                        |
| `warmth_score` | number (0-10) | Relationship strength score                                               |
| `warmth_label` | enum          | Human-readable category: `Poor`, `Fair`, `Good`, `Very Good`, `Excellent` |
| `introducers`  | array         | People who can make the introduction                                      |

### Success Response Envelope

```json
{
  "success": true,
  "data": {
    /* Endpoint-specific data */
  },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2024-01-06T12:00:00Z",
    "api_version": "2.0"
  },
  "pagination": {
    // Optional
    "cursor": "eyJvZmZzZXQiOjIwfQ==",
    "has_more": true,
    "count": 20
  }
}
```

### Error Response Envelope

```json
{
  "success": false,
  "error": {
    "type": "validation_error",
    "title": "Invalid request parameters",
    "status": 400,
    "detail": "Request validation failed",
    "trace_id": "trace_abc123",
    "errors": [
      {
        "field": "query",
        "message": "Query is required and must not be empty",
        "code": "REQUIRED_FIELD"
      }
    ]
  },
  "meta": {
    "request_id": "req_xyz789",
    "timestamp": "2024-01-06T12:00:00Z"
  }
}
```

**Error Types**:

- `validation_error` (400)
- `authentication_error` (401)
- `authorization_error` (403)
- `not_found_error` (404)
- `rate_limit_error` (429)
- `internal_error` (500)
- `service_unavailable_error` (503)

---

## Authentication & Authorization (OPEN DISCUSSION)

### Current V2 Design: Partner-Generated Tokens

V2 is designed for **partners** who generate tokens for their users:

```
┌──────────────┐
│   Partner    │
└──────┬───────┘
       │
       ├─ Has: Partner API Key (vill_partner_*)
       │
       ├─ Step 1: POST /auth/tokens
       │  Header: secret-key: vill_partner_abc123
       │  Body: {external_user_id, scopes}
       │
       ├─ Step 2: Receives token (village_usr_*)
       │
       └─ Step 3: Uses token for API calls
          Header: Authorization: Bearer village_usr_xyz789
```

**Works well for**: Partners integrating Village into their apps where users don't have Village accounts.

---

### ❓ Question: How do DIRECT USERS use the API?

Users who have Village accounts and want to use the API directly - how should they authenticate?

---

### Option 1: User API Keys (Recommended ✅)

Allow users to generate API keys from their dashboard and use directly:

```bash
# User generates API key from Village dashboard
# Key format: vill_user_abc123def456

# Use directly in all endpoints:
curl https://api.village.ai/v2/people/paths \
  -H "Authorization: Bearer vill_user_abc123..." \
  -d '{"url": "https://linkedin.com/in/johndoe"}'
```

**Pros**:

- ✅ Simple for users - no token generation needed
- ✅ Users already have Village accounts
- ✅ Clear separation: Partners vs Users
- ✅ Matches how Stripe, GitHub, OpenAI work
- ✅ Long-lived keys (user controls revocation)

**Cons**:

- ❌ Need to support two auth types (API keys + tokens)
- ❌ Long-lived keys = higher security risk if leaked

**Implementation**:

```typescript
// Detect auth type by prefix
if (authHeader.startsWith("vill_user_")) {
  // Direct user API key - full network access
} else if (authHeader.startsWith("village_usr_")) {
  // Partner-generated token - scoped access
}
```

---

### Option 2: Users Also Generate Tokens

Users call `/auth/tokens` to generate tokens for themselves:

```bash
# Step 1: User generates token for themselves
curl https://api.village.ai/v2/auth/tokens \
  -H "secret-key: vill_user_key_abc123" \
  -d '{"external_user_id": "self", "scopes": ["read:network"]}'

# Step 2: Use the token
curl https://api.village.ai/v2/people/paths \
  -H "Authorization: Bearer village_usr_xyz789..." \
  -d '{"url": "..."}'
```

**Pros**:

- ✅ Single auth flow for everyone
- ✅ Shorter-lived tokens (better security)
- ✅ Simpler implementation (one auth mechanism)

**Cons**:

- ❌ Extra step for users (feels redundant)
- ❌ Users generating tokens for themselves is unusual
- ❌ Confusing UX ("why do I need a token if I have an API key?")

---

### Option 3: Hybrid Approach ⭐

Support both User API Keys AND Partner Tokens:

```typescript
// API key format: vill_user_xxx or vill_partner_xxx
// Token format: village_usr_xxx

if (authHeader.startsWith("vill_user_")) {
  // Direct user API key
  // Full access to that user's network
} else if (authHeader.startsWith("vill_partner_")) {
  // Partner API key (only for /auth/tokens endpoint)
} else if (authHeader.startsWith("village_usr_")) {
  // Partner-generated user token
  // Access based on token scopes
}
```

**Flow for Direct Users**:

```
┌──────────────┐
│  Direct User │
└──────┬───────┘
       │
       ├─ Generates: User API Key (vill_user_*) from dashboard
       │
       └─ Uses key directly for all API calls
          Header: Authorization: Bearer vill_user_abc123
```

**Flow for Partners**:

```
┌──────────────┐
│   Partner    │
└──────┬───────┘
       │
       ├─ Step 1: POST /auth/tokens with Partner API Key
       │
       ├─ Step 2: Receives user token (village_usr_*)
       │
       └─ Step 3: Uses token with scopes
```

**Pros**:

- ✅ Best of both worlds
- ✅ Partners use token flow (scoped access)
- ✅ Users use API keys directly (simple)
- ✅ Common pattern (Stripe does exactly this)
- ✅ Flexibility for different use cases

**Cons**:

- ❌ Slightly more complex to implement
- ❌ Two auth flows to document/maintain

---

### Comparison Table

| Aspect                        | Option 1: User API Keys | Option 2: Users Generate Tokens | Option 3: Hybrid  |
| ----------------------------- | ----------------------- | ------------------------------- | ----------------- |
| **User Experience**           | ⭐ Simple (one step)    | ⚠️ Complex (two steps)          | ⭐ Simple         |
| **Partner Experience**        | ⭐ Token flow           | ⭐ Token flow                   | ⭐ Token flow     |
| **Security**                  | ⚠️ Long-lived keys      | ⭐ Short-lived tokens           | ⭐ Both options   |
| **Implementation Complexity** | ⚠️ Two auth types       | ⭐ One auth type                | ⚠️ Two auth types |
| **Industry Standard**         | ⭐ Yes (Stripe, GitHub) | ❌ Unusual                      | ⭐ Yes (Stripe)   |
| **Flexibility**               | ⚠️ Users only           | ⚠️ Limited                      | ⭐ Maximum        |

---

### 🎯 Recommendation

**Go with Option 3 (Hybrid)** because:

1. **Industry standard**: Stripe, GitHub, and other major APIs support both
2. **Best UX**: Users get simple API keys, partners get scoped tokens
3. **Flexibility**: Different use cases have different needs
4. **Future-proof**: Can add features to either auth type independently

**Implementation plan**:

- User API keys: `vill_user_*` (full network access, revokable from dashboard)
- Partner API keys: `vill_partner_*` (only for `/auth/tokens` endpoint)
- User tokens: `village_usr_*` (scoped access, expirable)

---

### ❓ OPEN QUESTION FOR TEAM

**Which authentication approach should we implement?**

- [ ] Option 1: User API Keys only
- [ ] Option 2: Users also generate tokens
- [ ] Option 3: Hybrid (User API Keys + Partner Tokens) ← Recommended

**Please discuss and document decision here:**

---

## Open Questions for Discussion

### 1. Rate Limiting Strategy

**Current V2 design**:

- Per User: 100 requests/minute
- Per Partner: 10,000 requests/minute
- Search Endpoints: 20 requests/minute per user

**Questions**:

- ❓ Are these limits appropriate?
- ❓ Should we have different limits for different endpoints?
- ❓ Should we offer paid tiers with higher limits?
- ❓ How do we handle burst traffic?

**Headers included in all responses**:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1704542430
```

---

### 2. API Versioning Strategy

**Current V2 design**: Version in URL (`/v2/people/paths`)

**Questions**:

- ❓ Should we support header-based versioning too? (`X-API-Version: 2.0`)
- ❓ How long do we support V1 after V2 launches?
- ❓ Do we support multiple V2 minor versions simultaneously?

**Options**:

- **URL versioning** (current): `/v2/people/paths` → Simple, clear, cache-friendly
- **Header versioning**: `X-API-Version: 2.0` → More flexible, harder to test
- **Both**: Support both for flexibility

---

### 3. Backward Compatibility with V1

**Questions**:

- ❓ Do we maintain V1 indefinitely?
- ❓ What's the migration timeline for existing partners?
- ❓ Do we build automatic V1→V2 adapters?
- ❓ Do we deprecate V1 endpoints gradually or all at once?

**Proposed migration timeline**:

1. **Month 1-2**: V2 beta release, invite partners to test
2. **Month 3**: V2 general availability, V1 marked as deprecated
3. **Month 6**: V1 "sunset" warning to all partners
4. **Month 9**: V1 becomes read-only (no new features)
5. **Month 12**: V1 shut down (partners must migrate)

---

### 4. Response Size Limits

**Questions**:

- ❓ What's the max response size we should support?
- ❓ Should we compress large responses automatically?
- ❓ Should we paginate endpoints that currently don't paginate?

**Current V2 limits**:

- Paths per person: 10 (max)
- People in batch: 50 (max)
- Search results: 100 per page (max)
- Companies in batch: 20 (max)

---

### 5. Webhook Support

**Questions**:

- ❓ Should V2 support webhooks for async operations?
- ❓ What events should trigger webhooks? (e.g., "new connection added", "search completed")
- ❓ How do partners register webhook endpoints?

**Note**: Current V2 design does NOT include webhooks. This could be a V2.1 feature.

---

### 6. SDK Generation

**Questions**:

- ❓ Do we auto-generate SDKs from OpenAPI spec?
- ❓ Which languages? (TypeScript, Python, Go, Java?)
- ❓ Do we maintain SDKs ourselves or let community handle it?

**Recommended**:

- Use `openapi-typescript` for TypeScript SDK
- Use `openapi-generator` for other languages
- Publish official TypeScript SDK, community SDKs for others

---

### 7. Observability & Monitoring

**Questions**:

- ❓ What metrics do we track? (latency, error rate, usage per partner)
- ❓ Do we expose metrics to partners? (dashboard? API?)
- ❓ How do we alert partners about issues?
- ❓ Do we provide status page?

**Included in every response**:

```json
"meta": {
  "request_id": "req_abc123",  // ← For support/debugging
  "timestamp": "...",
  "api_version": "2.0"
}
```

---

## Breaking Changes from V1

### For Existing Partners

| V1                                   | V2                                  | Breaking?                       |
| ------------------------------------ | ----------------------------------- | ------------------------------- |
| `GET /people/paths/{url}`            | `POST /people/paths`                | ✅ Yes - Method changed         |
| `GET /companies/paths/{url}`         | `POST /companies/paths`             | ✅ Yes - Method changed         |
| Response: `{ target, count, paths }` | Response: `{ success, data, meta }` | ✅ Yes - Structure changed      |
| Person fields: `firstName`           | Person fields: `first_name`         | ✅ Yes - Naming changed         |
| No auth token generation             | `/auth/tokens` endpoint required    | ✅ Yes - Auth flow changed      |
| URL-encoded identifiers in path      | Identifiers in request body         | ✅ Yes - Request format changed |

### Migration Required

**All V1 partners must update**:

1. **Change HTTP methods**: GET → POST for paths endpoints
2. **Update request format**: Move URL from path to request body
3. **Update response parsing**: Expect `{ success, data, meta }` wrapper
4. **Update field names**: `firstName` → `first_name`, etc.
5. **Implement token generation**: Call `/auth/tokens` to get user tokens
6. **Update error handling**: Expect RFC 7807 error format

### Migration Guide (TODO)

We need to provide:

- [ ] Step-by-step migration guide
- [ ] Code examples (Before/After)
- [ ] Migration timeline
- [ ] Support channel for migration questions
- [ ] V1→V2 compatibility layer (optional)

---

## Next Steps

### 1. Decisions Needed

**High Priority**:

- [ ] **Authentication approach** (Option 1, 2, or 3?)
- [ ] **Rate limiting values** (Are current limits appropriate?)
- [ ] **V1 deprecation timeline** (When do we sunset V1?)

**Medium Priority**:

- [ ] API versioning strategy (URL only vs headers)
- [ ] SDK generation plan (Which languages?)
- [ ] Observability approach (Metrics dashboard for partners?)

**Low Priority**:

- [ ] Webhook support (V2.0 or V2.1?)
- [ ] Response compression strategy
- [ ] Status page requirements

---

### 2. Implementation Phases

**Phase 1: Core API (Weeks 1-4)**

- [ ] Implement standardized response middleware
- [ ] Implement flexible identifier resolution
- [ ] Implement error handling (RFC 7807)
- [ ] Implement rate limiting
- [ ] Create Person/Company schema validators

**Phase 2: Endpoints (Weeks 5-8)**

- [ ] `/auth/tokens` - Token generation
- [ ] `/people/paths` - Person paths
- [ ] `/people/search` - People search
- [ ] `/companies/paths` - Company paths
- [ ] `/companies/search` - Company search

**Phase 3: Advanced Features (Weeks 9-10)**

- [ ] `/people/batch/paths` - Batch operations
- [ ] `/companies/batch/paths` - Batch operations
- [ ] `/people/sort` - Sort by warmth
- [ ] `/companies/sort` - Sort by warmth

**Phase 4: Documentation & Testing (Weeks 11-12)**

- [ ] Complete OpenAPI spec
- [ ] Generate TypeScript SDK
- [ ] Write integration tests
- [ ] Create migration guide for V1 partners
- [ ] Set up performance monitoring

**Phase 5: Beta & Launch (Weeks 13-16)**

- [ ] Beta release to select partners
- [ ] Gather feedback and iterate
- [ ] General availability launch
- [ ] Begin V1 deprecation timeline

---

### 3. Documentation To-Do

- [ ] Complete migration guide (V1 → V2)
- [ ] Create authentication guide
- [ ] Write error handling guide
- [ ] Create code examples for all endpoints
- [ ] Set up Mintlify docs for V2
- [ ] Update Activepieces integration

---

### 4. Engineering Discussion

**Please review and provide feedback on**:

1. **Authentication approach** - Which option (1, 2, or 3)?
2. **Schema completeness** - Are Person/Company schemas complete?
3. **Missing endpoints** - Any critical endpoints missing?
4. **Rate limits** - Are proposed limits appropriate?
5. **Migration timeline** - Is 12-month deprecation timeline realistic?
6. **Open questions** - Any other concerns or questions?

**Add comments/decisions below**:

```
[Team discussion notes go here]
```

---

## References

-**OpenAPI Spec**: `/new-docs/api-reference/openapi.json` -**V1 OpenAPI Spec**: `/docs/api-reference/openapi.json` -**RFC 7807**: https://tools.ietf.org/html/rfc7807 -**Stripe API Design**: https://stripe.com/docs/api -**GitHub API Design**: https://docs.github.com/en/rest

---

**Last Updated**: 2025-01-06
**Status**: Draft - Awaiting Engineering Review
**Next Review**: [TBD]
