# Unmarshaling into interface types in Go.

It's bothering me more and more that I don't know of a good way to unmarshal JSON into a struct with fields of interface type in Go.

Specifically, suppose I have types like
```go
type Dog struct {
  Species string `json:"species"`
  Barks bool `json:"barks"`
}

type Cat struct {
  Species string `json:"species"`
  Meows bool `json:"meows"`
}

type Animal interface { /* methods shared by Cat and Dog */ }

type Family struct {
  Pet Animal `json:"pet"`
  // many other fields
}
```

Obviously this isn't something the JSON library is going to be able to handle itself -- I need to write `Family.UnmarshalJSON` that will unmarshal just the `Pets.Species` field and use that to decide which concrete implementation of `Animal` to use for `Pet`; fine.  But how do I actually write it?  As far as I've been able to find, the "standard":

1. Manually handle each field of the json, i.e., unmarshal into a `map[string]json.RawMessage`, manually match those keys to fields of `Family`, (perhaps with reflection on the JSON tag, or perhaps just hardcode the shape of the struct), and then for `Pet` do the logic described above.
2. Create a type with the same fields as `Family` but omitting `Pets`; unmarshal into that and copy the fields over.  Separately, unmarshal `Pets` with the logic described above, and copy that over.
3. Don't implement `UnmarshalJSON`; instead, before unmarshaling into a Family, unmarshal just `Pet.Species`, then set the family's `.Pet` to the right implementation.
4. Instead, have `Animal` be a struct containing a `Species` field, and then either an interface field with the `Cat` or `Dog`, or one field for each (of which at most one will be set); define `UnmarshalJSON` on `Animal`.

In the first two cases, we have to do a bunch of extra work to handle the *other* fields of `Family`.  And we have to do this in the type that _contains_ the `Pet`, which means we potentially have to do it several times in a fairly complex way if you have, say, two fields `Livestock []Animal; Pets map[Name]Animal`.  The third case is even worse: we have to do it everywhere we unmarshal; and it gets even messier, to the point of impossibility, if our `Animal` is nested somewhere deeper.  And the fourth case, while convenient for unmarshaling, means we have to change how the _entire rest of our program_ handles this type.  Some of these limitations are fine in certain contexts, but I keep running into cases where they get very annoying.

A few other approaches one might consider do not work:
4. Implement `UnmarshalJSON` on the `Animal` interface; this is impossible because interfaces can't implement their own methods.
5. Implement `UnmarshalJSON`, and like (3) have it fill in the right implementation before calling back to `Unmarshal` on itself.  This recurses infinitely, because it doesn't know not to call back to `UnmarshalJSON`.

After a while trying, I did eventually manage to make a version of (5) that does work, by using some Go type tricks to prevent `Unmarshal` from calling back to `UnmarshalJSON`: https://play.golang.org/p/isYzVuF4Rcw.  It is... not pretty.  It's a TODO to see if there is a way to package this up (perhaps via reflection).  Note that this still has the second problem discussed above, because it has to recurse on `Family`, not `Animal`.

Does anyone know of a better way?
