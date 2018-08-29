---
description: >-
  Like many other frameworks, Revo also builds on the principles inversion of
  control (IoC). For the heavy lifting, the framework uses the open-source
  Ninject dependency container and injector.
---

# Dependency injection

## Module auto-loading

Upon the start of an application, the framework automatically locates and loads Ninject module definitions from all referenced assemblies.

This behavior can be suppressed by decorating the module class with `[AutoLoadModule(false)]` attribute. This can be furthermore overridden \(disabling implicitly enabled or re-enabling previously disabled modules\) using the DI kernel [configuration API](configuration.md), e.g.:

```csharp
new RevoConfiguration()
    ...
    .OverrideModuleLoading<XyzModule>(true);
    ;
```

In conjunction with the configuration API, this can be used, for example, for on-demand loading of module features.

## Further notes

{% hint style="info" %}
While it is possible to use the framework with other dependency container libraries, it would currently require a reimplementation of some internal dependency bindings using the specific container library, and is not very practical because of that.  
There are plans to remove this Ninject dependency and replace it with a more flexible system allowing to configure any other DI mechanisms, however.
{% endhint %}

