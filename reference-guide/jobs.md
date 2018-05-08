# Jobs

## Overview

The framework provides a support for jobs that would be executed in background, optionally at a later point in time. This concept was previously discussed in chapter 6.6.2. Execution “in background” means asynchronously to the current executing thread \(i.e. so it does not block the HTTP request currently processed, for example\).

### Job scheduler

The interface `IJobScheduler` provides a gateway to working with the jobs scheduler and offers a way to enqueue a job for immediate execution in background \(as soon the machine has the capacity to do so, e.g. when it is limited to process only limited amount of jobs in parallel\) or to schedule a job for execution at a specific date and time \(again started automatically in background\). It is also possible to delete a previously enqueued/schedule job \(if it has not already been processed\). 

### Jobs and job handlers

Jobs work in a way very similar to commands: it is possible to define job types by implementing the `IJob` interface \(which itself is empty\). For every job type needs to be a job han-dler registered in the dependency container. The job handler interface looks like this:

```csharp
public interface IJobHandler<in T>
	where T : IJob
{
	Task HandleAsync(T job, CancellationToken cancellationToken);
}
```

The job handler is responsible for the actual execution of a job.

There is an out-of-the-box support for `ExecuteCommandJob` which simply executes any com-mand specified. Moreover, there are `EnqueueJobCommand` and `ScheduleJobCommand` commands that on the other hand allow enqueuing and scheduling of jobs simply using a command. This is especially useful when used in conjunction with saga that can only produce commands to perform any side-effects.

## Hangfire

For a reliable execution of jobs, the framework currently uses the [Hangfire ](https://www.hangfire.io/)library to back its implementation. Hangfire also takes care of job failure management \(restarting the jobs when they fail, if configured to do so\), offers a web administration console and handles back-ground execution in ASP.NET applications well \(so their application pools are not recycled prematurely\).



