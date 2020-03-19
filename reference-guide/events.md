---
description: >-
  Events are one of the most powerful ways of communication used in Revo. The
  framework offers a few different ways for implementing an event-driven
  architecture.
---

# Events

## Quick start

There is a number of reasons you'd want to publish events, e.g.:

* progress the state of an [event sourced aggregate](domain-building-blocks.md#event-sourced-aggregates),
* [project read model](projections.md#entity-event-projections) from these events for such aggregates,
* use it as a mean for inter-aggregate communication \(event-driven architecture in DDD\),
* communicate across system boundaries \([integrations](integrations.md#rabbitmq-messaging-with-easynetq)\).

This part shows how to define an event, implement an event listener, register it and publish an event in the simplest manner. Further elaboration on how the system works is provided in later chapters.

{% hint style="info" %}
Please note that this is primarily intended for more advanced usage scenarios or learning the framework architecture. You generally don't need to write your own listeners when you just want to use projections.
{% endhint %}

### Define an event

```csharp
public class ShoppingCartItemAddedEvent : DomainAggregateEvent
{
	public ShoppingCartItemAddedEvent(Guid customerId, Guid itemId, int amount)
	{
	  CustomerId = customerId;
		ItemId = itemId;
		Amount = amount;
	}
	
	public Guid CustomerId { get; }
	public Guid ItemId { get; }
	public int Amount { get; }
}
```

### Implement an event listener

* **I need at-least-once delivery and event processing order guarantees.** __→ Implement an async event listener along with an event sequencer:

```csharp
public class ShoppingCartEventListener :
    IAsyncEventListener<ShoppingCartItemAddedEvent>
{
    private readonly IMarketingServiceApi marketingServiceApi;

    public SubmissionProcessingEventListener(SubmissionProcessingEventSequencer eventSequencer,
        IMarketingServiceApi marketingServiceApi)
    {
        EventSequencer = eventSequencer;
        this.marketingServiceApi = marketingServiceApi;
    }

    public async Task HandleAsync(IEventMessage<ShoppingCartItemAddedEvent> message, string sequenceName)
    {
        await marketingServiceApi.NotifyCustomerIsInterestedAsync(
            message.Event.CustomerId,
            message.Event.ItemId);
    }

    public Task OnFinishedEventQueueAsync(string sequenceName)
    {
        return Task.CompletedTask;
    }

    public IAsyncEventSequencer EventSequencer { get; }

    public class SubmissionProcessingEventSequencer : AsyncEventSequencer<ShoppingCartItemAddedEvent>
    {
        public readonly string QueueNamePrefix = "ShoppingCartEventListener:";

        protected override IEnumerable<EventSequencing> GetEventSequencing(IEventMessage<ShoppingCartItemAddedEvent> message)
        {
            yield return new EventSequencing()
            {
                SequenceName = QueueNamePrefix + message.Event.AggregateId.ToString(),
                EventSequenceNumber = message.Metadata.GetStreamSequenceNumber()
            };
        }

        protected override bool ShouldAttemptSynchronousDispatch(IEventMessage<ShoppingCartItemAddedEvent> message)
        {
            return false;
        }
    }
}
```

* **I don't care about delivery guarantees or don't want the overhead of async events** \(typically only a few usage scenarios, see following chapters for details\). → Implement a regular event listener:

```csharp
public class ShoppingCartEventHandler
   : IEventHandler<ShoppingCartItemAddedEvent>
{
	public ShoppingCartEventHandler()
	{
	}
	
	public Task HandleAsync(IEventMessage<ShoppingCartItemAddedEvent> message, CancellationToken cancellationToken)
	{
		Console.WriteLine($"Hey, a customer is interested in a product: {message.Event.ItemId}!");
	}
}
```

### Register the event listener

```csharp
public class MyModule : NinjectModule
{
    public override void Load()
    {
        // when using async listener
        Bind<IAsyncEventSequencer<ShoppingCartItemAddedEvent>, ShoppingCartEventHandler.SubmissionProcessingEventSequencer>()
            .To<ShoppingCartEventHandler.SubmissionProcessingEventSequencer>()
            .InTaskScope();

            Bind<IAsyncEventListener<ShoppingCartItemAddedEvent>>()
            .To<ShoppingCartEventHandler>()
            .InTaskScope();
        
        // when using regular listener
        Bind<IEventListener<ShoppingCartItemAddedEvent>>()
            .To<ShoppingCartEventHandler>()
            .InTaskScope();
    }
}
```

### Publish the event

Either,

* **Publish an event from an aggregate, in a transactional manner \(alongside saving the aggregate changes to database\):**

```csharp
[DomainClassId("F23D8F21-D2A4-4B47-BC64-8D01795A3471")]
public class ShoppingCart : EventSourcedAggregateRoot
{
    /** CODE OMITTED FOR BREVITY **/

    public void AddItem(ShopItem item, int amount)
    {
        /** omitted some business logic here... **/
        
        Publish(new ShoppingCartItemAddedEvent(item.Id,
          this.CustomerId, amount));
    }

    /** ... **/
}
```

* **Publish an event from anywhere else, e.g. straight from a command handler or from an external source:**

```csharp
IEventBus eventBus = ...;

// ... somewhere later in code ...

await eventBus.PublishAsync(
    EventMessage.FromEvent(
        new ShoppingCartItemAddedEvent(
            Guid.Parse("1DCF3A44-6343-48C9-8E34-A8C8C8F0D26F"),
            Guid.Parse("2AEEB591-53BF-4DE8-90FD-DBBDF3C7FFE1"),
            1),
        new Dictionary<string, string>()
        {
            { BasicEventMetadataNames.EventSourceName, "SupplierApi@1.2.3.4" }
        }));
```

## General overview

### Events and listeners

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

Because the generic event parameter in the event listener interface is defined as contravariant and so is the actual listener resolving mechanism under the hood of default event bus implementation, it is also possible to listen for more general base types of events \(e.g. `DomainEvent` listener will also receive events of type `DomainAggregateEvent` and all types derived from it\). Single event type can be handled by any amount of event handlers, which are registered using the dependency container \(their order of invocation is not guaranteed in any way\).

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
	
	public Guid CustomerId { get; }
	public Guid ItemId { get; }
	public int Amount { get; }
}
```

As you can see in the previous code listing, it is considered a good practice to make the event classes immutable in order to ensure their data never changes once they are created.

### Event messages and metadata

The HandleAsync does not take the event itself as an argument, but rather an event message \(`IEventMessage<T>`\). Event message is an envelope wrapping the event with additional metadata that may not be primarily important for the domain \(i.e. they are not a part of the event’s definition on its own\), but still might be needed elsewhere \(especially when consuming the events outside of the domain model core\). This allows to keep the event type definitions simple and clean of things unrelated to their purpose from the domain perspective.

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

Additional metadata can be appended to event messages by registering new implementations of `IEventMetadataProvider` in the DI container. These providers are invoked by the `EventMessageFactory` when the event messages get constructed \(i.e. typically before saving to event store or sending to other connected systems\).

### Event bus

The entry point for system-wide distribution of events is the event bus. Primarily, it handles the routing of events to listeners that are registered for the specified type\(s\) of events \(basically, it implements a simple messaging bus functionality – see chapter 6.4 for more details about this pattern\). When publishing an event, the event bus then iterates through all listeners registered for the type of the specified event \(or for a base type of it\) and invokes them. 

```csharp
public interface IEventBus
{
	Task PublishAsync(IEventMessage message,
        CancellationToken cancellationToken = default(CancellationToken));
}
```

### Event versioning

The framework implements a versioning system for events. There are situations when a need for changing the definition of an event arises - business domain requirements change over time, bugs get fixed and you may sometimes end up needing to change the definition of your domain classes and their events. Because the events in the event store in general cannot be modified once saved, it is possible to define multiple versions of the same event class. The type information when saving an event into an event store consists of event type name + event type version, so the system is able to correctly lookup the corresponding CLR type.

By default, any event class will be considered to be of version 1. When introducing a new version of the event, you can simply create a copy of the old event class and suffix its name with “V” + the number of its version. After that, the original event class updated with the new event definition is to be annotated with an `EventVersionAttribute` specifying its new version \(which makes it possible to preserve its original name\). An example:

```csharp
// Your new event class
[EventVersion(2)]
public class PageBookmarkedEvent : IEvent
{
	public PageBookmarkedEvent(string pageUrl, string folderName)
	{
	    PageUrl = pageUrl;
	    FolderName = folderName;
	}
	
	public string PageUrl { get; }
	public string FolderName { get; } // added new attribute in V2
}
```

```csharp
// Your original event class
public class PageBookmarkedEventV1 : IEvent
{
	public PageBookmarkedEvent(string pageUrl)
	{
	    PageUrl = pageUrl;
	}
	
	public string PageUrl { get; }
}
```

Note that this does not replace the old instances of the event \(like automatically upgrading them to the newer version\), so you either need to keep the event handlers for both of the versions of the event \(less practical approach\) or implement appropriate [event upgrades](events.md#event-upgrades).

### Event upgrades

Once your application matures and you define more [historical versions](events.md#event-versioning) of your events, it would become cumbersome to maintain handlers \(ApplyEvent methods in event-sourced aggregates...\) for all the previous and current versions at once. To mitigate this, framework offers automatic event upgrades based on the event transformations you define.

Event upgrades are applied any time an event-sourced aggregate is loaded from an event store and the aggregate is then loaded with the new, upgraded events. All you have to do to implement an event upgrade is to define a class deriving from `IEventUpgrade` \(Revo auto-discovers these upon startup\).

{% hint style="info" %}
For most cases, it is recommended to use the generic `EventUpgrade<TAggregate>` class as a base class for your upgrades, which also check the aggregate class before trying to apply any upgrades \(which is more efficient if the event is used by only one aggregate class\).
{% endhint %}

```csharp
public class BookmarkCollectionEventUpgrade : EventUpgrade<BookmarkCollection>
{
    protected override IEnumerable<IEventMessage<DomainAggregateEvent>> DoUpgradeStream(IEnumerable<IEventMessage<DomainAggregateEvent>> events)
    {
        foreach (var message in events)
        {
            if (message.Event is PageBookmarkedEventV1 pageBookmarkedEventV1)
            {
                yield return message.Upgrade(new PageBookmarkedEvent(message.Event.PageUrl, "Root"));
            }
            else
            {
                yield return message;
            }
        }
    }
}
```

An event upgrade takes a stream \(`IEnumerable`\) of all original aggregate's event messages and then returns an upgraded stream of event messages. You can easily implement this transformation using C\# `yield return` operator.

{% hint style="success" %}
As you can see in the previous code The framework implements a helper function `EventMessageUpgradeExtensions.Upgrade<TSource>` for upgrading the event message, which returns a new event message with the domain event replaced while keeping all original message metadata.
{% endhint %}

For code brevity, there are also a few helper methods in `EventMessageUpgradeExtensions` class making the transformation even easier, e.g.:

```csharp
return events
    .Replace<PageBookmarkedEventV1>(message => new DomainAggregateEvent[]
    {
        new PageBookmarkedEvent(message.Event.PageUrl, "Root")
    })
    .Remove<SomeOldCompletelyRedundantEvent>();
```

{% hint style="warning" %}
 At this moment, event upgrades do not work for your arbitrary event listeners that you have defined and are used **only for event-sourced aggregate loading**. This means that if you get an event from an external out-dated system or you have any unprocessed queued events of an outdated version, these events won't be automatically upgraded and you have to implement their support manually.
{% endhint %}

## Synchronous event processing

The flow of events from publisher to regular listeners \(`IEventListener<TEvent>`\) via the event bus is the basic way of processing events that is implemented by the framework. As such, it is completely synchronous – meaning that all the listeners are invoked sequentially, one-by-one and without guaranteed order, in a synchronous manner. Because of their nature, synchronous event listeners posess _no delivery guarantees_ and should _rarely be used_ in actual application code \(possible valid use cases include notifications that only have transient effects\).

{% hint style="info" %}
No synchronous listener is invoked before all preceding listeners have finished processing of the event. Furthermore, if any of the listeners fails, following listeners will not get invoked. This also impedes system performance in situations where multiple listeners could have been ran in parallel and also means that the entire processing stalls any further processing of the \(HTTP\) request whose handling originally caused publishing of the event.
{% endhint %}

{% hint style="success" %}
The distinction between synchronous and asynchronous event processing made here _does not refer_ to the`async/await`features of C\# \(the Task Asynchronous Pattern ak TAP; in fact, most of the framework codebase is already _async all the way_\), but rather to the a/synchronicity of the event delivery and processing itself in regard to the event publishing and to other listeners.
{% endhint %}

## Asynchronous event processing

This framework introduces the concept of asynchronous listeners as a more practical approach to event processing, effectively dealing with the issues that synchronous listeners have. Akin to their synchronous counterparts, asynchronous \(or just async for short\) listeners are defined by implementing `IAsyncListener<TEvent>` for the specific event \(base\) type and registered using the dependency container. Events asynchronously delivered via this interface are a bit different in their nature when compared to the events delivered using the regular \(synchronous\) event pipeline.

Firstly, asynchronous events have guaranteed delivery. This means that once an event gets accepted for the asynchronous processing, it will be delivered at least once \(a.k.a. _at-least-once delivery_\). For practical reasons, the system does not guarantee if the event gets delivered once or more than once and naturally it cannot guarantee when will the actual delivery happen. This is achieved using an intermediate persistent buffer to store the events that have not been fully processed \(with all registered asynchronous listeners\) yet. If an event listener was to fail, the asynchronous event processor can always retry later using this saved data.

Another important aspect previously discussed in regard to the asynchronous nature of event processing was ordering of the events delivered. In order to be able to guarantee that the listeners will always receive events in order \(if they signal they need to, as this may not be a requirement for all asynchronous listeners\), the system works with asynchronous event queues. With those, every async event listener is able to define its event sequencing requirements for every particular event. To do so, every async listener also needs to register its own event sequencer \(`IEventSequencer<TEvent>`\). When the async processing of an event begins, the event dispatcher calls all of these registered sequencers. An event sequencer primarily defines two properties – which event sequence\(s\) it wants to use for the event \(which in turn implies the event queue it will be pushed to\) and what sequence number\(s\) will it assign to that event. Based on that information, the event dispatcher subsequently creates corresponding event queues \(if it does not exist already\) and pushes the event to them. Later when an event backlog worker gets to process any of the queues, it iterates through all the events queued in it, ordered by their sequence number, and passes them to corresponding async event listener\(s\). When the backlog worker is done, it removes the events that were successfully processed from that queue. If processing of any of the events fails, it stays in the queue until the problem gets resolved and event successfully processed.

There is an important property of all event queues, which is that they always remember the sequence number of last successfully processed event \(which gets updated in a transactional manner when events get dequeued from it\). As long as there are gaps in the event sequence \(at the beginning or possibly in the middle\), the worker will block their processing until the sequence is fixed. Similarly, if the worker encounters events with sequence number lower than the number of last event processed in the queue, it skips them to avoid duplication of work \(thus providing certain degree of idempotency\). Once the sequence becomes fixed again and a worker starts processing the queue again, it dispatches both older events that were backlogged in the queue and the latest events that just got pushed to the queue in a single batch. This has several repercussions to the application design, but also automatically provides strong guarantees for the listeners that need it.

Because not all listeners require these strict ordering rules, any event sequencer can also declare an event to have no sequence number. These events will then be processed separately from sequenced events. This has the upside that these events will not be blocked by other possibly missing events in a sequence, meaning they can always be processed immediately. It is obvious, however, that the use of non-sequenced events will be appropriate under slightly different circumstances.

Depending on the current configuration of the system \(i.e. maximum number of Hangfire thread that are used for background job execution\), the framework will also allow concurrent processing of multiple event queues in parallel.

## Eventual synchronization of event sources

This mechanism described so far still does not solve the error-case scenario when the system fails to dispatch and save the events into queues. Without the events dispatched to corresponding queues, event queues will likely contain gaps in their sequences as soon as any later events arrive in it, blocking any further processing of sequenced events in them. The framework works around this issue using event catch-ups. The event catch-up process consists of three steps:

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

Even though the asynchronous event processing offers many advantages over the synchronous processing \(i.e. scalability, reliability…\), dealing with the consequences of its eventual consistency in relation to the originating event publishing can sometimes be difficult. For example, system might occasionally want to guarantee that when a processing of a request finishes, all its effects on the read model have already been applied as well, which would normally be complicated. To remedy this, the framework offers the possibility of attempted synchronous event dispatch even for asynchronous event handlers.

When any `IAsyncEventSequencer` signals that it wants to attempt synchronous dispatch by implementing`ShouldAttemptSynchronousDispatch`, the events added to its queue that would normally get scheduled for later processing in the background get directly passed to the event processor instead - that is, in a blocking \(synchronous\) manner, possibly on the same thread. If any of the async event listeners fail, they still retain the same properties of other asynchronous events – i.e. their processing gets retried later either upon the restart of the system or at the next processing of the same queue.

## Processing idempotency

As the chapter discussing asynchronous event processing already mentioned, the event pipeline can only guarantee _at-least-once delivery_ of events. This also means that some events may occasionally get delivered more than just once. This is important for the design of actual async event listener implementations, because they need to account for these scenarios. It is necessary that repeated submission of a single event will not change the final result. In mathematics and informatics, this is property also called _operation idempotency_.

Some listeners built-in to the framework already minimize or completely remove the need to do so \(it is however still required to handle this manually ad-hoc in other cases\). One of the automatically handled use cases are read model projections when implemented using `EntityEventToPocoProjectors<,>` along with projection row versioning \(see later chapter on [read models and projections](projections.md)\). 





