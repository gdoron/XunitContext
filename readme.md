<!--
This file was generate by MarkdownSnippets.
Source File: /mdsource/readme.source.md
To change this file edit the source file and then re-run the generation using either the dotnet global tool (https://github.com/SimonCropp/MarkdownSnippets#markdownsnippetstool) or using the api (https://github.com/SimonCropp/MarkdownSnippets#running-as-a-unit-test).
-->
# <img src="https://raw.github.com/SimonCropp/XunitLogger/master/icon.png" height="40px"> XunitLogger

Extends [xUnit](https://xunit.net/) to simplify logging.

Redirects [Trace.Write](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.trace.write), [Debug.Write](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.debug.write), and [Console.Write and Console.Error.Write](https://docs.microsoft.com/en-us/dotnet/api/system.console.write) to [ITestOutputHelper](https://xunit.net/docs/capturing-output). Also provides static access to the current [ITestOutputHelper](https://xunit.net/docs/capturing-output) for use within testing utility methods.

Uses [AsyncLocal](https://docs.microsoft.com/en-us/dotnet/api/system.threading.asynclocal-1) to track state.


## The NuGet package [![NuGet Status](http://img.shields.io/nuget/v/XunitLogger.svg)](https://www.nuget.org/packages/XunitLogger/)

https://nuget.org/packages/XunitLogger/


## Usage


### ClassBeingTested

<!-- snippet: ClassBeingTested.cs -->
```cs
using System;
using System.Diagnostics;

static class ClassBeingTested
{
    public static void Method()
    {
        Trace.WriteLine("From Trace");
        Console.WriteLine("From Console");
        Debug.WriteLine("From Debug");
        Console.Error.WriteLine("From Console Error");
    }
}
```
<sup>[snippet source](/src/Tests/Snippets/ClassBeingTested.cs#L1-L13)</sup>
<!-- endsnippet -->


### XunitLoggingBase

`XunitLoggingBase` is an abstract base class for tests. It exposes logging methods for use from unit tests, and handle the flushing of longs in its `Dispose` method. `XunitLoggingBase` is actually a thin wrapper over `XunitLogger`. `XunitLogger`s `Write*` methods can also be use inside a test inheriting from `XunitLoggingBase`.

<!-- snippet: TestBaseSample.cs -->
```cs
using Xunit;
using Xunit.Abstractions;

public class TestBaseSample  :
    XunitLoggingBase
{
    [Fact]
    public void Write_lines()
    {
        WriteLine("From Test");
        ClassBeingTested.Method();

        var logs = XunitLogging.Logs;

        Assert.Contains("From Test", logs);
        Assert.Contains("From Trace", logs);
        Assert.Contains("From Debug", logs);
        Assert.Contains("From Console", logs);
        Assert.Contains("From Console Error", logs);
    }

    public TestBaseSample(ITestOutputHelper output) :
        base(output)
    {
    }
}
```
<sup>[snippet source](/src/Tests/Snippets/TestBaseSample.cs#L1-L26)</sup>
<!-- endsnippet -->


### XunitLogger

`XunitLogger` provides static access to the logging state for tests. It exposes logging methods for use from unit tests, however registration of [ITestOutputHelper](https://xunit.net/docs/capturing-output) and flushing of logs must be handled explicitly.

<!-- snippet: XunitLoggerSample.cs -->
```cs
using System;
using Xunit;
using Xunit.Abstractions;

public class XunitLoggerSample :
    IDisposable
{
    [Fact]
    public void Usage()
    {
        XunitLogging.WriteLine("From Test");

        ClassBeingTested.Method();

        var logs = XunitLogging.Logs;

        Assert.Contains("From Test", logs);
        Assert.Contains("From Trace", logs);
        Assert.Contains("From Debug", logs);
        Assert.Contains("From Console", logs);
        Assert.Contains("From Console Error", logs);
    }

    public XunitLoggerSample(ITestOutputHelper testOutput)
    {
        XunitLogging.Register(testOutput);
    }

    public void Dispose()
    {
        XunitLogging.Flush();
    }
}
```
<sup>[snippet source](/src/Tests/Snippets/XunitLoggerSample.cs#L1-L33)</sup>
<!-- endsnippet -->

`XunitLogger` redirects [Trace.Write](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.trace.write), [Console.Write](https://docs.microsoft.com/en-us/dotnet/api/system.console.write), and [Debug.Write](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.debug.write) in its static constructor.

<!-- snippet: writeRedirects -->
```cs
Trace.Listeners.Clear();
Trace.Listeners.Add(new TraceListener());
#if (NETSTANDARD)
DebugPoker.Overwrite(
    text =>
    {
        if (string.IsNullOrEmpty(text))
        {
            return;
        }

        if (text.EndsWith(Environment.NewLine))
        {
            WriteLine(text.TrimTrailingNewline());
            return;
        }

        Write(text);
    });
#else
Debug.Listeners.Clear();
Debug.Listeners.Add(new TraceListener());
#endif
var writer = new TestWriter();
Console.SetOut(writer);
Console.SetError(writer);
```
<sup>[snippet source](/src/XunitLogger/XunitLogging.cs#L14-L43)</sup>
<!-- endsnippet -->

These API calls are then routed to the correct xUnit [ITestOutputHelper](https://xunit.net/docs/capturing-output) via a static [AsyncLocal](https://docs.microsoft.com/en-us/dotnet/api/system.threading.asynclocal-1).


### Filters

`XunitLogger.Filters` can be used to filter out unwanted lines:

<!-- snippet: FilterSample.cs -->
```cs
using Xunit;
using Xunit.Abstractions;
using XunitLogger;

public class FilterSample :
    XunitLoggingBase
{
    static FilterSample()
    {
        Filters.Add(x => x != null && !x.Contains("ignored"));
    }

    [Fact]
    public void Write_lines()
    {
        WriteLine("first");
        WriteLine("with ignored string");
        WriteLine("last");
        var logs = XunitLogging.Logs;

        Assert.Contains("first", logs);
        Assert.DoesNotContain("with ignored string", logs);
        Assert.Contains("last", logs);
    }

    public FilterSample(ITestOutputHelper output) :
        base(output)
    {
    }
}
```
<sup>[snippet source](/src/Tests/Snippets/FilterSample.cs#L1-L30)</sup>
<!-- endsnippet -->

Filters are static and shared for all tests.


### Context

For every tests there is a contextual API to perform several operations.

 * `Context.TestOutput`: Access to [ITestOutputHelper](https://xunit.net/docs/capturing-output).
 * `Context.Write` and `Context.WriteLine`: Write to the current log.
 * `Context.LogMessages`: Access to all log message for the current test.
 * Counters: Provide access in predicable and incrementing values for the following types: `Guid`, `Int`, `Long`, `UInt`, and `ULong`.

There is also access via a static API.

<!-- snippet: ContextSample.cs -->
```cs
using Xunit;
using Xunit.Abstractions;

public class ContextSample  :
    XunitLoggingBase
{
    [Fact]
    public void Usage()
    {
        var currentGuid = Context.CurrentGuid;

        var nextGuid = Context.NextGuid();

        Context.WriteLine("Some message");

        var currentLogMessages = Context.LogMessages;

        var testOutputHelper = Context.TestOutput;
    }

    [Fact]
    public void StaticUsage()
    {
        var currentGuid = XunitLogging.Context.CurrentGuid;

        var nextGuid = XunitLogging.Context.NextGuid();

        XunitLogging.Context.WriteLine("Some message");

        var currentLogMessages = XunitLogging.Context.LogMessages;

        var testOutputHelper = XunitLogging.Context.TestOutput;
    }

    public ContextSample(ITestOutputHelper output) :
        base(output)
    {
    }
}
```
<sup>[snippet source](/src/Tests/Snippets/ContextSample.cs#L1-L39)</sup>
<!-- endsnippet -->


#### Counters

Provide access to predicable and incrementing values for the following types: `Guid`, `Int`, `Long`, `UInt`, and `ULong`.


##### Non Test Context usage

Counters can also be used outside of the current test context:

<!-- snippet: NonTestContextUsage -->
```cs
var current = Counters.CurrentGuid;

var next = Counters.NextGuid();

var counter = new GuidCounter();
var localCurrent = counter.Current;
var localNext = counter.Next();
```
<sup>[snippet source](/src/Tests/Snippets/CountersSample.cs#L9-L17)</sup>
<!-- endsnippet -->


##### Implementation

<!-- snippet: LoggingContext_Counters.cs -->
```cs
using System;

namespace XunitLogger
{
    public partial class Context
    {
        GuidCounter GuidCounter = new GuidCounter();
        IntCounter IntCounter = new IntCounter();
        LongCounter LongCounter = new LongCounter();
        UIntCounter UIntCounter = new UIntCounter();
        ULongCounter ULongCounter = new ULongCounter();

        public uint CurrentUInt
        {
            get => UIntCounter.Current;
        }

        public int CurrentInt
        {
            get => IntCounter.Current;
        }

        public long CurrentLong
        {
            get => LongCounter.Current;
        }

        public ulong CurrentULong
        {
            get => ULongCounter.Current;
        }

        public Guid CurrentGuid
        {
            get => GuidCounter.Current;
        }

        public uint NextUInt()
        {
            return UIntCounter.Next();
        }

        public int NextInt()
        {
            return IntCounter.Next();
        }

        public long NextLong()
        {
            return LongCounter.Next();
        }

        public ulong NextULong()
        {
            return ULongCounter.Next();
        }

        public Guid NextGuid()
        {
            return GuidCounter.Next();
        }
    }
}
```
<sup>[snippet source](/src/XunitLogger/LoggingContext_Counters.cs#L1-L63)</sup>
<!-- endsnippet -->

<!-- snippet: Counters.cs -->
```cs
using System;

namespace XunitLogger
{
    public static class Counters
    {
        static GuidCounter GuidCounter = new GuidCounter();
        static IntCounter IntCounter = new IntCounter();
        static LongCounter LongCounter = new LongCounter();
        static UIntCounter UIntCounter = new UIntCounter();
        static ULongCounter ULongCounter = new ULongCounter();

        public static uint CurrentUInt
        {
            get => UIntCounter.Current;
        }

        public static int CurrentInt
        {
            get => IntCounter.Current;
        }

        public static long CurrentLong
        {
            get => LongCounter.Current;
        }

        public static ulong CurrentULong
        {
            get => ULongCounter.Current;
        }

        public static Guid CurrentGuid
        {
            get => GuidCounter.Current;
        }

        public static uint NextUInt()
        {
            return UIntCounter.Next();
        }

        public static int NextInt()
        {
            return IntCounter.Next();
        }

        public static long NextLong()
        {
            return LongCounter.Next();
        }

        public static ulong NextULong()
        {
            return ULongCounter.Next();
        }

        public static Guid NextGuid()
        {
            return GuidCounter.Next();
        }
    }
}
```
<sup>[snippet source](/src/XunitLogger/Counters.cs#L1-L63)</sup>
```cs
using System;

namespace XunitLogger
{
    public partial class Context
    {
        GuidCounter GuidCounter = new GuidCounter();
        IntCounter IntCounter = new IntCounter();
        LongCounter LongCounter = new LongCounter();
        UIntCounter UIntCounter = new UIntCounter();
        ULongCounter ULongCounter = new ULongCounter();

        public uint CurrentUInt
        {
            get => UIntCounter.Current;
        }

        public int CurrentInt
        {
            get => IntCounter.Current;
        }

        public long CurrentLong
        {
            get => LongCounter.Current;
        }

        public ulong CurrentULong
        {
            get => ULongCounter.Current;
        }

        public Guid CurrentGuid
        {
            get => GuidCounter.Current;
        }

        public uint NextUInt()
        {
            return UIntCounter.Next();
        }

        public int NextInt()
        {
            return IntCounter.Next();
        }

        public long NextLong()
        {
            return LongCounter.Next();
        }

        public ulong NextULong()
        {
            return ULongCounter.Next();
        }

        public Guid NextGuid()
        {
            return GuidCounter.Next();
        }
    }
}
```
<sup>[snippet source](/src/XunitLogger/LoggingContext_Counters.cs#L1-L63)</sup>
<!-- endsnippet -->

<!-- snippet: GuidCounter.cs -->
```cs
using System;
using System.Threading;

namespace XunitLogger
{
    public class GuidCounter
    {
        int current;

        public Guid Current
        {
            get => IntToGuid(current);
        }

        public Guid Next()
        {
            var value = Interlocked.Increment(ref current);
            return IntToGuid(value);
        }

        static Guid IntToGuid(int value)
        {
            var bytes = new byte[16];
            BitConverter.GetBytes(value).CopyTo(bytes, 0);
            return new Guid(bytes);
        }
    }
}
```
<sup>[snippet source](/src/XunitLogger/Counters/GuidCounter.cs#L1-L28)</sup>
<!-- endsnippet -->

<!-- snippet: LongCounter.cs -->
```cs
using System.Threading;

namespace XunitLogger
{
    public class LongCounter
    {
        long current;

        public long Current
        {
            get => current;
        }

        public long Next()
        {
            return Interlocked.Increment(ref current);
        }
    }
}
```
<sup>[snippet source](/src/XunitLogger/Counters/LongCounter.cs#L1-L19)</sup>
```cs
using System.Threading;

namespace XunitLogger
{
    public class ULongCounter
    {
        long current;

        public ulong Current
        {
            get => (ulong) current;
        }

        public ulong Next()
        {
            return (ulong) Interlocked.Increment(ref current);
        }
    }
}
```
<sup>[snippet source](/src/XunitLogger/Counters/ULongCounter.cs#L1-L19)</sup>
<!-- endsnippet -->


## Icon

[Wolverine](http://thenounproject.com/term/wolverine/18415/) designed by [Mike Rowe](https://thenounproject.com/itsmikerowe/) from The Noun Project
