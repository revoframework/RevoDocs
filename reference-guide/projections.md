---
description: >-
  Projection are specialized event listeners used for materializing events
  published by event sourced aggregates into better usable read models.
---

# Projections

It is a practical requirement for most applications with event sourcing to also maintain a co-existent read model for the needs of the query side of the applications \(client user-interfaces, reporting, etc.\). This read model will typically exist in a form of a record in another database, that we can **query easily** \(e.g. a **row in a relational DBMS**\). The object that takes care of creating and updating read model of a single aggregate type is called _projector_ \(as they project the effects of the events into some other form\).

## Short example

Following sample illustrates how a projector using EF Core read model can be implemented for simple Todo aggregate.

{% tabs %}
{% tab title="TodoReadModelProjector .cs" %}
```csharp
public class TodoReadModelProjector :
  EFCoreEntityEventToPocoProjector<Todo, TodoReadModel>
{
    public TodoReadModelProjector(IEFCoreCrudRepository repository)
      : base(repository)
    {
    }

    private void Apply(IEventMessage<TodoRenamedEvent> ev)
    {
        Target.Name = ev.Event.Name;
    }
}
```
{% endtab %}

{% tab title="TodoReadModel.cs" %}
```csharp
[TablePrefix(NamespacePrefix = "TODOS", ColumnPrefix = "TDO")]
public class TodoReadModel : EntityReadModel
{
  public string Text { get; set; }
}
```
{% endtab %}

{% tab title="Todo.cs" %}
```csharp
[DomainClassId("D8A1F0C6-CD0A-4F66-8181-336AAFE11248")]
public class Todo : EventSourcedAggregateRoot
{
    /** CODE OMITTED FOR BREVITY **/

    public string Name { get; private set; }
    
    public void Rename(string name)
    {
        if (Name != name)
        {
            Publish(new TodoRenamedEvent(name));
        }
    }
    
    private void Apply(TodoRenamedEvent ev)
    {
        Name = ev.Name;
    }
}
```
{% endtab %}

{% tab title="TodoRenamedEvent.cs" %}
```
public class TodoRenamedEvent : DomainAggregateEvent
{
    public TodoRenamedEvent(string name)
    {
        Name = name;
    }

    public string Name { get; }
}
```
{% endtab %}
{% endtabs %}

## Basics

You can easily define a projector by implementing a generic interface specific to used data persistence provider and specifying the projected aggregate type as one of its generic parameters. By default, these projectors are auto-discovered and registered upon application startup.

{% hint style="info" %}
A projector class always projects events of a single aggregate type.
{% endhint %}

Every persistence provider also offers a few convenience base classes that implement features like versioning with optimistic-concurrency or automatic creating of POCO read models.

### Entity event projectors

Any projector derived from `EntityEventProjector` \(which are all convenience projector base classes provided by Revo\) will use a convention to look for methods on the projector class which will be called for individual events projected.

These methods should have any of the following signatures:

```csharp
void Apply(IEventMessage<TEvent> ev);
Task Apply(IEventMessage<TEvent> ev);
void Apply(IEventMessage<TEvent> ev, Guid aggregateId);
Task Apply(IEventMessage<TEvent> ev, Guid aggregateId);
void Apply(IEventMessage<TEvent> ev, Guid aggregateId, TTarget target);
Task Apply(IEventMessage<TEvent> ev, Guid aggregateId, TTarget target);
```

\(with `TEvent` being the event class and `TTarget` being the read model class\).

### POCO \(CRUD\) read model projectors

A POCO projector is a CRUD \(create/read/update/delete\) repository-backed event projector for an aggregate type with a single POCO read model object \(row\) for every aggregate.

On top of the Apply-method convention, POCO read model projectors \(derived from`EntityEventToPocoProjector<TSource, TTarget>`\) also

* Automatically creates, loads and saves read model instances using the repository based on aggregate ID,
* If read model class is `IManuallyRowVersioned`, automatically handles read model versioning and projection idempotency using event sequence numbers,
* Automatically sets read model IDs for read model that is `IEntityReadModel`, class IDs for `IClassEntityReadModel` and tenant IDs for `ITenantReadModel`.

### Read model base clases

Because many projections will want to work with a simple object \(POCO\) representation of the read model, the framework also defines a stub base classes for these projection implementations. This is convenient when using object-relation mapping libraries \(ORM\), for example.

* If the read model class implements `IEntityReadModel` \(e.g. the predefined `EntityReadModel` base class\), the projector automatically injects the Id property based on the aggregate ID.
* If the read model class implements `IClassEntityReadModel` \(e.g. the predefined `ClassEntityReadModel` base class\), the projector automatically injects the ClassId property based on the aggregate class ID metadata from the event or from the `DomainClassIdAttribute` of the aggregate class itself \(used as a fallback\).
* If the read model class implements `ITenantReadModel` \(e.g. the predefined `TenantEntityReadModel` base class\), the projector automatically injects the `TenantId` property based on the ID specified in TenantAggregateRootCreated \(which is automatically emitted by the `TenantEventSourcedAggregateRoot` upon its creation\).

## Providers

### EF Core

EF Core persistence provider automatically discovers projectors implementing:

* `IEFCoreSyncEntityEventProjector<TSource>`
  * `EFCoreSyncEntityEventToPocoProjector<TSource, TTarget>` for POCO read models
  * `EFCoreSyncEntityEventProjector<TSource>` for arbitrary read models
* `IEFCoreEntityEventProjector<TSource>`
  * `EFCoreEntityEventToPocoProjector<TSource, TTarget>` for POCO read models
  * `EFCoreEntityEventProjector<TSource>` for arbitrary read models

{% hint style="success" %}
As you can see, EF Core provider also supports **synchronous variants** for all projector interfaces, which are then \(opposed to the normal projectors\) run inside single database transaction as saving the aggregate itself \(as well as queueing asynchronous events, etc.\). This usually leads to **better performance** if you need to wait for the projections anyway \(on the other hand, if you don't and/or are doing heavy computations in the projector, you may be better off with asynchronous versions\).
{% endhint %}

### Entity Framework 6

EF6 persistence provider automatically discovers projectors implementing:

`IEF6EntityEventProjector<TSource>`

* `EF6EntityEventToPocoProjector<TSource, TTarget>` for POCO read models
* `EF6EntityEventProjector<TSource>` for arbitrary read models

### RavenDB

RavenDB persistence provider automatically discovers projectors implementing:

`IRavenEntityEventProjector<TSource>`

* `RavenEntityEventToPocoProjector<TSource, TTarget>` for POCO read models
* `RavenEntityEventProjector<TSource>` for arbitrary read models

## Background technical details

The framework offers a number of facilities to work with projections and read models. For their operation, they expect a sequence of individual domain events on the input and return a data structure that is better usable for the read side model on the output. At its very core, projections are just conveniently set-up asynchronous event listeners – despite possibly hiding some of their internal complexity at a first glance thanks to the infrastructure provided by the framework. The facilities for read model projection consist of several parts. At the most basic level, it is possible to define entity event projector for a specific aggregate type by simply implementing `IEntityEventProjector` interface and registering it with the framework.

These projectors are always strictly bound to projecting events of single aggregate type – however, it is possible to define many projectors for one aggregate type. The contract of this interface is designed in a simple fashion – whenever the projection manager receives new events from the event bus, it sorts them based on the aggregate ID. Later, when the reading from an event queue gets finished, it starts to iterate through the aggregate and loading the one-by-one from the corresponding repository aggregate stores. By looking up the class ID of every aggregate, it then decides which projectors to invoke for every aggregate and its events. When the projections finish, the projection manager commits all of the registered projectors. This ensures that the projections are executed in an efficient manner, always with a complete batch of events coming from a finished unit of work.

{% hint style="info" %}
The batching of events by their source aggregates also clearly defines the scope and the lifetime of every projector, making it possible to apply several optimizations in the event projection process. However, it also requires that for creating projections combined from events of two or more aggregates \(or aggregate types\), it is necessary to create corresponding number of separate projectors \(all of them working on the same read model\). This is a certain compromise that was made in order to keep the framework architecture simpler and cleaner. Nevertheless, while it might seem to be something that makes writing these extra projections unnecessarily verbose, it makes sense from the perspective of write side consistency. Put it other words, because it is the aggregates themselves that define \(in terms of DDD\) the consistency boundaries, implying the system will always be modifying an aggregate at a time, the projectors will always receive events in batches corresponding to those aggregate modifications. Using the same reasoning, because the modifications of two different aggregates will always be independent and only eventually-consistent, any attempt of immediate consistency of read models across two or more aggregate would only be fictitious and in reality, unattainable.
{% endhint %}



