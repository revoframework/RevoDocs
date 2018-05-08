# Data persistence

## Repositories

Revo framework offers a number of facilities for working with entities and their persistence. Repositories are one of the core concepts in framework’s data persistence. There are two distinct types of repositories that a developer can use. Both of them come with a certain level of implementation abstraction.

### Aggregate repository

In terms of domain-driven design, `IRepository` \(defined in Revo.Infrastructure module\) is the high-level repository that would be used as a part of the domain for the write side of the application working with the domain model. The implementation of `IRepository` interface is not bound to any specific database system backend \(or an ORM library\) and instead delegates most of the actual data-persistence related responsibilities to a number of _aggregate stores_. The repository itself is then just a thin wrapper over those aggregate stores, making it easier to work with aggregate stored in different persistence backends by providing a single unified API for them while also handing some of the concepts related to domain repositories – like the unit-of-work pattern and event publishing. The actual definition of the interface \(which is listed below\) is kept to a bare minimum and offers only very basic features that are crucially needed for loading and saving aggregates, abstracting from the underlying database technology. This is in contrast with CRUD data repositories which on the other hand try to offer maximum flexibility when working with data, putting a minimum barrier between developer and the database \(see following subsection on CRUD repositories\).

The repository takes full advantage of .NET support of interface generic methods and declares all methods as such, simplifying all repository interactions to just one universal interface. When querying for an entity, the repository itself then tries to deduce the aggregate store it belongs \(would belong to\). The actual detection logic is implemented by the aggregate stores. When the correct aggregate store is found, the repository delegates the actual persistence work \(loading, saving, querying…\) to it.

To enforce aggregate consistency boundaries on compliance with DDD rules, repositories only allow working with aggregate roots, which is enforced by generic type constraints for all its generic methods \(requiring any `IAggregateRoot` descendant\). This prevents breaking the encapsulation of aggregates by querying and manually modifying single aggregate entities separately.

{% hint style="success" %}
Because of its generic nature, it is not a true repository that would be a part of the domain according to DDD as discussed previously \(i.e. with a meaning pertaining to a specific domain part, defining its queries for specific aggregates in ubiquitous language\). Rather, it is a service wrapping some of the complexities behind the persistence of aggregates and entities, providing a reasonable amount of universal abstraction over different persistence mechanism and entity handling \(e.g. publishing events to event bus\). However, if one wanted to create a repository in a DDD-fashion \(with custom queries for specific aggregates, etc.\), he should be able to use this universal `IRepository` as a base for its implementation.
{% endhint %}

Following  abridged code snippet of `IRepository` interface definition shows some of the basic functionality it offers for working with aggregates.

```csharp
public interface IRepository : IUnitOfWorkProvider, IDisposable
{
	void Add<T>(T aggregate) where T : class, IAggregateRoot;
	T FirstOrDefault<T>(Expression<Func<T, bool>> predicate) where T : class, 
        IAggregateRoot, IQueryableEntity;
	T First<T>(Expression<Func<T, bool>> predicate)
        where T : class, IAggregateRoot, IQueryableEntity;
	Task<T> FirstOrDefaultAsync<T>(Expression<Func<T, bool>> predicate)
        where T : class, IAggregateRoot, IQueryableEntity;
	Task<T> FirstAsync<T>(Expression<Func<T, bool>> predicate)
        where T : class, IAggregateRoot, IQueryableEntity;
	T Find<T>(Guid id) where T : class, IAggregateRoot;
	Task<T> FindAsync<T>(Guid id) where T : class, IAggregateRoot;
	IQueryable<T> FindAll<T>()
        where T : class, IAggregateRoot, IQueryableEntity;
	Task<IList<T>> FindAllAsync<T>()
        where T : class, IAggregateRoot, IQueryableEntity;
	T Get<T>(Guid id) where T : class, IAggregateRoot;
	Task<T> GetAsync<T>(Guid id) where T : class, IAggregateRoot;
	IQueryable<T> Where<T>(Expression<Func<T, bool>> predicate)
        where T : class, IAggregateRoot, IQueryableEntity;
	void Remove<T>(T aggregate) where T : class, IAggregateRoot;
	void SaveChanges();
	Task SaveChangesAsync();
}
```

Besides the common functionality of adding and removing aggregate roots and finding them by their ID, the repository also provides method for finding them using more advanced queries. These methods are, however, defined with additional generic method constraint and can only be used with entities that implement the `IQueryableEntity` interface \(this interface is empty and serves just to signify what entities can be used in those queries\). For example, this means while these methods will be available for aggregate roots stored in a RDBMS and accessed via Entity Framework \(generally any `BasicAggregateRoot`-derived aggregate roots\), they will not be available for event sourced aggregate roots because event store usually do not possess these kinds of querying features.

Aggregate changes are automatically persisted upon saving the repository. It is unnecessary to call the save method explicitly as the repository implements a unit of work provider, which means it gets automatically saved when the unit of work is committed at the end of the command processing \(unless any unhandled exceptions are thrown\). The mechanism for detecting changes in the entities depends on the actual aggregate store implementation \(e.g. event sourced entities are saved when they published any new uncommitted events, while changes in entities persisted by Entity Framework rely on its own internal change tracking mechanism\).

There are currently two aggregate stores implemented. The developer does need to directly manipulate with the aggregate stores, but he needs to know their underlying technology.

#### CRUD aggregate store

CRUD aggregate store is not tied to single specific database technology and is just a thin layer on top of ICrudRepository. CRUD data repository is the second type of repositories providing a more direct access to databases without the constraints of the domain and is in-depth discussed in chapter 5.5. All the persistence-specific features of their implementations \(e.g. the mapping of entities to database\) also apply to aggregates accessed via this aggregate store.

#### Event sourced aggregate store

Similar to the CRUD aggregate store, the event sourced aggregate store is not implemented using a single specific database technology and is internally using IEventStore which is the source of the actual database connector implementation. 

### CRUD data repository

CRUD data repositories \(implementations of `ICrudRepository` which is contained in Revo.DataAccess module\) represent a way of a more direct way of accessing database. Unlike the aggregate repository \(`IRepository`\), it is not burdened with domain concepts like aggregate consistency boundaries and encapsulation and provides more flexibility when working with data. Where IRepository offered only basic functionality for working with entities to abstract from the underlying technology, `ICrudRepository` strives to do the opposite and offer maximum of the features that the used database \(or an ORM\) has while still maintaining some minimum level of abstraction in order to enable easy testing. Having said that, it is important to emphasize that they serve a completely different purpose – while the aggregate IRepository would usually be used on the write side of the application in command handlers to update the domain, the CRUD repository will mostly be used just for efficient access to read models in projectors and query handlers and in other places that are not encumbered by the complex business rules handled by the domain model. 

{% hint style="danger" %}
Under no circumstances it should be used to modify domain model data bypassing its business rules. However, in some scenarios, when it is unnecessary to maintain different write model and read model in the database it might be possible to reuse the domain model for queries \(this is supported by the EF6 implementation, see chapter 5.5\).
{% endhint %}

The `ICrudRepository` interfaces relies heavily on the use of `IQueryable<T>` support of the .NET platform. This enables to perform arbitrary queries on the database without bloating the interface definition with a huge number of different query methods or a need for custom repositories for every specific, saving a lot of developer’s time, while still also providing type safety and the safety of compile-time checks to a certain degree.

Most of the read-related functionality of `ICrudRepository` is actually defined in a base type named `IReadRepository`. This base type does not allow any modifications to the entities \(i.e. it has no save or add/remove methods\), which means it is very suitable in situations when we want to limit the operations an object can do with entities in the repository – for example in a query handler that should always only performs reads of the database. It also potentially makes testing those entities easier.

Most of the CRUD repository implementations will also define its own repository interfaces derived from `ICrudRepository` in order to facilitate the use of the features specific to its technology \(for example, `IEF6CrudRepository` making it possible to directly work with some of the Entity Framework ORM features\).

Both types of repositories also offer an in-memory implementation of their interfaces that are suitable for testing purposes. For more information about this, see chapter 7.15.

## Repository filters

All repositories in Revo framework also support the concept of repository filters. Repository filters allow decorating of the behavior of repositories with cross-cutting concerns like authorization or multi-tenancy. These filters can control what entities are queried and saved by the repository. Repository filters can be registered to be implicitly enabled for all new repositories. By default, Revo currently registers two repository filters:

* tenant repository filter \(see [multi-tenancy support](multi-tenancy.md)\),
* authorization repository filter \(see [how to implement authorization](authorization.md)\).

CRUD data persistence using EF6/MSSQL

First data persistence backend \(usable for both non-event-sourced aggregates with IRepository and any other CRUD data access with `ICrudRepository`\) is implemented using Entity Framework 6 object-relational mapper \(also abbreviated as EF6\). EF6 provides out-of-box support for MSSQL databases and a number of other database systems when using third-party database providers . Revo framework primarily supports code-first mapping of entities using C\# attributes, mapping all entities annotated with DatabaseEntityAttribute. Besides Entity Framework’s default naming conventions, it is also possible to annotate entity class with `TablePrefixAttribute` specifying a prefix for SQL table name and a common common prefix for all of its column names. In case these automatic mapping features were not enough, the framework also scans all referenced assemblies from class implementing IModelDefinition making it easily possible to customize the mapping upon the creation of the internal DbContext.

Revo.DataAccess.EF6 also defines an `IEF6CrudRepository` interface, which extends the `ICrudRepository` and adds facilities for working with some of the EF6-specific features like working with ORM entries of entities, their change tracking, or delayed loading of collections.

## Event stores

### MSSQL event store

The framework default event store implementation is based on MSSQL database that is internally accessed using Entity Framework 6. It stores individual events as table rows. To implement optimistic concurrency, the database schema enforces that events have unique sequence numbers in their event stream.



