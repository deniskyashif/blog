---
title: "C# Channels - Streaming Data Pipelines"
date: 2019-12-13T10:42:15+02:00
draft: true
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
var fileGen = GetFilesRecursively(".");
var sourceCodeFiles = FilterByExtension(
    fileGen, new HashSet<string> { ".cs", ".ts" });
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

We're going to modify stage 2 which counts the lines in a file. Our definition for an invalid input is an empty file. Our pipeline should not pass them forward and instead, it should report the existence of such files. We solve that by introducing a second channel - a one which will contain errors.

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

## Cancellation

Similarily to the error handling, the stages being independent means that each has to handle cancellation on their own. Let's make `GetLineCount` cancellable.

```diff
static (ChannelReader<(FileInfo file, int lines)> output, 
        ChannelReader<string> errors)
-   GetLineCount(ChannelReader<FileInfo> input)
+   GetLineCount(ChannelReader<FileInfo> input, CancellationToken token = default)
{
    var output = Channel.CreateUnbounded<(FileInfo, int)>();
    var errors = Channel.CreateUnbounded<string>();

    Task.Run(async () =>
    {
+       try
+       {
-           await foreach (var file in input.ReadAllAsync())
+           await foreach (var file in input.ReadAllAsync(token))
            {
                var lines = CountLines(file);
                if (lines == 0)
-                   await errors.Writer.WriteAsync($"[Error] Empty file {file}");
+                   await errors.Writer.WriteAsync($"[Error] Empty file {file}", token);
                else
-                   await output.Writer.WriteAsync((file, lines));
+                   await output.Writer.WriteAsync((file, lines), token);
            }
+       }
-           output.Writer.Complete();
-           errors.Writer.Complete();
+       catch (OperationCanceledException)
+       {
+           await errors.Writer.WriteAsync("Line Counter was cancelled.");
+       }
+       finally
+       {
+           output.Writer.Complete();
+           errors.Writer.Complete();
+       }
    }, token);

    return (output, errors);
}
```

The change is straightforward, we need to handle the cancellation in a `try/catch` and not forget to close the channels no matter what the outcome is. Keep in mind that we have multiple stages so cancelling one of them would eventually lead to stopping the pipeline, but the rest of them will keep executing. This will lead to memory leaks, therefore, it's important to cancel all of the stages in the pipeline, beginning with the generator.
 
## Dealing with Bottlenecks

### Split & Merge

<img src="/images/posts/csharp-channels-part3/split.png" />

<details>
  <summary>Expand</summary>

```cs
Split<T>()
```

</details>

<img src="/images/posts/csharp-channels-part3/merge.png" />

<img src="/images/posts/csharp-channels-part3/bottleneck.png" />


## TPL Dataflow

## Conclusion
Benefits - system resource utilization (performance), composability, testability. Decreased cyclomatic complexity.


## References & Further Reading

