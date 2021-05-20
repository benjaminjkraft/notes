# Unmarshaling into interface types in Go.

It's bothering me more and more that I don't know of a good way to unmarshal JSON into a struct with fields of interface type in Go.

Specifically, suppose I want to unmarshal a type like the following:

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

## Non-approach 1: Animal.UnmarshalJSON

Obviously this isn't something the JSON library is going to be able to handle itself -- I need to write an `UnmarshalJSON`; fine.  The most natural thing I might want to do is to write:

```go
func (a *Animal) UnmarshalJSON(b []byte) error {
	var s struct {
        Species string `json:"species"`
	}
	err := json.Unmarshal(b, &s)
	if err != nil {
		return err
	}

	switch s.Species {
	case "Canis lupus":
        var dog Dog
        *a = &dog
        return json.Unmarshal(b, &dog)
	case "Felis catus":
        var cat Dog
        *a = &cat
        return json.Unmarshal(b, &cat)
	default:
		return fmt.Errorf("invalid species: %q", s.Species)
	}
}
```

But this doesn't work, because (pointers to) interfaces aren't allowed to have methods of their own.

## Approach 2: Use a different type

Here's an alternative representation of the data:

```go
// Dog, Cat, Animal as before

type AnimalWrapper struct {
    Species string `json:"species"
    Animal
}

type Family struct {
  Pet AnimalWrapper `json:"pet"`
  // many other fields
}
```

If we do things this way, the above approach works fine: we just put the `UnmarshalJSON` method on `AnimalWrapper`, and all is hunky-dory.  Except all is not hunky-dory, because we've now had to change our entire program's types just to satisfy the JSON library!  And this is definitely not the idiomatic way to do the types in Go: why have `Species` when we could just type-switch on the type of `Animal`?  We should be able to do better.

## Approach 3: Wrap Unmarshal

An easy way to get around this restriction is to not implement `UnmarshalJSON`, and instead, whenever we want to unmarshal into a family, we do that same logic:

```go
var f Family

var s struct {
    Pet struct {
        Species string `json:"species"`
    } `json:"pet"`
}
err := json.Unmarshal(b, &s)
if err != nil {
    return err
}

switch s.Pet.Species {
case "Canis lupus":
    f.Pet = &Dog{}
case "Felis catus":
    f.Pet = &Cat{}
default:
    return fmt.Errorf("invalid species: %q", s.Species)
}

return json.Unmarshal(b, &f)
```

This works, but it's super annoying: imagine if `Pet` were nested quite deep within `Family`; we'd have to traverse all of that structure when we call `Unmarshal`.  (In fact, if we have a potentially heterogeneous list of pets `[]Animal`, not only would we have to iterate over the JSON values to pre-fill the types, it does not work, because `json.Unmarshal` clears slices before unmarshaling into them.)  Again, we should be able to do better.

## Non-approach 4: Family.UnmarshalJSON (naively)

The often-recommended approach is to put our `UnmarshalJSON` a level up, on `Family`.  The naive way to do this is to just take the code from the previous approach, and put it in a method `Family.UnmarshalJSON`:

```go
func (f *Family) UnmarshalJSON(b []byte) error {
    var s struct {
        Pet struct {
            Species string `json:"species"`
        } `json:"pet"`
    }
    err := json.Unmarshal(b, &s)
    if err != nil {
        return err
    }

    switch s.Pet.Species {
    case "Canis lupus":
        f.Pet = &Dog{}
    case "Felis catus":
        f.Pet = &Cat{}
    default:
        return fmt.Errorf("invalid species: %q", s.Species)
    }

	return json.Unmarshal(b, f)
}
```

But this recurses infinitely!  The call to `json.Unmarshal(b, f)` calls back to `Family.UnmarshalJSON`, because that's how you unmarshal a `Family`, which calls back to `json.Unmarshal`, and so on.

## Approach 5: Family.UnmarshalJSON (fixed)

So, `Family.UnmarshalJSON` can't call `json.Unmarshal(f)`.  What can we do instead?  One option is to manually handle each field of the JSON: unmarshal into, say, a `map[string]json.RawMessage`, manually match those keys to fields of `Family` (via reflection on the JSON tags, or hardcoding the struct fields), and unmarshal the values into those fields one-by-one.  Alternately, we could define a struct with the same fields, but without the `UnmarshalJSON` method, call `Unmarshal` on that, and then copy each field back over to our actual `Family` (again manually or via reflection).

This seems to be the most common approach, but I find it very unsatisfying.  We have to do a bunch of extra work to handle the ordinary, non-interface fields of `Family`.  Plus, we have to repeat this on each type that refers to `Animal` (and for each field of such that uses it -- this can get fairly complex if you have, say, fields `Livestock []Animal; Pets map[Name]Animal`).

## Approach 6: Family.UnmarshalJSON (with a hack)

I spent a while looking for a way to get around the field-by-field copying, and eventually found one.  The trick is as follows:

```go
func (f *Family) UnmarshalJSON(b []byte) error {
    var s struct {
        Pet struct {
            Species string `json:"species"`
        } `json:"pet"`
    }
    err := json.Unmarshal(b, &s)
    if err != nil {
        return err
    }

    switch s.Pet.Species {
    case "Canis lupus":
        f.Pet = &Dog{}
    case "Felis catus":
        f.Pet = &Cat{}
    default:
        return fmt.Errorf("invalid species: %q", s.Pet.Species)
    }

    var wrappedFamily Family
    tmp := (*wrappedFamily)(f)
	return json.Unmarshal(b, tmp)
}
```

It's not much code, but it's a little strange.  The trick is, we want the type of `tmp` to look to the JSON library like `Family` (so the call to `json.Unmarshal` will fill it in properly), but we want it to *not* have an `UnmarshalJSON` method.  We can't remove the method as such, but we can make another type with the same underlying type (i.e. struct structure).  (There are a few other tricks to do this, such as embedding `Family` in a type with an additional field `UnmarshalJSON struct{}` that gets precedence over the method.)

I find this code a little weird.  But it does work, and it avoids listing all the other fields of `Family`.  It still has the other drawbacks of the previous approach: we have to repeat this on every type that uses `Animal`.  And it doesn't work as such for fields of type `[]Animal` or similar (for the same reason approach 3 doesn't), although that could be solved by unmarshaling those fields separately (with the same complexity as approach 5 in that case).

Thanks to [Emil](https://twitter.com/emilioemilianov) and [Steve](https://twitter.com/StevenACoffman) for simplifications to this approach.

## Onwards

A real alternative approach is provided by [Go proposal #5901](https://github.com/golang/go/issues/5901), assuming it implements support for interfaces.  But that proposal will arrive in Go 1.17 at the earliest, and looks like it may be delayed further.  Of course it would be possible to implement a third-party fork of `encoding/json` with such support today.  It might also be possible to pack most of the complexity of approach 6 (or even approach 5) into a library.

Other ideas are welcome.
