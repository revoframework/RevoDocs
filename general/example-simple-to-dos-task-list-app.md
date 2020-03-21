---
description: >-
  This quick guide walks you through recreating the simple TO-DOs (task list)
  app.
---

# Example: Task list app

**Revo.Examples.Todos** is a simple application intended as an introduction to Revo framework. It showcases some of its most basic features including DDD-style event sourced aggregates and entities, commands and queries, projections and read models.

Using this simple application, one should be able to track his tasks to do. User can create task lists, to which tasks \(to-dos\) can be added \(and later modified, deleted or marked as 'done'\).

{% hint style="success" %}
You can also instantly instantly download the complete application by cloning the Github repository: [https://github.com/revoframework/Revo/tree/develop/Examples/Todos](https://github.com/revoframework/Revo/tree/develop/Examples/Todos)
{% endhint %}

## 1. Create a new ASP.NET Core application in Visual Studio

Open Visual Studio or any other compatible IDE and create a new project targeting ASP.NET Core 3.0 or newer and add to it references to NuGet packages _Revo.Infrastructure_, _Revo.EFCore_ and _Revo.AspNetCore_.

{% hint style="info" %}
We are going to write the application using **EF Core**, **ASP.NET Core** and either **PostgreSQL**, **MSSQL** or **SQLite** database \(your choice\), but it is also possible to easily adapt the example to other platform or database system with minor modifications.
{% endhint %}

Open the generated Startup.cs file with your ASP.NET Core's `Startup` class and modify it so that it inherits from `RevoStartup`. This adds a light-weight support for the ASP.NET Core platform and bootstraps the framework application.

```csharp
public class Startup : RevoStartup
{
    public Startup(IConfiguration configuration) : base(configuration)
    {
    }

    /*** CODE OMITTED FOR BREVITY ***/

    protected override IRevoConfiguration CreateRevoConfiguration()
    {
        return new RevoConfiguration()
            .UseAspNetCore()
            .UseEFCoreDataAccess(
                contextBuilder => contextBuilder
                    //.UseSqlite("Data Source=todos.db"), // for real applications, you'll want to switch to more featured RDBMS as shown below.
                     .UseNpgsql(connectionString) // for PostgreSQL
                    // .UseSqlServer(connectionString) // for SQL Server you will also need to comment out SnakeCaseColumnNamesConvention and LowerCaseConventionbelow
                advancedAction: config =>
                {
                    config
                        .AddConvention<BaseTypeAttributeConvention>(-200)
                        .AddConvention<IdColumnsPrefixedWithTableNameConvention>(-110)
                        .AddConvention<PrefixConvention>(-9)
                        .AddConvention<SnakeCaseTableNamesConvention>(1)
                        .AddConvention<SnakeCaseColumnNamesConvention>(1)
                        .AddConvention<LowerCaseConvention>(2);
                })
            .UseAllEFCoreInfrastructure();
    }
}
```

You also have to implement a method called `CreateRevoConfiguration()` which configures the framework. This is the place to modify many of configuration options the framework offers.

You also have to uncomment the correct line \(15 - 17\) depending on what database system you decided to use \(PostgreSQL is recommended, but you can also start off with SQLite, for example\). 

## 2. Define domain model

First, we are going to define the domain model for our application. For the sake of simplicity, we are going to have only one aggregate root, which is going to be event-sourced \(_Revo_ also supports non-event-sourced aggregates and allows you to mix them in your domain models\).

{% hint style="success" %}
If you are feeling uncertain with the terminology used in this guide, I definitely recommend reading up on topics like _domain-driven design_ \(DDD\) or _event sourcing_ elsewhere first, as explaining these concepts is greatly beyond the scope of this documentation. Great book covering many practical aspects of these topics is [_Implementing Domain-Driven Design_](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577) by Vaughn Vernon \(2013\), for example.
{% endhint %}

### 2.1. Aggregate and entities

Our only aggregate is going to be a task \(to-do\) list, which represents a list \(e.g. a sticky note\) to which individual tasks can be added. Each task list can also have its name. The task list entity \(`TodoList`\), which is also the aggregate root, represents an entry point to interacting with the aggregate.

```csharp
[DomainClassId("9D1C248D-A389-41CC-A93D-3419D7F1CA37")]
public class TodoList : EventSourcedAggregateRoot
{
    private Dictionary<Guid, Todo> todos = new Dictionary<Guid, Todo>();

    public TodoList(Guid id, string name) : base(id)
    {
        Rename(name);
    }

    protected TodoList(Guid id) : base(id)
    {
    }

    public string Name { get; private set; }
    public IReadOnlyCollection<Todo> Todos => todos.Values;

    public Todo AddTodo(string text)
    {
        Guid todoId = Guid.NewGuid();
        Publish(new TodoAddedEvent(todoId));

        var todo = todos[todoId];
        todo.UpdateText(text);

        return todo;
    }

    public void Rename(string name)
    {
        if (Name != name)
        {
            Publish(new TodoListRenamedEvent(name));
        }
    }

    private void Apply(TodoAddedEvent ev)
    {
        todos[ev.TodoId] = new Todo(ev.TodoId, EventRouter);
    }

    private void Apply(TodoListRenamedEvent ev)
    {
        Name = ev.Name;
    }
}
```

As you can see, we are only modifying the state of the aggregate using events, so that these modifications can later be saved in form of a sequence of events and then the state later again reloaded from these events.

The aggregate root defines one public constructor with parameters \(for ensuring class invariants\) and one protected constructor that takes just the aggregate ID - this one is needed for the framework to be able to load from the event store.

{% hint style="info" %}
Note that we also defined a class ID of the aggregate root using the `[DomainClassId]` attribute. This is an arbitrary GUID value \(that must however be unique in your project\) and is needed by Revo to identify the class when saving the aggregate.
{% endhint %}

We also need to define the entity representing a task that can be added to our list.

```csharp
[DomainClassId("D8A1F0C6-CD0A-4F66-8181-336AAFE11248")]
public class Todo : EventSourcedEntity
{
    public Todo(Guid id, IAggregateEventRouter eventRouter) : base(id, eventRouter)
    {
    }

    public bool IsComplete { get; private set; }
    public string Text { get; private set; }

    public void UpdateText(string text)
    {
        if (Text != text)
        {
            Publish(new TodoTextUpdatedEvent(Id, text));
        }
    }

    public void MarkComplete(bool isComplete)
    {
        if (IsComplete != isComplete)
        {
            Publish(new TodoIsCompleteUpdatedEvent(Id, isComplete));
        }
    }

    private void Apply(TodoTextUpdatedEvent ev)
    {
        if (ev.TodoId == Id)
        {
            Text = ev.Text;
        }
    }

    private void Apply(TodoIsCompleteUpdatedEvent ev)
    {
        if (ev.TodoId == Id)
        {
            IsComplete = ev.IsComplete;
        }
    }
}
```

### 2.2 Domain events

We wouldn't be complete without the events. Note that their state is defined as immutable \(good practice\).

{% tabs %}
{% tab title="TodoListRenamedEvent.cs" %}
```csharp
public class TodoListRenamedEvent : DomainAggregateEvent
{
    public TodoListRenamedEvent(string name)
    {
        Name = name;
    }

    public string Name { get; }
}
```
{% endtab %}

{% tab title="TodoAddedEvent.cs" %}
```csharp
public class TodoAddedEvent : DomainAggregateEvent
{
    public TodoAddedEvent(Guid todoId)
    {
        TodoId = todoId;
    }

    public Guid TodoId { get; }
}
```
{% endtab %}

{% tab title="TodoTextUpdatedEvent.cs" %}
```csharp
public class TodoTextUpdatedEvent : DomainAggregateEvent
{
    public TodoTextUpdatedEvent(Guid todoId, string text)
    {
        TodoId = todoId;
        Text = text;
    }

    public Guid TodoId { get; }
    public string Text { get; }
}
```
{% endtab %}

{% tab title="TodoIsCompleteUpdatedEvent.cs" %}
```csharp
public class TodoIsCompleteUpdatedEvent : DomainAggregateEvent
{
    public TodoIsCompleteUpdatedEvent(Guid todoId, bool isComplete)
    {
        TodoId = todoId;
        IsComplete = isComplete;
    }

    public Guid TodoId { get; }
    public bool IsComplete { get; }
}
```
{% endtab %}
{% endtabs %}

## 3. Writing data with commands

### 3.1. Commands

Next, we are going to implement the write-side of our application, enabling us to create and modify tasks and tasks lists. Let's start with command classes.

{% tabs %}
{% tab title="CreateTodoListCommand.cs" %}
```csharp
public class CreateTodoListCommand : ICommand
{
    public CreateTodoListCommand(Guid id, string name)
    {
        Id = id;
        Name = name;
    }

    public Guid Id { get; }

    [Required]
    public string Name { get; }
}
```
{% endtab %}

{% tab title="UpdateTodoListCommand.cs" %}
```csharp
public class UpdateTodoListCommand : ICommand
{
    public UpdateTodoListCommand(Guid id, string name)
    {
        Id = id;
        Name = name;
    }

    public Guid Id { get; }

    [Required]
    public string Name { get; }
}
```
{% endtab %}

{% tab title="AddTodoCommand.cs" %}
```csharp
public class AddTodoCommand : ICommand
{
    public AddTodoCommand(Guid todoListId, string text)
    {
        TodoListId = todoListId;
        Text = text;
    }

    public Guid TodoListId { get; }

    [Required]
    public string Text { get; }
}
```
{% endtab %}

{% tab title="UpdateTodoCommand.cs" %}
```csharp
public class UpdateTodoCommand : ICommand
{
    public UpdateTodoCommand(Guid todoListId, Guid todoId, bool isComplete,
        string text)
    {
        TodoListId = todoListId;
        TodoId = todoId;
        IsComplete = isComplete;
        Text = text;
    }

    public Guid TodoListId { get; }
    public Guid TodoId { get; }
    public bool IsComplete { get; }
    public string Text { get; }
}
```
{% endtab %}
{% endtabs %}

A command can be any POCO class that implements the `ICommand` interface. Same as with events, we make its properties immutable. A single command always represents one write operation. Its scope can vary greatly depending on the needs of your consumers \(here a future REST API that we are going to write\), but it is usually a good practice that one command should always _modify just one aggregate_.

In Revo, however, this is _just a recommendation,_ not a requirement, and the framework doesn't limit you in what you do in your command handlers \(you don't even need to work with any aggregates, for example\).

{% hint style="info" %}
This concept of strictly segregating responsibilities of reading and writing \(query handlers and command handlers\) that Revo uses is called _CQRS_ \(command-query responsibility segregation\). Related concept _CQS_ \(command-query separation\) then means \(simply put\) that and operation always either returns data or modifies the data, not both \(also _asking a question should not change the answer_\).
{% endhint %}

We can also see we annotated some of command properties with the `[Required]` validation attribute, which ensures that only commands with non-empty data can get passed to the command handlers.

### 3.2. Command handler

Now we need to implement the actual code that gets executed when our commands get send to the commands bus - we do that by defining a class implementing `ICommandHandler<>` interfaces. By default, these handlers get auto-discovered and registered upon application startup, so it is enough to just define the class.

```csharp
public class TodoListCommandHandler :
    ICommandHandler<AddTodoCommand>,
    ICommandHandler<CreateTodoListCommand>,
    ICommandHandler<UpdateTodoListCommand>,
    ICommandHandler<UpdateTodoCommand>
{
    private readonly IRepository repository;

    public TodoListCommandHandler(IRepository repository)
    {
        this.repository = repository;
    }

    public async Task HandleAsync(AddTodoCommand command, CancellationToken cancellationToken)
    {
        var todoList = await repository.GetAsync<TodoList>(command.TodoListId);
        todoList.AddTodo(command.Text);
    }

    public Task HandleAsync(CreateTodoListCommand command, CancellationToken cancellationToken)
    {
        var todoList = new TodoList(command.Id, command.Name);
        repository.Add(todoList);

        return Task.CompletedTask;
    }

    public async Task HandleAsync(UpdateTodoListCommand command, CancellationToken cancellationToken)
    {
        var todoList = await repository.GetAsync<TodoList>(command.Id);
        todoList.Rename(command.Name);
    }

    public async Task HandleAsync(UpdateTodoCommand command, CancellationToken cancellationToken)
    {
        var todoList = await repository.GetAsync<TodoList>(command.TodoListId);
        var todo = todoList.Todos.First(x => x.Id == command.TodoId);
        todo.UpdateText(command.Text);
        todo.MarkComplete(command.IsComplete);
    }
}
```

The command handler's constructor gets a reference to the repository, which is used for loading and storing domain aggregates \(note that you can only get an aggregate root from it, not just any entity\). When a command handler executes, the command handler pipeline automatically creates and commits a new unit of work \(using pipeline filters in the background\). This also means that an execution of single commands defines a _strict transactional boundary_.

{% hint style="info" %}
Because the unit of work gets automatically commited at the end of a successful command execution, you don't need to explicitly call anything to save the repository.
{% endhint %}

## 4. Querying data from read model

Because we want to display the data \(tasks and task lists\) of our application on a simple web page, we also need a way to query the data we store in the database. Since directly querying individual events from the event store would be very cumbersome in our use case \(it usually is\), we are going to define a read model for our data.

### 4.1. Read model

{% tabs %}
{% tab title="TodoListReadModel.cs" %}
```csharp
[TablePrefix(NamespacePrefix = "TODOS", ColumnPrefix = "TLI")]
public class TodoListReadModel : EntityReadModel
{
    public string Name { get; set; }
    public List<TodoReadModel> Todos { get; set; }
}
```
{% endtab %}

{% tab title="TodoReadModel.cs" %}
```csharp
[TablePrefix(NamespacePrefix = "TODOS", ColumnPrefix = "TDO")]
public class TodoReadModel : EntityReadModel
{
    public Guid TodoListId { get; set; }
    public TodoListReadModel TodoList { get; set; }
    public bool IsComplete { get; set; }
    public string Text { get; set; }
}
```
{% endtab %}
{% endtabs %}

Read model can be structured pretty much in any way we or our consumers \(e.g. an UI or REST API\) need \(even denormalized, for example\). Here, for the simplicity of this example, we are going to project the events into simple POCO classes persisted by _Entity Framework Core_ ORM \(but you can also use other, like Entity Framework 6, RavenDB document-database or your own\).

{% hint style="info" %}
`[TablePrefix]` attribute is just Revo's convenience attribute which prefixes the names of the tables and columns and you don't need to use it if you don't like it.
{% endhint %}

### 4.2. Queries

Similarly to when we defined commands to modify our data, we need to defines queries to be able to query our read model. Here, `GetTodoListsQuery` loosely corresponds to a single REST API endpoint we are going to implement.

{% tabs %}
{% tab title="GetTodoListsQuery.cs" %}
```csharp
public class GetTodoListsQuery : IQuery<IQueryable<TodoListDto>>
{
}
```
{% endtab %}

{% tab title="TodoListDto.cs" %}
```csharp
public class TodoListDto : EntityDto
{
    public string Name { get; set; }
    public List<TodoDto> Todos { get; set; }
}
```
{% endtab %}

{% tab title="TodoDto.cs" %}
```csharp
public class TodoDto : EntityDto
{
    public Guid TodoListId { get; set; }
    public bool IsComplete { get; set; }
    public string Text { get; set; }
}
```
{% endtab %}

{% tab title="DtoAutoMapperProfile.cs" %}
```csharp
public class DtoAutoMapperProfile : Profile
{
    public DtoAutoMapperProfile()
    {
        CreateMap<TodoListReadModel, TodoListDto>();
        CreateMap<TodoReadModel, TodoDto>();
    }
}
```
{% endtab %}
{% endtabs %}

Query is any class that implements the `IQuery<T>` interface, where `T` is the type of the result it returns. It can also have parameters \(properties\) like commands.

{% hint style="info" %}
Since we don't like directly returning the read model the way it is stored by the ORM \(e.g. with recursive references\), we also defines DTO \(data transfer objects\) mapped by [AutoMapper](https://automapper.org/), but this is purely optional and you don't need to do it in your code and your queries can directly return your read model.
{% endhint %}

### 4.3. Query handler

To define what a query returns once it is executed, we are going to implement a query handler. Query handlers get also auto-discovered and registered upon startup. Note that Revo doesn't restrict how you work with your read model in any way and you can store your data in any way you like.

```csharp
public class TaskListQueryHandler :
    IQueryHandler<GetTodoListsQuery, IQueryable<TodoListDto>>
{
    private readonly IReadRepository readRepository;

    public TaskListQueryHandler(IReadRepository readRepository)
    {
        this.readRepository = readRepository;
    }

    public Task<IQueryable<TodoListDto>> HandleAsync(GetTodoListsQuery query, CancellationToken cancellationToken)
    {
        IQueryable<TodoListDto> taskLists = readRepository
            .FindAll<TodoListReadModel>()
            .Include(x => x.Todos)
            .ProjectTo<TodoListDto>();
        return Task.FromResult(taskLists);
    }
}
```

In contrary to our command handler, our query handler works with `IReadRepository` \(CRUD repository\) instead of `IRepository` \(domain repository\).

While you should use the \(and only\) domain `IRepository`  in command handlers to modify your aggregates, in your query handlers, you are going to need `IReadRepository` which is just a thin read-only abstraction layer over an ORM \(Entity Framework Core here in our case\).

{% hint style="info" %}
Compared to `IReadRepository` \(which behaves just like a thin wrapper over an CRUD-like ORM\),  domain `IRepository` can also persist aggregates in other forms, e.g. event sourced aggregates to an event store. It also deals with other aspects of domain aggregates like publishing events to event bus.
{% endhint %}

### 4.4. Projector

Finally, we need to specify how our read model gets populated with the data from events emitted by the aggregates. To do so, we are going to implement a projector. To find out more about how projectors work, see [Projections](../reference-guide/projections.md) in the reference guide.

```csharp
public class TodoListReadModelProjector :
    EFCoreEntityEventToPocoProjector<TodoList, TodoListReadModel>
{
    public TodoListReadModelProjector(IEFCoreCrudRepository repository) :
        base(repository)
    {
    }

    private void Apply(IEventMessage<TodoListRenamedEvent> ev)
    {
        Target.Name = ev.Event.Name;
    }

    private void Apply(IEventMessage<TodoAddedEvent> ev)
    {
        var task = new TodoReadModel()
        {
            Id = ev.Event.TodoId,
            TodoListId = ev.Event.AggregateId
        };

        Repository.Add(task);
    }

    private async Task Apply(IEventMessage<TodoTextUpdatedEvent> ev)
    {
        var task = await Repository.FindAsync<TodoReadModel>(ev.Event.TodoId);
        task.Text = ev.Event.Text;
    }

    private async Task Apply(IEventMessage<TodoIsCompleteUpdatedEvent> ev)
    {
        var task = await Repository.FindAsync<TodoReadModel>(ev.Event.TodoId);
        task.IsComplete = ev.Event.IsComplete;
    }
}
```

## 5. ASP.NET Core controller

We have now almost all the parts for a fully functional Revo application and need just one more thing - an endpoint to send the commands and queries to our application from. We can do this by implementing an ASP.NET Core controller.

```csharp
[Route("api/todo-lists")]
public class TodoListController : CommandApiController
{
    [HttpGet("")]
    public Task<IQueryable<TodoListDto>> Get()
    {
        return CommandBus.SendAsync(new GetTodoListsQuery());
    }

    [HttpPost("")]
    public Task Post([FromBody] CreateTodoListDto payload)
    {
        return CommandBus.SendAsync(new CreateTodoListCommand(payload.Id, payload.Name));
    }

    [HttpPut("{id}")]
    public Task Put(Guid id, [FromBody] UpdateTodoListDto payload)
    {
        return CommandBus.SendAsync(new UpdateTodoListCommand(id, payload.Name));
    }

    [HttpPost("{id}")]
    public Task PostTodo(Guid id, [FromBody] AddTodoDto payload)
    {
        return CommandBus.SendAsync(new AddTodoCommand(id, payload.Text));
    }

    [HttpPut("{id}/{todoId}")]
    public Task PutTodo(Guid id, Guid todoId, [FromBody] UpdateTodoDto payload)
    {
        return CommandBus.SendAsync(new UpdateTodoCommand(id, todoId, payload.IsComplete, payload.Text));
    }

    public class CreateTodoListDto
    {
        public Guid Id { get; set; }
        public string Name { get; set; }
    }

    public class UpdateTodoListDto
    {
        public Guid Id { get; set; }
        public string Name { get; set; }
    }

    public class AddTodoDto
    {
        public string Text { get; set; }
    }

    public class UpdateTodoDto
    {
        public string Text { get; set; }
        public bool IsComplete { get; set; }
    }
}
```

We are deriving our controller from a convenience base class called `CommandApiController` which does just one thing - gets an `ICommandBus` dependency injected. A command bus can be used for sending commands and queries to an application.

## 6. Finish!

You have now implemented a complete \(albeit simple\) application using _Revo_ framework. You can either use a REST API testing tool like Postman to manually send the requests to the API controller, or you can directly grab a simple JavaScript frontend UI that is implemented in [this example's repository](https://github.com/revoframework/Revo/tree/develop/Examples/Todos) and copy it to your project, it's your choice.

Happy testing!

