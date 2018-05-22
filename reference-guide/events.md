# Events

## Event basics

Events are one of the basic ways of communication used in Revo framework. To make this event-driven architecture possible, it offers a number of facilities to work with them.

### Overview

Framework defines a generic event listener `IEventListener<TEvent>` interface. Event can be any plain object that implements the IEvent interface \(which itself is empty in its definition\). 

```csharp
public interface IEvent
{
}
```

```csharp
public interface IEventListener<in T>
	where T : IEvent
{
   Task HandleAsync(IEventMessage<T> message,
      CancellationToken cancellationToken);
}
```

Because the generic event parameter in the event listener interface is defined as contravariant and so is the actual listener resolving mechanism under the hood of default event bus imple-mentation, it is also possible to listen for more general base types of events \(e.g. it is `DomainEvent` listener will also receive events of type `DomainAggregateEvent` and all types derived from it\). Single event type can be handled by an unlimited amount of event handlers that should be registered in the dependency container \(their order of invocation is not guaran-teed in any way\).

An example of an event with a corresponding event handler can be found below.

```csharp
public class ShoppingCartItemAddedEvent : IEvent
{
	public ShoppingCartItemAddedEvent(Guid customerId, Guid itemId, int amount)
	{
	    CustomerId = customerId;
		ItemId = itemId;
		Amount = amount;
	}
	
	public Guid CustomerId { get; private set; }
	public Guid ItemId { get; private set; }
	public int Amount { get; private set; }
}
```

```csharp
public class ShoppingCartEventHandler
   : IEventHandler<ShoppingCartItemAddedEvent>
{
	public ShoppingCartEventHandler()
	{
	}
	
	public Task HandleAsync(IEventMessage<ShoppingCartItemAddedEvent> message, CancellationToken cancellationToken)
	{
		Console.WriteLine($"Hey, a customer is interested in a product: {mes-sage.Event.ItemId}!");
	}
}
```

As can be seen in the previous code listing, it is considered a good practice to make the event classes immutable in order to ensure their data never changes once they are created \(event immutability is further discussed in chapter 2.5\).

### Event messages and metadata

It is worth mentioning that the HandleAsync does not take the event itself as an argument, but rather an event message \(`IEventMessage<T>`\). Event message is an envelope wrapping the event with additional metadata that may not be primarily important for the domain \(i.e. they are not a part of the event’s definition on its own\), but still might be necessary under some circumstances \(especially when consuming the events outside of the domain model core\). This allows to keep the event type definitions simple and clean of things unrelated to their purpose from the domain perspective.

```csharp
public interface IEventMessage
{
	IEvent Event { get; }
	IReadOnlyDictionary<string, string> Metadata { get; }
}

public interface IEventMessage<out TEvent> : IEventMessage
	where TEvent : IEvent
{
	new TEvent Event { get; }
}
```

These additional metadata are represented in form of a string key-value dictionary. Notable examples of such metadata include timestamp or event stream sequence number but might also optionally include other data used solely for other purposes like debugging, for example the name of the machine, that first created the event, or the command and URI of the sever request that triggered the creation of the event. The names for the predefined event metadata keys are defined in class `BasicEventMetadataNames`.

Additional metadata can be appended to event messages by registering new implementations of `IEventMetadataProvider` in the DI container. Using those providers, the actual event message is constructed by the `EventMessageFactory` whenever needed.

### Event bus

The entry point for system-wide distribution of events is the event bus. Primarily, it handles the routing of events to listeners that are registered for the specified type\(s\) of events \(basi-cally, it implements a simple messaging bus functionality – see chapter 6.4 for more details about this pattern\). When publishing an event, the event bus then iterates through all listeners registered for the type of the specified event \(or for a base type of it\) and invokes them. 

```csharp
public interface IEventBus
{
	Task PublishAsync(IEventMessage message,
        CancellationToken cancellationToken = default(CancellationToken));
}
```

### Event versioning

The framework implements a versioning support for events. Business domain requirements often change and so do have to the events and their definitions. Because the events in the event store cannot in general be ex-post modified and renaming event class every time there is a slight change in its definition would be cumbersome, it is possible to define multiple versions of the same event class. The type information when saving an event into an event store consists of event type name + event type version, so the system is able to correctly lookup the corresponding CLR type.

By default, any event class will be considered to be of version 1. When introducing a new version of the event, the developer simply duplicates the old event class and suffixes its name with “V” + the number of its version. After that, the original event class updated with the new event definition is annotated with an `EventVersionAttribute` specifying its new version \(which makes it possible to preserve its original name\). An example follows.

```csharp
[EventVersion(2)]
public class PageBookmarkedEvent : IEvent
{
	public PageBookmarkedEvent(string pageUrl, string folderName)
	{
	    PageUrl = pageUrl;
	    FolderName = folderName;
	}
	
	public string PageUrl { get; private set; }
	public string FolderName { get; private set; } // added new attribute in V2
}
```

```csharp
public class PageBookmarkedEventV1 : IEvent
{
	public PageBookmarkedEvent(string pageUrl)
	{
	    PageUrl = pageUrl;
	}
	
	public string PageUrl { get; private set; }
}
```

Note that this does not replace the old instances of the event \(not yet - like automatically upgrading them to the newer version\), so it is necessary to keep the event handlers for both of the versions of the event.

## Synchronous event processing

The flow of events from the publisher via event bus to listeners is the basic way of processing events that is supported by the framework. As such, it is also completely synchronous – meaning that all the listeners are invoked sequentially, one-by-one, in a synchronous man-ner. This possesses all disadvantages previously discussed in chapter 6.5. No following listener is invoked before all preceding listeners have finished processing of the event. That also means that the event publishing interrupts any further processing of the request that originally caused publishing of the event. This is the way the default event bus is implement-ed when using `IEventListener<TEvent>` interfaces for handling the events. It is, however, not the only way events can be processed in Revo framework.

## Asynchronous event processing

This framework introduces a concept of asynchronous events as a method for dealing with all kinds of these issues. Akin to their synchronous counterparts, asynchronous \(or just async for short\) listeners are defined by implementing `IAsyncListener<TEvent>` for the specific event type or base type. Events asynchronously delivered via this interface are a bit different in their nature when compared to the events delivered using the regular \(synchronous\) event pipeline.

Firstly, asynchronous events have guaranteed delivery. This means that once an event gets accepted for the asynchronous processing, it will be delivered at least once \(a.k.a. _at-least-once delivery_\). For practical reasons, the system does not guarantee if the event gets deliv-ered once or more than once and naturally it cannot guarantee when will the actual delivery happen. This is achieved using an intermediate persistent buffer to store the events that have not been fully processed \(with all registered asynchronous listeners\) yet. If an event listener was to fail, the asynchronous event processor can always retry later using this saved data.

Another important aspect previously discussed in regard to the asynchronous nature of event processing was ordering of the events delivered. In order to be able to guarantee that the listeners will always receive events in order \(if they signal they need to, as this may not be a requirement for all asynchronous listeners\), the system works with asynchronous event queues. With those, every async event listener is able to define its event sequencing requirements for every particular event. To do so, listeners register their own event sequencers. When the async processing of an event begins, the event dispatcher calls all of these registered sequencers. An event sequencer primarily defines two properties – which event sequence\(s\) it wants to use for the event \(which in turn implies the event queue it will be pushed to\) and what sequence number\(s\) will it assign to that event. Based on that information, the event dispatcher subsequently creates corresponding event queues \(if it does not exist already\) and pushes the event to them. Later when an event backlog worker gets to process any of the queues, it iterates through all the events queued in it, ordered by their sequence number, and passes them to corresponding async event listener\(s\). When the backlog worker is done, it removes the events that were successfully processed from that queue. If processing of any of the events fails, it stays in the queue until the problem gets resolved and event successfully processed.

There is an important property of all event queues, which is that they always remember the sequence number of last successfully processed event \(which gets updated in a transactional manner when events get dequeued from it\). As long as there are gaps in the event sequence \(at the beginning or possibly in the middle\), the worker will block their processing until the sequence is fixed. Similarly, if the worker encounters events with sequence number lower than the number of last event processed in the queue, it skips them to avoid duplication of work \(thus providing certain degree of idempotency\). Once the sequence becomes fixed again and a worker starts processing the queue again, it dispatches both older events that were backlogged in the queue and the latest events that just got pushed to the queue in a single batch. This has several repercussions to the application design, but also automatically pro-vides strong guarantees for the listeners that need it.

Because not all listeners require these strict ordering rules, any event sequencer can also declare an event to have no sequence number. These events will then be processed separately from sequenced events. This has the upside that these events will not be blocked by other possibly missing events in a sequence, meaning they can always be processed immediately. It is obvious, however, that the use of non-sequenced events will be appropriate under slightly different circumstances.

Depending on the current configuration of the system \(i.e. maximum number of Hangfire thread that are used for background job execution\), the framework will also allow concurrent processing of multiple event queues in parallel.

## Eventual synchronization of event sources

This mechanism described so far still does not solve the error-case scenario when the system fails to dispatch and save the events into queues. Without the events dispatched to corresponding queues, event queues will likely contain gaps in their sequences as soon as any later events arrive in it, blocking any further processing of sequenced events in them. The framework works-around this issue using event catch-ups. The event catch-up process consists of three steps:

	1.	Pulling non-dispatched events from all event stores. Different kinds of event stores can  
		implement this by implementing their own version of `IEventSourceCatchUp` interface.

	2.	Dispatching these events into corresponding async event queues.

	3.	Running async event queue workers for queues that are lagging behind with any unprocessed  
		\(backlogged\) events.

These catch-ups are ran regularly:

* 	During the application start-up, before processing of any new requests starts.
* 	Periodically in pre-defined \(configured\) time intervals. 

This ensures that all event sources get eventually back into a synchronized state. As noted before, while the regular event-processing path in the success-case scenarios works completely as a _push-based_ mechanism \(i.e. the events get propagated throughout the pipeline actively by their initiator at the time they are created\), which usually should have better efficiency in the use-case mentioned, the catch-ups resort to employing different _pull-based approach_ when loading events from the event store.

{% hint style="info" %}
Despite the fact that catch-ups could also deliver events to synchronous event listeners, it was decided they would rather not. This design decision stems from the fact that synchronous dispatchers cannot safely guarantee some other delivery properties \(like ordering\) and thus they should be kept completely that way with no delivery guarantees altogether to make a clear distinction from asynchronous events. Therefore, they should only be used for actions that have transient effects.
{% endhint %}

## Pseudo-synchronous event dispatch

Even though the asynchronous event processing offers many advantages over the synchro-nous processing \(i.e. scalability, reliability…\), dealing with the consequences of its eventual consistency in relation to the originating event publishing can sometimes be difficult. For example, system might occasionally want to guarantee that when a processing of a request finishes, all its effects on the read model have already been applied as well, which would normally be complicated. To remedy this, the framework offers the possibility of attempted synchronous event dispatch even for asynchronous event handlers.

When any `IAsyncEventSequencer` signals with `ShouldAttemptSynchronousDispatch` that it wants to attempt this synchronous dispatch by, the events added to its queue that would normal-ly get scheduled for later processing in the background get directly passed to the event pro-cessor instead, in a blocking \(synchronous\) manner, possibly on the same thread. If any of the async event listeners fail, they still retain the same properties of other asynchronous events – i.e. they get retried later either upon the restart of the system or at the next processing of the same queue.

## Processing idempotency

As the chapter discussing asynchronous event processing already mentioned, the event pipeline can only guarantee _at-least-once delivery_ of events. This also means that some events may occasionally get delivered more than just once. This is important for the design of actual async event listener implementations, because they need to account for these scenarios. It is necessary that repeated submission of a single event will not change the final result. In mathematics and informatics, this is property also called _operation idempotency_.

Some listeners built-in to the framework already minimize or completely remove the need to do so \(it is however still required to handle this manually ad-hoc in other cases\). One of the automatically handled use cases are read model projections when implemented using `EntityEventToPocoProjectors<,>` along with projection row versioning \(see later chapter on [read models and projections](projections.md)\). 





