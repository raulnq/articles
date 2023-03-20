---
title: "Injecting code before an application starts with .NET Startup Hooks"
datePublished: Mon Mar 20 2023 03:24:36 GMT+0000 (Coordinated Universal Time)
cuid: clfg9jqal00040al75u0b8xqh
slug: injecting-code-before-an-application-starts-with-net-startup-hooks
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1679083226531/9e33e5ff-a92c-41c6-b0aa-f4e50e3f9cda.png
tags: net

---

.NET Startup Hooks is a little-known feature that allows us to inject code into a target process before it starts running. This can be done without modifying the target process source code or recompiling it.

> For .NET Core 3+, we want to provide a low-level hook that allows injecting managed code to run before the main application's entry point. This hook will make it possible for the host to customize the behavior of managed applications during process launch after they have been deployed.

Building a hook is quite simple. Run `dotnet new classlib -n MyHook` and replace the default class with the following content:

```csharp
using System;

internal class StartupHook
{
    public static void Initialize()
    {
        Console.WriteLine("Hello from my hook!");
    }
}
```

Two things to notice here, the class is internal and there is no namespace. Let's run `dotnet new console -n MyApp` to create the target application. Modify the `Program.cs` file as follows:

```csharp
using System;

namespace MyApp
{
    public class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine(Message);
        }

        public static string Message = "Hello from my app!";
    }
}
```

Publish both projects with the following commands:

```bash
dotnet publish .\MyHook
dotnet publish .\MyApp
```

To inject our hook, create the `DOTNET_STARTUP_HOOKS` environment variable (a list of assemblies can be specified and delimited by `:` on Linux and `;` on Windows):

```bash
$env:DOTNET_STARTUP_HOOKS="C:\Source\dotnet-hooks\MyHook\bin\Debug\net7.0\publish\MyHook.dll"
```

Run `.\MyApp\bin\Debug\net7.0\MyApp.exe` to see the following output:

```bash
Hello from my hook!
Hello from my app!
```

So, what else can we do with this hook? Let's see some examples.

### Modify the console output

Go to the `MyHook` project and update the default class as follows:

```csharp
internal class StartupHook
{
    public static void Initialize()
    {
        Console.SetOut(new InvertedTextWriter(Console.Out));
    }
}

public class InvertedTextWriter : TextWriter
{
    private readonly TextWriter _writer;

    public InvertedTextWriter(TextWriter baseTextWriter)
    {
        _writer = baseTextWriter;
    }

    public override Encoding Encoding => _writer.Encoding;

    public override void Write(string value)
    {
        _writer.Write(string.Concat(value.Reverse()));
    }
}
```

Run `dotnet publish .\MyHook` and `.\MyApp\bin\Debug\net7.0\MyApp.exe` to see:

```bash
!ppa ym morf olleH
```

### Access to static fields

Go to the `MyHook` project and update the default class as follows:

```csharp
using System.Reflection;

internal class StartupHook
{
    public static void Initialize()
    {
        var program = Assembly.GetEntryAssembly()?.DefinedTypes
                             .FirstOrDefault(t => t.Name == "Program");
        var property = program?.DeclaredFields
                                   .FirstOrDefault(p => p.Name == "Message");
        property?.SetValue(null, "Text changed by the hook!");
    }
}
```

Run `dotnet publish .\MyHook` and `.\MyApp\bin\Debug\net7.0\MyApp.exe` to see:

```bash
Text changed by the hook!
```

This example is quite scary because there is no limit to what can be changed.

### Monitor your application

Go to the `MyHook` project and update the default class as follows:

```csharp
using System.Reflection;

internal class StartupHook
{
    public static void Initialize()
    {
        new Thread(MetricsPoller)
        {
            IsBackground = true,
            Name = "Poller"
        }.Start();
    }

    private static void MetricsPoller()
    {
        while (true)
        {
            var gen0 = GC.CollectionCount(0);
            var gen1 = GC.CollectionCount(1);
            var gen2 = GC.CollectionCount(2);

            Console.WriteLine($"Generation 0 {gen0}");
            Console.WriteLine($"Generation 1 {gen1}");
            Console.WriteLine($"Generation 2 {gen2}");

            Thread.Sleep(3000);
        }
    }
}
```

Go to the `MyApp` project and update the `Program.cs` file as follows:

```csharp
using System;

namespace MyApp
{
    public class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine(Message);
            Console.ReadLine();
        }

        public static string Message = "Hello from my app!";
    }
}
```

Run `dotnet publish .\MyApp`, `dotnet publish .\MyHook` and `.\MyApp\bin\Debug\net7.0\MyApp.exe` to see:

```bash
Hello from my app!
Generation 0 0
Generation 1 0
Generation 2 0
Generation 0 0
Generation 1 0
Generation 2 0
```

Creativity is the only limit to what we can do with this feature. [Here](https://github.com/dotnet/runtime/blob/main/docs/design/features/host-startup-hook.md) you can find the official documentation. Thank you, and happy coding.