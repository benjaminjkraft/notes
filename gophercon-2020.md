# GopherCon 2020 proposal

title: A Goliath Project: Khan Academy's Go Migration
tutorial (45m) / all audiences
tags: go, web, services, python

## Elevator Pitch (300 chars)
Khan Academy is in the midst of a huge effort to migrate our 10-year-old Python 2 monolith to a more modern Go service architecture, without disrupting the millions of learners and teachers who rely on us. This talk will discuss why we're moving to Go, how we're doing it, and what we've learned.

## Description (for attendees)
Khan Academy's mission is to provide a free, world-class education to anyone, anywhere. Our website, used by millions of learners, is powered by a codebase a few months older than Go, which until recently was written in Python 2. We're migrating to a new Go service architecture, to improve performance, better scale our team, and put us on a strong foundation to continue to improve education over the next 10 years and beyond.

Rewriting our backend is a huge project! Why did we consider such a big move? How did we set ourselves up for success? Where did we go right, and where did we go wrong? Which parts did we plan ahead, and which did we figure out on-the-fly? What were we thinking?! This talk will answer these questions, and more.

## Bio

Ben has been on the Infrastructure Team at Khan Academy for the last five years, where he tries to keep the engineers happy, the learners learning, and the loading-spinners brief. His biggest complaint since switching to Go is that it's hard to get up to the terrifying metaprogramming shenanigans he loved to do in Python.

## Notes (for reviewers)

I'm one of the technical leads of this project at Khan. It's very much in progress and in our team's collective brains, so it's a great time for us to talk about it! I think the way we've planned everything centrally and tried to maintain consistency from the start is especially unusual/interesting.

## Additional information

This is a draft outline if I were to give the talk now; there's plenty for a talk here but by Gophercon I'll have another 6 months worth of things to say, so I may replace some of what's below with things we haven't yet done.

- Intro/our old system (5 mins)
    - What it does/overview of our site: displaying content, managing content, tracking learner progress, individual and aggregate reporting.
    - Architecture: Python 2 monolith on App Engine.  (Flash back to 10 years ago; what's changed since then — React, GraphQL, Go.)
    - Why move/why Go: Python 3 is a lot of work, interest in a faster and more structured language, interest in more team independence; Go fits our style of consistency and simplicity
- A quick tour of our new system-in-progress (10 min)
    - Go everywhere
    - 27 services (maybe talk through a few, and how they looked in the monolith vs. now)
    - still App Engine
    - GraphQL everywhere, Apollo federation to glue it together
    - Keeping a monorepo
- Key things we did (25 min)
    - Drawing service boundaries in advance
        - building on prior work to define "components" within the monolith
        - sketch potential services; do static analysis to identify problems; discuss potential solutions and refine
        - come up with principles: clear data ownership, right size for services, etc.
        - include the whole product team, not just engineering, in the process: it will affect them too
        - make the plan at the start to avoid painting yourself into a corner, but don't build things until you actually need them
    - Agree on practices, rather than having each team go its own way
        - it's hard to get these right early, especially not being Go experts
        - shared libraries let a small infrastructure team support everyone better (if time: an in-depth example, such as our web-serving entrypoint)
        - consistent processes and style let people work fluidly across teams
        - spread this via explicit broadcast communication, code review, and linting
    - Build scaffolding for a smooth transition
        - refactor in Python first, not while we move (more on this later)
        - expose GraphQL from Python so we can decouple REST -> GraphQL from Python -> Go
        - the monolith is a service too; use it as a federation backend, for example
        - side-by-side testing to compare new and old routes
    - Limit what we change at the same time
        - obviously limited new feature work
        - staying on app engine; don't move at-rest data
        - REST gateway service to consolidate code to support old clients
        - if time: explain one of the cases where we tried to do too much (e.g. restructuring auth cookies while porting them) as an explanation of how they go wrong
- Conclusion (5 min)
    - Graph of replacing Python with Go over time
    - General team impressions
    - Talk to me next GopherCon for the thrilling conclusion!
