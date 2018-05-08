# Authorization

Authorization is one of the most common cross-cutting concerns that most applications need to deal with. Revo framework addresses this by providing a number of facilities for it.

## Permissions

### Defining permissions

The framework works with a simple and flexible concept of permissions. A permission defines the authorization to perform a certain action, optionally \(if specified\) on a specified resource or/and in a specified context. The types of these actions \(e.g. “edit blog post”\) are defined as `PermissionType`s in so called permission catalogs and are identified by their name and a GUID:

```csharp
[PermissionTypeCatalog("Revo.Infrastructure.Notifications.Channels.Apns")]
public static class Permissions
{
	public const string RegisterDeviceToken =
        "{EAA3FA48-1227-4479-969D-D48505335844}";
	public const string DeregisterDeviceToken =
        "{0F47EDC4-55C4-42E8-8F32-63C9BAE06181}";
	public const string PushExternalNotification =
        "{2A88944A-16F6-494F-8867-33CBD535C1E5}";
}
```

These permission catalog classes need to be static and decorated with the `PermissionTypeCatalogAttribute` that specifies the namespace of the permission catalog. Individual public constant string fields in the class then define the actual permission types in it \(field name is used as the permission name that is appended to the catalog namespace\).

An entity \(e.g. a user\) can possess a permission by creating an instance of `Permission` class which defines the actual ownership of a permission type \(“create a blogpost in a category”\) on a certain resource \(e.g. ID of the category\) and within a certain context \(e.g. “german section of the website”\). Both resource and context can also be null meaning a universal permission \(i.e. for all resources and within all contexts; also, the concept of resources and/or contexts may not necessarily be applicable to all permission types where it does not make sense\).

These `Permission`s will usually be attached to user roles, groups or individual users – this is, however, completely up to the implementation of the end applications using the framework and will usually depend on the business logic of the application. This gap between the application-specific implementation of users and permission is bridged by an implementation of `IUserContext` which resolves the user and his permission in the context of the current request. For the web ASP.NET platform, this is implemented by the framework by the Revo.Platforms.AspNet package whose implementation is backed by the enterprise-grade ASP.NET Identity framework developed by Microsoft that can be easily plugged with many authentication providers \(e.g. local database, OAuth, etc.\) and already contains the implementations for many common scenarios \(e.g. user management, user roles, etc.\).

Besides the possibility to use this user context manually for authorization \(e.g. in API controllers or command handlers\), it is also possible to use some of the framework infrastructure to make this easier \(see following chapters\).

### Command permissions

Using the concept of permissions, it is possible to decorate commands and queries with attributes that will require them to be authorized before their actual execution. This is implemented using command filters \(see chapter 7.4.2\) and is enabled by default. To define an authorization rule for a command, decorate its class with an `AuthorizePermissionsAttribut`e:

```csharp
[AuthorizePermissions(Permissions.PushExternalNotification)]
```

The attribute argument can refer to a number of permission type GUIDs defined in a references permission catalog \(here, the `Permissions` class\).

An alternative way for command authorization making it also possible to employ a custom command-authorization logic is to implement an `IPreCommandFilter<T>` \(possibly in the form of a `CommandAuthorizer<T>`\).

## Entity query filters

It is sometimes necessary to apply authorization rules in a different way – not specifying that a user can perform a certain action, but rather affecting what portion of data \(e.g. what rows in database\) he sees. To achieve this in a non-intrusive, aspect-oriented way, framework implements a concept of entity query filters. The entity query authorizer then takes a queryable collection and the current command as an input and returns a filtered \(i.e. with authorization rule filtering applied\) on the output, for example:

```csharp
IQueryable<Order> orders = await readRepository.FindAll<Order>()
    .Where(x => x.Status == OrderStatus.Pending)
    .AuthorizeAsync(command);
List<Order> orderList = await orders.ToListAsync();
```

The `orderList` will now contain list of pending orders filtered according to the registered system-wide rules for order authorization. It is also possible to authorize according to a nested entity, e.g. if authorization based on the customer who sent the order was needed instead:

```csharp
IQueryable<Order> orders = await readRepository.FindAll<Order>()
    .AuthorizeAsync(command, x => x.Customer);
```

The actual authorization rules are defined by implementing `IEntityQueryFilter<T>` interface \(where `T` denotes the type of the entity authorized\) and registering its instances in the dependency container. The `FilterAsync` method returns a filtering expression that is applied to the queryable object, for example:

```csharp
public class OrderQueryFilter : IEntityQueryFilter<Order>
{
	private readonly IUserContext userContext;
 
    ...
	
	public async Task<Expression<Func<Order, bool>>> FilterAsync(
		ICommandBase query)
	{
		return x => x.Customer.Id == userContext.UserId;
	}
	
	...	
}
```

\(some parts omitted for brevity\).

