# batching in GraphQL

- you want to avoid having to have `user(id: ID!): User` and `users(ids: [ID!]!): Users` for everything
  - batching helps, but:
    - can't do get-multi (or rather need data-loader)
    - doesn't work if you're somewhere deep down in query (need to run all ancestors twice)
    - requires repeating query a bunch
    - non-standard
    - doesn't play nice with federation
  - aliasing helps, but makes your queries nasty
- want to be able to avoid having `users { names }` call names O(n) times
  - this can be solved in libraries, in theory, at the cost of a lot of complexity in execution engine
