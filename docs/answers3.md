  1. System Architecture - Components, interactions, deployment
  A: If the content changes it should bump the hash automatically. If there are only field additions it should be a minor version bump but breaking changes should be a major version bump. This should be able to be automatically detected by protobuf librarires
  2. Proto Schema Management - Base definitions, extensions, aggregation algorithm
  Typed clients will be regenerated manually. We can create tools to automatically check for updates via a cli but that should be done by a developer who needs to make a repo change.
  3. Registration Protocol - API contract, validation rules, persistence
  Mark unhealthy with warn is good enough
  4. Query System - LISP DSL grammar, parsing, execution planning
  for this I'm thinking we want to enforce types at the top level of queries always so like
  ```
  (get :type User :id "uuid" :fields [name])
  ```