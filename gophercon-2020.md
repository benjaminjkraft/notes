# GopherCon 2020 proposal

title: A Goliath Project: Khan Academy's Go Migration

tutorial (45m) / all audiences

tags: go, web, services, python

## Elevator Pitch (300 chars)
Khan Academy is in the midst of a huge effort to migrate our aging Python 2 monolith to a modern Go service architecture, without disrupting the millions of learners and teachers who rely on us. This talk will discuss why we're moving to Go, how we're doing it, and what we've learned.

## Description (for attendees)
Khan Academy's mission is to provide a free, world-class education to anyone, anywhere. Our website, used by millions of learners, is powered by a codebase that dates to before Go's public release, and which until recently was written in Python 2. We're incrementally migrating to a new Go service architecture, to improve performance, better scale our team, and put us on a strong foundation to continue to improve education over the next 10 years and beyond.

Rewriting our backend is a huge project! Why did we consider such a big move? For whom would such a move be worth considering? How did we set ourselves up for success? How are we doing such a big migration incrementally and without affecting our users? Where did we go right, and where did we go wrong? Which parts did we plan ahead, and which did we figure out on-the-fly? What were we thinking?! This talk will answer these questions, and more.

## Bio

Ben has been on the Infrastructure Team at Khan Academy for the last five years, where he tries to keep the engineers happy, the learners learning, and the loading-spinners brief. His biggest complaint since switching to Go is that it's hard to get up to the terrifying metaprogramming shenanigans he loved to do in Python.

## Notes (for reviewers)

I'm one of the technical leads of this project at Khan. It's very much in progress and in our team's collective brains, so it's a great time for us to talk about it! I think the way we've planned things and tried to maintain consistency across teams from the start is the most unusual/interesting part of our experience, but many parts of our experience may be of interest to others considering any major language shift. (And to those who love Khan Academy!)

## Additional information

This is a draft outline if I were to give the talk now; there's plenty for a talk here but by Gophercon I'll have another 6 months worth of things to say, so I may replace some of what's below with things we haven't yet done.

- Intro/our old system (5 mins)
    - What it does/overview of our site: displaying content, managing content, tracking learner progress, individual and aggregate reporting.
    - Architecture: Python 2 monolith on App Engine.  (Flash back to 10 years ago; what's changed since then — React, GraphQL, Go.)
    - Why move/why Go: Python 3 is a lot of work, interest in a faster and more structured language, interest in more team independence; Go fits our style of consistency and simplicity
- A quick tour of our new system-in-progress (10 min)
    - (keep this quick, it's mostly just for context)
    - Go everywhere
    - 27 services (if time: maybe talk through a few, and how they looked in the monolith vs. now)
    - still App Engine (and other GCP services)
    - GraphQL everywhere, Apollo federation to glue it together (if time: a bit more on this; but really it's a talk of its own)
    - keeping a monorepo
    - migrating incrementally for a smooth transition
- Dos and don'ts (25 min)
    - Drawing service boundaries in advance
        - building on prior work to define "components" within the monolith
        - sketch potential services; do static analysis to identify problems; discuss potential solutions and refine
        - come up with principles: clear data ownership, right size for services, etc.
        - include the whole product team, not just engineering, in the process: it will affect them too
        - make the plan at the start to avoid painting yourself into a corner, but don't build things until you actually need them
    - Agree on practices, rather than having each team go its own way
        - shared libraries let a small infrastructure team support everyone better (likely with an in-depth example, such as our web-serving entrypoint)
        - consistent processes and style let people work fluidly across teams
        - spread this via explicit broadcast communication, code review, and linting
        - a cost: less room to experiment with alternatives that we might want to adopt (especially an issue if you don't already have go experience to draw on)
    - Limit what we change at the same time
        - (if I come out short on time or have more to add, this section probably gets condensed into some brief context for the next section)
        - obviously limited new feature work
        - staying on app engine and datastore; don't move at-rest data
        - REST gateway service to consolidate code to support old clients
        - explain one of the cases where we tried to do too much (e.g. restructuring auth cookies while porting them) as an explanation of how they go wrong
    - Build scaffolding for a smooth transition
        - anything built in the old system may seem like a waste, but when in service of porting it sure beats the alternative
        - refactor in Python first, not while we move (more on this later)
        - expose GraphQL from Python so we can decouple REST -> GraphQL from Python -> Go
        - the monolith is a service too; use it as a federation backend, for example
        - side-by-side testing to compare new and old routes
- Conclusion (5 min)
    - Graph of replacing Python with Go over time
    - General team impressions
    - Talk to me next GopherCon for the thrilling conclusion!

For additional context on the project, check out my coworker's [blog post](https://engineering.khanacademy.org/posts/goliath.htm).
