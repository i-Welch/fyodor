Schema definition language: Protobuf
Schema storage: The extended schemas exist in the repo of the service which adding that schema. That schema is then aggregated by the router service when the extending service deploys. When the extending service tries to "push" its schema to router service during a deploy this is when the router service will detect issues around compatability and type collisions.
Schema Change Approval: This will be automated. Breaking changes will be flagged and trigger an automated warning.

Execution Strategy: The subqueries should be executed in parallel
Response assembly: buffered
Router State: Stateless using redis

Kafka Questions
  - Event Schema: What's the contract for push events? Same schema as pull responses?
  A: Yes these also require a protobuf schema to be deployed and registered with the router before events will start being consumed
  - Storage Backend: Where does the router store pushed data? In-memory, Redis, PostgreSQL, dedicated time-series DB?
  A: Redis
  - TTL/Retention: How long is pushed data retained? Per-entity-type configurable?
  A: Start with a cache of 100 events per entity type. Allow configuration of this number up to unlimited
  - Consistency Model: What consistency guarantees exist between pushed and pulled data?
  A: Data across different sub-schemas is unrelated and we shouldn't expect consistency
  - Event Ordering: How are out-of-order events handled (especially across partitions)?
  A: It shouldn't be expected
  - Backpressure: What happens when events arrive faster than they can be processed?
  A: We don't guarantee the latest event from the router. Just the latest which has been processed

Is merge strategy per-entity-type, per-field, or per-service?
A: per-entity-type

Query Language and interface:
  - Query Syntax: Custom DSL, GraphQL subset, SQL-like, or protocol-specific (gRPC reflection)?
  A: Custom DSL. It should be very bare bones to start out with but it should be built on LISP
  - Field Selection: How granular? Can users select nested fields? Arrays with filters?
  A: Yes to nested fields, no to arrays with filters
  - Pagination: Cursor-based, offset-based, or keyset?
  A: Not supported
  - Filtering/Sorting: Supported at router level, delegated to services, or both?
  A: Not supported
  - Batch Queries: Can users query multiple entities in one request?
  A: Yes
  - Subscriptions: Are real-time updates (websockets, SSE) in scope?
  A: No

Data Exposer Library (SDK)

  - Language Support: Which languages get first-class SDKs? (Go, Java, Python, Node.js, Rust?)
  A: Kotlin and Ruby get first class support
  - Registration Flow: How does a service register its schema extensions with the router?
  A: There should be an API for a service to register its schema when it deploys.
  - Health Checks: How does the router know if a data exposer is healthy?
  A: As part of registration the service should also include a healtcheck endpoint to allow the registration service to know when it is healthy. This healthcheck should also return the schema of the data being exposed so the registry service knows if an old version of the service is what is serving the requests
  - Authentication: How do services authenticate to the router (and vice versa)?
  A: Don't worry about auth right now
  - Batching: Can the router batch multiple entity requests to a single service call?
  A: Yes. The data exposer library should handle splitting and combining the batch requests so that services exposing data shouldn't have to think about handling batch requests if they don't want to. Alternatively the data exposer could also have a "batchRequest" function so that services can handle batch requests more efficiently if possible.

The SDK Interface Sketch looks pretty good. Keep in mind that what we will probably want to do is auto generate an interface based on the protobuf created in the repo and expect the end users to extend that interface.

Observability:
  - Distributed Tracing: How are traces propagated from client → router → services?
  A: We should allow existing tracing libraries to continue to work as they do. Don't think about this too deeply
  - Metrics: What SLIs/SLOs are tracked? Per-service latency? Cache hit rates?
  A: We should track per service latency and error rates / codes.
  - Error Reporting: How are partial failures surfaced to clients?
  A: We should include a {data} and an {error} object in results. All accesible data is in the data object and the error object should enough data for humans to debug. The error object and include some data about the underlying service which failed to make it easier to find the team responsible.
  - Schema Introspection: How do developers explore the aggregated schema?
  A: It should work similar to Graphql. The router should expose an endpoint for looking at the schema with all its fields and data types.
  - Debug Mode: Can queries include diagnostic information about which services were called?
  A: Not important for MVP



