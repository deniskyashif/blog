---
title: "C# Channels - Streaming Data Pipelines"
date: 2019-12-13T10:42:15+02:00
draft: false
url: "csharp-channels-part-3"
tags: ["software-design", "csharp", "concurrency", "dotnet"]
summary: "How to implement an \"assembly line\" concurrency model using channels."
---

In this article we'll learn how to efficiently process data in a non-blocking way using the pipeline pattern. We'll see how to construct flexible and testable pipelines using C#'s channels, as well as how to perform cancellation and deal with errors.
    

## Pipelines

A pipeline is a concurrency model which consists of a sequence of stages. Each stage performs a part of the full job and when it's done, it forwards it to the next stage. It also runs in its own thread and **shares no state** with the other stages.

<img src="/images/posts/csharp-channels-part3/pipeline.png" />

The generator delegates the jobs, which are being processed through the pipeline. For example, we can represent the pizza preparation as a pipeline, consisting of the following stages:

0. Pizza order (Generator) 
1. Prepare dough
2. Add toppings
3. Bake in oven
4. Put in a box

In this concurrency model, each stage executes simultaneously, that is, when stage 2 adds toppings to pizza 1, stage 1 can prepare the dough for pizza 2, when stage 4 puts the baked pizza 1 in a box, stage 1 might be praparing the dough for pizza 3 and so on.

## Implementing a Channel-based Pipeline

Each pipeline starts with a generator function, which initiates jobs by passing it to the stages. Each stage is also a function which runs on its own thread. The communication between the stages in a pipeline is performed using channels. A stage takes a channel as an input, reads from it asynchronously, performs some work and passes data to an output channel.

<img src="/images/posts/csharp-channels-part3/pipeline-channels.png" />

To see it in action, we're going to design a program that efficiently count the lines of source code in a project.

### The Generator - Enumerate the Files

The initial stage of our pipeline would be to recursively enumerate the files in a workspace. We imeplement it as a generator.

```cs
ChannelReader<string> GetFilesRecursively(string root)
{
    var output = Channel.CreateUnbounded<string>();
    
    void WalkDir(string path)
    {
        foreach (var file in Directory.GetFiles(path))
            output.Writer.WriteAsync(file);

        foreach (var dir in Directory.GetDirectories(path))
            WalkDir(dir);
    }

    Task.Run(() =>
    {
        WalkDir(root);
        output.Writer.Complete();
    });

    return output;
}
```

We perform a simple depth first traversal of the workspace tree and write each file we enounter to the output channel. When we're done with the traversal, we mark the channel as complete so the consumer (the next stage) knows when to stop reading from it.

### Stage 1 - Keep the Source Files

Stage 1 is going to determine whether the file contains source code or not. The ones that are not should be discarded. It is pretty much a filtering function.

```cs
ChannelReader<FileInfo> FilterByExtension(
    ChannelReader<string> input, HashSet<string> exts)
{
    var output = Channel.CreateUnbounded<FileInfo>();
    
    Task.Run(async () =>
    {
        await foreach (var file in input.ReadAllAsync())
        {
            var fileInfo = new FileInfo(file);
            if (exts.Contains(fileInfo.Extension))
                await output.Writer.WriteAsync(fileInfo);
        }
        output.Writer.Complete();
    });

    return output;
}
```

This stage takes an input channel (produced by the generator) from which it asynchronously reads the file names. For each file, it gets its metadata and checks whether the extension belongs to the set of the source code file extensions. It also transforms the input, that is, for each file that satisfies the condition, it writes a `FileInfo` object to the output channel.

### Stage 2 - Get the Line Count

This stage is responsible for determining the number of lines in each file.

```cs
static ChannelReader<(FileInfo file, int lines)> 
    GetLineCount(ChannelReader<FileInfo> input)
{
    var output = Channel.CreateUnbounded<(FileInfo, int)>();

    Task.Run(async () =>
    {
        await foreach (var file in input.ReadAllAsync())
        {
            var lines = CountLines(file);
            await output.Writer.WriteAsync((file, lines));
        }
        output.Writer.Complete();
    });

    return output;
}
```

We write a tuple of type `(FileInfo, int)` which contains the file info and the number of its lines of code. It's obvious what `int CountLines(FileInfo file)` does. You can check out the implementation below.

<details>
  <summary>Expand <code>CountLines</code></summary>

```cs
int CountLines(FileInfo file)
{
    using var sr = new StreamReader(file.FullName);
    var lines = 0;
    
    while (sr.ReadLine() != null)
        lines++;

    return lines;
}
```

</details>

## The Sink Stage

Now we've implemented the stages of our pipeline, we are ready to put them all together.

```cs
var fileGen = GetFilesRecursively("node-modules");
var sourceCodeFiles = FilterByExtension(
    fileGen, new HashSet<string> { ".js", ".ts" });
var counter = GetLineCount(sourceCodeFiles);
```

The sink stage is the one which consumes the final output from the pipeline. Unline the generator, which doesn't consume, but only produces, the sink only consumes, but doesn't produce. That's where our pipeline comes to an end.

```cs
var totalLines = 0;
await foreach (var item in counter.ReadAllAsync())
{
    Console.WriteLine($"{item.file.FullName} {item.lines}");
    totalLines += item.lines;
}

Console.WriteLine($"Total lines: {totalLines}");
```

## Error Handling

We've covered the happy path, however, our pipeline has to be able deal with malformed or erroneous inputs. Each stage has an own notion of what an invalid input is, so it has an own responsibility to process such. To achieve a good error handling, our pipeline needs to satisfy the following:

1. An invalid input should not propagate to the next stage
2. An invalid input should not cause the pipeline to stop. The pipeline should continue to process the inputs thereafter.
3. The pipeline should not swallow errors. All the errors should be reported.

We're going to modify stage 2 which counts the lines in a file. Our definition for an invalid input is an empty file. Our pipeline should not pass them forward and instead, it should report the existence of such files. We solve that by introducing a second channel - a one which will contain the errors.

``` diff
- static ChannelReader<(FileInfo file, int lines)> GetLineCount(
+ static (ChannelReader<(FileInfo file, int lines)> output,
+         ChannelReader<string> errors)
    GetLineCount(ChannelReader<FileInfo> input)
    {
        var output = Channel.CreateUnbounded<(FileInfo, int)>();
+       var errors = Channel.CreateUnbounded<string>();

        Task.Run(async () =>
        {
            await foreach (var file in input.ReadAllAsync())
            {
                var lines = CountLines(file);
-               await output.Writer.WriteAsync((file, lines));
+               if (lines == 0)
+                   await errors.Writer.WriteAsync(
+                       $"[Error] Empty file {file}");
+               else
+                   await output.Writer.WriteAsync((file, lines));
            }
            output.Writer.Complete();
+           errors.Writer.Complete();
        });
-       return output;
+       return (output, errors);
    }
```

We've created a second channel for errors and changed the signature of the stage so it returns both the output and the error channels. Empty files are not passed to the next stage, we've provided a mechanism to report them using the error channel and after we get an invalid input, our pipeline continues processing the next ones.

```cs
...
var (counter, errors) = GetLineCount(sourceCodeFiles);
var totalLines = 0;

await foreach (var item in counter.ReadAllAsync())
    totalLines += item.lines;

Console.WriteLine($"Total lines: {totalLines}");

await foreach (var errMessage in errors.ReadAllAsync())
    Console.WriteLine(errMessage);
```

## Cancellation

Similarily to the error handling, the stages being independent means that each has to handle cancellation on their own. To stop the pipeline, we need to prevent the generator from delegating new jobs. Let's make it cancellable.

```cs
static ChannelReader<string> GetFilesRecursively(
    string root, CancellationToken token = default)
{
    var output = Channel.CreateUnbounded<string>();

    void WalkDir(string path)
    {
        if (token.IsCancellationRequested)
            throw new OperationCanceledException();
        foreach (var file in Directory.GetFiles(path))
            output.Writer.WriteAsync(file, token);
        foreach (var dir in Directory.GetDirectories(path))
            WalkDir(dir);
    }

    Task.Run(() =>
    {
        try
        {
            WalkDir(root);
        }
        catch (OperationCanceledException) { Console.WriteLine("Cancelled."); }
        finally { output.Writer.Complete(); }
    });

    return output;
}
```

The change is straightforward, we need to handle the cancellation in a `try/catch` and not forget to close the output channel. Keep in mind that cancelling only the initial stage will leave the next stages running with the existing jobs which might not be desired (especially if the stages are long running). Therefore, we have to pass the cancellation token to each of them and make them able to handle cancellation as well.

```cs
var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(0.5));
var fileSource = GetFilesRecursively("node-modules", cts.Token);
...
```
 
## Dealing with Bottlenecks

Our stages execute concurrently but this doesn't guarantee optimal performance. Remember the pizza preparation - it takes longer time to bake the pizza than to add the toppings. This becomes an issue when we have to process with many pizza orders as we're going to end up with many pizzas with their toppings added, waiting to be baked, but our oven fits in only one at a time. We solve this by getting a larger oven, or even multiple ovens.

<img src="/images/posts/csharp-channels-part3/bottleneck.png" />

In the line counter example, the stage where we read the file and count its lines might cause a bottleneck when a file is sufficiently large. It makes sense to increase the capacity of this stage and that's where `Split<T>` and `Merge<T>` which we discussed in [part 1](/csharp-channels-part-1) come into use. We'll summarize them briefly.

### Split

`Split<T>` is a function that takes an input channel and distributes its messages amongst several outputs. That way we can let several threads handle the message processing.

<img src="/images/posts/csharp-channels-part3/split.png" />

<details>
  <summary>Expand <code>Split&lt;T&gt;</code> implementation</summary>

```cs
IList<ChannelReader<T>> Split<T>(ChannelReader<T> input, int n)
{
    var outputs = new Channel<T>[n];
    for (var i = 0; i < n; i++)
        outputs[i] = Channel.CreateUnbounded<T>();

    Task.Run(async () =>
    {
        var index = 0;
        await foreach (var item in input.ReadAllAsync())
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
</details>
<br />

We're going to use it to distribute the source code files among 5 channels which will let us process up to 5 files simultaneously.

```cs
var fileSource = GetFilesRecursively("node-modules");
var sourceCodeFiles =
    FilterByExtension(fileSource, new HashSet<string> {".js", ".ts" });
var splitter = Split(sourceCodeFiles, 5);
```

### Merge

`Merge<T>` is the opposite operation. It takes multiple input channels and consolidates them in a single output.

<img src="/images/posts/csharp-channels-part3/merge.png" />

<details>
  <summary>Expand <code>Merge&lt;T&gt;</code> implementation</summary>

```cs
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
</details>
<br />

During `Merge<T>`, we read concurrently from several channels, so this is the stage where we need to tweak it a little bit and perform the line counting.

We introduce, `CountLinesAndMerge` which doesn't only redirect, but also transforms.

```cs {hl_lines=[11]}
static ChannelReader<(FileInfo file, int lines)>
    CountLinesAndMerge(IList<ChannelReader<FileInfo>> inputs)
{
    var output = Channel.CreateUnbounded<(FileInfo file, int lines)>();

    Task.Run(async () =>
    {
        async Task Redirect(ChannelReader<FileInfo> input)
        {
            await foreach (var file in input.ReadAllAsync())
                await output.Writer.WriteAsync((file, CountLines(file)));
        }
            
        await Task.WhenAll(inputs.Select(Redirect).ToArray());
        output.Writer.Complete();
    });

    return output;
}
```

The error handling and the cancellation are omitted for the sake of brevity, however, we've already seen how to implement them. Now we're ready to build our pipeline.

```diff
var fileSource = GetFilesRecursively("node-modules");
var sourceCodeFiles =
    FilterByExtension(fileSource, new HashSet<string> {".js", ".ts"});
- var counter = GetLineCount(sourceCodeFiles);
+ var splitter = Split(sourceCodeFiles, 5);
+ var counter = CountLinesAndMerge(splitter);
```

## TPL Dataflow

TPL Dataflow is another option for implementing data processing pipelines or even meshes in .NET. It has a declarative and a higher-level API compared to the channel-based approach, but it also comes with more complexity and provides less control. I haven't done an extensive performance comparison yet, but I think that deciding between the two should strongly depend on the case. If you want a simple API and more control, the lightweight channels would be the way to go. If you want a higher-level API with more features, check out TPL Dataflow.

## Conclusion

We defined the pipeline concurrency model and learned how to use it to implement efficient and flexible data processing pipelines. We learned how to deal with errors, perform cancellation as well as how to apply some of the concurrency techniques (multiplexing and demultiplexing), described in the previous articles, to deal with potential bottlenecks.

Besides performance, pipelines also make it very easy to be modified. Each stage is an atomic part of the whole composition which can be independently modified, replaced or removed as long as we keep the method signatures intact.

It is easy to see how it can lead to significant reduction of our code's cyclomatic complexity as well as making it easier to test. Each stage is simply a method with no side effects, which can be unit tested in isolation. Stages have a **single responsibility**, which makes them easier to reason about thus we can cover all the possible cases in our unit tests.

## References & Further Reading

- [GitHub Repo](https://github.com/deniskyashif/trydotnet-channels) with the interactive examples
- Part 1: [C# Channels - Publish / Subscribe workflows](/csharp-channels-part-1)
- Part 2: [C# Channels - Timeout and Cancellation](/csharp-channels-part-2)
- The graphics are implemented using [Lucidchart](https://www.lucidchart.com/)
