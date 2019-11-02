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
  a1: String @requires(fields: "b1")
  a2: String
  a3: String
}

# service B
extend type Query {
  b1: String @requires(fields: "a2")
}
```

To handle `{ a2 a3 }`, we need to resolve `a1`, then `b1`, then `a2`; and we need to resolve `a3`.  We can optionally merge `a3` into the query for `a1` or `a2`, or keep it separate; the most efficient option depends on a bunch of implementation details of the services.  If `a1` and `a2` are quick to compute, but `b1` and `a3` are slow to compute, then we want to pursue the two paths fully in parallel, so that fetching `a3` doesn't end up on the critical path to computing `a2`.  But if `a3` is cheaply computed from the same expensive-to-fetch data as `a1`, we want to bundle those two together; and conversely `a2`.  Short of streaming GraphQL responses, there's no universal solution; I propose that fetching `a3` separately is the least-bad option in most cases: combining the queries makes it impossible to optimize further in the case where `a3` is slow; but separating them allows the backend service the option to optimize.  Future updates could potentially leverage performance data from past queries to make a more educated guess.

To do so, we:

1. For each field in the query, add in any sibling fields (or descendants thereof) required to compute the queried fields -- i.e., the appropriate `@key` sibling of any extension field, as well as all fields `@requires`d by a queried fields.

2. Create a graph whose vertices are the fields in the query, and whose edges are dependencies between those fields.  Note that "fields" here refers to fields-of-query-nodes, not fields-of-types: a single field of a particular type may appear multiple times in a query.  Namely, there are edges from each extension field to its `@key` field(s); from each field to the sibling field(s) it `@requires`; and from each field to its child fields.  Note that in the case of a nested key or requires, there will be an edge to the base field and each descendant that is `@requires`d: for example if `a @requires(fields: "b { c { d } }")` we will have edges from `a` to `b`, `b.c`, and `b.c.d`.

3. Now, we want to merge the vertices representing fields that should be queried together.  To preserve correctness, we must avoid merging fields which would create cycles, i.e., those vertices which already have a path between them.  (Note that we can merge the vertices if there is a direct edge from one to the other, but no other such path.)    This is the part where we get to make choices; for the reasons described above we merge only those vertices which have precisely the same vertices as in-neighbors and out-neighbors (with the potential exception of direct edges from one to the other, as before).  This is necessarily a safe merger, as two such vertices cannot have a path (it would result in a cycle); and this process is well-defined and deterministic, as this neighbor-equivalence relation is transitive and symmetric.

4. We now have the query graph to be executed.  Each vertex of this graph is a subquery, and the edges of the graph determine which queries must block on which others.  (We have ensured there can be no cycles.)  Stitching the actual result data back together is nontrivial, but should work roughly as it does today.

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
In step 1 we will add the `currentUserIsSeller` field; in step 2 this field will have edges to all the others.  In step 3 we will merge the two fields from the users service, which results in the query plan one would expect: we fetch the permission, then, in parallel, fetch the fields from each service on which it depends.  Note that if we had also fetched `myUserProfile`, we would have done so in a separate request, in case this profile is slow to compute.

For user-specific permissions, consider:
```graphql
# users service
type User @key(fields: "id") {
  id: ID
  name: String @requires(fields: "isFriendsWithCurrentUser")
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
Here our graph looks like:
```
me ---> me.friends --> me.friends.isFriendsWithCurrentUser --> me.friends.name
                                                        `--> me.friends.phoneNumber
```
We can combine vertices to get:
```
me ---> me.friends, me.friends.isFriendsWithCurrentUser --> me.friends.name, me.friends.phoneNumber
```
which is to say again we do the obvious: we fetch the user's ID from `users`; then their friends, along with the bit that they are the user's friend (in this case, easy to know since that service has already checked that relationship); then those friends' names and phone numbers.  

In the case where `name` is publicly visible, we get a less ideal query plan. The graph
```
me ---> me.friends --> me.friends.isFriendsWithCurrentUser --> me.friends.phoneNumber
              `--> me.friends.name
```
sadly has no vertices we can combine, which is likely not ideal.  Potentially an `@provides` annotation on `isFriendsWithCurrentUser` could help us see the right query plan here, but the general case is tricky.
