# .BlockDetector

# Detect Blocking Calls in ASP.NET Core Applications

Blocking calls can lead to ThreadPool starvation. Outputs a warning to the log when blocking calls are made on the ThreadPool.

#Usage

Add early to your ASP.NET Core pipeline (see: samples)

#Caveats

Doesn't detect everything... so its not a panacea; you should actively try to avoid using blocking calls.

For async methods with occurances of .ConfigureAwait(false), detection won't alert for blocking Monitor calls after occurences in the case where the returned Task wasn't already completed
Won't alert for blocking calls that don't block, like on precompleted Tasks (e.g. a single small Body.Write)
Won't alert for blocking that happens in syscalls (e.g. File.Read(...), Thread.Sleep)
Will detect CLR initiated waits lock,ManualResetEventSlim,Semaphore{Slim},Task.Wait,Task.Result etc; if they do block.

#Example
If you had a method like:

[HttpGet("/sync-over-async")]
public static int BlockingTask()
{
    // Detected blocking
    return MethodAsync().Result;
}

private static async Task<int> MethodAsync()
{
    await Task.Delay(1000);
    return 5;
}


It would output to your log


warn: Ben.Diagnostics.BlockingMonitor[6]
  Blocking method has been invoked and blocked, this can lead to threadpool starvation.
    at System.Threading.Tasks.Task.InternalWait(Int32 millisecondsTimeout, CancellationToken cancellationToken)
    at System.Threading.Tasks.Task`1.GetResultCore(Boolean waitCompletionNotification)
    at mvc.HomeController.BlockingTask()                <------------------ ** Blocking function **
    at lambda_method(Closure , Object , Object[] )
    at Microsoft.Extensions.Internal.ObjectMethodExecutor.Execute(Object target, Object[] parameters)
    at Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker.<InvokeActionMethodAsync>d__12.MoveNext()
    at System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start[TStateMachine](TStateMachine& stateMachine)
    at Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker.InvokeActionMethodAsync()
    at Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker.Next(State& next, Scope& scope, Object& state, Boolean& isCompleted)
    at Microsoft.AspNetCore.Mvc.Internal.ControllerActionInvoker.<InvokeNextActionFilterAsync>d__10.MoveNext()
    ...
    at Microsoft.AspNetCore.Hosting.Internal.RequestServicesContainerMiddleware.Invoke(HttpContext httpContext)
    at Microsoft.AspNetCore.Hosting.Internal.HostingApplication.ProcessRequestAsync(Context context)
    at Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Http.Frame`1.<ProcessRequestsAsync>d__2.MoveNext()
    at System.Threading.ExecutionContext.Run(ExecutionContext executionContext, ContextCallback callback, Object state)
    at System.Runtime.CompilerServices.AsyncMethodBuilderCore.<>c__DisplayClass4_0.<OutputAsyncCausalityEvents>b__0()
    at Microsoft.AspNetCore.Server.Kestrel.Internal.System.IO.Pipelines.Pipe.<>c.<.cctor>b__67_3(Object o)
    at Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Infrastructure.LoggingThreadPool.<>c__DisplayClass6_0.<Schedule>b__0()
    at Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Infrastructure.LoggingThreadPool.<RunAction>b__3_0(Object o)
    at System.Threading.ExecutionContext.Run(ExecutionContext executionContext, ContextCallback callback, Object state)
    at System.Threading.ThreadPoolWorkQueue.Dispatch()


Also catches held locks

Outputs (with some extra formatting)

warn: Microsoft.AspNetCore.Diagnostics.DetectBlocking[6]
 Blocking method has been invoked and blocked, this can lead to threadpool starvation.
     at Task AoA.Gaia.Startup.LockedMethod(HttpContext httpContext)
       in C:\Work\AoA\src\AoA.Gaia\Startup.cs:line 125 ********
     at IApplicationBuilder Microsoft.AspNetCore.Builder.UseExtensions.Use(IApplicationBuilder app, Func<HttpContext, Func<Task>, Task> middleware)+() => { }
....
     at void Microsoft.AspNetCore.Server.Kestrel.Internal.System.IO.Pipelines.Pipe._scheduleContinuation(object o)
     at void Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Infrastructure.LoggingThreadPool.Schedule(Action<object> action, object state)+() => { }
     at void Microsoft.AspNetCore.Server.Kestrel.Core.Internal.Infrastructure.LoggingThreadPool.RunAction()+(object o) => { }
     at bool System.Threading.ThreadPoolWorkQueue.Dispatch()
