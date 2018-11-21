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

## Task and request scopes

To be able to correctly resolve all dependencies in all different situations \(in a command handlers, when processing a web request, when processing a message from other connected systems or when running a background job, in projections, etc.\), Revo defines Ninject binding extensions for defining object life-time scope in Ninject modules.

```csharp
public static class NinjectBindingExtensions
{
    ...
    
    public static IBindingNamedWithOrOnSyntax<T> InRequestScope<T>(this IBindingInSyntax<T> syntax) { ... }
    public static IBindingNamedWithOrOnSyntax<T> InTaskScope<T>(this IBindingInSyntax<T> syntax) { ... }
    
    ...
}
```

### InTaskScope

Most of the regular services used when processing requests \(command, projections...\), implemented by the application end-developer, that should not be transient or global singletons, will benefit from using the `InTaskScope()` life-time scope. This scope ensures the object gets correctly created only once \(singleton-like\) during a task ran by the framework. A _task_ could be a variety of things \(processing a command with command handler, running a background job, processing an event with an async listener...\). It is also the scope that most of the framework services visible to the application developer use \(repositories, default command handler bindings, etc.\).

Note that a single web request processing a single command mitypically consist of multiple tasks.  
If there is no current task active at time of the dependency resolution, it falls back to using the request-scope or the thread-scope \(in order\).

### InRequestScope

`InRequestScope` is a wrapper over what a would be a request-singleton, e.g. what official Nuget packages offer as InRequestScope for ASP.NET 4., but implemented in a more platform-independent way. In Revo, it is internally implemented by respective platform packages \(e.g. _Revo.AspNetCore_\). Falls-back to task-scope and thread-scope \(in order\) if there is no active request.

May be used for things that need to outlive the original task scopes.

## Further notes

{% hint style="info" %}
While it is possible to use the framework with other dependency container libraries, it would currently require a reimplementation of some internal dependency bindings using the specific container library, and is not very practical because of that.  
There are plans to remove this Ninject dependency and replace it with a more flexible system allowing to configure any other DI mechanisms, however.
{% endhint %}

