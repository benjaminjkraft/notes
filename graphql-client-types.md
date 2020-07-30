# The GraphQL Client Types Problem

One of the major benefits of GraphQL is that each piece of client code can request exactly the data it needs.  This is very convenient in a lot of respects, but it makes it somewhat difficult to write types for the returned data.  This is a problem we've had for many years in our JavaScript, and now we're starting to have it in our Go, too.  I'm interested in exploring how we might do better.

## The problem

For example, in various places in our code we have queries like the following (using a fictional, simplified content schema):
```graphql
query getExerciseDetails {
    exerciseById(...) {
        id
        title
        url
        prerequisites {
            id
            title
            url
        }
        problems {
            id
            data
        }
    }
}

query getLessonSummary {
    lessonByUrl(...) {
        id
        title
        children {
            id
            url
            ... on Exercise {
                numProblems
            }
            ... on Video {
                duration
            }
        }
    }
}

query getExerciseProgress {
    user {
        completedExercises {
            id
            title
            url
            numProblems
        }
    }
}
```

How do we type the results of these queries?  The obvious way is something like (here using Flow syntax; the problems are similar in Go):
```js
type GetExerciseDetailsResult = {| exerciseByID: ExerciseDetails |};
type ExerciseDetails = {|
    id: string,
    title: string,
    url: string,
    prerequisites: Array<PrerequisiteExercise>,
    problems: Array<Problem>,
|};
type PrerequisiteExercise = {|
    id: string,
    title: string,
    url: string,
|};
type Problem = {|
    id: string,
    data: string,
|}

type GetLessonSummaryResult = {| lessonByUrl: Lesson |};
type Lesson = {|
    id: string,
    title: string,
    children: Array<ExerciseChild | VideoChild>,
|};
type ExerciseChild = {|
    id: string,
    url: string,
    numProblems: number,
|};
type VideoChild = {|
    id: string,
    url: string,
    duration: number,
|};

type GetExerciseProgressResult = {| user: User |};
type User = {| completedExercises: Array<CompletedExercise> |};
type CompletedExercise = {|
    id: string,
    title: string,
    url: string,
    numProblems: number,
|};
```

That's a lot of types: we need four exercise types in three queries, because each place where exercises appear they have different fields!  It gets even messier if we want to write a utility function that can take in several of them:

```js
function linkToExercise(exercise: LinkableExercise): React.Node { ... }
type LinkableExercise = {
    title: string,
    url: string,
    ...
};
```

This is even harder in Go; we'd have to write an interface.  And it's worse if your helper wants to accept and return the same type:

```js
function shortExercises<E: {|numProblems: number|}>(exercises: $ReadOnlyArray<E>): $ReadOnlyArray<E> { ... }
```

That type is ugly enough in Flow, and impossible in Go.

## Solutions we've tried, none of them great

One option is: just live with it.  We can do codegen to make those types less annoying to write, and just deal with the awkwardness in helper functions.  In many cases, the friction is fundamental: our `linkToExercise` function can't take in an `ExerciseChild` without changing the data we request!  So we just live with that.

Another option is to try to make the fields we request less bespoke.  For example, we could say we always request `id`, `title`, and `url`, at least, even if we don't need them.  Then we at least only have 3 types.  This of course comes with some efficiency cost, but hopefully those fields are cheap to render.  It also arguably makes our code harder to change: what happens when we want to add a field to the shared type, or worse, remove one?

Another option is a catch-all type that's imperfect for all the uses:
```
type Exercise = {|
    id?: string,
    title?: string,
    url?: string,
    prerequisites?: Array<Exercise>,
    problems?: Array<Problem>,
    numProblems?: number,
|}
```
This is usable in all the places we want it to be, but never great: it means we have to do all sort of spurious null-checks.  Furthermore, we won't know that we can't pass an `ExerciseChild` to `linkToExercise` (it doesn't have a `title`); instead we'll just pass it, and get, presumably, an ugly link.  This seems to throw away the benefit of types.

A variant of the previous option is to define such types for all of our schema automatically.  The problems remain.
