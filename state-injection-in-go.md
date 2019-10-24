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

    logger.Info(ctx, "Got the stories")
    return retval
}

func (suite *storiesSuite) TestGetUserStories() {
    // populate test entities
    suite.DatastoreClient.Put(...)
    // set up fake service with known response
    serviceClient := testutil.FakeClient(...)

    stories := GetUserStories(
        context.Background(),
        suite.DatastoreClient,
        serviceClient,
        suite.Logger,
        100,
    )
    
    suite.Require().Equal(len(stories), 100)
    suite.Require().Equal(suite.Logger.LastLog().Text, "Got the stories")
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

    log.GetClient(ctx).Info("Got the stories")
    return retval
}

func (suite *storiesSuite) TestGetUserStories() {
    // populate test entities
    datastore.GetClient(ctx).Put(...)
    // set up fake service with known response
    serviceClient := testutil.FakeClient(...)
    ctx := serviceClient.AddToContext(suite.GetContext())

    stories := GetUserStories(ctx, 100)
    
    suite.Require().Equal(len(stories), 100)
    suite.Require().Equal(testutil.LastLog(ctx).Text, "Got the stories")
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

    log.Client.Info("Got the stories")
    return retval
}

func (suite *storiesSuite) TestGetUserStories() {
    // populate test entities
    datastore.Client.Put(...)
    // set up fake service with known response
    testutil.MockServiceClient(testutil.FakeClient(...))

    stories := GetUserStories(context.Background(), 100)
    
    suite.Require().Equal(len(stories), 100)
    suite.Require().Equal(testutil.LastLog().Text, "Got the stories")
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

    log.Info("Got the stories")
    return retval
}

// Note that with context-based dependency injection, things would look more
// like the test there; this is what they would look like for mocking/globals.
func (suite *storiesSuite) TestGetUserStories() {
    // populate test entities
    datastore.Put(...)
    // set up fake service with known response
    testutil.MockServiceClient(testutil.FakeClient(...))

    stories := GetUserStories(context.Background(), 100)
    
    suite.Require().Equal(len(stories), 100)
    suite.Require().Equal(testutil.LastLog().Text, "Got the stories")
}
```
