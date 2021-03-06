# Issue

Currently, the federation documentation says

> Note that you can only require fields that live on the original type definition, not on type extensions defined in other services.

This issue proposes to remove this limitation.

If the maintainers are open to this feature, we (@Khan) are interested in implementing it starting immediately.

**Motivation:**

Our goal, in implementing this, is to allow our services to coordinate more efficiently while separating concerns appropriately.

More specifically, we have [written](https://spectrum.chat/apollo/apollo-federation/federation-and-permissions-checking~feab0eaf-f6c0-4c8a-99cd-1aba78f4690b) about how we are planning to use federation for permissions checking.  To allow permissions to be owned by a service other than that which owns a type, we need to be able to `@requires` fields from such a service.  This is a very common need for us.  Specifically, we have a `User` type owned by a `users` service, which handles core data about a user on our site.  Some of its fields may be visible to certain administrators -- a status which is owned by a separate `permissions` service.  Other fields may be owned by a `teachers` service which owns the data mapping which teachers teach which students, such that a teacher can see their students' progress.  In both cases, those fields may be defined on the `users` service, or on another service which extends `User`, for example the service which owns progress data (i.e. which content you've completed).

Another common use case for us, which would be enabled by this feature, is the use of `@requires` on toplevel query objects, which is currently impossible since nobody defines the base `Query` type.  Again, for us a use case is permissions: maybe we have a toplevel field `actorIsAdmin: Boolean` and we want to `@requires` this for other fields that are only accessible to admins.  We can imagine many other use cases.

We're proposing to implement this in Federation because we don't see any good workarounds (and none were suggested in our [Spectrum thread](https://spectrum.chat/apollo/apollo-federation/using-requires-for-a-field-not-defined-in-the-owner-service~290199d3-2877-4a61-af6a-c0ca022cac9c)).

**Implementation details:**

As far as we can tell, no changes to the federation spec, nor to backend servers, are needed: the changes are only in validation, documentation, and query planning.  (Some backend servers may need additional functionality to support an `@requires` on `Query`, but only the server using the `@requires` needs to do so.  I haven't looked into whether this is the case of Apollo; if this is a blocker for this issue to be accepted we can look into it but we don't have a desire to do so since we use Go for our backends.)

The changes to validation and documentation are simple: we need only prohibit circular `@requires` directives, that is, those which induce a cycle in the graph of fields which require other fields.  Unfortunately, this is a global validation, rather than being type-local, because of nested `@requires`, but the rule is pretty simple.

The changes to the query planner are somewhat more complex, which is presumably why this was not implemented from day one.  In particular, the query planner can no longer assume that to fetch some set of fields of a type it is sufficient to fetch some set of fields from the service which owns it, then, in parallel, fetch some set of fields from services which extend it, potentially including in the query some fields from the owning service.

Instead, the query planner will need to create the directed graph of `@requires`'d fields (which must be acyclic due to the validation constraint), traverse it assigning the fields to services, and then merge those fields which are owned by the same service and which have no dependency on each other, even transitively.  (Later today or tomorrow, I'll write a precise description of this algorithm.)  This will significantly increase the complexity of the query planner.

A note on what I've been calling "nested `@requires`": this iswhen a field `f: U` on type `T` to do `@requires(fields: "f { g }")` where `g` is a field of `U`.  This is not explicitly documented, but it makes sense by analogy with `@key`.  As far as I can tell, the spec allows it, and Apollo validates and executes the query, so I'm assuming it is supported.

**Prior art:**

I've written about our desire in [Spectrum](https://spectrum.chat/apollo/apollo-federation/using-requires-for-a-field-not-defined-in-the-owner-service~290199d3-2877-4a61-af6a-c0ca022cac9c).  I suspect a review of past threads might uncover additional use cases and users.

This would make moot my prior issue (#3399) about a related schema that Apollo validates but cannot query.

# Detailed algorithm

To construct the query plan, we break the query into groups of field-resolution operations which can be executed as a single query to a single backend service.  There is no universally best algorithm to do so; my hope is that by providing an algorithm which does the same thing the query planner used to do in already-supported cases, does the best thing in certain cases I expect to be common, we don't have to worry about writing the perfect query planner.  For example, consider the following schema

```graphql
# service A
extend type Query {
  a1: String
  a2: String @requires(fields: "b1")
  a3: String
}

# service B
extend type Query {
  b1: String @requires(fields: "a1")
}
```

To handle `{ a2 a3 }`, we need to resolve `a1`, then `b1`, then `a2`; and we need to resolve `a3`.  We can optionally merge `a3` into the query for `a1` or `a2`, or keep it separate; the most efficient option depends on a bunch of implementation details of the services.  If `a1` and `a2` are quick to compute, but `b1` and `a3` are slow to compute, then we want to pursue the two paths fully in parallel, so that fetching `a3` doesn't end up on the critical path to computing `a2`.  But if `a3` is cheaply computed from the same expensive-to-fetch data as `a1`, we want to bundle those two together; and conversely `a2`.  Short of streaming GraphQL responses, there's no universal solution.

I propose to follow the approach that most closely matches the existing behavior: namely, we fetch `a3` along with `a1`.  For the toplevel `Query`, it's sort of arbitrary, but for other types, this supposes that fields from the owning service with no `@requires` are typically cheap to add onto a fetch of the `@key` field, which is plausibly true.  It feels a little asymmetric, but it seems like the most practical approach.  (Fetching `a3` separately might be better in that it in principle allows the backend to do optimization that it fundamentally can't if we merge any queries, but it likely adds the most overhead to a naive backend, since it adds a bunch of extra queries in common cases.)  Future updates could potentially leverage performance data from past queries to make a more educated guess, or allow backends to provide further hints to the query planner.

To do so, we:

1. For each field in the query, add in any sibling fields (or descendants thereof) required to compute the queried fields -- i.e., the appropriate `@key` sibling of any extension field, as well as all fields `@requires`d by a queried fields.

2. Create a graph whose vertices are the fields in the query, and whose edges are dependencies between those fields.  Note that "fields" here refers to fields-of-query-nodes, not fields-of-types: a single field of a particular type may appear multiple times in a query.  Namely, there are edges from each extension field to its `@key` field(s); from each field to the sibling field(s) it `@requires`; and from each field to its child fields.  Note that in the case of a nested key or requires, there will be an edge to the base field and each descendant that is `@requires`d: for example if `a @requires(fields: "b { c { d } }")` we will have edges from `a` to `b`, `b.c`, and `b.c.d`.

3. Now, we want to merge the vertices representing fields that should be queried together.  To preserve correctness, we must avoid merging fields which would create cycles, i.e., those vertices which already have a path between them.  (Note that we can merge the vertices if there is a direct edge from one to the other, but no other such path.)    This is the part where we get to make choices; for the reasons described above we merge those vertices which have the same dependencies (i.e. vertices with the same in-edges), with the exception of direct edges from one to the other which we also allow.  This is necessarily a safe merger, as two such vertices cannot have a path (it would result in a cycle via their common dependency); and this process is well-defined and deterministic, as this in-neighbor-equivalence relation is transitive and symmetric.

4. We now have the query graph to be executed.  Each vertex of this graph is a subquery, and the edges of the graph determine which queries must block on which others.  (We have ensured there can be no cycles.)  Stitching the actual result data back together is nontrivial, but should work roughly as it does today.

This description omits handling of interfaces, but the approach Apollo currently uses should work: namely, it follows the entire subquery affected by the interface through each type that implements the interface, separately.  Similarly, slight adjustments may be necessary to take advantage of `@provides`, but these should be easy to incorporate.

**Examples:**

In particular, this handles well the common permissions-checking cases I describe above.  First, for global permissions, consider the following schema and query:
```graphql
# permissions service
extend type Query {
  currentUserIsSeller: Boolean
}

# inventory service
extend type Query {
  myInventory: ... @requires(fields: "currentUserIsSeller")
}

# users service
extend type Query {
  myUserProfile: ...
  mySellerProfile: ... @requires(fields: "currentUserIsSeller")
  myCustomers: ... @requires(fields: "currentUserIsSeller")
}

query {
  myInventory
  mySellerProfile
  myCustomers
}
```
In step 1 we will add the `currentUserIsSeller` field; in step 2 this field will have edges to all the others.  In step 3 we will merge the two fields from the users service, which results in the query plan one would expect: we fetch the permission, then, in parallel, fetch the fields from each service on which it depends.  Note that if we had also fetched `myUserProfile`, we would have done so in a separate request, since it doesn't depend on the `@requires`.

For user-specific permissions, consider:
```graphql
# users service
type User @key(fields: "id") {
  id: ID
  name: String
  phoneNumber: String @requires(fields: "isFriendsWithCurrentUser")
}

extend type Query {
  me: User
}

# relationships service
extend type User {
  friends: [User]
  isFriendsWithCurrentUser: Boolean
}

query {
  me {
    friends {
      name
      phoneNumber
    }
  }
}
```
Here our graph looks like (brackets denote the services):
```
me [u] --> me.friends [r] --> me.friends.isFriendsWithCurrentUser [r] --> me.friends.phoneNumber [u]
                  `--> me.friends.name [u]
```
We can combine vertices to get:
```
me [u] --> me.friends, me.friends.isFriendsWithCurrentUser [r] --> me.friends.name, me.friends.phoneNumber [u]
```
which is to say: we fetch the user's ID from `users`; then their friends, along with the bits, for each user, that they are the user's friend (in this case, easy to know since that service has already checked that relationship); then those friends' names and phone numbers.  Note that this is a case where being more conservative about merging (leaving `a3` separate, in the above example) results in a likely-worse query plan; it would permit no merging at all.

This also matches current behavior.  Consider the query:
```graphql
# users service
type User @key(fields: "id") {
  id: ID
  name: String
}

extend type Query {
  me: User
}

# reviews service
extend type User @key(fields: "id") {
  reviews: [String]
}

query {
  me {
    name
    reviews
  }
}
```

We would fetch `me` and `me.name` in a single query, then `me.reviews`.  In the case where `name` is expensive, this is suboptimal, but the safest assumption is likely that given `me.id` it's cheap.

We still differ in some cases.  Consider the query:
```graphql
# users service
type User @key(fields: "id") {
  id: ID
  name: String
}

extend type Query {
  me: User
  user(id: ID!): User
}

# reviews service
extend type User @key(fields: "id") {
  reviews: [String]
}

query {
  me {
    reviews
  }
  user(id: "...") {
    reviews
  }
}
```

Apollo fetches `me` and `user` in a single query, and then sends out separate queries to the reviews service for each of their reviews, because it never merges across branches.  We would merge, because our graph looks like
```
  me [u] --> me.reviews [r]
user [u] --> user.reviews[r]
```
We first merge the two toplevel queries, then this allows us to merge their dependencies.
```
me, user [u] --> me.reviews, user.reviews [r]
```
In this case, our behavior is clearly better, but in other cases -- such as if we had requested some other field that depends on `reviews` only for `me` -- it might not be.  It seems to me like the new behavior is on the whole better, but if preserving current behavior is desirable, we could add an additional rule that we only merge vertices if they have the same path in the query except as to the final field -- so we can merge `[query].me` with `[query].user` but not `me.reviews` with `user.reviews`.  Such a rule might also make the implementation simpler.

# Series-parallel problems

I had originally hoped that -- perhaps by using the restriction I mention at the end of my prior comment, of never merging non-siblings -- I could avoid having to change the `QueryPlan` data structure (and thus its execution, serialization, etc.; along with any client code that depends on it -- while it's not documented it is exported from `apollo-gateway`).  Sadly, the data structure can only represent [series-parallel graphs](https://en.wikipedia.org/wiki/Series-parallel_graph)\* of fetches.  If the `@requires` on a single field can form an arbitrary DAG, then it's easy to construct a case where the most efficient query plan requires a fetch-dependency graph matching that DAG.  But we cannot represent an arbitrary DAG as series-parallel.

For example, consider the following overall schema (supposing each field is resolved by a different service):
```graphql
type T {
  a: String @requires(fields: "b c")
  b: String @requires(fields: "d e")
  c: String @requires(fields: "e")
  d: String @requires(fields: "f")
  e: String @requires(fields: "f")
  f: String
}
```
The most efficient way to fetch `a` is not series-parallel: it looks like
```
    a
   / \
  b   c
 / \ /
d   e
 \ /
  f
```
where we fetch each field once all its children are completed.  This query plan cannot currently be represented.

I see three ways to go forward:

1. Modify the `QueryPlan` data structure to represent arbitrary graphs.  I don't think there's much that's intrinsically hard about this -- the promises that are used to execute it can represent such a graph easily -- but it will require a lot more code changes.  If the structure of `QueryPlan` is considered a public API, this would break compatibility.  (At least, I presume it might require changes to Graph Manager or other Apollo-controlled code outside apollo-server.)
2. Compute an imperfect query plan which can be represented.  For example, in the above case, we could introduce an artificial dependency of `e` on `d`.  This isn't a huge problem: while it's inefficient I expect queries that actually take advantage of this behavior would be rare.  But it's pretty ugly: we will have to do extra work just to come up with a worse query plan.
3. Add further restrictions to what `@requires` are supported that still handle the common use cases, but avoid this problem.  The one that comes to mind is to say you can't `@requires` a field which itself has an `@requires` -- this means we can simplify the query planning to first fetch the `@key` (if needed), then fetch any `@requires`d fields, then fetch the desired fields.  This is far more restrictive than actually necessary, but it will still handle the cases I describe above, and it can always be extended later.

I'm currently thinking to go ahead with option 3.

\*There are some technicalities here: really I mean graphs which, when redundant edges are removed, are not series-parallel; but this doesn't affect the overall result.
