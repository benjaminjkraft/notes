# GraphQL Summit 2020 proposal

title: A Goliath Project: Khan Academy's GraphQL Migration

themes: server, scaling the graph

session (25m) / advanced

## Abstract (50 words)
Khan Academy is in the midst of a huge effort to break our monolith into services without disrupting the millions of learners and teachers who rely on us, and we're using GraphQL federation to make it happen.  This talk will discuss why we're using GraphQL, how we're doing the move, and what we've learned.

## Description (200 words)

Khan Academy's mission is to provide a free, world-class education to anyone, anywhere. Our website, used by millions of learners, is powered by an aging Python 2 monolith. We're incrementally migrating to a new service architecture, making heavy use of GraphQL federation. We're doing all of this to allow us to improve performance, better scale our team, and put us on a strong foundation to continue to improve education over the next 10 years and beyond.

Rewriting our backend is a huge project! Why did we consider such a big move? How did we set ourselves up for success, and avoid disrupting our users? Where did we go right, and where did we go wrong? Where did GraphQL and federation help, and where did they hurt? Which parts did we plan ahead, and which did we figure out on-the-fly? What were we thinking?! This talk will answer these questions, and more.

## Bio (70 words)

Khan Academy / Staff Software Engineer / SF, CA, USA / PDT 

Ben has been on the Infrastructure Team at Khan Academy for the last five years, where he tries to keep the engineers happy, the learners learning, and the loading-spinners brief.

## 2-3 minute video

Khan Academy is doing a big migration to services, and we're using GraphQL really everywhere, and federation in a big way.  So this talk is basically to give a large-scale worked example of how federation, and generally GraphQL works in real life in a large web backend with millions of users.  With 25 minutes I'll have to narrow to a few theme.  The most interesting things, I think, are the process we went through to draw the service boundaries (and how federation ended up playing into that), and how we're having the federation gateway do side-by-side testing, so as we migrate fields from one service to another, we actually have the gateway fetch from both and compare the results.  But I will also touch on how we're using federation -- what problems it has solved for us, what do we wish we had done differently, or wish federation helped more with.  And for fun we've also got some creative uses of GraphQL *everywhere* -- even internally for REST APIs, cron jobs, etc.  And of course it's always fun to be able to see how technology can make a real difference in the world -- and so talking about what Khan Academy is doing is great for that too.

## Anything else

For additional context on the project, and why it's interesting other than just GraphQL, check out my coworker's [blog post](https://engineering.khanacademy.org/posts/goliath.htm).

## Draft outline

(not submitted)

- Intro/our old system (3 min)
    - what it does/overview of our site: displaying content, managing content, tracking learner progress, individual and aggregate reporting
    - architecture: Python 2 monolith on App Engine (flash back to 10 years ago; what's changed since then — React, GraphQL, Go)
    - why move: Python 3 is a lot of work, interest in a faster and more structured language, interest in more team independence
    - why GraphQL: already in use for some client APIs, federation is a superpower
- A quick tour of our new system-in-progress (5 min)
    - (keep this quick, it's mostly just for context and people interested in the details can ask me after/read our blog posts)
    - GraphQL everywhere
    - Apollo federation to glue it together
    - 27 services (very brief examples)
    - Go, App Engine, monorepo
    - migrating incrementally for a smooth transition
- Dos and don'ts (15 min)
    - Drawing service boundaries in advance
        - building on prior work to define "components" within the monolith
        - sketch potential services; do static analysis to identify problems; discuss potential solutions and refine
        - come up with principles: clear data ownership, right size for services, API format, etc.
        - include the whole product team, not just engineering, in the process: service boundaries will affect them too
        - make the plan at the start to avoid painting yourself into a corner, but don't build things until you actually need them
    - GraphQL specifics
        - using federation
        - migrating with federation (use the monolith as a backend)
        - side-by-side testing
        - federation is really good, but hard for people to get used to thinking about (questions about what goes where)
        - schema design is hard, fixing bad schema design is harder
        - things we still don't understand: federating mutations
        - maybe some examples: e.g. federating content into progress, or users into things
    - Limit what we change, and build scaffolding for a smooth transition
        - obviously limited new feature work
        - anything built in the old system may seem like a waste, but when in service of porting it sure beats the alternative
        - refactor in Python first, not while we move
        - expose GraphQL from Python so we can decouple REST -> GraphQL from Python -> Go
        - REST gateway service to consolidate code to support old clients
- Conclusion (2 min)
    - graph of replacing Python with Go over time, and of GraphQL Gateway traffic
    - general team impressions
    - talk to me next GraphQL Summit for the thrilling conclusion!
