# Overview

Revo is an application framework for modern C\#/.NET applications built with _event sourcing_, _CQRS_ and _DDD_.

## Features

The framework combines the concepts of event sourcing, CQRS and DDD to provide support for building applications that are scalable, maintainable, can work in distributed environments and are easy to integrate with outside world. As such, it takes some rather opinionated approaches on the design of certain parts of its architecture. Revo also offers other common features and infrastructure that is often necessary for building complete applications – for example, authorizations, validations, messaging, integrations, multi-tenancy or testing. Furthermore, its extensions implement other useful features like entity history change-tracking, auditing or user notifications.

[**Domain-Driven Design**](../reference-guide/domain-building-blocks.md)**  
**Building blocks for rich DDD-style domain models \(aggregates, entities, domain events, repositories...\).

[**Event Sourcing**](../reference-guide/events.md)**  
**Implementing event-sourced entity persistence with multiple backends \(just _MSSQL_ for now\).

[**CQRS**](../reference-guide/commands-and-queries.md)  
Segregating command and query responsibilities with:

* [Commands and queries  ](../reference-guide/commands-and-queries.md#commands-queries)
* Command/query [handlers  ](../reference-guide/commands-and-queries.md#command-query-handlers)
* Processing pipeline with filters for cross-cutting concerns \([authorization](../reference-guide/authorization.md), [validation](../reference-guide/validation.md), etc.\)
* [Different read/write models](../reference-guide/projections.md)

[**A/synchronous event delivery**](../reference-guide/events.md)**  
**Support for both [synchronous](../reference-guide/events.md#synchronous-event-processing) and [asynchronous](../reference-guide/events.md#asynchronous-event-processing) event delivery, guaranteed _at-least-once_ delivery, event queues with strict sequence ordering \(optionally\), event source catch-ups, optional [pseudo-synchronous event dispatch](../reference-guide/events.md#pseudo-synchronous-event-dispatch) for listeners \(projectors, for example\).

[**Data access**](../reference-guide/data-persistence.md)**  
**Abstraction layer for _Entity Framework 6_, _RavenDB,_ testable _in-memory database_ or other data providers.

[**Projections**](../reference-guide/projections.md)**  
**Support for read-model projections with various backends** **\(e.g. _MSSQL_/_Entity Framework 6_, _RavenDB_...\), automatic idempotency- and concurrency-handling, etc.

[**SOA and integration**](../reference-guide/integrations.md)**  
**Scale and integrate by publishing and receiving events, commands and queries using common messaging patterns, e.g. with _RabbitMQ_ message queue and/or uses _Rebus_ service bus.

[**Sagas**](../reference-guide/sagas.md)**  
**Coordinating long-running processes or inter-aggregate cooperation with sagas that react to events \(a.k.a. _process managers_\).

[**Authorization**](../reference-guide/authorization.md)**  
**Basic permission/role-based ACL for commands and queries, fine-grained row filtering.

**Other minor features:  
**	[**Validation**](../reference-guide/validation.md) for commands, queries and other structures  
	[**Jobs**](../reference-guide/jobs.md)  
	[**Multi-tenancy**](../reference-guide/multi-tenancy.md)**  
**[	**Event message metadata**](../reference-guide/events.md#event-messages-and-metadata)**  
**	[**Event versioning  
**](../reference-guide/events.md#event-versioning)	**History and change-tracking  
**	**User notifications:** event-based, with mail/APNS/FCM output channels, supporting aggregation, etc.  
	**ASP.NET support** \(ASP.NET Core coming soon...\)

## Quick look at the design

Following diagram depicts the data flows during a typical processing of a request in the Revo framework. As such, it can also provide a broder insight into the the design and architecture of the framework.

![Data flows during a typical request processing in Revo.](../.gitbook/assets/revo_request_processing_data_flows-3%20%282%29.png)

To learn more about the request processing in Revo, see the [related chapter](../reference-guide/request-life-cycle.md).

## Requirements

The framework is written in C\# 7.1 and targets the .NET Standard 2.0 platform specification; some of its modules currently also require the .NET Framework 4.7.1, where needed \(e.g. Entity Framework 6 support\). Revo also makes a heavy use of the C\# async/await pattern and uses the TAP \(Task Asynchronous Pattern\) throughout its entire codebase \(i.e. _async all the way_ approach\).

## Project structure

The framework codebase is split into a number of submodules \(Visual Studio projects\), making it possible to only load what the developer needs. A listing of Visual Studio solution structure with description of individual submodule follows:

* **Core**
  * **Revo.Core **Implements framework’s core features like events or commands and defines basic struc-tures and interfaces for things like the unit of work pattern, security permissions and other utilities.
  * **Revo.Domain **Defines most of the framework’s building blocks for an application domain model – i.e. aggregates, entities, sagas, etc.
  * **Revo.Testing** Provides several useful facilities for easier application testing – e.g. assertions for event-sourced aggregates or fake repositories \(refer to the [Testing](../reference-guide/testing.md) chapter\).
  * **DataAccess**
    * **Revo.DataAccess** Lower-level data access layer that provides interfaces and other abstractions for working with databases. Not dependent on domain concepts \(aggregates, con-sistency boundaries, etc.\).
    * **Revo.DataAcess.EF6       **Data access layer implementation using Entity Framework 6. Provides also some facilities for easier entity mapping.
    * **Revo.DataAccess.Raven **Data access layer implementation using RavenDB database system.
  * **Extensions**
    * **Revo.Extensions.AspNet.Interop      **
    * **Revo.Extensions.History **Provides additional support for history-like features, e.g. entity change-tracking or auditing.
    * **Revo.Extensions.Notifications **Configurable event-based user notifications with output channels for email, APNS and FCM \(push notifications\) and features like notification aggregations.
  * **Infrastructure**
    * **Revo.Infrastructure **Interfaces for working with application domain and related aspects of an applica-tions – e.g. aggregate repositories, event stores, projections, jobs, asynchronous event queues, etc.
    * **Revo.Infrastructure.EF6 **Default infrastructure implementation using Entity Framework 6.
  * **Integrations**
    *  **Revo.Integrations.Rebus ** Rebus integration for command/query and event messaging with RabbitMQ.
  * **Platforms**
    * **Revo.Platforms.AspNet **Module providing support for ASP.NET applications – application life cycle hooks, user context, authentication, etc.
    * **Revo.Platforms.AspNet.Bootstrap**
* **Tests **Folder containing tests for all core submodules \(split in separate projects in the same structure\).
* **Examples**
  * **Revo.Examples.HelloAspNet.Bootstrap **Primitive “Hello World” sample application demonstrating some of the basic features of the framework.

## Dependency injection

Revo also builds on the principles inversion of control \(IoC\) uses the open-source [Ninject](http://www.ninject.org/) dependency injector. Upon the start of an ASP.NET application, the framework automatically locates and loads Ninject module definitions from all referenced assemblies.

{% hint style="info" %}
While it is possible to use the framework with other dependency container libraries, it would currently require a reimplementation of some internal dependency bindings using the specific container library, and is not very practical because of that.  
There are plans to remove this Ninject dependency and replace it with a more flexible system allowing to configure any other DI mechanisms, however.
{% endhint %}



