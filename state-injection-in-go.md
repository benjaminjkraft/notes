# Examples for ADR #229: State-injection in Go

These examples are intended to show off how the state injection will work; the details of the APIs aren't necessarily accurate.

## Traditional Dependency Injection

```go
func GetUserStories(
    ctx context.Context,
    dsClient datastore.Client, 
    serviceClient graphql.Client,
    logger log.Client,
    count int,
) []*Story {
    stories := make([]*Story, 0, count)

    err := dsClient.GetAll(ctx, &stories)
    if err != nil {
        logger.Error(ctx, "Error fetching stories: %v", err)
        return nil
    }

    // Filter stories from banned users
    retval := make([]*story, 0, count)
    for _, story := range stories {
        if !_isUserBanned(ctx, serviceClient, logger, story.userKaid) {
            storiesToReturn = append(storiesToReturn, story)
        }
    }
    return retval
}
```

## Client from context

```go
func GetUserStories(ctx context.Context, count int) []*Story {
    stories := make([]*Story, 0, count)

    err := datastore.GetClient(ctx).GetAll(&stories)
    if err != nil {
        log.GetClient(ctx).Error("Error fetching stories: %v", err)
        return nil
    }

    // Filter stories from banned users
    retval := make([]*story, 0, count)
    for _, story := range stories {
        if !_isUserBanned(ctx, story.userKaid) {
            storiesToReturn = append(storiesToReturn, story)
        }
    }
    return retval
}
```

## Global client

```go
func GetUserStories(ctx context.Context, count int) []*Story {
    stories := make([]*Story, 0, count)

    err := datastore.Client.GetAll(ctx, &stories)
    if err != nil {
        log.Client.Error(ctx, "Error fetching stories: %v", err)
        return nil
    }

    // Filter stories from banned users
    retval := make([]*story, 0, count)
    for _, story := range stories {
        if !_isUserBanned(ctx, story.userKaid) {
            storiesToReturn = append(storiesToReturn, story)
        }
    }
    return retval
}
```

## No client

```go
func GetUserStories(ctx context.Context, count int) []*Story {
    stories := make([]*Story, 0, count)

    err := datastore.GetAll(ctx, &stories)
    if err != nil {
        log.Error(ctx, "Error fetching stories: %v", err)
        return nil
    }

    // Filter stories from banned users
    retval := make([]*story, 0, count)
    for _, story := range stories {
        if !_isUserBanned(ctx, story.userKaid) {
            storiesToReturn = append(storiesToReturn, story)
        }
    }
    return retval
}
```
