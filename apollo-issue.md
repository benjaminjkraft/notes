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
