---
description: >-
  Prior to bootstrapping a Revo application, one must first set up the framework
  and specify its configuration, which makes it possible to customize its
  behavior in a number of ways.
---

# Configuration

## Configuration

Apart from the usual possibility to specify many run-time parameters in a dynamic manner \(such as loading database connection strings from configuration files\), many aspects of the framework can only be altered using the programmatic configuration API. This configuration is represented by an instance of `IRevoConfiguration` , which itself is only a container for a number of `IRevoConfigurationSection` sections. When done building the configuration, the framework bootstraps the application using the parameters provided.

## Examples

Typically, the configuration object would be created at an application entry point and then configured as neccessary using fluent-style extension methods.

### ASP.NET

In ASP.NET applications, this would be the inside a `RevoHttpApplication`, e.g.:

```aspnet
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



