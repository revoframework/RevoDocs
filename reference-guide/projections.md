# Projections

It s a practical requirement for most applications with event sourcing to also maintain a co-existent read model for the needs of the query side of the applications \(client user-interfaces, reporting, etc.\). The objects that take care of creating and updating \(or deleting\) read models are called _projectors_ \(as they project the effects of the events into some other form\) and the process itself a _projection_ \(which might sometimes also be used for the resulting generated model\).

## Entity event projections

The framework offers a number of facilities to work with projections and read models. For their operation, they expect a sequence of individual domain events on the input and return a data structure that is better usable for the read side model on the output. At its very core, projections are just conveniently set-up asynchronous event listeners – despite possibly hiding some of their internal complexity at a first glance thanks to the infrastructure provided by the framework. The facilities for read model projection consist of several parts. At the most basic level, it is possible to define entity event projector for a specific aggregate type by simply implementing `IEntityEventProjector` interface and registering it with the framework.

These projectors are always strictly bound to projecting events of single aggregate type – however, it is possible to define many projectors for one aggregate type. The contract of this interface is designed in a simple fashion – whenever the projection manager receives new events from the event bus, it sorts them based on the aggregate ID. Later, when the reading from an event queue gets finished, it starts to iterate through the aggregate and loading the one-by-one from the corresponding repository aggregate stores. By looking up the class ID of every aggregate, it then decides which projectors to invoke for every aggregate and its events. When the projections finish, the projection manager commits all of the registered projectors. This ensures that the projections are executed in an efficient manner, always with a complete batch of events coming from a finished unit of work.

{% hint style="info" %}
The batching of events by their source aggregates also clearly defines the scope and the lifetime of every projector, making it possible to apply several optimizations in the event projection process. However, it also requires that for creating projections combined from events of two or more aggregates \(or aggregate types\), it is necessary to create corresponding number of separate projectors \(all of them working on the same read model\). This is a certain compromise that was made in order to keep the framework architecture simpler and cleaner. Nevertheless, while it might seem to be something that makes writing these extra projections unnecessarily verbose, it makes sense from the perspective of write side consistency. Put it other words, because it is the aggregates themselves that define \(in terms of DDD\) the consistency boundaries, implying the system will always be modifying an aggregate at a time, the projectors will always receive events in batches corresponding to those aggregate modifications. Using the same reasoning, because the modifications of two different aggregates will always be independent and only eventually-consistent, any attempt of immediate consistency of read models across two or more aggregate would only be fictitious and in reality, unattainable.
{% endhint %}

Because many projections will want to work with a simple object \(POCO\) representation of the read model, the framework also defines a stub base classes for these projection implementations. This is convenient when using object-relation mapping libraries \(ORM\), for example.

### EntityEventToPocoProjector

By deriving from `EntityEventToPocoProjector<TSource, TTarget>`, the developer is provided with some projection behavior based on conventions. With the assumption of projection `TTarget` being just a POCO, it declares two abstract methods for creating new projection and finding and existing one, which it then invokes based on the sequence number of first event projected \(i.e. creates a new projection target when the first sequence number is 1\). The actual projection is then done with a number of `Apply` methods that the final projector classes implement, \(possibly, not mandatory\) one for every event type. These methods are located automatically  during the startup of the application and/or construction of the projector class \(there is caching involved for better performance\) using a naming convention . These methods should have any of the following signatures:

```csharp
void Apply(TEvent);
Task Apply(TEvent);
void Apply(TEvent, TSourceId, TTarget);
Task Apply(TEvent, TSourceId, TTarget);
```

With TEvent being the actual type of event applied, Apply methods can be either public or non-public, virtual or non-virtual and both of the parameter variations can either have void return type, or can return a `Task` and work in a C\# async manner. In this way, the amount of repeating logic needed for handling a projection decreases significantly. Writing a new projector is then basically reduced just to creating a new class deriving from this stub and implementing a few `Apply` methods as needed. Moreover, to be able to use object composition for more complex projectors \(instead of object inheritance, which is as experience show often more difficult to test and maintain\), the projectors also support a notion of  
_sub-projectors_. These work in the exact same way, with the exception that they do not need to worry about creating/loading and committing of the projection target, as they only implement the aforementioned `Apply` methods.

As a side benefit, the base implementations also automatically handle projection idempotency as long `TTarget` implements manual row versioning \(`IManuallyRowVersioned`\). If the projection target implements this interface, the projector stub automatically updates the row version based on the number of last event applied and next time it skips any events that are earlier in the sequence.

### CrudEntityEventToPocoProjector

Building on top of the previously introduced EntityEventToPocoProjector, projectors based on 

•	`CrudEntityEventToPocoProjector<TSource, TTarget>`

•	`EF6CrudEntityEventToPocoProjector<TSource, TTarget>` and

•	`RavenCrudEntityEventToPocoProjector<TSource, TTarget>`

add some additional features for easier generation of read models. They are all backed by their corresponding `ICrudRepository` repositories and create the read model instances with its default constructor. Moreover, they implement following features.

•	If the read model class implements `IEntityReadModel` \(e.g. the predefined `EntityReadModel` base class\), the projector automatically injects the Id property based on the aggregate ID.

•	If the read model class implements `IClassEntityReadModel` \(e.g. the predefined `ClassEntityReadModel` base class\), the projector automatically injects the ClassId property based on the aggregate class ID metadata from the event or from the `DomainClassIdAttribute` of the aggregate class itself \(used as a fallback\).

•	If the read model class implements `ITenantReadModel` \(e.g. the predefined `TenantEntityReadModel` base class\), the projector automatically injects the `TenantId` property based on the ID specified in TenantAggregateRootCreated \(which is automatically emitted by the `TenantEventSourcedAggregateRoot` upon its creation\).



