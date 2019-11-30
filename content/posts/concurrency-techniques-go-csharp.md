---
title: "Concurrency Patterns with Channels in C#"
date: 2019-11-30T16:38:09+02:00
draft: true
summary: "Working with channels and async data streams."
---

Recently, I watched Rob Pike's [talk on "Go Concurrency Patterns"](https://www.youtube.com/watch?v=f6kdp27TYZs) where he explains Go's approach and demonstrates some of its features for building concurrent programs. I found its simplicity and ease of use fascinating, and decided to do my own experiment by implementing some of these techniques in C#.

In this article, we'll explore the synchronization data structures in .NET's `System.Threading.Channels` namespace and learn how to structure concurrent workflows. You don't need to watch the talk to follow through the article, however, some basic understanding of the [Task Parallel Library
(TPL)](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl) in .NET would be helpful. 

We'll start by introducing some definitions.

## Concurrency

The relationship between concurrency and parallelism is commonly misunderstood. In fact, two procedures being concurrent doesn't mean that they'll run in parallel. The following quote by Martin Kleppmann has stood out in my mind when it comes to concurrency:

> For defining concurrency, the exact time doesn't matter: we simply call two operations concurrent if they **are both unaware of each other**, regardless of the physical time at which they occurred.
>
> -- _"Designing Data-Intensive Applications"_ by Martin Kleppmann

Concurrency is something that **enables parallelism**. On a single processor, two procedures can be concurrent, yet they won't be executed in parallel. A concurrent program **deals** with a lot of things at once, whereas a parallel program **does** a lot of things at once.  
Think of it this way - concurrency is about **structure** and parallelism is about **execution**. A concurrent program may benefit from parallelism, but that's not its goal. The goal of concurrency is a good structure.

## Channels

A concurrent program is structured into independent pieces that we have to coordinate. To make that work, we need some form of communication. There're several ways to achieve that with C#. In this article, we'll explore `System.Threading.Channels` (currently a NuGet package) which provides an API, analogous to Go's built in channel primitive.

A channel is a data structure which allows one thread to communicate with another one. By commucate we mean sending and receiving data usually in an asynchronous way. This is how we create a channel:

```csharp
Channel<string> ch = Channel.CreateUnbounded<string>();
```

`Channel` is a static class which exposes several factory methods for creating channels. `Channel<T>` is the data structure that supports reading and writing. That's how we write to a channel:

```csharp
await ch.Writer.WriteAsync("My first message");
await ch.Writer.WriteAsync("My second message");
ch.Writer.Complete();
```

And this is how we read from a channel.

```csharp
while (await ch.Reader.WaitToReadAsync()) 
    Console.WriteLine(await ch.Reader.ReadAsync());
```

The reader's `WaitToReadAsync()` will complete with `true` when data is available to read, or with `false` when no further data will ever be read, that is, after the writer's `Complete()` is called. `ChannelReader<T>` also provides an option to read the data as an async stream by exposing a method returning `IAsyncEnumerable<T>`:

```csharp
await foreach (var item in ch.Reader.ReadAllAsync())
    Console.WriteLine(item);
```

### Using Channels

Here we have a separate thread writing values asynchronously to the channel every 0 to 3 seconds and a reader that consumes these values.

```csharp
static async Task Main()
{
    var ch = Channel.CreateUnbounded<string>();
    var rnd = new Random();

    Task.Run(async () =>
    {
        for (int i = 0; i < 5; i++)
        {
            await Task.Delay(TimeSpan.FromSeconds(rnd.Next(3)));
            await ch.Writer.WriteAsync($"#{i} message");
        }
        ch.Writer.Complete();
    });

    while (await ch.Reader.WaitToReadAsync()) 
        Console.WriteLine(await ch.Reader.ReadAsync());
}
```

When the `Main()` method executes, the reader waits for a value to be sent. On the other hand, the writer waits until it's able to send a value, hence, we say that **channels both communicate and synchronize**.   
Notice that we have created an **unbounded** channel, meaning that it accepts as many messages as it can with regards to the available memory. With **bounded** channels, however, we can limit the amount of messages it can process at a time. So when this limit is reached, `WriteAsync()` won't be able to write, until there's an available slot.

## Concurrency Patterns

Now it's time to common techniques when working with channels.

### The Generator

A generator is a just a method that returns a channel.

```csharp
static ChannelReader<string> CreateMessenger(string msg, int count = 5)
{
    var ch = Channel.CreateUnbounded<string>();
    var rnd = new Random();

    Task.Run(async () =>
    {
        for (int i = 0; i < count; i++)
        {
            await ch.Writer.WriteAsync($"{msg} {i}");
            await Task.Delay(TimeSpan.FromSeconds(rnd.Next(3)));
        }
        ch.Writer.Complete();
    });

    return ch.Reader;
}
```

By returning a `ChannelReader` we ensure that our consumers won't be able to write to the channel.

```csharp
static async Task Main()
{
    var joe = CreateMessenger("Joe");
    await foreach (var item in joe.Reader.ReadAllAsync())
        Console.WriteLine($"[{DateTime.UtcNow.ToLongTimeString()}] {item}");
}
```
```sh
[7:31:39 AM] Joe 0
[7:31:40 AM] Joe 1
[7:31:42 AM] Joe 2
[7:31:44 AM] Joe 3
[7:31:44 AM] Joe 4
```

Let's try to read from multiple channels.

```csharp
var joe = CreateMessenger("Joe", 3);
var ann = CreateMessenger("Ann", 3);

while (await joe.WaitToReadAsync() || await ann.WaitToReadAsync())
{
    var messageFromJoe = await joe.ReadAsync();
    Console.WriteLine(
        $"[{DateTime.UtcNow.ToLongTimeString()}] {messageFromJoe}");
    var messageFromAnn = await ann.ReadAsync();
    Console.WriteLine(
        $"[{DateTime.UtcNow.ToLongTimeString()}] {messageFromAnn}");
}
```
```sh
[8:00:51 AM] Joe 0
[8:00:51 AM] Ann 0
[8:00:52 AM] Joe 1
[8:00:52 AM] Ann 1
[8:00:54 AM] Joe 2
[8:00:54 AM] Ann 2
```

This approach is clumsy and problematic in several ways. Suppose Ann sends more messages than Joe:

```csharp
var joe = CreateMessenger("Joe", 2);
var ann = CreateMessenger("Ann", 4);
```

We're still going to try and read from Joe, even when his channel is completed.

```sh
[8:05:01 AM] Joe 0
[8:05:01 AM] Ann 0
[8:05:02 AM] Joe 1
[8:05:02 AM] Ann 1
Unhandled exception. System.Threading.Channels.ChannelClosedException: The channel has been closed.
```

So catching the exception should solve the problem?

```csharp
try
{
    Console.WriteLine(
        $"[{DateTime.UtcNow.ToLongTimeString()}] {await joe.ReadAsync()}");
    Console.WriteLine(
        $"[{DateTime.UtcNow.ToLongTimeString()}] {await ann.ReadAsync()}");
}
catch (ChannelClosedException) { }
```

Our code is concurrent, but not optimal. Suppose, Ann is more talkative than Joe, so she sends messages on every with an interval of up to 3 second, whereas Joe sends messages on up to every 5 secods. This will force us to wait for Joe, even though Ann has several messages ready to be read, because we cannot read from Ann before reading from Joe. We can do better than that!

### Multiplexing

Let's define our fan-in method for channels.

```csharp
static ChannelReader<T> FanIn<T>(ChannelReader<T> first, ChannelReader<T> second)
{
    var output = Channel.CreateUnbounded<T>();

    Task.Run(async () =>
    {
        await foreach (var item in first.ReadAllAsync())
            await output.Writer.WriteAsync(item);
    });
    Task.Run(async () =>
    {
        await foreach (var item in second.ReadAllAsync())
            await output.Writer.WriteAsync(item);
    });

    return ch;
}
```

`FanIn<T>()` takes two channels and starts reading from them simultaneously. It creates and returns a new channel which consolidates the outputs from the input channels. Think of it like this:

[FAN-IN PICTURE]

```csharp
static async Task Main(string[] args)
{
    var ch = FanIn(CreateMessenger("Joe", 3), CreateMessenger("Ann", 5));

    await foreach (var item in ch.ReadAllAsync())
        Console.WriteLine($"[{DateTime.UtcNow.ToLongTimeString()}] {item}");
}
```
```sh
[8:39:32 AM] Ann 0
[8:39:32 AM] Joe 0
[8:39:32 AM] Ann 1
[8:39:33 AM] Ann 2
[8:39:33 AM] Ann 3
[8:39:34 AM] Joe 1
[8:39:35 AM] Ann 4
[8:39:36 AM] Joe 2
```

We have simplified our code and solved the problem with Ann sending more messages than Joe while also doing it more often. We're not blocking anything and thus are reading the messages immediately at the time of arrival. We also don't care if one of the writers complete (as you can see Ann has sent all of her messages faster than Joe), as we simply continue reading from the one that is available.

You might have noticed that `FanIn<T>()` has a defect - the output channel's writer never closes which leads us to continue waiting to read, even when Joe and Ann have finished sending messages. We have to modify `FanIn<T>()`.

```csharp
static ChannelReader<T> FanIn<T>(
    ChannelReader<T> first, 
    ChannelReader<T> second)
{
    var output = Channel.CreateUnbounded<T>();
    Task.Run(async () =>
    {
        async Task WriteToOutput(ChannelReader<T> input)
        {
            await foreach (var item in input.ReadAllAsync())
                await output.Writer.WriteAsync(item);
        }

        await Task.WhenAll(WriteToOutput(first), WriteToOutput(second));
        output.Writer.Complete();
    });

    return output;
}
```

We've created the local asynchronous function `WriteToOutput()` which takes a channel and for each message read - it writes it to the consolidated output. It returns a `Task` so we can run it simultaneously for both of our inputs channels and wait until they complete. After that, we know that there's nothing left to read so we can safely close the writer. All of this runs on a asynchronously, so we return the consolidated channel before any of the input channels has started writing messages.

### Timeout

We want to read from a channel for a certain amount of time.

```csharp
var joe = CreateMessenger("Joe", 10);
var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(5));
try
{
    await Task.Run(async () =>
    {
        await foreach (var item in joe.ReadAllAsync(cts.Token))
            Console.WriteLine(item);

        Console.WriteLine("Joe sent all of his messages.");
    });
}
catch (OperationCanceledException)
{
    Console.WriteLine("Joe, you are too slow!");
}
```
```sh
Joe 0
Joe 1
Joe 2
Joe 3
Joe, you are too slow!
```

In this example, Joe was set to send ten messages but over the course of five seconds, we received only four and then cancelled the reading operation. If we reduce the amount of messages Joe sends or sufficiently increase the timeout duration, we'll read everything and thus avoid ending up in the `catch` block.

### Quit Channel

Let's go the other way around and tell Joe to stop talking. We need to modify our `CreateMessenger()` generator.

```csharp
static ChannelReader<string> CreateMessenger(
    string msg,
    int count = 5,
    CancellationToken token = default(CancellationToken))
{
    var ch = Channel.CreateUnbounded<string>();
    var rnd = new Random();

    Task.Run(async () =>
    {
        for (int i = 0; i < count; i++)
        {
            if (token.IsCancellationRequested)
            {
                await ch.Writer.WriteAsync($"{msg} bye!");
                break;
            }
            await ch.Writer.WriteAsync($"{msg} {i}");
            await Task.Delay(TimeSpan.FromSeconds(rnd.Next(0, 3)));
        }
        ch.Writer.Complete();
    });

    return ch.Reader;
}
```

Now we need to pass our cancellation token to the generator which gives us control over the channel's logevity.

```csharp
var cts = new CancellationTokenSource();
var joe = CreateMessenger("Joe", 10, cts.Token);
cts.CancelAfter(TimeSpan.FromSeconds(5));

await Task.Run(async () =>
{
    await foreach (var item in joe.ReadAllAsync())
        Console.WriteLine(item);
});
```

Joe had 10 messages to send, but we gave him only 5 seconds, for which he managed to send only 4.

```sh
Joe 0
Joe 1
Joe 2
Joe 3
Joe bye!
```

### Web Search

## Using Channels with TPL Dataflow and Rx

## Conclusion

## References & Further Reading

* [C# Job Queues with Reactive Extensions and Channels](https://michaelscodingspot.com/c-job-queues-with-reactive-extensions-and-channels/)

<!-- 
The Go Approach: Don't communicate by sharing memory, share memory by communicating.

No locks

https://talks.golang.org/2012/concurrency.slide
goroutines, channels
multiplexing (the generator pattern)

 -->
