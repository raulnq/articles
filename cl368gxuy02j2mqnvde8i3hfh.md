---
title: "Nuke: Deploy ASP. NET Web App to Azure"
datePublished: Sat May 14 2022 19:00:17 GMT+0000 (Coordinated Universal Time)
cuid: cl368gxuy02j2mqnvde8i3hfh
slug: nuke-deploy-asp-net-web-app-to-azure
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1652382361886/gQhfmMprZ.png
tags: azure, continuous-integration, net, aspnet-core, nuke

---

In this post, we are going to explore the benefits of [Nuke](https://nuke.build/) and its most common features (while we deploy a web app to Azure) to achieve a flexible, maintainable, [automated build](https://martinfowler.com/articles/continuousIntegration.html#AutomateTheBuild) process.

> Nuke is an open source, cross-platform build automation solution for .NET projects. Nuke prides itself on its simplicity and extensibility that makes build automation approachable for everyone.

Today some tools are doing the same job, [Continuous Integration Servers](https://about.gitlab.com/topics/ci-cd/continuous-integration-server/). Then, why should we move the build process from the CI Servers to Nuke and let them as a dumb runner of it?:

* **Reduce Vendor Lock-In**: If you change from one CI platform to another, there will be a huge effort to do it again.
    
* **Democratize DevOps**: If something breaks or a developer adds a feature that requires changes in the build process, nothing can move forward until the build manager is free.
    
* **Reduce Mismatch**: There are cases where a change affects both code and the build process. Then when you move the feature to the develop branch, you must remember to update the associated build process (and the same happens when you move it to another type of branch).
    
* **Simplify Debugging**: When things go wrong on a CI server with custom logic, you can't set breakpoints, environmental differences are inaccessible, logging options are limited, and you frequently have to wait very long times to see the results of any changes.
    

## Concepts

### Target

A target is something that must happen. A collection of targets define your build process. They are analogous to your build steps in your CI server.

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

A dependency defines what targets should run before.

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

Let's begin installing Nuke:

```powershell
dotnet tool install Nuke.GlobalTool --global
```

Then in the same path of your solution, run:

```powershell
 nuke :setup
```

Nuke will start to ask questions to configure your build project:

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

At this point, Nuke is going to include a new project named \_build in your solution. Locate the file `Build.cs` inside and open it. Add the following namespaces to get access to all `dotnet` commands:

```csharp
using Nuke.Common.Tools.DotNet;
using static Nuke.Common.Tools.DotNet.DotNetTasks;
```

Delete all the targets and copy the following code:

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

Open a console in the same directory of your solution and run:

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

Congratulations, you are compiling your solution with Nuke. Now, to deploy the web app to Azure, we are going to use the [Kudu's zip API](https://github.com/projectkudu/kudu/wiki/Deploying-from-a-zip-file-or-url). We need to create a target to run the `dotnet publish` and another to zip the results. Add two `AbsolutePath` to store the results of the commands:

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

Let's go to the final step, add three parameters:

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

To get the user and password, go to the Azure portal and locate the Deployment Center option:

![azure-porta-deployment-center.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1652542876714/TVBTMZxhp.png align="left")

For the user, use the last part after the backslash. Run the `help` command to see all the available options to run your build:

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

Go to your site to see the web app running. Finally, to see your build process in a nice dependency graph, run:

```bash
nuke --plan
```

![nuke-plan.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1652543335648/yWhfoTKQ5.PNG align="left")

You can find the solution [here](https://github.com/raulnq/nuke-sandbox/tree/original) and the official Nuke documentation [here](https://nuke.build/docs/getting-started/philosophy.html).