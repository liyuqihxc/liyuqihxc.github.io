---
title: 深入理解ValueTask
categories:
- [翻译]
- [.Net]
tags: [.Net Core, BCL, Performance]
---

原文：[Understanding the Whys, Whats, and Whens of ValueTask](https://blogs.msdn.microsoft.com/dotnet/2018/11/07/understanding-the-whys-whats-and-whens-of-valuetask/)

作者：[Stephen Toub - MSFT](https://social.msdn.microsoft.com/profile/Stephen+Toub+-+MSFT)


The .NET Framework 4 saw the introduction of the System.Threading.Tasks namespace, and with it the Task class. This type and the derived Task&lt;TResult> have long since become a staple of .NET programming, key aspects of the asynchronous programming model introduced with C# 5 and its async / await keywords. In this post, I’ll cover the newer ValueTask/ValueTask&lt;TResult> types, which were introduced to help improve asynchronous performance in common use cases where decreased allocation overhead is important.

## Task

Task serves multiple purposes, but at its core it’s a “promise”, an object that represents the eventual completion of some operation. You initiate an operation and get back a Task for it, and that Task will complete when the operation completes, which may happen synchronously as part of initiating the operation (e.g. accessing some data that was already buffered), asynchronously but complete by the time you get back the Task (e.g. accessing some data that wasn’t yet buffered but that was very fast to access), or asynchronously and complete after you’re already holding the Task (e.g. accessing some data from across a network). Since operations might complete asynchronously, you either need to block waiting for the results (which often defeats the purpose of the operation having been asynchronous to begin with) or you need to supply a callback that’ll be invoked when the operation completes. In .NET 4, providing such a callback was achieved via ContinueWith methods on the Task, which explicitly exposed the callback model by accepting a delegate to invoke when the Task completed:

```csharp
SomeOperationAsync().ContinueWith(task =>
{
    try
    {
        TResult result = task.Result;
        UseResult(result);
    }
    catch (Exception e)
    {
        HandleException(e);
    }
});
```

But with the .NET Framework 4.5 and C# 5, Tasks could simply be awaited, making it easy to consume the results of an asynchronous operation, and with the generated code being able to optimize all of the aforementioned cases, correctly handling things regardless of whether the operation completes synchronously, completes asynchronously quickly, or completes asynchronously after already (implicitly) providing a callback:

```csharp
TResult result = await SomeOperationAsync();
UseResult(result);
```

Task as a class is very flexible and has resulting benefits. For example, you can await it multiple times, by any number of consumers concurrently. You can store one into a dictionary for any number of subsequent consumers to await in the future, which allows it to be used as a cache for asynchronous results. You can block waiting for one to complete should the scenario require that. And you can write and consume a large variety of operations over tasks (sometimes referred to as “combinators”), such as a “when any” operation that asynchronously waits for the first to complete.

However, that flexibility is not needed for the most common case: simply invoking an asynchronous operation and awaiting its resulting task:

```csharp
TResult result = await SomeOperationAsync();
UseResult(result);
```

In such usage, we don’t need to be able to await the task multiple times. We don’t need to be able to handle concurrent awaits. We don’t need to be able to handle synchronous blocking. We don’t need to write combinators. We simply need to be able to await the resulting promise of the asynchronous operation. This is, after all, how we write synchronous code (e.g. TResult result = SomeOperation();), and it naturally translates to the world of async / await.

Further, Task does have a potential downside, in particular for scenarios where instances are created a lot and where high-throughput and performance is a key concern: Task is a class. As a class, that means that any operation which needs to create one needs to allocate an object, and the more objects that are allocated, the more work the garbage collector (GC) needs to do, and the more resources we spend on it that could be spent doing other things.

The runtime and core libraries mitigate this in many situations. For example, if you write a method like the following:

```csharp
public async Task WriteAsync(byte value)
{
    if (_bufferedCount == _buffer.Length)
    {
        await FlushAsync();
    }
    _buffer[_bufferedCount++] = value;
}
```

in the common case there will be space available in the buffer and the operation will complete synchronously. When it does, there’s nothing special about the Task that needs to be returned, since there’s no return value: this is the Task-based equivalent of a void-returning synchronous method. Thus, the runtime can simply cache a single non-generic Task and use that over and over again as the result task for any async Task method that completes synchronously (that cached singleton is exposed via Task.CompletedTask). Or for example, if you write:

```csharp
public async Task<bool> MoveNextAsync()
{
    if (_bufferedCount == 0)
    {
        await FillBuffer();
    }
    return _bufferedCount > 0;
}
```
in the common case, we expect there to be some data buffered, in which case this method simply checks _bufferedCount, sees that it’s larger than 0, and returns true; only if there’s currently no buffered data does it need to perform an operation that might complete asynchronously. And since there are only two possible Boolean results (true and false), there are only two possible Task&lt;bool> objects needed to represent all possible result values, and so the runtime is able to cache two such objects and simply return a cached Task&lt;bool> with a Result of true, avoiding the need to allocate. Only if the operation completes asynchronously does the method then need to allocate a new Task&lt;bool>, because it needs to hand back the object to the caller before it knows what the result of the operation will be, and needs to have a unique object into which it can store the result when the operation does complete.

The runtime maintains a small such cache for other types as well, but it’s not feasible to cache everything. For example, a method like:

```csharp
public async Task<int> ReadNextByteAsync()
{
    if (_bufferedCount == 0)
    {
        await FillBuffer();
    }
 
    if (_bufferedCount == 0)
    {
        return -1;
    }
 
    _bufferedCount--;
    return _buffer[_position++];
}
```

will also frequently complete synchronously. But unlike the Boolean case, this method returns an Int32 value, which has ~4 billion possible results, and caching a Task&lt;int> for all such cases would consume potentially hundreds of gigabytes of memory. The runtime does maintain a small cache for Task&lt;int>, but only for a few small result values, so for example if this completes synchronously (there’s data in the buffer) with a value like 4, it’ll end up using a cached task, but if it completes synchronously with a value like 42, it’ll end up allocating a new Task&lt;int>, akin to calling Task.FromResult(42).

Many library implementations attempt to mitigate this further by maintaining their own cache as well. For example, the MemoryStream.ReadAsync overload introduced in the .NET Framework 4.5 always completes synchronously, since it’s just reading data from memory. ReadAsync returns a &lt;int>, where the Int32 result represents the number of bytes read. ReadAsync is often used in a loop, often with the number of bytes requested the same on each call, and often with ReadAsync able to fully fulfill that request. Thus, it’s common for repeated calls to ReadAsync to return a Task&lt;int> synchronously with the same result as it did on the previous call. As such, MemoryStream maintains a cache of a single task, the last one it returned successfully. Then on a subsequent call, if the new result matches that of its cached Task&lt;int>, it just returns the cached one again; otherwise, it uses Task.FromResult to create a new one, stores that as its new cached task, and returns it.

Even so, there are many cases where operations complete synchronously and are forced to allocate a Task&lt;TResult> to hand back.

## ValueTask&lt;TResult> and synchronous completion

All of this motivated the introduction of a new type in .NET Core 2.0 and made available for previous .NET releases via a System.Threading.Tasks.Extensions NuGet package: ValueTask&lt;TResult>.

ValueTask&lt;TResult> was introduced in .NET Core 2.0 as a struct capable of wrapping either a TResult or a Task&lt;TResult>. This means it can be returned from an async method, and if that method completes synchronously and successfully, nothing need be allocated: we can simply initialize this ValueTask&lt;TResult> struct with the TResult and return that. Only if the method completes asynchronously does a Task&lt;TResult> need to be allocated, with the ValueTask&lt;TResult> created to wrap that instance (to minimize the size of ValueTask&lt;TResult> and to optimize for the success path, an async method that faults with an unhandled exception will also allocate a Task&lt;TResult>, so that the ValueTask&lt;TResult> can simply wrap that Task&lt;TResult> rather than always having to carry around an additional field to store an Exception).

With that, a method like MemoryStream.ReadAsync that instead returns a ValueTask&lt;int> need not be concerned with caching, and can instead be written with code like:

```csharp
public override ValueTask<int> ReadAsync(byte[] buffer, int offset, int count)
{
    try
    {
        int bytesRead = Read(buffer, offset, count);
        return new ValueTask<int>(bytesRead);
    }
    catch (Exception e)
    {
        return new ValueTask<int>(Task.FromException<int>(e));
    }
}
```
## ValueTask&lt;TResult> and asynchronous completion

Being able to write an async method that can complete synchronously without incurring an additional allocation for the result type is a big win. This is why ValueTask&lt;TResult> was added to .NET Core 2.0, and why new methods that are expected to be used on hot paths are now defined to return ValueTask&lt;TResult> instead of Task&lt;TResult>. For example, when we added a new ReadAsync overload to Stream in .NET Core 2.1 in order to be able to pass in a Memory&lt;byte> instead of a byte[], we made the return type of that method be ValueTask&lt;int>. That way, Streams (which very often have a ReadAsync method that completes synchronously, as in the earlier MemoryStream example) can now be used with significantly less allocation.

However, when working on very high-throughput services, we still care about avoiding as much allocation as possible, and that means thinking about reducing and removing allocations associated with asynchronous completion paths as well.

With the await model, for any operation that completes asynchronously we need to be able to hand back an object that represents the eventual completion of the operation: the caller needs to be able to hand off a callback that’ll be invoked when the operation completes, and that requires having a unique object on the heap that can serve as the conduit for this specific operation. It doesn’t, however, imply anything about whether that object can be reused once an operation completes. If the object can be reused, then an API can maintain a cache of one or more such objects, and reuse them for serialized operations, meaning it can’t use the same object for multiple in-flight async operations, but it can reuse an object for non-concurrent accesses.

In .NET Core 2.1, ValueTask&lt;TResult> was augmented to support such pooling and reuse. Rather than just being able to wrap a TResult or a Task&lt;TResult>, a new interface was introduced, IValueTaskSource&lt;TResult>, and ValueTask&lt;TResult> was augmented to be able to wrap that as well. IValueTaskSource&lt;TResult> provides the core support necessary to represent an asynchronous operation to ValueTask&lt;TResult> in a similar manner to how Task&lt;TResult> does:

```csharp
public interface IValueTaskSource<out TResult>
{
    ValueTaskSourceStatus GetStatus(short token);
    void OnCompleted(Action<object> continuation, object state, short token, ValueTaskSourceOnCompletedFlags flags);
    TResult GetResult(short token);
}
```

GetStatus is used to satisfy properties like ValueTask&lt;TResult>.IsCompleted, returning an indication of whether the async operation is still pending or whether it’s completed and how (success or not). OnCompleted is used by the ValueTask&lt;TResult>‘s awaiter to hook up the callback necessary to continue execution from an await when the operation completes. And GetResult is used to retrieve the result of the operation, such that after the operation completes, the awaiter can either get the TResult or propagate any exception that may have occurred.

Most developers should never have a need to see this interface: methods simply hand back a ValueTask&lt;TResult> that may have been constructed to wrap an instance of this interface, and the consumer is none-the-wiser. The interface is primarily there so that developers of performance-focused APIs are able to avoid allocation.

There are several such APIs in .NET Core 2.1. The most notable are Socket.ReceiveAsync and Socket.SendAsync, with new overloads added in 2.1, e.g.

```csharp
public ValueTask<int> ReceiveAsync(Memory<byte> buffer, SocketFlags socketFlags, CancellationToken cancellationToken = default);
```

This overload returns a ValueTask&lt;int>. If the operation completes synchronously, it can simply construct a ValueTask&lt;int> with the appropriate result, e.g.

```csharp
int result = …;
return new ValueTask<int>(result);
```

If it completes asynchronously, it can use a pooled object that implements this interface:

```csharp
IValueTaskSource<int> vts = …;
return new ValueTask<int>(vts);
```

The Socket implementation maintains one such pooled object for receives and one for sends, such that as long as no more than one of each is outstanding at a time, these overloads will end up being allocation-free, even if they complete operations asynchronously. That’s then further surfaced through NetworkStream. For example, in .NET Core 2.1, Stream exposes:

```csharp
public virtual ValueTask<int> ReadAsync(Memory<byte> buffer, CancellationToken cancellationToken);
```

which NetworkStream overrides. NetworkStream.ReadAsync just delegates to Socket.ReceiveAsync, so the wins from Socket translate to NetworkStream, and NetworkStream.ReadAsync effectively becomes allocation-free as well.

## Non-generic ValueTask

When ValueTask&lt;TResult> was introduced in .NET Core 2.0, it was purely about optimizing for the synchronous completion case, in order to avoid having to allocate a Task&lt;TResult> to store the TResult already available. That also meant that a non-generic ValueTask wasn’t necessary: for the synchronous completion case, the Task.CompletedTask singleton could just be returned from a Task-returning method, and was implicitly by the runtime for async Task methods.

With the advent of enabling even asynchronous completions to be allocation-free, however, a non-generic ValueTask becomes relevant again. Thus, in .NET Core 2.1 we also introduced the non-generic ValueTask and IValueTaskSource. These provide direct counterparts to the generic versions, usable in similar ways, just with a void result.

## Implementing IValueTaskSource / IValueTaskSource&lt;T>
Most developers should never need to implement these interfaces. They’re also not particularly easy to implement. If you decide you need to, there are several implementations internal to .NET Core 2.1 that can serve as a reference, e.g.

* [AwaitableSocketAsyncEventArgs](https://github.com/dotnet/corefx/blob/61f51e6b2b26271de205eb8a14236afef482971b/src/System.Net.Sockets/src/System/Net/Sockets/Socket.Tasks.cs#L808)
* [AsyncOperation&lt;TResult>](https://github.com/dotnet/corefx/blob/89ab1e83a7e00d869e1580151e24f01226acaf3f/src/System.Threading.Channels/src/System/Threading/Channels/AsyncOperation.cs#L37)
* [DefaultPipeReader](https://github.com/dotnet/corefx/blob/a10890f4ffe0fadf090c922578ba0e606ebdd16c/src/System.IO.Pipelines/src/System/IO/Pipelines/Pipe.DefaultPipeReader.cs#L16)

To make this easier for developers that do want to do it, in .NET Core 3.0 we plan to introduce all of this logic encapsulated into a ManualResetValueTaskSourceCore&lt;TResult> type, a struct that can be encapsulated into another object that implements IValueTaskSource&lt;TResult> and/or IValueTaskSource, with that wrapper type simply delegating to the struct for the bulk of its implementation. You can learn more about this in the associated issue in the dotnet/corefx repo at [https://github.com/dotnet/corefx/issues/32664](https://github.com/dotnet/corefx/issues/32664).

## Valid consumption patterns for ValueTasks
From a surface area perspective, ValueTask and ValueTask&lt;TResult> are much more limited than Task and Task&lt;TResult>. That’s ok, even desirable, as the primary method for consumption is meant to simply be awaiting them.

However, because ValueTask and ValueTask&lt;TResult> may wrap reusable objects, there are actually significant constraints on their consumption when compared with Task and Task&lt;TResult>, should someone veer off the desired path of just awaiting them. In general, the following operations should never be performed on a ValueTask / ValueTask&lt;TResult>:

* **Awaiting a ValueTask / ValueTask&lt;TResult> multiple times.** The underlying object may have been recycled already and be in use by another operation. In contrast, a Task / Task&lt;TResult> will never transition from a complete to incomplete state, so you can await it as many times as you need to, and will always get the same answer every time.
* **Awaiting a ValueTask / ValueTask&lt;TResult> concurrently.** The underlying object expects to work with only a single callback from a single consumer at a time, and attempting to await it concurrently could easily introduce race conditions and subtle program errors. It’s also just a more specific case of the above bad operation: “awaiting a ValueTask / ValueTask&lt;TResult> multiple times.” In contrast, Task / Task&lt;TResult> do support any number of concurrent awaits.
* **Using .GetAwaiter().GetResult() when the operation hasn’t yet completed.** The IValueTaskSource / IValueTaskSource&lt;TResult> implementation need not support blocking until the operation completes, and likely doesn’t, so such an operation is inherently a race condition and is unlikely to behave the way the caller intends. In contrast, Task / Task&lt;TResult> do enable this, blocking the caller until the task completes.

If you have a ValueTask or a ValueTask&lt;TResult> and you need to do one of these things, you should use .AsTask() to get a Task / Task&lt;TResult> and then operate on that resulting task object. After that point, you should never interact with that ValueTask / ValueTask&lt;TResult> again.

**The short rule is this**: with a ValueTask or a ValueTask&lt;TResult>, you should either await it directly (optionally with .ConfigureAwait(false)) or call AsTask() on it directly, and then never use it again, e.g.

```csharp
// Given this ValueTask<int>-returning method…
public ValueTask<int> SomeValueTaskReturningMethodAsync();
…
// GOOD
int result = await SomeValueTaskReturningMethodAsync();
 
// GOOD
int result = await SomeValueTaskReturningMethodAsync().ConfigureAwait(false);
 
// GOOD
Task<int> t = SomeValueTaskReturningMethodAsync().AsTask();
 
// WARNING
ValueTask<int> vt = SomeValueTaskReturningMethodAsync();
... // storing the instance into a local makes it much more likely it'll be misused,
    // but it could still be ok
 
// BAD: awaits multiple times
ValueTask<int> vt = SomeValueTaskReturningMethodAsync();
int result = await vt;
int result2 = await vt;
 
// BAD: awaits concurrently (and, by definition then, multiple times)
ValueTask<int> vt = SomeValueTaskReturningMethodAsync();
Task.Run(async () => await vt);
Task.Run(async () => await vt);
 
// BAD: uses GetAwaiter().GetResult() when it's not known to be done
ValueTask<int> vt = SomeValueTaskReturningMethodAsync();
int result = vt.GetAwaiter().GetResult();
```

There is one additional advanced pattern that some developers may choose to use, hopefully only after measuring carefully and finding it provides meaningful benefit. Specifically, ValueTask / ValueTask&lt;TResult> do expose some properties that speak to the current state of the operation, for example the IsCompleted property returning false if the operation hasn’t yet completed, and returning true if it has (meaning it’s no longer running and may have completed successfully or otherwise), and the IsCompletedSuccessfully property returning true if and only if it’s completed and completed successfully (meaning attempting to await it or access its result will not result in an exception being thrown). For very hot paths where a developer wants to, for example, avoid some additional costs only necessary on the asynchronous path, these properties can be checked prior to performing one of the operations that essentially invalidates the ValueTask / ValueTask&lt;TResult>, e.g. await, .AsTask(). For example, in the SocketsHttpHandler implementation in .NET Core 2.1, the code issues a read on a connection, which returns a ValueTask&lt;int>. If that operation completed synchronously, then we don’t need to worry about being able to cancel the operation. But if it completes asynchronously, then while it’s running we want to hook up cancellation such that a cancellation request will tear down the connection. As this is a very hot code path, and as profiling showed it to make a small difference, the code is structured essentially as follows:

```csharp
int bytesRead;
{
    ValueTask<int> readTask = _connection.ReadAsync(buffer);
    if (readTask.IsCompletedSuccessfully)
    {
        bytesRead = readTask.Result;
    }
    else
    {
        using (_connection.RegisterCancellation())
        {
            bytesRead = await readTask;
        }
    }
}
```

This pattern is acceptable, because the ValueTask&lt;int> isn’t used again after either .Result is accessed or it’s awaited.

## Should every new asynchronous API return ValueTask / ValueTask&lt;TResult>?
In short, no: the default choice is still Task / Task&lt;TResult>.

As highlighted above, Task and Task&lt;TResult> are easier to use correctly than are ValueTask and ValueTask&lt;TResult>, and so unless the performance implications outweigh the usability implications, Task / Task&lt;TResult>are still preferred. There are also some minor costs associated with returning a ValueTask&lt;TResult> instead of a Task&lt;TResult>, e.g. in microbenchmarks it’s a bit faster to await a Task&lt;TResult> than it is to await a ValueTask&lt;TResult>, so if you can use cached tasks (e.g. you’re API returns Task or Task&lt;bool>), you might be better off performance-wise sticking with Task and Task&lt;bool>. ValueTask / ValueTask&lt;TResult> are also multiple words in size, and so when these are awaitd and a field for them is stored in a calling async method’s state machine, they’ll take up a little more space in that state machine object.

However, ValueTask / ValueTask&lt;TResult> are great choices when a) you expect consumers of your API to only await them directly, b) allocation-related overhead is important to avoid for your API, and c) either you expect synchronous completion to be a very common case, or you’re able to effectively pool objects for use with asynchronous completion. When adding abstract, virtual, or interface methods, you also need to consider whether these situations will exist for overrides/implementations of that method.

## What’s Next for ValueTask and ValueTask&lt;TResult>?
For the core .NET libraries, we’ll continue to see new Task / Task&lt;TResult>-returning APIs added, but we’ll also see new ValueTask / ValueTask&lt;TResult>-returning APIs added where appropriate. One key example of the latter is for the new IAsyncEnumerator&lt;T> support planned for .NET Core 3.0. IEnumerator&lt;T> exposes a bool-returning MoveNext method, and the asynchronous IAsyncEnumerator&lt;T> counterpart exposes a MoveNextAsyncmethod. When we initially started designing this feature, we thought of MoveNextAsync as returning a Task&lt;bool>, which could be made very efficient via cached tasks for the common case of MoveNextAsync completing synchronously. However, given how wide-reaching we expect async enumerables to be, and given that they’re based on interfaces that could end up with many different implementations (some of which may care deeply about performance and allocations), and given that the vast, vast majority of consumption will be through await foreach language support, we switched to having MoveNextAsync return a ValueTask&lt;bool>. This allows for the synchronous completion case to be fast but also for optimized implementations to use reusable objects to make the asynchronous completion case low-allocation as well. In fact, the C# compiler takes advantage of this when implementing async iterators to make async iterators as allocation-free as possible.
