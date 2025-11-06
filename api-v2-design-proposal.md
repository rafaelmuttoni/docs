# Public API V2 - Design Proposal

## Design Principles & Decisions

### 1. Standardized Response Envelopes

**Decision**: All endpoints return the same structure.

**V2 Response Format:**

```json
{
  "data": { ... },
  "metadata": { ... }, // Optional, might be worth to consider adding extra debuggging info like request_id
  "pagination": {  // Only for paginated endpoints
    "cursor": "eyJ...",
    "has_more": true,
    "count": 20
  }
}

```

### 2. Consistent snake_case

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

- **V1**: `GET /people/paths/{url}` (URL-encoded in path)
- **V2**: `POST /people/paths` with body

**Why**:

- ✅ Flexible request body
- ✅ No URL encoding issues with LinkedIn URLs
- ✅ Cleaner API design (body for complex input)

**Example**:

```bash
POST /v2/people/paths
{
  "url": "<https://linkedin.com/in/johndoe>",
  "limit": 5
}

```

### 4. Flexible Identifiers (Single `url` Parameter)

<aside>

**🚨 STILL NOT 100% ON THIS ONE**

- need to consider if we want to support IDs etc, if we want a fully generic endpoint `identifier` might be better
</aside>

**Decision**: Accept LinkedIn URLs, domains in a single `url` field.

**Why**:

- ✅ No separate resolution endpoint needed
- ✅ Backend auto-detects identifier type
- ✅ Simpler developer experience

**Accepted formats**:

```json
// LinkedIn profile URL
{"url": "<https://linkedin.com/in/johndoe>"}

// Company domain
{"url": "acme.com"}

// LinkedIn company URL
{"url": "<https://linkedin.com/company/acme>"}

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

- ✅ Industry standard
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

## API Endpoints Overview

<aside>

ℹ️ Some endpoints are not mentioned here, such as lists, they should follow same structure proposed in **Design Principles & Decisions**

</aside>

### Authentication Endpoints

| Endpoint       | Method | Description                | V1 Equivalent |
| -------------- | ------ | -------------------------- | ------------- |
| `/auth/tokens` | POST   | Generate user access token | ❌ New        |

### User Endpoints

| Endpoint             | Method | Description                                    | V1 Equivalent |
| -------------------- | ------ | ---------------------------------------------- | ------------- |
| `/user`              | GET    | Get current user profile and status            | ❌ New        |
| `/user/integrations` | GET    | List connected integrations (Google, LinkedIn) | ❌ New        |

**User Endpoints Details:**

**GET /user** - Returns authenticated user's profile and status:

```json
{
  "data": {
    "id": "user_abc123",
    "email": "john@example.com",
    "name": "John Doe",
    "created_at": "2024-01-01T00:00:00Z",
    "is_sync_complete": true,
    "is_active": true // check app_user active field
  },
  "meta": { ... }
}

```

**GET /user/integrations** - Returns connected integrations:

```json
{
  "data": {
    "integrations": [
      {
        "type": "google",
        "is_sync_complete": true,
      },
      {
        "type": "linkedin",
        "is_sync_complete": true,
      }
    ]
  },
  "meta": { ... }
}

```

### People Endpoints

| Endpoint         | Method | Description                      | V1 Equivalent         |
| ---------------- | ------ | -------------------------------- | --------------------- |
| `/people/paths`  | POST   | Get connection paths to a person | `GET /people/paths`   |
| `/people/search` | POST   | Search for people                | `POST /people/search` |
| `/people/sort`   | POST   | Sort people by warmth            | `POST /people/sort`   |

### Companies Endpoints

| Endpoint            | Method | Description                     | V1 Equivalent                |
| ------------------- | ------ | ------------------------------- | ---------------------------- |
| `/companies/paths`  | POST   | Get connection paths to company | `GET /companies/paths/{url}` |
| `/companies/search` | POST   | Search for companies            | `POST /companies/search`     |
| `/companies/sort`   | POST   | Sort companies by warmth        | `POST /companies/sort`       |

### Key Differences from V1

1. **All paths/sort endpoints are POST** (not GET)
2. **New authentication endpoint as a starting point** (`/auth/tokens`)
3. **New user endpoints** (`/user/me`, `/user/integrations`, `/user/sync-status`, `/user/usage`)
4. **Identifiers/urls in request body** (not URL path)
5. **Consistent response format** across all endpoints
6. **No `success` boolean** - HTTP status codes indicate success/failure

## Core Schema Definitions

### Person Schema

The **same Person schema** is used everywhere in V2.

```json
{
  "id": "person_abc123",
  "first_name": "John",
  "last_name": "Doe",
  "full_name": "John Doe",
  "linkedin_url": "<https://linkedin.com/in/johndoe>",
  "headline": "CEO at Acme Corporation",
  "location": {
    "city": "San Francisco",
    "state": "CA",
    "country": "US"
  },
  "position": {
    "title": "Chief Executive Officer",
    "company": {
      "id": "company_xyz789",
      "name": "Acme Corporation",
      "linkedin_url": "<https://linkedin.com/company/acme>",
      "domain": "acme.com"
    },
    "start_date": "2020-01-15",
    "end_date": null
  },
  "image_url": "<https://media.licdn.com/dms/image/>...",
  "email": "john.doe@example.com"
}
```

**Field Definitions**:

| Field                 | Type           | Required | Description                  |
| --------------------- | -------------- | -------- | ---------------------------- |
| `id`                  | string         | ✅ Yes   | Village person ID            |
| `first_name`          | string         | ✅ Yes   | First name                   |
| `last_name`           | string         | ✅ Yes   | Last name                    |
| `full_name`           | string         | No       | Full name (computed)         |
| `linkedin_url`        | string (uri)   | No       | LinkedIn profile URL         |
| `headline`            | string         | No       | Professional headline        |
| `location`            | object         | No       | Geographic location          |
| `location.city`       | string         | No       | City                         |
| `location.state`      | string         | No       | State/province               |
| `location.country`    | string         | No       | Country code                 |
| `position`            | object         | No       | Current job position         |
| `position.title`      | string         | No       | Job title                    |
| `position.company`    | Company        | No       | Company object (see below)   |
| `position.start_date` | string (date)  | No       | Start date (YYYY-MM-DD)      |
| `image_url`           | string (uri)   | No       | Profile image URL            |
| `email`               | string (email) | No       | Email address (if available) |

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
  "linkedin_url": "<https://linkedin.com/company/acme>",
  "domain": "acme.com",
  "industry": "Technology",
  "employee_count": 5000,
  "image_url": "<https://logo.clearbit.com/acme.com>",
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
| `image_url`        | string (uri) | No       | Company logo URL                         |
| `location`         | object       | No       | Company HQ location                      |
| `location.city`    | string       | No       | City                                     |
| `location.state`   | string       | No       | State/province                           |
| `location.country` | string       | No       | Country code                             |

**Used in**:

- `CompanySearchResult.company`
- `CompanyWarmthScore.company`
- `Person.position.company`

### User Schema

The **User schema** represents the authenticated user.

```json
{
  "id": "user_abc123",
  "email": "john@example.com",
  "name": "John Doe",
  "created_at": "2024-01-01T00:00:00Z",
  "is_sync_complete": true,
  "is_active": true /
}

```

**Field Definitions**:

| Field              | Type              | Required | Description                                           |
| ------------------ | ----------------- | -------- | ----------------------------------------------------- |
| `id`               | string            | ✅ Yes   | Village user ID (format: `user_*`)                    |
| `email`            | string (email)    | ✅ Yes   | User email address                                    |
| `name`             | string            | ✅ Yes   | User full name                                        |
| `created_at`       | string (datetime) | ✅ Yes   | Account creation timestamp                            |
| `is_sync_complete` | boolean           | ✅ Yes   | Whether the user's initial sync is complete           |
| `is_active`        | boolean           | ✅ Yes   | User active status (reflects `app_user.active` value) |

**Used in**:

- `GET /user` response

### Integration Schema

Represents a connected integration (Google, LinkedIn).

```json
{
  "type": "google",
  "is_sync_complete": true
}
```

**Field Definitions**:

| Field              | Type    | Required | Description                            |
| ------------------ | ------- | -------- | -------------------------------------- |
| `type`             | string  | ✅ Yes   | Integration type: `google`, `linkedin` |
| `is_sync_complete` | boolean | ✅ Yes   | Whether integration is finished        |

**Used in**:

- `GET /user/integrations` response

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

**HTTP Status**: 2xx (200, 201, etc.)

```json
{
  "data": {
    // Endpoint-specific data
  },
  "meta": {
    "request_id": "req_abc123",
    "timestamp": "2024-01-06T12:00:00Z",
    "api_version": "2.0"
  },
  "pagination": {
    "cursor": "eyJvZmZzZXQiOjIwfQ==",
    "has_more": true,
    "count": 20
  }
}
```

### Error Response Envelope

**HTTP Status**: 4xx or 5xx

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

**Note**: Check HTTP status code for errors. Presence of `error` key (instead of `data` key) indicates error response.

**Error Types**:

- `validation_error` (400)
- `authentication_error` (401)
- `authorization_error` (403)
- `not_found_error` (404)
- `rate_limit_error` (429)
- `internal_error` (500)
- `service_unavailable_error` (503)

## 📢  Discussion points

### How do direct users authenticate on the API?

### Current V2 Design: Partner-Generated Tokens

V2 is designed for **partners** who generate tokens for their users:

```
┌──────────────┐
│   Partner    │
└──────┬───────┘
       │
       ├─ Has: Partner API Key (sk_*)
       │
       ├─ Step 1: POST /auth/tokens
       │  Header: secret-key: sk_example
       │  Body: {external_user_id, email}
       │
       ├─ Step 2: Receives token (jwt)
       │
       └─ Step 3: Uses token for API calls
          Header: Authorization: Bearer jwt

```

- Works well for Partners integrating Village into their apps where users don't have Village accounts.
- Users who have Village accounts and want to use the API directly - how should they authenticate?
  ### Option 1: User API Keys
  Allow users to generate API keys from their dashboard and use directly:
  ```bash
  # User uses API key in the same Authorization header

  # Use directly in all endpoints:
  curl <https://api.village.ai/v2/people/paths> \\
    -H "Authorization: Bearer ak_..." \\
    -d '{"url": "<https://linkedin.com/in/johndoe>"}'

  ```
  **Pros**:
  - ✅ Simple for users - no token generation needed
  - ✅ Matches how Stripe, GitHub, OpenAI work
  - ✅ Long-lived keys (user controls revocation)
  **Cons**:
  - ❌ Need to support two auth types (API keys + tokens)
  ### Option 2: Users Also Generate Tokens
  Users call `/auth/tokens` to generate tokens for themselves:
  ```bash
  # Step 1: User generates token for themselves
  curl <https://api.village.ai/v2/auth/tokens> \\
    -H "secret-key: ak_" \\

  # Step 2: Use the token
  curl <https://api.village.ai/v2/people/paths> \\
    -H "Authorization: Bearer village_usr_xyz789..." \\
    -d '{"url": "..."}'

  ```
  **Pros**:
  - ✅ Single auth flow for everyone
  - ✅ Shorter-lived tokens (better security)
  - ✅ Simpler implementation (one auth mechanism)
  **Cons**:
  - ❌ Extra step for users (can feel a bit redundant)
  - ❌ Confusing UX ("why do I need a token if I have an API key?")
  ### Option 3: Both work
  Support both User API Keys AND Tokens for users.
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
  - ✅ Partners use token
  - ✅ Users use API keys directly or token
  - ✅ Flexibility for different use cases
  **Cons**:
  - ❌ Two auth flows to document/maintain

### What IDs should we return on APIs?

<aside>

Some considerations

- Neo4j Person id is not reliable due to possible person getting merged, we would have to implement some sort of record that maps to merged IDs
- Neo4j Company id is essentially the linkedin slug of the company
- Postgres auto increment id is also not a great choice
</aside>

### Suggestion

- Create a new resource based ID on Postgres, similar to Stripe:
  e.g. `com_NffrFeUfNV2Hib` `per_NffrFeUfNV2Hib`
