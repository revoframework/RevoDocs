# Overview

Revo is an application framework for modern C\#/.NET applications built with _event sourcing_, _CQRS_ and _DDD_.

* [**Features**](../features.md)\*\*\*\*
* \*\*\*\*[**Super-short example**](../super-short-example.md)\*\*\*\*
* \*\*\*\*[**Design overview**](design-overview.md)
* [Project structure](project-structure.md)

## Requirements

The framework is written in C\# 7.1 and primarily targets .NET Core 2.1+/.NET Standard 2.0. Some of its modules also require .NET Framework 4.7.1+ where needed \(e.g. Entity Framework 6 support\).

Revo also makes a heavy use of the C\# async/await pattern and uses the TAP \(Task Asynchronous Pattern\) throughout its entire codebase \(i.e. _async all the way_ approach\).

