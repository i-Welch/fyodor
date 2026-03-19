 Must Answer:

  1. Base entity definition: Is there always a "core" service, or can entities emerge from any service's extension?
  A: Base entities should have to be defined in a core service. This will be in the router service itself.
  2. Aggregated schema output format: JSON-only, or also binary proto for typed clients?
  A: I would like to allow proto as well for the clients using the library, but not ones using something like a on the wire editor. The ones using client libraries should have a snapshot "version" of the dynamicly generate proto schema created for them.
  3. Registration persistence: Where does the router store registered schemas? (Redis seems natural given other choices)
  A: Redis

  Should Answer:

  4. Namespace strategy for potential collisions: Reject? Prefix?
  A: Reject
  5. LISP DSL transport: gRPC service? REST? Both?
  A: Both
  6. Healthcheck response schema: What fields exactly?
  A: The full schema being exposed by the service. So in this case the full protobuf schema will be returned.