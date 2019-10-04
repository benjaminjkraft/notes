# The Value of GraphQL to the Content Library

Most of the content library API is bespoke to content pages: it's the kind of thing where GraphQL doesn't provide much value because there's only one caller.  (There are still some tooling advantages, of course.)  But some bits and pieces of the API might be simplified by GraphQL.  On the whole, it's probably not worth the porting effort now, but I wanted to write this down so we have it in mind whenever we're ready.

## The case of content items

Often, content library pages need to send down some fields of a particular content item.  In some cases, this item is very interrelated to the topic, and so we may as well just pass it down as-is.  But in others, we really just want to pass down whatever fields the caller needs of a particular content item.  In GraphQL, this could be super simple: we would just return a node of that content type.  (In fact, with federation, this might allow us to avoid fetching that item at all!)

For example, currently we have
```python
_RELATED_VIDEO_SIG = {
    _f('kind'): sig.string,
    _f('contentId', source='content_id'): sig.string,
    _f('description'): sig.string,
    _f('duration'): sig.number,
    _f('imageUrl', source='image_url'): sig.string,
    _f('nodeUrl', source='node_url'): sig.string,
    _f('slug'): sig.string,
    _f('title'): sig.string,
    _f('youtubeId', source='youtube_id'): sig.string,
    _f('translatedYoutubeId',
        source='translated_youtube_id'): sig.string,
    _f('translatedYoutubeLang',
        source='translated_youtube_lang'): sig.string,
}

_CONTENT_ITEM_DATA = {
    _f('kind'): sig.string,
    _f('contentId'): sig.string,
    _f('slug'): sig.string,
    _f('title'): sig.string,
    _f('description'): sig.nullable(sig.string),
    _f('thumbnailUrl'): sig.nullable(sig.string),
    _f('nodeUrl'): sig.string,
    _f('progressKey'): sig.string,
    _f('status', optional=True): sig.string,
    _f('sponsored'): sig.boolean,
    _f('bigBingoLinkConversions', optional=True): [sig.string],
}

_CONTENT_LIST_ITEM_SIG = sig.legacy_one_of({
    sig.legacy_tag('kind'): sig.string,
    sig.case('Video'): sig.merge(
        _CONTENT_ITEM_DATA,
        {
            _f('duration'): sig.number,
            _f('youtubeId'): sig.string,
            _f('translatedYoutubeId'): sig.string,
            _f('translatedYoutubeLang'): sig.string,
            _f('downloadUrls'): sig.any_type,
        }),
    sig.case('Article'): _CONTENT_ITEM_DATA,
    sig.case('Exercise'): sig.merge(
        _CONTENT_ITEM_DATA,
        {
            _f('isSkillCheck'): sig.boolean,
            _f('masteryEnabled', optional=True): sig.boolean,
            _f('subjectMasteryEnabled', optional=True): sig.boolean,
            _f('relatedVideos'): [_RELATED_VIDEO_SIG],
            _f('expectedDoNCount'): sig.number,
        }),
    # (some cases omitted)
})
```
The majority of these fields are populated by simply taking the relevant attribute of the item in question.  Note how there's already some duplication to handle the needs of different callers (in this case different parts of the same API call): videos have two different representations (once as a content item with kind `Video`, and once as a related video).  This sig is itself used in several places, each of which surely use different fields; and there are several more sigs like it.  Just using ordinary content item sigs could simplify implementation while making it easier to add a new field.

This approach isn't without problems.  Life is a little bit tricky at the point where we don't literally want all the fields of the content item.  For example, `nodeUrl` is often populated with a context-sensitive URL (i.e. the URL of this node within this topic, if possible).  This would be a little trickier to handle in GraphQL.  In principle, it would be possible to make a `type ExerciseInContext` which would be keyed on both exercise key and node URL.  But it may be more work than it's worth, and this would be necessary if even a single field diverges.  Ultimately, this a semantic problem too: in some places in content library we just want to say "show this item" but in other places we want to show some data related to an item which is not, semantically, "just" the item itself.

## Supported modules

While we don't yet have plans to use this API for mobile, if we do in the future, GraphQL can help us support the fact that mobile may not want every module on the page.  In particular, our toplevel content page query will look something like
```graphql
query {
  contentPage(url) {
    modules {
      kind
      ... on TableOfContentsModule { ... }
      ... on FloatingSidebarModule { ... }
      ... on MappersModule { ... }
      ... on ConceptIntroModule { ... }
      # etc.
    }
  }
}
```

If the mobile client wants to render a similar page, but doesn't support every module yet, or needs different fields to show a slightly different UI treatment, this is easy to do: it can just omit the `... on FooModule` section for modules it doesn't support, and add and remove fields as needed.
