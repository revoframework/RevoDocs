# Features

The framework combines the concepts of event sourcing, CQRS and DDD to provide support for building applications that are scalable, maintainable, can work in distributed environments and are easy to integrate with outside world. As such, it takes some rather opinionated approaches on the design of certain parts of its architecture. Revo also offers other common features and infrastructure that is often necessary for building complete applications – for example, database migrations, event upgrades, authorizations, validations, messaging, integrations, multi-tenancy or testing. Furthermore, its extensions implement other useful features like entity history change-tracking, auditing or user notifications.

[**Domain-Driven Design**](../reference-guide/domain-building-blocks.md)  
****Building blocks for rich DDD-style domain models \([aggregates](../reference-guide/domain-building-blocks.md#aggregates), [entities](../reference-guide/domain-building-blocks.md#entities), [value objects](../reference-guide/domain-building-blocks.md#value-objects), [domain events](../reference-guide/domain-building-blocks.md#domain-events), [repositories](../reference-guide/data-persistence.md#aggregate-repository)...\).

[**Event Sourcing**](../reference-guide/events.md)  
****Implementing event-sourced entity persistence with support for multiple event store backends \(PostgreSQL, MSSQL, SQLite...\).

[**CQRS**](../reference-guide/commands-and-queries.md)  
Segregating command and query responsibilities with:

* [Commands and queries  ](../reference-guide/commands-and-queries.md#commands-queries)
* Command/query [handlers  ](../reference-guide/commands-and-queries.md#command-query-handlers)
* Processing pipeline with filters for cross-cutting concerns \([authorization](../reference-guide/authorization.md), [validation](../reference-guide/validation.md), etc.\)
* [Different read/write models](../reference-guide/projections.md)

[**A/synchronous event processing**](../reference-guide/events.md)  
****Support for both [synchronous](../reference-guide/events.md#synchronous-event-processing) and [asynchronous](../reference-guide/events.md#asynchronous-event-processing) event processing, guaranteed _at-least-once_ delivery, event queues with strict sequence ordering \(optionally\), event source catch-ups, optional [pseudo-synchronous event dispatch](../reference-guide/events.md#pseudo-synchronous-event-dispatch) for listeners \(projectors, for example\).

[**Data access**](../reference-guide/data-persistence.md)  
**** Thin abstraction layer for easy data persistence \(e.g. querying read models\) using _Entity Framework Core_, _Entity Framework 6_, _RavenDB,_ testable _in-memory database_ or other data providers. Includes support for simple [database migrations](../reference-guide/database-migrations.md).

[**Projections**](../reference-guide/projections.md)  
**** Support for read-model projections with various backends \(e.g. _Entity Framework Core_ \(_PostgreSQL_, _MSSQL_, _SQLite_,...\), _Entity Framework 6_, _RavenDB_...\), automatic idempotency- and concurrency-handling, etc.

[**SOA, messaging and integration**](../reference-guide/integrations.md)  
****Scale and integrate by [publishing and receiving events](../reference-guide/integrations.md#rabbitmq-messaging-with-easynetq), commands and queries using common messaging patterns, e.g. with _RabbitMQ_ message queue \(using _EasyNetQ_ connector or _Rebus_ service bus\).

[**Sagas**](../reference-guide/sagas.md)  
****Coordinating long-running processes or inter-aggregate cooperation with sagas that react to events \(a.k.a. _process managers_\).

[**Authorization**](../reference-guide/authorization.md)  
****Basic permission/role-based ACL for commands and queries, fine-grained row filtering.

**Other minor features:**

* [**Validation**](../reference-guide/validation.md) for commands, queries and other structures
* [**Jobs**](../reference-guide/jobs.md)
* [**Multi-tenancy**](../reference-guide/multi-tenancy.md)
* [**Event message metadata**](../reference-guide/events.md#event-messages-and-metadata)
* [**Event versioning**  ](../reference-guide/events.md#event-versioning)\*\*\*\*
* \*\*\*\*[**Event upgrades**](../reference-guide/events.md#event-upgrades)
* [**Database migrations**](../reference-guide/database-migrations.md)\*\*\*\*
* **History and change-tracking**
* **User notifications:** event-based, with different output channels \(mail, etc.\), aggregation, buffering, etc.
* **.NET Core 3.0+, .NET Standard 2.0+ & .NET 4.7.2+ support \(with integration for ASP.NET Core and ASP.NET\)**

