# Multi-tenancy

## Multi-tenancy environments

With the rising popularity of cloud software deployment and the software-as-a-service \(SaaS\) model, it is a very common requirement that multiple instances of a single application \(e.g. multiple customers with separate environments\) should be able to run on a single infrastructure node \(one webserver, for example\) or using shared resources \(e.g. single database for a greater number of customers\). To be able to abstract from these concepts when developing an application \(which would usually make the application logic much more complicated\), the framework provides support for basic multi-tenancy support. Multi-tenant application is an application that is able to host multiple independent application instances in one single running instance of the system. In this terminology, a tenant is the unit that separates individual instances among themselves – i.e. it would be quite customary that each customer would also be a tenant. There are several facilities available to help with that.

## Tenant context

### Context resolver

The framework defines an `ITenantContextResolver` which is responsible for resolving the tenant that is active for the scope of an active request. By default, the framework uses `NullTenantContextResolver` which always resolves to a null tenant. It is also possible that certain parts of the application will have null tenant \(e.g. the login pages\) and some will actually resolve to a specific tenant \(i.e. the rest of the application\). The framework defines one more tenant context resolver – `SingleTenantContextResolver` which always resolver to a specific \(constant-value\) tenant. This is especially helpful during development \(rather than in a production environment\). Production-envionment applications are free to implement and register their own implementations of tenant context resolver – very commonly resolving by the HTTP request subdomain, request authorization or any other request-dependent properties. The resolvers return objects of the `ITenant` interface which simply contain just an ID and a name property and can be implemented in any way they need.

```csharp
public interface ITenantContextResolver
{
    ITenant ResolveTenant();
}
```

### Tenant context

The tenant context is useful for a number of reasons.

```csharp
public interface ITenantContext
{
    ITenant Tenant { get; }
}
```

Besides the option to manually work with it using `ITenantContext`, the framework implements a repository filter \(enabled by default for all repositories\) that automatically filters the returned and added or modified repository records by their `TenantId` provided they implement the `ITenantOwned` interface. 

```csharp
public interface ITenantOwned
{
    Guid? TenantId { get; }
}
```

Within any tenant context, the repository allows working with any records that belong to that specific tenant and records that have null tenant specified \(and throws an exception if the repository tries to modify records it should not\). By default, null tenant context has only access to records having null tenant ID. This enables an automatic workflow when working with any tenant-owned entities that prevents the risk of leaking unauthorized access to records of other tenants.

## Configuration

Multi-tenancy features can be configured when setting-up Revo application, e.g. in you Startup class:

```csharp
return new RevoConfiguration()
    ...
    .ConfigureInfrastructure(
      cfg =>
      {
        cfg.Tenancy.UseNullTenantContextResolver = false;// now you can bind your own tenant context resolver
        cfg.Tenancy.EnableTenantRepositoryFilter = true; // by default
        // more...
      });
```

