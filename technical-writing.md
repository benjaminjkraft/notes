# Notes on technical writing (and communication)

I've been thinking, generally, about helping engineers write better.  (And communicate better generally, but I think writing is singularly important for engineers.) 

In addition to my personal interest as an inveterate reader of documents, I think writing is a really valuable skill for engineers who want to level up. To me, writing is a big lever on the scope-and-influence components of engineering work: if you write well, people are more likely to read what you write and thus to be influenced by it. Because it’s a lever rather than the end goal, it doesn’t always make it into feedback: if you write decently but not excellently, no one will tell you to work on your writing, they’ll just encourage you to increase your scope. (For example, they might tell you to write more ADRs, even if the real problem is that your ADRs aren’t getting the discussion they need because they don’t pose their questions clearly enough.)

## Idea: style reviewers

I was chatting with a friend about this and I realized when we're reviewing code, we are generally reviewing both that it does what it should do, and that it's well-written stylistically. But with ADRs, we almost exclusively look at the former. I feel the stylistic quality of ADRs is just as important as that of code, if not more so: ADRs are the locus of discussion around a particular issue and an artifact for our future selve, and so a clear presentation of the issues and of the relationship of the decision to its alternatives is just as important as clean code.

One thought that came up was to have style reviewers for ADRs: that is, somebody who would read the ADR (perhaps before it's sent out to the rest of the world) with an eye more towards "this isn't clear" or "this could be organized better" rather than the content. It could be opt-in, with the idea that it's a way to learn and to improve your ADRs, or just a thing everyone does, like code review. Or, we could just try to normalize the idea that just as in a code review it's fine for any contributor to comment on style. (Or both.)

Implementation details are TBD, but we’d want to solve for:

- ensuring that, like code review, everyone is both getting and giving feedback and it’s not something where The Experts tell you what to do
- making it feel truly like a learning exercise, not like people dumping on your hard work
- choosing reviewers that have some context as to the technical content, so they’re not totally confused
- reminder of all the old stuff in [Code Reviews](https://khanacademy.atlassian.net/wiki/spaces/ENG/pages/129335720/Code+Reviews) and [Code reviews as relationship builders](https://bjk5.com/post/3994859683/code-reviews-as-relationship-builders-a-few-tips)

**Addendum 8/23:** I wonder if this should be more general than just
style reviewers. Sometimes ADRs need straight-up stylistic feedback, but
often I think they need help organizing thoughts, presenting the issues
more clearly, etc. And those are often interrelated, just as they are in
code review.

## Other ideas to be fleshed out:

- style guide for ADRs (perhaps based on common review comments)
- writing workshops
