---
description: So how does it actually look when writing an application with Revo?
---

# Super-short example

These few following paragraphs show how could one design super-simple application that can save tasks using event-sourced aggregates and then query them back from a RDBMS.

## Event

The event that happens when changing a task's name.

```csharp
public class TodoRenamedEvent : DomainAggregateEvent
{
    public TodoRenamedEvent(string name)
    {
        Name = name;
    }

    public string Name { get; }
}
```

## Aggregate

The task aggregate root.

```csharp
public class Todo : EventSourcedAggregateRoot
{
    public Todo(Guid id, string name) : base(id)
    {
        Rename(name);
    }
    
    protectedTodo(Guid id) : base(id)
    {
    }

    public string Name { get; private set; }

    public void Rename(string name)
    {
        if (!Name != name)
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

## Command and command handler

Command to save a new task.

```csharp
public class CreateTodoCommand : ICommand
{
    public CreateTodoCommand(string name)
    {
        Name = name;
    }

    [Required]
    public string Name { get; }
}
```

```csharp
public class TodoCommandHandler : ICommandHandler<CreateTodoCommand>
{
    private readonly IRepository repository;
    
    public TodoCommandHandler(IRepository repository)
    {
        this.repository = repository;
    }

    public Task HandleAsync(CreateTodoCommand command, CancellationToken cancellationToken)
    {
        var todo = new Todo(command.Id);
        todo.Rename(command.Name);
        repository.Add(todoList);
        return Task.CompletedTask;
    }   
}
```

## Read model and projection

Read model and a projection for the event-sourced aggregate.

```csharp
public class TodoReadModel : EntityReadModel
{
    public string Name { get; set; }
}
```

```csharp
public class TodoListReadModelProjector : EFCoreEntityEventToPocoProjector<Todo, TodoReadModel>
{
    public TodoListReadModelProjector(IEFCoreCrudRepository repository) : base(repository)
    {
    }

    private void Apply(IEventMessage<TodoRenamedEvent> ev)
    {
        Target.Name = ev.Event.Name;
    }
}
```

## Query and query handler

Query to read the tasks back from a RDBMS.

```csharp
public class GetTodosQuery : IQuery<IQueryable<TodoReadModel>>
{
}
```

```csharp
public class TaskQueryHandler : IQueryHandler<GetTodoQuery, IQueryable<TodoReadModel>>
{
    private readonly IReadRepository readRepository;

    public TaskListQueryHandler(IReadRepository readRepository)
    {
        this.readRepository = readRepository;
    }

    public Task<IQueryable<TodoReadModel>> HandleAsync(GetTodoListsQuery query, CancellationToken cancellationToken)
    {
        return Task.FromResult(readRepository
            .FindAll<TodoListReadModel>());
    }
}
```

## ASP.NET Core controller

Or just any arbitrary endpoint from where to send the command and queries from. :\)

```csharp
[Route("todos")]
public class TodoController : CommandApiController
{
    [HttpGet("")]
    public Task<IQueryable<TodoReadModel>> Get()
    {
        return CommandBus.SendAsync(new GetTodosQuery());
    }

    [HttpPost("")]
    public Task Post([FromBody] CreateTodoDto payload)
    {
        return CommandBus.SendAsync(new CreateTodoCommand(payload.Name));
    }
    
    public class CreateTodoDto
    {
        public string Name { get; set; }
    }
}
```

## Finish!

Now you are ready save the TO-DOs to an event store and read them from regular RDBMS read models.

