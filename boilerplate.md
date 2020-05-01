# Thoughts on boilerplate code

## what the problem is/isn't

- it's probably not, as much as we think, about too much to type
    - we write lots of things that get thrown away just for the benefit of others and our future selves' readability (commit messages, tests, etc.)
    - there are plenty of templating/snippeting tools
    - but why does it still feel like that?  the feeling is real
        - the typing requires just enough thought to be annoying
- copypasta leads to errors
    - e.g. datastore key-kind/struct mismatch: avoided by having one inferred
- readability and the "where's the code" problem

## why is it hard to reduce?

- the just enough thought to be annoying is just enough thought to be hard to automate
- the making one little change from the pattern problem
- the learning a new pattern problem -- a class that fills everything in for you doesn't "look like" the code it replaces so if you haven't seen it before, you have to go read it

## pros and cons of codegen

- pro: inspectable (but is it actually, when it's thousands of lines?)
- pro: can "fork" to extend (but actually you want to either do that, or not.)
