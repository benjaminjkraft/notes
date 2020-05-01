# The Problem with Content Slugs

**tl;dr:** Don't use `slug` (a.k.a. `readable_id` or `name`) as an identifier for content!  Use `content_id` instead.  Read on for why, as well as for common exceptions.

## What is a slug?

A slug, sometimes called `readable_id` (videos) or `name` (exercises) in older code, is one of the bits in the URL of a content page, like [`algebra`](https://www.khanacademy.org/math/algebra), [`parts-of-speech-the-pronoun`](https://www.khanacademy.org/humanities/grammar/parts-of-speech-the-pronoun), or [`hagia-sophia-mosque`](https://www.khanacademy.org/humanities/art-islam/islamic-art-late-period/v/hagia-sophia-mosque).  Each content item has a slug, which is unique (within a given localized topic tree).  However, item slugs do sometimes change, such as when the content team does "slug-swaps" to put a new content item at an existing URL.

## What are slugs good for?

Slugs are nice because they're human readable, but still safe for use in a URL.  For example, the URL `https://www.khanacademy.org/math/algebra` is a lot more useful to a human than `https://www.khanacademy.org/x7a488390/x2f8bb11595b61c86`.  Of course, on our website we use "Math › Algebra I" or similar, but that doesn't work so well in a URL.

In code, what this means is: use a slug if you have a URL, and want to know to what content it refers.  Ideally, use the entire URL, not just a single slug: this will make sure the item is shown in the correct context.  (Many leaf nodes appear in multiple different topics, for example as a part of different curricula in the same language.)  For example, our core content routing system uses slugs to decide what content page to show when a content URL is loaded.

## What aren't slugs good for?

Anything else.  Don't use slugs to identify items of content.  For example:

- If you're writing an API call, and need to pass in a content item, refer to it by (kind and) ID, not slug.
- If you're returning some data to the client, and need to refer to a content item, refer to it by (kind and) ID.  If the client needs to know the URL of the item (such as to link to it), include the URL.  Don't bother with the slug.
- If you need to store a reference to a content item in the datastore, especially don't use slugs: they're especially bad for anything that needs to still work in a year or two.  Use (kind and) ID.
- If you have a devshell script and need to print out information about a content item, it's not so bad to use a slug, but you may as well print out the (kind and) ID as well as the title and/or full URL, so you have something even more human-readable.

If in doubt: don't use slugs.  If still in doubt, feel free to ask me (benkraft) or the Content Platform engineering team.

## What are slugs unfortunately necessary for?

Unfortunately, we have a lot of legacy code that makes the use of slugs necessary.  For example:

- If you need to refer to a content item in the test-db, you can use a slug.  (Using IDs is unfortunately impossible because they change every time the test-db is rebuilt.)  We have some [ideas](https://khanacademy.slack.com/archives/C0A0EP14K/p1558552628084700) for how to fix this, but we haven't implemented them yet.
- If changing existing code to use slugs would require rewriting a ton of code, it's ok to avoid it for now; but please try to limit new uses for slugs.
- If you're stuck with a datastore model that use slugs, and it's too big or scary to backfill, so be it.  But consider adding an ID field to new entities.



## Kudos to

The mobile team!  They've almost entirely removed slugs from the mobile client codebase.
