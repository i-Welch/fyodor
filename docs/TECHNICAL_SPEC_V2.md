# Fyodor - Entity Facet Platform - Technical Specification

**Version**: 2.0.0-draft
**Last Updated**: 2026-03-19

---

## Table of Contents

1. [Overview](#1-overview)
2. [System Architecture](#2-system-architecture)
3. [Accounts & Authentication](#3-accounts--authentication)
4. [Dashboard](#4-dashboard)
5. [Schema System](#5-schema-system)
6. [Facet Push](#6-facet-push)
7. [Entity Links](#7-entity-links)
8. [Query Interface](#8-query-interface)
9. [Subscriptions](#9-subscriptions)
10. [Conflict Resolution](#10-conflict-resolution)
11. [Facet SDK](#11-facet-sdk)
12. [Client SDK](#12-client-sdk)
13. [Observability](#13-observability)
14. [Error Handling](#14-error-handling)
15. [API Reference](#15-api-reference)

---

## 1. Overview

### 1.1 Purpose

Fyodor is an entity facet platform. Services push **facets** — their slice of a shared entity — as plain JSON, and Fyodor materializes an aggregated view. Consumers read the full entity or subscribe to changes without knowing which services provide which fields.

### 1.2 Core Principles

- **Entity-Centric**: All data is organized around entities with stable UUID identifiers
- **Push-First**: Services push facets when data changes; no runtime fan-out queries
- **Zero-Config Schema**: Schema is inferred automatically from pushes — no upfront field definitions
- **Conflict-Aware**: Breaking changes surface in a Dashboard for human resolution
- **Explicit Links**: Entity relationships are first-class, bidirectional, and traversable
- **Three Access Patterns**: Push, Query, Listen

### 1.3 Key Components

| Component | Responsibility |
|-----------|----------------|
| **Fyodor Service** | Facet ingestion, schema inference, view materialization, link storage, query serving, subscription dispatch, conflict detection |
| **Fyodor Dashboard** | Account management, entity type creation, conflict resolution, API key management, frontend auth configuration, observability |
| **Facet SDK** | Language-specific libraries for services to push facets and manage links (Kotlin, Python, Go, Ruby, Node.js, Rust) |
| **Client SDK** | Language-specific backend libraries to read entities and subscribe to changes |
| **Redis** | Materialized view storage, link index, schema state |
| **Kafka** | Internal change event distribution (for subscriptions) |

### 1.4 The Facet Model

An entity is composed of facets from multiple services:

```
Entity: User "u-123"
├── core-service pushed:      {name: "Alice", email: "alice@co.com"}
├── billing-service pushed:   {billing_id: "b-1", payment_status: "active"}
├── prefs-service pushed:     {locale: "en-US", dark_mode: true}
└── activity-service pushed:  {last_login: "2026-03-19T12:00:00Z"}

Links:
├── orders → [Order "o-1", Order "o-2"]
└── primary_address → Address "a-1"

Materialized view:
{
  uuid: "u-123",
  name: "Alice",
  email: "alice@co.com",
  billing_id: "b-1",
  payment_status: "active",
  locale: "en-US",
  dark_mode: true,
  last_login: "2026-03-19T12:00:00Z",
  _links: {
    orders: [{type: "Order", id: "o-1"}, {type: "Order", id: "o-2"}],
    primary_address: {type: "Address", id: "a-1"}
  }
}
```

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
                                ┌────────────────────────────────┐
                                │          CONSUMERS             │
                                │  ┌──────────┐     ┌────────┐  │
                                │  │ Backend  │     │ cURL / │  │
                                │  │ (SDK)    │     │ REST   │  │
                                │  └─────┬────┘     └───┬────┘  │
                                └────────┼──────────────┼───────┘
                                         │              │
                                    API Key          API Key
                                         │              │
                                         ▼              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           FYODOR SERVICE                                 │
│                                                                          │
│  ┌────────────┐  ┌──────────────┐  ┌───────────┐  ┌──────────────────┐  │
│  │ Schema     │  │ Facet        │  │ Query     │  │ Subscription     │  │
│  │ Inference  │  │ Materializer │  │ Server    │  │ Dispatch         │  │
│  └──────┬─────┘  └──────┬───────┘  └─────┬─────┘  └────────┬─────────┘  │
│         │               │               │                  │            │
│  ┌────────────┐  ┌──────────────┐  ┌─────────────┐         │            │
│  │ Conflict   │  │ Link         │  │ Auth        │         │            │
│  │ Detector   │  │ Manager      │  │ Verifier    │         │            │
│  └──────┬─────┘  └──────┬───────┘  └──────┬──────┘         │            │
│         │               │                │                  │            │
│  ┌──────┴───────────────┴────────────────┴──────────────────┴─────────┐  │
│  │                            REDIS                                   │  │
│  │  • Materialized entity views      • Inferred schemas              │  │
│  │  • Bidirectional link index       • Conflict queue                │  │
│  │  • Accounts & API keys           • Public signing keys            │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │                        KAFKA (internal)                            │  │
│  │  • Change event topics (for subscription fan-out)                 │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         ▲                   ▲                   ▲                  ▲
         │ Push (HTTP)       │ Push (HTTP)        │ Push (HTTP)      │ Push (HTTP)
         │ API Key           │ API Key            │ API Key          │ API Key
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Service A   │    │  Service B   │    │  Service C   │    │  Frontend    │
│  (Kotlin)    │    │  (Python)    │    │  (Go)        │    │  (Browser)   │
│  API Key     │    │  API Key     │    │  API Key     │    │  JWT Token   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘

                    ┌──────────────────────────┐
                    │     FYODOR DASHBOARD      │
                    │                          │
                    │  • Account management    │
                    │  • Entity type creation  │
                    │  • Schema explorer       │
                    │  • Conflict resolution   │
                    │  • API key management    │
                    │  • Signing key config    │
                    │  • Error rates / health  │
                    │  • Team management       │
                    └──────────────────────────┘
```

### 2.2 Data Flows

#### Push Flow

```
1. Service calls fyodor.push() via SDK (HTTP POST with API key)
2. Fyodor authenticates the API key, identifies the service and org
3. Fyodor validates the entity type exists (must be created in Dashboard first)
4. Fyodor infers/validates field types against known schema
5. If non-breaking (new fields or same types): accept, update schema
6. If breaking (type change or field ownership collision): queue conflict, hold field
7. Fyodor merges accepted fields into materialized view in Redis
8. Fyodor emits change event internally for subscription dispatch
```

#### Frontend Push Flow

```
1. Frontend sends facet push with JWT token (Authorization: Bearer <jwt>)
2. Fyodor looks up the org's configured public signing key(s)
3. Fyodor verifies the JWT signature against the public key
4. If invalid/expired: reject with 401
5. Fyodor checks the fyodor_scope claim against the target entity_type:entity_id
6. If scope doesn't match: reject with 403 SCOPE_DENIED
7. If valid and in scope: accept the push
```

#### Link Flow

```
1. Service calls fyodor.link() via SDK (HTTP POST with API key)
2. Fyodor validates both entity types exist
3. Fyodor stores forward link (source → target) and reverse link (target → source)
4. Fyodor emits change event (link added)
```

#### Query Flow

```
1. Consumer requests entity by type + ID (REST or gRPC)
2. Fyodor reads materialized view from Redis
3. Fyodor optionally resolves links (inlines linked entities)
4. Fyodor returns response as JSON
```

#### Listen Flow

```
1. Consumer subscribes to entity type (or specific entity ID)
2. On facet push or link change that modifies the materialized view:
   a. Fyodor computes a diff (changed fields / links)
   b. Fyodor dispatches change event to subscribers
3. Consumer receives change event with entity ID + changed fields
```

### 2.3 Deployment Model

- **Fyodor**: Horizontally scalable, stateless instances
- **Redis**: Clustered for HA, stores materialized views + link index
- **Kafka**: Internal only — used for change event fan-out to subscription dispatch
- **Dashboard**: Standalone web app, reads from Fyodor API
- **PostgreSQL**: Account data, org membership, audit logs (not on the hot path)

---

## 3. Accounts & Authentication

Fyodor has three authentication contexts: **Dashboard users** (humans managing the platform), **service API keys** (backend services pushing and reading data), and **frontend tokens** (browser clients pushing data only — no read access).

### 3.1 Organizations

All resources in Fyodor belong to an **organization**. An org is a single tenant — entity types, schemas, API keys, and signing keys are all scoped to an org.

```
Organization "Acme Corp"
├── Users: alice@acme.com (admin), bob@acme.com (member)
├── Entity Types: User, Order, Product, Address
├── Service API Keys: billing-service, core-service, orders-service
└── Signing Keys: RS256 public key for frontend push auth
```

### 3.2 User Accounts

Dashboard users sign up with email + password (or SSO). Each user belongs to one or more orgs.

**Roles:**

| Role | Permissions |
|------|------------|
| **Owner** | Full access. Can delete org, manage billing, transfer ownership. |
| **Admin** | Manage entity types, API keys, signing keys, users. Resolve conflicts. |
| **Member** | View schemas, entities, error rates. Cannot modify configuration. |

### 3.3 Service API Keys

API keys authenticate backend services pushing facets and managing links. Created in the Dashboard.

**Key format:** `fyd_sk_{service_name}_{random}`

Example: `fyd_sk_billing_a1b2c3d4e5f6g7h8`

**Properties:**
- Scoped to an org
- Mapped to a service name (used for field ownership tracking)
- Can be restricted to specific entity types (optional)
- Can be revoked instantly
- Multiple keys per service are allowed (for rotation)

**Usage:**
```
Authorization: Bearer fyd_sk_billing_a1b2c3d4e5f6g7h8
```

### 3.4 Frontend Authentication (JWT + Public Signing Keys)

Frontend clients (browsers, mobile apps) need to push facets (e.g., user activity, events, preferences) but cannot use API keys — they'd be exposed in client-side code. Instead, Fyodor supports **JWT token verification** using public signing keys configured by the org.

Frontend tokens grant **push-only access**. They cannot query entities, subscribe to changes, or read any data. All read operations require a service API key.

#### How it works:

1. The customer's **own auth system** issues JWTs to their end users (e.g., after login)
2. An admin uploads the **public signing key** to Fyodor via the Dashboard
3. The frontend SDK sends the JWT with each push: `Authorization: Bearer <jwt>`
4. Fyodor verifies the JWT signature against the stored public key
5. If valid and not expired → push is accepted
6. If invalid → 401 Unauthorized

Fyodor **does not manage end-user identity**. It only verifies that the JWT was signed by a trusted key. The customer's auth system handles login, registration, session management, etc.

#### Signing Key Configuration

Configured in the Dashboard under **Settings → Frontend Auth**.

**Supported algorithms:**
- RS256, RS384, RS512 (RSA)
- ES256, ES384, ES512 (ECDSA)
- EdDSA (Ed25519)

**Key configuration:**
```json
{
  "key_id": "key-1",
  "algorithm": "RS256",
  "public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqh...\n-----END PUBLIC KEY-----",
  "status": "active",
  "created_at": "2026-03-19T10:00:00Z"
}
```

Multiple keys can be active simultaneously (for key rotation). Keys can be deactivated without deletion.

#### JWKS Endpoint (Alternative)

Instead of uploading individual keys, orgs can configure a JWKS (JSON Web Key Set) URL. Fyodor will periodically fetch the keys from this endpoint:

```json
{
  "jwks_url": "https://auth.acme.com/.well-known/jwks.json",
  "refresh_interval_seconds": 3600
}
```

#### JWT Claims

| Claim | Required | Description |
|-------|----------|-------------|
| `exp` | Yes | Expiration time. Fyodor rejects expired tokens. |
| `fyodor_scope` | Yes | Array of `entity_type:entity_id` patterns the token is authorized to update (see below). |
| `sub` | No | Subject (end-user ID). Recorded as the source in field ownership and audit logs. |
| `iat` | No | Issued at. Used for staleness checks if configured. |
| `aud` | No | Audience. If the org configures an expected audience, Fyodor validates it. |

#### Scope Claim (`fyodor_scope`)

The `fyodor_scope` claim is an array of strings, each specifying an `entity_type:entity_id` pair the token is allowed to push to. Wildcards (`*`) are supported for the entity ID.

**Example JWT payload:**
```json
{
  "sub": "user-alice-123",
  "exp": 1742500000,
  "fyodor_scope": [
    "User:u-123",
    "UserActivity:*",
    "UserPrefs:u-123"
  ]
}
```

This token can:
- Push to `User` entity `u-123` only
- Push to any `UserActivity` entity (wildcard)
- Push to `UserPrefs` entity `u-123` only
- **Nothing else** — pushes to `Order`, or to `User:u-456`, are rejected

**Scope patterns:**

| Pattern | Meaning |
|---------|---------|
| `User:u-123` | Exactly entity `u-123` of type `User` |
| `UserActivity:*` | Any entity of type `UserActivity` |
| `*:*` | Any entity of any type (full access — use with caution) |

**Scope enforcement pseudocode:**

```
function checkFrontendScope(token, entityType, entityId):
    for pattern in token.fyodor_scope:
        patternType, patternId = pattern.split(":")

        typeMatch = (patternType == "*") or (patternType == entityType)
        idMatch = (patternId == "*") or (patternId == entityId)

        if typeMatch and idMatch:
            return ALLOWED

    return DENIED  // "SCOPE_DENIED"
```

If the push is denied, Fyodor returns:
```json
{
  "success": false,
  "error": {
    "code": "SCOPE_DENIED",
    "message": "Token scope does not permit push to User:u-456. Allowed: [User:u-123, UserActivity:*]"
  }
}
```

#### Frontend Permissions

Frontend tokens are **push-only** and **scope-restricted**. They cannot read data from Fyodor.

| Permission | Allowed | Scope-Checked |
|------------|---------|---------------|
| Push facets | Yes | Yes — must match `fyodor_scope` |
| Create/remove links | Yes | Yes — source entity must match `fyodor_scope` |
| Query entities | **No** | N/A |
| Subscribe to changes | **No** | N/A |
| View schema | **No** | N/A |

The JWT `sub` claim is recorded as the source, and a synthetic service name like `frontend:{sub}` is used for field ownership tracking.

### 3.5 Auth Storage (Redis + PostgreSQL)

```
# PostgreSQL (durable, account data)
orgs                        id, name, slug, created_at
users                       id, email, password_hash, created_at
org_memberships             org_id, user_id, role
api_keys                    id, org_id, service_name, key_hash, entity_types[], status, created_at
signing_keys                id, org_id, key_id, algorithm, public_key_pem, jwks_url, status, created_at
audit_log                   id, org_id, actor, action, resource, details, timestamp

# Redis (hot path, cached for request auth)
auth:apikey:{key_hash}              -> JSON {org_id, service_name, entity_types, status}
auth:signingkeys:{org_id}           -> JSON [{key_id, algorithm, public_key_pem, status}]
auth:jwks:{org_id}:cached           -> JSON (cached JWKS response)
auth:jwks:{org_id}:expires          -> timestamp (next refresh)
```

---

## 4. Dashboard

The Fyodor Dashboard is a web application for managing the platform. All configuration that affects runtime behavior is done here.

### 4.1 Account Management

- **Sign up / Sign in**: Email + password, or SSO (Google, GitHub, SAML)
- **Organization creation**: Create a new org, set name and slug
- **User invitation**: Invite users by email, assign role (admin/member)
- **Role management**: Promote/demote users, remove from org
- **Org settings**: Name, billing, data retention policies

### 4.2 Entity Type Management

Entity types **must be created in the Dashboard** before any service can push to them. This prevents accidental entity creation from typos and gives admins control over the data model.

**Create Entity Type:**

| Field | Description |
|-------|-------------|
| Name | e.g., `User`, `Order`, `Product` |
| Description | Optional, for documentation |
| ID format | Optional regex for validating entity IDs (e.g., `^u-[a-f0-9]+$`) |

Once created, services can push facets to that entity type immediately — no field definitions required. Fields are inferred on first push (see Section 5).

**Entity Type List View:**

```
┌─────────────┬──────────┬────────────┬──────────┬────────────────┐
│ Entity Type │ Fields   │ Services   │ Entities │ Pending        │
│             │          │            │          │ Conflicts      │
├─────────────┼──────────┼────────────┼──────────┼────────────────┤
│ User        │ 12       │ 4          │ 50,234   │ 1              │
│ Order       │ 8        │ 3          │ 120,891  │ 0              │
│ Product     │ 15       │ 2          │ 5,432    │ 0              │
│ Address     │ 6        │ 1          │ 48,102   │ 0              │
└─────────────┴──────────┴────────────┴──────────┴────────────────┘
```

### 4.3 Schema Explorer

Visual exploration of the inferred schema per entity type:

- Field list with types, owners, and timestamps
- Link definitions with target types
- Schema change history (field added, type changed, etc.)
- Per-field push statistics

### 4.4 Conflict Resolution

See [Section 10: Conflict Resolution](#10-conflict-resolution) for the data model.

**Conflict Queue View:**

```
┌────────────┬──────────┬───────────────────┬─────────────┬──────────────┐
│ Entity Type│ Field    │ Conflict          │ Services    │ Created      │
├────────────┼──────────┼───────────────────┼─────────────┼──────────────┤
│ User       │ status   │ OWNERSHIP_COLLISION│ core vs     │ 5 min ago    │
│            │          │                   │ billing     │              │
└────────────┴──────────┴───────────────────┴─────────────┴──────────────┘
```

**Resolution UI:** For each conflict, present:
- Current field state (type, owner, sample values)
- Incoming data (type, service, sample values)
- Resolution strategy picker (see Section 10.3)
- Preview of what the resolution will do
- Apply button

### 4.5 API Key Management

- **Create key**: Select service name, optional entity type restrictions
- **List keys**: See all keys, their service names, last used timestamps
- **Revoke key**: Instantly revoke — takes effect within seconds (Redis cache TTL)
- **Rotate key**: Create a new key for the same service, then revoke the old one

### 4.6 Frontend Push Auth Configuration

- **Upload public key**: Paste PEM-encoded public key, select algorithm
- **Configure JWKS URL**: Enter URL, set refresh interval
- **Key status**: Activate/deactivate keys without deleting
- **Test token**: Paste a JWT to verify it against configured keys
- **Entity type permissions**: Toggle which entity types accept frontend pushes and link operations
- **Expected audience**: Optionally require a specific `aud` claim value

### 4.7 Observability Dashboard

- **Push rate**: Pushes per second by entity type and service
- **Error rate**: Push errors, query errors, auth failures
- **Latency**: p50/p95/p99 for pushes and queries
- **Conflict rate**: New conflicts per hour
- **Entity counts**: Total materialized entities per type
- **Service health**: Last push time per service, active/stale detection
- **Active subscriptions**: Count by entity type and protocol

### 4.8 Link Graph Viewer

Visual graph of entity relationships:
- Entity types as nodes, link field names as edges
- Click-through to see specific entity instances and their links
- Reverse link exploration

---

## 5. Schema System

### 5.1 Schema Inference

There is no upfront field definition. Entity types are created in the Dashboard (Section 4.2), but their fields are inferred from push data.

When a service pushes a facet for the first time, Fyodor:

1. Validates the entity type exists (created in Dashboard)
2. Inspects each field's JSON value
3. Infers the type
4. Records the field name, type, and owning service
5. This becomes the authoritative schema for that field

**Type inference rules:**

| JSON Value | Inferred Type |
|------------|---------------|
| `"hello"` | `string` |
| `42` | `number` |
| `42.5` | `number` |
| `true` / `false` | `boolean` |
| `null` | `nullable` (type determined on next non-null push) |
| `[1, 2, 3]` | `array<number>` |
| `["a", "b"]` | `array<string>` |
| `[mixed types]` | `array<any>` |
| `{"k": "v"}` | `object` (nested fields inferred recursively) |

### 5.2 Schema Evolution

On each subsequent push, Fyodor compares the incoming data against the known schema:

#### Non-Breaking Changes (Auto-Accepted)

| Change | Behavior |
|--------|----------|
| New field from the same service | Field added to schema, service becomes owner |
| `null` pushed for a known field | Accepted (field remains in schema, value set to null) |
| Field omitted from push | No change (existing value retained, partial updates are the norm) |

#### Breaking Changes (Queued for Resolution)

| Change | Conflict Type |
|--------|---------------|
| Field type changed (e.g., `string` → `number`) | `TYPE_MISMATCH` |
| Field pushed by a different service than the current owner | `OWNERSHIP_COLLISION` |

Breaking changes are **not applied**. The incoming value is held in a conflict queue, and the field retains its current value until a human resolves the conflict via the Dashboard.

### 5.3 Field Ownership

Every field has exactly one owning service. Ownership is established on first push:

```
User entity schema (inferred):
┌─────────────────┬──────────┬────────────────────┐
│ Field           │ Type     │ Owner              │
├─────────────────┼──────────┼────────────────────┤
│ name            │ string   │ core-service       │
│ email           │ string   │ core-service       │
│ billing_id      │ string   │ billing-service    │
│ payment_status  │ string   │ billing-service    │
│ locale          │ string   │ prefs-service      │
│ dark_mode       │ boolean  │ prefs-service      │
│ last_login      │ string   │ activity-service   │
└─────────────────┴──────────┴────────────────────┘
```

If billing-service pushes `{"name": "Bob"}`, this triggers an `OWNERSHIP_COLLISION` because `name` is owned by core-service.

### 5.4 Schema Storage (Redis)

```
# Per entity type — inferred schema
schema:{org_id}:{entity_type}:fields          hash     field_name -> JSON {type, owner, created_at, updated_at}
schema:{org_id}:{entity_type}:version         int64    Incremented on schema change
schema:{org_id}:{entity_type}:history         list     JSON diffs of schema changes

# Conflict queue
conflicts:{org_id}:{entity_type}:{conflict_id}   hash     conflict details
conflicts:{org_id}:{entity_type}:pending          set      conflict IDs awaiting resolution
```

---

## 6. Facet Push

### 6.1 Push Protocol

Services push facets to Fyodor via HTTP. All Facet SDKs wrap this endpoint.

#### Single Push

```
POST /facets/{entity_type}/{entity_id}
Authorization: Bearer fyd_sk_billing_a1b2c3d4
Content-Type: application/json

{
  "billing_id": "b-1",
  "payment_method": "card",
  "payment_status": "active"
}
```

**Response (success):**
```json
{
  "success": true,
  "entity_version": 43,
  "accepted_fields": ["billing_id", "payment_method", "payment_status"],
  "conflicts": []
}
```

**Response (partial — some fields conflicted):**
```json
{
  "success": true,
  "entity_version": 43,
  "accepted_fields": ["billing_id", "payment_method"],
  "conflicts": [
    {
      "conflict_id": "c-abc123",
      "field": "payment_status",
      "type": "TYPE_MISMATCH",
      "message": "Field 'payment_status' is string, got number",
      "resolution_url": "https://dashboard.fyodor.dev/conflicts/c-abc123"
    }
  ]
}
```

Non-conflicting fields are always accepted immediately, even if other fields in the same push conflict.

#### Batch Push

```
POST /facets/_batch
Authorization: Bearer fyd_sk_billing_a1b2c3d4
Content-Type: application/json

{
  "entity_type": "User",
  "facets": [
    { "entity_id": "u-1", "data": { "billing_id": "b-1", "payment_status": "active" } },
    { "entity_id": "u-2", "data": { "billing_id": "b-2", "payment_status": "suspended" } }
  ]
}
```

**Response:**
```json
{
  "succeeded": 2,
  "failed": 0,
  "results": [
    { "entity_id": "u-1", "entity_version": 43, "accepted_fields": ["billing_id", "payment_status"], "conflicts": [] },
    { "entity_id": "u-2", "entity_version": 18, "accepted_fields": ["billing_id", "payment_status"], "conflicts": [] }
  ]
}
```

#### Entity Type Validation

Pushes to an entity type that hasn't been created in the Dashboard are rejected:

```json
{
  "success": false,
  "error": {
    "code": "UNKNOWN_ENTITY_TYPE",
    "message": "Entity type 'Userr' does not exist. Create it in the Dashboard first."
  }
}
```

### 6.2 Materialization

When a facet is pushed:

```
function materializeFacet(serviceName, entityType, entityId, data):
    schema = getSchema(entityType)
    accepted = {}
    conflicts = []

    for field, value in data:
        inferred_type = inferType(value)
        existing = schema.getField(field)

        if existing is null:
            // New field — auto-accept, assign ownership
            schema.addField(field, inferred_type, serviceName)
            accepted[field] = value

        else if existing.owner == serviceName:
            if existing.type == inferred_type or value is null:
                // Same owner, same type — accept
                accepted[field] = value
            else:
                // Same owner, type changed — check for resolution rule
                rule = getResolutionRule(entityType, field)
                if rule:
                    applyRule(rule, field, value)
                else:
                    conflicts.add(createConflict("TYPE_MISMATCH", field, existing, inferred_type, value))

        else:
            // Different owner — check for resolution rule
            rule = getResolutionRule(entityType, field)
            if rule:
                applyRule(rule, field, value)
            else:
                conflicts.add(createConflict("OWNERSHIP_COLLISION", field, existing, serviceName, value))

    // Merge accepted fields into materialized view
    if accepted is not empty:
        entityKey = "entity:{entityType}:{entityId}"
        current = redis.get(entityKey)
        merged = merge(current, accepted)
        redis.set(entityKey, merged)
        redis.hset("entity:{entityType}:{entityId}:facets", serviceName, now())
        emitChangeEvent(entityType, entityId, acceptedFields)

    return {accepted, conflicts}
```

### 6.3 Partial Updates

Pushes are **partial by default**. If billing-service pushes `{"payment_status": "suspended"}`, only that field is updated — all other fields in the materialized view are untouched.

To explicitly set a field to null:
```json
{"payment_method": null}
```

To remove a field entirely from the entity:
```json
{"_delete": ["payment_method"]}
```

### 6.4 Facet Metadata (Redis)

```
# Per-entity materialized view
entity:{org_id}:{entity_type}:{entity_id}              -> JSON (merged facet data)

# Per-entity facet tracking
entity:{org_id}:{entity_type}:{entity_id}:facets       -> hash (service_name -> last_push_timestamp)

# Per-entity version (incremented on each materialization)
entity:{org_id}:{entity_type}:{entity_id}:version      -> int64
```

---

## 7. Entity Links

Links are first-class relationships between entities. They are separate from facet data — managed through a dedicated API with bidirectional storage for traversal.

### 7.1 Link API

#### Create Link

```
POST /links
Authorization: Bearer fyd_sk_orders_a1b2c3d4
Content-Type: application/json

{
  "source_type": "User",
  "source_id": "u-123",
  "field_name": "orders",
  "target_type": "Order",
  "target_id": "o-456"
}
```

**Response:**
```json
{
  "success": true,
  "link_id": "lnk_a1b2c3d4",
  "source": { "type": "User", "id": "u-123" },
  "target": { "type": "Order", "id": "o-456" },
  "field_name": "orders"
}
```

#### Remove Link

```
POST /unlink
Authorization: Bearer fyd_sk_orders_a1b2c3d4
Content-Type: application/json

{
  "source_type": "User",
  "source_id": "u-123",
  "field_name": "orders",
  "target_type": "Order",
  "target_id": "o-456"
}
```

**Response:**
```json
{
  "success": true,
  "removed": true
}
```

#### Batch Link

```
POST /links/_batch
Authorization: Bearer fyd_sk_orders_a1b2c3d4
Content-Type: application/json

{
  "links": [
    { "source_type": "User", "source_id": "u-123", "field_name": "orders", "target_type": "Order", "target_id": "o-1" },
    { "source_type": "User", "source_id": "u-123", "field_name": "orders", "target_type": "Order", "target_id": "o-2" },
    { "source_type": "User", "source_id": "u-123", "field_name": "primary_address", "target_type": "Address", "target_id": "a-1" }
  ]
}
```

### 7.2 Link Behavior

- **Multiple targets per field**: The same `field_name` can link to many entities (e.g., `orders` → many Orders). Returned as an array.
- **Single target per field**: If only one link exists for a field name, returned as a single object. Becomes an array when a second link is added.
- **Link ownership**: The service that creates a link owns it. Only the owning service (or an admin) can remove it.
- **Dangling links**: If a linked entity doesn't exist yet, the link is still stored. Queries return the link with `_exists: false`.

### 7.3 Link Storage (Redis)

Bidirectional storage for efficient traversal in both directions:

```
# Forward links: source entity → targets
links:{org_id}:{source_type}:{source_id}:forward       -> hash (field_name -> JSON array of {target_type, target_id, created_by, created_at})

# Reverse links: target entity → sources that link to it
links:{org_id}:{target_type}:{target_id}:reverse        -> set of JSON {source_type, source_id, field_name, created_by, created_at}
```

**Forward lookup** (User → their orders):
```
HGET links:org1:User:u-123:forward "orders"
→ [{"target_type":"Order","target_id":"o-1",...}, {"target_type":"Order","target_id":"o-2",...}]
```

**Reverse lookup** (Order → who links to it):
```
SMEMBERS links:org1:Order:o-1:reverse
→ [{"source_type":"User","source_id":"u-123","field_name":"orders",...}]
```

### 7.4 Links in Query Responses

Links appear under `_links`. Consumers can optionally expand (inline) linked entities.

**Default (links as references):**
```json
{
  "data": {
    "uuid": "u-123",
    "name": "Alice",
    "_links": {
      "orders": [
        { "type": "Order", "id": "o-1" },
        { "type": "Order", "id": "o-2" }
      ],
      "primary_address": { "type": "Address", "id": "a-1" }
    }
  }
}
```

**Expanded** (`GET /entities/User/u-123?expand=orders`):
```json
{
  "data": {
    "uuid": "u-123",
    "name": "Alice",
    "_links": {
      "orders": [
        {
          "type": "Order", "id": "o-1",
          "_data": { "uuid": "o-1", "status": "shipped", "total": 49.99 }
        },
        {
          "type": "Order", "id": "o-2",
          "_data": { "uuid": "o-2", "status": "pending", "total": 24.99 }
        }
      ],
      "primary_address": { "type": "Address", "id": "a-1" }
    }
  }
}
```

**Reverse traversal** (`GET /entities/Order/o-1?reverse_links=true`):
```json
{
  "data": {
    "uuid": "o-1",
    "status": "shipped",
    "_reverse_links": [
      { "source_type": "User", "source_id": "u-123", "field_name": "orders" }
    ]
  }
}
```

### 7.5 Link Schema

Links contribute to the entity schema automatically:

```
User entity schema:
┌─────────────────────┬────────────────┬────────────────────┐
│ Field               │ Type           │ Owner              │
├─────────────────────┼────────────────┼────────────────────┤
│ name                │ string         │ core-service       │
│ email               │ string         │ core-service       │
│ _link:orders        │ → Order[]      │ orders-service     │
│ _link:primary_addr  │ → Address      │ core-service       │
└─────────────────────┴────────────────┴────────────────────┘
```

Link ownership follows the same rules as facet fields — if another service tries to create a link with the same `field_name` on the same entity type, it triggers an `OWNERSHIP_COLLISION`.

---

## 8. Query Interface

Entities are materialized, so queries are simple reads.

### 8.1 REST API

#### Get Entity

```
GET /entities/{entity_type}/{entity_id}
GET /entities/{entity_type}/{entity_id}?fields=name,email,billing_id
GET /entities/{entity_type}/{entity_id}?expand=orders,primary_address
GET /entities/{entity_type}/{entity_id}?reverse_links=true
```

**Response:**
```json
{
  "data": {
    "uuid": "u-123",
    "name": "Alice",
    "email": "alice@co.com",
    "billing_id": "b-1",
    "_links": {
      "orders": [
        { "type": "Order", "id": "o-1" },
        { "type": "Order", "id": "o-2" }
      ]
    }
  },
  "metadata": {
    "schema_version": 12,
    "entity_version": 42,
    "facets": {
      "core-service": "2026-03-19T11:00:00Z",
      "billing-service": "2026-03-19T10:30:00Z"
    }
  }
}
```

#### Batch Get

```
POST /entities/_batch
Content-Type: application/json

{
  "queries": [
    { "entity_type": "User", "entity_id": "u-1", "fields": ["name", "email"] },
    { "entity_type": "User", "entity_id": "u-2", "expand": ["orders"] },
    { "entity_type": "Order", "entity_id": "o-1" }
  ]
}
```

#### Nested Field Selection

For embedded objects, use dot notation:

```
GET /entities/User/u-123?fields=name,billing_address.street,billing_address.city
```

### 8.2 gRPC API

```protobuf
service QueryService {
  rpc GetEntity(GetEntityRequest) returns (GetEntityResponse);
  rpc GetEntityBatch(GetEntityBatchRequest) returns (GetEntityBatchResponse);
}

message GetEntityRequest {
  string entity_type = 1;
  string entity_id = 2;
  repeated string fields = 3;
  repeated string expand = 4;
  bool reverse_links = 5;
}

message GetEntityResponse {
  string json_data = 1;
  EntityMetadata metadata = 2;
}

message EntityMetadata {
  int64 schema_version = 1;
  int64 entity_version = 2;
  map<string, string> facet_timestamps = 3;
}

message GetEntityBatchRequest {
  repeated GetEntityRequest queries = 1;
}

message GetEntityBatchResponse {
  repeated GetEntityResponse results = 1;
}
```

---

## 9. Subscriptions

### 9.1 Change Events

When a facet push or link change modifies the materialized view, Fyodor emits a change event:

```json
{
  "entity_type": "User",
  "entity_id": "u-123",
  "entity_version": 43,
  "source_service": "billing-service",
  "timestamp": "2026-03-19T14:30:00Z",
  "change_type": "facet_update",
  "changed_fields": ["payment_status"],
  "data": {
    "payment_status": "suspended"
  }
}
```

Link change events:
```json
{
  "entity_type": "User",
  "entity_id": "u-123",
  "entity_version": 44,
  "source_service": "orders-service",
  "timestamp": "2026-03-19T14:31:00Z",
  "change_type": "link_added",
  "link": {
    "field_name": "orders",
    "target_type": "Order",
    "target_id": "o-3"
  }
}
```

### 9.2 Subscription Protocols

#### WebSocket

```
WS /subscribe
```

**Subscribe message:**
```json
{
  "action": "subscribe",
  "subscriptions": [
    { "entity_type": "User", "entity_id": "u-123" },
    { "entity_type": "User", "entity_id": "u-456", "fields": ["last_login"] },
    { "entity_type": "Order" }
  ]
}
```

#### Server-Sent Events

```
GET /subscribe?entity_type=User&entity_id=u-123
```

#### gRPC Streaming

```protobuf
service SubscriptionService {
  rpc Subscribe(SubscribeRequest) returns (stream ChangeEvent);
}

message SubscribeRequest {
  string entity_type = 1;
  string entity_id = 2;
  repeated string fields = 3;
}
```

### 9.3 Subscription Filtering

| Filter | WebSocket | SSE | gRPC Stream |
|--------|-----------|-----|-------------|
| By entity type | Per-subscription | Query param | Request field |
| By entity ID | Per-subscription | Query param | Request field |
| By changed fields | Per-subscription | Query param | Request field |
| By change type (facet/link) | Per-subscription | Query param | Request field |

---

## 10. Conflict Resolution

### 10.1 Conflict Types

| Type | Trigger | Example |
|------|---------|---------|
| `TYPE_MISMATCH` | Same service pushes a field with a different type | `payment_status` was `string`, now `number` |
| `OWNERSHIP_COLLISION` | Different service pushes a field owned by another | billing-service pushes `name`, but core-service owns it |

### 10.2 Conflict Queue

When a conflict is detected, the incoming value is held (not applied):

```json
{
  "conflict_id": "c-abc123",
  "entity_type": "User",
  "field": "payment_status",
  "type": "TYPE_MISMATCH",
  "existing": {
    "type": "string",
    "owner": "billing-service",
    "current_value": "active"
  },
  "incoming": {
    "type": "number",
    "service": "billing-service",
    "value": 1
  },
  "status": "pending",
  "created_at": "2026-03-19T14:30:00Z"
}
```

### 10.3 Resolution Strategies

| Strategy | Behavior | Applies To |
|----------|----------|------------|
| **Accept Incoming** | Apply the incoming value, update schema type/owner | Both |
| **Keep Existing** | Discard the incoming value, keep current state | Both |
| **Accept From Service** | Permanently assign field ownership to a specific service. Future pushes from other services are silently dropped. | `OWNERSHIP_COLLISION` |
| **Last Write Wins** | Accept the most recent push regardless of service. Removes ownership exclusivity. | `OWNERSHIP_COLLISION` |
| **First Write Wins** | Accept the first value pushed per entity, ignore subsequent updates from other services. | `OWNERSHIP_COLLISION` |
| **Append to Array** | Convert the field to an array, append values from all services. | `OWNERSHIP_COLLISION` |

### 10.4 Resolution API

```
POST /conflicts/{conflict_id}/resolve
Authorization: Bearer {admin_api_key_or_dashboard_session}
Content-Type: application/json

{
  "strategy": "accept_from_service",
  "params": {
    "service": "core-service"
  }
}
```

### 10.5 Resolution Persistence

Chosen strategies are stored as **rules** so future conflicts on the same field auto-resolve:

```
rules:{org_id}:{entity_type}:{field_name}    -> JSON {strategy, params, created_by, created_at}
```

### 10.6 Conflict Notifications

When a conflict is created:
- Dashboard shows a notification badge
- Webhook fires (configurable per org)
- Optional: Slack/email alerts

---

## 11. Facet SDK

All Facet SDKs wrap the HTTP push API (Section 6.1) and link API (Section 7.1).

### 11.1 Kotlin / Java

```kotlin
dependencies {
    implementation("com.fyodor:facet-sdk:1.0.0")
}
```

```kotlin
val fyodor = Fyodor.facet(
    serviceName = "billing-service",
    endpoint = "https://fyodor.internal",
    apiKey = "fyd_sk_billing_a1b2c3d4",
)

// Push
fyodor.push("User", "u-123", mapOf(
    "billing_id" to "b-1",
    "payment_status" to "active",
))

// Link
fyodor.link("User", "u-123", "orders", "Order", "o-456")

// Unlink
fyodor.unlink("User", "u-123", "orders", "Order", "o-456")
```

### 11.2 Python

```bash
pip install fyodor-facet
```

```python
from fyodor import Fyodor

fyodor = Fyodor.facet(
    service_name="analytics-service",
    endpoint="https://fyodor.internal",
    api_key="fyd_sk_analytics_e5f6g7h8",
)

fyodor.push("User", "u-123", {"last_event": "page_view", "event_count": 42})
fyodor.link("User", "u-123", "recent_orders", "Order", "o-789")
fyodor.unlink("User", "u-123", "recent_orders", "Order", "o-789")
```

### 11.3 Go

```bash
go get github.com/fyodor-platform/fyodor-go
```

```go
fyd := fyodor.NewFacetClient(fyodor.Config{
    ServiceName: "shipping-service",
    Endpoint:    "https://fyodor.internal",
    APIKey:      "fyd_sk_shipping_i9j0k1l2",
})

fyd.Push("Order", "o-123", map[string]any{"shipping_status": "in_transit", "carrier": "fedex"})
fyd.Link("User", "u-123", "orders", "Order", "o-123")
fyd.Unlink("User", "u-123", "orders", "Order", "o-123")
```

### 11.4 Ruby

```ruby
gem 'fyodor-facet', '~> 1.0'
```

```ruby
Fyodor.configure do |config|
  config.service_name = 'prefs-service'
  config.endpoint = 'https://fyodor.internal'
  config.api_key = 'fyd_sk_prefs_m3n4o5p6'
end

Fyodor.push('User', 'u-123', { locale: 'en-US', dark_mode: true })
Fyodor.link('User', 'u-123', 'settings_profile', 'SettingsProfile', 'sp-1')
Fyodor.unlink('User', 'u-123', 'settings_profile', 'SettingsProfile', 'sp-1')
```

### 11.5 Node.js / TypeScript

```bash
npm install @fyodor/facet
```

```typescript
import { Fyodor } from '@fyodor/facet';

const fyodor = new Fyodor.Facet({
  serviceName: 'notifications-service',
  endpoint: 'https://fyodor.internal',
  apiKey: 'fyd_sk_notif_q7r8s9t0',
});

await fyodor.push('User', 'u-123', { unread_count: 3 });
await fyodor.link('User', 'u-123', 'notifications', 'Notification', 'n-1');
await fyodor.unlink('User', 'u-123', 'notifications', 'Notification', 'n-1');
```

### 11.6 Rust

```toml
[dependencies]
fyodor-facet = "1.0"
```

```rust
use fyodor_facet::Fyodor;

let fyodor = Fyodor::facet()
    .service_name("auth-service")
    .endpoint("https://fyodor.internal")
    .api_key("fyd_sk_auth_u1v2w3x4")
    .build().await?;

fyodor.push("User", "u-123", serde_json::json!({"last_login": "2026-03-19T14:30:00Z"})).await?;
fyodor.link("User", "u-123", "active_sessions", "Session", "s-1").await?;
fyodor.unlink("User", "u-123", "active_sessions", "Session", "s-1").await?;
```

### 11.7 Frontend (Browser) — Push Only

```bash
npm install @fyodor/facet
```

The frontend Facet SDK uses JWT tokens for authentication. It can **push facets and manage links only** — no read access.

#### Vanilla JavaScript

```typescript
import { Fyodor } from '@fyodor/facet';

const fyodor = new Fyodor.Facet({
  endpoint: 'https://fyodor.example.com',
  // JWT must include fyodor_scope claim, e.g.:
  // { "sub": "u-123", "fyodor_scope": ["User:u-123", "UserActivity:*"], "exp": ... }
  getToken: () => myAuthService.getAccessToken(),
});

// Push user activity (allowed — scope includes UserActivity:*)
await fyodor.push('UserActivity', userId, {
  page_viewed: '/dashboard',
  timestamp: new Date().toISOString(),
});

// Push preferences (allowed — scope includes User:u-123)
await fyodor.push('User', userId, {
  dark_mode: true,
  locale: 'en-US',
});

// This would fail with SCOPE_DENIED if the token doesn't include Order:*
// await fyodor.push('Order', 'o-456', { status: 'cancelled' });

// Link (allowed if source entity matches scope)
await fyodor.link('User', userId, 'viewed_products', 'Product', 'p-123');
```

#### React

```typescript
import { FyodorFacetProvider, useFyodorPush } from '@fyodor/react';

function App() {
  return (
    <FyodorFacetProvider
      endpoint="https://fyodor.example.com"
      getToken={() => myAuthService.getAccessToken()}
    >
      <TrackingComponent />
    </FyodorFacetProvider>
  );
}

function TrackingComponent() {
  const { push, link } = useFyodorPush();

  const handleClick = async (productId: string) => {
    await push('UserActivity', userId, { last_clicked_product: productId });
    await link('User', userId, 'viewed_products', 'Product', productId);
  };

  return <button onClick={() => handleClick('p-123')}>View Product</button>;
}
```

### 11.8 Raw HTTP (No SDK)

```bash
# Push
curl -X POST https://fyodor.internal/facets/User/u-123 \
  -H "Authorization: Bearer fyd_sk_myservice_a1b2c3d4" \
  -H "Content-Type: application/json" \
  -d '{"last_login": "2026-03-19T14:30:00Z"}'

# Link
curl -X POST https://fyodor.internal/links \
  -H "Authorization: Bearer fyd_sk_myservice_a1b2c3d4" \
  -H "Content-Type: application/json" \
  -d '{"source_type":"User","source_id":"u-123","field_name":"orders","target_type":"Order","target_id":"o-456"}'

# Unlink
curl -X POST https://fyodor.internal/unlink \
  -H "Authorization: Bearer fyd_sk_myservice_a1b2c3d4" \
  -H "Content-Type: application/json" \
  -d '{"source_type":"User","source_id":"u-123","field_name":"orders","target_type":"Order","target_id":"o-456"}'
```

---

## 12. Client SDK

All Client SDKs wrap the REST query API (Section 8.1) and subscription protocols (Section 9).

### 12.1 Kotlin / Java

```kotlin
val client = Fyodor.client(endpoint = "https://fyodor.example.com", apiKey = "fyd_sk_...")

val user = client.get("User", "u-123")
val user = client.get("User", "u-123", fields = listOf("name", "email"))
val user = client.get("User", "u-123", expand = listOf("orders"))

client.subscribe("User", entityId = "u-123") { event ->
    println("${event.changedFields} updated by ${event.sourceService}")
}
```

### 12.2 Python

```python
client = Fyodor.client(endpoint="https://fyodor.example.com", api_key="fyd_sk_...")

user = client.get("User", "u-123")
user = client.get("User", "u-123", expand=["orders"])

for event in client.subscribe("User", entity_id="u-123"):
    print(f"{event.changed_fields} updated")
```

### 12.3 Go

```go
client := fyodor.NewClient(fyodor.ClientConfig{
    Endpoint: "https://fyodor.example.com",
    APIKey:   "fyd_sk_...",
})

user, _ := client.Get("User", "u-123", &fyodor.GetOpts{Expand: []string{"orders"}})

ch := client.Subscribe("User", &fyodor.SubOpts{EntityID: "u-123"})
for event := range ch {
    fmt.Printf("%v updated\n", event.ChangedFields)
}
```

### 12.4 Ruby

```ruby
client = Fyodor::Client.new(endpoint: 'https://fyodor.example.com', api_key: 'fyd_sk_...')

user = client.get('User', 'u-123', expand: [:orders])

client.subscribe('User', entity_id: 'u-123') do |event|
  puts "#{event.changed_fields} updated"
end
```

### 12.5 Node.js / TypeScript

```typescript
import { Fyodor } from '@fyodor/client';

const client = new Fyodor.Client({
  endpoint: 'https://fyodor.example.com',
  apiKey: 'fyd_sk_...',
});

const user = await client.get('User', 'u-123', { expand: ['orders'] });

client.subscribe('User', { entityId: 'u-123' }, (event) => {
  console.log(`${event.changedFields} updated`);
});
```

### 12.6 Rust

```rust
let client = Fyodor::client()
    .endpoint("https://fyodor.example.com")
    .api_key("fyd_sk_...")
    .build().await?;

let user = client.get("User", "u-123", Some(GetOpts { expand: vec!["orders"] })).await?;

let mut stream = client.subscribe("User", Some("u-123")).await?;
while let Some(event) = stream.next().await {
    println!("{:?} updated", event.changed_fields);
}
```

The Client SDK is backend-only. There is no frontend client for reading data.

For frontend **push** capabilities, see the Frontend Facet SDK (Section 11.8).

### 12.7 REST / cURL

```bash
# Get entity (API key required)
curl https://fyodor.example.com/entities/User/u-123 \
  -H "Authorization: Bearer fyd_sk_myservice_a1b2c3d4"

# With expanded links
curl https://fyodor.example.com/entities/User/u-123?expand=orders \
  -H "Authorization: Bearer fyd_sk_myservice_a1b2c3d4"

# Subscribe (SSE, API key required)
curl https://fyodor.example.com/subscribe?entity_type=User&entity_id=u-123 \
  -H "Authorization: Bearer fyd_sk_myservice_a1b2c3d4"
```

---

## 13. Observability

### 13.1 Metrics

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `fyodor_push_total` | Counter | `entity_type`, `service` | Facet pushes received |
| `fyodor_push_duration_seconds` | Histogram | `entity_type`, `service` | Push-to-materialization latency |
| `fyodor_push_conflicts_total` | Counter | `entity_type`, `conflict_type` | Conflicts detected |
| `fyodor_link_total` | Counter | `source_type`, `target_type`, `action` | Links created/removed |
| `fyodor_query_total` | Counter | `entity_type` | Queries served |
| `fyodor_query_duration_seconds` | Histogram | `entity_type` | Query latency |
| `fyodor_auth_failures_total` | Counter | `auth_type`, `reason` | Auth failures (invalid key, expired JWT, bad signature, scope denied) |
| `fyodor_subscription_active` | Gauge | `entity_type`, `protocol` | Active subscriptions |
| `fyodor_change_events_total` | Counter | `entity_type`, `change_type` | Change events emitted |
| `fyodor_entity_count` | Gauge | `entity_type` | Materialized entities |
| `fyodor_schema_fields` | Gauge | `entity_type` | Fields in inferred schema |
| `fyodor_conflicts_pending` | Gauge | `entity_type` | Unresolved conflicts |

### 13.2 Distributed Tracing

- Facet pushes carry trace context via HTTP headers
- Query requests propagate W3C Trace Context
- Change events include the originating trace ID
- JWT `sub` claim is attached to traces for frontend pushes

### 13.3 Logging

Structured JSON logging:

```json
{
  "timestamp": "2026-03-19T12:00:00.000Z",
  "level": "INFO",
  "service": "fyodor",
  "org_id": "org-acme",
  "trace_id": "abc123",
  "message": "Facet materialized",
  "entity_type": "User",
  "entity_id": "u-123",
  "source_service": "billing-service",
  "accepted_fields": ["payment_status"],
  "conflicts": [],
  "entity_version": 43,
  "duration_ms": 3
}
```

### 13.4 Schema Introspection

```
GET /schema
GET /schema/{entity_type}
```

```json
{
  "entity_type": "User",
  "schema_version": 12,
  "fields": [
    { "name": "name", "type": "string", "owner": "core-service", "created_at": "2026-01-15T..." },
    { "name": "billing_id", "type": "string", "owner": "billing-service", "created_at": "2026-01-20T..." },
    { "name": "dark_mode", "type": "boolean", "owner": "prefs-service", "created_at": "2026-02-01T..." }
  ],
  "links": [
    { "field_name": "orders", "target_type": "Order", "owner": "orders-service" },
    { "field_name": "primary_address", "target_type": "Address", "owner": "core-service" }
  ],
  "services": [
    { "name": "core-service", "fields_owned": 2, "links_owned": 1, "last_push": "2026-03-19T11:00:00Z" },
    { "name": "billing-service", "fields_owned": 2, "links_owned": 0, "last_push": "2026-03-19T10:30:00Z" }
  ],
  "pending_conflicts": 0
}
```

---

## 14. Error Handling

### 14.1 Push Errors

| Code | Description |
|------|-------------|
| `UNAUTHORIZED` | Invalid or missing API key |
| `UNKNOWN_ENTITY_TYPE` | Entity type not created in Dashboard |
| `INVALID_PAYLOAD` | Malformed JSON |
| `FIELD_CONFLICT` | Breaking change detected (queued, push partially succeeds) |

### 14.2 Link Errors

| Code | Description |
|------|-------------|
| `UNAUTHORIZED` | Invalid or missing API key |
| `UNKNOWN_ENTITY_TYPE` | Source or target entity type doesn't exist |
| `LINK_OWNERSHIP_COLLISION` | Another service owns this link field name |
| `LINK_ALREADY_EXISTS` | Duplicate link |
| `LINK_NOT_FOUND` | Unlink for non-existent link |

### 14.3 Query Errors

| Code | Description |
|------|-------------|
| `UNAUTHORIZED` | Invalid API key or JWT |
| `TOKEN_EXPIRED` | JWT has expired |
| `TOKEN_INVALID_SIGNATURE` | JWT signature doesn't match any configured public key |
| `ENTITY_NOT_FOUND` | No materialized view for this entity |
| `UNKNOWN_ENTITY_TYPE` | Entity type doesn't exist |
| `UNKNOWN_FIELD` | Requested field not in schema |
| `EXPAND_DEPTH_EXCEEDED` | Too many levels of link expansion |
| `SCOPE_DENIED` | JWT `fyodor_scope` does not permit push to this entity type/ID |
| `SCOPE_MISSING` | JWT is missing the required `fyodor_scope` claim |
| `FRONTEND_READ_DENIED` | Frontend token attempted query or subscribe (not permitted) |
| `INTERNAL_ERROR` | Unexpected error |

### 14.4 Staleness Metadata

Every query response includes per-service timestamps so consumers can assess freshness:

```json
{
  "data": { "uuid": "u-123", "name": "Alice", "payment_status": "active" },
  "metadata": {
    "facets": {
      "core-service": "2026-03-19T11:00:00Z",
      "billing-service": "2026-03-18T08:00:00Z"
    }
  }
}
```

---

## 15. API Reference

### 15.1 REST Endpoints

**Facet & Link Operations (require API key or verified JWT):**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/facets/{type}/{id}` | Push a facet |
| `POST` | `/facets/_batch` | Batch push facets |
| `POST` | `/links` | Create a link |
| `POST` | `/unlink` | Remove a link |
| `POST` | `/links/_batch` | Batch create links |

**Query Operations (require API key — no JWT access):**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/entities/{type}/{id}` | Get materialized entity |
| `POST` | `/entities/_batch` | Batch get entities |
| `GET` | `/schema` | List all entity schemas |
| `GET` | `/schema/{entity_type}` | Get inferred schema |
| `GET` | `/subscribe` | SSE subscription |
| `WS` | `/subscribe` | WebSocket subscription |

**Admin Operations (require admin API key or Dashboard session):**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/conflicts` | List pending conflicts |
| `GET` | `/conflicts/{id}` | Get conflict details |
| `POST` | `/conflicts/{id}/resolve` | Resolve a conflict |
| `GET` | `/services` | List known services |
| `GET` | `/health` | Health check |
| `GET` | `/metrics` | Prometheus metrics |

### 15.2 Dashboard API

These endpoints back the Dashboard UI and require a Dashboard session cookie or admin API key:

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/dashboard/entity-types` | Create entity type |
| `GET` | `/dashboard/entity-types` | List entity types |
| `DELETE` | `/dashboard/entity-types/{type}` | Delete entity type |
| `POST` | `/dashboard/api-keys` | Create API key |
| `GET` | `/dashboard/api-keys` | List API keys |
| `DELETE` | `/dashboard/api-keys/{id}` | Revoke API key |
| `POST` | `/dashboard/signing-keys` | Upload public signing key |
| `GET` | `/dashboard/signing-keys` | List signing keys |
| `PUT` | `/dashboard/signing-keys/{id}` | Activate/deactivate key |
| `DELETE` | `/dashboard/signing-keys/{id}` | Delete signing key |
| `PUT` | `/dashboard/signing-keys/jwks` | Configure JWKS URL |
| `POST` | `/dashboard/signing-keys/test` | Test a JWT token |
| `POST` | `/dashboard/invitations` | Invite user to org |
| `GET` | `/dashboard/members` | List org members |
| `PUT` | `/dashboard/members/{id}/role` | Change member role |
| `DELETE` | `/dashboard/members/{id}` | Remove member |
| `GET` | `/dashboard/stats` | Observability summary |

### 15.3 gRPC Services

| Service | Method | Description |
|---------|--------|-------------|
| `QueryService` | `GetEntity` | Read a materialized entity |
| `QueryService` | `GetEntityBatch` | Batch read entities |
| `SubscriptionService` | `Subscribe` | Stream change events |

---

## Appendix A: Redis Key Reference

```
# Inferred Schema
schema:{org_id}:{entity_type}:fields                 hash     field_name -> JSON {type, owner, created_at}
schema:{org_id}:{entity_type}:version                int64    Incremented on schema change
schema:{org_id}:{entity_type}:history                list     JSON diffs of schema changes

# Materialized Entity Views
entity:{org_id}:{entity_type}:{entity_id}            JSON     Merged facet data
entity:{org_id}:{entity_type}:{entity_id}:version    int64    Incrementing version
entity:{org_id}:{entity_type}:{entity_id}:facets     hash     service_name -> last_push_timestamp

# Links (Bidirectional)
links:{org_id}:{source_type}:{source_id}:forward     hash     field_name -> JSON array of targets
links:{org_id}:{target_type}:{target_id}:reverse     set      JSON {source_type, source_id, field_name}

# Conflicts
conflicts:{org_id}:{entity_type}:{conflict_id}       hash     Conflict details
conflicts:{org_id}:{entity_type}:pending              set      Conflict IDs

# Resolution Rules
rules:{org_id}:{entity_type}:{field_name}            JSON     {strategy, params, created_by}

# Auth (cached from PostgreSQL)
auth:apikey:{key_hash}                                JSON     {org_id, service_name, entity_types, status}
auth:signingkeys:{org_id}                             JSON     [{key_id, algorithm, public_key_pem, status}]
auth:jwks:{org_id}:cached                             JSON     Cached JWKS response
auth:jwks:{org_id}:expires                            string   Next refresh timestamp
```

## Appendix B: PostgreSQL Schema

```sql
-- Organizations
CREATE TABLE orgs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        TEXT NOT NULL,
    slug        TEXT UNIQUE NOT NULL,
    created_at  TIMESTAMPTZ DEFAULT now()
);

-- User accounts
CREATE TABLE users (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email         TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    created_at    TIMESTAMPTZ DEFAULT now()
);

-- Org membership
CREATE TABLE org_memberships (
    org_id    UUID REFERENCES orgs(id),
    user_id   UUID REFERENCES users(id),
    role      TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'member')),
    PRIMARY KEY (org_id, user_id)
);

-- Entity types (created in Dashboard)
CREATE TABLE entity_types (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id      UUID REFERENCES orgs(id),
    name        TEXT NOT NULL,
    description TEXT,
    id_format   TEXT,  -- optional regex for entity ID validation
    created_at  TIMESTAMPTZ DEFAULT now(),
    UNIQUE (org_id, name)
);

-- Service API keys
CREATE TABLE api_keys (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id        UUID REFERENCES orgs(id),
    service_name  TEXT NOT NULL,
    key_hash      TEXT UNIQUE NOT NULL,
    entity_types  TEXT[],  -- null = all types allowed
    status        TEXT NOT NULL DEFAULT 'active',
    created_at    TIMESTAMPTZ DEFAULT now()
);

-- Public signing keys for frontend JWT verification
CREATE TABLE signing_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID REFERENCES orgs(id),
    key_id          TEXT NOT NULL,
    algorithm       TEXT NOT NULL,
    public_key_pem  TEXT,       -- null if using JWKS
    jwks_url        TEXT,       -- null if using direct key
    status          TEXT NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ DEFAULT now()
);

-- Audit log
CREATE TABLE audit_log (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id      UUID REFERENCES orgs(id),
    actor       TEXT NOT NULL,  -- user email or service name
    action      TEXT NOT NULL,  -- e.g. 'create_entity_type', 'resolve_conflict', 'revoke_key'
    resource    TEXT,           -- e.g. 'entity_type:User', 'api_key:fyd_sk_...'
    details     JSONB,
    timestamp   TIMESTAMPTZ DEFAULT now()
);
```

---

*End of Technical Specification*
