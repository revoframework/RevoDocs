---
description: >-
  Prior to bootstrapping a Revo application, one must first set up the framework
  and specify its configuration, which makes it possible to customize its
  behavior in a number of ways.
---

# Configuration and boostrapping

## Configuration

Apart from the usual possibility to specify many run-time parameters in a dynamic manner \(such as loading database connection strings from configuration files\), many aspects of the framework can only be altered using the programmatic configuration API. This configuration is represented by an instance of `IRevoConfiguration` , which itself is only a container for a number of `IRevoConfigurationSection` sections. When done building the configuration, the framework bootstraps the application using the parameters provided.

## Examples

Typically, the configuration object would be created at an application entry point and then configured as neccessary using fluent-style extension methods.

### ASP.NET Core

To easily bootstrap a Revo in an ASP.NET core application, simply inherit your `Startup` class from the `RevoStartup` base class and override the `CreateRevoConfiguration` method.

Example:

```csharp
public class Startup : RevoStartup
{
    public Startup(IConfiguration configuration) : base(configuration)
    {
    }

    public override void ConfigureServices(IServiceCollection services)
    {
        base.ConfigureServices(services);
        // TODO configure your services
    }

    public override void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)
    {
        base.Configure(app, env, loggerFactory);
        // TODO configure your ASP.NET core app, e.g.:
        app.UseMvcWithDefaultRoute();
    }

    protected override IRevoConfiguration CreateRevoConfiguration()
    {
        string connectionString = Configuration.GetConnectionString("TodosPostgreSQL");

        return new RevoConfiguration()
            .UseAspNetCore()
            .UseEFCoreDataAccess(contextBuilder => contextBuilder.UseNpgsql(connectionString))
            .UseAllEFCoreInfrastructure();
    }
}
```

### ASP.NET

In ASP.NET applications, this would be the inside a `RevoHttpApplication`, e.g.:

```csharp
public class MvcApplication : RevoHttpApplication
{
    protected override IRevoConfiguration CreateRevoConfiguration()
    {
        return new RevoConfiguration()
            .UseAspNet()
            .UseEF6DataAccess(useAsPrimaryRepository: true, connection: new EF6ConnectionConfiguration("EntityContext"))
            .UseAllEF6Infrastructure()
            .UseHangfire();
    }
}
```



