---
title: "C# Channels - Publish / Subscribe Workflows"
date: 2019-12-08T08:02:09+02:00
draft: false
summary: "Concurrency patterns in .NET using channels."
url: "/2019/12/08/csharp-channels-part-1"
aliases:
- "/csharp-concurrency-patterns-using-channels"
- "/csharp-channels-part-1"
images:
- "/images/posts/2019-12-08-csharp-channels-part1/channel-sketch-featured.png"
editLink: "https://github.com/deniskyashif/blog/blob/master/content/posts/2019-12-08-csharp-channels-part-1.md"
tags: ["software-design", "csharp", "concurrency", "dotnet"]
---

In this article, we'll explore the synchronization data structures in .NET's `System.Threading.Channels` namespace and learn how to use them for designing concurrent workflows. It would be helpful to have some basic understanding of .NET's [Task Parallel Library
(TPL)](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/task-parallel-library-tpl), but it's in no means necessary.

Recently, I watched Rob Pike's [talk on "Go Concurrency Patterns"](https://www.youtube.com/watch?v=f6kdp27TYZs) where he explains Go's approach to concurrency and demonstrates some of its features for building concurrent programs. I found its simplicity and ease of use fascinating and went on to implement some of these techniques in C#. Let's start by introducing some definitions.

## Concurrency

The relationship between concurrency and parallelism is commonly misunderstood. In fact, two procedures being concurrent doesn't mean that they'll run in parallel. The following quote by Martin Kleppmann has stood out in my mind when it comes to concurrency:

> For defining concurrency, the exact time doesn't matter: we simply call two operations concurrent if they **are both unaware of each other**, regardless of the physical time at which they occurred.
>
> -- _"Designing Data-Intensive Applications"_ by Martin Kleppmann

Concurrency is something that **enables parallelism**. On a single processor, two procedures can be concurrent, yet they won't run in parallel. A concurrent program **deals** with a lot of things at once, whereas a parallel program **does** a lot of things at once.  
Think of it this way - concurrency is about **structure** and parallelism is about **execution**. A concurrent program may benefit from parallelism, but that's not its goal. The goal of concurrency is a good structure.

## Channels

A concurrent program is structured into independent pieces that we have to coordinate. To make that work, we need some form of communication. There're several ways to achieve that in .NET. In this article, we'll explore the `System.Threading.Channels` (ships with .NET Core 3.1 SDK or available as a [NuGet package](https://www.nuget.org/packages/System.Threading.Channels/)) which provides an API, analogous to Go's built-in channel primitive.

<img src="/images/posts/2019-12-08-csharp-channels-part1/channel-sketch.png" width="600">

A channel is a data structure that allows one thread to communicate with another thread. In .NET, this was usually done by using a shared variable that supports concurrency (by implementing some synchronization/locking mechanism). Channels, on the other hand, can be used to send messages directly between threads without any external synchronization or locking required. These messages are sent in FIFO (first in first out) order. Here's how we create a channel:

```csharp
Channel<string> ch = Channel.CreateUnbounded<string>();
```

`Channel` is a static class that exposes several factory methods for creating channels. `Channel<T>` is a data structure that supports reading and writing. That's how we write asynchronously to a channel:

```csharp
await ch.Writer.WriteAsync("My first message");
await ch.Writer.WriteAsync("My second message");
ch.Writer.Complete();
```

This is how we read from a channel:

```csharp
while (await ch.Reader.WaitToReadAsync())
    Console.WriteLine(await ch.Reader.ReadAsync());
```

The reader's `WaitToReadAsync()` will complete with `true` when data is available to read, or with `false` when no further data will ever be read, that is, after the writer invokes `Complete()`. The reader also provides an option to consume the data as an async stream by exposing a method that returns `IAsyncEnumerable<T>`:

```csharp
await foreach (var item in ch.Reader.ReadAllAsync())
    Console.WriteLine(item);
```

Channels also have a blocking API which we won't cover in this article.

### Using Channels

Here's a basic example when we have a separate producer and consumer threads which communicate through a channel.

```csharp
var ch = Channel.CreateUnbounded<string>();

var consumer = Task.Run(async () =>
{
    while (await ch.Reader.WaitToReadAsync())
        Console.WriteLine(await ch.Reader.ReadAsync());
});
var producer = Task.Run(async () =>
{
    var rnd = new Random();
    for (int i = 0; i < 5; i++)
    {
        await Task.Delay(TimeSpan.FromSeconds(rnd.Next(3)));
        await ch.Writer.WriteAsync($"Message {i}");
    }
    ch.Writer.Complete();
});

await Task.WhenAll(producer, consumer);
```
```sh
[12:27:16 PM] Message 0
[12:27:18 PM] Message 1
[12:27:19 PM] Message 2
[12:27:20 PM] Message 3
[12:27:22 PM] Message 4
```

The consumer (reader) waits until there's an available message to read. On the other side, the producer (writer) waits until it's able to send a message, hence, we say that **channels both communicate and synchronize**. Both operations are non-blocking, that is, while we wait, the thread is free to do some other work.  
Notice that we have created an **unbounded** channel, meaning that it accepts as many messages as it can with regards to the available memory. With **bounded** channels, however, we can limit the number of messages that can be processed at a time. 

```csharp
var ch = Channel.CreateBounded<string>(capacity: 10);
```

So when this limit is reached, `WriteAsync()` won't be able to write, until there's an available slot in the channel's buffer. A slot is freed up when a consumer reads from the channel.

## Concurrency Patterns

> Don't communicate by sharing memory, share memory by communicating.

It's time to explore a few concurrent programming techniques for working with channels. This part consists of several examples that are independent of each other. You can also find the interactive version of them on [GitHub](https://github.com/deniskyashif/trydotnet-channels).

### The Generator

A generator is a method that returns a channel. The one below creates a channel and writes a given number of messages asynchronously from a separate thread.

```csharp
static ChannelReader<string> CreateMessenger(string msg, int count)
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

By returning a `ChannelReader<T>` we ensure that our consumers won't be able to attempt writing to it.

```csharp
var joe = CreateMessenger("Joe", 5);
await foreach (var item in joe.ReadAllAsync())
    Console.WriteLine(item);
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
    Console.WriteLine(await joe.ReadAsync());
    Console.WriteLine(await ann.ReadAsync());
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

This approach is problematic in several ways. Suppose Ann sends more messages than Joe:

```csharp
var joe = CreateMessenger("Joe", 2);
var ann = CreateMessenger("Ann", 5);
```

We're still going to try and read from Joe, even when his channel is completed which is going to throw an exception.

```sh
[8:05:01 AM] Joe 0
[8:05:01 AM] Ann 0
[8:05:02 AM] Joe 1
[8:05:02 AM] Ann 1
Unhandled exception. System.Threading.Channels.ChannelClosedException:
    The channel has been closed.
```

A quick and dirty solution would be to wrap it in a try/catch block:

```csharp
try
{
    Console.WriteLine(await joe.ReadAsync());
    Console.WriteLine(await ann.ReadAsync());
}
catch (ChannelClosedException) { }
```

Our code is concurrent, but not optimal because it executes in a lockstep. Suppose, Ann is more talkative than Joe, so her messages have an up to 3 seconds delay, whereas Joe sends messages on up to every 10 seconds. This will force us to wait for Joe, even though we might have several messages waiting ready to be read from Ann. Currently, we cannot read from Ann before reading from Joe. We should be doing better!

### Multiplexer

We want to read from both Joe and Ann and process whoever's message arrives first. We're going to solve this by consolidating their messages into a single channel. Let's define our `Merge<T>()` method:

```csharp
static ChannelReader<T> Merge<T>(
    ChannelReader<T> first, ChannelReader<T> second)
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

    return output;
}
```

`Merge<T>()` takes two channels and starts reading from them simultaneously. It creates and immediately returns a new channel which consolidates the outputs from the input channels. The reading procedures are run asynchronously on separate threads. Think of it like this:

<img src="/images/posts/2019-12-08-csharp-channels-part1/merge-sketch.png" width="600" />

That's how we use it.

```csharp
var ch = Merge(CreateMessenger("Joe", 3), CreateMessenger("Ann", 5));

await foreach (var item in ch.ReadAllAsync())
    Console.WriteLine(item);
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

We have simplified our code and solved the problem with Ann sending more messages than Joe while also doing it more often. However, you might have noticed that `Merge<T>()` has a defect - the output channel's writer never closes which leads us to continue waiting to read, even when Joe and Ann have finished sending messages. Also, there's no way for us to handle a potential failure of any of the readers. We have to modify our code:

```csharp
static ChannelReader<T> Merge<T>(params ChannelReader<T>[] inputs)
{
    var output = Channel.CreateUnbounded<T>();

    Task.Run(async () =>
    {
        async Task Redirect(ChannelReader<T> input)
        {
            await foreach (var item in input.ReadAllAsync())
                await output.Writer.WriteAsync(item);
    	}

        await Task.WhenAll(inputs.Select(i => Redirect(i)).ToArray());
        output.Writer.Complete();
    });

    return output;
}
```

Our `Merge<T>()` now also works with an arbitrary number of inputs.
We've created the local asynchronous `Redirect()` function which takes a channel as an input writes its messages to the consolidated output. It returns a `Task` so we can use `WhenAll()` to wait for the input channels to complete. This allows us to also capture potential exceptions. In the end, we know that there's nothing left to be read, so we can safely close the writer.

Our code is concurrent and non-blocking. The messages are being processed at the time of arrival and there's no need to use locks or any kind of conditional logic. While we're waiting, the thread is free to perform other work. We also don't have to handle the case when one of the writers complete (as you can see Ann has sent all of her messages before Joe).

### Demultiplexer

Joe talks too much and we cannot handle all of his messages. We want to distribute the work amongst several consumers. Let's define `Split<T>()`:

<img src="/images/posts/2019-12-08-csharp-channels-part1/split-sketch.png" width="600" />

```csharp
static IList<ChannelReader<T>> Split<T>(ChannelReader<T> ch, int n)
{
    var outputs = new Channel<T>[n];

    for (int i = 0; i < n; i++)
        outputs[i] = Channel.CreateUnbounded<T>();

    Task.Run(async () =>
    {
        var index = 0;
        await foreach (var item in ch.ReadAllAsync())
        {
            await outputs[index].Writer.WriteAsync(item);
            index = (index + 1) % n;
        }

        foreach (var ch in outputs)
            ch.Writer.Complete();
    });

    return outputs.Select(ch => ch.Reader).ToArray();
}
```

`Split<T>` takes a channel and redirects its messages to `n` number of newly created channels in a round-robin fashion. Here's how to use it:

```csharp
var joe = CreateMessenger("Joe", 10);
var readers = Split<int>(joe, 3);
var tasks = new List<Task>();

for (int i = 0; i < readers.Count; i++)
{
    var reader = readers[i];
    var index = i;
    tasks.Add(Task.Run(async () =>
    {
        await foreach (var item in reader.ReadAllAsync())
            Console.WriteLine($"Reader {index}: {item}");
    }));
}

await Task.WhenAll(tasks);
```

Joe sends 10 messages which we distribute amongst 3 channels. Below is a possible output we can get. Some channels may take longer to process a message, therefore, we have no guarantee that the order of emission is going to be preserved. Our code is structured so that we process (in this case log) a message as soon as it arrives.

```sh
Reader 0: Joe 0
Reader 1: Joe 1
Reader 0: Joe 3
Reader 2: Joe 2
Reader 1: Joe 4
Reader 2: Joe 5
Reader 0: Joe 6
Reader 1: Joe 7
Reader 2: Joe 8
Reader 0: Joe 9
```

**NB**: To keep things simple, in `Split<T>` and `Merge<T>` examples we ended up with uncollected tasks. In production scenarios you'd want to ensure they're also collected. A way to do it would be to pass those tasks back to the caller to be awaited on. Ensure that awaiting those tasks are concurrent with the reading from the multiplexed/demultiplexed channels.

## Conclusion

In this article, we defined the term concurrency and discussed how it relates to parallelism. We explained why the two terms should not be confused. Then we explored C#'s channel data structure and learned how to use it to implement publish/subscribe workflows. We've seen how to make an efficient use multiple CPUs by distrubuting the reading/writing operations amongst several workers.

_Check out [part 2](/csharp-channels-part-2) where we discuss some cancellation techniques and [part 3](/csharp-channels-part-3) where we put what we've learned into practice._

## References & Further Reading

- [GitHub Repo](https://github.com/deniskyashif/trydotnet-channels) with the interactive examples
- Part 2: [C# Channels - Timeout and Cancellation](/csharp-channels-part-2)
- Part 3: [C# Channels - Async Data Pipelines](/csharp-channels-part-3)
- [An Introduction to System.Threading.Channels](https://devblogs.microsoft.com/dotnet/an-introduction-to-system-threading-channels/) on the .NET Blog
- [Concurrency in C# Cookbook](https://stephencleary.com/book/) by Stephen Cleary - my highly recommended go-to reference for concurrent C#
- The graphics are implemented using [sketch.io](https://sketch.io/sketchpad/)
