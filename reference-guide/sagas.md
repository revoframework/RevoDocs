# Sagas

## Overview

Sagas implement a way of coordination of long-running processes and collaboration between eventually consistent aggregates. To do that, they listen for published domain events and send out new commands.

{% hint style="info" %}
Sagas in Revo framework are implemented as stateful process managers similarly to some other frameworks. These concepts and the terminology is further discussed in chapter 3.9.1.
{% endhint %}

## Basic usage with saga keys

By default, the two predefined saga base classes \(`BasicSaga` and `EventSourcedSaga`, based on different persistence mechanisms as explained below\) use a convention-based mapping for their registration and event handling. Events can be handled with  void `Handle(IEventMessage<TEvent> ev)` methods that take a single argument of type `IEventMessage<TEvent>` where `TEvent` is the specific type of event to handle \(its subtypes will not be matched\). These methods need to be decorated with a `SagaMethodAttribute` that specifies how the saga instances should be located. An example:

```csharp
[SagaMethod(SagaKey = "UserId", EventKey = "AggregateId")]
private void Handle(
    IEventMessage<UserVerificationTimeoutExiredEvent> ev)
{
    if (!IsUserVerified)
    {
        Send(new CancelUserRegistrationCommand(UserId));
        End();
    }
}
```

These methods can have any access modifier \(but private is usually preferred\). For sagas that implement `IConventionBasedSaga` \(applies for both mentioned saga base classes\), this also means they will be automatically registered in the saga registry for the event types they implement if they are found in any of the referenced assemblies during the startup, so they are invoked when a saga event dispatch happens.

### Saga method binding

The `SagaMethodAttribute` specifies what saga instances should the event be sent to. There are currently five options available:

* Always start a new saga when the event happens.

```csharp
[SagaEvent(IsAlwaysStarting = true)]
```

* Find all existing sagas.

```csharp
[SagaEvent]
```

* Find all existing sagas **and **start a new one if none were found.

```csharp
[SagaEvent(IsStartingIfSagaNotFound = true)]
```

* Find existing sagas by correlating a property of the event and a saga key.

```csharp
[SagaEvent(SagaKey = "Foo", EventKey = "Bar")]
```

* Find existing sagas by correlating a property of the event and a saga key and start a new one if none were found.

```csharp
[SagaEvent(SagaKey = "Foo", EventKey = "Bar", IsStartingIfSagaNotFound = true)]
```

The saga correlation keys need to be previously set by the saga itself using methods like `AddSagaKey`/`SetSagaKey`. Sagas can save multiple values for one key, allowing it to react to any of events correlated to them. It is also possible to specify multiple `SagaEventAttributes` for one method.

### Sending commands

Because sagas should not have any external side effects just like regular aggregates and by default, the framework will not inject any dependencies into them, they have only one primary means of communication with the outside world – sending commands and publishing events. Commands sent using `Send` method are queued and get actually processed by the command bus upon committing and saving the saga \(which happens automatically when saga event processing is finished\).

```csharp
Send(new CancelUserRegistrationCommand(UserId));
```

 If the processing of any of the commands fail, the saga state is not saved, and the handling of the saga will be retried later. For this reason, it is vital that the commands are idempotent in their effect, because they may get sent more than once in case of such failure. Sagas will often want to schedule the commands for processing at a later time or simply to enqueue them for a processing in a background worker queue \(asyn-chronously of their processing\) – this can easily be achieved using job commands. For more on this topic, please see chapter on [Jobs](jobs.md).

### Saga state and metadata

Sagas can also have their own state. This state will be persisted in the same way as with aggregates \(i.e. persisting state of `BasicSaga`s and persisting event stream of `EventSourcedSaga`s\), because the system also internally uses the regular IRepository.

{% hint style="warning" %}
Saga metadata \(such as the keys and class IDs\) are stored independently of the sagas in an ISagaMetadataRepository \(these two operations are not atomic and are carried out in order first saga data, then saga metadata; this also means that the sagas need to count with the possibility that their metadata are not up-to-date and synchronized with their state and that the processing of an event will be retried\).
{% endhint %}

