---
description: >-
  Scale and integrate by publishing and receiving events, commands and queries
  using common messaging patterns, e.g. with RabbitMQ message queue (using
  EasyNetQ connector or Rebus service bus).
---

# Messaging and integrations

## RabbitMQ messaging with EasyNetQ

[EasyNetQ ](http://easynetq.com/)is an easy-to-use library for communication with the popular open-source [RabbitMQ](https://www.rabbitmq.com/) message broker. Using the **Revo.EasyNetQ** package, you should be able to implement simple messaging patterns in a Revo application in a matter of minutes.

Currently, the EasyNetQ package support only publishing and subscribing to events.  
An example configuration follows:

```csharp
new RevoConfiguration()
    ...
    .UseEasyNetQ(true, new EasyNetQConnectionConfiguration("host=localhost"),
        subscriptions => subscriptions.AddType<ExternalIntegrationEvent>("MyApp.Integration.Machine0"),
        advancedAction: cfg => cfg.EventTransports.AddType<MyAppIntegrationEvent>())
    ...;
```

This configuration connects to a RabbitMQ server on localhost \(see the [EasyNetQ connection string formats](https://github.com/EasyNetQ/EasyNetQ/wiki/Connecting-to-RabbitMQ) for reference\) and subscribes to a single event \(base\) type `ExternalIntegrationEvent`  using the specified subscriber ID. Anytime a new event is received from a corresponding \(as per EasyNetQ configuration\) RabbitMQ queue\(s\), the event is propagated inside the Revo application to all its registered `IEventListener<T>` listeners \(see [event listener registration](events.md#register-the-event-listener)\).

Furthermore, it also registers an event transport for an event \(base\) type `MyAppIntegrationEvent`. Anytime this event type \(or its derived type\) is published on the `IEventBus`, the Revo also automatically publishes the event to the RabbitMQ \(in this case, to the default message exchange configured by EasyNetQ\).

## Rebus service bus integration

[Rebus](https://github.com/rebus-org/Rebus) is a simple and lean service bus implemented in .NET. Out-of-the-box, Revo currently offers only a limited and experimental support for its integration \(_.NET 4.7.1+ only_\). For simpler, but production-ready messaging integration, you can use the aforementioned [RabbitMQ connector](integrations.md#rabbitmq-messaging-with-easynetq) implemented using EasyNetQ library.

When using the **Revo.Rebus** module, the framework automatically hooks with the command and event bus. The connection parameters for the RabbitMQ message queue can be specified in default application configuration \(either via **app.config** or **Web.config** file\) in form of a connection string defining the server URL to connect to and the input queue to subscribe to, e.g.:

```text
<connectionStrings><add name="RabbitMQ" connectionString=
"Url=amqp://guest:guest@localhost:5672;InputQueue=Revo" /></connectionStrings>
```

Using this integration, the application is able to deliver events to other services connected to the same exchange and also receive the events published by those services to them. Further-more, it also makes it possible to offload some of the command and query handling to exter-nal services. As long as the system is configured to route a command or query type to an external exchange, it will always prefer the external transport over the use of local command handler \(if there are registered any\).

The use of this integrations is especially useful when building application using a micro-service architecture. For queries, it is also possible to use worker queues, effectively scaling the load of their processing to multiple query service instances.



