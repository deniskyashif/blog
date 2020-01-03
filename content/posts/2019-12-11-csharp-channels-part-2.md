---
title: "C# Channels - Timeout and Cancellation"
date: 2019-12-11T15:01:01+02:00
draft: false
summary: "Explore cancellation and timeout techniques with channels."
url: "/2019/12/11/csharp-channels-part-2"
aliases:
- "/csharp-channels-part-2"
images: 
- "/images/posts/2019-12-11-csharp-channels-part2/featured-image.png"
editLink: "https://github.com/deniskyashif/blog/blob/master/content/posts/2019-12-11-csharp-channels-part-2.md"
tags: ["software-design", "csharp", "concurrency", "dotnet"]
---

This is a continuation of the article on [how to build publish/subscribe workflows in C#](/csharp-channels-part-1) where we learned how to use channels in C#. We also went through some techniques about distributing computations among several workers to make use of the modern-day, multi-core CPUs.

In this article, we'll build on top of that by exploring some cases where we have to "exit" a channel. In the last example, we'll put what we've covered so far in practice. You can check out the interactive version of the demos on [GitHub](https://github.com/deniskyashif/trydotnet-channels).

## Timeout

We want to stop reading from a channel after a certain amount of time. This is quite simple because the channels' async API supports cancellation.

<img src="/images/posts/2019-12-11-csharp-channels-part2/timeout-sketch.png" width="600" />

```csharp
var joe = CreateMessenger("Joe", 10);
var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(5));

try
{
    await foreach (var item in joe.ReadAllAsync(cts.Token))
        Console.WriteLine(item);

    Console.WriteLine("Joe sent all of his messages."); 
}
catch (OperationCanceledException)
{
    Console.WriteLine("Joe, you are too slow!");
}
```
```
Joe 0
Joe 1
Joe 2
Joe 3
Joe, you are too slow!
```

Joe was set to send 10 messages but over 5 seconds, we received only 4 and then canceled the reading operation. If we reduced the number of messages Joe sends or sufficiently increased the timeout duration, we'd read everything and thus avoid ending up in the `catch` block.

## Quit Channel

<img src="/images/posts/2019-12-11-csharp-channels-part2/quit-channel-sketch.png" width="600" />

Let's go the other way around and tell Joe to stop talking. We need to modify our `CreateMessenger<T>()` generator so it supports cancellation.

```csharp
static ChannelReader<string> CreateMessenger(
    string msg, int count, CancellationToken token = default)
{
    var ch = Channel.CreateUnbounded<string>();

    Task.Run(async () =>
    {
        var rnd = new Random();
        for (int i = 0; i < count; i++)
        {
            if (token.IsCancellationRequested)
            {
                await ch.Writer.WriteAsync($"{msg} says bye!");
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

Now we need to pass our cancellation token to the generator which gives us control over the channel's longevity.

```csharp
var cts = new CancellationTokenSource();
var joe = CreateMessenger("Joe", 10, cts.Token);
cts.CancelAfter(TimeSpan.FromSeconds(5));

await foreach (var item in joe.ReadAllAsync())
    Console.WriteLine(item);
```

Joe had 10 messages to send, but we gave him only 5 seconds, for which he managed to send only 4. We can also manually send a cancellation request, for example, after reading `N` number of messages.

```
Joe 0
Joe 1
Joe 2
Joe 3
Joe says bye!
```

## Web Search

We're given the task to query several data sources and mix the results. The queries should run concurrently and we should disregard the ones taking too long. Also, we should handle a query response at the time of arrival, instead of waiting for all of them to complete. Let's see how we can solve this using what we've learned so far:

```cs
var ch = Channel.CreateUnbounded<string>();
```

We create a local function that simulates a remote async API call and writes its result to the channel.
```cs
async Task Search(string source, string term, CancellationToken token)
{
    await Task.Delay(TimeSpan.FromSeconds(new Random().Next(5)), token);
    await ch.Writer.WriteAsync($"Result from {source} for {term}", token);
}
```

Now we query several data sources by passing a search term and a cancellation token.

```cs
var term = "Milky Way";
var token = new CancellationTokenSource(TimeSpan.FromSeconds(3)).Token;

var search1 = Search("Google", term, token);
var search2 = Search("Quora", term, token);
var search3 = Search("Wikipedia", term, token);
```

We wait and process the results one at a time. We break from the loop once the cancellation kicks in.

```cs
try
{
    for (int i = 0; i < 3; i++)
        Console.WriteLine(await ch.Reader.ReadAsync(token));

    Console.WriteLine("All searches have completed.");
}
catch (OperationCanceledException)
{
    Console.WriteLine("Timeout.");
}

ch.Writer.Complete();
```

Depending on the timeout interval we might end up receiving responses for all of the queries,

```
[9:09:14 AM] Result from Google for Milky Way
[9:09:14 AM] Result from Wikipedia for Milky Way
[9:09:16 AM] Result from Quora for Milky Way
All searches have completed.
```

or cut off the ones that are too slow.

```
[9:09:19 AM] Result from Quora for Milky Way
[9:09:20 AM] Result from Wikipedia for Milky Way
Timeout.
```

Again - our code is non-blocking concurrent, thus there's no need to use locks, callbacks or any kind of conditional statements.

## References
- [GitHub Repo](https://github.com/deniskyashif/trydotnet-channels) with the interactive examples
- Part 1: [C# Channels - Publish / Subscribe workflows](/csharp-channels-part-1)
<!-- - Part 3: [C# Channels - Async Data Pipelines](/csharp-channels-part-3) -->
- The graphics are implemented using [sketch.io](https://sketch.io/sketchpad/)
