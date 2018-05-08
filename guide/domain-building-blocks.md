# Domain building blocks

The domain model is one of the integral parts of applications practicing domain-driven design where most of their business logic resides. Because the implementation of the DDD is not a trivial task, the framework includes a number of basic building blocks for the model implementation.

In most of the cases where working with identities, Revo framework takes an opinionated approach and chooses to use globally-unique GUID identifiers.

## Entities

An entity can be any object that has a \(be it local or global\) ID and is implemented using IEntity interface. There are two predefined base classes for entities – `BasicEntity` and `EventSourcedEntity` \(see below for more information\). They always should be a member of an aggregate.

## Aggregates

Aggregates are one of the most important concepts in DDD defining the consistency and transaction boundaries. They are implemented with their aggregate roots as `IAggregateRoot` \(which itself is also an entity\) that provide a public interface for working with the aggregate.

### Basic aggregates and entities

`BasicAggregateRoot` and `BasicEntity` define base classes for aggregates and entities which would usually be persisted using their visible state \(like their public properties - i.e. when mapping to a RDBMS using an ORM\). These are also annotated as database entities \(`DatabaseEntityAttribute`\) which means their model definition will automatically be picked up by ORMs like Entity Framework \(see 5.5.6\) and they implement `IQueryableEntity` which means it will be possible to query them using IQueryables with repositories \(see 5.5 for more information on repositories\). By default, they will be automatically row-versioned to implement optimistic concurrency when saving them to a database \(increasing their version number with each save\).

For compatibility with ORMs like Entity Framework, these basic entities will usually need to define a parameterless constructor \(which can be protected and should not limit the options to expose another public constructor taking parameters and ensuring entity invariants\).

### Event sourced aggregates

Event sourced aggregates implemented using `EventSourcedAggregateRoot` and `EventSourcedEntity` will use events to modify their state. Unlike the basic entities, they should only modify their state upon receiving a new event. Aggregates can internally publish new events using the `Publish<TEvent>` method. This pushes the event to the internal queue of uncommitted events that will get persisted once the repository is saved and invokes an event handler for the specific event type that actually produces the effect of modifying the internal state of the aggregate \(e.g. changes the values of its fields\). The same event handlers get also invoked when the aggregate gets loaded from a repository.

{% hint style="info" %}
When loading an event-sourced aggregate, the repository first creates a blank instance of the aggregate and then replays all the events that the aggregate published previously – effectively reconstructing its complete state. It is obvious that it is vital for event-sourced aggregates to only use event for progressing their state as change made in any other way will get lost upon the next loading.
{% endhint %}

`EventSourcedAggregateRoot` and `EventSourcedEntity` use a convention-based event handling by default and will invoke any methods with the following signature:

```csharp
void Apply(TEvent);
```

\(considering `TEvent` is the type of the event handled\).

These methods need to be either private, protected or internal \(non-public in general\). Furthermore, all events published by an aggregate need to derive from `DomainAggregateEvent`. This domain type includes the aggregate ID which gets automatically injected when publishing the event within an aggregate.

### Example

Here is a contrived example of a simple aggregate publishing and handling an event:

```csharp
[DomainClassId("061ABAC8-7BE0-46D6-A6DB-BA8593027EC9")]
public class Ticket : EventSourcedAggregateRoot
{
    public Ticket(Guid id, string subject, User assignedUser)
        : base(id)
    {
        ApplyEvent(new TicketCreatedEvent(name, assignedUser?.Id));
    }

    protected Ticket(Guid id) : base(id)
    {
    }

    public string Subject { get; private set; }
    public Guid? AssignedUser { get; private set; }
    public TicketState State { get; private set; }

    public void Resolve()
    {
        if (State == TicketState.Resolved)
        {
            throw new InvalidOperationException(
                $"{this} has already been resolved");
        }

        ApplyEvent(new TicketResolvedEvent());
    }

    private void Apply(TicketCreatedEvent ev)
    {
        Subject = ev.Subject;
        AssignedUser = ev.AssignedUser;
        State = TicketState.Pending;
    }
    
    private void Apply(TicketResolvedEvent ev)
    
        State = TicketState.Resolved;
    }
}
```

## Event routing inside an aggregate

Entities that are part of an event-sourced aggregate can also participate in the even-driven architecture. Any event-sourced entity can react to the publishing of an event inside an aggregate \(i.e. not just event published by itself, but also events published by other entities or the aggregate root itself\). For that, it is enough to simply derived from `EventSourcedEntity` or directly `EventSourcedComponent`, passing in the event router from the aggregate root \(its `EventRouter`\). The event router takes care of routing the events inside the aggregate, delivering them to all registered components. With that, the event sourced entities and components can use the same `Apply` method convention for handling events as with aggregate root and use the event router for publishing new events. 

As already mentioned, it is also possible to compose the aggregate roots and entities of several smaller components that are however not entities of their own \(i.e. they have no distinct identity\). To do that, all that is needed is to derive from `EventSourcedComponent` instead of `EventSourcedEntity` instead.

## Sagas

The framework implements a comprehensive support for long-running process coordination using sagas. Thanks to this support, sagas can be considered first-class citizens of the domain model. Because of complexity of this topic, there is a [separate chapter](sagas.md) dedicated to them.

