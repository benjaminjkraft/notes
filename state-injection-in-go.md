# Examples for ADR #229: State-injection in Go

These examples are intended to show off how the state injection will work; the
details of the APIs aren't necessarily accurate.

## Traditional Dependency Injection

```go
func (r *queryResolver) UserStories(ctx context.Context, count int) ([]*graphql.Story, error) {
    stories := GetUserStories(
        ctx, r.DatastoreClient, r.ServiceClient, r.Logger, r.Timer, count)
    return serializeStoriesToGraphQL(stories), nil
}

func GetUserStories(
    ctx context.Context,
    dsClient datastore.Client,
    serviceClient graphql.Client,
    logger log.Client,
    timer time.Source,
    count int,
) []*Story {
    stories := make([]*Story, 0, count)

    oneYearAgo := timer.Now().Sub(time.Year)
    err := dsClient.GetAll(ctx, "after=", oneYearAgo, &stories)
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
        suite.Timer,
        100,
    )

    suite.Require().Equal(len(stories), 100)
    suite.Require().Equal(suite.Logger.LastLog().Text, "Got the stories")
}
```

## Client from context

```go
func (r *queryResolver) UserStories(ctx context.Context, count int) ([]*graphql.Story, error) {
    stories := GetUserStories(ctx, count)
    return serializeStoriesToGraphQL(stories), nil
}

func GetUserStories(ctx context.Context, count int) []*Story {
    stories := make([]*Story, 0, count)

    oneYearAgo := lib.Now(ctx).Sub(time.Year)
    err := datastore.GetClient(ctx).GetAll("after=", oneYearAgo, &stories)
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
    ctx := testutil.WithDatastoreEmulator(context.Background())
    // populate test entities
    datastore.GetClient(ctx).Put(...)
    // set up fake service with known response
    serviceClient := testutil.FakeClient(...)
    ctx = serviceClient.AddToContext(ctx)

    stories := GetUserStories(ctx, 100)

    suite.Require().Equal(len(stories), 100)
    suite.Require().Equal(testutil.LastLog(ctx).Text, "Got the stories")
}
```

## Client from khantext

```go
type myContext interface {
    khantext.Base // includes context.Context, perhaps request data
    khantext.DB
    khantext.Service // or a specific service
    khantext.Time // or include this in Base
    khantext.Log // or include this in Base
}

func (r *queryResolver) UserStories(ctx context.Context, count int) ([]*graphql.Story, error) {
    // extract request data from Go context, and dependencies from query
    // resolver, to create khantext:
    ktx := dependencies.CreateKhantext(r, ctx)
    // or: ktx := r.CreateKhantext(ctx)
    stories := GetUserStories(ktx, count)
    return serializeStoriesToGraphQL(stories), nil
}

func GetUserStories(ctx myContext, count int) []*Story {
    stories := make([]*Story, 0, count)

    oneYearAgo := ctx.Now().Sub(time.Year)
    err := ctx.DBClient().GetAll("after=", oneYearAgo, &stories)
    if err != nil {
        ctx.Logger().Error("Error fetching stories: %v", err)
        return nil
    }

    // Filter stories from banned users
    retval := make([]*story, 0, count)
    for _, story := range stories {
        if !_isUserBanned(ctx, story.userKaid) {
            storiesToReturn = append(storiesToReturn, story)
        }
    }

    ctx.Logger().Info("Got the stories")
    return retval
}

func (suite *storiesSuite) TestGetUserStories() {
    ctx := testutil.AddDB(testutil.BaseTestContext())

    // populate test entities
    ctx.DBClient().Put(...)
    // set up fake service with known response
    serviceClient := testutil.FakeClient(...)
    ctx = serviceClient.AddToOptions(ctx)

    stories := GetUserStories(ctx, 100)

    suite.Require().Equal(len(stories), 100)
    suite.Require().Equal(ctx.TestLogs.Text, "Got the stories")
}
```

## Global client

```go
func (r *queryResolver) UserStories(ctx context.Context, count int) ([]*graphql.Story, error) {
    stories := GetUserStories(ctx, count)
    return serializeStoriesToGraphQL(stories), nil
}

func GetUserStories(ctx context.Context, count int) []*Story {
    stories := make([]*Story, 0, count)

    oneYearAgo := lib.Now().Sub(time.Year)
    err := datastore.Client.GetAll(ctx, "after=", oneYearAgo, &stories)
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
func (r *queryResolver) UserStories(ctx context.Context, count int) ([]*graphql.Story, error) {
    stories := GetUserStories(ctx, count)
    return serializeStoriesToGraphQL(stories), nil
}

func GetUserStories(ctx context.Context, count int) []*Story {
    stories := make([]*Story, 0, count)

    oneYearAgo := time.Now().Sub(time.Year)
    err := datastore.Client.GetAll(ctx, "after=", oneYearAgo, &stories)
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
