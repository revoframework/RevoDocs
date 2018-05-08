# Testing

## In-memory persistence

To make developer's life easier and encourage the test-driven development approach, Revo features a few helpers for testing of domain models and other parts of the application.

### InMemoryCrudRepository / EF6InMemoryCrudRepository

In-memory CRUD repository makes it possible test any components that persist data to a database with a regular CRUD repository. This in-memory version implements nearly all of the features of its “real” counterparts \(like the EF6\) with the exception of multi-key IDs. Furthermore, the `EF6InMemoryCrudRepository` implements the additional features of EF6 \(like the methods `Entries()`/`Entry<T>(T)` for features like explicit lazy-loading and other\) and by emulating its queryable providers, it makes it possible to seamlessly use the asynchronous `IQueryable<T>` extension methods defined by it, e.g. `ToListAsync()` and others.

### FakeRepository

Similar to the in-memory CRUD repository, `FakeRepository` implements the domain aggregate `IRepository` without any actual database dependencies, making it suitable for testing purposes.

## Domain model testing

To make testing of event-sourced entities less verbose and correctly test of all its aspects, i.e. not just the emission of new events and possibly a change in the public entity state, but also the fact that the entity is able to deserialize back to that state again when replaying the events from the database, the framework defines a few extension methods.

### EventSourcedEntityTestingHelpers

Method `AssertEvents` asserts that given an initial object state and an action performed, it publishes an array of specified events and possibly also asserts the final state of the object with a custom function. If loadEventsOnly is specified true, it only replays the events speci-fied and verifies the final state.

```csharp
static void AssertEvents<T>(this T aggregate, Action<T> action,
    Action<T> stateAssertion, bool loadEventsOnly,
    params DomainAggregateEvent[] expectedEvents)
where T : EventSourcedAggregateRoot;

static void AssertAllEvents<T>(this T aggregate, Action<T> action,
    Action<T> stateAssertion, bool loadEventsOnly,
    params DomainAggregateEvent[] expectedEvents)
where T : EventSourcedAggregateRoot;

static void AssertConstructorEvents<T>(this T aggregate,
	Action<T> stateAssertion, bool loadEventsOnly,
	params DomainAggregateEvent[] expectedEvents)
where T : EventSourcedAggregateRoot;
```

Example using xUnit test framework:

```csharp
[Theory]
[InlineData(false)]
[InlineData(true)]
public void UpdateDetails(bool loadEventsOnly)
{
	var user = new User(
        Guid.Parse("36418768-498B-47B2-BDE0-4165EAE7E5F7"),
        "original@email");

	sut.AssertEvents(
		x =>
		{
			x.UpdateEmailAddress("new@email");
		},
		x =>
	{
		x.EmailAddress.Should().Be("new@email");
	}, loadEventsOnly,
	new UserEmailUpdatedEvent("new@email"));
}
```

`AssertAllEvents` has the same call signature, but it asserts all of the uncommitted events that the entity contains and not just the events that were published after calling the action. The last variant, `AssertConstructorEvents`, does not take an action parameter, making it suitable for testing the state \(and events\) of newly constructed entities.

