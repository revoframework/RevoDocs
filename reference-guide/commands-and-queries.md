---
description: >-
  The framework implements a number of facilities for working with commands,
  queries and for implementing CQRS.
---

# Commands and queries

## Commands, queries

Commands and queries can be defined as regular POCO classes implementing `ICommand` or `IQuery<T>` \(with `T` defining the query return type\) interfaces. These interfaces are empty on their own and only define the contract of being processable by a command bus.

{% hint style="info" %}
Because both `IQuery` and `ICommand` derive from a common `ICommandBase` ancestor, the framework considers queries to be simply a specific subtype of commands that happen to also return a value.
{% endhint %}

 Example command:

```csharp
public class CreateUserCommand : ICommand
{
    public CreateUserCommand(string firstName, string lastName,
      string emailAddress, string password)
    {
        FirstName = firstName;
        LastName = lastName;
        EmailAddress = emailAddress;
        Password = password;
    }

    public string FirstName { get; }
    public string LastName { get; }
    public string EmailAddress { get; }
    public string Password { get; }
}
```

 Example query:

```csharp
public class GetAllUsersQuery : IQuery<List<UserDto>>
{
    public GetAllUsersQuery(string filterFirstName, string filterLastName)
    {
        FilterFirstName = filterFirstName;
        FilterLastName = filterLastName;
    }

    public string FilterFirstName { get; }
    public string FilterLastName { get; }
}
```

{% hint style="success" %}
It is advisable to make the command/query classes immutable \(as can be seen in the example\) to prevent undesirable modifications as the object gets passed throughout the system.
{% endhint %}

## Command/query handlers

Commands and queries can be handled by implementing `ICommandHandler<TCommand>` or `IQueryHandler<TQuery, TResult>` respectively. For every command or query type, there should be exactly one handler type registered in the dependency container, otherwise an exception will be thrown when trying to handle it.

```csharp
public interface ICommandHandler<in TCommand>
	 where TCommand : ICommand
{
	Task HandleAsync(T command, CancellationToken cancellationToken);
}

public interface ICommandHandler<in TCommand, TResult>
	where TCommand: ICommand<TResult>
{
	Task<TResult> HandleAsync(TCommand query, CancellationToken cancellationToken);
}

public interface IQueryHandler<TQuery, TResult>
    : ICommandHandler<TQuery, TResult>
	where TQuery : IQuery<TResult>
{
}
```

## Command bus

To send a command or query to the system, one can use an `ICommandBus` which encapsulates most of the command processing details.

```csharp
public interface ICommandBus
{
	Task<TResult> SendAsync<TResult>(ICommand<TResult> command,
		CancellationToken cancellationToken = default(CancellationToken));
	Task SendAsync(ICommandBase command,
        CancellationToken cancellationToken = default(CancellationToken));
}
```

The command bus is designed to accept any command or query type an resolves the corresponding handlers during runtime. By default, the command handling also starts a new unit of work that is automatically committed at the end of the handling pipeline – to find more about this, see chapter describing the [request life-cycle](request-life-cycle.md).

### Command filters

Command filters provide a way for dealing with cross-cutting concerns when handling their execution. It is possible to define action that will get executed before invoking a command handler, after invoking it and after invoking it in case it results in an error exception.

```csharp
public interface IPreCommandFilter<in T>
	where T : ICommandBase
{
	Task PreFilterAsync(T command);
}

public interface IPostCommandFilter<in T>
    where T : ICommandBase
{
	Task PostFilterAsync(T command, object result);
}

public interface IExceptionCommandFilter<in T>
    where T : ICommandBase
{
	Task FilterExceptionAsync(T command, Exception e);
}
```

These filters get automatically invoked when registered in the dependency container for a specific command \(base\) type. The framework uses them to deal with concerns like authorization or automatic unit-of-work management, but it is also possible to define custom application’s own filters.

