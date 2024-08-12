---
title: "Nuke: Deploy ASP. NET Web App to Azure"
datePublished: Sat May 14 2022 19:00:17 GMT+0000 (Coordinated Universal Time)
cuid: cl368gxuy02j2mqnvde8i3hfh
slug: nuke-deploy-asp-net-web-app-to-azure
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1652382361886/gQhfmMprZ.png
tags: azure, continuous-integration, net, aspnet-core, nuke

---

In this post, we will explore the benefits of [Nuke](https://nuke.build/) and its most common features as we deploy a web app to Azure. Our goal is to achieve a flexible, maintainable, [automated build](https://martinfowler.com/articles/continuousIntegration.html#AutomateTheBuild) process.

> Nuke is an open source, cross-platform build automation solution for .NET projects. Nuke prides itself on its simplicity and extensibility that makes build automation approachable for everyone.

Today, tools like [Continuous Integration Servers](https://about.gitlab.com/topics/ci-cd/continuous-integration-server/) perform the same job. So, why should we move the build process from CI Servers to Nuke and use them as simple runners?

* **Reduce Vendor Lock-In**: Switching from one CI platform to another requires a significant effort.
    
* **Democratize DevOps**: If something breaks or a developer adds a feature that needs changes in the build process, progress halts until the build manager is available.
    
* **Reduce Mismatch**: Occasionally, a change affects the code and the build process. When you move the feature to the development branch, you must to remember to update the related build process.
    
* **Simplify Debugging**: When things go wrong on a CI server with custom logic, you can't set breakpoints, you can't access environmental differences, logging options are limited, and you often have to wait a long time to see the results of any changes.
    

## Concepts

### Target

A target is something that must happen. A collection of targets defines your build process. They are similar to the build steps in your CI server.

```csharp
Target Restore => _ => _
    .Executes(() =>
    {
        DotNetRestore(s => s
            .SetProjectFile(Solution));
    });
```

### Parameters

A parameter is a value provided by the command line.

```csharp
[Parameter("Configuration to build - Default is 'Debug' (local) or 'Release' (server)")]
readonly Configuration Configuration = IsLocalBuild ? Configuration.Debug : Configuration.Release;
```

### Dependencies

A dependency specifies which targets need to run first.

```csharp
Target Compile => _ => _
    .DependsOn(Restore)
    .Executes(() =>
    {
        DotNetBuild(s => s
            .SetProjectFile(Solution)
            .SetConfiguration(Configuration)
            .EnableNoRestore());
    });
```

## Code

Let's start by installing Nuke:

```powershell
dotnet tool install Nuke.GlobalTool --global
```

Then, in the same directory as your solution, run:

```powershell
 nuke :setup
```

Nuke will start asking questions to set up your build project:

```powershell
NUKE Global Tool version 6.0.1 (Windows,.NETCoreApp,Version=v3.1)
How should the build project be named?
¬  _build
Where should the build project be located?
¬  ./build
Which NUKE version should be used?
¬  6.0.3 (latest release)
Which solution should be the default?
¬  nuke-sandbox-app.sln
Do you need help getting started with a basic build?
¬  No, I can do this myself...
```

At this point, Nuke will add a new project named `_build` to your solution. Find the `Build.cs` file and open it. Add the following namespaces to access all `dotnet` commands:

```csharp
using Nuke.Common.Tools.DotNet;
using static Nuke.Common.Tools.DotNet.DotNetTasks;
```

Delete all the targets and replace them with the following code:

```csharp
Target Restore => _ => _
    .Executes(() =>
    {
        DotNetRestore(s => s
            .SetProjectFile(Solution));
    });

Target Compile => _ => _
    .DependsOn(Restore)
    .Executes(() =>
    {
        DotNetBuild(s => s
            .SetProjectFile(Solution)
            .SetConfiguration(Configuration)
            .EnableNoRestore());
    });
```

Open a console in the same directory as your solution and run:

```powershell
nuke Compile
```

```powershell
╬════════════
║ Restore
╬═══
​
09:23:21 [INF] > "C:\Program Files\dotnet\dotnet.exe" restore D:\Source\Github\nuke-sandbox\nuke-sandbox-app.sln
09:23:21 [DBG]   Determining projects to restore...
09:23:22 [DBG]   Restored D:\Source\Github\nuke-sandbox\nuke-sandbox-app\nuke-sandbox-app.csproj (in 87 ms).
​
╬════════════
║ Compile
╬═══
​
09:23:22 [INF] > "C:\Program Files\dotnet\dotnet.exe" build D:\Source\Github\nuke-sandbox\nuke-sandbox-app.sln --configuration Debug --no-restore
09:23:22 [DBG] Microsoft (R) Build Engine version 17.1.0+ae57d105c for .NET
09:23:22 [DBG] Copyright (C) Microsoft Corporation. All rights reserved.
09:23:22 [DBG]
09:23:25 [DBG]   nuke-sandbox-app -> D:\Source\Github\nuke-sandbox\nuke-sandbox-app\bin\Debug\net6.0\nuke-sandbox-app.dll
09:23:26 [DBG]
09:23:26 [DBG] Build succeeded.
09:23:26 [DBG]     0 Warning(s)
09:23:26 [DBG]     0 Error(s)
09:23:26 [DBG]
09:23:26 [DBG] Time Elapsed 00:00:03.33
​
═══════════════════════════════════════
Target             Status      Duration
───────────────────────────────────────
Restore            Succeeded     < 1sec
Compile            Succeeded       0:03
───────────────────────────────────────
Total                              0:04
═══════════════════════════════════════
```

Congratulations, you are compiling your solution with Nuke. To deploy the web app to Azure, we will use the [Kudu's zip API](https://github.com/projectkudu/kudu/wiki/Deploying-from-a-zip-file-or-url). We must create a target to run the `dotnet publish` command and another to zip the results. Add two `AbsolutePath` variables to store the results of these commands:

```csharp
AbsolutePath OutputDirectory => RootDirectory / "output";

AbsolutePath ArtifactDirectory => RootDirectory / "artifact";
```

And add the following targets:

```csharp
Target Clean => _ => _
    .Before(Restore)
    .Executes(() =>
    {
        EnsureCleanDirectory(OutputDirectory);
        EnsureCleanDirectory(ArtifactDirectory);
    });

Target Publish => _ => _
    .DependsOn(Compile)
    .DependsOn(Clean)
    .Executes(() =>
    {
        DotNetPublish(s => s
            .SetProject(Solution)
            .SetConfiguration(Configuration)
            .SetOutput(OutputDirectory)
            .EnableNoRestore()
            .SetNoBuild(true));
    });

Target Zip => _ => _
    .DependsOn(Publish)
    .Executes(() =>
    {
        ZipFile.CreateFromDirectory(OutputDirectory, ArtifactDirectory / "deployment.zip");
    });
```

Run the nuke command and check the output and artifact folders:

```powershell
nuke Zip
```

Let's move to the final step and add three parameters:

```csharp
[Parameter()]
public string WebAppUser;

[Parameter()]
public string WebAppPassword;

[Parameter]
public string WebAppName;
```

And copy this target:

```csharp
Target Deploy => _ => _
    .DependsOn(Zip)
    .Requires(() => WebAppUser)
    .Requires(() => WebAppPassword)
    .Requires(() => WebAppName)
    .Executes(async () =>
    {
        var base64Auth = Convert.ToBase64String(Encoding.Default.GetBytes($"{WebAppUser}:{WebAppPassword}"));
        using (var memStream = new MemoryStream(File.ReadAllBytes(ArtifactDirectory / "deployment.zip")))
        {
            memStream.Position = 0;
            var content = new StreamContent(memStream);
            var httpClient = new HttpClient();
            httpClient.DefaultRequestHeaders.Authorization = new System.Net.Http.Headers.AuthenticationHeaderValue("Basic", base64Auth);
            var requestUrl = $"https://{WebAppName}.scm.azurewebsites.net/api/zipdeploy";
            var response = await httpClient.PostAsync(requestUrl, content);

            if (!response.IsSuccessStatusCode)
            {
                Assert.Fail("Deployment returned status code: " + response.StatusCode);
            }

        }
    });
```

To get the user and password, go to the Azure portal and find the Deployment Center option:

![azure-porta-deployment-center.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652542876714/TVBTMZxhp.png align="left")

For the user, use the last part after the backslash. Run the `help` command to see all the available options:

```powershell
nuke --help
```

```powershell
NUKE Execution Engine version 6.0.3 (Windows,.NETCoreApp,Version=v6.0)
​
Targets (with their direct dependencies):

  Restore
  Compile (default)    -> Restore
  Clean
  Publish              -> Compile, Clean
  Zip                  -> Publish
  Deploy               -> Zip

Parameters:

  --configuration         Configuration to build - Default is 'Debug' (local) or
                          'Release' (server).
  --web-app-password      <no description>
  --web-app-user          <no description>
  --web-app-name          <no description>

  --continue              Indicates to continue a previously failed build attempt.
  --help                  Shows the help text for this build assembly.
  --host                  Host for execution. Default is 'automatic'.
  --no-logo               Disables displaying the NUKE logo.
  --plan                  Shows the execution plan (HTML).
  --profile               Defines the profiles to load.
  --root                  Root directory during build execution.
  --skip                  List of targets to be skipped. Empty list skips all
                          dependencies.
  --target                List of targets to be invoked. Default is 'Compile'.
  --verbosity             Logging verbosity during build execution. Default is
                          'Normal'.
```

Run the following command:

```powershell
nuke Deploy --web-app-password <pasword> --web-app-name nuke-sandbox-app --web-app-user '$nuke-sandbox-app'
```

Go to your site to see the web app running. Finally, to see your build process in a nice dependency graph, run the following command:

```bash
nuke --plan
```

![nuke-plan.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1652543335648/yWhfoTKQ5.PNG align="left")

You can find the solution [here](https://github.com/raulnq/nuke-sandbox/tree/original). Thank you, and happy coding.