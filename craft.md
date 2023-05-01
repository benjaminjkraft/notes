# Disorganized Musings on Craft, Software Engineering, and AI

I've been thinking a lot about craft recently.

Some of this is from starting at Notion, where this sort of thing is in the culture. Some of this is from reading _The Way of the Cocktail_ which, I would argue, sees craft as a defining feature of Japanese bartending. Some of this is from thinking about AI and what it does and doesn't do.

I think of craft as the _consistent_ attention to getting things right, and the joy of doing so, even when it's mostly invisible and often doesn't matter. Often, craft is about the *process* of creation moreso than the result itself. Some examples:

- Wrapping lines nicely in commit messages
- Correctly handling edge cases that are unlikely to come up, or commenting about them if you can't (TODO: this is not a great example, something more specific probably gets the idea better)
- Warming the tea cup before brewing the tea
- Not tossing extra stones in a backcountry stream crossing to make it easier

TODO: examples that carry through will be a bit clearer

## Craft Makes You a Better Software Engineer

Software engineers love to talk about tradeoffs, and about knowing when to make compromises. This is entirely correct, and craft is not an excuse for overengineering. Craft is more orthogonal to how complete or complex a solution is: it's more about getting the details right whatever the level of the solution may be. In some sense, formatting a throwaway script neatly is just as much a waste of time as overengineering[^waste], but if we get in the habit of paying attention to these details, getting them right comes more naturally, which makes it easier to do well when it matters a little, which makes it more worth doing, which makes it easier to do well when it matters a lot.

## Craft and AI

The talk of the town is AI. One question that comes up is whether software engineers will, too, get automated out of our jobs sooner than we think. I don't intend to take a strong position on that right now, but I think that craft is part of the answer, in two ways.

The first thought is that craft is something that LLMs are not (yet) good at. LLMs are exactly interested in making their answers appear correct, and they often can't be trusted with the details. This doesn't prevent them from being useful! But it means that building things that require attention to craft is something that they can only help with, not replace. While I started by saying craft often doesn't matter, there are many cases where it does: code where careful thought and careful practice prevents hard-to-reproduce bugs or keeps us from painting ourselves into a corner, like complex concurrent code and typecheckers. Because craft requires a deep knowledge of one's tools, it's hard for even careful AI users to jump into; while help from an LLM will make it easier to learn how to use a library on the fly, it's hard to pick up a high level of craft in the same way. Combine that with the inevitable desire for the "hand-crafted"—and we'll be kept busy for a long time.

Of course, betting against LLMs picking up a new skill is a bad place to be. Perhaps it won't be long until almost everyone is out of a job! In a fully automated future where AI can do anything we do—and let's assume for the moment that we're not all turned into paperclips already—craft can still bring us joy. While I claim it's useful as a skill, the best way to hone craft is to get to the point where it's its own reward: where that little extra attention to detail is what makes the work feel good. Even if AI could have done the work just as well, that feeling remains.

[^waste]: This isn't true in general; overengineering is not just a waste of time now but a maintenance burden in the future.
