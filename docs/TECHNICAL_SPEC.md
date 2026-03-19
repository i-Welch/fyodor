# Data Mesh Platform - Technical Specification

**Version**: 1.0.0-draft
**Last Updated**: 2026-01-30

---

## Table of Contents

1. [Overview](#1-overview)
2. [System Architecture](#2-system-architecture)
3. [Schema Management](#3-schema-management)
4. [Router Service](#4-router-service)
5. [Registration Protocol](#5-registration-protocol)
6. [Query System (LISP DSL)](#6-query-system-lisp-dsl)
7. [Data Exposer SDK](#7-data-exposer-sdk)
8. [Push Data Handling (Kafka)](#8-push-data-handling-kafka)
9. [Client SDK](#9-client-sdk)
10. [Observability](#10-observability)
11. [Error Handling](#11-error-handling)
12. [API Reference](#12-api-reference)

---

## 1. Overview

### 1.1 Purpose

The Data Mesh Platform enables distributed services to expose data about shared entities through a unified query interface. Inspired by Apollo GraphQL Federation, it allows multiple services to extend entity types while hiding the complexity of data aggregation from consumers.

### 1.2 Core Principles

- **Entity-Centric**: All data is organized around entities with stable UUID identifiers
- **Distributed Ownership**: Services own their data and schema extensions
- **Unified Access**: Consumers query a single interface without knowing which services provide which fields
- **Protocol Flexibility**: Supports gRPC, REST, and Kafka for data ingestion
- **Type Safety**: Protobuf-based schemas with compile-time type generation

### 1.3 Key Components

| Component | Responsibility |
|-----------|----------------|
| **Router Service** | Schema aggregation, query routing, response assembly |
| **Data Exposer SDK** | Library for services to expose entity extensions |
| **Client SDK** | Typed query interface for consumers |
| **Schema Registry** | Redis-backed storage for aggregated schemas |

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
                                    ┌─────────────────────────────────────┐
                                    │            CLIENTS                  │
                                    │  ┌─────────┐       ┌─────────────┐  │
                                    │  │ Typed   │       │ Ad-hoc      │  │
                                    │  │ (Proto) │       │ (JSON/REST) │  │
                                    │  └────┬────┘       └──────┬──────┘  │
                                    └───────┼──────────────────┼─────────┘
                                            │                  │
                                            ▼                  ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                              ROUTER SERVICE                                   │
│                                                                               │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────┐  │
│  │ Base Entity    │  │ Schema         │  │ Query          │  │ Response   │  │
│  │ Definitions    │  │ Aggregator     │  │ Parser         │  │ Assembler  │  │
│  │ (Proto)        │  │                │  │ (LISP DSL)     │  │            │  │
│  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘  └─────┬──────┘  │
│          │                   │                   │                 │          │
│          │           ┌───────┴────────┐          │                 │          │
│          │           │ Registration   │          │                 │          │
│          │           │ Handler        │          │                 │          │
│          │           └───────┬────────┘          │                 │          │
│          │                   │                   │                 │          │
│  ┌───────┴───────────────────┴───────────────────┴─────────────────┴───────┐  │
│  │                              REDIS                                      │  │
│  │  • Schema Registry (aggregated descriptors)                             │  │
│  │  • Service Registry (endpoints, health status)                          │  │
│  │  • Push Data Cache (Kafka events)                                       │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
        │                    │                    │                    │
        │ gRPC               │ gRPC               │ gRPC               │ Kafka
        ▼                    ▼                    ▼                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Service A   │    │  Service B   │    │  Service C   │    │  Service D   │
│  (Kotlin)    │    │  (Ruby)      │    │  (Kotlin)    │    │  (Push)      │
│              │    │              │    │              │    │              │
│ Extends:User │    │ Extends:User │    │Extends:Order │    │ Extends:User │
│ billing_id   │    │ preferences  │    │ shipping_eta │    │ last_login   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

### 2.2 Data Flow

#### Query Flow (Pull)

```
1. Client sends LISP query to Router (gRPC or REST)
2. Router parses query, identifies required fields
3. Router determines which services own which fields
4. Router executes parallel requests to relevant services
5. Router assembles responses into unified result
6. Router returns aggregated response (Proto or JSON)
```

#### Registration Flow

```
1. Service deploys with Data Exposer SDK
2. SDK calls Router registration API with FileDescriptorSet
3. Router validates schema (no collisions, valid extension of base entity)
4. Router aggregates schema into unified descriptor
5. Router stores in Redis, bumps schema version
6. Router begins routing queries to new service
```

#### Push Flow (Kafka)

```
1. Service publishes event to entity-type topic
2. Router consumes event, validates against registered schema
3. Router stores event data in Redis cache
4. Subsequent queries return cached push data merged with pull data
```

### 2.3 Deployment Model

- **Router**: Horizontally scalable, stateless instances
- **Redis**: Clustered for HA, serves as shared state
- **Services**: Independent deployment, register on startup

---

## 3. Schema Management

### 3.1 Base Entity Definitions

Base entities are defined in the Router service repository using Protobuf:

```protobuf
// router/proto/entities/user.proto
syntax = "proto3";
package datamesh.entities;

message User {
  string uuid = 1;
}

// router/proto/entities/order.proto
message Order {
  string uuid = 1;
}
```

**Rules for base entities**:
- Must include `uuid` as field 1
- Define only universal fields; domain-specific fields go in extensions

### 3.2 Schema Extensions

Services define extensions in their own repositories:

```protobuf
// billing-service/proto/user_billing_extension.proto
syntax = "proto3";
package billing.extensions;

import "datamesh/entities/user.proto";

// Metadata annotation for router
option (datamesh.extends) = "datamesh.entities.User";

message UserBillingExtension {
  // Entity identifier (required, must match base entity)
  string uuid = 1;

  // Extension fields
  string billing_id = 2;
  string payment_method = 3;
  BillingAddress billing_address = 4;
  PaymentStatus payment_status = 5;
}

message BillingAddress {
  string street = 1;
  string city = 2;
  string country = 3;
  string postal_code = 4;
}

enum PaymentStatus {
  PAYMENT_STATUS_UNSPECIFIED = 0;
  PAYMENT_STATUS_ACTIVE = 1;
  PAYMENT_STATUS_SUSPENDED = 2;
  PAYMENT_STATUS_CANCELLED = 3;
}
```

### 3.3 Schema Aggregation Algorithm

When a service registers, the Router performs:

```
function aggregateSchema(baseEntity, newExtension):
    aggregated = copy(currentAggregatedSchema[baseEntity])
    serviceName = newExtension.serviceName

    // Step 1: Check if this is a re-registration (redeploy)
    isReregistration = serviceName in aggregated.registeredServices
    previousFields = []

    if isReregistration:
        // Collect fields previously owned by this service
        for fieldName, owner in aggregated.fieldToService:
            if owner == serviceName:
                previousFields.add(fieldName)

    // Step 2: Validate new fields against OTHER services' fields
    for field in newExtension.fields:
        if field.name == "uuid":
            continue  // Skip entity identifier

        if field.name in aggregated.fields:
            existingOwner = aggregated.fieldToService[field.name]

            // Allow if same service is re-registering its own field
            if existingOwner == serviceName:
                continue

            // Reject collision with another service's field
            return Error("Field collision: '${field.name}' already defined by '${existingOwner}'")

    // Step 3: Remove previous fields from this service (for redeploys)
    if isReregistration:
        for fieldName in previousFields:
            aggregated.fields.remove(fieldName)
            aggregated.fieldToService.remove(fieldName)

    // Step 4: Add new/updated fields
    for field in newExtension.fields:
        if field.name == "uuid":
            continue

        if field.number in aggregated.usedFieldNumbers:
            // Assign new field number in aggregated schema
            field.aggregatedNumber = nextAvailableNumber(aggregated)

        aggregated.fields.add(field)
        aggregated.fieldToService[field.name] = serviceName

    // Step 5: Track registered services
    aggregated.registeredServices.add(serviceName)

    // Step 6: Compute version based on changes
    removedFields = previousFields - newExtension.fieldNames
    addedFields = newExtension.fieldNames - previousFields
    changedTypes = detectTypeChanges(previousFields, newExtension)

    aggregated.version = computeVersion(aggregated, removedFields, addedFields, changedTypes)
    return aggregated
```

**Re-registration scenarios:**

| Scenario | Behavior |
|----------|----------|
| Same schema, redeploy | No version change, fields unchanged |
| New fields added | Minor version bump |
| Fields removed | Major version bump |
| Field types changed | Major version bump |
| Field added that collides with another service | Rejected with error |

### 3.4 Schema Versioning

Versions follow semantic versioning based on schema changes:

| Change Type | Version Bump | Example |
|-------------|--------------|---------|
| New field added | Minor | 1.2.0 → 1.3.0 |
| Field removed | Major | 1.2.0 → 2.0.0 |
| Field type changed | Major | 1.2.0 → 2.0.0 |
| Field renamed | Major | 1.2.0 → 2.0.0 |
| No change | None | 1.2.0 → 1.2.0 |

**Version identifier format**: `{major}.{minor}.{patch}-{content-hash}`

Example: `1.3.0-a1b2c3d4`

The content hash is SHA256 of the serialized `FileDescriptorSet`, truncated to 8 characters.

### 3.5 Schema Storage (Redis)

```
# Aggregated schema per entity type
schema:{entity_type}:descriptor    -> bytes (FileDescriptorSet)
schema:{entity_type}:version       -> string (semantic version)
schema:{entity_type}:hash          -> string (content hash)
schema:{entity_type}:field_map     -> hash (field_name -> service_name)

# Schema history for client compatibility
schema:{entity_type}:versions      -> sorted set (version -> timestamp)
schema:{entity_type}:v:{version}   -> bytes (historical FileDescriptorSet)

# Service registry
services:{service_name}:endpoint   -> string (gRPC endpoint)
services:{service_name}:health     -> string (health endpoint)
services:{service_name}:status     -> string (healthy|unhealthy)
services:{service_name}:entities   -> set (entity types this service extends)
services:{service_name}:descriptor -> bytes (service's FileDescriptorSet)
```

---

## 4. Router Service

### 4.1 Responsibilities

1. **Schema Management**: Aggregate and serve unified schemas
2. **Query Routing**: Parse queries, dispatch to relevant services
3. **Response Assembly**: Merge responses from multiple services
4. **Health Monitoring**: Track service health via healthchecks
5. **Push Data Storage**: Cache Kafka events in Redis
6. **Introspection**: Expose schema for exploration

### 4.2 Router Configuration

```yaml
# router-config.yaml
server:
  grpc_port: 9090
  rest_port: 8080

redis:
  addresses:
    - redis-cluster:6379
  password: ${REDIS_PASSWORD}

kafka:
  brokers:
    - kafka:9092
  consumer_group: datamesh-router

health_check:
  interval_seconds: 30
  timeout_seconds: 5
  unhealthy_threshold: 3

schema:
  base_entities_path: /proto/entities
  max_versions_retained: 10

query:
  max_batch_size: 100
  parallel_execution_limit: 20
  timeout_seconds: 30
  # Nested entity reference limits
  max_reference_depth: 5          # Max levels of entity nesting
  max_total_entities: 1000        # Max entities resolved per query
```

### 4.3 Query Execution Engine

```
function executeQuery(query):
    parsed = parseLispQuery(query)
    entityType = parsed.type
    entityId = parsed.id
    requestedFields = parsed.fields

    // Determine which services to call
    serviceCalls = []
    for field in requestedFields:
        service = schemaRegistry.getServiceForField(entityType, field)
        if service not in serviceCalls:
            serviceCalls[service] = {
                entityId: entityId,
                fields: []
            }
        serviceCalls[service].fields.add(field)

    // Execute in parallel
    responses = parallel_execute(serviceCalls)

    // Check for push data in Redis
    pushData = redis.get("push:{entityType}:{entityId}")

    // Assemble response
    result = assembleResponse(responses, pushData, requestedFields)

    return result
```

### 4.4 Service Health Monitoring

The Router periodically polls registered services:

```
function healthCheck(service):
    response = http.get(service.healthEndpoint, timeout=5s)

    if response.status != 200:
        markUnhealthy(service)
        return

    // Validate schema matches registration
    reportedSchema = response.body.schema  // FileDescriptorSet
    registeredSchema = redis.get("services:{service}:descriptor")

    if hash(reportedSchema) != hash(registeredSchema):
        log.warn("Schema mismatch for service {service}")
        markUnhealthy(service)
        return

    markHealthy(service)
```

### 4.5 Entity References and Nested Resolution

Extensions can define fields that reference other entity types. The Router must resolve these references in a breadth-first manner, gathering entity IDs at each level before resolving the next.

#### 4.5.1 Declaring Entity References

Entity references are declared using a message type with the `datamesh.entity_ref` protobuf field option. The reference message must contain a `uuid` field that the Router uses to resolve the referenced entity.

```protobuf
// orders-service/proto/user_orders_extension.proto
syntax = "proto3";
package orders.extensions;

import "datamesh/options.proto";

option (datamesh.extension) = {
  entity_type: "datamesh.entities.User"
  service_name: "orders-service"
};

message UserOrdersExtension {
  string uuid = 1;

  // Single reference to an Order entity
  OrderRef primary_order = 2 [(datamesh.entity_ref) = "datamesh.entities.Order"];

  // List of references to Order entities
  repeated OrderRef orders = 3 [(datamesh.entity_ref) = "datamesh.entities.Order"];
}

// Reference message MUST contain a 'uuid' field as string at field number 1
message OrderRef {
  string uuid = 1;              // REQUIRED - Router extracts this to resolve the Order entity

  // Optional inline metadata (returned as-is, not resolved)
  string role = 2;              // e.g., "primary", "gift", "subscription"
  string added_at = 3;          // When this order was linked to user
}
```

**Reference Message Requirements:**

All entity reference fields must use a message type. The Router validates at registration time:

| Rule | Error Code |
|------|------------|
| Field must be a message type | `ENTITY_REF_MUST_BE_MESSAGE` |
| Message must have a field named `uuid` | `ENTITY_REF_MISSING_UUID` |
| The `uuid` field must be of type `string` | `ENTITY_REF_UUID_NOT_STRING` |
| The `uuid` field must be field number 1 | `ENTITY_REF_UUID_WRONG_NUMBER` |

**Registration Validation Pseudocode:**

```
function validateEntityRefField(field, referencedEntityType):
    if field.type != MESSAGE:
        return Error("ENTITY_REF_MUST_BE_MESSAGE: Entity reference field '${field.name}' " +
                     "must be a message type, found ${field.type}")

    messageType = field.messageType

    uuidField = messageType.findField("uuid")
    if uuidField == null:
        return Error("ENTITY_REF_MISSING_UUID: Message '${messageType.name}' used as " +
                     "entity reference must contain a 'uuid' field")

    if uuidField.type != STRING:
        return Error("ENTITY_REF_UUID_NOT_STRING: Field 'uuid' in '${messageType.name}' " +
                     "must be of type string, found ${uuidField.type}")

    if uuidField.number != 1:
        return Error("ENTITY_REF_UUID_WRONG_NUMBER: Field 'uuid' in '${messageType.name}' " +
                     "must be field number 1, found ${uuidField.number}")

    return OK
```

**Query Field Selection:**

When querying entity references, you can select both inline metadata fields and resolved entity fields:

```lisp
(get :type User :id "u-1"
     :fields [(primary_order [role added_at   ; inline metadata from OrderRef
                              status total])]) ; resolved from Order entity
```

The Router distinguishes between:
- **Inline fields**: Defined in the reference message (e.g., `role`, `added_at`) - returned directly from the extension service
- **Resolved fields**: Defined in the referenced entity's schema (e.g., `status`, `total`) - triggers entity resolution using the `uuid`

#### 4.5.2 Query Execution with Nested Entities

When a query requests fields from referenced entities, the Router builds an execution graph:

```lisp
(get :type User :id "user-123"
     :fields [name email
              (orders [status created_at
                       (shipping [carrier tracking_number eta])])])
```

This query requires:
1. **Level 0**: Fetch `name`, `email` from User extensions
2. **Level 0**: Fetch `orders` from orders-service (returns `OrderRef` messages with UUIDs)
3. **Level 1**: For each Order UUID, fetch `status`, `created_at` from Order extensions
4. **Level 1**: Fetch `shipping` from Order (returns `ShippingRef` message with UUID)
5. **Level 2**: For each Shipping UUID, fetch `carrier`, `tracking_number`, `eta`

#### 4.5.3 Breadth-First Execution Algorithm

```
function executeQueryWithReferences(query):
    parsed = parseLispQuery(query)

    // Build execution plan as a DAG of resolution levels
    plan = buildExecutionPlan(parsed)

    // Level 0: Root entity
    currentLevel = [{
        entityType: parsed.type,
        entityIds: [parsed.id],
        requestedFields: plan.getFieldsForLevel(0)
    }]

    results = {}
    level = 0

    while currentLevel is not empty:
        // Execute all fetches for current level in parallel
        levelResults = parallelFetchLevel(currentLevel)

        // Merge results
        for result in levelResults:
            results[result.entityId] = merge(results[result.entityId], result.data)

        // Collect entity references for next level
        nextLevel = []
        for resolution in currentLevel:
            for field in resolution.requestedFields:
                if isEntityReference(field):
                    // Gather all referenced entity IDs from results
                    referencedIds = collectEntityIds(levelResults, field)

                    if referencedIds is not empty:
                        nextLevel.add({
                            entityType: field.referencedType,
                            entityIds: deduplicate(referencedIds),
                            requestedFields: plan.getFieldsForReference(field),
                            parentField: field  // For result assembly
                        })

        currentLevel = nextLevel
        level++

    // Assemble nested response structure
    return assembleNestedResponse(results, plan)

function parallelFetchLevel(resolutions):
    // Group by entity type and service for efficient batching
    batches = groupByEntityTypeAndService(resolutions)

    // Execute all batches in parallel
    return parallel_execute_all(batches)
```

#### 4.5.4 Execution Example

For the query:
```lisp
(get :type User :id "user-123"
     :fields [name (orders [status (items [product_name price])])])
```

**Execution timeline:**

```
Level 0 (parallel):
├── user-core-service.fetch("user-123", [name])
│   → {name: "Alice"}
└── orders-service.fetch("user-123", [orders])
    → {orders: [{uuid: "ord-1"}, {uuid: "ord-2"}]}  // OrderRef messages

Level 1 (parallel):
├── order-status-service.fetchBatch(["ord-1", "ord-2"], [status])
│   → {"ord-1": {status: "shipped"}, "ord-2": {status: "pending"}}
└── order-items-service.fetchBatch(["ord-1", "ord-2"], [items])
    → {"ord-1": {items: [{uuid: "item-a"}, {uuid: "item-b"}]},
       "ord-2": {items: [{uuid: "item-c"}]}}  // ItemRef messages

Level 2 (parallel):
└── product-service.fetchBatch(["item-a", "item-b", "item-c"], [product_name, price])
    → {"item-a": {product_name: "Widget", price: 9.99}, ...}
```

Note: At each level, the Router extracts UUIDs from the reference messages to resolve the next level.

**Assembled response:**
```json
{
  "data": {
    "uuid": "user-123",
    "name": "Alice",
    "orders": [
      {
        "uuid": "ord-1",
        "status": "shipped",
        "items": [
          {"uuid": "item-a", "product_name": "Widget", "price": 9.99},
          {"uuid": "item-b", "product_name": "Gadget", "price": 19.99}
        ]
      },
      {
        "uuid": "ord-2",
        "status": "pending",
        "items": [
          {"uuid": "item-c", "product_name": "Gizmo", "price": 14.99}
        ]
      }
    ]
  }
}
```

#### 4.5.5 Reference Field Resolution

When a field is marked as an entity reference, the Router:

1. **Fetches the reference field** - Gets the list of UUIDs from the extending service
2. **Expands to full entities** - Resolves each UUID as a full entity query
3. **Applies field selection** - Only fetches the nested fields specified in the query

The extending service only needs to return the UUIDs; it doesn't need to know about the referenced entity's schema.

#### 4.5.6 Circular Reference Protection

The Router prevents infinite loops from circular references:

```
config:
  query:
    max_reference_depth: 5      # Maximum nesting levels
    max_total_entities: 1000    # Maximum entities resolved per query
```

If limits are exceeded, the query fails with `QUERY_TOO_COMPLEX` error.

#### 4.5.7 Data Exposer SDK: Reference Fields

The SDK handles reference fields transparently. Services return reference messages containing UUIDs (and optional inline metadata), and the Router handles resolving the full entity details:

```kotlin
// orders-service implementation
class UserOrdersExposer : UserOrdersExposerInterface {
    override suspend fun resolve(
        entityId: String,
        requestedFields: Set<String>
    ): UserOrdersExtension? {
        // Fetch order associations for this user
        val orderAssociations = orderRepository.getOrdersForUser(entityId)

        // Build reference messages with UUIDs and inline metadata
        val orderRefs = orderAssociations.map { assoc ->
            OrderRef.newBuilder()
                .setUuid(assoc.orderId)           // Required: Router uses this to resolve Order
                .setRole(assoc.role)              // Optional inline metadata
                .setLinkedAt(assoc.linkedAt)      // Optional inline metadata
                .build()
        }

        return UserOrdersExtension.newBuilder()
            .setUuid(entityId)
            .addAllOrders(orderRefs)
            .build()
    }
}
```

The service doesn't need to fetch Order details (status, items, etc.)—it just returns `OrderRef` messages with UUIDs. The Router extracts the UUIDs and resolves the full Order entity for any additional requested fields.

---

## 5. Registration Protocol

### 5.1 Registration API

```protobuf
// router/proto/registration.proto
syntax = "proto3";
package datamesh.router;

service RegistrationService {
  // Register a new schema extension
  rpc RegisterExtension(RegisterExtensionRequest) returns (RegisterExtensionResponse);

  // Deregister (for graceful shutdown)
  rpc DeregisterExtension(DeregisterExtensionRequest) returns (DeregisterExtensionResponse);

  // Get current aggregated schema
  rpc GetSchema(GetSchemaRequest) returns (GetSchemaResponse);
}

message RegisterExtensionRequest {
  // Service identifier
  string service_name = 1;

  // Entity type being extended (e.g., "datamesh.entities.User")
  string entity_type = 2;

  // Compiled protobuf descriptor
  bytes file_descriptor_set = 3;

  // Endpoint for data fetching (gRPC)
  string data_endpoint = 4;

  // Endpoint for health checks (HTTP)
  string health_endpoint = 5;

  // For push-based sources
  PushConfig push_config = 6;
}

message PushConfig {
  // Kafka topic to consume from
  string kafka_topic = 1;

  // Merge strategy for this entity type
  MergeStrategy merge_strategy = 2;

  // Maximum events to retain per entity
  int32 max_events_per_entity = 3;
}

enum MergeStrategy {
  MERGE_STRATEGY_UNSPECIFIED = 0;
  MERGE_STRATEGY_LAST_WINS = 1;      // Latest event overwrites
  MERGE_STRATEGY_FIRST_WINS = 2;      // First event is canonical
  MERGE_STRATEGY_MERGE_NON_NULL = 3;  // Only overwrite non-null values
}

message RegisterExtensionResponse {
  bool success = 1;
  string error_message = 2;

  // New aggregated schema version
  string schema_version = 3;

  // Warnings (e.g., deprecation notices)
  repeated string warnings = 4;
}

message DeregisterExtensionRequest {
  string service_name = 1;
  string entity_type = 2;
}

message DeregisterExtensionResponse {
  bool success = 1;
  string schema_version = 2;
}

message GetSchemaRequest {
  // Optional: specific entity type
  string entity_type = 1;

  // Optional: specific version (default: latest)
  string version = 2;
}

message GetSchemaResponse {
  bytes file_descriptor_set = 1;
  string version = 2;

  // Mapping of field names to owning services
  map<string, string> field_ownership = 3;
}
```

### 5.2 Registration Validation Rules

| Rule | Error |
|------|-------|
| Entity type must be a known base entity | `UNKNOWN_ENTITY_TYPE` |
| Field name collision with existing field | `FIELD_COLLISION: field '{name}' already defined by service '{service}'` |
| Invalid protobuf descriptor | `INVALID_DESCRIPTOR: {parse_error}` |
| Missing required `uuid` field | `MISSING_UUID_FIELD` |
| Health endpoint unreachable | `HEALTH_CHECK_FAILED` |

### 5.3 Registration Sequence

```
┌─────────┐          ┌─────────┐          ┌───────┐
│ Service │          │ Router  │          │ Redis │
└────┬────┘          └────┬────┘          └───┬───┘
     │                    │                   │
     │ RegisterExtension  │                   │
     │───────────────────>│                   │
     │                    │                   │
     │                    │ Validate schema   │
     │                    │──────────────────>│
     │                    │                   │
     │                    │ Check collisions  │
     │                    │<──────────────────│
     │                    │                   │
     │                    │ Store descriptor  │
     │                    │──────────────────>│
     │                    │                   │
     │                    │ Update aggregated │
     │                    │──────────────────>│
     │                    │                   │
     │                    │ Health check      │
     │<───────────────────│                   │
     │                    │                   │
     │ Health response    │                   │
     │───────────────────>│                   │
     │                    │                   │
     │                    │ Mark healthy      │
     │                    │──────────────────>│
     │                    │                   │
     │ RegisterResponse   │                   │
     │<───────────────────│                   │
     │                    │                   │
```

---

## 6. Query System (LISP DSL)

### 6.1 Grammar Specification

```ebnf
query       = "(" operation ")" ;
operation   = get | batch ;

get         = "get" ":type" entity_type ":id" string_literal [":fields" field_list] ;
batch       = "batch" get+ ;

entity_type = identifier ;
field_list  = "[" field* "]" ;
field       = identifier | nested_field ;
nested_field = "(" identifier field_list ")" ;

identifier  = letter (letter | digit | "_")* ;
string_literal = '"' character* '"' ;

letter      = "a"-"z" | "A"-"Z" ;
digit       = "0"-"9" ;
character   = any printable character except '"' ;
```

### 6.2 Query Examples

**Get single entity with all fields:**
```lisp
(get :type User :id "550e8400-e29b-41d4-a716-446655440000")
```

**Get single entity with selected fields:**
```lisp
(get :type User :id "550e8400-e29b-41d4-a716-446655440000"
     :fields [name email billing_id])
```

**Get entity with nested field selection:**
```lisp
(get :type User :id "550e8400-e29b-41d4-a716-446655440000"
     :fields [name email (billing_address [street city])])
```

**Batch query multiple entities:**
```lisp
(batch
  (get :type User :id "uuid-1" :fields [name email])
  (get :type User :id "uuid-2" :fields [name billing_id])
  (get :type Order :id "uuid-3" :fields [status shipping_eta]))
```

### 6.3 Query Transport

#### gRPC Interface

```protobuf
// router/proto/query.proto
syntax = "proto3";
package datamesh.router;

service QueryService {
  rpc Execute(QueryRequest) returns (QueryResponse);
  rpc ExecuteStream(QueryRequest) returns (stream QueryResponse);
}

message QueryRequest {
  string query = 1;  // LISP DSL query string

  // Optional: schema version to use (for typed clients)
  string schema_version = 2;
}

message QueryResponse {
  // Successful data (dynamic proto or JSON)
  oneof result {
    bytes proto_data = 1;      // For typed clients
    string json_data = 2;      // For ad-hoc clients
  }

  // Errors encountered during execution
  QueryErrors errors = 3;

  // Metadata
  QueryMetadata metadata = 4;
}

message QueryErrors {
  repeated QueryError errors = 1;
}

message QueryError {
  string entity_id = 1;
  string field = 2;
  string service_name = 3;
  string error_code = 4;
  string error_message = 5;
}

message QueryMetadata {
  string schema_version = 1;
  int64 execution_time_ms = 2;
}
```

#### REST Interface

**Endpoint**: `POST /query`

**Request**:
```json
{
  "query": "(get :type User :id \"uuid-here\" :fields [name email])",
  "schema_version": "1.3.0-a1b2c3d4"
}
```

**Response**:
```json
{
  "data": {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    "email": "john@example.com"
  },
  "errors": null,
  "metadata": {
    "schema_version": "1.3.0-a1b2c3d4",
    "execution_time_ms": 45
  }
}
```

---

## 7. Data Exposer SDK

### 7.1 Kotlin SDK

#### Installation

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.datamesh:exposer-sdk:1.0.0")
    implementation("com.datamesh:exposer-codegen:1.0.0")
}

// Enable code generation
protobuf {
    plugins {
        create("datamesh") {
            artifact = "com.datamesh:protoc-gen-datamesh:1.0.0"
        }
    }
}
```

#### Generated Interface

From the protobuf extension definition, the SDK generates:

```kotlin
// Generated from user_billing_extension.proto
interface UserBillingExposer {
    /**
     * Resolve billing data for a single user.
     *
     * @param entityId The user's UUID
     * @param requestedFields Fields requested by the query
     * @return Partial billing data for the user
     */
    suspend fun resolve(
        entityId: String,
        requestedFields: Set<String>
    ): UserBillingExtension?

    /**
     * Optional: Batch resolve for efficiency.
     * Default implementation calls resolve() for each ID.
     *
     * @param entityIds List of user UUIDs
     * @param requestedFields Fields requested by the query
     * @return Map of UUID to billing data
     */
    suspend fun resolveBatch(
        entityIds: List<String>,
        requestedFields: Set<String>
    ): Map<String, UserBillingExtension?> {
        return entityIds.associateWith { resolve(it, requestedFields) }
    }
}
```

#### Implementation Example

```kotlin
class BillingExposerImpl(
    private val billingRepository: BillingRepository
) : UserBillingExposer {

    override suspend fun resolve(
        entityId: String,
        requestedFields: Set<String>
    ): UserBillingExtension? {
        val billing = billingRepository.findByUserId(entityId)
            ?: return null

        return UserBillingExtension.newBuilder().apply {
            uuid = entityId

            if ("billing_id" in requestedFields) {
                billingId = billing.billingId
            }
            if ("payment_method" in requestedFields) {
                paymentMethod = billing.paymentMethod
            }
            if ("billing_address" in requestedFields) {
                billingAddress = BillingAddress.newBuilder().apply {
                    street = billing.address.street
                    city = billing.address.city
                    country = billing.address.country
                    postalCode = billing.address.postalCode
                }.build()
            }
            if ("payment_status" in requestedFields) {
                paymentStatus = billing.status.toProto()
            }
        }.build()
    }

    // Override for optimized batch loading
    override suspend fun resolveBatch(
        entityIds: List<String>,
        requestedFields: Set<String>
    ): Map<String, UserBillingExtension?> {
        val billings = billingRepository.findByUserIds(entityIds)
        return entityIds.associateWith { id ->
            billings[id]?.let { billing ->
                // Build proto as above
            }
        }
    }
}
```

#### Service Bootstrap

```kotlin
fun main() {
    val exposer = BillingExposerImpl(billingRepository)

    DataMeshExposer.builder()
        .serviceName("billing-service")
        .routerEndpoint("router.internal:9090")
        .healthPort(8080)
        .register(exposer)
        .build()
        .start()
}
```

### 7.2 Ruby SDK

#### Installation

```ruby
# Gemfile
gem 'datamesh-exposer', '~> 1.0'
```

#### Generated Module

```ruby
# Generated from user_billing_extension.proto
module DataMesh
  module Extensions
    module UserBilling
      class Exposer
        # Override this method
        def resolve(entity_id:, requested_fields:)
          raise NotImplementedError
        end

        # Optional override for batch optimization
        def resolve_batch(entity_ids:, requested_fields:)
          entity_ids.each_with_object({}) do |id, result|
            result[id] = resolve(entity_id: id, requested_fields: requested_fields)
          end
        end
      end
    end
  end
end
```

#### Implementation Example

```ruby
class BillingExposer < DataMesh::Extensions::UserBilling::Exposer
  def initialize(billing_repository)
    @billing_repository = billing_repository
  end

  def resolve(entity_id:, requested_fields:)
    billing = @billing_repository.find_by_user_id(entity_id)
    return nil unless billing

    UserBillingExtension.new(
      uuid: entity_id,
      billing_id: requested_fields.include?('billing_id') ? billing.billing_id : nil,
      payment_method: requested_fields.include?('payment_method') ? billing.payment_method : nil,
      billing_address: build_address(billing, requested_fields),
      payment_status: requested_fields.include?('payment_status') ? billing.status : nil
    )
  end

  def resolve_batch(entity_ids:, requested_fields:)
    billings = @billing_repository.find_by_user_ids(entity_ids)
    entity_ids.each_with_object({}) do |id, result|
      result[id] = billings[id] ? build_extension(billings[id], requested_fields) : nil
    end
  end

  private

  def build_address(billing, requested_fields)
    return nil unless requested_fields.include?('billing_address')
    # ... build address proto
  end
end
```

#### Service Bootstrap

```ruby
DataMesh::Exposer.configure do |config|
  config.service_name = 'billing-service'
  config.router_endpoint = 'router.internal:9090'
  config.health_port = 8080
end

exposer = BillingExposer.new(BillingRepository.new)
DataMesh::Exposer.register(exposer)
DataMesh::Exposer.start!
```

### 7.3 SDK Internal Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      DATA EXPOSER SDK                           │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐ │
│  │ Registration │  │ gRPC Server  │  │ Health HTTP Server    │ │
│  │ Client       │  │              │  │                       │ │
│  │              │  │ DataExposer  │  │ GET /health           │ │
│  │ Calls router │  │ Service impl │  │ Returns: status +     │ │
│  │ on startup   │  │              │  │          schema bytes │ │
│  └──────┬───────┘  └──────┬───────┘  └───────────┬───────────┘ │
│         │                 │                      │              │
│         │          ┌──────┴───────┐              │              │
│         │          │ Batch        │              │              │
│         │          │ Coordinator  │              │              │
│         │          │              │              │              │
│         │          │ Splits batch │              │              │
│         │          │ requests     │              │              │
│         │          └──────┬───────┘              │              │
│         │                 │                      │              │
│  ┌──────┴─────────────────┴──────────────────────┴───────────┐ │
│  │                    User Implementation                     │ │
│  │                    (resolve / resolveBatch)                │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 7.4 Data Exposer gRPC Protocol

```protobuf
// Internal protocol between Router and Data Exposers
syntax = "proto3";
package datamesh.exposer;

service DataExposerService {
  rpc Fetch(FetchRequest) returns (FetchResponse);
  rpc FetchBatch(FetchBatchRequest) returns (FetchBatchResponse);
}

message FetchRequest {
  string entity_id = 1;
  repeated string requested_fields = 2;
}

message FetchResponse {
  bytes data = 1;  // Serialized extension proto
  bool found = 2;
}

message FetchBatchRequest {
  repeated string entity_ids = 1;
  repeated string requested_fields = 2;
}

message FetchBatchResponse {
  map<string, bytes> data = 1;  // entity_id -> serialized proto
}
```

---

## 8. Push Data Handling (Kafka)

### 8.1 Topic Structure

Topics are organized per entity type:

```
datamesh.user.events      # Events extending User entity
datamesh.order.events     # Events extending Order entity
datamesh.product.events   # Events extending Product entity
```

### 8.2 Event Schema

```protobuf
// Standard envelope for all push events
message DataMeshEvent {
  // Event metadata
  string event_id = 1;
  string timestamp = 2;
  string source_service = 3;

  // Entity identification
  string entity_type = 4;
  string entity_id = 5;

  // Payload (serialized extension proto)
  bytes payload = 6;

  // Optional: event type for filtering
  string event_type = 7;
}
```

### 8.3 Router Kafka Consumer

```kotlin
class PushDataConsumer(
    private val redis: RedisClient,
    private val schemaRegistry: SchemaRegistry
) {
    fun consume(event: DataMeshEvent) {
        // Validate event has registered schema
        val schema = schemaRegistry.getServiceSchema(event.sourceService)
            ?: throw UnregisteredServiceException(event.sourceService)

        // Validate payload against schema
        val descriptor = schema.findMessageByEntityType(event.entityType)
        val message = DynamicMessage.parseFrom(descriptor, event.payload)

        // Get merge strategy for this entity type
        val mergeStrategy = schemaRegistry.getMergeStrategy(event.entityType)

        // Store in Redis
        storeEvent(event.entityType, event.entityId, message, mergeStrategy)
    }

    private fun storeEvent(
        entityType: String,
        entityId: String,
        message: DynamicMessage,
        strategy: MergeStrategy
    ) {
        val key = "push:$entityType:$entityId"

        when (strategy) {
            LAST_WINS -> redis.set(key, message.toByteArray())

            FIRST_WINS -> redis.setNx(key, message.toByteArray())

            MERGE_NON_NULL -> {
                val existing = redis.get(key)?.let {
                    DynamicMessage.parseFrom(message.descriptorForType, it)
                }
                val merged = mergeNonNull(existing, message)
                redis.set(key, merged.toByteArray())
            }
        }

        // Maintain event count limit
        enforceEventLimit(entityType, entityId)
    }
}
```

### 8.4 Push Data Redis Storage

```
# Current merged state per entity
push:{entity_type}:{entity_id}:data     -> bytes (merged proto)

# Event history (for debugging/replay)
push:{entity_type}:{entity_id}:history  -> list (recent event bytes)

# Metadata
push:{entity_type}:{entity_id}:updated  -> timestamp
push:{entity_type}:{entity_id}:source   -> service name
```

### 8.5 Push + Pull Data Merging

When a query requests fields from both push and pull sources:

```kotlin
fun assembleResponse(
    pullResponses: Map<String, DynamicMessage>,
    pushData: DynamicMessage?,
    requestedFields: Set<String>
): DynamicMessage {
    val builder = DynamicMessage.newBuilder(aggregatedDescriptor)

    // Add pull data
    for ((service, response) in pullResponses) {
        for (field in response.allFields.keys) {
            if (field.name in requestedFields) {
                builder.setField(field, response.getField(field))
            }
        }
    }

    // Overlay push data (push takes precedence for its fields)
    if (pushData != null) {
        for (field in pushData.allFields.keys) {
            if (field.name in requestedFields) {
                builder.setField(field, pushData.getField(field))
            }
        }
    }

    return builder.build()
}
```

---

## 9. Client SDK

### 9.1 Schema Synchronization

Clients use a CLI tool to pull the aggregated schema:

```bash
# Pull latest schema for all entities
$ datamesh schema pull --router router.example.com:9090 --output ./generated/

Fetching schema from router.example.com:9090...
Schema version: 1.3.0-a1b2c3d4

Generated:
  - ./generated/user.kt (32 fields)
  - ./generated/order.kt (18 fields)
  - ./generated/product.kt (24 fields)

# Pull specific version
$ datamesh schema pull --version 1.2.0 --output ./generated/

# Check for updates
$ datamesh schema check --router router.example.com:9090
Current: 1.2.0-x9y8z7w6
Latest:  1.3.0-a1b2c3d4
Status:  UPDATE AVAILABLE (minor version, backward compatible)
```

### 9.2 Kotlin Client

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.datamesh:client-sdk:1.0.0")
}
```

```kotlin
// Initialize client
val client = DataMeshClient.builder()
    .routerEndpoint("router.example.com:9090")
    .schemaVersion("1.3.0-a1b2c3d4")  // Pin to known version
    .build()

// Query with generated types
val user: User? = client.get<User>(
    id = "550e8400-e29b-41d4-a716-446655440000",
    fields = listOf(User::name, User::email, User::billingId)
)

// Batch query
val users: Map<String, User?> = client.getBatch<User>(
    ids = listOf("uuid-1", "uuid-2", "uuid-3"),
    fields = listOf(User::name, User::paymentStatus)
)

// Raw query (untyped)
val response: QueryResponse = client.execute("""
    (get :type User :id "uuid-here" :fields [name email])
""")
val json: String = response.jsonData
```

### 9.3 Ruby Client

```ruby
# Gemfile
gem 'datamesh-client', '~> 1.0'
```

```ruby
# Initialize client
client = DataMesh::Client.new(
  router_endpoint: 'router.example.com:9090',
  schema_version: '1.3.0-a1b2c3d4'
)

# Query with generated types
user = client.get(
  type: User,
  id: '550e8400-e29b-41d4-a716-446655440000',
  fields: [:name, :email, :billing_id]
)

# Batch query
users = client.get_batch(
  type: User,
  ids: ['uuid-1', 'uuid-2', 'uuid-3'],
  fields: [:name, :payment_status]
)

# Raw query
response = client.execute('(get :type User :id "uuid-here" :fields [name])')
json = response.json_data
```

### 9.4 REST/cURL Usage

For ad-hoc queries without SDK:

```bash
# Single entity query
curl -X POST https://router.example.com/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "(get :type User :id \"uuid-here\" :fields [name email billing_address])"
  }'

# Response
{
  "data": {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    "email": "john@example.com",
    "billing_address": {
      "street": "123 Main St",
      "city": "San Francisco",
      "country": "USA",
      "postal_code": "94102"
    }
  },
  "errors": null,
  "metadata": {
    "schema_version": "1.3.0-a1b2c3d4",
    "execution_time_ms": 45
  }
}
```

---

## 10. Observability

### 10.1 Metrics

The Router exposes the following metrics:

| Metric | Type | Labels | Description |
|--------|------|--------|-------------|
| `datamesh_query_duration_seconds` | Histogram | `entity_type` | End-to-end query latency |
| `datamesh_service_request_duration_seconds` | Histogram | `service`, `entity_type` | Per-service fetch latency |
| `datamesh_service_request_total` | Counter | `service`, `entity_type`, `status` | Request count by status |
| `datamesh_service_error_total` | Counter | `service`, `entity_type`, `error_code` | Errors by type |
| `datamesh_batch_size` | Histogram | `entity_type` | Batch query sizes |
| `datamesh_push_events_total` | Counter | `entity_type`, `source_service` | Kafka events processed |
| `datamesh_schema_version` | Gauge | `entity_type` | Current schema version (as number) |
| `datamesh_registered_services` | Gauge | `entity_type` | Count of registered services |

### 10.2 Distributed Tracing

The Router propagates standard trace headers:

- W3C Trace Context (`traceparent`, `tracestate`)
- B3 headers (for Zipkin compatibility)

Services using the Data Exposer SDK automatically:
- Extract incoming trace context
- Create child spans for resolve operations
- Propagate context to downstream calls

### 10.3 Logging

Structured logging format:

```json
{
  "timestamp": "2026-01-30T12:00:00.000Z",
  "level": "INFO",
  "service": "datamesh-router",
  "trace_id": "abc123",
  "span_id": "def456",
  "message": "Query executed",
  "query_hash": "sha256:...",
  "entity_type": "User",
  "entity_id": "uuid-here",
  "services_called": ["billing-service", "preferences-service"],
  "duration_ms": 45,
  "status": "success"
}
```

### 10.4 Schema Introspection Endpoint

**GET /schema** - Returns aggregated schema for exploration

```bash
curl https://router.example.com/schema?entity_type=User
```

**Response**:
```json
{
  "entity_type": "datamesh.entities.User",
  "version": "1.3.0-a1b2c3d4",
  "fields": [
    {
      "name": "uuid",
      "type": "string",
      "source": "base",
      "description": "Unique entity identifier"
    },
    {
      "name": "name",
      "type": "string",
      "source": "user-core-service",
      "description": "User's display name"
    },
    {
      "name": "billing_address",
      "type": "BillingAddress",
      "source": "billing-service",
      "description": "User's billing address",
      "fields": [
        {"name": "street", "type": "string"},
        {"name": "city", "type": "string"},
        {"name": "country", "type": "string"},
        {"name": "postal_code", "type": "string"}
      ]
    }
  ],
  "services": [
    {
      "name": "billing-service",
      "status": "healthy",
      "fields_provided": ["billing_id", "payment_method", "billing_address", "payment_status"]
    },
    {
      "name": "preferences-service",
      "status": "healthy",
      "fields_provided": ["dark_mode", "locale", "notification_preferences"]
    }
  ]
}
```

---

## 11. Error Handling

### 11.1 Response Structure

All query responses include both `data` and `errors`:

```json
{
  "data": {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "name": "John Doe",
    "email": "john@example.com",
    "billing_id": null
  },
  "errors": [
    {
      "entity_id": "550e8400-e29b-41d4-a716-446655440000",
      "field": "billing_id",
      "service": {
        "name": "billing-service",
        "team": "payments-team",
        "contact": "#payments-oncall"
      },
      "error": {
        "code": "SERVICE_UNAVAILABLE",
        "message": "billing-service is currently unavailable",
        "retriable": true
      }
    }
  ],
  "metadata": {
    "schema_version": "1.3.0-a1b2c3d4",
    "execution_time_ms": 45,
    "partial_success": true
  }
}
```

### 11.2 Error Codes

| Code | Description | Retriable |
|------|-------------|-----------|
| `SERVICE_UNAVAILABLE` | Target service is unhealthy or unreachable | Yes |
| `SERVICE_TIMEOUT` | Request to service timed out | Yes |
| `ENTITY_NOT_FOUND` | Entity does not exist in target service | No |
| `FIELD_NOT_FOUND` | Requested field not in schema | No |
| `INVALID_QUERY` | Query syntax error | No |
| `SCHEMA_MISMATCH` | Client schema version incompatible | No |
| `INTERNAL_ERROR` | Unexpected router error | Maybe |

### 11.3 Partial Success Behavior

When some services fail but others succeed:

1. Return all successfully fetched data in `data`
2. Set failed fields to `null` in `data`
3. Include error details in `errors` array
4. Set `metadata.partial_success = true`

Clients can decide whether to use partial data based on their requirements.

---

## 12. API Reference

### 12.1 Router gRPC Services

| Service | Method | Description |
|---------|--------|-------------|
| `QueryService` | `Execute` | Execute a LISP DSL query |
| `QueryService` | `ExecuteStream` | Stream batch query results |
| `RegistrationService` | `RegisterExtension` | Register a schema extension |
| `RegistrationService` | `DeregisterExtension` | Remove a schema extension |
| `RegistrationService` | `GetSchema` | Fetch aggregated schema |
| `IntrospectionService` | `GetEntityTypes` | List all entity types |
| `IntrospectionService` | `GetEntitySchema` | Get schema for entity type |
| `IntrospectionService` | `GetServices` | List registered services |

### 12.2 Router REST Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/query` | Execute a LISP DSL query |
| `GET` | `/schema` | Get aggregated schema (all or by entity) |
| `GET` | `/schema/{entity_type}` | Get schema for specific entity |
| `GET` | `/services` | List registered services |
| `GET` | `/health` | Router health check |
| `GET` | `/metrics` | Prometheus metrics |

### 12.3 Data Exposer SDK Health Endpoint

**GET /health**

```json
{
  "status": "healthy",
  "service_name": "billing-service",
  "schema": {
    "entity_type": "datamesh.entities.User",
    "file_descriptor_set_base64": "CgdiaWxsaW5n..."
  },
  "uptime_seconds": 3600,
  "version": "1.2.3"
}
```

---

## Appendix A: Example Protobuf Files

### Base Entity (Router)

```protobuf
// router/proto/datamesh/entities/user.proto
syntax = "proto3";
package datamesh.entities;

option java_package = "com.datamesh.entities";
option java_multiple_files = true;

message User {
  string uuid = 1;
  string created_at = 2;
  string updated_at = 3;
}
```

### Extension (Service)

```protobuf
// billing-service/proto/extensions/user_billing.proto
syntax = "proto3";
package billing.extensions;

import "datamesh/options.proto";

option (datamesh.extension) = {
  entity_type: "datamesh.entities.User"
  service_name: "billing-service"
};

message UserBillingExtension {
  string uuid = 1;

  string billing_id = 2;
  string payment_method = 3;
  BillingAddress billing_address = 4;
  PaymentStatus payment_status = 5;
}

message BillingAddress {
  string street = 1;
  string city = 2;
  string country = 3;
  string postal_code = 4;
}

enum PaymentStatus {
  PAYMENT_STATUS_UNSPECIFIED = 0;
  PAYMENT_STATUS_ACTIVE = 1;
  PAYMENT_STATUS_SUSPENDED = 2;
  PAYMENT_STATUS_CANCELLED = 3;
}
```

### DataMesh Options

```protobuf
// router/proto/datamesh/options.proto
syntax = "proto3";
package datamesh;

import "google/protobuf/descriptor.proto";

// File-level option for declaring which entity this file extends
message ExtensionOptions {
  string entity_type = 1;
  string service_name = 2;
}

extend google.protobuf.FileOptions {
  ExtensionOptions extension = 50000;
}

// Field-level option for declaring entity references
// Used when a message field references another entity type
extend google.protobuf.FieldOptions {
  string entity_ref = 50001;  // Value is the fully-qualified entity type
}
```

**Usage example:**

```protobuf
message UserOrdersExtension {
  string uuid = 1;

  // Single entity reference
  OrderRef primary_order = 2 [(datamesh.entity_ref) = "datamesh.entities.Order"];

  // List of entity references
  repeated OrderRef orders = 3 [(datamesh.entity_ref) = "datamesh.entities.Order"];

  // Non-reference field (just data)
  int32 total_order_count = 4;
}

// Reference message - MUST have 'uuid' as string field number 1
message OrderRef {
  string uuid = 1;          // REQUIRED: Router extracts this for entity resolution

  // Optional inline metadata (returned as-is, not resolved)
  string role = 2;          // e.g., "primary", "gift"
  string linked_at = 3;     // When this order was associated
}
```

**Validation:** Entity reference fields must be message types. The Router validates at registration:
- Field must be a message type (not string, int, etc.)
- Message must have a `uuid` field
- `uuid` must be type `string`
- `uuid` must be field number 1

---

## Appendix B: LISP DSL Grammar (Formal)

```
query       ::= '(' operation ')'
operation   ::= get | batch

get         ::= 'get' type-clause id-clause fields-clause?
batch       ::= 'batch' get+

type-clause   ::= ':type' IDENTIFIER
id-clause     ::= ':id' STRING
fields-clause ::= ':fields' field-list

field-list  ::= '[' field* ']'
field       ::= IDENTIFIER | nested-field
nested-field ::= '(' IDENTIFIER field-list ')'

IDENTIFIER  ::= [a-zA-Z_][a-zA-Z0-9_]*
STRING      ::= '"' [^"]* '"'
```

**Nested field semantics:**

The `nested-field` production serves two purposes depending on the field type:

| Field Type | Behavior | Example |
|------------|----------|---------|
| Embedded message | Select sub-fields of the message | `(billing_address [street city])` |
| Entity reference | Resolve referenced entities and select their fields | `(orders [status created_at])` |

The Router determines behavior by checking if the field has a `datamesh.entity_ref` option.

**Query examples by complexity:**

```lisp
; Simple: scalar fields only
(get :type User :id "u-1" :fields [name email])

; Embedded: nested message fields
(get :type User :id "u-1" :fields [name (billing_address [street city])])

; Reference: single-level entity reference
(get :type User :id "u-1" :fields [name (orders [status total])])

; Deep: multi-level entity references
(get :type User :id "u-1"
     :fields [name
              (orders [status
                       (items [quantity
                               (product [name price])])])])

; Batch: multiple root entities
(batch
  (get :type User :id "u-1" :fields [name (orders [status])])
  (get :type User :id "u-2" :fields [name (orders [status])]))
```

---

## Appendix C: Redis Key Reference

```
# Schema Registry
schema:{entity_type}:descriptor          bytes    Aggregated FileDescriptorSet
schema:{entity_type}:version             string   Current semantic version
schema:{entity_type}:hash                string   Content hash
schema:{entity_type}:field_map           hash     field_name -> service_name
schema:{entity_type}:versions            zset     version -> timestamp
schema:{entity_type}:v:{version}         bytes    Historical descriptor

# Entity References (for query planning)
schema:{entity_type}:refs                hash     field_name -> referenced_entity_type
schema:{entity_type}:ref_sources         hash     field_name -> service_name (which service provides the ref)
schema:{entity_type}:referenced_by       set      Entity types that reference this type

# Service Registry
services:{service}:endpoint              string   gRPC endpoint
services:{service}:health                string   Health check URL
services:{service}:status                string   healthy|unhealthy
services:{service}:entities              set      Entity types extended
services:{service}:descriptor            bytes    Service's FileDescriptorSet
services:{service}:last_health_check     string   Timestamp
services:{service}:team                  string   Owning team name
services:{service}:contact               string   Contact channel

# Push Data
push:{entity_type}:{entity_id}:data      bytes    Merged proto data
push:{entity_type}:{entity_id}:history   list     Recent events (capped)
push:{entity_type}:{entity_id}:updated   string   Last update timestamp
push:{entity_type}:{entity_id}:source    string   Source service

# Configuration
config:merge_strategy:{entity_type}      string   Merge strategy enum
config:max_events:{entity_type}          string   Max events to retain

# Query Limits
config:query:max_reference_depth         string   Max nesting levels (default: 5)
config:query:max_total_entities          string   Max entities per query (default: 1000)
config:query:max_batch_size              string   Max root entities in batch (default: 100)
```

---

*End of Technical Specification*
